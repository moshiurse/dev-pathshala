# 📌 ফ্রন্টেন্ড পারফরম্যান্স (Frontend Performance)

## ফ্রন্টেন্ড পারফরম্যান্স কেন গুরুত্বপূর্ণ?

বাংলাদেশের প্রেক্ষাপটে ফ্রন্টেন্ড পারফরম্যান্স অত্যন্ত গুরুত্বপূর্ণ। আমাদের বেশিরভাগ ইউজার ৩G/৪G
কানেকশনে লো-এন্ড অ্যান্ড্রয়েড ডিভাইস ব্যবহার করেন। Daraz, Pathao, bKash, Nagad-এর মতো অ্যাপকে
দ্রুত লোড হতে হয়, না হলে ইউজার চলে যাবে।

```
পারফরম্যান্স কেন গুরুত্বপূর্ণ — বাংলাদেশ কনটেক্সট
═══════════════════════════════════════════════════════

  📱 ৮০%+ ইউজার → লো-এন্ড অ্যান্ড্রয়েড (2-4GB RAM)
  📶 বেশিরভাগ → 3G/4G (অনেক জায়গায় slow)
  💰 ডাটা ব্যয়বহুল → বড় বান্ডেল = বেশি খরচ
  ⏳ ৩ সেকেন্ডের বেশি লোড → ৫৩% ইউজার চলে যায়

  ┌─────────────────────────────────────────────────┐
  │   Fast Site (< 3s)     vs    Slow Site (> 5s)   │
  │                                                 │
  │   ✅ বেশি Conversion         ❌ কম Conversion    │
  │   ✅ ভালো SEO Ranking        ❌ খারাপ Ranking    │
  │   ✅ কম Bounce Rate          ❌ বেশি Bounce Rate │
  │   ✅ ইউজার সন্তুষ্ট          ❌ ইউজার হতাশ      │
  └─────────────────────────────────────────────────┘
```

### 🏠 বাস্তব উদাহরণ

- **Daraz Bangladesh**: প্রোডাক্ট পেজ ১ সেকেন্ড দ্রুত লোড হলে conversion ১০%+ বাড়ে
- **Pathao**: রাইড বুকিং পেজ দ্রুত না হলে ইউজার অন্য অ্যাপে চলে যায়
- **bKash / Nagad**: পেমেন্ট ট্রানজেকশন ধীর হলে ইউজার আস্থা হারায়
- **Grameenphone**: MyGP অ্যাপ লাইটওয়েট না হলে গ্রামের ইউজারদের কাজে আসে না

---

## 📊 Core Web Vitals

Google-এর Core Web Vitals হলো ওয়েব পারফরম্যান্স মাপার তিনটি প্রধান মেট্রিক। এগুলো SEO
ranking-এও সরাসরি প্রভাব ফেলে।

```
Core Web Vitals — টাইমলাইন ডায়াগ্রাম
══════════════════════════════════════════════════════════════

  User clicks link
  │
  ▼
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │ Navigation│────▶│ First Paint  │────▶│    LCP       │
  │  Start    │     │ (FP / FCP)   │     │ (Largest     │
  │  t=0      │     │ t ≈ 1.0s     │     │ Contentful   │
  │           │     │              │     │  Paint)      │
  └──────────┘     └──────────────┘     │ t ≤ 2.5s ✅  │
                                        └──────────────┘
  User interacts (click/tap/key)
  │
  ▼
  ┌──────────────────────────┐
  │        INP               │
  │  (Interaction to Next    │
  │   Paint)                 │
  │  delay ≤ 200ms ✅        │
  │                          │
  │  Input ──▶ Processing    │
  │       ──▶ Next Paint     │
  └──────────────────────────┘

  Page rendering / shifting
  │
  ▼
  ┌──────────────────────────┐
  │        CLS               │
  │  (Cumulative Layout      │
  │   Shift)                 │
  │  score ≤ 0.1 ✅          │
  │                          │
  │  Elements unexpectedly   │
  │  moving = BAD ❌         │
  └──────────────────────────┘
```

### 📖 প্রতিটি মেট্রিক বিস্তারিত

**LCP (Largest Contentful Paint)** — পেজের সবচেয়ে বড় কন্টেন্ট (ইমেজ, টেক্সট ব্লক)
কত দ্রুত ভিজিবল হয়। Daraz-এর হোমপেজে hero banner কত দ্রুত দেখা যায়, সেটাই LCP।

**INP (Interaction to Next Paint)** — ইউজার ক্লিক/ট্যাপ করার পর ব্রাউজার কত দ্রুত
রেসপন্স দেয়। bKash-এ "Send Money" বাটনে ক্লিক করার পর কত দ্রুত পরের স্ক্রিন আসে।

**CLS (Cumulative Layout Shift)** — পেজ লোড হওয়ার সময় এলিমেন্ট হঠাৎ সরে যায় কিনা।
Nagad-এ পেমেন্ট বাটনে ক্লিক করতে গেলে হঠাৎ অ্যাড এসে বাটন সরে গেলে — সেটা খারাপ CLS।

### 🎯 টার্গেট ভ্যালু

| মেট্রিক | ✅ ভালো | ⚠️ উন্নতি দরকার | ❌ খারাপ |
|---------|---------|-----------------|---------|
| LCP     | ≤ 2.5s  | ≤ 4.0s          | > 4.0s  |
| INP     | ≤ 200ms | ≤ 500ms         | > 500ms |
| CLS     | ≤ 0.1   | ≤ 0.25          | > 0.25  |

### মেট্রিক মাপার কোড

```javascript
// ✅ Web Vitals API দিয়ে Core Web Vitals মাপা
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP((metric) => {
  console.log('LCP:', metric.value, 'ms');
  // analytics-এ পাঠান
  sendToAnalytics({ name: 'LCP', value: metric.value });
});

onINP((metric) => {
  console.log('INP:', metric.value, 'ms');
  sendToAnalytics({ name: 'INP', value: metric.value });
});

onCLS((metric) => {
  console.log('CLS:', metric.value);
  sendToAnalytics({ name: 'CLS', value: metric.value });
});

function sendToAnalytics(data) {
  navigator.sendBeacon('/analytics', JSON.stringify(data));
}
```

---

## 🎨 ক্রিটিকাল রেন্ডারিং পাথ (Critical Rendering Path)

ব্রাউজার কীভাবে HTML থেকে পিক্সেল তৈরি করে সেটা বোঝা ফ্রন্টেন্ড পারফরম্যান্সের ভিত্তি।

```
ক্রিটিকাল রেন্ডারিং পাথ পাইপলাইন
═════════════════════════════════════════════════════════════════

  HTML ──────▶ DOM Tree
                 │
                 │   ┌──────────────┐
  CSS ──────────▶│──▶│  CSSOM Tree  │
                 │   └──────┬───────┘
                 │          │
                 ▼          ▼
            ┌────────────────────┐
            │    Render Tree     │  (DOM + CSSOM একত্রিত)
            │  (visible nodes    │
            │   only)            │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │     Layout         │  (প্রতিটি element এর
            │  (Reflow)          │   position ও size)
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │      Paint         │  (pixel এ convert)
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │    Composite       │  (layer একত্রিত করে
            │                    │   স্ক্রিনে দেখানো)
            └────────────────────┘
```

### প্রতিটি স্টেপ অপটিমাইজ করা

```html
<!-- ❌ রেন্ডার ব্লকিং CSS -->
<link rel="stylesheet" href="all-styles.css" />

<!-- ✅ ক্রিটিকাল CSS ইনলাইন + বাকিটা async -->
<style>
  /* ক্রিটিকাল CSS — above-the-fold কন্টেন্ট */
  body { margin: 0; font-family: sans-serif; }
  .hero { background: #1a73e8; color: white; padding: 2rem; }
  .nav { display: flex; gap: 1rem; }
</style>
<link rel="preload" href="full-styles.css" as="style" 
      onload="this.onload=null;this.rel='stylesheet'" />
```

```html
<!-- ❌ রেন্ডার ব্লকিং JavaScript -->
<script src="heavy-analytics.js"></script>
<script src="chat-widget.js"></script>

<!-- ✅ Non-blocking JavaScript -->
<script src="heavy-analytics.js" defer></script>
<script src="chat-widget.js" async></script>
```

```
defer vs async — কখন কোনটি
════════════════════════════════════════

  HTML Parsing:  ████████████████████████████████

  <script>       ──── Block ──── Execute
                 (parsing থামে)

  <script defer> ████████████████████████████████ Execute
                 (parsing শেষে execute)          ↑

  <script async> ████████ Execute ███████████████
                 (download হলেই execute)
```

---

## 📦 JavaScript বান্ডেল অপটিমাইজেশন

বড় JavaScript বান্ডেল = ধীর পেজ লোড। বাংলাদেশের slow নেটওয়ার্কে এটা আরো বেশি সমস্যা।

```
বান্ডেল সাইজের প্রভাব
═══════════════════════════════════

  ┌─────────────────────────────────────────┐
  │  বান্ডেল সাইজ      3G লোড টাইম          │
  │  ─────────────     ──────────────       │
  │  100 KB            ~1.5 সেকেন্ড         │
  │  500 KB            ~7.5 সেকেন্ড         │
  │  1 MB              ~15 সেকেন্ড   ❌     │
  │  2 MB              ~30 সেকেন্ড   💀     │
  └─────────────────────────────────────────┘

  Daraz-এর বাজেট: মোট JS < 300KB (gzipped)
```

### ১. Code Splitting

```javascript
// ❌ সব একসাথে import — বিশাল বান্ডেল
import { Chart } from 'chart.js';             // 60KB
import { DatePicker } from 'huge-datepicker'; // 45KB
import { Editor } from 'rich-editor';         // 80KB
import { Map } from 'leaflet';                // 140KB
// মোট: 325KB — প্রথম লোডেই সব ডাউনলোড!

// ✅ Dynamic import — যখন দরকার তখন লোড
async function showChart(data) {
  const { Chart } = await import('chart.js');
  const chart = new Chart(canvas, { data });
}

async function showMap(location) {
  const { Map } = await import('leaflet');
  const map = new Map('map-container');
  map.setView(location, 13);
}
```

### ২. React Lazy Loading

```javascript
import React, { Suspense, lazy } from 'react';

// ❌ সব কম্পোনেন্ট আগেই লোড
import Dashboard from './Dashboard';
import Settings from './Settings';
import AdminPanel from './AdminPanel';

// ✅ Lazy loading — route ভিত্তিক code splitting
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));
const AdminPanel = lazy(() => import('./AdminPanel'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/admin" element={<AdminPanel />} />
      </Routes>
    </Suspense>
  );
}
```

### ৩. Tree Shaking

```javascript
// ❌ পুরো lodash import — 70KB+
import _ from 'lodash';
const result = _.debounce(handler, 300);

// ✅ শুধু দরকারি ফাংশন import — ~1KB
import debounce from 'lodash/debounce';
const result = debounce(handler, 300);

// ✅ আরো ভালো — নিজে লিখুন সহজ ইউটিলিটি
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

### ৪. Webpack / Vite কনফিগারেশন

```javascript
// webpack.config.js — অপটিমাইজড বান্ডেল
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          // vendor কোড আলাদা chunk — ক্যাশ হবে বেশিদিন
        },
        common: {
          minChunks: 2,
          name: 'common',
          chunks: 'all',
        },
      },
    },
    usedExports: true, // tree shaking
  },
};
```

---

## 🖼️ ইমেজ অপটিমাইজেশন

ওয়েবসাইটের মোট ডাটার ৫০%+ ইমেজ থেকে আসে। Daraz-এর প্রোডাক্ট পেজে শত শত ইমেজ —
এগুলো অপটিমাইজ না করলে পেজ অত্যন্ত ধীর হবে।

```
ইমেজ ফরম্যাট তুলনা (100KB মূল JPEG)
════════════════════════════════════════

  ফরম্যাট    সাইজ      কোয়ালিটি   সাপোর্ট
  ────────  ────────  ──────────  ──────────
  JPEG      100 KB    ★★★☆☆      সব ব্রাউজার
  PNG       150 KB    ★★★★★      সব ব্রাউজার
  WebP       65 KB    ★★★★☆      ৯৫%+ ব্রাউজার  ✅
  AVIF       45 KB    ★★★★★      ৮৫%+ ব্রাউজার  ✅✅
```

### Responsive Images

```html
<!-- ❌ একটাই বড় ইমেজ সবার জন্য -->
<img src="product-1200.jpg" alt="Product" />
<!-- মোবাইলেও 1200px ইমেজ ডাউনলোড! অপচয়! -->

<!-- ✅ Responsive images — ডিভাইস অনুযায়ী সাইজ -->
<img 
  src="product-small.webp"
  srcset="product-300.webp 300w, 
          product-600.webp 600w, 
          product-1200.webp 1200w"
  sizes="(max-width: 600px) 300px, 
         (max-width: 1200px) 600px, 
         1200px"
  loading="lazy"
  decoding="async"
  alt="Product"
/>

<!-- ✅ Modern format with fallback -->
<picture>
  <source srcset="product.avif" type="image/avif" />
  <source srcset="product.webp" type="image/webp" />
  <img src="product.jpg" alt="Product" loading="lazy" />
</picture>
```

### Lazy Loading Images

```javascript
// ✅ Native lazy loading (সবচেয়ে সহজ)
// শুধু attribute যোগ করুন — ব্রাউজার নিজেই handle করবে
// <img src="photo.webp" loading="lazy" alt="..." />

// ✅ Intersection Observer দিয়ে advanced lazy loading
function lazyLoadImages() {
  const images = document.querySelectorAll('img[data-src]');
  
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        if (img.dataset.srcset) {
          img.srcset = img.dataset.srcset;
        }
        img.classList.add('loaded');
        observer.unobserve(img);
      }
    });
  }, {
    rootMargin: '200px', // ২০০px আগে থেকে লোড শুরু
  });

  images.forEach(img => observer.observe(img));
}

document.addEventListener('DOMContentLoaded', lazyLoadImages);
```

### CLS এড়াতে ইমেজে width/height দিন

```html
<!-- ❌ width/height নেই — CLS হবে -->
<img src="banner.webp" alt="Banner" />

<!-- ✅ width/height দিন — ব্রাউজার আগেই জায়গা রাখবে -->
<img 
  src="banner.webp" 
  width="1200" 
  height="400" 
  alt="Banner"
  style="width: 100%; height: auto;"
/>

<!-- ✅ CSS aspect-ratio ব্যবহার -->
<style>
.product-image {
  aspect-ratio: 4 / 3;
  width: 100%;
  object-fit: cover;
}
</style>
```

---

## 💾 ক্যাশিং স্ট্র্যাটেজি (Caching Strategy)

সঠিক ক্যাশিং = দ্বিতীয়বার ভিজিটে প্রায় instant লোড। বাংলাদেশের slow নেটওয়ার্কে
ক্যাশিং আরো বেশি গুরুত্বপূর্ণ।

```
ক্যাশিং লেয়ার ডায়াগ্রাম
═══════════════════════════════════════════════════════

  ইউজার (ঢাকা)
    │
    ▼
  ┌─────────────────────┐
  │  Browser Cache       │  ← সবচেয়ে দ্রুত (0ms)
  │  (Memory / Disk)     │     Cache-Control header
  └──────────┬──────────┘
             │ cache miss
             ▼
  ┌─────────────────────┐
  │  Service Worker      │  ← অফলাইনেও কাজ করে
  │  Cache               │     programmatic control
  └──────────┬──────────┘
             │ cache miss
             ▼
  ┌─────────────────────┐
  │  CDN Cache           │  ← সিঙ্গাপুর/মুম্বাই edge
  │  (Cloudflare/AWS)    │     ~50ms latency
  └──────────┬──────────┘
             │ cache miss
             ▼
  ┌─────────────────────┐
  │  Origin Server       │  ← মূল সার্ভার
  │  (AWS Singapore)     │     ~200ms+ latency
  └─────────────────────┘
```

### HTTP Cache Headers

```javascript
// Express.js — ক্যাশ হেডার সেটআপ

// ✅ Static assets — দীর্ঘ ক্যাশ (hashed filenames সহ)
app.use('/static', express.static('public', {
  maxAge: '1y',           // ১ বছর ক্যাশ
  immutable: true,        // কখনো পরিবর্তন হবে না
}));
// Cache-Control: public, max-age=31536000, immutable

// ✅ HTML — ক্যাশ করবেন না (সর্বদা fresh)
app.get('/', (req, res) => {
  res.set('Cache-Control', 'no-cache, no-store, must-revalidate');
  res.sendFile('index.html');
});

// ✅ API responses — ছোট ক্যাশ + revalidate
app.get('/api/products', (req, res) => {
  res.set('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
  // ৫ মিনিট ক্যাশ, তারপর background-এ revalidate
  res.json(products);
});
```

### Service Worker Caching

```javascript
// service-worker.js

const CACHE_NAME = 'daraz-clone-v2';
const STATIC_ASSETS = [
  '/',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.webp',
  '/offline.html',
];

// ইনস্টলের সময় static assets ক্যাশ করুন
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(STATIC_ASSETS);
    })
  );
});

// পুরনো ক্যাশ মুছে ফেলুন
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) => {
      return Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      );
    })
  );
});

// Network-first strategy (API calls)
// Cache-first strategy (static assets)
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  if (url.pathname.startsWith('/api/')) {
    // API — নেটওয়ার্ক আগে, ব্যর্থ হলে ক্যাশ
    event.respondWith(
      fetch(request)
        .then((response) => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(request, clone);
          });
          return response;
        })
        .catch(() => caches.match(request))
    );
  } else {
    // Static — ক্যাশ আগে, না থাকলে নেটওয়ার্ক
    event.respondWith(
      caches.match(request).then((cached) => {
        return cached || fetch(request).then((response) => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(request, clone);
          });
          return response;
        });
      })
    );
  }
});
```

---

## 📏 পারফরম্যান্স বাজেট (Performance Budget)

পারফরম্যান্স বাজেট হলো আপনার সাইটের সর্বোচ্চ অনুমোদিত সাইজ ও লোড টাইম।
বাজেট ছাড়া পারফরম্যান্স ধীরে ধীরে খারাপ হতে থাকে।

### বাংলাদেশী ই-কমার্স সাইটের জন্য বাজেট উদাহরণ

| রিসোর্স            | বাজেট (gzipped) | কারণ                               |
|--------------------|-----------------|------------------------------------|
| মোট JavaScript     | ≤ 300 KB        | 3G-তে ৪s এর মধ্যে interactive       |
| মোট CSS            | ≤ 80 KB         | রেন্ডার ব্লকিং কম                  |
| মোট Images (ATF)   | ≤ 200 KB        | above-the-fold দ্রুত লোড            |
| মোট HTML           | ≤ 50 KB         | সার্ভার রেসপন্স ছোট                 |
| Web Fonts          | ≤ 100 KB        | FOUT/FOIT কম                       |
| **মোট পেজ সাইজ**   | **≤ 750 KB**    | **3G-তে ১০s এর মধ্যে fully loaded** |

| মেট্রিক            | বাজেট          |
|--------------------|----------------|
| LCP                | ≤ 2.5 সেকেন্ড  |
| INP                | ≤ 200ms        |
| CLS                | ≤ 0.1          |
| Time to Interactive| ≤ 5.0 সেকেন্ড  |
| Speed Index        | ≤ 4.0 সেকেন্ড  |

### বাজেট Enforce করার পদ্ধতি

```javascript
// webpack.config.js — বান্ডেল সাইজ লিমিট
module.exports = {
  performance: {
    maxAssetSize: 250000,      // ২৫০KB per asset
    maxEntrypointSize: 300000, // ৩০০KB per entrypoint
    hints: 'error',            // লিমিট ক্রস করলে build fail
  },
};
```

```json
// package.json — bundlesize check
{
  "bundlesize": [
    { "path": "dist/main.*.js", "maxSize": "150 kB" },
    { "path": "dist/vendor.*.js", "maxSize": "120 kB" },
    { "path": "dist/main.*.css", "maxSize": "30 kB" }
  ]
}
```

```yaml
# CI/CD — Lighthouse CI
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [pull_request]
jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: treosh/lighthouse-ci-action@v10
        with:
          budgetPath: ./budget.json
          uploadArtifacts: true
```

---

## ⚡ অতিরিক্ত অপটিমাইজেশন টেকনিক

### Resource Hints

```html
<head>
  <!-- ✅ DNS আগেই resolve করুন -->
  <link rel="dns-prefetch" href="//api.daraz.com.bd" />
  <link rel="dns-prefetch" href="//cdn.daraz.com.bd" />

  <!-- ✅ গুরুত্বপূর্ণ রিসোর্স আগেই connect করুন -->
  <link rel="preconnect" href="https://fonts.googleapis.com" crossorigin />

  <!-- ✅ পরের পেজের রিসোর্স আগেই fetch করুন -->
  <link rel="prefetch" href="/product-page-bundle.js" />

  <!-- ✅ ক্রিটিকাল রিসোর্স আগেই লোড করুন -->
  <link rel="preload" href="/fonts/bangla.woff2" as="font" 
        type="font/woff2" crossorigin />
  <link rel="preload" href="/hero-banner.webp" as="image" />
</head>
```

### Font অপটিমাইজেশন

```css
/* ✅ ফন্ট swap — টেক্সট আগে দেখান, ফন্ট পরে */
@font-face {
  font-family: 'NotoSansBengali';
  src: url('/fonts/NotoSansBengali.woff2') format('woff2');
  font-display: swap; /* গুরুত্বপূর্ণ! */
  unicode-range: U+0980-09FF; /* শুধু বাংলা characters */
}

/* ❌ font-display না দিলে FOIT হয় — ফন্ট লোড না হওয়া পর্যন্ত
   টেক্সট invisible থাকে */
```

### Virtualization (লম্বা লিস্ট)

```javascript
// Daraz-এ হাজার হাজার প্রোডাক্ট — সব একসাথে render করবেন না!

// ❌ সব আইটেম render
function ProductList({ products }) {
  return (
    <div>
      {products.map(p => <ProductCard key={p.id} product={p} />)}
      {/* ১০,০০০ আইটেম = ১০,০০০ DOM node = ধীর! */}
    </div>
  );
}

// ✅ Virtualized list — শুধু visible আইটেম render
import { FixedSizeList } from 'react-window';

function ProductList({ products }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <ProductCard product={products[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={products.length}
      itemSize={120}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
  // শুধু ৫-৬টি visible আইটেম DOM-এ থাকে, বাকিগুলো virtual
}
```

### Debounce ও Throttle

```javascript
// Pathao-এর search bar — প্রতি keystroke-এ API call করবেন না!

// ❌ প্রতি keystroke-এ API call
searchInput.addEventListener('input', (e) => {
  fetch(`/api/search?q=${e.target.value}`); // অপচয়!
});

// ✅ Debounce — ইউজার টাইপ থামলে API call
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

const debouncedSearch = debounce((query) => {
  fetch(`/api/search?q=${query}`).then(r => r.json()).then(showResults);
}, 300); // ৩০০ms অপেক্ষা

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value);
});

// ✅ Throttle — scroll event-এ ব্যবহার (সর্বোচ্চ X বার/সেকেন্ড)
function throttle(fn, limit) {
  let inThrottle = false;
  return (...args) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

window.addEventListener('scroll', throttle(handleScroll, 100));
```

---

## 🔧 টুলস (Performance Tools)

### ১. Lighthouse

```
Lighthouse Score কিভাবে পড়বেন
═══════════════════════════════════════

  ┌────────────────────────────────────┐
  │  Performance:    92  ✅ (≥90 ভালো) │
  │  Accessibility:  88  ⚠️            │
  │  Best Practices: 95  ✅            │
  │  SEO:            100 ✅            │
  │  PWA:            ✅                │
  └────────────────────────────────────┘

  চালানোর উপায়:
  ১. Chrome DevTools → Lighthouse tab
  ২. CLI: npx lighthouse https://daraz.com.bd
  ৩. PageSpeed Insights (web)
```

```bash
# CLI-তে Lighthouse চালান
npx lighthouse https://your-site.com \
  --output=html \
  --output-path=./report.html \
  --throttling.cpuSlowdownMultiplier=4 \
  --emulated-form-factor=mobile
```

### ২. Chrome DevTools Performance Tab

```
Performance Tab ব্যবহার
═══════════════════════════════════════

  ১. Chrome DevTools খুলুন (F12)
  ২. Performance tab যান
  ৩. Record বাটন ক্লিক করুন (বা Ctrl+Shift+E)
  ৪. পেজ রিলোড করুন
  ৫. Stop recording

  যা দেখবেন:
  ┌──────────────────────────────────────┐
  │  Network    ▓▓▓▓░░░░░░░░░░░░░░░░░░  │
  │  Main       ░░▓▓▓▓▓▓░░░░░▓▓░░░░░░  │
  │  Compositor ░░░░░░░░░▓░░░░░░▓░░░░░  │
  │  GPU        ░░░░░░░░░░▓░░░░░░▓░░░░  │
  │                                      │
  │  Long Tasks (>50ms) = লাল ত্রিভুজ 🔺│
  │  Layout Shift = বেগুনি ব্লক          │
  └──────────────────────────────────────┘
```

### ৩. WebPageTest

```
WebPageTest (webpagetest.org)
═══════════════════════════════════════

  সুবিধা:
  ✅ বিভিন্ন লোকেশন থেকে টেস্ট (Singapore, Mumbai)
  ✅ বিভিন্ন ডিভাইস ও নেটওয়ার্ক সিমুলেশন
  ✅ Waterfall chart — কোন রিসোর্স কখন লোড হচ্ছে
  ✅ Filmstrip view — পেজ কিভাবে ধাপে ধাপে render হচ্ছে
  ✅ ফ্রি!
```

### ৪. Bundle Analyzer

```javascript
// webpack-bundle-analyzer — কোন মডিউল কত বড় দেখুন
// npm install --save-dev webpack-bundle-analyzer

const BundleAnalyzerPlugin = 
  require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html',
    }),
  ],
};
// বিশাল treemap দেখাবে — কোন library কত জায়গা নিচ্ছে
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন পারফরম্যান্স অপটিমাইজেশন করবেন

| পরিস্থিতি | কেন |
|-----------|-----|
| প্রোডাকশন অ্যাপ | ইউজার experience সরাসরি প্রভাবিত হয় |
| মোবাইল-ফার্স্ট সাইট | বাংলাদেশের ৮০%+ ইউজার মোবাইলে |
| ই-কমার্স (Daraz, Chaldal) | ধীর = কম বিক্রি |
| পেমেন্ট ফ্লো (bKash, Nagad) | ধীর = ইউজার আস্থা হারায় |
| SEO গুরুত্বপূর্ণ | Core Web Vitals = ranking factor |
| হাই ট্রাফিক সাইট | সার্ভার খরচও কমে |
| slow নেটওয়ার্ক ইউজার | গ্রামাঞ্চলের ইউজারদের জন্য |

### ❌ কখন premature optimization করবেন না

| পরিস্থিতি | কেন |
|-----------|-----|
| MVP / Prototype | আগে ফিচার বানান, পরে optimize করুন |
| Internal tools | ১০ জন ইউজার → পারফরম্যান্স কম গুরুত্বপূর্ণ |
| ছোট সাইট (< 5 পেজ) | সাধারণত fast enough |
| মাইক্রো-অপটিমাইজেশন | ১ms বাঁচানোর চেষ্টায় ১ ঘণ্টা খরচ = অপচয় |
| পরিমাপ ছাড়া | আগে measure করুন, তারপর optimize করুন! |

### 📖 পারফরম্যান্স অপটিমাইজেশনের সুবর্ণ নিয়ম

```
"প্রথমে পরিমাপ করুন, তারপর অপটিমাইজ করুন"
"Measure first, then optimize"

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Measure  │────▶│ Identify │────▶│  Fix     │
  │ (টুলস   │     │ (বড়     │     │ (সবচেয়ে │
  │  দিয়ে)  │     │ সমস্যা  │     │ impactful│
  │          │     │ খুঁজুন) │     │  আগে)   │
  └──────────┘     └──────────┘     └──────┬───┘
       ▲                                    │
       │                                    │
       └────────────────────────────────────┘
              (আবার measure করুন)
```

---

## 📊 পারফরম্যান্স চেকলিস্ট

প্রতিটি ফ্রন্টেন্ড প্রোজেক্টে এই চেকলিস্ট অনুসরণ করুন:

```
ফ্রন্টেন্ড পারফরম্যান্স চেকলিস্ট
═════════════════════════════════════════

  JavaScript:
  □ Code splitting / lazy loading আছে
  □ Tree shaking enabled
  □ বান্ডেল সাইজ < 300KB (gzipped)
  □ Third-party scripts defer/async
  □ Unused code removed

  Images:
  □ WebP/AVIF ফরম্যাট ব্যবহার
  □ Responsive images (srcset)
  □ Lazy loading enabled
  □ width/height attributes দেওয়া
  □ Image CDN ব্যবহার

  CSS:
  □ Critical CSS inlined
  □ Non-critical CSS async loaded
  □ Unused CSS removed
  □ Minified

  Caching:
  □ Static assets long cache (1y)
  □ HTML no-cache
  □ Service Worker for offline
  □ CDN configured

  Network:
  □ Gzip/Brotli compression
  □ HTTP/2 enabled
  □ Resource hints (preconnect, prefetch)
  □ DNS prefetch for third parties

  Monitoring:
  □ Core Web Vitals tracking
  □ Performance budget set
  □ Lighthouse CI in pipeline
  □ Real User Monitoring (RUM)
```

---

## 🔗 উপকারী রিসোর্স

| রিসোর্স | লিংক |
|---------|------|
| web.dev (Performance) | https://web.dev/performance |
| Core Web Vitals | https://web.dev/vitals |
| Lighthouse | https://developer.chrome.com/docs/lighthouse |
| WebPageTest | https://www.webpagetest.org |
| Bundle Analyzer | https://github.com/webpack-contrib/webpack-bundle-analyzer |
| web-vitals npm | https://github.com/GoogleChrome/web-vitals |
| Can I Use | https://caniuse.com |

---

## 📝 সারসংক্ষেপ

```
ফ্রন্টেন্ড পারফরম্যান্স — মূল কথা
════════════════════════════════════════════

  ১. পরিমাপ করুন (Lighthouse, DevTools, RUM)
  ২. Core Web Vitals টার্গেট ঠিক করুন
  ৩. JavaScript ছোট ও split রাখুন
  ৪. ইমেজ WebP/AVIF-এ optimize করুন
  ৫. ক্যাশিং সঠিকভাবে সেটআপ করুন
  ৬. Resource hints ব্যবহার করুন
  ৭. পারফরম্যান্স বাজেট enforce করুন
  ৮. আবার পরিমাপ করুন — cycle চালু রাখুন!

  বাংলাদেশের ইউজারদের কথা মনে রাখুন:
  📱 লো-এন্ড ডিভাইস
  📶 Slow নেটওয়ার্ক
  💰 ব্যয়বহুল ডাটা
  → প্রতিটি KB গুরুত্বপূর্ণ!
```
