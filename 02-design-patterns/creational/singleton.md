# 🔒 Singleton প্যাটার্ন

## 📌 সংজ্ঞা ও মূল ধারণা

### GoF সংজ্ঞা

> **"Ensure a class only has one instance, and provide a global point of access to it."**
> — *Gang of Four (GoF), Design Patterns: Elements of Reusable Object-Oriented Software, 1994*

Singleton হলো একটি **Creational Design Pattern** যা নিশ্চিত করে যে একটি class-এর শুধুমাত্র **একটি মাত্র instance** তৈরি হবে এবং সেই instance-টিতে অ্যাক্সেস করার জন্য একটি **global access point** প্রদান করবে।

### মূল ধারণা

Singleton প্যাটার্নের তিনটি মৌলিক বৈশিষ্ট্য রয়েছে:

1. **একক Instance নিশ্চিতকরণ**: পুরো application lifecycle-এ একটি class-এর শুধুমাত্র একটি object থাকবে।
2. **Global Access Point**: যেকোনো জায়গা থেকে ঐ instance-টিকে অ্যাক্সেস করা যাবে, সাধারণত একটি static method-এর মাধ্যমে।
3. **Private Constructor**: বাইরে থেকে `new` keyword দিয়ে object তৈরি করা সম্পূর্ণভাবে বন্ধ রাখা হয়।

### Classification

| বৈশিষ্ট্য | মান |
|---|---|
| **ক্যাটেগরি** | Creational Pattern |
| **স্কোপ** | Object |
| **জটিলতা** | ⭐ (Low) |
| **জনপ্রিয়তা** | ⭐⭐⭐ (High) |
| **GoF প্যাটার্ন** | হ্যাঁ |

### কখন ঠিক একটি Instance দরকার?

নিচের পরিস্থিতিগুলোতে Singleton ব্যবহারের প্রয়োজন হয়:

- **Shared Resource Management**: যখন একটি resource (database connection, file handle) পুরো application জুড়ে শেয়ার করতে হয়।
- **Coordination Point**: যখন একটি central point দরকার যা বিভিন্ন component-এর মধ্যে সমন্বয় করবে।
- **Expensive Object Creation**: যখন object তৈরি করা resource-intensive এবং বারবার তৈরি করা অপচয়।
- **Consistent State**: যখন পুরো application-এ একই state maintain করতে হয় (যেমন: configuration settings)।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### বাংলাদেশ সরকার — প্রধানমন্ত্রীর কার্যালয়

বাংলাদেশে একটি সময়ে শুধুমাত্র **একজন প্রধানমন্ত্রী** থাকেন। প্রধানমন্ত্রীর কার্যালয় (PMO) হলো একটি চমৎকার Singleton-এর উদাহরণ:

- **একক Instance**: দেশে একটি সময়ে শুধুমাত্র একটি PMO সক্রিয় থাকে।
- **Global Access**: দেশের যেকোনো প্রান্ত থেকে, যেকোনো মন্ত্রণালয় বা বিভাগ PMO-তে যোগাযোগ করতে পারে।
- **Private Constructor**: যেকেউ ইচ্ছেমতো নিজেকে প্রধানমন্ত্রী ঘোষণা করতে পারে না — নির্দিষ্ট প্রক্রিয়ার (সংবিধান অনুযায়ী) মাধ্যমেই এটি নির্ধারিত হয়।
- **Controlled Access**: PMO-তে access নির্দিষ্ট channel-এর মাধ্যমে হয়, সরাসরি "new PMO()" তৈরি করা সম্ভব না।

```
আপনি (Client)
    │
    ├── "নতুন PMO তৈরি করো" ──→ ❌ (Private Constructor)
    │
    └── "চলমান PMO-তে Access দাও" ──→ ✅ PMO::getInstance()
                                            │
                                            ▼
                                    [একমাত্র PMO Instance]
```

### আরও বাস্তব উদাহরণ

| বাস্তব জীবন | সফটওয়্যার সমতুল্য |
|---|---|
| বাংলাদেশ ব্যাংক (কেন্দ্রীয় ব্যাংক — একটিই) | Database Connection Pool |
| bKash-এর Central Transaction Ledger | Logger Service |
| জাতীয় পরিচয়পত্র (NID) সার্ভার | Configuration Manager |
| ঢাকা WASA-র কেন্দ্রীয় পানি সরবরাহ ব্যবস্থা | Cache Manager |

---

## 📊 UML ডায়াগ্রাম

### Class Diagram (ASCII)

```
┌──────────────────────────────────────┐
│            «Singleton»               │
│           Singleton                  │
├──────────────────────────────────────┤
│ - static instance: Singleton | null  │
│ - data: mixed                        │
├──────────────────────────────────────┤
│ - __construct()                      │
│ - __clone()                          │
│ - __wakeup()                         │
│ + static getInstance(): Singleton    │
│ + getData(): mixed                   │
│ + setData(value: mixed): void        │
└──────────────────────────────────────┘
         ▲
         │ «uses»
         │
┌────────┴─────────┐
│     Client        │
├──────────────────┤
│ + doWork(): void │
│   // uses        │
│   // Singleton:: │
│   // getInstance │
└──────────────────┘
```

### Sequence Diagram (ASCII)

```
Client A          Singleton           Client B
   │                  │                  │
   │  getInstance()   │                  │
   │─────────────────>│                  │
   │                  │                  │
   │  [instance null?]│                  │
   │  [হ্যাঁ → নতুন  │                  │
   │   instance তৈরি]│                  │
   │                  │                  │
   │  return instance │                  │
   │<─────────────────│                  │
   │                  │                  │
   │                  │  getInstance()   │
   │                  │<─────────────────│
   │                  │                  │
   │                  │  [instance null?]│
   │                  │  [না → আগের     │
   │                  │   instance ফেরত] │
   │                  │                  │
   │                  │  return instance │
   │                  │─────────────────>│
   │                  │                  │
   ▼                  ▼                  ▼
   
   [দুটি client-ই একই instance ব্যবহার করছে]
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1. Basic Singleton

#### PHP 8.3

```php
<?php

declare(strict_types=1);

final class Singleton
{
    private static ?self $instance = null;

    // PHP 8.3: readonly properties দিয়ে immutable state
    private function __construct(
        private readonly string $id,
        private readonly float $createdAt,
    ) {}

    // Cloning বন্ধ
    private function __clone(): void {}

    // Unserialization বন্ধ — এটি না করলে serialize/unserialize দিয়ে
    // নতুন instance তৈরি করা সম্ভব হয়ে যায়
    public function __wakeup(): never
    {
        throw new \RuntimeException('Singleton কে unserialize করা যাবে না।');
    }

    public static function getInstance(): static
    {
        if (static::$instance === null) {
            static::$instance = new static(
                id: uniqid('singleton_', true),
                createdAt: microtime(true),
            );
        }

        return static::$instance;
    }

    public function getId(): string
    {
        return $this->id;
    }

    public function getCreatedAt(): float
    {
        return $this->createdAt;
    }
}

// ব্যবহার
$a = Singleton::getInstance();
$b = Singleton::getInstance();

var_dump($a === $b); // true — দুটি reference একই object-কে নির্দেশ করে

// $c = new Singleton(); // ❌ Fatal Error: private constructor
// $d = clone $a;        // ❌ Fatal Error: private __clone
```

#### JavaScript (ES2022+)

```javascript
class Singleton {
  // ES2022: Private static field
  static #instance = null;

  // Private fields — বাইরে থেকে অ্যাক্সেসযোগ্য নয়
  #id;
  #createdAt;

  constructor() {
    // Constructor-কে সরাসরি কল করা ঠেকানোর কৌশল
    if (Singleton.#instance) {
      throw new Error('Singleton কে new দিয়ে তৈরি করা যাবে না। getInstance() ব্যবহার করুন।');
    }

    this.#id = crypto.randomUUID();
    this.#createdAt = Date.now();

    // Freeze করে দিচ্ছি যাতে কেউ property যোগ/পরিবর্তন করতে না পারে
    Object.freeze(this);
  }

  static getInstance() {
    if (!Singleton.#instance) {
      Singleton.#instance = new Singleton();
    }
    return Singleton.#instance;
  }

  get id() {
    return this.#id;
  }

  get createdAt() {
    return this.#createdAt;
  }
}

// ব্যবহার
const a = Singleton.getInstance();
const b = Singleton.getInstance();

console.log(a === b); // true
console.log(a.id === b.id); // true

// const c = new Singleton(); // ❌ Error: getInstance() ব্যবহার করুন
```

---

### 2. Thread-Safe Singleton

#### PHP 8.3 (Fiber এবং সমান্তরাল পরিবেশের জন্য)

```php
<?php

declare(strict_types=1);

/**
 * PHP মূলত single-threaded, তবে PHP 8.1+ এ Fibers এবং
 * ext-parallel extension ব্যবহারে race condition হতে পারে।
 * এই implementation সেই সমস্যাকে handle করে।
 */
final class ThreadSafeSingleton
{
    private static ?self $instance = null;
    private static bool $initializing = false;

    private function __construct(
        private readonly string $connectionId,
        private array $state = [],
    ) {}

    private function __clone(): void {}

    public function __wakeup(): never
    {
        throw new \RuntimeException('Singleton unserialize নিষিদ্ধ।');
    }

    public static function getInstance(): static
    {
        // Double-checked locking analog for PHP
        // ext-parallel ব্যবহার করলে Mutex ব্যবহার করা উচিত
        if (static::$instance === null) {
            if (static::$initializing) {
                throw new \RuntimeException(
                    'Singleton initialization চলাকালীন recursive call ধরা পড়েছে।'
                );
            }

            static::$initializing = true;

            try {
                static::$instance = new static(
                    connectionId: bin2hex(random_bytes(16)),
                );
            } finally {
                static::$initializing = false;
            }
        }

        return static::$instance;
    }

    public function getState(string $key, mixed $default = null): mixed
    {
        return $this->state[$key] ?? $default;
    }

    public function setState(string $key, mixed $value): void
    {
        $this->state[$key] = $value;
    }

    /**
     * শুধুমাত্র টেস্টিং-এর জন্য — production code-এ ব্যবহার করবেন না
     * @internal
     */
    public static function resetInstance(): void
    {
        static::$instance = null;
    }
}
```

#### JavaScript (ES2022+ — Async-Safe)

```javascript
/**
 * JavaScript single-threaded (Event Loop), কিন্তু async initialization-এ
 * race condition হতে পারে যদি getInstance() async কাজ করে।
 * নিচের implementation সেই সমস্যা সমাধান করে।
 */
class AsyncSafeSingleton {
  static #instance = null;
  static #initPromise = null;

  #config;
  #ready = false;

  constructor() {
    if (AsyncSafeSingleton.#instance) {
      throw new Error('getInstance() ব্যবহার করুন।');
    }
  }

  /**
   * Async initialization-এ race condition এড়াতে Promise caching করা হচ্ছে।
   * প্রথম কলে Promise তৈরি হবে, পরবর্তী কলগুলো সেই একই Promise পাবে।
   */
  static async getInstance() {
    if (AsyncSafeSingleton.#instance) {
      return AsyncSafeSingleton.#instance;
    }

    // Promise caching: একাধিক concurrent কল একই promise share করবে
    if (!AsyncSafeSingleton.#initPromise) {
      AsyncSafeSingleton.#initPromise = (async () => {
        const instance = new AsyncSafeSingleton();

        // ধরুন, config একটি async source থেকে আসছে
        instance.#config = await AsyncSafeSingleton.#loadConfig();
        instance.#ready = true;

        AsyncSafeSingleton.#instance = instance;
        return instance;
      })();
    }

    return AsyncSafeSingleton.#initPromise;
  }

  static async #loadConfig() {
    // Simulate: bKash configuration API থেকে config load
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          apiEndpoint: 'https://api.bkash.com/v2',
          timeout: 30000,
          retryAttempts: 3,
        });
      }, 100);
    });
  }

  get config() {
    if (!this.#ready) {
      throw new Error('Singleton এখনো initialize হয়নি।');
    }
    return structuredClone(this.#config); // Deep copy ফেরত দিচ্ছে
  }
}

// ব্যবহার — concurrent access test
const [inst1, inst2, inst3] = await Promise.all([
  AsyncSafeSingleton.getInstance(),
  AsyncSafeSingleton.getInstance(),
  AsyncSafeSingleton.getInstance(),
]);

console.log(inst1 === inst2); // true
console.log(inst2 === inst3); // true
```

---

### 3. Lazy vs Eager Initialization

#### Lazy Initialization (PHP)

```php
<?php

declare(strict_types=1);

/**
 * Lazy Initialization: Instance তখনই তৈরি হবে যখন প্রথমবার দরকার পড়বে।
 * সুবিধা: যদি কখনো ব্যবহার না হয়, তাহলে memory নষ্ট হবে না।
 * অসুবিধা: প্রথম কলে সামান্য latency হতে পারে।
 */
final class LazyLogger
{
    private static ?self $instance = null;
    private readonly \SplFileObject $fileHandle;

    private function __construct(
        private readonly string $logPath,
    ) {
        // Expensive operation: File handle তৈরি হচ্ছে শুধুমাত্র প্রথমবার
        $this->fileHandle = new \SplFileObject($this->logPath, 'a');
    }

    private function __clone(): void {}

    public static function getInstance(string $logPath = '/var/log/app.log'): static
    {
        // Instance তখনই তৈরি হবে যখন প্রথমবার getInstance() কল হবে
        return static::$instance ??= new static($logPath);
    }

    public function log(string $level, string $message, array $context = []): void
    {
        $entry = sprintf(
            "[%s] [%s] %s %s\n",
            date('Y-m-d H:i:s.u'),
            strtoupper($level),
            $message,
            $context ? json_encode($context, JSON_UNESCAPED_UNICODE) : '',
        );

        $this->fileHandle->fwrite($entry);
    }
}
```

#### Eager Initialization (PHP)

```php
<?php

declare(strict_types=1);

/**
 * Eager Initialization: Class load হওয়ার সাথে সাথে instance তৈরি হয়ে যায়।
 * সুবিধা: Thread-safe (multi-threaded পরিবেশে নিরাপদ), কোনো race condition নেই।
 * অসুবিধা: ব্যবহার না হলেও memory ব্যবহার করে।
 *
 * দ্রষ্টব্য: PHP-তে true eager initialization কম দেখা যায় কারণ PHP
 * মূলত request-per-process মডেলে কাজ করে। তবে long-running
 * processes (Swoole, RoadRunner, ReactPHP) এ এটি গুরুত্বপূর্ণ।
 */
final class EagerConfig
{
    // Class load হওয়ার সময়ই instance তৈরি
    // PHP 8.3-তে constant expression-এ new ব্যবহার করা যায় না,
    // তাই একটি কাছাকাছি পন্থা ব্যবহার করা হয়:
    private static self $instance;
    private static bool $initialized = false;

    private function __construct(
        private readonly array $settings,
    ) {}

    private function __clone(): void {}

    /**
     * Application bootstrap-এ কল করতে হবে — index.php বা bootstrap.php-তে।
     */
    public static function initialize(array $settings): void
    {
        if (static::$initialized) {
            throw new \LogicException('EagerConfig ইতোমধ্যেই initialize হয়ে গেছে।');
        }

        static::$instance = new static($settings);
        static::$initialized = true;
    }

    public static function getInstance(): static
    {
        if (!static::$initialized) {
            throw new \LogicException(
                'EagerConfig initialize হয়নি। আগে EagerConfig::initialize() কল করুন।'
            );
        }

        return static::$instance;
    }

    public function get(string $key, mixed $default = null): mixed
    {
        return $this->settings[$key] ?? $default;
    }
}

// bootstrap.php এ:
EagerConfig::initialize([
    'app_name' => 'bKash Merchant Portal',
    'db_host' => 'db.bkash.internal',
    'cache_ttl' => 3600,
]);

// পরে যেকোনো জায়গায়:
$config = EagerConfig::getInstance();
echo $config->get('app_name'); // "bKash Merchant Portal"
```

#### Lazy vs Eager তুলনা (JavaScript)

```javascript
// ──────── Eager Initialization ────────
// Module load হওয়ার সাথে সাথে instance তৈরি
class EagerSingleton {
  static #instance = new EagerSingleton(); // Eager: সাথে সাথে তৈরি

  #startTime = Date.now();

  constructor() {
    if (EagerSingleton.#instance) {
      throw new Error('getInstance() ব্যবহার করুন।');
    }
  }

  static getInstance() {
    return EagerSingleton.#instance;
  }

  get uptime() {
    return Date.now() - this.#startTime;
  }
}

// ──────── Lazy Initialization ────────
// প্রথম ব্যবহারের সময় instance তৈরি
class LazySingleton {
  static #instance = null;

  #data = new Map();

  constructor() {
    if (LazySingleton.#instance) {
      throw new Error('getInstance() ব্যবহার করুন।');
    }
  }

  static getInstance() {
    // Lazy: প্রথম কলেই শুধু তৈরি হবে
    LazySingleton.#instance ??= new LazySingleton();
    return LazySingleton.#instance;
  }

  set(key, value) {
    this.#data.set(key, value);
    return this;
  }

  get(key) {
    return this.#data.get(key);
  }
}
```

| বৈশিষ্ট্য | Lazy | Eager |
|---|---|---|
| **কখন তৈরি হয়** | প্রথম `getInstance()` কলে | Class load/module import-এ |
| **Memory ব্যবহার** | চাহিদা অনুযায়ী | আগে থেকেই |
| **প্রথম কলে Latency** | হ্যাঁ (initialization cost) | না |
| **Thread Safety** | অতিরিক্ত ব্যবস্থা দরকার | স্বাভাবিকভাবে নিরাপদ |
| **উপযুক্ত** | Resource-intensive object | হালকা object, সবসময় দরকার |

---

### 4. Singleton with Enum (PHP 8.1+)

```php
<?php

declare(strict_types=1);

/**
 * PHP 8.1+ এ Enum দিয়ে Singleton বানানো একটি elegant পদ্ধতি।
 * Java-তে এটি Joshua Bloch কর্তৃক "Effective Java"-তে
 * সবচেয়ে ভালো Singleton পদ্ধতি হিসেবে পরিচিত।
 *
 * সুবিধা:
 * - serialize/unserialize safe (PHP enum নিজেই এটি handle করে)
 * - Reflection দিয়ে নতুন instance তৈরি করা অসম্ভব
 * - Clone safe
 * - সংক্ষিপ্ত ও পরিষ্কার কোড
 *
 * অসুবিধা:
 * - Enum extend করা যায় না
 * - Constructor parameter নেওয়া সীমিত
 */
enum DatabaseConnection
{
    case INSTANCE;

    private const DSN = 'mysql:host=db.bkash.internal;dbname=transactions;charset=utf8mb4';
    private const USERNAME = 'app_user';
    private const PASSWORD = 'secret'; // বাস্তবে environment variable ব্যবহার করুন

    /**
     * PDO connection cache — static property enum-এ allowed
     */
    private static ?\PDO $pdo = null;

    public function getConnection(): \PDO
    {
        if (self::$pdo === null) {
            self::$pdo = new \PDO(
                dsn: self::DSN,
                username: self::USERNAME,
                password: self::PASSWORD,
                options: [
                    \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
                    \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
                    \PDO::ATTR_EMULATE_PREPARES => false,
                    \PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES 'utf8mb4'",
                ],
            );
        }

        return self::$pdo;
    }

    public function query(string $sql, array $params = []): array
    {
        $stmt = $this->getConnection()->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll();
    }

    /**
     * Transaction helper
     */
    public function transaction(\Closure $callback): mixed
    {
        $pdo = $this->getConnection();
        $pdo->beginTransaction();

        try {
            $result = $callback($pdo);
            $pdo->commit();
            return $result;
        } catch (\Throwable $e) {
            $pdo->rollBack();
            throw $e;
        }
    }
}

// ব্যবহার — অসাধারণ সংক্ষিপ্ত syntax
$users = DatabaseConnection::INSTANCE->query(
    'SELECT * FROM users WHERE district = :district',
    ['district' => 'ঢাকা'],
);

// Transaction
DatabaseConnection::INSTANCE->transaction(function (\PDO $pdo) {
    $pdo->prepare('UPDATE wallets SET balance = balance - 500 WHERE user_id = ?')
        ->execute([101]);
    $pdo->prepare('UPDATE wallets SET balance = balance + 500 WHERE user_id = ?')
        ->execute([202]);
});

// Reflection attack? ❌ অসম্ভব!
// new DatabaseConnection(); // Fatal error
// new ReflectionClass(DatabaseConnection::class)->newInstance(); // Error
```

---

### 5. Multiton Pattern (Singleton-এর Variation)

Multiton Pattern হলো Singleton-এর একটি সম্প্রসারণ যেখানে **নির্দিষ্ট key অনুযায়ী একটি instance** maintain করা হয়। একটি key-এর জন্য শুধুমাত্র একটিই instance থাকবে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

/**
 * Multiton: Named singleton instances
 * ব্যবহারের ক্ষেত্র: Multiple database connection (read/write replica),
 * multiple cache store (Redis, Memcached), multi-tenant application
 */
final class DatabasePool
{
    /** @var array<string, self> */
    private static array $instances = [];

    private readonly \PDO $pdo;

    private function __construct(
        private readonly string $name,
        private readonly string $dsn,
        string $username,
        string $password,
    ) {
        $this->pdo = new \PDO($dsn, $username, $password, [
            \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
            \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
        ]);
    }

    private function __clone(): void {}

    /**
     * নির্দিষ্ট নামে connection instance পাওয়া যাবে।
     * একই নামে দ্বিতীয়বার কল করলে আগের instance ফিরে আসবে।
     */
    public static function getInstance(
        string $name,
        ?string $dsn = null,
        ?string $username = null,
        ?string $password = null,
    ): static {
        if (!isset(static::$instances[$name])) {
            if ($dsn === null) {
                throw new \InvalidArgumentException(
                    "'{$name}' connection আগে register হয়নি। DSN প্রদান করুন।"
                );
            }

            static::$instances[$name] = new static($name, $dsn, $username ?? '', $password ?? '');
        }

        return static::$instances[$name];
    }

    public static function has(string $name): bool
    {
        return isset(static::$instances[$name]);
    }

    public function getConnection(): \PDO
    {
        return $this->pdo;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public static function closeAll(): void
    {
        static::$instances = [];
    }
}

// ব্যবহার
$writeDb = DatabasePool::getInstance(
    name: 'write',
    dsn: 'mysql:host=primary.db.bkash.internal;dbname=transactions',
    username: 'write_user',
    password: 'secret',
);

$readDb = DatabasePool::getInstance(
    name: 'read',
    dsn: 'mysql:host=replica.db.bkash.internal;dbname=transactions',
    username: 'read_user',
    password: 'secret',
);

// পরবর্তীতে যেকোনো জায়গায় — শুধু name দিলেই হবে
$db = DatabasePool::getInstance('write');
var_dump($db === $writeDb); // true
```

#### JavaScript (ES2022+)

```javascript
class CacheStore {
  static #instances = new Map();

  #name;
  #store = new Map();
  #ttlMap = new Map();

  constructor(name) {
    if (CacheStore.#instances.has(name)) {
      throw new Error(`'${name}' store ইতোমধ্যে বিদ্যমান। getInstance() ব্যবহার করুন।`);
    }
    this.#name = name;
  }

  static getInstance(name) {
    if (!CacheStore.#instances.has(name)) {
      CacheStore.#instances.set(name, new CacheStore(name));
    }
    return CacheStore.#instances.get(name);
  }

  static has(name) {
    return CacheStore.#instances.has(name);
  }

  set(key, value, ttlMs = 0) {
    this.#store.set(key, value);

    if (ttlMs > 0) {
      // আগের timer থাকলে clear করে দিচ্ছি
      if (this.#ttlMap.has(key)) {
        clearTimeout(this.#ttlMap.get(key));
      }
      this.#ttlMap.set(key, setTimeout(() => this.delete(key), ttlMs));
    }

    return this;
  }

  get(key) {
    return this.#store.get(key);
  }

  delete(key) {
    this.#store.delete(key);
    if (this.#ttlMap.has(key)) {
      clearTimeout(this.#ttlMap.get(key));
      this.#ttlMap.delete(key);
    }
  }

  get name() {
    return this.#name;
  }

  get size() {
    return this.#store.size;
  }
}

// ব্যবহার — একাধিক named singleton
const sessionCache = CacheStore.getInstance('session');
const apiCache = CacheStore.getInstance('api-response');

sessionCache.set('user:101', { name: 'রহিম', district: 'চট্টগ্রাম' }, 3600_000);
apiCache.set('rates:bdt-usd', 110.25, 60_000);

// পরে অন্য module-এ:
const sameSessionCache = CacheStore.getInstance('session');
console.log(sameSessionCache === sessionCache); // true
console.log(sameSessionCache.get('user:101')); // { name: 'রহিম', ... }
```

---

## 🌍 Real-World Applicable Areas

### 1. Database Connection Pool

ডাটাবেস connection তৈরি করা expensive (TCP handshake, authentication, SSL negotiation)। Singleton ব্যবহার করে একটি connection pool manage করা হয় যাতে বারবার নতুন connection তৈরি না করতে হয়।

#### PHP — PDO Singleton

```php
<?php

declare(strict_types=1);

final class DB
{
    private static ?self $instance = null;
    private readonly \PDO $pdo;

    private function __construct()
    {
        $this->pdo = new \PDO(
            dsn: sprintf(
                'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
                $_ENV['DB_HOST'] ?? 'localhost',
                (int) ($_ENV['DB_PORT'] ?? 3306),
                $_ENV['DB_NAME'] ?? 'bkash_db',
            ),
            username: $_ENV['DB_USER'] ?? 'root',
            password: $_ENV['DB_PASS'] ?? '',
            options: [
                \PDO::ATTR_ERRMODE            => \PDO::ERRMODE_EXCEPTION,
                \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
                \PDO::ATTR_EMULATE_PREPARES   => false,
                \PDO::ATTR_PERSISTENT         => true, // Connection pooling
            ],
        );
    }

    private function __clone(): void {}

    public static function getInstance(): static
    {
        return static::$instance ??= new static();
    }

    public function pdo(): \PDO
    {
        return $this->pdo;
    }

    /**
     * Convenience: prepared query
     */
    public function query(string $sql, array $params = []): \PDOStatement
    {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt;
    }

    public function transaction(\Closure $fn): mixed
    {
        $this->pdo->beginTransaction();
        try {
            $result = $fn($this->pdo);
            $this->pdo->commit();
            return $result;
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}

// ব্যবহার
$merchants = DB::getInstance()->query(
    'SELECT * FROM merchants WHERE city = :city AND active = 1',
    ['city' => 'ঢাকা']
)->fetchAll();
```

### 2. Logger Service

#### JS — ConfigManager Singleton

```javascript
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';

class ConfigManager {
  static #instance = null;

  #config = {};
  #loaded = false;
  #env;

  constructor() {
    if (ConfigManager.#instance) {
      throw new Error('getInstance() ব্যবহার করুন।');
    }
    this.#env = process.env.NODE_ENV ?? 'development';
  }

  static async getInstance() {
    if (!ConfigManager.#instance) {
      const instance = new ConfigManager();
      await instance.#load();
      ConfigManager.#instance = instance;
    }
    return ConfigManager.#instance;
  }

  async #load() {
    const configPath = join(process.cwd(), 'config', `${this.#env}.json`);

    try {
      const raw = await readFile(configPath, 'utf-8');
      this.#config = JSON.parse(raw);
      this.#loaded = true;
    } catch (err) {
      // Fallback defaults — bKash staging environment
      this.#config = {
        app: { name: 'bKash Merchant API', version: '3.2.0' },
        db: { host: 'staging-db.bkash.internal', port: 3306 },
        redis: { host: 'staging-redis.bkash.internal', port: 6379 },
        api: { rateLimit: 1000, timeout: 30000 },
      };
      this.#loaded = true;
    }
  }

  get(path, defaultValue = undefined) {
    const keys = path.split('.');
    let result = this.#config;

    for (const key of keys) {
      if (result == null || typeof result !== 'object') {
        return defaultValue;
      }
      result = result[key];
    }

    return result ?? defaultValue;
  }

  get env() {
    return this.#env;
  }

  get isLoaded() {
    return this.#loaded;
  }
}

// ব্যবহার
const config = await ConfigManager.getInstance();
console.log(config.get('db.host'));       // "staging-db.bkash.internal"
console.log(config.get('api.rateLimit')); // 1000
```

### 3. Framework উদাহরণ

#### Laravel — Service Container Singleton Binding

```php
<?php

// AppServiceProvider.php — Laravel-এ singleton binding
use App\Services\PaymentGateway;
use App\Services\BkashPaymentGateway;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // singleton() মেথড নিশ্চিত করে যে container-এ এই class-এর
        // শুধুমাত্র একটি instance থাকবে
        $this->app->singleton(PaymentGateway::class, function ($app) {
            return new BkashPaymentGateway(
                apiKey: config('services.bkash.api_key'),
                secretKey: config('services.bkash.secret_key'),
                baseUrl: config('services.bkash.base_url'),
            );
        });

        // অথবা সংক্ষেপে:
        $this->app->singleton(CacheManager::class);
    }
}

// Controller-এ ব্যবহার — DI এর মাধ্যমে
class PaymentController extends Controller
{
    public function __construct(
        private readonly PaymentGateway $gateway, // Laravel inject করবে singleton
    ) {}

    public function processPayment(Request $request): JsonResponse
    {
        $result = $this->gateway->charge(
            amount: $request->validated('amount'),
            walletNumber: $request->validated('wallet_number'),
        );

        return response()->json($result);
    }
}

// app() helper — global access point (Singleton Facade)
$gateway = app(PaymentGateway::class);
// প্রতিবার একই instance ফেরত আসবে
```

#### Node.js — Module Caching as Natural Singleton

```javascript
// Node.js-এ require()/import স্বাভাবিকভাবেই module cache করে।
// তাই একটি module থেকে export করা object স্বয়ংক্রিয়ভাবে singleton।

// services/logger.mjs
class Logger {
  #logs = [];
  #level;

  constructor(level = 'info') {
    this.#level = level;
    console.log('Logger initialized'); // একবারই দেখাবে
  }

  log(message, meta = {}) {
    const entry = {
      timestamp: new Date().toISOString(),
      level: this.#level,
      message,
      ...meta,
    };
    this.#logs.push(entry);
    console.log(JSON.stringify(entry));
  }

  get entries() {
    return [...this.#logs]; // Copy ফেরত দিচ্ছি, reference নয়
  }
}

// Module-level singleton — import করলেই একই instance পাবে সবাই
const logger = new Logger(process.env.LOG_LEVEL ?? 'info');
export default logger;

// ──── অন্য ফাইলে ব্যবহার ────
// routes/payment.mjs
import logger from '../services/logger.mjs';
logger.log('Payment processed', { amount: 500, currency: 'BDT' });

// routes/user.mjs
import logger from '../services/logger.mjs';
logger.log('User registered', { userId: 101 });

// দুটি ফাইলেই একই logger instance!
```

---

## 🔥 Advanced Deep Dive

### 1. Singleton in Dependency Injection Containers

DI Container-এর singleton binding এবং raw Singleton Pattern-এর মধ্যে একটি গুরুত্বপূর্ণ পার্থক্য আছে:

| দিক | Raw Singleton | DI Container Singleton |
|---|---|---|
| **Instance Management** | Class নিজেই manage করে | Container manage করে |
| **Testability** | কঠিন (global state) | সহজ (mock inject সম্ভব) |
| **Flexibility** | একটি implementation-এ locked | Interface swap সম্ভব |
| **Lifecycle** | Application-wide | Container scope-এ সীমিত |

#### Laravel: গভীর বিশ্লেষণ

```php
<?php

// Laravel-এর Container::singleton() অভ্যন্তরীণভাবে যা করে:
// 1. binding resolve হওয়ার পর instance cache করে
// 2. পরবর্তী resolve-এ cached instance ফেরত দেয়

// Container.php (simplified)
class Container
{
    protected array $instances = [];
    protected array $bindings = [];

    public function singleton(string $abstract, \Closure|string|null $concrete = null): void
    {
        $this->bind($abstract, $concrete, shared: true);
    }

    public function make(string $abstract): mixed
    {
        // ক্যাশে আছে? ফেরত দাও
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }

        $concrete = $this->bindings[$abstract]['concrete'] ?? $abstract;
        $object = $this->build($concrete);

        // Shared (singleton) হলে ক্যাশে রাখো
        if ($this->isShared($abstract)) {
            $this->instances[$abstract] = $object;
        }

        return $object;
    }
}

// টেস্টিং-এ সুবিধা — instance swap করা যায়:
$this->app->instance(PaymentGateway::class, new FakePaymentGateway());
```

### 2. Singleton vs Static Class

অনেক developer ভুলভাবে Singleton এবং Static Class-কে সমার্থক মনে করেন। বাস্তবে এদের মধ্যে গুরুত্বপূর্ণ পার্থক্য আছে:

```php
<?php

// ──── Static Class ────
final class MathHelper
{
    // কোনো instance নেই — সব method static
    public static function calculateVAT(float $amount, float $rate = 0.15): float
    {
        return $amount * $rate;
    }

    public static function roundTaka(float $amount): float
    {
        return round($amount, 2);
    }
}

// ──── Singleton ────
final class TaxCalculator
{
    private static ?self $instance = null;
    private array $taxRates = [];

    private function __construct(
        private readonly string $region,
    ) {
        // expensive: API বা DB থেকে tax rates লোড
        $this->taxRates = $this->loadRatesForRegion($region);
    }

    public static function getInstance(string $region = 'BD'): static
    {
        return static::$instance ??= new static($region);
    }

    public function calculate(float $amount, string $category): float
    {
        $rate = $this->taxRates[$category] ?? 0.15;
        return $amount * $rate;
    }

    private function loadRatesForRegion(string $region): array
    {
        // ধরে নিচ্ছি DB থেকে আসছে
        return match($region) {
            'BD' => ['electronics' => 0.20, 'food' => 0.05, 'services' => 0.15],
            default => ['electronics' => 0.18, 'food' => 0.08, 'services' => 0.12],
        };
    }
}
```

| বৈশিষ্ট্য | Singleton | Static Class |
|---|---|---|
| **State ধরে রাখা** | ✅ হ্যাঁ (instance variable) | ⚠️ শুধু static variable |
| **Interface Implement** | ✅ হ্যাঁ | ❌ না |
| **Polymorphism** | ✅ হ্যাঁ | ❌ না |
| **DI তে ব্যবহার** | ✅ হ্যাঁ | ❌ কঠিন |
| **Lazy Loading** | ✅ নিজে control করা যায় | ❌ প্রযোজ্য নয় |
| **Testability** | ⚠️ সম্ভব (কঠিন) | ❌ খুবই কঠিন |
| **Serialization** | ✅ সম্ভব | ❌ না |
| **Inheritance** | ✅ সম্ভব | ⚠️ সীমিত |
| **উপযুক্ত** | Stateful service, resource management | Stateless utility functions |

### 3. Singleton in Multi-threaded / Async Environments

#### PHP-FPM এবং Singleton

```
┌──────────────────────────────────────────────────┐
│                   Web Server (Nginx)              │
│                                                    │
│  Request A ──→ ┌─────────────────┐                │
│                │  PHP-FPM Worker 1│                │
│                │  Singleton: #1   │ ←── আলাদা     │
│                └─────────────────┘     instance    │
│                                                    │
│  Request B ──→ ┌─────────────────┐                │
│                │  PHP-FPM Worker 2│                │
│                │  Singleton: #2   │ ←── আলাদা     │
│                └─────────────────┘     instance    │
│                                                    │
│  Request C ──→ ┌─────────────────┐                │
│                │  PHP-FPM Worker 3│                │
│                │  Singleton: #3   │ ←── আলাদা     │
│                └─────────────────┘     instance    │
└──────────────────────────────────────────────────┘

গুরুত্বপূর্ণ: PHP-FPM-এ প্রতিটি request আলাদা process-এ চলে।
তাই Singleton শুধু ঐ request-এর lifecycle-এ "single" থাকে।
Request শেষ হলে সব মুছে যায়। Cross-request sharing হয় না।

ব্যতিক্রম: Swoole, RoadRunner, FrankenPHP (long-running workers)
— এদের ক্ষেত্রে Singleton request-এর পরেও বেঁচে থাকে!
```

#### Node.js Event Loop এবং Singleton

```
┌────────────────────────────────────────────────────┐
│                Node.js Process                      │
│                                                      │
│  ┌─────────────────────┐                            │
│  │  Event Loop (Single │                            │
│  │  Threaded)          │                            │
│  │                     │                            │
│  │  Singleton: #1 ◄────┼──── সকল request এই একটি  │
│  │                     │     instance শেয়ার করে    │
│  └─────────────────────┘                            │
│       ▲   ▲   ▲                                     │
│       │   │   │                                     │
│    Req A Req B Req C                                │
│                                                      │
│  ⚠️ সতর্কতা:                                       │
│  - Singleton-এ user-specific data রাখবেন না!        │
│  - একজনের data অন্যজন দেখে ফেলতে পারে            │
│  - Memory leak-এর ঝুঁকি (Singleton কখনো GC হয় না)│
└────────────────────────────────────────────────────┘
```

```javascript
// ❌ বিপজ্জনক — Node.js Singleton-এ user data
class BadAuthSingleton {
  static #instance = null;
  #currentUser = null; // ⚠️ একাধিক request শেয়ার করবে!

  static getInstance() {
    return (BadAuthSingleton.#instance ??= new BadAuthSingleton());
  }

  setUser(user) { this.#currentUser = user; }
  getUser() { return this.#currentUser; }
}

// Request A: BadAuthSingleton.getInstance().setUser({ id: 1, name: 'করিম' });
// Request B: BadAuthSingleton.getInstance().getUser();
// 😱 Request B পেয়ে যাবে করিমের data!

// ✅ সঠিক — Request-scoped context ব্যবহার করুন (AsyncLocalStorage)
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage();

// Middleware
function authMiddleware(req, res, next) {
  const store = { user: req.user, requestId: crypto.randomUUID() };
  requestContext.run(store, () => next());
}

// যেকোনো জায়গায় নিরাপদে access:
function getCurrentUser() {
  return requestContext.getStore()?.user;
}
```

### 4. Singleton Registry

```php
<?php

declare(strict_types=1);

/**
 * Singleton Registry: কেন্দ্রীয়ভাবে singleton instance manage করার pattern।
 * এটি Service Locator pattern-এর সাথে সম্পর্কিত।
 */
final class SingletonRegistry
{
    private static ?self $registry = null;

    /** @var array<class-string, object> */
    private array $instances = [];

    /** @var array<class-string, \Closure> */
    private array $factories = [];

    private function __construct() {}
    private function __clone(): void {}

    public static function getInstance(): static
    {
        return static::$registry ??= new static();
    }

    /**
     * Factory registration — lazy instantiation হবে
     * @template T of object
     * @param class-string<T> $key
     * @param \Closure(): T $factory
     */
    public function register(string $key, \Closure $factory): void
    {
        if (isset($this->instances[$key])) {
            throw new \LogicException("'{$key}' ইতোমধ্যে instantiate হয়ে গেছে। Reset করুন আগে।");
        }
        $this->factories[$key] = $factory;
    }

    /**
     * @template T of object
     * @param class-string<T> $key
     * @return T
     */
    public function get(string $key): object
    {
        if (!isset($this->instances[$key])) {
            if (!isset($this->factories[$key])) {
                throw new \RuntimeException("'{$key}' registered নয়।");
            }
            $this->instances[$key] = ($this->factories[$key])();
        }

        return $this->instances[$key];
    }

    public function has(string $key): bool
    {
        return isset($this->instances[$key]) || isset($this->factories[$key]);
    }

    /**
     * @internal শুধুমাত্র testing-এর জন্য
     */
    public function reset(?string $key = null): void
    {
        if ($key !== null) {
            unset($this->instances[$key]);
        } else {
            $this->instances = [];
        }
    }
}

// ব্যবহার
$registry = SingletonRegistry::getInstance();

$registry->register(Logger::class, fn() => new Logger('/var/log/app.log'));
$registry->register(CacheManager::class, fn() => new CacheManager('redis://cache.internal'));

$logger = $registry->get(Logger::class); // প্রথমবার: তৈরি
$sameLogger = $registry->get(Logger::class); // দ্বিতীয়বার: cached
var_dump($logger === $sameLogger); // true
```

---

## ✅ সুবিধা ও ❌ অসুবিধা

### সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|---|---|
| 1 | **একক Instance গ্যারান্টি** | পুরো application-এ একটি মাত্র instance — data inconsistency এড়ানো যায় |
| 2 | **Global Access Point** | যেকোনো জায়গা থেকে সহজে অ্যাক্সেস — `getInstance()` কল করলেই হয় |
| 3 | **Lazy Initialization** | দরকার না হলে resource খরচ হয় না — performance বাড়ে |
| 4 | **Shared State** | একটি central point-এ state manage করা সম্ভব (config, cache) |
| 5 | **Resource Management** | Expensive resource (DB connection) বারবার তৈরি না করে reuse |

### অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---|---|
| 1 | **Single Responsibility লঙ্ঘন** | Class দুটি কাজ করে — নিজের কাজ + instance management |
| 2 | **Hidden Dependencies** | `getInstance()` কল code-এ dependency গোপন করে — constructor দেখে বোঝা যায় না |
| 3 | **Testing কঠিন** | Global state টেস্টগুলোকে একে অপরের উপর নির্ভরশীল করে তোলে |
| 4 | **Tight Coupling** | Client code সরাসরি concrete class-এর উপর নির্ভর করে |
| 5 | **Concurrency সমস্যা** | Multi-threaded পরিবেশে race condition হতে পারে |
| 6 | **Inheritance জটিলতা** | Singleton subclass করা তত সহজ নয় |

---

## ⚠️ Anti-Pattern হিসেবে Singleton

### কেন অনেকে Singleton-কে Anti-Pattern মনে করেন?

Software engineering সম্প্রদায়ে Singleton সবচেয়ে বিতর্কিত pattern। এর কারণ:

#### 1. Hidden Dependencies সমস্যা

```php
<?php

// ❌ খারাপ — dependency লুকানো আছে
class OrderService
{
    public function createOrder(array $data): Order
    {
        // OrderService-এর constructor-এ কোনো ইঙ্গিত নেই যে
        // এই class DB এবং Logger-এর উপর নির্ভর করে
        $db = DB::getInstance();        // লুকানো dependency
        $logger = Logger::getInstance(); // লুকানো dependency
        $cache = Cache::getInstance();   // লুকানো dependency

        $order = $db->query('INSERT INTO orders...', $data);
        $logger->log('Order created: ' . $order->id);
        $cache->invalidate('orders:*');

        return $order;
    }
}

// ✅ ভালো — Dependency Injection ব্যবহার
class OrderService
{
    public function __construct(
        private readonly DB $db,
        private readonly Logger $logger,
        private readonly Cache $cache,
    ) {} // Constructor দেখেই বোঝা যায় কী dependency দরকার

    public function createOrder(array $data): Order
    {
        $order = $this->db->query('INSERT INTO orders...', $data);
        $this->logger->log('Order created: ' . $order->id);
        $this->cache->invalidate('orders:*');

        return $order;
    }
}
```

#### 2. Testing-এ সমস্যা

```php
<?php

// ❌ Singleton দিয়ে — টেস্ট isolate করা কঠিন
class OrderServiceTest extends TestCase
{
    public function testCreateOrder(): void
    {
        // সমস্যা: DB::getInstance() আসল database-এ যাবে!
        // আমরা mock করতে পারছি না সহজে
        $service = new OrderService();
        $order = $service->createOrder(['item' => 'Phone']);

        // এই টেস্ট: ধীর (আসল DB), fragile, আগের টেস্টের data
        // দ্বারা প্রভাবিত হতে পারে
    }
}

// ✅ DI দিয়ে — সহজে mock করা যায়
class OrderServiceTest extends TestCase
{
    public function testCreateOrder(): void
    {
        $mockDb = $this->createMock(DB::class);
        $mockDb->expects($this->once())
            ->method('query')
            ->willReturn(new Order(id: 1));

        $mockLogger = $this->createMock(Logger::class);
        $mockCache = $this->createMock(Cache::class);

        $service = new OrderService($mockDb, $mockLogger, $mockCache);
        $order = $service->createOrder(['item' => 'Phone']);

        $this->assertEquals(1, $order->id);
    }
}
```

#### 3. প্রশমন কৌশল (Mitigation)

**DI Container ব্যবহার করুন, raw Singleton নয়:**

```php
<?php

// Laravel-এ সঠিক পদ্ধতি:
// 1. Interface define করুন
interface PaymentGatewayInterface
{
    public function charge(float $amount, string $wallet): PaymentResult;
}

// 2. Implementation তৈরি করুন (Singleton নয়!)
class BkashGateway implements PaymentGatewayInterface
{
    public function __construct(
        private readonly HttpClient $http,
        private readonly string $apiKey,
    ) {}

    public function charge(float $amount, string $wallet): PaymentResult
    {
        // bKash API কল
    }
}

// 3. Container-এ singleton হিসেবে bind করুন
// AppServiceProvider.php
$this->app->singleton(
    PaymentGatewayInterface::class,
    fn($app) => new BkashGateway(
        http: $app->make(HttpClient::class),
        apiKey: config('services.bkash.key'),
    ),
);

// 4. Constructor injection ব্যবহার করুন
class CheckoutController
{
    public function __construct(
        private readonly PaymentGatewayInterface $gateway,
    ) {}
}

// ফলাফল: Singleton-এর সুবিধা (একটি instance) + DI-এর সুবিধা (testable, flexible)
```

---

## 🧪 টেস্টিং

### PHPUnit: Singleton টেস্ট করা

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

// Testable Singleton — reset method সহ
final class AppConfig
{
    private static ?self $instance = null;
    private array $data = [];

    private function __construct() {}
    private function __clone(): void {}

    public static function getInstance(): static
    {
        return static::$instance ??= new static();
    }

    public function set(string $key, mixed $value): void
    {
        $this->data[$key] = $value;
    }

    public function get(string $key, mixed $default = null): mixed
    {
        return $this->data[$key] ?? $default;
    }

    public function all(): array
    {
        return $this->data;
    }

    /**
     * ⚠️ শুধুমাত্র testing-এর জন্য!
     * @internal
     */
    public static function resetForTesting(): void
    {
        static::$instance = null;
    }
}

class AppConfigTest extends TestCase
{
    /**
     * প্রতিটি টেস্টের আগে singleton reset করা হচ্ছে
     * যাতে টেস্টগুলো একে অপরকে প্রভাবিত না করে
     */
    protected function setUp(): void
    {
        parent::setUp();
        AppConfig::resetForTesting();
    }

    protected function tearDown(): void
    {
        AppConfig::resetForTesting();
        parent::tearDown();
    }

    public function testGetInstanceReturnsSameObject(): void
    {
        $first = AppConfig::getInstance();
        $second = AppConfig::getInstance();

        $this->assertSame($first, $second);
    }

    public function testSetAndGetValue(): void
    {
        $config = AppConfig::getInstance();
        $config->set('app_name', 'bKash Portal');

        $this->assertSame('bKash Portal', $config->get('app_name'));
    }

    public function testGetReturnsDefaultWhenKeyMissing(): void
    {
        $config = AppConfig::getInstance();

        $this->assertNull($config->get('nonexistent'));
        $this->assertSame('fallback', $config->get('missing', 'fallback'));
    }

    public function testResetClearsInstance(): void
    {
        $first = AppConfig::getInstance();
        $first->set('key', 'value');

        AppConfig::resetForTesting();

        $second = AppConfig::getInstance();

        $this->assertNotSame($first, $second);
        $this->assertNull($second->get('key'));
    }

    /**
     * Reflection দিয়ে private constructor enforce হচ্ছে কিনা যাচাই
     */
    public function testConstructorIsPrivate(): void
    {
        $reflection = new \ReflectionClass(AppConfig::class);
        $constructor = $reflection->getConstructor();

        $this->assertNotNull($constructor);
        $this->assertTrue($constructor->isPrivate());
    }

    /**
     * Clone করা যায় না — এটি verify করা
     */
    public function testCloneIsPrivate(): void
    {
        $reflection = new \ReflectionClass(AppConfig::class);
        $cloneMethod = $reflection->getMethod('__clone');

        $this->assertTrue($cloneMethod->isPrivate());
    }
}
```

### Jest: Singleton টেস্ট করা (JavaScript)

```javascript
// config-manager.mjs
class ConfigManager {
  static #instance = null;

  #data = new Map();

  constructor() {
    if (ConfigManager.#instance) {
      throw new Error('getInstance() ব্যবহার করুন।');
    }
  }

  static getInstance() {
    ConfigManager.#instance ??= new ConfigManager();
    return ConfigManager.#instance;
  }

  set(key, value) {
    this.#data.set(key, value);
  }

  get(key) {
    return this.#data.get(key);
  }

  get size() {
    return this.#data.size;
  }

  /** @internal শুধু testing-এর জন্য */
  static _resetForTesting() {
    ConfigManager.#instance = null;
  }
}

export default ConfigManager;
```

```javascript
// config-manager.test.mjs
import { describe, it, expect, beforeEach, afterEach } from '@jest/globals';
import ConfigManager from './config-manager.mjs';

describe('ConfigManager Singleton', () => {
  beforeEach(() => {
    // প্রতিটি টেস্টের আগে reset
    ConfigManager._resetForTesting();
  });

  afterEach(() => {
    ConfigManager._resetForTesting();
  });

  it('getInstance() সবসময় একই instance ফেরত দেয়', () => {
    const a = ConfigManager.getInstance();
    const b = ConfigManager.getInstance();

    expect(a).toBe(b); // strict reference equality
  });

  it('new দিয়ে তৈরি করলে Error throw করে', () => {
    ConfigManager.getInstance(); // প্রথম instance তৈরি

    expect(() => new ConfigManager()).toThrow('getInstance() ব্যবহার করুন।');
  });

  it('data সঠিকভাবে store ও retrieve করে', () => {
    const config = ConfigManager.getInstance();

    config.set('db.host', 'db.bkash.internal');
    config.set('cache.ttl', 3600);

    expect(config.get('db.host')).toBe('db.bkash.internal');
    expect(config.get('cache.ttl')).toBe(3600);
  });

  it('reset করলে নতুন instance তৈরি হয়', () => {
    const first = ConfigManager.getInstance();
    first.set('key', 'value');

    ConfigManager._resetForTesting();

    const second = ConfigManager.getInstance();

    expect(first).not.toBe(second);
    expect(second.get('key')).toBeUndefined();
    expect(second.size).toBe(0);
  });

  it('shared state সকল reference-এ দেখা যায়', () => {
    const ref1 = ConfigManager.getInstance();
    const ref2 = ConfigManager.getInstance();

    ref1.set('shared', 'data');

    expect(ref2.get('shared')).toBe('data');
  });
});

// ──────────────────────────────────────────────
// Module cache reset করে singleton টেস্ট করা
// (alternative approach — _resetForTesting ছাড়া)
// ──────────────────────────────────────────────

describe('Module-level singleton testing (cache reset)', () => {
  it('jest.resetModules() দিয়ে fresh import', async () => {
    // প্রথম import
    const { default: config1 } = await import('./config-manager.mjs');
    config1._resetForTesting(); // Clean state

    const instance1 = config1.getInstance();
    instance1.set('env', 'test');

    // Module cache clear
    jest.resetModules();

    // নতুন import — সম্পূর্ণ নতুন module instance
    const { default: config2 } = await import('./config-manager.mjs');
    const instance2 = config2.getInstance();

    // নতুন module, তাই আগের data নেই
    expect(instance2.get('env')).toBeUndefined();
  });
});
```

### Singleton Mock করা

```php
<?php

// PHPUnit-এ Singleton mock করার কৌশল — Reflection ব্যবহার করে
class SingletonMockHelper
{
    /**
     * Singleton-এর internal instance-কে mock দিয়ে replace করে।
     * ⚠️ শুধুমাত্র legacy code-এ testing-এর জন্য ব্যবহার করুন!
     *
     * @template T
     * @param class-string<T> $className
     * @param T $mock
     */
    public static function overrideInstance(string $className, object $mock): void
    {
        $reflection = new \ReflectionClass($className);
        $property = $reflection->getProperty('instance');
        $property->setAccessible(true);
        $property->setValue(null, $mock);
    }

    /**
     * @param class-string $className
     */
    public static function resetInstance(string $className): void
    {
        $reflection = new \ReflectionClass($className);
        $property = $reflection->getProperty('instance');
        $property->setAccessible(true);
        $property->setValue(null, null);
    }
}

// ব্যবহার
class LegacyServiceTest extends TestCase
{
    protected function tearDown(): void
    {
        SingletonMockHelper::resetInstance(DB::class);
    }

    public function testWithMockedSingleton(): void
    {
        $mockDb = $this->createMock(DB::class);
        $mockDb->method('query')->willReturn([['id' => 1, 'name' => 'Test']]);

        SingletonMockHelper::overrideInstance(DB::class, $mockDb);

        // এখন DB::getInstance() আমাদের mock ফেরত দেবে
        $result = DB::getInstance()->query('SELECT * FROM users');
        $this->assertCount(1, $result);
    }
}
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Factory Method + Singleton

```php
<?php

// Factory Method singleton instance ফেরত দিতে পারে
abstract class NotificationFactory
{
    abstract public function createChannel(): NotificationChannel;

    // Factory নিজে singleton — একটি factory instance-ই যথেষ্ট
    private static ?self $instance = null;

    public static function getInstance(string $type): static
    {
        return static::$instance ??= match($type) {
            'sms' => new SmsNotificationFactory(),
            'email' => new EmailNotificationFactory(),
            'push' => new PushNotificationFactory(),
            default => throw new \InvalidArgumentException("Unknown type: {$type}"),
        };
    }
}
```

### Abstract Factory (প্রায়ই Singleton)

```php
<?php

// Abstract Factory সাধারণত singleton হিসেবে ব্যবহৃত হয়
// কারণ একটি factory instance-ই সব product তৈরি করতে সক্ষম
abstract class UIWidgetFactory
{
    private static ?self $instance = null;

    public static function getInstance(): static
    {
        return static::$instance ??= match(config('app.theme')) {
            'material' => new MaterialWidgetFactory(),
            'bootstrap' => new BootstrapWidgetFactory(),
            default => new DefaultWidgetFactory(),
        };
    }

    abstract public function createButton(string $label): Button;
    abstract public function createInput(string $placeholder): Input;
    abstract public function createModal(string $title): Modal;
}
```

### Facade (পেছনে Singleton ব্যবহার করে)

```php
<?php

// Laravel Facade অভ্যন্তরীণভাবে singleton resolve করে
// Cache::get('key') আসলে যা করে:

// 1. Cache Facade-এর getFacadeAccessor() return করে 'cache'
// 2. Container থেকে 'cache' key দিয়ে singleton resolve হয়
// 3. ঐ singleton instance-এর get() method কল হয়

// সরলীকৃত রূপ:
class Cache extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'cache'; // Container-এ singleton হিসেবে bound
    }
}

// Cache::get('key') ≈ app('cache')->get('key')
// app('cache') প্রতিবার একই singleton instance ফেরত দেয়
```

### সম্পর্কিত প্যাটার্নের সারসংক্ষেপ

| প্যাটার্ন | Singleton-এর সাথে সম্পর্ক |
|---|---|
| **Factory Method** | Factory নিজে singleton হতে পারে; singleton instance ফেরত দিতে পারে |
| **Abstract Factory** | Factory instance সাধারণত singleton — একটিই যথেষ্ট |
| **Builder** | Complex singleton object তৈরিতে Builder ব্যবহার করা যায় |
| **Facade** | Facade pattern সাধারণত singleton service-এর উপর simplified API প্রদান করে |
| **Flyweight** | Shared instance-এর ধারণা — কিন্তু Flyweight-এ একাধিক shared instance থাকে |
| **State / Strategy** | State বা Strategy object-গুলো stateless হলে singleton হিসেবে শেয়ার করা যায় |
| **Prototype** | Singleton-এর বিপরীত — Prototype object clone করে নতুন instance তৈরি করে |

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | উদাহরণ |
|---|---|
| **একটি Shared Resource** | Database connection pool — সব component একটি pool শেয়ার করবে |
| **Global Configuration** | Application settings — একবার load, সব জায়গায় ব্যবহার |
| **Logging Service** | একটি কেন্দ্রীয় logger যা সব component ব্যবহার করে |
| **Hardware Interface** | Printer spooler, device driver — hardware একটি, instance-ও একটি |
| **Cache Layer** | In-memory cache — একটি cache store সব module-এ শেয়ার |
| **Thread/Connection Pool** | Worker pool management — কেন্দ্রীয়ভাবে pool manage করা |
| **Registry/Service Locator** | Object registry — central point of access |
| **Feature Flag Manager** | Feature toggle — সব জায়গায় consistent flag |

### ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না | বিকল্প |
|---|---|---|
| **শুধু "সুবিধাজনক" global access** | Global variable-এর disguise | DI ব্যবহার করুন |
| **User-specific data** | Multi-user env-এ data leak | Request-scoped service |
| **Stateless utility** | Instance-এর দরকার নেই | Static class/function |
| **Unit test-heavy code** | Test isolation কঠিন হয় | Interface + DI |
| **Microservice architecture** | Service-এ state রাখা বিপজ্জনক | External state store (Redis) |
| **Data Transfer Object** | প্রতিটি DTO আলাদা data বহন করে | Regular class |

### সিদ্ধান্ত গ্রহণ Flowchart (ASCII)

```
একটি class-এর instance কয়টি দরকার?
│
├── একাধিক ──→ ❌ Singleton না
│
└── শুধু একটি
    │
    ├── Global access দরকার?
    │   │
    │   ├── না ──→ ❌ Singleton না (DI ব্যবহার করুন)
    │   │
    │   └── হ্যাঁ
    │       │
    │       ├── DI Container আছে?
    │       │   │
    │       │   ├── হ্যাঁ ──→ ✅ Container singleton binding ব্যবহার করুন
    │       │   │              (Laravel: $this->app->singleton())
    │       │   │
    │       │   └── না ──→ ✅ Singleton Pattern ব্যবহার করুন
    │       │                (তবে testability নিশ্চিত করুন)
    │       │
    │       └── Stateless?
    │           │
    │           ├── হ্যাঁ ──→ ❌ Static class/function ব্যবহার করুন
    │           │
    │           └── না ──→ ✅ Singleton উপযুক্ত হতে পারে
```

---

## 📋 সারসংক্ষেপ টেবিল

| বিষয় | বিবরণ |
|---|---|
| **প্যাটার্নের নাম** | Singleton |
| **ক্যাটেগরি** | Creational |
| **উদ্দেশ্য** | একটি class-এর শুধুমাত্র একটি instance নিশ্চিত করা এবং global access প্রদান |
| **মূল উপাদান** | Private constructor, static instance, static `getInstance()` method |
| **PHP বৈশিষ্ট্য** | `final class`, `private __construct()`, `private __clone()`, `__wakeup(): never` |
| **JS বৈশিষ্ট্য** | Private `#fields`, `static #instance`, `Object.freeze()` |
| **Enum Singleton (PHP)** | সবচেয়ে নিরাপদ — reflection, clone, serialize সব থেকে সুরক্ষিত |
| **Node.js Natural Singleton** | Module caching স্বয়ংক্রিয়ভাবে singleton আচরণ দেয় |
| **Laravel-এ** | `$this->app->singleton()` — DI Container-এ singleton binding |
| **PHP-FPM-এ** | প্রতি request-এ আলাদা singleton — cross-request share হয় না |
| **Node.js-এ** | Process-wide singleton — সকল request শেয়ার করে (সতর্কতা!) |
| **Anti-Pattern কারণ** | Hidden dependency, testing কঠিন, tight coupling |
| **সমাধান** | DI Container ব্যবহার করুন — singleton-এর সুবিধা + testability |
| **সম্পর্কিত প্যাটার্ন** | Factory Method, Abstract Factory, Facade, Flyweight |
| **সেরা পরামর্শ** | Raw Singleton এড়িয়ে DI Container-এ singleton binding করুন |

---

### 🎯 চূড়ান্ত পরামর্শ

> **"Singleton Pattern জানা দরকার, কিন্তু raw Singleton ব্যবহার না করাই উত্তম।"**

আধুনিক software development-এ Singleton Pattern সরাসরি implement করার বদলে DI Container-এর singleton binding ব্যবহার করাই best practice। এতে Singleton-এর সুবিধা (একটি instance, resource efficiency) পাওয়া যায় এবং এর অসুবিধাগুলো (hidden dependency, testing difficulty) এড়ানো যায়।

**মনে রাখুন:**
- 🏗️ **নতুন প্রজেক্ট**: DI Container singleton binding ব্যবহার করুন
- 🔧 **Legacy কোড**: Raw Singleton থাকলে ধীরে ধীরে DI-তে migrate করুন
- 🧪 **Testing**: `resetForTesting()` method রাখুন অথবা DI ব্যবহার করুন
- 📦 **Library/Package**: Singleton এড়িয়ে চলুন — consumer-কে lifecycle control দিন
