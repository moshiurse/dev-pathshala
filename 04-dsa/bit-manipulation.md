# 🔢 বিট ম্যানিপুলেশন (Bit Manipulation)

> _"কম্পিউটারের ভাষায় সবকিছুই ০ আর ১ — বিট ম্যানিপুলেশন হলো সেই ভাষায় সরাসরি কথা বলা।"_ ⚡

---

## 📖 সংজ্ঞা

**বিট ম্যানিপুলেশন** হলো ডেটার **বাইনারি রিপ্রেজেন্টেশনে** সরাসরি বিট-লেভেল অপারেশন করা। এটি অত্যন্ত **দ্রুত** কারণ প্রসেসর সরাসরি বিট অপারেশন সাপোর্ট করে — কোনো অতিরিক্ত গণনার দরকার হয় না।

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | বিট ম্যানিপুলেশন-এর সাথে তুলনা |
|---|---|
| লাইটের সুইচ | ON (1) / OFF (0) — প্রতিটি সুইচ = একটি বিট |
| পারমিশন সিস্টেম | rwx = 111 (read+write+execute) |
| ফিচার ফ্ল্যাগ | প্রতিটি ফিচার ON/OFF → বিটমাস্ক |
| রং মিশ্রণ (RGB) | R, G, B প্রতিটি ৮ বিটে → ২৪ বিট রং |
| নেটওয়ার্ক সাবনেট মাস্ক | 255.255.255.0 = বিটমাস্ক |

---

## ১. বিটওয়াইজ অপারেটর

### 📋 অপারেটর টেবিল

| অপারেটর | নাম | উদাহরণ | ফলাফল | ব্যাখ্যা |
|---|---|---|---|---|
| `&` | AND | `5 & 3` | `1` | দুটোই ১ হলে ১ |
| `\|` | OR | `5 \| 3` | `7` | যেকোনো একটি ১ হলে ১ |
| `^` | XOR | `5 ^ 3` | `6` | ভিন্ন হলে ১ |
| `~` | NOT | `~5` | `-6` | বিট উল্টে দেয় |
| `<<` | Left Shift | `5 << 1` | `10` | বামে সরায় (×2) |
| `>>` | Right Shift | `5 >> 1` | `2` | ডানে সরায় (÷2) |

### 🔍 বিস্তারিত উদাহরণ

```
5 = 0101 (বাইনারি)
3 = 0011

AND (&):    0101        OR (|):     0101        XOR (^):    0101
            0011                    0011                    0011
            ----                    ----                    ----
            0001 = 1                0111 = 7                0110 = 6

NOT (~):    ~0101 = 1010... = -6 (Two's complement)

Left Shift:  0101 << 1 = 1010 = 10  (5 × 2 = 10)
Right Shift: 0101 >> 1 = 0010 = 2   (5 ÷ 2 = 2)
```

---

## ২. সাধারণ বিট ট্রিকস

### ✅ জোড়/বিজোড় চেক (Check Odd/Even)

**কৌশল**: শেষ বিট ১ হলে বিজোড়, ০ হলে জোড়।

```php
<?php
function isOdd(int $n): bool {
    return ($n & 1) === 1;
}

echo isOdd(7) ? "বিজোড়" : "জোড়"; // বিজোড়
echo isOdd(4) ? "বিজোড়" : "জোড়"; // জোড়

// কেন কাজ করে?
// 7 = 0111 → শেষ বিট ১ → বিজোড়
// 4 = 0100 → শেষ বিট ০ → জোড়
```

```javascript
function isOdd(n) {
    return (n & 1) === 1;
}

console.log(isOdd(7)); // true
console.log(isOdd(4)); // false
```

---

### 🔄 XOR দিয়ে Swap (কোনো অতিরিক্ত ভ্যারিয়েবল ছাড়া)

```php
<?php
$a = 10; // 1010
$b = 7;  // 0111

$a = $a ^ $b; // a = 1101 (13)
$b = $a ^ $b; // b = 1010 (10) ← আগের a
$a = $a ^ $b; // a = 0111 (7)  ← আগের b

echo "a = $a, b = $b"; // a = 7, b = 10
```

```javascript
let a = 10, b = 7;

a = a ^ b;
b = a ^ b;
a = a ^ b;

console.log(`a = ${a}, b = ${b}`); // a = 7, b = 10
```

---

### ⚡ 2-এর পাওয়ার চেক (Power of 2)

**কৌশল**: `n & (n - 1)` যদি ০ হয়, তাহলে n হলো 2-এর পাওয়ার।

```
কেন কাজ করে?
8  = 1000        7  = 0111        8 & 7 = 0000 ✓ (2-এর পাওয়ার)
6  = 0110        5  = 0101        6 & 5 = 0100 ✗ (2-এর পাওয়ার নয়)

2-এর পাওয়ারে শুধু একটি বিট ১ থাকে!
```

```php
<?php
function isPowerOfTwo(int $n): bool {
    return $n > 0 && ($n & ($n - 1)) === 0;
}

echo isPowerOfTwo(16) ? "হ্যাঁ" : "না"; // হ্যাঁ (10000)
echo isPowerOfTwo(12) ? "হ্যাঁ" : "না"; // না (1100)
```

```javascript
function isPowerOfTwo(n) {
    return n > 0 && (n & (n - 1)) === 0;
}

console.log(isPowerOfTwo(16)); // true
console.log(isPowerOfTwo(12)); // false
```

---

### 🔢 নির্দিষ্ট বিট সেট/ক্লিয়ার/টগল

```php
<?php
$flags = 0b0000; // ০০০০

// ২য় বিট সেট করো (SET)
$flags |= (1 << 2);  // 0100 = 4
echo decbin($flags) . "\n"; // 100

// ১ম বিটও সেট করো
$flags |= (1 << 1);  // 0110 = 6
echo decbin($flags) . "\n"; // 110

// ২য় বিট ক্লিয়ার করো (CLEAR)
$flags &= ~(1 << 2); // 0010 = 2
echo decbin($flags) . "\n"; // 10

// ১ম বিট টগল করো (TOGGLE)
$flags ^= (1 << 1);  // 0000 = 0
echo decbin($flags) . "\n"; // 0

// নির্দিষ্ট বিট চেক করো (CHECK)
$flags = 0b1010;
$isSet = ($flags >> 3) & 1; // ৩য় বিট চেক → ১
echo "৩য় বিট: $isSet\n"; // 1
```

```javascript
let flags = 0b0000;

flags |= (1 << 2);   // SET 2nd bit → 0100
flags |= (1 << 1);   // SET 1st bit → 0110
flags &= ~(1 << 2);  // CLEAR 2nd bit → 0010
flags ^= (1 << 1);   // TOGGLE 1st bit → 0000

// চেক
flags = 0b1010;
console.log((flags >> 3) & 1); // ৩য় বিট → 1
console.log((flags >> 2) & 1); // ২য় বিট → 0
```

---

## ৩. প্র্যাকটিক্যাল ব্যবহার — পারমিশন সিস্টেম

```php
<?php
// ফিচার ফ্ল্যাগ বা পারমিশন সিস্টেম
const READ    = 1 << 0; // 001 = 1
const WRITE   = 1 << 1; // 010 = 2
const EXECUTE = 1 << 2; // 100 = 4

// পারমিশন দাও
$userPermission = READ | WRITE; // 011 = 3

// পারমিশন চেক
function hasPermission(int $userPerm, int $perm): bool {
    return ($userPerm & $perm) === $perm;
}

echo hasPermission($userPermission, READ) ? "পড়তে পারবে" : "না";      // হ্যাঁ
echo hasPermission($userPermission, EXECUTE) ? "চালাতে পারবে" : "না";   // না

// পারমিশন যোগ করো
$userPermission |= EXECUTE; // 111 = 7

// পারমিশন বাতিল করো
$userPermission &= ~WRITE;  // 101 = 5
```

```javascript
const Permission = {
    READ:    1 << 0, // 001
    WRITE:   1 << 1, // 010
    EXECUTE: 1 << 2, // 100
    ADMIN:   1 << 3, // 1000
};

let userPerm = Permission.READ | Permission.WRITE; // 011

const hasPermission = (userPerm, perm) => (userPerm & perm) === perm;

console.log(hasPermission(userPerm, Permission.READ));    // true
console.log(hasPermission(userPerm, Permission.ADMIN));   // false

userPerm |= Permission.ADMIN;   // অ্যাডমিন যোগ
userPerm &= ~Permission.WRITE;  // লেখার অনুমতি বাতিল
```

---

## ৪. একটি অনন্য সংখ্যা খোঁজা (Single Number)

**সমস্যা**: একটি অ্যারেতে প্রতিটি সংখ্যা দুইবার আছে, শুধু একটি সংখ্যা একবার। সেটি বের করো।

**কৌশল**: `a ^ a = 0` এবং `a ^ 0 = a` — তাই সব XOR করলে জোড়া বাতিল হয়ে যায়!

```php
<?php
function singleNumber(array $nums): int {
    $result = 0;
    foreach ($nums as $num) {
        $result ^= $num;
    }
    return $result;
}

echo singleNumber([2, 3, 5, 3, 2]); // 5
// 2^3^5^3^2 = (2^2)^(3^3)^5 = 0^0^5 = 5
```

```javascript
function singleNumber(nums) {
    return nums.reduce((result, num) => result ^ num, 0);
}

console.log(singleNumber([4, 1, 2, 1, 2])); // 4
```

---

## ৫. বিট গণনা (Count Set Bits — Brian Kernighan)

```php
<?php
function countSetBits(int $n): int {
    $count = 0;
    while ($n > 0) {
        $n &= ($n - 1); // সবচেয়ে ডানের ১ বিট মুছে দেয়
        $count++;
    }
    return $count;
}

echo countSetBits(13); // 3 (1101 → তিনটি ১)
echo countSetBits(7);  // 3 (0111 → তিনটি ১)
```

```javascript
function countSetBits(n) {
    let count = 0;
    while (n > 0) {
        n &= (n - 1);
        count++;
    }
    return count;
}

console.log(countSetBits(13)); // 3
console.log(countSetBits(255)); // 8 (11111111)
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন বিট ম্যানিপুলেশন |
|---|---|
| জোড়/বিজোড় চেক | `n & 1` — modulo-র চেয়ে দ্রুত |
| ফ্ল্যাগ/পারমিশন সিস্টেম | বিটমাস্কে একাধিক ফ্ল্যাগ একটি সংখ্যায় |
| ২-এর গুণিতক/ভাগ | শিফট অপারেশন (<<, >>) |
| ক্রিপ্টোগ্রাফি | XOR-ভিত্তিক এনক্রিপশন |
| নেটওয়ার্কিং | সাবনেট মাস্ক, IP গণনা |
| গেম ডেভেলপমেন্ট | স্টেট ম্যানেজমেন্ট, কলিশন ফ্ল্যাগ |
| কম্পিটিটিভ প্রোগ্রামিং | অপটিমাইজেশন ট্রিকস |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না |
|---|---|
| কোড রিডেবিলিটি গুরুত্বপূর্ণ | বিট অপারেশন কম পাঠযোগ্য |
| ফ্লোটিং পয়েন্ট সংখ্যা | বিটওয়াইজ ইন্টিজারে কাজ করে |
| পারফরম্যান্স critical না | সাধারণ অপারেটর ব্যবহার করো |
| টিমের সবাই বিট বোঝে না | মেইনটেনেন্স কঠিন হবে |

---

## 🧠 মনে রাখার সূত্র

```
জোড়/বিজোড়:     n & 1
2-এর পাওয়ার:    n & (n-1) === 0
বিট সেট:         n | (1 << pos)
বিট ক্লিয়ার:      n & ~(1 << pos)
বিট টগল:         n ^ (1 << pos)
বিট চেক:         (n >> pos) & 1
×2:              n << 1
÷2:              n >> 1
সবচেয়ে ডানের ১:  n & (-n)
ডানের ১ মুছো:    n & (n-1)
XOR ম্যাজিক:     a^a=0, a^0=a
```
