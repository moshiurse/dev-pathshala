# 🏗️ DDD Tactical Design (ট্যাক্টিক্যাল ডিজাইন)

> কোডের ভিতরে ডোমেইন মডেল কিভাবে বাস্তবায়ন করবেন — Entity, Value Object, Aggregate, Repository, Service, Factory, Specification

---

## 📖 সূচিপত্র

1. [Entities (এন্টিটি)](#-1-entities-এন্টিটি)
2. [Value Objects (ভ্যালু অবজেক্ট)](#-2-value-objects-ভ্যালু-অবজেক্ট)
3. [Aggregates (অ্যাগ্রিগেট)](#-3-aggregates-অ্যাগ্রিগেট)
4. [Repositories (রিপোজিটরি)](#-4-repositories-রিপোজিটরি)
5. [Domain Services vs Application Services](#-5-domain-services-vs-application-services)
6. [Factories (ফ্যাক্টরি)](#-6-factories-ফ্যাক্টরি)
7. [Specifications (স্পেসিফিকেশন)](#-7-specifications-স্পেসিফিকেশন-প্যাটার্ন)
8. [Complete Example — bKash-like System](#-8-সম্পূর্ণ-উদাহরণ--bkash-like-money-transfer)
9. [কখন ব্যবহার করবেন / করবেন না](#-9-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📊 Tactical Design Building Blocks — এক নজরে

```
┌─────────────────────────────────────────────────────────────┐
│                   TACTICAL DESIGN PATTERNS                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌───────────┐  ┌───────────────┐  ┌──────────────────┐   │
│   │  Entity   │  │ Value Object  │  │   Aggregate      │   │
│   │ (পরিচয়   │  │ (মান দ্বারা   │  │  (একক হিসেবে    │   │
│   │  আছে)    │  │  সংজ্ঞায়িত)  │  │   পরিচালিত)     │   │
│   └───────────┘  └───────────────┘  └──────────────────┘   │
│                                                             │
│   ┌───────────┐  ┌───────────────┐  ┌──────────────────┐   │
│   │Repository │  │Domain Service │  │    Factory        │   │
│   │ (সংগ্রহ- │  │ (ব্যবসায়িক  │  │  (জটিল অবজেক্ট  │   │
│   │  স্থল)   │  │  লজিক)       │  │   তৈরি)          │   │
│   └───────────┘  └───────────────┘  └──────────────────┘   │
│                                                             │
│   ┌───────────────────────┐  ┌─────────────────────────┐   │
│   │  Application Service  │  │    Specification        │   │
│   │  (ব্যবহার কেস        │  │  (ব্যবসায়িক নিয়ম     │   │
│   │   সমন্বয়)            │  │   অবজেক্ট হিসেবে)     │   │
│   └───────────────────────┘  └─────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📌 1. Entities (এন্টিটি)

### ধারণা

Entity হলো এমন একটি অবজেক্ট যার **নিজস্ব পরিচয় (identity)** আছে। সময়ের সাথে সাথে
এর অ্যাট্রিবিউট বদলালেও পরিচয় একই থাকে।

**বাস্তব উদাহরণ:** Pathao-তে একজন রাইডারের নাম বা ফোন নম্বর পরিবর্তন হতে পারে,
কিন্তু তার Rider ID সবসময় একই থাকবে — সে সেই একই রাইডার।

```
┌─────────────────────────────────────────┐
│              ENTITY                      │
│                                          │
│   ┌──────────┐    সময়ের সাথে বদলায়    │
│   │ ID: R001 │──► নাম: "করিম" → "আব্দুল"│
│   │ (স্থায়ী)│    ফোন: "017..." → "019.."│
│   └──────────┘                           │
│                                          │
│   সমতা (Equality) = শুধু ID দিয়ে যাচাই  │
└─────────────────────────────────────────┘
```

### ❌ Bad: পরিচয় ছাড়া Entity

```php
// ❌ খারাপ — কোনো স্পষ্ট পরিচয় নেই, সমতা যাচাই অসম্ভব
class Order {
    public string $customerName;
    public float $total;

    public function equals($other): bool {
        // কোন ID নেই, তাই নাম দিয়ে তুলনা — ভুল!
        return $this->customerName === $other->customerName;
    }
}
```

### ✅ Good: সঠিক Entity — ID-ভিত্তিক সমতা

```php
// ✅ ভালো — Entity-র নিজস্ব পরিচয় আছে
class OrderId {
    public function __construct(
        private readonly string $value
    ) {}

    public function getValue(): string {
        return $this->value;
    }

    public function equals(OrderId $other): bool {
        return $this->value === $other->value;
    }
}

class Order {
    private OrderId $id;
    private string $customerName;
    private float $total;
    private string $status;

    public function __construct(OrderId $id, string $customerName) {
        $this->id = $id;
        $this->customerName = $customerName;
        $this->total = 0;
        $this->status = 'pending';
    }

    public function getId(): OrderId {
        return $this->id;
    }

    // দুটি Order সমান কি না — শুধু ID দিয়ে যাচাই
    public function equals(Order $other): bool {
        return $this->id->equals($other->getId());
    }
}
```

### JavaScript Entity উদাহরণ

```javascript
// ✅ JS Entity — Daraz-এর প্রোডাক্ট
class ProductId {
    #value;
    constructor(value) {
        if (!value) throw new Error("ProductId খালি হতে পারে না");
        this.#value = value;
    }
    getValue() { return this.#value; }
    equals(other) { return this.#value === other.getValue(); }
}

class Product {
    #id;
    #name;
    #price;

    constructor(id, name, price) {
        this.#id = id;
        this.#name = name;
        this.#price = price;
    }

    getId() { return this.#id; }
    getName() { return this.#name; }

    // নাম বদলালেও এটি সেই একই প্রোডাক্ট
    rename(newName) {
        this.#name = newName;
    }

    equals(other) {
        return this.#id.equals(other.getId());
    }
}
```

---

## 📌 2. Value Objects (ভ্যালু অবজেক্ট)

### ধারণা

Value Object-এর কোনো পরিচয় নেই। এটি সম্পূর্ণভাবে তার **মান (attributes)** দ্বারা
সংজ্ঞায়িত। দুটি Value Object-এর সব অ্যাট্রিবিউট সমান হলে তারা সমান।
এবং এগুলো **অপরিবর্তনীয় (immutable)**।

```
┌──────────────────────────────────────────────┐
│           VALUE OBJECT বৈশিষ্ট্য             │
│                                               │
│  ✓ কোনো পরিচয় (ID) নেই                      │
│  ✓ মান দ্বারা সমতা যাচাই                      │
│  ✓ অপরিবর্তনীয় (Immutable)                   │
│  ✓ পরিবর্তন দরকার হলে নতুন অবজেক্ট তৈরি     │
│                                               │
│  Money(100, "BDT") == Money(100, "BDT") ✅     │
│  Money(100, "BDT") == Money(200, "BDT") ❌     │
└──────────────────────────────────────────────┘
```

### ❌ Bad: আদিম টাইপ দিয়ে টাকার পরিমাণ

```php
// ❌ খারাপ — float দিয়ে টাকা, মুদ্রা হারিয়ে যায়, দশমিক সমস্যা
class Payment {
    public float $amount;  // ৫০০.৫০ টাকা? ডলার? কে জানে!

    public function process(): void {
        // 0.1 + 0.2 = 0.30000000000000004 — float সমস্যা!
        $this->amount = $this->amount + 0.1 + 0.2;
    }
}
```

### ✅ Good: Money Value Object — সম্পূর্ণ বাস্তবায়ন

```php
// ✅ ভালো — Money Value Object, bKash/Nagad-এর জন্য উপযুক্ত
class Money {
    public function __construct(
        private readonly int $amountInPoisha,  // পয়সায় রাখি, দশমিক সমস্যা নেই
        private readonly string $currency
    ) {
        if ($amountInPoisha < 0) {
            throw new \InvalidArgumentException("পরিমাণ ঋণাত্মক হতে পারে না");
        }
    }

    public static function taka(int $taka, int $poisha = 0): self {
        return new self(($taka * 100) + $poisha, 'BDT');
    }

    public function add(Money $other): self {
        $this->ensureSameCurrency($other);
        return new self(
            $this->amountInPoisha + $other->amountInPoisha,
            $this->currency
        );
    }

    public function subtract(Money $other): self {
        $this->ensureSameCurrency($other);
        if ($this->amountInPoisha < $other->amountInPoisha) {
            throw new \DomainException("অপর্যাপ্ত ব্যালেন্স");
        }
        return new self(
            $this->amountInPoisha - $other->amountInPoisha,
            $this->currency
        );
    }

    public function equals(Money $other): bool {
        return $this->amountInPoisha === $other->amountInPoisha
            && $this->currency === $other->currency;
    }

    public function getAmountInPoisha(): int {
        return $this->amountInPoisha;
    }

    public function format(): string {
        $taka = intdiv($this->amountInPoisha, 100);
        $poisha = $this->amountInPoisha % 100;
        return sprintf("৳%d.%02d", $taka, $poisha);
    }

    private function ensureSameCurrency(Money $other): void {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException("মুদ্রা মিলছে না");
        }
    }
}

// ব্যবহার:
$sendAmount = Money::taka(500);
$fee = Money::taka(5);
$total = $sendAmount->add($fee); // নতুন Money অবজেক্ট তৈরি হবে
```

### JavaScript Money Value Object

```javascript
// ✅ JS Value Object — Nagad পেমেন্ট সিস্টেমের জন্য
class Money {
    #amountInPoisha;
    #currency;

    constructor(amountInPoisha, currency = 'BDT') {
        if (amountInPoisha < 0) throw new Error("ঋণাত্মক পরিমাণ অনুমোদিত নয়");
        this.#amountInPoisha = Math.floor(amountInPoisha);
        this.#currency = currency;
        Object.freeze(this); // immutable
    }

    static taka(taka, poisha = 0) {
        return new Money((taka * 100) + poisha);
    }

    add(other) {
        this.#ensureSameCurrency(other);
        return new Money(this.#amountInPoisha + other.getAmountInPoisha(), this.#currency);
    }

    subtract(other) {
        this.#ensureSameCurrency(other);
        if (this.#amountInPoisha < other.getAmountInPoisha()) {
            throw new Error("অপর্যাপ্ত ব্যালেন্স");
        }
        return new Money(this.#amountInPoisha - other.getAmountInPoisha(), this.#currency);
    }

    equals(other) {
        return this.#amountInPoisha === other.getAmountInPoisha()
            && this.#currency === other.getCurrency();
    }

    getAmountInPoisha() { return this.#amountInPoisha; }
    getCurrency() { return this.#currency; }

    #ensureSameCurrency(other) {
        if (this.#currency !== other.getCurrency()) {
            throw new Error("মুদ্রা মিলছে না");
        }
    }
}
```

### 📊 Entity vs Value Object তুলনা

```
┌─────────────────────┬───────────────────┬───────────────────┐
│      বৈশিষ্ট্য      │     Entity        │   Value Object    │
├─────────────────────┼───────────────────┼───────────────────┤
│ পরিচয় (Identity)   │ ✅ আছে (ID)       │ ❌ নেই            │
│ সমতা যাচাই         │ ID দিয়ে           │ সব মান দিয়ে      │
│ পরিবর্তনযোগ্য      │ ✅ হ্যাঁ (Mutable) │ ❌ না (Immutable) │
│ জীবনচক্র           │ স্বতন্ত্র         │ মালিকের উপর নির্ভর│
│ উদাহরণ             │ User, Order       │ Money, Address    │
│ DB-তে সংরক্ষণ      │ আলাদা টেবিল      │ এমবেডেড/একই টেবিল │
└─────────────────────┴───────────────────┴───────────────────┘
```

---

## 📌 3. Aggregates (অ্যাগ্রিগেট)

### ধারণা

Aggregate হলো **সম্পর্কিত অবজেক্টগুলোর একটি গুচ্ছ** যেগুলো একটি ইউনিট হিসেবে
পরিচালিত হয়। বাইরে থেকে শুধু **Aggregate Root** কে অ্যাক্সেস করা যায়।

**বাস্তব উদাহরণ:** Daraz-এ একটি অর্ডার (Order) → তার আইটেমগুলো (OrderLine) →
প্রোডাক্ট রেফারেন্স। অর্ডার হলো Aggregate Root।

```
┌──────────────────────────────────────────────────────┐
│                  ORDER AGGREGATE                      │
│    ┌──────────────────────────────────────┐           │
│    │      Order (Aggregate Root)          │           │
│    │      ID: ORD-2024-001               │           │
│    │      Status: confirmed              │           │
│    │      Total: ৳2,500                  │           │
│    │                                      │           │
│    │   ┌──────────────┐ ┌──────────────┐ │           │
│    │   │ OrderLine #1 │ │ OrderLine #2 │ │           │
│    │   │ Qty: 2       │ │ Qty: 1       │ │           │
│    │   │ Price: ৳500  │ │ Price: ৳1500 │ │           │
│    │   │ ProductId:   │ │ ProductId:   │ │           │
│    │   │  P-101 ──────┼─┼──────────┐   │ │           │
│    │   └──────────────┘ └──────────│───┘ │           │
│    └───────────────────────────────│─────┘           │
│                                    │                  │
│    বাইরের Aggregate শুধু ID দিয়ে   │                  │
│    রেফারেন্স করবে ──────────────►  ▼                  │
│                          ┌──────────────┐            │
│                          │   Product    │            │
│                          │ (আলাদা      │            │
│                          │  Aggregate)  │            │
│                          └──────────────┘            │
└──────────────────────────────────────────────────────┘
```

### Aggregate-এর নিয়মাবলী

```
┌─────────────────────────────────────────────┐
│         AGGREGATE নিয়ম (RULES)              │
│                                              │
│  1️⃣  বাইরে থেকে শুধু Root অ্যাক্সেসযোগ্য   │
│  2️⃣  অন্য Aggregate-কে শুধু ID দিয়ে রেফার   │
│  3️⃣  এক Transaction = এক Aggregate          │
│  4️⃣  Aggregate Root invariant রক্ষা করে     │
│  5️⃣  ছোট রাখুন — বড় Aggregate = সমস্যা     │
└─────────────────────────────────────────────┘
```

### PHP Order Aggregate — সম্পূর্ণ উদাহরণ

```php
// Daraz-এর মতো অর্ডার সিস্টেম
class OrderLine {
    public function __construct(
        private readonly string $productId, // অন্য Aggregate-র ID
        private readonly string $productName,
        private readonly Money $unitPrice,
        private int $quantity
    ) {
        if ($quantity <= 0) {
            throw new \DomainException("পরিমাণ ০-এর বেশি হতে হবে");
        }
    }

    public function getProductId(): string { return $this->productId; }
    public function getQuantity(): int { return $this->quantity; }

    public function lineTotal(): Money {
        $total = Money::taka(0);
        for ($i = 0; $i < $this->quantity; $i++) {
            $total = $total->add($this->unitPrice);
        }
        return $total;
    }

    public function increaseQuantity(int $amount): void {
        $this->quantity += $amount;
    }
}

class Order {
    private array $lines = [];
    private string $status = 'draft';

    public function __construct(
        private readonly OrderId $id,
        private readonly string $customerId  // অন্য Aggregate-র ID
    ) {}

    public function getId(): OrderId { return $this->id; }

    // Aggregate Root-এর মাধ্যমেই আইটেম যোগ করতে হবে
    public function addItem(string $productId, string $name, Money $price, int $qty): void {
        if ($this->status !== 'draft') {
            throw new \DomainException("নিশ্চিত অর্ডারে আইটেম যোগ করা যায় না");
        }
        // invariant: সর্বোচ্চ ২০টি আইটেম
        if (count($this->lines) >= 20) {
            throw new \DomainException("একটি অর্ডারে সর্বোচ্চ ২০টি আইটেম");
        }

        foreach ($this->lines as $line) {
            if ($line->getProductId() === $productId) {
                $line->increaseQuantity($qty);
                return;
            }
        }
        $this->lines[] = new OrderLine($productId, $name, $price, $qty);
    }

    public function removeItem(string $productId): void {
        if ($this->status !== 'draft') {
            throw new \DomainException("নিশ্চিত অর্ডার থেকে আইটেম সরানো যায় না");
        }
        $this->lines = array_values(
            array_filter($this->lines, fn($l) => $l->getProductId() !== $productId)
        );
    }

    // invariant: Aggregate Root মোট হিসাব রক্ষা করে
    public function calculateTotal(): Money {
        $total = Money::taka(0);
        foreach ($this->lines as $line) {
            $total = $total->add($line->lineTotal());
        }
        return $total;
    }

    public function confirm(): void {
        if (empty($this->lines)) {
            throw new \DomainException("খালি অর্ডার নিশ্চিত করা যায় না");
        }
        $this->status = 'confirmed';
    }

    public function getStatus(): string { return $this->status; }
}
```

### JavaScript Order Aggregate

```javascript
// ✅ JS Aggregate — Daraz-এর অর্ডার
class Order {
    #id;
    #customerId;
    #lines = [];
    #status = 'draft';

    constructor(id, customerId) {
        this.#id = id;
        this.#customerId = customerId;
    }

    addItem(productId, name, unitPrice, quantity) {
        if (this.#status !== 'draft') {
            throw new Error("নিশ্চিত অর্ডারে আইটেম যোগ করা যায় না");
        }
        if (this.#lines.length >= 20) {
            throw new Error("সর্বোচ্চ ২০টি আইটেম");
        }
        const existing = this.#lines.find(l => l.productId === productId);
        if (existing) {
            existing.quantity += quantity;
        } else {
            this.#lines.push({ productId, name, unitPrice, quantity });
        }
    }

    removeItem(productId) {
        if (this.#status !== 'draft') throw new Error("পরিবর্তন অনুমোদিত নয়");
        this.#lines = this.#lines.filter(l => l.productId !== productId);
    }

    calculateTotal() {
        return this.#lines.reduce((total, line) => {
            return total.add(line.unitPrice.multiply(line.quantity));
        }, Money.taka(0));
    }

    confirm() {
        if (this.#lines.length === 0) throw new Error("খালি অর্ডার নিশ্চিত করা যায় না");
        this.#status = 'confirmed';
    }
}
```

---

## 📌 4. Repositories (রিপোজিটরি)

### ধারণা

Repository হলো **সংগ্রহস্থল (collection)** -এর মতো ইন্টারফেস যা Aggregate Root-কে
সংরক্ষণ ও পুনরুদ্ধার করতে ব্যবহৃত হয়। এটি ডাটাবেসের বিস্তারিত লুকিয়ে রাখে —
ডোমেইন লেয়ারের কাছে মনে হয় যেন একটি ইন-মেমরি কালেকশন।

```
┌─────────────────────────────────────────────────┐
│              REPOSITORY প্যাটার্ন                │
│                                                  │
│   Domain Layer          Infrastructure Layer     │
│                                                  │
│  ┌──────────────┐      ┌─────────────────────┐  │
│  │ OrderRepo    │      │  MySqlOrderRepo     │  │
│  │ (interface)  │◄─────│  (implementation)    │  │
│  │              │      │                      │  │
│  │ + findById() │      │  - PDO $connection   │  │
│  │ + save()     │      │  + findById()        │  │
│  │ + nextId()   │      │  + save()            │  │
│  └──────────────┘      └─────────────────────┘  │
│         ▲                                        │
│         │ ব্যবহার করে                            │
│  ┌──────────────┐                                │
│  │ Application  │                                │
│  │ Service      │                                │
│  └──────────────┘                                │
└─────────────────────────────────────────────────┘
```

### ❌ Bad: চাইল্ড Entity সরাসরি ফেরত দেওয়া

```php
// ❌ খারাপ — OrderLine আলাদাভাবে ফেরত দিচ্ছে, Aggregate ভঙ্গ
class OrderLineRepository {
    public function findByProductId(string $productId): array {
        // সরাসরি OrderLine ফেরত দিলে invariant রক্ষা হয় না
        return $this->db->query("SELECT * FROM order_lines WHERE product_id = ?", [$productId]);
    }
}
```

### ✅ Good: শুধু Aggregate Root-এর জন্য Repository

```php
// ✅ ভালো — শুধু Aggregate Root (Order) এর জন্য Repository
interface OrderRepository {
    public function findById(OrderId $id): ?Order;
    public function save(Order $order): void;
    public function nextIdentity(): OrderId;
}

class MySqlOrderRepository implements OrderRepository {
    public function __construct(private \PDO $pdo) {}

    public function findById(OrderId $id): ?Order {
        $stmt = $this->pdo->prepare(
            "SELECT * FROM orders WHERE id = ?"
        );
        $stmt->execute([$id->getValue()]);
        $row = $stmt->fetch();

        if (!$row) return null;

        $order = new Order(
            new OrderId($row['id']),
            $row['customer_id']
        );

        // OrderLine-গুলোও একসাথে লোড করি — Aggregate সম্পূর্ণ
        $lineStmt = $this->pdo->prepare(
            "SELECT * FROM order_lines WHERE order_id = ?"
        );
        $lineStmt->execute([$id->getValue()]);

        foreach ($lineStmt->fetchAll() as $lineRow) {
            $order->addItem(
                $lineRow['product_id'],
                $lineRow['product_name'],
                Money::taka((int)$lineRow['price_taka']),
                (int)$lineRow['quantity']
            );
        }

        return $order;
    }

    public function save(Order $order): void {
        // সম্পূর্ণ Aggregate একসাথে সংরক্ষণ
        $this->pdo->beginTransaction();
        try {
            // Order সংরক্ষণ ...
            // OrderLines সংরক্ষণ ...
            $this->pdo->commit();
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }

    public function nextIdentity(): OrderId {
        return new OrderId('ORD-' . uniqid());
    }
}
```

### JavaScript Repository

```javascript
// ✅ JS Repository — Aggregate Root-এর জন্য
class OrderRepository {
    #db;
    constructor(db) { this.#db = db; }

    async findById(orderId) {
        const row = await this.#db.query(
            'SELECT * FROM orders WHERE id = $1', [orderId]
        );
        if (!row) return null;

        const order = new Order(row.id, row.customer_id);

        const lines = await this.#db.query(
            'SELECT * FROM order_lines WHERE order_id = $1', [orderId]
        );
        lines.forEach(l => order.addItem(l.product_id, l.name, Money.taka(l.price), l.qty));

        return order;  // সম্পূর্ণ Aggregate ফেরত
    }

    async save(order) {
        // সম্পূর্ণ Aggregate একটি transaction-এ সংরক্ষণ
        await this.#db.transaction(async (tx) => {
            await tx.query('INSERT INTO orders ...', [/* ... */]);
            // lines সংরক্ষণ ...
        });
    }
}
```

---

## 📌 5. Domain Services vs Application Services

### ধারণা

| ধরন | কাজ | উদাহরণ |
|------|------|--------|
| **Domain Service** | ব্যবসায়িক লজিক যা কোনো একক Entity/VO-তে বসে না | টাকা ট্রান্সফার (দুটি অ্যাকাউন্ট জড়িত) |
| **Application Service** | ব্যবহার কেস সমন্বয়, ডোমেইন অবজেক্ট একত্রিত করা | ট্রান্সফার অনুরোধ গ্রহণ → ভ্যালিডেশন → ট্রান্সফার → নোটিফিকেশন |

```
┌─────────────────────────────────────────────────────────────┐
│                    স্তর বিন্যাস                              │
│                                                              │
│   ┌───────────────────────────────────────────────────┐     │
│   │  Application Service (অ্যাপ্লিকেশন সার্ভিস)      │     │
│   │  - Use case সমন্বয়                                │     │
│   │  - Transaction পরিচালনা                            │     │
│   │  - Authorization                                   │     │
│   │  - DTO ↔ Domain অবজেক্ট রূপান্তর                  │     │
│   └──────────────────────┬────────────────────────────┘     │
│                          │ ব্যবহার করে                      │
│   ┌──────────────────────▼────────────────────────────┐     │
│   │  Domain Service (ডোমেইন সার্ভিস)                  │     │
│   │  - ব্যবসায়িক লজিক                                 │     │
│   │  - একাধিক Aggregate জড়িত                          │     │
│   │  - কোনো Entity-তে বসে না এমন লজিক                 │     │
│   └──────────────────────┬────────────────────────────┘     │
│                          │ ব্যবহার করে                      │
│   ┌──────────────────────▼────────────────────────────┐     │
│   │  Entity / Value Object / Aggregate                 │     │
│   └───────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### PHP Domain Service — bKash টাকা ট্রান্সফার

```php
// Domain Service — ব্যবসায়িক নিয়ম এখানে
class MoneyTransferService {
    public function transfer(
        Wallet $sender,
        Wallet $receiver,
        Money $amount
    ): void {
        // ব্যবসায়িক নিয়ম: একই ওয়ালেটে ট্রান্সফার অনুমোদিত নয়
        if ($sender->getId()->equals($receiver->getId())) {
            throw new \DomainException("নিজের ওয়ালেটে ট্রান্সফার করা যায় না");
        }

        // ব্যবসায়িক নিয়ম: সর্বোচ্চ ট্রান্সফার সীমা
        $maxLimit = Money::taka(25000);
        if ($amount->getAmountInPoisha() > $maxLimit->getAmountInPoisha()) {
            throw new \DomainException("সর্বোচ্চ ট্রান্সফার সীমা ৳২৫,০০০");
        }

        $sender->debit($amount);
        $receiver->credit($amount);
    }
}
```

### PHP Application Service — ব্যবহার কেস সমন্বয়

```php
// Application Service — use case সমন্বয়
class TransferMoneyHandler {
    public function __construct(
        private WalletRepository $walletRepo,
        private MoneyTransferService $transferService,
        private NotificationService $notifier,
        private TransactionManager $txManager
    ) {}

    public function handle(TransferMoneyCommand $command): void {
        $this->txManager->begin();
        try {
            // 1. Aggregate লোড
            $sender = $this->walletRepo->findById($command->senderId);
            $receiver = $this->walletRepo->findById($command->receiverId);

            // 2. Domain Service কল — ব্যবসায়িক লজিক
            $amount = Money::taka($command->amountTaka);
            $this->transferService->transfer($sender, $receiver, $amount);

            // 3. সংরক্ষণ
            $this->walletRepo->save($sender);
            $this->walletRepo->save($receiver);

            $this->txManager->commit();

            // 4. পার্শ্ব-প্রতিক্রিয়া (notification)
            $this->notifier->sendSms(
                $sender->getPhone(),
                "৳{$command->amountTaka} ট্রান্সফার সম্পন্ন"
            );
        } catch (\Exception $e) {
            $this->txManager->rollback();
            throw $e;
        }
    }
}
```

### JavaScript Domain Service ও Application Service

```javascript
// Domain Service
class MoneyTransferService {
    transfer(senderWallet, receiverWallet, amount) {
        if (senderWallet.getId() === receiverWallet.getId()) {
            throw new Error("নিজের ওয়ালেটে ট্রান্সফার অনুমোদিত নয়");
        }
        if (amount.getAmountInPoisha() > 2500000) { // ৳২৫,০০০
            throw new Error("সর্বোচ্চ সীমা অতিক্রম");
        }
        senderWallet.debit(amount);
        receiverWallet.credit(amount);
    }
}

// Application Service
class TransferMoneyHandler {
    #walletRepo;
    #transferService;
    #notifier;

    constructor(walletRepo, transferService, notifier) {
        this.#walletRepo = walletRepo;
        this.#transferService = transferService;
        this.#notifier = notifier;
    }

    async handle(command) {
        const sender = await this.#walletRepo.findById(command.senderId);
        const receiver = await this.#walletRepo.findById(command.receiverId);
        const amount = Money.taka(command.amountTaka);

        this.#transferService.transfer(sender, receiver, amount);

        await this.#walletRepo.save(sender);
        await this.#walletRepo.save(receiver);

        await this.#notifier.sendSms(sender.getPhone(), `৳${command.amountTaka} ট্রান্সফার সম্পন্ন`);
    }
}
```

### 📊 Domain Service vs Application Service তুলনা

```
┌──────────────────────┬─────────────────────┬─────────────────────┐
│      বৈশিষ্ট্য       │   Domain Service    │ Application Service │
├──────────────────────┼─────────────────────┼─────────────────────┤
│ ব্যবসায়িক লজিক      │ ✅ হ্যাঁ             │ ❌ না               │
│ Infrastructure নির্ভর│ ❌ না               │ ✅ হ্যাঁ             │
│ Transaction পরিচালনা │ ❌ না               │ ✅ হ্যাঁ             │
│ অন্য সার্ভিস কল     │ ❌ না               │ ✅ হ্যাঁ             │
│ DTO রূপান্তর         │ ❌ না               │ ✅ হ্যাঁ             │
│ কোন লেয়ারে          │ Domain              │ Application         │
│ উদাহরণ              │ TransferMoney       │ TransferHandler     │
└──────────────────────┴─────────────────────┴─────────────────────┘
```

---

## 📌 6. Factories (ফ্যাক্টরি)

### ধারণা

Factory প্যাটার্ন ব্যবহার করা হয় যখন কোনো Aggregate তৈরি করা জটিল হয়। কনস্ট্রাক্টরে
সব লজিক না রেখে Factory-তে রাখলে কোড পরিষ্কার থাকে এবং ডোমেইন নিয়ম মানা সহজ হয়।

### PHP Factory

```php
// Grameenphone সিম রেজিস্ট্রেশন — জটিল Aggregate তৈরি
class SubscriberFactory {
    public function createPrepaid(
        string $name,
        string $nid,
        string $phoneNumber,
        string $simType = 'regular'
    ): Subscriber {
        // ব্যবসায়িক নিয়ম: NID যাচাই
        if (strlen($nid) !== 10 && strlen($nid) !== 17) {
            throw new \DomainException("NID ১০ বা ১৭ সংখ্যার হতে হবে");
        }

        // ব্যবসায়িক নিয়ম: বাংলাদেশি নম্বর
        if (!preg_match('/^01[3-9]\d{8}$/', $phoneNumber)) {
            throw new \DomainException("সঠিক বাংলাদেশি মোবাইল নম্বর দিন");
        }

        $subscriber = new Subscriber(
            SubscriberId::generate(),
            new PhoneNumber($phoneNumber),
            $name,
            new NationalId($nid)
        );

        $subscriber->setPlan(Plan::prepaid());
        $subscriber->setSimType($simType);
        $subscriber->activate();

        return $subscriber;
    }
}
```

### JavaScript Factory

```javascript
// ✅ JS Factory — Pathao রাইড তৈরি
class RideFactory {
    static createRide(passengerId, pickup, dropoff, rideType = 'bike') {
        if (!pickup || !dropoff) {
            throw new Error("পিকআপ ও ড্রপঅফ লোকেশন আবশ্যক");
        }

        const ride = new Ride(
            RideId.generate(),
            passengerId,
            new Location(pickup.lat, pickup.lng),
            new Location(dropoff.lat, dropoff.lng)
        );

        const fare = FareCalculator.estimate(ride.getDistance(), rideType);
        ride.setEstimatedFare(fare);
        ride.setRideType(rideType);

        return ride;
    }
}

// ব্যবহার:
const ride = RideFactory.createRide(
    'P-001',
    { lat: 23.8103, lng: 90.4125 },  // গুলশান
    { lat: 23.7465, lng: 90.3763 },  // ধানমন্ডি
    'car'
);
```

---

## 📌 7. Specifications (স্পেসিফিকেশন প্যাটার্ন)

### ধারণা

Specification প্যাটার্ন ব্যবসায়িক নিয়মকে **আলাদা অবজেক্ট** হিসেবে প্রকাশ করে।
এগুলো AND, OR, NOT দিয়ে **সংযুক্ত (compose)** করা যায়।

```
┌─────────────────────────────────────────────────┐
│          SPECIFICATION কম্পোজিশন                 │
│                                                  │
│   PremiumCustomer  AND  HasNoOverdue             │
│        ✅                   ✅                    │
│         \                  /                     │
│          \                /                      │
│           ▼              ▼                       │
│       ┌──────────────────────┐                   │
│       │  EligibleForDiscount │                   │
│       │        ✅             │                   │
│       └──────────────────────┘                   │
│                                                  │
│   PremiumCustomer  AND  HasNoOverdue             │
│        ❌                   ✅                    │
│         \                  /                     │
│          \                /                      │
│           ▼              ▼                       │
│       ┌──────────────────────┐                   │
│       │  EligibleForDiscount │                   │
│       │        ❌             │                   │
│       └──────────────────────┘                   │
└─────────────────────────────────────────────────┘
```

### PHP Specification — সম্পূর্ণ বাস্তবায়ন

```php
// বেস Specification ইন্টারফেস
interface Specification {
    public function isSatisfiedBy($candidate): bool;
}

// AND কম্পোজিশন
class AndSpecification implements Specification {
    public function __construct(
        private Specification $left,
        private Specification $right
    ) {}

    public function isSatisfiedBy($candidate): bool {
        return $this->left->isSatisfiedBy($candidate)
            && $this->right->isSatisfiedBy($candidate);
    }
}

// OR কম্পোজিশন
class OrSpecification implements Specification {
    public function __construct(
        private Specification $left,
        private Specification $right
    ) {}

    public function isSatisfiedBy($candidate): bool {
        return $this->left->isSatisfiedBy($candidate)
            || $this->right->isSatisfiedBy($candidate);
    }
}

// NOT কম্পোজিশন
class NotSpecification implements Specification {
    public function __construct(private Specification $spec) {}

    public function isSatisfiedBy($candidate): bool {
        return !$this->spec->isSatisfiedBy($candidate);
    }
}

// ব্যবসায়িক নিয়ম: প্রিমিয়াম কাস্টমার কি না
class PremiumCustomerSpecification implements Specification {
    public function isSatisfiedBy($customer): bool {
        return $customer->getTotalPurchases()->getAmountInPoisha() >= 5000000; // ৳৫০,০০০+
    }
}

// ব্যবসায়িক নিয়ম: বকেয়া নেই
class HasNoOverdueSpecification implements Specification {
    public function isSatisfiedBy($customer): bool {
        return $customer->getOverduePayments() === 0;
    }
}

// ব্যবহার — Daraz-এ ডিসকাউন্ট যোগ্যতা যাচাই
$eligibleForDiscount = new AndSpecification(
    new PremiumCustomerSpecification(),
    new HasNoOverdueSpecification()
);

if ($eligibleForDiscount->isSatisfiedBy($customer)) {
    $order->applyDiscount(Percentage::of(15));
}
```

### JavaScript Specification

```javascript
// ✅ JS Specification — Daraz ফ্ল্যাশ সেল যোগ্যতা
class Specification {
    isSatisfiedBy(candidate) { throw new Error("Override করুন"); }
    and(other) { return new AndSpec(this, other); }
    or(other) { return new OrSpec(this, other); }
    not() { return new NotSpec(this); }
}

class AndSpec extends Specification {
    #left; #right;
    constructor(left, right) { super(); this.#left = left; this.#right = right; }
    isSatisfiedBy(c) { return this.#left.isSatisfiedBy(c) && this.#right.isSatisfiedBy(c); }
}

class OrSpec extends Specification {
    #left; #right;
    constructor(left, right) { super(); this.#left = left; this.#right = right; }
    isSatisfiedBy(c) { return this.#left.isSatisfiedBy(c) || this.#right.isSatisfiedBy(c); }
}

class NotSpec extends Specification {
    #spec;
    constructor(spec) { super(); this.#spec = spec; }
    isSatisfiedBy(c) { return !this.#spec.isSatisfiedBy(c); }
}

// ব্যবসায়িক নিয়ম
class VerifiedUserSpec extends Specification {
    isSatisfiedBy(user) { return user.isVerified; }
}

class MinBalanceSpec extends Specification {
    #minAmount;
    constructor(minAmount) { super(); this.#minAmount = minAmount; }
    isSatisfiedBy(user) { return user.balance >= this.#minAmount; }
}

// কম্পোজ করে ব্যবহার
const canParticipateInFlashSale = new VerifiedUserSpec()
    .and(new MinBalanceSpec(100));

const eligible = customers.filter(c => canParticipateInFlashSale.isSatisfiedBy(c));
```

---

## 🎯 8. সম্পূর্ণ উদাহরণ — bKash-like Money Transfer

### ডোমেইন মডেল ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────┐
│              bKash-like মানি ট্রান্সফার ডোমেইন মডেল            │
│                                                                  │
│  ┌─────────────────────┐         ┌─────────────────────┐        │
│  │  Wallet Aggregate   │         │  Transaction Agg.   │        │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │        │
│  │  │    Wallet     │  │   ID    │  │  Transaction  │  │        │
│  │  │  (Agg. Root)  │──┼────────►│  │  (Agg. Root)  │  │        │
│  │  │               │  │  দিয়ে   │  │               │  │        │
│  │  │ - walletId    │  │  রেফার  │  │ - txnId       │  │        │
│  │  │ - phoneNumber │  │         │  │ - senderWltId │  │        │
│  │  │ - balance VO  │  │         │  │ - receiverWId │  │        │
│  │  │ - status      │  │         │  │ - amount VO   │  │        │
│  │  │               │  │         │  │ - fee VO      │  │        │
│  │  │ + credit()    │  │         │  │ - status      │  │        │
│  │  │ + debit()     │  │         │  │ - timestamp   │  │        │
│  │  └───────────────┘  │         │  └───────────────┘  │        │
│  └─────────────────────┘         └─────────────────────┘        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                 Value Objects                         │       │
│  │  ┌─────────┐ ┌──────────────┐ ┌──────────────────┐  │       │
│  │  │  Money  │ │ PhoneNumber  │ │    WalletId      │  │       │
│  │  │ (টাকা)  │ │ (ফোন নম্বর) │ │  (ওয়ালেট আইডি)  │  │       │
│  │  └─────────┘ └──────────────┘ └──────────────────┘  │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
│  ┌─────────────────────┐  ┌──────────────────────────┐          │
│  │  Domain Service     │  │  Application Service     │          │
│  │  MoneyTransfer      │  │  TransferMoneyHandler    │          │
│  │  Service            │  │  (use case সমন্বয়)      │          │
│  └─────────────────────┘  └──────────────────────────┘          │
│                                                                  │
│  ┌─────────────────────┐  ┌──────────────────────────┐          │
│  │  Repository         │  │  Specification           │          │
│  │  WalletRepository   │  │  EligibleForTransfer     │          │
│  └─────────────────────┘  └──────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### সম্পূর্ণ PHP কোড

```php
// === Value Objects ===
class WalletId {
    public function __construct(private readonly string $value) {}
    public function getValue(): string { return $this->value; }
    public function equals(WalletId $other): bool {
        return $this->value === $other->value;
    }
}

class PhoneNumber {
    public function __construct(private readonly string $number) {
        if (!preg_match('/^01[3-9]\d{8}$/', $number)) {
            throw new \InvalidArgumentException("সঠিক বাংলাদেশি নম্বর দিন");
        }
    }
    public function getValue(): string { return $this->number; }
}

// === Entity / Aggregate Root ===
class Wallet {
    private Money $balance;
    private string $status = 'active';

    public function __construct(
        private readonly WalletId $id,
        private readonly PhoneNumber $phone
    ) {
        $this->balance = Money::taka(0);
    }

    public function getId(): WalletId { return $this->id; }
    public function getPhone(): PhoneNumber { return $this->phone; }
    public function getBalance(): Money { return $this->balance; }

    public function credit(Money $amount): void {
        $this->ensureActive();
        $this->balance = $this->balance->add($amount);
    }

    public function debit(Money $amount): void {
        $this->ensureActive();
        if ($this->balance->getAmountInPoisha() < $amount->getAmountInPoisha()) {
            throw new \DomainException("অপর্যাপ্ত ব্যালেন্স");
        }
        $this->balance = $this->balance->subtract($amount);
    }

    private function ensureActive(): void {
        if ($this->status !== 'active') {
            throw new \DomainException("ওয়ালেট সক্রিয় নয়");
        }
    }
}

// === Domain Service ===
class MoneyTransferDomainService {
    public function transfer(Wallet $sender, Wallet $receiver, Money $amount): Money {
        if ($sender->getId()->equals($receiver->getId())) {
            throw new \DomainException("নিজের ওয়ালেটে ট্রান্সফার অনুমোদিত নয়");
        }

        $fee = $this->calculateFee($amount);
        $totalDebit = $amount->add($fee);

        $sender->debit($totalDebit);
        $receiver->credit($amount);

        return $fee;
    }

    private function calculateFee(Money $amount): Money {
        $taka = $amount->getAmountInPoisha() / 100;
        if ($taka <= 500) return Money::taka(5);
        if ($taka <= 1000) return Money::taka(10);
        return Money::taka(15);
    }
}

// === Repository Interface ===
interface WalletRepository {
    public function findById(WalletId $id): ?Wallet;
    public function save(Wallet $wallet): void;
}

// === Application Service ===
class SendMoneyHandler {
    public function __construct(
        private WalletRepository $walletRepo,
        private MoneyTransferDomainService $transferService
    ) {}

    public function handle(string $senderWalletId, string $receiverWalletId, int $amountTaka): array {
        $sender = $this->walletRepo->findById(new WalletId($senderWalletId));
        $receiver = $this->walletRepo->findById(new WalletId($receiverWalletId));

        if (!$sender || !$receiver) {
            throw new \RuntimeException("ওয়ালেট পাওয়া যায়নি");
        }

        $amount = Money::taka($amountTaka);
        $fee = $this->transferService->transfer($sender, $receiver, $amount);

        $this->walletRepo->save($sender);
        $this->walletRepo->save($receiver);

        return [
            'status' => 'success',
            'amount' => $amount->format(),
            'fee' => $fee->format(),
        ];
    }
}
```

---

## 🎯 9. কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Tactical DDD ব্যবহার করবেন

```
┌─────────────────────────────────────────────────────────┐
│  ✅ ব্যবহার করুন যখন:                                   │
│                                                          │
│  • ব্যবসায়িক লজিক জটিল (bKash ট্রান্সফার নিয়ম)        │
│  • ডোমেইন বিশেষজ্ঞের সাথে কাজ করছেন                     │
│  • সিস্টেম দীর্ঘমেয়াদী ও বিবর্তনশীল                     │
│  • একাধিক Bounded Context আছে                           │
│  • ব্যবসায়িক নিয়ম ঘন ঘন পরিবর্তন হয়                    │
│  • Core Domain — প্রতিযোগিতামূলক সুবিধা দেয়              │
│                                                          │
│  উদাহরণ:                                                 │
│  • bKash — লেনদেন, ওয়ালেট, KYC                          │
│  • Pathao — রাইড ম্যাচিং, ভাড়া হিসাব, সার্জ প্রাইসিং   │
│  • Daraz — অর্ডার, ইনভেন্টরি, রিটার্ন ও রিফান্ড         │
└─────────────────────────────────────────────────────────┘
```

### ❌ কখন ব্যবহার করবেন না

```
┌─────────────────────────────────────────────────────────┐
│  ❌ ব্যবহার করবেন না যখন:                               │
│                                                          │
│  • সাধারণ CRUD অপারেশন (ব্লগ, টোডো লিস্ট)              │
│  • ব্যবসায়িক লজিক খুবই সরল                              │
│  • ছোট প্রজেক্ট বা প্রোটোটাইপ                           │
│  • Supporting/Generic Subdomain                          │
│  • টিমে DDD অভিজ্ঞতা নেই ও শেখার সময় নেই               │
│                                                          │
│  উদাহরণ:                                                 │
│  • ব্লগ ওয়েবসাইট — সাধারণ CRUD যথেষ্ট                   │
│  • Email পাঠানোর সার্ভিস — Generic Subdomain             │
│  • Static ওয়েবসাইট — DDD-র দরকার নেই                   │
└─────────────────────────────────────────────────────────┘
```

### 📊 প্যাটার্ন নির্বাচন গাইড

```
প্রশ্ন জিজ্ঞেস করুন:

     এর নিজস্ব পরিচয় আছে?
     ├── হ্যাঁ ──► ENTITY (User, Order, Product)
     └── না  ──► VALUE OBJECT (Money, Address, Email)

     এটি কি একাধিক অবজেক্টের গুচ্ছ?
     ├── হ্যাঁ ──► AGGREGATE (Order + OrderLines)
     └── না  ──► একক Entity বা Value Object

     লজিক কোনো Entity-তে বসে না?
     ├── হ্যাঁ ──► DOMAIN SERVICE (MoneyTransfer)
     └── না  ──► Entity-র মেথডে রাখুন

     অবজেক্ট তৈরি করা জটিল?
     ├── হ্যাঁ ──► FACTORY (SubscriberFactory)
     └── না  ──► Constructor যথেষ্ট

     ব্যবসায়িক নিয়ম composable হতে হবে?
     ├── হ্যাঁ ──► SPECIFICATION (EligibleForDiscount)
     └── না  ──► সরল if/else যথেষ্ট
```

---

## 🔗 সম্পর্কিত বিষয়

- **Strategic Design** — Bounded Context, Ubiquitous Language, Context Mapping
- **Domain Events** — Aggregate-এর মধ্যে ঘটনা প্রকাশ ও প্রতিক্রিয়া
- **CQRS** — Command Query Responsibility Segregation
- **Event Sourcing** — ইভেন্টের ধারাবাহিকতায় অবস্থা সংরক্ষণ

---

> **মনে রাখুন:** Tactical Design হলো হাতিয়ার — সব হাতিয়ার সব জায়গায় লাগে না।
> আপনার ডোমেইনের জটিলতা বুঝে সঠিক প্যাটার্ন বেছে নিন। সরল সমস্যার জন্য সরল সমাধানই শ্রেষ্ঠ! 🎯
