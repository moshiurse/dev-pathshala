# ⚡ লোডিং ডিজাইন প্যাটার্ন (Loading Design Patterns)

> **ক্যাটাগরি:** Performance Patterns
> **উৎস:** patterns.dev — PRPL, Import on Visibility, Import on Interaction, Virtual List

---

## ১. PRPL প্যাটার্ন

### সংজ্ঞা

**PRPL** হলো একটি পারফরম্যান্স-ফোকাসড লোডিং কৌশল যা মোবাইল নেটওয়ার্কে দ্রুত পেজ লোডের জন্য ডিজাইন করা হয়েছে।

```
   PRPL = Push + Render + Pre-cache + Lazy-load
   ════════════════════════════════════════════════════════

   P → Push / Preload    সবচেয়ে গুরুত্বপূর্ণ রিসোর্স প্রথমে পাঠাও
   R → Render            যত দ্রুত সম্ভব প্রথম রুট রেন্ডার করো
   P → Pre-cache         বাকি রুটগুলো Service Worker-এ ক্যাশ করো
   L → Lazy-load         চাহিদামতো বাকি রুট লোড করো
```

```
   PRPL ফ্লো — বাংলাদেশ ৩G নেটওয়ার্কে Daraz অ্যাপ
   ════════════════════════════════════════════════════════

   ব্রাউজার        Service Worker         Server
      │                  │                   │
      │ GET /            │                   │
      │─────────────────────────────────────▶│
      │                  │        HTML+critical CSS+JS
      │◀─────────────────────────────────────│
      │ [P] Preload      │                   │
      │ critical assets  │                   │
      │─────────────────▶│                   │
      │ [R] Render       │                   │
      │ initial route ✅ │                   │
      │                  │ [P] Pre-cache     │
      │                  │ other routes      │
      │                  │──────────────────▶│
      │ [L] Navigate     │                   │
      │ /product/123     │                   │
      │─────────────────▶│ (cache hit!) ✅   │
```

### বাস্তব বাস্তবায়ন

```javascript
// vite.config.js — PRPL-style bundle splitting
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor:   ['react', 'react-dom'],           // P: Preload
          router:   ['react-router-dom'],              // P: Preload
          checkout: ['./src/pages/Checkout'],          // L: Lazy
          profile:  ['./src/pages/Profile'],           // L: Lazy
        },
      },
    },
  },
});
```

```html
<!-- index.html — Critical resource hints -->
<!-- [P] Push — সবচেয়ে জরুরি জিনিস প্রথমে -->
<link rel="preload" href="/assets/main.js"        as="script">
<link rel="preload" href="/assets/vendor.js"      as="script">
<link rel="preload" href="/assets/main.css"       as="style">
<link rel="preload" href="/fonts/hind-siliguri.woff2" as="font" crossorigin>

<!-- [P] Pre-cache — পরবর্তী পেজের জন্য -->
<link rel="prefetch" href="/assets/checkout.js">
<link rel="prefetch" href="/assets/profile.js">
```

```javascript
// service-worker.js — [P] Pre-cache
const CACHE   = 'daraz-v1';
const PRECACHE = [
  '/',
  '/assets/vendor.js',
  '/assets/main.css',
  '/offline.html',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE).then(cache => cache.addAll(PRECACHE))
  );
});

// [L] Lazy-load বাকি রুট on-demand
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(
      cached => cached ?? fetch(event.request)
    )
  );
});
```

---

## ২. Import on Visibility (দৃশ্যমানতায় ইম্পোর্ট)

### সংজ্ঞা

কমপোনেন্ট বা রিসোর্স শুধুমাত্র তখনই লোড হয় যখন সেটি ভিউপোর্টে (দৃশ্যমান অংশে) প্রবেশ করে।

```
   Import on Visibility ফ্লো
   ════════════════════════════════════════════════════════

   ┌─────────────────────────────┐  ← viewport শুরু
   │  Header (এখনই লোড)          │
   │  HeroBanner (এখনই লোড)      │
   │  ProductList (এখনই লোড)     │
   └─────────────────────────────┘  ← viewport শেষ
   
   ↕ ব্যবহারকারী স্ক্রল করলে...
   
   ┌─────────────────────────────┐
   │  Reviews ← এখন লোড হবে! 📦 │
   │  RelatedProducts ← লোড! 📦  │
   └─────────────────────────────┘
   
   [ Footer ] ← এখনো লোড হয়নি
```

### IntersectionObserver দিয়ে বাস্তবায়ন

```javascript
// useIntersectionObserver.js — Custom Hook
import { useState, useEffect, useRef } from 'react';

export function useIntersectionObserver(options = {}) {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setIsVisible(true);
        observer.disconnect(); // একবার দেখালে আর দরকার নেই
      }
    }, {
      rootMargin: '200px', // ২০০px আগেই লোড শুরু
      threshold: 0.1,
      ...options,
    });

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return [ref, isVisible];
}
```

```jsx
// LazySection.jsx — দৃশ্যমান হলে লোড হয়
import { lazy, Suspense } from 'react';
import { useIntersectionObserver } from './useIntersectionObserver';

export function LazySection({ importFn, fallback = <div>লোড হচ্ছে...</div> }) {
  const [ref, isVisible] = useIntersectionObserver();
  const [Component, setComponent] = useState(null);

  useEffect(() => {
    if (isVisible && !Component) {
      importFn().then(mod => setComponent(() => mod.default));
    }
  }, [isVisible]);

  return (
    <div ref={ref} style={{ minHeight: '100px' }}>
      {isVisible && Component ? (
        <Suspense fallback={fallback}>
          <Component />
        </Suspense>
      ) : fallback}
    </div>
  );
}

// ব্যবহার — Daraz Product Page
function ProductPage() {
  return (
    <div>
      <ProductInfo />   {/* সাথে সাথে লোড */}
      <AddToCart />     {/* সাথে সাথে লোড */}

      {/* নিচে স্ক্রল না করলে লোড হবে না */}
      <LazySection importFn={() => import('./CustomerReviews')} />
      <LazySection importFn={() => import('./RelatedProducts')} />
      <LazySection importFn={() => import('./RecentlyViewed')} />
    </div>
  );
}
```

---

## ৩. Import on Interaction (ইন্টারঅ্যাকশনে ইম্পোর্ট)

### সংজ্ঞা

ব্যবহারকারী কোনো অ্যাকশন (ক্লিক, হোভার, ফোকাস) করলে তবেই সেই ফিচারের কোড লোড হয়।

```
   Import on Interaction ফ্লো
   ════════════════════════════════════════════════════════

   ব্যবহারকারী পেজ লোড করল
         │
         ▼
   [কিছু লোড হয়নি এখনো]
   ChatWidget   → শুধু placeholder বোতাম
   EmojiPicker  → শুধু ইমোজি আইকন
   RichEditor   → শুধু একটি সাধারণ textarea

         │ ব্যবহারকারী Chat বোতামে ক্লিক করল
         ▼
   📦 ChatWidget.js লোড হচ্ছে... ✅
   (অন্য কিছু লোড হয়নি)
```

### বাস্তব উদাহরণ

```jsx
// ChatWidget — ক্লিক করলে তবেই লোড হয়
function ChatButton() {
  const [ChatWidget, setChatWidget] = useState(null);
  const [isOpen, setIsOpen]         = useState(false);
  const [loading, setLoading]       = useState(false);

  async function handleClick() {
    if (!ChatWidget) {
      setLoading(true);
      const mod = await import('./ChatWidget'); // ক্লিকে লোড!
      setChatWidget(() => mod.default);
      setLoading(false);
    }
    setIsOpen(true);
  }

  return (
    <>
      <button onClick={handleClick} disabled={loading}>
        {loading ? '⏳' : '💬'} চ্যাট করুন
      </button>
      {ChatWidget && isOpen && <ChatWidget onClose={() => setIsOpen(false)} />}
    </>
  );
}
```

```jsx
// EmojiPicker — হোভার করলে preload, ক্লিকে লোড
function EmojiButton() {
  const [Picker, setPicker]   = useState(null);
  const [showPicker, setShow] = useState(false);

  // হোভারে preload শুরু (ব্যাকগ্রাউন্ডে)
  function handleMouseEnter() {
    import('./EmojiPicker'); // preload — UI block করে না
  }

  // ক্লিকে সম্পূর্ণ লোড
  async function handleClick() {
    const mod = await import('./EmojiPicker');
    setPicker(() => mod.default);
    setShow(true);
  }

  return (
    <>
      <button onMouseEnter={handleMouseEnter} onClick={handleClick}>
        😊
      </button>
      {Picker && showPicker && <Picker onSelect={insertEmoji} />}
    </>
  );
}
```

---

## ৪. ভার্চুয়াল লিস্ট / লিস্ট ভার্চুয়ালাইজেশন

### সংজ্ঞা

হাজার হাজার আইটেম থাকলেও শুধু **দৃশ্যমান আইটেমগুলোই DOM-এ রেন্ডার** করে। বাকিগুলোর জন্য শুধু খালি জায়গা রিজার্ভ রাখা হয়।

```
   ভার্চুয়াল লিস্ট কীভাবে কাজ করে
   ════════════════════════════════════════════════════════

   মোট ১০,০০০ পণ্য আছে — কিন্তু DOM-এ মাত্র ১০টি!

   ┌────────────────────────────────┐ ─── viewport
   │ [ফাঁকা জায়গা: ০-৪৯৯ px]      │  ← rendered: না
   ├────────────────────────────────┤
   │ পণ্য #৫১ (scroll position এখানে)│ ← rendered: হ্যাঁ
   │ পণ্য #৫২                       │ ← rendered: হ্যাঁ
   │ পণ্য #৫৩                       │ ← rendered: হ্যাঁ
   │ ... ১০ টি মাত্র ...             │
   │ পণ্য #৬০                       │ ← rendered: হ্যাঁ
   ├────────────────────────────────┤
   │ [ফাঁকা জায়গা: ৬০১-৯৯৯৯ px]   │  ← rendered: না
   └────────────────────────────────┘
```

### react-window দিয়ে বাস্তবায়ন

```jsx
import { FixedSizeList, VariableSizeList } from 'react-window';

// FixedSizeList — প্রতিটি আইটেম একই উচ্চতা
function ProductList({ products }) {
  const Row = ({ index, style }) => {
    const product = products[index];
    return (
      <div style={style} className="product-row">
        <img src={product.image} alt={product.name} loading="lazy" />
        <div>
          <h3>{product.name}</h3>
          <span>৳{product.price.toLocaleString('bn-BD')}</span>
        </div>
      </div>
    );
  };

  return (
    <FixedSizeList
      height={600}          // viewport উচ্চতা
      itemCount={products.length}  // মোট আইটেম
      itemSize={80}         // প্রতিটি আইটেমের উচ্চতা
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

```jsx
// VariableSizeList — আইটেমের উচ্চতা ভিন্ন হলে
function ChatHistory({ messages }) {
  const getItemSize = (index) => {
    const msg = messages[index];
    return msg.isLong ? 120 : 60;
  };

  const Message = ({ index, style }) => (
    <div style={style} className="message">
      {messages[index].text}
    </div>
  );

  return (
    <VariableSizeList
      height={500}
      itemCount={messages.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Message}
    </VariableSizeList>
  );
}
```

```jsx
// TanStack Virtual — আধুনিক বিকল্প (Headless)
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualTable({ rows }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 45,
    overscan: 5, // viewport-এর বাইরে ৫টি প্রি-রেন্ডার
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <TableRow data={rows[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## ✅ সুবিধা তুলনা

| প্যাটার্ন | সমস্যা সমাধান | মূল সুবিধা |
|----------|-------------|-----------|
| PRPL | ধীর প্রথম লোড | দ্রুত TTI, Offline সাপোর্ট |
| Import on Visibility | অপ্রয়োজনীয় লোড | কম ব্যান্ডউইথ |
| Import on Interaction | ব্যবহার না হওয়া ফিচার লোড | দ্রুত Initial Load |
| Virtual List | হাজার আইটেম রেন্ডার | কম মেমোরি, মসৃণ স্ক্রল |

---

## 📝 কখন কোনটা ব্যবহার করবেন

- **PRPL** → PWA, অফলাইন সাপোর্ট দরকার হলে
- **Import on Visibility** → পেজের নিচের সেকশন, রিভিউ, কমেন্ট লোড করতে
- **Import on Interaction** → Chat widget, Rich Editor, Modal-এর ভেতরের কমপোনেন্ট
- **Virtual List** → ১০০+ আইটেমের তালিকা, ইনফিনিট স্ক্রল, ডেটা টেবিল
