# 🛡️ Proxy প্যাটার্ন (Proxy Design Pattern)

## 📌 সংজ্ঞা ও পরিচিতি

> **GoF Definition:** *"Provide a surrogate or placeholder for another object to control access to it."*

Proxy প্যাটার্ন হলো একটি **Structural Design Pattern** যেখানে একটি অবজেক্টের পরিবর্তে একটি **প্রতিনিধি (surrogate)** বা **স্থানধারক (placeholder)** অবজেক্ট ব্যবহার করা হয়। এই প্রতিনিধি অবজেক্টটি মূল অবজেক্টের প্রতি **অ্যাক্সেস নিয়ন্ত্রণ** করে — অর্থাৎ ক্লায়েন্ট সরাসরি মূল অবজেক্টের সাথে কথা বলে না, বরং Proxy-র মাধ্যমে যোগাযোগ করে।

**মূল ধারণা:** ক্লায়েন্ট যখন কোনো অবজেক্ট ব্যবহার করতে চায়, তখন সে আসলে Proxy-র সাথে কাজ করে। Proxy সিদ্ধান্ত নেয় — কখন, কিভাবে, এবং আদৌ মূল অবজেক্টকে কল করবে কি না।

**কেন দরকার?**

- মূল অবজেক্ট তৈরি করা **ব্যয়বহুল** হতে পারে (Virtual Proxy)
- মূল অবজেক্টে **সবার অ্যাক্সেস** থাকা উচিত নয় (Protection Proxy)
- মূল অবজেক্ট **দূরবর্তী সার্ভারে** থাকতে পারে (Remote Proxy)
- অতিরিক্ত **লজিক যোগ** করতে হতে পারে — logging, caching, rate limiting (Smart Proxy)

---

## 🏠 বাস্তব উদাহরণ

### অফিস বিল্ডিং-এর সিকিউরিটি গার্ড

কল্পনা করুন, ঢাকার গুলশানে একটি করপোরেট অফিস বিল্ডিং আছে। বিল্ডিং-এর গেটে একজন **সিকিউরিটি গার্ড** আছেন।

```
👤 ভিজিটর ──→ 🛡️ সিকিউরিটি গার্ড (PROXY) ──→ 🏢 অফিস বিল্ডিং (Real Subject)
```

**সিকিউরিটি গার্ড যা করেন (Proxy-র কাজ):**

1. **Access Control (Protection Proxy):** আইডি কার্ড চেক করেন — অনুমোদিত ব্যক্তি ছাড়া কেউ ঢুকতে পারে না
2. **Logging (Logging Proxy):** কে কখন ঢুকলো/বের হলো — রেজিস্টারে লেখেন
3. **Rate Limiting:** একবারে ১০ জনের বেশি ভিজিটর ঢুকতে দেন না
4. **Lazy Access (Virtual Proxy):** VIP গেস্ট এলে তবেই লিফট চালু করেন

গার্ড নিজে বিল্ডিং নয়, কিন্তু বিল্ডিং-এ প্রবেশ করতে গেলে গার্ডের মধ্য দিয়েই যেতে হবে — এটাই Proxy প্যাটার্নের সারকথা।

### 🇧🇩 বাংলাদেশ কনটেক্সট

- **bKash API Proxy:** আপনার অ্যাপ সরাসরি bKash সার্ভারে কল করে না — একটি proxy layer মাঝখানে থাকে যা authentication, rate limiting, logging সব সামলায়
- **BTRC Content Proxy:** কিছু কনটেন্ট restricted — ISP-র proxy layer এই অ্যাক্সেস কন্ট্রোল করে
- **মোবাইল ব্যাংকিং:** Nagad/bKash-এ ট্রানজেকশন করার সময় middleware proxy আপনার PIN verify করে, balance check করে, তারপর actual transaction হয়

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────┐
│                    Proxy Pattern UML                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐         ┌──────────────────┐                 │
│  │  Client   │────────▶│   «interface»    │                 │
│  └──────────┘         │    Subject        │                 │
│                       ├──────────────────┤                 │
│                       │ + request()       │                 │
│                       │ + operation()     │                 │
│                       └────────┬─────────┘                 │
│                                │                            │
│                    ┌───────────┴───────────┐                │
│                    │                       │                │
│           ┌────────┴────────┐    ┌────────┴────────┐       │
│           │   RealSubject   │    │      Proxy      │       │
│           ├─────────────────┤    ├─────────────────┤       │
│           │ + request()     │◀───│ - realSubject   │       │
│           │ + operation()   │    │ + request()     │       │
│           └─────────────────┘    │ + operation()   │       │
│                                  └─────────────────┘       │
│                                                             │
│  ───────────────────────────────────────────────           │
│  Proxy.request() {                                         │
│      // pre-processing (auth, cache check, log)            │
│      this.realSubject.request();                           │
│      // post-processing (cache store, log)                 │
│  }                                                         │
└─────────────────────────────────────────────────────────────┘

Sequence Diagram:
┌────────┐     ┌───────┐     ┌─────────────┐
│ Client │     │ Proxy │     │ RealSubject │
└───┬────┘     └───┬───┘     └──────┬──────┘
    │              │                 │
    │  request()   │                 │
    │─────────────▶│                 │
    │              │  check access   │
    │              │────────┐        │
    │              │        │        │
    │              │◀───────┘        │
    │              │                 │
    │              │   request()     │
    │              │────────────────▶│
    │              │                 │
    │              │    response     │
    │              │◀────────────────│
    │   response   │                 │
    │◀─────────────│                 │
    │              │                 │
```

---

## 💻 Proxy Types ও ইমপ্লিমেন্টেশন

### ১. Virtual Proxy (Lazy Loading)

Virtual Proxy তখন ব্যবহার হয় যখন মূল অবজেক্ট তৈরি করা **ব্যয়বহুল** — মেমোরি, সময়, বা রিসোর্সের দিক থেকে। Proxy শুধুমাত্র তখনই মূল অবজেক্ট তৈরি করে যখন সত্যিই দরকার হয়।

#### PHP 8.3 — Virtual Proxy

```php
<?php

declare(strict_types=1);

// Subject Interface
interface Image
{
    public function display(): string;
    public function getResolution(): string;
    public function getFileSize(): int;
}

// Real Subject — ভারী অবজেক্ট (ডিস্ক থেকে ইমেজ লোড করে)
class HighResolutionImage implements Image
{
    private string $imageData;

    public function __construct(
        private readonly string $filename,
        private readonly int $width = 3840,
        private readonly int $height = 2160
    ) {
        // ব্যয়বহুল অপারেশন — ডিস্ক থেকে ইমেজ লোড
        $this->loadFromDisk();
    }

    private function loadFromDisk(): void
    {
        // সিমুলেশন: ডিস্ক থেকে বড় ইমেজ লোড হচ্ছে
        echo "⏳ [{$this->filename}] ডিস্ক থেকে লোড হচ্ছে... (ব্যয়বহুল অপারেশন)\n";
        usleep(500_000); // 500ms সিমুলেশন
        $this->imageData = str_repeat('x', 1024 * 1024); // 1MB ডেটা সিমুলেশন
    }

    public function display(): string
    {
        return "🖼️ [{$this->filename}] প্রদর্শিত হচ্ছে ({$this->width}x{$this->height})";
    }

    public function getResolution(): string
    {
        return "{$this->width}x{$this->height}";
    }

    public function getFileSize(): int
    {
        return strlen($this->imageData);
    }
}

// Virtual Proxy — শুধু দরকার হলেই ইমেজ লোড করবে
class ImageProxy implements Image
{
    private ?HighResolutionImage $realImage = null;

    public function __construct(
        private readonly string $filename,
        private readonly int $width = 3840,
        private readonly int $height = 2160
    ) {
        // কনস্ট্রাক্টরে কিছুই লোড হয় না — এটাই lazy loading-এর সুবিধা
        echo "✅ [{$this->filename}] Proxy তৈরি হলো (ইমেজ এখনো লোড হয়নি)\n";
    }

    private function getRealImage(): HighResolutionImage
    {
        if ($this->realImage === null) {
            echo "🔄 [{$this->filename}] এখন সত্যিই দরকার — Real Image তৈরি হচ্ছে\n";
            $this->realImage = new HighResolutionImage(
                $this->filename,
                $this->width,
                $this->height
            );
        }
        return $this->realImage;
    }

    public function display(): string
    {
        return $this->getRealImage()->display();
    }

    public function getResolution(): string
    {
        // রেজোলিউশন জানতে পুরো ইমেজ লোড করা লাগে না
        return "{$this->width}x{$this->height}";
    }

    public function getFileSize(): int
    {
        return $this->getRealImage()->getFileSize();
    }
}

// ব্যবহার
echo "=== ১০টি ইমেজের গ্যালারি পেজ ===\n\n";

$gallery = [];
for ($i = 1; $i <= 10; $i++) {
    $gallery[] = new ImageProxy("photo_{$i}.jpg");
}
// এখন পর্যন্ত কোনো ইমেজ লোড হয়নি!

echo "\n--- শুধু প্রথম ইমেজ দেখা হচ্ছে ---\n";
echo $gallery[0]->display() . "\n"; // শুধু এটাই লোড হবে

echo "\n--- রেজোলিউশন চেক (লোড ছাড়াই) ---\n";
echo $gallery[5]->getResolution() . "\n"; // ইমেজ লোড হবে না
```

#### JavaScript ES2022+ — Virtual Proxy

```javascript
// Subject Interface (TS-style documentation)
// interface Image { display(): string; getMetadata(): object; }

class HeavyDatabaseReport {
    #data = null;
    #reportName;
    #loadTimeMs;

    constructor(reportName) {
        this.#reportName = reportName;
        console.log(`⏳ [${reportName}] ডেটাবেস থেকে রিপোর্ট লোড হচ্ছে...`);

        // সিমুলেশন: ভারী ডেটাবেস কোয়েরি
        const start = performance.now();
        this.#data = this.#loadFromDatabase();
        this.#loadTimeMs = performance.now() - start;
    }

    #loadFromDatabase() {
        // সিমুলেট: বড় ডেটাসেট
        return Array.from({ length: 100_000 }, (_, i) => ({
            id: i,
            amount: Math.random() * 10000,
            date: new Date(2024, Math.floor(Math.random() * 12), 1),
        }));
    }

    getReport() {
        const total = this.#data.reduce((sum, row) => sum + row.amount, 0);
        return {
            name: this.#reportName,
            rows: this.#data.length,
            totalAmount: total.toFixed(2),
            loadTime: `${this.#loadTimeMs.toFixed(0)}ms`,
        };
    }

    getSummary() {
        return `📊 ${this.#reportName}: ${this.#data.length} rows`;
    }
}

// Virtual Proxy — ES2022+ Private Fields ব্যবহার করে
class ReportProxy {
    #realReport = null;
    #reportName;

    constructor(reportName) {
        this.#reportName = reportName;
        console.log(`✅ [${reportName}] Proxy তৈরি — ডেটা এখনো লোড হয়নি`);
    }

    #ensureLoaded() {
        if (this.#realReport === null) {
            console.log(`🔄 [${this.#reportName}] এখন লোড হচ্ছে...`);
            this.#realReport = new HeavyDatabaseReport(this.#reportName);
        }
        return this.#realReport;
    }

    getReport() {
        return this.#ensureLoaded().getReport();
    }

    getSummary() {
        return this.#ensureLoaded().getSummary();
    }

    // মেটাডেটা — লোড ছাড়াই পাওয়া যায়
    get name() {
        return this.#reportName;
    }

    get isLoaded() {
        return this.#realReport !== null;
    }
}

// ES6 Proxy দিয়ে আরও elegant সমাধান
function createLazyProxy(Factory, ...args) {
    let instance = null;

    return new Proxy({}, {
        get(target, prop, receiver) {
            if (prop === Symbol.toPrimitive || prop === 'inspect') {
                return () => `[LazyProxy: ${Factory.name}]`;
            }

            instance ??= new Factory(...args);
            const value = instance[prop];
            return typeof value === 'function' ? value.bind(instance) : value;
        },
    });
}

// ব্যবহার
const reports = [
    new ReportProxy('বার্ষিক বিক্রয় রিপোর্ট'),
    new ReportProxy('মাসিক খরচ রিপোর্ট'),
    new ReportProxy('কর্মচারী পারফরম্যান্স'),
];

console.log('\n--- শুধু নাম দেখা (কিছু লোড হবে না) ---');
reports.forEach(r => console.log(`  📄 ${r.name} | Loaded: ${r.isLoaded}`));

console.log('\n--- প্রথম রিপোর্ট দেখা হচ্ছে ---');
console.log(reports[0].getReport());
```

---

### ২. Protection Proxy (Access Control)

Protection Proxy নির্ধারণ করে কোন ক্লায়েন্টের কোন অপারেশনে **অ্যাক্সেস** আছে। Role-based access control (RBAC) এখানে প্রাসঙ্গিক।

#### PHP 8.3 — Protection Proxy

```php
<?php

declare(strict_types=1);

// Enum দিয়ে Role ও Permission ম্যানেজমেন্ট
enum Role: string
{
    case Admin = 'admin';
    case Manager = 'manager';
    case Employee = 'employee';
    case Guest = 'guest';
}

enum Permission: string
{
    case Read = 'read';
    case Write = 'write';
    case Delete = 'delete';
    case ManageUsers = 'manage_users';
    case ViewSalary = 'view_salary';
}

// ইউজার ভ্যালু অবজেক্ট
readonly class User
{
    public function __construct(
        public string $id,
        public string $name,
        public Role $role,
        public array $permissions = [],
    ) {}

    public function hasPermission(Permission $permission): bool
    {
        return in_array($permission, $this->permissions, true);
    }
}

// Subject Interface
interface EmployeeDatabase
{
    public function getEmployee(string $id): array;
    public function getSalary(string $id): float;
    public function updateSalary(string $id, float $amount): bool;
    public function deleteEmployee(string $id): bool;
}

// Real Subject
class RealEmployeeDatabase implements EmployeeDatabase
{
    private array $employees = [
        'E001' => ['name' => 'রহিম', 'department' => 'Engineering', 'salary' => 85000],
        'E002' => ['name' => 'করিম', 'department' => 'Marketing', 'salary' => 65000],
        'E003' => ['name' => 'সালমা', 'department' => 'HR', 'salary' => 75000],
    ];

    public function getEmployee(string $id): array
    {
        return $this->employees[$id] ?? throw new \RuntimeException("কর্মচারী পাওয়া যায়নি: {$id}");
    }

    public function getSalary(string $id): float
    {
        $emp = $this->getEmployee($id);
        return $emp['salary'];
    }

    public function updateSalary(string $id, float $amount): bool
    {
        if (!isset($this->employees[$id])) return false;
        $this->employees[$id]['salary'] = $amount;
        return true;
    }

    public function deleteEmployee(string $id): bool
    {
        if (!isset($this->employees[$id])) return false;
        unset($this->employees[$id]);
        return true;
    }
}

// Protection Proxy — Access Control
class ProtectedEmployeeDatabase implements EmployeeDatabase
{
    private EmployeeDatabase $database;

    public function __construct(
        private readonly User $currentUser,
        ?EmployeeDatabase $database = null,
    ) {
        $this->database = $database ?? new RealEmployeeDatabase();
    }

    private function checkPermission(Permission $required, string $action): void
    {
        if (!$this->currentUser->hasPermission($required)) {
            throw new \RuntimeException(
                "⛔ অ্যাক্সেস অস্বীকৃত: '{$this->currentUser->name}' ({$this->currentUser->role->value}) "
                . "এর '{$action}' অপারেশনে অনুমতি নেই। "
                . "প্রয়োজনীয় অনুমতি: {$required->value}"
            );
        }
    }

    public function getEmployee(string $id): array
    {
        $this->checkPermission(Permission::Read, 'কর্মচারী তথ্য দেখা');
        echo "✅ [{$this->currentUser->name}] কর্মচারী তথ্য দেখছেন\n";
        return $this->database->getEmployee($id);
    }

    public function getSalary(string $id): float
    {
        $this->checkPermission(Permission::ViewSalary, 'বেতন দেখা');
        echo "✅ [{$this->currentUser->name}] বেতন তথ্য দেখছেন\n";
        return $this->database->getSalary($id);
    }

    public function updateSalary(string $id, float $amount): bool
    {
        $this->checkPermission(Permission::Write, 'বেতন আপডেট');
        echo "✅ [{$this->currentUser->name}] বেতন আপডেট করছেন\n";
        return $this->database->updateSalary($id, $amount);
    }

    public function deleteEmployee(string $id): bool
    {
        $this->checkPermission(Permission::Delete, 'কর্মচারী ডিলিট');
        echo "✅ [{$this->currentUser->name}] কর্মচারী ডিলিট করছেন\n";
        return $this->database->deleteEmployee($id);
    }
}

// ব্যবহার
$admin = new User('U001', 'আদনান (Admin)', Role::Admin, [
    Permission::Read, Permission::Write, Permission::Delete,
    Permission::ManageUsers, Permission::ViewSalary,
]);

$employee = new User('U002', 'তানভীর (Employee)', Role::Employee, [
    Permission::Read,
]);

$adminDb = new ProtectedEmployeeDatabase($admin);
$employeeDb = new ProtectedEmployeeDatabase($employee);

// Admin সব করতে পারে
echo $adminDb->getSalary('E001') . "\n"; // ✅ কাজ করবে

// Employee বেতন দেখতে পারে না
try {
    $employeeDb->getSalary('E001'); // ⛔ Exception!
} catch (\RuntimeException $e) {
    echo $e->getMessage() . "\n";
}
```

#### JavaScript ES2022+ — Protection Proxy (ES6 Proxy দিয়ে)

```javascript
class BankAccount {
    #balance;
    #owner;
    #transactions = [];

    constructor(owner, initialBalance = 0) {
        this.#owner = owner;
        this.#balance = initialBalance;
    }

    get balance() { return this.#balance; }
    get owner() { return this.#owner; }

    deposit(amount) {
        this.#balance += amount;
        this.#transactions.push({ type: 'deposit', amount, date: new Date() });
        return this.#balance;
    }

    withdraw(amount) {
        if (amount > this.#balance) throw new Error('অপর্যাপ্ত ব্যালেন্স');
        this.#balance -= amount;
        this.#transactions.push({ type: 'withdraw', amount, date: new Date() });
        return this.#balance;
    }

    getTransactionHistory() {
        return [...this.#transactions];
    }
}

// Protection Proxy Factory — ES6 Proxy ব্যবহার করে
function createProtectedAccount(account, currentUser) {
    const permissions = {
        admin:    { balance: true, deposit: true, withdraw: true, getTransactionHistory: true },
        manager:  { balance: true, deposit: true, withdraw: true, getTransactionHistory: true },
        teller:   { balance: true, deposit: true, withdraw: false, getTransactionHistory: false },
        viewer:   { balance: true, deposit: false, withdraw: false, getTransactionHistory: false },
    };

    const userPerms = permissions[currentUser.role] ?? {};

    return new Proxy(account, {
        get(target, prop, receiver) {
            // owner সবসময় দেখা যাবে
            if (prop === 'owner') return target.owner;

            if (!(prop in userPerms)) {
                return Reflect.get(target, prop, receiver);
            }

            if (!userPerms[prop]) {
                if (typeof target[prop] === 'function') {
                    return () => {
                        throw new Error(
                            `⛔ ${currentUser.name} (${currentUser.role}) এর '${prop}' অ্যাক্সেস নেই`
                        );
                    };
                }
                throw new Error(
                    `⛔ ${currentUser.name} (${currentUser.role}) এর '${prop}' দেখার অনুমতি নেই`
                );
            }

            const value = Reflect.get(target, prop, receiver);
            return typeof value === 'function' ? value.bind(target) : value;
        },
    });
}

// ব্যবহার
const account = new BankAccount('রহিম উদ্দিন', 50000);
const tellerView = createProtectedAccount(account, { name: 'ক্যাশিয়ার', role: 'teller' });

console.log(tellerView.balance);        // ✅ 50000
tellerView.deposit(5000);               // ✅ কাজ করবে
try {
    tellerView.withdraw(1000);           // ⛔ অ্যাক্সেস নেই
} catch (e) {
    console.error(e.message);
}
```

---

### ৩. Remote Proxy (Network Access)

Remote Proxy একটি দূরবর্তী সার্ভারের অবজেক্টকে **লোকালি** ব্যবহার করার সুবিধা দেয়। ক্লায়েন্ট মনে করে সে লোকাল অবজেক্টের সাথে কাজ করছে, কিন্তু আসলে নেটওয়ার্ক কল হচ্ছে।

#### PHP 8.3 — Remote Proxy (bKash API)

```php
<?php

declare(strict_types=1);

// Subject Interface
interface PaymentGateway
{
    public function sendMoney(string $to, float $amount): array;
    public function checkBalance(): float;
    public function getTransactionStatus(string $txId): array;
}

// Remote Proxy — bKash API-র প্রতিনিধি
class BkashApiProxy implements PaymentGateway
{
    private ?string $authToken = null;
    private readonly string $baseUrl;

    public function __construct(
        private readonly string $appKey,
        private readonly string $appSecret,
        private readonly string $username,
        private readonly string $password,
        string $sandbox = 'true',
    ) {
        $this->baseUrl = $sandbox === 'true'
            ? 'https://tokenized.sandbox.bka.sh/v1.2.0-beta'
            : 'https://tokenized.pay.bka.sh/v1.2.0-beta';
    }

    private function ensureAuthenticated(): void
    {
        if ($this->authToken !== null) return;

        echo "🔐 bKash API-তে authenticate হচ্ছে...\n";

        // HTTP কল সিমুলেশন
        $response = $this->httpPost('/tokenized/checkout/token/grant', [
            'app_key' => $this->appKey,
            'app_secret' => $this->appSecret,
        ], [
            'username' => $this->username,
            'password' => $this->password,
        ]);

        $this->authToken = $response['id_token'] ?? throw new \RuntimeException('Auth ব্যর্থ');
    }

    public function sendMoney(string $to, float $amount): array
    {
        $this->ensureAuthenticated();
        $this->validatePhoneNumber($to);

        echo "💸 bKash-এ {$amount} টাকা পাঠানো হচ্ছে: {$to}\n";

        return $this->httpPost('/tokenized/checkout/payment/create', [
            'mode' => '0011',
            'payerReference' => $to,
            'amount' => number_format($amount, 2, '.', ''),
            'currency' => 'BDT',
            'intent' => 'sale',
        ]);
    }

    public function checkBalance(): float
    {
        $this->ensureAuthenticated();
        $response = $this->httpGet('/tokenized/checkout/balance');
        return (float) ($response['availableBalance'] ?? 0);
    }

    public function getTransactionStatus(string $txId): array
    {
        $this->ensureAuthenticated();
        return $this->httpPost('/tokenized/checkout/payment/status', [
            'paymentID' => $txId,
        ]);
    }

    private function validatePhoneNumber(string $phone): void
    {
        if (!preg_match('/^01[3-9]\d{8}$/', $phone)) {
            throw new \InvalidArgumentException("অবৈধ ফোন নম্বর: {$phone}");
        }
    }

    private function httpPost(string $endpoint, array $body, array $headers = []): array
    {
        // সিমুলেশন — আসল কোডে cURL/Guzzle ব্যবহার হবে
        return ['status' => 'success', 'id_token' => 'mock_token_' . bin2hex(random_bytes(8))];
    }

    private function httpGet(string $endpoint): array
    {
        return ['availableBalance' => '15000.00'];
    }
}
```

#### JavaScript ES2022+ — Remote Proxy

```javascript
class RemoteServiceProxy {
    #baseUrl;
    #cache = new Map();
    #pendingRequests = new Map();

    constructor(baseUrl, options = {}) {
        this.#baseUrl = baseUrl;
        this.timeout = options.timeout ?? 5000;
        this.retries = options.retries ?? 3;
    }

    // Deduplication: একই রিকোয়েস্ট একবারই যাবে
    async #fetchWithDedup(url, options = {}) {
        const key = `${options.method ?? 'GET'}:${url}`;

        if (this.#pendingRequests.has(key)) {
            return this.#pendingRequests.get(key);
        }

        const promise = this.#fetchWithRetry(url, options);
        this.#pendingRequests.set(key, promise);

        try {
            return await promise;
        } finally {
            this.#pendingRequests.delete(key);
        }
    }

    async #fetchWithRetry(url, options, attempt = 1) {
        try {
            const controller = new AbortController();
            const timeoutId = setTimeout(() => controller.abort(), this.timeout);

            const response = await fetch(url, {
                ...options,
                signal: controller.signal,
            });

            clearTimeout(timeoutId);

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            return response.json();
        } catch (error) {
            if (attempt < this.retries) {
                const delay = Math.min(1000 * 2 ** attempt, 10000);
                console.log(`🔄 পুনরায় চেষ্টা (${attempt}/${this.retries})... ${delay}ms পর`);
                await new Promise(r => setTimeout(r, delay));
                return this.#fetchWithRetry(url, options, attempt + 1);
            }
            throw error;
        }
    }

    async get(endpoint) {
        return this.#fetchWithDedup(`${this.#baseUrl}${endpoint}`);
    }

    async post(endpoint, data) {
        return this.#fetchWithDedup(`${this.#baseUrl}${endpoint}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data),
        });
    }
}

// ব্যবহার
const api = new RemoteServiceProxy('https://api.example.com', {
    timeout: 3000,
    retries: 3,
});
```

---

### ৪. Caching Proxy

Caching Proxy একই রিকোয়েস্টের জন্য **পুনরায় ব্যয়বহুল অপারেশন** এড়ায়। ক্যাশে থাকলে সেখান থেকে দেয়, না থাকলে মূল সোর্স থেকে এনে ক্যাশে রাখে।

#### PHP 8.3 — Caching Proxy

```php
<?php

declare(strict_types=1);

interface WeatherService
{
    public function getCurrentWeather(string $city): array;
    public function getForecast(string $city, int $days): array;
}

class RealWeatherService implements WeatherService
{
    public function getCurrentWeather(string $city): array
    {
        echo "🌐 API কল হচ্ছে: {$city}-এর আবহাওয়া\n";
        usleep(200_000); // নেটওয়ার্ক লেটেন্সি সিমুলেশন

        return [
            'city' => $city,
            'temp' => rand(20, 38),
            'humidity' => rand(60, 95),
            'condition' => ['☀️ রোদ', '🌧️ বৃষ্টি', '☁️ মেঘলা'][array_rand(['☀️ রোদ', '🌧️ বৃষ্টি', '☁️ মেঘলা'])],
            'fetched_at' => date('H:i:s'),
        ];
    }

    public function getForecast(string $city, int $days): array
    {
        echo "🌐 API কল হচ্ছে: {$city}-এর {$days} দিনের পূর্বাভাস\n";
        usleep(500_000);

        return array_map(fn(int $d) => [
            'day' => $d,
            'temp_high' => rand(28, 38),
            'temp_low' => rand(18, 26),
        ], range(1, $days));
    }
}

class CachingWeatherProxy implements WeatherService
{
    private array $cache = [];

    public function __construct(
        private readonly WeatherService $service,
        private readonly int $ttlSeconds = 300, // ৫ মিনিট ক্যাশ
    ) {}

    private function getCached(string $key): ?array
    {
        if (!isset($this->cache[$key])) return null;

        $entry = $this->cache[$key];
        if (time() - $entry['timestamp'] > $this->ttlSeconds) {
            echo "⏰ ক্যাশ মেয়াদ শেষ: {$key}\n";
            unset($this->cache[$key]);
            return null;
        }

        echo "⚡ ক্যাশ থেকে: {$key}\n";
        return $entry['data'];
    }

    private function setCache(string $key, array $data): void
    {
        $this->cache[$key] = [
            'data' => $data,
            'timestamp' => time(),
        ];
    }

    public function getCurrentWeather(string $city): array
    {
        $key = "weather:{$city}";
        return $this->getCached($key) ?? $this->fetchAndCache(
            $key,
            fn() => $this->service->getCurrentWeather($city)
        );
    }

    public function getForecast(string $city, int $days): array
    {
        $key = "forecast:{$city}:{$days}";
        return $this->getCached($key) ?? $this->fetchAndCache(
            $key,
            fn() => $this->service->getForecast($city, $days)
        );
    }

    private function fetchAndCache(string $key, callable $fetcher): array
    {
        $data = $fetcher();
        $this->setCache($key, $data);
        return $data;
    }

    public function invalidate(string $city): void
    {
        foreach (array_keys($this->cache) as $key) {
            if (str_contains($key, $city)) {
                unset($this->cache[$key]);
            }
        }
        echo "🗑️ {$city}-এর ক্যাশ মুছে ফেলা হয়েছে\n";
    }

    public function getCacheStats(): array
    {
        return [
            'entries' => count($this->cache),
            'keys' => array_keys($this->cache),
        ];
    }
}

// ব্যবহার
$weather = new CachingWeatherProxy(new RealWeatherService(), ttlSeconds: 600);

echo $weather->getCurrentWeather('ঢাকা')['condition'] . "\n";   // 🌐 API কল
echo $weather->getCurrentWeather('ঢাকা')['condition'] . "\n";   // ⚡ ক্যাশ থেকে
echo $weather->getCurrentWeather('চট্টগ্রাম')['condition'] . "\n"; // 🌐 নতুন API কল
```

#### JavaScript ES2022+ — Caching Proxy (WeakRef + FinalizationRegistry)

```javascript
class SmartCache {
    #cache = new Map();
    #ttlMs;
    #maxSize;
    #hits = 0;
    #misses = 0;

    constructor({ ttlMs = 60_000, maxSize = 100 } = {}) {
        this.#ttlMs = ttlMs;
        this.#maxSize = maxSize;
    }

    get(key) {
        const entry = this.#cache.get(key);
        if (!entry) { this.#misses++; return undefined; }
        if (Date.now() - entry.createdAt > this.#ttlMs) {
            this.#cache.delete(key);
            this.#misses++;
            return undefined;
        }
        entry.accessCount++;
        this.#hits++;
        return structuredClone(entry.value); // deep copy ফেরত দেয়
    }

    set(key, value) {
        if (this.#cache.size >= this.#maxSize) {
            this.#evictLeastUsed();
        }
        this.#cache.set(key, {
            value: structuredClone(value),
            createdAt: Date.now(),
            accessCount: 0,
        });
    }

    #evictLeastUsed() {
        let leastKey = null;
        let leastCount = Infinity;
        for (const [key, entry] of this.#cache) {
            if (entry.accessCount < leastCount) {
                leastCount = entry.accessCount;
                leastKey = key;
            }
        }
        if (leastKey) this.#cache.delete(leastKey);
    }

    get stats() {
        return {
            size: this.#cache.size,
            hits: this.#hits,
            misses: this.#misses,
            hitRate: this.#hits + this.#misses > 0
                ? ((this.#hits / (this.#hits + this.#misses)) * 100).toFixed(1) + '%'
                : 'N/A',
        };
    }
}

// Generic Caching Proxy Factory
function withCaching(target, { ttlMs = 30_000, methods = [] } = {}) {
    const cache = new SmartCache({ ttlMs });

    return new Proxy(target, {
        get(obj, prop, receiver) {
            const value = Reflect.get(obj, prop, receiver);

            if (typeof value !== 'function' || !methods.includes(prop)) {
                return value;
            }

            return async (...args) => {
                const cacheKey = `${prop}:${JSON.stringify(args)}`;
                const cached = cache.get(cacheKey);

                if (cached !== undefined) {
                    console.log(`⚡ ক্যাশ হিট: ${prop}(${args.join(', ')})`);
                    return cached;
                }

                console.log(`🌐 ফেচ করা হচ্ছে: ${prop}(${args.join(', ')})`);
                const result = await value.apply(obj, args);
                cache.set(cacheKey, result);
                return result;
            };
        },
    });
}

// ব্যবহার
class ProductService {
    async getProduct(id) {
        // API call সিমুলেশন
        return { id, name: `পণ্য-${id}`, price: Math.random() * 1000 };
    }

    async searchProducts(query) {
        return [{ name: `${query} ফলাফল ১` }, { name: `${query} ফলাফল ২` }];
    }
}

const cachedProducts = withCaching(new ProductService(), {
    ttlMs: 60_000,
    methods: ['getProduct', 'searchProducts'],
});
```

---

### ৫. Logging Proxy

Logging Proxy প্রতিটি অপারেশনের **বিস্তারিত লগ** রাখে — debugging, auditing এবং monitoring-এর জন্য অত্যন্ত কার্যকর।

#### PHP 8.3 — Logging Proxy

```php
<?php

declare(strict_types=1);

interface Logger
{
    public function log(string $level, string $message, array $context = []): void;
}

class ConsoleLogger implements Logger
{
    public function log(string $level, string $message, array $context = []): void
    {
        $emoji = match($level) {
            'info' => 'ℹ️', 'warning' => '⚠️', 'error' => '❌',
            'debug' => '🔍', default => '📝',
        };
        $ctx = $context ? ' | ' . json_encode($context, JSON_UNESCAPED_UNICODE) : '';
        echo "[{$emoji} {$level}] {$message}{$ctx}\n";
    }
}

// Generic Logging Proxy — যেকোনো ইন্টারফেসের জন্য কাজ করবে
class LoggingProxy
{
    public static function create(
        object $target,
        Logger $logger,
        string $serviceName = '',
    ): object {
        $serviceName = $serviceName ?: $target::class;

        return new class($target, $logger, $serviceName) {
            private int $callCount = 0;

            public function __construct(
                private readonly object $target,
                private readonly Logger $logger,
                private readonly string $serviceName,
            ) {}

            public function __call(string $method, array $args): mixed
            {
                $this->callCount++;
                $callId = substr(md5(uniqid()), 0, 8);
                $startTime = hrtime(true);

                $this->logger->log('info', "▶ [{$this->serviceName}] {$method}() শুরু", [
                    'call_id' => $callId,
                    'call_number' => $this->callCount,
                    'args' => array_map(fn($a) => is_scalar($a) ? $a : gettype($a), $args),
                ]);

                try {
                    $result = $this->target->{$method}(...$args);
                    $elapsed = (hrtime(true) - $startTime) / 1_000_000;

                    $this->logger->log('info', "✅ [{$this->serviceName}] {$method}() সফল", [
                        'call_id' => $callId,
                        'elapsed_ms' => round($elapsed, 2),
                        'result_type' => gettype($result),
                    ]);

                    return $result;
                } catch (\Throwable $e) {
                    $elapsed = (hrtime(true) - $startTime) / 1_000_000;

                    $this->logger->log('error', "❌ [{$this->serviceName}] {$method}() ব্যর্থ", [
                        'call_id' => $callId,
                        'elapsed_ms' => round($elapsed, 2),
                        'error' => $e->getMessage(),
                        'exception' => $e::class,
                    ]);

                    throw $e;
                }
            }
        };
    }
}

// ব্যবহার
class OrderService
{
    public function createOrder(string $product, int $qty): array
    {
        usleep(50_000);
        return ['id' => 'ORD-' . rand(1000, 9999), 'product' => $product, 'qty' => $qty];
    }
}

$service = LoggingProxy::create(new OrderService(), new ConsoleLogger(), 'OrderService');
$service->createOrder('ল্যাপটপ', 2);
```

#### JavaScript ES2022+ — Logging Proxy

```javascript
function createLoggingProxy(target, { name = target.constructor.name, logger = console } = {}) {
    let callCount = 0;

    return new Proxy(target, {
        get(obj, prop, receiver) {
            const value = Reflect.get(obj, prop, receiver);
            if (typeof value !== 'function') return value;

            return function (...args) {
                const callId = crypto.randomUUID().slice(0, 8);
                const start = performance.now();
                callCount++;

                logger.log(`▶ [${name}] ${prop}() শুরু | Call #${callCount} | ID: ${callId}`);

                try {
                    const result = value.apply(obj, args);

                    // Promise হ্যান্ডল করা (async মেথডের জন্য)
                    if (result instanceof Promise) {
                        return result
                            .then(res => {
                                const ms = (performance.now() - start).toFixed(2);
                                logger.log(`✅ [${name}] ${prop}() সফল | ${ms}ms | ID: ${callId}`);
                                return res;
                            })
                            .catch(err => {
                                const ms = (performance.now() - start).toFixed(2);
                                logger.error(`❌ [${name}] ${prop}() ব্যর্থ | ${ms}ms | ${err.message}`);
                                throw err;
                            });
                    }

                    const ms = (performance.now() - start).toFixed(2);
                    logger.log(`✅ [${name}] ${prop}() সফল | ${ms}ms | ID: ${callId}`);
                    return result;
                } catch (err) {
                    const ms = (performance.now() - start).toFixed(2);
                    logger.error(`❌ [${name}] ${prop}() ব্যর্থ | ${ms}ms | ${err.message}`);
                    throw err;
                }
            };
        },
    });
}
```

---

### ৬. Smart Reference Proxy

Smart Reference Proxy মূল অবজেক্টের **reference count** ট্র্যাক করে এবং অতিরিক্ত housekeeping কাজ করে — যেমন অবজেক্ট আর ব্যবহার না হলে রিসোর্স মুক্ত করা।

#### PHP 8.3 — Smart Reference Proxy

```php
<?php

declare(strict_types=1);

class DatabaseConnection
{
    private bool $connected = false;

    public function __construct(private readonly string $dsn) {}

    public function connect(): void
    {
        echo "🔌 ডেটাবেসে সংযোগ হচ্ছে: {$this->dsn}\n";
        $this->connected = true;
    }

    public function disconnect(): void
    {
        echo "🔌 সংযোগ বিচ্ছিন্ন: {$this->dsn}\n";
        $this->connected = false;
    }

    public function query(string $sql): array
    {
        if (!$this->connected) throw new \RuntimeException('সংযুক্ত নয়');
        return ['result' => "Query result for: {$sql}"];
    }

    public function isConnected(): bool { return $this->connected; }
}

class SmartConnectionProxy
{
    private ?DatabaseConnection $connection = null;
    private int $referenceCount = 0;
    private int $queryCount = 0;
    private float $lastUsedAt;

    public function __construct(
        private readonly string $dsn,
        private readonly int $maxIdleSeconds = 300,
    ) {
        $this->lastUsedAt = microtime(true);
    }

    public function acquire(): self
    {
        $this->referenceCount++;

        if ($this->connection === null) {
            $this->connection = new DatabaseConnection($this->dsn);
            $this->connection->connect();
        }

        echo "📊 Reference count: {$this->referenceCount}\n";
        return $this;
    }

    public function release(): void
    {
        $this->referenceCount = max(0, $this->referenceCount - 1);
        echo "📊 Reference count: {$this->referenceCount}\n";

        if ($this->referenceCount === 0) {
            echo "♻️ কেউ ব্যবহার করছে না — সংযোগ বন্ধ হচ্ছে\n";
            $this->connection?->disconnect();
            $this->connection = null;
        }
    }

    public function query(string $sql): array
    {
        if ($this->connection === null) {
            throw new \RuntimeException('acquire() আগে কল করুন');
        }

        $this->lastUsedAt = microtime(true);
        $this->queryCount++;

        return $this->connection->query($sql);
    }

    public function getStats(): array
    {
        return [
            'references' => $this->referenceCount,
            'queries' => $this->queryCount,
            'connected' => $this->connection?->isConnected() ?? false,
            'idle_seconds' => round(microtime(true) - $this->lastUsedAt, 2),
        ];
    }
}
```

---

## 🌍 Real-World Applicable Areas

### ১. Browser Image Lazy Loading (Virtual Proxy ধারণা)

```javascript
// IntersectionObserver দিয়ে Image Lazy Loading Proxy
class LazyImageLoader {
    #observer;
    #loadedCount = 0;

    constructor(options = {}) {
        this.#observer = new IntersectionObserver(
            (entries) => this.#handleIntersection(entries),
            { rootMargin: options.rootMargin ?? '200px', threshold: 0 }
        );
    }

    observe(imgElement) {
        // প্লেসহোল্ডার সেট করা
        const realSrc = imgElement.dataset.src;
        imgElement.src = 'data:image/svg+xml,...'; // প্লেসহোল্ডার

        imgElement._proxiedSrc = realSrc;
        this.#observer.observe(imgElement);
    }

    #handleIntersection(entries) {
        for (const entry of entries) {
            if (!entry.isIntersecting) continue;

            const img = entry.target;
            console.log(`🖼️ Lazy load: ${img._proxiedSrc}`);

            img.src = img._proxiedSrc;
            img.onload = () => { this.#loadedCount++; };
            this.#observer.unobserve(img);
        }
    }

    get stats() {
        return { loaded: this.#loadedCount };
    }
}
```

### ২. API Rate Limiting Proxy

#### PHP 8.3

```php
<?php

declare(strict_types=1);

class RateLimitingProxy
{
    private array $requestLog = [];

    public function __construct(
        private readonly object $service,
        private readonly int $maxRequests = 60,
        private readonly int $windowSeconds = 60,
    ) {}

    public function __call(string $method, array $args): mixed
    {
        $this->cleanOldEntries();

        if (count($this->requestLog) >= $this->maxRequests) {
            $oldestTimestamp = $this->requestLog[0];
            $retryAfter = $this->windowSeconds - (time() - $oldestTimestamp);

            throw new \RuntimeException(
                "⚠️ Rate limit অতিক্রম! সর্বোচ্চ {$this->maxRequests} রিকোয়েস্ট/"
                . "{$this->windowSeconds}সেকেন্ড। {$retryAfter} সেকেন্ড পর চেষ্টা করুন।"
            );
        }

        $this->requestLog[] = time();
        return $this->service->{$method}(...$args);
    }

    private function cleanOldEntries(): void
    {
        $cutoff = time() - $this->windowSeconds;
        $this->requestLog = array_values(
            array_filter($this->requestLog, fn(int $t) => $t > $cutoff)
        );
    }

    public function getRemainingRequests(): int
    {
        $this->cleanOldEntries();
        return max(0, $this->maxRequests - count($this->requestLog));
    }
}
```

#### JavaScript ES2022+

```javascript
class RateLimiter {
    #requests = [];
    #maxRequests;
    #windowMs;

    constructor(maxRequests = 60, windowMs = 60_000) {
        this.#maxRequests = maxRequests;
        this.#windowMs = windowMs;
    }

    tryAcquire() {
        const now = Date.now();
        this.#requests = this.#requests.filter(t => now - t < this.#windowMs);

        if (this.#requests.length >= this.#maxRequests) {
            const retryAfterMs = this.#windowMs - (now - this.#requests[0]);
            return { allowed: false, retryAfterMs };
        }

        this.#requests.push(now);
        return { allowed: true, remaining: this.#maxRequests - this.#requests.length };
    }
}

function withRateLimit(target, { maxRequests = 100, windowMs = 60_000, methods } = {}) {
    const limiter = new RateLimiter(maxRequests, windowMs);

    return new Proxy(target, {
        get(obj, prop) {
            const value = obj[prop];
            if (typeof value !== 'function') return value;
            if (methods && !methods.includes(prop)) return value.bind(obj);

            return function (...args) {
                const result = limiter.tryAcquire();
                if (!result.allowed) {
                    throw new Error(
                        `⚠️ Rate limit! ${Math.ceil(result.retryAfterMs / 1000)}s পর চেষ্টা করুন`
                    );
                }
                console.log(`✅ অনুমোদিত (বাকি: ${result.remaining})`);
                return value.apply(obj, args);
            };
        },
    });
}
```

### ৩. Full Working Example: API Proxy with Caching + Rate Limiting

```javascript
// সম্পূর্ণ প্রোডাকশন-রেডি API Proxy
class ApiGatewayProxy {
    #baseUrl;
    #cache;
    #rateLimiter;
    #authToken;
    #requestQueue = [];
    #processing = false;

    constructor(baseUrl, {
        cacheMaxAge = 60_000,
        maxRequestsPerMinute = 100,
        authToken = null,
    } = {}) {
        this.#baseUrl = baseUrl;
        this.#cache = new Map();
        this.#rateLimiter = new RateLimiter(maxRequestsPerMinute, 60_000);
        this.#authToken = authToken;
    }

    async get(endpoint, { cache = true, cacheTtl = 30_000 } = {}) {
        // ১. ক্যাশ চেক
        if (cache) {
            const cacheKey = `GET:${endpoint}`;
            const cached = this.#getFromCache(cacheKey, cacheTtl);
            if (cached) return cached;
        }

        // ২. Rate limit চেক
        const rl = this.#rateLimiter.tryAcquire();
        if (!rl.allowed) {
            throw new Error(`Rate limited. ${Math.ceil(rl.retryAfterMs / 1000)}s অপেক্ষা করুন`);
        }

        // ৩. আসল রিকোয়েস্ট
        const response = await this.#fetch(endpoint);

        // ৪. ক্যাশে সংরক্ষণ
        if (cache) {
            this.#cache.set(`GET:${endpoint}`, {
                data: response,
                timestamp: Date.now(),
            });
        }

        return response;
    }

    async post(endpoint, data) {
        const rl = this.#rateLimiter.tryAcquire();
        if (!rl.allowed) throw new Error('Rate limited');

        // POST ক্যাশ invalidate করে
        this.#invalidateRelatedCache(endpoint);

        return this.#fetch(endpoint, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data),
        });
    }

    #getFromCache(key, ttl) {
        const entry = this.#cache.get(key);
        if (!entry) return null;
        if (Date.now() - entry.timestamp > ttl) {
            this.#cache.delete(key);
            return null;
        }
        console.log(`⚡ ক্যাশ থেকে: ${key}`);
        return entry.data;
    }

    #invalidateRelatedCache(endpoint) {
        const basePath = endpoint.split('?')[0];
        for (const key of this.#cache.keys()) {
            if (key.includes(basePath)) this.#cache.delete(key);
        }
    }

    async #fetch(endpoint, options = {}) {
        const headers = { ...options.headers };
        if (this.#authToken) {
            headers['Authorization'] = `Bearer ${this.#authToken}`;
        }

        const response = await fetch(`${this.#baseUrl}${endpoint}`, {
            ...options,
            headers,
        });

        if (!response.ok) {
            throw new Error(`API Error: ${response.status} ${response.statusText}`);
        }

        return response.json();
    }
}

// ব্যবহার — bKash-style API Proxy
const bkashApi = new ApiGatewayProxy('https://api.bkash.com/v1', {
    maxRequestsPerMinute: 30,
    authToken: 'your-bkash-token',
    cacheMaxAge: 120_000,
});
```

### ৪. Laravel-তে Proxy প্যাটার্ন

```php
<?php

// Laravel Eloquent-এর Lazy Loading নিজেই Proxy Pattern!

// ১. Relationship Lazy Loading (Implicit Proxy)
$user = User::find(1);
// $user->posts এখনো লোড হয়নি — এটি একটি Proxy-like behavior
// শুধু যখন $user->posts অ্যাক্সেস করা হবে, তখনই SQL query হবে
foreach ($user->posts as $post) {
    // এখানে SELECT * FROM posts WHERE user_id = 1 চলবে
    echo $post->title;
}

// ২. Laravel Facades = Static Proxy
// Cache::get('key') আসলে:
// app('cache')->get('key')
// Facade একটি Proxy যা Service Container থেকে real instance resolve করে

// ৩. Custom Proxy উদাহরণ — Service Layer-এ
class ExternalApiServiceProxy
{
    public function __construct(
        private readonly ExternalApiService $service,
        private readonly CacheRepository $cache,
        private readonly RateLimiter $limiter,
    ) {}

    public function getData(string $key): array
    {
        // ক্যাশ চেক
        $cached = $this->cache->get("api:{$key}");
        if ($cached !== null) return $cached;

        // Rate limit চেক
        if (!$this->limiter->attempt("api-call", 60, 100)) {
            throw new TooManyRequestsException('API rate limit exceeded');
        }

        // আসল কল
        $data = $this->service->getData($key);
        $this->cache->put("api:{$key}", $data, 600);

        return $data;
    }
}
```

### ৫. CDN ও Nginx Reverse Proxy ধারণা

```
CDN as Proxy (ধারণাগত):
┌──────────┐     ┌──────────────┐     ┌───────────────┐
│  ব্যবহারকারী  │────▶│   CDN (Proxy) │────▶│ Origin Server │
│  (ঢাকা)     │     │  (সিঙ্গাপুর)   │     │   (US-East)    │
└──────────┘     └──────────────┘     └───────────────┘
                      │
                      ├─ ক্যাশিং (Static assets)
                      ├─ DDoS Protection
                      ├─ SSL Termination
                      └─ Geographic routing

Nginx Reverse Proxy:
┌────────┐     ┌────────────────┐     ┌──────────────┐
│ Client │────▶│ Nginx (Proxy)  │────▶│ App Server   │
└────────┘     ├────────────────┤     │ (PHP/Node)   │
               │ - Load Balance │     └──────────────┘
               │ - SSL          │     ┌──────────────┐
               │ - Rate Limit   │────▶│ App Server 2 │
               │ - Caching      │     └──────────────┘
               │ - Compression  │     ┌──────────────┐
               └────────────────┘────▶│ App Server 3 │
                                      └──────────────┘
```

---

## 🔥 Advanced Deep Dive

### ১. JavaScript Proxy Object — সম্পূর্ণ গভীর বিশ্লেষণ

ES6 `Proxy` অবজেক্ট হলো JavaScript-এর **metaprogramming** ক্ষমতার অন্যতম শক্তিশালী টুল। এটি দিয়ে যেকোনো অবজেক্টের fundamental operations **intercept** এবং **redefine** করা যায়।

```javascript
// সকল গুরুত্বপূর্ণ Handler Traps

const handler = {
    // ১. get — প্রোপার্টি পড়ার সময়
    get(target, prop, receiver) {
        console.log(`📖 পড়া হচ্ছে: ${String(prop)}`);
        return Reflect.get(target, prop, receiver);
    },

    // ২. set — প্রোপার্টি সেট করার সময়
    set(target, prop, value, receiver) {
        console.log(`✏️ সেট হচ্ছে: ${String(prop)} = ${value}`);

        // Validation proxy
        if (prop === 'age' && (typeof value !== 'number' || value < 0 || value > 150)) {
            throw new TypeError(`অবৈধ বয়স: ${value}`);
        }

        return Reflect.set(target, prop, value, receiver);
    },

    // ৩. has — 'in' অপারেটর
    has(target, prop) {
        console.log(`🔍 চেক হচ্ছে: '${String(prop)}' in object`);
        // প্রাইভেট প্রোপার্টি লুকানো
        if (typeof prop === 'string' && prop.startsWith('_')) return false;
        return Reflect.has(target, prop);
    },

    // ৪. deleteProperty — delete অপারেটর
    deleteProperty(target, prop) {
        console.log(`🗑️ মুছে ফেলা হচ্ছে: ${String(prop)}`);
        if (prop === 'id') {
            throw new Error('id মুছে ফেলা যাবে না!');
        }
        return Reflect.deleteProperty(target, prop);
    },

    // ৫. apply — ফাংশন কল (শুধু ফাংশন Proxy-র জন্য)
    apply(target, thisArg, argumentsList) {
        console.log(`📞 কল হচ্ছে: args = [${argumentsList}]`);
        const start = performance.now();
        const result = Reflect.apply(target, thisArg, argumentsList);
        console.log(`⏱️ সময় লেগেছে: ${(performance.now() - start).toFixed(2)}ms`);
        return result;
    },

    // ৬. construct — new অপারেটর
    construct(target, args, newTarget) {
        console.log(`🏗️ নতুন ইনস্ট্যান্স তৈরি হচ্ছে`);
        return Reflect.construct(target, args, newTarget);
    },

    // ৭. ownKeys — Object.keys(), for...in
    ownKeys(target) {
        // _ দিয়ে শুরু হওয়া কী লুকানো
        return Reflect.ownKeys(target).filter(
            key => typeof key !== 'string' || !key.startsWith('_')
        );
    },

    // ৮. getOwnPropertyDescriptor
    getOwnPropertyDescriptor(target, prop) {
        return Reflect.getOwnPropertyDescriptor(target, prop);
    },

    // ৯. defineProperty
    defineProperty(target, prop, descriptor) {
        console.log(`📌 নতুন প্রোপার্টি ডিফাইন: ${String(prop)}`);
        return Reflect.defineProperty(target, prop, descriptor);
    },

    // ১০. getPrototypeOf
    getPrototypeOf(target) {
        return Reflect.getPrototypeOf(target);
    },
};

// ব্যবহারিক উদাহরণ: Reactive/Observable Object
function reactive(obj, onChange) {
    return new Proxy(obj, {
        set(target, prop, value, receiver) {
            const oldValue = target[prop];
            const result = Reflect.set(target, prop, value, receiver);

            if (oldValue !== value) {
                onChange({
                    property: prop,
                    oldValue,
                    newValue: value,
                    timestamp: Date.now(),
                });
            }

            return result;
        },

        deleteProperty(target, prop) {
            const oldValue = target[prop];
            const result = Reflect.deleteProperty(target, prop);
            onChange({ property: prop, oldValue, newValue: undefined, deleted: true });
            return result;
        },
    });
}

// Vue.js-এর reactivity system এভাবেই কাজ করে!
const state = reactive({ count: 0, name: 'ঢাকা' }, (change) => {
    console.log(`🔔 পরিবর্তন: ${change.property}: ${change.oldValue} → ${change.newValue}`);
});

state.count = 1;   // 🔔 পরিবর্তন: count: 0 → 1
state.name = 'চট্টগ্রাম';  // 🔔 পরিবর্তন: name: ঢাকা → চট্টগ্রাম
```

### ২. PHP Magic Methods as Proxy Mechanism

```php
<?php

declare(strict_types=1);

// PHP-তে __get, __set, __call ম্যাজিক মেথড Proxy-র মতো কাজ করে

class DynamicProxy
{
    private array $propertyAccessLog = [];
    private array $methodCallLog = [];

    public function __construct(
        private readonly object $target,
    ) {}

    // প্রোপার্টি পড়ার Proxy
    public function __get(string $name): mixed
    {
        $this->propertyAccessLog[] = [
            'action' => 'get',
            'property' => $name,
            'time' => microtime(true),
        ];

        return $this->target->{$name};
    }

    // প্রোপার্টি সেট করার Proxy
    public function __set(string $name, mixed $value): void
    {
        $this->propertyAccessLog[] = [
            'action' => 'set',
            'property' => $name,
            'value' => $value,
            'time' => microtime(true),
        ];

        $this->target->{$name} = $value;
    }

    // মেথড কলের Proxy
    public function __call(string $name, array $arguments): mixed
    {
        $start = hrtime(true);

        try {
            $result = $this->target->{$name}(...$arguments);

            $this->methodCallLog[] = [
                'method' => $name,
                'args_count' => count($arguments),
                'elapsed_ns' => hrtime(true) - $start,
                'success' => true,
            ];

            return $result;
        } catch (\Throwable $e) {
            $this->methodCallLog[] = [
                'method' => $name,
                'elapsed_ns' => hrtime(true) - $start,
                'success' => false,
                'error' => $e->getMessage(),
            ];
            throw $e;
        }
    }

    // প্রোপার্টি আছে কি না চেক (has trap-এর সমতুল্য)
    public function __isset(string $name): bool
    {
        return isset($this->target->{$name});
    }

    public function __unset(string $name): void
    {
        unset($this->target->{$name});
    }

    public function getAccessLog(): array
    {
        return $this->propertyAccessLog;
    }

    public function getCallLog(): array
    {
        return $this->methodCallLog;
    }
}
```

### ৩. Proxy vs Decorator vs Adapter — তুলনামূলক বিশ্লেষণ

```
┌─────────────────┬────────────────────┬─────────────────────┬──────────────────────┐
│    বৈশিষ্ট্য      │   Proxy            │   Decorator          │   Adapter            │
├─────────────────┼────────────────────┼─────────────────────┼──────────────────────┤
│ উদ্দেশ্য         │ অ্যাক্সেস নিয়ন্ত্রণ    │ নতুন আচরণ যোগ করা    │ ইন্টারফেস রূপান্তর     │
│                 │                    │                     │                      │
│ ইন্টারফেস        │ Subject-এর         │ Component-এর        │ ভিন্ন (Target-এর      │
│                 │ একই ইন্টারফেস       │ একই ইন্টারফেস        │ ইন্টারফেসে রূপান্তর)    │
│                 │                    │                     │                      │
│ অবজেক্ট তৈরি     │ Proxy নিজে তৈরি     │ বাইরে থেকে inject    │ বাইরে থেকে inject     │
│                 │ করতে পারে          │ করা হয়              │ করা হয়               │
│                 │                    │                     │                      │
│ মূল কাজ          │ মধ্যস্থতাকারী — মূল   │ অতিরিক্ত দায়িত্ব যোগ  │ incompatible         │
│                 │ অবজেক্টে পৌঁছানোর   │ করে (wrapping)      │ ইন্টারফেস সামঞ্জস্যপূর্ণ │
│                 │ পূর্বে/পরে কাজ       │                     │ করে                  │
│                 │                    │                     │                      │
│ চেইনিং           │ সাধারণত একটি        │ একাধিক Decorator    │ সাধারণত একটি          │
│                 │                    │ চেইন করা যায়         │                      │
│                 │                    │                     │                      │
│ Real উদাহরণ      │ DB connection pool │ Java InputStream    │ USB-to-HDMI          │
│                 │ Auth middleware    │ React HOC           │ adapter              │
│                 │ Lazy loading       │ PHP Middleware       │ Legacy API wrapper   │
│                 │                    │                     │                      │
│ ক্লায়েন্ট জানে?  │ না (transparent)   │ সাধারণত না           │ হ্যাঁ (ভিন্ন interface) │
└─────────────────┴────────────────────┴─────────────────────┴──────────────────────┘
```

### ৪. Dynamic Proxy (Runtime Proxy Generation)

```php
<?php

declare(strict_types=1);

// Runtime-এ Proxy তৈরি করা — Reflection API ব্যবহার করে

class DynamicProxyFactory
{
    /**
     * যেকোনো ইন্টারফেসের জন্য Runtime-এ Proxy তৈরি করে
     *
     * @param class-string $interface
     * @param callable(string $method, array $args): mixed $invocationHandler
     */
    public static function create(string $interface, callable $invocationHandler): object
    {
        $reflection = new \ReflectionClass($interface);

        if (!$reflection->isInterface()) {
            throw new \InvalidArgumentException("{$interface} একটি ইন্টারফেস হতে হবে");
        }

        $methods = [];
        foreach ($reflection->getMethods() as $method) {
            $params = [];
            foreach ($method->getParameters() as $param) {
                $type = $param->getType()?->getName() ?? 'mixed';
                $params[] = "{$type} \${$param->getName()}";
            }
            $paramStr = implode(', ', $params);
            $argNames = array_map(
                fn($p) => '$' . $p->getName(),
                $method->getParameters()
            );
            $argStr = implode(', ', $argNames);

            $returnType = $method->getReturnType()?->getName() ?? 'mixed';

            $methods[] = "    public function {$method->getName()}({$paramStr}): {$returnType} {"
                . " return (\$this->handler)('{$method->getName()}', [{$argStr}]); }";
        }

        $className = 'DynamicProxy_' . md5($interface . microtime());
        $methodsCode = implode("\n", $methods);

        $code = "return new class(\$invocationHandler) implements \\{$interface} {"
            . "    private \$handler;"
            . "    public function __construct(callable \$handler) { \$this->handler = \$handler; }"
            . "    {$methodsCode}"
            . "};";

        return eval($code);
    }
}

// ব্যবহার
interface UserRepository
{
    public function findById(int $id): array;
    public function save(array $data): bool;
}

$proxy = DynamicProxyFactory::create(UserRepository::class, function(string $method, array $args) {
    echo "🔮 Dynamic Proxy: {$method}() কল হচ্ছে\n";
    return match($method) {
        'findById' => ['id' => $args[0], 'name' => 'ডায়নামিক ইউজার'],
        'save' => true,
        default => null,
    };
});

print_r($proxy->findById(42));
```

### ৫. Proxy Chain (Multiple Proxies Stacked)

```javascript
// একাধিক Proxy স্তরে সাজানো — প্রতিটি স্তর একটি নির্দিষ্ট দায়িত্ব পালন করে

class ProxyChainBuilder {
    #target;
    #layers = [];

    constructor(target) {
        this.#target = target;
    }

    addLogging(logger = console) {
        this.#layers.push((target) => new Proxy(target, {
            get(obj, prop) {
                const value = Reflect.get(obj, prop);
                if (typeof value !== 'function') return value;
                return function (...args) {
                    logger.log(`[LOG] ${prop}() কল হচ্ছে`);
                    return value.apply(obj, args);
                };
            },
        }));
        return this;
    }

    addValidation(rules = {}) {
        this.#layers.push((target) => new Proxy(target, {
            set(obj, prop, value) {
                if (rules[prop]) {
                    const error = rules[prop](value);
                    if (error) throw new TypeError(`[VALIDATION] ${prop}: ${error}`);
                }
                return Reflect.set(obj, prop, value);
            },
        }));
        return this;
    }

    addCaching(ttlMs = 30_000) {
        const cache = new Map();
        this.#layers.push((target) => new Proxy(target, {
            get(obj, prop) {
                const value = Reflect.get(obj, prop);
                if (typeof value !== 'function') return value;
                return function (...args) {
                    const key = `${prop}:${JSON.stringify(args)}`;
                    const cached = cache.get(key);
                    if (cached && Date.now() - cached.time < ttlMs) {
                        return cached.value;
                    }
                    const result = value.apply(obj, args);
                    cache.set(key, { value: result, time: Date.now() });
                    return result;
                };
            },
        }));
        return this;
    }

    addAccessControl(allowedMethods = []) {
        this.#layers.push((target) => new Proxy(target, {
            get(obj, prop) {
                if (typeof obj[prop] === 'function' && !allowedMethods.includes(prop)) {
                    return () => { throw new Error(`⛔ '${prop}' অ্যাক্সেস অস্বীকৃত`); };
                }
                return Reflect.get(obj, prop);
            },
        }));
        return this;
    }

    build() {
        // চেইনের প্রতিটি স্তর আগের স্তরকে wrap করে
        return this.#layers.reduce(
            (current, layer) => layer(current),
            this.#target
        );
    }
}

// ব্যবহার — একাধিক Proxy স্তর
class UserService {
    getUser(id) { return { id, name: 'রহিম' }; }
    deleteUser(id) { return `ব্যবহারকারী ${id} মুছে ফেলা হয়েছে`; }
    updateUser(id, data) { return { ...data, id, updated: true }; }
}

const protectedService = new ProxyChainBuilder(new UserService())
    .addLogging()                                              // স্তর ১: লগিং
    .addCaching(60_000)                                        // স্তর ২: ক্যাশিং
    .addAccessControl(['getUser', 'updateUser'])                // স্তর ৩: অ্যাক্সেস নিয়ন্ত্রণ
    .build();

// রিকোয়েস্ট ফ্লো:
// Client → AccessControl → Caching → Logging → RealService
console.log(protectedService.getUser(1));    // ✅ সব স্তর পার হয়ে কাজ করবে

try {
    protectedService.deleteUser(1);           // ⛔ AccessControl-এ আটকে যাবে
} catch (e) {
    console.error(e.message);
}
```

---

## ✅ Pros (সুবিধা) ও ❌ Cons (অসুবিধা)

### ✅ সুবিধাসমূহ

| সুবিধা | বিবরণ |
|--------|--------|
| **Single Responsibility** | প্রতিটি Proxy একটি নির্দিষ্ট দায়িত্ব পালন করে (SRP) |
| **Open/Closed Principle** | নতুন Proxy যোগ করা যায় মূল কোড পরিবর্তন ছাড়াই |
| **Lazy Initialization** | ভারী অবজেক্ট শুধু দরকার হলেই তৈরি হয় — মেমোরি বাঁচে |
| **Access Control** | ক্লায়েন্ট কোড পরিবর্তন ছাড়াই সিকিউরিটি যোগ করা যায় |
| **Transparent** | ক্লায়েন্ট জানে না সে Proxy-র সাথে কাজ করছে |
| **Caching** | পুনরাবৃত্তিমূলক অপারেশন দ্রুত হয় |
| **Logging/Monitoring** | সহজে অডিট ট্রেইল যোগ করা যায় |

### ❌ অসুবিধাসমূহ

| অসুবিধা | বিবরণ |
|---------|--------|
| **বিলম্ব (Latency)** | প্রতিটি কলে অতিরিক্ত একটি স্তর — কিছুটা ধীর |
| **জটিলতা বৃদ্ধি** | কোডবেসে নতুন ক্লাস/অবজেক্ট যোগ হয় |
| **ডিবাগিং কঠিন** | Proxy chain-এ সমস্যা খুঁজে পাওয়া কঠিন হতে পারে |
| **অপ্রয়োজনীয় Abstraction** | সব জায়গায় Proxy দরকার নয় — overengineering হতে পারে |
| **ক্যাশ Stale হওয়া** | Caching Proxy-তে পুরনো ডেটা সমস্যা তৈরি করতে পারে |

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ১. অপ্রয়োজনীয় Proxy তৈরি করা
```php
// ❌ ভুল — সাধারণ অবজেক্টের জন্য Proxy দরকার নেই
class SimpleConfigProxy implements Config
{
    public function __construct(private Config $config) {}
    public function get(string $key): string
    {
        return $this->config->get($key); // কোনো অতিরিক্ত কাজ নেই!
    }
}

// ✅ সঠিক — Proxy ব্যবহার করুন যখন সত্যিই অতিরিক্ত লজিক দরকার
```

### ২. Proxy-তে Business Logic রাখা
```php
// ❌ ভুল — Proxy-তে business logic
class OrderProxy implements OrderService
{
    public function createOrder(array $data): Order
    {
        // Proxy-র কাজ এটা না!
        $data['discount'] = $this->calculateDiscount($data);
        $data['tax'] = $this->calculateTax($data);

        return $this->service->createOrder($data);
    }
}

// ✅ সঠিক — Proxy শুধু cross-cutting concern (auth, cache, log) সামলাবে
```

### ৩. ক্যাশ Invalidation ভুলে যাওয়া
```javascript
// ❌ ভুল — ক্যাশ কখনো invalidate হয় না
class BadCachingProxy {
    get(key) {
        if (this.cache.has(key)) return this.cache.get(key); // চিরকালের জন্য ক্যাশ!
        const val = this.target.get(key);
        this.cache.set(key, val);
        return val;
    }
}

// ✅ সঠিক — TTL এবং invalidation mechanism রাখুন
```

### ৪. Proxy Interface অসম্পূর্ণ রাখা
```php
// ❌ ভুল — কিছু মেথড Proxy-তে নেই
class IncompleteProxy implements PaymentGateway
{
    public function pay(float $amount): bool { /* ... */ }
    // refund() মেথড ভুলে গেছি — ইন্টারফেস লঙ্ঘন!
}
```

### ৫. Circular Proxy Reference
```javascript
// ❌ ভুল — Proxy নিজেকে রেফারেন্স করছে → infinite loop
const proxy = new Proxy(obj, {
    get(target, prop) {
        return proxy[prop]; // 💥 infinite recursion!
    }
});

// ✅ সঠিক — Reflect ব্যবহার করুন
const proxy = new Proxy(obj, {
    get(target, prop) {
        return Reflect.get(target, prop);
    }
});
```

---

## 🧪 টেস্টিং

### PHPUnit — Proxy টেস্ট

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class CachingProxyTest extends TestCase
{
    private CachingWeatherProxy $proxy;
    private WeatherService&\PHPUnit\Framework\MockObject\MockObject $mockService;

    protected function setUp(): void
    {
        $this->mockService = $this->createMock(WeatherService::class);
        $this->proxy = new CachingWeatherProxy($this->mockService, ttlSeconds: 60);
    }

    public function test_প্রথম_কলে_আসল_সার্ভিস_কল_হবে(): void
    {
        $expected = ['city' => 'ঢাকা', 'temp' => 32];

        $this->mockService
            ->expects($this->once())
            ->method('getCurrentWeather')
            ->with('ঢাকা')
            ->willReturn($expected);

        $result = $this->proxy->getCurrentWeather('ঢাকা');

        $this->assertSame($expected, $result);
    }

    public function test_দ্বিতীয়_কলে_ক্যাশ_থেকে_আসবে(): void
    {
        $expected = ['city' => 'ঢাকা', 'temp' => 32];

        // আসল সার্ভিস শুধু একবার কল হবে — দ্বিতীয়বার ক্যাশ
        $this->mockService
            ->expects($this->once())
            ->method('getCurrentWeather')
            ->willReturn($expected);

        $this->proxy->getCurrentWeather('ঢাকা'); // প্রথম কল
        $result = $this->proxy->getCurrentWeather('ঢাকা'); // ক্যাশ থেকে

        $this->assertSame($expected, $result);
    }

    public function test_ভিন্ন_শহরে_নতুন_কল_হবে(): void
    {
        $this->mockService
            ->expects($this->exactly(2))
            ->method('getCurrentWeather')
            ->willReturnMap([
                ['ঢাকা', ['city' => 'ঢাকা', 'temp' => 32]],
                ['চট্টগ্রাম', ['city' => 'চট্টগ্রাম', 'temp' => 30]],
            ]);

        $this->proxy->getCurrentWeather('ঢাকা');
        $this->proxy->getCurrentWeather('চট্টগ্রাম');
    }
}

class ProtectionProxyTest extends TestCase
{
    public function test_অ্যাডমিন_সব_অপারেশন_করতে_পারবে(): void
    {
        $admin = new User('U001', 'Admin', Role::Admin, [
            Permission::Read, Permission::Write, Permission::Delete, Permission::ViewSalary,
        ]);

        $db = new ProtectedEmployeeDatabase($admin);

        $this->assertIsArray($db->getEmployee('E001'));
        $this->assertIsFloat($db->getSalary('E001'));
        $this->assertTrue($db->updateSalary('E001', 90000));
    }

    public function test_সাধারণ_কর্মচারী_বেতন_দেখতে_পারবে_না(): void
    {
        $employee = new User('U002', 'Employee', Role::Employee, [Permission::Read]);

        $db = new ProtectedEmployeeDatabase($employee);

        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessageMatches('/অ্যাক্সেস অস্বীকৃত/');

        $db->getSalary('E001');
    }

    public function test_গেস্ট_কিছুই_করতে_পারবে_না(): void
    {
        $guest = new User('U003', 'Guest', Role::Guest, []);

        $db = new ProtectedEmployeeDatabase($guest);

        $this->expectException(\RuntimeException::class);
        $db->getEmployee('E001');
    }
}

class VirtualProxyTest extends TestCase
{
    public function test_প্রক্সি_তৈরিতে_ইমেজ_লোড_হয়_না(): void
    {
        $startMemory = memory_get_usage();
        $proxy = new ImageProxy('large_image.jpg');
        $endMemory = memory_get_usage();

        // প্রক্সি তৈরিতে মেমোরি সামান্যই বাড়বে
        $this->assertLessThan(1024, $endMemory - $startMemory);
    }

    public function test_রেজোলিউশন_ইমেজ_লোড_ছাড়াই_পাওয়া_যায়(): void
    {
        $proxy = new ImageProxy('test.jpg', 1920, 1080);

        $this->assertSame('1920x1080', $proxy->getResolution());
    }
}
```

### Jest — Proxy টেস্ট

```javascript
describe('Caching Proxy', () => {
    let mockService;
    let cachedService;

    beforeEach(() => {
        mockService = {
            getProduct: jest.fn().mockImplementation(id =>
                Promise.resolve({ id, name: `পণ্য-${id}` })
            ),
            searchProducts: jest.fn().mockResolvedValue([]),
        };

        cachedService = withCaching(mockService, {
            ttlMs: 60_000,
            methods: ['getProduct', 'searchProducts'],
        });
    });

    test('প্রথম কলে আসল সার্ভিস কল হবে', async () => {
        await cachedService.getProduct(1);
        expect(mockService.getProduct).toHaveBeenCalledTimes(1);
        expect(mockService.getProduct).toHaveBeenCalledWith(1);
    });

    test('দ্বিতীয় কলে ক্যাশ থেকে আসবে', async () => {
        await cachedService.getProduct(1);
        await cachedService.getProduct(1);
        expect(mockService.getProduct).toHaveBeenCalledTimes(1); // শুধু একবার!
    });

    test('ভিন্ন আর্গুমেন্টে নতুন কল হবে', async () => {
        await cachedService.getProduct(1);
        await cachedService.getProduct(2);
        expect(mockService.getProduct).toHaveBeenCalledTimes(2);
    });
});

describe('Protection Proxy (ES6 Proxy)', () => {
    let account;

    beforeEach(() => {
        account = new BankAccount('টেস্ট ইউজার', 10000);
    });

    test('অনুমোদিত অপারেশন কাজ করবে', () => {
        const proxy = createProtectedAccount(account, { name: 'ম্যানেজার', role: 'manager' });

        expect(proxy.balance).toBe(10000);
        expect(proxy.deposit(5000)).toBe(15000);
        expect(proxy.withdraw(2000)).toBe(13000);
    });

    test('অননুমোদিত অপারেশনে Error হবে', () => {
        const proxy = createProtectedAccount(account, { name: 'ভিউয়ার', role: 'viewer' });

        expect(proxy.balance).toBe(10000); // ✅ দেখা যাবে
        expect(() => proxy.deposit(5000)).toThrow(/অ্যাক্সেস নেই/);
    });

    test('Teller withdraw করতে পারবে না', () => {
        const proxy = createProtectedAccount(account, { name: 'ক্যাশিয়ার', role: 'teller' });

        expect(() => proxy.withdraw(1000)).toThrow();
    });
});

describe('Rate Limiter', () => {
    test('সীমার মধ্যে রিকোয়েস্ট অনুমোদিত হবে', () => {
        const limiter = new RateLimiter(5, 60_000);

        for (let i = 0; i < 5; i++) {
            expect(limiter.tryAcquire().allowed).toBe(true);
        }
    });

    test('সীমা অতিক্রম করলে অস্বীকৃত হবে', () => {
        const limiter = new RateLimiter(3, 60_000);

        limiter.tryAcquire();
        limiter.tryAcquire();
        limiter.tryAcquire();

        const result = limiter.tryAcquire();
        expect(result.allowed).toBe(false);
        expect(result.retryAfterMs).toBeGreaterThan(0);
    });
});

describe('Proxy Chain', () => {
    test('সকল স্তর সঠিকভাবে কাজ করবে', () => {
        const service = new UserService();
        const logs = [];

        const proxied = new ProxyChainBuilder(service)
            .addLogging({ log: (msg) => logs.push(msg) })
            .addAccessControl(['getUser'])
            .build();

        expect(proxied.getUser(1)).toEqual({ id: 1, name: 'রহিম' });
        expect(logs.length).toBeGreaterThan(0);
    });

    test('নিষিদ্ধ মেথডে Error হবে', () => {
        const proxied = new ProxyChainBuilder(new UserService())
            .addAccessControl(['getUser'])
            .build();

        expect(() => proxied.deleteUser(1)).toThrow(/অ্যাক্সেস অস্বীকৃত/);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Adapter
- **উদ্দেশ্য:** ভিন্ন ইন্টারফেসকে সামঞ্জস্যপূর্ণ করা
- **সম্পর্ক:** Adapter ইন্টারফেস **পরিবর্তন** করে, Proxy **একই** ইন্টারফেস রাখে
- **উদাহরণ:** পুরনো Payment Gateway API-কে নতুন ইন্টারফেসে adapt করা

### Decorator
- **উদ্দেশ্য:** অবজেক্টে **নতুন আচরণ/ক্ষমতা** যোগ করা
- **সম্পর্ক:** Decorator ফিচার যোগ করে, Proxy অ্যাক্সেস **নিয়ন্ত্রণ** করে
- **উদাহরণ:** একটি Logger-এ Timestamp Decorator যোগ করা

### Facade
- **উদ্দেশ্য:** জটিল সাবসিস্টেমের জন্য **সরলীকৃত ইন্টারফেস**
- **সম্পর্ক:** Facade একাধিক অবজেক্টকে একত্রিত করে, Proxy একটি নির্দিষ্ট অবজেক্টের প্রতিনিধি
- **উদাহরণ:** Laravel-এর Cache Facade

### Flyweight
- **সম্পর্ক:** Virtual Proxy-র সাথে একত্রে ব্যবহার করা যায় — Proxy lazy loading করে, Flyweight শেয়ারিং করে

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **ভারী অবজেক্ট Lazy Load করতে হবে** — ডেটাবেস কানেকশন, বড় ইমেজ, ফাইল সিস্টেম
2. **Access Control দরকার** — Role-based permission, authentication middleware
3. **রিমোট রিসোর্স অ্যাক্সেস** — REST API, gRPC, মাইক্রোসার্ভিস যোগাযোগ
4. **ক্যাশিং প্রয়োজন** — ব্যয়বহুল অপারেশনের ফলাফল সংরক্ষণ
5. **Audit/Logging দরকার** — কে কি করলো ট্র্যাক করা
6. **Rate Limiting** — API abuse রোধ করা
7. **Connection Pooling** — ডেটাবেস কানেকশন সীমিত রাখা

### ❌ ব্যবহার করবেন না যখন:

1. **সাধারণ অবজেক্ট** — যেখানে কোনো অতিরিক্ত লজিক দরকার নেই
2. **Performance Critical Path** — প্রতিটি মাইক্রোসেকেন্ড গুরুত্বপূর্ণ এমন জায়গায়
3. **অল্প কোড** — Proxy যোগ করলে কোডবেস অকারণে জটিল হবে
4. **সঠিক Abstraction না থাকলে** — ইন্টারফেস ছাড়া Proxy ঠিকভাবে কাজ করে না
5. **শুধু একটি মেথড কল** — সাধারণ wrapper ফাংশনই যথেষ্ট

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট:

```
মূল অবজেক্টে সরাসরি অ্যাক্সেস দেবেন?
├─ হ্যাঁ → Proxy দরকার নেই
└─ না → কেন নয়?
    ├─ ভারী/ব্যয়বহুল → Virtual Proxy ✅
    ├─ সিকিউরিটি → Protection Proxy ✅
    ├─ রিমোট → Remote Proxy ✅
    ├─ পুনরাবৃত্তি রোধ → Caching Proxy ✅
    ├─ ট্র্যাকিং → Logging Proxy ✅
    └─ রিসোর্স ম্যানেজমেন্ট → Smart Reference Proxy ✅
```

---

## 📋 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────────┐
│                  Proxy প্যাটার্ন সারসংক্ষেপ                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  🎯 মূল কথা: মূল অবজেক্টের একটি প্রতিনিধি যা             │
│     অ্যাক্সেস নিয়ন্ত্রণ করে                                │
│                                                           │
│  📦 ৬ ধরনের Proxy:                                        │
│     ① Virtual  → Lazy loading                            │
│     ② Protection → Access control                        │
│     ③ Remote   → Network abstraction                     │
│     ④ Caching  → ফলাফল সংরক্ষণ                           │
│     ⑤ Logging  → অডিট ট্রেইল                             │
│     ⑥ Smart Ref → রিসোর্স ম্যানেজমেন্ট                    │
│                                                           │
│  🔑 মনে রাখবেন:                                           │
│     • Proxy = একই Interface                              │
│     • Client জানে না সে Proxy ব্যবহার করছে               │
│     • Cross-cutting concerns এর জন্য আদর্শ               │
│     • JS-এ ES6 Proxy অবজেক্ট অত্যন্ত শক্তিশালী           │
│     • PHP-তে Magic Methods (__get/__set/__call)           │
│       Proxy-র মতো কাজ করে                                │
│     • Laravel Facades, Eloquent Lazy Loading              │
│       সবই Proxy প্যাটার্নের বাস্তব প্রয়োগ                 │
│                                                           │
│  ⚖️ Proxy ≠ Decorator ≠ Adapter                          │
│     Proxy → অ্যাক্সেস নিয়ন্ত্রণ                            │
│     Decorator → নতুন আচরণ যোগ                            │
│     Adapter → ইন্টারফেস রূপান্তর                           │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Proxy প্যাটার্ন** সফটওয়্যার ইঞ্জিনিয়ারিং-এর অন্যতম বহুল ব্যবহৃত প্যাটার্ন। ওয়েব ডেভেলপমেন্ট থেকে সিস্টেম ডিজাইন — সর্বত্র এর প্রয়োগ দেখা যায়। মূল কথা একটাই: **মূল অবজেক্টের সামনে একটি বুদ্ধিমান গেটকিপার বসিয়ে দিন।** 🛡️
