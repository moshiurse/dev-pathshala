# ♾️ Strategies for Processing Infinite Streams of Events — Tumbling, Sliding, Session Window গভীর বিশ্লেষণ

> Infinite event stream থেকে কীভাবে finite, meaningful, business-ready output বের করা যায় — এই ডকুমেন্টে সেই core problem-এর গভীর, বাস্তবমুখী, algorithmic এবং implementation-level বিশ্লেষণ করা হয়েছে।
>
> এখানে focus হলো window type-এর নাম মুখস্থ করা নয়।
>
> focus হলো understanding the strategy.
>
> অর্থাৎ:
>
> - infinite input থেকে finite result বের করার logic
> - state কীভাবে রাখা হয়
> - window close কখন হয়
> - late event এলে কী হয়
> - memory কেন explode করতে পারে
> - কোন business semantics কোন strategy-এর সাথে naturally fit করে
>
> উদাহরণে বাংলাদেশ context ব্যবহার করা হয়েছে:
>
> - bKash fraud detection
> - Daraz browsing analytics
> - Pathao surge pricing
> - Prothom Alo trending articles
> - GP billing / SMS usage
> - Chaldal demand spikes
> - Foodpanda order bursts

---

## 1. Infinite Stream Processing — সংজ্ঞা ও চ্যালেঞ্জ

### 1.1 Definition (সংজ্ঞা)

**Infinite stream** বা **unbounded stream** হলো এমন event sequence যার logical end নেই।

ডেটা আসতেই থাকবে।

থামবে না।

কখনও “সব data এসে গেছে” এই নিশ্চয়তা থাকবে না।

উদাহরণ:

- bKash transaction topic
- GP call detail event stream
- Pathao ride request events
- Daraz clickstream
- Prothom Alo page-view stream
- Chaldal order updates
- Foodpanda delivery status events

Traditional batch world-এ আমরা বলি:

- সব data collect করো
- তারপর process করো
- তারপর report বের করো

কিন্তু stream world-এ এই model ভেঙে যায়।

কারণ এখানে “সব data” বলে কিছু নেই।

Stream-এর tail continuously extend হতে থাকে।

### 1.2 কেন unbounded stream আলাদা

Bounded dataset:

- start আছে
- end আছে
- total size জানা বা eventually জানা যায়
- full sort করা যায়
- complete aggregate করা যায়

Unbounded dataset:

- start থাকতে পারে
- end নেই
- total size unknown
- “wait until complete” করা যায় না
- memory-তে সব রাখা যায় না
- result produce করতে হলে partial, incremental, bounded computation দরকার

### 1.3 Why we cannot simply “collect all data and process”

ধরুন Prothom Alo-এর সব page view collect করে “trending article” বের করতে চান।

যদি আপনি বলেন:

- first collect all page views
- then rank them

তাহলে সমস্যা হবে:

1. **All data never arrives**
2. **Result never emits**
3. **Memory grows without bound**
4. **Latency becomes infinite**
5. **Business value becomes zero**

Fraud detection-এর ক্ষেত্রে ৩০ মিনিট পরে fraud detect করা প্রায় useless হতে পারে।

Surge pricing-এর ক্ষেত্রে ১৫ মিনিট পরে demand spike detect করা মানে surge apply করার সময়ই শেষ।

Recommendation-এর ক্ষেত্রে পুরনো session analyse করলে user already চলে গেছে।

### 1.4 Core problem: finite output from infinite input

এটাই stream processing-এর central question:

**Infinite input থেকে finite output কীভাবে বের করব?**

Examples of desired finite outputs:

- প্রতি ৫ মিনিটে bKash account-wise transaction count
- last ১ hour-এ Prothom Alo article-wise view count
- user inactivity gap অনুযায়ী Daraz shopping session summary
- GP subscriber-wise hourly SMS usage
- Pathao zone-wise প্রতি মিনিটে ride demand summary

এই finite result তৈরি করার সবচেয়ে standard answer হলো:

## Windowing strategies

Window stream-কে logically bounded chunk-এ ভাগ করে।

তখন computation finite হয়।

State bounded হয়।

Emission meaningful হয়।

### 1.5 Windowing is not just grouping

অনেকে ভাবে window মানে শুধু “time range”।

আসলে windowing হলো:

- state boundary definition
- result emission contract
- completeness expectation
- lateness policy
- memory bound strategy
- business semantics mapping

একই stream-এর জন্য different window strategy different truth produce করতে পারে।

উদাহরণ:

একই bKash transaction stream থেকে:

- Tumbling window বলবে: প্রতি fixed ৫ মিনিটে কত transaction হলো
- Sliding window বলবে: যেকোনো continuous ৫ মিনিটে count কত
- Session window বলবে: bursty behavior বা activity cluster কেমন

### 1.6 Event time vs processing time

Infinite stream processing-এ আরেকটি critical dimension হলো time semantics।

#### Event time

যখন actual event ঘটেছে সেই timestamp।

#### Processing time

যখন processor event receive করেছে সেই timestamp।

#### Ingestion time

যখন broker / pipeline event ingest করেছে সেই timestamp।

এই distinction না বুঝলে window wrong result দিতে পারে।

উদাহরণ:

- user transaction করেছে 10:00:02-এ
- network delay-এর জন্য Kafka-তে এসেছে 10:00:25-এ
- processor consume করেছে 10:00:27-এ

কোন time অনুযায়ী window assign করবেন?

Business answer depends on use case.

Fraud detection usually event time prefer করে।

Operational monitoring processing time prefer করতে পারে।

### 1.7 Main trade-off triangle

Stream processing-এ সব কিছু একসাথে maximize করা যায় না।

#### Latency

কত দ্রুত result emit হবে?

#### Completeness

সব late event consider করা হবে কি?

#### Cost

কত memory, CPU, network, storage লাগবে?

আপনি latency কমালে completeness কমতে পারে।

আপনি completeness বাড়ালে state retention বাড়বে, cost বাড়বে।

আপনি cost কমালে maybe approximation বা coarse window নিতে হবে।

### 1.8 Finite result তৈরির standard approaches

Infinite stream থেকে finite result বের করার কিছু common strategy:

- Time window
- Count window
- Session window
- Trigger-based micro-batches
- Continuous approximation structures

এই ডকুমেন্টের focus:

- Tumbling Window
- Sliding Window
- Session Window

---

### Real-world examples (বাংলাদেশ context)

#### Example 1: bKash fraud detection

একটি account থেকে ৫ মিনিটে ৫টির বেশি transaction হলে alert।

এখানে fixed reporting দরকার হলে tumbling কাজ করতে পারে।

কিন্তু “যেকোনো ৫ মিনিটে” detect করতে চাইলে sliding দরকার।

আর bursty behavior cluster detect করতে চাইলে session useful।

#### Example 2: Pathao surge pricing

ঢাকার ধানমন্ডি zone-এ প্রতি ১ মিনিটে ride request count দরকার।

এটি tumbling window-এর classic example।

কিন্তু “last 5 minutes demand trend” দরকার হলে sliding better।

#### Example 3: Prothom Alo trending

“এই মুহূর্তে last 1 hour-এ কোন article trend করছে?”

এখানে tumbling boundary artifact create করতে পারে।

Sliding বেশি natural।

#### Example 4: Daraz shopping journey

User add-to-cart, search, product view, checkout attempt — inactivity gap এর ভিত্তিতে grouping দরকার।

এটি session window-এর natural fit।

#### Example 5: GP billing

Subscriber প্রতি hour-এ SMS count bill calculation-এর জন্য দরকার।

Fixed, auditable bucket চাইলে tumbling।

#### Example 6: Chaldal inventory pulse

“শেষ ৩০ মিনিটে Banani warehouse থেকে কত order বের হয়েছে?”

Manager real-time pulse দেখতে চাইলে sliding useful।

#### Example 7: Foodpanda courier activity

Courier active burst detect করতে gap-based grouping লাগতে পারে।

এখানে session window natural।

---

### ASCII Diagrams

#### Diagram 1: Infinite stream never ends

```text
Time ─────────────────────────────────────────────────────────────────────▶

Events:
E1   E2 E3     E4  E5 E6 E7    E8      E9 E10 E11 E12   E13   E14 ...

Batch mindset says:
[ collect all events ] -> process -> emit result

Problem:
[ collect all events ................................................. never ends ]
```

#### Diagram 2: Windowing converts never-ending input into bounded computations

```text
Infinite input stream
──────────────────────────────────────────────────────────────────────────▶

E1 E2 E3 E4 E5 E6 E7 E8 E9 E10 E11 E12 E13 E14 E15 ...

Windowing view:

|---- Window A ----|---- Window B ----|---- Window C ----|
   finite compute       finite compute      finite compute
      emit A               emit B              emit C
```

#### Diagram 3: Latency vs Completeness vs Cost

```text
                  Completeness
                       ▲
                       │
                       │  wait longer for late events
                       │
                       │
Cost ◀─────────────────┼─────────────────▶ Latency
more state,            │               faster emission,
more storage           │               but maybe incomplete
                       │
                       ▼
                 cheapest path usually
                 sacrifices accuracy or freshness
```

#### Diagram 4: Event time vs processing time

```text
Actual world timeline:
10:00:02  User sent money from bKash app

Network / broker timeline:
10:00:25  Event reached Kafka
10:00:27  Processor consumed event

Event-time window assignment     -> 10:00:02 bucket
Processing-time window assignment -> 10:00:27 bucket

Same event, different window, different truth.
```

#### Diagram 5: Finite results from infinite input

```text
Input stream:
[∞ events coming forever]

Question:
How do we output finite summaries?

Answer:
┌──────────────┐
│  Windowing   │
└──────┬───────┘
       │
       ├── Tumbling -> fixed report buckets
       ├── Sliding  -> recent moving view
       └── Session  -> activity bursts / behavior chunks
```

---

### PHP Code — Motivation Example

```php
<?php

final class InfiniteCollectorProblem
{
    private array $allEvents = [];

    public function ingest(array $event): void
    {
        // এই design unbounded stream-এর জন্য বিপজ্জনক
        $this->allEvents[] = $event;
    }

    public function totalAmount(): float
    {
        return array_reduce(
            $this->allEvents,
            fn (float $sum, array $event) => $sum + $event['amount'],
            0.0
        );
    }
}

final class WindowedCounter
{
    private int $windowSizeSeconds;
    private array $counts = [];

    public function __construct(int $windowSizeSeconds)
    {
        $this->windowSizeSeconds = $windowSizeSeconds;
    }

    public function ingest(array $event): void
    {
        $windowStart = $event['ts'] - ($event['ts'] % $this->windowSizeSeconds);
        $this->counts[$windowStart] = ($this->counts[$windowStart] ?? 0) + 1;
    }

    public function snapshot(): array
    {
        ksort($this->counts);
        return $this->counts;
    }
}

$processor = new WindowedCounter(300);

$events = [
    ['ts' => 1719223201, 'amount' => 500],
    ['ts' => 1719223210, 'amount' => 200],
    ['ts' => 1719223520, 'amount' => 100],
];

foreach ($events as $event) {
    $processor->ingest($event);
}

print_r($processor->snapshot());
```

এই code-এর core lesson:

- সব event memory-তে চিরদিন রাখা লাগছে না
- bounded summary রাখা হচ্ছে
- infinite input থেকেও finite state maintain করা যাচ্ছে

### JavaScript / Node.js Code — Motivation Example

```javascript
class WindowedCounter {
  constructor(windowSizeMs) {
    this.windowSizeMs = windowSizeMs;
    this.counts = new Map();
  }

  ingest(event) {
    const windowStart = event.ts - (event.ts % this.windowSizeMs);
    const current = this.counts.get(windowStart) || 0;
    this.counts.set(windowStart, current + 1);
  }

  snapshot() {
    return [...this.counts.entries()]
      .sort((a, b) => a[0] - b[0])
      .map(([windowStart, count]) => ({ windowStart, count }));
  }
}

const processor = new WindowedCounter(5 * 60 * 1000);

[
  { ts: Date.parse('2025-01-01T10:00:01Z'), amount: 500 },
  { ts: Date.parse('2025-01-01T10:03:21Z'), amount: 700 },
  { ts: Date.parse('2025-01-01T10:05:01Z'), amount: 900 },
].forEach((event) => processor.ingest(event));

console.log(processor.snapshot());
```

---

### When to use this mental model

যখন আপনি নিচের ধরনের প্রশ্ন শুনবেন, বুঝবেন windowing দরকার:

- প্রতি X মিনিটে summary চাই
- last N minutes trend চাই
- inactivity gap based user session চাই
- any rolling interval fraud detection চাই
- bounded state দরকার
- infinite stream থেকে dashboard metric বের করতে হবে

### Key Takeaways

- Infinite stream-এর end নেই
- তাই full collection based processing কাজ করে না
- Finite result বের করতে boundary দরকার
- Windowing সেই boundary define করে
- Tumbling = fixed non-overlapping buckets
- Sliding = overlapping recent view
- Session = inactivity gap based behavior chunk
- Latency, completeness, cost — তিনটিই trade-off driven
- Event time vs processing time ভুল বুঝলে wrong analytics হবে

---

## 2. Event-Stream Processing and Tumbling Window Strategy (DEEP DIVE)

### 2.1 Definition (সংজ্ঞা)

**Tumbling window** হলো fixed-size, non-overlapping window strategy।

Window size আগে থেকে নির্ধারিত থাকে।

প্রতিটি event ঠিক **একটি** window-তে belong করে।

কোনো overlap নেই।

Window fixed interval শেষে close হয়।

Examples:

- প্রতি ৫ মিনিটে transaction count
- প্রতি ১ মিনিটে ride requests per zone
- প্রতি ১ ঘণ্টায় SMS usage per subscriber
- প্রতি ১৫ মিনিটে Chaldal order volume report

### 2.2 Core intuition

Tumbling window stream-কে partition করে।

যেন infinite timeline-এ fixed-size বাক্স রাখা হয়েছে।

যে event যে box-এর মধ্যে পড়বে, সেই box-এই যাবে।

একটি event কখনও দুইটি tumbling window-তে যাবে না।

এটি auditable reporting-এর জন্য খুব useful।

কারণ bucket boundary deterministic।

### 2.3 Formal property

Given:

- window size = `W`
- event timestamp = `t`

Then:

- `windowStart = t - (t % W)`
- `windowEnd = windowStart + W`

Alternative form:

- `windowStart = floor(t / W) * W`

Event belongs to:

- `[windowStart, windowEnd)`

অর্থাৎ start inclusive, end exclusive.

এই convention boundary ambiguity কমায়।

### 2.4 Window lifecycle

Tumbling window lifecycle সাধারণত এই stages follow করে:

1. **Open**
2. **Accumulate**
3. **Close**
4. **Emit**
5. **Purge**

#### Open

সময়ের কোনো bucket first event পেলে window state create হয়।

#### Accumulate

একই bucket-এর event এলে aggregate update হয়।

#### Close

Window end time পার হলে window logically close হয়।

#### Emit

Final result downstream পাঠানো হয়।

#### Purge

State memory থেকে remove করা হয়, অথবা retention অনুযায়ী compact করা হয়।

### 2.5 State management: কী রাখব?

সব event raw form-এ রাখা বাধ্যতামূলক নয়।

Business query-এর উপর নির্ভর করে state minimal করা যায়।

#### If count দরকার

রাখুন:

- count

#### If sum / average দরকার

রাখুন:

- count
- sum
- maybe min
- maybe max

#### If unique user count দরকার

রাখতে পারেন:

- exact set (expensive)
- HyperLogLog (approximate)

#### If fraud rule needs payload inspection

রাখতে পারেন:

- count
- total amount
- max amount
- unique receiver count
- suspicious device flags

### 2.6 Why tumbling is operationally attractive

কারণ:

- overlap নেই
- duplication নেই
- per event single update
- easy to shard by key + window start
- predictable state growth
- billing/reporting-friendly
- downstream consumers simple output পায়

### 2.7 Boundary effect — tumbling-এর classic weakness

এটি tumbling-এর সবচেয়ে famous downside।

ধরুন rule:

- “৫ মিনিটে ৫টির বেশি transaction হলে suspicious”

Events এসেছে:

- 10:04:59
- 10:05:01
- 10:05:20
- 10:05:40
- 10:05:55

প্রথম event previous window-এ গেল।

বাকি ৪টি next window-এ গেল।

Actual burst continuous হলেও tumbling bucket split করে detection miss করতে পারে।

### 2.8 Tumbling window types

#### Time-based tumbling

Fixed duration অনুযায়ী bucket।

Example:

- প্রতি ৫ মিনিটে
- প্রতি ১ ঘণ্টায়
- প্রতি দিনে

#### Count-based tumbling

প্রতি N events এ window close।

Example:

- প্রতি 100 order event এ summary
- প্রতি 1000 click এ sample output

#### Keyed tumbling

একই time bucket-এর ভিতরে key অনুযায়ী আলাদা state।

Example:

- per bKash account প্রতি ৫ মিনিটে count
- per Pathao zone প্রতি ১ মিনিটে requests
- per GP subscriber প্রতি ঘণ্টায় SMS usage

---

### Real-world examples (Bangladesh context)

#### Example A: bKash transaction count per 5-minute window

Use case:

- প্রতি account-এর transaction count monitor করা
- ৫ মিনিট শেষে risk score generate করা
- fraud analyst dashboard-এ exact ৫ মিনিট bucket দেখানো

কেন tumbling?

- audit সহজ
- billing/reporting style summary সহজ
- duplicate counting নেই
- memory predictable

#### Example B: Pathao ride requests per zone per 1-minute window

ঢাকার zones:

- Dhanmondi
- Banani
- Gulshan
- Mirpur
- Uttara

প্রতি ১ মিনিটে request count বের করলে surge model feed করা যায়।

কেন tumbling?

- প্রতি মিনিটে consistent snapshot দরকার
- dashboard buckets clean
- operational visibility সহজ

#### Example C: GP SMS count per hour per subscriber

Billing বা policy monitoring-এ hour-aligned bucket দরকার।

এখানে tumbling perfect কারণ business semantics নিজেই fixed-hour based।

#### Example D: Chaldal warehouse order volume report

প্রতি ১৫ মিনিটে Banani warehouse কত orders packed হলো — এই report tumbling-এ naturally fit করে।

---

### ASCII Diagrams

#### Diagram 1: Time-based tumbling windows

```text
Window size = 5 minutes

Time  ─────────────────────────────────────────────────────────────────▶
       10:00        10:05        10:10        10:15        10:20
       |------------|------------|------------|------------|
       |   W1       |    W2      |    W3      |    W4      |
       |------------|------------|------------|------------|
       E1 E2 E3       E4 E5         E6           E7 E8 E9

Rule:
Every event belongs to exactly one window.
```

#### Diagram 2: Count-based tumbling windows

```text
Count size = 4 events

Stream:
E1 E2 E3 E4 E5 E6 E7 E8 E9 E10 E11 E12

Windows:
|---- W1 ----|---- W2 ----|---- W3 ----|
   4 events      4 events      4 events

W1 = [E1 E2 E3 E4]
W2 = [E5 E6 E7 E8]
W3 = [E9 E10 E11 E12]
```

#### Diagram 3: Key-based tumbling windows

```text
5-minute windows, keyed by accountId

Time bucket 10:00 - 10:05

account:A -> 3 transactions
account:B -> 1 transaction
account:C -> 5 transactions

Internal state can look like:

window=10:00
  A -> {count:3, sum:1200}
  B -> {count:1, sum: 200}
  C -> {count:5, sum:9000}
```

#### Diagram 4: Window lifecycle

```text
            first event arrives                   watermark / timer
                    │                                  │
                    ▼                                  ▼
               ┌────────┐   updates   ┌──────────┐  close  ┌──────┐
               │  Open  │────────────▶│Accumulate│────────▶│ Emit │
               └────────┘             └──────────┘         └──┬───┘
                                                                │
                                                                ▼
                                                             ┌──────┐
                                                             │Purge │
                                                             └──────┘
```

#### Diagram 5: Boundary effect

```text
Rule: more than 5 transactions in 5 minutes

Time: 10:00 ------------------- 10:05 ------------------- 10:10

Events:
           T1                         T2 T3 T4 T5 T6
           |                          |  |  |  |  |
           10:04:59                   10:05:01 ... 10:05:55

Tumbling view:
Window[10:00,10:05) -> count = 1
Window[10:05,10:10) -> count = 5

Continuous human view:
10:04:59 to 10:09:59 interval -> count = 6

This is why tumbling can miss boundary-crossing bursts.
```

#### Diagram 6: Aligned vs unaligned windows

```text
Aligned to epoch:
00:00-00:05, 00:05-00:10, 00:10-00:15

Custom business alignment:
09:30-09:35, 09:35-09:40, 09:40-09:45

Business calendars matter.
For GP billing, hour boundary may be strict.
For a campaign, custom start time may matter.
```

---

### Algorithm Deep Dive

#### 2.9 Window assignment algorithm

For time-based tumbling:

```text
windowStart = timestamp - (timestamp % windowSize)
windowEnd   = windowStart + windowSize
windowKey   = businessKey + ':' + windowStart
```

For keyed tumbling:

```text
state[businessKey][windowStart] = aggregate
```

For count-based tumbling:

```text
bucketIndex = floor(eventSequenceNumber / countSize)
```

#### 2.10 Processing algorithm

For each incoming event:

1. extract event time
2. compute windowStart
3. locate aggregate state
4. update aggregate
5. schedule close if necessary
6. on close, emit result
7. purge old state

#### 2.11 Complexity

Per event update cost:

- usually O(1)

Memory:

- O(activeKeys × activeWindows)

For tumbling,

activeWindows সাধারণত খুব ছোট।

Often only:

- current window
- previous window for lateness

#### 2.12 Late arrival handling

Late event tumbling-এ relatively simple।

Policy options:

##### Option 1: Drop late events

সবচেয়ে low-latency, low-cost path।

কিন্তু completeness কমে যায়।

##### Option 2: Allowed lateness

Window close হওয়ার পরও grace period রাখুন।

Example:

- window size = 5m
- allowed lateness = 1m
- final purge at windowEnd + 1m

##### Option 3: Re-emit corrected result

Late event এলে aggregate update করে revised output পাঠানো।

Downstream-এ upsert semantics লাগবে।

#### 2.13 Event time alignment details

সব tumbling window same boundary use করবে — এটা মনে হলেও অনেক design choice আছে:

- epoch aligned
- business day aligned
- local timezone aligned
- UTC aligned
- shift aligned (e.g. factory shift)

Bangladesh context-এ timezone-aware alignment important,

কারণ billing, reporting, campaign window local business clock অনুসরণ করতে পারে।

#### 2.14 Memory management in tumbling

What to store per window?

##### Minimal aggregation only

Pros:

- low memory
- fast updates

Cons:

- detailed forensic analysis impossible

##### Aggregation + raw samples

Pros:

- debugging easier
- explainability better

Cons:

- memory higher

##### Spill to Redis / RocksDB

Pros:

- larger scale handle করা যায়

Cons:

- higher IO latency

---

### PHP Code — TumblingWindowProcessor

```php
<?php

final class TumblingWindowProcessor
{
    private int $windowSizeSeconds;
    private int $allowedLatenessSeconds;
    private array $state = [];

    public function __construct(int $windowSizeSeconds, int $allowedLatenessSeconds = 0)
    {
        $this->windowSizeSeconds = $windowSizeSeconds;
        $this->allowedLatenessSeconds = $allowedLatenessSeconds;
    }

    public function process(array $event): array
    {
        $eventTime = $event['event_time'];
        $key = $event['key'];

        $windowStart = $eventTime - ($eventTime % $this->windowSizeSeconds);
        $windowEnd = $windowStart + $this->windowSizeSeconds;
        $stateKey = $this->buildStateKey($key, $windowStart);

        if (!isset($this->state[$stateKey])) {
            $this->state[$stateKey] = [
                'key' => $key,
                'window_start' => $windowStart,
                'window_end' => $windowEnd,
                'count' => 0,
                'sum' => 0.0,
                'min' => null,
                'max' => null,
                'last_event_time' => null,
            ];
        }

        $aggregate = &$this->state[$stateKey];
        $amount = (float) ($event['amount'] ?? 0);

        $aggregate['count']++;
        $aggregate['sum'] += $amount;
        $aggregate['min'] = $aggregate['min'] === null ? $amount : min($aggregate['min'], $amount);
        $aggregate['max'] = $aggregate['max'] === null ? $amount : max($aggregate['max'], $amount);
        $aggregate['last_event_time'] = $eventTime;

        return $aggregate;
    }

    public function closeExpiredWindows(int $watermark): array
    {
        $emitted = [];

        foreach ($this->state as $stateKey => $aggregate) {
            $finalCloseTime = $aggregate['window_end'] + $this->allowedLatenessSeconds;

            if ($watermark >= $finalCloseTime) {
                $emitted[] = [
                    'type' => 'tumbling_result',
                    'key' => $aggregate['key'],
                    'window_start' => $aggregate['window_start'],
                    'window_end' => $aggregate['window_end'],
                    'count' => $aggregate['count'],
                    'sum' => $aggregate['sum'],
                    'min' => $aggregate['min'],
                    'max' => $aggregate['max'],
                    'avg' => $aggregate['count'] > 0 ? $aggregate['sum'] / $aggregate['count'] : 0,
                ];

                unset($this->state[$stateKey]);
            }
        }

        return $emitted;
    }

    public function currentState(): array
    {
        return array_values($this->state);
    }

    private function buildStateKey(string $key, int $windowStart): string
    {
        return $key . ':' . $windowStart;
    }
}

$processor = new TumblingWindowProcessor(300, 60);

$events = [
    ['key' => 'bkash:acct:1001', 'event_time' => 1719223202, 'amount' => 500],
    ['key' => 'bkash:acct:1001', 'event_time' => 1719223210, 'amount' => 200],
    ['key' => 'bkash:acct:1002', 'event_time' => 1719223388, 'amount' => 300],
    ['key' => 'bkash:acct:1001', 'event_time' => 1719223511, 'amount' => 900],
];

foreach ($events as $event) {
    $processor->process($event);
}

print_r($processor->closeExpiredWindows(1719223560));
```

#### এই implementation কী শেখায়

- event → one window
- keyed state রাখা হচ্ছে
- aggregate raw event ছাড়াই maintain করা যাচ্ছে
- watermark / timer দিয়ে close করা হচ্ছে
- allowed lateness যোগ করা যায়

---

### JavaScript / Node.js Code — TumblingWindowProcessor

```javascript
class TumblingWindowProcessor {
  constructor(windowSizeMs, allowedLatenessMs = 0) {
    this.windowSizeMs = windowSizeMs;
    this.allowedLatenessMs = allowedLatenessMs;
    this.state = new Map();
  }

  buildStateKey(key, windowStart) {
    return `${key}:${windowStart}`;
  }

  process(event) {
    const eventTime = event.eventTime;
    const key = event.key;

    const windowStart = eventTime - (eventTime % this.windowSizeMs);
    const windowEnd = windowStart + this.windowSizeMs;
    const stateKey = this.buildStateKey(key, windowStart);

    if (!this.state.has(stateKey)) {
      this.state.set(stateKey, {
        key,
        windowStart,
        windowEnd,
        count: 0,
        sum: 0,
        min: null,
        max: null,
        lastEventTime: null,
      });
    }

    const aggregate = this.state.get(stateKey);
    const amount = Number(event.amount || 0);

    aggregate.count += 1;
    aggregate.sum += amount;
    aggregate.min = aggregate.min === null ? amount : Math.min(aggregate.min, amount);
    aggregate.max = aggregate.max === null ? amount : Math.max(aggregate.max, amount);
    aggregate.lastEventTime = eventTime;

    return aggregate;
  }

  closeExpiredWindows(watermark) {
    const results = [];

    for (const [stateKey, aggregate] of this.state.entries()) {
      const finalCloseTime = aggregate.windowEnd + this.allowedLatenessMs;

      if (watermark >= finalCloseTime) {
        results.push({
          type: 'tumbling_result',
          key: aggregate.key,
          windowStart: aggregate.windowStart,
          windowEnd: aggregate.windowEnd,
          count: aggregate.count,
          sum: aggregate.sum,
          min: aggregate.min,
          max: aggregate.max,
          avg: aggregate.count ? aggregate.sum / aggregate.count : 0,
        });

        this.state.delete(stateKey);
      }
    }

    return results;
  }
}

const processor = new TumblingWindowProcessor(5 * 60 * 1000, 60 * 1000);

[
  { key: 'pathao:zone:dhanmondi', eventTime: Date.parse('2025-01-01T10:00:10Z'), amount: 1 },
  { key: 'pathao:zone:dhanmondi', eventTime: Date.parse('2025-01-01T10:00:31Z'), amount: 1 },
  { key: 'pathao:zone:banani', eventTime: Date.parse('2025-01-01T10:02:00Z'), amount: 1 },
].forEach((event) => processor.process(event));

console.log(
  processor.closeExpiredWindows(Date.parse('2025-01-01T10:06:30Z'))
);
```

---

### When to use Tumbling Window

Tumbling choose করুন যদি:

- fixed reporting interval দরকার
- audit-friendly bucket দরকার
- overlap avoid করতে চান
- low computation cost দরকার
- exact one-bucket-per-event behavior চান
- billing / compliance / regular dashboarding use case থাকে

Typical examples:

- GP hourly billing
- bKash operational reports
- Pathao per-minute zone heatmap
- Chaldal warehouse throughput summary

### Advantages

- Simple mental model
- Simple implementation
- No event duplication across windows
- Predictable memory usage
- Easy partitioning and horizontal scaling
- Cheap per-event CPU cost
- Downstream consumers-এর জন্য clean output

### Disadvantages

- Boundary effects
- Fixed size সব business pattern capture করে না
- “last N minutes right now” query-তে less natural
- burst crossing boundary miss হতে পারে

### Key Takeaways

- Tumbling হলো reporting-friendly window
- প্রতিটি event exactly one bucket-এ যায়
- Formula simple এবং deterministic
- Memory/state usually predictable
- Fraud detection-এর some rules tumbling-এ miss করতে পারে because boundary effect
- Fixed interval business semantics থাকলে tumbling শক্তিশালী choice

---
## 3. Sliding Window Event-Stream Processing Strategy (DEEP DIVE)

### 3.1 Definition (সংজ্ঞা)

**Sliding window** হলো fixed-size window যা time-এর সাথে move করে।

এখানে focus হলো “recent interval”।

Common question:

- last 5 minutes
- last 30 minutes
- last 1 hour
- last 100 events

Tumbling-এর মতো disjoint bucket নয়।

Sliding-এ overlap হতে পারে।

একটি event multiple windows-এ contribute করতে পারে।

### 3.2 কেন sliding বেশি accurate মনে হয়

Human intuition অনেক সময় fixed calendar bucket follow করে না।

User জিজ্ঞেস করে:

- এখন last ৫ মিনিটে কী হলো?
- এখন last ১ ঘণ্টায় কী trend?
- এখন last ৩০ মিনিটে কে active?

এই প্রশ্ন rolling nature-এর।

Sliding এই rolling truth capture করে।

### 3.3 Sliding window-এর প্রধান ধরন

#### Type A: Time-based sliding window

Window size = X duration

Slide interval = Y duration

If Y < X, overlap হয়।

Example:

- size = 1 hour
- slide = 5 minutes

প্রতি ৫ মিনিটে last ১ hour aggregate emit হবে।

#### Type B: Event-triggered true sliding

প্রতিটি event আসলে current event time থেকে look-back interval evaluate করা হয়।

Example:

- current transaction time = 10:05:30
- look back 5 minutes
- consider events from 10:00:30 onward

#### Type C: Count-based sliding

Last N events consider করা হয়, time irrelevant।

Example:

- last 100 page views
- last 20 user actions

#### Type D: Hopping window

এটি sliding-এর special case।

- size fixed
- hop interval fixed
- hop interval < size হলে overlap হয়

Example:

- size = 10m
- hop = 2m

### 3.4 Hopping vs true sliding

#### Hopping

- compute at fixed hop interval
- cheaper
- easier to schedule
- granularity limited by hop size

#### True sliding

- logically continuous
- every event can trigger update
- most accurate for exact rolling question
- more expensive

### 3.5 Event duplication in sliding

Sliding-এর সবচেয়ে গুরুত্বপূর্ণ operational consequence হলো overlap।

একটি event multiple output window-এ count হতে পারে।

এটি bug নয়।

এটি semantics।

কিন্তু downstream consumer যদি unaware হয়,

তাহলে double counting interpretation issue হতে পারে।

### 3.6 Overlap factor

If:

- window size = W
- slide interval = S

Then approximate overlap factor:

- `overlapFactor = ceil(W / S)`

উদাহরণ:

- W = 60 min
- S = 5 min
- overlapFactor ≈ 12

মানে একটি event approx 12 windows-এ appear করতে পারে।

এটাই why sliding can be expensive.

### 3.7 Sliding-এর business superpower

Sliding perfect যখন query inherently rolling:

- last N minutes fraud detection
- last 1 hour trend
- last 30 min recommendation context
- last 15 min system load

Tumbling এ boundary artifact থাকে।

Sliding continuous truth-এর কাছাকাছি।

---

### Real-world examples (Bangladesh context)

#### Example A: Prothom Alo trending articles in last 1 hour

Trending article যদি 10:59-এ spike করে,

11:00 tumbling bucket switch-এ sudden reset misleading হতে পারে।

Sliding window last ১ hour trend continuous রাখে।

#### Example B: bKash fraud rule — more than 5 transactions in ANY 5-minute window

এই rule tumbling দিয়ে imperfect।

Sliding naturally fit করে।

কারণ fraudster bucket boundary মানে না।

#### Example C: Daraz items viewed in last 30 minutes

Recommendation engine জানতে চায় recent intent।

গত ৩০ মিনিটে user কী কী category দেখেছে?

Sliding naturally captures that context.

#### Example D: Chaldal live demand pulse

Warehouse manager জানতে চায়:

- এখন last ১৫ মিনিটে rice orders বাড়ছে নাকি কমছে?

এটি sliding query।

#### Example E: Foodpanda live kitchen pressure

রেস্টুরেন্টভিত্তিক last ১০ মিনিটে order inflow sliding window-এ দেখলে kitchen overload early detect করা যায়।

---

### ASCII Diagrams

#### Diagram 1: Overlapping windows visualization

```text
Window size = 10 min
Slide = 5 min

Time  ───────────────────────────────────────────────────────────────▶
       10:00   10:05   10:10   10:15   10:20   10:25

W1:   [10:00 ---------------- 10:10)
W2:          [10:05 ---------------- 10:15)
W3:                 [10:10 ---------------- 10:20)
W4:                        [10:15 ---------------- 10:25)
```

#### Diagram 2: Event appears in multiple windows

```text
Event E arrives at 10:09

W1 = [10:00, 10:10) -> includes E
W2 = [10:05, 10:15) -> includes E

Same event contributes to multiple windows because overlap exists.
```

#### Diagram 3: True sliding look-back

```text
Current event at 10:05:30
Look-back size = 5 minutes

Relevant interval:
[10:00:30 ---------------------- 10:05:30]

Every new event shifts the left and right boundary.
No fixed bucket edges are visible to end user.
```

#### Diagram 4: Memory growth intuition

```text
Tumbling:
Each event -> one aggregate update

Sliding with overlap:
Each event -> multiple logical windows / or one recent-event structure

Naive implementation memory:
E1 stored in W1 W2 W3 ...
E2 stored in W2 W3 W4 ...

Optimized implementation:
Store recent events once
Evict by time
Compute rolling aggregate incrementally
```

#### Diagram 5: Hopping vs sliding

```text
Hopping (size=10m, hop=5m)
[00-10] [05-15] [10-20] [15-25]

True sliding
[00:00-10:00]
[00:01-10:01]
[00:02-10:02]
[00:03-10:03]
...
[05:37-15:37]

True sliding is conceptually continuous.
Hopping is sampled sliding.
```

#### Diagram 6: Fraud burst across bucket boundary

```text
Transactions:
10:04:59 10:05:01 10:05:10 10:05:20 10:05:40 10:05:50

Tumbling 5m:
[10:00-10:05) -> 1
[10:05-10:10) -> 5

Sliding 5m at 10:05:50:
[10:00:50-10:05:50] -> 6

Sliding catches the burst cleanly.
```

---

### 3.8 Algorithms Deep Dive

Sliding window implementation theoretically simple,

practically tricky।

কারণ overlap manage করতে হয়।

#### Algorithm A: Ring buffer / circular buffer

Best when:

- count-based sliding
- small time granularity buckets
- approximate or discretized time okay

Idea:

- fixed array use করা
- head / tail pointer move করা
- expired bucket overwrite করা

Pros:

- very fast
- low allocation

Cons:

- exact event-time precision limited হতে পারে
- sparse timestamps handle কঠিন হতে পারে

#### Algorithm B: Sorted set (Redis ZSET)

Best when:

- time-based sliding
- distributed state দরকার
- expiration by timestamp দরকার

Idea:

- score = eventTime
- member = eventId বা unique composite
- `ZADD` দিয়ে insert
- `ZREMRANGEBYSCORE` দিয়ে পুরনো event delete
- `ZCOUNT` / `ZRANGEBYSCORE` দিয়ে current window query

Pros:

- exact time ordering
- distributed and shared state
- natural eviction by score

Cons:

- network IO বেশি
- high-cardinality key-এ expensive হতে পারে

#### Algorithm C: Expire-based rolling aggregate

Idea:

- per event insert + expiry schedule
- add on arrival, subtract on expiry

This creates rolling sum/count efficiently.

Pros:

- true incremental
- low query cost

Cons:

- expiry scheduling complexity
- clock skew / late event tricky

#### Algorithm D: Monotonic deque for min/max

If query is last N window min/max,

monotonic deque efficient option।

Pros:

- O(1) amortized update

Cons:

- only specific aggregates
- generic aggregation নয়

### 3.9 Deduplication challenge

Sliding output overlap মানে logical duplication।

কিন্তু source stream-এ actual duplicate event থাকলে আরও সমস্যা।

Need:

- idempotent event ID
- dedup cache / seen set
- TTL matching window size

Otherwise one duplicate event can distort multiple overlapping outputs.

### 3.10 Emission policy in sliding

Result emit কবে করবেন?

#### On every event

Pros:

- freshest result

Cons:

- high output rate
- downstream load বেশি

#### Periodic emit

Example:

- compute rolling window continuously
- emit প্রতি 10 seconds

Balanced option।

#### Threshold-triggered emit

Only when significant change হয়।

Useful for alerts.

### 3.11 Memory bound strategies

Sliding memory growth control করার জন্য:

- maxTime enforce করুন
- maxEvents cap রাখুন
- compacted buckets use করুন
- approximate data structure use করুন
- key cardinality limit করুন

### 3.12 Late arrival in sliding

Late event tumbling-এর চেয়ে harder,

কারণ many overlapping outputs impacted হতে পারে।

Policy options:

- drop late event
- update current structure only if still inside active look-back
- re-emit corrected downstream outputs
- maintain grace window and upsert semantics

---

### PHP Code — SlidingWindowProcessor with Redis ZSET

```php
<?php

use Predis\Client;

final class SlidingWindowProcessor
{
    public function __construct(
        private Client $redis,
        private int $windowSizeSeconds,
        private int $emitEverySeconds = 10,
    ) {
    }

    public function ingest(string $key, array $event): array
    {
        $streamKey = $this->streamKey($key);
        $eventTime = $event['event_time'];
        $eventId = $event['event_id'];
        $amount = (float) ($event['amount'] ?? 0);

        $member = json_encode([
            'id' => $eventId,
            'amount' => $amount,
        ], JSON_THROW_ON_ERROR);

        $this->redis->zadd($streamKey, [$member => $eventTime]);
        $this->redis->zremrangebyscore($streamKey, '-inf', $eventTime - $this->windowSizeSeconds);
        $this->redis->expire($streamKey, $this->windowSizeSeconds * 2);

        $windowStart = $eventTime - $this->windowSizeSeconds;
        $members = $this->redis->zrangebyscore($streamKey, $windowStart, $eventTime);

        $count = count($members);
        $sum = 0.0;

        foreach ($members as $encoded) {
            $payload = json_decode($encoded, true, 512, JSON_THROW_ON_ERROR);
            $sum += (float) ($payload['amount'] ?? 0);
        }

        return [
            'type' => 'sliding_result',
            'key' => $key,
            'window_start' => $windowStart,
            'window_end' => $eventTime,
            'count' => $count,
            'sum' => $sum,
            'avg' => $count > 0 ? $sum / $count : 0,
        ];
    }

    private function streamKey(string $key): string
    {
        return 'sliding:' . $key;
    }
}
```

#### কেন Redis ZSET sliding-এ useful

- score দিয়ে timestamp ordered storage
- recent range query easy
- eviction by score natural
- distributed consumer group shared state possible

#### Caveat

Production scale-এ per event full range scan expensive হতে পারে।

তাই অনেক system incremental aggregates, bucketed sums, বা approximate sketches use করে।

---

### JavaScript / Node.js Code — Sliding Window with Redis Sorted Set

```javascript
import Redis from 'ioredis';

class SlidingWindowProcessor {
  constructor({ redis, windowSizeMs, emitEveryMs = 10_000 }) {
    this.redis = redis;
    this.windowSizeMs = windowSizeMs;
    this.emitEveryMs = emitEveryMs;
  }

  streamKey(key) {
    return `sliding:${key}`;
  }

  async ingest(key, event) {
    const redisKey = this.streamKey(key);
    const eventTime = event.eventTime;
    const member = JSON.stringify({
      id: event.eventId,
      amount: Number(event.amount || 0),
    });

    await this.redis.zadd(redisKey, eventTime, member);
    await this.redis.zremrangebyscore(redisKey, '-inf', eventTime - this.windowSizeMs);
    await this.redis.pexpire(redisKey, this.windowSizeMs * 2);

    const members = await this.redis.zrangebyscore(
      redisKey,
      eventTime - this.windowSizeMs,
      eventTime
    );

    const parsed = members.map((item) => JSON.parse(item));
    const count = parsed.length;
    const sum = parsed.reduce((acc, item) => acc + Number(item.amount || 0), 0);

    return {
      type: 'sliding_result',
      key,
      windowStart: eventTime - this.windowSizeMs,
      windowEnd: eventTime,
      count,
      sum,
      avg: count ? sum / count : 0,
    };
  }
}

const redis = new Redis();
const processor = new SlidingWindowProcessor({
  redis,
  windowSizeMs: 5 * 60 * 1000,
});

const result = await processor.ingest('bkash:acct:1001', {
  eventId: 'txn-100',
  eventTime: Date.now(),
  amount: 1500,
});

console.log(result);
```

---

### When to use Sliding Window

Sliding choose করুন যদি question rolling হয়:

- last N minutes কী হলো?
- any 5-minute burst detect করতে হবে
- trending needs freshness
- recommendation needs recent context
- boundary effect unacceptable

### Advantages

- “recent N time” query-এর জন্য most natural
- boundary effect নেই বা অনেক কম
- anomaly / burst detection-এ শক্তিশালী
- real-time trend detection better

### Disadvantages

- overlap-এর জন্য higher computation
- memory বেশি লাগতে পারে
- emission frequency decision কঠিন
- late events many outputs affect করতে পারে
- implementation more complex than tumbling

### Key Takeaways

- Sliding = rolling truth
- Hopping = sampled sliding
- True sliding = most accurate, most expensive
- Overlap factor cost drive করে
- Redis ZSET, ring buffer, incremental expiry — common implementations
- Fraud, trending, recency-driven analytics-এ sliding powerful choice

---
## 4. Session Window Event-Stream Processing Strategy (DEEP DIVE)

### 4.1 Definition (সংজ্ঞা)

**Session window** হলো variable-size window যা **activity gap** দিয়ে define হয়।

এখানে fixed start/end আগে থেকে জানা থাকে না।

Window grow করতে থাকে যতক্ষণ event flow চলছে।

যখন configured inactivity gap-এর বেশি gap দেখা যায়,

তখন current session close হয়,

নতুন session শুরু হয়।

### 4.2 কেন session human behavior-এর সাথে natural match করে

মানুষ fixed ৫ মিনিট bucket-এ কাজ করে না।

User shopping session, reading session, driver active session — এগুলো behavior burst।

Session window এই natural burst capture করে।

### 4.3 Formal rule

Given per-key ordered events:

- current event time = `t`
- previous event time = `prev`
- inactivity gap = `G`

If:

- `t - prev <= G` -> same session
- `t - prev > G`  -> close previous session, open new session

### 4.4 Session window is key-scoped

Session almost always keyed হয়।

কারণ one global session rarely meaningful।

Examples:

- per user session
- per device session
- per driver session
- per merchant session
- per account session

### 4.5 Session lifecycle

1. first event আসে
2. new session open হয়
3. more events আসে within gap
4. session extend হয়
5. gap exceeded
6. session close হয়
7. emit summary

### 4.6 Why session is harder than tumbling/sliding

কারণ:

- size unknown
- end unknown until inactivity observed
- memory unpredictable
- late event two sessions merge করতে পারে
- already-emitted result retract করতে হতে পারে

### 4.7 Session merging problem

Session window-এর সবচেয়ে advanced challenge হলো **merge**।

ধরুন gap = 10 minutes।

Existing sessions:

- S1 = 10:00 to 10:04
- S2 = 10:20 to 10:25

Late event এলো 10:12-এ।

If gap semantics অনুযায়ী:

- 10:12 is within 10 minutes of S1 end (10:04 -> 10:12 = 8m)
- 10:12 is within 10 minutes of S2 start (10:12 -> 10:20 = 8m)

তাহলে S1 এবং S2 merge হয়ে এক session হতে পারে:

- S = 10:00 to 10:25

This is operationally complex.

কারণ:

- old emitted outputs invalid হয়ে গেল
- new merged output emit করতে হবে
- downstream retract / upsert support লাগবে

### 4.8 Session window vs sliding window

Sliding বলে:

- last N time recent context দেখো

Session বলে:

- natural activity burst কোথায় শুরু-শেষ হলো দেখো

দুইটিই recent behavior নিয়ে কাজ করে,

কিন্তু semantics আলাদা।

---

### Real-world examples (Bangladesh context)

#### Example A: Daraz shopping session analysis

User ৩০ মিনিট gap-এর মধ্যে যদি product view, search, add-to-cart, checkout করে,

সেগুলো same shopping session।

Use cases:

- conversion funnel
- abandonment analysis
- recommendation context
- campaign attribution

#### Example B: Prothom Alo reading session

Reader ১০ মিনিট gap-এর মধ্যে multiple article পড়ছে।

এটি one reading session।

Metrics:

- articles per session
- average session duration
- topic transition within session

#### Example C: Pathao driver active session

Driver ride accept / arrive / drop / accept pattern-এর মধ্যে যদি ১৫ মিনিটের বেশি inactivity না থাকে,

তাহলে এটি one active session।

#### Example D: bKash suspicious transaction burst

এক account থেকে ৫ মিনিট gap-এর মধ্যে বারবার send money হচ্ছে।

এটি session-based suspicious burst হিসেবে analyze করা যায়।

#### Example E: Foodpanda courier online burst

Courier app online actions, pickup, dropoff, status updates — gap based grouping active work session বুঝতে সাহায্য করে।

---

### ASCII Diagrams

#### Diagram 1: Session creation and close

```text
Gap = 5 min

Time ─────────────────────────────────────────────────────────────▶

E1   E2   E3          E4   E5                 E6
|    |    |           |    |                  |

Session 1: [E1 E2 E3]
          gap > 5m
Session 2: [E4 E5]
          gap > 5m
Session 3: [E6]
```

#### Diagram 2: Gap-based boundaries

```text
Events for user U1:
10:00 10:03 10:07 10:19 10:21 10:40
Gap = 10m

10:00 -> start S1
10:03 -> same S1
10:07 -> same S1
10:19 -> new S2 because 12m gap from 10:07
10:21 -> same S2
10:40 -> new S3 because 19m gap from 10:21
```

#### Diagram 3: Session merging scenario

```text
Gap = 10m

Existing sessions:
S1: [10:00 -------- 10:04]
S2:                     [10:20 -------- 10:25]

Late event arrives at 10:12

10:04 -> 10:12 = 8m  (connects to S1)
10:12 -> 10:20 = 8m  (connects to S2)

Merged result:
S*: [10:00 ----------------------------- 10:25]
```

#### Diagram 4: Multiple users' sessions in parallel

```text
Gap = 15m

User A: 10:00 10:05 10:09      10:40 10:45
        |------ Session A1 ----|      |-A2-|

User B: 10:02        10:20 10:25 10:28
        |-B1-|       |----- Session B2 ----|

Keyed sessioning means each user has separate session state.
```

#### Diagram 5: Why session can stay open unpredictably

```text
If user keeps producing events every 4 minutes,
and gap threshold = 5 minutes,
then session never closes.

10:00 10:04 10:08 10:12 10:16 10:20 10:24 ...

One long session keeps extending.
```

---

### 4.9 Algorithm Deep Dive

#### Basic gap detection algorithm

For each key:

1. find current open session
2. compare incoming event time with session lastEventTime
3. if gap <= threshold, extend session
4. else close old session, emit, create new session

#### Pseudocode

```text
if no open session for key:
    create session
else if event.time - lastEvent.time <= gap:
    extend session
else:
    emit old session
    start new session
```

#### Handling out-of-order events

If events arrive late,

simple append logic breaks.

Need options:

- keep sessions ordered by start/end
- insert event into correct session if within gap
- if it bridges two sessions, merge them
- re-emit corrected output

#### Session state contents

Per session you may store:

- sessionId
- key / userId
- startTime
- endTime
- eventCount
- sumAmount
- eventTypesSeen
- lastEventTime
- maybe raw sample events

#### Session expiry / TTL

Because session can remain open,

TTL strategy দরকার:

- idle TTL = gap + grace
- hard TTL = max session duration
- safety TTL = store cleanup fallback

### 4.10 Session ID generation

Options:

- key + startTime
- UUID assigned on open
- deterministic hash of key + firstEventId

Caveat:

merge হলে session ID change করতে হতে পারে,

অথবা canonical session reference maintain করতে হয়।

### 4.11 Handling very long sessions

Session indefinite হলে problems:

- memory leak risk
- user bot / noisy device skew metrics
- state recovery expensive

Mitigation:

- max session duration limit
- max event count per session
- session split policy
- spill raw events to external storage

---

### PHP Code — SessionWindowProcessor

```php
<?php

final class SessionWindowProcessor
{
    private int $gapSeconds;
    private array $sessionsByKey = [];

    public function __construct(int $gapSeconds)
    {
        $this->gapSeconds = $gapSeconds;
    }

    public function process(array $event): array
    {
        $key = $event['key'];
        $time = $event['event_time'];
        $amount = (float) ($event['amount'] ?? 0);

        if (!isset($this->sessionsByKey[$key])) {
            $this->sessionsByKey[$key] = [];
        }

        $sessions = &$this->sessionsByKey[$key];

        foreach ($sessions as &$session) {
            if ($time >= $session['start_time'] - $this->gapSeconds && $time <= $session['end_time'] + $this->gapSeconds) {
                $session['start_time'] = min($session['start_time'], $time);
                $session['end_time'] = max($session['end_time'], $time);
                $session['event_count']++;
                $session['sum_amount'] += $amount;
                $session['last_event_time'] = max($session['last_event_time'], $time);

                $this->mergeOverlappingSessions($sessions);
                return $session;
            }
        }

        $sessions[] = [
            'session_id' => $key . ':' . $time,
            'key' => $key,
            'start_time' => $time,
            'end_time' => $time,
            'event_count' => 1,
            'sum_amount' => $amount,
            'last_event_time' => $time,
        ];

        usort($sessions, fn (array $a, array $b) => $a['start_time'] <=> $b['start_time']);
        return end($sessions);
    }

    public function closeInactiveSessions(int $watermark): array
    {
        $emitted = [];

        foreach ($this->sessionsByKey as $key => &$sessions) {
            $active = [];

            foreach ($sessions as $session) {
                if (($watermark - $session['last_event_time']) > $this->gapSeconds) {
                    $emitted[] = [
                        'type' => 'session_result',
                        'session_id' => $session['session_id'],
                        'key' => $session['key'],
                        'start_time' => $session['start_time'],
                        'end_time' => $session['end_time'],
                        'duration_seconds' => $session['end_time'] - $session['start_time'],
                        'event_count' => $session['event_count'],
                        'sum_amount' => $session['sum_amount'],
                    ];
                } else {
                    $active[] = $session;
                }
            }

            $sessions = $active;
        }

        return $emitted;
    }

    private function mergeOverlappingSessions(array &$sessions): void
    {
        if (count($sessions) <= 1) {
            return;
        }

        usort($sessions, fn (array $a, array $b) => $a['start_time'] <=> $b['start_time']);

        $merged = [];
        $current = array_shift($sessions);

        foreach ($sessions as $session) {
            if ($session['start_time'] <= $current['end_time'] + $this->gapSeconds) {
                $current['end_time'] = max($current['end_time'], $session['end_time']);
                $current['event_count'] += $session['event_count'];
                $current['sum_amount'] += $session['sum_amount'];
                $current['last_event_time'] = max($current['last_event_time'], $session['last_event_time']);
            } else {
                $merged[] = $current;
                $current = $session;
            }
        }

        $merged[] = $current;
        $sessions = $merged;
    }
}
```

---

### JavaScript / Node.js Code — SessionWindowProcessor

```javascript
class SessionWindowProcessor {
  constructor(gapMs) {
    this.gapMs = gapMs;
    this.sessionsByKey = new Map();
  }

  process(event) {
    const { key, eventTime } = event;
    const amount = Number(event.amount || 0);

    if (!this.sessionsByKey.has(key)) {
      this.sessionsByKey.set(key, []);
    }

    const sessions = this.sessionsByKey.get(key);

    for (const session of sessions) {
      if (
        eventTime >= session.startTime - this.gapMs &&
        eventTime <= session.endTime + this.gapMs
      ) {
        session.startTime = Math.min(session.startTime, eventTime);
        session.endTime = Math.max(session.endTime, eventTime);
        session.eventCount += 1;
        session.sumAmount += amount;
        session.lastEventTime = Math.max(session.lastEventTime, eventTime);

        this.mergeOverlappingSessions(sessions);
        return session;
      }
    }

    const newSession = {
      sessionId: `${key}:${eventTime}`,
      key,
      startTime: eventTime,
      endTime: eventTime,
      eventCount: 1,
      sumAmount: amount,
      lastEventTime: eventTime,
    };

    sessions.push(newSession);
    sessions.sort((a, b) => a.startTime - b.startTime);
    return newSession;
  }

  closeInactiveSessions(watermark) {
    const results = [];

    for (const [key, sessions] of this.sessionsByKey.entries()) {
      const active = [];

      for (const session of sessions) {
        if (watermark - session.lastEventTime > this.gapMs) {
          results.push({
            type: 'session_result',
            sessionId: session.sessionId,
            key: session.key,
            startTime: session.startTime,
            endTime: session.endTime,
            durationMs: session.endTime - session.startTime,
            eventCount: session.eventCount,
            sumAmount: session.sumAmount,
          });
        } else {
          active.push(session);
        }
      }

      this.sessionsByKey.set(key, active);
    }

    return results;
  }

  mergeOverlappingSessions(sessions) {
    if (sessions.length <= 1) return;

    sessions.sort((a, b) => a.startTime - b.startTime);

    const merged = [];
    let current = sessions[0];

    for (let i = 1; i < sessions.length; i += 1) {
      const next = sessions[i];

      if (next.startTime <= current.endTime + this.gapMs) {
        current = {
          ...current,
          endTime: Math.max(current.endTime, next.endTime),
          eventCount: current.eventCount + next.eventCount,
          sumAmount: current.sumAmount + next.sumAmount,
          lastEventTime: Math.max(current.lastEventTime, next.lastEventTime),
        };
      } else {
        merged.push(current);
        current = next;
      }
    }

    merged.push(current);
    sessions.splice(0, sessions.length, ...merged);
  }
}
```

---

### When to use Session Window

Session choose করুন যদি:

- user behavior burst analyze করতে চান
- inactivity gap business meaning রাখে
- fixed bucket unnatural লাগে
- journey / funnel / engagement study করতে চান
- bursty fraud cluster ধরতে চান

### Advantages

- human / user behavior-এর সাথে natural mapping
- variable length real-world irregularity handle করে
- session metrics powerful insights দেয়
- engagement analytics-এ অসাধারণ useful

### Disadvantages

- memory usage unpredictable
- session merge complexity high
- late events painful
- final close time unknown until inactivity observed
- downstream retraction support লাগতে পারে

### Key Takeaways

- Session = gap-based variable window
- fixed size নেই
- behavior modeling-এর জন্য best
- late event session merge create করতে পারে
- production-grade sessioning-এ watermark, grace, merge semantics very important

---

## 5. Comparison of All Window Strategies

### 5.1 Decision Matrix

| Criterion | Tumbling | Sliding | Session |
|-----------|----------|---------|---------|
| Main semantics | Fixed reporting bucket | Rolling recent interval | Activity gap based cluster |
| Window size | Fixed | Fixed | Variable |
| Overlap | No | Yes / possible | No fixed overlap notion |
| Event duplication across outputs | No | Yes | No, but merge may revise |
| Memory | Predictable | Higher | Unpredictable |
| CPU cost | Low | Medium to High | Medium to High |
| Late event handling | Simple | Medium | Complex |
| Best for | Billing/reporting | Recent trend / burst detection | User behavior / sessions |
| Boundary effect | High | Low | Not applicable in same way |
| Implementation complexity | Low | Medium | High |
| Result stability | High | Frequent updates | Can change due to merge |
| Natural example | GP hourly billing | Prothom Alo trending last 1h | Daraz shopping session |

### 5.2 Quick intuition table

| প্রশ্ন | Window Strategy |
|-------|------------------|
| প্রতি ৫ মিনিটে count চাই | Tumbling |
| last ৫ মিনিটে count চাই | Sliding |
| inactivity gap অনুযায়ী session চাই | Session |
| hourly billing চাই | Tumbling |
| recent fraud burst detect চাই | Sliding |
| user journey চাই | Session |

### 5.3 Same stream, different truths

ধরুন একই bKash transaction stream।

Tumbling answer:

- 10:00-10:05 bucket count = 4
- 10:05-10:10 bucket count = 3

Sliding answer at 10:07:

- last 5 minutes count = 6

Session answer:

- suspicious burst session count = 7, duration = 6m

সব result correct হতে পারে।

কারণ question আলাদা।

### 5.4 Combining strategies

#### Tumbling + Session

Use case:

- Daraz user sessions detect করুন
- তারপর প্রতি ১ ঘণ্টায় completed sessions-এর tumbling report বের করুন

#### Sliding + Tumbling

Use case:

- bKash sliding window দিয়ে fraud detect করুন
- tumbling window দিয়ে regular compliance report generate করুন

#### Session + Sliding

Use case:

- Prothom Alo reader session detect করুন
- last ৩০ মিনিটে active reading sessions sliding metric হিসেবেও দেখান

### 5.5 Decision questions checklist

নিজেকে জিজ্ঞেস করুন:

1. question fixed bucket নাকি rolling?
2. inactivity gap business meaning রাখে কি?
3. exact audit bucket লাগবে কি?
4. late event tolerance কত?
5. memory budget কেমন?
6. downstream upsert/retraction support আছে কি?
7. user-facing freshness কত critical?

---

## 6. Advanced Topics

### 6.1 Count-based vs Time-based windows

#### Count-based

Pros:

- workload-normalized
- traffic burst হলেও same event count

Cons:

- wall-clock meaning কম
- dashboard-এ explain কঠিন হতে পারে

Use cases:

- প্রতি 1000 click-এ summary
- প্রতি 50 order-এ anomaly check

#### Time-based

Pros:

- business-friendly
- reporting natural
- operational dashboards aligned

Cons:

- traffic sparse হলে window nearly empty হতে পারে
- traffic burst হলে same duration-এ lots of events

### 6.2 Watermarks and window closing triggers

**Watermark** হলো system-এর estimate যে event-time progress কোথায় পর্যন্ত fairly complete।

It answers:

“এই সময়ের আগের event আসার সম্ভাবনা খুব কম।” 

Watermark use cases:

- close tumbling window
- finalize sliding hops
- expire session after grace

#### Watermark diagram

```text
Event time axis:
10:00 ---- 10:05 ---- 10:10 ---- 10:15 ---- 10:20

Seen events up to ~10:14
Watermark = 10:12

Interpretation:
We believe most events <= 10:12 have arrived.
So windows ending before watermark may be closed.
```

### 6.3 Allowed lateness and grace periods

Window close হওয়ার পরও কিছু সময় late events allow করা হয়।

#### Example

- tumbling size = 5m
- watermark closes at 10:10
- grace = 2m
- purge at 10:12

Late event 10:09:30 যদি 10:11-এ আসে,

তাহলে still accepted হতে পারে।

### 6.4 Window result emission modes

#### Mode A: On close

- stable result
- low noise
- higher latency

#### Mode B: On each event

- freshest result
- noisy output
- higher downstream load

#### Mode C: Periodic emit

- compromise approach
- dashboards-এর জন্য common

#### Mode D: Threshold emit

- emit only if change > threshold
- alerts-এর জন্য useful

### 6.5 Retraction and update semantics

Late event বা session merge হলে old result ভুল হয়ে যেতে পারে।

Then system needs:

- retract old output
- emit corrected output

Common patterns:

- upsert by window key
- tombstone old session ID
- versioned output
- changelog stream

### 6.6 Update semantics example

```text
Initial emit:
window=10:00-10:05, key=A, count=4, version=1

Late event arrives

Correction emit:
window=10:00-10:05, key=A, count=5, version=2

Downstream rule:
latest version wins
```

### 6.7 Event-time vs processing-time pitfalls

If you use processing time for user analytics:

- mobile offline events wrong bucket-এ যেতে পারে
- network jitter misleading spikes create করতে পারে
- region latency কারণে zone comparison skewed হতে পারে

### 6.8 Backpressure implications

Windowed systems শুধু compute problem নয়,

state + IO + downstream pressure problem-ও।

Symptoms:

- Kafka lag বাড়ে
- Redis latency বাড়ে
- window close timers delayed হয়
- memory pressure বাড়ে

Mitigation:

- shard by key
- reduce emission frequency
- aggregate earlier
- batch writes
- compact state

### 6.9 Exactly-once vs at-least-once impact

If duplicate delivery possible:

- tumbling overcounts
- sliding overcounts in multiple overlapping results
- session gets polluted and may merge incorrectly

Need:

- event ID dedup
- transactional state update if possible
- idempotent sink writes

### 6.10 Reordering tolerance design

Questions:

- maximum out-of-order skew কত?
- grace period কত?
- corrected outputs support করবেন?
- late event drop acceptable?

### 6.11 Operational metadata to attach with outputs

Every emitted window result ideally carries:

- window type
- key
- window start
- window end
- emit time
- watermark used
- version
- late event count seen
- processing node ID

This helps audit and debugging.

---
## 7. PHP Code Examples (COMPREHENSIVE)

> এই section-এ Redis-backed, production-style conceptual implementation দেখানো হয়েছে।
>
> এখানে focus হলো class design, state handling, merge handling, output schema, এবং all-three strategy combine করে fraud pipeline বানানো।

### 7.1 Redis-backed architecture idea

```text
Kafka Topic -> PHP Consumer -> Window Processors -> Redis State
                                          │
                                          ├-> Alert Stream
                                          ├-> Analytics Topic
                                          └-> Dashboard Sink
```

### 7.2 Shared event schema

```php
<?php

$event = [
    'event_id' => 'txn-9001',
    'account_id' => 'ACCT-1001',
    'receiver_id' => 'ACCT-5555',
    'amount' => 2500,
    'event_time' => 1719223205,
    'device_id' => 'device-xyz',
    'type' => 'send_money',
    'zone' => 'dhaka',
];
```

### 7.3 Production-style PHP implementation

```php
<?php

declare(strict_types=1);

use Predis\Client;

interface WindowEmitter
{
    public function emit(array $payload): void;
}

final class ArrayEmitter implements WindowEmitter
{
    public array $events = [];

    public function emit(array $payload): void
    {
        $this->events[] = $payload;
    }
}

final class RedisJsonStore
{
    public function __construct(private Client $redis)
    {
    }

    public function put(string $key, array $value, int $ttlSeconds): void
    {
        $this->redis->setex($key, $ttlSeconds, json_encode($value, JSON_THROW_ON_ERROR));
    }

    public function get(string $key): ?array
    {
        $raw = $this->redis->get($key);
        return $raw ? json_decode($raw, true, 512, JSON_THROW_ON_ERROR) : null;
    }

    public function delete(string $key): void
    {
        $this->redis->del([$key]);
    }

    public function keys(string $pattern): array
    {
        return $this->redis->keys($pattern);
    }
}

final class TumblingWindowProcessorV2
{
    public function __construct(
        private RedisJsonStore $store,
        private WindowEmitter $emitter,
        private int $windowSizeSeconds,
        private int $allowedLatenessSeconds = 0,
    ) {
    }

    public function ingest(array $event): void
    {
        $eventTime = (int) $event['event_time'];
        $key = (string) $event['account_id'];
        $windowStart = $eventTime - ($eventTime % $this->windowSizeSeconds);
        $windowEnd = $windowStart + $this->windowSizeSeconds;

        $stateKey = $this->stateKey($key, $windowStart);
        $state = $this->store->get($stateKey) ?? [
            'window_type' => 'tumbling',
            'key' => $key,
            'window_start' => $windowStart,
            'window_end' => $windowEnd,
            'count' => 0,
            'sum' => 0.0,
            'unique_receivers' => [],
            'max_amount' => 0.0,
        ];

        $state['count']++;
        $state['sum'] += (float) $event['amount'];
        $state['max_amount'] = max($state['max_amount'], (float) $event['amount']);
        $state['unique_receivers'][(string) $event['receiver_id']] = true;

        $ttl = $this->windowSizeSeconds + $this->allowedLatenessSeconds + 600;
        $this->store->put($stateKey, $state, $ttl);
    }

    public function flush(int $watermark): void
    {
        foreach ($this->store->keys('tumbling:*') as $stateKey) {
            $state = $this->store->get($stateKey);
            if ($state === null) {
                continue;
            }

            if ($watermark >= $state['window_end'] + $this->allowedLatenessSeconds) {
                $payload = [
                    'type' => 'window_result',
                    'strategy' => 'tumbling',
                    'key' => $state['key'],
                    'window_start' => $state['window_start'],
                    'window_end' => $state['window_end'],
                    'count' => $state['count'],
                    'sum' => $state['sum'],
                    'unique_receiver_count' => count($state['unique_receivers']),
                    'max_amount' => $state['max_amount'],
                    'version' => 1,
                ];

                $this->emitter->emit($payload);
                $this->store->delete($stateKey);
            }
        }
    }

    private function stateKey(string $key, int $windowStart): string
    {
        return sprintf('tumbling:%s:%d', $key, $windowStart);
    }
}

final class SlidingWindowProcessorV2
{
    public function __construct(
        private Client $redis,
        private WindowEmitter $emitter,
        private int $windowSizeSeconds,
        private float $fraudAmountThreshold,
    ) {
    }

    public function ingest(array $event): void
    {
        $key = (string) $event['account_id'];
        $eventTime = (int) $event['event_time'];
        $memberId = (string) $event['event_id'];
        $payload = json_encode([
            'event_id' => $memberId,
            'amount' => (float) $event['amount'],
            'receiver_id' => (string) $event['receiver_id'],
        ], JSON_THROW_ON_ERROR);

        $streamKey = $this->streamKey($key);
        $this->redis->zadd($streamKey, [$payload => $eventTime]);
        $this->redis->zremrangebyscore($streamKey, '-inf', $eventTime - $this->windowSizeSeconds);
        $this->redis->expire($streamKey, ($this->windowSizeSeconds * 2));

        $rows = $this->redis->zrangebyscore($streamKey, $eventTime - $this->windowSizeSeconds, $eventTime);
        $parsed = array_map(
            fn (string $json) => json_decode($json, true, 512, JSON_THROW_ON_ERROR),
            $rows
        );

        $count = count($parsed);
        $sum = array_reduce($parsed, fn (float $acc, array $row) => $acc + (float) $row['amount'], 0.0);
        $uniqueReceivers = [];
        foreach ($parsed as $row) {
            $uniqueReceivers[$row['receiver_id']] = true;
        }

        $result = [
            'type' => 'window_result',
            'strategy' => 'sliding',
            'key' => $key,
            'window_start' => $eventTime - $this->windowSizeSeconds,
            'window_end' => $eventTime,
            'count' => $count,
            'sum' => $sum,
            'unique_receiver_count' => count($uniqueReceivers),
            'version' => 1,
        ];

        $this->emitter->emit($result);

        if ($count >= 5 && $sum >= $this->fraudAmountThreshold) {
            $this->emitter->emit([
                'type' => 'fraud_alert',
                'rule' => 'rolling_5m_high_velocity',
                'account_id' => $key,
                'window_start' => $eventTime - $this->windowSizeSeconds,
                'window_end' => $eventTime,
                'count' => $count,
                'sum' => $sum,
            ]);
        }
    }

    private function streamKey(string $key): string
    {
        return 'sliding:' . $key;
    }
}

final class SessionWindowProcessorV2
{
    public function __construct(
        private RedisJsonStore $store,
        private WindowEmitter $emitter,
        private int $gapSeconds,
        private int $maxSessionSeconds = 3600,
    ) {
    }

    public function ingest(array $event): void
    {
        $key = (string) $event['account_id'];
        $eventTime = (int) $event['event_time'];
        $amount = (float) $event['amount'];
        $pattern = sprintf('session:%s:*', $key);
        $sessionKeys = $this->store->keys($pattern);
        $matching = [];

        foreach ($sessionKeys as $sessionKey) {
            $session = $this->store->get($sessionKey);
            if ($session === null) {
                continue;
            }

            $withinGap = $eventTime >= ($session['start_time'] - $this->gapSeconds)
                && $eventTime <= ($session['end_time'] + $this->gapSeconds);

            if ($withinGap) {
                $matching[] = [$sessionKey, $session];
            }
        }

        if ($matching === []) {
            $session = [
                'session_id' => sprintf('%s:%d', $key, $eventTime),
                'key' => $key,
                'start_time' => $eventTime,
                'end_time' => $eventTime,
                'last_event_time' => $eventTime,
                'event_count' => 1,
                'sum_amount' => $amount,
                'event_ids' => [(string) $event['event_id']],
            ];

            $this->store->put($this->sessionKey($key, $session['session_id']), $session, $this->gapSeconds + $this->maxSessionSeconds);
            return;
        }

        $merged = [
            'session_id' => sprintf('%s:%d', $key, min(array_map(fn ($m) => $m[1]['start_time'], $matching))),
            'key' => $key,
            'start_time' => $eventTime,
            'end_time' => $eventTime,
            'last_event_time' => $eventTime,
            'event_count' => 1,
            'sum_amount' => $amount,
            'event_ids' => [(string) $event['event_id']],
        ];

        foreach ($matching as [$sessionKey, $session]) {
            $merged['start_time'] = min($merged['start_time'], $session['start_time']);
            $merged['end_time'] = max($merged['end_time'], $session['end_time']);
            $merged['last_event_time'] = max($merged['last_event_time'], $session['last_event_time']);
            $merged['event_count'] += $session['event_count'];
            $merged['sum_amount'] += $session['sum_amount'];
            $merged['event_ids'] = array_values(array_unique(array_merge($merged['event_ids'], $session['event_ids'])));

            $this->emitter->emit([
                'type' => 'retraction',
                'strategy' => 'session',
                'session_id' => $session['session_id'],
                'reason' => 'merged_by_late_event',
            ]);

            $this->store->delete($sessionKey);
        }

        $this->store->put(
            $this->sessionKey($key, $merged['session_id']),
            $merged,
            $this->gapSeconds + $this->maxSessionSeconds
        );
    }

    public function flush(int $watermark): void
    {
        foreach ($this->store->keys('session:*') as $sessionKey) {
            $session = $this->store->get($sessionKey);
            if ($session === null) {
                continue;
            }

            if (($watermark - $session['last_event_time']) > $this->gapSeconds) {
                $this->emitter->emit([
                    'type' => 'window_result',
                    'strategy' => 'session',
                    'session_id' => $session['session_id'],
                    'key' => $session['key'],
                    'start_time' => $session['start_time'],
                    'end_time' => $session['end_time'],
                    'duration_seconds' => $session['end_time'] - $session['start_time'],
                    'event_count' => $session['event_count'],
                    'sum_amount' => $session['sum_amount'],
                    'version' => 1,
                ]);

                if ($session['event_count'] >= 4 && $session['sum_amount'] >= 10000) {
                    $this->emitter->emit([
                        'type' => 'fraud_alert',
                        'rule' => 'session_burst_high_value',
                        'account_id' => $session['key'],
                        'session_id' => $session['session_id'],
                        'event_count' => $session['event_count'],
                        'sum_amount' => $session['sum_amount'],
                    ]);
                }

                $this->store->delete($sessionKey);
            }
        }
    }

    private function sessionKey(string $key, string $sessionId): string
    {
        return sprintf('session:%s:%s', $key, $sessionId);
    }
}

final class FraudDetectionPipeline
{
    public function __construct(
        private TumblingWindowProcessorV2 $tumbling,
        private SlidingWindowProcessorV2 $sliding,
        private SessionWindowProcessorV2 $session,
    ) {
    }

    public function onEvent(array $event): void
    {
        $this->tumbling->ingest($event);
        $this->sliding->ingest($event);
        $this->session->ingest($event);
    }

    public function onWatermark(int $watermark): void
    {
        $this->tumbling->flush($watermark);
        $this->session->flush($watermark);
    }
}

$redis = new Client(['host' => '127.0.0.1', 'port' => 6379]);
$store = new RedisJsonStore($redis);
$emitter = new ArrayEmitter();

$pipeline = new FraudDetectionPipeline(
    new TumblingWindowProcessorV2($store, $emitter, 300, 60),
    new SlidingWindowProcessorV2($redis, $emitter, 300, 8000),
    new SessionWindowProcessorV2($store, $emitter, 300, 3600),
);

$sampleEvents = [
    ['event_id' => 't1', 'account_id' => 'A-1001', 'receiver_id' => 'R-1', 'amount' => 2000, 'event_time' => 1719223200],
    ['event_id' => 't2', 'account_id' => 'A-1001', 'receiver_id' => 'R-2', 'amount' => 1500, 'event_time' => 1719223220],
    ['event_id' => 't3', 'account_id' => 'A-1001', 'receiver_id' => 'R-3', 'amount' => 1800, 'event_time' => 1719223240],
    ['event_id' => 't4', 'account_id' => 'A-1001', 'receiver_id' => 'R-4', 'amount' => 2200, 'event_time' => 1719223260],
    ['event_id' => 't5', 'account_id' => 'A-1001', 'receiver_id' => 'R-5', 'amount' => 3000, 'event_time' => 1719223280],
];

foreach ($sampleEvents as $event) {
    $pipeline->onEvent($event);
}

$pipeline->onWatermark(1719223700);
print_r($emitter->events);
```

### 7.4 এই PHP pipeline কী করছে

- tumbling দিয়ে fixed report তৈরি করছে
- sliding দিয়ে rolling fraud rule evaluate করছে
- session দিয়ে burst behavior detect করছে
- Redis state use করছে
- retraction concept দেখাচ্ছে
- watermark-based flush support করছে

### 7.5 Production notes for PHP implementation

- `KEYS` production-এ expensive; use scan / index sets
- per-event `ZRANGEBYSCORE` heavy হতে পারে; incremental buckets consider করুন
- exactly-once চাইলে transactional pattern দরকার
- Redis TTL cleanup helpful but logical close আলাদা concern
- emitter ideally Kafka producer / outbox pattern use করবে

---
## 8. Node.js Code Examples (COMPREHENSIVE)

> এই section-এ EventEmitter, Redis, KafkaJS ব্যবহার করে production-oriented JavaScript design দেখানো হয়েছে।

### 8.1 Architecture overview

```text
KafkaJS Consumer -> Parser -> Window Router -> Redis-backed Processors
                                           │
                                           ├-> EventEmitter 'window.result'
                                           ├-> EventEmitter 'fraud.alert'
                                           └-> Kafka producer / WebSocket push
```

### 8.2 Comprehensive Node.js implementation

```javascript
import EventEmitter from 'node:events';
import Redis from 'ioredis';
import { Kafka } from 'kafkajs';

class Bus extends EventEmitter {}

class RedisJsonStore {
  constructor(redis) {
    this.redis = redis;
  }

  async put(key, value, ttlSeconds) {
    await this.redis.set(key, JSON.stringify(value), 'EX', ttlSeconds);
  }

  async get(key) {
    const raw = await this.redis.get(key);
    return raw ? JSON.parse(raw) : null;
  }

  async del(key) {
    await this.redis.del(key);
  }

  async scan(pattern) {
    let cursor = '0';
    const keys = [];

    do {
      const [nextCursor, batch] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = nextCursor;
      keys.push(...batch);
    } while (cursor !== '0');

    return keys;
  }
}

class TumblingWindowProcessor {
  constructor({ store, bus, windowSizeMs, allowedLatenessMs = 0 }) {
    this.store = store;
    this.bus = bus;
    this.windowSizeMs = windowSizeMs;
    this.allowedLatenessMs = allowedLatenessMs;
  }

  stateKey(key, windowStart) {
    return `tumbling:${key}:${windowStart}`;
  }

  async ingest(event) {
    const key = event.accountId;
    const eventTime = event.eventTime;
    const windowStart = eventTime - (eventTime % this.windowSizeMs);
    const windowEnd = windowStart + this.windowSizeMs;
    const stateKey = this.stateKey(key, windowStart);

    const state = (await this.store.get(stateKey)) || {
      strategy: 'tumbling',
      key,
      windowStart,
      windowEnd,
      count: 0,
      sum: 0,
      maxAmount: 0,
      uniqueReceivers: {},
    };

    state.count += 1;
    state.sum += Number(event.amount || 0);
    state.maxAmount = Math.max(state.maxAmount, Number(event.amount || 0));
    state.uniqueReceivers[event.receiverId] = true;

    await this.store.put(
      stateKey,
      state,
      Math.ceil((this.windowSizeMs + this.allowedLatenessMs + 600_000) / 1000)
    );
  }

  async flush(watermark) {
    const keys = await this.store.scan('tumbling:*');

    for (const stateKey of keys) {
      const state = await this.store.get(stateKey);
      if (!state) continue;

      if (watermark >= state.windowEnd + this.allowedLatenessMs) {
        this.bus.emit('window.result', {
          type: 'window_result',
          strategy: 'tumbling',
          key: state.key,
          windowStart: state.windowStart,
          windowEnd: state.windowEnd,
          count: state.count,
          sum: state.sum,
          maxAmount: state.maxAmount,
          uniqueReceiverCount: Object.keys(state.uniqueReceivers).length,
          version: 1,
        });

        await this.store.del(stateKey);
      }
    }
  }
}

class SlidingWindowProcessor {
  constructor({ redis, bus, windowSizeMs, fraudCountThreshold, fraudAmountThreshold }) {
    this.redis = redis;
    this.bus = bus;
    this.windowSizeMs = windowSizeMs;
    this.fraudCountThreshold = fraudCountThreshold;
    this.fraudAmountThreshold = fraudAmountThreshold;
  }

  streamKey(key) {
    return `sliding:${key}`;
  }

  async ingest(event) {
    const key = event.accountId;
    const eventTime = event.eventTime;
    const redisKey = this.streamKey(key);
    const member = JSON.stringify({
      eventId: event.eventId,
      receiverId: event.receiverId,
      amount: Number(event.amount || 0),
    });

    await this.redis.zadd(redisKey, eventTime, member);
    await this.redis.zremrangebyscore(redisKey, '-inf', eventTime - this.windowSizeMs);
    await this.redis.pexpire(redisKey, this.windowSizeMs * 2);

    const members = await this.redis.zrangebyscore(
      redisKey,
      eventTime - this.windowSizeMs,
      eventTime
    );

    const rows = members.map((m) => JSON.parse(m));
    const count = rows.length;
    const sum = rows.reduce((acc, row) => acc + Number(row.amount || 0), 0);
    const uniqueReceiverCount = new Set(rows.map((row) => row.receiverId)).size;

    this.bus.emit('window.result', {
      type: 'window_result',
      strategy: 'sliding',
      key,
      windowStart: eventTime - this.windowSizeMs,
      windowEnd: eventTime,
      count,
      sum,
      uniqueReceiverCount,
      version: 1,
    });

    if (count >= this.fraudCountThreshold && sum >= this.fraudAmountThreshold) {
      this.bus.emit('fraud.alert', {
        type: 'fraud_alert',
        rule: 'rolling_5m_velocity',
        accountId: key,
        windowStart: eventTime - this.windowSizeMs,
        windowEnd: eventTime,
        count,
        sum,
      });
    }
  }
}

class SessionWindowProcessor {
  constructor({ store, bus, gapMs, maxSessionMs = 60 * 60 * 1000 }) {
    this.store = store;
    this.bus = bus;
    this.gapMs = gapMs;
    this.maxSessionMs = maxSessionMs;
  }

  sessionKey(key, sessionId) {
    return `session:${key}:${sessionId}`;
  }

  async ingest(event) {
    const key = event.accountId;
    const eventTime = event.eventTime;
    const pattern = `session:${key}:*`;
    const sessionKeys = await this.store.scan(pattern);
    const matching = [];

    for (const sessionKey of sessionKeys) {
      const session = await this.store.get(sessionKey);
      if (!session) continue;

      const withinGap =
        eventTime >= session.startTime - this.gapMs &&
        eventTime <= session.endTime + this.gapMs;

      if (withinGap) {
        matching.push({ sessionKey, session });
      }
    }

    if (matching.length === 0) {
      const newSession = {
        sessionId: `${key}:${eventTime}`,
        key,
        startTime: eventTime,
        endTime: eventTime,
        lastEventTime: eventTime,
        eventCount: 1,
        sumAmount: Number(event.amount || 0),
        eventIds: [event.eventId],
      };

      await this.store.put(
        this.sessionKey(key, newSession.sessionId),
        newSession,
        Math.ceil((this.gapMs + this.maxSessionMs) / 1000)
      );
      return;
    }

    const merged = {
      sessionId: `${key}:${Math.min(...matching.map((item) => item.session.startTime), eventTime)}`,
      key,
      startTime: eventTime,
      endTime: eventTime,
      lastEventTime: eventTime,
      eventCount: 1,
      sumAmount: Number(event.amount || 0),
      eventIds: [event.eventId],
    };

    for (const item of matching) {
      const { sessionKey, session } = item;
      merged.startTime = Math.min(merged.startTime, session.startTime);
      merged.endTime = Math.max(merged.endTime, session.endTime);
      merged.lastEventTime = Math.max(merged.lastEventTime, session.lastEventTime);
      merged.eventCount += session.eventCount;
      merged.sumAmount += Number(session.sumAmount || 0);
      merged.eventIds = [...new Set([...merged.eventIds, ...session.eventIds])];

      this.bus.emit('window.retraction', {
        type: 'retraction',
        strategy: 'session',
        sessionId: session.sessionId,
        reason: 'merged_by_late_event',
      });

      await this.store.del(sessionKey);
    }

    await this.store.put(
      this.sessionKey(key, merged.sessionId),
      merged,
      Math.ceil((this.gapMs + this.maxSessionMs) / 1000)
    );
  }

  async flush(watermark) {
    const sessionKeys = await this.store.scan('session:*');

    for (const sessionKey of sessionKeys) {
      const session = await this.store.get(sessionKey);
      if (!session) continue;

      if (watermark - session.lastEventTime > this.gapMs) {
        this.bus.emit('window.result', {
          type: 'window_result',
          strategy: 'session',
          sessionId: session.sessionId,
          key: session.key,
          startTime: session.startTime,
          endTime: session.endTime,
          durationMs: session.endTime - session.startTime,
          eventCount: session.eventCount,
          sumAmount: session.sumAmount,
          version: 1,
        });

        if (session.eventCount >= 4 && session.sumAmount >= 10_000) {
          this.bus.emit('fraud.alert', {
            type: 'fraud_alert',
            rule: 'session_burst_high_value',
            accountId: session.key,
            sessionId: session.sessionId,
            eventCount: session.eventCount,
            sumAmount: session.sumAmount,
          });
        }

        await this.store.del(sessionKey);
      }
    }
  }
}

class RealTimeAnalyticsPipeline {
  constructor({ tumbling, sliding, session, bus }) {
    this.tumbling = tumbling;
    this.sliding = sliding;
    this.session = session;
    this.bus = bus;
  }

  async onEvent(event) {
    await this.tumbling.ingest(event);
    await this.sliding.ingest(event);
    await this.session.ingest(event);
  }

  async onWatermark(watermark) {
    await this.tumbling.flush(watermark);
    await this.session.flush(watermark);
  }
}

const redis = new Redis(process.env.REDIS_URL || 'redis://127.0.0.1:6379');
const store = new RedisJsonStore(redis);
const bus = new Bus();

bus.on('window.result', (payload) => {
  console.log('[WINDOW]', payload);
});

bus.on('fraud.alert', (payload) => {
  console.log('[ALERT]', payload);
});

bus.on('window.retraction', (payload) => {
  console.log('[RETRACT]', payload);
});

const pipeline = new RealTimeAnalyticsPipeline({
  tumbling: new TumblingWindowProcessor({
    store,
    bus,
    windowSizeMs: 5 * 60 * 1000,
    allowedLatenessMs: 60 * 1000,
  }),
  sliding: new SlidingWindowProcessor({
    redis,
    bus,
    windowSizeMs: 5 * 60 * 1000,
    fraudCountThreshold: 5,
    fraudAmountThreshold: 8_000,
  }),
  session: new SessionWindowProcessor({
    store,
    bus,
    gapMs: 5 * 60 * 1000,
    maxSessionMs: 60 * 60 * 1000,
  }),
  bus,
});

async function runKafkaConsumer() {
  const kafka = new Kafka({
    clientId: 'bd-stream-analytics',
    brokers: ['localhost:9092'],
  });

  const consumer = kafka.consumer({ groupId: 'fraud-window-processors' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'bkash.transactions', fromBeginning: false });

  setInterval(async () => {
    await pipeline.onWatermark(Date.now() - 5_000);
  }, 5_000);

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      await pipeline.onEvent({
        eventId: event.event_id,
        accountId: event.account_id,
        receiverId: event.receiver_id,
        amount: event.amount,
        eventTime: event.event_time,
      });
    },
  });
}

// runKafkaConsumer().catch(console.error);
```

### 8.3 Node.js implementation notes

- EventEmitter internal decoupling দেয়
- KafkaJS consumer window processor-এর upstream হিসেবে কাজ করে
- Redis cross-process state sharing enable করে
- session retraction downstream stateful sinks-এর জন্য important
- periodic watermark loop close semantics চালায়

### 8.4 Real-time analytics pipeline example

Possible outputs:

- dashboard update events
- fraud alert topic
- WebSocket push to ops console
- hourly archival table
- session summary topic for offline ML features

---

## 9. Performance Considerations

### 9.1 Memory usage per strategy

#### Tumbling memory pattern

Approximate state:

- active keys × aggregates per active window

Usually predictable because active windows few।

Formula intuition:

`Memory ≈ activeKeys × statePerKeyPerWindow × activeWindowCount`

#### Sliding memory pattern

Naive sliding memory:

- recent events × active keys × overlap factor

Optimized sliding memory:

- recent events per key
- or compacted buckets per key

Still generally tumbling-এর চেয়ে বেশি।

#### Session memory pattern

- active sessions per key
- session metadata
- merge buffers / grace state

Worst case unpredictable,

because noisy key extremely long session hold করতে পারে।

### 9.2 CPU cost per strategy

#### Tumbling

- O(1) aggregate update common
- cheapest overall

#### Sliding

- O(log n) insert + range query in ZSET style
- or O(1) amortized with specialized structures
- generally more CPU intensive

#### Session

- match against active sessions
- merge operations
- sort / search overhead
- late event increases complexity

### 9.3 Network IO per strategy

If Redis-backed:

#### Tumbling

- one read/write per event or batched write
- moderate IO

#### Sliding

- insert + trim + range query per event
- high IO if naive

#### Session

- scan / fetch / put / delete on merge
- bursty IO pattern

### 9.4 Storage / checkpoint considerations

Need persist:

- active tumbling states
- recent sliding structures
- open sessions
- watermark position
- dedup state

### 9.5 When infrastructure is constrained

#### Low memory, low CPU, simple dashboard

Choose tumbling.

#### Medium resources, recent trend critical

Choose sliding or hopping.

#### User behavior understanding most important

Choose session, but put caps and TTLs.

### 9.6 Operational decision table

| Constraint | Best choice | Why |
|-----------|-------------|-----|
| Cheapest implementation | Tumbling | Simple and predictable |
| Lowest latency rolling alert | Sliding | Natural for recent-N |
| Best user journey model | Session | Gap-based behavior |
| Strict auditable billing | Tumbling | Fixed aligned buckets |
| Dynamic trend ranking | Sliding | Recency preserved |
| Irregular burst analysis | Session | Natural activity chunk |

### 9.7 Scaling advice

#### Partitioning

- partition by business key
- avoid hot key concentration
- consider composite key (zone + merchant)

#### State placement

- local in-memory for low latency
- Redis for shared distributed state
- RocksDB/local state + changelog for high throughput frameworks

#### Backpressure control

- batch downstream emits
- sample non-critical metrics
- reduce sliding emission frequency
- add max session duration

### 9.8 Practical heuristic

If unsure:

- start with tumbling for reporting
- add sliding for fraud / anomaly rules
- add session for behavior analytics

Do not force one strategy to solve every question.

### 9.9 Cost intuition using Bangladesh examples

#### bKash

- compliance reports -> tumbling
- fraud alerts -> sliding + session

#### Prothom Alo

- real-time trending -> sliding
- hourly editorial report -> tumbling
- reading journey analytics -> session

#### Pathao

- surge pricing input -> tumbling or hopping
- last 5-minute demand spike -> sliding
- driver active block -> session

#### Daraz

- warehouse throughput -> tumbling
- user recency intent -> sliding
- user shopping journey -> session

---

## 10. Key Takeaways

### 10.1 Core summary

- Infinite stream-এর end নেই
- তাই full-collection based processing stream world-এ কাজ করে না
- finite output চাইলে bounded computation দরকার
- windowing সেই bounded computation-এর contract

### 10.2 Tumbling summary

- fixed-size
- non-overlapping
- one event -> one window
- simple
- cheap
- predictable
- but boundary effects আছে

### 10.3 Sliding summary

- rolling recent view
- overlapping possible
- one event -> many logical windows/output updates
- more accurate for last N time
- more expensive
- more memory / IO heavy

### 10.4 Session summary

- gap-based
- variable-size
- best for user behavior and bursts
- end time unknown until inactivity happens
- late event merge can complicate everything

### 10.5 Selection cheat sheet

Use **Tumbling** when:

- fixed report bucket দরকার
- billing দরকার
- audit দরকার
- simplicity priority

Use **Sliding** when:

- last N minutes প্রশ্ন
- trend / recency / anomaly detection
- boundary effect unacceptable

Use **Session** when:

- user journey
- engagement burst
- inactivity gap meaningful
- funnel analysis

### 10.6 Design principles

- define business question first
- then choose window semantics
- never choose by framework buzzword only
- specify event time policy explicitly
- decide late event policy explicitly
- define output update semantics explicitly
- attach version / watermark metadata
- control memory with TTL / caps / eviction

### 10.7 Final mental model

```text
Tumbling -> calendar-like buckets
Sliding  -> moving camera over recent time
Session  -> natural clusters of activity separated by silence
```

### 10.8 Final takeaway bullets

- Infinite stream processing-এর heart হলো bounded state management
- Window strategy ভুল হলে metric technically valid but business-wrong হতে পারে
- Tumbling, Sliding, Session — তিনটি একই problem-এর তিনটি different semantic answer
- “Which one is best?” ভুল প্রশ্ন
- Right question হলো:
- “আমার business truth কোন semantics follow করে?”
- bKash fraud, Daraz recency, Prothom Alo trending, Pathao activity — সব use case একই window দিয়ে solve করা উচিত নয়
- Stream processing design is fundamentally about semantics, state, cost, and correctness under time

---

## Appendix A — অতিরিক্ত তুলনামূলক নোট

### A.1 একই use case-এ multiple answer

Question:

“Pathao Banani zone-এ demand কেমন?”

Possible answers:

- প্রতি ১ মিনিটে requests = tumbling
- এখন last ৫ মিনিট rolling requests = sliding
- rider bursts during active commute block = session

সব answer useful,

but for different decisions.

### A.2 Anti-patterns

- fraud burst rule tumbling-only দিয়ে করা
- billing sliding দিয়ে করা যখন fixed bucket দরকার
- session analytics-এ strict fixed bucket force করা
- late event policy unspecified রাখা
- output schema-তে version না রাখা
- window type downstream-এ না পাঠানো

### A.3 Production checklist

- [ ] key cardinality estimate করেছেন?
- [ ] hot key strategy আছে?
- [ ] event time trustworthy?
- [ ] clock skew tolerance defined?
- [ ] watermark strategy defined?
- [ ] late event drop / update policy documented?
- [ ] dedup event ID available?
- [ ] memory ceiling known?
- [ ] max session duration set?
- [ ] output sink upsert-aware?

### A.4 Common interview mistakes

- tumbling vs sliding difference শুধু overlap বলে থেমে যাওয়া
- session merge problem mention না করা
- event time vs processing time ignore করা
- memory trade-off explain না করা
- lateness handling skip করা

### A.5 One-line memory formulas

- Tumbling: `O(keys × activeWindows)`
- Sliding naive: `O(keys × recentEvents × overlapFactor)`
- Sliding optimized: `O(keys × recentEvents)`
- Session: `O(activeSessions + mergeBuffer)`

### A.6 One-line CPU formulas

- Tumbling: near `O(1)` per event
- Sliding ZSET: `O(log n + range)`
- Session: `O(session lookup + possible merge)`

### A.7 One-line correctness formulas

- Tumbling correctness risk = boundary effect
- Sliding correctness risk = duplicate / emission interpretation
- Session correctness risk = merge / late event revision

---

## Appendix B — Bangladesh-centered scenario gallery

### B.1 bKash

- per 5m transaction report -> tumbling
- any 5m fraud burst -> sliding
- suspicious send-money burst cluster -> session

### B.2 Daraz

- per 15m order count -> tumbling
- viewed in last 30m -> sliding
- shopping journey -> session

### B.3 Prothom Alo

- hourly editorial report -> tumbling
- trending in last 1h -> sliding
- reading session -> session

### B.4 Pathao

- per minute zone demand -> tumbling
- live rolling demand -> sliding
- driver active period -> session

### B.5 GP

- hourly SMS billing -> tumbling
- rolling abuse detection -> sliding
- call usage burst cluster -> session

### B.6 Chaldal

- warehouse throughput report -> tumbling
- rolling demand pulse -> sliding
- shopper active basket session -> session

### B.7 Foodpanda

- per 10m order dashboard -> tumbling
- last 15m kitchen load -> sliding
- courier work burst -> session

---

## Appendix C — Ultra-short strategy memory aid

```text
If the business says “every”      -> think Tumbling
If the business says “last”       -> think Sliding
If the business says “until idle” -> think Session
```
