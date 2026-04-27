# 🌐 REST API ডিজাইন — সম্পূর্ণ গভীর গাইড

## 📌 সংজ্ঞা (Definition)

REST (Representational State Transfer) হলো একটি **আর্কিটেকচারাল স্টাইল** যা Roy Fielding তাঁর ২০০০ সালের PhD dissertation-এ প্রথম উপস্থাপন করেন। এটি কোনো প্রোটোকল বা স্ট্যান্ডার্ড নয় — বরং এটি একটি **constraint-based architecture** যা distributed hypermedia system ডিজাইন করার জন্য তৈরি।

REST-এর মূল দর্শন হলো: **ওয়েবের যে বৈশিষ্ট্যগুলো এটিকে সফল করেছে, সেগুলোকে API ডিজাইনেও প্রয়োগ করা।**

### ৬টি আর্কিটেকচারাল Constraint

#### ১. Client-Server (ক্লায়েন্ট-সার্ভার পৃথকীকরণ)
ক্লায়েন্ট এবং সার্ভারের দায়িত্ব সম্পূর্ণ আলাদা। ক্লায়েন্ট UI/UX সামলায়, সার্ভার data storage ও business logic সামলায়। এই separation-এর ফলে দুটো পক্ষ স্বাধীনভাবে evolve করতে পারে। উদাহরণস্বরূপ, bKash তাদের Android app সম্পূর্ণ পাল্টে ফেলতে পারে backend API কোনো পরিবর্তন ছাড়াই।

#### ২. Stateless (স্টেটলেস)
প্রতিটি request-এ সার্ভারের কাজ করার জন্য প্রয়োজনীয় **সকল তথ্য** থাকতে হবে। সার্ভার কোনো client session store করে না। Authentication token, pagination cursor — সব কিছু request-এর সাথে পাঠাতে হয়।

```
// ❌ Stateful — সার্ভার session মনে রাখে
GET /api/next-page
// সার্ভার মনে রাখছে কোন পেজে আছো

// ✅ Stateless — সব তথ্য request-এ আছে
GET /api/products?page=3&per_page=20
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

#### ৩. Cacheable (ক্যাশযোগ্য)
প্রতিটি response-কে অবশ্যই cacheable বা non-cacheable হিসেবে চিহ্নিত করতে হবে। সঠিক caching নেটওয়ার্ক efficiency বাড়ায় এবং সার্ভার load কমায়।

#### ৪. Uniform Interface (একক ইন্টারফেস)
এটি REST-এর সবচেয়ে গুরুত্বপূর্ণ constraint। চারটি sub-constraint আছে:
- **Resource Identification** — প্রতিটি resource-এর unique URI আছে
- **Resource Manipulation through Representations** — JSON/XML representation দিয়ে resource পরিবর্তন
- **Self-descriptive Messages** — প্রতিটি message-এ নিজের processing-এর জন্য পর্যাপ্ত তথ্য থাকে
- **HATEOAS** — Hypermedia as the Engine of Application State

#### ৫. Layered System (স্তরভিত্তিক)
ক্লায়েন্ট জানে না সে সরাসরি সার্ভারের সাথে যুক্ত নাকি মাঝখানে load balancer, CDN, বা API gateway আছে। প্রতিটি layer শুধু তার adjacent layer-কে দেখে।

#### ৬. Code-on-Demand (ঐচ্ছিক)
সার্ভার ক্লায়েন্টকে executable code (JavaScript) পাঠাতে পারে। এটি একমাত্র optional constraint।

---

## 📊 REST Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     REST Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────────────────┐   │
│  │  Mobile   │     │   Web    │     │   Third-party        │   │
│  │  App      │     │  Client  │     │   Service            │   │
│  │ (bKash)   │     │ (React)  │     │   (Payment Gateway)  │   │
│  └────┬─────┘     └────┬─────┘     └──────────┬───────────┘   │
│       │                │                       │               │
│       └────────────────┼───────────────────────┘               │
│                        │                                       │
│                        ▼                                       │
│            ┌──────────────────────┐                            │
│            │    API Gateway /     │   ← Rate Limiting          │
│            │    Load Balancer     │   ← Authentication         │
│            │    (Nginx/Kong)      │   ← Request Routing        │
│            └──────────┬───────────┘                            │
│                       │                                        │
│            ┌──────────▼───────────┐                            │
│            │   Cache Layer        │                            │
│            │   (Redis/Varnish)    │   ← ETag, Cache-Control    │
│            └──────────┬───────────┘                            │
│                       │                                        │
│       ┌───────────────┼───────────────────┐                    │
│       ▼               ▼                   ▼                    │
│  ┌─────────┐   ┌───────────┐   ┌──────────────┐              │
│  │ Service  │   │  Service  │   │   Service    │              │
│  │  (Auth)  │   │ (Products)│   │  (Orders)    │              │
│  │ Laravel  │   │  Express  │   │   Laravel    │              │
│  └────┬─────┘   └─────┬─────┘   └──────┬──────┘              │
│       │               │                │                       │
│       └───────────────┼────────────────┘                       │
│                       ▼                                        │
│            ┌──────────────────────┐                            │
│            │     Database         │                            │
│            │  (MySQL/PostgreSQL)  │                            │
│            └──────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Request-Response Flow

```
Client                    Server
  │                         │
  │  GET /api/v1/products   │
  │  Accept: application/json
  │  Authorization: Bearer xxx
  │ ───────────────────────►│
  │                         │  ← Route Matching
  │                         │  ← Middleware Pipeline
  │                         │  ← Controller Logic
  │                         │  ← Resource Transform
  │  200 OK                 │
  │  Content-Type: json     │
  │  ETag: "abc123"         │
  │  Cache-Control: max-age │
  │◄─────────────────────── │
  │                         │
```

---

## 💻 REST Principles — গভীর আলোচনা

### ১. Resource Naming (রিসোর্স নামকরণ)

REST-এ সবকিছুই **resource**। প্রতিটি resource একটি **noun** দিয়ে চিহ্নিত হয়, verb দিয়ে নয়। URI হলো resource-এর ঠিকানা।

#### মূল নিয়মাবলী

```
✅ সঠিক (Nouns, Plural)              ❌ ভুল (Verbs)
────────────────────────────         ─────────────────────────
GET  /api/products                   GET  /api/getProducts
POST /api/products                   POST /api/createProduct
GET  /api/products/42                GET  /api/getProductById?id=42
PUT  /api/products/42                POST /api/updateProduct
DELETE /api/products/42              POST /api/deleteProduct
```

#### Nested Resources (সম্পর্কযুক্ত রিসোর্স)

```
# একটি দোকানের সব পণ্য
GET /api/shops/15/products

# একটি পণ্যের সব রিভিউ
GET /api/products/42/reviews

# একটি অর্ডারের সব আইটেম
GET /api/orders/100/items

# bKash: একজন ব্যবহারকারীর সব লেনদেন
GET /api/users/01712345678/transactions
```

#### URI Best Practices

```
✅ lowercase ব্যবহার করুন           → /api/product-categories
❌ camelCase এড়িয়ে চলুন             → /api/productCategories
✅ hyphen ব্যবহার করুন               → /api/order-items
❌ underscore এড়িয়ে চলুন            → /api/order_items
✅ trailing slash এড়িয়ে চলুন        → /api/products
❌ trailing slash দেবেন না           → /api/products/
✅ version prefix দিন                → /api/v1/products
✅ filter query param এ দিন          → /api/products?category=electronics
❌ filter URI তে দেবেন না            → /api/products/category/electronics
```

#### Action-based URI (যখন CRUD ছাড়া কাজ প্রয়োজন)

কিছু ক্ষেত্রে সত্যিকারের action প্রয়োজন:

```
POST /api/orders/100/cancel         ← অর্ডার বাতিল
POST /api/products/42/publish       ← পণ্য প্রকাশ
POST /api/users/5/verify            ← ব্যবহারকারী যাচাই
POST /api/payments/200/refund       ← bKash রিফান্ড
```

---

### ২. HTTP Methods (HTTP মেথড সমূহ)

প্রতিটি HTTP method-এর নির্দিষ্ট semantic meaning আছে। সঠিক method ব্যবহার না করলে API-এর predictability নষ্ট হয়।

| Method | উদ্দেশ্য | Request Body | Idempotent | Safe | Cacheable |
|--------|----------|-------------|------------|------|-----------|
| `GET` | রিসোর্স পড়া | ❌ নেই | ✅ হ্যাঁ | ✅ হ্যাঁ | ✅ হ্যাঁ |
| `POST` | নতুন রিসোর্স তৈরি | ✅ আছে | ❌ না | ❌ না | ❌ না |
| `PUT` | সম্পূর্ণ রিসোর্স প্রতিস্থাপন | ✅ আছে | ✅ হ্যাঁ | ❌ না | ❌ না |
| `PATCH` | আংশিক আপডেট | ✅ আছে | ❌ না* | ❌ না | ❌ না |
| `DELETE` | রিসোর্স মুছে ফেলা | ❌ নেই | ✅ হ্যাঁ | ❌ না | ❌ না |
| `HEAD` | শুধু headers (body ছাড়া) | ❌ নেই | ✅ হ্যাঁ | ✅ হ্যাঁ | ✅ হ্যাঁ |
| `OPTIONS` | supported methods জানা | ❌ নেই | ✅ হ্যাঁ | ✅ হ্যাঁ | ❌ না |

> **Idempotent** মানে একই request ১০০ বার পাঠালেও ফলাফল একই থাকবে। `PUT /products/42` বারবার পাঠালে product-42 একই থাকবে। কিন্তু `POST /products` বারবার পাঠালে প্রতিবার নতুন product তৈরি হবে।

#### PUT vs PATCH — গুরুত্বপূর্ণ পার্থক্য

```json
// আসল resource
{
  "id": 42,
  "name": "Samsung Galaxy S24",
  "price": 89999,
  "stock": 150,
  "category": "smartphones"
}

// PUT — সম্পূর্ণ resource পাঠাতে হবে (না পাঠালে null হয়ে যাবে)
PUT /api/products/42
{
  "name": "Samsung Galaxy S24 Ultra",
  "price": 129999,
  "stock": 150,
  "category": "smartphones"
}

// PATCH — শুধু যা পরিবর্তন করতে চান
PATCH /api/products/42
{
  "price": 129999
}
```

---

### ৩. Status Codes (স্ট্যাটাস কোড সমূহ)

HTTP status code ক্লায়েন্টকে জানায় request-এর ফলাফল কী হয়েছে। সঠিক code ব্যবহার client-side error handling সহজ করে।

#### 2xx — Success (সফল)

| Code | নাম | কখন ব্যবহার করবেন |
|------|------|-------------------|
| `200` | OK | GET সফল, PUT/PATCH আপডেট সফল |
| `201` | Created | POST-এ নতুন resource তৈরি হয়েছে |
| `202` | Accepted | Request গ্রহণ করা হয়েছে কিন্তু এখনো process হয়নি (async) |
| `204` | No Content | DELETE সফল, response body নেই |

#### 3xx — Redirection (পুনর্নির্দেশ)

| Code | নাম | কখন ব্যবহার করবেন |
|------|------|-------------------|
| `301` | Moved Permanently | Resource স্থায়ীভাবে সরানো হয়েছে |
| `304` | Not Modified | Cache valid আছে, নতুন data পাঠানোর দরকার নেই |

#### 4xx — Client Error (ক্লায়েন্ট ত্রুটি)

| Code | নাম | কখন ব্যবহার করবেন |
|------|------|-------------------|
| `400` | Bad Request | Validation ব্যর্থ, malformed JSON |
| `401` | Unauthorized | Authentication নেই বা ভুল token |
| `403` | Forbidden | Permission নেই (authenticated কিন্তু authorized না) |
| `404` | Not Found | Resource খুঁজে পাওয়া যায়নি |
| `405` | Method Not Allowed | এই URI-তে এই method সমর্থিত নয় |
| `409` | Conflict | Resource conflict (duplicate entry, version mismatch) |
| `413` | Payload Too Large | Request body অতিরিক্ত বড় |
| `415` | Unsupported Media Type | Content-Type সমর্থিত নয় |
| `422` | Unprocessable Entity | Syntactically সঠিক কিন্তু semantically ভুল |
| `429` | Too Many Requests | Rate limit অতিক্রম করা হয়েছে |

#### 5xx — Server Error (সার্ভার ত্রুটি)

| Code | নাম | কখন ব্যবহার করবেন |
|------|------|-------------------|
| `500` | Internal Server Error | সার্ভারে অপ্রত্যাশিত সমস্যা |
| `502` | Bad Gateway | Upstream service-এর সমস্যা |
| `503` | Service Unavailable | সার্ভার সাময়িকভাবে অক্ষম (maintenance) |
| `504` | Gateway Timeout | Upstream service response দিচ্ছে না |

---

### ৪. Request/Response Design

#### Standard JSON Response Structure

সকল API response-এর একটি consistent structure থাকা উচিত:

```json
// ✅ সফল response (Single Resource)
{
  "success": true,
  "data": {
    "id": 42,
    "name": "Grameenphone SIM",
    "price": 200,
    "created_at": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:05Z"
  }
}

// ✅ সফল response (Collection)
{
  "success": true,
  "data": [
    { "id": 1, "name": "Product A" },
    { "id": 2, "name": "Product B" }
  ],
  "meta": {
    "pagination": {
      "total": 245,
      "per_page": 20,
      "current_page": 1,
      "last_page": 13,
      "next_cursor": "eyJpZCI6MjB9"
    }
  },
  "links": {
    "self": "/api/v1/products?page=1",
    "next": "/api/v1/products?page=2",
    "last": "/api/v1/products?page=13"
  }
}

// ❌ Error response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "অনুগ্রহ করে সঠিক তথ্য দিন",
    "details": [
      {
        "field": "price",
        "message": "মূল্য অবশ্যই ০ এর চেয়ে বেশি হতে হবে",
        "rule": "min:1"
      },
      {
        "field": "name",
        "message": "পণ্যের নাম আবশ্যক",
        "rule": "required"
      }
    ]
  },
  "meta": {
    "request_id": "req_def456",
    "timestamp": "2024-01-15T10:30:05Z"
  }
}
```

#### HATEOAS Links (Hypermedia)

```json
{
  "data": {
    "id": 100,
    "status": "pending",
    "total": 5999,
    "customer": "Rahim Uddin"
  },
  "links": {
    "self": { "href": "/api/v1/orders/100" },
    "confirm": { "href": "/api/v1/orders/100/confirm", "method": "POST" },
    "cancel": { "href": "/api/v1/orders/100/cancel", "method": "POST" },
    "items": { "href": "/api/v1/orders/100/items" },
    "customer": { "href": "/api/v1/customers/55" }
  }
}
```

---

### ৫. Pagination (পেজিনেশন)

বড় dataset ফেরত দেওয়ার সময় pagination অপরিহার্য। তিনটি প্রধান পদ্ধতি আছে:

#### পদ্ধতি ১: Offset-based (সহজ, কিন্তু বড় dataset-এ ধীর)

```
GET /api/products?page=5&per_page=20
```

**সমস্যা:** `OFFSET 10000` হলে ডেটাবেস ১০,০০০ row skip করে — অত্যন্ত ধীর।

#### পদ্ধতি ২: Cursor-based (বড় dataset-এর জন্য সর্বোত্তম)

```
GET /api/products?cursor=eyJpZCI6MTAwfQ&limit=20
```

Cursor হলো একটি encoded pointer যা পরবর্তী page-এর শুরু নির্দেশ করে।

#### পদ্ধতি ৩: Keyset (দ্রুততম)

```
GET /api/products?after_id=100&limit=20
-- SQL: WHERE id > 100 ORDER BY id LIMIT 20
```

#### PHP (Laravel) — Cursor Pagination

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $perPage = min($request->input('per_page', 20), 100);

        // Offset-based pagination
        if ($request->has('page')) {
            $products = Product::with('category')
                ->orderBy('created_at', 'desc')
                ->paginate($perPage);

            return ProductResource::collection($products);
        }

        // Cursor-based pagination (বড় dataset-এর জন্য recommended)
        $products = Product::with('category')
            ->orderBy('id')
            ->cursorPaginate($perPage);

        return ProductResource::collection($products)->additional([
            'meta' => [
                'per_page' => $perPage,
                'next_cursor' => $products->nextCursor()?->encode(),
                'prev_cursor' => $products->previousCursor()?->encode(),
                'has_more' => $products->hasMorePages(),
            ],
        ]);
    }
}
```

#### JavaScript (Express) — Cursor Pagination

```javascript
// controllers/productController.js
const Product = require('../models/Product');

const getProducts = async (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const cursor = req.query.cursor;

  let query = {};

  // cursor decode করে last ID বের করা
  if (cursor) {
    const decoded = JSON.parse(
      Buffer.from(cursor, 'base64').toString('utf-8')
    );
    query._id = { $gt: decoded.lastId };
  }

  const products = await Product.find(query)
    .sort({ _id: 1 })
    .limit(limit + 1) // extra 1 টা নিয়ে বুঝবো আরো আছে কিনা
    .lean();

  const hasMore = products.length > limit;
  if (hasMore) products.pop(); // extra টা বাদ দাও

  const nextCursor = hasMore
    ? Buffer.from(
        JSON.stringify({ lastId: products[products.length - 1]._id })
      ).toString('base64')
    : null;

  res.json({
    success: true,
    data: products,
    meta: {
      limit,
      has_more: hasMore,
      next_cursor: nextCursor,
    },
    links: {
      self: `/api/v1/products?limit=${limit}${cursor ? `&cursor=${cursor}` : ''}`,
      next: nextCursor ? `/api/v1/products?limit=${limit}&cursor=${nextCursor}` : null,
    },
  });
};

module.exports = { getProducts };
```

---

### ৬. Filtering, Sorting, Field Selection

#### Query Parameter Convention

```
# ফিল্টারিং
GET /api/products?category=smartphones&price_min=5000&price_max=50000&brand=samsung

# সর্টিং (- prefix মানে descending)
GET /api/products?sort=-price,name

# ফিল্ড সিলেকশন (শুধু প্রয়োজনীয় ফিল্ড)
GET /api/products?fields=id,name,price,thumbnail

# সব একসাথে
GET /api/products?category=smartphones&sort=-price&fields=id,name,price&page=1&per_page=10
```

#### Laravel API Resource — Filtering

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $query = Product::query();

        // ফিল্টারিং
        if ($request->filled('category')) {
            $query->where('category_slug', $request->category);
        }

        if ($request->filled('price_min')) {
            $query->where('price', '>=', (int) $request->price_min);
        }

        if ($request->filled('price_max')) {
            $query->where('price', '<=', (int) $request->price_max);
        }

        if ($request->filled('search')) {
            $query->where('name', 'like', '%' . $request->search . '%');
        }

        // সর্টিং
        if ($request->filled('sort')) {
            foreach (explode(',', $request->sort) as $sortField) {
                $direction = 'asc';
                if (str_starts_with($sortField, '-')) {
                    $direction = 'desc';
                    $sortField = substr($sortField, 1);
                }

                $allowed = ['name', 'price', 'created_at', 'rating'];
                if (in_array($sortField, $allowed)) {
                    $query->orderBy($sortField, $direction);
                }
            }
        }

        // ফিল্ড সিলেকশন
        if ($request->filled('fields')) {
            $allowed = ['id', 'name', 'price', 'stock', 'category_slug', 'thumbnail'];
            $fields = array_intersect(
                explode(',', $request->fields),
                $allowed
            );
            $query->select(array_merge(['id'], $fields));
        }

        return ProductResource::collection(
            $query->paginate($request->input('per_page', 20))
        );
    }
}
```

#### Express — Filtering Middleware

```javascript
// middleware/queryParser.js
const parseFilters = (allowedFilters) => {
  return (req, res, next) => {
    req.filters = {};
    req.sorting = [];
    req.selectedFields = null;

    // ফিল্টারিং
    for (const [key, value] of Object.entries(req.query)) {
      if (allowedFilters.includes(key)) {
        if (key.endsWith('_min') || key.endsWith('_max')) {
          const field = key.replace(/_min$|_max$/, '');
          req.filters[field] = req.filters[field] || {};
          if (key.endsWith('_min')) req.filters[field].$gte = Number(value);
          if (key.endsWith('_max')) req.filters[field].$lte = Number(value);
        } else {
          req.filters[key] = value;
        }
      }
    }

    // সর্টিং: sort=-price,name → { price: -1, name: 1 }
    if (req.query.sort) {
      const sortObj = {};
      req.query.sort.split(',').forEach((field) => {
        if (field.startsWith('-')) {
          sortObj[field.substring(1)] = -1;
        } else {
          sortObj[field] = 1;
        }
      });
      req.sorting = sortObj;
    }

    // ফিল্ড সিলেকশন: fields=id,name,price → "id name price"
    if (req.query.fields) {
      const allowed = ['id', 'name', 'price', 'stock', 'thumbnail'];
      req.selectedFields = req.query.fields
        .split(',')
        .filter((f) => allowed.includes(f))
        .join(' ');
    }

    next();
  };
};

module.exports = { parseFilters };
```

---

### ৭. Content Negotiation

Content Negotiation হলো সেই প্রক্রিয়া যার মাধ্যমে ক্লায়েন্ট এবং সার্ভার response-এর format নির্ধারণ করে।

```
# ক্লায়েন্ট JSON চায়
GET /api/products/42
Accept: application/json

# ক্লায়েন্ট XML চায়
GET /api/products/42
Accept: application/xml

# ক্লায়েন্ট specific version চায়
GET /api/products/42
Accept: application/vnd.myapp.v2+json

# Request body-র format জানাচ্ছে
POST /api/products
Content-Type: application/json
```

```php
// Laravel — Content Negotiation
public function show(Request $request, Product $product)
{
    if ($request->wantsJson()) {
        return new ProductResource($product);
    }

    if ($request->accepts('application/xml')) {
        return response($product->toXml(), 200)
            ->header('Content-Type', 'application/xml');
    }

    return response()->json(['error' => 'Not Acceptable'], 406);
}
```

---

### ৮. HATEOAS — Hypermedia as the Engine of Application State

HATEOAS হলো REST-এর সর্বোচ্চ পরিপক্বতা স্তর। এতে response-এ সম্ভাব্য পরবর্তী action-এর link থাকে — ক্লায়েন্টকে URI hardcode করতে হয় না।

```json
// bKash Transaction Response — HATEOAS সহ
{
  "data": {
    "transaction_id": "TXN2024011501",
    "type": "send_money",
    "amount": 5000,
    "sender": "01712345678",
    "receiver": "01898765432",
    "status": "completed",
    "created_at": "2024-01-15T14:30:00+06:00"
  },
  "links": {
    "self": {
      "href": "/api/v1/transactions/TXN2024011501",
      "method": "GET"
    },
    "refund": {
      "href": "/api/v1/transactions/TXN2024011501/refund",
      "method": "POST",
      "title": "রিফান্ড করুন"
    },
    "sender_profile": {
      "href": "/api/v1/users/01712345678",
      "method": "GET"
    },
    "statement": {
      "href": "/api/v1/users/01712345678/statements?month=2024-01",
      "method": "GET"
    }
  }
}
```

---

## 🔧 Implementation — সম্পূর্ণ CRUD উদাহরণ

### Laravel API (E-commerce Products)

#### Routes

```php
// routes/api.php
use App\Http\Controllers\Api\V1\ProductController;
use App\Http\Controllers\Api\V1\ProductReviewController;

Route::prefix('v1')->middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::apiResource('products', ProductController::class);
    Route::apiResource('products.reviews', ProductReviewController::class)->shallow();

    Route::post('products/{product}/publish', [ProductController::class, 'publish']);
    Route::post('products/bulk', [ProductController::class, 'bulkStore']);
});
```

#### Form Request (Validation)

```php
<?php

namespace App\Http\Requests\Api\V1;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create-products');
    }

    public function rules(): array
    {
        return [
            'name'        => ['required', 'string', 'max:255'],
            'slug'        => ['required', 'string', 'unique:products,slug'],
            'description' => ['required', 'string', 'min:20'],
            'price'       => ['required', 'integer', 'min:1'], // পয়সায় রাখুন — ৳999.00 → 99900
            'stock'       => ['required', 'integer', 'min:0'],
            'category_id' => ['required', 'exists:categories,id'],
            'images'      => ['array', 'max:10'],
            'images.*'    => ['url'],
            'attributes'  => ['array'],
            'attributes.*.key'   => ['required_with:attributes', 'string'],
            'attributes.*.value' => ['required_with:attributes', 'string'],
        ];
    }

    public function messages(): array
    {
        return [
            'name.required'  => 'পণ্যের নাম আবশ্যক',
            'price.min'      => 'মূল্য ০ এর বেশি হতে হবে',
            'stock.min'      => 'স্টক ঋণাত্মক হতে পারে না',
        ];
    }

    protected function failedValidation(Validator $validator)
    {
        throw new HttpResponseException(
            response()->json([
                'success' => false,
                'error' => [
                    'code'    => 'VALIDATION_ERROR',
                    'message' => 'দেওয়া তথ্য সঠিক নয়',
                    'details' => $validator->errors()->toArray(),
                ],
            ], 422)
        );
    }
}
```

#### API Resource (Response Transformation)

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'          => $this->id,
            'name'        => $this->name,
            'slug'        => $this->slug,
            'description' => $this->when(
                !$request->is('*/products'), // collection-এ description দরকার নেই
                $this->description
            ),
            'price'       => [
                'amount'   => $this->price,
                'currency' => 'BDT',
                'display'  => '৳' . number_format($this->price / 100, 2),
            ],
            'stock'       => $this->stock,
            'is_available' => $this->stock > 0,
            'category'    => new CategoryResource($this->whenLoaded('category')),
            'images'      => $this->images ?? [],
            'rating'      => [
                'average' => round($this->reviews_avg_rating ?? 0, 1),
                'count'   => $this->reviews_count ?? 0,
            ],
            'created_at'  => $this->created_at->toISOString(),
            'updated_at'  => $this->updated_at->toISOString(),

            // HATEOAS links
            'links' => [
                'self'    => ['href' => route('products.show', $this->id)],
                'reviews' => ['href' => route('products.reviews.index', $this->id)],
                'update'  => $this->when(
                    $request->user()?->can('update', $this->resource),
                    ['href' => route('products.update', $this->id), 'method' => 'PUT']
                ),
            ],
        ];
    }
}
```

#### Controller (সম্পূর্ণ CRUD)

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\StoreProductRequest;
use App\Http\Requests\Api\V1\UpdateProductRequest;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class ProductController extends Controller
{
    public function __construct()
    {
        $this->authorizeResource(Product::class, 'product');
    }

    public function index(Request $request): AnonymousResourceCollection
    {
        $query = Product::query()
            ->withCount('reviews')
            ->withAvg('reviews', 'rating');

        // ফিল্টারিং
        $query->when($request->category, fn ($q, $cat) => $q->whereRelation('category', 'slug', $cat))
              ->when($request->price_min, fn ($q, $min) => $q->where('price', '>=', (int) $min))
              ->when($request->price_max, fn ($q, $max) => $q->where('price', '<=', (int) $max))
              ->when($request->search, fn ($q, $s) => $q->where('name', 'like', "%{$s}%"));

        // সর্টিং
        if ($request->filled('sort')) {
            collect(explode(',', $request->sort))->each(function ($field) use ($query) {
                $dir = 'asc';
                if (str_starts_with($field, '-')) {
                    $dir = 'desc';
                    $field = substr($field, 1);
                }
                if (in_array($field, ['name', 'price', 'created_at'])) {
                    $query->orderBy($field, $dir);
                }
            });
        } else {
            $query->latest();
        }

        return ProductResource::collection(
            $query->cursorPaginate(min($request->integer('per_page', 20), 100))
        );
    }

    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = Product::create($request->validated());

        return (new ProductResource($product))
            ->response()
            ->setStatusCode(201)
            ->header('Location', route('products.show', $product));
    }

    public function show(Product $product): ProductResource
    {
        $product->loadCount('reviews')->loadAvg('reviews', 'rating');

        return new ProductResource($product->load('category'));
    }

    public function update(UpdateProductRequest $request, Product $product): ProductResource
    {
        $product->update($request->validated());

        return new ProductResource($product->fresh()->load('category'));
    }

    public function destroy(Product $product): JsonResponse
    {
        $product->delete();

        return response()->json(null, 204);
    }

    public function publish(Product $product): JsonResponse
    {
        if ($product->is_published) {
            return response()->json([
                'success' => false,
                'error' => [
                    'code' => 'ALREADY_PUBLISHED',
                    'message' => 'পণ্যটি ইতোমধ্যে প্রকাশিত',
                ],
            ], 409);
        }

        $product->update([
            'is_published' => true,
            'published_at' => now(),
        ]);

        return response()->json([
            'success' => true,
            'data' => new ProductResource($product),
            'message' => 'পণ্য সফলভাবে প্রকাশিত হয়েছে',
        ]);
    }
}
```

#### Middleware Pipeline

```php
// app/Http/Kernel.php — API middleware group
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    \Illuminate\Routing\Middleware\ThrottleRequests::class.':api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
    \App\Http\Middleware\ForceJsonResponse::class,
    \App\Http\Middleware\LogApiRequest::class,
],

// app/Http/Middleware/ForceJsonResponse.php
class ForceJsonResponse
{
    public function handle(Request $request, Closure $next)
    {
        $request->headers->set('Accept', 'application/json');
        return $next($request);
    }
}
```

---

### Express API (E-commerce Products)

#### Router

```javascript
// routes/v1/products.js
const express = require('express');
const router = express.Router();
const productController = require('../../controllers/v1/productController');
const { authenticate, authorize } = require('../../middleware/auth');
const { validate } = require('../../middleware/validator');
const { productSchema, productUpdateSchema } = require('../../schemas/product');
const { parseFilters } = require('../../middleware/queryParser');
const rateLimiter = require('../../middleware/rateLimiter');

const productFilters = ['category', 'price_min', 'price_max', 'brand', 'search'];

router.get(
  '/',
  rateLimiter({ windowMs: 60000, max: 100 }),
  parseFilters(productFilters),
  productController.index
);

router.get('/:id', productController.show);

router.post(
  '/',
  authenticate,
  authorize('admin', 'seller'),
  validate(productSchema),
  productController.store
);

router.put(
  '/:id',
  authenticate,
  authorize('admin', 'seller'),
  validate(productSchema),
  productController.update
);

router.patch(
  '/:id',
  authenticate,
  authorize('admin', 'seller'),
  validate(productUpdateSchema),
  productController.patch
);

router.delete(
  '/:id',
  authenticate,
  authorize('admin'),
  productController.destroy
);

module.exports = router;
```

#### Joi Validation Schema

```javascript
// schemas/product.js
const Joi = require('joi');

const productSchema = Joi.object({
  name: Joi.string().min(3).max(255).required()
    .messages({ 'string.empty': 'পণ্যের নাম আবশ্যক' }),

  slug: Joi.string().pattern(/^[a-z0-9-]+$/).required(),

  description: Joi.string().min(20).required(),

  price: Joi.number().integer().min(1).required()
    .messages({ 'number.min': 'মূল্য ০ এর বেশি হতে হবে' }),

  stock: Joi.number().integer().min(0).required(),

  category_id: Joi.string().required(),

  images: Joi.array().items(Joi.string().uri()).max(10),

  attributes: Joi.array().items(
    Joi.object({
      key: Joi.string().required(),
      value: Joi.string().required(),
    })
  ),
});

const productUpdateSchema = productSchema.fork(
  ['name', 'slug', 'description', 'price', 'stock', 'category_id'],
  (schema) => schema.optional()
);

module.exports = { productSchema, productUpdateSchema };
```

#### Controller (সম্পূর্ণ CRUD)

```javascript
// controllers/v1/productController.js
const Product = require('../../models/Product');
const { NotFoundError, ConflictError } = require('../../utils/errors');

const index = async (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const cursor = req.query.cursor;

  const query = { ...req.filters, is_published: true };

  if (cursor) {
    const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString());
    query._id = { $gt: decoded.lastId };
  }

  if (req.query.search) {
    query.$text = { $search: req.query.search };
  }

  let findQuery = Product.find(query);

  if (req.selectedFields) {
    findQuery = findQuery.select(req.selectedFields);
  }

  if (Object.keys(req.sorting).length > 0) {
    findQuery = findQuery.sort(req.sorting);
  } else {
    findQuery = findQuery.sort({ _id: 1 });
  }

  const products = await findQuery
    .limit(limit + 1)
    .populate('category', 'name slug')
    .lean();

  const hasMore = products.length > limit;
  if (hasMore) products.pop();

  const nextCursor = hasMore
    ? Buffer.from(JSON.stringify({ lastId: products[products.length - 1]._id })).toString('base64')
    : null;

  res.json({
    success: true,
    data: products.map(formatProduct),
    meta: { limit, has_more: hasMore, next_cursor: nextCursor },
    links: {
      self: `${req.baseUrl}?limit=${limit}`,
      next: nextCursor ? `${req.baseUrl}?limit=${limit}&cursor=${nextCursor}` : null,
    },
  });
};

const show = async (req, res) => {
  const product = await Product.findById(req.params.id)
    .populate('category', 'name slug')
    .lean();

  if (!product) throw new NotFoundError('পণ্যটি খুঁজে পাওয়া যায়নি');

  res.json({
    success: true,
    data: formatProduct(product),
    links: {
      self: { href: `/api/v1/products/${product._id}` },
      reviews: { href: `/api/v1/products/${product._id}/reviews` },
    },
  });
};

const store = async (req, res) => {
  const product = await Product.create(req.body);

  res.status(201)
    .header('Location', `/api/v1/products/${product._id}`)
    .json({
      success: true,
      data: formatProduct(product.toObject()),
    });
};

const update = async (req, res) => {
  const product = await Product.findByIdAndUpdate(
    req.params.id,
    { $set: req.body },
    { new: true, runValidators: true }
  ).lean();

  if (!product) throw new NotFoundError('পণ্যটি খুঁজে পাওয়া যায়নি');

  res.json({ success: true, data: formatProduct(product) });
};

const patch = async (req, res) => {
  const product = await Product.findByIdAndUpdate(
    req.params.id,
    { $set: req.body },
    { new: true, runValidators: true }
  ).lean();

  if (!product) throw new NotFoundError('পণ্যটি খুঁজে পাওয়া যায়নি');

  res.json({ success: true, data: formatProduct(product) });
};

const destroy = async (req, res) => {
  const product = await Product.findByIdAndDelete(req.params.id);
  if (!product) throw new NotFoundError('পণ্যটি খুঁজে পাওয়া যায়নি');

  res.status(204).send();
};

function formatProduct(product) {
  return {
    id: product._id,
    name: product.name,
    slug: product.slug,
    description: product.description,
    price: {
      amount: product.price,
      currency: 'BDT',
      display: `৳${(product.price / 100).toFixed(2)}`,
    },
    stock: product.stock,
    is_available: product.stock > 0,
    category: product.category || null,
    images: product.images || [],
    created_at: product.createdAt,
    updated_at: product.updatedAt,
  };
}

module.exports = { index, show, store, update, patch, destroy };
```

#### Error Handler Middleware

```javascript
// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;

  const response = {
    success: false,
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message: err.message || 'সার্ভারে একটি সমস্যা হয়েছে',
    },
    meta: {
      request_id: req.id,
      timestamp: new Date().toISOString(),
    },
  };

  if (err.name === 'ValidationError') {
    response.error.code = 'VALIDATION_ERROR';
    response.error.details = Object.values(err.errors).map((e) => ({
      field: e.path,
      message: e.message,
    }));
    return res.status(422).json(response);
  }

  if (err.code === 11000) {
    response.error.code = 'DUPLICATE_ENTRY';
    response.error.message = 'এই তথ্যটি আগে থেকেই বিদ্যমান';
    return res.status(409).json(response);
  }

  if (process.env.NODE_ENV === 'development') {
    response.error.stack = err.stack;
  }

  res.status(statusCode).json(response);
};

module.exports = errorHandler;
```

---

## 🔥 Advanced Topics

### ১. API Rate Limiting (থ্রটলিং)

Rate limiting API abuse থেকে রক্ষা করে এবং সার্ভারের সুষ্ঠু ব্যবহার নিশ্চিত করে।

```php
// Laravel — routes/api.php
// বিভিন্ন endpoint-এ বিভিন্ন rate limit
Route::middleware('throttle:60,1')->group(function () {
    // প্রতি মিনিটে ৬০ request
    Route::get('/products', [ProductController::class, 'index']);
});

Route::middleware('throttle:10,1')->group(function () {
    // প্রতি মিনিটে ১০ request (write operation)
    Route::post('/products', [ProductController::class, 'store']);
});

// app/Providers/RouteServiceProvider.php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)
        ->by($request->user()?->id ?: $request->ip())
        ->response(function () {
            return response()->json([
                'success' => false,
                'error' => [
                    'code' => 'RATE_LIMIT_EXCEEDED',
                    'message' => 'অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন',
                ],
            ], 429)->withHeaders([
                'Retry-After' => 60,
                'X-RateLimit-Limit' => 60,
                'X-RateLimit-Remaining' => 0,
            ]);
        });
});
```

```javascript
// Express — Rate Limiter middleware
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);

const createLimiter = ({ windowMs = 60000, max = 60, keyPrefix = 'rl' }) => {
  return rateLimit({
    store: new RedisStore({
      sendCommand: (...args) => redis.call(...args),
      prefix: keyPrefix,
    }),
    windowMs,
    max,
    standardHeaders: true, // X-RateLimit-* headers
    legacyHeaders: false,
    keyGenerator: (req) => req.user?.id || req.ip,
    handler: (req, res) => {
      res.status(429).json({
        success: false,
        error: {
          code: 'RATE_LIMIT_EXCEEDED',
          message: 'অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন',
          retry_after: Math.ceil(windowMs / 1000),
        },
      });
    },
  });
};

// ব্যবহার
app.use('/api/v1', createLimiter({ max: 100 }));
app.use('/api/v1/auth', createLimiter({ max: 10, keyPrefix: 'rl:auth' }));
```

**Response Headers:**
```
X-RateLimit-Limit: 60          ← সর্বোচ্চ request সংখ্যা
X-RateLimit-Remaining: 45      ← বাকি request সংখ্যা
X-RateLimit-Reset: 1705312800  ← কখন reset হবে (Unix timestamp)
Retry-After: 30                ← কত সেকেন্ড পর আবার চেষ্টা করবেন
```

---

### ২. API Caching (ETag, Last-Modified, Cache-Control)

সঠিক caching strategy API performance কয়েকগুণ বাড়িয়ে দেয়।

```
┌─────────────┐                    ┌─────────────┐
│   Client     │                    │   Server     │
├─────────────┤                    ├─────────────┤
│ GET /product │───────────────────►│ 200 OK       │
│              │◄───────────────────│ ETag: "v1"   │
│              │                    │ Cache: 300s  │
│              │                    │              │
│ GET /product │                    │              │
│ If-None-Match│───────────────────►│ 304 Not      │
│  : "v1"      │◄───────────────────│ Modified     │
│              │                    │ (No body!)   │
└─────────────┘                    └─────────────┘
```

```php
// Laravel — ETag & Cache-Control
class ProductController extends Controller
{
    public function show(Request $request, Product $product)
    {
        $etag = md5($product->updated_at->timestamp . $product->id);

        // ETag match হলে 304 পাঠাও — bandwidth বাঁচবে
        if ($request->header('If-None-Match') === '"' . $etag . '"') {
            return response()->noContent(304);
        }

        return (new ProductResource($product))
            ->response()
            ->header('ETag', '"' . $etag . '"')
            ->header('Last-Modified', $product->updated_at->toRfc7231String())
            ->header('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
    }

    public function index(Request $request)
    {
        // Collection-এর জন্য — private cache, ছোট TTL
        $products = Product::latest()->paginate(20);

        return ProductResource::collection($products)
            ->response()
            ->header('Cache-Control', 'private, max-age=60, must-revalidate');
    }
}
```

```javascript
// Express — ETag middleware
const crypto = require('crypto');

const etagMiddleware = (req, res, next) => {
  const originalJson = res.json.bind(res);

  res.json = (body) => {
    const etag = `"${crypto.createHash('md5').update(JSON.stringify(body)).digest('hex')}"`;

    if (req.headers['if-none-match'] === etag) {
      return res.status(304).end();
    }

    res.set('ETag', etag);
    res.set('Cache-Control', 'public, max-age=300');
    return originalJson(body);
  };

  next();
};
```

---

### ৩. Bulk Operations (ব্যাচ অপারেশন)

একাধিক resource একসাথে create/update/delete করার জন্য bulk operations ডিজাইন করা হয়।

```json
// POST /api/v1/products/bulk
{
  "operations": [
    {
      "method": "create",
      "body": { "name": "Product A", "price": 9999, "stock": 100 }
    },
    {
      "method": "update",
      "id": 42,
      "body": { "price": 7999 }
    },
    {
      "method": "delete",
      "id": 55
    }
  ]
}

// Response — প্রতিটি operation-এর ফলাফল আলাদাভাবে
{
  "success": true,
  "results": [
    { "index": 0, "status": 201, "data": { "id": 101, "name": "Product A" } },
    { "index": 1, "status": 200, "data": { "id": 42, "price": 7999 } },
    { "index": 2, "status": 204, "data": null }
  ],
  "summary": {
    "total": 3,
    "succeeded": 3,
    "failed": 0
  }
}
```

```php
// Laravel — Bulk Operations Controller
public function bulkStore(Request $request): JsonResponse
{
    $request->validate([
        'operations' => ['required', 'array', 'max:100'],
        'operations.*.method' => ['required', 'in:create,update,delete'],
    ]);

    $results = [];

    DB::transaction(function () use ($request, &$results) {
        foreach ($request->operations as $index => $op) {
            try {
                $result = match ($op['method']) {
                    'create' => $this->handleCreate($op['body']),
                    'update' => $this->handleUpdate($op['id'], $op['body']),
                    'delete' => $this->handleDelete($op['id']),
                };
                $results[] = ['index' => $index, ...$result];
            } catch (\Exception $e) {
                $results[] = [
                    'index' => $index,
                    'status' => 422,
                    'error' => $e->getMessage(),
                ];
            }
        }
    });

    $succeeded = collect($results)->where('status', '<', 400)->count();

    return response()->json([
        'success' => true,
        'results' => $results,
        'summary' => [
            'total' => count($results),
            'succeeded' => $succeeded,
            'failed' => count($results) - $succeeded,
        ],
    ]);
}
```

---

### ৪. Long-running Operations (অ্যাসিঙ্ক্রোনাস API)

কিছু operation (report generation, bulk import, video processing) তাৎক্ষণিক সম্পন্ন হয় না। এসব ক্ষেত্রে asynchronous pattern ব্যবহার করতে হয়।

```
Client                         Server
  │  POST /api/reports           │
  │  { "type": "sales" }        │
  │ ────────────────────────────►│
  │  202 Accepted                │
  │  Location: /api/jobs/j123    │
  │◄──────────────────────────── │
  │                              │  ← Background processing...
  │  GET /api/jobs/j123          │
  │ ────────────────────────────►│
  │  200 { status: "processing", │
  │        progress: 45 }        │
  │◄──────────────────────────── │
  │                              │
  │  GET /api/jobs/j123          │
  │ ────────────────────────────►│
  │  303 See Other               │
  │  Location: /api/reports/r456 │
  │◄──────────────────────────── │
```

```php
// Laravel — Async Job Pattern
class ReportController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $request->validate([
            'type' => ['required', 'in:sales,inventory,transactions'],
            'date_from' => ['required', 'date'],
            'date_to' => ['required', 'date', 'after:date_from'],
        ]);

        $job = ReportJob::create([
            'type' => $request->type,
            'status' => 'queued',
            'params' => $request->only(['date_from', 'date_to']),
            'user_id' => $request->user()->id,
        ]);

        GenerateReport::dispatch($job);

        return response()->json([
            'success' => true,
            'data' => [
                'job_id' => $job->id,
                'status' => 'queued',
                'message' => 'রিপোর্ট তৈরি শুরু হয়েছে',
            ],
            'links' => [
                'status' => ['href' => "/api/v1/jobs/{$job->id}", 'method' => 'GET'],
                'cancel' => ['href' => "/api/v1/jobs/{$job->id}/cancel", 'method' => 'POST'],
            ],
        ], 202)->header('Location', "/api/v1/jobs/{$job->id}");
    }

    public function jobStatus(ReportJob $job): JsonResponse
    {
        if ($job->status === 'completed') {
            return response()->json(null, 303)
                ->header('Location', "/api/v1/reports/{$job->report_id}");
        }

        return response()->json([
            'data' => [
                'job_id' => $job->id,
                'status' => $job->status,
                'progress' => $job->progress,
                'estimated_time' => $job->estimated_completion,
            ],
        ]);
    }
}
```

#### Webhook Pattern — ফলাফল push করুন

```json
// Webhook callback — সার্ভার ক্লায়েন্টকে জানাচ্ছে কাজ শেষ
POST https://your-app.com/webhooks/report-ready
Content-Type: application/json
X-Webhook-Signature: sha256=abc123...

{
  "event": "report.completed",
  "data": {
    "job_id": "j123",
    "report_id": "r456",
    "download_url": "/api/v1/reports/r456/download",
    "expires_at": "2024-01-16T10:00:00Z"
  },
  "timestamp": "2024-01-15T15:00:00+06:00"
}
```

---

### ৫. File Upload API Design

#### পদ্ধতি ১: Multipart Upload (ছোট ফাইলের জন্য)

```php
// Laravel — Direct Upload
class FileUploadController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $request->validate([
            'file' => ['required', 'file', 'max:10240', 'mimes:jpg,png,pdf'],
            'type' => ['required', 'in:product_image,document'],
        ]);

        $path = $request->file('file')->store(
            "uploads/{$request->type}",
            's3'
        );

        return response()->json([
            'success' => true,
            'data' => [
                'url' => Storage::disk('s3')->url($path),
                'path' => $path,
                'size' => $request->file('file')->getSize(),
                'mime_type' => $request->file('file')->getMimeType(),
            ],
        ], 201);
    }
}
```

#### পদ্ধতি ২: Presigned URL (বড় ফাইলের জন্য — সার্ভার bypass)

```
Client                    API Server              S3/Storage
  │ POST /uploads/init      │                        │
  │ { filename, size }      │                        │
  │ ───────────────────────►│                        │
  │                         │  Generate presigned URL│
  │  { upload_url, id }     │                        │
  │◄─────────────────────── │                        │
  │                         │                        │
  │ PUT upload_url          │                        │
  │ [binary file data] ─────┼───────────────────────►│
  │                         │                        │
  │ POST /uploads/{id}/complete                      │
  │ ───────────────────────►│                        │
  │  { url, metadata }      │  Verify file exists    │
  │◄─────────────────────── │───────────────────────►│
```

```javascript
// Express — Presigned URL Pattern
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');

const initUpload = async (req, res) => {
  const { filename, content_type, size } = req.body;

  if (size > 50 * 1024 * 1024) {
    return res.status(413).json({
      success: false,
      error: { message: 'ফাইল সাইজ ৫০MB এর বেশি হতে পারে না' },
    });
  }

  const key = `uploads/${req.user.id}/${Date.now()}-${filename}`;

  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: content_type,
    ContentLength: size,
  });

  const uploadUrl = await getSignedUrl(s3Client, command, {
    expiresIn: 3600,
  });

  const upload = await Upload.create({
    key,
    user_id: req.user.id,
    status: 'pending',
    filename,
    size,
  });

  res.status(201).json({
    success: true,
    data: {
      upload_id: upload._id,
      upload_url: uploadUrl,
      expires_in: 3600,
    },
  });
};
```

---

### ৬. Large Dataset Pagination Best Practices

বড় dataset-এ pagination-এর কৌশল:

```
পদ্ধতি           | Performance | UI                | বিশেষত্ব
─────────────────|─────────────|───────────────────|───────────────────
Offset-based     | O(n) ❌     | page 1,2,3...     | সহজ, কিন্তু skip expensive
Cursor-based     | O(1) ✅     | Next/Prev only    | দ্রুত, real-time safe
Keyset           | O(1) ✅     | Next/Prev only    | সবচেয়ে দ্রুত, index-friendly
Seek (combined)  | O(1) ✅     | Flexible          | cursor + keyset hybrid
```

```php
// Laravel — Keyset Pagination (সবচেয়ে efficient)
public function index(Request $request)
{
    $limit = min($request->integer('limit', 20), 100);

    $query = Product::query()->orderBy('id');

    // keyset: id > last_seen_id ব্যবহার করে
    if ($request->filled('after_id')) {
        $query->where('id', '>', $request->after_id);
    }

    $products = $query->limit($limit + 1)->get();

    $hasMore = $products->count() > $limit;
    if ($hasMore) {
        $products->pop();
    }

    return response()->json([
        'data' => ProductResource::collection($products),
        'meta' => [
            'has_more' => $hasMore,
            'next_after_id' => $hasMore ? $products->last()->id : null,
        ],
    ]);
}
```

---

### ৭. Idempotency Keys (নিরাপদ Retry)

নেটওয়ার্ক সমস্যার কারণে client একই request দুবার পাঠাতে পারে। Idempotency key নিশ্চিত করে যে duplicate request-এ duplicate operation হবে না।

```
Client                              Server
  │ POST /api/orders                  │
  │ Idempotency-Key: idk_abc123      │
  │ ─────────────────────────────────►│ ← Key store করো
  │ 201 Created { order_id: 500 }    │
  │◄───────────────────────────────── │
  │                                   │
  │ (Network timeout — client retry)  │
  │                                   │
  │ POST /api/orders                  │
  │ Idempotency-Key: idk_abc123      │
  │ ─────────────────────────────────►│ ← Key আগে দেখেছি!
  │ 200 OK { order_id: 500 }         │  ← আগের response ফেরত দাও
  │◄───────────────────────────────── │
```

```php
// Laravel — Idempotency Middleware
namespace App\Http\Middleware;

use Illuminate\Support\Facades\Cache;
use Closure;

class IdempotencyMiddleware
{
    public function handle($request, Closure $next)
    {
        if (!in_array($request->method(), ['POST', 'PATCH', 'PUT'])) {
            return $next($request);
        }

        $key = $request->header('Idempotency-Key');
        if (!$key) {
            return $next($request);
        }

        $cacheKey = "idempotency:{$request->user()->id}:{$key}";

        // আগে এই key দিয়ে request এসেছিল?
        $cached = Cache::get($cacheKey);
        if ($cached) {
            return response()->json(
                $cached['body'],
                $cached['status']
            )->header('X-Idempotent-Replayed', 'true');
        }

        $response = $next($request);

        // Response ক্যাশ করো (২৪ ঘণ্টা)
        if ($response->isSuccessful()) {
            Cache::put($cacheKey, [
                'status' => $response->getStatusCode(),
                'body' => json_decode($response->getContent(), true),
            ], now()->addHours(24));
        }

        return $response;
    }
}
```

```javascript
// Express — Idempotency Middleware
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

const idempotency = async (req, res, next) => {
  if (!['POST', 'PUT', 'PATCH'].includes(req.method)) return next();

  const key = req.headers['idempotency-key'];
  if (!key) return next();

  const cacheKey = `idempotency:${req.user?.id}:${key}`;

  const cached = await redis.get(cacheKey);
  if (cached) {
    const { status, body } = JSON.parse(cached);
    return res.status(status).set('X-Idempotent-Replayed', 'true').json(body);
  }

  const originalJson = res.json.bind(res);
  res.json = (body) => {
    if (res.statusCode >= 200 && res.statusCode < 300) {
      redis.setex(cacheKey, 86400, JSON.stringify({
        status: res.statusCode,
        body,
      }));
    }
    return originalJson(body);
  };

  next();
};

module.exports = idempotency;
```

---

### ৮. Richardson Maturity Model — REST পরিপক্বতা স্তর

Leonard Richardson-এর এই মডেল REST API-এর পরিপক্বতা চারটি স্তরে মাপে:

```
Level 3 ─── Hypermedia Controls (HATEOAS)        ← সত্যিকারের REST
   ▲        Response-এ discoverable links
   │
Level 2 ─── HTTP Verbs + Status Codes            ← বেশিরভাগ API এখানে
   ▲        GET/POST/PUT/DELETE + 200/201/404
   │
Level 1 ─── Resources (URI-based)                ← /products, /orders
   ▲        একাধিক URI, কিন্তু শুধু POST
   │
Level 0 ─── The Swamp of POX                     ← RPC-style
            একটিমাত্র URI, শুধু POST
            POST /api → { "action": "getProduct" }
```

#### Level 0 — POX (Plain Old XML/JSON)

```
POST /api
{ "action": "getProducts", "category": "phones" }

POST /api
{ "action": "createOrder", "product_id": 42 }
```

একটি endpoint, সবকিছু POST — এটা REST নয়, এটা RPC।

#### Level 1 — Resources

```
POST /products        ← পড়ার জন্যও POST!
POST /products/42     ← আপডেটের জন্যও POST!
POST /orders          ← সবই POST
```

URI আলাদা আছে কিন্তু HTTP method সঠিকভাবে ব্যবহার হচ্ছে না।

#### Level 2 — HTTP Verbs (সবচেয়ে সাধারণ)

```
GET    /products          ← পড়া
POST   /products          ← তৈরি
PUT    /products/42       ← আপডেট
DELETE /products/42       ← মুছে ফেলা
```

বেশিরভাগ production API এই স্তরে আছে এবং এটাই যথেষ্ট।

#### Level 3 — HATEOAS (সম্পূর্ণ REST)

```json
{
  "data": { "id": 42, "name": "bKash Payment", "status": "pending" },
  "links": {
    "self": { "href": "/payments/42" },
    "execute": { "href": "/payments/42/execute", "method": "POST" },
    "cancel": { "href": "/payments/42/cancel", "method": "POST" }
  }
}
```

ক্লায়েন্ট response-এর links অনুসরণ করে navigate করে — URI hardcode করতে হয় না।

---

## ✅ Best Practices Checklist

```
✅ Resource Design
   □ Noun-based plural URIs (/products, /orders)
   □ Consistent naming convention (kebab-case)
   □ Maximum 2 levels নেস্টিং (/shops/5/products — তার বেশি নয়)
   □ API versioning (URI: /v1/ অথবা Header: Accept-Version)

✅ Request/Response
   □ সব response-এ consistent JSON structure
   □ Meaningful error messages (error code + human message + details)
   □ Proper HTTP status codes
   □ Pagination সব collection endpoint-এ
   □ Date/time সব জায়গায় ISO 8601 (UTC বা timezone সহ)
   □ Money integer হিসেবে store করুন (পয়সায়: ৳999.50 → 99950)

✅ Security
   □ HTTPS বাধ্যতামূলক
   □ Rate limiting সব endpoint-এ
   □ Input validation সার্ভার-সাইডে
   □ Authentication (Bearer token / API key)
   □ Authorization (role/permission-based)
   □ CORS সঠিকভাবে configure করা
   □ Sensitive data response-এ expose করবেন না (password, secret)

✅ Performance
   □ Caching headers (ETag, Cache-Control, Last-Modified)
   □ Field selection (sparse fieldsets)
   □ Eager loading (N+1 query এড়ানো)
   □ Response compression (gzip/brotli)
   □ Cursor-based pagination (বড় dataset)

✅ Developer Experience
   □ OpenAPI/Swagger documentation
   □ Consistent error format
   □ Request ID প্রতিটি response-এ (debugging-এর জন্য)
   □ Deprecation headers (Sunset, Deprecation)
   □ HATEOAS links (যেখানে সম্ভব)
```

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ১. Verb-based URI

```
❌ POST /api/getProducts
❌ POST /api/createUser
❌ GET  /api/deleteProduct/42

✅ GET    /api/products
✅ POST   /api/users
✅ DELETE /api/products/42
```

### ২. ভুল Status Code

```
❌ সবকিছুতে 200 ফেরত দেওয়া
   200 { "error": true, "message": "Not found" }   ← ভয়ংকর!

✅ সঠিক status code ব্যবহার
   404 { "error": { "code": "NOT_FOUND" } }
```

### ৩. Versioning না করা

```
❌ /api/products              ← breaking change হলে সব client ভেঙে যাবে!

✅ /api/v1/products           ← নতুন version: /api/v2/products
✅ Accept: application/vnd.myapi.v1+json  ← header-based versioning
```

### ৪. N+1 Query সমস্যা

```php
// ❌ N+1 — প্রতিটি product-এর জন্য আলাদা query
$products = Product::all();
foreach ($products as $product) {
    echo $product->category->name; // প্রতিবার নতুন query!
}

// ✅ Eager Loading — একটিমাত্র JOIN query
$products = Product::with('category')->get();
```

### ৫. Sensitive Data Exposure

```json
// ❌ Password hash response-এ!
{
  "id": 1,
  "name": "Karim",
  "email": "karim@example.com",
  "password": "$2y$10$abc...",
  "api_secret": "sk_live_abc123"
}

// ✅ শুধু প্রয়োজনীয় তথ্য
{
  "id": 1,
  "name": "Karim",
  "email": "karim@example.com"
}
```

### ৬. Pagination ছাড়া Collection Return

```
❌ GET /api/products → ১০,০০০ পণ্য একসাথে!
   • Memory overflow
   • ধীর response
   • Mobile-এ data খরচ

✅ GET /api/products?page=1&per_page=20
   • সর্বোচ্চ per_page limit রাখুন (100)
   • Default per_page দিন (20)
```

### ৭. PUT দিয়ে Partial Update

```json
// ❌ PUT-এ শুধু price পাঠালে বাকি field null হয়ে যাবে!
PUT /api/products/42
{ "price": 5999 }
// Result: name=null, stock=null — সব হারিয়ে গেছে!

// ✅ Partial update-এ PATCH ব্যবহার করুন
PATCH /api/products/42
{ "price": 5999 }
// Result: শুধু price পরিবর্তন হবে, বাকি সব আগের মতো
```

### ৮. Nested Resource অতিরিক্ত গভীরতা

```
❌ /api/countries/bd/divisions/dhaka/districts/gazipur/shops/5/products/42/reviews/3

✅ /api/reviews/3
✅ /api/shops/5/products?district=gazipur
```

---

## 📋 সারসংক্ষেপ (Summary)

### REST API ডিজাইনের মূল সূত্র

| বিষয় | সুপারিশ |
|-------|---------|
| **URI** | Noun-based, plural, kebab-case, max 2 নেস্টিং |
| **HTTP Method** | GET=পড়া, POST=তৈরি, PUT=প্রতিস্থাপন, PATCH=আংশিক, DELETE=মুছা |
| **Status Code** | সঠিক code ব্যবহার করুন — শুধু 200 দিয়ে কাজ চালাবেন না |
| **Response** | Consistent JSON structure, error details, pagination |
| **Versioning** | URI (/v1/) অথবা Header-based |
| **Pagination** | Cursor-based (বড় dataset), Offset (ছোট dataset) |
| **Caching** | ETag + Cache-Control ব্যবহার করুন |
| **Security** | HTTPS + Rate Limit + Input Validation + Auth |
| **Idempotency** | POST request-এ Idempotency-Key header |
| **Documentation** | OpenAPI/Swagger — সবসময় up-to-date রাখুন |

### বাংলাদেশ Context — bKash/Nagad API ডিজাইন থেকে শিক্ষণীয়

```
১. Transaction amount সবসময় smallest unit-এ রাখুন (পয়সায়)
   ৳100.50 → 10050 (integer) — floating point ভুল এড়ানো যায়

২. Phone number validation — বাংলাদেশের format:
   ^(\+?880|0)(1[3-9]\d{8})$

৩. Timezone সবসময় specify করুন:
   Asia/Dhaka (UTC+6), ISO 8601: 2024-01-15T14:30:00+06:00

৪. Bangla text support — UTF-8 encoding নিশ্চিত করুন
   Content-Type: application/json; charset=utf-8

৫. Dual language support — error message দুই ভাষায়:
   { "message_bn": "লেনদেন সফল", "message_en": "Transaction successful" }
```

### চূড়ান্ত পরামর্শ

REST API ডিজাইন কোনো কঠোর specification নয় — এটি একটি **architectural style**। Roy Fielding-এর constraints মেনে চললে আপনার API হবে scalable, maintainable, এবং developer-friendly। Level 2 (HTTP verbs + status codes) বেশিরভাগ ক্ষেত্রেই যথেষ্ট — HATEOAS (Level 3) সবসময় প্রয়োজন হয় না।

**সবচেয়ে গুরুত্বপূর্ণ:** Consistency বজায় রাখুন। একটি mediocre কিন্তু consistent API, একটি brilliant কিন্তু inconsistent API-এর চেয়ে অনেক ভালো।

---

> **"একটি ভালো API হলো সেটি যা ব্যবহার করার জন্য documentation কম লাগে।"** — ভালো naming convention এবং consistent structure-ই সবচেয়ে ভালো documentation।
