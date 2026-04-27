# 📐 নরমালাইজেশন (Normalization)

## 📌 সংজ্ঞা ও মূল ধারণা

### Edgar Codd-এর Normalization Theory

১৯৭০ সালে **Edgar F. Codd** তাঁর বিখ্যাত গবেষণাপত্র *"A Relational Model of Data for Large Shared Data Banks"*-এ relational model-এর ভিত্তি স্থাপন করেন। পরবর্তীতে তিনি **normalization**-এর ধারণা প্রবর্তন করেন যার মূল উদ্দেশ্য ছিল:

- **Data redundancy** (ডেটার পুনরাবৃত্তি) দূর করা
- **Data integrity** (ডেটার সঠিকতা) নিশ্চিত করা
- **Anomaly** (অস্বাভাবিকতা) প্রতিরোধ করা

Normalization হলো একটি **systematic process** যেখানে একটি relational database schema-কে ধাপে ধাপে এমনভাবে পুনর্গঠন করা হয় যাতে **data redundancy** কমে এবং **data dependency** যৌক্তিক থাকে।

### কেন Normalize করবেন?

একটি unnormalized database-এ তিন ধরনের মারাত্মক সমস্যা দেখা দেয়:

| সমস্যা | বিবরণ |
|---------|--------|
| **Update Anomaly** | একই ডেটা একাধিক জায়গায় থাকলে একটি আপডেট করলে বাকিগুলো inconsistent থেকে যায় |
| **Insert Anomaly** | নির্দিষ্ট ডেটা ছাড়া নতুন ডেটা insert করা যায় না |
| **Delete Anomaly** | একটি ডেটা মুছতে গেলে অপ্রাসঙ্গিক ডেটাও হারিয়ে যায় |

### Functional Dependency (ফাংশনাল নির্ভরশীলতা)

Functional Dependency হলো normalization-এর মূল গাণিতিক ভিত্তি। যদি attribute **A**-এর মান জানলে attribute **B**-এর মান **অনন্যভাবে** নির্ধারণ করা যায়, তাহলে বলা হয় **B** functionally dependent on **A**।

```
সংকেত: A → B  (A determines B)

উদাহরণ:
order_id → order_date         (order_id জানলে order_date জানা যায়)
product_id → product_name     (product_id জানলে product_name জানা যায়)
{order_id, product_id} → quantity  (composite key)
```

**Functional Dependency-র প্রকারভেদ:**

```
১. Full Functional Dependency:
   {order_id, product_id} → quantity
   (সম্পূর্ণ composite key-ই প্রয়োজন)

২. Partial Functional Dependency:
   {order_id, product_id} → order_date
   (শুধু order_id-ই যথেষ্ট, product_id প্রয়োজন নেই)

৩. Transitive Dependency:
   order_id → customer_id → customer_name
   (order_id সরাসরি customer_name নির্ধারণ করে না, customer_id-এর মাধ্যমে করে)
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### লাইব্রেরি কার্ড ক্যাটালগ Analogy

কল্পনা করুন একটি পুরানো লাইব্রেরি যেখানে সবকিছু **একটি বড় খাতায়** লেখা:

```
| বই নং | বইয়ের নাম        | লেখক           | লেখকের ঠিকানা    | ক্যাটাগরি        | সদস্য নাম  | ধার তারিখ |
|--------|-------------------|----------------|-------------------|-------------------|-------------|-----------|
| ১      | গীতাঞ্জলি          | রবীন্দ্রনাথ ঠাকুর | জোড়াসাঁকো, কলকাতা | কবিতা             | করিম সাহেব  | ০১-০১-২৪  |
| ২      | গোরা              | রবীন্দ্রনাথ ঠাকুর | জোড়াসাঁকো, কলকাতা | উপন্যাস           | করিম সাহেব  | ০১-০১-২৪  |
| ৩      | পথের পাঁচালী       | বিভূতিভূষণ       | ঘোষপাড়া, কলকাতা   | উপন্যাস           | রহিম সাহেব  | ০৫-০১-২৪  |
```

**সমস্যাগুলো:**
- রবীন্দ্রনাথের ঠিকানা **দুইবার** লেখা (redundancy)
- করিম সাহেবের তথ্য **দুইবার** লেখা (redundancy)
- রবীন্দ্রনাথের ঠিকানা বদলালে **সব জায়গায়** বদলাতে হবে (update anomaly)

**Normalized সমাধান — আলাদা আলাদা রেজিস্টার:**

```
📚 বই রেজিস্টার:     বই নং → বইয়ের নাম, লেখক_আইডি, ক্যাটাগরি_আইডি
✍️  লেখক রেজিস্টার:   লেখক_আইডি → লেখকের নাম, ঠিকানা
📂 ক্যাটাগরি রেজিস্টার: ক্যাটাগরি_আইডি → ক্যাটাগরি নাম
👤 সদস্য রেজিস্টার:   সদস্য_আইডি → সদস্য নাম
📋 ধার রেজিস্টার:     ধার_আইডি → বই নং, সদস্য_আইডি, তারিখ
```

এখন প্রতিটি তথ্য **একবারই** সংরক্ষিত, এবং **reference** (আইডি) দিয়ে সম্পর্ক স্থাপিত।

---

## 📊 Anomaly ডায়াগ্রাম

### বাংলাদেশী ই-কমার্স অর্ডার সিস্টেমের Unnormalized টেবিল

ধরুন **Daraz Bangladesh**-এর মতো একটি সিস্টেমের সব ডেটা একটি টেবিলে রাখা হলো:

```sql
-- ❌ সমস্যাযুক্ত Unnormalized টেবিল
CREATE TABLE orders_denormalized (
    order_id INT,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_division VARCHAR(50),    -- ঢাকা, চট্টগ্রাম, রাজশাহী...
    customer_district VARCHAR(50),
    product_name VARCHAR(200),
    product_category VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT,
    payment_method VARCHAR(50),       -- bKash, Nagad, COD
    payment_trx_id VARCHAR(100),
    delivery_status VARCHAR(50)
);
```

### ১. Update Anomaly

```
| order_id | customer_name | customer_phone | product_name   | product_price |
|----------|---------------|----------------|----------------|---------------|
| 1001     | রহিম          | 01712345678    | Samsung Galaxy  | 25000         |
| 1002     | রহিম          | 01712345678    | Xiaomi Redmi    | 15000         |
| 1003     | রহিম          | 01799999999    | iPhone Case     | 500           | ← ফোন নম্বর বদলানো হলো
```

**সমস্যা:** রহিমের ফোন নম্বর পরিবর্তন করতে হলে **সব row** আপডেট করতে হবে। তৃতীয় row-তে ভিন্ন নম্বর থাকায় কোনটি সঠিক বোঝা যাচ্ছে না — **data inconsistency**।

### ২. Insert Anomaly

```
| order_id | customer_name | customer_phone | product_name | product_price |
|----------|---------------|----------------|--------------|---------------|
| ?        | ?             | ?              | Walton TV    | 35000         | ← অর্ডার ছাড়া প্রোডাক্ট যোগ করা যাচ্ছে না!
```

**সমস্যা:** নতুন প্রোডাক্ট যোগ করতে হলে একটি অর্ডারও থাকতে হবে। কারণ `order_id` ছাড়া row insert করার উপায় নেই।

### ৩. Delete Anomaly

```
| order_id | customer_name | product_name | product_category |
|----------|---------------|--------------|------------------|
| 1005     | করিম          | Apex Shoe    | Footwear         | ← এটিই Footwear ক্যাটাগরির একমাত্র অর্ডার
```

**সমস্যা:** এই অর্ডার মুছে ফেললে **Footwear ক্যাটাগরির তথ্যও** হারিয়ে যাবে, এবং **Apex Shoe** প্রোডাক্টের অস্তিত্বও থাকবে না।

---

## 💻 Normal Forms Deep Dive

### একটি সম্পূর্ণ Progressive উদাহরণ: বাংলাদেশী ই-কমার্স অর্ডার সিস্টেম

আমরা একটি বাংলাদেশী ই-কমার্স সিস্টেম (যেমন Daraz/Chaldal) ব্যবহার করে ধাপে ধাপে normalize করব।

---

### UNF (Unnormalized Form)

```sql
-- সব ডেটা একটি টেবিলে — বাস্তবে এরকম দেখা যায় legacy সিস্টেমে
INSERT INTO orders_raw VALUES
(1001, '2024-01-15', 'রহিম উদ্দিন', '01712345678', 'ঢাকা', 'মিরপুর, সেকশন ১২, রোড ৫, বাসা ১০',
 'Samsung Galaxy S24, Spigen Case', 'Electronics, Accessories', '95000, 1500', '1, 2',
 'bKash', 'TRX123ABC', 'আমিনুল হক', '01898765432', 'ডেলিভারড');
```

**সমস্যাসমূহ:**
- `product_name` কলামে **একাধিক মান** comma-separated (repeating group)
- `category`, `price`, `quantity` — সবই **multi-valued**
- কোনো Primary Key সংজ্ঞায়িত নেই
- একটি row মুছলে সব তথ্য হারিয়ে যায়

```
সমস্যার চিত্র:
┌──────────────────────────────────────────────────┐
│              orders_raw (UNF)                     │
├──────────────────────────────────────────────────┤
│ order_id: 1001                                   │
│ products: "Samsung Galaxy S24, Spigen Case"  ❌  │
│ prices: "95000, 1500"                        ❌  │
│ quantities: "1, 2"                           ❌  │
│ categories: "Electronics, Accessories"       ❌  │
│ customer + address + payment — সব এক জায়গায়  ❌  │
└──────────────────────────────────────────────────┘
```

---

### 1NF (First Normal Form)

**নিয়ম:**
1. প্রতিটি কলামে **atomic value** (একক মান) থাকবে
2. কোনো **repeating group** থাকবে না
3. প্রতিটি row **uniquely identifiable** হবে (Primary Key)
4. প্রতিটি কলামে **একই ধরনের** ডেটা থাকবে

```sql
-- ✅ 1NF: Atomic values, composite Primary Key
CREATE TABLE orders_1nf (
    order_id INT NOT NULL,
    order_date DATE NOT NULL,
    customer_name VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_division VARCHAR(50),
    customer_address TEXT,
    product_name VARCHAR(200),
    product_category VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT,
    payment_method VARCHAR(50),
    payment_trx_id VARCHAR(100),
    delivery_person VARCHAR(100),
    delivery_phone VARCHAR(20),
    delivery_status VARCHAR(50),
    PRIMARY KEY (order_id, product_name)
);

INSERT INTO orders_1nf VALUES
(1001, '2024-01-15', 'রহিম উদ্দিন', '01712345678', 'ঢাকা',
 'মিরপুর, সেকশন ১২, রোড ৫, বাসা ১০',
 'Samsung Galaxy S24', 'Electronics', 95000.00, 1,
 'bKash', 'TRX123ABC', 'আমিনুল হক', '01898765432', 'ডেলিভারড'),
(1001, '2024-01-15', 'রহিম উদ্দিন', '01712345678', 'ঢাকা',
 'মিরপুর, সেকশন ১২, রোড ৫, বাসা ১০',
 'Spigen Case', 'Accessories', 1500.00, 2,
 'bKash', 'TRX123ABC', 'আমিনুল হক', '01898765432', 'ডেলিভারড'),
(1002, '2024-01-16', 'করিম মিয়া', '01819876543', 'চট্টগ্রাম',
 'আগ্রাবাদ, ব্লক বি, ফ্ল্যাট ৪০২',
 'Samsung Galaxy S24', 'Electronics', 95000.00, 1,
 'Nagad', 'NGD456DEF', 'সুমন আহমেদ', '01756781234', 'শিপড');
```

**1NF-এর পরেও যে সমস্যা রয়ে গেছে:**

```
Partial Dependencies (আংশিক নির্ভরশীলতা):
PK = {order_id, product_name}

order_id → order_date, customer_name, customer_phone, ...  (শুধু order_id-ই যথেষ্ট)
product_name → product_category, product_price              (শুধু product_name-ই যথেষ্ট)

এগুলো সম্পূর্ণ PK-এর উপর নির্ভরশীল নয় → Partial Dependency ❌
```

---

### 2NF (Second Normal Form)

**নিয়ম:**
1. অবশ্যই **1NF** হতে হবে
2. কোনো **partial dependency** থাকবে না — non-key attribute সম্পূর্ণ Primary Key-এর উপর নির্ভরশীল হবে

**Functional Dependency বিশ্লেষণ:**

```
order_id → order_date, customer_name, customer_phone, customer_division,
           customer_address, payment_method, payment_trx_id,
           delivery_person, delivery_phone, delivery_status

product_name → product_category, product_price

{order_id, product_name} → quantity   ← এটিই Full Dependency
```

```sql
-- ✅ 2NF: Partial dependencies সরানো হলো

-- অর্ডার টেবিল (order-specific ডেটা)
CREATE TABLE orders_2nf (
    order_id INT PRIMARY KEY,
    order_date DATE NOT NULL,
    customer_name VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_division VARCHAR(50),
    customer_address TEXT,
    payment_method VARCHAR(50),
    payment_trx_id VARCHAR(100),
    delivery_person VARCHAR(100),
    delivery_phone VARCHAR(20),
    delivery_status VARCHAR(50)
);

-- প্রোডাক্ট টেবিল (product-specific ডেটা)
CREATE TABLE products_2nf (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(200) UNIQUE NOT NULL,
    product_category VARCHAR(100),
    product_price DECIMAL(10,2)
);

-- অর্ডার-আইটেম টেবিল (সম্পর্ক ও quantity)
CREATE TABLE order_items_2nf (
    order_id INT,
    product_id INT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,  -- অর্ডারের সময়ের মূল্য সংরক্ষণ
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders_2nf(order_id),
    FOREIGN KEY (product_id) REFERENCES products_2nf(product_id)
);
```

**2NF-এর পরেও যে সমস্যা রয়ে গেছে:**

```
Transitive Dependencies (পরোক্ষ নির্ভরশীলতা):
orders_2nf টেবিলে:

order_id → customer_phone → customer_name, customer_division, customer_address
order_id → delivery_person → delivery_phone

customer_name, customer_division সরাসরি order_id-এর উপর নির্ভরশীল নয়,
বরং customer_phone-এর মাধ্যমে → Transitive Dependency ❌
```

---

### 3NF (Third Normal Form)

**নিয়ম:**
1. অবশ্যই **2NF** হতে হবে
2. কোনো **transitive dependency** থাকবে না — non-key attribute কেবল Primary Key-এর উপর সরাসরি নির্ভরশীল হবে, অন্য কোনো non-key attribute-এর মাধ্যমে নয়

```sql
-- ✅ 3NF: Transitive dependencies সরানো হলো

-- কাস্টমার টেবিল
CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_name VARCHAR(100) NOT NULL,
    customer_phone VARCHAR(20) UNIQUE NOT NULL,
    customer_email VARCHAR(150),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ঠিকানা টেবিল (একজন কাস্টমারের একাধিক ঠিকানা থাকতে পারে)
CREATE TABLE addresses (
    address_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    division VARCHAR(50) NOT NULL,       -- বিভাগ: ঢাকা, চট্টগ্রাম, রাজশাহী...
    district VARCHAR(50),                -- জেলা
    upazila VARCHAR(50),                 -- উপজেলা
    address_line TEXT NOT NULL,           -- বিস্তারিত ঠিকানা
    postal_code VARCHAR(10),
    address_type ENUM('home', 'office', 'other') DEFAULT 'home',
    is_default BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- ডেলিভারি পার্সন টেবিল
CREATE TABLE delivery_persons (
    delivery_person_id INT PRIMARY KEY AUTO_INCREMENT,
    person_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    zone VARCHAR(100)                    -- ডেলিভারি এরিয়া
);

-- পেমেন্ট টেবিল
CREATE TABLE payments (
    payment_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    method ENUM('bKash', 'Nagad', 'Rocket', 'COD', 'Card') NOT NULL,
    transaction_id VARCHAR(100),
    amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    paid_at TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- ক্যাটাগরি টেবিল
CREATE TABLE categories (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    category_name VARCHAR(100) NOT NULL,
    parent_category_id INT,
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- প্রোডাক্ট টেবিল (উন্নত)
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(200) NOT NULL,
    category_id INT,
    base_price DECIMAL(10,2) NOT NULL,
    sku VARCHAR(50) UNIQUE,
    stock_quantity INT DEFAULT 0,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- অর্ডার টেবিল (3NF)
CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    shipping_address_id INT NOT NULL,
    delivery_person_id INT,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delivery_status ENUM('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')
        DEFAULT 'pending',
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (delivery_person_id) REFERENCES delivery_persons(delivery_person_id)
);

-- অর্ডার আইটেম টেবিল
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,   -- অর্ডারের সময়ের মূল্য (historical price)
    discount DECIMAL(10,2) DEFAULT 0,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**3NF-এর সুবিধা:**
- প্রতিটি তথ্য **একবারই** সংরক্ষিত
- আপডেট করতে হলে **একটি জায়গায়** করলেই যথেষ্ট
- ডেটা **consistent** থাকে সবসময়

---

### BCNF (Boyce-Codd Normal Form)

**নিয়ম:**
1. অবশ্যই **3NF** হতে হবে
2. প্রতিটি **determinant** অবশ্যই একটি **candidate key** হবে

3NF ও BCNF-এর মধ্যে পার্থক্য তখনই দেখা যায় যখন:
- একটি টেবিলে **একাধিক candidate key** থাকে
- candidate key গুলো **composite** হয়
- candidate key গুলোর মধ্যে **overlapping** attribute থাকে

**উদাহরণ: কোর্স-শিক্ষক সমস্যা**

ধরুন একটি ই-কমার্স প্ল্যাটফর্মের বিক্রেতা প্রশিক্ষণ সিস্টেম:

```sql
-- ❌ 3NF কিন্তু BCNF নয়
-- নিয়ম: প্রতিটি বিষয় শুধু একজন প্রশিক্ষক পড়ান
-- একজন প্রশিক্ষক শুধু একটি বিষয় পড়ান
-- একজন বিক্রেতা একটি বিষয়ে একজন প্রশিক্ষকের কাছেই পড়ে

CREATE TABLE seller_training (
    seller_id INT,
    subject VARCHAR(100),
    trainer VARCHAR(100),
    PRIMARY KEY (seller_id, subject)
);

-- Candidate Keys: {seller_id, subject}, {seller_id, trainer}
-- Functional Dependency: trainer → subject
-- কিন্তু trainer একটি candidate key নয়!
-- তাই এটি BCNF ভঙ্গ করছে
```

```sql
-- ✅ BCNF-এ রূপান্তর
CREATE TABLE trainers (
    trainer_id INT PRIMARY KEY AUTO_INCREMENT,
    trainer_name VARCHAR(100) NOT NULL,
    subject VARCHAR(100) NOT NULL UNIQUE   -- একজন প্রশিক্ষক = একটি বিষয়
);

CREATE TABLE seller_enrollments (
    seller_id INT,
    trainer_id INT,
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (seller_id, trainer_id),
    FOREIGN KEY (trainer_id) REFERENCES trainers(trainer_id)
);
-- এখন trainer_id → subject, এবং trainer_id হলো trainers টেবিলের Primary Key (candidate key)
```

**BCNF-এর বাস্তব প্রয়োগ:**

বেশিরভাগ ক্ষেত্রে **3NF = BCNF**। পার্থক্য তখনই আসে যখন complex composite keys এবং overlapping candidate keys থাকে। Production সিস্টেমে এই পরিস্থিতি বিরল কিন্তু database design-এর তাত্ত্বিক সম্পূর্ণতার জন্য গুরুত্বপূর্ণ।

---

### 4NF (Fourth Normal Form)

**নিয়ম:**
1. অবশ্যই **BCNF** হতে হবে
2. কোনো **multi-valued dependency** (MVD) থাকবে না (trivial MVD ব্যতীত)

**Multi-Valued Dependency:** যখন একটি attribute-এর মান **স্বাধীনভাবে** একাধিক attribute-এর মান নির্ধারণ করে।

```
সংকেত: A →→ B (A multi-determines B)
```

**উদাহরণ: ই-কমার্স বিক্রেতার একাধিক স্বাধীন বৈশিষ্ট্য**

```sql
-- ❌ 4NF ভঙ্গ — একটি seller-এর delivery_zone ও accepted_payment স্বাধীন
CREATE TABLE seller_capabilities (
    seller_id INT,
    delivery_zone VARCHAR(100),      -- ঢাকা, চট্টগ্রাম, সিলেট...
    accepted_payment VARCHAR(50),    -- bKash, Nagad, COD...
    PRIMARY KEY (seller_id, delivery_zone, accepted_payment)
);

-- seller_id →→ delivery_zone (delivery_zone, accepted_payment থেকে স্বাধীন)
-- seller_id →→ accepted_payment (accepted_payment, delivery_zone থেকে স্বাধীন)

INSERT INTO seller_capabilities VALUES
(1, 'ঢাকা', 'bKash'),
(1, 'ঢাকা', 'Nagad'),
(1, 'ঢাকা', 'COD'),
(1, 'চট্টগ্রাম', 'bKash'),
(1, 'চট্টগ্রাম', 'Nagad'),
(1, 'চট্টগ্রাম', 'COD');
-- ৩টি zone × ৩টি payment = ৯টি row লাগবে! Cartesian product!
```

```sql
-- ✅ 4NF-এ রূপান্তর — স্বাধীন তথ্যগুলো আলাদা টেবিলে

CREATE TABLE seller_delivery_zones (
    seller_id INT,
    delivery_zone VARCHAR(100),
    PRIMARY KEY (seller_id, delivery_zone)
);

CREATE TABLE seller_payment_methods (
    seller_id INT,
    accepted_payment VARCHAR(50),
    PRIMARY KEY (seller_id, accepted_payment)
);

-- এখন ৩টি zone + ৩টি payment = ৬টি row (৯টির বদলে)
-- এবং স্বাধীনভাবে আপডেট করা যায়
```

---

### 5NF (Fifth Normal Form / Project-Join Normal Form)

**নিয়ম:**
1. অবশ্যই **4NF** হতে হবে
2. কোনো **join dependency** থাকবে না যা candidate key দ্বারা implied নয়

5NF অত্যন্ত বিরল এবং তাত্ত্বিক। এটি তখন প্রয়োজন হয় যখন একটি টেবিলকে **তিনটি বা তার বেশি** টেবিলে decompose করা যায় এবং সেগুলো join করলে **spurious data** ছাড়াই মূল টেবিল ফিরে আসে।

**উদাহরণ: সরবরাহকারী-পণ্য-প্রকল্প**

```sql
-- ধরুন নিয়ম: একটি সরবরাহকারী (supplier) একটি প্রোডাক্ট একটি
-- warehouse-এ সরবরাহ করে, কিন্তু সব combination বৈধ নয়

-- ❌ 4NF কিন্তু 5NF নয়
CREATE TABLE supply_info (
    supplier_id INT,
    product_id INT,
    warehouse_id INT,
    PRIMARY KEY (supplier_id, product_id, warehouse_id)
);

-- ✅ 5NF-এ রূপান্তর (যখন ternary relationship decomposable)
CREATE TABLE supplier_products (
    supplier_id INT,
    product_id INT,
    PRIMARY KEY (supplier_id, product_id)
);

CREATE TABLE supplier_warehouses (
    supplier_id INT,
    warehouse_id INT,
    PRIMARY KEY (supplier_id, warehouse_id)
);

CREATE TABLE product_warehouses (
    product_id INT,
    warehouse_id INT,
    PRIMARY KEY (product_id, warehouse_id)
);

-- মূল ডেটা পুনরুদ্ধার:
SELECT sp.supplier_id, sp.product_id, sw.warehouse_id
FROM supplier_products sp
JOIN supplier_warehouses sw ON sp.supplier_id = sw.supplier_id
JOIN product_warehouses pw ON sp.product_id = pw.product_id
    AND sw.warehouse_id = pw.warehouse_id;
```

**⚠️ সতর্কতা:** 5NF-এ decompose করা **সবসময়** সঠিক নয়। শুধুমাত্র যখন business rule গ্যারান্টি দেয় যে binary relationships থেকে ternary relationship পুনর্গঠন করা যায়, তখনই 5NF প্রযোজ্য। অন্যথায় **spurious tuples** তৈরি হবে।

---

## 🔥 Advanced Scenarios

### ১. Denormalization (বিপরীত প্রক্রিয়া)

Denormalization হলো **ইচ্ছাকৃতভাবে** redundancy যোগ করে read performance বাড়ানো। এটি normalize করার পর **সচেতন সিদ্ধান্ত** হিসেবে করা হয়, শুরুতেই unnormalized রাখা নয়।

#### কখন Denormalize করবেন?

```
✅ করবেন:
- Read-heavy সিস্টেম (read:write ratio > 10:1)
- Complex JOIN queries যেগুলো ঘন ঘন চলে
- Reporting/Analytics ড্যাশবোর্ড
- Caching layer হিসেবে

❌ করবেন না:
- Write-heavy সিস্টেম
- ডেটা consistency সর্বোচ্চ গুরুত্বপূর্ণ হলে (যেমন: ব্যাংকিং)
- Schema design শুরুতেই (আগে normalize করুন)
```

#### Materialized View Pattern

```sql
-- PostgreSQL Materialized View — ই-কমার্স ড্যাশবোর্ডের জন্য
CREATE MATERIALIZED VIEW mv_seller_dashboard AS
SELECT
    s.seller_id,
    s.shop_name,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(r.rating) AS avg_rating,
    COUNT(DISTINCT r.review_id) AS total_reviews
FROM sellers s
LEFT JOIN products p ON s.seller_id = p.seller_id
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY s.seller_id, s.shop_name
WITH DATA;

-- পর্যায়ক্রমে refresh (cron job বা pg_cron দিয়ে)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_seller_dashboard;

CREATE UNIQUE INDEX idx_mv_seller ON mv_seller_dashboard(seller_id);
```

#### JSON Column-এ Denormalized Data

```sql
-- MySQL 8.0+ / PostgreSQL: অর্ডারের snapshot সংরক্ষণ
ALTER TABLE orders ADD COLUMN order_snapshot JSON;

-- অর্ডার তৈরির সময় snapshot সেভ করা
UPDATE orders SET order_snapshot = JSON_OBJECT(
    'customer_name', (SELECT customer_name FROM customers WHERE customer_id = orders.customer_id),
    'items', (
        SELECT JSON_ARRAYAGG(JSON_OBJECT(
            'product_name', p.product_name,
            'quantity', oi.quantity,
            'unit_price', oi.unit_price
        ))
        FROM order_items oi
        JOIN products p ON oi.product_id = p.product_id
        WHERE oi.order_id = orders.order_id
    ),
    'shipping_address', (
        SELECT JSON_OBJECT('division', division, 'district', district, 'address', address_line)
        FROM addresses WHERE address_id = orders.shipping_address_id
    )
) WHERE order_id = 1001;
```

#### PHP Laravel: Denormalized Columns with Accessors

```php
// app/Models/Order.php
class Order extends Model
{
    protected $casts = [
        'order_snapshot' => 'array',
        'cached_total' => 'decimal:2',
    ];

    // Denormalized total — recalculate on item changes
    public function recalculateTotal(): void
    {
        $this->cached_total = $this->items()
            ->selectRaw('SUM(quantity * unit_price - discount) as total')
            ->value('total');
        $this->save();
    }

    // Observer-এ স্বয়ংক্রিয়ভাবে snapshot সেভ
    protected static function booted()
    {
        static::creating(function (Order $order) {
            $order->order_snapshot = $order->buildSnapshot();
        });
    }

    private function buildSnapshot(): array
    {
        return [
            'customer_name' => $this->customer->customer_name,
            'items' => $this->items->map(fn ($item) => [
                'product_name' => $item->product->product_name,
                'quantity' => $item->quantity,
                'unit_price' => $item->unit_price,
            ])->toArray(),
            'shipping_address' => [
                'division' => $this->shippingAddress->division,
                'address' => $this->shippingAddress->address_line,
            ],
        ];
    }

    // Accessor — snapshot থেকে সরাসরি পড়া (JOIN ছাড়া)
    public function getCustomerNameCachedAttribute(): string
    {
        return $this->order_snapshot['customer_name'] ?? $this->customer->customer_name;
    }
}
```

#### JavaScript: Sequelize Virtual Fields

```javascript
// models/Order.js
const { Model, DataTypes } = require('sequelize');

class Order extends Model {
  static init(sequelize) {
    return super.init({
      orderId: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
      customerId: { type: DataTypes.INTEGER, allowNull: false },
      orderDate: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },
      cachedTotal: { type: DataTypes.DECIMAL(10, 2), defaultValue: 0 },
      orderSnapshot: { type: DataTypes.JSON },

      // Virtual field — DB-তে সংরক্ষিত নয়, কিন্তু computed
      formattedTotal: {
        type: DataTypes.VIRTUAL,
        get() {
          const total = parseFloat(this.cachedTotal || 0);
          return `৳${total.toLocaleString('bn-BD')}`;
        },
      },
      customerNameCached: {
        type: DataTypes.VIRTUAL,
        get() {
          return this.orderSnapshot?.customer_name || null;
        },
      },
    }, { sequelize, modelName: 'Order', tableName: 'orders' });
  }

  // Denormalized total পুনর্গণনা
  async recalculateTotal() {
    const result = await OrderItem.findOne({
      where: { orderId: this.orderId },
      attributes: [
        [sequelize.fn('SUM',
          sequelize.literal('quantity * unit_price - discount')
        ), 'total'],
      ],
      raw: true,
    });
    this.cachedTotal = result?.total || 0;
    await this.save();
  }

  // Snapshot তৈরি
  async buildSnapshot() {
    const customer = await this.getCustomer();
    const items = await this.getOrderItems({ include: ['Product'] });
    const address = await this.getShippingAddress();

    this.orderSnapshot = {
      customer_name: customer.customerName,
      items: items.map(item => ({
        product_name: item.Product.productName,
        quantity: item.quantity,
        unit_price: parseFloat(item.unitPrice),
      })),
      shipping_address: {
        division: address.division,
        address: address.addressLine,
      },
    };
    await this.save();
  }
}
```

---

### ২. Normalization in E-commerce

#### সম্পূর্ণ ER Diagram (ASCII)

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│  customers   │     │    addresses     │     │  categories  │
├──────────────┤     ├─────────────────┤     ├──────────────┤
│ *customer_id │──┐  │ *address_id     │     │ *category_id │──┐
│  name        │  │  │  customer_id FK │     │  name        │  │
│  phone       │  └─>│  division       │     │  parent_id FK│──┘ (self-ref)
│  email       │     │  district       │     └──────────────┘
└──────────────┘     │  upazila        │            │
       │             │  address_line   │            │
       │             │  postal_code    │     ┌──────────────┐
       │             │  type           │     │   products   │
       │             └─────────────────┘     ├──────────────┤
       │                    │                │ *product_id  │
       │                    │                │  name        │
       v                    v                │  category_id │──>categories
┌──────────────┐     ┌─────────────────┐     │  base_price  │
│   orders     │     │  order_items    │     │  sku         │
├──────────────┤     ├─────────────────┤     │  stock_qty   │
│ *order_id    │──┐  │ *order_id FK   │     │  seller_id   │──>sellers
│  customer_id │  └─>│ *product_id FK │<────│              │
│  address_id  │──>  │  quantity       │     └──────────────┘
│  delivery_id │     │  unit_price     │
│  order_date  │     │  discount       │     ┌──────────────┐
│  status      │     └─────────────────┘     │   sellers    │
│  total       │                             ├──────────────┤
│  snapshot    │     ┌─────────────────┐     │ *seller_id   │
└──────────────┘     │    payments     │     │  shop_name   │
       │             ├─────────────────┤     │  phone       │
       └────────────>│ *payment_id    │     │  nid_number  │
                     │  order_id FK    │     └──────────────┘
                     │  method         │
                     │  trx_id         │     ┌──────────────────┐
                     │  amount         │     │ delivery_persons │
                     │  status         │     ├──────────────────┤
                     └─────────────────┘     │ *delivery_id     │
                                             │  name            │
                                             │  phone           │
                                             │  zone            │
                                             └──────────────────┘
```

#### Laravel Migration Code

```php
// database/migrations/2024_01_01_000001_create_ecommerce_schema.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('customers', function (Blueprint $table) {
            $table->id('customer_id');
            $table->string('customer_name', 100);
            $table->string('customer_phone', 20)->unique();
            $table->string('customer_email', 150)->nullable()->unique();
            $table->timestamps();
            $table->index('customer_phone');
        });

        Schema::create('addresses', function (Blueprint $table) {
            $table->id('address_id');
            $table->foreignId('customer_id')->constrained('customers', 'customer_id')->cascadeOnDelete();
            $table->string('division', 50);          // ঢাকা, চট্টগ্রাম, রাজশাহী, খুলনা, বরিশাল, সিলেট, রংপুর, ময়মনসিংহ
            $table->string('district', 50)->nullable();
            $table->string('upazila', 50)->nullable();
            $table->text('address_line');
            $table->string('postal_code', 10)->nullable();
            $table->enum('address_type', ['home', 'office', 'other'])->default('home');
            $table->boolean('is_default')->default(false);
            $table->timestamps();
            $table->index(['customer_id', 'is_default']);
        });

        Schema::create('categories', function (Blueprint $table) {
            $table->id('category_id');
            $table->string('category_name', 100);
            $table->foreignId('parent_category_id')->nullable()
                  ->constrained('categories', 'category_id')->nullOnDelete();
            $table->timestamps();
        });

        Schema::create('sellers', function (Blueprint $table) {
            $table->id('seller_id');
            $table->string('shop_name', 200);
            $table->string('phone', 20)->unique();
            $table->string('nid_number', 20)->unique();  // জাতীয় পরিচয়পত্র
            $table->enum('status', ['active', 'inactive', 'suspended'])->default('active');
            $table->timestamps();
        });

        Schema::create('products', function (Blueprint $table) {
            $table->id('product_id');
            $table->string('product_name', 200);
            $table->foreignId('category_id')->nullable()
                  ->constrained('categories', 'category_id')->nullOnDelete();
            $table->foreignId('seller_id')
                  ->constrained('sellers', 'seller_id')->cascadeOnDelete();
            $table->decimal('base_price', 10, 2);
            $table->string('sku', 50)->unique();
            $table->integer('stock_quantity')->default(0);
            $table->timestamps();
            $table->index(['category_id', 'seller_id']);
            $table->index('sku');
        });

        Schema::create('delivery_persons', function (Blueprint $table) {
            $table->id('delivery_person_id');
            $table->string('person_name', 100);
            $table->string('phone', 20)->unique();
            $table->string('zone', 100)->nullable();
            $table->timestamps();
        });

        Schema::create('orders', function (Blueprint $table) {
            $table->id('order_id');
            $table->foreignId('customer_id')->constrained('customers', 'customer_id');
            $table->foreignId('shipping_address_id')->constrained('addresses', 'address_id');
            $table->foreignId('delivery_person_id')->nullable()
                  ->constrained('delivery_persons', 'delivery_person_id')->nullOnDelete();
            $table->enum('delivery_status', ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'])
                  ->default('pending');
            $table->decimal('cached_total', 10, 2)->default(0);
            $table->json('order_snapshot')->nullable();
            $table->timestamps();
            $table->index(['customer_id', 'delivery_status']);
            $table->index('created_at');
        });

        Schema::create('order_items', function (Blueprint $table) {
            $table->foreignId('order_id')->constrained('orders', 'order_id')->cascadeOnDelete();
            $table->foreignId('product_id')->constrained('products', 'product_id');
            $table->integer('quantity');
            $table->decimal('unit_price', 10, 2);
            $table->decimal('discount', 10, 2)->default(0);
            $table->primary(['order_id', 'product_id']);
        });

        Schema::create('payments', function (Blueprint $table) {
            $table->id('payment_id');
            $table->foreignId('order_id')->constrained('orders', 'order_id');
            $table->enum('method', ['bKash', 'Nagad', 'Rocket', 'COD', 'Card']);
            $table->string('transaction_id', 100)->nullable();
            $table->decimal('amount', 10, 2);
            $table->enum('status', ['pending', 'completed', 'failed', 'refunded'])->default('pending');
            $table->timestamp('paid_at')->nullable();
            $table->timestamps();
            $table->index(['order_id', 'status']);
            $table->index('transaction_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('payments');
        Schema::dropIfExists('order_items');
        Schema::dropIfExists('orders');
        Schema::dropIfExists('delivery_persons');
        Schema::dropIfExists('products');
        Schema::dropIfExists('sellers');
        Schema::dropIfExists('categories');
        Schema::dropIfExists('addresses');
        Schema::dropIfExists('customers');
    }
};
```

#### Sequelize/Knex Migration Code

```javascript
// migrations/20240101_create_ecommerce_schema.js (Knex)
exports.up = function(knex) {
  return knex.schema
    .createTable('customers', (table) => {
      table.increments('customer_id').primary();
      table.string('customer_name', 100).notNullable();
      table.string('customer_phone', 20).notNullable().unique();
      table.string('customer_email', 150).nullable().unique();
      table.timestamps(true, true);
      table.index('customer_phone');
    })
    .createTable('addresses', (table) => {
      table.increments('address_id').primary();
      table.integer('customer_id').unsigned().notNullable()
        .references('customer_id').inTable('customers').onDelete('CASCADE');
      table.string('division', 50).notNullable();
      table.string('district', 50).nullable();
      table.string('upazila', 50).nullable();
      table.text('address_line').notNullable();
      table.string('postal_code', 10).nullable();
      table.enu('address_type', ['home', 'office', 'other']).defaultTo('home');
      table.boolean('is_default').defaultTo(false);
      table.timestamps(true, true);
      table.index(['customer_id', 'is_default']);
    })
    .createTable('categories', (table) => {
      table.increments('category_id').primary();
      table.string('category_name', 100).notNullable();
      table.integer('parent_category_id').unsigned().nullable()
        .references('category_id').inTable('categories').onDelete('SET NULL');
      table.timestamps(true, true);
    })
    .createTable('sellers', (table) => {
      table.increments('seller_id').primary();
      table.string('shop_name', 200).notNullable();
      table.string('phone', 20).notNullable().unique();
      table.string('nid_number', 20).notNullable().unique();
      table.enu('status', ['active', 'inactive', 'suspended']).defaultTo('active');
      table.timestamps(true, true);
    })
    .createTable('products', (table) => {
      table.increments('product_id').primary();
      table.string('product_name', 200).notNullable();
      table.integer('category_id').unsigned().nullable()
        .references('category_id').inTable('categories').onDelete('SET NULL');
      table.integer('seller_id').unsigned().notNullable()
        .references('seller_id').inTable('sellers').onDelete('CASCADE');
      table.decimal('base_price', 10, 2).notNullable();
      table.string('sku', 50).notNullable().unique();
      table.integer('stock_quantity').defaultTo(0);
      table.timestamps(true, true);
      table.index(['category_id', 'seller_id']);
    })
    .createTable('delivery_persons', (table) => {
      table.increments('delivery_person_id').primary();
      table.string('person_name', 100).notNullable();
      table.string('phone', 20).notNullable().unique();
      table.string('zone', 100).nullable();
      table.timestamps(true, true);
    })
    .createTable('orders', (table) => {
      table.increments('order_id').primary();
      table.integer('customer_id').unsigned().notNullable()
        .references('customer_id').inTable('customers');
      table.integer('shipping_address_id').unsigned().notNullable()
        .references('address_id').inTable('addresses');
      table.integer('delivery_person_id').unsigned().nullable()
        .references('delivery_person_id').inTable('delivery_persons').onDelete('SET NULL');
      table.enu('delivery_status', ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'])
        .defaultTo('pending');
      table.decimal('cached_total', 10, 2).defaultTo(0);
      table.json('order_snapshot').nullable();
      table.timestamps(true, true);
      table.index(['customer_id', 'delivery_status']);
    })
    .createTable('order_items', (table) => {
      table.integer('order_id').unsigned().notNullable()
        .references('order_id').inTable('orders').onDelete('CASCADE');
      table.integer('product_id').unsigned().notNullable()
        .references('product_id').inTable('products');
      table.integer('quantity').notNullable();
      table.decimal('unit_price', 10, 2).notNullable();
      table.decimal('discount', 10, 2).defaultTo(0);
      table.primary(['order_id', 'product_id']);
    })
    .createTable('payments', (table) => {
      table.increments('payment_id').primary();
      table.integer('order_id').unsigned().notNullable()
        .references('order_id').inTable('orders');
      table.enu('method', ['bKash', 'Nagad', 'Rocket', 'COD', 'Card']).notNullable();
      table.string('transaction_id', 100).nullable();
      table.decimal('amount', 10, 2).notNullable();
      table.enu('status', ['pending', 'completed', 'failed', 'refunded']).defaultTo('pending');
      table.timestamp('paid_at').nullable();
      table.timestamps(true, true);
      table.index(['order_id', 'status']);
      table.index('transaction_id');
    });
};

exports.down = function(knex) {
  return knex.schema
    .dropTableIfExists('payments')
    .dropTableIfExists('order_items')
    .dropTableIfExists('orders')
    .dropTableIfExists('delivery_persons')
    .dropTableIfExists('products')
    .dropTableIfExists('sellers')
    .dropTableIfExists('categories')
    .dropTableIfExists('addresses')
    .dropTableIfExists('customers');
};
```

---

### ৩. Normalization vs Performance

#### কখন Normalized Schema ধীর হয়

Normalized schema-তে **JOIN** অপারেশন বেশি হয়। নিচের query বিবেচনা করুন:

```sql
-- একটি অর্ডারের সম্পূর্ণ তথ্য — ৭টি টেবিল JOIN
SELECT
    o.order_id,
    c.customer_name, c.customer_phone,
    a.division, a.district, a.address_line,
    p.product_name, cat.category_name,
    oi.quantity, oi.unit_price,
    pay.method AS payment_method, pay.status AS payment_status,
    dp.person_name AS delivery_person
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN addresses a ON o.shipping_address_id = a.address_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
LEFT JOIN payments pay ON o.order_id = pay.order_id
LEFT JOIN delivery_persons dp ON o.delivery_person_id = dp.delivery_person_id
WHERE o.order_id = 1001;
```

**সমস্যা:** এই query প্রতিটি অর্ডার ভিউতে চালাতে হলে high-traffic সিস্টেমে performance bottleneck হবে।

#### Strategic Denormalization উদাহরণ

```sql
-- কৌশল ১: Cached/Computed Column
ALTER TABLE orders ADD COLUMN customer_name_cache VARCHAR(100);
ALTER TABLE orders ADD COLUMN items_count INT DEFAULT 0;

-- Trigger দিয়ে স্বয়ংক্রিয় আপডেট
CREATE TRIGGER trg_order_items_count
AFTER INSERT ON order_items
FOR EACH ROW
BEGIN
    UPDATE orders
    SET items_count = (
        SELECT COUNT(*) FROM order_items WHERE order_id = NEW.order_id
    )
    WHERE order_id = NEW.order_id;
END;

-- কৌশল ২: Summary Table (reporting-এর জন্য)
CREATE TABLE daily_sales_summary (
    summary_date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(12,2),
    total_items_sold INT,
    avg_order_value DECIMAL(10,2),
    top_division VARCHAR(50),
    bkash_count INT,
    nagad_count INT,
    cod_count INT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- প্রতিদিন রাতে populate করা হবে
INSERT INTO daily_sales_summary (summary_date, total_orders, total_revenue, total_items_sold, avg_order_value)
SELECT
    DATE(o.created_at),
    COUNT(DISTINCT o.order_id),
    SUM(oi.quantity * oi.unit_price - oi.discount),
    SUM(oi.quantity),
    AVG(o.cached_total)
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE DATE(o.created_at) = CURDATE() - INTERVAL 1 DAY
GROUP BY DATE(o.created_at)
ON DUPLICATE KEY UPDATE
    total_orders = VALUES(total_orders),
    total_revenue = VALUES(total_revenue),
    total_items_sold = VALUES(total_items_sold),
    avg_order_value = VALUES(avg_order_value);
```

#### Read Replicas + Normalized Write DB

```
┌─────────────────────────────────────────────────────┐
│                  Application Layer                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│   Write Operations          Read Operations         │
│   (INSERT/UPDATE/DELETE)    (SELECT/JOIN)           │
│         │                        │                  │
│         v                        v                  │
│  ┌──────────────┐     ┌──────────────────┐         │
│  │  Primary DB  │────>│  Read Replica(s) │         │
│  │ (Normalized) │     │  (Normalized +   │         │
│  │  3NF Schema  │     │   Denormalized   │         │
│  │              │     │   Views/Tables)  │         │
│  └──────────────┘     └──────────────────┘         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### CQRS Approach (Command Query Responsibility Segregation)

```php
// PHP Laravel — CQRS Pattern
// Command Side (Write): Normalized tables
class CreateOrderCommand
{
    public function execute(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            $order = Order::create([
                'customer_id' => $data['customer_id'],
                'shipping_address_id' => $data['address_id'],
                'delivery_status' => 'pending',
            ]);

            foreach ($data['items'] as $item) {
                $product = Product::findOrFail($item['product_id']);
                $order->items()->create([
                    'product_id' => $item['product_id'],
                    'quantity' => $item['quantity'],
                    'unit_price' => $product->base_price,
                ]);
                $product->decrement('stock_quantity', $item['quantity']);
            }

            $order->recalculateTotal();
            event(new OrderCreated($order));  // Event dispatch → read model আপডেট হবে

            return $order;
        });
    }
}

// Query Side (Read): Denormalized read model
class OrderReadModel extends Model
{
    protected $table = 'order_read_models';  // Denormalized flat table
    protected $casts = ['items' => 'array', 'customer' => 'array', 'address' => 'array'];
}

// Event Listener — read model আপডেট
class UpdateOrderReadModel
{
    public function handle(OrderCreated $event): void
    {
        $order = $event->order->load(['customer', 'shippingAddress', 'items.product']);

        OrderReadModel::updateOrCreate(
            ['order_id' => $order->order_id],
            [
                'customer' => $order->customer->toArray(),
                'address' => $order->shippingAddress->toArray(),
                'items' => $order->items->map(fn ($i) => [
                    'name' => $i->product->product_name,
                    'qty' => $i->quantity,
                    'price' => $i->unit_price,
                ])->toArray(),
                'total' => $order->cached_total,
                'status' => $order->delivery_status,
            ]
        );
    }
}
```

```javascript
// Node.js — CQRS Pattern
// Command Side (Write): Normalized
class CreateOrderCommand {
  async execute(data) {
    const transaction = await sequelize.transaction();
    try {
      const order = await Order.create({
        customerId: data.customerId,
        shippingAddressId: data.addressId,
        deliveryStatus: 'pending',
      }, { transaction });

      for (const item of data.items) {
        const product = await Product.findByPk(item.productId, { transaction });
        await OrderItem.create({
          orderId: order.orderId,
          productId: item.productId,
          quantity: item.quantity,
          unitPrice: product.basePrice,
        }, { transaction });
        await product.decrement('stockQuantity', { by: item.quantity, transaction });
      }

      await order.recalculateTotal();
      await transaction.commit();

      // Event emit → read model আপডেট
      eventBus.emit('order.created', { orderId: order.orderId });
      return order;
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}

// Query Side (Read): Denormalized
class OrderQueryService {
  async getOrderDetails(orderId) {
    // Denormalized read model থেকে সরাসরি পড়া — কোনো JOIN নেই
    return await OrderReadModel.findOne({ where: { orderId }, raw: true });
  }

  async getOrdersList(customerId, { page = 1, limit = 20 } = {}) {
    return await OrderReadModel.findAndCountAll({
      where: { 'customer.customer_id': customerId },
      order: [['createdAt', 'DESC']],
      offset: (page - 1) * limit,
      limit,
      raw: true,
    });
  }
}
```

---

### ৪. Normalization in NoSQL

NoSQL (বিশেষত Document Database যেমন MongoDB) তে normalization-এর ধারণা ভিন্ন। এখানে **embedding** vs **referencing** — এই দুটি কৌশল ব্যবহৃত হয়।

#### কখন Embed করবেন, কখন Reference করবেন

```
┌─────────────────────────┬─────────────────────────┐
│     Embed করুন          │    Reference করুন        │
├─────────────────────────┼─────────────────────────┤
│ ১:১ বা ১:কিছু সম্পর্ক    │ ১:অনেক (unbounded)      │
│ ডেটা একসাথে পড়া হয়      │ ডেটা আলাদা আলাদা পড়া হয় │
│ ডেটা কম পরিবর্তন হয়     │ ডেটা ঘন ঘন পরিবর্তন হয়  │
│ Document <16MB থাকবে    │ Document বড় হতে পারে     │
│ Atomic update প্রয়োজন   │ স্বাধীন update প্রয়োজন    │
│                         │ অনেক জায়গায় reuse হয়    │
└─────────────────────────┴─────────────────────────┘
```

#### PHP MongoDB উদাহরণ (Laravel MongoDB)

```php
// Embedded approach — অর্ডারের ভেতর আইটেম embed করা
// app/Models/MongoOrder.php (jenssegers/mongodb)
use Jenssegers\Mongodb\Eloquent\Model;

class MongoOrder extends Model
{
    protected $connection = 'mongodb';
    protected $collection = 'orders';

    protected $fillable = [
        'customer', 'shipping_address', 'items',
        'payment', 'delivery_status', 'total',
    ];

    // Embedded document হিসেবে সংরক্ষিত (denormalized)
    // {
    //   "customer": { "name": "রহিম", "phone": "01712345678" },
    //   "shipping_address": { "division": "ঢাকা", "address": "মিরপুর..." },
    //   "items": [
    //     { "product_name": "Samsung Galaxy", "qty": 1, "price": 95000 },
    //     { "product_name": "Spigen Case", "qty": 2, "price": 1500 }
    //   ],
    //   "payment": { "method": "bKash", "trx_id": "TRX123", "status": "completed" },
    //   "delivery_status": "delivered",
    //   "total": 98000
    // }
}

// Referenced approach — প্রোডাক্ট আলাদা collection-এ
class MongoProduct extends Model
{
    protected $connection = 'mongodb';
    protected $collection = 'products';
}

// Hybrid: অর্ডারে প্রোডাক্টের snapshot embed, কিন্তু product_id reference রাখা
class OrderService
{
    public function createOrder(array $data): MongoOrder
    {
        $items = collect($data['items'])->map(function ($item) {
            $product = MongoProduct::findOrFail($item['product_id']);
            return [
                'product_id' => $product->_id,           // Reference
                'product_name' => $product->name,         // Embedded snapshot
                'quantity' => $item['quantity'],
                'unit_price' => $product->base_price,     // Historical price
            ];
        })->toArray();

        return MongoOrder::create([
            'customer' => [
                'customer_id' => $data['customer_id'],    // Reference
                'name' => $data['customer_name'],          // Embedded
                'phone' => $data['customer_phone'],        // Embedded
            ],
            'items' => $items,
            'delivery_status' => 'pending',
            'total' => collect($items)->sum(fn ($i) => $i['quantity'] * $i['unit_price']),
        ]);
    }
}
```

#### Node.js Mongoose উদাহরণ

```javascript
const mongoose = require('mongoose');
const { Schema } = mongoose;

// Embedded Sub-document Schema
const orderItemSchema = new Schema({
  productId: { type: Schema.Types.ObjectId, ref: 'Product' },  // Reference
  productName: String,       // Embedded snapshot
  quantity: { type: Number, required: true, min: 1 },
  unitPrice: { type: Number, required: true },
}, { _id: false });

const addressSchema = new Schema({
  division: { type: String, required: true },
  district: String,
  upazila: String,
  addressLine: { type: String, required: true },
  postalCode: String,
}, { _id: false });

const paymentSchema = new Schema({
  method: { type: String, enum: ['bKash', 'Nagad', 'Rocket', 'COD', 'Card'], required: true },
  transactionId: String,
  amount: { type: Number, required: true },
  status: { type: String, enum: ['pending', 'completed', 'failed', 'refunded'], default: 'pending' },
  paidAt: Date,
}, { _id: false });

// Order Schema — Hybrid (embed + reference)
const orderSchema = new Schema({
  customer: {
    customerId: { type: Schema.Types.ObjectId, ref: 'Customer', required: true },
    name: String,            // Denormalized snapshot
    phone: String,           // Denormalized snapshot
  },
  shippingAddress: addressSchema,  // Embedded (অর্ডারের সময়ের ঠিকানা freeze করা)
  items: [orderItemSchema],        // Embedded array
  payment: paymentSchema,          // Embedded
  deliveryStatus: {
    type: String,
    enum: ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'],
    default: 'pending',
  },
  deliveryPersonId: { type: Schema.Types.ObjectId, ref: 'DeliveryPerson' },  // Reference only
  cachedTotal: Number,
}, { timestamps: true });

// Virtual populate — referenced ডেটা লোড করা (normalized access)
orderSchema.virtual('deliveryPerson', {
  ref: 'DeliveryPerson',
  localField: 'deliveryPersonId',
  foreignField: '_id',
  justOne: true,
});

// Pre-save hook — total গণনা
orderSchema.pre('save', function(next) {
  if (this.isModified('items')) {
    this.cachedTotal = this.items.reduce(
      (sum, item) => sum + item.quantity * item.unitPrice, 0
    );
  }
  next();
});

// Indexing কৌশল
orderSchema.index({ 'customer.customerId': 1, createdAt: -1 });
orderSchema.index({ deliveryStatus: 1 });
orderSchema.index({ 'payment.method': 1, 'payment.status': 1 });

const Order = mongoose.model('Order', orderSchema);

// Referenced Product Schema (normalized — আলাদা collection)
const productSchema = new Schema({
  productName: { type: String, required: true },
  category: { type: Schema.Types.ObjectId, ref: 'Category' },
  sellerId: { type: Schema.Types.ObjectId, ref: 'Seller' },
  basePrice: { type: Number, required: true },
  sku: { type: String, unique: true },
  stockQuantity: { type: Number, default: 0 },
}, { timestamps: true });

const Product = mongoose.model('Product', productSchema);

// ব্যবহার
async function createMongoOrder(data) {
  const itemsWithSnapshot = await Promise.all(
    data.items.map(async (item) => {
      const product = await Product.findById(item.productId);
      return {
        productId: product._id,
        productName: product.productName,  // Snapshot at order time
        quantity: item.quantity,
        unitPrice: product.basePrice,
      };
    })
  );

  const order = new Order({
    customer: {
      customerId: data.customerId,
      name: data.customerName,
      phone: data.customerPhone,
    },
    shippingAddress: data.address,
    items: itemsWithSnapshot,
    payment: { method: data.paymentMethod, amount: 0, status: 'pending' },
  });

  return await order.save();
}
```

---

## ✅ সুবিধা ও ❌ অসুবিধা (প্রতিটি Normal Form)

| Normal Form | ✅ সুবিধা | ❌ অসুবিধা |
|-------------|-----------|------------|
| **1NF** | Atomic data, query-friendly, basic structure | এখনও redundancy আছে, partial dependency বিদ্যমান |
| **2NF** | Partial dependency দূর, কম redundancy | Transitive dependency এখনও আছে, বেশি টেবিল |
| **3NF** | ন্যূনতম redundancy, clean schema, update-safe | JOIN বেশি, complex queries, read performance কমতে পারে |
| **BCNF** | সব determinant candidate key, তাত্ত্বিক পরিচ্ছন্নতা | Dependency preservation নষ্ট হতে পারে, অতিরিক্ত জটিলতা |
| **4NF** | Multi-valued dependency দূর, কম space | অনেক ছোট টেবিল, JOIN আরও বেশি |
| **5NF** | সম্পূর্ণ redundancy-মুক্ত, গাণিতিকভাবে optimal | অত্যন্ত জটিল, বাস্তবে খুব কমই প্রয়োজন, রক্ষণাবেক্ষণ কঠিন |

---

## ⚠️ সাধারণ ভুল

### ১. Over-Normalization (অতিরিক্ত নরমালাইজেশন)

```sql
-- ❌ অতিরিক্ত: phone number-এর জন্য আলাদা টেবিল (যদি শুধু ফোন নম্বরই থাকে)
CREATE TABLE phone_country_codes (code_id INT PRIMARY KEY, code VARCHAR(5));
CREATE TABLE phone_area_codes (area_id INT PRIMARY KEY, area_code VARCHAR(5));
CREATE TABLE phone_numbers (
    phone_id INT PRIMARY KEY,
    code_id INT REFERENCES phone_country_codes(code_id),
    area_id INT REFERENCES phone_area_codes(area_id),
    local_number VARCHAR(10)
);
-- এই মাত্রার বিভাজন প্রায় সব সময় অপ্রয়োজনীয়
-- বাংলাদেশে সব নম্বর +880 দিয়ে শুরু, 11 digit — একটি VARCHAR(20) কলামই যথেষ্ট
```

### ২. Under-Normalization (অপর্যাপ্ত নরমালাইজেশন)

```sql
-- ❌ সব তথ্য একটি টেবিলে রেখে দেওয়া
CREATE TABLE everything (
    order_id INT, order_date DATE,
    customer_name VARCHAR(100), customer_phone VARCHAR(20),
    customer_address TEXT, product_name VARCHAR(200),
    product_price DECIMAL(10,2), quantity INT,
    payment_method VARCHAR(50), delivery_person VARCHAR(100)
    -- ... ২০+ কলাম
);
-- এটি legacy সিস্টেম বা Excel-থেকে-DB migration-এ সবচেয়ে বেশি দেখা যায়
```

### ৩. Query Pattern উপেক্ষা করা

```sql
-- ❌ Schema ঠিক আছে কিন্তু query pattern বিবেচনা করা হয়নি
-- প্রতিটি product page view-তে ৫টি JOIN চালানো
SELECT p.*, c.category_name, s.shop_name, AVG(r.rating), COUNT(r.review_id)
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN sellers s ON p.seller_id = s.seller_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id;
-- প্রতি সেকেন্ডে হাজার বার চালালে DB ধীর হবে

-- ✅ সমাধান: Materialized view বা cached columns
ALTER TABLE products ADD COLUMN cached_avg_rating DECIMAL(2,1);
ALTER TABLE products ADD COLUMN cached_review_count INT DEFAULT 0;
ALTER TABLE products ADD COLUMN seller_name_cache VARCHAR(200);
```

### ৪. EAV (Entity-Attribute-Value) Anti-Pattern

```sql
-- ❌ EAV Pattern — সবকিছু key-value হিসেবে সংরক্ষণ
CREATE TABLE product_attributes (
    product_id INT,
    attribute_name VARCHAR(100),   -- 'color', 'size', 'weight', 'brand'...
    attribute_value VARCHAR(500),
    PRIMARY KEY (product_id, attribute_name)
);

-- সমস্যা:
-- ১. Data type enforce করা যায় না (সবই VARCHAR)
-- ২. Foreign key constraint দেওয়া যায় না
-- ৩. Query অত্যন্ত জটিল:
SELECT
    p.product_id,
    MAX(CASE WHEN pa.attribute_name = 'color' THEN pa.attribute_value END) AS color,
    MAX(CASE WHEN pa.attribute_name = 'size' THEN pa.attribute_value END) AS size,
    MAX(CASE WHEN pa.attribute_name = 'brand' THEN pa.attribute_value END) AS brand
FROM products p
LEFT JOIN product_attributes pa ON p.product_id = pa.product_id
GROUP BY p.product_id;
-- ৪. Indexing কঠিন, performance খারাপ

-- ✅ বিকল্প: JSON column (MySQL 8.0+ / PostgreSQL)
ALTER TABLE products ADD COLUMN attributes JSON;
-- {"color": "লাল", "size": "XL", "brand": "Apex", "material": "তুলা"}

-- অথবা ক্যাটাগরি-ভিত্তিক concrete table (ভালো পদ্ধতি):
CREATE TABLE product_clothing_attrs (
    product_id INT PRIMARY KEY REFERENCES products(product_id),
    color VARCHAR(50),
    size ENUM('XS','S','M','L','XL','XXL'),
    material VARCHAR(100),
    INDEX idx_color_size (color, size)
);
```

---

## 🧪 টেস্টিং Database Schema

### PHPUnit — Schema Validation ও Data Integrity Tests

```php
// tests/Feature/DatabaseSchemaTest.php
namespace Tests\Feature;

use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;
use App\Models\{Customer, Order, OrderItem, Product, Payment};

class DatabaseSchemaTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function all_required_tables_exist(): void
    {
        $requiredTables = [
            'customers', 'addresses', 'categories', 'sellers',
            'products', 'orders', 'order_items', 'payments',
            'delivery_persons',
        ];

        foreach ($requiredTables as $table) {
            $this->assertTrue(
                Schema::hasTable($table),
                "টেবিল '{$table}' বিদ্যমান নেই"
            );
        }
    }

    /** @test */
    public function orders_table_has_correct_columns(): void
    {
        $this->assertTrue(Schema::hasColumns('orders', [
            'order_id', 'customer_id', 'shipping_address_id',
            'delivery_status', 'cached_total',
        ]));
    }

    /** @test */
    public function foreign_key_constraints_prevent_orphan_records(): void
    {
        $this->expectException(\Illuminate\Database\QueryException::class);

        // অস্তিত্বহীন customer_id দিয়ে অর্ডার তৈরি — ব্যর্থ হওয়া উচিত
        DB::table('orders')->insert([
            'customer_id' => 99999,
            'shipping_address_id' => 1,
            'delivery_status' => 'pending',
        ]);
    }

    /** @test */
    public function cascade_delete_removes_dependent_records(): void
    {
        $customer = Customer::factory()->create();
        $address = $customer->addresses()->create([
            'division' => 'ঢাকা', 'address_line' => 'মিরপুর',
        ]);
        $order = Order::factory()->create([
            'customer_id' => $customer->customer_id,
            'shipping_address_id' => $address->address_id,
        ]);
        $item = OrderItem::factory()->create(['order_id' => $order->order_id]);

        $order->delete();

        $this->assertDatabaseMissing('order_items', [
            'order_id' => $order->order_id,
        ]);
    }

    /** @test */
    public function no_data_redundancy_in_normalized_tables(): void
    {
        $customer = Customer::factory()->create(['customer_phone' => '01712345678']);

        // একই ফোন নম্বরে দ্বিতীয় customer তৈরি — unique constraint-এ ব্যর্থ হওয়া উচিত
        $this->expectException(\Illuminate\Database\QueryException::class);
        Customer::factory()->create(['customer_phone' => '01712345678']);
    }

    /** @test */
    public function order_total_stays_consistent_with_items(): void
    {
        $order = Order::factory()->create();
        $product = Product::factory()->create(['base_price' => 1000]);

        OrderItem::create([
            'order_id' => $order->order_id,
            'product_id' => $product->product_id,
            'quantity' => 3,
            'unit_price' => 1000,
            'discount' => 100,
        ]);

        $order->recalculateTotal();

        $expectedTotal = (3 * 1000) - 100; // ২৯০০
        $this->assertEquals($expectedTotal, $order->fresh()->cached_total);
    }

    /** @test */
    public function payment_amount_matches_order_total(): void
    {
        $order = Order::factory()->create(['cached_total' => 5000]);

        $payment = Payment::create([
            'order_id' => $order->order_id,
            'method' => 'bKash',
            'amount' => 5000,
            'status' => 'completed',
        ]);

        $this->assertEquals(
            $order->cached_total,
            $payment->amount,
            'Payment amount should match order total'
        );
    }
}
```

### Jest — Schema ও Integrity Tests (Node.js)

```javascript
// tests/database/schema.test.js
const { sequelize, Customer, Order, OrderItem, Product, Payment } = require('../../models');

describe('Database Schema Validation', () => {
  beforeAll(async () => {
    await sequelize.sync({ force: true });
  });

  afterAll(async () => {
    await sequelize.close();
  });

  describe('টেবিল অস্তিত্ব পরীক্ষা', () => {
    const requiredTables = [
      'customers', 'addresses', 'categories', 'sellers',
      'products', 'orders', 'order_items', 'payments',
    ];

    test.each(requiredTables)('"%s" টেবিল বিদ্যমান', async (tableName) => {
      const [results] = await sequelize.query(
        `SELECT COUNT(*) as count FROM information_schema.tables
         WHERE table_name = '${tableName}'`
      );
      expect(results[0].count).toBeGreaterThan(0);
    });
  });

  describe('Foreign Key Constraint পরীক্ষা', () => {
    test('অস্তিত্বহীন customer_id দিয়ে অর্ডার তৈরি ব্যর্থ হয়', async () => {
      await expect(
        Order.create({
          customerId: 99999,
          shippingAddressId: 1,
          deliveryStatus: 'pending',
        })
      ).rejects.toThrow();
    });
  });

  describe('Data Integrity পরীক্ষা', () => {
    test('unique constraint — একই ফোন নম্বরে দুজন customer তৈরি করা যায় না', async () => {
      await Customer.create({
        customerName: 'রহিম',
        customerPhone: '01712345678',
      });

      await expect(
        Customer.create({
          customerName: 'করিম',
          customerPhone: '01712345678',
        })
      ).rejects.toThrow();
    });

    test('order total সঠিকভাবে গণনা হয়', async () => {
      const customer = await Customer.create({
        customerName: 'টেস্ট ইউজার',
        customerPhone: '01700000001',
      });

      const order = await Order.create({
        customerId: customer.customerId,
        shippingAddressId: 1,
        deliveryStatus: 'pending',
      });

      const product = await Product.create({
        productName: 'Test Product',
        basePrice: 500,
        sku: 'TEST-001',
      });

      await OrderItem.create({
        orderId: order.orderId,
        productId: product.productId,
        quantity: 4,
        unitPrice: 500,
        discount: 50,
      });

      await order.recalculateTotal();
      const refreshed = await Order.findByPk(order.orderId);

      expect(parseFloat(refreshed.cachedTotal)).toBe(4 * 500 - 50); // ১৯৫০
    });

    test('cascade delete — অর্ডার মুছলে আইটেমও মুছে যায়', async () => {
      const customer = await Customer.create({
        customerName: 'ডিলিট টেস্ট',
        customerPhone: '01700000002',
      });

      const order = await Order.create({
        customerId: customer.customerId,
        shippingAddressId: 1,
      });

      await OrderItem.create({
        orderId: order.orderId,
        productId: 1,
        quantity: 1,
        unitPrice: 100,
      });

      await order.destroy();

      const orphanItems = await OrderItem.findAll({
        where: { orderId: order.orderId },
      });
      expect(orphanItems).toHaveLength(0);
    });
  });
});
```

---

## 📏 কখন কোন Normal Form পর্যন্ত যাবেন

### সিস্টেমের ধরন অনুযায়ী সিদ্ধান্ত গাইড

```
┌────────────────────────────┬──────────────────────┬─────────────────────────────────┐
│ সিস্টেমের ধরন               │ প্রস্তাবিত Normal Form │ কারণ                             │
├────────────────────────────┼──────────────────────┼─────────────────────────────────┤
│ OLTP (e-commerce, banking) │ 3NF / BCNF           │ Data integrity সর্বোচ্চ গুরুত্বপূর্ণ  │
│ OLAP (analytics, DWH)      │ 2NF + denormalize    │ Read performance, star schema   │
│ CMS / Blog                 │ 3NF                  │ সহজ schema, মাঝারি traffic      │
│ IoT / Logging              │ 1NF + partitioning   │ High write throughput প্রয়োজন    │
│ Social Media               │ 3NF + caching layer  │ Complex relations, high read    │
│ Microservices              │ 3NF per service      │ Bounded context, স্বাধীন schema  │
│ Real-time Chat             │ NoSQL (embed)        │ Low latency, simple access      │
│ ERP System                 │ BCNF                 │ Complex business rules          │
│ Reporting Dashboard        │ Materialized views   │ Pre-computed aggregates         │
│ Mobile App Backend         │ 3NF + API caching    │ Network latency কমানো            │
└────────────────────────────┴──────────────────────┴─────────────────────────────────┘
```

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট

```
শুরু → Schema Design
  │
  ├── ১. সব ডেটা চিহ্নিত করুন
  │
  ├── ২. 3NF পর্যন্ত normalize করুন (প্রায় সবসময়)
  │
  ├── ৩. Query pattern বিশ্লেষণ করুন
  │     │
  │     ├── Read-heavy? → Strategic denormalization বিবেচনা করুন
  │     │                   ├── Materialized views
  │     │                   ├── Cached columns
  │     │                   └── Summary tables
  │     │
  │     └── Write-heavy? → 3NF/BCNF রাখুন, indexing optimize করুন
  │
  ├── ৪. Load test করুন
  │     │
  │     ├── ঠিক আছে? → শেষ ✅
  │     │
  │     └── ধীর? → Explain Analyze চালান
  │               ├── Missing index? → Index যোগ করুন
  │               ├── Too many JOINs? → Denormalize করুন
  │               └── Table too large? → Partitioning বিবেচনা করুন
  │
  └── ৫. Monitor ও iterate করুন
```

---

## 📋 সারসংক্ষেপ টেবিল (সব Normal Form তুলনা)

| বৈশিষ্ট্য | UNF | 1NF | 2NF | 3NF | BCNF | 4NF | 5NF |
|-----------|-----|-----|-----|-----|------|-----|-----|
| **Atomic values** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Primary Key** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **No partial dependency** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **No transitive dependency** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Every determinant is candidate key** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **No multi-valued dependency** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **No join dependency** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Redundancy মাত্রা** | 🔴🔴🔴 | 🔴🔴 | 🟡🟡 | 🟢 | 🟢 | 🟢 | 🟢 |
| **Query জটিলতা** | 🟢 | 🟢 | 🟡 | 🟡🟡 | 🟡🟡 | 🔴 | 🔴🔴 |
| **বাস্তব ব্যবহার** | Legacy | বিরল | মাঝেমাঝে | **সবচেয়ে বেশি** | ERP/ব্যাংকিং | বিরল | তাত্ত্বিক |

### মনে রাখার সূত্র

```
🔑 "The Key, The Whole Key, and Nothing But The Key, So Help Me Codd"

1NF → The Key              (Primary Key থাকতে হবে, atomic values)
2NF → The Whole Key        (সম্পূর্ণ key-এর উপর নির্ভরশীল, partial dependency নেই)
3NF → Nothing But The Key  (শুধুমাত্র key-এর উপর নির্ভরশীল, transitive dependency নেই)
```

### বাংলাদেশী প্রেক্ষাপটে চূড়ান্ত পরামর্শ

১. **Daraz/Chaldal-এর মতো e-commerce:** 3NF + strategic denormalization (order snapshots, cached totals)
২. **bKash/Nagad-এর মতো fintech:** BCNF — data integrity সর্বোচ্চ অগ্রাধিকার
৩. **Prothom Alo-এর মতো news portal:** 3NF + heavy caching (Redis/Memcached)
৪. **Pathao-এর মতো ride-sharing:** Mixed — 3NF for core, denormalized for real-time tracking
৫. **সরকারি সেবা (a2i):** 3NF — structured data, audit trail গুরুত্বপূর্ণ

> **সবচেয়ে গুরুত্বপূর্ণ নীতি:** "প্রথমে normalize করুন, তারপর প্রয়োজন অনুসারে denormalize করুন — কখনোই উল্টো নয়।"

---

*এই গাইডটি senior engineers-দের জন্য তৈরি। Normalization শুধু তাত্ত্বিক জ্ঞান নয় — এটি একটি সচেতন engineering decision যা system-এর performance, maintainability এবং data integrity-র মধ্যে ভারসাম্য রক্ষা করে।*
