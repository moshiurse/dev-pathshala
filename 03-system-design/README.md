# 📘 সিস্টেম ডিজাইন

> বড় আকারের স্কেলেবল, রিলায়েবল ও পারফরম্যান্ট সিস্টেম ডিজাইনের মূল ধারণা ও কৌশল।
> প্রতিটি টপিকে ASCII ডায়াগ্রাম, ট্রেড-অফ বিশ্লেষণ, PHP + Node.js কোড এবং বাংলাদেশ context সহ গভীর বিশ্লেষণ।

---

## 📂 টপিক সূচি

### 🧠 মূল ধারণা (Core Concepts)

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | মূল ধারণা | [fundamentals.md](./fundamentals.md) | Scalability, Availability, Consistency, Latency, Estimation, RESHADED |
| ২ | লোড ব্যালেন্সিং | [load-balancing.md](./load-balancing.md) | ৮টি Algorithm, Nginx/HAProxy, AWS ALB/NLB, GSLB, Auto Scaling |
| ৩ | ক্যাশিং | [caching.md](./caching.md) | ৫টি Strategy, Redis Deep Dive, CDN, Cache Stampede, Multi-tier |
| ৪ | ডাটাবেস স্কেলিং | [database-scaling.md](./database-scaling.md) | Sharding, Consistent Hashing, Read Replicas, Federation, NewSQL |
| ৫ | মেসেজ কিউ | [message-queue.md](./message-queue.md) | RabbitMQ, Kafka, SQS, Redis Streams, Laravel Queue, BullMQ |
| ৬ | CAP থিওরেম | [cap-theorem.md](./cap-theorem.md) | CAP/PACELC, Quorum, Vector Clocks, CRDTs, ১৭+ DB Classification |
| ৭ | রেট লিমিটিং | [rate-limiting.md](./rate-limiting.md) | ৫টি Algorithm, Redis Lua, Nginx, Tiered Plans, DDoS Mitigation |
| ৮ | কনসিস্টেন্ট হ্যাশিং | [consistent-hashing.md](./consistent-hashing.md) | Hash Ring, Virtual Nodes, Distributed Systems |
| ৯ | CDN | [cdn.md](./cdn.md) | Push vs Pull CDN, Cache Invalidation, Edge Computing |
| ১০ | ফল্ট টলারেন্স ও রেজিলিয়েন্স | [fault-tolerance.md](./fault-tolerance.md) | Circuit Breaker, Retry, Timeout, Bulkhead, Fallback, Chaos Engineering |

### 📋 কেস স্টাডি (System Design Interviews)

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ৮ | URL শর্টনার | [url-shortener.md](./url-shortener.md) | Base62, Bloom Filter, Analytics, Full Laravel + Express Code |
| ৯ | চ্যাট সিস্টেম | [chat-system.md](./chat-system.md) | WebSocket, Cassandra, E2EE, Presence, Read Receipts, Media |
| ১০ | নিউজফিড | [newsfeed.md](./newsfeed.md) | Fan-out Write/Read, Hybrid, Ranking Algorithm, Real-time |
| ১১ | নোটিফিকেশন সিস্টেম | [notification-system.md](./notification-system.md) | WebSocket, SSE, Polling, FCM Push |
| ১২ | সার্চ অটোকমপ্লিট | [search-autocomplete.md](./search-autocomplete.md) | Trie, Elasticsearch, Debounce |
| ১৩ | ওয়েব ক্রলার | [web-crawler.md](./web-crawler.md) | BFS Crawling, URL Frontier, Deduplication |
| ১৪ | ভিডিও স্ট্রিমিং | [video-streaming.md](./video-streaming.md) | Transcoding, HLS/DASH, Adaptive Bitrate |
| ১৫ | পেমেন্ট সিস্টেম | [payment-system.md](./payment-system.md) | Payment Gateway, Idempotency, Double-Entry, PCI DSS, Reconciliation |
| ১৬ | ডিস্ট্রিবিউটেড আইডি জেনারেশন | [distributed-id.md](./distributed-id.md) | UUID, Snowflake, ULID, Instagram-style ID, Comparison |
| ১৭ | ফাইল স্টোরেজ সিস্টেম | [file-storage.md](./file-storage.md) | S3, Pre-signed URL, Multipart Upload, CDN, Image Pipeline |

---

## 🗺️ সিস্টেম কম্পোনেন্ট সম্পর্ক

```
    Client ──► CDN ──► Load Balancer
                            │
                   ┌────────┼────────┐
                   ▼        ▼        ▼
               App Server  App    App Server
                   │        │        │
              ┌────┴────┐   │   ┌────┴────┐
              ▼         ▼   ▼   ▼         ▼
           Cache    Message Queue    Rate Limiter
          (Redis)   (Kafka/RabbitMQ)
              │         │
              ▼         ▼
           Database (Primary + Replicas + Shards)
```

---

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ বাংলায় বিস্তারিত ব্যাখ্যা ও ASCII আর্কিটেকচার ডায়াগ্রাম
- ✅ PHP (Laravel) + Node.js (Express) কোড উদাহরণ
- ✅ Back-of-envelope estimation (কেস স্টাডিতে)
- ✅ ট্রেড-অফ বিশ্লেষণ ও Decision Guide
- ✅ Advanced scenarios ও Scaling strategies
- ✅ বাংলাদেশ context (bKash, Daraz, Pathao উদাহরণ)
