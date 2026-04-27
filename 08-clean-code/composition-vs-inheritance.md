# 🧩 কম্পোজিশন বনাম ইনহেরিটেন্স (Composition vs Inheritance)

> "অবজেক্ট কম্পোজিশনকে ক্লাস ইনহেরিটেন্সের চেয়ে বেশি প্রাধান্য দিন।"
> — Gang of Four (GoF), Design Patterns (1994)

---

## 📑 সূচিপত্র

1. [ইনহেরিটেন্স পুনরালোচনা](#1--ইনহেরিটেন্স-পুনরালোচনা)
2. [কম্পোজিশন পুনরালোচনা](#2--কম্পোজিশন-পুনরালোচনা)
3. [ইনহেরিটেন্সের সমস্যা](#3--ইনহেরিটেন্সের-সমস্যা)
4. [কম্পোজিশন দিয়ে সমাধান](#4--কম্পোজিশন-দিয়ে-সমাধান)
5. ["Favor Composition over Inheritance"](#5--favor-composition-over-inheritance)
6. [হাইব্রিড অ্যাপ্রোচ](#6--হাইব্রিড-অ্যাপ্রোচ)
7. [রিয়েল-ওয়ার্ল্ড কেস স্টাডি: নোটিফিকেশন সিস্টেম](#7--নোটিফিকেশন-সিস্টেম)
8. [রিয়েল-ওয়ার্ল্ড কেস স্টাডি: গেম ক্যারেক্টার সিস্টেম](#8--গেম-ক্যারেক্টার-সিস্টেম)
9. [তুলনা সারণী](#9--তুলনা-সারণী)
10. [সিদ্ধান্ত ফ্লোচার্ট](#10--সিদ্ধান্ত-ফ্লোচার্ট)
11. [সম্পর্কিত টপিক ও রিসোর্স](#11--সম্পর্কিত-টপিক)

---

## 1. 🔄 ইনহেরিটেন্স পুনরালোচনা

### 📖 সংজ্ঞা

**ইনহেরিটেন্স (Inheritance)** হলো অবজেক্ট-ওরিয়েন্টেড প্রোগ্রামিংয়ের একটি মৌলিক ধারণা যেখানে একটি ক্লাস (চাইল্ড/সাবক্লাস) অন্য একটি ক্লাস (প্যারেন্ট/সুপারক্লাস) থেকে প্রপার্টি ও মেথড উত্তরাধিকার সূত্রে পায়। এটি **"is-a"** সম্পর্ক প্রকাশ করে।

### 🏠 বাস্তব জীবনের উপমা

একটি পরিবারের কথা ভাবুন:
- 👴 দাদু (Grandparent) → মৌলিক বৈশিষ্ট্য: চোখের রং, উচ্চতা
- 👨 বাবা (Parent) → দাদুর বৈশিষ্ট্য + নিজের কিছু বৈশিষ্ট্য
- 👦 আপনি (Child) → বাবার + দাদুর বৈশিষ্ট্য + নিজের বৈশিষ্ট্য

কিন্তু সমস্যা: আপনি দাদুর চোখের রং চাইলেও তাঁর ডায়াবেটিসও "উত্তরাধিকার" পেতে পারেন! 😅

### ✅ ইনহেরিটেন্সের ভালো উদাহরণ (PHP)

```php
<?php
// ✅ সঠিক ব্যবহার - সত্যিকারের "is-a" সম্পর্ক

abstract class Shape
{
    protected string $color;

    public function __construct(string $color)
    {
        $this->color = $color;
    }

    // প্রতিটি শেপের নিজস্ব এরিয়া হিসাব
    abstract public function calculateArea(): float;

    public function getColor(): string
    {
        return $this->color;
    }
}

class Circle extends Shape
{
    public function __construct(
        private float $radius,
        string $color
    ) {
        parent::__construct($color);
    }

    public function calculateArea(): float
    {
        return M_PI * $this->radius ** 2;
    }
}

class Rectangle extends Shape
{
    public function __construct(
        private float $width,
        private float $height,
        string $color
    ) {
        parent::__construct($color);
    }

    public function calculateArea(): float
    {
        return $this->width * $this->height;
    }
}

// ব্যবহার
$circle = new Circle(5, 'লাল');
echo $circle->calculateArea(); // 78.54
echo $circle->getColor();      // লাল
```

### ✅ ইনহেরিটেন্সের ভালো উদাহরণ (JavaScript)

```javascript
// ✅ সঠিক ব্যবহার - সত্যিকারের "is-a" সম্পর্ক

class Shape {
    constructor(color) {
        if (new.target === Shape) {
            throw new Error('Shape সরাসরি ইনস্ট্যান্স করা যাবে না');
        }
        this.color = color;
    }

    // প্রতিটি শেপ এটি ওভাররাইড করবে
    calculateArea() {
        throw new Error('calculateArea() ইমপ্লিমেন্ট করতে হবে');
    }

    getColor() {
        return this.color;
    }
}

class Circle extends Shape {
    constructor(radius, color) {
        super(color);
        this.radius = radius;
    }

    calculateArea() {
        return Math.PI * this.radius ** 2;
    }
}

class Rectangle extends Shape {
    constructor(width, height, color) {
        super(color);
        this.width = width;
        this.height = height;
    }

    calculateArea() {
        return this.width * this.height;
    }
}

// ব্যবহার
const circle = new Circle(5, 'লাল');
console.log(circle.calculateArea()); // 78.54
console.log(circle.getColor());      // লাল
```

### ⚠️ ইনহেরিটেন্সের মূল ধারণাগুলো

| ধারণা | ব্যাখ্যা |
|-------|----------|
| **is-a সম্পর্ক** | "Circle is a Shape" — সার্কেল একটি শেপ |
| **Single Inheritance** | PHP/JS-এ একটি ক্লাস শুধু একটি ক্লাস থেকে extend করতে পারে |
| **Method Overriding** | চাইল্ড ক্লাস প্যারেন্টের মেথড পুনরায় সংজ্ঞায়িত করতে পারে |
| **Abstract Class** | সরাসরি ইনস্ট্যান্স করা যায় না, শুধু extend করা যায় |
| **Protected Access** | শুধু নিজের ক্লাস ও চাইল্ড ক্লাস থেকে অ্যাক্সেসযোগ্য |

### 💎 Diamond Problem (ডায়মন্ড সমস্যা)

```
        A
       / \
      B   C
       \ /
        D
```

ক্লাস D যদি B এবং C উভয় থেকে ইনহেরিট করে, আর B ও C উভয়ই A থেকে ইনহেরিট করে,
তাহলে D কোন ভার্সনের মেথড ব্যবহার করবে? এটাই **Diamond Problem**।

> 💡 PHP ও JavaScript-এ Multiple Inheritance নেই, তাই সরাসরি Diamond Problem হয় না।
> তবে PHP-তে **Traits** এবং JS-এ **Mixins** ব্যবহারে একই ধরনের সমস্যা দেখা দিতে পারে।

### 🔥 Fragile Base Class Problem

**Fragile Base Class** সমস্যা তখন হয় যখন বেস ক্লাসে সামান্য পরিবর্তন করলে সকল সাবক্লাসে অপ্রত্যাশিত সমস্যা দেখা দেয়।

```php
<?php
// ❌ Fragile Base Class সমস্যা

class Collection
{
    protected array $items = [];
    protected int $addCount = 0;

    public function add(mixed $item): void
    {
        $this->items[] = $item;
        $this->addCount++;
    }

    public function addAll(array $items): void
    {
        // অভ্যন্তরীণভাবে add() কল করে
        foreach ($items as $item) {
            $this->add($item);
        }
    }

    public function getAddCount(): int
    {
        return $this->addCount;
    }
}

class CountingCollection extends Collection
{
    private int $myCount = 0;

    public function add(mixed $item): void
    {
        $this->myCount++;
        parent::add($item);
    }

    public function addAll(array $items): void
    {
        $this->myCount += count($items);
        parent::addAll($items); // 😱 এটি আবার add() কল করবে!
    }

    public function getMyCount(): int
    {
        return $this->myCount; // ❌ ডাবল কাউন্ট হবে!
    }
}

$col = new CountingCollection();
$col->addAll(['a', 'b', 'c']);
echo $col->getMyCount(); // আশা করেছিলাম 3, পেলাম 6! 😱
```

> ⚠️ এই সমস্যাটি **Java-র HashSet** ক্লাসেও বিখ্যাত। Joshua Bloch "Effective Java" বইতে এটি বিস্তারিত আলোচনা করেছেন।

---

## 2. 🧱 কম্পোজিশন পুনরালোচনা

### 📖 সংজ্ঞা

**কম্পোজিশন (Composition)** হলো এমন একটি ডিজাইন টেকনিক যেখানে একটি ক্লাস অন্য ক্লাসের **ইনস্ট্যান্সকে নিজের ফিল্ড** হিসেবে ধারণ করে, ইনহেরিট না করে। এটি **"has-a"** সম্পর্ক প্রকাশ করে।

### 🏠 বাস্তব জীবনের উপমা

একটি গাড়ির কথা ভাবুন:
- 🚗 গাড়ি (Car) **has-a** ইঞ্জিন (Engine)
- 🚗 গাড়ি **has-a** চাকা (Wheel) × 4
- 🚗 গাড়ি **has-a** স্টিয়ারিং (Steering)

গাড়ি ইঞ্জিন **নয়** (is-not-a), গাড়ির **আছে** (has-a) ইঞ্জিন।
আপনি ইঞ্জিন বদলাতে পারেন গাড়ি না বদলিয়ে! এটাই কম্পোজিশনের শক্তি। 💪

### ✅ কম্পোজিশন উদাহরণ (PHP)

```php
<?php
// ✅ কম্পোজিশন - "has-a" সম্পর্ক

// ইন্টারফেস দিয়ে কন্ট্র্যাক্ট সংজ্ঞায়িত
interface Engine
{
    public function start(): string;
    public function getHorsepower(): int;
}

// বিভিন্ন ইঞ্জিন ইমপ্লিমেন্টেশন
class PetrolEngine implements Engine
{
    public function start(): string
    {
        return '🔥 পেট্রোল ইঞ্জিন চালু হলো — ভ্রুম ভ্রুম!';
    }

    public function getHorsepower(): int
    {
        return 200;
    }
}

class ElectricEngine implements Engine
{
    public function start(): string
    {
        return '⚡ ইলেকট্রিক ইঞ্জিন চালু হলো — শুউউউ!';
    }

    public function getHorsepower(): int
    {
        return 300;
    }
}

// কম্পোজিশন ব্যবহার — গাড়ির "আছে" ইঞ্জিন
class Car
{
    public function __construct(
        private string $model,
        private Engine $engine // 👈 Dependency Injection (DI)
    ) {}

    public function drive(): string
    {
        $engineStatus = $this->engine->start();
        $hp = $this->engine->getHorsepower();
        return "{$this->model}: {$engineStatus} ({$hp} HP)";
    }

    // রানটাইমে ইঞ্জিন পরিবর্তন সম্ভব!
    public function swapEngine(Engine $newEngine): void
    {
        $this->engine = $newEngine;
    }
}

// ব্যবহার
$car = new Car('টয়োটা করোলা', new PetrolEngine());
echo $car->drive();
// টয়োটা করোলা: 🔥 পেট্রোল ইঞ্জিন চালু হলো — ভ্রুম ভ্রুম! (200 HP)

// রানটাইমে ইঞ্জিন বদলানো
$car->swapEngine(new ElectricEngine());
echo $car->drive();
// টয়োটা করোলা: ⚡ ইলেকট্রিক ইঞ্জিন চালু হলো — শুউউউ! (300 HP)
```

### ✅ কম্পোজিশন উদাহরণ (JavaScript)

```javascript
// ✅ কম্পোজিশন - "has-a" সম্পর্ক

// ইঞ্জিন ক্লাসসমূহ
class PetrolEngine {
    start() {
        return '🔥 পেট্রোল ইঞ্জিন চালু হলো — ভ্রুম ভ্রুম!';
    }

    getHorsepower() {
        return 200;
    }
}

class ElectricEngine {
    start() {
        return '⚡ ইলেকট্রিক ইঞ্জিন চালু হলো — শুউউউ!';
    }

    getHorsepower() {
        return 300;
    }
}

// কম্পোজিশন ব্যবহার
class Car {
    #model;
    #engine;

    constructor(model, engine) {
        this.#model = model;
        this.#engine = engine; // 👈 Dependency Injection
    }

    drive() {
        const engineStatus = this.#engine.start();
        const hp = this.#engine.getHorsepower();
        return `${this.#model}: ${engineStatus} (${hp} HP)`;
    }

    // রানটাইমে ইঞ্জিন পরিবর্তন সম্ভব!
    swapEngine(newEngine) {
        this.#engine = newEngine;
    }
}

// ব্যবহার
const car = new Car('টয়োটা করোলা', new PetrolEngine());
console.log(car.drive());

// রানটাইমে ইঞ্জিন বদলানো
car.swapEngine(new ElectricEngine());
console.log(car.drive());
```

### 🎯 Strategy Pattern — কম্পোজিশনের শক্তিশালী প্রয়োগ

Strategy Pattern হলো কম্পোজিশনের সবচেয়ে পরিচিত প্রয়োগ। এটি আলগোরিদমকে **ইন্টারচেঞ্জেবল** করে তোলে।

```php
<?php
// Strategy Pattern — পেমেন্ট প্রসেসিং

interface PaymentStrategy
{
    public function pay(float $amount): string;
}

class BkashPayment implements PaymentStrategy
{
    public function pay(float $amount): string
    {
        return "বিকাশে ৳{$amount} পেমেন্ট সফল ✅";
    }
}

class NagadPayment implements PaymentStrategy
{
    public function pay(float $amount): string
    {
        return "নগদে ৳{$amount} পেমেন্ট সফল ✅";
    }
}

class CardPayment implements PaymentStrategy
{
    public function pay(float $amount): string
    {
        return "কার্ডে ৳{$amount} পেমেন্ট সফল ✅";
    }
}

class Order
{
    private PaymentStrategy $paymentMethod;

    public function setPaymentMethod(PaymentStrategy $method): void
    {
        $this->paymentMethod = $method;
    }

    public function checkout(float $total): string
    {
        return $this->paymentMethod->pay($total);
    }
}

// ব্যবহার — রানটাইমে পেমেন্ট মেথড পরিবর্তন
$order = new Order();
$order->setPaymentMethod(new BkashPayment());
echo $order->checkout(500); // বিকাশে ৳500 পেমেন্ট সফল ✅

$order->setPaymentMethod(new NagadPayment());
echo $order->checkout(500); // নগদে ৳500 পেমেন্ট সফল ✅
```

---

## 3. 💥 ইনহেরিটেন্সের সমস্যা

### 🦍🍌 Gorilla-Banana Problem (গরিলা-কলা সমস্যা)

> "আমি শুধু একটি কলা চেয়েছিলাম, কিন্তু পেলাম একটি গরিলা যে কলা ধরে আছে, আর সাথে পুরো জঙ্গল!"
> — Joe Armstrong (Erlang ভাষার স্রষ্টা)

এই সমস্যা তখন হয় যখন আপনি প্যারেন্ট ক্লাসের একটি ছোট ফিচার চান, কিন্তু ইনহেরিটেন্সে পুরো ক্লাসের সব কিছু নিতে হয়।

```php
<?php
// ❌ গরিলা-কলা সমস্যা

class Animal
{
    public function eat(): void { /* ... */ }
    public function sleep(): void { /* ... */ }
    public function breathe(): void { /* ... */ }
    public function reproduce(): void { /* ... */ }
    public function move(): void { /* ... */ }
    // ... আরো ২০টি মেথড
}

class Mammal extends Animal
{
    public function produceMilk(): void { /* ... */ }
    public function regulateTemperature(): void { /* ... */ }
    // ... আরো ১০টি মেথড
}

class Dog extends Mammal
{
    public function bark(): void { /* ... */ }
    public function fetch(): void { /* ... */ }
}

// আমি শুধু bark() চেয়েছিলাম,
// কিন্তু Animal + Mammal-এর সব মেথড এসে গেলো! 😱
$dog = new Dog();
```

### 💣 Subclass Explosion (সাবক্লাস বিস্ফোরণ) — Shape উদাহরণ

#### ❌ ইনহেরিটেন্স দিয়ে ভুল পদ্ধতি (PHP)

```php
<?php
// ❌ সাবক্লাস বিস্ফোরণ — Shape + Color + Border কম্বিনেশন

abstract class Shape
{
    abstract public function draw(): string;
}

// রং অনুযায়ী সাবক্লাস
class RedCircle extends Shape
{
    public function draw(): string { return '🔴 লাল বৃত্ত'; }
}

class BlueCircle extends Shape
{
    public function draw(): string { return '🔵 নীল বৃত্ত'; }
}

class GreenCircle extends Shape
{
    public function draw(): string { return '🟢 সবুজ বৃত্ত'; }
}

class RedRectangle extends Shape
{
    public function draw(): string { return '🟥 লাল আয়তক্ষেত্র'; }
}

class BlueRectangle extends Shape
{
    public function draw(): string { return '🟦 নীল আয়তক্ষেত্র'; }
}

class GreenRectangle extends Shape
{
    public function draw(): string { return '🟩 সবুজ আয়তক্ষেত্র'; }
}

// 😱 বর্ডার যোগ করলে?
// RedCircleWithBorder, BlueCircleWithBorder, GreenCircleWithBorder...
// RedRectangleWithBorder, BlueRectangleWithBorder, GreenRectangleWithBorder...
// 3 রং × 2 শেপ × 2 বর্ডার = 12টি ক্লাস!
// 10 রং × 5 শেপ × 3 বর্ডার = 150টি ক্লাস!! 🤯
```

#### ❌ ইনহেরিটেন্স দিয়ে ভুল পদ্ধতি (JavaScript)

```javascript
// ❌ সাবক্লাস বিস্ফোরণ — একই সমস্যা JavaScript-এ

class Shape {
    draw() { throw new Error('ইমপ্লিমেন্ট করুন'); }
}

class RedCircle extends Shape {
    draw() { return '🔴 লাল বৃত্ত'; }
}

class BlueCircle extends Shape {
    draw() { return '🔵 নীল বৃত্ত'; }
}

class GreenCircle extends Shape {
    draw() { return '🟢 সবুজ বৃত্ত'; }
}

class RedRectangle extends Shape {
    draw() { return '🟥 লাল আয়তক্ষেত্র'; }
}

class BlueRectangle extends Shape {
    draw() { return '🟦 নীল আয়তক্ষেত্র'; }
}

// 😱 সাবক্লাস বিস্ফোরণ অনিবার্য!
// নতুন প্রপার্টি যোগ = ক্লাস সংখ্যা গুণিতক হারে বাড়ে
```

### 🔗 টাইট কাপলিং (Tight Coupling) সমস্যা

```
  ❌ গভীর ইনহেরিটেন্স হায়ারার্কি:

  BaseEntity
      ↓
  TimestampedEntity
      ↓
  SoftDeletableEntity
      ↓
  AuditableEntity
      ↓
  User ← 😱 ৪ স্তর গভীর! যেকোনো স্তরে পরিবর্তন = সব ভেঙে যায়!
```

> 🎯 **নিয়ম**: ইনহেরিটেন্স হায়ারার্কি ২-৩ স্তরের বেশি হওয়া উচিত নয়।

---

## 4. ✅ কম্পোজিশন দিয়ে সমাধান

### 🎨 Shape সমস্যার সমাধান — কম্পোজিশন (PHP)

```php
<?php
// ✅ কম্পোজিশন দিয়ে সমাধান

interface Color
{
    public function fill(): string;
}

interface Border
{
    public function apply(): string;
}

// রং ইমপ্লিমেন্টেশন
class Red implements Color
{
    public function fill(): string { return 'লাল'; }
}

class Blue implements Color
{
    public function fill(): string { return 'নীল'; }
}

class Green implements Color
{
    public function fill(): string { return 'সবুজ'; }
}

// বর্ডার ইমপ্লিমেন্টেশন
class SolidBorder implements Border
{
    public function apply(): string { return 'সলিড বর্ডার'; }
}

class DashedBorder implements Border
{
    public function apply(): string { return 'ড্যাশড বর্ডার'; }
}

class NoBorder implements Border
{
    public function apply(): string { return 'বর্ডার নেই'; }
}

// শেপ — কম্পোজিশন ব্যবহার
class Circle
{
    public function __construct(
        private float $radius,
        private Color $color,
        private Border $border
    ) {}

    public function draw(): string
    {
        return sprintf(
            '⭕ বৃত্ত (ব্যাসার্ধ: %.1f, রং: %s, %s)',
            $this->radius,
            $this->color->fill(),
            $this->border->apply()
        );
    }
}

class RectangleShape
{
    public function __construct(
        private float $width,
        private float $height,
        private Color $color,
        private Border $border
    ) {}

    public function draw(): string
    {
        return sprintf(
            '⬜ আয়তক্ষেত্র (%.1f×%.1f, রং: %s, %s)',
            $this->width,
            $this->height,
            $this->color->fill(),
            $this->border->apply()
        );
    }
}

// ব্যবহার — যেকোনো কম্বিনেশন সম্ভব!
$redCircleSolid = new Circle(5, new Red(), new SolidBorder());
echo $redCircleSolid->draw();
// ⭕ বৃত্ত (ব্যাসার্ধ: 5.0, রং: লাল, সলিড বর্ডার)

$blueRectDashed = new RectangleShape(10, 5, new Blue(), new DashedBorder());
echo $blueRectDashed->draw();
// ⬜ আয়তক্ষেত্র (10.0×5.0, রং: নীল, ড্যাশড বর্ডার)

// 🎉 মাত্র 3 + 3 + 2 = 8টি ক্লাস দিয়ে সব কম্বিনেশন!
// ইনহেরিটেন্সে লাগতো 3 × 2 × 3 = 18টি ক্লাস!
```

### 🎨 Shape সমাধান (JavaScript)

```javascript
// ✅ কম্পোজিশন দিয়ে সমাধান — JavaScript

// রং ফ্যাক্টরি ফাংশন
const createRed = () => ({ fill: () => 'লাল' });
const createBlue = () => ({ fill: () => 'নীল' });
const createGreen = () => ({ fill: () => 'সবুজ' });

// বর্ডার ফ্যাক্টরি ফাংশন
const createSolidBorder = () => ({ apply: () => 'সলিড বর্ডার' });
const createDashedBorder = () => ({ apply: () => 'ড্যাশড বর্ডার' });
const createNoBorder = () => ({ apply: () => 'বর্ডার নেই' });

// শেপ — কম্পোজিশন
class Circle {
    constructor(radius, color, border) {
        this.radius = radius;
        this.color = color;
        this.border = border;
    }

    draw() {
        return `⭕ বৃত্ত (ব্যাসার্ধ: ${this.radius}, রং: ${this.color.fill()}, ${this.border.apply()})`;
    }
}

class RectangleShape {
    constructor(width, height, color, border) {
        this.width = width;
        this.height = height;
        this.color = color;
        this.border = border;
    }

    draw() {
        return `⬜ আয়তক্ষেত্র (${this.width}×${this.height}, রং: ${this.color.fill()}, ${this.border.apply()})`;
    }
}

// ব্যবহার
const redCircle = new Circle(5, createRed(), createSolidBorder());
console.log(redCircle.draw());

const blueRect = new RectangleShape(10, 5, createBlue(), createDashedBorder());
console.log(blueRect.draw());
```

### 🎭 Decorator Pattern — কম্পোজিশনের শক্তিশালী রূপ (PHP)

```php
<?php
// Decorator Pattern — কম্পোজিশনের মাধ্যমে ফিচার যোগ

interface Logger
{
    public function log(string $message): void;
}

class ConsoleLogger implements Logger
{
    public function log(string $message): void
    {
        echo "[Console] {$message}\n";
    }
}

// ডেকোরেটর — বেস ক্লাস
abstract class LoggerDecorator implements Logger
{
    public function __construct(
        protected Logger $wrappedLogger
    ) {}
}

// টাইমস্ট্যাম্প যোগ করার ডেকোরেটর
class TimestampLogger extends LoggerDecorator
{
    public function log(string $message): void
    {
        $timestamp = date('Y-m-d H:i:s');
        $this->wrappedLogger->log("[{$timestamp}] {$message}");
    }
}

// এনক্রিপশন ডেকোরেটর
class EncryptedLogger extends LoggerDecorator
{
    public function log(string $message): void
    {
        $encrypted = base64_encode($message);
        $this->wrappedLogger->log("🔐 {$encrypted}");
    }
}

// ব্যবহার — ডেকোরেটর স্ট্যাক করা যায়!
$logger = new ConsoleLogger();
$logger = new TimestampLogger($logger);
$logger = new EncryptedLogger($logger);
$logger->log('গুরুত্বপূর্ণ বার্তা');
// [Console] [2024-01-15 10:30:00] 🔐 4oif4oCm4oCC
```

### 🧬 PHP Traits — Mixin-এর মতো ব্যবহার

```php
<?php
// PHP Traits — কোড পুনঃব্যবহারের জন্য

trait HasTimestamps
{
    private ?DateTime $createdAt = null;
    private ?DateTime $updatedAt = null;

    public function setCreatedAt(): void
    {
        $this->createdAt = new DateTime();
    }

    public function setUpdatedAt(): void
    {
        $this->updatedAt = new DateTime();
    }

    public function getCreatedAt(): ?DateTime
    {
        return $this->createdAt;
    }
}

trait SoftDeletes
{
    private ?DateTime $deletedAt = null;

    public function softDelete(): void
    {
        $this->deletedAt = new DateTime();
    }

    public function isDeleted(): bool
    {
        return $this->deletedAt !== null;
    }

    public function restore(): void
    {
        $this->deletedAt = null;
    }
}

// ✅ Traits ব্যবহার — গভীর ইনহেরিটেন্সের প্রয়োজন নেই!
class User
{
    use HasTimestamps, SoftDeletes;

    public function __construct(
        private string $name,
        private string $email
    ) {}
}

class Post
{
    use HasTimestamps, SoftDeletes;

    public function __construct(
        private string $title,
        private string $content
    ) {}
}

$user = new User('রহিম', 'rahim@example.com');
$user->setCreatedAt();
$user->softDelete();
echo $user->isDeleted() ? 'ডিলিটেড' : 'অ্যাক্টিভ'; // ডিলিটেড
$user->restore();
echo $user->isDeleted() ? 'ডিলিটেড' : 'অ্যাক্টিভ'; // অ্যাক্টিভ
```

### 🧬 JavaScript Mixins

```javascript
// JavaScript Mixins — কম্পোজিশনের মাধ্যমে কোড শেয়ারিং

const HasTimestamps = (superclass) => class extends superclass {
    #createdAt = null;
    #updatedAt = null;

    setCreatedAt() {
        this.#createdAt = new Date();
        return this;
    }

    setUpdatedAt() {
        this.#updatedAt = new Date();
        return this;
    }

    getCreatedAt() {
        return this.#createdAt;
    }
};

const SoftDeletes = (superclass) => class extends superclass {
    #deletedAt = null;

    softDelete() {
        this.#deletedAt = new Date();
        return this;
    }

    isDeleted() {
        return this.#deletedAt !== null;
    }

    restore() {
        this.#deletedAt = null;
        return this;
    }
};

// Mixins প্রয়োগ
class BaseModel {
    constructor(data = {}) {
        Object.assign(this, data);
    }
}

class User extends SoftDeletes(HasTimestamps(BaseModel)) {
    constructor(name, email) {
        super({ name, email });
    }
}

// ব্যবহার
const user = new User('রহিম', 'rahim@example.com');
user.setCreatedAt().softDelete();
console.log(user.isDeleted()); // true
user.restore();
console.log(user.isDeleted()); // false
```

---

## 5. ⚖️ "Favor Composition over Inheritance"

### 📖 GoF উদ্ধৃতি ও ব্যাখ্যা

Gang of Four (GoF) তাদের বিখ্যাত "Design Patterns" (1994) বইতে বলেছেন:

> "Favor object composition over class inheritance."

এর মানে এই **নয়** যে ইনহেরিটেন্স কখনো ব্যবহার করবেন না।
এর মানে হলো: **প্রথমে কম্পোজিশনের কথা ভাবুন। ইনহেরিটেন্স তখনই ব্যবহার করুন যখন এটি সত্যিই প্রয়োজন।**

### 🌳 Decision Tree — কখন কোনটি ব্যবহার করবেন?

```
🤔 "ক্লাস A এবং ক্লাস B এর মধ্যে সম্পর্ক কী?"
│
├─ ✅ "B সত্যিই A-এর একটি ধরন" (is-a)?
│   │
│   ├─ ✅ LSP (Liskov Substitution) মানে?
│   │   │  (যেখানে A ব্যবহার হয়, সেখানে B ব্যবহার করা যায়?)
│   │   │
│   │   ├─ ✅ হ্যাঁ → ইনহেরিটেন্স ব্যবহার করুন ✨
│   │   │
│   │   └─ ❌ না → কম্পোজিশন ব্যবহার করুন 🧱
│   │
│   └─ ❌ না → কম্পোজিশন ব্যবহার করুন 🧱
│
├─ 🔧 "B, A-এর কিছু ফিচার ব্যবহার করতে চায়" (has-a / uses-a)?
│   │
│   └─ → কম্পোজিশন / DI ব্যবহার করুন 🧱
│
└─ 🔄 "রানটাইমে বিহেভিয়ার পরিবর্তন দরকার"?
    │
    └─ → কম্পোজিশন + Strategy Pattern 🧱🎯
```

### 📋 ইনহেরিটেন্স ব্যবহারের চেকলিস্ট

ইনহেরিটেন্স ব্যবহারের আগে এই ৫টি প্রশ্নের উত্তর "হ্যাঁ" হতে হবে:

| # | প্রশ্ন | ব্যাখ্যা |
|---|--------|----------|
| 1 | সত্যিকারের **is-a** সম্পর্ক আছে? | Dog is-a Animal ✅, Car is-a Engine ❌ |
| 2 | **LSP** মানে? | যেখানে Animal চলবে, সেখানে Dog চলবে? |
| 3 | হায়ারার্কি **৩ স্তরের** কম? | গভীর হায়ারার্কি = সমস্যা |
| 4 | **সাবক্লাস বিস্ফোরণ** হবে না? | কম্বিনেশন বাড়লে ক্লাস সংখ্যা? |
| 5 | বেস ক্লাস **স্থিতিশীল**? | ঘন ঘন পরিবর্তন হলে Fragile Base Class সমস্যা |

> ⚠️ যদি কোনো উত্তর "না" হয়, **কম্পোজিশন** ব্যবহার করুন!

---

## 6. 🔀 হাইব্রিড অ্যাপ্রোচ

### 📖 সংজ্ঞা

অনেক সময় শুধু ইনহেরিটেন্স বা শুধু কম্পোজিশন যথেষ্ট নয়। **হাইব্রিড অ্যাপ্রোচ** হলো দুটোর সংমিশ্রণ — ইনহেরিটেন্সের গঠন ও কম্পোজিশনের নমনীয়তা একসাথে।

### 🏗️ Template Method + Composition (PHP)

```php
<?php
// হাইব্রিড: Template Method (ইনহেরিটেন্স) + Strategy (কম্পোজিশন)

interface DataFormatter
{
    public function format(array $data): string;
}

class JsonFormatter implements DataFormatter
{
    public function format(array $data): string
    {
        return json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    }
}

class CsvFormatter implements DataFormatter
{
    public function format(array $data): string
    {
        $header = implode(',', array_keys($data[0] ?? []));
        $rows = array_map(fn($row) => implode(',', $row), $data);
        return $header . "\n" . implode("\n", $rows);
    }
}

// Template Method — মূল কাঠামো ইনহেরিটেন্সে
abstract class ReportGenerator
{
    public function __construct(
        private DataFormatter $formatter // 👈 কম্পোজিশন
    ) {}

    // Template Method — ধাপগুলো নির্ধারিত
    final public function generate(): string
    {
        $data = $this->fetchData();
        $processed = $this->processData($data);
        $formatted = $this->formatter->format($processed);
        return $this->addHeader() . "\n" . $formatted . "\n" . $this->addFooter();
    }

    // সাবক্লাস এগুলো ইমপ্লিমেন্ট করবে
    abstract protected function fetchData(): array;
    abstract protected function processData(array $data): array;

    protected function addHeader(): string
    {
        return '=== রিপোর্ট শুরু ===';
    }

    protected function addFooter(): string
    {
        return '=== রিপোর্ট শেষ ===';
    }
}

// সুনির্দিষ্ট রিপোর্ট — ইনহেরিটেন্স (is-a ReportGenerator)
class SalesReport extends ReportGenerator
{
    protected function fetchData(): array
    {
        return [
            ['পণ্য' => 'ল্যাপটপ', 'বিক্রি' => 50, 'আয়' => 2500000],
            ['পণ্য' => 'মোবাইল', 'বিক্রি' => 200, 'আয়' => 4000000],
        ];
    }

    protected function processData(array $data): array
    {
        // মোট হিসাব যোগ
        $totalSales = array_sum(array_column($data, 'বিক্রি'));
        $totalRevenue = array_sum(array_column($data, 'আয়'));
        $data[] = ['পণ্য' => '*** মোট ***', 'বিক্রি' => $totalSales, 'আয়' => $totalRevenue];
        return $data;
    }

    protected function addHeader(): string
    {
        return '📊 বিক্রয় রিপোর্ট — ' . date('Y-m-d');
    }
}

// ব্যবহার — ফরম্যাট রানটাইমে পরিবর্তনযোগ্য
$report = new SalesReport(new JsonFormatter());
echo $report->generate();

$report2 = new SalesReport(new CsvFormatter());
echo $report2->generate();
```

### 🏗️ Abstract Class + Interface + Composition (JavaScript)

```javascript
// হাইব্রিড অ্যাপ্রোচ — JavaScript

// কম্পোজিশন কম্পোনেন্ট
class JsonFormatter {
    format(data) {
        return JSON.stringify(data, null, 2);
    }
}

class CsvFormatter {
    format(data) {
        if (data.length === 0) return '';
        const header = Object.keys(data[0]).join(',');
        const rows = data.map(row => Object.values(row).join(','));
        return [header, ...rows].join('\n');
    }
}

// বেস ক্লাস — Template Method
class ReportGenerator {
    #formatter;

    constructor(formatter) {
        if (new.target === ReportGenerator) {
            throw new Error('সরাসরি ইনস্ট্যান্স করা যাবে না');
        }
        this.#formatter = formatter;
    }

    // Template Method
    generate() {
        const data = this.fetchData();
        const processed = this.processData(data);
        const formatted = this.#formatter.format(processed);
        return `${this.addHeader()}\n${formatted}\n${this.addFooter()}`;
    }

    fetchData() { throw new Error('ইমপ্লিমেন্ট করুন'); }
    processData(data) { throw new Error('ইমপ্লিমেন্ট করুন'); }
    addHeader() { return '=== রিপোর্ট শুরু ==='; }
    addFooter() { return '=== রিপোর্ট শেষ ==='; }
}

// সুনির্দিষ্ট রিপোর্ট
class SalesReport extends ReportGenerator {
    fetchData() {
        return [
            { পণ্য: 'ল্যাপটপ', বিক্রি: 50, আয়: 2500000 },
            { পণ্য: 'মোবাইল', বিক্রি: 200, আয়: 4000000 },
        ];
    }

    processData(data) {
        const totalSales = data.reduce((sum, row) => sum + row.বিক্রি, 0);
        const totalRevenue = data.reduce((sum, row) => sum + row.আয়, 0);
        return [...data, { পণ্য: '*** মোট ***', বিক্রি: totalSales, আয়: totalRevenue }];
    }

    addHeader() {
        return `📊 বিক্রয় রিপোর্ট — ${new Date().toISOString().split('T')[0]}`;
    }
}

// ব্যবহার
const report = new SalesReport(new JsonFormatter());
console.log(report.generate());

const report2 = new SalesReport(new CsvFormatter());
console.log(report2.generate());
```

---

## 7. 📬 নোটিফিকেশন সিস্টেম — রিয়েল-ওয়ার্ল্ড কেস স্টাডি

### ❌ ইনহেরিটেন্স দিয়ে ভুল পদ্ধতি (PHP)

```php
<?php
// ❌ ইনহেরিটেন্সে নোটিফিকেশন সিস্টেম — সমস্যা!

abstract class Notification
{
    abstract public function send(string $message, string $recipient): void;
}

class EmailNotification extends Notification
{
    public function send(string $message, string $recipient): void
    {
        echo "📧 ইমেইল → {$recipient}: {$message}\n";
    }
}

class SmsNotification extends Notification
{
    public function send(string $message, string $recipient): void
    {
        echo "📱 SMS → {$recipient}: {$message}\n";
    }
}

class PushNotification extends Notification
{
    public function send(string $message, string $recipient): void
    {
        echo "🔔 Push → {$recipient}: {$message}\n";
    }
}

// 😱 এখন "urgent" নোটিফিকেশন চাই?
class UrgentEmailNotification extends EmailNotification
{
    public function send(string $message, string $recipient): void
    {
        $urgentMsg = "🚨 জরুরি: {$message}";
        parent::send($urgentMsg, $recipient);
    }
}

class UrgentSmsNotification extends SmsNotification
{
    public function send(string $message, string $recipient): void
    {
        $urgentMsg = "🚨 জরুরি: {$message}";
        parent::send($urgentMsg, $recipient);
    }
}

// 😱😱 এখন "scheduled" নোটিফিকেশনও চাই?
// ScheduledEmailNotification, ScheduledSmsNotification, ScheduledPushNotification
// UrgentScheduledEmailNotification... 💀 সাবক্লাস বিস্ফোরণ!
```

### ✅ কম্পোজিশন দিয়ে সমাধান (PHP)

```php
<?php
// ✅ কম্পোজিশনে নোটিফিকেশন সিস্টেম — সুন্দর ও নমনীয়!

// চ্যানেল ইন্টারফেস
interface NotificationChannel
{
    public function send(string $message, string $recipient): void;
}

// ফিল্টার/মিডলওয়্যার ইন্টারফেস
interface NotificationMiddleware
{
    public function process(string $message): string;
}

// চ্যানেল ইমপ্লিমেন্টেশন
class EmailChannel implements NotificationChannel
{
    public function send(string $message, string $recipient): void
    {
        echo "📧 ইমেইল → {$recipient}: {$message}\n";
    }
}

class SmsChannel implements NotificationChannel
{
    public function send(string $message, string $recipient): void
    {
        echo "📱 SMS → {$recipient}: {$message}\n";
    }
}

class PushChannel implements NotificationChannel
{
    public function send(string $message, string $recipient): void
    {
        echo "🔔 Push → {$recipient}: {$message}\n";
    }
}

class SlackChannel implements NotificationChannel
{
    public function send(string $message, string $recipient): void
    {
        echo "💬 Slack → #{$recipient}: {$message}\n";
    }
}

// মিডলওয়্যার ইমপ্লিমেন্টেশন
class UrgentMiddleware implements NotificationMiddleware
{
    public function process(string $message): string
    {
        return "🚨 জরুরি: {$message}";
    }
}

class ScheduleMiddleware implements NotificationMiddleware
{
    public function __construct(
        private string $scheduleTime
    ) {}

    public function process(string $message): string
    {
        return "[⏰ নির্ধারিত: {$this->scheduleTime}] {$message}";
    }
}

class TranslateMiddleware implements NotificationMiddleware
{
    public function process(string $message): string
    {
        return "🌐 [অনুবাদিত] {$message}";
    }
}

// নোটিফিকেশন সার্ভিস — সব একসাথে কম্পোজ
class NotificationService
{
    /** @var NotificationChannel[] */
    private array $channels = [];

    /** @var NotificationMiddleware[] */
    private array $middlewares = [];

    public function addChannel(NotificationChannel $channel): self
    {
        $this->channels[] = $channel;
        return $this; // ফ্লুয়েন্ট ইন্টারফেস
    }

    public function addMiddleware(NotificationMiddleware $middleware): self
    {
        $this->middlewares[] = $middleware;
        return $this;
    }

    public function notify(string $message, string $recipient): void
    {
        // মিডলওয়্যার দিয়ে মেসেজ প্রসেস
        $processed = $message;
        foreach ($this->middlewares as $middleware) {
            $processed = $middleware->process($processed);
        }

        // সব চ্যানেলে পাঠানো
        foreach ($this->channels as $channel) {
            $channel->send($processed, $recipient);
        }
    }
}

// ✅ ব্যবহার — যেকোনো কম্বিনেশন!
$service = new NotificationService();

// জরুরি নোটিফিকেশন — ইমেইল + SMS + Push
$service
    ->addMiddleware(new UrgentMiddleware())
    ->addChannel(new EmailChannel())
    ->addChannel(new SmsChannel())
    ->addChannel(new PushChannel())
    ->notify('সার্ভার ডাউন!', 'admin@company.com');

echo "\n---\n\n";

// নির্ধারিত + অনুবাদিত নোটিফিকেশন — শুধু Slack
$scheduledService = new NotificationService();
$scheduledService
    ->addMiddleware(new ScheduleMiddleware('2024-01-15 09:00'))
    ->addMiddleware(new TranslateMiddleware())
    ->addChannel(new SlackChannel())
    ->notify('সাপ্তাহিক মিটিং', 'general');
```

### ✅ কম্পোজিশন দিয়ে সমাধান (JavaScript)

```javascript
// ✅ কম্পোজিশনে নোটিফিকেশন সিস্টেম — JavaScript

// চ্যানেলসমূহ
class EmailChannel {
    send(message, recipient) {
        console.log(`📧 ইমেইল → ${recipient}: ${message}`);
    }
}

class SmsChannel {
    send(message, recipient) {
        console.log(`📱 SMS → ${recipient}: ${message}`);
    }
}

class PushChannel {
    send(message, recipient) {
        console.log(`🔔 Push → ${recipient}: ${message}`);
    }
}

class SlackChannel {
    send(message, recipient) {
        console.log(`💬 Slack → #${recipient}: ${message}`);
    }
}

// মিডলওয়্যারসমূহ
class UrgentMiddleware {
    process(message) {
        return `🚨 জরুরি: ${message}`;
    }
}

class ScheduleMiddleware {
    constructor(scheduleTime) {
        this.scheduleTime = scheduleTime;
    }

    process(message) {
        return `[⏰ নির্ধারিত: ${this.scheduleTime}] ${message}`;
    }
}

class TranslateMiddleware {
    process(message) {
        return `🌐 [অনুবাদিত] ${message}`;
    }
}

// নোটিফিকেশন সার্ভিস
class NotificationService {
    #channels = [];
    #middlewares = [];

    addChannel(channel) {
        this.#channels.push(channel);
        return this;
    }

    addMiddleware(middleware) {
        this.#middlewares.push(middleware);
        return this;
    }

    notify(message, recipient) {
        // মিডলওয়্যার চেইন
        let processed = message;
        for (const mw of this.#middlewares) {
            processed = mw.process(processed);
        }

        // সব চ্যানেলে পাঠানো
        for (const channel of this.#channels) {
            channel.send(processed, recipient);
        }
    }
}

// ব্যবহার
const urgentService = new NotificationService();
urgentService
    .addMiddleware(new UrgentMiddleware())
    .addChannel(new EmailChannel())
    .addChannel(new SmsChannel())
    .addChannel(new PushChannel())
    .notify('সার্ভার ডাউন!', 'admin@company.com');

console.log('\n---\n');

const scheduledService = new NotificationService();
scheduledService
    .addMiddleware(new ScheduleMiddleware('2024-01-15 09:00'))
    .addMiddleware(new TranslateMiddleware())
    .addChannel(new SlackChannel())
    .notify('সাপ্তাহিক মিটিং', 'general');
```

---

## 8. 🎮 গেম ক্যারেক্টার সিস্টেম — রিয়েল-ওয়ার্ল্ড কেস স্টাডি

### ❌ ইনহেরিটেন্স হেল (PHP)

```php
<?php
// ❌ ইনহেরিটেন্স দিয়ে গেম ক্যারেক্টার — বিপর্যয়!

class Character
{
    public function move(): string { return '🚶 হাঁটছে'; }
}

class Warrior extends Character
{
    public function attack(): string { return '⚔️ তলোয়ার দিয়ে আক্রমণ'; }
}

class Mage extends Character
{
    public function castSpell(): string { return '🔮 জাদু করছে'; }
}

class Archer extends Character
{
    public function shoot(): string { return '🏹 তীর ছুঁড়ছে'; }
}

// 😱 একটি ক্যারেক্টার যে যোদ্ধা + জাদুকর?
// class BattleMage extends Warrior, Mage {} // ❌ Multiple Inheritance নেই!

// 😱 তাহলে?
class BattleMage extends Warrior
{
    // Mage-এর কোড কপি-পেস্ট করতে হবে! 😭
    public function castSpell(): string { return '🔮 জাদু করছে'; }
}

class MageArcher extends Mage
{
    // Archer-এর কোড কপি-পেস্ট! 😭
    public function shoot(): string { return '🏹 তীর ছুঁড়ছে'; }
}

// 🤯 Warrior + Mage + Archer = TripleClass? কপি-পেস্ট ×3!
// ৩টি ক্লাসের ৩-কম্বিনেশন = ৭টি ক্লাস (প্রতিটিতে কোড ডুপ্লিকেশন)
```

### ✅ কম্পোজিশন দিয়ে সমাধান (PHP)

```php
<?php
// ✅ কম্পোজিশন দিয়ে গেম ক্যারেক্টার — এলিগ্যান্ট!

// অ্যাবিলিটি ইন্টারফেস
interface Ability
{
    public function getName(): string;
    public function execute(): string;
}

// বিভিন্ন অ্যাবিলিটি
class SwordAttack implements Ability
{
    public function getName(): string { return 'তলোয়ার'; }
    public function execute(): string { return '⚔️ তলোয়ার দিয়ে আক্রমণ!'; }
}

class BowAttack implements Ability
{
    public function getName(): string { return 'ধনুক'; }
    public function execute(): string { return '🏹 তীর ছোঁড়া!'; }
}

class FireSpell implements Ability
{
    public function getName(): string { return 'আগুনের জাদু'; }
    public function execute(): string { return '🔥 ফায়ারবল নিক্ষেপ!'; }
}

class HealSpell implements Ability
{
    public function getName(): string { return 'নিরাময়'; }
    public function execute(): string { return '💚 HP পুনরুদ্ধার!'; }
}

class StealthAbility implements Ability
{
    public function getName(): string { return 'অদৃশ্য'; }
    public function execute(): string { return '👻 অদৃশ্য হয়ে গেলো!'; }
}

class ShieldBlock implements Ability
{
    public function getName(): string { return 'ঢাল'; }
    public function execute(): string { return '🛡️ ঢাল দিয়ে ব্লক!'; }
}

// মুভমেন্ট ইন্টারফেস
interface MovementStyle
{
    public function move(): string;
}

class WalkMovement implements MovementStyle
{
    public function move(): string { return '🚶 হেঁটে চলছে'; }
}

class FlyMovement implements MovementStyle
{
    public function move(): string { return '🦅 উড়ে চলছে'; }
}

class TeleportMovement implements MovementStyle
{
    public function move(): string { return '✨ টেলিপোর্ট!'; }
}

// ক্যারেক্টার — কম্পোজিশন দিয়ে তৈরি
class GameCharacter
{
    private string $name;
    private MovementStyle $movement;
    /** @var Ability[] */
    private array $abilities = [];
    private int $hp;
    private int $level;

    public function __construct(string $name, MovementStyle $movement, int $hp = 100)
    {
        $this->name = $name;
        $this->movement = $movement;
        $this->hp = $hp;
        $this->level = 1;
    }

    public function addAbility(Ability $ability): self
    {
        $this->abilities[$ability->getName()] = $ability;
        return $this;
    }

    public function removeAbility(string $abilityName): self
    {
        unset($this->abilities[$abilityName]);
        return $this;
    }

    public function useAbility(string $abilityName): string
    {
        if (!isset($this->abilities[$abilityName])) {
            return "❌ {$this->name}-এর '{$abilityName}' অ্যাবিলিটি নেই!";
        }
        return "🎮 {$this->name}: " . $this->abilities[$abilityName]->execute();
    }

    public function move(): string
    {
        return "🎮 {$this->name}: " . $this->movement->move();
    }

    public function changeMovement(MovementStyle $newMovement): void
    {
        $this->movement = $newMovement;
    }

    public function getStatus(): string
    {
        $abilityNames = implode(', ', array_keys($this->abilities));
        return "👤 {$this->name} | HP: {$this->hp} | Lv: {$this->level} | অ্যাবিলিটি: [{$abilityNames}]";
    }
}

// ✅ ব্যবহার — যেকোনো কম্বিনেশন!

// যোদ্ধা
$warrior = new GameCharacter('রণবীর', new WalkMovement(), 150);
$warrior->addAbility(new SwordAttack())
        ->addAbility(new ShieldBlock());

echo $warrior->getStatus();
// 👤 রণবীর | HP: 150 | Lv: 1 | অ্যাবিলিটি: [তলোয়ার, ঢাল]

echo $warrior->useAbility('তলোয়ার');
// 🎮 রণবীর: ⚔️ তলোয়ার দিয়ে আক্রমণ!

// ব্যাটল ম্যাজ — যোদ্ধা + জাদুকর (কোনো নতুন ক্লাস দরকার নেই!)
$battleMage = new GameCharacter('জাদুযোদ্ধা', new TeleportMovement(), 120);
$battleMage->addAbility(new SwordAttack())
           ->addAbility(new FireSpell())
           ->addAbility(new HealSpell());

echo $battleMage->getStatus();
// 👤 জাদুযোদ্ধা | HP: 120 | Lv: 1 | অ্যাবিলিটি: [তলোয়ার, আগুনের জাদু, নিরাময়]

// রানটাইমে অ্যাবিলিটি যোগ/বাদ দেওয়া সম্ভব!
$battleMage->addAbility(new StealthAbility());
$battleMage->removeAbility('তলোয়ার');
echo $battleMage->getStatus();
// 👤 জাদুযোদ্ধা | HP: 120 | Lv: 1 | অ্যাবিলিটি: [আগুনের জাদু, নিরাময়, অদৃশ্য]
```

### ✅ কম্পোজিশন দিয়ে সমাধান (JavaScript)

```javascript
// ✅ কম্পোজিশন দিয়ে গেম ক্যারেক্টার — JavaScript

// অ্যাবিলিটিসমূহ
const createSwordAttack = () => ({
    name: 'তলোয়ার',
    execute: () => '⚔️ তলোয়ার দিয়ে আক্রমণ!',
});

const createBowAttack = () => ({
    name: 'ধনুক',
    execute: () => '🏹 তীর ছোঁড়া!',
});

const createFireSpell = () => ({
    name: 'আগুনের জাদু',
    execute: () => '🔥 ফায়ারবল নিক্ষেপ!',
});

const createHealSpell = () => ({
    name: 'নিরাময়',
    execute: () => '💚 HP পুনরুদ্ধার!',
});

const createStealthAbility = () => ({
    name: 'অদৃশ্য',
    execute: () => '👻 অদৃশ্য হয়ে গেলো!',
});

const createShieldBlock = () => ({
    name: 'ঢাল',
    execute: () => '🛡️ ঢাল দিয়ে ব্লক!',
});

// মুভমেন্ট স্টাইল
const createWalkMovement = () => ({ move: () => '🚶 হেঁটে চলছে' });
const createFlyMovement = () => ({ move: () => '🦅 উড়ে চলছে' });
const createTeleportMovement = () => ({ move: () => '✨ টেলিপোর্ট!' });

// ক্যারেক্টার ক্লাস
class GameCharacter {
    #name;
    #movement;
    #abilities = new Map();
    #hp;
    #level;

    constructor(name, movement, hp = 100) {
        this.#name = name;
        this.#movement = movement;
        this.#hp = hp;
        this.#level = 1;
    }

    addAbility(ability) {
        this.#abilities.set(ability.name, ability);
        return this;
    }

    removeAbility(abilityName) {
        this.#abilities.delete(abilityName);
        return this;
    }

    useAbility(abilityName) {
        const ability = this.#abilities.get(abilityName);
        if (!ability) {
            return `❌ ${this.#name}-এর '${abilityName}' অ্যাবিলিটি নেই!`;
        }
        return `🎮 ${this.#name}: ${ability.execute()}`;
    }

    move() {
        return `🎮 ${this.#name}: ${this.#movement.move()}`;
    }

    changeMovement(newMovement) {
        this.#movement = newMovement;
    }

    getStatus() {
        const abilityNames = [...this.#abilities.keys()].join(', ');
        return `👤 ${this.#name} | HP: ${this.#hp} | Lv: ${this.#level} | অ্যাবিলিটি: [${abilityNames}]`;
    }
}

// ব্যবহার
const warrior = new GameCharacter('রণবীর', createWalkMovement(), 150);
warrior.addAbility(createSwordAttack()).addAbility(createShieldBlock());
console.log(warrior.getStatus());
console.log(warrior.useAbility('তলোয়ার'));

const battleMage = new GameCharacter('জাদুযোদ্ধা', createTeleportMovement(), 120);
battleMage
    .addAbility(createSwordAttack())
    .addAbility(createFireSpell())
    .addAbility(createHealSpell());

console.log(battleMage.getStatus());

// রানটাইমে পরিবর্তন
battleMage.addAbility(createStealthAbility());
battleMage.removeAbility('তলোয়ার');
console.log(battleMage.getStatus());
```

---

## 9. 📊 তুলনা সারণী

### মূল তুলনা

| বৈশিষ্ট্য | 🔗 ইনহেরিটেন্স | 🧱 কম্পোজিশন |
|-----------|----------------|--------------|
| **সম্পর্কের ধরন** | is-a (একটি ধরন) | has-a (ধারণ করে) |
| **কাপলিং** | টাইট (শক্ত বন্ধন) | লুজ (আলগা বন্ধন) |
| **রানটাইম পরিবর্তন** | ❌ সম্ভব নয় | ✅ সম্ভব |
| **কোড রিইউজ** | সুপারক্লাস থেকে | ডেলিগেশনের মাধ্যমে |
| **নমনীয়তা** | কম | বেশি |
| **বোঝার সহজতা** | সহজ (ছোট হায়ারার্কিতে) | কিছুটা জটিল |
| **টেস্টিং** | কঠিন (মক করা কঠিন) | সহজ (DI দিয়ে মক) |
| **এনক্যাপসুলেশন** | ভাঙে (protected মেম্বার) | রক্ষা করে |
| **Diamond Problem** | সম্ভব | নেই |
| **Fragile Base Class** | সম্ভব | নেই |
| **SOLID নীতি** | LSP লঙ্ঘনের ঝুঁকি | DIP, OCP, SRP মানে |

### কখন কোনটি ব্যবহার করবেন

| পরিস্থিতি | সুপারিশ | কারণ |
|-----------|---------|------|
| সত্যিকারের is-a সম্পর্ক, স্থিতিশীল হায়ারার্কি | ✅ ইনহেরিটেন্স | Circle is-a Shape |
| কোড রিইউজ (শুধু কিছু ফিচার দরকার) | ✅ কম্পোজিশন | গরিলা-কলা এড়াতে |
| রানটাইমে বিহেভিয়ার পরিবর্তন | ✅ কম্পোজিশন | Strategy Pattern |
| একাধিক ফিচার মিক্স করা | ✅ কম্পোজিশন | Traits/Mixins |
| ফ্রেমওয়ার্ক এক্সটেনশন পয়েন্ট | ✅ ইনহেরিটেন্স | Template Method |
| ক্রস-কাটিং কনসার্ন (লগিং, ক্যাশিং) | ✅ কম্পোজিশন | Decorator Pattern |
| GUI ফ্রেমওয়ার্ক উইজেট | ✅ ইনহেরিটেন্স | Button is-a Widget |
| ডোমেইন মডেল ভ্যারিয়েশন | ✅ কম্পোজিশন | পণ্যের বৈশিষ্ট্য |

---

## 10. 🗺️ সিদ্ধান্ত ফ্লোচার্ট

```
┌─────────────────────────────────────────────────────┐
│      🤔 ক্লাস সম্পর্ক নির্ধারণ করুন                 │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │  "B is a A" সত্য?           │
        │  (B সত্যিই A-এর একটি ধরন?)  │
        └──────┬──────────────┬───────┘
               │              │
           হ্যাঁ ▼              ▼ না
     ┌──────────────┐   ┌──────────────────┐
     │  LSP মানে?    │   │  কম্পোজিশন      │
     │  (সুপারটাইপের │   │  ব্যবহার করুন   │
     │  জায়গায়      │   │  🧱              │
     │  সাবটাইপ     │   └──────────────────┘
     │  চলবে?)      │
     └──┬───────┬───┘
        │       │
    হ্যাঁ ▼       ▼ না
  ┌─────────┐  ┌──────────────────┐
  │ হায়ারার্কি│  │  কম্পোজিশন      │
  │ ≤ ৩     │  │  ব্যবহার করুন   │
  │ স্তর?   │  │  🧱              │
  └──┬──┬───┘  └──────────────────┘
     │  │
 হ্যাঁ ▼  ▼ না
┌────────┐ ┌──────────────────┐
│ সাবক্লাস│ │  কম্পোজিশন      │
│ বিস্ফোরণ│ │  ব্যবহার করুন   │
│ হবে না? │ │  🧱              │
└──┬──┬──┘ └──────────────────┘
   │  │
হ্যাঁ▼  ▼ না
┌────────────┐ ┌──────────────────┐
│ রানটাইমে   │ │  কম্পোজিশন      │
│ পরিবর্তন   │ │  ব্যবহার করুন   │
│ লাগবে না?  │ │  🧱              │
└──┬─────┬──┘ └──────────────────┘
   │     │
হ্যাঁ▼     ▼ না
┌──────┐  ┌──────────────────┐
│ ✅    │  │  কম্পোজিশন      │
│ইনহেরি-│  │  ব্যবহার করুন   │
│টেন্স  │  │  🧱              │
│ঠিক আছে│  └──────────────────┘
└──────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📌 সংক্ষেপে: বেশিরভাগ ক্ষেত্রেই আপনি কম্পোজিশনে 🧱 পৌঁছাবেন!
   এটাই GoF-এর "Favor Composition over Inheritance" নীতির সারমর্ম।
```

### 📐 SOLID নীতির সাথে সম্পর্ক

```
┌──────────────────────────────────────────────────────────────┐
│                    SOLID ও কম্পোজিশন                         │
├──────────┬───────────────────────────────────────────────────┤
│ S (SRP)  │ কম্পোজিশন: প্রতিটি ক্লাসের একটি দায়িত্ব          │
│          │ ইনহেরিটেন্স: সুপারক্লাসের সব দায়িত্ব আসে         │
├──────────┼───────────────────────────────────────────────────┤
│ O (OCP)  │ কম্পোজিশন: নতুন ক্লাস যোগ করে এক্সটেন্ড          │
│          │ ইনহেরিটেন্স: বেস ক্লাস পরিবর্তনের প্রলোভন        │
├──────────┼───────────────────────────────────────────────────┤
│ L (LSP)  │ কম্পোজিশন: প্রযোজ্য নয় (is-a সম্পর্ক নেই)      │
│          │ ইনহেরিটেন্স: সবসময় মানতে হবে (ভুল করলে বাগ)      │
├──────────┼───────────────────────────────────────────────────┤
│ I (ISP)  │ কম্পোজিশন: ছোট ইন্টারফেস কম্পোজ করা সহজ         │
│          │ ইনহেরিটেন্স: বড় ইন্টারফেস ভাঙা কঠিন              │
├──────────┼───────────────────────────────────────────────────┤
│ D (DIP)  │ কম্পোজিশন: ইন্টারফেসের উপর নির্ভরশীল (✅)        │
│          │ ইনহেরিটেন্স: কংক্রিট ক্লাসের উপর নির্ভরশীল (❌)   │
└──────────┴───────────────────────────────────────────────────┘
```

---

## 11. 🔗 সম্পর্কিত টপিক ও রিসোর্স

### 📚 সম্পর্কিত ডিজাইন প্যাটার্ন

| প্যাটার্ন | ধরন | কম্পোজিশন/ইনহেরিটেন্স | বিবরণ |
|-----------|------|----------------------|--------|
| **Strategy** | Behavioral | কম্পোজিশন | আলগোরিদম ইন্টারচেঞ্জ |
| **Decorator** | Structural | কম্পোজিশন | ফিচার যোগ করা |
| **Bridge** | Structural | কম্পোজিশন | অ্যাবস্ট্রাকশন ও ইমপ্লিমেন্টেশন আলাদা |
| **Template Method** | Behavioral | ইনহেরিটেন্স | আলগোরিদমের কাঠামো নির্ধারণ |
| **Observer** | Behavioral | কম্পোজিশন | ইভেন্ট হ্যান্ডলিং |
| **State** | Behavioral | কম্পোজিশন | স্টেট-ভিত্তিক বিহেভিয়ার |
| **Adapter** | Structural | কম্পোজিশন | ইন্টারফেস রূপান্তর |

### 📖 পড়ার জন্য বই ও রিসোর্স

1. **"Design Patterns" — GoF (1994)** — কম্পোজিশন ওভার ইনহেরিটেন্স নীতির উৎস
2. **"Effective Java" — Joshua Bloch** — Item 18: "Favor composition over inheritance"
3. **"Clean Code" — Robert C. Martin** — SOLID নীতি ও ক্লিন আর্কিটেকচার
4. **"Head First Design Patterns"** — সহজ ভাষায় প্যাটার্ন শেখা
5. **"Refactoring" — Martin Fowler** — "Replace Inheritance with Delegation" রিফ্যাক্টরিং

### 🔗 সম্পর্কিত ক্লিন কোড টপিক

- **SOLID Principles** — বিশেষত LSP ও DIP
- **Dependency Injection (DI)** — কম্পোজিশনের মূল হাতিয়ার
- **Interface Segregation** — ছোট, ফোকাসড ইন্টারফেস
- **Coupling & Cohesion** — লুজ কাপলিং ও হাই কোহেশন
- **Design Patterns** — Strategy, Decorator, Bridge, Observer
- **Refactoring** — ইনহেরিটেন্স থেকে কম্পোজিশনে মাইগ্রেশন

---

## 🎯 সারসংক্ষেপ

```
╔══════════════════════════════════════════════════════════════╗
║                  মূল শিক্ষা (Key Takeaways)                  ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  1️⃣  ইনহেরিটেন্স "is-a" সম্পর্কের জন্য                      ║
║     কম্পোজিশন "has-a" সম্পর্কের জন্য                        ║
║                                                              ║
║  2️⃣  প্রথমে কম্পোজিশন ভাবুন, তারপর ইনহেরিটেন্স              ║
║     (GoF: "Favor Composition over Inheritance")              ║
║                                                              ║
║  3️⃣  ইনহেরিটেন্স হায়ারার্কি ৩ স্তরের বেশি হলে               ║
║     কম্পোজিশনে রিফ্যাক্টর করুন                               ║
║                                                              ║
║  4️⃣  রানটাইম নমনীয়তা দরকার হলে সবসময়                       ║
║     কম্পোজিশন ব্যবহার করুন                                   ║
║                                                              ║
║  5️⃣  দুটোই হাতিয়ার — সঠিক পরিস্থিতিতে সঠিকটি              ║
║     ব্যবহার করাই আসল দক্ষতা                                  ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

> 💡 **মনে রাখবেন**: কম্পোজিশন ও ইনহেরিটেন্স কোনোটাই "ভালো" বা "খারাপ" নয়।
> এগুলো হলো **হাতিয়ার** — সঠিক সমস্যায় সঠিক হাতিয়ার ব্যবহার করাই প্রকৃত সফটওয়্যার ইঞ্জিনিয়ারিং। 🛠️

---

*📝 এই ডকুমেন্ট "Clean Code" সিরিজের অংশ।*
*🔄 সর্বশেষ আপডেট: ২০২৫*
