# 🎯 Strategy (স্ট্র্যাটেজি) প্যাটার্ন

## 📌 সংজ্ঞা

> **GoF Definition:** "Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it."

Strategy প্যাটার্ন হলো একটি **Behavioral Design Pattern** যা একগুচ্ছ অ্যালগরিদমকে আলাদা আলাদা ক্লাসে encapsulate করে এবং তাদের পরস্পর বিনিময়যোগ্য (interchangeable) করে তোলে। এই প্যাটার্নের মূল উদ্দেশ্য হলো — **runtime-এ অ্যালগরিদম পরিবর্তন করার সুযোগ দেওয়া** কোনো ক্লায়েন্ট কোড পরিবর্তন না করেই।

### মূল ধারণা

ধরুন আপনার একটি সিস্টেমে একই কাজ বিভিন্নভাবে করা যায়। যেমন পেমেন্ট প্রসেস করা — bKash, Nagad, CreditCard, CashOnDelivery ইত্যাদি। প্রতিটি পেমেন্ট মেথড আলাদা অ্যালগরিদম অনুসরণ করে, কিন্তু শেষ পর্যন্ত কাজ একটাই — টাকা ট্রান্সফার করা। Strategy প্যাটার্ন এই সমস্যার একটি elegant সমাধান দেয়।

### কেন দরকার?

কোড যখন এরকম হয়:

```php
// ❌ খারাপ পদ্ধতি — Open/Closed Principle ভঙ্গ
function processPayment(string $method, float $amount): void {
    if ($method === 'bkash') {
        // bKash পেমেন্ট লজিক
    } elseif ($method === 'nagad') {
        // Nagad পেমেন্ট লজিক
    } elseif ($method === 'credit_card') {
        // Credit Card পেমেন্ট লজিক
    }
    // নতুন পেমেন্ট মেথড যোগ করতে হলে এই ফাংশন modify করতে হবে!
}
```

এই কোডে প্রতিটি নতুন পেমেন্ট মেথড যোগ করতে গেলে মূল ফাংশন পরিবর্তন করতে হচ্ছে। এটি **SOLID-এর Open/Closed Principle** সরাসরি ভঙ্গ করে। Strategy প্যাটার্ন এই সমস্যা সমাধান করে conditional logic-কে polymorphism দিয়ে replace করে।

---

## 🏠 বাস্তব উদাহরণ — GPS Navigation

কল্পনা করুন আপনি Google Maps ব্যবহার করছেন ঢাকার গুলশান থেকে মতিঝিলে যাওয়ার জন্য। Maps আপনাকে তিনটি route strategy দেয়:

```
🚗 গাড়ি (Driving Strategy):
   গুলশান → বনানী → ফার্মগেট → মতিঝিল
   সময়: ৪৫ মিনিট | দূরত্ব: ১২ কিমি

🚶 হেঁটে (Walking Strategy):
   গুলশান → হাতিরঝিল → কমলাপুর → মতিঝিল
   সময়: ৩ ঘণ্টা | দূরত্ব: ৯ কিমি

🚲 সাইকেলে (Cycling Strategy):
   গুলশান → তেজগাঁও → মতিঝিল
   সময়: ১ ঘণ্টা ১৫ মিনিট | দূরত্ব: ১০ কিমি
```

এখানে **গন্তব্য একই** (মতিঝিল), কিন্তু **যাওয়ার কৌশল ভিন্ন**। ব্যবহারকারী runtime-এ strategy পরিবর্তন করতে পারে। Navigator (Context) জানে না route কীভাবে calculate হচ্ছে — সে শুধু strategy-কে বলে "route দাও"।

**আরও বাস্তব উদাহরণ:**
- **পেমেন্ট:** দারাজে কেনাকাটার সময় bKash/Nagad/COD সিলেক্ট করা
- **Compression:** ফাইল ZIP, RAR, বা 7z ফরম্যাটে compress করা
- **Authentication:** Login-এ email/password, Google OAuth, বা OTP ব্যবহার

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────┐
│                     Strategy Pattern UML                     │
└─────────────────────────────────────────────────────────────┘

  ┌──────────────────────┐         ┌──────────────────────────┐
  │       Context        │         │    <<interface>>          │
  │──────────────────────│         │      Strategy             │
  │ - strategy: Strategy │────────>│──────────────────────────│
  │──────────────────────│         │ + execute(data): Result   │
  │ + setStrategy(s)     │         └──────────┬───────────────┘
  │ + executeStrategy()  │                    │
  └──────────────────────┘                    │
                                 ┌────────────┼────────────┐
                                 │            │            │
                          ┌──────┴─────┐┌─────┴──────┐┌───┴────────┐
                          │ ConcreteA  ││ ConcreteB  ││ ConcreteC  │
                          │────────────││────────────││────────────│
                          │+ execute() ││+ execute() ││+ execute() │
                          └────────────┘└────────────┘└────────────┘

  ─────────────────────────────────────────────────────────────────

  সম্পর্ক ব্যাখ্যা:

  Context ──────> Strategy        : Context Strategy-র reference রাখে
  Strategy <|── ConcreteStrategy  : Concrete ক্লাস Strategy implement করে
  Client ──────> Context          : Client শুধু Context-এর সাথে কথা বলে

  ─────────────────────────────────────────────────────────────────

  কাজের ধারা (Sequence):

  Client          Context              Strategy (Concrete)
    │                │                        │
    │ setStrategy(s) │                        │
    │───────────────>│                        │
    │                │                        │
    │ doWork()       │                        │
    │───────────────>│   execute(data)        │
    │                │───────────────────────>│
    │                │        result          │
    │                │<───────────────────────│
    │    result      │                        │
    │<───────────────│                        │
```

### তিনটি মূল অংশ:

| অংশ | দায়িত্ব | উদাহরণ |
|------|----------|--------|
| **Strategy (Interface)** | অ্যালগরিদমের common interface সংজ্ঞায়িত করে | `PaymentStrategy` |
| **Concrete Strategy** | নির্দিষ্ট অ্যালগরিদম implement করে | `BkashPayment`, `NagadPayment` |
| **Context** | Strategy-র reference রাখে এবং delegate করে | `PaymentProcessor` |

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Strategy — Navigation উদাহরণ

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

// Strategy Interface
interface RouteStrategy
{
    public function buildRoute(string $origin, string $destination): RouteResult;
}

// Value Object — Result
readonly class RouteResult
{
    public function __construct(
        public string $description,
        public float $distanceKm,
        public int $estimatedMinutes,
        public string $mode,
    ) {}

    public function __toString(): string
    {
        return sprintf(
            "[%s] %s → দূরত্ব: %.1f কিমি, সময়: %d মিনিট",
            $this->mode,
            $this->description,
            $this->distanceKm,
            $this->estimatedMinutes
        );
    }
}

// Concrete Strategy — Driving
class DrivingStrategy implements RouteStrategy
{
    public function buildRoute(string $origin, string $destination): RouteResult
    {
        return new RouteResult(
            description: "{$origin} থেকে {$destination} — মূল সড়ক দিয়ে",
            distanceKm: 12.5,
            estimatedMinutes: 45,
            mode: '🚗 গাড়ি',
        );
    }
}

// Concrete Strategy — Walking
class WalkingStrategy implements RouteStrategy
{
    public function buildRoute(string $origin, string $destination): RouteResult
    {
        return new RouteResult(
            description: "{$origin} থেকে {$destination} — শর্টকাট পথ দিয়ে",
            distanceKm: 9.0,
            estimatedMinutes: 180,
            mode: '🚶 হাঁটা',
        );
    }
}

// Concrete Strategy — Cycling
class CyclingStrategy implements RouteStrategy
{
    public function buildRoute(string $origin, string $destination): RouteResult
    {
        return new RouteResult(
            description: "{$origin} থেকে {$destination} — সাইকেল লেন দিয়ে",
            distanceKm: 10.0,
            estimatedMinutes: 75,
            mode: '🚲 সাইকেল',
        );
    }
}

// Context
class Navigator
{
    private RouteStrategy $strategy;

    public function __construct(RouteStrategy $strategy)
    {
        $this->strategy = $strategy;
    }

    public function setStrategy(RouteStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }

    public function navigate(string $origin, string $destination): RouteResult
    {
        return $this->strategy->buildRoute($origin, $destination);
    }
}

// ব্যবহার
$navigator = new Navigator(new DrivingStrategy());
echo $navigator->navigate('গুলশান', 'মতিঝিল') . PHP_EOL;

$navigator->setStrategy(new WalkingStrategy());
echo $navigator->navigate('গুলশান', 'মতিঝিল') . PHP_EOL;

$navigator->setStrategy(new CyclingStrategy());
echo $navigator->navigate('গুলশান', 'মতিঝিল') . PHP_EOL;
```

**JavaScript (ES2022+):**

```javascript
// Strategy Interface (duck typing + JSDoc)
/**
 * @typedef {Object} RouteResult
 * @property {string} description
 * @property {number} distanceKm
 * @property {number} estimatedMinutes
 * @property {string} mode
 */

// Concrete Strategies — class-based
class DrivingStrategy {
    buildRoute(origin, destination) {
        return {
            description: `${origin} থেকে ${destination} — মূল সড়ক দিয়ে`,
            distanceKm: 12.5,
            estimatedMinutes: 45,
            mode: '🚗 গাড়ি',
        };
    }
}

class WalkingStrategy {
    buildRoute(origin, destination) {
        return {
            description: `${origin} থেকে ${destination} — শর্টকাট পথ দিয়ে`,
            distanceKm: 9.0,
            estimatedMinutes: 180,
            mode: '🚶 হাঁটা',
        };
    }
}

class CyclingStrategy {
    buildRoute(origin, destination) {
        return {
            description: `${origin} থেকে ${destination} — সাইকেল লেন দিয়ে`,
            distanceKm: 10.0,
            estimatedMinutes: 75,
            mode: '🚲 সাইকেল',
        };
    }
}

// Context
class Navigator {
    #strategy;

    constructor(strategy) {
        this.#strategy = strategy;
    }

    setStrategy(strategy) {
        this.#strategy = strategy;
    }

    navigate(origin, destination) {
        const result = this.#strategy.buildRoute(origin, destination);
        console.log(
            `[${result.mode}] ${result.description} → দূরত্ব: ${result.distanceKm} কিমি, সময়: ${result.estimatedMinutes} মিনিট`
        );
        return result;
    }
}

// ব্যবহার
const navigator = new Navigator(new DrivingStrategy());
navigator.navigate('গুলশান', 'মতিঝিল');

navigator.setStrategy(new CyclingStrategy());
navigator.navigate('গুলশান', 'মতিঝিল');
```

---

### ২. Payment Strategy — বাংলাদেশ কন্টেক্সট (bKash, Nagad, SSLCommerz, COD)

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

// Payment Result DTO
readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public float $amount,
        public float $charge,
        public string $provider,
        public string $message,
    ) {}

    public function netAmount(): float
    {
        return $this->amount - $this->charge;
    }
}

// Strategy Interface
interface PaymentStrategy
{
    public function pay(float $amount, array $metadata = []): PaymentResult;
    public function supportsRefund(): bool;
    public function getProviderName(): string;
    public function calculateCharge(float $amount): float;
}

// bKash Payment Strategy
class BkashPayment implements PaymentStrategy
{
    private const float CHARGE_PERCENT = 1.5;
    private const float MIN_AMOUNT = 10.0;
    private const float MAX_AMOUNT = 25_000.0;

    public function pay(float $amount, array $metadata = []): PaymentResult
    {
        $phone = $metadata['phone'] ?? throw new \InvalidArgumentException('bKash-এ ফোন নম্বর আবশ্যক');

        if ($amount < self::MIN_AMOUNT || $amount > self::MAX_AMOUNT) {
            throw new \RangeException(
                sprintf('bKash-এ %.2f থেকে %.2f টাকা পাঠানো যায়', self::MIN_AMOUNT, self::MAX_AMOUNT)
            );
        }

        $charge = $this->calculateCharge($amount);
        $txnId = 'BK' . strtoupper(bin2hex(random_bytes(8)));

        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            amount: $amount,
            charge: $charge,
            provider: $this->getProviderName(),
            message: "bKash পেমেন্ট সফল — {$phone} থেকে ৳{$amount} কাটা হয়েছে",
        );
    }

    public function supportsRefund(): bool
    {
        return true;
    }

    public function getProviderName(): string
    {
        return 'bKash';
    }

    public function calculateCharge(float $amount): float
    {
        return round($amount * self::CHARGE_PERCENT / 100, 2);
    }
}

// Nagad Payment Strategy
class NagadPayment implements PaymentStrategy
{
    private const float CHARGE_PERCENT = 1.0;

    public function pay(float $amount, array $metadata = []): PaymentResult
    {
        $phone = $metadata['phone'] ?? throw new \InvalidArgumentException('Nagad-এ ফোন নম্বর আবশ্যক');

        $charge = $this->calculateCharge($amount);
        $txnId = 'NGD' . strtoupper(bin2hex(random_bytes(8)));

        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            amount: $amount,
            charge: $charge,
            provider: $this->getProviderName(),
            message: "Nagad পেমেন্ট সফল — {$phone} থেকে ৳{$amount} কাটা হয়েছে",
        );
    }

    public function supportsRefund(): bool
    {
        return true;
    }

    public function getProviderName(): string
    {
        return 'Nagad';
    }

    public function calculateCharge(float $amount): float
    {
        return round($amount * self::CHARGE_PERCENT / 100, 2);
    }
}

// Credit Card Payment Strategy (SSLCommerz)
class SSLCommerzPayment implements PaymentStrategy
{
    private const float CHARGE_PERCENT = 2.5;

    public function __construct(
        private readonly string $storeId,
        private readonly string $storePassword,
    ) {}

    public function pay(float $amount, array $metadata = []): PaymentResult
    {
        $cardLast4 = $metadata['card_last4'] ?? '****';
        $charge = $this->calculateCharge($amount);
        $txnId = 'SSL' . strtoupper(bin2hex(random_bytes(8)));

        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            amount: $amount,
            charge: $charge,
            provider: $this->getProviderName(),
            message: "SSLCommerz পেমেন্ট সফল — কার্ড ****{$cardLast4} থেকে ৳{$amount}",
        );
    }

    public function supportsRefund(): bool
    {
        return true;
    }

    public function getProviderName(): string
    {
        return 'SSLCommerz';
    }

    public function calculateCharge(float $amount): float
    {
        return round($amount * self::CHARGE_PERCENT / 100, 2);
    }
}

// Cash on Delivery Strategy
class CashOnDelivery implements PaymentStrategy
{
    private const float FLAT_CHARGE = 50.0;

    public function pay(float $amount, array $metadata = []): PaymentResult
    {
        $txnId = 'COD' . strtoupper(bin2hex(random_bytes(8)));

        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            amount: $amount,
            charge: self::FLAT_CHARGE,
            provider: $this->getProviderName(),
            message: "ক্যাশ অন ডেলিভারি নিশ্চিত — ৳{$amount} (+ ৳" . self::FLAT_CHARGE . " ডেলিভারি চার্জ)",
        );
    }

    public function supportsRefund(): bool
    {
        return false;
    }

    public function getProviderName(): string
    {
        return 'Cash on Delivery';
    }

    public function calculateCharge(float $amount): float
    {
        return self::FLAT_CHARGE;
    }
}

// Context — Payment Processor
class PaymentProcessor
{
    private PaymentStrategy $strategy;

    /** @var array<string, float> পেমেন্ট হিস্ট্রি */
    private array $history = [];

    public function __construct(PaymentStrategy $strategy)
    {
        $this->strategy = $strategy;
    }

    public function setPaymentMethod(PaymentStrategy $strategy): self
    {
        $this->strategy = $strategy;
        return $this;
    }

    public function checkout(float $amount, array $metadata = []): PaymentResult
    {
        $charge = $this->strategy->calculateCharge($amount);
        echo "প্রোভাইডার: {$this->strategy->getProviderName()}" . PHP_EOL;
        echo "মূল্য: ৳{$amount} | চার্জ: ৳{$charge} | মোট: ৳" . ($amount + $charge) . PHP_EOL;

        $result = $this->strategy->pay($amount, $metadata);
        $this->history[$result->transactionId] = $amount;

        return $result;
    }

    public function canRefund(): bool
    {
        return $this->strategy->supportsRefund();
    }
}

// ব্যবহার
$processor = new PaymentProcessor(new BkashPayment());
$result = $processor->checkout(1500.00, ['phone' => '01712345678']);
echo $result->message . PHP_EOL;

// runtime-এ strategy পরিবর্তন
$processor->setPaymentMethod(new NagadPayment());
$result = $processor->checkout(2000.00, ['phone' => '01812345678']);
echo $result->message . PHP_EOL;

$processor->setPaymentMethod(new CashOnDelivery());
$result = $processor->checkout(3500.00);
echo $result->message . PHP_EOL;
```

**JavaScript (ES2022+):**

```javascript
// Strategy Interface enforced via base class
class PaymentStrategy {
    pay(amount, metadata = {}) {
        throw new Error('pay() must be implemented');
    }
    supportsRefund() { return false; }
    get providerName() { throw new Error('providerName must be defined'); }
    calculateCharge(amount) { return 0; }
}

class BkashPayment extends PaymentStrategy {
    static #CHARGE_PERCENT = 1.5;
    static #MAX_AMOUNT = 25_000;

    pay(amount, metadata = {}) {
        const { phone } = metadata;
        if (!phone) throw new Error('bKash-এ ফোন নম্বর আবশ্যক');
        if (amount > BkashPayment.#MAX_AMOUNT) {
            throw new RangeError(`bKash-এ সর্বোচ্চ ৳${BkashPayment.#MAX_AMOUNT} পাঠানো যায়`);
        }

        const txnId = `BK${crypto.randomUUID().slice(0, 16)}`;
        return {
            success: true,
            transactionId: txnId,
            amount,
            charge: this.calculateCharge(amount),
            provider: this.providerName,
            message: `bKash পেমেন্ট সফল — ${phone} থেকে ৳${amount} কাটা হয়েছে`,
        };
    }

    supportsRefund() { return true; }
    get providerName() { return 'bKash'; }

    calculateCharge(amount) {
        return Math.round(amount * BkashPayment.#CHARGE_PERCENT) / 100;
    }
}

class NagadPayment extends PaymentStrategy {
    static #CHARGE_PERCENT = 1.0;

    pay(amount, metadata = {}) {
        const { phone } = metadata;
        if (!phone) throw new Error('Nagad-এ ফোন নম্বর আবশ্যক');

        const txnId = `NGD${crypto.randomUUID().slice(0, 16)}`;
        return {
            success: true,
            transactionId: txnId,
            amount,
            charge: this.calculateCharge(amount),
            provider: this.providerName,
            message: `Nagad পেমেন্ট সফল — ${phone} থেকে ৳${amount} কাটা হয়েছে`,
        };
    }

    supportsRefund() { return true; }
    get providerName() { return 'Nagad'; }

    calculateCharge(amount) {
        return Math.round(amount * NagadPayment.#CHARGE_PERCENT) / 100;
    }
}

class CashOnDelivery extends PaymentStrategy {
    static #FLAT_CHARGE = 50;

    pay(amount, _metadata = {}) {
        const txnId = `COD${crypto.randomUUID().slice(0, 16)}`;
        return {
            success: true,
            transactionId: txnId,
            amount,
            charge: CashOnDelivery.#FLAT_CHARGE,
            provider: this.providerName,
            message: `ক্যাশ অন ডেলিভারি নিশ্চিত — ৳${amount} (+ ৳${CashOnDelivery.#FLAT_CHARGE} ডেলিভারি চার্জ)`,
        };
    }

    get providerName() { return 'Cash on Delivery'; }
    calculateCharge(_amount) { return CashOnDelivery.#FLAT_CHARGE; }
}

// Context
class PaymentProcessor {
    #strategy;
    #history = new Map();

    constructor(strategy) {
        this.#strategy = strategy;
    }

    setPaymentMethod(strategy) {
        this.#strategy = strategy;
        return this;
    }

    checkout(amount, metadata = {}) {
        const charge = this.#strategy.calculateCharge(amount);
        console.log(`প্রোভাইডার: ${this.#strategy.providerName}`);
        console.log(`মূল্য: ৳${amount} | চার্জ: ৳${charge} | মোট: ৳${amount + charge}`);

        const result = this.#strategy.pay(amount, metadata);
        this.#history.set(result.transactionId, amount);
        return result;
    }
}

// ব্যবহার
const processor = new PaymentProcessor(new BkashPayment());
console.log(processor.checkout(1500, { phone: '01712345678' }).message);

processor.setPaymentMethod(new NagadPayment());
console.log(processor.checkout(2000, { phone: '01812345678' }).message);

processor.setPaymentMethod(new CashOnDelivery());
console.log(processor.checkout(3500).message);
```

---

### ৩. Sorting Strategy — ডেটা সাইজ অনুযায়ী অ্যালগরিদম নির্বাচন

এই উদাহরণে Context নিজেই ডেটা সাইজ দেখে সবচেয়ে উপযুক্ত strategy বেছে নেবে।

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

interface SortStrategy
{
    /** @param array<int> $data */
    public function sort(array &$data): void;
    public function getName(): string;
    public function getTimeComplexity(): string;
}

class BubbleSortStrategy implements SortStrategy
{
    public function sort(array &$data): void
    {
        $n = count($data);
        for ($i = 0; $i < $n - 1; $i++) {
            $swapped = false;
            for ($j = 0; $j < $n - $i - 1; $j++) {
                if ($data[$j] > $data[$j + 1]) {
                    [$data[$j], $data[$j + 1]] = [$data[$j + 1], $data[$j]];
                    $swapped = true;
                }
            }
            if (!$swapped) break;
        }
    }

    public function getName(): string { return 'Bubble Sort'; }
    public function getTimeComplexity(): string { return 'O(n²)'; }
}

class QuickSortStrategy implements SortStrategy
{
    public function sort(array &$data): void
    {
        $this->quickSort($data, 0, count($data) - 1);
    }

    private function quickSort(array &$arr, int $low, int $high): void
    {
        if ($low >= $high) return;

        $pivot = $arr[$high];
        $i = $low - 1;

        for ($j = $low; $j < $high; $j++) {
            if ($arr[$j] <= $pivot) {
                $i++;
                [$arr[$i], $arr[$j]] = [$arr[$j], $arr[$i]];
            }
        }
        [$arr[$i + 1], $arr[$high]] = [$arr[$high], $arr[$i + 1]];
        $pi = $i + 1;

        $this->quickSort($arr, $low, $pi - 1);
        $this->quickSort($arr, $pi + 1, $high);
    }

    public function getName(): string { return 'Quick Sort'; }
    public function getTimeComplexity(): string { return 'O(n log n) avg'; }
}

class MergeSortStrategy implements SortStrategy
{
    public function sort(array &$data): void
    {
        $data = $this->mergeSort($data);
    }

    private function mergeSort(array $arr): array
    {
        if (count($arr) <= 1) return $arr;

        $mid = intdiv(count($arr), 2);
        $left = $this->mergeSort(array_slice($arr, 0, $mid));
        $right = $this->mergeSort(array_slice($arr, $mid));

        return $this->merge($left, $right);
    }

    private function merge(array $left, array $right): array
    {
        $result = [];
        $i = $j = 0;

        while ($i < count($left) && $j < count($right)) {
            $result[] = ($left[$i] <= $right[$j]) ? $left[$i++] : $right[$j++];
        }

        return [...$result, ...array_slice($left, $i), ...array_slice($right, $j)];
    }

    public function getName(): string { return 'Merge Sort'; }
    public function getTimeComplexity(): string { return 'O(n log n)'; }
}

// Context — ডেটা সাইজ অনুযায়ী strategy অটো-সিলেক্ট করে
class Sorter
{
    private ?SortStrategy $strategy = null;

    public function setStrategy(SortStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }

    public function autoSelectAndSort(array &$data): string
    {
        $this->strategy = match (true) {
            count($data) <= 50   => new BubbleSortStrategy(),
            count($data) <= 1000 => new QuickSortStrategy(),
            default              => new MergeSortStrategy(),
        };

        $start = hrtime(true);
        $this->strategy->sort($data);
        $elapsed = (hrtime(true) - $start) / 1_000_000;

        return sprintf(
            "%s ব্যবহার করা হয়েছে (ডেটা সাইজ: %d, জটিলতা: %s, সময়: %.2fms)",
            $this->strategy->getName(),
            count($data),
            $this->strategy->getTimeComplexity(),
            $elapsed
        );
    }
}

// ব্যবহার
$sorter = new Sorter();

$small = [64, 34, 25, 12, 22, 11, 90];
echo $sorter->autoSelectAndSort($small) . PHP_EOL;
// → Bubble Sort ব্যবহার করা হয়েছে (ডেটা সাইজ: 7, ...)

$medium = range(1, 500);
shuffle($medium);
echo $sorter->autoSelectAndSort($medium) . PHP_EOL;
// → Quick Sort ব্যবহার করা হয়েছে (ডেটা সাইজ: 500, ...)

$large = range(1, 5000);
shuffle($large);
echo $sorter->autoSelectAndSort($large) . PHP_EOL;
// → Merge Sort ব্যবহার করা হয়েছে (ডেটা সাইজ: 5000, ...)
```

**JavaScript (ES2022+):**

```javascript
class BubbleSortStrategy {
    sort(data) {
        const arr = [...data];
        for (let i = 0; i < arr.length - 1; i++) {
            let swapped = false;
            for (let j = 0; j < arr.length - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
                    swapped = true;
                }
            }
            if (!swapped) break;
        }
        return arr;
    }
    get name() { return 'Bubble Sort'; }
    get complexity() { return 'O(n²)'; }
}

class QuickSortStrategy {
    sort(data) {
        if (data.length <= 1) return [...data];
        const [pivot, ...rest] = data;
        const left = rest.filter(x => x <= pivot);
        const right = rest.filter(x => x > pivot);
        return [...this.sort(left), pivot, ...this.sort(right)];
    }
    get name() { return 'Quick Sort'; }
    get complexity() { return 'O(n log n) avg'; }
}

class MergeSortStrategy {
    sort(data) {
        if (data.length <= 1) return [...data];
        const mid = Math.floor(data.length / 2);
        const left = this.sort(data.slice(0, mid));
        const right = this.sort(data.slice(mid));
        return this.#merge(left, right);
    }

    #merge(left, right) {
        const result = [];
        let i = 0, j = 0;
        while (i < left.length && j < right.length) {
            result.push(left[i] <= right[j] ? left[i++] : right[j++]);
        }
        return [...result, ...left.slice(i), ...right.slice(j)];
    }
    get name() { return 'Merge Sort'; }
    get complexity() { return 'O(n log n)'; }
}

// Context — অটো সিলেক্ট
class Sorter {
    #strategy = null;

    autoSelectAndSort(data) {
        this.#strategy = data.length <= 50
            ? new BubbleSortStrategy()
            : data.length <= 1000
                ? new QuickSortStrategy()
                : new MergeSortStrategy();

        const start = performance.now();
        const sorted = this.#strategy.sort(data);
        const elapsed = (performance.now() - start).toFixed(2);

        console.log(
            `${this.#strategy.name} ব্যবহার করা হয়েছে ` +
            `(ডেটা সাইজ: ${data.length}, জটিলতা: ${this.#strategy.complexity}, সময়: ${elapsed}ms)`
        );
        return sorted;
    }
}

const sorter = new Sorter();
sorter.autoSelectAndSort([64, 34, 25, 12, 22, 11, 90]);
sorter.autoSelectAndSort(Array.from({ length: 500 }, () => Math.random() * 1000 | 0));
sorter.autoSelectAndSort(Array.from({ length: 5000 }, () => Math.random() * 10000 | 0));
```

---

### ৪. Pricing/Discount Strategy

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

interface DiscountStrategy
{
    public function calculate(float $originalPrice): float;
    public function getDescription(): string;
}

class PercentageDiscount implements DiscountStrategy
{
    public function __construct(private readonly float $percent) {}

    public function calculate(float $originalPrice): float
    {
        return round($originalPrice * (1 - $this->percent / 100), 2);
    }

    public function getDescription(): string
    {
        return "{$this->percent}% ছাড়";
    }
}

class FlatDiscount implements DiscountStrategy
{
    public function __construct(private readonly float $discountAmount) {}

    public function calculate(float $originalPrice): float
    {
        return max(0, $originalPrice - $this->discountAmount);
    }

    public function getDescription(): string
    {
        return "৳{$this->discountAmount} ফ্ল্যাট ছাড়";
    }
}

class BuyOneGetOne implements DiscountStrategy
{
    public function calculate(float $originalPrice): float
    {
        // দুইটি পণ্যের দামে একটি ফ্রি = ৫০% ছাড়
        return round($originalPrice / 2, 2);
    }

    public function getDescription(): string
    {
        return 'একটি কিনলে একটি ফ্রি (BOGO)';
    }
}

// বাংলাদেশ কন্টেক্সট — ঈদ/পূজা বিশেষ ছাড়
class RegionalFestivalDiscount implements DiscountStrategy
{
    public function __construct(
        private readonly string $festivalName,
        private readonly float $percent,
        private readonly float $maxDiscount,
    ) {}

    public function calculate(float $originalPrice): float
    {
        $discount = $originalPrice * $this->percent / 100;
        $discount = min($discount, $this->maxDiscount);
        return round($originalPrice - $discount, 2);
    }

    public function getDescription(): string
    {
        return "{$this->festivalName} স্পেশাল — {$this->percent}% ছাড় (সর্বোচ্চ ৳{$this->maxDiscount})";
    }
}

// Context
class PricingEngine
{
    private DiscountStrategy $strategy;

    public function __construct(DiscountStrategy $strategy)
    {
        $this->strategy = $strategy;
    }

    public function setDiscount(DiscountStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }

    public function getFinalPrice(float $originalPrice): float
    {
        $final = $this->strategy->calculate($originalPrice);
        echo "{$this->strategy->getDescription()}: ৳{$originalPrice} → ৳{$final}" . PHP_EOL;
        return $final;
    }
}

// ব্যবহার
$engine = new PricingEngine(new PercentageDiscount(15));
$engine->getFinalPrice(5000);
// → 15% ছাড়: ৳5000 → ৳4250

$engine->setDiscount(new FlatDiscount(500));
$engine->getFinalPrice(3000);
// → ৳500 ফ্ল্যাট ছাড়: ৳3000 → ৳2500

$engine->setDiscount(new RegionalFestivalDiscount('ঈদ-উল-ফিতর', 25, 1000));
$engine->getFinalPrice(8000);
// → ঈদ-উল-ফিতর স্পেশাল — 25% ছাড় (সর্বোচ্চ ৳1000): ৳8000 → ৳7000
```

---

### ৫. Shipping Strategy

**JavaScript (ES2022+):**

```javascript
class ShippingStrategy {
    calculate(_weight, _distance) { throw new Error('implement calculate()'); }
    get estimatedDays() { throw new Error('implement estimatedDays'); }
    get name() { throw new Error('implement name'); }
}

class StandardShipping extends ShippingStrategy {
    calculate(weight, distance) {
        const base = 60;
        return base + weight * 5 + distance * 0.5;
    }
    get estimatedDays() { return '৫-৭ দিন'; }
    get name() { return 'সাধারণ ডেলিভারি'; }
}

class ExpressShipping extends ShippingStrategy {
    calculate(weight, distance) {
        const base = 120;
        return base + weight * 10 + distance * 1.5;
    }
    get estimatedDays() { return '২-৩ দিন'; }
    get name() { return 'এক্সপ্রেস ডেলিভারি'; }
}

class SameDayShipping extends ShippingStrategy {
    static #AVAILABLE_CITIES = ['ঢাকা', 'চট্টগ্রাম'];

    calculate(weight, _distance) {
        const base = 250;
        return base + weight * 20;
    }
    get estimatedDays() { return 'আজকেই'; }
    get name() { return 'সেম ডে ডেলিভারি'; }

    isAvailable(city) {
        return SameDayShipping.#AVAILABLE_CITIES.includes(city);
    }
}

// Context
class ShippingCalculator {
    #strategy;

    constructor(strategy) {
        this.#strategy = strategy;
    }

    setStrategy(strategy) {
        this.#strategy = strategy;
    }

    getQuote(weight, distance) {
        const cost = this.#strategy.calculate(weight, distance);
        return {
            method: this.#strategy.name,
            cost: Math.round(cost),
            delivery: this.#strategy.estimatedDays,
        };
    }
}

// ব্যবহার — সকল অপশন দেখানো
function getAllShippingQuotes(weight, distance) {
    const strategies = [new StandardShipping(), new ExpressShipping(), new SameDayShipping()];
    const calc = new ShippingCalculator(strategies[0]);

    return strategies.map(strategy => {
        calc.setStrategy(strategy);
        return calc.getQuote(weight, distance);
    });
}

const quotes = getAllShippingQuotes(2.5, 300);
quotes.forEach(q => {
    console.log(`${q.method}: ৳${q.cost} (${q.delivery})`);
});
// সাধারণ ডেলিভারি: ৳223 (৫-৭ দিন)
// এক্সপ্রেস ডেলিভারি: ৳595 (২-৩ দিন)
// সেম ডে ডেলিভারি: ৳300 (আজকেই)
```

---

## 🌍 Real-World Applicable Areas

### ১. Authentication Strategy (পূর্ণ উদাহরণ)

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

readonly class AuthUser
{
    public function __construct(
        public string $id,
        public string $name,
        public string $email,
        public array $roles = [],
    ) {}
}

interface AuthStrategy
{
    public function authenticate(array $credentials): AuthUser;
    public function getName(): string;
}

class JwtAuthStrategy implements AuthStrategy
{
    public function __construct(
        private readonly string $secretKey,
    ) {}

    public function authenticate(array $credentials): AuthUser
    {
        $token = $credentials['token'] ?? throw new \RuntimeException('JWT token আবশ্যক');

        // Token decode ও verify (সরলীকৃত)
        $parts = explode('.', $token);
        if (count($parts) !== 3) {
            throw new \RuntimeException('অবৈধ JWT ফরম্যাট');
        }

        $payload = json_decode(base64_decode($parts[1]), true);

        return new AuthUser(
            id: $payload['sub'],
            name: $payload['name'] ?? 'Unknown',
            email: $payload['email'] ?? '',
            roles: $payload['roles'] ?? [],
        );
    }

    public function getName(): string { return 'JWT'; }
}

class SessionAuthStrategy implements AuthStrategy
{
    public function authenticate(array $credentials): AuthUser
    {
        $sessionId = $credentials['session_id'] ?? throw new \RuntimeException('Session ID আবশ্যক');

        // Session store থেকে user data fetch (সরলীকৃত)
        return new AuthUser(
            id: 'user_' . substr($sessionId, 0, 8),
            name: 'Session User',
            email: 'user@example.com',
        );
    }

    public function getName(): string { return 'Session'; }
}

class ApiKeyAuthStrategy implements AuthStrategy
{
    /** @param array<string, AuthUser> $validKeys */
    public function __construct(
        private readonly array $validKeys,
    ) {}

    public function authenticate(array $credentials): AuthUser
    {
        $apiKey = $credentials['api_key'] ?? throw new \RuntimeException('API Key আবশ্যক');

        return $this->validKeys[$apiKey]
            ?? throw new \RuntimeException('অবৈধ API Key');
    }

    public function getName(): string { return 'API Key'; }
}

class OAuthStrategy implements AuthStrategy
{
    public function __construct(
        private readonly string $provider, // 'google', 'facebook'
        private readonly string $clientId,
        private readonly string $clientSecret,
    ) {}

    public function authenticate(array $credentials): AuthUser
    {
        $code = $credentials['auth_code'] ?? throw new \RuntimeException('OAuth code আবশ্যক');

        // OAuth provider-এ token exchange (সরলীকৃত)
        return new AuthUser(
            id: 'oauth_' . $this->provider . '_' . substr(md5($code), 0, 8),
            name: ucfirst($this->provider) . ' User',
            email: 'user@' . $this->provider . '.com',
        );
    }

    public function getName(): string { return "OAuth ({$this->provider})"; }
}

// Context — Auth Manager
class AuthManager
{
    /** @var array<string, AuthStrategy> */
    private array $strategies = [];
    private ?string $defaultStrategy = null;

    public function registerStrategy(string $name, AuthStrategy $strategy): void
    {
        $this->strategies[$name] = $strategy;
        $this->defaultStrategy ??= $name;
    }

    public function authenticate(string $strategyName, array $credentials): AuthUser
    {
        $strategy = $this->strategies[$strategyName]
            ?? throw new \InvalidArgumentException("অজানা auth strategy: {$strategyName}");

        echo "🔐 {$strategy->getName()} দিয়ে authentication চলছে..." . PHP_EOL;
        return $strategy->authenticate($credentials);
    }
}

// ব্যবহার — Laravel Guard-এর মতো multi-strategy auth
$auth = new AuthManager();
$auth->registerStrategy('jwt', new JwtAuthStrategy('my-secret-key'));
$auth->registerStrategy('session', new SessionAuthStrategy());
$auth->registerStrategy('api', new ApiKeyAuthStrategy([
    'sk_live_abc123' => new AuthUser('1', 'API ব্যবহারকারী', 'api@example.com', ['admin']),
]));
$auth->registerStrategy('google', new OAuthStrategy('google', 'client-id', 'client-secret'));

// API request → API Key auth
$user = $auth->authenticate('api', ['api_key' => 'sk_live_abc123']);
echo "✅ স্বাগতম, {$user->name}!" . PHP_EOL;
```

### ২. Caching Strategy

```php
<?php

interface CacheStrategy
{
    public function get(string $key): mixed;
    public function put(string $key, mixed $value): void;
    public function evict(): ?string; // evicted key ফেরত দেয়
    public function getName(): string;
}

// LRU — সবচেয়ে পুরনো ব্যবহৃত আইটেম বের করে
class LruCacheStrategy implements CacheStrategy
{
    private array $cache = [];
    private int $maxSize;

    public function __construct(int $maxSize = 3)
    {
        $this->maxSize = $maxSize;
    }

    public function get(string $key): mixed
    {
        if (!isset($this->cache[$key])) return null;

        // ব্যবহার হলে শেষে সরান (most recently used)
        $value = $this->cache[$key];
        unset($this->cache[$key]);
        $this->cache[$key] = $value;
        return $value;
    }

    public function put(string $key, mixed $value): void
    {
        if (isset($this->cache[$key])) {
            unset($this->cache[$key]);
        }
        $this->cache[$key] = $value;

        if (count($this->cache) > $this->maxSize) {
            $this->evict();
        }
    }

    public function evict(): ?string
    {
        $evictedKey = array_key_first($this->cache);
        unset($this->cache[$evictedKey]);
        return $evictedKey;
    }

    public function getName(): string { return 'LRU'; }
}

// FIFO — প্রথমে ঢোকা আইটেম প্রথমে বের
class FifoCacheStrategy implements CacheStrategy
{
    private array $cache = [];
    private int $maxSize;

    public function __construct(int $maxSize = 3)
    {
        $this->maxSize = $maxSize;
    }

    public function get(string $key): mixed
    {
        return $this->cache[$key] ?? null;
    }

    public function put(string $key, mixed $value): void
    {
        $this->cache[$key] = $value;

        if (count($this->cache) > $this->maxSize) {
            $this->evict();
        }
    }

    public function evict(): ?string
    {
        $evictedKey = array_key_first($this->cache);
        unset($this->cache[$evictedKey]);
        return $evictedKey;
    }

    public function getName(): string { return 'FIFO'; }
}
```

### ৩. Tax Calculation Strategy (দেশভিত্তিক)

```php
<?php

interface TaxStrategy
{
    public function calculate(float $amount): float;
    public function getRegion(): string;
}

class BangladeshVat implements TaxStrategy
{
    public function calculate(float $amount): float
    {
        return round($amount * 0.15, 2); // ১৫% VAT
    }
    public function getRegion(): string { return 'বাংলাদেশ (১৫% ভ্যাট)'; }
}

class IndiaGst implements TaxStrategy
{
    public function __construct(private readonly float $gstRate = 18.0) {}

    public function calculate(float $amount): float
    {
        return round($amount * $this->gstRate / 100, 2);
    }
    public function getRegion(): string { return "ভারত ({$this->gstRate}% GST)"; }
}

class UsSalesTax implements TaxStrategy
{
    public function __construct(private readonly string $state = 'CA') {}

    public function calculate(float $amount): float
    {
        $rates = ['CA' => 7.25, 'TX' => 6.25, 'NY' => 8.0];
        $rate = $rates[$this->state] ?? 0;
        return round($amount * $rate / 100, 2);
    }
    public function getRegion(): string { return "USA ({$this->state})"; }
}

class TaxCalculator
{
    public function __construct(private TaxStrategy $strategy) {}

    public function setRegion(TaxStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }

    public function getInvoice(float $subtotal): array
    {
        $tax = $this->strategy->calculate($subtotal);
        return [
            'region' => $this->strategy->getRegion(),
            'subtotal' => $subtotal,
            'tax' => $tax,
            'total' => $subtotal + $tax,
        ];
    }
}
```

### ৪. ফ্রেমওয়ার্কে Strategy প্যাটার্ন

#### Laravel-এ Strategy প্যাটার্নের ব্যবহার:

```php
// config/auth.php — Guard Drivers (Strategy Pattern!)
'guards' => [
    'web' => ['driver' => 'session'],    // SessionGuard strategy
    'api' => ['driver' => 'token'],      // TokenGuard strategy
    'sanctum' => ['driver' => 'sanctum'] // SanctumGuard strategy
],

// config/filesystems.php — Filesystem Drivers
'disks' => [
    'local' => ['driver' => 'local'],
    's3'    => ['driver' => 's3'],
    'ftp'   => ['driver' => 'ftp'],
],

// config/cache.php — Cache Stores
'stores' => [
    'file'     => ['driver' => 'file'],
    'redis'    => ['driver' => 'redis'],
    'database' => ['driver' => 'database'],
],

// ব্যবহার — runtime-এ strategy switch
Storage::disk('s3')->put('file.txt', $content);  // S3 strategy
Storage::disk('local')->put('file.txt', $content); // Local strategy
Cache::store('redis')->put('key', 'value', 3600); // Redis strategy
```

#### Passport.js-এ Strategy প্যাটার্ন:

```javascript
import passport from 'passport';
import { Strategy as LocalStrategy } from 'passport-local';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

// প্রতিটি passport.use() একটি auth strategy register করে
passport.use('local', new LocalStrategy(
    { usernameField: 'email' },
    async (email, password, done) => {
        const user = await User.findOne({ email });
        if (!user || !await user.verifyPassword(password)) {
            return done(null, false, { message: 'ভুল email বা password' });
        }
        return done(null, user);
    }
));

passport.use('google', new GoogleStrategy(
    {
        clientID: process.env.GOOGLE_CLIENT_ID,
        clientSecret: process.env.GOOGLE_CLIENT_SECRET,
        callbackURL: '/auth/google/callback',
    },
    async (accessToken, refreshToken, profile, done) => {
        let user = await User.findOne({ googleId: profile.id });
        if (!user) {
            user = await User.create({
                googleId: profile.id,
                name: profile.displayName,
                email: profile.emails[0].value,
            });
        }
        return done(null, user);
    }
));

// route-এ strategy নাম দিয়ে authenticate
app.post('/login', passport.authenticate('local'));
app.get('/auth/google', passport.authenticate('google', { scope: ['profile', 'email'] }));
```

---

## 🔥 Advanced Deep Dive

### ১. Strategy vs State — বিস্তারিত তুলনা

এই দুটি প্যাটার্ন দেখতে প্রায় একই কিন্তু উদ্দেশ্য সম্পূর্ণ আলাদা:

```
┌──────────────────────┬────────────────────────────────────────┐
│       Strategy        │              State                     │
├──────────────────────┼────────────────────────────────────────┤
│ Client strategy বেছে │ State নিজেই পরবর্তী                   │
│ নেয় ও সেট করে       │ state-এ transition করে                 │
│                      │                                        │
│ Strategy-গুলো পরস্পর │ State-গুলো পরস্পরকে                   │
│ সম্পর্কে জানে না     │ জানে (transition logic)                │
│                      │                                        │
│ "কীভাবে করবে" পাল্টায়│ "কী অবস্থায় আছে" পাল্টায়            │
│                      │                                        │
│ Stateless            │ Stateful — অবস্থা ট্র্যাক করে         │
│                      │                                        │
│ উদাহরণ: পেমেন্ট      │ উদাহরণ: অর্ডার                        │
│ মেথড সিলেকশন         │ (pending→confirmed→shipped→delivered)  │
└──────────────────────┴────────────────────────────────────────┘
```

```php
// Strategy — Client সিদ্ধান্ত নেয়
$processor->setPaymentMethod(new BkashPayment()); // Client বলছে কোনটি
$processor->checkout(1000);

// State — Object নিজেই transition করে
$order->process(); // pending → confirmed (order নিজেই পরবর্তী state জানে)
$order->ship();    // confirmed → shipped (order নিজেই transition পরিচালনা করে)
```

### ২. Strategy vs Template Method

```
Strategy:
- Composition ব্যবহার করে (HAS-A সম্পর্ক)
- পুরো অ্যালগরিদম replace হয়
- Runtime-এ পরিবর্তনযোগ্য

Template Method:
- Inheritance ব্যবহার করে (IS-A সম্পর্ক)
- অ্যালগরিদমের নির্দিষ্ট step override হয়
- Compile-time-এ fixed
```

```php
// Strategy — পুরো payment logic encapsulated
class BkashPayment implements PaymentStrategy {
    public function pay(float $amount, array $meta): PaymentResult { /* পুরো logic */ }
}

// Template Method — শুধু নির্দিষ্ট step override
abstract class PaymentTemplate {
    // Template method — structure fixed
    final public function processPayment(float $amount): void {
        $this->validate($amount);
        $this->deductAmount($amount);
        $this->sendConfirmation();
    }

    abstract protected function validate(float $amount): void;   // step override
    abstract protected function deductAmount(float $amount): void; // step override

    protected function sendConfirmation(): void {
        echo "নিশ্চিতকরণ পাঠানো হয়েছে" . PHP_EOL;
    }
}
```

### ৩. Strategy + Factory — Strategy Selection

Runtime-এ কোন strategy ব্যবহার হবে সেটি Factory নির্ধারণ করতে পারে:

**PHP 8.3:**

```php
<?php

// Strategy Factory — কনফিগ বা ইনপুট থেকে strategy তৈরি করে
class PaymentStrategyFactory
{
    /** @var array<string, class-string<PaymentStrategy>> */
    private static array $strategies = [
        'bkash'      => BkashPayment::class,
        'nagad'      => NagadPayment::class,
        'sslcommerz' => SSLCommerzPayment::class,
        'cod'        => CashOnDelivery::class,
    ];

    public static function create(string $method): PaymentStrategy
    {
        $class = self::$strategies[$method]
            ?? throw new \InvalidArgumentException("অসমর্থিত পেমেন্ট মেথড: {$method}");

        return match ($method) {
            'sslcommerz' => new $class(
                storeId: config('sslcommerz.store_id'),
                storePassword: config('sslcommerz.store_password'),
            ),
            default => new $class(),
        };
    }

    public static function register(string $name, string $class): void
    {
        if (!is_subclass_of($class, PaymentStrategy::class)) {
            throw new \InvalidArgumentException("{$class} must implement PaymentStrategy");
        }
        self::$strategies[$name] = $class;
    }

    /** @return array<string> */
    public static function available(): array
    {
        return array_keys(self::$strategies);
    }
}

// ব্যবহার — HTTP request থেকে dynamic strategy selection
$method = $_POST['payment_method']; // 'bkash'
$strategy = PaymentStrategyFactory::create($method);
$processor = new PaymentProcessor($strategy);
$result = $processor->checkout(1500, ['phone' => '01712345678']);
```

**JavaScript (ES2022+):**

```javascript
class PaymentStrategyFactory {
    static #registry = new Map([
        ['bkash', BkashPayment],
        ['nagad', NagadPayment],
        ['cod', CashOnDelivery],
    ]);

    static create(method) {
        const StrategyClass = this.#registry.get(method);
        if (!StrategyClass) {
            throw new Error(`অসমর্থিত পেমেন্ট মেথড: ${method}`);
        }
        return new StrategyClass();
    }

    static register(name, strategyClass) {
        this.#registry.set(name, strategyClass);
    }

    static get available() {
        return [...this.#registry.keys()];
    }
}

// ব্যবহার
const strategy = PaymentStrategyFactory.create('bkash');
const processor = new PaymentProcessor(strategy);
```

### ৪. Strategy with Lambda/Closure — Functional Approach

যখন strategy-র logic খুব ছোট, তখন পুরো class তৈরি করার দরকার নেই। Closure দিয়েই কাজ চলে।

**PHP 8.3 — First-class callable syntax:**

```php
<?php

class FunctionalPricingEngine
{
    /** @var \Closure(float): float */
    private \Closure $discountStrategy;

    public function __construct(callable $strategy)
    {
        $this->discountStrategy = $strategy(...); // first-class callable
    }

    public function setStrategy(callable $strategy): void
    {
        $this->discountStrategy = $strategy(...);
    }

    public function calculate(float $price): float
    {
        return ($this->discountStrategy)($price);
    }
}

// Lambda strategies — class তৈরি করতে হচ্ছে না!
$percentOff = fn(float $p): float => $p * 0.85;       // ১৫% ছাড়
$flatOff    = fn(float $p): float => max(0, $p - 500); // ৳৫০০ ছাড়
$noDiscount = fn(float $p): float => $p;               // কোনো ছাড় নেই

// Higher-order function দিয়ে strategy তৈরি
function cappedPercentDiscount(float $percent, float $maxDiscount): \Closure
{
    return function (float $price) use ($percent, $maxDiscount): float {
        $discount = min($price * $percent / 100, $maxDiscount);
        return $price - $discount;
    };
}

$engine = new FunctionalPricingEngine($percentOff);
echo $engine->calculate(5000) . PHP_EOL; // 4250

$engine->setStrategy(cappedPercentDiscount(25, 1000));
echo $engine->calculate(8000) . PHP_EOL; // 7000

// PHP 8.3 first-class callable — বিল্ট-ইন ফাংশনও strategy হিসেবে
$engine->setStrategy(ceil(...));  // ceil() কে strategy হিসেবে ব্যবহার
$engine->setStrategy(floor(...)); // floor() কে strategy হিসেবে ব্যবহার
```

**JavaScript — Arrow functions as strategies:**

```javascript
class FunctionalPricingEngine {
    #strategy;

    constructor(strategy) {
        this.#strategy = strategy;
    }

    setStrategy(strategy) {
        this.#strategy = strategy;
    }

    calculate(price) {
        return this.#strategy(price);
    }
}

// Arrow function strategies
const percentOff = (p) => p * 0.85;
const flatOff = (p) => Math.max(0, p - 500);

// Strategy factory via closure
const cappedDiscount = (percent, maxDiscount) => (price) => {
    const discount = Math.min(price * percent / 100, maxDiscount);
    return price - discount;
};

const engine = new FunctionalPricingEngine(percentOff);
console.log(engine.calculate(5000)); // 4250

engine.setStrategy(cappedDiscount(25, 1000));
console.log(engine.calculate(8000)); // 7000

// Strategy Map — Object literal as strategy registry
const discountStrategies = {
    regular: (p) => p,
    silver: (p) => p * 0.90,
    gold: (p) => p * 0.80,
    platinum: (p) => p * 0.70,
};

const userTier = 'gold';
engine.setStrategy(discountStrategies[userTier]);
console.log(engine.calculate(10000)); // 8000
```

### ৫. Strategy + Dependency Injection (DI Container)

**PHP 8.3 — Laravel-স্টাইল DI:**

```php
<?php

// Service Provider-এ strategy binding
class PaymentServiceProvider
{
    public function register(Container $container): void
    {
        // কনফিগ থেকে ডিফল্ট strategy নির্ধারণ
        $container->bind(PaymentStrategy::class, function () {
            $default = getenv('DEFAULT_PAYMENT') ?: 'bkash';
            return PaymentStrategyFactory::create($default);
        });

        // Context-এ auto-inject
        $container->bind(PaymentProcessor::class, function (Container $c) {
            return new PaymentProcessor($c->make(PaymentStrategy::class));
        });
    }
}

// Controller-এ ব্যবহার — strategy injected হয়
class CheckoutController
{
    public function __construct(
        private readonly PaymentProcessor $processor,
    ) {}

    public function processOrder(Request $request): Response
    {
        // User-এর সিলেক্ট করা method অনুযায়ী strategy সেট
        $strategy = PaymentStrategyFactory::create($request->input('payment_method'));
        $this->processor->setPaymentMethod($strategy);

        $result = $this->processor->checkout(
            amount: $request->input('total'),
            metadata: $request->input('payment_details', []),
        );

        return new Response($result);
    }
}
```

### ৬. Runtime Strategy Switching

```javascript
// Adaptive strategy — পরিস্থিতি অনুযায়ী strategy পাল্টায়
class AdaptiveCompressionContext {
    #strategies;

    constructor() {
        this.#strategies = {
            fast: { compress: (data) => `fast_${data.length}`, ratio: 0.7, speed: 'দ্রুত' },
            balanced: { compress: (data) => `balanced_${data.length}`, ratio: 0.5, speed: 'মাঝারি' },
            best: { compress: (data) => `best_${data.length}`, ratio: 0.3, speed: 'ধীর' },
        };
    }

    compress(data, { prioritize = 'balanced' } = {}) {
        // ডেটা সাইজ ও priority অনুযায়ী strategy পরিবর্তন
        let strategy;
        if (data.length > 1_000_000 && prioritize === 'speed') {
            strategy = this.#strategies.fast;
        } else if (data.length > 100_000) {
            strategy = this.#strategies.balanced;
        } else {
            strategy = this.#strategies.best;
        }

        console.log(`${strategy.speed} compression ব্যবহার হচ্ছে (ratio: ${strategy.ratio})`);
        return strategy.compress(data);
    }
}
```

### ৭. Strategy in Functional Programming

Functional programming-এ Strategy প্যাটার্ন naturally বিদ্যমান কারণ function নিজেই first-class citizen:

```javascript
// Strategy pattern পুরোটাই function composition দিয়ে
const pipe = (...fns) => (x) => fns.reduce((v, f) => f(v), x);

// Individual strategies as pure functions
const applyDiscount = (percent) => (price) => price * (1 - percent / 100);
const applyTax = (rate) => (price) => price * (1 + rate / 100);
const roundUp = (price) => Math.ceil(price);

// Strategy composition — একাধিক strategy একত্রে
const bdPricing = pipe(
    applyDiscount(10),    // ১০% ছাড়
    applyTax(15),         // ১৫% ভ্যাট
    roundUp,
);

const indiaPricing = pipe(
    applyDiscount(5),     // ৫% ছাড়
    applyTax(18),         // ১৮% GST
    roundUp,
);

console.log(bdPricing(1000));    // 1000 → 900 → 1035 → 1035
console.log(indiaPricing(1000)); // 1000 → 950 → 1121 → 1121

// Strategy selection — function-ই strategy
const pricingStrategies = new Map([
    ['BD', bdPricing],
    ['IN', indiaPricing],
]);

function calculatePrice(country, basePrice) {
    const strategy = pricingStrategies.get(country) ?? ((p) => p);
    return strategy(basePrice);
}
```

---

## ✅ Pros (সুবিধা)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|----------|
| 1 | **Open/Closed Principle** | নতুন strategy যোগ করতে বিদ্যমান কোড পরিবর্তন লাগে না |
| 2 | **Runtime flexibility** | চলমান অবস্থায় অ্যালগরিদম পরিবর্তন করা যায় |
| 3 | **Isolation** | প্রতিটি strategy স্বতন্ত্রভাবে develop ও test করা যায় |
| 4 | **if/else দূরীকরণ** | বিশাল conditional block polymorphism দিয়ে replace হয় |
| 5 | **Single Responsibility** | প্রতিটি strategy-র একটি মাত্র দায়িত্ব |
| 6 | **Composition over Inheritance** | Inheritance-এর জটিলতা এড়ানো যায় |
| 7 | **Testability** | প্রতিটি strategy আলাদাভাবে unit test করা যায় |

## ❌ Cons (অসুবিধা)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|----------|
| 1 | **Class explosion** | অনেকগুলো strategy ক্লাস তৈরি হয় |
| 2 | **Client awareness** | Client-কে জানতে হয় কোন strategy কখন ব্যবহার করতে হবে |
| 3 | **Overhead** | ২-৩টি সহজ condition-এর জন্য অতিরিক্ত abstraction |
| 4 | **Communication cost** | Context ও Strategy-র মধ্যে ডেটা আদান-প্রদানে অতিরিক্ত parameter |
| 5 | **Functional alternative** | আধুনিক ভাষায় lambda/closure দিয়ে সহজেই সমাধান হয় |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ১. যেখানে দরকার নেই সেখানে Strategy ব্যবহার

```php
// ❌ মাত্র ২টি অপশন — if/else-ই যথেষ্ট
interface ToggleStrategy {
    public function execute(): void;
}
class OnStrategy implements ToggleStrategy { /* ... */ }
class OffStrategy implements ToggleStrategy { /* ... */ }

// ✅ সরল সমাধানই ভালো
$isOn ? $this->enable() : $this->disable();
```

### ২. Strategy-তে state রাখা

```php
// ❌ Strategy-তে mutable state — বিপজ্জনক!
class DiscountStrategy {
    private float $appliedCount = 0; // State!

    public function calculate(float $price): float {
        $this->appliedCount++; // Side effect!
        return $price * 0.9;
    }
}

// ✅ Strategy stateless হওয়া উচিত
class DiscountStrategy {
    public function calculate(float $price): float {
        return $price * 0.9;
    }
}
```

### ৩. Context-এ null strategy

```php
// ❌ Null check করতে হচ্ছে
class Processor {
    private ?PaymentStrategy $strategy = null;

    public function process(float $amount): void {
        if ($this->strategy === null) {
            throw new \RuntimeException('Strategy সেট করা হয়নি');
        }
        $this->strategy->pay($amount);
    }
}

// ✅ Constructor-এ mandatory করুন অথবা Null Object Pattern ব্যবহার করুন
class Processor {
    public function __construct(
        private PaymentStrategy $strategy, // required
    ) {}
}

// অথবা Null Object Strategy
class NullPaymentStrategy implements PaymentStrategy {
    public function pay(float $amount, array $metadata = []): PaymentResult {
        return new PaymentResult(
            success: false,
            transactionId: '',
            amount: $amount,
            charge: 0,
            provider: 'none',
            message: 'কোনো পেমেন্ট মেথড সিলেক্ট করা হয়নি',
        );
    }
}
```

### ৪. Strategy interface অত্যধিক বড় করা

```php
// ❌ Fat interface — সব strategy-র সব method দরকার নেই
interface PaymentStrategy {
    public function pay(): void;
    public function refund(): void;
    public function recurring(): void;
    public function partialPayment(): void;
    public function installment(): void;
}

// ✅ Interface Segregation Principle মানুন
interface PaymentStrategy {
    public function pay(float $amount): PaymentResult;
}

interface Refundable {
    public function refund(string $transactionId): bool;
}

interface SupportsRecurring {
    public function setupRecurring(float $amount, string $interval): void;
}

// শুধু যেটুকু দরকার সেটুকু implement
class BkashPayment implements PaymentStrategy, Refundable {
    public function pay(float $amount): PaymentResult { /* ... */ }
    public function refund(string $transactionId): bool { /* ... */ }
}
```

### ৫. Hardcoded strategy instantiation

```php
// ❌ Context-এর ভিতরে strategy তৈরি — flexibility নষ্ট
class PaymentProcessor {
    public function process(string $method, float $amount): void {
        $strategy = match($method) {
            'bkash' => new BkashPayment(), // hardcoded!
            'nagad' => new NagadPayment(),
        };
    }
}

// ✅ Injection বা Factory ব্যবহার করুন
class PaymentProcessor {
    public function __construct(private PaymentStrategy $strategy) {}
}
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class PaymentStrategyTest extends TestCase
{
    #[Test]
    public function bkash_payment_succeeds_with_valid_phone(): void
    {
        $strategy = new BkashPayment();
        $result = $strategy->pay(1500.00, ['phone' => '01712345678']);

        $this->assertTrue($result->success);
        $this->assertStringStartsWith('BK', $result->transactionId);
        $this->assertEquals(1500.00, $result->amount);
        $this->assertEquals('bKash', $result->provider);
    }

    #[Test]
    public function bkash_throws_without_phone(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('ফোন নম্বর আবশ্যক');

        (new BkashPayment())->pay(1000.00);
    }

    #[Test]
    public function bkash_throws_on_exceeding_max_amount(): void
    {
        $this->expectException(\RangeException::class);

        (new BkashPayment())->pay(30000.00, ['phone' => '01712345678']);
    }

    #[Test]
    public function cash_on_delivery_does_not_support_refund(): void
    {
        $strategy = new CashOnDelivery();
        $this->assertFalse($strategy->supportsRefund());
    }

    #[Test]
    public function processor_can_switch_strategies_at_runtime(): void
    {
        $processor = new PaymentProcessor(new BkashPayment());
        $result1 = $processor->checkout(500, ['phone' => '01712345678']);
        $this->assertEquals('bKash', $result1->provider);

        $processor->setPaymentMethod(new NagadPayment());
        $result2 = $processor->checkout(500, ['phone' => '01812345678']);
        $this->assertEquals('Nagad', $result2->provider);
    }

    #[DataProvider('chargeDataProvider')]
    #[Test]
    public function charge_calculation_is_correct(
        PaymentStrategy $strategy,
        float $amount,
        float $expectedCharge
    ): void {
        $this->assertEquals($expectedCharge, $strategy->calculateCharge($amount));
    }

    public static function chargeDataProvider(): array
    {
        return [
            'bKash 1000'  => [new BkashPayment(), 1000.0, 15.0],
            'bKash 5000'  => [new BkashPayment(), 5000.0, 75.0],
            'Nagad 1000'  => [new NagadPayment(), 1000.0, 10.0],
            'COD any'     => [new CashOnDelivery(), 999.0, 50.0],
        ];
    }

    #[Test]
    public function mock_strategy_for_context_isolation(): void
    {
        $mockStrategy = $this->createMock(PaymentStrategy::class);
        $mockStrategy->method('calculateCharge')->willReturn(0.0);
        $mockStrategy->method('getProviderName')->willReturn('Mock');
        $mockStrategy->method('pay')->willReturn(
            new PaymentResult(
                success: true,
                transactionId: 'MOCK001',
                amount: 1000.0,
                charge: 0.0,
                provider: 'Mock',
                message: 'Mock পেমেন্ট',
            )
        );

        $processor = new PaymentProcessor($mockStrategy);
        $result = $processor->checkout(1000.0);

        $this->assertTrue($result->success);
        $this->assertEquals('MOCK001', $result->transactionId);
    }
}

class DiscountStrategyTest extends TestCase
{
    #[Test]
    public function percentage_discount_calculates_correctly(): void
    {
        $strategy = new PercentageDiscount(20);
        $this->assertEquals(4000.0, $strategy->calculate(5000.0));
    }

    #[Test]
    public function flat_discount_never_goes_negative(): void
    {
        $strategy = new FlatDiscount(1000);
        $this->assertEquals(0.0, $strategy->calculate(500.0));
    }

    #[Test]
    public function bogo_halves_the_price(): void
    {
        $strategy = new BuyOneGetOne();
        $this->assertEquals(2500.0, $strategy->calculate(5000.0));
    }

    #[Test]
    public function regional_festival_respects_max_cap(): void
    {
        $strategy = new RegionalFestivalDiscount('ঈদ', 50, 1000);
        // 50% of 5000 = 2500, কিন্তু cap 1000 → 5000 - 1000 = 4000
        $this->assertEquals(4000.0, $strategy->calculate(5000.0));
    }
}

class SortStrategyTest extends TestCase
{
    #[DataProvider('sortStrategyProvider')]
    #[Test]
    public function strategy_sorts_correctly(SortStrategy $strategy): void
    {
        $data = [64, 34, 25, 12, 22, 11, 90];
        $expected = [11, 12, 22, 25, 34, 64, 90];

        $strategy->sort($data);

        $this->assertEquals($expected, $data);
    }

    public static function sortStrategyProvider(): array
    {
        return [
            'bubble' => [new BubbleSortStrategy()],
            'quick'  => [new QuickSortStrategy()],
            'merge'  => [new MergeSortStrategy()],
        ];
    }

    #[Test]
    public function sorter_auto_selects_appropriate_strategy(): void
    {
        $sorter = new Sorter();

        $small = [5, 3, 1];
        $result = $sorter->autoSelectAndSort($small);
        $this->assertStringContainsString('Bubble Sort', $result);
    }
}
```

### Jest

```javascript
import { describe, test, expect, jest } from '@jest/globals';

describe('PaymentStrategy', () => {
    describe('BkashPayment', () => {
        const bkash = new BkashPayment();

        test('সফল পেমেন্ট — সঠিক ফোন নম্বর দিলে', () => {
            const result = bkash.pay(1500, { phone: '01712345678' });

            expect(result.success).toBe(true);
            expect(result.transactionId).toMatch(/^BK/);
            expect(result.amount).toBe(1500);
            expect(result.provider).toBe('bKash');
        });

        test('ফোন ছাড়া error throw করে', () => {
            expect(() => bkash.pay(1000)).toThrow('ফোন নম্বর আবশ্যক');
        });

        test('সর্বোচ্চ সীমা অতিক্রম করলে RangeError', () => {
            expect(() => bkash.pay(30000, { phone: '01712345678' })).toThrow(RangeError);
        });
    });

    describe('PaymentProcessor — Runtime strategy switching', () => {
        test('strategy পরিবর্তন হলে provider পরিবর্তন হয়', () => {
            const processor = new PaymentProcessor(new BkashPayment());
            const r1 = processor.checkout(500, { phone: '01712345678' });
            expect(r1.provider).toBe('bKash');

            processor.setPaymentMethod(new NagadPayment());
            const r2 = processor.checkout(500, { phone: '01812345678' });
            expect(r2.provider).toBe('Nagad');
        });
    });

    describe('Mock strategy — Context isolation', () => {
        test('mock strategy দিয়ে context test', () => {
            const mockStrategy = {
                pay: jest.fn().mockReturnValue({
                    success: true,
                    transactionId: 'MOCK001',
                    amount: 1000,
                    charge: 0,
                    provider: 'Mock',
                    message: 'Mock পেমেন্ট',
                }),
                calculateCharge: jest.fn().mockReturnValue(0),
                providerName: 'Mock',
            };

            const processor = new PaymentProcessor(mockStrategy);
            const result = processor.checkout(1000);

            expect(result.success).toBe(true);
            expect(mockStrategy.pay).toHaveBeenCalledWith(1000, {});
            expect(mockStrategy.calculateCharge).toHaveBeenCalledWith(1000);
        });
    });
});

describe('Sorting Strategy', () => {
    const testData = [64, 34, 25, 12, 22, 11, 90];
    const expected = [11, 12, 22, 25, 34, 64, 90];

    test.each([
        ['BubbleSort', new BubbleSortStrategy()],
        ['QuickSort', new QuickSortStrategy()],
        ['MergeSort', new MergeSortStrategy()],
    ])('%s — সঠিকভাবে sort করে', (name, strategy) => {
        const result = strategy.sort([...testData]);
        expect(result).toEqual(expected);
    });

    test('Sorter — ছোট ডেটায় BubbleSort সিলেক্ট হয়', () => {
        const sorter = new Sorter();
        const consoleSpy = jest.spyOn(console, 'log');

        sorter.autoSelectAndSort([5, 3, 1]);

        expect(consoleSpy).toHaveBeenCalledWith(expect.stringContaining('Bubble Sort'));
        consoleSpy.mockRestore();
    });
});

describe('Functional Strategy — Lambda/Arrow', () => {
    test('arrow function strategy কাজ করে', () => {
        const engine = new FunctionalPricingEngine((p) => p * 0.85);
        expect(engine.calculate(1000)).toBe(850);
    });

    test('runtime-এ strategy পরিবর্তন', () => {
        const engine = new FunctionalPricingEngine((p) => p);
        expect(engine.calculate(1000)).toBe(1000);

        engine.setStrategy((p) => p * 0.5);
        expect(engine.calculate(1000)).toBe(500);
    });

    test('strategy composition কাজ করে', () => {
        const pipe = (...fns) => (x) => fns.reduce((v, f) => f(v), x);
        const pricing = pipe(
            (p) => p * 0.9,  // ১০% ছাড়
            (p) => p * 1.15, // ১৫% ভ্যাট
        );
        expect(pricing(1000)).toBeCloseTo(1035);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Strategy ↔ State

```
Strategy: Client বাইরে থেকে অ্যালগরিদম সেট করে
State: Object অভ্যন্তরীণভাবে behavior পাল্টায়
→ দুটোরই structure প্রায় একই, intent ভিন্ন
```

### Strategy ↔ Template Method

```
Strategy: Composition (HAS-A) → পুরো অ্যালগরিদম replace
Template Method: Inheritance (IS-A) → অ্যালগরিদমের step override
→ Strategy বেশি flexible, Template Method বেশি structured
```

### Strategy ↔ Factory Method

```
Factory: কোন strategy তৈরি হবে সেটি নির্ধারণ করে
Strategy: তৈরি হওয়া strategy execute করে
→ প্রায়ই একসাথে ব্যবহৃত হয় (Factory strategy তৈরি করে, Context execute করে)
```

### Strategy ↔ Decorator

```
Decorator: behavior যোগ করে (wrap করে)
Strategy: behavior পাল্টায় (replace করে)
→ Decorator চেইন করে feature যোগ করে, Strategy পুরো logic বদলায়
```

### Strategy ↔ Bridge

```
Bridge: Abstraction ও Implementation আলাদা করে
Strategy: অ্যালগরিদম আলাদা করে
→ Bridge বৃহত্তর পরিসরে কাজ করে, Strategy নির্দিষ্ট behavior-এ
```

```
সম্পর্কের মানচিত্র:

                    ┌──────────────┐
                    │   Factory    │──── তৈরি করে ──→ Strategy
                    └──────────────┘
                           │
      ┌────────────────────┼────────────────────┐
      │                    │                    │
┌─────┴──────┐     ┌──────┴───────┐     ┌──────┴──────┐
│  Strategy  │     │    State     │     │  Template   │
│  (replace) │     │ (transition) │     │  (override) │
└────────────┘     └──────────────┘     └─────────────┘
      │
      │ behavior যোগ/বদল
      │
┌─────┴──────┐     ┌──────────────┐
│ Decorator  │     │   Bridge     │
│  (wrap)    │     │ (separate)   │
└────────────┘     └──────────────┘
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

1. **একই কাজ একাধিক উপায়ে করা যায়** — পেমেন্ট, সর্টিং, compression
2. **Runtime-এ অ্যালগরিদম পরিবর্তনের প্রয়োজন** — ব্যবহারকারী সিলেক্ট করে
3. **বড় conditional block** — `if/else` বা `switch` ৩+ branch হলে
4. **অ্যালগরিদম আলাদাভাবে test করা দরকার** — unit testing সহজ করতে
5. **তৃতীয় পক্ষের integration** — পেমেন্ট গেটওয়ে, SMS provider ইত্যাদি
6. **Framework extension point** — plugin/driver architecture

### ❌ কখন ব্যবহার করবেন না:

1. **মাত্র ১-২টি variant** — সাধারণ `if/else` যথেষ্ট
2. **অ্যালগরিদম কখনো পরিবর্তন হবে না** — অতিরিক্ত abstraction অর্থহীন
3. **Client strategy সম্পর্কে অবগত নয়** — State প্যাটার্ন বেশি উপযুক্ত হতে পারে
4. **অ্যালগরিদমে শুধু ছোট পার্থক্য** — Template Method বিবেচনা করুন
5. **Functional approach সহজ** — lambda/closure দিয়ে সমাধান হলে class তৈরি করবেন না

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট:

```
একই কাজ একাধিক উপায়ে করা যায়?
│
├─ না → Strategy দরকার নেই
│
└─ হ্যাঁ → Runtime-এ পরিবর্তন লাগবে?
           │
           ├─ না → Template Method বা সাধারণ inheritance বিবেচনা করুন
           │
           └─ হ্যাঁ → কতগুলো variant?
                      │
                      ├─ ১-২টি → if/else যথেষ্ট
                      │
                      ├─ ৩-৫টি → Strategy প্যাটার্ন ✅
                      │
                      └─ ৫+ → Strategy + Factory ✅
```

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                    Strategy প্যাটার্ন — সারসংক্ষেপ               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ধরন:      Behavioral                                          │
│  উদ্দেশ্য:  অ্যালগরিদমকে encapsulate করে interchangeable করা   │
│  মূলনীতি:   "Favor composition over inheritance"                │
│                                                                 │
│  তিন অংশ:                                                      │
│  ├─ Strategy Interface → অ্যালগরিদমের contract                 │
│  ├─ Concrete Strategy  → নির্দিষ্ট implementation               │
│  └─ Context            → Strategy ব্যবহার করে, delegate করে    │
│                                                                 │
│  SOLID নীতি যা মানে:                                            │
│  ├─ S: প্রতিটি strategy-র একটি দায়িত্ব                        │
│  ├─ O: নতুন strategy যোগে বিদ্যমান কোড অপরিবর্তিত              │
│  ├─ L: যেকোনো concrete strategy ব্যবহারযোগ্য                  │
│  ├─ I: ছোট, focused interface                                  │
│  └─ D: Context interface-এর উপর নির্ভর করে, concrete-এ নয়     │
│                                                                 │
│  মনে রাখুন:                                                     │
│  • Strategy = "কীভাবে" পাল্টানো                                │
│  • State = "কী অবস্থায়" পাল্টানো                               │
│  • সরল হলে lambda/closure ব্যবহার করুন                         │
│  • জটিল হলে Strategy + Factory একসাথে ব্যবহার করুন             │
│  • বাংলাদেশ কন্টেক্সটে: bKash/Nagad/SSLCommerz payment         │
│    selection এই প্যাটার্নের আদর্শ ব্যবহার                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **"Strategy প্যাটার্ন হলো conditional logic-এর বিরুদ্ধে polymorphism-এর বিজয়।"**
> — আপনার কোডে `if/else`-এর জঙ্গল দেখলেই Strategy প্যাটার্নের কথা ভাবুন! 🎯
