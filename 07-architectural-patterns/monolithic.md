# 🏢 মনোলিথিক আর্কিটেকচার (Monolithic Architecture)

## 📌 সংজ্ঞা ও মূল ধারণা

**Monolithic Architecture** হলো এমন একটি সফটওয়্যার ডিজাইন প্যাটার্ন যেখানে পুরো অ্যাপ্লিকেশনটি একটি **একক, অবিভাজ্য ইউনিট** হিসেবে তৈরি, ডিপ্লয় এবং স্কেল করা হয়। সমস্ত business logic, data access layer, UI layer — সবকিছু একটি single codebase-এ থাকে এবং একটি single process হিসেবে রান করে।

> 💡 Monolith মানে এই না যে কোডটি খারাপ বা unstructured। একটি well-designed monolith হতে পারে highly maintainable এবং performant।

### মনোলিথের প্রকারভেদ

| প্রকার | বিবরণ | উদাহরণ |
|--------|--------|--------|
| **Single-Process Monolith** | সবকিছু একটি process-এ চলে, কোনো module boundary নেই | ছোট Laravel/Express অ্যাপ |
| **Modular Monolith** | একটি process, কিন্তু ভেতরে well-defined module boundaries আছে | Shopify (Ruby), Laravel Modules |
| **Distributed Monolith** | দেখতে microservice, কিন্তু tightly coupled — সবচেয়ে খারাপ পরিস্থিতি ⚠️ | Poorly designed "microservices" |

### ইতিহাস ও বিবর্তন

মনোলিথিক আর্কিটেকচার সফটওয়্যার ইঞ্জিনিয়ারিং-এর **সবচেয়ে পুরনো এবং প্রমাণিত** প্যাটার্ন। ১৯৬০-৭০ দশকের mainframe যুগ থেকে শুরু করে আজকের আধুনিক web application পর্যন্ত — monolith সর্বত্র বিদ্যমান।

```
Timeline:
1960-70s → Mainframe monoliths (COBOL, Fortran)
1990s   → Client-Server monoliths (VB, Delphi)
2000s   → Web monoliths (PHP, Java EE, Rails, Django)
2010s   → Microservices hype — "Monolith bad!"
2020s   → "Modular Monolith" renaissance — প্রাজ্ঞ পছন্দ
```

আজকের দিনে Amazon, Shopify, Basecamp-এর মতো কোম্পানি প্রমাণ করেছে যে **well-structured monolith** অনেক সময় microservices-এর চেয়ে বেশি কার্যকর।

---

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন আপনি **ঢাকার একটি বড় শপিং মল** কল্পনা করছেন:

```
🏢 মনোলিথ = একটি বিশাল শপিং মল
├── Ground Floor: Reception (Presentation Layer / API)
├── 1st Floor: Office Departments (Business Logic)
│   ├── HR Department
│   ├── Sales Department
│   └── Accounting Department
├── 2nd Floor: Storage (Data Access Layer)
└── Basement: Database (Database)

সবকিছু এক ভবনে — যোগাযোগ সহজ, কিন্তু ভবন বড় হলে সমস্যা!

🏘️ মাইক্রোসার্ভিস = বিশ্ববিদ্যালয় ক্যাম্পাস
├── Building A: Arts Faculty (User Service)
├── Building B: Science Faculty (Product Service)
├── Building C: Business Faculty (Order Service)
└── Building D: Library (Payment Service)

প্রতিটি ভবন আলাদা — স্বাধীন, কিন্তু ভবনগুলোর মধ্যে যোগাযোগে সময় লাগে।
```

**বাংলাদেশি startup context:** আপনি "BD Shop" নামে একটি e-commerce platform বানাচ্ছেন। শুরুতে ৩-৫ জনের টিম, তাড়াতাড়ি MVP launch করতে হবে। এই পরিস্থিতিতে monolith হলো সবচেয়ে **বুদ্ধিমান সিদ্ধান্ত** — কারণ দ্রুত feature deliver করা যায়, debugging সহজ, এবং infrastructure cost কম।

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### ১. Layered Monolith

```
┌──────────────────────────────────────────────────────┐
│                    CLIENT (Browser/App)                │
└──────────────────────┬───────────────────────────────┘
                       │ HTTP Request
┌──────────────────────▼───────────────────────────────┐
│              🌐 PRESENTATION LAYER                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│  │  Controllers │ │ Middleware  │ │  Views/API   │    │
│  └──────┬──────┘ └─────────────┘ └─────────────┘    │
│─────────┼────────────────────────────────────────────│
│         ▼      📋 BUSINESS LOGIC LAYER               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│  │  Services   │ │  Validators │ │   Events     │    │
│  └──────┬──────┘ └─────────────┘ └─────────────┘    │
│─────────┼────────────────────────────────────────────│
│         ▼      💾 DATA ACCESS LAYER                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│  │Repositories │ │   Models    │ │    Cache     │    │
│  └──────┬──────┘ └─────────────┘ └─────────────┘    │
└─────────┼────────────────────────────────────────────┘
          │
┌─────────▼────────────────────────────────────────────┐
│              🗄️ DATABASE (MySQL/PostgreSQL)           │
└──────────────────────────────────────────────────────┘
```

### ২. Modular Monolith

```
┌──────────────────────────────────────────────────────────────┐
│                    SINGLE DEPLOYMENT UNIT                      │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  👤 User     │  │  📦 Product  │  │  🛒 Order    │        │
│  │   Module     │  │   Module     │  │   Module     │        │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │        │
│  │ │Controller│ │  │ │Controller│ │  │ │Controller│ │        │
│  │ │Service   │ │  │ │Service   │ │  │ │Service   │ │        │
│  │ │Model     │ │  │ │Model     │ │  │ │Model     │ │        │
│  │ │Migration │ │  │ │Migration │ │  │ │Migration │ │        │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
│         │    Event Bus / Contracts    │      │                │
│         └──────────────┼──────────────┘      │                │
│                        │                                      │
│  ┌─────────────────────▼──────────────────────────────┐      │
│  │          🔗 Shared Kernel (Events, Contracts)       │      │
│  └─────────────────────┬──────────────────────────────┘      │
└────────────────────────┼──────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   🗄️ Single DB      │
              │  (Schema per module) │
              └─────────────────────┘
```

### ৩. Deployment Diagram

```
┌──────────────────────────────────────────────┐
│              LOAD BALANCER (Nginx)            │
│              (Round Robin / Least Conn)       │
└────────┬──────────┬──────────┬───────────────┘
         │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
    │Server 1│ │Server 2│ │Server 3│
    │ (App)  │ │ (App)  │ │ (App)  │
    └────┬───┘ └────┬───┘ └────┬───┘
         │          │          │
    ┌────▼──────────▼──────────▼───┐
    │     Redis (Session/Cache)     │
    └──────────────┬───────────────┘
                   │
    ┌──────────────▼───────────────┐
    │   MySQL Primary (Write)      │──── MySQL Replica (Read)
    └──────────────────────────────┘
```

---

## 💻 মনোলিথিক ডিজাইন

### Traditional Layered Monolith

একটি সম্পূর্ণ e-commerce অ্যাপ্লিকেশনের layered monolith structure দেখা যাক।

#### PHP (Laravel 11) — Project Structure

```
bd-shop/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── ProductController.php
│   │   │   ├── OrderController.php
│   │   │   └── UserController.php
│   │   ├── Middleware/
│   │   │   └── EnsureUserIsVerified.php
│   │   └── Requests/
│   │       ├── StoreProductRequest.php
│   │       └── PlaceOrderRequest.php
│   ├── Services/
│   │   ├── ProductService.php
│   │   ├── OrderService.php
│   │   └── PaymentService.php
│   ├── Repositories/
│   │   ├── Contracts/
│   │   │   ├── ProductRepositoryInterface.php
│   │   │   └── OrderRepositoryInterface.php
│   │   ├── ProductRepository.php
│   │   └── OrderRepository.php
│   └── Models/
│       ├── Product.php
│       ├── Order.php
│       └── User.php
├── routes/
│   ├── api.php
│   └── web.php
└── database/
    └── migrations/
```

#### PHP — Controller → Service → Repository → Model

```php
<?php
// app/Http/Controllers/ProductController.php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\StoreProductRequest;
use App\Services\ProductService;
use Illuminate\Http\JsonResponse;

final class ProductController extends Controller
{
    public function __construct(
        private readonly ProductService $productService,
    ) {}

    public function index(): JsonResponse
    {
        $products = $this->productService->getActiveProducts();

        return response()->json([
            'status' => 'success',
            'data' => $products,
        ]);
    }

    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = $this->productService->createProduct(
            $request->validated()
        );

        return response()->json([
            'status' => 'success',
            'data' => $product,
        ], 201);
    }

    public function show(int $id): JsonResponse
    {
        $product = $this->productService->findOrFail($id);

        return response()->json([
            'status' => 'success',
            'data' => $product,
        ]);
    }
}
```

```php
<?php
// app/Services/ProductService.php

declare(strict_types=1);

namespace App\Services;

use App\Repositories\Contracts\ProductRepositoryInterface;
use App\Models\Product;
use Illuminate\Support\Facades\Cache;
use Illuminate\Database\Eloquent\Collection;

final readonly class ProductService
{
    public function __construct(
        private ProductRepositoryInterface $productRepository,
    ) {}

    public function getActiveProducts(): Collection
    {
        return Cache::remember(
            key: 'products:active',
            ttl: now()->addMinutes(30),
            callback: fn () => $this->productRepository->getActive(),
        );
    }

    public function createProduct(array $data): Product
    {
        $product = $this->productRepository->create($data);
        Cache::forget('products:active');

        return $product;
    }

    public function findOrFail(int $id): Product
    {
        return Cache::remember(
            key: "product:{$id}",
            ttl: now()->addHour(),
            callback: fn () => $this->productRepository->findOrFail($id),
        );
    }
}
```

```php
<?php
// app/Repositories/Contracts/ProductRepositoryInterface.php

declare(strict_types=1);

namespace App\Repositories\Contracts;

use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

interface ProductRepositoryInterface
{
    public function getActive(): Collection;
    public function create(array $data): Product;
    public function findOrFail(int $id): Product;
}
```

```php
<?php
// app/Repositories/ProductRepository.php

declare(strict_types=1);

namespace App\Repositories;

use App\Models\Product;
use App\Repositories\Contracts\ProductRepositoryInterface;
use Illuminate\Database\Eloquent\Collection;

final readonly class ProductRepository implements ProductRepositoryInterface
{
    public function getActive(): Collection
    {
        return Product::query()
            ->where('is_active', true)
            ->with(['category', 'images'])
            ->orderByDesc('created_at')
            ->get();
    }

    public function create(array $data): Product
    {
        return Product::create($data);
    }

    public function findOrFail(int $id): Product
    {
        return Product::with(['category', 'images', 'reviews'])
            ->findOrFail($id);
    }
}
```

#### JavaScript (Node.js/Express) — Same Layered Pattern

```
bd-shop/
├── src/
│   ├── controllers/
│   │   ├── productController.js
│   │   ├── orderController.js
│   │   └── userController.js
│   ├── services/
│   │   ├── productService.js
│   │   ├── orderService.js
│   │   └── paymentService.js
│   ├── repositories/
│   │   ├── productRepository.js
│   │   └── orderRepository.js
│   ├── models/
│   │   ├── Product.js
│   │   ├── Order.js
│   │   └── User.js
│   ├── middleware/
│   │   └── auth.js
│   ├── routes/
│   │   ├── productRoutes.js
│   │   └── orderRoutes.js
│   └── app.js
├── package.json
└── .env
```

```javascript
// src/controllers/productController.js

export class ProductController {
    #productService;

    constructor(productService) {
        this.#productService = productService;
    }

    getAll = async (req, res, next) => {
        try {
            const products = await this.#productService.getActiveProducts();
            res.json({ status: 'success', data: products });
        } catch (error) {
            next(error);
        }
    };

    create = async (req, res, next) => {
        try {
            const product = await this.#productService.createProduct(req.body);
            res.status(201).json({ status: 'success', data: product });
        } catch (error) {
            next(error);
        }
    };

    getById = async (req, res, next) => {
        try {
            const product = await this.#productService.findOrFail(
                parseInt(req.params.id)
            );
            res.json({ status: 'success', data: product });
        } catch (error) {
            next(error);
        }
    };
}
```

```javascript
// src/services/productService.js

import { createClient } from 'redis';

export class ProductService {
    #productRepository;
    #cache;

    constructor(productRepository, cacheClient) {
        this.#productRepository = productRepository;
        this.#cache = cacheClient;
    }

    async getActiveProducts() {
        const cached = await this.#cache.get('products:active');
        if (cached) return JSON.parse(cached);

        const products = await this.#productRepository.getActive();
        await this.#cache.setEx('products:active', 1800, JSON.stringify(products));
        return products;
    }

    async createProduct(data) {
        const product = await this.#productRepository.create(data);
        await this.#cache.del('products:active');
        return product;
    }

    async findOrFail(id) {
        const cacheKey = `product:${id}`;
        const cached = await this.#cache.get(cacheKey);
        if (cached) return JSON.parse(cached);

        const product = await this.#productRepository.findOrFail(id);
        await this.#cache.setEx(cacheKey, 3600, JSON.stringify(product));
        return product;
    }
}
```

```javascript
// src/repositories/productRepository.js

import { Product } from '../models/Product.js';

export class ProductRepository {
    async getActive() {
        return Product.findAll({
            where: { isActive: true },
            include: ['category', 'images'],
            order: [['createdAt', 'DESC']],
        });
    }

    async create(data) {
        return Product.create(data);
    }

    async findOrFail(id) {
        const product = await Product.findByPk(id, {
            include: ['category', 'images', 'reviews'],
        });
        if (!product) throw new NotFoundError(`Product #${id} not found`);
        return product;
    }
}
```

```javascript
// src/routes/productRoutes.js

import { Router } from 'express';
import { ProductController } from '../controllers/productController.js';
import { ProductService } from '../services/productService.js';
import { ProductRepository } from '../repositories/productRepository.js';
import { authMiddleware } from '../middleware/auth.js';
import { cacheClient } from '../config/redis.js';

const router = Router();

// Manual DI — production-এ awilix বা tsyringe ব্যবহার করুন
const productRepository = new ProductRepository();
const productService = new ProductService(productRepository, cacheClient);
const controller = new ProductController(productService);

router.get('/products', controller.getAll);
router.get('/products/:id', controller.getById);
router.post('/products', authMiddleware, controller.create);

export default router;
```

---

### Modular Monolith (Advanced)

Modular Monolith হলো monolithic architecture-এর সবচেয়ে **পরিণত রূপ**। এখানে অ্যাপ্লিকেশন একটি single deployment unit, কিন্তু ভেতরে প্রতিটি domain একটি স্বতন্ত্র module হিসেবে organized। Module-গুলো শুধুমাত্র **contracts/interfaces** এর মাধ্যমে communicate করে।

#### PHP (Laravel) — Modular Monolith Structure

```
bd-shop/
├── modules/
│   ├── User/
│   │   ├── Contracts/
│   │   │   └── UserServiceInterface.php
│   │   ├── Controllers/
│   │   │   └── UserController.php
│   │   ├── Events/
│   │   │   └── UserRegistered.php
│   │   ├── Models/
│   │   │   └── User.php
│   │   ├── Services/
│   │   │   └── UserService.php
│   │   ├── Providers/
│   │   │   └── UserServiceProvider.php
│   │   ├── Routes/
│   │   │   └── api.php
│   │   └── Database/
│   │       └── Migrations/
│   ├── Product/
│   │   ├── Contracts/
│   │   ├── Controllers/
│   │   ├── Models/
│   │   ├── Services/
│   │   ├── Providers/
│   │   ├── Routes/
│   │   └── Database/
│   ├── Order/
│   │   └── ... (same structure)
│   └── Payment/
│       └── ... (same structure)
├── app/
│   └── Shared/
│       ├── Events/
│       ├── Contracts/
│       └── Kernel/
└── config/
    └── modules.php
```

```php
<?php
// modules/User/Contracts/UserServiceInterface.php
// অন্য module শুধু এই interface জানবে, implementation জানবে না

declare(strict_types=1);

namespace Modules\User\Contracts;

interface UserServiceInterface
{
    public function findById(int $id): ?UserDTO;
    public function verifyUser(int $id): bool;
}
```

```php
<?php
// modules/User/DTOs/UserDTO.php

declare(strict_types=1);

namespace Modules\User\DTOs;

final readonly class UserDTO
{
    public function __construct(
        public int $id,
        public string $name,
        public string $email,
        public bool $isVerified,
    ) {}

    public static function fromModel(\Modules\User\Models\User $user): self
    {
        return new self(
            id: $user->id,
            name: $user->name,
            email: $user->email,
            isVerified: $user->is_verified,
        );
    }
}
```

```php
<?php
// modules/User/Providers/UserServiceProvider.php

declare(strict_types=1);

namespace Modules\User\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\User\Contracts\UserServiceInterface;
use Modules\User\Services\UserService;

final class UserServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(
            UserServiceInterface::class,
            UserService::class,
        );
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/../Database/Migrations');
        $this->loadRoutesFrom(__DIR__ . '/../Routes/api.php');
    }
}
```

```php
<?php
// modules/Order/Services/OrderService.php
// Order module User module-কে Contract দিয়ে access করে — সরাসরি Model ব্যবহার করে না

declare(strict_types=1);

namespace Modules\Order\Services;

use Modules\Order\Models\Order;
use Modules\Order\Enums\OrderStatus;
use Modules\User\Contracts\UserServiceInterface;
use Modules\Order\Events\OrderPlaced;
use Illuminate\Support\Facades\DB;

final readonly class OrderService
{
    public function __construct(
        private UserServiceInterface $userService,
    ) {}

    public function placeOrder(int $userId, array $items): Order
    {
        $user = $this->userService->findById($userId)
            ?? throw new \DomainException('User not found');

        if (!$user->isVerified) {
            throw new \DomainException('Unverified users cannot place orders');
        }

        return DB::transaction(function () use ($userId, $items) {
            $order = Order::create([
                'user_id' => $userId,
                'status' => OrderStatus::Pending,
                'total' => $this->calculateTotal($items),
            ]);

            $order->items()->createMany($items);

            // Event দিয়ে অন্য module-কে জানানো হচ্ছে
            event(new OrderPlaced($order));

            return $order;
        });
    }

    private function calculateTotal(array $items): float
    {
        return array_reduce(
            $items,
            fn (float $total, array $item) => $total + ($item['price'] * $item['quantity']),
            0.0,
        );
    }
}
```

```php
<?php
// modules/Order/Enums/OrderStatus.php

declare(strict_types=1);

namespace Modules\Order\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Confirmed = 'confirmed';
    case Processing = 'processing';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';

    public function canTransitionTo(self $next): bool
    {
        return match ($this) {
            self::Pending    => in_array($next, [self::Confirmed, self::Cancelled]),
            self::Confirmed  => in_array($next, [self::Processing, self::Cancelled]),
            self::Processing => in_array($next, [self::Shipped, self::Cancelled]),
            self::Shipped    => in_array($next, [self::Delivered]),
            self::Delivered,
            self::Cancelled  => false,
        };
    }
}
```

#### JavaScript (Node.js/Express) — Modular Monolith

```
bd-shop/
├── modules/
│   ├── user/
│   │   ├── contracts/
│   │   │   └── userServiceInterface.js
│   │   ├── controllers/
│   │   │   └── userController.js
│   │   ├── models/
│   │   │   └── User.js
│   │   ├── services/
│   │   │   └── userService.js
│   │   ├── events/
│   │   │   └── userRegistered.js
│   │   └── routes.js
│   ├── product/
│   ├── order/
│   └── payment/
├── shared/
│   ├── eventBus.js
│   ├── container.js
│   └── errors.js
└── src/
    └── app.js
```

```javascript
// shared/eventBus.js — Module-গুলোর মধ্যে loose coupling-এর জন্য Event Bus

import { EventEmitter } from 'node:events';

class AppEventBus extends EventEmitter {
    #history = [];

    emitDomainEvent(eventName, payload) {
        this.#history.push({
            event: eventName,
            payload,
            timestamp: Date.now(),
        });
        this.emit(eventName, payload);
    }

    getHistory() {
        return structuredClone(this.#history);
    }
}

export const eventBus = new AppEventBus();
```

```javascript
// shared/container.js — Simple DI container

class Container {
    #bindings = new Map();
    #singletons = new Map();

    singleton(name, factory) {
        this.#bindings.set(name, { factory, singleton: true });
    }

    bind(name, factory) {
        this.#bindings.set(name, { factory, singleton: false });
    }

    resolve(name) {
        const binding = this.#bindings.get(name);
        if (!binding) throw new Error(`No binding for: ${name}`);

        if (binding.singleton) {
            if (!this.#singletons.has(name)) {
                this.#singletons.set(name, binding.factory(this));
            }
            return this.#singletons.get(name);
        }

        return binding.factory(this);
    }
}

export const container = new Container();
```

```javascript
// modules/order/services/orderService.js

import { eventBus } from '../../../shared/eventBus.js';

export class OrderService {
    #userService;
    #orderRepository;

    constructor(userService, orderRepository) {
        this.#userService = userService;
        this.#orderRepository = orderRepository;
    }

    async placeOrder(userId, items) {
        const user = await this.#userService.findById(userId);
        if (!user) throw new DomainError('User not found');
        if (!user.isVerified) {
            throw new DomainError('Unverified users cannot place orders');
        }

        const total = items.reduce(
            (sum, item) => sum + item.price * item.quantity, 0
        );

        const order = await this.#orderRepository.createWithItems({
            userId,
            status: 'pending',
            total,
            items,
        });

        eventBus.emitDomainEvent('order.placed', {
            orderId: order.id,
            userId,
            total,
        });

        return order;
    }
}
```

```javascript
// modules/payment/services/paymentListener.js
// Payment module Order module-এর event শুনছে — কোনো direct dependency নেই

import { eventBus } from '../../../shared/eventBus.js';

export function registerPaymentListeners(paymentService) {
    eventBus.on('order.placed', async ({ orderId, userId, total }) => {
        try {
            await paymentService.initiatePayment(orderId, userId, total);
        } catch (error) {
            console.error(`Payment initiation failed for order ${orderId}:`, error);
            eventBus.emitDomainEvent('payment.failed', { orderId, error: error.message });
        }
    });
}
```

---

## 🔥 Complex Scenarios

### ১. Scaling a Monolith

অনেকে মনে করেন monolith scale করা যায় না — এটা **সম্পূর্ণ ভুল ধারণা**। বাংলাদেশের বড় e-commerce বা government portal-গুলো monolith architecture-এ সফলভাবে scale করে চলছে।

#### Vertical Scaling

```
আগে:                    পরে:
┌──────────┐            ┌──────────────┐
│  2 CPU   │   ────►    │   16 CPU     │
│  4GB RAM │            │   64GB RAM   │
│  50GB SSD│            │   500GB NVMe │
└──────────┘            └──────────────┘
```

#### Horizontal Scaling with Load Balancer

**PHP (Laravel) — Session Management with Redis:**

```php
<?php
// config/session.php — Redis-based session for horizontal scaling

return [
    'driver' => env('SESSION_DRIVER', 'redis'),
    'lifetime' => env('SESSION_LIFETIME', 120),
    'connection' => env('SESSION_CONNECTION', 'session'),
];

// config/database.php — Redis configuration
'redis' => [
    'session' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_SESSION_DB', 1),
    ],
    'cache' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_CACHE_DB', 2),
    ],
],
```

**PHP — Caching Strategy:**

```php
<?php
// app/Services/ProductService.php — Multi-layer caching

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Cache;

final readonly class CachedProductService
{
    public function __construct(
        private ProductService $inner,
    ) {}

    public function getPopularProducts(int $limit = 20): array
    {
        // L1: Application cache (30 sec) → L2: Redis (5 min) → DB
        return Cache::store('array')->remember(
            "popular_products:{$limit}",
            30,
            fn () => Cache::store('redis')->remember(
                "popular_products:{$limit}",
                300,
                fn () => $this->inner->getPopularProducts($limit),
            ),
        );
    }
}
```

**JavaScript — Redis Session + Cluster:**

```javascript
// src/config/scaling.js

import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';

// Redis-based session — সব server instance একই session share করবে
export function configureSession(app) {
    const redisClient = createClient({
        url: process.env.REDIS_URL,
    });
    redisClient.connect();

    app.use(session({
        store: new RedisStore({ client: redisClient }),
        secret: process.env.SESSION_SECRET,
        resave: false,
        saveUninitialized: false,
        cookie: { secure: true, maxAge: 86400000 },
    }));
}

// Cluster mode — CPU core অনুযায়ী multiple process চালানো
export function startWithCluster(app, port) {
    if (cluster.isPrimary) {
        const numCPUs = availableParallelism();
        console.log(`Primary ${process.pid} starting ${numCPUs} workers`);

        for (let i = 0; i < numCPUs; i++) {
            cluster.fork();
        }

        cluster.on('exit', (worker) => {
            console.log(`Worker ${worker.process.pid} died, restarting...`);
            cluster.fork();
        });
    } else {
        app.listen(port, () => {
            console.log(`Worker ${process.pid} listening on port ${port}`);
        });
    }
}
```

### ২. Database Design in Monolith

Monolith-এ সাধারণত একটি single database থাকে, কিন্তু **schema organization** খুবই গুরুত্বপূর্ণ।

```sql
-- Schema Organization by Domain (PostgreSQL)
-- প্রতিটি module-এর জন্য আলাদা schema — data isolation নিশ্চিত করে

CREATE SCHEMA user_mgmt;
CREATE SCHEMA product_catalog;
CREATE SCHEMA order_mgmt;
CREATE SCHEMA payment;

-- User Module Tables
CREATE TABLE user_mgmt.users (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    phone       VARCHAR(20),
    district    VARCHAR(100),           -- বাংলাদেশের জেলা
    is_verified BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON user_mgmt.users(email);
CREATE INDEX idx_users_district ON user_mgmt.users(district);

-- Product Module Tables
CREATE TABLE product_catalog.products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(500) NOT NULL,
    slug        VARCHAR(500) UNIQUE NOT NULL,
    price       DECIMAL(12,2) NOT NULL,
    stock       INTEGER NOT NULL DEFAULT 0,
    category_id BIGINT REFERENCES product_catalog.categories(id),
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_active ON product_catalog.products(is_active)
    WHERE is_active = TRUE;
CREATE INDEX idx_products_category ON product_catalog.products(category_id);

-- Order Module — cross-schema reference
CREATE TABLE order_mgmt.orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL,       -- user_mgmt.users(id) — FK optional
    status     VARCHAR(20) NOT NULL DEFAULT 'pending',
    total      DECIMAL(14,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_user ON order_mgmt.orders(user_id);
CREATE INDEX idx_orders_status ON order_mgmt.orders(status);
```

**PHP — Laravel Migration:**

```php
<?php
// modules/Product/Database/Migrations/2024_01_01_create_products_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name', 500);
            $table->string('slug', 500)->unique();
            $table->decimal('price', 12, 2);
            $table->unsignedInteger('stock')->default(0);
            $table->foreignId('category_id')
                  ->constrained()
                  ->cascadeOnDelete();
            $table->boolean('is_active')->default(true)->index();
            $table->timestamps();

            $table->index(['is_active', 'created_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

**JavaScript — Sequelize Migration:**

```javascript
// migrations/20240101-create-products.js

export async function up(queryInterface, Sequelize) {
    await queryInterface.createTable('products', {
        id: {
            type: Sequelize.BIGINT,
            autoIncrement: true,
            primaryKey: true,
        },
        name: {
            type: Sequelize.STRING(500),
            allowNull: false,
        },
        slug: {
            type: Sequelize.STRING(500),
            allowNull: false,
            unique: true,
        },
        price: {
            type: Sequelize.DECIMAL(12, 2),
            allowNull: false,
        },
        stock: {
            type: Sequelize.INTEGER.UNSIGNED,
            defaultValue: 0,
        },
        categoryId: {
            type: Sequelize.BIGINT,
            references: { model: 'categories', key: 'id' },
            onDelete: 'CASCADE',
        },
        isActive: {
            type: Sequelize.BOOLEAN,
            defaultValue: true,
        },
        createdAt: Sequelize.DATE,
        updatedAt: Sequelize.DATE,
    });

    await queryInterface.addIndex('products', ['isActive', 'createdAt']);
}

export async function down(queryInterface) {
    await queryInterface.dropTable('products');
}
```

### ৩. Modular Monolith — সম্পূর্ণ E-commerce উদাহরণ

এখানে দেখানো হচ্ছে কিভাবে BD Shop-এর চারটি module (User, Product, Order, Payment) পরস্পরের সাথে **contracts ও events** দিয়ে communicate করে।

```
Module Communication Flow:
                                                         
  [User Module]                    [Product Module]       
       │                                │                 
       │ UserServiceInterface           │ ProductServiceInterface
       │                                │                 
       ▼                                ▼                 
  [Order Module] ──── OrderPlaced Event ───► [Payment Module]
       │                                          │       
       │◄────── PaymentCompleted Event ───────────┘       
```

**PHP — Complete Module Communication:**

```php
<?php
// modules/Order/Events/OrderPlaced.php

declare(strict_types=1);

namespace Modules\Order\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

final class OrderPlaced
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public readonly int $orderId,
        public readonly int $userId,
        public readonly float $total,
        public readonly string $currency = 'BDT',
    ) {}
}
```

```php
<?php
// modules/Payment/Listeners/InitiatePaymentOnOrderPlaced.php

declare(strict_types=1);

namespace Modules\Payment\Listeners;

use Modules\Order\Events\OrderPlaced;
use Modules\Payment\Services\PaymentService;
use Modules\Payment\Enums\PaymentGateway;

final readonly class InitiatePaymentOnOrderPlaced
{
    public function __construct(
        private PaymentService $paymentService,
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $this->paymentService->initiate(
            orderId: $event->orderId,
            amount: $event->total,
            currency: $event->currency,
            gateway: PaymentGateway::BKash,
        );
    }
}
```

```php
<?php
// modules/Payment/Enums/PaymentGateway.php

declare(strict_types=1);

namespace Modules\Payment\Enums;

enum PaymentGateway: string
{
    case BKash = 'bkash';
    case Nagad = 'nagad';
    case Rocket = 'rocket';
    case SSLCommerz = 'sslcommerz';
    case Stripe = 'stripe';

    public function getServiceClass(): string
    {
        return match ($this) {
            self::BKash      => \Modules\Payment\Gateways\BKashGateway::class,
            self::Nagad      => \Modules\Payment\Gateways\NagadGateway::class,
            self::Rocket     => \Modules\Payment\Gateways\RocketGateway::class,
            self::SSLCommerz => \Modules\Payment\Gateways\SSLCommerzGateway::class,
            self::Stripe     => \Modules\Payment\Gateways\StripeGateway::class,
        };
    }
}
```

**JavaScript — Module Communication via Events:**

```javascript
// modules/order/events/handlers.js

import { eventBus } from '../../../shared/eventBus.js';

export function registerOrderEventHandlers(container) {
    // Stock কমানো — Product module-এর কাজ, কিন্তু Order event থেকে trigger হবে
    eventBus.on('order.placed', async ({ orderId, items }) => {
        const productService = container.resolve('productService');
        for (const item of items) {
            await productService.decrementStock(item.productId, item.quantity);
        }
    });

    // Payment সফল হলে order status update
    eventBus.on('payment.completed', async ({ orderId }) => {
        const orderService = container.resolve('orderService');
        await orderService.updateStatus(orderId, 'confirmed');
    });

    // Payment fail হলে order cancel + stock restore
    eventBus.on('payment.failed', async ({ orderId }) => {
        const orderService = container.resolve('orderService');
        const order = await orderService.findById(orderId);

        await orderService.updateStatus(orderId, 'cancelled');

        const productService = container.resolve('productService');
        for (const item of order.items) {
            await productService.incrementStock(item.productId, item.quantity);
        }
    });
}
```

### ৪. Background Jobs in Monolith

Production-grade monolith-এ background jobs অপরিহার্য। Email পাঠানো, report generation, data cleanup — সবই queue-তে যায়।

**PHP — Laravel Queue System:**

```php
<?php
// app/Jobs/SendOrderConfirmationEmail.php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Modules\Order\Models\Order;
use Modules\User\Contracts\UserServiceInterface;

final class SendOrderConfirmationEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60; // ৬০ সেকেন্ড পর retry

    public function __construct(
        private readonly int $orderId,
    ) {}

    public function middleware(): array
    {
        return [
            new RateLimited('emails'),
            new WithoutOverlapping($this->orderId),
        ];
    }

    public function handle(UserServiceInterface $userService): void
    {
        $order = Order::with('items')->findOrFail($this->orderId);
        $user = $userService->findById($order->user_id);

        // Email পাঠানো হচ্ছে...
        \Mail::to($user->email)->send(
            new \App\Mail\OrderConfirmation($order)
        );
    }

    public function failed(\Throwable $exception): void
    {
        \Log::error("Order confirmation email failed", [
            'order_id' => $this->orderId,
            'error' => $exception->getMessage(),
        ]);
    }
}

// Dispatch:
// SendOrderConfirmationEmail::dispatch($order->id)->onQueue('emails');
```

```php
<?php
// app/Console/Kernel.php — Scheduled Tasks

declare(strict_types=1);

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

final class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        // প্রতিদিন রাত ২টায় পুরনো cart পরিষ্কার
        $schedule->command('cart:cleanup --days=7')
            ->dailyAt('02:00')
            ->withoutOverlapping();

        // প্রতি ঘণ্টায় জনপ্রিয় পণ্যের cache refresh
        $schedule->command('cache:refresh-popular-products')
            ->hourly();

        // প্রতি সোমবার সকালে weekly report generate
        $schedule->command('report:weekly-sales')
            ->weeklyOn(1, '08:00')
            ->emailOutputTo('admin@bdshop.com');
    }
}
```

**JavaScript — BullMQ Queue:**

```javascript
// src/queues/emailQueue.js

import { Queue, Worker } from 'bullmq';
import { sendEmail } from '../services/emailService.js';

const connection = { host: 'localhost', port: 6379 };

export const emailQueue = new Queue('emails', {
    connection,
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 5000 },
        removeOnComplete: { count: 1000 },
        removeOnFail: { count: 5000 },
    },
});

// Worker — আলাদা process-এ চালানো উচিত
const worker = new Worker('emails', async (job) => {
    const { type, data } = job.data;

    switch (type) {
        case 'order-confirmation':
            await sendEmail({
                to: data.email,
                subject: `BD Shop — Order #${data.orderId} Confirmed!`,
                template: 'order-confirmation',
                context: data,
            });
            break;

        case 'password-reset':
            await sendEmail({
                to: data.email,
                subject: 'BD Shop — পাসওয়ার্ড রিসেট',
                template: 'password-reset',
                context: data,
            });
            break;
    }
}, {
    connection,
    concurrency: 5,
    limiter: { max: 50, duration: 60_000 }, // প্রতি মিনিটে সর্বোচ্চ ৫০টি email
});

worker.on('failed', (job, err) => {
    console.error(`Email job ${job.id} failed:`, err.message);
});

// Scheduled/Cron Jobs
import cron from 'node-cron';

cron.schedule('0 2 * * *', async () => {
    console.log('Running cart cleanup...');
    await cartService.cleanupAbandoned(7);
});

cron.schedule('0 * * * *', async () => {
    console.log('Refreshing popular products cache...');
    await productService.refreshPopularCache();
});
```

### ৫. Monolith to Microservices Migration — Strangler Fig Pattern

বাংলাদেশের একটি বড় e-commerce কোম্পানি তাদের monolith থেকে microservices-এ migrate করতে চাইছে। **Strangler Fig Pattern** হলো সবচেয়ে নিরাপদ পদ্ধতি:

```
Step 1: Monolith-এর সামনে API Gateway বসানো
┌──────────────┐      ┌──────────────────┐
│  API Gateway │─────►│    Monolith      │
│  (Nginx)     │      │  (All features)  │
└──────────────┘      └──────────────────┘

Step 2: একটি module বের করা (যেমন: Payment)
┌──────────────┐      ┌──────────────────┐
│  API Gateway │─────►│    Monolith      │
│              │      │  (User, Product, │
│              │      │   Order)         │
│              │      └──────────────────┘
│              │      ┌──────────────────┐
│              │─────►│ Payment Service  │ ◄── নতুন microservice
└──────────────┘      └──────────────────┘

Step 3: আরো module বের করা
┌──────────────┐      ┌──────────────────┐
│  API Gateway │─────►│ Monolith (Core)  │
│              │      └──────────────────┘
│              │─────►│ Payment Service  │
│              │─────►│ Notification Svc │
│              │─────►│ Search Service   │
└──────────────┘      └──────────────────┘
```

**Migration-এর ক্রম (কোনটা আগে বের করবেন):**

| ক্রম | Module | কারণ |
|------|--------|------|
| ১ | Notification/Email | সবচেয়ে কম coupling, স্বাধীনভাবে scale দরকার |
| ২ | Search/Elasticsearch | আলাদা data store, read-heavy |
| ৩ | Payment | Third-party integration heavy, security isolation |
| ৪ | User/Auth | অনেক module নির্ভর করে, তাই সবশেষে |

**PHP — Feature Flag During Migration:**

```php
<?php
// app/Services/PaymentRouter.php — Migration চলাকালীন feature flag

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Http;

final readonly class PaymentRouter
{
    public function __construct(
        private PaymentService $legacyPayment,  // পুরনো monolith service
    ) {}

    public function processPayment(int $orderId, float $amount): PaymentResult
    {
        // Feature flag check — ধীরে ধীরে নতুন service-এ traffic পাঠানো
        if ($this->shouldUseNewService($orderId)) {
            return $this->callNewPaymentService($orderId, $amount);
        }

        return $this->legacyPayment->process($orderId, $amount);
    }

    private function shouldUseNewService(int $orderId): bool
    {
        // প্রথমে ১০% traffic, তারপর ধীরে ধীরে বাড়ানো
        $rolloutPercentage = (int) config('features.new_payment_service', 0);
        return ($orderId % 100) < $rolloutPercentage;
    }

    private function callNewPaymentService(int $orderId, float $amount): PaymentResult
    {
        $response = Http::timeout(5)
            ->retry(3, 100)
            ->post(config('services.payment.url') . '/api/payments', [
                'order_id' => $orderId,
                'amount' => $amount,
                'currency' => 'BDT',
            ]);

        return PaymentResult::fromResponse($response->json());
    }
}
```

**JavaScript — Strangler Fig Implementation:**

```javascript
// src/middleware/featureRouter.js

export function createFeatureRouter(config) {
    const httpClient = config.httpClient;

    return {
        async routePayment(orderId, amount) {
            const rollout = parseInt(process.env.NEW_PAYMENT_ROLLOUT ?? '0');
            const useNewService = (orderId % 100) < rollout;

            if (useNewService) {
                try {
                    const response = await httpClient.post(
                        `${process.env.PAYMENT_SERVICE_URL}/api/payments`,
                        { orderId, amount, currency: 'BDT' },
                        { timeout: 5000 }
                    );
                    return response.data;
                } catch (error) {
                    // Fallback — নতুন service fail হলে পুরনোটায় ফেরত
                    console.warn('New payment service failed, falling back:', error.message);
                    return config.legacyPaymentService.process(orderId, amount);
                }
            }

            return config.legacyPaymentService.process(orderId, amount);
        },
    };
}
```

---

## ✅ সুবিধা (Pros) ও ❌ অসুবিধা (Cons)

| বিষয় | ✅ সুবিধা | ❌ অসুবিধা |
|--------|-----------|------------|
| **Development** | সহজ শুরু, single codebase, IDE support ভালো | বড় হলে compile/build time বাড়ে |
| **Deployment** | একবারে পুরো অ্যাপ deploy — সহজ CI/CD | ছোট পরিবর্তনেও পুরো অ্যাপ redeploy |
| **Testing** | Integration test সহজ, end-to-end test straightforward | Test suite ধীর হয়ে যায় |
| **Debugging** | Single process — stack trace সম্পূর্ণ দেখা যায় | বড় codebase-এ issue trace কঠিন |
| **Performance** | In-process call — কোনো network latency নেই | একটি component slow হলে পুরো অ্যাপ slow |
| **Scaling** | Vertical scaling সহজ | Feature-ভিত্তিক আলাদা scaling অসম্ভব |
| **Team** | ছোট টিমের জন্য আদর্শ (২-১৫ জন) | বড় টিমে code conflict, merge hell |
| **Technology** | একটি tech stack — সবাই একই language জানে | নতুন technology adopt করা কঠিন |
| **Data** | Single database — ACID transaction সহজ | Database bottleneck হতে পারে |
| **Cost** | কম infrastructure cost, কম DevOps প্রয়োজন | Large scale-এ resource waste |
| **Reliability** | সরল architecture — কম failure point | Single point of failure — crash হলে সব বন্ধ |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Big Ball of Mud (সবচেয়ে বিপজ্জনক)

কোনো structure বা layer boundary ছাড়া কোড লেখা:

```php
// ❌ খারাপ — Controller-এ সবকিছু: DB query, business logic, validation
class ProductController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate(['name' => 'required']);
        $slug = Str::slug($validated['name']);
        $exists = DB::table('products')->where('slug', $slug)->exists();
        if ($exists) {
            $slug .= '-' . rand(100, 999);
        }
        $id = DB::table('products')->insertGetId([
            'name' => $validated['name'],
            'slug' => $slug,
            'price' => $request->price,
        ]);
        DB::table('audit_logs')->insert([
            'action' => 'product_created',
            'entity_id' => $id,
        ]);
        Mail::to('admin@site.com')->send(new ProductCreated($id));
        return response()->json(['id' => $id]);
    }
}
```

```php
// ✅ ভালো — Responsibility আলাদা, testable, maintainable
final class ProductController extends Controller
{
    public function __construct(
        private readonly ProductService $productService,
    ) {}

    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = $this->productService->create(
            ProductDTO::fromRequest($request)
        );

        return response()->json($product, 201);
    }
}
```

### ২. Circular Dependencies

```
❌ খারাপ:
User Module ──imports──► Order Module
Order Module ──imports──► User Module    // Circular!

✅ ভালো:
User Module ──implements──► UserServiceInterface
Order Module ──depends on──► UserServiceInterface (Contract)
```

### ৩. God Class

```javascript
// ❌ খারাপ — একটি class সবকিছু করছে (৫০০+ লাইন)
class AppService {
    createUser() { /* ... */ }
    deleteUser() { /* ... */ }
    createProduct() { /* ... */ }
    placeOrder() { /* ... */ }
    processPayment() { /* ... */ }
    sendEmail() { /* ... */ }
    generateReport() { /* ... */ }
    // ... আরো ৫০টি method
}
```

```javascript
// ✅ ভালো — Single Responsibility
class UserService { /* শুধু user সংক্রান্ত কাজ */ }
class ProductService { /* শুধু product সংক্রান্ত কাজ */ }
class OrderService { /* শুধু order সংক্রান্ত কাজ */ }
```

### ৪. No Module Boundaries — Shared Mutable State

```php
// ❌ খারাপ — Order module সরাসরি User model access করছে
namespace Modules\Order\Services;

use Modules\User\Models\User;  // সরাসরি অন্য module-এর internal ব্যবহার!

class OrderService {
    public function placeOrder(int $userId): void {
        $user = User::find($userId);
        $user->order_count++;        // অন্য module-এর data modify!
        $user->save();
    }
}
```

```php
// ✅ ভালো — Contract দিয়ে communication
namespace Modules\Order\Services;

use Modules\User\Contracts\UserServiceInterface;  // শুধু contract

final readonly class OrderService {
    public function __construct(
        private UserServiceInterface $userService,
    ) {}

    public function placeOrder(int $userId): void {
        $user = $this->userService->findById($userId);
        // Order-এর কাজ এখানে, User update-এর জন্য event fire করো
        event(new OrderPlaced($userId));
    }
}
```

---

## 🧪 টেস্টিং মনোলিথ

### PHP — PHPUnit Examples

```php
<?php
// tests/Unit/Services/ProductServiceTest.php

declare(strict_types=1);

namespace Tests\Unit\Services;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use App\Services\ProductService;
use App\Repositories\Contracts\ProductRepositoryInterface;

final class ProductServiceTest extends TestCase
{
    private ProductService $service;
    private ProductRepositoryInterface $repository;

    protected function setUp(): void
    {
        $this->repository = $this->createMock(ProductRepositoryInterface::class);
        $this->service = new ProductService($this->repository);
    }

    #[Test]
    public function it_creates_product_and_clears_cache(): void
    {
        $data = ['name' => 'Jamdani Saree', 'price' => 5000.00];

        $this->repository
            ->expects($this->once())
            ->method('create')
            ->with($data);

        $this->service->createProduct($data);
    }

    #[Test]
    #[DataProvider('invalidPriceProvider')]
    public function it_rejects_invalid_prices(float $price): void
    {
        $this->expectException(\InvalidArgumentException::class);

        $this->service->createProduct([
            'name' => 'Test Product',
            'price' => $price,
        ]);
    }

    public static function invalidPriceProvider(): array
    {
        return [
            'negative' => [-100.0],
            'zero' => [0.0],
            'too high' => [10_000_001.0],
        ];
    }
}
```

```php
<?php
// tests/Feature/ProductApiTest.php — Integration/Feature Test

declare(strict_types=1);

namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;
use PHPUnit\Framework\Attributes\Test;
use App\Models\Product;
use App\Models\User;

final class ProductApiTest extends TestCase
{
    use RefreshDatabase;

    #[Test]
    public function authenticated_user_can_create_product(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/products', [
                'name' => 'Nakshi Kantha',
                'price' => 2500.00,
                'stock' => 50,
                'category_id' => 1,
            ]);

        $response->assertStatus(201)
            ->assertJsonStructure([
                'status',
                'data' => ['id', 'name', 'slug', 'price'],
            ]);

        $this->assertDatabaseHas('products', [
            'name' => 'Nakshi Kantha',
            'slug' => 'nakshi-kantha',
        ]);
    }

    #[Test]
    public function guest_cannot_create_product(): void
    {
        $this->postJson('/api/products', ['name' => 'Test'])
            ->assertStatus(401);
    }
}
```

### JavaScript — Jest Examples

```javascript
// tests/unit/services/productService.test.js

import { describe, it, expect, jest, beforeEach } from '@jest/globals';
import { ProductService } from '../../../src/services/productService.js';

describe('ProductService', () => {
    let productService;
    let mockRepository;
    let mockCache;

    beforeEach(() => {
        mockRepository = {
            getActive: jest.fn(),
            create: jest.fn(),
            findOrFail: jest.fn(),
        };
        mockCache = {
            get: jest.fn().mockResolvedValue(null),
            setEx: jest.fn().mockResolvedValue('OK'),
            del: jest.fn().mockResolvedValue(1),
        };
        productService = new ProductService(mockRepository, mockCache);
    });

    describe('getActiveProducts', () => {
        it('should return cached products when available', async () => {
            const cachedProducts = [{ id: 1, name: 'Hilsa Fish' }];
            mockCache.get.mockResolvedValue(JSON.stringify(cachedProducts));

            const result = await productService.getActiveProducts();

            expect(result).toEqual(cachedProducts);
            expect(mockRepository.getActive).not.toHaveBeenCalled();
        });

        it('should query DB and cache when cache miss', async () => {
            const products = [{ id: 1, name: 'Muslin Fabric' }];
            mockRepository.getActive.mockResolvedValue(products);

            const result = await productService.getActiveProducts();

            expect(result).toEqual(products);
            expect(mockRepository.getActive).toHaveBeenCalledTimes(1);
            expect(mockCache.setEx).toHaveBeenCalledWith(
                'products:active',
                1800,
                JSON.stringify(products)
            );
        });
    });

    describe('createProduct', () => {
        it('should create product and invalidate cache', async () => {
            const data = { name: 'Tangail Saree', price: 3000 };
            const created = { id: 1, ...data };
            mockRepository.create.mockResolvedValue(created);

            const result = await productService.createProduct(data);

            expect(result).toEqual(created);
            expect(mockCache.del).toHaveBeenCalledWith('products:active');
        });
    });
});
```

```javascript
// tests/integration/api/products.test.js — Supertest Integration Test

import { describe, it, expect, beforeAll, afterAll } from '@jest/globals';
import request from 'supertest';
import { createApp } from '../../../src/app.js';
import { sequelize } from '../../../src/config/database.js';

describe('Product API', () => {
    let app;
    let authToken;

    beforeAll(async () => {
        app = await createApp();
        await sequelize.sync({ force: true });

        // Test user তৈরি ও login
        const res = await request(app)
            .post('/api/auth/register')
            .send({ name: 'Test', email: 'test@bd.com', password: 'secret123' });
        authToken = res.body.token;
    });

    afterAll(async () => {
        await sequelize.close();
    });

    it('POST /api/products — should create product', async () => {
        const res = await request(app)
            .post('/api/products')
            .set('Authorization', `Bearer ${authToken}`)
            .send({
                name: 'Jamdani Saree',
                price: 5000,
                stock: 25,
            });

        expect(res.status).toBe(201);
        expect(res.body.data).toMatchObject({
            name: 'Jamdani Saree',
            price: 5000,
        });
    });

    it('GET /api/products — should list active products', async () => {
        const res = await request(app).get('/api/products');

        expect(res.status).toBe(200);
        expect(Array.isArray(res.body.data)).toBe(true);
    });

    it('POST /api/products — should reject unauthenticated', async () => {
        const res = await request(app)
            .post('/api/products')
            .send({ name: 'Test' });

        expect(res.status).toBe(401);
    });
});
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ মনোলিথ বেছে নিন যখন:

| পরিস্থিতি | কারণ |
|-----------|------|
| **Startup/MVP** | দ্রুত launch, কম infrastructure cost |
| **ছোট টিম (২-১৫ জন)** | সবাই একই codebase-এ কাজ করতে পারে |
| **Domain পরিষ্কার না** | Monolith-এ ভুল boundary ঠিক করা সহজ |
| **Simple business logic** | Microservices-এর complexity অপ্রয়োজনীয় |
| **সরকারি প্রকল্প** | Compliance সহজ, deployment straightforward |
| **Tight deadline** | Feature delivery দ্রুত হয় |
| **Budget সীমিত** | একটি server-এ সব চলে, DevOps cost কম |

### ❌ মনোলিথ এড়িয়ে চলুন যখন:

| পরিস্থিতি | কারণ |
|-----------|------|
| **বিশাল টিম (৫০+ engineer)** | Code conflict, slow CI/CD |
| **Feature-ভিত্তিক independent scaling দরকার** | যেমন: search service-এ ১০x বেশি traffic |
| **বিভিন্ন technology stack দরকার** | ML module Python-এ, API Node.js-এ |
| **খুব high availability দরকার** | Single point of failure গ্রহণযোগ্য না |
| **দ্রুত independent deployment দরকার** | প্রতিটি টিম আলাদাভাবে deploy করতে চায় |

### বাংলাদেশ Context — বাস্তব সিদ্ধান্ত:

- **ঢাকার একটি startup** — Monolith দিয়ে শুরু করুন। MVP ৩ মাসে বের করুন। ১ লক্ষ user হলে optimize করুন।
- **সরকারি portal (Land registry / NID)** — Modular Monolith সবচেয়ে ভালো। Security audit সহজ, deployment predictable।
- **Pathao/Foodpanda-level scale** — শুরু Monolith-এ, পরে Strangler Fig দিয়ে critical service আলাদা করুন।

---

## 🆚 মনোলিথ vs মাইক্রোসার্ভিস

| মানদণ্ড | 🏢 মনোলিথ | 🏘️ মাইক্রোসার্ভিস |
|---------|-----------|-------------------|
| **Codebase** | Single repository | Multiple repositories / Monorepo |
| **Deployment** | একবারে পুরো অ্যাপ | প্রতিটি service আলাদাভাবে |
| **Scaling** | পুরো অ্যাপ scale হয় | নির্দিষ্ট service scale করা যায় |
| **Technology** | একটি tech stack | Polyglot — service অনুযায়ী technology |
| **Data** | Single shared database | Database per service |
| **Communication** | In-process function call | HTTP/gRPC/Message Queue (network call) |
| **Latency** | কম (in-process) | বেশি (network hop) |
| **Transaction** | ACID transaction সহজ | Saga pattern দরকার — জটিল |
| **Debugging** | সহজ — single process, full stack trace | কঠিন — distributed tracing দরকার |
| **Team Structure** | Feature teams, shared codebase | Service teams, independent ownership |
| **DevOps** | সহজ — ১টি server, ১টি CI/CD pipeline | জটিল — Kubernetes, service mesh, monitoring |
| **Initial Cost** | কম | বেশি (infrastructure + tooling) |
| **Failure Impact** | পুরো অ্যাপ down | শুধু affected service down (graceful degradation) |
| **Development Speed (শুরুতে)** | দ্রুত | ধীর (boilerplate, infrastructure setup) |
| **Development Speed (বড় হলে)** | ধীর (merge conflict, long CI) | দ্রুত (independent teams) |

### Decision Framework:

```
শুরু করুন এই প্রশ্ন দিয়ে:

১. টিম কত বড়? 
   └─ ≤ 15 জন → Monolith ✅
   └─ > 50 জন → Microservices বিবেচনা করুন

২. Domain কি পরিষ্কার?
   └─ না → Monolith দিয়ে শুরু করুন, পরে extract করুন
   └─ হ্যাঁ → Modular Monolith বা Microservices

৩. Budget কেমন?
   └─ সীমিত → Monolith ✅
   └─ পর্যাপ্ত → যেটা domain demand করে

৪. Timeline কেমন?
   └─ Tight (< 6 মাস) → Monolith ✅
   └─ Flexible → Requirements অনুযায়ী

৫. Scale কেমন হবে?
   └─ Uniform → Monolith ভালো কাজ করে
   └─ Feature-specific spikes → Microservices সুবিধাজনক
```

---

## 📋 সারসংক্ষেপ

### মূল বিষয়গুলো মনে রাখুন:

1. **Monolith মানে "খারাপ" না** — এটি একটি legitimate architectural choice যা বিশ্বের অনেক সফল কোম্পানি ব্যবহার করছে।

2. **Modular Monolith** হলো monolith-এর সবচেয়ে পরিণত রূপ — module boundaries, contracts, এবং event-driven communication দিয়ে microservices-এর অনেক সুবিধা পাওয়া যায়।

3. **"Monolith First" strategy** — Martin Fowler-এর পরামর্শ অনুযায়ী, প্রায় সবসময় monolith দিয়ে শুরু করা উচিত এবং প্রয়োজনে microservices-এ migrate করা উচিত।

4. **Strangler Fig Pattern** — monolith থেকে microservices-এ যাওয়ার সবচেয়ে নিরাপদ পথ। একবারে সব পরিবর্তন না করে ধীরে ধীরে module extract করুন।

5. **Anti-patterns এড়িয়ে চলুন** — Big Ball of Mud, God Class, Circular Dependencies — এগুলো monolith-কে unmaintainable করে দেয়।

6. **Scaling সম্ভব** — Horizontal scaling, Redis caching, database read replicas, এবং background job queues দিয়ে monolith-কে বিশাল scale-এ চালানো সম্ভব।

> 🇧🇩 **বাংলাদেশের developer-দের জন্য:** আমাদের দেশের startup ecosystem-এ monolith হলো সবচেয়ে practical choice। দ্রুত MVP launch করুন, user feedback নিন, এবং যখন সত্যিই দরকার হবে তখন microservices-এ migrate করুন। Premature optimization হলো সফটওয়্যারের সবচেয়ে বড় শত্রু।

---

*"Start with a monolith, and extract microservices only when the pain of the monolith exceeds the pain of distribution."* — Sam Newman

*শেষ আপডেট: ২০২৫*
