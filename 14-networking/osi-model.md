# 📡 OSI মডেল (Open Systems Interconnection Model)

> নেটওয়ার্কিং বুঝতে হলে OSI মডেল হলো প্রথম ধাপ।
> এটি ৭টি লেয়ারে বিভক্ত একটি কনসেপ্চুয়াল ফ্রেমওয়ার্ক যা বোঝায় কীভাবে দুটি কম্পিউটার একে অপরের সাথে যোগাযোগ করে।

---

## 📖 সূচিপত্র

- [OSI মডেল কী?](#-osi-মডেল-কী)
- [৭ লেয়ার বিস্তারিত](#-৭-লেয়ার-বিস্তারিত)
- [পোস্টাল সিস্টেম অ্যানালজি](#-পোস্টাল-সিস্টেম-অ্যানালজি)
- [TCP/IP মডেল তুলনা](#-tcpip-মডেল-তুলনা)
- [ওয়েব রিকোয়েস্ট ফ্লো](#-ওয়েব-রিকোয়েস্ট-ফ্লো)
- [প্রতিটি লেয়ারের প্রোটোকল](#-প্রতিটি-লেয়ারের-প্রোটোকল)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন)

---

## 📌 OSI মডেল কী?

OSI মডেল হলো ISO (International Organization for Standardization) কর্তৃক তৈরি একটি রেফারেন্স মডেল। এটি নেটওয়ার্ক কমিউনিকেশনকে ৭টি অ্যাবস্ট্রাক্ট লেয়ারে ভাগ করে।

```
┌─────────────────────────────────────────────────────────┐
│                    OSI মডেল (৭ লেয়ার)                    │
├─────────┬──────────────────────┬────────────────────────┤
│ লেয়ার # │       নাম             │     দায়িত্ব             │
├─────────┼──────────────────────┼────────────────────────┤
│    7    │ Application          │ ইউজার ইন্টারফেস         │
│    6    │ Presentation         │ ডেটা ফরম্যাটিং/এনক্রিপশন │
│    5    │ Session              │ সেশন ম্যানেজমেন্ট        │
│    4    │ Transport            │ এন্ড-টু-এন্ড ডেলিভারি     │
│    3    │ Network              │ রাউটিং/অ্যাড্রেসিং        │
│    2    │ Data Link            │ ফ্রেমিং/এরর ডিটেকশন      │
│    1    │ Physical             │ বিট ট্রান্সমিশন           │
└─────────┴──────────────────────┴────────────────────────┘
```

### মনে রাখার সহজ উপায়

```
উপর থেকে নিচে: "All People Seem To Need Data Processing"
  A - Application
  P - Presentation
  S - Session
  T - Transport
  N - Network
  D - Data Link
  P - Physical

নিচ থেকে উপরে: "Please Do Not Throw Sausage Pizza Away"
  P - Physical
  D - Data Link
  N - Network
  T - Transport
  S - Session
  P - Presentation
  A - Application
```

---

## 📊 ৭ লেয়ার বিস্তারিত

### Layer 7: Application Layer (অ্যাপ্লিকেশন লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 7: Application                  │
│                                                  │
│  ইউজার সরাসরি এই লেয়ারের সাথে ইন্টারঅ্যাক্ট করে  │
│                                                  │
│  প্রোটোকল: HTTP, HTTPS, FTP, SMTP, DNS, SSH     │
│                                                  │
│  উদাহরণ:                                         │
│  - ব্রাউজারে daraz.com.bd ভিজিট করা              │
│  - bKash অ্যাপে টাকা পাঠানো                       │
│  - Gmail এ ইমেইল পাঠানো                          │
└─────────────────────────────────────────────────┘
```

**বাস্তব উদাহরণ**: আপনি যখন Daraz এ প্রোডাক্ট সার্চ করেন, ব্রাউজার HTTP প্রোটোকল ব্যবহার করে সার্ভারে রিকোয়েস্ট পাঠায়।

```javascript
// JavaScript - Application Layer এ কাজ
const response = await fetch('https://api.daraz.com.bd/products?q=phone');
const data = await response.json();
console.log(data); // ইউজার দেখতে পায় প্রোডাক্ট লিস্ট
```

```php
<?php
// PHP - Application Layer এ কাজ
$ch = curl_init('https://api.daraz.com.bd/products?q=phone');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
$data = json_decode($response, true);
print_r($data);
```

---

### Layer 6: Presentation Layer (প্রেজেন্টেশন লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 6: Presentation                 │
│                                                  │
│  ডেটা ট্রান্সলেশন, এনক্রিপশন, কম্প্রেশন          │
│                                                  │
│  কাজসমূহ:                                        │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐      │
│  │Encoding │  │Encryption│  │Compression │      │
│  │UTF-8    │  │SSL/TLS   │  │gzip/br     │      │
│  │ASCII    │  │AES       │  │deflate     │      │
│  └─────────┘  └──────────┘  └────────────┘      │
│                                                  │
│  ফরম্যাট: JPEG, PNG, JSON, XML, HTML             │
└─────────────────────────────────────────────────┘
```

**বাস্তব উদাহরণ**: bKash অ্যাপ যখন আপনার ট্রানজ্যাকশন ডেটা পাঠায়, এটি TLS দ্বারা এনক্রিপ্ট হয় এবং JSON ফরম্যাটে পাঠানো হয়।

```javascript
// JSON encoding/decoding - Presentation Layer
const transactionData = {
    sender: '01712345678',
    receiver: '01898765432',
    amount: 500
};

// Encoding (serialize)
const jsonString = JSON.stringify(transactionData);
// '{"sender":"01712345678","receiver":"01898765432","amount":500}'

// Decoding (deserialize)
const parsed = JSON.parse(jsonString);
```

---

### Layer 5: Session Layer (সেশন লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 5: Session                      │
│                                                  │
│  কানেকশন স্থাপন, ম্যানেজ ও টার্মিনেট করা         │
│                                                  │
│  Client                         Server           │
│    │     ── Session Open ──►      │               │
│    │     ◄── Acknowledge ──       │               │
│    │     ── Data Exchange ──►     │               │
│    │     ◄── Data Exchange ──     │               │
│    │     ── Session Close ──►     │               │
│    │     ◄── Acknowledge ──       │               │
│                                                  │
│  প্রোটোকল: NetBIOS, RPC, PPTP                    │
│  কনসেপ্ট: Session ID, Authentication              │
└─────────────────────────────────────────────────┘
```

**বাস্তব উদাহরণ**: Pathao অ্যাপে আপনি লগইন করলে একটি সেশন তৈরি হয়। এই সেশন ততক্ষণ থাকে যতক্ষণ আপনি লগআউট না করেন।

```php
<?php
// PHP Session - Session Layer concept
session_start();

// সেশন তৈরি (লগইন এর সময়)
$_SESSION['user_id'] = 12345;
$_SESSION['phone'] = '01712345678';
$_SESSION['role'] = 'rider';

// সেশন চেক
if (isset($_SESSION['user_id'])) {
    echo "Pathao Rider logged in: " . $_SESSION['phone'];
}

// সেশন শেষ (লগআউট)
session_destroy();
```

---

### Layer 4: Transport Layer (ট্রান্সপোর্ট লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 4: Transport                    │
│                                                  │
│  এন্ড-টু-এন্ড ডেটা ডেলিভারি নিশ্চিত করে            │
│                                                  │
│  ┌──────────────┐    ┌──────────────┐            │
│  │     TCP      │    │     UDP      │            │
│  │  Reliable    │    │    Fast      │            │
│  │  Ordered     │    │  Unordered   │            │
│  │  Connection  │    │ Connectionless│            │
│  │              │    │              │            │
│  │  HTTP, FTP   │    │  DNS, Video  │            │
│  │  SMTP, SSH   │    │  VoIP, Games │            │
│  └──────────────┘    └──────────────┘            │
│                                                  │
│  PDU: Segment                                    │
│  অ্যাড্রেসিং: Port Number                          │
└─────────────────────────────────────────────────┘
```

**বাস্তব উদাহরণ**: 
- Grameenphone এর ভিডিও কল → **UDP** (দ্রুততা দরকার, সামান্য ডেটা হারালে সমস্যা নেই)
- bKash ট্রানজ্যাকশন → **TCP** (প্রতিটি বাইট পৌঁছানো নিশ্চিত করতে হবে)

---

### Layer 3: Network Layer (নেটওয়ার্ক লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 3: Network                      │
│                                                  │
│  প্যাকেটকে সোর্স থেকে ডেস্টিনেশনে রাউট করে       │
│                                                  │
│  ┌────────┐     ┌────────┐     ┌────────┐       │
│  │ Host A │────►│ Router │────►│ Host B │       │
│  │10.0.0.1│     │        │     │10.0.0.2│       │
│  └────────┘     └────────┘     └────────┘       │
│                                                  │
│  প্রোটোকল: IP (IPv4/IPv6), ICMP, OSPF, BGP      │
│  ডিভাইস: Router, Layer 3 Switch                  │
│  PDU: Packet                                     │
│  অ্যাড্রেসিং: IP Address                           │
└─────────────────────────────────────────────────┘
```

**IP Address এর ধরন**:
```
IPv4: 103.108.12.45    (৩২ বিট, ৪.৩ বিলিয়ন ঠিকানা)
IPv6: 2001:0db8:85a3::8a2e:0370:7334  (১২৮ বিট, প্রায় অসীম)

বাংলাদেশের উদাহরণ:
- Grameenphone DNS: 203.130.193.74
- BTCL DNS: 203.112.200.5
```

---

### Layer 2: Data Link Layer (ডেটা লিংক লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 2: Data Link                    │
│                                                  │
│  নোড-টু-নোড ডেটা ট্রান্সফার ও এরর ডিটেকশন       │
│                                                  │
│  দুটি সাব-লেয়ার:                                  │
│  ┌─────────────────────────────────────┐        │
│  │ LLC (Logical Link Control)          │        │
│  │ - Flow control                      │        │
│  │ - Error detection                   │        │
│  ├─────────────────────────────────────┤        │
│  │ MAC (Media Access Control)          │        │
│  │ - MAC addressing                    │        │
│  │ - Frame transmission                │        │
│  └─────────────────────────────────────┘        │
│                                                  │
│  প্রোটোকল: Ethernet, Wi-Fi (802.11), PPP         │
│  ডিভাইস: Switch, Bridge                          │
│  PDU: Frame                                      │
│  অ্যাড্রেসিং: MAC Address                          │
└─────────────────────────────────────────────────┘
```

**MAC Address উদাহরণ**:
```
MAC: AA:BB:CC:DD:EE:FF  (৪৮ বিট, প্রতিটি NIC এর জন্য ইউনিক)

আপনার ল্যাপটপের Wi-Fi কার্ড: 4C:ED:FB:12:34:56
আপনার রাউটার:              DC:A6:32:AB:CD:EF
```

---

### Layer 1: Physical Layer (ফিজিক্যাল লেয়ার)

```
┌─────────────────────────────────────────────────┐
│            Layer 1: Physical                     │
│                                                  │
│  বিট (0 এবং 1) কে ইলেক্ট্রিক্যাল/অপটিক্যাল      │
│  সিগন্যালে রূপান্তর করে                           │
│                                                  │
│  মাধ্যম:                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐     │
│  │ তামার তার │ │ ফাইবার   │ │ ওয়্যারলেস     │     │
│  │ (Cat5/6)  │ │ অপটিক    │ │ (Wi-Fi/5G)   │     │
│  │ 1 Gbps   │ │ 100 Gbps │ │ 1-10 Gbps    │     │
│  └──────────┘ └──────────┘ └──────────────┘     │
│                                                  │
│  ডিভাইস: Hub, Repeater, Cable, Antenna           │
│  PDU: Bit                                        │
└─────────────────────────────────────────────────┘
```

**বাংলাদেশী উদাহরণ**:
- ঢাকা-চট্টগ্রাম সাবমেরিন কেবল (ফাইবার অপটিক)
- গ্রামীণফোনের 4G/5G টাওয়ার (ওয়্যারলেস)
- বাসার broadband কেবল (Cat5e/Cat6)

---

## 🏠 পোস্টাল সিস্টেম অ্যানালজি

আসুন OSI মডেলকে বাংলাদেশের ডাক ব্যবস্থার সাথে তুলনা করি:

```
🏠 আপনি একটি চিঠি পাঠাচ্ছেন ঢাকা থেকে চট্টগ্রামে

┌─────────────────────────────────────────────────────────────────┐
│ Layer 7 (Application)                                           │
│ → আপনি বাংলায় চিঠি লিখলেন                                      │
│   "প্রিয় করিম ভাই, কেমন আছেন?"                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 6 (Presentation)                                          │
│ → চিঠি ভাঁজ করলেন, খামে ঢোকালেন                                 │
│   (Data Formatting / Encoding)                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 5 (Session)                                               │
│ → চিঠির রেফারেন্স নম্বর দিলেন: "Letter #001"                    │
│   (Session tracking)                                            │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4 (Transport)                                             │
│ → রেজিস্ট্রি ডাক বেছে নিলেন (TCP = নিশ্চিত ডেলিভারি)           │
│   বা সাধারণ ডাক (UDP = দ্রুত কিন্তু গ্যারান্টি নেই)              │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3 (Network)                                               │
│ → খামে ঠিকানা লিখলেন:                                           │
│   To: বাড়ি ৫, রোড ৩, চট্টগ্রাম (Destination IP)                 │
│   From: বাড়ি ১০, রোড ৭, ঢাকা (Source IP)                       │
│   → পোস্ট অফিস রাউট ঠিক করবে                                   │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2 (Data Link)                                             │
│ → পোস্টম্যান নিকটতম পোস্ট অফিসে নিয়ে গেল                       │
│   (MAC = পোস্টম্যানের ID, Hop-by-Hop delivery)                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1 (Physical)                                              │
│ → ভ্যান/ট্রাক/ট্রেনে চিঠি ঢাকা থেকে চট্টগ্রামে গেল              │
│   (তামার তার / ফাইবার / ওয়্যারলেস)                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 TCP/IP মডেল তুলনা

TCP/IP মডেল হলো বাস্তবে ব্যবহৃত মডেল। OSI মডেল একটি থিওরেটিক্যাল রেফারেন্স।

```
     OSI মডেল (৭ লেয়ার)              TCP/IP মডেল (৪ লেয়ার)
    ┌──────────────────┐
    │  7. Application  │
    ├──────────────────┤         ┌──────────────────┐
    │  6. Presentation │ ───────►│  4. Application  │
    ├──────────────────┤         │   (HTTP, FTP,    │
    │  5. Session      │         │    DNS, SMTP)    │
    └──────────────────┘         └──────────────────┘
    ┌──────────────────┐         ┌──────────────────┐
    │  4. Transport    │ ───────►│  3. Transport    │
    │                  │         │   (TCP, UDP)     │
    └──────────────────┘         └──────────────────┘
    ┌──────────────────┐         ┌──────────────────┐
    │  3. Network      │ ───────►│  2. Internet     │
    │                  │         │   (IP, ICMP)     │
    └──────────────────┘         └──────────────────┘
    ┌──────────────────┐
    │  2. Data Link    │         ┌──────────────────┐
    ├──────────────────┤ ───────►│  1. Network      │
    │  1. Physical     │         │   Access         │
    └──────────────────┘         │  (Ethernet,WiFi) │
                                 └──────────────────┘
```

### তুলনা টেবিল

```
┌─────────────────┬──────────────────┬──────────────────┐
│    বৈশিষ্ট্য      │    OSI মডেল       │   TCP/IP মডেল    │
├─────────────────┼──────────────────┼──────────────────┤
│ লেয়ার সংখ্যা     │       ৭          │        ৪         │
│ উদ্ভব           │      ISO         │      DARPA       │
│ পদ্ধতি          │  Theory first    │ Implementation   │
│                 │                  │     first        │
│ ব্যবহার         │  রেফারেন্স মডেল   │  বাস্তব ইন্টারনেট │
│ প্রোটোকল        │  জেনেরিক         │   স্পেসিফিক      │
│ Session লেয়ার   │     আলাদা         │  Application এ   │
│                 │                  │    মিশ্রিত        │
└─────────────────┴──────────────────┴──────────────────┘
```

---

## 🔗 ওয়েব রিকোয়েস্ট ফ্লো

আপনি যখন `https://www.daraz.com.bd` ভিজিট করেন, ডেটা প্রতিটি লেয়ারের মধ্য দিয়ে যায়:

### প্রেরক (আপনার ব্রাউজার) - Encapsulation

```
Layer 7: Application
    ব্রাউজার HTTP GET রিকোয়েস্ট তৈরি করে
    GET / HTTP/1.1
    Host: www.daraz.com.bd
         │
         ▼
Layer 6: Presentation
    ডেটা SSL/TLS দিয়ে এনক্রিপ্ট হয়
    JSON/HTML encoding নির্ধারিত হয়
         │
         ▼
Layer 5: Session
    TLS সেশন স্থাপিত হয়
    Session ID তৈরি হয়
         │
         ▼
Layer 4: Transport
    TCP সেগমেন্ট তৈরি হয়
    ┌──────────┬──────────────────────────┐
    │ TCP Hdr  │       HTTP Data          │
    │ Src:5000 │  GET / HTTP/1.1 ...      │
    │ Dst:443  │                          │
    └──────────┴──────────────────────────┘
         │
         ▼
Layer 3: Network
    IP প্যাকেট তৈরি হয়
    ┌──────────┬──────────┬───────────────┐
    │  IP Hdr  │ TCP Hdr  │  HTTP Data    │
    │Src:      │          │               │
    │192.168.1.│          │               │
    │100       │          │               │
    │Dst:      │          │               │
    │103.108.  │          │               │
    │12.45     │          │               │
    └──────────┴──────────┴───────────────┘
         │
         ▼
Layer 2: Data Link
    ইথারনেট ফ্রেম তৈরি হয়
    ┌──────────┬──────────┬──────────┬─────────┬─────┐
    │ Eth Hdr  │  IP Hdr  │ TCP Hdr  │  Data   │ FCS │
    │ MAC Addr │          │          │         │     │
    └──────────┴──────────┴──────────┴─────────┴─────┘
         │
         ▼
Layer 1: Physical
    বিট স্ট্রিম (0 এবং 1) তারের মাধ্যমে পাঠানো হয়
    01101001 01010110 11001010 ...
```

### প্রাপক (Daraz সার্ভার) - Decapsulation

```
Layer 1: Physical
    বিট রিসিভ করে
    01101001 01010110 11001010 ...
         │
         ▼ (Ethernet Header সরানো হয়)
Layer 2: Data Link
    ফ্রেম যাচাই, MAC চেক
         │
         ▼ (IP Header সরানো হয়)
Layer 3: Network
    IP ঠিকানা যাচাই, সঠিক মেশিনে পৌঁছানো
         │
         ▼ (TCP Header সরানো হয়)
Layer 4: Transport
    পোর্ট 443 এ ডেটা পাঠানো, সিকোয়েন্স চেক
         │
         ▼
Layer 5: Session
    TLS সেশন যাচাই
         │
         ▼
Layer 6: Presentation
    ডেটা ডিক্রিপ্ট, ডিকোড
         │
         ▼
Layer 7: Application
    ওয়েব সার্ভার (Nginx/Apache) HTTP রিকোয়েস্ট প্রসেস করে
    HTML রেসপন্স পাঠায়
```

---

## 📖 প্রতিটি লেয়ারের প্রোটোকল

```
┌────────────┬────────────────────────────────────────────────────┐
│   লেয়ার    │                    প্রোটোকল                        │
├────────────┼────────────────────────────────────────────────────┤
│ Application│ HTTP, HTTPS, FTP, SFTP, SSH, Telnet, SMTP,        │
│     (7)    │ POP3, IMAP, DNS, DHCP, SNMP, LDAP, MQTT          │
├────────────┼────────────────────────────────────────────────────┤
│Presentation│ SSL/TLS, JPEG, GIF, PNG, MPEG, ASCII, UTF-8,     │
│     (6)    │ JSON, XML, gzip, brotli                           │
├────────────┼────────────────────────────────────────────────────┤
│  Session   │ NetBIOS, RPC, PPTP, L2TP, SIP, NFS               │
│     (5)    │                                                    │
├────────────┼────────────────────────────────────────────────────┤
│ Transport  │ TCP, UDP, SCTP, DCCP, QUIC                        │
│     (4)    │                                                    │
├────────────┼────────────────────────────────────────────────────┤
│  Network   │ IPv4, IPv6, ICMP, IGMP, IPSec, OSPF, BGP, RIP   │
│     (3)    │                                                    │
├────────────┼────────────────────────────────────────────────────┤
│ Data Link  │ Ethernet (802.3), Wi-Fi (802.11), PPP, HDLC,     │
│     (2)    │ ARP, VLAN (802.1Q), STP                           │
├────────────┼────────────────────────────────────────────────────┤
│  Physical  │ Ethernet cable, Fiber optic, USB, Bluetooth,      │
│     (1)    │ DSL, RS-232, 802.11 (physical)                    │
└────────────┴────────────────────────────────────────────────────┘
```

---

## 🎯 প্রতিটি লেয়ারের PDU (Protocol Data Unit)

```
┌──────────────┬────────────────┐
│    লেয়ার      │      PDU       │
├──────────────┼────────────────┤
│ Application  │    Data        │
│ Presentation │    Data        │
│ Session      │    Data        │
│ Transport    │    Segment     │
│ Network      │    Packet      │
│ Data Link    │    Frame       │
│ Physical     │    Bit         │
└──────────────┴────────────────┘

মনে রাখুন: "Do Some People Fear Birthdays?"
  D - Data
  S - Segment
  P - Packet
  F - Frame
  B - Bit
```

---

## 🎯 কোড উদাহরণ: নেটওয়ার্ক ডিবাগিং

### PHP - প্রতিটি লেয়ারের তথ্য দেখা

```php
<?php
// Layer 3 - IP Address তথ্য
echo "Server IP: " . $_SERVER['SERVER_ADDR'] . "\n";
echo "Client IP: " . $_SERVER['REMOTE_ADDR'] . "\n";

// Layer 4 - Port তথ্য
echo "Server Port: " . $_SERVER['SERVER_PORT'] . "\n";
echo "Client Port: " . $_SERVER['REMOTE_PORT'] . "\n";

// Layer 7 - HTTP তথ্য
echo "HTTP Method: " . $_SERVER['REQUEST_METHOD'] . "\n";
echo "Host: " . $_SERVER['HTTP_HOST'] . "\n";
echo "User-Agent: " . $_SERVER['HTTP_USER_AGENT'] . "\n";

// DNS Lookup (Layer 7 → Layer 3)
$ip = gethostbyname('www.daraz.com.bd');
echo "daraz.com.bd IP: $ip\n";

// Get all DNS records
$records = dns_get_record('daraz.com.bd', DNS_ALL);
print_r($records);
```

### JavaScript (Node.js) - নেটওয়ার্ক তথ্য

```javascript
const dns = require('dns');
const os = require('os');
const net = require('net');

// Layer 1/2 - Network Interfaces
const interfaces = os.networkInterfaces();
for (const [name, addrs] of Object.entries(interfaces)) {
    for (const addr of addrs) {
        console.log(`Interface: ${name}`);
        console.log(`  MAC (Layer 2): ${addr.mac}`);
        console.log(`  IP (Layer 3): ${addr.address}`);
        console.log(`  Family: ${addr.family}`);
    }
}

// Layer 3 - DNS Lookup
dns.lookup('www.daraz.com.bd', (err, address, family) => {
    console.log(`\ndaraz.com.bd → ${address} (IPv${family})`);
});

// Layer 4 - TCP Connection Test
const socket = new net.Socket();
socket.connect(443, 'www.daraz.com.bd', () => {
    console.log(`\nTCP Connected to daraz.com.bd:443`);
    console.log(`Local: ${socket.localAddress}:${socket.localPort}`);
    console.log(`Remote: ${socket.remoteAddress}:${socket.remotePort}`);
    socket.destroy();
});
```

---

## ❌ সাধারণ ভুল ধারণা

```
❌ ভুল: "OSI মডেলের প্রতিটি লেয়ার আলাদা সফটওয়্যার"
✅ সঠিক: OSI একটি কনসেপ্চুয়াল মডেল, বাস্তবে TCP/IP মডেল ব্যবহৃত হয়

❌ ভুল: "ডেটা সরাসরি Layer 7 থেকে Layer 7 এ যায়"
✅ সঠিক: ডেটা প্রথমে Layer 1 পর্যন্ত Encapsulate হয়,
         তারপর নেটওয়ার্কে ভ্রমণ করে, তারপর Decapsulate হয়

❌ ভুল: "HTTP হলো Layer 4 প্রোটোকল"
✅ সঠিক: HTTP হলো Layer 7 (Application) প্রোটোকল,
         এটি Layer 4 এ TCP ব্যবহার করে

❌ ভুল: "MAC Address পরিবর্তন করা যায় না"
✅ সঠিক: সফটওয়্যার দিয়ে MAC Address স্পুফ করা যায়,
         তবে হার্ডওয়্যারে বার্ন করা আসল MAC অপরিবর্তনীয়
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### কখন OSI মডেল জ্ঞান দরকার

```
✅ নেটওয়ার্ক সমস্যা ডিবাগ করতে
   → "সমস্যাটা কোন লেয়ারে?" এভাবে চিন্তা করুন

✅ সিস্টেম ডিজাইন ইন্টারভিউতে
   → "আমাদের সিস্টেম কোন লেয়ারে কাজ করে" বোঝাতে

✅ সিকিউরিটি বিশ্লেষণে
   → প্রতিটি লেয়ারে ভিন্ন ধরনের আক্রমণ হয়

✅ ক্লাউড আর্কিটেকচারে
   → VPC, Subnet, Security Group সব Layer 3/4 কনসেপ্ট
```

### ডিবাগিং গাইড

```
সমস্যা: ওয়েবসাইট লোড হচ্ছে না

Layer 1: তার/Wi-Fi সংযোগ আছে? ⚡
         → "ক্যাবল লাগানো আছে?" "Wi-Fi কানেক্টেড?"

Layer 2: NIC কাজ করছে? 🔌
         → `ip link show` বা `ifconfig`

Layer 3: IP আছে? রাউটিং ঠিক আছে? 🗺️
         → `ping 8.8.8.8` (Google DNS)

Layer 4: পোর্ট ওপেন? TCP কানেকশন হচ্ছে? 🚪
         → `telnet daraz.com.bd 443`

Layer 5-6: TLS হ্যান্ডশেক সফল? 🔐
         → `openssl s_client -connect daraz.com.bd:443`

Layer 7: HTTP রেসপন্স আসছে? 🌐
         → `curl -v https://www.daraz.com.bd`
```

---

## 📊 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────┐
│              OSI মডেল - মূল পয়েন্ট                   │
├─────────────────────────────────────────────────────┤
│ ১. ৭টি লেয়ার: Application → Physical               │
│ ২. প্রতিটি লেয়ারের নিজস্ব PDU আছে                   │
│ ৩. Encapsulation (পাঠানো) / Decapsulation (গ্রহণ)   │
│ ৪. বাস্তবে TCP/IP মডেল (৪ লেয়ার) ব্যবহৃত হয়        │
│ ৫. সমস্যা ডিবাগে লেয়ার-বাই-লেয়ার চিন্তা করুন       │
│ ৬. প্রতিটি লেয়ারের নিজস্ব প্রোটোকল ও ডিভাইস আছে    │
└─────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [TCP ও UDP](./tcp-udp.md) — ট্রান্সপোর্ট লেয়ারের গভীর আলোচনা
