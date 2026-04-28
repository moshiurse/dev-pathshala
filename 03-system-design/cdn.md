# 🌐 সিডিএন (Content Delivery Network)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**CDN (Content Delivery Network)** হলো ভৌগোলিকভাবে বিতরিত সার্ভারের একটি নেটওয়ার্ক যা ইউজারের নিকটতম সার্ভার (Edge Server) থেকে স্ট্যাটিক ও ডায়নামিক কন্টেন্ট সরবরাহ করে। এটি লেটেন্সি কমায়, ব্যান্ডউইথ সাশ্রয় করে এবং অরিজিন সার্ভারের লোড হ্রাস করে।

### কেন CDN দরকার?

```
সমস্যা — Prothom Alo ওয়েবসাইট (অরিজিন সার্ভার: ঢাকা):
  ঢাকার ইউজার     → ২০ms latency  ✅
  নিউইয়র্কের ইউজার → ৩৫০ms latency 😰
  লন্ডনের ইউজার    → ২৮০ms latency 😰

CDN দিয়ে সমাধান:
  ঢাকার ইউজার     → ঢাকা Edge     → ১৫ms  ✅
  নিউইয়র্কের ইউজার → নিউইয়র্ক Edge → ২০ms  ✅
  লন্ডনের ইউজার    → লন্ডন Edge    → ১৮ms  ✅
```

---

## 🎯 বাস্তব উদাহরণ — ফার্মেসি চেইন

```
কল্পনা করুন, একটি ওষুধ কোম্পানির কারখানা শুধু গাজীপুরে।

CDN ছাড়া:
  বরিশালের রোগী → গাজীপুর কারখানায় যাও → ওষুধ নাও → ফেরত আসো
  সময়: ১ দিন 😩

CDN সহ (ফার্মেসি চেইন = Edge Server):
  বরিশালের রোগী → বরিশালের ফার্মেসি → ওষুধ পাও
  সময়: ১০ মিনিট ✅

  ফার্মেসিতে ওষুধ না থাকলে → কারখানা থেকে আনো → ফার্মেসিতে স্টক করো
  পরবর্তী রোগী সরাসরি ফার্মেসি থেকে পাবে
```

---

## 📊 CDN আর্কিটেকচার

```
                    CDN আর্কিটেকচার
                    ================

  ইউজার (চট্টগ্রাম)          ইউজার (সিলেট)
        │                          │
        ▼                          ▼
  ┌───────────┐              ┌───────────┐
  │ Edge PoP  │              │ Edge PoP  │
  │ চট্টগ্রাম  │              │ সিলেট     │
  └─────┬─────┘              └─────┬─────┘
        │ Cache Miss?              │ Cache Miss?
        └──────────┐    ┌─────────┘
                   ▼    ▼
             ┌──────────────┐
             │  Origin      │
             │  Server      │
             │  (ঢাকা)      │
             └──────────────┘
```

---

## 🔀 Push CDN vs Pull CDN

### Push CDN

```
কন্টেন্ট আপলোড হলেই CDN-এ পাঠিয়ে দাও
  ┌────────┐  Push   ┌─────────┐
  │ Origin │ ──────→ │ CDN Edge│
  │ Server │         │ Servers │
  └────────┘         └─────────┘

সুবিধা: প্রথম রিকোয়েস্টেও ফাস্ট (কোনো Cache Miss নেই)
অসুবিধা: স্টোরেজ বেশি লাগে, আপডেট ম্যানেজমেন্ট জটিল
```

### Pull CDN

```
ইউজার রিকোয়েস্ট করলে CDN অরিজিন থেকে টেনে আনে
  ইউজার → CDN Edge (নেই!) → Origin থেকে Fetch → Cache → Response

সুবিধা: স্টোরেজ অপটিমাল, শুধু জনপ্রিয় কন্টেন্ট ক্যাশ হয়
অসুবিধা: প্রথম রিকোয়েস্ট ধীর (Cold Start)
```

### তুলনা

| বৈশিষ্ট্য | Push CDN | Pull CDN |
|---|---|---|
| কন্টেন্ট আপডেট | অরিজিন থেকে পুশ করে | চাহিদা অনুযায়ী টেনে আনে |
| প্রথম রিকোয়েস্ট | দ্রুত | ধীর (Cache Miss) |
| স্টোরেজ ব্যবহার | বেশি | কম (শুধু জনপ্রিয় কন্টেন্ট) |
| উপযুক্ত | কম পরিবর্তনশীল কন্টেন্ট | ডায়নামিক/বেশি কন্টেন্ট |
| উদাহরণ | ফার্মওয়্যার আপডেট | নিউজ সাইটের ছবি |

---

## 🗑️ ক্যাশ ইনভ্যালিডেশন কৌশল

### ১. TTL (Time-To-Live)

```
নির্দিষ্ট সময় পর ক্যাশ মেয়াদোত্তীর্ণ হয়

Cache-Control: max-age=3600   → ১ ঘণ্টা পর expire
Cache-Control: s-maxage=86400 → CDN-এ ২৪ ঘণ্টা রাখো

কন্টেন্ট টাইপ অনুযায়ী TTL:
  HTML পেজ      → ৫-১০ মিনিট (ঘন ঘন পরিবর্তন হয়)
  CSS/JS        → ১ বছর (ফাইলনামে হ্যাশ ব্যবহার করো)
  ছবি/ভিডিও    → ১ মাস - ১ বছর
  API Response  → ৩০ সেকেন্ড - ৫ মিনিট
```

### ২. Cache Purge (তাৎক্ষণিক মুছে ফেলা)

```
জরুরি আপডেটে নির্দিষ্ট কন্টেন্ট CDN থেকে মুছে দাও

PURGE /images/breaking-news.jpg → সব Edge থেকে মুছে যাবে
```

### ৩. Versioned URLs

```
ফাইলনামে ভার্সন/হ্যাশ যুক্ত করো → পুরনো ক্যাশ সমস্যা দূর

style.css       → style.abc123.css
app.js          → app.def456.js

ফাইল পরিবর্তন হলে নতুন হ্যাশ → নতুন URL → CDN নতুন ফাইল আনবে
```

### ৪. Stale-While-Revalidate

```
পুরনো ক্যাশ দেখাও, পেছনে নতুন কপি আনো

Cache-Control: max-age=60, stale-while-revalidate=300

→ ৬০ সেকেন্ড পর্যন্ত সরাসরি ক্যাশ থেকে
→ ৬০-৩৬০ সেকেন্ডে: পুরনো দেখাও + ব্যাকগ্রাউন্ডে রিফ্রেশ
→ ৩৬০ সেকেন্ড পর: নতুন করে আনো
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

class CDNManager
{
    private string $cdnBaseUrl;
    private string $originBaseUrl;

    public function __construct(string $cdnBaseUrl, string $originBaseUrl)
    {
        $this->cdnBaseUrl = rtrim($cdnBaseUrl, '/');
        $this->originBaseUrl = rtrim($originBaseUrl, '/');
    }

    // ভার্সনড URL তৈরি করো (Cache Busting)
    public function asset(string $path): string
    {
        $filePath = public_path($path);
        if (file_exists($filePath)) {
            $hash = substr(md5_file($filePath), 0, 8);
            $ext = pathinfo($path, PATHINFO_EXTENSION);
            $name = pathinfo($path, PATHINFO_FILENAME);
            $dir = pathinfo($path, PATHINFO_DIRNAME);
            return "{$this->cdnBaseUrl}/{$dir}/{$name}.{$hash}.{$ext}";
        }
        return "{$this->cdnBaseUrl}/{$path}";
    }

    // CDN ক্যাশ পার্জ করো
    public function purge(string $path): bool
    {
        $ch = curl_init("{$this->cdnBaseUrl}/{$path}");
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PURGE');
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        return $httpCode === 200;
    }

    // সঠিক Cache-Control হেডার সেট করো
    public function setCacheHeaders(string $contentType): void
    {
        $ttl = match ($contentType) {
            'image'  => 'public, max-age=2592000',             // ৩০ দিন
            'css_js' => 'public, max-age=31536000, immutable', // ১ বছর
            'html'   => 'public, max-age=300, stale-while-revalidate=600',
            'api'    => 'public, max-age=30, s-maxage=60',
            default  => 'public, max-age=3600',
        };
        header("Cache-Control: {$ttl}");
    }
}

// ব্যবহার
$cdn = new CDNManager('https://cdn.prothomalo.com', 'https://origin.prothomalo.com');
echo $cdn->asset('css/style.css');
// → https://cdn.prothomalo.com/css/style.a1b2c3d4.css
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
class CDNManager {
  constructor(cdnBase, originBase) {
    this.cdnBase = cdnBase.replace(/\/$/, '');
    this.originBase = originBase.replace(/\/$/, '');
  }

  // ভার্সনড URL তৈরি (ফ্রন্টেন্ড)
  asset(path, version) {
    const ext = path.split('.').pop();
    const name = path.replace(`.${ext}`, '');
    return `${this.cdnBase}/${name}.${version}.${ext}`;
  }

  // CDN থেকে কন্টেন্ট ফেচ (ফলব্যাক সহ)
  async fetchWithFallback(path, options = {}) {
    try {
      const cdnUrl = `${this.cdnBase}/${path}`;
      const response = await fetch(cdnUrl, {
        ...options,
        signal: AbortSignal.timeout(3000), // ৩ সেকেন্ড টাইমআউট
      });

      if (response.ok) return response;
      throw new Error(`CDN error: ${response.status}`);
    } catch (err) {
      // CDN ব্যর্থ হলে সরাসরি অরিজিন থেকে আনো
      console.warn('CDN ব্যর্থ, অরিজিন থেকে আনছি:', err.message);
      return fetch(`${this.originBase}/${path}`, options);
    }
  }

  // Service Worker দিয়ে CDN ক্যাশিং
  registerServiceWorker() {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/sw.js');
    }
  }
}

// Service Worker (sw.js) — CDN-first কৌশল
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/static/')) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        if (cached) return cached; // লোকাল ক্যাশ থেকে
        return fetch(event.request).then((response) => {
          const clone = response.clone();
          caches.open('cdn-cache-v1').then((cache) => {
            cache.put(event.request, clone);
          });
          return response;
        });
      })
    );
  }
});

// ব্যবহার
const cdn = new CDNManager('https://cdn.daraz.com.bd', 'https://origin.daraz.com.bd');
const img = cdn.asset('images/product.jpg', 'v3');
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন |
|---|---|
| স্ট্যাটিক ফাইল সার্ভিং (ছবি, CSS, JS) | ব্যান্ডউইথ সাশ্রয়, দ্রুত লোডিং |
| গ্লোবাল ইউজারবেস | ভৌগোলিক লেটেন্সি কমানো |
| ভিডিও/অডিও স্ট্রিমিং | বাফারিং কমানো |
| ট্রাফিক স্পাইক প্রতিরোধ | অরিজিন সার্ভার রক্ষা |
| DDoS প্রটেকশন | Edge-এ আক্রমণ শোষণ |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| সম্পূর্ণ ডায়নামিক কন্টেন্ট | ক্যাশিংয়ের সুবিধা পাবে না |
| লোকাল-অনলি অ্যাপ্লিকেশন | অতিরিক্ত খরচ, কোনো লাভ নেই |
| রিয়েল-টাইম ডেটা (স্টক প্রাইস) | ক্যাশ করলে ভুল তথ্য দেখাবে |
| ছোট ট্রাফিক (দিনে ১০০ ভিজিটর) | খরচ বেশি, লাভ কম |

---

## 🔑 মনে রাখার পয়েন্ট

1. **Pull CDN** বেশি জনপ্রিয় — চাহিদা অনুযায়ী ক্যাশ করে
2. **Push CDN** — কম পরিবর্তনশীল, গুরুত্বপূর্ণ কন্টেন্টের জন্য
3. **Versioned URLs** — সবচেয়ে নির্ভরযোগ্য ক্যাশ ইনভ্যালিডেশন কৌশল
4. **Edge PoP** — ইউজারের কাছাকাছি, তাই দ্রুত
5. ইন্টারভিউতে — Cloudflare, CloudFront, Akamai এর রেফারেন্স দিন
