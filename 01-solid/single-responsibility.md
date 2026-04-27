# S — Single Responsibility Principle (SRP)

> **"একটি ক্লাসের পরিবর্তনের একটিমাত্র কারণ থাকা উচিত।"**
> — Robert C. Martin (Uncle Bob)

---

## 📖 সংজ্ঞা

Single Responsibility Principle বলে যে, **একটি ক্লাস বা মডিউলের শুধুমাত্র একটি দায়িত্ব থাকবে** এবং সেই দায়িত্ব সম্পূর্ণভাবে ঐ ক্লাসের মধ্যে encapsulate থাকবে।

অন্যভাবে বললে — একটি ক্লাস পরিবর্তন করার শুধুমাত্র **একটি কারণ** থাকা উচিত। যদি দুইটি ভিন্ন কারণে একটি ক্লাস পরিবর্তন করতে হয়, তাহলে সেটি SRP ভঙ্গ করছে।

### "দায়িত্ব" বলতে কী বোঝায়?

দায়িত্ব = **পরিবর্তনের কারণ (reason to change)**

যদি একটি ক্লাসে দুটি ফাংশনালিটি থাকে যেগুলো ভিন্ন কারণে পরিবর্তন হতে পারে, তাহলে সেগুলো আলাদা ক্লাসে রাখা উচিত।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### রেস্তোরাঁর উদাহরণ

একটি রেস্তোরাঁয় চিন্তা করুন:
- **শেফ** — শুধু রান্না করে
- **ওয়েটার** — শুধু অর্ডার নেয় ও খাবার সার্ভ করে
- **ক্যাশিয়ার** — শুধু বিল তৈরি ও টাকা নেয়

যদি একজনকে তিনটাই করতে হয়, তাহলে:
- রান্নার রেসিপি বদলালে ক্যাশিয়ারের কাজও ক্ষতিগ্রস্ত হতে পারে
- বিলিং সিস্টেম বদলালে রান্নাও প্রভাবিত হতে পারে

এটাই SRP ভঙ্গ!

---

## ❌ খারাপ উদাহরণ — SRP ভঙ্গ

### PHP (❌ ভুল)

```php
<?php

class Employee
{
    private string $name;
    private float $salary;

    public function __construct(string $name, float $salary)
    {
        $this->name = $name;
        $this->salary = $salary;
    }

    // দায়িত্ব ১: কর্মচারীর ডাটা ম্যানেজমেন্ট
    public function getName(): string
    {
        return $this->name;
    }

    public function getSalary(): float
    {
        return $this->salary;
    }

    // দায়িত্ব ২: বেতন হিসাব (ব্যবসায়িক লজিক)
    public function calculateTax(): float
    {
        if ($this->salary > 50000) {
            return $this->salary * 0.3;
        }
        return $this->salary * 0.2;
    }

    // দায়িত্ব ৩: ডাটাবেসে সেভ করা (পারসিস্টেন্স)
    public function saveToDatabase(): void
    {
        $db = new PDO('mysql:host=localhost;dbname=company', 'root', '');
        $stmt = $db->prepare("INSERT INTO employees (name, salary) VALUES (?, ?)");
        $stmt->execute([$this->name, $this->salary]);
    }

    // দায়িত্ব ৪: রিপোর্ট তৈরি (প্রেজেন্টেশন)
    public function generateReport(): string
    {
        return sprintf(
            "নাম: %s\nবেতন: %.2f\nট্যাক্স: %.2f",
            $this->name,
            $this->salary,
            $this->calculateTax()
        );
    }

    // দায়িত্ব ৫: ইমেইল পাঠানো (নোটিফিকেশন)
    public function sendPayslipEmail(): void
    {
        mail(
            "{$this->name}@company.com",
            "বেতন স্লিপ",
            $this->generateReport()
        );
    }
}

// ব্যবহার
$emp = new Employee("রহিম", 60000);
$emp->saveToDatabase();       // ডাটাবেস সংক্রান্ত
$emp->calculateTax();         // ট্যাক্স লজিক
$emp->generateReport();       // রিপোর্ট জেনারেশন
$emp->sendPayslipEmail();     // ইমেইল পাঠানো
```

**সমস্যা:** এই ক্লাসটি পরিবর্তন করতে হবে যদি:
1. কর্মচারীর ডাটা স্ট্রাকচার বদলায়
2. ট্যাক্স নিয়ম বদলায়
3. ডাটাবেস বদলায় (MySQL → PostgreSQL)
4. রিপোর্ট ফরম্যাট বদলায়
5. ইমেইল সিস্টেম বদলায়

**৫টি কারণে ক্লাস পরিবর্তন = ৫টি দায়িত্ব = SRP ভঙ্গ!**

### JavaScript (❌ ভুল)

```javascript
class Employee {
    constructor(name, salary) {
        this.name = name;
        this.salary = salary;
    }

    // দায়িত্ব ১: ট্যাক্স ক্যালকুলেশন
    calculateTax() {
        if (this.salary > 50000) {
            return this.salary * 0.3;
        }
        return this.salary * 0.2;
    }

    // দায়িত্ব ২: ডাটাবেসে সেভ
    async saveToDatabase() {
        const db = await connectToMongoDB();
        await db.collection('employees').insertOne({
            name: this.name,
            salary: this.salary
        });
    }

    // দায়িত্ব ৩: রিপোর্ট তৈরি
    generateReport() {
        return `নাম: ${this.name}\nবেতন: ${this.salary}\nট্যাক্স: ${this.calculateTax()}`;
    }

    // দায়িত্ব ৪: ইমেইল পাঠানো
    async sendPayslipEmail() {
        await sendEmail({
            to: `${this.name}@company.com`,
            subject: "বেতন স্লিপ",
            body: this.generateReport()
        });
    }
}
```

---

## ✅ ভালো উদাহরণ — SRP মেনে

### PHP (✅ সঠিক)

```php
<?php

// ক্লাস ১: শুধুমাত্র কর্মচারীর ডাটা ধারণ করে
class Employee
{
    public function __construct(
        private readonly string $name,
        private readonly float $salary,
        private readonly string $email
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getSalary(): float
    {
        return $this->salary;
    }

    public function getEmail(): string
    {
        return $this->email;
    }
}

// ক্লাস ২: শুধুমাত্র ট্যাক্স হিসাবের দায়িত্ব
class TaxCalculator
{
    private const HIGH_TAX_THRESHOLD = 50000;
    private const HIGH_TAX_RATE = 0.3;
    private const LOW_TAX_RATE = 0.2;

    public function calculate(Employee $employee): float
    {
        if ($employee->getSalary() > self::HIGH_TAX_THRESHOLD) {
            return $employee->getSalary() * self::HIGH_TAX_RATE;
        }
        return $employee->getSalary() * self::LOW_TAX_RATE;
    }
}

// ক্লাস ৩: শুধুমাত্র ডাটাবেস অপারেশনের দায়িত্ব
class EmployeeRepository
{
    public function __construct(
        private readonly PDO $connection
    ) {}

    public function save(Employee $employee): void
    {
        $stmt = $this->connection->prepare(
            "INSERT INTO employees (name, salary, email) VALUES (?, ?, ?)"
        );
        $stmt->execute([
            $employee->getName(),
            $employee->getSalary(),
            $employee->getEmail()
        ]);
    }

    public function findByName(string $name): ?Employee
    {
        $stmt = $this->connection->prepare(
            "SELECT * FROM employees WHERE name = ?"
        );
        $stmt->execute([$name]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$row) return null;

        return new Employee($row['name'], $row['salary'], $row['email']);
    }
}

// ক্লাস ৪: শুধুমাত্র রিপোর্ট তৈরির দায়িত্ব
class PayslipReportGenerator
{
    public function __construct(
        private readonly TaxCalculator $taxCalculator
    ) {}

    public function generate(Employee $employee): string
    {
        $tax = $this->taxCalculator->calculate($employee);
        $netSalary = $employee->getSalary() - $tax;

        return sprintf(
            "=== বেতন স্লিপ ===\nনাম: %s\nমোট বেতন: %.2f\nট্যাক্স: %.2f\nনিট বেতন: %.2f",
            $employee->getName(),
            $employee->getSalary(),
            $tax,
            $netSalary
        );
    }
}

// ক্লাস ৫: শুধুমাত্র নোটিফিকেশন পাঠানোর দায়িত্ব
class EmailNotifier
{
    public function sendPayslip(Employee $employee, string $reportContent): void
    {
        mail(
            $employee->getEmail(),
            "বেতন স্লিপ - {$employee->getName()}",
            $reportContent
        );
    }
}

// ✅ ব্যবহার — প্রতিটি ক্লাস তার নিজের কাজ করে
$employee = new Employee("রহিম", 60000, "rahim@company.com");

$taxCalculator = new TaxCalculator();
$repository = new EmployeeRepository($pdo);
$reportGenerator = new PayslipReportGenerator($taxCalculator);
$emailNotifier = new EmailNotifier();

$repository->save($employee);
$report = $reportGenerator->generate($employee);
$emailNotifier->sendPayslip($employee, $report);
```

### JavaScript (✅ সঠিক)

```javascript
// ক্লাস ১: শুধুমাত্র কর্মচারীর ডাটা
class Employee {
    #name;
    #salary;
    #email;

    constructor(name, salary, email) {
        this.#name = name;
        this.#salary = salary;
        this.#email = email;
    }

    get name() { return this.#name; }
    get salary() { return this.#salary; }
    get email() { return this.#email; }
}

// ক্লাস ২: শুধুমাত্র ট্যাক্স হিসাব
class TaxCalculator {
    static HIGH_TAX_THRESHOLD = 50000;
    static HIGH_TAX_RATE = 0.3;
    static LOW_TAX_RATE = 0.2;

    calculate(employee) {
        if (employee.salary > TaxCalculator.HIGH_TAX_THRESHOLD) {
            return employee.salary * TaxCalculator.HIGH_TAX_RATE;
        }
        return employee.salary * TaxCalculator.LOW_TAX_RATE;
    }
}

// ক্লাস ৩: শুধুমাত্র ডাটাবেস অপারেশন
class EmployeeRepository {
    #db;

    constructor(database) {
        this.#db = database;
    }

    async save(employee) {
        await this.#db.collection('employees').insertOne({
            name: employee.name,
            salary: employee.salary,
            email: employee.email
        });
    }

    async findByName(name) {
        const doc = await this.#db.collection('employees').findOne({ name });
        if (!doc) return null;
        return new Employee(doc.name, doc.salary, doc.email);
    }
}

// ক্লাস ৪: শুধুমাত্র রিপোর্ট তৈরি
class PayslipReportGenerator {
    #taxCalculator;

    constructor(taxCalculator) {
        this.#taxCalculator = taxCalculator;
    }

    generate(employee) {
        const tax = this.#taxCalculator.calculate(employee);
        const netSalary = employee.salary - tax;

        return [
            '=== বেতন স্লিপ ===',
            `নাম: ${employee.name}`,
            `মোট বেতন: ${employee.salary.toFixed(2)}`,
            `ট্যাক্স: ${tax.toFixed(2)}`,
            `নিট বেতন: ${netSalary.toFixed(2)}`
        ].join('\n');
    }
}

// ক্লাস ৫: শুধুমাত্র ইমেইল পাঠানো
class EmailNotifier {
    #mailer;

    constructor(mailer) {
        this.#mailer = mailer;
    }

    async sendPayslip(employee, reportContent) {
        await this.#mailer.send({
            to: employee.email,
            subject: `বেতন স্লিপ - ${employee.name}`,
            body: reportContent
        });
    }
}

// ✅ ব্যবহার
const employee = new Employee("রহিম", 60000, "rahim@company.com");

const taxCalculator = new TaxCalculator();
const repository = new EmployeeRepository(db);
const reportGenerator = new PayslipReportGenerator(taxCalculator);
const emailNotifier = new EmailNotifier(mailer);

await repository.save(employee);
const report = reportGenerator.generate(employee);
await emailNotifier.sendPayslip(employee, report);
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### কোহিশন (Cohesion) ও কাপলিং (Coupling)

SRP সরাসরি **High Cohesion** এবং **Low Coupling** এর সাথে সম্পর্কিত:

```
┌─────────────────────────────────────────────┐
│          SRP মেনে চললে                       │
│                                             │
│  High Cohesion ──► একটি ক্লাসের সব মেথড     │
│                    একই উদ্দেশ্যে কাজ করে     │
│                                             │
│  Low Coupling ───► ক্লাসগুলো একে অপরের      │
│                    উপর কম নির্ভরশীল          │
└─────────────────────────────────────────────┘
```

### SRP কোথায় কোথায় প্রযোজ্য?

SRP শুধু ক্লাসের জন্য না — এটি সকল স্তরে প্রযোজ্য:

| স্তর | উদাহরণ |
|------|--------|
| **ফাংশন** | একটি ফাংশন একটিই কাজ করবে |
| **ক্লাস** | একটি ক্লাসের একটিই দায়িত্ব |
| **মডিউল** | একটি মডিউল একটিই ফিচার হ্যান্ডেল করবে |
| **সার্ভিস** | মাইক্রোসার্ভিসে একটি সার্ভিস একটি ডোমেইন |

### ফাংশন লেভেলে SRP

```php
<?php

// ❌ একটি ফাংশন অনেক কিছু করছে
function processOrder(array $orderData): void
{
    // ভ্যালিডেশন
    if (empty($orderData['items'])) {
        throw new InvalidArgumentException("আইটেম নেই");
    }
    if ($orderData['total'] <= 0) {
        throw new InvalidArgumentException("টোটাল ভুল");
    }

    // ডিসকাউন্ট হিসাব
    $discount = 0;
    if ($orderData['total'] > 1000) {
        $discount = $orderData['total'] * 0.1;
    }

    // ডাটাবেসে সেভ
    $db = new PDO('mysql:host=localhost;dbname=shop', 'root', '');
    $stmt = $db->prepare("INSERT INTO orders ...");
    $stmt->execute([...]);

    // ইমেইল পাঠানো
    mail($orderData['email'], "অর্ডার কনফার্ম", "...");
}

// ✅ প্রতিটি ফাংশন একটি কাজ করে
function validateOrder(array $orderData): void
{
    if (empty($orderData['items'])) {
        throw new InvalidArgumentException("আইটেম নেই");
    }
    if ($orderData['total'] <= 0) {
        throw new InvalidArgumentException("টোটাল ভুল");
    }
}

function calculateDiscount(float $total): float
{
    if ($total > 1000) {
        return $total * 0.1;
    }
    return 0;
}
```

```javascript
// ❌ একটি ফাংশন অনেক কিছু করছে
async function processOrder(orderData) {
    // ভ্যালিডেশন + ডিসকাউন্ট + সেভ + ইমেইল সব একসাথে
    if (!orderData.items?.length) throw new Error("আইটেম নেই");
    const discount = orderData.total > 1000 ? orderData.total * 0.1 : 0;
    await db.collection('orders').insertOne({ ...orderData, discount });
    await sendEmail(orderData.email, "অর্ডার কনফার্ম");
}

// ✅ প্রতিটি ফাংশন একটি কাজ করে
function validateOrder(orderData) {
    if (!orderData.items?.length) throw new Error("আইটেম নেই");
    if (orderData.total <= 0) throw new Error("টোটাল ভুল");
}

function calculateDiscount(total) {
    return total > 1000 ? total * 0.1 : 0;
}
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **সহজ মেইনটেন্যান্স** | ছোট, ফোকাসড ক্লাস বুঝতে ও বদলাতে সহজ |
| ২ | **কম বাগ** | একটি পরিবর্তন অন্য ফিচারকে ভাঙে না |
| ৩ | **সহজ টেস্টিং** | প্রতিটি ক্লাস আলাদাভাবে ইউনিট টেস্ট করা যায় |
| ৪ | **পুনর্ব্যবহারযোগ্যতা** | `TaxCalculator` অন্য জায়গায়ও ব্যবহার করা যায় |
| ৫ | **টিম কোলাবোরেশন** | ভিন্ন ডেভেলপার ভিন্ন ক্লাসে কাজ করতে পারে, কনফ্লিক্ট কম |
| ৬ | **ফ্লেক্সিবিলিটি** | ডাটাবেস MySQL থেকে PostgreSQL এ বদলালে শুধু `Repository` ক্লাস বদলাতে হবে |
| ৭ | **পড়তে সহজ** | ক্লাসের নাম দেখেই বোঝা যায় এটা কী করে |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **ক্লাস সংখ্যা বাড়ে** | একটি বড় ক্লাসের বদলে ৫-৬টি ছোট ক্লাস তৈরি হয় |
| ২ | **জটিলতা বাড়তে পারে** | ছোট প্রজেক্টে অতিরিক্ত ক্লাস বিভ্রান্তিকর হতে পারে |
| ৩ | **নেভিগেশন কঠিন** | কোড অনেক ফাইলে ছড়িয়ে যায়, IDE ছাড়া ট্র্যাক করা কঠিন |
| ৪ | **অতি-বিভাজনের ঝুঁকি** | খুব বেশি ভাগ করলে "ক্লাস এক্সপ্লোশন" হতে পারে |
| ৫ | **ইনিশিয়াল ডেভেলপমেন্ট ধীর** | শুরুতে বেশি সময় লাগে সঠিক দায়িত্ব ভাগ করতে |

---

## 🚫 সাধারণ ভুল (Common Mistakes)

### ভুল ১: অতি-বিভাজন (Over-splitting)

```php
<?php

// ❌ অতি-বিভাজন — এটা SRP না, এটা পাগলামি!
class EmployeeNameGetter
{
    public function getName(Employee $emp): string
    {
        return $emp->name;
    }
}

class EmployeeSalaryGetter
{
    public function getSalary(Employee $emp): float
    {
        return $emp->salary;
    }
}

// ✅ সঠিক — ডাটা হোল্ড করা একটিই দায়িত্ব
class Employee
{
    public function __construct(
        public readonly string $name,
        public readonly float $salary
    ) {}
}
```

### ভুল ২: "দায়িত্ব" ভুলভাবে চিহ্নিত করা

```php
<?php

// ❌ ভুল ধারণা: প্রতিটি মেথড = একটি দায়িত্ব
// getName() আর getSalary() ভিন্ন দায়িত্ব নয়!

// ✅ সঠিক ধারণা: দায়িত্ব = পরিবর্তনের কারণ
// getName() আর getSalary() একই কারণে পরিবর্তন হবে (ডাটা মডেল বদলালে)
// তাই এগুলো একই দায়িত্বের অংশ
```

### ভুল ৩: God Class তৈরি করা

```javascript
// ❌ "UserManager" — সাধারণত God Class হয়ে যায়
class UserManager {
    createUser() { /* ... */ }
    deleteUser() { /* ... */ }
    sendWelcomeEmail() { /* ... */ }
    generateReport() { /* ... */ }
    validatePassword() { /* ... */ }
    uploadAvatar() { /* ... */ }
    calculateSubscription() { /* ... */ }
}

// ✅ ভেঙে ফেলুন
class UserService { /* create, delete, update */ }
class UserNotifier { /* emails, notifications */ }
class UserReportGenerator { /* reports */ }
class PasswordValidator { /* validation */ }
class AvatarUploader { /* file upload */ }
class SubscriptionCalculator { /* billing */ }
```

---

## 🧪 টেস্টিং সুবিধা

SRP মানলে ইউনিট টেস্ট অনেক সহজ হয়:

### PHP (PHPUnit)

```php
<?php

class TaxCalculatorTest extends TestCase
{
    private TaxCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new TaxCalculator();
    }

    public function test_high_salary_gets_30_percent_tax(): void
    {
        $employee = new Employee("রহিম", 60000, "rahim@test.com");
        $tax = $this->calculator->calculate($employee);

        $this->assertEquals(18000, $tax); // 60000 * 0.3
    }

    public function test_low_salary_gets_20_percent_tax(): void
    {
        $employee = new Employee("করিম", 40000, "karim@test.com");
        $tax = $this->calculator->calculate($employee);

        $this->assertEquals(8000, $tax); // 40000 * 0.2
    }

    // ✅ ডাটাবেস, ইমেইল কিছুই মক করতে হয়নি!
    // কারণ TaxCalculator শুধু ট্যাক্স হিসাব করে, আর কিছু না
}
```

### JavaScript (Jest)

```javascript
describe('TaxCalculator', () => {
    const calculator = new TaxCalculator();

    test('উচ্চ বেতনে ৩০% ট্যাক্স', () => {
        const employee = new Employee("রহিম", 60000, "rahim@test.com");
        expect(calculator.calculate(employee)).toBe(18000);
    });

    test('কম বেতনে ২০% ট্যাক্স', () => {
        const employee = new Employee("করিম", 40000, "karim@test.com");
        expect(calculator.calculate(employee)).toBe(8000);
    });

    // ✅ কোনো মক নেই, কোনো ডাটাবেস সেটআপ নেই
    // পিউর ইউনিট টেস্ট — দ্রুত ও নির্ভরযোগ্য
});
```

---

## 📏 কখন SRP প্রয়োগ করবেন

### ✅ করবেন যখন:

- ক্লাসটি **২০০+ লাইনের** বেশি হয়ে যাচ্ছে
- ক্লাসে **ভিন্ন ধরনের dependency** inject হচ্ছে (DB + Mailer + Logger)
- ক্লাস পরিবর্তনের **একাধিক কারণ** আছে
- **টেস্ট লিখতে** অনেক মক দরকার হচ্ছে
- ক্লাসের নাম `Manager`, `Processor`, `Handler` দিয়ে শেষ হচ্ছে (God Class এর লক্ষণ)

### ❌ করবেন না যখন:

- প্রজেক্ট **খুব ছোট** (একটি স্ক্রিপ্ট বা প্রোটোটাইপ)
- ক্লাসের সব মেথড **একই ডাটার** উপর একই কারণে কাজ করে
- বিভাজন করলে **কোড আরো জটিল** হয়ে যায়
- **YAGNI** (You Aren't Gonna Need It) — ভবিষ্যতে হয়তো লাগবে এই চিন্তায় আগেভাগে ভাগ করবেন না

---

## 🔗 অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক

```
SRP ──────────► OCP
(আলাদা দায়িত্ব)  (প্রতিটি আলাদাভাবে এক্সটেন্ড করা যায়)
  │
  ├──────────► ISP
  │            (ছোট ইন্টারফেস = ছোট দায়িত্ব)
  │
  └──────────► DIP
               (dependency inject করে ক্লাস আলাদা রাখা)
```

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **মূল ধারণা** | একটি ক্লাস, একটি দায়িত্ব, একটি পরিবর্তনের কারণ |
| **লক্ষ্য** | High Cohesion, Low Coupling |
| **চেক করুন** | "এই ক্লাস কোন কোন কারণে বদলাতে পারে?" |
| **সাবধান** | অতি-বিভাজন এড়িয়ে চলুন |
| **মনে রাখুন** | দায়িত্ব = পরিবর্তনের কারণ, মেথড সংখ্যা নয় |
