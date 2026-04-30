# 🎨 ফ্রন্টএন্ড ইঞ্জিনিয়ারিং (Frontend Engineering)

> ব্রাউজার রেন্ডারিং থেকে শুরু করে স্টেট ম্যানেজমেন্ট, বান্ডলিং, কম্পোনেন্ট ডিজাইন, অ্যাকসেসিবিলিটি, PWA, মাইক্রো-ফ্রন্টএন্ড, WebAssembly এবং সিকিউরিটি পর্যন্ত — আধুনিক ফ্রন্টএন্ড ইঞ্জিনিয়ারিং এর গভীর বিশ্লেষণ।
>
> প্রতিটি টপিকে JavaScript/TypeScript কোড, যেখানে প্রযোজ্য PHP backend ইন্টিগ্রেশন, ASCII ডায়াগ্রাম এবং বাংলাদেশী রিয়েল-ওয়ার্ল্ড উদাহরণ (Daraz, Pathao, bKash, Foodpanda, Chaldal, ShareTrip ইত্যাদি)।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | ব্রাউজার রেন্ডারিং পাইপলাইন | [browser-rendering-pipeline.md](./browser-rendering-pipeline.md) | DOM, CSSOM, Render Tree, Layout, Paint, Composite, Reflow vs Repaint, GPU layers |
| ২ | স্টেট ম্যানেজমেন্ট | [state-management.md](./state-management.md) | Server/Client/URL/Form state, Redux, Zustand, Recoil, Jotai, MobX, React Query |
| ৩ | মডিউল বান্ডলার | [module-bundlers.md](./module-bundlers.md) | Webpack, Vite, Rollup, esbuild, Turbopack, Tree shaking, HMR, Module Federation |
| ৪ | CSS আর্কিটেকচার | [css-architecture.md](./css-architecture.md) | BEM, OOCSS, SMACSS, ITCSS, CSS Modules, CSS-in-JS, Tailwind, Design Tokens, @layer |
| ৫ | কম্পোনেন্ট ডিজাইন | [component-design.md](./component-design.md) | Atomic Design, Compound Components, Headless, Render Props, Controlled vs Uncontrolled |
| ৬ | TypeScript অ্যাডভান্সড | [typescript-advanced.md](./typescript-advanced.md) | Generics, Conditional/Mapped Types, infer, Discriminated Unions, Branded Types |
| ৭ | অ্যাকসেসিবিলিটি (a11y) | [accessibility.md](./accessibility.md) | WCAG, ARIA, Keyboard Navigation, Screen Readers, Focus Management |
| ৮ | PWA ও Service Workers | [pwa-service-workers.md](./pwa-service-workers.md) | Service Worker lifecycle, Caching strategies, Web App Manifest, Push Notifications |
| ৯ | Web Components | [web-components.md](./web-components.md) | Custom Elements, Shadow DOM, HTML Templates, Slots, Lit |
| ১০ | মাইক্রো-ফ্রন্টএন্ড | [micro-frontends.md](./micro-frontends.md) | Module Federation, Single-SPA, iframe, Build-time vs Runtime Integration |
| ১১ | Web Workers | [web-workers.md](./web-workers.md) | Dedicated/Shared/Service Workers, postMessage, SharedArrayBuffer, Comlink |
| ১২ | WebAssembly | [webassembly.md](./webassembly.md) | Wasm fundamentals, Rust/C++ → Wasm, JS interop, use cases |
| ১৩ | ব্রাউজার স্টোরেজ | [browser-storage.md](./browser-storage.md) | LocalStorage, SessionStorage, IndexedDB, Cookies, Cache API |
| ১৪ | ফ্রন্টএন্ড সিকিউরিটি | [frontend-security.md](./frontend-security.md) | XSS, CSRF, CSP, SRI, CORS, Clickjacking, Trusted Types |
| ১৫ | SEO ও SPA | [seo-spa.md](./seo-spa.md) | SSR, SSG, Meta tags, Structured Data, Open Graph, Sitemap |
| ১৬ | i18n ও l10n | [i18n-l10n.md](./i18n-l10n.md) | Translation, RTL, Date/Number/Currency formatting, Bangla/English Switching |

> 📝 **নোট**: এই সেকশনের ১-৭ ফাইলগুলো (Part A: Core) প্রথম ধাপে তৈরি হয়েছে। ৮-১৬ ফাইলগুলো Part B এ আসবে।

---

## 🗺️ টপিক সম্পর্ক ডায়াগ্রাম

```
                ┌──────────────────────────────────────────┐
                │      ফ্রন্টএন্ড ইঞ্জিনিয়ারিং               │
                │   (ব্রাউজার + ফ্রেমওয়ার্ক + টুলিং)         │
                └────────────────────┬─────────────────────┘
                                     │
        ┌────────────┬───────────────┼───────────────┬────────────┐
        ▼            ▼               ▼               ▼            ▼
   ┌─────────┐  ┌─────────┐    ┌──────────┐   ┌─────────┐  ┌──────────┐
   │ব্রাউজার  │  │ ভাষা ও  │    │ আর্কি-   │   │  অ্যাড.  │  │  ক্রস-   │
   │ ভিত্তি   │  │ টুলিং   │    │ টেকচার   │   │  ফিচার  │  │ কাটিং   │
   └────┬────┘  └────┬────┘    └────┬─────┘   └────┬────┘  └────┬─────┘
        │            │              │              │            │
        ▼            ▼              ▼              ▼            ▼
  ┌──────────┐ ┌──────────┐  ┌──────────┐   ┌──────────┐ ┌──────────┐
  │Rendering │ │TypeScript│  │Component │   │   PWA    │ │Security  │
  │Pipeline  │ │Advanced  │  │  Design  │   │+SW       │ │(XSS/CSRF)│
  ├──────────┤ ├──────────┤  ├──────────┤   ├──────────┤ ├──────────┤
  │ Browser  │ │ Module   │  │  State   │   │ Web      │ │  a11y    │
  │ Storage  │ │ Bundlers │  │  Mgmt    │   │ Workers  │ │ (WCAG)   │
  ├──────────┤ ├──────────┤  ├──────────┤   ├──────────┤ ├──────────┤
  │   Web    │ │   CSS    │  │  Micro-  │   │WebAssem- │ │   SEO    │
  │Components│ │ Archi.   │  │ Frontend │   │  bly     │ │  + SPA   │
  └──────────┘ └──────────┘  └──────────┘   └──────────┘ ├──────────┤
                                                         │  i18n /  │
                                                         │  l10n    │
                                                         └──────────┘
```

---

## 🎯 শেখার পথ (Learning Path)

```
Step 1: Browser Rendering Pipeline → ব্রাউজার ভেতরে কী হয় বুঝুন
         │
Step 2: TypeScript Advanced → টাইপ-সেফ কোডিং আয়ত্ত করুন
         │
Step 3: Module Bundlers → বিল্ড পাইপলাইন শিখুন (Vite/Webpack)
         │
Step 4: CSS Architecture → স্কেলেবল স্টাইল লিখতে শিখুন
         │
Step 5: Component Design → পুনঃব্যবহারযোগ্য কম্পোনেন্ট ডিজাইন
         │
Step 6: State Management → ক্লায়েন্ট/সার্ভার স্টেট ম্যানেজ করুন
         │
Step 7: Accessibility, PWA, Security → প্রোডাকশন-গ্রেড অ্যাপ
         │
Step 8: Advanced (Web Workers, Wasm, Micro-frontend) → স্কেল
```

---

## 🏠 বাংলাদেশী প্রসঙ্গ — কে কী ব্যবহার করে

| কোম্পানি | ব্যবহৃত প্রযুক্তি | প্রাসঙ্গিক টপিক |
|----------|-------------------|------------------|
| **Daraz BD** | React, Webpack, Module Federation, SSR | Bundler, State Mgmt, SEO |
| **Pathao** | React Native + Web (Next.js), Redux | State Mgmt, Component Design |
| **bKash** | Custom Web SDK, CSP, Vault, secure inputs | Frontend Security |
| **Nagad** | Hybrid app, WebView optimization | Rendering Pipeline, Performance |
| **Foodpanda** | Next.js, Atomic Design, Tailwind, Service Worker | PWA, CSS Architecture |
| **Chaldal** | SPA + offline-first cart (IndexedDB) | Browser Storage, PWA |
| **ShareTrip** | React, i18n (Bangla/English), SEO-heavy | i18n, SEO + SPA |
| **Pickaboo** | E-commerce SPA, image optimization, lazy load | Rendering, Bundlers |
| **Prothom Alo** | SSR + SSG (news), Open Graph, AMP | SEO, SSR |
| **Robi/GP MyApp** | PWA, push notifications, low-data mode | PWA, Performance |

---

## 📊 ফ্রেমওয়ার্ক/টুল তুলনা সারসংক্ষেপ

```
┌────────────────┬──────────┬──────────┬──────────┬──────────┐
│   বৈশিষ্ট্য    │  React   │   Vue    │  Svelte  │  Solid   │
├────────────────┼──────────┼──────────┼──────────┼──────────┤
│ Virtual DOM    │    ✅    │    ✅    │    ❌    │    ❌    │
│ Reactivity     │  Manual  │  Auto    │  Auto    │  Auto    │
│ Bundle Size    │  Medium  │  Medium  │  Small   │  Small   │
│ Learning       │  Medium  │   Easy   │   Easy   │  Medium  │
│ Ecosystem      │   🌟🌟🌟 │   🌟🌟   │    🌟    │    🌟    │
│ TS Support     │    ✅    │    ✅    │    ✅    │    ✅    │
└────────────────┴──────────┴──────────┴──────────┴──────────┘

┌────────────────┬──────────┬──────────┬──────────┬──────────┐
│   Bundler      │ Webpack  │  Vite    │  esbuild │ Turbopack│
├────────────────┼──────────┼──────────┼──────────┼──────────┤
│ Dev Speed      │   Slow   │   Fast   │  Fastest │   Fast   │
│ Prod Output    │  Mature  │   Good   │  Basic   │   New    │
│ HMR            │  Good    │ Excellent│  Basic   │ Excellent│
│ Plugin Ecosys. │   🌟🌟🌟 │   🌟🌟   │    🌟    │    🌟    │
└────────────────┴──────────┴──────────┴──────────┴──────────┘
```

---

> 💡 **টিপ**: প্রতিটি ফাইলে গভীর ব্যাখ্যা, JS/TS কোড, ASCII ডায়াগ্রাম এবং বাংলাদেশী কনটেক্সট রয়েছে।
> নতুন হলে `browser-rendering-pipeline.md` দিয়ে শুরু করুন — পুরো ফ্রন্টএন্ডের ভিত্তি ওখানে।
