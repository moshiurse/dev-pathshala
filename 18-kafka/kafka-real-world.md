# 📘 Kafka বাস্তব প্রজেক্ট কেস স্টাডি (Real-World Use Cases)

## 📖 ভূমিকা

Kafka আজকের দুনিয়ায় সবচেয়ে বেশি ব্যবহৃত distributed streaming platform। LinkedIn, Netflix, Uber,
Airbnb, Spotify সহ হাজার হাজার কোম্পানি প্রতিদিন ট্রিলিয়ন ইভেন্ট প্রসেস করে Kafka দিয়ে।

বাংলাদেশের কোম্পানিগুলোও একই ধরনের সমস্যা সমাধান করে — Daraz-এ লক্ষ লক্ষ অর্ডার,
Pathao-তে রিয়েল-টাইম রাইড ট্র্যাকিং, bKash/Nagad-এ কোটি কোটি ট্রানজ্যাকশন।
এই ফাইলে আমরা বাস্তব বাংলাদেশ কনটেক্সটে Kafka-র ব্যবহার দেখব — প্রতিটি কেস স্টাডিতে
আর্কিটেকচার ডায়াগ্রাম, কোড এবং শেখার বিষয় থাকবে।

### 🌍 কারা Kafka ব্যবহার করে?

```
কোম্পানি              দৈনিক ইভেন্ট          ব্যবহারের ক্ষেত্র
─────────────────────────────────────────────────────────────────
LinkedIn               7 ট্রিলিয়ন+          Activity tracking, messaging
Netflix                1 ট্রিলিয়ন+          Real-time recommendations
Uber                   পেটাবাইট/দিন          Ride tracking, surge pricing
Airbnb                 বিলিয়ন+              Search, payments, messaging
Spotify                বিলিয়ন+              Music recommendations
```

### 🇧🇩 বাংলাদেশে সম্ভাব্য ব্যবহার

```
বাংলাদেশি কোম্পানি      সমস্যা                    Kafka সমাধান
──────────────────────────────────────────────────────────────────
Daraz                   Flash sale traffic        Event-driven order pipeline
Pathao                  Real-time tracking        Location streaming
bKash/Nagad             Transaction processing    Exactly-once payment events
Grameenphone            CDR processing            Real-time telecom analytics
Chaldal                 Delivery optimization     Event-driven logistics
Sheba.xyz               Service matching          Async service orchestration
```

---

## 🛒 কেস স্টাডি ১: ই-কমার্স অর্ডার প্রসেসিং (Daraz-like)

### 📌 সমস্যার বিবরণ

Daraz-এর মতো ই-কমার্স প্ল্যাটফর্মে প্রতিদিন হাজার হাজার অর্ডার আসে। ১১.১১ সেল-এর সময়
এক মিনিটে হাজার হাজার অর্ডার হয়। প্রতিটি অর্ডারে অনেকগুলো কাজ হয় — পেমেন্ট ভেরিফাই,
ইনভেন্টরি চেক, নোটিফিকেশন পাঠানো, অ্যানালিটিক্স আপডেট। সবকিছু synchronous করলে
সিস্টেম ভেঙে পড়বে।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
                          ┌─────────────────────────────────────────────────┐
                          │           Kafka Cluster (3 Brokers)             │
                          │                                                 │
 ┌──────────┐  HTTP POST  │  ┌─────────────────┐                           │
 │ Customer ├────────────►│  │  order-placed    │ (6 partitions)            │
 │  (App)   │             │  │  (main topic)    │                           │
 └──────────┘             │  └────────┬─────────┘                           │
                          │           │                                     │
                          │     ┌─────┼──────┬──────────┬──────────┐       │
                          │     ▼     ▼      ▼          ▼          ▼       │
                          │  ┌──────┐┌────┐┌──────┐ ┌───────┐ ┌───────┐   │
                          │  │Pay-  ││Inv-││Noti- │ │Analy- │ │Search │   │
                          │  │ment  ││ent-││fica- │ │tics   │ │Service│   │
                          │  │Svc   ││ory ││tion  │ │Svc    │ │       │   │
                          │  └──┬───┘└──┬─┘└──────┘ └───────┘ └───────┘   │
                          │     ▼       ▼                                   │
                          │  ┌──────┐┌──────────┐                           │
                          │  │pay-  ││inventory-│                           │
                          │  │ment- ││reserved  │                           │
                          │  │done  ││          │                           │
                          │  └──┬───┘└──────────┘                           │
                          │     ▼                                           │
                          │  ┌───────────────┐  ┌────────────────┐         │
                          │  │ order-shipped  │→ │ order-delivered│         │
                          │  └───────────────┘  └────────────────┘         │
                          └─────────────────────────────────────────────────┘
```

### 📊 টপিক ডিজাইন

```
Topic Name              Partitions   Retention   Key              Use
────────────────────────────────────────────────────────────────────────
order-placed            6            7 days      order_id         নতুন অর্ডার
payment-processed       6            7 days      order_id         পেমেন্ট সম্পন্ন
inventory-reserved      4            3 days      product_id       স্টক রিজার্ভ
order-shipped           4            14 days     order_id         শিপমেন্ট
order-delivered         4            30 days     order_id         ডেলিভারি
order-cancelled         4            30 days     order_id         ক্যান্সেলেশন
```

### 📦 Consumer Groups

```
Consumer Group                  Subscribed Topics          Instances
──────────────────────────────────────────────────────────────────────
payment-service-group           order-placed               3
inventory-service-group         order-placed               3
notification-service-group      order-placed,              4
                                payment-processed,
                                order-shipped
analytics-service-group         সব topics                   2
search-update-group             order-placed,              2
                                inventory-reserved
```

### 💻 PHP কোড: Order Producer

```php
<?php
// OrderProducer.php — Daraz-like অর্ডার ইভেন্ট প্রডিউসার

class OrderProducer
{
    private $producer;
    private $topic;

    public function __construct()
    {
        $config = new RdKafka\Conf();
        $config->set('bootstrap.servers', 'kafka1:9092,kafka2:9092,kafka3:9092');
        $config->set('acks', 'all');                    // সব replica তে লেখা নিশ্চিত
        $config->set('retries', '3');                    // ব্যর্থ হলে ৩ বার retry
        $config->set('retry.backoff.ms', '100');
        $config->set('enable.idempotence', 'true');      // duplicate এড়াতে

        $this->producer = new RdKafka\Producer($config);
        $this->topic = $this->producer->newTopic('order-placed');
    }

    public function publishOrder(array $orderData): bool
    {
        $event = [
            'event_id'   => uniqid('ord_', true),
            'event_type' => 'ORDER_PLACED',
            'timestamp'  => date('c'),
            'data'       => [
                'order_id'    => $orderData['order_id'],
                'customer_id' => $orderData['customer_id'],
                'items'       => $orderData['items'],
                'total_amount'=> $orderData['total_amount'],
                'currency'    => 'BDT',
                'address'     => $orderData['shipping_address'],
                'payment_method' => $orderData['payment_method'],
            ],
            'metadata' => [
                'source'  => 'web-app',
                'version' => '1.0',
            ],
        ];

        $payload = json_encode($event);

        // order_id দিয়ে partition key — একই অর্ডারের সব ইভেন্ট একই partition-এ যাবে
        $this->topic->producev(
            RD_KAFKA_PARTITION_UA,
            0,
            $payload,
            $orderData['order_id']  // partition key
        );

        $this->producer->flush(5000);
        return true;
    }
}

// ব্যবহার:
$producer = new OrderProducer();
$producer->publishOrder([
    'order_id'         => 'DRZ-2024-000123',
    'customer_id'      => 'CUST-5678',
    'items'            => [
        ['product_id' => 'P001', 'name' => 'Samsung Galaxy A15', 'qty' => 1, 'price' => 18999],
        ['product_id' => 'P002', 'name' => 'Phone Cover',        'qty' => 2, 'price' => 250],
    ],
    'total_amount'     => 19499,
    'shipping_address' => 'মিরপুর-১০, ঢাকা',
    'payment_method'   => 'bkash',
]);
```

### 💻 JS কোড: Notification Consumer

```javascript
// notificationConsumer.js — অর্ডারের নোটিফিকেশন পাঠানোর সার্ভিস

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'notification-service',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const consumer = kafka.consumer({ groupId: 'notification-service-group' });

async function startNotificationService() {
  await consumer.connect();

  // একাধিক topic subscribe — অর্ডারের বিভিন্ন পর্যায়ে নোটিফিকেশন পাঠাতে হবে
  await consumer.subscribe({ topics: ['order-placed', 'payment-processed', 'order-shipped'] });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      const orderId = event.data.order_id;
      const customerId = event.data.customer_id;

      switch (topic) {
        case 'order-placed':
          await sendSMS(customerId, `আপনার অর্ডার ${orderId} সফলভাবে প্লেস হয়েছে! 🛒`);
          await sendEmail(customerId, 'অর্ডার কনফার্মেশন', buildOrderConfirmEmail(event));
          break;

        case 'payment-processed':
          await sendSMS(customerId, `অর্ডার ${orderId}-এর পেমেন্ট সফল হয়েছে। ধন্যবাদ! ✅`);
          break;

        case 'order-shipped':
          const trackingId = event.data.tracking_id;
          await sendSMS(
            customerId,
            `আপনার অর্ডার ${orderId} শিপ হয়েছে। ট্র্যাকিং: ${trackingId} 🚚`
          );
          await sendPushNotification(customerId, {
            title: 'অর্ডার শিপ হয়েছে!',
            body: `ট্র্যাকিং নম্বর: ${trackingId}`,
          });
          break;
      }
    },
  });
}

startNotificationService().catch(console.error);
```

### 🔥 Flash Sale ট্রাফিক ম্যানেজমেন্ট

```
সাধারণ দিন:     ~100 orders/min   ════════░░░░░░░░░░░░░░░░
১১.১১ সেলের দিন: ~10,000 orders/min ════════════════════════

সমাধান:
  ১. Partition বাড়ানো: 6 → 24 (সেলের আগে)
  ২. Consumer instance বাড়ানো: 3 → 12
  ৩. Burst buffer: Kafka নিজেই buffer — consumer নিজের speed-এ পড়বে
  ৪. Back-pressure: Consumer slow হলে Kafka তে message জমা থাকবে, কিছু হারাবে না
```

### ✅ শেখার বিষয়

- **Partition key = order_id** → একই অর্ডারের সব ইভেন্ট ordered থাকে
- **Consumer group প্রতি সার্ভিস** → প্রতিটি সার্ভিস স্বাধীনভাবে কাজ করে
- **Idempotent producer** → duplicate অর্ডার ইভেন্ট এড়ায়
- **Flash sale তে Kafka buffer** হিসেবে কাজ করে — কোনো ডাটা হারায় না

---

## 🚗 কেস স্টাডি ২: রিয়েল-টাইম রাইড আপডেট (Pathao-like)

### 📌 সমস্যার বিবরণ

Pathao-তে হাজার হাজার ড্রাইভার একসাথে রাস্তায় থাকে। প্রতি ৩ সেকেন্ডে তাদের GPS location
পাঠাতে হয়। রাইডার যখন রাইড রিকোয়েস্ট করে, নিকটতম ড্রাইভার খুঁজে ম্যাচ করতে হয়।
এই পুরো সিস্টেম রিয়েল-টাইম এবং high-throughput হতে হবে।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌──────────────┐                              ┌──────────────┐
│  Driver App  │  GPS every 3s                │  Rider App   │
│  (1000s of   ├─────────────┐    ┌───────────┤  (ride req)  │
│   drivers)   │             │    │           │              │
└──────────────┘             ▼    ▼           └──────┬───────┘
                     ┌───────────────────┐           │
                     │   Kafka Cluster   │           │
                     │                   │           │
                     │ [driver-location] │◄──────────┘
                     │  (12 partitions)  │    [ride-request]
                     │                   │    (6 partitions)
                     │ [ride-matched]    │
                     │ [ride-status]     │
                     │ [ride-completed]  │
                     └────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
      ┌──────────────┐ ┌───────────┐ ┌──────────────┐
      │ Ride Matching │ │ Live Map  │ │  ETA Calc    │
      │ Service      │ │ Service   │ │  Service     │
      │              │ │           │ │              │
      │ (নিকটতম     │ │ (রিয়েল-  │ │ (আনুমানিক   │
      │  ড্রাইভার   │ │  টাইম ম্যাপ│ │  সময় হিসাব) │
      │  খুঁজে বের  │ │  আপডেট)  │ │              │
      │  করে)        │ │           │ │              │
      └──────────────┘ └───────────┘ └──────────────┘
```

### 🗺️ Geo-Partitioning কৌশল (ঢাকা জোন ভিত্তিক)

```
ঢাকা শহরকে জোনে ভাগ করে partition key বানানো হয়:

  ┌──────────────────────────────────────┐
  │           ঢাকা শহর                    │
  │                                      │
  │   ┌──────────┐  ┌──────────┐        │
  │   │ Zone-1   │  │ Zone-2   │        │
  │   │ উত্তরা,   │  │ মিরপুর,  │        │
  │   │ বিমানবন্দর│  │ মোহাম্মদপুর│       │
  │   │ Part: 0,1│  │ Part: 2,3│        │
  │   └──────────┘  └──────────┘        │
  │   ┌──────────┐  ┌──────────┐        │
  │   │ Zone-3   │  │ Zone-4   │        │
  │   │ গুলশান,  │  │ মতিঝিল, │        │
  │   │ বনানী    │  │ পুরান ঢাকা│        │
  │   │ Part: 4,5│  │ Part: 6,7│        │
  │   └──────────┘  └──────────┘        │
  │   ┌──────────┐  ┌──────────┐        │
  │   │ Zone-5   │  │ Zone-6   │        │
  │   │ ধানমন্ডি,│  │ যাত্রাবাড়ী│       │
  │   │ লালমাটিয়া│  │ ডেমরা    │        │
  │   │ Part:8,9 │  │Part:10,11│        │
  │   └──────────┘  └──────────┘        │
  └──────────────────────────────────────┘

Partition Key = zone_id (latitude/longitude থেকে calculate)
একই জোনের সব ড্রাইভার একই partition-এ → দ্রুত matching
```

### 💻 JS কোড: Driver Location Producer

```javascript
// driverLocationProducer.js — ড্রাইভার অ্যাপ থেকে GPS লোকেশন স্ট্রিম

const { Kafka, Partitioners } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'driver-app',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const producer = kafka.producer({
  createPartitioner: Partitioners.DefaultPartitioner,
});

// GPS coordinate থেকে zone ID বের করা
function getZoneId(lat, lng) {
  // ঢাকার GPS range: lat 23.7-23.9, lng 90.3-90.5
  const latZone = Math.floor((lat - 23.7) / 0.067); // ৩ ভাগ
  const lngZone = Math.floor((lng - 90.3) / 0.1);    // ২ ভাগ
  return `zone-${latZone}-${lngZone}`;
}

async function streamDriverLocation(driverId) {
  await producer.connect();

  // প্রতি ৩ সেকেন্ডে location পাঠাও
  setInterval(async () => {
    const location = getCurrentGPSLocation(); // device GPS API

    const zoneId = getZoneId(location.lat, location.lng);

    await producer.send({
      topic: 'driver-location',
      messages: [{
        key: zoneId,         // zone ভিত্তিক partitioning
        value: JSON.stringify({
          driver_id: driverId,
          lat: location.lat,
          lng: location.lng,
          heading: location.heading,
          speed: location.speed,
          timestamp: Date.now(),
          vehicle_type: 'bike',
          is_available: true,
        }),
        headers: { 'driver-id': driverId },
      }],
    });
  }, 3000); // ৩ সেকেন্ড interval
}

streamDriverLocation('DRV-DHAKA-4521');
```

### 💻 JS কোড: Ride Matching Consumer

```javascript
// rideMatchingConsumer.js — রাইড ম্যাচিং সার্ভিস

const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

const kafka = new Kafka({
  clientId: 'ride-matching-service',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const consumer = kafka.consumer({ groupId: 'ride-matching-group' });
const producer = kafka.producer();
const redis = new Redis();

// ড্রাইভার লোকেশন Redis-এ ক্যাশ (GEOADD ব্যবহার)
async function updateDriverLocation(event) {
  await redis.geoadd(
    'drivers:available',
    event.lng, event.lat, event.driver_id
  );
  // ড্রাইভারের শেষ আপডেট সময় সেভ
  await redis.set(`driver:${event.driver_id}:last_seen`, Date.now(), 'EX', 30);
}

// নিকটতম ড্রাইভার খোঁজা
async function findNearestDriver(riderLat, riderLng, radiusKm = 3) {
  const nearbyDrivers = await redis.georadius(
    'drivers:available',
    riderLng, riderLat,
    radiusKm, 'km',
    'WITHDIST', 'ASC', 'COUNT', 5
  );
  // শুধু active ড্রাইভার (৩০ সেকেন্ডের মধ্যে location পাঠিয়েছে)
  for (const [driverId, distance] of nearbyDrivers) {
    const lastSeen = await redis.get(`driver:${driverId}:last_seen`);
    if (lastSeen && Date.now() - parseInt(lastSeen) < 30000) {
      return { driverId, distance: parseFloat(distance) };
    }
  }
  return null;
}

async function startRideMatching() {
  await consumer.connect();
  await producer.connect();

  await consumer.subscribe({ topics: ['driver-location', 'ride-request'] });

  await consumer.run({
    eachMessage: async ({ topic, message }) => {
      const event = JSON.parse(message.value.toString());

      if (topic === 'driver-location') {
        await updateDriverLocation(event);
      }

      if (topic === 'ride-request') {
        const match = await findNearestDriver(event.pickup_lat, event.pickup_lng);
        if (match) {
          await producer.send({
            topic: 'ride-matched',
            messages: [{
              key: event.ride_id,
              value: JSON.stringify({
                ride_id: event.ride_id,
                rider_id: event.rider_id,
                driver_id: match.driverId,
                distance_km: match.distance,
                matched_at: new Date().toISOString(),
              }),
            }],
          });
        }
      }
    },
  });
}

startRideMatching().catch(console.error);
```

### ✅ শেখার বিষয়

- **High-frequency data** (প্রতি ৩ সেকেন্ড) → Kafka অনায়াসে handle করে
- **Geo-partitioning** → একই এলাকার ড্রাইভার-রাইডার একই partition-এ, ম্যাচিং দ্রুত হয়
- **Redis + Kafka combo** → Kafka দিয়ে stream, Redis দিয়ে real-time geo-query
- **Short retention** → location data ২৪ ঘণ্টা রাখলেই যথেষ্ট, পুরনো data মুছে যাক

---

## 💰 কেস স্টাডি ৩: পেমেন্ট ইভেন্ট প্রসেসিং (bKash/Nagad-like)

### 📌 সমস্যার বিবরণ

bKash/Nagad-এর মতো মোবাইল ফিন্যান্সিয়াল সার্ভিসে প্রতিদিন কোটি কোটি টাকার লেনদেন হয়।
এখানে কোনো ট্রানজ্যাকশন হারানো যাবে না, duplicate হওয়া যাবে না, এবং বাংলাদেশ ব্যাংকের
কমপ্লায়েন্স মানতে হবে। Exactly-once semantics এখানে অত্যন্ত গুরুত্বপূর্ণ।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌────────────┐      ┌───────────────────┐
│  User App  ├─────►│ Transaction API   │
│ (Send ৳500)│      │ (Gateway)         │
└────────────┘      └────────┬──────────┘
                             │
                             ▼
                   ┌─────────────────────┐
                   │ [transaction-init]   │  (exactly-once)
                   │  (12 partitions)     │
                   └─────────┬───────────┘
                             │
          ┌──────────────────┼──────────────────────┐
          ▼                  ▼                       ▼
  ┌───────────────┐ ┌───────────────┐    ┌──────────────────┐
  │ Fraud         │ │ Balance       │    │ Notification     │
  │ Detection     │ │ Service       │    │ Service          │
  │               │ │               │    │                  │
  │ ML মডেল দিয়ে │ │ ব্যালেন্স    │    │ SMS: "৳500      │
  │ সন্দেহজনক    │ │ আপডেট ও      │    │  পাঠানো হয়েছে"  │
  │ লেনদেন চেক   │ │ ডাবল-এন্ট্রি │    │                  │
  └───────┬───────┘ └───────┬───────┘    └──────────────────┘
          ▼                 ▼
  ┌───────────────┐ ┌───────────────┐    ┌──────────────────┐
  │[fraud-result] │ │[balance-upd]  │    │ Ledger Service   │
  └───────────────┘ └───────────────┘    │ (হিসাবের খাতা)  │
                                          └────────┬─────────┘
                                                   ▼
                                          ┌──────────────────┐
                                          │ [ledger-entry]   │
                                          └────────┬─────────┘
                                                   ▼
                                          ┌──────────────────┐
                                          │ Regulatory       │
                                          │ Reporting        │
                                          │ (বাংলাদেশ ব্যাংক │
                                          │  কমপ্লায়েন্স)   │
                                          └──────────────────┘
```

### 📊 Saga Pattern: Send Money Flow

```
Send Money: রহিম → করিম, ৳500

Step 1: [transaction-initiated]
        ─── রহিম ৳500 পাঠাতে চায় ───

Step 2: Fraud Check
        ├── ✅ PASS → চালিয়ে যাও
        └── ❌ FAIL → [transaction-rejected] → টাকা ফেরত

Step 3: Sender Balance Debit
        ├── ✅ রহিমের ব্যালেন্স: ৳2000 → ৳1500
        └── ❌ Insufficient → [transaction-failed] → নোটিফাই

Step 4: Receiver Balance Credit
        ├── ✅ করিমের ব্যালেন্স: ৳800 → ৳1300
        └── ❌ FAIL → Compensate: রহিমকে ৳500 ফেরত

Step 5: [transaction-completed]
        ─── দুইজনকেই SMS পাঠাও ───

Compensating Transaction (rollback):
যেকোনো step fail করলে আগের সব step reverse করতে হবে।
Kafka তে compensating event পাঠানো হয়।
```

### 💻 PHP কোড: Transaction Producer (Exactly-Once)

```php
<?php
// TransactionProducer.php — bKash-like ট্রানজ্যাকশন ইভেন্ট

class TransactionProducer
{
    private $producer;

    public function __construct()
    {
        $config = new RdKafka\Conf();
        $config->set('bootstrap.servers', 'kafka1:9092,kafka2:9092,kafka3:9092');

        // Exactly-once সেটিংস — ফিন্যান্সিয়াল ডাটায় অত্যন্ত গুরুত্বপূর্ণ
        $config->set('enable.idempotence', 'true');
        $config->set('acks', 'all');
        $config->set('max.in.flight.requests.per.connection', '5');
        $config->set('transactional.id', 'txn-producer-bkash-01');

        $this->producer = new RdKafka\Producer($config);
        $this->producer->initTransactions(10000);
    }

    public function processSendMoney(string $senderId, string $receiverId, float $amount): void
    {
        $transactionId = 'TXN-' . date('Ymd') . '-' . bin2hex(random_bytes(8));

        // Kafka Transaction শুরু — হয় সব হবে, না হলে কিছুই হবে না
        $this->producer->beginTransaction();

        try {
            // ১. ট্রানজ্যাকশন ইনিশিয়েট ইভেন্ট
            $this->produce('transaction-initiated', $transactionId, [
                'txn_id'      => $transactionId,
                'sender_id'   => $senderId,
                'receiver_id' => $receiverId,
                'amount'      => $amount,
                'currency'    => 'BDT',
                'type'        => 'SEND_MONEY',
                'timestamp'   => date('c'),
            ]);

            // ২. Ledger entry (ডাবল-এন্ট্রি)
            $this->produce('ledger-entry', $transactionId, [
                'txn_id' => $transactionId,
                'entries' => [
                    ['account' => $senderId,   'type' => 'DEBIT',  'amount' => $amount],
                    ['account' => $receiverId, 'type' => 'CREDIT', 'amount' => $amount],
                ],
            ]);

            // সব সফল — transaction commit
            $this->producer->commitTransaction(10000);

        } catch (\Exception $e) {
            // কিছু ভুল হলে পুরো transaction abort
            $this->producer->abortTransaction(10000);
            throw $e;
        }
    }

    private function produce(string $topicName, string $key, array $data): void
    {
        $topic = $this->producer->newTopic($topicName);
        $topic->producev(RD_KAFKA_PARTITION_UA, 0, json_encode($data), $key);
        $this->producer->poll(0);
    }
}

// ব্যবহার:
$producer = new TransactionProducer();
$producer->processSendMoney('USER-RAHIM-001', 'USER-KARIM-002', 500.00);
```

### 💻 JS কোড: Fraud Detection Consumer

```javascript
// fraudDetectionConsumer.js — সন্দেহজনক লেনদেন শনাক্তকরণ

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'fraud-detection-service',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const consumer = kafka.consumer({ groupId: 'fraud-detection-group' });
const producer = kafka.producer({ idempotent: true });

// সহজ fraud detection নিয়ম
const FRAUD_RULES = {
  MAX_SINGLE_TXN: 100000,     // একবারে ১ লক্ষ টাকার বেশি
  MAX_DAILY_TXN: 500000,      // দিনে ৫ লক্ষ টাকার বেশি
  MAX_TXN_COUNT_PER_HOUR: 20, // ঘণ্টায় ২০টির বেশি লেনদেন
};

async function checkFraud(txnEvent) {
  const flags = [];

  if (txnEvent.amount > FRAUD_RULES.MAX_SINGLE_TXN) {
    flags.push('HIGH_AMOUNT');
  }

  // ঘণ্টায় কতবার লেনদেন করেছে সেটা চেক (Redis থেকে)
  const hourlyCount = await getHourlyTxnCount(txnEvent.sender_id);
  if (hourlyCount > FRAUD_RULES.MAX_TXN_COUNT_PER_HOUR) {
    flags.push('HIGH_FREQUENCY');
  }

  // আজকের মোট পরিমাণ চেক
  const dailyTotal = await getDailyTotal(txnEvent.sender_id);
  if (dailyTotal + txnEvent.amount > FRAUD_RULES.MAX_DAILY_TXN) {
    flags.push('DAILY_LIMIT_EXCEEDED');
  }

  return {
    is_suspicious: flags.length > 0,
    flags,
    risk_score: flags.length * 30, // সহজ স্কোরিং
  };
}

async function start() {
  await consumer.connect();
  await producer.connect();
  await consumer.subscribe({ topic: 'transaction-initiated' });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const txn = JSON.parse(message.value.toString());
      const result = await checkFraud(txn);

      await producer.send({
        topic: 'fraud-check-result',
        messages: [{
          key: txn.txn_id,
          value: JSON.stringify({
            txn_id: txn.txn_id,
            ...result,
            checked_at: new Date().toISOString(),
          }),
        }],
      });

      if (result.is_suspicious) {
        console.warn(`⚠️ সন্দেহজনক লেনদেন: ${txn.txn_id}, flags: ${result.flags.join(', ')}`);
      }
    },
  });
}

start().catch(console.error);
```

### ✅ শেখার বিষয়

- **Exactly-once semantics** ফিন্যান্সিয়াল সিস্টেমে বাধ্যতামূলক
- **Kafka Transactions** দিয়ে multiple topic-এ atomic write
- **Saga pattern** দিয়ে distributed transaction manage
- **Audit trail** → Kafka-র immutable log নিজেই অডিট ট্রেইল হিসেবে কাজ করে
- **বাংলাদেশ ব্যাংক কমপ্লায়েন্স** → সব লেনদেন log থাকতে হবে, Kafka এটা নিশ্চিত করে

---

## 📝 কেস স্টাডি ৪: লগ অ্যাগ্রিগেশন পাইপলাইন

### 📌 সমস্যার বিবরণ

একটি বড় প্ল্যাটফর্মে শত শত মাইক্রোসার্ভিস থাকে। প্রতিটি সার্ভিস লগ তৈরি করে। সব লগ
এক জায়গায় জমা করে সার্চ, এলার্ট এবং আর্কাইভ করতে হয়।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ User Service │  │ Order Service│  │ Payment Svc  │  │ Auth Service │
│  (Node.js)   │  │  (PHP)       │  │  (Java)      │  │  (Go)        │
│              │  │              │  │              │  │              │
│ winston/     │  │ monolog/     │  │ log4j/       │  │ zap/         │
│ pino logger  │  │ fluentd      │  │ logback      │  │ logrus       │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                 │
       └─────────────────┼─────────────────┼─────────────────┘
                         ▼                 ▼
                ┌──────────────────────────────┐
                │     Kafka Cluster            │
                │                              │
                │  [application-logs]           │
                │   (12 partitions)            │
                │   key: service_name          │
                │   retention: 7 days          │
                │                              │
                │  [error-logs]                │
                │   (4 partitions)             │
                │   retention: 30 days         │
                └──────────────┬───────────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
       ┌───────────────┐ ┌──────────┐ ┌──────────────┐
       │ Elasticsearch │ │ S3/MinIO │ │ Alert        │
       │ (Kibana UI)   │ │ Archive  │ │ Service      │
       │               │ │          │ │              │
       │ রিয়েল-টাইম   │ │ দীর্ঘমেয়াদী│ │ ERROR হলে   │
       │ সার্চ ও       │ │ সংরক্ষণ  │ │ Slack/SMS   │
       │ ভিজুয়ালাইজেশন│ │ (সস্তা)   │ │ পাঠাও      │
       └───────────────┘ └──────────┘ └──────────────┘
```

### 📊 Structured Log Format

```json
{
  "timestamp": "2024-11-15T10:30:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "instance": "payment-svc-pod-3",
  "trace_id": "abc123def456",
  "span_id": "span-789",
  "message": "bKash API timeout after 5000ms",
  "context": {
    "transaction_id": "TXN-20241115-abc",
    "amount": 1500,
    "retry_count": 2
  },
  "error": {
    "type": "TimeoutException",
    "stack": "at PaymentGateway.process..."
  }
}
```

### 🔍 লগ ফিল্টারিং ও রাউটিং

```
application-logs (সব লগ)
    │
    ├── level = ERROR  ──────► error-logs topic ──► Alert Service
    │                                              (Slack/SMS পাঠায়)
    │
    ├── level = WARN   ──────► Elasticsearch only
    │                          (dashboard-এ দেখায়)
    │
    └── level = INFO/DEBUG ──► S3 Archive only
                               (পরে দরকার হলে দেখা যায়)
```

### ✅ শেখার বিষয়

- **Structured JSON logs** → সহজে parse ও search করা যায়
- **trace_id** → একটি request-এর সব সার্ভিসের লগ trace করা যায়
- **Kafka buffer** → Elasticsearch slow হলেও লগ হারায় না
- **Multi-consumer** → একই লগ Elasticsearch + S3 + Alert সবাই পায়

---

## 📊 কেস স্টাডি ৫: রিয়েল-টাইম অ্যানালিটিক্স ড্যাশবোর্ড

### 📌 সমস্যার বিবরণ

Grameenphone-এর মতো টেলিকম কোম্পানিতে প্রতি সেকেন্ডে হাজার হাজার কল হয়, ডাটা ব্যবহার হয়।
রিয়েল-টাইম ড্যাশবোর্ডে দেখাতে হয় — কত কল হচ্ছে, কোন এলাকায় নেটওয়ার্ক সমস্যা,
কত GB ডাটা ব্যবহার হচ্ছে। CDR (Call Detail Record) থেকে রিয়েল-টাইম অ্যানালিটিক্স বানাতে হবে।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌────────────────────────────────────────────────────────────────────┐
│                    Telecom Network                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Cell     │  │ Cell     │  │ Cell     │  │ Cell     │          │
│  │ Tower 1  │  │ Tower 2  │  │ Tower 3  │  │ Tower N  │          │
│  │ (ঢাকা)  │  │ (চট্টগ্রাম)│ │ (সিলেট) │  │ (রাজশাহী)│          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │             │             │                  │
│       └─────────────┼─────────────┼─────────────┘                  │
│                     ▼             ▼                                 │
│              CDR Generator (Call Detail Records)                    │
└────────────────────┬───────────────────────────────────────────────┘
                     ▼
           ┌─────────────────────┐
           │   Kafka Cluster     │
           │                     │
           │ [cdr-events]        │  প্রতি সেকেন্ডে ৫০,০০০+ ইভেন্ট
           │ [data-usage]        │
           │ [network-alerts]    │
           └──────────┬──────────┘
                      │
          ┌───────────┼───────────────┐
          ▼           ▼               ▼
   ┌────────────┐ ┌──────────┐ ┌───────────────┐
   │ Kafka      │ │ ClickHouse│ │ Alert Engine  │
   │ Streams    │ │ (storage) │ │               │
   │ (aggregate)│ │           │ │ নেটওয়ার্ক    │
   │            │ │ দ্রুত     │ │ সমস্যা হলে   │
   │ প্রতি ঘণ্টায়│ │ analytics │ │ NOC টিমকে    │
   │ কল সংখ্যা, │ │ query     │ │ এলার্ট      │
   │ ডাটা ব্যবহার│ │           │ │              │
   └────────────┘ └──────────┘ └───────────────┘
          │           │
          ▼           ▼
   ┌──────────────────────────┐
   │    Grafana Dashboard     │
   │                          │
   │  📊 কল/ঘণ্টা: 1.2M      │
   │  📶 ডাটা: 450 TB/দিন    │
   │  ⚠️ ডাউন টাওয়ার: 3      │
   │  🗺️ এলাকাভিত্তিক হিটম্যাপ│
   └──────────────────────────┘
```

### 📊 Windowed Aggregation উদাহরণ

```
CDR Raw Events:
  10:00:01 → Call from 017xxxxxxxx, duration 120s, tower: DHK-101
  10:00:01 → Data usage 017yyyyyyyy, 50MB, tower: CTG-045
  10:00:02 → Call from 019xxxxxxxx, duration 45s, tower: DHK-203
  ...

Kafka Streams Aggregation (৫ মিনিট window):

  Window [10:00 - 10:05]:
    ├── Total calls: 15,234
    ├── Total data: 2.3 TB
    ├── Avg call duration: 87s
    ├── Top tower: DHK-101 (3,456 calls)
    └── Failed calls: 23 (0.15%)

  Window [10:05 - 10:10]:
    ├── Total calls: 14,891
    ├── Total data: 2.1 TB
    ...
```

### ✅ শেখার বিষয়

- **Ultra-high throughput** → সেকেন্ডে ৫০K+ ইভেন্ট handle
- **ClickHouse** → টাইম-সিরিজ ডাটায় অত্যন্ত দ্রুত
- **Windowed aggregation** → Kafka Streams দিয়ে রিয়েল-টাইম aggregate
- **Geo-based monitoring** → এলাকা ভিত্তিক নেটওয়ার্ক স্বাস্থ্য পর্যবেক্ষণ

---

## 🌡️ কেস স্টাডি ৬: IoT ডাটা পাইপলাইন

### 📌 সমস্যার বিবরণ

বাংলাদেশে বন্যা পূর্বাভাস ও আবহাওয়া পর্যবেক্ষণের জন্য সারা দেশে শত শত সেন্সর স্টেশন বসানো
আছে। এই সেন্সরগুলো প্রতি মিনিটে তাপমাত্রা, আর্দ্রতা, বৃষ্টিপাত, নদীর পানির উচ্চতা ইত্যাদি
পাঠায়। এই ডাটা রিয়েল-টাইমে প্রসেস করে এলার্ট দিতে হবে।

### 🏗️ আর্কিটেকচার ডায়াগ্রাম

```
┌──────────────────────────────────────────────────────────────┐
│                 সেন্সর নেটওয়ার্ক (সারা বাংলাদেশ)               │
│                                                              │
│  🌡️ তাপমাত্রা   💧 বৃষ্টিপাত    🌊 নদী পানি    💨 বায়ু     │
│  সেন্সর (200+)  সেন্সর (150+)  সেন্সর (100+)  সেন্সর (50+) │
│                                                              │
└────────────────────────┬─────────────────────────────────────┘
                         │  MQTT / HTTP
                         ▼
                ┌──────────────────┐
                │  IoT Gateway     │
                │  (Protocol       │
                │   Converter)     │
                └────────┬─────────┘
                         ▼
              ┌────────────────────┐
              │   Kafka Cluster    │
              │                    │
              │ [sensor-raw]       │  ৫০০+ sensors × ১/min
              │ [sensor-processed] │  = ৫০০+ msg/min
              │ [weather-alerts]   │
              └─────────┬──────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
    ┌────────────┐ ┌──────────┐ ┌───────────┐
    │ Processing │ │ TimeSeries│ │ Alert     │
    │ Engine     │ │ DB       │ │ Engine    │
    │            │ │ (InfluxDB)│ │           │
    │ ক্যালিব্রেশন│ │          │ │ বন্যা    │
    │ ও ভ্যালিডেশন│ │ ঐতিহাসিক│ │ সতর্কতা  │
    │            │ │ ডাটা সংরক্ষণ│ │ পাঠানো  │
    └────────────┘ └──────────┘ └───────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ 📱 SMS Alert  │
                              │ "⚠️ যমুনা নদীর │
                              │  পানি বিপদসীমার│
                              │  উপরে!"       │
                              └───────────────┘
```

### 📊 সেন্সর ডাটা ফরম্যাট

```json
{
  "sensor_id": "WS-DHAKA-MIRPUR-007",
  "station_name": "মিরপুর আবহাওয়া স্টেশন",
  "timestamp": "2024-07-15T14:30:00Z",
  "readings": {
    "temperature_c": 34.5,
    "humidity_pct": 85,
    "rainfall_mm": 12.3,
    "wind_speed_kmh": 25,
    "river_level_m": 8.7
  },
  "battery_pct": 78,
  "signal_strength": -65
}
```

### 🚨 Alert Rules

```
Rule 1: নদীর পানি বিপদসীমার উপরে
  IF river_level_m > DANGER_LEVEL[station] → CRITICAL ALERT

Rule 2: ভারী বৃষ্টি
  IF rainfall_mm > 50 (per hour) → WARNING

Rule 3: সেন্সর ব্যাটারি কম
  IF battery_pct < 15 → MAINTENANCE ALERT

Rule 4: সেন্সর অফলাইন
  IF no data for > 10 minutes → SENSOR DOWN ALERT
```

### ✅ শেখার বিষয়

- **IoT → MQTT → Kafka** → প্রোটোকল ব্রিজিং গুরুত্বপূর্ণ
- **Time-series DB** (InfluxDB/TimescaleDB) → সেন্সর ডাটার জন্য আদর্শ
- **Edge processing** → সব ডাটা ক্লাউডে পাঠানোর আগে edge-এ ফিল্টার করা
- **Alert thresholds** → প্রাকৃতিক দুর্যোগ সতর্কতায় Kafka-র reliability অত্যন্ত গুরুত্বপূর্ণ

---

## 🔗 কেস স্টাডি ৭: মাইক্রোসার্ভিস কমিউনিকেশন ব্যাকবোন

### 📌 সমস্যার বিবরণ

একটি বড় ই-কমার্স প্ল্যাটফর্মে ৫-১০টি মাইক্রোসার্ভিস থাকে। Synchronous HTTP call দিয়ে সব
সার্ভিস কানেক্ট করলে সমস্যা হয় — একটি সার্ভিস down হলে পুরো chain ভেঙে পড়ে।
Kafka দিয়ে asynchronous event-driven architecture বানালে সার্ভিসগুলো loosely coupled
হয় এবং কোনো একটি সার্ভিস fail করলে বাকিরা চলতে থাকে।

### 🏗️ Synchronous vs Asynchronous তুলনা

```
❌ Synchronous (HTTP call chain):

User → API Gateway → User Svc ──HTTP──► Order Svc ──HTTP──► Payment Svc
                                              │                    │
                                              │ HTTP               │ HTTP
                                              ▼                    ▼
                                        Inventory Svc        Notification Svc

সমস্যা: Payment Svc down হলে Order Svc-ও ব্লক হয়ে যায়!


✅ Asynchronous (Kafka event-driven):

┌──────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
│  User    │ │  Product  │ │  Order    │ │  Payment  │ │ Shipping  │
│  Service │ │  Service  │ │  Service  │ │  Service  │ │  Service  │
└────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
     │             │             │             │             │
     │  publish    │  publish    │  publish    │  publish    │  publish
     │  consume    │  consume    │  consume    │  consume    │  consume
     │             │             │             │             │
═════╧═════════════╧═════════════╧═════════════╧═════════════╧═════════
                         KAFKA CLUSTER
  Topics: user-events, product-events, order-events, payment-events,
          shipping-events, notification-events
═══════════════════════════════════════════════════════════════════════

সুবিধা: Payment Svc down হলে message Kafka তে জমা থাকে,
        Payment Svc ফিরে এলে সব process করে ফেলবে!
```

### 📊 Event Choreography Flow

```
১. Customer অর্ডার করে
   User Service → publish → [order-events] → "ORDER_CREATED"
                                │
   ┌────────────────────────────┤
   ▼                            ▼
   Order Service               Analytics Service
   (অর্ডার সেভ করে)             (ড্যাশবোর্ড আপডেট)
   │
   ▼ publish → [payment-events] → "PAYMENT_REQUESTED"
                     │
                     ▼
              Payment Service
              (পেমেন্ট প্রসেস)
              │
              ▼ publish → [payment-events] → "PAYMENT_COMPLETED"
                               │
              ┌────────────────┤
              ▼                ▼
        Order Service     Notification Service
        (status update)   (SMS: "পেমেন্ট সফল!")
        │
        ▼ publish → [shipping-events] → "SHIPMENT_CREATED"
                          │
                          ▼
                   Shipping Service
                   (ডেলিভারি ব্যবস্থা)
```

### 🛡️ Schema Evolution Handling

```
Version 1 (শুরুতে):
{
  "user_id": "U001",
  "name": "রহিম"
}

Version 2 (নতুন ফিল্ড যোগ — backward compatible):
{
  "user_id": "U001",
  "name": "রহিম",
  "phone": "01712345678"     ← নতুন, optional
}

Version 3 (ফিল্ড rename — breaking change ⚠️):
{
  "user_id": "U001",
  "full_name": "রহিম উদ্দিন",  ← "name" থেকে rename (❌ এটা করবেন না!)
  "phone": "01712345678"
}

✅ নিয়ম:
  - নতুন ফিল্ড যোগ করা যায় (default value সহ)
  - পুরনো ফিল্ড মুছা যাবে না
  - ফিল্ড rename করা যাবে না
  - Schema Registry ব্যবহার করুন (Avro/Protobuf)
```

### ❌ Error Handling Across Services

```
Dead Letter Queue (DLQ) প্যাটার্ন:

  [order-events] ──► Payment Service
                         │
                    ┌────┴─────┐
                    │ Process  │
                    └────┬─────┘
                         │
                   ┌─────┴──────┐
                   ▼            ▼
                 ✅ সফল       ❌ ব্যর্থ (৩ বার retry-র পর)
                   │            │
                   ▼            ▼
            [payment-events]  [payment-events-dlq]
            (সফল ফলাফল)       (Dead Letter Queue)
                               │
                               ▼
                        Manual Review / Alert
                        (ডেভেলপার দেখে ঠিক করবে)
```

### ✅ শেখার বিষয়

- **Loose coupling** → সার্ভিসগুলো একে অপরকে সরাসরি চেনে না
- **Resilience** → একটি সার্ভিস down হলে বাকি সিস্টেম চলতে থাকে
- **Schema evolution** → backward compatibility মেনে চলুন
- **DLQ pattern** → ব্যর্থ message হারানোর বদলে আলাদা queue-তে রাখুন

---

## 📋 সকল কেস স্টাডির তুলনা টেবিল

```
┌─────────────────────┬────────────────┬─────────────┬──────────────┬───────────────────────────┐
│ কেস স্টাডি           │ Throughput      │ Latency     │ Semantics    │ Key Patterns              │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ১. ই-কমার্স অর্ডার   │ 1K-100K/min    │ সেকেন্ড     │ At-least-    │ Event sourcing,           │
│    (Daraz-like)      │ (flash sale)   │ acceptable  │ once         │ Consumer groups           │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ২. রাইড ট্র্যাকিং    │ 50K-500K/min   │ < 1 sec     │ At-most-     │ Geo-partitioning,         │
│    (Pathao-like)     │ (GPS updates)  │ (real-time) │ once         │ Redis + Kafka             │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ৩. পেমেন্ট প্রসেসিং  │ 10K-100K/min   │ < 2 sec     │ Exactly-     │ Saga, Transactions,       │
│    (bKash-like)      │ (transactions) │             │ once         │ Audit trail               │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ৪. লগ অ্যাগ্রিগেশন   │ 100K+/min      │ মিনিট       │ At-least-    │ Multi-consumer,           │
│                     │ (log lines)    │ acceptable  │ once         │ ELK stack                 │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ৫. টেলিকম অ্যানালিটিক্স│ 500K+/min     │ সেকেন্ড     │ At-least-    │ Windowed aggregation,     │
│    (GP-like)         │ (CDR events)   │             │ once         │ ClickHouse                │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ৬. IoT পাইপলাইন     │ 1K-10K/min     │ মিনিট       │ At-least-    │ MQTT bridge,              │
│                     │ (sensor data)  │             │ once         │ Time-series DB            │
├─────────────────────┼────────────────┼─────────────┼──────────────┼───────────────────────────┤
│ ৭. মাইক্রোসার্ভিস    │ পরিবর্তনশীল    │ পরিবর্তনশীল │ Configurable │ Event choreography,       │
│    ব্যাকবোন          │                │             │              │ DLQ, Schema evolution     │
└─────────────────────┴────────────────┴─────────────┴──────────────┴───────────────────────────┘
```

---

## 🎯 সাধারণ শিক্ষা ও বেস্ট প্র্যাকটিস

সকল কেস স্টাডি থেকে যে সাধারণ শিক্ষাগুলো পাওয়া যায়:

### ১. সহজ থেকে শুরু করুন, প্রয়োজনে স্কেল করুন

```
শুরু করুন:
  - ১ broker, ৩ partition, ১ consumer instance
  - replication factor 1
  - default settings

প্রয়োজনে বাড়ান:
  - ৩ broker, ১২+ partition, multiple consumers
  - replication factor 3
  - tuned settings based on monitoring data
```

### ২. প্রথম দিন থেকে মনিটরিং করুন

```
অবশ্যই মনিটর করুন:
  ✅ Consumer lag (সবচেয়ে গুরুত্বপূর্ণ!)
  ✅ Broker disk usage
  ✅ Under-replicated partitions
  ✅ Request latency (produce & consume)
  ✅ Consumer group status

Tools:
  - Prometheus + Grafana (ফ্রি, সবচেয়ে জনপ্রিয়)
  - Kafka Manager / AKHQ (Kafka-specific UI)
  - Burrow (consumer lag monitoring)
```

### ৩. সঠিক Partition Key বাছাই করুন

```
ভালো Partition Key:
  ✅ order_id    → একই অর্ডারের ইভেন্ট ordered
  ✅ user_id     → একই user-এর ইভেন্ট একসাথে
  ✅ zone_id     → geographical locality
  ✅ sensor_id   → একই সেন্সরের ডাটা ordered

খারাপ Partition Key:
  ❌ timestamp   → সব message একই partition-এ যায়
  ❌ null        → round-robin হয়, ordering নেই
  ❌ country_code → বাংলাদেশের সব data একটাই partition-এ!
```

### ৪. ব্যর্থতার জন্য পরিকল্পনা করুন

```
যা ভুল হতে পারে:
  - Broker crash → replication factor 3 রাখুন
  - Consumer crash → consumer group auto-rebalance করবে
  - Network issue → producer retries + idempotence
  - Disk full → retention policy ও monitoring
  - Poison message → Dead Letter Queue (DLQ)
  - Schema change → Schema Registry + backward compatibility
```

### ৫. Schema Evolution কৌশল

```
✅ করুন:
  - নতুন optional field যোগ করুন (default value সহ)
  - Schema Registry ব্যবহার করুন
  - Avro বা Protobuf ব্যবহার করুন (JSON নয়)
  - Consumer-কে unknown field ignore করতে শেখান

❌ করবেন না:
  - Field delete করবেন না
  - Field rename করবেন না
  - Field type change করবেন না
  - Breaking change একসাথে deploy করবেন না
```

### ৬. প্রোডাকশন-like ডাটা দিয়ে টেস্ট করুন

```
টেস্ট পরিকল্পনা:
  ১. Unit test: serialization/deserialization ঠিক আছে কি?
  ২. Integration test: embedded Kafka দিয়ে end-to-end flow
  ৩. Load test: প্রোডাকশনের ২-৩ গুণ load দিয়ে টেস্ট
  ৪. Chaos test: broker kill করে দেখুন কী হয়
  ৫. Failover test: consumer restart করে দেখুন ডাটা হারায় কি না
```

---

## 📊 Kafka প্রোডাকশন রেডিনেস চেকলিস্ট

প্রোডাকশনে যাওয়ার আগে এই চেকলিস্ট সম্পূর্ণ করুন:

### ☐ ইনফ্রাস্ট্রাকচার

```
☐ কমপক্ষে ৩টি broker (বিভিন্ন সার্ভারে)
☐ ZooKeeper / KRaft cluster (৩ node)
☐ পর্যাপ্ত disk space (retention অনুযায়ী)
☐ Network bandwidth পর্যাপ্ত
☐ JVM heap size সঠিকভাবে সেট (6-8 GB)
☐ OS-level tuning (file descriptors, vm.swappiness)
```

### ☐ টপিক কনফিগারেশন

```
☐ Partition সংখ্যা সঠিক (throughput অনুযায়ী)
☐ Replication factor ≥ 3
☐ min.insync.replicas = 2
☐ Retention period সঠিক
☐ Cleanup policy (delete vs compact)
☐ Max message size সেট
```

### ☐ প্রডিউসার কনফিগারেশন

```
☐ acks = all (গুরুত্বপূর্ণ ডাটার জন্য)
☐ retries সেট
☐ enable.idempotence = true
☐ Proper serialization (Avro/Protobuf with Schema Registry)
☐ Error handling ও DLQ
☐ Compression enable (lz4 বা snappy)
```

### ☐ কনজিউমার কনফিগারেশন

```
☐ Consumer group ID meaningful
☐ auto.offset.reset সঠিক (earliest vs latest)
☐ enable.auto.commit = false (manual commit recommended)
☐ max.poll.records সঠিক
☐ session.timeout.ms ও heartbeat.interval.ms সেট
☐ Idempotent processing (duplicate handle)
```

### ☐ সিকিউরিটি

```
☐ SSL/TLS encryption enable
☐ SASL authentication configure
☐ ACL (Access Control List) সেট
☐ Network firewall rules
☐ Audit logging enable
☐ Secrets management (credentials কোডে নয়!)
```

### ☐ মনিটরিং ও এলার্টিং

```
☐ JMX metrics → Prometheus
☐ Consumer lag monitoring + alert
☐ Disk usage alert (80% threshold)
☐ Under-replicated partitions alert
☐ Broker down alert
☐ Grafana dashboard সেটআপ
☐ Log aggregation (broker logs)
☐ On-call rotation সেটআপ
```

### ☐ ডিজাস্টার রিকভারি

```
☐ Backup strategy (MirrorMaker 2 / Confluent Replicator)
☐ Recovery procedure documented
☐ RTO (Recovery Time Objective) নির্ধারিত
☐ RPO (Recovery Point Objective) নির্ধারিত
☐ DR drill practiced
☐ Runbook তৈরি
```

### ☐ ডকুমেন্টেশন

```
☐ Architecture diagram আপডেট
☐ Topic catalog (কোন topic কী কাজ করে)
☐ Schema documentation
☐ Runbook (common issues ও solutions)
☐ On-call guide
☐ Capacity planning document
```

---

## 🏁 উপসংহার

```
মনে রাখুন:

  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  Kafka শুধু একটি message queue না —              │
  │  এটি আপনার পুরো সিস্টেমের                       │
  │  nervous system (স্নায়ুতন্ত্র)।                   │
  │                                                  │
  │  সঠিকভাবে ডিজাইন করলে এটি আপনার                │
  │  সিস্টেমকে scalable, resilient এবং               │
  │  real-time বানিয়ে দেবে।                          │
  │                                                  │
  │  ভুলভাবে ডিজাইন করলে এটি আপনার                  │
  │  সবচেয়ে বড় single point of failure হবে।        │
  │                                                  │
  │  📚 শিখুন → 🧪 প্র্যাকটিস করুন → 🚀 প্রোডাকশনে দিন │
  │                                                  │
  └──────────────────────────────────────────────────┘
```

---

**📚 পরবর্তী পড়ুন:**
- Kafka অফিসিয়াল ডকুমেন্টেশন: https://kafka.apache.org/documentation/
- Confluent Developer: https://developer.confluent.io/
- Kafka Streams ডকুমেন্টেশন: https://kafka.apache.org/documentation/streams/

> **💡 টিপ:** এই কেস স্টাডিগুলো হুবহু কপি না করে, আপনার নিজের প্রজেক্টের requirement অনুযায়ী
> customize করুন। প্রতিটি সিস্টেমের চাহিদা আলাদা!
