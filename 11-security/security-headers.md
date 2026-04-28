# 🛡️ HTTP সিকিউরিটি হেডারস (Security Headers)

## 📌 ধারণা (Concept)

HTTP সিকিউরিটি হেডারস হলো সার্ভার থেকে ব্রাউজারে পাঠানো নির্দেশনা যা ব্রাউজারকে বলে কিভাবে নিরাপদে কন্টেন্ট রেন্ডার ও প্রসেস করতে হবে। সঠিক সিকিউরিটি হেডার ব্যবহার করলে XSS, Clickjacking, MIME sniffing সহ বিভিন্ন আক্রমণ থেকে সুরক্ষা পাওয়া যায়।

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন আপনি একটি চিঠি পাঠাচ্ছেন। চিঠির সাথে কিছু নির্দেশনা দেওয়া আছে — "শুধু এই ব্যক্তি পড়তে পারবে", "কপি করা যাবে না", "ফটো তোলা যাবে না"। HTTP সিকিউরিটি হেডারও তেমনই — সার্ভার ব্রাউজারকে নিরাপত্তা সংক্রান্ত নির্দেশনা দেয়।

---

## 📋 প্রধান সিকিউরিটি হেডারস

### ১. Content-Security-Policy (CSP)

কোন সোর্স থেকে কন্টেন্ট (স্ক্রিপ্ট, স্টাইল, ইমেজ) লোড করা যাবে তা নিয়ন্ত্রণ করে। XSS আক্রমণ প্রতিরোধে অত্যন্ত কার্যকর।

```php
// PHP — CSP হেডার সেট করা
header("Content-Security-Policy: " . implode('; ', [
    "default-src 'self'",                          // ডিফল্টে শুধু নিজের ডোমেইন
    "script-src 'self' 'nonce-" . $nonce . "'",    // স্ক্রিপ্ট শুধু নিজের ডোমেইন + nonce সহ
    "style-src 'self' 'unsafe-inline'",            // স্টাইল নিজের ডোমেইন + inline
    "img-src 'self' data: https://cdn.example.com",// ইমেজ সোর্স
    "font-src 'self' https://fonts.gstatic.com",   // ফন্ট সোর্স
    "connect-src 'self' https://api.example.com",  // API কানেকশন
    "frame-ancestors 'none'",                       // iframe এ লোড করা যাবে না
    "base-uri 'self'",                             // base tag সীমাবদ্ধ
    "form-action 'self'"                           // ফর্ম একশন সীমাবদ্ধ
]));

// HTML এ nonce ব্যবহার
echo "<script nonce='$nonce'>console.log('নিরাপদ স্ক্রিপ্ট');</script>";
```

```javascript
// Express.js — Helmet দিয়ে CSP
import helmet from 'helmet';
import crypto from 'crypto';

app.use((req, res, next) => {
    res.locals.nonce = crypto.randomBytes(16).toString('base64');
    next();
});

app.use(helmet.contentSecurityPolicy({
    directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.nonce}'`],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https://cdn.example.com"],
        connectSrc: ["'self'", "https://api.example.com"],
        frameAncestors: ["'none'"],
        formAction: ["'self'"]
    }
}));
```

### ২. Strict-Transport-Security (HSTS)

ব্রাউজারকে বলে সবসময় HTTPS ব্যবহার করতে। HTTP তে কোনো রিকোয়েস্ট যাবে না।

```php
// PHP
header('Strict-Transport-Security: max-age=31536000; includeSubDomains; preload');
// max-age: ১ বছর (সেকেন্ডে)
// includeSubDomains: সাবডোমেইনেও প্রযোজ্য
// preload: ব্রাউজারের preload লিস্টে যুক্ত হওয়ার জন্য
```

```javascript
// Express.js
app.use(helmet.hsts({
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
}));
```

### ৩. X-Frame-Options

পেজটি iframe এ লোড করা যাবে কিনা তা নিয়ন্ত্রণ করে। Clickjacking আক্রমণ প্রতিরোধ করে।

```php
// PHP
header('X-Frame-Options: DENY');          // কোথাও iframe এ লোড করা যাবে না
// header('X-Frame-Options: SAMEORIGIN'); // শুধু নিজের ডোমেইনে iframe এ অনুমোদিত
```

```javascript
// Express.js
app.use(helmet.frameguard({ action: 'deny' }));
```

### ৪. X-Content-Type-Options

ব্রাউজারকে MIME type sniffing থেকে বিরত রাখে।

```php
// PHP
header('X-Content-Type-Options: nosniff');
```

```javascript
// Express.js
app.use(helmet.noSniff());
```

### ৫. Referrer-Policy

অন্য সাইটে নেভিগেট করার সময় কতটুকু referrer তথ্য পাঠানো হবে।

```php
// PHP
header('Referrer-Policy: strict-origin-when-cross-origin');
```

```javascript
// Express.js
app.use(helmet.referrerPolicy({
    policy: 'strict-origin-when-cross-origin'
}));
```

### ৬. Permissions-Policy (আগে Feature-Policy)

ব্রাউজারের বিভিন্ন ফিচার (ক্যামেরা, মাইক্রোফোন, জিওলোকেশন) কন্ট্রোল করে।

```php
// PHP
header('Permissions-Policy: camera=(), microphone=(), geolocation=(self), payment=(self)');
// camera=(): ক্যামেরা অ্যাক্সেস সম্পূর্ণ বন্ধ
// geolocation=(self): শুধু নিজের ডোমেইনে অনুমোদিত
```

```javascript
// Express.js
app.use(helmet.permittedCrossDomainPolicies());
app.use((req, res, next) => {
    res.setHeader('Permissions-Policy',
        'camera=(), microphone=(), geolocation=(self), payment=(self)'
    );
    next();
});
```

### ৭. X-XSS-Protection

পুরোনো ব্রাউজারের জন্য XSS ফিল্টার। আধুনিক ব্রাউজারে CSP যথেষ্ট।

```php
// PHP
header('X-XSS-Protection: 0');
// আধুনিক সুপারিশ: বন্ধ রাখুন (CSP ব্যবহার করুন)
// পুরোনো ব্রাউজারে: header('X-XSS-Protection: 1; mode=block');
```

---

## 🔧 সম্পূর্ণ কনফিগারেশন

### PHP — সব হেডার একসাথে

```php
class SecurityHeaders {
    public static function apply(): void {
        $nonce = base64_encode(random_bytes(16));

        $headers = [
            'Content-Security-Policy' => implode('; ', [
                "default-src 'self'",
                "script-src 'self' 'nonce-$nonce'",
                "style-src 'self' 'unsafe-inline'",
                "img-src 'self' data:",
                "frame-ancestors 'none'",
                "base-uri 'self'",
                "form-action 'self'"
            ]),
            'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains; preload',
            'X-Frame-Options' => 'DENY',
            'X-Content-Type-Options' => 'nosniff',
            'Referrer-Policy' => 'strict-origin-when-cross-origin',
            'Permissions-Policy' => 'camera=(), microphone=(), geolocation=(self)',
            'X-XSS-Protection' => '0',
        ];

        foreach ($headers as $name => $value) {
            header("$name: $value");
        }

        // Cookie সিকিউরিটি
        ini_set('session.cookie_httponly', '1');
        ini_set('session.cookie_secure', '1');
        ini_set('session.cookie_samesite', 'Strict');
    }
}

// ব্যবহার — অ্যাপ্লিকেশনের শুরুতে কল করুন
SecurityHeaders::apply();
```

### Express.js — Helmet দিয়ে সব হেডার

```javascript
import express from 'express';
import helmet from 'helmet';
import crypto from 'crypto';

const app = express();

// Nonce তৈরি
app.use((req, res, next) => {
    res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
    next();
});

// Helmet দিয়ে সব সিকিউরিটি হেডার একসাথে
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.cspNonce}'`],
            styleSrc: ["'self'", "'unsafe-inline'"],
            imgSrc: ["'self'", "data:"],
            frameAncestors: ["'none'"]
        }
    },
    hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
    frameguard: { action: 'deny' },
    noSniff: true,
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    xssFilter: false // CSP ব্যবহার করা হচ্ছে
}));
```

### Nginx কনফিগারেশন

```nginx
# /etc/nginx/snippets/security-headers.conf
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-ancestors 'none'" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(self)" always;

# ব্যবহার
# server {
#     include /etc/nginx/snippets/security-headers.conf;
# }
```

---

## 🧪 সিকিউরিটি হেডার পরীক্ষা

```bash
# কমান্ড লাইন থেকে হেডার চেক
curl -I https://your-site.com

# অনলাইন টুল
# https://securityheaders.com
# https://observatory.mozilla.org
```

---

## ✅ কখন ব্যবহার করবেন

- **সবসময়** — প্রতিটি প্রোডাকশন ওয়েব অ্যাপ্লিকেশনে সিকিউরিটি হেডার ব্যবহার করুন
- **CSP:** XSS প্রতিরোধের সবচেয়ে কার্যকর উপায়
- **HSTS:** HTTPS ব্যবহার নিশ্চিত করতে
- **X-Frame-Options:** Clickjacking প্রতিরোধে

## ❌ কখন ব্যবহার করবেন না

- **অত্যন্ত কঠোর CSP** প্রথমেই দেবেন না — `Content-Security-Policy-Report-Only` দিয়ে পরীক্ষা করুন
- **inline স্ক্রিপ্ট নির্ভর অ্যাপ** থাকলে CSP ধীরে ধীরে implement করুন
- **HSTS preload** ভালোভাবে পরীক্ষা না করে দেবেন না — ভুল হলে সাইট অ্যাক্সেসযোগ্য থাকবে না
- **ডেভেলপমেন্টে** অতিরিক্ত কঠোর CSP ডিবাগিং কঠিন করতে পারে
