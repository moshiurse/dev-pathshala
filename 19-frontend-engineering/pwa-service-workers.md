# 📲 PWA & Service Workers — অফলাইন-ফার্স্ট ওয়েব

## 📌 সংজ্ঞা

**Progressive Web App (PWA)** হলো এমন একটি ওয়েব অ্যাপ যা **native app-এর মতো অভিজ্ঞতা**
দেয়: হোম স্ক্রিনে install হয়, অফলাইনে কাজ করে, push notification পাঠায়, fast load হয়।
PWA কোনো ফ্রেমওয়ার্ক না — এটি ওয়েব স্ট্যান্ডার্ডের একটা সেট।

**Service Worker** হলো একটি JavaScript ফাইল যা **ব্রাউজার ও নেটওয়ার্কের মাঝে proxy**
হিসেবে কাজ করে। এটি background thread-এ চলে, পেজ বন্ধ থাকলেও চলতে পারে, এবং network
request intercept করতে পারে।

বাংলাদেশে গুরুত্বপূর্ণ কারণ:
- 📶 মেট্রো এলাকার বাইরে network unreliable
- 💾 প্রতিবার ডাটা লোড = টাকা খরচ
- 📱 অনেকের কাছে app install করার মতো storage নেই (16GB phone)
- 🛒 Pathao Foods, Foodpanda, Chaldal — অর্ডার অর্ধেক পথে network গেলে?

---

## 🎯 PWA-র ৩ স্তম্ভ

```
  ┌─────────────────────────────────────────────────────┐
  │  Web App Manifest  →  "এটি কীভাবে install হবে"     │
  │  Service Worker    →  "এটি কীভাবে অফলাইনে চলবে"    │
  │  HTTPS             →  "এটি নিরাপদ"                  │
  └─────────────────────────────────────────────────────┘
```

### App-like vs Web

| বৈশিষ্ট্য | Native App | PWA |
|----------|-----------|-----|
| Install size | 50-200 MB | 0.5-5 MB |
| Update | App Store approval | তাত্ক্ষণিক (server push) |
| Cross-platform | আলাদা কোডবেস | একটাই |
| App Store fee | ৩০% | নাই |
| Offline | হ্যাঁ | হ্যাঁ (Service Worker দিয়ে) |
| Push notification | হ্যাঁ | হ্যাঁ (iOS 16.4+ সহ) |
| Hardware access | পূর্ণ | সীমিত (camera, GPS, BT) |
| SEO | নেই | আছে |
| Discoverability | App Store | URL share |

---

## 📜 Web App Manifest

`manifest.webmanifest` ফাইল browser-কে বলে কীভাবে অ্যাপ install হবে।

```json
{
  "name": "Pathao Foods Bangladesh",
  "short_name": "Pathao Foods",
  "description": "ঢাকার বেস্ট রেস্টুরেন্ট থেকে দ্রুত ফুড ডেলিভারি",
  "start_url": "/?source=pwa",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#e91e63",
  "lang": "bn-BD",
  "dir": "ltr",
  "icons": [
    { "src": "/icons/icon-72.png",  "sizes": "72x72",   "type": "image/png" },
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icons/maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "shortcuts": [
    {
      "name": "নতুন অর্ডার",
      "url": "/order/new",
      "icons": [{ "src": "/icons/order.png", "sizes": "96x96" }]
    },
    {
      "name": "আমার অর্ডার",
      "url": "/orders",
      "icons": [{ "src": "/icons/list.png", "sizes": "96x96" }]
    }
  ],
  "screenshots": [
    { "src": "/ss/home.png", "sizes": "1080x1920", "type": "image/png", "form_factor": "narrow" }
  ]
}
```

### display মোড

```
  fullscreen  →  পুরো স্ক্রিন (game)
  standalone  →  status bar আছে, browser UI নেই (native look — সবচেয়ে কমন)
  minimal-ui  →  ছোট browser UI
  browser     →  সাধারণ ব্রাউজার ট্যাব
```

### HTML-এ link

```html
<link rel="manifest" href="/manifest.webmanifest">
<meta name="theme-color" content="#e91e63">
<link rel="apple-touch-icon" href="/icons/icon-192.png">
```

### Maskable icons

Android-এ launcher icon-কে different shape (circle/square/squircle) দিতে পারে। `purpose:
"maskable"` icon-এ subject-কে "safe zone"-এ রাখুন (center 80%)।

---

## ⚙️ Service Worker — Lifecycle

Service Worker browser-এ ৩টি phase-এ চলে:

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  ① Register   →  ② Install   →  ③ Activate              │
  │                                                          │
  │  ┌─────┐         ┌─────────┐     ┌──────────┐            │
  │  │ JS  │────────▶│installing│───▶│activated │───┐        │
  │  │ regs│         └─────────┘     └──────────┘   │        │
  │  └─────┘              │              ▲          │        │
  │                       │              │     fetch│push    │
  │                       ▼              │     sync │msg     │
  │                  ┌─────────┐         │          │        │
  │                  │installed│ waiting │          ▼        │
  │                  └─────────┘ ────────┘     handles events│
  │                       │                                  │
  │                       ▼                                  │
  │                  ┌──────────┐                            │
  │                  │redundant │ (নতুন SW এলে)              │
  │                  └──────────┘                            │
  └──────────────────────────────────────────────────────────┘
```

### রেজিস্ট্রেশন

```javascript
// app.js (main thread)
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register('/sw.js', {
        scope: '/'
      });
      console.log('SW registered:', reg.scope);

      // আপডেট চেক
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing;
        newWorker?.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            // নতুন SW রেডি, ইউজারকে refresh-এর notification দিন
            showUpdateBanner();
          }
        });
      });
    } catch (err) {
      console.error('SW failed:', err);
    }
  });
}
```

### sw.js — Lifecycle events

```javascript
// sw.js
const CACHE_NAME = 'pathao-foods-v3';
const APP_SHELL = [
  '/',
  '/index.html',
  '/css/app.css',
  '/js/app.js',
  '/icons/icon-192.png',
  '/offline.html'
];

// ────────── install ──────────
self.addEventListener('install', (event) => {
  console.log('[SW] install');
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(APP_SHELL))
  );
  // নতুন SW সাথে সাথে activate (নাহলে waiting-এ থাকে)
  self.skipWaiting();
});

// ────────── activate ──────────
self.addEventListener('activate', (event) => {
  console.log('[SW] activate');
  event.waitUntil(
    Promise.all([
      // পুরনো cache delete
      caches.keys().then(keys =>
        Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
      ),
      // আগের পেজগুলোও এই SW দিয়ে control
      self.clients.claim()
    ])
  );
});

// ────────── fetch ──────────
self.addEventListener('fetch', (event) => {
  // ... caching strategy
});
```

### Scope — গুরুত্বপূর্ণ নিয়ম

```
  /sw.js                → scope: /  (সম্পূর্ণ origin control)
  /app/sw.js            → scope: /app/  (শুধু /app/* control)
  
  ⚠️ SW নিজের URL-এর "নিচে" থাকা path control করতে পারে, উপরে নয়
  ⚠️ root /-এ রাখলে সবচেয়ে নিরাপদ
```

---

## 🗂️ Cache API + Caching Strategies

Service Worker-এর সবচেয়ে গুরুত্বপূর্ণ কাজ — **fetch event intercept** করে cache থেকে
serve করা।

### সাধারণ ৫টি Strategy

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  ① Cache First                                              │
  │     cache → না পেলে network → response cache-এ রাখে        │
  │     ব্যবহার: static assets (CSS, JS, fonts, logo)           │
  │                                                              │
  │  ② Network First                                            │
  │     network → ব্যর্থ হলে cache                              │
  │     ব্যবহার: API responses যেখানে fresh data দরকার         │
  │                                                              │
  │  ③ Stale While Revalidate                                   │
  │     cache দ্রুত return, পেছনে network আপডেট cache          │
  │     ব্যবহার: news, product list, profile                    │
  │                                                              │
  │  ④ Network Only                                             │
  │     শুধু network, cache-এ রাখে না                           │
  │     ব্যবহার: analytics, payment, login                      │
  │                                                              │
  │  ⑤ Cache Only                                               │
  │     শুধু cache, network-এ যায় না                           │
  │     ব্যবহার: pre-cached assets, offline-only resources     │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### বিস্তারিত implementation

```javascript
// sw.js
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // ১। App shell (HTML) — Network first
  if (request.mode === 'navigate') {
    event.respondWith(networkFirst(request, '/offline.html'));
    return;
  }

  // ২। Static assets — Cache first
  if (/\.(css|js|woff2|png|jpg|svg)$/.test(url.pathname)) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // ৩। API GET — Stale while revalidate
  if (url.pathname.startsWith('/api/') && request.method === 'GET') {
    event.respondWith(staleWhileRevalidate(request));
    return;
  }

  // ৪। Payment/auth — Network only
  if (url.pathname.startsWith('/api/payment') ||
      url.pathname.startsWith('/api/auth')) {
    event.respondWith(fetch(request));
    return;
  }
});

// ────────── Cache First ──────────
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  try {
    const fresh = await fetch(request);
    if (fresh.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, fresh.clone());
    }
    return fresh;
  } catch {
    return new Response('Offline', { status: 503 });
  }
}

// ────────── Network First ──────────
async function networkFirst(request, fallbackUrl) {
  try {
    const fresh = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, fresh.clone());
    return fresh;
  } catch {
    const cached = await caches.match(request);
    return cached || caches.match(fallbackUrl);
  }
}

// ────────── Stale While Revalidate ──────────
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request)
    .then(response => {
      if (response.ok) cache.put(request, response.clone());
      return response;
    })
    .catch(() => cached);

  return cached || fetchPromise;
}
```

### কোথায় কোন strategy?

| Resource | Strategy | কারণ |
|----------|----------|------|
| HTML pages | Network First | ছোটখাটো আপডেট চাই |
| CSS / JS bundles (hashed) | Cache First | hash-এ version, immutable |
| Fonts | Cache First | কখনো বদলায় না |
| Restaurant menu (Pathao Foods) | SWR | এক ঘণ্টায় বদলায় না কিন্তু freshness ভালো |
| Cart contents | Network First → IndexedDB fallback | offline-first |
| Order history | SWR | পুরনো ডেটা OK, ব্যাকগ্রাউন্ডে refresh |
| Live order tracking | Network Only | real-time অপরিহার্য |
| `/api/payment/charge` | Network Only | কখনো cache না |
| User avatar | Cache First, expire 7d | বেশি বদলায় না |

---

## 🏠 App Shell Pattern

App shell = অ্যাপের minimum HTML/CSS/JS যা ছাড়া UI render হয় না।

```
  ┌─────────────────────────────────────────────────┐
  │ ┌─────────────────────────────────────────────┐ │
  │ │             Header (Pathao Foods)           │ │
  │ ├─────────────────────────────────────────────┤ │  ← App Shell
  │ │ Sidebar │                                   │ │     (always cached)
  │ │         │     ⏳ Loading content...         │ │
  │ │         │                                   │ │
  │ │         │    ← এই অংশ network থেকে        │ │
  │ │         │                                   │ │
  │ ├─────────────────────────────────────────────┤ │  ← App Shell
  │ │              Footer                         │ │
  │ └─────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────┘

  Strategy:
   App shell → Cache First (instant load)
   Content   → Network First / SWR
```

ফলাফল: প্রথম paint তাত্ক্ষণিক, ইউজার মনে করে "অ্যাপ" খুলেছে — তারপর content আসে।

---

## 🔄 Background Sync

ইউজার অফলাইনে অর্ডার দিলে — service worker network পেলে server-এ পাঠাবে।

```javascript
// app.js — Pathao Foods অর্ডার
async function placeOrder(orderData) {
  try {
    // network আছে — সরাসরি পাঠান
    await fetch('/api/orders', {
      method: 'POST',
      body: JSON.stringify(orderData)
    });
  } catch {
    // network নেই — IndexedDB-তে জমা রেখে background sync
    await db.pendingOrders.add(orderData);
    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register('sync-orders');
    showToast('অর্ডার রেডি — network এলেই পাঠানো হবে');
  }
}

// sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-orders') {
    event.waitUntil(flushPendingOrders());
  }
});

async function flushPendingOrders() {
  const orders = await db.pendingOrders.toArray();
  for (const order of orders) {
    try {
      await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify(order)
      });
      await db.pendingOrders.delete(order.id);
    } catch {
      // নেক্সট sync-এ আবার চেষ্টা
      throw new Error('retry');
    }
  }
}
```

> ⚠️ Background Sync iOS Safari-তে এখনো নেই (২০২৪)। Periodic Background Sync শুধু
> Chrome-এ। Fallback হিসেবে `online` event listener রাখুন।

---

## 🔔 Push Notifications

Foodpanda অর্ডার রেডি — push notification পাঠাবে কীভাবে?

### ১। Subscription

```javascript
// app.js
async function subscribeToPush() {
  const reg = await navigator.serviceWorker.ready;
  
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') return;

  const subscription = await reg.pushManager.subscribe({
    userVisibleOnly: true,  // অবশ্যই UI-তে notification দেখাতে হবে
    applicationServerKey: VAPID_PUBLIC_KEY  // server-এর VAPID public key
  });

  // server-এ পাঠান
  await fetch('/api/push/subscribe', {
    method: 'POST',
    body: JSON.stringify(subscription)
  });
}
```

### ২। Server থেকে push পাঠানো (Node.js)

```javascript
import webpush from 'web-push';

webpush.setVapidDetails(
  'mailto:dev@foodpanda.com.bd',
  VAPID_PUBLIC_KEY,
  VAPID_PRIVATE_KEY
);

await webpush.sendNotification(subscription, JSON.stringify({
  title: 'অর্ডার রেডি! 🎉',
  body: 'আপনার অর্ডার #1234 ডেলিভারি হচ্ছে',
  url: '/orders/1234',
  badge: '/icons/badge.png',
  icon: '/icons/icon-192.png'
}));
```

### ৩। Service Worker-এ receive

```javascript
// sw.js
self.addEventListener('push', (event) => {
  const data = event.data.json();
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: data.icon,
      badge: data.badge,
      vibrate: [100, 50, 100],
      data: { url: data.url },
      actions: [
        { action: 'view', title: 'দেখুন' },
        { action: 'dismiss', title: 'বন্ধ' }
      ]
    })
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  if (event.action === 'dismiss') return;

  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(clientList => {
      // ইতিমধ্যে অ্যাপ খোলা থাকলে সেটায় ফোকাস
      for (const client of clientList) {
        if ('focus' in client) {
          client.navigate(event.notification.data.url);
          return client.focus();
        }
      }
      return clients.openWindow(event.notification.data.url);
    })
  );
});
```

---

## 📦 Workbox — সহজে SW লেখা

Google-এর Workbox library SW boilerplate কমায়।

```bash
npm install workbox-webpack-plugin
```

### webpack.config.js

```javascript
import { GenerateSW } from 'workbox-webpack-plugin';

export default {
  // ...
  plugins: [
    new GenerateSW({
      clientsClaim: true,
      skipWaiting: true,
      runtimeCaching: [
        {
          urlPattern: /\.(?:png|jpg|jpeg|svg|webp)$/,
          handler: 'CacheFirst',
          options: {
            cacheName: 'images',
            expiration: { maxEntries: 100, maxAgeSeconds: 30 * 24 * 60 * 60 }
          }
        },
        {
          urlPattern: /^https:\/\/api\.pathao\.com\/menu/,
          handler: 'StaleWhileRevalidate',
          options: { cacheName: 'menu-api' }
        },
        {
          urlPattern: /^https:\/\/api\.pathao\.com\/payment/,
          handler: 'NetworkOnly'
        }
      ]
    })
  ]
};
```

### Custom SW + Workbox modules

```javascript
// sw.js
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { precacheAndRoute } from 'workbox-precaching';
import { BackgroundSyncPlugin } from 'workbox-background-sync';

precacheAndRoute(self.__WB_MANIFEST);  // build-time injected list

registerRoute(
  /\.(?:png|webp)$/,
  new CacheFirst({
    cacheName: 'img',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 7 * 24 * 60 * 60 })
    ]
  })
);

const bgSync = new BackgroundSyncPlugin('orders-queue', {
  maxRetentionTime: 24 * 60  // 24 hours
});

registerRoute(
  /\/api\/orders/,
  new NetworkFirst({ plugins: [bgSync] }),
  'POST'
);
```

---

## 🛒 কেস স্টাডি — Pathao Foods Offline Cart

ইউজার মেট্রো রেলে — network নেই। কিন্তু লাঞ্চের অর্ডার দিতে চান।

```javascript
// app.js — IndexedDB (Dexie) দিয়ে cart
import Dexie from 'dexie';

const db = new Dexie('pathao-foods');
db.version(1).stores({
  cart: '++id, restaurantId, itemId, qty',
  pendingOrders: '++id, payload, createdAt'
});

// add to cart — সবসময় local first
async function addToCart(item) {
  await db.cart.add(item);
  // online হলে server cart sync
  if (navigator.onLine) syncCart();
}

// অর্ডার সাবমিট
async function checkout(payload) {
  if (navigator.onLine) {
    try {
      const res = await fetch('/api/orders', {
        method: 'POST',
        body: JSON.stringify(payload)
      });
      if (res.ok) {
        await db.cart.clear();
        return await res.json();
      }
    } catch {}
  }

  // অফলাইন বা failed — queue-এ যোগ
  const id = await db.pendingOrders.add({
    payload,
    createdAt: Date.now()
  });
  
  const reg = await navigator.serviceWorker.ready;
  if ('sync' in reg) {
    await reg.sync.register('flush-orders');
  }
  
  return { offline: true, queueId: id };
}
```

### UX এ communicate

```
  ┌────────────────────────────────────────────────────┐
  │ 🍔 কাচ্চি ভাই — বিরিয়ানি ৩৫০৳                       │
  │                                                    │
  │ 🟡 অফলাইনে আছেন                                     │
  │ অর্ডার রেডি করা হয়েছে — internet এলেই           │
  │ পাঠানো হবে                                         │
  │                                                    │
  │ [ অর্ডার রেডি করুন ]                                │
  └────────────────────────────────────────────────────┘
```

ইউজারকে স্পষ্ট বলুন: "অফলাইনে আছেন" — কিন্তু কাজ থামাবেন না।

---

## 🧪 Testing & Debugging

```
  Chrome DevTools → Application tab
  ─────────────────────────────────────
  • Service Workers → status, update on reload, bypass for network
  • Cache Storage  → cache contents inspect
  • IndexedDB     → DB inspect
  • Manifest       → install eligibility check
  • Storage        → quota check, clear all

  Lighthouse → PWA category
  ─────────────────────────────────────
  • Installable check
  • PWA optimized
  • Manifest validity
```

### chrome://inspect/#service-workers

ফোনে DevTools চালু করে SW এর actual behavior দেখা।

---

## ⚠️ সাধারণ ভুল

```
  ❌ POST request cache করা — Cache API শুধু GET সাপোর্ট করে
  ❌ Auth token cache-এ — অন্য ইউজারকে পেয়ে যাবে
  ❌ HTML সবসময় cache-first — আপডেট pull করতে পারবেন না
  ❌ skipWaiting() not handled — পুরনো ও নতুন SW একসাথে চলে
  ❌ Cache name versioning নেই — পুরনো cache কখনো delete হয় না
  ❌ scope ভুল — /sw.js কে /admin/sw.js-এ রেখে / control আশা
  ❌ HTTP-এ SW রান করার চেষ্টা — শুধু HTTPS / localhost
  ❌ চিরকাল cache — quota ফুরিয়ে যাবে; expiration প্লাগইন দিন
  ❌ Update banner নেই — ইউজার পুরনো version-এ আটকে থাকে
  ❌ Service worker নিজে ব্যবহার করছে fetch দিয়ে chain করে
```

---

## ✅ যেখানে PWA ব্যবহার করুন

- E-commerce (Daraz, Chaldal): SEO + offline browse
- Food delivery (Pathao Foods, Foodpanda): offline cart
- News (Prothom Alo, BD News24): offline reading
- Banking (bKash, Nagad): কিন্তু payment Network Only
- Education (10 Minute School): offline lesson access

## ❌ যেখানে কম দরকার

- পুরোপুরি real-time app (ride tracking — তাও app shell দরকার)
- Heavy native API দরকার (Bluetooth printer, USB device)
- iOS-only (PWA support সীমিত হলেও ২০২৪-এ অনেক ভালো)

---

## 📋 সারসংক্ষেপ

```
  PWA Checklist
  ════════════════════════════════════════════════
  □ HTTPS-এ deploy
  □ Web App Manifest (with maskable icon)
  □ Service Worker registered, scope="/"
  □ App shell পেজে precached
  □ অফলাইন fallback পেজ (/offline.html)
  □ Static asset → Cache First (versioned)
  □ HTML → Network First
  □ API GET → SWR
  □ Payment → Network Only
  □ Update notification (skipWaiting + clients.claim)
  □ Background Sync for queued mutations
  □ Push notification (যদি প্রয়োজন)
  □ Lighthouse PWA score ≥ 90
```

PWA মানে শুধু "অ্যাপ install হবে" না — এটা একটা **mindset**: ইউজারের network যেমনই থাকুক
(2G, 3G, slow Wi-Fi, no signal), অ্যাপ যেন সবসময় কাজ করে। বাংলাদেশের contextে এটাই
সবচেয়ে বড় UX win।
