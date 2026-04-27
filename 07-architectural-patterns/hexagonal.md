# ⬡ হেক্সাগোনাল আর্কিটেকচার (পোর্টস ও অ্যাডাপ্টার্স)

## 📌 সংজ্ঞা ও মূল ধারণা

**Hexagonal Architecture** (যা **Ports and Adapters Pattern** নামেও পরিচিত) হলো একটি সফটওয়্যার আর্কিটেকচারাল প্যাটার্ন যা ২০০৫ সালে **Alistair Cockburn** প্রথম প্রস্তাব করেন। এই আর্কিটেকচারের মূল উদ্দেশ্য হলো — **অ্যাপ্লিকেশনের business logic কে বাইরের সমস্ত infrastructure concerns থেকে সম্পূর্ণভাবে আলাদা করা**।

### কেন "Hexagonal"?

অনেকে মনে করেন hexagon-এর ৬টি বাহু মানে ৬টি port — এটি ভুল ধারণা। Cockburn hexagonal shape ব্যবহার করেছেন কারণ:

- **ষড়ভুজ আকৃতি যথেষ্ট পরিমাণ "বাহু" প্রদান করে** যাতে বোঝানো যায় যে অ্যাপ্লিকেশনের সাথে বাইরের বিশ্বের একাধিক connection point থাকতে পারে
- **এটি কোনো নির্দিষ্ট সংখ্যা নয়** — application-এর যতগুলো port দরকার ততগুলো থাকতে পারে
- **Traditional layered architecture-এর box diagram থেকে আলাদা** দেখানোর জন্য এই shape ব্যবহার করা হয়েছে

### মূল ধারণাসমূহ (Core Concepts)

| ধারণা | বাংলায় ব্যাখ্যা | English |
|-------|-----------------|---------|
| **Domain** | অ্যাপ্লিকেশনের হৃৎপিণ্ড — সমস্ত business logic এখানে থাকে | Core Business Logic |
| **Port** | একটি interface/contract যা domain-এর সাথে বাইরের জগতের যোগাযোগের নিয়ম নির্ধারণ করে | Interface/Contract |
| **Adapter** | Port-এর concrete implementation যা নির্দিষ্ট technology ব্যবহার করে | Implementation |
| **Primary/Driving Port** | বাইরের জগৎ domain-কে ব্যবহার করার জন্য যে port দিয়ে ঢোকে | Inbound Port |
| **Secondary/Driven Port** | Domain বাইরের জগতের সাথে কথা বলার জন্য যে port ব্যবহার করে | Outbound Port |
| **Driving Adapter** | ব্যবহারকারীর request নিয়ে domain-এ পৌঁছে দেয় (Controller, CLI) | Inbound Adapter |
| **Driven Adapter** | Domain-এর নির্দেশে বাইরের কাজ সম্পন্ন করে (Database, API) | Outbound Adapter |

### Dependency Rule — সবচেয়ে গুরুত্বপূর্ণ নিয়ম

```
🚫 ভুল: Domain → Infrastructure (Domain জানে কোন database ব্যবহার হচ্ছে)
✅ সঠিক: Infrastructure → Domain (Infrastructure domain-এর interface implement করে)
```

**Dependency সবসময় বাইরে থেকে ভেতরের দিকে যায়, কখনোই ভেতর থেকে বাইরে নয়।** Domain layer কখনো জানবে না যে MySQL ব্যবহার হচ্ছে নাকি MongoDB, Stripe ব্যবহার হচ্ছে নাকি bKash। Domain শুধু interface (port) জানে, concrete implementation (adapter) নয়।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### ইলেকট্রিক্যাল অ্যাপ্লায়েন্স অ্যানালজি

কল্পনা করুন আপনার একটি **ল্যাপটপ** আছে:

```
🔌 ল্যাপটপ চার্জিং সিস্টেম:

┌─────────────────────────────────────────────────┐
│                                                   │
│   বিদ্যুৎ সরবরাহ          ল্যাপটপ (Domain)       │
│   ─────────────────────────────────────────────── │
│                                                   │
│   🏠 বাড়ির সকেট ──┐                              │
│                     │    ┌──────────────┐          │
│   🏢 অফিস সকেট ──┤───→│ চার্জিং পোর্ট │──→ ⚡    │
│                     │    │   (Port)      │   ব্যাটারি│
│   🚗 গাড়ির সকেট ──┘    └──────────────┘          │
│                                                   │
│   Adapters (Plugs)        Port              Domain │
│   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│   - বাংলাদেশি প্লাগ       চার্জিং স্ট্যান্ডার্ড    ল্যাপটপ │
│   - ইউরোপিয়ান প্লাগ       (USB-C/Barrel)          CPU,RAM │
│   - আমেরিকান প্লাগ                                 GPU     │
└─────────────────────────────────────────────────┘
```

**মূল বিষয়:** ল্যাপটপ (domain) জানে না বিদ্যুৎ কোত্থেকে আসছে — বাড়ি, অফিস, নাকি গাড়ি থেকে। সে শুধু জানে তার একটি **চার্জিং পোর্ট** আছে (interface)। বিভিন্ন দেশের বিভিন্ন **প্লাগ** (adapter) সেই পোর্টে সংযুক্ত হয়। ল্যাপটপের hardware (domain logic) পরিবর্তন করতে হয় না — শুধু সঠিক adapter ব্যবহার করলেই চলে।

### বাংলাদেশি ই-কমার্স উদাহরণ

ধরুন, **"দারাজ" স্টাইলের একটি ই-কমার্স সাইট** চালাচ্ছেন:

- **Domain (ব্যবসায়িক নিয়ম):** অর্ডার তৈরি, মূল্য গণনা, ডিসকাউন্ট, ইনভেন্টরি চেক
- **Primary Port:** "অর্ডার দিন" — এই কাজটি কে করতে চায়?
  - **Adapter 1:** ওয়েবসাইট (REST API)
  - **Adapter 2:** মোবাইল অ্যাপ (GraphQL)
  - **Adapter 3:** কল সেন্টার অপারেটর (CLI tool)
- **Secondary Port:** "পেমেন্ট নিন" — domain কিভাবে পেমেন্ট নেবে?
  - **Adapter 1:** bKash
  - **Adapter 2:** Nagad
  - **Adapter 3:** Visa/Mastercard

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Hexagonal Architecture — সম্পূর্ণ চিত্র

```
                    DRIVING SIDE                           DRIVEN SIDE
                  (Primary/Inbound)                    (Secondary/Outbound)

          ┌──────────────────┐              ┌──────────────────┐
          │  REST Controller │              │  MySQL Repository│
          │  (HTTP Adapter)  │              │  (DB Adapter)    │
          └────────┬─────────┘              └────────▲─────────┘
                   │                                  │
          ┌────────┴─────────┐              ┌────────┴─────────┐
          │  GraphQL Resolver│              │  Stripe/bKash    │
          │  (GQL Adapter)   │              │  (Payment Adapter│
          └────────┬─────────┘              └────────▲─────────┘
                   │                                  │
                   ▼                                  │
        ┌─────────────────────────────────────────────────────┐
        │                                                       │
        │    ┌─────────────────────────────────────────────┐    │
        │    │          PRIMARY PORTS (Interfaces)          │    │
        │    │   OrderServicePort, UserServicePort          │    │
        │    └──────────────────┬────────────────────────── ┘    │
        │                       │                                │
        │    ┌──────────────────▼──────────────────────────┐    │
        │    │              🏛️ DOMAIN                       │    │
        │    │                                              │    │
        │    │   Entities:   Order, User, Product           │    │
        │    │   Value Objs: Money, Address, OrderId        │    │
        │    │   Services:   PricingService, StockService   │    │
        │    │   Events:     OrderPlaced, PaymentReceived   │    │
        │    │                                              │    │
        │    └──────────────────┬────────────────────────── ┘    │
        │                       │                                │
        │    ┌──────────────────▼──────────────────────────┐    │
        │    │         SECONDARY PORTS (Interfaces)         │    │
        │    │   OrderRepoPort, PaymentPort, NotifyPort     │    │
        │    └─────────────────────────────────────────────┘    │
        │                       │                                │
        └───────────────────────┼────────────────────────────── ┘
                                │
          ┌────────┬────────────┼────────────┬──────────┐
          ▼        ▼            ▼            ▼          ▼
       ┌──────┐ ┌──────┐  ┌────────┐  ┌────────┐ ┌────────┐
       │ Email│ │ SMS  │  │ Redis  │  │ S3     │ │Webhook │
       │Notify│ │Notify│  │ Cache  │  │Storage │ │ Event  │
       └──────┘ └──────┘  └────────┘  └────────┘ └────────┘
```

### Dependency Direction — নির্ভরতার দিক

```
     ┌──────────────────────────────────────────────────┐
     │              INFRASTRUCTURE LAYER                 │
     │   Controllers, Repositories, External Services    │
     │                                                    │
     │    ┌──────────────────────────────────────────┐   │
     │    │           APPLICATION LAYER               │   │
     │    │    Application Services, Use Cases         │   │
     │    │                                            │   │
     │    │    ┌────────────────────────────────────┐  │   │
     │    │    │          DOMAIN LAYER              │  │   │
     │    │    │  Entities, Value Objects,           │  │   │
     │    │    │  Domain Services, Domain Events     │  │   │
     │    │    │  ★ কোনো বাইরের নির্ভরতা নেই ★      │  │   │
     │    │    └────────────────────────────────────┘  │   │
     │    │                                            │   │
     │    └──────────────────────────────────────────┘   │
     │                                                    │
     └──────────────────────────────────────────────────┘

     নির্ভরতার দিক: বাইরে ──────→ ভেতরে (সর্বদা)
```

---

## 💻 Core Concepts Deep Dive

### ১. Domain Layer (কেন্দ্রবিন্দু)

Domain layer হলো আপনার অ্যাপ্লিকেশনের **হৃৎপিণ্ড**। এখানে সমস্ত business logic থাকে এবং এই layer-এ কোনো framework dependency থাকবে না — কোনো Laravel, Express, Eloquent, Mongoose কিছুই না।

#### PHP 8.3 — Domain Entities ও Value Objects

```php
<?php

declare(strict_types=1);

namespace Domain\Order\ValueObject;

// Value Object — immutable, কোনো identity নেই, শুধু মান দিয়ে তুলনা হয়
final readonly class Money
{
    public function __construct(
        public float $amount,
        public string $currency = 'BDT',
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

    public function subtract(self $other): self
    {
        $this->ensureSameCurrency($other);
        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(int|float $factor): self
    {
        return new self($this->amount * $factor, $this->currency);
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function ensureSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException("মুদ্রা মিলছে না: {$this->currency} vs {$other->currency}");
        }
    }
}

// Domain Enum — order-এর সম্ভাব্য অবস্থাসমূহ
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
            self::Pending => in_array($next, [self::Confirmed, self::Cancelled]),
            self::Confirmed => in_array($next, [self::Processing, self::Cancelled]),
            self::Processing => in_array($next, [self::Shipped, self::Cancelled]),
            self::Shipped => in_array($next, [self::Delivered]),
            self::Delivered, self::Cancelled => false,
        };
    }
}

// Entity — unique identity আছে, জীবনচক্র আছে
final class Order
{
    /** @var OrderItem[] */
    private array $items = [];
    private array $domainEvents = [];

    public function __construct(
        public readonly string $id,
        public readonly string $customerId,
        private OrderStatus $status = OrderStatus::Pending,
        private ?Money $totalAmount = null,
    ) {
        $this->totalAmount = new Money(0);
    }

    public function addItem(string $productId, string $name, Money $price, int $quantity): void
    {
        if ($this->status !== OrderStatus::Pending) {
            throw new \DomainException('শুধুমাত্র Pending অর্ডারে আইটেম যোগ করা যায়');
        }

        $item = new OrderItem($productId, $name, $price, $quantity);
        $this->items[] = $item;
        $this->recalculateTotal();
    }

    public function confirm(): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Confirmed)) {
            throw new \DomainException("অর্ডার {$this->status->value} অবস্থায় confirm করা যায় না");
        }

        if (empty($this->items)) {
            throw new \DomainException('খালি অর্ডার confirm করা যায় না');
        }

        $this->status = OrderStatus::Confirmed;
        $this->domainEvents[] = new OrderConfirmedEvent($this->id, $this->customerId, $this->totalAmount);
    }

    public function cancel(): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Cancelled)) {
            throw new \DomainException("অর্ডার {$this->status->value} অবস্থায় cancel করা যায় না");
        }

        $this->status = OrderStatus::Cancelled;
        $this->domainEvents[] = new OrderCancelledEvent($this->id);
    }

    public function getStatus(): OrderStatus { return $this->status; }
    public function getTotal(): Money { return $this->totalAmount; }
    public function getItems(): array { return $this->items; }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    private function recalculateTotal(): void
    {
        $total = new Money(0);
        foreach ($this->items as $item) {
            $total = $total->add($item->getSubtotal());
        }
        $this->totalAmount = $total;
    }
}

final readonly class OrderItem
{
    public function __construct(
        public string $productId,
        public string $name,
        public Money $price,
        public int $quantity,
    ) {
        if ($quantity <= 0) {
            throw new \DomainException('পরিমাণ অবশ্যই ধনাত্মক হতে হবে');
        }
    }

    public function getSubtotal(): Money
    {
        return $this->price->multiply($this->quantity);
    }
}
```

#### JavaScript (Node.js) — Domain Entities ও Value Objects

```javascript
// domain/valueObjects/Money.js

export class Money {
  #amount;
  #currency;

  constructor(amount, currency = 'BDT') {
    if (amount < 0) {
      throw new DomainError('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
    }
    this.#amount = amount;
    this.#currency = currency;
    Object.freeze(this);
  }

  get amount() { return this.#amount; }
  get currency() { return this.#currency; }

  add(other) {
    this.#ensureSameCurrency(other);
    return new Money(this.#amount + other.amount, this.#currency);
  }

  subtract(other) {
    this.#ensureSameCurrency(other);
    return new Money(this.#amount - other.amount, this.#currency);
  }

  multiply(factor) {
    return new Money(this.#amount * factor, this.#currency);
  }

  equals(other) {
    return this.#amount === other.amount && this.#currency === other.currency;
  }

  #ensureSameCurrency(other) {
    if (this.#currency !== other.currency) {
      throw new DomainError(`মুদ্রা মিলছে না: ${this.#currency} vs ${other.currency}`);
    }
  }
}

// domain/valueObjects/OrderStatus.js

export class OrderStatus {
  static Pending = new OrderStatus('pending');
  static Confirmed = new OrderStatus('confirmed');
  static Processing = new OrderStatus('processing');
  static Shipped = new OrderStatus('shipped');
  static Delivered = new OrderStatus('delivered');
  static Cancelled = new OrderStatus('cancelled');

  #value;
  #transitions = new Map([
    ['pending', ['confirmed', 'cancelled']],
    ['confirmed', ['processing', 'cancelled']],
    ['processing', ['shipped', 'cancelled']],
    ['shipped', ['delivered']],
    ['delivered', []],
    ['cancelled', []],
  ]);

  constructor(value) {
    this.#value = value;
    Object.freeze(this);
  }

  get value() { return this.#value; }

  canTransitionTo(next) {
    return this.#transitions.get(this.#value)?.includes(next.value) ?? false;
  }
}

// domain/entities/Order.js

export class Order {
  #id;
  #customerId;
  #status;
  #items = [];
  #totalAmount;
  #domainEvents = [];

  constructor(id, customerId) {
    this.#id = id;
    this.#customerId = customerId;
    this.#status = OrderStatus.Pending;
    this.#totalAmount = new Money(0);
  }

  get id() { return this.#id; }
  get customerId() { return this.#customerId; }
  get status() { return this.#status; }
  get total() { return this.#totalAmount; }
  get items() { return [...this.#items]; }

  addItem(productId, name, price, quantity) {
    if (this.#status.value !== 'pending') {
      throw new DomainError('শুধুমাত্র Pending অর্ডারে আইটেম যোগ করা যায়');
    }
    this.#items.push(new OrderItem(productId, name, price, quantity));
    this.#recalculateTotal();
  }

  confirm() {
    if (!this.#status.canTransitionTo(OrderStatus.Confirmed)) {
      throw new DomainError(`অর্ডার ${this.#status.value} অবস্থায় confirm করা যায় না`);
    }
    if (this.#items.length === 0) {
      throw new DomainError('খালি অর্ডার confirm করা যায় না');
    }
    this.#status = OrderStatus.Confirmed;
    this.#domainEvents.push({
      type: 'OrderConfirmed',
      orderId: this.#id,
      customerId: this.#customerId,
      total: this.#totalAmount,
    });
  }

  cancel() {
    if (!this.#status.canTransitionTo(OrderStatus.Cancelled)) {
      throw new DomainError(`অর্ডার ${this.#status.value} অবস্থায় cancel করা যায় না`);
    }
    this.#status = OrderStatus.Cancelled;
    this.#domainEvents.push({ type: 'OrderCancelled', orderId: this.#id });
  }

  pullDomainEvents() {
    const events = [...this.#domainEvents];
    this.#domainEvents = [];
    return events;
  }

  #recalculateTotal() {
    this.#totalAmount = this.#items.reduce(
      (sum, item) => sum.add(item.subtotal),
      new Money(0),
    );
  }
}

export class OrderItem {
  #productId;
  #name;
  #price;
  #quantity;

  constructor(productId, name, price, quantity) {
    if (quantity <= 0) throw new DomainError('পরিমাণ অবশ্যই ধনাত্মক হতে হবে');
    this.#productId = productId;
    this.#name = name;
    this.#price = price;
    this.#quantity = quantity;
    Object.freeze(this);
  }

  get productId() { return this.#productId; }
  get name() { return this.#name; }
  get price() { return this.#price; }
  get quantity() { return this.#quantity; }
  get subtotal() { return this.#price.multiply(this.#quantity); }
}
```

---

### ২. Ports (ইন্টারফেস/চুক্তিপত্র)

Port হলো **চুক্তিপত্র (contract)** — এটি বলে "কী করতে হবে" কিন্তু "কিভাবে করতে হবে" তা বলে না।

#### PHP 8.3 — Ports (Interfaces)

```php
<?php

declare(strict_types=1);

// ═══════════════════════════════════════════════════════
// PRIMARY/DRIVING PORTS — বাইরে থেকে domain-কে ব্যবহার
// ═══════════════════════════════════════════════════════

namespace Domain\Order\Port\In;

interface PlaceOrderUseCasePort
{
    public function execute(PlaceOrderCommand $command): OrderResult;
}

interface CancelOrderUseCasePort
{
    public function execute(string $orderId): void;
}

interface GetOrderQueryPort
{
    public function findById(string $orderId): ?OrderDTO;
    public function findByCustomer(string $customerId, int $page = 1): PaginatedResult;
}

// Command object — use case-এর input
final readonly class PlaceOrderCommand
{
    /**
     * @param OrderItemDTO[] $items
     */
    public function __construct(
        public string $customerId,
        public array $items,
        public string $shippingAddress,
        public string $paymentMethod,
    ) {}
}

// ═══════════════════════════════════════════════════════
// SECONDARY/DRIVEN PORTS — domain বাইরের জগতের সাথে কথা বলে
// ═══════════════════════════════════════════════════════

namespace Domain\Order\Port\Out;

interface OrderRepositoryPort
{
    public function save(Order $order): void;
    public function findById(string $id): ?Order;
    public function findByCustomerId(string $customerId): array;
    public function nextIdentity(): string;
}

interface PaymentGatewayPort
{
    public function charge(string $orderId, Money $amount, string $method): PaymentResult;
    public function refund(string $transactionId, Money $amount): RefundResult;
}

interface NotificationPort
{
    public function notifyOrderConfirmed(string $customerId, string $orderId, Money $total): void;
    public function notifyOrderShipped(string $customerId, string $orderId, string $trackingNumber): void;
}

interface InventoryPort
{
    public function checkAvailability(string $productId, int $quantity): bool;
    public function reserve(string $productId, int $quantity, string $orderId): void;
    public function release(string $productId, int $quantity, string $orderId): void;
}

// Payment result — port-এর output
final readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public ?string $transactionId,
        public ?string $errorMessage = null,
    ) {}
}
```

#### JavaScript (Node.js) — Ports

```javascript
// domain/ports/in/PlaceOrderUseCase.js

/**
 * Primary Port — অর্ডার তৈরির চুক্তিপত্র
 * @interface
 */
export class PlaceOrderUseCasePort {
  /**
   * @param {PlaceOrderCommand} command
   * @returns {Promise<OrderResult>}
   */
  async execute(command) {
    throw new Error('Port method execute() must be implemented');
  }
}

// domain/ports/out/OrderRepositoryPort.js

/**
 * Secondary Port — অর্ডার সংরক্ষণের চুক্তিপত্র
 * Domain জানে না এটি MySQL, MongoDB নাকি file-এ save হচ্ছে
 * @interface
 */
export class OrderRepositoryPort {
  async save(order) { throw new Error('Not implemented'); }
  async findById(id) { throw new Error('Not implemented'); }
  async findByCustomerId(customerId) { throw new Error('Not implemented'); }
  async nextIdentity() { throw new Error('Not implemented'); }
}

// domain/ports/out/PaymentGatewayPort.js

export class PaymentGatewayPort {
  async charge(orderId, amount, method) { throw new Error('Not implemented'); }
  async refund(transactionId, amount) { throw new Error('Not implemented'); }
}

// domain/ports/out/NotificationPort.js

export class NotificationPort {
  async notifyOrderConfirmed(customerId, orderId, total) { throw new Error('Not implemented'); }
  async notifyOrderShipped(customerId, orderId, trackingNumber) { throw new Error('Not implemented'); }
}

// domain/ports/out/InventoryPort.js

export class InventoryPort {
  async checkAvailability(productId, quantity) { throw new Error('Not implemented'); }
  async reserve(productId, quantity, orderId) { throw new Error('Not implemented'); }
  async release(productId, quantity, orderId) { throw new Error('Not implemented'); }
}
```

---

### ৩. Adapters (বাস্তবায়ন)

Adapter হলো port-এর **concrete implementation** — এটি নির্দিষ্ট technology ব্যবহার করে port-এর contract পূরণ করে।

#### PHP 8.3 — Driving Adapter (REST Controller)

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Http\Controller;

use Domain\Order\Port\In\PlaceOrderUseCasePort;
use Domain\Order\Port\In\PlaceOrderCommand;

// Driving Adapter — HTTP request নিয়ে domain-এ পৌঁছে দেয়
final readonly class OrderController
{
    public function __construct(
        private PlaceOrderUseCasePort $placeOrder,
        private GetOrderQueryPort $getOrder,
        private CancelOrderUseCasePort $cancelOrder,
    ) {}

    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'customer_id' => 'required|string',
            'items' => 'required|array|min:1',
            'items.*.product_id' => 'required|string',
            'items.*.quantity' => 'required|integer|min:1',
            'shipping_address' => 'required|string',
            'payment_method' => 'required|in:bkash,nagad,card',
        ]);

        $command = new PlaceOrderCommand(
            customerId: $validated['customer_id'],
            items: array_map(
                fn(array $item) => new OrderItemDTO($item['product_id'], $item['quantity']),
                $validated['items'],
            ),
            shippingAddress: $validated['shipping_address'],
            paymentMethod: $validated['payment_method'],
        );

        $result = $this->placeOrder->execute($command);

        return response()->json([
            'order_id' => $result->orderId,
            'total' => $result->total->amount,
            'status' => $result->status,
        ], 201);
    }

    public function show(string $orderId): JsonResponse
    {
        $order = $this->getOrder->findById($orderId);
        if (!$order) {
            return response()->json(['error' => 'অর্ডার পাওয়া যায়নি'], 404);
        }
        return response()->json($order);
    }
}
```

#### PHP 8.3 — Driven Adapter (MySQL Repository)

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Persistence\MySQL;

use Domain\Order\Port\Out\OrderRepositoryPort;
use Domain\Order\Entity\Order;
use Illuminate\Support\Str;

// Driven Adapter — domain-এর নির্দেশে database-এ কাজ করে
final class MySQLOrderRepository implements OrderRepositoryPort
{
    public function __construct(
        private readonly \PDO $connection,
    ) {}

    public function save(Order $order): void
    {
        $this->connection->beginTransaction();

        try {
            $stmt = $this->connection->prepare(
                'INSERT INTO orders (id, customer_id, status, total_amount, currency)
                 VALUES (:id, :cid, :status, :amount, :currency)
                 ON DUPLICATE KEY UPDATE status = :status, total_amount = :amount'
            );

            $stmt->execute([
                'id' => $order->id,
                'cid' => $order->customerId,
                'status' => $order->getStatus()->value,
                'amount' => $order->getTotal()->amount,
                'currency' => $order->getTotal()->currency,
            ]);

            foreach ($order->getItems() as $item) {
                $itemStmt = $this->connection->prepare(
                    'INSERT IGNORE INTO order_items (order_id, product_id, name, price, quantity)
                     VALUES (:oid, :pid, :name, :price, :qty)'
                );
                $itemStmt->execute([
                    'oid' => $order->id,
                    'pid' => $item->productId,
                    'name' => $item->name,
                    'price' => $item->price->amount,
                    'qty' => $item->quantity,
                ]);
            }

            $this->connection->commit();
        } catch (\Throwable $e) {
            $this->connection->rollBack();
            throw $e;
        }
    }

    public function findById(string $id): ?Order
    {
        $stmt = $this->connection->prepare('SELECT * FROM orders WHERE id = :id');
        $stmt->execute(['id' => $id]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);

        if (!$row) return null;

        return $this->hydrateOrder($row);
    }

    public function findByCustomerId(string $customerId): array
    {
        $stmt = $this->connection->prepare('SELECT * FROM orders WHERE customer_id = :cid');
        $stmt->execute(['cid' => $customerId]);

        return array_map(
            fn(array $row) => $this->hydrateOrder($row),
            $stmt->fetchAll(\PDO::FETCH_ASSOC),
        );
    }

    public function nextIdentity(): string
    {
        return Str::uuid()->toString();
    }

    private function hydrateOrder(array $row): Order
    {
        // Database row থেকে Domain Entity তৈরি
        $order = new Order(
            id: $row['id'],
            customerId: $row['customer_id'],
            status: OrderStatus::from($row['status']),
            totalAmount: new Money((float) $row['total_amount'], $row['currency']),
        );

        // Items লোড করা
        $itemStmt = $this->connection->prepare(
            'SELECT * FROM order_items WHERE order_id = :oid'
        );
        $itemStmt->execute(['oid' => $row['id']]);

        foreach ($itemStmt->fetchAll(\PDO::FETCH_ASSOC) as $itemRow) {
            $order->addItem(
                $itemRow['product_id'],
                $itemRow['name'],
                new Money((float) $itemRow['price']),
                (int) $itemRow['quantity'],
            );
        }

        return $order;
    }
}
```

#### JavaScript (Node.js) — Driving Adapter (Express Controller)

```javascript
// infrastructure/http/controllers/OrderController.js

export class OrderController {
  #placeOrderUseCase;
  #getOrderQuery;

  constructor(placeOrderUseCase, getOrderQuery) {
    this.#placeOrderUseCase = placeOrderUseCase;
    this.#getOrderQuery = getOrderQuery;
  }

  async store(req, res) {
    try {
      const { customerId, items, shippingAddress, paymentMethod } = req.body;

      const result = await this.#placeOrderUseCase.execute({
        customerId,
        items,
        shippingAddress,
        paymentMethod,
      });

      res.status(201).json({
        orderId: result.orderId,
        total: result.total.amount,
        status: result.status,
      });
    } catch (error) {
      if (error instanceof DomainError) {
        res.status(400).json({ error: error.message });
      } else {
        res.status(500).json({ error: 'সার্ভারে সমস্যা হয়েছে' });
      }
    }
  }

  async show(req, res) {
    const order = await this.#getOrderQuery.findById(req.params.id);
    if (!order) {
      return res.status(404).json({ error: 'অর্ডার পাওয়া যায়নি' });
    }
    res.json(order);
  }
}
```

#### JavaScript (Node.js) — Driven Adapter (MongoDB Repository)

```javascript
// infrastructure/persistence/MongoOrderRepository.js

import { randomUUID } from 'node:crypto';

export class MongoOrderRepository extends OrderRepositoryPort {
  #collection;

  constructor(mongoDb) {
    super();
    this.#collection = mongoDb.collection('orders');
  }

  async save(order) {
    await this.#collection.updateOne(
      { _id: order.id },
      {
        $set: {
          customerId: order.customerId,
          status: order.status.value,
          totalAmount: order.total.amount,
          currency: order.total.currency,
          items: order.items.map(item => ({
            productId: item.productId,
            name: item.name,
            price: item.price.amount,
            quantity: item.quantity,
          })),
          updatedAt: new Date(),
        },
        $setOnInsert: { createdAt: new Date() },
      },
      { upsert: true },
    );
  }

  async findById(id) {
    const doc = await this.#collection.findOne({ _id: id });
    return doc ? this.#hydrate(doc) : null;
  }

  async findByCustomerId(customerId) {
    const docs = await this.#collection.find({ customerId }).toArray();
    return docs.map(doc => this.#hydrate(doc));
  }

  async nextIdentity() {
    return randomUUID();
  }

  #hydrate(doc) {
    const order = new Order(doc._id, doc.customerId);
    for (const item of doc.items) {
      order.addItem(item.productId, item.name, new Money(item.price), item.quantity);
    }
    return order;
  }
}
```

---

## 🔥 Complex Implementation Scenarios

### ১. সম্পূর্ণ Application Service — Use Case Implementation

Application Service হলো **orchestrator** — এটি domain objects, ports এবং adapters-কে সমন্বয় করে একটি সম্পূর্ণ use case সম্পন্ন করে।

#### PHP 8.3 — Application Service

```php
<?php

declare(strict_types=1);

namespace Application\Order;

use Domain\Order\Entity\Order;
use Domain\Order\Port\In\PlaceOrderUseCasePort;
use Domain\Order\Port\In\PlaceOrderCommand;
use Domain\Order\Port\Out\OrderRepositoryPort;
use Domain\Order\Port\Out\PaymentGatewayPort;
use Domain\Order\Port\Out\NotificationPort;
use Domain\Order\Port\Out\InventoryPort;

final readonly class PlaceOrderService implements PlaceOrderUseCasePort
{
    public function __construct(
        private OrderRepositoryPort $orderRepo,
        private PaymentGatewayPort $paymentGateway,
        private NotificationPort $notification,
        private InventoryPort $inventory,
    ) {}

    public function execute(PlaceOrderCommand $command): OrderResult
    {
        // ১. নতুন অর্ডার তৈরি
        $orderId = $this->orderRepo->nextIdentity();
        $order = new Order($orderId, $command->customerId);

        // ২. আইটেম যোগ ও ইনভেন্টরি চেক
        foreach ($command->items as $item) {
            $available = $this->inventory->checkAvailability(
                $item->productId,
                $item->quantity,
            );

            if (!$available) {
                throw new InsufficientStockException(
                    "পণ্য {$item->productId}-এর পর্যাপ্ত স্টক নেই"
                );
            }

            $order->addItem(
                $item->productId,
                $item->name,
                new Money($item->price),
                $item->quantity,
            );
        }

        // ৩. পেমেন্ট প্রক্রিয়াকরণ
        $paymentResult = $this->paymentGateway->charge(
            $orderId,
            $order->getTotal(),
            $command->paymentMethod,
        );

        if (!$paymentResult->success) {
            throw new PaymentFailedException(
                "পেমেন্ট ব্যর্থ: {$paymentResult->errorMessage}"
            );
        }

        // ৪. ইনভেন্টরি reserve
        foreach ($command->items as $item) {
            $this->inventory->reserve($item->productId, $item->quantity, $orderId);
        }

        // ৫. অর্ডার confirm ও সংরক্ষণ
        $order->confirm();
        $this->orderRepo->save($order);

        // ৬. গ্রাহককে জানানো
        $this->notification->notifyOrderConfirmed(
            $command->customerId,
            $orderId,
            $order->getTotal(),
        );

        return new OrderResult(
            orderId: $orderId,
            total: $order->getTotal(),
            status: $order->getStatus()->value,
        );
    }
}
```

#### JavaScript (Node.js) — Application Service

```javascript
// application/order/PlaceOrderService.js

export class PlaceOrderService extends PlaceOrderUseCasePort {
  #orderRepo;
  #paymentGateway;
  #notification;
  #inventory;

  constructor(orderRepo, paymentGateway, notification, inventory) {
    super();
    this.#orderRepo = orderRepo;
    this.#paymentGateway = paymentGateway;
    this.#notification = notification;
    this.#inventory = inventory;
  }

  async execute(command) {
    // ১. নতুন অর্ডার তৈরি
    const orderId = await this.#orderRepo.nextIdentity();
    const order = new Order(orderId, command.customerId);

    // ২. আইটেম যোগ ও ইনভেন্টরি চেক
    for (const item of command.items) {
      const available = await this.#inventory.checkAvailability(
        item.productId,
        item.quantity,
      );

      if (!available) {
        throw new InsufficientStockError(`পণ্য ${item.productId}-এর পর্যাপ্ত স্টক নেই`);
      }

      order.addItem(item.productId, item.name, new Money(item.price), item.quantity);
    }

    // ৩. পেমেন্ট প্রক্রিয়াকরণ
    const paymentResult = await this.#paymentGateway.charge(
      orderId,
      order.total,
      command.paymentMethod,
    );

    if (!paymentResult.success) {
      throw new PaymentFailedError(`পেমেন্ট ব্যর্থ: ${paymentResult.errorMessage}`);
    }

    // ৪. ইনভেন্টরি reserve
    for (const item of command.items) {
      await this.#inventory.reserve(item.productId, item.quantity, orderId);
    }

    // ৫. অর্ডার confirm ও সংরক্ষণ
    order.confirm();
    await this.#orderRepo.save(order);

    // ৬. গ্রাহককে জানানো
    await this.#notification.notifyOrderConfirmed(
      command.customerId,
      orderId,
      order.total,
    );

    return { orderId, total: order.total, status: order.status.value };
  }
}
```

---

### ২. Swappable Adapters — অ্যাডাপ্টার পরিবর্তনযোগ্যতা

এটি hexagonal architecture-এর **সবচেয়ে শক্তিশালী বৈশিষ্ট্য** — domain-এর একটি লাইনও পরিবর্তন না করে adapter বদলানো যায়।

#### bKash Payment Adapter (PHP)

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Payment;

use Domain\Order\Port\Out\PaymentGatewayPort;
use Domain\Order\Port\Out\PaymentResult;

final class BkashPaymentAdapter implements PaymentGatewayPort
{
    public function __construct(
        private readonly string $appKey,
        private readonly string $appSecret,
        private readonly HttpClientInterface $http,
    ) {}

    public function charge(string $orderId, Money $amount, string $method): PaymentResult
    {
        $token = $this->authenticate();

        $response = $this->http->post('https://tokenized.sandbox.bka.sh/v1.2.0-beta/checkout/payment/create', [
            'headers' => ['Authorization' => $token, 'X-APP-Key' => $this->appKey],
            'json' => [
                'mode' => '0011',
                'payerReference' => $orderId,
                'callbackURL' => config('app.url') . '/api/bkash/callback',
                'amount' => $amount->amount,
                'currency' => 'BDT',
                'intent' => 'sale',
                'merchantInvoiceNumber' => $orderId,
            ],
        ]);

        $data = json_decode($response->getBody()->getContents(), true);

        return new PaymentResult(
            success: $data['statusCode'] === '0000',
            transactionId: $data['paymentID'] ?? null,
            errorMessage: $data['statusMessage'] ?? null,
        );
    }

    public function refund(string $transactionId, Money $amount): RefundResult
    {
        // bKash refund implementation
        $token = $this->authenticate();
        $response = $this->http->post('https://tokenized.sandbox.bka.sh/v1.2.0-beta/checkout/payment/refund', [
            'headers' => ['Authorization' => $token, 'X-APP-Key' => $this->appKey],
            'json' => [
                'paymentID' => $transactionId,
                'amount' => $amount->amount,
                'trxID' => $transactionId,
                'sku' => 'refund',
                'reason' => 'Customer requested refund',
            ],
        ]);

        $data = json_decode($response->getBody()->getContents(), true);
        return new RefundResult($data['statusCode'] === '0000', $data['statusMessage'] ?? null);
    }

    private function authenticate(): string
    {
        // bKash token generation
        $response = $this->http->post('https://tokenized.sandbox.bka.sh/v1.2.0-beta/checkout/token/grant', [
            'json' => [
                'app_key' => $this->appKey,
                'app_secret' => $this->appSecret,
            ],
        ]);

        return json_decode($response->getBody()->getContents(), true)['id_token'];
    }
}
```

#### Nagad Payment Adapter (PHP) — একই port, ভিন্ন adapter

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Payment;

final class NagadPaymentAdapter implements PaymentGatewayPort
{
    public function __construct(
        private readonly string $merchantId,
        private readonly string $merchantPrivateKey,
        private readonly HttpClientInterface $http,
    ) {}

    public function charge(string $orderId, Money $amount, string $method): PaymentResult
    {
        $timestamp = now()->format('YmdHis');
        $sensitiveData = $this->encryptData([
            'merchantId' => $this->merchantId,
            'datetime' => $timestamp,
            'orderId' => $orderId,
            'challenge' => bin2hex(random_bytes(16)),
        ]);

        $response = $this->http->post("https://api.mynagad.com/api/dfs/check-out/initialize/{$this->merchantId}/{$orderId}", [
            'json' => [
                'accountNumber' => $this->merchantId,
                'dateTime' => $timestamp,
                'sensitiveData' => $sensitiveData,
                'signature' => $this->generateSignature($sensitiveData),
            ],
        ]);

        $data = json_decode($response->getBody()->getContents(), true);

        return new PaymentResult(
            success: ($data['status'] ?? '') === 'Success',
            transactionId: $data['paymentRefId'] ?? null,
            errorMessage: $data['message'] ?? null,
        );
    }

    public function refund(string $transactionId, Money $amount): RefundResult
    {
        // Nagad refund implementation
        // ...
        return new RefundResult(true);
    }

    private function encryptData(array $data): string
    {
        // Nagad-specific encryption
        return base64_encode(json_encode($data));
    }

    private function generateSignature(string $data): string
    {
        openssl_sign($data, $signature, $this->merchantPrivateKey, OPENSSL_ALGO_SHA256);
        return base64_encode($signature);
    }
}
```

#### JavaScript (Node.js) — bKash ও Nagad Adapters

```javascript
// infrastructure/payment/BkashPaymentAdapter.js

export class BkashPaymentAdapter extends PaymentGatewayPort {
  #appKey;
  #appSecret;
  #baseUrl;

  constructor(appKey, appSecret, sandbox = true) {
    super();
    this.#appKey = appKey;
    this.#appSecret = appSecret;
    this.#baseUrl = sandbox
      ? 'https://tokenized.sandbox.bka.sh/v1.2.0-beta'
      : 'https://tokenized.pay.bka.sh/v1.2.0-beta';
  }

  async charge(orderId, amount, method) {
    const token = await this.#authenticate();

    const response = await fetch(`${this.#baseUrl}/checkout/payment/create`, {
      method: 'POST',
      headers: {
        'Authorization': token,
        'X-APP-Key': this.#appKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        mode: '0011',
        payerReference: orderId,
        amount: amount.amount.toString(),
        currency: 'BDT',
        intent: 'sale',
        merchantInvoiceNumber: orderId,
      }),
    });

    const data = await response.json();
    return {
      success: data.statusCode === '0000',
      transactionId: data.paymentID ?? null,
      errorMessage: data.statusMessage ?? null,
    };
  }

  async refund(transactionId, amount) {
    const token = await this.#authenticate();
    // bKash refund implementation
    return { success: true };
  }

  async #authenticate() {
    const response = await fetch(`${this.#baseUrl}/checkout/token/grant`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ app_key: this.#appKey, app_secret: this.#appSecret }),
    });
    const data = await response.json();
    return data.id_token;
  }
}

// infrastructure/payment/NagadPaymentAdapter.js

export class NagadPaymentAdapter extends PaymentGatewayPort {
  #merchantId;
  #merchantKey;

  constructor(merchantId, merchantKey) {
    super();
    this.#merchantId = merchantId;
    this.#merchantKey = merchantKey;
  }

  async charge(orderId, amount, method) {
    const timestamp = new Date().toISOString().replace(/[-:T.Z]/g, '').slice(0, 14);

    const response = await fetch(
      `https://api.mynagad.com/api/dfs/check-out/initialize/${this.#merchantId}/${orderId}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          accountNumber: this.#merchantId,
          dateTime: timestamp,
          sensitiveData: this.#encrypt({ merchantId: this.#merchantId, orderId }),
        }),
      },
    );

    const data = await response.json();
    return {
      success: data.status === 'Success',
      transactionId: data.paymentRefId ?? null,
      errorMessage: data.message ?? null,
    };
  }

  async refund(transactionId, amount) {
    return { success: true };
  }

  #encrypt(data) {
    return Buffer.from(JSON.stringify(data)).toString('base64');
  }
}
```

#### Adapter Swapping — Dependency Injection দিয়ে পরিবর্তন

```php
<?php
// Laravel Service Provider — এখানে adapter bind করা হয়

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

final class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // শুধু এই একটি লাইন বদলালে পুরো payment system বদলে যায়!
        $this->app->bind(
            PaymentGatewayPort::class,
            fn() => match (config('payment.default')) {
                'bkash' => new BkashPaymentAdapter(
                    config('payment.bkash.app_key'),
                    config('payment.bkash.app_secret'),
                    app(HttpClientInterface::class),
                ),
                'nagad' => new NagadPaymentAdapter(
                    config('payment.nagad.merchant_id'),
                    config('payment.nagad.private_key'),
                    app(HttpClientInterface::class),
                ),
                'stripe' => new StripePaymentAdapter(config('payment.stripe.secret')),
                default => throw new \RuntimeException('Unknown payment gateway'),
            },
        );
    }
}
```

```javascript
// infrastructure/di/container.js (Node.js)

export function createContainer(config) {
  const paymentAdapter = (() => {
    switch (config.payment.default) {
      case 'bkash':
        return new BkashPaymentAdapter(config.payment.bkash.appKey, config.payment.bkash.appSecret);
      case 'nagad':
        return new NagadPaymentAdapter(config.payment.nagad.merchantId, config.payment.nagad.key);
      default:
        throw new Error(`Unknown payment gateway: ${config.payment.default}`);
    }
  })();

  const orderRepo = config.database.driver === 'mongodb'
    ? new MongoOrderRepository(mongoDb)
    : new MySQLOrderRepository(mysqlPool);

  const notification = new CompositeNotificationAdapter([
    new EmailNotificationAdapter(config.mail),
    new SMSNotificationAdapter(config.sms),
  ]);

  const placeOrderService = new PlaceOrderService(
    orderRepo,
    paymentAdapter,
    notification,
    new InventoryServiceAdapter(config.inventory.url),
  );

  return { placeOrderService, orderRepo };
}
```

---

### ৩. টেস্টিং — Hexagonal Architecture-র সবচেয়ে বড় সুবিধা

Hexagonal architecture-র কারণে **domain logic সম্পূর্ণ আলাদাভাবে test করা যায়** — কোনো database, API বা external service লাগে না।

#### PHP (PHPUnit) — In-Memory Adapter দিয়ে টেস্টিং

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

// In-Memory Adapter — শুধু টেস্টিং-এর জন্য
final class InMemoryOrderRepository implements OrderRepositoryPort
{
    /** @var array<string, Order> */
    private array $orders = [];
    private int $sequence = 0;

    public function save(Order $order): void
    {
        $this->orders[$order->id] = $order;
    }

    public function findById(string $id): ?Order
    {
        return $this->orders[$id] ?? null;
    }

    public function findByCustomerId(string $customerId): array
    {
        return array_values(array_filter(
            $this->orders,
            fn(Order $o) => $o->customerId === $customerId,
        ));
    }

    public function nextIdentity(): string
    {
        return 'order-' . (++$this->sequence);
    }
}

final class FakePaymentGateway implements PaymentGatewayPort
{
    private bool $shouldFail = false;

    public function willFail(): void { $this->shouldFail = true; }

    public function charge(string $orderId, Money $amount, string $method): PaymentResult
    {
        if ($this->shouldFail) {
            return new PaymentResult(false, null, 'পেমেন্ট ব্যর্থ হয়েছে');
        }
        return new PaymentResult(true, 'txn-' . $orderId);
    }

    public function refund(string $transactionId, Money $amount): RefundResult
    {
        return new RefundResult(true);
    }
}

final class SpyNotification implements NotificationPort
{
    public array $notifications = [];

    public function notifyOrderConfirmed(string $customerId, string $orderId, Money $total): void
    {
        $this->notifications[] = [
            'type' => 'order_confirmed',
            'customerId' => $customerId,
            'orderId' => $orderId,
            'total' => $total,
        ];
    }

    public function notifyOrderShipped(string $customerId, string $orderId, string $tracking): void
    {
        $this->notifications[] = [
            'type' => 'order_shipped',
            'customerId' => $customerId,
            'orderId' => $orderId,
            'tracking' => $tracking,
        ];
    }
}

// ═══════════ আসল টেস্ট ═══════════

final class PlaceOrderServiceTest extends TestCase
{
    private InMemoryOrderRepository $orderRepo;
    private FakePaymentGateway $paymentGateway;
    private SpyNotification $notification;
    private StubInventory $inventory;
    private PlaceOrderService $service;

    protected function setUp(): void
    {
        $this->orderRepo = new InMemoryOrderRepository();
        $this->paymentGateway = new FakePaymentGateway();
        $this->notification = new SpyNotification();
        $this->inventory = new StubInventory(alwaysAvailable: true);

        $this->service = new PlaceOrderService(
            $this->orderRepo,
            $this->paymentGateway,
            $this->notification,
            $this->inventory,
        );
    }

    public function test_successfully_places_order(): void
    {
        $command = new PlaceOrderCommand(
            customerId: 'cust-1',
            items: [
                new OrderItemDTO('prod-1', 'ঢাকাই জামদানি শাড়ি', 5000.00, 2),
                new OrderItemDTO('prod-2', 'রাজশাহী সিল্ক', 3000.00, 1),
            ],
            shippingAddress: 'ধানমন্ডি ২৭, ঢাকা ১২০৫',
            paymentMethod: 'bkash',
        );

        $result = $this->service->execute($command);

        // অর্ডার সফলভাবে তৈরি হয়েছে
        $this->assertNotEmpty($result->orderId);
        $this->assertEquals('confirmed', $result->status);
        $this->assertEquals(13000.00, $result->total->amount);

        // অর্ডার database-এ আছে
        $savedOrder = $this->orderRepo->findById($result->orderId);
        $this->assertNotNull($savedOrder);
        $this->assertCount(2, $savedOrder->getItems());

        // গ্রাহককে notify করা হয়েছে
        $this->assertCount(1, $this->notification->notifications);
        $this->assertEquals('order_confirmed', $this->notification->notifications[0]['type']);
    }

    public function test_fails_when_payment_fails(): void
    {
        $this->paymentGateway->willFail();

        $command = new PlaceOrderCommand(
            customerId: 'cust-1',
            items: [new OrderItemDTO('prod-1', 'টেস্ট পণ্য', 500.00, 1)],
            shippingAddress: 'গুলশান, ঢাকা',
            paymentMethod: 'bkash',
        );

        $this->expectException(PaymentFailedException::class);
        $this->expectExceptionMessage('পেমেন্ট ব্যর্থ');

        $this->service->execute($command);
    }

    public function test_fails_when_insufficient_stock(): void
    {
        $this->inventory = new StubInventory(alwaysAvailable: false);
        $this->service = new PlaceOrderService(
            $this->orderRepo, $this->paymentGateway, $this->notification, $this->inventory,
        );

        $command = new PlaceOrderCommand(
            customerId: 'cust-1',
            items: [new OrderItemDTO('prod-1', 'স্টক নেই পণ্য', 500.00, 100)],
            shippingAddress: 'মিরপুর, ঢাকা',
            paymentMethod: 'nagad',
        );

        $this->expectException(InsufficientStockException::class);
        $this->service->execute($command);
    }
}
```

#### JavaScript (Jest) — In-Memory Adapter দিয়ে টেস্টিং

```javascript
// tests/unit/PlaceOrderService.test.js

class InMemoryOrderRepository extends OrderRepositoryPort {
  #orders = new Map();
  #sequence = 0;

  async save(order) { this.#orders.set(order.id, order); }
  async findById(id) { return this.#orders.get(id) ?? null; }
  async findByCustomerId(cid) {
    return [...this.#orders.values()].filter(o => o.customerId === cid);
  }
  async nextIdentity() { return `order-${++this.#sequence}`; }

  get savedOrders() { return [...this.#orders.values()]; }
}

class FakePaymentGateway extends PaymentGatewayPort {
  #shouldFail = false;
  willFail() { this.#shouldFail = true; }

  async charge(orderId, amount, method) {
    if (this.#shouldFail) return { success: false, transactionId: null, errorMessage: 'ব্যর্থ' };
    return { success: true, transactionId: `txn-${orderId}`, errorMessage: null };
  }

  async refund() { return { success: true }; }
}

class SpyNotification extends NotificationPort {
  calls = [];
  async notifyOrderConfirmed(customerId, orderId, total) {
    this.calls.push({ type: 'confirmed', customerId, orderId, total });
  }
  async notifyOrderShipped(customerId, orderId, tracking) {
    this.calls.push({ type: 'shipped', customerId, orderId, tracking });
  }
}

describe('PlaceOrderService', () => {
  let orderRepo, paymentGateway, notification, inventory, service;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    paymentGateway = new FakePaymentGateway();
    notification = new SpyNotification();
    inventory = { checkAvailability: async () => true, reserve: async () => {}, release: async () => {} };

    service = new PlaceOrderService(orderRepo, paymentGateway, notification, inventory);
  });

  test('সফলভাবে অর্ডার তৈরি করে', async () => {
    const result = await service.execute({
      customerId: 'cust-1',
      items: [
        { productId: 'prod-1', name: 'পাঞ্জাবি', price: 2500, quantity: 2 },
        { productId: 'prod-2', name: 'পায়জামা', price: 800, quantity: 2 },
      ],
      shippingAddress: 'বনানী, ঢাকা',
      paymentMethod: 'bkash',
    });

    expect(result.orderId).toBeDefined();
    expect(result.status).toBe('confirmed');
    expect(result.total.amount).toBe(6600);

    const saved = await orderRepo.findById(result.orderId);
    expect(saved).not.toBeNull();
    expect(saved.items).toHaveLength(2);

    expect(notification.calls).toHaveLength(1);
    expect(notification.calls[0].type).toBe('confirmed');
  });

  test('পেমেন্ট ব্যর্থ হলে exception throw করে', async () => {
    paymentGateway.willFail();

    await expect(
      service.execute({
        customerId: 'cust-1',
        items: [{ productId: 'p1', name: 'টেস্ট', price: 100, quantity: 1 }],
        shippingAddress: 'ঢাকা',
        paymentMethod: 'bkash',
      }),
    ).rejects.toThrow('পেমেন্ট ব্যর্থ');
  });

  test('স্টক না থাকলে exception throw করে', async () => {
    inventory.checkAvailability = async () => false;

    await expect(
      service.execute({
        customerId: 'cust-1',
        items: [{ productId: 'p1', name: 'টেস্ট', price: 100, quantity: 1 }],
        shippingAddress: 'ঢাকা',
        paymentMethod: 'nagad',
      }),
    ).rejects.toThrow('স্টক নেই');
  });
});
```

---

### ৪. Hexagonal + DDD (Domain-Driven Design)

Hexagonal architecture এবং DDD একসাথে অসাধারণভাবে কাজ করে। প্রতিটি **Bounded Context** একটি আলাদা hexagon হিসেবে কাজ করতে পারে।

```
   ┌─────────────────────┐      ┌─────────────────────┐
   │   🛒 Order Context  │      │  💰 Payment Context │
   │   ╭──────────────╮  │      │  ╭──────────────╮   │
   │   │   Order       │  │◄────►│  │   Payment    │   │
   │   │   OrderItem   │  │ ACL  │  │   Transaction│   │
   │   │   OrderStatus │  │      │  │   Refund     │   │
   │   ╰──────────────╯  │      │  ╰──────────────╯   │
   └─────────────────────┘      └─────────────────────┘
              │                            │
              ▼                            ▼
   ┌─────────────────────┐      ┌─────────────────────┐
   │  📦 Inventory Ctx   │      │  📧 Notification Ctx │
   │   ╭──────────────╮  │      │  ╭──────────────╮   │
   │   │   Product     │  │      │  │   Template   │   │
   │   │   Stock       │  │      │  │   Channel    │   │
   │   │   Warehouse   │  │      │  │   Delivery   │   │
   │   ╰──────────────╯  │      │  ╰──────────────╯   │
   └─────────────────────┘      └─────────────────────┘

   ACL = Anti-Corruption Layer (context-এর মধ্যে অনুবাদক)
```

#### Aggregate Root + Domain Events (PHP)

```php
<?php

declare(strict_types=1);

namespace Domain\Order\Entity;

// Aggregate Root — বাইরে থেকে শুধু এর মাধ্যমে Order context-এ প্রবেশ করতে হবে
final class Order
{
    private array $items = [];
    private array $domainEvents = [];

    private function __construct(
        public readonly OrderId $id,
        public readonly CustomerId $customerId,
        private OrderStatus $status,
        private Money $total,
        private readonly \DateTimeImmutable $createdAt,
    ) {}

    // Named constructor — Aggregate তৈরির একমাত্র উপায়
    public static function place(
        OrderId $id,
        CustomerId $customerId,
        array $items,
    ): self {
        if (empty($items)) {
            throw new EmptyOrderException();
        }

        $order = new self(
            $id,
            $customerId,
            OrderStatus::Pending,
            new Money(0),
            new \DateTimeImmutable(),
        );

        foreach ($items as $item) {
            $order->addItem($item);
        }

        // Domain Event — order তৈরি হওয়ার ঘটনা record করা
        $order->domainEvents[] = new OrderPlacedEvent(
            orderId: $id,
            customerId: $customerId,
            total: $order->total,
            occurredAt: new \DateTimeImmutable(),
        );

        return $order;
    }

    public function confirm(PaymentConfirmation $payment): void
    {
        $this->guardTransition(OrderStatus::Confirmed);
        $this->status = OrderStatus::Confirmed;

        $this->domainEvents[] = new OrderConfirmedEvent(
            orderId: $this->id,
            paymentId: $payment->transactionId,
            occurredAt: new \DateTimeImmutable(),
        );
    }

    public function ship(TrackingNumber $tracking): void
    {
        $this->guardTransition(OrderStatus::Shipped);
        $this->status = OrderStatus::Shipped;

        $this->domainEvents[] = new OrderShippedEvent(
            orderId: $this->id,
            customerId: $this->customerId,
            trackingNumber: $tracking,
            occurredAt: new \DateTimeImmutable(),
        );
    }

    private function guardTransition(OrderStatus $to): void
    {
        if (!$this->status->canTransitionTo($to)) {
            throw new InvalidOrderTransitionException($this->status, $to);
        }
    }

    private function addItem(OrderItemData $data): void
    {
        $item = new OrderItem($data->productId, $data->name, $data->price, $data->quantity);
        $this->items[] = $item;
        $this->total = $this->total->add($item->getSubtotal());
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

---

### ৫. Hexagonal + CQRS

CQRS (Command Query Responsibility Segregation) hexagonal architecture-র সাথে চমৎকারভাবে মিলে যায় — **ভিন্ন command ও query port, ভিন্ন adapter**।

```
    ╔═══════════════════════════════════════════════════╗
    ║              HEXAGONAL + CQRS                      ║
    ╠═══════════════════════════════════════════════════╣
    ║                                                    ║
    ║  COMMAND SIDE              QUERY SIDE              ║
    ║  ┌──────────┐              ┌──────────┐            ║
    ║  │ POST /api│              │ GET /api │            ║
    ║  │ (Write)  │              │ (Read)   │            ║
    ║  └────┬─────┘              └────┬─────┘            ║
    ║       │                         │                  ║
    ║  ┌────▼──────────┐    ┌────────▼──────────┐       ║
    ║  │ Command Port  │    │   Query Port      │       ║
    ║  │ PlaceOrder    │    │   GetOrder         │       ║
    ║  │ CancelOrder   │    │   ListOrders       │       ║
    ║  └────┬──────────┘    └────────┬──────────┘       ║
    ║       │                         │                  ║
    ║  ┌────▼──────────┐    ┌────────▼──────────┐       ║
    ║  │  DOMAIN       │    │   Read Model      │       ║
    ║  │  (Entities)   │──→ │   (Projections)   │       ║
    ║  └────┬──────────┘    └────────┬──────────┘       ║
    ║       │   Domain Events         │                  ║
    ║  ┌────▼──────────┐    ┌────────▼──────────┐       ║
    ║  │ Write DB Port │    │  Read DB Port     │       ║
    ║  │ (MySQL/PG)    │    │  (ElasticSearch)  │       ║
    ║  └───────────────┘    └───────────────────┘       ║
    ╚═══════════════════════════════════════════════════╝
```

#### PHP — CQRS Ports

```php
<?php

// Command Ports (Write side)
interface PlaceOrderCommandPort
{
    public function handle(PlaceOrderCommand $command): string;
}

interface CancelOrderCommandPort
{
    public function handle(CancelOrderCommand $command): void;
}

// Query Ports (Read side) — সম্পূর্ণ আলাদা, domain entity return করে না
interface OrderQueryPort
{
    public function findById(string $id): ?OrderReadModel;
    public function findByCustomer(string $customerId, OrderFilters $filters): PaginatedResult;
    public function searchOrders(string $query): array;
}

// Read Model — শুধু display-এর জন্য optimized, কোনো business logic নেই
final readonly class OrderReadModel
{
    public function __construct(
        public string $id,
        public string $customerName,
        public string $status,
        public float $totalAmount,
        public string $currency,
        public int $itemCount,
        public string $formattedDate,
        public ?string $trackingNumber,
    ) {}
}

// Query Adapter — ElasticSearch ব্যবহার করে দ্রুত search
final class ElasticSearchOrderQueryAdapter implements OrderQueryPort
{
    public function __construct(
        private readonly Client $elastic,
    ) {}

    public function findById(string $id): ?OrderReadModel
    {
        $result = $this->elastic->get(['index' => 'orders', 'id' => $id]);
        return $result['found'] ? $this->mapToReadModel($result['_source']) : null;
    }

    public function searchOrders(string $query): array
    {
        $results = $this->elastic->search([
            'index' => 'orders',
            'body' => [
                'query' => [
                    'multi_match' => [
                        'query' => $query,
                        'fields' => ['customerName', 'items.name', 'id'],
                    ],
                ],
            ],
        ]);

        return array_map(
            fn($hit) => $this->mapToReadModel($hit['_source']),
            $results['hits']['hits'],
        );
    }
}
```

---

### ৬. Multi-Adapter Notification System

একই port-এ একাধিক adapter — পরিস্থিতি অনুযায়ী সঠিক adapter বেছে নেওয়া।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Notification;

// Composite Adapter — একাধিক adapter একসাথে ব্যবহার
final readonly class CompositeNotificationAdapter implements NotificationPort
{
    /**
     * @param NotificationChannel[] $channels
     */
    public function __construct(
        private array $channels,
        private NotificationStrategyPort $strategy,
    ) {}

    public function notifyOrderConfirmed(string $customerId, Money $total): void
    {
        $selectedChannels = $this->strategy->selectChannels(
            event: 'order_confirmed',
            customerId: $customerId,
        );

        foreach ($selectedChannels as $channelName) {
            $channel = $this->findChannel($channelName);
            $channel?->send($customerId, "আপনার অর্ডার নিশ্চিত হয়েছে! মোট: ৳{$total->amount}");
        }
    }
}

// প্রতিটি channel একটি আলাদা adapter
final readonly class EmailNotificationChannel implements NotificationChannel
{
    public function __construct(private MailerInterface $mailer) {}

    public function name(): string { return 'email'; }

    public function send(string $customerId, string $message): void
    {
        $email = $this->resolveEmail($customerId);
        $this->mailer->send($email, 'অর্ডার আপডেট', $message);
    }
}

final readonly class SMSNotificationChannel implements NotificationChannel
{
    public function __construct(private SMSClientInterface $smsClient) {}

    public function name(): string { return 'sms'; }

    public function send(string $customerId, string $message): void
    {
        $phone = $this->resolvePhone($customerId);
        $this->smsClient->send($phone, $message);
    }
}

final readonly class SlackNotificationChannel implements NotificationChannel
{
    public function __construct(private SlackClient $slack, private string $channel) {}

    public function name(): string { return 'slack'; }

    public function send(string $customerId, string $message): void
    {
        $this->slack->postMessage($this->channel, "🛒 Customer {$customerId}: {$message}");
    }
}

// Strategy — কোন পরিস্থিতিতে কোন channel ব্যবহার হবে
final readonly class PriorityBasedNotificationStrategy implements NotificationStrategyPort
{
    public function selectChannels(string $event, string $customerId): array
    {
        return match ($event) {
            'order_confirmed' => ['email', 'sms'],              // দুইটাই পাঠাও
            'order_shipped' => ['sms', 'push'],                  // দ্রুত জানানো দরকার
            'order_delivered' => ['email'],                       // confirmation যথেষ্ট
            'payment_failed' => ['sms', 'email', 'push'],       // সব channel দিয়ে জানাও
            default => ['email'],
        };
    }
}
```

#### JavaScript (Node.js)

```javascript
// infrastructure/notification/CompositeNotificationAdapter.js

export class CompositeNotificationAdapter extends NotificationPort {
  #channels;
  #strategy;

  constructor(channels, strategy) {
    super();
    this.#channels = new Map(channels.map(ch => [ch.name, ch]));
    this.#strategy = strategy;
  }

  async notifyOrderConfirmed(customerId, orderId, total) {
    const selected = this.#strategy.selectChannels('order_confirmed', customerId);
    const message = `আপনার অর্ডার #${orderId} নিশ্চিত হয়েছে! মোট: ৳${total.amount}`;

    await Promise.allSettled(
      selected
        .map(name => this.#channels.get(name))
        .filter(Boolean)
        .map(channel => channel.send(customerId, message)),
    );
  }

  async notifyOrderShipped(customerId, orderId, trackingNumber) {
    const selected = this.#strategy.selectChannels('order_shipped', customerId);
    const message = `আপনার অর্ডার #${orderId} শিপ হয়েছে! ট্র্যাকিং: ${trackingNumber}`;

    await Promise.allSettled(
      selected.map(name => this.#channels.get(name)).filter(Boolean)
        .map(ch => ch.send(customerId, message)),
    );
  }
}

// Channels
export class EmailChannel {
  name = 'email';
  #transporter;

  constructor(transporter) { this.#transporter = transporter; }

  async send(customerId, message) {
    const email = await this.#resolveEmail(customerId);
    await this.#transporter.sendMail({
      to: email, subject: 'অর্ডার আপডেট', text: message,
    });
  }
}

export class SMSChannel {
  name = 'sms';
  #client;

  constructor(smsClient) { this.#client = smsClient; }

  async send(customerId, message) {
    const phone = await this.#resolvePhone(customerId);
    await this.#client.send(phone, message);
  }
}
```

---

## 🆚 Hexagonal vs Other Architectures

| বৈশিষ্ট্য | Hexagonal | Clean Architecture | Onion Architecture | Traditional Layered |
|-----------|-----------|-------------------|-------------------|-------------------|
| **প্রবর্তক** | Alistair Cockburn (2005) | Robert C. Martin (2012) | Jeffrey Palermo (2008) | Traditional |
| **মূল ধারণা** | Ports & Adapters | Dependency Rule, Use Cases | Layers around domain | Sequential layers |
| **Layer সংখ্যা** | সংজ্ঞায়িত নয় | ৪ (Entity, UC, Interface, FW) | ৩+ concentric rings | ৩-৪ |
| **Dependency Rule** | বাইরে → ভেতরে | বাইরে → ভেতরে | বাইরে → ভেতরে | উপরে → নিচে |
| **Symmetry** | ✅ Driving/Driven সমান | ❌ Input/Output আলাদা | ❌ আলাদা layers | ❌ একমুখী |
| **Port ধারণা** | ✅ স্পষ্ট | ❌ Boundary হিসেবে | ❌ Interface হিসেবে | ❌ নেই |
| **Testability** | ✅ চমৎকার | ✅ চমৎকার | ✅ ভালো | ⚠️ মাঝামাঝি |
| **জটিলতা** | মাঝামাঝি | বেশি | মাঝামাঝি | কম |
| **Framework Freedom** | ✅ সম্পূর্ণ | ✅ সম্পূর্ণ | ✅ প্রায় সম্পূর্ণ | ❌ tightly coupled |

### মূল পার্থক্য

```
 Hexagonal:               Clean:                   Onion:                Layered:
 ┌───────────┐            ┌───────────┐            ┌───────────┐        ┌───────────┐
 │ Adapter   │            │ Framework │            │ Infra     │        │ UI        │
 │  ┌─────┐  │            │  ┌─────┐  │            │  ┌─────┐  │        ├───────────┤
 │  │Port │  │            │  │I.Adp│  │            │  │App   │  │        │ Business  │
 │  │┌───┐│  │            │  │┌───┐│  │            │  │┌───┐│  │        ├───────────┤
 │  ││Dom││  │            │  ││UC ││  │            │  ││Dom││  │        │ Data      │
 │  │└───┘│  │            │  │└───┘│  │            │  │└───┘│  │        └───────────┘
 │  └─────┘  │            │  └─────┘  │            │  └─────┘  │         ↓ ↓ ↓ ↓ ↓
 └───────────┘            └───────────┘            └───────────┘        একমুখী
  সব দিক সমান              ৪টি ring                concentric           নির্ভরতা
```

**Hexagonal-এর বিশেষত্ব:** এটি driving ও driven side-কে সমানভাবে দেখে। Clean Architecture আর Onion Architecture মূলত hexagonal-এর variations — মূল ধারণা একই, শুধু কাঠামো কিছুটা আলাদা।

---

## ✅ সুবিধা (Pros)

1. **Testability** — domain logic কোনো infrastructure ছাড়াই test করা যায়
2. **Flexibility** — adapter বদলানো যায় domain পরিবর্তন ছাড়া (MySQL → MongoDB, bKash → Nagad)
3. **Framework Independence** — Laravel থেকে Symfony, Express থেকে Fastify — domain অপরিবর্তিত
4. **Parallel Development** — domain team ও infrastructure team একই সময়ে কাজ করতে পারে
5. **Clear Boundaries** — কোন code কোথায় যাবে তা সুস্পষ্ট
6. **Domain Focus** — business logic-ই সবার আগে, technology পরে
7. **Multiple Entry Points** — একই domain logic REST, GraphQL, CLI, gRPC দিয়ে ব্যবহার করা যায়

## ❌ অসুবিধা (Cons)

1. **Boilerplate** — ছোট প্রজেক্টে অনেক বেশি interface ও class তৈরি করতে হয়
2. **Learning Curve** — নতুন developers-এর জন্য বোঝা কঠিন
3. **Over-Engineering** — CRUD-heavy অ্যাপ্লিকেশনে এই architecture মাত্রাতিরিক্ত
4. **Mapping Overhead** — domain ↔ persistence ↔ API response — একাধিক mapping
5. **Initial Setup Time** — প্রথমবার setup করতে বেশি সময় লাগে
6. **Performance** — অতিরিক্ত abstraction layer-এ minor performance cost

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Domain-এ Infrastructure Leak

```php
// 🚫 ভুল — Domain entity-তে Eloquent ব্যবহার
class Order extends Model  // ← Eloquent Model!
{
    protected $fillable = ['customer_id', 'status'];

    public function confirm(): void
    {
        $this->status = 'confirmed';
        $this->save();  // ← Database call domain-এ!
    }
}

// ✅ সঠিক — Pure domain entity
final class Order
{
    public function confirm(): void
    {
        $this->guardTransition(OrderStatus::Confirmed);
        $this->status = OrderStatus::Confirmed;
        // কোনো save() নেই — repository এটি handle করবে
    }
}
```

### ২. Anemic Domain Model

```php
// 🚫 ভুল — Entity শুধু data holder, সব logic service-এ
class Order
{
    public string $id;
    public string $status;     // public field!
    public float $total;       // কোনো validation নেই
    public array $items;
}

class OrderService
{
    public function confirm(Order $order): void
    {
        if ($order->status !== 'pending') {
            throw new \Exception('Cannot confirm');
        }
        $order->status = 'confirmed';  // বাইরে থেকে status বদলানো
    }
}

// ✅ সঠিক — Rich domain model, logic entity-র ভেতরে
final class Order
{
    private OrderStatus $status;

    public function confirm(): void
    {
        if (!$this->status->canTransitionTo(OrderStatus::Confirmed)) {
            throw new InvalidTransitionException($this->status, OrderStatus::Confirmed);
        }
        if (empty($this->items)) {
            throw new EmptyOrderException();
        }
        $this->status = OrderStatus::Confirmed;
    }
}
```

### ৩. Over-Abstraction (Port Explosion)

```php
// 🚫 ভুল — প্রতিটি ছোট কাজের জন্য আলাদা port
interface SaveOrderPort { public function save(Order $order): void; }
interface FindOrderPort { public function find(string $id): ?Order; }
interface DeleteOrderPort { public function delete(string $id): void; }
interface CountOrderPort { public function count(): int; }
interface ListOrderPort { public function list(int $page): array; }
// ৫টি আলাদা interface — unnecessary!

// ✅ সঠিক — সম্পর্কিত operations একটি port-এ
interface OrderRepositoryPort
{
    public function save(Order $order): void;
    public function findById(string $id): ?Order;
    public function delete(string $id): void;
    public function findAll(int $page = 1, int $perPage = 20): PaginatedResult;
}
```

### ৪. Adapter-এ Business Logic

```javascript
// 🚫 ভুল — Controller-এ business logic
class OrderController {
  async store(req, res) {
    const order = new Order(req.body.id, req.body.customerId);

    // ❌ Business rule controller-এ!
    if (order.total.amount > 50000) {
      await this.fraudService.check(order);
    }

    // ❌ Discount calculation controller-এ!
    if (req.body.couponCode) {
      order.applyDiscount(0.10);
    }

    await this.repo.save(order);
  }
}

// ✅ সঠিক — Controller শুধু translate করে, logic domain-এ
class OrderController {
  async store(req, res) {
    const result = await this.placeOrderUseCase.execute({
      customerId: req.body.customerId,
      items: req.body.items,
      couponCode: req.body.couponCode,
    });
    res.status(201).json(result);
  }
}
```

---

## 🧪 টেস্টিং হেক্সাগোনাল — টেস্টিং পিরামিড

```
                    ╱╲
                   ╱  ╲           E2E Tests
                  ╱ E2E╲          (কম সংখ্যক, ধীর)
                 ╱──────╲         Adapter + Domain + Infrastructure
                ╱        ╲
               ╱ Adapter  ╲      Integration Tests
              ╱  Integration╲    (মাঝামাঝি সংখ্যক)
             ╱──────────────╲    Real DB, Real API
            ╱                ╲
           ╱   Port Contract  ╲  Contract Tests
          ╱     Tests          ╲ (প্রতিটি adapter port fulfill করে কিনা)
         ╱────────────────────╲
        ╱                      ╲
       ╱    Domain Unit Tests   ╲  Unit Tests
      ╱     (সবচেয়ে বেশি, দ্রুত) ╲  In-memory adapters
     ╱──────────────────────────╲
```

### Port Contract Test — adapter সত্যিই port-এর চুক্তি মানছে কিনা

```php
<?php

// Abstract test — যেকোনো OrderRepository adapter-এর জন্য কাজ করবে
abstract class OrderRepositoryContractTest extends TestCase
{
    abstract protected function createRepository(): OrderRepositoryPort;

    public function test_can_save_and_retrieve_order(): void
    {
        $repo = $this->createRepository();
        $orderId = $repo->nextIdentity();
        $order = new Order($orderId, 'cust-1');
        $order->addItem('p-1', 'টেস্ট পণ্য', new Money(1000), 2);

        $repo->save($order);
        $found = $repo->findById($orderId);

        $this->assertNotNull($found);
        $this->assertEquals($orderId, $found->id);
        $this->assertCount(1, $found->getItems());
    }

    public function test_returns_null_for_nonexistent_order(): void
    {
        $repo = $this->createRepository();
        $this->assertNull($repo->findById('nonexistent'));
    }

    public function test_finds_orders_by_customer(): void
    {
        $repo = $this->createRepository();

        $order1 = new Order($repo->nextIdentity(), 'cust-A');
        $order1->addItem('p-1', 'পণ্য ১', new Money(500), 1);
        $repo->save($order1);

        $order2 = new Order($repo->nextIdentity(), 'cust-A');
        $order2->addItem('p-2', 'পণ্য ২', new Money(800), 1);
        $repo->save($order2);

        $order3 = new Order($repo->nextIdentity(), 'cust-B');
        $order3->addItem('p-3', 'পণ্য ৩', new Money(300), 1);
        $repo->save($order3);

        $custAOrders = $repo->findByCustomerId('cust-A');
        $this->assertCount(2, $custAOrders);
    }
}

// এখন প্রতিটি adapter এই contract test inherit করবে
final class MySQLOrderRepositoryTest extends OrderRepositoryContractTest
{
    protected function createRepository(): OrderRepositoryPort
    {
        return new MySQLOrderRepository($this->getPDOConnection());
    }
}

final class MongoOrderRepositoryTest extends OrderRepositoryContractTest
{
    protected function createRepository(): OrderRepositoryPort
    {
        return new MongoOrderRepository($this->getMongoDb());
    }
}

final class InMemoryOrderRepositoryTest extends OrderRepositoryContractTest
{
    protected function createRepository(): OrderRepositoryPort
    {
        return new InMemoryOrderRepository();
    }
}
```

```javascript
// tests/contracts/OrderRepositoryContract.js

export function orderRepositoryContract(createRepo) {
  describe('OrderRepository Contract', () => {
    let repo;
    beforeEach(() => { repo = createRepo(); });

    test('save এবং retrieve করতে পারে', async () => {
      const id = await repo.nextIdentity();
      const order = new Order(id, 'cust-1');
      order.addItem('p1', 'টেস্ট', new Money(1000), 1);

      await repo.save(order);
      const found = await repo.findById(id);

      expect(found).not.toBeNull();
      expect(found.id).toBe(id);
    });

    test('না থাকলে null return করে', async () => {
      expect(await repo.findById('nonexistent')).toBeNull();
    });
  });
}

// প্রতিটি adapter এই contract ব্যবহার করবে
orderRepositoryContract(() => new InMemoryOrderRepository());
orderRepositoryContract(() => new MongoOrderRepository(testDb));
```

---

## 📂 প্রজেক্ট স্ট্রাকচার

### PHP Laravel — Hexagonal Structure

```
app/
├── Domain/                          # 🏛️ Domain Layer (কোনো Laravel dependency নেই)
│   ├── Order/
│   │   ├── Entity/
│   │   │   ├── Order.php
│   │   │   └── OrderItem.php
│   │   ├── ValueObject/
│   │   │   ├── Money.php
│   │   │   ├── OrderId.php
│   │   │   └── OrderStatus.php
│   │   ├── Event/
│   │   │   ├── OrderPlacedEvent.php
│   │   │   ├── OrderConfirmedEvent.php
│   │   │   └── OrderCancelledEvent.php
│   │   ├── Exception/
│   │   │   ├── InsufficientStockException.php
│   │   │   └── InvalidOrderTransitionException.php
│   │   ├── Port/
│   │   │   ├── In/                  # Primary Ports
│   │   │   │   ├── PlaceOrderUseCasePort.php
│   │   │   │   ├── CancelOrderUseCasePort.php
│   │   │   │   └── GetOrderQueryPort.php
│   │   │   └── Out/                 # Secondary Ports
│   │   │       ├── OrderRepositoryPort.php
│   │   │       ├── PaymentGatewayPort.php
│   │   │       ├── NotificationPort.php
│   │   │       └── InventoryPort.php
│   │   └── Service/
│   │       └── PricingService.php
│   └── Shared/
│       ├── ValueObject/
│       └── Event/
│
├── Application/                     # 🔧 Application Layer (Use Cases)
│   └── Order/
│       ├── PlaceOrderService.php
│       ├── CancelOrderService.php
│       ├── GetOrderQueryService.php
│       ├── Command/
│       │   ├── PlaceOrderCommand.php
│       │   └── CancelOrderCommand.php
│       └── DTO/
│           ├── OrderDTO.php
│           └── OrderItemDTO.php
│
├── Infrastructure/                  # 🔌 Infrastructure Layer (Adapters)
│   ├── Http/                        # Driving Adapters
│   │   ├── Controller/
│   │   │   └── OrderController.php
│   │   ├── Request/
│   │   │   └── PlaceOrderRequest.php
│   │   └── Resource/
│   │       └── OrderResource.php
│   ├── Console/                     # Driving Adapter (CLI)
│   │   └── ProcessPendingOrdersCommand.php
│   ├── GraphQL/                     # Driving Adapter
│   │   └── OrderResolver.php
│   ├── Persistence/                 # Driven Adapters
│   │   ├── MySQL/
│   │   │   └── MySQLOrderRepository.php
│   │   ├── Mongo/
│   │   │   └── MongoOrderRepository.php
│   │   └── InMemory/
│   │       └── InMemoryOrderRepository.php
│   ├── Payment/                     # Driven Adapters
│   │   ├── BkashPaymentAdapter.php
│   │   ├── NagadPaymentAdapter.php
│   │   └── StripePaymentAdapter.php
│   ├── Notification/                # Driven Adapters
│   │   ├── EmailNotificationAdapter.php
│   │   ├── SMSNotificationAdapter.php
│   │   └── CompositeNotificationAdapter.php
│   └── Provider/
│       ├── OrderServiceProvider.php
│       └── PaymentServiceProvider.php
│
└── tests/
    ├── Unit/
    │   ├── Domain/
    │   │   ├── OrderTest.php
    │   │   └── MoneyTest.php
    │   └── Application/
    │       └── PlaceOrderServiceTest.php
    ├── Contract/
    │   └── OrderRepositoryContractTest.php
    └── Integration/
        ├── MySQLOrderRepositoryTest.php
        └── BkashPaymentAdapterTest.php
```

### Node.js — Hexagonal Structure

```
src/
├── domain/                          # 🏛️ Domain Layer
│   ├── order/
│   │   ├── entities/
│   │   │   ├── Order.js
│   │   │   └── OrderItem.js
│   │   ├── valueObjects/
│   │   │   ├── Money.js
│   │   │   ├── OrderId.js
│   │   │   └── OrderStatus.js
│   │   ├── events/
│   │   │   ├── OrderPlacedEvent.js
│   │   │   └── OrderConfirmedEvent.js
│   │   ├── errors/
│   │   │   ├── DomainError.js
│   │   │   └── InsufficientStockError.js
│   │   └── ports/
│   │       ├── in/
│   │       │   ├── PlaceOrderUseCase.js
│   │       │   └── GetOrderQuery.js
│   │       └── out/
│   │           ├── OrderRepositoryPort.js
│   │           ├── PaymentGatewayPort.js
│   │           └── NotificationPort.js
│   └── shared/
│       └── ValueObject.js
│
├── application/                     # 🔧 Application Layer
│   └── order/
│       ├── PlaceOrderService.js
│       ├── CancelOrderService.js
│       └── GetOrderQueryService.js
│
├── infrastructure/                  # 🔌 Infrastructure Layer
│   ├── http/                        # Driving
│   │   ├── controllers/
│   │   │   └── OrderController.js
│   │   ├── middleware/
│   │   │   └── validation.js
│   │   └── routes/
│   │       └── orderRoutes.js
│   ├── persistence/                 # Driven
│   │   ├── MongoOrderRepository.js
│   │   ├── MySQLOrderRepository.js
│   │   └── InMemoryOrderRepository.js
│   ├── payment/                     # Driven
│   │   ├── BkashPaymentAdapter.js
│   │   └── NagadPaymentAdapter.js
│   ├── notification/                # Driven
│   │   ├── EmailChannel.js
│   │   ├── SMSChannel.js
│   │   └── CompositeNotificationAdapter.js
│   └── di/
│       └── container.js
│
├── tests/
│   ├── unit/
│   │   ├── domain/
│   │   │   ├── Order.test.js
│   │   │   └── Money.test.js
│   │   └── application/
│   │       └── PlaceOrderService.test.js
│   ├── contracts/
│   │   └── OrderRepositoryContract.js
│   └── integration/
│       ├── MongoOrderRepository.test.js
│       └── BkashPaymentAdapter.test.js
│
├── app.js
└── server.js
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

| পরিস্থিতি | কারণ |
|-----------|------|
| **জটিল business logic** আছে | Domain layer আলাদা থাকায় logic পরিষ্কার থাকে |
| **একাধিক entry point** দরকার (REST + GraphQL + CLI) | একই domain, ভিন্ন driving adapter |
| **Infrastructure বদলানোর সম্ভাবনা** আছে | bKash থেকে Nagad, MySQL থেকে PostgreSQL |
| **দীর্ঘমেয়াদী প্রজেক্ট** | Maintainability অনেক বেশি |
| **বড় টিম** কাজ করছে | সুস্পষ্ট boundary, parallel development |
| **High test coverage** দরকার | Domain testing infrastructure ছাড়াই সম্ভব |
| **Microservices** বানাচ্ছেন | প্রতিটি service একটি hexagon |

### ❌ ব্যবহার করবেন না যখন:

| পরিস্থিতি | কারণ |
|-----------|------|
| **সহজ CRUD অ্যাপ** | Over-engineering, Laravel/Express-এর নিজস্ব structure যথেষ্ট |
| **প্রটোটাইপ বা MVP** | দ্রুত delivery প্রয়োজন, পরে refactor করা যায় |
| **ছোট টিম, ছোট প্রজেক্ট** | Boilerplate-এর overhead benefit-এর চেয়ে বেশি |
| **Read-heavy অ্যাপ** (blog, CMS) | জটিল domain logic নেই |
| **টিম অভিজ্ঞতা কম** | ভুলভাবে implement করলে আরও জটিল হয়ে যায় |

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

```
  Hexagonal Architecture
        │
        ├──→ Clean Architecture (Robert C. Martin)
        │     └── Hexagonal-এর expansion, ৪টি সুনির্দিষ্ট layer
        │
        ├──→ Onion Architecture (Jeffrey Palermo)
        │     └── Hexagonal-এর variation, concentric layers
        │
        ├──→ CQRS (Command Query Responsibility Segregation)
        │     └── Hexagonal + আলাদা read/write port ও adapter
        │
        ├──→ DDD (Domain-Driven Design)
        │     └── Hexagonal domain layer = DDD-র Bounded Context
        │
        ├──→ Event-Driven Architecture
        │     └── Domain Events port দিয়ে publish হয়
        │
        ├──→ Microservices
        │     └── প্রতিটি microservice একটি hexagon
        │
        ├──→ Repository Pattern
        │     └── Secondary port-এর সবচেয়ে সাধারণ উদাহরণ
        │
        ├──→ Dependency Injection
        │     └── Port-Adapter binding-এর মূল কৌশল
        │
        └──→ Strategy Pattern
              └── Adapter selection = Strategy Pattern
```

---

## 📋 সারসংক্ষেপ

| বিষয় | মূল কথা |
|-------|---------|
| **সংজ্ঞা** | Domain-কে infrastructure থেকে port ও adapter দিয়ে আলাদা করা |
| **মূল নিয়ম** | Dependency বাইরে → ভেতরে, কখনো ভেতর → বাইরে নয় |
| **Port** | Interface/Contract — কী করতে হবে |
| **Adapter** | Implementation — কিভাবে করতে হবে |
| **Driving** | বাইরের জগৎ → Domain (Controller, CLI, GraphQL) |
| **Driven** | Domain → বাইরের জগৎ (Database, Payment, Email) |
| **সবচেয়ে বড় সুবিধা** | Test করা সহজ, adapter বদলানো সহজ |
| **সবচেয়ে বড় সমস্যা** | Boilerplate বেশি, ছোট প্রজেক্টে overkill |
| **ব্যবহার** | জটিল business logic, একাধিক integration, বড় টিম |
| **এড়িয়ে চলুন** | সহজ CRUD, prototype, ছোট প্রজেক্ট |

> **মনে রাখবেন:** Hexagonal architecture কোনো silver bullet নয়। এটি একটি tool — সঠিক সমস্যায় ব্যবহার করলে অসাধারণ ফলাফল দেয়, ভুল জায়গায় ব্যবহার করলে অপ্রয়োজনীয় জটিলতা তৈরি করে। আপনার প্রজেক্টের আকার, টিমের অভিজ্ঞতা, এবং business logic-এর জটিলতা বিবেচনা করে সিদ্ধান্ত নিন। 🎯
