# 🔨 Builder প্যাটার্ন (Builder Pattern)

## 📌 সংজ্ঞা ও পরিচিতি

> **GoF সংজ্ঞা:** *"Separate the construction of a complex object from its representation so that the same construction process can create different representations."*

Builder প্যাটার্ন হলো একটি **Creational Design Pattern** যা জটিল অবজেক্ট তৈরির প্রক্রিয়াকে তার representation থেকে আলাদা করে। এর মূল উদ্দেশ্য হলো — একই construction process ব্যবহার করে ভিন্ন ভিন্ন representation তৈরি করা।

### 🤔 সমস্যা কী?

ধরুন আপনি একটি e-commerce অর্ডার সিস্টেম তৈরি করছেন। একটি অর্ডারে থাকতে পারে:
- প্রোডাক্ট তালিকা
- শিপিং অ্যাড্রেস
- বিলিং অ্যাড্রেস
- কুপন কোড
- পেমেন্ট মেথড (bKash, Nagad, COD)
- গিফট র‍্যাপিং
- ডেলিভারি নোট
- ইনভয়েস প্রেফারেন্স

এত প্যারামিটার দিয়ে constructor বানালে কোড হয়ে যায় অপঠনযোগ্য এবং error-prone। এটাই **Telescoping Constructor Problem** — Builder প্যাটার্ন এর সমাধান।

### 🎯 মূল নীতি

```
১. Complex object → step-by-step নির্মাণ
২. Construction process → Product থেকে আলাদা
৩. একই process → ভিন্ন output
৪. Client কোড → নির্মাণ বিস্তারিত জানে না
```

---

## 🏠 বাস্তব উদাহরণ — বাড়ি নির্মাণ

কল্পনা করুন একজন ঠিকাদার (Director) বাড়ি বানাচ্ছেন। তিনি একই blueprint অনুসরণ করেন, কিন্তু:

```
🏗️ ঠিকাদার (Director): "আমি বলি কী বানাতে হবে"
    │
    ├── 🧱 ইটের মিস্ত্রি (ConcreteBuilder A):
    │       → ইটের দেয়াল, টিনের ছাদ = গ্রামের বাড়ি
    │
    ├── 🪨 কংক্রিট মিস্ত্রি (ConcreteBuilder B):
    │       → কংক্রিটের দেয়াল, RCC ছাদ = শহরের বিল্ডিং
    │
    └── 🪵 কাঠমিস্ত্রি (ConcreteBuilder C):
            → কাঠের দেয়াল, কাঠের ছাদ = পাহাড়ি কটেজ

🏠 প্রতিটি ক্ষেত্রে প্রক্রিয়া একই:
   ভিত্তি → দেয়াল → ছাদ → দরজা-জানালা → ফিনিশিং

🎯 কিন্তু ফলাফল ভিন্ন!
```

### বাংলাদেশ কনটেক্সট — bKash ট্রানজ্যাকশন

```
📱 bKash Transaction Builder:
    ├── setSender("01712345678")
    ├── setReceiver("01898765432")
    ├── setAmount(5000)
    ├── setReference("বাড়ি ভাড়া")
    ├── setPin("*****")
    └── build() → Transaction অবজেক্ট

✅ প্রতিটি ধাপ পরিষ্কার, ভুল হওয়ার সম্ভাবনা কম
```

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────┐
│                        Builder Pattern UML                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐         ┌─────────────────────────┐           │
│  │   Director    │         │    <<interface>>         │           │
│  │──────────────│ uses    │       Builder            │           │
│  │ - builder    │────────▶│─────────────────────────│           │
│  │──────────────│         │ + buildPartA(): Builder  │           │
│  │ + construct()│         │ + buildPartB(): Builder  │           │
│  │ + make___()  │         │ + buildPartC(): Builder  │           │
│  └──────────────┘         │ + getResult(): Product   │           │
│                            └────────────┬────────────┘           │
│                                  ▲      │      ▲                 │
│                    ┌─────────────┘      │      └──────────────┐ │
│          ┌─────────┴──────────┐         │   ┌─────────┴──────┐ │
│          │ ConcreteBuilder A  │         │   │ ConcreteBuilder B│ │
│          │───────────────────│         │   │────────────────-│ │
│          │ - product: Product │         │   │ - product       │ │
│          │───────────────────│         │   │────────────────-│ │
│          │ + buildPartA()    │         │   │ + buildPartA()  │ │
│          │ + buildPartB()    │         │   │ + buildPartB()  │ │
│          │ + buildPartC()    │         │   │ + buildPartC()  │ │
│          │ + getResult()     │         │   │ + getResult()   │ │
│          └───────────────────┘         │   └────────────────┘ │
│                                         │                       │
│                               ┌─────────┴─────────┐            │
│                               │     Product        │            │
│                               │───────────────────│            │
│                               │ - partA            │            │
│                               │ - partB            │            │
│                               │ - partC            │            │
│                               └───────────────────┘            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Flow ডায়াগ্রাম:

  Client → Director.construct()
              │
              ├── builder.buildPartA()
              ├── builder.buildPartB()
              ├── builder.buildPartC()
              │
              └── builder.getResult() → Product ✅
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Classic Builder with Director

এটি GoF-এর মূল প্যাটার্ন। Director নির্মাণ প্রক্রিয়া নিয়ন্ত্রণ করে, Builder নির্দিষ্ট পার্ট তৈরি করে।

#### PHP 8.3 — Classic Builder

```php
<?php

declare(strict_types=1);

// Product — চূড়ান্ত অবজেক্ট যা তৈরি হবে
class House
{
    private array $parts = [];

    public function addPart(string $name, string $detail): void
    {
        $this->parts[$name] = $detail;
    }

    public function describe(): string
    {
        $description = "🏠 বাড়ির বিবরণ:\n";
        foreach ($this->parts as $name => $detail) {
            $description .= "  ├── {$name}: {$detail}\n";
        }
        return $description;
    }

    public function getParts(): array
    {
        return $this->parts;
    }
}

// Builder Interface
interface HouseBuilder
{
    public function reset(): static;
    public function buildFoundation(): static;
    public function buildWalls(): static;
    public function buildRoof(): static;
    public function buildWindows(): static;
    public function buildDoors(): static;
    public function getResult(): House;
}

// ConcreteBuilder A — গ্রামের মাটির বাড়ি
class VillageHouseBuilder implements HouseBuilder
{
    private House $house;

    public function __construct()
    {
        $this->house = new House();
    }

    public function reset(): static
    {
        $this->house = new House();
        return $this;
    }

    public function buildFoundation(): static
    {
        $this->house->addPart('ভিত্তি', 'মাটির ভিত্তি (৩ ফুট গভীর)');
        return $this;
    }

    public function buildWalls(): static
    {
        $this->house->addPart('দেয়াল', 'ইটের দেয়াল (১০ ইঞ্চি)');
        return $this;
    }

    public function buildRoof(): static
    {
        $this->house->addPart('ছাদ', 'টিনের চালা');
        return $this;
    }

    public function buildWindows(): static
    {
        $this->house->addPart('জানালা', 'কাঠের জানালা (৪টি)');
        return $this;
    }

    public function buildDoors(): static
    {
        $this->house->addPart('দরজা', 'কাঠের দরজা (২টি)');
        return $this;
    }

    public function getResult(): House
    {
        $house = $this->house;
        $this->reset();
        return $house;
    }
}

// ConcreteBuilder B — শহরের আধুনিক বিল্ডিং
class ModernBuildingBuilder implements HouseBuilder
{
    private House $house;

    public function __construct()
    {
        $this->house = new House();
    }

    public function reset(): static
    {
        $this->house = new House();
        return $this;
    }

    public function buildFoundation(): static
    {
        $this->house->addPart('ভিত্তি', 'RCC পাইলিং ফাউন্ডেশন (২০ ফুট গভীর)');
        return $this;
    }

    public function buildWalls(): static
    {
        $this->house->addPart('দেয়াল', 'কংক্রিট শিয়ার ওয়াল + গ্লাস ফ্যাসাড');
        return $this;
    }

    public function buildRoof(): static
    {
        $this->house->addPart('ছাদ', 'ফ্ল্যাট RCC রুফ + ওয়াটারপ্রুফিং');
        return $this;
    }

    public function buildWindows(): static
    {
        $this->house->addPart('জানালা', 'অ্যালুমিনিয়াম স্লাইডিং উইন্ডো (২০টি)');
        return $this;
    }

    public function buildDoors(): static
    {
        $this->house->addPart('দরজা', 'ফায়ার-রেটেড স্টিল ডোর (১০টি)');
        return $this;
    }

    public function getResult(): House
    {
        $house = $this->house;
        $this->reset();
        return $house;
    }
}

// Director — নির্মাণ প্রক্রিয়া পরিচালনা করে
class ConstructionDirector
{
    public function __construct(
        private HouseBuilder $builder
    ) {}

    public function setBuilder(HouseBuilder $builder): void
    {
        $this->builder = $builder;
    }

    // সম্পূর্ণ বাড়ি তৈরি
    public function buildFullHouse(): House
    {
        return $this->builder
            ->reset()
            ->buildFoundation()
            ->buildWalls()
            ->buildRoof()
            ->buildWindows()
            ->buildDoors()
            ->getResult();
    }

    // শুধু কাঠামো (ছাদ + দেয়াল)
    public function buildShell(): House
    {
        return $this->builder
            ->reset()
            ->buildFoundation()
            ->buildWalls()
            ->buildRoof()
            ->getResult();
    }
}

// ব্যবহার
$director = new ConstructionDirector(new VillageHouseBuilder());
$villageHouse = $director->buildFullHouse();
echo $villageHouse->describe();

$director->setBuilder(new ModernBuildingBuilder());
$modernBuilding = $director->buildFullHouse();
echo $modernBuilding->describe();
```

#### JavaScript ES2022+ — Classic Builder

```javascript
// Product
class House {
    #parts = new Map();

    addPart(name, detail) {
        this.#parts.set(name, detail);
        return this;
    }

    describe() {
        let desc = "🏠 বাড়ির বিবরণ:\n";
        for (const [name, detail] of this.#parts) {
            desc += `  ├── ${name}: ${detail}\n`;
        }
        return desc;
    }

    get parts() {
        return Object.fromEntries(this.#parts);
    }
}

// Builder Interface (JS তে class + JSDoc দিয়ে enforce করি)
/**
 * @interface HouseBuilder
 * @method reset(): this
 * @method buildFoundation(): this
 * @method buildWalls(): this
 * @method buildRoof(): this
 * @method buildWindows(): this
 * @method buildDoors(): this
 * @method getResult(): House
 */

// ConcreteBuilder A
class VillageHouseBuilder {
    #house = new House();

    reset() {
        this.#house = new House();
        return this;
    }

    buildFoundation() {
        this.#house.addPart('ভিত্তি', 'মাটির ভিত্তি (৩ ফুট গভীর)');
        return this;
    }

    buildWalls() {
        this.#house.addPart('দেয়াল', 'ইটের দেয়াল (১০ ইঞ্চি)');
        return this;
    }

    buildRoof() {
        this.#house.addPart('ছাদ', 'টিনের চালা');
        return this;
    }

    buildWindows() {
        this.#house.addPart('জানালা', 'কাঠের জানালা (৪টি)');
        return this;
    }

    buildDoors() {
        this.#house.addPart('দরজা', 'কাঠের দরজা (২টি)');
        return this;
    }

    getResult() {
        const house = this.#house;
        this.reset();
        return house;
    }
}

// ConcreteBuilder B
class ModernBuildingBuilder {
    #house = new House();

    reset() {
        this.#house = new House();
        return this;
    }

    buildFoundation() {
        this.#house.addPart('ভিত্তি', 'RCC পাইলিং ফাউন্ডেশন');
        return this;
    }

    buildWalls() {
        this.#house.addPart('দেয়াল', 'কংক্রিট + গ্লাস ফ্যাসাড');
        return this;
    }

    buildRoof() {
        this.#house.addPart('ছাদ', 'ফ্ল্যাট RCC রুফ');
        return this;
    }

    buildWindows() {
        this.#house.addPart('জানালা', 'অ্যালুমিনিয়াম স্লাইডিং (২০টি)');
        return this;
    }

    buildDoors() {
        this.#house.addPart('দরজা', 'স্টিল ডোর (১০টি)');
        return this;
    }

    getResult() {
        const house = this.#house;
        this.reset();
        return house;
    }
}

// Director
class ConstructionDirector {
    #builder;

    constructor(builder) {
        this.#builder = builder;
    }

    setBuilder(builder) {
        this.#builder = builder;
    }

    buildFullHouse() {
        return this.#builder
            .reset()
            .buildFoundation()
            .buildWalls()
            .buildRoof()
            .buildWindows()
            .buildDoors()
            .getResult();
    }

    buildShell() {
        return this.#builder
            .reset()
            .buildFoundation()
            .buildWalls()
            .buildRoof()
            .getResult();
    }
}

// ব্যবহার
const director = new ConstructionDirector(new VillageHouseBuilder());
console.log(director.buildFullHouse().describe());

director.setBuilder(new ModernBuildingBuilder());
console.log(director.buildFullHouse().describe());
```

---

### ২. Fluent Builder / Method Chaining

Fluent Builder হলো আধুনিক সফটওয়্যারে সবচেয়ে জনপ্রিয় Builder ভ্যারিয়েন্ট। প্রতিটি setter মেথড `$this` / `this` রিটার্ন করে, ফলে চেইনিং সম্ভব হয়।

#### PHP 8.3 — Fluent Builder (bKash Transaction)

```php
<?php

declare(strict_types=1);

// বাংলাদেশ কনটেক্সট: bKash ট্রানজ্যাকশন বিল্ডার

readonly class BkashTransaction
{
    public function __construct(
        public string $sender,
        public string $receiver,
        public float $amount,
        public string $type,           // SendMoney, Payment, CashOut
        public ?string $reference = null,
        public ?string $merchantId = null,
        public float $charge = 0.0,
        public string $currency = 'BDT',
        public \DateTimeImmutable $timestamp = new \DateTimeImmutable(),
    ) {}

    public function summary(): string
    {
        return sprintf(
            "📱 bKash %s\n  প্রেরক: %s → প্রাপক: %s\n  পরিমাণ: ৳%.2f (চার্জ: ৳%.2f)\n  রেফারেন্স: %s",
            $this->type,
            $this->sender,
            $this->receiver,
            $this->amount,
            $this->charge,
            $this->reference ?? 'N/A'
        );
    }
}

class BkashTransactionBuilder
{
    private string $sender = '';
    private string $receiver = '';
    private float $amount = 0.0;
    private string $type = 'SendMoney';
    private ?string $reference = null;
    private ?string $merchantId = null;
    private float $charge = 0.0;
    private string $currency = 'BDT';

    // বাধ্যতামূলক ফিল্ডগুলো ট্র্যাক করি
    private array $requiredFields = ['sender', 'receiver', 'amount'];
    private array $setFields = [];

    public static function create(): static
    {
        return new static();
    }

    public function from(string $sender): static
    {
        if (!preg_match('/^01[3-9]\d{8}$/', $sender)) {
            throw new \InvalidArgumentException("অবৈধ বাংলাদেশি মোবাইল নম্বর: {$sender}");
        }
        $this->sender = $sender;
        $this->setFields[] = 'sender';
        return $this;
    }

    public function to(string $receiver): static
    {
        if (!preg_match('/^01[3-9]\d{8}$/', $receiver)) {
            throw new \InvalidArgumentException("অবৈধ প্রাপক নম্বর: {$receiver}");
        }
        $this->receiver = $receiver;
        $this->setFields[] = 'receiver';
        return $this;
    }

    public function amount(float $amount): static
    {
        if ($amount <= 0 || $amount > 25000) {
            throw new \InvalidArgumentException("পরিমাণ ১-২৫,০০০ টাকার মধ্যে হতে হবে");
        }
        $this->amount = $amount;
        $this->setFields[] = 'amount';
        return $this;
    }

    public function asSendMoney(): static
    {
        $this->type = 'SendMoney';
        $this->charge = $this->calculateCharge('SendMoney');
        return $this;
    }

    public function asPayment(string $merchantId): static
    {
        $this->type = 'Payment';
        $this->merchantId = $merchantId;
        $this->charge = 0.0; // মার্চেন্ট পেমেন্টে চার্জ নেই
        return $this;
    }

    public function asCashOut(): static
    {
        $this->type = 'CashOut';
        $this->charge = $this->calculateCharge('CashOut');
        return $this;
    }

    public function withReference(string $reference): static
    {
        $this->reference = $reference;
        return $this;
    }

    private function calculateCharge(string $type): float
    {
        return match ($type) {
            'SendMoney' => min($this->amount * 0.01, 25.0),  // ১% সর্বোচ্চ ২৫ টাকা
            'CashOut'   => $this->amount * 0.0185,            // ১.৮৫%
            default     => 0.0,
        };
    }

    public function build(): BkashTransaction
    {
        // বাধ্যতামূলক ফিল্ড ভ্যালিডেশন
        $missing = array_diff($this->requiredFields, array_unique($this->setFields));
        if (!empty($missing)) {
            throw new \LogicException(
                'বাধ্যতামূলক ফিল্ড অনুপস্থিত: ' . implode(', ', $missing)
            );
        }

        if ($this->sender === $this->receiver) {
            throw new \LogicException('প্রেরক ও প্রাপক একই হতে পারবে না');
        }

        // চার্জ পুনঃগণনা (amount সেট হওয়ার পর type সেট হলে)
        $this->charge = $this->calculateCharge($this->type);

        return new BkashTransaction(
            sender: $this->sender,
            receiver: $this->receiver,
            amount: $this->amount,
            type: $this->type,
            reference: $this->reference,
            merchantId: $this->merchantId,
            charge: $this->charge,
            currency: $this->currency,
        );
    }
}

// ব্যবহার
$transaction = BkashTransactionBuilder::create()
    ->from('01712345678')
    ->to('01898765432')
    ->amount(5000)
    ->asSendMoney()
    ->withReference('বাড়ি ভাড়া - জানুয়ারি')
    ->build();

echo $transaction->summary();
```

#### JavaScript ES2022+ — Fluent Builder (bKash Transaction)

```javascript
class BkashTransaction {
    #data;

    constructor(data) {
        this.#data = Object.freeze({ ...data });
    }

    get sender()    { return this.#data.sender; }
    get receiver()  { return this.#data.receiver; }
    get amount()    { return this.#data.amount; }
    get type()      { return this.#data.type; }
    get charge()    { return this.#data.charge; }
    get reference() { return this.#data.reference; }

    summary() {
        return [
            `📱 bKash ${this.type}`,
            `  প্রেরক: ${this.sender} → প্রাপক: ${this.receiver}`,
            `  পরিমাণ: ৳${this.amount.toFixed(2)} (চার্জ: ৳${this.charge.toFixed(2)})`,
            `  রেফারেন্স: ${this.reference ?? 'N/A'}`,
        ].join('\n');
    }
}

class BkashTransactionBuilder {
    #sender = '';
    #receiver = '';
    #amount = 0;
    #type = 'SendMoney';
    #reference = null;
    #merchantId = null;
    #charge = 0;
    #setFields = new Set();

    static create() {
        return new BkashTransactionBuilder();
    }

    #validatePhone(number) {
        if (!/^01[3-9]\d{8}$/.test(number)) {
            throw new Error(`অবৈধ মোবাইল নম্বর: ${number}`);
        }
    }

    from(sender) {
        this.#validatePhone(sender);
        this.#sender = sender;
        this.#setFields.add('sender');
        return this;
    }

    to(receiver) {
        this.#validatePhone(receiver);
        this.#receiver = receiver;
        this.#setFields.add('receiver');
        return this;
    }

    amount(amount) {
        if (amount <= 0 || amount > 25000) {
            throw new RangeError('পরিমাণ ১-২৫,০০০ টাকার মধ্যে হতে হবে');
        }
        this.#amount = amount;
        this.#setFields.add('amount');
        return this;
    }

    asSendMoney() {
        this.#type = 'SendMoney';
        return this;
    }

    asPayment(merchantId) {
        this.#type = 'Payment';
        this.#merchantId = merchantId;
        return this;
    }

    asCashOut() {
        this.#type = 'CashOut';
        return this;
    }

    withReference(ref) {
        this.#reference = ref;
        return this;
    }

    #calculateCharge() {
        switch (this.#type) {
            case 'SendMoney': return Math.min(this.#amount * 0.01, 25.0);
            case 'CashOut':   return this.#amount * 0.0185;
            default:          return 0;
        }
    }

    build() {
        const required = ['sender', 'receiver', 'amount'];
        const missing = required.filter(f => !this.#setFields.has(f));

        if (missing.length > 0) {
            throw new Error(`বাধ্যতামূলক ফিল্ড অনুপস্থিত: ${missing.join(', ')}`);
        }

        if (this.#sender === this.#receiver) {
            throw new Error('প্রেরক ও প্রাপক একই হতে পারবে না');
        }

        return new BkashTransaction({
            sender: this.#sender,
            receiver: this.#receiver,
            amount: this.#amount,
            type: this.#type,
            charge: this.#calculateCharge(),
            reference: this.#reference,
            merchantId: this.#merchantId,
            currency: 'BDT',
            timestamp: new Date(),
        });
    }
}

// ব্যবহার
const txn = BkashTransactionBuilder.create()
    .from('01712345678')
    .to('01898765432')
    .amount(5000)
    .asSendMoney()
    .withReference('বাড়ি ভাড়া - জানুয়ারি')
    .build();

console.log(txn.summary());
```

---

### ৩. Step Builder (Enforced Order)

Step Builder গ্যারান্টি দেয় যে builder-এর মেথডগুলো **নির্দিষ্ট ক্রমে** কল হবে। প্রতিটি ধাপ শুধুমাত্র পরবর্তী ধাপের ইন্টারফেস রিটার্ন করে — ভুল ক্রমে কল করা **compile-time/IDE-level** অসম্ভব।

#### PHP 8.3 — Step Builder

```php
<?php

declare(strict_types=1);

// প্রতিটি step-এর জন্য আলাদা interface
interface OrderStepProduct
{
    public function product(string $name, float $price): OrderStepQuantity;
}

interface OrderStepQuantity
{
    public function quantity(int $qty): OrderStepShipping;
}

interface OrderStepShipping
{
    public function shipping(string $address, string $method): OrderStepPayment;
}

interface OrderStepPayment
{
    public function payWithBkash(string $number): OrderStepFinal;
    public function payWithCOD(): OrderStepFinal;
    public function payWithCard(string $cardToken): OrderStepFinal;
}

interface OrderStepFinal
{
    public function withCoupon(string $code): OrderStepFinal;
    public function withGiftWrap(string $message = ''): OrderStepFinal;
    public function build(): EcommerceOrder;
}

readonly class EcommerceOrder
{
    public function __construct(
        public string $product,
        public float $price,
        public int $quantity,
        public string $shippingAddress,
        public string $shippingMethod,
        public string $paymentMethod,
        public ?string $paymentDetail = null,
        public ?string $coupon = null,
        public bool $giftWrap = false,
        public string $giftMessage = '',
    ) {}

    public function total(): float
    {
        return $this->price * $this->quantity;
    }
}

class EcommerceOrderBuilder implements
    OrderStepProduct,
    OrderStepQuantity,
    OrderStepShipping,
    OrderStepPayment,
    OrderStepFinal
{
    private string $product = '';
    private float $price = 0;
    private int $quantity = 1;
    private string $shippingAddress = '';
    private string $shippingMethod = '';
    private string $paymentMethod = '';
    private ?string $paymentDetail = null;
    private ?string $coupon = null;
    private bool $giftWrap = false;
    private string $giftMessage = '';

    // প্রথম ধাপ হিসেবে শুধু OrderStepProduct রিটার্ন
    public static function create(): OrderStepProduct
    {
        return new self();
    }

    public function product(string $name, float $price): OrderStepQuantity
    {
        $this->product = $name;
        $this->price = $price;
        return $this;
    }

    public function quantity(int $qty): OrderStepShipping
    {
        $this->quantity = $qty;
        return $this;
    }

    public function shipping(string $address, string $method): OrderStepPayment
    {
        $this->shippingAddress = $address;
        $this->shippingMethod = $method;
        return $this;
    }

    public function payWithBkash(string $number): OrderStepFinal
    {
        $this->paymentMethod = 'bKash';
        $this->paymentDetail = $number;
        return $this;
    }

    public function payWithCOD(): OrderStepFinal
    {
        $this->paymentMethod = 'Cash on Delivery';
        return $this;
    }

    public function payWithCard(string $cardToken): OrderStepFinal
    {
        $this->paymentMethod = 'Card';
        $this->paymentDetail = $cardToken;
        return $this;
    }

    public function withCoupon(string $code): OrderStepFinal
    {
        $this->coupon = $code;
        return $this;
    }

    public function withGiftWrap(string $message = ''): OrderStepFinal
    {
        $this->giftWrap = true;
        $this->giftMessage = $message;
        return $this;
    }

    public function build(): EcommerceOrder
    {
        return new EcommerceOrder(
            product: $this->product,
            price: $this->price,
            quantity: $this->quantity,
            shippingAddress: $this->shippingAddress,
            shippingMethod: $this->shippingMethod,
            paymentMethod: $this->paymentMethod,
            paymentDetail: $this->paymentDetail,
            coupon: $this->coupon,
            giftWrap: $this->giftWrap,
            giftMessage: $this->giftMessage,
        );
    }
}

// ব্যবহার — ক্রম বাধ্যতামূলক!
// create() → product() → quantity() → shipping() → pay___() → build()
$order = EcommerceOrderBuilder::create()
    ->product('শাড়ি - জামদানি', 8500.00)    // ধাপ ১
    ->quantity(2)                              // ধাপ ২
    ->shipping('ধানমন্ডি, ঢাকা', 'SA Paribahan')  // ধাপ ৩
    ->payWithBkash('01712345678')              // ধাপ ৪
    ->withCoupon('EID2024')                    // ঐচ্ছিক
    ->withGiftWrap('ঈদ মুবারক! 🎁')           // ঐচ্ছিক
    ->build();                                 // চূড়ান্ত

// ❌ এটা compile-time এ ধরা পড়বে:
// EcommerceOrderBuilder::create()->quantity(2);  // Error! product() আগে কল করতে হবে
```

#### JavaScript ES2022+ — Step Builder (TypeScript-style enforcement)

```javascript
// JS তে Step Builder — ক্লোজার দিয়ে enforce করা
class EcommerceOrderBuilder {
    static create() {
        const state = {};

        // প্রতিটি step শুধু পরবর্তী step-এর মেথড expose করে
        const stepProduct = {
            product(name, price) {
                state.product = name;
                state.price = price;
                return stepQuantity;
            }
        };

        const stepQuantity = {
            quantity(qty) {
                state.quantity = qty;
                return stepShipping;
            }
        };

        const stepShipping = {
            shipping(address, method) {
                state.shippingAddress = address;
                state.shippingMethod = method;
                return stepPayment;
            }
        };

        const stepPayment = {
            payWithBkash(number) {
                state.paymentMethod = 'bKash';
                state.paymentDetail = number;
                return stepFinal;
            },
            payWithCOD() {
                state.paymentMethod = 'Cash on Delivery';
                return stepFinal;
            },
            payWithCard(token) {
                state.paymentMethod = 'Card';
                state.paymentDetail = token;
                return stepFinal;
            }
        };

        const stepFinal = {
            withCoupon(code) {
                state.coupon = code;
                return stepFinal;
            },
            withGiftWrap(message = '') {
                state.giftWrap = true;
                state.giftMessage = message;
                return stepFinal;
            },
            build() {
                return Object.freeze({ ...state, total: state.price * state.quantity });
            }
        };

        return stepProduct;
    }
}

// ব্যবহার — IDE autocomplete শুধু বৈধ মেথড দেখাবে
const order = EcommerceOrderBuilder.create()
    .product('জামদানি শাড়ি', 8500)
    .quantity(2)
    .shipping('ধানমন্ডি, ঢাকা', 'SA Paribahan')
    .payWithBkash('01712345678')
    .withCoupon('EID2024')
    .build();

console.log(order);
// { product: 'জামদানি শাড়ি', price: 8500, quantity: 2, ... total: 17000 }
```

---

### ৪. Telescoping Constructor Problem → Builder Solution

**Telescoping Constructor** হলো এমন অবস্থা যেখানে constructor-এ প্যারামিটার বাড়তে বাড়তে কোড অপঠনযোগ্য হয়ে যায়।

#### সমস্যা (PHP)

```php
// ❌ Telescoping Constructor — দুঃস্বপ্ন!
class Notification {
    public function __construct(
        string $to,
        string $message,
        string $channel = 'sms',
        ?string $subject = null,
        ?string $templateId = null,
        bool $urgent = false,
        ?array $attachments = null,
        ?string $replyTo = null,
        ?string $cc = null,
        ?string $bcc = null,
        string $language = 'bn',
        ?array $metadata = null,
        int $retryCount = 3,
        ?string $scheduledAt = null,
    ) {
        // ... 😱
    }
}

// কোন প্যারামিটার কী? বোঝা অসম্ভব!
$n = new Notification('01712345678', 'হ্যালো', 'email', 'বিষয়', null, true, null, null, 'x@y.com');
```

#### সমাধান (Builder)

```php
// ✅ Builder সমাধান — পরিষ্কার ও পঠনযোগ্য
$notification = NotificationBuilder::create()
    ->to('01712345678')
    ->message('আপনার অর্ডার কনফার্ম হয়েছে!')
    ->viaEmail()
    ->subject('অর্ডার কনফার্মেশন #12345')
    ->urgent()
    ->cc('manager@shop.com.bd')
    ->scheduledAt('2024-12-25 09:00:00')
    ->build();
```

---

## 🌍 Real-World Applicable Areas

### ১. Query Builder (Laravel-স্টাইল)

Laravel-এর `DB::table()->where()->orderBy()->get()` — এটি Builder প্যাটার্নের সবচেয়ে পরিচিত বাস্তব ব্যবহার।

#### PHP 8.3 — Query Builder

```php
<?php

declare(strict_types=1);

class QueryBuilder
{
    private string $table = '';
    private array $columns = ['*'];
    private array $conditions = [];
    private array $bindings = [];
    private array $orderBy = [];
    private ?int $limit = null;
    private ?int $offset = null;
    private array $joins = [];
    private ?string $groupBy = null;
    private array $having = [];

    public static function table(string $table): static
    {
        $builder = new static();
        $builder->table = $table;
        return $builder;
    }

    public function select(string ...$columns): static
    {
        $this->columns = $columns;
        return $this;
    }

    public function where(string $column, string $operator, mixed $value): static
    {
        $this->conditions[] = [
            'type' => 'AND',
            'clause' => "{$column} {$operator} ?",
        ];
        $this->bindings[] = $value;
        return $this;
    }

    public function orWhere(string $column, string $operator, mixed $value): static
    {
        $this->conditions[] = [
            'type' => 'OR',
            'clause' => "{$column} {$operator} ?",
        ];
        $this->bindings[] = $value;
        return $this;
    }

    public function whereIn(string $column, array $values): static
    {
        $placeholders = implode(', ', array_fill(0, count($values), '?'));
        $this->conditions[] = [
            'type' => 'AND',
            'clause' => "{$column} IN ({$placeholders})",
        ];
        $this->bindings = [...$this->bindings, ...$values];
        return $this;
    }

    public function join(string $table, string $first, string $operator, string $second): static
    {
        $this->joins[] = "JOIN {$table} ON {$first} {$operator} {$second}";
        return $this;
    }

    public function leftJoin(string $table, string $first, string $operator, string $second): static
    {
        $this->joins[] = "LEFT JOIN {$table} ON {$first} {$operator} {$second}";
        return $this;
    }

    public function orderBy(string $column, string $direction = 'ASC'): static
    {
        $this->orderBy[] = "{$column} {$direction}";
        return $this;
    }

    public function groupBy(string $column): static
    {
        $this->groupBy = $column;
        return $this;
    }

    public function limit(int $limit): static
    {
        $this->limit = $limit;
        return $this;
    }

    public function offset(int $offset): static
    {
        $this->offset = $offset;
        return $this;
    }

    public function toSQL(): string
    {
        $sql = 'SELECT ' . implode(', ', $this->columns);
        $sql .= " FROM {$this->table}";

        if (!empty($this->joins)) {
            $sql .= ' ' . implode(' ', $this->joins);
        }

        if (!empty($this->conditions)) {
            $sql .= ' WHERE ';
            foreach ($this->conditions as $i => $condition) {
                if ($i > 0) {
                    $sql .= " {$condition['type']} ";
                }
                $sql .= $condition['clause'];
            }
        }

        if ($this->groupBy !== null) {
            $sql .= " GROUP BY {$this->groupBy}";
        }

        if (!empty($this->orderBy)) {
            $sql .= ' ORDER BY ' . implode(', ', $this->orderBy);
        }

        if ($this->limit !== null) {
            $sql .= " LIMIT {$this->limit}";
        }

        if ($this->offset !== null) {
            $sql .= " OFFSET {$this->offset}";
        }

        return $sql;
    }

    public function getBindings(): array
    {
        return $this->bindings;
    }
}

// ব্যবহার — Laravel-এর মতো!
$query = QueryBuilder::table('products')
    ->select('products.name', 'categories.name AS category', 'products.price')
    ->join('categories', 'products.category_id', '=', 'categories.id')
    ->where('products.price', '>=', 500)
    ->where('products.status', '=', 'active')
    ->whereIn('categories.slug', ['electronics', 'clothing'])
    ->orderBy('products.price', 'DESC')
    ->limit(20)
    ->offset(0);

echo $query->toSQL();
// SELECT products.name, categories.name AS category, products.price
// FROM products
// JOIN categories ON products.category_id = categories.id
// WHERE products.price >= ? AND products.status = ? AND categories.slug IN (?, ?)
// ORDER BY products.price DESC
// LIMIT 20 OFFSET 0
```

#### JavaScript ES2022+ — Query Builder

```javascript
class QueryBuilder {
    #table = '';
    #columns = ['*'];
    #conditions = [];
    #bindings = [];
    #orderClauses = [];
    #limitVal = null;
    #offsetVal = null;
    #joins = [];

    static table(name) {
        const qb = new QueryBuilder();
        qb.#table = name;
        return qb;
    }

    select(...columns) {
        this.#columns = columns;
        return this;
    }

    where(column, operator, value) {
        this.#conditions.push({ type: 'AND', clause: `${column} ${operator} ?` });
        this.#bindings.push(value);
        return this;
    }

    orWhere(column, operator, value) {
        this.#conditions.push({ type: 'OR', clause: `${column} ${operator} ?` });
        this.#bindings.push(value);
        return this;
    }

    whereIn(column, values) {
        const placeholders = values.map(() => '?').join(', ');
        this.#conditions.push({ type: 'AND', clause: `${column} IN (${placeholders})` });
        this.#bindings.push(...values);
        return this;
    }

    join(table, first, operator, second) {
        this.#joins.push(`JOIN ${table} ON ${first} ${operator} ${second}`);
        return this;
    }

    orderBy(column, direction = 'ASC') {
        this.#orderClauses.push(`${column} ${direction}`);
        return this;
    }

    limit(n) {
        this.#limitVal = n;
        return this;
    }

    offset(n) {
        this.#offsetVal = n;
        return this;
    }

    toSQL() {
        const parts = [`SELECT ${this.#columns.join(', ')}`, `FROM ${this.#table}`];

        if (this.#joins.length)    parts.push(this.#joins.join(' '));
        if (this.#conditions.length) {
            const where = this.#conditions
                .map((c, i) => (i === 0 ? c.clause : `${c.type} ${c.clause}`))
                .join(' ');
            parts.push(`WHERE ${where}`);
        }
        if (this.#orderClauses.length) parts.push(`ORDER BY ${this.#orderClauses.join(', ')}`);
        if (this.#limitVal !== null)    parts.push(`LIMIT ${this.#limitVal}`);
        if (this.#offsetVal !== null)   parts.push(`OFFSET ${this.#offsetVal}`);

        return parts.join(' ');
    }

    get bindings() { return [...this.#bindings]; }
}

// ব্যবহার
const sql = QueryBuilder.table('products')
    .select('name', 'price', 'stock')
    .where('price', '>=', 500)
    .where('status', '=', 'active')
    .orderBy('price', 'DESC')
    .limit(20)
    .toSQL();

console.log(sql);
```

---

### ২. Email Builder

#### PHP 8.3 — Email Builder

```php
<?php

declare(strict_types=1);

readonly class Email
{
    public function __construct(
        public string $from,
        public array $to,
        public string $subject,
        public string $body,
        public string $contentType = 'text/html',
        public array $cc = [],
        public array $bcc = [],
        public ?string $replyTo = null,
        public array $attachments = [],
        public array $headers = [],
        public int $priority = 3,
    ) {}
}

class EmailBuilder
{
    private string $from = '';
    private array $to = [];
    private string $subject = '';
    private string $body = '';
    private string $contentType = 'text/html';
    private array $cc = [];
    private array $bcc = [];
    private ?string $replyTo = null;
    private array $attachments = [];
    private array $headers = [];
    private int $priority = 3; // 1=সর্বোচ্চ, 5=সর্বনিম্ন

    public static function create(): static
    {
        return new static();
    }

    public function from(string $email, ?string $name = null): static
    {
        $this->from = $name ? "{$name} <{$email}>" : $email;
        return $this;
    }

    public function to(string ...$emails): static
    {
        $this->to = [...$this->to, ...$emails];
        return $this;
    }

    public function cc(string ...$emails): static
    {
        $this->cc = [...$this->cc, ...$emails];
        return $this;
    }

    public function bcc(string ...$emails): static
    {
        $this->bcc = [...$this->bcc, ...$emails];
        return $this;
    }

    public function replyTo(string $email): static
    {
        $this->replyTo = $email;
        return $this;
    }

    public function subject(string $subject): static
    {
        $this->subject = $subject;
        return $this;
    }

    public function body(string $body): static
    {
        $this->body = $body;
        return $this;
    }

    public function html(string $html): static
    {
        $this->body = $html;
        $this->contentType = 'text/html';
        return $this;
    }

    public function text(string $text): static
    {
        $this->body = $text;
        $this->contentType = 'text/plain';
        return $this;
    }

    public function attach(string $path, ?string $name = null): static
    {
        $this->attachments[] = [
            'path' => $path,
            'name' => $name ?? basename($path),
        ];
        return $this;
    }

    public function header(string $key, string $value): static
    {
        $this->headers[$key] = $value;
        return $this;
    }

    public function highPriority(): static
    {
        $this->priority = 1;
        return $this;
    }

    public function lowPriority(): static
    {
        $this->priority = 5;
        return $this;
    }

    public function build(): Email
    {
        if (empty($this->to)) {
            throw new \LogicException('প্রাপক (to) অবশ্যই দিতে হবে');
        }
        if (empty($this->subject)) {
            throw new \LogicException('বিষয় (subject) অবশ্যই দিতে হবে');
        }

        return new Email(
            from: $this->from,
            to: $this->to,
            subject: $this->subject,
            body: $this->body,
            contentType: $this->contentType,
            cc: $this->cc,
            bcc: $this->bcc,
            replyTo: $this->replyTo,
            attachments: $this->attachments,
            headers: $this->headers,
            priority: $this->priority,
        );
    }
}

// ব্যবহার
$email = EmailBuilder::create()
    ->from('noreply@daraz.com.bd', 'Daraz Bangladesh')
    ->to('customer@example.com')
    ->cc('support@daraz.com.bd')
    ->subject('আপনার অর্ডার #BD12345 কনফার্ম হয়েছে! ✅')
    ->html('<h1>ধন্যবাদ!</h1><p>আপনার অর্ডার প্রসেস হচ্ছে...</p>')
    ->attach('/invoices/BD12345.pdf', 'ইনভয়েস.pdf')
    ->highPriority()
    ->build();
```

#### JavaScript ES2022+ — Email Builder

```javascript
class Email {
    #data;
    constructor(data) {
        this.#data = Object.freeze(structuredClone(data));
    }
    get from()        { return this.#data.from; }
    get to()          { return this.#data.to; }
    get subject()     { return this.#data.subject; }
    get body()        { return this.#data.body; }
    get cc()          { return this.#data.cc; }
    get bcc()         { return this.#data.bcc; }
    get attachments() { return this.#data.attachments; }
    get priority()    { return this.#data.priority; }

    toJSON() { return { ...this.#data }; }
}

class EmailBuilder {
    #from = '';
    #to = [];
    #subject = '';
    #body = '';
    #contentType = 'text/html';
    #cc = [];
    #bcc = [];
    #replyTo = null;
    #attachments = [];
    #headers = {};
    #priority = 3;

    static create() { return new EmailBuilder(); }

    from(email, name = null) {
        this.#from = name ? `${name} <${email}>` : email;
        return this;
    }

    to(...emails) {
        this.#to.push(...emails);
        return this;
    }

    cc(...emails) { this.#cc.push(...emails); return this; }
    bcc(...emails) { this.#bcc.push(...emails); return this; }
    replyTo(email) { this.#replyTo = email; return this; }
    subject(s) { this.#subject = s; return this; }

    html(content) {
        this.#body = content;
        this.#contentType = 'text/html';
        return this;
    }

    text(content) {
        this.#body = content;
        this.#contentType = 'text/plain';
        return this;
    }

    attach(path, name = null) {
        this.#attachments.push({ path, name: name ?? path.split('/').pop() });
        return this;
    }

    highPriority() { this.#priority = 1; return this; }
    lowPriority()  { this.#priority = 5; return this; }

    build() {
        if (!this.#to.length) throw new Error('প্রাপক (to) অবশ্যই দিতে হবে');
        if (!this.#subject)   throw new Error('বিষয় (subject) অবশ্যই দিতে হবে');

        return new Email({
            from: this.#from, to: [...this.#to], subject: this.#subject,
            body: this.#body, contentType: this.#contentType,
            cc: [...this.#cc], bcc: [...this.#bcc], replyTo: this.#replyTo,
            attachments: [...this.#attachments], headers: { ...this.#headers },
            priority: this.#priority,
        });
    }
}

// ব্যবহার
const email = EmailBuilder.create()
    .from('noreply@daraz.com.bd', 'Daraz Bangladesh')
    .to('customer@example.com')
    .subject('অর্ডার কনফার্মেশন #BD12345 ✅')
    .html('<h1>ধন্যবাদ!</h1>')
    .attach('/invoices/BD12345.pdf')
    .highPriority()
    .build();
```

### ৩. অন্যান্য প্রযোজ্য ক্ষেত্র (সংক্ষিপ্ত)

| ক্ষেত্র | Builder কেন? | উদাহরণ |
|---------|-------------|--------|
| **HTTP Request** | Headers, body, auth, timeout — অনেক কনফিগ | `Guzzle::request()->withHeader()->withBody()->send()` |
| **Report Builder** | Header, sections, charts, footer — ক্রমিক | `Report::create()->header()->section()->chart()->pdf()` |
| **Form Builder** | Fields, validation, layout — dynamic | `Form::create()->text('name')->email('email')->rules(...)` |
| **API Response** | Status, data, meta, pagination, links | `ApiResponse::success()->data($d)->paginate($p)->build()` |
| **Config Builder** | DB, cache, queue — nested complex config | `Config::database()->host('localhost')->port(3306)` |

---

## 🔥 Advanced Deep Dive

### ১. Builder vs Factory — বিস্তারিত তুলনা

```
┌─────────────────────┬────────────────────────────────────────┐
│       দিক           │    Factory              Builder        │
├─────────────────────┼────────────────────────────────────────┤
│ উদ্দেশ্য            │ কোন অবজেক্ট তৈরি     কীভাবে তৈরি     │
│                     │ হবে তা নির্ধারণ       হবে তা নিয়ন্ত্রণ│
├─────────────────────┼────────────────────────────────────────┤
│ জটিলতা              │ সাধারণ অবজেক্ট        জটিল অবজেক্ট    │
│                     │ (কম প্যারামিটার)     (অনেক প্যারামিটার)│
├─────────────────────┼────────────────────────────────────────┤
│ ধাপ                 │ একটি ধাপে তৈরি       ধাপে ধাপে তৈরি   │
├─────────────────────┼────────────────────────────────────────┤
│ ফোকাস               │ Product family        Construction     │
│                     │ (গরু/ছাগল/মুরগি)     process           │
├─────────────────────┼────────────────────────────────────────┤
│ Return              │ সরাসরি Product        Builder (chain),  │
│                     │                       শেষে build()     │
├─────────────────────┼────────────────────────────────────────┤
│ উদাহরণ              │ PaymentGateway        QueryBuilder     │
│                     │ Factory(bKash/Nagad)  EmailBuilder     │
└─────────────────────┴────────────────────────────────────────┘
```

**সহজ সূত্র:**
- **Factory** → "আমাকে **একটা** গাড়ি দাও" (কোন ধরনের সেটা Factory ঠিক করবে)
- **Builder** → "আমি **নিজে** গাড়ি কনফিগার করব — রঙ, ইঞ্জিন, সিট, AC..."

### ২. Immutable Builder

Immutable Builder প্রতিটি মেথড কলে **নতুন Builder instance** রিটার্ন করে। এতে thread-safety ও predictability বাড়ে।

```php
<?php

declare(strict_types=1);

// Immutable Builder — প্রতিটি ধাপে নতুন instance
final class ImmutableRequestBuilder
{
    private function __construct(
        private readonly string $method = 'GET',
        private readonly string $url = '',
        private readonly array $headers = [],
        private readonly ?string $body = null,
        private readonly int $timeout = 30,
    ) {}

    public static function create(): self
    {
        return new self();
    }

    public function withMethod(string $method): self
    {
        return new self($method, $this->url, $this->headers, $this->body, $this->timeout);
    }

    public function withUrl(string $url): self
    {
        return new self($this->method, $url, $this->headers, $this->body, $this->timeout);
    }

    public function withHeader(string $key, string $value): self
    {
        $headers = [...$this->headers, $key => $value];
        return new self($this->method, $this->url, $headers, $this->body, $this->timeout);
    }

    public function withBody(string $body): self
    {
        return new self($this->method, $this->url, $this->headers, $body, $this->timeout);
    }

    public function withTimeout(int $seconds): self
    {
        return new self($this->method, $this->url, $this->headers, $this->body, $seconds);
    }

    public function build(): array
    {
        if (empty($this->url)) {
            throw new \LogicException('URL অবশ্যই দিতে হবে');
        }

        return [
            'method'  => $this->method,
            'url'     => $this->url,
            'headers' => $this->headers,
            'body'    => $this->body,
            'timeout' => $this->timeout,
        ];
    }
}

// সুবিধা: base builder থেকে ভিন্ন ভিন্ন variation তৈরি
$base = ImmutableRequestBuilder::create()
    ->withUrl('https://api.example.com/v1')
    ->withHeader('Authorization', 'Bearer token123')
    ->withHeader('Accept', 'application/json');

// $base অপরিবর্তিত থাকে!
$getRequest  = $base->withMethod('GET')->build();
$postRequest = $base->withMethod('POST')->withBody('{"key":"value"}')->build();
```

```javascript
// JavaScript — Immutable Builder
class ImmutableRequestBuilder {
    #state;

    constructor(state = {}) {
        this.#state = Object.freeze({
            method: 'GET', url: '', headers: {}, body: null, timeout: 30,
            ...state,
        });
    }

    static create() { return new ImmutableRequestBuilder(); }

    withMethod(m)    { return new ImmutableRequestBuilder({ ...this.#state, method: m }); }
    withUrl(u)       { return new ImmutableRequestBuilder({ ...this.#state, url: u }); }
    withBody(b)      { return new ImmutableRequestBuilder({ ...this.#state, body: b }); }
    withTimeout(t)   { return new ImmutableRequestBuilder({ ...this.#state, timeout: t }); }

    withHeader(key, value) {
        return new ImmutableRequestBuilder({
            ...this.#state,
            headers: { ...this.#state.headers, [key]: value },
        });
    }

    build() {
        if (!this.#state.url) throw new Error('URL অবশ্যই দিতে হবে');
        return structuredClone(this.#state);
    }
}

// base থেকে branching
const base = ImmutableRequestBuilder.create()
    .withUrl('https://api.example.com')
    .withHeader('Authorization', 'Bearer xyz');

const get  = base.withMethod('GET').build();
const post = base.withMethod('POST').withBody('{}').build();
// base অপরিবর্তিত ✅
```

### ৩. Builder with Validation

```php
<?php

declare(strict_types=1);

// Validation স্ট্র্যাটেজি: build() সময়ে সব একসাথে validate
class ValidatedOrderBuilder
{
    private array $errors = [];
    private array $data = [];

    public function product(string $name): static
    {
        if (mb_strlen($name) < 2) {
            $this->errors[] = 'প্রোডাক্ট নাম কমপক্ষে ২ অক্ষরের হতে হবে';
        }
        $this->data['product'] = $name;
        return $this;
    }

    public function price(float $price): static
    {
        if ($price <= 0) {
            $this->errors[] = 'মূল্য অবশ্যই ধনাত্মক হতে হবে';
        }
        if ($price > 1_000_000) {
            $this->errors[] = 'মূল্য ১০ লক্ষ টাকার বেশি হতে পারবে না';
        }
        $this->data['price'] = $price;
        return $this;
    }

    public function quantity(int $qty): static
    {
        if ($qty < 1 || $qty > 100) {
            $this->errors[] = 'পরিমাণ ১-১০০ এর মধ্যে হতে হবে';
        }
        $this->data['quantity'] = $qty;
        return $this;
    }

    public function build(): array
    {
        // বাধ্যতামূলক ফিল্ড চেক
        $required = ['product', 'price', 'quantity'];
        foreach ($required as $field) {
            if (!isset($this->data[$field])) {
                $this->errors[] = "{$field} বাধ্যতামূলক";
            }
        }

        if (!empty($this->errors)) {
            throw new \DomainException(
                "অবৈধ অর্ডার:\n• " . implode("\n• ", $this->errors)
            );
        }

        return $this->data;
    }

    // ভ্যালিডেশন চেক (build() এর আগে)
    public function isValid(): bool
    {
        return empty($this->errors);
    }

    public function getErrors(): array
    {
        return $this->errors;
    }
}
```

### ৪. Builder with Default Values

```php
<?php

// ডিফল্ট ভ্যালু সহ Builder — সাধারণ কেসে কম কনফিগারেশন লাগে
class ApiResponseBuilder
{
    private int $status = 200;
    private bool $success = true;
    private mixed $data = null;
    private ?string $message = null;
    private array $meta = [];
    private array $errors = [];
    private ?array $pagination = null;

    public static function success(mixed $data = null): static
    {
        $builder = new static();
        $builder->data = $data;
        $builder->success = true;
        $builder->status = 200;
        return $builder;
    }

    public static function error(string $message, int $status = 400): static
    {
        $builder = new static();
        $builder->message = $message;
        $builder->success = false;
        $builder->status = $status;
        return $builder;
    }

    public function message(string $msg): static { $this->message = $msg; return $this; }
    public function status(int $code): static    { $this->status = $code; return $this; }
    public function meta(array $meta): static    { $this->meta = $meta; return $this; }

    public function paginate(int $total, int $page, int $perPage): static
    {
        $this->pagination = [
            'total'    => $total,
            'page'     => $page,
            'per_page' => $perPage,
            'pages'    => (int)ceil($total / $perPage),
        ];
        return $this;
    }

    public function build(): array
    {
        $response = [
            'success' => $this->success,
            'status'  => $this->status,
        ];

        if ($this->data !== null)       $response['data'] = $this->data;
        if ($this->message !== null)    $response['message'] = $this->message;
        if (!empty($this->meta))        $response['meta'] = $this->meta;
        if ($this->pagination !== null) $response['pagination'] = $this->pagination;
        if (!empty($this->errors))      $response['errors'] = $this->errors;

        return $response;
    }
}

// সহজ ব্যবহার — ডিফল্ট দিয়ে অধিকাংশ কাজ হয়ে যায়
$simple = ApiResponseBuilder::success(['id' => 1, 'name' => 'পণ্য'])->build();

$paginated = ApiResponseBuilder::success($products)
    ->message('পণ্য তালিকা সফল')
    ->paginate(total: 150, page: 3, perPage: 20)
    ->build();

$error = ApiResponseBuilder::error('পণ্য পাওয়া যায়নি', 404)->build();
```

### ৫. Recursive/Nested Builders

জটিল অবজেক্টে অবজেক্টের ভেতর অবজেক্ট থাকে। Nested Builder এই সমস্যা সমাধান করে।

```php
<?php

declare(strict_types=1);

class ReportBuilder
{
    private string $title = '';
    private array $sections = [];
    private ?array $header = null;
    private ?array $footer = null;

    public static function create(string $title): static
    {
        $builder = new static();
        $builder->title = $title;
        return $builder;
    }

    public function header(callable $configure): static
    {
        $headerBuilder = new ReportHeaderBuilder();
        $configure($headerBuilder);
        $this->header = $headerBuilder->build();
        return $this;
    }

    public function section(callable $configure): static
    {
        $sectionBuilder = new ReportSectionBuilder();
        $configure($sectionBuilder);
        $this->sections[] = $sectionBuilder->build();
        return $this;
    }

    public function footer(callable $configure): static
    {
        $footerBuilder = new ReportFooterBuilder();
        $configure($footerBuilder);
        $this->footer = $footerBuilder->build();
        return $this;
    }

    public function build(): array
    {
        return [
            'title'    => $this->title,
            'header'   => $this->header,
            'sections' => $this->sections,
            'footer'   => $this->footer,
        ];
    }
}

class ReportSectionBuilder
{
    private string $title = '';
    private string $content = '';
    private array $charts = [];

    public function title(string $t): static   { $this->title = $t; return $this; }
    public function content(string $c): static { $this->content = $c; return $this; }
    public function chart(string $type, array $data): static
    {
        $this->charts[] = ['type' => $type, 'data' => $data];
        return $this;
    }
    public function build(): array
    {
        return ['title' => $this->title, 'content' => $this->content, 'charts' => $this->charts];
    }
}

class ReportHeaderBuilder
{
    private string $logo = '';
    private string $company = '';
    public function logo(string $l): static    { $this->logo = $l; return $this; }
    public function company(string $c): static { $this->company = $c; return $this; }
    public function build(): array { return ['logo' => $this->logo, 'company' => $this->company]; }
}

class ReportFooterBuilder
{
    private string $text = '';
    private bool $pageNumbers = true;
    public function text(string $t): static         { $this->text = $t; return $this; }
    public function pageNumbers(bool $p): static    { $this->pageNumbers = $p; return $this; }
    public function build(): array { return ['text' => $this->text, 'pageNumbers' => $this->pageNumbers]; }
}

// ব্যবহার — nested builders callback দিয়ে
$report = ReportBuilder::create('মাসিক বিক্রয় রিপোর্ট - ডিসেম্বর ২০২৪')
    ->header(fn(ReportHeaderBuilder $h) => $h
        ->logo('/images/logo.png')
        ->company('বাংলা কমার্স লিমিটেড')
    )
    ->section(fn(ReportSectionBuilder $s) => $s
        ->title('বিক্রয় সারাংশ')
        ->content('এই মাসে মোট বিক্রয় ১৫ লক্ষ টাকা...')
        ->chart('bar', ['জানু' => 500, 'ফেব্রু' => 700])
    )
    ->section(fn(ReportSectionBuilder $s) => $s
        ->title('শীর্ষ পণ্য')
        ->content('সবচেয়ে বেশি বিক্রি হওয়া পণ্যের তালিকা')
    )
    ->footer(fn(ReportFooterBuilder $f) => $f
        ->text('© ২০২৪ বাংলা কমার্স')
        ->pageNumbers(true)
    )
    ->build();
```

### ৬. Laravel Query Builder অভ্যন্তরীণ কাঠামো (সংক্ষেপে)

```
Laravel Query Builder অভ্যন্তরীণ আর্কিটেকচার:

Illuminate\Database\Query\Builder
├── $connection    → Database Connection (PDO wrapper)
├── $grammar       → Grammar (MySQL/PostgreSQL/SQLite → SQL string)
├── $processor     → Processor (result processing)
│
├── State Properties (Builder Pattern):
│   ├── $columns, $distinct
│   ├── $from (table)
│   ├── $joins, $wheres, $groups
│   ├── $havings, $orders
│   ├── $limit, $offset
│   ├── $unions, $bindings
│   └── $aggregate
│
├── Fluent Methods (return $this):
│   ├── select(), where(), orWhere()
│   ├── join(), leftJoin()
│   ├── orderBy(), groupBy()
│   ├── limit(), offset()
│   └── having(), union()
│
└── Terminal Methods (execute + return result):
    ├── get() → Collection
    ├── first() → Model|null
    ├── count() → int
    ├── paginate() → Paginator
    └── toSql() → string

মূল পর্যবেক্ষণ:
১. Grammar pattern: Builder state → SQL string (Strategy pattern সহ)
২. প্রতিটি where() কল $wheres array-তে push করে
৩. get()/first() = terminal operation = build() এর সমতুল্য
৪. clone() সাপোর্ট আছে — sub-queries-এর জন্য
```

### ৭. Director Pattern: কখন ব্যবহার করবেন / স্কিপ করবেন

```
✅ Director ব্যবহার করুন যখন:
──────────────────────────────
• একই construction sequence বারবার লাগে
• বিভিন্ন "preset" configuration দরকার
  (যেমন: buildLuxuryHouse(), buildBudgetHouse())
• Client কোডকে construction details থেকে সম্পূর্ণ আড়াল করতে চান
• টেস্টিং সহজ করতে চান — Director mock করা যায়

❌ Director স্কিপ করুন যখন:
──────────────────────────────
• Builder-ই যথেষ্ট সহজ (Fluent Builder)
• প্রতিটি build unique — কোনো পুনরাবৃত্তি নেই
• Client-কে পূর্ণ নিয়ন্ত্রণ দিতে চান
• YAGNI — অতিরিক্ত abstraction দরকার নেই
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | বিবরণ |
|---|--------|-------|
| 1 | **Step-by-step নির্মাণ** | জটিল অবজেক্ট ধাপে ধাপে তৈরি করা যায় |
| 2 | **Readable Code** | Method chaining কোড স্ব-ব্যাখ্যামূলক করে |
| 3 | **Telescoping সমাধান** | ১৫+ প্যারামিটারের constructor থেকে মুক্তি |
| 4 | **Immutable Product** | `readonly` product তৈরি করা সহজ |
| 5 | **Single Responsibility** | Construction logic আলাদা class-এ |
| 6 | **Reusability** | Director দিয়ে preset configuration |
| 7 | **Validation** | `build()` এ সমস্ত validation একসাথে |
| 8 | **IDE Support** | Autocomplete ও type checking ভালো কাজ করে |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | বিবরণ |
|---|---------|-------|
| 1 | **বাড়তি Class** | প্রতিটি Product-এর জন্য Builder class লাগে |
| 2 | **Over-engineering** | সাধারণ অবজেক্টে Builder অতিরিক্ত জটিলতা |
| 3 | **Mutable State** | Builder-এর internal state ভুলে দূষিত হতে পারে |
| 4 | **Code Duplication** | Product ও Builder-এ একই field দুবার define |
| 5 | **Learning Curve** | নতুন ডেভেলপারদের জন্য Step Builder জটিল |

---

## ⚠️ Common Mistakes ও সমাধান

### ভুল ১: Mutable State — Builder পুনঃব্যবহারে সমস্যা

```php
// ❌ বিপদ! Builder reset না করলে আগের state থেকে যায়
$builder = new OrderBuilder();

$order1 = $builder->product('A')->price(100)->build();
$order2 = $builder->product('B')->build();
// $order2 তে price 100 থেকে যাবে! 😱

// ✅ সমাধান: build() এ auto-reset অথবা Immutable Builder ব্যবহার
public function build(): Order
{
    $order = new Order(...$this->data);
    $this->reset(); // auto-reset
    return $order;
}
```

### ভুল ২: Required Fields ভুলে যাওয়া

```php
// ❌ বিপদ! সব field optional হলে incomplete object তৈরি হয়
$email = EmailBuilder::create()->subject('Test')->build();
// to নেই! কাকে পাঠাবে? 💥

// ✅ সমাধান: build() এ validation
public function build(): Email
{
    $this->validate(); // required fields চেক
    return new Email(...);
}

// ✅ আরো ভালো: Step Builder দিয়ে compile-time enforcement
```

### ভুল ৩: Builder-কে God Class বানানো

```php
// ❌ খারাপ — Builder নিজেই সব করছে
class OrderBuilder {
    public function build(): Order { /* ... */ }
    public function save(): void { /* DB save — Builder-এর কাজ না! */ }
    public function sendEmail(): void { /* Email — Builder-এর কাজ না! */ }
    public function generatePDF(): void { /* PDF — Builder-এর কাজ না! */ }
}

// ✅ Builder শুধু অবজেক্ট তৈরি করবে, বাকি কাজ আলাদা class-এ
$order = $builder->build();
$repository->save($order);
$notifier->send($order);
```

### ভুল ৪: অপ্রয়োজনীয় Builder

```php
// ❌ অতিরিক্ত — এত সহজ অবজেক্টে Builder লাগে না
class PointBuilder {
    public function x(int $x): static { /* ... */ }
    public function y(int $y): static { /* ... */ }
    public function build(): Point { /* ... */ }
}

// ✅ সরাসরি constructor-ই যথেষ্ট
$point = new Point(x: 10, y: 20); // PHP named arguments
```

---

## 🧪 টেস্টিং

### PHPUnit — Builder টেস্ট

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class BkashTransactionBuilderTest extends TestCase
{
    #[Test]
    public function সফল_লেনদেন_তৈরি_করে(): void
    {
        $txn = BkashTransactionBuilder::create()
            ->from('01712345678')
            ->to('01898765432')
            ->amount(1000)
            ->asSendMoney()
            ->build();

        $this->assertSame('01712345678', $txn->sender);
        $this->assertSame('01898765432', $txn->receiver);
        $this->assertSame(1000.0, $txn->amount);
        $this->assertSame('SendMoney', $txn->type);
    }

    #[Test]
    public function চার্জ_সঠিকভাবে_গণনা_করে(): void
    {
        $txn = BkashTransactionBuilder::create()
            ->from('01712345678')
            ->to('01898765432')
            ->amount(5000)
            ->asSendMoney()
            ->build();

        // ১% = ৫০ টাকা, কিন্তু সর্বোচ্চ ২৫
        $this->assertSame(25.0, $txn->charge);
    }

    #[Test]
    public function অবৈধ_নম্বরে_এক্সেপশন_দেয়(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('অবৈধ বাংলাদেশি মোবাইল নম্বর');

        BkashTransactionBuilder::create()
            ->from('12345') // অবৈধ!
            ->to('01898765432')
            ->amount(100)
            ->build();
    }

    #[Test]
    public function বাধ্যতামূলক_ফিল্ড_ছাড়া_ব্যর্থ_হয়(): void
    {
        $this->expectException(\LogicException::class);
        $this->expectExceptionMessage('বাধ্যতামূলক ফিল্ড অনুপস্থিত');

        BkashTransactionBuilder::create()
            ->from('01712345678')
            ->build(); // receiver ও amount নেই!
    }

    #[Test]
    public function প্রেরক_প্রাপক_একই_হলে_ব্যর্থ_হয়(): void
    {
        $this->expectException(\LogicException::class);

        BkashTransactionBuilder::create()
            ->from('01712345678')
            ->to('01712345678') // একই!
            ->amount(100)
            ->build();
    }

    #[Test]
    public function সীমার_বাইরে_পরিমাণে_ব্যর্থ_হয়(): void
    {
        $this->expectException(\InvalidArgumentException::class);

        BkashTransactionBuilder::create()
            ->from('01712345678')
            ->to('01898765432')
            ->amount(30000) // সর্বোচ্চ ২৫,০০০
            ->build();
    }

    #[Test]
    public function ক্যাশআউটে_চার্জ_সঠিক(): void
    {
        $txn = BkashTransactionBuilder::create()
            ->from('01712345678')
            ->to('01898765432')
            ->amount(10000)
            ->asCashOut()
            ->build();

        // ১.৮৫% of 10000 = ১৮৫
        $this->assertEqualsWithDelta(185.0, $txn->charge, 0.01);
    }

    #[DataProvider('validTransactionProvider')]
    #[Test]
    public function বিভিন্ন_বৈধ_ট্রানজ্যাকশন_তৈরি_করে(
        string $from, string $to, float $amount, string $type
    ): void {
        $builder = BkashTransactionBuilder::create()
            ->from($from)
            ->to($to)
            ->amount($amount);

        $builder = match($type) {
            'SendMoney' => $builder->asSendMoney(),
            'CashOut'   => $builder->asCashOut(),
            'Payment'   => $builder->asPayment('MERCHANT001'),
        };

        $txn = $builder->build();
        $this->assertSame($type, $txn->type);
    }

    public static function validTransactionProvider(): array
    {
        return [
            'send_money'   => ['01712345678', '01898765432', 500, 'SendMoney'],
            'cash_out'     => ['01712345678', '01311111111', 10000, 'CashOut'],
            'payment'      => ['01712345678', '01511111111', 2500, 'Payment'],
        ];
    }
}
```

### Jest — Builder টেস্ট

```javascript
import { describe, test, expect } from '@jest/globals';

describe('BkashTransactionBuilder', () => {

    test('সফল লেনদেন তৈরি করে', () => {
        const txn = BkashTransactionBuilder.create()
            .from('01712345678')
            .to('01898765432')
            .amount(1000)
            .asSendMoney()
            .build();

        expect(txn.sender).toBe('01712345678');
        expect(txn.receiver).toBe('01898765432');
        expect(txn.amount).toBe(1000);
        expect(txn.type).toBe('SendMoney');
    });

    test('SendMoney চার্জ সর্বোচ্চ ২৫ টাকা', () => {
        const txn = BkashTransactionBuilder.create()
            .from('01712345678')
            .to('01898765432')
            .amount(5000)
            .asSendMoney()
            .build();

        expect(txn.charge).toBe(25.0);
    });

    test('CashOut চার্জ ১.৮৫%', () => {
        const txn = BkashTransactionBuilder.create()
            .from('01712345678')
            .to('01898765432')
            .amount(10000)
            .asCashOut()
            .build();

        expect(txn.charge).toBeCloseTo(185.0, 2);
    });

    test('অবৈধ মোবাইল নম্বরে error throw করে', () => {
        expect(() => {
            BkashTransactionBuilder.create().from('12345');
        }).toThrow('অবৈধ মোবাইল নম্বর');
    });

    test('বাধ্যতামূলক ফিল্ড ছাড়া build ব্যর্থ হয়', () => {
        expect(() => {
            BkashTransactionBuilder.create()
                .from('01712345678')
                .build();
        }).toThrow('বাধ্যতামূলক ফিল্ড অনুপস্থিত');
    });

    test('প্রেরক ও প্রাপক একই হলে error', () => {
        expect(() => {
            BkashTransactionBuilder.create()
                .from('01712345678')
                .to('01712345678')
                .amount(100)
                .build();
        }).toThrow('প্রেরক ও প্রাপক একই হতে পারবে না');
    });

    test('সীমার বাইরে পরিমাণে error', () => {
        expect(() => {
            BkashTransactionBuilder.create()
                .from('01712345678')
                .to('01898765432')
                .amount(30000);
        }).toThrow();
    });

    test('reference সঠিকভাবে সেট হয়', () => {
        const txn = BkashTransactionBuilder.create()
            .from('01712345678')
            .to('01898765432')
            .amount(500)
            .withReference('বাড়ি ভাড়া')
            .build();

        expect(txn.reference).toBe('বাড়ি ভাড়া');
    });

    test('Payment-এ চার্জ শূন্য', () => {
        const txn = BkashTransactionBuilder.create()
            .from('01712345678')
            .to('01898765432')
            .amount(2000)
            .asPayment('MERCHANT001')
            .build();

        expect(txn.charge).toBe(0);
    });
});

describe('QueryBuilder', () => {
    test('সাধারণ SELECT query তৈরি করে', () => {
        const sql = QueryBuilder.table('users')
            .select('name', 'email')
            .where('status', '=', 'active')
            .toSQL();

        expect(sql).toContain('SELECT name, email');
        expect(sql).toContain('FROM users');
        expect(sql).toContain('WHERE status = ?');
    });

    test('JOIN সহ query তৈরি করে', () => {
        const sql = QueryBuilder.table('orders')
            .join('users', 'orders.user_id', '=', 'users.id')
            .where('orders.total', '>', 1000)
            .orderBy('orders.created_at', 'DESC')
            .toSQL();

        expect(sql).toContain('JOIN users');
        expect(sql).toContain('ORDER BY');
    });

    test('LIMIT ও OFFSET সেট করে', () => {
        const sql = QueryBuilder.table('products')
            .limit(10)
            .offset(20)
            .toSQL();

        expect(sql).toContain('LIMIT 10');
        expect(sql).toContain('OFFSET 20');
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Builder ↔ অন্যান্য প্যাটার্নের সম্পর্ক

```
┌─────────────────────────────────────────────────────────────┐
│                    সম্পর্কিত প্যাটার্ন                       │
├──────────────────┬──────────────────────────────────────────┤
│ Abstract Factory │ Builder জটিল অবজেক্ট তৈরি করে,          │
│                  │ Factory একটি family-র অবজেক্ট তৈরি করে।│
│                  │ দুটো একসাথে ব্যবহার করা যায়:            │
│                  │ Factory → কোন Builder ব্যবহার হবে সেটা    │
│                  │ নির্ধারণ করে।                            │
├──────────────────┼──────────────────────────────────────────┤
│ Composite        │ Builder দিয়ে complex Composite tree      │
│                  │ ধাপে ধাপে তৈরি করা যায়। যেমন: nested    │
│                  │ menu, org chart, DOM tree।                │
├──────────────────┼──────────────────────────────────────────┤
│ Prototype        │ Builder product তৈরির পর Prototype       │
│                  │ (clone) দিয়ে variation তৈরি করতে পারেন।  │
│                  │ Immutable builder + clone = শক্তিশালী।    │
├──────────────────┼──────────────────────────────────────────┤
│ Fluent Interface │ Builder প্যাটার্নের সবচেয়ে জনপ্রিয়     │
│                  │ বাস্তবায়ন হলো Fluent Interface দিয়ে।     │
│                  │ Fluent সব কিছু Builder না, কিন্তু        │
│                  │ অধিকাংশ Builder-ই Fluent।                │
├──────────────────┼──────────────────────────────────────────┤
│ Strategy         │ Laravel Query Builder Grammar-কে         │
│                  │ Strategy হিসেবে ব্যবহার করে              │
│                  │ (MySQL Grammar, PostgreSQL Grammar)।      │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন

```
১. Constructor-এ ৪+ optional প্যারামিটার থাকলে
২. অবজেক্ট তৈরিতে একাধিক ধাপ থাকলে
৩. একই process এ ভিন্ন representation দরকার হলে
৪. Immutable object তৈরি করতে চাইলে (readonly class)
৫. Complex configuration object তৈরি করতে
   (Database config, HTTP request, email)
৬. Test fixture / test data তৈরিতে
   (UserBuilder::create()->withName('test')->build())
৭. DSL (Domain Specific Language) তৈরি করতে
   (Query builder, form builder)
```

### ❌ ব্যবহার করবেন না

```
১. সাধারণ অবজেক্ট (২-৩ প্যারামিটার) → সরাসরি constructor
২. PHP named arguments যথেষ্ট হলে
   new Point(x: 10, y: 20) → Builder দরকার নেই
৩. Configuration array যথেষ্ট হলে
   ['host' => 'localhost', 'port' => 3306]
৪. Object creation one-time হলে এবং variation না থাকলে
৫. Performance-critical hot path-এ
   (প্রতি request-এ লক্ষ লক্ষ বার call হলে overhead বাড়বে)
```

### 🇧🇩 বাংলাদেশ-নির্দিষ্ট ব্যবহার ক্ষেত্র

```
✅ চমৎকার ব্যবহার:
  • E-commerce Order Builder (Daraz/Chaldal-এর মতো)
  • bKash/Nagad Transaction Builder
  • NID Verification Request Builder
  • SMS Gateway Builder (BulkSMS BD)
  • Shipping Label Builder (Pathao/RedX)
  • Invoice Builder (VAT/AIT সহ)
  • Government Form Builder (e-Service)
```

---

## 📋 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────────┐
│                  Builder প্যাটার্ন সারসংক্ষেপ                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  🎯 উদ্দেশ্য: জটিল অবজেক্ট step-by-step তৈরি                │
│                                                                │
│  🧩 মূল উপাদান:                                              │
│     • Builder (interface)  — নির্মাণ ধাপ সংজ্ঞায়িত           │
│     • ConcreteBuilder      — নির্দিষ্ট বাস্তবায়ন             │
│     • Director (optional)  — নির্মাণ ক্রম নিয়ন্ত্রণ           │
│     • Product              — চূড়ান্ত অবজেক্ট                  │
│                                                                │
│  🔑 মনে রাখুন:                                                │
│     ① Telescoping Constructor → Builder দিয়ে সমাধান           │
│     ② Fluent API → method chaining → পঠনযোগ্য কোড            │
│     ③ build() এ validation → নিরাপদ অবজেক্ট                  │
│     ④ Immutable Builder → thread-safe ও predictable           │
│     ⑤ Step Builder → compile-time ক্রম enforcement            │
│                                                                │
│  📝 সূত্র:                                                     │
│     "যদি constructor-এ ৪+ optional parameter থাকে,            │
│      Builder ব্যবহার করুন।"                                   │
│                                                                │
│  🏆 সেরা ব্যবহার: Query Builder, Email Builder,               │
│     Config Builder, Test Data Builder                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

> **💡 চূড়ান্ত কথা:** Builder প্যাটার্ন সফটওয়্যার ডেভেলপমেন্টের সবচেয়ে ব্যবহারিক প্যাটার্নগুলোর একটি। Laravel Query Builder, Guzzle HTTP Client, PHPUnit MockBuilder — সবখানেই এই প্যাটার্ন ব্যবহৃত হচ্ছে। জটিল অবজেক্ট তৈরিতে এটি আপনার প্রথম পছন্দ হওয়া উচিত।
