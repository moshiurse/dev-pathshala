# 🧩 API Composition & Aggregation Pattern (API কম্পোজিশন ও অ্যাগ্রিগেশন প্যাটার্ন)

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [BFF Pattern](#bff-pattern)
- [Parallel vs Sequential Calls](#parallel-vs-sequential-calls)
- [Error Handling ও Partial Failure](#error-handling-ও-partial-failure)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [CQRS-এর সাথে তুলনা](#cqrs-এর-সাথে-তুলনা)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**API Composition Pattern** হলো মাইক্রোসার্ভিস আর্কিটেকচারে একটি প্যাটার্ন যেখানে একটি **API Composer** (বা Aggregator) একাধিক সার্ভিস থেকে ডেটা সংগ্রহ করে, একত্রিত করে এবং ক্লায়েন্টকে একটি unified response হিসেবে ফেরত দেয়।

### 🤔 কেন API Composition দরকার?

মাইক্রোসার্ভিসে ডেটা বিভিন্ন সার্ভিসে বিভক্ত থাকে। একটি পেজ রেন্ডার করতে একাধিক সার্ভিস থেকে ডেটা প্রয়োজন হতে পারে:

```
একটি E-commerce Product Page-এ যা দরকার:
├── 📦 Product Service     → পণ্যের নাম, বিবরণ, ছবি
├── ⭐ Review Service      → রেটিং ও রিভিউ
├── 📊 Inventory Service   → স্টক আছে কিনা
├── 💰 Pricing Service     → মূল্য, ডিসকাউন্ট
├── 🚚 Shipping Service    → ডেলিভারি সময় ও খরচ
└── 👤 Seller Service      → বিক্রেতার তথ্য
```

**সমস্যা:** Client-কে ৬টি আলাদা API call করতে হবে? ❌
**সমাধান:** একটি API Composer সব ডেটা সংগ্রহ করে একটি response দেবে! ✅

### 📖 মূল ধারণাগুলো:

| ধারণা | বিবরণ |
|-------|--------|
| **API Composer** | যে সার্ভিস অন্যদের থেকে ডেটা সংগ্রহ ও একত্রিত করে |
| **BFF (Backend for Frontend)** | নির্দিষ্ট frontend-এর জন্য বিশেষায়িত composer |
| **Parallel Calls** | একসাথে একাধিক সার্ভিসে call পাঠানো |
| **Partial Failure** | কোনো একটি সার্ভিস fail করলেও বাকিদের response দেওয়া |
| **Timeout Management** | সার্ভিস response না দিলে কতক্ষণ অপেক্ষা করা হবে |
| **Response Caching** | composed response cache করে performance বাড়ানো |

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🛒 Daraz Bangladesh - Product Page

ধরুন Daraz-এর product page `/product/12345` load করতে:

```
User Request: GET /product/12345

API Composer-এর কাজ:
1. Product Service → পণ্যের তথ্য (নাম, বিবরণ, ছবি)
2. Review Service → 4.5⭐ (১২৩ reviews)
3. Inventory Service → "স্টকে আছে" ✅
4. Pricing Service → ৳1,299 (৳1,599 থেকে ১৯% ছাড়)
5. Shipping Service → "ঢাকায় ২ দিনে ডেলিভারি"
6. Seller Service → "TechBD Official Store" (৯৮% positive)

Combined Response → Client-কে একটি response দেয়
```

### 📰 Prothom Alo - Homepage

Prothom Alo homepage-এ একসাথে দরকার:
- **News Service** → সর্বশেষ সংবাদ
- **Ad Service** → বিজ্ঞাপন
- **Trending Service** → জনপ্রিয় পোস্ট
- **Weather Service** → আবহাওয়া
- **Sports Service** → খেলার স্কোর

### 📱 bKash App - Dashboard

bKash অ্যাপ খুললে dashboard-এ দেখা যায়:
- **Account Service** → ব্যালেন্স ৳৫,৪৩২
- **Transaction Service** → সাম্প্রতিক লেনদেন
- **Offer Service** → ক্যাশব্যাক অফার
- **Bill Service** → পেন্ডিং বিল
- **Notification Service** → unread notifications

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Basic API Composition Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                   API COMPOSITION PATTERN                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐          ┌──────────────────┐                      │
│  │          │  Single  │                  │                      │
│  │  Client  │─────────►│  API Composer /  │                      │
│  │  (App)   │◄─────────│  Aggregator      │                      │
│  │          │ Combined │                  │                      │
│  └──────────┘ Response └────────┬─────────┘                      │
│                                 │                                │
│                    ┌────────────┼────────────┐                   │
│                    │            │            │                    │
│                    ▼            ▼            ▼                    │
│             ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│             │ Product  │ │ Review   │ │ Pricing  │              │
│             │ Service  │ │ Service  │ │ Service  │              │
│             │          │ │          │ │          │              │
│             │  ┌────┐  │ │  ┌────┐  │ │  ┌────┐  │              │
│             │  │ DB │  │ │  │ DB │  │ │  │ DB │  │              │
│             │  └────┘  │ │  └────┘  │ │  └────┘  │              │
│             └──────────┘ └──────────┘ └──────────┘              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### BFF (Backend for Frontend) Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    BFF PATTERN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │  Mobile  │  │   Web    │  │  Smart   │                       │
│  │   App    │  │  Browser │  │   TV     │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       │              │              │                             │
│       ▼              ▼              ▼                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │  Mobile  │  │   Web    │  │    TV    │                       │
│  │   BFF    │  │   BFF    │  │   BFF    │                       │
│  │(কম ডেটা) │  │(বেশি ডেটা)│  │(ভিডিও)  │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       │              │              │                             │
│       └──────────────┼──────────────┘                            │
│                      │                                           │
│         ┌────────────┼────────────┐                              │
│         │            │            │                              │
│         ▼            ▼            ▼                              │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│   │ Product  │ │  User    │ │  Order   │                        │
│   │ Service  │ │ Service  │ │ Service  │                        │
│   └──────────┘ └──────────┘ └──────────┘                        │
│                                                                   │
│  Mobile BFF: শুধু প্রয়োজনীয় ফিল্ড, compressed image           │
│  Web BFF: পূর্ণ ডেটা, high-res image, SEO data                  │
│  TV BFF: ভিডিও URL, large thumbnails                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Parallel vs Sequential Execution

```
┌─────────────────────────────────────────────────────────────────┐
│          PARALLEL vs SEQUENTIAL CALLS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SEQUENTIAL (ধীর ❌):                                            │
│  ══════════════════════════════════════════════════►  Time       │
│  │──Product──│──Review──│──Price──│──Stock──│                    │
│  0ms       200ms      400ms    550ms    700ms                   │
│  Total: 700ms 😢                                                │
│                                                                   │
│  ─────────────────────────────────────────────────              │
│                                                                   │
│  PARALLEL (দ্রুত ✅):                                            │
│  ══════════════════════════════════════════════════►  Time       │
│  │──Product──────│                                              │
│  │──Review───│                                                  │
│  │──Price─│                                                     │
│  │──Stock────────────│                                          │
│  0ms              250ms                                         │
│  Total: 250ms (সবচেয়ে ধীর সার্ভিসের সময়) 🚀                  │
│                                                                   │
│  ─────────────────────────────────────────────────              │
│                                                                   │
│  MIXED (স্মার্ট ✅✅):                                           │
│  ══════════════════════════════════════════════════►  Time       │
│  │──Product──│                                                  │
│                │──Price (needs product_id)──│                    │
│  │──Review───│                                                  │
│  │──Stock────│                                                  │
│  0ms       200ms                        350ms                   │
│  Total: 350ms (dependent calls sequential, rest parallel)       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Partial Failure Handling

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTIAL FAILURE HANDLING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  API Composer calls 4 services:                                  │
│                                                                   │
│  Product Service  ──► ✅ 200 OK (name, description)             │
│  Review Service   ──► ❌ 503 Service Unavailable                │
│  Pricing Service  ──► ✅ 200 OK (price: ৳1,299)                │
│  Stock Service    ──► ⏰ Timeout (5s exceeded)                  │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                │
│  │  Response Strategy Options:                  │                │
│  ├─────────────────────────────────────────────┤                │
│  │                                             │                │
│  │  1. FAIL ALL (কঠোর):                       │                │
│  │     → 503 Error response                    │                │
│  │     ❌ খারাপ UX                             │                │
│  │                                             │                │
│  │  2. PARTIAL RESPONSE (নমনীয়):              │                │
│  │     → Return available data + null fields   │                │
│  │     {                                       │                │
│  │       product: {...},     ✅               │                │
│  │       reviews: null,      ⚠️ unavailable   │                │
│  │       price: {...},       ✅               │                │
│  │       stock: "unknown"    ⚠️ timeout       │                │
│  │     }                                       │                │
│  │     ✅ ভালো UX                              │                │
│  │                                             │                │
│  │  3. CACHED FALLBACK (সেরা):                │                │
│  │     → Return cached data for failed ones    │                │
│  │     {                                       │                │
│  │       product: {...},           ✅ fresh    │                │
│  │       reviews: {cached: true},  📦 cached  │                │
│  │       price: {...},             ✅ fresh    │                │
│  │       stock: {cached: true}     📦 cached  │                │
│  │     }                                       │                │
│  │     ✅✅ সেরা UX                            │                │
│  │                                             │                │
│  └─────────────────────────────────────────────┘                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Circuit Breaker Integration

```
┌───────────────────────────────────────────────────────────────┐
│         API COMPOSER + CIRCUIT BREAKER                          │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│                    ┌──────────────┐                             │
│                    │ API Composer │                             │
│                    └──────┬───────┘                             │
│                           │                                    │
│           ┌───────────────┼───────────────┐                    │
│           │               │               │                    │
│           ▼               ▼               ▼                    │
│    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│    │   Circuit   │ │   Circuit   │ │   Circuit   │            │
│    │   Breaker   │ │   Breaker   │ │   Breaker   │            │
│    │   [CLOSED]  │ │   [OPEN]    │ │   [CLOSED]  │            │
│    │      ✅     │ │     🔴      │ │      ✅     │            │
│    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │
│           │               │               │                    │
│           ▼               ▼               ▼                    │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│    │ Product  │    │ Review   │    │ Pricing  │               │
│    │ Service  │    │ Service  │    │ Service  │               │
│    │   ✅     │    │   ☠️     │    │   ✅     │               │
│    └──────────┘    └──────────┘    └──────────┘               │
│                                                                │
│  Review Service-এর Circuit Breaker OPEN:                      │
│  → Request পাঠানো হবে না                                     │
│  → Immediately fallback/cached data return করবে              │
│  → ৩০ সেকেন্ড পর HALF-OPEN হবে (test request পাঠাবে)        │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

---

## 🖥️ PHP কোড উদাহরণ

### Complete API Composer Implementation

```php
<?php

/**
 * API Composition Pattern - PHP Implementation
 * E-commerce Product Page Aggregation (Daraz-like)
 */

// ===============================
// Service Response DTO
// ===============================

class ServiceResponse
{
    public bool $success;
    public mixed $data;
    public ?string $error;
    public float $responseTime;
    public bool $fromCache;

    public function __construct(bool $success, mixed $data = null, ?string $error = null, float $responseTime = 0, bool $fromCache = false)
    {
        $this->success = $success;
        $this->data = $data;
        $this->error = $error;
        $this->responseTime = $responseTime;
        $this->fromCache = $fromCache;
    }
}

// ===============================
// Circuit Breaker
// ===============================

class CircuitBreaker
{
    private string $serviceName;
    private string $state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    private int $failureCount = 0;
    private int $failureThreshold;
    private int $resetTimeout;
    private int $lastFailureTime = 0;

    public function __construct(string $serviceName, int $failureThreshold = 3, int $resetTimeout = 30)
    {
        $this->serviceName = $serviceName;
        $this->failureThreshold = $failureThreshold;
        $this->resetTimeout = $resetTimeout;
    }

    public function isOpen(): bool
    {
        if ($this->state === 'OPEN') {
            // Reset timeout পার হলে HALF_OPEN এ যাবে
            if ((time() - $this->lastFailureTime) > $this->resetTimeout) {
                $this->state = 'HALF_OPEN';
                return false;
            }
            return true;
        }
        return false;
    }

    public function recordSuccess(): void
    {
        $this->failureCount = 0;
        $this->state = 'CLOSED';
    }

    public function recordFailure(): void
    {
        $this->failureCount++;
        $this->lastFailureTime = time();

        if ($this->failureCount >= $this->failureThreshold) {
            $this->state = 'OPEN';
            echo "🔴 Circuit OPEN for {$this->serviceName}\n";
        }
    }

    public function getState(): string
    {
        return $this->state;
    }
}

// ===============================
// Service Client (Individual Service Caller)
// ===============================

class ServiceClient
{
    private string $baseUrl;
    private int $timeout;
    private CircuitBreaker $circuitBreaker;
    private array $cache = [];
    private int $cacheTTL;

    public function __construct(string $baseUrl, int $timeout = 5, int $cacheTTL = 60)
    {
        $this->baseUrl = $baseUrl;
        $this->timeout = $timeout;
        $this->cacheTTL = $cacheTTL;
        $this->circuitBreaker = new CircuitBreaker($baseUrl);
    }

    public function call(string $endpoint, array $params = []): ServiceResponse
    {
        $cacheKey = $this->getCacheKey($endpoint, $params);
        $startTime = microtime(true);

        // Circuit Breaker চেক
        if ($this->circuitBreaker->isOpen()) {
            // Cached data ফেরত দেওয়া
            if (isset($this->cache[$cacheKey])) {
                return new ServiceResponse(true, $this->cache[$cacheKey]['data'], null, 0, true);
            }
            return new ServiceResponse(false, null, "Circuit breaker OPEN for {$this->baseUrl}");
        }

        try {
            $url = $this->baseUrl . $endpoint . '?' . http_build_query($params);

            $ch = curl_init($url);
            curl_setopt_array($ch, [
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => $this->timeout,
                CURLOPT_CONNECTTIMEOUT => 2,
                CURLOPT_HTTPHEADER => ['Accept: application/json']
            ]);

            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            $error = curl_error($ch);
            curl_close($ch);

            $responseTime = (microtime(true) - $startTime) * 1000;

            if ($error || $httpCode >= 500) {
                throw new \RuntimeException($error ?: "HTTP {$httpCode}");
            }

            $data = json_decode($response, true);

            // Success - cache ও circuit breaker update
            $this->cache[$cacheKey] = ['data' => $data, 'time' => time()];
            $this->circuitBreaker->recordSuccess();

            return new ServiceResponse(true, $data, null, $responseTime);

        } catch (\Exception $e) {
            $responseTime = (microtime(true) - $startTime) * 1000;
            $this->circuitBreaker->recordFailure();

            // Fallback to cache
            if (isset($this->cache[$cacheKey])) {
                return new ServiceResponse(true, $this->cache[$cacheKey]['data'], null, $responseTime, true);
            }

            return new ServiceResponse(false, null, $e->getMessage(), $responseTime);
        }
    }

    private function getCacheKey(string $endpoint, array $params): string
    {
        return md5($endpoint . serialize($params));
    }
}

// ===============================
// API Composer - Product Page
// ===============================

class ProductPageComposer
{
    private ServiceClient $productService;
    private ServiceClient $reviewService;
    private ServiceClient $inventoryService;
    private ServiceClient $pricingService;
    private ServiceClient $shippingService;
    private ServiceClient $sellerService;

    public function __construct()
    {
        $this->productService = new ServiceClient('http://product-service:8001', timeout: 3);
        $this->reviewService = new ServiceClient('http://review-service:8002', timeout: 3);
        $this->inventoryService = new ServiceClient('http://inventory-service:8003', timeout: 2);
        $this->pricingService = new ServiceClient('http://pricing-service:8004', timeout: 2);
        $this->shippingService = new ServiceClient('http://shipping-service:8005', timeout: 3);
        $this->sellerService = new ServiceClient('http://seller-service:8006', timeout: 3);
    }

    /**
     * Product Page-এর সকল ডেটা compose করা
     * Parallel execution with partial failure handling
     */
    public function compose(string $productId, string $userId = null): array
    {
        $startTime = microtime(true);

        // Parallel calls (PHP-তে curl_multi ব্যবহার করে)
        $results = $this->executeParallel([
            'product' => fn() => $this->productService->call("/products/{$productId}"),
            'reviews' => fn() => $this->reviewService->call("/reviews", ['product_id' => $productId]),
            'inventory' => fn() => $this->inventoryService->call("/stock/{$productId}"),
            'pricing' => fn() => $this->pricingService->call("/price/{$productId}"),
            'shipping' => fn() => $this->shippingService->call("/estimate", [
                'product_id' => $productId,
                'user_id' => $userId
            ]),
            'seller' => fn() => $this->getSellerInfo($productId),
        ]);

        $totalTime = (microtime(true) - $startTime) * 1000;

        return $this->buildResponse($results, $totalTime);
    }

    /**
     * Parallel execution simulation
     * বাস্তবে curl_multi_exec বা Guzzle Promises ব্যবহার হবে
     */
    private function executeParallel(array $tasks): array
    {
        $results = [];

        // PHP-তে সত্যিকারের parallel execution-এর জন্য
        // Guzzle async requests বা curl_multi ব্যবহার করুন
        foreach ($tasks as $key => $task) {
            $results[$key] = $task();
        }

        return $results;
    }

    /**
     * Seller info - dependent call (product থেকে seller_id প্রয়োজন)
     */
    private function getSellerInfo(string $productId): ServiceResponse
    {
        // এটি sequential - product data থেকে seller_id লাগবে
        $productResponse = $this->productService->call("/products/{$productId}/seller");
        if (!$productResponse->success) {
            return new ServiceResponse(false, null, "Seller info unavailable");
        }
        $sellerId = $productResponse->data['seller_id'] ?? null;
        if (!$sellerId) {
            return new ServiceResponse(false, null, "No seller_id found");
        }
        return $this->sellerService->call("/sellers/{$sellerId}");
    }

    /**
     * Final response তৈরি - partial failure handle করে
     */
    private function buildResponse(array $results, float $totalTime): array
    {
        $response = [
            'success' => true,
            'data' => [],
            'metadata' => [
                'total_time_ms' => round($totalTime, 2),
                'services' => [],
                'partial_failure' => false,
            ]
        ];

        foreach ($results as $key => $serviceResponse) {
            $response['metadata']['services'][$key] = [
                'success' => $serviceResponse->success,
                'response_time_ms' => round($serviceResponse->responseTime, 2),
                'from_cache' => $serviceResponse->fromCache,
                'error' => $serviceResponse->error,
            ];

            if ($serviceResponse->success) {
                $response['data'][$key] = $serviceResponse->data;
            } else {
                $response['data'][$key] = null;
                $response['metadata']['partial_failure'] = true;
            }
        }

        // Critical service fail হলে error response
        if (!$results['product']->success) {
            $response['success'] = false;
            $response['error'] = 'Product data unavailable';
        }

        return $response;
    }
}

// ===============================
// BFF - Mobile App এর জন্য
// ===============================

class MobileBFF
{
    private ProductPageComposer $composer;

    public function __construct()
    {
        $this->composer = new ProductPageComposer();
    }

    /**
     * Mobile-এর জন্য optimized response
     * কম ডেটা, compressed images, essential fields only
     */
    public function getProductForMobile(string $productId, string $userId): array
    {
        $fullData = $this->composer->compose($productId, $userId);

        if (!$fullData['success']) {
            return $fullData;
        }

        // Mobile-এর জন্য শুধু প্রয়োজনীয় ফিল্ড
        return [
            'success' => true,
            'data' => [
                'name' => $fullData['data']['product']['name'] ?? '',
                'price' => $fullData['data']['pricing']['final_price'] ?? 0,
                'discount_percent' => $fullData['data']['pricing']['discount_percent'] ?? 0,
                'rating' => $fullData['data']['reviews']['average_rating'] ?? 0,
                'review_count' => $fullData['data']['reviews']['total_count'] ?? 0,
                'in_stock' => $fullData['data']['inventory']['available'] ?? false,
                'image_url' => $fullData['data']['product']['thumbnail_url'] ?? '', // ছোট ছবি
                'delivery_days' => $fullData['data']['shipping']['estimated_days'] ?? null,
            ]
        ];
    }
}

// ===============================
// ব্যবহার
// ===============================

// Product Page API
$composer = new ProductPageComposer();
$result = $composer->compose('PROD-12345', 'USER-789');

echo json_encode($result, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);

// Mobile BFF
$mobileBff = new MobileBFF();
$mobileResult = $mobileBff->getProductForMobile('PROD-12345', 'USER-789');

echo "\n\n=== Mobile Response ===\n";
echo json_encode($mobileResult, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
```

---

## 🟨 JavaScript কোড উদাহরণ

### Complete API Composer with Parallel Execution

```javascript
/**
 * API Composition Pattern - JavaScript/Node.js Implementation
 * E-commerce Product Page & bKash Dashboard Aggregation
 */

// ===============================
// Circuit Breaker
// ===============================

class CircuitBreaker {
    constructor(serviceName, options = {}) {
        this.serviceName = serviceName;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.failureCount = 0;
        this.failureThreshold = options.failureThreshold || 3;
        this.resetTimeout = options.resetTimeout || 30000; // 30s
        this.lastFailureTime = 0;
        this.successCount = 0;
        this.halfOpenMaxAttempts = options.halfOpenMaxAttempts || 2;
    }

    canExecute() {
        if (this.state === 'CLOSED') return true;

        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.resetTimeout) {
                this.state = 'HALF_OPEN';
                this.successCount = 0;
                console.log(`🟡 Circuit HALF_OPEN for ${this.serviceName}`);
                return true;
            }
            return false;
        }

        // HALF_OPEN - সীমিত request allow
        return true;
    }

    recordSuccess() {
        if (this.state === 'HALF_OPEN') {
            this.successCount++;
            if (this.successCount >= this.halfOpenMaxAttempts) {
                this.state = 'CLOSED';
                this.failureCount = 0;
                console.log(`🟢 Circuit CLOSED for ${this.serviceName}`);
            }
        } else {
            this.failureCount = 0;
        }
    }

    recordFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();

        if (this.state === 'HALF_OPEN' || this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
            console.log(`🔴 Circuit OPEN for ${this.serviceName}`);
        }
    }
}

// ===============================
// Service Client with Timeout & Retry
// ===============================

class ServiceClient {
    constructor(baseUrl, options = {}) {
        this.baseUrl = baseUrl;
        this.timeout = options.timeout || 5000;
        this.retries = options.retries || 2;
        this.circuitBreaker = new CircuitBreaker(baseUrl, options.circuitBreaker);
        this.cache = new Map();
        this.cacheTTL = options.cacheTTL || 60000; // 60s
    }

    async call(endpoint, params = {}) {
        const cacheKey = `${endpoint}:${JSON.stringify(params)}`;
        const startTime = Date.now();

        // Circuit Breaker check
        if (!this.circuitBreaker.canExecute()) {
            const cached = this.getFromCache(cacheKey);
            if (cached) {
                return { success: true, data: cached, fromCache: true, responseTime: 0 };
            }
            return {
                success: false,
                error: `Circuit breaker OPEN: ${this.baseUrl}`,
                responseTime: 0
            };
        }

        // Retry logic
        let lastError;
        for (let attempt = 1; attempt <= this.retries; attempt++) {
            try {
                const url = new URL(endpoint, this.baseUrl);
                Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));

                const response = await fetch(url.toString(), {
                    signal: AbortSignal.timeout(this.timeout),
                    headers: { 'Accept': 'application/json' }
                });

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }

                const data = await response.json();
                const responseTime = Date.now() - startTime;

                // Success
                this.circuitBreaker.recordSuccess();
                this.setCache(cacheKey, data);

                return { success: true, data, responseTime, fromCache: false };
            } catch (error) {
                lastError = error;
                if (attempt < this.retries) {
                    await new Promise(r => setTimeout(r, 100 * attempt)); // Backoff
                }
            }
        }

        // All retries failed
        this.circuitBreaker.recordFailure();
        const responseTime = Date.now() - startTime;

        // Fallback to cache
        const cached = this.getFromCache(cacheKey);
        if (cached) {
            return { success: true, data: cached, fromCache: true, responseTime };
        }

        return { success: false, error: lastError.message, responseTime };
    }

    getFromCache(key) {
        const entry = this.cache.get(key);
        if (entry && (Date.now() - entry.timestamp) < this.cacheTTL) {
            return entry.data;
        }
        return null;
    }

    setCache(key, data) {
        this.cache.set(key, { data, timestamp: Date.now() });
    }
}

// ===============================
// API Composer - Product Page
// ===============================

class ProductPageComposer {
    constructor() {
        this.productService = new ServiceClient('http://product-service:8001', { timeout: 3000 });
        this.reviewService = new ServiceClient('http://review-service:8002', { timeout: 3000 });
        this.inventoryService = new ServiceClient('http://inventory-service:8003', { timeout: 2000 });
        this.pricingService = new ServiceClient('http://pricing-service:8004', { timeout: 2000 });
        this.shippingService = new ServiceClient('http://shipping-service:8005', { timeout: 3000 });
        this.sellerService = new ServiceClient('http://seller-service:8006', { timeout: 3000 });
    }

    /**
     * Product Page compose করা - Parallel + Sequential mix
     */
    async compose(productId, userId = null) {
        const startTime = Date.now();

        // Phase 1: Independent calls - PARALLEL
        const [productResult, reviewResult, inventoryResult, pricingResult] = await Promise.allSettled([
            this.productService.call(`/products/${productId}`),
            this.reviewService.call('/reviews', { product_id: productId }),
            this.inventoryService.call(`/stock/${productId}`),
            this.pricingService.call(`/price/${productId}`)
        ]);

        // Phase 2: Dependent calls - product data-র উপর নির্ভরশীল
        let shippingResult = { success: false, error: 'skipped' };
        let sellerResult = { success: false, error: 'skipped' };

        if (productResult.status === 'fulfilled' && productResult.value.success) {
            const productData = productResult.value.data;

            // Shipping ও Seller parallel-এ call করা যায় (দুটোই product data-র উপর নির্ভরশীল)
            const [shipping, seller] = await Promise.allSettled([
                this.shippingService.call('/estimate', {
                    product_id: productId,
                    weight: productData.weight,
                    user_id: userId
                }),
                this.sellerService.call(`/sellers/${productData.seller_id}`)
            ]);

            shippingResult = shipping.status === 'fulfilled' ? shipping.value : { success: false, error: shipping.reason?.message };
            sellerResult = seller.status === 'fulfilled' ? seller.value : { success: false, error: seller.reason?.message };
        }

        const totalTime = Date.now() - startTime;

        return this.buildResponse({
            product: productResult.status === 'fulfilled' ? productResult.value : { success: false, error: productResult.reason?.message },
            reviews: reviewResult.status === 'fulfilled' ? reviewResult.value : { success: false, error: reviewResult.reason?.message },
            inventory: inventoryResult.status === 'fulfilled' ? inventoryResult.value : { success: false, error: inventoryResult.reason?.message },
            pricing: pricingResult.status === 'fulfilled' ? pricingResult.value : { success: false, error: pricingResult.reason?.message },
            shipping: shippingResult,
            seller: sellerResult
        }, totalTime);
    }

    buildResponse(results, totalTime) {
        const response = {
            success: true,
            data: {},
            metadata: {
                totalTimeMs: totalTime,
                services: {},
                partialFailure: false,
                cachedServices: []
            }
        };

        for (const [key, result] of Object.entries(results)) {
            response.metadata.services[key] = {
                success: result.success,
                responseTimeMs: result.responseTime || 0,
                fromCache: result.fromCache || false,
                error: result.error || null
            };

            if (result.success) {
                response.data[key] = result.data;
                if (result.fromCache) {
                    response.metadata.cachedServices.push(key);
                }
            } else {
                response.data[key] = null;
                response.metadata.partialFailure = true;
            }
        }

        // Critical service (product) fail হলে
        if (!results.product?.success) {
            response.success = false;
            response.error = 'Product data unavailable - cannot render page';
        }

        return response;
    }
}

// ===============================
// BFF - bKash Mobile App Dashboard
// ===============================

class BKashDashboardBFF {
    constructor() {
        this.accountService = new ServiceClient('http://account-service:8010', { timeout: 2000 });
        this.transactionService = new ServiceClient('http://transaction-service:8011', { timeout: 3000 });
        this.offerService = new ServiceClient('http://offer-service:8012', { timeout: 2000 });
        this.billService = new ServiceClient('http://bill-service:8013', { timeout: 3000 });
        this.notificationService = new ServiceClient('http://notification-service:8014', { timeout: 2000 });
    }

    /**
     * bKash Dashboard - সকল ডেটা একসাথে compose করা
     */
    async getDashboard(userId) {
        const startTime = Date.now();

        // সকল call parallel - কোনোটি অন্যটির উপর নির্ভরশীল নয়
        const [balance, transactions, offers, bills, notifications] = await Promise.allSettled([
            this.accountService.call(`/accounts/${userId}/balance`),
            this.transactionService.call('/transactions', { user_id: userId, limit: '5' }),
            this.offerService.call('/offers', { user_id: userId }),
            this.billService.call('/bills/pending', { user_id: userId }),
            this.notificationService.call('/notifications/unread', { user_id: userId })
        ]);

        return {
            success: true,
            data: {
                balance: this.extractData(balance, { amount: 0, currency: 'BDT' }),
                recentTransactions: this.extractData(transactions, []),
                activeOffers: this.extractData(offers, []),
                pendingBills: this.extractData(bills, []),
                unreadNotifications: this.extractData(notifications, { count: 0, items: [] })
            },
            metadata: {
                totalTimeMs: Date.now() - startTime,
                anyFailure: [balance, transactions, offers, bills, notifications]
                    .some(r => r.status !== 'fulfilled' || !r.value?.success)
            }
        };
    }

    extractData(promiseResult, defaultValue) {
        if (promiseResult.status === 'fulfilled' && promiseResult.value?.success) {
            return promiseResult.value.data;
        }
        return defaultValue; // Graceful degradation
    }
}

// ===============================
// Prothom Alo Homepage Composer
// ===============================

class ProthomAloHomepageComposer {
    constructor() {
        this.newsService = new ServiceClient('http://news-service:8020', { timeout: 3000 });
        this.adService = new ServiceClient('http://ad-service:8021', { timeout: 1000 }); // বিজ্ঞাপন দ্রুত লোড হওয়া উচিত
        this.trendingService = new ServiceClient('http://trending-service:8022', { timeout: 2000 });
        this.weatherService = new ServiceClient('http://weather-service:8023', { timeout: 2000 });
        this.sportsService = new ServiceClient('http://sports-service:8024', { timeout: 2000 });
    }

    async getHomepage(userLocation = 'dhaka') {
        const [news, ads, trending, weather, sports] = await Promise.allSettled([
            this.newsService.call('/latest', { limit: '10', category: 'all' }),
            this.adService.call('/homepage-ads', { location: userLocation }),
            this.trendingService.call('/trending', { limit: '5' }),
            this.weatherService.call('/current', { city: userLocation }),
            this.sportsService.call('/live-scores')
        ]);

        return {
            // সংবাদ critical - এটা fail হলে পুরো page fail
            news: this.extractOrThrow(news, 'News service unavailable'),
            // বাকিগুলো optional - fail হলেও page দেখাবে
            ads: this.extractSafe(ads, []),
            trending: this.extractSafe(trending, []),
            weather: this.extractSafe(weather, null),
            sports: this.extractSafe(sports, [])
        };
    }

    extractOrThrow(result, errorMessage) {
        if (result.status === 'fulfilled' && result.value?.success) {
            return result.value.data;
        }
        throw new Error(errorMessage);
    }

    extractSafe(result, defaultValue) {
        if (result.status === 'fulfilled' && result.value?.success) {
            return result.value.data;
        }
        return defaultValue;
    }
}

// ===============================
// Response Caching Layer
// ===============================

class ComposedResponseCache {
    constructor(options = {}) {
        this.cache = new Map();
        this.defaultTTL = options.ttl || 30000; // 30s
        this.maxSize = options.maxSize || 1000;
    }

    get(key) {
        const entry = this.cache.get(key);
        if (!entry) return null;

        if (Date.now() - entry.timestamp > entry.ttl) {
            this.cache.delete(key);
            return null;
        }

        return entry.data;
    }

    set(key, data, ttl = null) {
        // LRU eviction
        if (this.cache.size >= this.maxSize) {
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }

        this.cache.set(key, {
            data,
            timestamp: Date.now(),
            ttl: ttl || this.defaultTTL
        });
    }

    /**
     * Composed response cache করা
     * বিভিন্ন অংশের জন্য ভিন্ন TTL
     */
    setComposed(key, response, ttlMap = {}) {
        this.set(key, {
            ...response,
            cachedAt: Date.now(),
            ttlMap
        });
    }
}

// ===============================
// Express.js Route Example
// ===============================

/*
const express = require('express');
const app = express();

const productComposer = new ProductPageComposer();
const bkashBFF = new BKashDashboardBFF();
const responseCache = new ComposedResponseCache({ ttl: 15000 });

// Product Page API
app.get('/api/products/:id', async (req, res) => {
    const { id } = req.params;
    const cacheKey = `product:${id}`;

    // Cache check
    const cached = responseCache.get(cacheKey);
    if (cached) {
        return res.json({ ...cached, _cached: true });
    }

    try {
        const result = await productComposer.compose(id, req.user?.id);
        responseCache.set(cacheKey, result);
        res.json(result);
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});

// bKash Dashboard API
app.get('/api/bkash/dashboard', async (req, res) => {
    try {
        const result = await bkashBFF.getDashboard(req.user.id);
        res.json(result);
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});

app.listen(3000, () => console.log('API Composer running on port 3000'));
*/

// ===============================
// Demo
// ===============================

async function demo() {
    console.log('=== 🛒 E-commerce Product Page Composition Demo ===\n');

    const composer = new ProductPageComposer();

    try {
        const result = await composer.compose('PROD-12345', 'USER-789');
        console.log('Product Page Result:');
        console.log(JSON.stringify(result.metadata, null, 2));
    } catch (e) {
        console.log('Demo mode - services not running:', e.message);
    }

    console.log('\n=== 💰 bKash Dashboard Composition Demo ===\n');

    const bkash = new BKashDashboardBFF();
    try {
        const dashboard = await bkash.getDashboard('USER-001');
        console.log('Dashboard Result:');
        console.log(JSON.stringify(dashboard.metadata, null, 2));
    } catch (e) {
        console.log('Demo mode - services not running:', e.message);
    }
}

demo();
```

---

## 🔄 CQRS-এর সাথে তুলনা

### API Composition vs CQRS

```
┌─────────────────────────────────────────────────────────────────┐
│        API COMPOSITION vs CQRS APPROACH                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  API COMPOSITION:                                                │
│  ────────────────                                                │
│  Request time-এ একাধিক সার্ভিস থেকে ডেটা জোড়া লাগায়           │
│                                                                   │
│  Client ──► Composer ──┬──► Service A ──► DB A                   │
│                        ├──► Service B ──► DB B                   │
│                        └──► Service C ──► DB C                   │
│                                                                   │
│  সুবিধা: Simple, real-time data, no data duplication            │
│  অসুবিধা: Slow (multiple calls), complex error handling         │
│                                                                   │
│  ═══════════════════════════════════════════════════════════════  │
│                                                                   │
│  CQRS (Pre-computed View):                                       │
│  ─────────────────────────                                       │
│  আগে থেকেই ডেটা join করে একটি read-optimized view তৈরি রাখে    │
│                                                                   │
│  Service A ──┐                    ┌──► Read Model ──► Client     │
│  Service B ──┼──► Event Bus ──►  View │  (pre-joined │            │
│  Service C ──┘    (Kafka)        └──── │   data)     │            │
│                                                                   │
│  সুবিধা: Fast reads (single query), no runtime composition      │
│  অসুবিধা: Complex, eventual consistency, data duplication       │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  কখন কোনটি ব্যবহার করবেন?                                       │
│                                                                   │
│  API Composition ভালো যখন:         │  CQRS ভালো যখন:            │
│  ─────────────────────────────      │  ──────────────────────    │
│  • ডেটা সবসময় fresh লাগবে         │  • খুব দ্রুত read লাগবে    │
│  • Query pattern simple             │  • Complex joins/filters   │
│  • ডেটার পরিমাণ কম                 │  • High read volume        │
│  • Eventually consistent চলবে না   │  • Eventual consistency OK │
│  • দ্রুত implement করতে হবে        │  • Large dataset           │
│                                                                   │
│  Daraz Product Page → API Composition (fresh price/stock)        │
│  Prothom Alo Search → CQRS (pre-indexed, fast search)           │
│  bKash Statement → CQRS (pre-computed monthly summary)           │
│  bKash Balance → API Composition (must be real-time)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚡ Performance Considerations

```
┌───────────────────────────────────────────────────────────────┐
│              PERFORMANCE OPTIMIZATION                           │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  1. PARALLEL EXECUTION:                                        │
│     - Independent calls সবসময় parallel করুন                  │
│     - Promise.all / Promise.allSettled ব্যবহার করুন           │
│                                                                │
│  2. TIMEOUT STRATEGY:                                          │
│     ┌──────────────────┬──────────────────┐                   │
│     │ Service          │ Timeout          │                   │
│     ├──────────────────┼──────────────────┤                   │
│     │ Critical (Product)│ 3000ms          │                   │
│     │ Important (Price) │ 2000ms          │                   │
│     │ Optional (Reviews)│ 2000ms          │                   │
│     │ Non-critical (Ads)│ 1000ms          │                   │
│     └──────────────────┴──────────────────┘                   │
│                                                                │
│  3. CACHING LAYERS:                                            │
│     Client ──► CDN ──► Composer Cache ──► Service Cache       │
│     (5min)    (1min)    (30s)             (varies)            │
│                                                                │
│  4. DATA SIZE OPTIMIZATION:                                    │
│     - GraphQL-style field selection                            │
│     - Pagination for lists                                     │
│     - Compression (gzip)                                       │
│                                                                │
│  5. CONNECTION POOLING:                                         │
│     - HTTP keep-alive                                          │
│     - Connection reuse                                         │
│     - gRPC for internal communication                         │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

| পরিস্থিতি | কারণ |
|-----------|------|
| একটি পেজে একাধিক সার্ভিসের ডেটা দরকার | Client-কে multiple call থেকে বাঁচায় |
| Mobile app-এর জন্য optimized API দরকার | BFF pattern-এ কম ডেটা পাঠানো |
| Frontend team-কে backend complexity থেকে আলাদা রাখতে | একটি clean API দিতে |
| Cross-service data join করতে হবে | Microservice-এ direct DB join সম্ভব নয় |
| বিভিন্ন client-এর জন্য ভিন্ন response দরকার | Web vs Mobile vs TV |
| Network round-trip কমাতে | একটি call-এ সব ডেটা |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|------|
| সব ডেটা একই সার্ভিসে আছে | Composition-এর দরকার নেই |
| Real-time consistency critical | CQRS বা Saga ভালো |
| খুব complex join/filter operations | CQRS materialized view ব্যবহার করুন |
| Write operations (data modification) | Composition শুধু read-এর জন্য |
| High volume batch processing | Event-driven approach ভালো |
| Single responsibility সার্ভিস | Unnecessary coupling তৈরি করবে |

### ⚖️ তুলনা সারণি:

```
┌───────────────────┬────────────────────┬───────────────────────┐
│    বৈশিষ্ট্য      │  API Composition   │    API Gateway        │
├───────────────────┼────────────────────┼───────────────────────┤
│ মূল কাজ          │ Data aggregation   │ Request routing       │
│ Logic            │ Business logic     │ Infrastructure logic  │
│ Coupling         │ Service-aware      │ Service-agnostic      │
│ Data transform   │ Complex joins      │ Simple pass-through   │
│ Error handling   │ Partial response   │ Pass error to client  │
│ উদাহরণ          │ Product page BFF   │ Kong, AWS API GW      │
└───────────────────┴────────────────────┴───────────────────────┘
```

### 💡 সেরা অনুশীলন (Best Practices):

1. **সবসময় timeout সেট করুন** - কোনো সার্ভিসের জন্য অনির্দিষ্টকাল অপেক্ষা করবেন না
2. **Partial failure handle করুন** - একটি সার্ভিস fail মানে পুরো response fail নয়
3. **Circuit breaker ব্যবহার করুন** - dead সার্ভিসে বারবার call করবেন না
4. **Response cache করুন** - একই ডেটা বারবার compose করবেন না
5. **Parallel calls maximize করুন** - independent calls সবসময় parallel-এ
6. **Critical vs Optional সার্ভিস আলাদা করুন** - critical fail = error, optional fail = degraded response
7. **Monitoring ও tracing** - কোন সার্ভিস slow তা track করুন
8. **BFF per client** - Mobile, Web, TV-এর জন্য আলাদা BFF

---

## 🔗 সম্পর্কিত প্যাটার্ন

- **CQRS** - Pre-computed views দিয়ে fast reads
- **API Gateway** - Routing ও authentication
- **Circuit Breaker** - Failed সার্ভিসে call বন্ধ করা
- **BFF (Backend for Frontend)** - Client-specific composition
- **Service Mesh** - Inter-service communication management
- **Event Sourcing** - CQRS-এর সাথে ব্যবহার
- **Saga Pattern** - Cross-service write operations
