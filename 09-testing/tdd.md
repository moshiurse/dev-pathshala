# 🔴🟢🔵 TDD (টেস্ট ড্রিভেন ডেভেলপমেন্ট)

> "Clean code that works — that's the goal of Test-Driven Development." — Kent Beck

---

## 📌 সংজ্ঞা

**Test-Driven Development (TDD)** হলো একটি সফটওয়্যার ডেভেলপমেন্ট পদ্ধতি যেখানে আপনি **প্রথমে টেস্ট লেখেন**, তারপর সেই টেস্ট পাস করার জন্য **ন্যূনতম কোড** লেখেন, এবং সবশেষে কোড **রিফ্যাক্টর** করেন। এটি Kent Beck ১৯৯০-এর দশকে Extreme Programming (XP)-এর অংশ হিসেবে জনপ্রিয় করেন।

TDD-এর মূল দর্শন হলো — **আপনি কোনো প্রোডাকশন কোড লিখবেন না যতক্ষণ না একটি ফেইলিং টেস্ট থাকে।** এটি শুধু টেস্টিং কৌশল নয়, এটি একটি **ডিজাইন কৌশল** যা আপনার কোডের আর্কিটেকচার এবং API ডিজাইনকে গাইড করে।

### Red-Green-Refactor Cycle

TDD তিনটি ধাপে কাজ করে, যা **Red-Green-Refactor** নামে পরিচিত:

| ধাপ | রং | কাজ | সময় |
|------|-----|------|-------|
| **Red** | 🔴 | একটি ফেইলিং টেস্ট লিখুন | ১-৩ মিনিট |
| **Green** | 🟢 | ন্যূনতম কোড লিখে টেস্ট পাস করান | ১-৫ মিনিট |
| **Refactor** | 🔵 | কোড পরিষ্কার করুন, duplication দূর করুন | ২-৫ মিনিট |

> ⚡ প্রতিটি সাইকেল ১০ মিনিটের বেশি হওয়া উচিত নয়। বড় সমস্যা ছোট ছোট ধাপে ভাগ করুন।

---

## 📊 TDD Cycle Diagram

```
              ┌─────────────────────────────────────────────┐
              │           TDD Red-Green-Refactor             │
              └─────────────────────────────────────────────┘

                         ┌──────────┐
                         │  🔴 RED   │
                         │  Write a  │
                         │ Failing   │
                    ┌───▶│  Test     │───────┐
                    │    └──────────┘        │
                    │                        ▼
              ┌──────────┐           ┌──────────┐
              │ 🔵 REFAC- │           │ 🟢 GREEN │
              │   TOR     │◀──────── │ Make it   │
              │ Clean up  │          │  Pass     │
              └──────────┘           └──────────┘
                    │
                    │    ┌──────────────────────────┐
                    └───▶│ পরবর্তী ফিচারের জন্য     │
                         │ আবার Red থেকে শুরু       │
                         └──────────────────────────┘

  ═══════════════════════════════════════════════════════════

  সম্পূর্ণ Flow:

  [নতুন ফিচার] ──▶ 🔴 ফেইলিং টেস্ট লিখুন
                          │
                          ▼
                    টেস্ট রান করুন ──▶ ❌ FAIL (প্রত্যাশিত!)
                          │
                          ▼
                    🟢 ন্যূনতম কোড লিখুন
                          │
                          ▼
                    টেস্ট রান করুন ──▶ ✅ PASS
                          │
                          ▼
                    🔵 রিফ্যাক্টর করুন
                          │
                          ▼
                    টেস্ট রান করুন ──▶ ✅ PASS (এখনও!)
                          │
                          ▼
                    [পরবর্তী ফিচার] ──▶ 🔴 ...
```

---

## 💻 TDD Step-by-Step: সম্পূর্ণ উদাহরণ

TDD শেখার সবচেয়ে ভালো উপায় হলো **সম্পূর্ণ একটি ফিচার TDD দিয়ে বিল্ড করা**। এখানে আমরা দুটি বাস্তব উদাহরণ দেখবো — PHP এবং JavaScript-এ — যেখানে প্রতিটি Red-Green-Refactor সাইকেল বিস্তারিত দেখানো হবে।

---

### 🐘 PHP Example: Shopping Cart TDD দিয়ে বিল্ড করা (PHPUnit)

আমরা একটি `ShoppingCart` ক্লাস বিল্ড করবো TDD পদ্ধতিতে। প্রতিটি ফিচার যোগ করার আগে টেস্ট লিখবো।

---

#### 🔴 সাইকেল ১ — Red: Cart-এ আইটেম যোগ করা

প্রথমে টেস্ট লিখি। `ShoppingCart` ক্লাস এখনও তৈরি হয়নি!

```php
<?php
// tests/ShoppingCartTest.php

use PHPUnit\Framework\TestCase;

class ShoppingCartTest extends TestCase
{
    public function test_can_add_item_to_cart(): void
    {
        $cart = new ShoppingCart();
        $cart->addItem('Laptop', 75000, 1);

        $this->assertCount(1, $cart->getItems());
        $this->assertEquals('Laptop', $cart->getItems()[0]['name']);
        $this->assertEquals(75000, $cart->getItems()[0]['price']);
    }
}
```

```bash
$ vendor/bin/phpunit tests/ShoppingCartTest.php
# 🔴 FAIL: Class "ShoppingCart" not found
```

#### 🟢 সাইকেল ১ — Green: ন্যূনতম কোড লিখি

```php
<?php
// src/ShoppingCart.php

class ShoppingCart
{
    private array $items = [];

    public function addItem(string $name, float $price, int $quantity): void
    {
        $this->items[] = [
            'name' => $name,
            'price' => $price,
            'quantity' => $quantity,
        ];
    }

    public function getItems(): array
    {
        return $this->items;
    }
}
```

```bash
$ vendor/bin/phpunit tests/ShoppingCartTest.php
# 🟢 OK (1 test, 3 assertions)
```

#### 🔵 সাইকেল ১ — Refactor

এই পর্যায়ে কোড যথেষ্ট সহজ, তাই বড় রিফ্যাক্টরিং দরকার নেই। তবে আমরা একটি `CartItem` ভ্যালু অবজেক্ট বানাতে পারি — কিন্তু YAGNI (You Ain't Gonna Need It) মনে রেখে এখন রাখি।

---

#### 🔴 সাইকেল ২ — Red: Cart থেকে আইটেম রিমুভ করা

```php
public function test_can_remove_item_from_cart(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Laptop', 75000, 1);
    $cart->addItem('Mouse', 500, 2);

    $cart->removeItem('Laptop');

    $this->assertCount(1, $cart->getItems());
    $this->assertEquals('Mouse', $cart->getItems()[0]['name']);
}
```

```bash
$ vendor/bin/phpunit
# 🔴 FAIL: Call to undefined method ShoppingCart::removeItem()
```

#### 🟢 সাইকেল ২ — Green

```php
public function removeItem(string $name): void
{
    $this->items = array_values(
        array_filter($this->items, fn($item) => $item['name'] !== $name)
    );
}
```

```bash
# 🟢 OK (2 tests, 5 assertions)
```

#### 🔵 সাইকেল ২ — Refactor

`removeItem` মেথডটি ঠিক আছে। তবে আমরা লক্ষ্য করছি item গুলো associative array হিসেবে আছে। পরে DTO/Value Object বানাবো যদি জটিলতা বাড়ে।

---

#### 🔴 সাইকেল ৩ — Red: মোট মূল্য হিসাব (calculateTotal)

```php
public function test_calculates_total_price(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Laptop', 75000, 1);
    $cart->addItem('Mouse', 500, 2);

    // Laptop: 75000*1 + Mouse: 500*2 = 76000
    $this->assertEquals(76000, $cart->calculateTotal());
}
```

```bash
# 🔴 FAIL: Call to undefined method ShoppingCart::calculateTotal()
```

#### 🟢 সাইকেল ৩ — Green

```php
public function calculateTotal(): float
{
    return array_reduce($this->items, function (float $total, array $item) {
        return $total + ($item['price'] * $item['quantity']);
    }, 0);
}
```

```bash
# 🟢 OK (3 tests, 6 assertions)
```

---

#### 🔴 সাইকেল ৪ — Red: ডিসকাউন্ট সিস্টেম

```php
public function test_applies_percentage_discount(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Laptop', 10000, 1);

    $cart->applyDiscount(10); // ১০% ডিসকাউন্ট

    $this->assertEquals(9000, $cart->calculateTotal());
}

public function test_discount_cannot_exceed_100_percent(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Laptop', 10000, 1);

    $this->expectException(\InvalidArgumentException::class);
    $cart->applyDiscount(150);
}

public function test_discount_cannot_be_negative(): void
{
    $cart = new ShoppingCart();
    $this->expectException(\InvalidArgumentException::class);
    $cart->applyDiscount(-5);
}
```

```bash
# 🔴 FAIL: Call to undefined method ShoppingCart::applyDiscount()
```

#### 🟢 সাইকেল ৪ — Green

```php
private float $discountPercent = 0;

public function applyDiscount(float $percent): void
{
    if ($percent < 0 || $percent > 100) {
        throw new \InvalidArgumentException(
            "ডিসকাউন্ট ০ থেকে ১০০-এর মধ্যে হতে হবে। দেওয়া হয়েছে: {$percent}"
        );
    }
    $this->discountPercent = $percent;
}

public function calculateTotal(): float
{
    $subtotal = array_reduce($this->items, function (float $total, array $item) {
        return $total + ($item['price'] * $item['quantity']);
    }, 0);

    return $subtotal * (1 - $this->discountPercent / 100);
}
```

```bash
# 🟢 OK (6 tests, 9 assertions)
```

---

#### 🔴 সাইকেল ৫ — Red: কুপন কোড সিস্টেম

```php
public function test_applies_coupon_code(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Phone', 20000, 1);

    $cart->applyCoupon('SAVE500', 500); // ৫০০ টাকা ছাড়

    $this->assertEquals(19500, $cart->calculateTotal());
}

public function test_coupon_discount_cannot_exceed_total(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Pen', 50, 1);

    $cart->applyCoupon('BIG_SAVE', 500);

    // Total ০-এর নিচে যেতে পারবে না
    $this->assertEquals(0, $cart->calculateTotal());
}
```

#### 🟢 সাইকেল ৫ — Green

```php
private float $couponDiscount = 0;
private ?string $couponCode = null;

public function applyCoupon(string $code, float $amount): void
{
    $this->couponCode = $code;
    $this->couponDiscount = $amount;
}

public function calculateTotal(): float
{
    $subtotal = array_reduce($this->items, function (float $total, array $item) {
        return $total + ($item['price'] * $item['quantity']);
    }, 0);

    $afterPercentDiscount = $subtotal * (1 - $this->discountPercent / 100);
    $afterCoupon = $afterPercentDiscount - $this->couponDiscount;

    return max(0, $afterCoupon);
}
```

```bash
# 🟢 OK (8 tests, 11 assertions)
```

---

#### 🔴 সাইকেল ৬ — Red: Cart-এর সারসংক্ষেপ (Summary)

```php
public function test_generates_cart_summary(): void
{
    $cart = new ShoppingCart();
    $cart->addItem('Laptop', 75000, 1);
    $cart->addItem('Mouse', 500, 2);
    $cart->applyDiscount(10);
    $cart->applyCoupon('WELCOME', 200);

    $summary = $cart->getSummary();

    $this->assertEquals(76000, $summary['subtotal']);
    $this->assertEquals(7600, $summary['discount_amount']);
    $this->assertEquals(200, $summary['coupon_discount']);
    $this->assertEquals(68200, $summary['total']);
    $this->assertCount(2, $summary['items']);
}
```

#### 🟢 সাইকেল ৬ — Green

```php
public function getSummary(): array
{
    $subtotal = array_reduce($this->items, function (float $total, array $item) {
        return $total + ($item['price'] * $item['quantity']);
    }, 0);

    $discountAmount = $subtotal * ($this->discountPercent / 100);
    $afterDiscount = $subtotal - $discountAmount;
    $total = max(0, $afterDiscount - $this->couponDiscount);

    return [
        'items' => $this->items,
        'subtotal' => $subtotal,
        'discount_percent' => $this->discountPercent,
        'discount_amount' => $discountAmount,
        'coupon_code' => $this->couponCode,
        'coupon_discount' => $this->couponDiscount,
        'total' => $total,
    ];
}
```

#### 🔵 সাইকেল ৬ — Refactor: Duplication দূর করি

লক্ষ্য করুন `calculateTotal()` এবং `getSummary()` তে subtotal ক্যালকুলেশন ডুপ্লিকেট হয়ে গেছে। এবার রিফ্যাক্টর করি:

```php
<?php
// src/ShoppingCart.php — চূড়ান্ত রিফ্যাক্টরড সংস্করণ

class ShoppingCart
{
    private array $items = [];
    private float $discountPercent = 0;
    private float $couponDiscount = 0;
    private ?string $couponCode = null;

    public function addItem(string $name, float $price, int $quantity): void
    {
        $this->items[] = compact('name', 'price', 'quantity');
    }

    public function removeItem(string $name): void
    {
        $this->items = array_values(
            array_filter($this->items, fn($item) => $item['name'] !== $name)
        );
    }

    public function applyDiscount(float $percent): void
    {
        if ($percent < 0 || $percent > 100) {
            throw new \InvalidArgumentException(
                "ডিসকাউন্ট ০-১০০ এর মধ্যে হতে হবে।"
            );
        }
        $this->discountPercent = $percent;
    }

    public function applyCoupon(string $code, float $amount): void
    {
        $this->couponCode = $code;
        $this->couponDiscount = $amount;
    }

    public function calculateTotal(): float
    {
        return $this->getSummary()['total'];
    }

    public function getItems(): array
    {
        return $this->items;
    }

    public function getSummary(): array
    {
        $subtotal = $this->calculateSubtotal();
        $discountAmount = $subtotal * ($this->discountPercent / 100);
        $total = max(0, $subtotal - $discountAmount - $this->couponDiscount);

        return [
            'items'             => $this->items,
            'subtotal'          => $subtotal,
            'discount_percent'  => $this->discountPercent,
            'discount_amount'   => $discountAmount,
            'coupon_code'       => $this->couponCode,
            'coupon_discount'   => $this->couponDiscount,
            'total'             => $total,
        ];
    }

    private function calculateSubtotal(): float
    {
        return array_reduce(
            $this->items,
            fn(float $sum, array $item) => $sum + ($item['price'] * $item['quantity']),
            0
        );
    }
}
```

```bash
$ vendor/bin/phpunit
# 🟢 OK (9 tests, 14 assertions) — সবগুলো টেস্ট এখনও পাস!
```

> 💡 **গুরুত্বপূর্ণ**: রিফ্যাক্টরের পরে সব টেস্ট পাস করতে হবে। যদি কোনো টেস্ট ফেইল করে, তাহলে রিফ্যাক্টরিং ভুল হয়েছে — আগের অবস্থায় ফিরে যান।

---

### 🟨 JavaScript Example: Password Validator TDD দিয়ে বিল্ড করা (Jest)

এবার আমরা JavaScript-এ একটি `PasswordValidator` বিল্ড করবো TDD পদ্ধতিতে।

---

#### 🔴 সাইকেল ১ — Red: ন্যূনতম দৈর্ঘ্য যাচাই

```javascript
// __tests__/passwordValidator.test.js

const { PasswordValidator } = require('../src/passwordValidator');

describe('PasswordValidator', () => {
  let validator;

  beforeEach(() => {
    validator = new PasswordValidator();
  });

  test('ন্যূনতম ৮ অক্ষরের কম পাসওয়ার্ড reject করে', () => {
    const result = validator.validate('Ab1!xyz');
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
  });

  test('৮ বা তার বেশি অক্ষরের পাসওয়ার্ড এই নিয়মে পাস করে', () => {
    const result = validator.validate('Ab1!xyzw');
    expect(result.errors).not.toContain('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
  });
});
```

```bash
$ npx jest
# 🔴 FAIL: Cannot find module '../src/passwordValidator'
```

#### 🟢 সাইকেল ১ — Green

```javascript
// src/passwordValidator.js

class PasswordValidator {
  validate(password) {
    const errors = [];

    if (password.length < 8) {
      errors.push('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
    }

    return {
      isValid: errors.length === 0,
      errors,
    };
  }
}

module.exports = { PasswordValidator };
```

```bash
# 🟢 PASS (2 tests)
```

---

#### 🔴 সাইকেল ২ — Red: বড় হাতের অক্ষর থাকতে হবে

```javascript
test('বড় হাতের অক্ষর না থাকলে reject করে', () => {
  const result = validator.validate('abcd1234!');
  expect(result.isValid).toBe(false);
  expect(result.errors).toContain('কমপক্ষে একটি বড় হাতের অক্ষর (A-Z) থাকতে হবে');
});
```

```bash
# 🔴 FAIL: Expected errors to contain "কমপক্ষে একটি বড় হাতের অক্ষর..."
```

#### 🟢 সাইকেল ২ — Green

```javascript
validate(password) {
  const errors = [];

  if (password.length < 8) {
    errors.push('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('কমপক্ষে একটি বড় হাতের অক্ষর (A-Z) থাকতে হবে');
  }

  return { isValid: errors.length === 0, errors };
}
```

```bash
# 🟢 PASS (3 tests)
```

---

#### 🔴 সাইকেল ৩ — Red: সংখ্যা থাকতে হবে

```javascript
test('সংখ্যা না থাকলে reject করে', () => {
  const result = validator.validate('Abcdefgh!');
  expect(result.isValid).toBe(false);
  expect(result.errors).toContain('কমপক্ষে একটি সংখ্যা (0-9) থাকতে হবে');
});
```

#### 🟢 সাইকেল ৩ — Green

```javascript
if (!/[0-9]/.test(password)) {
  errors.push('কমপক্ষে একটি সংখ্যা (0-9) থাকতে হবে');
}
```

---

#### 🔴 সাইকেল ৪ — Red: বিশেষ চিহ্ন থাকতে হবে

```javascript
test('বিশেষ চিহ্ন না থাকলে reject করে', () => {
  const result = validator.validate('Abcdefg1');
  expect(result.isValid).toBe(false);
  expect(result.errors).toContain('কমপক্ষে একটি বিশেষ চিহ্ন (!@#$%^&*) থাকতে হবে');
});
```

#### 🟢 সাইকেল ৪ — Green

```javascript
if (!/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) {
  errors.push('কমপক্ষে একটি বিশেষ চিহ্ন (!@#$%^&*) থাকতে হবে');
}
```

---

#### 🔴 সাইকেল ৫ — Red: কমন পাসওয়ার্ড ব্লক করা

```javascript
test('কমন পাসওয়ার্ড ব্লক করে', () => {
  const result = validator.validate('Password1!');
  expect(result.isValid).toBe(false);
  expect(result.errors).toContain('এই পাসওয়ার্ড খুব সাধারণ, অন্য কিছু ব্যবহার করুন');
});
```

#### 🟢 সাইকেল ৫ — Green

```javascript
const COMMON_PASSWORDS = [
  'password1!', 'qwerty123!', 'admin123!', '12345678',
  'letmein', 'welcome1', 'monkey123',
];

if (COMMON_PASSWORDS.includes(password.toLowerCase())) {
  errors.push('এই পাসওয়ার্ড খুব সাধারণ, অন্য কিছু ব্যবহার করুন');
}
```

---

#### 🔴 সাইকেল ৬ — Red: পাসওয়ার্ড শক্তি পরিমাপক (Strength Meter)

```javascript
test('দুর্বল পাসওয়ার্ডের শক্তি "weak" রিটার্ন করে', () => {
  const result = validator.validate('Abcdefg1!');
  expect(result.strength).toBe('weak');
});

test('মাঝারি দৈর্ঘ্যের পাসওয়ার্ডের শক্তি "medium" রিটার্ন করে', () => {
  const result = validator.validate('Abcdefgh1!xy');
  expect(result.strength).toBe('medium');
});

test('দীর্ঘ ও জটিল পাসওয়ার্ডের শক্তি "strong" রিটার্ন করে', () => {
  const result = validator.validate('MyS3cur3P@ssw0rd!XYZ');
  expect(result.strength).toBe('strong');
});
```

#### 🟢 সাইকেল ৬ — Green

```javascript
calculateStrength(password) {
  let score = 0;

  if (password.length >= 8)  score += 1;
  if (password.length >= 12) score += 1;
  if (password.length >= 16) score += 1;
  if (/[A-Z]/.test(password) && /[a-z]/.test(password)) score += 1;
  if (/[0-9]/.test(password)) score += 1;
  if (/[!@#$%^&*]/.test(password)) score += 1;

  if (score <= 3) return 'weak';
  if (score <= 4) return 'medium';
  return 'strong';
}
```

#### 🔵 সাইকেল ৬ — Refactor: সম্পূর্ণ ক্লাস পরিষ্কার করি

```javascript
// src/passwordValidator.js — চূড়ান্ত রিফ্যাক্টরড সংস্করণ

const COMMON_PASSWORDS = [
  'password1!', 'qwerty123!', 'admin123!',
  '12345678', 'letmein', 'welcome1',
];

const RULES = [
  {
    test: (pw) => pw.length >= 8,
    message: 'পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে',
  },
  {
    test: (pw) => /[A-Z]/.test(pw),
    message: 'কমপক্ষে একটি বড় হাতের অক্ষর (A-Z) থাকতে হবে',
  },
  {
    test: (pw) => /[a-z]/.test(pw),
    message: 'কমপক্ষে একটি ছোট হাতের অক্ষর (a-z) থাকতে হবে',
  },
  {
    test: (pw) => /[0-9]/.test(pw),
    message: 'কমপক্ষে একটি সংখ্যা (0-9) থাকতে হবে',
  },
  {
    test: (pw) => /[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(pw),
    message: 'কমপক্ষে একটি বিশেষ চিহ্ন (!@#$%^&*) থাকতে হবে',
  },
  {
    test: (pw) => !COMMON_PASSWORDS.includes(pw.toLowerCase()),
    message: 'এই পাসওয়ার্ড খুব সাধারণ, অন্য কিছু ব্যবহার করুন',
  },
];

class PasswordValidator {
  validate(password) {
    const errors = RULES
      .filter((rule) => !rule.test(password))
      .map((rule) => rule.message);

    return {
      isValid: errors.length === 0,
      errors,
      strength: this.#calculateStrength(password),
    };
  }

  #calculateStrength(password) {
    let score = 0;

    if (password.length >= 8)  score++;
    if (password.length >= 12) score++;
    if (password.length >= 16) score++;
    if (/[A-Z]/.test(password) && /[a-z]/.test(password)) score++;
    if (/[0-9]/.test(password)) score++;
    if (/[!@#$%^&*]/.test(password)) score++;

    if (score <= 3) return 'weak';
    if (score <= 4) return 'medium';
    return 'strong';
  }
}

module.exports = { PasswordValidator };
```

```bash
$ npx jest
# 🟢 PASS (9 tests) — রিফ্যাক্টরের পরেও সব পাস!
```

> 🎯 **লক্ষ্য করুন**: রিফ্যাক্টরে আমরা `RULES` array ব্যবহার করে Open/Closed Principle প্রয়োগ করেছি — নতুন নিয়ম যোগ করতে শুধু array-তে একটি অবজেক্ট যোগ করলেই হবে, কোনো কোড পরিবর্তন লাগবে না।

---

## 🔥 Advanced TDD

### ১. Kent Beck-এর TDD তিনটি আইন (Three Laws of TDD)

Robert C. Martin (Uncle Bob) এই আইনগুলো সুস্পষ্টভাবে লিখেছেন, যা Kent Beck-এর TDD দর্শন থেকে এসেছে:

```
┌─────────────────────────────────────────────────────────────┐
│  আইন ১: আপনি কোনো প্রোডাকশন কোড লিখতে পারবেন না         │
│          যতক্ষণ না একটি ফেইলিং ইউনিট টেস্ট থাকে।           │
│                                                             │
│  আইন ২: আপনি ফেইল করার জন্য যতটুকু ইউনিট টেস্ট দরকার     │
│          তার বেশি লিখতে পারবেন না (কম্পাইল না হওয়াও       │
│          ফেইলের মধ্যে পড়ে)।                                │
│                                                             │
│  আইন ৩: বর্তমানে ফেইল হওয়া টেস্ট পাস করার জন্য যতটুকু    │
│          প্রোডাকশন কোড দরকার তার বেশি লিখতে পারবেন না।     │
└─────────────────────────────────────────────────────────────┘
```

**ব্যবহারিক অর্থ:**

এই তিনটি আইন মানলে আপনি **৩০ সেকেন্ড থেকে ২ মিনিটের** মধ্যে একটি Red-Green সাইকেল শেষ করবেন। এটি নিশ্চিত করে যে:
- আপনার কোড সবসময় "almost working" অবস্থায় থাকে
- কোনো বাগ আসলে সেটি সর্বশেষ ১-২ মিনিটের কোডে আছে
- ডিবাগিং প্রায় শূন্যে নেমে আসে

---

### ২. Inside-Out vs Outside-In TDD

TDD-তে দুটি প্রধান স্কুল আছে:

```
┌──────────────────────────┬──────────────────────────────────┐
│     Inside-Out (Detroit)  │       Outside-In (London)        │
│     Classicist School     │        Mockist School            │
├──────────────────────────┼──────────────────────────────────┤
│ ভেতর থেকে শুরু করে       │ বাইরে থেকে শুরু করে              │
│ ক্ষুদ্র ইউনিট → বড় ফিচার│ ইউজার ইন্টারফেস → ভেতরের লজিক  │
│ সত্যিকার অবজেক্ট ব্যবহার │ Mocks/Stubs বেশি ব্যবহার করে    │
│ ইমার্জেন্ট ডিজাইন        │ পরিকল্পিত ডিজাইন                │
│ Kent Beck, Martin Fowler  │ Steve Freeman, Nat Pryce         │
└──────────────────────────┴──────────────────────────────────┘
```

#### Inside-Out উদাহরণ (PHP):

```php
// ভেতরের ইউনিট আগে বানাই
class MoneyTest extends TestCase
{
    public function test_adds_two_amounts(): void
    {
        $a = new Money(100, 'BDT');
        $b = new Money(250, 'BDT');
        $this->assertEquals(new Money(350, 'BDT'), $a->add($b));
    }
}

// তারপর এই ইউনিট ব্যবহার করে বড় ফিচার বানাই
class WalletTest extends TestCase
{
    public function test_calculates_balance(): void
    {
        $wallet = new Wallet();
        $wallet->deposit(new Money(1000, 'BDT'));
        $wallet->withdraw(new Money(300, 'BDT'));
        $this->assertEquals(new Money(700, 'BDT'), $wallet->balance());
    }
}
```

#### Outside-In উদাহরণ (PHP):

```php
// বাইরের ইন্টারফেস আগে ডিজাইন করি, ভেতরটা মক করি
class TransferServiceTest extends TestCase
{
    public function test_transfers_money_between_wallets(): void
    {
        $sourceWallet = $this->createMock(WalletInterface::class);
        $destWallet = $this->createMock(WalletInterface::class);
        $notifier = $this->createMock(NotificationService::class);

        $sourceWallet->expects($this->once())
            ->method('withdraw')
            ->with(new Money(500, 'BDT'));

        $destWallet->expects($this->once())
            ->method('deposit')
            ->with(new Money(500, 'BDT'));

        $notifier->expects($this->once())
            ->method('send');

        $service = new TransferService($sourceWallet, $destWallet, $notifier);
        $service->transfer(new Money(500, 'BDT'));
    }
}
```

**কখন কোনটি ব্যবহার করবেন?**
- **Inside-Out**: যখন ডোমেইন লজিক জটিল, অ্যালগরিদম ভারী (যেমন pricing engine, mathematical calculations)
- **Outside-In**: যখন অনেক collaborator আছে, সিস্টেম ইন্টিগ্রেশন পয়েন্ট বেশি (যেমন payment gateway, notification system)

---

### ৩. TDD with Design Patterns

TDD ডিজাইন প্যাটার্ন "আবিষ্কার" করতে সাহায্য করে। আপনি ইচ্ছাকৃতভাবে প্যাটার্ন প্রয়োগ করেন না — TDD চক্রের রিফ্যাক্টর ধাপে এগুলো স্বাভাবিকভাবে আসে।

#### Strategy Pattern — TDD থেকে আবিষ্কার (JavaScript):

```javascript
// 🔴 Red: বিভিন্ন ধরনের শিপিং ক্যালকুলেশন
describe('ShippingCalculator', () => {
  test('স্ট্যান্ডার্ড শিপিং হিসাব করে', () => {
    const calc = new ShippingCalculator('standard');
    expect(calc.calculate(1000)).toBe(60);
  });

  test('এক্সপ্রেস শিপিং হিসাব করে', () => {
    const calc = new ShippingCalculator('express');
    expect(calc.calculate(1000)).toBe(150);
  });
});

// 🟢 Green: প্রথমে if-else দিয়ে পাস করি
class ShippingCalculator {
  constructor(type) { this.type = type; }

  calculate(orderAmount) {
    if (this.type === 'standard') return orderAmount * 0.06;
    if (this.type === 'express')  return orderAmount * 0.15;
    throw new Error('Unknown shipping type');
  }
}

// 🔵 Refactor: Strategy Pattern আবিষ্কৃত হলো!
const shippingStrategies = {
  standard: (amount) => amount * 0.06,
  express:  (amount) => amount * 0.15,
  free:     (_amount) => 0,
};

class ShippingCalculator {
  constructor(type) {
    this.strategy = shippingStrategies[type];
    if (!this.strategy) throw new Error(`Unknown type: ${type}`);
  }

  calculate(orderAmount) {
    return this.strategy(orderAmount);
  }
}
```

> 🔑 **মূল শিক্ষা**: TDD-তে ডিজাইন "emerge" করে — আপনি আগে থেকে প্যাটার্ন ঠিক করেন না। Red-Green-Refactor চক্র আপনাকে সঠিক প্যাটার্নের দিকে গাইড করে।

---

### ৪. TDD with Legacy Code

Legacy কোডে TDD প্রয়োগ করা সবচেয়ে কঠিন। Michael Feathers তাঁর "Working Effectively with Legacy Code" বইয়ে বলেছেন:

> "Legacy code is code without tests."

#### কৌশল ১: Characterization Tests

Characterization test বর্তমান আচরণ ক্যাপচার করে, সঠিক আচরণ নয়:

```php
// বর্তমান কোড — আমরা জানি না এটি "সঠিক" কিনা
class LegacyPriceCalculator
{
    public function calculate($items, $region)
    {
        // ৫০০ লাইনের স্প্যাগেটি কোড...
        return $total;
    }
}

// Characterization Test — বর্তমান আচরণ রেকর্ড করি
class LegacyPriceCalculatorTest extends TestCase
{
    public function test_characterize_dhaka_region_pricing(): void
    {
        $calc = new LegacyPriceCalculator();
        $items = [['name' => 'Rice', 'price' => 60, 'qty' => 5]];

        $result = $calc->calculate($items, 'dhaka');

        // প্রথমে রান করে দেখুন আসল আউটপুট কত,
        // তারপর সেই মান assert করুন
        $this->assertEquals(315, $result); // ৩১৫ আসলো — এটাই assert করি
    }

    public function test_characterize_empty_items(): void
    {
        $calc = new LegacyPriceCalculator();
        $result = $calc->calculate([], 'dhaka');
        $this->assertEquals(0, $result);
    }
}
```

#### কৌশল ২: Golden Master Testing

```javascript
// সম্পূর্ণ আউটপুট স্ন্যাপশট হিসেবে সেভ করি
const { LegacyReportGenerator } = require('../src/legacy');
const fs = require('fs');

describe('Golden Master - Legacy Report', () => {
  test('রিপোর্ট আউটপুট গোল্ডেন মাস্টারের সাথে মিলে', () => {
    const generator = new LegacyReportGenerator();
    const output = generator.generate(sampleData);

    // প্রথমবার: fs.writeFileSync('golden-master.txt', output);
    const goldenMaster = fs.readFileSync('golden-master.txt', 'utf8');
    expect(output).toBe(goldenMaster);
  });
});
```

#### Legacy Code-এ TDD ধাপ:

```
১. Characterization test লিখুন (বর্তমান আচরণ বুঝুন)
        │
        ▼
২. "Seam" খুঁজুন (যেখানে কোড আলাদা করা যায়)
        │
        ▼
৩. Extract Method / Extract Class করুন
        │
        ▼
৪. নতুন ফিচারের জন্য TDD শুরু করুন
        │
        ▼
৫. ধীরে ধীরে legacy কোড প্রতিস্থাপন করুন
```

---

### ৫. BDD (Behavior Driven Development)

BDD হলো TDD-এর একটি এক্সটেনশন যেখানে **Given/When/Then** ফরম্যাটে টেস্ট লেখা হয়। এটি ব্যবসায়িক ভাষায় (ubiquitous language) লেখা হয়।

#### PHP — Behat ব্যবহার করে:

```gherkin
# features/shopping_cart.feature

Feature: শপিং কার্ট
  একজন ক্রেতা হিসেবে
  আমি কার্টে পণ্য যোগ করতে চাই
  যাতে আমি সেগুলো একসাথে কিনতে পারি

  Scenario: কার্টে পণ্য যোগ করা
    Given আমার কার্ট খালি
    When আমি "Laptop" যোগ করি যার দাম ৭৫০০০ টাকা
    And আমি "Mouse" যোগ করি যার দাম ৫০০ টাকা পরিমাণ ২
    Then কার্টে ২টি আইটেম থাকবে
    And মোট মূল্য হবে ৭৬০০০ টাকা

  Scenario: ডিসকাউন্ট প্রয়োগ
    Given আমার কার্টে ১০০০০ টাকার পণ্য আছে
    When আমি ১০% ডিসকাউন্ট প্রয়োগ করি
    Then মোট মূল্য হবে ৯০০০ টাকা
```

```php
<?php
// features/bootstrap/CartContext.php

use Behat\Behat\Context\Context;
use PHPUnit\Framework\Assert;

class CartContext implements Context
{
    private ShoppingCart $cart;

    /** @Given আমার কার্ট খালি */
    public function emptyCart(): void
    {
        $this->cart = new ShoppingCart();
    }

    /** @When আমি :name যোগ করি যার দাম :price টাকা */
    public function addItem(string $name, int $price): void
    {
        $this->cart->addItem($name, $price, 1);
    }

    /** @Then মোট মূল্য হবে :total টাকা */
    public function assertTotal(int $total): void
    {
        Assert::assertEquals($total, $this->cart->calculateTotal());
    }
}
```

#### JavaScript — Cucumber ব্যবহার করে:

```gherkin
# features/password.feature

Feature: পাসওয়ার্ড যাচাই
  Scenario: দুর্বল পাসওয়ার্ড প্রত্যাখ্যান
    Given আমি একটি পাসওয়ার্ড ভ্যালিডেটর তৈরি করেছি
    When আমি "abc" পাসওয়ার্ড যাচাই করি
    Then ফলাফল অবৈধ হবে
    And ত্রুটি তালিকায় "পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে" থাকবে
```

```javascript
// features/step_definitions/passwordSteps.js

const { Given, When, Then } = require('@cucumber/cucumber');
const { PasswordValidator } = require('../../src/passwordValidator');
const assert = require('assert');

let validator, result;

Given('আমি একটি পাসওয়ার্ড ভ্যালিডেটর তৈরি করেছি', () => {
  validator = new PasswordValidator();
});

When('আমি {string} পাসওয়ার্ড যাচাই করি', (password) => {
  result = validator.validate(password);
});

Then('ফলাফল অবৈধ হবে', () => {
  assert.strictEqual(result.isValid, false);
});

Then('ত্রুটি তালিকায় {string} থাকবে', (errorMsg) => {
  assert.ok(result.errors.includes(errorMsg));
});
```

---

### ৬. ATDD (Acceptance Test Driven Development)

ATDD হলো TDD-এর একটি উচ্চ-স্তরের সংস্করণ যেখানে **ব্যবসায়িক প্রয়োজনীয়তা** থেকে সরাসরি acceptance test লেখা হয়।

```
       ATDD Pyramid
    ┌────────────────┐
    │  Acceptance     │  ← ATDD (ব্যবসায়িক ভাষায়)
    │  Tests          │
    ├────────────────┤
    │  Integration    │  ← কম্পোনেন্ট ইন্টিগ্রেশন
    │  Tests          │
    ├────────────────┤
    │  Unit Tests     │  ← TDD (ডেভেলপার টেস্ট)
    │  (most tests)   │
    └────────────────┘
```

```php
// ATDD: Acceptance test প্রথমে লিখুন
class UserRegistrationAcceptanceTest extends TestCase
{
    public function test_user_can_register_and_login(): void
    {
        $app = new Application();

        // Acceptance criteria অনুযায়ী
        $response = $app->post('/register', [
            'name' => 'রহিম',
            'email' => 'rahim@example.com',
            'password' => 'Str0ng!Pass',
        ]);
        $this->assertEquals(201, $response->status());

        $loginResponse = $app->post('/login', [
            'email' => 'rahim@example.com',
            'password' => 'Str0ng!Pass',
        ]);
        $this->assertEquals(200, $loginResponse->status());
        $this->assertNotEmpty($loginResponse->json('token'));
    }
}
```

**ATDD + TDD ওয়ার্কফ্লো:**

```
১. Product Owner acceptance criteria দেন
        │
        ▼
২. ATDD: Acceptance test লিখুন (🔴 ফেইল)
        │
        ▼
৩. TDD সাইকেল শুরু করুন:
   │  ┌──▶ 🔴 Unit test লিখুন
   │  │    🟢 ন্যূনতম কোড
   │  │    🔵 রিফ্যাক্টর
   │  └──── পুনরাবৃত্তি
   │
   ▼
৪. Acceptance test পাস (🟢)
        │
        ▼
৫. পরবর্তী acceptance criteria
```

---

### ৭. TDD in API Development (API-First TDD)

API ডেভেলপমেন্টে TDD অত্যন্ত কার্যকর। প্রথমে API contract (request/response) ডিজাইন করুন, তারপর TDD করুন।

#### PHP (Laravel) — API TDD:

```php
class BkashTransferApiTest extends TestCase
{
    // 🔴 Red: সফল ট্রান্সফার
    public function test_successful_transfer_returns_201(): void
    {
        $payload = [
            'sender'   => '01712345678',
            'receiver' => '01898765432',
            'amount'   => 500.00,
            'pin'      => '12345',
        ];

        $response = $this->postJson('/api/v1/transfer', $payload);

        $response->assertStatus(201)
                 ->assertJsonStructure([
                     'transaction_id',
                     'status',
                     'amount',
                     'fee',
                     'timestamp',
                 ]);
    }

    // 🔴 Red: অপর্যাপ্ত ব্যালেন্স
    public function test_insufficient_balance_returns_422(): void
    {
        $payload = [
            'sender'   => '01712345678',
            'receiver' => '01898765432',
            'amount'   => 999999.00,
            'pin'      => '12345',
        ];

        $response = $this->postJson('/api/v1/transfer', $payload);

        $response->assertStatus(422)
                 ->assertJson([
                     'error' => 'অপর্যাপ্ত ব্যালেন্স',
                 ]);
    }

    // 🔴 Red: ভুল পিন
    public function test_wrong_pin_returns_401(): void
    {
        $payload = [
            'sender'   => '01712345678',
            'receiver' => '01898765432',
            'amount'   => 500.00,
            'pin'      => '00000',
        ];

        $response = $this->postJson('/api/v1/transfer', $payload);

        $response->assertStatus(401)
                 ->assertJson(['error' => 'ভুল পিন']);
    }

    // 🔴 Red: ভ্যালিডেশন — সর্বনিম্ন পরিমাণ
    public function test_minimum_transfer_amount_validation(): void
    {
        $payload = [
            'sender'   => '01712345678',
            'receiver' => '01898765432',
            'amount'   => 5.00, // সর্বনিম্ন ১০ টাকা
            'pin'      => '12345',
        ];

        $response = $this->postJson('/api/v1/transfer', $payload);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['amount']);
    }
}
```

#### JavaScript (Express + Supertest) — API TDD:

```javascript
// __tests__/api/transfer.test.js

const request = require('supertest');
const app = require('../../src/app');

describe('POST /api/v1/transfer', () => {
  test('সফল ট্রান্সফারে 201 রিটার্ন করে', async () => {
    const res = await request(app)
      .post('/api/v1/transfer')
      .send({
        sender: '01712345678',
        receiver: '01898765432',
        amount: 500,
        pin: '12345',
      });

    expect(res.status).toBe(201);
    expect(res.body).toHaveProperty('transaction_id');
    expect(res.body.status).toBe('completed');
    expect(res.body.fee).toBeGreaterThan(0);
  });

  test('অবৈধ ফোন নম্বরে 422 রিটার্ন করে', async () => {
    const res = await request(app)
      .post('/api/v1/transfer')
      .send({
        sender: '123',
        receiver: '01898765432',
        amount: 500,
        pin: '12345',
      });

    expect(res.status).toBe(422);
    expect(res.body.errors).toBeDefined();
  });

  test('দৈনিক লিমিট অতিক্রম করলে 429 রিটার্ন করে', async () => {
    // দৈনিক সর্বোচ্চ ২৫,০০০ টাকা
    const res = await request(app)
      .post('/api/v1/transfer')
      .send({
        sender: '01712345678',
        receiver: '01898765432',
        amount: 30000,
        pin: '12345',
      });

    expect(res.status).toBe(429);
    expect(res.body.error).toContain('দৈনিক লিমিট');
  });
});
```

---

### ৮. TDD কখন করবেন না (When NOT to do TDD)

TDD সর্বজনীন সমাধান নয়। কিছু ক্ষেত্রে TDD কম কার্যকর বা সময়ের অপচয়:

```
┌──────────────────────────────────────────────────────────┐
│              TDD এড়িয়ে যাওয়ার ক্ষেত্রসমূহ               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ❌ Prototype / Spike কোড                                │
│     → আপনি সমস্যা বুঝতে কোড লিখছেন, সমাধান করতে নয়    │
│     → এই কোড ফেলে দেওয়া হবে                             │
│                                                          │
│  ❌ UI/View Layer (পরিবর্তনশীল HTML/CSS)                 │
│     → Visual regression tools ভালো কাজ করে              │
│     → TDD এখানে ভঙ্গুর টেস্ট তৈরি করে                   │
│                                                          │
│  ❌ Third-party API Integration                          │
│     → Integration test বা contract test ব্যবহার করুন     │
│     → ইউনিট টেস্টে মক করলে বাস্তব API পরিবর্তনে ধরা    │
│       পড়বে না                                            │
│                                                          │
│  ❌ Trivial কোড (getters/setters, config)                │
│     → যেখানে লজিক নেই সেখানে TDD অপচয়                  │
│                                                          │
│  ❌ একেবারে নতুন ডোমেইন শেখার সময়                       │
│     → আগে ডোমেইন বুঝুন, তারপর TDD শুরু করুন             │
│                                                          │
│  ❌ ডেটাবেস মাইগ্রেশন / স্ক্রিপ্ট                        │
│     → Integration test যথেষ্ট                            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**তবে মনে রাখবেন**: "TDD কঠিন" মানে "TDD করবেন না" নয়। TDD কঠিন মনে হলে সাধারণত কোড ডিজাইনে সমস্যা আছে।

---

### ৯. TDD Kata Exercises

TDD অনুশীলনের জন্য Kata হলো সবচেয়ে ভালো উপায়। নিচে তিনটি ক্লাসিক Kata:

#### Kata ১: FizzBuzz (PHP)

```php
class FizzBuzzTest extends TestCase
{
    private FizzBuzz $fb;

    protected function setUp(): void
    {
        $this->fb = new FizzBuzz();
    }

    public function test_returns_number_as_string(): void
    {
        $this->assertEquals('1', $this->fb->convert(1));
        $this->assertEquals('2', $this->fb->convert(2));
    }

    public function test_returns_fizz_for_multiples_of_3(): void
    {
        $this->assertEquals('Fizz', $this->fb->convert(3));
        $this->assertEquals('Fizz', $this->fb->convert(9));
    }

    public function test_returns_buzz_for_multiples_of_5(): void
    {
        $this->assertEquals('Buzz', $this->fb->convert(5));
        $this->assertEquals('Buzz', $this->fb->convert(20));
    }

    public function test_returns_fizzbuzz_for_multiples_of_15(): void
    {
        $this->assertEquals('FizzBuzz', $this->fb->convert(15));
        $this->assertEquals('FizzBuzz', $this->fb->convert(30));
    }
}

class FizzBuzz
{
    public function convert(int $number): string
    {
        if ($number % 15 === 0) return 'FizzBuzz';
        if ($number % 3 === 0)  return 'Fizz';
        if ($number % 5 === 0)  return 'Buzz';
        return (string) $number;
    }
}
```

#### Kata ২: Roman Numerals (JavaScript)

```javascript
describe('RomanNumerals', () => {
  const converter = new RomanNumeralConverter();

  // TDD সাইকেল ১: সহজ সংখ্যা
  test.each([
    [1, 'I'], [2, 'II'], [3, 'III'],
  ])('%i → %s', (num, roman) => {
    expect(converter.toRoman(num)).toBe(roman);
  });

  // TDD সাইকেল ২: subtractive notation
  test.each([
    [4, 'IV'], [9, 'IX'], [40, 'XL'], [90, 'XC'],
  ])('%i → %s (subtractive)', (num, roman) => {
    expect(converter.toRoman(num)).toBe(roman);
  });

  // TDD সাইকেল ৩: বড় সংখ্যা
  test.each([
    [1994, 'MCMXCIV'], [2024, 'MMXXIV'], [3999, 'MMMCMXCIX'],
  ])('%i → %s (complex)', (num, roman) => {
    expect(converter.toRoman(num)).toBe(roman);
  });
});

class RomanNumeralConverter {
  #map = [
    [1000, 'M'], [900, 'CM'], [500, 'D'], [400, 'CD'],
    [100, 'C'],  [90, 'XC'],  [50, 'L'],  [40, 'XL'],
    [10, 'X'],   [9, 'IX'],   [5, 'V'],   [4, 'IV'],
    [1, 'I'],
  ];

  toRoman(num) {
    let result = '';
    for (const [value, numeral] of this.#map) {
      while (num >= value) {
        result += numeral;
        num -= value;
      }
    }
    return result;
  }
}
```

#### Kata ৩: Bowling Game (PHP)

```php
class BowlingGameTest extends TestCase
{
    private BowlingGame $game;

    protected function setUp(): void
    {
        $this->game = new BowlingGame();
    }

    private function rollMany(int $rolls, int $pins): void
    {
        for ($i = 0; $i < $rolls; $i++) {
            $this->game->roll($pins);
        }
    }

    public function test_gutter_game_scores_zero(): void
    {
        $this->rollMany(20, 0);
        $this->assertEquals(0, $this->game->score());
    }

    public function test_all_ones_scores_twenty(): void
    {
        $this->rollMany(20, 1);
        $this->assertEquals(20, $this->game->score());
    }

    public function test_spare_adds_next_roll_bonus(): void
    {
        $this->game->roll(5);
        $this->game->roll(5); // spare
        $this->game->roll(3);
        $this->rollMany(17, 0);
        $this->assertEquals(16, $this->game->score());
    }

    public function test_strike_adds_next_two_rolls_bonus(): void
    {
        $this->game->roll(10); // strike
        $this->game->roll(3);
        $this->game->roll(4);
        $this->rollMany(16, 0);
        $this->assertEquals(24, $this->game->score());
    }

    public function test_perfect_game_scores_300(): void
    {
        $this->rollMany(12, 10);
        $this->assertEquals(300, $this->game->score());
    }
}

class BowlingGame
{
    private array $rolls = [];

    public function roll(int $pins): void
    {
        $this->rolls[] = $pins;
    }

    public function score(): int
    {
        $score = 0;
        $rollIndex = 0;

        for ($frame = 0; $frame < 10; $frame++) {
            if ($this->isStrike($rollIndex)) {
                $score += 10 + $this->strikeBonus($rollIndex);
                $rollIndex++;
            } elseif ($this->isSpare($rollIndex)) {
                $score += 10 + $this->spareBonus($rollIndex);
                $rollIndex += 2;
            } else {
                $score += $this->frameScore($rollIndex);
                $rollIndex += 2;
            }
        }

        return $score;
    }

    private function isStrike(int $i): bool
    {
        return ($this->rolls[$i] ?? 0) === 10;
    }

    private function isSpare(int $i): bool
    {
        return (($this->rolls[$i] ?? 0) + ($this->rolls[$i + 1] ?? 0)) === 10;
    }

    private function strikeBonus(int $i): int
    {
        return ($this->rolls[$i + 1] ?? 0) + ($this->rolls[$i + 2] ?? 0);
    }

    private function spareBonus(int $i): int
    {
        return $this->rolls[$i + 2] ?? 0;
    }

    private function frameScore(int $i): int
    {
        return ($this->rolls[$i] ?? 0) + ($this->rolls[$i + 1] ?? 0);
    }
}
```

---

## 🇧🇩 বাংলাদেশ কনটেক্সট: bKash ট্রান্সফার ভ্যালিডেশন TDD দিয়ে বিল্ড

বাস্তব উদাহরণ হিসেবে আমরা bKash-এর মতো মোবাইল ব্যাংকিং ট্রান্সফার ভ্যালিডেশন TDD দিয়ে বানাবো।

### PHP: BkashTransferValidator

```php
<?php
// tests/BkashTransferValidatorTest.php

class BkashTransferValidatorTest extends TestCase
{
    private BkashTransferValidator $validator;

    protected function setUp(): void
    {
        $this->validator = new BkashTransferValidator();
    }

    // 🔴🟢 সাইকেল ১: ফোন নম্বর ভ্যালিডেশন
    public function test_rejects_invalid_phone_number(): void
    {
        $result = $this->validator->validate([
            'sender'   => '12345',
            'receiver' => '01898765432',
            'amount'   => 100,
        ]);
        $this->assertFalse($result->isValid);
        $this->assertContains(
            'বৈধ বাংলাদেশি মোবাইল নম্বর দিন (01XXXXXXXXX)',
            $result->errors
        );
    }

    public function test_accepts_valid_bd_phone_numbers(): void
    {
        $result = $this->validator->validate([
            'sender'   => '01712345678',
            'receiver' => '01898765432',
            'amount'   => 100,
        ]);
        $this->assertNotContains(
            'বৈধ বাংলাদেশি মোবাইল নম্বর দিন (01XXXXXXXXX)',
            $result->errors
        );
    }

    // 🔴🟢 সাইকেল ২: পরিমাণ ভ্যালিডেশন
    public function test_rejects_amount_below_minimum(): void
    {
        $result = $this->validator->validate([
            'sender' => '01712345678', 'receiver' => '01898765432',
            'amount' => 5, // সর্বনিম্ন ১০ টাকা
        ]);
        $this->assertContains('সর্বনিম্ন ট্রান্সফার পরিমাণ ১০ টাকা', $result->errors);
    }

    public function test_rejects_amount_above_maximum(): void
    {
        $result = $this->validator->validate([
            'sender' => '01712345678', 'receiver' => '01898765432',
            'amount' => 30000, // সর্বোচ্চ ২৫,০০০ টাকা
        ]);
        $this->assertContains(
            'একবারে সর্বোচ্চ ২৫,০০০ টাকা ট্রান্সফার করা যায়',
            $result->errors
        );
    }

    // 🔴🟢 সাইকেল ৩: নিজের নম্বরে ট্রান্সফার
    public function test_rejects_self_transfer(): void
    {
        $result = $this->validator->validate([
            'sender' => '01712345678', 'receiver' => '01712345678',
            'amount' => 100,
        ]);
        $this->assertContains('নিজের নম্বরে টাকা পাঠানো যায় না', $result->errors);
    }

    // 🔴🟢 সাইকেল ৪: ফি হিসাব
    public function test_calculates_fee_for_small_amount(): void
    {
        $result = $this->validator->validate([
            'sender' => '01712345678', 'receiver' => '01898765432',
            'amount' => 100,
        ]);
        $this->assertEquals(5, $result->fee);
    }

    public function test_calculates_fee_for_large_amount(): void
    {
        $result = $this->validator->validate([
            'sender' => '01712345678', 'receiver' => '01898765432',
            'amount' => 10000,
        ]);
        $this->assertEquals(25, $result->fee);
    }

    // 🔴🟢 সাইকেল ৫: দৈনিক লিমিট
    public function test_rejects_if_daily_limit_exceeded(): void
    {
        $todayTransfers = 20000; // আজকে ইতোমধ্যে ২০,০০০ পাঠিয়েছে
        $validator = new BkashTransferValidator($todayTransfers);

        $result = $validator->validate([
            'sender' => '01712345678', 'receiver' => '01898765432',
            'amount' => 10000, // মোট হবে ৩০,০০০ > দৈনিক লিমিট ২৫,০০০
        ]);
        $this->assertContains('দৈনিক সর্বোচ্চ ২৫,০০০ টাকা ট্রান্সফার করা যায়', $result->errors);
    }
}
```

```php
<?php
// src/BkashTransferValidator.php

class TransferValidationResult
{
    public function __construct(
        public readonly bool $isValid,
        public readonly array $errors,
        public readonly float $fee = 0,
        public readonly float $totalAmount = 0,
    ) {}
}

class BkashTransferValidator
{
    private const MIN_AMOUNT = 10;
    private const MAX_AMOUNT = 25000;
    private const DAILY_LIMIT = 25000;

    private const FEE_STRUCTURE = [
        ['min' => 10,    'max' => 500,   'fee' => 5],
        ['min' => 501,   'max' => 1000,  'fee' => 10],
        ['min' => 1001,  'max' => 5000,  'fee' => 15],
        ['min' => 5001,  'max' => 10000, 'fee' => 25],
        ['min' => 10001, 'max' => 25000, 'fee' => 40],
    ];

    public function __construct(
        private readonly float $todayTotalTransferred = 0,
    ) {}

    public function validate(array $data): TransferValidationResult
    {
        $errors = [];
        $sender   = $data['sender'] ?? '';
        $receiver = $data['receiver'] ?? '';
        $amount   = $data['amount'] ?? 0;

        // ফোন নম্বর ভ্যালিডেশন
        $phonePattern = '/^01[3-9]\d{8}$/';
        if (!preg_match($phonePattern, $sender) || !preg_match($phonePattern, $receiver)) {
            $errors[] = 'বৈধ বাংলাদেশি মোবাইল নম্বর দিন (01XXXXXXXXX)';
        }

        // নিজের নম্বরে ট্রান্সফার
        if ($sender === $receiver) {
            $errors[] = 'নিজের নম্বরে টাকা পাঠানো যায় না';
        }

        // পরিমাণ ভ্যালিডেশন
        if ($amount < self::MIN_AMOUNT) {
            $errors[] = 'সর্বনিম্ন ট্রান্সফার পরিমাণ ১০ টাকা';
        }
        if ($amount > self::MAX_AMOUNT) {
            $errors[] = 'একবারে সর্বোচ্চ ২৫,০০০ টাকা ট্রান্সফার করা যায়';
        }

        // দৈনিক লিমিট
        if (($this->todayTotalTransferred + $amount) > self::DAILY_LIMIT) {
            $errors[] = 'দৈনিক সর্বোচ্চ ২৫,০০০ টাকা ট্রান্সফার করা যায়';
        }

        // ফি হিসাব
        $fee = $this->calculateFee($amount);

        return new TransferValidationResult(
            isValid: empty($errors),
            errors: $errors,
            fee: $fee,
            totalAmount: $amount + $fee,
        );
    }

    private function calculateFee(float $amount): float
    {
        foreach (self::FEE_STRUCTURE as $tier) {
            if ($amount >= $tier['min'] && $amount <= $tier['max']) {
                return $tier['fee'];
            }
        }
        return 0;
    }
}
```

### JavaScript: bKash টেস্ট (Jest)

```javascript
// __tests__/bkashValidator.test.js

const { BkashTransferValidator } = require('../src/bkashValidator');

describe('BkashTransferValidator', () => {
  let validator;

  beforeEach(() => {
    validator = new BkashTransferValidator();
  });

  describe('ফোন নম্বর যাচাই', () => {
    test('অবৈধ নম্বর প্রত্যাখ্যান করে', () => {
      const result = validator.validate({
        sender: '12345', receiver: '01898765432', amount: 100,
      });
      expect(result.errors).toContain('বৈধ মোবাইল নম্বর দিন');
    });

    test('GP, Robi, Banglalink, Teletalk নম্বর গ্রহণ করে', () => {
      ['017', '018', '019', '016', '015', '013'].forEach((prefix) => {
        const result = validator.validate({
          sender: `${prefix}12345678`,
          receiver: '01898765432',
          amount: 100,
        });
        expect(result.errors).not.toContain('বৈধ মোবাইল নম্বর দিন');
      });
    });
  });

  describe('পরিমাণ যাচাই', () => {
    test('সর্বনিম্ন ১০ টাকা না হলে প্রত্যাখ্যান', () => {
      const result = validator.validate({
        sender: '01712345678', receiver: '01898765432', amount: 5,
      });
      expect(result.isValid).toBe(false);
    });

    test('সর্বোচ্চ ২৫,০০০ টাকার বেশি হলে প্রত্যাখ্যান', () => {
      const result = validator.validate({
        sender: '01712345678', receiver: '01898765432', amount: 30000,
      });
      expect(result.isValid).toBe(false);
    });
  });

  describe('ফি হিসাব', () => {
    test.each([
      [50, 5], [500, 5], [750, 10], [3000, 15], [8000, 25], [20000, 40],
    ])('%i টাকায় ফি %i টাকা', (amount, expectedFee) => {
      const result = validator.validate({
        sender: '01712345678', receiver: '01898765432', amount,
      });
      expect(result.fee).toBe(expectedFee);
    });
  });
});
```

---

## ✅ TDD-এর সুবিধাসমূহ

| # | সুবিধা | ব্যাখ্যা |
|---|--------|----------|
| ১ | **আত্মবিশ্বাসী রিফ্যাক্টরিং** | টেস্ট সুইট থাকায় যেকোনো সময় নির্ভয়ে কোড পরিবর্তন করা যায় |
| ২ | **ভালো ডিজাইন** | টেস্টযোগ্য কোড লিখতে বাধ্য হওয়ায় loosely coupled ডিজাইন আসে |
| ৩ | **জীবন্ত ডকুমেন্টেশন** | টেস্টগুলো কোডের আচরণ ডকুমেন্ট করে, যা সবসময় আপডেট থাকে |
| ৪ | **কম ডিবাগিং** | বাগ পাওয়া যায় মিনিটের মধ্যে, ঘণ্টার পর ঘণ্টা ডিবাগ করতে হয় না |
| ৫ | **ফিডব্যাক লুপ দ্রুত** | প্রতিটি পরিবর্তনের ফলাফল সাথে সাথে জানা যায় |
| ৬ | **Regression প্রতিরোধ** | নতুন কোড পুরনো ফিচার ভাঙছে কিনা তা স্বয়ংক্রিয়ভাবে ধরা পড়ে |
| ৭ | **সহজ কোড রিভিউ** | টেস্ট দেখেই বোঝা যায় কোড কী করে |
| ৮ | **মানসিক শান্তি** | প্রোডাকশনে ডিপ্লয় করার সময় ভয় কম থাকে |

---

## ❌ TDD-এর চ্যালেঞ্জসমূহ

| # | চ্যালেঞ্জ | সমাধান |
|---|-----------|--------|
| ১ | **শেখার বক্ররেখা খাড়া** | Kata অনুশীলন করুন, ছোট প্রজেক্ট দিয়ে শুরু করুন |
| ২ | **প্রথম দিকে ধীর গতি** | ২-৩ সপ্তাহ পর গতি বাড়ে, ডিবাগিং সময় কমে |
| ৩ | **টেস্ট মেইনটেন্যান্স** | টেস্ট ভালোভাবে organize করুন, DRY রাখুন |
| ৪ | **Over-mocking** | সত্যিকারের অবজেক্ট ব্যবহার করুন যেখানে সম্ভব |
| ৫ | **ম্যানেজমেন্ট কনভিন্স করা কঠিন** | বাগ কমে যাওয়ার মেট্রিক্স দেখান |
| ৬ | **Legacy কোডে প্রয়োগ কঠিন** | Characterization test দিয়ে শুরু করুন |
| ৭ | **ভুলভাবে TDD করলে ক্ষতিকর** | TDD community-তে যোগ দিন, mentorship নিন |

---

## ⚠️ সাধারণ ভুলসমূহ (Common Mistakes)

### ১. একবারে বড় ধাপ নেওয়া
```
❌ ভুল: একটি টেস্টে পুরো ফিচার কভার করার চেষ্টা
✅ সঠিক: প্রতিটি টেস্ট একটি ছোট আচরণ পরীক্ষা করে
```

### ২. Red ধাপ এড়িয়ে যাওয়া
```
❌ ভুল: প্রোডাকশন কোড আগে লিখে তারপর টেস্ট লেখা
✅ সঠিক: সবসময় 🔴 Red আগে — টেস্ট ফেইল হতে দেখুন!
```

### ৩. Refactor ধাপ ভুলে যাওয়া
```
❌ ভুল: Green হলেই পরবর্তী ফিচারে চলে যাওয়া
✅ সঠিক: প্রতিটি Green-এর পর থামুন, কোড পরিষ্কার করুন
```

### ৪. Implementation টেস্ট করা, Behavior নয়
```php
// ❌ ভুল: implementation-এ আবদ্ধ
$this->assertEquals(3, count($cart->items)); // private property access

// ✅ সঠিক: behavior টেস্ট করুন
$this->assertEquals(3, $cart->getItemCount());
$this->assertEquals(1500, $cart->calculateTotal());
```

### ৫. টেস্ট নাম অস্পষ্ট
```php
// ❌ ভুল
public function testCart(): void { ... }

// ✅ সঠিক
public function test_empty_cart_has_zero_total(): void { ... }
public function test_applying_expired_coupon_throws_exception(): void { ... }
```

### ৬. একটি টেস্টে অনেক assertion
```javascript
// ❌ ভুল: একটি টেস্ট ফেইল হলে বুঝতে কষ্ট হয়
test('সব কিছু কাজ করে', () => {
  expect(result.isValid).toBe(true);
  expect(result.fee).toBe(5);
  expect(result.errors).toHaveLength(0);
  expect(result.strength).toBe('strong');
  // ... ২০টি assertion
});

// ✅ সঠিক: প্রতিটি আচরণ আলাদা টেস্টে
test('বৈধ ট্রান্সফার সফল হয়', () => {
  expect(result.isValid).toBe(true);
});

test('১০০ টাকায় ফি ৫ টাকা', () => {
  expect(result.fee).toBe(5);
});
```

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                      TDD সারসংক্ষেপ                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔴 Red    → ফেইলিং টেস্ট লিখুন (আচরণ বর্ণনা করুন)            │
│  🟢 Green  → ন্যূনতম কোড লিখে পাস করান                         │
│  🔵 Refactor → কোড পরিষ্কার করুন, সব টেস্ট পাস রাখুন           │
│                                                                 │
│  📏 আইন: ফেইলিং টেস্ট ছাড়া কোনো প্রোডাকশন কোড নয়             │
│  ⏱️ সাইকেল: ১-১০ মিনিট (সংক্ষিপ্ত রাখুন)                      │
│  🏗️ ডিজাইন: TDD থেকে ভালো ডিজাইন "emerge" করে                 │
│                                                                 │
│  Inside-Out: ছোট ইউনিট → বড় ফিচার (classicist)                │
│  Outside-In: বড় ফিচার → ছোট ইউনিট (mockist)                   │
│                                                                 │
│  BDD: Given/When/Then ব্যবসায়িক ভাষায়                          │
│  ATDD: Acceptance test → TDD সাইকেল → Acceptance pass           │
│                                                                 │
│  Legacy Code: Characterization test → Seam → Extract → TDD      │
│                                                                 │
│  ❌ এড়িয়ে যান: Prototype, UI, Trivial code                     │
│  ✅ ব্যবহার করুন: Business logic, API, Domain rules             │
│                                                                 │
│  🏋️ অনুশীলন: FizzBuzz → Roman Numerals → Bowling Game           │
│             → String Calculator → Bank Account Kata              │
│                                                                 │
│  "TDD শুধু টেস্টিং নয়, এটি একটি ডিজাইন টুল।"                  │
│                          — Kent Beck                             │
└─────────────────────────────────────────────────────────────────┘
```

---

> 📚 **আরও পড়ুন:**
> - "Test-Driven Development: By Example" — Kent Beck
> - "Growing Object-Oriented Software, Guided by Tests" — Steve Freeman & Nat Pryce
> - "Working Effectively with Legacy Code" — Michael Feathers
> - "The Art of Unit Testing" — Roy Osherove

---

*TDD আয়ত্ত করতে সময় লাগে — প্রতিদিন একটি Kata অনুশীলন করুন, এবং ৩০ দিনের মধ্যে TDD আপনার দ্বিতীয় প্রকৃতি হয়ে যাবে। 🚀*
