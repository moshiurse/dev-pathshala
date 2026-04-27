# 📋 Prototype প্যাটার্ন

## 📌 সংজ্ঞা ও পরিচিতি

> **GoF সংজ্ঞা:** *"Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype."*

Prototype প্যাটার্ন হলো একটি **Creational Design Pattern** যেখানে নতুন অবজেক্ট তৈরি করার জন্য `new` কীওয়ার্ড ব্যবহার না করে একটি **বিদ্যমান অবজেক্টকে ক্লোন (clone)** করা হয়। এই প্যাটার্নের মূল ধারণা হচ্ছে — যখন কোনো অবজেক্ট তৈরি করা ব্যয়বহুল (expensive), জটিল (complex), অথবা অনেকগুলো ধাপের মধ্য দিয়ে যেতে হয়, তখন একটি "প্রোটোটাইপ" (আদর্শ নমুনা) রেখে দিয়ে সেটাকে কপি করে নতুন অবজেক্ট বানানো হয়।

### মূল ধারণা

ধরুন আপনি বাংলাদেশের একটি ই-কমার্স প্ল্যাটফর্মে কাজ করছেন। প্রতিটি ইনভয়েস তৈরি করতে ডাটাবেস থেকে কোম্পানির তথ্য, ট্যাক্স রেট, ডিফল্ট টার্মস অ্যান্ড কন্ডিশনস ইত্যাদি আনতে হয়। প্রতিবার এই পুরো প্রক্রিয়া চালানোর বদলে, একটি **টেমপ্লেট ইনভয়েস** তৈরি করে রাখলে এবং সেটাকে ক্লোন করে নতুন ইনভয়েস বানালে — পারফরম্যান্স অনেক বাড়বে।

```
❌ প্রতিবার new Invoice() → DB query → Config load → Tax calculate
✅ একবার prototype তৈরি → clone → শুধু customer-specific data পরিবর্তন
```

### কেন দরকার?

| সমস্যা | Prototype সমাধান |
|---------|------------------|
| অবজেক্ট তৈরি expensive (DB, API call) | একবার তৈরি, বারবার clone |
| Complex initialization logic | Prototype-এ সব setup রাখা |
| Runtime-এ dynamic object creation | Registry থেকে clone |
| Concrete class-এর উপর নির্ভরশীলতা কমানো | Interface-based cloning |
| Subclass explosion এড়ানো | Clone + modify |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### ১. কোষ বিভাজন (Cell Division)
জীববিজ্ঞানের সবচেয়ে সুন্দর উদাহরণ। একটি কোষ (cell) নিজেকে **বিভাজিত** করে দুটি নতুন কোষ তৈরি করে। প্রতিটি নতুন কোষ মূল কোষের DNA-র একটি কপি পায়, কিন্তু পরে স্বতন্ত্রভাবে পরিবর্তিত হতে পারে — ঠিক Prototype clone-এর মতো।

### ২. ডকুমেন্ট টেমপ্লেট ক্লোনিং
অফিসে আপনি একটি "চিঠির ফরম্যাট" তৈরি করে রাখেন (লেটারহেড, সিগনেচার, ফুটার সহ)। প্রতিটি নতুন চিঠি লিখতে গেলে সেই টেমপ্লেট কপি করে শুধু বডি পরিবর্তন করেন।

### ৩. বাংলাদেশ প্রসঙ্গ: চালান (Invoice) টেমপ্লেট
বাংলাদেশের ব্যবসায় প্রতিদিন শত শত চালান তৈরি হয়। কোম্পানির নাম, ঠিকানা, VAT নম্বর, BIN নম্বর, ব্যাংক ডিটেইলস — এগুলো প্রতিটি চালানে একই থাকে। একটি **মাস্টার চালান টেমপ্লেট** ক্লোন করে শুধু ক্রেতার তথ্য ও পণ্যের তালিকা যোগ করলেই হয়।

### ৪. গেম ডেভেলপমেন্ট
একটি গেমে ১০০টি শত্রু (enemy) স্পন করতে হবে। প্রতিটির mesh, texture, animation, AI behavior প্রায় একই। একটি enemy prototype তৈরি করে ক্লোন করা হয়, তারপর শুধু position ও difficulty level পরিবর্তন করা হয়।

---

## 📊 UML ডায়াগ্রাম

### ক্লাস ডায়াগ্রাম (ASCII)

```
    ┌─────────────────────────┐
    │    <<interface>>         │
    │      Prototype           │
    ├─────────────────────────┤
    │ + clone(): Prototype     │
    └──────────┬──────────────┘
               │ implements
        ┌──────┴──────────┐
        │                 │
┌───────▼───────┐  ┌──────▼────────┐
│ ConcreteProto1 │  │ ConcreteProto2 │
├───────────────┤  ├───────────────┤
│ - field1      │  │ - fieldA      │
│ - field2      │  │ - fieldB      │
├───────────────┤  ├───────────────┤
│ + clone()     │  │ + clone()     │
└───────────────┘  └───────────────┘

        ┌─────────────────────────┐
        │   PrototypeRegistry     │
        ├─────────────────────────┤
        │ - prototypes: Map       │
        ├─────────────────────────┤
        │ + register(key, proto)  │
        │ + get(key): Prototype   │
        │ + unregister(key)       │
        └─────────────────────────┘

 Client ───► Registry ──get("key")──► clone() ──► New Object
```

### সিকোয়েন্স ডায়াগ্রাম

```
 Client          Registry         Prototype
   │                │                 │
   │─ get("enemy") ─►                 │
   │                │── clone() ─────►│
   │                │                 │── creates copy
   │                │◄── new object ──│
   │◄── cloned obj ─│                 │
   │                │                 │
   │─ modify ──────►│                 │
   │  (position,    │                 │
   │   health)      │                 │
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Prototype with Clone

#### PHP 8.3 — `__clone` ম্যাজিক মেথড

PHP-তে `clone` কীওয়ার্ড ব্যবহার করলে অবজেক্টের একটি **shallow copy** তৈরি হয়। `__clone()` ম্যাজিক মেথড দিয়ে ক্লোনিং আচরণ কাস্টমাইজ করা যায়।

```php
<?php

declare(strict_types=1);

// Prototype Interface
interface Prototype
{
    public function clone(): static;
}

// বাংলাদেশী চালান (Invoice) — Concrete Prototype
class Invoice implements Prototype
{
    private string $id;
    private \DateTimeImmutable $createdAt;

    public function __construct(
        private string $companyName,
        private string $companyAddress,
        private string $vatNumber,      // VAT রেজিস্ট্রেশন নম্বর
        private string $binNumber,      // BIN নম্বর
        private string $bankDetails,
        private array $defaultTerms,
        private ?string $customerName = null,
        private array $lineItems = [],
    ) {
        $this->id = uniqid('INV-', true);
        $this->createdAt = new \DateTimeImmutable();
    }

    // clone হলে নতুন ID ও তারিখ সেট হবে
    public function clone(): static
    {
        $cloned = clone $this;
        return $cloned;
    }

    public function __clone(): void
    {
        $this->id = uniqid('INV-', true);
        $this->createdAt = new \DateTimeImmutable();
        $this->customerName = null;
        $this->lineItems = [];
    }

    public function setCustomer(string $name): static
    {
        $this->customerName = $name;
        return $this;
    }

    public function addLineItem(string $item, float $price, int $qty): static
    {
        $this->lineItems[] = [
            'item' => $item,
            'price' => $price,
            'qty' => $qty,
            'total' => $price * $qty,
        ];
        return $this;
    }

    public function getTotal(): float
    {
        return array_sum(array_column($this->lineItems, 'total'));
    }

    public function getSummary(): array
    {
        return [
            'id' => $this->id,
            'company' => $this->companyName,
            'vat' => $this->vatNumber,
            'customer' => $this->customerName,
            'items' => count($this->lineItems),
            'total' => $this->getTotal(),
            'created' => $this->createdAt->format('Y-m-d H:i:s'),
        ];
    }
}

// ব্যবহার
$masterInvoice = new Invoice(
    companyName: 'ডিজিটাল সলিউশনস বাংলাদেশ',
    companyAddress: 'বনানী, ঢাকা-১২১৩',
    vatNumber: 'VAT-20240001234',
    binNumber: 'BIN-001234567',
    bankDetails: 'ডাচ-বাংলা ব্যাংক, A/C: 1234567890',
    defaultTerms: ['৩০ দিনের মধ্যে পরিশোধযোগ্য', 'চেক/বিকাশ/নগদ গ্রহণযোগ্য'],
);

// ক্লোন করে নতুন ইনভয়েস
$invoice1 = $masterInvoice->clone()
    ->setCustomer('রহিম এন্টারপ্রাইজ')
    ->addLineItem('ওয়েব ডেভেলপমেন্ট', 50000.00, 1)
    ->addLineItem('হোস্টিং (১ বছর)', 5000.00, 1);

$invoice2 = $masterInvoice->clone()
    ->setCustomer('করিম ট্রেডার্স')
    ->addLineItem('মোবাইল অ্যাপ', 150000.00, 1);

print_r($invoice1->getSummary());
print_r($invoice2->getSummary());
// দুটি আলাদা ID, আলাদা customer, কিন্তু একই company info
```

#### JavaScript ES2022+ — `structuredClone` ও Spread

```javascript
// Prototype Interface (Symbol ব্যবহার করে)
const CLONE = Symbol('clone');

class Invoice {
    #id;
    #createdAt;
    #companyName;
    #companyAddress;
    #vatNumber;
    #binNumber;
    #bankDetails;
    #defaultTerms;
    #customerName;
    #lineItems;

    constructor({
        companyName,
        companyAddress,
        vatNumber,
        binNumber,
        bankDetails,
        defaultTerms,
        customerName = null,
        lineItems = [],
    }) {
        this.#id = crypto.randomUUID();
        this.#createdAt = new Date();
        this.#companyName = companyName;
        this.#companyAddress = companyAddress;
        this.#vatNumber = vatNumber;
        this.#binNumber = binNumber;
        this.#bankDetails = bankDetails;
        this.#defaultTerms = [...defaultTerms];
        this.#customerName = customerName;
        this.#lineItems = [...lineItems];
    }

    // Deep clone — নতুন ID সহ
    [CLONE]() {
        const cloned = new Invoice({
            companyName: this.#companyName,
            companyAddress: this.#companyAddress,
            vatNumber: this.#vatNumber,
            binNumber: this.#binNumber,
            bankDetails: this.#bankDetails,
            defaultTerms: structuredClone(this.#defaultTerms),
            customerName: null,
            lineItems: [],
        });
        return cloned;
    }

    setCustomer(name) {
        this.#customerName = name;
        return this;
    }

    addLineItem(item, price, qty) {
        this.#lineItems.push({
            item,
            price,
            qty,
            total: price * qty,
        });
        return this;
    }

    get total() {
        return this.#lineItems.reduce((sum, li) => sum + li.total, 0);
    }

    get summary() {
        return {
            id: this.#id,
            company: this.#companyName,
            vat: this.#vatNumber,
            customer: this.#customerName,
            items: this.#lineItems.length,
            total: this.total,
            created: this.#createdAt.toISOString(),
        };
    }
}

// ব্যবহার
const masterInvoice = new Invoice({
    companyName: 'ডিজিটাল সলিউশনস বাংলাদেশ',
    companyAddress: 'বনানী, ঢাকা-১২১৩',
    vatNumber: 'VAT-20240001234',
    binNumber: 'BIN-001234567',
    bankDetails: 'ডাচ-বাংলা ব্যাংক, A/C: 1234567890',
    defaultTerms: ['৩০ দিনের মধ্যে পরিশোধযোগ্য', 'চেক/বিকাশ/নগদ গ্রহণযোগ্য'],
});

const invoice1 = masterInvoice[CLONE]()
    .setCustomer('রহিম এন্টারপ্রাইজ')
    .addLineItem('ওয়েব ডেভেলপমেন্ট', 50000, 1);

const invoice2 = masterInvoice[CLONE]()
    .setCustomer('করিম ট্রেডার্স')
    .addLineItem('মোবাইল অ্যাপ', 150000, 1);

console.log(invoice1.summary);
console.log(invoice2.summary);
```

---

### ২. Shallow Copy vs Deep Copy — গুরুত্বপূর্ণ পার্থক্য

এটি Prototype প্যাটার্নের **সবচেয়ে ক্রিটিক্যাল** বিষয়। ভুল করলে ভয়ানক বাগ তৈরি হয়।

```
Shallow Copy:
  original ──► { name: "A", address: ──► {city: "Dhaka"} }
  clone    ──► { name: "A", address: ──┘  (একই reference!)  }

Deep Copy:
  original ──► { name: "A", address: ──► {city: "Dhaka"} }
  clone    ──► { name: "A", address: ──► {city: "Dhaka"} }  (আলাদা object)
```

#### PHP — Shallow vs Deep Clone

```php
<?php

declare(strict_types=1);

class Address
{
    public function __construct(
        public string $street,
        public string $city,
        public string $district,
    ) {}
}

class Product
{
    public function __construct(
        public string $name,
        public float $price,
        public Address $warehouse,   // nested object!
        public array $tags = [],
    ) {}

    // ⚠️ Shallow Clone — ডিফল্ট PHP আচরণ
    // clone করলে $warehouse এখনও মূল object-এর reference ধরে রাখবে
}

// সমস্যা দেখা যাক
$original = new Product(
    name: 'স্মার্টফোন',
    price: 25000.00,
    warehouse: new Address('মতিঝিল', 'ঢাকা', 'ঢাকা'),
    tags: ['ইলেকট্রনিক্স', 'মোবাইল'],
);

$shallowClone = clone $original;
$shallowClone->name = 'ট্যাবলেট';

// ⚠️ এখানে বিপদ!
$shallowClone->warehouse->city = 'চট্টগ্রাম';

echo $original->warehouse->city;     // "চট্টগ্রাম" 😱 — মূল অবজেক্টও বদলে গেছে!
echo $shallowClone->warehouse->city; // "চট্টগ্রাম"

// ✅ সমাধান — Deep Clone
class ProductDeep
{
    public function __construct(
        public string $name,
        public float $price,
        public Address $warehouse,
        public array $tags = [],
    ) {}

    public function __clone(): void
    {
        // nested object-কে আলাদাভাবে clone করতে হবে
        $this->warehouse = clone $this->warehouse;
        // array ইতোমধ্যে value দিয়ে কপি হয় (primitive-দের জন্য)
        $this->tags = [...$this->tags];
    }
}

$originalDeep = new ProductDeep(
    name: 'স্মার্টফোন',
    price: 25000.00,
    warehouse: new Address('মতিঝিল', 'ঢাকা', 'ঢাকা'),
    tags: ['ইলেকট্রনিক্স', 'মোবাইল'],
);

$deepClone = clone $originalDeep;
$deepClone->warehouse->city = 'চট্টগ্রাম';

echo $originalDeep->warehouse->city; // "ঢাকা" ✅ — মূল অবজেক্ট অক্ষত!
echo $deepClone->warehouse->city;    // "চট্টগ্রাম" ✅
```

#### JavaScript — Shallow vs Deep Copy তুলনা

```javascript
// ========== Shallow Copy পদ্ধতিগুলো ==========

const original = {
    name: 'স্মার্টফোন',
    price: 25000,
    warehouse: { city: 'ঢাকা', district: 'ঢাকা' },
    tags: ['ইলেকট্রনিক্স', 'মোবাইল'],
};

// পদ্ধতি ১: Spread operator — SHALLOW
const spread = { ...original };
spread.warehouse.city = 'চট্টগ্রাম';
console.log(original.warehouse.city); // "চট্টগ্রাম" 😱

// পদ্ধতি ২: Object.assign — SHALLOW
const assigned = Object.assign({}, original);

// ========== Deep Copy পদ্ধতিগুলো ==========

// পদ্ধতি ৩: structuredClone (সেরা!) — DEEP
const deep = structuredClone(original);
deep.warehouse.city = 'সিলেট';
console.log(original.warehouse.city); // "চট্টগ্রাম" (আগের পরিবর্তন) ✅ আর বদলায়নি

// পদ্ধতি ৪: JSON parse/stringify — DEEP (সীমাবদ্ধতা আছে)
const jsonClone = JSON.parse(JSON.stringify(original));
// ⚠️ Date, Map, Set, RegExp, undefined, Function — এগুলো হারিয়ে যায়!

// ========== তুলনা সারণী ==========
/*
┌──────────────────────┬────────┬─────────┬──────────────────────┐
│ পদ্ধতি                │ গভীরতা  │ গতি     │ সীমাবদ্ধতা             │
├──────────────────────┼────────┼─────────┼──────────────────────┤
│ Spread {...obj}      │ Shallow│ দ্রুততম   │ Nested ref শেয়ার করে   │
│ Object.assign()      │ Shallow│ দ্রুত    │ Nested ref শেয়ার করে   │
│ structuredClone()    │ Deep   │ মাঝারি   │ Function clone হয় না  │
│ JSON parse/stringify │ Deep   │ ধীর     │ Date/Map/Set হারায়    │
└──────────────────────┴────────┴─────────┴──────────────────────┘
*/
```

---

### ৩. Prototype Registry / Manager

একটি কেন্দ্রীয় রেজিস্ট্রি যেখানে বিভিন্ন prototype সংরক্ষিত থাকে এবং key দিয়ে ক্লোন করা যায়।

#### PHP 8.3 — Prototype Registry

```php
<?php

declare(strict_types=1);

interface CloneablePrototype
{
    public function clone(): static;
    public function getType(): string;
}

class DocumentTemplate implements CloneablePrototype
{
    public function __construct(
        private string $type,
        private string $header,
        private string $footer,
        private string $fontFamily,
        private float $fontSize,
        private array $metadata = [],
        private ?string $content = null,
    ) {}

    public function clone(): static
    {
        $cloned = clone $this;
        return $cloned;
    }

    public function __clone(): void
    {
        $this->metadata = [...$this->metadata];
        $this->content = null;
    }

    public function getType(): string
    {
        return $this->type;
    }

    public function setContent(string $content): static
    {
        $this->content = $content;
        return $this;
    }

    public function toArray(): array
    {
        return [
            'type' => $this->type,
            'header' => $this->header,
            'footer' => $this->footer,
            'font' => "{$this->fontFamily} {$this->fontSize}pt",
            'content' => $this->content,
        ];
    }
}

// Generic Registry — যেকোনো Prototype-এর জন্য
class PrototypeRegistry
{
    /** @var array<string, CloneablePrototype> */
    private array $prototypes = [];

    public function register(string $key, CloneablePrototype $prototype): void
    {
        $this->prototypes[$key] = $prototype;
    }

    public function unregister(string $key): void
    {
        unset($this->prototypes[$key]);
    }

    public function get(string $key): CloneablePrototype
    {
        if (!isset($this->prototypes[$key])) {
            throw new \InvalidArgumentException(
                "Prototype '{$key}' রেজিস্ট্রিতে নেই।"
            );
        }

        return $this->prototypes[$key]->clone();
    }

    public function has(string $key): bool
    {
        return isset($this->prototypes[$key]);
    }

    /** @return string[] */
    public function listKeys(): array
    {
        return array_keys($this->prototypes);
    }
}

// ব্যবহার — বাংলাদেশী ডকুমেন্ট টেমপ্লেট
$registry = new PrototypeRegistry();

$registry->register('invoice-bn', new DocumentTemplate(
    type: 'চালান',
    header: 'বাংলাদেশ ডিজিটাল সলিউশনস | VAT: 20240001234',
    footer: '© ২০২৪ সর্বস্বত্ব সংরক্ষিত',
    fontFamily: 'SolaimanLipi',
    fontSize: 12.0,
    metadata: ['language' => 'bn', 'currency' => 'BDT'],
));

$registry->register('contract-en', new DocumentTemplate(
    type: 'Contract',
    header: 'Bangladesh Digital Solutions | REG: RJSC-12345',
    footer: '© 2024 All Rights Reserved',
    fontFamily: 'Arial',
    fontSize: 11.0,
    metadata: ['language' => 'en', 'currency' => 'BDT'],
));

// Registry থেকে clone করে ব্যবহার
$newInvoice = $registry->get('invoice-bn')
    ->setContent('পণ্য: ওয়েব ডেভেলপমেন্ট সার্ভিস | মূল্য: ৫০,০০০ টাকা');

$newContract = $registry->get('contract-en')
    ->setContent('This agreement is entered between...');

print_r($newInvoice->toArray());
print_r($newContract->toArray());
```

#### JavaScript ES2022+ — Prototype Registry

```javascript
const CLONE = Symbol('clone');

class DocumentTemplate {
    #type; #header; #footer; #fontFamily; #fontSize; #metadata; #content;

    constructor({ type, header, footer, fontFamily, fontSize, metadata = {}, content = null }) {
        this.#type = type;
        this.#header = header;
        this.#footer = footer;
        this.#fontFamily = fontFamily;
        this.#fontSize = fontSize;
        this.#metadata = { ...metadata };
        this.#content = content;
    }

    [CLONE]() {
        return new DocumentTemplate({
            type: this.#type,
            header: this.#header,
            footer: this.#footer,
            fontFamily: this.#fontFamily,
            fontSize: this.#fontSize,
            metadata: structuredClone(this.#metadata),
            content: null,
        });
    }

    setContent(content) {
        this.#content = content;
        return this;
    }

    toJSON() {
        return {
            type: this.#type,
            header: this.#header,
            font: `${this.#fontFamily} ${this.#fontSize}pt`,
            content: this.#content,
        };
    }
}

class PrototypeRegistry {
    #prototypes = new Map();

    register(key, prototype) {
        if (typeof prototype[CLONE] !== 'function') {
            throw new TypeError(`Prototype-এ [CLONE] মেথড নেই`);
        }
        this.#prototypes.set(key, prototype);
        return this;
    }

    get(key) {
        const proto = this.#prototypes.get(key);
        if (!proto) throw new RangeError(`Prototype '${key}' পাওয়া যায়নি`);
        return proto[CLONE]();
    }

    has(key) { return this.#prototypes.has(key); }
    keys() { return [...this.#prototypes.keys()]; }
    unregister(key) { this.#prototypes.delete(key); }
}

// ব্যবহার
const registry = new PrototypeRegistry();

registry
    .register('invoice-bn', new DocumentTemplate({
        type: 'চালান',
        header: 'বাংলাদেশ ডিজিটাল সলিউশনস | VAT: 20240001234',
        footer: '© ২০২৪ সর্বস্বত্ব সংরক্ষিত',
        fontFamily: 'SolaimanLipi',
        fontSize: 12,
        metadata: { language: 'bn', currency: 'BDT' },
    }))
    .register('report-en', new DocumentTemplate({
        type: 'Report',
        header: 'BD Digital Solutions',
        footer: '© 2024',
        fontFamily: 'Arial',
        fontSize: 11,
        metadata: { language: 'en' },
    }));

const inv = registry.get('invoice-bn').setContent('ওয়েব ডেভ সার্ভিস');
console.log(JSON.stringify(inv, null, 2));
```

---

### ৪. Prototype with Serialization

কিছু ক্ষেত্রে serialization ব্যবহার করে deep clone করা সুবিধাজনক, বিশেষত জটিল নেস্টেড অবজেক্টের ক্ষেত্রে।

#### PHP — Serialization-based Deep Clone

```php
<?php

declare(strict_types=1);

class GameCharacter
{
    public function __construct(
        private string $name,
        private int $level,
        private array $inventory,        // nested arrays
        private array $skillTree,        // complex nested structure
        private array $equipment,
    ) {}

    // Serialization-based deep clone
    public function deepClone(): static
    {
        return unserialize(serialize($this));
    }

    // __clone দিয়ে manual deep clone
    public function __clone(): void
    {
        $this->inventory = array_map(
            fn(array $item) => [...$item],
            $this->inventory
        );
        $this->skillTree = $this->deepCopyArray($this->skillTree);
        $this->equipment = $this->deepCopyArray($this->equipment);
    }

    private function deepCopyArray(array $arr): array
    {
        $copy = [];
        foreach ($arr as $key => $value) {
            $copy[$key] = is_array($value) ? $this->deepCopyArray($value) : $value;
        }
        return $copy;
    }

    public function setName(string $name): static
    {
        $this->name = $name;
        return $this;
    }

    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'level' => $this->level,
            'inventory_count' => count($this->inventory),
            'skills' => $this->skillTree,
        ];
    }
}

$warrior = new GameCharacter(
    name: 'যোদ্ধা-প্রোটোটাইপ',
    level: 10,
    inventory: [
        ['name' => 'তলোয়ার', 'damage' => 50],
        ['name' => 'ঢাল', 'defense' => 30],
    ],
    skillTree: [
        'attack' => ['slash' => 3, 'thrust' => 2],
        'defense' => ['block' => 4, 'parry' => 1],
    ],
    equipment: ['helmet' => 'iron', 'armor' => 'chain'],
);

// Serialization clone
$warrior1 = $warrior->deepClone();
$warrior1->setName('যোদ্ধা-১');

// Manual clone
$warrior2 = clone $warrior;
$warrior2->setName('যোদ্ধা-২');

print_r($warrior1->toArray());
print_r($warrior2->toArray());
```

#### JavaScript — Serialization Clone তুলনা

```javascript
class GameCharacter {
    #name; #level; #inventory; #skillTree;

    constructor({ name, level, inventory, skillTree }) {
        this.#name = name;
        this.#level = level;
        this.#inventory = inventory;
        this.#skillTree = skillTree;
    }

    // structuredClone ব্যবহার — সবচেয়ে নির্ভরযোগ্য
    cloneWithStructured() {
        return new GameCharacter({
            name: this.#name,
            level: this.#level,
            inventory: structuredClone(this.#inventory),
            skillTree: structuredClone(this.#skillTree),
        });
    }

    // JSON ব্যবহার — দ্রুত কিন্তু সীমিত
    cloneWithJSON() {
        const data = JSON.parse(JSON.stringify({
            name: this.#name,
            level: this.#level,
            inventory: this.#inventory,
            skillTree: this.#skillTree,
        }));
        return new GameCharacter(data);
    }

    setName(name) { this.#name = name; return this; }
    get info() {
        return { name: this.#name, level: this.#level, skills: this.#skillTree };
    }
}

const proto = new GameCharacter({
    name: 'যোদ্ধা-মূল',
    level: 10,
    inventory: [{ name: 'তলোয়ার', damage: 50 }],
    skillTree: { attack: { slash: 3 }, defense: { block: 4 } },
});

const w1 = proto.cloneWithStructured().setName('যোদ্ধা-১');
const w2 = proto.cloneWithJSON().setName('যোদ্ধা-২');

console.log(w1.info);
console.log(w2.info);
```

---

## 🌍 Real-World Applicable Areas

### ১. ই-কমার্স: কার্ট আইটেম ডুপ্লিকেশন

```php
<?php

declare(strict_types=1);

class CartItem
{
    private string $cartItemId;

    public function __construct(
        private string $productId,
        private string $productName,
        private float $unitPrice,
        private int $quantity,
        private array $customizations = [],
        private ?float $discount = null,
    ) {
        $this->cartItemId = uniqid('CI-');
    }

    public function __clone(): void
    {
        $this->cartItemId = uniqid('CI-');
        $this->customizations = [...$this->customizations];
    }

    public function setQuantity(int $qty): static
    {
        $this->quantity = $qty;
        return $this;
    }

    public function addCustomization(string $key, mixed $value): static
    {
        $this->customizations[$key] = $value;
        return $this;
    }

    public function getSubtotal(): float
    {
        $subtotal = $this->unitPrice * $this->quantity;
        return $this->discount ? $subtotal - $this->discount : $subtotal;
    }

    public function getSummary(): string
    {
        return sprintf(
            "[%s] %s x%d = ৳%.2f",
            $this->cartItemId,
            $this->productName,
            $this->quantity,
            $this->getSubtotal()
        );
    }
}

// ক্রেতা একটি পণ্য দুইবার কিনতে চায়, সামান্য পরিবর্তন সহ
$item = new CartItem(
    productId: 'PROD-001',
    productName: 'কাস্টম টি-শার্ট',
    unitPrice: 500.00,
    quantity: 2,
    customizations: ['size' => 'L', 'color' => 'লাল'],
);

$duplicated = clone $item;
$duplicated->setQuantity(3)
    ->addCustomization('color', 'নীল')
    ->addCustomization('print', 'কাস্টম লোগো');

echo $item->getSummary() . PHP_EOL;        // [CI-xxx] কাস্টম টি-শার্ট x2 = ৳1000.00
echo $duplicated->getSummary() . PHP_EOL;  // [CI-yyy] কাস্টম টি-শার্ট x3 = ৳1500.00
```

### ২. কনফিগারেশন প্রিসেট: Dev/Staging/Prod

```javascript
class AppConfig {
    #env; #dbHost; #dbPort; #cacheEnabled; #logLevel; #features;

    constructor({ env, dbHost, dbPort, cacheEnabled, logLevel, features }) {
        this.#env = env;
        this.#dbHost = dbHost;
        this.#dbPort = dbPort;
        this.#cacheEnabled = cacheEnabled;
        this.#logLevel = logLevel;
        this.#features = new Map(features);
    }

    clone(overrides = {}) {
        return new AppConfig({
            env: overrides.env ?? this.#env,
            dbHost: overrides.dbHost ?? this.#dbHost,
            dbPort: overrides.dbPort ?? this.#dbPort,
            cacheEnabled: overrides.cacheEnabled ?? this.#cacheEnabled,
            logLevel: overrides.logLevel ?? this.#logLevel,
            features: overrides.features ?? new Map(this.#features),
        });
    }

    get summary() {
        return {
            env: this.#env,
            db: `${this.#dbHost}:${this.#dbPort}`,
            cache: this.#cacheEnabled,
            logLevel: this.#logLevel,
        };
    }
}

// Base config — common defaults
const baseConfig = new AppConfig({
    env: 'base',
    dbHost: 'localhost',
    dbPort: 5432,
    cacheEnabled: false,
    logLevel: 'debug',
    features: [['darkMode', true], ['betaFeatures', false]],
});

// Clone করে environment-specific config তৈরি
const devConfig = baseConfig.clone({ env: 'development' });

const stagingConfig = baseConfig.clone({
    env: 'staging',
    dbHost: 'staging-db.bd-digital.com',
    cacheEnabled: true,
    logLevel: 'info',
});

const prodConfig = baseConfig.clone({
    env: 'production',
    dbHost: 'prod-db.bd-digital.com',
    cacheEnabled: true,
    logLevel: 'error',
});

console.table([devConfig.summary, stagingConfig.summary, prodConfig.summary]);
```

### ৩. ডাটাবেস রেকর্ড ডুপ্লিকেশন

```php
<?php

declare(strict_types=1);

class ProductCatalogEntry
{
    private ?int $id = null;
    private \DateTimeImmutable $createdAt;

    public function __construct(
        private string $sku,
        private string $name,
        private string $category,
        private float $basePrice,
        private array $specifications,
        private array $images,
        private bool $isActive = true,
    ) {
        $this->createdAt = new \DateTimeImmutable();
    }

    // ক্যাটালগ এন্ট্রি ডুপ্লিকেট — নতুন SKU সহ
    public function __clone(): void
    {
        $this->id = null;   // DB-তে নতুন এন্ট্রি হিসেবে সেভ হবে
        $this->sku = $this->sku . '-COPY-' . random_int(1000, 9999);
        $this->createdAt = new \DateTimeImmutable();
        $this->specifications = [...$this->specifications];
        $this->images = [...$this->images];
    }

    public function setSku(string $sku): static { $this->sku = $sku; return $this; }
    public function setName(string $name): static { $this->name = $name; return $this; }
    public function setPrice(float $price): static { $this->basePrice = $price; return $this; }

    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'sku' => $this->sku,
            'name' => $this->name,
            'category' => $this->category,
            'price' => $this->basePrice,
            'specs' => $this->specifications,
            'active' => $this->isActive,
        ];
    }
}

// মূল পণ্য (ডাটাবেস থেকে লোড করা)
$originalProduct = new ProductCatalogEntry(
    sku: 'PHONE-SAM-S24',
    name: 'Samsung Galaxy S24',
    category: 'স্মার্টফোন',
    basePrice: 89999.00,
    specifications: ['ram' => '8GB', 'storage' => '256GB', 'display' => '6.2"'],
    images: ['s24-front.jpg', 's24-back.jpg'],
);

// ভ্যারিয়েন্ট তৈরি — clone করে modification
$variant = clone $originalProduct;
$variant->setSku('PHONE-SAM-S24-ULTRA')
    ->setName('Samsung Galaxy S24 Ultra')
    ->setPrice(139999.00);

print_r($originalProduct->toArray());
print_r($variant->toArray());
```

### ৪. PHP `clone` কীওয়ার্ড Deep Dive

```php
<?php

declare(strict_types=1);

// PHP-তে clone-এর অভ্যন্তরীণ কাজ:
// ১. নতুন মেমরি allocation হয়
// ২. সকল property-র value bit-by-bit কপি হয় (shallow)
// ৩. __clone() ম্যাজিক মেথড কল হয় (যদি থাকে)
// ৪. নতুন object return হয়

class DeepCloneDemo
{
    public function __construct(
        private string $name,
        private \SplDoublyLinkedList $linkedList,  // complex object
        private \DateTimeImmutable $timestamp,
        private array $nested = [],
    ) {}

    public function __clone(): void
    {
        // DateTimeImmutable ক্লোন করার দরকার নেই — immutable!
        // কিন্তু SplDoublyLinkedList ক্লোন করতে হবে
        $newList = new \SplDoublyLinkedList();
        $this->linkedList->rewind();
        while ($this->linkedList->valid()) {
            $newList->push($this->linkedList->current());
            $this->linkedList->next();
        }
        $this->linkedList = $newList;

        // nested array deep copy
        $this->nested = array_map(function (mixed $val): mixed {
            return is_object($val) ? clone $val : $val;
        }, $this->nested);
    }
}
```

### ৫. JS `Object.assign` vs `structuredClone` বিস্তারিত

```javascript
// ==========================================
// Object.assign — Shallow copy
// ==========================================
const src = {
    name: 'মূল',
    meta: { version: 1 },
    tags: ['a', 'b'],
    createdAt: new Date('2024-01-01'),
    pattern: /test/gi,
    data: new Map([['key', 'val']]),
};

const assigned = Object.assign({}, src);
// ✅ name — primitive, আলাদা কপি
// ❌ meta — reference শেয়ার
// ❌ tags — reference শেয়ার
// ❌ createdAt — reference শেয়ার
// ❌ pattern — reference শেয়ার
// ❌ data — reference শেয়ার

// ==========================================
// structuredClone — Deep copy
// ==========================================
const cloned = structuredClone(src);
// ✅ name — আলাদা কপি
// ✅ meta — deep copy, আলাদা object
// ✅ tags — deep copy, আলাদা array
// ✅ createdAt — Date সঠিকভাবে clone হয়!
// ✅ data — Map সঠিকভাবে clone হয়!
// ❌ pattern — RegExp clone হয় ✅ (actually supported!)

// ⚠️ structuredClone-এর সীমাবদ্ধতা:
// - Function clone হয় না → DataCloneError
// - DOM nodes clone হয় না
// - Symbol, WeakMap, WeakSet — clone হয় না
// - prototype chain হারিয়ে যায়

class MyClass {
    constructor(val) { this.val = val; }
    greet() { return `Hello ${this.val}`; }
}

const obj = new MyClass('World');
const scClone = structuredClone(obj);
// scClone.val === 'World' ✅
// scClone instanceof MyClass === false ❌ — prototype হারিয়ে গেছে!
// scClone.greet === undefined ❌

// 🔧 সমাধান: Custom clone method
class MyClassFixed {
    #val;
    constructor(val) { this.#val = val; }
    clone() {
        return new MyClassFixed(structuredClone(this.#val));
    }
}
```

---

## 🔥 Advanced Deep Dive

### ১. PHP `__clone` ম্যাজিক মেথড — Nested Object Deep Clone

```php
<?php

declare(strict_types=1);

class Department
{
    /** @var Employee[] */
    private array $employees = [];

    public function __construct(
        private string $name,
        private ?Manager $manager = null,
    ) {}

    public function addEmployee(Employee $emp): void
    {
        $this->employees[] = $emp;
    }

    public function __clone(): void
    {
        // Manager-কে deep clone
        if ($this->manager !== null) {
            $this->manager = clone $this->manager;
        }

        // প্রতিটি Employee-কে আলাদাভাবে clone
        $this->employees = array_map(
            fn(Employee $emp) => clone $emp,
            $this->employees
        );
    }

    public function getInfo(): array
    {
        return [
            'department' => $this->name,
            'manager' => $this->manager?->getName(),
            'employees' => count($this->employees),
        ];
    }
}

class Manager
{
    public function __construct(
        private string $name,
        private string $designation,
    ) {}

    public function getName(): string { return $this->name; }
    public function setName(string $name): void { $this->name = $name; }
}

class Employee
{
    public function __construct(
        private string $name,
        private string $role,
    ) {}

    public function __clone(): void
    {
        // Employee-specific clone logic
    }
}

// ব্যবহার — পুরো ডিপার্টমেন্ট ক্লোন
$devDept = new Department('ডেভেলপমেন্ট', new Manager('আরিফ', 'CTO'));
$devDept->addEmployee(new Employee('সাকিব', 'Senior Dev'));
$devDept->addEmployee(new Employee('নাফিসা', 'Junior Dev'));

// নতুন শাখায় একই structure — ক্লোন!
$devDeptChittagong = clone $devDept;
// manager পরিবর্তন করলে মূল dept-এ প্রভাব পড়বে না
```

### ২. JavaScript Proxy-based Prototype

```javascript
// Proxy ব্যবহার করে automatic deep clone tracking
function createCloneableProxy(target) {
    const handler = {
        get(obj, prop) {
            if (prop === Symbol.for('clone')) {
                return () => createCloneableProxy(structuredClone(obj));
            }
            if (prop === Symbol.for('isDirty')) {
                return obj.__dirty ?? false;
            }
            return Reflect.get(obj, prop);
        },
        set(obj, prop, value) {
            obj.__dirty = true;
            return Reflect.set(obj, prop, value);
        },
    };
    return new Proxy(target, handler);
}

const config = createCloneableProxy({
    appName: 'BD Digital',
    database: { host: 'localhost', port: 5432 },
    cache: { enabled: true, ttl: 3600 },
});

const devConfig = config[Symbol.for('clone')]();
devConfig.database.host = 'dev-db.local';

console.log(config.database.host);    // 'localhost' — মূল অক্ষত ✅
console.log(devConfig.database.host); // 'dev-db.local'
console.log(devConfig[Symbol.for('isDirty')]); // true
```

### ৩. Prototype vs Factory Method — তুলনা

```
┌─────────────────────────┬────────────────────────────────────────┐
│        মাপকাঠি          │  Prototype       │  Factory Method     │
├─────────────────────────┼──────────────────┼─────────────────────┤
│ অবজেক্ট তৈরির পদ্ধতি    │ Clone (কপি)      │ new (নতুন তৈরি)      │
│ সাবক্লাস দরকার?         │ না               │ হ্যাঁ (সাধারণত)       │
│ Initialization cost     │ কম (clone)       │ বেশি হতে পারে        │
│ Runtime flexibility     │ বেশি             │ কম                  │
│ Complex object          │ সহজে handle করে  │ Builder দরকার হতে পারে│
│ State preservation      │ ক্লোনে পূর্বের     │ শূন্য থেকে শুরু       │
│                         │ state থাকে       │                     │
│ কখন ব্যবহার             │ Costly creation, │ সাবক্লাস ভিত্তিক      │
│                         │ runtime config   │ creation logic       │
└─────────────────────────┴──────────────────┴─────────────────────┘
```

```php
<?php

// Factory Method — প্রতিবার নতুন করে তৈরি
interface NotificationFactory
{
    public function create(string $to, string $msg): Notification;
}

class EmailNotificationFactory implements NotificationFactory
{
    public function create(string $to, string $msg): Notification
    {
        // প্রতিবার SMTP config load, template parse...
        return new EmailNotification($to, $msg);
    }
}

// Prototype — একবার configure, বারবার clone
class NotificationPrototype
{
    public function __construct(
        private string $type,
        private string $template,
        private array $config,
        private ?string $to = null,
        private ?string $message = null,
    ) {}

    public function __clone(): void
    {
        $this->to = null;
        $this->message = null;
    }

    public function prepare(string $to, string $msg): static
    {
        $cloned = clone $this;
        $cloned->to = $to;
        $cloned->message = $msg;
        return $cloned;
    }
}
```

### ৪. JavaScript-এর Prototypal Inheritance ও Prototype Pattern

JavaScript-এ **prototypal inheritance** এবং **Prototype Design Pattern** — এ দুটি আলাদা কিন্তু সম্পর্কিত ধারণা।

```javascript
// ====================================
// JavaScript Prototypal Inheritance
// ====================================
// JS-এ প্রতিটি object-এর একটি [[Prototype]] (internal slot) আছে।
// Object.create() দিয়ে prototype chain তৈরি হয়।

const vehicleProto = {
    type: 'vehicle',
    describe() {
        return `${this.brand} ${this.model} (${this.type})`;
    },
};

// vehicleProto-কে prototype হিসেবে ব্যবহার করে নতুন object
const car = Object.create(vehicleProto, {
    brand: { value: 'Toyota', writable: true, enumerable: true },
    model: { value: 'Corolla', writable: true, enumerable: true },
    type:  { value: 'car', writable: true, enumerable: true },
});

console.log(car.describe()); // "Toyota Corolla (car)"
console.log(Object.getPrototypeOf(car) === vehicleProto); // true

// ====================================
// Prototype Design Pattern in JS
// ====================================
// এটি GoF-এর pattern — clone-based object creation

class Vehicle {
    #brand; #model; #specs;

    constructor({ brand, model, specs }) {
        this.#brand = brand;
        this.#model = model;
        this.#specs = specs;
    }

    clone(overrides = {}) {
        return new Vehicle({
            brand: overrides.brand ?? this.#brand,
            model: overrides.model ?? this.#model,
            specs: structuredClone(overrides.specs ?? this.#specs),
        });
    }

    get info() { return `${this.#brand} ${this.#model}`; }
}

const corolla = new Vehicle({
    brand: 'Toyota',
    model: 'Corolla',
    specs: { engine: '1.8L', transmission: 'CVT' },
});

// clone-based creation
const corollaHybrid = corolla.clone({
    model: 'Corolla Hybrid',
    specs: { engine: '1.8L Hybrid', transmission: 'eCVT' },
});
```

### ৫. Prototype + Flyweight — Shared vs Cloned State

```php
<?php

declare(strict_types=1);

// Flyweight: শেয়ারড intrinsic state
// Prototype: clone-based extrinsic state

class TextureAtlas
{
    // Flyweight — shared, immutable
    private static array $textures = [];

    public static function get(string $name): string
    {
        if (!isset(self::$textures[$name])) {
            // ভারী অপারেশন — একবারই হয়
            self::$textures[$name] = "texture_data_{$name}_" . str_repeat('█', 100);
        }
        return self::$textures[$name];
    }
}

class GameEnemy
{
    private string $instanceId;

    public function __construct(
        private string $type,
        private string $textureName,    // Flyweight reference (শেয়ার্ড)
        private float $x = 0.0,         // Clone-specific (আলাদা)
        private float $y = 0.0,
        private int $health = 100,
    ) {
        $this->instanceId = uniqid('enemy-');
    }

    public function __clone(): void
    {
        $this->instanceId = uniqid('enemy-');
        // textureName শেয়ার্ড থাকবে — Flyweight!
        // x, y, health — primitive, automatic copy হবে
    }

    public function setPosition(float $x, float $y): static
    {
        $this->x = $x;
        $this->y = $y;
        return $this;
    }

    public function getTexture(): string
    {
        return TextureAtlas::get($this->textureName);
    }

    public function getInfo(): string
    {
        return "{$this->instanceId}: {$this->type} @ ({$this->x}, {$this->y}) HP:{$this->health}";
    }
}

// Prototype + Flyweight — কম্বিনেশন
$zombieProto = new GameEnemy('zombie', 'zombie_texture');

// ১০০টি zombie স্পন — সবাই একই texture শেয়ার করে
$enemies = [];
for ($i = 0; $i < 100; $i++) {
    $enemy = clone $zombieProto;
    $enemy->setPosition(
        x: random_int(0, 1000) / 10.0,
        y: random_int(0, 1000) / 10.0,
    );
    $enemies[] = $enemy;
}

echo $enemies[0]->getInfo() . PHP_EOL;
echo $enemies[99]->getInfo() . PHP_EOL;
// সব enemy একই texture data শেয়ার করে (Flyweight)
// কিন্তু position আলাদা (Prototype clone)
```

### ৬. Copy-on-Write (CoW) Optimization

```javascript
// Copy-on-Write: Clone করার সময় আসলে কপি হয় না।
// পরিবর্তন করার সময়ই কপি হয় — memory efficient!

class CowDocument {
    #data;
    #isShared;

    constructor(data, isShared = false) {
        this.#data = data;
        this.#isShared = isShared;
    }

    // "Clone" — আসলে শুধু reference শেয়ার
    clone() {
        this.#isShared = true;
        return new CowDocument(this.#data, true);
    }

    // Write করার সময়ই real copy হয়
    set(key, value) {
        if (this.#isShared) {
            // এখন আসল deep copy হচ্ছে
            this.#data = structuredClone(this.#data);
            this.#isShared = false;
        }
        this.#data[key] = value;
        return this;
    }

    get(key) {
        return this.#data[key];
    }

    get isShared() { return this.#isShared; }
}

const original = new CowDocument({
    title: 'গুরুত্বপূর্ণ ডকুমেন্ট',
    content: 'অনেক বড় ডেটা...'.repeat(10000),
    metadata: { author: 'সাকিব', date: '2024-01-01' },
});

// Clone — প্রায় zero cost, শুধু reference
const draft = original.clone();
console.log(draft.isShared); // true — এখনো shared

// Write — এখন actual copy হবে
draft.set('title', 'ড্রাফট কপি');
console.log(draft.isShared); // false — নিজস্ব কপি পেয়ে গেছে
console.log(original.get('title')); // 'গুরুত্বপূর্ণ ডকুমেন্ট' — মূল অপরিবর্তিত ✅
```

---

## ✅ Pros (সুবিধা)

| সুবিধা | বিবরণ |
|--------|--------|
| **পারফরম্যান্স** | Costly initialization একবারই — বাকি সব clone |
| **Runtime flexibility** | Runtime-এ নতুন object type যোগ করা যায় |
| **Reduced subclassing** | Factory-র মতো subclass hierarchy লাগে না |
| **Complex object সহজে তৈরি** | Builder ছাড়াই pre-configured object পাওয়া যায় |
| **State preservation** | ক্লোন করা object-এ পূর্বের state সংরক্ষিত থাকে |
| **Concrete class থেকে decouple** | Client শুধু Prototype interface জানলেই চলে |
| **Registry pattern সাপোর্ট** | কেন্দ্রীয়ভাবে prototype ম্যানেজমেন্ট |

## ❌ Cons (অসুবিধা)

| অসুবিধা | বিবরণ |
|----------|--------|
| **Deep copy জটিলতা** | Nested objects-এর জন্য manual deep clone দরকার |
| **Circular reference সমস্যা** | A → B → A — infinite loop হতে পারে |
| **__clone / clone method রক্ষণাবেক্ষণ** | নতুন field যোগ হলে __clone আপডেট করতে ভুললে বাগ |
| **Hidden coupling** | Clone-এর মাধ্যমে অপ্রত্যাশিত state শেয়ার হতে পারে |
| **Testing difficulty** | Clone-এর গভীরতা (shallow/deep) পরীক্ষা করা কঠিন |
| **Serialization constraint** | কিছু object serialize করা যায় না |

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ১. Shallow Copy Bug — সবচেয়ে কমন

```php
<?php
// ❌ ভুল — __clone ছাড়া nested object শেয়ার হয়
class Order
{
    public function __construct(
        public string $orderId,
        public Address $shippingAddress,  // object reference!
        public array $items,
    ) {}
    // __clone নেই!
}

$order1 = new Order('ORD-1', new Address('মিরপুর', 'ঢাকা', 'ঢাকা'), ['item1']);
$order2 = clone $order1;
$order2->orderId = 'ORD-2';
$order2->shippingAddress->city = 'চট্টগ্রাম';

// 😱 $order1->shippingAddress->city === 'চট্টগ্রাম' — দুটোই বদলে গেছে!

// ✅ সমাধান
class OrderFixed
{
    public function __construct(
        public string $orderId,
        public Address $shippingAddress,
        public array $items,
    ) {}

    public function __clone(): void
    {
        $this->shippingAddress = clone $this->shippingAddress;
        $this->items = [...$this->items];
    }
}
```

### ২. Circular Reference — Infinite Loop

```php
<?php
class Node
{
    public ?Node $parent = null;
    /** @var Node[] */
    public array $children = [];

    public function __construct(public string $name) {}

    // ❌ ভুল — circular reference-এ infinite loop!
    public function __clone(): void
    {
        // parent clone করলে parent-এর children clone হবে
        // children clone করলে child-এর parent clone হবে → ♾️
        $this->children = array_map(fn($c) => clone $c, $this->children);
    }

    // ✅ সমাধান — visited tracking
    public function deepClone(array &$visited = []): static
    {
        $id = spl_object_id($this);
        if (isset($visited[$id])) {
            return $visited[$id];
        }

        $cloned = clone $this;
        $visited[$id] = $cloned;

        $cloned->children = array_map(
            fn(Node $child) => $child->deepClone($visited),
            $this->children
        );

        if ($this->parent !== null) {
            $parentId = spl_object_id($this->parent);
            $cloned->parent = $visited[$parentId] ?? $this->parent->deepClone($visited);
        }

        return $cloned;
    }
}
```

### ৩. JavaScript-এ Circular Reference

```javascript
// structuredClone circular reference handle করে!
const obj = { name: 'root' };
obj.self = obj; // circular!

const cloned = structuredClone(obj); // ✅ কাজ করে!
console.log(cloned.self === cloned); // true — correctly circular

// কিন্তু JSON.parse/stringify ব্যর্থ হয়:
// JSON.stringify(obj); // ❌ TypeError: Converting circular structure to JSON

// Custom class-এ circular reference handle
class TreeNode {
    #name; #parent; #children;

    constructor(name) {
        this.#name = name;
        this.#parent = null;
        this.#children = [];
    }

    addChild(node) {
        node.#parent = this;
        this.#children.push(node);
        return this;
    }

    clone(visited = new Map()) {
        if (visited.has(this)) return visited.get(this);

        const cloned = new TreeNode(this.#name);
        visited.set(this, cloned);

        cloned.#children = this.#children.map(c => c.clone(visited));
        if (this.#parent && visited.has(this.#parent)) {
            cloned.#parent = visited.get(this.#parent);
        }

        return cloned;
    }
}
```

### ৪. Clone-এ নতুন Property যোগ করতে ভোলা

```php
<?php
// ❌ মাসের পর মাস ক্লাসে নতুন property যোগ হয়, কিন্তু __clone আপডেট হয় না
class UserProfile
{
    public function __construct(
        public string $name,
        public Address $address,
        public array $preferences,
        // ৬ মাস পরে যোগ হলো:
        public ?PaymentMethod $paymentMethod = null,  // __clone-এ নেই!
    ) {}

    public function __clone(): void
    {
        $this->address = clone $this->address;
        $this->preferences = [...$this->preferences];
        // ❌ paymentMethod clone করা হয়নি!
    }
}

// ✅ সমাধান: Reflection-based automatic deep clone
function autoDeepClone(object $obj): object
{
    $cloned = clone $obj;
    $reflection = new \ReflectionObject($cloned);

    foreach ($reflection->getProperties() as $prop) {
        $prop->setAccessible(true);
        $value = $prop->getValue($cloned);

        if (is_object($value)) {
            $prop->setValue($cloned, clone $value);
        } elseif (is_array($value)) {
            $prop->setValue($cloned, array_map(
                fn($v) => is_object($v) ? clone $v : $v,
                $value
            ));
        }
    }

    return $cloned;
}
```

---

## 🧪 টেস্টিং

### PHPUnit — Prototype Testing

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class InvoicePrototypeTest extends TestCase
{
    private Invoice $masterInvoice;

    protected function setUp(): void
    {
        $this->masterInvoice = new Invoice(
            companyName: 'টেস্ট কোম্পানি',
            companyAddress: 'ঢাকা',
            vatNumber: 'VAT-TEST',
            binNumber: 'BIN-TEST',
            bankDetails: 'টেস্ট ব্যাংক',
            defaultTerms: ['টার্ম ১', 'টার্ম ২'],
        );
    }

    public function testCloneCreatesNewInstance(): void
    {
        $cloned = $this->masterInvoice->clone();
        $this->assertNotSame($this->masterInvoice, $cloned);
    }

    public function testCloneHasDifferentId(): void
    {
        $clone1 = $this->masterInvoice->clone();
        $clone2 = $this->masterInvoice->clone();

        $summary1 = $clone1->getSummary();
        $summary2 = $clone2->getSummary();

        $this->assertNotEquals($summary1['id'], $summary2['id']);
    }

    public function testClonePreservesCompanyInfo(): void
    {
        $cloned = $this->masterInvoice->clone();
        $summary = $cloned->getSummary();

        $this->assertEquals('টেস্ট কোম্পানি', $summary['company']);
        $this->assertEquals('VAT-TEST', $summary['vat']);
    }

    public function testCloneResetsCustomerSpecificData(): void
    {
        $original = $this->masterInvoice->clone()
            ->setCustomer('আসল ক্রেতা')
            ->addLineItem('পণ্য', 1000.0, 1);

        // original থেকে নয়, master থেকে clone
        $fresh = $this->masterInvoice->clone();
        $summary = $fresh->getSummary();

        $this->assertNull($summary['customer']);
        $this->assertEquals(0, $summary['items']);
    }

    public function testDeepCloneIndependence(): void
    {
        $product = new ProductDeep(
            name: 'ফোন',
            price: 25000.0,
            warehouse: new Address('মতিঝিল', 'ঢাকা', 'ঢাকা'),
            tags: ['ইলেকট্রনিক্স'],
        );

        $cloned = clone $product;
        $cloned->warehouse->city = 'চট্টগ্রাম';

        // মূল product-এর address অপরিবর্তিত
        $this->assertEquals('ঢাকা', $product->warehouse->city);
        $this->assertEquals('চট্টগ্রাম', $cloned->warehouse->city);
    }

    public function testPrototypeRegistryReturnsClone(): void
    {
        $registry = new PrototypeRegistry();
        $registry->register('test', new DocumentTemplate(
            type: 'টেস্ট',
            header: 'H',
            footer: 'F',
            fontFamily: 'Arial',
            fontSize: 12.0,
        ));

        $doc1 = $registry->get('test');
        $doc2 = $registry->get('test');

        $this->assertNotSame($doc1, $doc2);
        $this->assertInstanceOf(DocumentTemplate::class, $doc1);
    }

    public function testRegistryThrowsForUnknownKey(): void
    {
        $registry = new PrototypeRegistry();

        $this->expectException(\InvalidArgumentException::class);
        $registry->get('nonexistent');
    }

    public function testCircularReferenceClone(): void
    {
        $parent = new Node('root');
        $child = new Node('child');
        $parent->children[] = $child;
        $child->parent = $parent;

        $clonedParent = $parent->deepClone();

        $this->assertEquals('root', $clonedParent->name);
        $this->assertNotSame($parent, $clonedParent);
        $this->assertCount(1, $clonedParent->children);
        $this->assertNotSame($child, $clonedParent->children[0]);
    }
}
```

### Jest — Prototype Testing (JavaScript)

```javascript
import { describe, it, expect, beforeEach } from '@jest/globals';

describe('Prototype Pattern', () => {
    const CLONE = Symbol('clone');

    class TestInvoice {
        #company; #vat; #customer; #items; #id;

        constructor({ company, vat, customer = null, items = [] }) {
            this.#company = company;
            this.#vat = vat;
            this.#customer = customer;
            this.#items = [...items];
            this.#id = crypto.randomUUID();
        }

        [CLONE]() {
            return new TestInvoice({
                company: this.#company,
                vat: this.#vat,
                customer: null,
                items: [],
            });
        }

        setCustomer(name) { this.#customer = name; return this; }
        addItem(item) { this.#items.push(item); return this; }

        get id() { return this.#id; }
        get company() { return this.#company; }
        get customer() { return this.#customer; }
        get itemCount() { return this.#items.length; }
    }

    let masterInvoice;

    beforeEach(() => {
        masterInvoice = new TestInvoice({
            company: 'টেস্ট কোম্পানি',
            vat: 'VAT-TEST',
        });
    });

    it('clone তৈরি করলে নতুন instance পাওয়া যায়', () => {
        const cloned = masterInvoice[CLONE]();
        expect(cloned).not.toBe(masterInvoice);
        expect(cloned).toBeInstanceOf(TestInvoice);
    });

    it('clone-এ আলাদা ID থাকে', () => {
        const clone1 = masterInvoice[CLONE]();
        const clone2 = masterInvoice[CLONE]();
        expect(clone1.id).not.toBe(clone2.id);
    });

    it('clone company info সংরক্ষণ করে', () => {
        const cloned = masterInvoice[CLONE]();
        expect(cloned.company).toBe('টেস্ট কোম্পানি');
    });

    it('clone customer-specific data রিসেট করে', () => {
        masterInvoice.setCustomer('ক্রেতা').addItem({ name: 'পণ্য' });

        const fresh = masterInvoice[CLONE]();
        expect(fresh.customer).toBeNull();
        expect(fresh.itemCount).toBe(0);
    });

    describe('Shallow vs Deep Copy', () => {
        it('spread operator shallow copy করে', () => {
            const obj = { a: 1, nested: { b: 2 } };
            const shallow = { ...obj };

            shallow.nested.b = 99;
            expect(obj.nested.b).toBe(99); // মূলও বদলে গেছে!
        });

        it('structuredClone deep copy করে', () => {
            const obj = { a: 1, nested: { b: 2 } };
            const deep = structuredClone(obj);

            deep.nested.b = 99;
            expect(obj.nested.b).toBe(2); // মূল অক্ষত ✅
        });

        it('structuredClone circular reference handle করে', () => {
            const obj = { name: 'root' };
            obj.self = obj;

            const cloned = structuredClone(obj);
            expect(cloned.self).toBe(cloned);
            expect(cloned.self).not.toBe(obj);
        });

        it('structuredClone Date সঠিকভাবে clone করে', () => {
            const date = new Date('2024-06-15');
            const obj = { created: date };

            const cloned = structuredClone(obj);
            expect(cloned.created).toEqual(date);
            expect(cloned.created).not.toBe(date);
        });

        it('structuredClone Map/Set clone করে', () => {
            const map = new Map([['key', 'value']]);
            const cloned = structuredClone(map);

            cloned.set('key', 'modified');
            expect(map.get('key')).toBe('value'); // মূল অক্ষত ✅
        });
    });

    describe('Prototype Registry', () => {
        class SimpleRegistry {
            #map = new Map();

            register(key, proto) {
                this.#map.set(key, proto);
                return this;
            }

            get(key) {
                const proto = this.#map.get(key);
                if (!proto) throw new RangeError(`'${key}' পাওয়া যায়নি`);
                return proto[CLONE]();
            }

            has(key) { return this.#map.has(key); }
        }

        it('রেজিস্ট্রি থেকে clone পাওয়া যায়', () => {
            const reg = new SimpleRegistry();
            reg.register('inv', masterInvoice);

            const doc = reg.get('inv');
            expect(doc).toBeInstanceOf(TestInvoice);
            expect(doc).not.toBe(masterInvoice);
        });

        it('অজানা key-তে error throw করে', () => {
            const reg = new SimpleRegistry();
            expect(() => reg.get('unknown')).toThrow(RangeError);
        });
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্নসমূহ

### ১. Factory Method ↔ Prototype
- **Factory Method** subclass-ভিত্তিক — প্রতিটি product type-এর জন্য আলাদা factory।
- **Prototype** clone-ভিত্তিক — registry-তে prototype রেখে clone। Subclass explosion এড়ায়।
- দুটো একসাথে ব্যবহার করা যায়: Factory registry-তে prototype রেখে clone-based creation।

### ২. Abstract Factory ↔ Prototype
- **Abstract Factory** দিয়ে concrete prototype তৈরি করা যায়।
- Prototype registry একটি সরলীকৃত Abstract Factory হিসেবে কাজ করতে পারে।

### ৩. Memento ↔ Prototype
- **Memento** অবজেক্টের state snapshot সংরক্ষণ করে।
- Prototype-এর clone আসলে একটি memento-র মতো কাজ করে — পূর্বের state সংরক্ষণ।
- Undo/Redo ফিচারে দুটো একসাথে ব্যবহৃত হয়।

### ৪. Flyweight ↔ Prototype
- **Flyweight** shared (intrinsic) state আলাদা রাখে।
- **Prototype** clone করে unique (extrinsic) state তৈরি করে।
- Game development-এ দুটো মিলিয়ে ব্যবহার হয় (shared textures + unique positions)।

### ৫. Builder ↔ Prototype
- জটিল object তৈরিতে **Builder** step-by-step approach নেয়।
- **Prototype** pre-built object clone করে — Builder-এর বিকল্প যখন initialization costly।

```
┌──────────────────┬──────────────────────────────────────────────┐
│ প্যাটার্ন          │ Prototype-এর সাথে সম্পর্ক                     │
├──────────────────┼──────────────────────────────────────────────┤
│ Factory Method   │ বিকল্প — clone vs subclass-based creation    │
│ Abstract Factory │ Prototype registry = simplified AF           │
│ Memento          │ Clone = state snapshot                       │
│ Flyweight        │ Shared state + cloned unique state           │
│ Builder          │ Pre-built clone vs step-by-step build        │
│ Singleton        │ বিপরীত — একটি instance vs অনেক clone          │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

1. **অবজেক্ট তৈরি ব্যয়বহুল** — DB query, API call, file parsing প্রয়োজন হয়
2. **Runtime-এ dynamic object creation** — ব্যবহারকারী config-এ নতুন type যোগ করতে পারে
3. **Complex initialization** — অনেক dependency injection, configuration step আছে
4. **Similar objects with slight variations** — ই-কমার্সে পণ্যের variant
5. **State snapshot দরকার** — undo/redo, versioning
6. **Subclass explosion এড়াতে** — ১০০টি ভিন্ন configuration-এর জন্য ১০০টি subclass বানাতে চান না
7. **Template-based object creation** — document template, email template, invoice template

### ❌ ব্যবহার করবেন না যখন:

1. **অবজেক্ট তৈরি সস্তা** — `new` keyword যথেষ্ট, clone-এর overhead অপ্রয়োজনীয়
2. **কোনো mutable state নেই** — immutable object-এ clone-এর দরকার নেই, শেয়ার করলেই চলে
3. **Deep copy অত্যন্ত জটিল** — circular reference, external resource handle (file, DB connection)
4. **Object graph বিশাল** — গিগাবাইট ডেটা clone করা practical না
5. **Clone-এর পর ৯০% property বদলাতে হয়** — তখন `new` করাই সহজ
6. **Immutable value objects** — `readonly` class/record-এ clone অর্থহীন

### সিদ্ধান্ত গ্রাফ

```
                    অবজেক্ট তৈরি কি costly?
                    /                    \
                  হ্যাঁ                    না
                  /                        \
         Similar objects                 new ব্যবহার করুন
         দরকার?
         /        \
       হ্যাঁ        না
       /            \
  Prototype      Factory Method
  ব্যবহার করুন    বিবেচনা করুন

         Clone-এর পর কতটুকু
         পরিবর্তন দরকার?
         /              \
     অল্প (< 30%)     বেশি (> 70%)
       /                  \
   ✅ Prototype          ❌ new/Builder
   perfect!              ভালো হবে
```

---

## 📋 সারসংক্ষেপ

### মূল বিষয়সমূহ

| বিষয় | সংক্ষেপ |
|-------|---------|
| **উদ্দেশ্য** | বিদ্যমান object clone করে নতুন object তৈরি |
| **মূল সুবিধা** | Costly creation avoid, runtime flexibility |
| **মূল ঝুঁকি** | Shallow copy bug, circular reference |
| **PHP keyword** | `clone`, `__clone()` magic method |
| **JS পদ্ধতি** | `structuredClone()`, spread, `Object.create()` |
| **সেরা ব্যবহার** | Template cloning, game spawning, config preset |
| **সাথে ভালো যায়** | Registry, Flyweight, Factory Method |
| **বিকল্প** | Factory Method, Builder, Abstract Factory |

### চেকলিস্ট — Prototype সঠিকভাবে ইমপ্লিমেন্ট করছেন কি?

- [ ] `clone()` মেথড / `__clone()` ম্যাজিক মেথড আছে?
- [ ] Nested objects deep clone হচ্ছে?
- [ ] Unique identifiers (ID, timestamp) রিসেট হচ্ছে?
- [ ] Circular reference handle করা হয়েছে?
- [ ] নতুন property যোগ হলে `__clone()` আপডেট হচ্ছে?
- [ ] Unit test-এ shallow vs deep copy verify হচ্ছে?
- [ ] Prototype Registry ব্যবহার হচ্ছে (যদি একাধিক type থাকে)?
- [ ] Copy-on-Write বিবেচনা করা হয়েছে (performance-critical ক্ষেত্রে)?

### এক লাইনে মনে রাখুন

> **"একবার বানাও, বারবার কপি করো, শুধু যা আলাদা তা পরিবর্তন করো।"**

---

*এই ডকুমেন্টটি বাংলাদেশের সফটওয়্যার ডেভেলপারদের জন্য তৈরি। উদাহরণগুলোতে বাংলাদেশী ব্যবসায়িক প্রসঙ্গ (চালান, VAT, BIN, বিকাশ/নগদ) ব্যবহার করা হয়েছে।*
