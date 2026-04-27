# 🚶 Visitor প্যাটার্ন

## 📌 সংজ্ঞা

**Gang of Four (GoF) সংজ্ঞা:**
> *"Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates."*

Visitor প্যাটার্ন হলো একটি **Behavioral Design Pattern** যেটা আপনাকে একটি অবজেক্ট স্ট্রাকচারের এলিমেন্টগুলোর উপর নতুন অপারেশন সংজ্ঞায়িত করতে দেয় — **এলিমেন্ট ক্লাসগুলো পরিবর্তন না করেই**। এটি **Open/Closed Principle** এর একটি চমৎকার বাস্তবায়ন: নতুন behavior যোগ করার জন্য existing code modify করতে হয় না।

### মূল ধারণা: Double Dispatch

সাধারণ method call এ **single dispatch** হয় — কোন method execute হবে সেটা শুধুমাত্র receiver object এর type দ্বারা নির্ধারিত হয়। কিন্তু Visitor প্যাটার্নে **double dispatch** ঘটে — method selection নির্ভর করে **দুটি** অবজেক্টের type এর উপর: Element এবং Visitor উভয়ের উপর।

```
element.accept(visitor)     →  visitor.visitConcreteElement(this)
   ↑ 1st dispatch                    ↑ 2nd dispatch
(Element type অনুযায়ী)        (Visitor type অনুযায়ী)
```

### মূল চারটি অংশ:

| অংশ | ভূমিকা |
|------|--------|
| **Visitor** | প্রতিটি ConcreteElement এর জন্য visit method ঘোষণা করে |
| **ConcreteVisitor** | প্রতিটি element type এর জন্য নির্দিষ্ট operation বাস্তবায়ন করে |
| **Element** | `accept(visitor)` method ঘোষণা করে |
| **ConcreteElement** | `accept()` এ visitor কে নিজের reference পাঠায় (`visitor.visit(this)`) |

---

## 🏠 বাস্তব উদাহরণ

### বীমা পরিদর্শক (Insurance Agent)

ধরুন একজন বীমা এজেন্ট (Visitor) বিভিন্ন ধরনের ভবন পরিদর্শন করছেন:

```
🏠 আবাসিক ভবন    → বীমা এজেন্ট → আবাসিক মূল্যায়ন (ফ্যামিলি সাইজ, ফ্লোর সংখ্যা)
🏭 শিল্প কারখানা    → বীমা এজেন্ট → শিল্প মূল্যায়ন (মেশিনারি, ঝুঁকি মাত্রা)
🏥 হাসপাতাল       → বীমা এজেন্ট → চিকিৎসা মূল্যায়ন (বেড সংখ্যা, সরঞ্জাম)
🏫 স্কুল           → বীমা এজেন্ট → শিক্ষা প্রতিষ্ঠান মূল্যায়ন (ছাত্র সংখ্যা)
```

**একই এজেন্ট**, কিন্তু প্রতিটি ভবনের জন্য **ভিন্ন মূল্যায়ন পদ্ধতি**। ভবনগুলো তাদের কাঠামো পরিবর্তন করে না — শুধু দরজা খুলে দেয় (`accept`) এবং এজেন্ট তার কাজ করেন।

### বাংলাদেশ প্রসঙ্গ: NBR ট্যাক্স পরিদর্শক

জাতীয় রাজস্ব বোর্ডের (NBR) একজন ট্যাক্স ইন্সপেক্টর বিভিন্ন ব্যবসা পরিদর্শন করেন:

```
🛒 মুদি দোকান     → VAT 5%  + সরলীকৃত হিসাব
🖥️ IT কোম্পানি    → VAT 15% + সার্ভিস ট্যাক্স
🏗️ নির্মাণ সংস্থা   → VAT 7.5% + উৎসে কর
💊 ফার্মেসি        → VAT 0%  + ড্রাগ লাইসেন্স যাচাই
```

প্রতিটি ব্যবসা তাদের নিজস্ব ক্লাস — কিন্তু **ট্যাক্স ক্যালকুলেশন লজিক** ভিজিটরে থাকে।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────┐
│      <<interface>>          │
│         Visitor             │
├─────────────────────────────┤
│ + visitElementA(a: ElemA)   │
│ + visitElementB(b: ElemB)   │
└──────────┬──────────────────┘
           │ implements
     ┌─────┴──────┐
     │            │
┌────▼────┐  ┌───▼──────┐
│Concrete │  │Concrete  │
│Visitor1 │  │Visitor2  │
├─────────┤  ├──────────┤
│+visitA()│  │+visitA() │
│+visitB()│  │+visitB() │
└─────────┘  └──────────┘

┌─────────────────────────────┐
│      <<interface>>          │
│         Element             │
├─────────────────────────────┤
│ + accept(v: Visitor): void  │
└──────────┬──────────────────┘
           │ implements
     ┌─────┴──────┐
     │            │
┌────▼────┐  ┌───▼──────┐
│Concrete │  │Concrete  │
│ElementA │  │ElementB  │
├─────────┤  ├──────────┤
│+accept()│  │+accept() │
│ v.visit │  │ v.visit  │
│ ElemA   │  │ ElemB    │
│ (this)  │  │ (this)   │
└─────────┘  └──────────┘

 Double Dispatch ফ্লো:
 ═══════════════════════════════════════════
 client → element.accept(visitor)
              │
              └→ visitor.visitConcreteElement(this)
                            │
                            └→ নির্দিষ্ট লজিক execute
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1. Basic Visitor with Double Dispatch

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Element ইন্টারফেস — প্রতিটি element কে visitor গ্রহণ করতে হবে
interface Element
{
    public function accept(Visitor $visitor): mixed;
}

// Visitor ইন্টারফেস — প্রতিটি concrete element এর জন্য আলাদা method
interface Visitor
{
    public function visitCircle(Circle $circle): mixed;
    public function visitRectangle(Rectangle $rect): mixed;
    public function visitTriangle(Triangle $triangle): mixed;
}

// Concrete Elements
class Circle implements Element
{
    public function __construct(
        public readonly float $radius,
    ) {}

    public function accept(Visitor $visitor): mixed
    {
        return $visitor->visitCircle($this); // Double dispatch: 2nd dispatch
    }
}

class Rectangle implements Element
{
    public function __construct(
        public readonly float $width,
        public readonly float $height,
    ) {}

    public function accept(Visitor $visitor): mixed
    {
        return $visitor->visitRectangle($this);
    }
}

class Triangle implements Element
{
    public function __construct(
        public readonly float $a,
        public readonly float $b,
        public readonly float $c,
    ) {}

    public function accept(Visitor $visitor): mixed
    {
        return $visitor->visitTriangle($this);
    }
}

// Concrete Visitor — ক্ষেত্রফল হিসাব
class AreaCalculator implements Visitor
{
    public function visitCircle(Circle $circle): float
    {
        return M_PI * $circle->radius ** 2;
    }

    public function visitRectangle(Rectangle $rect): float
    {
        return $rect->width * $rect->height;
    }

    public function visitTriangle(Triangle $t): float
    {
        $s = ($t->a + $t->b + $t->c) / 2; // হেরনের সূত্র
        return sqrt($s * ($s - $t->a) * ($s - $t->b) * ($s - $t->c));
    }
}

// Concrete Visitor — পরিসীমা হিসাব
class PerimeterCalculator implements Visitor
{
    public function visitCircle(Circle $circle): float
    {
        return 2 * M_PI * $circle->radius;
    }

    public function visitRectangle(Rectangle $rect): float
    {
        return 2 * ($rect->width + $rect->height);
    }

    public function visitTriangle(Triangle $t): float
    {
        return $t->a + $t->b + $t->c;
    }
}

// ব্যবহার
$shapes = [
    new Circle(5.0),
    new Rectangle(4.0, 6.0),
    new Triangle(3.0, 4.0, 5.0),
];

$areaCalc = new AreaCalculator();
$perimCalc = new PerimeterCalculator();

foreach ($shapes as $shape) {
    $area = $shape->accept($areaCalc);       // 1st dispatch: accept()
    $perimeter = $shape->accept($perimCalc);  // visitor.visitX(this) → 2nd dispatch
    printf(
        "%s → ক্ষেত্রফল: %.2f, পরিসীমা: %.2f\n",
        $shape::class, $area, $perimeter
    );
}
```

#### JavaScript ES2022+

```javascript
// Element base class
class Shape {
    accept(visitor) {
        throw new Error('accept() must be implemented');
    }
}

class Circle extends Shape {
    #radius;
    constructor(radius) {
        super();
        this.#radius = radius;
    }
    get radius() { return this.#radius; }

    accept(visitor) {
        return visitor.visitCircle(this); // Double dispatch
    }
}

class Rectangle extends Shape {
    #width; #height;
    constructor(width, height) {
        super();
        this.#width = width;
        this.#height = height;
    }
    get width() { return this.#width; }
    get height() { return this.#height; }

    accept(visitor) {
        return visitor.visitRectangle(this);
    }
}

class Triangle extends Shape {
    #a; #b; #c;
    constructor(a, b, c) {
        super();
        this.#a = a; this.#b = b; this.#c = c;
    }
    get a() { return this.#a; }
    get b() { return this.#b; }
    get c() { return this.#c; }

    accept(visitor) {
        return visitor.visitTriangle(this);
    }
}

// Concrete Visitors
class AreaCalculator {
    visitCircle(circle) {
        return Math.PI * circle.radius ** 2;
    }
    visitRectangle(rect) {
        return rect.width * rect.height;
    }
    visitTriangle(t) {
        const s = (t.a + t.b + t.c) / 2;
        return Math.sqrt(s * (s - t.a) * (s - t.b) * (s - t.c));
    }
}

class PerimeterCalculator {
    visitCircle(circle) {
        return 2 * Math.PI * circle.radius;
    }
    visitRectangle(rect) {
        return 2 * (rect.width + rect.height);
    }
    visitTriangle(t) {
        return t.a + t.b + t.c;
    }
}

// ব্যবহার
const shapes = [new Circle(5), new Rectangle(4, 6), new Triangle(3, 4, 5)];
const areaCalc = new AreaCalculator();
const perimCalc = new PerimeterCalculator();

for (const shape of shapes) {
    const area = shape.accept(areaCalc);
    const perimeter = shape.accept(perimCalc);
    console.log(`${shape.constructor.name} → ক্ষেত্রফল: ${area.toFixed(2)}, পরিসীমা: ${perimeter.toFixed(2)}`);
}
```

---

### 2. বাংলাদেশ ট্যাক্স ক্যালকুলেশন ভিজিটর

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// পণ্য/সেবা ক্যাটাগরি — Element hierarchy
interface TaxableEntity
{
    public function accept(TaxVisitor $visitor): TaxResult;
    public function getRevenue(): float;
}

readonly class TaxResult
{
    public function __construct(
        public string $entityName,
        public float $revenue,
        public float $vatRate,
        public float $vatAmount,
        public float $supplementaryDuty,
        public float $totalTax,
    ) {}
}

class GroceryShop implements TaxableEntity
{
    public function __construct(
        private readonly string $name,
        private readonly float $monthlyRevenue,
        private readonly bool $hasTradeLicense = true,
    ) {}

    public function accept(TaxVisitor $visitor): TaxResult
    {
        return $visitor->visitGroceryShop($this);
    }

    public function getRevenue(): float { return $this->monthlyRevenue; }
    public function getName(): string { return $this->name; }
    public function hasTradeLicense(): bool { return $this->hasTradeLicense; }
}

class ITCompany implements TaxableEntity
{
    public function __construct(
        private readonly string $name,
        private readonly float $monthlyRevenue,
        private readonly bool $isExportOriented = false,
    ) {}

    public function accept(TaxVisitor $visitor): TaxResult
    {
        return $visitor->visitITCompany($this);
    }

    public function getRevenue(): float { return $this->monthlyRevenue; }
    public function getName(): string { return $this->name; }
    public function isExportOriented(): bool { return $this->isExportOriented; }
}

class Pharmacy implements TaxableEntity
{
    public function __construct(
        private readonly string $name,
        private readonly float $monthlyRevenue,
        private readonly bool $hasValidDrugLicense = true,
    ) {}

    public function accept(TaxVisitor $visitor): TaxResult
    {
        return $visitor->visitPharmacy($this);
    }

    public function getRevenue(): float { return $this->monthlyRevenue; }
    public function getName(): string { return $this->name; }
    public function hasValidDrugLicense(): bool { return $this->hasValidDrugLicense; }
}

class ConstructionFirm implements TaxableEntity
{
    public function __construct(
        private readonly string $name,
        private readonly float $monthlyRevenue,
        private readonly float $projectValue = 0,
    ) {}

    public function accept(TaxVisitor $visitor): TaxResult
    {
        return $visitor->visitConstructionFirm($this);
    }

    public function getRevenue(): float { return $this->monthlyRevenue; }
    public function getName(): string { return $this->name; }
    public function getProjectValue(): float { return $this->projectValue; }
}

// Visitor ইন্টারফেস
interface TaxVisitor
{
    public function visitGroceryShop(GroceryShop $shop): TaxResult;
    public function visitITCompany(ITCompany $company): TaxResult;
    public function visitPharmacy(Pharmacy $pharmacy): TaxResult;
    public function visitConstructionFirm(ConstructionFirm $firm): TaxResult;
}

// NBR ট্যাক্স ক্যালকুলেটর — বাংলাদেশ VAT আইন ২০১২ অনুযায়ী (সরলীকৃত)
class BangladeshVATCalculator implements TaxVisitor
{
    public function visitGroceryShop(GroceryShop $shop): TaxResult
    {
        $rate = 0.05; // ৫% ট্রেড VAT
        $vat = $shop->getRevenue() * $rate;
        return new TaxResult(
            entityName: $shop->getName(),
            revenue: $shop->getRevenue(),
            vatRate: $rate,
            vatAmount: $vat,
            supplementaryDuty: 0,
            totalTax: $vat,
        );
    }

    public function visitITCompany(ITCompany $company): TaxResult
    {
        // রপ্তানিমুখী IT কোম্পানি ২০২৪ পর্যন্ত কর অব্যাহতিপ্রাপ্ত
        $rate = $company->isExportOriented() ? 0.0 : 0.15;
        $vat = $company->getRevenue() * $rate;
        $sd = $company->getRevenue() * 0.05; // সম্পূরক শুল্ক
        return new TaxResult(
            entityName: $company->getName(),
            revenue: $company->getRevenue(),
            vatRate: $rate,
            vatAmount: $vat,
            supplementaryDuty: $company->isExportOriented() ? 0 : $sd,
            totalTax: $vat + ($company->isExportOriented() ? 0 : $sd),
        );
    }

    public function visitPharmacy(Pharmacy $pharmacy): TaxResult
    {
        $rate = 0.024; // ওষুধে ২.৪% VAT
        $vat = $pharmacy->getRevenue() * $rate;
        return new TaxResult(
            entityName: $pharmacy->getName(),
            revenue: $pharmacy->getRevenue(),
            vatRate: $rate,
            vatAmount: $vat,
            supplementaryDuty: 0,
            totalTax: $vat,
        );
    }

    public function visitConstructionFirm(ConstructionFirm $firm): TaxResult
    {
        $rate = 0.075; // নির্মাণে ৭.৫% VAT
        $vat = $firm->getRevenue() * $rate;
        $tds = $firm->getProjectValue() * 0.07; // উৎসে কর ৭%
        return new TaxResult(
            entityName: $firm->getName(),
            revenue: $firm->getRevenue(),
            vatRate: $rate,
            vatAmount: $vat,
            supplementaryDuty: $tds,
            totalTax: $vat + $tds,
        );
    }
}

// রিপোর্ট ভিজিটর — ট্যাক্স রিপোর্ট তৈরি
class TaxReportVisitor implements TaxVisitor
{
    private array $reports = [];

    public function visitGroceryShop(GroceryShop $shop): TaxResult
    {
        $result = (new BangladeshVATCalculator())->visitGroceryShop($shop);
        $this->reports[] = $this->formatReport($result, 'মুদি দোকান');
        return $result;
    }

    public function visitITCompany(ITCompany $company): TaxResult
    {
        $result = (new BangladeshVATCalculator())->visitITCompany($company);
        $this->reports[] = $this->formatReport($result, 'IT কোম্পানি');
        return $result;
    }

    public function visitPharmacy(Pharmacy $pharmacy): TaxResult
    {
        $result = (new BangladeshVATCalculator())->visitPharmacy($pharmacy);
        $this->reports[] = $this->formatReport($result, 'ফার্মেসি');
        return $result;
    }

    public function visitConstructionFirm(ConstructionFirm $firm): TaxResult
    {
        $result = (new BangladeshVATCalculator())->visitConstructionFirm($firm);
        $this->reports[] = $this->formatReport($result, 'নির্মাণ সংস্থা');
        return $result;
    }

    private function formatReport(TaxResult $r, string $type): string
    {
        return sprintf(
            "[%s] %s | আয়: ৳%.2f | VAT: ৳%.2f (%.1f%%) | মোট কর: ৳%.2f",
            $type, $r->entityName, $r->revenue, $r->vatAmount,
            $r->vatRate * 100, $r->totalTax
        );
    }

    public function getFullReport(): string
    {
        return "=== বাংলাদেশ VAT রিপোর্ট ===\n" . implode("\n", $this->reports);
    }
}

// ব্যবহার
$entities = [
    new GroceryShop('রহিম স্টোর', 500_000),
    new ITCompany('বাংলাটেক সলিউশনস', 2_000_000, isExportOriented: true),
    new ITCompany('ঢাকা সফটওয়্যার', 1_500_000, isExportOriented: false),
    new Pharmacy('লাইফ ফার্মেসি', 800_000),
    new ConstructionFirm('গ্রীন বিল্ডার্স', 5_000_000, projectValue: 50_000_000),
];

$reportVisitor = new TaxReportVisitor();
foreach ($entities as $entity) {
    $entity->accept($reportVisitor);
}
echo $reportVisitor->getFullReport();
```

#### JavaScript ES2022+

```javascript
// Tax Result DTO
class TaxResult {
    constructor({ entityName, revenue, vatRate, vatAmount, supplementaryDuty, totalTax }) {
        Object.assign(this, { entityName, revenue, vatRate, vatAmount, supplementaryDuty, totalTax });
        Object.freeze(this);
    }
}

// Taxable Entities
class GroceryShop {
    #name; #revenue; #hasTradeLicense;
    constructor(name, revenue, hasTradeLicense = true) {
        this.#name = name;
        this.#revenue = revenue;
        this.#hasTradeLicense = hasTradeLicense;
    }
    get name() { return this.#name; }
    get revenue() { return this.#revenue; }
    get hasTradeLicense() { return this.#hasTradeLicense; }
    accept(visitor) { return visitor.visitGroceryShop(this); }
}

class ITCompany {
    #name; #revenue; #exportOriented;
    constructor(name, revenue, exportOriented = false) {
        this.#name = name;
        this.#revenue = revenue;
        this.#exportOriented = exportOriented;
    }
    get name() { return this.#name; }
    get revenue() { return this.#revenue; }
    get exportOriented() { return this.#exportOriented; }
    accept(visitor) { return visitor.visitITCompany(this); }
}

class Pharmacy {
    #name; #revenue;
    constructor(name, revenue) {
        this.#name = name;
        this.#revenue = revenue;
    }
    get name() { return this.#name; }
    get revenue() { return this.#revenue; }
    accept(visitor) { return visitor.visitPharmacy(this); }
}

// Bangladesh VAT Calculator Visitor
class BangladeshVATCalculator {
    visitGroceryShop(shop) {
        const rate = 0.05;
        const vat = shop.revenue * rate;
        return new TaxResult({
            entityName: shop.name, revenue: shop.revenue,
            vatRate: rate, vatAmount: vat, supplementaryDuty: 0, totalTax: vat,
        });
    }

    visitITCompany(company) {
        const rate = company.exportOriented ? 0.0 : 0.15;
        const vat = company.revenue * rate;
        const sd = company.exportOriented ? 0 : company.revenue * 0.05;
        return new TaxResult({
            entityName: company.name, revenue: company.revenue,
            vatRate: rate, vatAmount: vat, supplementaryDuty: sd, totalTax: vat + sd,
        });
    }

    visitPharmacy(pharmacy) {
        const rate = 0.024;
        const vat = pharmacy.revenue * rate;
        return new TaxResult({
            entityName: pharmacy.name, revenue: pharmacy.revenue,
            vatRate: rate, vatAmount: vat, supplementaryDuty: 0, totalTax: vat,
        });
    }
}

// ব্যবহার
const entities = [
    new GroceryShop('রহিম স্টোর', 500_000),
    new ITCompany('বাংলাটেক', 2_000_000, true),
    new ITCompany('ঢাকা সফটওয়্যার', 1_500_000, false),
    new Pharmacy('লাইফ ফার্মেসি', 800_000),
];

const calculator = new BangladeshVATCalculator();
for (const entity of entities) {
    const result = entity.accept(calculator);
    console.log(`${result.entityName}: VAT ৳${result.vatAmount.toFixed(2)}, মোট কর ৳${result.totalTax.toFixed(2)}`);
}
```

---

### 3. AST (Abstract Syntax Tree) Visitor

কম্পাইলার ও ট্রান্সপাইলারে Visitor প্যাটার্ন সবচেয়ে বেশি ব্যবহৃত হয়। এখানে একটি সরল expression evaluator:

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// AST Node hierarchy
interface ASTNode
{
    public function accept(ASTVisitor $visitor): mixed;
}

interface ASTVisitor
{
    public function visitNumberLiteral(NumberLiteral $node): mixed;
    public function visitBinaryExpression(BinaryExpression $node): mixed;
    public function visitUnaryExpression(UnaryExpression $node): mixed;
    public function visitFunctionCall(FunctionCall $node): mixed;
}

class NumberLiteral implements ASTNode
{
    public function __construct(public readonly float $value) {}

    public function accept(ASTVisitor $visitor): mixed
    {
        return $visitor->visitNumberLiteral($this);
    }
}

enum BinaryOp: string
{
    case Add = '+';
    case Subtract = '-';
    case Multiply = '*';
    case Divide = '/';
    case Power = '**';
}

class BinaryExpression implements ASTNode
{
    public function __construct(
        public readonly ASTNode $left,
        public readonly BinaryOp $operator,
        public readonly ASTNode $right,
    ) {}

    public function accept(ASTVisitor $visitor): mixed
    {
        return $visitor->visitBinaryExpression($this);
    }
}

class UnaryExpression implements ASTNode
{
    public function __construct(
        public readonly string $operator,
        public readonly ASTNode $operand,
    ) {}

    public function accept(ASTVisitor $visitor): mixed
    {
        return $visitor->visitUnaryExpression($this);
    }
}

class FunctionCall implements ASTNode
{
    public function __construct(
        public readonly string $name,
        /** @var ASTNode[] */
        public readonly array $arguments,
    ) {}

    public function accept(ASTVisitor $visitor): mixed
    {
        return $visitor->visitFunctionCall($this);
    }
}

// Evaluator Visitor — AST থেকে ফলাফল হিসাব করে
class EvaluatorVisitor implements ASTVisitor
{
    public function visitNumberLiteral(NumberLiteral $node): float
    {
        return $node->value;
    }

    public function visitBinaryExpression(BinaryExpression $node): float
    {
        $left = $node->left->accept($this);
        $right = $node->right->accept($this);

        return match($node->operator) {
            BinaryOp::Add      => $left + $right,
            BinaryOp::Subtract => $left - $right,
            BinaryOp::Multiply => $left * $right,
            BinaryOp::Divide   => $right != 0 ? $left / $right : throw new \DivisionByZeroError(),
            BinaryOp::Power    => $left ** $right,
        };
    }

    public function visitUnaryExpression(UnaryExpression $node): float
    {
        $value = $node->operand->accept($this);
        return match($node->operator) {
            '-' => -$value,
            '+' => +$value,
            default => throw new \InvalidArgumentException("Unknown operator: {$node->operator}"),
        };
    }

    public function visitFunctionCall(FunctionCall $node): float
    {
        $args = array_map(fn(ASTNode $arg) => $arg->accept($this), $node->arguments);
        return match($node->name) {
            'sqrt' => sqrt($args[0]),
            'abs'  => abs($args[0]),
            'max'  => max(...$args),
            'min'  => min(...$args),
            default => throw new \RuntimeException("Unknown function: {$node->name}"),
        };
    }
}

// Pretty Printer Visitor — AST কে string এ রূপান্তর
class PrettyPrinterVisitor implements ASTVisitor
{
    public function visitNumberLiteral(NumberLiteral $node): string
    {
        return (string)$node->value;
    }

    public function visitBinaryExpression(BinaryExpression $node): string
    {
        $left = $node->left->accept($this);
        $right = $node->right->accept($this);
        return "({$left} {$node->operator->value} {$right})";
    }

    public function visitUnaryExpression(UnaryExpression $node): string
    {
        return "({$node->operator}" . $node->operand->accept($this) . ")";
    }

    public function visitFunctionCall(FunctionCall $node): string
    {
        $args = array_map(fn(ASTNode $a) => $a->accept($this), $node->arguments);
        return $node->name . '(' . implode(', ', $args) . ')';
    }
}

// AST তৈরি: sqrt(3^2 + 4^2)
$ast = new FunctionCall('sqrt', [
    new BinaryExpression(
        new BinaryExpression(new NumberLiteral(3), BinaryOp::Power, new NumberLiteral(2)),
        BinaryOp::Add,
        new BinaryExpression(new NumberLiteral(4), BinaryOp::Power, new NumberLiteral(2)),
    ),
]);

$evaluator = new EvaluatorVisitor();
$printer = new PrettyPrinterVisitor();

echo $printer->visitFunctionCall($ast) . "\n";   // sqrt(((3 ** 2) + (4 ** 2)))
echo "ফলাফল: " . $ast->accept($evaluator) . "\n"; // ফলাফল: 5
```

#### JavaScript ES2022+

```javascript
// AST Nodes
class NumberLiteral {
    constructor(value) { this.value = value; }
    accept(visitor) { return visitor.visitNumberLiteral(this); }
}

class BinaryExpression {
    constructor(left, operator, right) {
        this.left = left;
        this.operator = operator;
        this.right = right;
    }
    accept(visitor) { return visitor.visitBinaryExpression(this); }
}

class FunctionCall {
    constructor(name, args) {
        this.name = name;
        this.arguments = args;
    }
    accept(visitor) { return visitor.visitFunctionCall(this); }
}

// Evaluator Visitor
class Evaluator {
    visitNumberLiteral(node) { return node.value; }

    visitBinaryExpression(node) {
        const left = node.left.accept(this);
        const right = node.right.accept(this);
        const ops = { '+': (a, b) => a + b, '-': (a, b) => a - b,
                      '*': (a, b) => a * b, '/': (a, b) => a / b,
                      '**': (a, b) => a ** b };
        return ops[node.operator](left, right);
    }

    visitFunctionCall(node) {
        const args = node.arguments.map(a => a.accept(this));
        const fns = { sqrt: Math.sqrt, abs: Math.abs,
                      max: (...a) => Math.max(...a), min: (...a) => Math.min(...a) };
        return fns[node.name](...args);
    }
}

// Pretty Printer
class PrettyPrinter {
    visitNumberLiteral(node) { return String(node.value); }
    visitBinaryExpression(node) {
        return `(${node.left.accept(this)} ${node.operator} ${node.right.accept(this)})`;
    }
    visitFunctionCall(node) {
        return `${node.name}(${node.arguments.map(a => a.accept(this)).join(', ')})`;
    }
}

// sqrt(3² + 4²)
const ast = new FunctionCall('sqrt', [
    new BinaryExpression(
        new BinaryExpression(new NumberLiteral(3), '**', new NumberLiteral(2)),
        '+',
        new BinaryExpression(new NumberLiteral(4), '**', new NumberLiteral(2)),
    ),
]);

console.log(new PrettyPrinter().visitFunctionCall(ast)); // sqrt(((3 ** 2) + (4 ** 2)))
console.log('ফলাফল:', ast.accept(new Evaluator()));       // ফলাফল: 5
```

---

### 4. Export Visitor (JSON/XML/HTML)

ডকুমেন্ট এলিমেন্টগুলো একই থাকবে, শুধু export format পরিবর্তন হবে — এটি Visitor এর ক্লাসিক ব্যবহার।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface DocumentElement
{
    public function accept(ExportVisitor $visitor): string;
}

interface ExportVisitor
{
    public function visitHeading(Heading $h): string;
    public function visitParagraph(Paragraph $p): string;
    public function visitCodeBlock(CodeBlock $c): string;
    public function visitTable(Table $t): string;
}

class Heading implements DocumentElement
{
    public function __construct(
        public readonly string $text,
        public readonly int $level = 1,
    ) {}

    public function accept(ExportVisitor $visitor): string
    {
        return $visitor->visitHeading($this);
    }
}

class Paragraph implements DocumentElement
{
    public function __construct(public readonly string $text) {}

    public function accept(ExportVisitor $visitor): string
    {
        return $visitor->visitParagraph($this);
    }
}

class CodeBlock implements DocumentElement
{
    public function __construct(
        public readonly string $code,
        public readonly string $language = 'php',
    ) {}

    public function accept(ExportVisitor $visitor): string
    {
        return $visitor->visitCodeBlock($this);
    }
}

class Table implements DocumentElement
{
    public function __construct(
        public readonly array $headers,
        public readonly array $rows,
    ) {}

    public function accept(ExportVisitor $visitor): string
    {
        return $visitor->visitTable($this);
    }
}

// HTML Exporter
class HTMLExportVisitor implements ExportVisitor
{
    public function visitHeading(Heading $h): string
    {
        return "<h{$h->level}>{$this->escape($h->text)}</h{$h->level}>";
    }

    public function visitParagraph(Paragraph $p): string
    {
        return "<p>{$this->escape($p->text)}</p>";
    }

    public function visitCodeBlock(CodeBlock $c): string
    {
        return "<pre><code class=\"language-{$c->language}\">{$this->escape($c->code)}</code></pre>";
    }

    public function visitTable(Table $t): string
    {
        $html = '<table><thead><tr>';
        foreach ($t->headers as $h) {
            $html .= "<th>{$this->escape($h)}</th>";
        }
        $html .= '</tr></thead><tbody>';
        foreach ($t->rows as $row) {
            $html .= '<tr>' . implode('', array_map(
                fn($cell) => "<td>{$this->escape($cell)}</td>", $row
            )) . '</tr>';
        }
        return $html . '</tbody></table>';
    }

    private function escape(string $text): string
    {
        return htmlspecialchars($text, ENT_QUOTES | ENT_HTML5);
    }
}

// JSON Exporter
class JSONExportVisitor implements ExportVisitor
{
    public function visitHeading(Heading $h): string
    {
        return json_encode(['type' => 'heading', 'level' => $h->level, 'text' => $h->text]);
    }

    public function visitParagraph(Paragraph $p): string
    {
        return json_encode(['type' => 'paragraph', 'text' => $p->text]);
    }

    public function visitCodeBlock(CodeBlock $c): string
    {
        return json_encode(['type' => 'code', 'language' => $c->language, 'content' => $c->code]);
    }

    public function visitTable(Table $t): string
    {
        return json_encode(['type' => 'table', 'headers' => $t->headers, 'rows' => $t->rows]);
    }
}

// Markdown Exporter
class MarkdownExportVisitor implements ExportVisitor
{
    public function visitHeading(Heading $h): string
    {
        return str_repeat('#', $h->level) . " {$h->text}";
    }

    public function visitParagraph(Paragraph $p): string
    {
        return $p->text;
    }

    public function visitCodeBlock(CodeBlock $c): string
    {
        return "```{$c->language}\n{$c->code}\n```";
    }

    public function visitTable(Table $t): string
    {
        $md = '| ' . implode(' | ', $t->headers) . " |\n";
        $md .= '| ' . implode(' | ', array_fill(0, count($t->headers), '---')) . " |\n";
        foreach ($t->rows as $row) {
            $md .= '| ' . implode(' | ', $row) . " |\n";
        }
        return $md;
    }
}

// ব্যবহার
$document = [
    new Heading('বাংলাদেশ প্রযুক্তি রিপোর্ট', 1),
    new Paragraph('এই রিপোর্টে বাংলাদেশের IT সেক্টরের বিশ্লেষণ রয়েছে।'),
    new Table(
        headers: ['কোম্পানি', 'আয় (কোটি)', 'কর্মী সংখ্যা'],
        rows: [
            ['বাংলাটেক', '১২০', '৫০০'],
            ['ঢাকা সফটওয়্যার', '৮৫', '৩২০'],
        ]
    ),
    new CodeBlock('echo "Hello Bangladesh!";', 'php'),
];

$exporters = [
    'HTML'     => new HTMLExportVisitor(),
    'JSON'     => new JSONExportVisitor(),
    'Markdown' => new MarkdownExportVisitor(),
];

foreach ($exporters as $format => $visitor) {
    echo "=== {$format} ===\n";
    foreach ($document as $element) {
        echo $element->accept($visitor) . "\n";
    }
    echo "\n";
}
```

---

## 🌍 Real-World Applicable Areas

### ১. কম্পাইলার / ইন্টারপ্রেটার (AST Traversal)

Visitor প্যাটার্নের সবচেয়ে ক্লাসিক ব্যবহার। **PHP-Parser** লাইব্রেরি (`nikic/php-parser`) এটি ব্যাপকভাবে ব্যবহার করে:

```php
// PHP-Parser এ Visitor ব্যবহার
use PhpParser\NodeVisitorAbstract;
use PhpParser\Node;

class DeprecationFinder extends NodeVisitorAbstract
{
    public array $deprecated = [];

    public function enterNode(Node $node): void
    {
        if ($node instanceof Node\Stmt\Function_) {
            $docComment = $node->getDocComment();
            if ($docComment && str_contains($docComment->getText(), '@deprecated')) {
                $this->deprecated[] = $node->name->toString();
            }
        }
    }
}
```

### ২. Babel.js AST Transformations

```javascript
// Babel plugin — visitor pattern ব্যবহার করে AST transform
export default function myBabelPlugin() {
    return {
        visitor: {
            // প্রতিটি node type এর জন্য visit method
            Identifier(path) {
                if (path.node.name === 'oldName') {
                    path.node.name = 'newName';
                }
            },
            ArrowFunctionExpression(path) {
                // Arrow function কে regular function এ রূপান্তর
            },
        },
    };
}
```

### ৩. ডকুমেন্ট এক্সপোর্ট

একই ডকুমেন্ট স্ট্রাকচার → বিভিন্ন ফরম্যাট (PDF, HTML, Markdown, DOCX)।

### ৪. ট্যাক্স / প্রাইসিং ক্যালকুলেশন

বিভিন্ন পণ্য ক্যাটাগরিতে ভিন্ন ভিন্ন কর/মূল্য নির্ধারণ — উপরের Bangladesh VAT উদাহরণ।

### ৫. রিপোর্টিং সিস্টেম

বিভিন্ন entity type এর উপর একই ধরনের রিপোর্ট তৈরি (monthly, yearly, audit)।

### ৬. Static Code Analysis

PHPStan, ESLint — এরা AST traverse করে Visitor প্যাটার্নে code analysis চালায়।

### ৭. Serialization / Deserialization

অবজেক্ট স্ট্রাকচারকে বিভিন্ন ফরম্যাটে serialize করা (JSON, XML, ProtoBuf, MessagePack)।

---

## 🔥 Advanced Deep Dive

### Double Dispatch কেন দরকার?

PHP এবং JavaScript এ **method overloading** নেই (PHP তে partial আছে, JS তে একদম নেই)। তাই আমরা সরাসরি করতে পারি না:

```php
// এটি PHP তে সম্ভব নয়!
class Calculator {
    public function calculate(Circle $c): float { ... }
    public function calculate(Rectangle $r): float { ... } // Fatal error!
}
```

Double dispatch এই সমস্যা সমাধান করে:

```
১। element.accept(visitor)           → Element এর type অনুযায়ী accept() কল হয়
২। visitor.visitSpecificElement(this) → Visitor এর type + Element এর type দুটোই resolved
```

**প্রথম dispatch**: `accept()` call — কোন Element এর accept কল হবে (polymorphism)
**দ্বিতীয় dispatch**: `visitConcreteElement()` — কোন visitor method কল হবে (explicit method name)

### Visitor vs Strategy

| বৈশিষ্ট্য | Visitor | Strategy |
|-----------|---------|----------|
| **উদ্দেশ্য** | একটি object structure এর উপর নতুন operation যোগ | একটি algorithm পরিবর্তনযোগ্য করা |
| **ফোকাস** | একাধিক element type, প্রতিটিতে ভিন্ন behavior | একটি context, algorithm interchangeable |
| **Dispatch** | Double dispatch (element + visitor) | Single dispatch (strategy object) |
| **নতুন যোগ** | নতুন operation সহজে যোগ হয় | নতুন algorithm সহজে যোগ হয় |
| **Element জানে** | শুধু accept() জানে | Context strategy কে delegate করে |

### Visitor vs Iterator

| বৈশিষ্ট্য | Visitor | Iterator |
|-----------|---------|----------|
| **traversal** | কাঠামো জানে, প্রতিটি element এ ভিন্ন কাজ | শুধু sequential access |
| **element type** | প্রতিটি type এর জন্য আলাদা method | সব element একই ভাবে traverse |
| **operation** | operation ভিজিটরে থাকে | operation বাইরে থাকে |

### Visitor + Composite (Tree Structure Visit)

Composite প্যাটার্নের সাথে Visitor অত্যন্ত শক্তিশালী — tree structure traverse করে প্রতিটি node এ ভিন্ন operation চালানো যায়:

```php
<?php

// File system — Composite + Visitor
interface FileSystemNode
{
    public function accept(FileSystemVisitor $visitor): mixed;
}

interface FileSystemVisitor
{
    public function visitFile(File $file): mixed;
    public function visitDirectory(Directory $dir): mixed;
}

class File implements FileSystemNode
{
    public function __construct(
        public readonly string $name,
        public readonly int $sizeBytes,
        public readonly string $extension,
    ) {}

    public function accept(FileSystemVisitor $visitor): mixed
    {
        return $visitor->visitFile($this);
    }
}

class Directory implements FileSystemNode
{
    /** @var FileSystemNode[] */
    private array $children = [];

    public function __construct(public readonly string $name) {}

    public function add(FileSystemNode $node): void
    {
        $this->children[] = $node;
    }

    /** @return FileSystemNode[] */
    public function getChildren(): array { return $this->children; }

    public function accept(FileSystemVisitor $visitor): mixed
    {
        return $visitor->visitDirectory($this);
    }
}

// মোট সাইজ হিসাব করার ভিজিটর
class SizeCalculatorVisitor implements FileSystemVisitor
{
    public function visitFile(File $file): int
    {
        return $file->sizeBytes;
    }

    public function visitDirectory(Directory $dir): int
    {
        $total = 0;
        foreach ($dir->getChildren() as $child) {
            $total += $child->accept($this);
        }
        return $total;
    }
}

// নির্দিষ্ট extension এর ফাইল খুঁজে বের করার ভিজিটর
class FileFinderVisitor implements FileSystemVisitor
{
    private array $found = [];

    public function __construct(private readonly string $targetExtension) {}

    public function visitFile(File $file): void
    {
        if ($file->extension === $this->targetExtension) {
            $this->found[] = $file->name;
        }
    }

    public function visitDirectory(Directory $dir): void
    {
        foreach ($dir->getChildren() as $child) {
            $child->accept($this);
        }
    }

    public function getFound(): array { return $this->found; }
}
```

### Acyclic Visitor

Traditional Visitor এ Visitor interface এ সব concrete element এর visit method থাকতে হয় — এতে নতুন element যোগ করলে Visitor interface ভেঙে যায়। **Acyclic Visitor** এই সমস্যা সমাধান করে:

```php
<?php

// Acyclic Visitor — degenerate interface per element type
interface AcyclicVisitor {} // marker interface

interface CircleVisitor
{
    public function visitCircle(Circle $c): mixed;
}

interface RectangleVisitor
{
    public function visitRectangle(Rectangle $r): mixed;
}

class Circle
{
    public function __construct(public readonly float $radius) {}

    public function accept(AcyclicVisitor $visitor): mixed
    {
        if ($visitor instanceof CircleVisitor) {
            return $visitor->visitCircle($this);
        }
        return null; // এই visitor Circle handle করে না
    }
}

class AreaCalc implements AcyclicVisitor, CircleVisitor, RectangleVisitor
{
    public function visitCircle(Circle $c): float
    {
        return M_PI * $c->radius ** 2;
    }

    public function visitRectangle(Rectangle $r): float
    {
        return $r->width * $r->height;
    }
}
```

**সুবিধা**: নতুন element যোগ করলে existing Visitor ভাঙে না।
**অসুবিধা**: Runtime type check (`instanceof`) — compile-time safety কমে যায়।

### Functional Programming এ Visitor (Pattern Matching)

Visitor আসলে **pattern matching** এর OOP সমতুল্য:

```javascript
// Functional visitor — object-based pattern matching
const evaluate = (node) => {
    const visitors = {
        NumberLiteral: (n) => n.value,
        BinaryExpression: (n) => {
            const ops = { '+': (a, b) => a + b, '-': (a, b) => a - b,
                          '*': (a, b) => a * b, '/': (a, b) => a / b };
            return ops[n.operator](evaluate(n.left), evaluate(n.right));
        },
        FunctionCall: (n) => {
            const fns = { sqrt: Math.sqrt, abs: Math.abs };
            return fns[n.name](...n.arguments.map(evaluate));
        },
    };

    const handler = visitors[node.constructor.name];
    if (!handler) throw new Error(`Unknown node: ${node.constructor.name}`);
    return handler(node);
};
```

### PHP `match` Expression — Lightweight Visitor

PHP 8.0+ এর `match` expression কে lightweight visitor হিসেবে ব্যবহার করা যায়:

```php
<?php

function calculateArea(object $shape): float
{
    return match(true) {
        $shape instanceof Circle    => M_PI * $shape->radius ** 2,
        $shape instanceof Rectangle => $shape->width * $shape->height,
        $shape instanceof Triangle  => (function() use ($shape) {
            $s = ($shape->a + $shape->b + $shape->c) / 2;
            return sqrt($s * ($s - $shape->a) * ($s - $shape->b) * ($s - $shape->c));
        })(),
        default => throw new \InvalidArgumentException('Unknown shape: ' . $shape::class),
    };
}
```

> ⚠️ এটি true Visitor নয় — OCP ভায়োলেট হয়। কিন্তু ছোট কেসে pragmatic।

---

## ✅ Pros (সুবিধাসমূহ)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **Open/Closed Principle** | নতুন operation যোগ করতে element class modify করতে হয় না |
| ২ | **Single Responsibility** | সম্পর্কিত operations এক visitor class এ গ্রুপ করা যায় |
| ৩ | **Accumulation** | Visitor traverse করতে করতে state accumulate করতে পারে |
| ৪ | **Type Safety** | প্রতিটি element type এর জন্য dedicated method — compile-time checking |
| ৫ | **Extensibility** | নতুন visitor যোগ করা সহজ — existing code untouched |

## ❌ Cons (অসুবিধাসমূহ)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **নতুন Element যোগ কঠিন** | নতুন ConcreteElement যোগ করলে সব Visitor interface আপডেট করতে হয় |
| ২ | **Encapsulation ভাঙে** | Visitor কে element এর internal state access করতে হয় |
| ৩ | **জটিলতা** | ছোট hierarchy তে over-engineering মনে হতে পারে |
| ৪ | **Circular Dependency** | Visitor ↔ Element একে অপরকে জানে |
| ৫ | **বোঝা কঠিন** | Double dispatch নতুনদের জন্য confusing |

---

## ⚠️ Common Mistakes (সচরাচর ভুলসমূহ)

### ভুল ১: Element এ লজিক রাখা

```php
// ❌ ভুল — visitor ব্যবহার করছেন কিন্তু element এ লজিক
class Circle implements Element
{
    public function accept(Visitor $visitor): float
    {
        return M_PI * $this->radius ** 2; // visitor কে ignore করছে!
    }
}

// ✅ সঠিক — visitor কে delegate
class Circle implements Element
{
    public function accept(Visitor $visitor): mixed
    {
        return $visitor->visitCircle($this);
    }
}
```

### ভুল ২: accept() এ this পাঠাতে ভুলে যাওয়া

```php
// ❌ ভুল — this পাঠাচ্ছে না
public function accept(Visitor $visitor): mixed
{
    return $visitor->visitCircle(new Circle(5)); // নতুন object তৈরি করছে!
}

// ✅ সঠিক
public function accept(Visitor $visitor): mixed
{
    return $visitor->visitCircle($this);
}
```

### ভুল ৩: অস্থির Element hierarchy তে Visitor ব্যবহার

```
যদি Element type ঘন ঘন যোগ/বাদ হয়, Visitor ব্যবহার করবেন না।
Visitor সবচেয়ে ভালো কাজ করে যখন Element hierarchy স্থিতিশীল কিন্তু
operation ঘন ঘন পরিবর্তন/যোগ হয়।
```

### ভুল ৪: Composite traverse এ children visit ভুলে যাওয়া

```php
// ❌ ভুল — children traverse করেনি
public function visitDirectory(Directory $dir): int
{
    return 0; // children এর সাইজ মিস!
}

// ✅ সঠিক
public function visitDirectory(Directory $dir): int
{
    $total = 0;
    foreach ($dir->getChildren() as $child) {
        $total += $child->accept($this);
    }
    return $total;
}
```

### ভুল ৫: Visitor কে Stateful করে concurrent ব্যবহার

```php
// ❌ বিপদজনক — shared mutable state
class CounterVisitor implements Visitor
{
    public int $count = 0;
    // concurrent environment এ race condition হবে!
}

// ✅ নিরাপদ — প্রতিটি traversal এ নতুন instance
$visitor = new CounterVisitor();
$element->accept($visitor); // শুধু একটি traversal এ ব্যবহার
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

class AreaCalculatorTest extends TestCase
{
    private AreaCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new AreaCalculator();
    }

    #[Test]
    public function circleAreaIsCorrect(): void
    {
        $circle = new Circle(5.0);
        $area = $circle->accept($this->calculator);
        $this->assertEqualsWithDelta(78.539, $area, 0.001);
    }

    #[Test]
    public function rectangleAreaIsCorrect(): void
    {
        $rect = new Rectangle(4.0, 6.0);
        $area = $rect->accept($this->calculator);
        $this->assertSame(24.0, $area);
    }

    #[Test]
    public function triangleAreaUsesHeronsFormula(): void
    {
        $triangle = new Triangle(3.0, 4.0, 5.0);
        $area = $triangle->accept($this->calculator);
        $this->assertEqualsWithDelta(6.0, $area, 0.001);
    }

    #[Test]
    #[DataProvider('shapeProvider')]
    public function shapesAcceptMultipleVisitors(Element $shape): void
    {
        $areaCalc = new AreaCalculator();
        $perimCalc = new PerimeterCalculator();

        $area = $shape->accept($areaCalc);
        $perimeter = $shape->accept($perimCalc);

        $this->assertIsFloat($area);
        $this->assertIsFloat($perimeter);
        $this->assertGreaterThan(0, $area);
        $this->assertGreaterThan(0, $perimeter);
    }

    public static function shapeProvider(): array
    {
        return [
            'circle'    => [new Circle(3.0)],
            'rectangle' => [new Rectangle(5.0, 3.0)],
            'triangle'  => [new Triangle(3.0, 4.0, 5.0)],
        ];
    }
}

class BangladeshVATCalculatorTest extends TestCase
{
    private BangladeshVATCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new BangladeshVATCalculator();
    }

    #[Test]
    public function groceryShopGets5PercentVAT(): void
    {
        $shop = new GroceryShop('টেস্ট দোকান', 100_000);
        $result = $shop->accept($this->calculator);

        $this->assertSame(0.05, $result->vatRate);
        $this->assertSame(5_000.0, $result->vatAmount);
        $this->assertSame(5_000.0, $result->totalTax);
    }

    #[Test]
    public function exportOrientedITCompanyGetsZeroVAT(): void
    {
        $company = new ITCompany('এক্সপোর্ট কোম্পানি', 1_000_000, isExportOriented: true);
        $result = $company->accept($this->calculator);

        $this->assertSame(0.0, $result->vatRate);
        $this->assertSame(0.0, $result->vatAmount);
        $this->assertSame(0.0, $result->totalTax);
    }

    #[Test]
    public function domesticITCompanyGets15PercentVAT(): void
    {
        $company = new ITCompany('লোকাল কোম্পানি', 1_000_000, isExportOriented: false);
        $result = $company->accept($this->calculator);

        $this->assertSame(0.15, $result->vatRate);
        $this->assertSame(150_000.0, $result->vatAmount);
        $this->assertGreaterThan(150_000.0, $result->totalTax); // SD যোগ হবে
    }

    #[Test]
    public function pharmacyGetsReducedVAT(): void
    {
        $pharmacy = new Pharmacy('টেস্ট ফার্মেসি', 500_000);
        $result = $pharmacy->accept($this->calculator);

        $this->assertSame(0.024, $result->vatRate);
        $this->assertEqualsWithDelta(12_000.0, $result->vatAmount, 0.01);
    }

    #[Test]
    public function doubleDispatchEnsuresCorrectVisitorMethodIsCalled(): void
    {
        $visitor = $this->createMock(TaxVisitor::class);

        $visitor->expects($this->once())
            ->method('visitGroceryShop')
            ->willReturn(new TaxResult('mock', 0, 0, 0, 0, 0));

        // accept() কল করলে সঠিক visit method কল হবে
        $shop = new GroceryShop('টেস্ট', 100);
        $shop->accept($visitor);
    }
}
```

### Jest

```javascript
// visitor.test.js
describe('AreaCalculator Visitor', () => {
    const calculator = new AreaCalculator();

    test('Circle area calculation', () => {
        const circle = new Circle(5);
        const area = circle.accept(calculator);
        expect(area).toBeCloseTo(78.539, 2);
    });

    test('Rectangle area calculation', () => {
        const rect = new Rectangle(4, 6);
        expect(rect.accept(calculator)).toBe(24);
    });

    test('Triangle uses Heron\'s formula', () => {
        const triangle = new Triangle(3, 4, 5);
        expect(triangle.accept(calculator)).toBeCloseTo(6.0, 2);
    });
});

describe('Bangladesh VAT Calculator', () => {
    const calculator = new BangladeshVATCalculator();

    test('Grocery shop gets 5% VAT', () => {
        const shop = new GroceryShop('টেস্ট', 100_000);
        const result = shop.accept(calculator);
        expect(result.vatRate).toBe(0.05);
        expect(result.vatAmount).toBe(5000);
    });

    test('Export-oriented IT company gets zero VAT', () => {
        const company = new ITCompany('এক্সপোর্ট', 1_000_000, true);
        const result = company.accept(calculator);
        expect(result.vatRate).toBe(0);
        expect(result.totalTax).toBe(0);
    });

    test('Domestic IT company gets 15% VAT + supplementary duty', () => {
        const company = new ITCompany('লোকাল', 1_000_000, false);
        const result = company.accept(calculator);
        expect(result.vatRate).toBe(0.15);
        expect(result.totalTax).toBeGreaterThan(result.vatAmount);
    });

    test('Double dispatch routes to correct method', () => {
        const mockVisitor = {
            visitGroceryShop: jest.fn().mockReturnValue(new TaxResult({
                entityName: 'mock', revenue: 0, vatRate: 0,
                vatAmount: 0, supplementaryDuty: 0, totalTax: 0,
            })),
            visitITCompany: jest.fn(),
            visitPharmacy: jest.fn(),
        };

        new GroceryShop('টেস্ট', 100).accept(mockVisitor);
        expect(mockVisitor.visitGroceryShop).toHaveBeenCalledTimes(1);
        expect(mockVisitor.visitITCompany).not.toHaveBeenCalled();
    });
});

describe('AST Evaluator Visitor', () => {
    const evaluator = new Evaluator();

    test('evaluates simple addition', () => {
        const ast = new BinaryExpression(
            new NumberLiteral(3), '+', new NumberLiteral(4)
        );
        expect(ast.accept(evaluator)).toBe(7);
    });

    test('evaluates nested expression: sqrt(3² + 4²) = 5', () => {
        const ast = new FunctionCall('sqrt', [
            new BinaryExpression(
                new BinaryExpression(new NumberLiteral(3), '**', new NumberLiteral(2)),
                '+',
                new BinaryExpression(new NumberLiteral(4), '**', new NumberLiteral(2)),
            ),
        ]);
        expect(ast.accept(evaluator)).toBeCloseTo(5, 5);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Composite + Visitor
**সবচেয়ে শক্তিশালী জোড়া।** Composite tree structure তৈরি করে, Visitor সেই tree traverse করে প্রতিটি node এ operation চালায়। File system, DOM, AST — সবখানে এই কম্বিনেশন কাজ করে।

### Iterator + Visitor
Iterator element collection traverse করে, Visitor প্রতিটি element এ নির্দিষ্ট operation চালায়। Iterator **কিভাবে traverse** করবে সেটা handle করে, Visitor **কি করবে** সেটা handle করে।

### Strategy vs Visitor
Strategy একটি object এর একটি behavior পরিবর্তন করে। Visitor একটি **object structure** এর সব element এর উপর operation চালায়। Strategy = single object focused, Visitor = structure focused।

### Interpreter + Visitor
Interpreter গ্রামার define করে, Visitor সেই গ্রামারের node গুলোতে বিভিন্ন operation (evaluation, optimization, pretty-printing) চালায়।

```
সম্পর্ক ম্যাপ:

Composite ──── tree structure তৈরি ────→ Visitor traverse করে
Iterator  ──── sequential access ──────→ Visitor operation চালায়
Strategy  ──── single algorithm swap ──→ Visitor multi-element operation
Interpreter ── grammar/AST define ────→ Visitor বিভিন্ন interpretation
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

| পরিস্থিতি | উদাহরণ |
|-----------|---------|
| **Element hierarchy স্থিতিশীল** কিন্তু **operation ঘন ঘন যোগ** হয় | Shape types fixed, কিন্তু নতুন নতুন calculation যোগ হয় |
| **একই structure এ একাধিক unrelated operation** | একটি AST তে evaluation, printing, optimization, linting |
| **Cross-cutting concerns** আলাদা করতে চান | Tax calculation, reporting, validation আলাদা visitor এ |
| **Composite tree structure traverse** করে বিভিন্ন কাজ করতে চান | File system analysis, DOM manipulation |
| **Double dispatch দরকার** | Operation নির্ভর করে element type + operation type দুটোর উপর |

### ❌ ব্যবহার করবেন না যখন:

| পরিস্থিতি | কেন না |
|-----------|--------|
| **Element hierarchy ঘন ঘন পরিবর্তন হয়** | প্রতিটি নতুন element এ সব visitor আপডেট করতে হবে |
| **অল্প সংখ্যক operation** | Over-engineering — সরাসরি method যোগ করুন |
| **Element এর internal data expose** করতে না চাইলে | Visitor encapsulation ভাঙে |
| **Simple polymorphism যথেষ্ট** | শুধু একটি operation থাকলে visitor দরকার নেই |
| **Performance critical code** | Double dispatch এর overhead থাকে |

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট:

```
Element types ঘন ঘন যোগ হয়?
├── হ্যাঁ → Visitor ব্যবহার করবেন না ❌
└── না → Operations ঘন ঘন যোগ হয়?
    ├── না → সরাসরি method যোগ করুন
    └── হ্যাঁ → Element types কি heterogeneous?
        ├── না → Strategy/Template Method ব্যবহার করুন
        └── হ্যাঁ → ✅ Visitor ব্যবহার করুন!
```

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────┐
│              🚶 Visitor প্যাটার্ন সারসংক্ষেপ          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  🎯 উদ্দেশ্য:                                        │
│  Element class পরিবর্তন না করে নতুন operation যোগ   │
│                                                     │
│  🔑 মূল প্রক্রিয়া: Double Dispatch                   │
│  element.accept(visitor) → visitor.visitX(this)     │
│                                                     │
│  ✅ শক্তি:                                           │
│  • OCP মেনে চলে (নতুন operation = নতুন visitor)     │
│  • SRP মেনে চলে (প্রতিটি operation আলাদা class)     │
│  • State accumulate করতে পারে                       │
│                                                     │
│  ❌ দুর্বলতা:                                        │
│  • নতুন Element যোগ করা কঠিন                        │
│  • Encapsulation ভাঙে                               │
│  • নতুনদের জন্য জটিল                                │
│                                                     │
│  🏆 সেরা ব্যবহার:                                    │
│  • AST/Compiler (PHP-Parser, Babel)                 │
│  • Document export (HTML/JSON/XML)                  │
│  • Tax/Pricing calculation                          │
│  • File system analysis                             │
│  • Static code analysis (PHPStan, ESLint)           │
│                                                     │
│  📐 মনে রাখুন:                                      │
│  "Element স্থিতিশীল, Operation পরিবর্তনশীল"         │
│  → Visitor ব্যবহার করুন                              │
│  "Element পরিবর্তনশীল, Operation স্থিতিশীল"         │
│  → Visitor এড়িয়ে চলুন                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

> **"Visitor হলো object-oriented programming এ pattern matching এর সমতুল্য। যখন আপনার কাছে স্থিতিশীল type hierarchy আছে এবং সেই type গুলোর উপর বিভিন্ন operation চালাতে চান — Visitor আপনার বন্ধু।"**
