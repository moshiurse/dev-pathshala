# 🌐 ওয়েব রেন্ডারিং প্যাটার্নস (Web Rendering Patterns)

## 📌 সংজ্ঞা ও পরিচিতি

**Web Rendering Patterns** হলো সেই কৌশলগুলো যা নির্ধারণ করে আপনার ওয়েব পেজের HTML কখন, কোথায়, এবং কীভাবে তৈরি হবে। আধুনিক ওয়েব ডেভেলপমেন্টে চারটি মূল রেন্ডারিং প্যাটার্ন রয়েছে:

| প্যাটার্ন | পূর্ণরূপ | বাংলায় অর্থ | কোথায় রেন্ডার হয় |
|-----------|----------|-------------|-------------------|
| **CSR** | Client-Side Rendering | ক্লায়েন্ট-সাইড রেন্ডারিং | ব্রাউজারে |
| **SSR** | Server-Side Rendering | সার্ভার-সাইড রেন্ডারিং | সার্ভারে (প্রতি রিকোয়েস্টে) |
| **SSG** | Static Site Generation | স্ট্যাটিক সাইট জেনারেশন | বিল্ড টাইমে |
| **ISR** | Incremental Static Regeneration | ইনক্রিমেন্টাল স্ট্যাটিক রিজেনারেশন | বিল্ড টাইমে + রিভ্যালিডেশন |

### মূল প্রশ্ন: HTML কে তৈরি করে?

```
  প্রশ্ন: আপনার পেজের HTML কে বানায়?

  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │   🖥️ ব্রাউজার (Client)  ──────────────── CSR                    │
  │                                                                  │
  │   🖧 সার্ভার (Server)    ──────────────── SSR (প্রতি রিকোয়েস্ট)  │
  │                                                                  │
  │   🏗️ বিল্ড প্রসেস       ──────────────── SSG (ডিপ্লয়ের আগে)     │
  │                                                                  │
  │   🔄 বিল্ড + রিভ্যালিডেশন ─────────────── ISR (স্মার্ট ক্যাশিং)  │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 🏠 বাস্তব জীবনের উদাহরণ (বাংলাদেশ প্রসঙ্গে)

### 🍽️ রেস্তোরাঁ অ্যানালজি

কল্পনা করুন আপনি **ঢাকার** বিভিন্ন রেস্তোরাঁয় খেতে গেছেন:

```
  🍽️ রেন্ডারিং প্যাটার্ন = খাবার পরিবেশনের ধরন

  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  CSR = বুফে সিস্টেম (Star Kabab, Dhaka)                            │
  │  ─────────────────────────────────────────                          │
  │  খালি প্লেট পান (blank HTML), নিজে খাবার তুলে নেন (JS renders)     │
  │  প্রথমে অপেক্ষা বেশি, কিন্তু পরে দ্রুত সার্ভ নিতে পারেন           │
  │                                                                     │
  │  SSR = ওয়েটার সার্ভিস (Haji Biryani)                               │
  │  ─────────────────────────────────────                              │
  │  অর্ডার দিলে রান্নাঘর থেকে রেডি খাবার আসে (server-rendered HTML)   │
  │  প্রতিবার অপেক্ষা করতে হয়, কিন্তু পুরো খাবার একবারে পান           │
  │                                                                     │
  │  SSG = প্যাকেজড ফুড (প্রাণ/ACI-র প্যাকেট খাবার)                   │
  │  ─────────────────────────────────────────────                      │
  │  আগেই তৈরি করে রাখা (build time), দোকান থেকে তুলে নেন (CDN)       │
  │  সবচেয়ে দ্রুত, কিন্তু কাস্টমাইজ করা কঠিন                         │
  │                                                                     │
  │  ISR = ক্যাটারিং সার্ভিস (বিয়ের খাবার)                            │
  │  ──────────────────────────────────────                             │
  │  আগেই তৈরি, কিন্তু নির্দিষ্ট সময় পর ফ্রেশ ব্যাচ আসে             │
  │  (stale-while-revalidate)                                           │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## 📖 ১. CSR — ক্লায়েন্ট-সাইড রেন্ডারিং (Client-Side Rendering)

### কীভাবে কাজ করে?

CSR-এ সার্ভার একটি **প্রায় খালি HTML** ফাইল পাঠায় যেখানে শুধু একটি `<div id="root">` থাকে। এরপর ব্রাউজার JavaScript ডাউনলোড করে, পার্স করে, এবং **ব্রাউজারেই পুরো পেজ রেন্ডার** করে।

```
  CSR ফ্লো (Client-Side Rendering Flow)

  ব্রাউজার                          সার্ভার                     CDN
  ───────                          ──────                     ───
     │                                │                         │
     │  ① GET /dashboard              │                         │
     │ ─────────────────────────────▶ │                         │
     │                                │                         │
     │  ② খালি HTML + <script> ট্যাগ  │                         │
     │ ◀───────────────────────────── │                         │
     │                                │                         │
     │  ⬜ ব্যবহারকারী সাদা স্ক্রিন দেখে                        │
     │                                │                         │
     │  ③ JS বান্ডেল ডাউনলোড (2-5MB)  │                         │
     │ ─────────────────────────────────────────────────────▶  │
     │ ◀───────────────────────────────────────────────────────│
     │                                │                         │
     │  ④ JS পার্স ও এক্সিকিউট        │                         │
     │  [████████████░░░░] ২-৫ সেকেন্ড │                         │
     │                                │                         │
     │  ⑤ API কল (ডেটা আনতে)          │                         │
     │ ─────────────────────────────▶ │                         │
     │ ◀───────────────────────────── │                         │
     │  { users: [...], stats: {...} } │                         │
     │                                │                         │
     │  ⑥ DOM রেন্ডার — পেজ দৃশ্যমান! │                         │
     │  ✅ ইন্টারঅ্যাক্টিভ             │                         │
     │                                │                         │
```

### সার্ভার যা পাঠায় (প্রায় খালি HTML):

```html
<!-- CSR: সার্ভার থেকে আসা HTML -->
<!DOCTYPE html>
<html lang="bn">
<head>
    <title>bKash Merchant Dashboard</title>
    <link rel="stylesheet" href="/static/css/app.3f2a1b.css">
</head>
<body>
    <!-- শুধু একটি খালি div! কোনো কন্টেন্ট নেই -->
    <div id="root"></div>

    <!-- বিশাল JS বান্ডেল — এটাই সব কাজ করে -->
    <script src="/static/js/vendor.8a3b2c.js"></script>  <!-- 800KB -->
    <script src="/static/js/app.5d4e3f.js"></script>      <!-- 1.2MB -->
</body>
</html>
```

### React CSR উদাহরণ (bKash Dashboard):

```jsx
// App.jsx — CSR React Application
import { useState, useEffect } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function MerchantDashboard() {
    const [transactions, setTransactions] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // ব্রাউজারে এক্সিকিউট হয় — সার্ভারে নয়
        fetch('/api/transactions')
            .then(res => res.json())
            .then(data => {
                setTransactions(data);
                setLoading(false);
            });
    }, []);

    if (loading) return <div className="spinner">লোড হচ্ছে...</div>;

    return (
        <div>
            <h1>bKash মার্চেন্ট ড্যাশবোর্ড</h1>
            <p>মোট লেনদেন: {transactions.length}</p>
            {transactions.map(tx => (
                <div key={tx.id}>
                    {tx.sender} → {tx.receiver}: ৳{tx.amount}
                </div>
            ))}
        </div>
    );
}

function App() {
    return (
        <BrowserRouter>
            <Routes>
                <Route path="/dashboard" element={<MerchantDashboard />} />
                <Route path="/reports" element={<Reports />} />
                <Route path="/settings" element={<Settings />} />
            </Routes>
        </BrowserRouter>
    );
}
```

### ✅ CSR-এর সুবিধা

| সুবিধা | ব্যাখ্যা |
|--------|---------|
| সমৃদ্ধ ইন্টারঅ্যাক্টিভিটি | রিয়েল-টাইম আপডেট, ড্র্যাগ-ড্রপ, অ্যানিমেশন সহজ |
| দ্রুত নেভিগেশন | প্রথম লোডের পর পেজ ট্রানজিশন তাৎক্ষণিক (কোনো ফুল পেজ রিলোড নেই) |
| সার্ভার লোড কম | সার্ভার শুধু API ডেটা দেয়, HTML রেন্ডার করতে হয় না |
| সহজ ডিপ্লয়মেন্ট | স্ট্যাটিক ফাইল যেকোনো CDN-এ হোস্ট করা যায় |
| ভালো অফলাইন সাপোর্ট | Service Worker দিয়ে অফলাইনে কাজ করানো সম্ভব |

### ❌ CSR-এর অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---------|---------|
| খারাপ SEO | সার্চ ইঞ্জিন বট খালি HTML দেখে — কন্টেন্ট ইন্ডেক্স হয় না |
| ধীর প্রথম লোড | বড় JS বান্ডেল ডাউনলোড ও পার্স করতে সময় লাগে |
| সাদা স্ক্রিন সমস্যা | JS লোড না হওয়া পর্যন্ত ব্যবহারকারী কিছুই দেখে না |
| ভারী ক্লায়েন্ট | পুরনো মোবাইলে (বাংলাদেশে সাধারণ) ধীরগতি |
| JS নির্ভরতা | JavaScript বন্ধ/ব্যর্থ হলে পুরো সাইট অচল |

### 🎯 কখন CSR ব্যবহার করবেন

```
  ✅ করবেন                              ❌ করবেন না
  ──────────                            ────────────
  • অ্যাডমিন প্যানেল                     • ব্লগ বা নিউজ সাইট
    (bKash Merchant Panel)                (Prothom Alo)
  • ইন্টারনাল ড্যাশবোর্ড                  • ই-কমার্স প্রোডাক্ট পেজ
    (Pathao Driver App)                   (Daraz Product Listing)
  • রিয়েল-টাইম চ্যাট অ্যাপ               • ল্যান্ডিং পেজ
    (Messenger-এর মতো)                   (Grameenphone Offers)
  • ডেটা ভিজুয়ালাইজেশন টুল              • মার্কেটিং সাইট
  • SaaS অ্যাপ্লিকেশন                   • যেকোনো SEO-নির্ভর পেজ
```

---

## 📖 ২. SSR — সার্ভার-সাইড রেন্ডারিং (Server-Side Rendering)

### কীভাবে কাজ করে?

SSR-এ প্রতিটি রিকোয়েস্টে **সার্ভার পূর্ণাঙ্গ HTML তৈরি করে** ব্রাউজারে পাঠায়। ব্রাউজার তাৎক্ষণিকভাবে পেজ দেখাতে পারে। পরে JavaScript লোড হয়ে পেজকে **ইন্টারঅ্যাক্টিভ** করে তোলে — একে **হাইড্রেশন** বলে।

```
  SSR ফ্লো (Server-Side Rendering Flow)

  ব্রাউজার                          সার্ভার                     ডেটাবেস
  ───────                          ──────                     ────────
     │                                │                          │
     │  ① GET /product/123            │                          │
     │ ─────────────────────────────▶ │                          │
     │                                │                          │
     │                                │  ② ডেটা ফেচ              │
     │                                │ ───────────────────────▶ │
     │                                │ ◀─────────────────────── │
     │                                │  { name: "Samsung A54" } │
     │                                │                          │
     │                                │  ③ HTML তৈরি (রেন্ডার)   │
     │                                │  [████████] ৫০-২০০ms     │
     │                                │                          │
     │  ④ পূর্ণ HTML পাঠানো            │                          │
     │ ◀───────────────────────────── │                          │
     │                                │                          │
     │  ✅ তাৎক্ষণিক কন্টেন্ট দৃশ্যমান │                          │
     │  (কিন্তু এখনো ইন্টারঅ্যাক্টিভ নয়)                        │
     │                                │                          │
     │  ⑤ JS বান্ডেল ডাউনলোড          │                          │
     │ ─────────────────────────────▶ │                          │
     │ ◀───────────────────────────── │                          │
     │                                │                          │
     │  ⑥ হাইড্রেশন সম্পন্ন           │                          │
     │  ✅ ইন্টারঅ্যাক্টিভ!            │                          │
     │                                │                          │
```

### 🔄 হাইড্রেশন (Hydration) কী?

হাইড্রেশন হলো সেই প্রক্রিয়া যেখানে ব্রাউজার সার্ভার-রেন্ডার্ড **স্ট্যাটিক HTML**-এর উপরে JavaScript **ইভেন্ট লিসেনার** ও **স্টেট** যুক্ত করে পেজকে ইন্টারঅ্যাক্টিভ বানায়।

```
  হাইড্রেশন প্রক্রিয়া:

  সার্ভার রেন্ডার্ড HTML          +       JavaScript
  (দেখা যায় কিন্তু ক্লিক             (ইন্টারঅ্যাক্টিভিটি
   করা যায় না)                        যোগ করে)
        │                                    │
        ▼                                    ▼
  ┌──────────────┐                 ┌──────────────────┐
  │  <button>    │                 │  onClick handler │
  │  কিনুন      │    ═══════▶     │  + state         │
  │  </button>   │   হাইড্রেশন     │  + effects       │
  │  (নিষ্ক্রিয়) │                 │  (সক্রিয়!)       │
  └──────────────┘                 └──────────────────┘
```

### PHP / Laravel Blade — ঐতিহ্যবাহী SSR:

Laravel Blade হলো **মূল SSR** — সার্ভারে PHP দিয়ে HTML তৈরি করা:

```php
{{-- resources/views/products/show.blade.php --}}
{{-- Daraz-এর মতো প্রোডাক্ট পেজ (Laravel Blade SSR) --}}

@extends('layouts.app')

@section('title', $product->name . ' - Daraz Bangladesh')

{{-- SEO মেটা ট্যাগ — সার্ভারে রেন্ডার হয় তাই Google পড়তে পারে --}}
@section('meta')
    <meta name="description" content="{{ Str::limit($product->description, 160) }}">
    <meta property="og:title" content="{{ $product->name }}">
    <meta property="og:image" content="{{ $product->image_url }}">
    <meta property="og:price:amount" content="{{ $product->price }}">
@endsection

@section('content')
<div class="product-page">
    <div class="product-gallery">
        @foreach($product->images as $image)
            <img src="{{ $image->url }}" alt="{{ $product->name }}" loading="lazy">
        @endforeach
    </div>

    <div class="product-info">
        <h1>{{ $product->name }}</h1>
        <span class="price">৳{{ number_format($product->price) }}</span>

        @if($product->discount_percent > 0)
            <span class="original-price">৳{{ number_format($product->original_price) }}</span>
            <span class="discount">-{{ $product->discount_percent }}%</span>
        @endif

        <div class="rating">
            ⭐ {{ $product->avg_rating }} ({{ $product->review_count }} রিভিউ)
        </div>

        {{-- সার্ভারে রেন্ডার — Google Bot এই দাম ও তথ্য দেখতে পায় --}}
        <form action="{{ route('cart.add') }}" method="POST">
            @csrf
            <select name="size">
                @foreach($product->sizes as $size)
                    <option value="{{ $size }}">{{ $size }}</option>
                @endforeach
            </select>
            <button type="submit">🛒 কার্টে যোগ করুন</button>
        </form>
    </div>

    {{-- রিলেটেড প্রোডাক্ট --}}
    <div class="related-products">
        <h2>একই ধরনের পণ্য</h2>
        @foreach($relatedProducts as $related)
            <a href="{{ route('products.show', $related->slug) }}">
                <img src="{{ $related->thumbnail }}" alt="{{ $related->name }}">
                <p>{{ $related->name }}</p>
                <span>৳{{ number_format($related->price) }}</span>
            </a>
        @endforeach
    </div>
</div>
@endsection
```

```php
// app/Http/Controllers/ProductController.php
class ProductController extends Controller
{
    public function show(string $slug)
    {
        // প্রতি রিকোয়েস্টে ডেটাবেস থেকে ডেটা আনে → HTML রেন্ডার → পাঠায়
        $product = Product::where('slug', $slug)
            ->with(['images', 'reviews', 'seller'])
            ->firstOrFail();

        $relatedProducts = Product::where('category_id', $product->category_id)
            ->where('id', '!=', $product->id)
            ->limit(8)
            ->get();

        // Blade টেমপ্লেটে ডেটা পাঠানো — সার্ভারে HTML তৈরি হয়
        return view('products.show', compact('product', 'relatedProducts'));
    }
}
```

### Next.js SSR উদাহরণ (Daraz-স্টাইল প্রোডাক্ট পেজ):

```jsx
// app/product/[slug]/page.jsx (Next.js App Router — SSR)
// এই পেজ প্রতি রিকোয়েস্টে সার্ভারে রেন্ডার হয়

export const dynamic = 'force-dynamic'; // SSR নিশ্চিত করে

async function getProduct(slug) {
    const res = await fetch(`https://api.daraz.com.bd/products/${slug}`, {
        cache: 'no-store' // প্রতিবার ফ্রেশ ডেটা আনে
    });
    return res.json();
}

export async function generateMetadata({ params }) {
    const product = await getProduct(params.slug);
    return {
        title: `${product.name} - Daraz Bangladesh`,
        description: product.description.slice(0, 160),
        openGraph: {
            images: [product.image_url],
        },
    };
}

export default async function ProductPage({ params }) {
    const product = await getProduct(params.slug);

    return (
        <div className="product-page">
            <h1>{product.name}</h1>
            <span className="price">৳{product.price.toLocaleString()}</span>

            {product.discount > 0 && (
                <span className="discount">-{product.discount}% ছাড়!</span>
            )}

            <p>⭐ {product.rating} ({product.reviewCount} রিভিউ)</p>

            {/* এই HTML সার্ভারে তৈরি হয় — SEO ফ্রেন্ডলি */}
            <AddToCartButton productId={product.id} />
        </div>
    );
}
```

### Express.js + React SSR উদাহরণ:

```javascript
// server.js — Express + React SSR
import express from 'express';
import { renderToString } from 'react-dom/server';
import App from './src/App';

const app = express();

app.get('*', async (req, res) => {
    // সার্ভারে React কম্পোনেন্ট থেকে HTML স্ট্রিং তৈরি
    const appHtml = renderToString(<App url={req.url} />);

    const html = `
        <!DOCTYPE html>
        <html lang="bn">
        <head><title>Pathao Food</title></head>
        <body>
            <div id="root">${appHtml}</div>
            <script src="/bundle.js"></script>
        </body>
        </html>
    `;

    res.send(html);
});

app.listen(3000);
```

### ✅ SSR-এর সুবিধা

| সুবিধা | ব্যাখ্যা |
|--------|---------|
| চমৎকার SEO | সার্চ ইঞ্জিন পূর্ণ HTML পায় — কন্টেন্ট সঠিকভাবে ইন্ডেক্স হয় |
| দ্রুত FCP | ব্যবহারকারী তাৎক্ষণিকভাবে কন্টেন্ট দেখে (First Contentful Paint) |
| JS ছাড়াও কাজ করে | JavaScript নিষ্ক্রিয় থাকলেও মূল কন্টেন্ট দৃশ্যমান |
| সোশ্যাল শেয়ারিং | OG ট্যাগ সার্ভারে থাকে — Facebook/Twitter প্রিভিউ কাজ করে |
| ডায়নামিক কন্টেন্ট | প্রতি রিকোয়েস্টে ফ্রেশ ডেটা দেখানো সম্ভব |

### ❌ SSR-এর অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---------|---------|
| সার্ভার লোড | প্রতি রিকোয়েস্টে HTML রেন্ডার করতে হয় — বেশি CPU/RAM লাগে |
| ধীর TTFB | সার্ভার রেন্ডারিং-এ সময় লাগে (Time to First Byte বেড়ে যায়) |
| জটিলতা | হাইড্রেশন বাগ, সার্ভার/ক্লায়েন্ট মিসম্যাচ সমস্যা হতে পারে |
| খরচ বেশি | সার্ভারলেস / VPS-এ বেশি রিসোর্স প্রয়োজন |
| ক্যাশিং কঠিন | ইউজার-ভিত্তিক কন্টেন্ট থাকলে ক্যাশ করা জটিল |

---

## 📖 ৩. SSG — স্ট্যাটিক সাইট জেনারেশন (Static Site Generation)

### কীভাবে কাজ করে?

SSG-তে **বিল্ড টাইমে** সমস্ত পেজের HTML ফাইল তৈরি করে রাখা হয়। এই স্ট্যাটিক ফাইলগুলো **CDN থেকে সরাসরি** সার্ভ করা হয় — কোনো সার্ভার-সাইড প্রসেসিং দরকার নেই।

```
  SSG ফ্লো (Static Site Generation Flow)

  ডেভেলপার         বিল্ড সার্ভার          CDN               ব্রাউজার
  ────────         ───────────          ───               ───────
     │                  │                 │                   │
     │  ① npm run build │                 │                   │
     │ ───────────────▶ │                 │                   │
     │                  │                 │                   │
     │                  │  ② সব পেজের     │                   │
     │                  │  HTML তৈরি      │                   │
     │                  │  /index.html    │                   │
     │                  │  /about.html    │                   │
     │                  │  /blog/1.html   │                   │
     │                  │  /blog/2.html   │                   │
     │                  │  ... (৫০০০+)    │                   │
     │                  │                 │                   │
     │                  │  ③ CDN-এ আপলোড  │                   │
     │                  │ ──────────────▶ │                   │
     │                  │                 │                   │
     │                  │                 │  ④ GET /blog/1    │
     │                  │                 │ ◀──────────────── │
     │                  │                 │                   │
     │                  │                 │  ⑤ আগে-থেকে-     │
     │                  │                 │  তৈরি HTML        │
     │                  │                 │ ──────────────▶── │
     │                  │                 │   ⚡ < ৫০ms!      │
     │                  │                 │                   │
```

### Next.js SSG উদাহরণ (Prothom Alo-স্টাইল ব্লগ):

```jsx
// app/blog/[slug]/page.jsx — Next.js SSG

// বিল্ড টাইমে কোন কোন পেজ তৈরি করতে হবে তা বলুন
export async function generateStaticParams() {
    const posts = await fetch('https://api.prothomalo.com/posts')
        .then(res => res.json());

    // প্রতিটি ব্লগ পোস্টের জন্য একটি স্ট্যাটিক পেজ তৈরি হবে
    return posts.map(post => ({
        slug: post.slug,
    }));
}

export default async function BlogPost({ params }) {
    const post = await fetch(
        `https://api.prothomalo.com/posts/${params.slug}`
    ).then(res => res.json());

    return (
        <article>
            <h1>{post.title}</h1>
            <time>{new Date(post.publishedAt).toLocaleDateString('bn-BD')}</time>
            <div dangerouslySetInnerHTML={{ __html: post.content }} />
        </article>
    );
}
```

### SSG টুলস ও ফ্রেমওয়ার্ক:

```
  ┌─────────────────────────────────────────────────────────────┐
  │              জনপ্রিয় SSG টুলস                                │
  ├──────────────┬───────────────┬──────────────────────────────┤
  │ টুল          │ ভাষা          │ সেরা ব্যবহার                  │
  ├──────────────┼───────────────┼──────────────────────────────┤
  │ Next.js      │ React         │ ফুল-স্ট্যাক অ্যাপ + ব্লগ     │
  │ Gatsby       │ React         │ মার্কেটিং সাইট, ব্লগ          │
  │ Astro        │ Multi-fw      │ কন্টেন্ট সাইট (কম JS)        │
  │ Hugo         │ Go            │ অতি দ্রুত বিল্ড, ডকুমেন্টেশন  │
  │ 11ty         │ JS            │ সিম্পল স্ট্যাটিক সাইট         │
  │ Nuxt.js      │ Vue           │ Vue-ভিত্তিক স্ট্যাটিক সাইট   │
  └──────────────┴───────────────┴──────────────────────────────┘
```

### ✅ SSG-এর সুবিধা

| সুবিধা | ব্যাখ্যা |
|--------|---------|
| সর্বোচ্চ গতি | CDN থেকে প্রি-বিল্ট HTML — সবচেয়ে দ্রুত লোড টাইম |
| সর্বোচ্চ নিরাপত্তা | কোনো সার্ভার নেই — হ্যাক করার কিছু নেই |
| সস্তা হোস্টিং | স্ট্যাটিক ফাইল — Netlify/Vercel/Cloudflare Pages-এ ফ্রি হোস্ট |
| চমৎকার SEO | পূর্ণ HTML আগে থেকে তৈরি — সার্চ ইঞ্জিনের জন্য আদর্শ |
| নির্ভরযোগ্যতা | সার্ভার ডাউন হলেও CDN চলতে থাকে |

### ❌ SSG-এর অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---------|---------|
| ধীর বিল্ড | হাজার হাজার পেজ থাকলে বিল্ড টাইম অনেক বেড়ে যায় |
| পুরনো ডেটা | নতুন বিল্ড না হওয়া পর্যন্ত পুরনো কন্টেন্ট থাকে |
| ডায়নামিক কন্টেন্ট অসম্ভব | ইউজার-ভিত্তিক বা ঘন ঘন বদলানো কন্টেন্টের জন্য উপযুক্ত নয় |
| বিল্ড জটিলতা | বড় সাইটের জন্য বিল্ড পাইপলাইন পরিচালনা কঠিন |

---

## 📖 ৪. ISR — ইনক্রিমেন্টাল স্ট্যাটিক রিজেনারেশন (Incremental Static Regeneration)

### কীভাবে কাজ করে?

ISR হলো **SSG + SSR এর সেরা সমন্বয়**। পেজগুলো SSG-এর মতো স্ট্যাটিক থাকে, কিন্তু নির্দিষ্ট সময় পর **ব্যাকগ্রাউন্ডে আপডেট** হয়। এটি **stale-while-revalidate** প্যাটার্ন অনুসরণ করে।

```
  ISR ফ্লো (Incremental Static Regeneration Flow)

  ব্রাউজার               CDN/Edge              সার্ভার            ডেটাবেস
  ───────               ────────              ──────            ────────
     │                     │                     │                  │
     │  ① GET /product/123 │                     │                  │
     │ ──────────────────▶ │                     │                  │
     │                     │                     │                  │
     │                     │  ক্যাশে আছে?         │                  │
     │                     │  হ্যাঁ + মেয়াদ আছে   │                  │
     │                     │                     │                  │
     │  ② ক্যাশড HTML ⚡    │                     │                  │
     │ ◀────────────────── │                     │                  │
     │                     │                     │                  │
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
     ৬০ সেকেন্ড পর (revalidate: 60)...
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
     │                     │                     │                  │
     │  ③ GET /product/123 │                     │                  │
     │ ──────────────────▶ │                     │                  │
     │                     │                     │                  │
     │                     │  ক্যাশে আছে কিন্তু   │                  │
     │                     │  মেয়াদ শেষ (stale)   │                  │
     │                     │                     │                  │
     │  ④ পুরনো HTML দেয়   │                     │                  │
     │ ◀────────────────── │  (ইউজার অপেক্ষা     │                  │
     │  (stale কিন্তু দ্রুত) │  করে না!)            │                  │
     │                     │                     │                  │
     │                     │  ⑤ ব্যাকগ্রাউন্ডে    │                  │
     │                     │  রিজেনারেশন          │                  │
     │                     │ ──────────────────▶ │                  │
     │                     │                     │ ───────────────▶ │
     │                     │                     │ ◀─────────────── │
     │                     │                     │                  │
     │                     │  ⑥ নতুন HTML         │                  │
     │                     │  ক্যাশে আপডেট        │                  │
     │                     │ ◀────────────────── │                  │
     │                     │                     │                  │
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
     পরের রিকোয়েস্ট...
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
     │                     │                     │                  │
     │  ⑦ GET /product/123 │                     │                  │
     │ ──────────────────▶ │                     │                  │
     │  ⑧ নতুন HTML! ⚡     │                     │                  │
     │ ◀────────────────── │                     │                  │
     │                     │                     │                  │
```

### Next.js ISR উদাহরণ (Daraz ক্যাটাগরি পেজ):

```jsx
// app/category/[slug]/page.jsx — Next.js ISR

// প্রতি ৬০ সেকেন্ডে ব্যাকগ্রাউন্ডে রিভ্যালিডেট করবে
export const revalidate = 60;

export async function generateStaticParams() {
    const categories = await fetch('https://api.daraz.com.bd/categories')
        .then(res => res.json());

    return categories.map(cat => ({ slug: cat.slug }));
}

export default async function CategoryPage({ params }) {
    const products = await fetch(
        `https://api.daraz.com.bd/categories/${params.slug}/products`
    ).then(res => res.json());

    return (
        <div>
            <h1>{params.slug} ক্যাটাগরি</h1>
            <p>মোট {products.length} পণ্য পাওয়া গেছে</p>

            <div className="product-grid">
                {products.map(product => (
                    <ProductCard key={product.id} product={product} />
                ))}
            </div>
        </div>
    );
}
```

### On-Demand Revalidation (চাহিদা অনুযায়ী রিভ্যালিডেশন):

```javascript
// app/api/revalidate/route.js — Next.js API Route
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request) {
    const { secret, path, tag } = await request.json();

    // সিকিউরিটি চেক
    if (secret !== process.env.REVALIDATION_SECRET) {
        return Response.json({ error: 'অবৈধ টোকেন' }, { status: 401 });
    }

    // নির্দিষ্ট পাথ রিভ্যালিডেট
    if (path) {
        revalidatePath(path);
        return Response.json({ revalidated: true, path });
    }

    // ট্যাগ-ভিত্তিক রিভ্যালিডেশন
    if (tag) {
        revalidateTag(tag);
        return Response.json({ revalidated: true, tag });
    }
}

// ব্যবহার: যখন CMS-এ নতুন পোস্ট পাবলিশ হয়
// POST /api/revalidate { secret: "xxx", path: "/blog/new-post" }
```

### ✅ ISR-এর সুবিধা

| সুবিধা | ব্যাখ্যা |
|--------|---------|
| SSG-এর মতো দ্রুত | বেশিরভাগ সময় ক্যাশড HTML সার্ভ হয় |
| আপ-টু-ডেট ডেটা | নির্দিষ্ট বিরতিতে ব্যাকগ্রাউন্ডে আপডেট হয় |
| কম সার্ভার লোড | শুধু revalidation-এ সার্ভার কাজ করে, প্রতি রিকোয়েস্টে নয় |
| দ্রুত বিল্ড | সব পেজ বিল্ড টাইমে তৈরি করতে হয় না |

### ❌ ISR-এর অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---------|---------|
| সামান্য পুরনো ডেটা | revalidate সময়ের মধ্যে পুরনো কন্টেন্ট দেখা যেতে পারে |
| প্ল্যাটফর্ম নির্ভরতা | মূলত Next.js/Vercel-এ ভালো কাজ করে |
| ডিবাগিং কঠিন | ক্যাশিং সংক্রান্ত সমস্যা খুঁজে বের করা জটিল |

---

## 📊 তুলনামূলক বিশ্লেষণ — CSR vs SSR vs SSG vs ISR

### সম্পূর্ণ তুলনা টেবিল:

```
  ┌──────────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
  │ বৈশিষ্ট্য         │   CSR       │    SSR      │    SSG      │    ISR      │
  ├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
  │ HTML তৈরি হয়     │ ব্রাউজারে    │ সার্ভারে     │ বিল্ড টাইমে  │ বিল্ড + রিভ্যা│
  │ SEO              │ ❌ খারাপ     │ ✅ চমৎকার   │ ✅ চমৎকার   │ ✅ চমৎকার   │
  │ প্রথম লোড (FCP)   │ 🐌 ধীর      │ 🚀 দ্রুত    │ ⚡ সবচেয়ে   │ ⚡ খুব দ্রুত │
  │                  │             │             │    দ্রুত     │             │
  │ TTFB             │ ⚡ দ্রুত     │ 🐌 ধীর      │ ⚡ দ্রুত     │ ⚡ দ্রুত     │
  │ ডেটা ফ্রেশনেস     │ ✅ রিয়েল-   │ ✅ প্রতি     │ ❌ বিল্ড    │ ✅ কনফিগার- │
  │                  │    টাইম     │    রিকোয়েস্ট │    পর্যন্ত   │    যোগ্য    │
  │ সার্ভার লোড      │ ✅ কম       │ ❌ বেশি     │ ✅ নেই      │ ✅ কম       │
  │ হোস্টিং খরচ      │ 💰 কম      │ 💰💰💰 বেশি │ 💰 সবচেয়ে  │ 💰💰 মাঝারি │
  │                  │             │             │    কম       │             │
  │ ইন্টারঅ্যাক্টিভিটি│ ✅ সর্বোচ্চ  │ ✅ ভালো     │ 🔶 সীমিত   │ 🔶 সীমিত   │
  │ CDN ক্যাশযোগ্য    │ ✅ (JS)     │ ❌ কঠিন    │ ✅ সম্পূর্ণ  │ ✅ সম্পূর্ণ  │
  │ স্কেলেবিলিটি     │ ✅ সহজ      │ ❌ কঠিন    │ ✅ সহজ      │ ✅ সহজ      │
  │ জটিলতা           │ 🔶 মাঝারি   │ ❌ বেশি    │ ✅ কম       │ 🔶 মাঝারি   │
  └──────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### বাংলাদেশি সাইটের প্রেক্ষাপটে কোনটি কোথায় মানানসই:

```
  ┌───────────────────────────────┬──────────────┬──────────────────────────┐
  │ বাংলাদেশি সাইট                │ আদর্শ প্যাটার্ন │ কেন                      │
  ├───────────────────────────────┼──────────────┼──────────────────────────┤
  │ Prothom Alo (নিউজ)            │ ISR / SSG    │ SEO গুরুত্বপূর্ণ, কন্টেন্ট│
  │                               │              │ ঘন ঘন বদলায়              │
  │ Daraz (ই-কমার্স)              │ SSR + ISR    │ SEO + ডায়নামিক দাম/স্টক  │
  │ bKash Merchant Dashboard      │ CSR          │ লগইন-নির্ভর, SEO দরকার   │
  │                               │              │ নেই                      │
  │ Pathao (রাইড শেয়ারিং)         │ CSR + SSR    │ অ্যাপ: CSR, ল্যান্ডিং: SSR│
  │ Nagad (মোবাইল ব্যাংকিং)       │ CSR          │ সিকিউর ড্যাশবোর্ড         │
  │ Grameenphone (টেলকো)          │ SSR + SSG    │ মার্কেটিং পেজ + অফার      │
  │ Robi/Banglalink               │ SSG + ISR    │ প্যাকেজ তথ্য, অফার পেজ    │
  │ BD Railway Ticket             │ SSR          │ রিয়েল-টাইম সিট তথ্য       │
  │ সরকারি ওয়েবসাইট              │ SSG          │ স্ট্যাটিক তথ্য, কম আপডেট   │
  │ ব্যক্তিগত ব্লগ/পোর্টফোলিও     │ SSG          │ সিম্পল, দ্রুত, সস্তা       │
  └───────────────────────────────┴──────────────┴──────────────────────────┘
```

---

## 📖 ৫. পারফরম্যান্স মেট্রিক্স তুলনা

```
  Core Web Vitals প্রভাব:

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  LCP (Largest Contentful Paint) — সবচেয়ে বড় কন্টেন্ট কখন দৃশ্যমান    │
  │  ─────────────────────────────────────────────────────────────────────  │
  │  CSR:  ████████████████████████████████████░░░░░  3.5s  ❌ খারাপ       │
  │  SSR:  █████████████████░░░░░░░░░░░░░░░░░░░░░░░  1.8s  ✅ ভালো        │
  │  SSG:  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.8s  ✅ সেরা        │
  │  ISR:  █████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.9s  ✅ সেরা        │
  │                                                                         │
  │  FID (First Input Delay) — প্রথম ইন্টারঅ্যাকশনে বিলম্ব                  │
  │  ─────────────────────────────────────────────────────────────────────  │
  │  CSR:  ████████████████████████░░░░░░░░░░░░░░░  250ms  ❌ খারাপ       │
  │  SSR:  ██████████████████░░░░░░░░░░░░░░░░░░░░░  180ms  🔶 মাঝারি      │
  │  SSG:  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   50ms  ✅ সেরা        │
  │  ISR:  █████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   60ms  ✅ সেরা        │
  │                                                                         │
  │  CLS (Cumulative Layout Shift) — লেআউট স্থিতিশীলতা                     │
  │  ─────────────────────────────────────────────────────────────────────  │
  │  CSR:  ████████████████░░░░░░░░░░░░░░░░░░░░░░░  0.25   ❌ খারাপ       │
  │  SSR:  █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.05   ✅ ভালো        │
  │  SSG:  ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.02   ✅ সেরা        │
  │  ISR:  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  0.03   ✅ সেরা        │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## 📖 ৬. হাইব্রিড অ্যাপ্রোচ (Hybrid / Mixed Rendering)

আধুনিক ফ্রেমওয়ার্কগুলো এখন **একই অ্যাপ্লিকেশনে একাধিক রেন্ডারিং প্যাটার্ন** ব্যবহার করতে দেয়। প্রতিটি পেজ বা কম্পোনেন্ট তার প্রয়োজন অনুযায়ী আলাদা প্যাটার্নে রেন্ডার হতে পারে।

### 🏝️ আইল্যান্ড আর্কিটেকচার (Island Architecture — Astro)

```
  আইল্যান্ড আর্কিটেকচার:

  ┌──────────────────────────────────────────────────────────────┐
  │                  স্ট্যাটিক HTML পেজ (সমুদ্র)                  │
  │                                                              │
  │   ┌──────────┐     স্ট্যাটিক      ┌──────────────────┐       │
  │   │ 🏝️ React │     কন্টেন্ট      │ 🏝️ Vue            │       │
  │   │ কাউন্টার  │    (JS নেই)      │ ইমেজ ক্যারোসেল    │       │
  │   │ কম্পোনেন্ট│                   │ কম্পোনেন্ট         │       │
  │   └──────────┘                   └──────────────────┘       │
  │                                                              │
  │   স্ট্যাটিক          ┌──────────────────┐    স্ট্যাটিক      │
  │   হেডার              │ 🏝️ Svelte         │    ফুটার         │
  │   (JS নেই)          │ সার্চ বার          │    (JS নেই)     │
  │                      │ কম্পোনেন্ট         │                  │
  │                      └──────────────────┘                   │
  │                                                              │
  │   মূলনীতি: পুরো পেজ স্ট্যাটিক, শুধু ইন্টারঅ্যাক্টিভ অংশে   │
  │   (আইল্যান্ড) JS লোড হয়। বাকি অংশ শূন্য JavaScript!        │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### React Server Components (RSC):

```
  React Server Components — সার্ভার ও ক্লায়েন্ট কম্পোনেন্ট আলাদা:

  ┌──────────────────────────────────────────────────────────────┐
  │  সার্ভার কম্পোনেন্ট (ডিফল্ট)         ক্লায়েন্ট কম্পোনেন্ট   │
  │  ─────────────────────              ───────────────────     │
  │  • DB সরাসরি অ্যাক্সেস              • "use client" দিয়ে     │
  │  • কোনো JS ব্রাউজারে যায় না          মার্ক করতে হয়           │
  │  • async/await সরাসরি ব্যবহার       • useState, useEffect   │
  │  • বড় লাইব্রেরি ইম্পোর্ট            • onClick ইভেন্ট        │
  │    (বান্ডেল সাইজে প্রভাব নেই)       • ব্রাউজার API           │
  │                                                              │
  │  উদাহরণ:                           উদাহরণ:                  │
  │  ProductList (ডেটা ফেচ)             AddToCart (বাটন ক্লিক)   │
  │  BlogContent (মার্কডাউন পার্স)       SearchBar (ইনপুট)        │
  │  UserProfile (DB কুয়েরি)            ImageGallery (স্লাইডার)  │
  └──────────────────────────────────────────────────────────────┘
```

### Next.js App Router-এ মিক্সড রেন্ডারিং:

```jsx
// Next.js App Router — একই অ্যাপে মিক্সড রেন্ডারিং

// ─── হোমপেজ: SSG (স্ট্যাটিক) ─────────────────────
// app/page.jsx
export default function Home() {
    return <h1>Daraz Bangladesh-এ স্বাগতম</h1>;
}

// ─── প্রোডাক্ট পেজ: ISR (৬০s রিভ্যালিডেট) ──────────
// app/product/[slug]/page.jsx
export const revalidate = 60;
export default async function Product({ params }) {
    const product = await getProduct(params.slug);
    return <ProductDetail product={product} />;
}

// ─── সার্চ রেজাল্ট: SSR (প্রতি রিকোয়েস্টে) ─────────
// app/search/page.jsx
export const dynamic = 'force-dynamic';
export default async function Search({ searchParams }) {
    const results = await searchProducts(searchParams.q);
    return <SearchResults results={results} />;
}

// ─── ড্যাশবোর্ড: CSR (ক্লায়েন্ট-সাইড) ───────────────
// app/dashboard/page.jsx
'use client';
export default function Dashboard() {
    const [data, setData] = useState(null);
    useEffect(() => { fetchDashboardData().then(setData); }, []);
    return <DashboardView data={data} />;
}
```

### Streaming SSR (স্ট্রিমিং SSR):

```
  ঐতিহ্যবাহী SSR বনাম Streaming SSR:

  ── ঐতিহ্যবাহী SSR ──────────────────────────────────────────

  সার্ভার:  [████ ডেটা ফেচ ████][████ রেন্ডার ████] ─── পাঠানো ──▶
  ব্রাউজার: [░░░░░░░░░ অপেক্ষা ░░░░░░░░░░░░░░░░░░░] [██ দেখানো ██]

  সব কিছু রেডি না হওয়া পর্যন্ত ব্যবহারকারী কিছুই দেখে না!

  ── Streaming SSR ────────────────────────────────────────────

  সার্ভার:  [██ শেল ██] ──▶ [███ ডেটা ১ ███] ──▶ [███ ডেটা ২ ███]
  ব্রাউজার: [██ শেল ██]    [██ আপডেট ১ ██]    [██ আপডেট ২ ██]
                ▲                 ▲                    ▲
              তাৎক্ষণিক         প্রথম ডেটা            বাকি ডেটা
              লেআউট দেখে       আসলে দেখায়           পূর্ণাঙ্গ পেজ

  ব্যবহারকারী ধীরে ধীরে কন্টেন্ট দেখতে পায় — অপেক্ষার অনুভূতি কম!
```

```jsx
// Next.js Streaming SSR with Suspense
import { Suspense } from 'react';

export default function ProductPage({ params }) {
    return (
        <div>
            {/* এই অংশ তাৎক্ষণিকভাবে পাঠানো হয় */}
            <h1>পণ্যের বিবরণ</h1>
            <ProductInfo slug={params.slug} />

            {/* রিভিউ ধীরে লোড হলেও পেজ আটকে থাকে না */}
            <Suspense fallback={<p>রিভিউ লোড হচ্ছে...</p>}>
                <ProductReviews slug={params.slug} />
            </Suspense>

            {/* সাজেশন আলাদাভাবে স্ট্রিম হয় */}
            <Suspense fallback={<p>সাজেস্টেড পণ্য লোড হচ্ছে...</p>}>
                <RelatedProducts slug={params.slug} />
            </Suspense>
        </div>
    );
}
```

---

## 📖 ৭. PHP প্রেক্ষাপট — Laravel রেন্ডারিং অপশন

PHP/Laravel ইকোসিস্টেমে রেন্ডারিং-এর তিনটি প্রধান পদ্ধতি রয়েছে:

```
  Laravel রেন্ডারিং ইকোসিস্টেম:

  ┌───────────────────────────────────────────────────────────────────┐
  │                                                                   │
  │  ① Laravel Blade (ঐতিহ্যবাহী SSR)                                │
  │  ─────────────────────────────────                                │
  │  সার্ভারে PHP দিয়ে HTML তৈরি → ব্রাউজারে পাঠানো                   │
  │  সবচেয়ে সিম্পল, সবচেয়ে পুরনো পদ্ধতি                             │
  │                                                                   │
  │  ② Inertia.js (SPA অনুভূতি + SSR সুবিধা)                         │
  │  ──────────────────────────────────────                           │
  │  Laravel ব্যাকএন্ড + React/Vue ফ্রন্টএন্ড                        │
  │  SPA-এর মতো নেভিগেশন, কিন্তু Laravel রাউটিং ও কন্ট্রোলার ব্যবহার│
  │                                                                   │
  │  ③ Livewire (JS ছাড়া রিঅ্যাক্টিভিটি)                             │
  │  ─────────────────────────────────                                │
  │  PHP কম্পোনেন্ট যা AJAX দিয়ে রিঅ্যাক্টিভ হয়                      │
  │  JavaScript ফ্রেমওয়ার্ক শেখার দরকার নেই                           │
  │                                                                   │
  └───────────────────────────────────────────────────────────────────┘
```

### Inertia.js উদাহরণ (Pathao-স্টাইল রাইড ম্যানেজমেন্ট):

```php
// Laravel Controller — Inertia.js ব্যবহার
// app/Http/Controllers/RideController.php
class RideController extends Controller
{
    public function index()
    {
        $rides = Ride::where('user_id', auth()->id())
            ->with('driver')
            ->latest()
            ->paginate(20);

        // Inertia রেসপন্স — React/Vue কম্পোনেন্টে ডেটা পাঠায়
        return Inertia::render('Rides/Index', [
            'rides' => $rides,
            'stats' => [
                'total_rides' => $rides->total(),
                'total_spent' => auth()->user()->total_spent,
            ],
        ]);
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'pickup' => 'required|string',
            'destination' => 'required|string',
        ]);

        $ride = Ride::create([
            'user_id' => auth()->id(),
            'pickup_location' => $validated['pickup'],
            'destination' => $validated['destination'],
            'status' => 'searching',
        ]);

        // Inertia রিডাইরেক্ট — ফুল পেজ রিলোড হয় না (SPA-এর মতো)
        return redirect()->route('rides.show', $ride);
    }
}
```

```jsx
// resources/js/Pages/Rides/Index.jsx — Inertia React কম্পোনেন্ট
import { Head, Link, usePage } from '@inertiajs/react';

export default function RidesIndex({ rides, stats }) {
    return (
        <>
            <Head title="আমার রাইডসমূহ - Pathao" />

            <div className="dashboard">
                <h1>আমার রাইডসমূহ</h1>
                <div className="stats">
                    <p>মোট রাইড: {stats.total_rides}</p>
                    <p>মোট খরচ: ৳{stats.total_spent}</p>
                </div>

                {rides.data.map(ride => (
                    <Link key={ride.id} href={`/rides/${ride.id}`}>
                        <div className="ride-card">
                            <p>{ride.pickup_location} → {ride.destination}</p>
                            <p>ড্রাইভার: {ride.driver?.name}</p>
                            <span>{ride.status}</span>
                        </div>
                    </Link>
                ))}
            </div>
        </>
    );
}
```

### Livewire উদাহরণ (Nagad-স্টাইল ট্রানজেকশন লিস্ট):

```php
// app/Livewire/TransactionList.php
use Livewire\Component;
use Livewire\WithPagination;

class TransactionList extends Component
{
    use WithPagination;

    public string $search = '';
    public string $filter = 'all';

    // সার্চ বদলালে প্রথম পেজে ফিরে যায়
    public function updatingSearch()
    {
        $this->resetPage();
    }

    public function render()
    {
        $transactions = Transaction::query()
            ->where('user_id', auth()->id())
            ->when($this->search, fn($q) =>
                $q->where('reference', 'like', "%{$this->search}%")
            )
            ->when($this->filter !== 'all', fn($q) =>
                $q->where('type', $this->filter)
            )
            ->latest()
            ->paginate(15);

        return view('livewire.transaction-list', [
            'transactions' => $transactions,
        ]);
    }
}
```

```html
{{-- resources/views/livewire/transaction-list.blade.php --}}
<div>
    <h2>লেনদেন তালিকা</h2>

    {{-- JavaScript ছাড়াই রিঅ্যাক্টিভ সার্চ! --}}
    <input wire:model.live.debounce.300ms="search"
           placeholder="রেফারেন্স নম্বর খুঁজুন...">

    <select wire:model.live="filter">
        <option value="all">সব লেনদেন</option>
        <option value="send">পাঠানো</option>
        <option value="receive">প্রাপ্ত</option>
        <option value="payment">পেমেন্ট</option>
    </select>

    {{-- Livewire স্বয়ংক্রিয়ভাবে এই অংশ আপডেট করে --}}
    <div wire:loading class="spinner">লোড হচ্ছে...</div>

    <table>
        <thead>
            <tr>
                <th>তারিখ</th>
                <th>ধরন</th>
                <th>পরিমাণ</th>
                <th>স্ট্যাটাস</th>
            </tr>
        </thead>
        <tbody>
            @foreach($transactions as $tx)
            <tr>
                <td>{{ $tx->created_at->format('d/m/Y h:i A') }}</td>
                <td>{{ $tx->type === 'send' ? 'পাঠানো' : 'প্রাপ্ত' }}</td>
                <td>৳{{ number_format($tx->amount) }}</td>
                <td>{{ $tx->status === 'completed' ? '✅ সম্পন্ন' : '⏳ প্রক্রিয়াধীন' }}</td>
            </tr>
            @endforeach
        </tbody>
    </table>

    {{ $transactions->links() }}
</div>
```

```
  Laravel রেন্ডারিং অপশন তুলনা:

  ┌──────────────┬─────────────────┬───────────────────┬─────────────────┐
  │ বৈশিষ্ট্য     │ Blade (SSR)     │ Inertia.js        │ Livewire        │
  ├──────────────┼─────────────────┼───────────────────┼─────────────────┤
  │ রেন্ডারিং     │ সার্ভার          │ সার্ভার + ক্লায়েন্ট │ সার্ভার          │
  │ JS ফ্রেমওয়ার্ক│ প্রয়োজন নেই    │ React/Vue দরকার   │ প্রয়োজন নেই    │
  │ SPA অনুভূতি  │ ❌ নেই          │ ✅ সম্পূর্ণ        │ 🔶 আংশিক       │
  │ শেখার কষ্ট   │ ✅ সহজ          │ 🔶 মাঝারি         │ ✅ সহজ          │
  │ SEO          │ ✅ চমৎকার       │ 🔶 SSR দরকার     │ ✅ চমৎকার       │
  │ রিয়েল-টাইম   │ ❌ কঠিন        │ 🔶 আলাদা সেটআপ   │ ✅ সহজ          │
  │ জটিল UI      │ ❌ কঠিন        │ ✅ সহজ            │ 🔶 মাঝারি       │
  │ পারফরম্যান্স  │ ✅ দ্রুত         │ ✅ দ্রুত           │ 🔶 AJAX ওভারহেড │
  └──────────────┴─────────────────┴───────────────────┴─────────────────┘
```

---

## 📖 ৮. ডিসিশন ফ্লোচার্ট — কোন রেন্ডারিং প্যাটার্ন বেছে নেবেন?

```
  আপনার প্রজেক্টের জন্য সঠিক রেন্ডারিং প্যাটার্ন নির্বাচন:

                        ┌─────────────────────┐
                        │ আপনার সাইটে SEO     │
                        │ গুরুত্বপূর্ণ?         │
                        └─────────┬───────────┘
                           ┌──────┴──────┐
                           │             │
                          হ্যাঁ           না
                           │             │
                           ▼             ▼
                 ┌──────────────┐  ┌────────────────┐
                 │ কন্টেন্ট কত   │  │ ✅ CSR ব্যবহার  │
                 │ ঘন ঘন বদলায়? │  │ (Dashboard,    │
                 └──────┬───────┘  │  Admin Panel)  │
                        │          └────────────────┘
           ┌────────────┼────────────┐
           │            │            │
        কদাচিৎ      মাঝে মাঝে    প্রতি রিকোয়েস্টে
        (সপ্তাহে)   (ঘণ্টায়)     (প্রতি সেকেন্ডে)
           │            │            │
           ▼            ▼            ▼
    ┌────────────┐ ┌──────────┐ ┌──────────────┐
    │ ✅ SSG     │ │ ✅ ISR   │ │ ✅ SSR       │
    │ সবচেয়ে    │ │ স্ট্যাটিক │ │ প্রতিবার     │
    │ দ্রুত +    │ │ + রিভ্যালি│ │ ফ্রেশ ডেটা    │
    │ সস্তা      │ │ ডেশন     │ │ দেখানো হয়   │
    └────────────┘ └──────────┘ └──────────────┘

    উদাহরণ:          উদাহরণ:        উদাহরণ:
    - ব্লগ            - নিউজ সাইট    - ই-কমার্স সার্চ
    - ডকুমেন্টেশন     - প্রোডাক্ট     - লাইভ ড্যাশবোর্ড
    - পোর্টফোলিও      - ক্যাটাগরি     - বুকিং সিস্টেম
```

### বিস্তারিত ডিসিশন ট্রি:

```
  প্রশ্ন ১: আপনার অ্যাপ কি লগইন-নির্ভর?
  ├── হ্যাঁ → SEO দরকার নেই → CSR (React SPA)
  │         উদাহরণ: bKash Dashboard, Nagad App
  │
  └── না → প্রশ্ন ২: কন্টেন্ট কি ব্যবহারকারী-ভিত্তিক?
       ├── হ্যাঁ → SSR (প্রতি রিকোয়েস্টে সার্ভারে রেন্ডার)
       │         উদাহরণ: Daraz সার্চ রেজাল্ট
       │
       └── না → প্রশ্ন ৩: কন্টেন্ট কত ঘন ঘন বদলায়?
            ├── খুব কম (সপ্তাহে/মাসে) → SSG
            │         উদাহরণ: কোম্পানি ওয়েবসাইট, ব্লগ
            │
            ├── মাঝারি (মিনিট/ঘণ্টায়) → ISR
            │         উদাহরণ: Prothom Alo আর্টিকেল
            │
            └── খুব বেশি (সেকেন্ডে) → SSR
                      উদাহরণ: লাইভ স্কোরবোর্ড
```

---

## 📖 ৯. কখন ব্যবহার করবেন / করবেন না

### CSR — ক্লায়েন্ট-সাইড রেন্ডারিং

```
  ✅ করবেন:                                ❌ করবেন না:
  ─────────                                ──────────────
  • অ্যাডমিন ড্যাশবোর্ড                      • পাবলিক নিউজ সাইট
  • অভ্যন্তরীণ এন্টারপ্রাইজ টুল               • ই-কমার্স ক্যাটালগ
  • রিয়েল-টাইম চ্যাট/কোলাবোরেশন            • ল্যান্ডিং পেজ
  • ডেটা ভিজুয়ালাইজেশন অ্যাপ               • মার্কেটিং সাইট
  • গেম ও ইন্টারঅ্যাক্টিভ টুল                • কন্টেন্ট-ভিত্তিক সাইট

  বাংলাদেশি উদাহরণ:
  ✅ bKash Merchant Dashboard
  ✅ Pathao Driver Dashboard
  ❌ Prothom Alo হোমপেজ
  ❌ Daraz প্রোডাক্ট লিস্টিং
```

### SSR — সার্ভার-সাইড রেন্ডারিং

```
  ✅ করবেন:                                ❌ করবেন না:
  ─────────                                ──────────────
  • ই-কমার্স প্রোডাক্ট পেজ                  • সিম্পল স্ট্যাটিক সাইট
  • সোশ্যাল মিডিয়া প্ল্যাটফর্ম              • হাই-ট্রাফিক স্ট্যাটিক কন্টেন্ট
  • পার্সোনালাইজড কন্টেন্ট                  • অফলাইন-ফার্স্ট অ্যাপ
  • সার্চ রেজাল্ট পেজ                      • বাজেট সীমিত (সার্ভার খরচ)
  • ডায়নামিক ড্যাশবোর্ড (SEO দরকার)        • খুব হাই-ট্রাফিক (লক্ষ RPM)

  বাংলাদেশি উদাহরণ:
  ✅ Daraz সার্চ ও প্রোডাক্ট পেজ
  ✅ Chaldal ক্যাটাগরি পেজ
  ❌ ব্যক্তিগত ব্লগ
  ❌ ডকুমেন্টেশন সাইট
```

### SSG — স্ট্যাটিক সাইট জেনারেশন

```
  ✅ করবেন:                                ❌ করবেন না:
  ─────────                                ──────────────
  • ব্লগ ও ডকুমেন্টেশন                      • রিয়েল-টাইম ড্যাশবোর্ড
  • মার্কেটিং ল্যান্ডিং পেজ                  • ই-কমার্স সার্চ পেজ
  • কোম্পানি ওয়েবসাইট                      • সোশ্যাল মিডিয়া ফিড
  • পোর্টফোলিও সাইট                        • লাইভ আপডেট প্রয়োজন
  • API ডকুমেন্টেশন                        • ইউজার-জেনারেটেড কন্টেন্ট

  বাংলাদেশি উদাহরণ:
  ✅ Grameenphone প্যাকেজ পেজ
  ✅ সরকারি তথ্য পোর্টাল
  ❌ bKash লেনদেন পেজ
  ❌ Pathao রাইড ট্র্যাকিং
```

### ISR — ইনক্রিমেন্টাল স্ট্যাটিক রিজেনারেশন

```
  ✅ করবেন:                                ❌ করবেন না:
  ─────────                                ──────────────
  • নিউজ আর্টিকেল পেজ                     • রিয়েল-টাইম ডেটা
  • ই-কমার্স ক্যাটাগরি পেজ                  • ইউজার-স্পেসিফিক কন্টেন্ট
  • প্রোডাক্ট ডিটেইলস পেজ                  • চ্যাট অ্যাপ্লিকেশন
  • FAQ ও হেল্প পেজ                       • লাইভ ট্র্যাকিং
  • ইভেন্ট লিস্টিং                          • ফিন্যান্সিয়াল ট্রেডিং

  বাংলাদেশি উদাহরণ:
  ✅ Prothom Alo আর্টিকেল পেজ
  ✅ Daraz ক্যাটাগরি পেজ
  ❌ Nagad ট্রানজেকশন হিস্ট্রি
  ❌ Cricket লাইভ স্কোর
```

---

## 📖 ১০. অ্যান্টি-প্যাটার্নস (❌ Bad / ✅ Good)

### ❌ অ্যান্টি-প্যাটার্ন ১: SEO-নির্ভর পেজে CSR ব্যবহার

```jsx
// ❌ ভুল: Prothom Alo-এর মতো নিউজ সাইটে CSR
// Google Bot খালি পেজ দেখবে — আর্টিকেল ইন্ডেক্স হবে না!

function NewsArticle() {
    const [article, setArticle] = useState(null);

    useEffect(() => {
        fetch(`/api/articles/${slug}`)
            .then(res => res.json())
            .then(setArticle);
    }, [slug]);

    if (!article) return <div>লোড হচ্ছে...</div>;

    return (
        <article>
            <h1>{article.title}</h1>  {/* Google Bot এটি দেখে না! */}
            <p>{article.content}</p>
        </article>
    );
}
```

```jsx
// ✅ সঠিক: SSR বা ISR দিয়ে নিউজ আর্টিকেল — সার্ভারেই HTML তৈরি
// app/articles/[slug]/page.jsx (Next.js)

export const revalidate = 300; // ৫ মিনিটে রিভ্যালিডেট (ISR)

export default async function NewsArticle({ params }) {
    const article = await fetch(
        `https://api.prothomalo.com/articles/${params.slug}`
    ).then(res => res.json());

    return (
        <article>
            <h1>{article.title}</h1>  {/* Google Bot সরাসরি দেখতে পায়! */}
            <p>{article.content}</p>
        </article>
    );
}
```

### ❌ অ্যান্টি-প্যাটার্ন ২: প্রতি রিকোয়েস্টে SSR যেখানে SSG/ISR যথেষ্ট

```jsx
// ❌ ভুল: About পেজের জন্য SSR — প্রতিবার সার্ভার কাজ করে অযথা
// app/about/page.jsx
export const dynamic = 'force-dynamic'; // প্রতি রিকোয়েস্টে সার্ভারে রেন্ডার!

export default async function About() {
    const team = await fetch('https://api.example.com/team').then(r => r.json());
    // টিম তথ্য মাসে একবার বদলায় — SSR এখানে অপচয়!
    return <TeamPage members={team} />;
}
```

```jsx
// ✅ সঠিক: SSG ব্যবহার করুন — বিল্ড টাইমে তৈরি, CDN থেকে সার্ভ
// app/about/page.jsx (কোনো dynamic ফ্ল্যাগ নেই — ডিফল্ট SSG)

export default async function About() {
    const team = await fetch('https://api.example.com/team').then(r => r.json());
    return <TeamPage members={team} />;
}
// SSG: বিল্ড টাইমে একবার HTML তৈরি হয় → CDN-এ ক্যাশ → ⚡ তাৎক্ষণিক!
```

### ❌ অ্যান্টি-প্যাটার্ন ৩: SSG দিয়ে রিয়েল-টাইম ডেটা দেখানোর চেষ্টা

```jsx
// ❌ ভুল: bKash ব্যালেন্স SSG দিয়ে দেখানো
// বিল্ড টাইমের পুরনো ব্যালেন্স দেখাবে!
export default async function Balance() {
    const balance = await fetch('https://api.bkash.com/balance').then(r => r.json());
    // এই ব্যালেন্স বিল্ড টাইমের — বর্তমান ব্যালেন্স নয়!
    return <p>ব্যালেন্স: ৳{balance.amount}</p>;
}
```

```jsx
// ✅ সঠিক: CSR ব্যবহার করুন — রিয়েল-টাইম ব্যালেন্স
'use client';
import { useState, useEffect } from 'react';

export default function Balance() {
    const [balance, setBalance] = useState(null);

    useEffect(() => {
        // ব্রাউজারে API কল — সবসময় ফ্রেশ ডেটা
        fetch('/api/balance', {
            headers: { Authorization: `Bearer ${getToken()}` }
        })
            .then(r => r.json())
            .then(setBalance);
    }, []);

    return <p>ব্যালেন্স: ৳{balance?.amount ?? 'লোড হচ্ছে...'}</p>;
}
```

### ❌ অ্যান্টি-প্যাটার্ন ৪: বিশাল JS বান্ডেল দিয়ে CSR

```javascript
// ❌ ভুল: সব কিছু একটি বান্ডেলে — ধীর লোড
import moment from 'moment';          // 67KB
import lodash from 'lodash';           // 72KB
import Chart from 'chart.js/auto';     // 200KB
import FullCalendar from '@fullcalendar/react'; // 50KB
// মোট বান্ডেল: 2MB+ — ব্যবহারকারী ৫-১০ সেকেন্ড অপেক্ষা করে!
```

```javascript
// ✅ সঠিক: কোড স্প্লিটিং ও লেজি লোডিং
import { lazy, Suspense } from 'react';

// চাহিদা অনুযায়ী লোড — শুরুতে সব আনতে হয় না
const Chart = lazy(() => import('./components/Chart'));
const Calendar = lazy(() => import('./components/Calendar'));

// date-fns ব্যবহার করুন moment-এর বদলে (2KB vs 67KB)
import { format } from 'date-fns';
import { bn } from 'date-fns/locale';

function Dashboard() {
    return (
        <div>
            <h1>{format(new Date(), 'dd MMMM yyyy', { locale: bn })}</h1>
            <Suspense fallback={<p>চার্ট লোড হচ্ছে...</p>}>
                <Chart data={chartData} />
            </Suspense>
        </div>
    );
}
```

### ❌ অ্যান্টি-প্যাটার্ন ৫: হাইড্রেশন মিসম্যাচ

```jsx
// ❌ ভুল: সার্ভার ও ক্লায়েন্টে আলাদা আউটপুট — হাইড্রেশন এরর!
export default function TimeDisplay() {
    return <p>বর্তমান সময়: {new Date().toLocaleTimeString('bn-BD')}</p>;
    // সার্ভারে: "বিকাল ৩:৪৫:১২"
    // ক্লায়েন্টে: "বিকাল ৩:৪৫:১৫" (৩ সেকেন্ড পরে!)
    // React হাইড্রেশন এরর দেবে!
}
```

```jsx
// ✅ সঠিক: ক্লায়েন্ট-নির্দিষ্ট কন্টেন্ট useEffect-এ রাখুন
'use client';
import { useState, useEffect } from 'react';

export default function TimeDisplay() {
    const [time, setTime] = useState('');

    useEffect(() => {
        // শুধু ব্রাউজারে চলবে — হাইড্রেশন মিসম্যাচ হবে না
        const timer = setInterval(() => {
            setTime(new Date().toLocaleTimeString('bn-BD'));
        }, 1000);
        return () => clearInterval(timer);
    }, []);

    return <p>বর্তমান সময়: {time || 'লোড হচ্ছে...'}</p>;
}
```

---

## 📊 সারসংক্ষেপ — চারটি প্যাটার্নের তুলনামূলক ASCII ডায়াগ্রাম

```
  ╔═══════════════════════════════════════════════════════════════════════════╗
  ║                    রেন্ডারিং প্যাটার্ন সারসংক্ষেপ                        ║
  ╠═══════════════════════════════════════════════════════════════════════════╣
  ║                                                                         ║
  ║  CSR:   ক্লায়েন্ট ────▶  [খালি HTML] ──▶ [JS ডাউনলোড] ──▶ [রেন্ডার]    ║
  ║         🖥️ ব্রাউজার সব কাজ করে                                         ║
  ║                                                                         ║
  ║  SSR:   সার্ভার ─────▶ [ডেটা ফেচ + HTML তৈরি] ──▶ [পূর্ণ HTML পাঠানো]  ║
  ║         🖧 প্রতি রিকোয়েস্টে সার্ভার কাজ করে                             ║
  ║                                                                         ║
  ║  SSG:   বিল্ড ──────▶ [সব HTML আগেই তৈরি] ──▶ [CDN থেকে সার্ভ]        ║
  ║         🏗️ ডিপ্লয়ের আগেই সব প্রস্তুত                                    ║
  ║                                                                         ║
  ║  ISR:   বিল্ড + ──▶ [HTML তৈরি] ──▶ [ক্যাশ] ──▶ [ব্যাকগ্রাউন্ডে আপডেট] ║
  ║         🔄 SSG + স্মার্ট রিভ্যালিডেশন                                     ║
  ║                                                                         ║
  ╠═══════════════════════════════════════════════════════════════════════════╣
  ║                                                                         ║
  ║  গতি (FCP):     SSG ⚡ > ISR ⚡ > SSR 🚀 > CSR 🐌                       ║
  ║  SEO:           SSG ✅ = ISR ✅ = SSR ✅ > CSR ❌                        ║
  ║  ডেটা ফ্রেশনেস:  CSR ✅ = SSR ✅ > ISR 🔶 > SSG ❌                      ║
  ║  সার্ভার খরচ:    SSG 💰 < ISR 💰 < CSR 💰💰 < SSR 💰💰💰               ║
  ║  জটিলতা:        SSG ✅ < CSR 🔶 < ISR 🔶 < SSR ❌                      ║
  ║                                                                         ║
  ╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 🔗 সম্পর্কিত প্যাটার্নস

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| **MVC** | Laravel Blade SSR হলো MVC প্যাটার্নের View অংশ |
| **Microservices** | SSR/ISR সার্ভারগুলো মাইক্রোসার্ভিস হিসেবে ডিপ্লয় করা যায় |
| **CQRS** | Read (SSG/ISR) ও Write (API) আলাদা রাখা CQRS-এর ধারণা |
| **Event-Driven** | On-demand ISR revalidation ইভেন্ট-ড্রিভেন প্যাটার্ন অনুসরণ করে |
| **Repository Pattern** | ডেটা ফেচিং লেয়ার সব রেন্ডারিং প্যাটার্নেই ব্যবহৃত হয় |
| **Serverless** | SSG/ISR সাইটগুলো serverless Edge Functions-এ চমৎকার কাজ করে |

---

## 🎯 চূড়ান্ত পরামর্শ

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  ১. "ডিফল্ট SSG, প্রয়োজনে SSR" — এই নীতি অনুসরণ করুন             │
  │                                                                     │
  │  ২. বাংলাদেশের প্রেক্ষাপটে ইন্টারনেট গতি কম —                       │
  │     তাই ছোট JS বান্ডেল ও দ্রুত FCP অত্যন্ত গুরুত্বপূর্ণ            │
  │                                                                     │
  │  ৩. প্রতিটি পেজের জন্য আলাদাভাবে রেন্ডারিং প্যাটার্ন বেছে নিন     │
  │     — পুরো সাইটে একটি প্যাটার্ন ব্যবহার করবেন না                    │
  │                                                                     │
  │  ৪. হাইব্রিড অ্যাপ্রোচ (Next.js App Router) সেরা ফলাফল দেয়         │
  │                                                                     │
  │  ৫. PHP ডেভেলপারদের জন্য: Blade → Livewire → Inertia.js             │
  │     — এই ক্রমে শিখুন, প্রয়োজন অনুযায়ী বেছে নিন                    │
  │                                                                     │
  │  ৬. মনে রাখুন: সঠিক রেন্ডারিং প্যাটার্ন নির্বাচন আপনার সাইটের      │
  │     SEO, পারফরম্যান্স, এবং ব্যবহারকারীর অভিজ্ঞতা — তিনটিকেই        │
  │     সরাসরি প্রভাবিত করে।                                            │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

*📚 এই ডকুমেন্টটি বাংলাদেশি ডেভেলপারদের জন্য তৈরি — বাস্তব প্রজেক্ট ও পরিচিত ব্র্যান্ডের উদাহরণ দিয়ে ওয়েব রেন্ডারিং প্যাটার্নের সম্পূর্ণ গাইড।*
