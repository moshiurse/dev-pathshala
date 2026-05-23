# ⏱️ স্ট্রিম প্রসেসিং-এ Time Semantics ও Late Events Handling — গভীর বিশ্লেষণ

> স্ট্রিম প্রসেসিং-এ "event কবে ঘটেছে?" এই প্রশ্নের উত্তর ভুল হলে পুরো business logic ভুল হতে পারে।
> তাই Event Time, Watermark, Trigger, Allowed Lateness, Retraction এবং Exactly-Once semantics একসাথে বুঝতে হয়।
> এই নথিটি বাংলাদেশ context, ASCII ডায়াগ্রাম, PHP এবং Node.js উদাহরণসহ একটি গভীর গাইড।

---

## 📌 সংজ্ঞা — Time in Stream Processing

### Time semantics আসলে কী?

স্ট্রিম প্রসেসিং-এ **time semantics** বলতে বোঝায়:
কোন টাইমস্ট্যাম্পকে "সত্য" ধরা হবে,
কোন ইভেন্ট কোন window-তে পড়বে,
কখন একটি window বন্ধ হবে,
এবং দেরিতে আসা ইভেন্ট এলে কী করা হবে।

ব্যাচ সিস্টেমে আপনি অনেক সময় পরে সব ডেটা একসাথে সাজিয়ে নিতে পারেন।
স্ট্রিম সিস্টেমে সেই luxury থাকে না।
ইভেন্ট আসছে,
প্রসেস হচ্ছে,
রেজাল্ট বের হচ্ছে,
ড্যাশবোর্ড আপডেট হচ্ছে,
অ্যালার্ট ফায়ার হচ্ছে,
ফ্রড ডিটেকশন চলছে,
সব কিছু ঘটছে চলমান অবস্থায়।

এই কারণেই time শুধু metadata নয়,
এটি correctness-এর মূল অংশ।

### কেন time এত গুরুত্বপূর্ণ?

কারণ stream processing-এ বেশিরভাগ business question time-নির্ভর।

উদাহরণ:

- গত ৫ মিনিটে bKash-এ এক ইউজার কতবার transaction করেছে?
- শেষ ১০ মিনিটে Pathao driver কত দূর গিয়েছে?
- Prothom Alo-তে ১ মিনিট window-তে কত pageview এসেছে?
- Daraz flash sale-এ ৩০ সেকেন্ডে কত order placement হয়েছে?
- Foodpanda rider availability প্রতি ১৫ সেকেন্ডে কেমন বদলাচ্ছে?

এই সব প্রশ্নে শুধু event count যথেষ্ট নয়।
**সঠিক সময় অনুযায়ী count** দরকার।

### Stream processing-এর মৌলিক সমস্যা

সবচেয়ে বড় সমস্যা হলো:

**Events almost never arrive perfectly in order.**

কারণ:

- মোবাইল নেটওয়ার্ক delay
- retry
- queue backlog
- broker replication delay
- consumer lag
- batch upload from edge devices
- clock skew
- application crash and replay
- offline mobile sync
- regional connectivity issues

অর্থাৎ,
যে event আগে ঘটেছে,
সেটি পরে পৌঁছাতে পারে।

আর যে event পরে ঘটেছে,
সেটি আগে প্রসেস হতে পারে।

এটাই out-of-order arrival.

### Bangladesh context: কেন এই সমস্যা বাস্তবে খুবই সাধারণ?

বাংলাদেশে stream-time সমস্যা আরও বেশি দৃশ্যমান হতে পারে,
কারণ source device এবং central processing system-এর মধ্যে latency variation অনেক বেশি হতে পারে।

কিছু বাস্তব কারণ:

- গ্রামীণ এলাকায় 3G/4G coverage fluctuation
- GP, Robi, Banglalink, Teletalk network congestion
- handset battery saver mode-এর কারণে delayed sync
- মোবাইল অ্যাপ background restriction
- intermittent connectivity
- retry-driven duplicate delivery
- cash-in/cash-out agent apps-এর poor connectivity
- ride-sharing app-এ GPS burst upload
- e-commerce courier app-এর offline-first sync

উদাহরণ:

- bKash user 10:01:00-এ transaction করল,
  কিন্তু poor network-এর কারণে event broker-এ পৌঁছাল 10:05:30-এ।
- Pathao driver ৫ মিনিট offline ছিল,
  তারপর একসাথে ২০টা GPS point upload করল।
- Chaldal delivery app route event locally buffer করে পরে upload করল।
- Prothom Alo mobile app offline read metrics পরে sync করল।

এই ধরনের সিস্টেমে Processing Time-এর ওপর blind trust দিলে analytics এবং business action দুটোই ভুল হতে পারে।

### Time semantics-এর core goal

একটি stream processing engine সাধারণত চারটি প্রশ্নের উত্তর দিতে চায়:

1. কোন timestamp business truth হিসেবে ব্যবহার করা হবে?
2. কখন ধরে নেওয়া হবে যে একটি time window-এর অধিকাংশ event এসে গেছে?
3. late event এলে drop, update, নাকি আলাদা stream-এ পাঠানো হবে?
4. replay বা retry হলেও result কীভাবে logically consistent থাকবে?

এই চারটি প্রশ্নের সম্মিলিত উত্তরই time semantics.

---

### 2. Types of Time

### 2.1 Event Time (ইভেন্ট টাইম)

**Event Time** হলো event আসলে source-এ কবে ঘটেছে সেই সময়।

এটি source device,
application server,
POS terminal,
IoT sensor,
mobile app,
বা upstream domain system দ্বারা সেট হয়।

উদাহরণ:

- bKash transaction timestamp on user's phone
- Pathao driver GPS point capture time
- Daraz order creation time
- Foodpanda rider location ping time
- Prothom Alo pageview generated time
- GP cell tower metrics sampling time

Event Time business logic-এর জন্য সবচেয়ে meaningful,
কারণ এটি বাস্তব ঘটনার সময়কে capture করে।

#### Event Time-এর সুবিধা

- business correctness বজায় থাকে
- replay হলেও meaning বদলায় না
- out-of-order events সামলানো যায়
- window aggregation logically consistent হয়
- fraud detection time order অনুযায়ী কাজ করতে পারে
- sessionization accurate হয়
- billing/usage aggregation reliable হয়

#### Event Time-এর challenge

- source clock ভুল হতে পারে
- event late arrive করতে পারে
- event out of order হতে পারে
- watermark দরকার হয়
- state দীর্ঘ সময় ধরে রাখতে হতে পারে
- implementation complexity বাড়ে

#### bKash example

ধরা যাক Rahim 10:01:00-এ টাকা পাঠাল।
network slow হওয়ায় event broker-এ ঢুকল 10:05:30-এ।

যদি fraud system question হয়:
"Rahim কি 10:00-10:02 window-তে ৩টির বেশি high-value transfer করেছে?"

তাহলে event অবশ্যই 10:01:00 window-তে পড়বে,
10:05:30 window-তে নয়।

এটাই Event Time-এর গুরুত্ব।

---

### 2.2 Processing Time (প্রসেসিং টাইম)

**Processing Time** হলো processing system event-টিকে কবে পড়ল বা প্রসেস করল সেই সময়।

অর্থাৎ:

- consumer read time
- operator execution time
- application processing timestamp

উদাহরণ:

- Kafka consumer 10:05:30-এ message read করল
- Node.js service 10:05:31-এ handler execute করল
- Flink operator 10:05:31.200-এ record process করল

Processing Time implementation-এ সবচেয়ে সহজ।

কারণ engine শুধু `now()` ব্যবহার করে window assign করতে পারে।

#### Processing Time-এর সুবিধা

- খুব সহজ
- watermark complexity নেই
- কম state ধরে রাখতে হয়
- low-latency output পাওয়া যায়
- local benchmark/monitoring use-case-এ ভালো

#### Processing Time-এর অসুবিধা

- network delay দ্বারা প্রভাবিত
- consumer lag দ্বারা প্রভাবিত
- replay হলে history distort হয়
- queue backlog থাকলে ভুল windowing হয়
- business truth represent করে না
- cross-region ingest-এর variance analytics নষ্ট করতে পারে

#### কোথায় Processing Time acceptable?

- operational monitoring
- machine health metrics
- "consumer এখন কত event/sec প্রসেস করছে" টাইপ প্রশ্ন
- approximate dashboard
- non-critical alerting
- metrics that truly depend on processing lag itself

উদাহরণ:

"এই মুহূর্তে consumer প্রতি সেকেন্ডে কত message handle করছে?"
এটি Processing Time question.

কিন্তু
"শেষ ৫ মিনিটে user কত order করেছে?"
এটি সাধারণত Event Time question.

---

### 2.3 Ingestion Time (ইনজেশন টাইম)

**Ingestion Time** হলো event streaming system-এ কবে enter করেছে সেই সময়।

উদাহরণ:

- Kafka broker record append time
- Pub/Sub ingestion timestamp
- Kinesis arrival timestamp

এটি source-set নয়,
system-set।

তাই Event Time-এর তুলনায় source truth কম,
কিন্তু Processing Time-এর তুলনায় stable।

#### Ingestion Time-এর সুবিধা

- source clock-এর ওপর নির্ভরতা কম
- broker-level consistent timestamp পাওয়া যায়
- multiple consumer replay হলেও ingest timestamp অপরিবর্তিত থাকতে পারে
- implementation moderate complexity

#### Ingestion Time-এর সীমাবদ্ধতা

- source-to-broker latency ignore করে
- offline sync scenario-তে inaccurate
- mobile event delay থাকলে business truth হারায়
- event order partially improved হলেও perfect নয়

#### কোথায় Ingestion Time useful?

- যখন source timestamp unreliable
- যখন ingestion boundary-ই business truth-এর কাছাকাছি
- log aggregation
- semi-trusted device ecosystem
- rough ordering where source clocks are wild

---

### 2.4 একটি event-এর তিন ধরনের time একসাথে

একটি event-এর জন্য তিনটি সময় আলাদা হতে পারে:

- Event Time = event actually happened
- Ingestion Time = broker event গ্রহণ করল
- Processing Time = processing app event হ্যান্ডেল করল

উদাহরণ:

- user phone-এ transaction tap: 10:01:00
- Kafka broker receives it: 10:01:04
- consumer processes it: 10:01:20

কিন্তু আরও খারাপ scenario-ও হতে পারে:

- Event Time: 10:01:00
- Ingestion Time: 10:05:28
- Processing Time: 10:05:30

এখানে business logic যদি Event Time-aware না হয়,
তাহলে পুরো window aggregation ভুল হবে।

### তিন time-এর comparison table

| বৈশিষ্ট্য | Event Time | Ingestion Time | Processing Time |
|---|---|---|---|
| timestamp কে সেট করে | source | broker/system | processor |
| business correctness | সবচেয়ে ভালো | মাঝারি | সবচেয়ে দুর্বল |
| implementation complexity | বেশি | মাঝারি | কম |
| late event handling দরকার | হ্যাঁ | কিছুটা | কম |
| replay-safe | হ্যাঁ | হ্যাঁ/মাঝারি | না |
| out-of-order resilience | উচ্চ | মাঝারি | খুব কম |
| source clock skew sensitivity | আছে | কম | নেই |
| best use case | finance, fraud, analytics | broker-level analytics | ops metrics |

### Rule of thumb

- financial correctness চাইলে Event Time
- broker-arrival truth চাইলে Ingestion Time
- engine-centric operational metrics চাইলে Processing Time

---

### 4. Watermarks (ওয়াটারমার্ক) — The Core Mechanism

### 4.1 Watermark কী?

**Watermark** হলো একটি logical progress signal।

সহজ ভাষায়:

> "আমি বিশ্বাস করি timestamp <= W এমন অধিকাংশ event ইতোমধ্যে এসে গেছে।"

খেয়াল করুন,
এটি **guarantee নয়**।
এটি **heuristic**।

কারণ distributed system-এ absolutely নিশ্চিত হওয়া কঠিন যে আর কোনো পুরনো event আসবে না।

Watermark engine-কে সাহায্য করে এই সিদ্ধান্ত নিতে:

- window close করা যাবে কি না
- on-time result emit করা যাবে কি না
- event late কি না
- state কবে cleanup করা যাবে

#### কেন Watermark দরকার?

ধরুন 10:00-10:05 window।
System যদি exact certainty-র জন্য অপেক্ষা করে,
তাহলে window হয়তো কখনও বন্ধই হবে না।
কারণ future-এ late event আসতে পারে।

সুতরাং practical system একটি guess করে:
"Enough data এসেছে, এখন result দাও।"

এই practical guess-ই watermark.

---

### 4.2 Watermarks কীভাবে কাজ করে?

সাধারণ pattern:

1. source/partition-এ max event time track করা হয়
2. out-of-orderness buffer subtract করা হয়
3. result watermark forward করা হয়
4. watermark যখন window end অতিক্রম করে,
   window on-time close হতে পারে

একটি common formula:

`source_watermark = max_event_time_seen - allowed_out_of_orderness`

multi-input operator-এর ক্ষেত্রে:

`operator_watermark = min(all_input_watermarks)`

কারণ কোনো একটি input পিছিয়ে থাকলে,
সামগ্রিক correctness তার দ্বারাই সীমাবদ্ধ হয়।

#### Window close rule

যদি window হয় `[10:00, 10:05)` এবং watermark `10:05:00` বা তার বেশি হয়ে যায়,
তাহলে engine ধরে নিতে পারে window-এর on-time phase শেষ।

তারপর late events-এর জন্য grace policy প্রয়োগ করা হতে পারে।

---

### 4.3 Watermark generation strategies

#### Strategy A: Monotonically Increasing

এখানে ধরে নেওয়া হয় events order-এ আসবে।

Formula:

`watermark = max_event_time_seen`

এটি তখনই safe,
যখন source strict ordering নিশ্চিত করে।

#### সুবিধা

- low latency
- simple
- state কম লাগে

#### সমস্যা

একটি out-of-order event এলেই সেটি late হিসেবে ধরা পড়বে।

উদাহরণ:

- 10:03 event এল
- watermark = 10:03
- এরপর 10:02 event এল
- event late হয়ে গেল

এটি financial/mobile scenarios-এ সাধারণত খুব aggressive।

---

#### Strategy B: Bounded Out-of-Orderness

এটি production-এ সবচেয়ে common pattern।

Formula:

`watermark = max_event_time_seen - maxDelay`

যদি `maxDelay = 5 minutes` হয়,
তাহলে system ৫ মিনিট পর্যন্ত out-of-order tolerate করবে।

উদাহরণ:

- max event seen = 10:15:00
- watermark = 10:10:00
- অর্থাৎ 10:10:00 পর্যন্ত event on-time ধরা হচ্ছে

#### সুবিধা

- practical
- tunable
- most stream frameworks-এ supported

#### trade-off

- maxDelay যত বড়,
  latency তত বেশি
- maxDelay যত ছোট,
  late drop তত বেশি

---

#### Strategy C: Custom / Punctuated Watermarks

কখনও source নিজেই watermark-like signal পাঠায়।

উদাহরণ:

- CCTV camera batch close marker
- POS terminal day-close record
- telecom mediation system end-of-batch marker
- special control event: `{"type":"watermark","event_time":"10:05:00"}`

এখানে watermark external signal দ্বারা drive হয়।

#### সুবিধা

- domain-aware
- exact progress hint দিতে পারে

#### সমস্যা

- source cooperation দরকার
- ভুল control event serious error ঘটাতে পারে

---

#### Strategy D: Idle Source Handling

multi-partition system-এ একটি input idle হয়ে গেলে watermark stall হতে পারে।

কারণ merged watermark = min(all input watermarks).

উদাহরণ:

- Partition A watermark = 10:15
- Partition B watermark = 10:14
- Partition C idle at 10:02
- merged watermark stuck at 10:02

এক্ষেত্রে idle source detect করে "temporarily ignore" করতে হয়।

Framework-এ একে idle partition detection বলা হয়।

#### কেন দরকার?

- sparse traffic
- geo-distributed inputs
- partition imbalance
- one branch temporarily silent

---

### 4.4 Watermark propagation in a pipeline

একটি pipeline-এ watermark শুধু source-এ তৈরি হলেই শেষ নয়।
এটি operator থেকে operator-এ propagate হয়।

উদাহরণ:

Source -> Parse -> Filter -> KeyBy -> Window -> Aggregate -> Sink

প্রতিটি operator incoming watermark observe করে,
নিজের state অনুযায়ী safe হলে forward করে।

#### Multiple inputs-এর ক্ষেত্রে

যদি operator-এর একাধিক input থাকে,
তাহলে effective watermark সাধারণত **minimum** input watermark হয়।

কারণ:

- input A 10:10 পর্যন্ত গেছে
- input B 10:07 পর্যন্ত গেছে
- join operator 10:07-এর বেশি safe ধরতে পারে না

#### Join-এর ক্ষেত্রে এর গুরুত্ব

ধরা যাক Daraz order stream এবং payment stream join করবেন।
payment stream ধীর হলে,
join watermark payment stream-এর কারণে পিছিয়ে থাকবে।
তা না হলে আপনি premature close করে late matching records হারাতে পারেন।

---

### 4.5 Watermark delay trade-offs

Watermark tuning মূলত latency বনাম completeness trade-off।

#### Delay খুব ছোট হলে

- result দ্রুত পাওয়া যায়
- late event বেশি drop হয়
- financial accuracy কমে
- dashboard correction বেশি লাগে

#### Delay খুব বড় হলে

- completeness বাড়ে
- result পেতে দেরি হয়
- state size বাড়ে
- memory/storage cost বাড়ে
- user-facing latency বাড়ে

#### Practical interpretation

- 5 second delay => low latency, lower completeness
- 5 minute delay => higher completeness, slower dashboards
- 30 minute delay => near-batch behavior

এখানে one-size-fits-all solution নেই।
Workload-specific tuning দরকার।

---

### 5. Handling Late Events (দেরিতে আসা ইভেন্টের ব্যবস্থাপনা)

### 5.1 Late Event কী?

**Late Event** হলো এমন event,
যার event time-এর window ইতিমধ্যে watermark দ্বারা অতিক্রান্ত।

অর্থাৎ,
system logicalভাবে সেই window close করেছে,
কিন্তু পরে ঐ window-এর আরেকটি event এসে গেল।

Formal intuition:

`if event.eventTime <= currentWatermark then event is late`

তবে production-এ আরও nuanced definition লাগে,
কারণ allowed lateness থাকলে late-but-still-acceptable এবং too-late আলাদা।

#### Late event-এর তিনটি level

1. **On-time**
   - event time > watermark
   - window এখনও logicalভাবে open

2. **Late but within grace**
   - watermark window end পার করেছে
   - কিন্তু grace period এখনও শেষ হয়নি

3. **Too late**
   - grace period-ও শেষ
   - main computation আর update হবে না

#### "How late is too late?"

এটি business-driven.

উদাহরণ:

- ad analytics: 30 sec late acceptable
- ride tracking: 2 min late acceptable
- fraud detection: 5-10 min grace useful
- financial ledger: never lose, side stream required
- audit/reporting: main stream থেকে বাদ গেলেও later correction mandatory

---

### 5.2 Late event handling strategies

#### Strategy 1: Drop Late Events

late event এলে ignore করে দিন।

#### সুবিধা

- simplest
- low state cost
- low latency
- downstream stable
- dashboards easy

#### অসুবিধা

- data loss
- undercount
- wrong aggregates
- financial/audit ক্ষেত্রে unacceptable

#### acceptable যখন

- approximate analytics চলে
- ad/traffic dashboard
- exploratory metrics
- non-critical operational trend

#### unacceptable যখন

- bKash settlement
- anti-fraud rules
- invoice/billing
- compliance
- audit trail
- rider payroll
- cash reconciliation

---

#### Strategy 2: Allowed Lateness / Grace Period

window watermark pass করার পরও কিছু সময় state alive রাখা হয়।

একে grace period বলা হয়।

উদাহরণ:

- window end = 10:05
- watermark crosses at 10:06
- grace = 5 min
- until 10:10 event এলে result update হবে

Framework examples:

- Kafka Streams: `.grace(Duration.ofMinutes(5))`
- Flink: `.allowedLateness(Time.minutes(5))`

#### সুবিধা

- most practical
- majority late events capture করে
- exactness ও latency balance করে

#### অসুবিধা

- state দীর্ঘক্ষণ রাখতে হয়
- late update handling দরকার
- downstream consumers must accept corrections

---

#### Strategy 3: Side Output / Dead Letter Late Stream

late events main result update না করে separate stream/topic-এ পাঠানো হয়।

এটি real-time result stable রাখে,
আবার data loss-ও এড়ায়।

উদাহরণ:

- main topic: `fraud-window-results`
- late topic: `fraud-window-late-events`
- batch correction job প্রতি ঘণ্টায় late topic reprocess করে

#### সুবিধা

- no silent loss
- real-time path fast থাকে
- audit-friendly
- correction pipeline possible

#### অসুবিধা

- extra pipeline দরকার
- reconciliation logic লাগে
- operational complexity বাড়ে

---

#### Strategy 4: Retraction and Update

late event এলে system পুরনো result retract করে,
তারপর corrected result emit করে।

উদাহরণ:

- first result: `10:00-10:05 => 500 transactions`
- late event arrives
- retraction: `retract 500`
- new result: `10:00-10:05 => 501 transactions`

এটি relational/stream-table semantics-এ common.

#### সুবিধা

- most accurate live correction
- dashboard exact হতে পারে
- warehouse sink exactness improve করে

#### অসুবিধা

- downstream must understand updates
- idempotency দরকার
- sink semantics complex
- append-only systems-এ awkward

---

#### Strategy 5: Accumulating and Retracting Modes

late update emit করার বিভিন্ন mode আছে।

##### Accumulating
নতুন result emit হয়।
পুরনো result explicit retract হয় না।

উদাহরণ:

- emit 500
- late event
- emit 501

consumer ধরে নেবে latest wins.

##### Retracting
পুরনো result retract,
তারপর new result emit.

উদাহরণ:

- emit `RETRACT 500`
- emit `INSERT 501`

##### Accumulating + Retracting
কিছু system both semantics expose করে:

- old result retraction
- new current result
- optional delta

এটি changelog-oriented consumers-এর জন্য useful.

---

### 5.3 Late event decision matrix

| Strategy | Data Loss | Complexity | Latency | Use Case |
|---|---|---|---|---|
| Drop | Yes | Low | Low | Analytics যেখানে approximate acceptable |
| Grace Period | Minimal | Medium | Medium | অধিকাংশ production real-time system |
| Side Output | None | Medium | Low (main) + High (side) | Financial, audit, compliance |
| Retraction | None | High | Variable | Exact dashboard, changelog sinks |
| Accumulating | None | Medium | Variable | Consumer latest-wins হলে |
| Accumulating + Retracting | None | High | Variable | Advanced stream-table consumers |

### একটি practical recommendation

অনেক production system-এ hybrid approach সবচেয়ে ভালো:

- main pipeline: grace period + accumulating updates
- too-late path: side output
- offline reconciliation: batch correction

এতে near real-time UX বজায় থাকে,
আবার data loss কমে।

---

### 6. Triggers — কখন window result emit হবে?

Window তৈরি করা আর result emit করা একই জিনিস নয়।
Trigger ঠিক করে **কখন** output বের হবে।

### 6.1 Trigger types

#### On Watermark Trigger

default trigger.

watermark window end অতিক্রম করলে on-time result emit হয়।

এটি completeness-aware.

#### On Each Event / Early Firing

প্রতিটি event আসার সাথে partial result emit হতে পারে।

উদাহরণ:

- 10:00-10:05 window এখনও open
- 10:00:20-এ partial count = 12
- 10:01:30-এ partial count = 40
- 10:05+: final count = 97

dashboard-এ এটি useful.

#### Periodic Trigger

প্রতি N second/minute-এ emit হয়।

উদাহরণ:
প্রতি 30 seconds-এ rolling partial result.

#### Count Trigger

window-এ N event জমলে emit হয়।

উদাহরণ:
per 100 events partial fraud score update.

#### Composite Trigger

সবচেয়ে useful advanced pattern:

- early firing
- on-time firing
- late firing

অর্থাৎ:

- window বন্ধ হওয়ার আগে speculative result
- watermark pass করলে official result
- late event এলে correction result

---

### 6.2 Early Firing

Early firing-এর উদ্দেশ্য completeness নয়,
freshness.

এটি useful যখন dashboard user-কে "current picture" দেখাতে হবে।

উদাহরণ:

- Prothom Alo breaking news dashboard
- Foodpanda live order heatmap
- Pathao active ride count board
- GP NOC traffic spike dashboard

#### early firing result-এর nature

- partial
- speculative
- updateable
- not final

#### downside

- বেশি output
- downstream noise
- result revision দরকার
- consumer confusion হতে পারে যদি versioning না থাকে

---

### 6.3 Late Firing

Late event grace period-এর মধ্যে এলে আবার emit করা যায়।
এটিই late firing.

উদাহরণ:

- 10:00-10:05 window final emit = 500
- 10:07-এ late event arrives within grace
- late firing emit = 501

late firing সাধারণত retraction বা versioning-এর সাথে ভালো কাজ করে।

---

### Trigger design-এর practical pattern

একটি খুব common production pattern:

- every 30 sec early firing
- watermark crossing-এ final firing
- grace-এর মধ্যে late update firing
- grace শেষ হলে state cleanup

এটি dashboards + correctness দুইকেই balance করে।

---

### 7. Event Time Skew and Clock Synchronization

সব source-এর clock এক নয়।
distributed systems-এ clock skew বাস্তব সমস্যা।

### skew কোথা থেকে আসে?

- mobile phone manual time settings
- disabled automatic timezone
- cheap device oscillator drift
- NTP not synced
- edge server clock drift
- VM resume after pause
- container host time issues

### Bangladesh context

বাংলাদেশে mobile device ecosystem heterogeneous।
সব handset সমান quality বা configuration-এ থাকে না।

বাস্তব সমস্যা:

- user phone 2 minute fast
- driver phone 90 second slow
- offline app old cached timezone settings ব্যবহার করছে
- low-end Android device background sync delay করছে
- rural connectivity-তে burst upload হচ্ছে

এই কারণে source timestamp blindly trust করাও ঝুঁকিপূর্ণ।

### skew handling techniques

#### 1. NTP synchronization
server-side system অবশ্যই strict NTP sync-এ থাকা উচিত।

#### 2. sanity bounds
যদি event time `now + allowed_future_skew` এর বেশি হয়,
তাহলে suspect event হিসেবে mark করুন।

#### 3. server-enriched metadata
source event-এর সাথে broker ingest time,
API receive time,
gateway receive time store করুন।

#### 4. hybrid trust model
trusted devices-এর event time,
untrusted devices-এর ingestion time fallback.

#### 5. skew-aware watermarks
out-of-orderness delay-এর মধ্যে expected clock skew consider করুন।

#### 6. device quality segmentation
high-trust POS terminal এবং low-trust mobile app same rule-এ না চালানোই ভালো।

---

### 8. Exactly-Once with Event Time

Exactly-once semantics শুধু duplicate না হওয়া নয়।
Event Time-aware system-এ exactly-once মানে:

- একই logical event দুইবার result বদলাবে না
- replay হলেও final aggregate consistent থাকবে
- late update idempotent হবে
- window state duplicate tolerant হবে

### dedup key কী হতে পারে?

- transaction_id
- event_id
- producer sequence number
- source_id + event_time + nonce
- order_id + status_version
- rider_id + gps_timestamp + sample_id

### কেন event time এখানে relevant?

কারণ dedup করার সময় শুধু event ID নয়,
কোন window/which logical state update হচ্ছে সেটিও গুরুত্বপূর্ণ।

উদাহরণ:

- Kafka consumer crash করল
- offset commit আগের জায়গায় ছিল
- restart হয়ে old records আবার পড়ল
- Processing Time এখন completely নতুন
- কিন্তু Event Time + Event ID থাকলে duplicate update avoid করা যায়

### Kafka exactly-once + event time

Kafka transactional producer/consumer pipeline duplicate append কমাতে পারে,
কিন্তু business-level dedup তবুও লাগতে পারে।

কারণ:

- source duplicates
- retried API calls
- mobile resend
- upstream system replay
- CDC duplication edge case

### practical exactly-once stack

- producer idempotence
- transactional writes where possible
- stateful dedup store
- event-time keyed window state
- versioned sink updates
- retraction-aware consumers

---

### 11. Framework Support Comparison

| Feature | Kafka Streams | Apache Flink | Spark Streaming |
|---|---|---|---|
| Event Time | Yes | Yes | Yes |
| Watermarks | Yes (stream time oriented) | Yes (custom and rich) | Yes (threshold based) |
| Late Events | Grace period | Side output + allowed lateness | Watermark delay |
| Triggers | Limited | Rich | Limited |
| Retraction | Limited/indirect | Yes | Limited/rare |
| Idle source handling | Partial/workaround | Strong support | Available with caveats |
| Best fit | app-embedded JVM streams | full-featured event-time processing | micro-batch heavy ecosystems |

### Framework-wise intuition

#### Kafka Streams
- app-embedded
- simpler operational model
- grace period useful
- stream time concept practical
- custom trigger flexibility limited

#### Apache Flink
- richest event-time model
- custom watermark strategies
- side outputs
- allowed lateness
- retractions and advanced window semantics

#### Spark Structured Streaming
- micro-batch oriented mental model
- watermarks আছে
- simpler model for many data pipelines
- ultra-rich trigger/retraction use-case-এ Flink often stronger

---

### একটি mental model

Time semantics বুঝতে তিনটি axis ধরতে পারেন:

1. **Correctness axis**
   - event time best
   - processing time weakest

2. **Latency axis**
   - small watermark delay fastest
   - big delay slowest

3. **Complexity axis**
   - drop late easiest
   - side output/retraction hardest

Production design হলো এই তিন axis balance করা।

---

## 🌍 বাস্তব উদাহরণ — Bangladesh context

### Scenario 1: Out-of-order bKash transactions

ধরা যাক user `U-88017XXXXXXX` 10:01:00-এ Send Money করল `৳20,000`।
network issue-এর কারণে event central pipeline-এ পৌঁছাল 10:05:30-এ।

এর মধ্যে একই user 10:03:00-এ আরেকটি transaction করল `৳15,000`,
এবং সেটি 10:03:01-এই Kafka topic-এ চলে এলো।

#### যদি Processing Time ব্যবহার করেন

event order হবে:

1. 10:03 transaction
2. 10:01 transaction

Fraud rule যদি বলে:
"২ মিনিটে ধারাবাহিক high-value transfers detect করো"
তাহলে system ভুল sequence পাবে।

#### সম্ভাব্য ক্ষতি

- fraud pattern miss
- wrong velocity score
- risk engine delayed response
- manual review trigger wrong time window-তে যাবে

#### যদি Event Time ব্যবহার করেন

logical order হবে:

1. 10:01 transaction
2. 10:03 transaction

এখন rule engine সঠিক sequence দেখে বুঝতে পারবে,
user ২ মিনিটের মধ্যে repeated large transfer করেছে।

#### real lesson

finance, wallet, banking-like domain-এ time ordering business-critical।

---

### Scenario 2: Pathao driver GPS events delayed

একজন Pathao driver Rangpur জেলার rural route-এ চলছে।
network coverage দুর্বল।
app locally GPS points buffer করছে।

actual capture:

- 11:00:10
- 11:00:20
- 11:00:30
- 11:00:40
- 11:00:50

কিন্তু upload হলো 11:05:05-এ burst আকারে।

#### Processing Time consequence

system মনে করবে সব points 11:05 minute-এই ঘটেছে।

#### impact

- live heatmap jump করবে
- route reconstruction incorrect হবে
- ETA model distorted হবে
- surge pricing zone wrong signal পেতে পারে
- driver idle/active detection misbehave করতে পারে

#### Event Time consequence

system points-গুলো original capture time অনুযায়ী place করবে।
route smooth থাকবে।
distance calculation realistic হবে।
surge analytics accurate হবে।

---

### Scenario 3: System replay after crash

ধরা যাক একটি Kafka consumer crash করল 14:10-এ।
শেষ committed offset ছিল 14:04-এর কাছাকাছি।
restart-এর পর consumer 14:04 থেকে old events replay করছে।

#### যদি Processing Time-based windows হয়

14:04-এর পুরোনো event-গুলো এখন 14:10, 14:11, 14:12 processing time পাচ্ছে।
অর্থাৎ historical event নতুন window-তে ঢুকে যাচ্ছে।

#### impact

- dashboard double counting
- anomaly detection false positive
- replay safe নয়
- incident investigation harder

#### যদি Event Time-based windows হয়

replayed events তাদের original event time ধরে আগের window-তেই যায়।
Processing Time বদলালেও business truth বদলায় না।

#### lesson

replay-safe systems almost always event-time aware হওয়া উচিত।

---

### Scenario 4: Prothom Alo mobile pageview sync

একজন user train-এ বসে Prothom Alo mobile app-এ অনেক article পড়েছে।
network intermittent ছিল।
analytics events local storage-তে জমা ছিল।
২৫ মিনিট পরে network পাওয়ার পর sync হলো।

#### যদি only processing time use করা হয়

সব pageview ২৫ মিনিট later spike হিসেবে দেখা যাবে।

#### impact

- breaking news traffic graph misleading
- campaign attribution wrong
- editor dashboard sudden spike দেখে ভুল editorial decision নিতে পারে
- article freshness analysis নষ্ট হয়

#### better approach

Event Time pageview capture moment ধরে রাখবে।
Main dashboard grace period সহ চলতে পারে।
Very late pageview side-output দিয়ে batch correction হতে পারে।

---

### Scenario 5: Chaldal order picking events

warehouse picker app থেকে events আসে:

- item_picked
- item_missing
- substitute_confirmed
- packed
- dispatched

picker tablet connectivity দুর্বল হলে কিছু events delay হতে পারে।

যদি `packed` আগে আসে এবং `item_missing` পরে আসে,
Processing Time ordering warehouse KPI-কে wrong দেখাতে পারে।

Event Time-aware pipeline picker process timeline reconstruct করতে পারবে।

---

### Scenario 6: Foodpanda rider payout and attendance

rider online/offline,
pickup,
dropoff,
waiting,
arrived-at-restaurant ইভেন্ট payout logic-এ ভূমিকা রাখে।

late event handling ভুল হলে:

- rider active minutes undercount
- incentive ভুল
- payout dispute
- audit mismatch

এইখানে main real-time dashboard-এ grace period,
এবং payroll finalization-এ side-output reconciliation common strategy হতে পারে।

---

### Scenario 7: GP/Robi network telemetry

cell tower metrics sampling time ও collection time এক নয়।

যদি 15-second interval radio utilization metrics delayed আসে,
Processing Time-based NOC dashboard wrong hotspot report করতে পারে।

Event Time + watermark + periodic early firing ব্যবহার করলে:

- live estimate পাওয়া যাবে
- final corrected utilization-ও পাওয়া যাবে

---

### Scenario 8: Daraz flash sale inventory demand

flash sale window 20:00-20:05।
mobile app traffic burst।
কিছু checkout event retry এবং network delay-এ পরে আসছে।

যদি Processing Time ব্যবহার হয়,
20:05-এর পরে আসা event wrong time bucket-এ পড়বে।

impact:

- demand forecast ভুল
- inventory reallocation late
- promotion effectiveness skewed

Grace period + retraction-aware dashboard এখানে ভালো কাজ করে।

---

### Scenario 9: bKash fraud + compliance hybrid model

একই event দুই pipeline-এ যেতে পারে:

1. real-time fraud scoring pipeline
2. compliance/audit pipeline

real-time pipeline 2-minute watermark delay + 5-minute grace ব্যবহার করে।
compliance pipeline all events store করে,
too-late events side stream থেকে batch correction আনে।

এটি বাস্তব production architecture-এর খুব common shape।

---

### Scenario 10: Payment reversal and status correction

ধরা যাক:

- initial event: payment_success at 16:00:05
- later delayed event: payment_timeout at 15:59:59
- then final reversal event: payment_reversed at 16:04:00

এখানে শুধু late arrival নয়,
event-time ordering business state machine-ও বদলে দিতে পারে।

যদি latest processing time blindly নেওয়া হয়,
status history wrong হতে পারে।

Event Time + versioned event handling + retractions দরকার হতে পারে।

---

## 🗺️ ASCII Diagrams

### Diagram 1: একটি event-এর Event Time, Ingestion Time, Processing Time

```text
Single Event Timeline
=====================

User Phone               API Gateway             Kafka Broker            Consumer
    |                         |                      |                    |
    |-- tap "Send" ------->|                      |                    |
    |  Event Time=10:01:00 |                      |                    |
    |                      |-- HTTP arrives ----->|                    |
    |                      |  In-flight delay     |                    |
    |                      |  4 sec               |                    |
    |                      |                      |-- append ---------->|
    |                      |                      |  Ingestion Time     |
    |                      |                      |  = 10:01:04         |
    |                      |                      |                    |
    |                      |                      |  queue backlog      |
    |                      |                      |  16 sec             |
    |                      |                      |                    |
    |                      |                      |---------- read ---->|
    |                      |                      |                     |
    |                      |                      | Processing Time     |
    |                      |                      | = 10:01:20          |
    |                      |                      |                     |

Summary:
  Event Time      = 10:01:00
  Ingestion Time  = 10:01:04
  Processing Time = 10:01:20
```

### Diagram 2: Out-of-order arrival

```text
Out-of-Order Arrival
====================

Actual event order (business truth):
  E1 @ 10:01:00  -----> "Send Money 20000"
  E2 @ 10:03:00  -----> "Send Money 15000"

Arrival order (processing reality):
  E2 arrives @ 10:03:01
  E1 arrives @ 10:05:30

If ordered by Processing Time:
  [E2] -> [E1]   WRONG for business logic

If ordered by Event Time:
  [E1] -> [E2]   CORRECT
```

### Diagram 3: Watermark progression with bounded lateness

```text
Watermark Progression
=====================

Assume:
  maxOutOfOrderness = 2 minutes
  watermark = maxEventSeen - 2m

Incoming events:
  see event @ 10:01  -> maxSeen=10:01 -> watermark=09:59
  see event @ 10:03  -> maxSeen=10:03 -> watermark=10:01
  see event @ 10:02  -> maxSeen=10:03 -> watermark=10:01
  see event @ 10:06  -> maxSeen=10:06 -> watermark=10:04
  see event @ 10:05  -> maxSeen=10:06 -> watermark=10:04

Visual:
  Event Time Axis -------------------------------------------->
  09:59   10:00   10:01   10:02   10:03   10:04   10:05   10:06

  after event 10:01:  WM at 09:59
  after event 10:03:  WM at 10:01
  after event 10:02:  WM stays 10:01
  after event 10:06:  WM at 10:04
  after event 10:05:  WM stays 10:04
```

### Diagram 4: Window closing by watermark

```text
Window Closing
==============

Window A: [10:00, 10:05)
Window B: [10:05, 10:10)

Event time axis:
  10:00 -------- 10:05 -------- 10:10 -------- 10:15

Current watermark moves:
  10:02  -> Window A still open
  10:04  -> Window A still open
  10:05  -> Window A can emit on-time result
  10:07  -> Window B still open
  10:10  -> Window B can emit on-time result

Rule:
  when watermark >= window_end
  then window is eligible for on-time firing
```

### Diagram 5: Grace period and too-late events

```text
Late Event Lifecycle
====================

Window:
  [10:00, 10:05)

Watermark crosses:
  10:06

Grace period:
  5 minutes

Timeline:
  10:00 -------- 10:05 -------- 10:06 -------- 10:10 -------- 10:11
                  window end      on-time emit   grace end      too late

Cases:
  eventTime=10:04 arrives at WM=10:04   -> on-time
  eventTime=10:04 arrives at WM=10:06   -> late but within grace
  eventTime=10:04 arrives at WM=10:11   -> too late
```

### Diagram 6: Multiple input watermark propagation

```text
Multiple Inputs
===============

Source A watermark = 10:12
Source B watermark = 10:09
Source C watermark = 10:15

             +-----------+
Source A --->|           |
             |   Join    |----> output watermark = min(10:12, 10:09, 10:15)
Source B --->| Operator  |----> output watermark = 10:09
             |           |
Source C --->|           |
             +-----------+

Reason:
  join cannot safely conclude anything beyond the slowest input
```

### Diagram 7: Idle source problem

```text
Idle Source Problem
===================

Partition P0 watermark = 10:20
Partition P1 watermark = 10:19
Partition P2 watermark = 10:03 and no new data

Merged watermark = min(P0, P1, P2) = 10:03

Without idle detection:
  all downstream windows stall

With idle detection:
  mark P2 idle
  merged watermark = min(P0, P1) = 10:19
```

### Diagram 8: Latency vs Completeness spectrum

```text
Latency vs Completeness
=======================

small delay ---------------------------------------------- large delay
fast result                                               slow result
more missing late data                                    more complete

0 sec delay     5 sec delay     30 sec delay     5 min delay    30 min
|---------------|---------------|----------------|--------------|
very fast       dashboard       common realtime  finance-ish    near batch
but risky       friendly        compromise       safer          very safe
```

### Diagram 9: Trigger timeline

```text
Trigger Timeline for One Window
===============================

Window = [10:00, 10:05)
Early trigger every 1 minute
On-time trigger at watermark >= 10:05
Late trigger on each accepted late event
Grace = 5 minutes

10:00   10:01   10:02   10:03   10:04   10:05   10:06   10:07   10:10
  |       |       |       |       |       |       |       |       |
  |-- early fire #1
          |-- early fire #2
                  |-- early fire #3
                          |-- early fire #4
                                  |-- on-time fire
                                          |-- late fire #1
                                                  |-- late fire #2
                                                                  grace end
```

### Diagram 10: Retraction flow

```text
Retraction and Update
=====================

Initial on-time result:
  window[10:00-10:05] = 500

Late event arrives:
  +1 transaction

System emits:
  RETRACT window[10:00-10:05] = 500
  INSERT  window[10:00-10:05] = 501

Consumer view:
  old number invalid
  new number is authoritative
```

### Diagram 11: Accumulating mode

```text
Accumulating Mode
=================

Emit:
  window[10:00-10:05] = 500
Late event:
  window[10:00-10:05] = 501

Consumer contract:
  latest record wins
  no explicit delete of old 500
```

### Diagram 12: Event-time skew

```text
Clock Skew Example
==================

Real world:
  server time = 12:00:00

Devices:
  Phone A = 12:01:30   (90 sec fast)
  Phone B = 11:59:20   (40 sec slow)
  POS C   = 12:00:02   (good)

If all timestamps blindly trusted:
  ordering may become weird
  watermark may jump or lag unexpectedly

Mitigation:
  if event_time > ingest_time + allowed_future_skew
     mark suspicious
```

### Diagram 13: Exactly-once with replay

```text
Replay-Safe Event Time Processing
=================================

Producer ---> Kafka ---> Consumer ---> Window State ---> Dashboard
                 ^
                 |
             crash/restart
                 |
           re-read same offsets

Protection layers:
  1. event_id dedup
  2. event_time window mapping
  3. idempotent state update
  4. versioned output

Result:
  replay does not create logical double count
```

### Diagram 14: Fraud detection pipeline with late-event side stream

```text
Fraud Pipeline
==============

+-------------+      +----------------+      +-----------------+
| Mobile App  | ---> | Kafka Raw Topic | ---> | Parse / Validate |
+-------------+      +----------------+      +-----------------+
                                                   |
                                                   v
                                            +--------------+
                                            | Watermark    |
                                            | Generator    |
                                            +--------------+
                                                   |
                                        +----------+----------+
                                        |                     |
                                        v                     v
                                +---------------+    +------------------+
                                | Main Windowed |    | Late Side Topic  |
                                | Fraud Engine  |    | / Dead Letter    |
                                +---------------+    +------------------+
                                        |
                                        v
                                +----------------+
                                | Risk Score     |
                                | Alerts         |
                                +----------------+
                                        |
                                        v
                                +----------------+
                                | Dashboard /    |
                                | Case Manager   |
                                +----------------+
```

### Diagram 15: Watermark in a DAG

```text
Watermark in a DAG
==================

         +--------+
         |Source A|----+
         +--------+    |
                       v
                    +------+
                    |Parse |
                    +------+
                       |
                       v
                    +------+
                    |Filter|-------------------+
                    +------+                   |
                                               v
         +--------+                         +--------+
         |Source B|-----------------------> |  Join  | ----> +---------+
         +--------+                         +--------+       | Window  |
             |                                  |            +---------+
             |                                  |                |
             |                                  v                v
             +----------------------------> effective WM ----> output WM

Rules:
  parse WM_out   = WM_in
  filter WM_out  = WM_in
  join WM_out    = min(left WM, right WM)
  window emits when WM_out >= window_end
```

### Diagram 16: Business decision timeline

```text
Business Decision vs Time Semantics
===================================

Event happens ----> ingested ----> processed ----> dashboard shown ----> corrected later
     |                  |              |                 |                     |
  business truth     system truth   engine truth     user perception      final truth

Design challenge:
  how much delay can business tolerate
  before showing a result?
```

---

## 💻 PHP Code

### PHP example 1: Basic event model and watermark generator

```php
<?php

declare(strict_types=1);

final class StreamEvent
{
    public function __construct(
        public readonly string $eventId,
        public readonly string $userId,
        public readonly string $type,
        public readonly int $amount,
        public readonly int $eventTimeMs,
        public readonly int $ingestionTimeMs,
        public readonly array $payload = [],
    ) {}
}

final class WatermarkGenerator
{
    private int $maxEventTimeSeen = PHP_INT_MIN;
    private int $currentWatermark = PHP_INT_MIN;

    public function __construct(
        private readonly int $outOfOrdernessMs
    ) {}

    public function observe(StreamEvent $event): int
    {
        if ($event->eventTimeMs > $this->maxEventTimeSeen) {
            $this->maxEventTimeSeen = $event->eventTimeMs;
        }

        $candidate = $this->maxEventTimeSeen - $this->outOfOrdernessMs;
        $this->currentWatermark = max($this->currentWatermark, $candidate);

        return $this->currentWatermark;
    }

    public function getCurrentWatermark(): int
    {
        return $this->currentWatermark;
    }

    public function formatWatermark(): string
    {
        if ($this->currentWatermark === PHP_INT_MIN) {
            return 'UNINITIALIZED';
        }

        return gmdate('Y-m-d H:i:s', (int) floor($this->currentWatermark / 1000));
    }
}

// bKash-like example
$generator = new WatermarkGenerator(outOfOrdernessMs: 2 * 60 * 1000);

$events = [
    new StreamEvent('e1', 'u1', 'send_money', 20000, strtotime('2025-01-01 10:01:00') * 1000, strtotime('2025-01-01 10:01:04') * 1000),
    new StreamEvent('e2', 'u1', 'send_money', 15000, strtotime('2025-01-01 10:03:00') * 1000, strtotime('2025-01-01 10:03:01') * 1000),
    new StreamEvent('e3', 'u1', 'send_money', 18000, strtotime('2025-01-01 10:02:00') * 1000, strtotime('2025-01-01 10:05:30') * 1000),
];

foreach ($events as $event) {
    $watermark = $generator->observe($event);

    echo sprintf(
        "event=%s eventTime=%s watermark=%s\n",
        $event->eventId,
        gmdate('H:i:s', (int) floor($event->eventTimeMs / 1000)),
        gmdate('H:i:s', (int) floor($watermark / 1000)),
    );
}
```

### PHP example 2: GracePeriodManager with Redis TTL

```php
<?php

declare(strict_types=1);

use Redis;

final class GracePeriodManager
{
    public function __construct(
        private readonly Redis $redis,
        private readonly int $windowSizeMs,
        private readonly int $graceMs,
        private readonly int $extraCleanupBufferMs = 60_000,
    ) {}

    public function rememberWindow(string $windowKey, int $windowEndMs): void
    {
        $ttlSeconds = (int) ceil(($this->windowSizeMs + $this->graceMs + $this->extraCleanupBufferMs) / 1000);

        $this->redis->hMSet($windowKey, [
            'window_end_ms' => (string) $windowEndMs,
            'grace_end_ms'  => (string) ($windowEndMs + $this->graceMs),
        ]);

        $this->redis->expire($windowKey, $ttlSeconds);
    }

    public function isWithinGrace(string $windowKey, int $watermarkMs): bool
    {
        $graceEnd = $this->redis->hGet($windowKey, 'grace_end_ms');

        if ($graceEnd === false) {
            return false;
        }

        return $watermarkMs <= (int) $graceEnd;
    }

    public function graceEndMs(string $windowKey): ?int
    {
        $value = $this->redis->hGet($windowKey, 'grace_end_ms');

        return $value === false ? null : (int) $value;
    }
}
```

### PHP example 3: Late event handling strategies

```php
<?php

declare(strict_types=1);

enum LateEventStrategy: string
{
    case DROP = 'drop';
    case GRACE_PERIOD = 'grace_period';
    case SIDE_OUTPUT = 'side_output';
    case RETRACT_AND_UPDATE = 'retract_and_update';
    case ACCUMULATE = 'accumulate';
}

final class LateEventHandler
{
    public function __construct(
        private readonly LateEventStrategy $strategy,
        private readonly GracePeriodManager $graceManager,
        private $lateSink = null,
    ) {}

    public function handle(
        StreamEvent $event,
        string $windowKey,
        int $watermarkMs,
        array $currentAggregate
    ): array {
        $withinGrace = $this->graceManager->isWithinGrace($windowKey, $watermarkMs);

        if ($this->strategy === LateEventStrategy::DROP && !$withinGrace) {
            return [
                'action' => 'dropped',
                'reason' => 'too_late',
                'event_id' => $event->eventId,
            ];
        }

        if ($this->strategy === LateEventStrategy::SIDE_OUTPUT && !$withinGrace) {
            if ($this->lateSink !== null) {
                ($this->lateSink)($event, $windowKey, 'too_late');
            }

            return [
                'action' => 'side_output',
                'event_id' => $event->eventId,
                'window_key' => $windowKey,
            ];
        }

        if ($this->strategy === LateEventStrategy::RETRACT_AND_UPDATE) {
            $old = $currentAggregate['count'] ?? 0;
            $new = $old + 1;

            return [
                'action' => 'retract_and_update',
                'retract' => [
                    'window_key' => $windowKey,
                    'count' => $old,
                ],
                'update' => [
                    'window_key' => $windowKey,
                    'count' => $new,
                ],
            ];
        }

        if ($this->strategy === LateEventStrategy::ACCUMULATE) {
            return [
                'action' => 'accumulate',
                'update' => [
                    'window_key' => $windowKey,
                    'count' => ($currentAggregate['count'] ?? 0) + 1,
                ],
            ];
        }

        if ($withinGrace) {
            return [
                'action' => 'accepted_within_grace',
                'update' => [
                    'window_key' => $windowKey,
                    'count' => ($currentAggregate['count'] ?? 0) + 1,
                ],
            ];
        }

        return [
            'action' => 'dropped',
            'reason' => 'unsupported_combination',
        ];
    }
}
```

### PHP example 4: Event-time window processor with watermark-based closing

```php
<?php

declare(strict_types=1);

final class EventTimeWindowProcessor
{
    private array $windowState = [];
    private array $closedWindows = [];
    private array $seenEventIds = [];

    public function __construct(
        private readonly int $windowSizeMs,
        private readonly WatermarkGenerator $watermarkGenerator,
        private readonly LateEventHandler $lateEventHandler,
        private readonly GracePeriodManager $graceManager,
    ) {}

    public function process(StreamEvent $event): array
    {
        if (isset($this->seenEventIds[$event->eventId])) {
            return [['type' => 'duplicate_ignored', 'event_id' => $event->eventId]];
        }

        $this->seenEventIds[$event->eventId] = true;

        $watermark = $this->watermarkGenerator->observe($event);
        $window = $this->windowFor($event->eventTimeMs);
        $windowKey = $this->windowKey($window['start'], $window['end']);

        $this->graceManager->rememberWindow($windowKey, $window['end']);

        $results = [];

        if (isset($this->closedWindows[$windowKey]) && $event->eventTimeMs <= $watermark) {
            $lateAction = $this->lateEventHandler->handle(
                event: $event,
                windowKey: $windowKey,
                watermarkMs: $watermark,
                currentAggregate: $this->windowState[$windowKey] ?? ['count' => 0, 'sum' => 0],
            );

            if (($lateAction['action'] ?? null) === 'accepted_within_grace') {
                $this->applyEvent($windowKey, $event);
            }

            if (($lateAction['action'] ?? null) === 'accumulate') {
                $this->applyEvent($windowKey, $event);
            }

            if (($lateAction['action'] ?? null) === 'retract_and_update') {
                $this->applyEvent($windowKey, $event);
            }

            $results[] = ['type' => 'late_event', 'payload' => $lateAction];
        } else {
            $this->applyEvent($windowKey, $event);
        }

        foreach ($this->closeEligibleWindows($watermark) as $emission) {
            $results[] = $emission;
        }

        return $results;
    }

    private function applyEvent(string $windowKey, StreamEvent $event): void
    {
        if (!isset($this->windowState[$windowKey])) {
            $this->windowState[$windowKey] = [
                'count' => 0,
                'sum' => 0,
                'high_value_count' => 0,
                'users' => [],
            ];
        }

        $this->windowState[$windowKey]['count']++;
        $this->windowState[$windowKey]['sum'] += $event->amount;

        if ($event->amount >= 15000) {
            $this->windowState[$windowKey]['high_value_count']++;
        }

        $this->windowState[$windowKey]['users'][$event->userId] = true;
    }

    private function closeEligibleWindows(int $watermarkMs): array
    {
        $emissions = [];

        foreach ($this->windowState as $windowKey => $aggregate) {
            [$startMs, $endMs] = array_map('intval', explode(':', $windowKey));

            if ($watermarkMs >= $endMs && !isset($this->closedWindows[$windowKey])) {
                $this->closedWindows[$windowKey] = true;

                $emissions[] = [
                    'type' => 'on_time_window_result',
                    'window_key' => $windowKey,
                    'window_start' => gmdate('H:i:s', (int) floor($startMs / 1000)),
                    'window_end' => gmdate('H:i:s', (int) floor($endMs / 1000)),
                    'count' => $aggregate['count'],
                    'sum' => $aggregate['sum'],
                    'high_value_count' => $aggregate['high_value_count'],
                    'unique_users' => count($aggregate['users']),
                ];
            }
        }

        return $emissions;
    }

    private function windowFor(int $eventTimeMs): array
    {
        $start = intdiv($eventTimeMs, $this->windowSizeMs) * $this->windowSizeMs;
        $end = $start + $this->windowSizeMs;

        return ['start' => $start, 'end' => $end];
    }

    private function windowKey(int $startMs, int $endMs): string
    {
        return $startMs . ':' . $endMs;
    }
}
```

### PHP example 5: Fraud detection pipeline with multiple late-event outcomes

```php
<?php

declare(strict_types=1);

final class FraudDetectionPipeline
{
    public function __construct(
        private readonly EventTimeWindowProcessor $processor,
        private readonly Redis $redis,
    ) {}

    public function handleRawRecord(array $record): void
    {
        $event = new StreamEvent(
            eventId: $record['event_id'],
            userId: $record['user_id'],
            type: $record['type'],
            amount: (int) $record['amount'],
            eventTimeMs: (int) $record['event_time_ms'],
            ingestionTimeMs: (int) $record['ingestion_time_ms'],
            payload: $record,
        );

        foreach ($this->processor->process($event) as $result) {
            $this->dispatch($result, $event);
        }
    }

    private function dispatch(array $result, StreamEvent $event): void
    {
        switch ($result['type']) {
            case 'on_time_window_result':
                $risk = $this->calculateRisk($result);

                $this->emitMainResult([
                    'window_key' => $result['window_key'],
                    'count' => $result['count'],
                    'sum' => $result['sum'],
                    'risk_score' => $risk,
                    'kind' => 'on_time',
                ]);

                if ($risk >= 90) {
                    $this->raiseAlert($event->userId, $result, $risk);
                }
                break;

            case 'late_event':
                $payload = $result['payload'];
                $action = $payload['action'] ?? 'unknown';

                if ($action === 'retract_and_update') {
                    $this->emitCorrection($payload['retract'], 'retract');
                    $this->emitCorrection($payload['update'], 'update');
                } elseif ($action === 'accepted_within_grace' || $action === 'accumulate') {
                    $this->emitCorrection($payload['update'], 'late_update');
                } elseif ($action === 'side_output') {
                    $this->emitLateSideStream($event, 'too_late');
                } elseif ($action === 'dropped') {
                    $this->incrementMetric('late_events_dropped_total');
                }
                break;

            case 'duplicate_ignored':
                $this->incrementMetric('duplicate_events_ignored_total');
                break;
        }
    }

    private function calculateRisk(array $windowResult): int
    {
        $score = 0;

        if ($windowResult['count'] >= 3) {
            $score += 30;
        }

        if ($windowResult['sum'] >= 50000) {
            $score += 35;
        }

        if ($windowResult['high_value_count'] >= 2) {
            $score += 25;
        }

        if ($windowResult['unique_users'] === 1 && $windowResult['count'] >= 4) {
            $score += 10;
        }

        return min($score, 100);
    }

    private function emitMainResult(array $payload): void
    {
        $this->redis->rPush('fraud:main-results', json_encode($payload, JSON_UNESCAPED_UNICODE));
    }

    private function emitCorrection(array $payload, string $kind): void
    {
        $payload['kind'] = $kind;
        $this->redis->rPush('fraud:corrections', json_encode($payload, JSON_UNESCAPED_UNICODE));
    }

    private function emitLateSideStream(StreamEvent $event, string $reason): void
    {
        $this->redis->rPush('fraud:late-events', json_encode([
            'reason' => $reason,
            'event_id' => $event->eventId,
            'user_id' => $event->userId,
            'event_time_ms' => $event->eventTimeMs,
            'ingestion_time_ms' => $event->ingestionTimeMs,
            'payload' => $event->payload,
        ], JSON_UNESCAPED_UNICODE));
    }

    private function raiseAlert(string $userId, array $windowResult, int $risk): void
    {
        $this->redis->rPush('fraud:alerts', json_encode([
            'user_id' => $userId,
            'risk_score' => $risk,
            'window_key' => $windowResult['window_key'],
            'message' => 'bKash-style high velocity transaction pattern detected',
        ], JSON_UNESCAPED_UNICODE));
    }

    private function incrementMetric(string $metric): void
    {
        $this->redis->incr('metric:' . $metric);
    }
}
```

### PHP example 6: Complete bootstrap for a Redis-backed pipeline

```php
<?php

declare(strict_types=1);

use Redis;

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$watermarkGenerator = new WatermarkGenerator(
    outOfOrdernessMs: 2 * 60 * 1000
);

$graceManager = new GracePeriodManager(
    redis: $redis,
    windowSizeMs: 5 * 60 * 1000,
    graceMs: 5 * 60 * 1000,
);

$lateHandler = new LateEventHandler(
    strategy: LateEventStrategy::RETRACT_AND_UPDATE,
    graceManager: $graceManager,
    lateSink: function (StreamEvent $event, string $windowKey, string $reason) use ($redis): void {
        $redis->rPush('late:dead-letter', json_encode([
            'reason' => $reason,
            'window_key' => $windowKey,
            'event_id' => $event->eventId,
            'event_time_ms' => $event->eventTimeMs,
            'payload' => $event->payload,
        ], JSON_UNESCAPED_UNICODE));
    }
);

$processor = new EventTimeWindowProcessor(
    windowSizeMs: 5 * 60 * 1000,
    watermarkGenerator: $watermarkGenerator,
    lateEventHandler: $lateHandler,
    graceManager: $graceManager,
);

$pipeline = new FraudDetectionPipeline($processor, $redis);

$records = [
    [
        'event_id' => 'txn-1001',
        'user_id' => '01711111111',
        'type' => 'send_money',
        'amount' => 20000,
        'event_time_ms' => strtotime('2025-01-01 10:01:00') * 1000,
        'ingestion_time_ms' => strtotime('2025-01-01 10:01:05') * 1000,
    ],
    [
        'event_id' => 'txn-1002',
        'user_id' => '01711111111',
        'type' => 'send_money',
        'amount' => 15000,
        'event_time_ms' => strtotime('2025-01-01 10:03:00') * 1000,
        'ingestion_time_ms' => strtotime('2025-01-01 10:03:01') * 1000,
    ],
    [
        'event_id' => 'txn-1003',
        'user_id' => '01711111111',
        'type' => 'send_money',
        'amount' => 18000,
        'event_time_ms' => strtotime('2025-01-01 10:02:00') * 1000,
        'ingestion_time_ms' => strtotime('2025-01-01 10:05:30') * 1000,
    ],
];

foreach ($records as $record) {
    $pipeline->handleRawRecord($record);
}

echo "Fraud pipeline processed records successfully.\n";
```

### PHP implementation notes

এই PHP code-এ কিছু গুরুত্বপূর্ণ pattern আছে:

- `event_id` ভিত্তিক dedup
- watermark progression
- window close on watermark
- grace period via Redis TTL
- retraction/update support
- side output ready
- replay-safe aggregate mapping
- financial stream-এর জন্য extensible foundation

যদি Laravel context-এ integrate করেন,
তাহলে:

- Redis facade ব্যবহার করতে পারেন
- queue worker দিয়ে raw topic consumer simulate করতে পারেন
- Horizon metrics দিয়ে late event rates track করতে পারেন
- job retry এবং idempotency key combine করতে পারেন

---

## 🟨 JavaScript / Node.js Code

### Node.js example 1: Watermark tracking and sorted event buffer

```javascript
class WatermarkTracker {
  constructor({ maxOutOfOrderMs }) {
    this.maxOutOfOrderMs = maxOutOfOrderMs;
    this.maxEventTimeSeen = Number.NEGATIVE_INFINITY;
    this.currentWatermark = Number.NEGATIVE_INFINITY;
  }

  observe(event) {
    if (event.eventTimeMs > this.maxEventTimeSeen) {
      this.maxEventTimeSeen = event.eventTimeMs;
    }

    const candidate = this.maxEventTimeSeen - this.maxOutOfOrderMs;
    this.currentWatermark = Math.max(this.currentWatermark, candidate);

    return this.currentWatermark;
  }

  getWatermark() {
    return this.currentWatermark;
  }
}

class SortedEventBuffer {
  constructor() {
    this.items = [];
  }

  insert(event) {
    const index = this.items.findIndex((item) => item.eventTimeMs > event.eventTimeMs);

    if (index === -1) {
      this.items.push(event);
      return;
    }

    this.items.splice(index, 0, event);
  }

  drainReady(watermarkMs) {
    const ready = [];

    while (this.items.length > 0 && this.items[0].eventTimeMs <= watermarkMs) {
      ready.push(this.items.shift());
    }

    return ready;
  }

  size() {
    return this.items.length;
  }
}
```

### Node.js example 2: Window aggregation with late-event classification

```javascript
class EventTimeWindowProcessor {
  constructor({
    windowSizeMs,
    graceMs,
    watermarkTracker,
    sideOutput,
    dashboardEmitter,
  }) {
    this.windowSizeMs = windowSizeMs;
    this.graceMs = graceMs;
    this.watermarkTracker = watermarkTracker;
    this.sideOutput = sideOutput;
    this.dashboardEmitter = dashboardEmitter;

    this.windows = new Map();
    this.closedAt = new Map();
    this.seenEvents = new Set();
  }

  process(event) {
    if (this.seenEvents.has(event.eventId)) {
      return [{ type: 'duplicate_ignored', eventId: event.eventId }];
    }

    this.seenEvents.add(event.eventId);

    const watermark = this.watermarkTracker.observe(event);
    const { startMs, endMs } = this.windowFor(event.eventTimeMs);
    const key = this.windowKey(startMs, endMs);
    const graceEndMs = endMs + this.graceMs;

    if (!this.windows.has(key)) {
      this.windows.set(key, {
        count: 0,
        sum: 0,
        highValueCount: 0,
        users: new Set(),
        version: 0,
      });
    }

    const isClosed = this.closedAt.has(key);
    const isLate = event.eventTimeMs <= watermark;

    if (isClosed && isLate) {
      if (watermark > graceEndMs) {
        this.sideOutput.publish({
          reason: 'too_late',
          windowKey: key,
          event,
          watermarkMs: watermark,
        });

        return [{ type: 'late_side_output', key, eventId: event.eventId }];
      }

      const previous = this.snapshot(key);
      this.applyEvent(key, event);
      const current = this.snapshot(key);

      return [
        {
          type: 'retraction',
          key,
          value: previous,
        },
        {
          type: 'update',
          key,
          value: current,
        },
      ];
    }

    this.applyEvent(key, event);

    const emissions = [];
    for (const [windowKey] of this.windows.entries()) {
      const { endMs: candidateEndMs } = this.parseWindowKey(windowKey);

      if (watermark >= candidateEndMs && !this.closedAt.has(windowKey)) {
        this.closedAt.set(windowKey, Date.now());

        emissions.push({
          type: 'on_time_result',
          key: windowKey,
          value: {
            ...this.snapshot(windowKey),
            windowKey,
            firedAtWatermarkMs: watermark,
          },
        });

        this.dashboardEmitter.emit('window-result', emissions[emissions.length - 1]);
      }
    }

    return emissions;
  }

  applyEvent(key, event) {
    const aggregate = this.windows.get(key);

    aggregate.count += 1;
    aggregate.sum += event.amount;

    if (event.amount >= 15000) {
      aggregate.highValueCount += 1;
    }

    aggregate.users.add(event.userId);
    aggregate.version += 1;
  }

  snapshot(key) {
    const aggregate = this.windows.get(key);

    return {
      count: aggregate.count,
      sum: aggregate.sum,
      highValueCount: aggregate.highValueCount,
      uniqueUsers: aggregate.users.size,
      version: aggregate.version,
    };
  }

  windowFor(eventTimeMs) {
    const startMs = Math.floor(eventTimeMs / this.windowSizeMs) * this.windowSizeMs;
    return { startMs, endMs: startMs + this.windowSizeMs };
  }

  windowKey(startMs, endMs) {
    return `${startMs}:${endMs}`;
  }

  parseWindowKey(key) {
    const [startMs, endMs] = key.split(':').map(Number);
    return { startMs, endMs };
  }
}
```

### Node.js example 3: Late event side-output publisher

```javascript
class LateEventSideOutput {
  constructor({ producer, topic }) {
    this.producer = producer;
    this.topic = topic;
  }

  async publish({ reason, windowKey, event, watermarkMs }) {
    await this.producer.send({
      topic: this.topic,
      messages: [
        {
          key: event.userId,
          value: JSON.stringify({
            reason,
            windowKey,
            watermarkMs,
            eventId: event.eventId,
            eventTimeMs: event.eventTimeMs,
            ingestionTimeMs: event.ingestionTimeMs,
            payload: event,
          }),
          headers: {
            kind: 'late-event',
          },
        },
      ],
    });
  }
}
```

### Node.js example 4: Dashboard with correction and retraction handling

```javascript
class RetractionAwareDashboard {
  constructor() {
    this.store = new Map();
  }

  apply(message) {
    if (message.type === 'on_time_result' || message.type === 'update') {
      this.store.set(message.key, message.value);
      return;
    }

    if (message.type === 'retraction') {
      this.store.delete(message.key);
    }
  }

  renderTopWindows() {
    return [...this.store.entries()]
      .sort((a, b) => b[1].sum - a[1].sum)
      .slice(0, 5)
      .map(([windowKey, value]) => ({
        windowKey,
        count: value.count,
        sum: value.sum,
        version: value.version,
      }));
  }
}
```

### Node.js example 5: KafkaJS consumer with event-time processing

```javascript
const { Kafka } = require('kafkajs');
const EventEmitter = require('events');

async function startBkashFraudPipeline() {
  const kafka = new Kafka({
    clientId: 'bkash-fraud-event-time-pipeline',
    brokers: ['localhost:9092'],
  });

  const consumer = kafka.consumer({ groupId: 'bkash-fraud-group-v1' });
  const producer = kafka.producer();
  const dashboardEmitter = new EventEmitter();

  await consumer.connect();
  await producer.connect();

  await consumer.subscribe({
    topic: 'bkash.transactions.raw',
    fromBeginning: false,
  });

  const sideOutput = new LateEventSideOutput({
    producer,
    topic: 'bkash.transactions.late',
  });

  const watermarkTracker = new WatermarkTracker({
    maxOutOfOrderMs: 2 * 60 * 1000,
  });

  const processor = new EventTimeWindowProcessor({
    windowSizeMs: 5 * 60 * 1000,
    graceMs: 5 * 60 * 1000,
    watermarkTracker,
    sideOutput,
    dashboardEmitter,
  });

  const dashboard = new RetractionAwareDashboard();

  dashboardEmitter.on('window-result', (result) => {
    dashboard.apply(result);
    console.log('dashboard snapshot', dashboard.renderTopWindows());
  });

  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());

      const normalized = {
        eventId: event.event_id,
        userId: event.user_id,
        amount: Number(event.amount),
        type: event.type,
        eventTimeMs: Number(event.event_time_ms),
        ingestionTimeMs: Number(event.ingestion_time_ms || Date.now()),
      };

      const outputs = processor.process(normalized);

      for (const output of outputs) {
        dashboard.apply(output);

        if (output.type === 'on_time_result' || output.type === 'update') {
          await producer.send({
            topic: 'bkash.fraud.window-results',
            messages: [
              {
                key: output.key,
                value: JSON.stringify(output),
                headers: {
                  semantics: 'event-time',
                },
              },
            ],
          });
        }

        if (output.type === 'retraction') {
          await producer.send({
            topic: 'bkash.fraud.window-corrections',
            messages: [
              {
                key: output.key,
                value: JSON.stringify(output),
                headers: {
                  semantics: 'retraction',
                },
              },
            ],
          });
        }
      }

      await consumer.commitOffsets([
        {
          topic,
          partition,
          offset: (BigInt(message.offset) + 1n).toString(),
        },
      ]);
    },
  });
}

startBkashFraudPipeline().catch((error) => {
  console.error('pipeline failed', error);
  process.exit(1);
});
```

### Node.js example 6: Out-of-order reordering before windowing

```javascript
class ReorderingStage {
  constructor({ watermarkTracker, downstream }) {
    this.watermarkTracker = watermarkTracker;
    this.buffer = new SortedEventBuffer();
    this.downstream = downstream;
  }

  onEvent(event) {
    const watermark = this.watermarkTracker.observe(event);
    this.buffer.insert(event);

    const readyEvents = this.buffer.drainReady(watermark);
    const outputs = [];

    for (const readyEvent of readyEvents) {
      outputs.push(...this.downstream.process(readyEvent));
    }

    return outputs;
  }
}
```

### Node.js example 7: Real-time dashboard correction example

```javascript
async function emitDashboardCorrection(producer, key, oldValue, newValue) {
  await producer.send({
    topic: 'dashboard.corrections',
    messages: [
      {
        key,
        value: JSON.stringify({
          type: 'RETRACT',
          previous: oldValue,
        }),
      },
      {
        key,
        value: JSON.stringify({
          type: 'UPSERT',
          current: newValue,
        }),
      },
    ],
  });
}
```

### Node.js example 8: Full pipeline composition

```javascript
async function buildPathaoGpsPipeline() {
  const kafka = new Kafka({
    clientId: 'pathao-gps-pipeline',
    brokers: ['localhost:9092'],
  });

  const consumer = kafka.consumer({ groupId: 'pathao-gps-et-v1' });
  const producer = kafka.producer();

  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'pathao.driver.gps.raw', fromBeginning: false });

  const watermarkTracker = new WatermarkTracker({
    maxOutOfOrderMs: 5 * 60 * 1000,
  });

  const dashboardEmitter = new EventEmitter();
  const sideOutput = new LateEventSideOutput({
    producer,
    topic: 'pathao.driver.gps.late',
  });

  const processor = new EventTimeWindowProcessor({
    windowSizeMs: 60 * 1000,
    graceMs: 2 * 60 * 1000,
    watermarkTracker,
    sideOutput,
    dashboardEmitter,
  });

  const reorderStage = new ReorderingStage({
    watermarkTracker,
    downstream: processor,
  });

  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      const gps = JSON.parse(message.value.toString());

      const normalized = {
        eventId: gps.event_id,
        userId: gps.driver_id,
        amount: 1,
        type: 'gps_ping',
        eventTimeMs: Number(gps.captured_at_ms),
        ingestionTimeMs: Date.now(),
        lat: gps.lat,
        lng: gps.lng,
      };

      const results = reorderStage.onEvent(normalized);

      for (const result of results) {
        if (result.type === 'on_time_result' || result.type === 'update') {
          await producer.send({
            topic: 'pathao.driver.gps.minute-rollup',
            messages: [
              {
                key: result.key,
                value: JSON.stringify(result),
              },
            ],
          });
        }
      }

      await consumer.commitOffsets([
        {
          topic,
          partition,
          offset: (BigInt(message.offset) + 1n).toString(),
        },
      ]);
    },
  });
}
```

### Node.js example 9: Monitoring late event rate

```javascript
class StreamMetrics {
  constructor() {
    this.counters = {
      onTime: 0,
      lateWithinGrace: 0,
      tooLate: 0,
      duplicates: 0,
    };
  }

  mark(type) {
    if (this.counters[type] !== undefined) {
      this.counters[type] += 1;
    }
  }

  lateRate() {
    const total =
      this.counters.onTime +
      this.counters.lateWithinGrace +
      this.counters.tooLate;

    if (total === 0) {
      return 0;
    }

    return (this.counters.lateWithinGrace + this.counters.tooLate) / total;
  }
}
```

### Node.js implementation notes

এই Node.js code-গুলো production-ready blueprint হিসেবে ভাবা ভালো,
exact library-grade framework নয়।

এখানে যে pattern-গুলো গুরুত্বপূর্ণ:

- event-time aware normalization
- sorted buffer for mild reordering
- watermark-based release
- retraction-aware dashboard
- Kafka late side topic
- manual offset commit after processing
- duplicate guard
- correction stream

Node.js ecosystem-এ lightweight custom stream processors,
KafkaJS consumers,
BullMQ workers,
বা Redis Streams consumer group-এর ওপর এই pattern apply করা যায়।

---

## ✅ কখন ব্যবহার করবেন

### Event Time কখন ব্যবহার করবেন?

Event Time ব্যবহার করুন যখন:

- business truth source timestamp-এর ওপর নির্ভর করে
- out-of-order arrival common
- mobile/edge/offline events আছে
- replay-safe correctness দরকার
- fraud detection time sequence matter করে
- financial calculation exact হওয়া দরকার
- billing, payroll, settlement, compliance আছে
- user behavior analytics historically correct হওয়া দরকার

বাংলাদেশ context উদাহরণ:

- bKash transaction risk engine
- Nagad or wallet settlement
- Pathao/Foodpanda rider movement analytics
- Daraz flash sale conversion analysis
- Chaldal picking and dispatch timeline
- Prothom Alo pageview analytics
- GP cell utilization sampling

### Processing Time কখন ব্যবহার করবেন?

Processing Time ব্যবহার করুন যখন:

- প্রশ্নটি processing system-কেন্দ্রিক
- exact business order critical না
- ultra-low latency দরকার
- approximate trend enough
- late corrections undesirable
- ops metrics main concern

উদাহরণ:

- consumer throughput per second
- queue drain speed
- current system load
- live worker health
- temporary internal dashboard

### Ingestion Time কখন ব্যবহার করবেন?

Ingestion Time useful যখন:

- source clocks untrusted
- broker receive time meaningful
- ingestion boundary-র পরের latency কম
- moderate simplicity + moderate correctness চান

উদাহরণ:

- network edge logs
- partner systems with bad clocks
- partially trusted telemetry

---

### কোন late-event strategy বেছে নেবেন?

#### Drop
ব্যবহার করুন যখন:
- slight inaccuracy acceptable
- dashboard approximate
- small data loss okay

#### Grace Period
ব্যবহার করুন যখন:
- অধিকাংশ late event ধরতে চান
- dashboard একটু পরে update হলেও সমস্যা নেই
- mainstream production pipeline

#### Side Output
ব্যবহার করুন যখন:
- data loss চলবে না
- audit/compliance আছে
- later reconciliation করতে পারবেন

#### Retraction
ব্যবহার করুন যখন:
- exact live dashboard দরকার
- downstream changelog-aware
- consumer update/retract বুঝতে পারে

---

### Practical selection guide

| Requirement | Recommended choice |
|---|---|
| bKash fraud detection | Event Time + watermark + grace + late correction |
| Prothom Alo traffic dashboard | Event Time + early trigger + accumulating updates |
| Pathao GPS live heatmap | Event Time + bounded lateness + periodic firing |
| Foodpanda rider payout | Event Time + side output + batch reconciliation |
| Consumer lag monitoring | Processing Time |
| Telco broker arrival analytics | Ingestion Time or Event Time depending trust |

---

### Production best practices

#### 1. observed latency distribution measure করুন

Guess করে watermark delay set করবেন না।
প্রথমে measure করুন:

- p50 arrival delay
- p95 arrival delay
- p99 arrival delay
- worst region/device class
- retry burst pattern
- offline sync behavior

উদাহরণ:

- Dhaka users p95 delay = 8 sec
- rural GP users p95 delay = 55 sec
- low-end devices p99 delay = 4 min

এখন business SLA অনুযায়ী watermark choose করুন।

#### 2. one watermark for all sources দেবেন না

সব source সমান নয়।

আলাদা profile করুন:

- mobile app events
- POS terminal events
- backend generated events
- partner API events
- IoT/telemetry streams

#### 3. late event rate monitor করুন

Monitor metrics:

- late_events_total
- late_events_within_grace
- too_late_events_total
- corrections_emitted_total
- average watermark lag
- partition idle duration
- replay_count
- duplicate_event_rate

#### 4. alerting define করুন

Alert দিন যখন:

- too-late rate suddenly spikes
- one partition watermark stalls
- side-output volume abnormal
- skewed device timestamps increase
- grace updates surge
- replay frequency grows

#### 5. state TTL carefully tune করুন

window state, dedup state, and correction state কতক্ষণ রাখবেন তা business-driven।

too short TTL:
- late update impossible

too long TTL:
- memory and storage blow up

#### 6. correction contract downstream-এ স্পষ্ট করুন

Consumer জানবে কি?

- append-only?
- latest-wins?
- retract + upsert?
- versioned row?
- finality flag আছে?

এটি না জানালে dashboard inconsistency হবে।

#### 7. future timestamps guard করুন

device clock এগিয়ে থাকলে watermark jump করতে পারে।
সুতরাং validate করুন:

- event_time too far in future?
- negative lateness impossible?
- timezone parsing issue?

#### 8. dead-letter replay automation রাখুন

too-late topic শুধু জমিয়ে রাখবেন না।
batch or ad-hoc replay process রাখুন।

উদাহরণ:

- nightly reconciliation job
- hourly correction DAG
- finance close process
- audit export

#### 9. backfill এবং replay আলাদা code path রাখুন

same live pipeline replay করলে surprise behavior হতে পারে।
Backfill mode-এ:

- larger watermark delay
- disable early triggers
- emit final-only results
- dedup strict mode

#### 10. business stakeholders-কে semantics বুঝিয়ে দিন

Product manager, analyst, finance team, operations team-কে জানাতে হবে:

- dashboard number final না provisional?
- correction window কত?
- cut-off time কত?
- "today total" late correction পাবে কি?

Time semantics purely engineering topic নয়।
এটি business contract.

---

### A sample production recipe

#### bKash-like fraud analytics

- Event Time as primary
- watermark delay = 2 minutes
- window = 5 minutes
- grace = 5 minutes
- early trigger = প্রতি 30 sec
- on-time trigger = watermark
- too-late = side output
- dedup = transaction_id
- final settlement = batch reconcile

#### Pathao GPS analytics

- Event Time as primary
- reorder buffer = 30 sec
- watermark delay = 5 minutes
- minute windows for heatmap
- grace = 2 minutes
- side output for very late rural uploads
- dashboard uses latest-wins update

#### Prothom Alo pageview analytics

- Event Time
- aggressive early firing
- dashboard flagged as provisional
- correction window = 10 min
- hourly batch normalization for very late mobile sync

---

### Common mistakes to avoid

- Processing Time দিয়ে business KPI build করা
- watermark-কে guarantee ভাবা
- grace period ছাড়া financial window close করা
- late events silently drop করা
- idempotency ছাড়া replay allow করা
- clock skew ignore করা
- all sources একই lateness ধরে নেওয়া
- downstream consumers-কে correction semantics না জানানো
- watermark stall debug না করা
- idle partitions ignore করা

---

### Short decision checklist

নিজেকে প্রশ্ন করুন:

1. event বাস্তবে কবে ঘটেছে সেটা কি গুরুত্বপূর্ণ?
2. source delay কি variable?
3. replay কি possible?
4. exactness বনাম latency-র কোন পাশে business বেশি sensitive?
5. late update consumer সামলাতে পারবে?
6. too-late events হারালে business impact কত?
7. source clocks trustworthy কি?
8. batch reconciliation path আছে কি?

যদি ১, ২, ৩, ৬-এর উত্তর "হ্যাঁ" হয়,
তাহলে Event Time-centric design-এর দিকে যান।

---

## 🔚 Key Takeaways

- Stream processing-এ time semantics correctness-এর কেন্দ্রবিন্দু।
- Event Time business truth-এর সবচেয়ে কাছাকাছি।
- Processing Time সহজ হলেও delayed/out-of-order events-এ ভুল ফল দিতে পারে।
- Ingestion Time একটি pragmatic compromise।
- Watermark হলো heuristic progress signal, guarantee নয়।
- `watermark = max seen event time - delay` bounded disorder handle করার common strategy।
- multi-input pipeline-এ effective watermark সাধারণত slowest input দ্বারা নির্ধারিত হয়।
- watermark delay tuning মানে latency বনাম completeness trade-off।
- late event মানেই সবসময় drop নয়; grace, side output, retraction, accumulating সবই viable options।
- financial এবং audit domain-এ late events silently drop করা বিপজ্জনক।
- early triggers dashboard freshness বাড়ায়, but result provisional করে।
- late firing corrected results দিতে সাহায্য করে।
- clock skew mobile-heavy ecosystems-এ বাস্তব সমস্যা।
- exactly-once মানে শুধু broker-level duplicate prevention নয়, business-level idempotency-ও।
- replay-safe system করতে Event Time + dedup + versioned output খুব গুরুত্বপূর্ণ।
- bKash, Pathao, Daraz, Prothom Alo, Foodpanda, Chaldal, GP-এর মতো বাংলাদেশ context-এ Event Time reasoning বাস্তব ও প্রয়োজনীয়।
- production-এ observed delay distribution measure না করে watermark সেট করা উচিত নয়।
- late event rate, watermark lag, idle partition, correction volume monitor করতে হবে।
- অনেক mature system hybrid model ব্যবহার করে:
  real-time main pipeline + grace updates + late side stream + batch reconciliation।
- Time semantics ঠিকমতো না বুঝলে stream system "fast" হতে পারে,
  কিন্তু "correct" হবে না।
- আর stream processing-এ incorrect fast result অনেক সময় slow correct result-এর চেয়েও বেশি ক্ষতিকর।

---
