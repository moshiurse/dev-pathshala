# ⚡ SQL অপটিমাইজেশন (Query Performance Tuning)

## 📌 সংজ্ঞা ও মূল ধারণা

SQL Query Optimization হলো database query-গুলোকে এমনভাবে লেখা ও tune করা যাতে সর্বনিম্ন resource (CPU, memory, I/O) ব্যবহার করে সর্বদ্রুত result পাওয়া যায়। একজন senior engineer-এর জন্য শুধু "query কাজ করছে" যথেষ্ট নয় — query কতটা efficiently কাজ করছে সেটাই আসল বিষয়।

### Query Lifecycle

প্রতিটি SQL query execute হওয়ার আগে কয়েকটি ধাপের মধ্য দিয়ে যায়:

```
Client Request → Parse → Optimize → Execute → Fetch → Return
     │              │         │          │        │
     │         Syntax &    Cost      Storage   Network
     │         Semantic   Analysis   Engine    Transfer
     │         Check      + Plan     Access
     │                    Selection
```

**১. Parse Phase:** Query-এর syntax ও semantic validity check হয়। Table, column exist কিনা verify হয়।
**২. Optimize Phase:** Query optimizer বিভিন্ন execution plan তৈরি করে এবং সবচেয়ে কম cost-এর plan বাছাই করে।
**৩. Execute Phase:** Storage engine থেকে data read/write হয়। Index ব্যবহার হয় কি না সেটা এখানে determine হয়।
**৪. Fetch Phase:** Result set client-এ transfer হয়। Row count, network latency এখানে গুরুত্বপূর্ণ।

### Query Optimizer Internals

Database-এর query optimizer একটি sophisticated component যা internally কাজ করে:

```sql
-- Optimizer এই simple query-এর জন্যও অনেকগুলো plan consider করে
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed' AND u.city = 'Dhaka';

-- Optimizer যা consider করে:
-- ১. users আগে scan করবে নাকি orders?
-- ২. কোন index ব্যবহার করবে?
-- ৩. Nested Loop, Hash Join নাকি Merge Join?
-- ৪. WHERE condition কোন order-এ apply করবে?
```

Optimizer statistics-এর উপর নির্ভর করে — table-এ কতগুলো row আছে, column-এ কতগুলো distinct value আছে (cardinality), index-এর selectivity কত — এসব information ব্যবহার করে best plan নির্ধারণ করে।

### Cost-Based vs Rule-Based Optimization

| বৈশিষ্ট্য | Cost-Based (CBO) | Rule-Based (RBO) |
|---|---|---|
| সিদ্ধান্ত ভিত্তি | Statistics, cardinality | নির্দিষ্ট rule set |
| আধুনিকতা | Modern (MySQL 8, PostgreSQL) | Legacy (Oracle পুরানো version) |
| নমনীয়তা | Data distribution অনুযায়ী পরিবর্তন হয় | সবসময় একই plan |
| Statistics প্রয়োজন | হ্যাঁ (`ANALYZE TABLE`) | না |
| সঠিকতা | সাধারণত বেশি accurate | Simple query-তে ভালো, complex-এ দুর্বল |

```sql
-- Statistics update করুন (MySQL)
ANALYZE TABLE orders;
ANALYZE TABLE users;

-- PostgreSQL-এ
ANALYZE orders;
ANALYZE users;
```

---

## 📊 EXPLAIN Deep Dive

EXPLAIN হলো query optimization-এর সবচেয়ে শক্তিশালী হাতিয়ার। এটি query execute না করেই execution plan দেখায়।

### MySQL EXPLAIN Output

```sql
EXPLAIN SELECT u.name, o.total, p.name as product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01'
  AND u.city = 'Dhaka'
  AND o.status = 'completed';
```

Output columns বিস্তারিত:

| Column | বর্ণনা | গুরুত্বপূর্ণ মান |
|---|---|---|
| `id` | SELECT identifier, সমান id মানে একই level | Higher id আগে execute হয় |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DERIVED | DEPENDENT SUBQUERY avoid করুন |
| `table` | কোন table access হচ্ছে | NULL মানে optimizer shortcut নিয়েছে |
| `type` | Access method (সবচেয়ে গুরুত্বপূর্ণ!) | system > const > eq_ref > ref > range > index > ALL |
| `possible_keys` | ব্যবহারযোগ্য index-এর list | NULL হলে index নেই |
| `key` | আসলে যে index ব্যবহৃত হচ্ছে | NULL মানে কোনো index ব্যবহার হচ্ছে না |
| `key_len` | Index-এর কতটুকু ব্যবহৃত হচ্ছে | Composite index-এ কত column ব্যবহৃত |
| `rows` | Estimated rows to examine | যত কম তত ভালো |
| `filtered` | WHERE condition-এ কত % row filter হবে | ১০০% মানে সব row qualify করে |
| `Extra` | অতিরিক্ত তথ্য | "Using filesort", "Using temporary" সমস্যার ইঙ্গিত |

### PostgreSQL EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.city = 'Dhaka'
  AND o.created_at >= '2024-01-01'
GROUP BY u.id, u.name
HAVING SUM(o.total) > 10000
ORDER BY total_spent DESC
LIMIT 20;

-- Output example:
-- Limit  (cost=1250.43..1250.48 rows=20 width=48) (actual time=15.234..15.240 rows=20 loops=1)
--   ->  Sort  (cost=1250.43..1256.18 rows=2300 width=48) (actual time=15.232..15.236 rows=20 loops=1)
--         Sort Key: (sum(o.total)) DESC
--         Sort Method: top-N heapsort  Memory: 27kB
--         ->  HashAggregate  (cost=1180.50..1215.00 rows=2300 width=48) (actual time=14.567..14.890 rows=1847 loops=1)
--               Group Key: u.id
--               Batches: 1  Memory Usage: 369kB
--               Filter: (sum(o.total) > 10000)
--               ->  Hash Join  (cost=125.00..1065.00 rows=23100 width=20) (actual time=1.234..10.567 rows=23100 loops=1)
--                     Buffers: shared hit=456 read=23
```

**গুরুত্বপূর্ণ metrics:**
- `actual time` — প্রকৃত execution time (milliseconds)
- `rows` — আসলে কতটি row process হয়েছে
- `loops` — এই node কতবার execute হয়েছে
- `Buffers: shared hit` — cache থেকে পড়া pages (বেশি ভালো)
- `Buffers: shared read` — disk থেকে পড়া pages (কম ভালো)

### Access Type বিশ্লেষণ (ভালো → খারাপ)

```
const       → Primary key / unique index lookup (১টি row)
eq_ref      → JOIN-এ unique index lookup (প্রতি combination-এ ১টি row)
ref         → Non-unique index lookup (কিছু rows)
range       → Index range scan (BETWEEN, >, <, IN)
index       → Full index scan (সব index entry)
ALL         → Full table scan (সবচেয়ে খারাপ — পুরো table পড়ে)
```

### Bottleneck চিহ্নিতকরণ

```sql
-- ❌ সমস্যাজনক EXPLAIN output-এর লক্ষণ:
-- type: ALL                     → Full table scan
-- Extra: Using filesort         → Memory/disk-এ sort করছে
-- Extra: Using temporary        → Temporary table তৈরি করছে
-- rows: 1000000+                → অনেক বেশি row scan করছে
-- key: NULL                     → কোনো index ব্যবহার হচ্ছে না
-- select_type: DEPENDENT SUBQUERY → প্রতি row-এ subquery চলছে

-- ✅ ভালো EXPLAIN output-এর লক্ষণ:
-- type: const, eq_ref, ref      → Index ব্যবহার হচ্ছে
-- Extra: Using index            → Covering index (table access নেই)
-- rows: ছোট সংখ্যা              → কম row scan হচ্ছে
-- Extra: Using index condition   → Index condition pushdown
```

---

## 💻 Optimization Techniques

### ১. SELECT Optimization

#### SELECT * কেন ক্ষতিকর

`SELECT *` ব্যবহারে যে সমস্যাগুলো হয়:
- **অপ্রয়োজনীয় I/O:** বড় TEXT/BLOB column পড়া হয় যা প্রয়োজন নেই
- **Covering index নষ্ট:** Index-এ সব column না থাকলে table lookup প্রয়োজন
- **Network bandwidth অপচয়:** অতিরিক্ত data transfer
- **Schema change fragility:** নতুন column যোগ হলে application break করতে পারে

```sql
-- ❌ খারাপ: সব column পড়ছে (products table-এ description TEXT, images JSON থাকতে পারে)
SELECT * FROM products WHERE category_id = 5;

-- ✅ ভালো: শুধু প্রয়োজনীয় column
SELECT id, name, price, stock_quantity
FROM products
WHERE category_id = 5;

-- ✅ Covering index তৈরি করুন
CREATE INDEX idx_products_category_covering
ON products(category_id, id, name, price, stock_quantity);
-- এখন table access ছাড়াই শুধু index থেকে data আসবে (Using index)
```

**Laravel:**

```php
// ❌ সব column
$products = Product::where('category_id', 5)->get();

// ✅ নির্দিষ্ট column
$products = Product::where('category_id', 5)
    ->select(['id', 'name', 'price', 'stock_quantity'])
    ->get();

// ✅ শুধু একটি column-এর array দরকার হলে
$productNames = Product::where('category_id', 5)->pluck('name');

// ✅ Key-value pair
$productPrices = Product::where('category_id', 5)->pluck('price', 'id');
```

**Sequelize:**

```javascript
// ❌ সব column
const products = await Product.findAll({
  where: { categoryId: 5 }
});

// ✅ নির্দিষ্ট column
const products = await Product.findAll({
  attributes: ['id', 'name', 'price', 'stockQuantity'],
  where: { categoryId: 5 }
});

// ✅ Column exclude করুন (বড় column বাদ দিতে)
const products = await Product.findAll({
  attributes: { exclude: ['description', 'images'] },
  where: { categoryId: 5 }
});

// ✅ Aggregation সহ
const products = await Product.findAll({
  attributes: [
    'categoryId',
    [sequelize.fn('COUNT', sequelize.col('id')), 'productCount'],
    [sequelize.fn('AVG', sequelize.col('price')), 'avgPrice']
  ],
  group: ['categoryId']
});
```

---

### ২. WHERE Clause Optimization

#### Sargable vs Non-sargable Queries

**Sargable** (Search ARGument ABLE) query মানে যে query index ব্যবহার করতে পারে। Column-এ function বা expression apply করলে query non-sargable হয়ে যায়।

```sql
-- ❌ Non-sargable: Column-এ function — index ব্যবহার হবে না
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users WHERE LOWER(email) = 'karim@example.com';
SELECT * FROM products WHERE price * 1.15 > 1000;
SELECT * FROM orders WHERE DATE(created_at) = '2024-06-15';

-- ✅ Sargable: Value-তে transformation — index ব্যবহার হবে
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

SELECT * FROM users WHERE email = 'karim@example.com';
-- অথবা generated column + index ব্যবহার করুন:
-- ALTER TABLE users ADD email_lower VARCHAR(255) GENERATED ALWAYS AS (LOWER(email));
-- CREATE INDEX idx_email_lower ON users(email_lower);

SELECT * FROM products WHERE price > 869.56; -- 1000 / 1.15

SELECT * FROM orders
WHERE created_at >= '2024-06-15 00:00:00'
  AND created_at < '2024-06-16 00:00:00';
```

#### OR vs UNION

```sql
-- ❌ OR ব্যবহারে optimizer প্রায়ই full table scan করে
SELECT * FROM products
WHERE category_id = 5 OR seller_id = 100;

-- ✅ UNION ALL ব্যবহারে প্রতিটি condition আলাদা index ব্যবহার করতে পারে
SELECT * FROM products WHERE category_id = 5
UNION ALL
SELECT * FROM products WHERE seller_id = 100 AND category_id != 5;
```

#### IN vs EXISTS vs JOIN

```sql
-- বাংলাদেশি e-commerce context: ঢাকায় order করা user-দের খুঁজুন

-- IN — ছোট subquery result-এ ভালো
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders WHERE city = 'Dhaka');

-- EXISTS — বড় outer table, ছোট lookup-এ ভালো (early termination)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.city = 'Dhaka'
);

-- JOIN — সাধারণত সবচেয়ে ভালো performance
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.city = 'Dhaka';
```

**সাধারণ নিয়ম:**
- `IN` → subquery result set ছোট হলে
- `EXISTS` → outer table ছোট, inner table বড় হলে (correlated subquery-তে)
- `JOIN` → বেশিরভাগ ক্ষেত্রে সবচেয়ে ভালো, optimizer-এর সবচেয়ে বেশি flexibility

#### NULL Handling

```sql
-- NULL comparison-এ সতর্কতা
-- IS NULL index ব্যবহার করতে পারে (MySQL তে)
SELECT * FROM orders WHERE deleted_at IS NULL; -- ✅ index ব্যবহৃত হয়

-- কিন্তু NULL-এর সাথে comparison অপ্রত্যাশিত ফলাফল দেয়
-- NULL != NULL, NULL = NULL দুটোই NULL (false) return করে
SELECT * FROM orders WHERE discount_code != 'SUMMER'; -- NULL row বাদ পড়বে!
SELECT * FROM orders WHERE discount_code != 'SUMMER' OR discount_code IS NULL; -- ✅ সঠিক
```

---

### ৩. JOIN Optimization

#### JOIN Type Performance

```sql
-- INNER JOIN: দুই table-এ match আছে এমন row (সবচেয়ে দ্রুত কারণ কম row return)
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: বাম table-এর সব row + ডান table-এ match (INNER-এর চেয়ে ধীর)
SELECT u.name, COALESCE(SUM(o.total), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- CROSS JOIN: Cartesian product — সাবধানে ব্যবহার করুন!
-- 1000 × 1000 = 1,000,000 rows!
```

#### JOIN Order ও ছোট Table দিয়ে Drive করা

```sql
-- MySQL optimizer সাধারণত নিজেই best order বেছে নেয়,
-- কিন্তু hint দেওয়া যায়:

-- ✅ ছোট result set থেকে বড় table-এ join করুন
-- categories (50 rows) → products (100,000 rows) → order_items (1,000,000 rows)
SELECT c.name, COUNT(oi.id)
FROM categories c                          -- ছোট table আগে
JOIN products p ON c.id = p.category_id    -- মাঝারি table
JOIN order_items oi ON p.id = oi.product_id -- বড় table
WHERE c.is_active = 1
GROUP BY c.id;

-- MySQL-এ hint:
SELECT /*+ JOIN_ORDER(c, p, oi) */ c.name, COUNT(oi.id)
FROM categories c
JOIN products p ON c.id = p.category_id
JOIN order_items oi ON p.id = oi.product_id
WHERE c.is_active = 1
GROUP BY c.id;
```

#### JOIN vs Subquery

```sql
-- ❌ Correlated subquery: প্রতিটি row-এর জন্য subquery চলে
SELECT p.name, p.price,
    (SELECT c.name FROM categories c WHERE c.id = p.category_id) as category_name
FROM products p;

-- ✅ JOIN: একবারে সব data আনে
SELECT p.name, p.price, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id;
```

**Laravel:**

```php
// ✅ Eager loading (separate query, কিন্তু efficient)
$orders = Order::with(['user:id,name,email', 'items.product:id,name,price'])
    ->where('status', 'completed')
    ->get();

// ✅ Raw JOIN যখন complex query দরকার
$topCustomers = DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.id', 'users.name', DB::raw('SUM(orders.total) as total_spent'))
    ->where('orders.created_at', '>=', '2024-01-01')
    ->groupBy('users.id', 'users.name')
    ->orderByDesc('total_spent')
    ->limit(20)
    ->get();
```

**Sequelize:**

```javascript
// Eager loading
const orders = await Order.findAll({
  where: { status: 'completed' },
  include: [
    { model: User, attributes: ['id', 'name', 'email'] },
    {
      model: OrderItem,
      include: [{ model: Product, attributes: ['id', 'name', 'price'] }]
    }
  ]
});

// subQuery: false দিলে single JOIN query হয় (ছোট dataset-এ ভালো)
const users = await User.findAll({
  include: [{ model: Order, where: { status: 'completed' } }],
  subQuery: false,
  limit: 20
});
```

---

### ৪. N+1 Query Problem

N+1 সমস্যা হলো database performance-এর সবচেয়ে সাধারণ এবং ক্ষতিকর সমস্যা। এটি তখন হয় যখন একটি query চালিয়ে N টি record আনা হয়, তারপর প্রতিটি record-এর জন্য আরেকটি আলাদা query চালানো হয়।

#### সমস্যা চিত্রিত

```sql
-- ধাপ ১: সব order আনো (1 query)
SELECT * FROM orders WHERE status = 'pending'; -- 100 টি order পাওয়া গেল

-- ধাপ ২: প্রতিটি order-এর user আনো (100 queries!)
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 3;
-- ... আরো 97 টি query!
-- মোট: 1 + 100 = 101 queries! 😱
```

#### Detection

```php
// Laravel Debugbar বা Telescope দিয়ে detect করুন
// config/app.php তে Debugbar enable করুন

// Query log enable করে দেখুন:
DB::enableQueryLog();

$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name; // প্রতিবার নতুন query!
}

$queries = DB::getQueryLog();
dd(count($queries)); // 101 দেখাবে!
```

#### সমাধান: Eager Loading

**Laravel (Bad → Good):**

```php
// ❌ N+1 সমস্যা: ১০০টি order-এ ১০১টি query
$orders = Order::where('status', 'pending')->get();
foreach ($orders as $order) {
    echo $order->user->name;         // N queries for users
    foreach ($order->items as $item) {
        echo $item->product->name;   // N*M queries for products!
    }
}

// ✅ Eager Loading: মাত্র ৪টি query!
$orders = Order::with(['user:id,name', 'items.product:id,name,price'])
    ->where('status', 'pending')
    ->get();
// Query 1: SELECT * FROM orders WHERE status = 'pending'
// Query 2: SELECT id, name FROM users WHERE id IN (1, 2, 3, ...)
// Query 3: SELECT * FROM order_items WHERE order_id IN (10, 20, 30, ...)
// Query 4: SELECT id, name, price FROM products WHERE id IN (5, 15, 25, ...)

foreach ($orders as $order) {
    echo $order->user->name;         // কোনো নতুন query নেই!
    foreach ($order->items as $item) {
        echo $item->product->name;   // কোনো নতুন query নেই!
    }
}

// ✅ withCount — relationship count আনতে (extra query নেই)
$categories = Category::withCount('products')
    ->having('products_count', '>', 10)
    ->get();
// SELECT categories.*, (SELECT COUNT(*) FROM products WHERE ...) as products_count

// ✅ Explicit Loading — পরে load করতে হলে
$orders = Order::where('status', 'pending')->get();
if ($needUserDetails) {
    $orders->load('user:id,name,email'); // তখনই load হবে
}

// ✅ preventLazyLoading — development-এ N+1 detect করতে
// AppServiceProvider::boot()
Model::preventLazyLoading(!app()->isProduction());
```

**Sequelize (Bad → Good):**

```javascript
// ❌ N+1 সমস্যা
const orders = await Order.findAll({ where: { status: 'pending' } });
for (const order of orders) {
  const user = await order.getUser();       // N queries!
  const items = await order.getOrderItems(); // N queries!
}

// ✅ Eager Loading
const orders = await Order.findAll({
  where: { status: 'pending' },
  include: [
    { model: User, attributes: ['id', 'name'] },
    {
      model: OrderItem,
      include: [{ model: Product, attributes: ['id', 'name', 'price'] }]
    }
  ]
});

for (const order of orders) {
  console.log(order.User.name);          // আগেই load হয়ে আছে
  for (const item of order.OrderItems) {
    console.log(item.Product.name);      // কোনো নতুন query নেই
  }
}
```

#### Lazy vs Eager vs Explicit Loading তুলনা

| পদ্ধতি | কখন Load হয় | Performance | ব্যবহার |
|---|---|---|---|
| Lazy Loading | Access করলে | ❌ N+1 ঝুঁকি | ছোট dataset, development |
| Eager Loading (`with()`) | Query-র সাথেই | ✅ Optimized | সবসময় relation দরকার হলে |
| Explicit Loading (`load()`) | ম্যানুয়ালি call করলে | ✅ Conditional | শর্তসাপেক্ষে relation দরকার হলে |

---

### ৫. Subquery Optimization

#### Correlated vs Non-correlated

```sql
-- Non-correlated: Subquery একবারই চলে (ভালো performance)
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Correlated: প্রতিটি outer row-এর জন্য subquery চলে (ধীর!)
SELECT p.name, p.price,
    (SELECT COUNT(*) FROM order_items oi WHERE oi.product_id = p.id) as order_count
FROM products p;

-- ✅ Correlated subquery-কে JOIN-এ রূপান্তর
SELECT p.name, p.price, COALESCE(oi_count.cnt, 0) as order_count
FROM products p
LEFT JOIN (
    SELECT product_id, COUNT(*) as cnt
    FROM order_items
    GROUP BY product_id
) oi_count ON p.id = oi_count.product_id;
```

#### EXISTS vs IN Performance

```sql
-- বাংলাদেশি e-commerce: যেসব product-এ অন্তত একটি review আছে

-- EXISTS: outer table ছোট হলে efficient (early termination)
SELECT p.id, p.name FROM products p
WHERE EXISTS (
    SELECT 1 FROM reviews r WHERE r.product_id = p.id AND r.rating >= 4
);

-- IN: subquery result set ছোট হলে efficient
SELECT p.id, p.name FROM products p
WHERE p.id IN (
    SELECT DISTINCT product_id FROM reviews WHERE rating >= 4
);

-- ✅ সেরা: JOIN
SELECT DISTINCT p.id, p.name
FROM products p
JOIN reviews r ON p.id = r.product_id
WHERE r.rating >= 4;
```

#### Derived Tables (Subquery in FROM)

```sql
-- প্রতিটি category-র top 3 selling product
SELECT ranked.category_name, ranked.product_name, ranked.total_sold
FROM (
    SELECT c.name as category_name, p.name as product_name,
        SUM(oi.quantity) as total_sold,
        ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY SUM(oi.quantity) DESC) as rn
    FROM categories c
    JOIN products p ON c.id = p.category_id
    JOIN order_items oi ON p.id = oi.product_id
    GROUP BY c.id, c.name, p.id, p.name
) ranked
WHERE ranked.rn <= 3;
```

---

### ৬. Aggregation Optimization

#### GROUP BY with Indexes

```sql
-- ✅ Index column দিয়ে GROUP BY — index scan ব্যবহার হবে
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

SELECT status, DATE(created_at) as order_date, COUNT(*) as order_count
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY status, DATE(created_at);

-- ⚠️ GROUP BY column order index order-এর সাথে match করা উচিত
```

#### HAVING vs WHERE

```sql
-- ❌ HAVING দিয়ে filter — GROUP BY-র পরে filter হয় (বেশি row process)
SELECT category_id, COUNT(*) as product_count
FROM products
GROUP BY category_id
HAVING category_id IN (1, 2, 3);

-- ✅ WHERE দিয়ে আগেই filter — কম row process
SELECT category_id, COUNT(*) as product_count
FROM products
WHERE category_id IN (1, 2, 3)
GROUP BY category_id;

-- HAVING শুধু aggregate condition-এ ব্যবহার করুন
SELECT category_id, COUNT(*) as product_count
FROM products
WHERE is_active = 1
GROUP BY category_id
HAVING COUNT(*) > 10; -- ✅ aggregate condition — HAVING সঠিক
```

#### COUNT Variations

```sql
-- COUNT(*): সব row গণনা করে (NULL সহ) — সবচেয়ে দ্রুত
-- COUNT(column): non-NULL value গণনা করে
-- COUNT(DISTINCT column): unique non-NULL value গণনা করে

SELECT
    COUNT(*) as total_orders,                    -- সব order
    COUNT(discount_code) as discounted_orders,   -- discount code আছে এমন
    COUNT(DISTINCT user_id) as unique_customers  -- unique customer সংখ্যা
FROM orders
WHERE created_at >= '2024-01-01';
```

#### Window Functions

```sql
-- Window functions GROUP BY ছাড়া aggregate calculation করে — row হারায় না

-- প্রতিটি user-এর order rank (সবচেয়ে বেশি খরচ করা user আগে)
SELECT
    u.name,
    o.total,
    o.created_at,
    ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.created_at DESC) as order_seq,
    RANK() OVER (ORDER BY o.total DESC) as spending_rank,
    SUM(o.total) OVER (PARTITION BY u.id) as user_total,
    LAG(o.total) OVER (PARTITION BY u.id ORDER BY o.created_at) as prev_order_total,
    LEAD(o.total) OVER (PARTITION BY u.id ORDER BY o.created_at) as next_order_total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed';

-- Running total (চলমান যোগফল)
SELECT
    DATE(created_at) as order_date,
    SUM(total) as daily_total,
    SUM(SUM(total)) OVER (ORDER BY DATE(created_at)) as running_total
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY DATE(created_at);
```

#### Summary Tables / Pre-aggregation

```sql
-- বারবার aggregate query চালানোর বদলে summary table ব্যবহার করুন

CREATE TABLE daily_sales_summary (
    date DATE NOT NULL,
    category_id INT NOT NULL,
    total_orders INT DEFAULT 0,
    total_revenue DECIMAL(12,2) DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    unique_customers INT DEFAULT 0,
    PRIMARY KEY (date, category_id),
    INDEX idx_date (date),
    INDEX idx_category (category_id)
);

-- Cron job বা event দিয়ে populate করুন (প্রতিদিন রাতে)
INSERT INTO daily_sales_summary (date, category_id, total_orders, total_revenue, avg_order_value, unique_customers)
SELECT
    DATE(o.created_at),
    p.category_id,
    COUNT(DISTINCT o.id),
    SUM(oi.quantity * oi.price),
    AVG(o.total),
    COUNT(DISTINCT o.user_id)
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE DATE(o.created_at) = CURDATE() - INTERVAL 1 DAY
GROUP BY DATE(o.created_at), p.category_id
ON DUPLICATE KEY UPDATE
    total_orders = VALUES(total_orders),
    total_revenue = VALUES(total_revenue),
    avg_order_value = VALUES(avg_order_value),
    unique_customers = VALUES(unique_customers);
```

---

### ৭. Pagination Optimization

#### OFFSET/LIMIT সমস্যা

```sql
-- ❌ বড় OFFSET-এ performance ভয়াবহ হয় — O(n) complexity
-- Page 1000 পেতে হলে প্রথম 999,980 টি row skip করতে হবে!
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 999980;
-- MySQL আসলে 1,000,000 row scan করে 999,980 টি ফেলে দেয়!
```

#### Cursor-Based / Keyset Pagination

```sql
-- ✅ Cursor-based: সবসময় O(1) — last seen id ব্যবহার করে
-- প্রথম page
SELECT id, name, price FROM products
WHERE is_active = 1
ORDER BY id
LIMIT 20;

-- পরবর্তী page (last_id = 20 ছিল)
SELECT id, name, price FROM products
WHERE is_active = 1 AND id > 20
ORDER BY id
LIMIT 20;

-- Multiple column sort-এ cursor pagination
-- (created_at, id) দিয়ে sort করলে:
SELECT id, name, created_at FROM products
WHERE is_active = 1
  AND (created_at, id) > ('2024-06-15 10:30:00', 5432)
ORDER BY created_at, id
LIMIT 20;
```

#### Deferred JOIN Technique

```sql
-- ❌ বড় offset-এ সব column পড়ে
SELECT * FROM products ORDER BY created_at DESC LIMIT 20 OFFSET 100000;

-- ✅ Deferred JOIN: আগে শুধু id নিন, তারপর join করে data আনুন
SELECT p.*
FROM products p
INNER JOIN (
    SELECT id FROM products
    ORDER BY created_at DESC
    LIMIT 20 OFFSET 100000
) AS tmp ON p.id = tmp.id
ORDER BY p.created_at DESC;
-- Inner query covering index ব্যবহার করে (শুধু id ও created_at) — অনেক দ্রুত
```

**Laravel:**

```php
// ❌ OFFSET-based (বড় page-এ ধীর)
$products = Product::paginate(20); // page=5000 হলে সমস্যা

// ✅ Cursor pagination
$products = Product::orderBy('id')->cursorPaginate(20);
// URL: ?cursor=eyJpZCI6MjAsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0

// ✅ cursor() — memory-efficient iteration (বড় dataset export-এ)
Product::where('is_active', true)
    ->orderBy('id')
    ->cursor()
    ->each(function ($product) {
        // একটি একটি করে process করে, পুরো collection memory-তে রাখে না
        processProduct($product);
    });

// ✅ chunk() — batch processing
Product::where('is_active', true)
    ->orderBy('id')
    ->chunk(1000, function ($products) {
        foreach ($products as $product) {
            $product->updateSearchIndex();
        }
    });
```

**Sequelize:**

```javascript
// ✅ Cursor-based pagination (Knex example)
const knex = require('knex')(config);

async function cursorPaginate(lastId = 0, limit = 20) {
  const products = await knex('products')
    .where('id', '>', lastId)
    .where('is_active', true)
    .orderBy('id')
    .limit(limit);

  const nextCursor = products.length > 0
    ? products[products.length - 1].id
    : null;

  return { data: products, nextCursor };
}

// Sequelize-তে cursor pagination
async function cursorPaginate(cursor, limit = 20) {
  const where = { isActive: true };
  if (cursor) {
    where.id = { [Op.gt]: cursor };
  }

  const products = await Product.findAll({
    where,
    order: [['id', 'ASC']],
    limit: limit + 1 // extra 1 দিয়ে hasMore check
  });

  const hasMore = products.length > limit;
  if (hasMore) products.pop();

  return {
    data: products,
    nextCursor: products.length > 0 ? products[products.length - 1].id : null,
    hasMore
  };
}
```

---

### ৮. INSERT/UPDATE/BULK Optimization

#### Batch INSERT

```sql
-- ❌ একটি একটি INSERT (প্রতিটিতে network round-trip + transaction)
INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (1, 10, 2, 500);
INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (1, 20, 1, 300);
INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (1, 30, 3, 150);

-- ✅ Batch INSERT (একটি round-trip, একটি transaction)
INSERT INTO order_items (order_id, product_id, quantity, price) VALUES
    (1, 10, 2, 500),
    (1, 20, 1, 300),
    (1, 30, 3, 150);
-- 10x-100x দ্রুত (বিশেষত network latency বেশি হলে)
```

#### UPSERT

```sql
-- MySQL: INSERT ON DUPLICATE KEY UPDATE
INSERT INTO product_inventory (product_id, warehouse_id, quantity)
VALUES (101, 1, 50), (102, 1, 30), (103, 1, 20)
ON DUPLICATE KEY UPDATE
    quantity = quantity + VALUES(quantity),
    updated_at = NOW();

-- PostgreSQL: INSERT ON CONFLICT
INSERT INTO product_inventory (product_id, warehouse_id, quantity)
VALUES (101, 1, 50), (102, 1, 30), (103, 1, 20)
ON CONFLICT (product_id, warehouse_id) DO UPDATE SET
    quantity = product_inventory.quantity + EXCLUDED.quantity,
    updated_at = NOW();
```

#### Bulk UPDATE with CASE

```sql
-- ❌ একটি একটি UPDATE
UPDATE products SET price = 500 WHERE id = 1;
UPDATE products SET price = 300 WHERE id = 2;
UPDATE products SET price = 150 WHERE id = 3;

-- ✅ CASE দিয়ে একটি query-তে সব update
UPDATE products SET
    price = CASE id
        WHEN 1 THEN 500
        WHEN 2 THEN 300
        WHEN 3 THEN 150
    END,
    updated_at = NOW()
WHERE id IN (1, 2, 3);
```

**Laravel:**

```php
// ✅ Batch insert
Order::insert([
    ['user_id' => 1, 'total' => 500, 'status' => 'pending', 'created_at' => now()],
    ['user_id' => 2, 'total' => 300, 'status' => 'pending', 'created_at' => now()],
    ['user_id' => 3, 'total' => 150, 'status' => 'pending', 'created_at' => now()],
]);

// ✅ Upsert (Laravel 8+)
Product::upsert(
    [
        ['sku' => 'SKU-001', 'name' => 'Product A', 'price' => 500],
        ['sku' => 'SKU-002', 'name' => 'Product B', 'price' => 300],
    ],
    ['sku'],           // unique key column(s)
    ['name', 'price']  // update করতে হবে যে column-গুলো
);

// ✅ chunk দিয়ে বড় dataset insert
collect($largeDataset)->chunk(1000)->each(function ($chunk) {
    DB::table('products')->insert($chunk->toArray());
});
```

**Sequelize:**

```javascript
// ✅ bulkCreate
await OrderItem.bulkCreate([
  { orderId: 1, productId: 10, quantity: 2, price: 500 },
  { orderId: 1, productId: 20, quantity: 1, price: 300 },
  { orderId: 1, productId: 30, quantity: 3, price: 150 }
], { validate: true });

// ✅ Upsert (bulkCreate with updateOnDuplicate)
await Product.bulkCreate(
  [
    { sku: 'SKU-001', name: 'Product A', price: 500 },
    { sku: 'SKU-002', name: 'Product B', price: 300 }
  ],
  {
    updateOnDuplicate: ['name', 'price', 'updatedAt'],
    conflictAttributes: ['sku']
  }
);

// Knex batch insert
await knex.batchInsert('order_items', items, 1000); // 1000 per batch
```

---

### ৯. Locking & Concurrency

#### Optimistic vs Pessimistic Locking

```sql
-- Pessimistic Locking: Row-কে lock করে অন্যদের আটকায়
-- ব্যবহার: Bank transaction, inventory management
BEGIN;
SELECT stock FROM products WHERE id = 101 FOR UPDATE;
-- অন্য transaction এই row access করতে পারবে না যতক্ষণ না COMMIT হয়
UPDATE products SET stock = stock - 1 WHERE id = 101;
COMMIT;

-- Optimistic Locking: Version/timestamp check করে conflict detect করে
-- ব্যবহার: CMS, user profile update
UPDATE products
SET price = 600, version = version + 1
WHERE id = 101 AND version = 5;
-- affected rows = 0 হলে অন্য কেউ আগেই update করেছে — retry করুন
```

#### SKIP LOCKED (Queue Pattern)

```sql
-- Worker queue-র জন্য — প্রতিটি worker আলাদা job নেবে
BEGIN;
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- SKIP LOCKED: locked row skip করে পরেরটি নেয় (queue-র জন্য আদর্শ)

UPDATE jobs SET status = 'processing', worker_id = 'worker-1' WHERE id = ?;
COMMIT;
```

**Laravel:**

```php
// Pessimistic Locking
DB::transaction(function () {
    $product = Product::where('id', 101)
        ->lockForUpdate()  // SELECT ... FOR UPDATE
        ->first();

    if ($product->stock > 0) {
        $product->decrement('stock');
        Order::create([...]);
    }
});

// Shared Lock (read lock — অন্যরা পড়তে পারবে কিন্তু write করতে পারবে না)
$product = Product::where('id', 101)
    ->sharedLock()  // SELECT ... LOCK IN SHARE MODE
    ->first();

// Optimistic Locking (manual implementation)
$product = Product::find(101);
$updated = Product::where('id', 101)
    ->where('version', $product->version)
    ->update([
        'price' => 600,
        'version' => $product->version + 1
    ]);

if (!$updated) {
    throw new ConflictException('Product was modified by another user');
}
```

#### Deadlock Prevention

```sql
-- Deadlock হয় যখন দুটি transaction একে অপরের lock-এর জন্য অপেক্ষা করে

-- ✅ সবসময় একই order-এ table/row access করুন
-- ✅ Transaction ছোট রাখুন
-- ✅ Retry mechanism রাখুন

-- MySQL-এ deadlock monitoring:
SHOW ENGINE INNODB STATUS; -- LATEST DETECTED DEADLOCK section দেখুন
```

```php
// Laravel-এ deadlock retry
use Illuminate\Database\DeadlockException;

$maxRetries = 3;
for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
    try {
        DB::transaction(function () {
            // ... transaction logic
        });
        break;
    } catch (DeadlockException $e) {
        if ($attempt === $maxRetries) throw $e;
        usleep(100000 * $attempt); // progressive delay
    }
}
```

---

## 🔥 Advanced Scenarios

### ১. Query Caching

#### MySQL Query Cache (8.0-তে deprecated)

MySQL 8.0 এর আগে built-in query cache ছিল কিন্তু table-level invalidation-এর কারণে write-heavy workload-এ সমস্যা হতো। এখন application-level caching ব্যবহার করা উচিত।

#### Application-Level Caching (Redis)

**Laravel:**

```php
// Cache::remember pattern — সবচেয়ে সাধারণ এবং কার্যকর
$topProducts = Cache::remember('top_products:dhaka', 3600, function () {
    return Product::select(['id', 'name', 'price', 'image_url'])
        ->withCount('orders')
        ->where('city', 'Dhaka')
        ->orderByDesc('orders_count')
        ->limit(20)
        ->get();
});

// Tag-based cache (Redis/Memcached only)
$categoryProducts = Cache::tags(['products', "category:{$categoryId}"])
    ->remember("category_products:{$categoryId}:page:{$page}", 1800, function () use ($categoryId, $page) {
        return Product::where('category_id', $categoryId)
            ->paginate(20, ['*'], 'page', $page);
    });

// Cache invalidation: category-র সব product cache clear
Cache::tags(["category:{$categoryId}"])->flush();

// Model Observer-এ auto invalidation
class ProductObserver {
    public function saved(Product $product) {
        Cache::tags(["category:{$product->category_id}"])->flush();
        Cache::forget("product:{$product->id}");
    }
}
```

**Node.js (Redis + Sequelize):**

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getTopProducts(city) {
  const cacheKey = `top_products:${city}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const products = await Product.findAll({
    attributes: ['id', 'name', 'price', 'imageUrl'],
    where: { city },
    order: [['salesCount', 'DESC']],
    limit: 20
  });

  await redis.setex(cacheKey, 3600, JSON.stringify(products));
  return products;
}

// Cache invalidation
async function invalidateProductCache(product) {
  const keys = await redis.keys(`*category:${product.categoryId}*`);
  if (keys.length) await redis.del(...keys);
  await redis.del(`product:${product.id}`);
}
```

#### Cache Invalidation Strategies

| Strategy | বর্ণনা | ব্যবহার |
|---|---|---|
| TTL-based | নির্দিষ্ট সময় পর expire | Product listing, analytics |
| Event-based | Data পরিবর্তন হলে invalidate | Shopping cart, inventory |
| Tag-based | Related cache group clear | Category-র সব products |
| Write-through | Write-এর সময়ই cache update | Critical data (price, stock) |

---

### ২. Partitioning

বড় table-কে ছোট ছোট ভাগে ভাগ করা হলো partitioning। এতে query শুধু relevant partition scan করে।

```sql
-- Range Partitioning (সবচেয়ে সাধারণ — date-based)
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    user_id INT NOT NULL,
    total DECIMAL(10,2),
    status ENUM('pending','completed','cancelled'),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at),
    INDEX idx_user (user_id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Query করলে partition pruning হয় — শুধু relevant partition scan হয়
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-07-01';
-- শুধু p2024 partition scan হবে, বাকি partition touch হবে না ✅

-- Hash Partitioning (equal distribution-এর জন্য)
CREATE TABLE sessions (
    id BIGINT AUTO_INCREMENT,
    user_id INT NOT NULL,
    data JSON,
    created_at DATETIME,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH(user_id) PARTITIONS 8;

-- List Partitioning
CREATE TABLE orders_by_region (
    id BIGINT AUTO_INCREMENT,
    region VARCHAR(20),
    total DECIMAL(10,2),
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS(region) (
    PARTITION p_dhaka VALUES IN ('Dhaka', 'Gazipur', 'Narayanganj'),
    PARTITION p_chittagong VALUES IN ('Chittagong', 'Comilla', 'Noakhali'),
    PARTITION p_rajshahi VALUES IN ('Rajshahi', 'Rangpur', 'Bogra'),
    PARTITION p_others VALUES IN ('Sylhet', 'Khulna', 'Barisal', 'Mymensingh')
);
```

**কখন Partitioning করবেন:**
- Table ১০ million+ row হলে
- Query সবসময় একটি নির্দিষ্ট range-এ হলে (date, region)
- পুরানো data archive/delete করতে হলে (partition drop অনেক দ্রুত)
- সব query-তে partition key ব্যবহার হলে

---

### ৩. Slow Query Analysis

#### MySQL Slow Query Log

```sql
-- Slow query log enable করুন
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 1 সেকেন্ডের বেশি সময় নেওয়া query log হবে
SET GLOBAL log_queries_not_using_indexes = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

#### PostgreSQL pg_stat_statements

```sql
-- pg_stat_statements extension enable করুন
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- সবচেয়ে সময়সাপেক্ষ query খুঁজুন
SELECT
    query,
    calls,
    ROUND(total_exec_time::numeric, 2) as total_ms,
    ROUND(mean_exec_time::numeric, 2) as avg_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

#### Percona pt-query-digest

```bash
# Slow query log analyze করুন
pt-query-digest /var/log/mysql/slow.log

# Output-এ দেখাবে:
# - সবচেয়ে ধীর query গুলো ranked
# - প্রতিটি query-র execution count, avg time, max time
# - Query fingerprint (similar query grouped)
```

#### Laravel Telescope

```php
// config/telescope.php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'slow' => 100,  // 100ms-এর বেশি সময় নেওয়া query flag করবে
    ],
],

// Custom slow query alert (production-এ)
DB::listen(function ($query) {
    if ($query->time > 1000) { // 1 সেকেন্ডের বেশি
        Log::warning('Slow query detected', [
            'sql' => $query->sql,
            'bindings' => $query->bindings,
            'time_ms' => $query->time,
        ]);
        // Slack notification পাঠান
        SlowQueryAlert::dispatch($query);
    }
});
```

---

### ৪. Database Connection Pooling

Connection তৈরি করা expensive — TCP handshake, authentication, session setup সবকিছু সময় নেয়। Connection pooling আগে থেকে connection তৈরি করে রাখে এবং reuse করে।

**PHP / Laravel:**

```php
// config/database.php — Laravel pool configuration
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'database' => env('DB_DATABASE', 'ecommerce'),
    // Persistent connections — PHP-FPM-এ connection reuse
    'options' => [
        PDO::ATTR_PERSISTENT => true,
    ],
    // Swoole/Octane ব্যবহারে:
    'pool' => [
        'min_connections' => 5,
        'max_connections' => 50,
        'wait_timeout' => 3.0,
        'max_idle_time' => 60,
    ],
],
```

**Node.js (Sequelize & Knex):**

```javascript
// Sequelize pool configuration
const sequelize = new Sequelize('ecommerce', 'user', 'pass', {
  host: 'localhost',
  dialect: 'mysql',
  pool: {
    max: 20,        // সর্বোচ্চ connection
    min: 5,         // সর্বনিম্ন connection (idle-তেও রাখবে)
    acquire: 30000, // connection পেতে max wait (ms)
    idle: 10000,    // idle connection কখন ছাড়বে (ms)
    evict: 1000     // কত ঘন ঘন idle connection check করবে
  },
  retry: { max: 3 } // connection failure-তে retry
});

// Knex pool configuration
const knex = require('knex')({
  client: 'pg',
  connection: {
    host: 'localhost',
    database: 'ecommerce'
  },
  pool: {
    min: 5,
    max: 30,
    acquireTimeoutMillis: 30000,
    createTimeoutMillis: 30000,
    idleTimeoutMillis: 30000,
    reapIntervalMillis: 1000,
    propagateCreateError: false
  }
});
```

**PgBouncer (PostgreSQL):**

```ini
; pgbouncer.ini
[databases]
ecommerce = host=127.0.0.1 port=5432 dbname=ecommerce

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
pool_mode = transaction    ; transaction-level pooling (সবচেয়ে efficient)
max_client_conn = 1000     ; client থেকে max connection
default_pool_size = 25     ; database-এ max connection per user/db pair
min_pool_size = 5
reserve_pool_size = 5
```

---

### ৫. Read Replica Optimization

#### Read/Write Splitting

Production system-এ write একটি primary server-এ যাবে এবং read গুলো replica-তে distribute হবে।

**Laravel:**

```php
// config/database.php
'mysql' => [
    'read' => [
        'host' => [
            env('DB_READ_HOST_1', '10.0.1.10'),
            env('DB_READ_HOST_2', '10.0.1.11'),
        ],
    ],
    'write' => [
        'host' => env('DB_WRITE_HOST', '10.0.1.1'),
    ],
    'sticky' => true, // write-এর পর same request-এ read-ও write server থেকে হবে
    'driver' => 'mysql',
    'database' => env('DB_DATABASE', 'ecommerce'),
],

// নির্দিষ্ট connection ব্যবহার
$analytics = DB::connection('read')->table('orders')
    ->where('created_at', '>=', '2024-01-01')
    ->sum('total');

// Write সবসময় primary-তে যাবে
$order = new Order([...]);
$order->save(); // automatically write connection ব্যবহার করবে
```

**Sequelize (Read Replication):**

```javascript
const sequelize = new Sequelize('ecommerce', null, null, {
  dialect: 'mysql',
  replication: {
    read: [
      { host: '10.0.1.10', username: 'read_user', password: 'pass' },
      { host: '10.0.1.11', username: 'read_user', password: 'pass' }
    ],
    write: {
      host: '10.0.1.1', username: 'write_user', password: 'pass'
    }
  },
  pool: { max: 20, idle: 30000 }
});

// SELECT automatically read replica-তে যাবে
const products = await Product.findAll({ where: { isActive: true } });

// INSERT/UPDATE automatically write server-এ যাবে
await Product.update({ price: 600 }, { where: { id: 101 } });
```

#### Replication Lag Handling

```php
// সমস্যা: write-এর পরই read করলে replica-তে data নাও থাকতে পারে
$order = Order::create([...]);

// ❌ Replica-তে এখনও আসেনি হয়তো
$freshOrder = Order::find($order->id);

// ✅ সমাধান ১: sticky connection (config-এ 'sticky' => true)
// একই request-এ write-এর পর read-ও primary থেকে হবে

// ✅ সমাধান ২: Force primary connection
$freshOrder = Order::on('write')->find($order->id);

// ✅ সমাধান ৩: Critical read-এ primary ব্যবহার
DB::connection('write')->table('orders')->where('id', $order->id)->first();
```

---

## ✅ Optimization Checklist

```
□ SELECT * এড়িয়ে প্রয়োজনীয় column select করুন
□ EXPLAIN চালিয়ে query plan verify করুন
□ WHERE clause-এ column-এ function apply করবেন না (sargable রাখুন)
□ সঠিক index তৈরি করুন (composite index-এ column order গুরুত্বপূর্ণ)
□ N+1 সমস্যা check করুন — Eager Loading ব্যবহার করুন
□ বড় OFFSET এড়িয়ে cursor pagination ব্যবহার করুন
□ Batch INSERT/UPDATE ব্যবহার করুন
□ Slow query log enable ও monitor করুন
□ Connection pooling configure করুন
□ Application-level caching (Redis) implement করুন
□ Read replica setup করুন (read-heavy system-এ)
□ HAVING-এর বদলে WHERE ব্যবহার করুন (যেখানে সম্ভব)
□ Transaction ছোট রাখুন — deadlock এড়ান
□ ANALYZE TABLE নিয়মিত চালান (statistics update)
□ বড় table-এ partitioning consider করুন
```

---

## ⚠️ সাধারণ ভুল ও সমাধান

### ১. SELECT * সর্বত্র ব্যবহার
```sql
-- ❌ ভুল
SELECT * FROM products JOIN categories ON ...;
-- ✅ সমাধান: নির্দিষ্ট column select করুন
SELECT p.id, p.name, p.price, c.name FROM products p JOIN categories c ON ...;
```

### ২. Index ছাড়া WHERE/JOIN
```sql
-- ❌ ভুল: foreign key-তে index নেই
SELECT * FROM orders WHERE user_id = 100; -- full table scan!
-- ✅ সমাধান
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### ৩. WHERE-এ Column-এ Function
```sql
-- ❌ ভুল
WHERE YEAR(created_at) = 2024
-- ✅ সমাধান
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```

### ৪. N+1 Query সমস্যা Ignore করা
```php
// ❌ ভুল: Loop-এ query
foreach (Order::all() as $order) { echo $order->user->name; }
// ✅ সমাধান: Eager loading
foreach (Order::with('user')->get() as $order) { echo $order->user->name; }
```

### ৫. OFFSET দিয়ে Deep Pagination
```sql
-- ❌ ভুল
SELECT * FROM products LIMIT 20 OFFSET 1000000;
-- ✅ সমাধান: Cursor pagination
SELECT * FROM products WHERE id > 1000000 ORDER BY id LIMIT 20;
```

### ৬. Unnecessary DISTINCT/ORDER BY
```sql
-- ❌ ভুল: DISTINCT প্রায়ই JOIN ভুলের লক্ষণ
SELECT DISTINCT u.* FROM users u JOIN orders o ON ...;
-- ✅ সমাধান: JOIN logic ঠিক করুন বা EXISTS ব্যবহার করুন
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### ৭. Implicit Type Conversion
```sql
-- ❌ ভুল: phone VARCHAR column-এ numeric comparison — index ব্যবহার হবে না
WHERE phone = 01712345678
-- ✅ সমাধান: সঠিক type ব্যবহার করুন
WHERE phone = '01712345678'
```

### ৮. একটি একটি INSERT/UPDATE
```php
// ❌ ভুল: Loop-এ insert
foreach ($items as $item) { OrderItem::create($item); }
// ✅ সমাধান: Batch insert
OrderItem::insert($items);
```

### ৯. Transaction-এ External API Call
```php
// ❌ ভুল: Transaction দীর্ঘ হয়, lock ধরে রাখে
DB::transaction(function () {
    $order = Order::create([...]);
    Http::post('payment-gateway/charge', [...]); // ধীর API call!
    $order->update(['status' => 'paid']);
});
// ✅ সমাধান: API call transaction-এর বাইরে
$order = DB::transaction(fn() => Order::create([...]));
$paymentResult = Http::post('payment-gateway/charge', [...]);
DB::transaction(fn() => $order->update(['status' => 'paid']));
```

### ১০. Statistics Update না করা
```sql
-- ❌ ভুল: বড় data migration-এর পর statistics পুরানো থাকে
-- Query optimizer ভুল plan বেছে নেয়
-- ✅ সমাধান: Migration-এর পর
ANALYZE TABLE orders;
ANALYZE TABLE products;
-- PostgreSQL:
ANALYZE orders;
VACUUM ANALYZE products;
```

---

## 🧪 Performance Testing

#### Query Benchmarking

```sql
-- MySQL-এ query profiling
SET profiling = 1;
SELECT ... ; -- আপনার query
SHOW PROFILE FOR QUERY 1;
-- CPU, Block I/O, Context Switches সব দেখাবে

-- Multiple run-এ average নিন (cache effect এড়াতে)
-- প্রথম run: cold cache
-- দ্বিতীয়+ run: warm cache
```

**PHP Benchmarking:**

```php
// Simple benchmark
$start = microtime(true);

$results = DB::table('orders')
    ->join('users', 'orders.user_id', '=', 'users.id')
    ->where('orders.created_at', '>=', '2024-01-01')
    ->select('users.name', DB::raw('SUM(orders.total) as total'))
    ->groupBy('users.id', 'users.name')
    ->get();

$elapsed = microtime(true) - $start;
Log::info("Query time: {$elapsed}s, Results: {$results->count()}");

// Laravel-এ Benchmark utility (9.x+)
use Illuminate\Support\Benchmark;

$time = Benchmark::measure(fn () => Order::with('user')->where('status', 'pending')->get());
// Returns execution time in milliseconds
```

**JavaScript Benchmarking:**

```javascript
// Sequelize query benchmarking
const { performance } = require('perf_hooks');

async function benchmarkQuery(label, queryFn, iterations = 10) {
  const times = [];

  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    await queryFn();
    times.push(performance.now() - start);
  }

  const avg = times.reduce((a, b) => a + b, 0) / times.length;
  const min = Math.min(...times);
  const max = Math.max(...times);
  const p95 = times.sort((a, b) => a - b)[Math.floor(times.length * 0.95)];

  console.log(`[${label}] avg: ${avg.toFixed(2)}ms, min: ${min.toFixed(2)}ms, max: ${max.toFixed(2)}ms, p95: ${p95.toFixed(2)}ms`);
}

await benchmarkQuery('Top products', async () => {
  await Product.findAll({
    attributes: ['id', 'name', 'price'],
    where: { isActive: true },
    order: [['salesCount', 'DESC']],
    limit: 20
  });
});
```

---

## 📋 সারসংক্ষেপ (Quick Reference)

### Optimization Decision Matrix

| সমস্যা | চিহ্নিতকরণ | সমাধান | Impact |
|---|---|---|---|
| Full table scan | EXPLAIN type: ALL | Index তৈরি করুন | 🔴 High |
| N+1 queries | Query log-এ অনেক similar query | Eager loading | 🔴 High |
| Slow pagination | বড় OFFSET-এ response time বাড়ে | Cursor pagination | 🟡 Medium |
| Non-sargable WHERE | Function on indexed column | Query rewrite | 🔴 High |
| Filesort | Extra: Using filesort | Composite index | 🟡 Medium |
| Temporary table | Extra: Using temporary | Query restructure | 🟡 Medium |
| Missing covering index | type: ref but not Using index | Covering index | 🟢 Low-Med |
| Lock contention | Slow writes, deadlocks | Optimistic locking, smaller tx | 🔴 High |
| Connection exhaustion | "Too many connections" error | Connection pooling | 🔴 High |
| Repetitive queries | Same query বারবার চলছে | Redis caching | 🟡 Medium |

### Index Strategy Cheat Sheet

```sql
-- WHERE শুধু একটি column: Single index
CREATE INDEX idx_status ON orders(status);

-- WHERE multiple columns: Composite index (selective column আগে)
CREATE INDEX idx_user_status ON orders(user_id, status);

-- WHERE + ORDER BY: Composite index (WHERE columns আগে, then ORDER BY)
CREATE INDEX idx_status_created ON orders(status, created_at);

-- Covering index: সব selected column include করুন
CREATE INDEX idx_covering ON orders(status, created_at, id, total);

-- Prefix index (দীর্ঘ string column-এ)
CREATE INDEX idx_email_prefix ON users(email(20));
```

### বাংলাদেশি E-Commerce Context-এ প্রয়োগ

```sql
-- ঢাকায় সর্বাধিক বিক্রি হওয়া product (optimized)
SELECT p.id, p.name, p.price, SUM(oi.quantity) as total_sold
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
WHERE o.shipping_city = 'Dhaka'
  AND o.created_at >= '2024-01-01'
  AND o.status = 'delivered'
GROUP BY p.id, p.name, p.price
ORDER BY total_sold DESC
LIMIT 20;

-- প্রয়োজনীয় indexes:
CREATE INDEX idx_orders_city_status_date ON orders(shipping_city, status, created_at);
CREATE INDEX idx_order_items_order_product ON order_items(order_id, product_id, quantity);

-- User search (phone number / name)
-- বাংলাদেশে phone number দিয়ে search সবচেয়ে সাধারণ
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_name ON users(name(20));

SELECT id, name, phone, email
FROM users
WHERE phone = '01712345678';

-- বড় transaction table-এ monthly report
-- Partition ব্যবহার করুন + summary table
SELECT
    ds.date,
    SUM(ds.total_orders) as orders,
    SUM(ds.total_revenue) as revenue
FROM daily_sales_summary ds
WHERE ds.date BETWEEN '2024-01-01' AND '2024-06-30'
GROUP BY ds.date
ORDER BY ds.date;
-- Summary table থেকে পড়া হচ্ছে — milliseconds-এ result আসবে!
```

---

> **মূল কথা:** Query optimization কোনো one-time কাজ নয় — এটি একটি continuous process। Application বড় হওয়ার সাথে সাথে data বাড়ে, query pattern পরিবর্তন হয়, নতুন bottleneck তৈরি হয়। নিয়মিত slow query monitor করুন, EXPLAIN ব্যবহার করুন, এবং data-driven সিদ্ধান্ত নিন। Premature optimization না করে, আগে measure করুন তারপর optimize করুন।
