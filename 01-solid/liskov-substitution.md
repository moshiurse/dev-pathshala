# L — Liskov Substitution Principle (LSP)

> **"যদি S, T এর সাবটাইপ হয়, তাহলে T টাইপের অবজেক্টের জায়গায় S টাইপের অবজেক্ট ব্যবহার করলেও প্রোগ্রামের আচরণ সঠিক থাকবে।"**
> — Barbara Liskov, 1987

---

## 📖 সংজ্ঞা

Liskov Substitution Principle বলে যে, **চাইল্ড ক্লাস (subtype) যেকোনো জায়গায় প্যারেন্ট ক্লাস (base type) এর বিকল্প হিসেবে ব্যবহার করা যাবে** — এবং প্রোগ্রাম কোনো ত্রুটি ছাড়াই সঠিকভাবে কাজ করবে।

সহজ ভাষায়: **চাইল্ড ক্লাস প্যারেন্টের প্রতিশ্রুতি ভাঙবে না।**

### LSP এর নিয়মগুলো:

1. **Preconditions শক্তিশালী করা যাবে না** — চাইল্ড প্যারেন্টের চেয়ে বেশি শর্ত আরোপ করতে পারবে না
2. **Postconditions দুর্বল করা যাবে না** — চাইল্ড প্যারেন্টের চেয়ে কম গ্যারান্টি দিতে পারবে না
3. **Invariants রক্ষা করতে হবে** — প্যারেন্টের নিয়ম চাইল্ডেও বলবত থাকবে
4. **Exception** — চাইল্ড প্যারেন্টের চেয়ে নতুন/ভিন্ন exception throw করবে না

---

## 🏠 বাস্তব জীবনের উদাহরণ

### হাঁসের উদাহরণ (The Duck Test... Kind of)

> "যদি এটি হাঁসের মতো দেখায়, হাঁসের মতো সাঁতার কাটে, হাঁসের মতো ডাকে — কিন্তু ব্যাটারি লাগে, তাহলে আপনার abstraction ভুল!"

একটি আসল হাঁস এবং একটি রাবার হাঁস চিন্তা করুন:
- আসল হাঁস: উড়তে পারে, সাঁতার কাটতে পারে, ডাকতে পারে ✅
- রাবার হাঁস: উড়তে পারে না! ❌

যদি আপনার কোড "সব হাঁস উড়তে পারে" ধরে নেয়, তাহলে রাবার হাঁস দিলে ভেঙে যাবে। **এটাই LSP ভঙ্গ।**

---

## ❌ খারাপ উদাহরণ — LSP ভঙ্গ

### ক্লাসিক Rectangle-Square সমস্যা

### PHP (❌ ভুল)

```php
<?php

class Rectangle
{
    protected float $width;
    protected float $height;

    public function __construct(float $width, float $height)
    {
        $this->width = $width;
        $this->height = $height;
    }

    public function setWidth(float $width): void
    {
        $this->width = $width;
    }

    public function setHeight(float $height): void
    {
        $this->height = $height;
    }

    public function getWidth(): float
    {
        return $this->width;
    }

    public function getHeight(): float
    {
        return $this->height;
    }

    public function area(): float
    {
        return $this->width * $this->height;
    }
}

// ❌ Square, Rectangle এর সাবটাইপ — কিন্তু আচরণ ভিন্ন!
class Square extends Rectangle
{
    public function __construct(float $side)
    {
        parent::__construct($side, $side);
    }

    // ❌ বর্গক্ষেত্রে width বদলালে height ও বদলাতে হয়
    // এটা Rectangle এর আচরণ ভঙ্গ করে
    public function setWidth(float $width): void
    {
        $this->width = $width;
        $this->height = $width; // প্যারেন্টের আচরণ পরিবর্তন!
    }

    public function setHeight(float $height): void
    {
        $this->width = $height; // প্যারেন্টের আচরণ পরিবর্তন!
        $this->height = $height;
    }
}

// ❌ এই ফাংশন Rectangle এর জন্য কাজ করে, কিন্তু Square এ ভেঙে যায়
function testRectangleArea(Rectangle $rectangle): void
{
    $rectangle->setWidth(5);
    $rectangle->setHeight(4);

    $expectedArea = 20; // 5 * 4 = 20

    echo "প্রত্যাশিত: {$expectedArea}\n";
    echo "প্রাপ্ত: {$rectangle->area()}\n";

    // Rectangle দিলে: 5 * 4 = 20 ✅
    // Square দিলে:   4 * 4 = 16 ❌ (setHeight width ও বদলে দিয়েছে!)
    assert($rectangle->area() === $expectedArea, "ক্ষেত্রফল মেলেনি!");
}

$rect = new Rectangle(10, 10);
testRectangleArea($rect); // ✅ কাজ করে: 20

$square = new Square(10);
testRectangleArea($square); // ❌ ফেল! 16 ≠ 20 — LSP ভঙ্গ!
```

### JavaScript (❌ ভুল)

```javascript
class Bird {
    fly() {
        return `${this.constructor.name} উড়ছে 🕊️`;
    }

    eat() {
        return `${this.constructor.name} খাচ্ছে`;
    }
}

class Eagle extends Bird {
    fly() {
        return "ঈগল আকাশে উড়ছে! 🦅";
    }
}

// ❌ পেঙ্গুইন উড়তে পারে না — কিন্তু Bird এর সাবটাইপ হিসেবে fly() আছে
class Penguin extends Bird {
    fly() {
        // ❌ প্যারেন্টের প্রতিশ্রুতি ভঙ্গ!
        throw new Error("পেঙ্গুইন উড়তে পারে না! 🐧");
    }
}

// ❌ এই ফাংশন Bird এর জন্য কাজ করে, কিন্তু Penguin এ ক্র্যাশ করে
function makeBirdFly(bird) {
    console.log(bird.fly()); // Penguin দিলে Error throw করবে!
}

makeBirdFly(new Eagle());   // ✅ কাজ করে
makeBirdFly(new Penguin()); // ❌ Error! LSP ভঙ্গ!
```

### আরো একটি PHP উদাহরণ (❌)

```php
<?php

interface FileStorage
{
    public function read(string $path): string;
    public function write(string $path, string $content): void;
    public function delete(string $path): void;
}

class LocalFileStorage implements FileStorage
{
    public function read(string $path): string
    {
        return file_get_contents($path);
    }

    public function write(string $path, string $content): void
    {
        file_put_contents($path, $content);
    }

    public function delete(string $path): void
    {
        unlink($path);
    }
}

// ❌ ReadOnlyStorage — write ও delete সাপোর্ট করে না
// কিন্তু FileStorage ইন্টারফেস মেনে চলতে বাধ্য
class ReadOnlyStorage implements FileStorage
{
    public function read(string $path): string
    {
        return file_get_contents($path);
    }

    public function write(string $path, string $content): void
    {
        // ❌ প্রতিশ্রুতি ভঙ্গ! ইন্টারফেস বলে write করা যাবে
        throw new RuntimeException("রিড-অনলি স্টোরেজে লেখা যায় না!");
    }

    public function delete(string $path): void
    {
        // ❌ প্রতিশ্রুতি ভঙ্গ!
        throw new RuntimeException("রিড-অনলি স্টোরেজ থেকে ডিলিট করা যায় না!");
    }
}

function backupFile(FileStorage $storage, string $path): void
{
    $content = $storage->read($path);
    $storage->write("backup/{$path}", $content); // ReadOnlyStorage এ ক্র্যাশ!
}
```

---

## ✅ ভালো উদাহরণ — LSP মেনে

### Rectangle-Square সমাধান

### PHP (✅ সঠিক)

```php
<?php

// সমাধান ১: আলাদা hierarchy — কোনো inheritance নেই
interface Shape
{
    public function area(): float;
    public function describe(): string;
}

class Rectangle implements Shape
{
    public function __construct(
        private float $width,
        private float $height
    ) {}

    public function getWidth(): float { return $this->width; }
    public function getHeight(): float { return $this->height; }

    public function area(): float
    {
        return $this->width * $this->height;
    }

    public function describe(): string
    {
        return "আয়তক্ষেত্র ({$this->width} x {$this->height})";
    }
}

class Square implements Shape
{
    public function __construct(
        private float $side
    ) {}

    public function getSide(): float { return $this->side; }

    public function area(): float
    {
        return $this->side ** 2;
    }

    public function describe(): string
    {
        return "বর্গক্ষেত্র ({$this->side})";
    }
}

// ✅ দুটোই Shape — কিন্তু একটি অন্যটির সাবটাইপ নয়
function printArea(Shape $shape): void
{
    echo "{$shape->describe()}: ক্ষেত্রফল = {$shape->area()}\n";
}

printArea(new Rectangle(5, 4)); // আয়তক্ষেত্র (5 x 4): ক্ষেত্রফল = 20 ✅
printArea(new Square(5));        // বর্গক্ষেত্র (5): ক্ষেত্রফল = 25 ✅

// সমাধান ২: Immutable অবজেক্ট ব্যবহার
class ImmutableRectangle
{
    public function __construct(
        public readonly float $width,
        public readonly float $height
    ) {}

    public function area(): float
    {
        return $this->width * $this->height;
    }

    // নতুন অবজেক্ট রিটার্ন করে — পুরনোটা বদলায় না
    public function withWidth(float $width): self
    {
        return new self($width, $this->height);
    }

    public function withHeight(float $height): self
    {
        return new self($this->width, $height);
    }
}

// ✅ Immutable হওয়ায় setter এর সমস্যা নেই
$rect = new ImmutableRectangle(5, 4);
$newRect = $rect->withWidth(10);
echo $rect->area();    // 20 (অপরিবর্তিত)
echo $newRect->area();  // 40
```

### Bird সমস্যার সমাধান

### JavaScript (✅ সঠিক)

```javascript
// ✅ সঠিক hierarchy — ক্ষমতা অনুযায়ী ইন্টারফেস ভাগ
class Bird {
    #name;

    constructor(name) {
        this.#name = name;
    }

    get name() { return this.#name; }

    eat() {
        return `${this.#name} খাচ্ছে`;
    }
}

// আলাদা capability হিসেবে মডেল করা
class FlyingBird extends Bird {
    fly() {
        return `${this.name} উড়ছে 🕊️`;
    }
}

class SwimmingBird extends Bird {
    swim() {
        return `${this.name} সাঁতার কাটছে 🏊`;
    }
}

// কিছু পাখি উড়তেও পারে, সাঁতারও কাটতে পারে
class FlyingSwimmingBird extends Bird {
    fly() {
        return `${this.name} উড়ছে 🕊️`;
    }

    swim() {
        return `${this.name} সাঁতার কাটছে 🏊`;
    }
}

class Eagle extends FlyingBird {
    fly() {
        return `ঈগল আকাশে উড়ছে! 🦅`;
    }
}

class Penguin extends SwimmingBird {
    swim() {
        return `পেঙ্গুইন বরফের পানিতে সাঁতার কাটছে! 🐧`;
    }
}

class Duck extends FlyingSwimmingBird {
    fly() { return `হাঁস উড়ছে! 🦆`; }
    swim() { return `হাঁস সাঁতার কাটছে! 🦆`; }
}

// ✅ প্রতিটি ফাংশন সঠিক টাইপ গ্রহণ করে
function makeFly(bird) {
    // শুধু FlyingBird বা FlyingSwimmingBird আসবে
    console.log(bird.fly());
}

function makeSwim(bird) {
    console.log(bird.swim());
}

makeFly(new Eagle());    // ✅ ঈগল উড়তে পারে
makeFly(new Duck());     // ✅ হাঁস উড়তে পারে
makeSwim(new Penguin()); // ✅ পেঙ্গুইন সাঁতার কাটতে পারে
makeSwim(new Duck());    // ✅ হাঁস সাঁতারও কাটতে পারে
// makeFly(new Penguin()) — এটি করা উচিত নয়, Penguin FlyingBird না
```

### FileStorage সমাধান

### PHP (✅ সঠিক)

```php
<?php

// ✅ ইন্টারফেস আলাদা করুন (ISP ও LSP একসাথে)
interface Readable
{
    public function read(string $path): string;
}

interface Writable
{
    public function write(string $path, string $content): void;
}

interface Deletable
{
    public function delete(string $path): void;
}

// ফুল স্টোরেজ — সব পারে
class LocalFileStorage implements Readable, Writable, Deletable
{
    public function read(string $path): string
    {
        return file_get_contents($path);
    }

    public function write(string $path, string $content): void
    {
        file_put_contents($path, $content);
    }

    public function delete(string $path): void
    {
        unlink($path);
    }
}

// রিড-অনলি স্টোরেজ — শুধু Readable ইমপ্লিমেন্ট করে
class ReadOnlyStorage implements Readable
{
    public function read(string $path): string
    {
        return file_get_contents($path);
    }
    // ✅ write() ও delete() নেই — কারণ এই ক্লাস সেটা প্রমিজ করে না!
}

// ✅ ফাংশন সঠিক টাইপ ব্যবহার করে
function readFile(Readable $storage, string $path): string
{
    return $storage->read($path); // ✅ সব Readable এ কাজ করবে
}

function backupFile(Readable $source, Writable $destination, string $path): void
{
    $content = $source->read($path);
    $destination->write("backup/{$path}", $content);
    // ✅ ReadOnlyStorage কখনো $destination হিসেবে আসবে না
}
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### LSP এর ৪টি শর্ত বিস্তারিত

#### ১. Precondition শক্তিশালী করা যাবে না

```php
<?php

class PaymentGateway
{
    public function charge(float $amount): void
    {
        // প্যারেন্ট: amount > 0 হলেই কাজ করবে
        if ($amount <= 0) {
            throw new InvalidArgumentException("পরিমাণ ধনাত্মক হতে হবে");
        }
        // চার্জ করো...
    }
}

// ❌ চাইল্ড বেশি শর্ত আরোপ করছে (precondition strengthening)
class PremiumPaymentGateway extends PaymentGateway
{
    public function charge(float $amount): void
    {
        // ❌ নতুন শর্ত: minimum ১০০ টাকা!
        // প্যারেন্ট ৫০ টাকায় কাজ করতো, চাইল্ড করবে না
        if ($amount < 100) {
            throw new InvalidArgumentException("ন্যূনতম ১০০ টাকা!");
        }
        parent::charge($amount);
    }
}

// ✅ সঠিক — precondition একই বা শিথিল
class FlexiblePaymentGateway extends PaymentGateway
{
    public function charge(float $amount): void
    {
        // ✅ প্যারেন্টের শর্ত বজায়: amount > 0
        // অতিরিক্ত শর্ত আরোপ করেনি
        parent::charge($amount);
        // ক্যাশব্যাক দেওয়ার লজিক...
    }
}
```

#### ২. Postcondition দুর্বল করা যাবে না

```javascript
class Sorter {
    sort(array) {
        // postcondition: রিটার্ন করা অ্যারে ছোট থেকে বড় সর্ট করা থাকবে
        return [...array].sort((a, b) => a - b);
    }
}

// ❌ postcondition দুর্বল — সব সময় সর্ট করা নাও থাকতে পারে
class LazySorter extends Sorter {
    sort(array) {
        if (array.length > 1000) {
            return array; // ❌ বড় অ্যারে সর্ট না করেই রিটার্ন!
        }
        return super.sort(array);
    }
}

// ✅ postcondition একই — সবসময় সর্ট করা অ্যারে রিটার্ন
class QuickSorter extends Sorter {
    sort(array) {
        // ভিন্ন অ্যালগরিদম কিন্তু একই ফলাফল (সর্ট করা অ্যারে)
        if (array.length <= 1) return [...array];
        const pivot = array[0];
        const left = array.slice(1).filter(x => x <= pivot);
        const right = array.slice(1).filter(x => x > pivot);
        return [...this.sort(left), pivot, ...this.sort(right)];
    }
}
```

#### ৩. Invariant রক্ষা করতে হবে

```php
<?php

class BankAccount
{
    protected float $balance;

    public function __construct(float $initialBalance)
    {
        $this->balance = $initialBalance;
    }

    // invariant: ব্যালেন্স কখনো নেগেটিভ হবে না
    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new RuntimeException("ব্যালেন্স অপর্যাপ্ত");
        }
        $this->balance -= $amount;
    }

    public function getBalance(): float
    {
        return $this->balance;
    }
}

// ❌ invariant ভঙ্গ — ব্যালেন্স নেগেটিভ হতে পারে
class CreditAccount extends BankAccount
{
    public function withdraw(float $amount): void
    {
        // ❌ ব্যালেন্সের চেয়ে বেশি তোলা যাচ্ছে!
        // প্যারেন্টের invariant: balance >= 0 ভেঙে গেছে
        $this->balance -= $amount;
    }
}

// ✅ invariant রক্ষা — নেগেটিভ ব্যালেন্স একটি সীমা পর্যন্ত
class OverdraftAccount extends BankAccount
{
    private float $overdraftLimit;

    public function __construct(float $initialBalance, float $overdraftLimit)
    {
        parent::__construct($initialBalance);
        $this->overdraftLimit = $overdraftLimit;
    }

    public function withdraw(float $amount): void
    {
        // ✅ ওভারড্রাফ্ট সীমা পর্যন্ত অনুমোদিত
        if ($amount > ($this->balance + $this->overdraftLimit)) {
            throw new RuntimeException("ওভারড্রাফ্ট সীমা অতিক্রম");
        }
        $this->balance -= $amount;
    }
}
```

#### ৪. Exception সামঞ্জস্য

```javascript
class DataParser {
    parse(data) {
        try {
            return JSON.parse(data);
        } catch (e) {
            throw new SyntaxError(`পার্স করা যায়নি: ${e.message}`);
        }
    }
}

// ❌ ভিন্ন ধরনের exception throw করছে
class StrictDataParser extends DataParser {
    parse(data) {
        if (typeof data !== 'string') {
            // ❌ TypeError — প্যারেন্ট এই ধরনের error throw করে না
            throw new TypeError("শুধুমাত্র স্ট্রিং গ্রহণযোগ্য");
        }
        return super.parse(data);
    }
}

// ✅ একই ধরনের exception
class LenientDataParser extends DataParser {
    parse(data) {
        if (typeof data !== 'string') {
            // ✅ SyntaxError — প্যারেন্টের মতো একই ধরন
            throw new SyntaxError("অবৈধ ডাটা ফরম্যাট");
        }
        try {
            return JSON.parse(data);
        } catch (e) {
            // ফলব্যাক: ডিফল্ট ভ্যালু রিটার্ন (postcondition দুর্বল করেনি — null valid return)
            return null;
        }
    }
}
```

---

### "Is-A" vs "Behaves-Like-A"

```
❌ ভুল চিন্তা:                    ✅ সঠিক চিন্তা:
"Square IS-A Rectangle"           "Square BEHAVES-LIKE-A Rectangle?"
(গাণিতিকভাবে সত্য)               (কোডে কি সত্যিই সাবস্টিটিউট করা যায়?)

গণিত ≠ কোড!
গণিতে বর্গক্ষেত্র আয়তক্ষেত্রের
বিশেষ রূপ, কিন্তু কোডে তাদের
আচরণ ভিন্ন!
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **নির্ভরযোগ্য পলিমরফিজম** | যেকোনো সাবটাইপ নিরাপদে ব্যবহার করা যায় |
| ২ | **কম বাগ** | অপ্রত্যাশিত আচরণ থেকে রক্ষা |
| ৩ | **সহজ টেস্টিং** | প্যারেন্টের টেস্ট চাইল্ডেও পাস করবে |
| ৪ | **OCP সাপোর্ট** | সঠিক substitution না হলে OCP কাজ করবে না |
| ৫ | **আত্মবিশ্বাস** | কোড রিভিউতে সাবটাইপ নিয়ে চিন্তা কমে |
| ৬ | **সঠিক inheritance** | শুধু "is-a" নয়, "behaves-like-a" নিশ্চিত করে |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **Hierarchy ডিজাইন কঠিন** | সঠিক abstraction খুঁজে পাওয়া কঠিন |
| ২ | **Inheritance সীমাবদ্ধ** | অনেক সময় inheritance না করে composition ভালো |
| ৩ | **অতিরিক্ত চিন্তা** | ছোট প্রজেক্টে প্রতিটি subtype নিয়ে বেশি ভাবতে হয় |
| ৪ | **ডকুমেন্টেশন দরকার** | কন্ট্র্যাক্ট (pre/post condition) স্পষ্ট করে লিখতে হয় |
| ৫ | **রিফ্যাক্টরিং** | LSP ভঙ্গ ধরা পড়লে hierarchy পুনর্গঠন দরকার |

---

## 🧪 LSP টেস্ট করার কৌশল

### "প্যারেন্ট টেস্ট" নীতি

> প্যারেন্ট ক্লাসের সব টেস্ট চাইল্ড ক্লাসেও পাস করতে হবে।

```php
<?php

// PHP (PHPUnit) — বেস টেস্ট ক্লাস
abstract class ShapeTestCase extends TestCase
{
    abstract protected function createShape(): Shape;

    public function test_area_is_positive(): void
    {
        $shape = $this->createShape();
        $this->assertGreaterThan(0, $shape->area());
    }

    public function test_area_returns_float(): void
    {
        $shape = $this->createShape();
        $this->assertIsFloat($shape->area());
    }

    public function test_describe_returns_string(): void
    {
        $shape = $this->createShape();
        $this->assertIsString($shape->describe());
        $this->assertNotEmpty($shape->describe());
    }
}

// ✅ প্রতিটি সাবটাইপ একই টেস্ট পাস করবে
class RectangleTest extends ShapeTestCase
{
    protected function createShape(): Shape
    {
        return new Rectangle(5, 4);
    }
}

class SquareTest extends ShapeTestCase
{
    protected function createShape(): Shape
    {
        return new Square(5);
    }
}

class CircleTest extends ShapeTestCase
{
    protected function createShape(): Shape
    {
        return new Circle(3);
    }
}
```

```javascript
// Jest — শেয়ার্ড টেস্ট
function testShapeBehavior(createShape) {
    test('area ধনাত্মক হতে হবে', () => {
        const shape = createShape();
        expect(shape.area()).toBeGreaterThan(0);
    });

    test('area সংখ্যা রিটার্ন করবে', () => {
        const shape = createShape();
        expect(typeof shape.area()).toBe('number');
    });

    test('describe স্ট্রিং রিটার্ন করবে', () => {
        const shape = createShape();
        expect(typeof shape.describe()).toBe('string');
    });
}

// ✅ সব সাবটাইপে একই টেস্ট চলে
describe('Rectangle', () => testShapeBehavior(() => new Rectangle(5, 4)));
describe('Square', () => testShapeBehavior(() => new Square(5)));
describe('Circle', () => testShapeBehavior(() => new Circle(3)));
```

---

## 📏 কখন LSP প্রয়োগ করবেন / LSP ভঙ্গের লক্ষণ

### ⚠️ LSP ভঙ্গের লক্ষণ:

- চাইল্ড ক্লাসে **Exception throw** করা হচ্ছে যেখানে প্যারেন্ট করে না
- চাইল্ড মেথডে **empty body** বা `return null` দেওয়া হচ্ছে
- কোডে **instanceof** বা **type check** করে ভিন্ন আচরণ
- চাইল্ড ক্লাসে প্যারেন্টের কিছু মেথড **অর্থহীন** হয়ে যাচ্ছে
- ইউনিট টেস্ট **প্যারেন্টে পাস** কিন্তু **চাইল্ডে ফেল**

### ✅ সমাধান কৌশল:

1. **Inheritance বাদ দিন, Composition ব্যবহার করুন**
2. **ইন্টারফেস ভেঙে ছোট করুন** (ISP)
3. **"Is-A" না, "Behaves-Like-A" চিন্তা করুন**
4. **Contract/Invariant স্পষ্টভাবে ডকুমেন্ট করুন**

---

## 🔗 অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক

```
LSP ──► OCP: সাবটাইপ সঠিক না হলে পলিমরফিজম ভেঙে যায়, OCP কাজ করবে না
LSP ──► ISP: বড় ইন্টারফেস ভেঙে ছোট করলে LSP ভঙ্গের সম্ভাবনা কমে
LSP ──► DIP: abstraction এর উপর নির্ভর করলে LSP গুরুত্বপূর্ণ হয়ে ওঠে
```

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **মূল ধারণা** | চাইল্ড ক্লাস প্যারেন্টের জায়গায় নিরাপদে ব্যবহারযোগ্য |
| **চেক করুন** | "প্যারেন্টের সব টেস্ট কি চাইল্ডে পাস করবে?" |
| **লক্ষণ** | instanceof check, empty methods, unexpected exceptions |
| **সমাধান** | Composition > Inheritance, ছোট interface |
| **মনে রাখুন** | "Is-A" (গণিত) ≠ "Behaves-Like-A" (কোড) |
