# 🌊 ব্যাকপ্রেশার (Backpressure) — Producer-Consumer ভারসাম্যের কৌশল

> **"যখন producer আপনাকে যতটা পারে দিচ্ছে, কিন্তু আপনার পেট তত নেই — তখন আপনার প্রয়োজন backpressure।"**

ডিস্ট্রিবিউটেড সিস্টেমে অন্যতম নীরব ঘাতক হলো **unbounded queue**। মেমোরি ভরে যায়, GC pause বাড়ে, latency
spike করে, এবং একসময় OOM kill হয়। সমাধান হলো — consumer-কে producer-কে বলতে হবে: "একটু থামো।" এই
"একটু থামো" বার্তাটাই **backpressure**।

এই ডকুমেন্টে আমরা backpressure-এর জল-পাইপ analogy থেকে শুরু করে Node.js streams, RxJS reactive,
HTTP/2 flow control, Kafka consumer pause/resume, WebSocket bufferedAmount, এবং Pathao Foods/bKash-এর
real-world সিনারিও পর্যন্ত গভীরে যাব।

---

## 📑 সূচিপত্র

1. [Definition: জল-পাইপ analogy](#-definition-জল-পাইপ-analogy)
2. [Backpressure ছাড়া কী হয়?](#-backpressure-ছাড়া-কী-হয়)
3. [Backpressure Strategies](#-backpressure-strategies)
4. [Node.js Streams](#-nodejs-streams)
5. [Reactive Streams (Project Reactor / RxJava)](#-reactive-streams-project-reactor--rxjava)
6. [RxJS — UI side backpressure](#-rxjs--ui-side-backpressure)
7. [gRPC ও HTTP/2/3 flow control](#-grpc--http23-flow-control)
8. [TCP — যেখান থেকে সব শুরু](#-tcp--যেখান-থেকে-সব-শুরু)
9. [Kafka consumer backpressure](#-kafka-consumer-backpressure)
10. [Message Queue (RabbitMQ, SQS) prefetch](#-message-queue-rabbitmq-sqs-prefetch)
11. [WebSockets ও SSE](#-websockets--sse)
12. [Database connection pool: implicit backpressure](#-database-connection-pool-implicit-backpressure)
13. [PHP-এ backpressure (request-driven model)](#-php-এ-backpressure-request-driven-model)
14. [Backpressure vs Throttling vs Rate Limiting](#-backpressure-vs-throttling-vs-rate-limiting)
15. [BD Scenario: Pathao Foods + bKash SMS](#-bd-scenario-pathao-foods--bkash-sms)
16. [Anti-patterns](#️-anti-patterns)
17. [Checklist](#-checklist)

---

## 📌 Definition: জল-পাইপ analogy

কল্পনা করুন:

```
   ┌─────────┐    1000 L/s     ┌──────────┐    100 L/s    ┌──────────┐
   │ ট্যাঙ্ক  │ ─────────────── │   পাইপ   │ ───────────── │  বালতি   │
   │(Producer)│                 │  (Buffer)│                │(Consumer)│
   └─────────┘                  └──────────┘                └──────────┘
                                     ↑
                          এখানে জল জমতে থাকে
                       → Buffer overflow → ছিটকে পড়ে
```

Producer যদি 1000 L/s ছাড়ে, কিন্তু consumer 100 L/s নিতে পারে — পার্থক্যটা কোথাও জমা হবে। যদি buffer
infinite হয় → memory blowup। যদি bounded হয় → কিছু একটা পদক্ষেপ নিতে হবে: **drop, block, throttle, বা
backpressure signal।**

> **Backpressure** = consumer থেকে producer-এর দিকে পাঠানো এমন signal যা বলে — "তোমার rate কমাও,
> আমি cope করতে পারছি না।"

---

## 💥 Backpressure ছাড়া কী হয়?

```
   T+0s:    Producer 10K msg/s, Consumer 1K msg/s
   T+1s:    Queue size = 9K, RAM 200MB
   T+10s:   Queue size = 90K, RAM 2GB
   T+30s:   Queue size = 270K, RAM 6GB → OS swap → latency 50× spike
   T+45s:   GC pause 8s → consumer effectively 0 → queue 360K
   T+50s:   💀 OOM kill → process restart → in-flight সব হারিয়ে গেল
```

**লক্ষণ:**
- 📈 Memory ক্রমাগত বাড়ছে (memory leak মনে হবে — কিন্তু আসলে queue blowup)
- 📉 P99 latency বাড়ছে
- 🐢 GC pause বাড়ছে (heap-এ object বেশি)
- ⏰ End-to-end latency: producer যা পাঠাল ১ মিনিট পরে process হচ্ছে
- 💥 অবশেষে — OOM, container restart

---

## 🎛️ Backpressure Strategies

```
┌────────────────────────┬────────────────────────────────────────────────┐
│ Strategy               │ ব্যাখ্যা                                         │
├────────────────────────┼────────────────────────────────────────────────┤
│ Pause/Resume           │ Buffer full → producer-কে stop signal           │
│ Drop oldest            │ Queue full → head element drop, নতুন push       │
│ Drop newest            │ Queue full → incoming reject                    │
│ Sample (every N-th)    │ N message-এ ১টি keep, বাকিদের drop              │
│ Buffer + Reject        │ Bounded queue, full হলে error throw             │
│ Buffer + Block         │ Queue full হলে producer thread block            │
│ Batch                  │ N message জমিয়ে একসাথে process                  │
│ Throttle (rate)        │ X requests/sec এর বেশি ignore                   │
│ Debounce               │ Burst শেষ হওয়ার পরে শুধু latest                  │
└────────────────────────┴────────────────────────────────────────────────┘
```

### কখন কোনটা?

```
                    ┌─────────────────────────────────┐
                    │ প্রতিটি event কি critical?       │
                    └────────────┬────────────────────┘
                       Yes ─────┘└───── No
                       │                │
                       ▼                ▼
              Pause/Resume        ┌─────────────┐
              বা bounded queue    │ Stale OK?   │
              + reject + retry    └──────┬──────┘
              (banking, order)    Yes ──┘└─── No
                                  │            │
                                  ▼            ▼
                              Drop oldest   Drop newest
                              (sensor avg)  (mouse-move)
                              Sample
                              (analytics)
```

---

## 🟢 Node.js Streams

Node.js streams-এ backpressure built-in। মূল মেকানিজম:

```
   write() returns false  →  "buffer full, একটু থামো"
   'drain' event fires    →  "এখন আবার দিতে পারো"
```

### highWaterMark

প্রতিটি stream-এর একটা buffer threshold আছে। ডিফল্ট:
- Readable stream: 16 KB
- Writable stream (object mode): 16 objects

`buffered > highWaterMark` হলে `write()` `false` ফেরায়।

### Manual backpressure handling (raw streams)

```javascript
// ❌ ভুল — backpressure ignore
readable.on('data', (chunk) => {
  writable.write(chunk);  // false return করলেও আমরা avoid করছি
});
// → memory blowup যদি writable slow হয়

// ✅ সঠিক — backpressure aware
readable.on('data', (chunk) => {
  const ok = writable.write(chunk);
  if (!ok) {
    readable.pause();
    writable.once('drain', () => readable.resume());
  }
});
```

### `pipe()` ও `pipeline()` — backpressure automatic

```javascript
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');
const { createGzip } = require('node:zlib');

await pipeline(
  fs.createReadStream('huge-input.csv'),
  createGzip(),
  fs.createWriteStream('output.csv.gz')
);
// ✅ backpressure automatic; error propagation; cleanup
```

> **`pipeline()` ব্যবহার করুন `pipe()` এর বদলে** — এটি error handling ও cleanup সঠিকভাবে করে।

### Real-world: Daraz product image → S3 upload

```javascript
const { pipeline } = require('node:stream/promises');
const { S3Client, Upload } = require('@aws-sdk/lib-storage');
const fs = require('node:fs');
const { Transform } = require('node:stream');

async function uploadProductImage(localPath, s3Key) {
  const watermark = new Transform({
    transform(chunk, _enc, cb) {
      // simulated watermarking; backpressure-aware
      cb(null, chunk);
    },
    highWaterMark: 64 * 1024,   // 64KB chunks
  });

  const upload = new Upload({
    client: new S3Client({ region: 'ap-southeast-1' }),
    params: {
      Bucket: 'daraz-product-images',
      Key: s3Key,
      Body: fs.createReadStream(localPath).pipe(watermark),
    },
    queueSize: 4,                // S3 multipart concurrency = 4
    partSize: 5 * 1024 * 1024,   // 5MB parts
  });

  upload.on('httpUploadProgress', (p) =>
    console.log(`Uploaded ${((p.loaded / p.total) * 100).toFixed(1)}%`)
  );

  return upload.done();
}
```

### CSV parsing with backpressure

```javascript
const fs = require('node:fs');
const { parse } = require('csv-parse');
const { Transform, pipeline } = require('node:stream');
const { promisify } = require('node:util');

const pipelineAsync = promisify(pipeline);

// Daraz: ১০ লাখ products import
async function importProducts(csvPath, dbInsertBatch) {
  const batch = [];
  const batchSize = 500;

  const batcher = new Transform({
    objectMode: true,
    highWaterMark: 100,    // ১০০ row পর্যন্ত buffer
    async transform(row, _enc, cb) {
      batch.push(row);
      if (batch.length >= batchSize) {
        try {
          await dbInsertBatch(batch.splice(0, batchSize));
          cb();              // ← এখান থেকে cb call → upstream resume
        } catch (e) { cb(e); }
      } else {
        cb();
      }
    },
    async flush(cb) {
      if (batch.length) {
        try { await dbInsertBatch(batch); cb(); }
        catch (e) { cb(e); }
      } else cb();
    },
  });

  await pipelineAsync(
    fs.createReadStream(csvPath),
    parse({ columns: true, skip_empty_lines: true }),
    batcher
  );
}
// DB slow হলে batcher এর internal buffer ভরে → CSV parser pause →
// fs read pause → kernel page cache সীমিত। RAM controlled।
```

---

## ⚛️ Reactive Streams (Project Reactor / RxJava)

JVM ecosystem-এ Reactive Streams **specification**: `Publisher`, `Subscriber`, `Subscription`. Subscriber
explicitly বলে "আমাকে আর N টি item দাও" — এটি **pull-based** backpressure।

```
   Publisher                                  Subscriber
       │                                          │
       │  ◄──── subscribe() ────────────────────  │
       │                                          │
       │  ────  onSubscribe(Subscription) ─────►  │
       │                                          │
       │  ◄──── subscription.request(10) ────────│  "১০টা চাই"
       │                                          │
       │  ────  onNext(item1) ─────────────────►  │
       │  ────  onNext(item2) ─────────────────►  │
       │                ...                       │
       │  ────  onNext(item10) ────────────────►  │
       │                                          │
       │  ◄──── subscription.request(10) ─────── │  "আরো ১০টা"
```

### Backpressure overflow strategies

```java
Flux<Order> orders = orderStream();

orders.onBackpressureBuffer(1000)    // bounded buffer
      .onBackpressureDrop(o -> log.warn("Dropped {}", o))
      .onBackpressureLatest()        // শুধু latest রাখো
      .onBackpressureError()         // overflow → IllegalStateException
      .subscribe(consumer);
```

| Strategy | Behavior |
|---|---|
| `BUFFER` | Unbounded buffer (⚠️ memory risk) বা bounded |
| `DROP` | Overflow হলে নতুন element drop |
| `LATEST` | শুধু সবচেয়ে recent রাখে (UI updates এ আদর্শ) |
| `ERROR` | OverflowException → fail fast |

---

## 🎨 RxJS — UI side backpressure

Browser-এ search-as-you-type, mouse-move, scroll — সবই high-frequency event। backpressure operator-গুলো:

```
┌────────────────┬───────────────────────────────────────────────────┐
│ Operator       │ Behavior                                           │
├────────────────┼───────────────────────────────────────────────────┤
│ throttleTime   │ প্রতি window-এ first emission keep                  │
│ debounceTime   │ Burst শেষ হওয়ার পরে last emission                  │
│ sampleTime     │ প্রতি interval-এ latest sample                     │
│ auditTime      │ Burst শেষে last emission (debounce + throttle mix) │
│ buffer/bufferTime│ N সময় window-এ batch                            │
│ exhaustMap     │ Active inner Observable চলাকালে নতুন ignore        │
│ switchMap      │ নতুন emission এলে previous cancel                  │
└────────────────┴───────────────────────────────────────────────────┘
```

### Daraz search-as-you-type

```javascript
import { fromEvent } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap, map, filter } from 'rxjs/operators';

const $input = document.querySelector('#search');

fromEvent($input, 'input').pipe(
  map(e => e.target.value.trim()),
  filter(q => q.length >= 2),
  debounceTime(300),                // ৩০০ms typing pause না হলে fire হবে না
  distinctUntilChanged(),           // একই query দু'বার না
  switchMap(q => fetch(`/api/search?q=${encodeURIComponent(q)}`)
    .then(r => r.json()))           // আগের in-flight request cancel
).subscribe(renderResults);
```

**ব্যাখ্যা:**
- `debounceTime` → user typing শেষ না করা পর্যন্ত API call না (request reduction ~10×)
- `switchMap` → "biriy" টাইপ করার মাঝখানে "biri" এর response এলে discard

### Mouse move analytics — sample

```javascript
fromEvent(document, 'mousemove').pipe(
  sampleTime(100)                    // ১০০ms-এ ১টা sample → 600 events/min
).subscribe(e => trackHeatmap(e.clientX, e.clientY));
```

---

## 🚀 gRPC ও HTTP/2/3 flow control

### HTTP/2 stream flow control

HTTP/2-এ প্রতিটি stream-এর জন্য আলাদা flow control window:

```
   Sender                                Receiver
     │                                       │
     │  WINDOW_UPDATE (size=65535)           │
     │ ◄──────────────────────────────────  │
     │                                       │
     │  DATA frame (size=10000)              │
     │ ──────────────────────────────────►  │  receiver buffer = 65535-10000=55535
     │                                       │
     │  DATA frame (size=55535)              │
     │ ──────────────────────────────────►  │  buffer = 0 → STOP
     │                                       │
     │  WINDOW_UPDATE (delta=65535)          │  app consumed buffer
     │ ◄──────────────────────────────────  │
```

`SETTINGS_INITIAL_WINDOW_SIZE` (default 65 KB) tunable। Bandwidth-Delay Product বড় হলে window বাড়াতে হয়।

### gRPC server-streaming + client backpressure

```go
// Go server-streaming RPC
func (s *server) Orders(req *pb.OrdersReq, stream pb.OrderSvc_OrdersServer) error {
    for _, order := range fetchOrders() {
        if err := stream.Send(order); err != nil {
            return err   // ← receiver slow হলে এখানে block / error
        }
    }
    return nil
}
```

`stream.Send()` HTTP/2 flow control respect করে — receiver slow হলে block হয়, যা producer-এর কাছে
implicit backpressure।

### HTTP/3 (QUIC)

QUIC-এ stream-level + connection-level আলাদা flow control। Head-of-line blocking নেই — তাই packet
loss হলেও অন্য stream affected হয় না।

---

## 🔗 TCP — যেখান থেকে সব শুরু

পুরো OS-level network stack-এই backpressure built-in:

```
                    TCP Flow Control
   ┌──────────────────────────────────────────────────┐
   │                                                  │
   │   Sender                          Receiver       │
   │     │                                 │          │
   │     │  ── Data ──────────►            │          │
   │     │                                  │         │
   │     │            ◄── ACK + rwnd=8KB ── │         │
   │     │                                  │         │
   │     │  cwnd = min(cwnd, rwnd) = 8KB    │         │
   │     │                                  │         │
   │     │  receiver buffer ভরা হলে rwnd=0  │         │
   │     │  → sender stop                   │         │
   │     │                                  │         │
   │   Application কোনো signal ছাড়াই       │         │
   │   send() block হয়ে যাবে                │         │
   └──────────────────────────────────────────────────┘
```

- **rwnd** (receive window): receiver-এর free buffer space → ACK packet-এ advertise
- **cwnd** (congestion window): sender-এর congestion estimate (network)
- Effective send rate = `min(rwnd, cwnd)`

> Application-level backpressure (Node streams, gRPC) eventually এই TCP mechanism-এর উপর rely করে।

আরও বিস্তারিত: [TCP/UDP](../14-networking/tcp-udp.md)

---

## 📨 Kafka consumer backpressure

Kafka-এ producer ও consumer fully decoupled — broker queue। কিন্তু **slow consumer** = ever-growing
**lag**।

### `pause()` ও `resume()`

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ clientId: 'pathao-foods', brokers: ['kafka:9092'] });
const consumer = kafka.consumer({
  groupId: 'order-processor',
  maxBytes: 1_048_576,      // 1MB per partition fetch
  sessionTimeout: 30_000,
});

await consumer.connect();
await consumer.subscribe({ topic: 'orders.placed', fromBeginning: false });

const inflight = new Set();
const MAX_INFLIGHT = 100;

await consumer.run({
  partitionsConsumedConcurrently: 4,
  eachMessage: async ({ topic, partition, message }) => {
    if (inflight.size >= MAX_INFLIGHT) {
      consumer.pause([{ topic, partitions: [partition] }]);
      // background-এ recover মেকানিজম
      setTimeout(() => consumer.resume([{ topic, partitions: [partition] }]), 100);
    }

    const id = message.offset;
    inflight.add(id);
    try {
      await processOrder(JSON.parse(message.value.toString()));
    } finally {
      inflight.delete(id);
    }
  },
});
```

### Tuning knobs (Kafka)

```
┌─────────────────────────────┬────────────────────────────────────────┐
│ Config                      │ ব্যাখ্যা                                 │
├─────────────────────────────┼────────────────────────────────────────┤
│ max.poll.records            │ poll() এ একসাথে কতগুলো record (def 500)│
│ max.poll.interval.ms        │ এর মধ্যে process শেষ না হলে rebalance   │
│ fetch.max.bytes             │ একবারে fetch ভলিউম                      │
│ max.partition.fetch.bytes   │ per-partition limit                     │
│ enable.auto.commit          │ false রাখুন (ম্যানুয়াল ack নিরাপদ)       │
└─────────────────────────────┴────────────────────────────────────────┘
```

### Lag-based autoscaling (KEDA)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: order-processor }
spec:
  scaleTargetRef: { name: order-processor }
  minReplicaCount: 2
  maxReplicaCount: 50
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-processor
        topic: orders.placed
        lagThreshold: "1000"        # 1000+ lag হলে scale up
```

আরও বিস্তারিত: [Kafka Performance](../18-kafka/kafka-performance.md)

---

## 🐰 Message Queue (RabbitMQ, SQS) prefetch

### RabbitMQ: `prefetch_count` (consumer credit)

```javascript
const amqp = require('amqplib');

const conn = await amqp.connect('amqp://...');
const ch = await conn.createChannel();
await ch.prefetch(10);   // ← একসাথে ১০টা unacked message-এর বেশি না
await ch.consume('sms.outbox', async (msg) => {
  await sendSms(JSON.parse(msg.content));
  ch.ack(msg);
});
```

- Prefetch = 10 → broker maximum 10 unacked message এই consumer-কে পাঠাবে
- Consumer slow হলে → broker queue grow → producer eventually block (যদি bounded queue)
- Prefetch খুব বেশি = একটি consumer অনেক message hoard করছে → poor distribution
- খুব কম = network round-trip overhead

### AWS SQS: in-flight messages

SQS-এ "in-flight" বলে যেগুলো received but not yet deleted। প্রতি queue-এর per-consumer in-flight cap
(default 120,000)।

```javascript
const { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');
// long-poll WaitTimeSeconds=20, MaxNumberOfMessages=10
```

---

## 🌐 WebSockets ও SSE

### `ws` library: `bufferedAmount`

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  // Server → client high-frequency price tick (Daraz live auction)
  setInterval(() => {
    if (ws.bufferedAmount > 1_048_576) {  // 1MB pending
      // backpressure: client slow নেটওয়ার্ক
      return; // skip this tick
    }
    ws.send(JSON.stringify({ price: getCurrentPrice() }));
  }, 100);
});
```

`bufferedAmount` = OS socket buffer-এ যা write হয়েছে কিন্তু ACK আসেনি। বেশি হলে drop / sample করুন।

### SSE-এর সমস্যা

SSE-এ built-in backpressure **নেই**। সার্ভার যদি একতরফা পাঠাতে থাকে → client slow হলে proxy/LB
buffer ভরে → connection drop। সমাধান:

- Server-side rate limit (per-client emit interval)
- Coalesce (multiple update merge)
- TCP-level backpressure-এর উপর rely (Node-এ `res.write()` returns false)

```javascript
app.get('/events', (req, res) => {
  res.set({ 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache' });
  let lastEmit = 0;
  bus.on('order-update', (data) => {
    const now = Date.now();
    if (now - lastEmit < 200) return;     // 5 Hz max
    const ok = res.write(`data: ${JSON.stringify(data)}\n\n`);
    if (!ok) {
      // TCP buffer ভরা → এই client পিছিয়ে আছে
      bus.pause(); res.once('drain', () => bus.resume());
    }
    lastEmit = now;
  });
});
```

আরও: [WebSockets](../14-networking/websockets.md), [Server-Sent Events](../06-api-design/server-sent-events.md)

---

## 🗃️ Database connection pool: implicit backpressure

Database connection pool **bounded resource** — এটি implicit backpressure দেয়:

```
   Pool size = 20
   ──────────────────────────────────────────────────
   Active queries: 20  →  21st query waits for free conn
                           (acquireTimeout এর পরে error)
```

```javascript
const { Pool } = require('pg');
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 2000,   // 2s এর মধ্যে conn না পেলে error
});

try {
  const result = await pool.query('SELECT * FROM orders WHERE user_id=$1', [userId]);
} catch (err) {
  if (err.message.includes('timeout')) {
    // backpressure signal: DB overloaded
    metric('db.pool.exhausted', 1);
    return cachedFallback(userId);
  }
  throw err;
}
```

Daraz peak time-এ pool exhaust হলে — এটাই signal: হয় শেড লোড করো (load shedding), অথবা cached
serve করো।

---

## 🐘 PHP-এ backpressure (request-driven model)

PHP-FPM-এ প্রতিটি request আলাদা process — তাই Node.js-এর মতো in-process pause/resume নেই। কিন্তু
backpressure-এর **semantic** এখানেও দরকার:

### ১. Generators (yield) — memory-bound

```php
<?php
// ❌ ভুল — পুরো ফাইল RAM-এ
function loadOrdersBad(string $csvPath): array {
    return array_map('str_getcsv', file($csvPath));
    // ১০ লাখ row × 1KB = 1GB RAM
}

// ✅ সঠিক — generator (lazy, streaming)
function loadOrders(string $csvPath): \Generator {
    $h = fopen($csvPath, 'r');
    fgetcsv($h); // header skip
    while (($row = fgetcsv($h)) !== false) {
        yield $row;
    }
    fclose($h);
}

foreach (loadOrders('orders-1M.csv') as $row) {
    processOrder($row);   // RAM ~constant
}
```

Generator-এ consumer যতটা pull করছে, producer ততটাই push করছে — natural backpressure।

### ২. Laravel Queue + chunking

```php
// Daraz: ৫০ লাখ user-কে notification পাঠানো
class SendBigSaleNotification implements ShouldQueue
{
    public function handle()
    {
        User::where('subscribed', true)
            ->chunk(1000, function ($users) {  // ১০০০-এর batch
                foreach ($users as $user) {
                    SendOneNotification::dispatch($user)
                        ->onQueue('notifications')
                        ->delay(now()->addSeconds(rand(0, 600))); // jitter
                }
            });
    }
}
```

- Bounded chunk → memory bound
- Queue worker pool size = effective consumer rate
- Job dispatch delay-এ jitter → downstream API spike avoid

### ৩. Worker pool — Supervisor

```ini
[program:sms-worker]
command=php artisan queue:work redis --queue=sms --tries=3 --timeout=60
numprocs=10                    ; ১০টি concurrent worker = max throughput
stopwaitsecs=70
autorestart=true
```

10 worker → Twilio/SMS gateway-এ একসাথে 10 call-এর বেশি না → implicit rate limit।

### ৪. ReactPHP / Swoole — true async

```php
// ReactPHP streaming API
use React\Http\Browser;
use React\Stream\WritableResourceStream;

$client = new Browser();
$client->requestStreaming('GET', 'https://api.daraz.com/products/feed.csv')
       ->then(function ($response) {
           $body = $response->getBody();
           $body->on('data', function ($chunk) use (&$buffer) {
               // backpressure: $body->pause() / $body->resume()
               processChunk($chunk);
           });
       });
```

---

## 🆚 Backpressure vs Throttling vs Rate Limiting

```
┌──────────────────┬──────────────────────────────────────────────────────┐
│ Concept          │ ব্যাখ্যা                                                │
├──────────────────┼──────────────────────────────────────────────────────┤
│ Backpressure     │ Consumer → Producer signal: "আমি cope করতে পারছি না"  │
│                  │ Direction: downstream → upstream (reverse)            │
│                  │ Adaptive (current capacity-based)                     │
│                  │ Example: TCP rwnd, RxJS request(n)                    │
│                  │                                                       │
│ Throttling       │ Producer/consumer-এর rate কমানো (smoothing)           │
│                  │ Direction: self-imposed                               │
│                  │ Static or adaptive                                    │
│                  │ Example: setInterval-এর মধ্যে throttle, debounce      │
│                  │                                                       │
│ Rate Limiting    │ Producer-এ external limit enforce (ক্লায়েন্ট-সম্পর্কিত)│
│                  │ Direction: provider → consumer                        │
│                  │ Static (per-key quota)                                │
│                  │ Example: 100 req/min per API key, token bucket        │
│                  │                                                       │
│ Load Shedding    │ Overload হলে কিছু request reject (system-wide)       │
│                  │ Direction: server → all clients                       │
│                  │ Adaptive (system health-based)                        │
│                  │ Example: 503 + Retry-After                            │
└──────────────────┴──────────────────────────────────────────────────────┘
```

### Combined diagram

```
   Client ──[rate limit per key]──▶ API Gateway
            ↑                          │
            │                          ▼
   Retry-After                  [load shedding if 80% capacity]
                                       │
                                       ▼
                                Service ──[backpressure]──▶ Kafka
                                                              │
                                                              ▼
                                                        Consumer
                                                       [throttle internal]
```

আরও: [Rate Limiting](../03-system-design/rate-limiting.md), [Load Shedding](../03-system-design/load-shedding.md)

---

## 🇧🇩 BD Scenario: Pathao Foods + bKash SMS

### Pathao Foods — peak hour Kafka backpressure

**সমস্যা:** Friday রাত ৯টা — ১০ মিনিটে ৫০K order placed। `order-processor` consumer ৫K/min process
করতে পারে। Lag বাড়ছে।

```
   Producer (REST API): 50K orders / 10 min = 5K orders / min × 10 = 50K
   Consumer:            5K orders / min
   Lag growth:          0 → 45K in 10 min
```

**Solution stack:**

```
   ┌────────────────────────────────────────────────────────────────┐
   │ 1. Kafka topic partition: 16 (was 4) → parallelism 4×            │
   │ 2. Consumer group: 16 instance, partitionsConsumedConcurrently=4│
   │ 3. KEDA autoscale: lag > 1K → scale up; max 32 pod              │
   │ 4. Consumer per-msg processing < 200ms (DB batch insert)        │
   │ 5. Backpressure: max-inflight=100, pause partition যদি ভরে যায়   │
   │ 6. Idempotency: retry safe (orderId unique constraint)          │
   └────────────────────────────────────────────────────────────────┘
```

**Result:** lag 5 মিনিটে শূন্য, P99 end-to-end latency 30s → 800ms।

### bKash SMS dispatch — bounded worker pool

**সমস্যা:** Payment success-এর পরে SMS পাঠানো। SMS gateway 200 req/sec accept করে। flash hour-এ
১০K payment/min → ১৬৬ SMS/sec (ok), কিন্তু worker pool unbounded → connection exhaust।

```php
// Laravel Horizon configuration
'environments' => [
    'production' => [
        'sms-supervisor' => [
            'connection' => 'redis',
            'queue' => ['sms.transaction', 'sms.otp'],
            'balance' => 'auto',
            'maxProcesses' => 50,     // ← bounded
            'minProcesses' => 5,
            'tries' => 3,
            'timeout' => 30,
            'memory' => 256,
            'sleep' => 1,
        ],
    ],
],
```

50 worker × 4 SMS/sec/worker = 200 SMS/sec — gateway capacity match। অতিরিক্ত job queue-এ wait
করছে (bounded backpressure)। Queue depth alert:

```
   ALERT: redis:queues:sms.transaction.length > 5000 for 2m
   → autoscale worker বা SMS gateway add channel
```

---

## ⚠️ Anti-patterns

### ১. Unbounded queue
```javascript
// ❌
const queue = [];
emitter.on('event', e => queue.push(e));   // memory blowup
// ✅
const queue = new BoundedQueue(10_000);   // overflow → drop oldest
```

### ২. Ignoring write() return value
```javascript
// ❌
stream.on('data', d => writable.write(d));  // backpressure blind
// ✅ pipeline() or pause/resume
```

### ৩. `forEach` on async (no concurrency control)
```javascript
// ❌ — সব একসাথে fire, downstream API DDoS
items.forEach(async i => await callApi(i));

// ✅ — bounded concurrency (p-limit / Promise pool)
import pLimit from 'p-limit';
const limit = pLimit(10);
await Promise.all(items.map(i => limit(() => callApi(i))));
```

### ৪. Auto-commit Kafka before processing
```
   ❌ enable.auto.commit=true → consumer crash → in-flight msg lost
   ✅ Manual ack only after successful process
```

### ৫. Prefetch too high (RabbitMQ)
```
   ❌ prefetch=10000 → ১টি slow consumer সব hoard করে → অন্য worker idle
   ✅ prefetch ≈ avg processing rate × ack RTT
```

### ৬. Buffer-everything-in-memory CSV
```php
// ❌ file() → 1GB RAM
$rows = file('big.csv');
// ✅ generator + fgetcsv loop
```

### ৭. SSE without TCP backpressure check
```
   ❌ res.write() loop → slow client → proxy timeout → connection storm
   ✅ check return value, drain event
```

### ৮. Backpressure ছাড়া retry
```
   Downstream slow → retry → আরও load → আরও slow → retry storm
   ✅ Backpressure + circuit breaker + jitter
```

---

## ✅ Checklist

```
General:
[ ] সব queue/buffer bounded (highWaterMark / maxSize / capacity)
[ ] Overflow strategy explicit (drop / reject / block)
[ ] End-to-end memory test (load > steady state)

Node.js:
[ ] pipeline() ব্যবহার, raw pipe() নয়
[ ] write() return value check
[ ] objectMode stream-এ highWaterMark (count) সেট
[ ] Async generator + for-await-of safe pattern

Reactive:
[ ] onBackpressureBuffer/Drop/Latest explicit চয়ন
[ ] request(n) sensible (UI: latest; analytics: drop)
[ ] debounce/throttle/sample সঠিক operator UI event-এ

Kafka:
[ ] max.poll.records, max.poll.interval tuned
[ ] Manual commit (auto.commit=false)
[ ] Lag-based autoscaling (KEDA / custom)
[ ] Pause/resume on inflight cap

MQ:
[ ] RabbitMQ prefetch তুলিত
[ ] SQS in-flight cap সচেতন
[ ] Dead-letter queue for poison messages

WebSocket / SSE:
[ ] bufferedAmount check
[ ] Server-side rate / sample
[ ] Client reconnect with exponential backoff

DB:
[ ] Connection pool size traffic-aware
[ ] acquireTimeout configured
[ ] Pool exhaustion = fallback (cached / 503)

PHP:
[ ] Generator pattern for large datasets
[ ] Queue chunking
[ ] Bounded worker pool (Horizon/Supervisor)

Observability:
[ ] Queue depth metric
[ ] Drop count metric
[ ] Lag (Kafka), in-flight (SQS)
[ ] Memory growth graph (steady state baseline)

Testing:
[ ] Slow consumer test (artificial sleep)
[ ] Producer burst test
[ ] Recovery test (consumer scale up → lag drain time)
```

---

## 📚 আরও পড়ুন

- [Circuit Breaker](./circuit-breaker.md)
- [Load Shedding](../03-system-design/load-shedding.md)
- [Rate Limiting](../03-system-design/rate-limiting.md)
- [Thread Pools](./thread-pools.md)
- [Async/Await](./async-await.md)
- [Event Loop](./event-loop.md)
- [Kafka Performance](../18-kafka/kafka-performance.md)
- [WebSockets](../14-networking/websockets.md)
- [Server-Sent Events](../06-api-design/server-sent-events.md)
- [TCP/UDP](../14-networking/tcp-udp.md)

> **চূড়ান্ত কথা:** Backpressure = honesty। Producer-কে বলা দরকার "আমি যা পারি ততটাই দাও"। Unbounded
> buffer = সমস্যা পরে আসবে, কিন্তু আসবেই — production-এ OOM-এর রূপে।
