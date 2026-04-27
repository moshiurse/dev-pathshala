# 📇 ইনডেক্সিং (Indexing)

## 📌 সংজ্ঞা ও মূল ধারণা

### Index কী?

Database index হলো একটি আলাদা data structure যা table-এর এক বা একাধিক column-এর উপর তৈরি করা হয়, যাতে query execution দ্রুততর হয়। বইয়ের সূচিপত্রের মতো — আপনি পুরো বই না পড়েই নির্দিষ্ট বিষয় খুঁজে পান।

Index ছাড়া database engine-কে **Full Table Scan** করতে হয় — অর্থাৎ প্রতিটি row একে একে পরীক্ষা করতে হয়। ১ কোটি row-এর table-এ এটা অত্যন্ত ধীর এবং resource-intensive।

### কেন Index প্রয়োজন?

- **SELECT query acceleration**: WHERE, JOIN, ORDER BY, GROUP BY clause দ্রুত execute হয়
- **Unique constraint enforcement**: UNIQUE index duplicate data প্রতিরোধ করে
- **Sort optimization**: ORDER BY clause-এ filesort এড়ানো যায়
- **Join performance**: Foreign key column-এ index থাকলে JOIN অনেক দ্রুত হয়

### Index অভ্যন্তরীণভাবে কীভাবে কাজ করে (Page/Block Level)

Database data **page** বা **block** আকারে disk-এ সংরক্ষিত থাকে (সাধারণত 8KB PostgreSQL-এ, 16KB InnoDB-তে)। প্রতিটি page-এ কিছু row থাকে।

```
┌─────────────────────────────────────────────────┐
│              Without Index (Full Table Scan)      │
│                                                   │
│  Page 1 → Page 2 → Page 3 → ... → Page N        │
│  [scan]   [scan]   [scan]         [scan]         │
│                                                   │
│  সব page পড়তে হবে = O(N) disk I/O               │
├─────────────────────────────────────────────────┤
│              With Index (B-Tree Lookup)           │
│                                                   │
│  Root → Internal Node → Leaf Node → Data Page    │
│  [1 I/O]  [1 I/O]      [1 I/O]    [1 I/O]      │
│                                                   │
│  মাত্র 3-4 page পড়তে হবে = O(log N) disk I/O    │
└─────────────────────────────────────────────────┘
```

Index আলাদা data structure হিসেবে disk-এ থাকে। এতে **indexed column value** এবং সেই row-এর **pointer** (row ID / primary key) সংরক্ষিত থাকে।

### Index Overhead (Write Penalty ও Storage Cost)

Index বিনামূল্যে আসে না। প্রতিটি index-এর একটি মূল্য আছে:

| Overhead Type | ব্যাখ্যা |
|---|---|
| **Write Penalty** | INSERT/UPDATE/DELETE-এ index-ও update হয়। ৫টি index থাকলে ১টি INSERT-এ ৫টি index update হবে |
| **Storage Cost** | প্রতিটি index আলাদা disk space নেয়। বড় table-এ এটি GB পরিমাণ হতে পারে |
| **Memory Pressure** | Index buffer pool-এ load হয়। বেশি index = কম data page cache-এ থাকে |
| **Maintenance** | Index fragmentation হয়, পর্যায়ক্রমে REINDEX/OPTIMIZE দরকার |

```sql
-- Index-এর storage size দেখুন (MySQL)
SELECT 
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE table_name = 'orders'
AND stat_name = 'size';

-- PostgreSQL-এ index size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'orders';
```

---

## 🏠 বাস্তব উদাহরণ

কল্পনা করুন আপনার কাছে একটি ১০০০ পৃষ্ঠার বই আছে এবং আপনি "B-Tree" বিষয়টি খুঁজছেন:

**সূচিপত্র ছাড়া (Without Index):**
পৃষ্ঠা ১ থেকে শুরু করে প্রতিটি পৃষ্ঠা পড়তে হবে। গড়ে ৫০০ পৃষ্ঠা পড়তে হবে (O(N))।

**সূচিপত্র সহ (With Index):**
সূচিপত্রে "B-Tree → পৃষ্ঠা ৩৪২" লেখা আছে। সরাসরি পৃষ্ঠা ৩৪২-তে যান (O(log N))।

**বাংলাদেশের প্রেক্ষাপটে:** ধরুন Daraz-এর মতো একটি e-commerce platform-এ ৫০ লক্ষ product আছে। একজন customer "Samsung Galaxy S24" search করলে:

- **Index ছাড়া**: ৫০ লক্ষ product-এর নাম একে একে match করবে → কয়েক সেকেন্ড লাগবে
- **Index সহ**: Product name-এর উপর index থাকলে → মিলিসেকেন্ডে result আসবে

```sql
-- বাংলাদেশী e-commerce product table
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    category_id INT,
    price DECIMAL(10,2),
    seller_phone VARCHAR(14),  -- বাংলাদেশী ফোন নম্বর: +8801XXXXXXXXX
    district VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product search দ্রুত করতে index
CREATE INDEX idx_products_name ON products(name);

-- Phone number দিয়ে seller lookup
CREATE INDEX idx_products_seller_phone ON products(seller_phone);

-- District ভিত্তিক product filter
CREATE INDEX idx_products_district ON products(district);
```

---

## 📊 Index Internal Structure ডায়াগ্রাম

### B-Tree Structure Visualization

```
                    ┌─────────────────────┐
                    │     Root Node       │
                    │   [30]  [60]  [90]  │
                    └──┬──────┬──────┬────┘
                       │      │      │
          ┌────────────┘      │      └────────────┐
          ▼                   ▼                    ▼
   ┌─────────────┐   ┌─────────────┐    ┌──────────────┐
   │ Internal     │   │ Internal     │    │ Internal      │
   │ [10] [20]   │   │ [40] [50]   │    │ [70] [80]    │
   └──┬────┬──┬──┘   └──┬────┬──┬──┘    └──┬────┬──┬───┘
      │    │  │          │    │  │           │    │  │
      ▼    ▼  ▼          ▼    ▼  ▼           ▼    ▼  ▼
   ┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐
   │1-9 ││10- ││20- ││30- ││40- ││50- ││60- ││70- ││80- │
   │    ││ 19 ││ 29 ││ 39 ││ 49 ││ 59 ││ 69 ││ 79 ││ 89 │
   └────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘
   Leaf   Leaf  Leaf  Leaf  Leaf  Leaf  Leaf  Leaf  Leaf

   B-Tree: প্রতিটি node-এ data এবং key দুটোই থাকে
   Branching Factor (order) = 4 এই উদাহরণে
   Height = O(log_b(N)), যেখানে b = branching factor
```

### B+ Tree Leaf Node Linked List (InnoDB ব্যবহার করে)

```
                    ┌──────────────────┐
                    │    Root Node     │
                    │   [30]    [60]   │
                    └──┬─────────┬─────┘
                       │         │
          ┌────────────┘         └────────────┐
          ▼                                    ▼
   ┌─────────────┐                     ┌─────────────┐
   │ Internal     │                     │ Internal     │
   │ [10] [20]   │                     │ [40] [50]   │
   └──┬────┬──┬──┘                     └──┬────┬──┬──┘
      │    │  │                            │    │  │
      ▼    ▼  ▼                            ▼    ▼  ▼

   ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
   │ 1-9  │→ │10-19 │→ │20-29 │→ │30-39 │→ │40-49 │→ ...
   │[ptr] │← │[ptr] │← │[ptr] │← │[ptr] │← │[ptr] │
   └──────┘  └──────┘  └──────┘  └──────┘  └──────┘
      Leaf nodes doubly-linked list (range scan-এ কাজে লাগে)

   B+ Tree পার্থক্য:
   ✓ শুধু leaf node-এ actual data/pointer থাকে
   ✓ Internal node-এ শুধু key এবং child pointer
   ✓ Leaf node-গুলো linked list-এ সংযুক্ত → range scan দ্রুত
   ✓ InnoDB-র default index structure
```

### Hash Index Bucket Diagram

```
   Key          Hash Function        Bucket Array
   ─────        ─────────────        ────────────
   "dhaka"  ──→ hash("dhaka")=2  ──→ [0] → NULL
   "sylhet" ──→ hash("sylhet")=5 ──→ [1] → NULL
   "ctg"    ──→ hash("ctg")=2   ──→ [2] → ("dhaka",ptr) → ("ctg",ptr)  ← collision!
   "khulna" ──→ hash("khulna")=4 ──→ [3] → NULL
                                      [4] → ("khulna",ptr)
                                      [5] → ("sylhet",ptr)

   Hash Index:
   ✓ O(1) average lookup (equality only)
   ✗ Range query সমর্থন করে না (WHERE price > 100)
   ✗ ORDER BY-তে কাজ করে না
   ✗ Collision handle করতে হয় (chaining/open addressing)
```

---

## 💻 Index Types Deep Dive

### ১. B-Tree / B+ Tree Index

B-Tree (Balanced Tree) হলো সবচেয়ে সাধারণ index type। MySQL InnoDB এবং PostgreSQL উভয়ের default index structure।

**কীভাবে কাজ করে:**
- Tree সর্বদা balanced থাকে — root থেকে যেকোনো leaf-এর দূরত্ব সমান
- প্রতিটি node-এ একাধিক key থাকে (branching factor অনুযায়ী)
- Lookup complexity: **O(log N)**
- ১ কোটি row-এর table-এ মাত্র ৩-৪ level traversal লাগে

**কোন ধরনের query-তে কাজ করে:**

```sql
-- ✅ Equality query
SELECT * FROM users WHERE email = 'karim@example.com';

-- ✅ Range query
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31';

-- ✅ Prefix LIKE
SELECT * FROM products WHERE name LIKE 'Samsung%';

-- ✅ ORDER BY (sort এড়ানো যায়)
SELECT * FROM transactions ORDER BY amount DESC LIMIT 10;

-- ❌ Suffix LIKE (index ব্যবহার হয় না)
SELECT * FROM products WHERE name LIKE '%Galaxy';
```

**SQL Examples:**

```sql
-- MySQL: B-Tree index তৈরি
CREATE INDEX idx_users_email ON users(email);

-- PostgreSQL: B-Tree index (explicitly)
CREATE INDEX idx_users_email ON users USING btree(email);

-- Index তৈরির পর EXPLAIN দিয়ে যাচাই
EXPLAIN SELECT * FROM users WHERE email = 'karim@example.com';

-- MySQL EXPLAIN output:
-- +----+-------------+-------+------+----------------+----------------+---------+-------+------+-------+
-- | id | select_type | table | type | possible_keys  | key            | key_len | ref   | rows | Extra |
-- +----+-------------+-------+------+----------------+----------------+---------+-------+------+-------+
-- |  1 | SIMPLE      | users | ref  | idx_users_email| idx_users_email| 767     | const |    1 |       |
-- +----+-------------+-------+------+----------------+----------------+---------+-------+------+-------+
-- type = "ref" মানে index ব্যবহার হচ্ছে ✅
-- type = "ALL" মানে full table scan ❌
```

**PHP (Laravel) Example:**

```php
// Migration-এ index তৈরি
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('email')->unique(); // unique index
    $table->string('phone', 14)->index(); // regular B-Tree index
    $table->string('district')->index();
    $table->timestamps();
});

// Eloquent query — index ব্যবহার হবে
$user = User::where('email', 'karim@example.com')->first();

// EXPLAIN দিয়ে query plan দেখুন
DB::enableQueryLog();
$users = User::where('district', 'Dhaka')->get();
$queries = DB::getQueryLog();

// Raw EXPLAIN
$plan = DB::select('EXPLAIN SELECT * FROM users WHERE district = ?', ['Dhaka']);
```

**JavaScript (Sequelize) Example:**

```javascript
// Model definition with index
const User = sequelize.define('User', {
    email: {
        type: DataTypes.STRING,
        unique: true, // unique index তৈরি হবে
    },
    phone: DataTypes.STRING(14),
    district: DataTypes.STRING(50),
}, {
    indexes: [
        { fields: ['phone'] },          // B-Tree index
        { fields: ['district'] },        // B-Tree index
    ]
});

// Query — index ব্যবহার হবে
const user = await User.findOne({ where: { email: 'karim@example.com' } });

// Raw EXPLAIN query
const [results] = await sequelize.query(
    'EXPLAIN ANALYZE SELECT * FROM "Users" WHERE district = :district',
    { replacements: { district: 'Dhaka' } }
);
console.log(results);
```

---

### ২. Hash Index

Hash index একটি hash function ব্যবহার করে key-কে সরাসরি bucket location-এ map করে।

**কীভাবে কাজ করে:**
1. Key → hash function → bucket number
2. Bucket-এ (key, row pointer) pair সংরক্ষিত
3. Average case: **O(1)** lookup
4. Worst case (collision): O(N)

**সীমাবদ্ধতা:**
- শুধু equality query (`=`, `IN`) — range query (`>`, `<`, `BETWEEN`) সমর্থন করে না
- ORDER BY-তে কাজ করে না
- Prefix matching (`LIKE 'abc%'`) সমর্থন করে না

```sql
-- MySQL: MEMORY engine-এ hash index
CREATE TABLE session_cache (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id BIGINT,
    data JSON,
    INDEX idx_user USING HASH (user_id)
) ENGINE = MEMORY;

-- PostgreSQL: Hash index
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- ✅ Hash index কাজ করবে
SELECT * FROM users WHERE email = 'rahim@example.com';

-- ❌ Hash index কাজ করবে না
SELECT * FROM users WHERE email > 'r%';
SELECT * FROM users ORDER BY email;
```

**কখন Hash vs B-Tree ব্যবহার করবেন:**

| বৈশিষ্ট্য | Hash Index | B-Tree Index |
|---|---|---|
| Equality (`=`) | ✅ O(1) | ✅ O(log N) |
| Range (`>`, `<`) | ❌ | ✅ |
| ORDER BY | ❌ | ✅ |
| Prefix LIKE | ❌ | ✅ |
| Storage | কম | বেশি |
| ব্যবহারের ক্ষেত্র | Session lookup, cache key | সাধারণ সকল ক্ষেত্র |

---

### ৩. Composite/Compound Index

একাধিক column মিলিয়ে একটি index তৈরি করা হলে সেটি Composite Index। এটি indexing-এর সবচেয়ে গুরুত্বপূর্ণ এবং ভুল হওয়ার সম্ভাবনা বেশি এমন একটি ক্ষেত্র।

**Leftmost Prefix Rule — সবচেয়ে গুরুত্বপূর্ণ নিয়ম:**

```sql
-- Composite index তৈরি
CREATE INDEX idx_orders_composite 
ON orders(district, status, created_at);

-- এই index কোন query-তে কাজ করবে:
-- ✅ সব ৩টি column (পুরো index ব্যবহৃত)
SELECT * FROM orders WHERE district = 'Dhaka' AND status = 'delivered' AND created_at > '2024-01-01';

-- ✅ প্রথম ২টি column
SELECT * FROM orders WHERE district = 'Dhaka' AND status = 'pending';

-- ✅ শুধু প্রথম column
SELECT * FROM orders WHERE district = 'Chittagong';

-- ❌ প্রথম column ছাড়া (index ব্যবহৃত হবে না!)
SELECT * FROM orders WHERE status = 'pending';

-- ❌ মাঝের column skip করলে (শুধু district অংশ ব্যবহৃত হবে)
SELECT * FROM orders WHERE district = 'Dhaka' AND created_at > '2024-01-01';
```

**Column Order কেন গুরুত্বপূর্ণ — Selectivity ও Cardinality:**

```sql
-- Cardinality পরীক্ষা করুন
SELECT 
    COUNT(DISTINCT district) AS district_cardinality,    -- ~64 (কম)
    COUNT(DISTINCT status) AS status_cardinality,        -- ~5 (খুব কম)
    COUNT(DISTINCT user_id) AS user_cardinality,         -- ~লক্ষাধিক (বেশি)
    COUNT(*) AS total_rows
FROM orders;

-- নিয়ম: বেশি selective (high cardinality) column আগে রাখুন
-- ভালো: (user_id, status) — user_id বেশি selective
-- খারাপ: (status, user_id) — status মাত্র ৫টি value
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**PHP (Laravel) Example:**

```php
// Migration-এ composite index
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id');
    $table->string('status');
    $table->string('district');
    $table->decimal('total_amount', 12, 2);
    $table->timestamps();

    // Composite index — column order গুরুত্বপূর্ণ!
    $table->index(['user_id', 'status', 'created_at']);
    $table->index(['district', 'status']);
});

// এই query composite index ব্যবহার করবে
$orders = Order::where('user_id', 1234)
    ->where('status', 'delivered')
    ->where('created_at', '>=', '2024-01-01')
    ->get();
```

**JavaScript (Knex) Example:**

```javascript
// Knex migration
exports.up = function(knex) {
    return knex.schema.createTable('orders', (table) => {
        table.bigIncrements('id');
        table.bigInteger('user_id').notNullable();
        table.string('status', 20);
        table.string('district', 50);
        table.decimal('total_amount', 12, 2);
        table.timestamps(true, true);

        // Composite index
        table.index(['user_id', 'status', 'created_at']);
        table.index(['district', 'status']);
    });
};

// Knex query
const orders = await knex('orders')
    .where({ user_id: 1234, status: 'delivered' })
    .where('created_at', '>=', '2024-01-01');
```

**EXPLAIN দিয়ে যাচাই:**

```sql
EXPLAIN SELECT * FROM orders 
WHERE user_id = 1234 AND status = 'delivered' AND created_at >= '2024-01-01';

-- আদর্শ output:
-- key: idx_orders_user_status_created
-- key_len: 30 (সব ৩টি column ব্যবহৃত হচ্ছে)
-- type: range
-- rows: 15 (খুব কম row scan)
```

---

### ৪. Covering Index

Covering index এমন একটি index যেখানে query-র প্রয়োজনীয় **সব column** index-এর মধ্যেই আছে। ফলে table-এ ফিরে গিয়ে data পড়ার দরকার হয় না (table lookup / bookmark lookup এড়ানো যায়)।

```sql
-- ধরুন এই query প্রায়ই চলে:
SELECT user_id, status, total_amount 
FROM orders 
WHERE user_id = 1234 AND status = 'delivered';

-- সাধারণ index:
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- এটি row খুঁজে পাবে, কিন্তু total_amount পড়তে table-এ যেতে হবে

-- Covering index (total_amount INCLUDE করা):
-- PostgreSQL:
CREATE INDEX idx_orders_covering 
ON orders(user_id, status) INCLUDE (total_amount);

-- MySQL: INCLUDE নেই, তাই column যোগ করুন
CREATE INDEX idx_orders_covering 
ON orders(user_id, status, total_amount);

-- EXPLAIN output-এ "Using index" বা "Index Only Scan" দেখলে বুঝবেন covering index কাজ করছে
EXPLAIN SELECT user_id, status, total_amount 
FROM orders WHERE user_id = 1234 AND status = 'delivered';
-- Extra: Using index ← MySQL
-- Index Only Scan ← PostgreSQL
```

**Performance পার্থক্য:**

```
Regular Index:     Index → Leaf (row pointer) → Table Page → Data
                   [I/O 1]  [I/O 2]             [I/O 3]

Covering Index:    Index → Leaf (সব data এখানেই আছে) → Done!
                   [I/O 1]  [I/O 2]

Random I/O কমে = Performance বাড়ে (বিশেষত HDD-তে)
```

**PHP (Laravel) Example:**

```php
// Covering index migration
Schema::table('orders', function (Blueprint $table) {
    // user_id + status দিয়ে search, total_amount ও created_at select
    $table->index(['user_id', 'status', 'total_amount', 'created_at'], 
                  'idx_orders_covering');
});

// এই query-তে table lookup লাগবে না
$summary = Order::select('user_id', 'status', 'total_amount', 'created_at')
    ->where('user_id', 1234)
    ->where('status', 'delivered')
    ->get();
```

---

### ৫. Full-Text Index

সাধারণ B-Tree index `LIKE '%keyword%'` query-তে কাজ করে না। Full-text index natural language text search-এর জন্য বিশেষভাবে তৈরি।

**MySQL FULLTEXT Index:**

```sql
-- Full-text index তৈরি
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    description TEXT,
    FULLTEXT INDEX ft_idx_name_desc (name, description)
) ENGINE = InnoDB;

-- Natural Language Mode search
SELECT *, MATCH(name, description) AGAINST('স্যামসাং মোবাইল' IN NATURAL LANGUAGE MODE) AS relevance
FROM products
WHERE MATCH(name, description) AGAINST('স্যামসাং মোবাইল' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC;

-- Boolean Mode (AND, OR, NOT সমর্থন করে)
SELECT * FROM products
WHERE MATCH(name, description) AGAINST('+samsung -refurbished +galaxy' IN BOOLEAN MODE);

-- Query Expansion (সম্পর্কিত শব্দও খোঁজে)
SELECT * FROM products
WHERE MATCH(name, description) AGAINST('mobile phone' WITH QUERY EXPANSION);
```

**PostgreSQL GIN/GiST Full-Text:**

```sql
-- tsvector column এবং GIN index
ALTER TABLE products ADD COLUMN search_vector tsvector;

UPDATE products SET search_vector = 
    to_tsvector('english', COALESCE(name, '') || ' ' || COALESCE(description, ''));

CREATE INDEX idx_products_fts ON products USING gin(search_vector);

-- tsquery দিয়ে search
SELECT name, ts_rank(search_vector, query) AS rank
FROM products, to_tsquery('english', 'samsung & galaxy & !refurbished') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Auto-update trigger
CREATE FUNCTION products_search_trigger() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', 
        COALESCE(NEW.name, '') || ' ' || COALESCE(NEW.description, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_search
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION products_search_trigger();
```

**JavaScript (Sequelize) Example:**

```javascript
// PostgreSQL full-text search with Sequelize
const { Op, literal } = require('sequelize');

const results = await Product.findAll({
    where: literal(
        `search_vector @@ to_tsquery('english', 'samsung & galaxy')`
    ),
    order: literal(
        `ts_rank(search_vector, to_tsquery('english', 'samsung & galaxy')) DESC`
    ),
    limit: 20,
});
```

---

### ৬. Partial/Filtered Index (PostgreSQL)

Partial index শুধু নির্দিষ্ট শর্ত পূরণকারী row-গুলোকে index করে। ফলে index size অনেক ছোট হয় এবং maintenance cost কমে।

```sql
-- শুধু active user-দের index করুন (৯০% user inactive হতে পারে)
CREATE INDEX idx_users_active_email 
ON users(email) 
WHERE status = 'active';

-- শুধু pending order-দের index (বেশিরভাগ order delivered/cancelled)
CREATE INDEX idx_orders_pending 
ON orders(created_at, user_id) 
WHERE status = 'pending';

-- NULL নয় এমন row-দের index
CREATE INDEX idx_users_phone 
ON users(phone) 
WHERE phone IS NOT NULL;

-- ব্যবহার: query-তে WHERE clause match করতে হবে
-- ✅ Index ব্যবহৃত হবে:
SELECT * FROM users WHERE email = 'karim@bd.com' AND status = 'active';

-- ❌ Index ব্যবহৃত হবে না:
SELECT * FROM users WHERE email = 'karim@bd.com' AND status = 'inactive';
```

**Use Cases (বাংলাদেশ প্রেক্ষাপট):**
- E-commerce: শুধু `status = 'pending'` order-এ index — দ্রুত processing queue
- bKash/Nagad transaction: শুধু `status = 'processing'` — real-time settlement
- Job portal: শুধু `is_active = true` job posting-এ index

---

### ৭. Expression/Functional Index

Computed expression-এর উপর index তৈরি করা যায়। এটি এমন query-র জন্য উপযোগী যেখানে column-এ function apply হয়।

```sql
-- সমস্যা: LOWER() ব্যবহারে regular index কাজ করে না
-- ❌ Index ব্যবহৃত হবে না:
SELECT * FROM users WHERE LOWER(email) = 'karim@example.com';

-- সমাধান: Expression index (PostgreSQL)
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- ✅ এখন index ব্যবহৃত হবে:
SELECT * FROM users WHERE LOWER(email) = 'karim@example.com';

-- Date function-এর উপর index
CREATE INDEX idx_orders_year_month 
ON orders(EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at));

-- JSON field index (PostgreSQL)
CREATE INDEX idx_users_settings_lang 
ON users((settings->>'language'));

SELECT * FROM users WHERE settings->>'language' = 'bn';

-- MySQL: Generated column + index (expression index alternative)
ALTER TABLE users 
ADD COLUMN email_lower VARCHAR(255) GENERATED ALWAYS AS (LOWER(email)) STORED;

CREATE INDEX idx_users_email_lower ON users(email_lower);
```

---

### ৮. Spatial Index

Geographic data (latitude, longitude, polygon) query-র জন্য R-Tree ভিত্তিক Spatial index ব্যবহৃত হয়।

```sql
-- MySQL: Spatial index
CREATE TABLE stores (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    location POINT NOT NULL SRID 4326,
    SPATIAL INDEX idx_stores_location (location)
);

-- নিকটতম দোকান খোঁজা (ঢাকার একটি স্থান থেকে)
SELECT name,
    ST_Distance_Sphere(
        location, 
        ST_GeomFromText('POINT(90.4125 23.8103)', 4326)  -- ঢাকা
    ) AS distance_meters
FROM stores
WHERE ST_Distance_Sphere(
    location,
    ST_GeomFromText('POINT(90.4125 23.8103)', 4326)
) < 5000  -- ৫ কিমি ব্যাসার্ধে
ORDER BY distance_meters
LIMIT 10;

-- PostgreSQL: PostGIS সহ
CREATE INDEX idx_stores_location ON stores USING gist(location);

SELECT name, ST_Distance(location, ST_MakePoint(90.4125, 23.8103)::geography) AS dist
FROM stores
WHERE ST_DWithin(location, ST_MakePoint(90.4125, 23.8103)::geography, 5000)
ORDER BY dist;
```

---

### ৯. Unique Index vs Primary Key

| বৈশিষ্ট্য | Primary Key | Unique Index |
|---|---|---|
| NULL অনুমতি | ❌ | ✅ (একটি NULL) |
| প্রতি table-এ সংখ্যা | ১টি মাত্র | একাধিক |
| Clustered (InnoDB) | ✅ হ্যাঁ | ❌ না |
| Auto-created index | ✅ | ❌ (CREATE INDEX দরকার) |
| Foreign Key reference | ✅ সাধারণত | ✅ সম্ভব |

```sql
-- Primary Key
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(14),
    nid VARCHAR(17),
    
    UNIQUE INDEX idx_email (email),         -- email unique
    UNIQUE INDEX idx_phone (phone),         -- phone unique (NULL allowed)
    UNIQUE INDEX idx_nid (nid)              -- NID unique (NULL allowed)
);

-- Composite unique index
CREATE UNIQUE INDEX idx_enrollment 
ON course_enrollments(user_id, course_id);  -- একজন user একটি course-এ একবারই enroll হতে পারবে
```

---

## 🔥 Advanced Scenarios

### ১. EXPLAIN / Query Plan Analysis

Query optimization-এর সবচেয়ে শক্তিশালী হাতিয়ার হলো EXPLAIN। এটি দিয়ে দেখা যায় database engine কীভাবে query execute করার plan করছে।

**MySQL EXPLAIN Format:**

```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 1234 AND status = 'delivered';

-- JSON format (বিস্তারিত তথ্য)
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 1234;

-- EXPLAIN ANALYZE (actual execution — MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1234;
```

**MySQL EXPLAIN-এর গুরুত্বপূর্ণ column:**

| Column | ব্যাখ্যা | ভালো মান |
|---|---|---|
| `type` | Access method | system > const > eq_ref > ref > range > index > ALL |
| `key` | ব্যবহৃত index | NULL না হওয়া উচিত |
| `rows` | আনুমানিক scan করা row | যত কম তত ভালো |
| `Extra` | অতিরিক্ত তথ্য | "Using index" ভালো, "Using filesort" খারাপ |

**PostgreSQL EXPLAIN ANALYZE:**

```sql
-- Plan + actual execution time
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 1234 AND status = 'delivered';

-- Output পড়া:
-- Index Scan using idx_orders_user_status on orders  (cost=0.43..8.45 rows=1 width=120)
--                                                     (actual time=0.023..0.025 rows=1 loops=1)
--   Index Cond: ((user_id = 1234) AND (status = 'delivered'))
--   Buffers: shared hit=4
-- Planning Time: 0.150 ms
-- Execution Time: 0.045 ms
```

**Scan Types বোঝা:**

```
┌─────────────────────────────────────────────────────────┐
│ Scan Type              │ ব্যাখ্যা          │ Performance │
├─────────────────────────────────────────────────────────┤
│ Seq Scan               │ Full table scan    │ ❌ সবচেয়ে ধীর│
│ Index Scan             │ Index → Table      │ ✅ দ্রুত     │
│ Index Only Scan        │ শুধু Index (cover) │ ✅✅ সবচেয়ে  │
│ Bitmap Index Scan      │ Bitmap → Table     │ ✅ মাঝারি    │
│ Bitmap Heap Scan       │ Bitmap ফলাফল পড়া  │ ✅ মাঝারি    │
└─────────────────────────────────────────────────────────┘
```

**PHP: Query Plan Analysis:**

```php
// Laravel-এ query log enable
DB::enableQueryLog();

$orders = Order::where('user_id', 1234)
    ->where('status', 'delivered')
    ->whereDate('created_at', '>=', '2024-01-01')
    ->get();

// Log থেকে query দেখুন
$queries = DB::getQueryLog();
foreach ($queries as $query) {
    Log::info('Query: ' . $query['query']);
    Log::info('Time: ' . $query['time'] . 'ms');
    
    // EXPLAIN চালান
    $explain = DB::select('EXPLAIN ' . $query['query'], $query['bindings']);
    Log::info('Plan: ', (array) $explain[0]);
}

// Laravel Debugbar / Telescope integration
// config/telescope.php এ query watcher enable করুন
```

**JavaScript: EXPLAIN with Sequelize:**

```javascript
// Sequelize-এ query logging
const sequelize = new Sequelize(database, username, password, {
    logging: (sql, timing) => {
        console.log(`[${timing}ms] ${sql}`);
    },
    benchmark: true,
});

// Raw EXPLAIN query
async function analyzeQuery(sql, replacements = {}) {
    const [plan] = await sequelize.query(`EXPLAIN ANALYZE ${sql}`, {
        replacements,
        type: QueryTypes.SELECT,
    });
    
    plan.forEach(row => console.log(row['QUERY PLAN']));
    return plan;
}

// ব্যবহার
await analyzeQuery(
    'SELECT * FROM orders WHERE user_id = :userId AND status = :status',
    { userId: 1234, status: 'delivered' }
);
```

---

### ২. Index Optimization Strategies

**Missing Index চিহ্নিত করা (Slow Query Log):**

```sql
-- MySQL: Slow query log enable
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- ১ সেকেন্ডের বেশি সময় লাগা query log হবে
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Slow query log দেখুন
-- /var/log/mysql/mysql-slow.log

-- PostgreSQL: Slow query detection
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 1 second
SELECT pg_reload_conf();
```

**Unused Index চিহ্নিত করা:**

```sql
-- MySQL: Unused index খোঁজা
SELECT 
    s.table_name,
    s.index_name,
    s.column_name,
    t.table_rows
FROM information_schema.statistics s
JOIN information_schema.tables t 
    ON s.table_name = t.table_name AND s.table_schema = t.table_schema
LEFT JOIN performance_schema.table_io_waits_summary_by_index_usage w 
    ON s.index_name = w.INDEX_NAME AND s.table_name = w.OBJECT_NAME
WHERE w.COUNT_READ = 0
    AND s.table_schema = 'your_database'
    AND s.index_name != 'PRIMARY';

-- PostgreSQL: Unused index (pg_stat_user_indexes)
SELECT 
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Index Bloat ও Maintenance:**

```sql
-- PostgreSQL: Index bloat পরীক্ষা
SELECT 
    nspname || '.' || relname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    last_idx_scan
FROM pg_stat_user_indexes
JOIN pg_class ON pg_class.oid = indexrelid
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- REINDEX (PostgreSQL)
REINDEX INDEX idx_orders_user_status;
REINDEX TABLE orders;
REINDEX INDEX CONCURRENTLY idx_orders_user_status;  -- production-এ non-blocking

-- MySQL: Index optimize
OPTIMIZE TABLE orders;
ALTER TABLE orders ENGINE=InnoDB;  -- InnoDB table rebuild
ANALYZE TABLE orders;              -- Index statistics update
```

---

### ৩. Clustered vs Non-Clustered Index

**InnoDB-তে Clustered Index:**

InnoDB-তে PRIMARY KEY হলো clustered index। Table data physically primary key অনুযায়ী সাজানো থাকে।

```
┌─────────────────────────────────────────────────────┐
│                 Clustered Index (InnoDB)              │
│                                                       │
│  Primary Key B+ Tree:                                │
│  ┌─────┐                                             │
│  │Root │ → [50]                                      │
│  └──┬──┘                                             │
│     ├──────────────┐                                 │
│     ▼              ▼                                 │
│  ┌──────┐      ┌──────┐                             │
│  │[10,30]│      │[70,90]│                            │
│  └──┬─┬──┘      └──┬─┬──┘                           │
│     ▼  ▼           ▼  ▼                              │
│  [Leaf: id=10,     [Leaf: id=70,                     │
│   name=Karim,       name=Rahim,                      │
│   email=k@bd.com]   email=r@bd.com]                  │
│   ↑ পুরো row data leaf-এ আছে!                       │
│                                                       │
│  Secondary Index (email):                             │
│  ┌──────────────────┐                                │
│  │ k@bd.com → PK=10 │  ← Primary Key store করে      │
│  │ r@bd.com → PK=70 │  ← row pointer নয়!           │
│  └──────────────────┘                                │
│                                                       │
│  Secondary index lookup:                              │
│  email index → PK value পাওয়া → PK index-এ lookup   │
│  (double lookup — bookmark lookup)                    │
└─────────────────────────────────────────────────────┘
```

```sql
-- InnoDB: PRIMARY KEY = Clustered Index (automatically)
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,  -- clustered index
    user_id BIGINT,
    amount DECIMAL(12,2),
    created_at TIMESTAMP,
    INDEX idx_user (user_id),              -- secondary index (non-clustered)
    INDEX idx_created (created_at)         -- secondary index
);

-- Secondary index lookup-এ double lookup হয়:
-- 1. idx_user-এ user_id=1234 search → PK=5678 পাওয়া
-- 2. Clustered index-এ PK=5678 search → row data পাওয়া

-- PostgreSQL CLUSTER command (one-time physical reorder)
CLUSTER orders USING idx_orders_created_at;
-- ⚠️ Table lock নেয়, VACUUM FULL-এর মতো
-- নতুন INSERT-এ আবার unordered হয়ে যায়
```

**Performance Implications:**

| বিষয় | Clustered | Non-Clustered |
|---|---|---|
| Range scan by PK | অত্যন্ত দ্রুত (sequential I/O) | Random I/O |
| Secondary index lookup | ধীর (double lookup) | সরাসরি row pointer |
| INSERT order | Sequential PK দ্রুত, Random PK ধীর | কম প্রভাব |
| UUID as PK | ❌ খারাপ (random insert) | কম সমস্যা |

---

### ৪. Index and Locking

Index locking behavior বোঝা high-concurrency system-এ অত্যন্ত গুরুত্বপূর্ণ।

```sql
-- InnoDB Gap Lock example
-- Transaction 1:
BEGIN;
SELECT * FROM orders WHERE user_id = 100 FOR UPDATE;
-- user_id=100-এর row lock + gap lock

-- Transaction 2 (blocked!):
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (100, 500);
-- ❌ BLOCKED — gap lock-এর কারণে

-- Gap lock সমস্যা কমাতে:
-- 1. Index ব্যবহার নিশ্চিত করুন (index ছাড়া full table lock হতে পারে)
-- 2. Transaction ছোট রাখুন
-- 3. READ COMMITTED isolation level বিবেচনা করুন
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Deadlock scenario:
-- T1: UPDATE orders SET status='done' WHERE id=1; (locks row 1)
-- T2: UPDATE orders SET status='done' WHERE id=2; (locks row 2)
-- T1: UPDATE orders SET status='done' WHERE id=2; (waits for T2)
-- T2: UPDATE orders SET status='done' WHERE id=1; (waits for T1) → DEADLOCK!

-- Index থাকলে deadlock কম হয় কারণ:
-- Index-এ নির্দিষ্ট row lock হয়, index ছাড়া বেশি row/page lock হয়

-- Lock monitoring (MySQL)
SELECT * FROM performance_schema.data_locks;
SELECT * FROM information_schema.innodb_lock_waits;
SHOW ENGINE INNODB STATUS;  -- LATEST DETECTED DEADLOCK section
```

---

### ৫. Indexing JSON Columns

আধুনিক application-এ JSON column প্রচলিত। সঠিক indexing ছাড়া JSON query অত্যন্ত ধীর।

**MySQL JSON Index (Generated Column approach):**

```sql
-- MySQL-এ JSON column-এ সরাসরি index দেওয়া যায় না
CREATE TABLE user_profiles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    data JSON NOT NULL
);

-- Generated column + index
ALTER TABLE user_profiles 
ADD COLUMN city VARCHAR(100) GENERATED ALWAYS AS (JSON_UNQUOTE(data->>'$.address.city')) STORED,
ADD INDEX idx_city (city);

-- Multi-valued index (MySQL 8.0.17+)
CREATE INDEX idx_tags ON user_profiles((CAST(data->>'$.tags' AS CHAR(50) ARRAY)));

-- Query
SELECT * FROM user_profiles 
WHERE 'developer' MEMBER OF (data->>'$.tags');
```

**PostgreSQL JSONB + GIN Index:**

```sql
-- JSONB column-এ GIN index (সবচেয়ে শক্তিশালী)
CREATE TABLE user_profiles (
    id BIGSERIAL PRIMARY KEY,
    data JSONB NOT NULL
);

-- GIN index — সব key-value pair index হবে
CREATE INDEX idx_profiles_data ON user_profiles USING gin(data);

-- নির্দিষ্ট path-এ index
CREATE INDEX idx_profiles_city ON user_profiles USING gin((data->'address'));

-- Expression index on specific JSON key
CREATE INDEX idx_profiles_city_btree ON user_profiles((data->>'city'));

-- Query examples
-- ✅ GIN index ব্যবহৃত হবে:
SELECT * FROM user_profiles WHERE data @> '{"address": {"city": "Dhaka"}}';
SELECT * FROM user_profiles WHERE data ? 'premium_member';
SELECT * FROM user_profiles WHERE data->>'city' = 'Dhaka';
```

**JavaScript (Mongoose) — MongoDB context:**

```javascript
// Mongoose schema-তে nested field index
const userProfileSchema = new mongoose.Schema({
    name: String,
    address: {
        city: { type: String, index: true },   // nested field index
        district: String,
    },
    tags: [{ type: String, index: true }],      // array field index
    metadata: mongoose.Schema.Types.Mixed,
});

// Compound index on nested fields
userProfileSchema.index({ 'address.city': 1, 'address.district': 1 });

// Text index for search
userProfileSchema.index({ name: 'text', 'address.city': 'text' });

// Query with index
const users = await UserProfile.find({ 'address.city': 'Dhaka' })
    .explain('executionStats');
```

---

### ৬. Indexing for Laravel Eloquent

Laravel project-এ সঠিক indexing strategy অত্যন্ত গুরুত্বপূর্ণ। অনেক developer migration-এ index দিতে ভুলে যান।

```php
// ========================================
// Migration-এ Index Methods
// ========================================
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('sku')->unique();              // Unique index
    $table->foreignId('category_id')->constrained(); // Foreign key + index
    $table->string('seller_phone', 14)->index();  // Regular index
    $table->decimal('price', 10, 2);
    $table->enum('status', ['active', 'inactive', 'draft']);
    $table->text('description')->nullable();
    $table->json('attributes')->nullable();
    $table->timestamps();
    $table->softDeletes();

    // Composite index
    $table->index(['category_id', 'status', 'price'], 'idx_cat_status_price');
    
    // Full-text index (MySQL)
    $table->fullText(['name', 'description'], 'ft_product_search');
});

// ========================================
// Separate migration-এ index যোগ/সরানো
// ========================================
Schema::table('products', function (Blueprint $table) {
    $table->index(['status', 'created_at']);       // Index যোগ
    $table->dropIndex('idx_cat_status_price');     // Index সরানো
    $table->renameIndex('old_name', 'new_name');   // Index rename
});

// ========================================
// Eloquent Query Optimization with Indexes
// ========================================

// ❌ খারাপ: N+1 query + index ছাড়া
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name;  // প্রতি loop-এ আলাদা query
}

// ✅ ভালো: Eager loading + indexed foreign key
$orders = Order::with('user')->where('status', 'pending')->get();

// ❌ খারাপ: function ব্যবহারে index কাজ করে না
$users = User::whereRaw('YEAR(created_at) = 2024')->get();

// ✅ ভালো: range query — index ব্যবহৃত হবে
$users = User::whereBetween('created_at', ['2024-01-01', '2024-12-31'])->get();

// ========================================
// Laravel Telescope / Debugbar দিয়ে Analysis
// ========================================

// AppServiceProvider-এ slow query detection
DB::listen(function ($query) {
    if ($query->time > 1000) { // ১ সেকেন্ডের বেশি
        Log::warning('Slow Query Detected', [
            'sql' => $query->sql,
            'bindings' => $query->bindings,
            'time_ms' => $query->time,
            'explain' => DB::select('EXPLAIN ' . $query->sql, $query->bindings),
        ]);
    }
});
```

---

### ৭. Indexing for Large Tables

বড় table-এ (কোটি+ row) index তৈরি বা পরিবর্তন করা production-এ বিপজ্জনক হতে পারে কারণ এটি table lock নিতে পারে।

**Online Index Creation:**

```sql
-- PostgreSQL: CONCURRENTLY (non-blocking)
CREATE INDEX CONCURRENTLY idx_orders_amount ON orders(amount);
-- ⚠️ বেশি সময় নেয়, কিন্তু table lock হয় না
-- ⚠️ Transaction block-এর ভিতরে ব্যবহার করা যায় না

-- MySQL 8.0: Online DDL (default)
ALTER TABLE orders ADD INDEX idx_amount(amount), ALGORITHM=INPLACE, LOCK=NONE;

-- pt-online-schema-change (Percona Tool — production safe)
-- বড় MySQL table-এ index যোগ করতে ব্যবহৃত হয়
-- shadow table তৈরি করে, trigger দিয়ে sync করে, rename করে
```

```bash
# pt-online-schema-change ব্যবহার
pt-online-schema-change \
    --alter "ADD INDEX idx_amount(amount)" \
    --execute \
    --max-load "Threads_running=50" \
    --critical-load "Threads_running=200" \
    D=ecommerce,t=orders
```

**Partitioning + Indexing:**

```sql
-- PostgreSQL: Partitioned table with index
CREATE TABLE transactions (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    amount DECIMAL(12,2),
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partition
CREATE TABLE transactions_2024_01 
PARTITION OF transactions 
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE transactions_2024_02 
PARTITION OF transactions 
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Partitioned table-এ index (সব partition-এ automatically তৈরি হবে)
CREATE INDEX idx_transactions_user ON transactions(user_id);

-- Query: partition pruning + index scan
EXPLAIN ANALYZE
SELECT * FROM transactions 
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
AND user_id = 1234;
-- শুধু 2024_01 partition scan হবে + user_id index ব্যবহৃত হবে
```

**PHP (Laravel) — Large Table Index Management:**

```php
// Chunk processing with index hint
Order::where('status', 'pending')
    ->orderBy('id')                  // PK index ব্যবহার — consistent ordering
    ->chunk(1000, function ($orders) {
        foreach ($orders as $order) {
            $order->process();
        }
    });

// Cursor (memory efficient — একটি row একবারে)
foreach (Order::where('status', 'pending')->cursor() as $order) {
    $order->process();
}

// Large table migration with index
Schema::table('orders', function (Blueprint $table) {
    // ⚠️ বড় table-এ raw SQL ব্যবহার করুন
    DB::statement('CREATE INDEX CONCURRENTLY idx_orders_amount ON orders(amount)');
});
```

---

## ✅ সুবিধা

| সুবিধা | বিবরণ |
|---|---|
| **দ্রুত SELECT** | O(N) থেকে O(log N) বা O(1)-এ নামিয়ে আনে |
| **Sort এড়ানো** | ORDER BY-তে filesort এড়ানো যায় |
| **Join acceleration** | JOIN operation অনেক দ্রুত হয় |
| **Unique enforcement** | Data integrity নিশ্চিত করে |
| **Covering index** | Table lookup সম্পূর্ণ এড়ানো যায় |

## ❌ অসুবিধা

| অসুবিধা | বিবরণ |
|---|---|
| **Write penalty** | INSERT/UPDATE/DELETE ধীর হয় |
| **Storage cost** | আলাদা disk space দরকার |
| **Maintenance overhead** | Index fragmentation, REINDEX দরকার |
| **Memory usage** | Buffer pool-এ আলাদা জায়গা নেয় |
| **Wrong index = worse** | ভুল index optimizer-কে বিভ্রান্ত করতে পারে |

---

## ⚠️ সাধারণ ভুল

### ১. Over-Indexing
```sql
-- ❌ প্রতিটি column-এ আলাদা index — অপ্রয়োজনীয়
CREATE INDEX idx_1 ON orders(user_id);
CREATE INDEX idx_2 ON orders(status);
CREATE INDEX idx_3 ON orders(created_at);
CREATE INDEX idx_4 ON orders(user_id, status);      -- idx_1 redundant!
CREATE INDEX idx_5 ON orders(user_id, status, created_at); -- idx_4 redundant!

-- ✅ Redundant index সরান
DROP INDEX idx_1 ON orders;  -- idx_4 ও idx_5 user_id cover করে
DROP INDEX idx_4 ON orders;  -- idx_5 (user_id, status) cover করে
```

### ২. Wrong Column Order
```sql
-- ❌ Low cardinality column আগে
CREATE INDEX idx_bad ON orders(status, user_id);
-- status মাত্র ৫টি value — কম selective

-- ✅ High cardinality column আগে
CREATE INDEX idx_good ON orders(user_id, status);
-- user_id লক্ষাধিক value — বেশি selective
```

### ৩. Low-Cardinality Column-এ Index
```sql
-- ❌ gender column-এ index (মাত্র ২-৩টি value)
CREATE INDEX idx_gender ON users(gender);
-- Full table scan-ই বেশি efficient হতে পারে

-- ✅ Composite index-এর অংশ হিসেবে ঠিক আছে
CREATE INDEX idx_district_gender ON users(district, gender);
```

### ৪. Function ব্যবহারে Index নষ্ট করা
```sql
-- ❌ Column-এ function apply করলে index কাজ করে না
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users WHERE LOWER(email) = 'karim@bd.com';

-- ✅ Range query বা expression index ব্যবহার করুন
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
CREATE INDEX idx_email_lower ON users((LOWER(email)));  -- PostgreSQL
```

### ৫. SELECT * ব্যবহার
```sql
-- ❌ SELECT * covering index নষ্ট করে
SELECT * FROM orders WHERE user_id = 1234;

-- ✅ প্রয়োজনীয় column-ই select করুন
SELECT id, status, total_amount FROM orders WHERE user_id = 1234;
```

---

## 🧪 Index Performance Testing

Index যোগ করার আগে ও পরে benchmark করা অত্যন্ত গুরুত্বপূর্ণ। অনুমানের উপর নির্ভর করবেন না — পরিমাপ করুন।

```sql
-- ========================================
-- Step 1: Index ছাড়া benchmark
-- ========================================

-- MySQL: Query profiling
SET profiling = 1;

SELECT * FROM orders WHERE district = 'Dhaka' AND status = 'pending';

SHOW PROFILES;
-- +----------+------------+--------------------------------------------------+
-- | Query_ID | Duration   | Query                                            |
-- +----------+------------+--------------------------------------------------+
-- |        1 | 2.45000000 | SELECT * FROM orders WHERE district = ...        |
-- +----------+------------+--------------------------------------------------+

EXPLAIN ANALYZE SELECT * FROM orders WHERE district = 'Dhaka' AND status = 'pending';
-- -> Filter: ... (cost=105234 rows=50000) (actual time=1850..2450 rows=12543 loops=1)
--     -> Table scan on orders (cost=105234 rows=5000000) (actual time=0.05..1500 rows=5000000)

-- ========================================
-- Step 2: Index তৈরি
-- ========================================
CREATE INDEX idx_orders_district_status ON orders(district, status);

-- ========================================
-- Step 3: Index-সহ benchmark
-- ========================================
EXPLAIN ANALYZE SELECT * FROM orders WHERE district = 'Dhaka' AND status = 'pending';
-- -> Index lookup on orders using idx_orders_district_status
--    (district='Dhaka', status='pending')
--    (cost=1234 rows=12543) (actual time=0.05..15 rows=12543 loops=1)

-- তুলনা:
-- আগে: 2450ms, 5000000 rows scanned
-- পরে: 15ms, 12543 rows scanned
-- উন্নতি: ~163x দ্রুত! ✅
```

**PostgreSQL Benchmark:**

```sql
-- pg_stat_statements দিয়ে query performance ট্র্যাক
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
    query,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    rows
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## 📏 Index Selection Decision Guide

```
                         Query Analysis শুরু
                              │
                              ▼
                    ┌─────────────────────┐
                    │ WHERE clause আছে?   │
                    └──────┬──────────────┘
                     হ্যাঁ │           │ না
                           ▼           ▼
                 ┌──────────────┐  ORDER BY/GROUP BY
                 │ Equality বা  │  column-এ index দিন
                 │ Range query? │
                 └──┬───────┬───┘
           Equality │       │ Range
                    ▼       ▼
              ┌──────────────────────┐
              │ Column cardinality   │
              │ কত?                  │
              └──┬────────────┬──────┘
             বেশি│            │কম
                 ▼            ▼
          B-Tree Index    Composite index-এর
          তৈরি করুন       অংশ হিসেবে রাখুন
                              │
                              ▼
              ┌───────────────────────────┐
              │ একাধিক column WHERE-এ?    │
              └──────┬────────────────────┘
                হ্যাঁ│
                     ▼
              ┌───────────────────────────┐
              │ Composite Index তৈরি করুন │
              │ (High cardinality আগে)   │
              └───────────────────────────┘
                     │
                     ▼
              ┌───────────────────────────┐
              │ SELECT-এ কম column?       │──হ্যাঁ──→ Covering Index বিবেচনা করুন
              └──────┬────────────────────┘
                     │ না
                     ▼
              ┌───────────────────────────┐
              │ Text search?              │──হ্যাঁ──→ Full-Text Index
              └──────┬────────────────────┘
                     │ না
                     ▼
              ┌───────────────────────────┐
              │ JSON column?              │──হ্যাঁ──→ GIN Index (PostgreSQL)
              └──────┬────────────────────┘          Generated Column (MySQL)
                     │ না
                     ▼
              ┌───────────────────────────┐
              │ Geographic data?          │──হ্যাঁ──→ Spatial Index (R-Tree)
              └───────────────────────────┘
```

---

## 📋 সারসংক্ষেপ — Index Type Comparison Table

| Index Type | Data Structure | Equality | Range | Sort | Storage | ব্যবহারের ক্ষেত্র |
|---|---|---|---|---|---|---|
| **B-Tree** | Balanced Tree | ✅ O(log N) | ✅ | ✅ | মাঝারি | সাধারণ সকল query |
| **Hash** | Hash Table | ✅ O(1) | ❌ | ❌ | কম | Exact match, cache key |
| **Composite** | B-Tree (multi-col) | ✅ | ✅ | ✅ | বেশি | Multi-column WHERE |
| **Covering** | B-Tree + extra cols | ✅ | ✅ | ✅ | বেশি | SELECT-heavy query |
| **Full-Text** | Inverted Index | ❌ | ❌ | Rank | বেশি | Text search |
| **Partial** | B-Tree (filtered) | ✅ | ✅ | ✅ | কম | নির্দিষ্ট subset |
| **Expression** | B-Tree (computed) | ✅ | ✅ | ✅ | মাঝারি | Function-based query |
| **Spatial** | R-Tree | ❌ | Spatial | ❌ | বেশি | Geographic/GIS |
| **GIN** | Generalized Inverted | ✅ | ❌ | ❌ | বেশি | JSONB, array, full-text |
| **BRIN** | Block Range | ❌ | ✅ | ❌ | খুব কম | Time-series, append-only |

### বাংলাদেশ প্রেক্ষাপটে সাধারণ Index Strategy:

| ব্যবসায়িক ক্ষেত্র | Table | প্রস্তাবিত Index |
|---|---|---|
| **E-commerce (Daraz/Evaly)** | products | `(category_id, status, price)`, FULLTEXT `(name, description)` |
| **ফোন নম্বর lookup (bKash)** | users | UNIQUE `(phone)`, `(phone, status)` |
| **Transaction query** | transactions | `(user_id, created_at)`, `(status)` WHERE `status='pending'` |
| **Delivery tracking** | shipments | `(order_id)`, `(district, status)`, SPATIAL `(location)` |
| **Job portal (BDJobs)** | jobs | FULLTEXT `(title, description)`, `(category_id, is_active)` |
| **Real estate (Bikroy)** | listings | `(district, price)`, `(type, status)`, SPATIAL `(coordinates)` |

---

> **মনে রাখবেন:** Index হলো trade-off — পড়ার গতি বাড়ায়, লেখার গতি কমায়। সব সময় EXPLAIN দিয়ে যাচাই করুন, benchmark করুন, এবং production-এ monitoring চালু রাখুন। "Premature optimization is the root of all evil" — কিন্তু database indexing এমন একটি optimization যা দিন ১ থেকেই পরিকল্পনা করা উচিত।
