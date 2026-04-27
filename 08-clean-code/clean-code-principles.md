# 🧹 Clean Code Principles — ক্লিন কোড নীতিমালা

> _"যেকোনো বোকা এমন কোড লিখতে পারে যা কম্পিউটার বুঝতে পারে। ভালো প্রোগ্রামাররা এমন কোড লেখে যা মানুষ বুঝতে পারে।"_
> — **Martin Fowler**

---

## 📖 সংজ্ঞা

**Clean Code** হলো এমন কোড যা সহজে পড়া যায়, বোঝা যায় এবং পরিবর্তন করা যায়। Robert C. Martin (Uncle Bob) তাঁর বিখ্যাত বই "Clean Code" এ এই নীতিমালাগুলো বিস্তারিতভাবে আলোচনা করেছেন। ক্লিন কোড শুধু কাজ করে না — এটি **পরিষ্কার**, **সুসংগঠিত** এবং **রক্ষণাবেক্ষণযোগ্য**।

ক্লিন কোডের মূল বৈশিষ্ট্যগুলো হলো:
1. **পাঠযোগ্যতা** — কোড পড়লেই বোঝা যায় কী করছে
2. **সরলতা** — অপ্রয়োজনীয় জটিলতা নেই
3. **পরীক্ষাযোগ্যতা** — সহজেই টেস্ট করা যায়
4. **রক্ষণাবেক্ষণযোগ্যতা** — পরিবর্তন করা সহজ

এই গাইডে আমরা ৫টি মূল বিষয় নিয়ে আলোচনা করবো:
- 🏷️ অর্থপূর্ণ নামকরণ (Meaningful Naming)
- ⚙️ ফাংশন ডিজাইন (Function Design)
- 📐 কোড ফরম্যাটিং (Code Formatting)
- 💬 কমেন্ট লেখার নিয়ম (Comments)
- 🚨 এরর হ্যান্ডলিং (Error Handling)

---

## 🏠 বাস্তব জীবনের উদাহরণ

ক্লিন কোডকে একটি **সুসজ্জিত রান্নাঘরের** সাথে তুলনা করা যায়:

- **অগোছালো রান্নাঘর** → মশলা এলোমেলো, বাসন ছড়ানো, কিছু খুঁজে পাওয়া কঠিন
- **সুসজ্জিত রান্নাঘর** → প্রতিটি জিনিস নির্দিষ্ট জায়গায়, লেবেল করা, সহজেই খুঁজে পাওয়া যায়

ঠিক একইভাবে, ক্লিন কোডে প্রতিটি ভেরিয়েবল, ফাংশন এবং ক্লাসের একটি নির্দিষ্ট উদ্দেশ্য থাকে এবং সেটি সহজেই বোঝা যায়।

---

# 🏷️ ১. অর্থপূর্ণ নামকরণ (Meaningful Naming)

> _"নামকরণ হলো প্রোগ্রামিংয়ের সবচেয়ে কঠিন কাজগুলোর মধ্যে একটি।"_

নামকরণ সঠিক হলে কোড নিজেই নিজের ডকুমেন্টেশন হয়ে যায়। ভুল নামকরণ কোডকে দুর্বোধ্য করে তোলে।

**নামকরণের মূলনীতি:**
1. **উদ্দেশ্য প্রকাশ করে** — নাম দেখেই বোঝা যায় কী করে
2. **বিভ্রান্তিকর নয়** — ভুল ধারণা তৈরি করে না
3. **উচ্চারণযোগ্য** — সহজে বলা যায়
4. **খুঁজে পাওয়া যায়** — সার্চ করলে পাওয়া যায়

## ❌ খারাপ উদাহরণ (Naming)

### PHP (❌ ভুল নামকরণ)

```php
<?php
// ❌ খারাপ নামকরণ — কিছুই বোঝা যায় না

class Mgr {
    private $d; // কীসের ডাটা?
    private $lst; // কীসের লিস্ট?

    public function proc($x, $y) {
        // proc মানে কী? process? procedure?
        $t = $x * $y; // t মানে কী?
        $d2 = date('Y-m-d'); // d2 কেন?
        
        if ($t > 1000) {
            $flag = true; // কীসের flag?
        }
        
        return $t;
    }

    public function getData($tp) {
        // tp মানে কী? type? temp?
        $res = []; // res কি result? resource? response?
        
        foreach ($this->lst as $i) {
            if ($i['tp'] == $tp) {
                $res[] = $i;
            }
        }
        
        return $res;
    }

    public function calc($a, $b, $c) {
        // কী calculate করছে?
        $tmp = $a + $b;
        $tmp2 = $tmp * $c;
        return $tmp2 - ($tmp2 * 0.15);
    }
}
```

### JavaScript (❌ ভুল নামকরণ)

```javascript
// ❌ খারাপ নামকরণ — কোড পড়ে কিছু বোঝা যায় না

class Mgr {
    constructor() {
        this.d = [];  // কীসের ডাটা?
        this.n = 0;   // কীসের নম্বর?
    }

    proc(x, y, z) {
        // কী process করছে?
        const t = x * y;
        const r = t - (t * z / 100);
        return r;
    }

    chk(val) {
        // chk মানে check? কী check করছে?
        if (val > 0 && val < 999) {
            return true;
        }
        return false;
    }

    getInfo(tp, dt) {
        // tp? dt? কীসের info?
        const res = this.d.filter(i => i.tp === tp && i.dt > dt);
        return res;
    }
}

// ব্যবহার — কিছুই বোঝা যাচ্ছে না
const m = new Mgr();
const r = m.proc(100, 5, 15);
const f = m.chk(r);
```

## ✅ ভালো উদাহরণ (Naming)

### PHP (✅ সঠিক নামকরণ)

```php
<?php
// ✅ পরিষ্কার নামকরণ — কোড পড়লেই সব বোঝা যায়

class OrderManager {
    private array $orders;
    private array $customerList;

    public function calculateOrderTotal(float $unitPrice, int $quantity): float {
        // নাম দেখেই বোঝা যাচ্ছে — অর্ডারের মোট মূল্য হিসাব করছে
        $subtotal = $unitPrice * $quantity;
        $orderDate = date('Y-m-d');
        
        if ($subtotal > 1000) {
            $isEligibleForDiscount = true;
        }
        
        return $subtotal;
    }

    public function getOrdersByType(string $orderType): array {
        // orderType দিয়ে অর্ডার ফিল্টার করছে — পরিষ্কার
        $filteredOrders = [];
        
        foreach ($this->orders as $order) {
            if ($order['type'] === $orderType) {
                $filteredOrders[] = $order;
            }
        }
        
        return $filteredOrders;
    }

    public function calculateDiscountedPrice(
        float $basePrice,
        float $taxRate,
        float $discountPercentage
    ): float {
        // প্রতিটি প্যারামিটারের নাম অর্থবহ
        $priceWithTax = $basePrice + $taxRate;
        $totalBeforeDiscount = $priceWithTax * $discountPercentage;
        return $totalBeforeDiscount - ($totalBeforeDiscount * 0.15);
    }
}
```

### JavaScript (✅ সঠিক নামকরণ)

```javascript
// ✅ পরিষ্কার নামকরণ — প্রতিটি নাম উদ্দেশ্য প্রকাশ করে

class OrderManager {
    constructor() {
        this.orders = [];
        this.totalOrderCount = 0;
    }

    calculateDiscountedPrice(unitPrice, quantity, discountPercentage) {
        // নাম দেখেই বোঝা যাচ্ছে — ডিসকাউন্টসহ মূল্য হিসাব
        const subtotal = unitPrice * quantity;
        const discountAmount = subtotal * (discountPercentage / 100);
        return subtotal - discountAmount;
    }

    isValidOrderAmount(orderAmount) {
        // boolean ফাংশনে is/has/can দিয়ে শুরু
        const MIN_ORDER_AMOUNT = 1;
        const MAX_ORDER_AMOUNT = 999;
        return orderAmount > MIN_ORDER_AMOUNT && orderAmount < MAX_ORDER_AMOUNT;
    }

    getOrdersByTypeAndDate(orderType, startDate) {
        // প্যারামিটারের নাম দেখেই বোঝা যাচ্ছে কী ফিল্টার হচ্ছে
        const filteredOrders = this.orders.filter(
            order => order.type === orderType && order.createdAt > startDate
        );
        return filteredOrders;
    }
}

// ব্যবহার — কোড পড়লেই সব পরিষ্কার
const orderManager = new OrderManager();
const finalPrice = orderManager.calculateDiscountedPrice(100, 5, 15);
const isValid = orderManager.isValidOrderAmount(finalPrice);
```

### 🔑 নামকরণের মূল পার্থক্য

| ❌ খারাপ নাম | ✅ ভালো নাম | কেন ভালো |
|---|---|---|
| `d`, `d2` | `orders`, `orderDate` | উদ্দেশ্য পরিষ্কার |
| `proc()` | `calculateOrderTotal()` | কী করে বোঝা যায় |
| `tp` | `orderType` | সংক্ষেপে বিভ্রান্তি নেই |
| `flag` | `isEligibleForDiscount` | boolean এর অর্থ পরিষ্কার |
| `tmp`, `tmp2` | `priceWithTax`, `totalBeforeDiscount` | মধ্যবর্তী মানের অর্থ আছে |

---

# ⚙️ ২. ফাংশন ডিজাইন (Function Design)

> _"ফাংশন একটি কাজ করবে, সেটি ভালোভাবে করবে এবং শুধুমাত্র সেটিই করবে।"_
> — **Robert C. Martin**

ভালো ফাংশনের বৈশিষ্ট্য:
1. **ছোট হবে** — ২০ লাইনের বেশি নয়
2. **একটি কাজ করবে** — Single Responsibility
3. **একটি abstraction level** — মিশ্রণ নয়
4. **কম প্যারামিটার** — আদর্শভাবে ০-২টি

## ❌ খারাপ উদাহরণ (Function)

### PHP (❌ বিশাল ফাংশন)

```php
<?php
// ❌ একটি ফাংশনে অনেকগুলো কাজ — পড়া ও রক্ষণাবেক্ষণ কঠিন

class UserService {
    public function registerUser($name, $email, $password, $phone, $address, $city, $country, $role) {
        // ❌ সমস্যা ১: অনেক বেশি প্যারামিটার (৮টি!)
        
        // ❌ সমস্যা ২: ভ্যালিডেশন, সেভ, ইমেইল — সব একসাথে
        if (empty($name) || strlen($name) < 2) {
            throw new Exception('Invalid name');
        }
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new Exception('Invalid email');
        }
        if (strlen($password) < 8) {
            throw new Exception('Password too short');
        }
        if (!preg_match('/[A-Z]/', $password)) {
            throw new Exception('Password needs uppercase');
        }
        if (!preg_match('/[0-9]/', $password)) {
            throw new Exception('Password needs number');
        }
        
        // ডাটাবেইজে চেক
        $db = new PDO('mysql:host=localhost;dbname=app', 'root', '');
        $stmt = $db->prepare('SELECT id FROM users WHERE email = ?');
        $stmt->execute([$email]);
        if ($stmt->fetch()) {
            throw new Exception('Email already exists');
        }
        
        // পাসওয়ার্ড হ্যাশ
        $hashedPassword = password_hash($password, PASSWORD_BCRYPT);
        
        // ইউজার সেভ
        $stmt = $db->prepare('INSERT INTO users (name, email, password, phone, address, city, country, role) VALUES (?, ?, ?, ?, ?, ?, ?, ?)');
        $stmt->execute([$name, $email, $hashedPassword, $phone, $address, $city, $country, $role]);
        $userId = $db->lastInsertId();
        
        // ইমেইল পাঠানো
        $subject = 'Welcome!';
        $message = "Hello $name, welcome to our platform!";
        $headers = "From: noreply@app.com\r\nContent-Type: text/html";
        mail($email, $subject, $message, $headers);
        
        // লগ লেখা
        $logFile = fopen('logs/registration.log', 'a');
        fwrite($logFile, date('Y-m-d H:i:s') . " - User registered: $email\n");
        fclose($logFile);
        
        return $userId;
    }
}
```

### JavaScript (❌ বিশাল ফাংশন)

```javascript
// ❌ একটি ফাংশনে সবকিছু — টেস্ট করা প্রায় অসম্ভব

class UserService {
    async registerUser(name, email, password, phone, address, city, country, role) {
        // ❌ ৮টি প্যারামিটার — মনে রাখা কঠিন
        
        // ভ্যালিডেশন
        if (!name || name.length < 2) throw new Error('Invalid name');
        if (!email.includes('@')) throw new Error('Invalid email');
        if (password.length < 8) throw new Error('Password too short');
        if (!/[A-Z]/.test(password)) throw new Error('Need uppercase');
        if (!/[0-9]/.test(password)) throw new Error('Need number');

        // ডাটাবেইজ চেক
        const existingUser = await db.query('SELECT id FROM users WHERE email = ?', [email]);
        if (existingUser.length > 0) throw new Error('Email exists');

        // পাসওয়ার্ড হ্যাশ ও সেভ
        const hashedPassword = await bcrypt.hash(password, 10);
        const result = await db.query(
            'INSERT INTO users (name, email, password, phone, address, city, country, role) VALUES (?, ?, ?, ?, ?, ?, ?, ?)',
            [name, email, hashedPassword, phone, address, city, country, role]
        );

        // ইমেইল পাঠানো
        await sendEmail({
            to: email,
            subject: 'Welcome!',
            body: `Hello ${name}, welcome!`
        });

        // লগিং
        console.log(`${new Date().toISOString()} - User registered: ${email}`);
        await fs.appendFile('logs/reg.log', `${email} registered\n`);

        return result.insertId;
    }
}
```

## ✅ ভালো উদাহরণ (Function)

### PHP (✅ ছোট, একক দায়িত্বের ফাংশন)

```php
<?php
// ✅ প্রতিটি ফাংশন একটি কাজ করে — পরিষ্কার ও টেস্টযোগ্য

class UserRegistrationData {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
        public readonly string $phone,
        public readonly Address $address,
        public readonly string $role = 'user'
    ) {}
}

class UserValidator {
    public function validate(UserRegistrationData $data): void {
        $this->validateName($data->name);
        $this->validateEmail($data->email);
        $this->validatePassword($data->password);
    }

    private function validateName(string $name): void {
        if (empty($name) || strlen($name) < 2) {
            throw new ValidationException('নাম কমপক্ষে ২ অক্ষরের হতে হবে');
        }
    }

    private function validateEmail(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new ValidationException('সঠিক ইমেইল দিন');
        }
    }

    private function validatePassword(string $password): void {
        if (strlen($password) < 8) {
            throw new ValidationException('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
        }
    }
}

class UserService {
    public function __construct(
        private UserValidator $validator,
        private UserRepository $repository,
        private EmailService $emailService,
        private Logger $logger
    ) {}

    public function register(UserRegistrationData $data): int {
        // ✅ প্রতিটি ধাপ পরিষ্কার — একটি ফাংশন একটি কাজ
        $this->validator->validate($data);
        $this->ensureEmailIsUnique($data->email);
        $userId = $this->repository->save($data);
        $this->emailService->sendWelcomeEmail($data->email, $data->name);
        $this->logger->info("ইউজার রেজিস্ট্রেশন সফল: {$data->email}");
        return $userId;
    }

    private function ensureEmailIsUnique(string $email): void {
        if ($this->repository->existsByEmail($email)) {
            throw new DuplicateEmailException('এই ইমেইল আগে থেকে আছে');
        }
    }
}
```

### JavaScript (✅ ছোট, একক দায়িত্বের ফাংশন)

```javascript
// ✅ প্রতিটি ফাংশন একটি কাজ — সহজে টেস্ট ও রক্ষণাবেক্ষণযোগ্য

class UserRegistrationData {
    constructor({ name, email, password, phone, address, role = 'user' }) {
        this.name = name;
        this.email = email;
        this.password = password;
        this.phone = phone;
        this.address = address;
        this.role = role;
    }
}

class UserValidator {
    validate(data) {
        this.#validateName(data.name);
        this.#validateEmail(data.email);
        this.#validatePassword(data.password);
    }

    #validateName(name) {
        if (!name || name.length < 2) {
            throw new ValidationError('নাম কমপক্ষে ২ অক্ষরের হতে হবে');
        }
    }

    #validateEmail(email) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(email)) {
            throw new ValidationError('সঠিক ইমেইল দিন');
        }
    }

    #validatePassword(password) {
        if (password.length < 8) {
            throw new ValidationError('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
        }
    }
}

class UserService {
    constructor(validator, userRepository, emailService, logger) {
        this.validator = validator;
        this.repository = userRepository;
        this.emailService = emailService;
        this.logger = logger;
    }

    async register(registrationData) {
        // ✅ প্রতিটি ধাপ আলাদা ও পরিষ্কার
        this.validator.validate(registrationData);
        await this.#ensureEmailIsUnique(registrationData.email);
        const userId = await this.repository.save(registrationData);
        await this.emailService.sendWelcomeEmail(registrationData.email, registrationData.name);
        this.logger.info(`ইউজার রেজিস্ট্রেশন সফল: ${registrationData.email}`);
        return userId;
    }

    async #ensureEmailIsUnique(email) {
        const exists = await this.repository.existsByEmail(email);
        if (exists) {
            throw new DuplicateEmailError('এই ইমেইল আগে থেকে আছে');
        }
    }
}
```

---

# 📐 ৩. কোড ফরম্যাটিং (Code Formatting)

> _"কোড ফরম্যাটিং হলো যোগাযোগের একটি মাধ্যম। এবং যোগাযোগ হলো পেশাদার ডেভেলপারের প্রথম অগ্রাধিকার।"_
> — **Robert C. Martin**

ফরম্যাটিং নিয়মাবলী:
1. **Vertical Formatting** — সম্পর্কিত কোড কাছাকাছি রাখুন
2. **Horizontal Formatting** — লাইন ১২০ অক্ষরের বেশি নয়
3. **Indentation** — সঠিক ইন্ডেন্টেশন ব্যবহার করুন
4. **Blank Lines** — লজিক্যাল ব্লকের মধ্যে ফাঁকা লাইন দিন

## ❌ খারাপ উদাহরণ (Formatting)

### PHP (❌ বিশৃঙ্খল ফরম্যাটিং)

```php
<?php
// ❌ কোনো ফরম্যাটিং নেই — পড়া অত্যন্ত কষ্টকর
class ProductService{
private $repo;private $cache;private $logger;
public function __construct($r,$c,$l){$this->repo=$r;$this->cache=$c;$this->logger=$l;}
public function getProduct($id){$cached=$this->cache->get("product_$id");if($cached){return $cached;}
$product=$this->repo->find($id);if(!$product){throw new Exception("Not found");}
$this->cache->set("product_$id",$product,3600);$this->logger->info("Product fetched: $id");return $product;}
public function updateProduct($id,$data){$product=$this->repo->find($id);
if(!$product){throw new Exception("Not found");}
$product->name=$data['name'];$product->price=$data['price'];$product->description=$data['description'];
$this->repo->save($product);$this->cache->delete("product_$id");
$this->logger->info("Product updated: $id");return $product;}
public function deleteProduct($id){$product=$this->repo->find($id);if(!$product){throw new Exception("Not found");}$this->repo->delete($product);$this->cache->delete("product_$id");$this->logger->info("Deleted: $id");}
}
```

### JavaScript (❌ বিশৃঙ্খল ফরম্যাটিং)

```javascript
// ❌ অসামঞ্জস্যপূর্ণ ফরম্যাটিং — প্রতিটি ব্লক আলাদা স্টাইলে

class ProductService{
    constructor(repo,cache,logger){this.repo=repo;this.cache=cache;this.logger=logger}

async getProduct(id)
{
        const cached = this.cache.get(`product_${id}`)
    if(cached){return cached}
const product=await this.repo.find(id)
        if (!product)
    {throw new Error("Not found")}
    this.cache.set(`product_${id}`,product,3600)
            this.logger.info(`Fetched: ${id}`)
    return product
}

    async updateProduct(id,data){
const product=await this.repo.find(id);if(!product){throw new Error("Not found")}
        product.name=data.name
product.price=data.price;product.description=data.description
        await this.repo.save(product);this.cache.delete(`product_${id}`)
return product}
}
```

## ✅ ভালো উদাহরণ (Formatting)

### PHP (✅ সুন্দর ফরম্যাটিং)

```php
<?php
// ✅ পরিষ্কার ফরম্যাটিং — পড়তে আনন্দ লাগে

class ProductService
{
    private ProductRepository $repository;
    private CacheService $cache;
    private Logger $logger;

    public function __construct(
        ProductRepository $repository,
        CacheService $cache,
        Logger $logger
    ) {
        $this->repository = $repository;
        $this->cache = $cache;
        $this->logger = $logger;
    }

    public function getProduct(int $productId): Product
    {
        $cachedProduct = $this->cache->get("product_{$productId}");

        if ($cachedProduct) {
            return $cachedProduct;
        }

        $product = $this->findProductOrFail($productId);
        $this->cache->set("product_{$productId}", $product, 3600);
        $this->logger->info("প্রোডাক্ট লোড হয়েছে: {$productId}");

        return $product;
    }

    public function updateProduct(int $productId, array $data): Product
    {
        $product = $this->findProductOrFail($productId);

        $product->name = $data['name'];
        $product->price = $data['price'];
        $product->description = $data['description'];

        $this->repository->save($product);
        $this->cache->delete("product_{$productId}");
        $this->logger->info("প্রোডাক্ট আপডেট হয়েছে: {$productId}");

        return $product;
    }

    public function deleteProduct(int $productId): void
    {
        $product = $this->findProductOrFail($productId);

        $this->repository->delete($product);
        $this->cache->delete("product_{$productId}");
        $this->logger->info("প্রোডাক্ট মুছে ফেলা হয়েছে: {$productId}");
    }

    private function findProductOrFail(int $productId): Product
    {
        $product = $this->repository->find($productId);

        if (!$product) {
            throw new ProductNotFoundException("প্রোডাক্ট পাওয়া যায়নি: {$productId}");
        }

        return $product;
    }
}
```

### JavaScript (✅ সুন্দর ফরম্যাটিং)

```javascript
// ✅ সামঞ্জস্যপূর্ণ ফরম্যাটিং — সবকিছু সুসংগঠিত

class ProductService {
    constructor(repository, cache, logger) {
        this.repository = repository;
        this.cache = cache;
        this.logger = logger;
    }

    async getProduct(productId) {
        const cachedProduct = this.cache.get(`product_${productId}`);

        if (cachedProduct) {
            return cachedProduct;
        }

        const product = await this.#findProductOrFail(productId);
        this.cache.set(`product_${productId}`, product, 3600);
        this.logger.info(`প্রোডাক্ট লোড হয়েছে: ${productId}`);

        return product;
    }

    async updateProduct(productId, data) {
        const product = await this.#findProductOrFail(productId);

        product.name = data.name;
        product.price = data.price;
        product.description = data.description;

        await this.repository.save(product);
        this.cache.delete(`product_${productId}`);
        this.logger.info(`প্রোডাক্ট আপডেট হয়েছে: ${productId}`);

        return product;
    }

    async deleteProduct(productId) {
        const product = await this.#findProductOrFail(productId);

        await this.repository.delete(product);
        this.cache.delete(`product_${productId}`);
        this.logger.info(`প্রোডাক্ট মুছে ফেলা হয়েছে: ${productId}`);
    }

    async #findProductOrFail(productId) {
        const product = await this.repository.find(productId);

        if (!product) {
            throw new ProductNotFoundError(`প্রোডাক্ট পাওয়া যায়নি: ${productId}`);
        }

        return product;
    }
}
```

---

# 💬 ৪. কমেন্ট লেখার নিয়ম (Comments)

> _"ভালো কোডের সবচেয়ে ভালো কমেন্ট হলো — কোনো কমেন্ট না লেখা।"_

কমেন্টের নিয়ম:
1. **কোড নিজেই বলুক** — কমেন্ট ছাড়াই বোঝা যাওয়া উচিত
2. **"কেন" লিখুন, "কী" নয়** — কোড "কী" করছে তা দেখলেই বোঝা যায়
3. **পুরনো কমেন্ট মুছুন** — আউটডেটেড কমেন্ট ক্ষতিকর
4. **কমেন্ট আউট কোড রাখবেন না** — Git আছে হিস্ট্রির জন্য

## ❌ খারাপ উদাহরণ (Comments)

### PHP (❌ অপ্রয়োজনীয় ও বিভ্রান্তিকর কমেন্ট)

```php
<?php
// ❌ খারাপ কমেন্টের উদাহরণ

class InvoiceService {
    // ইনভয়েস সার্ভিস ক্লাস (❌ অপ্রয়োজনীয় — নাম দেখেই বোঝা যায়)

    /**
     * ইনভয়েস তৈরি করে
     * @param array $data ডাটা
     * @return Invoice ইনভয়েস
     * (❌ কমেন্ট কোনো নতুন তথ্য দিচ্ছে না)
     */
    public function createInvoice(array $data): Invoice {
        // ইনভয়েস অবজেক্ট তৈরি করি (❌ কোড দেখেই বোঝা যাচ্ছে)
        $invoice = new Invoice();

        // নাম সেট করি (❌ একদম অপ্রয়োজনীয়)
        $invoice->customerName = $data['name'];

        // মূল্য সেট করি (❌ কোড নিজেই বলছে)
        $invoice->amount = $data['amount'];

        // ১৫% ট্যাক্স যোগ করি (❌ ম্যাজিক নম্বর ব্যাখ্যার চেয়ে কনস্ট্যান্ট ব্যবহার করুন)
        $invoice->tax = $data['amount'] * 0.15;

        // TODO: পরে ঠিক করবো (❌ কবে? কী ঠিক করবো?)
        // $invoice->discount = calculateDiscount($data); (❌ কমেন্ট আউট কোড)
        
        // নিচের লাইনটা কাজ করছে না তাই কমেন্ট করে রাখলাম
        // $this->sendNotification($invoice);

        return $invoice; // রিটার্ন (❌ সবচেয়ে অপ্রয়োজনীয় কমেন্ট)
    }
}
```

### JavaScript (❌ অপ্রয়োজনীয় ও বিভ্রান্তিকর কমেন্ট)

```javascript
// ❌ খারাপ কমেন্ট — কোড বোঝায় সাহায্য করে না, বরং বিভ্রান্ত করে

class InvoiceService {
    // কনস্ট্রাক্টর (❌ অপ্রয়োজনীয়)
    constructor(repository) {
        this.repository = repository; // রিপোজিটরি সেট করি
    }

    // ইনভয়েস তৈরি করার মেথড (❌ নাম দেখেই বোঝা যায়)
    createInvoice(data) {
        const invoice = {}; // খালি অবজেক্ট তৈরি করি

        // লুপ চালাই (❌ কোড দেখেই বোঝা যাচ্ছে)
        for (let i = 0; i < data.items.length; i++) {
            // আইটেম যোগ করি (❌ অপ্রয়োজনীয়)
            invoice.items.push(data.items[i]);
        }

        // ২০১৮ সালে আহমেদ ভাই এই লজিক লিখেছিলেন (❌ Git blame ব্যবহার করুন)
        invoice.total = this.calculateTotal(data.items);

        // invoice.discount = getDiscount(); // পুরনো কোড, কাজ করে না
        // invoice.shipping = getShipping(); // এটাও বাদ

        return invoice; // ইনভয়েস রিটার্ন
    }
}
```

## ✅ ভালো উদাহরণ (Comments)

### PHP (✅ অর্থবহ কমেন্ট)

```php
<?php
// ✅ কমেন্ট শুধু "কেন" ব্যাখ্যা করে, "কী" নয়

class InvoiceService {
    // সরকারি নিয়ম অনুযায়ী VAT হার (Finance Act 2023, Section 12)
    private const VAT_RATE = 0.15;

    // ন্যূনতম ইনভয়েস পরিমাণ — ব্যাংক ট্রান্সফার ফি কভার করতে
    private const MIN_INVOICE_AMOUNT = 100;

    public function createInvoice(InvoiceData $data): Invoice
    {
        $invoice = new Invoice();
        $invoice->customerName = $data->customerName;
        $invoice->amount = $data->amount;
        $invoice->tax = $data->amount * self::VAT_RATE;

        // ক্রেডিট কার্ড প্রসেসরের API সীমাবদ্ধতার কারণে
        // পরিমাণ ২ দশমিক স্থানে রাউন্ড করতে হয়
        $invoice->total = round($invoice->amount + $invoice->tax, 2);

        return $invoice;
    }

    /**
     * ব্যবসায়িক নিয়ম: ৩০ দিনের বেশি পুরনো অপরিশোধিত ইনভয়েসে
     * ৫% বিলম্ব ফি যোগ হবে (চুক্তি অনুযায়ী, বিভাগ ৪.২)
     */
    public function applyLateFee(Invoice $invoice): void
    {
        $daysSinceCreation = $invoice->createdAt->diffInDays(now());

        if ($daysSinceCreation > 30 && !$invoice->isPaid()) {
            $invoice->addLateFee($invoice->total * 0.05);
        }
    }
}
```

### JavaScript (✅ অর্থবহ কমেন্ট)

```javascript
// ✅ কমেন্ট শুধু জটিল লজিক ও ব্যবসায়িক নিয়ম ব্যাখ্যা করে

class InvoiceService {
    // সরকারি নিয়ম অনুযায়ী VAT হার (Finance Act 2023)
    static VAT_RATE = 0.15;

    // পেমেন্ট গেটওয়ের সীমাবদ্ধতা — ন্যূনতম ট্রানজ্যাকশন পরিমাণ
    static MIN_INVOICE_AMOUNT = 100;

    createInvoice(invoiceData) {
        const invoice = new Invoice();
        invoice.customerName = invoiceData.customerName;
        invoice.amount = invoiceData.amount;
        invoice.tax = invoiceData.amount * InvoiceService.VAT_RATE;

        // Stripe API দশমিকের পরে ২ ঘর পর্যন্ত গ্রহণ করে,
        // তাই রাউন্ড করা বাধ্যতামূলক
        invoice.total = Math.round((invoice.amount + invoice.tax) * 100) / 100;

        return invoice;
    }

    /**
     * WARNING: এই মেথড প্রতিদিন রাত ২টায় cron job দ্বারা কল হয়।
     * পরিবর্তন করলে DevOps টিমকে জানাতে হবে।
     * @see https://wiki.internal/late-fee-policy
     */
    applyLateFee(invoice) {
        const daysSinceCreation = this.#calculateDaysSince(invoice.createdAt);

        if (daysSinceCreation > 30 && !invoice.isPaid) {
            invoice.addLateFee(invoice.total * 0.05);
        }
    }
}
```

---

# 🚨 ৫. এরর হ্যান্ডলিং (Error Handling)

> _"এরর হ্যান্ডলিং গুরুত্বপূর্ণ, কিন্তু এটি যদি কোডের লজিককে ঢেকে দেয়, তাহলে ভুল হচ্ছে।"_
> — **Robert C. Martin**

এরর হ্যান্ডলিং নীতি:
1. **Exception ব্যবহার করুন**, রিটার্ন কোড নয়
2. **নির্দিষ্ট Exception** ছুঁড়ুন, জেনেরিক নয়
3. **কনটেক্সট দিন** — কী ভুল হয়েছে তা পরিষ্কার থাকুক
4. **null রিটার্ন করবেন না** — Optional বা Exception ব্যবহার করুন

## ❌ খারাপ উদাহরণ (Error Handling)

### PHP (❌ দুর্বল এরর হ্যান্ডলিং)

```php
<?php
// ❌ খারাপ এরর হ্যান্ডলিং — সমস্যা চিহ্নিত করা কঠিন

class PaymentService {
    public function processPayment($orderId, $amount, $cardNumber) {
        // ❌ সমস্যা ১: null রিটার্ন — কী ভুল হয়েছে বোঝা যায় না
        if (!$orderId) {
            return null;
        }

        // ❌ সমস্যা ২: জেনেরিক Exception — কোনো তথ্য নেই
        try {
            $order = $this->orderRepo->find($orderId);
        } catch (Exception $e) {
            return false; // কী ত্রুটি? ডাটাবেইজ? নেটওয়ার্ক?
        }

        // ❌ সমস্যা ৩: error code রিটার্ন — ম্যাজিক নম্বর
        if ($amount <= 0) {
            return -1; // -1 মানে কী?
        }
        if ($amount > 100000) {
            return -2; // -2 মানে কী?
        }

        // ❌ সমস্যা ৪: ব্যাপক try-catch — সব এরর একসাথে ধরছে
        try {
            $this->validateCard($cardNumber);
            $this->chargeCard($cardNumber, $amount);
            $this->updateOrder($orderId, 'paid');
            $this->sendReceipt($orderId);
            $this->notifyWarehouse($orderId);
        } catch (Exception $e) {
            // ❌ সব এরর একই ভাবে হ্যান্ডেল — কোনটায় সমস্যা?
            error_log($e->getMessage());
            return false;
        }

        return true;
    }
}
```

### JavaScript (❌ দুর্বল এরর হ্যান্ডলিং)

```javascript
// ❌ খারাপ এরর হ্যান্ডলিং — ডিবাগ করা দুঃস্বপ্ন

class PaymentService {
    async processPayment(orderId, amount, cardNumber) {
        // ❌ null/undefined চেক ছাড়াই এগিয়ে যাওয়া
        const order = await this.orderRepo.find(orderId);

        // ❌ সাইলেন্ট ফেইলিউর — এরর গিলে ফেলা
        try {
            await this.chargeCard(cardNumber, amount);
        } catch (e) {
            console.log('error'); // কী error? কোথায়?
            return false;
        }

        // ❌ error code ব্যবহার
        const result = await this.updateOrder(orderId, 'paid');
        if (result === 0) return { success: false, code: 'UPDATE_FAILED' };
        if (result === -1) return { success: false, code: 'ORDER_LOCKED' };

        // ❌ nested try-catch — জটিল ও দুর্বোধ্য
        try {
            try {
                await this.sendReceipt(orderId);
            } catch (e) {
                try {
                    await this.sendReceiptFallback(orderId);
                } catch (e2) {
                    console.log('receipt failed');
                }
            }
        } catch (e) {
            // কিছুই না
        }

        return { success: true };
    }
}
```

## ✅ ভালো উদাহরণ (Error Handling)

### PHP (✅ শক্তিশালী এরর হ্যান্ডলিং)

```php
<?php
// ✅ পরিষ্কার এরর হ্যান্ডলিং — প্রতিটি সমস্যা সুনির্দিষ্ট

// কাস্টম Exception — নির্দিষ্ট সমস্যা চিহ্নিত করে
class OrderNotFoundException extends RuntimeException {}
class InvalidPaymentAmountException extends InvalidArgumentException {}
class PaymentGatewayException extends RuntimeException {}
class CardDeclinedException extends PaymentGatewayException {}

class PaymentService {
    private const MIN_PAYMENT_AMOUNT = 1;
    private const MAX_PAYMENT_AMOUNT = 100000;

    public function __construct(
        private OrderRepository $orderRepo,
        private PaymentGateway $gateway,
        private NotificationService $notifier,
        private Logger $logger
    ) {}

    /**
     * @throws OrderNotFoundException
     * @throws InvalidPaymentAmountException
     * @throws PaymentGatewayException
     */
    public function processPayment(int $orderId, float $amount, string $cardToken): PaymentResult
    {
        $order = $this->findOrderOrFail($orderId);
        $this->validatePaymentAmount($amount);

        try {
            $transactionId = $this->gateway->charge($cardToken, $amount);
        } catch (CardDeclinedException $e) {
            $this->logger->warning("কার্ড প্রত্যাখ্যান: অর্ডার #{$orderId}", ['error' => $e->getMessage()]);
            throw $e;
        } catch (PaymentGatewayException $e) {
            $this->logger->error("পেমেন্ট গেটওয়ে সমস্যা: অর্ডার #{$orderId}", ['error' => $e->getMessage()]);
            throw $e;
        }

        $order->markAsPaid($transactionId);
        $this->orderRepo->save($order);

        // রিসিপ্ট পাঠানো ব্যর্থ হলেও পেমেন্ট সফল
        $this->trySendReceipt($orderId);

        return new PaymentResult($transactionId, $amount);
    }

    private function findOrderOrFail(int $orderId): Order
    {
        $order = $this->orderRepo->find($orderId);

        if (!$order) {
            throw new OrderNotFoundException("অর্ডার পাওয়া যায়নি: #{$orderId}");
        }

        return $order;
    }

    private function validatePaymentAmount(float $amount): void
    {
        if ($amount < self::MIN_PAYMENT_AMOUNT) {
            throw new InvalidPaymentAmountException(
                "পরিমাণ ন্যূনতম " . self::MIN_PAYMENT_AMOUNT . " হতে হবে, পাওয়া গেছে: {$amount}"
            );
        }

        if ($amount > self::MAX_PAYMENT_AMOUNT) {
            throw new InvalidPaymentAmountException(
                "পরিমাণ সর্বোচ্চ " . self::MAX_PAYMENT_AMOUNT . " হতে পারে, পাওয়া গেছে: {$amount}"
            );
        }
    }

    private function trySendReceipt(int $orderId): void
    {
        try {
            $this->notifier->sendPaymentReceipt($orderId);
        } catch (\Throwable $e) {
            // রিসিপ্ট পাঠানো ব্যর্থ হলে লগ করি, কিন্তু পেমেন্ট ফেইল করি না
            $this->logger->warning("রিসিপ্ট পাঠানো ব্যর্থ: অর্ডার #{$orderId}", [
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

### JavaScript (✅ শক্তিশালী এরর হ্যান্ডলিং)

```javascript
// ✅ পরিষ্কার এরর হ্যান্ডলিং — সুনির্দিষ্ট ও তথ্যবহুল

// কাস্টম Error ক্লাস — প্রতিটি সমস্যার জন্য আলাদা
class OrderNotFoundError extends Error {
    constructor(orderId) {
        super(`অর্ডার পাওয়া যায়নি: #${orderId}`);
        this.name = 'OrderNotFoundError';
        this.orderId = orderId;
    }
}

class InvalidPaymentAmountError extends Error {
    constructor(amount, min, max) {
        super(`অবৈধ পরিমাণ: ${amount} (সীমা: ${min}-${max})`);
        this.name = 'InvalidPaymentAmountError';
    }
}

class PaymentGatewayError extends Error {
    constructor(message, originalError) {
        super(message);
        this.name = 'PaymentGatewayError';
        this.cause = originalError;
    }
}

class PaymentService {
    static MIN_PAYMENT = 1;
    static MAX_PAYMENT = 100000;

    constructor(orderRepo, gateway, notifier, logger) {
        this.orderRepo = orderRepo;
        this.gateway = gateway;
        this.notifier = notifier;
        this.logger = logger;
    }

    async processPayment(orderId, amount, cardToken) {
        const order = await this.#findOrderOrFail(orderId);
        this.#validatePaymentAmount(amount);

        let transactionId;
        try {
            transactionId = await this.gateway.charge(cardToken, amount);
        } catch (error) {
            this.logger.error(`পেমেন্ট ব্যর্থ: অর্ডার #${orderId}`, { error: error.message });
            throw new PaymentGatewayError(`পেমেন্ট প্রসেস করা যায়নি`, error);
        }

        order.markAsPaid(transactionId);
        await this.orderRepo.save(order);

        // রিসিপ্ট পাঠানো ব্যর্থ হলেও পেমেন্ট সফল থাকবে
        await this.#trySendReceipt(orderId);

        return { transactionId, amount, success: true };
    }

    async #findOrderOrFail(orderId) {
        const order = await this.orderRepo.find(orderId);

        if (!order) {
            throw new OrderNotFoundError(orderId);
        }

        return order;
    }

    #validatePaymentAmount(amount) {
        if (amount < PaymentService.MIN_PAYMENT || amount > PaymentService.MAX_PAYMENT) {
            throw new InvalidPaymentAmountError(
                amount,
                PaymentService.MIN_PAYMENT,
                PaymentService.MAX_PAYMENT
            );
        }
    }

    async #trySendReceipt(orderId) {
        try {
            await this.notifier.sendPaymentReceipt(orderId);
        } catch (error) {
            this.logger.warning(`রিসিপ্ট পাঠানো ব্যর্থ: অর্ডার #${orderId}`, {
                error: error.message
            });
        }
    }
}
```

---

# 📚 সম্পূর্ণ উদাহরণ: লাইব্রেরি ম্যানেজমেন্ট সিস্টেম

এখন আমরা উপরের সব নীতি একসাথে প্রয়োগ করে একটি **লাইব্রেরি ম্যানেজমেন্ট সিস্টেম** তৈরি করবো।

## PHP (✅ সম্পূর্ণ ক্লিন কোড উদাহরণ)

```php
<?php
// 📚 লাইব্রেরি ম্যানেজমেন্ট সিস্টেম — ক্লিন কোড নীতি অনুসরণ করে

// ✅ নামকরণ: নির্দিষ্ট Exception ক্লাস
class BookNotFoundException extends RuntimeException {}
class BookAlreadyBorrowedException extends RuntimeException {}
class MemberSuspendedException extends RuntimeException {}
class BorrowLimitExceededException extends RuntimeException {}

// ✅ নামকরণ: Value Object — উদ্দেশ্য পরিষ্কার
class ISBN {
    public function __construct(private readonly string $value) {
        if (!$this->isValid($value)) {
            throw new InvalidArgumentException("অবৈধ ISBN: {$value}");
        }
    }

    private function isValid(string $isbn): bool {
        return preg_match('/^(97[89])\d{10}$/', str_replace('-', '', $isbn)) === 1;
    }

    public function getValue(): string {
        return $this->value;
    }
}

// ✅ ফাংশন: প্রতিটি ক্লাস একটি দায়িত্ব পালন করে
class Book {
    private bool $isBorrowed = false;
    private ?string $borrowedByMemberId = null;
    private ?DateTimeImmutable $dueDate = null;

    public function __construct(
        private readonly string $id,
        private readonly string $title,
        private readonly string $author,
        private readonly ISBN $isbn
    ) {}

    public function markAsBorrowed(string $memberId, DateTimeImmutable $dueDate): void {
        if ($this->isBorrowed) {
            throw new BookAlreadyBorrowedException(
                "'{$this->title}' বইটি ইতিমধ্যে ধার দেওয়া হয়েছে"
            );
        }

        $this->isBorrowed = true;
        $this->borrowedByMemberId = $memberId;
        $this->dueDate = $dueDate;
    }

    public function markAsReturned(): void {
        $this->isBorrowed = false;
        $this->borrowedByMemberId = null;
        $this->dueDate = null;
    }

    public function isOverdue(): bool {
        if (!$this->isBorrowed || !$this->dueDate) {
            return false;
        }
        return new DateTimeImmutable() > $this->dueDate;
    }

    public function isBorrowed(): bool { return $this->isBorrowed; }
    public function getId(): string { return $this->id; }
    public function getTitle(): string { return $this->title; }
    public function getAuthor(): string { return $this->author; }
    public function getDueDate(): ?DateTimeImmutable { return $this->dueDate; }
}

class LibraryMember {
    // সরকারি লাইব্রেরি নীতি অনুযায়ী সর্বোচ্চ ধার সীমা
    private const MAX_BORROW_LIMIT = 5;
    private array $borrowedBookIds = [];

    public function __construct(
        private readonly string $id,
        private readonly string $name,
        private bool $isSuspended = false
    ) {}

    public function canBorrowBooks(): bool {
        return !$this->isSuspended
            && count($this->borrowedBookIds) < self::MAX_BORROW_LIMIT;
    }

    public function addBorrowedBook(string $bookId): void {
        $this->borrowedBookIds[] = $bookId;
    }

    public function removeBorrowedBook(string $bookId): void {
        $this->borrowedBookIds = array_filter(
            $this->borrowedBookIds,
            fn($id) => $id !== $bookId
        );
    }

    public function getBorrowedBookCount(): int {
        return count($this->borrowedBookIds);
    }

    public function getId(): string { return $this->id; }
    public function getName(): string { return $this->name; }
    public function isSuspended(): bool { return $this->isSuspended; }
}

// ✅ ফাংশন: প্রতিটি মেথড একটি কাজ করে
class LibraryService {
    // ধার দেওয়ার সময়কাল — লাইব্রেরি পলিসি অনুযায়ী
    private const BORROW_PERIOD_DAYS = 14;

    public function __construct(
        private BookRepository $bookRepo,
        private MemberRepository $memberRepo,
        private NotificationService $notifier,
        private Logger $logger
    ) {}

    public function borrowBook(string $memberId, string $bookId): BorrowResult
    {
        $member = $this->findMemberOrFail($memberId);
        $book = $this->findBookOrFail($bookId);

        $this->ensureMemberCanBorrow($member);
        $this->ensureBookIsAvailable($book);

        $dueDate = new DateTimeImmutable("+".self::BORROW_PERIOD_DAYS." days");
        $book->markAsBorrowed($memberId, $dueDate);
        $member->addBorrowedBook($bookId);

        $this->bookRepo->save($book);
        $this->memberRepo->save($member);

        $this->logger->info("বই ধার দেওয়া হয়েছে", [
            'book' => $book->getTitle(),
            'member' => $member->getName(),
            'dueDate' => $dueDate->format('Y-m-d')
        ]);

        $this->trySendBorrowNotification($member, $book, $dueDate);

        return new BorrowResult($book, $member, $dueDate);
    }

    public function returnBook(string $memberId, string $bookId): ReturnResult
    {
        $member = $this->findMemberOrFail($memberId);
        $book = $this->findBookOrFail($bookId);

        $isOverdue = $book->isOverdue();
        $book->markAsReturned();
        $member->removeBorrowedBook($bookId);

        $this->bookRepo->save($book);
        $this->memberRepo->save($member);

        $this->logger->info("বই ফেরত নেওয়া হয়েছে", [
            'book' => $book->getTitle(),
            'member' => $member->getName(),
            'wasOverdue' => $isOverdue
        ]);

        return new ReturnResult($book, $member, $isOverdue);
    }

    // ✅ এরর হ্যান্ডলিং: নির্দিষ্ট Exception ও পরিষ্কার বার্তা
    private function findMemberOrFail(string $memberId): LibraryMember
    {
        $member = $this->memberRepo->find($memberId);

        if (!$member) {
            throw new MemberNotFoundException("সদস্য পাওয়া যায়নি: #{$memberId}");
        }

        return $member;
    }

    private function findBookOrFail(string $bookId): Book
    {
        $book = $this->bookRepo->find($bookId);

        if (!$book) {
            throw new BookNotFoundException("বই পাওয়া যায়নি: #{$bookId}");
        }

        return $book;
    }

    private function ensureMemberCanBorrow(LibraryMember $member): void
    {
        if ($member->isSuspended()) {
            throw new MemberSuspendedException(
                "{$member->getName()} এর একাউন্ট স্থগিত করা আছে"
            );
        }

        if (!$member->canBorrowBooks()) {
            throw new BorrowLimitExceededException(
                "{$member->getName()} সর্বোচ্চ ধার সীমায় পৌঁছেছেন ({$member->getBorrowedBookCount()} বই)"
            );
        }
    }

    private function ensureBookIsAvailable(Book $book): void
    {
        if ($book->isBorrowed()) {
            throw new BookAlreadyBorrowedException(
                "'{$book->getTitle()}' বইটি এখন পাওয়া যাচ্ছে না"
            );
        }
    }

    // ✅ নোটিফিকেশন ব্যর্থ হলে মূল কাজ ব্যাহত হবে না
    private function trySendBorrowNotification(
        LibraryMember $member,
        Book $book,
        DateTimeImmutable $dueDate
    ): void {
        try {
            $this->notifier->sendBorrowConfirmation($member, $book, $dueDate);
        } catch (\Throwable $e) {
            $this->logger->warning("নোটিফিকেশন পাঠানো ব্যর্থ", [
                'member' => $member->getId(),
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

## JavaScript (✅ সম্পূর্ণ ক্লিন কোড উদাহরণ)

```javascript
// 📚 লাইব্রেরি ম্যানেজমেন্ট সিস্টেম — ক্লিন কোড নীতি অনুসরণ করে

// ✅ নামকরণ: নির্দিষ্ট Error ক্লাস
class BookNotFoundError extends Error {
    constructor(bookId) {
        super(`বই পাওয়া যায়নি: #${bookId}`);
        this.name = 'BookNotFoundError';
    }
}

class BookAlreadyBorrowedError extends Error {
    constructor(bookTitle) {
        super(`'${bookTitle}' বইটি ইতিমধ্যে ধার দেওয়া হয়েছে`);
        this.name = 'BookAlreadyBorrowedError';
    }
}

class MemberSuspendedError extends Error {
    constructor(memberName) {
        super(`${memberName} এর একাউন্ট স্থগিত`);
        this.name = 'MemberSuspendedError';
    }
}

class BorrowLimitExceededError extends Error {
    constructor(memberName, currentCount) {
        super(`${memberName} সর্বোচ্চ ধার সীমায় পৌঁছেছেন (${currentCount} বই)`);
        this.name = 'BorrowLimitExceededError';
    }
}

// ✅ ছোট ক্লাস — একটি দায়িত্ব
class Book {
    #isBorrowed = false;
    #borrowedByMemberId = null;
    #dueDate = null;

    constructor(id, title, author, isbn) {
        this.id = id;
        this.title = title;
        this.author = author;
        this.isbn = isbn;
    }

    markAsBorrowed(memberId, dueDate) {
        if (this.#isBorrowed) {
            throw new BookAlreadyBorrowedError(this.title);
        }
        this.#isBorrowed = true;
        this.#borrowedByMemberId = memberId;
        this.#dueDate = dueDate;
    }

    markAsReturned() {
        this.#isBorrowed = false;
        this.#borrowedByMemberId = null;
        this.#dueDate = null;
    }

    isOverdue() {
        if (!this.#isBorrowed || !this.#dueDate) return false;
        return new Date() > this.#dueDate;
    }

    get isBorrowed() { return this.#isBorrowed; }
    get dueDate() { return this.#dueDate; }
}

class LibraryMember {
    // সরকারি লাইব্রেরি নীতি অনুযায়ী সর্বোচ্চ ধার সীমা
    static MAX_BORROW_LIMIT = 5;
    #borrowedBookIds = [];

    constructor(id, name, isSuspended = false) {
        this.id = id;
        this.name = name;
        this.isSuspended = isSuspended;
    }

    canBorrowBooks() {
        return !this.isSuspended
            && this.#borrowedBookIds.length < LibraryMember.MAX_BORROW_LIMIT;
    }

    addBorrowedBook(bookId) {
        this.#borrowedBookIds.push(bookId);
    }

    removeBorrowedBook(bookId) {
        this.#borrowedBookIds = this.#borrowedBookIds.filter(id => id !== bookId);
    }

    get borrowedBookCount() {
        return this.#borrowedBookIds.length;
    }
}

// ✅ ফাংশন: প্রতিটি মেথড ছোট ও একক দায়িত্ব পালন করে
class LibraryService {
    // ধার দেওয়ার সময়কাল — লাইব্রেরি পলিসি অনুযায়ী
    static BORROW_PERIOD_DAYS = 14;

    constructor(bookRepo, memberRepo, notifier, logger) {
        this.bookRepo = bookRepo;
        this.memberRepo = memberRepo;
        this.notifier = notifier;
        this.logger = logger;
    }

    async borrowBook(memberId, bookId) {
        const member = await this.#findMemberOrFail(memberId);
        const book = await this.#findBookOrFail(bookId);

        this.#ensureMemberCanBorrow(member);
        this.#ensureBookIsAvailable(book);

        const dueDate = new Date();
        dueDate.setDate(dueDate.getDate() + LibraryService.BORROW_PERIOD_DAYS);

        book.markAsBorrowed(memberId, dueDate);
        member.addBorrowedBook(bookId);

        await this.bookRepo.save(book);
        await this.memberRepo.save(member);

        this.logger.info('বই ধার দেওয়া হয়েছে', {
            book: book.title,
            member: member.name,
            dueDate: dueDate.toISOString().split('T')[0]
        });

        await this.#trySendBorrowNotification(member, book, dueDate);

        return { book, member, dueDate };
    }

    async returnBook(memberId, bookId) {
        const member = await this.#findMemberOrFail(memberId);
        const book = await this.#findBookOrFail(bookId);

        const wasOverdue = book.isOverdue();
        book.markAsReturned();
        member.removeBorrowedBook(bookId);

        await this.bookRepo.save(book);
        await this.memberRepo.save(member);

        this.logger.info('বই ফেরত নেওয়া হয়েছে', {
            book: book.title,
            member: member.name,
            wasOverdue
        });

        return { book, member, wasOverdue };
    }

    async #findMemberOrFail(memberId) {
        const member = await this.memberRepo.find(memberId);
        if (!member) throw new MemberNotFoundError(memberId);
        return member;
    }

    async #findBookOrFail(bookId) {
        const book = await this.bookRepo.find(bookId);
        if (!book) throw new BookNotFoundError(bookId);
        return book;
    }

    #ensureMemberCanBorrow(member) {
        if (member.isSuspended) {
            throw new MemberSuspendedError(member.name);
        }
        if (!member.canBorrowBooks()) {
            throw new BorrowLimitExceededError(member.name, member.borrowedBookCount);
        }
    }

    #ensureBookIsAvailable(book) {
        if (book.isBorrowed) {
            throw new BookAlreadyBorrowedError(book.title);
        }
    }

    // ✅ নোটিফিকেশন ব্যর্থ হলে মূল কাজ ব্যাহত হবে না
    async #trySendBorrowNotification(member, book, dueDate) {
        try {
            await this.notifier.sendBorrowConfirmation(member, book, dueDate);
        } catch (error) {
            this.logger.warning('নোটিফিকেশন পাঠানো ব্যর্থ', {
                memberId: member.id,
                error: error.message
            });
        }
    }
}
```

---

## ✅ ক্লিন কোড চেকলিস্ট

| # | নীতি | চেক |
|---|---|---|
| ১ | ভেরিয়েবল/ফাংশনের নাম কি উদ্দেশ্য প্রকাশ করে? | ☐ |
| ২ | ফাংশন কি ২০ লাইনের কম? | ☐ |
| ৩ | প্রতিটি ফাংশন কি একটি কাজ করে? | ☐ |
| ৪ | ফাংশনে কি ৩টির কম প্যারামিটার আছে? | ☐ |
| ৫ | কোড কি সামঞ্জস্যপূর্ণভাবে ফরম্যাটেড? | ☐ |
| ৬ | কমেন্ট কি "কেন" ব্যাখ্যা করে, "কী" নয়? | ☐ |
| ৭ | কমেন্ট আউট কোড কি মুছে ফেলা হয়েছে? | ☐ |
| ৮ | Exception কি নির্দিষ্ট ও তথ্যবহুল? | ☐ |
| ৯ | null রিটার্নের বদলে কি Exception ব্যবহার হচ্ছে? | ☐ |
| ১০ | ম্যাজিক নম্বরের বদলে কি কনস্ট্যান্ট আছে? | ☐ |

---

## 🔗 পরবর্তী পড়ুন

- [Code Smells — কোড স্মেল চিহ্নিত করুন](./code-smells.md)

---

> _"সবসময় এমন কোড লিখুন যেন যে রক্ষণাবেক্ষণ করবে সে একজন হিংস্র সাইকোপ্যাথ যে জানে আপনি কোথায় থাকেন।"_
> — **John Woods**
