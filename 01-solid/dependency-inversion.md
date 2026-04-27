# D — Dependency Inversion Principle (DIP)

> **"উচ্চ-স্তরের মডিউল নিম্ন-স্তরের মডিউলের উপর নির্ভরশীল হবে না। উভয়ই অ্যাবস্ট্রাকশনের উপর নির্ভরশীল হবে।"**
> **"অ্যাবস্ট্রাকশন ডিটেইলসের উপর নির্ভরশীল হবে না। ডিটেইলস অ্যাবস্ট্রাকশনের উপর নির্ভরশীল হবে।"**
> — Robert C. Martin

---

## 📖 সংজ্ঞা

Dependency Inversion Principle এর দুটি অংশ:

1. **High-level modules** (ব্যবসায়িক লজিক) **low-level modules** (ডাটাবেস, ফাইল সিস্টেম, API) এর উপর সরাসরি নির্ভরশীল হবে না
2. দুটোই **abstractions** (interface/abstract class) এর উপর নির্ভরশীল হবে

সহজ ভাষায়: **কংক্রিট ক্লাসের বদলে ইন্টারফেসের উপর নির্ভর করুন।**

### "Inversion" কেন?

সাধারণত dependency flow হয়:
```
High-level → Low-level (উপর থেকে নিচে)
```

DIP এই flow "উল্টে" দেয়:
```
High-level → Abstraction ← Low-level
(দুটোই abstraction এর দিকে তাকায়)
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### বৈদ্যুতিক যন্ত্রের উদাহরণ

আপনার ল্যাপটপ চিন্তা করুন:
- ল্যাপটপ (high-level) সরাসরি দেয়ালের তারে (low-level) যুক্ত না
- মাঝে একটি **চার্জার/অ্যাডাপ্টার** (abstraction) আছে
- ল্যাপটপ অ্যাডাপ্টারের কথা জানে, দেয়ালের ভোল্টেজ জানে না
- দেশ বদলালে শুধু অ্যাডাপ্টার বদলান — ল্যাপটপ একই থাকে

**অ্যাডাপ্টার = Abstraction (Interface)**
**দেয়ালের সকেট = Low-level detail**
**ল্যাপটপ = High-level module**

---

## ❌ খারাপ উদাহরণ — DIP ভঙ্গ

### PHP (❌ ভুল)

```php
<?php

// ❌ Low-level module — MySQL ডাটাবেস
class MySQLDatabase
{
    private PDO $connection;

    public function __construct()
    {
        $this->connection = new PDO(
            'mysql:host=localhost;dbname=shop',
            'root',
            'password'
        );
    }

    public function query(string $sql, array $params = []): array
    {
        $stmt = $this->connection->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function insert(string $table, array $data): int
    {
        $columns = implode(',', array_keys($data));
        $placeholders = implode(',', array_fill(0, count($data), '?'));
        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})";

        $stmt = $this->connection->prepare($sql);
        $stmt->execute(array_values($data));
        return (int) $this->connection->lastInsertId();
    }
}

// ❌ Low-level module — Mailtrap ইমেইল
class MailtrapEmailer
{
    public function send(string $to, string $subject, string $body): void
    {
        // Mailtrap API কল
        $ch = curl_init('https://send.api.mailtrap.io/api/send');
        // ...
    }
}

// ❌ High-level module — সরাসরি low-level এর উপর নির্ভরশীল
class OrderService
{
    private MySQLDatabase $db;          // ❌ কংক্রিট ক্লাস
    private MailtrapEmailer $emailer;   // ❌ কংক্রিট ক্লাস

    public function __construct()
    {
        // ❌ new দিয়ে নিজেই তৈরি করছে — tight coupling
        $this->db = new MySQLDatabase();
        $this->emailer = new MailtrapEmailer();
    }

    public function placeOrder(array $orderData): int
    {
        // ❌ MySQL এর নির্দিষ্ট মেথড ব্যবহার
        $orderId = $this->db->insert('orders', $orderData);

        // ❌ Mailtrap এর নির্দিষ্ট ক্লাস ব্যবহার
        $this->emailer->send(
            $orderData['email'],
            "অর্ডার কনফার্ম",
            "আপনার অর্ডার #{$orderId} সফল!"
        );

        return $orderId;
    }
}

/*
 * সমস্যাগুলো:
 * 1. MySQL → PostgreSQL বদলাতে হলে OrderService মডিফাই করতে হবে
 * 2. Mailtrap → SendGrid বদলাতে হলে OrderService মডিফাই করতে হবে
 * 3. ইউনিট টেস্ট করতে হলে সত্যিকারের DB ও Email দরকার
 * 4. OrderService এর ব্যবসায়িক লজিক infrastructure এর সাথে গুলিয়ে গেছে
 */
```

### JavaScript (❌ ভুল)

```javascript
// ❌ Low-level module
class MongoDBDatabase {
    async connect() {
        this.client = await MongoClient.connect('mongodb://localhost:27017');
        this.db = this.client.db('shop');
    }

    async find(collection, query) {
        return this.db.collection(collection).find(query).toArray();
    }

    async insertOne(collection, document) {
        const result = await this.db.collection(collection).insertOne(document);
        return result.insertedId;
    }
}

// ❌ High-level module — সরাসরি MongoDB এর উপর নির্ভরশীল
class ProductService {
    constructor() {
        // ❌ new দিয়ে নিজেই তৈরি — tight coupling
        this.db = new MongoDBDatabase();
        this.stripe = new StripePayment('sk_test_...');
    }

    async createProduct(productData) {
        await this.db.connect();
        // ❌ MongoDB এর নির্দিষ্ট মেথড ব্যবহার
        return this.db.insertOne('products', productData);
    }

    async chargeCustomer(amount) {
        // ❌ Stripe এর নির্দিষ্ট API ব্যবহার
        return this.stripe.charges.create({ amount, currency: 'bdt' });
    }
}
```

---

## ✅ ভালো উদাহরণ — DIP মেনে

### PHP (✅ সঠিক)

```php
<?php

// ধাপ ১: Abstraction (ইন্টারফেস) তৈরি
interface DatabaseInterface
{
    public function find(string $table, array $conditions): array;
    public function findOne(string $table, array $conditions): ?array;
    public function insert(string $table, array $data): int;
    public function update(string $table, array $conditions, array $data): int;
    public function delete(string $table, array $conditions): int;
}

interface EmailServiceInterface
{
    public function send(string $to, string $subject, string $body): bool;
}

interface PaymentGatewayInterface
{
    public function charge(float $amount, string $currency, array $metadata): PaymentResult;
    public function refund(string $transactionId): PaymentResult;
}

class PaymentResult
{
    public function __construct(
        public readonly bool $success,
        public readonly string $transactionId,
        public readonly string $message
    ) {}
}

// ধাপ ২: Low-level implementations — ইন্টারফেস ইমপ্লিমেন্ট করে
class MySQLDatabase implements DatabaseInterface
{
    public function __construct(private readonly PDO $pdo) {}

    public function find(string $table, array $conditions): array
    {
        $where = $this->buildWhereClause($conditions);
        $sql = "SELECT * FROM {$table} {$where}";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute(array_values($conditions));
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function findOne(string $table, array $conditions): ?array
    {
        $results = $this->find($table, $conditions);
        return $results[0] ?? null;
    }

    public function insert(string $table, array $data): int
    {
        $columns = implode(',', array_keys($data));
        $placeholders = implode(',', array_fill(0, count($data), '?'));
        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute(array_values($data));
        return (int) $this->pdo->lastInsertId();
    }

    public function update(string $table, array $conditions, array $data): int
    {
        // ... ইমপ্লিমেন্টেশন
        return 0;
    }

    public function delete(string $table, array $conditions): int
    {
        // ... ইমপ্লিমেন্টেশন
        return 0;
    }

    private function buildWhereClause(array $conditions): string
    {
        if (empty($conditions)) return '';
        $parts = array_map(fn($key) => "{$key} = ?", array_keys($conditions));
        return 'WHERE ' . implode(' AND ', $parts);
    }
}

class PostgreSQLDatabase implements DatabaseInterface
{
    public function __construct(private readonly PDO $pdo) {}

    // ... একই ইন্টারফেস, ভিন্ন ইমপ্লিমেন্টেশন
    public function find(string $table, array $conditions): array { /* ... */ return []; }
    public function findOne(string $table, array $conditions): ?array { return null; }
    public function insert(string $table, array $data): int { return 0; }
    public function update(string $table, array $conditions, array $data): int { return 0; }
    public function delete(string $table, array $conditions): int { return 0; }
}

class SendGridEmailService implements EmailServiceInterface
{
    public function __construct(private readonly string $apiKey) {}

    public function send(string $to, string $subject, string $body): bool
    {
        // SendGrid API কল
        return true;
    }
}

class MailgunEmailService implements EmailServiceInterface
{
    public function __construct(
        private readonly string $domain,
        private readonly string $apiKey
    ) {}

    public function send(string $to, string $subject, string $body): bool
    {
        // Mailgun API কল
        return true;
    }
}

class BkashPaymentGateway implements PaymentGatewayInterface
{
    public function charge(float $amount, string $currency, array $metadata): PaymentResult
    {
        // বিকাশ API কল
        return new PaymentResult(true, 'bkash_txn_123', 'বিকাশ পেমেন্ট সফল');
    }

    public function refund(string $transactionId): PaymentResult
    {
        return new PaymentResult(true, $transactionId, 'রিফান্ড সফল');
    }
}

class StripePaymentGateway implements PaymentGatewayInterface
{
    public function __construct(private readonly string $secretKey) {}

    public function charge(float $amount, string $currency, array $metadata): PaymentResult
    {
        // Stripe API কল
        return new PaymentResult(true, 'stripe_pi_123', 'Stripe পেমেন্ট সফল');
    }

    public function refund(string $transactionId): PaymentResult
    {
        return new PaymentResult(true, $transactionId, 'রিফান্ড সফল');
    }
}

// ধাপ ৩: High-level module — শুধু ইন্টারফেসের উপর নির্ভরশীল
class OrderService
{
    // ✅ কংক্রিট ক্লাস নয়, ইন্টারফেস!
    public function __construct(
        private readonly DatabaseInterface $database,
        private readonly EmailServiceInterface $emailService,
        private readonly PaymentGatewayInterface $paymentGateway
    ) {}

    public function placeOrder(array $orderData): array
    {
        // ✅ ইন্টারফেস মেথড ব্যবহার — কোন DB, কোন Email সেটা জানে না / জানতে হয় না
        $paymentResult = $this->paymentGateway->charge(
            $orderData['total'],
            'BDT',
            ['order' => $orderData['id'] ?? 'new']
        );

        if (!$paymentResult->success) {
            return ['success' => false, 'message' => 'পেমেন্ট ব্যর্থ'];
        }

        $orderId = $this->database->insert('orders', [
            'customer_email' => $orderData['email'],
            'total' => $orderData['total'],
            'transaction_id' => $paymentResult->transactionId,
            'status' => 'confirmed'
        ]);

        $this->emailService->send(
            $orderData['email'],
            "অর্ডার কনফার্ম #ORD-{$orderId}",
            "আপনার অর্ডার সফলভাবে গ্রহণ করা হয়েছে। ট্রানজ্যাকশন: {$paymentResult->transactionId}"
        );

        return [
            'success' => true,
            'order_id' => $orderId,
            'transaction_id' => $paymentResult->transactionId
        ];
    }

    public function cancelOrder(int $orderId): array
    {
        $order = $this->database->findOne('orders', ['id' => $orderId]);

        if (!$order) {
            return ['success' => false, 'message' => 'অর্ডার পাওয়া যায়নি'];
        }

        $refundResult = $this->paymentGateway->refund($order['transaction_id']);

        $this->database->update('orders', ['id' => $orderId], ['status' => 'cancelled']);

        $this->emailService->send(
            $order['customer_email'],
            "অর্ডার বাতিল #ORD-{$orderId}",
            "আপনার অর্ডার বাতিল হয়েছে। রিফান্ড প্রক্রিয়াধীন।"
        );

        return ['success' => true, 'message' => 'অর্ডার বাতিল হয়েছে'];
    }
}

// ✅ ব্যবহার — dependency inject করা হচ্ছে
$orderService = new OrderService(
    database: new MySQLDatabase(new PDO('mysql:host=localhost;dbname=shop', 'root', '')),
    emailService: new SendGridEmailService('sg_api_key_123'),
    paymentGateway: new BkashPaymentGateway()
);

$result = $orderService->placeOrder([
    'email' => 'rahim@example.com',
    'total' => 1500.00
]);

// ✅ PostgreSQL + Mailgun + Stripe এ বদলাতে হলে:
// OrderService কোডে কোনো পরিবর্তন নেই!
$orderServiceV2 = new OrderService(
    database: new PostgreSQLDatabase(new PDO('pgsql:host=localhost;dbname=shop', 'user', 'pass')),
    emailService: new MailgunEmailService('mg.example.com', 'mg_api_key_123'),
    paymentGateway: new StripePaymentGateway('sk_test_stripe_key')
);
```

### JavaScript (✅ সঠিক)

```javascript
// ✅ Abstraction — ইন্টারফেস হিসেবে base class / duck typing

// ধাপ ১: Low-level implementations
class MongoDBRepository {
    #db;

    constructor(database) {
        this.#db = database;
    }

    async find(collection, query = {}) {
        return this.#db.collection(collection).find(query).toArray();
    }

    async findOne(collection, query) {
        return this.#db.collection(collection).findOne(query);
    }

    async insert(collection, document) {
        const result = await this.#db.collection(collection).insertOne(document);
        return result.insertedId.toString();
    }

    async update(collection, query, data) {
        await this.#db.collection(collection).updateOne(query, { $set: data });
    }

    async delete(collection, query) {
        await this.#db.collection(collection).deleteOne(query);
    }
}

class InMemoryRepository {
    #store = new Map();

    async find(collection, query = {}) {
        const items = this.#store.get(collection) || [];
        return items.filter(item => {
            return Object.entries(query).every(([key, val]) => item[key] === val);
        });
    }

    async findOne(collection, query) {
        const results = await this.find(collection, query);
        return results[0] || null;
    }

    async insert(collection, document) {
        if (!this.#store.has(collection)) {
            this.#store.set(collection, []);
        }
        const id = `${Date.now()}_${Math.random().toString(36).slice(2)}`;
        const doc = { ...document, _id: id };
        this.#store.get(collection).push(doc);
        return id;
    }

    async update(collection, query, data) {
        const items = this.#store.get(collection) || [];
        const index = items.findIndex(item =>
            Object.entries(query).every(([key, val]) => item[key] === val)
        );
        if (index >= 0) {
            Object.assign(items[index], data);
        }
    }

    async delete(collection, query) {
        const items = this.#store.get(collection) || [];
        const index = items.findIndex(item =>
            Object.entries(query).every(([key, val]) => item[key] === val)
        );
        if (index >= 0) items.splice(index, 1);
    }
}

class ConsoleLogger {
    info(message) { console.log(`[INFO] ${message}`); }
    error(message) { console.error(`[ERROR] ${message}`); }
    warn(message) { console.warn(`[WARN] ${message}`); }
}

class FileLogger {
    #fs;
    #path;

    constructor(fs, path) {
        this.#fs = fs;
        this.#path = path;
    }

    info(message) {
        this.#fs.appendFileSync(this.#path, `[INFO] ${new Date().toISOString()} ${message}\n`);
    }

    error(message) {
        this.#fs.appendFileSync(this.#path, `[ERROR] ${new Date().toISOString()} ${message}\n`);
    }

    warn(message) {
        this.#fs.appendFileSync(this.#path, `[WARN] ${new Date().toISOString()} ${message}\n`);
    }
}

// ধাপ ২: High-level module — শুধু abstraction এর উপর নির্ভরশীল
class UserService {
    #repository;
    #logger;
    #emailService;

    // ✅ Constructor Injection — dependency বাইরে থেকে আসে
    constructor({ repository, logger, emailService }) {
        this.#repository = repository;
        this.#logger = logger;
        this.#emailService = emailService;
    }

    async createUser(userData) {
        this.#logger.info(`ইউজার তৈরি হচ্ছে: ${userData.email}`);

        const existingUser = await this.#repository.findOne('users', { email: userData.email });
        if (existingUser) {
            this.#logger.warn(`ইমেইল ইতিমধ্যে আছে: ${userData.email}`);
            throw new Error('এই ইমেইল ইতিমধ্যে ব্যবহৃত');
        }

        const userId = await this.#repository.insert('users', {
            ...userData,
            createdAt: new Date().toISOString()
        });

        await this.#emailService.send(
            userData.email,
            "স্বাগতম!",
            `প্রিয় ${userData.name}, আপনার অ্যাকাউন্ট তৈরি হয়েছে।`
        );

        this.#logger.info(`ইউজার তৈরি সফল: ${userId}`);
        return userId;
    }

    async getUser(userId) {
        return this.#repository.findOne('users', { _id: userId });
    }

    async listUsers() {
        return this.#repository.find('users');
    }
}

// ✅ প্রোডাকশন ব্যবহার
const productionUserService = new UserService({
    repository: new MongoDBRepository(mongoDb),
    logger: new FileLogger(fs, '/var/log/app.log'),
    emailService: new SendGridService('api_key')
});

// ✅ টেস্টিং ব্যবহার — কোনো বাহ্যিক সার্ভিস দরকার নেই!
const testUserService = new UserService({
    repository: new InMemoryRepository(),
    logger: new ConsoleLogger(),
    emailService: {
        async send(to, subject, body) {
            console.log(`[MOCK EMAIL] To: ${to}, Subject: ${subject}`);
        }
    }
});

// একই UserService কোড — ভিন্ন dependency!
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### Dependency Injection এর ৩টি পদ্ধতি

#### ১. Constructor Injection (সবচেয়ে সাধারণ ও সুপারিশকৃত)

```php
<?php

class NotificationService
{
    // ✅ Constructor দিয়ে dependency নেওয়া
    public function __construct(
        private readonly EmailServiceInterface $email,
        private readonly SMSServiceInterface $sms
    ) {}

    public function notify(string $userId, string $message): void
    {
        $this->email->send($userId, "বিজ্ঞপ্তি", $message);
        $this->sms->send($userId, $message);
    }
}
```

#### ২. Method Injection

```php
<?php

class ReportGenerator
{
    // ✅ মেথড প্যারামিটারে dependency
    // যখন dependency প্রতি কলে ভিন্ন হতে পারে
    public function generate(DatabaseInterface $db, string $reportType): string
    {
        $data = $db->find('reports', ['type' => $reportType]);
        return json_encode($data);
    }
}
```

#### ৩. Setter Injection

```php
<?php

class CacheService
{
    private ?LoggerInterface $logger = null;

    // ✅ Setter দিয়ে optional dependency
    public function setLogger(LoggerInterface $logger): void
    {
        $this->logger = $logger;
    }

    public function get(string $key): mixed
    {
        $this->logger?->info("Cache get: {$key}");
        // ...
        return null;
    }
}
```

### DI Container — বাস্তব প্রজেক্টে

```php
<?php

// সিম্পল DI Container
class Container
{
    private array $bindings = [];
    private array $instances = [];

    // ইন্টারফেস → কংক্রিট ক্লাস ম্যাপিং
    public function bind(string $abstract, callable $factory): void
    {
        $this->bindings[$abstract] = $factory;
    }

    // Singleton — একবারই তৈরি হবে
    public function singleton(string $abstract, callable $factory): void
    {
        $this->bindings[$abstract] = function () use ($abstract, $factory) {
            if (!isset($this->instances[$abstract])) {
                $this->instances[$abstract] = $factory($this);
            }
            return $this->instances[$abstract];
        };
    }

    // Resolve — ইন্টারফেস থেকে কংক্রিট ইনস্ট্যান্স পাওয়া
    public function make(string $abstract): object
    {
        if (isset($this->bindings[$abstract])) {
            return ($this->bindings[$abstract])($this);
        }
        throw new RuntimeException("{$abstract} বাইন্ড করা হয়নি");
    }
}

// ✅ কনফিগারেশন — একটি জায়গায় সব binding
$container = new Container();

$container->singleton(DatabaseInterface::class, function (Container $c) {
    return new MySQLDatabase(
        new PDO('mysql:host=localhost;dbname=shop', 'root', '')
    );
});

$container->singleton(EmailServiceInterface::class, function (Container $c) {
    return new SendGridEmailService(getenv('SENDGRID_API_KEY'));
});

$container->bind(PaymentGatewayInterface::class, function (Container $c) {
    return new BkashPaymentGateway();
});

$container->bind(OrderService::class, function (Container $c) {
    return new OrderService(
        $c->make(DatabaseInterface::class),
        $c->make(EmailServiceInterface::class),
        $c->make(PaymentGatewayInterface::class)
    );
});

// ✅ ব্যবহার
$orderService = $container->make(OrderService::class);

// ✅ পরিবেশ বদলাতে হলে শুধু binding বদলান!
// টেস্ট পরিবেশ:
$testContainer = new Container();
$testContainer->singleton(DatabaseInterface::class, fn() => new InMemoryDatabase());
$testContainer->singleton(EmailServiceInterface::class, fn() => new FakeEmailService());
// ...
```

```javascript
// JavaScript Simple DI Container
class DIContainer {
    #bindings = new Map();
    #singletons = new Map();

    register(name, factory, singleton = false) {
        this.#bindings.set(name, { factory, singleton });
    }

    resolve(name) {
        const binding = this.#bindings.get(name);
        if (!binding) throw new Error(`${name} রেজিস্টার করা হয়নি`);

        if (binding.singleton) {
            if (!this.#singletons.has(name)) {
                this.#singletons.set(name, binding.factory(this));
            }
            return this.#singletons.get(name);
        }

        return binding.factory(this);
    }
}

// ✅ কনফিগারেশন
const container = new DIContainer();

container.register('database', () => new MongoDBRepository(mongoDb), true);
container.register('logger', () => new FileLogger(fs, './app.log'), true);
container.register('emailService', () => new SendGridService(process.env.SENDGRID_KEY), true);

container.register('userService', (c) => new UserService({
    repository: c.resolve('database'),
    logger: c.resolve('logger'),
    emailService: c.resolve('emailService')
}));

// ✅ ব্যবহার
const userService = container.resolve('userService');
```

### Dependency Inversion ≠ Dependency Injection

```
⚠️ একটি গুরুত্বপূর্ণ পার্থক্য:

Dependency Inversion Principle (DIP):
  → একটি ডিজাইন নীতি
  → "কংক্রিটের বদলে abstraction এর উপর নির্ভর করুন"

Dependency Injection (DI):
  → একটি টেকনিক/প্যাটার্ন
  → "dependency বাইরে থেকে inject করুন"

DI হলো DIP অর্জনের একটি উপায়!

DIP = কী করতে হবে (WHAT)
DI  = কিভাবে করতে হবে (HOW)
```

---

## 🧪 টেস্টিং সুবিধা

DIP মানলে ইউনিট টেস্ট অত্যন্ত সহজ:

### PHP (PHPUnit)

```php
<?php

class OrderServiceTest extends TestCase
{
    private OrderService $orderService;

    // ✅ মক দিয়ে dependency replace — কোনো সত্যিকারের DB/Email দরকার নেই
    protected function setUp(): void
    {
        $mockDb = $this->createMock(DatabaseInterface::class);
        $mockDb->method('insert')->willReturn(42);
        $mockDb->method('findOne')->willReturn([
            'id' => 42,
            'customer_email' => 'test@test.com',
            'transaction_id' => 'txn_123'
        ]);

        $mockEmail = $this->createMock(EmailServiceInterface::class);
        $mockEmail->method('send')->willReturn(true);

        $mockPayment = $this->createMock(PaymentGatewayInterface::class);
        $mockPayment->method('charge')->willReturn(
            new PaymentResult(true, 'txn_123', 'সফল')
        );

        $this->orderService = new OrderService($mockDb, $mockEmail, $mockPayment);
    }

    public function test_place_order_successfully(): void
    {
        $result = $this->orderService->placeOrder([
            'email' => 'rahim@test.com',
            'total' => 1500.00
        ]);

        $this->assertTrue($result['success']);
        $this->assertEquals(42, $result['order_id']);
    }

    public function test_place_order_payment_failure(): void
    {
        // নতুন মক — পেমেন্ট ফেল
        $mockPayment = $this->createMock(PaymentGatewayInterface::class);
        $mockPayment->method('charge')->willReturn(
            new PaymentResult(false, '', 'ব্যালেন্স অপর্যাপ্ত')
        );

        $service = new OrderService(
            $this->createMock(DatabaseInterface::class),
            $this->createMock(EmailServiceInterface::class),
            $mockPayment
        );

        $result = $service->placeOrder([
            'email' => 'rahim@test.com',
            'total' => 1500.00
        ]);

        $this->assertFalse($result['success']);
    }
}
```

### JavaScript (Jest)

```javascript
describe('UserService', () => {
    let userService;
    let mockRepository;
    let mockLogger;
    let mockEmailService;

    beforeEach(() => {
        // ✅ মক dependency — সত্যিকারের DB/Email দরকার নেই
        mockRepository = {
            find: jest.fn().mockResolvedValue([]),
            findOne: jest.fn().mockResolvedValue(null),
            insert: jest.fn().mockResolvedValue('user_123'),
            update: jest.fn(),
            delete: jest.fn()
        };

        mockLogger = {
            info: jest.fn(),
            error: jest.fn(),
            warn: jest.fn()
        };

        mockEmailService = {
            send: jest.fn().mockResolvedValue(true)
        };

        userService = new UserService({
            repository: mockRepository,
            logger: mockLogger,
            emailService: mockEmailService
        });
    });

    test('নতুন ইউজার তৈরি করতে পারে', async () => {
        const userId = await userService.createUser({
            name: 'রহিম',
            email: 'rahim@test.com'
        });

        expect(userId).toBe('user_123');
        expect(mockRepository.insert).toHaveBeenCalledWith('users', expect.objectContaining({
            name: 'রহিম',
            email: 'rahim@test.com'
        }));
        expect(mockEmailService.send).toHaveBeenCalled();
    });

    test('ডুপ্লিকেট ইমেইলে ইউজার তৈরি হবে না', async () => {
        mockRepository.findOne.mockResolvedValue({ email: 'rahim@test.com' });

        await expect(
            userService.createUser({ name: 'রহিম', email: 'rahim@test.com' })
        ).rejects.toThrow('এই ইমেইল ইতিমধ্যে ব্যবহৃত');

        expect(mockRepository.insert).not.toHaveBeenCalled();
    });
});
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **Loose Coupling** | হাই-লেভেল কোড লো-লেভেল ডিটেইল জানে না |
| ২ | **সহজ টেস্টিং** | মক/ফেক দিয়ে dependency replace করা যায় |
| ৩ | **সহজ পরিবর্তন** | MySQL → PostgreSQL বদলাতে শুধু binding বদলান |
| ৪ | **পুনর্ব্যবহারযোগ্য** | হাই-লেভেল মডিউল বিভিন্ন লো-লেভেল দিয়ে কাজ করে |
| ৫ | **সমান্তরাল উন্নয়ন** | ইন্টারফেস ঠিক হলে টিম আলাদাভাবে কাজ করতে পারে |
| ৬ | **প্লাগেবল আর্কিটেকচার** | নতুন implementation "প্লাগ ইন" করা যায় |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **জটিলতা বাড়ে** | ইন্টারফেস, DI Container, বাইন্ডিং — বেশি কোড |
| ২ | **লার্নিং কার্ভ** | DI, IoC Container বুঝতে সময় লাগে |
| ৩ | **ওভার-ইঞ্জিনিয়ারিং** | ছোট প্রজেক্টে DI অপ্রয়োজনীয় জটিলতা আনতে পারে |
| ৪ | **ডিবাগিং কঠিন** | Container থেকে কোন implementation আসছে তা ট্র্যাক করা কঠিন |
| ৫ | **রানটাইম এরর** | কম্পাইল টাইমে ধরা পড়ে না, বাইন্ডিং ভুল হলে রানটাইমে ক্র্যাশ |

---

## 🚫 সাধারণ ভুল

### ভুল ১: new দিয়ে dependency তৈরি করা

```php
<?php

// ❌ ক্লাসের ভিতরে new — tight coupling
class UserController
{
    public function store()
    {
        $service = new UserService();      // ❌
        $mailer = new MailtrapMailer();     // ❌
    }
}

// ✅ বাইরে থেকে inject করুন
class UserController
{
    public function __construct(
        private readonly UserService $service  // ✅
    ) {}
}
```

### ভুল ২: সব কিছুতে DIP করা

```php
<?php

// ❌ OverKill — built-in/stable class এ DIP দরকার নেই
interface DateTimeInterface { /* ... */ }
interface StringInterface { /* ... */ }

// ✅ DIP দরকার:
// - ডাটাবেস, ইমেইল, পেমেন্ট (external dependency)
// - ফাইল সিস্টেম, API কল (I/O)
// - বদলাতে পারে এমন business rule

// ✅ DIP দরকার নেই:
// - DateTime, String, Array (stable, built-in)
// - সিম্পল ভ্যালু অবজেক্ট
// - ইউটিলিটি ক্লাস যা বদলায় না
```

---

## 📏 কখন DIP প্রয়োগ করবেন

### ✅ করবেন যখন:

- **বাহ্যিক সার্ভিস** (DB, Email, Payment, API) ব্যবহার করছেন
- কোড **ইউনিট টেস্ট** করতে চান
- **implementation বদলানোর** সম্ভাবনা আছে
- **টিমে** কাজ হচ্ছে — ইন্টারফেস ঠিক হলে সবাই আলাদা কাজ করতে পারে
- **প্লাগিন/মডিউলার** সিস্টেম তৈরি করছেন

### ❌ করবেন না যখন:

- প্রজেক্ট **খুব ছোট** (স্ক্রিপ্ট, CLI টুল)
- dependency **কখনো বদলাবে না** (built-in class)
- **শুধু একটি implementation** আছে এবং থাকবে
- **প্রোটোটাইপ** বানাচ্ছেন

---

## 🔗 অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক

```
DIP ──► OCP: abstraction দিয়ে নতুন implementation যোগ সহজ
DIP ──► LSP: সব implementation ইন্টারফেসের contract মানতে হবে
DIP ──► ISP: ছোট ইন্টারফেস = সহজে inject করা যায়
DIP ──► SRP: dependency আলাদা করলে ক্লাসের দায়িত্ব পরিষ্কার হয়
```

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **মূল ধারণা** | কংক্রিটের বদলে abstraction এর উপর নির্ভর করুন |
| **কৌশল** | Constructor Injection, DI Container |
| **চেক করুন** | "ক্লাসে কি `new` দিয়ে dependency তৈরি হচ্ছে?" |
| **সাবধান** | সব কিছুতে DIP করবেন না — stable dependency তে দরকার নেই |
| **মনে রাখুন** | DIP ≠ DI। DIP হলো নীতি, DI হলো টেকনিক |
