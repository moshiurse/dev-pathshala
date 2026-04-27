# 🔗 ইন্টিগ্রেশন টেস্টিং (Integration Testing) — গভীর বিশ্লেষণ

> **"Unit tests tell you that the gears work. Integration tests tell you that the gears mesh."** — অর্থাৎ, Unit test প্রতিটি গিয়ার আলাদাভাবে ঘোরে কিনা তা পরীক্ষা করে, কিন্তু Integration test নিশ্চিত করে যে গিয়ারগুলো একসাথে কাজ করছে।

---

## 📌 সংজ্ঞা ও পরিসর

### Unit vs Integration vs E2E — মৌলিক পার্থক্য

সফটওয়্যার টেস্টিং পিরামিডে তিনটি প্রধান স্তর রয়েছে। প্রতিটির উদ্দেশ্য এবং কাভারেজ সম্পূর্ণ আলাদা:

| বৈশিষ্ট্য | Unit Test | Integration Test | E2E Test |
|---|---|---|---|
| **পরিসর** | একটি ফাংশন/মেথড | একাধিক মডিউলের সংযোগ | সম্পূর্ণ ব্যবহারকারী প্রবাহ |
| **গতি** | অত্যন্ত দ্রুত (ms) | মাঝারি (সেকেন্ড) | ধীর (মিনিট) |
| **নির্ভরতা** | কোনো বাহ্যিক নির্ভরতা নেই | ডাটাবেস, API, ক্যাশ | ব্রাউজার, সম্পূর্ণ ইনফ্রা |
| **ভঙ্গুরতা** | কম | মাঝারি | বেশি |
| **আত্মবিশ্বাস** | নিম্ন-মাঝারি | উচ্চ | সর্বোচ্চ |
| **রক্ষণাবেক্ষণ** | সহজ | মাঝারি | কঠিন |
| **উদাহরণ** | `calculateTax()` | API → DB → Response | Login → Order → Payment |

### ইন্টিগ্রেশন টেস্ট আসলে কী পরীক্ষা করে?

ইন্টিগ্রেশন টেস্ট সিস্টেমের বিভিন্ন কম্পোনেন্টের **মধ্যবর্তী সংযোগস্থল** (interface) পরীক্ষা করে। এটি নিশ্চিত করে যে:

- **HTTP Layer → Controller → Service → Repository → Database** — এই সম্পূর্ণ চেইন সঠিকভাবে কাজ করছে
- ডাটাবেস কুয়েরি প্রকৃত ডাটায় সঠিক ফলাফল দিচ্ছে
- API রিকোয়েস্ট ও রেসপন্স সঠিক ফরম্যাটে আসছে
- বাহ্যিক সার্ভিস কল (পেমেন্ট গেটওয়ে, SMS API) সঠিকভাবে হ্যান্ডেল হচ্ছে
- Authentication ও Authorization middleware সঠিকভাবে কাজ করছে
- Queue, Cache এবং Event System একসাথে সঠিকভাবে interact করছে

**বাংলাদেশ প্রসঙ্গ:** ধরুন আপনি bKash পেমেন্ট ইন্টিগ্রেশন তৈরি করছেন। Unit test শুধু amount validation পরীক্ষা করবে। কিন্তু Integration test পরীক্ষা করবে — API call → bKash sandbox → callback handle → ডাটাবেসে transaction save → user notification পাঠানো — এই সম্পূর্ণ প্রবাহ।

---

## 📊 Integration Test Scope — আর্কিটেকচারাল ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION TEST BOUNDARY                        │
│                                                                     │
│  ┌──────────┐    ┌────────────┐    ┌───────────┐    ┌────────────┐ │
│  │  HTTP     │───▶│ Controller │───▶│  Service  │───▶│ Repository │ │
│  │  Request  │    │ Middleware │    │   Layer   │    │   Layer    │ │
│  └──────────┘    └────────────┘    └─────┬─────┘    └─────┬──────┘ │
│                                          │                │        │
│                    ┌─────────────────────┼────────────────┤        │
│                    │                     │                │        │
│              ┌─────▼─────┐    ┌─────────▼──┐    ┌───────▼──────┐ │
│              │   Queue    │    │   Cache     │    │  Database    │ │
│              │ (Redis/    │    │ (Redis/     │    │ (MySQL/      │ │
│              │  BullMQ)   │    │  Memcached) │    │  PostgreSQL) │ │
│              └────────────┘    └────────────┘    └──────────────┘ │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────────┐                      │
│  │ External APIs     │    │ File Storage     │                      │
│  │ (bKash, SSLComm,  │    │ (S3, Local)      │                      │
│  │  SMS Gateway)     │    │                  │                      │
│  └──────────────────┘    └──────────────────┘                      │
│                                                                     │
│  ◄─── এই সীমানার ভেতরে সব কিছু integration test এ cover হয় ───►  │
└─────────────────────────────────────────────────────────────────────┘

    ┌────────────────────────────────────────────────┐
    │          TEST ISOLATION STRATEGIES              │
    │                                                │
    │  Real DB  ◄──── SQLite in-memory / Docker DB   │
    │  Real API ◄──── Http::fake() / nock            │
    │  Real Queue ◄── Queue::fake()                  │
    │  Real Cache ◄── Cache store: array             │
    │  Real Storage ◄─ Storage::fake()               │
    └────────────────────────────────────────────────┘
```

---

## 💻 ইন্টিগ্রেশন টেস্টের প্রকারভেদ

### ১. API/HTTP Integration Tests

এটি সবচেয়ে সাধারণ ধরনের ইন্টিগ্রেশন টেস্ট। পুরো HTTP lifecycle — request → middleware → controller → service → database → response — সম্পূর্ণ chain পরীক্ষা করা হয়।

#### Laravel Feature Test (PHP/PHPUnit)

```php
<?php

namespace Tests\Feature\Api;

use Tests\TestCase;
use App\Models\User;
use App\Models\Product;
use Illuminate\Foundation\Testing\RefreshDatabase;

class ProductApiTest extends TestCase
{
    use RefreshDatabase;

    private User $admin;
    private User $customer;

    protected function setUp(): void
    {
        parent::setUp();
        $this->admin = User::factory()->admin()->create();
        $this->customer = User::factory()->create();
    }

    /** @test */
    public function authenticated_admin_can_create_product(): void
    {
        $productData = [
            'name'        => 'Premium Jamdani Saree',
            'price'       => 15000.00,
            'category'    => 'clothing',
            'stock'       => 50,
            'description' => 'Handwoven Jamdani from Demra',
        ];

        $response = $this->actingAs($this->admin, 'sanctum')
            ->postJson('/api/v1/products', $productData);

        $response->assertStatus(201)
            ->assertJson([
                'data' => [
                    'name'     => 'Premium Jamdani Saree',
                    'price'    => 15000.00,
                    'category' => 'clothing',
                    'stock'    => 50,
                ],
                'message' => 'Product created successfully',
            ])
            ->assertJsonStructure([
                'data' => ['id', 'name', 'price', 'category', 'stock', 'slug', 'created_at'],
                'message',
            ]);

        // ডাটাবেসে সত্যিই সেভ হয়েছে কিনা যাচাই
        $this->assertDatabaseHas('products', [
            'name'  => 'Premium Jamdani Saree',
            'price' => 15000.00,
        ]);
    }

    /** @test */
    public function regular_customer_cannot_create_product(): void
    {
        $response = $this->actingAs($this->customer, 'sanctum')
            ->postJson('/api/v1/products', [
                'name'  => 'Test Product',
                'price' => 100,
            ]);

        $response->assertStatus(403)
            ->assertJson(['message' => 'Unauthorized action.']);
    }

    /** @test */
    public function product_listing_supports_filtering_and_pagination(): void
    {
        Product::factory()->count(25)->create(['category' => 'electronics']);
        Product::factory()->count(10)->create(['category' => 'clothing']);

        $response = $this->getJson('/api/v1/products?category=electronics&per_page=10&page=2');

        $response->assertStatus(200)
            ->assertJsonCount(10, 'data')
            ->assertJsonPath('meta.total', 25)
            ->assertJsonPath('meta.current_page', 2)
            ->assertJsonPath('meta.per_page', 10)
            ->assertJsonFragment(['category' => 'electronics']);

        // clothing ক্যাটেগরি এই রেসপন্সে থাকবে না
        $response->assertJsonMissing(['category' => 'clothing']);
    }

    /** @test */
    public function product_search_returns_relevant_results(): void
    {
        Product::factory()->create(['name' => 'Organic Hilsa Fish']);
        Product::factory()->create(['name' => 'Frozen Hilsa Pack']);
        Product::factory()->create(['name' => 'Mango Pickle']);

        $response = $this->getJson('/api/v1/products/search?q=hilsa');

        $response->assertStatus(200)
            ->assertJsonCount(2, 'data');
    }

    /** @test */
    public function create_product_validates_required_fields(): void
    {
        $response = $this->actingAs($this->admin, 'sanctum')
            ->postJson('/api/v1/products', []);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['name', 'price', 'category']);
    }

    /** @test */
    public function product_price_cannot_be_negative(): void
    {
        $response = $this->actingAs($this->admin, 'sanctum')
            ->postJson('/api/v1/products', [
                'name'     => 'Test Item',
                'price'    => -500,
                'category' => 'general',
            ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['price']);
    }
}
```

#### Supertest (Node.js/Express + Jest)

```javascript
// tests/integration/product.api.test.js
const request = require('supertest');
const app = require('../../src/app');
const { sequelize, Product, User } = require('../../src/models');
const { generateToken } = require('../../src/utils/auth');

describe('Product API Integration Tests', () => {
    let adminToken;
    let customerToken;
    let adminUser;

    beforeAll(async () => {
        await sequelize.sync({ force: true });
    });

    beforeEach(async () => {
        await Product.destroy({ where: {} });
        await User.destroy({ where: {} });

        adminUser = await User.create({
            name: 'Admin',
            email: 'admin@example.com',
            password: 'hashed_password',
            role: 'admin',
        });

        const customer = await User.create({
            name: 'Rahim',
            email: 'rahim@example.com',
            password: 'hashed_password',
            role: 'customer',
        });

        adminToken = generateToken(adminUser);
        customerToken = generateToken(customer);
    });

    afterAll(async () => {
        await sequelize.close();
    });

    describe('POST /api/v1/products', () => {
        const validProduct = {
            name: 'Nakshi Kantha',
            price: 5000,
            category: 'handicraft',
            stock: 20,
            description: 'Traditional embroidered quilt from Rajshahi',
        };

        it('should create product when admin is authenticated', async () => {
            const res = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${adminToken}`)
                .send(validProduct)
                .expect(201);

            expect(res.body.data).toMatchObject({
                name: 'Nakshi Kantha',
                price: 5000,
                category: 'handicraft',
            });
            expect(res.body.data.id).toBeDefined();

            // ডাটাবেসে আছে কিনা যাচাই
            const dbProduct = await Product.findByPk(res.body.data.id);
            expect(dbProduct).not.toBeNull();
            expect(dbProduct.name).toBe('Nakshi Kantha');
        });

        it('should reject creation by non-admin users', async () => {
            await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${customerToken}`)
                .send(validProduct)
                .expect(403);
        });

        it('should validate required fields', async () => {
            const res = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${adminToken}`)
                .send({})
                .expect(422);

            expect(res.body.errors).toEqual(
                expect.arrayContaining([
                    expect.objectContaining({ field: 'name' }),
                    expect.objectContaining({ field: 'price' }),
                ])
            );
        });

        it('should handle concurrent product creation gracefully', async () => {
            const products = Array.from({ length: 10 }, (_, i) => ({
                ...validProduct,
                name: `Product ${i}`,
            }));

            const results = await Promise.all(
                products.map(p =>
                    request(app)
                        .post('/api/v1/products')
                        .set('Authorization', `Bearer ${adminToken}`)
                        .send(p)
                )
            );

            const successCount = results.filter(r => r.status === 201).length;
            expect(successCount).toBe(10);

            const totalProducts = await Product.count();
            expect(totalProducts).toBe(10);
        });
    });

    describe('GET /api/v1/products', () => {
        beforeEach(async () => {
            await Product.bulkCreate([
                { name: 'Muslin Fabric', price: 8000, category: 'textile', stock: 15 },
                { name: 'Clay Pottery', price: 500, category: 'handicraft', stock: 100 },
                { name: 'Brass Lamp', price: 2500, category: 'handicraft', stock: 30 },
            ]);
        });

        it('should return paginated results with filtering', async () => {
            const res = await request(app)
                .get('/api/v1/products?category=handicraft&per_page=10')
                .expect(200);

            expect(res.body.data).toHaveLength(2);
            expect(res.body.meta.total).toBe(2);
            res.body.data.forEach(product => {
                expect(product.category).toBe('handicraft');
            });
        });

        it('should sort by price ascending', async () => {
            const res = await request(app)
                .get('/api/v1/products?sort=price&order=asc')
                .expect(200);

            const prices = res.body.data.map(p => p.price);
            expect(prices).toEqual([...prices].sort((a, b) => a - b));
        });
    });
});
```

---

### ২. Database Integration Tests

ডাটাবেস ইন্টিগ্রেশন টেস্ট প্রকৃত ডাটাবেসের সাথে যোগাযোগ পরীক্ষা করে। এখানে ORM queries, migrations, relationships, constraints — সব কিছু real database এ চালানো হয়।

#### Laravel Database Tests (PHP)

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\Order;
use App\Models\User;
use App\Models\Product;
use App\Services\OrderService;
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderDatabaseTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->seed(\Database\Seeders\CategorySeeder::class);
    }

    /** @test */
    public function order_creation_persists_with_all_relations(): void
    {
        $user = User::factory()->create([
            'phone' => '+8801712345678',
            'division' => 'Dhaka',
        ]);
        $products = Product::factory()->count(3)->create();

        $orderService = app(OrderService::class);

        $order = $orderService->createOrder($user, [
            'items' => $products->map(fn ($p) => [
                'product_id' => $p->id,
                'quantity'   => 2,
            ])->toArray(),
            'shipping_address' => 'House 42, Road 11, Dhanmondi, Dhaka',
            'payment_method'   => 'bkash',
        ]);

        // Order টেবিলে সেভ হয়েছে
        $this->assertDatabaseHas('orders', [
            'id'               => $order->id,
            'user_id'          => $user->id,
            'status'           => 'pending',
            'payment_method'   => 'bkash',
            'shipping_address' => 'House 42, Road 11, Dhanmondi, Dhaka',
        ]);

        // Order items সঠিকভাবে তৈরি হয়েছে
        $this->assertDatabaseCount('order_items', 3);

        foreach ($products as $product) {
            $this->assertDatabaseHas('order_items', [
                'order_id'   => $order->id,
                'product_id' => $product->id,
                'quantity'   => 2,
            ]);
        }

        // Relationships load হচ্ছে কিনা
        $freshOrder = Order::with(['items.product', 'user'])->find($order->id);
        $this->assertCount(3, $freshOrder->items);
        $this->assertEquals($user->id, $freshOrder->user->id);
    }

    /** @test */
    public function order_total_calculated_correctly_with_tax(): void
    {
        $user = User::factory()->create();
        $product = Product::factory()->create(['price' => 1000.00]);

        $order = Order::factory()->create(['user_id' => $user->id]);
        $order->items()->create([
            'product_id' => $product->id,
            'quantity'   => 3,
            'unit_price' => 1000.00,
        ]);

        // বাংলাদেশে ১৫% VAT
        $expectedTotal = 3000.00 * 1.15; // 3450.00

        $this->assertEquals($expectedTotal, $order->calculateTotal());
    }

    /** @test */
    public function stock_reduces_after_order_placement(): void
    {
        $product = Product::factory()->create(['stock' => 50]);
        $user = User::factory()->create();

        $orderService = app(OrderService::class);
        $orderService->createOrder($user, [
            'items' => [['product_id' => $product->id, 'quantity' => 5]],
            'shipping_address' => 'Chittagong',
            'payment_method'   => 'cod',
        ]);

        $product->refresh();
        $this->assertEquals(45, $product->stock);
    }

    /** @test */
    public function order_fails_when_stock_insufficient(): void
    {
        $product = Product::factory()->create(['stock' => 2]);
        $user = User::factory()->create();

        $this->expectException(\App\Exceptions\InsufficientStockException::class);

        $orderService = app(OrderService::class);
        $orderService->createOrder($user, [
            'items' => [['product_id' => $product->id, 'quantity' => 10]],
            'shipping_address' => 'Sylhet',
            'payment_method'   => 'nagad',
        ]);

        // Stock অপরিবর্তিত থাকবে (transaction rollback)
        $product->refresh();
        $this->assertEquals(2, $product->stock);
    }

    /** @test */
    public function complex_query_scopes_work_correctly(): void
    {
        $user = User::factory()->create();

        Order::factory()->count(5)->create([
            'user_id' => $user->id,
            'status'  => 'completed',
            'created_at' => now()->subDays(10),
        ]);

        Order::factory()->count(3)->create([
            'user_id' => $user->id,
            'status'  => 'pending',
        ]);

        Order::factory()->count(2)->create([
            'user_id' => $user->id,
            'status'  => 'completed',
            'created_at' => now()->subMonths(2),
        ]);

        // এই মাসের completed orders
        $thisMonthCompleted = Order::forUser($user->id)
            ->completed()
            ->thisMonth()
            ->count();

        $this->assertEquals(5, $thisMonthCompleted);

        // মোট pending orders
        $pendingOrders = Order::forUser($user->id)->pending()->count();
        $this->assertEquals(3, $pendingOrders);
    }
}
```

#### Sequelize Database Test (Node.js/Jest)

```javascript
// tests/integration/order.database.test.js
const { sequelize, Order, OrderItem, Product, User } = require('../../src/models');
const OrderService = require('../../src/services/OrderService');

describe('Order Database Integration', () => {
    beforeAll(async () => {
        await sequelize.sync({ force: true });
    });

    beforeEach(async () => {
        // প্রতিটি টেস্টের আগে টেবিল খালি করা
        await OrderItem.destroy({ where: {}, force: true });
        await Order.destroy({ where: {}, force: true });
        await Product.destroy({ where: {}, force: true });
        await User.destroy({ where: {}, force: true });
    });

    afterAll(async () => {
        await sequelize.close();
    });

    it('should persist order with all relations in a transaction', async () => {
        const user = await User.create({
            name: 'Karim',
            email: 'karim@test.com',
            phone: '+8801812345678',
        });

        const products = await Product.bulkCreate([
            { name: 'Panjabi', price: 2500, stock: 20 },
            { name: 'Lungi', price: 800, stock: 50 },
        ]);

        const orderService = new OrderService();
        const order = await orderService.createOrder(user.id, {
            items: products.map(p => ({ productId: p.id, quantity: 2 })),
            shippingAddress: 'Mirpur-10, Dhaka',
            paymentMethod: 'bkash',
        });

        // DB তে Order আছে কিনা
        const dbOrder = await Order.findByPk(order.id, {
            include: [{ model: OrderItem, include: [Product] }, User],
        });

        expect(dbOrder).not.toBeNull();
        expect(dbOrder.User.name).toBe('Karim');
        expect(dbOrder.OrderItems).toHaveLength(2);
        expect(dbOrder.status).toBe('pending');
        expect(dbOrder.paymentMethod).toBe('bkash');

        // Stock কমেছে কিনা
        const updatedPanjabi = await Product.findByPk(products[0].id);
        expect(updatedPanjabi.stock).toBe(18);
    });

    it('should rollback entire transaction on insufficient stock', async () => {
        const user = await User.create({
            name: 'Halim',
            email: 'halim@test.com',
        });

        const product = await Product.create({
            name: 'Limited Edition Saree',
            price: 25000,
            stock: 1,
        });

        const orderService = new OrderService();

        await expect(
            orderService.createOrder(user.id, {
                items: [{ productId: product.id, quantity: 5 }],
                shippingAddress: 'Banani, Dhaka',
                paymentMethod: 'card',
            })
        ).rejects.toThrow('Insufficient stock');

        // Transaction rollback হওয়ায় কোনো order তৈরি হয়নি
        const orderCount = await Order.count();
        expect(orderCount).toBe(0);

        // Stock অপরিবর্তিত
        const unchangedProduct = await Product.findByPk(product.id);
        expect(unchangedProduct.stock).toBe(1);
    });

    it('should handle complex aggregation queries', async () => {
        const user = await User.create({ name: 'Rina', email: 'rina@test.com' });

        // বিভিন্ন স্ট্যাটাসে অর্ডার তৈরি
        const orders = await Order.bulkCreate([
            { userId: user.id, status: 'completed', totalAmount: 5000 },
            { userId: user.id, status: 'completed', totalAmount: 3000 },
            { userId: user.id, status: 'cancelled', totalAmount: 2000 },
            { userId: user.id, status: 'pending', totalAmount: 1500 },
        ]);

        const stats = await Order.findAll({
            where: { userId: user.id },
            attributes: [
                'status',
                [sequelize.fn('COUNT', sequelize.col('id')), 'count'],
                [sequelize.fn('SUM', sequelize.col('totalAmount')), 'total'],
            ],
            group: ['status'],
            raw: true,
        });

        const completed = stats.find(s => s.status === 'completed');
        expect(Number(completed.count)).toBe(2);
        expect(Number(completed.total)).toBe(8000);
    });
});
```

---

### ৩. External Service Integration — বাহ্যিক সার্ভিস টেস্টিং

বাস্তব প্রজেক্টে আমরা অনেক বাহ্যিক সার্ভিসের সাথে কাজ করি — bKash/Nagad পেমেন্ট, SSL Commerz, SMS গেটওয়ে (BulkSMSBD, Infobip), ইমেইল সার্ভিস ইত্যাদি। এগুলোর ইন্টিগ্রেশন টেস্টে আমরা **HTTP mocking** ব্যবহার করি — কারণ প্রতিটি টেস্টে সত্যিকার API call করা অবাস্তব এবং ব্যয়বহুল।

#### Laravel Http::fake() — bKash Payment Integration

```php
<?php

namespace Tests\Feature\Payment;

use Tests\TestCase;
use App\Models\User;
use App\Models\Order;
use App\Services\Payment\BkashPaymentService;
use Illuminate\Support\Facades\Http;
use Illuminate\Foundation\Testing\RefreshDatabase;

class BkashPaymentTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function bkash_payment_initiation_works_correctly(): void
    {
        Http::fake([
            // bKash Token API
            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/token/grant' => Http::response([
                'id_token'    => 'fake_token_12345',
                'token_type'  => 'Bearer',
                'expires_in'  => 3600,
            ], 200),

            // bKash Create Payment API
            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/create' => Http::response([
                'paymentID'        => 'TR001234567890',
                'bkashURL'         => 'https://sandbox.payment.bka.sh/redirect/tokenized/?paymentID=TR001234567890',
                'callbackURL'      => 'https://myapp.com/api/bkash/callback',
                'successCallbackURL'=> 'https://myapp.com/payment/success',
                'statusCode'       => '0000',
                'statusMessage'    => 'Successful',
            ], 200),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->create([
            'user_id' => $user->id,
            'total'   => 2500.00,
        ]);

        $bkashService = app(BkashPaymentService::class);
        $result = $bkashService->initiatePayment($order);

        $this->assertEquals('TR001234567890', $result['paymentID']);
        $this->assertStringContains('sandbox.payment.bka.sh', $result['bkashURL']);

        // সঠিক API কল হয়েছে কিনা যাচাই
        Http::assertSent(function ($request) use ($order) {
            return $request->url() === 'https://tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/create'
                && $request['amount'] === '2500.00'
                && $request['currency'] === 'BDT'
                && $request['intent'] === 'sale';
        });

        // Order স্ট্যাটাস আপডেট হয়েছে
        $order->refresh();
        $this->assertEquals('payment_initiated', $order->payment_status);
        $this->assertEquals('TR001234567890', $order->payment_id);
    }

    /** @test */
    public function bkash_payment_callback_processes_successfully(): void
    {
        Http::fake([
            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/token/grant' => Http::response([
                'id_token' => 'fake_token',
            ], 200),

            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/execute' => Http::response([
                'paymentID'      => 'TR001234567890',
                'trxID'          => 'TRX9876543210',
                'transactionStatus' => 'Completed',
                'amount'         => '2500.00',
                'currency'       => 'BDT',
                'statusCode'     => '0000',
                'statusMessage'  => 'Successful',
            ], 200),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->create([
            'user_id'        => $user->id,
            'total'          => 2500.00,
            'payment_status' => 'payment_initiated',
            'payment_id'     => 'TR001234567890',
        ]);

        $response = $this->postJson('/api/bkash/callback', [
            'paymentID' => 'TR001234567890',
            'status'    => 'success',
        ]);

        $response->assertStatus(200);

        $order->refresh();
        $this->assertEquals('paid', $order->payment_status);
        $this->assertEquals('TRX9876543210', $order->transaction_id);
    }

    /** @test */
    public function handles_bkash_api_failure_gracefully(): void
    {
        Http::fake([
            'tokenized.sandbox.bka.sh/*' => Http::response([
                'statusCode'    => '2001',
                'statusMessage' => 'Invalid App Key',
            ], 401),
        ]);

        $user = User::factory()->create();
        $order = Order::factory()->create(['user_id' => $user->id, 'total' => 1000]);

        $bkashService = app(BkashPaymentService::class);

        $this->expectException(\App\Exceptions\PaymentGatewayException::class);
        $bkashService->initiatePayment($order);

        // Order স্ট্যাটাস payment_failed হবে
        $order->refresh();
        $this->assertEquals('payment_failed', $order->payment_status);
    }

    /** @test */
    public function handles_bkash_timeout_with_retry(): void
    {
        Http::fake([
            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/token/grant' => Http::response([
                'id_token' => 'fake_token',
            ], 200),

            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/create' => Http::sequence()
                ->push(null, 500)           // প্রথম চেষ্টা: সার্ভার error
                ->push(null, 500)           // দ্বিতীয় চেষ্টা: সার্ভার error
                ->push([                    // তৃতীয় চেষ্টা: সফল
                    'paymentID'  => 'TR0099',
                    'statusCode' => '0000',
                    'bkashURL'   => 'https://sandbox.payment.bka.sh/redirect/tokenized/?paymentID=TR0099',
                ], 200),
        ]);

        $order = Order::factory()->create(['total' => 500]);
        $bkashService = app(BkashPaymentService::class);
        $result = $bkashService->initiatePayment($order);

        $this->assertEquals('TR0099', $result['paymentID']);

        // মোট ৩ বার API কল হয়েছে (২ বার retry + ১ বার সফল)
        Http::assertSentCount(4); // 1 token + 3 create attempts
    }
}
```

#### Nock (Node.js) — SMS Gateway Integration

```javascript
// tests/integration/sms-gateway.test.js
const nock = require('nock');
const SmsService = require('../../src/services/SmsService');
const { User, SmsLog } = require('../../src/models');

describe('SMS Gateway Integration', () => {
    afterEach(() => {
        nock.cleanAll();
    });

    it('should send OTP via BulkSMSBD gateway', async () => {
        const smsApi = nock('https://bulksmsbd.net')
            .post('/api/smsapi', body => {
                // রিকোয়েস্ট বডি সঠিক কিনা যাচাই
                expect(body.number).toBe('01712345678');
                expect(body.message).toContain('Your OTP is');
                expect(body.message).toMatch(/\d{6}/); // ৬ ডিজিট OTP আছে
                return true;
            })
            .reply(200, {
                response_code: 202,
                success_message: 'SMS sent successfully',
                message_id: 'MSG12345',
            });

        const smsService = new SmsService();
        const result = await smsService.sendOtp('+8801712345678');

        expect(result.success).toBe(true);
        expect(result.messageId).toBe('MSG12345');
        expect(smsApi.isDone()).toBe(true);

        // SMS Log ডাটাবেসে সেভ হয়েছে
        const log = await SmsLog.findOne({
            where: { phone: '+8801712345678', type: 'otp' },
        });
        expect(log).not.toBeNull();
        expect(log.status).toBe('sent');
    });

    it('should handle SMS gateway downtime', async () => {
        nock('https://bulksmsbd.net')
            .post('/api/smsapi')
            .replyWithError('ECONNREFUSED');

        const smsService = new SmsService();
        const result = await smsService.sendOtp('+8801712345678');

        expect(result.success).toBe(false);
        expect(result.error).toBe('gateway_unavailable');

        // ব্যর্থ SMS log-ও সেভ হয়েছে
        const log = await SmsLog.findOne({
            where: { phone: '+8801712345678' },
        });
        expect(log.status).toBe('failed');
        expect(log.failureReason).toBe('gateway_unavailable');
    });

    it('should respect rate limiting per phone number', async () => {
        // ৫ মিনিটে সর্বোচ্চ ৩ বার OTP পাঠানো যাবে
        const smsApi = nock('https://bulksmsbd.net')
            .post('/api/smsapi')
            .times(3)
            .reply(200, { response_code: 202, success_message: 'Sent' });

        const smsService = new SmsService();

        await smsService.sendOtp('+8801712345678');
        await smsService.sendOtp('+8801712345678');
        await smsService.sendOtp('+8801712345678');

        // ৪র্থ বার চেষ্টা করলে rate limit error আসবে
        await expect(
            smsService.sendOtp('+8801712345678')
        ).rejects.toThrow('OTP rate limit exceeded. Try again in 5 minutes.');
    });
});
```

---

### ৪. Queue Integration Tests

Queue system টেস্টিং নিশ্চিত করে যে ব্যাকগ্রাউন্ড জব সঠিকভাবে dispatch, process এবং retry হচ্ছে।

#### Laravel Queue::fake()

```php
<?php

namespace Tests\Feature\Queue;

use Tests\TestCase;
use App\Models\Order;
use App\Models\User;
use App\Jobs\ProcessPayment;
use App\Jobs\SendOrderConfirmation;
use App\Jobs\UpdateInventory;
use App\Jobs\NotifyShippingPartner;
use App\Notifications\OrderPlacedNotification;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Notification;
use Illuminate\Foundation\Testing\RefreshDatabase;

class OrderQueueTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function order_placement_dispatches_all_required_jobs(): void
    {
        Queue::fake();
        Notification::fake();

        $user = User::factory()->create();

        $response = $this->actingAs($user, 'sanctum')
            ->postJson('/api/v1/orders', [
                'items' => [
                    ['product_id' => 1, 'quantity' => 2],
                    ['product_id' => 2, 'quantity' => 1],
                ],
                'shipping_address' => 'Uttara, Dhaka',
                'payment_method'   => 'bkash',
            ]);

        $response->assertStatus(201);

        // সঠিক জব dispatch হয়েছে কিনা
        Queue::assertPushed(ProcessPayment::class, function ($job) {
            return $job->paymentMethod === 'bkash';
        });

        Queue::assertPushed(SendOrderConfirmation::class);
        Queue::assertPushed(UpdateInventory::class);

        // শিপিং পার্টনারকে এখনো notify করা হয়নি (পেমেন্ট complete হওয়ার পরে হবে)
        Queue::assertNotPushed(NotifyShippingPartner::class);

        // জব চেইন সঠিকভাবে সেটআপ হয়েছে
        Queue::assertPushedOn('payments', ProcessPayment::class);
        Queue::assertPushedOn('notifications', SendOrderConfirmation::class);
        Queue::assertPushedOn('inventory', UpdateInventory::class);
    }

    /** @test */
    public function failed_payment_job_retries_and_notifies_admin(): void
    {
        $order = Order::factory()->create([
            'payment_method' => 'bkash',
            'total'          => 5000,
        ]);

        // আসল জব চালানো (fake ছাড়া)
        $job = new ProcessPayment($order);

        // Simulate bKash API failure
        Http::fake([
            'tokenized.sandbox.bka.sh/*' => Http::response([], 500),
        ]);

        try {
            $job->handle();
        } catch (\Exception $e) {
            // ৩ বার retry করার পরে failed হলে
            $job->failed($e);
        }

        $order->refresh();
        $this->assertEquals('payment_failed', $order->payment_status);

        // Admin notification পাঠানো হয়েছে
        $this->assertDatabaseHas('notifications', [
            'type' => 'App\Notifications\PaymentFailedNotification',
        ]);
    }
}
```

#### BullMQ Queue Test (Node.js)

```javascript
// tests/integration/queue.test.js
const { Queue, Worker, QueueEvents } = require('bullmq');
const IORedis = require('ioredis');
const { processOrderJob } = require('../../src/jobs/processOrder');
const { Order, User } = require('../../src/models');

const redisConnection = new IORedis({
    host: process.env.TEST_REDIS_HOST || 'localhost',
    port: 6379,
    maxRetriesPerRequest: null,
});

describe('Order Queue Integration (BullMQ)', () => {
    let orderQueue;
    let queueEvents;

    beforeAll(async () => {
        orderQueue = new Queue('order-processing-test', {
            connection: redisConnection,
        });
        queueEvents = new QueueEvents('order-processing-test', {
            connection: redisConnection,
        });
    });

    afterEach(async () => {
        await orderQueue.drain();
    });

    afterAll(async () => {
        await orderQueue.close();
        await queueEvents.close();
        await redisConnection.quit();
    });

    it('should process order job and update status', async () => {
        const user = await User.create({ name: 'Test', email: 'q@test.com' });
        const order = await Order.create({
            userId: user.id,
            status: 'pending',
            totalAmount: 3000,
        });

        // জব যোগ করা
        const job = await orderQueue.add('process-order', {
            orderId: order.id,
            action: 'confirm',
        });

        // Worker দিয়ে process করা
        const worker = new Worker(
            'order-processing-test',
            processOrderJob,
            { connection: redisConnection }
        );

        // জব সম্পন্ন হওয়া পর্যন্ত অপেক্ষা
        await new Promise((resolve, reject) => {
            worker.on('completed', async (completedJob) => {
                if (completedJob.id === job.id) {
                    resolve();
                }
            });
            worker.on('failed', (failedJob, err) => {
                if (failedJob.id === job.id) reject(err);
            });
        });

        await worker.close();

        const updatedOrder = await Order.findByPk(order.id);
        expect(updatedOrder.status).toBe('confirmed');
    });

    it('should handle job retries on transient failures', async () => {
        let attemptCount = 0;

        const worker = new Worker(
            'order-processing-test',
            async (job) => {
                attemptCount++;
                if (attemptCount < 3) {
                    throw new Error('Transient failure');
                }
                return { success: true };
            },
            {
                connection: redisConnection,
                settings: { backoffStrategy: () => 100 },
            }
        );

        const job = await orderQueue.add(
            'retry-test',
            { orderId: 999 },
            { attempts: 3, backoff: { type: 'fixed', delay: 100 } }
        );

        await new Promise(resolve => {
            worker.on('completed', (completedJob) => {
                if (completedJob.id === job.id) resolve();
            });
        });

        await worker.close();
        expect(attemptCount).toBe(3);
    });
});
```

---

### ৫. Cache Integration Tests

ক্যাশ ইন্টিগ্রেশন টেস্ট নিশ্চিত করে যে ক্যাশিং লজিক সঠিকভাবে কাজ করছে — cache hit, miss, invalidation এবং TTL।

```php
<?php

namespace Tests\Feature\Cache;

use Tests\TestCase;
use App\Models\Product;
use App\Services\ProductService;
use Illuminate\Support\Facades\Cache;
use Illuminate\Foundation\Testing\RefreshDatabase;

class ProductCacheTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function product_listing_is_cached_after_first_request(): void
    {
        Product::factory()->count(10)->create();
        Cache::flush();

        // প্রথম রিকোয়েস্ট — ক্যাশ miss, DB থেকে আনবে
        $this->getJson('/api/v1/products')->assertStatus(200);
        $this->assertTrue(Cache::has('products:page:1'));

        // দ্বিতীয় রিকোয়েস্ট — ক্যাশ থেকে আসবে
        $cachedResponse = Cache::get('products:page:1');
        $this->assertCount(10, $cachedResponse['data']);
    }

    /** @test */
    public function cache_invalidates_when_product_updated(): void
    {
        $product = Product::factory()->create(['name' => 'Old Name']);
        Cache::put("product:{$product->id}", $product, 3600);
        Cache::put('products:page:1', ['data' => [$product]], 3600);

        // প্রোডাক্ট আপডেট
        $this->actingAs(User::factory()->admin()->create(), 'sanctum')
            ->putJson("/api/v1/products/{$product->id}", ['name' => 'New Name']);

        // ক্যাশ invalidate হয়েছে
        $this->assertFalse(Cache::has("product:{$product->id}"));
        $this->assertFalse(Cache::has('products:page:1'));

        // নতুন রিকোয়েস্টে updated data আসবে
        $response = $this->getJson("/api/v1/products/{$product->id}");
        $response->assertJsonPath('data.name', 'New Name');
    }

    /** @test */
    public function cache_ttl_respected_correctly(): void
    {
        $service = app(ProductService::class);

        Product::factory()->create(['name' => 'Cached Item']);
        $service->getPopularProducts(); // ক্যাশে রাখা

        $this->assertTrue(Cache::has('popular_products'));

        // সময় এগিয়ে নিয়ে যাওয়া (TTL expire)
        $this->travel(2)->hours();

        $this->assertFalse(Cache::has('popular_products'));
    }
}
```

---

### ৬. File Storage Integration Tests

ফাইল আপলোড, প্রসেসিং এবং স্টোরেজ সংক্রান্ত ইন্টিগ্রেশন টেস্ট।

```php
<?php

namespace Tests\Feature\Storage;

use Tests\TestCase;
use App\Models\User;
use App\Models\Product;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Foundation\Testing\RefreshDatabase;

class FileUploadTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function product_image_upload_stores_and_resizes(): void
    {
        Storage::fake('public');

        $admin = User::factory()->admin()->create();
        $product = Product::factory()->create();

        $response = $this->actingAs($admin, 'sanctum')
            ->postJson("/api/v1/products/{$product->id}/images", [
                'image' => UploadedFile::fake()->image('jamdani.jpg', 1200, 800),
            ]);

        $response->assertStatus(201)
            ->assertJsonStructure(['data' => ['original_url', 'thumbnail_url']]);

        // Original ফাইল সেভ হয়েছে
        $imagePath = $response->json('data.original_path');
        Storage::disk('public')->assertExists($imagePath);

        // Thumbnail তৈরি হয়েছে
        $thumbnailPath = $response->json('data.thumbnail_path');
        Storage::disk('public')->assertExists($thumbnailPath);
    }

    /** @test */
    public function rejects_invalid_file_types(): void
    {
        Storage::fake('public');
        $admin = User::factory()->admin()->create();
        $product = Product::factory()->create();

        $response = $this->actingAs($admin, 'sanctum')
            ->postJson("/api/v1/products/{$product->id}/images", [
                'image' => UploadedFile::fake()->create('malware.exe', 500),
            ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['image']);
    }

    /** @test */
    public function csv_import_processes_products_from_file(): void
    {
        Storage::fake('local');

        $csvContent = "name,price,category,stock\n"
            . "Hilsa Fish,800,food,100\n"
            . "Mango,150,food,500\n"
            . "Panjabi,2500,clothing,50";

        $file = UploadedFile::fake()->createWithContent('products.csv', $csvContent);

        $admin = User::factory()->admin()->create();

        $response = $this->actingAs($admin, 'sanctum')
            ->postJson('/api/v1/products/import', ['file' => $file]);

        $response->assertStatus(200)
            ->assertJson(['imported' => 3]);

        $this->assertDatabaseCount('products', 3);
        $this->assertDatabaseHas('products', [
            'name'  => 'Hilsa Fish',
            'price' => 800,
        ]);
    }

    /** @test */
    public function old_images_deleted_when_product_image_replaced(): void
    {
        Storage::fake('public');

        $admin = User::factory()->admin()->create();
        $product = Product::factory()->create();

        // প্রথম ইমেজ আপলোড
        $firstResponse = $this->actingAs($admin, 'sanctum')
            ->postJson("/api/v1/products/{$product->id}/images", [
                'image' => UploadedFile::fake()->image('first.jpg'),
            ]);

        $firstImagePath = $firstResponse->json('data.original_path');

        // নতুন ইমেজ আপলোড (replace)
        $this->actingAs($admin, 'sanctum')
            ->postJson("/api/v1/products/{$product->id}/images", [
                'image'   => UploadedFile::fake()->image('second.jpg'),
                'replace' => true,
            ]);

        // পুরোনো ইমেজ মুছে গেছে
        Storage::disk('public')->assertMissing($firstImagePath);
    }
}
```

---

## 🔥 Advanced Integration Testing কৌশল

### ১. Test Database Strategies — ডাটাবেস কৌশল

ইন্টিগ্রেশন টেস্টে ডাটাবেস ব্যবস্থাপনা সবচেয়ে গুরুত্বপূর্ণ সিদ্ধান্ত। তিনটি প্রধান কৌশল:

```
┌────────────────────────────────────────────────────────────────┐
│              DATABASE STRATEGY COMPARISON                       │
├──────────────┬──────────────┬──────────────┬──────────────────┤
│              │ SQLite       │ Docker DB    │ Transactions     │
│              │ In-Memory    │              │                  │
├──────────────┼──────────────┼──────────────┼──────────────────┤
│ গতি         │ ⚡ সবচেয়ে দ্রুত │ 🐢 ধীর       │ ⚡ দ্রুত         │
│ বিশ্বস্ততা  │ ⚠️  কম        │ ✅ সর্বোচ্চ   │ ✅ উচ্চ          │
│ সেটআপ       │ সহজ          │ জটিল          │ সহজ              │
│ Isolation    │ ✅ প্রতিটি টেস্ট │ ✅ প্রতিটি রান │ ✅ প্রতিটি টেস্ট │
│ সীমাবদ্ধতা  │ DB-specific  │ রিসোর্স      │ Nested TX        │
│              │ features নেই │ প্রয়োজন     │ সমস্যা          │
└──────────────┴──────────────┴──────────────┴──────────────────┘
```

#### SQLite In-Memory (দ্রুত কিন্তু সীমিত)

```php
// phpunit.xml
<php>
    <env name="DB_CONNECTION" value="sqlite"/>
    <env name="DB_DATABASE" value=":memory:"/>
</php>
```

#### Docker Test Database (বিশ্বস্ত)

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  test-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: app_test
      MYSQL_ROOT_PASSWORD: test_secret
    ports:
      - "33061:3306"
    tmpfs:
      - /var/lib/mysql  # RAM-এ চালানো দ্রুততার জন্য
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 10
```

```javascript
// jest.setup.js — Docker-based test DB config
const { Sequelize } = require('sequelize');

const testSequelize = new Sequelize({
    dialect: 'mysql',
    host: 'localhost',
    port: 33061,
    username: 'root',
    password: 'test_secret',
    database: 'app_test',
    logging: false,
});

beforeAll(async () => {
    await testSequelize.authenticate();
    await testSequelize.sync({ force: true });
});

afterAll(async () => {
    await testSequelize.close();
});

module.exports = testSequelize;
```

#### Transaction Wrapping (দ্রুত rollback)

```php
<?php
// Laravel DatabaseTransactions trait ব্যবহার
// প্রতিটি টেস্ট একটি transaction এ wrap হয় এবং শেষে rollback হয়

namespace Tests\Feature;

use Tests\TestCase;
use Illuminate\Foundation\Testing\DatabaseTransactions;

class FastDatabaseTest extends TestCase
{
    // RefreshDatabase-র বদলে DatabaseTransactions
    // RefreshDatabase: প্রতিটি টেস্টে migrate:fresh চালায় (ধীর)
    // DatabaseTransactions: শুধু rollback করে (দ্রুত)
    use DatabaseTransactions;

    /** @test */
    public function any_db_changes_auto_rollback_after_test(): void
    {
        // এই টেস্টে যা-ই create/update হোক, পরের টেস্টে থাকবে না
        $user = User::factory()->create();
        $this->assertDatabaseHas('users', ['id' => $user->id]);
        // টেস্ট শেষে automatic rollback
    }
}
```

---

### ২. Factory ও Seeder Patterns — টেস্ট ডাটা তৈরি

কার্যকর ইন্টিগ্রেশন টেস্টের জন্য বাস্তবসম্মত টেস্ট ডাটা অত্যন্ত গুরুত্বপূর্ণ।

#### Laravel Factory — উন্নত কৌশল

```php
<?php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name'     => fake('bn_BD')->name(),
            'email'    => fake()->unique()->safeEmail(),
            'phone'    => '+880' . fake()->numerify('1#########'),
            'division' => fake()->randomElement([
                'Dhaka', 'Chittagong', 'Rajshahi',
                'Khulna', 'Barisal', 'Sylhet',
                'Rangpur', 'Mymensingh',
            ]),
            'password' => bcrypt('password'),
        ];
    }

    public function admin(): static
    {
        return $this->state(fn () => ['role' => 'admin']);
    }

    public function merchant(): static
    {
        return $this->state(fn () => [
            'role'          => 'merchant',
            'business_name' => fake()->company(),
            'trade_license' => fake()->numerify('TL-####-####'),
        ]);
    }

    public function withVerifiedPhone(): static
    {
        return $this->state(fn () => [
            'phone_verified_at' => now(),
        ]);
    }

    public function fromDhaka(): static
    {
        return $this->state(fn () => [
            'division' => 'Dhaka',
            'district' => fake()->randomElement(['Dhaka', 'Gazipur', 'Narayanganj']),
        ]);
    }
}
```

```php
<?php

namespace Database\Factories;

use App\Models\Order;
use Illuminate\Database\Eloquent\Factories\Factory;

class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition(): array
    {
        return [
            'user_id'          => User::factory(),
            'status'           => 'pending',
            'total'            => fake()->randomFloat(2, 100, 50000),
            'payment_method'   => fake()->randomElement(['bkash', 'nagad', 'card', 'cod']),
            'payment_status'   => 'unpaid',
            'shipping_address' => fake()->address(),
        ];
    }

    public function completed(): static
    {
        return $this->state(fn () => [
            'status'         => 'completed',
            'payment_status' => 'paid',
            'completed_at'   => now(),
        ]);
    }

    public function withItems(int $count = 3): static
    {
        return $this->afterCreating(function (Order $order) use ($count) {
            $products = Product::factory()->count($count)->create();
            foreach ($products as $product) {
                $order->items()->create([
                    'product_id' => $product->id,
                    'quantity'   => rand(1, 5),
                    'unit_price' => $product->price,
                ]);
            }
            $order->update(['total' => $order->items->sum(fn ($i) => $i->quantity * $i->unit_price)]);
        });
    }
}
```

#### টেস্টে ব্যবহার — জটিল সিনারিও

```php
/** @test */
public function monthly_sales_report_for_dhaka_merchants(): void
{
    // ৩ জন ঢাকার মার্চেন্ট, প্রত্যেকের ৫টি করে completed order
    $merchants = User::factory()
        ->count(3)
        ->merchant()
        ->fromDhaka()
        ->has(
            Order::factory()
                ->count(5)
                ->completed()
                ->withItems(2)
        )
        ->create();

    // রিপোর্ট API কল
    $response = $this->actingAs(User::factory()->admin()->create(), 'sanctum')
        ->getJson('/api/v1/reports/monthly-sales?division=Dhaka');

    $response->assertStatus(200)
        ->assertJsonPath('data.total_merchants', 3)
        ->assertJsonPath('data.total_orders', 15);
}
```

---

### ৩. Contract Testing (Pact) — Consumer-Driven Contracts

মাইক্রোসার্ভিস আর্কিটেকচারে Consumer-Driven Contract Testing নিশ্চিত করে যে সার্ভিসগুলো একে অপরের সাথে সামঞ্জস্যপূর্ণ (compatible) থাকে।

```
┌──────────────────────────────────────────────────┐
│            CONTRACT TESTING FLOW                  │
│                                                  │
│  Consumer (Order Service)    Provider (Payment)  │
│  ┌──────────────────────┐   ┌────────────────┐  │
│  │ 1. Define expected   │   │                │  │
│  │    interactions      │──▶│ 4. Verify the  │  │
│  │ 2. Generate contract │   │    contract    │  │
│  │    (Pact file)       │   │    against     │  │
│  │ 3. Publish to        │   │    real API    │  │
│  │    Pact Broker       │   │                │  │
│  └──────────────────────┘   └────────────────┘  │
│                                                  │
│  Contract = উভয় পক্ষের মধ্যে চুক্তি            │
│  Consumer যা আশা করে, Provider তা প্রদান করবে   │
└──────────────────────────────────────────────────┘
```

```javascript
// tests/contract/payment.consumer.pact.test.js
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { like, eachLike, regex } = MatchersV3;
const PaymentClient = require('../../src/clients/PaymentClient');

const provider = new PactV3({
    consumer: 'OrderService',
    provider: 'PaymentService',
    dir: './pacts',
});

describe('Payment Service Contract (Consumer Side)', () => {
    it('should initiate a bKash payment', async () => {
        // আমরা (Consumer) এটা আশা করি Provider থেকে
        await provider
            .given('a valid order exists')
            .uponReceiving('a request to initiate bKash payment')
            .withRequest({
                method: 'POST',
                path: '/api/payments/initiate',
                headers: { 'Content-Type': 'application/json' },
                body: {
                    orderId: like('ORD-12345'),
                    amount: like(2500.0),
                    currency: 'BDT',
                    method: 'bkash',
                    callbackUrl: like('https://example.com/callback'),
                },
            })
            .willRespondWith({
                status: 200,
                headers: { 'Content-Type': 'application/json' },
                body: {
                    paymentId: like('PAY-67890'),
                    redirectUrl: regex(
                        /https:\/\/.+\.bka\.sh\/.+/,
                        'https://sandbox.payment.bka.sh/redirect/abc123'
                    ),
                    status: 'initiated',
                    expiresAt: like('2025-01-15T10:30:00Z'),
                },
            })
            .executeTest(async (mockServer) => {
                const client = new PaymentClient(mockServer.url);
                const result = await client.initiatePayment({
                    orderId: 'ORD-12345',
                    amount: 2500.0,
                    currency: 'BDT',
                    method: 'bkash',
                    callbackUrl: 'https://example.com/callback',
                });

                expect(result.paymentId).toBeDefined();
                expect(result.redirectUrl).toMatch(/bka\.sh/);
                expect(result.status).toBe('initiated');
            });
    });

    it('should handle payment verification', async () => {
        await provider
            .given('a completed payment exists')
            .uponReceiving('a request to verify payment')
            .withRequest({
                method: 'GET',
                path: regex(/\/api\/payments\/PAY-\w+/, '/api/payments/PAY-67890'),
            })
            .willRespondWith({
                status: 200,
                body: {
                    paymentId: like('PAY-67890'),
                    transactionId: like('TRX-11111'),
                    status: 'completed',
                    amount: like(2500.0),
                    paidAt: like('2025-01-15T10:35:00Z'),
                },
            })
            .executeTest(async (mockServer) => {
                const client = new PaymentClient(mockServer.url);
                const result = await client.verifyPayment('PAY-67890');

                expect(result.status).toBe('completed');
                expect(result.transactionId).toBeDefined();
            });
    });
});
```

---

### ৪. API Snapshot Testing

API রেসপন্সের কাঠামো যাতে অজান্তে পরিবর্তন না হয়, তা নিশ্চিত করতে snapshot testing ব্যবহার করা হয়।

```javascript
// tests/integration/api-snapshot.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('API Response Snapshot Tests', () => {
    it('should match product list response structure', async () => {
        const res = await request(app)
            .get('/api/v1/products')
            .expect(200);

        // প্রথমবার চালালে snapshot তৈরি হবে
        // পরবর্তীতে কোনো পরিবর্তন হলে টেস্ট fail করবে
        expect(res.body).toMatchSnapshot({
            data: expect.arrayContaining([
                expect.objectContaining({
                    id: expect.any(Number),
                    name: expect.any(String),
                    price: expect.any(Number),
                    created_at: expect.any(String),
                }),
            ]),
            meta: {
                current_page: expect.any(Number),
                total: expect.any(Number),
                per_page: expect.any(Number),
            },
        });
    });

    it('should match error response format consistently', async () => {
        const res = await request(app)
            .post('/api/v1/products')
            .send({}) // খালি body
            .expect(422);

        expect(res.body).toMatchSnapshot({
            message: expect.any(String),
            errors: expect.any(Array),
        });
    });
});
```

#### Laravel API Snapshot

```php
/** @test */
public function product_api_response_matches_snapshot(): void
{
    Product::factory()->count(3)->create();

    $response = $this->getJson('/api/v1/products');

    // assertJsonStructure snapshot হিসেবে কাজ করে
    $response->assertJsonStructure([
        'data' => [
            '*' => [
                'id', 'name', 'slug', 'price', 'category',
                'stock', 'description', 'images',
                'created_at', 'updated_at',
            ],
        ],
        'links' => ['first', 'last', 'prev', 'next'],
        'meta'  => [
            'current_page', 'from', 'last_page',
            'per_page', 'to', 'total',
        ],
    ]);
}
```

---

### ৫. WebSocket Connection Testing

রিয়েল-টাইম ফিচার (চ্যাট, লাইভ নোটিফিকেশন, অর্ডার ট্র্যাকিং) টেস্ট করার জন্য WebSocket ইন্টিগ্রেশন টেস্ট প্রয়োজন।

```javascript
// tests/integration/websocket.test.js
const { createServer } = require('http');
const { Server } = require('socket.io');
const Client = require('socket.io-client');
const app = require('../../src/app');

describe('WebSocket Integration — Order Tracking', () => {
    let io, httpServer, clientSocket;
    const PORT = 4001;

    beforeAll((done) => {
        httpServer = createServer(app);
        io = new Server(httpServer);

        // WebSocket event handlers register
        require('../../src/websocket/handlers')(io);

        httpServer.listen(PORT, () => done());
    });

    beforeEach((done) => {
        clientSocket = Client(`http://localhost:${PORT}`, {
            auth: { token: 'valid_test_token' },
        });
        clientSocket.on('connect', done);
    });

    afterEach(() => {
        if (clientSocket.connected) clientSocket.disconnect();
    });

    afterAll(() => {
        io.close();
        httpServer.close();
    });

    it('should receive real-time order status updates', (done) => {
        const orderId = 'ORD-12345';

        clientSocket.emit('subscribe:order', { orderId });

        clientSocket.on('order:status-updated', (data) => {
            expect(data.orderId).toBe(orderId);
            expect(data.status).toBe('shipped');
            expect(data.message).toBe('Your order has been shipped');
            expect(data.timestamp).toBeDefined();
            done();
        });

        // সার্ভার সাইড থেকে স্ট্যাটাস আপডেট trigger
        setTimeout(() => {
            io.to(`order:${orderId}`).emit('order:status-updated', {
                orderId,
                status: 'shipped',
                message: 'Your order has been shipped',
                timestamp: new Date().toISOString(),
            });
        }, 100);
    });

    it('should handle authentication failure', (done) => {
        const unauthClient = Client(`http://localhost:${PORT}`, {
            auth: { token: 'invalid_token' },
        });

        unauthClient.on('connect_error', (err) => {
            expect(err.message).toBe('Authentication failed');
            unauthClient.disconnect();
            done();
        });
    });

    it('should handle multiple simultaneous room subscriptions', (done) => {
        const receivedUpdates = [];

        clientSocket.emit('subscribe:order', { orderId: 'ORD-001' });
        clientSocket.emit('subscribe:order', { orderId: 'ORD-002' });

        clientSocket.on('order:status-updated', (data) => {
            receivedUpdates.push(data);
            if (receivedUpdates.length === 2) {
                expect(receivedUpdates.map(u => u.orderId)).toEqual(
                    expect.arrayContaining(['ORD-001', 'ORD-002'])
                );
                done();
            }
        });

        setTimeout(() => {
            io.to('order:ORD-001').emit('order:status-updated', {
                orderId: 'ORD-001', status: 'processing',
            });
            io.to('order:ORD-002').emit('order:status-updated', {
                orderId: 'ORD-002', status: 'delivered',
            });
        }, 100);
    });
});
```

---

### ৬. Docker Compose দিয়ে Full Stack Test Environment

সম্পূর্ণ সিস্টেমের ইন্টিগ্রেশন টেস্ট চালানোর জন্য Docker Compose ব্যবহার করে পুরো ইনফ্রাস্ট্রাকচার সেটআপ করা যায়।

```yaml
# docker-compose.integration-test.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - APP_ENV=testing
      - DB_HOST=test-mysql
      - DB_DATABASE=app_test
      - REDIS_HOST=test-redis
      - QUEUE_CONNECTION=redis
      - CACHE_DRIVER=redis
    depends_on:
      test-mysql:
        condition: service_healthy
      test-redis:
        condition: service_healthy
    volumes:
      - ./tests:/app/tests
    command: ["php", "artisan", "test", "--testsuite=Integration"]

  test-mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: app_test
      MYSQL_ROOT_PASSWORD: secret
    tmpfs:
      - /var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 3s
      timeout: 3s
      retries: 15

  test-redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 10

  test-minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 5s
      timeout: 3s
      retries: 10
```

```bash
#!/bin/bash
# scripts/run-integration-tests.sh

set -e

echo "🔧 Integration test environment চালু হচ্ছে..."
docker compose -f docker-compose.integration-test.yml up -d test-mysql test-redis test-minio

echo "⏳ সার্ভিস healthy হওয়ার অপেক্ষা..."
docker compose -f docker-compose.integration-test.yml up --wait test-mysql test-redis

echo "🗃️  Database migration চালানো হচ্ছে..."
docker compose -f docker-compose.integration-test.yml run --rm app php artisan migrate --force

echo "🧪 Integration tests চালানো হচ্ছে..."
docker compose -f docker-compose.integration-test.yml run --rm app php artisan test \
    --testsuite=Integration \
    --parallel \
    --processes=4

EXIT_CODE=$?

echo "🧹 পরিষ্কার করা হচ্ছে..."
docker compose -f docker-compose.integration-test.yml down -v

exit $EXIT_CODE
```

---

### ৭. Parallel Test Execution — সমান্তরাল টেস্ট চালানো

বড় টেস্ট সুইটের জন্য parallel execution অপরিহার্য — ১০ মিনিটের টেস্ট ২-৩ মিনিটে শেষ হতে পারে।

#### Laravel Parallel Testing

```php
// phpunit.xml — parallel configuration
<phpunit>
    <testsuites>
        <testsuite name="Integration">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
</phpunit>
```

```bash
# Laravel parallel testing (artisan)
php artisan test --parallel --processes=4

# অথবা Paratest দিয়ে সরাসরি
./vendor/bin/paratest -p 4 --testsuite=Integration
```

```php
<?php
// tests/TestCase.php — parallel-safe base class

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;
use Illuminate\Support\Facades\ParallelTesting;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // প্রতিটি parallel process এর জন্য আলাদা DB
        // Laravel স্বয়ংক্রিয়ভাবে app_test_1, app_test_2, ... তৈরি করে
        ParallelTesting::setUpProcess(function (int $token) {
            // প্রতিটি process শুরুতে একবার চলে
            config(['database.connections.mysql.database' => "app_test_{$token}"]);
        });

        ParallelTesting::setUpTestCase(function (int $token, TestCase $testCase) {
            // প্রতিটি test class শুরুতে চলে
        });
    }
}
```

#### Jest Parallel Testing (Node.js)

```javascript
// jest.config.js
module.exports = {
    // Jest ডিফল্টভাবে parallel চালায়
    // worker সংখ্যা নিয়ন্ত্রণ
    maxWorkers: '50%',

    // Integration tests আলাদা project হিসেবে
    projects: [
        {
            displayName: 'integration',
            testMatch: ['<rootDir>/tests/integration/**/*.test.js'],
            // প্রতিটি worker-এর জন্য আলাদা setup
            globalSetup: '<rootDir>/tests/setup/globalSetup.js',
            globalTeardown: '<rootDir>/tests/setup/globalTeardown.js',
            setupFilesAfterFramework: ['<rootDir>/tests/setup/perWorker.js'],
        },
    ],
};
```

```javascript
// tests/setup/perWorker.js
const workerId = process.env.JEST_WORKER_ID;

// প্রতিটি worker আলাদা ডাটাবেস ব্যবহার করে
process.env.DB_DATABASE = `app_test_${workerId}`;
process.env.REDIS_DB = workerId;

// বিচ্ছিন্নতা নিশ্চিত করতে port offset
process.env.TEST_PORT = 3000 + parseInt(workerId);
```

---

### ৮. CI/CD Integration Test Pipeline

#### GitHub Actions Pipeline

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  integration-tests-laravel:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: app_test
          MYSQL_ROOT_PASSWORD: secret
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: pdo_mysql, redis, gd
          coverage: xdebug

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run migrations
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: app_test
          DB_USERNAME: root
          DB_PASSWORD: secret
        run: php artisan migrate --force

      - name: Run Integration Tests
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: app_test
          DB_USERNAME: root
          DB_PASSWORD: secret
          REDIS_HOST: 127.0.0.1
        run: |
          php artisan test --testsuite=Integration \
            --parallel --processes=4 \
            --coverage-clover=coverage.xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml
          flags: integration

  integration-tests-node:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: app_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run Integration Tests
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/app_test
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test
        run: npx jest --testPathPattern=integration --forceExit --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## ✅ Best Practices — সর্বোত্তম চর্চা

### ১. টেস্ট আইসোলেশন নিশ্চিত করুন

```php
// ✅ ভালো — প্রতিটি টেস্ট স্বনির্ভর
/** @test */
public function user_can_place_order(): void
{
    // এই টেস্টের নিজস্ব ডাটা তৈরি
    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 10]);

    $response = $this->actingAs($user)->postJson('/api/orders', [...]);
    $response->assertStatus(201);
}

// ❌ খারাপ — অন্য টেস্টের ডাটার উপর নির্ভরশীল
/** @test */
public function user_can_place_order(): void
{
    // ধরে নেওয়া হচ্ছে DB তে user আছে — ভঙ্গুর!
    $response = $this->postJson('/api/orders', [...]);
}
```

### ২. Arrange-Act-Assert প্যাটার্ন মেনে চলুন

```php
/** @test */
public function discount_applied_for_bulk_orders(): void
{
    // ARRANGE — প্রস্তুতি
    $user = User::factory()->create();
    $product = Product::factory()->create(['price' => 100, 'bulk_discount_threshold' => 10]);

    // ACT — কাজ
    $response = $this->actingAs($user, 'sanctum')
        ->postJson('/api/v1/orders', [
            'items' => [['product_id' => $product->id, 'quantity' => 15]],
            'shipping_address' => 'Gulshan, Dhaka',
            'payment_method' => 'cod',
        ]);

    // ASSERT — যাচাই
    $response->assertStatus(201);
    $order = Order::latest()->first();
    $this->assertTrue($order->total < 1500); // ডিসকাউন্ট প্রয়োগ হয়েছে
}
```

### ৩. টেস্ট ডাটা ফ্যাক্টরি ব্যবহার করুন

```javascript
// ✅ ভালো — ফ্যাক্টরি প্যাটার্ন
const { createTestUser, createTestProduct } = require('../factories');

it('should create order', async () => {
    const user = await createTestUser({ role: 'customer' });
    const product = await createTestProduct({ price: 500, stock: 10 });
    // ...
});

// ❌ খারাপ — হার্ডকোডেড ডাটা সরাসরি টেস্টে
it('should create order', async () => {
    await db.query("INSERT INTO users (id, name, email) VALUES (1, 'Test', 'test@x.com')");
    await db.query("INSERT INTO products (id, name, price) VALUES (1, 'Item', 500)");
    // ...
});
```

### ৪. টাইমআউট ও ফ্লেকি টেস্ট সামলান

```javascript
// ✅ ভালো — নির্দিষ্ট timeout এবং retry logic
jest.setTimeout(15000);

it('should process payment webhook', async () => {
    // waitFor pattern ব্যবহার করুন polling-এর বদলে
    await waitForCondition(
        async () => {
            const order = await Order.findByPk(orderId);
            return order.status === 'paid';
        },
        { timeout: 5000, interval: 200 }
    );
});
```

### ৫. Environment Configuration আলাদা রাখুন

```
# .env.testing
APP_ENV=testing
DB_DATABASE=app_test
CACHE_DRIVER=array
QUEUE_CONNECTION=sync
MAIL_MAILER=array
BKASH_MODE=sandbox
SMS_GATEWAY=mock
```

---

## ⚠️ Anti-patterns — যা এড়িয়ে চলবেন

### ১. টেস্টের মধ্যে পারস্পরিক নির্ভরতা

```php
// ❌ অত্যন্ত খারাপ — টেস্ট ক্রম-নির্ভর
public function test_1_create_user(): void
{
    $this->postJson('/api/users', ['name' => 'Rahim'])->assertStatus(201);
}

public function test_2_user_can_login(): void
{
    // test_1 আগে চলতে হবে — ভঙ্গুর!
    $this->postJson('/api/login', ['name' => 'Rahim'])->assertStatus(200);
}
```

### ২. Sleep() ব্যবহার করে টাইমিং নিয়ন্ত্রণ

```javascript
// ❌ খারাপ — নির্দিষ্ট সময়ে sleep
it('should update after processing', async () => {
    await triggerProcessing();
    await new Promise(r => setTimeout(r, 3000)); // CI তে fail হতে পারে
    const result = await getResult();
});

// ✅ ভালো — condition-based waiting
it('should update after processing', async () => {
    await triggerProcessing();
    const result = await pollUntil(() => getResult(), {
        condition: r => r.status === 'done',
        timeout: 10000,
    });
    expect(result.status).toBe('done');
});
```

### ৩. Production Database এ টেস্ট চালানো

```php
// ❌ বিপজ্জনক — production DB ব্যবহার
// .env এ DB_DATABASE=production_db রেখে টেস্ট চালানো

// ✅ নিরাপদ — আলাদা test database এবং safeguard
// TestCase.php এ check যোগ করুন
protected function setUp(): void
{
    parent::setUp();

    if (config('database.connections.mysql.database') === 'production_db') {
        $this->fail('DANGER: Tests are running against production database!');
    }
}
```

### ৪. অতিরিক্ত Mocking — সব কিছু fake করে দেওয়া

```php
// ❌ খারাপ — সব কিছু mock করলে integration test-র উদ্দেশ্য নষ্ট হয়
/** @test */
public function create_order(): void
{
    $mockRepo = Mockery::mock(OrderRepository::class);
    $mockService = Mockery::mock(PaymentService::class);
    $mockCache = Mockery::mock(CacheManager::class);
    // এটা unit test হয়ে গেছে, integration test নয়!
}

// ✅ ভালো — শুধু বাহ্যিক সার্ভিস mock করুন
/** @test */
public function create_order(): void
{
    Http::fake([...]); // শুধু বাহ্যিক API mock
    // DB, Cache, Queue — সব real ব্যবহার করুন
}
```

### ৫. Flaky Test উপেক্ষা করা

```javascript
// ❌ খারাপ — flaky test কে skip করে এগিয়ে যাওয়া
it.skip('should handle concurrent orders', async () => { ... });

// ✅ ভালো — মূল কারণ খুঁজে বের করা
// সাধারণত flaky test-এর কারণ:
// - Race condition (সমাধান: proper locking/waiting)
// - Shared state (সমাধান: test isolation)
// - Time dependency (সমাধান: clock mocking)
// - Network dependency (সমাধান: proper mocking)
```

---

## 📋 সারসংক্ষেপ

### মূল পয়েন্টগুলো

```
┌──────────────────────────────────────────────────────────────┐
│               INTEGRATION TESTING CHECKLIST                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ API Endpoints — সব route সঠিক response দিচ্ছে          │
│  ✅ Database — CRUD, relations, constraints কাজ করছে        │
│  ✅ Authentication — login, token, permission ঠিক আছে      │
│  ✅ External APIs — bKash, SMS gateway mock করে টেস্ট       │
│  ✅ Queue Jobs — dispatch, process, retry সঠিক              │
│  ✅ Cache — hit, miss, invalidation কাজ করছে               │
│  ✅ File Upload — store, validate, delete ঠিক আছে          │
│  ✅ Error Handling — সব edge case handle হচ্ছে              │
│  ✅ Test Isolation — প্রতিটি টেস্ট স্বনির্ভর               │
│  ✅ CI/CD — পাইপলাইনে স্বয়ংক্রিয়ভাবে চলছে               │
│                                                              │
│  📊 লক্ষ্যমাত্রা:                                           │
│     - Critical paths: ১০০% coverage                          │
│     - Integration tests: মোট টেস্টের ২০-৩০%                 │
│     - Execution time: ৫ মিনিটের নিচে (parallel)             │
│     - Flaky rate: ১% এর কম                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### কখন কোন ধরনের টেস্ট লিখবেন?

| পরিস্থিতি | টেস্ট ধরন |
|---|---|
| নতুন API endpoint | HTTP Integration Test |
| জটিল DB query/relationship | Database Integration Test |
| বাহ্যিক API (bKash, SMS) | Mocked External Service Test |
| ব্যাকগ্রাউন্ড জব | Queue Integration Test |
| ক্যাশিং লজিক | Cache Integration Test |
| ফাইল আপলোড | Storage Integration Test |
| মাইক্রোসার্ভিস যোগাযোগ | Contract Test (Pact) |
| রিয়েল-টাইম ফিচার | WebSocket Integration Test |

### বাংলাদেশ প্রসঙ্গে বিশেষ বিবেচনা

- **পেমেন্ট গেটওয়ে:** bKash, Nagad, SSLCommerz — সবসময় sandbox mode এ টেস্ট করুন। `Http::fake()` বা `nock` দিয়ে timeout, error, success সব সিনারিও কভার করুন।
- **SMS Gateway:** BulkSMSBD, Infobip — OTP rate limiting, delivery status, বাংলা SMS encoding টেস্ট করুন।
- **ভ্যাট ক্যালকুলেশন:** বাংলাদেশের ১৫% VAT এবং বিভিন্ন পণ্যের ভিন্ন ভ্যাট হার সঠিকভাবে টেস্ট করুন।
- **মোবাইল নম্বর ভ্যালিডেশন:** বাংলাদেশি মোবাইল নম্বর ফরম্যাট (`+8801XXXXXXXXX`) সঠিকভাবে validate হচ্ছে কিনা টেস্ট করুন।

> **মনে রাখুন:** ইন্টিগ্রেশন টেস্ট আপনার সিস্টেমের বিভিন্ন অংশ একসাথে সঠিকভাবে কাজ করছে কিনা তার নিশ্চয়তা দেয়। Unit test যেখানে শেষ, সেখানে integration test শুরু — আর integration test যেখানে শেষ, সেখানে E2E test শুরু। তিনটি মিলেই সম্পূর্ণ টেস্টিং কৌশল।
