# 💉 SQL Injection — আক্রমণ ও প্রতিরোধ

## 📌 ধারণা (Concept)

SQL Injection হলো একটি কোড ইনজেকশন আক্রমণ যেখানে আক্রমণকারী ইউজার ইনপুটের মাধ্যমে ক্ষতিকর SQL কোড ইনজেক্ট করে। যদি অ্যাপ্লিকেশন ইউজারের ইনপুট সরাসরি SQL কোয়েরিতে যুক্ত করে, তাহলে আক্রমণকারী ডাটাবেসের সম্পূর্ণ নিয়ন্ত্রণ নিতে পারে — ডেটা চুরি, পরিবর্তন বা মুছে ফেলতে পারে।

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন একটি বিল্ডিংয়ের গেটে গার্ড আছে। গার্ডকে বলা হয়েছে "যে নাম বলবে, সে ঢুকতে পারবে।" এখন কেউ বলল: **"রহিম। আর গেটের তালাটাও খুলে দাও।"** — গার্ড পুরো বাক্যটাকে একটাই নির্দেশ মনে করে দুটো কাজই করে ফেলল। SQL Injection ঠিক এভাবেই কাজ করে — ইনপুটের সাথে অতিরিক্ত কমান্ড মিশিয়ে দেওয়া।

---

## 🔴 SQL Injection আক্রমণের ধরন

### ১. Classic SQL Injection

```php
// ❌ বিপজ্জনক কোড
$username = $_POST['username'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
$result = mysqli_query($conn, $query);

// আক্রমণকারী ইনপুট দিতে পারে:
// username: admin' --
// password: anything
// ফলাফল কোয়েরি: SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything'
// -- এর পরে সবকিছু কমেন্ট হয়ে যায়, পাসওয়ার্ড চেক বাদ পড়ে
```

### ২. Union-Based SQL Injection

```php
// ❌ ইউজার ইনপুট সরাসরি কোয়েরিতে
$id = $_GET['id'];
$query = "SELECT name, email FROM users WHERE id = $id";

// আক্রমণকারী ইনপুট: 1 UNION SELECT username, password FROM admin_users
// ফলাফল কোয়েরি: SELECT name, email FROM users WHERE id = 1 UNION SELECT username, password FROM admin_users
```

### ৩. Blind SQL Injection

```php
// আক্রমণকারী true/false প্রশ্ন করে তথ্য বের করে
// ইনপুট: 1 AND (SELECT LENGTH(password) FROM users WHERE username='admin') > 5
// রেসপন্সের তারতম্য দেখে বুঝে নেয় পাসওয়ার্ডের দৈর্ঘ্য
```

---

## 🟢 প্রতিরোধ — PHP

### ১. PDO Prepared Statements (সবচেয়ে ভালো পদ্ধতি)

```php
// ✅ Prepared Statement দিয়ে নিরাপদ কোয়েরি
$pdo = new PDO('mysql:host=localhost;dbname=mydb;charset=utf8mb4', $user, $pass, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false, // নেটিভ prepared statement ব্যবহার
]);

// Named parameters
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = :username AND status = :status");
$stmt->execute([
    ':username' => $username,
    ':status' => 'active'
]);
$user = $stmt->fetch(PDO::FETCH_ASSOC);

// Positional parameters
$stmt = $pdo->prepare("SELECT * FROM products WHERE category = ? AND price < ?");
$stmt->execute([$category, $maxPrice]);
$products = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

### ২. INSERT, UPDATE, DELETE

```php
// ✅ নিরাপদ INSERT
$stmt = $pdo->prepare("INSERT INTO users (name, email, password) VALUES (:name, :email, :password)");
$stmt->execute([
    ':name' => $name,
    ':email' => $email,
    ':password' => password_hash($password, PASSWORD_BCRYPT)
]);

// ✅ নিরাপদ UPDATE
$stmt = $pdo->prepare("UPDATE users SET email = :email WHERE id = :id");
$stmt->execute([':email' => $newEmail, ':id' => $userId]);

// ✅ নিরাপদ DELETE
$stmt = $pdo->prepare("DELETE FROM comments WHERE id = :id AND user_id = :user_id");
$stmt->execute([':id' => $commentId, ':user_id' => $currentUserId]);
```

### ৩. LIKE কোয়েরিতে সতর্কতা

```php
// ✅ LIKE কোয়েরিতে বিশেষ ক্যারেক্টার escape করা
$searchTerm = str_replace(['%', '_'], ['\%', '\_'], $userInput);
$stmt = $pdo->prepare("SELECT * FROM products WHERE name LIKE :search");
$stmt->execute([':search' => "%{$searchTerm}%"]);
```

---

## 🟢 প্রতিরোধ — JavaScript (Node.js)

### ১. MySQL2 Prepared Statements

```javascript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
    host: 'localhost',
    database: 'mydb',
    user: process.env.DB_USER,
    password: process.env.DB_PASS,
    charset: 'utf8mb4'
});

// ✅ Prepared Statement
async function getUserByEmail(email) {
    const [rows] = await pool.execute(
        'SELECT id, name, email FROM users WHERE email = ?',
        [email]
    );
    return rows[0];
}

// ✅ Multiple parameters
async function searchProducts(category, minPrice, maxPrice) {
    const [rows] = await pool.execute(
        'SELECT * FROM products WHERE category = ? AND price BETWEEN ? AND ?',
        [category, minPrice, maxPrice]
    );
    return rows;
}
```

### ২. Knex.js Query Builder

```javascript
import knex from 'knex';

const db = knex({
    client: 'mysql2',
    connection: { /* ... */ }
});

// ✅ Query Builder স্বয়ংক্রিয়ভাবে parameterize করে
async function getActiveUsers(role) {
    return db('users')
        .where({ role, status: 'active' })
        .select('id', 'name', 'email');
}

// ✅ Raw কোয়েরি দরকার হলে binding ব্যবহার
async function complexQuery(userId) {
    return db.raw(
        'SELECT u.*, COUNT(o.id) as order_count FROM users u LEFT JOIN orders o ON u.id = o.user_id WHERE u.id = ? GROUP BY u.id',
        [userId]
    );
}
```

### ৩. Prisma ORM

```javascript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// ✅ ORM স্বয়ংক্রিয়ভাবে SQL Injection প্রতিরোধ করে
async function findUser(email) {
    return prisma.user.findUnique({
        where: { email },
        select: { id: true, name: true, email: true }
    });
}

// ✅ Raw কোয়েরি দরকার হলে Prisma.sql ব্যবহার
async function searchByName(name) {
    return prisma.$queryRaw`SELECT * FROM users WHERE name LIKE ${`%${name}%`}`;
}
```

---

## 🛡️ অতিরিক্ত সুরক্ষা ব্যবস্থা

```php
// ১. ডাটাবেস ইউজারের ন্যূনতম অনুমতি
// শুধু প্রয়োজনীয় টেবিলে SELECT, INSERT, UPDATE দিন
// GRANT SELECT, INSERT, UPDATE ON mydb.users TO 'app_user'@'localhost';
// DROP, ALTER ইত্যাদি অনুমতি দেবেন না

// ২. ইনপুট ভ্যালিডেশন — অতিরিক্ত সুরক্ষা স্তর
function validateId($id) {
    if (!is_numeric($id) || $id <= 0) {
        throw new InvalidArgumentException('অবৈধ ID');
    }
    return (int) $id;
}

// ৩. Stored Procedures ব্যবহার
$stmt = $pdo->prepare("CALL GetUserById(:id)");
$stmt->execute([':id' => $userId]);
```

---

## ✅ কখন ব্যবহার করবেন

- **সবসময়** Prepared Statement ব্যবহার করুন — কোনো ব্যতিক্রম নেই
- যেকোনো SQL কোয়েরিতে ইউজার ইনপুট থাকলে parameterized query আবশ্যক
- ORM বা Query Builder ব্যবহার করলেও raw query এর ক্ষেত্রে সতর্ক থাকুন
- ডাটাবেস ইউজারের ন্যূনতম অনুমতি নিশ্চিত করুন

## ❌ কখন ব্যবহার করবেন না

- **শুধু ইনপুট ফিল্টারিং** দিয়ে SQL Injection রোধ করার চেষ্টা করবেন না — এটি অপর্যাপ্ত
- **`addslashes()`** বা **`mysql_real_escape_string()`** ব্যবহার করবেন না — এগুলো পুরোনো ও অনিরাপদ পদ্ধতি
- **String concatenation** দিয়ে কোয়েরি তৈরি করবেন না
- ইউজার ইনপুট থেকে টেবিল বা কলামের নাম নেওয়া এড়িয়ে চলুন — প্রয়োজন হলে whitelist ব্যবহার করুন
