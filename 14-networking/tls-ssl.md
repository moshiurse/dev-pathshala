# 🔒 TLS/SSL (Transport Layer Security)

> TLS (Transport Layer Security) হলো ইন্টারনেটে সুরক্ষিত যোগাযোগের ভিত্তি।
> যখন আপনি bKash এ টাকা পাঠান বা Daraz এ ক্রেডিট কার্ড তথ্য দেন — TLS সেটি সুরক্ষিত রাখে।

---

## 📖 সূচিপত্র

- [TLS/SSL কী?](#-tlsssl-কী)
- [TLS ইতিহাস](#-tls-ইতিহাস)
- [TLS 1.2 Handshake](#-tls-12-handshake)
- [TLS 1.3 Handshake](#-tls-13-handshake)
- [Certificate Chain](#-certificate-chain)
- [Certificate Types](#-certificate-types)
- [Let's Encrypt](#-lets-encrypt)
- [mTLS](#-mtls-mutual-tls)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 TLS/SSL কী?

TLS (আগে SSL নামে পরিচিত) তিনটি মূল সুরক্ষা প্রদান করে:

```
┌─────────────────────────────────────────────────────────┐
│                 TLS এর তিন স্তম্ভ                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🔐 Encryption (গোপনীয়তা)                               │
│  ├── ডেটা এনক্রিপ্ট করে, তৃতীয় পক্ষ পড়তে পারে না       │
│  ├── AES-256-GCM, ChaCha20                              │
│  └── উদাহরণ: bKash এ PIN ও টাকার তথ্য এনক্রিপ্টেড       │
│                                                         │
│  ✅ Authentication (পরিচয় যাচাই)                         │
│  ├── সার্ভার আসলেই সেই সার্ভার কিনা যাচাই                  │
│  ├── X.509 Certificate                                   │
│  └── উদাহরণ: bkash.com সত্যিই bKash এর কিনা             │
│                                                         │
│  🛡️ Integrity (অখণ্ডতা)                                  │
│  ├── ডেটা মাঝপথে পরিবর্তন হয়নি তা নিশ্চিত                │
│  ├── HMAC, SHA-256                                       │
│  └── উদাহরণ: ৫০০ টাকা মাঝে ৫০,০০০ হয়নি নিশ্চিত          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### TLS ছাড়া vs সহ

```
HTTP (TLS ছাড়া):
──────────────────
আপনার ফোন                  হ্যাকার (মাঝখানে)            bKash সার্ভার
    │                           │                         │
    │── PIN: 12345 ────────────►│ 👀 "PIN দেখতে পাচ্ছি!"   │
    │── Amount: 50000 ─────────►│ 👀 "৫০,০০০ টাকা!"       │
    │                           │── পরিবর্তিত ডেটা ───────►│
    │                           │                         │

HTTPS (TLS সহ):
─────────────────
আপনার ফোন                  হ্যাকার (মাঝখানে)            bKash সার্ভার
    │                           │                         │
    │── 🔒 a8f3c2d1e5... ────►│ 🤷 "কিছুই বুঝতে পারছি না"│
    │── 🔒 7b2e9f4a1c... ────►│ 🤷 "Encrypted gibberish" │
    │                           │                         │
    │                    ◄──────────── 🔒 Encrypted ──────│
    │                           │                         │
    ✅ PIN ও টাকার তথ্য সুরক্ষিত│                         │
```

---

## 📊 TLS ইতিহাস

```
┌──────────┬──────────┬──────────────────────────────────────┐
│ Version  │   সাল    │              স্ট্যাটাস                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ SSL 1.0  │  ১৯৯৪    │ ❌ কখনো প্রকাশ হয়নি (গুরুতর ত্রুটি)  │
│ SSL 2.0  │  ১৯৯৫    │ ❌ Deprecated (অনেক দুর্বলতা)          │
│ SSL 3.0  │  ১৯৯৬    │ ❌ Deprecated (POODLE আক্রমণ)         │
│ TLS 1.0  │  ১৯৯৯    │ ❌ Deprecated (২০২০ এ)                │
│ TLS 1.1  │  ২০০৬    │ ❌ Deprecated (২০২০ এ)                │
│ TLS 1.2  │  ২০০৮    │ ✅ এখনো ব্যবহৃত, নিরাপদ               │
│ TLS 1.3  │  ২০১৮    │ ✅ সর্বশেষ, সবচেয়ে নিরাপদ ও দ্রুত     │
└──────────┴──────────┴──────────────────────────────────────┘

                 SSL                        TLS
          ┌──────────────────┐    ┌──────────────────────┐
          │ 1.0  2.0  3.0   │    │ 1.0  1.1  1.2  1.3  │
          │  ❌   ❌    ❌    │    │  ❌   ❌    ✅    ✅   │
          └──────────────────┘    └──────────────────────┘
           Netscape তৈরি            IETF তৈরি

  💡 দৈনন্দিন কথায় "SSL" বললেও আসলে "TLS" ই ব্যবহৃত হয়
```

---

## 🔗 TLS 1.2 Handshake

TLS 1.2 হ্যান্ডশেকে **২ RTT** লাগে:

```
Client (bKash App)                    Server (bKash API)
       │                                    │
       │  ──── ClientHello ───────────────► │
       │  │ TLS Version: 1.2                │
       │  │ Random: client_random           │
       │  │ Cipher Suites:                  │
       │  │   TLS_ECDHE_RSA_WITH_AES_256.. │
       │  │   TLS_ECDHE_RSA_WITH_AES_128.. │
       │  │ Compression Methods             │
       │  │ Extensions (SNI, ALPN)          │
       │                                    │
       │  ◄──── ServerHello ──────────────  │  ┐
       │  │ TLS Version: 1.2                │  │
       │  │ Random: server_random           │  │
       │  │ Selected Cipher:                │  │
       │  │   TLS_ECDHE_RSA_WITH_AES_256.. │  │
       │                                    │  │ 1st RTT
       │  ◄──── Certificate ──────────────  │  │
       │  │ Server's X.509 Certificate      │  │
       │  │ (bkash.com সার্টিফিকেট)          │  │
       │                                    │  │
       │  ◄──── ServerKeyExchange ────────  │  │
       │  │ ECDHE Parameters                │  │
       │                                    │  │
       │  ◄──── ServerHelloDone ──────────  │  ┘
       │                                    │
       │  ──── ClientKeyExchange ─────────► │  ┐
       │  │ Client's ECDHE public key       │  │
       │                                    │  │
       │  ──── ChangeCipherSpec ──────────► │  │ 2nd RTT
       │  │ "এখন থেকে encrypted!"           │  │
       │                                    │  │
       │  ──── Finished (Encrypted) ─────► │  │
       │                                    │  │
       │  ◄──── ChangeCipherSpec ─────────  │  │
       │  ◄──── Finished (Encrypted) ────  │  ┘
       │                                    │
       │  ═══ Encrypted Data Exchange ═══  │
       │                                    │

  মোট: ২ RTT (Round Trip Time)
  Key Exchange: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
  Encryption: AES-256-GCM
```

---

## 🔗 TLS 1.3 Handshake

TLS 1.3 অনেক দ্রুত — মাত্র **১ RTT** (বা 0-RTT পুনঃসংযোগে):

```
Client (bKash App)                    Server (bKash API)
       │                                    │
       │  ──── ClientHello ───────────────► │
       │  │ TLS Version: 1.3                │
       │  │ Random: client_random           │
       │  │ Supported Cipher Suites:        │
       │  │   TLS_AES_256_GCM_SHA384       │
       │  │   TLS_CHACHA20_POLY1305_SHA256 │
       │  │ Key Share: ECDHE public key     │  ← TLS 1.2 তে
       │  │   (আগেই key পাঠাচ্ছে!)          │     এটা পরে হতো
       │                                    │
       │  ◄──── ServerHello ──────────────  │  ┐
       │  │ TLS Version: 1.3                │  │
       │  │ Random: server_random           │  │
       │  │ Selected Cipher:                │  │
       │  │   TLS_AES_256_GCM_SHA384       │  │
       │  │ Key Share: ECDHE public key     │  │
       │                                    │  │
       │  ◄──── {EncryptedExtensions} ────  │  │ 1 RTT!
       │  ◄──── {Certificate} ────────────  │  │
       │  ◄──── {CertificateVerify} ──────  │  │
       │  ◄──── {Finished} ───────────────  │  │
       │                                    │  │
       │  ──── {Finished} ────────────────► │  ┘
       │                                    │
       │  ═══ Encrypted Data Exchange ═══  │

  মোট: ১ RTT! (TLS 1.2 এর অর্ধেক সময়)

  {} = Encrypted (সার্ভার Certificate ও encrypted!)
```

### 0-RTT Resumption (পুনঃসংযোগ)

```
পূর্বে সংযুক্ত ক্লায়েন্ট:
──────────────────────────

Client                               Server
  │                                    │
  │  ──── ClientHello ───────────────► │
  │  │ + Early Data (0-RTT)            │  ← আগের session এর
  │  │ + "আমি আগে এসেছিলাম,            │    PSK (Pre-Shared Key)
  │  │    এই হলো আমার ডেটা"             │    ব্যবহার করে!
  │                                    │
  │  ◄──── ServerHello ──────────────  │
  │  ◄──── {Finished} ───────────────  │
  │                                    │
  │  ══ ০ RTT তে ডেটা পৌঁছে গেছে! ══  │

  ⚠️ 0-RTT সতর্কতা: Replay attack ঝুঁকি আছে
  শুধু idempotent request (GET) এ ব্যবহার করুন, POST এ নয়
```

### TLS 1.2 vs 1.3 তুলনা

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │    TLS 1.2       │    TLS 1.3       │
├────────────────────┼──────────────────┼──────────────────┤
│ Handshake RTT      │ ২ RTT            │ ১ RTT            │
│ Resumption         │ Session ID/      │ 0-RTT PSK        │
│                    │ Ticket (1 RTT)   │                  │
│ Key Exchange       │ RSA, DHE, ECDHE  │ শুধু ECDHE/DHE   │
│                    │ (RSA অনিরাপদ)    │ (Forward Secrecy)│
│ Cipher Suites      │ ৩৭+ (অনেক দুর্বল)│ মাত্র ৫ (সব শক্ত) │
│ Server Certificate │ Plaintext        │ Encrypted 🔒     │
│ Hash Functions     │ MD5, SHA-1 allowed│ শুধু SHA-256+    │
│ 0-RTT             │ ❌               │ ✅               │
│ Forward Secrecy    │ Optional         │ বাধ্যতামূলক       │
└────────────────────┴──────────────────┴──────────────────┘

Forward Secrecy কেন গুরুত্বপূর্ণ:
──────────────────────────────────
TLS 1.2 (RSA key exchange):
  আজ: encrypted ডেটা ক্যাপচার 📦
  ভবিষ্যৎ: সার্ভার private key চুরি হলে → সব পুরানো ডেটা ডিক্রিপ্ট! ❌

TLS 1.3 (ECDHE - Ephemeral keys):
  প্রতিটি session এ নতুন key তৈরি হয়
  সার্ভার key চুরি হলেও → পুরানো session ডিক্রিপ্ট করা অসম্ভব ✅
```

---

## 📜 Certificate Chain

TLS সার্টিফিকেট একটি **Chain of Trust** এ কাজ করে:

```
┌─────────────────────────────────────────────────────────┐
│                  Certificate Chain                        │
│                                                          │
│  ┌──────────────────────────┐                            │
│  │    Root CA Certificate   │  ← ব্রাউজারে pre-installed │
│  │    (DigiCert, Let's      │     (~150+ Root CAs)       │
│  │     Encrypt, Comodo)     │                            │
│  │    Self-signed           │                            │
│  └────────────┬─────────────┘                            │
│               │ signs                                    │
│  ┌────────────▼─────────────┐                            │
│  │  Intermediate CA Cert    │  ← Root CA কে সুরক্ষিত     │
│  │  (R3, E1, etc.)          │     রাখতে মধ্যবর্তী CA      │
│  │  Signed by Root CA       │                            │
│  └────────────┬─────────────┘                            │
│               │ signs                                    │
│  ┌────────────▼─────────────┐                            │
│  │  Server Certificate      │  ← আপনার ওয়েবসাইটের cert │
│  │  (bkash.com)             │                            │
│  │  Signed by Intermediate  │                            │
│  └──────────────────────────┘                            │
│                                                          │
└─────────────────────────────────────────────────────────┘

যাচাই প্রক্রিয়া:
──────────────────
ব্রাউজার: bkash.com এর cert পেলাম
    │
    ├── cert কে Intermediate CA sign করেছে?
    │   └── হ্যাঁ ✅ → Intermediate cert যাচাই
    │
    ├── Intermediate কে Root CA sign করেছে?
    │   └── হ্যাঁ ✅ → Root CA ব্রাউজারে trusted?
    │
    ├── Root CA trusted?
    │   └── হ্যাঁ ✅ (DigiCert ব্রাউজারে pre-installed)
    │
    └── পুরো chain verified! 🔒 সংযোগ নিরাপদ
```

### X.509 Certificate এর বিষয়বস্তু

```
Certificate:
    Data:
        Version: 3
        Serial Number: 04:00:00:00:00:01:2F:4E:E1:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Let's Encrypt, CN=R3
        Validity:
            Not Before: Jan  1 00:00:00 2024 GMT
            Not After : Apr  1 00:00:00 2024 GMT  ← ৯০ দিন!
        Subject: CN=bkash.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
            Public-Key: (256 bit)
                04:3e:7b:...
        X509v3 Extensions:
            Subject Alternative Name:
                DNS:bkash.com
                DNS:www.bkash.com
                DNS:api.bkash.com
            Authority Info Access:
                OCSP - URI:http://ocsp.r3.letsencrypt.org
            Certificate Policies:
                Policy: 2.23.140.1.2.1  (DV)
    Signature Algorithm: sha256WithRSAEncryption
        5a:7b:c3:...
```

---

## 📋 Certificate Types

```
┌──────────┬──────────────────┬──────────┬──────────┬──────────────┐
│   Type   │     যাচাই         │  সময়     │   মূল্য    │   উদাহরণ      │
├──────────┼──────────────────┼──────────┼──────────┼──────────────┤
│    DV    │ Domain শুধু       │ মিনিট    │  বিনামূল্যে │ ব্লগ, ছোট    │
│ Domain   │ ডোমেইন মালিকানা   │          │ (Let's   │ ওয়েবসাইট     │
│Validated │ যাচাই            │          │ Encrypt) │              │
├──────────┼──────────────────┼──────────┼──────────┼──────────────┤
│    OV    │ Domain +         │ ১-৩ দিন  │ $50-     │ ব্যবসা,      │
│ Org.     │ Organization     │          │ $200/yr  │ Daraz,       │
│Validated │ প্রতিষ্ঠান যাচাই   │          │          │ Pathao       │
├──────────┼──────────────────┼──────────┼──────────┼──────────────┤
│    EV    │ Domain + Org +   │ ১-৪ সপ্তাহ│ $200-    │ ব্যাংক,      │
│Extended  │ Legal + Physical │          │ $1000/yr │ bKash,       │
│Validated │ আইনি যাচাই        │          │          │ Nagad        │
└──────────┴──────────────────┴──────────┴──────────┴──────────────┘

ব্রাউজারে দেখতে কেমন:
──────────────────────

DV: 🔒 bkash.com
OV: 🔒 bkash.com (Certificate details এ org দেখা যায়)
EV: 🔒 bKash Limited [BD] (আগে সবুজ বার দেখাতো)
```

---

## 🏆 Let's Encrypt

Let's Encrypt হলো বিনামূল্যে, স্বয়ংক্রিয় DV সার্টিফিকেট প্রদানকারী। ACME প্রোটোকল ব্যবহার করে।

```
Let's Encrypt কীভাবে কাজ করে:
─────────────────────────────

Step 1: Certbot ইনস্টল ও চালানো
  $ sudo certbot --nginx -d bkash.com -d www.bkash.com

Step 2: ACME Challenge
  Let's Encrypt: "প্রমাণ করো যে তুমি bkash.com এর মালিক"

  HTTP-01 Challenge:
  ┌──────────────────────────────────────────────────┐
  │ Let's Encrypt সার্ভার:                            │
  │ "http://bkash.com/.well-known/acme-challenge/    │
  │  abc123xyz এ এই token রাখো"                      │
  │                                                   │
  │ Certbot: ✅ token রাখলাম!                         │
  │                                                   │
  │ Let's Encrypt: ✅ verified! সার্টিফিকেট দিচ্ছি    │
  └──────────────────────────────────────────────────┘

  DNS-01 Challenge (Wildcard এর জন্য):
  ┌──────────────────────────────────────────────────┐
  │ Let's Encrypt: "TXT record যোগ করো:              │
  │ _acme-challenge.bkash.com = def456..."           │
  │                                                   │
  │ Certbot: ✅ DNS record যোগ করলাম!                 │
  │                                                   │
  │ Let's Encrypt: ✅ verified! *.bkash.com cert দিচ্ছি│
  └──────────────────────────────────────────────────┘

Step 3: Auto-Renewal (Cron/Timer)
  # প্রতিদিন চেক, expire হওয়ার ৩০ দিন আগে renew
  0 0 * * * certbot renew --quiet

  ⚠️ Let's Encrypt cert ৯০ দিনে expire হয়!
  Auto-renewal অবশ্যই সেটআপ করুন!
```

### Nginx TLS কনফিগারেশন

```nginx
# Nginx TLS Best Practices (bkash.com এর জন্য)

server {
    listen 443 ssl http2;
    server_name bkash.com www.bkash.com;
    
    # Certificate files
    ssl_certificate     /etc/letsencrypt/live/bkash.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bkash.com/privkey.pem;
    
    # TLS version (শুধু 1.2 ও 1.3)
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Cipher suites
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/bkash.com/chain.pem;
    
    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    
    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name bkash.com www.bkash.com;
    return 301 https://$server_name$request_uri;
}
```

---

## 🔄 mTLS (Mutual TLS)

সাধারণ TLS এ শুধু সার্ভার নিজেকে প্রমাণ করে। mTLS এ **উভয় পক্ষ** নিজেকে প্রমাণ করে।

```
সাধারণ TLS (One-Way):
─────────────────────
Client (ব্রাউজার)              Server (bKash)
    │                            │
    │  ◄── Server Certificate ── │  সার্ভার নিজেকে প্রমাণ করে
    │      "আমি bkash.com" ✅     │
    │                            │
    │      ক্লায়েন্ট? 🤷           │  ক্লায়েন্ট কে সেটা সার্ভার জানে না
    │                            │  (Username/Password দিয়ে জানে)

mTLS (Two-Way / Mutual):
────────────────────────
Service A (Payment)             Service B (Account)
    │                            │
    │  ◄── Server Certificate ── │  Service B নিজেকে প্রমাণ করে
    │      "আমি Account Service"  │
    │                            │
    │  ── Client Certificate ──► │  Service A ও নিজেকে প্রমাণ করে
    │     "আমি Payment Service"   │
    │                            │
    │  ═══ দুজনেই verified ═══   │  ✅ উভয় দিক থেকে বিশ্বস্ত
```

### mTLS Use Cases

```
┌─────────────────────────────────────────────────────────┐
│              mTLS কোথায় ব্যবহৃত হয়                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Microservice-to-Microservice Communication          │
│     bKash Payment Service ←→ Account Service            │
│     Pathao Ride Service ←→ Payment Service              │
│                                                         │
│  2. Zero Trust Architecture                             │
│     প্রতিটি সার্ভিস verification ছাড়া কাউকে trust করে না │
│                                                         │
│  3. Service Mesh (Istio, Linkerd)                       │
│     ┌─────────┐    mTLS    ┌─────────┐                  │
│     │Service A│◄═══════════►│Service B│                  │
│     │ +Sidecar│            │ +Sidecar│                  │
│     └─────────┘            └─────────┘                  │
│     Sidecar proxy mTLS handle করে,                     │
│     অ্যাপ কোডে কোনো পরিবর্তন দরকার নেই                  │
│                                                         │
│  4. IoT Device Authentication                           │
│     GP IoT sensors ←→ Central Server                    │
│                                                         │
│  5. API Gateway ←→ Backend Services                     │
│     Daraz API Gateway ←→ Internal Services              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 কোড উদাহরণ

### PHP OpenSSL

```php
<?php
// =============================================
// PHP - TLS/SSL Certificate তথ্য দেখা
// =============================================

function getCertificateInfo(string $domain): array
{
    $context = stream_context_create([
        'ssl' => [
            'capture_peer_cert' => true,
            'capture_peer_cert_chain' => true,
            'verify_peer' => true,
            'verify_peer_name' => true,
        ]
    ]);
    
    $client = stream_socket_client(
        "ssl://$domain:443",
        $errno, $errstr, 30,
        STREAM_CLIENT_CONNECT,
        $context
    );
    
    if (!$client) {
        throw new Exception("সংযোগ ব্যর্থ: $errstr ($errno)");
    }
    
    $params = stream_context_get_params($client);
    $cert = openssl_x509_parse($params['options']['ssl']['peer_certificate']);
    
    $validFrom = date('Y-m-d', $cert['validFrom_time_t']);
    $validTo = date('Y-m-d', $cert['validTo_time_t']);
    $daysLeft = (int) (($cert['validTo_time_t'] - time()) / 86400);
    
    fclose($client);
    
    return [
        'domain' => $domain,
        'subject' => $cert['subject']['CN'] ?? 'N/A',
        'issuer' => $cert['issuer']['O'] ?? $cert['issuer']['CN'] ?? 'N/A',
        'valid_from' => $validFrom,
        'valid_to' => $validTo,
        'days_left' => $daysLeft,
        'serial' => $cert['serialNumber'] ?? 'N/A',
        'san' => $cert['extensions']['subjectAltName'] ?? 'N/A',
        'signature_algorithm' => $cert['signatureTypeSN'] ?? 'N/A',
    ];
}

// বাংলাদেশী ওয়েবসাইটের cert চেক
$domains = ['bkash.com', 'daraz.com.bd', 'nagad.com.bd'];

foreach ($domains as $domain) {
    try {
        $info = getCertificateInfo($domain);
        echo "\n🔒 $domain:\n";
        echo "   Issuer: {$info['issuer']}\n";
        echo "   Valid: {$info['valid_from']} → {$info['valid_to']}\n";
        echo "   Days Left: {$info['days_left']}\n";
        echo "   SAN: {$info['san']}\n";
    } catch (Exception $e) {
        echo "❌ $domain: {$e->getMessage()}\n";
    }
}

// =============================================
// PHP - HTTPS রিকোয়েস্ট (সঠিক TLS সেটিংস)
// =============================================

function secureApiCall(string $url, array $data = []): string
{
    $ch = curl_init($url);
    
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        
        // TLS Settings
        CURLOPT_SSL_VERIFYPEER => true,          // ✅ cert যাচাই করো
        CURLOPT_SSL_VERIFYHOST => 2,             // ✅ hostname যাচাই করো
        CURLOPT_SSLVERSION     => CURL_SSLVERSION_TLSv1_2, // ✅ ন্যূনতম TLS 1.2
        CURLOPT_CAINFO         => '/etc/ssl/certs/ca-certificates.crt',
        
        // Cipher suites (শক্তিশালীগুলো)
        CURLOPT_SSL_CIPHER_LIST => 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384',
        
        // Timeout
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_CONNECTTIMEOUT => 10,
    ]);
    
    if (!empty($data)) {
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    }
    
    $response = curl_exec($ch);
    
    if (curl_errno($ch)) {
        $error = curl_error($ch);
        curl_close($ch);
        throw new Exception("TLS Error: $error");
    }
    
    // TLS তথ্য দেখা
    $tlsVersion = curl_getinfo($ch, CURLINFO_SSL_VERIFYRESULT);
    $protocol = curl_getinfo($ch, CURLINFO_PROTOCOL);
    
    curl_close($ch);
    return $response;
}

// =============================================
// PHP - mTLS (Client Certificate সহ)
// =============================================

function mTLSApiCall(string $url): string
{
    $ch = curl_init($url);
    
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        
        // Server cert verify
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_SSL_VERIFYHOST => 2,
        CURLOPT_CAINFO         => '/etc/ssl/ca-cert.pem',
        
        // Client cert (mTLS)
        CURLOPT_SSLCERT    => '/etc/ssl/client-cert.pem',    // ক্লায়েন্ট cert
        CURLOPT_SSLKEY     => '/etc/ssl/client-key.pem',     // ক্লায়েন্ট private key
        CURLOPT_SSLKEYPASSWD => 'passphrase',                // key password
    ]);
    
    $response = curl_exec($ch);
    
    if (curl_errno($ch)) {
        throw new Exception("mTLS Error: " . curl_error($ch));
    }
    
    curl_close($ch);
    return $response;
}
```

### Node.js HTTPS

```javascript
// =============================================
// Node.js - HTTPS Server (TLS সহ)
// =============================================

const https = require('https');
const fs = require('fs');
const tls = require('tls');

// HTTPS Server তৈরি
const server = https.createServer({
    key: fs.readFileSync('/etc/ssl/private/server-key.pem'),
    cert: fs.readFileSync('/etc/ssl/certs/server-cert.pem'),
    ca: fs.readFileSync('/etc/ssl/certs/ca-cert.pem'),
    
    // TLS Settings
    minVersion: 'TLSv1.2',
    maxVersion: 'TLSv1.3',
    
    ciphers: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_CHACHA20_POLY1305_SHA256',
        'ECDHE-ECDSA-AES256-GCM-SHA384',
        'ECDHE-RSA-AES256-GCM-SHA384'
    ].join(':'),
    
    // OCSP Stapling
    // honorCipherOrder: true
}, (req, res) => {
    // TLS তথ্য দেখা
    const tlsSocket = req.socket;
    console.log('TLS Version:', tlsSocket.getProtocol());
    console.log('Cipher:', tlsSocket.getCipher().name);
    
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ message: 'সুরক্ষিত সংযোগ! 🔒' }));
});

server.listen(443, () => {
    console.log('HTTPS সার্ভার চালু: https://localhost:443');
});

// =============================================
// Node.js - TLS Certificate তথ্য দেখা
// =============================================

async function getCertInfo(domain) {
    return new Promise((resolve, reject) => {
        const socket = tls.connect({
            host: domain,
            port: 443,
            servername: domain, // SNI
            rejectUnauthorized: true
        }, () => {
            const cert = socket.getPeerCertificate(true);
            
            resolve({
                domain,
                subject: cert.subject.CN,
                issuer: cert.issuer.O || cert.issuer.CN,
                validFrom: cert.valid_from,
                validTo: cert.valid_to,
                protocol: socket.getProtocol(),
                cipher: socket.getCipher(),
                fingerprint: cert.fingerprint256,
                san: cert.subjectaltname
            });
            
            socket.end();
        });
        
        socket.on('error', reject);
    });
}

// বাংলাদেশী সাইট চেক
(async () => {
    const domains = ['bkash.com', 'daraz.com.bd', 'pathao.com'];
    
    for (const domain of domains) {
        try {
            const info = await getCertInfo(domain);
            console.log(`\n🔒 ${domain}:`);
            console.log(`  Subject: ${info.subject}`);
            console.log(`  Issuer: ${info.issuer}`);
            console.log(`  Protocol: ${info.protocol}`);
            console.log(`  Cipher: ${info.cipher.name}`);
            console.log(`  Valid: ${info.validFrom} → ${info.validTo}`);
        } catch (err) {
            console.error(`❌ ${domain}: ${err.message}`);
        }
    }
})();

// =============================================
// Node.js - mTLS Client
// =============================================

const mTLSRequest = https.request({
    hostname: 'internal-api.bkash.com',
    port: 443,
    path: '/api/v1/accounts',
    method: 'GET',
    
    // Client Certificate (mTLS)
    key: fs.readFileSync('/etc/ssl/client-key.pem'),
    cert: fs.readFileSync('/etc/ssl/client-cert.pem'),
    ca: fs.readFileSync('/etc/ssl/ca-cert.pem'),
    
    rejectUnauthorized: true
}, (res) => {
    let data = '';
    res.on('data', (chunk) => data += chunk);
    res.on('end', () => {
        console.log('mTLS Response:', JSON.parse(data));
    });
});

mTLSRequest.on('error', (err) => {
    console.error('mTLS Error:', err.message);
});

mTLSRequest.end();

// =============================================
// Node.js - mTLS Server (ক্লায়েন্ট cert বাধ্যতামূলক)
// =============================================

const mTLSServer = https.createServer({
    key: fs.readFileSync('/etc/ssl/server-key.pem'),
    cert: fs.readFileSync('/etc/ssl/server-cert.pem'),
    ca: fs.readFileSync('/etc/ssl/ca-cert.pem'),
    
    requestCert: true,        // ← ক্লায়েন্ট cert চাও
    rejectUnauthorized: true  // ← cert না থাকলে reject
}, (req, res) => {
    const clientCert = req.socket.getPeerCertificate();
    
    if (clientCert.subject) {
        console.log('Client CN:', clientCert.subject.CN);
        console.log('Client Org:', clientCert.subject.O);
        
        res.writeHead(200);
        res.end(JSON.stringify({
            message: 'mTLS verified!',
            client: clientCert.subject.CN
        }));
    } else {
        res.writeHead(401);
        res.end('Client certificate required');
    }
});

mTLSServer.listen(8443, () => {
    console.log('mTLS সার্ভার চালু: https://localhost:8443');
});
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: SSL verification বন্ধ করা
──────────────────────────────────
// PHP
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);  // ❌ NEVER!

// Node.js
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';   // ❌ NEVER!

// Python
requests.get(url, verify=False)                     // ❌ NEVER!

✅ সঠিক: সঠিক CA bundle ব্যবহার করুন
─────────────────────────────────────
// PHP
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
curl_setopt($ch, CURLOPT_CAINFO, '/etc/ssl/certs/ca-certificates.crt');

// Node.js
const agent = new https.Agent({ ca: fs.readFileSync('ca-cert.pem') });

──────────────────────────────────────────────

❌ ভুল: TLS 1.0/1.1 সাপোর্ট করা
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  // ❌ পুরানো ভার্সন!

✅ সঠিক: শুধু TLS 1.2+ সাপোর্ট
   ssl_protocols TLSv1.2 TLSv1.3;          // ✅

──────────────────────────────────────────────

❌ ভুল: HSTS হেডার না দেওয়া
   → ইউজার http:// দিয়ে ঢুকলে MITM attack সম্ভব

✅ সঠিক: HSTS enable করুন
   Strict-Transport-Security: max-age=63072000; includeSubDomains; preload

──────────────────────────────────────────────

❌ ভুল: Self-signed cert production এ ব্যবহার
   → ব্রাউজার warning দেখাবে, ইউজার trust করবে না

✅ সঠিক: Let's Encrypt (বিনামূল্যে!) বা অন্য trusted CA ব্যবহার করুন

──────────────────────────────────────────────

❌ ভুল: Certificate auto-renewal সেটআপ না করা
   → ৯০ দিন পর সাইট ডাউন!

✅ সঠিক: Certbot timer/cron সেটআপ
   systemctl enable certbot.timer
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

```
TLS (HTTPS) সবসময় ব্যবহার করুন:
──────────────────────────────
✅ যেকোনো ওয়েবসাইট (SEO তেও সাহায্য করে)
✅ API endpoints
✅ লগইন/সাইনআপ পেজ
✅ পেমেন্ট পেজ (PCI-DSS বাধ্যতামূলক)
✅ ব্যবহারকারীর ব্যক্তিগত তথ্য

TLS 1.3 ব্যবহার করুন:
─────────────────────
✅ নতুন সিস্টেমে (দ্রুত handshake)
✅ মোবাইল অ্যাপ (কম latency)
✅ উচ্চ ট্রাফিক সাইট (সার্ভার লোড কম)

mTLS ব্যবহার করুন:
──────────────────
✅ Microservice communication
✅ Zero Trust architecture
✅ IoT device authentication
✅ B2B API integration
✅ Service mesh (Istio/Linkerd)

mTLS ব্যবহার করবেন না:
──────────────────────
❌ পাবলিক ওয়েবসাইট (ইউজারদের cert ম্যানেজ কঠিন)
❌ মোবাইল অ্যাপ → পাবলিক API (OAuth/JWT ব্যবহার করুন)
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│               TLS/SSL সারসংক্ষেপ                       │
├───────────────────────────────────────────────────────┤
│                                                       │
│  TLS = ইন্টারনেটের তালা 🔒                             │
│  তিন সুরক্ষা: Encryption + Authentication + Integrity │
│                                                       │
│  TLS 1.2: ২ RTT, এখনো নিরাপদ                         │
│  TLS 1.3: ১ RTT (0-RTT resumption), দ্রুত ও নিরাপদ    │
│                                                       │
│  Certificate Chain: Root CA → Intermediate → Server   │
│  Let's Encrypt: বিনামূল্যে DV cert, ৯০ দিন validity    │
│  mTLS: সার্ভিস-টু-সার্ভিস authentication                │
│                                                       │
│  মূল নিয়ম:                                             │
│  ১. সবসময় HTTPS ব্যবহার করুন                           │
│  ২. TLS 1.2+ only                                     │
│  ৩. SSL verify কখনো বন্ধ করবেন না                      │
│  ৪. HSTS enable করুন                                  │
│  ৫. Auto-renewal সেটআপ করুন                           │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পূর্ববর্তী**: [DNS](./dns.md) | **সূচি**: [README](./README.md)
