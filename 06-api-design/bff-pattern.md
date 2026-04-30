# 🎭 Backend-for-Frontend (BFF) Pattern — ক্লায়েন্ট-নির্দিষ্ট API লেয়ার

## 📖 সংজ্ঞা ও মূল ধারণা

**Backend-for-Frontend (BFF)** হলো একটি API আর্কিটেকচার প্যাটার্ন যেখানে প্রতিটি ক্লায়েন্ট (Web, Mobile, IoT, Smart TV) এর জন্য আলাদা একটি API লেয়ার তৈরি করা হয়, যা সেই ক্লায়েন্টের প্রয়োজন অনুযায়ী backend microservices থেকে ডেটা aggregate, transform ও orchestrate করে। এটি Sam Newman ও SoundCloud এর Phil Calçado কর্তৃক ২০১৫ সালে জনপ্রিয় হয়।

**মূল ধারণা:** "One backend per user experience" — অর্থাৎ প্রতিটি UX-এর জন্য একটি optimized API।

```
Traditional Single API:
                    ┌──────────────────┐
   Web Client ──┐   │                  │
   Mobile App ──┼──►│  Generic API    │──► Microservices
   Smart TV  ──┘   │ (One-size-fits-all)│
                    └──────────────────┘
   সমস্যা: Mobile এর কম bandwidth, কিন্তু same JSON; Web এর বেশি data দরকার।

BFF Pattern:
   Web Client ──► Web BFF    ─┐
   Mobile App ──► Mobile BFF ─┼──► Microservices (Product, Order, User, Cart)
   Smart TV  ──► TV BFF     ─┘
   প্রতিটি BFF নিজের ক্লায়েন্টের জন্য optimized।
```

---

## 🎯 কেন BFF? — সমস্যা যা সমাধান করে

### সমস্যা ১: ভিন্ন ক্লায়েন্ট, ভিন্ন প্রয়োজন
Daraz Web এ একটি product page-এ ৫০+ field দেখানো হয় (full description, reviews, related products, seller info, shipping calculator)। Daraz Mobile App-এ same product page-এ মাত্র ১৫টি field দরকার (ছোট স্ক্রিন, কম bandwidth, faster load)। একই API দিয়ে পরিবেশন করলে—

- **Mobile:** অতিরিক্ত data ডাউনলোড → slow, বেশি data খরচ (গ্রামে 2G/3G)
- **Web:** কম data → frontend-এ multiple API call → waterfall latency

### সমস্যা ২: Frontend ↔ Backend coupling
Frontend team নতুন ফিচার আনতে চায়, কিন্তু backend team busy। প্রতিটা UI পরিবর্তনে backend API change দরকার পড়ে। BFF এই coupling ভাঙে — frontend team তাদের নিজের BFF maintain করতে পারে।

### সমস্যা ৩: Multiple round-trips (chatty API)
Pathao Foods এর order detail page লোড করতে দরকার:
1. `GET /orders/{id}` (order info)
2. `GET /restaurants/{id}` (restaurant details)
3. `GET /users/{id}` (rider info)
4. `GET /payments/{orderId}` (payment status)

Mobile-এ ৪টা separate API call = 4× latency। BFF একটাই endpoint দেয় যা parallel ভাবে aggregate করে।

### সমস্যা ৪: Authentication/authorization complexity
প্রতিটা microservice এ token verify করার পরিবর্তে BFF একবার verify করে, internal service-এ propagate করে।

---

## 🏗️ আর্কিটেকচার Variants

### Variant 1: BFF per Client Type
```
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ Web BFF  │    │Mobile BFF│    │Partner BFF│
   └────┬─────┘    └────┬─────┘    └────┬─────┘
        └───────────────┼───────────────┘
                        ▼
              ┌─────────────────┐
              │  Microservices  │
              │ (Product, Cart, │
              │  Order, User)   │
              └─────────────────┘
```

### Variant 2: BFF per Team (Squad-owned)
Spotify-style — প্রতিটি squad তাদের নিজের BFF maintain করে।

### Variant 3: BFF + GraphQL Gateway hybrid
BFF এর ভেতরেই GraphQL ব্যবহার, যেন frontend নিজে query shape করতে পারে।

### BFF বনাম API Gateway তুলনা

| বৈশিষ্ট্য | API Gateway | BFF |
|----------|-------------|-----|
| উদ্দেশ্য | Routing, auth, rate limit, cross-cutting concerns | Client-specific data shaping |
| Logic | Generic, thin | Business logic + aggregation |
| Owner | Platform team | Frontend team |
| Per-client | একটাই Gateway | প্রতিটা client-এ আলাদা |
| Coupling | Backend-driven | Frontend-driven |
| Use case | Public API, rate limit | Internal product team |

> **Best practice:** API Gateway + BFF একসাথে ব্যবহার — Gateway হ্যান্ডল করে edge concerns (TLS, WAF, rate limit), BFF হ্যান্ডল করে data composition।

---

## 🔬 Deep Dive — BFF এর Responsibilities

### 1. **Aggregation (Fan-out)**
একাধিক service থেকে parallel fetch করে একটি response বানানো।

### 2. **Transformation**
- Backend response থেকে ফিল্ড filter (over-fetching reduce)
- ফিল্ড rename (snake_case → camelCase for JS)
- Type conversion (timestamp → human-readable)
- Localization (BDT format, বাংলা translation)

### 3. **Orchestration**
Sequential service call — যেমন: cart valid কিনা চেক → stock check → price calc → checkout।

### 4. **Caching**
Per-client cache — Mobile BFF এর response cache key আলাদা। Redis দিয়ে aggressive caching।

### 5. **Authentication Offload**
JWT verify একবার BFF-তে, তারপর internal mTLS দিয়ে microservices এ pass।

### 6. **Resilience patterns**
Circuit breaker, retry, fallback (e.g., recommendations service down হলে empty array return)।

---

## 💻 Node.js (Express) BFF — Daraz Mobile BFF

```javascript
// daraz-mobile-bff/src/server.js
import express from 'express';
import axios from 'axios';
import CircuitBreaker from 'opossum';
import { LRUCache } from 'lru-cache';

const app = express();
app.use(express.json());

// Service URLs (internal)
const SERVICES = {
  product: 'http://product-service:8080',
  inventory: 'http://inventory-service:8080',
  review: 'http://review-service:8080',
  seller: 'http://seller-service:8080',
  recommendation: 'http://reco-service:8080',
};

// In-memory cache (production-এ Redis)
const cache = new LRUCache({ max: 1000, ttl: 60_000 });

// Circuit breaker for non-critical service
const recoBreaker = new CircuitBreaker(
  async (productId) => axios.get(`${SERVICES.recommendation}/related/${productId}`),
  { timeout: 500, errorThresholdPercentage: 50, resetTimeout: 10_000 }
);
recoBreaker.fallback(() => ({ data: { items: [] } })); // fallback empty

// JWT verification middleware
import jwt from 'jsonwebtoken';
function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (e) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// 🎯 Mobile-optimized product detail endpoint
app.get('/mobile/v1/products/:id', authenticate, async (req, res) => {
  const { id } = req.params;
  const cacheKey = `mobile:product:${id}:${req.user.lang || 'bn'}`;

  // Cache check
  const cached = cache.get(cacheKey);
  if (cached) return res.json(cached);

  try {
    // Parallel fetch — fan-out
    const [productRes, inventoryRes, reviewRes, recoRes] = await Promise.allSettled([
      axios.get(`${SERVICES.product}/products/${id}`),
      axios.get(`${SERVICES.inventory}/stock/${id}`),
      axios.get(`${SERVICES.review}/products/${id}/summary`),
      recoBreaker.fire(id), // fallback enabled
    ]);

    if (productRes.status === 'rejected') {
      return res.status(502).json({ error: 'Product service unavailable' });
    }

    const product = productRes.value.data;
    const inventory = inventoryRes.status === 'fulfilled' ? inventoryRes.value.data : null;
    const reviews = reviewRes.status === 'fulfilled' ? reviewRes.value.data : { avg: 0, count: 0 };
    const reco = recoRes.status === 'fulfilled' ? recoRes.value.data : { items: [] };

    // 📱 Mobile-specific transformation
    const mobileResponse = {
      id: product.id,
      title: product.title_bn || product.title, // localized
      price: {
        amount: product.price,
        formatted: `৳${product.price.toLocaleString('bn-BD')}`, // BDT format
        original: product.original_price,
        discount: product.discount_pct,
      },
      // Mobile-এ ১টা thumbnail যথেষ্ট
      thumbnail: product.images?.[0]?.url_small,
      // 5টার বেশি image না, lazy load
      images: (product.images || []).slice(0, 5).map(i => i.url_medium),
      stock: inventory?.available > 0 ? 'in_stock' : 'out_of_stock',
      stockCount: inventory?.available > 10 ? '10+' : inventory?.available || 0,
      rating: {
        avg: Math.round(reviews.avg * 10) / 10,
        count: reviews.count,
      },
      // Description মোবাইলে summary
      summary: product.description?.substring(0, 200) + '...',
      // Recommendations max 6
      relatedProducts: (reco.items || []).slice(0, 6).map(r => ({
        id: r.id,
        title: r.title,
        price: r.price,
        thumbnail: r.thumbnail_url,
      })),
      // Mobile-specific: cash on delivery check
      cod: product.cod_eligible && req.user.district !== 'remote',
    };

    cache.set(cacheKey, mobileResponse);
    res.json(mobileResponse);
  } catch (err) {
    console.error('BFF aggregation error', err);
    res.status(500).json({ error: 'Internal error' });
  }
});

// 🛒 Mobile cart checkout — orchestration
app.post('/mobile/v1/checkout', authenticate, async (req, res) => {
  const { cartId, paymentMethod, addressId } = req.body;

  try {
    // Sequential orchestration
    const cart = await axios.get(`${SERVICES.product}/carts/${cartId}`).then(r => r.data);
    if (cart.userId !== req.user.id) return res.status(403).json({ error: 'Forbidden' });

    // Stock validation (parallel)
    const stockChecks = await Promise.all(
      cart.items.map(item =>
        axios.get(`${SERVICES.inventory}/stock/${item.productId}`).then(r => r.data)
      )
    );
    const outOfStock = cart.items.filter((it, i) => stockChecks[i].available < it.qty);
    if (outOfStock.length) {
      return res.status(409).json({ error: 'Out of stock', items: outOfStock });
    }

    // Create order
    const order = await axios.post(`${SERVICES.product}/orders`, {
      userId: req.user.id, cartId, addressId, paymentMethod,
    }).then(r => r.data);

    // Mobile-specific: deep link for bKash payment
    if (paymentMethod === 'bkash') {
      const bkashUrl = `bkash://pay?orderId=${order.id}&amount=${order.total}`;
      return res.json({ orderId: order.id, deepLink: bkashUrl, fallbackUrl: order.paymentUrl });
    }

    res.json({ orderId: order.id, status: order.status });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => console.log('Mobile BFF on :3000'));
```

### TypeScript + Fastify + DataLoader

```typescript
// daraz-web-bff/src/server.ts
import Fastify from 'fastify';
import DataLoader from 'dataloader';
import axios from 'axios';

const fastify = Fastify({ logger: true });

// DataLoader prevents N+1 in batched fetches
const sellerLoader = new DataLoader<string, Seller>(async (sellerIds) => {
  const { data } = await axios.post('http://seller-service/batch', { ids: sellerIds });
  const map = new Map(data.sellers.map((s: Seller) => [s.id, s]));
  return sellerIds.map(id => map.get(id) ?? null);
});

interface Seller { id: string; name: string; rating: number; }

fastify.get('/web/v1/home-feed', async (req, reply) => {
  const products = await axios.get('http://product-service/featured').then(r => r.data);

  // Web BFF: rich data, parallel seller load via DataLoader
  const enriched = await Promise.all(products.map(async (p: any) => ({
    ...p,
    seller: await sellerLoader.load(p.sellerId),
    breadcrumb: p.categoryPath.map((c: any) => ({ id: c.id, name: c.name })),
  })));

  return { products: enriched, totalCount: products.length };
});

fastify.listen({ port: 3001 });
```

---

## 🐘 PHP (Laravel) BFF — bKash Merchant Mobile BFF

```php
// app/Http/Controllers/MerchantBffController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class MerchantBffController extends Controller
{
    private array $services = [
        'transaction' => 'http://txn-service.internal',
        'merchant'    => 'http://merchant-service.internal',
        'settlement'  => 'http://settlement-service.internal',
        'kyc'         => 'http://kyc-service.internal',
    ];

    /**
     * Mobile dashboard endpoint — aggregates 4 services
     */
    public function mobileDashboard(Request $request)
    {
        $merchantId = $request->user()->merchant_id;
        $cacheKey   = "bff:mobile:dashboard:{$merchantId}";

        return Cache::remember($cacheKey, 30, function () use ($merchantId) {
            // Laravel HTTP pool — parallel calls
            $responses = Http::pool(fn ($pool) => [
                $pool->as('merchant')
                     ->timeout(2)
                     ->get("{$this->services['merchant']}/merchants/{$merchantId}"),
                $pool->as('today_txn')
                     ->timeout(2)
                     ->get("{$this->services['transaction']}/merchants/{$merchantId}/today"),
                $pool->as('settlement')
                     ->timeout(2)
                     ->get("{$this->services['settlement']}/merchants/{$merchantId}/pending"),
                $pool->as('kyc')
                     ->timeout(1)
                     ->get("{$this->services['kyc']}/status/{$merchantId}"),
            ]);

            // Critical service down → fail fast
            if ($responses['merchant']->failed()) {
                abort(502, 'Merchant service unavailable');
            }

            $merchant = $responses['merchant']->json();
            $todayTxn = $responses['today_txn']->successful()
                        ? $responses['today_txn']->json()
                        : ['count' => 0, 'amount' => 0]; // fallback

            $settlement = $responses['settlement']->successful()
                          ? $responses['settlement']->json()
                          : ['pending_amount' => 0];

            $kyc = $responses['kyc']->successful()
                   ? $responses['kyc']->json()
                   : ['status' => 'unknown'];

            // 📱 Mobile-specific shape — minimal, formatted
            return [
                'merchant_name' => $merchant['business_name'],
                'mobile'        => $this->maskMobile($merchant['mobile']), // 017****8901
                'today' => [
                    'transactions' => $todayTxn['count'],
                    'amount'       => '৳' . number_format($todayTxn['amount'], 2),
                    'amount_raw'   => $todayTxn['amount'],
                ],
                'settlement' => [
                    'pending'     => '৳' . number_format($settlement['pending_amount'], 2),
                    'next_date'   => $settlement['next_settlement_date'] ?? null,
                ],
                'kyc_status'   => $kyc['status'],
                'kyc_action'   => $kyc['status'] === 'pending'
                                  ? ['label' => 'KYC সম্পন্ন করুন', 'url' => '/kyc']
                                  : null,
                // Mobile-specific quick actions
                'quick_actions' => [
                    ['icon' => 'qr',        'label' => 'QR দেখান',    'route' => 'qr.show'],
                    ['icon' => 'send',      'label' => 'টাকা পাঠান',   'route' => 'send'],
                    ['icon' => 'history',   'label' => 'লেনদেন',      'route' => 'txn.list'],
                ],
            ];
        });
    }

    /**
     * Web Dashboard — different shape, more data
     */
    public function webDashboard(Request $request)
    {
        $merchantId = $request->user()->merchant_id;

        // Web-এ আরও বিস্তারিত: 30-day chart, top products, multi-branch
        $responses = Http::pool(fn ($pool) => [
            $pool->as('merchant')->get("{$this->services['merchant']}/merchants/{$merchantId}"),
            $pool->as('chart')->get("{$this->services['transaction']}/merchants/{$merchantId}/chart?days=30"),
            $pool->as('top_branches')->get("{$this->services['transaction']}/merchants/{$merchantId}/top-branches"),
            $pool->as('disputes')->get("{$this->services['transaction']}/merchants/{$merchantId}/disputes"),
        ]);

        return response()->json([
            'profile'      => $responses['merchant']->json(),
            'chart_data'   => $responses['chart']->json(),
            'top_branches' => $responses['top_branches']->json(),
            'open_disputes'=> $responses['disputes']->json(),
            // Web-এ raw amount, frontend chart library নিজে format করে
        ]);
    }

    private function maskMobile(string $mobile): string
    {
        return substr($mobile, 0, 3) . '****' . substr($mobile, -4);
    }
}
```

```php
// routes/api.php
Route::prefix('bff/mobile')->middleware(['auth:sanctum', 'throttle:60,1'])->group(function () {
    Route::get('/dashboard', [MerchantBffController::class, 'mobileDashboard']);
});

Route::prefix('bff/web')->middleware(['auth:sanctum'])->group(function () {
    Route::get('/dashboard', [MerchantBffController::class, 'webDashboard']);
});
```

---

## ⚖️ Trade-offs

### ✅ সুবিধা
| সুবিধা | ব্যাখ্যা |
|--------|---------|
| Client-optimized | প্রতিটা ক্লায়েন্ট পাবে exact যা দরকার |
| Frontend autonomy | Frontend team তাদের API নিজে evolve করতে পারে |
| Reduced over-fetching | Mobile bandwidth ৭০% পর্যন্ত সাশ্রয় |
| Independent deployment | Mobile BFF deploy করতে web team blocked না |
| Easier auth | JWT একবার verify, internal traffic mTLS |
| Resilience boundary | Service down হলে BFF fallback দিতে পারে |

### ❌ অসুবিধা
| সমস্যা | ব্যাখ্যা |
|--------|---------|
| Code duplication | Web BFF আর Mobile BFF এ similar logic |
| More services to maintain | প্রতিটা BFF আলাদা CI/CD, monitoring |
| Team ownership confusion | "এই endpoint কোন BFF-এ?" |
| Latency hop | Client → BFF → Service (extra hop) |
| Versioning complexity | একই microservice change ৩টা BFF-এ propagate |
| Local dev pain | Developer-কে local-এ multiple BFF run করতে হয় |

---

## 🛡️ Resilience Patterns BFF-এ

```javascript
// Bulkhead — isolate failures
import { Bulkhead } from 'cockatiel';
const recoBulkhead = Bulkhead.create(10); // max 10 concurrent calls

// Timeout per service
const productCall = axios.create({ timeout: 2000 });
const recoCall    = axios.create({ timeout: 500 }); // non-critical, low timeout

// Retry with exponential backoff (for transient errors only)
import retry from 'async-retry';
const fetchProduct = (id) => retry(
  async (bail) => {
    const res = await axios.get(`/products/${id}`);
    if (res.status === 404) bail(new Error('Not found')); // don't retry 4xx
    return res.data;
  },
  { retries: 3, factor: 2, minTimeout: 100 }
);
```

---

## ✅ কখন BFF ব্যবহার করবেন

- ৩+ ক্লায়েন্ট টাইপ আছে যাদের API need উল্লেখযোগ্যভাবে ভিন্ন
- Mobile bandwidth/battery optimization জরুরি (BD-তে 2G/3G ইউজার)
- Frontend ও Backend team আলাদা cadence-এ deploy করে
- Microservices architecture, frontend-কে multiple service-এ চক্কর কাটতে হয়
- Public-facing partner API + internal product API ভিন্ন (e.g., bKash partner API vs bKash app)

## ❌ কখন BFF এড়াবেন

- ছোট app, একটাই client (web only) — over-engineering
- Team ছোট, প্রতিটা BFF maintain করার লোক নেই
- API simple CRUD, transformation না-ই বা থাকল
- GraphQL যথেষ্ট হলে — client নিজেই query shape করতে পারে

---

## ⚠️ সাধারণ Pitfalls

1. **BFF এ business logic ঢুকিয়ে ফেলা।** BFF presentation logic-এ থাকবে; core domain logic microservice-এ। নাহলে BFF "monolith in disguise" হয়ে যায়।
2. **No timeout on downstream calls** — একটা slow service পুরো BFF hang করে দেবে।
3. **Synchronous N+1** — `for` loop-এ axios call। সর্বদা `Promise.all` বা DataLoader।
4. **Cache invalidation absence** — BFF cache stale হলে user ভুল price দেখতে পারে।
5. **Auth duplication** — প্রতিটা BFF আলাদা JWT lib version → security drift।
6. **Logging gap** — distributed trace ID propagate না করলে debug অসম্ভব। OpenTelemetry/X-Request-Id mandatory।
7. **Bypass করা** — Mobile team direct microservice call করা শুরু করলে BFF এর উদ্দেশ্য নষ্ট।

---

## 🇧🇩 বাংলাদেশ Real-world Examples

- **Daraz:** Web BFF ও Mobile BFF আলাদা; Mobile-এ image-গুলো WebP format ও ছোট size, Web-এ original। ফেস্টিভ্যাল sale-এ প্রায় ১০× traffic — Mobile BFF aggressive Redis cache (30s TTL)।
- **Pathao:** Rider App, Customer App, Merchant Dashboard — তিনটার BFF ভিন্ন। Rider BFF-এ location update real-time, Customer BFF-এ batched।
- **bKash:** Customer App, Merchant App, Agent App — ৩টা আলাদা BFF। Agent BFF-এ extra cash-in/cash-out flow, Customer BFF-এ simpler send-money UI।
- **Foodpanda BD:** Vendor App-এর জন্য আলাদা BFF — restaurant owner-কে show করে orders, inventory, accept/reject; Customer App-এ different shape।
- **Chaldal:** Web (desktop grocery shopping, lots of filter), Mobile App (quick reorder) — Mobile BFF এ "reorder previous" endpoint single call-এ পুরো cart তৈরি করে দেয়।
- **Shohoz:** Bus ticketing Web vs Mobile — Mobile-এ seat layout simplified SVG, Web-এ interactive।
- **Nagad:** USSD gateway-এর জন্যও আলাদা BFF — text-based, minimal payload।

---

## 📊 Performance Impact (Daraz BD case study, illustrative)

| মেট্রিক | No BFF | With Mobile BFF |
|---------|--------|-----------------|
| Product page payload | 180 KB | 42 KB |
| API calls per page | 6 | 1 |
| TTI (3G network) | 4.8s | 1.9s |
| Backend service load | High | 40% reduced (cache hit) |

---

## 📝 সারসংক্ষেপ

- **BFF = One backend per UX** — প্রতিটি client experience-এর জন্য আলাদা thin API লেয়ার।
- দায়িত্ব: aggregation, transformation, orchestration, auth offload, client-specific cache।
- **API Gateway != BFF** — Gateway edge concerns, BFF data shaping; দুটাই একসাথে চলতে পারে।
- Trade-off: code duplication ও maintenance overhead, কিন্তু client experience ও team autonomy বহুগুণ ভালো।
- Resilience patterns (timeout, circuit breaker, fallback) BFF-এ obligatory।
- BD context-এ low bandwidth ও diverse client (Web/Mobile/USSD) থাকার কারণে BFF বিশেষভাবে কার্যকর।
- Pitfall: BFF-কে business logic-এ ভরাবেন না; presentation layer হিসেবে রাখুন।
