# 🏗️ Micro-Frontends — বড় ফ্রন্টএন্ডকে ভাগ করে চালানো

## 📌 সংজ্ঞা

**Micro-Frontends (MFE)** হলো micro-services-এর frontend version। এক বিশাল Single
Page Application (monolith) কে **স্বাধীন, ছোট, deployable অ্যাপ্লিকেশনে** ভাগ করে দেওয়া
— যেখানে প্রতিটি অংশ আলাদা team, আলাদা codebase, এমনকি আলাদা framework দিয়ে
বানানো যায়।

```
  Monolith Frontend                 vs       Micro-Frontends
  ──────────────────────                     ─────────────────────
  ┌────────────────────────┐                 ┌─────┬─────┬─────┐
  │   এক বিশাল React App   │                 │ MFE │ MFE │ MFE │
  │  ─────────────────────  │                 │  A  │  B  │  C  │
  │  500K+ lines of code    │                 ├─────┴─────┴─────┤
  │  ১০টি team blocked      │                 │ Container Shell │
  │  ১বার deploy = সব এক‌সাথে│                 └─────────────────┘
  └────────────────────────┘                 আলাদা deploy, আলাদা stack
```

---

## 🎯 কেন দরকার?

বাংলাদেশের উদাহরণ — **Daraz Bangladesh**:

```
  Daraz-এর ভিতরে আলাদা team:
  ─────────────────────────────
  🛒 Catalog team       → product list, search, filter
  🏠 Home team          → homepage, banner, recommendations
  💳 Checkout team      → cart, payment, address
  👤 Account team       → profile, orders, returns
  🎮 Gamification team  → DarazMall, flash sale, coupons
  📦 Seller team        → seller dashboard
  💬 Chat team          → live chat with seller

  Monolith হলে:
  ❌ Catalog team-এর কোড deploy করতে hold-up করছে Checkout team
  ❌ এক bug সবার merge block করে
  ❌ ১০ মিনিট build time
  ❌ React 16 → 18 migration = সব team একসাথে আটকে যায়
  ❌ Performance: এক page-এর জন্য পুরো ১০MB JS load

  Micro-Frontend হলে:
  ✅ প্রতিটি team নিজের repo, নিজের deploy
  ✅ Catalog team React 18, Seller team Vue 3 (gradual migrate)
  ✅ এক bug = এক MFE ফেরত যায়, বাকি অক্ষত
  ✅ Lazy load — ইউজার শুধু যা দরকার তাই download করে
```

---

## 🧬 ৪টি প্রধান ইন্টিগ্রেশন প্যাটার্ন

```
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │   ১. Build-time integration                                   │
  │      npm package হিসেবে publish করে container-এ import        │
  │                                                                │
  │   ২. Run-time via iframe                                      │
  │      প্রতিটি MFE আলাদা iframe                                  │
  │                                                                │
  │   ৩. Run-time via JavaScript                                  │
  │      Module Federation, Single-SPA, import-maps               │
  │                                                                │
  │   ৪. Run-time via Web Components                              │
  │      প্রতিটি MFE একটি custom element                          │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

### তুলনা টেবিল

| বৈশিষ্ট্য | Build-time | iframe | Module Federation | Web Components |
|----------|-----------|--------|-------------------|----------------|
| Independent deploy | ❌ | ✅ | ✅ | ✅ |
| Different framework | ❌ | ✅ | ⚠️ same major | ✅ |
| Shared dependencies | N/A (single bundle) | ❌ duplicated | ✅ shared | ⚠️ manual |
| CSS isolation | ❌ | ✅ | ❌ | ✅ Shadow DOM |
| SEO friendly | ✅ | ❌ | ✅ | ✅ |
| Performance | best | worst | good | good |
| Communication | function call | postMessage | Pub-Sub / events | Custom events |
| জটিলতা | কম | কম | বেশি | মাঝারি |

---

## 1️⃣ Build-Time Integration

প্রতিটি MFE-কে npm package হিসেবে publish করা হয়, container সেটি import করে।

```bash
@daraz/header@2.4.1
@daraz/catalog@1.8.0
@daraz/checkout@3.1.5
```

```jsx
// container-app/src/App.jsx
import { Header } from '@daraz/header';
import { Catalog } from '@daraz/catalog';
import { Checkout } from '@daraz/checkout';

function App() {
  return (
    <>
      <Header />
      <Routes>
        <Route path="/" element={<Catalog />} />
        <Route path="/checkout" element={<Checkout />} />
      </Routes>
    </>
  );
}
```

**সমস্যা:** এক MFE update করতে হলে container redeploy লাগবে। এটা **আসলে micro-frontend
না** — শুধু modular monolith।

---

## 2️⃣ iframe-based

```html
<!-- container -->
<iframe src="https://checkout.daraz.com.bd" sandbox="allow-scripts allow-forms"></iframe>
```

**Pro:** সম্পূর্ণ isolation (CSS, JS, security)। legacy অ্যাপ embed করার সবচেয়ে সহজ।

**Con:**
- Performance বাজে (প্রতিটা iframe full browser context)
- SEO খারাপ
- responsive তৈরি কঠিন
- communication শুধু `postMessage`
- back/forward button তালগোল পাকায়

```javascript
// container ↔ iframe communication
const iframe = document.querySelector('iframe');
iframe.contentWindow.postMessage(
  { type: 'CART_UPDATED', items: [...] },
  'https://checkout.daraz.com.bd'
);

window.addEventListener('message', (e) => {
  if (e.origin !== 'https://checkout.daraz.com.bd') return;
  if (e.data.type === 'PAYMENT_SUCCESS') {
    // ...
  }
});
```

> 💡 আজকাল iframe ব্যবহার হয় শুধু special case-এ: payment widget (bKash/SSLCommerz),
> ad iframe, embeddable preview।

---

## 3️⃣ Module Federation (Webpack 5)

Module Federation হলো **runtime code sharing**। এক অ্যাপ অন্য অ্যাপের module live load
করতে পারে — code-splitting-এর মতো, কিন্তু cross-application।

### আর্কিটেকচার

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   Container (Shell — Host)                                 │
  │   ┌────────────────────────────────────────────────────┐   │
  │   │ remoteEntry: catalog (https://catalog.daraz.com)  │   │
  │   │ remoteEntry: checkout (https://checkout.daraz.com)│   │
  │   │                                                    │   │
  │   │  Routing:                                          │   │
  │   │   /          → load catalog/HomePage              │   │
  │   │   /checkout  → load checkout/CheckoutPage         │   │
  │   └────────────────────────────────────────────────────┘   │
  │                                                             │
  │   Each remote serves /remoteEntry.js                       │
  │   ↓ runtime fetch                                          │
  │   ↓ runtime evaluate                                       │
  │   ↓ component mount                                        │
  └─────────────────────────────────────────────────────────────┘
```

### Container (Host) webpack.config.js

```javascript
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  // ...
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        catalog:  'catalog@https://catalog.daraz.com.bd/remoteEntry.js',
        checkout: 'checkout@https://checkout.daraz.com.bd/remoteEntry.js',
        seller:   'seller@https://seller.daraz.com.bd/remoteEntry.js'
      },
      shared: {
        react:        { singleton: true, requiredVersion: '^18.2.0' },
        'react-dom':  { singleton: true, requiredVersion: '^18.2.0' },
        'react-router-dom': { singleton: true }
      }
    })
  ]
};
```

### Remote (Catalog) webpack.config.js

```javascript
new ModuleFederationPlugin({
  name: 'catalog',
  filename: 'remoteEntry.js',
  exposes: {
    './HomePage':       './src/pages/Home.jsx',
    './ProductDetail':  './src/pages/Product.jsx',
    './SearchWidget':   './src/components/Search.jsx'
  },
  shared: {
    react:       { singleton: true, requiredVersion: '^18.2.0' },
    'react-dom': { singleton: true, requiredVersion: '^18.2.0' }
  }
});
```

### Container-এ ব্যবহার

```jsx
import { lazy, Suspense } from 'react';

const HomePage    = lazy(() => import('catalog/HomePage'));
const Checkout    = lazy(() => import('checkout/CheckoutPage'));
const SellerPanel = lazy(() => import('seller/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/"          element={<HomePage />} />
        <Route path="/checkout"  element={<Checkout />} />
        <Route path="/seller/*"  element={<SellerPanel />} />
      </Routes>
    </Suspense>
  );
}
```

### TypeScript declaration

```typescript
// types/remotes.d.ts
declare module 'catalog/HomePage' {
  const HomePage: React.ComponentType;
  export default HomePage;
}
```

### Shared dependencies-এর গুরুত্ব

`shared.singleton: true` — একই React instance সবাই use করবে। নাহলে দুইটা React চালালে
hooks ভেঙে যাবে ("Invalid hook call")।

`requiredVersion` mismatch হলে Webpack warning দিয়ে দুটোই load করে — bundle size বেড়ে
যাবে। তাই version policy থাকা জরুরি।

---

## 4️⃣ Single-SPA

framework-agnostic micro-frontend orchestrator। React, Vue, Angular একসাথে চালাতে
পারে।

```javascript
// root-config.js
import { registerApplication, start } from 'single-spa';

registerApplication({
  name: '@daraz/header',
  app: () => System.import('@daraz/header'),
  activeWhen: () => true   // সব route-এ
});

registerApplication({
  name: '@daraz/catalog',
  app: () => System.import('@daraz/catalog'),
  activeWhen: ['/products', '/search']
});

registerApplication({
  name: '@daraz/seller',
  app: () => System.import('@daraz/seller'),
  activeWhen: '/seller'
});

start();
```

### Lifecycle

প্রতিটি single-spa app ৩টি function export করে:

```javascript
// catalog/main.js
let reactRoot;

export async function bootstrap(props) {
  // একবার call হয় — initialization
}

export async function mount(props) {
  // route active হলে render
  reactRoot = ReactDOM.createRoot(document.getElementById('catalog-root'));
  reactRoot.render(<App {...props} />);
}

export async function unmount(props) {
  // route ছেড়ে গেলে cleanup
  reactRoot.unmount();
}
```

### import-map

```html
<script type="systemjs-importmap">
{
  "imports": {
    "@daraz/header":   "https://cdn.daraz.com.bd/header/v2.4.1.js",
    "@daraz/catalog":  "https://cdn.daraz.com.bd/catalog/v1.8.0.js",
    "@daraz/seller":   "https://cdn.daraz.com.bd/seller/v3.1.0.js"
  }
}
</script>
```

import-map স্বাধীনভাবে update করে individual MFE deploy — container redeploy লাগে না।

---

## 5️⃣ Web Components-based

প্রতিটি MFE একটি custom element হিসেবে publish হয়।

```html
<header-mfe></header-mfe>
<router-outlet>
  <!-- route অনুযায়ী -->
  <catalog-mfe></catalog-mfe>
  <checkout-mfe></checkout-mfe>
</router-outlet>
<footer-mfe></footer-mfe>
```

**সুবিধা:** framework-agnostic, Shadow DOM-এ CSS isolation, browser-native।

**বিস্তারিত: `web-components.md` দেখুন।**

---

## 🛣️ Routing — কে responsible?

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   ১. Container routes everything (top-level routing)    │
  │      - shell-এ react-router                              │
  │      - URL দেখে কোন MFE mount করবে decide করে            │
  │      - সবচেয়ে কমন                                       │
  │                                                          │
  │   ২. Per-MFE sub-routing                                │
  │      - shell /seller/* MFE-কে দেয়                       │
  │      - MFE নিজে /seller/dashboard, /seller/products      │
  │        manage করে                                        │
  │                                                          │
  │   ৩. Hybrid                                              │
  │      - shell parent routes                               │
  │      - MFE child routes                                  │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

```jsx
// container
<Routes>
  <Route path="/"            element={<Home />} />
  <Route path="/seller/*"    element={<SellerMFE />} />  {/* * = sub-routes */}
  <Route path="/checkout/*"  element={<CheckoutMFE />} />
</Routes>
```

---

## 🔄 Inter-MFE Communication

### Pattern ১: Custom Events (recommended)

```javascript
// catalog MFE — cart-এ যোগ
window.dispatchEvent(new CustomEvent('daraz:cart:add', {
  detail: { productId: 123, qty: 2 }
}));

// header MFE — listen
window.addEventListener('daraz:cart:add', (e) => {
  this.cartCount += e.detail.qty;
});
```

### Pattern ২: Shared State (Pub-Sub)

```javascript
// container exposes
window.darazStore = createStore({
  cart: [],
  user: null
});

// MFE subscribes
window.darazStore.subscribe(state => {
  this.cartCount = state.cart.length;
});
```

⚠️ Tight coupling-এ পরিণত হলে micro-frontend-এর সুফল চলে যায়।

### Pattern ৩: URL state

URL parameter / query string ব্যবহার করে state pass — সবচেয়ে decoupled।

```
  /products?category=mobile&seller=123
  → catalog MFE URL থেকে read করে
  → অন্য MFE-এর সাথে কথা বলতে হয় না
```

---

## 🎨 Shared Design System

প্রতিটি MFE একই button, modal, color use করতে হবে — নাহলে UI ভঙ্গুর দেখাবে।

```
  ┌────────────────────────────────────────────────┐
  │      @daraz/design-system (npm package)        │
  │   ─────────────────────────────                │
  │   • Web Components: <daraz-button>             │
  │   • CSS tokens:    --daraz-color-primary       │
  │   • Icons:          SVG sprite                 │
  │                                                │
  │   প্রতিটি MFE এই dependency use করে            │
  │   ─────────────────────────────────             │
  │   shared in Module Federation                   │
  └────────────────────────────────────────────────┘
```

Design tokens (CSS custom properties) container-এ inject করুন — সব MFE নিজের mode (dark,
RTL, locale) inherit করবে।

---

## 🔐 Authentication

JWT token shell-এ save → cookie / localStorage → MFE-রা read করে।

```javascript
// shell
const token = await login(...);
localStorage.setItem('daraz_token', token);

// MFE-এর fetch wrapper
const fetchAPI = (url, opts={}) => {
  return fetch(url, {
    ...opts,
    headers: {
      ...opts.headers,
      Authorization: `Bearer ${localStorage.getItem('daraz_token')}`
    }
  });
};
```

ভালো practice: shell **`AuthProvider`** context Module Federation দিয়ে expose করে।

---

## 🇧🇩 Daraz Multi-Team উদাহরণ — Module Federation

```
  daraz.com.bd-এর architecture
  ════════════════════════════════════════════════════════════════

  https://www.daraz.com.bd/
  │
  ├─ shell.js           (container, ১৫০KB)
  ├─ /home/             ── catalog-team    React 18
  ├─ /products/:id      ── catalog-team    React 18
  ├─ /checkout          ── checkout-team   React 18
  ├─ /account           ── account-team    Vue 3 (legacy migration)
  ├─ /seller            ── seller-team     React 17 (in upgrade)
  └─ /campaign/11-11    ── marketing-team  vanilla JS

  প্রতিটি team:
  • আলাদা Git repo
  • আলাদা CI/CD
  • আলাদা S3+CloudFront/CDN
  • আলাদা release cadence (catalog daily, seller weekly)
  • shared: design system, auth, analytics
```

### Daraz-এর শেয়ার্ড infrastructure

```javascript
// shell webpack
new ModuleFederationPlugin({
  name: 'daraz-shell',
  remotes: {
    catalog:    'catalog@https://catalog-cdn.daraz.com.bd/[buildId]/remoteEntry.js',
    checkout:   'checkout@https://checkout-cdn.daraz.com.bd/[buildId]/remoteEntry.js',
    account:    'account@https://account-cdn.daraz.com.bd/[buildId]/remoteEntry.js'
  },
  shared: {
    react:                   { singleton: true, eager: false },
    'react-dom':             { singleton: true },
    '@daraz/design-system':  { singleton: true },
    '@daraz/auth-sdk':       { singleton: true },
    '@daraz/analytics':      { singleton: true }
  }
});
```

### Performance considerations

```
  বিপদ: প্রতিটি MFE আলাদা React চালালে
  ─────────────────────────────────────
  ❌ ৩টি MFE × React 50KB = ১৫০KB extra
  ❌ Hooks ভেঙে যাবে (different React instance)

  সমাধান: shared singleton + version policy
  ─────────────────────────────────────────
  ✅ এক React instance, সবাই use করে
  ✅ requiredVersion: '^18.2.0' policy
  ✅ Quarterly upgrade plan
```

---

## ⚖️ Pros & Cons

### ✅ Pros

```
  • Independent deployments → release velocity বাড়ে
  • Team autonomy → "two-pizza team" scale
  • Tech diversity → gradual migration possible
  • Smaller bundles per route
  • Fault isolation → এক MFE crash, বাকি অক্ষত
  • Codebase manageable → এক team এক mental model
```

### ❌ Cons

```
  • Operational complexity (CI/CD × N)
  • Bundle duplication risk
  • Shared state difficult
  • Cross-MFE refactor কঠিন
  • Debugging harder (multiple sources)
  • Performance overhead (network, glue code)
  • Design consistency বজায় রাখা challenging
  • Version management hell (different React versions)
```

---

## ✅ যেখানে ব্যবহার করুন

- 🏢 **বড় organization** — ১০০+ frontend developer (Daraz, Pathao, bKash মাপের)
- 🚀 **Independent release cycles** চাই
- 🧬 **Tech migration** চলছে (Angular → React, Vue 2 → Vue 3)
- 🎯 **Distinct business domains** যেগুলো খুব কম inter-depend
- 🛍️ **Plugin architecture** (Shopify-এর মতো 3rd-party widget)

## ❌ যেখানে এড়ান

- ছোট অ্যাপ (< ১০ developer)
- Single team
- Tightly coupled UI (nested data, shared real-time state)
- Performance-critical app (latency-sensitive)
- Team-এর DevOps maturity কম

---

## ⚠️ Pitfalls

```
  ❌ Distributed monolith — সব MFE coupled, একটা update করলে বাকি ভাঙে
  ❌ Different React/Vue version — runtime conflict
  ❌ Style leak — global CSS এক MFE-র another-কে ভাঙে
  ❌ Auth token mismatch — re-login loop
  ❌ Shared state via window globals — race condition
  ❌ Different routing libraries — back button ভাঙে
  ❌ MFE cache busting ভুল — পুরনো version load
  ❌ Bundle duplication — ৩x React-DOM
  ❌ E2E test cross-MFE setup কঠিন
  ❌ Design drift — ৬ মাস পর প্রতিটি MFE আলাদা দেখায়
```

---

## 📋 সারসংক্ষেপ

```
  Micro-Frontend ৪টি integration pattern
  ════════════════════════════════════════════════
  ① Build-time         → npm package, redeploy needed (নাম এ MFE হলেও আসলে monolith)
  ② iframe             → maximum isolation, worst UX
  ③ Module Federation  → modern Webpack 5, shared deps
  ④ Web Components     → framework-agnostic, native CSS isolation
  + Single-SPA           → orchestrator (multi-framework)

  মূল্য:
  • Team scale > technical purity
  • Independent deploy > performance
  • Operational complexity বাড়ে — DevOps maturity লাগবে
  • Design system + shared infra আগে set করুন
```

Micro-frontend একটা **organizational pattern**, technical না। যখন আপনার অ্যাপ এত বড়
যে এক team এর সব দেখা সম্ভব না, যখন release frequency বাড়াতে হবে, যখন ৫০+ developer-এর
git merge conflict-এ অর্ধেক সময় যায় — তখনই MFE adopt করুন। ছোট team-এ এটা
**over-engineering**।
