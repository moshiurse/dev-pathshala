# 🌍 DNS (Domain Name System)

> DNS হলো ইন্টারনেটের "ফোন বুক"। এটি মানুষের পড়ার যোগ্য ডোমেইন নাম (daraz.com.bd) কে
> কম্পিউটারের বোঝার যোগ্য IP address (103.108.12.45) এ রূপান্তর করে।

---

## 📖 সূচিপত্র

- [DNS কী?](#-dns-কী)
- [DNS Resolution Process](#-dns-resolution-process)
- [DNS Record Types](#-dns-record-types)
- [DNS Caching](#-dns-caching)
- [DNS Propagation](#-dns-propagation)
- [Round-Robin DNS](#-round-robin-dns-load-balancing)
- [DNS Security (DNSSEC)](#-dns-security-dnssec)
- [বাস্তব উদাহরণ: daraz.com.bd](#-বাস্তব-উদাহরণ-darazcombd-ভিজিট)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 DNS কী?

DNS ছাড়া ইন্টারনেট ব্যবহার করতে হলে আপনাকে প্রতিটি ওয়েবসাইটের IP মনে রাখতে হতো:

```
DNS ছাড়া:                          DNS সহ:
──────────                          ───────
103.108.12.45 → Daraz               daraz.com.bd → Daraz ✅
172.217.14.206 → Google             google.com → Google ✅
31.13.65.36 → Facebook              facebook.com → Facebook ✅
157.240.1.35 → Instagram            instagram.com → Instagram ✅

DNS = ইন্টারনেটের ফোন বুক 📖
ডোমেইন নাম (নাম) → IP Address (নম্বর)
```

### DNS Hierarchy

```
                         ┌──────────┐
                         │  . (Root)│
                         └────┬─────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
       ┌─────────┐      ┌─────────┐      ┌─────────┐
       │  .com   │      │  .bd    │      │  .org   │
       │  (TLD)  │      │  (ccTLD)│      │  (TLD)  │
       └────┬────┘      └────┬────┘      └─────────┘
            │                │
      ┌─────┤          ┌─────┤
      ▼     ▼          ▼     ▼
  ┌──────┐┌──────┐ ┌──────┐┌──────────┐
  │google││amazon│ │.com  ││.gov      │
  │.com  ││.com  │ │.bd   ││.bd       │
  └──────┘└──────┘ └──┬───┘└──────────┘
                      │
                ┌─────┤
                ▼     ▼
           ┌───────┐┌───────┐
           │daraz  ││bkash  │
           │.com.bd││.com.bd│
           └───────┘└───────┘

  TLD = Top Level Domain
  ccTLD = Country Code TLD (.bd = বাংলাদেশ)
  SLD = Second Level Domain (daraz, bkash)
  FQDN = Fully Qualified Domain Name (www.daraz.com.bd.)
```

---

## 🔗 DNS Resolution Process

আপনি যখন ব্রাউজারে `www.daraz.com.bd` টাইপ করেন, ৪ ধরনের DNS সার্ভার কাজ করে:

```
আপনার ব্রাউজার                                     ওয়েবসাইট
      │                                                │
      │  Step 1: ব্রাউজার ক্যাশ চেক                      │
      │  Step 2: OS ক্যাশ চেক (/etc/hosts)               │
      │  Step 3: Router ক্যাশ চেক                        │
      │                                                │
      ▼                                                │
┌──────────────┐                                       │
│  Recursive   │  (আপনার ISP এর DNS, যেমন GP DNS)       │
│  Resolver    │                                       │
│ 203.130.     │                                       │
│  193.74      │                                       │
└──────┬───────┘                                       │
       │                                               │
       │  Step 4: ". (Root)" কে জিজ্ঞাসা                │
       ▼                                               │
┌──────────────┐                                       │
│ Root Name    │  "আমি .bd এর কথা জানি না, কিন্তু      │
│ Server       │   .bd TLD সার্ভারের ঠিকানা দিচ্ছি"       │
│ (13 clusters)│                                       │
└──────┬───────┘                                       │
       │                                               │
       │  Step 5: ".bd TLD" কে জিজ্ঞাসা                 │
       ▼                                               │
┌──────────────┐                                       │
│ TLD Name     │  "daraz.com.bd এর Authoritative       │
│ Server (.bd) │   NS সার্ভারের ঠিকানা দিচ্ছি"           │
│ (BTCL)       │                                       │
└──────┬───────┘                                       │
       │                                               │
       │  Step 6: Authoritative NS কে জিজ্ঞাসা          │
       ▼                                               │
┌──────────────┐                                       │
│Authoritative │  "www.daraz.com.bd = 103.108.12.45"   │
│ Name Server  │  ✅ চূড়ান্ত উত্তর!                     │
│(Daraz's DNS) │                                       │
└──────┬───────┘                                       │
       │                                               │
       │  Step 7: IP address ফিরিয়ে দেওয়া               │
       ▼                                               │
  ব্রাউজার ─── TCP/TLS ─── 103.108.12.45 ────────────► ওয়েবসাইট
```

### বিস্তারিত ডায়াগ্রাম

```
ব্রাউজার: "www.daraz.com.bd এর IP কী?"
    │
    ├── ① ব্রাউজার ক্যাশ আছে? ──── হ্যাঁ ──► IP ব্যবহার করো ✅
    │                               না
    ├── ② OS DNS ক্যাশ আছে? ────── হ্যাঁ ──► IP ব্যবহার করো ✅
    │   (/etc/hosts চেক)           না
    ├── ③ Router ক্যাশ আছে? ────── হ্যাঁ ──► IP ব্যবহার করো ✅
    │                               না
    ▼
Recursive Resolver (ISP DNS):
    │
    ├── ④ নিজের ক্যাশে আছে? ────── হ্যাঁ ──► IP রিটার্ন ✅
    │                               না
    ├── ⑤ Root Server জিজ্ঞাসা ──► ".bd = ns1.btcl.net.bd"
    │
    ├── ⑥ .bd TLD জিজ্ঞাসা ──────► "daraz.com.bd NS = ns1.daraz.com"
    │
    ├── ⑦ Authoritative NS ──────► "www.daraz.com.bd = 103.108.12.45"
    │
    └── IP ক্যাশ করো + ব্রাউজারে ফিরিয়ে দাও ✅
```

---

## 📖 DNS Record Types

```
┌──────────┬──────────────────────┬──────────────────────────────────┐
│  Type    │       নাম             │            উদ্দেশ্য               │
├──────────┼──────────────────────┼──────────────────────────────────┤
│    A     │ Address              │ ডোমেইন → IPv4 Address            │
│          │                      │ daraz.com.bd → 103.108.12.45    │
├──────────┼──────────────────────┼──────────────────────────────────┤
│   AAAA   │ IPv6 Address         │ ডোমেইন → IPv6 Address            │
│          │                      │ google.com → 2404:6800:4003::   │
├──────────┼──────────────────────┼──────────────────────────────────┤
│  CNAME   │ Canonical Name       │ ডোমেইন → অন্য ডোমেইন (Alias)     │
│          │                      │ www.daraz.com.bd → daraz.com.bd  │
├──────────┼──────────────────────┼──────────────────────────────────┤
│    MX    │ Mail Exchange        │ ইমেইল সার্ভার নির্দেশ              │
│          │                      │ daraz.com.bd → mail.daraz.com.bd│
│          │                      │ (priority: 10)                  │
├──────────┼──────────────────────┼──────────────────────────────────┤
│    NS    │ Name Server          │ ডোমেইনের DNS সার্ভার              │
│          │                      │ daraz.com.bd → ns1.cloudflare..│
├──────────┼──────────────────────┼──────────────────────────────────┤
│   TXT    │ Text                 │ টেক্সট তথ্য (SPF, DKIM, verify) │
│          │                      │ v=spf1 include:_spf.google.com │
├──────────┼──────────────────────┼──────────────────────────────────┤
│   SRV    │ Service              │ নির্দিষ্ট সেবার হোস্ট ও পোর্ট     │
│          │                      │ _sip._tcp.example.com           │
├──────────┼──────────────────────┼──────────────────────────────────┤
│   PTR    │ Pointer              │ IP → ডোমেইন (Reverse DNS)       │
│          │                      │ 103.108.12.45 → daraz.com.bd   │
├──────────┼──────────────────────┼──────────────────────────────────┤
│   SOA    │ Start of Authority   │ ডোমেইনের প্রাথমিক তথ্য            │
│          │                      │ Primary NS, Admin email, Serial │
├──────────┼──────────────────────┼──────────────────────────────────┤
│   CAA    │ Cert. Authority Auth │ কোন CA সার্টিফিকেট দিতে পারবে    │
│          │                      │ 0 issue "letsencrypt.org"       │
└──────────┴──────────────────────┴──────────────────────────────────┘
```

### বাস্তব DNS Records (daraz.com.bd এর মতো)

```
; daraz.com.bd এর DNS Zone File (উদাহরণ)
; ──────────────────────────────────────────

; SOA Record
daraz.com.bd.    IN  SOA  ns1.cloudflare.com. admin.daraz.com.bd. (
                         2024010101 ; Serial
                         3600       ; Refresh (1 hour)
                         900        ; Retry (15 min)
                         604800     ; Expire (1 week)
                         300        ; Minimum TTL (5 min)
                     )

; NS Records (Name Servers)
daraz.com.bd.    IN  NS   ns1.cloudflare.com.
daraz.com.bd.    IN  NS   ns2.cloudflare.com.

; A Records (IPv4)
daraz.com.bd.    IN  A    103.108.12.45
daraz.com.bd.    IN  A    103.108.12.46    ; Load balancing

; AAAA Record (IPv6)
daraz.com.bd.    IN  AAAA 2606:4700:3033::6815:1234

; CNAME Records (Aliases)
www              IN  CNAME daraz.com.bd.
api              IN  CNAME api-gateway.daraz.com.bd.
cdn              IN  CNAME d1234.cloudfront.net.
shop             IN  CNAME daraz.com.bd.

; MX Records (Email)
daraz.com.bd.    IN  MX   10 mail1.daraz.com.bd.
daraz.com.bd.    IN  MX   20 mail2.daraz.com.bd.

; TXT Records
daraz.com.bd.    IN  TXT  "v=spf1 include:_spf.google.com ~all"
daraz.com.bd.    IN  TXT  "google-site-verification=abc123..."

; SRV Record
_sip._tcp.daraz.com.bd. IN SRV 10 5 5060 sip.daraz.com.bd.
```

---

## 📊 DNS Caching

DNS resolution ত্বরান্বিত করতে একাধিক স্তরে ক্যাশিং হয়:

```
┌─────────────────────────────────────────────────────────┐
│                   DNS Caching Layers                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Layer 1: ব্রাউজার ক্যাশ                                 │
│  ├── TTL: সাধারণত ৬০ সেকেন্ড                              │
│  ├── Chrome: chrome://net-internals/#dns                │
│  └── সবচেয়ে দ্রুত, RAM এ থাকে                             │
│                                                         │
│  Layer 2: OS ক্যাশ                                       │
│  ├── Linux: systemd-resolved                            │
│  ├── macOS: mDNSResponder                               │
│  ├── Windows: DNS Client Service                        │
│  └── /etc/hosts ফাইলও এখানে চেক হয়                     │
│                                                         │
│  Layer 3: Router ক্যাশ                                    │
│  ├── বাসার Wi-Fi router                                  │
│  └── ISP gateway                                        │
│                                                         │
│  Layer 4: ISP DNS Resolver ক্যাশ                          │
│  ├── Grameenphone: 203.130.193.74                       │
│  ├── BTCL: 203.112.200.5                                │
│  ├── Google: 8.8.8.8                                    │
│  ├── Cloudflare: 1.1.1.1                                │
│  └── সবচেয়ে বড় ক্যাশ, অনেক ইউজার শেয়ার করে              │
│                                                         │
└─────────────────────────────────────────────────────────┘

TTL (Time To Live) = ক্যাশে কতক্ষণ রাখবে

ক্যাশ হিট:
  Browser ─ HIT! ──► IP পেয়ে গেছি! (< 1ms)

ক্যাশ মিস:
  Browser ─ MISS ──► OS ─ MISS ──► Router ─ MISS ──► ISP DNS
  ─ MISS ──► Root ──► TLD ──► Authoritative  (~50-200ms)
```

### TTL উদাহরণ

```
┌────────────────────┬──────────┬───────────────────────────┐
│  Record Type       │  TTL     │  কারণ                      │
├────────────────────┼──────────┼───────────────────────────┤
│ স্ট্যাটিক ওয়েবসাইট  │ 86400   │ IP কদাচিৎ পরিবর্তন হয়    │
│ (24 ঘণ্টা)          │ (1 day)  │                           │
├────────────────────┼──────────┼───────────────────────────┤
│ API সার্ভার         │ 300      │ স্কেলিং এ IP বদলাতে পারে   │
│ (5 মিনিট)          │ (5 min)  │                           │
├────────────────────┼──────────┼───────────────────────────┤
│ CDN                │ 60       │ ঘন ঘন পরিবর্তন হতে পারে   │
│ (1 মিনিট)          │ (1 min)  │                           │
├────────────────────┼──────────┼───────────────────────────┤
│ Failover Record    │ 30       │ দ্রুত সুইচ করতে হবে        │
│ (30 সেকেন্ড)       │ (30 sec) │                           │
└────────────────────┴──────────┴───────────────────────────┘
```

---

## 🔄 DNS Propagation

DNS রেকর্ড পরিবর্তন করলে পুরো বিশ্বে ছড়াতে সময় লাগে (propagation):

```
DNS রেকর্ড পরিবর্তন করলাম: daraz.com.bd → নতুন IP

t=0: Authoritative NS আপডেট হলো ✅
     │
t=5m: ISP DNS (ঢাকা) ক্যাশ expire → নতুন IP পেলো ✅
     │
t=30m: ISP DNS (চট্টগ্রাম) ক্যাশ expire → নতুন IP পেলো ✅
     │
t=1h: Google DNS (8.8.8.8) ক্যাশ expire → নতুন IP পেলো ✅
     │
t=2h: বিভিন্ন দেশের ISP ক্যাশ expire হচ্ছে...
     │
t=24-48h: বেশিরভাগ DNS সার্ভার আপডেট হয়ে গেছে ✅

Propagation সময় কমানোর উপায়:
──────────────────────────────
1. মাইগ্রেশনের আগে TTL কমিয়ে রাখুন (৩০০ → ৬০ সেকেন্ড)
2. পুরানো ও নতুন দুই সার্ভারই কিছুদিন চালু রাখুন
3. Cloudflare/Route53 এর মতো সার্ভিস ব্যবহার করুন
```

---

## ⚖️ Round-Robin DNS Load Balancing

একটি ডোমেইনে একাধিক IP রেকর্ড রাখলে DNS round-robin করে:

```
daraz.com.bd DNS Records:
  A  103.108.12.45   (Server 1 - ঢাকা)
  A  103.108.12.46   (Server 2 - ঢাকা)
  A  103.108.12.47   (Server 3 - চট্টগ্রাম)

Query 1: daraz.com.bd → 103.108.12.45 ──► Server 1
Query 2: daraz.com.bd → 103.108.12.46 ──► Server 2
Query 3: daraz.com.bd → 103.108.12.47 ──► Server 3
Query 4: daraz.com.bd → 103.108.12.45 ──► Server 1 (আবার!)
Query 5: daraz.com.bd → 103.108.12.46 ──► Server 2

┌──────────────┐    ┌──────────────┐
│   Client 1   │───►│  Server 1    │
└──────────────┘    │  103.108.    │
                    │  12.45       │
┌──────────────┐    └──────────────┘
│   Client 2   │───►┌──────────────┐
└──────────────┘    │  Server 2    │
                    │  103.108.    │
┌──────────────┐    │  12.46       │
│   Client 3   │───►└──────────────┘
└──────────────┘    ┌──────────────┐
                    │  Server 3    │
                    │  103.108.    │
                    │  12.47       │
                    └──────────────┘
```

### সীমাবদ্ধতা

```
┌─────────────────────────────────────────────────────────┐
│         Round-Robin DNS এর সীমাবদ্ধতা                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ❌ Health Check নেই                                     │
│    → সার্ভার ডাউন থাকলেও DNS সেই IP দিতে পারে           │
│                                                         │
│ ❌ সমান বিতরণ নিশ্চিত নয়                                │
│    → ক্যাশিং এর কারণে কিছু সার্ভারে বেশি ট্রাফিক যেতে পারে│
│                                                         │
│ ❌ Session Persistence নেই                               │
│    → পরবর্তী request ভিন্ন সার্ভারে যেতে পারে             │
│                                                         │
│ ✅ সমাধান: AWS Route53, Cloudflare DNS                   │
│    → Health check + Weighted routing + Geo-routing       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔒 DNS Security (DNSSEC)

DNSSEC (DNS Security Extensions) DNS রেসপন্সের সত্যতা যাচাই করে।

```
DNS ছাড়া DNSSEC (ঝুঁকি):
──────────────────────────
ইউজার: "bkash.com এর IP কী?"
    │
    ▼
DNS: "103.108.12.45"  ← আসল IP
    │
    ▼ ❌ আক্রমণকারী DNS রেসপন্স পরিবর্তন করে!
    │
ইউজার পায়: "66.66.66.66"  ← ভুয়া IP (ফিশিং সাইট!)
    │
    ▼
ইউজার ভুয়া bKash সাইটে PIN দেয় → টাকা চুরি! 😱

DNSSEC সহ (সুরক্ষিত):
──────────────────────
ইউজার: "bkash.com এর IP কী?"
    │
    ▼
DNS: "103.108.12.45" + ডিজিটাল সিগনেচার (RRSIG)
    │
    ▼
ইউজারের Resolver: সিগনেচার যাচাই ✅
    │
    ▼ ❌ আক্রমণকারী রেসপন্স পরিবর্তন করলে
    │    সিগনেচার মিলবে না → রিজেক্ট! ✅
    │
ইউজার নিরাপদ 🔒
```

### DNSSEC Record Types

```
┌──────────┬────────────────────────────────────────────┐
│  Record  │                  উদ্দেশ্য                    │
├──────────┼────────────────────────────────────────────┤
│  RRSIG   │ প্রতিটি DNS record এর ডিজিটাল সিগনেচার     │
│  DNSKEY  │ সিগনেচার যাচাইয়ের জন্য পাবলিক কী            │
│  DS      │ Parent zone এ child zone এর key hash      │
│  NSEC    │ "এই record এর অস্তিত্ব নেই" প্রমাণ          │
│  NSEC3   │ NSEC এর উন্নত ভার্সন (zone walking রোধ)     │
└──────────┴────────────────────────────────────────────┘

DNSSEC Chain of Trust:
──────────────────────
Root (.) ──DS──► .bd ──DS──► daraz.com.bd
  │ DNSKEY        │ DNSKEY       │ DNSKEY
  │ RRSIG         │ RRSIG        │ RRSIG
  │               │              │
  ▼               ▼              ▼
Root KSK      .bd KSK      daraz.com.bd
  signs         signs        ZSK signs
  .bd DS        daraz DS     A, MX, etc.
```

---

## 🎯 বাস্তব উদাহরণ: daraz.com.bd ভিজিট

আপনি ব্রাউজারে `www.daraz.com.bd` টাইপ করলে পুরো প্রক্রিয়াটি:

```
Step 1: ব্রাউজার ক্যাশ চেক
────────────────────────────
Chrome ক্যাশে www.daraz.com.bd আছে? → না (প্রথমবার)

Step 2: OS ক্যাশ ও /etc/hosts চেক
──────────────────────────────────
/etc/hosts ফাইলে daraz.com.bd আছে? → না

Step 3: ISP DNS Resolver কে জিজ্ঞাসা
──────────────────────────────────────
আপনার Grameenphone DNS (203.130.193.74) কে query:
"www.daraz.com.bd এর IP কী?"

Step 4: Root Server কে জিজ্ঞাসা
─────────────────────────────────
GP DNS → Root Server (a.root-servers.net)
Root: "আমি .bd জানি না, কিন্তু .bd NS হলো: ns1.btcl.net.bd"

Step 5: .bd TLD Server কে জিজ্ঞাসা
──────────────────────────────────────
GP DNS → .bd TLD (ns1.btcl.net.bd)
.bd TLD: "daraz.com.bd এর NS হলো: ns1.cloudflare.com"

Step 6: Authoritative NS কে জিজ্ঞাসা
───────────────────────────────────────
GP DNS → Cloudflare (ns1.cloudflare.com)
Cloudflare: "www.daraz.com.bd = CNAME → daraz.com.bd"
            "daraz.com.bd = A → 103.108.12.45"

Step 7: IP পেলাম!
──────────────────
GP DNS ক্যাশ করলো (TTL: 300s)
ব্রাউজারকে জানালো: 103.108.12.45

Step 8: TCP + TLS সংযোগ
─────────────────────────
ব্রাউজার → 103.108.12.45:443 → TCP Handshake → TLS Handshake

Step 9: HTTP Request
───────────────────
GET / HTTP/2
Host: www.daraz.com.bd

Step 10: Daraz পেজ লোড! 🎉
──────────────────────────

মোট সময়: ~100-300ms (ক্যাশ না থাকলে)
           ~1-5ms (ক্যাশ থাকলে)
```

---

## 💻 কোড উদাহরণ

### PHP DNS Functions

```php
<?php
// =============================================
// PHP DNS Functions
// =============================================

// A Record Lookup
$ip = gethostbyname('www.daraz.com.bd');
echo "daraz.com.bd IP: $ip\n";
// Output: 103.108.12.45

// Reverse DNS (IP → Domain)
$host = gethostbyaddr('103.108.12.45');
echo "Reverse DNS: $host\n";

// সব DNS Records দেখা
$records = dns_get_record('daraz.com.bd', DNS_ALL);
foreach ($records as $record) {
    echo "Type: {$record['type']} | ";
    
    switch ($record['type']) {
        case 'A':
            echo "IP: {$record['ip']}";
            break;
        case 'AAAA':
            echo "IPv6: {$record['ipv6']}";
            break;
        case 'CNAME':
            echo "Target: {$record['target']}";
            break;
        case 'MX':
            echo "Mail: {$record['target']} (Priority: {$record['pri']})";
            break;
        case 'NS':
            echo "Nameserver: {$record['target']}";
            break;
        case 'TXT':
            echo "Text: {$record['txt']}";
            break;
    }
    
    echo " | TTL: {$record['ttl']}s\n";
}

// নির্দিষ্ট Record Type lookup
$mxRecords = dns_get_record('daraz.com.bd', DNS_MX);
echo "\nMX Records:\n";
foreach ($mxRecords as $mx) {
    echo "  Priority: {$mx['pri']} → {$mx['target']}\n";
}

// DNS Resolution সময় মাপা
function measureDNS(string $domain): array
{
    $start = microtime(true);
    $ip = gethostbyname($domain);
    $time = (microtime(true) - $start) * 1000;
    
    return [
        'domain' => $domain,
        'ip' => $ip,
        'time_ms' => round($time, 2)
    ];
}

$sites = ['daraz.com.bd', 'bkash.com', 'pathao.com', 'google.com'];
foreach ($sites as $site) {
    $result = measureDNS($site);
    echo "{$result['domain']} → {$result['ip']} ({$result['time_ms']}ms)\n";
}
```

### Node.js DNS Module

```javascript
// =============================================
// Node.js DNS Module
// =============================================

const dns = require('dns');
const { promisify } = require('util');

const resolve4 = promisify(dns.resolve4);
const resolve6 = promisify(dns.resolve6);
const resolveMx = promisify(dns.resolveMx);
const resolveTxt = promisify(dns.resolveTxt);
const resolveNs = promisify(dns.resolveNs);
const resolveCname = promisify(dns.resolveCname);
const reverse = promisify(dns.reverse);

async function fullDNSLookup(domain) {
    console.log(`\n🔍 DNS Lookup: ${domain}`);
    console.log('─'.repeat(50));
    
    try {
        // A Records (IPv4)
        const ipv4 = await resolve4(domain);
        console.log(`A Records (IPv4): ${ipv4.join(', ')}`);
    } catch (e) { /* no record */ }
    
    try {
        // AAAA Records (IPv6)
        const ipv6 = await resolve6(domain);
        console.log(`AAAA Records (IPv6): ${ipv6.join(', ')}`);
    } catch (e) { /* no record */ }
    
    try {
        // MX Records
        const mx = await resolveMx(domain);
        mx.sort((a, b) => a.priority - b.priority);
        console.log('MX Records:');
        mx.forEach(r => console.log(`  Priority ${r.priority}: ${r.exchange}`));
    } catch (e) { /* no record */ }
    
    try {
        // NS Records
        const ns = await resolveNs(domain);
        console.log(`NS Records: ${ns.join(', ')}`);
    } catch (e) { /* no record */ }
    
    try {
        // TXT Records
        const txt = await resolveTxt(domain);
        console.log('TXT Records:');
        txt.forEach(r => console.log(`  ${r.join('')}`));
    } catch (e) { /* no record */ }
    
    try {
        // CNAME (for www subdomain)
        const cname = await resolveCname(`www.${domain}`);
        console.log(`CNAME (www): ${cname.join(', ')}`);
    } catch (e) { /* no record */ }
}

// DNS Resolution Time measurement
async function measureDNSTime(domain) {
    const start = process.hrtime.bigint();
    
    return new Promise((resolve) => {
        dns.lookup(domain, (err, address, family) => {
            const end = process.hrtime.bigint();
            const timeMs = Number(end - start) / 1e6;
            
            resolve({
                domain,
                address,
                family: `IPv${family}`,
                timeMs: timeMs.toFixed(2)
            });
        });
    });
}

// Custom DNS Resolver ব্যবহার
async function useCustomDNS() {
    const resolver = new dns.Resolver();
    
    // Google DNS ব্যবহার
    resolver.setServers(['8.8.8.8', '8.8.4.4']);
    
    const resolve = promisify(resolver.resolve4.bind(resolver));
    const ips = await resolve('daraz.com.bd');
    console.log('Google DNS result:', ips);
    
    // Cloudflare DNS ব্যবহার
    resolver.setServers(['1.1.1.1', '1.0.0.1']);
    const ips2 = await resolve('daraz.com.bd');
    console.log('Cloudflare DNS result:', ips2);
}

// চালানো
(async () => {
    await fullDNSLookup('daraz.com.bd');
    
    // বাংলাদেশী সাইটগুলোর DNS সময় তুলনা
    const sites = ['daraz.com.bd', 'bkash.com', 'pathao.com', 'prothomalo.com'];
    console.log('\n⏱️ DNS Resolution Time:');
    for (const site of sites) {
        const result = await measureDNSTime(site);
        console.log(`  ${result.domain} → ${result.address} (${result.timeMs}ms)`);
    }
    
    await useCustomDNS();
})();
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: খুব বেশি TTL সেট করা
   TTL: 86400 (24h) → সার্ভার মাইগ্রেশনে ২৪ ঘণ্টা লাগবে!

✅ সঠিক: ব্যবহার অনুযায়ী TTL
   স্ট্যাটিক সাইট: TTL 3600 (1h)
   API: TTL 300 (5min)
   Failover: TTL 60 (1min)

──────────────────────────────────────────────

❌ ভুল: DNS lookup কে ক্যাশ না করা (অ্যাপ্লিকেশন লেভেলে)
   // প্রতিটি API call এ DNS lookup!
   for (const item of items) {
       await fetch('https://api.bkash.com/transaction');
   }

✅ সঠিক: HTTP Keep-Alive / Connection pooling ব্যবহার
   // একটি connection reuse করুন
   const agent = new http.Agent({ keepAlive: true });
   for (const item of items) {
       await fetch('https://api.bkash.com/transaction', { agent });
   }

──────────────────────────────────────────────

❌ ভুল: /etc/hosts ফাইলে হার্ডকোড IP
   127.0.0.1  api.myapp.com  ← production এ ভুলে গেলে বিপদ!

✅ সঠিক: শুধু development এ /etc/hosts ব্যবহার, production এ DNS

──────────────────────────────────────────────

❌ ভুল: একটি মাত্র NS সার্ভার
   daraz.com.bd  NS  ns1.example.com  ← এটা ডাউন হলে পুরো সাইট ডাউন!

✅ সঠিক: একাধিক NS সার্ভার (ভিন্ন provider)
   daraz.com.bd  NS  ns1.cloudflare.com
   daraz.com.bd  NS  ns2.cloudflare.com
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

```
DNS Records কখন পরিবর্তন করবেন:
──────────────────────────────
✅ সার্ভার মাইগ্রেশন (নতুন IP)
✅ CDN সেটআপ (CNAME → CDN)
✅ ইমেইল সার্ভার পরিবর্তন (MX record)
✅ SSL certificate verification (TXT record)
✅ Load balancing সেটআপ (একাধিক A record)
✅ Subdomain তৈরি (api., cdn., mail.)

DNSSEC কখন ব্যবহার করবেন:
──────────────────────────
✅ ব্যাংকিং/আর্থিক সেবা (bKash, Nagad)
✅ সরকারি ওয়েবসাইট (.gov.bd)
✅ ই-কমার্স (Daraz)
✅ যেকোনো সংবেদনশীল ডেটা সাইট

Round-Robin DNS vs Load Balancer:
──────────────────────────────────
Round-Robin DNS → সিম্পল লোড বিতরণ, health check নেই
Load Balancer → উন্নত, health check + session persistence
বড় সিস্টেমে → দুটোই ব্যবহার করুন (DNS → LB → Servers)
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│                 DNS সারসংক্ষেপ                         │
├───────────────────────────────────────────────────────┤
│                                                       │
│  DNS = ইন্টারনেটের ফোন বুক 📖                          │
│  Domain Name → IP Address রূপান্তর                    │
│                                                       │
│  Resolution পথ:                                       │
│  Browser → OS → Router → ISP DNS →                   │
│  Root → TLD → Authoritative NS                       │
│                                                       │
│  গুরুত্বপূর্ণ Records:                                   │
│  A (IPv4), AAAA (IPv6), CNAME (Alias),               │
│  MX (Mail), NS (DNS Server), TXT (Text)              │
│                                                       │
│  পারফরম্যান্স টিপস:                                    │
│  - সঠিক TTL সেট করুন                                  │
│  - Cloudflare/Route53 ব্যবহার করুন                     │
│  - DNS prefetch (<link rel="dns-prefetch">)           │
│  - DNSSEC enable করুন (সংবেদনশীল সাইটে)               │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [TLS/SSL](./tls-ssl.md) — কীভাবে ইন্টারনেট যোগাযোগ সুরক্ষিত হয়
