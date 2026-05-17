# 🌍 রিয়েল-ওয়ার্ল্ড সিস্টেম ডিজাইন কেস স্টাডি

> বাস্তব জীবনের জনপ্রিয় সিস্টেমগুলোর বিস্তারিত ডিজাইন — ASCII ডায়াগ্রাম, ট্রেড-অফ বিশ্লেষণ, PHP (Laravel) + Node.js (Express) কোড এবং বাংলাদেশ context সহ।
> প্রতিটি কেস স্টাডি সিস্টেম ডিজাইন ইন্টারভিউয়ের জন্য প্রস্তুত।

---

## 📂 কেস স্টাডি সূচি

| # | সিস্টেম | ফাইল | মূল বিষয়বস্তু |
|---|---------|------|--------------|
| ১ | 🔗 URL শর্টনার (TinyURL) | [url-shortener-tinyurl.md](./url-shortener-tinyurl.md) | Base62, Counter Range, Bloom Filter, Analytics Pipeline |
| ২ | 🎬 টিকেটিং সিস্টেম (BookMyShow) | [ticketing-system-bookmyshow.md](./ticketing-system-bookmyshow.md) | Seat Locking, Distributed Lock, Virtual Queue, Dynamic Pricing |
| ৩ | 📰 নিউজ ফিড (Twitter/Instagram) | [news-feed-twitter-instagram.md](./news-feed-twitter-instagram.md) | Fan-out Write/Read, Hybrid Approach, Feed Ranking, Real-time |
| ৪ | 🔔 নোটিফিকেশন সিস্টেম | [notification-system.md](./notification-system.md) | Multi-channel (Push/SMS/Email/WebSocket), Priority Queue, Retry |
| ৫ | 💬 চ্যাট অ্যাপ্লিকেশন (WhatsApp) | [chat-application-whatsapp.md](./chat-application-whatsapp.md) | WebSocket, E2E Encryption, Offline Queue, Group Fan-out |
| ৬ | 🏷️ অকশন প্ল্যাটফর্ম (eBay) | [auction-platform-ebay.md](./auction-platform-ebay.md) | Real-time Bidding, Proxy Bid, Escrow, Fraud Detection |
| ৭ | 🏠 অনলাইন রেন্টাল (Airbnb) | [rental-platform-airbnb.md](./rental-platform-airbnb.md) | Geo-search, Booking Calendar, Dynamic Pricing, Trust System |
| ৮ | ☁️ ক্লাউড স্টোরেজ (Google Drive) | [cloud-storage-google-drive.md](./cloud-storage-google-drive.md) | Block-level Sync, Chunked Upload, Deduplication, Versioning |
| ৯ | 🎥 ভিডিও শেয়ারিং (YouTube) | [video-sharing-youtube.md](./video-sharing-youtube.md) | Transcoding Pipeline, HLS/DASH, Adaptive Bitrate, CDN |
| ১০ | 🔍 সার্চ ইঞ্জিন (Google) | [search-engine-google.md](./search-engine-google.md) | Web Crawler, Inverted Index, PageRank, Autocomplete |
| ১১ | 🛒 ই-কমার্স (Amazon) | [ecommerce-platform-amazon.md](./ecommerce-platform-amazon.md) | Saga Pattern, Inventory Lock, Payment Idempotency, Flash Sale |
| ১২ | 🚕 ট্যাক্সি হেইলিং (Uber) | [taxi-hailing-uber.md](./taxi-hailing-uber.md) | Geohash/H3, Driver Matching, Surge Pricing, Real-time Tracking |
| ১৩ | 📝 কোলাবোরেটিভ এডিটর (Google Docs) | [collaborative-editor-google-docs.md](./collaborative-editor-google-docs.md) | OT vs CRDT, Real-time Sync, Conflict Resolution, Presence |

---

## 📖 প্রতিটি কেস স্টাডিতে যা পাবেন

- ✅ **Requirements** — Functional ও Non-Functional রিকোয়ারমেন্টস
- ✅ **Back-of-envelope Estimation** — ট্রাফিক, স্টোরেজ, ব্যান্ডউইথ হিসাব
- ✅ **High-Level Design** — ASCII আর্কিটেকচার ডায়াগ্রাম
- ✅ **Detailed Design** — ডাটা মডেল, অ্যালগরিদম, কোড উদাহরণ
- ✅ **ট্রেড-অফ বিশ্লেষণ** — বিভিন্ন পদ্ধতির তুলনামূলক বিশ্লেষণ
- ✅ **কেস স্টাডি** — বাংলাদেশ context সহ বাস্তব পরিস্থিতি
- ✅ **Advanced Topics** — স্কেলিং, সিকিউরিটি, ML ইন্টিগ্রেশন

---

## 🗺️ সিস্টেম ক্যাটাগরি

```
┌─────────────────────────────────────────────────────┐
│              রিয়েল-ওয়ার্ল্ড সিস্টেম ডিজাইন           │
├─────────────────────────────────────────────────────┤
│                                                     │
│  📡 রিয়েল-টাইম সিস্টেম                              │
│  ├── 💬 চ্যাট (WhatsApp)                            │
│  ├── 📝 কোলাবোরেটিভ এডিটর (Google Docs)             │
│  ├── 🔔 নোটিফিকেশন সিস্টেম                          │
│  └── 🏷️ অকশন (eBay)                                │
│                                                     │
│  🔍 ডাটা-ইনটেনসিভ সিস্টেম                            │
│  ├── 📰 নিউজ ফিড (Twitter)                          │
│  ├── 🔍 সার্চ ইঞ্জিন (Google)                        │
│  ├── 🎥 ভিডিও প্ল্যাটফর্ম (YouTube)                  │
│  └── 🔗 URL শর্টনার (TinyURL)                       │
│                                                     │
│  💰 ট্রানজ্যাকশনাল সিস্টেম                           │
│  ├── 🎬 টিকেটিং (BookMyShow)                       │
│  ├── 🛒 ই-কমার্স (Amazon)                           │
│  └── 🚕 রাইড শেয়ারিং (Uber)                         │
│                                                     │
│  📦 স্টোরেজ সিস্টেম                                  │
│  ├── ☁️ ক্লাউড স্টোরেজ (Google Drive)                │
│  └── 🏠 রেন্টাল প্ল্যাটফর্ম (Airbnb)                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

> **নোট:** প্রতিটি কেস স্টাডি স্বতন্ত্রভাবে পড়া যায়। তবে Core Concepts (মূল ধারণা) ফোল্ডারের টপিকগুলো আগে পড়লে এই কেস স্টাডিগুলো আরও ভালো বুঝবেন।
