# 🏛️ ক্লিন আর্কিটেকচার (Clean Architecture)

## 📌 সংজ্ঞা ও ইতিহাস

**Clean Architecture** হলো ২০১২ সালে **Robert C. Martin** (যিনি "Uncle Bob" নামে সুপরিচিত) কর্তৃক প্রস্তাবিত একটি সফটওয়্যার আর্কিটেকচারাল প্যাটার্ন। তিনি তাঁর বিখ্যাত ব্লগ পোস্ট "The Clean Architecture" এবং পরবর্তীতে ২০১৭ সালে প্রকাশিত *"Clean Architecture: A Craftsman's Guide to Software Structure and Design"* বইতে এই ধারণাটি বিস্তারিত উপস্থাপন করেন।

Uncle Bob আগের কয়েকটি আর্কিটেকচারাল ধারণা — **Hexagonal Architecture** (Alistair Cockburn), **Onion Architecture** (Jeffrey Palermo), **Screaming Architecture**, এবং **DCI (Data, Context, Interaction)** — থেকে অনুপ্রাণিত হয়ে Clean Architecture তৈরি করেন। এদের সবার মূল বক্তব্য একই: **ব্যবসায়িক নিয়ম (business rules) কে বাহ্যিক প্রযুক্তি থেকে সম্পূর্ণ আলাদা রাখো।**

### মূল লক্ষ্য (Core Goals)

| লক্ষ্য | বাংলায় ব্যাখ্যা | English |
|--------|-----------------|---------|
| **Framework Independence** | আর্কিটেকচার কোনো নির্দিষ্ট ফ্রেমওয়ার্কের উপর নির্ভরশীল নয় | Not tied to any framework |
| **Testability** | বাহ্যিক উপাদান ছাড়াই business rules পরীক্ষা করা যায় | Business rules testable without externals |
| **UI Independence** | UI পরিবর্তন করলে বাকি সিস্টেমে প্রভাব পড়ে না | UI can change without affecting rest |
| **Database Independence** | MySQL থেকে MongoDB-তে যেতে business logic পরিবর্তন লাগে না | Swap DB without changing logic |
| **External Agency Independence** | বাইরের কোনো সার্ভিসের উপর core logic নির্ভর করে না | No dependency on external services |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 🏢 অফিস বিল্ডিং অ্যানালজি (বাংলাদেশ প্রসঙ্গে)

কল্পনা করুন **বসুন্ধরা সিটি** শপিং মলের কথা:

```
🏢 বসুন্ধরা সিটি = আপনার সফটওয়্যার সিস্টেম

┌──────────────────────────────────────────────────────────────┐
│  🅿️ পার্কিং, সিকিউরিটি গেট, এসকেলেটর  (Frameworks/Drivers) │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  🏪 দোকানদার, ক্যাশিয়ার, ডেলিভারি  (Interface Adapters) │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │  📋 কেনাবেচার নিয়ম, রিটার্ন পলিসি  (Use Cases)     │ │ │
│  │  │  ┌──────────────────────────────────────────────────┐ │ │ │
│  │  │  │  💎 পণ্য, মূল্য, মালিকানা  (Entities)           │ │ │ │
│  │  │  └──────────────────────────────────────────────────┘ │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

- **Entities (পণ্য, মূল্য)**: পণ্যের দাম ও মালিকানা — এগুলো মল ভাঙলেও সত্য থাকে
- **Use Cases (কেনাবেচার নিয়ম)**: "৩ দিনের মধ্যে রিটার্ন করা যাবে" — নিয়ম মল-ভিত্তিক
- **Interface Adapters (দোকানদার)**: গ্রাহকের সাথে কথা বলে, ভেতরের নিয়ম অনুসরণ করে
- **Frameworks (পার্কিং, গেট)**: বদলানো যায়, মূল ব্যবসায় কোনো প্রভাব পড়ে না

---

## 📊 বিখ্যাত বৃত্তাকার ডায়াগ্রাম (The Concentric Circles)

Uncle Bob-এর Clean Architecture-এর সবচেয়ে পরিচিত চিত্রটি হলো কেন্দ্রমুখী বৃত্ত:

```
    ╔═══════════════════════════════════════════════════════════════════════╗
    ║                   FRAMEWORKS & DRIVERS (বাহ্যিক স্তর)                ║
    ║     Web Framework │ Database │ UI │ External APIs │ Devices          ║
    ║   ┌─────────────────────────────────────────────────────────────┐    ║
    ║   │              INTERFACE ADAPTERS (অ্যাডাপ্টার স্তর)           │    ║
    ║   │     Controllers │ Gateways │ Presenters │ Repositories      │    ║
    ║   │   ┌─────────────────────────────────────────────────────┐   │    ║
    ║   │   │            USE CASES (ইউজ কেস স্তর)                 │   │    ║
    ║   │   │     Application-specific business rules              │   │    ║
    ║   │   │   ┌─────────────────────────────────────────────┐   │   │    ║
    ║   │   │   │          ENTITIES (এন্টিটি স্তর)             │   │   │    ║
    ║   │   │   │   Enterprise-wide business rules              │   │   │    ║
    ║   │   │   │                                               │   │   │    ║
    ║   │   │   │         💎 সবচেয়ে গুরুত্বপূর্ণ                │   │   │    ║
    ║   │   │   │         কোনো নির্ভরতা নেই                    │   │   │    ║
    ║   │   │   └─────────────────────────────────────────────┘   │   │    ║
    ║   │   └─────────────────────────────────────────────────────┘   │    ║
    ║   └─────────────────────────────────────────────────────────────┘    ║
    ╚═══════════════════════════════════════════════════════════════════════╝

    ════════════════════════════════════════════════════════════════════
    ডিপেন্ডেন্সির দিক (DEPENDENCY DIRECTION):
    
         বাইরে ──────────────────────▶ ভেতরে
         Frameworks ──▶ Adapters ──▶ Use Cases ──▶ Entities
    ════════════════════════════════════════════════════════════════════
```

**মূলনীতি**: বাইরের স্তর ভেতরের স্তরকে জানতে পারে, কিন্তু ভেতরের স্তর কখনোই বাইরের স্তর সম্পর্কে জানবে না।

---

## 📖 ডিপেন্ডেন্সি রুল (The Dependency Rule)

Clean Architecture-এর **সবচেয়ে গুরুত্বপূর্ণ নিয়ম** হলো Dependency Rule:

> **ভেতরের স্তর কখনো বাইরের স্তর সম্পর্কে কিছু জানবে না। সোর্স কোডের dependency সবসময় ভেতরের দিকে নির্দেশ করবে।**

### ডিপেন্ডেন্সি দিকনির্দেশনা

```
    ❌ ভুল দিক (Entity বাইরের দিকে তাকাচ্ছে):
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ Entity   │─────▶│ Use Case │─────▶│Framework │
    │(ভেতরে)   │      │(মাঝে)    │      │(বাইরে)   │
    └──────────┘      └──────────┘      └──────────┘
         সোর্স কোড dependency বাইরের দিকে → নিষিদ্ধ! 🚫


    ✅ সঠিক দিক (বাইরে থেকে ভেতরের দিকে):
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │Framework │─────▶│ Use Case │─────▶│ Entity   │
    │(বাইরে)   │      │(মাঝে)    │      │(ভেতরে)   │
    └──────────┘      └──────────┘      └──────────┘
         সোর্স কোড dependency ভেতরের দিকে → সঠিক! ✅
```

### ❌ Bad: Entity তে Framework ক্লাস ইম্পোর্ট করা (PHP)

```php
<?php
// ❌ ভুল! Entity তে Laravel-এর Eloquent Model ব্যবহার করা হচ্ছে
// Entity জানছে যে Laravel ব্যবহার হচ্ছে — Dependency Rule ভঙ্গ!

namespace App\Domain\Entities;

use Illuminate\Database\Eloquent\Model; // 🚫 Framework dependency!

class Order extends Model  // 🚫 Eloquent-এর উপর নির্ভর!
{
    protected $table = 'orders';

    public function calculateTotal(): float
    {
        return $this->items->sum('price');
    }
}
```

### ✅ Good: Entity শুধু নিজের ব্যবসায়িক নিয়ম জানে (PHP)

```php
<?php
// ✅ সঠিক! Entity কোনো framework জানে না — শুধু business logic

namespace Domain\Entities;

class Order
{
    private string $id;
    private array $items;
    private string $status;

    public function __construct(string $id, array $items)
    {
        $this->id = $id;
        $this->items = $items;
        $this->status = 'pending';
    }

    public function calculateTotal(): float
    {
        return array_reduce($this->items, function (float $total, OrderItem $item) {
            return $total + $item->getSubtotal();
        }, 0.0);
    }

    public function confirm(): void
    {
        if (empty($this->items)) {
            throw new \DomainException('খালি অর্ডার confirm করা যায় না');
        }
        $this->status = 'confirmed';
    }
}
```

### ❌ Bad: Use Case তে Express/HTTP dependency (JavaScript)

```javascript
// ❌ ভুল! Use Case সরাসরি Express-এর req, res ব্যবহার করছে
class PlaceOrderUseCase {
    async execute(req, res) {  // 🚫 HTTP concept ভেতরে ঢুকেছে!
        const items = req.body.items;
        const order = new Order(items);
        await order.save();  // 🚫 Database concept!
        res.status(201).json(order);  // 🚫 HTTP response!
    }
}
```

### ✅ Good: Use Case শুধু ব্যবসায়িক নিয়ম জানে (JavaScript)

```javascript
// ✅ সঠিক! Use Case শুধু input নেয়, business logic চালায়, output দেয়
class PlaceOrderUseCase {
    constructor(orderRepository, paymentGateway) {
        this.orderRepository = orderRepository;  // Interface-এর উপর নির্ভর
        this.paymentGateway = paymentGateway;
    }

    async execute(inputDto) {
        const order = new Order(inputDto.customerId, inputDto.items);
        order.confirm();

        await this.paymentGateway.charge(order.calculateTotal());
        await this.orderRepository.save(order);

        return { orderId: order.id, total: order.calculateTotal(), status: order.status };
    }
}
```

---

## 🎯 চারটি স্তর বিস্তারিত (The Four Layers)

```
    স্তর ১ (সবচেয়ে ভেতরে)     স্তর ২                 স্তর ৩                 স্তর ৪ (সবচেয়ে বাইরে)
    ┌─────────────┐      ┌─────────────┐      ┌─────────────────┐    ┌─────────────────────┐
    │  ENTITIES   │◀─────│  USE CASES  │◀─────│    INTERFACE    │◀───│   FRAMEWORKS &      │
    │             │      │             │      │    ADAPTERS     │    │   DRIVERS           │
    │ • Order     │      │ • PlaceOrder│      │ • Controllers   │    │ • Laravel / Express │
    │ • Customer  │      │ • CancelOrder│     │ • Repositories  │    │ • MySQL / MongoDB   │
    │ • Money     │      │ • SendNotif │      │ • Presenters    │    │ • bKash API         │
    └─────────────┘      └─────────────┘      └─────────────────┘    └─────────────────────┘
     নির্ভরতা: ০          নির্ভরতা: Entity     নির্ভরতা: Use Case    নির্ভরতা: Adapter
```

---

### 💎 স্তর ১: Entities (এন্টিটি স্তর) — সবচেয়ে ভেতরে

Entity হলো **enterprise-wide business rules** এর ধারক। এগুলো সবচেয়ে সাধারণ এবং উচ্চ-স্তরের নিয়ম ধারণ করে। আপনার অ্যাপ্লিকেশন না থাকলেও এই নিয়মগুলো সত্য। উদাহরণস্বরূপ, "একটি অর্ডারের মোট মূল্য = প্রতিটি আইটেমের দাম × পরিমাণ এর যোগফল" — এই নিয়ম Daraz ব্যবহার করুক বা না করুক, সত্য থাকবে।

**PHP Entity উদাহরণ:**

```php
<?php

namespace Domain\Entities;

class Money
{
    private float $amount;
    private string $currency;

    public function __construct(float $amount, string $currency = 'BDT')
    {
        if ($amount < 0) {
            throw new \DomainException('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        }
        $this->amount = $amount;
        $this->currency = $currency;
    }

    public function add(Money $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('ভিন্ন মুদ্রা যোগ করা যায় না');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function getAmount(): float
    {
        return $this->amount;
    }
}

class OrderItem
{
    private string $productId;
    private string $productName;
    private Money $unitPrice;
    private int $quantity;

    public function __construct(string $productId, string $productName, Money $unitPrice, int $quantity)
    {
        if ($quantity <= 0) {
            throw new \DomainException('পরিমাণ শূন্যের বেশি হতে হবে');
        }
        $this->productId = $productId;
        $this->productName = $productName;
        $this->unitPrice = $unitPrice;
        $this->quantity = $quantity;
    }

    public function getSubtotal(): Money
    {
        return new Money($this->unitPrice->getAmount() * $this->quantity);
    }
}
```

**JavaScript Entity উদাহরণ:**

```javascript
// domain/entities/Money.js
class Money {
    #amount;
    #currency;

    constructor(amount, currency = 'BDT') {
        if (amount < 0) throw new Error('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        this.#amount = amount;
        this.#currency = currency;
    }

    add(other) {
        if (this.#currency !== other.#currency) {
            throw new Error('ভিন্ন মুদ্রা যোগ করা যায় না');
        }
        return new Money(this.#amount + other.#amount, this.#currency);
    }

    get amount() { return this.#amount; }
    get currency() { return this.#currency; }
}

// domain/entities/Order.js
class Order {
    #id;
    #customerId;
    #items;
    #status;
    #createdAt;

    constructor(id, customerId, items = []) {
        this.#id = id;
        this.#customerId = customerId;
        this.#items = items;
        this.#status = 'pending';
        this.#createdAt = new Date();
    }

    addItem(item) {
        this.#items.push(item);
    }

    calculateTotal() {
        return this.#items.reduce(
            (total, item) => total.add(item.getSubtotal()),
            new Money(0)
        );
    }

    confirm() {
        if (this.#items.length === 0) throw new Error('খালি অর্ডার confirm করা যায় না');
        this.#status = 'confirmed';
    }

    get id() { return this.#id; }
    get status() { return this.#status; }
}
```

---

### 📋 স্তর ২: Use Cases (ইউজ কেস স্তর)

Use Case স্তর **application-specific business rules** ধারণ করে। এটি Entity-গুলোকে সমন্বয় (orchestrate) করে নির্দিষ্ট ব্যবসায়িক কাজ সম্পাদন করে। Use Case জানে **কী করতে হবে**, কিন্তু জানে না **কোন database বা API ব্যবহার হচ্ছে**।

Use Case **Input Port** (ইনপুট পোর্ট) এবং **Output Port** (আউটপুট পোর্ট) ব্যবহার করে boundaries তৈরি করে:

```
    ┌────────────────────────────────────────────────────┐
    │                    Use Case                         │
    │                                                     │
    │   Input DTO ──▶ [ Business Logic ] ──▶ Output DTO  │
    │                       │    │                        │
    │              OrderRepo│    │PaymentGateway           │
    │             Interface │    │Interface                │
    │                  ▼         ▼                        │
    │          (কিভাবে সেভ হবে     (কিভাবে পেমেন্ট হবে    │
    │           সেটা জানে না)       সেটা জানে না)          │
    └────────────────────────────────────────────────────┘
```

**PHP Use Case উদাহরণ (Daraz-সদৃশ ই-কমার্স):**

```php
<?php

namespace Application\UseCases;

use Domain\Entities\Order;
use Domain\Entities\OrderItem;
use Domain\Entities\Money;

// Input DTO — Use Case তে কী ডেটা আসবে
class PlaceOrderInput
{
    public string $customerId;
    public array $items; // [{productId, name, price, qty}]
}

// Output DTO — Use Case কী ডেটা ফেরত দেবে
class PlaceOrderOutput
{
    public string $orderId;
    public float $totalAmount;
    public string $status;
}

// Output Port — Repository-র Interface (বাস্তবায়ন বাইরের স্তরে)
interface OrderRepositoryInterface
{
    public function save(Order $order): void;
    public function generateId(): string;
}

// Output Port — Payment Gateway-র Interface
interface PaymentGatewayInterface
{
    public function charge(string $customerId, float $amount): bool;
}

// Use Case বাস্তবায়ন
class PlaceOrderUseCase
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private PaymentGatewayInterface $paymentGateway
    ) {}

    public function execute(PlaceOrderInput $input): PlaceOrderOutput
    {
        $orderId = $this->orderRepository->generateId();

        $items = array_map(function ($itemData) {
            return new OrderItem(
                $itemData['productId'],
                $itemData['name'],
                new Money($itemData['price']),
                $itemData['qty']
            );
        }, $input->items);

        $order = new Order($orderId, $items);
        $order->confirm();

        $total = $order->calculateTotal();

        $paymentSuccess = $this->paymentGateway->charge(
            $input->customerId,
            $total->getAmount()
        );

        if (!$paymentSuccess) {
            throw new \RuntimeException('পেমেন্ট ব্যর্থ হয়েছে');
        }

        $this->orderRepository->save($order);

        $output = new PlaceOrderOutput();
        $output->orderId = $orderId;
        $output->totalAmount = $total->getAmount();
        $output->status = $order->getStatus();

        return $output;
    }
}
```

**JavaScript Use Case উদাহরণ:**

```javascript
// application/usecases/PlaceOrderUseCase.js
class PlaceOrderUseCase {
    constructor(orderRepository, paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }

    async execute({ customerId, items }) {
        const orderId = await this.orderRepository.generateId();

        const orderItems = items.map(
            (i) => new OrderItem(i.productId, i.name, new Money(i.price), i.qty)
        );

        const order = new Order(orderId, customerId, orderItems);
        order.confirm();

        const total = order.calculateTotal();
        const paid = await this.paymentGateway.charge(customerId, total.amount);

        if (!paid) throw new Error('পেমেন্ট ব্যর্থ হয়েছে');

        await this.orderRepository.save(order);

        return { orderId: order.id, total: total.amount, status: order.status };
    }
}
```

---

### 🔌 স্তর ৩: Interface Adapters (ইন্টারফেস অ্যাডাপ্টার স্তর)

এই স্তরে **Controllers**, **Presenters**, এবং **Gateways/Repositories** থাকে। এদের কাজ হলো বাইরের জগতের ডেটা ফরম্যাটকে Use Case-এর জন্য উপযোগী ফরম্যাটে রূপান্তর করা এবং Use Case-এর ফলাফলকে বাইরের জগতের জন্য রূপান্তর করা।

```
    ┌─────────────────────────────────────────────────────────┐
    │                  Interface Adapters                       │
    │                                                           │
    │   HTTP Request ──▶ [Controller] ──▶ Use Case Input DTO   │
    │                                                           │
    │   Use Case Output DTO ──▶ [Presenter] ──▶ HTTP Response  │
    │                                                           │
    │   Repository Interface ◀── [Repository Impl] ──▶ DB     │
    └─────────────────────────────────────────────────────────┘
```

**PHP Controller ও Repository বাস্তবায়ন:**

```php
<?php

// --- Controller (HTTP → Use Case) ---
namespace Adapters\Controllers;

use Application\UseCases\PlaceOrderUseCase;
use Application\UseCases\PlaceOrderInput;

class OrderController
{
    public function __construct(
        private PlaceOrderUseCase $placeOrderUseCase
    ) {}

    public function placeOrder(array $requestData): array
    {
        $input = new PlaceOrderInput();
        $input->customerId = $requestData['customer_id'];
        $input->items = $requestData['items'];

        $output = $this->placeOrderUseCase->execute($input);

        return [
            'success' => true,
            'data' => [
                'order_id' => $output->orderId,
                'total' => $output->totalAmount,
                'status' => $output->status,
            ],
        ];
    }
}

// --- Repository (Use Case Interface → Database) ---
namespace Adapters\Repositories;

use Domain\Entities\Order;
use Application\UseCases\OrderRepositoryInterface;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void
    {
        // Eloquent model ব্যবহার করে database-এ সেভ করে
        // কিন্তু Entity জানে না Eloquent সম্পর্কে!
        $model = new \App\Models\OrderModel();
        $model->id = $order->getId();
        $model->status = $order->getStatus();
        $model->total = $order->calculateTotal()->getAmount();
        $model->save();
    }

    public function generateId(): string
    {
        return 'ORD-' . uniqid();
    }
}
```

**JavaScript Controller ও Repository বাস্তবায়ন:**

```javascript
// adapters/controllers/OrderController.js
class OrderController {
    constructor(placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    async handlePlaceOrder(req, res) {
        try {
            const result = await this.placeOrderUseCase.execute({
                customerId: req.body.customer_id,
                items: req.body.items,
            });

            res.status(201).json({ success: true, data: result });
        } catch (error) {
            res.status(400).json({ success: false, error: error.message });
        }
    }
}

// adapters/repositories/MongoOrderRepository.js
class MongoOrderRepository {
    constructor(mongoClient) {
        this.collection = mongoClient.db('shop').collection('orders');
    }

    async save(order) {
        await this.collection.insertOne({
            _id: order.id,
            customerId: order.customerId,
            status: order.status,
            total: order.calculateTotal().amount,
            createdAt: new Date(),
        });
    }

    async generateId() {
        return 'ORD-' + Date.now().toString(36);
    }
}
```

---

### 🌐 স্তর ৪: Frameworks & Drivers (ফ্রেমওয়ার্ক ও ড্রাইভার স্তর)

এটি সবচেয়ে বাইরের স্তর। এখানে থাকে — web framework (Laravel, Express), database driver (MySQL, MongoDB), UI framework, external API clients (bKash, Pathao API), ইত্যাদি। এই স্তরে মূলত **glue code** লেখা হয় যা সব স্তরকে একত্রিত করে।

**PHP (Laravel) Integration:**

```php
<?php
// routes/api.php — Laravel Route (Frameworks স্তর)
use Illuminate\Support\Facades\Route;
use Adapters\Controllers\OrderController;

Route::post('/orders', function (Request $request) {
    // Dependency Injection — সব স্তর একত্রিত করা হচ্ছে
    $repository = new EloquentOrderRepository();
    $paymentGateway = new BkashPaymentGateway();
    $useCase = new PlaceOrderUseCase($repository, $paymentGateway);
    $controller = new OrderController($useCase);

    return response()->json(
        $controller->placeOrder($request->all())
    );
});
```

**JavaScript (Express) Integration:**

```javascript
// frameworks/express/server.js
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
app.use(express.json());

// সব স্তর একত্রিত করা (composition root)
const mongoClient = new MongoClient('mongodb://localhost:27017');
const orderRepo = new MongoOrderRepository(mongoClient);
const paymentGW = new BkashPaymentGateway(process.env.BKASH_API_KEY);
const placeOrderUC = new PlaceOrderUseCase(orderRepo, paymentGW);
const orderCtrl = new OrderController(placeOrderUC);

app.post('/api/orders', (req, res) => orderCtrl.handlePlaceOrder(req, res));

app.listen(3000, () => console.log('সার্ভার চালু: port 3000'));
```

---

## 🔗 তুলনা: Clean vs Hexagonal vs Onion Architecture

তিনটি আর্কিটেকচারেরই মূলনীতি একই — **dependency inversion** এবং **ব্যবসায়িক নিয়মকে কেন্দ্রে রাখা**। তবে তাদের উপস্থাপন ও পরিভাষায় পার্থক্য আছে।

### পাশাপাশি ASCII ডায়াগ্রাম

```
    CLEAN ARCHITECTURE          HEXAGONAL                    ONION
    (Robert C. Martin)          (Alistair Cockburn)          (Jeffrey Palermo)

    ┌───────────────┐           ┌───────────────┐            ┌───────────────┐
    │  Frameworks   │           │   Adapters    │            │Infrastructure │
    │ ┌───────────┐ │           │  ╱    ╲       │            │ ┌───────────┐ │
    │ │ Adapters  │ │           │ ╱ Ports ╲     │            │ │App Service│ │
    │ │┌─────────┐│ │           │╱         ╲    │            │ │┌─────────┐│ │
    │ ││Use Cases││ │           ││  Domain   │   │            │ ││ Domain  ││ │
    │ ││┌───────┐││ │           ││  Model    │   │            │ ││ Service ││ │
    │ │││Entity │││ │           ││           │   │            │ ││┌───────┐││ │
    │ ││└───────┘││ │           │╲           ╱   │            │ │││Domain │││ │
    │ │└─────────┘│ │           │ ╲  Ports  ╱   │            │ ││└Model──┘││ │
    │ └───────────┘ │           │  ╲       ╱    │            │ │└─────────┘│ │
    └───────────────┘           └───────────────┘            └───────────────┘
```

### তুলনামূলক টেবিল

| বৈশিষ্ট্য | Clean Architecture | Hexagonal Architecture | Onion Architecture |
|-----------|-------------------|----------------------|-------------------|
| **প্রবর্তক** | Robert C. Martin (2012) | Alistair Cockburn (2005) | Jeffrey Palermo (2008) |
| **মূল ধারণা** | ৪টি বৃত্তাকার স্তর | Ports & Adapters | স্তরবিশিষ্ট পেঁয়াজ |
| **স্তর সংখ্যা** | ৪টি নির্দিষ্ট | ৩টি (Domain, Ports, Adapters) | ৪টি (Domain Model, Domain Service, App Service, Infra) |
| **কেন্দ্রে** | Entities | Domain Model | Domain Model |
| **পোর্ট ধারণা** | Implicit (interface) | Explicit (Port/Adapter) | Implicit (interface) |
| **Use Case স্তর** | ✅ আলাদা স্তর | ❌ Domain-এর ভেতরে | ❌ Application Service |
| **ড্রাইভিং/ড্রিভেন** | Controller/Gateway | Primary/Secondary Adapter | ❌ নেই |
| **জনপ্রিয়তা** | সবচেয়ে বেশি আলোচিত | বেশ জনপ্রিয় | মাঝামাঝি |
| **ফ্রেমওয়ার্ক** | সব ভাষায় | Java/Kotlin-এ বেশি | .NET-এ বেশি |

### কোনটি কখন ব্যবহার করবেন?

- **Clean Architecture**: যখন ৪টি স্তরের স্পষ্ট বিভাজন দরকার এবং টিম বড়
- **Hexagonal**: যখন multiple external systems (API, DB, Queue) সাথে কাজ করতে হয়
- **Onion**: যখন .NET ব্যবহার করেন এবং DDD অনুসরণ করেন

**তিনটিরই মূলমন্ত্র এক**: ব্যবসায়িক নিয়ম কেন্দ্রে, প্রযুক্তি বাইরে। Dependency সবসময় ভেতরের দিকে।

---

## 📂 ডিরেক্টরি স্ট্রাকচার (Directory Structure)

### PHP প্রজেক্ট ডিরেক্টরি

```
📁 daraz-lite/
├── 📁 src/
│   ├── 📁 Domain/                          ← স্তর ১: Entities
│   │   ├── 📁 Entities/
│   │   │   ├── Order.php
│   │   │   ├── OrderItem.php
│   │   │   ├── Customer.php
│   │   │   └── Money.php
│   │   ├── 📁 ValueObjects/
│   │   │   ├── Address.php
│   │   │   └── PhoneNumber.php
│   │   └── 📁 Exceptions/
│   │       ├── InvalidOrderException.php
│   │       └── InsufficientBalanceException.php
│   │
│   ├── 📁 Application/                     ← স্তর ২: Use Cases
│   │   ├── 📁 UseCases/
│   │   │   ├── PlaceOrderUseCase.php
│   │   │   ├── CancelOrderUseCase.php
│   │   │   └── GetOrderStatusUseCase.php
│   │   ├── 📁 DTOs/
│   │   │   ├── PlaceOrderInput.php
│   │   │   └── PlaceOrderOutput.php
│   │   └── 📁 Interfaces/                  ← Output Ports
│   │       ├── OrderRepositoryInterface.php
│   │       ├── PaymentGatewayInterface.php
│   │       └── NotificationServiceInterface.php
│   │
│   ├── 📁 Adapters/                        ← স্তর ৩: Interface Adapters
│   │   ├── 📁 Controllers/
│   │   │   └── OrderController.php
│   │   ├── 📁 Presenters/
│   │   │   └── OrderJsonPresenter.php
│   │   └── 📁 Repositories/
│   │       ├── EloquentOrderRepository.php
│   │       └── InMemoryOrderRepository.php  ← টেস্টিং-এর জন্য
│   │
│   └── 📁 Frameworks/                      ← স্তর ৪: Frameworks
│       ├── 📁 Laravel/
│       │   ├── routes/
│       │   ├── config/
│       │   └── Providers/
│       └── 📁 Persistence/
│           ├── migrations/
│           └── Models/
│
└── 📁 tests/
    ├── 📁 Unit/
    │   ├── Domain/
    │   └── Application/
    └── 📁 Integration/
        └── Adapters/
```

### JavaScript/Node.js প্রজেক্ট ডিরেক্টরি

```
📁 daraz-lite-node/
├── 📁 src/
│   ├── 📁 domain/                          ← স্তর ১: Entities
│   │   ├── 📁 entities/
│   │   │   ├── Order.js
│   │   │   ├── OrderItem.js
│   │   │   ├── Customer.js
│   │   │   └── Money.js
│   │   ├── 📁 value-objects/
│   │   │   ├── Address.js
│   │   │   └── PhoneNumber.js
│   │   └── 📁 errors/
│   │       └── DomainError.js
│   │
│   ├── 📁 application/                     ← স্তর ২: Use Cases
│   │   ├── 📁 use-cases/
│   │   │   ├── PlaceOrderUseCase.js
│   │   │   ├── CancelOrderUseCase.js
│   │   │   └── GetOrderStatusUseCase.js
│   │   ├── 📁 dtos/
│   │   │   ├── PlaceOrderInput.js
│   │   │   └── PlaceOrderOutput.js
│   │   └── 📁 interfaces/                  ← Output Ports
│   │       ├── OrderRepository.js
│   │       ├── PaymentGateway.js
│   │       └── NotificationService.js
│   │
│   ├── 📁 adapters/                        ← স্তর ৩: Interface Adapters
│   │   ├── 📁 controllers/
│   │   │   └── OrderController.js
│   │   ├── 📁 presenters/
│   │   │   └── OrderJsonPresenter.js
│   │   └── 📁 repositories/
│   │       ├── MongoOrderRepository.js
│   │       └── InMemoryOrderRepository.js
│   │
│   └── 📁 frameworks/                      ← স্তর ৪: Frameworks
│       ├── 📁 express/
│       │   ├── server.js
│       │   ├── routes.js
│       │   └── middleware/
│       └── 📁 database/
│           ├── mongo-connection.js
│           └── migrations/
│
├── 📁 tests/
│   ├── 📁 unit/
│   └── 📁 integration/
└── package.json
```

---

## 🛒 সম্পূর্ণ বাস্তবায়ন উদাহরণ: Daraz-সদৃশ "Place Order"

### সম্পূর্ণ PHP বাস্তবায়ন

```php
<?php
// =====================================================
// স্তর ১: DOMAIN / ENTITIES
// =====================================================

// Domain/Entities/Money.php
namespace Domain\Entities;

class Money
{
    public function __construct(
        private readonly float $amount,
        private readonly string $currency = 'BDT'
    ) {
        if ($amount < 0) {
            throw new \DomainException('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        }
    }

    public function add(self $other): self
    {
        $this->ensureSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(int $factor): self
    {
        return new self($this->amount * $factor, $this->currency);
    }

    public function getAmount(): float { return $this->amount; }

    private function ensureSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('ভিন্ন মুদ্রা দিয়ে অপারেশন করা যায় না');
        }
    }
}

// Domain/Entities/OrderItem.php
namespace Domain\Entities;

class OrderItem
{
    public function __construct(
        private readonly string $productId,
        private readonly string $productName,
        private readonly Money $unitPrice,
        private readonly int $quantity
    ) {
        if ($quantity <= 0) {
            throw new \DomainException('পণ্যের পরিমাণ শূন্যের বেশি হতে হবে');
        }
    }

    public function getSubtotal(): Money
    {
        return $this->unitPrice->multiply($this->quantity);
    }

    public function getProductId(): string { return $this->productId; }
    public function getProductName(): string { return $this->productName; }
    public function getQuantity(): int { return $this->quantity; }
}

// Domain/Entities/Order.php
namespace Domain\Entities;

class Order
{
    private string $status = 'pending';
    private \DateTimeImmutable $createdAt;

    /** @param OrderItem[] $items */
    public function __construct(
        private readonly string $id,
        private readonly string $customerId,
        private array $items = []
    ) {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function addItem(OrderItem $item): void
    {
        if ($this->status !== 'pending') {
            throw new \DomainException('শুধুমাত্র pending অর্ডারে আইটেম যোগ করা যায়');
        }
        $this->items[] = $item;
    }

    public function calculateTotal(): Money
    {
        return array_reduce($this->items, function (Money $total, OrderItem $item) {
            return $total->add($item->getSubtotal());
        }, new Money(0));
    }

    public function confirm(): void
    {
        if (empty($this->items)) {
            throw new \DomainException('খালি অর্ডার confirm করা যায় না');
        }
        $this->status = 'confirmed';
    }

    public function cancel(): void
    {
        if ($this->status === 'delivered') {
            throw new \DomainException('ডেলিভারি হয়ে গেছে, বাতিল করা যাবে না');
        }
        $this->status = 'cancelled';
    }

    public function getId(): string { return $this->id; }
    public function getCustomerId(): string { return $this->customerId; }
    public function getStatus(): string { return $this->status; }
    public function getItems(): array { return $this->items; }
}

// =====================================================
// স্তর ২: APPLICATION / USE CASES
// =====================================================

// Application/Interfaces/OrderRepositoryInterface.php
namespace Application\Interfaces;

use Domain\Entities\Order;

interface OrderRepositoryInterface
{
    public function save(Order $order): void;
    public function findById(string $id): ?Order;
    public function generateId(): string;
}

// Application/Interfaces/PaymentGatewayInterface.php
namespace Application\Interfaces;

interface PaymentGatewayInterface
{
    public function charge(string $customerId, float $amount): bool;
}

// Application/DTOs/PlaceOrderInput.php
namespace Application\DTOs;

class PlaceOrderInput
{
    public function __construct(
        public readonly string $customerId,
        public readonly array $items
    ) {}
}

// Application/UseCases/PlaceOrderUseCase.php
namespace Application\UseCases;

use Domain\Entities\{Order, OrderItem, Money};
use Application\Interfaces\{OrderRepositoryInterface, PaymentGatewayInterface};
use Application\DTOs\PlaceOrderInput;

class PlaceOrderUseCase
{
    public function __construct(
        private OrderRepositoryInterface $orderRepo,
        private PaymentGatewayInterface $paymentGW
    ) {}

    public function execute(PlaceOrderInput $input): array
    {
        $orderId = $this->orderRepo->generateId();

        $order = new Order($orderId, $input->customerId);

        foreach ($input->items as $item) {
            $order->addItem(new OrderItem(
                $item['productId'],
                $item['name'],
                new Money($item['price']),
                $item['qty']
            ));
        }

        $order->confirm();
        $total = $order->calculateTotal();

        if (!$this->paymentGW->charge($input->customerId, $total->getAmount())) {
            throw new \RuntimeException('পেমেন্ট ব্যর্থ হয়েছে');
        }

        $this->orderRepo->save($order);

        return [
            'orderId'  => $order->getId(),
            'total'    => $total->getAmount(),
            'status'   => $order->getStatus(),
            'items'    => count($order->getItems()),
        ];
    }
}

// =====================================================
// স্তর ৩: ADAPTERS
// =====================================================

// Adapters/Controllers/OrderController.php
namespace Adapters\Controllers;

use Application\UseCases\PlaceOrderUseCase;
use Application\DTOs\PlaceOrderInput;

class OrderController
{
    public function __construct(private PlaceOrderUseCase $useCase) {}

    public function placeOrder(array $httpRequest): array
    {
        try {
            $input = new PlaceOrderInput(
                customerId: $httpRequest['customer_id'],
                items: $httpRequest['items']
            );

            $result = $this->useCase->execute($input);

            return ['status' => 201, 'body' => ['success' => true, 'data' => $result]];
        } catch (\DomainException $e) {
            return ['status' => 422, 'body' => ['success' => false, 'error' => $e->getMessage()]];
        } catch (\RuntimeException $e) {
            return ['status' => 402, 'body' => ['success' => false, 'error' => $e->getMessage()]];
        }
    }
}

// Adapters/Repositories/EloquentOrderRepository.php
namespace Adapters\Repositories;

use Domain\Entities\Order;
use Application\Interfaces\OrderRepositoryInterface;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): void
    {
        \App\Models\OrderModel::create([
            'id'          => $order->getId(),
            'customer_id' => $order->getCustomerId(),
            'status'      => $order->getStatus(),
            'total'       => $order->calculateTotal()->getAmount(),
        ]);

        foreach ($order->getItems() as $item) {
            \App\Models\OrderItemModel::create([
                'order_id'   => $order->getId(),
                'product_id' => $item->getProductId(),
                'name'       => $item->getProductName(),
                'quantity'   => $item->getQuantity(),
                'subtotal'   => $item->getSubtotal()->getAmount(),
            ]);
        }
    }

    public function findById(string $id): ?Order { /* ... */ }
    public function generateId(): string { return 'ORD-' . uniqid(); }
}

// Adapters/Gateways/BkashPaymentGateway.php
namespace Adapters\Gateways;

use Application\Interfaces\PaymentGatewayInterface;

class BkashPaymentGateway implements PaymentGatewayInterface
{
    public function charge(string $customerId, float $amount): bool
    {
        // bKash API কল করে পেমেন্ট চার্জ করে
        // Entity ও Use Case জানে না যে bKash ব্যবহার হচ্ছে!
        $response = Http::post('https://tokenized.pay.bka.sh/v1.2.0-beta/checkout/payment/create', [
            'amount'  => $amount,
            'intent'  => 'sale',
        ]);

        return $response->successful();
    }
}

// =====================================================
// স্তর ৪: FRAMEWORKS (Laravel)
// =====================================================

// routes/api.php
Route::post('/orders', function (Request $request) {
    $repo       = app(EloquentOrderRepository::class);
    $paymentGW  = app(BkashPaymentGateway::class);
    $useCase    = new PlaceOrderUseCase($repo, $paymentGW);
    $controller = new OrderController($useCase);

    $result = $controller->placeOrder($request->all());

    return response()->json($result['body'], $result['status']);
});
```

### সম্পূর্ণ JavaScript বাস্তবায়ন

```javascript
// =====================================================
// স্তর ১: DOMAIN / ENTITIES
// =====================================================

// domain/entities/Money.js
class Money {
    #amount;
    #currency;

    constructor(amount, currency = 'BDT') {
        if (amount < 0) throw new Error('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        this.#amount = amount;
        this.#currency = currency;
    }

    add(other) {
        if (this.#currency !== other.currency)
            throw new Error('ভিন্ন মুদ্রা দিয়ে অপারেশন করা যায় না');
        return new Money(this.#amount + other.amount, this.#currency);
    }

    multiply(factor) {
        return new Money(this.#amount * factor, this.#currency);
    }

    get amount() { return this.#amount; }
    get currency() { return this.#currency; }
}

// domain/entities/OrderItem.js
class OrderItem {
    #productId; #productName; #unitPrice; #quantity;

    constructor(productId, productName, unitPrice, quantity) {
        if (quantity <= 0) throw new Error('পণ্যের পরিমাণ শূন্যের বেশি হতে হবে');
        this.#productId = productId;
        this.#productName = productName;
        this.#unitPrice = unitPrice;
        this.#quantity = quantity;
    }

    getSubtotal() { return this.#unitPrice.multiply(this.#quantity); }
    get productId() { return this.#productId; }
    get productName() { return this.#productName; }
    get quantity() { return this.#quantity; }
}

// domain/entities/Order.js
class Order {
    #id; #customerId; #items; #status; #createdAt;

    constructor(id, customerId, items = []) {
        this.#id = id;
        this.#customerId = customerId;
        this.#items = [...items];
        this.#status = 'pending';
        this.#createdAt = new Date();
    }

    addItem(item) {
        if (this.#status !== 'pending')
            throw new Error('শুধু pending অর্ডারে আইটেম যোগ করা যায়');
        this.#items.push(item);
    }

    calculateTotal() {
        return this.#items.reduce(
            (total, item) => total.add(item.getSubtotal()),
            new Money(0)
        );
    }

    confirm() {
        if (this.#items.length === 0) throw new Error('খালি অর্ডার confirm করা যায় না');
        this.#status = 'confirmed';
    }

    cancel() {
        if (this.#status === 'delivered')
            throw new Error('ডেলিভারি হয়ে গেছে, বাতিল করা যাবে না');
        this.#status = 'cancelled';
    }

    get id() { return this.#id; }
    get customerId() { return this.#customerId; }
    get status() { return this.#status; }
    get items() { return [...this.#items]; }
}

// =====================================================
// স্তর ২: APPLICATION / USE CASES
// =====================================================

// application/use-cases/PlaceOrderUseCase.js
class PlaceOrderUseCase {
    #orderRepo;
    #paymentGW;

    constructor(orderRepository, paymentGateway) {
        this.#orderRepo = orderRepository;
        this.#paymentGW = paymentGateway;
    }

    async execute({ customerId, items }) {
        const orderId = await this.#orderRepo.generateId();
        const order = new Order(orderId, customerId);

        for (const item of items) {
            order.addItem(new OrderItem(
                item.productId, item.name, new Money(item.price), item.qty
            ));
        }

        order.confirm();
        const total = order.calculateTotal();

        const paid = await this.#paymentGW.charge(customerId, total.amount);
        if (!paid) throw new Error('পেমেন্ট ব্যর্থ হয়েছে');

        await this.#orderRepo.save(order);

        return {
            orderId: order.id,
            total: total.amount,
            status: order.status,
            itemCount: order.items.length,
        };
    }
}

// =====================================================
// স্তর ৩: ADAPTERS
// =====================================================

// adapters/controllers/OrderController.js
class OrderController {
    #placeOrderUC;

    constructor(placeOrderUseCase) {
        this.#placeOrderUC = placeOrderUseCase;
    }

    async handlePlaceOrder(req, res) {
        try {
            const result = await this.#placeOrderUC.execute({
                customerId: req.body.customer_id,
                items: req.body.items,
            });
            res.status(201).json({ success: true, data: result });
        } catch (err) {
            const status = err.message.includes('পেমেন্ট') ? 402 : 422;
            res.status(status).json({ success: false, error: err.message });
        }
    }
}

// adapters/repositories/MongoOrderRepository.js
class MongoOrderRepository {
    #collection;

    constructor(mongoClient) {
        this.#collection = mongoClient.db('shop').collection('orders');
    }

    async save(order) {
        await this.#collection.insertOne({
            _id: order.id,
            customerId: order.customerId,
            status: order.status,
            total: order.calculateTotal().amount,
            items: order.items.map((i) => ({
                productId: i.productId,
                name: i.productName,
                qty: i.quantity,
                subtotal: i.getSubtotal().amount,
            })),
            createdAt: new Date(),
        });
    }

    async findById(id) { /* ... */ }
    async generateId() { return 'ORD-' + Date.now().toString(36); }
}

// adapters/gateways/NagadPaymentGateway.js
class NagadPaymentGateway {
    #apiKey;

    constructor(apiKey) {
        this.#apiKey = apiKey;
    }

    async charge(customerId, amount) {
        // Nagad API কল — Entity ও Use Case জানে না কোন gateway!
        const response = await fetch('https://api.nagad.com.bd/payment/create', {
            method: 'POST',
            headers: { Authorization: `Bearer ${this.#apiKey}` },
            body: JSON.stringify({ amount, merchant_id: customerId }),
        });
        return response.ok;
    }
}

// =====================================================
// স্তর ৪: FRAMEWORKS (Express)
// =====================================================

// frameworks/express/server.js
const express = require('express');
const { MongoClient } = require('mongodb');

async function bootstrap() {
    const app = express();
    app.use(express.json());

    const mongo = await MongoClient.connect('mongodb://localhost:27017');
    const orderRepo = new MongoOrderRepository(mongo);
    const paymentGW = new NagadPaymentGateway(process.env.NAGAD_API_KEY);
    const placeOrderUC = new PlaceOrderUseCase(orderRepo, paymentGW);
    const orderCtrl = new OrderController(placeOrderUC);

    app.post('/api/orders', (req, res) => orderCtrl.handlePlaceOrder(req, res));

    app.listen(3000, () => console.log('🚀 সার্ভার চালু: http://localhost:3000'));
}

bootstrap();
```

---

## ✅❌ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

- **জটিল business logic** আছে (যেমন: Daraz-এর অর্ডার ম্যানেজমেন্ট, bKash-এর লেনদেন যাচাই)
- **দীর্ঘমেয়াদী প্রজেক্ট** যা ৩+ বছর চলবে
- **বড় টিম** (৩+ ডেভেলপার) — প্রতিটি স্তরে আলাদা দল কাজ করতে পারে
- **Database বা framework পরিবর্তনের সম্ভাবনা** আছে
- **একই business logic** একাধিক platform-এ ব্যবহার হবে (web, mobile API, CLI)
- **Test-driven development (TDD)** অনুসরণ করতে চান
- Pathao-র মতো প্রজেক্ট যেখানে ride matching, payment, notification — সবই আলাদা concern

### ❌ কখন ব্যবহার করবেন না

- **সাধারণ CRUD অ্যাপ** (ছোট ব্লগ, টু-ডু লিস্ট)
- **Prototype বা MVP** যা দ্রুত বাজারে আনতে হবে
- **ছোট স্ক্রিপ্ট** বা utility tool
- **একক ডেভেলপারের ছোট প্রজেক্ট** যেখানে over-engineering হয়ে যায়
- **Hackathon বা PoC** — যেখানে গতিই মূল্যবান

### সিদ্ধান্ত ফ্লোচার্ট

```
    শুরু: নতুন প্রজেক্ট শুরু করছেন?
      │
      ▼
    ┌──────────────────────────────────┐
    │ Business logic কি জটিল?          │
    │ (শুধু CRUD নয়)                   │
    └──────────┬───────────────────────┘
               │
          হ্যাঁ │              না
               │               │
               ▼               ▼
    ┌─────────────────┐   ┌─────────────────────┐
    │ টিম কি ৩+ জন?  │   │ সাধারণ MVC/CRUD      │
    └────────┬────────┘   │ ব্যবহার করুন          │
             │            └─────────────────────┘
        হ্যাঁ │       না
             │        │
             ▼        ▼
    ┌──────────────┐  ┌───────────────────────┐
    │ প্রজেক্ট কি │  │ Simplified Clean       │
    │ ৩+ বছর      │  │ Architecture ব্যবহার   │
    │ চলবে?       │  │ করুন (২-৩ স্তর)        │
    └──────┬──────┘  └───────────────────────┘
           │
      হ্যাঁ │       না
           │        │
           ▼        ▼
    ┌──────────┐  ┌────────────────────────┐
    │ ✅ Full  │  │ মাঝামাঝি পদ্ধতি:        │
    │ Clean    │  │ Use Case + Repository  │
    │ Arch     │  │ Pattern যথেষ্ট          │
    │ ব্যবহার  │  └────────────────────────┘
    │ করুন!    │
    └──────────┘
```

---

## 🚨 সাধারণ ভুল (Common Mistakes)

### ভুল ১: স্তর মিশ্রিত করা (Mixing Layers)

```
    ❌ ভুল — Entity তে SQL query:

    class Order {
        public function save() {
            DB::table('orders')->insert([...]);  // 🚫 Entity তে database!
        }
    }

    ✅ সঠিক — Entity শুধু business rule জানে:

    class Order {
        public function confirm() {
            if (empty($this->items)) throw new DomainException('...');
            $this->status = 'confirmed';
        }
    }
    // Database কাজ Repository-তে হবে
```

### ভুল ২: Anemic Domain Model

Anemic Domain Model হলো এমন Entity যেখানে **শুধু getter/setter** আছে, কোনো business logic নেই। Entity শুধু ডেটা ধারক হয়ে যায়, সমস্ত logic Use Case বা Service-এ চলে যায়।

```php
// ❌ Anemic — শুধু ডেটা বহন করছে, কোনো নিয়ম নেই
class Order {
    public string $id;
    public string $status;
    public float $total;
    // শুধু public properties — কোনো behavior নেই!
}

// ✅ Rich Domain Model — Entity নিজের নিয়ম জানে
class Order {
    private string $status = 'pending';

    public function confirm(): void {
        if (empty($this->items)) {
            throw new \DomainException('খালি অর্ডার confirm করা যায় না');
        }
        if ($this->calculateTotal()->getAmount() > 500000) {
            throw new \DomainException('৫ লাখ টাকার বেশি অর্ডারে ম্যানুয়াল অনুমোদন লাগবে');
        }
        $this->status = 'confirmed';
    }
}
```

### ভুল ৩: সাধারণ অ্যাপে Over-engineering

```
    ❌ একটি সাধারণ টু-ডু অ্যাপে Clean Architecture:

    📁 todo-app/
    ├── Domain/Entities/Todo.php           ← ১ লাইন: $title, $done
    ├── Application/UseCases/AddTodo.php   ← ৩ লাইন: $repo->save()
    ├── Adapters/Controllers/...           ← ৫ লাইন
    ├── Adapters/Repositories/...          ← ১০ লাইন
    └── Frameworks/Laravel/...             ← ২০ লাইন config

    মোট: ১৫টি ফাইল, ৫০ লাইন actual logic 😅

    ✅ সাধারণ MVC-ই যথেষ্ট:

    📁 todo-app/
    ├── TodoController.php                 ← ২০ লাইন
    ├── Todo.php (Model)                   ← ৫ লাইন
    └── routes.php                         ← ৩ লাইন

    মোট: ৩টি ফাইল, ২৮ লাইন — সরল ও কার্যকর! ✅
```

### ভুল ৪: Dependency Rule উল্টো করা

```
    ❌ Use Case সরাসরি concrete class ব্যবহার করছে:

    class PlaceOrderUseCase {
        public function execute() {
            $repo = new MySqlOrderRepository();  // 🚫 Concrete dependency!
            $repo->save($order);
        }
    }

    ✅ Use Case interface-এর উপর নির্ভর, concrete class inject হয়:

    class PlaceOrderUseCase {
        public function __construct(
            private OrderRepositoryInterface $repo  // ✅ Interface!
        ) {}

        public function execute() {
            $this->repo->save($order);
        }
    }
```

---

## 📊 সংক্ষিপ্ত রেফারেন্স (Quick Reference)

```
    ╔══════════════════════════════════════════════════════════════════╗
    ║              CLEAN ARCHITECTURE — মনে রাখার সূত্র               ║
    ╠══════════════════════════════════════════════════════════════════╣
    ║                                                                  ║
    ║  ১. Dependency Rule: বাইরে → ভেতরে (কখনো উল্টো নয়)            ║
    ║                                                                  ║
    ║  ২. Entity: ব্যবসায়িক নিয়ম, কোনো dependency নেই               ║
    ║                                                                  ║
    ║  ৩. Use Case: অ্যাপ-নির্দিষ্ট নিয়ম, Entity ব্যবহার করে         ║
    ║                                                                  ║
    ║  ৪. Adapter: ডেটা রূপান্তর (HTTP ↔ DTO, Entity ↔ DB)          ║
    ║                                                                  ║
    ║  ৫. Framework: Glue code, সবকিছু একত্রিত করে                   ║
    ║                                                                  ║
    ║  ৬. Interface দিয়ে dependency inversion করুন                    ║
    ║                                                                  ║
    ║  ৭. সাধারণ CRUD অ্যাপে ব্যবহার করবেন না — over-engineering!    ║
    ║                                                                  ║
    ╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 আরও জানতে

- 📖 **Clean Architecture** — Robert C. Martin (বই)
- 📖 **The Clean Coder** — Robert C. Martin
- 🌐 [The Clean Architecture Blog Post](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- 🎥 Uncle Bob-এর Clean Architecture বিষয়ক YouTube lectures
- 📖 সম্পর্কিত নথি: [`hexagonal.md`](./hexagonal.md) — হেক্সাগোনাল আর্কিটেকচার বিস্তারিত
