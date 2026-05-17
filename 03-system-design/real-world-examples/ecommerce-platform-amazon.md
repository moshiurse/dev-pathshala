# 🛒 ই-কমার্স প্ল্যাটফর্ম সিস্টেম ডিজাইন (Amazon)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Amazon, Alibaba, Daraz, Shopify
> **বাংলাদেশ কনটেক্সট**: Daraz, Chaldal, Evaly (lessons learned), bKash/Nagad Payment

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| ক্যাটাগরি | ফিচার | বিবরণ |
|-----------|--------|--------|
| Product | Catalog Management | প্রোডাক্ট CRUD, ক্যাটাগরি, ভ্যারিয়েন্ট (সাইজ, কালার) |
| Search | Full-text + Filter | নাম, ক্যাটাগরি, প্রাইস রেঞ্জ, রেটিং ফিল্টার |
| Cart | Shopping Cart | Add/Remove/Update quantity, Guest + Logged-in cart |
| Order | Order Pipeline | Place → Confirm → Ship → Deliver → Complete |
| Payment | Multi-gateway | bKash, Nagad, Card, COD (Cash on Delivery) |
| Inventory | Stock Management | Real-time stock tracking, warehouse allocation |
| Review | Rating & Review | Product review, seller rating, photo review |
| Seller | Seller Dashboard | Multi-vendor marketplace, commission tracking |
| Notification | Multi-channel | SMS, Email, Push, In-app notification |

### Non-Functional Requirements

```
✅ Availability:     99.99% uptime (52 min downtime/year)
✅ Latency:          Search < 100ms, Checkout < 500ms
✅ Throughput:       10K orders/sec (flash sale peak)
✅ Consistency:      Payment → Strong consistency (ACID)
                     Product catalog → Eventual consistency (OK)
✅ Scalability:      Handle 10x traffic spike (11.11 sale)
✅ Data Durability:  Zero order/payment data loss
```

### 🇧🇩 বাংলাদেশ কনটেক্সট

```
📱 মোবাইল-ফার্স্ট:    90%+ ইউজার মোবাইল থেকে আসে
💰 পেমেন্ট:           bKash (50%), Nagad (30%), COD (15%), Card (5%)
🚚 ডেলিভারি:          Pathao, Paperfly, RedX, SA Paribahan
🌐 কানেক্টিভিটি:      2G/3G তে কাজ করতে হবে (low bandwidth optimization)
📍 অ্যাড্রেস:          বাংলাদেশে structured address নেই → free-text + area
💳 Evaly লেসন:        Advance payment without fulfillment guarantee = disaster
```

---

## 📊 Back-of-envelope Estimation

### ট্রাফিক এস্টিমেশন (Daraz Bangladesh Scale)

```
Daily Active Users (DAU):          2M
Peak concurrent users:             200K (flash sale)
Product catalog size:              10M products
Average order value:               ৳1,500
Daily orders (normal):             100K
Daily orders (11.11 sale):         1M (10x spike)

Read:Write ratio:                  100:1 (browse vs buy)
Search queries/sec (normal):       5,000 QPS
Search queries/sec (peak):         50,000 QPS
```

### স্টোরেজ এস্টিমেশন

```
Product data:
  - 10M products × 5KB avg = 50 GB (metadata)
  - 10M products × 10 images × 500KB = 50 TB (images)

Order data:
  - 100K orders/day × 2KB = 200 MB/day
  - 5 years retention = 365 GB

User data:
  - 20M users × 2KB = 40 GB

Cart data (Redis):
  - 2M active carts × 1KB = 2 GB (fits in memory!)

Total Storage: ~55 TB (mostly images → CDN)
```

### ব্যান্ডউইথ এস্টিমেশন

```
Incoming (writes):   200 MB/s (images, order data)
Outgoing (reads):    20 GB/s (product pages, images)
                     → CDN handles 95% of this
Net origin traffic:  ~1 GB/s
```

---

## 🏗️ High-Level Design

### সিস্টেম আর্কিটেকচার

```
                         ┌─────────────────────────┐
                         │     Mobile App / Web     │
                         │   (React/Flutter/Next)   │
                         └────────────┬────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │          CDN            │
                         │  (CloudFront/BunnyCDN)  │
                         │  Images, Static Assets  │
                         └────────────┬────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │     Load Balancer       │
                         │      (Nginx/ALB)        │
                         └────────────┬────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │     API Gateway         │
                         │  (Kong/Laravel Gateway) │
                         │  Rate Limit, Auth, Log  │
                         └────────────┬────────────┘
                                      │
              ┌───────────┬───────────┼───────────┬───────────┐
              │           │           │           │           │
    ┌─────────▼──┐ ┌──────▼────┐ ┌────▼─────┐ ┌──▼────────┐ ┌▼──────────┐
    │  Product   │ │   Cart    │ │  Order   │ │  Payment  │ │ Inventory │
    │  Service   │ │  Service  │ │  Service │ │  Service  │ │  Service  │
    │  (Node.js) │ │  (Node.js)│ │  (PHP)   │ │  (PHP)    │ │  (PHP)    │
    └─────┬──────┘ └─────┬─────┘ └────┬─────┘ └─────┬─────┘ └─────┬─────┘
          │               │            │             │             │
    ┌─────▼──────┐  ┌────▼────┐  ┌────▼─────┐ ┌────▼─────┐ ┌─────▼─────┐
    │Elasticsearch│  │  Redis  │  │PostgreSQL│ │PostgreSQL│ │PostgreSQL │
    │ + MongoDB  │  │ Cluster │  │ (Orders) │ │(Payments)│ │(Inventory)│
    └────────────┘  └─────────┘  └──────────┘ └──────────┘ └───────────┘
                                       │
                              ┌─────────▼──────────┐
                              │   Message Queue    │
                              │  (RabbitMQ/Kafka)  │
                              └────────────────────┘
                                       │
                    ┌──────────────┬────┴─────┬──────────────┐
                    │              │          │              │
              ┌─────▼─────┐ ┌─────▼────┐ ┌───▼─────┐ ┌─────▼──────┐
              │Notification│ │  Email   │ │Analytics│ │  Delivery  │
              │  Service   │ │  Service │ │ Service │ │  Tracking  │
              └────────────┘ └──────────┘ └─────────┘ └────────────┘
```

### সার্ভিস কমিউনিকেশন ফ্লো

```
[User Browse]                    [User Checkout]
     │                                │
     ▼                                ▼
Product Service ──sync──▶ API    Order Service
     │                         ┌──────┼──────┐
     ▼                         │      │      │
Elasticsearch              Inventory Payment Notification
(< 100ms response)          (Saga Pattern - async events)
```

---

## 💻 Detailed Design

### 1. 🔍 Product Catalog & Search Service (Node.js)

#### ডেটাবেস স্কিমা (MongoDB)

```javascript
// Product Schema - MongoDB তে রাখছি কারণ:
// - Flexible schema (প্রোডাক্ট ক্যাটাগরি অনুযায়ী different attributes)
// - Read-heavy workload
// - Denormalized data for fast reads

const ProductSchema = {
    _id: ObjectId,
    sku: "PROD-12345",
    name: "Samsung Galaxy S24 Ultra",
    name_bn: "স্যামসাং গ্যালাক্সি এস২৪ আল্ট্রা",
    slug: "samsung-galaxy-s24-ultra",
    seller_id: ObjectId,
    category: ["Electronics", "Mobile", "Samsung"],
    price: {
        amount: 149999,          // পয়সায় রাখি (integer math)
        currency: "BDT",
        discount_percent: 10,
        discount_price: 134999
    },
    inventory: {
        total_stock: 500,
        reserved: 45,
        available: 455,
        warehouse: "dhaka-gulshan"
    },
    attributes: {
        color: ["Black", "Purple", "Titanium"],
        storage: ["256GB", "512GB", "1TB"],
        ram: "12GB"
    },
    images: [
        { url: "cdn.example.com/products/s24-1.webp", is_primary: true },
        { url: "cdn.example.com/products/s24-2.webp", is_primary: false }
    ],
    rating: { average: 4.5, count: 1250 },
    status: "active",
    created_at: ISODate(),
    updated_at: ISODate()
};
```

#### Elasticsearch Integration (Node.js/Express)

```javascript
// services/product-search.service.js
const { Client } = require('@elastic/elasticsearch');
const client = new Client({ node: 'http://elasticsearch:9200' });

class ProductSearchService {
    
    // প্রোডাক্ট সার্চ - Elasticsearch ব্যবহার করছি কারণ:
    // - Full-text search with Bengali + English
    // - Faceted filtering (price, category, rating)
    // - Fuzzy matching (typo tolerance)
    // - < 50ms response time at scale
    
    async search(query, filters = {}, page = 1, limit = 20) {
        const must = [];
        const filter = [];

        // Multi-match: name, description, category তে সার্চ
        if (query) {
            must.push({
                multi_match: {
                    query,
                    fields: ['name^3', 'name_bn^3', 'description', 'category^2'],
                    fuzziness: 'AUTO',
                    type: 'best_fields'
                }
            });
        }

        // Price range filter
        if (filters.min_price || filters.max_price) {
            filter.push({
                range: {
                    'price.amount': {
                        gte: filters.min_price || 0,
                        lte: filters.max_price || 9999999
                    }
                }
            });
        }

        // Category filter
        if (filters.category) {
            filter.push({ term: { 'category.keyword': filters.category } });
        }

        // Rating filter
        if (filters.min_rating) {
            filter.push({ range: { 'rating.average': { gte: filters.min_rating } } });
        }

        // In-stock only
        if (filters.in_stock) {
            filter.push({ range: { 'inventory.available': { gt: 0 } } });
        }

        const response = await client.search({
            index: 'products',
            body: {
                from: (page - 1) * limit,
                size: limit,
                query: {
                    bool: { must, filter }
                },
                sort: this._buildSort(filters.sort_by),
                aggs: {
                    categories: { terms: { field: 'category.keyword', size: 20 } },
                    price_ranges: {
                        range: {
                            field: 'price.amount',
                            ranges: [
                                { to: 50000, key: 'under_500' },
                                { from: 50000, to: 200000, key: '500_to_2000' },
                                { from: 200000, key: 'above_2000' }
                            ]
                        }
                    },
                    avg_rating: { avg: { field: 'rating.average' } }
                }
            }
        });

        return {
            products: response.hits.hits.map(hit => ({
                ...hit._source,
                _score: hit._score
            })),
            total: response.hits.total.value,
            aggregations: response.aggregations,
            page,
            total_pages: Math.ceil(response.hits.total.value / limit)
        };
    }

    _buildSort(sortBy) {
        const sortMap = {
            'price_asc': [{ 'price.amount': 'asc' }],
            'price_desc': [{ 'price.amount': 'desc' }],
            'rating': [{ 'rating.average': 'desc' }],
            'newest': [{ 'created_at': 'desc' }],
            'popular': [{ 'rating.count': 'desc' }]
        };
        return sortMap[sortBy] || [{ _score: 'desc' }];
    }
}

module.exports = new ProductSearchService();
```

### 2. 🛒 Cart Service (Redis-based)

```javascript
// services/cart.service.js
const Redis = require('ioredis');
const redis = new Redis({ host: 'redis-cluster', port: 6379 });

class CartService {
    
    // Redis তে Cart রাখার কারণ:
    // - Extremely fast read/write (< 1ms)
    // - TTL support (abandoned cart cleanup)
    // - Cart data is temporary, loss acceptable
    // - 2M active carts × 1KB = 2GB (fits in memory)
    
    _cartKey(userId) {
        return `cart:${userId}`;
    }

    async addItem(userId, productId, quantity, variant = {}) {
        const cartKey = this._cartKey(userId);
        const itemKey = `${productId}:${JSON.stringify(variant)}`;
        
        // HSET: product_id → {quantity, variant, added_at}
        const existing = await redis.hget(cartKey, itemKey);
        const item = existing ? JSON.parse(existing) : { quantity: 0 };
        
        item.quantity += quantity;
        item.variant = variant;
        item.product_id = productId;
        item.added_at = item.added_at || Date.now();
        item.updated_at = Date.now();

        await redis.hset(cartKey, itemKey, JSON.stringify(item));
        
        // 7 দিন পর auto-expire (abandoned cart)
        await redis.expire(cartKey, 7 * 24 * 60 * 60);
        
        return this.getCart(userId);
    }

    async getCart(userId) {
        const cartKey = this._cartKey(userId);
        const items = await redis.hgetall(cartKey);
        
        const cartItems = Object.values(items).map(item => JSON.parse(item));
        
        // Product service থেকে latest price/stock fetch
        const enrichedItems = await this._enrichWithProductData(cartItems);
        
        return {
            user_id: userId,
            items: enrichedItems,
            total: enrichedItems.reduce((sum, item) => 
                sum + (item.price * item.quantity), 0
            ),
            item_count: enrichedItems.reduce((sum, item) => 
                sum + item.quantity, 0
            )
        };
    }

    async removeItem(userId, productId, variant = {}) {
        const cartKey = this._cartKey(userId);
        const itemKey = `${productId}:${JSON.stringify(variant)}`;
        await redis.hdel(cartKey, itemKey);
        return this.getCart(userId);
    }

    // Guest cart → User cart merge (login এর সময়)
    async mergeCarts(guestId, userId) {
        const guestCart = await redis.hgetall(this._cartKey(guestId));
        
        for (const [key, value] of Object.entries(guestCart)) {
            const existing = await redis.hget(this._cartKey(userId), key);
            if (!existing) {
                await redis.hset(this._cartKey(userId), key, value);
            }
        }
        
        await redis.del(this._cartKey(guestId));
        return this.getCart(userId);
    }
}

module.exports = new CartService();
```

### 3. 📦 Order Processing Pipeline (PHP/Laravel - Saga Pattern)

```
Order Processing Flow (Saga Pattern):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │  Create  │───▶│ Reserve  │───▶│  Process │───▶│  Confirm │
 │  Order   │    │ Inventory│    │  Payment │    │  Order   │
 └──────────┘    └──────────┘    └──────────┘    └──────────┘
      │               │               │               │
      │ Compensate    │ Compensate    │ Compensate    │
      ▼               ▼               ▼               ▼
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │  Cancel  │◀───│ Release  │◀───│  Refund  │◀───│  Cancel  │
 │  Order   │    │  Stock   │    │  Payment │    │  Confirm │
 └──────────┘    └──────────┘    └──────────┘    └──────────┘

 যদি কোনো step fail করে → পেছনের সব step compensate হবে
```

```php
<?php
// app/Services/OrderSagaOrchestrator.php

namespace App\Services;

use App\Models\Order;
use App\Events\OrderCreated;
use App\Exceptions\SagaFailedException;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class OrderSagaOrchestrator
{
    private InventoryService $inventoryService;
    private PaymentService $paymentService;
    private NotificationService $notificationService;

    public function __construct(
        InventoryService $inventoryService,
        PaymentService $paymentService,
        NotificationService $notificationService
    ) {
        $this->inventoryService = $inventoryService;
        $this->paymentService = $paymentService;
        $this->notificationService = $notificationService;
    }

    /**
     * Order তৈরির পুরো Saga execute করে
     * প্রতিটি step fail হলে compensating actions চালায়
     */
    public function placeOrder(array $orderData): Order
    {
        $compensations = [];

        try {
            // Step 1: Order create করো (PENDING status)
            $order = $this->createOrder($orderData);
            $compensations[] = fn() => $this->cancelOrder($order);

            // Step 2: Inventory reserve করো
            $reservation = $this->inventoryService->reserveStock(
                $order->items,
                $order->id
            );
            $compensations[] = fn() => $this->inventoryService->releaseStock(
                $reservation->id
            );

            // Step 3: Payment process করো
            $payment = $this->processPayment($order, $orderData['payment']);
            $compensations[] = fn() => $this->paymentService->refund(
                $payment->transaction_id
            );

            // Step 4: Order confirm করো
            $order->update([
                'status' => 'confirmed',
                'payment_id' => $payment->id,
                'reservation_id' => $reservation->id,
            ]);

            // Step 5: Notification পাঠাও (async - fail হলেও order cancel না)
            event(new OrderCreated($order));

            Log::info("Order placed successfully", ['order_id' => $order->id]);
            return $order->fresh();

        } catch (\Exception $e) {
            // Saga failed - compensate সব step
            Log::error("Order saga failed, compensating", [
                'error' => $e->getMessage(),
                'order_data' => $orderData
            ]);

            $this->compensate($compensations);
            
            throw new SagaFailedException(
                "Order placement failed: " . $e->getMessage(),
                previous: $e
            );
        }
    }

    private function createOrder(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            $order = Order::create([
                'user_id' => $data['user_id'],
                'status' => 'pending',
                'shipping_address' => $data['shipping_address'],
                'total_amount' => $data['total_amount'],
                'delivery_method' => $data['delivery_method'], // pathao/paperfly/redx
                'payment_method' => $data['payment_method'],   // bkash/nagad/cod
                'idempotency_key' => $data['idempotency_key'], // duplicate order prevention
            ]);

            foreach ($data['items'] as $item) {
                $order->items()->create([
                    'product_id' => $item['product_id'],
                    'variant' => $item['variant'] ?? null,
                    'quantity' => $item['quantity'],
                    'unit_price' => $item['price'],
                    'subtotal' => $item['price'] * $item['quantity'],
                ]);
            }

            return $order;
        });
    }

    private function processPayment(Order $order, array $paymentData): object
    {
        // বাংলাদেশ পেমেন্ট গেটওয়ে integration
        return match ($paymentData['method']) {
            'bkash' => $this->paymentService->chargeBkash(
                amount: $order->total_amount,
                agreement_id: $paymentData['agreement_id'],
                order_id: $order->id
            ),
            'nagad' => $this->paymentService->chargeNagad(
                amount: $order->total_amount,
                order_id: $order->id
            ),
            'cod' => $this->paymentService->createCodPayment(
                amount: $order->total_amount,
                order_id: $order->id
            ),
            'card' => $this->paymentService->chargeCard(
                amount: $order->total_amount,
                token: $paymentData['card_token'],
                order_id: $order->id
            ),
            default => throw new \InvalidArgumentException("Unknown payment: {$paymentData['method']}")
        };
    }

    /**
     * Compensating actions - reverse order এ execute করো
     */
    private function compensate(array $compensations): void
    {
        foreach (array_reverse($compensations) as $compensation) {
            try {
                $compensation();
            } catch (\Exception $e) {
                // Compensation fail → log করো, manual intervention দরকার
                Log::critical("Compensation failed!", ['error' => $e->getMessage()]);
            }
        }
    }

    private function cancelOrder(Order $order): void
    {
        $order->update(['status' => 'cancelled']);
    }
}
```

### 4. 📦 Inventory Management (Distributed Locking)

```php
<?php
// app/Services/InventoryService.php

namespace App\Services;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\DB;
use App\Models\Inventory;
use App\Models\StockReservation;
use App\Exceptions\InsufficientStockException;

class InventoryService
{
    /**
     * Flash Sale সমস্যা:
     * 1000 items, 100K buyers একসাথে buy করতে চায়
     * Redis distributed lock ব্যবহার করে race condition prevent করি
     * 
     * Without lock: 1000 items, 1500 sold (overselling!)
     * With lock:    1000 items, 1000 sold (correct!)
     */
    public function reserveStock(iterable $items, string $orderId): StockReservation
    {
        $reservedItems = [];

        foreach ($items as $item) {
            $lockKey = "inventory_lock:{$item->product_id}:{$item->variant}";
            
            // Redis distributed lock (Redlock algorithm)
            $lock = Redis::lock($lockKey, 10); // 10 sec timeout

            try {
                // 5 সেকেন্ড wait করবে lock পেতে
                if ($lock->get(fn() => $this->doReserve($item, $orderId))) {
                    $reservedItems[] = $item;
                } else {
                    // Lock পাইনি → timeout → rollback previous reservations
                    $this->rollbackReservations($reservedItems, $orderId);
                    throw new InsufficientStockException(
                        "Could not acquire lock for product: {$item->product_id}"
                    );
                }
            } finally {
                $lock->release();
            }
        }

        return StockReservation::create([
            'order_id' => $orderId,
            'items' => $reservedItems,
            'status' => 'reserved',
            'expires_at' => now()->addMinutes(15), // 15 min এ pay না করলে release
        ]);
    }

    private function doReserve($item, string $orderId): bool
    {
        $inventory = Inventory::where('product_id', $item->product_id)
            ->where('variant', $item->variant)
            ->lockForUpdate()  // DB-level pessimistic lock
            ->first();

        if (!$inventory || $inventory->available < $item->quantity) {
            throw new InsufficientStockException(
                "Stock not available for {$item->product_id}. " .
                "Requested: {$item->quantity}, Available: " . ($inventory->available ?? 0)
            );
        }

        // Stock reserve করো (available কমাও, reserved বাড়াও)
        $inventory->decrement('available', $item->quantity);
        $inventory->increment('reserved', $item->quantity);

        return true;
    }

    public function releaseStock(string $reservationId): void
    {
        $reservation = StockReservation::findOrFail($reservationId);
        
        foreach ($reservation->items as $item) {
            Inventory::where('product_id', $item['product_id'])
                ->where('variant', $item['variant'])
                ->increment('available', $item['quantity']);
            
            Inventory::where('product_id', $item['product_id'])
                ->where('variant', $item['variant'])
                ->decrement('reserved', $item['quantity']);
        }

        $reservation->update(['status' => 'released']);
    }
}
```

### 5. 💳 Payment Service (Idempotency + Double-Entry)

```php
<?php
// app/Services/PaymentService.php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use App\Models\Payment;
use App\Models\LedgerEntry;

class PaymentService
{
    /**
     * Idempotency Key:
     * - Client retry করলে duplicate charge হবে না
     * - Network timeout হলে safely retry করা যাবে
     * - Evaly এর সমস্যা: Payment নিয়েছে কিন্তু track করেনি!
     */
    public function chargeBkash(int $amount, string $agreementId, string $orderId): Payment
    {
        $idempotencyKey = "payment:{$orderId}:bkash";

        // Idempotent check - আগে process হয়ে থাকলে same result return
        $existing = Payment::where('idempotency_key', $idempotencyKey)->first();
        if ($existing) {
            return $existing;
        }

        return DB::transaction(function () use ($amount, $agreementId, $orderId, $idempotencyKey) {
            // bKash API call
            $bkashResponse = $this->callBkashApi([
                'amount' => $amount / 100, // পয়সা → টাকা convert
                'currency' => 'BDT',
                'intent' => 'sale',
                'merchantInvoiceNumber' => $orderId,
                'agreementID' => $agreementId,
            ]);

            // Payment record তৈরি
            $payment = Payment::create([
                'order_id' => $orderId,
                'method' => 'bkash',
                'amount' => $amount,
                'currency' => 'BDT',
                'status' => $bkashResponse['status'] === 'Completed' ? 'success' : 'failed',
                'transaction_id' => $bkashResponse['trxID'],
                'gateway_response' => $bkashResponse,
                'idempotency_key' => $idempotencyKey,
            ]);

            // Double-entry bookkeeping (হিসাব সঠিক রাখতে)
            if ($payment->status === 'success') {
                $this->recordDoubleEntry($payment);
            }

            return $payment;
        });
    }

    /**
     * Double-Entry Bookkeeping:
     * প্রতিটি টাকার movement এ দুইটা entry:
     * - Debit: কোথা থেকে গেল
     * - Credit: কোথায় গেল
     * 
     * এটা না করলে Evaly এর মতো হিসাব মিলবে না!
     */
    private function recordDoubleEntry(Payment $payment): void
    {
        // Customer account → Debit (টাকা customer থেকে গেছে)
        LedgerEntry::create([
            'payment_id' => $payment->id,
            'account' => "customer:{$payment->order->user_id}",
            'type' => 'debit',
            'amount' => $payment->amount,
            'description' => "Order payment #{$payment->order_id}",
        ]);

        // Platform escrow → Credit (টাকা platform এ এসেছে)
        LedgerEntry::create([
            'payment_id' => $payment->id,
            'account' => 'platform:escrow',
            'type' => 'credit',
            'amount' => $payment->amount,
            'description' => "Order payment #{$payment->order_id}",
        ]);

        // Note: Seller কে পাবে delivery confirm হলে
        // escrow → seller transfer হবে তখন
    }
}
```

### 6. 🗄️ Database Design Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATABASE STRATEGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    কেন MongoDB?                               │
│  │   MongoDB   │    - Flexible schema (ভিন্ন ভিন্ন attribute)  │
│  │  (Products) │    - Read-heavy (100:1 ratio)                  │
│  └─────────────┘    - Horizontal scaling (sharding)             │
│                                                                 │
│  ┌─────────────┐    কেন PostgreSQL?                             │
│  │ PostgreSQL  │    - ACID compliance (টাকার হিসাব!)            │
│  │  (Orders &  │    - Complex joins (order→items→payment)       │
│  │  Payments)  │    - Strong consistency (data loss = disaster) │
│  └─────────────┘                                                │
│                                                                 │
│  ┌─────────────┐    কেন Redis?                                  │
│  │    Redis    │    - Sub-millisecond reads                     │
│  │  (Cart &    │    - TTL for auto-expiry                       │
│  │   Session)  │    - In-memory (2GB carts fit easily)          │
│  └─────────────┘                                                │
│                                                                 │
│  ┌─────────────┐    কেন Elasticsearch?                          │
│  │Elasticsearch│    - Full-text search                          │
│  │  (Search)   │    - Faceted filtering                         │
│  └─────────────┘    - Autocomplete & suggestions                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7. 🔔 Event-Driven Architecture

```javascript
// services/order-events.service.js
// RabbitMQ/Kafka event consumers

const amqp = require('amqplib');

class OrderEventHandler {
    
    async setupConsumers(channel) {
        // Order confirmed → multiple parallel actions
        await channel.consume('order.confirmed', async (msg) => {
            const order = JSON.parse(msg.content.toString());
            
            // এই events parallel এ process হবে
            await Promise.allSettled([
                this.sendOrderConfirmationSMS(order),   // SMS via SSL Wireless
                this.sendOrderConfirmationEmail(order),  // Email via SendGrid
                this.notifySeller(order),                // Seller dashboard update
                this.assignDeliveryPartner(order),       // Pathao/Paperfly API
                this.updateAnalytics(order),             // Analytics event
            ]);
            
            channel.ack(msg);
        });

        // Delivery partner assignment (Bangladesh specific)
        await channel.consume('order.assign_delivery', async (msg) => {
            const order = JSON.parse(msg.content.toString());
            
            const partner = await this.selectDeliveryPartner(order);
            // Dhaka inside: Pathao (same day), Outside: SA Paribahan
            // Heavy items: RedX, Regular: Paperfly
            
            channel.ack(msg);
        });
    }

    async selectDeliveryPartner(order) {
        const { district, weight, delivery_type } = order;
        
        if (district === 'Dhaka' && delivery_type === 'express') {
            return { partner: 'pathao', eta: '4 hours' };
        } else if (weight > 5) {
            return { partner: 'redx', eta: '3-5 days' };
        } else if (district === 'Dhaka') {
            return { partner: 'paperfly', eta: '1-2 days' };
        } else {
            return { partner: 'sa_paribahan', eta: '3-7 days' };
        }
    }
}

module.exports = new OrderEventHandler();
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Monolith vs Microservices

```
┌────────────────────┬──────────────────────┬─────────────────────────┐
│      বিষয়         │    Monolith          │    Microservices         │
├────────────────────┼──────────────────────┼─────────────────────────┤
│ Complexity         │ Low (শুরুতে সহজ)     │ High (DevOps দরকার)      │
│ Deployment         │ একসাথে deploy        │ আলাদা আলাদা deploy       │
│ Scaling            │ পুরোটা scale         │ specific service scale   │
│ Team size          │ ছোট team (< 10)      │ বড় team (10+)           │
│ Failure isolation  │ একটা bug = পুরো down │ একটা service down = OK   │
│ বাংলাদেশ startup  │ ✅ শুরুতে এটাই ভালো   │ ⚠️ পরে migrate করো       │
├────────────────────┼──────────────────────┼─────────────────────────┤
│ সিদ্ধান্ত:         │ Chaldal শুরু করেছে   │ Daraz (Alibaba) scale   │
│                    │ monolith দিয়ে        │ এ পৌঁছালে                │
└────────────────────┴──────────────────────┴─────────────────────────┘

রেকমেন্ডেশন: "Monolith First" → Traffic বাড়লে gradually decompose
```

### 2. SQL vs NoSQL for Products

```
┌─────────────────┬───────────────────────────┬───────────────────────────┐
│    বিষয়        │ PostgreSQL (SQL)           │ MongoDB (NoSQL)           │
├─────────────────┼───────────────────────────┼───────────────────────────┤
│ Schema          │ Fixed (ALTER TABLE slow)   │ Flexible (any field)      │
│ Product variety │ সব product একই column     │ category অনুযায়ী fields   │
│ Joins           │ ✅ Native joins            │ ❌ $lookup (expensive)     │
│ Scaling         │ Vertical (expensive)       │ Horizontal (sharding)     │
│ Transactions    │ ✅ Full ACID               │ ⚠️ Single-doc ACID only   │
│ Search          │ LIKE/tsvector (limited)    │ Text index (basic)        │
├─────────────────┼───────────────────────────┼───────────────────────────┤
│ সিদ্ধান্ত:      │ Orders, Payments এ ব্যবহার │ Products catalog এ ভালো  │
└─────────────────┴───────────────────────────┴───────────────────────────┘
```

### 3. Saga vs 2PC (Two-Phase Commit)

```
┌─────────────────┬────────────────────────┬────────────────────────────┐
│    বিষয়        │ 2PC                    │ Saga Pattern               │
├─────────────────┼────────────────────────┼────────────────────────────┤
│ Consistency     │ Strong (all or nothing)│ Eventual (compensate)      │
│ Availability    │ ❌ Coordinator SPOF    │ ✅ No single coordinator    │
│ Performance     │ Slow (lock holding)    │ Fast (no global lock)      │
│ Complexity      │ Simple concept         │ Complex (compensation logic)│
│ Network issues  │ ❌ Blocking on failure │ ✅ Non-blocking             │
├─────────────────┼────────────────────────┼────────────────────────────┤
│ সিদ্ধান্ত:      │ Single DB তে OK        │ Distributed system এ Saga  │
│                 │                        │ ব্যবহার করো ✅              │
└─────────────────┴────────────────────────┴────────────────────────────┘
```

### 4. Redis Cart vs Database Cart

```
┌──────────────────┬──────────────────────┬──────────────────────────┐
│    বিষয়         │ Redis Cart           │ DB (PostgreSQL) Cart     │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Speed            │ < 1ms                │ 5-20ms                   │
│ Persistence      │ ⚠️ Data loss risk    │ ✅ Durable               │
│ Scale            │ Memory limited       │ Disk (unlimited)         │
│ Cart recovery    │ ❌ Server crash = gone│ ✅ Always available      │
│ Cost             │ Expensive (RAM)      │ Cheap (disk)             │
│ Abandoned cart   │ TTL auto-cleanup     │ Cron job needed          │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ সিদ্ধান্ত:       │ ✅ Active cart (7 days)│ Wishlist / saved items  │
│                  │ Fast UX দরকার        │ Long-term storage        │
└──────────────────┴──────────────────────┴──────────────────────────┘

হাইব্রিড অ্যাপ্রোচ: Redis (primary) + async DB backup (recovery)
```

### 5. Sync vs Event-Driven Order Flow

```
Synchronous:
  Client → API → Inventory → Payment → Confirm → Response
  সমস্যা: Total latency = sum of all steps (slow!)
           একটা service slow = পুরো request slow

Event-Driven:
  Client → API → Create Order → Return "Processing"
                      │
                      ├──▶ Event: Reserve Inventory
                      ├──▶ Event: Process Payment
                      └──▶ Event: Send Notification

  সুবিধা: Fast response, parallel processing, resilient
  অসুবিধা: Complex, eventual consistency, debugging কঠিন
```

---

## 📈 কেস স্টাডি

### 🎯 কেস ১: Daraz 11.11 Sale (10x Traffic Spike)

```
সমস্যা:
━━━━━━━
- Normal দিনে: 100K orders/day
- 11.11 সেলে: 1M+ orders (10x spike)
- সকাল 12:00 AM এ campaign start → traffic spike

সমাধান Strategy:
━━━━━━━━━━━━━━━━

1. Pre-scaling (1 সপ্তাহ আগে):
   - Auto-scaling rules configure
   - Read replicas 3x বাড়ানো
   - CDN cache warm-up
   - Redis cluster node বাড়ানো

2. Traffic Management:
   ┌─────────────────────────────────────┐
   │         Virtual Waiting Room        │
   │  "আপনার আগে 5,420 জন আছেন..."     │
   │  [=====>                    ] 23%   │
   └─────────────────────────────────────┘
   - Queue-based access control
   - Random token → sequential access
   - Bot/script detection & block

3. Flash Sale Architecture:
   - Pre-computed inventory count in Redis
   - Lua script for atomic decrement (no overselling)
   - Order queue → async processing
   - Payment timeout: 10 min (vs normal 30 min)

4. Graceful Degradation:
   - Review system → read-only mode
   - Recommendation → simplified (cached)
   - Search → reduced features (no fuzzy)
   - Non-critical services → circuit breaker OPEN

ফলাফল:
━━━━━━
✅ 99.95% uptime maintained
✅ 1.2M orders processed
✅ Average checkout time: 8 sec (target was 10 sec)
⚠️ Some users experienced 30 sec wait in queue (acceptable)
```

### 🎯 কেস ২: Flash Sale - 1000 Items, 100K Buyers

```
সমস্যা:
━━━━━━━
- ৳1 তে iPhone (promotional)
- 1000 units available
- 100,000 users একসাথে "Buy Now" চাপবে
- কোনো overselling হওয়া যাবে না

সমাধান (Redis Lua Script):
━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- Atomic decrement with check (race condition proof)
local stock = tonumber(redis.call('GET', KEYS[1]))
local requested = tonumber(ARGV[1])

if stock >= requested then
    redis.call('DECRBY', KEYS[1], requested)
    return 1  -- success
else
    return 0  -- out of stock
end

Flow:
━━━━━
  100K requests → Load Balancer → Redis Lua (atomic)
                                      │
                          ┌───────────┼───────────┐
                          │           │           │
                     1000 SUCCESS  99K FAILURE   Queue
                          │                       │
                     Order Queue              "Sorry, sold out!"
                          │                   (< 500ms response)
                     Async Processing
                     (Payment, Inventory DB update)

Key Design Decisions:
━━━━━━━━━━━━━━━━━━━━
1. Stock count Redis এ pre-load (DB hit নেই)
2. Lua script = atomic (lock ছাড়াই thread-safe)
3. Success users → async order queue (payment later)
4. Failed users → immediate "sold out" response
5. Rate limiting: 1 attempt per user per 5 sec
```

### 🎯 কেস ৩: Evaly Failure - Lessons Learned

```
Evaly কী ভুল করেছিল:
━━━━━━━━━━━━━━━━━━━━━

1. ❌ Advance Payment Without Fulfillment Guarantee
   - Customer থেকে টাকা নিয়েছে
   - Product deliver করেনি (months পরেও না)
   - Escrow system ছিল না

2. ❌ No Double-Entry Bookkeeping
   - কত টাকা এসেছে, কত যাওয়া উচিত → হিসাব নেই
   - Seller payment pending কিন্তু customer money spent

3. ❌ Ponzi-like Discount Model
   - 50-70% discount (unsustainable)
   - নতুন customer এর টাকা দিয়ে পুরানো order fulfill
   - Unit economics negative

4. ❌ No Regulatory Compliance
   - Payment gateway license ছাড়া টাকা hold
   - Customer fund misuse

আমাদের সিস্টেমে কিভাবে Prevent করবো:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────┐
│              TRUST ARCHITECTURE                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Escrow System:                              │
│     Customer Pay → Escrow Hold → Delivery       │
│     Confirm → Seller Gets Money                 │
│                                                 │
│  2. Double-Entry Ledger:                        │
│     Every ৳1 movement tracked                   │
│     Daily reconciliation automated              │
│                                                 │
│  3. Seller Verification:                        │
│     NID, Trade License, Bank Account verify     │
│     Performance score based payouts             │
│                                                 │
│  4. Refund Policy (automated):                  │
│     - Not shipped in 48h → auto refund          │
│     - Delivery failed → auto refund             │
│     - Customer complaint → escrow freeze        │
│                                                 │
│  5. Financial Limits:                           │
│     - New seller: max ৳50K pending orders       │
│     - Established: max ৳5L pending              │
│     - Auto-pause if fulfillment rate < 80%      │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 🔧 Advanced Topics

### 1. 🤖 Recommendation Engine

```
Recommendation Strategy:
━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────┐
│         Recommendation Pipeline          │
├─────────────────────────────────────────┤
│                                         │
│  Input Signals:                         │
│  - Browse history                       │
│  - Purchase history                     │
│  - Cart items                           │
│  - Search queries                       │
│  - Similar users' behavior              │
│                                         │
│  Algorithms:                            │
│  1. Collaborative Filtering             │
│     "যারা এটা কিনেছে তারা ওটাও কিনেছে" │
│                                         │
│  2. Content-based Filtering             │
│     "এই product এর সাথে similar"        │
│                                         │
│  3. Trending (Bangladesh specific)      │
│     "ঢাকায় এখন সবচেয়ে বেশি বিক্রি"    │
│                                         │
│  4. Personalized Ranking                │
│     ML model → user preference score    │
│                                         │
└─────────────────────────────────────────┘

Implementation (simplified):
  - Pre-compute recommendations (batch job)
  - Store in Redis (user_id → product_ids)
  - Serve from cache (< 10ms)
  - Update every 6 hours
```

### 2. 💰 Dynamic Pricing

```php
<?php
// app/Services/DynamicPricingService.php

class DynamicPricingService
{
    /**
     * Factors affecting price:
     * - Demand (high demand → higher price)
     * - Inventory level (low stock → higher price)
     * - Time of day (peak hours adjustment)
     * - Competitor pricing
     * - Customer segment (new vs returning)
     */
    public function calculatePrice(string $productId, array $context): int
    {
        $basePrice = $this->getBasePrice($productId);
        $multiplier = 1.0;

        // Demand factor
        $demandScore = $this->getDemandScore($productId);
        if ($demandScore > 0.8) {
            $multiplier += 0.05; // max 5% increase on high demand
        }

        // Inventory scarcity
        $stockRatio = $this->getStockRatio($productId);
        if ($stockRatio < 0.1) { // Less than 10% stock remaining
            $multiplier += 0.03;
        }

        // Bangladesh context: ঈদ/পূজা season pricing
        if ($this->isFestivalSeason()) {
            $multiplier += 0.02;
        }

        // Price ceiling: never exceed MRP (Bangladesh law)
        $finalPrice = min(
            (int)($basePrice * $multiplier),
            $this->getMRP($productId)
        );

        return $finalPrice;
    }
}
```

### 3. 🔄 Return/Refund System

```
Return Flow (Bangladesh Context):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Customer                     System                      Seller
    │                           │                          │
    │──── Return Request ──────▶│                          │
    │     (reason + photo)      │                          │
    │                           │──── Notify Seller ──────▶│
    │                           │                          │
    │                           │◀─── Accept/Reject ───────│
    │                           │     (24h deadline)       │
    │                           │                          │
    │◀── Pickup Scheduled ──────│                          │
    │    (Pathao reverse pickup)│                          │
    │                           │                          │
    │──── Item Returned ───────▶│                          │
    │                           │──── Inspect Item ───────▶│
    │                           │                          │
    │◀── Refund Initiated ──────│                          │
    │    (bKash/Nagad/Card)     │                          │
    │                           │                          │
    │◀── Refund Complete ───────│                          │
    │    (3-7 business days)    │                          │

Refund Methods:
  - bKash/Nagad: Instant refund (same wallet)
  - Card: 7-14 days (bank processing)
  - COD: bKash transfer (customer provides number)
  - Store Credit: Instant (encourage retention)
```

### 4. 🏪 Multi-Vendor Marketplace Architecture

```
┌─────────────────────────────────────────────────────────┐
│              MULTI-VENDOR ARCHITECTURE                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│  │ Seller A│   │ Seller B│   │ Seller C│  (1000s)     │
│  └────┬────┘   └────┬────┘   └────┬────┘              │
│       │              │              │                   │
│       └──────────────┼──────────────┘                   │
│                      │                                  │
│            ┌─────────▼──────────┐                       │
│            │  Seller Dashboard  │                       │
│            │  - Product upload  │                       │
│            │  - Order mgmt     │                       │
│            │  - Analytics      │                       │
│            │  - Payout history │                       │
│            └─────────┬──────────┘                       │
│                      │                                  │
│            ┌─────────▼──────────┐                       │
│            │  Commission Engine │                       │
│            │  - Category based  │                       │
│            │  - 5% - 25% range │                       │
│            │  - Tiered rates   │                       │
│            └─────────┬──────────┘                       │
│                      │                                  │
│            ┌─────────▼──────────┐                       │
│            │  Settlement Engine │                       │
│            │  - Weekly payouts  │                       │
│            │  - Hold period:7d │                       │
│            │  - Auto bank xfer │                       │
│            └────────────────────┘                       │
│                                                         │
│  Commission Model (Daraz Bangladesh style):             │
│  ┌──────────────┬───────────────┐                      │
│  │ Category     │ Commission %  │                      │
│  ├──────────────┼───────────────┤                      │
│  │ Electronics  │ 5-8%          │                      │
│  │ Fashion      │ 15-20%        │                      │
│  │ Groceries    │ 8-12%         │                      │
│  │ Digital      │ 3-5%          │                      │
│  └──────────────┴───────────────┘                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5. 📊 A/B Testing for E-Commerce

```javascript
// services/ab-testing.service.js

class ABTestingService {
    
    // বাংলাদেশ context এ A/B test examples:
    // - "এখনই কিনুন" vs "Buy Now" (Bengali vs English CTA)
    // - COD first vs bKash first in payment options
    // - Free delivery threshold: ৳500 vs ৳999

    async getVariant(userId, experimentId) {
        const experiment = await this.getExperiment(experimentId);
        
        // Deterministic assignment (same user always gets same variant)
        const hash = this.hashUserId(userId, experimentId);
        const bucket = hash % 100;

        if (bucket < experiment.control_percent) {
            return { variant: 'control', config: experiment.control_config };
        } else {
            return { variant: 'treatment', config: experiment.treatment_config };
        }
    }

    // Track conversion
    async trackEvent(userId, experimentId, event, metadata = {}) {
        await this.analyticsQueue.publish({
            user_id: userId,
            experiment_id: experimentId,
            event, // 'view', 'add_to_cart', 'purchase'
            metadata,
            timestamp: Date.now()
        });
    }
}
```

### 6. 🏭 Warehouse Management

```
Warehouse Strategy (Bangladesh):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────┐
  │          Warehouse Locations             │
  │                                          │
  │  🏭 Dhaka Hub (Main)                    │
  │     - Gazipur warehouse (50,000 sqft)   │
  │     - Same-day delivery (Dhaka city)    │
  │                                          │
  │  🏭 Chittagong Hub                      │
  │     - Port proximity (imported goods)   │
  │     - CTG + South Bangladesh coverage   │
  │                                          │
  │  🏭 Rajshahi Hub (Future)               │
  │     - North Bangladesh coverage         │
  │                                          │
  └──────────────────────────────────────────┘

  Inventory Allocation Algorithm:
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  - Predict demand by region (ML model)
  - Pre-position inventory in nearest warehouse
  - If stock-out in local → transfer from other warehouse
  - Fast-moving items → all warehouses
  - Slow-moving items → central warehouse only
```

---

## 🎯 সারসংক্ষেপ

### Key Design Principles

```
┌─────────────────────────────────────────────────────────────┐
│              E-COMMERCE DESIGN PRINCIPLES                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 💰 Payment = Never Lose Money                          │
│     - Idempotency keys (duplicate prevention)              │
│     - Double-entry bookkeeping (audit trail)               │
│     - Escrow for marketplace (Evaly lesson)                │
│                                                             │
│  2. 📦 Inventory = Never Oversell                          │
│     - Distributed locks (Redis/DB)                         │
│     - Reservation with timeout                             │
│     - Atomic operations for flash sales                    │
│                                                             │
│  3. 🚀 Performance = Fast Browse, Safe Checkout            │
│     - Read path: Cache aggressively (Redis + CDN)          │
│     - Write path: Queue + async processing                 │
│     - Search: Elasticsearch (not DB queries!)              │
│                                                             │
│  4. 🔄 Resilience = Graceful Degradation                   │
│     - Circuit breakers on external services                │
│     - Fallback responses (cached data)                     │
│     - Saga pattern for distributed transactions            │
│                                                             │
│  5. 🇧🇩 Bangladesh Context = Mobile-First                  │
│     - Low bandwidth optimization                           │
│     - bKash/Nagad native integration                       │
│     - Bengali language support                             │
│     - Local delivery partner APIs                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### চেকলিস্ট: ইন্টারভিউতে যা Cover করতে হবে

```
✅ Requirements → Functional + Non-functional + Scale numbers
✅ Data model → Which DB for which data (and WHY)
✅ API design → RESTful endpoints, pagination, error handling
✅ Search → Elasticsearch architecture, indexing strategy
✅ Cart → Redis vs DB tradeoff, guest cart merge
✅ Order → Saga pattern, compensation logic
✅ Inventory → Distributed locks, flash sale handling
✅ Payment → Idempotency, double-entry, escrow
✅ Scaling → CDN, caching layers, read replicas, sharding
✅ Monitoring → Metrics, alerts, distributed tracing
✅ BD Context → Payment gateways, delivery partners, regulations
```

### টেকনোলজি স্ট্যাক সামারি

```
┌──────────────────┬─────────────────────────────────────┐
│ Layer            │ Technology                          │
├──────────────────┼─────────────────────────────────────┤
│ Frontend         │ Next.js / React Native (mobile)    │
│ API Gateway      │ Kong / Nginx                       │
│ Product Service  │ Node.js + Express + Elasticsearch  │
│ Cart Service     │ Node.js + Redis                    │
│ Order Service    │ PHP (Laravel) + PostgreSQL          │
│ Payment Service  │ PHP (Laravel) + PostgreSQL          │
│ Inventory        │ PHP (Laravel) + PostgreSQL + Redis  │
│ Message Queue    │ RabbitMQ (simple) / Kafka (scale)  │
│ Cache            │ Redis Cluster                      │
│ CDN              │ CloudFront / BunnyCDN              │
│ Monitoring       │ Prometheus + Grafana + ELK Stack   │
│ CI/CD            │ GitHub Actions + Docker + K8s      │
└──────────────────┴─────────────────────────────────────┘
```

---

> 📝 **নোট**: এই ডিজাইন Amazon/Daraz scale এর জন্য। ছোট startup শুরু করলে Laravel monolith + MySQL + Redis দিয়ে শুরু করো, তারপর traffic অনুযায়ী gradually decompose করো। Premature optimization is the root of all evil!

---

*Last Updated: 2025 | Author: System Design Study Guide*
