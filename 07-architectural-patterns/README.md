# 📘 আর্কিটেকচারাল প্যাটার্ন

> সফটওয়্যার সিস্টেমের বড় আকারের স্ট্রাকচার ডিজাইন করার প্যাটার্নসমূহ।
> প্রতিটি টপিকে Advanced complex scenarios, PHP 8.3 + Node.js ES2022+ কোড, ASCII ডায়াগ্রাম, এবং বাংলাদেশ context সহ গভীর বিশ্লেষণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | MVC প্যাটার্ন | [mvc.md](./mvc.md) | MVC/MVP/MVVM তুলনা, Service Layer, Caching Strategy, API Resources, Anti-patterns |
| ২ | মাইক্রোসার্ভিস | [microservices.md](./microservices.md) | API Gateway, Saga, Circuit Breaker, Service Mesh, Distributed Tracing, Strangler Fig |
| ৩ | মনোলিথিক | [monolithic.md](./monolithic.md) | Modular Monolith, Scaling, Background Jobs, মাইক্রোসার্ভিসে Migration |
| ৪ | ইভেন্ট-ড্রিভেন | [event-driven.md](./event-driven.md) | Event Sourcing, RabbitMQ/Kafka, Saga, WebSocket/SSE, Outbox Pattern |
| ৫ | CQRS | [cqrs.md](./cqrs.md) | Command/Query Bus, Materialized Views, Eventual Consistency, Multi-tenant |
| ৬ | হেক্সাগোনাল | [hexagonal.md](./hexagonal.md) | Ports & Adapters, DDD, Swappable Adapters, Clean/Onion তুলনা |
| ৭ | সার্ভারলেস | [serverless.md](./serverless.md) | Lambda (Bref+Node), Step Functions, WebSocket, Cost Analysis, Cold Start |
| ৮ | রিপোজিটরি প্যাটার্ন | [repository-pattern.md](./repository-pattern.md) | Specification, Caching Decorator, Unit of Work, Multi-Database, Criteria Builder |
| ৯ | ক্লিন আর্কিটেকচার | [clean-architecture.md](./clean-architecture.md) | Uncle Bob's Clean Architecture, Dependency Rule, Use Cases, Entity Layer |
| ১০ | লেয়ার্ড আর্কিটেকচার | [layered-architecture.md](./layered-architecture.md) | Presentation, Business, Persistence Layer, Strict vs Relaxed Layering |
| ১১ | পাইপ অ্যান্ড ফিল্টার | [pipe-and-filter.md](./pipe-and-filter.md) | Pipeline Pattern, Data Transformation, Middleware Chain, ETL |
| ১২ | রেন্ডারিং প্যাটার্ন | [rendering-patterns.md](./rendering-patterns.md) | SSR, CSR, SSG, ISR, Hydration, Island Architecture, Inertia.js |

---

## 🗺️ প্যাটার্ন সম্পর্ক ডায়াগ্রাম

```
                    ┌─────────────────┐
                    │   Monolithic     │
                    │  (শুরুর পয়েন্ট)  │
                    └────────┬────────┘
                             │ Strangler Fig
                             ▼
┌──────────┐    ┌─────────────────────────┐    ┌──────────────┐
│   MVC    │◄───│    Microservices         │───►│  Serverless  │
│ (UI Layer)│    │  (সার্ভিস ভিত্তিক)       │    │  (FaaS)      │
└──────────┘    └────────┬───┬────────────┘    └──────────────┘
                         │   │
              ┌──────────┘   └──────────┐
              ▼                         ▼
    ┌─────────────────┐      ┌──────────────────┐
    │  Event-Driven    │      │   Hexagonal       │
    │  (ইভেন্ট ভিত্তিক) │◄────►│  (Ports+Adapters) │
    └────────┬────────┘      └────────┬─────────┘
             │                        │
             ▼                        ▼
    ┌─────────────────┐      ┌──────────────────┐
    │     CQRS         │      │   Repository      │
    │ (Read/Write আলাদা)│◄────►│  (Data Access)    │
    └─────────────────┘      └──────────────────┘
```

---

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ বাংলায় বিস্তারিত ব্যাখ্যা ও বাস্তব জীবনের analogy
- ✅ ASCII আর্কিটেকচার ডায়াগ্রাম
- ✅ PHP 8.3 (Laravel) + Node.js ES2022+ (Express) কোড উদাহরণ
- ✅ Advanced complex scenarios (Distributed systems, Saga, Event Sourcing)
- ✅ সুবিধা (Pros) ও অসুবিধা (Cons) টেবিল
- ✅ Anti-patterns সহ ❌ Bad → ✅ Good কোড উদাহরণ
- ✅ টেস্টিং (PHPUnit + Jest)
- ✅ কখন ব্যবহার করবেন / করবেন না
- ✅ অন্যান্য প্যাটার্নের সাথে সম্পর্ক
- ✅ বাংলাদেশ context (bKash, Pathao, Daraz উদাহরণ)
