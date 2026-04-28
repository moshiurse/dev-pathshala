# 🧬 মিউটেশন টেস্টিং (Mutation Testing) — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা ও মূল ধারণা

**মিউটেশন টেস্টিং** হলো একটি উন্নত টেস্টিং কৌশল যেখানে সোর্স কোডে ইচ্ছাকৃতভাবে ছোট ছোট পরিবর্তন (mutation) করা হয় এবং দেখা হয় বিদ্যমান টেস্ট সুইট সেই পরিবর্তনগুলো ধরতে পারে কিনা। এটি আপনার টেস্টের **গুণগত মান** (quality) পরিমাপ করে — শুধু code coverage নয়।

### Code Coverage-এর সীমাবদ্ধতা

```
╔══════════════════════════════════════════════════════════════════════════╗
║              Code Coverage vs Mutation Testing                           ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  Code Coverage (পরিমাণগত):                                              ║
║  "কতটুকু কোড execute হয়েছে?"                                           ║
║  • ১০০% coverage থাকলেও assertion না থাকতে পারে                        ║
║  • কোড চলেছে, কিন্তু সঠিকভাবে যাচাই হয়েছে কিনা জানা যায় না           ║
║                                                                          ║
║  Mutation Testing (গুণগত):                                               ║
║  "টেস্টগুলো কতটা কার্যকর?"                                              ║
║  • কোড পরিবর্তন করলে টেস্ট ফেইল করে কিনা যাচাই করে                    ║
║  • দুর্বল assertion ও অপ্রয়োজনীয় টেস্ট শনাক্ত করে                     ║
║                                                                          ║
║  উদাহরণ — এই টেস্ট ১০০% coverage দেয় কিন্তু কিছুই যাচাই করে না:       ║
║  function test() {                                                       ║
║      $result = calculateDiscount(100, 20);                               ║
║      // কোনো assertion নেই! — mutation testing এটা ধরবে                 ║
║  }                                                                       ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### 🔑 মূল পরিভাষা

| পরিভাষা | বর্ণনা |
|---------|--------|
| **Mutant** | সোর্স কোডের একটি পরিবর্তিত সংস্করণ |
| **Mutator/Mutation Operator** | যে নিয়মে কোড পরিবর্তন করা হয় (যেমন: `+` → `-`) |
| **Killed Mutant** | যে mutant-কে টেস্ট ধরতে পেরেছে (টেস্ট ফেইল হয়েছে) ✅ |
| **Survived Mutant** | যে mutant-কে টেস্ট ধরতে পারেনি (টেস্ট পাস হয়ে গেছে) ❌ |
| **Equivalent Mutant** | যে mutation কোডের আচরণ পরিবর্তন করে না (false positive) |
| **Mutation Score** | `(killed / total) × 100%` — যত বেশি তত ভালো |
| **Timeout Mutant** | যে mutant-এ টেস্ট timeout হয়ে গেছে (সাধারণত killed ধরা হয়) |

### সাধারণ Mutators (পরিবর্তনকারী)

```
╔══════════════════════════════════════════════════════════════════╗
║  Mutator ক্যাটেগরি       ║  আসল কোড      ║  Mutated কোড       ║
╠══════════════════════════════════════════════════════════════════╣
║  Arithmetic               ║  $a + $b       ║  $a - $b           ║
║  Arithmetic               ║  $a * $b       ║  $a / $b           ║
║  Comparison               ║  $a > $b       ║  $a >= $b          ║
║  Comparison               ║  $a === $b     ║  $a !== $b         ║
║  Logical                  ║  $a && $b      ║  $a || $b          ║
║  Negation                 ║  !$condition   ║  $condition        ║
║  Return Value             ║  return $val;  ║  return null;      ║
║  Return Value             ║  return true;  ║  return false;     ║
║  Removal                  ║  method();     ║  (মুছে ফেলা)       ║
║  Increment                ║  $i++          ║  $i--              ║
║  Boundary                 ║  $a > 0        ║  $a >= 0           ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ বাস্তব উদাহরণ (Real-life Analogy)

ধরুন আপনি একটি **ফায়ার অ্যালার্ম সিস্টেম** টেস্ট করছেন:

- **Code Coverage** = শুধু দেখা যে অ্যালার্মটি বিদ্যুতের সাথে সংযুক্ত আছে কিনা (connected ≠ working)
- **Mutation Testing** = ইচ্ছাকৃতভাবে ছোট ধোঁয়া তৈরি করে দেখা অ্যালার্ম বাজে কিনা

যদি ধোঁয়া দিলেও অ্যালার্ম না বাজে — তাহলে অ্যালার্ম সিস্টেম (আপনার টেস্ট) দুর্বল! এটাই **survived mutant** — কোডে সমস্যা ঢোকানো হয়েছে কিন্তু টেস্ট ধরতে পারেনি।

আরেকটি উদাহরণ: **পরীক্ষার প্রশ্নপত্র যাচাই**:
- আপনি একটি পরীক্ষার প্রশ্ন তৈরি করেছেন (test suite)
- এখন ইচ্ছাকৃতভাবে কিছু ভুল উত্তর দিন (mutations)
- যদি আপনার প্রশ্নপত্র সেই ভুল উত্তর pass করে দেয় — প্রশ্নপত্র (test) দুর্বল!

---

## 🐘 PHP — Infection দিয়ে Mutation Testing

### ইনস্টলেশন ও কনফিগারেশন

```bash
# Infection ইনস্টল করুন
composer require --dev infection/infection

# প্রাথমিক কনফিগারেশন তৈরি
vendor/bin/infection --init
```

### infection.json5 কনফিগারেশন

```json5
{
    "$schema": "vendor/infection/infection/resources/schema.json",
    "source": {
        "directories": ["src"],
        "excludes": ["Migrations", "Kernel.php"]
    },
    "logs": {
        "text": "infection.log",
        "html": "infection.html",
        "summary": "infection-summary.log"
    },
    "mutators": {
        "@default": true,
        // নির্দিষ্ট mutator বন্ধ করতে
        "TrueValue": {
            "ignoreSourceCodeByRegex": [".*@infection-ignore-all.*"]
        }
    },
    "minMsi": 70,          // সর্বনিম্ন Mutation Score Indicator
    "minCoveredMsi": 80,   // covered code-এর সর্বনিম্ন MSI
    "testFramework": "phpunit",
    "phpUnit": {
        "configDir": "."
    }
}
```

### উদাহরণ — সোর্স কোড ও টেস্ট

```php
<?php
// src/PriceCalculator.php

declare(strict_types=1);

class PriceCalculator
{
    public function calculateDiscount(float $price, float $discountPercent): float
    {
        if ($discountPercent < 0 || $discountPercent > 100) {
            throw new InvalidArgumentException('Discount must be between 0 and 100');
        }

        if ($price <= 0) {
            throw new InvalidArgumentException('Price must be positive');
        }

        $discount = $price * ($discountPercent / 100);
        return round($price - $discount, 2);
    }

    public function calculateTax(float $price, float $taxRate): float
    {
        if ($taxRate < 0) {
            throw new InvalidArgumentException('Tax rate cannot be negative');
        }

        return round($price * (1 + $taxRate / 100), 2);
    }

    public function calculateFinalPrice(float $price, float $discount, float $taxRate): float
    {
        $discountedPrice = $this->calculateDiscount($price, $discount);
        return $this->calculateTax($discountedPrice, $taxRate);
    }
}
```

```php
<?php
// tests/PriceCalculatorTest.php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new PriceCalculator();
    }

    // ভালো টেস্ট — mutation ধরবে
    public function testCalculateDiscountWithValidValues(): void
    {
        $result = $this->calculator->calculateDiscount(100.0, 20.0);
        $this->assertSame(80.0, $result); // exact value assertion — mutant killed!
    }

    public function testCalculateDiscountWithZeroDiscount(): void
    {
        $result = $this->calculator->calculateDiscount(100.0, 0.0);
        $this->assertSame(100.0, $result); // boundary check
    }

    public function testCalculateDiscountWithFullDiscount(): void
    {
        $result = $this->calculator->calculateDiscount(100.0, 100.0);
        $this->assertSame(0.0, $result); // boundary check
    }

    public function testNegativeDiscountThrowsException(): void
    {
        $this->expectException(InvalidArgumentException::class);
        $this->calculator->calculateDiscount(100.0, -5.0);
    }

    public function testDiscountOver100ThrowsException(): void
    {
        $this->expectException(InvalidArgumentException::class);
        $this->calculator->calculateDiscount(100.0, 101.0);
    }

    public function testZeroPriceThrowsException(): void
    {
        $this->expectException(InvalidArgumentException::class);
        $this->calculator->calculateDiscount(0.0, 20.0);
    }

    public function testNegativePriceThrowsException(): void
    {
        $this->expectException(InvalidArgumentException::class);
        $this->calculator->calculateDiscount(-50.0, 20.0);
    }

    public function testCalculateTax(): void
    {
        $result = $this->calculator->calculateTax(100.0, 15.0);
        $this->assertSame(115.0, $result);
    }

    public function testCalculateTaxNegativeRateThrowsException(): void
    {
        $this->expectException(InvalidArgumentException::class);
        $this->calculator->calculateTax(100.0, -5.0);
    }

    public function testCalculateFinalPrice(): void
    {
        // price=200, discount=10%, tax=15%
        // discounted = 200 - 20 = 180
        // with tax = 180 * 1.15 = 207
        $result = $this->calculator->calculateFinalPrice(200.0, 10.0, 15.0);
        $this->assertSame(207.0, $result);
    }
}
```

### Infection চালানো ও ফলাফল ব্যাখ্যা

```bash
# সম্পূর্ণ mutation testing চালান
vendor/bin/infection --threads=4 --show-mutations

# নির্দিষ্ট ফাইলে চালান
vendor/bin/infection --filter=PriceCalculator --threads=4

# শুধু covered code-তে চালান (দ্রুত)
vendor/bin/infection --only-covered --min-msi=80

# CI-তে চালান — threshold-এর নিচে হলে fail
vendor/bin/infection --min-msi=70 --min-covered-msi=80 --threads=max
```

```
# আউটপুট উদাহরণ:
# ╔═══════════════════════════════════════════╗
# ║  Mutations: 25                            ║
# ║  Killed: 22        ✅                     ║
# ║  Survived: 2       ❌ (এগুলো ঠিক করুন)   ║
# ║  Errors: 0                                ║
# ║  Timeout: 1        ⏱️                     ║
# ║  MSI: 92.00%                              ║
# ║  Covered MSI: 95.65%                      ║
# ╚═══════════════════════════════════════════╝
```

---

## 🟨 JavaScript — Stryker দিয়ে Mutation Testing

### ইনস্টলেশন ও কনফিগারেশন

```bash
# Stryker ইনস্টল ও initialize
npm install --save-dev @stryker-mutator/core
npx stryker init
```

### stryker.config.mjs কনফিগারেশন

```javascript
// stryker.config.mjs
/** @type {import('@stryker-mutator/api/core').PartialStrykerOptions} */
const config = {
    packageManager: 'npm',
    reporters: ['html', 'clear-text', 'progress', 'dashboard'],
    testRunner: 'jest',                    // অথবা 'mocha', 'vitest'
    jest: {
        configFile: 'jest.config.js',
    },
    coverageAnalysis: 'perTest',           // প্রতিটি টেস্ট আলাদাভাবে বিশ্লেষণ
    mutate: [
        'src/**/*.js',
        '!src/**/*.test.js',               // টেস্ট ফাইল বাদ
        '!src/**/*.spec.js',
        '!src/index.js',                   // entry point বাদ
    ],
    thresholds: {
        high: 80,                          // ≥80% = সবুজ
        low: 60,                           // <60% = লাল
        break: 50,                         // <50% হলে CI fail
    },
    concurrency: 4,
    tempDirName: 'stryker-tmp',
    cleanTempDir: true,
};

export default config;
```

### উদাহরণ — সোর্স কোড ও টেস্ট

```javascript
// src/cart.js
class ShoppingCart {
    constructor() {
        this.items = [];
    }

    addItem(product, quantity) {
        if (quantity <= 0) {
            throw new Error('Quantity must be positive');
        }
        if (!product || !product.price) {
            throw new Error('Invalid product');
        }

        const existingItem = this.items.find(item => item.product.id === product.id);
        if (existingItem) {
            existingItem.quantity += quantity;
        } else {
            this.items.push({ product, quantity });
        }
    }

    removeItem(productId) {
        const index = this.items.findIndex(item => item.product.id === productId);
        if (index === -1) {
            throw new Error('Item not found in cart');
        }
        this.items.splice(index, 1);
    }

    getSubtotal() {
        return this.items.reduce((total, item) => {
            return total + (item.product.price * item.quantity);
        }, 0);
    }

    applyDiscount(percent) {
        if (percent < 0 || percent > 100) {
            throw new Error('Invalid discount percentage');
        }
        const subtotal = this.getSubtotal();
        return subtotal - (subtotal * percent / 100);
    }

    getItemCount() {
        return this.items.reduce((count, item) => count + item.quantity, 0);
    }

    isEmpty() {
        return this.items.length === 0;
    }
}

module.exports = ShoppingCart;
```

```javascript
// src/cart.test.js
const ShoppingCart = require('./cart');

describe('ShoppingCart — Mutation-Resistant Tests', () => {
    let cart;
    const product1 = { id: 1, name: 'Book', price: 500 };
    const product2 = { id: 2, name: 'Pen', price: 50 };

    beforeEach(() => {
        cart = new ShoppingCart();
    });

    describe('addItem', () => {
        it('should add a new item to the cart', () => {
            cart.addItem(product1, 2);
            expect(cart.items).toHaveLength(1);
            expect(cart.items[0].quantity).toBe(2);
            expect(cart.items[0].product.id).toBe(1);
        });

        it('should increase quantity for existing item', () => {
            cart.addItem(product1, 2);
            cart.addItem(product1, 3);
            expect(cart.items).toHaveLength(1);   // এখনো ১টি item
            expect(cart.items[0].quantity).toBe(5); // ২+৩ = ৫
        });

        it('should throw for zero quantity', () => {
            expect(() => cart.addItem(product1, 0)).toThrow('Quantity must be positive');
        });

        it('should throw for negative quantity', () => {
            expect(() => cart.addItem(product1, -1)).toThrow('Quantity must be positive');
        });

        it('should throw for invalid product', () => {
            expect(() => cart.addItem(null, 1)).toThrow('Invalid product');
            expect(() => cart.addItem({}, 1)).toThrow('Invalid product');
        });
    });

    describe('removeItem', () => {
        it('should remove item from cart', () => {
            cart.addItem(product1, 1);
            cart.addItem(product2, 1);
            cart.removeItem(1);
            expect(cart.items).toHaveLength(1);
            expect(cart.items[0].product.id).toBe(2);
        });

        it('should throw for non-existent item', () => {
            expect(() => cart.removeItem(999)).toThrow('Item not found');
        });
    });

    describe('getSubtotal', () => {
        it('should return 0 for empty cart', () => {
            expect(cart.getSubtotal()).toBe(0);
        });

        it('should calculate subtotal correctly', () => {
            cart.addItem(product1, 2); // 500 * 2 = 1000
            cart.addItem(product2, 3); // 50 * 3 = 150
            expect(cart.getSubtotal()).toBe(1150);
        });
    });

    describe('applyDiscount', () => {
        beforeEach(() => {
            cart.addItem(product1, 2); // subtotal = 1000
        });

        it('should apply percentage discount', () => {
            expect(cart.applyDiscount(10)).toBe(900); // 1000 - 100 = 900
        });

        it('should handle 0% discount', () => {
            expect(cart.applyDiscount(0)).toBe(1000);
        });

        it('should handle 100% discount', () => {
            expect(cart.applyDiscount(100)).toBe(0);
        });

        it('should throw for negative discount', () => {
            expect(() => cart.applyDiscount(-5)).toThrow('Invalid discount');
        });

        it('should throw for discount over 100', () => {
            expect(() => cart.applyDiscount(101)).toThrow('Invalid discount');
        });
    });

    describe('getItemCount', () => {
        it('should return 0 for empty cart', () => {
            expect(cart.getItemCount()).toBe(0);
        });

        it('should return total quantity of all items', () => {
            cart.addItem(product1, 2);
            cart.addItem(product2, 3);
            expect(cart.getItemCount()).toBe(5);
        });
    });

    describe('isEmpty', () => {
        it('should return true for empty cart', () => {
            expect(cart.isEmpty()).toBe(true);
        });

        it('should return false for non-empty cart', () => {
            cart.addItem(product1, 1);
            expect(cart.isEmpty()).toBe(false);
        });
    });
});
```

### Stryker চালানো

```bash
# Mutation testing চালান
npx stryker run

# নির্দিষ্ট ফাইলে চালান
npx stryker run --mutate "src/cart.js"
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন দরকার |
|-----------|----------|
| টেস্ট সুইটের কার্যকারিতা যাচাই করতে | Code coverage উচ্চ কিন্তু টেস্ট কতটা শক্তিশালী তা জানতে |
| Critical business logic-এ | পেমেন্ট, ডিসকাউন্ট, অথেন্টিকেশন — যেখানে বাগ ব্যয়বহুল |
| Legacy code-এ টেস্ট যোগ করার পর | নতুন টেস্টগুলো আসলেই কার্যকর কিনা যাচাই করতে |
| Code review-তে টেস্ট কোয়ালিটি নিশ্চিত করতে | PR-এ টেস্ট আছে, কিন্তু সেগুলো মানসম্মত কিনা |
| CI pipeline-এ quality gate হিসেবে | সর্বনিম্ন mutation score enforce করতে |

## ❌ কখন করবেন না / সীমাবদ্ধতা

| পরিস্থিতি | কারণ |
|-----------|------|
| প্রতিটি commit-এ সম্পূর্ণ codebase-এ | অনেক ধীর — নির্দিষ্ট changed files-এ চালান |
| UI/View layer-এ | Template/rendering কোডে mutation testing কম কার্যকর |
| Configuration/boilerplate কোডে | মূল্যবান তথ্য দেয় না |
| টেস্ট সুইট না থাকলে | আগে unit test লিখুন, তারপর mutation testing |
| Equivalent mutant বেশি হলে | কিছু কোড pattern-এ false positive বেশি হয়, সময় নষ্ট |
| খুব বড় codebase-এ একবারে | incremental/focused ভাবে চালান |

---

## 🧠 সিনিয়র ইঞ্জিনিয়ারদের জন্য পরামর্শ

১. **Incremental Mutation Testing করুন** — সম্পূর্ণ codebase-এ না চালিয়ে শুধু পরিবর্তিত ফাইলগুলোতে চালান। Infection-এ `--git-diff-filter=AM` ব্যবহার করুন।

২. **Survived Mutants বিশ্লেষণ করুন** — প্রতিটি survived mutant একটি সম্ভাব্য দুর্বল assertion বা missing test case নির্দেশ করে। এগুলো ঠিক করলে টেস্ট সুইট শক্তিশালী হবে।

৩. **MSI Threshold সেট করুন** — CI/CD-তে সর্বনিম্ন mutation score enforce করুন। PHP-তে `--min-msi=70`, Stryker-এ `thresholds.break: 50`।

৪. **Equivalent Mutants চিহ্নিত করুন** — কিছু mutation কোডের আচরণ পরিবর্তন করে না। এগুলো `@infection-ignore-all` বা Stryker-এর `// Stryker disable` দিয়ে বাদ দিন।

৫. **Code Coverage-র সাথে মিলিয়ে ব্যবহার করুন** — আগে coverage বাড়ান, তারপর mutation testing দিয়ে টেস্টের গুণগত মান বাড়ান। দুটি একে অপরের পরিপূরক।

> **মনে রাখবেন**: "১০০% code coverage মানে কোড bug-free নয় — mutation testing দিয়ে নিশ্চিত করুন যে আপনার টেস্ট আসলেই কিছু যাচাই করছে।" 🎯
