# 📌 লেয়ার্ড (N-Tier) আর্কিটেকচার — Layered Architecture

## 📖 সূচিপত্র (Table of Contents)

- [ভূমিকা ও ইতিহাস](#-ভূমিকা-ও-ইতিহাস)
- [মূল লেয়ারসমূহ](#-মূল-লেয়ারসমূহ-the-main-layers)
- [Strict vs Relaxed লেয়ারিং](#-strict-vs-relaxed-লেয়ারিং)
- [সুবিধা ও অসুবিধা](#-সুবিধা-ও-অসুবিধা)
- [কখন ভালো কাজ করে](#-কখন-ভালো-কাজ-করে)
- [PHP (Laravel) সম্পূর্ণ উদাহরণ](#-php-laravel-সম্পূর্ণ-উদাহরণ)
- [JS (Express) সম্পূর্ণ উদাহরণ](#-js-express-সম্পূর্ণ-উদাহরণ)
- [অন্যান্য আর্কিটেকচারের সাথে তুলনা](#-অন্যান্য-আর্কিটেকচারের-সাথে-তুলনা)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)
- [বাস্তব উদাহরণ](#-বাস্তব-উদাহরণ)

---

## 📖 ভূমিকা ও ইতিহাস

লেয়ার্ড আর্কিটেকচার হলো সফটওয়্যার ডিজাইনের সবচেয়ে পুরনো এবং সবচেয়ে বেশি ব্যবহৃত প্যাটার্ন।
১৯৯০-এর দশক থেকে এটি ওয়েব অ্যাপ্লিকেশনের **ডিফল্ট আর্কিটেকচার** হিসেবে বিবেচিত হয়ে আসছে।
আপনি যদি কোনো আর্কিটেকচার নির্দিষ্ট না করে কোডিং শুরু করেন, তাহলে সম্ভবত আপনি লেয়ার্ড
আর্কিটেকচারই তৈরি করছেন!

মূল ধারণা সহজ — **অ্যাপ্লিকেশনকে অনুভূমিক স্তরে ভাগ করুন, প্রতিটি স্তরের একটি নির্দিষ্ট দায়িত্ব আছে।**

```
╔══════════════════════════════════════════════════════════════════╗
║                  LAYERED ARCHITECTURE                          ║
║                  লেয়ার্ড আর্কিটেকচার                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                ║
║   ┌──────────────────────────────────────────────────────┐     ║
║   │          📱 PRESENTATION LAYER                       │     ║
║   │          প্রেজেন্টেশন লেয়ার                           │     ║
║   │   (UI, Controllers, Views, API Endpoints)            │     ║
║   └──────────────────────┬───────────────────────────────┘     ║
║                          │                                     ║
║                          ▼                                     ║
║   ┌──────────────────────────────────────────────────────┐     ║
║   │          ⚙️  BUSINESS LOGIC LAYER                    │     ║
║   │          বিজনেস লজিক লেয়ার                           │     ║
║   │   (Services, Validation, Workflows)                  │     ║
║   └──────────────────────┬───────────────────────────────┘     ║
║                          │                                     ║
║                          ▼                                     ║
║   ┌──────────────────────────────────────────────────────┐     ║
║   │          💾 DATA ACCESS LAYER                        │     ║
║   │          ডাটা অ্যাক্সেস লেয়ার                         │     ║
║   │   (Repositories, DAOs, ORM Queries)                  │     ║
║   └──────────────────────┬───────────────────────────────┘     ║
║                          │                                     ║
║                          ▼                                     ║
║   ┌──────────────────────────────────────────────────────┐     ║
║   │          🗄️  DATABASE LAYER                          │     ║
║   │          ডাটাবেজ লেয়ার                                │     ║
║   │   (MySQL, PostgreSQL, MongoDB)                       │     ║
║   └──────────────────────────────────────────────────────┘     ║
║                                                                ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🏠 বাস্তব উপমা — ঢাকার একটি রেস্তোরাঁ

```
  ঢাকার রেস্তোরাঁ                         সফটওয়্যার অ্যাপ
  ════════════════                         ═══════════════════

  👤 কাস্টমার (Customer)                   👤 ইউজার (User)
       │                                        │
       ▼                                        ▼
  ┌─────────────┐                         ┌─────────────────┐
  │  🧑‍🍳 ওয়েটার   │  ← অর্ডার নেয়,         │  Presentation   │
  │  (Waiter)   │    খাবার সার্ভ করে       │  Layer          │
  └──────┬──────┘                         └────────┬────────┘
         │                                         │
         ▼                                         ▼
  ┌─────────────┐                         ┌─────────────────┐
  │  👨‍🍳 শেফ     │  ← রান্না করে,           │  Business Logic │
  │  (Chef)     │    রেসিপি জানে           │  Layer          │
  └──────┬──────┘                         └────────┬────────┘
         │                                         │
         ▼                                         ▼
  ┌─────────────┐                         ┌─────────────────┐
  │  📦 প্যান্ট্রি  │  ← মালামাল বের করে,     │  Data Access    │
  │  (Pantry)   │    স্টক চেক করে          │  Layer          │
  └──────┬──────┘                         └────────┬────────┘
         │                                         │
         ▼                                         ▼
  ┌─────────────┐                         ┌─────────────────┐
  │  🏪 গুদাম    │  ← সব মালামাল           │  Database       │
  │  (Storage)  │    সংরক্ষিত থাকে         │  Layer          │
  └─────────────┘                         └─────────────────┘

  ওয়েটার সরাসরি গুদামে যায় না!             Presentation সরাসরি
  শেফের মাধ্যমে অর্ডার যায়।                DB access করে না!
```

> **মনে রাখুন:** ওয়েটার কখনোই সরাসরি গুদামে গিয়ে মালামাল বের করে না।
> ঠিক তেমনি, Presentation Layer সরাসরি Database-এ যাবে না।

---

## 📊 মূল লেয়ারসমূহ (The Main Layers)

### 🔹 a) Presentation Layer (প্রেজেন্টেশন লেয়ার)

এটি হলো ইউজারের সাথে সরাসরি যোগাযোগের স্তর। UI, Controller, View, API endpoint — সবকিছু এখানে থাকে।

**দায়িত্ব:**
- ইউজারের ইনপুট গ্রহণ করা
- রেসপন্স ফরম্যাট করে দেখানো (HTML, JSON)
- রাউটিং পরিচালনা করা
- রিকোয়েস্ট ভ্যালিডেশন (বেসিক)

**PHP (Laravel) Controller উদাহরণ:**

```php
// app/Http/Controllers/ProductController.php

class ProductController extends Controller
{
    private ProductService $productService;

    public function __construct(ProductService $productService)
    {
        $this->productService = $productService;
    }

    // ইউজার থেকে রিকোয়েস্ট আসে, সার্ভিসে পাঠায়
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name'  => 'required|string|max:255',
            'price' => 'required|numeric|min:0',
        ]);

        $product = $this->productService->createProduct($validated);

        return response()->json([
            'message' => 'পণ্য সফলভাবে তৈরি হয়েছে',
            'data'    => $product,
        ], 201);
    }
}
```

**JS (Express) Route Handler উদাহরণ:**

```javascript
// controllers/productController.js

class ProductController {
    constructor(productService) {
        this.productService = productService;
    }

    // ইউজার থেকে রিকোয়েস্ট আসে, সার্ভিসে পাঠায়
    async createProduct(req, res) {
        try {
            const { name, price } = req.body;
            const product = await this.productService.createProduct({ name, price });

            return res.status(201).json({
                message: 'পণ্য সফলভাবে তৈরি হয়েছে',
                data: product,
            });
        } catch (error) {
            return res.status(400).json({ error: error.message });
        }
    }
}

module.exports = ProductController;
```

---

### 🔹 b) Business Logic Layer (বিজনেস লজিক লেয়ার)

এটি অ্যাপ্লিকেশনের **হৃদপিণ্ড**। সমস্ত ব্যবসায়িক নিয়ম, গণনা, এবং ওয়ার্কফ্লো এখানে থাকে।
Daraz-এ অর্ডার প্লেস করলে ডিসকাউন্ট ক্যালকুলেশন, স্টক চেক, পেমেন্ট ভেরিফিকেশন — সব এই লেয়ারে হয়।

**দায়িত্ব:**
- বিজনেস রুল প্রয়োগ করা
- ডাটা ভ্যালিডেশন (বিজনেস লেভেল)
- ক্যালকুলেশন ও ট্রান্সফর্মেশন
- ওয়ার্কফ্লো অর্কেস্ট্রেশন

**PHP Service উদাহরণ — Daraz-স্টাইল OrderService:**

```php
// app/Services/OrderService.php

class OrderService
{
    private OrderRepository $orderRepo;
    private ProductRepository $productRepo;
    private PaymentService $paymentService;

    public function __construct(
        OrderRepository $orderRepo,
        ProductRepository $productRepo,
        PaymentService $paymentService
    ) {
        $this->orderRepo = $orderRepo;
        $this->productRepo = $productRepo;
        $this->paymentService = $paymentService;
    }

    public function placeOrder(array $data): Order
    {
        // ১. স্টক চেক করো
        $product = $this->productRepo->findById($data['product_id']);
        if ($product->stock < $data['quantity']) {
            throw new \Exception('পর্যাপ্ত স্টক নেই!');
        }

        // ২. মূল্য হিসাব করো (ডিসকাউন্ট সহ)
        $totalPrice = $this->calculatePrice($product, $data['quantity']);

        // ৩. অর্ডার তৈরি করো
        $order = $this->orderRepo->create([
            'product_id' => $data['product_id'],
            'quantity'   => $data['quantity'],
            'total'      => $totalPrice,
            'status'     => 'pending',
        ]);

        // ৪. স্টক আপডেট করো
        $this->productRepo->decrementStock($data['product_id'], $data['quantity']);

        return $order;
    }

    private function calculatePrice(Product $product, int $quantity): float
    {
        $subtotal = $product->price * $quantity;

        // ১০০০ টাকার বেশি হলে ৫% ছাড়
        if ($subtotal > 1000) {
            $subtotal *= 0.95;
        }

        return round($subtotal, 2);
    }
}
```

**JS Service উদাহরণ:**

```javascript
// services/orderService.js

class OrderService {
    constructor(orderRepo, productRepo) {
        this.orderRepo = orderRepo;
        this.productRepo = productRepo;
    }

    async placeOrder(data) {
        const product = await this.productRepo.findById(data.productId);

        if (product.stock < data.quantity) {
            throw new Error('পর্যাপ্ত স্টক নেই!');
        }

        const totalPrice = this.calculatePrice(product, data.quantity);

        const order = await this.orderRepo.create({
            productId: data.productId,
            quantity: data.quantity,
            total: totalPrice,
            status: 'pending',
        });

        await this.productRepo.decrementStock(data.productId, data.quantity);

        return order;
    }

    calculatePrice(product, quantity) {
        let subtotal = product.price * quantity;

        // ১০০০ টাকার বেশি হলে ৫% ছাড়
        if (subtotal > 1000) {
            subtotal *= 0.95;
        }

        return Math.round(subtotal * 100) / 100;
    }
}

module.exports = OrderService;
```

---

### 🔹 c) Data Access Layer (ডাটা অ্যাক্সেস লেয়ার)

এই লেয়ার ডাটাবেজের সাথে সরাসরি কথা বলে। ORM queries, raw SQL, CRUD operations — সব এখানে।
বিজনেস লজিক লেয়ার জানে না ডাটা MySQL-এ আছে নাকি MongoDB-তে — সে শুধু Repository-কে বলে "ডাটা দাও"।

**PHP Repository উদাহরণ (Eloquent):**

```php
// app/Repositories/ProductRepository.php

class ProductRepository
{
    public function findById(int $id): Product
    {
        $product = Product::find($id);

        if (!$product) {
            throw new \Exception('পণ্য খুঁজে পাওয়া যায়নি');
        }

        return $product;
    }

    public function create(array $data): Product
    {
        return Product::create($data);
    }

    public function decrementStock(int $productId, int $quantity): void
    {
        Product::where('id', $productId)->decrement('stock', $quantity);
    }

    public function getAll(int $perPage = 15)
    {
        return Product::orderBy('created_at', 'desc')->paginate($perPage);
    }
}
```

**JS Repository উদাহরণ (Knex):**

```javascript
// repositories/productRepository.js

class ProductRepository {
    constructor(db) {
        this.db = db; // Knex instance
    }

    async findById(id) {
        const product = await this.db('products').where({ id }).first();

        if (!product) {
            throw new Error('পণ্য খুঁজে পাওয়া যায়নি');
        }

        return product;
    }

    async create(data) {
        const [id] = await this.db('products').insert(data);
        return this.findById(id);
    }

    async decrementStock(productId, quantity) {
        await this.db('products')
            .where({ id: productId })
            .decrement('stock', quantity);
    }
}

module.exports = ProductRepository;
```

---

### 🔹 d) Database Layer (ডাটাবেজ লেয়ার)

এটি প্রকৃত ডাটাবেজ — MySQL, PostgreSQL, MongoDB ইত্যাদি। মাইগ্রেশন ও স্কিমা এখানে সংজ্ঞায়িত হয়।

**Laravel Migration উদাহরণ:**

```php
// database/migrations/2024_01_01_create_products_table.php

public function up(): void
{
    Schema::create('products', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->text('description')->nullable();
        $table->decimal('price', 10, 2);
        $table->integer('stock')->default(0);
        $table->string('category')->nullable();
        $table->timestamps();
    });
}
```

**Knex Migration উদাহরণ:**

```javascript
// migrations/20240101_create_products.js

exports.up = function (knex) {
    return knex.schema.createTable('products', (table) => {
        table.increments('id').primary();
        table.string('name').notNullable();
        table.text('description');
        table.decimal('price', 10, 2).notNullable();
        table.integer('stock').defaultTo(0);
        table.string('category');
        table.timestamps(true, true);
    });
};

exports.down = function (knex) {
    return knex.schema.dropTable('products');
};
```

---

## 🎯 Strict vs Relaxed লেয়ারিং

লেয়ার্ড আর্কিটেকচারে দুই ধরনের নিয়ম মানা যায়: **কঠোর (Strict)** এবং **শিথিল (Relaxed)**।

```
     কঠোর (Strict) লেয়ারিং              শিথিল (Relaxed) লেয়ারিং
     ═══════════════════════             ═══════════════════════

  ┌──────────────────────┐           ┌──────────────────────┐
  │    Presentation      │           │    Presentation      │
  └──────────┬───────────┘           └───┬──────────┬───────┘
             │ (শুধু নিচের                │          │
             │  লেয়ারকে কল)              │          │
             ▼                           ▼          │
  ┌──────────────────────┐           ┌──────────────┤──────┐
  │    Business Logic    │           │  Business    │      │
  └──────────┬───────────┘           │  Logic       │      │
             │ (শুধু নিচের                └──────┬──────┘      │
             │  লেয়ারকে কল)                     │             │
             ▼                              ▼             │
  ┌──────────────────────┐           ┌──────────────┐     │
  │    Data Access       │           │  Data Access │     │
  └──────────┬───────────┘           └──────┬───────┘     │
             │                              │             │
             ▼                              ▼             ▼
  ┌──────────────────────┐           ┌────────────────────┐
  │    Database          │           │  Database          │
  └──────────────────────┘           └────────────────────┘

  প্রতিটি লেয়ার শুধু তার             যেকোনো নিচের লেয়ারকে
  ঠিক নিচের লেয়ারকে কল করে।          সরাসরি কল করতে পারে।
```

### ❌ Bad: লেয়ার ভাঙা — Presentation সরাসরি DB কল করছে

```php
// ❌ খারাপ — Controller সরাসরি ডাটাবেজ কোয়েরি করছে!
class ProductController extends Controller
{
    public function index()
    {
        // Presentation Layer থেকে সরাসরি DB!
        $products = DB::table('products')
            ->where('stock', '>', 0)
            ->orderBy('price', 'asc')
            ->get();

        return response()->json($products);
    }
}
```

**কেন খারাপ?**
- বিজনেস লজিক Controller-এ চলে গেছে
- টেস্ট করা কঠিন (DB mock করতে হবে)
- একই কোয়েরি বিভিন্ন জায়গায় কপি-পেস্ট হবে

### ✅ Good: Strict লেয়ারিং মেনে চলা

```php
// ✅ ভালো — প্রতিটি লেয়ার তার দায়িত্ব পালন করছে

// Controller → শুধু রিকোয়েস্ট/রেসপন্স হ্যান্ডেল করে
class ProductController extends Controller
{
    public function __construct(private ProductService $service) {}

    public function index()
    {
        $products = $this->service->getAvailableProducts();
        return response()->json($products);
    }
}

// Service → বিজনেস লজিক ধারণ করে
class ProductService
{
    public function __construct(private ProductRepository $repo) {}

    public function getAvailableProducts()
    {
        return $this->repo->findAvailableProducts();
    }
}

// Repository → ডাটাবেজ অপারেশন পরিচালনা করে
class ProductRepository
{
    public function findAvailableProducts()
    {
        return Product::where('stock', '>', 0)
            ->orderBy('price', 'asc')
            ->get();
    }
}
```

### কখন Strict, কখন Relaxed?

| বিষয়                 | Strict                   | Relaxed                  |
| --------------------- | ------------------------ | ------------------------ |
| জটিলতা               | বেশি বয়লারপ্লেট কোড      | কম কোড                   |
| নিরাপত্তা             | বেশি নিরাপদ              | ঝুঁকি আছে               |
| পারফরম্যান্স           | সামান্য ধীর               | দ্রুত                    |
| রক্ষণাবেক্ষণ          | সহজ                      | কঠিন হতে পারে           |
| সেরা ব্যবহার          | বড় প্রজেক্ট, টিম ওয়ার্ক  | ছোট প্রজেক্ট, প্রোটোটাইপ |

---

## 📊 সুবিধা ও অসুবিধা

### ✅ সুবিধাসমূহ (Advantages)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    ✅ সুবিধাসমূহ                         │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  🟢 সহজে বোঝা যায় (Easy to Understand)                  │
  │     → নতুন ডেভেলপার দ্রুত কাজ শুরু করতে পারে             │
  │                                                          │
  │  🟢 Separation of Concerns                               │
  │     → প্রতিটি লেয়ারের দায়িত্ব আলাদা                     │
  │                                                          │
  │  🟢 টিম ওয়ার্ক সহজ                                      │
  │     → একজন Frontend, একজন Backend, একজন DB              │
  │                                                          │
  │  🟢 পৃথকভাবে টেস্টিং সম্ভব                               │
  │     → Service টেস্ট করতে DB লাগে না (Mock ব্যবহার)       │
  │                                                          │
  │  🟢 ইন্ডাস্ট্রি স্ট্যান্ডার্ড                               │
  │     → প্রচুর ডকুমেন্টেশন ও রিসোর্স পাওয়া যায়            │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### ❌ অসুবিধাসমূহ (Disadvantages)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    ❌ অসুবিধাসমূহ                        │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  🔴 সময়ের সাথে "Big Ball of Mud" হতে পারে               │
  │     → নিয়ম না মানলে লেয়ার মিশে যায়                     │
  │                                                          │
  │  🔴 পরিবর্তন সব লেয়ারে ছড়িয়ে পড়ে (Change Cascade)      │
  │     → একটি ফিল্ড যোগ করলে ৪টি লেয়ারে পরিবর্তন লাগে     │
  │                                                          │
  │  🔴 পারফরম্যান্স ওভারহেড                                 │
  │     → প্রতিটি রিকোয়েস্ট সব লেয়ারের মধ্য দিয়ে যায়       │
  │                                                          │
  │  🔴 টেকনিক্যাল কনসার্ন দিয়ে ভাগ করা হয়                  │
  │     → বিজনেস ডোমেইন অনুযায়ী নয়                         │
  │                                                          │
  │  🔴 সংলগ্ন লেয়ারের মধ্যে টাইট কাপলিং                    │
  │     → একটি লেয়ার বদলালে পাশের লেয়ারেও প্রভাব পড়ে       │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### Change Cascade সমস্যা — একটি ফিল্ড যোগ করার প্রভাব:

```
  "phone_number" ফিল্ড যোগ করতে হবে:

  📱 Presentation  →  ফর্মে ফিল্ড যোগ, ভ্যালিডেশন যোগ
         │
         ▼
  ⚙️  Business      →  সার্ভিসে প্যারামিটার যোগ
         │
         ▼
  💾 Data Access   →  রিপোজিটরিতে ফিল্ড ম্যাপিং যোগ
         │
         ▼
  🗄️  Database      →  মাইগ্রেশন তৈরি, কলাম যোগ

  মোট পরিবর্তন: ৪টি লেয়ারে ৪-৮টি ফাইল!
```

---

## 🏠 কখন ভালো কাজ করে

লেয়ার্ড আর্কিটেকচার **এই পরিস্থিতিতে** চমৎকার কাজ করে:

```
  ✅ কখন ব্যবহার করবেন:
  ┌───────────────────────────────────────────────────┐
  │                                                   │
  │  • সাধারণ CRUD অ্যাপ্লিকেশন                       │
  │    (ইনভেন্টরি ম্যানেজমেন্ট, ব্লগ, CMS)           │
  │                                                   │
  │  • ছোট-মাঝারি প্রজেক্ট                            │
  │    (২-৫ জন ডেভেলপারের টিম)                        │
  │                                                   │
  │  • আর্কিটেকচারে নতুন টিম                          │
  │    (শেখার জন্য সবচেয়ে সহজ)                       │
  │                                                   │
  │  • ট্র্যাডিশনাল ওয়েব অ্যাপ                        │
  │    (Laravel, Express, Django প্রজেক্ট)             │
  │                                                   │
  │  • সরকারি/এন্টারপ্রাইজ সিস্টেম                     │
  │    (বাংলাদেশের ই-গভর্ন্যান্স প্রজেক্ট)             │
  │                                                   │
  └───────────────────────────────────────────────────┘
```

**উদাহরণ:** ঢাকার একটি দোকানের জন্য ইনভেন্টরি ম্যানেজমেন্ট সিস্টেম — পণ্য যোগ করা, স্টক
আপডেট, বিক্রি রেকর্ড। এই ধরনের সিস্টেমে লেয়ার্ড আর্কিটেকচার যথেষ্ট।

---

## 📌 PHP (Laravel) সম্পূর্ণ উদাহরণ

একটি অর্ডার তৈরির সম্পূর্ণ প্রবাহ — Route থেকে Database পর্যন্ত:

```
  রিকোয়েস্ট ফ্লো:
  ════════════════

  POST /api/orders  ──►  Route  ──►  Controller  ──►  Service  ──►  Repository  ──►  DB
                                         │                │              │
                                    ভ্যালিডেশন       বিজনেস রুল     DB কোয়েরি
                                    রেসপন্স         ক্যালকুলেশন    Eloquent ORM
```

### ফোল্ডার স্ট্রাকচার:

```
  app/
  ├── Http/
  │   ├── Controllers/
  │   │   └── OrderController.php        ← Presentation
  │   └── Requests/
  │       └── CreateOrderRequest.php     ← Validation
  ├── Services/
  │   └── OrderService.php               ← Business Logic
  ├── Repositories/
  │   └── OrderRepository.php            ← Data Access
  │   └── ProductRepository.php          ← Data Access
  └── Models/
      ├── Order.php                      ← Database (Eloquent)
      └── Product.php                    ← Database (Eloquent)
```

### ১. Route (রাউট):

```php
// routes/api.php
Route::post('/orders', [OrderController::class, 'store']);
```

### ২. Controller (Presentation Layer):

```php
// app/Http/Controllers/OrderController.php

class OrderController extends Controller
{
    public function __construct(private OrderService $orderService) {}

    public function store(CreateOrderRequest $request)
    {
        try {
            $order = $this->orderService->placeOrder($request->validated());

            return response()->json([
                'success' => true,
                'message' => 'অর্ডার সফলভাবে তৈরি হয়েছে!',
                'data'    => $order,
            ], 201);

        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage(),
            ], 422);
        }
    }
}
```

### ৩. Service (Business Logic Layer):

```php
// app/Services/OrderService.php

class OrderService
{
    public function __construct(
        private OrderRepository $orderRepo,
        private ProductRepository $productRepo
    ) {}

    public function placeOrder(array $data): Order
    {
        $product = $this->productRepo->findById($data['product_id']);

        if ($product->stock < $data['quantity']) {
            throw new \Exception('পর্যাপ্ত স্টক নেই!');
        }

        $total = $product->price * $data['quantity'];

        // bKash/Nagad পেমেন্ট হলে ২% ক্যাশব্যাক
        if (in_array($data['payment_method'], ['bkash', 'nagad'])) {
            $total *= 0.98;
        }

        $order = $this->orderRepo->create([
            'product_id'     => $product->id,
            'quantity'       => $data['quantity'],
            'total'          => round($total, 2),
            'payment_method' => $data['payment_method'],
            'status'         => 'pending',
        ]);

        $this->productRepo->decrementStock($product->id, $data['quantity']);

        return $order;
    }
}
```

### ৪. Repository (Data Access Layer):

```php
// app/Repositories/OrderRepository.php

class OrderRepository
{
    public function create(array $data): Order
    {
        return Order::create($data);
    }

    public function findById(int $id): Order
    {
        return Order::findOrFail($id);
    }

    public function getByStatus(string $status)
    {
        return Order::where('status', $status)
            ->with('product')
            ->orderBy('created_at', 'desc')
            ->get();
    }
}
```

### ৫. Model & Migration (Database Layer):

```php
// app/Models/Order.php

class Order extends Model
{
    protected $fillable = [
        'product_id', 'quantity', 'total',
        'payment_method', 'status',
    ];

    public function product()
    {
        return $this->belongsTo(Product::class);
    }
}

// database/migrations/2024_01_01_create_orders_table.php

public function up(): void
{
    Schema::create('orders', function (Blueprint $table) {
        $table->id();
        $table->foreignId('product_id')->constrained();
        $table->integer('quantity');
        $table->decimal('total', 10, 2);
        $table->string('payment_method'); // bkash, nagad, cod
        $table->string('status')->default('pending');
        $table->timestamps();
    });
}
```

---

## 📌 JS (Express) সম্পূর্ণ উদাহরণ

Express-এ একই অর্ডার সিস্টেমের সম্পূর্ণ প্রবাহ:

### ফোল্ডার স্ট্রাকচার:

```
  src/
  ├── routes/
  │   └── orderRoutes.js             ← Presentation
  ├── controllers/
  │   └── orderController.js         ← Presentation
  ├── services/
  │   └── orderService.js            ← Business Logic
  ├── repositories/
  │   └── orderRepository.js         ← Data Access
  │   └── productRepository.js       ← Data Access
  └── database/
      └── knexfile.js                ← Database Config
      └── migrations/                ← Database
```

### ১. Route:

```javascript
// src/routes/orderRoutes.js

const express = require('express');
const router = express.Router();

module.exports = (orderController) => {
    router.post('/orders', (req, res) => orderController.store(req, res));
    router.get('/orders/:id', (req, res) => orderController.show(req, res));
    return router;
};
```

### ২. Controller (Presentation Layer):

```javascript
// src/controllers/orderController.js

class OrderController {
    constructor(orderService) {
        this.orderService = orderService;
    }

    async store(req, res) {
        try {
            const { productId, quantity, paymentMethod } = req.body;

            if (!productId || !quantity) {
                return res.status(400).json({
                    success: false,
                    message: 'productId এবং quantity আবশ্যক',
                });
            }

            const order = await this.orderService.placeOrder({
                productId,
                quantity,
                paymentMethod,
            });

            return res.status(201).json({
                success: true,
                message: 'অর্ডার সফলভাবে তৈরি হয়েছে!',
                data: order,
            });
        } catch (error) {
            return res.status(422).json({
                success: false,
                message: error.message,
            });
        }
    }
}

module.exports = OrderController;
```

### ৩. Service (Business Logic Layer):

```javascript
// src/services/orderService.js

class OrderService {
    constructor(orderRepo, productRepo) {
        this.orderRepo = orderRepo;
        this.productRepo = productRepo;
    }

    async placeOrder(data) {
        const product = await this.productRepo.findById(data.productId);

        if (product.stock < data.quantity) {
            throw new Error('পর্যাপ্ত স্টক নেই!');
        }

        let total = product.price * data.quantity;

        // bKash/Nagad পেমেন্ট হলে ২% ক্যাশব্যাক
        if (['bkash', 'nagad'].includes(data.paymentMethod)) {
            total *= 0.98;
        }

        const order = await this.orderRepo.create({
            product_id: data.productId,
            quantity: data.quantity,
            total: Math.round(total * 100) / 100,
            payment_method: data.paymentMethod,
            status: 'pending',
        });

        await this.productRepo.decrementStock(data.productId, data.quantity);

        return order;
    }
}

module.exports = OrderService;
```

### ৪. Repository (Data Access Layer):

```javascript
// src/repositories/orderRepository.js

class OrderRepository {
    constructor(db) {
        this.db = db;
    }

    async create(data) {
        const [id] = await this.db('orders').insert(data);
        return this.findById(id);
    }

    async findById(id) {
        const order = await this.db('orders').where({ id }).first();
        if (!order) throw new Error('অর্ডার খুঁজে পাওয়া যায়নি');
        return order;
    }
}

module.exports = OrderRepository;
```

---

## 🔗 অন্যান্য আর্কিটেকচারের সাথে তুলনা

```
  ┌────────────────┬──────────────┬───────────────┬────────────────┬──────────────┐
  │     বিষয়       │   Layered    │    Clean      │   Hexagonal    │ Microservice │
  ├────────────────┼──────────────┼───────────────┼────────────────┼──────────────┤
  │ জটিলতা         │    কম ⭐      │  মাঝারি ⭐⭐   │   মাঝারি ⭐⭐   │  বেশি ⭐⭐⭐   │
  │ শেখার সময়      │    কম        │  মাঝারি       │   বেশি         │  অনেক বেশি   │
  │ টেস্টযোগ্যতা    │    মাঝারি    │  খুব ভালো     │   খুব ভালো     │  ভালো        │
  │ নমনীয়তা        │    কম        │  বেশি         │   বেশি         │  খুব বেশি    │
  │ পারফরম্যান্স    │    ভালো      │  ভালো         │   ভালো         │  নেটওয়ার্ক    │
  │                │              │               │                │  ওভারহেড     │
  │ টিম সাইজ       │    ২-৫       │  ৩-১০         │   ৩-১০         │  ১০+         │
  │ সেরা ব্যবহার    │  CRUD, ছোট  │  জটিল ডোমেইন  │  Multiple UI   │  বড় সিস্টেম  │
  │                │  প্রজেক্ট     │               │  Adapters      │              │
  └────────────────┴──────────────┴───────────────┴────────────────┴──────────────┘
```

### Layered → Clean Architecture-এ মাইগ্রেশন পথ:

```
  ধাপ ১: Repository Pattern যোগ করুন
  ═══════════════════════════════════
  Controller → Service → DB
              পরিবর্তন হবে:
  Controller → Service → Repository → DB

  ধাপ ২: Interface/Contract ব্যবহার করুন
  ═══════════════════════════════════════
  Service সরাসরি Repository ক্লাস ব্যবহার না করে Interface-এ নির্ভর করবে

  ধাপ ৩: Dependency Inversion প্রয়োগ করুন
  ═══════════════════════════════════════
  Business Logic কোনো ফ্রেমওয়ার্ক বা DB-র উপর নির্ভর করবে না

  ধাপ ৪: Domain Model আলাদা করুন
  ═══════════════════════════════
  Eloquent Model ≠ Domain Entity → আলাদা করুন

  আপনি এখন Clean Architecture-এ আছেন! 🎉
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

- **সাধারণ ডোমেইন:** CRUD-ভিত্তিক অ্যাপ, সিম্পল বিজনেস লজিক
- **ছোট টিম:** ২-৫ জন ডেভেলপার, সবাই একই কোডবেজে কাজ করে
- **দ্রুত ডেভেলপমেন্ট দরকার:** MVP, প্রোটোটাইপ, স্টার্টআপ-এর প্রথম ভার্সন
- **নতুন টিম:** আর্কিটেকচার প্যাটার্নে অভিজ্ঞতা কম
- **ট্র্যাডিশনাল অ্যাপ:** ই-কমার্স, ব্লগ, CMS, অ্যাডমিন প্যানেল

### ❌ ব্যবহার করবেন না যখন:

- **জটিল বিজনেস লজিক:** bKash-এর মতো ফিনান্সিয়াল সিস্টেম যেখানে লজিক অনেক গভীর
- **একাধিক ইন্টারফেস:** Web + Mobile App + API সবগুলো আলাদা আলাদা ভাবে কাজ করে
- **হাই স্কেলেবিলিটি:** Pathao-এর মতো যেখানে লাখো রিকোয়েস্ট আসে প্রতি মিনিটে
- **ঘন ঘন পরিবর্তন:** বিজনেস রুল প্রতিদিন বদলায়

### সিদ্ধান্ত গাইড:

```
  আপনার প্রজেক্ট কি সাধারণ CRUD?
         │
         ├── হ্যাঁ ──► লেয়ার্ড আর্কিটেকচার ✅
         │
         └── না
              │
              ├── বিজনেস লজিক কি জটিল?
              │         │
              │         ├── হ্যাঁ ──► Clean / Hexagonal আর্কিটেকচার
              │         │
              │         └── না ──► লেয়ার্ড আর্কিটেকচার ✅
              │
              └── অনেক বড় সিস্টেম, অনেক টিম?
                        │
                        ├── হ্যাঁ ──► Microservices
                        │
                        └── না ──► Modular Monolith / Clean
```

---

## 📖 বাস্তব উদাহরণ

### ই-কমার্স প্রোডাক্ট ম্যানেজমেন্ট সিস্টেম (Daraz-স্টাইল)

ধরুন, আপনি একটি ছোট ই-কমার্স সাইট বানাচ্ছেন যেখানে পণ্য যোগ, সম্পাদনা, মুছে ফেলা ও
তালিকা দেখা যায়। এটি লেয়ার্ড আর্কিটেকচারের জন্য আদর্শ।

### সম্পূর্ণ ফোল্ডার স্ট্রাকচার (Laravel):

```
  daraz-lite/
  │
  ├── app/
  │   ├── Http/
  │   │   ├── Controllers/
  │   │   │   └── ProductController.php       📱 Presentation
  │   │   └── Requests/
  │   │       └── ProductRequest.php          📱 Presentation
  │   │
  │   ├── Services/
  │   │   └── ProductService.php              ⚙️  Business Logic
  │   │
  │   ├── Repositories/
  │   │   ├── Contracts/
  │   │   │   └── ProductRepositoryInterface.php
  │   │   └── Eloquent/
  │   │       └── ProductRepository.php       💾 Data Access
  │   │
  │   └── Models/
  │       └── Product.php                     🗄️  Database
  │
  ├── database/
  │   └── migrations/
  │       └── 2024_01_01_create_products.php  🗄️  Database
  │
  ├── routes/
  │   └── api.php                             📱 Presentation
  │
  └── tests/
      ├── Feature/
      │   └── ProductApiTest.php
      └── Unit/
          └── ProductServiceTest.php
```

### রিকোয়েস্ট ফ্লো ভিজ্যুয়ালাইজেশন:

```
  ইউজার: POST /api/products { name: "মোবাইল", price: 15000 }

  ┌─────────────────────────────────────────────────────┐
  │  📱 Route (api.php)                                 │
  │  POST /api/products → ProductController@store       │
  └─────────────────────────┬───────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────┐
  │  📱 ProductController                               │
  │  ─────────────────────────────                      │
  │  • রিকোয়েস্ট ভ্যালিডেশন (ProductRequest)            │
  │  • $this->productService->createProduct($data)      │
  │  • JSON রেসপন্স ফরম্যাট করে রিটার্ন                  │
  └─────────────────────────┬───────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────┐
  │  ⚙️  ProductService                                 │
  │  ─────────────────────────────                      │
  │  • নাম ডুপ্লিকেট কিনা চেক                           │
  │  • মূল্য সঠিক পরিসরে আছে কিনা চেক                  │
  │  • ক্যাটাগরি অনুযায়ী অতিরিক্ত নিয়ম প্রয়োগ          │
  │  • $this->productRepo->create($data)                │
  └─────────────────────────┬───────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────┐
  │  💾 ProductRepository                               │
  │  ─────────────────────────────                      │
  │  • Product::create($data)                           │
  │  • Eloquent ORM ব্যবহার করে DB-তে সংরক্ষণ           │
  └─────────────────────────┬───────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────┐
  │  🗄️  MySQL Database                                 │
  │  ─────────────────────────────                      │
  │  • INSERT INTO products (name, price, ...)          │
  │  • ডাটা সংরক্ষিত হলো! ✅                             │
  └─────────────────────────────────────────────────────┘
```

### সম্পূর্ণ ফোল্ডার স্ট্রাকচার (Express):

```
  daraz-lite-express/
  │
  ├── src/
  │   ├── routes/
  │   │   └── productRoutes.js           📱 Presentation
  │   ├── controllers/
  │   │   └── productController.js       📱 Presentation
  │   ├── services/
  │   │   └── productService.js          ⚙️  Business Logic
  │   ├── repositories/
  │   │   └── productRepository.js       💾 Data Access
  │   └── database/
  │       ├── knexfile.js                🗄️  Database Config
  │       └── migrations/               🗄️  Database
  │
  ├── tests/
  │   └── productService.test.js
  │
  ├── app.js
  └── package.json
```

---

## 📌 সারসংক্ষেপ (Summary)

```
  ╔══════════════════════════════════════════════════════════════╗
  ║               লেয়ার্ড আর্কিটেকচার সারসংক্ষেপ                ║
  ╠══════════════════════════════════════════════════════════════╣
  ║                                                            ║
  ║  🎯 মূল ধারণা:                                              ║
  ║     অ্যাপকে অনুভূমিক স্তরে ভাগ করা,                        ║
  ║     প্রতিটি স্তরের নির্দিষ্ট দায়িত্ব আছে                    ║
  ║                                                            ║
  ║  📊 লেয়ারসমূহ:                                              ║
  ║     Presentation → Business → Data Access → Database       ║
  ║                                                            ║
  ║  ✅ সেরা ব্যবহার:                                           ║
  ║     CRUD অ্যাপ, ছোট-মাঝারি প্রজেক্ট, নতুন টিম              ║
  ║                                                            ║
  ║  ❌ এড়িয়ে চলুন:                                             ║
  ║     জটিল ডোমেইন, হাই স্কেলেবিলিটি, একাধিক UI              ║
  ║                                                            ║
  ║  🇧🇩 বাংলাদেশ কনটেক্সট:                                    ║
  ║     Grameenphone-এর অ্যাডমিন প্যানেল, ছোট ই-কমার্স,       ║
  ║     সরকারি পোর্টাল — এসবে লেয়ার্ড আর্কিটেকচার যথেষ্ট      ║
  ║                                                            ║
  ║  🔄 পরবর্তী ধাপ:                                            ║
  ║     প্রজেক্ট বড় হলে Clean Architecture-এ মাইগ্রেট করুন     ║
  ║                                                            ║
  ╚══════════════════════════════════════════════════════════════╝
```

---

> **💡 টিপ:** আপনি যদি এখনো কোনো আর্কিটেকচার প্যাটার্ন না শিখে থাকেন,
> তাহলে লেয়ার্ড আর্কিটেকচার দিয়ে শুরু করুন। এটি শেখার পরে Clean Architecture
> বা Hexagonal Architecture বুঝতে অনেক সহজ হবে।

---

_এই ডকুমেন্ট **dev-pathshala** প্রজেক্টের অংশ — বাংলায় সফটওয়্যার ইঞ্জিনিয়ারিং শেখার একটি উন্মুক্ত রিসোর্স।_
