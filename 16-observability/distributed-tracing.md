# 📌 ডিস্ট্রিবিউটেড ট্রেসিং (Distributed Tracing)

> **একটি রিকোয়েস্ট যখন অনেকগুলো সার্ভিসের মধ্য দিয়ে যায়, তখন প্রতিটি ধাপ ট্র্যাক করার পদ্ধতিকে ডিস্ট্রিবিউটেড ট্রেসিং বলে।**

---

## 📖 সূচিপত্র

1. [ডিস্ট্রিবিউটেড ট্রেসিং কী?](#-ডিস্ট্রিবিউটেড-ট্রেসিং-কী)
2. [Trace, Span এবং Context Propagation](#-trace-span-এবং-context-propagation)
3. [W3C Trace Context হেডার ফরম্যাট](#-w3c-trace-context-হেডার-ফরম্যাট)
4. [ট্রেসিং টুলস তুলনা: Jaeger vs Zipkin vs X-Ray](#-ট্রেসিং-টুলস-তুলনা)
5. [OpenTelemetry ট্রেসিং SDK](#-opentelemetry-ট্রেসিং-sdk)
6. [স্যাম্পলিং স্ট্র্যাটেজি](#-স্যাম্পলিং-স্ট্র্যাটেজি)
7. [কোড ইনস্ট্রুমেন্টেশন](#-কোড-ইনস্ট্রুমেন্টেশন)
8. [PHP ট্রেসিং উদাহরণ](#-php-ট্রেসিং-উদাহরণ)
9. [JavaScript/Node.js ট্রেসিং উদাহরণ](#-javascriptnodejs-ট্রেসিং-উদাহরণ)
10. [❌ Bad vs ✅ Good প্যাটার্ন](#-bad-vs--good-প্যাটার্ন)
11. [বাস্তব উদাহরণ: bKash ও Daraz](#-বাস্তব-উদাহরণ)
12. [Jaeger Docker সেটআপ](#-jaeger-docker-সেটআপ)
13. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📖 ডিস্ট্রিবিউটেড ট্রেসিং কী?

মনে করুন একজন ইউজার bKash অ্যাপ থেকে ৫০০ টাকা সেন্ড মানি করলেন। এই একটি রিকোয়েস্ট পর্দার
আড়ালে অনেকগুলো সার্ভিসের মধ্য দিয়ে যায়:

```
ইউজারের অ্যাপ → API Gateway → Auth Service → Transaction Service
                                                      ↓
                                              Balance Service
                                                      ↓
                                            Notification Service
                                                      ↓
                                               SMS Gateway
```

এখন যদি ইউজার বলেন "আমার টাকা কাটা গেছে কিন্তু প্রাপক পাননি" — আপনি কীভাবে খুঁজবেন
কোন সার্ভিসে সমস্যা হয়েছে? **ডিস্ট্রিবিউটেড ট্রেসিং** এই সমস্যার সমাধান।

**ডিস্ট্রিবিউটেড ট্রেসিং** প্রতিটি রিকোয়েস্টকে একটি **ইউনিক আইডি (Trace ID)** দেয় এবং
প্রতিটি সার্ভিসে সেই আইডি পাস করে। ফলে পরে আপনি পুরো রিকোয়েস্টের জীবনচক্র দেখতে পারেন।

```
┌─────────────────────────────────────────────────────────┐
│              Monolith (সিঙ্গেল অ্যাপ)                    │
│                                                         │
│  রিকোয়েস্ট → ফাংশন A → ফাংশন B → ফাংশন C → রেসপন্স   │
│                                                         │
│  Stack trace দেখলেই সব বোঝা যায় ✅                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│           Microservices (অনেক সার্ভিস)                   │
│                                                         │
│  রিকোয়েস্ট → সার্ভিস A ──HTTP──→ সার্ভিস B             │
│                                   ──gRPC──→ সার্ভিস C   │
│                                   ──Kafka──→ সার্ভিস D  │
│                                                         │
│  Stack trace যথেষ্ট নয়! ❌                              │
│  ডিস্ট্রিবিউটেড ট্রেসিং দরকার! ✅                       │
└─────────────────────────────────────────────────────────┘
```

---

## 🔗 Trace, Span এবং Context Propagation

### মূল ধারণাসমূহ

| টার্ম | বাংলা ব্যাখ্যা |
|-------|----------------|
| **Trace** | একটি সম্পূর্ণ রিকোয়েস্টের শুরু থেকে শেষ পর্যন্ত পুরো যাত্রা |
| **Span** | একটি Trace-এর মধ্যে একটি নির্দিষ্ট অপারেশন বা ধাপ |
| **Parent Span** | যে Span থেকে অন্য Span তৈরি হয়েছে |
| **Child Span** | Parent Span-এর অধীনে তৈরি হওয়া Span |
| **Trace ID** | পুরো Trace-এর জন্য একটি ইউনিক আইডি (সব সার্ভিসে একই) |
| **Span ID** | প্রতিটি Span-এর নিজস্ব ইউনিক আইডি |
| **Context Propagation** | এক সার্ভিস থেকে অন্য সার্ভিসে Trace Context পাঠানো |

### Trace/Span হায়ারার্কি ডায়াগ্রাম

```
Trace ID: abc123def456
═══════════════════════════════════════════════════════════════

[Root Span] API Gateway: POST /send-money         (Span ID: s1)
│   start: 0ms                                     end: 450ms
│
├──[Child Span] Auth Service: validateToken        (Span ID: s2)
│      start: 5ms                end: 25ms          parent: s1
│
├──[Child Span] Transaction Service: initTransfer  (Span ID: s3)
│   │  start: 30ms                               end: 400ms
│   │                                              parent: s1
│   │
│   ├──[Child Span] Balance Service: debit         (Span ID: s4)
│   │      start: 35ms          end: 120ms          parent: s3
│   │      └─ Event: "balance_checked" at 50ms
│   │      └─ Event: "amount_debited" at 110ms
│   │
│   ├──[Child Span] Balance Service: credit        (Span ID: s5)
│   │      start: 125ms         end: 250ms          parent: s3
│   │      └─ Event: "recipient_credited" at 240ms
│   │
│   └──[Child Span] Notification Service: sendSMS  (Span ID: s6)
│          start: 255ms         end: 390ms          parent: s3
│          └─ Event: "sms_queued" at 260ms
│          └─ Event: "sms_sent" at 385ms
│
└── Attributes:
        user.id = "01XXXXXXXXX"
        transaction.amount = 500
        transaction.currency = "BDT"
        transaction.type = "send_money"
```

### Waterfall ডায়াগ্রাম (Timeline ভিউ)

```
সময় (ms)    0    50   100   150   200   250   300   350   400   450
             │    │     │     │     │     │     │     │     │     │
API Gateway  ╠════════════════════════════════════════════════════╣
             │    │     │     │     │     │     │     │     │     │
Auth Svc     │╠══╣     │     │     │     │     │     │     │     │
             │    │     │     │     │     │     │     │     │     │
Txn Service  │    ╠════════════════════════════════════════════╣  │
             │    │     │     │     │     │     │     │     │     │
Balance:debit│    ╠═══════════╣     │     │     │     │     │     │
             │    │     │     │     │     │     │     │     │     │
Balance:credit    │     │     ╠═══════════╣     │     │     │     │
             │    │     │     │     │     │     │     │     │     │
Notification │    │     │     │     │     ╠══════════════════╣    │
             │    │     │     │     │     │     │     │     │     │

 ← 5ms →                                              ← 450ms total →
```

### Context Propagation কীভাবে কাজ করে

```
┌──────────────┐  HTTP Headers  ┌──────────────┐  HTTP Headers  ┌──────────────┐
│              │───────────────→│              │───────────────→│              │
│  Service A   │  traceparent:  │  Service B   │  traceparent:  │  Service C   │
│              │  00-abc123-s1  │              │  00-abc123-s3  │              │
│              │←───────────────│              │←───────────────│              │
└──────────────┘                └──────────────┘                └──────────────┘

    Trace ID: abc123               Trace ID: abc123           Trace ID: abc123
    Span ID:  s1 (নতুন)            Span ID:  s3 (নতুন)        Span ID:  s5 (নতুন)
    Parent:   none                 Parent:   s1               Parent:   s3

    প্রতিটি সার্ভিস একই Trace ID রাখে কিন্তু নিজের নতুন Span ID তৈরি করে।
    আগের সার্ভিসের Span ID হয়ে যায় নতুন Span-এর Parent ID।
```

---

## 📊 W3C Trace Context হেডার ফরম্যাট

W3C Trace Context হলো ইন্ডাস্ট্রি স্ট্যান্ডার্ড যা বিভিন্ন ট্রেসিং সিস্টেমের মধ্যে
context propagation নিশ্চিত করে।

### `traceparent` হেডার ফরম্যাট ব্রেকডাউন

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ││ │                                │ │                │ ││
             ││ │         Trace ID               │ │   Span ID     │ ││
             ││ │    (32 hex characters)         │ │(16 hex chars) │ ││
             ││ └────────────────────────────────┘ └───────────────┘ ││
             ││                                                      ││
             │└── version (সবসময় "00")              trace-flags ─────┘│
             │                                     (01 = sampled)     │
             └── ফরম্যাট শুরু                       ফরম্যাট শেষ ──────┘

ভাঙা ভাঙা দেখি:
┌─────────┬──────────────────────────────────┬──────────────────┬─────────────┐
│ Version │           Trace ID               │     Span ID      │ Trace Flags │
│  (2)    │           (32)                   │      (16)         │    (2)      │
├─────────┼──────────────────────────────────┼──────────────────┼─────────────┤
│   00    │ 4bf92f3577b34da6a3ce929d0e0e4736 │ 00f067aa0ba902b7 │     01      │
├─────────┼──────────────────────────────────┼──────────────────┼─────────────┤
│ always  │ রিকোয়েস্টের ইউনিক আইডি          │ বর্তমান Span ID  │ 01=sampled  │
│  "00"   │ সব সার্ভিসে একই থাকে              │ প্রতি সার্ভিসে   │ 00=not      │
│         │                                  │ পরিবর্তন হয়      │  sampled    │
└─────────┴──────────────────────────────────┴──────────────────┴─────────────┘
```

### `tracestate` হেডার

```
tracestate: vendor1=opaque_value,vendor2=another_value

উদাহরণ:
tracestate: bkash=transaction_id:TXN001,region=dhaka
```

`tracestate` ভেন্ডর-নির্দিষ্ট অতিরিক্ত তথ্য বহন করে। `traceparent` সবসময় থাকবে,
`tracestate` ঐচ্ছিক।

---

## 🏠 ট্রেসিং টুলস তুলনা

### Jaeger আর্কিটেকচার ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Jaeger Architecture                          │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐   ┌───────────┐  │
│  │ Your App │    │ Your App │    │   Your App   │   │ Your App  │  │
│  │(Service) │    │(Service) │    │  (Service)   │   │ (Service) │  │
│  └────┬─────┘    └────┬─────┘    └──────┬───────┘   └─────┬─────┘  │
│       │               │                │                  │        │
│       ▼               ▼                ▼                  ▼        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Jaeger Agent (UDP)                        │   │
│  │         প্রতিটি হোস্টে একটি Agent চলে (sidecar/DaemonSet)    │   │
│  │         Spans ব্যাচ করে Collector-এ পাঠায়                    │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │ gRPC/HTTP                              │
│                           ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Jaeger Collector                           │   │
│  │         Spans validate, transform ও store করে               │   │
│  │         Kafka-তেও পাঠাতে পারে (async pipeline)              │   │
│  └──────────┬─────────────────────────────────┬────────────────┘   │
│             │                                 │                    │
│             ▼                                 ▼                    │
│  ┌──────────────────┐              ┌─────────────────────┐        │
│  │     Storage       │              │   Kafka (Optional)  │        │
│  │  ┌─────────────┐ │              │   Ingestion buffer  │        │
│  │  │Elasticsearch│ │              └──────────┬──────────┘        │
│  │  │  or Cassandra│ │                        │                    │
│  │  │  or Badger   │ │                        ▼                    │
│  │  └─────────────┘ │              ┌─────────────────────┐        │
│  └────────┬─────────┘              │  Jaeger Ingester    │        │
│           │                        │  Kafka → Storage    │        │
│           │                        └─────────────────────┘        │
│           ▼                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Jaeger Query Service                       │   │
│  │         Storage থেকে traces পড়ে API-তে expose করে          │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                        │
│                           ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Jaeger UI                               │   │
│  │         ব্রাউজারে trace সার্চ, waterfall ভিউ দেখায়           │   │
│  │         http://localhost:16686                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Jaeger vs Zipkin vs AWS X-Ray তুলনা

```
┌──────────────────────┬──────────────┬──────────────┬────────────────┐
│       বৈশিষ্ট্য       │    Jaeger     │    Zipkin     │   AWS X-Ray    │
├──────────────────────┼──────────────┼──────────────┼────────────────┤
│ তৈরি করেছে           │ Uber → CNCF  │ Twitter      │ Amazon         │
│ ওপেন সোর্স           │ ✅ হ্যাঁ       │ ✅ হ্যাঁ       │ ❌ না           │
│ CNCF গ্র্যাজুয়েটেড   │ ✅ হ্যাঁ       │ ❌ না         │ ❌ না           │
│ OpenTelemetry সাপোর্ট │ ✅ নেটিভ     │ ✅ এক্সপোর্টার│ ✅ এক্সপোর্টার  │
│ স্টোরেজ              │ ES/Cassandra │ ES/MySQL/    │ AWS managed    │
│                      │ /Badger      │ Cassandra    │                │
│ স্যাম্পলিং           │ Adaptive     │ Rate-based   │ Fixed/Rules    │
│ সার্ভিস ম্যাপ         │ ✅ হ্যাঁ       │ ✅ হ্যাঁ       │ ✅ হ্যাঁ         │
│ ডিপেন্ডেন্সি গ্রাফ    │ ✅ হ্যাঁ       │ ✅ হ্যাঁ       │ ✅ হ্যাঁ         │
│ লেটেন্সি হিস্টোগ্রাম  │ ✅ হ্যাঁ       │ ❌ না         │ ✅ হ্যাঁ         │
│ কুবারনেটিস           │ ✅ Operator   │ Manual       │ EKS integrated │
│ UI কোয়ালিটি          │ ⭐⭐⭐⭐       │ ⭐⭐⭐        │ ⭐⭐⭐⭐        │
│ AWS ইন্টিগ্রেশন      │ Manual       │ Manual       │ ✅ নেটিভ       │
│ খরচ                  │ ফ্রি          │ ফ্রি          │ পে-পার-ট্রেস   │
│ বাংলাদেশে সুপারিশ     │ ⭐⭐⭐⭐⭐    │ ⭐⭐⭐        │ ⭐⭐⭐⭐        │
│                      │ (সেলফ-হোস্ট) │ (সিম্পল)     │ (AWS ব্যবহারে) │
└──────────────────────┴──────────────┴──────────────┴────────────────┘
```

**বাংলাদেশের কোম্পানির জন্য সুপারিশ:**
- **bKash/Nagad** (ফিনটেক): Jaeger — ওপেন সোর্স, বিস্তৃত ফিচার, CNCF ব্যাকড
- **Daraz** (ই-কমার্স): Jaeger বা X-Ray — ট্র্যাফিক বেশি, adaptive sampling দরকার
- **Pathao** (রাইড-শেয়ারিং): Jaeger — রিয়েল-টাইম ট্রেসিং, কুবারনেটিস সাপোর্ট
- **Grameenphone** (টেলকো): X-Ray — যদি AWS-এ থাকে, নাহলে Jaeger

---

## 🎯 OpenTelemetry ট্রেসিং SDK

OpenTelemetry (OTel) হলো observability-র জন্য ইন্ডাস্ট্রি স্ট্যান্ডার্ড। এটি vendor-neutral,
অর্থাৎ আপনি Jaeger, Zipkin, বা X-Ray — যেকোনোটায় ডেটা পাঠাতে পারেন।

```
┌──────────────────────────────────────────────────────────────┐
│                    OpenTelemetry Architecture                  │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │   Traces   │  │   Metrics  │  │    Logs    │  ← Signals  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │
│        │               │               │                     │
│        ▼               ▼               ▼                     │
│  ┌──────────────────────────────────────────────┐            │
│  │              OTel SDK (আপনার অ্যাপে)          │            │
│  │  TracerProvider + SpanProcessor + Exporter    │            │
│  └────────────────────┬─────────────────────────┘            │
│                       │  OTLP (OpenTelemetry Protocol)       │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────┐            │
│  │           OTel Collector (ঐচ্ছিক)             │            │
│  │    Receivers → Processors → Exporters         │            │
│  └────────┬──────────┬──────────┬───────────────┘            │
│           │          │          │                             │
│           ▼          ▼          ▼                             │
│       ┌───────┐ ┌────────┐ ┌─────────┐                      │
│       │Jaeger │ │Zipkin  │ │ X-Ray   │  ← Backends          │
│       └───────┘ └────────┘ └─────────┘                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 স্যাম্পলিং স্ট্র্যাটেজি

প্রোডাকশনে প্রতিটি রিকোয়েস্ট ট্রেস করা ব্যয়বহুল। স্যাম্পলিং দিয়ে আমরা ঠিক করি
কোন ট্রেস সংরক্ষণ করব।

### স্যাম্পলিং স্ট্র্যাটেজি তুলনা

```
┌────────────────────┬─────────────────────┬────────────┬──────────────────────┐
│    স্ট্র্যাটেজি      │      কীভাবে কাজ করে  │   খরচ      │    কখন ব্যবহার করবেন  │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Always On          │ সব ট্রেস রাখে        │ 💰💰💰💰💰  │ ডেভ/স্টেজিং           │
│                    │ কিছু বাদ দেয় না      │ সবচেয়ে বেশি│ environment          │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Always Off         │ কোনো ট্রেস রাখে না   │ 💰         │ ট্রেসিং বন্ধ করতে     │
│                    │                     │ শূন্য       │ চাইলে                 │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Probabilistic      │ X% ট্রেস রাখে       │ 💰💰       │ প্রোডাকশন —           │
│ (সম্ভাব্যতা ভিত্তিক)│ যেমন: ১০% ট্রেস    │ মাঝারি     │ সাধারণ ক্ষেত্রে        │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Rate Limiting      │ প্রতি সেকেন্ডে N টি  │ 💰💰       │ প্রোডাকশন —           │
│ (রেট সীমিত)        │ ট্রেস রাখে          │ নিয়ন্ত্রিত  │ ট্র্যাফিক পিকে        │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Tail-Based         │ পুরো ট্রেস শেষ হওয়ার │ 💰💰💰     │ শুধু error/slow       │
│ (টেইল-বেসড)        │ পর সিদ্ধান্ত নেয়     │ বেশি       │ ট্রেস রাখতে চাইলে     │
│                    │ error হলে রাখে      │ মেমোরি     │ bKash/Nagad-এ আদর্শ  │
├────────────────────┼─────────────────────┼────────────┼──────────────────────┤
│ Parent-Based       │ প্যারেন্ট Span-এর    │ 💰💰       │ ডাউনস্ট্রিম সার্ভিস    │
│                    │ সিদ্ধান্ত অনুসরণ করে  │ মাঝারি     │ গুলোতে consistency    │
└────────────────────┴─────────────────────┴────────────┴──────────────────────┘
```

**bKash/Nagad-এর জন্য সুপারিশ:**
- প্রোডাকশনে **Tail-Based Sampling** — শুধু error ও slow ট্রেস রাখুন
- ডেভেলপমেন্টে **Always On** — সব দেখুন
- স্টেজিংয়ে **Probabilistic 50%** — অর্ধেক রাখুন

---

## 🔗 কোড ইনস্ট্রুমেন্টেশন

দুই ধরনের ইনস্ট্রুমেন্টেশন আছে:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. Auto Instrumentation (স্বয়ংক্রিয়)                      │
│     - SDK/Agent নিজেই HTTP, DB, gRPC ইত্যাদি ট্রেস করে     │
│     - কোড পরিবর্তন কম                                      │
│     - দ্রুত শুরু করা যায়                                     │
│                                                             │
│  2. Manual Instrumentation (ম্যানুয়াল)                      │
│     - আপনি নিজে Span তৈরি করেন                              │
│     - বিজনেস লজিক ট্রেস করতে দরকার                         │
│     - বেশি কন্ট্রোল পাওয়া যায়                               │
│                                                             │
│  সেরা পদ্ধতি: দুটোই ব্যবহার করুন! 🎯                        │
│  Auto → HTTP/DB spans                                       │
│  Manual → বিজনেস-নির্দিষ্ট spans                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📌 PHP ট্রেসিং উদাহরণ

### PHP OpenTelemetry SDK সেটআপ

```bash
# Composer দিয়ে OpenTelemetry PHP SDK ইনস্টল করুন
composer require open-telemetry/sdk
composer require open-telemetry/exporter-otlp
composer require open-telemetry/transport-grpc
```

### সম্পূর্ণ PHP ট্রেসিং উদাহরণ (bKash Send Money)

```php
<?php

use OpenTelemetry\API\Globals;
use OpenTelemetry\API\Trace\SpanKind;
use OpenTelemetry\API\Trace\StatusCode;
use OpenTelemetry\Context\Context;
use OpenTelemetry\SDK\Trace\TracerProviderBuilder;
use OpenTelemetry\SDK\Trace\SpanProcessor\SimpleSpanProcessor;
use OpenTelemetry\Contrib\Otlp\SpanExporter;
use OpenTelemetry\SDK\Resource\ResourceInfo;
use OpenTelemetry\SDK\Common\Attribute\Attributes;
use OpenTelemetry\SemConv\ResourceAttributes;

// ============================
// TracerProvider সেটআপ
// ============================
$resource = ResourceInfo::create(Attributes::create([
    ResourceAttributes::SERVICE_NAME => 'bkash-transaction-service',
    ResourceAttributes::SERVICE_VERSION => '2.1.0',
    ResourceAttributes::DEPLOYMENT_ENVIRONMENT => 'production',
]));

$exporter = new SpanExporter(
    'http://jaeger-collector:4318/v1/traces'
);

$tracerProvider = (new TracerProviderBuilder())
    ->setResource($resource)
    ->addSpanProcessor(new SimpleSpanProcessor($exporter))
    ->build();

$tracer = $tracerProvider->getTracer(
    'bkash.transaction',
    '1.0.0'
);

// ============================
// Send Money ট্রানজ্যাকশন ট্রেসিং
// ============================
function processSendMoney(
    string $senderId,
    string $receiverId,
    float $amount
) {
    global $tracer;

    // Root span তৈরি করুন
    $rootSpan = $tracer->spanBuilder('send-money')
        ->setSpanKind(SpanKind::KIND_SERVER)
        ->setAttribute('transaction.sender', $senderId)
        ->setAttribute('transaction.receiver', $receiverId)
        ->setAttribute('transaction.amount', $amount)
        ->setAttribute('transaction.currency', 'BDT')
        ->startSpan();

    $scope = $rootSpan->activate();

    try {
        // Step 1: ব্যালেন্স চেক
        $balanceSpan = $tracer->spanBuilder('check-balance')
            ->setSpanKind(SpanKind::KIND_INTERNAL)
            ->startSpan();

        $balance = checkBalance($senderId);
        $balanceSpan->setAttribute('user.balance', $balance);

        if ($balance < $amount) {
            $balanceSpan->setStatus(StatusCode::STATUS_ERROR, 'Insufficient balance');
            $balanceSpan->end();
            throw new \Exception('Insufficient balance');
        }

        $balanceSpan->addEvent('balance_verified', [
            'available_balance' => $balance,
            'required_amount' => $amount,
        ]);
        $balanceSpan->end();

        // Step 2: ডেবিট
        $debitSpan = $tracer->spanBuilder('debit-sender')
            ->setSpanKind(SpanKind::KIND_INTERNAL)
            ->startSpan();

        debitAccount($senderId, $amount);
        $debitSpan->addEvent('amount_debited', [
            'account' => $senderId,
            'amount' => $amount,
        ]);
        $debitSpan->end();

        // Step 3: ক্রেডিট
        $creditSpan = $tracer->spanBuilder('credit-receiver')
            ->setSpanKind(SpanKind::KIND_INTERNAL)
            ->startSpan();

        creditAccount($receiverId, $amount);
        $creditSpan->addEvent('amount_credited', [
            'account' => $receiverId,
            'amount' => $amount,
        ]);
        $creditSpan->end();

        // Step 4: নোটিফিকেশন পাঠান
        $notifySpan = $tracer->spanBuilder('send-notification')
            ->setSpanKind(SpanKind::KIND_CLIENT)
            ->startSpan();

        sendSmsNotification($senderId, $receiverId, $amount);
        $notifySpan->addEvent('sms_sent');
        $notifySpan->end();

        $rootSpan->setStatus(StatusCode::STATUS_OK);

    } catch (\Throwable $e) {
        $rootSpan->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
        $rootSpan->recordException($e);
        throw $e;
    } finally {
        $rootSpan->end();
        $scope->detach();
    }
}
```

### PHP Middleware: Trace Context Propagation

```php
<?php
// Laravel/Slim-এর জন্য Trace Context Propagation Middleware

use OpenTelemetry\API\Globals;
use OpenTelemetry\API\Trace\SpanKind;
use OpenTelemetry\API\Trace\Propagation\TraceContextPropagator;
use OpenTelemetry\Context\Context;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ResponseInterface;

class TraceContextMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {

        $propagator = TraceContextPropagator::getInstance();
        $tracer = Globals::tracerProvider()->getTracer('http-server');

        // ইনকামিং হেডার থেকে parent context extract করুন
        $parentContext = $propagator->extract(
            $request->getHeaders()
        );

        $routeName = $request->getMethod() . ' ' . $request->getUri()->getPath();

        // নতুন server span তৈরি করুন (parent context সহ)
        $span = $tracer->spanBuilder($routeName)
            ->setParent($parentContext)
            ->setSpanKind(SpanKind::KIND_SERVER)
            ->setAttribute('http.method', $request->getMethod())
            ->setAttribute('http.url', (string) $request->getUri())
            ->setAttribute('http.user_agent', $request->getHeaderLine('User-Agent'))
            ->startSpan();

        $scope = $span->activate();

        try {
            $response = $handler->handle($request);

            $span->setAttribute('http.status_code', $response->getStatusCode());

            if ($response->getStatusCode() >= 400) {
                $span->setStatus(
                    \OpenTelemetry\API\Trace\StatusCode::STATUS_ERROR,
                    'HTTP ' . $response->getStatusCode()
                );
            }

            return $response;

        } catch (\Throwable $e) {
            $span->recordException($e);
            $span->setStatus(
                \OpenTelemetry\API\Trace\StatusCode::STATUS_ERROR,
                $e->getMessage()
            );
            throw $e;
        } finally {
            $span->end();
            $scope->detach();
        }
    }
}
```

---

## 📌 JavaScript/Node.js ট্রেসিং উদাহরণ

### Node.js OpenTelemetry SDK সেটআপ

```bash
# প্রয়োজনীয় প্যাকেজ ইনস্টল করুন
npm install @opentelemetry/sdk-trace-node \
            @opentelemetry/sdk-trace-base \
            @opentelemetry/api \
            @opentelemetry/resources \
            @opentelemetry/semantic-conventions \
            @opentelemetry/exporter-trace-otlp-http \
            @opentelemetry/instrumentation-http \
            @opentelemetry/instrumentation-express
```

### সম্পূর্ণ Node.js Tracing সেটআপ (tracing.js)

```javascript
// tracing.js — অ্যাপ শুরু হওয়ার আগে এই ফাইল লোড করুন
// node -r ./tracing.js app.js

const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor, BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { Resource } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } = require('@opentelemetry/semantic-conventions');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { trace, SpanKind, SpanStatusCode } = require('@opentelemetry/api');

// ============================
// TracerProvider কনফিগারেশন
// ============================
const provider = new NodeTracerProvider({
  resource: new Resource({
    [ATTR_SERVICE_NAME]: 'daraz-order-service',
    [ATTR_SERVICE_VERSION]: '3.0.1',
    'deployment.environment': 'production',
    'service.region': 'bd-dhaka-1',
  }),
});

// Jaeger-এ traces পাঠানোর জন্য Exporter
const exporter = new OTLPTraceExporter({
  url: 'http://jaeger-collector:4318/v1/traces',
});

// প্রোডাকশনে BatchSpanProcessor ব্যবহার করুন (পারফরম্যান্সের জন্য)
provider.addSpanProcessor(new BatchSpanProcessor(exporter, {
  maxQueueSize: 2048,
  maxExportBatchSize: 512,
  scheduledDelayMillis: 5000,
}));

provider.register();

// Auto Instrumentation — HTTP ও Express স্বয়ংক্রিয়ভাবে ট্রেস হবে
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

console.log('✅ OpenTelemetry tracing initialized for daraz-order-service');
```

### Express.js অ্যাপে Manual Tracing (Daraz Order Flow)

```javascript
// app.js — Daraz অর্ডার সার্ভিস
const express = require('express');
const { trace, SpanKind, SpanStatusCode } = require('@opentelemetry/api');

const app = express();
app.use(express.json());

const tracer = trace.getTracer('daraz.order', '1.0.0');

// ============================
// Daraz অর্ডার প্লেস করার API
// ============================
app.post('/api/orders', async (req, res) => {
  const { userId, items, shippingAddress, paymentMethod } = req.body;

  // Manual span — অর্ডার প্রসেসিং
  const orderSpan = tracer.startSpan('process-order', {
    kind: SpanKind.SERVER,
    attributes: {
      'order.user_id': userId,
      'order.item_count': items.length,
      'order.payment_method': paymentMethod,
      'order.shipping_city': shippingAddress.city,
    },
  });

  const ctx = trace.setSpan(trace.getActiveSpan()
    ? trace.getActiveSpan().context
    : undefined, orderSpan);

  try {
    // Step 1: ইনভেন্টরি চেক
    const inventorySpan = tracer.startSpan('check-inventory', {
      kind: SpanKind.CLIENT,
    });
    const stockAvailable = await checkInventory(items);
    inventorySpan.setAttribute('inventory.all_available', stockAvailable);
    if (!stockAvailable) {
      inventorySpan.setStatus({ code: SpanStatusCode.ERROR, message: 'Out of stock' });
      inventorySpan.end();
      return res.status(400).json({ error: 'Some items are out of stock' });
    }
    inventorySpan.end();

    // Step 2: পেমেন্ট প্রসেস (bKash/Nagad)
    const paymentSpan = tracer.startSpan('process-payment', {
      kind: SpanKind.CLIENT,
      attributes: {
        'payment.method': paymentMethod,
        'payment.gateway': paymentMethod === 'bkash' ? 'bKash' : 'Nagad',
      },
    });
    const paymentResult = await processPayment(userId, items, paymentMethod);
    paymentSpan.addEvent('payment_completed', {
      'payment.transaction_id': paymentResult.txnId,
      'payment.amount': paymentResult.amount,
    });
    paymentSpan.end();

    // Step 3: অর্ডার তৈরি
    const createOrderSpan = tracer.startSpan('create-order-record', {
      kind: SpanKind.INTERNAL,
    });
    const order = await createOrder(userId, items, paymentResult);
    createOrderSpan.setAttribute('order.id', order.id);
    createOrderSpan.end();

    // Step 4: শিপিং সার্ভিসে পাঠান
    const shippingSpan = tracer.startSpan('initiate-shipping', {
      kind: SpanKind.CLIENT,
    });
    await initiateShipping(order, shippingAddress);
    shippingSpan.addEvent('shipping_initiated', {
      'shipping.provider': 'Pathao Courier',
      'shipping.city': shippingAddress.city,
    });
    shippingSpan.end();

    orderSpan.setStatus({ code: SpanStatusCode.OK });
    res.json({ orderId: order.id, status: 'confirmed' });

  } catch (err) {
    orderSpan.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
    orderSpan.recordException(err);
    res.status(500).json({ error: 'Order processing failed' });
  } finally {
    orderSpan.end();
  }
});

app.listen(3000, () => console.log('Daraz Order Service running on :3000'));
```

### Express.js Middleware: Trace Context Propagation

```javascript
// middleware/traceContext.js
const { trace, context, propagation, SpanKind, SpanStatusCode } = require('@opentelemetry/api');

const tracer = trace.getTracer('http-middleware');

function traceContextMiddleware(req, res, next) {
  // ইনকামিং হেডার থেকে parent context extract করুন
  const parentContext = propagation.extract(context.active(), req.headers);

  const span = tracer.startSpan(
    `${req.method} ${req.path}`,
    {
      kind: SpanKind.SERVER,
      attributes: {
        'http.method': req.method,
        'http.url': req.originalUrl,
        'http.user_agent': req.get('user-agent'),
        'http.client_ip': req.ip,
      },
    },
    parentContext
  );

  // আউটগোয়িং রিকোয়েস্টের জন্য context inject করার হেল্পার
  req.injectTraceContext = (headers = {}) => {
    propagation.inject(
      trace.setSpan(context.active(), span),
      headers
    );
    return headers;
  };

  // রেসপন্সে trace ID যোগ করুন (ডিবাগিংয়ের জন্য)
  res.setHeader('X-Trace-Id', span.spanContext().traceId);

  res.on('finish', () => {
    span.setAttribute('http.status_code', res.statusCode);
    if (res.statusCode >= 400) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: `HTTP ${res.statusCode}`,
      });
    }
    span.end();
  });

  // span-কে active context-এ সেট করুন
  context.with(trace.setSpan(context.active(), span), () => {
    next();
  });
}

module.exports = { traceContextMiddleware };
```

---

## ❌ Bad vs ✅ Good প্যাটার্ন

### প্যাটার্ন ১: Span নামকরণ

```javascript
// ❌ Bad: অর্থহীন span নাম
const span = tracer.startSpan('span1');
const span2 = tracer.startSpan('doStuff');
const span3 = tracer.startSpan('handler');

// ✅ Good: বর্ণনামূলক span নাম — অপারেশন স্পষ্ট
const span = tracer.startSpan('POST /api/send-money');
const span2 = tracer.startSpan('validate-recipient-account');
const span3 = tracer.startSpan('bkash-payment-gateway.charge');
```

### প্যাটার্ন ২: Error ট্র্যাকিং

```php
// ❌ Bad: Error হলে span-এ কিছু রেকর্ড করা হয়নি
try {
    processPayment($amount);
} catch (\Exception $e) {
    log_error($e->getMessage());  // শুধু লগ করেছে, span-এ নেই
}
$span->end();

// ✅ Good: Error span-এ রেকর্ড করা হয়েছে — Jaeger-এ দেখা যাবে
try {
    processPayment($amount);
    $span->setStatus(StatusCode::STATUS_OK);
} catch (\Throwable $e) {
    $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
    $span->recordException($e);  // full stack trace রেকর্ড হবে
    throw $e;
} finally {
    $span->end();  // সবসময় end() কল করুন
}
```

### প্যাটার্ন ৩: Attribute ব্যবহার

```javascript
// ❌ Bad: কোনো attribute নেই — trace দেখে কিছু বোঝা যায় না
const span = tracer.startSpan('process-order');
await processOrder(orderId, userId);
span.end();

// ✅ Good: প্রাসঙ্গিক attribute আছে — trace থেকে debugging সম্ভব
const span = tracer.startSpan('process-order', {
  attributes: {
    'order.id': orderId,
    'order.user_id': userId,
    'order.total_amount': totalAmount,
    'order.item_count': items.length,
    'order.payment_method': 'bkash',
    'order.shipping_city': 'Dhaka',
  },
});
await processOrder(orderId, userId);
span.addEvent('order_validated');
span.end();
```

### প্যাটার্ন ৪: Span Scope ম্যানেজমেন্ট

```php
// ❌ Bad: Span end() কল করা হয়নি — মেমোরি লিক ও ভুল timing
$span = $tracer->spanBuilder('db-query')->startSpan();
$scope = $span->activate();
$result = $db->query('SELECT * FROM users');
// end() কল করতে ভুলে গেছে! 😱

// ✅ Good: try-finally দিয়ে নিশ্চিত করা হয়েছে end() কল হবে
$span = $tracer->spanBuilder('db-query')->startSpan();
$scope = $span->activate();
try {
    $result = $db->query('SELECT * FROM users');
    $span->setAttribute('db.row_count', count($result));
} finally {
    $span->end();       // সবসময় কল হবে
    $scope->detach();   // context cleanup
}
```

### প্যাটার্ন ৫: Context Propagation

```javascript
// ❌ Bad: ডাউনস্ট্রিম সার্ভিসে trace context পাঠানো হয়নি
const response = await fetch('http://payment-service/charge', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ amount: 500 }),
});
// trace chain ভেঙে যাবে! ডাউনস্ট্রিম সার্ভিস নতুন trace শুরু করবে

// ✅ Good: trace context হেডারে inject করা হয়েছে
const headers = { 'Content-Type': 'application/json' };
propagation.inject(context.active(), headers);
// headers এখন traceparent ও tracestate ধারণ করে

const response = await fetch('http://payment-service/charge', {
  method: 'POST',
  headers,
  body: JSON.stringify({ amount: 500 }),
});
// trace chain অক্ষুণ্ণ থাকবে! 🎯
```

### প্যাটার্ন ৬: Span Granularity

```javascript
// ❌ Bad: একটি বিশাল span — কোথায় সময় লাগছে বোঝা যায় না
const span = tracer.startSpan('handle-request');
await validateUser();
await checkInventory();
await processPayment();
await createOrder();
await sendConfirmation();
span.end(); // 2 সেকেন্ড — কোথায় সমস্যা?

// ✅ Good: প্রতিটি ধাপের জন্য আলাদা child span — bottleneck চিহ্নিত করা সহজ
const parentSpan = tracer.startSpan('handle-request');

const s1 = tracer.startSpan('validate-user');
await validateUser();
s1.end(); // 50ms

const s2 = tracer.startSpan('check-inventory');
await checkInventory();
s2.end(); // 100ms

const s3 = tracer.startSpan('process-payment');
await processPayment();
s3.end(); // 1500ms ← bottleneck!

const s4 = tracer.startSpan('create-order');
await createOrder();
s4.end(); // 200ms

const s5 = tracer.startSpan('send-confirmation');
await sendConfirmation();
s5.end(); // 150ms

parentSpan.end();
```

---

## 🏠 বাস্তব উদাহরণ

### বাস্তব উদাহরণ ১: bKash Send Money ট্রানজ্যাকশন ট্রেসিং

```
bKash Send Money: 01712345678 → 01898765432, ৫০০ টাকা
═══════════════════════════════════════════════════════

Trace ID: 7f3e2a1b9c8d4e5f6a7b8c9d0e1f2a3b

Request Flow:

 ┌─────────┐     ┌─────────────┐     ┌─────────────┐
 │  bKash  │────→│ API Gateway │────→│ Auth Service │
 │   App   │     │  (Kong/Nginx)│     │ (JWT verify) │
 └─────────┘     └──────┬──────┘     └─────────────┘
                        │
            traceparent: 00-7f3e...3b-aabb001122-01
                        │
                        ▼
                 ┌──────────────┐
                 │ Transaction  │
                 │   Service    │
                 └──────┬───────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
   ┌────────────┐ ┌──────────┐ ┌────────────────┐
   │  Balance   │ │ Balance  │ │  Fraud Check   │
   │  Service   │ │ Service  │ │   Service      │
   │ (Sender    │ │(Receiver │ │ (ML-based      │
   │  debit)    │ │ credit)  │ │  analysis)     │
   └────────────┘ └──────────┘ └────────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Notification │──→ SMS: "৫০০ টাকা পাঠানো হয়েছে"
                 │   Service    │──→ Push Notification
                 └──────────────┘

Waterfall View:
──────────────────────────────────────────────────────────
সময়       0ms    100ms   200ms   300ms   400ms   500ms
           │       │       │       │       │       │
Gateway    ╠═══════════════════════════════════════════╣
Auth       │╠════╣ │       │       │       │       │
Txn Svc    │     ╠═══════════════════════════════╣   │
Fraud      │     │╠══════╣ │       │       │       │
Balance:D  │     │       ╠═══════╣ │       │       │
Balance:C  │     │       │       ╠═══════╣ │       │
Notify     │     │       │       │       ╠═══════╣ │
──────────────────────────────────────────────────────────
মোট সময়: ~480ms

Span Details:
  gateway     : 0-500ms  (root span)
  auth        : 5-60ms   (JWT validation)
  txn-service : 65-470ms (business logic)
  fraud-check : 70-150ms (ML fraud detection)
  balance-dbt : 155-270ms (sender debit, DB write)
  balance-crt : 275-380ms (receiver credit, DB write)
  notification: 385-465ms (SMS + push)
```

### বাস্তব উদাহরণ ২: Daraz অর্ডার ফ্লো ট্রেসিং

```
Daraz Order: ইউজার ১টি Samsung Galaxy + ১টি Case অর্ডার করলেন
═══════════════════════════════════════════════════════════════

Trace ID: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6

 ┌──────────┐     ┌─────────────┐
 │  Daraz   │────→│ API Gateway │
 │   App    │     │ (CDN/LB)    │
 └──────────┘     └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │ Order Service│
                  └──────┬───────┘
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌──────────┐    ┌──────────────┐    ┌──────────────┐
│Inventory │    │   Payment    │    │   Pricing    │
│ Service  │    │   Service    │    │   Service    │
│(stock চেক)│    │(bKash/Nagad) │    │(কুপন/ডিসকাউন্ট)│
└──────────┘    └──────┬───────┘    └──────────────┘
                       │
                       ▼
                ┌──────────────┐
                │   Shipping   │──→ Pathao Courier API
                │   Service    │──→ RedX Delivery API
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ Notification │──→ Email: "অর্ডার কনফার্মড"
                │   Service    │──→ SMS: "আপনার অর্ডার #12345"
                │              │──→ App Push Notification
                └──────────────┘

Span Hierarchy:
  [Root] POST /api/v2/orders                     (0-1200ms)
    ├── [Child] validate-user-session             (10-30ms)
    ├── [Child] calculate-pricing                 (35-120ms)
    │     ├── [Child] apply-coupon:DARAZ50        (40-60ms)
    │     └── [Child] calculate-vat               (65-80ms)
    ├── [Child] check-inventory                   (125-300ms)
    │     ├── [Child] check:samsung-galaxy-a54    (130-220ms)
    │     └── [Child] check:phone-case-001        (130-180ms)
    ├── [Child] process-payment:bkash             (310-850ms) ← সবচেয়ে ধীর
    │     ├── [Child] bkash-api:create-payment    (320-650ms)
    │     ├── [Child] bkash-api:execute-payment   (660-830ms)
    │     └── Event: "payment_confirmed" at 835ms
    ├── [Child] create-order-record               (860-950ms)
    │     └── [Child] db:insert:orders            (865-940ms)
    ├── [Child] initiate-shipping                 (955-1080ms)
    │     └── [Child] pathao-courier-api:create   (960-1070ms)
    └── [Child] send-notifications                (1085-1180ms)
          ├── [Child] send-email                  (1090-1150ms)
          └── [Child] send-sms                    (1090-1120ms)
```

---

## 📌 Jaeger Docker সেটআপ

### docker-compose.yml (Jaeger All-in-One)

```yaml
# docker-compose.yml
# Jaeger All-in-One — ডেভেলপমেন্ট ও টেস্টিংয়ের জন্য

version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:1.54
    container_name: jaeger
    restart: unless-stopped
    ports:
      - "16686:16686"   # Jaeger UI (ব্রাউজারে এই পোর্টে যান)
      - "4317:4317"     # OTLP gRPC receiver
      - "4318:4318"     # OTLP HTTP receiver
      - "14268:14268"   # Jaeger HTTP Thrift receiver
      - "14250:14250"   # Jaeger gRPC receiver
      - "6831:6831/udp" # Jaeger Agent compact thrift
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - LOG_LEVEL=info
      # প্রোডাকশনে Elasticsearch/Cassandra ব্যবহার করুন
      # - SPAN_STORAGE_TYPE=elasticsearch
      # - ES_SERVER_URLS=http://elasticsearch:9200

  # --- আপনার PHP সার্ভিস ---
  bkash-transaction-service:
    build: ./services/transaction
    container_name: bkash-txn-service
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318
      - OTEL_SERVICE_NAME=bkash-transaction-service
      - OTEL_TRACES_SAMPLER=parentbased_traceidratio
      - OTEL_TRACES_SAMPLER_ARG=0.1  # ১০% sampling
    depends_on:
      - jaeger

  # --- আপনার Node.js সার্ভিস ---
  daraz-order-service:
    build: ./services/order
    container_name: daraz-order-svc
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318
      - OTEL_SERVICE_NAME=daraz-order-service
      - OTEL_TRACES_SAMPLER=parentbased_traceidratio
      - OTEL_TRACES_SAMPLER_ARG=0.1
    depends_on:
      - jaeger
```

```bash
# Jaeger চালু করুন
docker compose up -d jaeger

# ব্রাউজারে Jaeger UI দেখুন
# http://localhost:16686

# লগ দেখুন
docker compose logs -f jaeger
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ডিস্ট্রিবিউটেড ট্রেসিং ব্যবহার করবেন

```
✅ মাইক্রোসার্ভিসেস আর্কিটেকচার ব্যবহার করলে
   → Pathao-র ride service, payment service, notification service ইত্যাদি

✅ রিকোয়েস্ট অনেক সার্ভিসের মধ্য দিয়ে যায়
   → bKash-এর Send Money: auth → txn → balance → notification

✅ পারফরম্যান্স বটলনেক খুঁজতে হলে
   → Daraz-এর checkout কেন ধীর? কোন সার্ভিসে সময় বেশি?

✅ ইন্টার-সার্ভিস ডিপেন্ডেন্সি বুঝতে হলে
   → Grameenphone-র কোন সার্ভিস কোন সার্ভিসকে কল করে?

✅ Error root cause analysis করতে হলে
   → Nagad-এ ব্যর্থ ট্রানজ্যাকশন — ঠিক কোন ধাপে সমস্যা?

✅ SLA/SLO মনিটরিং করতে হলে
   → "৯৯% রিকোয়েস্ট ৫০০ms-এর মধ্যে complete হতে হবে"
```

### ❌ কখন ডিস্ট্রিবিউটেড ট্রেসিং ব্যবহার করবেন না

```
❌ মনোলিথ অ্যাপ্লিকেশনে (সিঙ্গেল সার্ভিস)
   → APM (Application Performance Monitoring) যথেষ্ট

❌ খুব ছোট প্রজেক্টে (২-৩ জন ডেভেলপার)
   → শুধু structured logging দিয়ে কাজ চলবে

❌ ব্যাচ প্রসেসিং/ক্রন জবে (সাধারণত)
   → সিম্পল লগিং যথেষ্ট, যদি না জব অনেক সার্ভিস কল করে

❌ শুধু লগিংয়ের বিকল্প হিসেবে
   → ট্রেসিং ≠ লগিং; দুটো আলাদা উদ্দেশ্য পূরণ করে

❌ ইনফ্রাস্ট্রাকচার মনিটরিংয়ের জন্য
   → Prometheus/Grafana ব্যবহার করুন CPU, Memory, Disk মনিটরিংয়ে
```

### তিন পিলার অফ Observability

```
┌─────────────────────────────────────────────────────────────────┐
│              Three Pillars of Observability                      │
│                                                                 │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐     │
│  │   Logging    │   │   Metrics    │   │    Tracing       │     │
│  │  (লগিং)     │   │  (মেট্রিক্স)  │   │   (ট্রেসিং)      │     │
│  ├─────────────┤   ├──────────────┤   ├──────────────────┤     │
│  │ কী হয়েছে?   │   │ কতটুকু?       │   │ কোথায় সময়       │     │
│  │ Error msg,  │   │ Request/sec, │   │ লাগছে? কোন       │     │
│  │ stack trace │   │ CPU usage,   │   │ সার্ভিসে সমস্যা?  │     │
│  │             │   │ error rate   │   │                  │     │
│  ├─────────────┤   ├──────────────┤   ├──────────────────┤     │
│  │ ELK Stack   │   │ Prometheus   │   │ Jaeger           │     │
│  │ Loki        │   │ Grafana      │   │ Zipkin           │     │
│  │ CloudWatch  │   │ DataDog      │   │ AWS X-Ray        │     │
│  └─────────────┘   └──────────────┘   └──────────────────┘     │
│                                                                 │
│  তিনটিই একসাথে ব্যবহার করলে সেরা ফলাফল পাওয়া যায়!              │
│  Trace ID দিয়ে logs ও metrics কানেক্ট করুন = full picture 🎯   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────────────┐
│                     মনে রাখার চেকলিস্ট                           │
│                                                                  │
│  ☐ OpenTelemetry SDK সেটআপ করুন (vendor-neutral)                │
│  ☐ Auto instrumentation দিয়ে শুরু করুন (HTTP, DB, gRPC)         │
│  ☐ বিজনেস-critical ফ্লো-তে manual spans যোগ করুন                │
│  ☐ প্রতিটি span-এ অর্থপূর্ণ attributes দিন                       │
│  ☐ Error হলে recordException() কল করুন                          │
│  ☐ try-finally দিয়ে span.end() নিশ্চিত করুন                      │
│  ☐ Context propagation সঠিকভাবে করুন (traceparent header)       │
│  ☐ প্রোডাকশনে sampling কনফিগার করুন (সব ট্রেস রাখবেন না)        │
│  ☐ Jaeger/Zipkin সেটআপ করুন trace visualize করতে                │
│  ☐ Trace ID-কে logs-এর সাথে correlate করুন                     │
│  ☐ টিমকে Jaeger UI ব্যবহার শেখান                                │
│  ☐ Alert সেটআপ করুন slow/error traces-এর জন্য                  │
└──────────────────────────────────────────────────────────────────┘

মনে রাখুন: ডিস্ট্রিবিউটেড ট্রেসিং হলো মাইক্রোসার্ভিসেস আর্কিটেকচারে
debugging-এর সবচেয়ে শক্তিশালী হাতিয়ার। bKash, Daraz, Pathao-র মতো
বড় সিস্টেমে এটি ছাড়া সমস্যা খুঁজে বের করা প্রায় অসম্ভব। 🎯
```
