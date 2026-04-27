# 🧪 ইউনিট টেস্টিং (Unit Testing) — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা ও পরিচিতি

**ইউনিট টেস্টিং** হলো সফটওয়্যার টেস্টিং-এর সবচেয়ে মৌলিক স্তর, যেখানে কোডের ক্ষুদ্রতম পরীক্ষাযোগ্য একক (unit) — সাধারণত একটি ফাংশন, মেথড বা ক্লাস — আলাদাভাবে যাচাই করা হয়। এটি নিশ্চিত করে যে প্রতিটি কম্পোনেন্ট তার নির্ধারিত দায়িত্ব সঠিকভাবে পালন করছে।

### কেন ইউনিট টেস্ট দরকার?

১. **বাগ তাড়াতাড়ি ধরা পড়ে** — ডেভেলপমেন্ট পর্যায়েই সমস্যা শনাক্ত হয়, প্রোডাকশনে যাওয়ার আগেই
২. **রিফ্যাক্টরিং-এ সাহস দেয়** — টেস্ট থাকলে কোড পরিবর্তন করতে ভয় লাগে না, কারণ ভাঙলে টেস্ট ফেইল করবে
৩. **ডকুমেন্টেশন হিসেবে কাজ করে** — টেস্ট কোড পড়লেই বোঝা যায় কোডটি কী করার কথা
৪. **ডিজাইন উন্নত করে** — টেস্টযোগ্য কোড লিখতে গেলে স্বাভাবিকভাবেই loosely coupled ও SOLID কোড লেখা হয়
৫. **ডিবাগিং সময় কমায়** — কোন unit ভেঙেছে সেটা সরাসরি জানা যায়

### FIRST নীতিমালা — ভালো ইউনিট টেস্টের বৈশিষ্ট্য

```
╔══════════════════════════════════════════════════════════════════════╗
║                    FIRST Principles                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║  F — Fast (দ্রুত)                                                   ║
║      • মিলিসেকেন্ডে চলবে, সেকেন্ডেও না                            ║
║      • ডেটাবেস, নেটওয়ার্ক, ফাইল সিস্টেম — কিছুতেই নির্ভর করবে না ║
║                                                                      ║
║  I — Independent/Isolated (স্বাধীন)                                  ║
║      • একটি টেস্ট অন্য টেস্টের উপর নির্ভর করবে না                  ║
║      • যেকোনো ক্রমে চালালেও ফলাফল একই হবে                          ║
║                                                                      ║
║  R — Repeatable (পুনরাবৃত্তিযোগ্য)                                  ║
║      • যেকোনো environment-এ (local, CI, staging) একই ফলাফল দেবে    ║
║      • কোনো external state-এর উপর নির্ভরশীল নয়                     ║
║                                                                      ║
║  S — Self-validating (স্বয়ং-যাচাইকারী)                              ║
║      • টেস্ট নিজেই pass/fail বলে দেবে                               ║
║      • ম্যানুয়ালি আউটপুট চেক করার দরকার নেই                        ║
║                                                                      ║
║  T — Timely (সময়োচিত)                                               ║
║      • প্রোডাকশন কোডের সাথে সাথে বা আগেই (TDD) লেখা হবে           ║
║      • পরে লিখব বললে আর লেখা হয় না                                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 📊 টেস্টিং পিরামিড (Testing Pyramid)

টেস্টিং পিরামিড মাইক ‍কন (Mike Cohn) প্রবর্তন করেছিলেন। এটি দেখায় কোন ধরনের টেস্ট কত সংখ্যায় থাকা উচিত:

```
                          ╱╲
                         ╱  ╲
                        ╱ E2E╲          ← কম সংখ্যক, ধীর, ব্যয়বহুল
                       ╱ Tests╲           (Cypress, Selenium, Playwright)
                      ╱────────╲
                     ╱          ╲
                    ╱ Integration╲      ← মাঝামাঝি সংখ্যক
                   ╱    Tests     ╲       (API tests, DB integration)
                  ╱────────────────╲
                 ╱                  ╲
                ╱    Unit Tests      ╲  ← সবচেয়ে বেশি, দ্রুত, সস্তা
               ╱                      ╲   (PHPUnit, Jest, pytest)
              ╱────────────────────────╲
             ╱══════════════════════════╲

   গতি:      দ্রুততম ◄──────────────► ধীরতম
   সংখ্যা:    সবচেয়ে বেশি ◄──────► সবচেয়ে কম
   খরচ:      সবচেয়ে কম ◄──────────► সবচেয়ে বেশি
   আস্থা:     কম (isolated) ◄──────► বেশি (real scenario)
```

**বাংলাদেশী প্রেক্ষাপটে উদাহরণ:**
- **Unit Test:** bKash পেমেন্ট অ্যামাউন্ট ভ্যালিডেশন ফাংশন সঠিকভাবে কাজ করছে কিনা
- **Integration Test:** bKash API-র সাথে আমাদের সার্ভিস সঠিকভাবে communicate করছে কিনা
- **E2E Test:** ইউজার লগইন থেকে শুরু করে bKash দিয়ে পেমেন্ট সম্পন্ন করা পর্যন্ত পুরো flow

---

## 💻 PHPUnit Deep Dive

### সেটআপ ও কনফিগারেশন

`phpunit.xml` ফাইল প্রজেক্টের রুটে রাখা হয়। এটি PHPUnit-এর সমস্ত কনফিগারেশন নিয়ন্ত্রণ করে:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         failOnRisky="true"
         failOnWarning="true"
         stopOnFailure="false"
         executionOrder="random"
         cacheDirectory=".phpunit.cache">

    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>

    <source>
        <include>
            <directory>src</directory>
        </include>
        <exclude>
            <directory>src/Database/Migrations</directory>
        </exclude>
    </source>

    <coverage>
        <report>
            <html outputDirectory="reports/coverage"/>
            <clover outputFile="reports/clover.xml"/>
        </report>
    </coverage>

    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
    </php>
</phpunit>
```

### টেস্ট ক্লাসের গঠন ও PHP 8 Attributes

PHPUnit 10+ এ পুরনো annotation-ভিত্তিক ও নতুন PHP 8 attribute-ভিত্তিক — দুটোই ব্যবহার করা যায়:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Depends;
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\CoversClass;
use App\Services\BkashAmountValidator;

#[CoversClass(BkashAmountValidator::class)]
#[Group('payment')]
class BkashAmountValidatorTest extends TestCase
{
    private BkashAmountValidator $validator;

    // প্রতিটি টেস্টের আগে চলে — fresh state নিশ্চিত করে
    protected function setUp(): void
    {
        parent::setUp();
        $this->validator = new BkashAmountValidator();
    }

    // প্রতিটি টেস্টের পরে চলে — cleanup
    protected function tearDown(): void
    {
        parent::tearDown();
        // প্রয়োজনে resource মুক্ত করুন
    }

    // পুরনো annotation পদ্ধতি
    /** @test */
    public function it_accepts_valid_bkash_amount(): void
    {
        $result = $this->validator->validate(500.00);
        $this->assertTrue($result);
    }

    // নতুন PHP 8 Attribute পদ্ধতি (প্রস্তাবিত)
    #[Test]
    public function it_rejects_amount_below_minimum(): void
    {
        $result = $this->validator->validate(0.50);
        $this->assertFalse($result);
    }

    // অথবা "test" prefix দিয়ে নামকরণ
    public function testRejectsAmountAboveMaximum(): void
    {
        $result = $this->validator->validate(100_000);
        $this->assertFalse($result);
    }
}
```

### Assertions — বিস্তারিত

PHPUnit-এ অনেক ধরনের assertion আছে। প্রতিটির নির্দিষ্ট ব্যবহারক্ষেত্র রয়েছে:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use App\ValueObjects\Money;
use App\Collections\ProductCollection;

class AssertionDemoTest extends TestCase
{
    // ─── মৌলিক সমতা পরীক্ষা ─────────────────────────────
    #[Test]
    public function equality_assertions(): void
    {
        // assertEquals — loose comparison (==), type juggling হয়
        $this->assertEquals(100, '100');       // pass ✅
        $this->assertEquals(0, false);          // pass ✅

        // assertSame — strict comparison (===), type ও value দুটোই মেলাবে
        $this->assertSame(100, 100);            // pass ✅
        // $this->assertSame(100, '100');        // fail ❌

        // assertNotEquals, assertNotSame — উল্টো যাচাই
        $this->assertNotEquals('BDT', 'USD');
        $this->assertNotSame(1, true);
    }

    // ─── বুলিয়ান পরীক্ষা ──────────────────────────────────
    #[Test]
    public function boolean_assertions(): void
    {
        $isActive = true;
        $this->assertTrue($isActive);
        $this->assertFalse(!$isActive);

        $this->assertNull(null);
        $this->assertNotNull('bKash');

        $this->assertEmpty([]);
        $this->assertNotEmpty(['item1']);
    }

    // ─── টাইপ ও ইনস্ট্যান্স পরীক্ষা ───────────────────────
    #[Test]
    public function type_assertions(): void
    {
        $money = new Money(1500, 'BDT');

        $this->assertInstanceOf(Money::class, $money);
        $this->assertIsString($money->getCurrency());
        $this->assertIsFloat($money->getAmount());
        $this->assertIsArray([1, 2, 3]);
        $this->assertIsBool(true);
        $this->assertIsInt(42);
    }

    // ─── অ্যারে ও কালেকশন পরীক্ষা ─────────────────────────
    #[Test]
    public function array_assertions(): void
    {
        $products = ['চাল', 'ডাল', 'তেল', 'মাছ'];

        $this->assertCount(4, $products);
        $this->assertContains('ডাল', $products);
        $this->assertNotContains('মাংস', $products);

        $config = ['currency' => 'BDT', 'country' => 'BD'];
        $this->assertArrayHasKey('currency', $config);
        $this->assertArrayNotHasKey('language', $config);
    }

    // ─── স্ট্রিং পরীক্ষা ────────────────────────────────────
    #[Test]
    public function string_assertions(): void
    {
        $message = 'আপনার bKash পেমেন্ট সফল হয়েছে। ট্রানজেকশন আইডি: TXN123';

        $this->assertStringContainsString('সফল', $message);
        $this->assertStringStartsWith('আপনার', $message);
        $this->assertStringEndsWith('TXN123', $message);
        $this->assertMatchesRegularExpression('/TXN\d+/', $message);
    }

    // ─── ফাইল ও JSON পরীক্ষা ─────────────────────────────
    #[Test]
    public function json_assertion(): void
    {
        $response = json_encode([
            'status'  => 'success',
            'amount'  => 1500.00,
            'currency' => 'BDT',
        ]);

        $this->assertJson($response);
        $this->assertJsonStringEqualsJsonString(
            '{"status":"success","amount":1500,"currency":"BDT"}',
            $response
        );
    }
}
```

### Data Providers — প্যারামেট্রাইজড টেস্টিং

একই টেস্ট লজিক বিভিন্ন ডেটা দিয়ে চালানোর জন্য Data Provider অত্যন্ত কার্যকর। এতে কোড ডুপ্লিকেশন কমে এবং edge case coverage বাড়ে:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use App\Services\PriceCalculator;

class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new PriceCalculator();
    }

    // ─── PHP 8 Attribute স্টাইল DataProvider ───────────────
    #[Test]
    #[DataProvider('vatCalculationProvider')]
    public function it_calculates_vat_correctly(
        float $price,
        float $vatRate,
        float $expectedVat,
        string $scenario
    ): void {
        $vat = $this->calculator->calculateVat($price, $vatRate);
        $this->assertEqualsWithDelta($expectedVat, $vat, 0.01, $scenario);
    }

    public static function vatCalculationProvider(): iterable
    {
        // বাংলাদেশে VAT হার ১৫%
        yield 'সাধারণ পণ্য — ১৫% ভ্যাট' => [
            'price'       => 1000.00,
            'vatRate'     => 15.0,
            'expectedVat' => 150.00,
            'scenario'    => '১০০০ টাকায় ১৫% ভ্যাট = ১৫০ টাকা',
        ];

        yield 'শূন্য মূল্য' => [
            'price'       => 0.00,
            'vatRate'     => 15.0,
            'expectedVat' => 0.00,
            'scenario'    => 'শূন্য মূল্যে ভ্যাট শূন্য',
        ];

        yield 'ছোট মূল্য — পয়সা পর্যায়ের সূক্ষ্মতা' => [
            'price'       => 49.99,
            'vatRate'     => 15.0,
            'expectedVat' => 7.50,
            'scenario'    => '৪৯.৯৯ টাকায় ভ্যাট ≈ ৭.৫০',
        ];

        yield 'বড় অ্যামাউন্ট' => [
            'price'       => 500_000.00,
            'vatRate'     => 15.0,
            'expectedVat' => 75_000.00,
            'scenario'    => '৫ লক্ষ টাকায় ভ্যাট = ৭৫,০০০',
        ];

        yield 'কাস্টম ভ্যাট হার — ৫%' => [
            'price'       => 2000.00,
            'vatRate'     => 5.0,
            'expectedVat' => 100.00,
            'scenario'    => 'হ্রাসকৃত হারে ভ্যাট',
        ];
    }

    // ─── bKash ট্রানজেকশন ফি ক্যালকুলেশন ──────────────────
    #[Test]
    #[DataProvider('bkashFeeProvider')]
    public function it_calculates_bkash_cash_out_fee(
        float $amount,
        float $expectedFee
    ): void {
        $fee = $this->calculator->calculateBkashCashOutFee($amount);
        $this->assertEqualsWithDelta($expectedFee, $fee, 0.01);
    }

    public static function bkashFeeProvider(): iterable
    {
        // bKash Cash Out চার্জ: প্রতি হাজারে ১৮.৫০ টাকা
        yield '১,০০০ টাকা ক্যাশ আউট' => [1000.00, 18.50];
        yield '৫,০০০ টাকা ক্যাশ আউট' => [5000.00, 92.50];
        yield '২৫,০০০ টাকা ক্যাশ আউট' => [25000.00, 462.50];
        yield '৫০০ টাকা ক্যাশ আউট'   => [500.00, 9.25];
    }
}
```

### টেস্ট লাইফসাইকেল (Lifecycle)

PHPUnit-এ টেস্ট লাইফসাইকেল বোঝা গুরুত্বপূর্ণ — কখন কোন মেথড চলে সেটা জানা থাকলে সঠিক জায়গায় সেটআপ ও ক্লিনআপ করা যায়:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class LifecycleDemoTest extends TestCase
{
    /*
     * কার্যক্রমের ক্রম (Execution Order):
     *
     * ┌─────────────────────────────────────────────┐
     * │  setUpBeforeClass()    ← পুরো ক্লাসে ১ বার  │
     * │  ┌─────────────────────────────────────────┐ │
     * │  │  setUp()            ← প্রতি টেস্টে       │ │
     * │  │  testMethodOne()                         │ │
     * │  │  tearDown()         ← প্রতি টেস্টে       │ │
     * │  └─────────────────────────────────────────┘ │
     * │  ┌─────────────────────────────────────────┐ │
     * │  │  setUp()                                 │ │
     * │  │  testMethodTwo()                         │ │
     * │  │  tearDown()                              │ │
     * │  └─────────────────────────────────────────┘ │
     * │  tearDownAfterClass() ← পুরো ক্লাসে ১ বার   │
     * └─────────────────────────────────────────────┘
     */

    private static array $sharedExpensiveResource;
    private array $testData;

    // পুরো টেস্ট ক্লাসের আগে একবার চলে — ব্যয়বহুল সেটআপের জন্য
    public static function setUpBeforeClass(): void
    {
        parent::setUpBeforeClass();
        // যেমন: বড় ডেটাসেট লোড, ক্যাশ তৈরি
        self::$sharedExpensiveResource = ['config' => 'loaded'];
    }

    // প্রতিটি টেস্ট মেথডের আগে চলে — fresh state দেয়
    protected function setUp(): void
    {
        parent::setUp();
        $this->testData = [
            'products' => ['চাল', 'ডাল', 'তেল'],
            'prices'   => [65.00, 120.00, 180.00],
        ];
    }

    // প্রতিটি টেস্ট মেথডের পরে চলে
    protected function tearDown(): void
    {
        // resource cleanup
        unset($this->testData);
        parent::tearDown();
    }

    // পুরো টেস্ট ক্লাসের পরে একবার চলে
    public static function tearDownAfterClass(): void
    {
        self::$sharedExpensiveResource = [];
        parent::tearDownAfterClass();
    }

    public function testExample(): void
    {
        $this->assertCount(3, $this->testData['products']);
    }
}
```

### প্রাইভেট মেথড টেস্টিং

প্রাইভেট মেথড সরাসরি টেস্ট করা উচিত নয় — এটি implementation detail। তবে কখনো কখনো Reflection ব্যবহার করা যেতে পারে। **সেরা পদ্ধতি হলো পাবলিক ইন্টারফেসের মাধ্যমে পরোক্ষভাবে টেস্ট করা:**

```php
<?php

declare(strict_types=1);

namespace App\Services;

class TransactionIdGenerator
{
    public function generate(string $prefix = 'TXN'): string
    {
        $timestamp = $this->formatTimestamp(time());
        $random = $this->generateRandomSuffix(6);
        return "{$prefix}-{$timestamp}-{$random}";
    }

    private function formatTimestamp(int $time): string
    {
        return date('YmdHis', $time);
    }

    private function generateRandomSuffix(int $length): string
    {
        return substr(bin2hex(random_bytes($length)), 0, $length);
    }
}
```

```php
<?php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use App\Services\TransactionIdGenerator;

class TransactionIdGeneratorTest extends TestCase
{
    // ✅ সেরা পদ্ধতি: পাবলিক ইন্টারফেসের মাধ্যমে টেস্ট
    #[Test]
    public function generated_id_has_correct_format(): void
    {
        $generator = new TransactionIdGenerator();
        $id = $generator->generate('BKS');

        $this->assertMatchesRegularExpression(
            '/^BKS-\d{14}-[a-f0-9]{6}$/',
            $id
        );
    }

    // ⚠️ বিকল্প পদ্ধতি: Reflection — শুধু legacy code-এ ব্যবহার করুন
    #[Test]
    public function private_format_timestamp_works(): void
    {
        $generator = new TransactionIdGenerator();

        $reflection = new \ReflectionMethod($generator, 'formatTimestamp');
        // PHP 8.1+ এ setAccessible() আর দরকার নেই, তবে পুরনো ভার্সনে লাগে
        $reflection->setAccessible(true);

        $result = $reflection->invoke($generator, 1700000000);
        $this->assertSame('20231114222640', $result);
    }
}
```

### Exception ও Error টেস্টিং

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use App\Services\BkashPaymentService;
use App\Exceptions\InsufficientBalanceException;
use App\Exceptions\InvalidAmountException;
use InvalidArgumentException;

class ExceptionTestingTest extends TestCase
{
    private BkashPaymentService $service;

    protected function setUp(): void
    {
        $this->service = new BkashPaymentService();
    }

    // পদ্ধতি ১: expectException — সবচেয়ে সাধারণ
    #[Test]
    public function it_throws_on_negative_amount(): void
    {
        $this->expectException(InvalidAmountException::class);
        $this->expectExceptionMessage('পরিমাণ ঋণাত্মক হতে পারে না');
        $this->expectExceptionCode(422);

        $this->service->sendMoney(-500.00, '01712345678');
    }

    // পদ্ধতি ২: try-catch — যখন exception-এর পরেও assert করতে হয়
    #[Test]
    public function it_includes_amount_in_exception_for_limit_breach(): void
    {
        try {
            $this->service->sendMoney(200_000.00, '01712345678');
            $this->fail('প্রত্যাশিত InvalidAmountException ছুঁড়ে দেওয়া হয়নি');
        } catch (InvalidAmountException $e) {
            $this->assertSame(422, $e->getCode());
            $this->assertStringContainsString('সীমা অতিক্রম', $e->getMessage());
            $this->assertSame(200_000.00, $e->getAttemptedAmount());
        }
    }

    // পদ্ধতি ৩: callback পদ্ধতি — PHPUnit 10+
    #[Test]
    public function it_throws_for_invalid_phone_number(): void
    {
        $this->expectExceptionObject(
            new InvalidArgumentException('অবৈধ মোবাইল নম্বর')
        );

        $this->service->sendMoney(100.00, '123');
    }
}
```

### পূর্ণাঙ্গ উদাহরণ — Service Class ও Value Object টেস্টিং

```php
<?php

declare(strict_types=1);

namespace App\ValueObjects;

// ─── Value Object: Money (BDT) ──────────────────────────────
class Money
{
    public function __construct(
        private readonly float $amount,
        private readonly string $currency = 'BDT'
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('পরিমাণ ঋণাত্মক হতে পারে না');
        }
    }

    public function getAmount(): float
    {
        return $this->amount;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function add(Money $other): self
    {
        $this->ensureSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function subtract(Money $other): self
    {
        $this->ensureSameCurrency($other);
        if ($other->amount > $this->amount) {
            throw new \DomainException('অপর্যাপ্ত ব্যালেন্স');
        }
        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(float $factor): self
    {
        return new self(round($this->amount * $factor, 2), $this->currency);
    }

    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function ensureSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException(
                "মুদ্রা মিলছে না: {$this->currency} ≠ {$other->currency}"
            );
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\ValueObjects;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\CoversClass;
use App\ValueObjects\Money;

#[CoversClass(Money::class)]
class MoneyTest extends TestCase
{
    #[Test]
    public function it_creates_money_with_amount_and_currency(): void
    {
        $money = new Money(1500.00, 'BDT');

        $this->assertSame(1500.00, $money->getAmount());
        $this->assertSame('BDT', $money->getCurrency());
    }

    #[Test]
    public function it_defaults_to_bdt_currency(): void
    {
        $money = new Money(100.00);
        $this->assertSame('BDT', $money->getCurrency());
    }

    #[Test]
    public function it_rejects_negative_amount(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        new Money(-50.00);
    }

    #[Test]
    public function it_adds_two_money_values(): void
    {
        $price = new Money(850.00, 'BDT');   // চালের দাম
        $vat   = new Money(127.50, 'BDT');   // ১৫% ভ্যাট

        $total = $price->add($vat);

        $this->assertSame(977.50, $total->getAmount());
        $this->assertSame('BDT', $total->getCurrency());
    }

    #[Test]
    public function it_prevents_adding_different_currencies(): void
    {
        $bdt = new Money(1000.00, 'BDT');
        $usd = new Money(10.00, 'USD');

        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('মুদ্রা মিলছে না');

        $bdt->add($usd);
    }

    #[Test]
    public function it_subtracts_money_correctly(): void
    {
        $balance = new Money(5000.00, 'BDT');
        $expense = new Money(1200.00, 'BDT');

        $remaining = $balance->subtract($expense);
        $this->assertSame(3800.00, $remaining->getAmount());
    }

    #[Test]
    public function it_prevents_negative_balance_on_subtraction(): void
    {
        $balance = new Money(500.00, 'BDT');
        $expense = new Money(1000.00, 'BDT');

        $this->expectException(\DomainException::class);
        $this->expectExceptionMessage('অপর্যাপ্ত ব্যালেন্স');

        $balance->subtract($expense);
    }

    #[Test]
    #[DataProvider('multiplicationProvider')]
    public function it_multiplies_correctly(
        float $amount,
        float $factor,
        float $expected
    ): void {
        $money = new Money($amount, 'BDT');
        $result = $money->multiply($factor);
        $this->assertSame($expected, $result->getAmount());
    }

    public static function multiplicationProvider(): iterable
    {
        yield 'দ্বিগুণ মূল্য'        => [100.00, 2.0, 200.00];
        yield 'অর্ধেক মূল্য'         => [100.00, 0.5, 50.00];
        yield 'ভ্যাট সহ (১.১৫x)'    => [1000.00, 1.15, 1150.00];
        yield 'ডিসকাউন্ট (০.৯x)'    => [500.00, 0.9, 450.00];
        yield 'পয়সা রাউন্ডিং'       => [33.33, 3.0, 99.99];
    }

    #[Test]
    public function it_checks_equality(): void
    {
        $a = new Money(1500.00, 'BDT');
        $b = new Money(1500.00, 'BDT');
        $c = new Money(2000.00, 'BDT');

        $this->assertTrue($a->equals($b));
        $this->assertFalse($a->equals($c));
    }

    // immutability যাচাই — অপারেশনের পর মূল অবজেক্ট অপরিবর্তিত থাকে
    #[Test]
    public function operations_return_new_instances(): void
    {
        $original = new Money(1000.00, 'BDT');
        $added = $original->add(new Money(500.00, 'BDT'));

        $this->assertSame(1000.00, $original->getAmount());
        $this->assertSame(1500.00, $added->getAmount());
        $this->assertNotSame($original, $added);
    }
}
```

---

## 💻 Jest Deep Dive

### সেটআপ ও কনফিগারেশন

`jest.config.js` ফাইলে Jest-এর সমস্ত কনফিগারেশন থাকে:

```javascript
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  // টেস্ট environment
  testEnvironment: 'node',

  // টেস্ট ফাইলের প্যাটার্ন
  testMatch: [
    '<rootDir>/tests/**/*.test.js',
    '<rootDir>/tests/**/*.spec.js',
  ],

  // কভারেজ কনফিগারেশন
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/index.js',
    '!src/database/migrations/**',
  ],

  coverageDirectory: 'reports/coverage',
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 85,
      lines: 85,
      statements: 85,
    },
  },

  // কভারেজ রিপোর্টার
  coverageReporters: ['text', 'text-summary', 'lcov', 'html'],

  // সেটআপ ফাইল — সব টেস্টের আগে চলবে
  setupFilesAfterFramework: ['<rootDir>/tests/setup.js'],

  // ফাইল ট্রান্সফর্ম (Babel/TypeScript)
  transform: {},

  // ধীর টেস্ট সতর্কতা (৫ সেকেন্ডের বেশি হলে)
  slowTestThreshold: 5,

  // verbose আউটপুট
  verbose: true,
};
```

### describe / it / test ও expect Matchers

```javascript
// tests/unit/bkashValidator.test.js

const { BkashAmountValidator } = require('../../src/validators/bkashValidator');

describe('BkashAmountValidator', () => {
  let validator;

  // প্রতিটি টেস্টের আগে fresh instance
  beforeEach(() => {
    validator = new BkashAmountValidator();
  });

  // ─── গ্রুপিং: describe ব্লকে সম্পর্কিত টেস্ট একত্রে ──
  describe('validate()', () => {
    // it() ও test() সমার্থক — দুটোই ব্যবহার করা যায়
    it('should accept valid bKash amount', () => {
      expect(validator.validate(500)).toBe(true);
    });

    test('should reject amount below minimum (10 BDT)', () => {
      expect(validator.validate(5)).toBe(false);
    });

    it('should reject amount above maximum (25,000 BDT per txn)', () => {
      expect(validator.validate(30000)).toBe(false);
    });

    it('should reject negative amounts', () => {
      expect(validator.validate(-100)).toBe(false);
    });

    it('should reject non-numeric values', () => {
      expect(validator.validate('abc')).toBe(false);
      expect(validator.validate(null)).toBe(false);
      expect(validator.validate(undefined)).toBe(false);
    });
  });

  // ─── সাধারণ Matchers ──────────────────────────────────
  describe('Common Jest Matchers', () => {
    // সমতা পরীক্ষা
    test('equality matchers', () => {
      expect(100).toBe(100);                    // === strict
      expect({ a: 1 }).toEqual({ a: 1 });      // deep equality
      expect({ a: 1, b: 2 }).toMatchObject({ a: 1 }); // আংশিক মিল
    });

    // সত্যতা পরীক্ষা
    test('truthiness matchers', () => {
      expect(true).toBeTruthy();
      expect(0).toBeFalsy();
      expect(null).toBeNull();
      expect(undefined).toBeUndefined();
      expect('bKash').toBeDefined();
    });

    // সংখ্যা পরীক্ষা
    test('number matchers', () => {
      expect(1500).toBeGreaterThan(1000);
      expect(1500).toBeGreaterThanOrEqual(1500);
      expect(50).toBeLessThan(100);

      // ভাসমান বিন্দু — toBeCloseTo ব্যবহার করুন
      expect(0.1 + 0.2).toBeCloseTo(0.3, 10);
    });

    // স্ট্রিং ও রেগেক্স পরীক্ষা
    test('string matchers', () => {
      const msg = 'bKash পেমেন্ট সফল: TXN-20231115-ABC123';
      expect(msg).toContain('সফল');
      expect(msg).toMatch(/TXN-\d{8}-[A-Z0-9]+/);
    });

    // অ্যারে পরীক্ষা
    test('array matchers', () => {
      const items = ['চাল', 'ডাল', 'তেল'];
      expect(items).toHaveLength(3);
      expect(items).toContain('ডাল');
      expect(items).toEqual(expect.arrayContaining(['তেল', 'চাল']));
    });

    // অবজেক্ট পরীক্ষা
    test('object matchers', () => {
      const txn = {
        id: 'TXN001',
        amount: 1500,
        currency: 'BDT',
        sender: '01712345678',
      };

      expect(txn).toHaveProperty('amount');
      expect(txn).toHaveProperty('amount', 1500);
      expect(txn).toEqual(
        expect.objectContaining({ currency: 'BDT' })
      );
    });
  });
});
```

### Lifecycle — beforeEach / afterEach / beforeAll / afterAll

```javascript
// tests/unit/lifecycle.test.js

describe('Jest Lifecycle ডেমো', () => {
  /*
   * কার্যক্রম ক্রম:
   * 
   * beforeAll()        ← পুরো describe ব্লকে ১ বার
   *   beforeEach()     ← প্রতি test-এ
   *   test 1
   *   afterEach()      ← প্রতি test-এ
   *   
   *   beforeEach()
   *   test 2
   *   afterEach()
   * afterAll()         ← পুরো describe ব্লকে ১ বার
   */

  let sharedConfig;
  let testData;

  beforeAll(() => {
    // ব্যয়বহুল সেটআপ — একবারই চলে
    sharedConfig = { currency: 'BDT', vatRate: 0.15 };
  });

  beforeEach(() => {
    // প্রতি টেস্টে fresh data
    testData = {
      products: [
        { name: 'বাসমতি চাল', price: 120 },
        { name: 'মসুর ডাল', price: 95 },
        { name: 'সয়াবিন তেল', price: 180 },
      ],
    };
  });

  afterEach(() => {
    testData = null;
  });

  afterAll(() => {
    sharedConfig = null;
  });

  test('products are fresh each time', () => {
    testData.products.push({ name: 'চিনি', price: 85 });
    expect(testData.products).toHaveLength(4);
  });

  test('products still have 3 items (fresh from beforeEach)', () => {
    expect(testData.products).toHaveLength(3);
  });
});
```

### test.each — প্যারামেট্রাইজড টেস্টিং

```javascript
// tests/unit/priceCalculator.test.js

const { PriceCalculator } = require('../../src/services/priceCalculator');

describe('PriceCalculator', () => {
  const calculator = new PriceCalculator();

  // ─── টেবিল ফরম্যাটে test.each ──────────────────────────
  test.each([
    { price: 1000, vatRate: 0.15, expected: 150, label: 'সাধারণ পণ্য' },
    { price: 0, vatRate: 0.15, expected: 0, label: 'শূন্য মূল্য' },
    { price: 49.99, vatRate: 0.15, expected: 7.50, label: 'ছোট মূল্য' },
    { price: 500000, vatRate: 0.15, expected: 75000, label: 'বড় অ্যামাউন্ট' },
  ])('$label: ৳$price এ $vatRate হারে ভ্যাট = ৳$expected', ({ price, vatRate, expected }) => {
    const vat = calculator.calculateVat(price, vatRate);
    expect(vat).toBeCloseTo(expected, 2);
  });

  // ─── অ্যারে ফরম্যাটে test.each ─────────────────────────
  describe('bKash ক্যাশ আউট ফি', () => {
    test.each([
      [1000, 18.50],
      [5000, 92.50],
      [25000, 462.50],
      [500, 9.25],
    ])('৳%i ক্যাশ আউটে ফি = ৳%f', (amount, expectedFee) => {
      const fee = calculator.calculateBkashCashOutFee(amount);
      expect(fee).toBeCloseTo(expectedFee, 2);
    });
  });

  // ─── describe.each — গ্রুপ লেভেলে প্যারামেট্রাইজ ─────
  describe.each([
    { discount: 0.10, label: '১০%' },
    { discount: 0.20, label: '২০%' },
    { discount: 0.50, label: '৫০%' },
  ])('$label ডিসকাউন্ট', ({ discount }) => {
    test('should reduce price correctly', () => {
      const result = calculator.applyDiscount(1000, discount);
      expect(result).toBe(1000 * (1 - discount));
    });

    test('should not go below zero', () => {
      const result = calculator.applyDiscount(0, discount);
      expect(result).toBe(0);
    });
  });
});
```

### Async কোড টেস্টিং

```javascript
// tests/unit/asyncOperations.test.js

const { PaymentGateway } = require('../../src/services/paymentGateway');
const { fetchExchangeRate } = require('../../src/utils/currency');

describe('Async কোড টেস্টিং', () => {
  // ─── পদ্ধতি ১: async/await (সবচেয়ে পরিষ্কার) ──────────
  describe('async/await', () => {
    test('should process bKash payment successfully', async () => {
      const gateway = new PaymentGateway();
      const result = await gateway.processPayment({
        amount: 1500,
        currency: 'BDT',
        method: 'bkash',
        phone: '01712345678',
      });

      expect(result).toMatchObject({
        status: 'success',
        amount: 1500,
        currency: 'BDT',
      });
      expect(result.transactionId).toBeDefined();
    });

    test('should reject invalid payment', async () => {
      const gateway = new PaymentGateway();

      await expect(
        gateway.processPayment({ amount: -100, method: 'bkash' })
      ).rejects.toThrow('অবৈধ পরিমাণ');
    });
  });

  // ─── পদ্ধতি ২: Promise return ──────────────────────────
  describe('Promise', () => {
    test('should fetch BDT exchange rate', () => {
      return fetchExchangeRate('USD', 'BDT').then((rate) => {
        expect(rate).toBeGreaterThan(100);
        expect(rate).toBeLessThan(150);
      });
    });
  });

  // ─── পদ্ধতি ৩: resolves / rejects matchers ────────────
  describe('resolves/rejects', () => {
    test('should resolve with exchange rate object', async () => {
      await expect(fetchExchangeRate('USD', 'BDT')).resolves.toEqual(
        expect.objectContaining({
          from: 'USD',
          to: 'BDT',
          rate: expect.any(Number),
        })
      );
    });

    test('should reject for unsupported currency', async () => {
      await expect(fetchExchangeRate('XYZ', 'BDT')).rejects.toThrow(
        'অসমর্থিত মুদ্রা'
      );
    });
  });

  // ─── পদ্ধতি ৪: Callback (legacy কোডের জন্য) ───────────
  describe('Callback style (done parameter)', () => {
    test('should notify on payment completion', (done) => {
      const gateway = new PaymentGateway();

      gateway.onPaymentComplete(1500, 'BDT', (error, result) => {
        try {
          expect(error).toBeNull();
          expect(result.status).toBe('completed');
          done();
        } catch (e) {
          done(e);
        }
      });
    });
  });

  // ─── Fake Timers — setTimeout/setInterval টেস্টিং ────
  describe('Fake Timers', () => {
    beforeEach(() => {
      jest.useFakeTimers();
    });

    afterEach(() => {
      jest.useRealTimers();
    });

    test('payment timeout after 30 seconds', () => {
      const gateway = new PaymentGateway();
      const callback = jest.fn();

      gateway.processWithTimeout(1500, callback);

      // ২৯ সেকেন্ড — এখনো timeout হয়নি
      jest.advanceTimersByTime(29000);
      expect(callback).not.toHaveBeenCalled();

      // ৩০ সেকেন্ড — timeout!
      jest.advanceTimersByTime(1000);
      expect(callback).toHaveBeenCalledWith(
        expect.objectContaining({ error: 'TIMEOUT' })
      );
    });
  });
});
```

### পূর্ণাঙ্গ উদাহরণ — Utility ও Service Class টেস্টিং

```javascript
// src/utils/bdtFormatter.js
class BdtFormatter {
  format(amount, options = {}) {
    if (typeof amount !== 'number' || isNaN(amount)) {
      throw new TypeError('পরিমাণ অবশ্যই সংখ্যা হতে হবে');
    }

    const { showSymbol = true, decimals = 2 } = options;
    const formatted = amount.toFixed(decimals).replace(
      /\B(?=(\d{2})+(?!\d))/g,  // বাংলাদেশী গ্রুপিং: ১,০০,০০০
      ','
    );

    return showSymbol ? `৳${formatted}` : formatted;
  }

  parse(formatted) {
    const cleaned = formatted.replace(/[৳,\s]/g, '');
    const amount = parseFloat(cleaned);
    if (isNaN(amount)) {
      throw new Error('পার্স করা সম্ভব হয়নি');
    }
    return amount;
  }
}

module.exports = { BdtFormatter };
```

```javascript
// tests/unit/bdtFormatter.test.js

const { BdtFormatter } = require('../../src/utils/bdtFormatter');

describe('BdtFormatter', () => {
  let formatter;

  beforeEach(() => {
    formatter = new BdtFormatter();
  });

  describe('format()', () => {
    test('should format with ৳ symbol by default', () => {
      expect(formatter.format(1500)).toBe('৳1,500.00');
    });

    test('should format without symbol when specified', () => {
      expect(formatter.format(1500, { showSymbol: false })).toBe('1,500.00');
    });

    test('should handle zero', () => {
      expect(formatter.format(0)).toBe('৳0.00');
    });

    test('should handle large amounts with BD grouping', () => {
      expect(formatter.format(1000000)).toBe('৳10,00,000.00');
    });

    test('should respect decimal places option', () => {
      expect(formatter.format(99.5, { decimals: 0 })).toBe('৳100');
    });

    test('should throw for non-numeric input', () => {
      expect(() => formatter.format('abc')).toThrow(TypeError);
      expect(() => formatter.format(null)).toThrow('পরিমাণ অবশ্যই সংখ্যা হতে হবে');
    });
  });

  describe('parse()', () => {
    test('should parse formatted BDT string', () => {
      expect(formatter.parse('৳1,500.00')).toBe(1500);
    });

    test('should parse without symbol', () => {
      expect(formatter.parse('10,00,000.50')).toBe(1000000.50);
    });

    test('should throw for unparseable string', () => {
      expect(() => formatter.parse('abc')).toThrow('পার্স করা সম্ভব হয়নি');
    });
  });
});
```

---

## 🔥 Advanced টপিকস

### ১. টেস্টিং প্যাটার্নস

#### AAA Pattern (Arrange-Act-Assert)

সবচেয়ে বহুল ব্যবহৃত প্যাটার্ন — প্রতিটি টেস্টকে তিনটি স্পষ্ট ভাগে ভাগ করে:

```php
<?php
#[Test]
public function it_calculates_total_with_vat(): void
{
    // Arrange — প্রস্তুতি: টেস্টের জন্য প্রয়োজনীয় সবকিছু সাজানো
    $calculator = new PriceCalculator();
    $basePrice = new Money(1000.00, 'BDT');
    $vatRate = 0.15;

    // Act — কার্যকরণ: যে ব্যবহার টেস্ট করতে চাই সেটা চালানো
    $total = $calculator->calculateTotalWithVat($basePrice, $vatRate);

    // Assert — প্রতিপাদন: ফলাফল প্রত্যাশা অনুযায়ী কিনা যাচাই
    $this->assertSame(1150.00, $total->getAmount());
    $this->assertSame('BDT', $total->getCurrency());
}
```

#### Given-When-Then (BDD স্টাইল)

BDD-ভিত্তিক — ব্যবসায়িক ভাষায় টেস্ট বর্ণনা করে:

```javascript
describe('শপিং কার্ট', () => {
  test('Given কার্টে পণ্য আছে, When ডিসকাউন্ট কোড প্রয়োগ হয়, Then মূল্য কমে', () => {
    // Given — প্রদত্ত অবস্থা
    const cart = new ShoppingCart();
    cart.addItem({ name: 'শার্ট', price: 1200 });
    cart.addItem({ name: 'প্যান্ট', price: 1800 });

    // When — যখন এই ক্রিয়া ঘটে
    cart.applyDiscount('EID20'); // ঈদ উপলক্ষে ২০% ছাড়

    // Then — তখন এই ফলাফল পাওয়া যায়
    expect(cart.getTotal()).toBe(2400); // ৩০০০ - ২০%
  });
});
```

#### Object Mother Pattern

জটিল টেস্ট অবজেক্ট তৈরির জন্য ফ্যাক্টরি ক্লাস — বারবার একই সেটআপ কোড লেখা থেকে বাঁচায়:

```php
<?php

declare(strict_types=1);

namespace Tests\Mothers;

use App\Entities\Customer;
use App\ValueObjects\Money;

class CustomerMother
{
    public static function createRegular(): Customer
    {
        return new Customer(
            name: 'রহিম উদ্দিন',
            phone: '01712345678',
            balance: new Money(5000.00, 'BDT'),
            tier: 'regular'
        );
    }

    public static function createPremium(): Customer
    {
        return new Customer(
            name: 'করিম সাহেব',
            phone: '01898765432',
            balance: new Money(50000.00, 'BDT'),
            tier: 'premium'
        );
    }

    public static function createWithBalance(float $amount): Customer
    {
        return new Customer(
            name: 'টেস্ট গ্রাহক',
            phone: '01700000000',
            balance: new Money($amount, 'BDT'),
            tier: 'regular'
        );
    }

    public static function createWithZeroBalance(): Customer
    {
        return self::createWithBalance(0.00);
    }
}

// ব্যবহার:
// $customer = CustomerMother::createPremium();
// $broke = CustomerMother::createWithZeroBalance();
```

#### Test Builder Pattern

আরও flexible — fluent interface দিয়ে step-by-step অবজেক্ট তৈরি:

```php
<?php

declare(strict_types=1);

namespace Tests\Builders;

use App\Entities\Order;
use App\Entities\OrderItem;
use App\ValueObjects\Money;

class OrderBuilder
{
    private string $customerId = 'CUST001';
    private array $items = [];
    private string $status = 'pending';
    private ?string $couponCode = null;

    public static function anOrder(): self
    {
        return new self();
    }

    public function forCustomer(string $customerId): self
    {
        $this->customerId = $customerId;
        return $this;
    }

    public function withItem(string $name, float $price, int $qty = 1): self
    {
        $this->items[] = new OrderItem($name, new Money($price, 'BDT'), $qty);
        return $this;
    }

    public function withStatus(string $status): self
    {
        $this->status = $status;
        return $this;
    }

    public function withCoupon(string $code): self
    {
        $this->couponCode = $code;
        return $this;
    }

    public function build(): Order
    {
        $order = new Order($this->customerId);
        foreach ($this->items as $item) {
            $order->addItem($item);
        }
        $order->setStatus($this->status);
        if ($this->couponCode) {
            $order->applyCoupon($this->couponCode);
        }
        return $order;
    }
}

// ব্যবহার:
// $order = OrderBuilder::anOrder()
//     ->forCustomer('CUST-RAHIM')
//     ->withItem('বাসমতি চাল ৫কেজি', 650.00, 2)
//     ->withItem('সয়াবিন তেল ৫লি', 780.00)
//     ->withCoupon('EID2024')
//     ->build();
```

### ২. Edge Cases টেস্টিং

Edge case মিস করলেই প্রোডাকশনে বাগ আসে। একটি পদ্ধতিগত চেকলিস্ট মেনে চলা উচিত:

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use App\Services\PhoneNumberValidator;

class EdgeCaseTest extends TestCase
{
    // ─── Null ও Empty মান ─────────────────────────────────
    #[Test]
    #[DataProvider('nullAndEmptyProvider')]
    public function it_handles_null_and_empty_values(mixed $input): void
    {
        $validator = new PhoneNumberValidator();
        $this->assertFalse($validator->isValid($input));
    }

    public static function nullAndEmptyProvider(): iterable
    {
        yield 'null'          => [null];
        yield 'empty string'  => [''];
        yield 'whitespace'    => ['   '];
        yield 'empty array'   => [[]];
        yield 'zero'          => [0];
        yield 'false'         => [false];
    }

    // ─── Boundary Values (সীমানা মান) ──────────────────────
    #[Test]
    #[DataProvider('boundaryProvider')]
    public function it_validates_bkash_amount_boundaries(
        float $amount,
        bool $expected,
        string $case
    ): void {
        $validator = new BkashAmountValidator();
        $this->assertSame($expected, $validator->validate($amount), $case);
    }

    public static function boundaryProvider(): iterable
    {
        // bKash সীমা: সর্বনিম্ন ১০ টাকা, সর্বোচ্চ ২৫,০০০ টাকা
        yield 'সীমার নিচে: ৯.৯৯'           => [9.99, false, 'just below min'];
        yield 'সর্বনিম্ন সীমা: ১০.০০'       => [10.00, true, 'exact minimum'];
        yield 'সীমার ঠিক উপরে: ১০.০১'       => [10.01, true, 'just above min'];
        yield 'মাঝামাঝি মান: ১২,৫০০'        => [12500.00, true, 'middle value'];
        yield 'সর্বোচ্চ সীমার ঠিক নিচে'     => [24999.99, true, 'just below max'];
        yield 'সর্বোচ্চ সীমা: ২৫,০০০'       => [25000.00, true, 'exact maximum'];
        yield 'সীমার উপরে: ২৫,০০০.০১'       => [25000.01, false, 'just above max'];
        yield 'শূন্য'                        => [0.00, false, 'zero'];
        yield 'ঋণাত্মক'                     => [-1.00, false, 'negative'];
    }

    // ─── Error Paths (ত্রুটি পথ) ───────────────────────────
    #[Test]
    public function it_handles_network_timeout_gracefully(): void
    {
        $service = new PaymentService(
            timeout: 0  // তাৎক্ষণিক timeout
        );

        $result = $service->attemptPayment(1500.00);

        $this->assertSame('failed', $result->status);
        $this->assertSame('TIMEOUT', $result->errorCode);
    }
}
```

### ৩. Pure Functions বনাম Side Effects

**Pure function** টেস্ট করা সবচেয়ে সহজ — একই ইনপুটে সবসময় একই আউটপুট দেয়, কোনো বাইরের state বদলায় না:

```javascript
// ─── Pure Function — টেস্ট করা অত্যন্ত সহজ ────────────
// src/utils/calculations.js
function calculateDiscount(price, discountPercent) {
  if (price < 0 || discountPercent < 0 || discountPercent > 100) {
    throw new RangeError('অবৈধ মান');
  }
  return Math.round(price * (1 - discountPercent / 100) * 100) / 100;
}

// tests — কোনো mock বা setup লাগে না
test('pure function: discount calculation', () => {
  expect(calculateDiscount(1000, 20)).toBe(800);
  expect(calculateDiscount(1000, 0)).toBe(1000);
  expect(calculateDiscount(0, 50)).toBe(0);
});
```

```javascript
// ─── Side Effect আছে — টেস্ট করতে isolation দরকার ──────
// src/services/notificationService.js
class NotificationService {
  constructor(smsClient, logger) {
    this.smsClient = smsClient;
    this.logger = logger;
  }

  async sendPaymentConfirmation(phone, amount) {
    const message = `আপনার ৳${amount} পেমেন্ট সফল হয়েছে`;

    // Side effects: SMS পাঠানো + লগ করা
    await this.smsClient.send(phone, message);
    this.logger.info('SMS sent', { phone, amount });

    return { sent: true, message };
  }
}

// tests — dependency injection দিয়ে mock ব্যবহার
test('side effect: sends SMS and logs', async () => {
  // mock dependencies
  const mockSmsClient = { send: jest.fn().mockResolvedValue(true) };
  const mockLogger = { info: jest.fn() };

  const service = new NotificationService(mockSmsClient, mockLogger);
  const result = await service.sendPaymentConfirmation('01712345678', 1500);

  expect(result.sent).toBe(true);
  expect(mockSmsClient.send).toHaveBeenCalledWith(
    '01712345678',
    'আপনার ৳1500 পেমেন্ট সফল হয়েছে'
  );
  expect(mockLogger.info).toHaveBeenCalledTimes(1);
});
```

**মূল শিক্ষা:** যতটা সম্ভব কোডকে pure function-এ ভাঙুন। Side effects আলাদা স্তরে রাখুন — এতে টেস্টিং অনেক সহজ হয়।

### ৪. Code Coverage কনফিগারেশন ও ব্যাখ্যা

```
কভারেজ মেট্রিক্স ব্যাখ্যা:

┌──────────────────┬─────────────────────────────────────────────────┐
│ Line Coverage     │ কতগুলো লাইন execute হয়েছে                      │
│                   │ সবচেয়ে সাধারণ মেট্রিক                          │
├──────────────────┼─────────────────────────────────────────────────┤
│ Branch Coverage   │ if/else, switch, ternary-র কতগুলো শাখা         │
│                   │ cover হয়েছে — Line-এর চেয়ে বেশি গুরুত্বপূর্ণ   │
├──────────────────┼─────────────────────────────────────────────────┤
│ Function Coverage │ কতগুলো ফাংশন/মেথড কমপক্ষে একবার call হয়েছে   │
├──────────────────┼─────────────────────────────────────────────────┤
│ Statement Coverage│ কতগুলো statement execute হয়েছে                 │
│                   │ (একই লাইনে একাধিক statement থাকতে পারে)         │
└──────────────────┴─────────────────────────────────────────────────┘

⚠️ সতর্কতা: ১০০% কভারেজ ≠ ১০০% সঠিক কোড!
   কভারেজ বলে "কোন কোড চলেছে" — "কোড সঠিক" তা বলে না।
```

```bash
# PHPUnit — কভারেজ রিপোর্ট তৈরি
php vendor/bin/phpunit --coverage-html reports/coverage --coverage-text

# Jest — কভারেজ সহ টেস্ট চালানো
npx jest --coverage --coverageReporters="text" --coverageReporters="html"
```

```
# Jest কভারেজ আউটপুট উদাহরণ:
--------------------|---------|----------|---------|---------|
File                | % Stmts | % Branch | % Funcs | % Lines |
--------------------|---------|----------|---------|---------|
All files           |   87.5  |   75.0   |   90.0  |   88.2  |
 bdtFormatter.js    |   95.0  |   85.7   |  100.0  |   94.4  |
 priceCalculator.js |   80.0  |   66.7   |   80.0  |   82.4  |
--------------------|---------|----------|---------|---------|

বিশ্লেষণ:
• bdtFormatter.js — ভালো কভারেজ ✅ (তবে branch 85.7% — কিছু if/else মিস)
• priceCalculator.js — branch 66.7% ⚠️ — আরো edge case টেস্ট দরকার
```

### ৫. Snapshot Testing (Jest)

স্ন্যাপশট টেস্টিং UI কম্পোনেন্ট বা বড় ডেটা স্ট্রাকচারের রিগ্রেশন ধরতে ব্যবহৃত হয়। প্রথমবার চালালে স্ন্যাপশট সেভ হয়, পরবর্তীতে পরিবর্তন হলে ফেইল করে:

```javascript
// tests/unit/receiptGenerator.test.js

const { ReceiptGenerator } = require('../../src/services/receiptGenerator');

describe('ReceiptGenerator', () => {
  test('should generate correct bKash payment receipt', () => {
    const generator = new ReceiptGenerator();
    const receipt = generator.generate({
      transactionId: 'TXN-20231115-ABC123',
      amount: 1500,
      currency: 'BDT',
      sender: '01712345678',
      receiver: '01898765432',
      type: 'send-money',
      timestamp: '2023-11-15T10:30:00Z',
    });

    // পুরো অবজেক্ট স্ন্যাপশট — .snap ফাইলে সেভ হয়
    expect(receipt).toMatchSnapshot();
  });

  // ─── Inline Snapshot — ছোট আউটপুটের জন্য ─────────────
  test('should format receipt header', () => {
    const generator = new ReceiptGenerator();
    const header = generator.formatHeader('bKash');

    expect(header).toMatchInlineSnapshot(`
      "═══════════════════════════════
       bKash পেমেন্ট রিসিট
       ═══════════════════════════════"
    `);
  });

  // ─── Property Matcher সহ Snapshot ─────────────────────
  test('snapshot with dynamic values', () => {
    const generator = new ReceiptGenerator();
    const receipt = generator.generate({
      transactionId: 'TXN-DYNAMIC',
      amount: 2000,
      currency: 'BDT',
      timestamp: new Date().toISOString(),
    });

    // dynamic field-গুলোর জন্য matcher ব্যবহার
    expect(receipt).toMatchSnapshot({
      transactionId: expect.any(String),
      timestamp: expect.any(String),
      generatedAt: expect.any(String),
    });
  });
});

// স্ন্যাপশট আপডেট করতে: npx jest --updateSnapshot (বা -u)
```

### ৬. Mutation Testing ধারণা

মিউটেশন টেস্টিং টেস্ট স্যুটের **গুণমান** পরিমাপ করে। এটি সোর্স কোডে ছোট ছোট পরিবর্তন (mutant) করে দেখে টেস্ট ফেইল করে কিনা:

```
মিউটেশন টেস্টিং কীভাবে কাজ করে:

  মূল কোড                    মিউটেন্ট (পরিবর্তিত কোড)
  ─────────                   ────────────────────────
  if (amount > 0)      →     if (amount >= 0)        // সীমানা পরিবর্তন
  if (amount > 0)      →     if (amount < 0)         // শর্ত উল্টানো
  return price * 1.15  →     return price * 1.00     // মান পরিবর্তন
  return price + tax   →     return price - tax      // অপারেটর পরিবর্তন
  if (a && b)          →     if (a || b)             // লজিক পরিবর্তন

  ✅ Killed Mutant: টেস্ট ফেইল করেছে — টেস্ট ভালো!
  ❌ Survived Mutant: টেস্ট পাস করেছে — টেস্ট দুর্বল!

  Mutation Score = (Killed / Total) × ১০০%
  লক্ষ্য: ৮০%+ স্কোর
```

```bash
# PHP — Infection (Mutation Testing Framework)
composer require --dev infection/infection
vendor/bin/infection --min-msi=80 --min-covered-msi=90

# JavaScript — Stryker Mutator
npx stryker init
npx stryker run
```

```json
// stryker.conf.json (Jest-এর জন্য)
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "mutate": ["src/**/*.js", "!src/**/*.test.js"],
  "testRunner": "jest",
  "jest": { "configFile": "jest.config.js" },
  "reporters": ["html", "clear-text", "progress"],
  "thresholds": { "high": 80, "low": 60, "break": 50 }
}
```

### ৭. Property-Based Testing ধারণা

প্রচলিত টেস্টিং-এ আমরা নির্দিষ্ট মান দিয়ে টেস্ট করি। Property-based testing-এ আমরা **বৈশিষ্ট্য (property)** সংজ্ঞায়িত করি এবং ফ্রেমওয়ার্ক শত শত র‍্যান্ডম ইনপুট তৈরি করে সেই বৈশিষ্ট্য ভাঙে কিনা দেখে:

```javascript
// fast-check লাইব্রেরি ব্যবহার করে Property-Based Testing
const fc = require('fast-check');

describe('Property-Based Testing', () => {
  // বৈশিষ্ট্য ১: ভ্যাট যোগ করলে মূল্য বাড়বে বা সমান থাকবে
  test('VAT সবসময় মূল্য বাড়ায় বা সমান রাখে', () => {
    fc.assert(
      fc.property(
        fc.float({ min: 0, max: 1_000_000, noNaN: true }),
        fc.float({ min: 0, max: 100, noNaN: true }),
        (price, vatRate) => {
          const withVat = price * (1 + vatRate / 100);
          return withVat >= price;
        }
      )
    );
  });

  // বৈশিষ্ট্য ২: ডিসকাউন্ট প্রয়োগ করলে মূল্য কমবে বা সমান থাকবে
  test('ডিসকাউন্ট কখনো মূল্য বাড়ায় না', () => {
    fc.assert(
      fc.property(
        fc.float({ min: 0, max: 1_000_000, noNaN: true }),
        fc.float({ min: 0, max: 100, noNaN: true }),
        (price, discountPercent) => {
          const discounted = price * (1 - discountPercent / 100);
          return discounted <= price;
        }
      )
    );
  });

  // বৈশিষ্ট্য ৩: format → parse → সমান মান
  test('format এবং parse একে অপরের inverse', () => {
    const formatter = new BdtFormatter();

    fc.assert(
      fc.property(
        fc.float({ min: 0, max: 99_999_999.99, noNaN: true }),
        (amount) => {
          const rounded = Math.round(amount * 100) / 100;
          const formatted = formatter.format(rounded);
          const parsed = formatter.parse(formatted);
          return Math.abs(parsed - rounded) < 0.01;
        }
      )
    );
  });
});
```

### ৮. SOLID কোড টেস্টিং বনাম Tightly Coupled কোড

**Tightly Coupled কোড — টেস্ট করা কঠিন:**

```php
<?php
// ❌ খারাপ — সরাসরি dependency তৈরি করে, টেস্ট করা প্রায় অসম্ভব
class PaymentProcessor
{
    public function process(float $amount, string $phone): array
    {
        // সরাসরি bKash API কল — mock করা যায় না
        $bkash = new BkashApiClient('live-api-key');
        $response = $bkash->sendMoney($amount, $phone);

        // সরাসরি ডেটাবেস কল
        $db = new PDO('mysql:host=localhost;dbname=payments', 'root', 'pass');
        $stmt = $db->prepare('INSERT INTO transactions ...');
        $stmt->execute([$amount, $phone, $response['txnId']]);

        // সরাসরি SMS পাঠায়
        $sms = new SmsGateway();
        $sms->send($phone, "৳{$amount} পেমেন্ট সফল");

        return $response;
    }
}
// টেস্ট করতে গেলে: আসল API কল হবে, আসল DB লাগবে, আসল SMS যাবে! 😱
```

**SOLID কোড — টেস্ট করা সহজ:**

```php
<?php
// ✅ ভালো — Dependency Injection, Interface Segregation
interface PaymentGatewayInterface
{
    public function sendMoney(float $amount, string $phone): PaymentResult;
}

interface TransactionRepositoryInterface
{
    public function save(Transaction $transaction): void;
}

interface NotificationServiceInterface
{
    public function sendPaymentConfirmation(string $phone, float $amount): void;
}

class PaymentProcessor
{
    public function __construct(
        private readonly PaymentGatewayInterface $gateway,
        private readonly TransactionRepositoryInterface $repository,
        private readonly NotificationServiceInterface $notifier,
    ) {}

    public function process(float $amount, string $phone): PaymentResult
    {
        $result = $this->gateway->sendMoney($amount, $phone);

        $this->repository->save(
            new Transaction($amount, $phone, $result->transactionId)
        );

        $this->notifier->sendPaymentConfirmation($phone, $amount);

        return $result;
    }
}
```

```php
<?php
// এখন টেস্ট করা সহজ — mock inject করলেই হলো
class PaymentProcessorTest extends TestCase
{
    #[Test]
    public function it_processes_payment_successfully(): void
    {
        // Arrange — mock dependencies
        $mockGateway = $this->createMock(PaymentGatewayInterface::class);
        $mockGateway->method('sendMoney')
            ->willReturn(new PaymentResult('TXN123', 'success'));

        $mockRepo = $this->createMock(TransactionRepositoryInterface::class);
        $mockRepo->expects($this->once())->method('save');

        $mockNotifier = $this->createMock(NotificationServiceInterface::class);
        $mockNotifier->expects($this->once())
            ->method('sendPaymentConfirmation')
            ->with('01712345678', 1500.00);

        $processor = new PaymentProcessor($mockGateway, $mockRepo, $mockNotifier);

        // Act
        $result = $processor->process(1500.00, '01712345678');

        // Assert
        $this->assertSame('success', $result->status);
        $this->assertSame('TXN123', $result->transactionId);
    }

    #[Test]
    public function it_does_not_save_or_notify_on_gateway_failure(): void
    {
        $mockGateway = $this->createMock(PaymentGatewayInterface::class);
        $mockGateway->method('sendMoney')
            ->willThrowException(new PaymentFailedException('Gateway error'));

        $mockRepo = $this->createMock(TransactionRepositoryInterface::class);
        $mockRepo->expects($this->never())->method('save');

        $mockNotifier = $this->createMock(NotificationServiceInterface::class);
        $mockNotifier->expects($this->never())->method('sendPaymentConfirmation');

        $processor = new PaymentProcessor($mockGateway, $mockRepo, $mockNotifier);

        $this->expectException(PaymentFailedException::class);
        $processor->process(1500.00, '01712345678');
    }
}
```

---

## ✅ সেরা অনুশীলন (Best Practices)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ইউনিট টেস্টিং সেরা অনুশীলন                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ১. প্রতি টেস্টে একটি ধারণা (One Concept per Test)                 │
│     • একটি টেস্ট একটি জিনিসই যাচাই করবে                           │
│     • একাধিক assertion থাকতে পারে, তবে একই ধারণার জন্য              │
│                                                                     │
│  ২. বর্ণনামূলক নামকরণ                                              │
│     ❌ testCalculate()                                               │
│     ✅ it_calculates_vat_for_standard_rate()                        │
│     ✅ it_throws_when_amount_is_negative()                          │
│                                                                     │
│  ৩. AAA প্যাটার্ন অনুসরণ                                           │
│     Arrange → Act → Assert — তিনটি অংশ স্পষ্টভাবে আলাদা           │
│                                                                     │
│  ৪. টেস্ট ডেটা অর্থবহ হবে                                          │
│     ❌ assert(calc(1, 2), 3)                                        │
│     ✅ assert(calcVat(1000, 0.15), 150)  // ১০০০ টাকায় ১৫% VAT     │
│                                                                     │
│  ৫. Implementation নয়, Behavior টেস্ট করুন                         │
│     • কী return করে তা টেস্ট করুন, কীভাবে করে তা নয়               │
│     • Internal method call verify করবেন না (fragile হয়)             │
│                                                                     │
│  ৬. DRY নয়, DAMP (Descriptive And Meaningful Phrases)              │
│     • টেস্টে কিছুটা repetition গ্রহণযোগ্য                          │
│     • প্রতিটি টেস্ট স্বতন্ত্রভাবে পড়ে বোঝা যায় — এটাই গুরুত্বপূর্ণ │
│                                                                     │
│  ৭. টেস্ট তাড়াতাড়ি ও ঘন ঘন চালান                                 │
│     • প্রতিটি commit-এ, প্রতিটি PR-এ                                │
│     • CI/CD pipeline-এ স্বয়ংক্রিয়                                   │
│                                                                     │
│  ৮. Test Data Builder / Object Mother ব্যবহার করুন                 │
│     • জটিল অবজেক্ট তৈরির জন্য                                      │
│     • DRY setup, readable tests                                     │
│                                                                     │
│  ৯. Edge cases ভুলবেন না                                            │
│     • null, undefined, empty, boundary, overflow, special chars      │
│                                                                     │
│  ১০. টেস্ট কোডকেও production কোডের মতো maintain করুন              │
│      • ডেড টেস্ট মুছে ফেলুন, রিফ্যাক্টর করুন                       │
│      • কিন্তু পড়ার সুবিধার্থে কিছু repetition রাখুন                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ⚠️ অ্যান্টি-প্যাটার্নস (যা করবেন না)

### ১. Implementation টেস্টিং (Fragile Tests)

```php
<?php
// ❌ খারাপ — internal method call verify করছে
#[Test]
public function bad_test_checks_internal_calls(): void
{
    $calc = $this->createPartialMock(PriceCalculator::class, ['roundToTwoDecimals']);

    // রাউন্ডিং কীভাবে হচ্ছে সেটা টেস্ট — implementation বদলালে ভাঙবে
    $calc->expects($this->once())
         ->method('roundToTwoDecimals')
         ->with(150.00);

    $calc->calculateVat(1000.00, 0.15);
}

// ✅ ভালো — শুধু behavior (আউটপুট) টেস্ট করছে
#[Test]
public function good_test_checks_result(): void
{
    $calc = new PriceCalculator();
    $vat = $calc->calculateVat(1000.00, 0.15);
    $this->assertSame(150.00, $vat);
}
```

### ২. ধীর টেস্ট

```javascript
// ❌ খারাপ — আসল API কল, আসল টাইমআউট
test('slow: real API call', async () => {
  const result = await fetch('https://api.bkash.com/rate'); // ধীর + unreliable
  expect(result.status).toBe(200);
}, 30000);

// ✅ ভালো — mock ব্যবহার, মিলিসেকেন্ডে শেষ
test('fast: mocked response', async () => {
  jest.spyOn(global, 'fetch').mockResolvedValue({
    status: 200,
    json: async () => ({ rate: 110.5 }),
  });

  const result = await fetch('https://api.bkash.com/rate');
  expect(result.status).toBe(200);
});
```

### ৩. God Test — একটি টেস্টে অনেক কিছু

```php
<?php
// ❌ খারাপ — একটি টেস্টে সবকিছু পরীক্ষা
#[Test]
public function test_everything(): void
{
    $service = new OrderService();

    // তৈরি
    $order = $service->create('CUST001', [['চাল', 65], ['ডাল', 120]]);
    $this->assertNotNull($order);

    // ক্যালকুলেশন
    $this->assertSame(185.00, $order->getSubtotal());

    // ভ্যাট
    $this->assertSame(27.75, $order->getVat());

    // ডিসকাউন্ট
    $order->applyDiscount(10);
    $this->assertSame(191.48, $order->getTotal());

    // ক্যান্সেল
    $order->cancel();
    $this->assertSame('cancelled', $order->getStatus());
}

// ✅ ভালো — প্রতিটি ধারণার জন্য আলাদা টেস্ট
#[Test]
public function it_calculates_subtotal(): void { /* ... */ }

#[Test]
public function it_adds_vat_to_subtotal(): void { /* ... */ }

#[Test]
public function it_applies_percentage_discount(): void { /* ... */ }

#[Test]
public function it_can_be_cancelled(): void { /* ... */ }
```

### ৪. টেস্টের মধ্যে নির্ভরতা (Shared State)

```javascript
// ❌ খারাপ — টেস্টগুলো একে অপরের উপর নির্ভরশীল
let counter = 0;

test('increment', () => {
  counter++;
  expect(counter).toBe(1);
});

test('depends on previous test', () => {
  // counter === 1 ধরে নিচ্ছে — আগের টেস্ট না চললে ফেইল!
  expect(counter).toBe(1);
  counter++;
  expect(counter).toBe(2);
});

// ✅ ভালো — প্রতিটি টেস্ট স্বাধীন
describe('Counter', () => {
  let counter;

  beforeEach(() => {
    counter = 0; // প্রতিবার fresh
  });

  test('increment once', () => {
    counter++;
    expect(counter).toBe(1);
  });

  test('increment twice', () => {
    counter++;
    counter++;
    expect(counter).toBe(2);
  });
});
```

### ৫. অন্যান্য অ্যান্টি-প্যাটার্নস

```
⚠️ এড়িয়ে চলুন:

• Hidden Test Logic — helper method-এ assert লুকানো
  (টেস্ট পড়ে বোঝা যায় না কী assert হচ্ছে)

• Flaky Tests — কখনো পাস, কখনো ফেইল
  (সাধারণত shared state, timing, বা external dependency-র কারণে)

• Testing Framework/Language Features
  (PHP-র array_map() কাজ করে কিনা টেস্ট করবেন না)

• Assertion Roulette — কোন assertion ফেইল করেছে বোঝা যায় না
  (assertion-এ message যোগ করুন)

• Test Without Assertion — টেস্ট চলে কিন্তু কিছু যাচাই করে না
  (PHPUnit-এ failOnRisky=true দিয়ে ধরা যায়)

• Copy-Paste Test — একই টেস্ট সামান্য পরিবর্তন করে কপি
  (DataProvider / test.each ব্যবহার করুন)
```

---

## 📋 সারসংক্ষেপ

```
ইউনিট টেস্টিং — মূল পাঠ সংক্ষেপে:

┌─────────────────────────────────────────────────────────────────┐
│  ধারণা              │  মূল কথা                                  │
├─────────────────────┼───────────────────────────────────────────┤
│  FIRST নীতি          │  Fast, Independent, Repeatable,          │
│                      │  Self-validating, Timely                  │
├─────────────────────┼───────────────────────────────────────────┤
│  AAA Pattern         │  Arrange → Act → Assert                  │
├─────────────────────┼───────────────────────────────────────────┤
│  Data Provider       │  একই লজিক, ভিন্ন ডেটা — DRY টেস্ট       │
├─────────────────────┼───────────────────────────────────────────┤
│  Edge Cases          │  null, empty, boundary, error paths      │
├─────────────────────┼───────────────────────────────────────────┤
│  SOLID = Testable    │  DI + Interface = সহজে mock করা যায়     │
├─────────────────────┼───────────────────────────────────────────┤
│  Coverage            │  পরিমাণ নয়, গুণমান দেখুন (mutation!)     │
├─────────────────────┼───────────────────────────────────────────┤
│  Behavior Test করুন  │  Implementation নয়, আউটপুট টেস্ট করুন   │
├─────────────────────┼───────────────────────────────────────────┤
│  Test = Doc          │  ভালো টেস্ট = জীবন্ত ডকুমেন্টেশন         │
└─────────────────────┴───────────────────────────────────────────┘

মনে রাখবেন: টেস্ট না থাকা মানে "কোড কাজ করে" এটা আশা করা।
টেস্ট থাকা মানে "কোড কাজ করে" এটা প্রমাণ করা। 🚀
```

---

> **"Legacy code is code without tests."** — Michael Feathers
>
> যে কোডে টেস্ট নেই, সেটাই legacy code — সে যতই নতুন হোক না কেন।
