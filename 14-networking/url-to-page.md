# 🌐 ব্রাউজারে URL টাইপ করলে কী হয়? (URL to Rendered Page)

> এটি সিস্টেম ডিজাইন ইন্টারভিউয়ের সবচেয়ে জনপ্রিয় প্রশ্নগুলোর একটি।
> আপনি যখন ব্রাউজারে `www.daraz.com.bd` টাইপ করে Enter চাপেন, পর্দায় পেজ দেখা পর্যন্ত
> পেছনে একটি জটিল ও সুন্দর প্রক্রিয়া চলে। আসুন প্রতিটি ধাপ গভীরভাবে বুঝি।

---

## 📖 সূচিপত্র

- [Overview: সম্পূর্ণ যাত্রা](#-overview-সম্পূর্ণ-যাত্রা)
- [Step 1: URL পার্সিং](#-step-1-url-পার্সিং)
- [Step 2: DNS রেজোল্যুশন](#-step-2-dns-রেজোল্যুশন)
- [Step 3: TCP কানেকশন](#-step-3-tcp-কানেকশন)
- [Step 4: TLS হ্যান্ডশেক](#-step-4-tls-হ্যান্ডশেক)
- [Step 5: HTTP রিকোয়েস্ট](#-step-5-http-রিকোয়েস্ট)
- [Step 6: সার্ভার প্রসেসিং](#-step-6-সার্ভার-প্রসেসিং)
- [Step 7: HTTP রেসপন্স](#-step-7-http-রেসপন্স)
- [Step 8: ব্রাউজার রেন্ডারিং](#-step-8-ব্রাউজার-রেন্ডারিং)
- [সম্পূর্ণ ফ্লো ডায়াগ্রাম](#-সম্পূর্ণ-ফ্লো-ডায়াগ্রাম)
- [PHP ও JavaScript কোড উদাহরণ](#-php-ও-javascript-কোড-উদাহরণ)
- [বাংলাদেশী উদাহরণ](#-বাংলাদেশী-উদাহরণ)
- [ইন্টারভিউ টিপস](#-ইন্টারভিউ-টিপস)

---

## 📌 Overview: সম্পূর্ণ যাত্রা

আপনি ব্রাউজারে `https://www.daraz.com.bd/products/iphone-15` টাইপ করলেন। এখন Enter চাপা থেকে পেজ দেখা পর্যন্ত ৮টি প্রধান ধাপ ঘটে:

```
┌─────────────────────────────────────────────────────────────────────┐
│              URL থেকে রেন্ডার পেজ — ৮টি ধাপ                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ① URL পার্সিং        ─── ব্রাউজার URL ভাঙে ও বোঝে                  │
│         │                                                           │
│         ▼                                                           │
│  ② DNS রেজোল্যুশন     ─── ডোমেইন নাম → IP Address                   │
│         │                                                           │
│         ▼                                                           │
│  ③ TCP কানেকশন        ─── 3-Way Handshake (SYN → SYN-ACK → ACK)    │
│         │                                                           │
│         ▼                                                           │
│  ④ TLS হ্যান্ডশেক      ─── HTTPS হলে সুরক্ষিত সংযোগ স্থাপন          │
│         │                                                           │
│         ▼                                                           │
│  ⑤ HTTP রিকোয়েস্ট     ─── GET /products/iphone-15 পাঠানো           │
│         │                                                           │
│         ▼                                                           │
│  ⑥ সার্ভার প্রসেসিং    ─── Load Balancer → App → DB → Response      │
│         │                                                           │
│         ▼                                                           │
│  ⑦ HTTP রেসপন্স       ─── 200 OK + HTML/CSS/JS পাঠানো               │
│         │                                                           │
│         ▼                                                           │
│  ⑧ ব্রাউজার রেন্ডারিং  ─── DOM → CSSOM → Render Tree → Paint       │
│                                                                     │
│  ⏱️ মোট সময়: ~100ms - 2s (নেটওয়ার্ক ও সার্ভারের উপর নির্ভর)       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### সময়ের হিসাব (আনুমানিক)

| ধাপ | কাজ | আনুমানিক সময় |
|-----|------|---------------|
| ① | URL পার্সিং | < 1ms |
| ② | DNS Lookup | 1ms - 100ms (ক্যাশ থাকলে 0ms) |
| ③ | TCP Handshake | 10ms - 50ms |
| ④ | TLS Handshake | 30ms - 100ms |
| ⑤ | HTTP Request পাঠানো | 1ms - 5ms |
| ⑥ | সার্ভার প্রসেসিং | 50ms - 500ms |
| ⑦ | HTTP Response আসা | 10ms - 100ms |
| ⑧ | ব্রাউজার রেন্ডারিং | 50ms - 500ms |

---

## 🔗 Step 1: URL পার্সিং

ব্রাউজার প্রথমে আপনার টাইপ করা URL কে বিভিন্ন অংশে ভাগ করে বিশ্লেষণ করে:

```
         https://www.daraz.com.bd:443/products/iphone-15?color=black&storage=128gb#reviews
         ──┬──   ───────┬────────  ─┬─ ──────┬────────── ──────────┬──────────── ───┬───
           │            │           │        │                     │                │
       Protocol      Domain       Port     Path                 Query           Fragment
      (প্রোটোকল)    (ডোমেইন)    (পোর্ট)   (পাথ)            (কোয়েরি স্ট্রিং)     (ফ্র্যাগমেন্ট)

      ┌──────────────────────────────────────────────────────────────────────────────┐
      │ অংশ              │ মান                    │ কাজ                              │
      ├──────────────────────────────────────────────────────────────────────────────┤
      │ Protocol (স্কিম)   │ https                 │ সুরক্ষিত HTTP ব্যবহার করো         │
      │ Domain (হোস্ট)     │ www.daraz.com.bd      │ কোন সার্ভারে যেতে হবে             │
      │ Port (পোর্ট)       │ 443 (HTTPS ডিফল্ট)    │ সার্ভারের কোন দরজায় যাবো         │
      │ Path (পাথ)         │ /products/iphone-15   │ কোন রিসোর্স চাই                  │
      │ Query String       │ color=black&...       │ ফিল্টার/সার্চ প্যারামিটার         │
      │ Fragment           │ #reviews              │ পেজের কোন অংশে যাবো (সার্ভারে যায় না) │
      └──────────────────────────────────────────────────────────────────────────────┘
```

### ব্রাউজার প্রথমে যা চেক করে

```
  ইউজার টাইপ করলো: "daraz.com.bd"
         │
         ▼
  ┌───────────────────────┐
  │ এটা কি URL নাকি        │
  │ সার্চ কোয়েরি?          │──── "iphone 15 price" → Google সার্চে পাঠাও
  └──────────┬────────────┘
             │ URL
             ▼
  ┌───────────────────────┐
  │ প্রোটোকল আছে?         │
  │ (http:// বা https://)  │──── না থাকলে: https:// যোগ করো
  └──────────┬────────────┘
             │
             ▼
  ┌───────────────────────┐
  │ HSTS লিস্টে আছে?      │
  │ (HTTP Strict Transport │──── থাকলে: http → https এ রূপান্তর
  │  Security)             │
  └──────────┬────────────┘
             │
             ▼
  ┌───────────────────────┐
  │ URL Encode করো         │
  │ (স্পেশাল ক্যারেক্টার    │──── স্পেস → %20, বাংলা → UTF-8 encode
  │  এনকোড)                │
  └────────────────────────┘
```

### PHP তে URL পার্সিং

```php
<?php
// URL পার্স করা
$url = "https://www.daraz.com.bd/products/iphone-15?color=black&storage=128gb#reviews";
$parsed = parse_url($url);

echo "Protocol: " . $parsed['scheme'] . "\n";   // https
echo "Host: " . $parsed['host'] . "\n";          // www.daraz.com.bd
echo "Port: " . ($parsed['port'] ?? 443) . "\n"; // 443
echo "Path: " . $parsed['path'] . "\n";          // /products/iphone-15
echo "Query: " . $parsed['query'] . "\n";        // color=black&storage=128gb
echo "Fragment: " . $parsed['fragment'] . "\n";  // reviews

// কোয়েরি স্ট্রিং পার্স
parse_str($parsed['query'], $queryParams);
echo "Color: " . $queryParams['color'] . "\n";   // black
echo "Storage: " . $queryParams['storage'] . "\n"; // 128gb
```

### JavaScript/Node.js তে URL পার্সিং

```javascript
// URL পার্স করা
const url = new URL("https://www.daraz.com.bd/products/iphone-15?color=black&storage=128gb#reviews");

console.log("Protocol:", url.protocol);   // https:
console.log("Host:", url.hostname);       // www.daraz.com.bd
console.log("Port:", url.port || 443);    // 443
console.log("Path:", url.pathname);       // /products/iphone-15
console.log("Search:", url.search);       // ?color=black&storage=128gb
console.log("Hash:", url.hash);           // #reviews

// কোয়েরি প্যারামিটার পার্স
console.log("Color:", url.searchParams.get("color"));     // black
console.log("Storage:", url.searchParams.get("storage")); // 128gb
```

---

## 🔗 Step 2: DNS রেজোল্যুশন

ব্রাউজার `www.daraz.com.bd` ডোমেইন নামকে IP Address এ রূপান্তর করতে DNS Resolution শুরু করে। এটি একটি ক্যাসকেডিং ক্যাশ লুকআপ প্রক্রিয়া:

```
  ব্রাউজার "www.daraz.com.bd" এর IP কোথায়?

  ┌─────────────────────────────────────────────────────────────────────┐
  │  ① ব্রাউজার ক্যাশ        ─── Chrome এর নিজস্ব DNS ক্যাশ চেক করো     │
  │     │ ❌ নেই                                                        │
  │     ▼                                                               │
  │  ② OS ক্যাশ              ─── Windows/Mac/Linux এর DNS ক্যাশ চেক     │
  │     │ ❌ নেই                 (hosts ফাইলও দেখে)                      │
  │     ▼                                                               │
  │  ③ Router ক্যাশ           ─── WiFi Router এর ক্যাশে আছে?           │
  │     │ ❌ নেই                                                        │
  │     ▼                                                               │
  │  ④ ISP DNS সার্ভার        ─── BTCL/GP/Robi এর DNS সার্ভার           │
  │     │ ❌ নেই                                                        │
  │     ▼                                                               │
  │  ⑤ Recursive DNS Query শুরু                                        │
  │     │                                                               │
  │     ├──► Root DNS Server (.)                                        │
  │     │      "bd এর TLD সার্ভার 203.112.2.4 তে যাও"                    │
  │     │                                                               │
  │     ├──► TLD DNS Server (.bd)                                       │
  │     │      ".com.bd এর NS সার্ভার 77.68.67.18 তে যাও"               │
  │     │                                                               │
  │     ├──► SLD DNS Server (.com.bd)                                   │
  │     │      "daraz.com.bd এর NS সার্ভার ns1.daraz.com তে যাও"        │
  │     │                                                               │
  │     └──► Authoritative DNS Server (daraz.com.bd)                    │
  │            "www.daraz.com.bd = 103.108.12.45" ✅                     │
  │                                                                     │
  │  ফলাফল: IP 103.108.12.45 পাওয়া গেলো!                               │
  │  এখন এই IP ক্যাশে রাখো (TTL অনুযায়ী)                                │
  └─────────────────────────────────────────────────────────────────────┘
```

### DNS Resolution এর বিস্তারিত ফ্লো

```
  ব্রাউজার                ISP DNS             Root DNS        .bd TLD         Authoritative
  (Chrome)              (GP/BTCL)            (13টি আছে)      (BTCL)         (Daraz NS)
     │                     │                     │               │               │
     │─ "www.daraz.com.bd  │                     │               │               │
     │   এর IP দাও" ──────►│                     │               │               │
     │                     │─ ".bd কোথায়?" ─────►│               │               │
     │                     │                     │               │               │
     │                     │◄─ "203.112.2.4      │               │               │
     │                     │   তে যাও" ──────────│               │               │
     │                     │                     │               │               │
     │                     │─ "daraz.com.bd      │               │               │
     │                     │   কোথায়?" ─────────────────────────►│               │
     │                     │                     │               │               │
     │                     │◄─ "ns1.daraz.com    │               │               │
     │                     │   তে যাও" ─────────────────────────│               │
     │                     │                     │               │               │
     │                     │─ "www.daraz.com.bd  │               │               │
     │                     │   এর A record?" ──────────────────────────────────►│
     │                     │                     │               │               │
     │                     │◄─ "103.108.12.45" ◄────────────────────────────────│
     │                     │                     │               │               │
     │◄─ "103.108.12.45" ──│                     │               │               │
     │                     │                     │               │               │
```

### DNS ক্যাশ TTL উদাহরণ

| ক্যাশ লেভেল | TTL (সাধারণত) | উদাহরণ |
|-------------|---------------|--------|
| ব্রাউজার ক্যাশ | 60 সেকেন্ড | Chrome 60s ক্যাশ রাখে |
| OS ক্যাশ | পরিবর্তনশীল | DNS record এর TTL অনুযায়ী |
| Router ক্যাশ | 15 মিনিট - 1 ঘণ্টা | Router নির্মাতার সেটিং |
| ISP DNS ক্যাশ | TTL অনুযায়ী | daraz.com.bd TTL=300 (5 মিনিট) |

---

## 🔗 Step 3: TCP কানেকশন

IP Address পাওয়ার পর ব্রাউজার সার্ভারের সাথে TCP সংযোগ স্থাপন করে। এটি **Three-Way Handshake** এর মাধ্যমে হয়:

```
  ব্রাউজার (Client)                              সার্ভার (Daraz - 103.108.12.45:443)
       │                                                    │
       │                                                    │
       │  ①  SYN (seq=100)                                  │
       │  "হ্যালো! আমি সংযোগ করতে চাই"                       │
       │──────────────────────────────────────────────────►  │
       │                                                    │
       │  ②  SYN-ACK (seq=300, ack=101)                     │
       │  "হ্যালো! ঠিক আছে, আমিও রাজি"                      │
       │  ◄──────────────────────────────────────────────────│
       │                                                    │
       │  ③  ACK (ack=301)                                  │
       │  "ধন্যবাদ! সংযোগ প্রতিষ্ঠিত"                       │
       │──────────────────────────────────────────────────►  │
       │                                                    │
       │  ✅ TCP সংযোগ প্রতিষ্ঠিত!                            │
       │  এখন ডেটা আদান-প্রদান শুরু হতে পারে                  │
       │                                                    │
```

### কেন Three-Way Handshake?

```
┌─────────────────────────────────────────────────────────────┐
│              Three-Way Handshake কেন দরকার?                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣  উভয় পক্ষ পাঠাতে ও গ্রহণ করতে পারে তা নিশ্চিত করা     │
│                                                             │
│  ক্লায়েন্ট → SYN → সার্ভার                                   │
│    "আমি পাঠাতে পারি, তুমি কি পাচ্ছ?"                         │
│                                                             │
│  সার্ভার → SYN-ACK → ক্লায়েন্ট                               │
│    "হ্যাঁ পেয়েছি! আমিও পাঠাতে পারি, তুমি কি পাচ্ছ?"         │
│                                                             │
│  ক্লায়েন্ট → ACK → সার্ভার                                   │
│    "হ্যাঁ পেয়েছি! চলো শুরু করি!"                              │
│                                                             │
│  2️⃣  Sequence Number সিঙ্ক্রোনাইজ করা (ISN exchange)       │
│  3️⃣  দুর্ঘটনাজনিত পুরানো প্যাকেট থেকে রক্ষা               │
│                                                             │
│  বাস্তব উদাহরণ:                                              │
│  ফোনে কথা বলার মতো —                                        │
│  "হ্যালো?" → "হ্যালো, শুনতে পাচ্ছ?" → "হ্যাঁ, বলো"          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### TCP Socket তৈরি — PHP উদাহরণ

```php
<?php
// TCP সকেট দিয়ে সরাসরি কানেকশন (নিম্ন-স্তরের উদাহরণ)
$socket = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);

// Three-way handshake এখানেই ঘটে
$result = socket_connect($socket, '103.108.12.45', 443);

if ($result) {
    echo "TCP সংযোগ সফল! (3-way handshake সম্পন্ন)\n";
} else {
    echo "সংযোগ ব্যর্থ: " . socket_strerror(socket_last_error()) . "\n";
}

socket_close($socket);
```

### TCP Socket — Node.js উদাহরণ

```javascript
const net = require('net');

// TCP সকেট তৈরি ও কানেক্ট
const client = new net.Socket();

client.connect(443, '103.108.12.45', () => {
    console.log('TCP সংযোগ সফল! (3-way handshake সম্পন্ন)');
});

client.on('error', (err) => {
    console.log('সংযোগ ব্যর্থ:', err.message);
});
```

---

## 🔗 Step 4: TLS হ্যান্ডশেক

HTTPS ব্যবহার করলে TCP কানেকশনের পরে TLS হ্যান্ডশেক ঘটে। এটি এনক্রিপ্টেড যোগাযোগ স্থাপন করে:

```
  ব্রাউজার (Client)                                  সার্ভার (Daraz)
       │                                                    │
       │  ① ClientHello                                     │
       │  ├── TLS Version: 1.3                              │
       │  ├── Cipher Suites: [AES_256_GCM, CHACHA20...]     │
       │  ├── Random Number (Client Random)                 │
       │  └── SNI: www.daraz.com.bd                         │
       │──────────────────────────────────────────────────►  │
       │                                                    │
       │  ② ServerHello + Certificate + Key Share            │
       │  ├── TLS Version: 1.3                              │
       │  ├── Selected Cipher: AES_256_GCM_SHA384           │
       │  ├── Random Number (Server Random)                 │
       │  ├── Certificate (X.509)                           │
       │  └── Server Key Share (ECDHE)                      │
       │  ◄──────────────────────────────────────────────────│
       │                                                    │
       │  ③ Certificate যাচাই                                │
       │  ├── Certificate chain যাচাই (Root CA পর্যন্ত)       │
       │  ├── মেয়াদ উত্তীর্ণ হয়নি তো?                        │
       │  ├── ডোমেইন নাম মিলছে?                               │
       │  └── Revoked হয়নি তো? (OCSP/CRL)                   │
       │                                                    │
       │  ④ Client Key Share + Finished                     │
       │  ├── Client Key Share (ECDHE)                      │
       │  └── Finished (MAC verification)                   │
       │──────────────────────────────────────────────────►  │
       │                                                    │
       │  ⑤ Server Finished                                 │
       │  ◄──────────────────────────────────────────────────│
       │                                                    │
       │  🔐 Symmetric Session Key তৈরি হলো!                  │
       │  এখন সব ডেটা এনক্রিপ্টেড আদান-প্রদান হবে             │
       │                                                    │
```

### Certificate Chain (বিশ্বাসের শৃঙ্খল)

```
  ┌──────────────────────────┐
  │     Root CA              │ ◄── ব্রাউজারে আগে থেকেই বিশ্বাসযোগ্য
  │  (DigiCert, Let's        │     (OS/Browser এ প্রি-ইনস্টলড)
  │   Encrypt, Comodo)       │
  └────────────┬─────────────┘
               │ স্বাক্ষর করে
               ▼
  ┌──────────────────────────┐
  │   Intermediate CA        │ ◄── Root CA এটিকে বিশ্বস্ত ঘোষণা করেছে
  │  (R3 - Let's Encrypt)    │
  └────────────┬─────────────┘
               │ স্বাক্ষর করে
               ▼
  ┌──────────────────────────┐
  │   Server Certificate     │ ◄── www.daraz.com.bd এর সার্টিফিকেট
  │  (*.daraz.com.bd)        │
  │  Valid: 2024-01-01 to    │
  │         2025-01-01       │
  └──────────────────────────┘

  ব্রাউজার নিচ থেকে উপরে যাচাই করে:
  Server Cert → Intermediate CA → Root CA ✅
```

### TLS 1.2 vs TLS 1.3

| বিষয় | TLS 1.2 | TLS 1.3 |
|-------|---------|---------|
| Handshake Round Trips | 2 RTT | 1 RTT |
| 0-RTT Resumption | নেই | আছে |
| Cipher Suites | অনেক (কিছু দুর্বল) | মাত্র ৫টি (সব শক্তিশালী) |
| Forward Secrecy | ঐচ্ছিক | বাধ্যতামূলক |
| আনুমানিক সময় | 60-100ms | 30-50ms |

---

## 🔗 Step 5: HTTP রিকোয়েস্ট

TLS সংযোগ প্রতিষ্ঠিত হওয়ার পর ব্রাউজার HTTP রিকোয়েস্ট পাঠায়:

```
┌─────────────────────────────────────────────────────────────┐
│  GET /products/iphone-15?color=black HTTP/1.1              │
│                                                             │
│  Host: www.daraz.com.bd                                    │
│  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)    │
│              AppleWebKit/537.36 Chrome/120.0.0.0           │
│  Accept: text/html,application/xhtml+xml,application/xml  │
│  Accept-Language: bn-BD,bn;q=0.9,en-US;q=0.8,en;q=0.7    │
│  Accept-Encoding: gzip, deflate, br                        │
│  Connection: keep-alive                                    │
│  Cookie: session_id=abc123def456;                          │
│          _ga=GA1.2.123456789.1234567890;                   │
│          preferred_lang=bn                                 │
│  Cache-Control: max-age=0                                  │
│  Sec-Fetch-Dest: document                                  │
│  Sec-Fetch-Mode: navigate                                  │
│  Sec-Fetch-Site: none                                      │
│  Sec-Fetch-User: ?1                                        │
│                                                             │
│  (Body: খালি — GET রিকোয়েস্টে Body থাকে না)                │
└─────────────────────────────────────────────────────────────┘
```

### গুরুত্বপূর্ণ Headers

```
┌────────────────────────────────────────────────────────────────────┐
│  Header                │ কাজ                                       │
├────────────────────────────────────────────────────────────────────┤
│  Host                  │ কোন ডোমেইনে রিকোয়েস্ট (virtual hosting)  │
│  User-Agent            │ ব্রাউজার ও OS তথ্য                        │
│  Accept                │ কোন ফরম্যাটে রেসপন্স চাই                  │
│  Accept-Language       │ bn-BD = বাংলা (বাংলাদেশ)                  │
│  Accept-Encoding       │ কম্প্রেশন সাপোর্ট (gzip, br)              │
│  Cookie                │ আগের সেশন তথ্য (লগইন ইত্যাদি)             │
│  Cache-Control         │ ক্যাশিং নির্দেশনা                          │
│  If-None-Match         │ ETag দিয়ে ক্যাশ যাচাই (304 পেতে)         │
│  If-Modified-Since     │ শেষ পরিবর্তনের সময় দিয়ে ক্যাশ যাচাই      │
└────────────────────────────────────────────────────────────────────┘
```

### ব্রাউজার কুকি পাঠানোর নিয়ম

```
  ব্রাউজারে সংরক্ষিত কুকি:
  ┌──────────────────────────────────────────────────┐
  │ Name: session_id                                 │
  │ Value: abc123def456                              │
  │ Domain: .daraz.com.bd                            │
  │ Path: /                                          │
  │ Secure: true (শুধু HTTPS এ পাঠাবে)                │
  │ HttpOnly: true (JavaScript অ্যাক্সেস নেই)        │
  │ SameSite: Lax                                    │
  │ Expires: 2025-12-31                              │
  └──────────────────────────────────────────────────┘

  ব্রাউজার চেক করে:
  ✅ Domain মিলছে? (.daraz.com.bd) → হ্যাঁ
  ✅ Path মিলছে? (/ → /products/...) → হ্যাঁ
  ✅ Secure এবং HTTPS? → হ্যাঁ
  ✅ মেয়াদ শেষ হয়নি? → হ্যাঁ
  → কুকি রিকোয়েস্টের সাথে পাঠাও ✅
```

---

## 🔗 Step 6: সার্ভার প্রসেসিং

রিকোয়েস্ট সার্ভারে পৌঁছানোর পর একটি জটিল প্রক্রিয়া শুরু হয়:

```
  ইন্টারনেট থেকে আসা রিকোয়েস্ট
         │
         ▼
  ┌──────────────────┐
  │   CDN / Edge     │ ◄── ক্যাশে থাকলে এখান থেকেই রেসপন্স দেয়
  │   (Cloudflare)   │     (Static assets: CSS, JS, Images)
  └────────┬─────────┘
           │ ক্যাশে নেই
           ▼
  ┌──────────────────┐
  │   Load Balancer  │ ◄── একাধিক সার্ভারে রিকোয়েস্ট ভাগ করে
  │   (Nginx/HAProxy)│     (Round Robin, Least Connection)
  └────────┬─────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
  ┌─────┐┌─────┐┌─────┐
  │App 1││App 2││App 3│ ◄── অ্যাপ্লিকেশন সার্ভার (PHP-FPM / Node.js)
  └──┬──┘└──┬──┘└──┬──┘
     │      │      │
     ▼      ▼      ▼
  ┌──────────────────┐
  │   Cache Layer    │ ◄── Redis/Memcached — ঘন ঘন ব্যবহৃত ডেটা
  │   (Redis)        │     সেশন, প্রোডাক্ট ক্যাটালগ
  └────────┬─────────┘
           │ ক্যাশে নেই
           ▼
  ┌──────────────────┐
  │   Database       │ ◄── MySQL/PostgreSQL — মূল ডেটা সংরক্ষণ
  │   (MySQL)        │     প্রোডাক্ট তথ্য, মূল্য, স্টক
  └──────────────────┘
```

### অ্যাপ্লিকেশন সার্ভারে কী ঘটে

```
  GET /products/iphone-15 রিকোয়েস্ট আসলো
         │
         ▼
  ┌───────────────────────┐
  │  ① Middleware          │
  │  ├── CORS চেক          │
  │  ├── Rate Limiting     │──── অতিরিক্ত রিকোয়েস্ট? → 429 Too Many Requests
  │  ├── Authentication    │──── কুকি/টোকেন যাচাই
  │  └── Logging           │
  └──────────┬────────────┘
             │
             ▼
  ┌───────────────────────┐
  │  ② Routing             │
  │  URL → Controller Map  │──── /products/{slug} → ProductController@show
  └──────────┬────────────┘
             │
             ▼
  ┌───────────────────────┐
  │  ③ Controller Logic    │
  │  ├── Input Validation  │──── slug ভ্যালিড?
  │  ├── Business Logic    │──── প্রোডাক্ট আছে? স্টকে আছে?
  │  ├── DB Query          │──── SELECT * FROM products WHERE slug=...
  │  └── View Render       │──── HTML টেমপ্লেট তৈরি
  └──────────┬────────────┘
             │
             ▼
  ┌───────────────────────┐
  │  ④ Response তৈরি       │
  │  ├── HTML Generate     │
  │  ├── Headers সেট       │
  │  └── Compression       │──── gzip/brotli দিয়ে কম্প্রেস
  └────────────────────────┘
```

### PHP (Laravel) — সার্ভার প্রসেসিং উদাহরণ

```php
<?php
// routes/web.php
Route::get('/products/{slug}', [ProductController::class, 'show']);

// app/Http/Controllers/ProductController.php
class ProductController extends Controller
{
    public function show(string $slug)
    {
        // ① Redis ক্যাশে চেক
        $cacheKey = "product:{$slug}";
        $product = Cache::remember($cacheKey, 3600, function () use ($slug) {
            // ② ক্যাশে নেই — ডেটাবেস থেকে আনো
            return Product::where('slug', $slug)
                ->with(['images', 'reviews', 'seller'])
                ->firstOrFail();
        });

        // ③ রিলেটেড প্রোডাক্টও আনো
        $relatedProducts = Product::where('category_id', $product->category_id)
            ->where('id', '!=', $product->id)
            ->limit(6)
            ->get();

        // ④ HTML রেন্ডার করে রেসপন্স দাও
        return view('products.show', [
            'product' => $product,
            'related' => $relatedProducts,
        ]);
    }
}
```

### Node.js (Express) — সার্ভার প্রসেসিং উদাহরণ

```javascript
const express = require('express');
const redis = require('redis');
const mysql = require('mysql2/promise');

const app = express();
const redisClient = redis.createClient();

// GET /products/:slug
app.get('/products/:slug', async (req, res) => {
    const { slug } = req.params;

    try {
        // ① Redis ক্যাশে চেক
        const cached = await redisClient.get(`product:${slug}`);
        if (cached) {
            return res.send(JSON.parse(cached));
        }

        // ② ডেটাবেস থেকে আনো
        const [rows] = await db.query(
            'SELECT * FROM products WHERE slug = ?',
            [slug]
        );

        if (rows.length === 0) {
            return res.status(404).send('প্রোডাক্ট পাওয়া যায়নি');
        }

        const product = rows[0];

        // ③ ক্যাশে রাখো (1 ঘণ্টা)
        await redisClient.setEx(
            `product:${slug}`,
            3600,
            JSON.stringify(product)
        );

        // ④ রেসপন্স দাও
        res.render('product', { product });
    } catch (error) {
        console.error('Error:', error);
        res.status(500).send('সার্ভার সমস্যা');
    }
});
```

---

## 🔗 Step 7: HTTP রেসপন্স

সার্ভার প্রসেসিং শেষে HTTP রেসপন্স পাঠায়:

```
┌─────────────────────────────────────────────────────────────┐
│  HTTP/1.1 200 OK                                           │
│                                                             │
│  Content-Type: text/html; charset=UTF-8                    │
│  Content-Length: 45678                                      │
│  Content-Encoding: gzip                                    │
│  Cache-Control: public, max-age=3600                       │
│  ETag: "abc123def456"                                      │
│  Set-Cookie: _daraz_session=xyz789; Path=/;                │
│              Secure; HttpOnly; SameSite=Lax                │
│  X-Request-Id: req-12345-abcde                             │
│  X-Response-Time: 45ms                                     │
│  Server: nginx/1.24.0                                      │
│  Strict-Transport-Security: max-age=31536000               │
│                                                             │
│  <!DOCTYPE html>                                           │
│  <html lang="bn">                                          │
│  <head>                                                    │
│    <title>iPhone 15 - Daraz Bangladesh</title>             │
│    <link rel="stylesheet" href="/css/app.css">             │
│    <script defer src="/js/app.js"></script>                 │
│  </head>                                                   │
│  <body>                                                    │
│    <!-- পেজের HTML কন্টেন্ট -->                              │
│    ...                                                     │
│  </body>                                                   │
│  </html>                                                   │
└─────────────────────────────────────────────────────────────┘
```

### প্রধান HTTP Status Codes

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Status Codes                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  2xx — সফল ✅                                               │
│  ├── 200 OK          → সফলভাবে রেসপন্স দেওয়া হলো           │
│  ├── 201 Created     → নতুন রিসোর্স তৈরি হলো (POST)        │
│  └── 204 No Content  → সফল কিন্তু কোনো বডি নেই             │
│                                                             │
│  3xx — রিডাইরেক্ট 🔄                                        │
│  ├── 301 Moved       → স্থায়ী রিডাইরেক্ট                    │
│  ├── 302 Found       → অস্থায়ী রিডাইরেক্ট                   │
│  └── 304 Not Modified→ ক্যাশ ব্যবহার করো (পরিবর্তন হয়নি)   │
│                                                             │
│  4xx — ক্লায়েন্ট ত্রুটি ❌                                   │
│  ├── 400 Bad Request → ভুল রিকোয়েস্ট                       │
│  ├── 401 Unauthorized→ লগইন করো                             │
│  ├── 403 Forbidden   → অনুমতি নেই                           │
│  ├── 404 Not Found   → পেজ পাওয়া যায়নি                     │
│  └── 429 Too Many    → অতিরিক্ত রিকোয়েস্ট (Rate Limit)     │
│                                                             │
│  5xx — সার্ভার ত্রুটি 💥                                     │
│  ├── 500 Internal    → সার্ভারে সমস্যা                       │
│  ├── 502 Bad Gateway → আপস্ট্রিম সার্ভার সমস্যা              │
│  ├── 503 Unavailable → সার্ভার ব্যস্ত/রক্ষণাবেক্ষণে          │
│  └── 504 Timeout     → আপস্ট্রিম সার্ভার সময়মতো জবাব দেয়নি │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔗 Step 8: ব্রাউজার রেন্ডারিং

ব্রাউজার HTML রেসপন্স পাওয়ার পর রেন্ডারিং প্রক্রিয়া শুরু করে। এটি সবচেয়ে জটিল ধাপগুলোর একটি:

```
  HTML ডকুমেন্ট প্রাপ্ত
         │
         ▼
  ┌──────────────────┐        ┌──────────────────┐
  │  HTML Parsing     │        │  CSS Parsing      │
  │  (HTML → DOM)     │        │  (CSS → CSSOM)    │
  │                   │        │                   │
  │  <html>           │        │  body {           │
  │   <head>...</head>│        │    font: 16px;    │
  │   <body>          │        │  }                │
  │    <div>          │        │  .product {       │
  │      <h1>iPhone   │        │    color: blue;   │
  │      </h1>        │        │  }                │
  │    </div>         │        │                   │
  │   </body>         │        │                   │
  │  </html>          │        │                   │
  └────────┬─────────┘        └────────┬──────────┘
           │                           │
           ▼                           ▼
  ┌────────────────────────────────────────────┐
  │         Render Tree তৈরি                    │
  │  (DOM + CSSOM = দৃশ্যমান উপাদানগুলো)         │
  │                                            │
  │  শুধু দৃশ্যমান elements                      │
  │  display:none বাদ যায়                       │
  │  <head> বাদ যায়                             │
  └──────────────────┬─────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────┐
  │  Layout (Reflow)         │
  │  প্রতিটি element এর       │
  │  position ও size গণনা    │
  │  (x, y, width, height)   │
  └────────────┬─────────────┘
               │
               ▼
  ┌──────────────────────────┐
  │  Paint                   │
  │  প্রকৃত পিক্সেল আঁকা     │
  │  (text, colors, images,  │
  │   borders, shadows)      │
  └────────────┬─────────────┘
               │
               ▼
  ┌──────────────────────────┐
  │  Composite               │
  │  লেয়ারগুলো একত্রিত করো   │
  │  GPU acceleration ব্যবহার │
  │  স্ক্রিনে চূড়ান্ত আউটপুট  │
  └──────────────────────────┘
```

### DOM Tree উদাহরণ

```
  HTML:                              DOM Tree:
  ─────                              ─────────
  <html>                             Document
    <head>                              │
      <title>iPhone 15</title>          ├── html
    </head>                             │    ├── head
    <body>                              │    │    └── title
      <div class="product">            │    │         └── "iPhone 15"
        <h1>iPhone 15</h1>             │    └── body
        <p class="price">             │         └── div.product
          ৳159,999                     │              ├── h1
        </p>                           │              │    └── "iPhone 15"
        <img src="iphone.jpg">         │              ├── p.price
      </div>                           │              │    └── "৳159,999"
    </body>                            │              └── img
  </html>                              │                   (src="iphone.jpg")
```

### Critical Rendering Path

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Critical Rendering Path                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  সময়রেখা ──────────────────────────────────────────────────►       │
│                                                                     │
│  HTML    ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░              │
│          (পার্সিং)                                                   │
│                                                                     │
│  CSS     ░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░              │
│               (ডাউনলোড ও পার্সিং — render blocking!)                │
│                                                                     │
│  JS      ░░░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░               │
│                     (ডাউনলোড ও এক্সিকিউশন — parser blocking!)      │
│                                                                     │
│  DOM     ████████████████████░░░░░░░░░░░░░░░░░░░░░░░               │
│          (ধীরে ধীরে তৈরি হচ্ছে)                                      │
│                                                                     │
│  CSSOM   ░░░░░░░░████████████░░░░░░░░░░░░░░░░░░░░░░░               │
│                                                                     │
│  Render  ░░░░░░░░░░░░░░░░░░░░██████░░░░░░░░░░░░░░░░               │
│  Tree                                                               │
│                                                                     │
│  Layout  ░░░░░░░░░░░░░░░░░░░░░░░░░░████░░░░░░░░░░░░               │
│                                                                     │
│  Paint   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████░░░░░░░░               │
│                                                                     │
│  FCP ⭐  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░█ (First Contentful     │
│                                                Paint)               │
│                                                                     │
│  💡 CSS হলো render-blocking: CSSOM ছাড়া Render Tree হয় না          │
│  💡 JS হলো parser-blocking: <script> পেলে HTML পার্সিং থামে         │
│  💡 async/defer দিয়ে JS blocking কমানো যায়                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Sub-resource লোডিং

ব্রাউজার HTML পার্স করতে গিয়ে অতিরিক্ত রিসোর্স আবিষ্কার করে এবং সমান্তরালে ডাউনলোড করে:

```
  HTML পার্সিং চলাকালে আবিষ্কৃত রিসোর্স:

  ┌──────────────────────────────────────────────────────────┐
  │  রিসোর্স             │ প্রকার    │ Blocking?              │
  ├──────────────────────────────────────────────────────────┤
  │  app.css             │ CSS      │ ✅ Render Blocking     │
  │  app.js              │ JS       │ ✅ Parser Blocking     │
  │  app.js (defer)      │ JS       │ ❌ Non-blocking        │
  │  app.js (async)      │ JS       │ ❌ Non-blocking        │
  │  iphone.jpg          │ Image    │ ❌ Non-blocking        │
  │  font.woff2          │ Font     │ ⚠️ Text render block   │
  │  analytics.js        │ JS       │ ❌ Non-blocking (async)│
  └──────────────────────────────────────────────────────────┘

  প্রতিটি রিসোর্সের জন্য আবার:
  DNS → TCP → TLS → HTTP Request → Response
  (Keep-Alive ও HTTP/2 multiplexing এটি দ্রুত করে)
```

---

## 📊 সম্পূর্ণ ফ্লো ডায়াগ্রাম

নিচে URL টাইপ থেকে রেন্ডার পেজ পর্যন্ত সম্পূর্ণ যাত্রার বিস্তারিত ডায়াগ্রাম:

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                               │
│   ইউজার: "www.daraz.com.bd" টাইপ করে Enter চাপলো                              │
│                                                                               │
│   ┌─────────────┐                                                             │
│   │  ব্রাউজার    │                                                             │
│   │  (Chrome)    │                                                             │
│   └──────┬──────┘                                                             │
│          │                                                                    │
│          │ ① URL পার্সিং                                                       │
│          │ ├── protocol: https                                                │
│          │ ├── domain: www.daraz.com.bd                                       │
│          │ └── port: 443                                                      │
│          │                                                                    │
│          │ ② DNS Resolution                                                   │
│          │ ┌─────────────────────────────────────────────────┐                 │
│          │ │ Browser Cache ──miss──► OS Cache ──miss──►      │                 │
│          │ │ Router Cache ──miss──► ISP DNS ──miss──►        │                 │
│          │ │ Root (.─►) TLD (.bd─►) Auth (daraz.com.bd)     │                 │
│          │ │ ──► IP: 103.108.12.45 ✅                        │                 │
│          │ └─────────────────────────────────────────────────┘                 │
│          │                                                                    │
│          │ ③ TCP 3-Way Handshake                                              │
│          │         Client ──SYN──────────────► Server                         │
│          │         Client ◄──SYN-ACK────────── Server                         │
│          │         Client ──ACK──────────────► Server                         │
│          │                                                                    │
│          │ ④ TLS Handshake (HTTPS)                                            │
│          │         Client ──ClientHello──────► Server                         │
│          │         Client ◄──ServerHello+Cert─ Server                         │
│          │         Client ──KeyShare+Finish──► Server                         │
│          │         🔐 Encrypted Channel Ready                                 │
│          │                                                                    │
│          │ ⑤ HTTP Request                                                     │
│          │ ┌──────────────────────────────────┐                                │
│          │ │ GET /products/iphone-15 HTTP/1.1 │                                │
│          │ │ Host: www.daraz.com.bd           │                                │
│          │ │ Cookie: session=abc123           │                                │
│          ├─┴──────────────────────────────────┘───────────────────────┐        │
│          │                                                           │        │
│          │                    ⑥ সার্ভার সাইড                           │        │
│          │          ┌─────────────────────────────────┐               │        │
│          │          │         CDN / Edge               │               │        │
│          │          │      (Static ক্যাশ চেক)           │               │        │
│          │          └──────────┬──────────────────────┘               │        │
│          │                    │ Dynamic content                       │        │
│          │          ┌─────────▼──────────────────────┐               │        │
│          │          │      Load Balancer              │               │        │
│          │          │    (Nginx / HAProxy)            │               │        │
│          │          └──┬──────────┬──────────┬───────┘               │        │
│          │             ▼          ▼          ▼                        │        │
│          │          ┌─────┐  ┌─────┐  ┌─────┐                        │        │
│          │          │App 1│  │App 2│  │App 3│  (PHP/Node.js)         │        │
│          │          └──┬──┘  └─────┘  └─────┘                        │        │
│          │             │                                             │        │
│          │             ├──► Redis Cache (ক্যাশ চেক)                   │        │
│          │             │                                             │        │
│          │             ├──► MySQL DB (ডেটা আনো)                      │        │
│          │             │                                             │        │
│          │             ▼                                             │        │
│          │          HTML Response তৈরি                                │        │
│          │                                                           │        │
│          │ ⑦ HTTP Response                                           │        │
│          │ ┌──────────────────────────────────┐                       │        │
│          │ │ HTTP/1.1 200 OK                  │◄──────────────────────┘        │
│          │ │ Content-Type: text/html          │                                │
│          │ │ Content-Encoding: gzip           │                                │
│          │ │                                  │                                │
│          │ │ <!DOCTYPE html>                  │                                │
│          │ │ <html>...</html>                 │                                │
│          │ └──────────────────────────────────┘                                │
│          │                                                                    │
│          │ ⑧ ব্রাউজার রেন্ডারিং                                                │
│          │ ┌──────────────────────────────────────────────────┐                │
│          │ │                                                  │                │
│          │ │  HTML ──► DOM Tree                               │                │
│          │ │                    ╲                              │                │
│          │ │                     ► Render Tree ► Layout ► Paint │               │
│          │ │                    ╱                              │                │
│          │ │  CSS ──► CSSOM                                   │                │
│          │ │                                                  │                │
│          │ │  JS ──► DOM পরিবর্তন ──► Re-render               │                │
│          │ │                                                  │                │
│          │ └──────────────────────────────────────────────────┘                │
│          │                                                                    │
│          ▼                                                                    │
│   ┌──────────────────────────────────────┐                                    │
│   │  🖥️ ইউজার পর্দায় Daraz এর পেজ দেখে!  │                                    │
│   │     iPhone 15 — ৳159,999              │                                    │
│   │     ⭐⭐⭐⭐☆ (4.2 রেটিং)                │                                    │
│   └──────────────────────────────────────┘                                    │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP ও JavaScript কোড উদাহরণ

### সম্পূর্ণ PHP সার্ভার — রিকোয়েস্ট হ্যান্ডলিং

```php
<?php
/**
 * একটি সরল PHP সার্ভার যা URL টু পেজ যাত্রার সার্ভার সাইড দেখায়
 */

// ① রিকোয়েস্ট তথ্য পার্স করা
$method  = $_SERVER['REQUEST_METHOD'];          // GET
$uri     = $_SERVER['REQUEST_URI'];             // /products/iphone-15?color=black
$host    = $_SERVER['HTTP_HOST'];               // www.daraz.com.bd
$agent   = $_SERVER['HTTP_USER_AGENT'];         // Chrome/120.0...
$ip      = $_SERVER['REMOTE_ADDR'];             // ইউজারের IP

// ② রাউটিং
$path = parse_url($uri, PHP_URL_PATH);          // /products/iphone-15
$query = [];
parse_str(parse_url($uri, PHP_URL_QUERY) ?? '', $query);

// ③ সেশন ও অথেনটিকেশন চেক
session_start();
$isLoggedIn = isset($_SESSION['user_id']);
$userId = $_SESSION['user_id'] ?? null;

// ④ রাউট ম্যাচিং
if (preg_match('#^/products/([a-z0-9-]+)$#', $path, $matches)) {
    $slug = $matches[1]; // iphone-15

    // ⑤ ডেটাবেস থেকে প্রোডাক্ট আনো
    $pdo = new PDO('mysql:host=localhost;dbname=daraz', 'root', 'password');
    $stmt = $pdo->prepare('SELECT * FROM products WHERE slug = :slug AND active = 1');
    $stmt->execute(['slug' => $slug]);
    $product = $stmt->fetch(PDO::FETCH_ASSOC);

    if (!$product) {
        // প্রোডাক্ট পাওয়া যায়নি
        http_response_code(404);
        echo '<h1>প্রোডাক্ট পাওয়া যায়নি</h1>';
        exit;
    }

    // ⑥ রেসপন্স হেডার সেট
    header('Content-Type: text/html; charset=UTF-8');
    header('Cache-Control: public, max-age=3600');
    header('X-Response-Time: ' . (microtime(true) - $_SERVER['REQUEST_TIME_FLOAT']) . 's');

    // ⑦ HTML আউটপুট
    ?>
    <!DOCTYPE html>
    <html lang="bn">
    <head>
        <meta charset="UTF-8">
        <title><?= htmlspecialchars($product['name']) ?> - Daraz BD</title>
        <link rel="stylesheet" href="/css/app.css">
    </head>
    <body>
        <h1><?= htmlspecialchars($product['name']) ?></h1>
        <p class="price">৳<?= number_format($product['price']) ?></p>
        <p><?= htmlspecialchars($product['description']) ?></p>
        <script defer src="/js/app.js"></script>
    </body>
    </html>
    <?php
} else {
    http_response_code(404);
    echo '<h1>৪০৪ - পেজ পাওয়া যায়নি</h1>';
}
```

### সম্পূর্ণ Node.js সার্ভার — রিকোয়েস্ট হ্যান্ডলিং

```javascript
const http = require('http');
const { URL } = require('url');
const mysql = require('mysql2/promise');

// ডেটাবেস কানেকশন পুল
const pool = mysql.createPool({
    host: 'localhost',
    user: 'root',
    password: 'password',
    database: 'daraz',
    connectionLimit: 10,
});

const server = http.createServer(async (req, res) => {
    const startTime = Date.now();
    const url = new URL(req.url, `https://${req.headers.host}`);

    console.log(`${req.method} ${url.pathname} — ${req.headers['user-agent']}`);

    // রাউট ম্যাচিং
    const productMatch = url.pathname.match(/^\/products\/([a-z0-9-]+)$/);

    if (productMatch && req.method === 'GET') {
        const slug = productMatch[1];

        try {
            // ডেটাবেস কোয়েরি
            const [rows] = await pool.query(
                'SELECT * FROM products WHERE slug = ? AND active = 1',
                [slug]
            );

            if (rows.length === 0) {
                res.writeHead(404, { 'Content-Type': 'text/html; charset=utf-8' });
                res.end('<h1>প্রোডাক্ট পাওয়া যায়নি</h1>');
                return;
            }

            const product = rows[0];
            const responseTime = Date.now() - startTime;

            // রেসপন্স হেডার
            res.writeHead(200, {
                'Content-Type': 'text/html; charset=utf-8',
                'Cache-Control': 'public, max-age=3600',
                'X-Response-Time': `${responseTime}ms`,
            });

            // HTML রেসপন্স
            res.end(`
                <!DOCTYPE html>
                <html lang="bn">
                <head>
                    <meta charset="UTF-8">
                    <title>${product.name} - Daraz BD</title>
                    <link rel="stylesheet" href="/css/app.css">
                </head>
                <body>
                    <h1>${product.name}</h1>
                    <p class="price">৳${product.price.toLocaleString('bn-BD')}</p>
                    <p>${product.description}</p>
                    <script defer src="/js/app.js"></script>
                </body>
                </html>
            `);
        } catch (error) {
            console.error('DB Error:', error);
            res.writeHead(500, { 'Content-Type': 'text/html; charset=utf-8' });
            res.end('<h1>সার্ভারে সমস্যা হয়েছে</h1>');
        }
    } else {
        res.writeHead(404, { 'Content-Type': 'text/html; charset=utf-8' });
        res.end('<h1>৪০৪ — পেজ পাওয়া যায়নি</h1>');
    }
});

server.listen(3000, () => {
    console.log('সার্ভার চালু: http://localhost:3000');
});
```

---

## 🇧🇩 বাংলাদেশী উদাহরণ

### উদাহরণ ১: `daraz.com.bd` ভিজিট

```
  আপনি Chrome এ টাইপ করলেন: daraz.com.bd
  ──────────────────────────────────────────

  ① URL পার্সিং
     ├── প্রোটোকল নেই → https:// যোগ করো
     ├── HSTS লিস্টে আছে → https বাধ্যতামূলক
     └── চূড়ান্ত URL: https://www.daraz.com.bd/

  ② DNS Resolution
     ├── ব্রাউজার ক্যাশ: ❌ (প্রথমবার)
     ├── OS ক্যাশ: ❌
     ├── ISP DNS (GP/BTCL): ❌
     ├── Root → .bd TLD → .com.bd → daraz.com.bd
     └── IP: 103.108.12.45 (Daraz BD এর Cloudflare CDN)

  ③ TCP Handshake
     └── Dhaka → Singapore (Cloudflare Edge) ≈ 30ms RTT

  ④ TLS Handshake
     ├── Certificate: *.daraz.com.bd (Cloudflare)
     ├── TLS 1.3 → 1 RTT
     └── মোট: ≈ 30ms

  ⑤ HTTP Request
     ├── GET / HTTP/2
     ├── Host: www.daraz.com.bd
     └── Accept-Language: bn-BD

  ⑥ সার্ভার প্রসেসিং
     ├── Cloudflare CDN → Edge ক্যাশ চেক
     ├── Origin: Singapore ডেটাসেন্টার
     ├── PHP/Java App Server
     ├── MySQL → প্রোডাক্ট ক্যাটালগ
     ├── Redis → হোমপেজ ক্যাশ, সেশন
     └── ≈ 100ms

  ⑦ HTTP Response
     ├── 200 OK
     ├── Content-Encoding: br (Brotli compression)
     ├── HTML: 85KB (compressed: 18KB)
     └── পরে আরো: CSS (120KB), JS (450KB), Images (2MB)

  ⑧ রেন্ডারিং
     ├── HTML → DOM (200+ elements)
     ├── CSS → CSSOM (responsive layout)
     ├── Lazy load images
     ├── JavaScript SPA framework লোড
     ├── FCP (First Contentful Paint): ≈ 800ms
     └── LCP (Largest Contentful Paint): ≈ 2.5s

  মোট সময়: ≈ 2-4 সেকেন্ড (ফুল পেজ লোড)
```

### উদাহরণ ২: `bkash.com` (বিকাশ ওয়েবসাইট)

```
  আপনি টাইপ করলেন: bkash.com
  ─────────────────────────────

  ① URL পার্সিং
     └── https://www.bkash.com/

  ② DNS Resolution
     ├── bkash.com → Cloudflare DNS
     └── IP: 104.18.xx.xx (Cloudflare CDN)

  ③-④ TCP + TLS
     ├── Certificate: bkash.com (DigiCert — ব্যাংকিং গ্রেড)
     ├── Extended Validation (EV) Certificate
     └── HSTS preloaded

  ⑤-⑥ Request & Processing
     ├── Static site (মার্কেটিং পেজ)
     ├── CDN থেকে সরাসরি সার্ভ
     └── ≈ 50ms (CDN ক্যাশ)

  ⑦-⑧ Response & Render
     ├── HTML + CSS + JS
     ├── বাংলা ফন্ট লোড (SolaimanLipi/Hind Siliguri)
     ├── FCP: ≈ 500ms
     └── মোট: ≈ 1-2 সেকেন্ড

  🔒 নিরাপত্তা বিশেষত্ব:
     ├── EV Certificate (সবুজ লক আইকন)
     ├── HSTS + Preload (সবসময় HTTPS)
     ├── CSP Headers (XSS প্রতিরোধ)
     ├── Rate Limiting (DDoS প্রতিরোধ)
     └── WAF (Web Application Firewall)
```

### উদাহরণ ৩: Pathao রাইড বুকিং (API Call)

```
  Pathao অ্যাপে রাইড বুকিং করলে পেছনে HTTP রিকোয়েস্ট যায়:

  POST https://api.pathao.com/v1/rides HTTP/2
  ─────────────────────────────────────────────
  Authorization: Bearer eyJhbGciOiJSUzI1...
  Content-Type: application/json

  {
    "pickup": {
      "lat": 23.8103,
      "lng": 90.4125,
      "address": "মিরপুর ১০, ঢাকা"
    },
    "destination": {
      "lat": 23.7806,
      "lng": 90.2792,
      "address": "গুলশান ২, ঢাকা"
    },
    "ride_type": "bike",
    "payment_method": "bkash"
  }

  ──── সার্ভারে কী ঘটে ────
  ├── JWT Token যাচাই
  ├── কাছাকাছি ড্রাইভার খোঁজো (geospatial query)
  ├── ভাড়া হিসাব করো (দূরত্ব × রেট)
  ├── ড্রাইভারকে নোটিফিকেশন পাঠাও (WebSocket/FCM)
  └── রেসপন্স:

  HTTP/2 201 Created
  {
    "ride_id": "RIDE-123456",
    "status": "finding_driver",
    "estimated_fare": 85,
    "currency": "BDT",
    "estimated_time": "12 min"
  }
```

---

## 🎯 ইন্টারভিউ টিপস

### ৩০ সেকেন্ডে উত্তর দিন (সংক্ষিপ্ত ভার্সন)

```
"ব্রাউজারে URL টাইপ করলে ৮টি প্রধান ধাপ ঘটে:

 ১. URL পার্সিং — ব্রাউজার URL এর অংশগুলো ভাগ করে
 ২. DNS Lookup — ডোমেইন নাম থেকে IP Address বের করে
 ৩. TCP Handshake — 3-way handshake দিয়ে সংযোগ স্থাপন
 ৪. TLS Handshake — HTTPS হলে এনক্রিপ্টেড চ্যানেল তৈরি
 ৫. HTTP Request — সার্ভারে GET রিকোয়েস্ট পাঠায়
 ৬. Server Processing — Load balancer, app server, database
 ৭. HTTP Response — HTML/CSS/JS সহ রেসপন্স আসে
 ৮. Browser Rendering — DOM → CSSOM → Render Tree → Paint"
```

### ইন্টারভিউয়ারকে ইমপ্রেস করার পয়েন্ট

```
┌─────────────────────────────────────────────────────────────────────┐
│                   গুরুত্বপূর্ণ ফলো-আপ পয়েন্ট                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  🔍 DNS অপটিমাইজেশন                                                │
│  ├── DNS Prefetching: <link rel="dns-prefetch" href="//cdn.com">   │
│  ├── DNS over HTTPS (DoH) — প্রাইভেসি                              │
│  └── Anycast DNS — কাছের সার্ভার থেকে রেজোল্ভ                      │
│                                                                     │
│  ⚡ TCP অপটিমাইজেশন                                                │
│  ├── TCP Fast Open (TFO) — SYN এর সাথেই ডেটা                      │
│  ├── Connection Pooling (Keep-Alive)                                │
│  └── HTTP/2 Multiplexing — একটি কানেকশনে অনেক রিকোয়েস্ট          │
│                                                                     │
│  🔐 TLS অপটিমাইজেশন                                                │
│  ├── TLS 1.3 → 1-RTT (আগে 2-RTT ছিলো)                             │
│  ├── 0-RTT Resumption — পুনঃসংযোগে হ্যান্ডশেক ছাড়াই               │
│  ├── OCSP Stapling — আলাদা certificate check এড়ানো                 │
│  └── Certificate Pinning — MITM আক্রমণ প্রতিরোধ                    │
│                                                                     │
│  🚀 রেন্ডারিং অপটিমাইজেশন                                          │
│  ├── Critical CSS Inline করো                                        │
│  ├── JavaScript async/defer ব্যবহার করো                             │
│  ├── Image Lazy Loading                                             │
│  ├── Preload গুরুত্বপূর্ণ রিসোর্স                                    │
│  └── Service Worker → Offline Support                               │
│                                                                     │
│  📊 পারফরম্যান্স মেট্রিক্স (Core Web Vitals)                        │
│  ├── FCP (First Contentful Paint) < 1.8s                            │
│  ├── LCP (Largest Contentful Paint) < 2.5s                          │
│  ├── FID (First Input Delay) < 100ms                                │
│  └── CLS (Cumulative Layout Shift) < 0.1                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### সাধারণ ইন্টারভিউ প্রশ্ন ও উত্তর

| # | প্রশ্ন | মূল উত্তর |
|---|--------|-----------|
| ১ | DNS কেন দরকার? | মানুষ নাম মনে রাখে, কম্পিউটার IP বোঝে — DNS দুটো সংযুক্ত করে |
| ২ | TCP কেন 3-way? | উভয় পক্ষ পাঠাতে ও গ্রহণ করতে পারে নিশ্চিত করতে |
| ৩ | HTTPS কেন HTTP নয়? | ডেটা এনক্রিপশন, সার্ভার যাচাই, ডেটা অখণ্ডতা |
| ৪ | ক্যাশিং কোথায় কোথায় হয়? | Browser, CDN, Load Balancer, App (Redis), DB Query Cache |
| ৫ | কীভাবে পেজ দ্রুত লোড করবেন? | CDN, compression, lazy loading, code splitting, caching |
| ৬ | HTTP/2 তে কী সুবিধা? | Multiplexing, header compression, server push |
| ৭ | DOM কী? | HTML এর প্রোগ্রামেটিক রূপ — tree structure যা JS দিয়ে manipulate করা যায় |
| ৮ | Render blocking কী? | CSS ও sync JS যা পেজ রেন্ডারিং থামিয়ে দেয় |

### মনে রাখার সূত্র

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   "ইউ ডি টি টি আর-আর-আর পেইন্ট"                            │
│                                                             │
│   U — URL Parse                                             │
│   D — DNS Lookup                                            │
│   T — TCP Handshake                                         │
│   T — TLS Handshake                                         │
│   R — Request (HTTP)                                        │
│   R — (Server) Response Processing                          │
│   R — Response (HTTP)                                       │
│   Paint — Browser Rendering (DOM → Paint)                   │
│                                                             │
│   অথবা বাংলায়:                                              │
│   "ইউআরএল ডিএনএস টিসিপি টিএলএস — রিকোয়েস্ট সার্ভার        │
│    রেসপন্স রেন্ডার"                                          │
│                                                             │
│   ৪টি নেটওয়ার্ক ধাপ (U-D-T-T) + ৪টি ডেটা ধাপ (R-R-R-P)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 আরও পড়ুন

- [DNS বিস্তারিত](./dns.md)
- [TCP ও UDP](./tcp-udp.md)
- [HTTP প্রোটোকল](./http-protocol.md)
- [TLS/SSL](./tls-ssl.md)
- [OSI মডেল](./osi-model.md)

---

> **💡 মনে রাখুন:** ইন্টারভিউতে এই প্রশ্নের উত্তর দেওয়ার সময় প্রথমে সংক্ষেপে ৮টি ধাপ বলুন, তারপর
> ইন্টারভিউয়ার যে ধাপে গভীরে যেতে চান সেখানে বিস্তারিত বলুন। সব একসাথে বলতে গেলে সময় শেষ হয়ে যাবে!
