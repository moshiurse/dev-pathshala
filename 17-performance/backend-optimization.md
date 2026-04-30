# 📌 ব্যাকেন্ড অপটিমাইজেশন (Backend Optimization)

## সূচিপত্র (Table of Contents)
- [কেন দরকার](#-ব্যাকেন্ড-অপটিমাইজেশন-কেন-দরকার)
- [N+1 কোয়েরি সমস্যা](#-n1-কোয়েরি-সমস্যা)
- [কানেকশন পুলিং](#-কানেকশন-পুলিং)
- [কোয়েরি অপটিমাইজেশন](#-কোয়েরি-অপটিমাইজেশন)
- [অ্যাসিঙ্ক প্রসেসিং ও জব কিউ](#-অ্যাসিঙ্ক-প্রসেসিং-ও-জব-কিউ)
- [রেসপন্স কম্প্রেশন](#-রেসপন্স-কম্প্রেশন)
- [পেজিনেশন অপটিমাইজেশন](#-পেজিনেশন-অপটিমাইজেশন)
- [Before/After পারফরম্যান্স তুলনা](#-beforeafter-পারফরম্যান্স-তুলনা)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 ব্যাকেন্ড অপটিমাইজেশন কেন দরকার

বাংলাদেশের প্রেক্ষাপটে চিন্তা করুন:

- **bKash** প্রতিদিন কোটি কোটি টাকার ট্রানজেকশন প্রসেস করে — প্রতিটি রিকোয়েস্ট যদি ১ সেকেন্ড বেশি নেয়, তাহলে লক্ষ লক্ষ ইউজার হতাশ হবে
- **Pathao** রিয়েল-টাইমে রাইডার ম্যাচিং করে — ড্রাইভার খোঁজার জন্য ১০ সেকেন্ড অপেক্ষা করলে ইউজার অন্য অ্যাপে চলে যাবে
- **Daraz** ১১.১১ সেলে মিলিয়ন রিকোয়েস্ট হ্যান্ডেল করে — সার্ভার ডাউন মানে কোটি টাকার লস
- **Nagad** মোবাইল ব্যাংকিং-এ প্রতি মিলিসেকেন্ড গুরুত্বপূর্ণ
- **Grameenphone** MyGP অ্যাপে লক্ষ লক্ষ ইউজার একসাথে ব্যালান্স চেক করে

```
📊 অপটিমাইজেশনের প্রভাব (Real Impact):

  অপটিমাইজেশন ছাড়া          অপটিমাইজেশনের পরে
  ┌──────────────────┐        ┌──────────────────┐
  │ Response: 2.5s   │   →    │ Response: 200ms  │
  │ Memory:   512MB  │   →    │ Memory:   128MB  │
  │ CPU:      85%    │   →    │ CPU:      25%    │
  │ DB Hits:  1000/s │   →    │ DB Hits:  50/s   │
  │ Users:    500    │   →    │ Users:    10000  │
  └──────────────────┘        └──────────────────┘
         ❌ ধীরগতি                  ✅ দ্রুতগতি
```

### 🏠 বাস্তব উদাহরণ: bKash ট্রানজেকশন ফ্লো

```
ইউজার রিকোয়েস্ট → ব্যাকেন্ড প্রসেসিং → ডাটাবেস → রেসপন্স

অপটিমাইজেশন ছাড়া:
┌─────────┐    ┌─────────────┐    ┌──────────┐    ┌──────────┐
│  User   │───→│  API Server │───→│    DB    │───→│ Response │
│ Request │    │  (500ms)    │    │ (1500ms) │    │ (2000ms) │
└─────────┘    └─────────────┘    └──────────┘    └──────────┘
                 ❌ Slow              ❌ N+1          ❌ Late

অপটিমাইজেশনের পরে:
┌─────────┐    ┌─────────────┐    ┌──────────┐    ┌──────────┐
│  User   │───→│  API + Cache│───→│  Pooled  │───→│ Response │
│ Request │    │  (50ms)     │    │  DB(80ms)│    │ (150ms)  │
└─────────┘    └─────────────┘    └──────────┘    └──────────┘
                 ✅ Cached            ✅ Pooled       ✅ Fast
```

---

## 🔄 N+1 কোয়েরি সমস্যা

### 📖 N+1 সমস্যা কী?

N+1 সমস্যা হলো যখন আমরা একটি লিস্ট ফেচ করার জন্য ১টি কোয়েরি চালাই, তারপর সেই লিস্টের **প্রতিটি আইটেমের** জন্য আলাদা আলাদা কোয়েরি চালাই। মানে N আইটেম থাকলে মোট (N + 1) টি কোয়েরি হয়।

### ASCII ডায়াগ্রাম: N+1 সমস্যার ফ্লো

```
❌ N+1 সমস্যা (যদি ১০০টি অর্ডার থাকে):

Query 1: SELECT * FROM orders;                    ← ১টি কোয়েরি (সব অর্ডার)
    │
    ├── Query 2:  SELECT * FROM customers WHERE id = 1;   ← অর্ডার ১ এর কাস্টমার
    ├── Query 3:  SELECT * FROM customers WHERE id = 2;   ← অর্ডার ২ এর কাস্টমার
    ├── Query 4:  SELECT * FROM customers WHERE id = 3;   ← অর্ডার ৩ এর কাস্টমার
    ├── ...
    └── Query 101: SELECT * FROM customers WHERE id = 100; ← অর্ডার ১০০ এর কাস্টমার

    মোট কোয়েরি: ১ + ১০০ = ১০১ টি! 😱

✅ Eager Loading দিয়ে সমাধান:

Query 1: SELECT * FROM orders;                              ← ১টি কোয়েরি
Query 2: SELECT * FROM customers WHERE id IN (1,2,3,...100); ← ১টি কোয়েরি

    মোট কোয়েরি: ২টি! 🎉
```

### ❌ খারাপ প্র্যাক্টিস: N+1 Problem (Laravel)

```php
// ❌ N+1 Problem — প্রতিটি অর্ডারের জন্য আলাদা কোয়েরি চলবে
$orders = Order::all(); // 1 query: SELECT * FROM orders
foreach ($orders as $order) {
    echo $order->customer->name; // N queries! প্রতিটিতে আলাদা SELECT
}
// যদি ১০০০ অর্ডার থাকে → ১০০১ টি কোয়েরি! 😱
```

### ✅ ভালো প্র্যাক্টিস: Eager Loading (Laravel)

```php
// ✅ Eager Loading — মাত্র ২টি কোয়েরি
$orders = Order::with('customer')->get(); // 2 queries only!
// Query 1: SELECT * FROM orders
// Query 2: SELECT * FROM customers WHERE id IN (1, 2, 3, ...)

// ✅ আরও ভালো: শুধু দরকারি কলাম সিলেক্ট করুন
$orders = Order::with('customer:id,name')->get();

// ✅ Nested Eager Loading (অর্ডার → কাস্টমার → ঠিকানা)
$orders = Order::with('customer.address')->get();

// ✅ Conditional Eager Loading
$orders = Order::with(['customer' => function ($query) {
    $query->where('status', 'active');
}])->get();
```

### ❌ খারাপ প্র্যাক্টিস: N+1 in Node.js (Sequelize)

```javascript
// ❌ N+1 Problem — লুপের ভিতরে async কোয়েরি
const orders = await Order.findAll();
for (const order of orders) {
    const customer = await Customer.findByPk(order.customerId); // N queries!
    console.log(customer.name);
}
```

### ✅ ভালো প্র্যাক্টিস: Eager Loading (Sequelize)

```javascript
// ✅ Include দিয়ে Eager Loading
const orders = await Order.findAll({
    include: [{ model: Customer, attributes: ['id', 'name'] }]
});

// ✅ Nested Include
const orders = await Order.findAll({
    include: [{
        model: Customer,
        attributes: ['id', 'name'],
        include: [{ model: Address }]
    }]
});

// ✅ DataLoader ব্যবহার করে (GraphQL এর জন্য চমৎকার)
const DataLoader = require('dataloader');

const customerLoader = new DataLoader(async (ids) => {
    const customers = await Customer.findAll({ where: { id: ids } });
    const customerMap = {};
    customers.forEach(c => customerMap[c.id] = c);
    return ids.map(id => customerMap[id]);
});

// ব্যবহার — DataLoader অটোমেটিক batch করবে
const customer = await customerLoader.load(order.customerId);
```

### 📊 N+1 প্রভাব তুলনা (Daraz প্রোডাক্ট লিস্ট)

```
ধরুন Daraz-এ ৫০০টি প্রোডাক্ট লিস্ট পেজ আছে:

❌ N+1 পদ্ধতি:
   কোয়েরি সংখ্যা: ৫০১ (1 + 500)
   সময়: ~2500ms
   DB লোড: অনেক বেশি

✅ Eager Loading:
   কোয়েরি সংখ্যা: ২
   সময়: ~120ms
   DB লোড: স্বাভাবিক

   পারফরম্যান্স উন্নতি: ~20x দ্রুত! 🚀
```

---

## 🔗 কানেকশন পুলিং

### 📖 কানেকশন পুলিং কী?

প্রতিটি ডাটাবেস কানেকশন তৈরি করতে সময় লাগে (TCP handshake, authentication, ইত্যাদি)। কানেকশন পুলিং হলো আগে থেকেই কিছু কানেকশন তৈরি করে রাখা এবং বারবার ব্যবহার করা।

### ASCII ডায়াগ্রাম: কানেকশন পুলিং আর্কিটেকচার

```
❌ পুলিং ছাড়া (প্রতিবার নতুন কানেকশন):

  Request 1 ──→ [New Connection] ──→ DB ──→ [Close] ──→ Response
  Request 2 ──→ [New Connection] ──→ DB ──→ [Close] ──→ Response
  Request 3 ──→ [New Connection] ──→ DB ──→ [Close] ──→ Response
  Request 4 ──→ [New Connection] ──→ DB ──→ [Close] ──→ Response

  প্রতিটি কানেকশনে ~50ms overhead! 😢

✅ কানেকশন পুল দিয়ে (reuse করা হয়):

                         ┌─────────────────────────────┐
                         │      Connection Pool         │
                         │   ┌─────┐ ┌─────┐ ┌─────┐   │
  Request 1 ──→ Acquire ─┤   │Conn1│ │Conn2│ │Conn3│   ├──→ DB
  Request 2 ──→ Acquire ─┤   │ ✅  │ │ ✅  │ │ 💤  │   │
  Request 3 ──→ Wait... ─┤   └─────┘ └─────┘ └─────┘   │
  Request 4 ──→ Wait... ─┤   min: 2   max: 10          │
                         └─────────────────────────────┘

  কানেকশন reuse হয় → overhead নেই! 🎉
```

### ✅ PHP — PDO Persistent Connection

```php
// ✅ Laravel database.php — কানেকশন পুলিং কনফিগারেশন
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'database' => env('DB_DATABASE', 'bkash_db'),
    'username' => env('DB_USERNAME', 'root'),
    'password' => env('DB_PASSWORD', ''),
    'options' => [
        PDO::ATTR_PERSISTENT => true, // ← Persistent কানেকশন চালু
        PDO::ATTR_EMULATE_PREPARES => false,
    ],
],

// ✅ Raw PDO দিয়ে persistent connection
$pdo = new PDO(
    'mysql:host=localhost;dbname=nagad_db',
    'user',
    'password',
    [PDO::ATTR_PERSISTENT => true] // কানেকশন বন্ধ হবে না, reuse হবে
);
```

### ✅ Node.js — pg-pool (PostgreSQL)

```javascript
const { Pool } = require('pg');

// ✅ PostgreSQL কানেকশন পুল
const pool = new Pool({
    host: 'localhost',
    database: 'pathao_db',
    user: 'admin',
    password: 'secret',
    max: 20,               // সর্বোচ্চ ২০টি কানেকশন
    min: 5,                // সর্বনিম্ন ৫টি কানেকশন সবসময় থাকবে
    idleTimeoutMillis: 30000,    // ৩০ সেকেন্ড idle থাকলে কানেকশন বন্ধ
    connectionTimeoutMillis: 2000 // ২ সেকেন্ডের মধ্যে কানেকশন না পেলে error
});

// ব্যবহার
const result = await pool.query('SELECT * FROM rides WHERE status = $1', ['active']);
```

### ✅ Node.js — mysql2 Pool

```javascript
const mysql = require('mysql2/promise');

// ✅ MySQL কানেকশন পুল
const pool = mysql.createPool({
    host: 'localhost',
    database: 'daraz_db',
    user: 'root',
    password: 'password',
    waitForConnections: true,  // পুল ফুল হলে অপেক্ষা করবে
    connectionLimit: 20,       // সর্বোচ্চ ২০টি কানেকশন
    queueLimit: 0,             // অপেক্ষার queue-তে সীমা নেই
    enableKeepAlive: true,     // কানেকশন alive রাখবে
    keepAliveInitialDelay: 0
});

// ব্যবহার
const [rows] = await pool.execute('SELECT * FROM products WHERE category = ?', ['electronics']);
```

### 📊 কানেকশন পুলিং এর প্রভাব

```
bKash API Server — ১০,০০০ concurrent রিকোয়েস্ট:

❌ পুল ছাড়া:                    ✅ পুল সহ:
┌──────────────────────┐       ┌──────────────────────┐
│ Connections: 10,000  │       │ Connections: 20      │
│ Conn Time:  50ms/ea  │       │ Conn Time:  0ms      │
│ Total Wait: 500s     │       │ Total Wait: 50ms     │
│ Memory:     2GB      │       │ Memory:     100MB    │
│ DB Crashes: হ্যাঁ 💀  │       │ DB Crashes: না ✅    │
└──────────────────────┘       └──────────────────────┘
```

---

## 📝 কোয়েরি অপটিমাইজেশন

### 📖 ইনডেক্স স্ট্র্যাটেজি

ইনডেক্স হলো বইয়ের সূচিপত্রের মতো — পুরো বই পড়ার বদলে সরাসরি পেজ নম্বরে যাওয়া যায়।

```
❌ ইনডেক্স ছাড়া (Full Table Scan):
┌────────────────────────────────────────────────┐
│ Table: users (১০ লক্ষ rows)                     │
│                                                │
│  Row 1 → চেক ❌                                │
│  Row 2 → চেক ❌                                │
│  Row 3 → চেক ❌                                │
│  ...                                           │
│  Row 500,000 → চেক ✅ পাওয়া গেছে!              │
│  ...                                           │
│  Row 1,000,000 → চেক ❌ (তবুও সব দেখতে হবে)   │
│                                                │
│  সময়: ~500ms  😢                               │
└────────────────────────────────────────────────┘

✅ ইনডেক্স সহ (B-Tree Index Lookup):
┌────────────────────────────────────────────────┐
│ Index: email_idx                               │
│                                                │
│         [M]                                    │
│        /   \                                   │
│      [F]   [S]                                 │
│      / \   / \                                 │
│    [A] [J][P] [Z]                              │
│         ↓                                      │
│    Row 500,000 → সরাসরি পাওয়া! ✅              │
│                                                │
│  সময়: ~2ms  🚀                                 │
└────────────────────────────────────────────────┘
```

### ✅ ইনডেক্স তৈরির উদাহরণ

```sql
-- ✅ সবচেয়ে বেশি সার্চ হওয়া কলামে ইনডেক্স দিন
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- ✅ Composite Index — যখন একসাথে দুটি কলামে WHERE চলে
-- Daraz: ক্যাটাগরি + স্ট্যাটাস দিয়ে ফিল্টার
CREATE INDEX idx_products_cat_status ON products(category_id, status);

-- ✅ Partial Index (PostgreSQL) — শুধু active ডেটাতে ইনডেক্স
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

### 📊 EXPLAIN দিয়ে কোয়েরি বিশ্লেষণ

```sql
-- EXPLAIN ব্যবহার করে দেখুন কোয়েরি কীভাবে execute হচ্ছে
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 12345;
```

```
❌ খারাপ EXPLAIN output (Full Table Scan):
┌─────────────────────────────────────────────────────────────┐
│ Seq Scan on orders  (cost=0.00..25432.00 rows=1000000)      │
│   Filter: (customer_id = 12345)                             │
│   Rows Removed by Filter: 999990                            │
│   Planning Time: 0.1ms                                      │
│   Execution Time: 450.234ms  ← ❌ অনেক ধীর!                │
└─────────────────────────────────────────────────────────────┘

✅ ভালো EXPLAIN output (Index Scan):
┌─────────────────────────────────────────────────────────────┐
│ Index Scan using idx_orders_customer on orders              │
│   Index Cond: (customer_id = 12345)                         │
│   Planning Time: 0.05ms                                     │
│   Execution Time: 0.832ms  ← ✅ অনেক দ্রুত!                │
└─────────────────────────────────────────────────────────────┘
```

### ❌ SELECT * এড়িয়ে চলুন

```php
// ❌ সব কলাম নেওয়া — অপ্রয়োজনীয় ডেটা ট্রান্সফার
$users = User::all(); // SELECT * FROM users
// ১০০টি কলাম থাকলে সবগুলোই আসবে!

// ✅ শুধু দরকারি কলাম সিলেক্ট করুন
$users = User::select('id', 'name', 'email')->get();
// SELECT id, name, email FROM users
```

```javascript
// ❌ Node.js — সব কলাম
const users = await User.findAll();

// ✅ শুধু দরকারি কলাম
const users = await User.findAll({
    attributes: ['id', 'name', 'email']
});
```

### ✅ কোয়েরি ক্যাশিং (Laravel)

```php
// ✅ কোয়েরি রেজাল্ট ক্যাশ করুন — বারবার DB-তে যেতে হবে না
use Illuminate\Support\Facades\Cache;

// Grameenphone MyGP — প্ল্যান লিস্ট ক্যাশ (প্রতি ঘণ্টায় পরিবর্তন হয়)
$plans = Cache::remember('gp_plans', 3600, function () {
    return Plan::where('status', 'active')
               ->select('id', 'name', 'price', 'data_limit')
               ->get();
});

// ✅ ট্যাগ ব্যবহার করে ক্যাশ ম্যানেজমেন্ট
$products = Cache::tags(['products', 'electronics'])->remember('electronics_list', 1800, function () {
    return Product::where('category', 'electronics')
                  ->where('stock', '>', 0)
                  ->get();
});

// ক্যাশ পরিষ্কার করা (যখন প্রোডাক্ট আপডেট হয়)
Cache::tags(['products'])->flush();
```

---

## ⚡ অ্যাসিঙ্ক প্রসেসিং ও জব কিউ

### 📖 কেন জব কিউ দরকার?

কিছু কাজ আছে যেগুলো সময়সাপেক্ষ কিন্তু ইউজারকে তাৎক্ষণিক রেসপন্স দেওয়া দরকার। এই কাজগুলো ব্যাকগ্রাউন্ডে জব কিউ-তে পাঠিয়ে দেওয়া হয়।

```
রিকোয়েস্ট ফ্লো — জব কিউ ছাড়া বনাম সহ:

❌ কিউ ছাড়া (সিঙ্ক্রোনাস):
┌──────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ User │──→│ API Call │──→│Send Email│──→│ Generate │──→│ Response │
│      │   │  (50ms)  │   │ (2000ms) │   │  PDF     │   │ (4050ms) │
│      │   │          │   │          │   │ (2000ms) │   │  ❌ ধীর! │
└──────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘

ইউজার ৪ সেকেন্ড অপেক্ষা করছে! 😢

✅ কিউ সহ (অ্যাসিঙ্ক্রোনাস):
┌──────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ User │──→│ API Call │──→│ Push to  │──→│ Response │
│      │   │  (50ms)  │   │  Queue   │   │  (80ms)  │
│      │   │          │   │  (30ms)  │   │  ✅ দ্রুত!│
└──────┘   └──────────┘   └──────────┘   └──────────┘
                                │
              ┌─────────────────┘ (ব্যাকগ্রাউন্ডে চলবে)
              ▼
    ┌──────────────────────┐
    │     Queue Worker     │
    │  ┌────────────────┐  │
    │  │ Send Email     │  │
    │  │ Generate PDF   │  │
    │  │ Process Image  │  │
    │  └────────────────┘  │
    └──────────────────────┘
```

### কোন কাজগুলো কিউ-তে পাঠাবেন?

```
✅ কিউ-তে পাঠানো উচিত:           ❌ কিউ-তে পাঠানো উচিত না:
┌──────────────────────────┐    ┌──────────────────────────┐
│ • ইমেইল পাঠানো           │    │ • ইউজার লগইন             │
│ • SMS নোটিফিকেশন         │    │ • ব্যালেন্স চেক           │
│ • PDF/রিপোর্ট জেনারেশন   │    │ • প্রোডাক্ট সার্চ         │
│ • ইমেজ রিসাইজ/কম্প্রেশন  │    │ • কার্ট আপডেট            │
│ • ব্যাচ ডেটা প্রসেসিং     │    │ • পেমেন্ট verification   │
│ • থার্ড-পার্টি API কল     │    │ • রিয়েল-টাইম ক্যালকুলেশন │
│ • ডেটা এক্সপোর্ট (CSV)   │    │                          │
└──────────────────────────┘    └──────────────────────────┘
```

### ✅ Laravel Queue — জব তৈরি এবং ডিসপ্যাচ

```php
// ✅ জব ক্লাস তৈরি করুন
// Daraz — অর্ডার কনফার্মেশন ইমেইল
class SendOrderEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public Order $order
    ) {}

    public function handle(): void
    {
        Mail::to($this->order->customer)->send(
            new OrderConfirmation($this->order)
        );
    }

    // ব্যর্থ হলে ৩ বার retry করবে
    public int $tries = 3;

    // retry-র মাঝে ৬০ সেকেন্ড অপেক্ষা
    public int $backoff = 60;

    public function failed(\Throwable $exception): void
    {
        Log::error("Order email failed: {$this->order->id}", [
            'error' => $exception->getMessage()
        ]);
    }
}

// ✅ জব ডিসপ্যাচ করা
SendOrderEmail::dispatch($order);

// ✅ ডিলে দিয়ে ডিসপ্যাচ (৫ মিনিট পরে)
SendOrderEmail::dispatch($order)->delay(now()->addMinutes(5));

// ✅ নির্দিষ্ট কিউ-তে পাঠানো
SendOrderEmail::dispatch($order)->onQueue('emails');
```

### ✅ BullMQ — Node.js জব কিউ

```javascript
const { Queue, Worker } = require('bullmq');
const IORedis = require('ioredis');

const connection = new IORedis({ host: '127.0.0.1', port: 6379 });

// ✅ কিউ তৈরি
const emailQueue = new Queue('email', { connection });

// ✅ জব যোগ করা — Pathao রাইড কনফার্মেশন
await emailQueue.add('sendRideConfirmation', {
    orderId: 123,
    riderName: 'Rahim',
    destination: 'Dhanmondi, Dhaka'
}, {
    attempts: 3,               // ৩ বার retry
    backoff: { type: 'exponential', delay: 5000 }, // exponential backoff
    removeOnComplete: 100,     // শেষ ১০০টি completed জব রাখবে
    removeOnFail: 200          // শেষ ২০০টি failed জব রাখবে
});

// ✅ Worker — জব প্রসেস করা
const worker = new Worker('email', async (job) => {
    console.log(`Processing job ${job.id}: ${job.name}`);

    if (job.name === 'sendRideConfirmation') {
        await sendEmail(job.data.orderId);
    }
}, {
    connection,
    concurrency: 5 // একসাথে ৫টি জব প্রসেস করবে
});

worker.on('completed', (job) => {
    console.log(`Job ${job.id} completed`);
});

worker.on('failed', (job, err) => {
    console.error(`Job ${job.id} failed:`, err.message);
});
```

---

## 🗜️ রেসপন্স কম্প্রেশন

### 📖 কম্প্রেশন কেন দরকার?

API রেসপন্সের সাইজ কমালে নেটওয়ার্ক ব্যান্ডউইথ কম লাগে এবং ইউজার দ্রুত রেসপন্স পায়। বাংলাদেশে যেখানে মোবাইল ইন্টারনেটের স্পিড সবসময় ভালো নয়, সেখানে এটি খুবই গুরুত্বপূর্ণ।

### gzip vs Brotli তুলনা

```
┌──────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য    │      gzip        │     Brotli       │
├──────────────────┼──────────────────┼──────────────────┤
│ কম্প্রেশন রেশিও  │ ভালো (60-70%)    │ আরও ভালো (70-80%)│
│ কম্প্রেশন স্পিড  │ দ্রুত            │ কিছুটা ধীর       │
│ ডিকম্প্রেশন      │ দ্রুত            │ দ্রুত             │
│ ব্রাউজার সাপোর্ট │ সবাই ✅          │ আধুনিক ✅        │
│ CPU ব্যবহার      │ কম               │ বেশি             │
│ সেরা ব্যবহার     │ Dynamic content  │ Static assets    │
└──────────────────┴──────────────────┴──────────────────┘

📊 সাইজ তুলনা (১০০ KB JSON রেসপন্স):
┌───────────────┬──────────┬──────────────┐
│   পদ্ধতি      │  সাইজ    │  সেভ %       │
├───────────────┼──────────┼──────────────┤
│ কম্প্রেশন ছাড়া│ 100 KB  │  0%          │
│ gzip          │  32 KB   │  68%         │
│ Brotli        │  25 KB   │  75%         │
└───────────────┴──────────┴──────────────┘
```

### ✅ Express.js — কম্প্রেশন মিডলওয়্যার

```javascript
const express = require('express');
const compression = require('compression');

const app = express();

// ✅ gzip কম্প্রেশন চালু
app.use(compression({
    level: 6,                    // কম্প্রেশন লেভেল (1-9, 6 হলো balanced)
    threshold: 1024,             // ১ KB এর বেশি হলেই কম্প্রেস করবে
    filter: (req, res) => {
        // JSON এবং text রেসপন্স কম্প্রেস করবে
        if (req.headers['x-no-compression']) return false;
        return compression.filter(req, res);
    }
}));

// ✅ Brotli সাপোর্ট সহ (Node.js 11.7+)
const zlib = require('zlib');

app.use(compression({
    threshold: 1024,
    brotli: {
        enabled: true,
        zlib: {
            params: {
                [zlib.constants.BROTLI_PARAM_QUALITY]: 4 // 0-11, ৪ হলো balanced
            }
        }
    }
}));
```

### ✅ Laravel — কম্প্রেশন মিডলওয়্যার

```php
// ✅ app/Http/Middleware/CompressResponse.php
class CompressResponse
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        $acceptEncoding = $request->header('Accept-Encoding', '');

        if (str_contains($acceptEncoding, 'gzip') && function_exists('gzencode')) {
            $content = $response->getContent();

            // ১ KB এর বেশি হলেই কম্প্রেস করবে
            if (strlen($content) > 1024) {
                $compressed = gzencode($content, 6);
                $response->setContent($compressed);
                $response->headers->set('Content-Encoding', 'gzip');
                $response->headers->set('Content-Length', strlen($compressed));
            }
        }

        return $response;
    }
}

// Kernel.php-তে যোগ করুন
protected $middleware = [
    \App\Http\Middleware\CompressResponse::class,
];
```

> 💡 **প্রোডাকশন টিপ:** সাধারণত Nginx/Apache লেভেলে কম্প্রেশন করা ভালো। অ্যাপ্লিকেশন লেভেলে করলে CPU overhead বাড়ে।

```nginx
# ✅ Nginx কনফিগারেশন — সবচেয়ে ভালো পদ্ধতি
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain application/json application/javascript text/css;
gzip_comp_level 6;

# Brotli (nginx-brotli মডিউল লাগবে)
brotli on;
brotli_comp_level 4;
brotli_types text/plain application/json application/javascript text/css;
```

---

## 📄 পেজিনেশন অপটিমাইজেশন

### 📖 Offset vs Cursor-based পেজিনেশন

পেজিনেশন মানে হলো বড় ডেটাসেট ছোট ছোট পেজে ভাগ করে দেখানো। কিন্তু সব পেজিনেশন সমান নয়!

### ASCII ডায়াগ্রাম: Offset vs Cursor

```
❌ Offset Pagination (OFFSET 100000, LIMIT 20):

  ডাটাবেস যা করে:
  ┌─────────────────────────────────────────────────┐
  │ Row 1      → পড়ো, বাদ দাও                      │
  │ Row 2      → পড়ো, বাদ দাও                      │
  │ Row 3      → পড়ো, বাদ দাও                      │
  │ ...                                             │
  │ Row 99,999 → পড়ো, বাদ দাও   ← ১ লক্ষ row      │
  │ Row 100,000→ পড়ো, বাদ দাও      বৃথা পড়া হলো!  │
  │─────────────────────────────────────────────────│
  │ Row 100,001→ ✅ রিটার্ন করো                     │
  │ Row 100,002→ ✅ রিটার্ন করো                     │
  │ ...                                             │
  │ Row 100,020→ ✅ রিটার্ন করো                     │
  └─────────────────────────────────────────────────┘
  ১ লক্ষ row বৃথা পড়া! → ধীরগতি 😢

✅ Cursor Pagination (WHERE id > 100000 LIMIT 20):

  ডাটাবেস যা করে:
  ┌─────────────────────────────────────────────────┐
  │ Index থেকে সরাসরি id > 100000 এ যাও (B-Tree)  │
  │─────────────────────────────────────────────────│
  │ Row 100,001→ ✅ রিটার্ন করো                     │
  │ Row 100,002→ ✅ রিটার্ন করো                     │
  │ ...                                             │
  │ Row 100,020→ ✅ রিটার্ন করো                     │
  └─────────────────────────────────────────────────┘
  শুধু ২০টি row পড়া! → দ্রুতগতি 🚀
```

### ❌ Offset Pagination (Laravel — ধীরগতি)

```php
// ❌ Offset pagination — বড় ডেটাসেটে অনেক ধীর
// Daraz-এ ১০ লক্ষ প্রোডাক্ট থাকলে পেজ ৫০০০ লোড করতে অনেক সময় লাগবে
$products = Product::offset(100000)->limit(20)->get();
// SQL: SELECT * FROM products LIMIT 20 OFFSET 100000
// ডাটাবেস ১ লক্ষ row স্ক্যান করবে শুধু ২০টি রিটার্ন করতে!

// ❌ Laravel paginate — এটাও offset ব্যবহার করে
$products = Product::paginate(20); // পেজ ৫০০০ এ গেলে ধীর হবে
```

### ✅ Cursor Pagination (Laravel — দ্রুতগতি)

```php
// ✅ Cursor pagination — সব পেজে সমান দ্রুত!
$products = Product::where('id', '>', $lastId)
                    ->orderBy('id')
                    ->limit(20)
                    ->get();
// SQL: SELECT * FROM products WHERE id > 100000 ORDER BY id LIMIT 20
// ইনডেক্স ব্যবহার করে সরাসরি সঠিক জায়গায় যাবে!

// ✅ Laravel cursorPaginate (built-in)
$products = Product::orderBy('id')->cursorPaginate(20);

// ✅ API Response এ next cursor পাঠানো
return response()->json([
    'data' => $products,
    'next_cursor' => $products->last()?->id,
    'has_more' => $products->count() === 20
]);
```

### ✅ Cursor Pagination (Node.js)

```javascript
// ✅ Cursor-based pagination
app.get('/api/products', async (req, res) => {
    const cursor = req.query.cursor;  // শেষ প্রোডাক্টের ID
    const limit = parseInt(req.query.limit) || 20;

    const whereClause = cursor
        ? { id: { [Op.gt]: cursor } }
        : {};

    const products = await Product.findAll({
        where: whereClause,
        order: [['id', 'ASC']],
        limit: limit + 1,  // ১টি বেশি নিই has_more চেক করতে
        attributes: ['id', 'name', 'price', 'image']
    });

    const hasMore = products.length > limit;
    if (hasMore) products.pop();  // extra টি বাদ দিই

    res.json({
        data: products,
        next_cursor: products.length ? products[products.length - 1].id : null,
        has_more: hasMore
    });
});
```

### 📊 Offset vs Cursor পারফরম্যান্স তুলনা

```
Daraz প্রোডাক্ট টেবিল — ১০ লক্ষ row:

পেজ নম্বর    │  Offset সময়  │  Cursor সময়  │  পার্থক্য
─────────────┼──────────────┼──────────────┼───────────
পেজ ১        │    5ms       │    5ms       │  সমান
পেজ ১০       │    8ms       │    5ms       │  ১.৬x
পেজ ১০০      │   25ms       │    5ms       │  ৫x
পেজ ১,০০০    │   180ms      │    5ms       │  ৩৬x
পেজ ১০,০০০   │  1,200ms     │    5ms       │  ২৪০x
পেজ ৫০,০০০   │  5,500ms     │    5ms       │  ১,১০০x 😱

Cursor pagination সবসময় ~5ms! 🚀
```

---

## 📊 Before/After পারফরম্যান্স তুলনা

### সামগ্রিক অপটিমাইজেশনের ফলাফল

```
🏢 কেস স্টাডি: Daraz-এর মতো ই-কমার্স প্ল্যাটফর্ম

┌──────────────────────────┬─────────────────┬─────────────────┬────────────┐
│       মেট্রিক            │   আগে (❌)      │   পরে (✅)      │  উন্নতি    │
├──────────────────────────┼─────────────────┼─────────────────┼────────────┤
│ প্রোডাক্ট লিস্ট API     │   2,500ms       │   120ms         │  20x ⬆️   │
│ অর্ডার হিস্ট্রি API      │   3,200ms       │   200ms         │  16x ⬆️   │
│ সার্চ API                │   1,800ms       │   150ms         │  12x ⬆️   │
│ DB কোয়েরি/সেকেন্ড        │   500           │   50            │  10x ⬇️   │
│ সার্ভার মেমরি            │   4 GB          │   1 GB          │  4x ⬇️    │
│ রেসপন্স সাইজ (avg)      │   250 KB        │   65 KB         │  3.8x ⬇️  │
│ Concurrent ইউজার         │   1,000         │   15,000        │  15x ⬆️   │
│ সার্ভার খরচ/মাস          │   $500          │   $150          │  3.3x ⬇️  │
└──────────────────────────┴─────────────────┴─────────────────┴────────────┘
```

### প্রতিটি অপটিমাইজেশনের আলাদা প্রভাব

```
📊 কোন অপটিমাইজেশন কতটা প্রভাব ফেলেছে:

N+1 সমাধান         ████████████████████████ 40% উন্নতি
ইনডেক্সিং           ██████████████████░░░░░ 25% উন্নতি
কানেকশন পুলিং       ████████████░░░░░░░░░░░ 15% উন্নতি
কম্প্রেশন            ██████████░░░░░░░░░░░░░ 10% উন্নতি
কার্সর পেজিনেশন     ████░░░░░░░░░░░░░░░░░░░  5% উন্নতি
জব কিউ              ████░░░░░░░░░░░░░░░░░░░  5% উন্নতি
                    ─────────────────────────
                              মোট: ~100% 🚀
```

### bKash ট্রানজেকশন API — Before/After

```
❌ অপটিমাইজেশনের আগে:

  User ──→ API (200ms) ──→ Auth Check (150ms) ──→ DB Query (800ms)
       ──→ Balance Check (500ms) ──→ Transaction (600ms)
       ──→ Notification (1000ms) ──→ Response

  মোট সময়: 3,250ms 😱
  প্রতি সেকেন্ডে: ৩০ ট্রানজেকশন

✅ অপটিমাইজেশনের পরে:

  User ──→ API (50ms) ──→ Cached Auth (10ms) ──→ Indexed Query (30ms)
       ──→ Pooled DB (20ms) ──→ Transaction (50ms)
       ──→ Queue Notification ──→ Response

  মোট সময়: 180ms 🚀
  প্রতি সেকেন্ডে: ৫৫০ ট্রানজেকশন (18x বেশি!)
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### N+1 সমাধান (Eager Loading)

```
✅ কখন ব্যবহার করবেন:
  • যখন সম্পর্কিত ডেটা (relationships) লুপে ব্যবহার হচ্ছে
  • লিস্ট পেজে যেখানে related ডেটা দেখানো হয়
  • API রেসপন্সে nested ডেটা থাকলে
  • Daraz প্রোডাক্ট লিস্টে ক্যাটাগরি, সেলার তথ্য দেখানোর সময়

❌ কখন করবেন না:
  • যখন শুধু একটি রেকর্ড ফেচ করছেন (single item)
  • যখন related ডেটা লাগছে না
  • যখন related টেবিলে অনেক বেশি ডেটা আছে (selective loading ব্যবহার করুন)
```

### কানেকশন পুলিং

```
✅ কখন ব্যবহার করবেন:
  • সবসময়! প্রোডাকশনে কানেকশন পুলিং অবশ্যই ব্যবহার করুন
  • High-traffic অ্যাপ্লিকেশনে (bKash, Pathao)
  • মাইক্রোসার্ভিসে যেখানে অনেক সার্ভিস DB-তে কানেক্ট করে

❌ কখন করবেন না:
  • ওয়ান-টাইম স্ক্রিপ্ট বা CLI কমান্ডে (pool overhead অপ্রয়োজনীয়)
  • টেস্টিং-এ যেখানে isolation দরকার
  • খুব কম ট্রাফিকের অ্যাপে (pool maintain করা overhead হতে পারে)
```

### জব কিউ

```
✅ কখন ব্যবহার করবেন:
  • সময়সাপেক্ষ কাজ: ইমেইল, SMS, PDF জেনারেশন
  • থার্ড-পার্টি API কল (পেমেন্ট গেটওয়ে callback)
  • ডেটা প্রসেসিং: CSV এক্সপোর্ট, রিপোর্ট তৈরি
  • ইমেজ প্রসেসিং: থাম্বনেইল তৈরি, ওয়াটারমার্ক

❌ কখন করবেন না:
  • যখন ইউজারকে তাৎক্ষণিক ফলাফল দেখাতে হবে
  • পেমেন্ট ভেরিফিকেশন (ইউজার অপেক্ষা করবে)
  • সিম্পল CRUD অপারেশন
  • ১০০ms এর কম সময়ের কাজ
```

### কার্সর পেজিনেশন

```
✅ কখন ব্যবহার করবেন:
  • বড় ডেটাসেট (১ লক্ষ+ rows)
  • Infinite scroll UI
  • মোবাইল অ্যাপ API (Pathao রাইড হিস্ট্রি)
  • রিয়েল-টাইম ফিড (নতুন ডেটা আসতে থাকে)

❌ কখন করবেন না:
  • যখন ইউজারকে নির্দিষ্ট পেজে jump করতে হবে (Page 5, Page 10)
  • ছোট ডেটাসেট (১০০০ এর কম rows)
  • অ্যাডমিন প্যানেলে যেখানে টোটাল কাউন্ট এবং পেজ নম্বর দেখাতে হয়
  • কমপ্লেক্স সর্টিং যেখানে cursor maintain করা কঠিন
```

### রেসপন্স কম্প্রেশন

```
✅ কখন ব্যবহার করবেন:
  • JSON API রেসপন্স (সাধারণত 50-80% কমে)
  • HTML পেজ
  • CSS/JS ফাইল (static assets)
  • বড় রেসপন্স (>1KB)

❌ কখন করবেন না:
  • ইমেজ, ভিডিও (ইতিমধ্যে compressed)
  • খুব ছোট রেসপন্স (<1KB — overhead > benefit)
  • রিয়েল-টাইম স্ট্রিমিং ডেটা (latency বাড়ে)
  • Binary ডেটা (protobuf, ইতিমধ্যে compact)
```

---

## 📖 সারসংক্ষেপ (Summary)

```
ব্যাকেন্ড অপটিমাইজেশন চেকলিস্ট:

  ☐ N+1 কোয়েরি চেক করুন → Eager Loading ব্যবহার করুন
  ☐ স্লো কোয়েরি খুঁজুন → EXPLAIN দিয়ে বিশ্লেষণ করুন
  ☐ দরকারি কলামে Index দিন → Composite Index চিন্তা করুন
  ☐ SELECT * বাদ দিন → শুধু দরকারি কলাম নিন
  ☐ কানেকশন পুলিং চালু করুন → সঠিক pool size সেট করুন
  ☐ সময়সাপেক্ষ কাজ Queue-তে পাঠান → Worker দিয়ে প্রসেস করুন
  ☐ রেসপন্স কম্প্রেশন চালু করুন → Nginx লেভেলে করুন
  ☐ বড় লিস্টে Cursor Pagination ব্যবহার করুন
  ☐ কোয়েরি ক্যাশিং চালু করুন → TTL সঠিকভাবে সেট করুন
```

```
🎯 মনে রাখুন:

  "Premature optimization is the root of all evil"
                                    — Donald Knuth

  কিন্তু...

  "Knowing WHERE to optimize is the root of all performance"
                                    — Every Senior Developer

  ┌──────────────────────────────────────────────────────┐
  │  ১. আগে Measure করুন (profiling, EXPLAIN)           │
  │  ২. Bottleneck চিহ্নিত করুন                         │
  │  ৩. সবচেয়ে বেশি impact-এর জায়গায় optimize করুন     │
  │  ৪. আবার Measure করুন — উন্নতি হয়েছে কিনা দেখুন    │
  └──────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী পড়ুন:** ক্যাশিং স্ট্র্যাটেজি (Redis, Memcached), ডাটাবেস শার্ডিং, লোড ব্যালান্সিং
