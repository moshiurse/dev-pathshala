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
| ১৮ | লোড শেডিং | [load-shedding.md](./load-shedding.md) | Graceful Degradation, Priority-based Drop, Adaptive Concurrency, Retry-After, BD Surge Examples |
| ১৯ | RabbitMQ / AMQP | [rabbitmq-amqp.md](./rabbitmq-amqp.md) | AMQP 0-9-1, Exchanges (direct/topic/fanout), Quorum/Stream Queues, DLX, Federation, vs Kafka |

### 🌍 রিয়েল-ওয়ার্ল্ড সিস্টেম ডিজাইন (Real-World Examples)

> বাস্তব জীবনের জনপ্রিয় সিস্টেমগুলোর বিস্তারিত কেস স্টাডি — [সম্পূর্ণ সূচি](./real-world-examples/README.md)

| # | সিস্টেম | ফাইল | মূল বিষয়বস্তু |
|---|---------|------|--------------|
| ১ | 🔗 URL শর্টনার (TinyURL) | [url-shortener-tinyurl.md](./real-world-examples/url-shortener-tinyurl.md) | Base62, Counter Range, Bloom Filter, Analytics Pipeline |
| ২ | 🎬 টিকেটিং সিস্টেম (BookMyShow) | [ticketing-system-bookmyshow.md](./real-world-examples/ticketing-system-bookmyshow.md) | Seat Locking, Distributed Lock, Virtual Queue, Dynamic Pricing |
| ৩ | 📰 নিউজ ফিড (Twitter/Instagram) | [news-feed-twitter-instagram.md](./real-world-examples/news-feed-twitter-instagram.md) | Fan-out Write/Read, Hybrid Approach, Feed Ranking |
| ৪ | 🔔 নোটিফিকেশন সিস্টেম | [notification-system.md](./real-world-examples/notification-system.md) | Multi-channel, Priority Queue, Retry & Dedup |
| ৫ | 💬 চ্যাট অ্যাপ (WhatsApp) | [chat-application-whatsapp.md](./real-world-examples/chat-application-whatsapp.md) | WebSocket, E2E Encryption, Offline Queue, Group Fan-out |
| ৬ | 🏷️ অকশন প্ল্যাটফর্ম (eBay) | [auction-platform-ebay.md](./real-world-examples/auction-platform-ebay.md) | Real-time Bidding, Proxy Bid, Escrow, Fraud Detection |
| ৭ | 🏠 রেন্টাল প্ল্যাটফর্ম (Airbnb) | [rental-platform-airbnb.md](./real-world-examples/rental-platform-airbnb.md) | Geo-search, Booking Calendar, Dynamic Pricing |
| ৮ | ☁️ ক্লাউড স্টোরেজ (Google Drive) | [cloud-storage-google-drive.md](./real-world-examples/cloud-storage-google-drive.md) | Block-level Sync, Chunked Upload, Deduplication |
| ৯ | 🎥 ভিডিও শেয়ারিং (YouTube) | [video-sharing-youtube.md](./real-world-examples/video-sharing-youtube.md) | Transcoding Pipeline, HLS/DASH, Adaptive Bitrate, CDN |
| ১০ | 🔍 সার্চ ইঞ্জিন (Google) | [search-engine-google.md](./real-world-examples/search-engine-google.md) | Web Crawler, Inverted Index, PageRank, Autocomplete |
| ১১ | 🛒 ই-কমার্স (Amazon) | [ecommerce-platform-amazon.md](./real-world-examples/ecommerce-platform-amazon.md) | Saga Pattern, Inventory Lock, Payment Idempotency |
| ১২ | 🚕 ট্যাক্সি হেইলিং (Uber) | [taxi-hailing-uber.md](./real-world-examples/taxi-hailing-uber.md) | Geohash/H3, Driver Matching, Surge Pricing |
| ১৩ | 📝 কোলাবোরেটিভ এডিটর (Google Docs) | [collaborative-editor-google-docs.md](./real-world-examples/collaborative-editor-google-docs.md) | OT vs CRDT, Real-time Sync, Conflict Resolution |

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
