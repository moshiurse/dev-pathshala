# O — Open/Closed Principle (OCP)

> **"সফটওয়্যার এন্টিটি (ক্লাস, মডিউল, ফাংশন) এক্সটেনশনের জন্য খোলা কিন্তু মডিফিকেশনের জন্য বন্ধ হওয়া উচিত।"**
> — Bertrand Meyer

---

## 📖 সংজ্ঞা

Open/Closed Principle বলে:

- **Open for Extension (এক্সটেনশনের জন্য খোলা):** নতুন ফিচার যোগ করা যাবে
- **Closed for Modification (মডিফিকেশনের জন্য বন্ধ):** বিদ্যমান কোড পরিবর্তন করতে হবে না

অর্থাৎ, নতুন ফিচার যোগ করতে গেলে **নতুন কোড লিখবেন**, পুরনো কোড **বদলাবেন না**।

### কেন এটি গুরুত্বপূর্ণ?

প্রতিবার বিদ্যমান কোড পরিবর্তন করলে:
- আগে যা কাজ করতো তা ভেঙে যেতে পারে
- পুরনো টেস্ট ফেল করতে পারে
- অন্য ডেভেলপারের কোডে প্রভাব পড়তে পারে

---

## 🏠 বাস্তব জীবনের উদাহরণ

### বৈদ্যুতিক সকেটের উদাহরণ

একটি বৈদ্যুতিক সকেট চিন্তা করুন:
- সকেট **এক্সটেনশনের জন্য খোলা** — যেকোনো নতুন যন্ত্র (ফ্যান, ল্যাম্প, চার্জার) প্লাগ ইন করা যায়
- সকেট **মডিফিকেশনের জন্য বন্ধ** — নতুন যন্ত্রের জন্য সকেটের ডিজাইন বদলাতে হয় না

যদি প্রতিটি নতুন যন্ত্রের জন্য সকেট পরিবর্তন করতে হতো — সেটা হতো OCP ভঙ্গ!

---

## ❌ খারাপ উদাহরণ — OCP ভঙ্গ

### PHP (❌ ভুল)

```php
<?php

class PaymentProcessor
{
    public function processPayment(string $type, float $amount): string
    {
        // ❌ প্রতিটি নতুন পেমেন্ট মেথডে এই ক্লাস মডিফাই করতে হবে
        if ($type === 'bkash') {
            // বিকাশ পেমেন্ট লজিক
            $fee = $amount * 0.015;
            return "বিকাশ পেমেন্ট সফল। ফি: {$fee}";

        } elseif ($type === 'nagad') {
            // নগদ পেমেন্ট লজিক
            $fee = $amount * 0.012;
            return "নগদ পেমেন্ট সফল। ফি: {$fee}";

        } elseif ($type === 'rocket') {
            // রকেট পেমেন্ট লজিক
            $fee = $amount * 0.018;
            return "রকেট পেমেন্ট সফল। ফি: {$fee}";

        } elseif ($type === 'credit_card') {
            // ক্রেডিট কার্ড লজিক
            $fee = $amount * 0.025;
            return "ক্রেডিট কার্ড পেমেন্ট সফল। ফি: {$fee}";

        }
        // ❌ নতুন পেমেন্ট মেথড যোগ করতে হলে এখানে আরেকটি elseif যোগ করতে হবে!

        throw new InvalidArgumentException("অজানা পেমেন্ট টাইপ: {$type}");
    }
}

// ব্যবহার
$processor = new PaymentProcessor();
echo $processor->processPayment('bkash', 1000);

// ❌ সমস্যা: নতুন "upay" যোগ করতে হলে processPayment() মেথড মডিফাই করতে হবে
// এতে বিদ্যমান bkash, nagad ইত্যাদির কোডেও বাগ আসতে পারে
```

### JavaScript (❌ ভুল)

```javascript
class DiscountCalculator {
    // ❌ প্রতিটি নতুন কাস্টমার টাইপে এই মেথড মডিফাই করতে হবে
    calculate(customerType, amount) {
        switch (customerType) {
            case 'regular':
                return amount * 0.05;
            case 'premium':
                return amount * 0.15;
            case 'vip':
                return amount * 0.25;
            // ❌ নতুন "platinum" কাস্টমার যোগ করতে হলে এই switch-এ নতুন case যোগ করতে হবে
            default:
                return 0;
        }
    }
}

class AreaCalculator {
    // ❌ প্রতিটি নতুন শেপে মডিফাই করতে হবে
    calculateArea(shape) {
        if (shape.type === 'circle') {
            return Math.PI * shape.radius ** 2;
        } else if (shape.type === 'rectangle') {
            return shape.width * shape.height;
        } else if (shape.type === 'triangle') {
            return 0.5 * shape.base * shape.height;
        }
        // ❌ নতুন "pentagon" যোগ করতে হলে এখানে মডিফাই করতে হবে
        throw new Error(`অজানা শেপ: ${shape.type}`);
    }
}
```

---

## ✅ ভালো উদাহরণ — OCP মেনে

### PHP (✅ সঠিক) — Strategy Pattern ব্যবহার করে

```php
<?php

// ধাপ ১: পেমেন্ট গেটওয়ের জন্য একটি ইন্টারফেস/কন্ট্র্যাক্ট তৈরি
interface PaymentGateway
{
    public function getName(): string;
    public function getFeeRate(): float;
    public function process(float $amount): PaymentResult;
}

// ভ্যালু অবজেক্ট — পেমেন্ট রেজাল্ট
class PaymentResult
{
    public function __construct(
        public readonly bool $success,
        public readonly string $gateway,
        public readonly float $amount,
        public readonly float $fee,
        public readonly string $message
    ) {}
}

// ধাপ ২: প্রতিটি পেমেন্ট মেথডের জন্য আলাদা ক্লাস
class BkashPayment implements PaymentGateway
{
    public function getName(): string
    {
        return 'বিকাশ';
    }

    public function getFeeRate(): float
    {
        return 0.015;
    }

    public function process(float $amount): PaymentResult
    {
        $fee = $amount * $this->getFeeRate();
        // বিকাশ API কল করার লজিক এখানে
        return new PaymentResult(
            success: true,
            gateway: $this->getName(),
            amount: $amount,
            fee: $fee,
            message: "{$this->getName()} পেমেন্ট সফল"
        );
    }
}

class NagadPayment implements PaymentGateway
{
    public function getName(): string
    {
        return 'নগদ';
    }

    public function getFeeRate(): float
    {
        return 0.012;
    }

    public function process(float $amount): PaymentResult
    {
        $fee = $amount * $this->getFeeRate();
        return new PaymentResult(
            success: true,
            gateway: $this->getName(),
            amount: $amount,
            fee: $fee,
            message: "{$this->getName()} পেমেন্ট সফল"
        );
    }
}

class CreditCardPayment implements PaymentGateway
{
    public function getName(): string
    {
        return 'ক্রেডিট কার্ড';
    }

    public function getFeeRate(): float
    {
        return 0.025;
    }

    public function process(float $amount): PaymentResult
    {
        $fee = $amount * $this->getFeeRate();
        return new PaymentResult(
            success: true,
            gateway: $this->getName(),
            amount: $amount,
            fee: $fee,
            message: "{$this->getName()} পেমেন্ট সফল"
        );
    }
}

// ধাপ ৩: PaymentProcessor — এটি আর কখনো মডিফাই করতে হবে না!
class PaymentProcessor
{
    public function processPayment(PaymentGateway $gateway, float $amount): PaymentResult
    {
        // ✅ এই মেথড কখনো পরিবর্তন করতে হবে না
        // নতুন পেমেন্ট মেথড যোগ হলে শুধু নতুন ক্লাস তৈরি করলেই হবে
        if ($amount <= 0) {
            return new PaymentResult(
                success: false,
                gateway: $gateway->getName(),
                amount: $amount,
                fee: 0,
                message: "অবৈধ পরিমাণ"
            );
        }

        return $gateway->process($amount);
    }
}

// ✅ ব্যবহার
$processor = new PaymentProcessor();

echo $processor->processPayment(new BkashPayment(), 1000)->message;
echo $processor->processPayment(new NagadPayment(), 2000)->message;
echo $processor->processPayment(new CreditCardPayment(), 5000)->message;

// ✅ নতুন "Upay" যোগ করতে হলে শুধু নতুন ক্লাস তৈরি করুন!
// PaymentProcessor কোডে কোনো পরিবর্তন নেই
class UpayPayment implements PaymentGateway
{
    public function getName(): string { return 'উপায়'; }
    public function getFeeRate(): float { return 0.013; }

    public function process(float $amount): PaymentResult
    {
        $fee = $amount * $this->getFeeRate();
        return new PaymentResult(
            success: true,
            gateway: $this->getName(),
            amount: $amount,
            fee: $fee,
            message: "{$this->getName()} পেমেন্ট সফল"
        );
    }
}

// এটাই কাজ করবে — কোনো পুরনো কোড মডিফাই করতে হয়নি!
echo $processor->processPayment(new UpayPayment(), 3000)->message;
```

### JavaScript (✅ সঠিক)

```javascript
// ধাপ ১: বেস ক্লাস / ইন্টারফেস
class Shape {
    area() {
        throw new Error('area() মেথড ইমপ্লিমেন্ট করতে হবে');
    }

    describe() {
        return `${this.constructor.name}: ক্ষেত্রফল = ${this.area().toFixed(2)}`;
    }
}

// ধাপ ২: প্রতিটি শেপের জন্য আলাদা ক্লাস
class Circle extends Shape {
    #radius;

    constructor(radius) {
        super();
        this.#radius = radius;
    }

    area() {
        return Math.PI * this.#radius ** 2;
    }
}

class Rectangle extends Shape {
    #width;
    #height;

    constructor(width, height) {
        super();
        this.#width = width;
        this.#height = height;
    }

    area() {
        return this.#width * this.#height;
    }
}

class Triangle extends Shape {
    #base;
    #height;

    constructor(base, height) {
        super();
        this.#base = base;
        this.#height = height;
    }

    area() {
        return 0.5 * this.#base * this.#height;
    }
}

// ধাপ ৩: AreaCalculator — এটি আর কখনো মডিফাই করতে হবে না!
class AreaCalculator {
    calculateTotal(shapes) {
        // ✅ যতো শেপই যোগ হোক, এই মেথড একই থাকবে
        return shapes.reduce((total, shape) => total + shape.area(), 0);
    }

    generateReport(shapes) {
        const lines = shapes.map(shape => shape.describe());
        const total = this.calculateTotal(shapes);
        return [...lines, `মোট ক্ষেত্রফল: ${total.toFixed(2)}`].join('\n');
    }
}

// ✅ ব্যবহার
const calculator = new AreaCalculator();
const shapes = [
    new Circle(5),
    new Rectangle(4, 6),
    new Triangle(3, 8)
];

console.log(calculator.generateReport(shapes));

// ✅ নতুন Pentagon যোগ — AreaCalculator তে কোনো পরিবর্তন নেই!
class Pentagon extends Shape {
    #side;

    constructor(side) {
        super();
        this.#side = side;
    }

    area() {
        return (Math.sqrt(5 * (5 + 2 * Math.sqrt(5))) / 4) * this.#side ** 2;
    }
}

shapes.push(new Pentagon(5));
console.log(calculator.generateReport(shapes)); // ✅ কাজ করবে!
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### OCP অর্জনের ৩টি প্রধান কৌশল

#### ১. Polymorphism (পলিমরফিজম) — সবচেয়ে সাধারণ

```php
<?php

// ইন্টারফেস দিয়ে পলিমরফিজম
interface Logger
{
    public function log(string $message): void;
}

class FileLogger implements Logger
{
    public function log(string $message): void
    {
        file_put_contents('app.log', $message . PHP_EOL, FILE_APPEND);
    }
}

class DatabaseLogger implements Logger
{
    public function log(string $message): void
    {
        // DB তে সেভ
    }
}

// নতুন SlackLogger যোগ — কোনো পুরনো কোড বদলানো লাগবে না
class SlackLogger implements Logger
{
    public function log(string $message): void
    {
        // Slack webhook কল
    }
}

class Application
{
    public function __construct(private readonly Logger $logger) {}

    public function doSomething(): void
    {
        // কাজ...
        $this->logger->log("কাজ সম্পন্ন");
    }
}
```

#### ২. Decorator Pattern (ডেকোরেটর প্যাটার্ন)

```php
<?php

interface Notifier
{
    public function send(string $message): void;
}

class EmailNotifier implements Notifier
{
    public function send(string $message): void
    {
        echo "ইমেইল: {$message}\n";
    }
}

// ডেকোরেটর — বিদ্যমান ক্লাস মডিফাই না করে ফিচার যোগ
class SMSNotifierDecorator implements Notifier
{
    public function __construct(private readonly Notifier $wrappee) {}

    public function send(string $message): void
    {
        $this->wrappee->send($message);  // আগের কাজ
        echo "SMS: {$message}\n";        // নতুন ফিচার
    }
}

class SlackNotifierDecorator implements Notifier
{
    public function __construct(private readonly Notifier $wrappee) {}

    public function send(string $message): void
    {
        $this->wrappee->send($message);
        echo "Slack: {$message}\n";
    }
}

// ✅ চেইন করে একাধিক নোটিফিকেশন — কোনো ক্লাস মডিফাই না করে!
$notifier = new SlackNotifierDecorator(
    new SMSNotifierDecorator(
        new EmailNotifier()
    )
);

$notifier->send("সার্ভার ডাউন!");
// আউটপুট:
// ইমেইল: সার্ভার ডাউন!
// SMS: সার্ভার ডাউন!
// Slack: সার্ভার ডাউন!
```

```javascript
// JavaScript এ Decorator
class EmailNotifier {
    send(message) {
        console.log(`ইমেইল: ${message}`);
    }
}

class SMSDecorator {
    #wrappee;

    constructor(notifier) {
        this.#wrappee = notifier;
    }

    send(message) {
        this.#wrappee.send(message);
        console.log(`SMS: ${message}`);
    }
}

class SlackDecorator {
    #wrappee;

    constructor(notifier) {
        this.#wrappee = notifier;
    }

    send(message) {
        this.#wrappee.send(message);
        console.log(`Slack: ${message}`);
    }
}

// ✅ চেইন করে ফিচার যোগ
const notifier = new SlackDecorator(
    new SMSDecorator(
        new EmailNotifier()
    )
);

notifier.send("সার্ভার ডাউন!");
```

#### ৩. Composition ও Configuration

```javascript
// কনফিগারেশন দিয়ে OCP
class ValidationEngine {
    #rules;

    constructor(rules = []) {
        this.#rules = rules;
    }

    addRule(rule) {
        this.#rules.push(rule);
        return this; // chaining
    }

    validate(data) {
        const errors = [];
        for (const rule of this.#rules) {
            const error = rule.check(data);
            if (error) errors.push(error);
        }
        return { valid: errors.length === 0, errors };
    }
}

// প্রতিটি রুল আলাদা ক্লাস — নতুন রুল যোগে ValidationEngine বদলায় না
class RequiredRule {
    #field;
    constructor(field) { this.#field = field; }

    check(data) {
        if (!data[this.#field]) {
            return `${this.#field} আবশ্যক`;
        }
        return null;
    }
}

class MinLengthRule {
    #field;
    #min;
    constructor(field, min) { this.#field = field; this.#min = min; }

    check(data) {
        if (data[this.#field]?.length < this.#min) {
            return `${this.#field} কমপক্ষে ${this.#min} অক্ষর হতে হবে`;
        }
        return null;
    }
}

class EmailFormatRule {
    #field;
    constructor(field) { this.#field = field; }

    check(data) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (data[this.#field] && !emailRegex.test(data[this.#field])) {
            return `${this.#field} সঠিক ইমেইল নয়`;
        }
        return null;
    }
}

// ✅ ব্যবহার — নতুন রুল যোগে ValidationEngine কোড বদলায় না
const validator = new ValidationEngine()
    .addRule(new RequiredRule('name'))
    .addRule(new RequiredRule('email'))
    .addRule(new MinLengthRule('name', 3))
    .addRule(new EmailFormatRule('email'));

console.log(validator.validate({ name: 'রহ', email: 'bad' }));
// { valid: false, errors: ['name কমপক্ষে 3 অক্ষর হতে হবে', 'email সঠিক ইমেইল নয়'] }

// ✅ নতুন রুল — কোনো কিছু মডিফাই না করে
class BanglaOnlyRule {
    #field;
    constructor(field) { this.#field = field; }

    check(data) {
        if (data[this.#field] && !/^[\u0980-\u09FF\s]+$/.test(data[this.#field])) {
            return `${this.#field} শুধুমাত্র বাংলায় হতে হবে`;
        }
        return null;
    }
}

validator.addRule(new BanglaOnlyRule('name'));
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **নিরাপদ এক্সটেনশন** | নতুন ফিচার যোগে পুরনো কোড ভাঙার ঝুঁকি নেই |
| ২ | **কম রিগ্রেশন বাগ** | বিদ্যমান কোড অপরিবর্তিত থাকে |
| ৩ | **সহজ টেস্টিং** | নতুন ক্লাস আলাদাভাবে টেস্ট করা যায় |
| ৪ | **প্লাগ-ইন আর্কিটেকচার** | নতুন ফিচার "প্লাগ ইন" করা যায় |
| ৫ | **প্যারালাল ডেভেলপমেন্ট** | ভিন্ন ডেভেলপার ভিন্ন ক্লাসে কাজ করতে পারে |
| ৬ | **কম কোড রিভিউ** | নতুন ফাইলে শুধু নতুন কোড — পুরনো কোড দেখতে হয় না |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **বেশি abstraction** | ইন্টারফেস, abstract ক্লাস বাড়ে |
| ২ | **শুরুতে সময় বেশি** | সঠিক abstraction ডিজাইন করতে সময় লাগে |
| ৩ | **ভুল abstraction ক্ষতিকর** | ভুল জায়গায় OCP করলে কোড আরো জটিল হয় |
| ৪ | **ওভার-ইঞ্জিনিয়ারিংয়ের ঝুঁকি** | সব জায়গায় OCP করা মানে অপ্রয়োজনীয় জটিলতা |
| ৫ | **নতুনদের জন্য কঠিন** | পলিমরফিজম ও abstraction বুঝতে সময় লাগে |

---

## 🚫 সাধারণ ভুল

### ভুল ১: if/else দিয়ে টাইপ চেক করা

```php
<?php

// ❌ instanceof বা type check = OCP ভঙ্গের লক্ষণ
function getShippingCost(object $order): float
{
    if ($order instanceof DomesticOrder) {
        return 50;
    } elseif ($order instanceof InternationalOrder) {
        return 200;
    }
    // নতুন অর্ডার টাইপে মডিফাই করতে হবে
    return 0;
}

// ✅ পলিমরফিজম ব্যবহার করুন
interface Shippable
{
    public function getShippingCost(): float;
}

class DomesticOrder implements Shippable
{
    public function getShippingCost(): float { return 50; }
}

class InternationalOrder implements Shippable
{
    public function getShippingCost(): float { return 200; }
}
```

### ভুল ২: সময়ের আগেই OCP করা (Premature Abstraction)

```php
<?php

// ❌ শুধু একটা পেমেন্ট মেথড থাকলে OCP এর দরকার নেই
// এটা ওভার-ইঞ্জিনিয়ারিং

interface PaymentGateway { /* ... */ }
class BkashPayment implements PaymentGateway { /* ... */ }
class PaymentProcessor { /* ... */ }

// ✅ প্রথমে সিম্পল রাখুন, যখন দ্বিতীয় ভ্যারিয়েশন আসবে তখন রিফ্যাক্টর করুন
// "Rule of Three" — তিনবার একই প্যাটার্ন দেখলে abstraction করুন
```

---

## 📏 কখন OCP প্রয়োগ করবেন

### ✅ করবেন যখন:

- কোডে **if/else বা switch** দিয়ে টাইপ চেক হচ্ছে
- নতুন ফিচার যোগে **বারবার একই ফাইল** মডিফাই করতে হচ্ছে
- একটি ক্লাসে **একই ধরনের ভ্যারিয়েশন** বাড়ছে (পেমেন্ট মেথড, লগার, নোটিফায়ার)
- **প্লাগিন সিস্টেম** বা **থার্ড-পার্টি ইন্টিগ্রেশন** দরকার
- টিমে **একাধিক ডেভেলপার** একই ফিচারে কাজ করছে

### ❌ করবেন না যখন:

- শুধুমাত্র **একটি ভ্যারিয়েশন** আছে এবং আরো আসার সম্ভাবনা কম
- প্রজেক্ট **খুব ছোট** বা প্রোটোটাইপ
- **YAGNI** — ভবিষ্যতের জন্য আগেভাগে abstraction করবেন না
- বিদ্যমান কোড **স্থিতিশীল** এবং পরিবর্তনের সম্ভাবনা নেই

---

## 🔗 অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক

```
SRP ──► OCP: প্রতিটি ক্লাসের একটি দায়িত্ব থাকলে এক্সটেনশন সহজ
LSP ──► OCP: চাইল্ড ক্লাস সঠিকভাবে কাজ না করলে OCP ভেঙে যায়
ISP ──► OCP: ছোট ইন্টারফেস মানে সহজে এক্সটেন্ড করা যায়
DIP ──► OCP: abstraction এর উপর নির্ভর করলে নতুন implementation যোগ সহজ
```

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **মূল ধারণা** | নতুন কোড লিখুন, পুরনো কোড বদলাবেন না |
| **কৌশল** | Polymorphism, Decorator, Composition |
| **চেক করুন** | "নতুন ফিচারে কি পুরনো ক্লাস মডিফাই করতে হচ্ছে?" |
| **সাবধান** | ভুল বা অকালে abstraction এড়িয়ে চলুন |
| **মনে রাখুন** | "Rule of Three" — তৃতীয়বার ভ্যারিয়েশন দেখলে রিফ্যাক্টর |
