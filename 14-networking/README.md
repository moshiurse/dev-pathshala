# 📡 নেটওয়ার্কিং ফান্ডামেন্টালস

> নেটওয়ার্কিং এর মৌলিক ধারণা থেকে শুরু করে অ্যাডভান্সড প্রোটোকল পর্যন্ত।
> প্রতিটি টপিকে PHP, JavaScript (Node.js) উদাহরণ, ASCII ডায়াগ্রাম এবং বাংলাদেশী রিয়েল-ওয়ার্ল্ড উদাহরণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | OSI মডেল | [osi-model.md](./osi-model.md) | ৭ লেয়ার, TCP/IP তুলনা, পোস্টাল সিস্টেম অ্যানালজি, ওয়েব রিকোয়েস্ট ফ্লো |
| ২ | TCP ও UDP | [tcp-udp.md](./tcp-udp.md) | Three-way Handshake, Flow Control, Congestion Control, Socket Programming |
| ৩ | HTTP প্রোটোকল | [http-protocol.md](./http-protocol.md) | HTTP/1.1, HTTP/2, HTTP/3 (QUIC), Methods, Status Codes, Headers |
| ৪ | WebSockets | [websockets.md](./websockets.md) | Full-Duplex, Socket.IO, Ratchet, SSE তুলনা, রিয়েল-টাইম অ্যাপ |
| ৫ | DNS | [dns.md](./dns.md) | DNS Resolution, Record Types, Caching, DNSSEC, Load Balancing |
| ৬ | TLS/SSL | [tls-ssl.md](./tls-ssl.md) | TLS 1.2 vs 1.3, Certificate Chain, Let's Encrypt, mTLS |
| ৭ | WebRTC | [webrtc.md](./webrtc.md) | ICE/STUN/TURN, SDP, P2P Media, DataChannel, SFU/MCU (Mediasoup/LiveKit), coturn, NAT Traversal |

---

## 🗺️ টপিক সম্পর্ক ডায়াগ্রাম

```
    ┌─────────────────────────────────────────────────────┐
    │              নেটওয়ার্কিং ফান্ডামেন্টালস                │
    └──────────────────────┬──────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    ┌──────────┐    ┌──────────┐     ┌───────────┐
    │ OSI মডেল  │    │   DNS    │     │  TLS/SSL  │
    │ (ভিত্তি)   │    │(নেম রেজ.)│     │ (সিকিউরিটি)│
    └─────┬────┘    └──────────┘     └─────┬─────┘
          │                                │
    ┌─────▼──────────────────────────┐     │
    │       TCP / UDP               │     │
    │  (ট্রান্সপোর্ট লেয়ার)         │     │
    └─────┬──────────────────────────┘     │
          │                                │
    ┌─────▼────────────────────────────────▼──┐
    │           HTTP প্রোটোকল                  │
    │    (HTTP/1.1 → HTTP/2 → HTTP/3)        │
    └─────────────────┬───────────────────────┘
                      │
              ┌───────▼────────┐
              │  WebSockets    │
              │ (রিয়েল-টাইম)   │
              └────────────────┘
```

---

## 🎯 শেখার পথ (Learning Path)

```
Step 1: OSI মডেল → নেটওয়ার্কিং এর ভিত্তি বুঝুন
         │
Step 2: TCP/UDP → ডেটা কীভাবে ট্রান্সফার হয়
         │
Step 3: DNS → ডোমেইন নেম কীভাবে IP তে রূপান্তর হয়
         │
Step 4: TLS/SSL → যোগাযোগ কীভাবে সুরক্ষিত হয়
         │
Step 5: HTTP → ওয়েবের ভাষা শিখুন
         │
Step 6: WebSockets → রিয়েল-টাইম কমিউনিকেশন
```

---

## 🏠 বাংলাদেশী প্রসঙ্গ

| সার্ভিস | ব্যবহৃত প্রযুক্তি | টপিক |
|---------|-------------------|------|
| bKash | HTTPS, TLS 1.3, TCP | HTTP, TLS/SSL |
| Pathao | WebSocket (Real-time tracking) | WebSockets |
| Daraz | CDN, DNS Load Balancing | DNS, HTTP/2 |
| Grameenphone | UDP (VoLTE), TCP | TCP/UDP |
| Nagad | mTLS (Service-to-Service) | TLS/SSL |
| Prothom Alo | HTTP/2, Server Push | HTTP Protocol |

---

## 📊 প্রোটোকল তুলনা সারসংক্ষেপ

```
┌──────────────┬──────────┬───────────┬──────────┬────────────┐
│  বৈশিষ্ট্য    │   TCP    │    UDP    │   HTTP   │ WebSocket  │
├──────────────┼──────────┼───────────┼──────────┼────────────┤
│ Reliable     │    ✅    │    ❌     │    ✅    │     ✅     │
│ Ordered      │    ✅    │    ❌     │    ✅    │     ✅     │
│ Connection   │  Conn.   │ Connless  │ Req/Res  │ Full-Duplex│
│ Speed        │  Medium  │   Fast    │  Medium  │    Fast    │
│ Overhead     │  High    │   Low     │  Medium  │    Low     │
│ Use Case     │  Web/API │  Video/   │  REST    │  Real-time │
│              │          │  Gaming   │  API     │  Chat      │
└──────────────┴──────────┴───────────┴──────────┴────────────┘
```

---

> 💡 **টিপ**: প্রতিটি ফাইলে কোড উদাহরণ, ASCII ডায়াগ্রাম এবং বাস্তব-জীবনের প্রয়োগ রয়েছে।
> নতুন হলে OSI মডেল দিয়ে শুরু করুন, তারপর ক্রমানুসারে এগিয়ে যান।
