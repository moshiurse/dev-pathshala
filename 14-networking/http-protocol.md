# 🌐 HTTP প্রোটোকল (HyperText Transfer Protocol)

> HTTP হলো ওয়েবের মূল ভাষা। ব্রাউজার এবং সার্ভারের মধ্যে যোগাযোগের প্রোটোকল।
> HTTP/1.1 থেকে HTTP/3 পর্যন্ত প্রতিটি ভার্সনের গভীর আলোচনা।

---

## 📖 সূচিপত্র

- [HTTP কী?](#-http-কী)
- [Request/Response Structure](#-requestresponse-structure)
- [HTTP Methods](#-http-methods)
- [Status Codes](#-status-codes)
- [HTTP Headers](#-http-headers)
- [HTTP/1.1](#-http11)
- [HTTP/2](#-http2)
- [HTTP/3 (QUIC)](#-http3-quic)
- [Version তুলনা](#-http-version-তুলনা)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 HTTP কী?

HTTP হলো একটি **Application Layer** প্রোটোকল যা **Client-Server** মডেলে কাজ করে। ক্লায়েন্ট (ব্রাউজার) রিকোয়েস্ট পাঠায়, সার্ভার রেসপন্স দেয়।

```
  Client (ব্রাউজার)                    Server (Daraz)
  ┌──────────────┐                  ┌──────────────┐
  │              │  HTTP Request    │              │
  │   Chrome/    │ ───────────────► │   Nginx/     │
  │   Firefox    │                  │   Apache     │
  │              │  HTTP Response   │              │
  │              │ ◄─────────────── │              │
  └──────────────┘                  └──────────────┘

  রিকোয়েস্ট: "আমাকে iPhone 15 এর তথ্য দাও"
  রেসপন্স: "এই নাও HTML পেজ + প্রোডাক্ট ডেটা"
```

---

## 📖 Request/Response Structure

### HTTP Request

```
┌─────────────────────────────────────────────────┐
│ Request Line                                     │
│   GET /products/iphone-15 HTTP/1.1              │
├─────────────────────────────────────────────────┤
│ Headers                                          │
│   Host: www.daraz.com.bd                        │
│   User-Agent: Mozilla/5.0                       │
│   Accept: text/html                             │
│   Accept-Language: bn-BD,bn;q=0.9              │
│   Accept-Encoding: gzip, deflate, br           │
│   Connection: keep-alive                        │
│   Cookie: session_id=abc123                     │
├─────────────────────────────────────────────────┤
│ Empty Line (CRLF)                               │
├─────────────────────────────────────────────────┤
│ Body (POST/PUT এ থাকে)                          │
│   {"name": "iPhone 15", "qty": 1}              │
└─────────────────────────────────────────────────┘
```

### HTTP Response

```
┌─────────────────────────────────────────────────┐
│ Status Line                                      │
│   HTTP/1.1 200 OK                               │
├─────────────────────────────────────────────────┤
│ Headers                                          │
│   Content-Type: text/html; charset=UTF-8        │
│   Content-Length: 45678                          │
│   Content-Encoding: gzip                        │
│   Cache-Control: max-age=3600                   │
│   Set-Cookie: session_id=xyz789                 │
│   X-Powered-By: Laravel                         │
├─────────────────────────────────────────────────┤
│ Empty Line (CRLF)                               │
├─────────────────────────────────────────────────┤
│ Body                                             │
│   <!DOCTYPE html>                                │
│   <html><head>...</head>                        │
│   <body>iPhone 15 - ৳159,999</body></html>     │
└─────────────────────────────────────────────────┘
```

---

## 📊 HTTP Methods

```
┌─────────┬─────────────────────┬──────────┬──────────┬────────────┐
│ Method  │      উদ্দেশ্য         │ Body আছে │Idempotent│   Safe     │
├─────────┼─────────────────────┼──────────┼──────────┼────────────┤
│ GET     │ ডেটা আনা             │   ❌     │    ✅    │    ✅      │
│ POST    │ নতুন রিসোর্স তৈরি     │   ✅     │    ❌    │    ❌      │
│ PUT     │ পুরো রিসোর্স আপডেট    │   ✅     │    ✅    │    ❌      │
│ PATCH   │ আংশিক আপডেট         │   ✅     │    ❌    │    ❌      │
│ DELETE  │ রিসোর্স মুছে ফেলা     │   ❌     │    ✅    │    ❌      │
│ HEAD    │ শুধু হেডার আনা        │   ❌     │    ✅    │    ✅      │
│ OPTIONS │ সাপোর্টেড মেথড জানা   │   ❌     │    ✅    │    ✅      │
└─────────┴─────────────────────┴──────────┴──────────┴────────────┘

Idempotent = বারবার কল করলেও একই ফল
Safe = সার্ভারের ডেটা পরিবর্তন করে না
```

### বাস্তব উদাহরণ (Daraz API)

```
GET    /api/products              → সব প্রোডাক্ট দেখা
GET    /api/products/123          → iPhone 15 দেখা
POST   /api/products              → নতুন প্রোডাক্ট যোগ
PUT    /api/products/123          → iPhone 15 পুরো আপডেট
PATCH  /api/products/123          → শুধু দাম পরিবর্তন
DELETE /api/products/123          → প্রোডাক্ট মুছে ফেলা

POST   /api/orders                → নতুন অর্ডার দেওয়া
GET    /api/orders/456            → অর্ডার ট্র্যাকিং
PATCH  /api/orders/456/cancel     → অর্ডার বাতিল
```

---

## 📊 Status Codes

```
┌──────────┬──────────────────────────────────────────────────┐
│ Range    │                    অর্থ                          │
├──────────┼──────────────────────────────────────────────────┤
│ 1xx      │ Informational (তথ্যমূলক)                        │
│ 2xx      │ Success (সফল) ✅                                │
│ 3xx      │ Redirection (পুনঃনির্দেশ) ↪️                     │
│ 4xx      │ Client Error (ক্লায়েন্ট ভুল) ❌                  │
│ 5xx      │ Server Error (সার্ভার ভুল) 💥                    │
└──────────┴──────────────────────────────────────────────────┘

প্রধান Status Codes:

┌──────┬────────────────────────┬──────────────────────────────┐
│ Code │         নাম             │     বাস্তব উদাহরণ             │
├──────┼────────────────────────┼──────────────────────────────┤
│ 200  │ OK                     │ প্রোডাক্ট পেজ লোড হয়েছে      │
│ 201  │ Created                │ নতুন অর্ডার তৈরি হয়েছে       │
│ 204  │ No Content             │ প্রোডাক্ট ডিলিট সফল          │
│ 301  │ Moved Permanently      │ পুরানো URL → নতুন URL        │
│ 302  │ Found (Temp Redirect)  │ লগইন → ড্যাশবোর্ড            │
│ 304  │ Not Modified           │ ক্যাশ থেকে দেখাও             │
│ 400  │ Bad Request            │ ভুল ফরম্যাটের JSON           │
│ 401  │ Unauthorized           │ লগইন করেননি                  │
│ 403  │ Forbidden              │ এডমিন প্যানেলে অ্যাক্সেস নেই  │
│ 404  │ Not Found              │ প্রোডাক্ট পাওয়া যায়নি        │
│ 405  │ Method Not Allowed     │ DELETE /users → অনুমোদিত নয়  │
│ 409  │ Conflict               │ ডুপ্লিকেট অর্ডার              │
│ 422  │ Unprocessable Entity   │ ভ্যালিডেশন ব্যর্থ             │
│ 429  │ Too Many Requests      │ Rate limit exceeded          │
│ 500  │ Internal Server Error  │ সার্ভারে বাগ                  │
│ 502  │ Bad Gateway            │ Upstream সার্ভার ডাউন         │
│ 503  │ Service Unavailable    │ সার্ভার মেইনটেন্যান্সে         │
│ 504  │ Gateway Timeout        │ সার্ভার সাড়া দেয়নি           │
└──────┴────────────────────────┴──────────────────────────────┘
```

---

## 📖 HTTP Headers

### গুরুত্বপূর্ণ Request Headers

```
┌──────────────────────┬────────────────────────────────────┐
│ Header               │ উদ্দেশ্য                             │
├──────────────────────┼────────────────────────────────────┤
│ Host                 │ সার্ভারের ডোমেইন নাম                │
│ User-Agent           │ ক্লায়েন্ট সফটওয়্যার তথ্য            │
│ Accept               │ কোন ফরম্যাট চাই (json/html)       │
│ Accept-Language      │ ভাষা (bn-BD, en-US)               │
│ Accept-Encoding      │ কম্প্রেশন (gzip, br)               │
│ Authorization        │ অথেনটিকেশন টোকেন                  │
│ Cookie               │ সেশন তথ্য                          │
│ Content-Type         │ Body এর ফরম্যাট                    │
│ Content-Length       │ Body এর সাইজ                      │
│ Cache-Control        │ ক্যাশিং নির্দেশনা                    │
│ If-None-Match        │ ETag ভিত্তিক ক্যাশিং                │
│ If-Modified-Since    │ তারিখ ভিত্তিক ক্যাশিং               │
│ Origin               │ CORS এর জন্য অরিজিন                │
│ Referer              │ কোন পেজ থেকে এসেছে               │
│ X-Request-ID         │ রিকোয়েস্ট ট্র্যাকিং                  │
└──────────────────────┴────────────────────────────────────┘
```

### গুরুত্বপূর্ণ Response Headers

```
┌──────────────────────┬────────────────────────────────────┐
│ Header               │ উদ্দেশ্য                             │
├──────────────────────┼────────────────────────────────────┤
│ Content-Type         │ রেসপন্স ফরম্যাট                     │
│ Content-Length       │ রেসপন্স সাইজ                       │
│ Content-Encoding     │ কম্প্রেশন মেথড                      │
│ Cache-Control        │ ক্যাশিং নীতি                        │
│ ETag                 │ রিসোর্স ভার্সন ট্যাগ                │
│ Last-Modified        │ শেষ পরিবর্তনের সময়                  │
│ Set-Cookie           │ কুকি সেট করা                       │
│ Location             │ রিডাইরেক্ট URL                      │
│ Access-Control-*     │ CORS হেডার                        │
│ X-RateLimit-*        │ Rate limiting তথ্য                  │
│ Strict-Transport-    │ HTTPS বাধ্যতামূলক                   │
│   Security           │                                    │
│ X-Content-Type-      │ MIME sniffing প্রতিরোধ              │
│   Options            │                                    │
└──────────────────────┴────────────────────────────────────┘
```

---

## 📌 HTTP/1.1

HTTP/1.1 (১৯৯৭ সাল থেকে) এখনো ব্যাপকভাবে ব্যবহৃত হয়।

```
┌─────────────────────────────────────────────────────────┐
│                    HTTP/1.1 বৈশিষ্ট্য                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Keep-Alive (একই TCP সংযোগে একাধিক request)          │
│  ✅ Pipelining (একসাথে একাধিক request পাঠানো)            │
│  ✅ Chunked Transfer Encoding                            │
│  ✅ Host Header (একই IP তে একাধিক ওয়েবসাইট)             │
│  ❌ Head-of-Line (HOL) Blocking                          │
│  ❌ Text-based protocol (বেশি bandwidth)                  │
│  ❌ প্রতি request এ সব header পুনরায় পাঠানো               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Keep-Alive

```
HTTP/1.0 (Keep-Alive ছাড়া):
──────────────────────────
Client                    Server
  │── TCP Open ──────────►│
  │── GET /index.html ──►│
  │◄── 200 OK ───────────│
  │── TCP Close ──────────│   ← প্রতিটি request এ নতুন TCP!
  │                       │
  │── TCP Open ──────────►│
  │── GET /style.css ───►│
  │◄── 200 OK ───────────│
  │── TCP Close ──────────│   ← আবার নতুন TCP! ❌ অদক্ষ
  │                       │

HTTP/1.1 (Keep-Alive সহ):
──────────────────────────
Client                    Server
  │── TCP Open ──────────►│   ← একবার TCP খোলো
  │── GET /index.html ──►│
  │◄── 200 OK ───────────│
  │── GET /style.css ───►│   ← একই TCP ব্যবহার! ✅
  │◄── 200 OK ───────────│
  │── GET /app.js ──────►│   ← একই TCP! ✅
  │◄── 200 OK ───────────│
  │── TCP Close ──────────│   ← সব শেষে বন্ধ
```

### Head-of-Line (HOL) Blocking সমস্যা

```
HTTP/1.1 এ একটি slow response পুরো connection block করে:

Client                              Server
  │── GET /heavy-image.jpg ──────►│
  │          (অপেক্ষা...)            │   ← ৫ সেকেন্ড লাগছে!
  │── GET /tiny-css.css ──────────►│   ← এটা ১ms এ হতো
  │          (ব্লক! ⏳)              │      কিন্তু image এর
  │◄── 200 (image) ────────────────│      জন্য অপেক্ষা করছে
  │◄── 200 (css) ──────────────────│   ← এখন পেল! ❌ দেরি

সমাধান (HTTP/1.1 workaround):
──────────────────────────────
ব্রাউজার একই সার্ভারে ৬টি parallel TCP connection খোলে:

  Connection 1: GET /image1.jpg ─────►
  Connection 2: GET /image2.jpg ─────►
  Connection 3: GET /style.css ──────►
  Connection 4: GET /app.js ─────────►
  Connection 5: GET /font.woff ──────►
  Connection 6: GET /favicon.ico ────►

  ❌ সমস্যা: প্রতিটি connection এ TCP handshake + TLS handshake দরকার
```

---

## 📌 HTTP/2

HTTP/2 (২০১৫) HTTP/1.1 এর সমস্যা সমাধান করে। এটি **binary**, **multiplexed**, এবং **compressed**।

```
┌─────────────────────────────────────────────────────────┐
│                    HTTP/2 বৈশিষ্ট্য                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Binary Protocol (টেক্সট নয়, দ্রুত parsing)           │
│  ✅ Multiplexing (একই TCP তে parallel requests)          │
│  ✅ Header Compression (HPACK)                           │
│  ✅ Server Push (ক্লায়েন্ট চাওয়ার আগেই পাঠানো)           │
│  ✅ Stream Priority (গুরুত্বপূর্ণ request আগে)             │
│  ✅ Single TCP Connection (per origin)                   │
│  ❌ TCP HOL Blocking (TCP লেভেলে এখনো আছে)              │
│  ❌ TCP+TLS Handshake Latency                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Multiplexing (HTTP/2 এর মূল সুবিধা)

```
HTTP/1.1 (Sequential):
───────────────────────
      TCP Connection 1        TCP Connection 2
  ┌────────────────────┐  ┌────────────────────┐
  │ Req1 ──► Res1      │  │ Req3 ──► Res3      │
  │ Req2 ──► Res2      │  │ Req4 ──► Res4      │
  └────────────────────┘  └────────────────────┘
  ❌ ৬টা পর্যন্ত connection, HOL blocking

HTTP/2 (Multiplexed):
─────────────────────
      একটি TCP Connection
  ┌──────────────────────────────────────────┐
  │ Stream 1: ═══▶ Req1 ◀══════ Res1        │
  │ Stream 2: ══════▶ Req2 ◀═══ Res2        │
  │ Stream 3: ═▶ Req3 ◀════════════ Res3    │
  │ Stream 4: ════════▶ Req4 ◀══ Res4       │
  └──────────────────────────────────────────┘
  ✅ একটি connection, parallel streams, কোনো HOL blocking নেই

Daraz পেজ লোড তুলনা:
────────────────────
HTTP/1.1: index.html → style.css → app.js → image1 → image2...
          (একটার পর একটা বা ৬টা connection দিয়ে parallel)

HTTP/2:   index.html ─┐
          style.css  ──┤
          app.js    ───┤ সব একসাথে, একটি connection!
          image1   ────┤
          image2   ────┘
```

### Header Compression (HPACK)

```
HTTP/1.1 এ প্রতিটি request এ পুরো header পাঠাতে হয়:

Request 1:                      Request 2:
Host: www.daraz.com.bd         Host: www.daraz.com.bd      ← একই!
User-Agent: Mozilla/5.0...     User-Agent: Mozilla/5.0...  ← একই!
Accept: text/html              Accept: image/jpeg          ← ভিন্ন
Cookie: session=abc123...      Cookie: session=abc123...   ← একই!

❌ প্রতিবার ~800 bytes header পাঠানো

HTTP/2 HPACK:
Request 1: পুরো header পাঠায় → হেডার টেবিলে সংরক্ষণ
Request 2: শুধু পরিবর্তিত field পাঠায়
           Accept: image/jpeg  ← শুধু এটুকু!

✅ ~90% header ডেটা সাশ্রয়
```

### Server Push

```
HTTP/1.1 (Server Push ছাড়া):
──────────────────────────────
Client                         Server
  │── GET /index.html ──────►│
  │◄── 200 (HTML) ───────────│  ← ক্লায়েন্ট HTML parse করে
  │── GET /style.css ────────►│  ← তারপর CSS চায়
  │◄── 200 (CSS) ────────────│  ← আরো দেরি!
  │── GET /app.js ───────────►│  ← তারপর JS চায়
  │◄── 200 (JS) ─────────────│

  মোট: ৩ round trips

HTTP/2 (Server Push সহ):
─────────────────────────
Client                         Server
  │── GET /index.html ──────►│
  │◄── 200 (HTML) ───────────│
  │◄── PUSH /style.css ──────│  ← সার্ভার নিজে থেকে পাঠায়!
  │◄── PUSH /app.js ─────────│  ← ক্লায়েন্ট চাওয়ার আগেই!

  মোট: ১ round trip! ✅
```

---

## 📌 HTTP/3 (QUIC)

HTTP/3 (২০২২) **QUIC** প্রোটোকলের উপর ভিত্তি করে যা **UDP** ব্যবহার করে।

```
┌─────────────────────────────────────────────────────────┐
│                    HTTP/3 বৈশিষ্ট্য                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ QUIC Protocol (UDP-based, Google তৈরি)               │
│  ✅ 0-RTT Connection (আগের সংযোগ মনে রাখে)              │
│  ✅ No HOL Blocking (TCP লেভেলেও না!)                    │
│  ✅ Built-in Encryption (TLS 1.3 অন্তর্ভুক্ত)             │
│  ✅ Connection Migration (Wi-Fi → 4G seamless)           │
│  ✅ Independent Streams                                  │
│  ❌ UDP-based (কিছু ফায়ারওয়াল ব্লক করতে পারে)            │
│  ❌ নতুন, সব জায়গায় সাপোর্ট নেই                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Protocol Stack তুলনা

```
HTTP/1.1 & HTTP/2:           HTTP/3:
┌──────────────┐             ┌──────────────┐
│    HTTP      │             │    HTTP      │
├──────────────┤             ├──────────────┤
│    TLS 1.2   │             │    QUIC      │
│    or 1.3    │             │  (TLS 1.3    │
├──────────────┤             │   built-in)  │
│    TCP       │             ├──────────────┤
├──────────────┤             │    UDP       │
│    IP        │             ├──────────────┤
└──────────────┘             │    IP        │
                             └──────────────┘
```

### 0-RTT Connection Setup

```
HTTP/1.1 (TCP + TLS 1.2):
──────────────────────────
Client                         Server
  │── SYN ─────────────────►│  ← TCP Handshake
  │◄── SYN-ACK ─────────────│     (1 RTT)
  │── ACK ─────────────────►│
  │── ClientHello ──────────►│  ← TLS Handshake
  │◄── ServerHello ──────────│     (2 RTT)
  │── Finished ─────────────►│
  │◄── Finished ─────────────│
  │── HTTP Request ─────────►│  ← এখন ডেটা পাঠাতে পারে
  │                          │
  মোট: ৩ RTT (TCP 1 + TLS 2)

HTTP/2 (TCP + TLS 1.3):
─────────────────────────
Client                         Server
  │── SYN ─────────────────►│  ← TCP Handshake
  │◄── SYN-ACK ─────────────│     (1 RTT)
  │── ACK + ClientHello ───►│  ← TLS 1.3
  │◄── ServerHello ──────────│     (1 RTT)
  │── HTTP Request ─────────►│
  │                          │
  মোট: ২ RTT (TCP 1 + TLS 1)

HTTP/3 (QUIC - প্রথমবার):
──────────────────────────
Client                         Server
  │── QUIC Initial ─────────►│  ← QUIC + TLS একসাথে!
  │◄── QUIC Handshake ───────│     (1 RTT)
  │── HTTP Request ─────────►│
  │                          │
  মোট: ১ RTT!

HTTP/3 (QUIC - পুনঃসংযোগ):
────────────────────────────
Client                         Server
  │── QUIC 0-RTT + Data ───►│  ← আগের session key মনে আছে!
  │◄── Response ─────────────│
  │                          │
  মোট: ০ RTT!! 🚀
```

### Connection Migration

```
Pathao রাইডার অ্যাপ ব্যবহার করছে:

HTTP/2 (TCP-based):
──────────────────
রাইডার Wi-Fi zone এ ঢুকলো → 4G থেকে Wi-Fi তে switch
  ┌────────┐                    ┌────────┐
  │ 4G     │ ──TCP broken!──── │ Server │
  │ IP:    │ ❌ Connection Lost │        │
  │10.0.0.1│                    │        │
  └────────┘                    │        │
  ┌────────┐                    │        │
  │ Wi-Fi  │ ──New TCP+TLS──► │        │
  │ IP:    │    (3 RTT!) 😞    │        │
  │192.168.│                    └────────┘
  │1.100   │

HTTP/3 (QUIC):
──────────────
রাইডার Wi-Fi zone এ ঢুকলো → seamless switch!
  ┌────────┐                    ┌────────┐
  │ 4G     │ ─QUIC──────────── │ Server │
  │ IP:    │   Connection ID    │        │
  │10.0.0.1│   maintained!      │        │
  └────────┘                    │        │
  ┌────────┐                    │        │
  │ Wi-Fi  │ ─QUIC──────────── │        │
  │ IP:    │   Same Conn ID! ✅ │        │
  │192.168.│   ০ RTT! 🚀        └────────┘
  │1.100   │

  QUIC Connection ID ≠ IP+Port, তাই নেটওয়ার্ক বদলালেও connection থাকে!
```

---

## 📊 HTTP Version তুলনা

```
┌────────────────────┬──────────┬──────────┬──────────────┐
│     বৈশিষ্ট্য       │ HTTP/1.1 │ HTTP/2   │   HTTP/3     │
├────────────────────┼──────────┼──────────┼──────────────┤
│ সাল               │  ১৯৯৭    │  ২০১৫    │    ২০২২      │
│ Transport         │  TCP     │  TCP     │  QUIC (UDP)  │
│ Format            │  Text   │  Binary  │  Binary      │
│ Multiplexing      │  ❌      │  ✅      │  ✅          │
│ Header Compress.  │  ❌      │  HPACK  │  QPACK       │
│ Server Push       │  ❌      │  ✅      │  ✅          │
│ HOL Blocking      │  ✅ আছে  │  TCP তে  │  ❌ নেই!     │
│                    │          │  আছে    │              │
│ TLS               │  Optional│  De facto│  Built-in    │
│ Connection Setup  │  3 RTT  │  2 RTT  │  0-1 RTT     │
│ Conn. Migration   │  ❌      │  ❌      │  ✅          │
│ Streams           │  ❌      │  TCP-   │  Independent │
│                    │          │  coupled │              │
└────────────────────┴──────────┴──────────┴──────────────┘
```

### বাংলাদেশী ওয়েবসাইটে ব্যবহার

```
┌──────────────────────┬──────────┬─────────────────────┐
│ ওয়েবসাইট              │ HTTP Ver │ কারণ                 │
├──────────────────────┼──────────┼─────────────────────┤
│ daraz.com.bd         │ HTTP/2   │ দ্রুত পেজ লোড, CDN    │
│ bkash.com            │ HTTP/2   │ TLS বাধ্যতামূলক       │
│ prothomalo.com       │ HTTP/2   │ মিডিয়া-ভারী সাইট     │
│ robi.com.bd          │ HTTP/1.1 │ লিগ্যাসি সার্ভার       │
│ google.com.bd        │ HTTP/3   │ Google QUIC pioneer  │
│ youtube.com          │ HTTP/3   │ ভিডিও স্ট্রিমিং       │
│ facebook.com         │ HTTP/3   │ কম latency দরকার     │
└──────────────────────┴──────────┴─────────────────────┘
```

---

## 💻 কোড উদাহরণ

### PHP cURL

```php
<?php
// =============================================
// PHP cURL - HTTP/1.1, HTTP/2 রিকোয়েস্ট
// =============================================

// --- GET Request ---
function getProducts(): array
{
    $ch = curl_init();
    
    curl_setopt_array($ch, [
        CURLOPT_URL            => 'https://api.daraz.com.bd/products?page=1&limit=20',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => [
            'Accept: application/json',
            'Accept-Language: bn-BD',
            'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...',
        ],
        CURLOPT_HTTP_VERSION   => CURL_HTTP_VERSION_2_0, // HTTP/2 ব্যবহার
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_TIMEOUT        => 30,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $httpVersion = curl_getinfo($ch, CURLINFO_HTTP_VERSION);
    
    echo "HTTP Version: " . ($httpVersion == 2 ? 'HTTP/2' : 'HTTP/1.1') . "\n";
    echo "Status Code: $httpCode\n";
    
    curl_close($ch);
    return json_decode($response, true);
}

// --- POST Request ---
function createOrder(array $orderData): array
{
    $ch = curl_init();
    
    curl_setopt_array($ch, [
        CURLOPT_URL            => 'https://api.daraz.com.bd/orders',
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($orderData),
        CURLOPT_HTTPHEADER     => [
            'Content-Type: application/json',
            'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...',
            'X-Idempotency-Key: order-' . uniqid(), // ডুপ্লিকেট প্রতিরোধ
        ],
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    
    if ($httpCode === 201) {
        echo "অর্ডার সফলভাবে তৈরি হয়েছে! ✅\n";
    } elseif ($httpCode === 422) {
        echo "ভ্যালিডেশন ব্যর্থ! ❌\n";
    }
    
    curl_close($ch);
    return json_decode($response, true);
}

// --- Parallel Requests (cURL Multi) ---
function getMultipleProducts(array $productIds): array
{
    $multiHandle = curl_multi_init();
    $handles = [];
    
    foreach ($productIds as $id) {
        $ch = curl_init("https://api.daraz.com.bd/products/$id");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_multi_add_handle($multiHandle, $ch);
        $handles[$id] = $ch;
    }
    
    // সব রিকোয়েস্ট parallel এ চালানো (HTTP/1.1 তেও!)
    $running = 0;
    do {
        curl_multi_exec($multiHandle, $running);
        curl_multi_select($multiHandle);
    } while ($running > 0);
    
    $results = [];
    foreach ($handles as $id => $ch) {
        $results[$id] = json_decode(curl_multi_getcontent($ch), true);
        curl_multi_remove_handle($multiHandle, $ch);
        curl_close($ch);
    }
    
    curl_multi_close($multiHandle);
    return $results;
}

// ব্যবহার
$products = getProducts();
$order = createOrder([
    'product_id' => 123,
    'quantity' => 1,
    'address' => 'ঢাকা, বাংলাদেশ'
]);
```

### JavaScript fetch API

```javascript
// =============================================
// JavaScript fetch - আধুনিক HTTP Client
// =============================================

// --- GET Request ---
async function getProducts() {
    try {
        const response = await fetch('https://api.daraz.com.bd/products?page=1&limit=20', {
            method: 'GET',
            headers: {
                'Accept': 'application/json',
                'Accept-Language': 'bn-BD',
                'Authorization': 'Bearer eyJhbGciOiJIUzI1NiJ9...'
            }
        });
        
        console.log(`Status: ${response.status} ${response.statusText}`);
        console.log(`Content-Type: ${response.headers.get('content-type')}`);
        
        if (!response.ok) {
            throw new Error(`HTTP Error: ${response.status}`);
        }
        
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('রিকোয়েস্ট ব্যর্থ:', error.message);
    }
}

// --- POST Request ---
async function createOrder(orderData) {
    const response = await fetch('https://api.daraz.com.bd/orders', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer eyJhbGciOiJIUzI1NiJ9...',
            'X-Idempotency-Key': `order-${Date.now()}`
        },
        body: JSON.stringify(orderData)
    });
    
    switch (response.status) {
        case 201:
            console.log('অর্ডার সফল! ✅');
            return await response.json();
        case 401:
            console.log('লগইন প্রয়োজন! 🔒');
            break;
        case 422:
            const errors = await response.json();
            console.log('ভ্যালিডেশন ব্যর্থ:', errors);
            break;
        case 429:
            const retryAfter = response.headers.get('Retry-After');
            console.log(`Rate limited! ${retryAfter}s পর চেষ্টা করুন`);
            break;
        case 500:
            console.log('সার্ভার সমস্যা! 💥');
            break;
    }
}

// --- Parallel Requests ---
async function getMultipleProducts(productIds) {
    const requests = productIds.map(id =>
        fetch(`https://api.daraz.com.bd/products/${id}`)
            .then(res => res.json())
    );
    
    // Promise.all — সব একসাথে!
    const products = await Promise.all(requests);
    return products;
}

// --- Streaming Response (HTTP/2 friendly) ---
async function streamLargeData() {
    const response = await fetch('https://api.daraz.com.bd/export/orders');
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        console.log('Chunk received:', chunk.length, 'bytes');
    }
}

// --- Abort Controller (Timeout) ---
async function fetchWithTimeout(url, timeoutMs = 5000) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);
    
    try {
        const response = await fetch(url, {
            signal: controller.signal
        });
        clearTimeout(timeout);
        return await response.json();
    } catch (error) {
        if (error.name === 'AbortError') {
            console.log(`${timeoutMs}ms timeout! সার্ভার সাড়া দেয়নি`);
        }
        throw error;
    }
}
```

### Node.js HTTP/2 Client

```javascript
// =============================================
// Node.js HTTP/2 ক্লায়েন্ট
// =============================================
const http2 = require('http2');

function makeHTTP2Request(url) {
    return new Promise((resolve, reject) => {
        const client = http2.connect(url);
        
        const req = client.request({
            ':method': 'GET',
            ':path': '/products',
            'accept': 'application/json'
        });
        
        let data = '';
        
        req.on('response', (headers) => {
            console.log('Status:', headers[':status']);
            console.log('Content-Type:', headers['content-type']);
        });
        
        req.on('data', (chunk) => {
            data += chunk;
        });
        
        req.on('end', () => {
            client.close();
            resolve(JSON.parse(data));
        });
        
        req.on('error', reject);
        req.end();
    });
}

// HTTP/2 Server Push handling
const client = http2.connect('https://www.daraz.com.bd');

client.on('stream', (pushedStream, requestHeaders) => {
    console.log('Server Push received:', requestHeaders[':path']);
    
    let pushData = '';
    pushedStream.on('data', (chunk) => {
        pushData += chunk;
    });
    pushedStream.on('end', () => {
        console.log('Pushed resource size:', pushData.length);
    });
});
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: HTTP/1.1 এ Domain Sharding এখনো ব্যবহার করা
   (HTTP/2 তে একটি connection ই যথেষ্ট)

✅ সঠিক: HTTP/2 enable করুন, multiplexing এর সুবিধা নিন

──────────────────────────────────────────────

❌ ভুল: প্রতিটি API call এ পুরো অবজেক্ট রিটার্ন করা
   GET /users/123 → { id, name, email, address, orders, ... }

✅ সঠিক: স্পার্স ফিল্ড ব্যবহার করুন
   GET /users/123?fields=id,name,email

──────────────────────────────────────────────

❌ ভুল: Cache-Control হেডার ব্যবহার না করা
   → প্রতিবার সার্ভারে যাওয়া, ধীর

✅ সঠিক: Cache-Control: public, max-age=3600
   → স্ট্যাটিক রিসোর্স ক্যাশ হবে, দ্রুত!

──────────────────────────────────────────────

❌ ভুল: Status code সঠিকভাবে ব্যবহার না করা
   POST /orders → 200 OK { error: "validation failed" }

✅ সঠিক: সঠিক status code ব্যবহার করুন
   POST /orders → 422 Unprocessable Entity { errors: [...] }
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

```
HTTP/1.1 ব্যবহার করুন:
✅ সিম্পল অ্যাপ্লিকেশন (ছোট ওয়েবসাইট)
✅ লিগ্যাসি সার্ভার যা HTTP/2 সাপোর্ট করে না
✅ Debugging (text-based, পড়তে সুবিধা)

HTTP/2 ব্যবহার করুন:
✅ বেশিরভাগ ওয়েবসাইট ও API
✅ মিডিয়া-ভারী সাইট (Daraz, Prothom Alo)
✅ SPA (Single Page Application)
✅ মাইক্রোসার্ভিস (gRPC HTTP/2 ব্যবহার করে)

HTTP/3 ব্যবহার করুন:
✅ মোবাইল অ্যাপ (নেটওয়ার্ক স্যুইচিং common)
✅ উচ্চ latency নেটওয়ার্ক (বাংলাদেশের গ্রামীণ এলাকা)
✅ রিয়েল-টাইম অ্যাপ (Pathao, gaming)
✅ গ্লোবাল CDN (Cloudflare, Google)
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│              HTTP প্রোটোকল সারসংক্ষেপ                  │
├───────────────────────────────────────────────────────┤
│                                                       │
│  HTTP/1.1: টেক্সট-ভিত্তিক, Keep-Alive, HOL Blocking │
│  HTTP/2:   বাইনারি, Multiplexing, Server Push         │
│  HTTP/3:   QUIC/UDP, 0-RTT, Connection Migration     │
│                                                       │
│  গুরুত্বপূর্ণ বিষয়:                                     │
│  - সঠিক Method ব্যবহার করুন (GET/POST/PUT/DELETE)     │
│  - সঠিক Status Code রিটার্ন করুন                      │
│  - Cache-Control হেডার ব্যবহার করুন                    │
│  - HTTPS (TLS) সবসময় ব্যবহার করুন                     │
│  - HTTP/2 enable করুন (Nginx/Apache তে সহজ)          │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [WebSockets](./websockets.md) — রিয়েল-টাইম কমিউনিকেশন
