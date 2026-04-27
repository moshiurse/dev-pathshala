# 🦨 কোড স্মেল (Code Smells) — খারাপ কোডের লক্ষণ চেনা

> **"কোড স্মেল হলো সোর্স কোডের সেই বৈশিষ্ট্য যা গভীর সমস্যার ইঙ্গিত দেয়।"**
> — Martin Fowler

---

## 📖 সংজ্ঞা

**কোড স্মেল** হলো কোডের এমন কিছু প্যাটার্ন বা বৈশিষ্ট্য যা সরাসরি বাগ না হলেও, কোডের ডিজাইনে দুর্বলতা নির্দেশ করে। এগুলো কোডকে পড়তে, বুঝতে এবং পরিবর্তন করতে কঠিন করে তোলে।

কোড স্মেল মানে এই না যে কোড ভুল — কিন্তু এটা একটি **সতর্কতা সংকেত** যে কোডটি রিফ্যাক্টরিং প্রয়োজন।

### কোড স্মেল ≠ বাগ

```
কোড স্মেল                          বাগ
──────────                         ─────
কোড কাজ করে ✓                    কোড কাজ করে না ✗
মেইনটেইন করা কঠিন                 ক্র্যাশ/ভুল আউটপুট
ভবিষ্যতে সমস্যা তৈরি করবে         এখনই সমস্যা তৈরি করছে
রিফ্যাক্টরিং দিয়ে সমাধান           বাগ ফিক্স দিয়ে সমাধান
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### বাসার উদাহরণ 🏠

একটা বাসায় চিন্তা করুন:
- **রান্নাঘরে গন্ধ** → হয়তো পচা খাবার ফ্রিজে আছে (Dead Code)
- **একটা ড্রয়ারে সবকিছু** → কাপড়, কাগজ, টুল সব একসাথে (God Class)
- **একই চাবি ১০ জায়গায় কপি** → মূল চাবি হারালে সব বদলাতে হবে (Duplicate Code)
- **যে ঘর কেউ ব্যবহার করে না** → শুধু জায়গা নষ্ট (Lazy Class)
- **সব তার জট পাকানো** → একটা খুলতে গেলে সব নড়ে (Tight Coupling)

```
🏠 বাসা = কোডবেস
┌──────────────────────────────────────────┐
│  🚪 প্রবেশদ্বার (Entry Point)              │
│                                          │
│  📦 স্টোররুম        📦 স্টোররুম            │
│  (অব্যবহৃত জিনিস)   (অব্যবহৃত জিনিস)      │
│  = Dead Code       = Dead Code          │
│                                          │
│  🍳 রান্নাঘর + 🛏️ শোবার ঘর + 🛁 বাথরুম    │
│  (সব একসাথে = God Class)                 │
│                                          │
│  🔌 সব তার জট পাকানো                     │
│  (Tight Coupling)                        │
└──────────────────────────────────────────┘
```

---

## 🗂️ কোড স্মেলের ৫টি প্রধান ক্যাটাগরি

```
                    🦨 কোড স্মেল
                         │
        ┌────────┬───────┼────────┬──────────┐
        ▼        ▼       ▼        ▼          ▼
   ব্লোটারস  OO       চেঞ্জ    ডিসপেন্সে-  কাপলার্স
   (ফোলা)  অ্যাবিউজ  প্রিভেন্ট   বলস
     │        │       │         │          │
     │        │       │         │          │
  বড় মেথড  switch  Divergent  Dead     Feature
  বড় ক্লাস  চেইন   Change    Code     Envy
  লম্বা     Refused  Shotgun  Duplicate  Message
  প্যারা-   Bequest  Surgery  Code      Chains
  মিটার
```

---

# ১. 🎈 ব্লোটারস (Bloaters)

> কোড এতটাই বড় হয়ে যায় যে সেটা নিয়ে কাজ করা কঠিন হয়ে পড়ে।

---

## ১.১ Long Method — অতিরিক্ত বড় মেথড

### সমস্যা কী?

একটি মেথড যখন ৫০+ লাইনের হয়ে যায়, তখন সেটা পড়া, বোঝা এবং ডিবাগ করা কঠিন হয়ে পড়ে।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

class OrderProcessor
{
    // ❌ এই মেথডটি অনেক কিছু করছে — ভ্যালিডেশন, ক্যালকুলেশন, ডাটাবেস, ইমেইল
    public function processOrder(array $orderData): array
    {
        // ভ্যালিডেশন শুরু
        if (empty($orderData['customer_name'])) {
            throw new Exception('কাস্টমারের নাম দরকার');
        }
        if (empty($orderData['email'])) {
            throw new Exception('ইমেইল দরকার');
        }
        if (!filter_var($orderData['email'], FILTER_VALIDATE_EMAIL)) {
            throw new Exception('ভুল ইমেইল ফরম্যাট');
        }
        if (empty($orderData['items']) || !is_array($orderData['items'])) {
            throw new Exception('প্রোডাক্ট তালিকা দরকার');
        }
        foreach ($orderData['items'] as $item) {
            if ($item['quantity'] <= 0) {
                throw new Exception('পরিমাণ শূন্যের বেশি হতে হবে');
            }
            if ($item['price'] < 0) {
                throw new Exception('দাম ঋণাত্মক হতে পারবে না');
            }
        }

        // মোট দাম হিসাব
        $subtotal = 0;
        foreach ($orderData['items'] as $item) {
            $subtotal += $item['price'] * $item['quantity'];
        }

        // ডিসকাউন্ট হিসাব
        $discount = 0;
        if ($subtotal > 10000) {
            $discount = $subtotal * 0.15;
        } elseif ($subtotal > 5000) {
            $discount = $subtotal * 0.10;
        } elseif ($subtotal > 1000) {
            $discount = $subtotal * 0.05;
        }

        // ট্যাক্স হিসাব
        $taxableAmount = $subtotal - $discount;
        $tax = $taxableAmount * 0.15;
        $total = $taxableAmount + $tax;

        // ডাটাবেসে সেভ
        $db = new PDO('mysql:host=localhost;dbname=shop', 'root', '');
        $stmt = $db->prepare("INSERT INTO orders (customer, email, total) VALUES (?, ?, ?)");
        $stmt->execute([$orderData['customer_name'], $orderData['email'], $total]);
        $orderId = $db->lastInsertId();

        // ইমেইল পাঠানো
        $subject = "অর্ডার নিশ্চিতকরণ #" . $orderId;
        $message = "প্রিয় {$orderData['customer_name']},\n";
        $message .= "আপনার অর্ডার সফলভাবে সম্পন্ন হয়েছে।\n";
        $message .= "মোট: ৳{$total}\n";
        mail($orderData['email'], $subject, $message);

        return ['order_id' => $orderId, 'total' => $total];
    }
}
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ একটি ফাংশনে সবকিছু — ভ্যালিডেশন, হিসাব, সেভ, নোটিফিকেশন
function processOrder(orderData) {
    // ভ্যালিডেশন
    if (!orderData.customerName) {
        throw new Error('কাস্টমারের নাম দরকার');
    }
    if (!orderData.email) {
        throw new Error('ইমেইল দরকার');
    }
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(orderData.email)) {
        throw new Error('ভুল ইমেইল ফরম্যাট');
    }
    if (!orderData.items || orderData.items.length === 0) {
        throw new Error('প্রোডাক্ট তালিকা দরকার');
    }
    for (const item of orderData.items) {
        if (item.quantity <= 0) throw new Error('পরিমাণ শূন্যের বেশি হতে হবে');
        if (item.price < 0) throw new Error('দাম ঋণাত্মক হতে পারবে না');
    }

    // মোট দাম হিসাব
    let subtotal = 0;
    for (const item of orderData.items) {
        subtotal += item.price * item.quantity;
    }

    // ডিসকাউন্ট
    let discount = 0;
    if (subtotal > 10000) discount = subtotal * 0.15;
    else if (subtotal > 5000) discount = subtotal * 0.10;
    else if (subtotal > 1000) discount = subtotal * 0.05;

    // ট্যাক্স
    const taxable = subtotal - discount;
    const tax = taxable * 0.15;
    const total = taxable + tax;

    // ডাটাবেসে সেভ
    const db = require('./database');
    const orderId = db.insert('orders', {
        customer: orderData.customerName,
        email: orderData.email,
        total: total
    });

    // ইমেইল পাঠানো
    const mailer = require('./mailer');
    mailer.send(orderData.email, 'অর্ডার নিশ্চিতকরণ', `মোট: ৳${total}`);

    return { orderId, total };
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ প্রতিটি দায়িত্ব আলাদা ছোট মেথডে ভাগ করা হয়েছে
class OrderProcessor
{
    public function __construct(
        private OrderValidator $validator,
        private PriceCalculator $calculator,
        private OrderRepository $repository,
        private NotificationService $notifier
    ) {}

    public function processOrder(array $orderData): array
    {
        // প্রতিটি ধাপ পরিষ্কার ও আলাদা
        $this->validator->validate($orderData);
        $pricing = $this->calculator->calculate($orderData['items']);
        $orderId = $this->repository->save($orderData, $pricing);
        $this->notifier->sendConfirmation($orderData['email'], $orderId, $pricing);

        return ['order_id' => $orderId, 'total' => $pricing->getTotal()];
    }
}

class OrderValidator
{
    public function validate(array $data): void
    {
        $this->validateCustomerInfo($data);
        $this->validateItems($data['items'] ?? []);
    }

    private function validateCustomerInfo(array $data): void
    {
        if (empty($data['customer_name'])) {
            throw new InvalidArgumentException('কাস্টমারের নাম দরকার');
        }
        if (empty($data['email']) || !filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('সঠিক ইমেইল দরকার');
        }
    }

    private function validateItems(array $items): void
    {
        if (empty($items)) {
            throw new InvalidArgumentException('প্রোডাক্ট তালিকা দরকার');
        }
        foreach ($items as $item) {
            if ($item['quantity'] <= 0 || $item['price'] < 0) {
                throw new InvalidArgumentException('পরিমাণ ও দাম সঠিক হতে হবে');
            }
        }
    }
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ছোট, একক-দায়িত্ব ফাংশনে ভাগ করা হয়েছে
class OrderProcessor {
    constructor(validator, calculator, repository, notifier) {
        this.validator = validator;
        this.calculator = calculator;
        this.repository = repository;
        this.notifier = notifier;
    }

    async processOrder(orderData) {
        // প্রতিটি ধাপ স্পষ্ট ও আলাদা
        this.validator.validate(orderData);
        const pricing = this.calculator.calculate(orderData.items);
        const orderId = await this.repository.save(orderData, pricing);
        await this.notifier.sendConfirmation(orderData.email, orderId, pricing);

        return { orderId, total: pricing.total };
    }
}

class OrderValidator {
    validate(data) {
        this.#validateCustomerInfo(data);
        this.#validateItems(data.items ?? []);
    }

    #validateCustomerInfo(data) {
        if (!data.customerName) throw new Error('কাস্টমারের নাম দরকার');
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!data.email || !emailRegex.test(data.email)) {
            throw new Error('সঠিক ইমেইল দরকার');
        }
    }

    #validateItems(items) {
        if (items.length === 0) throw new Error('প্রোডাক্ট তালিকা দরকার');
        items.forEach(item => {
            if (item.quantity <= 0 || item.price < 0) {
                throw new Error('পরিমাণ ও দাম সঠিক হতে হবে');
            }
        });
    }
}
```

---

## ১.২ Large Class / God Class — একটি ক্লাসে সব কিছু

### সমস্যা কী?

একটি ক্লাস যখন অনেকগুলো দায়িত্ব পালন করে — ডাটা ম্যানেজমেন্ট, ভ্যালিডেশন, বিজনেস লজিক, ফরম্যাটিং সব একসাথে।

```
❌ God Class                    ✅ ছোট ক্লাসসমূহ
┌──────────────────┐           ┌─────────┐ ┌─────────┐
│   UserManager    │           │  User   │ │UserRepo │
│ ─────────────    │           │  Model  │ │  sitory │
│ - নাম সেট করা    │    →      └─────────┘ └─────────┘
│ - ইমেইল পাঠানো   │           ┌─────────┐ ┌─────────┐
│ - ডাটাবেস সেভ   │           │  Email  │ │ User    │
│ - পাসওয়ার্ড হ্যাশ│           │ Service │ │Validator│
│ - রিপোর্ট তৈরি  │           └─────────┘ └─────────┘
│ - লগিং          │           ┌─────────┐ ┌─────────┐
│ - ক্যাশিং       │           │ Report  │ │ Logger  │
└──────────────────┘           │ Builder │ │         │
                               └─────────┘ └─────────┘
```

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ God Class — এই ক্লাস সবকিছু করছে
class UserManager
{
    private PDO $db;

    public function __construct()
    {
        $this->db = new PDO('mysql:host=localhost;dbname=app', 'root', '');
    }

    // দায়িত্ব ১: ইউজার তৈরি
    public function createUser(string $name, string $email, string $password): int
    {
        $hashedPassword = password_hash($password, PASSWORD_BCRYPT);
        $stmt = $this->db->prepare("INSERT INTO users (name, email, password) VALUES (?, ?, ?)");
        $stmt->execute([$name, $email, $hashedPassword]);
        return (int) $this->db->lastInsertId();
    }

    // দায়িত্ব ২: ভ্যালিডেশন
    public function validateEmail(string $email): bool
    {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }

    // দায়িত্ব ৩: ইমেইল পাঠানো
    public function sendWelcomeEmail(string $email, string $name): void
    {
        mail($email, 'স্বাগতম!', "প্রিয় {$name}, আমাদের সাইটে স্বাগতম!");
    }

    // দায়িত্ব ৪: রিপোর্ট তৈরি
    public function generateUserReport(): string
    {
        $stmt = $this->db->query("SELECT * FROM users");
        $users = $stmt->fetchAll();
        $report = "মোট ইউজার: " . count($users) . "\n";
        foreach ($users as $user) {
            $report .= "- {$user['name']} ({$user['email']})\n";
        }
        return $report;
    }

    // দায়িত্ব ৫: লগিং
    public function logAction(string $action): void
    {
        file_put_contents('app.log', date('Y-m-d H:i:s') . " - {$action}\n", FILE_APPEND);
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ প্রতিটি দায়িত্ব আলাদা ক্লাসে
class User
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        private string $hashedPassword
    ) {}

    public static function create(string $name, string $email, string $password): self
    {
        return new self($name, $email, password_hash($password, PASSWORD_BCRYPT));
    }
}

class UserRepository
{
    public function __construct(private PDO $db) {}

    public function save(User $user): int
    {
        $stmt = $this->db->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
        $stmt->execute([$user->name, $user->email]);
        return (int) $this->db->lastInsertId();
    }

    public function findAll(): array
    {
        return $this->db->query("SELECT * FROM users")->fetchAll();
    }
}

class EmailService
{
    public function sendWelcome(string $email, string $name): void
    {
        mail($email, 'স্বাগতম!', "প্রিয় {$name}, আমাদের সাইটে স্বাগতম!");
    }
}

class UserReportGenerator
{
    public function __construct(private UserRepository $repo) {}

    public function generate(): string
    {
        $users = $this->repo->findAll();
        $report = "মোট ইউজার: " . count($users) . "\n";
        foreach ($users as $user) {
            $report .= "- {$user['name']} ({$user['email']})\n";
        }
        return $report;
    }
}
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ God Class — সব দায়িত্ব একসাথে
class UserManager {
    constructor() {
        this.db = require('./database');
        this.cache = {};
    }

    createUser(name, email, password) {
        const bcrypt = require('bcrypt');
        const hashed = bcrypt.hashSync(password, 10);
        return this.db.insert('users', { name, email, password: hashed });
    }

    validateEmail(email) {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    }

    sendEmail(to, subject, body) {
        const nodemailer = require('nodemailer');
        // ইমেইল পাঠানোর পুরো লজিক এখানে...
    }

    generateReport() {
        const users = this.db.findAll('users');
        return users.map(u => `${u.name} (${u.email})`).join('\n');
    }

    log(message) {
        const fs = require('fs');
        fs.appendFileSync('app.log', `${new Date().toISOString()} - ${message}\n`);
    }
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ আলাদা দায়িত্ব, আলাদা ক্লাস
class User {
    constructor(name, email, hashedPassword) {
        this.name = name;
        this.email = email;
        this.hashedPassword = hashedPassword;
    }
}

class UserRepository {
    constructor(db) {
        this.db = db;
    }

    save(user) {
        return this.db.insert('users', {
            name: user.name,
            email: user.email
        });
    }

    findAll() {
        return this.db.findAll('users');
    }
}

class EmailService {
    sendWelcome(email, name) {
        // ইমেইল পাঠানোর সুনির্দিষ্ট দায়িত্ব
        console.log(`স্বাগতম ইমেইল পাঠানো হচ্ছে: ${email}`);
    }
}

class UserReportGenerator {
    constructor(repository) {
        this.repository = repository;
    }

    generate() {
        const users = this.repository.findAll();
        return users.map(u => `${u.name} (${u.email})`).join('\n');
    }
}
```

---

## ১.৩ Long Parameter List — ৪+ প্যারামিটার

### সমস্যা কী?

একটি ফাংশনে ৪টির বেশি প্যারামিটার থাকলে সেটা পড়া ও ব্যবহার করা কঠিন হয়ে যায়।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ অনেকগুলো প্যারামিটার — কোনটা কী বোঝা কঠিন
function createProduct(
    string $name,
    string $description,
    float $price,
    float $discount,
    string $category,
    string $brand,
    int $stock,
    float $weight,
    string $color,
    bool $isFeatured
): array {
    // ...
    return [];
}

// ❌ কল করার সময় বোঝা যায় না কোনটা কী
createProduct('ফোন', 'স্মার্টফোন', 15000, 10, 'ইলেক্ট্রনিক্স', 'Samsung', 50, 0.5, 'কালো', true);
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ প্যারামিটার অবজেক্টে গ্রুপ করা হয়েছে
class ProductData
{
    public function __construct(
        public readonly string $name,
        public readonly string $description,
        public readonly PricingInfo $pricing,
        public readonly CategoryInfo $category,
        public readonly InventoryInfo $inventory
    ) {}
}

class PricingInfo
{
    public function __construct(
        public readonly float $price,
        public readonly float $discount = 0
    ) {}
}

class CategoryInfo
{
    public function __construct(
        public readonly string $category,
        public readonly string $brand
    ) {}
}

class InventoryInfo
{
    public function __construct(
        public readonly int $stock,
        public readonly float $weight,
        public readonly string $color
    ) {}
}

// ✅ পরিষ্কার ও বোধগম্য
function createProduct(ProductData $data, bool $isFeatured = false): array
{
    // ...
    return [];
}
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ অনেক প্যারামিটার — কোন পজিশনে কী মনে রাখা কঠিন
function createUser(name, email, age, phone, address, city, country, zipCode, role, isActive) {
    // ...
}

// ❌ কল করার সময় কোনটা কী?
createUser('রহিম', 'rahim@mail.com', 25, '01711111111', 'মিরপুর', 'ঢাকা', 'বাংলাদেশ', '1216', 'admin', true);
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ অবজেক্ট ডিস্ট্রাকচারিং ব্যবহার
function createUser({ name, email, age, contact, address, role = 'user', isActive = true }) {
    // প্রতিটি ফিল্ড নাম দেখেই বোঝা যায়
}

// ✅ পরিষ্কার ও সেলফ-ডকুমেন্টিং
createUser({
    name: 'রহিম',
    email: 'rahim@mail.com',
    age: 25,
    contact: { phone: '01711111111' },
    address: { street: 'মিরপুর', city: 'ঢাকা', country: 'বাংলাদেশ', zip: '1216' },
    role: 'admin'
});
```

---

## ১.৪ Data Clumps — একসাথে ঘুরে বেড়ানো ডাটা গ্রুপ

### সমস্যা কী?

কিছু ডাটা সবসময় একসাথে ব্যবহৃত হয় কিন্তু আলাদা ভ্যারিয়েবল হিসেবে পাস করা হয়।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ latitude, longitude সবসময় একসাথে ব্যবহৃত হয় কিন্তু আলাদা পাস করা হচ্ছে
function calculateDistance(float $lat1, float $lng1, float $lat2, float $lng2): float
{
    // হ্যাভারসাইন ফর্মুলা
    $earthRadius = 6371;
    $dLat = deg2rad($lat2 - $lat1);
    $dLng = deg2rad($lng2 - $lng1);
    $a = sin($dLat / 2) ** 2 + cos(deg2rad($lat1)) * cos(deg2rad($lat2)) * sin($dLng / 2) ** 2;
    return $earthRadius * 2 * atan2(sqrt($a), sqrt(1 - $a));
}

function showOnMap(float $lat, float $lng, string $label): void { /* ... */ }
function saveLocation(float $lat, float $lng, int $userId): void { /* ... */ }
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ একসাথে ঘুরে বেড়ানো ডাটাকে ক্লাসে ক্যাপসুলেট করা হয়েছে
class GeoPoint
{
    public function __construct(
        public readonly float $latitude,
        public readonly float $longitude
    ) {}

    public function distanceTo(GeoPoint $other): float
    {
        $earthRadius = 6371;
        $dLat = deg2rad($other->latitude - $this->latitude);
        $dLng = deg2rad($other->longitude - $this->longitude);
        $a = sin($dLat / 2) ** 2
            + cos(deg2rad($this->latitude)) * cos(deg2rad($other->latitude))
            * sin($dLng / 2) ** 2;
        return $earthRadius * 2 * atan2(sqrt($a), sqrt(1 - $a));
    }
}

function showOnMap(GeoPoint $point, string $label): void { /* ... */ }
function saveLocation(GeoPoint $point, int $userId): void { /* ... */ }
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ startDate, endDate সবসময় একসাথে — কিন্তু আলাদা পাস করা হচ্ছে
function getReport(startDate, endDate, format) { /* ... */ }
function filterEvents(startDate, endDate, category) { /* ... */ }
function calculateDuration(startDate, endDate) { /* ... */ }
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ DateRange ক্লাস দিয়ে গ্রুপ করা হয়েছে
class DateRange {
    constructor(start, end) {
        if (end < start) throw new Error('শেষ তারিখ শুরুর আগে হতে পারে না');
        this.start = start;
        this.end = end;
    }

    getDurationInDays() {
        return Math.ceil((this.end - this.start) / (1000 * 60 * 60 * 24));
    }

    contains(date) {
        return date >= this.start && date <= this.end;
    }
}

function getReport(dateRange, format) { /* ... */ }
function filterEvents(dateRange, category) { /* ... */ }
```

---

## ১.৫ Primitive Obsession — প্রিমিটিভ টাইপ দিয়ে সব কাজ

### সমস্যা কী?

ফোন নম্বর, ইমেইল, টাকার পরিমাণ ইত্যাদি সবকিছু `string` বা `int` দিয়ে রিপ্রেজেন্ট করা — কোনো ভ্যালিডেশন বা আচরণ নেই।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ সব প্রিমিটিভ টাইপ — কোনো ভ্যালিডেশন বা মিনিং নেই
class Order
{
    public function __construct(
        private string $customerPhone,  // ❌ যেকোনো স্ট্রিং গ্রহণযোগ্য
        private string $currency,       // ❌ "abc" হতে পারে
        private float $amount,          // ❌ ঋণাত্মক হতে পারে
        private string $email           // ❌ ভুল ফরম্যাট হতে পারে
    ) {}
}

// ❌ ভুল ডাটা দিয়ে অবজেক্ট তৈরি সম্ভব
$order = new Order('না-ফোন', 'xyz', -500, 'ভুল-ইমেইল');
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ ভ্যালু অবজেক্ট দিয়ে টাইপ সেফটি নিশ্চিত করা হয়েছে
class PhoneNumber
{
    private string $number;

    public function __construct(string $number)
    {
        if (!preg_match('/^01[3-9]\d{8}$/', $number)) {
            throw new InvalidArgumentException('ভুল ফোন নম্বর ফরম্যাট');
        }
        $this->number = $number;
    }

    public function __toString(): string
    {
        return $this->number;
    }
}

class Money
{
    public function __construct(
        private readonly float $amount,
        private readonly Currency $currency
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        }
    }

    public function add(Money $other): self
    {
        if (!$this->currency->equals($other->currency)) {
            throw new RuntimeException('ভিন্ন মুদ্রায় যোগ করা যাবে না');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }
}

class Email
{
    private string $address;

    public function __construct(string $address)
    {
        if (!filter_var($address, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('ভুল ইমেইল ফরম্যাট');
        }
        $this->address = $address;
    }
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ভ্যালু অবজেক্ট দিয়ে প্রিমিটিভকে অর্থবহ করা হয়েছে
class PhoneNumber {
    #number;

    constructor(number) {
        if (!/^01[3-9]\d{8}$/.test(number)) {
            throw new Error('ভুল ফোন নম্বর ফরম্যাট');
        }
        this.#number = number;
    }

    toString() {
        return this.#number;
    }
}

class Money {
    #amount;
    #currency;

    constructor(amount, currency) {
        if (amount < 0) throw new Error('টাকার পরিমাণ ঋণাত্মক হতে পারে না');
        this.#amount = amount;
        this.#currency = currency;
    }

    add(other) {
        if (this.#currency !== other.#currency) {
            throw new Error('ভিন্ন মুদ্রায় যোগ করা যাবে না');
        }
        return new Money(this.#amount + other.#amount, this.#currency);
    }

    get amount() { return this.#amount; }
    get currency() { return this.#currency; }
}
```

---

# ২. 🔀 অবজেক্ট-ওরিয়েন্টেড অ্যাবিউজার্স (OO Abusers)

> অবজেক্ট-ওরিয়েন্টেড প্রিন্সিপলের ভুল বা অসম্পূর্ণ প্রয়োগ।

---

## ২.১ Switch Statements — বারবার switch/if-else চেইন

### সমস্যা কী?

একই switch/if-else ব্লক কোডের বিভিন্ন জায়গায় পুনরাবৃত্তি হয়। নতুন কেস যোগ করতে গেলে সবগুলো জায়গায় পরিবর্তন করতে হয়।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ একই switch বিভিন্ন জায়গায়
class PaymentProcessor
{
    public function calculateFee(string $type, float $amount): float
    {
        switch ($type) {
            case 'credit_card':
                return $amount * 0.03;
            case 'bkash':
                return $amount * 0.015;
            case 'bank_transfer':
                return $amount * 0.01;
            default:
                throw new Exception('অজানা পেমেন্ট টাইপ');
        }
    }

    // ❌ আবার একই switch অন্য মেথডে
    public function processPayment(string $type, float $amount): bool
    {
        switch ($type) {
            case 'credit_card':
                return $this->chargeCreditCard($amount);
            case 'bkash':
                return $this->processBkash($amount);
            case 'bank_transfer':
                return $this->processBankTransfer($amount);
            default:
                throw new Exception('অজানা পেমেন্ট টাইপ');
        }
    }

    // ❌ আবার একই switch
    public function getPaymentLabel(string $type): string
    {
        switch ($type) {
            case 'credit_card': return 'ক্রেডিট কার্ড';
            case 'bkash': return 'বিকাশ';
            case 'bank_transfer': return 'ব্যাংক ট্রান্সফার';
            default: return 'অজানা';
        }
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ পলিমরফিজম দিয়ে switch সরানো হয়েছে
interface PaymentMethod
{
    public function calculateFee(float $amount): float;
    public function process(float $amount): bool;
    public function getLabel(): string;
}

class CreditCardPayment implements PaymentMethod
{
    public function calculateFee(float $amount): float { return $amount * 0.03; }
    public function process(float $amount): bool { /* ক্রেডিট কার্ড প্রসেসিং */ return true; }
    public function getLabel(): string { return 'ক্রেডিট কার্ড'; }
}

class BkashPayment implements PaymentMethod
{
    public function calculateFee(float $amount): float { return $amount * 0.015; }
    public function process(float $amount): bool { /* বিকাশ প্রসেসিং */ return true; }
    public function getLabel(): string { return 'বিকাশ'; }
}

class BankTransferPayment implements PaymentMethod
{
    public function calculateFee(float $amount): float { return $amount * 0.01; }
    public function process(float $amount): bool { /* ব্যাংক ট্রান্সফার */ return true; }
    public function getLabel(): string { return 'ব্যাংক ট্রান্সফার'; }
}

// ✅ নতুন পেমেন্ট টাইপ যোগ করতে শুধু নতুন ক্লাস তৈরি করতে হবে
class PaymentProcessor
{
    public function process(PaymentMethod $method, float $amount): bool
    {
        $fee = $method->calculateFee($amount);
        return $method->process($amount + $fee);
    }
}
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ বারবার একই if-else চেইন
function getShippingCost(type, weight) {
    if (type === 'express') return weight * 50;
    if (type === 'standard') return weight * 20;
    if (type === 'economy') return weight * 10;
    throw new Error('অজানা শিপিং টাইপ');
}

function getDeliveryDays(type) {
    if (type === 'express') return 1;
    if (type === 'standard') return 3;
    if (type === 'economy') return 7;
    throw new Error('অজানা শিপিং টাইপ');
}

function getTrackingEnabled(type) {
    if (type === 'express') return true;
    if (type === 'standard') return true;
    if (type === 'economy') return false;
    return false;
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ স্ট্র্যাটেজি প্যাটার্ন দিয়ে সমাধান
class ShippingStrategy {
    getCost(weight) { throw new Error('ইমপ্লিমেন্ট করতে হবে'); }
    getDeliveryDays() { throw new Error('ইমপ্লিমেন্ট করতে হবে'); }
    isTrackingEnabled() { return false; }
}

class ExpressShipping extends ShippingStrategy {
    getCost(weight) { return weight * 50; }
    getDeliveryDays() { return 1; }
    isTrackingEnabled() { return true; }
}

class StandardShipping extends ShippingStrategy {
    getCost(weight) { return weight * 20; }
    getDeliveryDays() { return 3; }
    isTrackingEnabled() { return true; }
}

class EconomyShipping extends ShippingStrategy {
    getCost(weight) { return weight * 10; }
    getDeliveryDays() { return 7; }
    isTrackingEnabled() { return false; }
}

// ✅ ফ্যাক্টরি দিয়ে তৈরি
const shippingFactory = {
    express: () => new ExpressShipping(),
    standard: () => new StandardShipping(),
    economy: () => new EconomyShipping()
};
```

---

## ২.২ Parallel Inheritance Hierarchies

### সমস্যা কী?

একটি ক্লাসের সাবক্লাস তৈরি করলে অন্য ক্লাসেরও সাবক্লাস তৈরি করতে হয়।

```
❌ সমান্তরাল ইনহেরিটেন্স
Employee              EmployeeReport
   │                       │
   ├── Engineer         ├── EngineerReport
   ├── Designer         ├── DesignerReport
   └── Manager          └── ManagerReport

নতুন যোগ করলে দুইটা ক্লাসই বানাতে হয়!
```

### ✅ সমাধান — কম্পোজিশন ব্যবহার

```php
<?php

// ✅ কম্পোজিশন দিয়ে সমাধান — রিপোর্ট জেনারেটর সবার জন্য কাজ করে
interface ReportDataProvider
{
    public function getReportData(): array;
}

class Employee implements ReportDataProvider
{
    public function __construct(
        private string $name,
        private string $role,
        private float $salary
    ) {}

    public function getReportData(): array
    {
        return [
            'নাম' => $this->name,
            'পদবি' => $this->role,
            'বেতন' => $this->salary
        ];
    }
}

class ReportGenerator
{
    // ✅ যেকোনো ReportDataProvider এর জন্য কাজ করে
    public function generate(ReportDataProvider $provider): string
    {
        $data = $provider->getReportData();
        return implode(', ', array_map(
            fn($key, $value) => "{$key}: {$value}",
            array_keys($data),
            array_values($data)
        ));
    }
}
```

---

## ২.৩ Refused Bequest — ইনহেরিটেন্সের অপব্যবহার

### সমস্যা কী?

চাইল্ড ক্লাস প্যারেন্ট ক্লাসের বেশিরভাগ মেথড/প্রপার্টি ব্যবহার করে না বা ওভাররাইড করে ফেলে।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ Stack, ArrayList এক্সটেন্ড করলে সব মেথড পায় যা Stack এর জন্য অর্থহীন
class Stack extends ArrayList
{
    // ❌ push/pop ছাড়া ArrayList এর অন্য সব মেথড (insert, sort, etc.) এখন Stack এ আছে
    public function push($item): void
    {
        $this->add($item);
    }

    public function pop()
    {
        return $this->removeLast();
    }

    // ❌ এই মেথডগুলো Stack এ থাকা উচিত না কিন্তু ইনহেরিটেন্সের কারণে আছে
    // insert(), sort(), get(index), etc.
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ কম্পোজিশন ব্যবহার — শুধু যা দরকার তাই এক্সপোজ করা
class Stack
{
    private array $items = [];

    public function push($item): void
    {
        $this->items[] = $item;
    }

    public function pop()
    {
        if (empty($this->items)) {
            throw new UnderflowException('স্ট্যাক খালি');
        }
        return array_pop($this->items);
    }

    public function peek()
    {
        if (empty($this->items)) {
            throw new UnderflowException('স্ট্যাক খালি');
        }
        return end($this->items);
    }

    public function isEmpty(): bool
    {
        return empty($this->items);
    }
}
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ পেঙ্গুইন Bird থেকে fly() পেয়েছে — কিন্তু পেঙ্গুইন উড়তে পারে না!
class Bird {
    fly() { return 'উড়ছে...'; }
    eat() { return 'খাচ্ছে...'; }
}

class Penguin extends Bird {
    // ❌ Refused Bequest — fly() অর্থহীন
    fly() { throw new Error('পেঙ্গুইন উড়তে পারে না!'); }
    swim() { return 'সাঁতার কাটছে...'; }
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ইন্টারফেস সেগ্রিগেশন — প্রতিটি ক্ষমতা আলাদা
class Animal {
    eat() { return 'খাচ্ছে...'; }
}

// মিক্সিন দিয়ে শুধু প্রয়োজনীয় ক্ষমতা যোগ করা
const Flyable = (Base) => class extends Base {
    fly() { return 'উড়ছে...'; }
};

const Swimmable = (Base) => class extends Base {
    swim() { return 'সাঁতার কাটছে...'; }
};

class Eagle extends Flyable(Animal) {}       // ✅ উড়তে পারে
class Penguin extends Swimmable(Animal) {}   // ✅ সাঁতার কাটতে পারে, fly() নেই
class Duck extends Swimmable(Flyable(Animal)) {} // ✅ দুটোই পারে
```

---

## ২.৪ Alternative Classes with Different Interfaces

### সমস্যা কী?

দুটি ক্লাস একই কাজ করে কিন্তু ভিন্ন মেথড নাম ব্যবহার করে — ইন্টারচেঞ্জ করা যায় না।

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ একই কাজ কিন্তু ভিন্ন ইন্টারফেস
class EmailNotifier {
    sendEmail(to, message) { /* ... */ }
}

class SMSNotifier {
    fireSMS(phoneNumber, text) { /* ... */ }  // ❌ ভিন্ন মেথড নাম
}

class PushNotifier {
    dispatchPush(deviceId, payload) { /* ... */ }  // ❌ ভিন্ন মেথড নাম
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ সবার একই ইন্টারফেস
class EmailNotifier {
    send(recipient, message) {
        // ইমেইল পাঠানো
    }
}

class SMSNotifier {
    send(recipient, message) {
        // SMS পাঠানো
    }
}

class PushNotifier {
    send(recipient, message) {
        // পুশ নোটিফিকেশন পাঠানো
    }
}

// ✅ এখন সবাই ইন্টারচেঞ্জেবল
function notify(notifier, recipient, message) {
    notifier.send(recipient, message);
}
```

---

# ৩. 🔒 চেঞ্জ প্রিভেন্টার্স (Change Preventers)

> এই স্মেলগুলো কোডে পরিবর্তন করা কঠিন করে তোলে।

---

## ৩.১ Divergent Change — একটি ক্লাস বিভিন্ন কারণে বদলায়

### সমস্যা কী?

একটি ক্লাস **বিভিন্ন কারণে** পরিবর্তন করতে হয় — ডাটাবেস স্কিমা বদলালেও এটা, UI বদলালেও এটা।

```
Divergent Change:
                    ┌──── ডাটাবেস বদলালে
                    │
UserService ◄───────┼──── UI বদলালে
                    │
                    ├──── ভ্যালিডেশন নিয়ম বদলালে
                    │
                    └──── রিপোর্ট ফরম্যাট বদলালে

একটি ক্লাস, অনেক পরিবর্তনের কারণ!
```

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ একটি ক্লাসে অনেক পরিবর্তনের কারণ
class UserService
{
    // কারণ ১: ডাটাবেস স্কিমা বদলালে এটা বদলাতে হবে
    public function saveUser(array $data): void
    {
        $db = new PDO('mysql:host=localhost;dbname=app', 'root', '');
        $stmt = $db->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
        $stmt->execute([$data['name'], $data['email']]);
    }

    // কারণ ২: ভ্যালিডেশন নিয়ম বদলালে এটা বদলাতে হবে
    public function validateUser(array $data): bool
    {
        return !empty($data['name']) && filter_var($data['email'], FILTER_VALIDATE_EMAIL);
    }

    // কারণ ৩: ইমেইল প্রোভাইডার বদলালে এটা বদলাতে হবে
    public function sendNotification(string $email, string $message): void
    {
        mail($email, 'বিজ্ঞপ্তি', $message);
    }

    // কারণ ৪: রিপোর্ট ফরম্যাট বদলালে এটা বদলাতে হবে
    public function generateReport(): string
    {
        return "ইউজার রিপোর্ট...";
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ প্রতিটি কারণ আলাদা ক্লাসে
class UserRepository
{
    public function __construct(private PDO $db) {}
    public function save(array $data): void { /* ডাটাবেস সংক্রান্ত */ }
}

class UserValidator
{
    public function validate(array $data): bool { /* ভ্যালিডেশন সংক্রান্ত */ }
}

class NotificationService
{
    public function send(string $email, string $message): void { /* নোটিফিকেশন সংক্রান্ত */ }
}

class UserReportService
{
    public function generate(): string { /* রিপোর্ট সংক্রান্ত */ }
}
```

---

## ৩.২ Shotgun Surgery — একটি পরিবর্তনে অনেক ক্লাস বদলাতে হয়

### সমস্যা কী?

একটি ছোট পরিবর্তনের জন্য অনেকগুলো ক্লাস/ফাইলে হাত দিতে হয়।

```
Shotgun Surgery (Divergent Change এর বিপরীত):

"কাস্টমারের ফোন নম্বর ফরম্যাট বদলাতে হবে"
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
CustomerForm  CustomerAPI  CustomerReport
     ▼            ▼            ▼
CustomerDB    Invoice      Analytics

একটি পরিবর্তন, অনেক ক্লাস বদলাতে হয়!
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ ফোন নম্বর ফরম্যাটিং বিভিন্ন জায়গায় ছড়িয়ে আছে
class CustomerForm {
    formatPhone(phone) {
        return phone.replace(/(\d{3})(\d{4})(\d{4})/, '$1-$2-$3');
    }
}

class CustomerAPI {
    formatPhone(phone) {
        return phone.replace(/(\d{3})(\d{4})(\d{4})/, '$1-$2-$3');
    }
}

class Invoice {
    formatPhone(phone) {
        return phone.replace(/(\d{3})(\d{4})(\d{4})/, '$1-$2-$3');
    }
}

// ❌ ফরম্যাট বদলাতে হলে ৩ জায়গায় বদলাতে হবে!
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ফোন ফরম্যাটিং একটি জায়গায় কেন্দ্রীভূত
class PhoneFormatter {
    static format(phone) {
        return phone.replace(/(\d{3})(\d{4})(\d{4})/, '$1-$2-$3');
    }
}

class CustomerForm {
    displayPhone(phone) {
        return PhoneFormatter.format(phone);  // ✅ এক জায়গা থেকে ব্যবহার
    }
}

class Invoice {
    printPhone(phone) {
        return PhoneFormatter.format(phone);  // ✅ ফরম্যাট বদলালে শুধু PhoneFormatter বদলাতে হবে
    }
}
```

---

## ৩.৩ Feature Envy — অন্য ক্লাসের ডাটা বেশি ব্যবহার

### সমস্যা কী?

একটি মেথড নিজের ক্লাসের চেয়ে অন্য ক্লাসের ডাটা/মেথড বেশি ব্যবহার করে।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

class Customer
{
    public string $name;
    public string $city;
    public string $street;
    public string $zipCode;
    public string $country;
}

// ❌ এই মেথড Customer এর ডাটা বেশি ব্যবহার করছে — এটা Customer এ থাকা উচিত
class InvoicePrinter
{
    public function printAddress(Customer $customer): string
    {
        return $customer->name . "\n"
            . $customer->street . "\n"
            . $customer->city . ", " . $customer->zipCode . "\n"
            . $customer->country;
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

class Customer
{
    public function __construct(
        public readonly string $name,
        private Address $address
    ) {}

    // ✅ ঠিকানা ফরম্যাটিং Customer এর নিজের দায়িত্ব
    public function getFormattedAddress(): string
    {
        return $this->name . "\n" . $this->address->format();
    }
}

class Address
{
    public function __construct(
        private string $street,
        private string $city,
        private string $zipCode,
        private string $country
    ) {}

    public function format(): string
    {
        return "{$this->street}\n{$this->city}, {$this->zipCode}\n{$this->country}";
    }
}

class InvoicePrinter
{
    public function printAddress(Customer $customer): string
    {
        return $customer->getFormattedAddress();  // ✅ শুধু একটি মেথড কল
    }
}
```

---

# ৪. 🗑️ ডিসপেন্সেবলস (Dispensables)

> অপ্রয়োজনীয় কোড যা সরিয়ে ফেললে কোড আরও ভালো হয়।

---

## ৪.১ Dead Code — অব্যবহৃত কোড

### সমস্যা কী?

এমন কোড যা কখনো এক্সিকিউট হয় না — অব্যবহৃত ভ্যারিয়েবল, মেথড, ক্লাস, ইমপোর্ট।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

class CartService
{
    // ❌ এই মেথড কোথাও কল হয় না
    public function calculateOldDiscount(float $total): float
    {
        return $total * 0.1;
    }

    // ❌ এই ভ্যারিয়েবল ব্যবহৃত হয় না
    private string $legacyApiKey = 'old-key-123';

    // ❌ কমেন্ট আউট কোড — ডিলিট করুন, Git এ আছে
    // public function processOldPayment() {
    //     // পুরনো পেমেন্ট সিস্টেম
    //     $gateway = new OldGateway();
    //     $gateway->charge($amount);
    // }

    public function calculateTotal(array $items): float
    {
        $total = 0;
        // ❌ $debug ভ্যারিয়েবল সেট করা হয়েছে কিন্তু কোথাও ব্যবহৃত হয়নি
        $debug = true;
        foreach ($items as $item) {
            $total += $item['price'] * $item['quantity'];
        }
        return $total;
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ শুধু প্রয়োজনীয় কোড রাখা হয়েছে
class CartService
{
    public function calculateTotal(array $items): float
    {
        $total = 0;
        foreach ($items as $item) {
            $total += $item['price'] * $item['quantity'];
        }
        return $total;
    }
}
```

---

## ৪.২ Duplicate Code — কপি-পেস্ট কোড

### সমস্যা কী?

একই লজিক বিভিন্ন জায়গায় কপি-পেস্ট করা — একটা বদলালে সবগুলো বদলাতে হয়।

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ একই ভ্যালিডেশন লজিক তিন জায়গায়
class UserRegistration {
    register(data) {
        // ❌ ইমেইল ভ্যালিডেশন কপি
        if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
            throw new Error('ভুল ইমেইল');
        }
        // রেজিস্ট্রেশন লজিক...
    }
}

class ProfileUpdate {
    update(data) {
        // ❌ একই ইমেইল ভ্যালিডেশন আবার কপি
        if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
            throw new Error('ভুল ইমেইল');
        }
        // আপডেট লজিক...
    }
}

class NewsletterSubscription {
    subscribe(data) {
        // ❌ আবার একই কোড!
        if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
            throw new Error('ভুল ইমেইল');
        }
        // সাবস্ক্রিপশন লজিক...
    }
}
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ভ্যালিডেশন লজিক এক জায়গায়
class EmailValidator {
    static validate(email) {
        if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
            throw new Error('ভুল ইমেইল');
        }
    }
}

class UserRegistration {
    register(data) {
        EmailValidator.validate(data.email);  // ✅ একই সোর্স
        // রেজিস্ট্রেশন লজিক...
    }
}

class ProfileUpdate {
    update(data) {
        EmailValidator.validate(data.email);  // ✅ একই সোর্স
        // আপডেট লজিক...
    }
}
```

---

## ৪.৩ Speculative Generality — ভবিষ্যতের জন্য অতিরিক্ত কোড

### সমস্যা কী?

"ভবিষ্যতে লাগতে পারে" ভেবে অতিরিক্ত অ্যাবস্ট্রাকশন, ইন্টারফেস, প্যারামিটার যোগ করা।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ শুধু একটি ইমপ্লিমেন্টেশন আছে কিন্তু অনেক লেয়ার তৈরি করা হয়েছে "ভবিষ্যতের জন্য"
interface MessageSenderInterface {}
interface EmailSenderInterface extends MessageSenderInterface {}
abstract class AbstractEmailSender implements EmailSenderInterface {}
abstract class AbstractSmtpEmailSender extends AbstractEmailSender {}

class SmtpEmailSender extends AbstractSmtpEmailSender
{
    // ❌ শেষমেশ একটাই ক্লাস আসলে কাজ করে!
    public function send(string $to, string $message): void
    {
        mail($to, 'বিজ্ঞপ্তি', $message);
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

// ✅ YAGNI — যতটুকু দরকার ততটুকুই
class EmailSender
{
    public function send(string $to, string $message): void
    {
        mail($to, 'বিজ্ঞপ্তি', $message);
    }
}

// ভবিষ্যতে দরকার হলে তখন ইন্টারফেস এক্সট্র্যাক্ট করা যাবে
```

---

## ৪.৪ Lazy Class — খুব ছোট অপ্রয়োজনীয় ক্লাস

### সমস্যা কী?

এমন ক্লাস যা প্রায় কিছুই করে না — তার অস্তিত্ব ন্যায্য না।

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ এই ক্লাস শুধু একটি প্রপার্টি রাখছে — কোনো আচরণ নেই
class UserName {
    constructor(name) {
        this.name = name;
    }

    getName() {
        return this.name;
    }
}

// ❌ এই ক্লাসও শুধু একটি মেথড wrap করছে
class StringHelper {
    capitalize(str) {
        return str.charAt(0).toUpperCase() + str.slice(1);
    }
}
```

### ✅ সমাধান

```javascript
// ✅ সরাসরি ব্যবহার করুন — অতিরিক্ত ক্লাস দরকার নেই
const userName = 'রহিম';
const capitalize = (str) => str.charAt(0).toUpperCase() + str.slice(1);

// তবে, যদি ক্লাসে ভ্যালিডেশন/আচরণ থাকে তাহলে রাখা যায়:
class UserName {
    #name;
    constructor(name) {
        if (!name || name.length < 2) throw new Error('নাম কমপক্ষে ২ অক্ষরের হতে হবে');
        if (name.length > 100) throw new Error('নাম ১০০ অক্ষরের বেশি হতে পারে না');
        this.#name = name.trim();
    }

    get value() { return this.#name; }
    get initials() { return this.#name.split(' ').map(n => n[0]).join(''); }
}
```

---

# ৫. 🔗 কাপলার্স (Couplers)

> ক্লাসগুলোর মধ্যে অতিরিক্ত নির্ভরতা তৈরি করে।

---

## ৫.১ Inappropriate Intimacy — ক্লাসগুলোর মধ্যে অতিরিক্ত নির্ভরতা

### সমস্যা কী?

দুটি ক্লাস একে অপরের internal details জানে এবং সরাসরি ব্যবহার করে।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

class Order
{
    public array $items = [];
    public float $tax = 0;
    public float $discount = 0;
    public string $status = 'pending';
}

// ❌ Invoice সরাসরি Order এর internal ফিল্ডে হাত দিচ্ছে
class Invoice
{
    public function generate(Order $order): string
    {
        $subtotal = 0;
        // ❌ Order এর internal items অ্যারে সরাসরি অ্যাক্সেস
        foreach ($order->items as $item) {
            $subtotal += $item['price'] * $item['qty'];
        }

        // ❌ Order এর internal ফিল্ড সরাসরি ব্যবহার
        $total = $subtotal - $order->discount + $order->tax;

        // ❌ Order এর status সরাসরি পরিবর্তন!
        $order->status = 'invoiced';

        return "মোট: ৳{$total}";
    }
}
```

### ✅ ভালো উদাহরণ — PHP

```php
<?php

class Order
{
    private array $items = [];
    private float $discount = 0;
    private string $status = 'pending';

    // ✅ Order নিজে তার ডাটা এক্সপোজ করে পরিষ্কার API দিয়ে
    public function getSubtotal(): float
    {
        return array_reduce($this->items, fn($sum, $item) =>
            $sum + $item['price'] * $item['qty'], 0);
    }

    public function getTotal(): float
    {
        return $this->getSubtotal() - $this->discount + $this->calculateTax();
    }

    public function markAsInvoiced(): void
    {
        $this->status = 'invoiced';
    }

    private function calculateTax(): float
    {
        return $this->getSubtotal() * 0.15;
    }
}

class Invoice
{
    // ✅ শুধু পাবলিক API ব্যবহার করে
    public function generate(Order $order): string
    {
        $total = $order->getTotal();
        $order->markAsInvoiced();
        return "মোট: ৳{$total}";
    }
}
```

---

## ৫.২ Message Chains — a.getB().getC().getD()

### সমস্যা কী?

একটি অবজেক্ট থেকে অন্য অবজেক্ট, তারপর আরেকটি — লম্বা চেইন তৈরি হয়। মাঝখানের কোনো ক্লাস বদলালে পুরো চেইন ভেঙে যায়।

```
❌ Message Chain:
client → orderService.getOrder(id).getCustomer().getAddress().getCity()

চেইনের প্রতিটি লিংকে নির্ভরতা তৈরি হয়:
Client → Order → Customer → Address → City
```

### ❌ খারাপ উদাহরণ — JavaScript

```javascript
// ❌ লম্বা মেসেজ চেইন
const city = company
    .getDepartment('engineering')
    .getManager()
    .getAddress()
    .getCity();

// ❌ মাঝখানে কোনো ক্লাস বদলালে সব ভেঙে যায়
const managerPhone = order
    .getCustomer()
    .getProfile()
    .getContactInfo()
    .getPhoneNumber()
    .format();
```

### ✅ ভালো উদাহরণ — JavaScript

```javascript
// ✅ ডেলিগেট মেথড তৈরি করা — Law of Demeter অনুসরণ
class Company {
    getEngineeringManagerCity() {
        // চেইনটি ক্লাসের ভেতরে লুকানো
        return this.getDepartment('engineering')?.getManager()?.getCity() ?? 'অজানা';
    }
}

class Order {
    getCustomerPhone() {
        return this.customer.getFormattedPhone();
    }
}

// ✅ ক্লায়েন্ট কোড পরিষ্কার
const city = company.getEngineeringManagerCity();
const phone = order.getCustomerPhone();
```

---

## ৫.৩ Middle Man — শুধু ডেলিগেট করা ক্লাস

### সমস্যা কী?

একটি ক্লাসের বেশিরভাগ মেথড শুধু অন্য ক্লাসের মেথড কল করে — নিজে কিছুই করে না।

### ❌ খারাপ উদাহরণ — PHP

```php
<?php

// ❌ এই ক্লাস শুধু ডেলিগেট করছে — কোনো নিজস্ব লজিক নেই
class CustomerManager
{
    private CustomerRepository $repo;

    public function __construct(CustomerRepository $repo)
    {
        $this->repo = $repo;
    }

    public function find(int $id): ?Customer
    {
        return $this->repo->find($id);  // ❌ শুধু পাস-থ্রু
    }

    public function save(Customer $customer): void
    {
        $this->repo->save($customer);  // ❌ শুধু পাস-থ্রু
    }

    public function delete(int $id): void
    {
        $this->repo->delete($id);  // ❌ শুধু পাস-থ্রু
    }

    public function findAll(): array
    {
        return $this->repo->findAll();  // ❌ শুধু পাস-থ্রু
    }
}
```

### ✅ সমাধান

```php
<?php

// ✅ Middle Man সরিয়ে সরাসরি Repository ব্যবহার করুন
// অথবা Middle Man এ নিজস্ব লজিক যোগ করুন

class CustomerService
{
    public function __construct(
        private CustomerRepository $repo,
        private EventDispatcher $events,
        private CacheService $cache
    ) {}

    // ✅ এখন নিজস্ব লজিক আছে — ক্যাশিং + ইভেন্ট
    public function find(int $id): ?Customer
    {
        $cached = $this->cache->get("customer:{$id}");
        if ($cached) return $cached;

        $customer = $this->repo->find($id);
        if ($customer) {
            $this->cache->set("customer:{$id}", $customer, 3600);
        }
        return $customer;
    }

    public function save(Customer $customer): void
    {
        $this->repo->save($customer);
        $this->cache->invalidate("customer:{$customer->getId()}");
        $this->events->dispatch(new CustomerUpdatedEvent($customer));
    }
}
```

---

# 🏪 বাস্তব প্রজেক্ট উদাহরণ — ই-কমার্স অর্ডার সিস্টেম

> একটি বাস্তব সিস্টেমে একাধিক কোড স্মেল চিহ্নিত করা ও রিফ্যাক্টর করা।

---

## ❌ স্মেলযুক্ত কোড — PHP

```php
<?php

// 🦨 এই ক্লাসে যত স্মেল আছে:
// - God Class (সব এক জায়গায়)
// - Long Method (processOrder অনেক বড়)
// - Long Parameter List
// - Feature Envy
// - Primitive Obsession
// - Duplicate Code
// - Dead Code
class ECommerceSystem
{
    private $db;

    public function __construct()
    {
        $this->db = new PDO('mysql:host=localhost;dbname=shop', 'root', '');
    }

    // 🦨 Dead Code — কোথাও ব্যবহৃত হয় না
    public function oldCalculateShipping($weight)
    {
        return $weight * 5;
    }

    // 🦨 Long Method + God Class — একটি মেথডে সব কিছু
    public function processOrder(
        string $customerName,      // 🦨 Long Parameter List
        string $customerEmail,
        string $customerPhone,
        string $shippingStreet,
        string $shippingCity,
        string $shippingZip,
        string $shippingCountry,
        array $items,
        string $paymentType,
        string $cardNumber,
        string $couponCode
    ): array {
        // 🦨 Duplicate Code — ভ্যালিডেশন বারবার
        if (empty($customerName)) throw new Exception('নাম দরকার');
        if (empty($customerEmail)) throw new Exception('ইমেইল দরকার');
        if (!filter_var($customerEmail, FILTER_VALIDATE_EMAIL)) throw new Exception('ভুল ইমেইল');
        if (empty($customerPhone)) throw new Exception('ফোন দরকার');
        if (!preg_match('/^01[3-9]\d{8}$/', $customerPhone)) throw new Exception('ভুল ফোন');
        if (empty($items)) throw new Exception('প্রোডাক্ট দরকার');

        // 🦨 Primitive Obsession — টাকা আর ঠিকানা সব string/float
        $subtotal = 0;
        foreach ($items as $item) {
            $subtotal += $item['price'] * $item['quantity'];
        }

        // 🦨 Duplicate Code — ডিসকাউন্ট হিসাব অন্য জায়গায়ও আছে
        $discount = 0;
        if ($couponCode === 'SAVE10') {
            $discount = $subtotal * 0.10;
        } elseif ($couponCode === 'SAVE20') {
            $discount = $subtotal * 0.20;
        } elseif ($couponCode === 'BDFREE') {
            $discount = $subtotal > 5000 ? 500 : 0;
        }

        $afterDiscount = $subtotal - $discount;

        // 🦨 Switch Statements — পেমেন্ট টাইপ ভেদে
        $paymentFee = 0;
        switch ($paymentType) {
            case 'credit_card':
                $paymentFee = $afterDiscount * 0.03;
                break;
            case 'bkash':
                $paymentFee = $afterDiscount * 0.015;
                break;
            case 'cod':
                $paymentFee = 50;
                break;
        }

        // 🦨 Feature Envy — শিপিং হিসাব এখানে না থেকে আলাদা থাকা উচিত
        $totalWeight = 0;
        foreach ($items as $item) {
            $totalWeight += ($item['weight'] ?? 0.5) * $item['quantity'];
        }
        $shippingCost = $totalWeight * 15;
        if ($shippingCountry !== 'বাংলাদেশ') {
            $shippingCost *= 5;
        }

        $tax = $afterDiscount * 0.15;
        $total = $afterDiscount + $paymentFee + $shippingCost + $tax;

        // ডাটাবেসে সেভ
        $stmt = $this->db->prepare(
            "INSERT INTO orders (customer_name, email, phone, total, status) VALUES (?, ?, ?, ?, 'pending')"
        );
        $stmt->execute([$customerName, $customerEmail, $customerPhone, $total]);
        $orderId = $this->db->lastInsertId();

        // 🦨 Shotgun Surgery — ইমেইল ফরম্যাট বদলালে এখানেও বদলাতে হবে
        $emailBody = "প্রিয় {$customerName},\n";
        $emailBody .= "আপনার অর্ডার #{$orderId} সফলভাবে গৃহীত হয়েছে।\n";
        $emailBody .= "সাবটোটাল: ৳{$subtotal}\n";
        $emailBody .= "ডিসকাউন্ট: ৳{$discount}\n";
        $emailBody .= "শিপিং: ৳{$shippingCost}\n";
        $emailBody .= "ট্যাক্স: ৳{$tax}\n";
        $emailBody .= "মোট: ৳{$total}\n";
        mail($customerEmail, "অর্ডার #{$orderId}", $emailBody);

        return [
            'order_id' => $orderId,
            'subtotal' => $subtotal,
            'discount' => $discount,
            'shipping' => $shippingCost,
            'tax' => $tax,
            'total' => $total
        ];
    }
}
```

---

## ❌ স্মেলযুক্ত কোড — JavaScript

```javascript
// 🦨 সব স্মেল একসাথে — God Class, Long Method, Primitive Obsession, Duplicate Code
class ECommerceSystem {
    constructor() {
        this.db = require('./database');
    }

    // 🦨 Dead Code
    oldShippingCalc(weight) {
        return weight * 5;
    }

    // 🦨 Long Method + Long Parameter List
    processOrder(name, email, phone, street, city, zip, country, items, paymentType, cardNum, coupon) {
        // 🦨 ভ্যালিডেশন Duplicate
        if (!name) throw new Error('নাম দরকার');
        if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) throw new Error('ভুল ইমেইল');
        if (!phone || !/^01[3-9]\d{8}$/.test(phone)) throw new Error('ভুল ফোন');
        if (!items || items.length === 0) throw new Error('প্রোডাক্ট দরকার');

        // 🦨 Primitive Obsession
        let subtotal = 0;
        items.forEach(i => { subtotal += i.price * i.quantity; });

        // 🦨 Duplicate discount logic
        let discount = 0;
        if (coupon === 'SAVE10') discount = subtotal * 0.10;
        else if (coupon === 'SAVE20') discount = subtotal * 0.20;
        else if (coupon === 'BDFREE') discount = subtotal > 5000 ? 500 : 0;

        const afterDiscount = subtotal - discount;

        // 🦨 Switch Statement
        let paymentFee = 0;
        if (paymentType === 'credit_card') paymentFee = afterDiscount * 0.03;
        else if (paymentType === 'bkash') paymentFee = afterDiscount * 0.015;
        else if (paymentType === 'cod') paymentFee = 50;

        // 🦨 Feature Envy
        let totalWeight = 0;
        items.forEach(i => { totalWeight += (i.weight || 0.5) * i.quantity; });
        let shippingCost = totalWeight * 15;
        if (country !== 'বাংলাদেশ') shippingCost *= 5;

        const tax = afterDiscount * 0.15;
        const total = afterDiscount + paymentFee + shippingCost + tax;

        const orderId = this.db.insert('orders', { name, email, phone, total, status: 'pending' });

        // 🦨 Shotgun Surgery — ইমেইল টেমপ্লেট এখানে হার্ডকোড
        const mailer = require('./mailer');
        mailer.send(email, `অর্ডার #${orderId}`, `মোট: ৳${total}`);

        return { orderId, subtotal, discount, shippingCost, tax, total };
    }
}
```

---

## ✅ রিফ্যাক্টর করা ক্লিন কোড — PHP

```php
<?php

// ✅ ভ্যালু অবজেক্ট — Primitive Obsession সমাধান
class Money
{
    public function __construct(private readonly float $amount)
    {
        if ($amount < 0) throw new InvalidArgumentException('ঋণাত্মক টাকা অসম্ভব');
    }

    public function add(Money $other): self { return new self($this->amount + $other->amount); }
    public function subtract(Money $other): self { return new self($this->amount - $other->amount); }
    public function multiply(float $factor): self { return new self($this->amount * $factor); }
    public function getAmount(): float { return $this->amount; }
    public function __toString(): string { return "৳" . number_format($this->amount, 2); }
}

class Email
{
    private string $address;
    public function __construct(string $address)
    {
        if (!filter_var($address, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('ভুল ইমেইল');
        }
        $this->address = $address;
    }
    public function __toString(): string { return $this->address; }
}

class PhoneNumber
{
    private string $number;
    public function __construct(string $number)
    {
        if (!preg_match('/^01[3-9]\d{8}$/', $number)) {
            throw new InvalidArgumentException('ভুল ফোন নম্বর');
        }
        $this->number = $number;
    }
    public function __toString(): string { return $this->number; }
}

// ✅ Data Clumps সমাধান — ঠিকানা ক্লাস
class Address
{
    public function __construct(
        public readonly string $street,
        public readonly string $city,
        public readonly string $zip,
        public readonly string $country
    ) {}

    public function isInternational(): bool
    {
        return $this->country !== 'বাংলাদেশ';
    }
}

// ✅ Parameter Object — Long Parameter List সমাধান
class CustomerInfo
{
    public function __construct(
        public readonly string $name,
        public readonly Email $email,
        public readonly PhoneNumber $phone
    ) {}
}

class OrderItem
{
    public function __construct(
        public readonly string $productName,
        public readonly Money $price,
        public readonly int $quantity,
        public readonly float $weight = 0.5
    ) {}

    public function getTotal(): Money { return $this->price->multiply($this->quantity); }
    public function getTotalWeight(): float { return $this->weight * $this->quantity; }
}

// ✅ Switch Statement সমাধান — পলিমরফিজম
interface PaymentMethod
{
    public function calculateFee(Money $amount): Money;
    public function process(Money $amount): bool;
}

class CreditCardPayment implements PaymentMethod
{
    public function calculateFee(Money $amount): Money { return $amount->multiply(0.03); }
    public function process(Money $amount): bool { return true; }
}

class BkashPayment implements PaymentMethod
{
    public function calculateFee(Money $amount): Money { return $amount->multiply(0.015); }
    public function process(Money $amount): bool { return true; }
}

class CodPayment implements PaymentMethod
{
    public function calculateFee(Money $amount): Money { return new Money(50); }
    public function process(Money $amount): bool { return true; }
}

// ✅ Duplicate Code সমাধান — কুপন লজিক এক জায়গায়
interface CouponStrategy
{
    public function calculateDiscount(Money $subtotal): Money;
}

class PercentageCoupon implements CouponStrategy
{
    public function __construct(private float $percentage) {}

    public function calculateDiscount(Money $subtotal): Money
    {
        return $subtotal->multiply($this->percentage / 100);
    }
}

class ConditionalFlatCoupon implements CouponStrategy
{
    public function __construct(
        private float $minAmount,
        private float $discountAmount
    ) {}

    public function calculateDiscount(Money $subtotal): Money
    {
        return $subtotal->getAmount() > $this->minAmount
            ? new Money($this->discountAmount)
            : new Money(0);
    }
}

class CouponRegistry
{
    private array $coupons = [];

    public function __construct()
    {
        $this->coupons['SAVE10'] = new PercentageCoupon(10);
        $this->coupons['SAVE20'] = new PercentageCoupon(20);
        $this->coupons['BDFREE'] = new ConditionalFlatCoupon(5000, 500);
    }

    public function resolve(string $code): ?CouponStrategy
    {
        return $this->coupons[$code] ?? null;
    }
}

// ✅ Feature Envy সমাধান — শিপিং হিসাব নিজের ক্লাসে
class ShippingCalculator
{
    private const RATE_PER_KG = 15;
    private const INTERNATIONAL_MULTIPLIER = 5;

    public function calculate(array $items, Address $address): Money
    {
        $totalWeight = array_reduce($items, fn($sum, OrderItem $item) =>
            $sum + $item->getTotalWeight(), 0);

        $cost = $totalWeight * self::RATE_PER_KG;
        if ($address->isInternational()) {
            $cost *= self::INTERNATIONAL_MULTIPLIER;
        }
        return new Money($cost);
    }
}

// ✅ God Class ভেঙে ছোট ক্লাসে — প্রতিটি ক্লাসের একটি দায়িত্ব
class PriceCalculator
{
    public function __construct(
        private ShippingCalculator $shipping,
        private CouponRegistry $coupons
    ) {}

    public function calculate(array $items, Address $address, PaymentMethod $payment, ?string $couponCode): OrderPricing
    {
        $subtotal = array_reduce($items, fn(Money $sum, OrderItem $item) =>
            $sum->add($item->getTotal()), new Money(0));

        $discount = new Money(0);
        if ($couponCode) {
            $coupon = $this->coupons->resolve($couponCode);
            if ($coupon) {
                $discount = $coupon->calculateDiscount($subtotal);
            }
        }

        $afterDiscount = $subtotal->subtract($discount);
        $paymentFee = $payment->calculateFee($afterDiscount);
        $shippingCost = $this->shipping->calculate($items, $address);
        $tax = $afterDiscount->multiply(0.15);
        $total = $afterDiscount->add($paymentFee)->add($shippingCost)->add($tax);

        return new OrderPricing($subtotal, $discount, $paymentFee, $shippingCost, $tax, $total);
    }
}

class OrderPricing
{
    public function __construct(
        public readonly Money $subtotal,
        public readonly Money $discount,
        public readonly Money $paymentFee,
        public readonly Money $shippingCost,
        public readonly Money $tax,
        public readonly Money $total
    ) {}
}

// ✅ Shotgun Surgery সমাধান — ইমেইল টেমপ্লেট এক জায়গায়
class OrderConfirmationEmail
{
    public function build(int $orderId, string $customerName, OrderPricing $pricing): string
    {
        return "প্রিয় {$customerName},\n"
            . "আপনার অর্ডার #{$orderId} সফলভাবে গৃহীত হয়েছে।\n"
            . "সাবটোটাল: {$pricing->subtotal}\n"
            . "ডিসকাউন্ট: {$pricing->discount}\n"
            . "শিপিং: {$pricing->shippingCost}\n"
            . "ট্যাক্স: {$pricing->tax}\n"
            . "মোট: {$pricing->total}\n";
    }
}

// ✅ মূল অর্ডার প্রসেসর — ছোট, পরিষ্কার, পড়তে সহজ
class OrderProcessor
{
    public function __construct(
        private PriceCalculator $calculator,
        private OrderRepository $repository,
        private EmailService $emailService,
        private OrderConfirmationEmail $emailTemplate
    ) {}

    public function process(
        CustomerInfo $customer,
        Address $shippingAddress,
        array $items,
        PaymentMethod $payment,
        ?string $couponCode = null
    ): OrderResult {
        $pricing = $this->calculator->calculate($items, $shippingAddress, $payment, $couponCode);
        $payment->process($pricing->total);
        $orderId = $this->repository->save($customer, $shippingAddress, $items, $pricing);

        $emailBody = $this->emailTemplate->build($orderId, $customer->name, $pricing);
        $this->emailService->send((string) $customer->email, "অর্ডার #{$orderId}", $emailBody);

        return new OrderResult($orderId, $pricing);
    }
}
```

---

## ✅ রিফ্যাক্টর করা ক্লিন কোড — JavaScript

```javascript
// ✅ ভ্যালু অবজেক্ট — Primitive Obsession সমাধান
class Money {
    #amount;
    constructor(amount) {
        if (amount < 0) throw new Error('ঋণাত্মক টাকা অসম্ভব');
        this.#amount = amount;
    }
    add(other) { return new Money(this.#amount + other.amount); }
    subtract(other) { return new Money(this.#amount - other.amount); }
    multiply(factor) { return new Money(this.#amount * factor); }
    get amount() { return this.#amount; }
    toString() { return `৳${this.#amount.toFixed(2)}`; }
}

class Email {
    #address;
    constructor(address) {
        if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(address)) throw new Error('ভুল ইমেইল');
        this.#address = address;
    }
    toString() { return this.#address; }
}

// ✅ Data Clumps সমাধান
class Address {
    constructor(street, city, zip, country) {
        this.street = street;
        this.city = city;
        this.zip = zip;
        this.country = country;
    }
    isInternational() { return this.country !== 'বাংলাদেশ'; }
}

// ✅ Parameter Object
class CustomerInfo {
    constructor(name, email, phone) {
        this.name = name;
        this.email = email;
        this.phone = phone;
    }
}

class OrderItem {
    constructor(productName, price, quantity, weight = 0.5) {
        this.productName = productName;
        this.price = price;
        this.quantity = quantity;
        this.weight = weight;
    }
    getTotal() { return this.price.multiply(this.quantity); }
    getTotalWeight() { return this.weight * this.quantity; }
}

// ✅ Switch Statement সমাধান — পলিমরফিজম
class CreditCardPayment {
    calculateFee(amount) { return amount.multiply(0.03); }
    async process(amount) { return true; }
}

class BkashPayment {
    calculateFee(amount) { return amount.multiply(0.015); }
    async process(amount) { return true; }
}

class CodPayment {
    calculateFee(amount) { return new Money(50); }
    async process(amount) { return true; }
}

// ✅ কুপন লজিক কেন্দ্রীভূত
class CouponRegistry {
    #coupons = new Map([
        ['SAVE10', (subtotal) => subtotal.multiply(0.10)],
        ['SAVE20', (subtotal) => subtotal.multiply(0.20)],
        ['BDFREE', (subtotal) => subtotal.amount > 5000 ? new Money(500) : new Money(0)],
    ]);

    resolve(code) {
        return this.#coupons.get(code) ?? null;
    }
}

// ✅ Feature Envy সমাধান
class ShippingCalculator {
    static RATE_PER_KG = 15;
    static INTERNATIONAL_MULTIPLIER = 5;

    calculate(items, address) {
        const totalWeight = items.reduce((sum, item) => sum + item.getTotalWeight(), 0);
        let cost = totalWeight * ShippingCalculator.RATE_PER_KG;
        if (address.isInternational()) cost *= ShippingCalculator.INTERNATIONAL_MULTIPLIER;
        return new Money(cost);
    }
}

// ✅ মূল প্রসেসর — ছোট ও পরিষ্কার
class OrderProcessor {
    constructor(calculator, repository, emailService) {
        this.calculator = calculator;
        this.repository = repository;
        this.emailService = emailService;
    }

    async process(customer, shippingAddress, items, payment, couponCode = null) {
        const pricing = this.calculator.calculate(items, shippingAddress, payment, couponCode);
        await payment.process(pricing.total);
        const orderId = await this.repository.save(customer, shippingAddress, items, pricing);
        await this.emailService.sendOrderConfirmation(customer, orderId, pricing);

        return { orderId, pricing };
    }
}
```

---

## 🔍 রিফ্যাক্টরিং আগে ও পরে তুলনা

```
❌ আগে (১টি ক্লাস, ১৫০+ লাইন)          ✅ পরে (১৫+ ছোট ক্লাস)
─────────────────────────────            ───────────────────────────
ECommerceSystem                          Money (ভ্যালু অবজেক্ট)
  │                                      Email (ভ্যালু অবজেক্ট)
  ├── processOrder() (৮০+ লাইন)          PhoneNumber (ভ্যালু অবজেক্ট)
  │   ├── ভ্যালিডেশন                     Address (ডাটা ক্লাম্প)
  │   ├── দাম হিসাব                      CustomerInfo (প্যারামিটার অবজেক্ট)
  │   ├── ডিসকাউন্ট                      OrderItem
  │   ├── শিপিং                          PaymentMethod (ইন্টারফেস)
  │   ├── ট্যাক্স                        ├── CreditCardPayment
  │   ├── ডাটাবেস সেভ                    ├── BkashPayment
  │   └── ইমেইল পাঠানো                   └── CodPayment
  │                                      CouponStrategy (ইন্টারফেস)
  └── oldCalculateShipping() (Dead)      ├── PercentageCoupon
                                         └── ConditionalFlatCoupon
                                         ShippingCalculator
                                         PriceCalculator
                                         OrderProcessor (মূল অর্কেস্ট্রেটর)
                                         OrderConfirmationEmail
                                         OrderRepository

✅ প্রতিটি ক্লাস ছোট, পড়তে সহজ, টেস্ট করা সহজ
✅ নতুন পেমেন্ট/কুপন যোগ করতে বিদ্যমান কোড বদলাতে হয় না
✅ একটি বদলালে অন্যগুলো প্রভাবিত হয় না
```

---

# 📋 কোড স্মেল চেকলিস্ট

| # | স্মেল | ক্যাটাগরি | লক্ষণ | সমাধান |
|---|-------|----------|-------|--------|
| ১ | Long Method | ব্লোটার | মেথড ৫০+ লাইন, একাধিক কাজ করে | Extract Method — ছোট মেথডে ভাগ |
| ২ | God Class | ব্লোটার | ক্লাসে ১০+ মেথড, বিভিন্ন দায়িত্ব | ক্লাস ভেঙে SRP অনুসরণ |
| ৩ | Long Parameter List | ব্লোটার | ফাংশনে ৪+ প্যারামিটার | Parameter Object তৈরি |
| ৪ | Data Clumps | ব্লোটার | একই ডাটা গ্রুপ বারবার পাস হয় | ডাটা ক্লাস তৈরি |
| ৫ | Primitive Obsession | ব্লোটার | সব কিছু string/int দিয়ে | Value Object তৈরি |
| ৬ | Switch Statements | OO অ্যাবিউজ | একই switch বিভিন্ন জায়গায় | পলিমরফিজম ব্যবহার |
| ৭ | Parallel Inheritance | OO অ্যাবিউজ | সাবক্লাস জোড়ায় জোড়ায় বাড়ে | কম্পোজিশন ব্যবহার |
| ৮ | Refused Bequest | OO অ্যাবিউজ | চাইল্ড প্যারেন্টের মেথড ব্যবহার করে না | কম্পোজিশন বা ইন্টারফেস |
| ৯ | Alt. Classes Diff. Interface | OO অ্যাবিউজ | একই কাজ ভিন্ন API দিয়ে | কমন ইন্টারফেস তৈরি |
| ১০ | Divergent Change | চেঞ্জ প্রিভেন্টার | একটি ক্লাস বিভিন্ন কারণে বদলায় | দায়িত্ব আলাদা করা |
| ১১ | Shotgun Surgery | চেঞ্জ প্রিভেন্টার | একটি বদলে অনেক ফাইল বদলাতে হয় | সম্পর্কিত লজিক একত্রিত করা |
| ১২ | Feature Envy | চেঞ্জ প্রিভেন্টার | মেথড অন্য ক্লাসের ডাটা বেশি ব্যবহার করে | মেথড সঠিক ক্লাসে সরানো |
| ১৩ | Dead Code | ডিসপেন্সেবল | অব্যবহৃত ভ্যারিয়েবল/মেথড/ক্লাস | মুছে ফেলুন (Git এ আছে) |
| ১৪ | Duplicate Code | ডিসপেন্সেবল | একই লজিক কপি-পেস্ট | Extract Method/Class |
| ১৫ | Speculative Generality | ডিসপেন্সেবল | "কাজে লাগতে পারে" কোড | YAGNI — দরকার হলে পরে করুন |
| ১৬ | Lazy Class | ডিসপেন্সেবল | প্রায় খালি ক্লাস | ইনলাইন বা মার্জ করুন |
| ১৭ | Inappropriate Intimacy | কাপলার | ক্লাস অন্যের internal জানে | এনক্যাপসুলেশন শক্তিশালী করুন |
| ১৮ | Message Chains | কাপলার | a.b().c().d() চেইন | ডেলিগেট মেথড তৈরি |
| ১৯ | Middle Man | কাপলার | শুধু পাস-থ্রু মেথড | সরিয়ে দিন বা লজিক যোগ করুন |

---

## 🧠 কোড স্মেল শনাক্তকরণ মানসিক মডেল

```
কোড রিভিউ করার সময় নিজেকে জিজ্ঞাসা করুন:

┌─────────────────────────────────────────────────────────┐
│  ১. এই মেথড/ক্লাস কি একটির বেশি কাজ করছে?             │
│     হ্যাঁ → God Class / Long Method                     │
│                                                         │
│  ২. এই ফাংশন কি ৪+ প্যারামিটার নিচ্ছে?                │
│     হ্যাঁ → Long Parameter List / Data Clumps           │
│                                                         │
│  ৩. একই কোড কি অন্য জায়গায়ও আছে?                     │
│     হ্যাঁ → Duplicate Code                              │
│                                                         │
│  ৪. এই কোড কি আসলে ব্যবহৃত হচ্ছে?                     │
│     না → Dead Code                                      │
│                                                         │
│  ৫. একটি ছোট পরিবর্তনে কতগুলো ফাইল বদলাতে হচ্ছে?     │
│     অনেক → Shotgun Surgery                              │
│                                                         │
│  ৬. এই মেথড কি অন্য ক্লাসের ডাটা বেশি ব্যবহার করছে?   │
│     হ্যাঁ → Feature Envy                                │
│                                                         │
│  ৭. switch/if-else কি বারবার একই কন্ডিশনে?             │
│     হ্যাঁ → Switch Statements (পলিমরফিজম দরকার)         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 কোড স্মেল vs রিফ্যাক্টরিং কৌশল ম্যাপিং

```
কোড স্মেল                    রিফ্যাক্টরিং কৌশল
─────────────                ──────────────────────
Long Method           ──→    Extract Method
God Class             ──→    Extract Class
Long Parameter List   ──→    Introduce Parameter Object
Data Clumps           ──→    Extract Class / Value Object
Primitive Obsession   ──→    Replace Primitive with Object
Switch Statements     ──→    Replace Conditional with Polymorphism
Duplicate Code        ──→    Extract Method / Pull Up Method
Feature Envy          ──→    Move Method
Dead Code             ──→    Remove Dead Code
Shotgun Surgery       ──→    Move Method / Inline Class
Message Chains        ──→    Hide Delegate
Middle Man            ──→    Remove Middle Man
Speculative Generality──→    Collapse Hierarchy / Remove
Lazy Class            ──→    Inline Class
```

---

## 📊 স্মেল তীব্রতা স্কেল

| স্তর | তীব্রতা | উদাহরণ | পদক্ষেপ |
|-------|---------|--------|---------|
| 🟢 | হালকা | একটি Lazy Class, ছোট ডুপ্লিকেশন | সময় পেলে ঠিক করুন |
| 🟡 | মাঝারি | Long Method, Data Clumps | পরবর্তী স্প্রিন্টে ঠিক করুন |
| 🟠 | গুরুতর | God Class, Shotgun Surgery | যত দ্রুত সম্ভব ঠিক করুন |
| 🔴 | জরুরি | ব্যাপক Duplicate Code + God Class | এখনই রিফ্যাক্টর করুন |

---

> **"প্রথমবার লেখা কোড খসড়া মাত্র। আসল কোড আসে রিফ্যাক্টরিংয়ের পরে।"**

---

## 🔗 পরবর্তী টপিক

➡️ [রিফ্যাক্টরিং কৌশল (Refactoring Techniques)](./refactoring-techniques.md) — কোড স্মেল দূর করার ব্যবহারিক পদ্ধতি শিখুন।
