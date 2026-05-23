# ⚡ Lambda Architecture এবং Kappa Architecture

Lambda Architecture হলো এমন একটি data architecture যেখানে একই raw event stream থেকে **দুই ধরনের truth** তৈরি করা হয়।
একদিকে থাকে **অত্যন্ত accurate কিন্তু slow batch computation**।
অন্যদিকে থাকে **low-latency but incremental real-time computation**।
এই দুই ফলাফল একত্রে serve করে এমন layer-ই Lambda Architecture-এর কেন্দ্র।

এই ডকুমেন্টে Lambda Architecture-এর গভীর বিশ্লেষণ, Kappa Architecture-এর সাথে তুলনা, বাংলাদেশি ব্যবসায়িক উদাহরণ, ASCII diagram, PHP code, এবং Node.js/JavaScript implementation দেখানো হয়েছে।

## 🧾 Definition

### 1. Lambda Architecture — সংজ্ঞা ও ইতিহাস

**Lambda Architecture** প্রথম জনপ্রিয় করেন **Nathan Marz**।
Nathan Marz-ই **Apache Storm**-এর creator হিসেবে পরিচিত।
তিনি এবং James Warren মিলে **Big Data: Principles and best practices of scalable realtime data systems** বইতে এই architecture-এর philosophy ব্যাখ্যা করেন।

Lambda Architecture-এর জন্ম হয় একটি খুব বাস্তব problem থেকে।
Problemটি হলো:
একই সাথে কীভাবে **accurate historical results** এবং **low-latency real-time results** পাওয়া যাবে?

ধরা যাক bKash প্রতিদিন কোটি কোটি transaction process করছে।
যদি সব data নিয়ে Spark বা Hadoop-এ nightly batch job চালানো হয়, তাহলে ফলাফল accurate হবে, কিন্তু fraud alert real-time-এ পাওয়া যাবে না।
আবার যদি শুধু streaming pipeline ব্যবহার করা হয়, তাহলে speed পাওয়া যায়, কিন্তু late event, bug fix, replay, audit, reconciliation—এসব জায়গায় accuracy ঝুঁকিতে পড়ে।
Lambda এই দুই চাহিদাকে একই architecture-এ মিলিয়ে দেয়।

Lambda Architecture-এর fundamental equation সাধারণত এভাবে লেখা হয়:

```text
query = λ(batch_view, realtime_view)
```

এখানে `λ` কোনো programming language-এর lambda expression নয়।
এখানে `λ` দ্বারা বোঝানো হচ্ছে merge function বা serving logic।
অর্থাৎ final query result আসবে দুইটি source থেকে:

- `batch_view` = historical raw data থেকে recompute করা authoritative result
- `realtime_view` = recent event-এর উপর incremental fast result

Lambda Architecture-এর মূল idea হলো:

- raw data কখনও discard করবে না
- raw data immutable রাখবে
- batch layer সব data থেকে truth recompute করবে
- speed layer last batch-এর পরের gap পূরণ করবে
- serving layer দুই result merge করে user-facing answer দেবে

এটি Big Data যুগের একটি architecture।
বিশেষ করে যখন Hadoop-based offline computation এবং Storm-based real-time computation আলাদা দুনিয়া ছিল, তখন Lambda ছিল একটি bridge pattern।

### 1.1 কোন সমস্যার সমাধান করে?

Lambda Architecture মূলত চারটি tension একসাথে solve করতে চায়:

- **Accuracy vs Latency** — accurate result সাধারণত slow, fast result সাধারণত incremental
- **Scale vs Simplicity** — petabyte scale data reprocess করতে batch দরকার, কিন্তু product feature-এর জন্য live dashboard-ও দরকার
- **Auditability vs Speed** — financial systems-এ history দরকার, operations team-এর জন্য instant anomaly detection-ও দরকার
- **Bug Recovery vs Continuity** — code bug থাকলে full recomputation দরকার, কিন্তু business-কে real-time numbers দেখানোও বন্ধ করা যাবে না

যে সময় Lambda আসে, তখন ecosystem-এর চেহারা roughly এমন ছিল:

- Batch world = Hadoop MapReduce, Hive, Pig
- Stream world = Storm, Samza, পরে Kafka Streams ও Flink
- Serving world = HBase, Cassandra, Druid, ElasticSearch

এই বিচ্ছিন্ন stack-গুলোকে একত্রে design করার জন্য Lambda একটি vocabulary দেয়।

### 1.2 “Big Data” বইয়ের context

Nathan Marz-এর big data philosophy-তে raw data-কে **immutable fact** হিসেবে ধরার উপর জোর দেওয়া হয়।
কারণ event-level fact একবার সঠিকভাবে capture করতে পারলে পরের যেকোনো derived view আবার বানানো যায়।

এই বইয়ের context-এ Lambda Architecture-এর key assumptions ছিল:

- distributed systems fail করবে
- code-এ bug থাকবে
- schema evolve করবে
- late data আসবেই
- human error unavoidable
- therefore raw immutable data save করা সবচেয়ে strategic decision

এ কারণেই Lambda Architecture শুধু pipeline design নয়।
এটি data correctness philosophy।

### 2. The Three Layers

Lambda Architecture-এর তিনটি principal layer আছে:

1. **Batch Layer**
2. **Speed Layer**
3. **Serving Layer**

এখন প্রতিটি layer গভীরভাবে দেখি।

### 2.1 Batch Layer (ব্যাচ লেয়ার)

Batch layer হলো Lambda Architecture-এর foundation।
এখানে পুরো system-এর **master dataset** রাখা হয়।
এই dataset immutable, append-only, এবং raw facts-ভিত্তিক।

Batch layer-এর প্রধান দায়িত্ব:

- সব raw event store করা
- historical data preserve করা
- periodic recomputation চালানো
- full batch view তৈরি করা
- correctness-এর authoritative source হওয়া

Batch layer-এর key characteristics:

- **Master dataset immutable**
- **Append-only ingestion**
- **সব data থেকে recomputation**
- **High latency**
- **Highest accuracy**
- **Fault recovery friendly**

এখানে সাধারণত query-ready normalized OLTP model রাখা হয় না।
বরং raw log, fact record, event stream, অথবা denormalized immutable blob রাখা হয়।

উদাহরণ:

- bKash-এর সব transaction event
- Daraz-এর সব order, click, impression, payment, return event
- GP-এর call detail records (CDR)
- Prothom Alo-এর article view log

Batch layer technologies হিসেবে ঐতিহাসিকভাবে ব্যবহৃত হয়:

- Hadoop MapReduce
- Apache Spark batch jobs
- Hive
- Presto/Trino for batch analytics
- object storage like S3 বা data lake
- HDFS

Batch layer-এর output হলো **batch views**।
এই batch views precomputed table, index, aggregate, feature set, cube, বা search snapshot হতে পারে।

Bangladesh example:

- **bKash daily reconciliation**: দিন শেষে সব transaction, reversal, fee, settlement event মিলিয়ে exact ledger balance recompute করা
- **Daraz monthly sales reports**: seller-wise, category-wise, district-wise sales data full historical processing করে বের করা
- **Prothom Alo readership archive**: গত পাঁচ বছরের article traffic trend recompute করে editorial insight তৈরি করা

Batch layer slow হওয়াটা defect নয়।
এটি design trade-off।
কারণ এখানে লক্ষ্য speed নয়, **accuracy এবং recoverability**।

### 2.2 Speed Layer (স্পিড লেয়ার / Real-time Layer)

Speed layer কাজ করে recent data নিয়ে।
এটি full historical recomputation করে না।
এটি ধরে নেয় last successful batch view পর্যন্ত accurate truth already আছে।
তারপর সেই cutoff point-এর পরে আসা event-গুলো fast path-এ process করে।

Speed layer-এর responsibilities:

- recent event consume করা
- incremental aggregation maintain করা
- low latency alert generate করা
- live dashboard metrics update করা
- batch refresh হওয়া পর্যন্ত temporary result serve করা

Key characteristics:

- **Low latency**: milliseconds থেকে seconds
- **Approximate / incremental result**
- **Only recent data**
- **Temporary view**
- **Complexity around ordering and duplicates**

Technologies:

- Apache Storm
- Kafka Streams
- Apache Flink
- Apache Samza
- Spark Structured Streaming (modern hybrid use case)

Speed layer-এর output হলো **real-time views**।
এগুলো সাধারণত ephemeral বা overwrite-able materialized view।
পরবর্তী batch run এসে historical portion absorb করে নিলে speed view-এর পুরনো অংশ discard হয়ে যায়।

Bangladesh example:

- **bKash real-time fraud alerts**: একই device থেকে অস্বাভাবিক send money pattern detect করা
- **Pathao live order tracking**: rider location event থেকে current ETA update করা
- **Foodpanda surge monitoring**: প্রতি মিনিটে কোন area-তে demand spike হচ্ছে দেখা
- **Daraz trending product counter**: এখনই কোন product বেশি click পাচ্ছে সেটা dashboard-এ দেখানো

Speed layer-এ exactness-এর চেয়ে freshness অনেক সময় বেশি গুরুত্বপূর্ণ।
তবে approximate মানেই careless নয়।
বরং carefully engineered incremental correctness দরকার।

### 2.3 Serving Layer (সার্ভিং লেয়ার)

Serving layer হলো user-facing access layer।
এখানেই batch view এবং real-time view merge করে final query result তৈরি করা হয়।

Serving layer-এর responsibilities:

- batch view index করা
- real-time view read করা
- timestamp-aware merge করা
- API বা dashboard-friendly response বানানো
- query performance optimize করা

Serving layer technologies:

- Apache Druid
- HBase
- Cassandra
- ElasticSearch
- Redis + custom API layer
- Pinot / ClickHouse (modern variants)

Serving layer-এর merge logic conceptualভাবে এমন:

```text
final_result = merge(batch_result_up_to_T, realtime_result_after_T)
```

এখানে `T` হলো last successful batch cutoff time।
Batch layer cover করে `[0, T]` interval।
Speed layer cover করে `(T, now]` interval।
Serving layer নিশ্চিত করে যেন overlap, double count, missing gap—কোনোটাই না হয়।

Bangladesh example:

- **Daraz analytics dashboard** historical sales trend batch view থেকে নেবে, আর last 15 minute-এর live click/order stats speed view থেকে নেবে
- **bKash operations dashboard** গতকাল পর্যন্ত exact numbers batch থেকে নেবে, এখনকার suspicious transaction count speed layer থেকে নেবে
- **GP customer 360** monthly usage profile batch view থেকে, current network incident status speed layer থেকে নেবে

Serving layer শুধু database নয়।
এটি অনেক সময় orchestration logic, API composition, cache invalidation এবং versioned view switching-এর সমন্বয়।

### 3. Master Dataset — Immutable, Append-only

Lambda Architecture-এর সবচেয়ে powerful অংশ হলো **master dataset** concept।
এটি mutable business table নয়।
এটি raw fact log।

উদাহরণ fact-based schema:

- কে কাজটি করেছে?
- কী কাজ করেছে?
- কখন করেছে?
- কোন entity-র উপর করেছে?
- context কী ছিল?
- event source কী ছিল?

উদাহরণ event fields:

- `event_id`
- `event_type`
- `occurred_at`
- `producer`
- `entity_id`
- `customer_id`
- `payload`
- `schema_version`
- `trace_id`

Immutability কেন জরুরি?

- recomputation সম্ভব হয়
- audit trail থাকে
- late event পরে ঢোকানো যায়
- code bug fix করে historical correction করা যায়
- derived table corrupt হলে raw data safe থাকে
- regulatory review-তে raw trail দেখানো যায়

Append-only storage-এর common options:

- HDFS
- S3 / object storage
- Kafka with long retention
- Iceberg/Delta raw bronze table
- archival log store

Bangladesh example:

ধরুন bKash-এর ১০ বছরের সব transaction event append-only format-এ রাখা হলো।
পরের বছর fraud scoring logic বদলালো।
এখন নতুন algorithm দিয়ে পুরনো event replay/recompute করে exact historical feature set বানানো সম্ভব।
এই flexibility master dataset ছাড়া পাওয়া কঠিন।

### 4. Batch View Recomputation

Lambda Architecture-এর batch layer time-based recomputation চালায়।
এই recomputation hourly, six-hourly, daily, nightly, weekly—business need অনুযায়ী হতে পারে।

Batch recomputation-এর ধাপ:

1. raw master dataset scan করা
2. filter/normalize/enrich করা
3. aggregate/model/train/index তৈরি করা
4. নতুন batch view materialize করা
5. atomic swap করে serving layer-এ publish করা

এখানে গুরুত্বপূর্ণ idea হলো:
পুরো historical data থেকে view আবার বানানো যায়।
এই কারণে batch result সবচেয়ে authoritative।

Self-healing property:

- গত সপ্তাহে ETL code-এ VAT calculation bug ছিল
- bug fix করার পর full recomputation চালাও
- new batch view old wrong view replace করবে
- downstream queries automatically corrected output পাবে

Computationally expensive কেন?

- whole dataset scan লাগে
- join/aggregation heavy হতে পারে
- large cluster দরকার হতে পারে
- ML feature computation cost high হতে পারে

তবু enterprise systems এই খরচ দেয় কারণ correctness premium।
বিশেষত banking, telecom, ad-tech billing, marketplace settlement—এসব জায়গায় batch correction essential।

Incremental batch processing-ও সম্ভব।
কখনও full recompute নয়, বরং partition-aware recompute করা হয়।
উদাহরণ:

- date partition by day
- seller partition by region
- late-event impacted partition only rebuild

তবে philosophy একই থাকে:
raw data authority, derived view disposable।

### 5. Speed Layer — Incremental Processing in Detail

Speed layer full recomputation করে না।
এটি last batch completion-এর পরে আসা event-গুলো incrementalভাবে process করে।

Typical responsibilities:

- live counters
- sliding window metrics
- anomaly detection
- current session stats
- live leaderboard/trending
- alert generation

Complexity points:

- duplicate event handling
- event ordering
- watermark / late arrival policy
- exactly-once বা effectively-once semantics
- state recovery
- checkpointing

Speed layer temporary কেন?

কারণ batch layer eventually historical truth refresh করে দেয়।
Speed layer মূলত সেই gap পূরণ করে যা batch latency create করে।

Example:

- nightly batch completes at 02:00 AM
- এখন 03:15 PM
- batch view covers data until 02:00 AM
- speed layer covers 02:00 AM → 03:15 PM
- next batch run হলে speed layer-এর ওই অংশ historical truth-এর মধ্যে fold হয়ে যাবে

Speed vs accuracy trade-off:

- live page view count approximate হতে পারে
- fraud rule false positive সামান্য বেশি হতে পারে
- trending list small correction পেতে পারে
- কিন্তু user experience fresh থাকে

বাংলাদেশ context:

- Chaldal live inventory reservation counter
- Pathao rider demand heatmap
- GP network incident event stream
- Prothom Alo breaking news article trend score প্রতি minute update

যেখানে “এখন কী হচ্ছে?” প্রশ্নের উত্তর দরকার, speed layer সেখানে অপরিহার্য।

### 6. Serving Layer — Query Merging in Detail

Serving layer-এর সবচেয়ে delicate কাজ হলো merge correctness।

Common merge strategies:

- **Time-split merge**
- **Partition-split merge**
- **Entity-level override**
- **Union then aggregate**
- **Batch baseline + speed delta**

সবচেয়ে common strategy হলো time-split merge:

- batch view covers `[0, T_batch]`
- speed view covers `(T_batch, now]`
- final answer = batch baseline + real-time delta

উদাহরণ: Daraz seller dashboard

- batch table: `seller_daily_revenue_until_batch`
- speed Redis counter: `seller_live_revenue_since_batch`
- API response: `batch_total + live_delta`

Serving layer-এর concerns:

- overlap হলে double count হবে
- batch lag হলে stale baseline হতে পারে
- clock skew থাকলে boundary mismatch হতে পারে
- speed layer reset হলে gap detect করতে হবে
- atomic batch view swap না হলে inconsistent result আসবে

অনেক system-এ serving layer আলাদা service হিসেবে থাকে।
কিছু system-এ Druid/Pinot-এর মতো engine নিজেই batch + real-time segment serve করতে পারে।
আবার কিছু system-এ custom API merge logic লেখা হয়।

Serving layer-ই user-facing contract define করে।
তাই architecture diagram-এ এটি simple box হলেও বাস্তবে এটি governance-heavy zone।

### 7. Lambda Architecture — Advantages

- Fault-tolerant: raw master data থাকলে যেকোনো corrupted view আবার বানানো যায়
- Late-arriving data handle করতে পারে, কারণ batch recomputation historical truth correct করে
- Speed layer crash করলেও batch layer long-term correctness ধরে রাখে
- Batch এবং speed আলাদাভাবে scale করা যায়
- Regulated environment-এ audit, replay, reconciliation সহজ হয়
- Derived view disposable হওয়ায় experimentation করা যায়
- Fraud detection, dashboard, alerting-এর জন্য low latency path দেয়
- Historical analytics, finance settlement, KPI reporting-এর জন্য accurate path দেয়
- Code bug fix করে recompute করার সুযোগ দেয়
- Data science team raw event history ব্যবহার করে নতুন feature engineering করতে পারে

### 8. Lambda Architecture — Disadvantages (Critical)

- **Code duplication**: একই business rule batch pipeline এবং streaming pipeline—দুই জায়গায় লিখতে হয়
- **Operational complexity**: দুই ধরনের infrastructure maintain করতে হয়
- **Debugging difficulty**: batch আর speed result mismatch হলে root cause খুঁজে বের করা কঠিন
- **Eventual convergence**: speed layer-এর live answer পরে batch result-এর সাথে slightly different হতে পারে
- **Cost**: দুই parallel system চালাতে storage, compute, ops cost বেশি
- **Skill burden**: team-কে batch compute + stream processing + serving merge—তিনটাতেই দক্ষ হতে হয়
- **Testing complexity**: একই logic দুবার test করতে হয়
- **Boundary coordination**: batch cutoff timestamp, watermark, late event policy ঠিক করা কঠিন
- **Data contract drift**: schema evolution দুই pipeline-এ sync রাখা কঠিন
- **Human overhead**: incident response-এ “truth কোনটা?” প্রশ্নটি operationally painful হতে পারে

### 9. Kappa Architecture — The Alternative

Kappa Architecture প্রস্তাব করেন **Jay Kreps**।
Jay Kreps Kafka-এর co-creator হিসেবে পরিচিত।
Kappa-এর মূল বক্তব্য খুব সরল:

> যদি streaming system-ই historical replay handle করতে পারে, তাহলে আলাদা batch layer কেন রাখবো?

অর্থাৎ Kappa Architecture batch layer eliminate করে।
একটি single stream processing codepath দিয়েই real-time processing এবং historical reprocessing—দুই কাজ করা হবে।

### 9.1 What is Kappa Architecture?

Kappa Architecture-এ:

- event log-ই master dataset
- stream processor-ই primary compute layer
- materialized view-ই queryable state
- reprocessing দরকার হলে event log replay করা হয়

এখানে historical recomputation-এর জন্য Hadoop-style আলাদা batch job লিখতে হয় না।
নতুন version-এর stream app deploy করে event log শুরু থেকে replay করা যায়।
এই replay Kafka retention, tiered storage, অথবা durable event log-এর উপর নির্ভর করে।

### 9.2 How Kappa Works

Kappa flow সাধারণত এমন:

1. data আসে Kafka topic-এ
2. stream processor event consume করে
3. stateful aggregation / transformation চালায়
4. materialized view update করে
5. query service ওই view থেকে উত্তর দেয়
6. logic change হলে নতুন processor version deploy করে replay করা হয়

এখানে same codepath ব্যবহার হয়:

- live processing-এর জন্য
- backfill/rebuild-এর জন্য
- bug fix replay-এর জন্য
- new derived view build-এর জন্য

এ কারণেই Kappa proponents বলে:
এক কোড, এক system, এক operational model।

### 9.3 Kappa Advantages

- Single codebase: batch + speed duplication নেই
- Operational simplicity: একটিমাত্র processing paradigm maintain করতে হয়
- Debugging easier: one code path means fewer semantic mismatches
- Replay uses the same business logic as live stream
- Developer productivity বাড়তে পারে
- Small-to-medium streaming-first products-এ খুব practical
- Pathao-style live analytics, content trending, clickstream metrics-এ ভালো fit হতে পারে

### 9.4 Kappa Disadvantages

- Very large datasets replay করতে অনেক সময় লাগতে পারে
- Complex historical computations সবসময় stream-friendly নয়
- Heavy ML training বা large cube recomputation stream model-এ awkward হতে পারে
- Long retention event log storage cost বাড়ায়
- Replay performance business SLA-এর সাথে খাপ নাও খেতে পারে
- Exactly-once state recovery and long-running stream correctness maintain করা কঠিন
- যদি event log অসম্পূর্ণ হয়, master truth দুর্বল হয়ে যায়

### 10. Lambda vs Kappa — Detailed Comparison

| Feature | Lambda | Kappa |
|---|---|---|
| Codebase | Dual: batch + stream | Single: stream only |
| Accuracy | Batch recomputation authoritative | Stream correctness-এর উপর নির্ভরশীল |
| Latency | Low via speed layer | Low via streaming |
| Reprocessing | Full/partitioned batch recomputation | Event log replay |
| Operational complexity | High | তুলনামূলক কম |
| Cost | Higher | Lower to medium |
| Best fit | Massive historical + real-time mix | Stream-friendly workloads |
| Late data repair | Strong | Depends on replay discipline |
| Regulatory reconciliation | Excellent | Possible but domain-dependent |
| Team skill need | Batch + streaming + serving | Strong streaming expertise |

আরও সূক্ষ্ম পার্থক্য:

- Lambda accuracy-first architecture
- Kappa simplicity-first architecture
- Lambda old-school big data ecosystem-এ natural
- Kappa Kafka-centric ecosystem-এ natural
- Lambda-তে derived views disposable
- Kappa-তে state stores disposable কিন্তু replay time critical
- Lambda-তে batch truth authoritative
- Kappa-তে event log + streaming correctness authoritative

### 11. When would Lambda still win?

অনেকেই ধরে নেয় Kappa এলে Lambda obsolete।
বাস্তবে তা নয়।

Lambda এখনও খুব relevant যখন:

- dataset এত বড় যে full replay impractical
- nightly reconciliation legal requirement
- batch computation stream logic-এর থেকে মৌলিকভাবে ভিন্ন
- ML training, feature backfill, finance settlement, tax reporting, risk audit—এসব heavy offline workload আছে
- business already mature Hadoop/Spark ecosystem ব্যবহার করে
- correctness preview vs final authoritative report—দুই ধরনের answer business officially accept করে

bKash core ledger reconciliation Lambda-এর textbook use case হতে পারে।
কারণ “লাইভ approximate number” এবং “end-of-day exact number” একই জিনিস নয়।

### 12. Implementation Patterns

#### 12.1 Dual Write vs Single Ingestion

সবচেয়ে বড় anti-pattern হলো source application থেকে আলাদা করে batch store-এ write এবং speed pipeline-এ write করা।
এতে divergence তৈরি হয়।

ভালো pattern:

- **single ingestion point**
- event once produce করা
- তারপর fork to batch + speed

Best practice flow:

- application emits event to Kafka
- Kafka topic retained durably
- batch ingestion reads same topic into lake
- speed processor reads same topic into incremental state

এতে source of truth একটাই থাকে।

#### 12.2 View Synchronization

Serving layer-কে জানতে হবে current batch view কোন cutoff পর্যন্ত valid।
এজন্য metadata table দরকার।
সাধারণ fields:

- `view_name`
- `version`
- `batch_start`
- `batch_end`
- `published_at`
- `state`

Atomic swap pattern:

- build new batch view in shadow table/index
- validate row count/checksum
- update serving metadata atomically
- old readers finish
- new readers switch to latest version

#### 12.3 Schema Evolution

Master dataset দীর্ঘমেয়াদি হলে schema change হবেই।
তাই backward/forward compatibility দরকার।

Common tools:

- Avro
- Protobuf
- JSON Schema with discipline
- schema registry

Rules:

- additive field change preferred
- default value maintain করা
- event version include করা
- replay-aware deserializer লেখা
- data contract testing চালানো

#### 12.4 Idempotency

Speed layer duplicate event পেতে পারে।
Batch layer-এও replay হতে পারে।
তাই deterministic idempotent update দরকার।

Example techniques:

- dedupe by event_id
- commutative counters
- upsert with version check
- watermark-based close window
- exactly-once transaction যেখানে supported

#### 12.5 Observability

Lambda system monitor করার জন্য শুধু CPU বা memory যথেষ্ট নয়।
Need metrics like:

- batch lag
- speed lag
- merge latency
- event duplication rate
- late event count
- view freshness
- batch vs speed discrepancy
- replay duration
- serving error rate

### 13. Modern Alternatives

আজকের data ecosystem-এ কিছু technology Lambda vs Kappa debate-কে নতুনভাবে shape করেছে।

#### Unified Batch + Stream

- Apache Beam
- Spark Structured Streaming
- Flink unified APIs

এগুলো code reuse কিছুটা বাড়ায়।
একই API দিয়ে bounded এবং unbounded data handle করা যায়।

#### Data Lakehouse

- Delta Lake
- Apache Iceberg
- Apache Hudi

এগুলো append-only raw data, ACID table, incremental read, streaming ingestion—সব এক storage plane-এ আনতে সাহায্য করে।
ফলে classic Lambda-এর batch/speed split কিছু ক্ষেত্রে softer হয়ে যায়।

#### Materialized Streaming Views

- ksqlDB
- Materialize
- Apache Pinot
- ClickHouse with materialized views

এগুলো streaming-first serving model দেয়।
কিছু use case-এ pure Kappa বা hybrid architecture অনেক সহজ হয়ে যায়।

তবুও conceptual lesson একই থাকে:

- raw immutable fact save করো
- derived view disposable ভাবো
- correctness boundary explicit করো
- latency এবং accuracy trade-off পরিষ্কার রাখো

## 🌍 Real-world examples

এই section-এ Lambda এবং Kappa চিন্তাকে বাংলাদেশি product ও platform context-এ map করা হলো।

### bKash Lambda Architecture

- Core problem: কোটি কোটি transaction-এর মধ্যে live fraud alert এবং end-of-day exact reconciliation—দুইটাই দরকার।
- Batch Layer: S3/HDFS-এ append-only transaction, reversal, fee, settlement event store করা হলো।
- Batch job: Spark nightly run করে exact ledger, agent settlement, merchant settlement, fee breakdown recompute করলো।
- Speed Layer: Kafka Streams/Flink suspicious velocity, geo anomaly, device anomaly detect করলো।
- Serving Layer: operations dashboard batch accuracy + live anomaly count merge করে দেখালো।
- Why Lambda: বাংলাদেশি fintech domain-এ audit, dispute, reconciliation, compliance অত্যন্ত critical।
- Why not pure Kappa only: multi-year replay on every logic change expensive, আর regulatory batch close-off দরকার।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

### Daraz Analytics

- Core problem: seller analytics, historical GMV, live trending product, campaign response—সব একসাথে দরকার।
- Batch Layer: nightly Spark jobs category-wise revenue, refund-adjusted GMV, repeat buyer segments compute করলো।
- Speed Layer: live clickstream, add-to-cart, order placed event থেকে trending score update করলো।
- Serving Layer: seller dashboard historical charts + last 10 minute trend merge করলো।
- Extra twist: Ramadan campaign-এ live demand spike speed layer সামলায়, monthly finance report batch layer সামলায়।
- Lambda benefit: marketing team freshness পায়, finance team exactness পায়।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

### GP Telecom

- Core problem: CDR processing, monthly subscriber analytics, network outage alert, live usage monitoring।
- Batch Layer: petabyte-scale CDR lake থেকে subscriber behavior, ARPU, churn feature build করা।
- Speed Layer: cell tower incident, abnormal drop rate, current usage spike detect করা।
- Serving Layer: customer 360 portal-এ historical profile + live incident status দেখানো।
- Why Lambda: telecom data volume বিশাল, replay-only strategy সবসময় practical নয়।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

### Pathao এবং Foodpanda

- Core problem: rider heatmap, current ETA, live surge, historical supply-demand analysis।
- Batch Layer: demand by hour, zone profitability, rider retention model build করা।
- Speed Layer: rider location stream, order assignment stream, cancellation spike stream process করা।
- Serving Layer: dispatch console historical baseline + current live queue merge করে decision support দেয়।
- Kappa possibility: যদি business logic mostly stream-friendly হয়, এখানে Kappa ভালো candidate হতে পারে।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

### Prothom Alo Trending and Editorial Insights

- Core problem: এখন কোন খবর trend করছে এবং গত ৬ মাসে কোন category long-term traffic আনছে—দুই উত্তরই দরকার।
- Batch Layer: article, referrer, scroll depth, subscription conversion historical analysis।
- Speed Layer: breaking news publish হওয়ার পর প্রতি minute trend score update।
- Serving Layer: newsroom dashboard-এ current trend + historical benchmark।
- Why Kappa may fit too: trending logic streaming-friendly, replay manageable হলে simpler architecture সম্ভব।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

### Chaldal Grocery Operations

- Core problem: live order volume, slot utilization, warehouse pick efficiency, historical demand forecasting।
- Batch Layer: daily replenishment planning, SKU-level demand forecast, city-wise margin analysis।
- Speed Layer: live basket addition, stock reservation, failed delivery signal।
- Serving Layer: ops dashboard batch forecast + live fulfillment status merge করে।
- Takeaway: grocery operations-এ live control plane এবং slow planning plane—দুইটাই দরকার।

#### প্রশ্ন যা business জিজ্ঞেস করে

- historical exact answer কী?
- এখন live অবস্থাটা কী?
- late event এলে correction কীভাবে হবে?
- compliance বা reconciliation কোন layer validate করবে?
- product team-এর dashboard কোন freshness SLA পাবে?

## 🗺️ ASCII Diagrams

### 3.1 Full Lambda Architecture

```text
                                   ┌──────────────────────────────┐
                                   │         DATA SOURCES          │
                                   │------------------------------│
                                   │ bKash App / Daraz / Pathao   │
                                   │ Web Clicks / API / Mobile    │
                                   │ POS / Payment / GPS / Logs   │
                                   └──────────────┬───────────────┘
                                                  │
                                                  │ Same immutable events
                                                  ▼
                         ┌────────────────────────────────────────────────────┐
                         │              INGESTION / EVENT LOG                 │
                         │----------------------------------------------------│
                         │ Kafka / Kinesis / Append-only log / Object store   │
                         └──────────────┬───────────────────────┬─────────────┘
                                        │                       │
                                        │                       │
                         ┌──────────────▼─────────────┐   ┌─────▼────────────────────┐
                         │        BATCH LAYER         │   │       SPEED LAYER        │
                         │----------------------------│   │---------------------------│
                         │ Store ALL raw data         │   │ Process ONLY recent data │
                         │ Recompute from beginning   │   │ Incremental aggregation   │
                         │ High latency               │   │ Low latency               │
                         │ Highest accuracy           │   │ Approx / temporary        │
                         └──────────────┬─────────────┘   └─────┬────────────────────┘
                                        │                       │
                                        ▼                       ▼
                         ┌──────────────────────────┐   ┌───────────────────────────┐
                         │       BATCH VIEWS        │   │      REAL-TIME VIEWS      │
                         │--------------------------│   │---------------------------│
                         │ Daily revenue tables     │   │ Live counters             │
                         │ Exact ledger snapshots   │   │ Recent anomaly scores     │
                         │ Historical aggregates    │   │ Last-minute trends        │
                         └──────────────┬───────────┘   └───────────┬───────────────┘
                                        │                           │
                                        └──────────────┬────────────┘
                                                       │
                                                       ▼
                                 ┌────────────────────────────────────────────┐
                                 │               SERVING LAYER                │
                                 │--------------------------------------------│
                                 │ Merge(batch_view, realtime_view)           │
                                 │ Index + Cache + Query API                  │
                                 │ Boundary aware result composition          │
                                 └─────────────────┬──────────────────────────┘
                                                   │
                                                   ▼
                                 ┌────────────────────────────────────────────┐
                                 │                QUERY RESULTS               │
                                 │--------------------------------------------│
                                 │ Dashboard / Fraud Alert / Seller Report    │
                                 │ Customer 360 / Trending / Ops Console      │
                                 └────────────────────────────────────────────┘
```

### 3.2 Data Flow Diagram — Same Data Goes to Both Layers

```text
        Event Produced Once
               │
               ▼
      ┌─────────────────┐
      │  app event bus  │
      │  or Kafka topic │
      └────────┬────────┘
               │
     ┌─────────┴─────────┐
     │                   │
     ▼                   ▼
┌───────────────┐   ┌───────────────┐
│ batch ingest  │   │ speed consume │
│---------------│   │---------------│
│ raw lake      │   │ live state     │
│ append only   │   │ incremental    │
└──────┬────────┘   └──────┬────────┘
       │                   │
       ▼                   ▼
┌───────────────┐   ┌───────────────┐
│ spark/hadoop  │   │ flink/kstreams│
│ full recompute│   │ delta update  │
└──────┬────────┘   └──────┬────────┘
       │                   │
       ▼                   ▼
┌───────────────┐   ┌───────────────┐
│ batch view v7 │   │ rt view now   │
└──────┬────────┘   └──────┬────────┘
       └──────────┬────────┘
                  ▼
          ┌───────────────┐
          │ serving merge │
          └───────────────┘
```

Key lesson:

- write once
- fan out internally
- never dual-write from app to two truths

### 3.3 Timeline — Batch Recomputation vs Real-time Updates

```text
Time ─────────────────────────────────────────────────────────────────────────▶

Raw Events:
E1  E2  E3  E4  E5  E6  E7  E8  E9  E10 E11 E12 E13 E14 E15 ...
│   │   │   │   │   │   │   │   │   │   │   │   │   │

Batch Run #1 starts scanning all data ------------------------┐
                                                              │
Batch Run #1 finishes at T1 ----------------------------------┘
Batch View covers [0, T1]

After T1, speed layer covers:
                         (T1, now]
                          │  │  │  │  │  │
                          E9 E10 E11 E12 E13 E14

Next batch run #2 recomputes all/affected partitions ------------------------┐
                                                                             │
Batch Run #2 finishes at T2 -------------------------------------------------┘
Batch View now covers [0, T2]

Speed layer discards old covered deltas and now only tracks:
                                                (T2, now]
                                                 │  │  │
                                                E15 E16 E17

Interpretation:
- speed layer is always chasing the gap after latest batch cutoff
- batch layer eventually absorbs historical truth
- serving layer always knows current boundary T
```

### 3.4 Query Merging at Serving Layer

```text
User Query: "আজ পর্যন্ত seller 123-এর revenue কত?"

                           ┌─────────────────────────┐
                           │   Serving Layer Query   │
                           └────────────┬────────────┘
                                        │
                 ┌──────────────────────┼──────────────────────┐
                 │                      │                      │
                 ▼                      ▼                      ▼
      ┌──────────────────┐   ┌────────────────────┐   ┌─────────────────┐
      │ Metadata Store   │   │ Batch View Store   │   │ Speed View Store│
      │------------------│   │--------------------│   │-----------------│
      │ latest_batch=T   │   │ seller_total_uptoT │   │ seller_delta_T+ │
      └────────┬─────────┘   └──────────┬─────────┘   └────────┬────────┘
               │                        │                      │
               ▼                        ▼                      ▼
         boundary = T          baseline = 12,40,000      live = 18,500
               │                        │                      │
               └────────────────────────┼──────────────────────┘
                                        ▼
                               final = baseline + live
                                        ▼
                             12,58,500 returned to user
```

Potential risks:

- wrong `T` means overlap or missing gap
- stale speed state means low live accuracy
- non-atomic batch publish means mixed-version results

### 3.5 Kappa Architecture Diagram

```text
                           ┌────────────────────────────┐
                           │        DATA SOURCES         │
                           └─────────────┬──────────────┘
                                         │
                                         ▼
                           ┌────────────────────────────┐
                           │     EVENT LOG / KAFKA      │
                           │----------------------------│
                           │ long retention / replay    │
                           └─────────────┬──────────────┘
                                         │
                                         ▼
                           ┌────────────────────────────┐
                           │      STREAM PROCESSOR      │
                           │----------------------------│
                           │ Kafka Streams / Flink      │
                           │ same code for live+replay  │
                           └─────────────┬──────────────┘
                                         │
                                         ▼
                           ┌────────────────────────────┐
                           │     MATERIALIZED VIEWS     │
                           │----------------------------│
                           │ Redis / RocksDB / Druid    │
                           └─────────────┬──────────────┘
                                         │
                                         ▼
                           ┌────────────────────────────┐
                           │          QUERIES           │
                           └────────────────────────────┘

Need recomputation?

Deploy new processor version
         │
         ▼
Replay Kafka from offset 0 / chosen checkpoint
         │
         ▼
Rebuild views using same streaming logic
```

### 3.6 Lambda vs Kappa Conceptual Shape

```text
Lambda:

             ┌─────────── Batch Compute ────────────┐
Raw Data ───▶│ Accurate, slow, full recomputation   │──┐
             └──────────────────────────────────────┘  │
                                                        ├──▶ Merge ─▶ Query
             ┌────────── Speed Compute ─────────────┐  │
Raw Data ───▶│ Fast, recent only, incremental       │──┘
             └──────────────────────────────────────┘

Kappa:

Raw Data ───▶ Durable Event Log ───▶ Single Stream Compute ───▶ Query
                                      ▲
                                      │ replay for rebuild
                                      └──────────────────────────────
```

### 3.7 Atomic Batch View Swap

```text
Old world serving traffic
        │
        ▼
┌────────────────────┐
│ batch_view_v41     │  <── live readers still use this
└────────────────────┘

New batch computation finished
        │
        ▼
┌────────────────────┐
│ batch_view_v42_tmp │  <── validate counts/checksum/partitions
└────────────────────┘
        │
        │ atomic metadata update
        ▼
┌──────────────────────────────────────────────┐
│ serving_metadata.current_version = v42       │
│ serving_metadata.batch_end = 2025-05-22T02:00│
└──────────────────────────────────────────────┘
        │
        ▼
New readers switch to v42 immediately
Old readers finish naturally
Later cleanup drops v41
```

### 3.8 Schema Evolution over Time

```text
Version 1 event:
OrderPlaced {
  order_id,
  customer_id,
  total_amount,
  occurred_at
}

Version 2 event:
OrderPlaced {
  order_id,
  customer_id,
  total_amount,
  discount_amount = 0,
  currency = "BDT",
  occurred_at,
  schema_version = 2
}

Replay strategy:
- old events without discount_amount use default 0
- new deserializer understands both versions
- batch recomputation and speed processing both stay compatible
```

## 💻 PHP Code

এই section-এ PHP উদাহরণগুলো framework-agnostic style-এ লেখা হয়েছে, তবে Laravel project-এ সহজেই adapt করা যাবে।

### PHP Example 1 — Domain Event এবং Master Dataset Model

```php
<?php

declare(strict_types=1);

final readonly class TransactionEvent
{
    public function __construct(
        public string $eventId,
        public string $eventType,
        public string $accountId,
        public int $amount,
        public DateTimeImmutable $occurredAt,
        public array $payload = [],
        public int $schemaVersion = 1,
    ) {}

    public static function fromArray(array $row): self
    {
        return new self(
            eventId: (string) $row['event_id'],
            eventType: (string) $row['event_type'],
            accountId: (string) $row['account_id'],
            amount: (int) $row['amount'],
            occurredAt: new DateTimeImmutable((string) $row['occurred_at']),
            payload: $row['payload'] ?? [],
            schemaVersion: (int) ($row['schema_version'] ?? 1),
        );
    }

    public function isCredit(): bool
    {
        return in_array($this->eventType, ['money_received', 'cash_in', 'refund_received'], true);
    }

    public function signedAmount(): int
    {
        return $this->isCredit() ? $this->amount : -$this->amount;
    }
}

interface MasterDatasetRepository
{
    /** @return iterable<TransactionEvent> */
    public function allEventsUntil(DateTimeImmutable $until): iterable;

    /** @return iterable<TransactionEvent> */
    public function eventsBetween(DateTimeImmutable $from, DateTimeImmutable $to): iterable;
}

final class InMemoryMasterDatasetRepository implements MasterDatasetRepository
{
    /** @param list<TransactionEvent> $events */
    public function __construct(private array $events) {}

    public function allEventsUntil(DateTimeImmutable $until): iterable
    {
        foreach ($this->events as $event) {
            if ($event->occurredAt <= $until) {
                yield $event;
            }
        }
    }

    public function eventsBetween(DateTimeImmutable $from, DateTimeImmutable $to): iterable
    {
        foreach ($this->events as $event) {
            if ($event->occurredAt > $from && $event->occurredAt <= $to) {
                yield $event;
            }
        }
    }
}
```

Explanation:

- `TransactionEvent` হলো immutable fact
- `signedAmount()` credit/debit merge সহজ করে
- real master dataset HDFS, S3, Kafka archival topic, বা append-only table হতে পারে
- replay-friendly model হিসেবে event timestamp এবং schema version রাখা হয়েছে

### PHP Example 2 — Batch Layer Processor (সব data reprocess)

```php
<?php

declare(strict_types=1);

final readonly class BatchViewRow
{
    public function __construct(
        public string $accountId,
        public int $balance,
        public int $transactionCount,
        public DateTimeImmutable $batchCutoff,
    ) {}
}

interface BatchViewStore
{
    public function beginVersion(string $version, DateTimeImmutable $cutoff): void;
    public function put(string $version, BatchViewRow $row): void;
    public function publish(string $version, DateTimeImmutable $cutoff): void;
}

final class BatchProcessor
{
    public function __construct(
        private MasterDatasetRepository $dataset,
        private BatchViewStore $store,
    ) {}

    public function recompute(DateTimeImmutable $cutoff): string
    {
        $version = 'batch_' . $cutoff->format('Ymd_His');
        $this->store->beginVersion($version, $cutoff);

        $balances = [];
        $counts = [];

        foreach ($this->dataset->allEventsUntil($cutoff) as $event) {
            $balances[$event->accountId] = ($balances[$event->accountId] ?? 0) + $event->signedAmount();
            $counts[$event->accountId] = ($counts[$event->accountId] ?? 0) + 1;
        }

        foreach ($balances as $accountId => $balance) {
            $this->store->put(
                $version,
                new BatchViewRow(
                    accountId: $accountId,
                    balance: $balance,
                    transactionCount: $counts[$accountId] ?? 0,
                    batchCutoff: $cutoff,
                )
            );
        }

        $this->store->publish($version, $cutoff);

        return $version;
    }
}
```

এটি classic Lambda batch logic:

- cutoff পর্যন্ত সব event scan করছে
- derived state নতুন version-এ লিখছে
- পরে atomic publish করছে
- পুরনো view corrupt হলে full recompute possible

### PHP Example 3 — Speed Layer Consumer (recent events only)

```php
<?php

declare(strict_types=1);

interface RealTimeViewStore
{
    public function incrementBalanceDelta(string $accountId, int $amount): void;
    public function incrementTransactionDelta(string $accountId): void;
    public function markSeen(string $eventId): bool;
    public function resetOlderThan(DateTimeImmutable $cutoff): void;
    public function getDelta(string $accountId): array;
}

final class SpeedLayerConsumer
{
    public function __construct(
        private RealTimeViewStore $realTimeViewStore,
        private DateTimeImmutable $currentBatchCutoff,
    ) {}

    public function onEvent(TransactionEvent $event): void
    {
        if ($event->occurredAt <= $this->currentBatchCutoff) {
            return;
        }

        if (! $this->realTimeViewStore->markSeen($event->eventId)) {
            return;
        }

        $this->realTimeViewStore->incrementBalanceDelta($event->accountId, $event->signedAmount());
        $this->realTimeViewStore->incrementTransactionDelta($event->accountId);
    }

    public function onBatchPublished(DateTimeImmutable $newCutoff): void
    {
        $this->currentBatchCutoff = $newCutoff;
        $this->realTimeViewStore->resetOlderThan($newCutoff);
    }
}
```

এখানে important points:

- speed layer only post-batch events process করছে
- duplicate event প্রতিরোধে `markSeen()` idempotency guard
- batch publish হলে outdated delta discard করা হচ্ছে

### PHP Example 4 — Serving Layer Query Merger

```php
<?php

declare(strict_types=1);

final readonly class BatchMetadata
{
    public function __construct(
        public string $version,
        public DateTimeImmutable $cutoff,
    ) {}
}

interface BatchMetadataRepository
{
    public function latest(): BatchMetadata;
}

interface PublishedBatchQueryStore
{
    public function getAccountRow(string $version, string $accountId): ?BatchViewRow;
}

final readonly class AccountSummary
{
    public function __construct(
        public string $accountId,
        public int $balance,
        public int $transactionCount,
        public DateTimeImmutable $authoritativeUntil,
    ) {}
}

final class AccountSummaryService
{
    public function __construct(
        private BatchMetadataRepository $metadataRepository,
        private PublishedBatchQueryStore $batchQueryStore,
        private RealTimeViewStore $realTimeViewStore,
    ) {}

    public function getSummary(string $accountId): AccountSummary
    {
        $metadata = $this->metadataRepository->latest();
        $batchRow = $this->batchQueryStore->getAccountRow($metadata->version, $accountId);
        $delta = $this->realTimeViewStore->getDelta($accountId);

        $batchBalance = $batchRow?->balance ?? 0;
        $batchCount = $batchRow?->transactionCount ?? 0;
        $deltaBalance = (int) ($delta['balance_delta'] ?? 0);
        $deltaCount = (int) ($delta['tx_delta'] ?? 0);

        return new AccountSummary(
            accountId: $accountId,
            balance: $batchBalance + $deltaBalance,
            transactionCount: $batchCount + $deltaCount,
            authoritativeUntil: $metadata->cutoff,
        );
    }
}
```

Serving layer merge formula এখানে explicit:

- baseline = batch row
- delta = real-time row
- final = baseline + delta

Dashboard response-এ `authoritativeUntil` expose করলে user জানে exact batch coverage কোথায় শেষ।

### PHP Example 5 — MySQL Batch View + Redis Speed View Pipeline

```php
<?php

declare(strict_types=1);

use PDO;
use Redis;

final class MysqlBatchViewStore implements BatchViewStore, PublishedBatchQueryStore, BatchMetadataRepository
{
    public function __construct(private PDO $pdo) {}

    public function beginVersion(string $version, DateTimeImmutable $cutoff): void
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO batch_versions(version, cutoff_at, state) VALUES(:version, :cutoff, :state)'
        );

        $stmt->execute([
            'version' => $version,
            'cutoff' => $cutoff->format('Y-m-d H:i:s'),
            'state' => 'building',
        ]);
    }

    public function put(string $version, BatchViewRow $row): void
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO batch_account_summaries(version, account_id, balance, transaction_count)
             VALUES(:version, :account_id, :balance, :transaction_count)'
        );

        $stmt->execute([
            'version' => $version,
            'account_id' => $row->accountId,
            'balance' => $row->balance,
            'transaction_count' => $row->transactionCount,
        ]);
    }

    public function publish(string $version, DateTimeImmutable $cutoff): void
    {
        $this->pdo->beginTransaction();

        $this->pdo->exec("UPDATE batch_versions SET state = 'archived' WHERE state = 'published'");

        $stmt = $this->pdo->prepare(
            "UPDATE batch_versions SET state = 'published', cutoff_at = :cutoff WHERE version = :version"
        );

        $stmt->execute([
            'version' => $version,
            'cutoff' => $cutoff->format('Y-m-d H:i:s'),
        ]);

        $this->pdo->commit();
    }

    public function latest(): BatchMetadata
    {
        $row = $this->pdo->query(
            "SELECT version, cutoff_at FROM batch_versions WHERE state = 'published' LIMIT 1"
        )->fetch(PDO::FETCH_ASSOC);

        return new BatchMetadata(
            version: (string) $row['version'],
            cutoff: new DateTimeImmutable((string) $row['cutoff_at']),
        );
    }

    public function getAccountRow(string $version, string $accountId): ?BatchViewRow
    {
        $stmt = $this->pdo->prepare(
            'SELECT account_id, balance, transaction_count FROM batch_account_summaries
             WHERE version = :version AND account_id = :account_id LIMIT 1'
        );
        $stmt->execute(['version' => $version, 'account_id' => $accountId]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        if (! $row) {
            return null;
        }

        return new BatchViewRow(
            accountId: (string) $row['account_id'],
            balance: (int) $row['balance'],
            transactionCount: (int) $row['transaction_count'],
            batchCutoff: $this->latest()->cutoff,
        );
    }
}

final class RedisRealTimeViewStore implements RealTimeViewStore
{
    public function __construct(private Redis $redis) {}

    public function incrementBalanceDelta(string $accountId, int $amount): void
    {
        $this->redis->hIncrBy("rt:account:{$accountId}", 'balance_delta', $amount);
    }

    public function incrementTransactionDelta(string $accountId): void
    {
        $this->redis->hIncrBy("rt:account:{$accountId}", 'tx_delta', 1);
    }

    public function markSeen(string $eventId): bool
    {
        return (bool) $this->redis->set("seen:{$eventId}", '1', ['nx', 'ex' => 86400]);
    }

    public function resetOlderThan(DateTimeImmutable $cutoff): void
    {
        $this->redis->set('rt:last_cutoff', $cutoff->format(DateTimeInterface::ATOM));
    }

    public function getDelta(string $accountId): array
    {
        return $this->redis->hGetAll("rt:account:{$accountId}");
    }
}
```

এই pattern production-ready concept দেখায়:

- MySQL-এ versioned batch views
- Redis-এ fast deltas
- metadata table দিয়ে published version control
- serving API সহজে two-store merge করতে পারে

### PHP Example 6 — Laravel-style Controller Response

```php
<?php

declare(strict_types=1);

use Illuminate\Http\JsonResponse;
use Illuminate\Routing\Controller;

final class AccountAnalyticsController extends Controller
{
    public function __construct(private AccountSummaryService $summaryService) {}

    public function show(string $accountId): JsonResponse
    {
        $summary = $this->summaryService->getSummary($accountId);

        return response()->json([
            'account_id' => $summary->accountId,
            'balance' => $summary->balance,
            'transaction_count' => $summary->transactionCount,
            'authoritative_until' => $summary->authoritativeUntil->format(DateTimeInterface::ATOM),
            'note' => 'Balance = batch baseline + live delta',
        ]);
    }
}
```

বাংলাদেশি উদাহরণে এই endpoint bKash ops dashboard, Daraz seller center, বা GP internal analytics portal-এ ব্যবহার করা যেতে পারে।

## 🟨 JS Code

এই section-এ Node.js ES2022+ style-এ Lambda/Kappa-friendly implementation দেখানো হলো।

### JavaScript Example 1 — KafkaJS Speed Layer Consumer

```js
import { Kafka } from 'kafkajs';
import Redis from 'ioredis';

const kafka = new Kafka({
  clientId: 'bkash-fraud-speed-layer',
  brokers: ['localhost:9092'],
});

const consumer = kafka.consumer({ groupId: 'fraud-live-group' });
const redis = new Redis();

let currentBatchCutoff = new Date('2025-05-22T02:00:00Z');

async function markSeen(eventId) {
  const result = await redis.set(`seen:${eventId}`, '1', 'EX', 86400, 'NX');
  return result === 'OK';
}

async function processFraudSignal(event) {
  const accountKey = `fraud:${event.accountId}`;
  await redis.hincrby(accountKey, 'txn_count', 1);
  await redis.hincrby(accountKey, 'amount_total', Number(event.amount));

  if (event.deviceId) {
    await redis.sadd(`${accountKey}:devices`, event.deviceId);
  }

  const [txnCount, amountTotal, distinctDevices] = await Promise.all([
    redis.hget(accountKey, 'txn_count'),
    redis.hget(accountKey, 'amount_total'),
    redis.scard(`${accountKey}:devices`),
  ]);

  const suspicious = Number(txnCount) >= 5 && Number(amountTotal) >= 50000 && distinctDevices >= 3;

  if (suspicious) {
    await redis.hset(accountKey, 'alert', 'possible_velocity_fraud');
  }
}

await consumer.connect();
await consumer.subscribe({ topic: 'transactions', fromBeginning: false });

await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    const occurredAt = new Date(event.occurredAt);

    if (occurredAt <= currentBatchCutoff) {
      return;
    }

    if (!(await markSeen(event.eventId))) {
      return;
    }

    await processFraudSignal(event);
  },
});
```

What this shows:

- Kafka topic is the single ingestion point
- speed layer consumes only post-batch events
- Redis stores temporary incremental state
- duplicate handling uses `NX` set

### JavaScript Example 2 — Batch Processor with Worker Threads

```js
import { Worker } from 'node:worker_threads';
import os from 'node:os';

function runWorker(chunk) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      new URL('./workers/batch-account-worker.js', import.meta.url),
      { workerData: chunk }
    );

    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

function chunkArray(items, size) {
  const chunks = [];
  for (let i = 0; i < items.length; i += size) {
    chunks.push(items.slice(i, i + size));
  }
  return chunks;
}

export async function recomputeBatchViews(allEvents) {
  const cpuCount = Math.max(1, os.cpus().length - 1);
  const chunks = chunkArray(allEvents, Math.ceil(allEvents.length / cpuCount));
  const partials = await Promise.all(chunks.map(runWorker));

  const merged = new Map();

  for (const partial of partials) {
    for (const row of partial) {
      const current = merged.get(row.accountId) ?? { balance: 0, txCount: 0 };
      current.balance += row.balance;
      current.txCount += row.txCount;
      merged.set(row.accountId, current);
    }
  }

  return [...merged.entries()].map(([accountId, value]) => ({
    accountId,
    balance: value.balance,
    txCount: value.txCount,
  }));
}
```

Worker file:

```js
import { parentPort, workerData } from 'node:worker_threads';

const partial = new Map();

for (const event of workerData) {
  const signedAmount = ['money_received', 'cash_in', 'refund_received'].includes(event.eventType)
    ? Number(event.amount)
    : -Number(event.amount);

  const current = partial.get(event.accountId) ?? { balance: 0, txCount: 0 };
  current.balance += signedAmount;
  current.txCount += 1;
  partial.set(event.accountId, current);
}

parentPort.postMessage(
  [...partial.entries()].map(([accountId, value]) => ({ accountId, ...value }))
);
```

এই batch job পুরো historical dataset-এর উপর parallel recomputation simulate করছে।

### JavaScript Example 3 — Serving Layer API that Merges Batch + Real-time

```js
import express from 'express';
import Redis from 'ioredis';
import mysql from 'mysql2/promise';

const app = express();
const redis = new Redis();
const db = await mysql.createPool({
  host: '127.0.0.1',
  user: 'root',
  database: 'lambda_demo',
  waitForConnections: true,
  connectionLimit: 10,
});

async function getLatestBatchMetadata() {
  const [rows] = await db.query(
    "SELECT version, cutoff_at FROM batch_versions WHERE state = 'published' LIMIT 1"
  );
  return rows[0];
}

async function getBatchRow(version, accountId) {
  const [rows] = await db.query(
    `SELECT balance, transaction_count
     FROM batch_account_summaries
     WHERE version = ? AND account_id = ? LIMIT 1`,
    [version, accountId]
  );
  return rows[0] ?? { balance: 0, transaction_count: 0 };
}

async function getRealtimeDelta(accountId) {
  const delta = await redis.hgetall(`rt:account:${accountId}`);
  return {
    balanceDelta: Number(delta.balance_delta ?? 0),
    txDelta: Number(delta.tx_delta ?? 0),
  };
}

app.get('/accounts/:accountId/summary', async (req, res) => {
  const { accountId } = req.params;
  const metadata = await getLatestBatchMetadata();
  const batchRow = await getBatchRow(metadata.version, accountId);
  const delta = await getRealtimeDelta(accountId);

  res.json({
    accountId,
    balance: Number(batchRow.balance) + delta.balanceDelta,
    transactionCount: Number(batchRow.transaction_count) + delta.txDelta,
    authoritativeUntil: metadata.cutoff_at,
    formula: 'result = batch_view + realtime_delta',
  });
});

app.listen(3000, () => {
  console.log('Serving layer listening on port 3000');
});
```

API-এর responsibility এখানে business semantics explain করা:

- baseline কোথা থেকে এলো
- live delta কী
- authoritative coverage কোথায় শেষ

### JavaScript Example 4 — Materialized View Updater for Daraz-style Trending Products

```js
import Redis from 'ioredis';

const redis = new Redis();
const TRENDING_KEY = 'daraz:trending:products';

export async function handleClickEvent(event) {
  const minuteBucket = event.occurredAt.slice(0, 16);
  const perMinuteKey = `clicks:${minuteBucket}`;

  await redis.hincrby(perMinuteKey, event.productId, 1);
  await redis.expire(perMinuteKey, 60 * 60 * 6);

  const score = await calculateTrendScore(event.productId);
  await redis.zadd(TRENDING_KEY, score, event.productId);
}

async function calculateTrendScore(productId) {
  const now = new Date();
  let weightedScore = 0;

  for (let i = 0; i < 15; i += 1) {
    const bucketTime = new Date(now.getTime() - i * 60_000);
    const bucket = bucketTime.toISOString().slice(0, 16);
    const count = Number(await redis.hget(`clicks:${bucket}`, productId) ?? 0);
    const weight = 15 - i;
    weightedScore += count * weight;
  }

  return weightedScore;
}

export async function getTopTrending(limit = 10) {
  return redis.zrevrange(TRENDING_KEY, 0, limit - 1, 'WITHSCORES');
}
```

এটি speed layer-এর ideal example:

- recent events only
- sliding window computation
- approximate but useful live ranking
- next batch historical trend rebuild করলে correction possible

### JavaScript Example 5 — Dashboard Backend Querying Both Layers

```js
import express from 'express';
import Redis from 'ioredis';
import mysql from 'mysql2/promise';

const app = express();
const redis = new Redis();
const db = await mysql.createPool({ host: '127.0.0.1', user: 'root', database: 'analytics' });

async function getHistoricalSellerMetrics(sellerId) {
  const [rows] = await db.query(
    `SELECT total_revenue, total_orders, last_batch_cutoff
     FROM seller_batch_metrics
     WHERE seller_id = ? LIMIT 1`,
    [sellerId]
  );
  return rows[0] ?? { total_revenue: 0, total_orders: 0, last_batch_cutoff: null };
}

async function getLiveSellerMetrics(sellerId) {
  const live = await redis.hgetall(`seller:live:${sellerId}`);
  return {
    liveRevenue: Number(live.revenue ?? 0),
    liveOrders: Number(live.orders ?? 0),
    liveVisitors: Number(live.visitors ?? 0),
  };
}

app.get('/seller/:sellerId/dashboard', async (req, res) => {
  const { sellerId } = req.params;
  const historical = await getHistoricalSellerMetrics(sellerId);
  const live = await getLiveSellerMetrics(sellerId);

  res.json({
    sellerId,
    revenue: Number(historical.total_revenue) + live.liveRevenue,
    orders: Number(historical.total_orders) + live.liveOrders,
    liveVisitors: live.liveVisitors,
    historicalCutoff: historical.last_batch_cutoff,
    note: 'Historical metrics come from batch; live counters come from speed layer.',
  });
});

app.listen(4000, () => {
  console.log('Daraz-style dashboard backend is running');
});
```

বাংলাদেশ context-এ এই pattern seller center, newsroom console, courier ops dashboard—সবখানেই ব্যবহার করা যায়।

### JavaScript Example 6 — Kappa-style Replay Processor

```js
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ clientId: 'kappa-replay', brokers: ['localhost:9092'] });

export async function replayTopic({ topic, fromBeginning = true, handleEvent }) {
  const consumer = kafka.consumer({ groupId: `replay-${Date.now()}` });
  await consumer.connect();
  await consumer.subscribe({ topic, fromBeginning });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      await handleEvent(event);
    },
  });

  return consumer;
}

// Example usage:
// replayTopic({
//   topic: 'article-events',
//   handleEvent: async (event) => rebuildTrendingView(event),
// });
```

Kappa lesson:

- same handler can process live events
- same handler can rebuild from offset zero
- no separate batch code path needed

## ✅ When to use

### Choose Lambda when:

- dataset MASSIVE এবং full replay practical নয়
- batch computation fundamentally different, যেমন heavy ML training, ledger reconciliation, billing close
- regulatory বা audit requirement আছে
- late-arriving data regular occurrence
- business approximate live preview + exact final report—দুইটাই অফিসিয়ালি চায়
- team already Spark/Hadoop batch ecosystem-এ invested
- Bangladesh example: bKash core ledger reconciliation, GP large-scale CDR analytics, Daraz finance settlement

### Choose Kappa when:

- business logic mostly streaming-friendly
- entire log reasonable time-এ replay করা যায়
- team streaming systems-এ strong
- operational simplicity বেশি গুরুত্বপূর্ণ
- one codebase maintain করতে চাও
- Bangladesh example: Pathao live dispatch analytics, Prothom Alo trending stories, Foodpanda real-time ops telemetry

### Do not choose Lambda blindly when:

- system small কিন্তু team unnecessarily “big data architecture” সাজাতে চাইছে
- real-time এবং historical logic essentially একই, তবু duplicate pipelines বানানো হচ্ছে
- organization-এর ops maturity কম
- batch cutoff coordination maintain করার মতো discipline নেই
- actual problem OLTP query tuning দিয়ে solve হয়ে যেত

### Simple decision heuristic

```text
Need exact historical truth + live preview + large-scale recomputation?
    └── Yes → Lambda ভাবুন

Need one stream-first code path + affordable replay + lower ops complexity?
    └── Yes → Kappa ভাবুন

Need unified engine with modern tooling?
    └── Beam / Structured Streaming / Lakehouse hybrid evaluate করুন
```

### Bangladesh-focused recommendations

- **bKash / Nagad / ব্যাংকিং** → Lambda বা hybrid Lambda বেশি natural
- **Daraz / Chaldal marketplace analytics** → mixed; finance/reporting অংশে Lambda, trend অংশে Kappa/hybrid
- **Pathao / Foodpanda dispatch telemetry** → Kappa-friendly
- **Prothom Alo / news analytics** → trending use case Kappa-friendly, subscription revenue analytics Lambda-friendly
- **GP / telecom subscriber + CDR** → Lambda strong candidate

## 🔑 Key Takeaways

- Lambda Architecture accuracy এবং latency—দুই দুনিয়াকে একসাথে ধরার চেষ্টা করে
- Batch layer হলো authoritative historical truth engine
- Speed layer হলো low-latency temporary delta engine
- Serving layer boundary-aware merge না করলে Lambda ভেঙে পড়ে
- Master dataset immutable হলে replay, audit, correction—সব সহজ হয়
- Lambda-এর biggest strength = recoverability and correctness
- Lambda-এর biggest weakness = code duplication and operational complexity
- Kappa Architecture duplication কমায়, কিন্তু replay feasibility-এর উপর নির্ভর করে
- সব real-time system-এ Lambda দরকার নেই
- সব regulated system-এ Kappa enough নাও হতে পারে
- Bangladesh fintech, telecom, large marketplace contexts-এ Lambda এখনও খুবই practical
- Trending, live telemetry, session analytics-এর মতো stream-native problem-এ Kappa চমৎকার fit হতে পারে

### Interview-ready one-liners

- Lambda = batch truth + speed delta + serving merge
- Kappa = single stream code path + replay for rebuild
- If you can replay cheaply, Kappa becomes attractive
- If you need guaranteed batch reconciliation, Lambda stays relevant
- Immutability is not storage dogma; it is recovery strategy

### Common mistakes to avoid

- application থেকে dual write করা
- batch cutoff metadata expose না করা
- late event policy স্পষ্ট না করা
- batch এবং speed logic semantic mismatch রাখা
- view publish atomic না করা
- replay test না করা
- schema version ignore করা
- serving layer-এ overlap protection না রাখা

### Glossary

#### Master Dataset

Immutable raw fact store যা থেকে সব derived view তৈরি হয়।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Batch View

Historical full/partition recomputation-এর output।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Real-time View

Recent event-এর incremental temporary state।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Serving Layer

Batch + speed merge করে query answer দেয়।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Replay

Stored event log আবার চালিয়ে state rebuild করা।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Materialized View

Precomputed queryable state/table/index।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Late Event

যে event দেরিতে আসে এবং original event time পুরনো।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Idempotency

একই event বারবার এলেও final state একই থাকা।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Watermark

Event-time progress tracking boundary।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Cutoff Time

Batch view যে সময় পর্যন্ত authoritative coverage দেয়।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Atomic Swap

নতুন view এক ধাপে publish করা যাতে mixed read না হয়।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Event Log

Append-only ordered event history।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Append-only

শুধু নতুন record যোগ হয়, পুরনো mutate হয় না।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Schema Evolution

পুরনো event নষ্ট না করে data shape evolve করা।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Reconciliation

Independent sources মিলিয়ে exact correctness verify করা।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Dedupe

Duplicate event filter করা।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Delta

Batch baseline-এর পরে incremental change।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Historical Truth

Full dataset থেকে derived authoritative answer।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Live Freshness

এই মুহূর্তের কাছাকাছি data visibility।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।

#### Hybrid Architecture

Batch এবং stream capability modern unified tool-এ মিশে যাওয়া model।

কেন গুরুত্বপূর্ণ: Lambda এবং Kappa discussion-এ এই term বারবার আসে।
