# 🏭 Abstract Factory প্যাটার্ন

## 📌 সংজ্ঞা ও পরিচিতি

> **GoF Definition:** "Provide an interface for creating families of related or dependent objects without specifying their concrete classes."
> — *Design Patterns: Elements of Reusable Object-Oriented Software (1994)*

Abstract Factory হলো একটি **Creational Design Pattern** যেটি সম্পর্কিত বা পরস্পর-নির্ভরশীল অবজেক্টগুলোর **পুরো পরিবার (family)** তৈরি করার জন্য একটি ইন্টারফেস প্রদান করে — কিন্তু কোন concrete class ব্যবহার হচ্ছে সেটা ক্লায়েন্ট কোডকে জানতে হয় না।

### মূল ধারণা

ধরুন আপনি একটি **bKash-এর মতো পেমেন্ট সিস্টেম** তৈরি করছেন। bKash ecosystem-এ Payment, Refund, আর Transfer — এই তিনটি অপারেশন একসাথে কাজ করে। আবার Nagad ecosystem-এও একই তিনটি অপারেশন আছে, কিন্তু ভিন্ন implementation-এ। Abstract Factory বলে: "আমি তোমাকে একটা factory দেব, সেই factory থেকে তুমি পুরো ecosystem-এর সব অবজেক্ট পাবে — কোনটা bKash, কোনটা Nagad সেটা factory ঠিক করবে।"

### কেন দরকার?

সাধারণ Factory Method শুধু **একটি** প্রোডাক্ট তৈরি করে। কিন্তু বাস্তবে অনেক সময় **একগুচ্ছ সম্পর্কিত প্রোডাক্ট** একসাথে তৈরি করতে হয় যেগুলো পরস্পর সামঞ্জস্যপূর্ণ (compatible) হতে হবে। উদাহরণস্বরূপ:

- **UI Theme:** Dark theme-এর Button + Input + Modal একসাথে কাজ করবে, Light theme-এর গুলো একসাথে
- **Database Driver:** MySQL-এর Connection + QueryBuilder + Migration একসাথে, PostgreSQL-এরগুলো একসাথে
- **Payment Gateway:** bKash-এর Payment + Refund + Transfer একসাথে, Nagad-এরগুলো একসাথে

আপনি যদি ভুল করে Dark theme-এর Button-এর সাথে Light theme-এর Modal মিশিয়ে ফেলেন — সেটা হবে **inconsistency**। Abstract Factory এই inconsistency রোধ করে।

---

## 🏠 বাস্তব উদাহরণ: Furniture Factory

বাংলাদেশের **Hatil** বা **Otobi** ফার্নিচার কোম্পানির কথা ভাবুন। তাদের দুইটা স্টাইল আছে:

| | Modern Style | Victorian Style |
|---|---|---|
| **Chair** | Chrome legs, minimal cushion | Carved wood, velvet cushion |
| **Table** | Glass top, steel frame | Mahogany top, ornate legs |
| **Sofa** | L-shaped, fabric | Chesterfield, leather |

এখানে মূল বিষয় হলো — আপনি যদি **Modern Chair** নেন, তাহলে **Modern Table** আর **Modern Sofa**-ও নিতে হবে। Victorian Chair-এর সাথে Modern Table মানানসই হবে না।

**Abstract Factory** ঠিক এই কাজটাই করে — একটা `ModernFurnitureFactory` দিলে পুরো Modern set পাবেন, `VictorianFurnitureFactory` দিলে পুরো Victorian set। ক্লায়েন্ট কোড শুধু জানে "আমি একটা Chair চাই" — সেটা Modern না Victorian, সেটা factory decide করে।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT CODE                                  │
│  (শুধু AbstractFactory ও Abstract Product interfaces জানে)          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ uses
                           ▼
            ┌──────────────────────────────┐
            │    <<interface>>             │
            │    AbstractFactory           │
            │──────────────────────────────│
            │ + createProductA(): ProductA │
            │ + createProductB(): ProductB │
            │ + createProductC(): ProductC │
            └──────────┬───────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
┌───────────────────┐    ┌───────────────────┐
│ ConcreteFactory1  │    │ ConcreteFactory2  │
│───────────────────│    │───────────────────│
│+createProductA()  │    │+createProductA()  │
│+createProductB()  │    │+createProductB()  │
│+createProductC()  │    │+createProductC()  │
└───────────────────┘    └───────────────────┘
    │ creates                 │ creates
    ▼                         ▼
┌─────────────┐        ┌─────────────┐
│ ProductA1   │        │ ProductA2   │
│ ProductB1   │        │ ProductB2   │
│ ProductC1   │        │ ProductC2   │
└─────────────┘        └─────────────┘

          ┌──────────────────────┐
          │  <<interface>>       │
          │  AbstractProductA    │
          │──────────────────────│
          │ + operationA()       │
          └──────┬───────────────┘
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
┌──────────────┐  ┌──────────────┐
│ ProductA1    │  │ ProductA2    │
│ (Family 1)   │  │ (Family 2)   │
└──────────────┘  └──────────────┘
```

**মূল সম্পর্কগুলো:**
- `Client` → শুধুমাত্র abstract interfaces ব্যবহার করে
- `ConcreteFactory` → নির্দিষ্ট family-র সব product তৈরি করে
- প্রতিটি family-র products পরস্পর compatible

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Abstract Factory — Furniture Example

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ===================== Abstract Products =====================

interface Chair
{
    public function sitOn(): string;
    public function getStyle(): string;
    public function getPrice(): float;
}

interface Table
{
    public function placeItems(): string;
    public function getStyle(): string;
    public function getPrice(): float;
}

interface Sofa
{
    public function lieOn(): string;
    public function getStyle(): string;
    public function getPrice(): float;
}

// ===================== Modern Family =====================

final readonly class ModernChair implements Chair
{
    public function sitOn(): string
    {
        return 'আপনি একটি মিনিমালিস্ট Modern Chair-এ বসলেন — chrome legs, ergonomic design';
    }

    public function getStyle(): string { return 'Modern'; }
    public function getPrice(): float { return 15_000.00; }
}

final readonly class ModernTable implements Table
{
    public function placeItems(): string
    {
        return 'Glass-top Modern Table-এ আইটেম রাখলেন — steel frame, sleek finish';
    }

    public function getStyle(): string { return 'Modern'; }
    public function getPrice(): float { return 25_000.00; }
}

final readonly class ModernSofa implements Sofa
{
    public function lieOn(): string
    {
        return 'L-shaped Modern Sofa-তে শুয়ে পড়লেন — fabric upholstery, foam cushion';
    }

    public function getStyle(): string { return 'Modern'; }
    public function getPrice(): float { return 45_000.00; }
}

// ===================== Victorian Family =====================

final readonly class VictorianChair implements Chair
{
    public function sitOn(): string
    {
        return 'একটি ক্লাসিক Victorian Chair-এ বসলেন — carved wood, velvet cushion';
    }

    public function getStyle(): string { return 'Victorian'; }
    public function getPrice(): float { return 35_000.00; }
}

final readonly class VictorianTable implements Table
{
    public function placeItems(): string
    {
        return 'Mahogany Victorian Table-এ আইটেম রাখলেন — ornate legs, polished top';
    }

    public function getStyle(): string { return 'Victorian'; }
    public function getPrice(): float { return 55_000.00; }
}

final readonly class VictorianSofa implements Sofa
{
    public function lieOn(): string
    {
        return 'Chesterfield Victorian Sofa-তে শুয়ে পড়লেন — genuine leather, tufted back';
    }

    public function getStyle(): string { return 'Victorian'; }
    public function getPrice(): float { return 95_000.00; }
}

// ===================== Abstract Factory =====================

interface FurnitureFactory
{
    public function createChair(): Chair;
    public function createTable(): Table;
    public function createSofa(): Sofa;
}

// ===================== Concrete Factories =====================

final readonly class ModernFurnitureFactory implements FurnitureFactory
{
    public function createChair(): Chair { return new ModernChair(); }
    public function createTable(): Table { return new ModernTable(); }
    public function createSofa(): Sofa { return new ModernSofa(); }
}

final readonly class VictorianFurnitureFactory implements FurnitureFactory
{
    public function createChair(): Chair { return new VictorianChair(); }
    public function createTable(): Table { return new VictorianTable(); }
    public function createSofa(): Sofa { return new VictorianSofa(); }
}

// ===================== Client Code =====================

function furnishRoom(FurnitureFactory $factory): void
{
    $chair = $factory->createChair();
    $table = $factory->createTable();
    $sofa = $factory->createSofa();

    echo "--- {$chair->getStyle()} রুম সাজানো হচ্ছে ---\n";
    echo $chair->sitOn() . "\n";
    echo $table->placeItems() . "\n";
    echo $sofa->lieOn() . "\n";

    $total = $chair->getPrice() + $table->getPrice() + $sofa->getPrice();
    echo "মোট খরচ: ৳" . number_format($total, 2) . "\n\n";
}

// Runtime-এ factory নির্বাচন
$style = 'modern'; // এটা config/env থেকে আসতে পারে

$factory = match ($style) {
    'modern' => new ModernFurnitureFactory(),
    'victorian' => new VictorianFurnitureFactory(),
    default => throw new InvalidArgumentException("Unknown style: {$style}"),
};

furnishRoom($factory);
```

#### JavaScript (ES2022+)

```javascript
// ===================== Abstract Products (via class + error throwing) =====================

class Chair {
    sitOn()    { throw new Error('Abstract method: sitOn() must be implemented'); }
    getStyle() { throw new Error('Abstract method: getStyle() must be implemented'); }
    getPrice() { throw new Error('Abstract method: getPrice() must be implemented'); }
}

class Table {
    placeItems() { throw new Error('Abstract method: placeItems() must be implemented'); }
    getStyle()   { throw new Error('Abstract method: getStyle() must be implemented'); }
    getPrice()   { throw new Error('Abstract method: getPrice() must be implemented'); }
}

class Sofa {
    lieOn()    { throw new Error('Abstract method: lieOn() must be implemented'); }
    getStyle() { throw new Error('Abstract method: getStyle() must be implemented'); }
    getPrice() { throw new Error('Abstract method: getPrice() must be implemented'); }
}

// ===================== Modern Family =====================

class ModernChair extends Chair {
    sitOn()    { return 'আপনি একটি মিনিমালিস্ট Modern Chair-এ বসলেন — chrome legs, ergonomic design'; }
    getStyle() { return 'Modern'; }
    getPrice() { return 15_000; }
}

class ModernTable extends Table {
    placeItems() { return 'Glass-top Modern Table-এ আইটেম রাখলেন — steel frame, sleek finish'; }
    getStyle()   { return 'Modern'; }
    getPrice()   { return 25_000; }
}

class ModernSofa extends Sofa {
    lieOn()    { return 'L-shaped Modern Sofa-তে শুয়ে পড়লেন — fabric upholstery, foam cushion'; }
    getStyle() { return 'Modern'; }
    getPrice() { return 45_000; }
}

// ===================== Victorian Family =====================

class VictorianChair extends Chair {
    sitOn()    { return 'একটি ক্লাসিক Victorian Chair-এ বসলেন — carved wood, velvet cushion'; }
    getStyle() { return 'Victorian'; }
    getPrice() { return 35_000; }
}

class VictorianTable extends Table {
    placeItems() { return 'Mahogany Victorian Table-এ আইটেম রাখলেন — ornate legs, polished top'; }
    getStyle()   { return 'Victorian'; }
    getPrice()   { return 55_000; }
}

class VictorianSofa extends Sofa {
    lieOn()    { return 'Chesterfield Victorian Sofa-তে শুয়ে পড়লেন — genuine leather, tufted back'; }
    getStyle() { return 'Victorian'; }
    getPrice() { return 95_000; }
}

// ===================== Abstract Factory =====================

class FurnitureFactory {
    createChair() { throw new Error('Abstract: createChair()'); }
    createTable() { throw new Error('Abstract: createTable()'); }
    createSofa()  { throw new Error('Abstract: createSofa()'); }
}

// ===================== Concrete Factories =====================

class ModernFurnitureFactory extends FurnitureFactory {
    createChair() { return new ModernChair(); }
    createTable() { return new ModernTable(); }
    createSofa()  { return new ModernSofa(); }
}

class VictorianFurnitureFactory extends FurnitureFactory {
    createChair() { return new VictorianChair(); }
    createTable() { return new VictorianTable(); }
    createSofa()  { return new VictorianSofa(); }
}

// ===================== Client Code =====================

function furnishRoom(factory) {
    const chair = factory.createChair();
    const table = factory.createTable();
    const sofa  = factory.createSofa();

    console.log(`--- ${chair.getStyle()} রুম সাজানো হচ্ছে ---`);
    console.log(chair.sitOn());
    console.log(table.placeItems());
    console.log(sofa.lieOn());

    const total = chair.getPrice() + table.getPrice() + sofa.getPrice();
    console.log(`মোট খরচ: ৳${total.toLocaleString('bn-BD')}\n`);
}

// Runtime factory selection
const style = 'modern';

const factories = {
    modern:    ModernFurnitureFactory,
    victorian: VictorianFurnitureFactory,
};

const FactoryClass = factories[style];
if (!FactoryClass) throw new Error(`Unknown style: ${style}`);

furnishRoom(new FactoryClass());
```

---

### 2️⃣ UI Component Factory — Light Theme vs Dark Theme

এটি একটি অত্যন্ত বাস্তবসম্মত উদাহরণ। বাংলাদেশের যেকোনো বড় অ্যাপ (যেমন Pathao, Shohoz, bKash) — সবগুলোতেই dark/light theme toggle থাকে। এখানে **Button, Input, Modal** — এই তিনটি component একটি theme family তৈরি করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ===================== Abstract Products =====================

interface Button
{
    public function render(): string;
    public function onClick(string $action): string;
}

interface Input
{
    public function render(): string;
    public function validate(string $value): bool;
}

interface Modal
{
    public function render(): string;
    public function open(string $title, string $content): string;
    public function close(): string;
}

// ===================== Light Theme Family =====================

final readonly class LightButton implements Button
{
    public function __construct(
        private string $label = 'Submit',
    ) {}

    public function render(): string
    {
        return "<button class=\"bg-white text-gray-900 border border-gray-300 "
             . "px-4 py-2 rounded hover:bg-gray-100\">{$this->label}</button>";
    }

    public function onClick(string $action): string
    {
        return "Light Button '{$this->label}' clicked → executing: {$action}";
    }
}

final readonly class LightInput implements Input
{
    public function render(): string
    {
        return '<input class="bg-white text-gray-900 border border-gray-300 '
             . 'px-3 py-2 rounded focus:ring-blue-500" />';
    }

    public function validate(string $value): bool
    {
        return trim($value) !== '';
    }
}

final readonly class LightModal implements Modal
{
    public function render(): string
    {
        return '<div class="bg-white text-gray-900 shadow-xl rounded-lg p-6 border">';
    }

    public function open(string $title, string $content): string
    {
        return "🔲 Light Modal opened: [{$title}] — {$content}";
    }

    public function close(): string
    {
        return '🔲 Light Modal closed — background: white overlay fade-out';
    }
}

// ===================== Dark Theme Family =====================

final readonly class DarkButton implements Button
{
    public function __construct(
        private string $label = 'Submit',
    ) {}

    public function render(): string
    {
        return "<button class=\"bg-gray-800 text-white border border-gray-600 "
             . "px-4 py-2 rounded hover:bg-gray-700\">{$this->label}</button>";
    }

    public function onClick(string $action): string
    {
        return "Dark Button '{$this->label}' clicked → executing: {$action}";
    }
}

final readonly class DarkInput implements Input
{
    public function render(): string
    {
        return '<input class="bg-gray-800 text-white border border-gray-600 '
             . 'px-3 py-2 rounded focus:ring-purple-500" />';
    }

    public function validate(string $value): bool
    {
        return trim($value) !== '';
    }
}

final readonly class DarkModal implements Modal
{
    public function render(): string
    {
        return '<div class="bg-gray-900 text-white shadow-2xl rounded-lg p-6 border border-gray-700">';
    }

    public function open(string $title, string $content): string
    {
        return "🌙 Dark Modal opened: [{$title}] — {$content}";
    }

    public function close(): string
    {
        return '🌙 Dark Modal closed — background: dark overlay fade-out';
    }
}

// ===================== Abstract Factory =====================

interface UIComponentFactory
{
    public function createButton(string $label): Button;
    public function createInput(): Input;
    public function createModal(): Modal;
}

// ===================== Concrete Factories =====================

final readonly class LightThemeFactory implements UIComponentFactory
{
    public function createButton(string $label = 'Submit'): Button
    {
        return new LightButton($label);
    }

    public function createInput(): Input { return new LightInput(); }
    public function createModal(): Modal { return new LightModal(); }
}

final readonly class DarkThemeFactory implements UIComponentFactory
{
    public function createButton(string $label = 'Submit'): Button
    {
        return new DarkButton($label);
    }

    public function createInput(): Input { return new DarkInput(); }
    public function createModal(): Modal { return new DarkModal(); }
}

// ===================== Client: Form Builder =====================

final readonly class FormBuilder
{
    public function __construct(
        private UIComponentFactory $factory,
    ) {}

    public function buildLoginForm(): string
    {
        $emailInput    = $this->factory->createInput();
        $passwordInput = $this->factory->createInput();
        $submitButton  = $this->factory->createButton('লগইন করুন');
        $errorModal    = $this->factory->createModal();

        $output  = "=== লগইন ফর্ম ===\n";
        $output .= "Email: " . $emailInput->render() . "\n";
        $output .= "Password: " . $passwordInput->render() . "\n";
        $output .= "Button: " . $submitButton->render() . "\n";
        $output .= $submitButton->onClick('auth.login') . "\n";

        if (!$emailInput->validate('')) {
            $output .= $errorModal->open('ত্রুটি', 'ইমেইল ফাঁকা রাখা যাবে না!') . "\n";
        }

        return $output;
    }
}

// ব্যবহার — theme config থেকে নেওয়া
$theme = getenv('APP_THEME') ?: 'dark';

$factory = match ($theme) {
    'light' => new LightThemeFactory(),
    'dark'  => new DarkThemeFactory(),
    default => throw new InvalidArgumentException("Unknown theme: {$theme}"),
};

$formBuilder = new FormBuilder($factory);
echo $formBuilder->buildLoginForm();
```

#### JavaScript (ES2022+)

```javascript
// ===================== Abstract Factory — UI Theme =====================

// Light Theme products
class LightButton {
    #label;
    constructor(label = 'Submit') { this.#label = label; }
    render()  { return `<button class="bg-white text-gray-900 border">${this.#label}</button>`; }
    onClick(action) { return `Light Button '${this.#label}' → ${action}`; }
}

class LightInput {
    render()   { return '<input class="bg-white text-gray-900 border rounded" />'; }
    validate(v) { return v.trim() !== ''; }
}

class LightModal {
    render() { return '<div class="bg-white shadow-xl rounded-lg p-6">'; }
    open(title, content) { return `🔲 Light Modal: [${title}] — ${content}`; }
    close() { return '🔲 Light Modal closed'; }
}

// Dark Theme products
class DarkButton {
    #label;
    constructor(label = 'Submit') { this.#label = label; }
    render()  { return `<button class="bg-gray-800 text-white border">${this.#label}</button>`; }
    onClick(action) { return `Dark Button '${this.#label}' → ${action}`; }
}

class DarkInput {
    render()   { return '<input class="bg-gray-800 text-white border rounded" />'; }
    validate(v) { return v.trim() !== ''; }
}

class DarkModal {
    render() { return '<div class="bg-gray-900 text-white shadow-2xl p-6">'; }
    open(title, content) { return `🌙 Dark Modal: [${title}] — ${content}`; }
    close() { return '🌙 Dark Modal closed'; }
}

// Factories
class LightThemeFactory {
    createButton(label) { return new LightButton(label); }
    createInput()       { return new LightInput(); }
    createModal()       { return new LightModal(); }
}

class DarkThemeFactory {
    createButton(label) { return new DarkButton(label); }
    createInput()       { return new DarkInput(); }
    createModal()       { return new DarkModal(); }
}

// Client code — FormBuilder কোন theme ব্যবহার হচ্ছে জানে না
class FormBuilder {
    #factory;
    constructor(factory) { this.#factory = factory; }

    buildLoginForm() {
        const emailInput = this.#factory.createInput();
        const passInput  = this.#factory.createInput();
        const submitBtn  = this.#factory.createButton('লগইন করুন');
        const modal      = this.#factory.createModal();

        const lines = [
            '=== লগইন ফর্ম ===',
            `Email: ${emailInput.render()}`,
            `Password: ${passInput.render()}`,
            `Button: ${submitBtn.render()}`,
            submitBtn.onClick('auth.login'),
        ];

        if (!emailInput.validate('')) {
            lines.push(modal.open('ত্রুটি', 'ইমেইল ফাঁকা রাখা যাবে না!'));
        }

        return lines.join('\n');
    }
}

// ব্যবহার
const theme = process.env.APP_THEME ?? 'dark';
const factoryMap = { light: LightThemeFactory, dark: DarkThemeFactory };
const Factory = factoryMap[theme] ?? (() => { throw new Error(`Unknown: ${theme}`); })();
console.log(new FormBuilder(new Factory()).buildLoginForm());
```

---

### 3️⃣ Database Factory — MySQL vs PostgreSQL vs MongoDB

এটি **Laravel-এর DatabaseManager** যেভাবে কাজ করে তার simplified version। বাংলাদেশের অনেক কোম্পানি (যেমন SSL Wireless, Brain Station 23) — multi-database support দিতে এই প্যাটার্ন ব্যবহার করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ===================== Abstract Products =====================

interface Connection
{
    public function connect(): string;
    public function disconnect(): string;
    public function isConnected(): bool;
    public function getDriverName(): string;
}

interface QueryBuilder
{
    public function select(string $table, array $columns = ['*']): string;
    public function insert(string $table, array $data): string;
    public function where(string $column, string $operator, mixed $value): static;
    public function toSql(): string;
}

interface Migration
{
    public function createTable(string $name, array $columns): string;
    public function dropTable(string $name): string;
    public function addColumn(string $table, string $column, string $type): string;
}

// ===================== MySQL Family =====================

final class MySQLConnection implements Connection
{
    private bool $connected = false;

    public function connect(): string
    {
        $this->connected = true;
        return 'MySQL: Connected via PDO mysql driver (port 3306)';
    }

    public function disconnect(): string
    {
        $this->connected = false;
        return 'MySQL: Connection closed';
    }

    public function isConnected(): bool { return $this->connected; }
    public function getDriverName(): string { return 'mysql'; }
}

final class MySQLQueryBuilder implements QueryBuilder
{
    private array $wheres = [];

    public function select(string $table, array $columns = ['*']): string
    {
        $cols = implode(', ', array_map(fn($c) => "`{$c}`", $columns));
        return "SELECT {$cols} FROM `{$table}`" . $this->buildWheres();
    }

    public function insert(string $table, array $data): string
    {
        $cols = implode(', ', array_map(fn($k) => "`{$k}`", array_keys($data)));
        $vals = implode(', ', array_map(fn($v) => "'{$v}'", array_values($data)));
        return "INSERT INTO `{$table}` ({$cols}) VALUES ({$vals})";
    }

    public function where(string $column, string $operator, mixed $value): static
    {
        $this->wheres[] = "`{$column}` {$operator} '{$value}'";
        return $this;
    }

    public function toSql(): string { return implode(' AND ', $this->wheres); }

    private function buildWheres(): string
    {
        return $this->wheres ? ' WHERE ' . $this->toSql() : '';
    }
}

final readonly class MySQLMigration implements Migration
{
    public function createTable(string $name, array $columns): string
    {
        $cols = implode(', ', $columns);
        return "CREATE TABLE `{$name}` ({$cols}) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci";
    }

    public function dropTable(string $name): string
    {
        return "DROP TABLE IF EXISTS `{$name}`";
    }

    public function addColumn(string $table, string $column, string $type): string
    {
        return "ALTER TABLE `{$table}` ADD COLUMN `{$column}` {$type}";
    }
}

// ===================== PostgreSQL Family =====================

final class PostgreSQLConnection implements Connection
{
    private bool $connected = false;

    public function connect(): string
    {
        $this->connected = true;
        return 'PostgreSQL: Connected via PDO pgsql driver (port 5432)';
    }

    public function disconnect(): string
    {
        $this->connected = false;
        return 'PostgreSQL: Connection closed';
    }

    public function isConnected(): bool { return $this->connected; }
    public function getDriverName(): string { return 'pgsql'; }
}

final class PostgreSQLQueryBuilder implements QueryBuilder
{
    private array $wheres = [];

    public function select(string $table, array $columns = ['*']): string
    {
        $cols = implode(', ', array_map(fn($c) => "\"{$c}\"", $columns));
        return "SELECT {$cols} FROM \"{$table}\"" . $this->buildWheres();
    }

    public function insert(string $table, array $data): string
    {
        $cols = implode(', ', array_map(fn($k) => "\"{$k}\"", array_keys($data)));
        $vals = implode(', ', array_map(fn($v) => "'{$v}'", array_values($data)));
        return "INSERT INTO \"{$table}\" ({$cols}) VALUES ({$vals}) RETURNING id";
    }

    public function where(string $column, string $operator, mixed $value): static
    {
        $this->wheres[] = "\"{$column}\" {$operator} '{$value}'";
        return $this;
    }

    public function toSql(): string { return implode(' AND ', $this->wheres); }

    private function buildWheres(): string
    {
        return $this->wheres ? ' WHERE ' . $this->toSql() : '';
    }
}

final readonly class PostgreSQLMigration implements Migration
{
    public function createTable(string $name, array $columns): string
    {
        $cols = implode(', ', $columns);
        return "CREATE TABLE \"{$name}\" ({$cols})";
    }

    public function dropTable(string $name): string
    {
        return "DROP TABLE IF EXISTS \"{$name}\" CASCADE";
    }

    public function addColumn(string $table, string $column, string $type): string
    {
        return "ALTER TABLE \"{$table}\" ADD COLUMN \"{$column}\" {$type}";
    }
}

// ===================== MongoDB Family =====================

final class MongoDBConnection implements Connection
{
    private bool $connected = false;

    public function connect(): string
    {
        $this->connected = true;
        return 'MongoDB: Connected via MongoDB PHP Driver (port 27017)';
    }

    public function disconnect(): string
    {
        $this->connected = false;
        return 'MongoDB: Connection closed';
    }

    public function isConnected(): bool { return $this->connected; }
    public function getDriverName(): string { return 'mongodb'; }
}

final class MongoDBQueryBuilder implements QueryBuilder
{
    private array $filters = [];

    public function select(string $table, array $columns = ['*']): string
    {
        $projection = $columns === ['*'] ? '{}' : json_encode(
            array_fill_keys($columns, 1),
            JSON_THROW_ON_ERROR
        );
        $filter = $this->filters ? json_encode($this->filters, JSON_THROW_ON_ERROR) : '{}';
        return "db.{$table}.find({$filter}, {$projection})";
    }

    public function insert(string $table, array $data): string
    {
        $doc = json_encode($data, JSON_THROW_ON_ERROR);
        return "db.{$table}.insertOne({$doc})";
    }

    public function where(string $column, string $operator, mixed $value): static
    {
        $mongoOps = ['=' => '$eq', '>' => '$gt', '<' => '$lt', '>=' => '$gte', '<=' => '$lte'];
        $op = $mongoOps[$operator] ?? '$eq';
        $this->filters[$column] = [$op => $value];
        return $this;
    }

    public function toSql(): string
    {
        return json_encode($this->filters, JSON_THROW_ON_ERROR);
    }
}

final readonly class MongoDBMigration implements Migration
{
    public function createTable(string $name, array $columns): string
    {
        $validator = json_encode(['$jsonSchema' => ['required' => $columns]], JSON_THROW_ON_ERROR);
        return "db.createCollection(\"{$name}\", { validator: {$validator} })";
    }

    public function dropTable(string $name): string
    {
        return "db.{$name}.drop()";
    }

    public function addColumn(string $table, string $column, string $type): string
    {
        return "db.{$table}.updateMany({}, { \$set: { \"{$column}\": null } })";
    }
}

// ===================== Abstract Factory =====================

interface DatabaseFactory
{
    public function createConnection(): Connection;
    public function createQueryBuilder(): QueryBuilder;
    public function createMigration(): Migration;
}

final readonly class MySQLFactory implements DatabaseFactory
{
    public function createConnection(): Connection { return new MySQLConnection(); }
    public function createQueryBuilder(): QueryBuilder { return new MySQLQueryBuilder(); }
    public function createMigration(): Migration { return new MySQLMigration(); }
}

final readonly class PostgreSQLFactory implements DatabaseFactory
{
    public function createConnection(): Connection { return new PostgreSQLConnection(); }
    public function createQueryBuilder(): QueryBuilder { return new PostgreSQLQueryBuilder(); }
    public function createMigration(): Migration { return new PostgreSQLMigration(); }
}

final readonly class MongoDBFactory implements DatabaseFactory
{
    public function createConnection(): Connection { return new MongoDBConnection(); }
    public function createQueryBuilder(): QueryBuilder { return new MongoDBQueryBuilder(); }
    public function createMigration(): Migration { return new MongoDBMigration(); }
}

// ===================== Client: Database Manager =====================

final class DatabaseManager
{
    private Connection $connection;
    private QueryBuilder $queryBuilder;
    private Migration $migration;

    public function __construct(
        private readonly DatabaseFactory $factory,
    ) {
        $this->connection   = $this->factory->createConnection();
        $this->queryBuilder = $this->factory->createQueryBuilder();
        $this->migration    = $this->factory->createMigration();
    }

    public function demo(): void
    {
        echo $this->connection->connect() . "\n";
        echo "Driver: {$this->connection->getDriverName()}\n\n";

        echo "--- Migration ---\n";
        echo $this->migration->createTable('users', [
            'id INT PRIMARY KEY AUTO_INCREMENT',
            'name VARCHAR(255) NOT NULL',
            'email VARCHAR(255) UNIQUE',
        ]) . "\n\n";

        echo "--- Query ---\n";
        echo $this->queryBuilder->insert('users', [
            'name'  => 'রফিকুল ইসলাম',
            'email' => 'rafiq@example.com',
        ]) . "\n";

        $this->queryBuilder->where('name', '=', 'রফিকুল ইসলাম');
        echo $this->queryBuilder->select('users', ['name', 'email']) . "\n\n";

        echo $this->connection->disconnect() . "\n";
    }
}

// Runtime selection — config থেকে driver নাম নিয়ে factory তৈরি
$driver = getenv('DB_DRIVER') ?: 'mysql';

$factoryMap = [
    'mysql'   => new MySQLFactory(),
    'pgsql'   => new PostgreSQLFactory(),
    'mongodb' => new MongoDBFactory(),
];

$dbFactory = $factoryMap[$driver]
    ?? throw new InvalidArgumentException("Unsupported driver: {$driver}");

$manager = new DatabaseManager($dbFactory);
$manager->demo();
```

#### JavaScript (ES2022+)

```javascript
// ===================== Database Abstract Factory — JS =====================

// MySQL Family
class MySQLConnection {
    #connected = false;
    connect()    { this.#connected = true; return 'MySQL: Connected (port 3306)'; }
    disconnect() { this.#connected = false; return 'MySQL: Disconnected'; }
    get isConnected() { return this.#connected; }
    get driverName()  { return 'mysql'; }
}

class MySQLQueryBuilder {
    #wheres = [];
    select(table, cols = ['*']) {
        const c = cols.map(c => `\`${c}\``).join(', ');
        const w = this.#wheres.length ? ` WHERE ${this.#wheres.join(' AND ')}` : '';
        return `SELECT ${c} FROM \`${table}\`${w}`;
    }
    insert(table, data) {
        const keys = Object.keys(data).map(k => `\`${k}\``).join(', ');
        const vals = Object.values(data).map(v => `'${v}'`).join(', ');
        return `INSERT INTO \`${table}\` (${keys}) VALUES (${vals})`;
    }
    where(col, op, val) { this.#wheres.push(`\`${col}\` ${op} '${val}'`); return this; }
}

class MySQLMigration {
    createTable(name, cols) { return `CREATE TABLE \`${name}\` (${cols.join(', ')}) ENGINE=InnoDB`; }
    dropTable(name)         { return `DROP TABLE IF EXISTS \`${name}\``; }
    addColumn(table, col, type) { return `ALTER TABLE \`${table}\` ADD \`${col}\` ${type}`; }
}

// PostgreSQL Family
class PgConnection {
    #connected = false;
    connect()    { this.#connected = true; return 'PostgreSQL: Connected (port 5432)'; }
    disconnect() { this.#connected = false; return 'PostgreSQL: Disconnected'; }
    get isConnected() { return this.#connected; }
    get driverName()  { return 'pgsql'; }
}

class PgQueryBuilder {
    #wheres = [];
    select(table, cols = ['*']) {
        const c = cols.map(c => `"${c}"`).join(', ');
        const w = this.#wheres.length ? ` WHERE ${this.#wheres.join(' AND ')}` : '';
        return `SELECT ${c} FROM "${table}"${w}`;
    }
    insert(table, data) {
        const keys = Object.keys(data).map(k => `"${k}"`).join(', ');
        const vals = Object.values(data).map(v => `'${v}'`).join(', ');
        return `INSERT INTO "${table}" (${keys}) VALUES (${vals}) RETURNING id`;
    }
    where(col, op, val) { this.#wheres.push(`"${col}" ${op} '${val}'`); return this; }
}

class PgMigration {
    createTable(name, cols) { return `CREATE TABLE "${name}" (${cols.join(', ')})`; }
    dropTable(name)         { return `DROP TABLE IF EXISTS "${name}" CASCADE`; }
    addColumn(table, col, type) { return `ALTER TABLE "${table}" ADD "${col}" ${type}`; }
}

// Factories
class MySQLFactory {
    createConnection()   { return new MySQLConnection(); }
    createQueryBuilder() { return new MySQLQueryBuilder(); }
    createMigration()    { return new MySQLMigration(); }
}

class PostgreSQLFactory {
    createConnection()   { return new PgConnection(); }
    createQueryBuilder() { return new PgQueryBuilder(); }
    createMigration()    { return new PgMigration(); }
}

// Client — DatabaseManager
class DatabaseManager {
    #conn; #qb; #migration;

    constructor(factory) {
        this.#conn      = factory.createConnection();
        this.#qb        = factory.createQueryBuilder();
        this.#migration = factory.createMigration();
    }

    demo() {
        console.log(this.#conn.connect());
        console.log(`Driver: ${this.#conn.driverName}\n`);
        console.log(this.#migration.createTable('users', ['id SERIAL PRIMARY KEY', 'name TEXT']));
        console.log(this.#qb.insert('users', { name: 'রফিকুল ইসলাম', email: 'rafiq@example.com' }));
        this.#qb.where('name', '=', 'রফিকুল ইসলাম');
        console.log(this.#qb.select('users', ['name', 'email']));
        console.log(this.#conn.disconnect());
    }
}

const driver = process.env.DB_DRIVER ?? 'mysql';
const map = { mysql: MySQLFactory, pgsql: PostgreSQLFactory };
new DatabaseManager(new (map[driver] ?? map.mysql)()).demo();
```

---

## 🌍 Real-World Applicable Areas

### 1. Cross-Platform UI Components
```
WebUIFactory    → HTMLButton, HTMLInput, HTMLModal
MobileUIFactory → NativeButton, NativeInput, NativeSheet
DesktopUIFactory → WinButton, WinInput, WinDialog
```
বাংলাদেশের **Pathao** অ্যাপের কথা ভাবুন — Web, Android, iOS তিন প্ল্যাটফর্মে একই feature কিন্তু ভিন্ন UI component। Abstract Factory দিয়ে platform-specific component family তৈরি করা যায়।

### 2. Multi-Database Support (Laravel অভ্যন্তরীণভাবে এটাই করে)
Laravel-এর `Illuminate\Database\DatabaseManager` ক্লাস `config/database.php`-এ দেওয়া driver অনুযায়ী `MySqlConnector`, `PostgresConnector`, `SQLiteConnector` — এগুলো তৈরি করে। প্রতিটি connector-এর সাথে তার নিজস্ব `Grammar`, `Processor`, `SchemaBuilder` আসে — এটাই Abstract Factory pattern।

### 3. Payment System Families (বাংলাদেশ প্রসঙ্গ)
```
bKashFactory → bKashPayment + bKashRefund + bKashTransfer + bKashStatement
NagadFactory → NagadPayment + NagadRefund + NagadTransfer + NagadStatement
RocketFactory → RocketPayment + RocketRefund + RocketTransfer + RocketStatement
```
প্রতিটি gateway-র নিজস্ব API format, authentication mechanism, আর response structure আছে। Abstract Factory এগুলোকে একটি unified interface-এর পেছনে লুকিয়ে রাখে।

### 4. Multi-Tenant Theming
```
TenantA_ThemeFactory → TenantA_Header, TenantA_Footer, TenantA_Sidebar
TenantB_ThemeFactory → TenantB_Header, TenantB_Footer, TenantB_Sidebar
```
বাংলাদেশের **SaaS** প্ল্যাটফর্ম যেমন একটি HR Management System — প্রতিটি কোম্পানি (tenant) তাদের নিজস্ব branding চায়। Abstract Factory tenant অনুযায়ী পুরো UI component set তৈরি করে।

### 5. Report Generation
```
PDFReportFactory  → PDFChart + PDFTable + PDFHeader + PDFExporter
ExcelReportFactory → ExcelChart + ExcelTable + ExcelHeader + ExcelExporter
HTMLReportFactory  → HTMLChart + HTMLTable + HTMLHeader + HTMLExporter
```

### 6. Payment Gateway Complete Example (PHP)

```php
<?php

declare(strict_types=1);

// Abstract Products
interface Payment
{
    public function charge(float $amount, string $account): string;
}

interface Refund
{
    public function process(string $transactionId, float $amount): string;
}

interface Transfer
{
    public function send(string $from, string $to, float $amount): string;
}

// bKash Family
final readonly class BKashPayment implements Payment
{
    public function charge(float $amount, string $account): string
    {
        return "bKash: ৳{$amount} চার্জ করা হয়েছে অ্যাকাউন্ট {$account} থেকে (API: /checkout/payment/create)";
    }
}

final readonly class BKashRefund implements Refund
{
    public function process(string $transactionId, float $amount): string
    {
        return "bKash: ৳{$amount} রিফান্ড — TxnID: {$transactionId} (API: /checkout/payment/refund)";
    }
}

final readonly class BKashTransfer implements Transfer
{
    public function send(string $from, string $to, float $amount): string
    {
        return "bKash: ৳{$amount} ট্রান্সফার {$from} → {$to} (API: /checkout/payment/b2cPayment)";
    }
}

// Nagad Family
final readonly class NagadPayment implements Payment
{
    public function charge(float $amount, string $account): string
    {
        return "Nagad: ৳{$amount} চার্জ — অ্যাকাউন্ট {$account} (API: /remote-payment/initialize)";
    }
}

final readonly class NagadRefund implements Refund
{
    public function process(string $transactionId, float $amount): string
    {
        return "Nagad: ৳{$amount} রিফান্ড — OrderID: {$transactionId}";
    }
}

final readonly class NagadTransfer implements Transfer
{
    public function send(string $from, string $to, float $amount): string
    {
        return "Nagad: ৳{$amount} ট্রান্সফার {$from} → {$to}";
    }
}

// Abstract Factory
interface PaymentGatewayFactory
{
    public function createPayment(): Payment;
    public function createRefund(): Refund;
    public function createTransfer(): Transfer;
}

final readonly class BKashFactory implements PaymentGatewayFactory
{
    public function createPayment(): Payment   { return new BKashPayment(); }
    public function createRefund(): Refund     { return new BKashRefund(); }
    public function createTransfer(): Transfer { return new BKashTransfer(); }
}

final readonly class NagadFactory implements PaymentGatewayFactory
{
    public function createPayment(): Payment   { return new NagadPayment(); }
    public function createRefund(): Refund     { return new NagadRefund(); }
    public function createTransfer(): Transfer { return new NagadTransfer(); }
}

// Client: PaymentService — gateway জানে না, শুধু factory ব্যবহার করে
final readonly class PaymentService
{
    public function __construct(
        private PaymentGatewayFactory $factory,
    ) {}

    public function processOrder(string $account, float $amount): void
    {
        $payment = $this->factory->createPayment();
        echo $payment->charge($amount, $account) . "\n";
    }

    public function refundOrder(string $txnId, float $amount): void
    {
        $refund = $this->factory->createRefund();
        echo $refund->process($txnId, $amount) . "\n";
    }
}

// ব্যবহার
$gateway = getenv('PAYMENT_GATEWAY') ?: 'bkash';
$factory = match ($gateway) {
    'bkash' => new BKashFactory(),
    'nagad' => new NagadFactory(),
    default => throw new InvalidArgumentException("Unknown gateway: {$gateway}"),
};

$service = new PaymentService($factory);
$service->processOrder('01712345678', 500.00);
$service->refundOrder('TXN_ABC123', 200.00);
```

---

## 🔥 Advanced Deep Dive

### Abstract Factory vs Factory Method — বিস্তারিত তুলনা

| বৈশিষ্ট্য | Factory Method | Abstract Factory |
|---|---|---|
| **উদ্দেশ্য** | একটি product তৈরি করা | সম্পর্কিত product-দের পুরো family তৈরি করা |
| **কতগুলো create method** | একটি | একাধিক (`createA()`, `createB()`, `createC()`) |
| **Inheritance vs Composition** | Inheritance-based (subclass override করে) | Composition-based (factory inject করা হয়) |
| **কখন extend করা কঠিন** | নতুন product type যোগ করা সহজ | নতুন product type যোগ করলে সব factory-তে method যোগ করতে হয় |
| **কখন extend করা সহজ** | নতুন variant কঠিন | নতুন family (variant set) যোগ করা সহজ |

```php
<?php

// Factory Method — শুধু একটি product
abstract class NotificationFactory
{
    abstract public function createNotification(): Notification;

    public function send(string $message): void
    {
        $notification = $this->createNotification();
        $notification->deliver($message);
    }
}

class SmsNotificationFactory extends NotificationFactory
{
    public function createNotification(): Notification
    {
        return new SmsNotification();
    }
}

// Abstract Factory — পুরো notification family
interface NotificationSuiteFactory
{
    public function createNotification(): Notification;
    public function createTemplate(): Template;
    public function createLogger(): NotificationLogger;
}

class SmsNotificationSuiteFactory implements NotificationSuiteFactory
{
    public function createNotification(): Notification { return new SmsNotification(); }
    public function createTemplate(): Template         { return new SmsTemplate(); }
    public function createLogger(): NotificationLogger { return new SmsLogger(); }
}
```

### Abstract Factory with Prototype Pattern

কখনো কখনো concrete product তৈরি করা expensive হতে পারে। সেক্ষেত্রে Prototype pattern-এর সাথে Abstract Factory combine করা যায় — factory prototype clone করে নতুন object দেয়:

```php
<?php

declare(strict_types=1);

interface Cloneable
{
    public function clone(): static;
}

final class ExpensiveReport implements Cloneable
{
    public function __construct(
        private string $title,
        private array $data = [],
        private string $format = 'pdf',
    ) {
        // ধরুন এখানে heavy computation হয় — external API call, data aggregation ইত্যাদি
        sleep(0); // simulate
    }

    public function clone(): static
    {
        return clone $this; // shallow clone — deep clone দরকার হলে __clone() override করতে হবে
    }

    public function setTitle(string $title): void { $this->title = $title; }
    public function getTitle(): string { return $this->title; }
}

final class PrototypeReportFactory
{
    private array $prototypes = [];

    public function registerPrototype(string $key, Cloneable $prototype): void
    {
        $this->prototypes[$key] = $prototype;
    }

    public function create(string $key): Cloneable
    {
        if (!isset($this->prototypes[$key])) {
            throw new InvalidArgumentException("No prototype registered for: {$key}");
        }
        return $this->prototypes[$key]->clone();
    }
}

// ব্যবহার
$factory = new PrototypeReportFactory();
$factory->registerPrototype('monthly', new ExpensiveReport('Monthly Report', [], 'pdf'));
$factory->registerPrototype('yearly', new ExpensiveReport('Yearly Report', [], 'excel'));

// Clone করে নতুন report তৈরি — constructor-এর expensive computation বারবার হবে না
$report1 = $factory->create('monthly');
$report2 = $factory->create('monthly'); // আরেকটি clone
```

### Abstract Factory in Laravel

Laravel-এর ভেতরে Abstract Factory pattern ব্যাপকভাবে ব্যবহৃত:

```php
<?php

// Laravel-এর DatabaseManager (simplified)
// vendor/laravel/framework/src/Illuminate/Database/DatabaseManager.php

// Laravel যেভাবে কাজ করে — simplified illustration
namespace App\Illustrations;

class DatabaseManager
{
    protected array $connections = [];

    public function connection(?string $name = null): ConnectionInterface
    {
        $name ??= $this->getDefaultConnection();

        // Factory pattern: driver name অনুযায়ী connection family তৈরি
        if (!isset($this->connections[$name])) {
            $this->connections[$name] = $this->makeConnection($name);
        }

        return $this->connections[$name];
    }

    protected function makeConnection(string $name): ConnectionInterface
    {
        $config = config("database.connections.{$name}");

        // Abstract Factory decision point — driver অনুযায়ী পুরো family তৈরি
        return match ($config['driver']) {
            'mysql'  => new MySqlConnection(
                connector: new MySqlConnector(),
                grammar:   new MySqlGrammar(),     // Query grammar
                processor: new MySqlProcessor(),   // Result processor
                schema:    new MySqlSchemaBuilder() // Schema/migration builder
            ),
            'pgsql'  => new PostgresConnection(
                connector: new PostgresConnector(),
                grammar:   new PostgresGrammar(),
                processor: new PostgresProcessor(),
                schema:    new PostgresSchemaBuilder()
            ),
            'sqlite' => new SQLiteConnection(
                connector: new SQLiteConnector(),
                grammar:   new SQLiteGrammar(),
                processor: new SQLiteProcessor(),
                schema:    new SQLiteSchemaBuilder()
            ),
        };
    }

    protected function getDefaultConnection(): string
    {
        return config('database.default'); // .env-এর DB_CONNECTION
    }
}

// config/database.php-এ driver পরিবর্তন করলেই পুরো connection family বদলে যায়
// ক্লায়েন্ট কোডে কোনো পরিবর্তন লাগে না:
// DB::table('users')->where('name', 'রফিক')->get();
// এই কোড MySQL, PostgreSQL, SQLite — সবখানেই কাজ করে
```

### Dynamic Factory Selection at Runtime

```php
<?php

declare(strict_types=1);

// Registry pattern দিয়ে runtime-এ factory register ও resolve করা
final class FactoryRegistry
{
    /** @var array<string, class-string<DatabaseFactory>> */
    private static array $factories = [];

    public static function register(string $driver, string $factoryClass): void
    {
        if (!is_subclass_of($factoryClass, DatabaseFactory::class)) {
            throw new InvalidArgumentException("{$factoryClass} must implement DatabaseFactory");
        }
        self::$factories[$driver] = $factoryClass;
    }

    public static function resolve(string $driver): DatabaseFactory
    {
        $class = self::$factories[$driver]
            ?? throw new InvalidArgumentException("No factory for driver: {$driver}");

        return new $class();
    }

    public static function available(): array
    {
        return array_keys(self::$factories);
    }
}

// Boot time-এ register
FactoryRegistry::register('mysql', MySQLFactory::class);
FactoryRegistry::register('pgsql', PostgreSQLFactory::class);
FactoryRegistry::register('mongodb', MongoDBFactory::class);

// Runtime-এ resolve — user preference, config, বা environment অনুযায়ী
$factory = FactoryRegistry::resolve(
    getenv('DB_DRIVER') ?: 'mysql'
);
```

### Abstract Factory + Dependency Injection (Laravel Service Container)

```php
<?php

declare(strict_types=1);

// Laravel Service Provider-এ Abstract Factory bind করা
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Abstract Factory bind — config অনুযায়ী concrete factory resolve হবে
        $this->app->bind(PaymentGatewayFactory::class, function ($app) {
            $gateway = config('services.payment.default'); // 'bkash' বা 'nagad'

            return match ($gateway) {
                'bkash' => new BKashFactory(),
                'nagad' => new NagadFactory(),
                default => throw new \InvalidArgumentException(
                    "Unsupported payment gateway: {$gateway}"
                ),
            };
        });

        // PaymentService auto-resolve হবে — constructor injection
        $this->app->bind(PaymentService::class, function ($app) {
            return new PaymentService(
                $app->make(PaymentGatewayFactory::class)
            );
        });
    }
}

// Controller-এ ব্যবহার — কোন gateway ব্যবহার হচ্ছে Controller জানে না
class OrderController
{
    public function __construct(
        private readonly PaymentService $paymentService,
    ) {}

    public function checkout(Request $request): JsonResponse
    {
        $this->paymentService->processOrder(
            $request->input('account'),
            $request->float('amount'),
        );

        return response()->json(['status' => 'success']);
    }
}
```

```javascript
// JavaScript — DI Container সহ Abstract Factory

class DIContainer {
    #bindings = new Map();
    #singletons = new Map();

    bind(key, resolver) { this.#bindings.set(key, { resolver, singleton: false }); }

    singleton(key, resolver) { this.#bindings.set(key, { resolver, singleton: true }); }

    resolve(key) {
        const binding = this.#bindings.get(key);
        if (!binding) throw new Error(`No binding for: ${key}`);

        if (binding.singleton) {
            if (!this.#singletons.has(key)) {
                this.#singletons.set(key, binding.resolver(this));
            }
            return this.#singletons.get(key);
        }

        return binding.resolver(this);
    }
}

// Setup
const container = new DIContainer();

container.singleton('config', () => ({
    paymentGateway: process.env.PAYMENT_GATEWAY ?? 'bkash',
    dbDriver: process.env.DB_DRIVER ?? 'mysql',
}));

container.bind('PaymentGatewayFactory', (c) => {
    const gateway = c.resolve('config').paymentGateway;
    const factories = { bkash: BKashFactory, nagad: NagadFactory };
    const Factory = factories[gateway];
    if (!Factory) throw new Error(`Unknown gateway: ${gateway}`);
    return new Factory();
});

container.bind('PaymentService', (c) => {
    return new PaymentService(c.resolve('PaymentGatewayFactory'));
});

// ব্যবহার — কোন gateway আসছে সেটা container ঠিক করবে
const service = container.resolve('PaymentService');
```

---

## ✅ Pros ও ❌ Cons

| | বিবরণ |
|---|---|
| ✅ **Product Compatibility** | একই family-র products সবসময় compatible থাকে — কোনো mismatch হয় না |
| ✅ **Open/Closed Principle** | নতুন product family যোগ করতে existing code পরিবর্তন লাগে না |
| ✅ **Single Responsibility** | Product তৈরির কোড এক জায়গায় centralized |
| ✅ **Loose Coupling** | Client code concrete classes থেকে decoupled — শুধু interfaces জানে |
| ✅ **Easy Swapping** | Runtime-এ পুরো product family বদলানো যায় (factory swap করলেই হলো) |
| ✅ **Testability** | Mock factory inject করে সহজেই unit test করা যায় |
| ❌ **Complexity** | অনেক interface আর class তৈরি হয় — ছোট প্রজেক্টে overkill |
| ❌ **New Product Type কঠিন** | নতুন product type (যেমন `createFooter()`) যোগ করলে সব factory-তে implement করতে হয় |
| ❌ **Code Volume** | Interface + Abstract Factory + N concrete factories + N×M concrete products = অনেক কোড |
| ❌ **Learning Curve** | Junior developers-এর জন্য বুঝতে সময় লাগে |

---

## ⚠️ Common Mistakes

### ❌ ভুল ১: Factory-তে Conditional Logic রাখা

```php
// ❌ ভুল — Factory-র মধ্যে if/else দিয়ে product মেশানো
class BadFactory implements UIComponentFactory
{
    public function createButton(string $label): Button
    {
        if ($this->theme === 'dark') {
            return new DarkButton($label);
        }
        return new LightButton($label);  // 😱 এটা Factory Method, Abstract Factory না!
    }

    public function createInput(): Input
    {
        // সমস্যা: কেউ ভুল করে এখানে ভিন্ন theme-এর Input return করতে পারে
        return new DarkInput(); // ❌ inconsistency!
    }
}
```

```php
// ✅ সঠিক — প্রতিটি theme-র জন্য আলাদা factory
class DarkThemeFactory implements UIComponentFactory
{
    public function createButton(string $label): Button { return new DarkButton($label); }
    public function createInput(): Input { return new DarkInput(); }
    public function createModal(): Modal { return new DarkModal(); }
}
```

### ❌ ভুল ২: Client Code-এ Concrete Class ব্যবহার

```php
// ❌ ভুল — client সরাসরি concrete class ব্যবহার করছে
function buildUI(): void
{
    $button = new DarkButton('Click'); // 😱 concrete class-এ coupled
    $input  = new LightInput();        // 😱 theme mismatch!
}

// ✅ সঠিক — factory দিয়ে তৈরি করুন
function buildUI(UIComponentFactory $factory): void
{
    $button = $factory->createButton('Click');
    $input  = $factory->createInput();
}
```

### ❌ ভুল ৩: God Factory — সব কিছু এক factory-তে

```php
// ❌ ভুল — এক factory-তে সম্পর্কহীন products
interface EverythingFactory
{
    public function createButton(): Button;
    public function createDatabase(): Connection;
    public function createPayment(): Payment;
    public function createLogger(): Logger;
    // 😱 এগুলো একই "family" না!
}

// ✅ সঠিক — সম্পর্কিত products আলাদা factory-তে
interface UIComponentFactory { /* UI related */ }
interface DatabaseFactory    { /* DB related */ }
interface PaymentFactory     { /* Payment related */ }
```

### ❌ ভুল ৪: Abstract Factory যেখানে দরকার নেই সেখানে ব্যবহার

```php
// ❌ Overkill — শুধু একটি product type থাকলে Factory Method যথেষ্ট
interface SingleProductFactory
{
    public function createLogger(): Logger;
    // শুধু একটি method? এটা Factory Method, Abstract Factory না!
}

// ✅ এক্ষেত্রে Factory Method ব্যবহার করুন
abstract class LoggerFactory
{
    abstract protected function createLogger(): Logger;

    public function log(string $message): void
    {
        $this->createLogger()->write($message);
    }
}
```

### ❌ ভুল ৫: Factory-তে Business Logic রাখা

```php
// ❌ ভুল — factory-র কাজ শুধু object তৈরি করা, business logic না
class BadPaymentFactory implements PaymentGatewayFactory
{
    public function createPayment(): Payment
    {
        $payment = new BKashPayment();
        $payment->validateMerchant();  // ❌ business logic factory-তে না
        $payment->setCommission(1.5);  // ❌ configuration factory-র কাজ না
        return $payment;
    }
}

// ✅ সঠিক — factory শুধু তৈরি করে, configure/validate আলাদা layer-এ
class BKashFactory implements PaymentGatewayFactory
{
    public function createPayment(): Payment
    {
        return new BKashPayment(); // শুধু তৈরি করা
    }
}
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class AbstractFactoryTest extends TestCase
{
    #[Test]
    public function modern_factory_creates_all_modern_products(): void
    {
        $factory = new \ModernFurnitureFactory();

        $chair = $factory->createChair();
        $table = $factory->createTable();
        $sofa  = $factory->createSofa();

        $this->assertInstanceOf(\ModernChair::class, $chair);
        $this->assertInstanceOf(\ModernTable::class, $table);
        $this->assertInstanceOf(\ModernSofa::class, $sofa);
    }

    #[Test]
    public function all_products_from_same_factory_share_style(): void
    {
        $factory = new \VictorianFurnitureFactory();

        $chair = $factory->createChair();
        $table = $factory->createTable();
        $sofa  = $factory->createSofa();

        $this->assertSame('Victorian', $chair->getStyle());
        $this->assertSame('Victorian', $table->getStyle());
        $this->assertSame('Victorian', $sofa->getStyle());
    }

    #[Test]
    public function client_code_works_with_any_factory(): void
    {
        $factories = [
            new \ModernFurnitureFactory(),
            new \VictorianFurnitureFactory(),
        ];

        foreach ($factories as $factory) {
            $chair = $factory->createChair();
            $table = $factory->createTable();
            $sofa  = $factory->createSofa();

            // সব product-ই তাদের interface implement করে
            $this->assertInstanceOf(\Chair::class, $chair);
            $this->assertInstanceOf(\Table::class, $table);
            $this->assertInstanceOf(\Sofa::class, $sofa);

            // সব product-এর style একই
            $this->assertSame($chair->getStyle(), $table->getStyle());
            $this->assertSame($table->getStyle(), $sofa->getStyle());
        }
    }

    #[Test]
    #[DataProvider('factoryProvider')]
    public function products_have_valid_prices(string $factoryClass): void
    {
        /** @var \FurnitureFactory $factory */
        $factory = new $factoryClass();

        $this->assertGreaterThan(0, $factory->createChair()->getPrice());
        $this->assertGreaterThan(0, $factory->createTable()->getPrice());
        $this->assertGreaterThan(0, $factory->createSofa()->getPrice());
    }

    public static function factoryProvider(): array
    {
        return [
            'modern'    => [\ModernFurnitureFactory::class],
            'victorian' => [\VictorianFurnitureFactory::class],
        ];
    }

    #[Test]
    public function database_factory_products_are_consistent(): void
    {
        $factory = new \MySQLFactory();

        $conn = $factory->createConnection();
        $qb   = $factory->createQueryBuilder();

        $this->assertSame('mysql', $conn->getDriverName());
        $this->assertStringContainsString('`', $qb->select('users'));
    }

    #[Test]
    public function mock_factory_for_unit_testing(): void
    {
        // Mock factory দিয়ে dependency isolate করা
        $mockButton = $this->createMock(\Button::class);
        $mockButton->method('render')->willReturn('<button>Test</button>');

        $mockInput = $this->createMock(\Input::class);
        $mockInput->method('validate')->willReturn(true);

        $mockModal = $this->createMock(\Modal::class);

        $factory = $this->createMock(\UIComponentFactory::class);
        $factory->method('createButton')->willReturn($mockButton);
        $factory->method('createInput')->willReturn($mockInput);
        $factory->method('createModal')->willReturn($mockModal);

        // এখন FormBuilder test করা যায় কোনো real UI component ছাড়াই
        $formBuilder = new \FormBuilder($factory);
        $result = $formBuilder->buildLoginForm();

        $this->assertStringContainsString('Test', $result);
    }

    #[Test]
    public function payment_factory_returns_correct_family(): void
    {
        $bkash = new \BKashFactory();

        $payment  = $bkash->createPayment();
        $refund   = $bkash->createRefund();
        $transfer = $bkash->createTransfer();

        $this->assertInstanceOf(\BKashPayment::class, $payment);
        $this->assertInstanceOf(\BKashRefund::class, $refund);
        $this->assertInstanceOf(\BKashTransfer::class, $transfer);

        // Charge message-এ bKash keyword আছে কিনা
        $this->assertStringContainsString('bKash', $payment->charge(100, '01712345678'));
    }
}
```

### Jest (JavaScript)

```javascript
// abstract-factory.test.js

describe('Abstract Factory Pattern', () => {
    // Furniture Factory Tests
    describe('Furniture Factory', () => {
        test('ModernFactory creates all Modern products', () => {
            const factory = new ModernFurnitureFactory();

            expect(factory.createChair()).toBeInstanceOf(ModernChair);
            expect(factory.createTable()).toBeInstanceOf(ModernTable);
            expect(factory.createSofa()).toBeInstanceOf(ModernSofa);
        });

        test('All products from same factory share style', () => {
            const factory = new VictorianFurnitureFactory();

            const chair = factory.createChair();
            const table = factory.createTable();
            const sofa  = factory.createSofa();

            expect(chair.getStyle()).toBe('Victorian');
            expect(table.getStyle()).toBe('Victorian');
            expect(sofa.getStyle()).toBe('Victorian');
        });

        test.each([
            ['Modern', ModernFurnitureFactory],
            ['Victorian', VictorianFurnitureFactory],
        ])('%s factory products have consistent style', (expectedStyle, FactoryClass) => {
            const factory = new FactoryClass();
            const products = [
                factory.createChair(),
                factory.createTable(),
                factory.createSofa(),
            ];

            products.forEach(product => {
                expect(product.getStyle()).toBe(expectedStyle);
                expect(product.getPrice()).toBeGreaterThan(0);
            });
        });
    });

    // UI Theme Factory Tests
    describe('UI Theme Factory', () => {
        test('Light theme components render with light classes', () => {
            const factory = new LightThemeFactory();
            const button = factory.createButton('টেস্ট');

            expect(button.render()).toContain('bg-white');
            expect(button.render()).toContain('টেস্ট');
        });

        test('Dark theme components render with dark classes', () => {
            const factory = new DarkThemeFactory();
            const button = factory.createButton('টেস্ট');

            expect(button.render()).toContain('bg-gray-800');
        });

        test('FormBuilder works with any theme factory', () => {
            const factories = [new LightThemeFactory(), new DarkThemeFactory()];

            factories.forEach(factory => {
                const builder = new FormBuilder(factory);
                const form = builder.buildLoginForm();

                expect(form).toContain('লগইন ফর্ম');
                expect(form).toContain('Email');
                expect(form).toContain('Password');
            });
        });

        test('Input validation works across themes', () => {
            [new LightThemeFactory(), new DarkThemeFactory()].forEach(factory => {
                const input = factory.createInput();
                expect(input.validate('hello')).toBe(true);
                expect(input.validate('')).toBe(false);
                expect(input.validate('   ')).toBe(false);
            });
        });
    });

    // Mock Factory for isolated testing
    describe('Mock Factory Testing', () => {
        test('can use mock factory to isolate dependencies', () => {
            const mockFactory = {
                createButton: jest.fn(() => ({
                    render: () => '<button>Mock</button>',
                    onClick: (action) => `Mock clicked: ${action}`,
                })),
                createInput: jest.fn(() => ({
                    render: () => '<input />',
                    validate: (v) => v.trim() !== '',
                })),
                createModal: jest.fn(() => ({
                    render: () => '<div>Modal</div>',
                    open: (t, c) => `${t}: ${c}`,
                    close: () => 'closed',
                })),
            };

            const builder = new FormBuilder(mockFactory);
            const form = builder.buildLoginForm();

            expect(mockFactory.createButton).toHaveBeenCalled();
            expect(mockFactory.createInput).toHaveBeenCalled();
            expect(form).toContain('Mock');
        });
    });

    // Database Factory Tests
    describe('Database Factory', () => {
        test('MySQL factory produces MySQL-syntax queries', () => {
            const factory = new MySQLFactory();
            const qb = factory.createQueryBuilder();

            const query = qb.select('users', ['name']);
            expect(query).toContain('`name`');
            expect(query).toContain('`users`');
        });

        test('PostgreSQL factory produces PG-syntax queries', () => {
            const factory = new PostgreSQLFactory();
            const qb = factory.createQueryBuilder();

            const insertQuery = qb.insert('users', { name: 'test' });
            expect(insertQuery).toContain('RETURNING id');
            expect(insertQuery).toContain('"users"');
        });

        test('Connection lifecycle works correctly', () => {
            const factory = new MySQLFactory();
            const conn = factory.createConnection();

            expect(conn.isConnected).toBe(false);
            conn.connect();
            expect(conn.isConnected).toBe(true);
            conn.disconnect();
            expect(conn.isConnected).toBe(false);
        });
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### 1. Factory Method
**সম্পর্ক:** Abstract Factory-র প্রতিটি `create` method আসলে একেকটি Factory Method। Abstract Factory কে "Factory of Factories" বলা যায়।

```
Abstract Factory = Factory Method × N (multiple product types)
```

### 2. Builder Pattern
**সম্পর্ক:** Builder step-by-step একটি complex object তৈরি করে, Abstract Factory একবারেই family তৈরি করে। কখনো কখনো Abstract Factory-র ভেতরে Builder ব্যবহার হয় complex product তৈরি করতে।

```php
// Abstract Factory + Builder combination
class ReportFactory {
    public function createReport(): Report {
        return (new ReportBuilder())   // Builder ব্যবহার করে complex product তৈরি
            ->setHeader('Monthly Report')
            ->addChart('sales')
            ->addTable('transactions')
            ->build();
    }
}
```

### 3. Prototype Pattern
**সম্পর্ক:** Factory-তে object তৈরি expensive হলে Prototype clone ব্যবহার করা যায় (উপরে উদাহরণ দেওয়া হয়েছে)।

### 4. Singleton Pattern
**সম্পর্ক:** সাধারণত একটি application-এ একটিমাত্র factory instance দরকার হয়। তাই Abstract Factory কে Singleton হিসেবে implement করা common:

```php
final class ThemeFactoryManager
{
    private static ?UIComponentFactory $instance = null;

    public static function getInstance(): UIComponentFactory
    {
        if (self::$instance === null) {
            $theme = getenv('APP_THEME') ?: 'light';
            self::$instance = match ($theme) {
                'dark' => new DarkThemeFactory(),
                default => new LightThemeFactory(),
            };
        }
        return self::$instance;
    }

    private function __construct() {} // Singleton enforcement
}
```

### 5. Bridge Pattern
**সম্পর্ক:** Bridge abstraction কে implementation থেকে আলাদা করে। Abstract Factory-র factory selection part কে Bridge দিয়ে implement করা যায়:

```
Abstract Factory: কোন family তৈরি হবে সেটা ঠিক করে
Bridge: তৈরি হওয়া product-দের implementation switch করে
```

দুটো একসাথে ব্যবহার করলে: Factory ঠিক করে "কোন platform", Bridge ঠিক করে "কোন rendering engine"।

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **সম্পর্কিত product-দের family আছে** — এবং এগুলো একসাথে ব্যবহার হতে হবে
   - উদাহরণ: UI theme-এর Button + Input + Modal
2. **Product compatibility জরুরি** — ভিন্ন family-র product মেশানো যাবে না
   - উদাহরণ: bKash-এর Payment-এর সাথে Nagad-এর Refund ব্যবহার করা যাবে না
3. **Multiple platform/variant support** — এবং runtime-এ switch করতে হবে
   - উদাহরণ: Database driver, OS-specific UI, payment gateway
4. **Library/SDK তৈরি করছেন** — যেখানে user নিজের concrete implementation দিতে পারবে
5. **System-এর একটা অংশ বদলালে পুরো related অংশ বদলাতে হবে**

### ❌ ব্যবহার করবেন না যখন:

1. **শুধু একটি product type** — Factory Method যথেষ্ট
2. **Product-গুলো সম্পর্কিত না** — আলাদা Factory Method ব্যবহার করুন
3. **ছোট প্রজেক্ট** — interface ও class-এর সংখ্যা ছোট প্রজেক্টে burden
4. **Family বদলানোর সম্ভাবনা নেই** — শুধু একটি variant থাকলে pattern-এর কোনো সুবিধা নেই
5. **Product type ঘন ঘন বাড়ে** — প্রতিটি নতুন product type-এ সব factory modify করতে হবে

### Decision Flowchart:

```
একাধিক সম্পর্কিত product আছে?
├── না → Factory Method বা Simple Factory ব্যবহার করুন
└── হ্যাঁ → Product-গুলো কি একসাথে compatible হতে হবে?
    ├── না → আলাদা Factory Method ব্যবহার করুন
    └── হ্যাঁ → একাধিক family/variant আছে?
        ├── না → Concrete factory সরাসরি ব্যবহার করুন
        └── হ্যাঁ → ✅ Abstract Factory ব্যবহার করুন
```

---

## 📋 সারসংক্ষেপ

### মূল শিক্ষা

| বিষয় | সারাংশ |
|---|---|
| **কী** | সম্পর্কিত object-দের family তৈরি করার interface |
| **কেন** | Product compatibility নিশ্চিত করা + concrete classes থেকে decoupling |
| **কখন** | একাধিক variant/platform + সম্পর্কিত products + runtime switching |
| **কীভাবে** | Abstract Factory interface → Concrete Factory per family → Abstract Products → Concrete Products per family |
| **সুবিধা** | Consistency, OCP, SRP, loose coupling, testability |
| **অসুবিধা** | Complexity, নতুন product type যোগ করা কঠিন, বেশি কোড |

### মনে রাখার সূত্র

> **"Abstract Factory = পুরো পরিবার একসাথে।"**
>
> ঠিক যেমন Hatil showroom-এ গিয়ে "Modern Set দিন" বললে পুরো Modern Chair + Table + Sofa একসাথে পান — Abstract Factory ঠিক সেই কাজটাই কোডে করে। আপনি factory-কে বলেন "bKash দিন" — factory পুরো bKash Payment + Refund + Transfer ecosystem আপনাকে দিয়ে দেয়। কোনটা bKash, কোনটা Nagad — সেটা factory-র ব্যাপার, আপনার না।

### Quick Reference

```php
// ১. Abstract Factory interface তৈরি করুন
interface MyFactory {
    public function createA(): ProductA;
    public function createB(): ProductB;
}

// ২. প্রতিটি family-র জন্য Concrete Factory
class Family1Factory implements MyFactory { /* ... */ }
class Family2Factory implements MyFactory { /* ... */ }

// ৩. Client code শুধু interface ব্যবহার করে
function clientCode(MyFactory $factory): void {
    $a = $factory->createA();
    $b = $factory->createB();
}

// ৪. Runtime-এ factory select করুন
$factory = match ($config) {
    'family1' => new Family1Factory(),
    'family2' => new Family2Factory(),
};
clientCode($factory);
```

---

> **📝 লেখকের নোট:** Abstract Factory pattern বাংলাদেশের software industry-তে সবচেয়ে বেশি দেখা যায় payment gateway integration-এ (bKash, Nagad, SSLCommerz) এবং multi-database support-এ। Laravel framework নিজেই এই pattern ব্যাপকভাবে ব্যবহার করে — `DatabaseManager`, `CacheManager`, `SessionManager` — সবই Abstract Factory-র বাস্তব উদাহরণ। এই pattern বুঝলে আপনি framework-এর architecture অনেক ভালোভাবে বুঝতে পারবেন।
