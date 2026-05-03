# 🎬 ভিউ ট্রানজিশন (View Transitions)

> **ক্যাটাগরি:** Frontend Rendering Pattern
> **উৎস:** patterns.dev/vanilla/view-transitions

---

## সংজ্ঞা

**View Transitions API** ব্রাউজারের নেটিভ ফিচার যা পেজ বা DOM পরিবর্তনের মাঝে মসৃণ অ্যানিমেশন যোগ করে — আলাদা অ্যানিমেশন লাইব্রেরি ছাড়াই।

```
   View Transitions ছাড়া vs সহ
   ════════════════════════════════════════════════════════

   ❌ ছাড়া:
   পেজ A ──[ক্লিক]──▶ হঠাৎ পেজ B (ঝাঁকুনি 😵)

   ✅ সহ:
   পেজ A ──[ক্লিক]──▶ ✨ ফেড/স্লাইড অ্যানিমেশন ──▶ পেজ B 😍
```

---

## 🏠 বাস্তব উদাহরণ (Bangladesh Context)

> Pathao অ্যাপে রাইড বুকিং স্ক্রিন থেকে ম্যাপ স্ক্রিনে যাওয়া।
> Daraz-এ পণ্যের তালিকা থেকে পণ্যের বিস্তারিত পেজে যাওয়া — হিরো ইমেজ মসৃণভাবে বড় হয়।

---

## ✅ মৌলিক ব্যবহার

```javascript
// সহজতম ব্যবহার — document.startViewTransition()
function navigateTo(url) {
  if (!document.startViewTransition) {
    // পুরনো ব্রাউজারে fallback
    updateDOM(url);
    return;
  }

  document.startViewTransition(() => {
    updateDOM(url); // DOM পরিবর্তন এখানে
  });
}

function updateDOM(url) {
  document.querySelector('#main').innerHTML = getPageContent(url);
}
```

---

## ✅ React Router-এ View Transitions

```jsx
import { useNavigate } from 'react-router-dom';

function ProductCard({ product }) {
  const navigate = useNavigate();

  function handleClick() {
    if (!document.startViewTransition) {
      navigate(`/product/${product.id}`);
      return;
    }

    document.startViewTransition(() => {
      // React 18 flushSync দিয়ে sync রেন্ডার
      flushSync(() => {
        navigate(`/product/${product.id}`);
      });
    });
  }

  return (
    <div
      className="product-card"
      style={{ viewTransitionName: `product-${product.id}` }} // 🔑 নাম দিন
      onClick={handleClick}
    >
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
    </div>
  );
}

// ProductDetail পেজে একই নাম
function ProductDetail({ product }) {
  return (
    <div>
      <div
        className="product-hero"
        style={{ viewTransitionName: `product-${product.id}` }} // 🔑 একই নাম
      >
        <img src={product.image} alt={product.name} />
      </div>
      <h1>{product.name}</h1>
    </div>
  );
}
```

---

## ✅ CSS দিয়ে কাস্টম অ্যানিমেশন

```css
/* ডিফল্ট: ক্রস-ফেড */
::view-transition-old(root) {
  animation: fade-out 0.3s ease-in;
}

::view-transition-new(root) {
  animation: fade-in 0.3s ease-out;
}

/* স্লাইড ট্রানজিশন */
@keyframes slide-in-right {
  from { transform: translateX(100%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}

@keyframes slide-out-left {
  from { transform: translateX(0);     opacity: 1; }
  to   { transform: translateX(-100%); opacity: 0; }
}

.slide-transition::view-transition-old(root) {
  animation: slide-out-left 0.3s ease-in-out;
}
.slide-transition::view-transition-new(root) {
  animation: slide-in-right 0.3s ease-in-out;
}

/* একক এলিমেন্ট হিরো ট্রানজিশন */
::view-transition-old(product-hero),
::view-transition-new(product-hero) {
  animation-duration: 0.4s;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}
```

---

## ✅ Next.js-এ View Transitions

```jsx
// next.config.js
const nextConfig = {
  experimental: {
    viewTransition: true, // Next.js 15+
  },
};

// page.jsx — App Router
import { unstable_ViewTransition as ViewTransition } from 'react';

export default function ProductList({ products }) {
  return (
    <div>
      {products.map(product => (
        <ViewTransition key={product.id} name={`product-${product.id}`}>
          <ProductCard product={product} />
        </ViewTransition>
      ))}
    </div>
  );
}
```

---

## ✅ SPA-তে দিকনির্দেশনা অনুযায়ী অ্যানিমেশন

```javascript
// নেভিগেশনের দিক বুঝে অ্যানিমেশন বদলানো
let isForward = true;

function navigate(url, direction = 'forward') {
  isForward = direction === 'forward';
  document.documentElement.dataset.direction = direction;

  document.startViewTransition(() => updateDOM(url));
}
```

```css
/* দিক অনুযায়ী আলাদা অ্যানিমেশন */
[data-direction="forward"]  ::view-transition-old(root) {
  animation: slide-out-left 0.3s ease;
}
[data-direction="forward"]  ::view-transition-new(root) {
  animation: slide-in-right 0.3s ease;
}
[data-direction="backward"] ::view-transition-old(root) {
  animation: slide-out-right 0.3s ease;
}
[data-direction="backward"] ::view-transition-new(root) {
  animation: slide-in-left 0.3s ease;
}
```

---

## ✅ ব্রাউজার সাপোর্ট চেক ও Fallback

```javascript
// Progressive Enhancement
function withTransition(callback) {
  if (typeof document.startViewTransition === 'function') {
    document.startViewTransition(callback);
  } else {
    callback(); // সরাসরি DOM পরিবর্তন
  }
}

// বা
const supportsVT = 'startViewTransition' in document;
```

```css
/* motion preference respect করুন */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

---

## 🔄 সম্পর্কিত প্যাটার্ন

| প্যাটার্ন | সম্পর্ক |
|----------|---------|
| Islands Architecture | Islands re-hydrate হওয়ার সময় VT ব্যবহার করা যায় |
| Route-Based Splitting | রুট পরিবর্তনে VT smooth transition দেয় |
| Import on Interaction | ক্লিকে নতুন কমপোনেন্ট লোড + VT দিয়ে smooth |

---

## ✅ সুবিধা (Pros)

- **নেটিভ পারফরম্যান্স** — ব্রাউজার GPU-accelerated
- **কম কোড** — আলাদা animation library নেই
- **UX উন্নতি** — native app-এর মতো অনুভব
- **Progressive Enhancement** — পুরনো ব্রাউজারে fallback

## ❌ অসুবিধা (Cons)

- **ব্রাউজার সাপোর্ট** — Chrome 111+, Safari 18+; Firefox-এ এখনো নেই
- **React integration** — flushSync লাগে, জটিল হতে পারে
- **Debug** — DevTools-এ পুরো সাপোর্ট সীমিত

---

## 📊 ব্রাউজার সাপোর্ট (2025)

| ব্রাউজার | সাপোর্ট |
|---------|--------|
| Chrome/Edge 111+ | ✅ Same-Document VT |
| Chrome 123+ | ✅ Cross-Document VT |
| Safari 18+ | ✅ Same-Document VT |
| Firefox | ❌ এখনো নেই |

---

## 📝 কখন ব্যবহার করবেন

- ✅ SPA রুট ট্রানজিশনে
- ✅ পণ্য তালিকা → বিস্তারিত পেজ (shared element)
- ✅ মোবাইল-ফার্স্ট অ্যাপে native feel
- ❌ যেখানে Firefox সাপোর্ট জরুরি (fallback পরিকল্পনা রাখুন)
- ❌ `prefers-reduced-motion` ইউজারদের জন্য সবসময় fallback রাখুন
