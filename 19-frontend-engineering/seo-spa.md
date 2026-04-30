# 🔍 SEO for SPA — Single-Page App ও Search Engine

## 📌 সমস্যা

Traditional সাইট server-side HTML পাঠায় — Googlebot crawl করে সাথে সাথে content পায়।
SPA (Single-Page App) JS execute হওয়ার পর content render হয় — Googlebot এর কাছে empty
HTML ছাড়া কিছু থাকে না।

```
  Traditional SSR সাইট                SPA (CSR)
  ────────────────────                ──────────
  
  Googlebot:                          Googlebot:
  GET /products/iphone                GET /products/iphone
  ↓                                   ↓
  <html>                              <html>
   <h1>iPhone 15 — ১,২৫,০০০৳</h1>     <body>
   <p>...</p>                          <div id="root"></div>
  </html>                             </body>
  
  ✅ "iPhone 15 ১,২৫,০০০৳"            ❌ "" (খালি)
     index হবে                          কিছু index হবে না
```

বাংলাদেশের context-এ SEO crucial:
- **Daraz**: ৬০%+ traffic Google থেকে
- **Bikroy**: classified — keyword-rich
- **প্রথম আলো, BD News24**: news SEO = traffic survival

---

## 🤖 কীভাবে Googlebot JS handle করে

```
  Step ১: HTML crawl
  ──────────────────
  Googlebot HTTP request পাঠায় → response নেয়
  HTML ও link extract করে → URL queue-এ যোগ
  
  Step ২: Render queue
  ──────────────────
  JS-needed পেজ render queue-এ যায়
  Render করতে hours থেকে weeks লাগতে পারে
  
  Step ৩: Headless Chrome
  ──────────────────────
  Web Rendering Service (WRS) — modern Chromium
  JS execute, DOM finalize, content extract
  
  Step ৪: Index
  ─────────────
  Final rendered HTML index হয়
```

> ⚠️ Googlebot এর rendering delay-এর কারণে time-sensitive content (news, product price)
> CSR-এ index হতে দেরি হয়। Bing, Yandex, DuckDuckGo, Baidu আরো দুর্বল JS handle করে।
> তাই SEO চাইলে SPA-এ pre-rendering / SSR / SSG আবশ্যক।

---

## 🛠️ Rendering Strategies (SEO perspective)

বিস্তারিত: `07-architectural-patterns/rendering-patterns.md`। SEO পক্ষে summary:

```
  ┌──────────────────────────────────────────────────────────┐
  │  Strategy           SEO     TTFB    Personalization      │
  │  ────────           ───     ────    ───────────────      │
  │  CSR (pure)         ❌       ✅      ✅                  │
  │  SSR                ✅       ❌      ✅                  │
  │  SSG                ✅✅     ✅✅    ❌                  │
  │  ISR                ✅       ✅✅    Partial             │
  │  Pre-rendering      ✅       ✅      ❌                  │
  │  (only for bots)                                         │
  └──────────────────────────────────────────────────────────┘
```

### Pre-rendering (Prerender.io, Rendertron)

User → CSR; Bot → pre-rendered HTML। User-Agent ভিত্তিক switching।

```nginx
# Nginx config
location / {
  if ($http_user_agent ~* "googlebot|bingbot|yandex|baiduspider") {
    proxy_pass http://prerender-service;
  }
  try_files $uri /index.html;
}
```

⚠️ **Cloaking accusation এড়ানোর জন্য** content identical থাকতে হবে।

### SSR (Next.js, Nuxt, Remix, SvelteKit)

```jsx
// Next.js — pages/products/[id].js
export async function getServerSideProps({ params }) {
  const product = await fetchProduct(params.id);
  return { props: { product } };
}

export default function ProductPage({ product }) {
  return (
    <>
      <Head>
        <title>{product.name} — Daraz Bangladesh</title>
        <meta name="description" content={product.summary} />
      </Head>
      <h1>{product.name}</h1>
      <p>৳{product.price}</p>
    </>
  );
}
```

### SSG / ISR (best for product/blog)

```jsx
// Next.js — ISR
export async function getStaticProps({ params }) {
  const product = await fetchProduct(params.id);
  return {
    props: { product },
    revalidate: 60  // ৬০ সেকেন্ড পর re-generate
  };
}

export async function getStaticPaths() {
  // top ১০০০ product pre-generate
  const top = await fetchTopProducts(1000);
  return {
    paths: top.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking'  // অন্যগুলো on-demand SSR
  };
}
```

Vercel, Netlify এই pattern automate করে।

---

## 🏷️ Meta Tags — SEO-র মূল

```html
<head>
  <!-- Title — সবচেয়ে গুরুত্বপূর্ণ tag -->
  <title>iPhone 15 Pro Max ২৫৬GB — দাম ১,৫৫,০০০৳ | Daraz Bangladesh</title>

  <!-- Description — SERP snippet-এ দেখা যায় -->
  <meta name="description" content="iPhone 15 Pro Max 256GB কিনুন Daraz BD থেকে। অরিজিনাল, ১ বছর ওয়ারেন্টি, ফ্রি ডেলিভারি ঢাকায়। দাম ১,৫৫,০০০৳।">

  <!-- Canonical — duplicate content avoid -->
  <link rel="canonical" href="https://daraz.com.bd/iphone-15-pro-max">

  <!-- Robots — indexing rules -->
  <meta name="robots" content="index, follow, max-image-preview:large">

  <!-- Language -->
  <html lang="bn-BD">
  <meta http-equiv="content-language" content="bn-BD">

  <!-- Mobile -->
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- Charset -->
  <meta charset="UTF-8">
</head>
```

### Title best practices

```
  ✅ ৫০-৬০ characters (truncate-এর আগে)
  ✅ Primary keyword প্রথমে
  ✅ Brand শেষে (পরিচিত হলে)
  ✅ প্রতিটি page-এর unique title
  ✅ Bengali keyword (যদি বাংলা audience)
  
  ❌ Generic ("Home", "Product")
  ❌ Keyword stuffing
  ❌ একই title সব page-এ
```

### Description best practices

```
  ✅ ১৫০-১৬০ characters
  ✅ User benefit + call to action
  ✅ Natural language (keyword robotic না)
  ✅ Unique per page
  
  Direct ranking factor না, কিন্তু CTR বাড়ায় — যা ranking-এ effect করে
```

---

## 🔗 Canonical URLs

একই content একাধিক URL-এ থাকতে পারে — canonical দিয়ে "primary" version বলতে হবে।

```
  https://daraz.com.bd/iphone-15
  https://daraz.com.bd/iphone-15?ref=fb
  https://daraz.com.bd/iphone-15?utm_source=email
  https://daraz.com.bd/products/iphone-15
  https://m.daraz.com.bd/iphone-15
  
  সবগুলোতে:
  <link rel="canonical" href="https://daraz.com.bd/iphone-15">
```

ফলে Googlebot duplicate content penalty দেয় না, এবং সব signal এক URL-এ consolidate হয়।

### Self-referential canonical

প্রতিটি page-এর canonical নিজের URL — অন্য কেউ scrape করে republish করলেও Google original
identify করতে পারে।

---

## 🗺️ Sitemap & robots.txt

### sitemap.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://daraz.com.bd/iphone-15</loc>
    <lastmod>2024-09-15</lastmod>
    <changefreq>daily</changefreq>
    <priority>0.9</priority>
    <xhtml:link rel="alternate" hreflang="bn-BD" href="https://daraz.com.bd/iphone-15" />
    <xhtml:link rel="alternate" hreflang="en-BD" href="https://daraz.com.bd/en/iphone-15" />
  </url>
</urlset>
```

বড় সাইটে sitemap index ব্যবহার করুন (per file ৫০K URL limit):

```xml
<sitemapindex>
  <sitemap><loc>https://daraz.com.bd/sitemap-products-1.xml</loc></sitemap>
  <sitemap><loc>https://daraz.com.bd/sitemap-products-2.xml</loc></sitemap>
  <sitemap><loc>https://daraz.com.bd/sitemap-categories.xml</loc></sitemap>
</sitemapindex>
```

### robots.txt

```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/
Disallow: /cart/
Disallow: /checkout/
Disallow: /*?session=

User-agent: Googlebot
Allow: /

User-agent: GPTBot
Disallow: /

Sitemap: https://daraz.com.bd/sitemap.xml
```

> ⚠️ `Disallow` মানে crawl না — কিন্তু URL anyway index হতে পারে যদি external link থাকে।
> Truly hide করতে `<meta name="robots" content="noindex">` use করুন।

---

## 🎴 Open Graph & Twitter Cards

Facebook/WhatsApp/LinkedIn-এ share করলে কেমন দেখাবে — এটা OG tag controls।
Bangladesh-এ Facebook থেকে Daraz/Pathao-এ traffic-এর বড় অংশ আসে।

```html
<!-- Open Graph (Facebook, LinkedIn, WhatsApp) -->
<meta property="og:type" content="product">
<meta property="og:title" content="iPhone 15 Pro Max — ১,৫৫,০০০৳">
<meta property="og:description" content="অরিজিনাল, ১ বছর ওয়ারেন্টি, ফ্রি ডেলিভারি">
<meta property="og:image" content="https://daraz.com.bd/imgs/iphone-15-1200x630.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:url" content="https://daraz.com.bd/iphone-15">
<meta property="og:site_name" content="Daraz Bangladesh">
<meta property="og:locale" content="bn_BD">

<!-- Twitter Cards -->
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@DarazBD">
<meta name="twitter:title" content="iPhone 15 Pro Max — ১,৫৫,০০০৳">
<meta name="twitter:description" content="অরিজিনাল, ১ বছর ওয়ারেন্টি">
<meta name="twitter:image" content="https://daraz.com.bd/imgs/iphone-15-1200x600.jpg">
```

**Image size:** ১২০০×৬৩০px (1.91:1)। Mobile preview-এ proper crop।

---

## 🧬 Structured Data (JSON-LD, schema.org)

Schema.org markup দিয়ে Google rich snippet, knowledge panel, voice search-এর জন্য
structured info পায়।

### Product schema

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org/",
  "@type": "Product",
  "name": "iPhone 15 Pro Max",
  "image": "https://daraz.com.bd/imgs/iphone-15.jpg",
  "description": "অরিজিনাল আইফোন ১৫ প্রো ম্যাক্স, ২৫৬ জিবি",
  "sku": "APL-IP15PM-256",
  "brand": { "@type": "Brand", "name": "Apple" },
  "offers": {
    "@type": "Offer",
    "url": "https://daraz.com.bd/iphone-15-pro-max",
    "priceCurrency": "BDT",
    "price": "155000",
    "availability": "https://schema.org/InStock",
    "priceValidUntil": "2024-12-31",
    "shippingDetails": {
      "@type": "OfferShippingDetails",
      "shippingDestination": { "@type": "DefinedRegion", "addressCountry": "BD" }
    }
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "reviewCount": "237"
  },
  "review": [{
    "@type": "Review",
    "author": { "@type": "Person", "name": "আকাশ আহমেদ" },
    "datePublished": "2024-09-10",
    "reviewBody": "চমৎকার ফোন",
    "reviewRating": { "@type": "Rating", "ratingValue": "5" }
  }]
}
</script>
```

ফলাফল: SERP-এ price, ★rating, stock status সরাসরি দেখা যায় → CTR ২x পর্যন্ত বাড়ে।

### Article schema (BD News, Prothom Alo)

```json
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "headline": "ঢাকায় মেট্রো রেলের নতুন স্টেশন উদ্বোধন",
  "image": ["https://prothomalo.com/img/metro.jpg"],
  "datePublished": "2024-09-15T08:00:00+06:00",
  "dateModified": "2024-09-15T10:30:00+06:00",
  "author": { "@type": "Person", "name": "প্রতিবেদক" },
  "publisher": {
    "@type": "Organization",
    "name": "প্রথম আলো",
    "logo": { "@type": "ImageObject", "url": "https://prothomalo.com/logo.png" }
  }
}
```

### Breadcrumb

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Home", "item": "https://daraz.com.bd" },
    { "@type": "ListItem", "position": 2, "name": "Mobile", "item": "https://daraz.com.bd/mobile" },
    { "@type": "ListItem", "position": 3, "name": "iPhone 15" }
  ]
}
```

### LocalBusiness (Pathao Foods restaurant)

```json
{
  "@type": "Restaurant",
  "name": "কাচ্চি ভাই — গুলশান",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "গুলশান-২, রোড ৭৪",
    "addressLocality": "ঢাকা",
    "addressCountry": "BD"
  },
  "geo": { "@type": "GeoCoordinates", "latitude": 23.7942, "longitude": 90.4143 },
  "telephone": "+880-2-...",
  "servesCuisine": "Bangladeshi",
  "priceRange": "৳৳"
}
```

### Validation

- Google Rich Results Test: https://search.google.com/test/rich-results
- Schema.org Validator: https://validator.schema.org

---

## 📊 Core Web Vitals & SEO

Google ২০২১ থেকে Core Web Vitals ranking factor। বিস্তারিত:
`17-performance/frontend-performance.md`।

```
  LCP ≤ 2.5s   →  Largest Contentful Paint
  INP ≤ 200ms  →  Interaction to Next Paint
  CLS ≤ 0.1    →  Cumulative Layout Shift
```

Bangladesh-এর mostly 3G/4G — slow LCP penalize হয় বেশি। ranking signal: real-user data
(CrUX dataset)।

---

## 🌍 hreflang — Internationalization SEO

Daraz BD-এর English ও Bangla version আছে। কোন user-কে কোনটা দেখাবে?

```html
<link rel="alternate" hreflang="bn-BD" href="https://daraz.com.bd/iphone-15">
<link rel="alternate" hreflang="en-BD" href="https://daraz.com.bd/en/iphone-15">
<link rel="alternate" hreflang="x-default" href="https://daraz.com.bd/iphone-15">
```

`x-default` = কোনোটার সাথে match না হলে যেটা।

### Reciprocal links

`bn-BD` page-এ `en-BD` reference থাকতে হবে এবং উল্টোটাও — না হলে Google ignore করে।

### URL structure

```
  ccTLD          → daraz.com.bd, daraz.pk        (best signal)
  Subdomain      → bn.daraz.com, en.daraz.com    (good)
  Subdirectory   → daraz.com/bn/, daraz.com/en/  (most common)
  URL parameter  → daraz.com?lang=bn             (worst, avoid)
```

---

## 🇧🇩 কেস স্টাডি — Daraz Product Page SEO

### ১. URL structure

```
  https://daraz.com.bd/products/apple-iphone-15-pro-max-256gb-blue-i123456789-s987654321.html
                            │              └─product slug─┘                  │      │
                            └─category─                            product id┘  store id
```

URL-এ keyword থাকা = direct ranking signal।

### ২. পেজ structure (SSR)

```html
<!DOCTYPE html>
<html lang="bn-BD">
<head>
  <title>Apple iPhone 15 Pro Max 256GB Blue — Buy Online at ৳155,000 | Daraz BD</title>
  <meta name="description" content="...">
  <link rel="canonical" href="...">
  <link rel="alternate" hreflang="en-BD" href=".../en/...">
  
  <!-- OG / Twitter -->
  <meta property="og:..." />
  
  <!-- Schema -->
  <script type="application/ld+json">{ Product, AggregateRating, Offer }</script>
  <script type="application/ld+json">{ BreadcrumbList }</script>
</head>
<body>
  <nav>... breadcrumb ...</nav>
  
  <h1>Apple iPhone 15 Pro Max 256GB</h1>      <!-- ১টাই h1 -->
  
  <section>
    <h2>Specifications</h2>     <!-- semantic structure -->
  </section>
  
  <section>
    <h2>Reviews (237)</h2>
  </section>
  
  <section>
    <h2>Related Products</h2>
    <a href="/iphone-14">iPhone 14</a>          <!-- internal linking -->
  </section>
</body>
</html>
```

### ৩. Performance optimization

- Hero image — preload, WebP/AVIF
- Above-fold inline CSS
- Defer non-critical JS
- Lazy-load below-fold images
- Edge cache via CDN (CloudFlare)

### ৪. CrUX-এ ranking-এর ROI

Daraz internal A/B: LCP 3.8s → 2.4s করার পর — organic conversion ১৮% বাড়ে।

---

## 🔬 SEO Testing & Monitoring

```
  Google tools (free)
  ─────────────────────
  • Search Console    — index status, CTR, queries
  • PageSpeed Insights — Core Web Vitals
  • Rich Results Test — schema validate
  • Mobile-Friendly Test
  • URL Inspection   — render-as-Google
  
  Third-party
  ─────────────
  • Ahrefs / SEMrush / Moz — keyword research
  • Screaming Frog — crawl audit
  • Sitebulb        — technical SEO
  • Lighthouse SEO  — basic checks
```

### Search Console-এ যা দেখবেন

- **Coverage** — কোন URL index হচ্ছে না, কেন
- **Core Web Vitals** — slow URL group
- **Mobile usability** — clickable element overlap
- **Performance** — কোন query থেকে কত click

---

## ⚠️ Pitfalls

```
  ❌ Pure CSP-এ critical content       → SEO অদৃশ্য
  ❌ Hash routing (/#/products/123)     → অনেক crawler skip
  ❌ Infinite scroll without paginate   → bot scroll করে না
  ❌ JS-only links (button onclick=nav) → bot link follow করে না
  ❌ User-Agent ভিত্তিক content swap   → cloaking penalty
  ❌ noindex robots.txt-এ allow         → চক্কর
  ❌ Same title সব page-এ              → duplicate
  ❌ thin content (< 300 words)         → low quality
  ❌ keyword stuffing                   → penalty
  ❌ broken canonical (অন্য পেজে point) → ranking lost
  ❌ Slow LCP / poor CLS                → ranking drop
  ❌ Bangla content without lang="bn"   → wrong language detection
  ❌ Image without alt                  → image search miss
```

---

## ✅ Checklist

```
  Technical
  ──────────
  □ SSR বা SSG বা ISR (pure CSR এড়িয়ে)
  □ Unique title (< 60 chars) প্রতিটি page
  □ Description (< 160 chars)
  □ Canonical URL
  □ Mobile viewport
  □ HTTPS
  □ XML sitemap submitted
  □ robots.txt valid
  □ ৪০৪ → proper 404 status
  □ Redirect → 301 (not 302)
  
  Content
  ─────────
  □ একটাই H1 per page
  □ Logical heading hierarchy
  □ Image alt text
  □ Internal linking
  □ Breadcrumb (visible + schema)
  □ Bangla content lang="bn-BD"
  
  Schema
  ────────
  □ Product schema (e-commerce)
  □ Article schema (news)
  □ BreadcrumbList
  □ Organization
  □ FAQ (যেখানে applicable)
  
  Performance
  ────────────
  □ LCP ≤ 2.5s
  □ INP ≤ 200ms
  □ CLS ≤ 0.1
  
  i18n
  ─────
  □ hreflang reciprocal
  □ x-default
```

---

## 📋 সারসংক্ষেপ

```
  SPA SEO Survival Kit
  ════════════════════════════════════════════════
  
  ১. SSR/SSG/ISR — pure CSR এড়ান
  ২. Meta tags ঠিক রাখুন (title, desc, canonical, OG)
  ৩. Structured data (JSON-LD) সব page-এ
  ৪. Sitemap + robots.txt
  ৫. Core Web Vitals — performance ranking signal
  ৬. hreflang multi-language
  ৭. Real URL (no #), real <a href>
  ৮. Search Console monitor
```

SPA মানে SEO শেষ — এই myth ভুল। **আধুনিক framework (Next.js, Nuxt, Remix, SvelteKit)
SSR/SSG by default**। সঠিক rendering strategy + meta tags + structured data + Core Web
Vitals-এ Daraz-এর মতো বিশাল SPA-ও Google-এ #1 rank করে। বাংলাদেশে যে কোম্পানির ৬০%
traffic search-এ আসে — তার জন্য এটা ব্যবসায়িক জীবনমরণ।
