# 🗃️ অ্যারে ও স্ট্রিং (Array & String)

> _"ডাটা স্ট্রাকচারের ভিত্তিপ্রস্তর হলো অ্যারে — এটি না বুঝলে বাকি সব শূন্যের উপর দাঁড়ানো।"_

---

## 📖 সংজ্ঞা

**অ্যারে (Array)** হলো একই ধরনের ডাটা এলিমেন্টের একটি সিকোয়েন্সিয়াল কালেকশন যা মেমোরিতে পাশাপাশি (contiguous) অবস্থান দখল করে। প্রতিটি এলিমেন্ট একটি **ইনডেক্স** দ্বারা সরাসরি অ্যাক্সেসযোগ্য।

**স্ট্রিং (String)** হলো ক্যারেক্টারের একটি সিকোয়েন্স — মূলত এটি একটি ক্যারেক্টার অ্যারে। তবে বিভিন্ন প্রোগ্রামিং ভাষায় স্ট্রিং-এর আচরণ ভিন্ন — কোথাও immutable, কোথাও mutable।

অ্যারে ও স্ট্রিং হলো **সবচেয়ে মৌলিক ডাটা স্ট্রাকচার** এবং প্রায় প্রতিটি কোডিং ইন্টারভিউতে এগুলো থেকে প্রশ্ন আসে। এই গাইডে আমরা গভীরভাবে অ্যারে ও স্ট্রিং-এর ধারণা, টেকনিক, এবং জটিল সমস্যার সমাধান শিখবো।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### অ্যারে = ট্রেনের বগি

একটি ট্রেনের কথা ভাবুন:
- প্রতিটি **বগি** হলো একটি এলিমেন্ট
- বগিগুলো **পাশাপাশি** সংযুক্ত (contiguous memory)
- প্রতিটি বগির একটি **নম্বর** আছে (index)
- আপনি সরাসরি "বগি নং ৫"-এ যেতে পারেন (O(1) access)
- কিন্তু মাঝখানে নতুন বগি যোগ করতে গেলে বাকিগুলো সরাতে হয় (O(n) insert)

### স্ট্রিং = পুঁতির মালা

একটি পুঁতির মালার কথা ভাবুন:
- প্রতিটি **পুঁতি** হলো একটি ক্যারেক্টার
- পুঁতিগুলো একটি **নির্দিষ্ট ক্রমে** সাজানো
- মালা ভাঙলে (modify) নতুন মালা বানাতে হয় (immutable string in JS)
- PHP-তে আপনি একটি পুঁতি বদলাতে পারেন (mutable string)

---

# 📦 ১. অ্যারে (Array) — মৌলিক ধারণা

## মেমোরিতে অ্যারে কিভাবে সংরক্ষিত হয়

অ্যারে মেমোরিতে **contiguous (ধারাবাহিক)** ব্লকে সংরক্ষিত হয়। প্রতিটি এলিমেন্ট সমান আকারের মেমোরি দখল করে।

```
মেমোরি অ্যাড্রেস:  1000   1004   1008   1012   1016   1020
                  ┌──────┬──────┬──────┬──────┬──────┬──────┐
  অ্যারে []:      │  10  │  20  │  30  │  40  │  50  │  60  │
                  └──────┴──────┴──────┴──────┴──────┴──────┘
  ইনডেক্স:          0      1      2      3      4      5

  বেস অ্যাড্রেস = 1000
  প্রতিটি এলিমেন্টের আকার = 4 bytes (int)

  সূত্র: address(i) = base_address + (i × element_size)
  উদাহরণ: address(3) = 1000 + (3 × 4) = 1012 → মান 40 ✓
```

এই সূত্রের কারণেই অ্যারেতে **যেকোনো ইনডেক্সে O(1)** সময়ে অ্যাক্সেস করা যায়!

---

## Static vs Dynamic Array

```
┌──────────────────────────────────────────────────────────────┐
│              Static Array vs Dynamic Array                    │
├──────────────────────────────┬───────────────────────────────┤
│      Static Array            │      Dynamic Array             │
├──────────────────────────────┼───────────────────────────────┤
│ আকার নির্দিষ্ট (fixed)       │ আকার পরিবর্তনশীল (resizable) │
│ কম্পাইল টাইমে আকার ঠিক হয়  │ রানটাইমে আকার বাড়ে           │
│ মেমোরি অপচয় হতে পারে       │ মেমোরি দক্ষ ব্যবহার           │
│ দ্রুত (কোনো resize নেই)     │ resize-এ O(n) লাগতে পারে     │
│ C/C++ int arr[10]            │ PHP array, JS Array            │
└──────────────────────────────┴───────────────────────────────┘
```

### Dynamic Array-এর Resize প্রক্রিয়া (Amortized Analysis)

```
ধাপ ১: capacity = 4, size = 4  → অ্যারে পূর্ণ!
┌───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │
└───┴───┴───┴───┘

ধাপ ২: নতুন capacity = 8 (দ্বিগুণ), পুরনো ডাটা কপি
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘

ধাপ ৩: নতুন এলিমেন্ট যোগ
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘

→ Amortized O(1) per insertion
```

---

## অ্যারে অপারেশন ও কমপ্লেক্সিটি টেবিল

| অপারেশন | টাইম কমপ্লেক্সিটি | ব্যাখ্যা |
|---------|-------------------|---------|
| Access (arr[i]) | **O(1)** | সরাসরি ইনডেক্স দিয়ে অ্যাক্সেস |
| Search (unsorted) | **O(n)** | প্রতিটি এলিমেন্ট চেক করতে হয় |
| Search (sorted) | **O(log n)** | বাইনারি সার্চ ব্যবহার করা যায় |
| Insert (শেষে) | **O(1)** amortized | Dynamic array-তে শেষে যোগ |
| Insert (শুরুতে/মাঝে) | **O(n)** | এলিমেন্ট শিফট করতে হয় |
| Delete (শেষে) | **O(1)** | শুধু size কমানো |
| Delete (শুরুতে/মাঝে) | **O(n)** | এলিমেন্ট শিফট করতে হয় |
| Resize | **O(n)** | পুরো অ্যারে কপি করতে হয় |

---

## PHP Array Internals

PHP-তে অ্যারে আসলে একটি **ordered hash table**। এটি C-এর সাধারণ অ্যারের মতো নয়।

```
PHP অ্যারের অভ্যন্তরীণ গঠন (Hash Table):

┌─────────────┐
│  Hash Table │
├─────────────┤     ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Bucket 0   │────▶│ key: 0  │───▶│ key: 4  │───▶│  NULL   │
│  Bucket 1   │     │ val: 10 │    │ val: 50 │    └─────────┘
│  Bucket 2   │     └─────────┘    └─────────┘
│  ...        │
│  Bucket n   │
└─────────────┘

→ তাই PHP array-তে key অ্যাক্সেস গড়ে O(1), worst case O(n)
→ PHP array-তে integer ও string key মিশ্রিত থাকতে পারে
```

### PHP অ্যারের মৌলিক অপারেশন

```php
<?php
// === PHP অ্যারে — মৌলিক অপারেশন ===

// ১. অ্যারে তৈরি
$numbers = [10, 20, 30, 40, 50];

// ২. অ্যাক্সেস — O(1)
echo $numbers[2]; // 30

// ৩. সার্চ — O(n)
$target = 30;
$found = false;
foreach ($numbers as $index => $value) {
    if ($value === $target) {
        echo "পাওয়া গেছে ইনডেক্স: $index\n";
        $found = true;
        break;
    }
}

// ৪. শেষে ইনসার্ট — O(1) amortized
$numbers[] = 60; // [10, 20, 30, 40, 50, 60]

// ৫. নির্দিষ্ট অবস্থানে ইনসার্ট — O(n)
array_splice($numbers, 2, 0, [25]); // ইনডেক্স 2-তে 25 ঢোকানো

// ৬. ডিলিট — O(n) (ইনডেক্স পুনর্গঠন)
unset($numbers[3]);
$numbers = array_values($numbers); // ইনডেক্স রিসেট

// ৭. অ্যারের আকার — O(1)
echo count($numbers);
```

### JavaScript অ্যারের মৌলিক অপারেশন

```javascript
// === JavaScript অ্যারে — মৌলিক অপারেশন ===

// ১. অ্যারে তৈরি
const numbers = [10, 20, 30, 40, 50];

// ২. অ্যাক্সেস — O(1)
console.log(numbers[2]); // 30

// ৩. সার্চ — O(n)
const target = 30;
const index = numbers.indexOf(target);
if (index !== -1) {
    console.log(`পাওয়া গেছে ইনডেক্স: ${index}`);
}

// ৪. শেষে ইনসার্ট — O(1) amortized
numbers.push(60); // [10, 20, 30, 40, 50, 60]

// ৫. নির্দিষ্ট অবস্থানে ইনসার্ট — O(n)
numbers.splice(2, 0, 25); // ইনডেক্স 2-তে 25 ঢোকানো

// ৬. ডিলিট — O(n)
numbers.splice(3, 1); // ইনডেক্স 3 থেকে ১টি মুছুন

// ৭. অ্যারের আকার — O(1)
console.log(numbers.length);
```

---

# 🔧 ২. অ্যারে টেকনিক সমূহ

## ২.১ Two Pointer Technique (টু পয়েন্টার টেকনিক)

**ধারণা:** দুটি পয়েন্টার ব্যবহার করে অ্যারের দুই প্রান্ত বা দুটি অবস্থান থেকে একসাথে কাজ করা।

**কখন ব্যবহার করবেন:**
- Sorted অ্যারেতে pair খোঁজা
- অ্যারে reverse করা
- Duplicate মুছে ফেলা
- Container with most water

### সমস্যা: Sorted Array-তে Pair Sum খোঁজা

> একটি sorted অ্যারে দেওয়া আছে। এমন দুটি সংখ্যা খুঁজুন যাদের যোগফল একটি নির্দিষ্ট target-এর সমান।

```
অ্যারে: [1, 3, 5, 7, 9, 11]     target = 12

ধাপ ১: left=0, right=5 → 1+11=12 ✓ পাওয়া গেছে!
        L                    R
       [1,  3,  5,  7,  9,  11]

যদি যোগফল < target → left++ (বড় সংখ্যা দরকার)
যদি যোগফল > target → right-- (ছোট সংখ্যা দরকার)
যদি যোগফল = target → উত্তর পাওয়া গেছে!

ASCII ট্রেস (target = 10):
ধাপ ১: left=0, right=5 → 1+11=12 > 10  → right--
ধাপ ২: left=0, right=4 → 1+9=10  = 10  ✓ পাওয়া!
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস

### PHP (Two Pointer — Pair Sum)

```php
<?php
/**
 * Sorted অ্যারেতে target sum-এর pair খোঁজা
 * টাইম: O(n), স্পেস: O(1)
 */
function twoSumSorted(array $arr, int $target): array
{
    $left = 0;
    $right = count($arr) - 1;

    while ($left < $right) {
        $sum = $arr[$left] + $arr[$right];

        if ($sum === $target) {
            return [$left, $right]; // pair পাওয়া গেছে
        } elseif ($sum < $target) {
            $left++; // যোগফল ছোট, বাম পয়েন্টার এগিয়ে নাও
        } else {
            $right--; // যোগফল বড়, ডান পয়েন্টার পিছিয়ে নাও
        }
    }

    return []; // pair নেই
}

// পরীক্ষা
$arr = [1, 3, 5, 7, 9, 11];
$result = twoSumSorted($arr, 10);
echo "ইনডেক্স: [{$result[0]}, {$result[1]}]\n"; // [0, 4] → 1+9=10
```

### JavaScript (Two Pointer — Pair Sum)

```javascript
/**
 * Sorted অ্যারেতে target sum-এর pair খোঁজা
 * টাইম: O(n), স্পেস: O(1)
 */
function twoSumSorted(arr, target) {
    let left = 0;
    let right = arr.length - 1;

    while (left < right) {
        const sum = arr[left] + arr[right];

        if (sum === target) {
            return [left, right]; // pair পাওয়া গেছে
        } else if (sum < target) {
            left++; // যোগফল ছোট, বাম পয়েন্টার এগিয়ে নাও
        } else {
            right--; // যোগফল বড়, ডান পয়েন্টার পিছিয়ে নাও
        }
    }

    return []; // pair নেই
}

// পরীক্ষা
const arr = [1, 3, 5, 7, 9, 11];
const result = twoSumSorted(arr, 10);
console.log(`ইনডেক্স: [${result[0]}, ${result[1]}]`); // [0, 4]
```

### সমস্যা: Container With Most Water

> n-টি উচ্চতা দেওয়া আছে। দুটি রেখা বেছে নিন যেন সবচেয়ে বেশি পানি ধরে।

```
উচ্চতা: [1, 8, 6, 2, 5, 4, 8, 3, 7]

    8 |   █           █
    7 |   █           █       █
    6 |   █   █       █       █
    5 |   █   █   █   █       █
    4 |   █   █   █ █ █       █
    3 |   █   █   █ █ █ █     █
    2 |   █   █ █ █ █ █ █     █
    1 | █ █   █ █ █ █ █ █     █
      └──────────────────────────
        0 1 2 3 4 5 6 7 8

    area = min(height[L], height[R]) × (R - L)
    সবচেয়ে ছোট উচ্চতার পয়েন্টার সরাও
```

### PHP (Container With Most Water)

```php
<?php
/**
 * Container With Most Water
 * টাইম: O(n), স্পেস: O(1)
 */
function maxArea(array $height): int
{
    $left = 0;
    $right = count($height) - 1;
    $maxWater = 0;

    while ($left < $right) {
        // ক্ষেত্রফল = ছোট উচ্চতা × দূরত্ব
        $width = $right - $left;
        $h = min($height[$left], $height[$right]);
        $area = $h * $width;
        $maxWater = max($maxWater, $area);

        // ছোট উচ্চতার দিকের পয়েন্টার সরাও
        if ($height[$left] < $height[$right]) {
            $left++;
        } else {
            $right--;
        }
    }

    return $maxWater;
}

// পরীক্ষা
echo maxArea([1, 8, 6, 2, 5, 4, 8, 3, 7]); // 49
```

### JavaScript (Container With Most Water)

```javascript
/**
 * Container With Most Water
 * টাইম: O(n), স্পেস: O(1)
 */
function maxArea(height) {
    let left = 0;
    let right = height.length - 1;
    let maxWater = 0;

    while (left < right) {
        const width = right - left;
        const h = Math.min(height[left], height[right]);
        const area = h * width;
        maxWater = Math.max(maxWater, area);

        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }

    return maxWater;
}

// পরীক্ষা
console.log(maxArea([1, 8, 6, 2, 5, 4, 8, 3, 7])); // 49
```

---

## ২.২ Sliding Window (স্লাইডিং উইন্ডো)

**ধারণা:** একটি নির্দিষ্ট আকারের "উইন্ডো" অ্যারের উপর দিয়ে স্লাইড করানো। প্রতিটি ধাপে উইন্ডোর বাম দিকের এলিমেন্ট বাদ দিয়ে ডান দিকে নতুন এলিমেন্ট যোগ করা হয়।

**কখন ব্যবহার করবেন:**
- Subarray/substring সংক্রান্ত সমস্যা
- Fixed বা variable size window
- Maximum/minimum sum of subarray

### সমস্যা: Maximum Sum Subarray of Size K

> একটি অ্যারে ও K দেওয়া আছে। K আকারের subarray-এর সর্বোচ্চ যোগফল বের করুন।

```
অ্যারে: [2, 1, 5, 1, 3, 2]     K = 3

Brute Force: প্রতিটি K-size subarray-এর যোগফল বের করো → O(n×k)
Sliding Window: উইন্ডো স্লাইড করো → O(n)

ASCII ট্রেস:
ধাপ ১: [2, 1, 5] 1, 3, 2    → sum = 8
         └──────┘
ধাপ ২:  2 [1, 5, 1] 3, 2    → sum = 8-2+1 = 7
            └──────┘
ধাপ ৩:  2, 1 [5, 1, 3] 2    → sum = 7-1+3 = 9  ← সর্বোচ্চ!
               └──────┘
ধাপ ৪:  2, 1, 5 [1, 3, 2]   → sum = 9-5+2 = 6
                  └──────┘

সর্বোচ্চ যোগফল = 9
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস

### PHP (Sliding Window)

```php
<?php
/**
 * K আকারের subarray-এর সর্বোচ্চ যোগফল
 * টাইম: O(n), স্পেস: O(1)
 */
function maxSumSubarray(array $arr, int $k): int
{
    $n = count($arr);
    if ($n < $k) return -1;

    // প্রথম উইন্ডোর যোগফল
    $windowSum = 0;
    for ($i = 0; $i < $k; $i++) {
        $windowSum += $arr[$i];
    }

    $maxSum = $windowSum;

    // উইন্ডো স্লাইড করো
    for ($i = $k; $i < $n; $i++) {
        $windowSum += $arr[$i] - $arr[$i - $k]; // নতুন যোগ, পুরনো বাদ
        $maxSum = max($maxSum, $windowSum);
    }

    return $maxSum;
}

// পরীক্ষা
echo maxSumSubarray([2, 1, 5, 1, 3, 2], 3); // 9
```

### JavaScript (Sliding Window)

```javascript
/**
 * K আকারের subarray-এর সর্বোচ্চ যোগফল
 * টাইম: O(n), স্পেস: O(1)
 */
function maxSumSubarray(arr, k) {
    if (arr.length < k) return -1;

    // প্রথম উইন্ডোর যোগফল
    let windowSum = 0;
    for (let i = 0; i < k; i++) {
        windowSum += arr[i];
    }

    let maxSum = windowSum;

    // উইন্ডো স্লাইড করো
    for (let i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k]; // নতুন যোগ, পুরনো বাদ
        maxSum = Math.max(maxSum, windowSum);
    }

    return maxSum;
}

// পরীক্ষা
console.log(maxSumSubarray([2, 1, 5, 1, 3, 2], 3)); // 9
```

### সমস্যা: Longest Substring Without Repeating Characters

> একটি স্ট্রিং দেওয়া আছে। রিপিটিং ক্যারেক্টার ছাড়া দীর্ঘতম substring-এর দৈর্ঘ্য বের করুন।

```
ইনপুট: "abcabcbb"

Variable-size sliding window ব্যবহার:

  a b c a b c b b
  L
  R
  উইন্ডো: {a}         → দৈর্ঘ্য 1

  a b c a b c b b
  L R
  উইন্ডো: {a,b}       → দৈর্ঘ্য 2

  a b c a b c b b
  L   R
  উইন্ডো: {a,b,c}     → দৈর্ঘ্য 3

  a b c a b c b b
  L     R
  'a' রিপিট! → L সরাও যতক্ষণ না 'a' বের হয়
    L   R
  উইন্ডো: {b,c,a}     → দৈর্ঘ্য 3

  সর্বোচ্চ দৈর্ঘ্য = 3 ("abc")
```

### PHP (Longest Substring Without Repeating)

```php
<?php
/**
 * রিপিটিং ক্যারেক্টার ছাড়া দীর্ঘতম substring
 * টাইম: O(n), স্পেস: O(min(m, n)) — m = ক্যারেক্টার সেটের আকার
 */
function lengthOfLongestSubstring(string $s): int
{
    $charIndex = []; // ক্যারেক্টারের শেষ ইনডেক্স
    $maxLen = 0;
    $left = 0;

    for ($right = 0; $right < strlen($s); $right++) {
        $char = $s[$right];

        // যদি ক্যারেক্টার আগে দেখা গেছে এবং উইন্ডোর মধ্যে আছে
        if (isset($charIndex[$char]) && $charIndex[$char] >= $left) {
            $left = $charIndex[$char] + 1; // বাম পয়েন্টার সরাও
        }

        $charIndex[$char] = $right;
        $maxLen = max($maxLen, $right - $left + 1);
    }

    return $maxLen;
}

// পরীক্ষা
echo lengthOfLongestSubstring("abcabcbb"); // 3
echo lengthOfLongestSubstring("bbbbb");    // 1
echo lengthOfLongestSubstring("pwwkew");   // 3
```

### JavaScript (Longest Substring Without Repeating)

```javascript
/**
 * রিপিটিং ক্যারেক্টার ছাড়া দীর্ঘতম substring
 * টাইম: O(n), স্পেস: O(min(m, n))
 */
function lengthOfLongestSubstring(s) {
    const charIndex = new Map();
    let maxLen = 0;
    let left = 0;

    for (let right = 0; right < s.length; right++) {
        const char = s[right];

        if (charIndex.has(char) && charIndex.get(char) >= left) {
            left = charIndex.get(char) + 1;
        }

        charIndex.set(char, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}

// পরীক্ষা
console.log(lengthOfLongestSubstring("abcabcbb")); // 3
console.log(lengthOfLongestSubstring("bbbbb"));    // 1
console.log(lengthOfLongestSubstring("pwwkew"));   // 3
```

---

## ২.৩ Kadane's Algorithm (কাদানের অ্যালগরিদম)

**ধারণা:** Maximum subarray sum বের করার জন্য সবচেয়ে কার্যকর অ্যালগরিদম। মূল আইডিয়া — প্রতিটি পজিশনে সিদ্ধান্ত নাও: আগের subarray-এ যুক্ত হবো, নাকি নতুন subarray শুরু করবো?

```
সূত্র: currentMax = max(arr[i], currentMax + arr[i])

অ্যারে: [-2, 1, -3, 4, -1, 2, 1, -5, 4]

ASCII ট্রেস:
ইনডেক্স:    0    1    2    3    4    5    6    7    8
মান:       -2    1   -3    4   -1    2    1   -5    4
──────────────────────────────────────────────────────
currentMax: -2    1   -2    4    3    5    6    1    5
globalMax:  -2    1    1    4    4    5    6    6    6
──────────────────────────────────────────────────────

ইনডেক্স 0: max(-2, -∞ + -2) = -2, globalMax = -2
ইনডেক্স 1: max(1, -2 + 1) = max(1, -1) = 1, globalMax = 1
ইনডেক্স 2: max(-3, 1 + -3) = max(-3, -2) = -2, globalMax = 1
ইনডেক্স 3: max(4, -2 + 4) = max(4, 2) = 4, globalMax = 4
           → নতুন subarray শুরু! (4 > -2+4)
ইনডেক্স 4: max(-1, 4 + -1) = max(-1, 3) = 3, globalMax = 4
ইনডেক্স 5: max(2, 3 + 2) = max(2, 5) = 5, globalMax = 5
ইনডেক্স 6: max(1, 5 + 1) = max(1, 6) = 6, globalMax = 6 ← চূড়ান্ত!
ইনডেক্স 7: max(-5, 6 + -5) = max(-5, 1) = 1, globalMax = 6
ইনডেক্স 8: max(4, 1 + 4) = max(4, 5) = 5, globalMax = 6

উত্তর: Maximum Subarray Sum = 6
Subarray: [4, -1, 2, 1] (ইনডেক্স 3 থেকে 6)
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস

### PHP (Kadane's Algorithm)

```php
<?php
/**
 * Kadane's Algorithm — Maximum Subarray Sum
 * টাইম: O(n), স্পেস: O(1)
 */
function maxSubarraySum(array $arr): array
{
    $currentMax = $arr[0];
    $globalMax = $arr[0];
    $start = 0;
    $end = 0;
    $tempStart = 0;

    for ($i = 1; $i < count($arr); $i++) {
        // নতুন subarray শুরু করবো নাকি আগেরটায় যুক্ত হবো?
        if ($arr[$i] > $currentMax + $arr[$i]) {
            $currentMax = $arr[$i];
            $tempStart = $i; // নতুন subarray শুরু
        } else {
            $currentMax = $currentMax + $arr[$i];
        }

        if ($currentMax > $globalMax) {
            $globalMax = $currentMax;
            $start = $tempStart;
            $end = $i;
        }
    }

    return [
        'maxSum'   => $globalMax,
        'subarray' => array_slice($arr, $start, $end - $start + 1),
        'start'    => $start,
        'end'      => $end,
    ];
}

// পরীক্ষা
$result = maxSubarraySum([-2, 1, -3, 4, -1, 2, 1, -5, 4]);
echo "সর্বোচ্চ যোগফল: {$result['maxSum']}\n"; // 6
echo "Subarray: [" . implode(', ', $result['subarray']) . "]\n"; // [4, -1, 2, 1]
```

### JavaScript (Kadane's Algorithm)

```javascript
/**
 * Kadane's Algorithm — Maximum Subarray Sum
 * টাইম: O(n), স্পেস: O(1)
 */
function maxSubarraySum(arr) {
    let currentMax = arr[0];
    let globalMax = arr[0];
    let start = 0, end = 0, tempStart = 0;

    for (let i = 1; i < arr.length; i++) {
        if (arr[i] > currentMax + arr[i]) {
            currentMax = arr[i];
            tempStart = i;
        } else {
            currentMax = currentMax + arr[i];
        }

        if (currentMax > globalMax) {
            globalMax = currentMax;
            start = tempStart;
            end = i;
        }
    }

    return {
        maxSum: globalMax,
        subarray: arr.slice(start, end + 1),
        start,
        end,
    };
}

// পরীক্ষা
const result = maxSubarraySum([-2, 1, -3, 4, -1, 2, 1, -5, 4]);
console.log(`সর্বোচ্চ যোগফল: ${result.maxSum}`); // 6
console.log(`Subarray: [${result.subarray}]`);     // [4, -1, 2, 1]
```

---

## ২.৪ Prefix Sum (প্রিফিক্স সাম)

**ধারণা:** অ্যারের প্রতিটি অবস্থান পর্যন্ত ক্রমযোগফল আগে থেকে হিসাব করে রাখা, যাতে যেকোনো range-এর যোগফল O(1)-এ বের করা যায়।

```
অ্যারে:       [2, 4, 1, 3, 5, 2]
প্রিফিক্স:    [2, 6, 7, 10, 15, 17]

                0    1    2    3    4    5
মূল অ্যারে:  [ 2  , 4  , 1  , 3  , 5  , 2  ]
prefix[i]:   [ 2  , 6  , 7  , 10 , 15 , 17 ]
              ↑    ↑    ↑    ↑    ↑    ↑
              2   2+4  6+1  7+3  10+5 15+2

Range Sum Query:
sum(i, j) = prefix[j] - prefix[i-1]  (i > 0)
sum(i, j) = prefix[j]                 (i = 0)

উদাহরণ: sum(1, 4) = prefix[4] - prefix[0] = 15 - 2 = 13
পরীক্ষা: 4 + 1 + 3 + 5 = 13 ✓
```

**কমপ্লেক্সিটি:**
- Preprocessing: O(n)
- Query: O(1)
- স্পেস: O(n)

### PHP (Prefix Sum — Range Query)

```php
<?php
/**
 * Prefix Sum দিয়ে Range Sum Query
 * Preprocessing: O(n), Query: O(1)
 */
class PrefixSum
{
    private array $prefix;

    public function __construct(array $arr)
    {
        // প্রিফিক্স অ্যারে তৈরি
        $this->prefix = [];
        $this->prefix[0] = $arr[0];

        for ($i = 1; $i < count($arr); $i++) {
            $this->prefix[$i] = $this->prefix[$i - 1] + $arr[$i];
        }
    }

    /** i থেকে j পর্যন্ত range-এর যোগফল — O(1) */
    public function rangeSum(int $i, int $j): int
    {
        if ($i === 0) {
            return $this->prefix[$j];
        }
        return $this->prefix[$j] - $this->prefix[$i - 1];
    }
}

// পরীক্ষা
$ps = new PrefixSum([2, 4, 1, 3, 5, 2]);
echo $ps->rangeSum(1, 4) . "\n"; // 13 (4+1+3+5)
echo $ps->rangeSum(0, 2) . "\n"; // 7  (2+4+1)
echo $ps->rangeSum(2, 5) . "\n"; // 11 (1+3+5+2)
```

### JavaScript (Prefix Sum — Range Query)

```javascript
/**
 * Prefix Sum দিয়ে Range Sum Query
 * Preprocessing: O(n), Query: O(1)
 */
class PrefixSum {
    constructor(arr) {
        this.prefix = [arr[0]];
        for (let i = 1; i < arr.length; i++) {
            this.prefix[i] = this.prefix[i - 1] + arr[i];
        }
    }

    /** i থেকে j পর্যন্ত range-এর যোগফল — O(1) */
    rangeSum(i, j) {
        if (i === 0) return this.prefix[j];
        return this.prefix[j] - this.prefix[i - 1];
    }
}

// পরীক্ষা
const ps = new PrefixSum([2, 4, 1, 3, 5, 2]);
console.log(ps.rangeSum(1, 4)); // 13
console.log(ps.rangeSum(0, 2)); // 7
console.log(ps.rangeSum(2, 5)); // 11
```

### সমস্যা: Subarray Sum Equals K

> একটি অ্যারে ও K দেওয়া আছে। কতগুলো continuous subarray-এর যোগফল K-এর সমান?

```
ইনপুট: [1, 1, 1], K = 2
আউটপুট: 2 → [1,1] (ইনডেক্স 0-1) এবং [1,1] (ইনডেক্স 1-2)

আইডিয়া: prefix[j] - prefix[i] = K → prefix[i] = prefix[j] - K
হ্যাশম্যাপে prefix sum-এর frequency রাখো
```

### PHP (Subarray Sum Equals K)

```php
<?php
/**
 * Subarray Sum Equals K — Prefix Sum + HashMap
 * টাইম: O(n), স্পেস: O(n)
 */
function subarraySum(array $nums, int $k): int
{
    $count = 0;
    $prefixSum = 0;
    $map = [0 => 1]; // বেস কেস: prefix sum 0 একবার আছে

    foreach ($nums as $num) {
        $prefixSum += $num;
        $target = $prefixSum - $k;

        // এই target কি আগে prefix sum হিসেবে দেখা গেছে?
        if (isset($map[$target])) {
            $count += $map[$target];
        }

        // বর্তমান prefix sum রেকর্ড করো
        $map[$prefixSum] = ($map[$prefixSum] ?? 0) + 1;
    }

    return $count;
}

// পরীক্ষা
echo subarraySum([1, 1, 1], 2) . "\n";     // 2
echo subarraySum([1, 2, 3], 3) . "\n";     // 2 → [1,2] ও [3]
echo subarraySum([1, -1, 0], 0) . "\n";    // 3
```

### JavaScript (Subarray Sum Equals K)

```javascript
/**
 * Subarray Sum Equals K — Prefix Sum + HashMap
 * টাইম: O(n), স্পেস: O(n)
 */
function subarraySum(nums, k) {
    let count = 0;
    let prefixSum = 0;
    const map = new Map([[0, 1]]); // বেস কেস

    for (const num of nums) {
        prefixSum += num;
        const target = prefixSum - k;

        if (map.has(target)) {
            count += map.get(target);
        }

        map.set(prefixSum, (map.get(prefixSum) || 0) + 1);
    }

    return count;
}

// পরীক্ষা
console.log(subarraySum([1, 1, 1], 2));  // 2
console.log(subarraySum([1, 2, 3], 3));  // 2
console.log(subarraySum([1, -1, 0], 0)); // 3
```

---

# 🧩 ৩. অ্যারে সমস্যা (Complex Problems)

## ৩.১ Two Sum

> একটি অ্যারে ও target দেওয়া আছে। দুটি সংখ্যার ইনডেক্স খুঁজুন যাদের যোগফল target-এর সমান।

### Approach ১: Brute Force — O(n²)

```
প্রতিটি pair চেক করো:
[2, 7, 11, 15]  target = 9

(2,7) → 9 ✓ → উত্তর [0, 1]
```

### Approach ২: Hash Map — O(n)

```
প্রতিটি সংখ্যার complement (target - num) হ্যাশম্যাপে খোঁজো:

ইনডেক্স 0: num=2, need=9-2=7, map={} → 7 নেই, map-এ 2 রাখো → {2:0}
ইনডেক্স 1: num=7, need=9-7=2, map={2:0} → 2 আছে! → উত্তর [0, 1]
```

### PHP (Two Sum)

```php
<?php
/**
 * Two Sum — Brute Force
 * টাইম: O(n²), স্পেস: O(1)
 */
function twoSumBrute(array $nums, int $target): array
{
    $n = count($nums);
    for ($i = 0; $i < $n; $i++) {
        for ($j = $i + 1; $j < $n; $j++) {
            if ($nums[$i] + $nums[$j] === $target) {
                return [$i, $j];
            }
        }
    }
    return [];
}

/**
 * Two Sum — Hash Map (অপটিমাইজড)
 * টাইম: O(n), স্পেস: O(n)
 */
function twoSum(array $nums, int $target): array
{
    $map = []; // মান → ইনডেক্স

    foreach ($nums as $i => $num) {
        $complement = $target - $num;

        if (isset($map[$complement])) {
            return [$map[$complement], $i]; // পাওয়া গেছে!
        }

        $map[$num] = $i; // বর্তমান সংখ্যা রেকর্ড
    }

    return [];
}

// পরীক্ষা
print_r(twoSum([2, 7, 11, 15], 9)); // [0, 1]
print_r(twoSum([3, 2, 4], 6));      // [1, 2]
```

### JavaScript (Two Sum)

```javascript
/**
 * Two Sum — Hash Map
 * টাইম: O(n), স্পেস: O(n)
 */
function twoSum(nums, target) {
    const map = new Map();

    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];

        if (map.has(complement)) {
            return [map.get(complement), i];
        }

        map.set(nums[i], i);
    }

    return [];
}

// পরীক্ষা
console.log(twoSum([2, 7, 11, 15], 9)); // [0, 1]
console.log(twoSum([3, 2, 4], 6));      // [1, 2]
```

---

## ৩.২ Three Sum

> একটি অ্যারে দেওয়া আছে। তিনটি সংখ্যা খুঁজুন যাদের যোগফল ০। (duplicate triplet বাদ দিতে হবে)

```
ইনপুট: [-1, 0, 1, 2, -1, -4]

পদ্ধতি: Sort + Two Pointer
১. অ্যারে সর্ট করো: [-4, -1, -1, 0, 1, 2]
২. প্রতিটি এলিমেন্টের জন্য বাকিটায় two pointer চালাও

ট্রেস (i=0, nums[i]=-4):
  [-4, -1, -1, 0, 1, 2]
    i       L         R     → -4+(-1)+2 = -3 < 0 → L++
    i          L      R     → -4+0+2 = -2 < 0 → L++
    i             L   R     → -4+1+2 = -1 < 0 → L++
    ... কোনো triplet নেই

ট্রেস (i=1, nums[i]=-1):
  [-4, -1, -1, 0, 1, 2]
        i   L         R    → -1+(-1)+2 = 0 ✓ → triplet: [-1,-1,2]
        i      L   R       → -1+0+1 = 0 ✓ → triplet: [-1,0,1]

উত্তর: [[-1,-1,2], [-1,0,1]]
```

**কমপ্লেক্সিটি:** O(n²) টাইম, O(1) স্পেস (আউটপুট ছাড়া)

### PHP (Three Sum)

```php
<?php
/**
 * Three Sum — Sorting + Two Pointer
 * টাইম: O(n²), স্পেস: O(1) আউটপুট ছাড়া
 */
function threeSum(array $nums): array
{
    sort($nums); // O(n log n)
    $result = [];
    $n = count($nums);

    for ($i = 0; $i < $n - 2; $i++) {
        // duplicate এড়ানো
        if ($i > 0 && $nums[$i] === $nums[$i - 1]) continue;

        $left = $i + 1;
        $right = $n - 1;

        while ($left < $right) {
            $sum = $nums[$i] + $nums[$left] + $nums[$right];

            if ($sum === 0) {
                $result[] = [$nums[$i], $nums[$left], $nums[$right]];

                // duplicate এড়ানো
                while ($left < $right && $nums[$left] === $nums[$left + 1]) $left++;
                while ($left < $right && $nums[$right] === $nums[$right - 1]) $right--;

                $left++;
                $right--;
            } elseif ($sum < 0) {
                $left++;
            } else {
                $right--;
            }
        }
    }

    return $result;
}

// পরীক্ষা
print_r(threeSum([-1, 0, 1, 2, -1, -4]));
// [[-1, -1, 2], [-1, 0, 1]]
```

### JavaScript (Three Sum)

```javascript
/**
 * Three Sum — Sorting + Two Pointer
 * টাইম: O(n²), স্পেস: O(1) আউটপুট ছাড়া
 */
function threeSum(nums) {
    nums.sort((a, b) => a - b);
    const result = [];

    for (let i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] === nums[i - 1]) continue;

        let left = i + 1;
        let right = nums.length - 1;

        while (left < right) {
            const sum = nums[i] + nums[left] + nums[right];

            if (sum === 0) {
                result.push([nums[i], nums[left], nums[right]]);
                while (left < right && nums[left] === nums[left + 1]) left++;
                while (left < right && nums[right] === nums[right - 1]) right--;
                left++;
                right--;
            } else if (sum < 0) {
                left++;
            } else {
                right--;
            }
        }
    }

    return result;
}

// পরীক্ষা
console.log(threeSum([-1, 0, 1, 2, -1, -4]));
// [[-1, -1, 2], [-1, 0, 1]]
```

---

## ৩.৩ Merge Intervals

> interval-এর একটি অ্যারে দেওয়া আছে। overlapping interval গুলো merge করুন।

```
ইনপুট: [[1,3], [2,6], [8,10], [15,18]]

ধাপ ১: start দিয়ে সর্ট (ইতিমধ্যে sorted)
ধাপ ২: পাশাপাশি interval চেক করো

  1---3
    2------6
               8---10
                        15---18

  overlap: [1,3] ও [2,6] → merge → [1,6]
  no overlap: [1,6] ও [8,10]
  no overlap: [8,10] ও [15,18]

আউটপুট: [[1,6], [8,10], [15,18]]
```

**কমপ্লেক্সিটি:** O(n log n) টাইম (sorting), O(n) স্পেস

### PHP (Merge Intervals)

```php
<?php
/**
 * Merge Intervals
 * টাইম: O(n log n), স্পেস: O(n)
 */
function mergeIntervals(array $intervals): array
{
    if (count($intervals) <= 1) return $intervals;

    // start দিয়ে সর্ট
    usort($intervals, fn($a, $b) => $a[0] - $b[0]);

    $merged = [$intervals[0]];

    for ($i = 1; $i < count($intervals); $i++) {
        $last = &$merged[count($merged) - 1];

        if ($intervals[$i][0] <= $last[1]) {
            // overlap — merge করো
            $last[1] = max($last[1], $intervals[$i][1]);
        } else {
            // overlap নেই — নতুন interval যোগ করো
            $merged[] = $intervals[$i];
        }
    }

    return $merged;
}

// পরীক্ষা
print_r(mergeIntervals([[1,3], [2,6], [8,10], [15,18]]));
// [[1,6], [8,10], [15,18]]
```

### JavaScript (Merge Intervals)

```javascript
/**
 * Merge Intervals
 * টাইম: O(n log n), স্পেস: O(n)
 */
function mergeIntervals(intervals) {
    if (intervals.length <= 1) return intervals;

    intervals.sort((a, b) => a[0] - b[0]);

    const merged = [intervals[0]];

    for (let i = 1; i < intervals.length; i++) {
        const last = merged[merged.length - 1];

        if (intervals[i][0] <= last[1]) {
            last[1] = Math.max(last[1], intervals[i][1]);
        } else {
            merged.push(intervals[i]);
        }
    }

    return merged;
}

// পরীক্ষা
console.log(mergeIntervals([[1,3], [2,6], [8,10], [15,18]]));
// [[1,6], [8,10], [15,18]]
```

---

## ৩.৪ Rotate Array

> একটি অ্যারেকে k ঘর ডানদিকে ঘোরান।

```
ইনপুট: [1, 2, 3, 4, 5, 6, 7], k = 3

3 Reversal Technique:
ধাপ ১: পুরো অ্যারে reverse    → [7, 6, 5, 4, 3, 2, 1]
ধাপ ২: প্রথম k টি reverse     → [5, 6, 7, 4, 3, 2, 1]
ধাপ ৩: বাকিটা reverse        → [5, 6, 7, 1, 2, 3, 4] ✓

কেন কাজ করে?
- মূল: [1,2,3,4 | 5,6,7]  (শেষ k=3 টি ডানে যাবে)
- reverse all: [7,6,5 | 4,3,2,1]
- reverse [0..k-1]: [5,6,7 | 4,3,2,1]
- reverse [k..n-1]: [5,6,7 | 1,2,3,4] ✓
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস

### PHP (Rotate Array)

```php
<?php
/**
 * Rotate Array — 3 Reversal Technique
 * টাইম: O(n), স্পেস: O(1)
 */
function rotate(array &$nums, int $k): void
{
    $n = count($nums);
    $k = $k % $n; // k যদি n-এর বেশি হয়

    if ($k === 0) return;

    // সহায়ক ফাংশন: নির্দিষ্ট পরিসীমা reverse করো
    $reverse = function (int $start, int $end) use (&$nums) {
        while ($start < $end) {
            [$nums[$start], $nums[$end]] = [$nums[$end], $nums[$start]];
            $start++;
            $end--;
        }
    };

    $reverse(0, $n - 1);       // পুরো অ্যারে reverse
    $reverse(0, $k - 1);       // প্রথম k টি reverse
    $reverse($k, $n - 1);     // বাকিটা reverse
}

// পরীক্ষা
$arr = [1, 2, 3, 4, 5, 6, 7];
rotate($arr, 3);
echo implode(', ', $arr); // 5, 6, 7, 1, 2, 3, 4
```

### JavaScript (Rotate Array)

```javascript
/**
 * Rotate Array — 3 Reversal Technique
 * টাইম: O(n), স্পেস: O(1)
 */
function rotate(nums, k) {
    k = k % nums.length;
    if (k === 0) return;

    const reverse = (start, end) => {
        while (start < end) {
            [nums[start], nums[end]] = [nums[end], nums[start]];
            start++;
            end--;
        }
    };

    reverse(0, nums.length - 1);
    reverse(0, k - 1);
    reverse(k, nums.length - 1);
}

// পরীক্ষা
const arr = [1, 2, 3, 4, 5, 6, 7];
rotate(arr, 3);
console.log(arr); // [5, 6, 7, 1, 2, 3, 4]
```

---

## ৩.৫ Product of Array Except Self

> একটি অ্যারে দেওয়া আছে। প্রতিটি ইনডেক্সে বাকি সব এলিমেন্টের গুণফল রাখুন। Division ব্যবহার করা যাবে না।

```
ইনপুট: [1, 2, 3, 4]

আইডিয়া: leftProduct × rightProduct

ইনডেক্স:       0    1    2    3
মূল অ্যারে:   [1,   2,   3,   4]
leftProduct:  [1,   1,   2,   6]   ← বামের সব এলিমেন্টের গুণফল
rightProduct: [24,  12,   4,   1]   ← ডানের সব এলিমেন্টের গুণফল
result:       [24,  12,   8,   6]   ← left × right

পরীক্ষা: result[1] = 1 × 12 = 12 = 1 × 3 × 4 ✓ (2 বাদে সবার গুণফল)
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস (আউটপুট ছাড়া)

### PHP (Product of Array Except Self)

```php
<?php
/**
 * Product of Array Except Self
 * টাইম: O(n), স্পেস: O(1) আউটপুট ছাড়া
 */
function productExceptSelf(array $nums): array
{
    $n = count($nums);
    $result = array_fill(0, $n, 1);

    // বাম থেকে ডানে — বামের গুণফল জমা করো
    $leftProduct = 1;
    for ($i = 0; $i < $n; $i++) {
        $result[$i] = $leftProduct;
        $leftProduct *= $nums[$i];
    }

    // ডান থেকে বামে — ডানের গুণফল দিয়ে গুণ করো
    $rightProduct = 1;
    for ($i = $n - 1; $i >= 0; $i--) {
        $result[$i] *= $rightProduct;
        $rightProduct *= $nums[$i];
    }

    return $result;
}

// পরীক্ষা
print_r(productExceptSelf([1, 2, 3, 4])); // [24, 12, 8, 6]
```

### JavaScript (Product of Array Except Self)

```javascript
/**
 * Product of Array Except Self
 * টাইম: O(n), স্পেস: O(1) আউটপুট ছাড়া
 */
function productExceptSelf(nums) {
    const n = nums.length;
    const result = new Array(n).fill(1);

    // বাম থেকে ডানে
    let leftProduct = 1;
    for (let i = 0; i < n; i++) {
        result[i] = leftProduct;
        leftProduct *= nums[i];
    }

    // ডান থেকে বামে
    let rightProduct = 1;
    for (let i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }

    return result;
}

// পরীক্ষা
console.log(productExceptSelf([1, 2, 3, 4])); // [24, 12, 8, 6]
```

---

# 🔤 ৪. স্ট্রিং (String) — মৌলিক ধারণা

## স্ট্রিং কি Character Array?

| বৈশিষ্ট্য | PHP | JavaScript |
|-----------|-----|-----------|
| স্ট্রিং টাইপ | byte sequence | UTF-16 encoded |
| Mutability | **Mutable** — সরাসরি পরিবর্তন যোগ্য | **Immutable** — পরিবর্তন করলে নতুন স্ট্রিং তৈরি হয় |
| Index Access | `$s[0]` → byte | `s[0]` বা `s.charAt(0)` → character |
| বাংলা ক্যারেক্টার | `mb_strlen()` লাগে | `.length` সাধারণত কাজ করে |
| Concatenation | `.` operator — O(n) | `+` operator — O(n), নতুন স্ট্রিং তৈরি হয় |

### PHP-তে Mutability

```php
<?php
// PHP-তে স্ট্রিং mutable
$s = "Hello";
$s[0] = "J";     // সরাসরি পরিবর্তন সম্ভব
echo $s;          // "Jello"

// কিন্তু mb (multibyte) ফাংশন দরকার বাংলার জন্য
$bangla = "বাংলা";
echo strlen($bangla);      // 15 (bytes) — ভুল!
echo mb_strlen($bangla);   // 5 (characters) — সঠিক!
```

### JavaScript-এ Immutability

```javascript
// JavaScript-এ স্ট্রিং immutable
let s = "Hello";
s[0] = "J";       // কোনো প্রভাব নেই!
console.log(s);    // "Hello" — পরিবর্তন হয়নি

// নতুন স্ট্রিং তৈরি করতে হবে
s = "J" + s.slice(1);
console.log(s);    // "Jello"

// বাংলা ক্যারেক্টার
const bangla = "বাংলা";
console.log(bangla.length);          // 5 (সাধারণত সঠিক)
console.log([...bangla].length);     // 5 (নিরাপদ উপায়)
```

---

## Encoding: ASCII, UTF-8, Unicode

```
┌────────────────────────────────────────────────────────────────┐
│                 ক্যারেক্টার এনকোডিং                            │
├──────────┬──────────┬──────────────────────────────────────────┤
│  ASCII   │  7 bit   │ 0-127, শুধু ইংরেজি (A=65, a=97, 0=48) │
│  UTF-8   │ 1-4 byte │ সব ভাষা সাপোর্ট, backward compatible   │
│  UTF-16  │ 2-4 byte │ JavaScript অভ্যন্তরীণভাবে ব্যবহার করে  │
│  Unicode │ কোড পয়েন্ট │ সব ক্যারেক্টারের ইউনিক নম্বর          │
└──────────┴──────────┴──────────────────────────────────────────┘

বাংলা ক্যারেক্টার UTF-8 এনকোডিং:
'ব' = U+09AC → UTF-8: 0xE0 0x9A 0xAC (3 bytes)
'া' = U+09BE → UTF-8: 0xE0 0x9A 0xBE (3 bytes)

তাই PHP-তে strlen("বা") = 6 (bytes), mb_strlen("বা") = 2 (characters)
```

---

## স্ট্রিং অপারেশন কমপ্লেক্সিটি টেবিল

| অপারেশন | PHP | JavaScript | ব্যাখ্যা |
|---------|-----|-----------|---------|
| Access s[i] | O(1) | O(1) | ইনডেক্স দিয়ে অ্যাক্সেস |
| Search (indexOf) | O(n×m) | O(n×m) | n=string, m=pattern |
| Concatenation | O(n) | O(n+m) | নতুন স্ট্রিং তৈরি (JS immutable) |
| Substring | O(k) | O(k) | k = substring-এর দৈর্ঘ্য |
| Compare | O(n) | O(n) | ক্যারেক্টার-বাই-ক্যারেক্টার |
| Length | O(1)* | O(1) | *PHP: strlen O(1), mb_strlen O(n) |
| Replace | O(n) | O(n) | পুরো স্ট্রিং স্ক্যান |

---

# 🔧 ৫. স্ট্রিং টেকনিক ও সমস্যা

## ৫.১ Palindrome Check (প্যালিনড্রোম পরীক্ষা)

> একটি স্ট্রিং সামনে থেকে ও পেছন থেকে একই কিনা যাচাই করুন।

```
"racecar" → r-a-c-e-c-a-r
             ↑           ↑   r == r ✓
               ↑       ↑     a == a ✓
                 ↑   ↑       c == c ✓
                   ↑         মাঝখান → প্যালিনড্রোম ✓
```

**কমপ্লেক্সিটি:** O(n) টাইম, O(1) স্পেস

### PHP (Palindrome)

```php
<?php
/**
 * Palindrome Check — Two Pointer
 * টাইম: O(n), স্পেস: O(1)
 */
function isPalindrome(string $s): bool
{
    // শুধু alphanumeric রাখো, lowercase করো
    $s = strtolower(preg_replace('/[^a-zA-Z0-9]/', '', $s));
    $left = 0;
    $right = strlen($s) - 1;

    while ($left < $right) {
        if ($s[$left] !== $s[$right]) {
            return false; // মিলছে না
        }
        $left++;
        $right--;
    }

    return true;
}

// পরীক্ষা
var_dump(isPalindrome("racecar"));              // true
var_dump(isPalindrome("A man, a plan, a canal: Panama")); // true
var_dump(isPalindrome("hello"));                // false
```

### JavaScript (Palindrome)

```javascript
/**
 * Palindrome Check — Two Pointer
 * টাইম: O(n), স্পেস: O(1)
 */
function isPalindrome(s) {
    s = s.toLowerCase().replace(/[^a-z0-9]/g, '');
    let left = 0;
    let right = s.length - 1;

    while (left < right) {
        if (s[left] !== s[right]) return false;
        left++;
        right--;
    }

    return true;
}

// পরীক্ষা
console.log(isPalindrome("racecar"));              // true
console.log(isPalindrome("A man, a plan, a canal: Panama")); // true
console.log(isPalindrome("hello"));                // false
```

---

## ৫.২ Anagram Detection (অ্যানাগ্রাম শনাক্তকরণ)

> দুটি স্ট্রিং-এ একই ক্যারেক্টার একই সংখ্যায় আছে কিনা যাচাই।

```
"listen" ও "silent" → অ্যানাগ্রাম ✓
কারণ: e=1, i=1, l=1, n=1, s=1, t=1 → দুটোতেই একই

পদ্ধতি ১: Sorting — O(n log n)
  sort("listen") = "eilnst"
  sort("silent") = "eilnst"  → সমান ✓

পদ্ধতি ২: Frequency Counter — O(n)
  প্রথম স্ট্রিং-এ প্রতিটি char-এর count বাড়াও
  দ্বিতীয় স্ট্রিং-এ প্রতিটি char-এর count কমাও
  সব count 0 হলে → অ্যানাগ্রাম
```

### PHP (Anagram)

```php
<?php
/**
 * Anagram Detection — Frequency Counter
 * টাইম: O(n), স্পেস: O(1) — সর্বোচ্চ 26 ক্যারেক্টার
 */
function isAnagram(string $s, string $t): bool
{
    if (strlen($s) !== strlen($t)) return false;

    $freq = [];

    // প্রথম স্ট্রিং-এর frequency বাড়াও
    for ($i = 0; $i < strlen($s); $i++) {
        $freq[$s[$i]] = ($freq[$s[$i]] ?? 0) + 1;
    }

    // দ্বিতীয় স্ট্রিং-এর frequency কমাও
    for ($i = 0; $i < strlen($t); $i++) {
        if (!isset($freq[$t[$i]]) || $freq[$t[$i]] === 0) {
            return false;
        }
        $freq[$t[$i]]--;
    }

    return true;
}

// পরীক্ষা
var_dump(isAnagram("listen", "silent")); // true
var_dump(isAnagram("hello", "world"));   // false
var_dump(isAnagram("anagram", "nagaram")); // true
```

### JavaScript (Anagram)

```javascript
/**
 * Anagram Detection — Frequency Counter
 * টাইম: O(n), স্পেস: O(1)
 */
function isAnagram(s, t) {
    if (s.length !== t.length) return false;

    const freq = {};

    for (const ch of s) {
        freq[ch] = (freq[ch] || 0) + 1;
    }

    for (const ch of t) {
        if (!freq[ch]) return false;
        freq[ch]--;
    }

    return true;
}

// পরীক্ষা
console.log(isAnagram("listen", "silent")); // true
console.log(isAnagram("hello", "world"));   // false
console.log(isAnagram("anagram", "nagaram")); // true
```

---

## ৫.৩ Longest Palindromic Substring (দীর্ঘতম প্যালিনড্রোমিক সাবস্ট্রিং)

> একটি স্ট্রিং-এর দীর্ঘতম palindromic substring খুঁজুন।

```
ইনপুট: "babad"

Expand Around Center পদ্ধতি:
প্রতিটি ক্যারেক্টার (ও প্রতিটি জোড়া) কেন্দ্র ধরে দুদিকে expand করো।

কেন্দ্র 'b'(0): b → শুধু 'b'
কেন্দ্র 'a'(1): a → bab (b==b) → palindrome "bab" (দৈর্ঘ্য 3)
কেন্দ্র 'b'(2): b → aba (a==a) → palindrome "aba" (দৈর্ঘ্য 3)
কেন্দ্র 'a'(3): a → শুধু 'a'
কেন্দ্র 'd'(4): d → শুধু 'd'

সম্ভাব্য কেন্দ্র: 2n-1 টি (n টি single + n-1 টি between)

উত্তর: "bab" (বা "aba")
```

**কমপ্লেক্সিটি:** O(n²) টাইম, O(1) স্পেস

### PHP (Longest Palindromic Substring)

```php
<?php
/**
 * Longest Palindromic Substring — Expand Around Center
 * টাইম: O(n²), স্পেস: O(1)
 */
function longestPalindrome(string $s): string
{
    $n = strlen($s);
    if ($n < 2) return $s;

    $start = 0;
    $maxLen = 1;

    // কেন্দ্র থেকে expand করার সহায়ক ফাংশন
    $expand = function (int $left, int $right) use ($s, $n, &$start, &$maxLen) {
        while ($left >= 0 && $right < $n && $s[$left] === $s[$right]) {
            $len = $right - $left + 1;
            if ($len > $maxLen) {
                $maxLen = $len;
                $start = $left;
            }
            $left--;
            $right++;
        }
    };

    for ($i = 0; $i < $n; $i++) {
        $expand($i, $i);       // বিজোড় দৈর্ঘ্যের palindrome (একক কেন্দ্র)
        $expand($i, $i + 1);   // জোড় দৈর্ঘ্যের palindrome (দুই কেন্দ্র)
    }

    return substr($s, $start, $maxLen);
}

// পরীক্ষা
echo longestPalindrome("babad") . "\n";  // "bab" বা "aba"
echo longestPalindrome("cbbd") . "\n";   // "bb"
echo longestPalindrome("racecar") . "\n"; // "racecar"
```

### JavaScript (Longest Palindromic Substring)

```javascript
/**
 * Longest Palindromic Substring — Expand Around Center
 * টাইম: O(n²), স্পেস: O(1)
 */
function longestPalindrome(s) {
    let start = 0, maxLen = 1;

    function expand(left, right) {
        while (left >= 0 && right < s.length && s[left] === s[right]) {
            const len = right - left + 1;
            if (len > maxLen) {
                maxLen = len;
                start = left;
            }
            left--;
            right++;
        }
    }

    for (let i = 0; i < s.length; i++) {
        expand(i, i);       // বিজোড় দৈর্ঘ্য
        expand(i, i + 1);   // জোড় দৈর্ঘ্য
    }

    return s.substring(start, start + maxLen);
}

// পরীক্ষা
console.log(longestPalindrome("babad"));  // "bab"
console.log(longestPalindrome("cbbd"));   // "bb"
console.log(longestPalindrome("racecar")); // "racecar"
```

---

## ৫.৪ String Matching — KMP Algorithm

> একটি text-এ pattern খোঁজার জন্য সবচেয়ে কার্যকর অ্যালগরিদম।

### Brute Force String Matching — O(n×m)

```
text:    "AABAACAADAABAABA"
pattern: "AABA"

প্রতিটি পজিশনে pattern ম্যাচ করো:
পজিশন 0: AABA == AABA ✓
পজিশন 1: ABAA != AABA ✗
পজিশন 2: BAAC != AABA ✗
... ইত্যাদি
```

### KMP (Knuth-Morris-Pratt) Algorithm — O(n+m)

**মূল আইডিয়া:** mismatch হলে pattern-এর শুরু থেকে আবার শুরু না করে, pattern-এর **failure function (LPS array)** ব্যবহার করে বুদ্ধিমানভাবে এগিয়ে যাও।

```
LPS (Longest Proper Prefix which is also Suffix) Array:

pattern: "AABAAB"
ইনডেক্স:  0 1 2 3 4 5
LPS:      [0,1,0,1,2,3]

ব্যাখ্যা:
i=0: "A"       → কোনো proper prefix=suffix নেই → 0
i=1: "AA"      → "A" prefix ও suffix → 1
i=2: "AAB"     → কোনো match নেই → 0
i=3: "AABA"    → "A" match → 1
i=4: "AABAA"   → "AA" match → 2
i=5: "AABAAB"  → "AAB" match → 3

LPS তৈরির প্রক্রিয়া:

       A  A  B  A  A  B
       ↑
       len=0, lps[0]=0, i=1

       A  A  B  A  A  B
          ↑
       pattern[1]=A == pattern[0]=A → len=1, lps[1]=1, i=2

       A  A  B  A  A  B
             ↑
       pattern[2]=B != pattern[1]=A → len=lps[0]=0
       pattern[2]=B != pattern[0]=A → lps[2]=0, i=3

       ... এভাবে চলতে থাকে
```

### PHP (KMP Algorithm)

```php
<?php
/**
 * KMP String Matching Algorithm
 * টাইম: O(n+m), স্পেস: O(m)
 * n = text দৈর্ঘ্য, m = pattern দৈর্ঘ্য
 */
class KMP
{
    /**
     * LPS (failure function) অ্যারে তৈরি — O(m)
     */
    private static function buildLPS(string $pattern): array
    {
        $m = strlen($pattern);
        $lps = array_fill(0, $m, 0);
        $len = 0; // আগের longest prefix suffix-এর দৈর্ঘ্য
        $i = 1;

        while ($i < $m) {
            if ($pattern[$i] === $pattern[$len]) {
                $len++;
                $lps[$i] = $len;
                $i++;
            } else {
                if ($len !== 0) {
                    $len = $lps[$len - 1]; // পিছিয়ে যাও
                } else {
                    $lps[$i] = 0;
                    $i++;
                }
            }
        }

        return $lps;
    }

    /**
     * text-এ pattern-এর সব occurrence খোঁজো — O(n+m)
     */
    public static function search(string $text, string $pattern): array
    {
        $n = strlen($text);
        $m = strlen($pattern);
        $positions = [];

        if ($m === 0) return $positions;

        $lps = self::buildLPS($pattern);
        $i = 0; // text pointer
        $j = 0; // pattern pointer

        while ($i < $n) {
            if ($text[$i] === $pattern[$j]) {
                $i++;
                $j++;
            }

            if ($j === $m) {
                $positions[] = $i - $j; // ম্যাচ পাওয়া গেছে!
                $j = $lps[$j - 1];      // পরবর্তী ম্যাচ খোঁজো
            } elseif ($i < $n && $text[$i] !== $pattern[$j]) {
                if ($j !== 0) {
                    $j = $lps[$j - 1]; // LPS ব্যবহার করে skip করো
                } else {
                    $i++;
                }
            }
        }

        return $positions;
    }
}

// পরীক্ষা
$text = "AABAACAADAABAABA";
$pattern = "AABA";
$result = KMP::search($text, $pattern);
echo "পজিশন: " . implode(', ', $result) . "\n"; // 0, 9, 12
```

### JavaScript (KMP Algorithm)

```javascript
/**
 * KMP String Matching Algorithm
 * টাইম: O(n+m), স্পেস: O(m)
 */
class KMP {
    /** LPS (failure function) অ্যারে তৈরি — O(m) */
    static buildLPS(pattern) {
        const m = pattern.length;
        const lps = new Array(m).fill(0);
        let len = 0;
        let i = 1;

        while (i < m) {
            if (pattern[i] === pattern[len]) {
                len++;
                lps[i] = len;
                i++;
            } else {
                if (len !== 0) {
                    len = lps[len - 1];
                } else {
                    lps[i] = 0;
                    i++;
                }
            }
        }

        return lps;
    }

    /** text-এ pattern-এর সব occurrence খোঁজো — O(n+m) */
    static search(text, pattern) {
        const n = text.length;
        const m = pattern.length;
        const positions = [];

        if (m === 0) return positions;

        const lps = this.buildLPS(pattern);
        let i = 0, j = 0;

        while (i < n) {
            if (text[i] === pattern[j]) {
                i++;
                j++;
            }

            if (j === m) {
                positions.push(i - j);
                j = lps[j - 1];
            } else if (i < n && text[i] !== pattern[j]) {
                if (j !== 0) {
                    j = lps[j - 1];
                } else {
                    i++;
                }
            }
        }

        return positions;
    }
}

// পরীক্ষা
const text = "AABAACAADAABAABA";
const pattern = "AABA";
console.log(`পজিশন: ${KMP.search(text, pattern)}`); // [0, 9, 12]
```

---

## ৫.৫ Rabin-Karp Algorithm (রবিন-কার্প অ্যালগরিদম)

> Hash-based string matching — rolling hash ব্যবহার করে pattern খোঁজা।

```
মূল আইডিয়া:
১. pattern-এর hash বের করো
২. text-এর প্রতিটি window-এর hash বের করো (rolling hash)
৩. hash মিললে → character-by-character verify করো

Rolling Hash সূত্র:
hash("abc") = a×d² + b×d¹ + c×d⁰  (d = base, যেমন 256)

নতুন window-এর hash পুরনো থেকে:
hash("bcd") = (hash("abc") - a×d²) × d + d×d⁰
            = (পুরনো hash - বাম char × d^(m-1)) × d + নতুন char

এভাবে O(1)-এ নতুন hash বের করা যায়!
```

**কমপ্লেক্সিটি:**
- Average: O(n+m)
- Worst case: O(n×m) — সব hash collision হলে

### PHP (Rabin-Karp)

```php
<?php
/**
 * Rabin-Karp String Matching
 * Average: O(n+m), Worst: O(n×m)
 */
function rabinKarp(string $text, string $pattern): array
{
    $n = strlen($text);
    $m = strlen($pattern);
    $positions = [];

    if ($m > $n) return $positions;

    $base = 256;           // ক্যারেক্টার সেটের আকার
    $mod = 1000000007;     // বড় prime — hash collision কমাতে
    $patternHash = 0;
    $textHash = 0;
    $h = 1; // base^(m-1) mod

    // h = base^(m-1) % mod হিসাব করো
    for ($i = 0; $i < $m - 1; $i++) {
        $h = ($h * $base) % $mod;
    }

    // প্রথম window-এর hash
    for ($i = 0; $i < $m; $i++) {
        $patternHash = ($patternHash * $base + ord($pattern[$i])) % $mod;
        $textHash = ($textHash * $base + ord($text[$i])) % $mod;
    }

    // window slide করো
    for ($i = 0; $i <= $n - $m; $i++) {
        if ($patternHash === $textHash) {
            // hash মিলেছে — verify করো (spurious hit এড়ানো)
            if (substr($text, $i, $m) === $pattern) {
                $positions[] = $i;
            }
        }

        // পরবর্তী window-এর rolling hash
        if ($i < $n - $m) {
            $textHash = (($textHash - ord($text[$i]) * $h) * $base + ord($text[$i + $m])) % $mod;
            if ($textHash < 0) $textHash += $mod;
        }
    }

    return $positions;
}

// পরীক্ষা
$result = rabinKarp("AABAACAADAABAABA", "AABA");
echo "পজিশন: " . implode(', ', $result) . "\n"; // 0, 9, 12
```

### JavaScript (Rabin-Karp)

```javascript
/**
 * Rabin-Karp String Matching
 * Average: O(n+m), Worst: O(n×m)
 */
function rabinKarp(text, pattern) {
    const n = text.length;
    const m = pattern.length;
    const positions = [];

    if (m > n) return positions;

    const base = 256;
    const mod = 1000000007;
    let patternHash = 0;
    let textHash = 0;
    let h = 1;

    for (let i = 0; i < m - 1; i++) {
        h = (h * base) % mod;
    }

    for (let i = 0; i < m; i++) {
        patternHash = (patternHash * base + pattern.charCodeAt(i)) % mod;
        textHash = (textHash * base + text.charCodeAt(i)) % mod;
    }

    for (let i = 0; i <= n - m; i++) {
        if (patternHash === textHash) {
            if (text.substring(i, i + m) === pattern) {
                positions.push(i);
            }
        }

        if (i < n - m) {
            textHash = ((textHash - text.charCodeAt(i) * h) * base + text.charCodeAt(i + m)) % mod;
            if (textHash < 0) textHash += mod;
        }
    }

    return positions;
}

// পরীক্ষা
console.log(rabinKarp("AABAACAADAABAABA", "AABA")); // [0, 9, 12]
```

### String Matching অ্যালগরিদম তুলনা

| অ্যালগরিদম | Preprocessing | Matching | মোট | স্পেস | ব্যবহারের ক্ষেত্র |
|-----------|--------------|---------|-----|------|-----------------|
| Brute Force | O(0) | O(n×m) | O(n×m) | O(1) | ছোট text/pattern |
| KMP | O(m) | O(n) | O(n+m) | O(m) | গ্যারান্টিড linear time |
| Rabin-Karp | O(m) | O(n) avg | O(n+m) avg | O(1) | Multiple pattern search |

---

# 🛠️ ৬. সম্পূর্ণ প্র্যাক্টিক্যাল প্রজেক্ট

## টেক্সট এডিটর অটোকমপ্লিট (Text Editor Autocomplete)

> Trie (prefix tree) ও string matching ব্যবহার করে একটি autocomplete সিস্টেম তৈরি করা।

```
সমস্যা:
একটি শব্দভান্ডারে (dictionary) অনেক শব্দ আছে।
ব্যবহারকারী কিছু টাইপ করলে matching শব্দগুলো suggest করতে হবে।

উদাহরণ:
শব্দভান্ডার: ["apple", "app", "application", "apply", "banana", "band", "bat"]

ব্যবহারকারী টাইপ করছে: "app"
Suggestions: ["app", "apple", "application", "apply"]
```

### ❌ Brute Force Approach — O(n × m) per search

```
প্রতিটি সার্চে পুরো dictionary loop করো:
for each word in dictionary:
    if word.startsWith(prefix):
        add to results

n = dictionary-র আকার, m = prefix-এর দৈর্ঘ্য
প্রতিটি সার্চ: O(n × m) — ধীর!
```

### ✅ Optimized: Trie (Prefix Tree) — O(m + k)

```
Trie গঠন:

              (root)
             /      \
            a        b
           /        / \
          p        a    a
         / \      /      \
        p   l    n        t
       / \   \    \
      l   i   y    d
     /     \
    e       c
             \
              a
               \
                t
                 \
                  i
                   \
                    o
                     \
                      n

"app" prefix সার্চ → 'a' → 'p' → 'p' নোডে পৌঁছাও
→ এই নোড থেকে সব child traverse করো
→ ফলাফল: ["app", "apple", "application", "apply"]

টাইম: O(m + k) — m = prefix দৈর্ঘ্য, k = মোট ফলাফলের ক্যারেক্টার সংখ্যা
```

### PHP (Trie-based Autocomplete)

```php
<?php
/**
 * Trie Node — প্রতিটি নোড একটি ক্যারেক্টার উপস্থাপন করে
 */
class TrieNode
{
    public array $children = [];  // ক্যারেক্টার → TrieNode
    public bool $isEndOfWord = false;
    public ?string $word = null;  // শব্দ শেষ হলে পূর্ণ শব্দ সংরক্ষণ
}

/**
 * Trie (Prefix Tree) — Autocomplete সিস্টেম
 * Insert: O(m), Search: O(m + k)
 * m = শব্দ/prefix দৈর্ঘ্য, k = ফলাফলের মোট ক্যারেক্টার
 */
class Autocomplete
{
    private TrieNode $root;

    public function __construct()
    {
        $this->root = new TrieNode();
    }

    /** শব্দ যোগ করো — O(m) */
    public function insert(string $word): void
    {
        $node = $this->root;

        for ($i = 0; $i < strlen($word); $i++) {
            $char = $word[$i];
            if (!isset($node->children[$char])) {
                $node->children[$char] = new TrieNode();
            }
            $node = $node->children[$char];
        }

        $node->isEndOfWord = true;
        $node->word = $word;
    }

    /** prefix দিয়ে সব matching শব্দ খোঁজো — O(m + k) */
    public function suggest(string $prefix): array
    {
        $node = $this->root;

        // prefix-এর শেষ নোডে যাও
        for ($i = 0; $i < strlen($prefix); $i++) {
            $char = $prefix[$i];
            if (!isset($node->children[$char])) {
                return []; // এই prefix-এ কোনো শব্দ নেই
            }
            $node = $node->children[$char];
        }

        // এই নোড থেকে সব শব্দ সংগ্রহ করো (DFS)
        $results = [];
        $this->collectWords($node, $results);
        return $results;
    }

    /** DFS দিয়ে নোডের নিচের সব শব্দ সংগ্রহ */
    private function collectWords(TrieNode $node, array &$results): void
    {
        if ($node->isEndOfWord) {
            $results[] = $node->word;
        }

        // বর্ণানুক্রমে children traverse করো
        ksort($node->children);
        foreach ($node->children as $child) {
            $this->collectWords($child, $results);
        }
    }

    /** শব্দ আছে কিনা চেক করো — O(m) */
    public function search(string $word): bool
    {
        $node = $this->root;

        for ($i = 0; $i < strlen($word); $i++) {
            $char = $word[$i];
            if (!isset($node->children[$char])) {
                return false;
            }
            $node = $node->children[$char];
        }

        return $node->isEndOfWord;
    }
}

// === ব্যবহার ===
$ac = new Autocomplete();

// শব্দভান্ডার যোগ করো
$words = ["apple", "app", "application", "apply", "banana", "band", "bat", "ball"];
foreach ($words as $word) {
    $ac->insert($word);
}

// অটোকমপ্লিট পরীক্ষা
echo "Prefix 'app': " . implode(', ', $ac->suggest("app")) . "\n";
// app, apple, application, apply

echo "Prefix 'ba': " . implode(', ', $ac->suggest("ba")) . "\n";
// ball, banana, band, bat

echo "Prefix 'ban': " . implode(', ', $ac->suggest("ban")) . "\n";
// banana, band

echo "Prefix 'xyz': " . implode(', ', $ac->suggest("xyz")) . "\n";
// (কিছু নেই)

echo "'apple' আছে? " . ($ac->search("apple") ? "হ্যাঁ" : "না") . "\n"; // হ্যাঁ
echo "'apex' আছে? " . ($ac->search("apex") ? "হ্যাঁ" : "না") . "\n";   // না
```

### JavaScript (Trie-based Autocomplete)

```javascript
/**
 * Trie Node
 */
class TrieNode {
    constructor() {
        this.children = {};        // ক্যারেক্টার → TrieNode
        this.isEndOfWord = false;
        this.word = null;
    }
}

/**
 * Trie (Prefix Tree) — Autocomplete সিস্টেম
 * Insert: O(m), Search: O(m + k)
 */
class Autocomplete {
    constructor() {
        this.root = new TrieNode();
    }

    /** শব্দ যোগ করো — O(m) */
    insert(word) {
        let node = this.root;

        for (const char of word) {
            if (!node.children[char]) {
                node.children[char] = new TrieNode();
            }
            node = node.children[char];
        }

        node.isEndOfWord = true;
        node.word = word;
    }

    /** prefix দিয়ে suggestions খোঁজো — O(m + k) */
    suggest(prefix) {
        let node = this.root;

        for (const char of prefix) {
            if (!node.children[char]) return [];
            node = node.children[char];
        }

        const results = [];
        this._collectWords(node, results);
        return results;
    }

    /** DFS দিয়ে শব্দ সংগ্রহ */
    _collectWords(node, results) {
        if (node.isEndOfWord) {
            results.push(node.word);
        }

        // বর্ণানুক্রমে traverse
        const sortedKeys = Object.keys(node.children).sort();
        for (const key of sortedKeys) {
            this._collectWords(node.children[key], results);
        }
    }

    /** শব্দ আছে কিনা — O(m) */
    search(word) {
        let node = this.root;

        for (const char of word) {
            if (!node.children[char]) return false;
            node = node.children[char];
        }

        return node.isEndOfWord;
    }
}

// === ব্যবহার ===
const ac = new Autocomplete();

const words = ["apple", "app", "application", "apply", "banana", "band", "bat", "ball"];
words.forEach(word => ac.insert(word));

console.log("Prefix 'app':", ac.suggest("app"));
// ["app", "apple", "application", "apply"]

console.log("Prefix 'ba':", ac.suggest("ba"));
// ["ball", "banana", "band", "bat"]

console.log("Prefix 'ban':", ac.suggest("ban"));
// ["banana", "band"]

console.log("Prefix 'xyz':", ac.suggest("xyz"));
// []

console.log(`'apple' আছে? ${ac.search("apple") ? "হ্যাঁ" : "না"}`); // হ্যাঁ
console.log(`'apex' আছে? ${ac.search("apex") ? "হ্যাঁ" : "না"}`);   // না
```

### Brute Force vs Trie তুলনা

| বৈশিষ্ট্য | Brute Force | Trie |
|-----------|-------------|------|
| সার্চ টাইম | O(n × m) | O(m + k) |
| ইনসার্ট | O(1) | O(m) |
| স্পেস | O(total chars) | O(total chars × alphabet) |
| Prefix সার্চ | ধীর | খুব দ্রুত |
| ব্যবহারের ক্ষেত্র | ছোট dictionary | বড় dictionary, autocomplete |

> n = dictionary-র শব্দ সংখ্যা, m = শব্দ/prefix দৈর্ঘ্য, k = ফলাফলের ক্যারেক্টার

---

# 📋 অ্যারে ও স্ট্রিং কমপ্লেক্সিটি চিটশিট

## অ্যারে অপারেশন কমপ্লেক্সিটি

| অপারেশন | Average | Worst | স্পেস |
|---------|---------|-------|------|
| Access | O(1) | O(1) | — |
| Search (unsorted) | O(n) | O(n) | — |
| Search (sorted) | O(log n) | O(log n) | — |
| Insert (শেষে) | O(1)* | O(n) | — |
| Insert (শুরুতে) | O(n) | O(n) | — |
| Delete (শেষে) | O(1) | O(1) | — |
| Delete (শুরুতে) | O(n) | O(n) | — |

\* amortized — Dynamic array resize-এ O(n) লাগতে পারে

## অ্যারে টেকনিক কমপ্লেক্সিটি

| টেকনিক | টাইম | স্পেস | ব্যবহার |
|--------|------|------|---------|
| Two Pointer | O(n) | O(1) | Sorted pair, palindrome, container |
| Sliding Window (fixed) | O(n) | O(1) | Max sum subarray of size k |
| Sliding Window (variable) | O(n) | O(k) | Longest substring, min window |
| Kadane's Algorithm | O(n) | O(1) | Maximum subarray sum |
| Prefix Sum | O(n) build, O(1) query | O(n) | Range sum, subarray sum = k |
| Binary Search | O(log n) | O(1) | Sorted array search |

## স্ট্রিং অ্যালগরিদম কমপ্লেক্সিটি

| অ্যালগরিদম | টাইম | স্পেস | ব্যাখ্যা |
|-----------|------|------|---------|
| Palindrome (Two Pointer) | O(n) | O(1) | দুই প্রান্ত থেকে তুলনা |
| Anagram (Frequency) | O(n) | O(1) | 26 ক্যারেক্টার counter |
| Longest Palindromic Substr | O(n²) | O(1) | Expand around center |
| KMP Matching | O(n+m) | O(m) | LPS array preprocessing |
| Rabin-Karp | O(n+m) avg | O(1) | Rolling hash |
| Trie Insert/Search | O(m) | O(m) per word | Prefix-based operations |

## সমস্যা কমপ্লেক্সিটি সারসংক্ষেপ

| সমস্যা | Brute Force | Optimized | অপটিমাইজড পদ্ধতি |
|--------|-------------|-----------|-----------------|
| Two Sum | O(n²) | O(n) | Hash Map |
| Three Sum | O(n³) | O(n²) | Sort + Two Pointer |
| Merge Intervals | O(n²) | O(n log n) | Sort + Linear Scan |
| Rotate Array | O(n×k) | O(n) | 3 Reversal |
| Product Except Self | O(n²) | O(n) | Left-Right Product |
| Max Subarray | O(n³) | O(n) | Kadane's Algorithm |
| Subarray Sum = K | O(n²) | O(n) | Prefix Sum + HashMap |

---

# 🎯 ইন্টারভিউ টিপস

## অ্যারে সমস্যায় চিন্তা করার ধাপ

```
১. অ্যারে কি sorted?
   → হ্যাঁ → Two Pointer / Binary Search বিবেচনা করো
   → না → Sort করার দরকার আছে কি?

২. Subarray/substring সমস্যা?
   → Fixed size → Sliding Window
   → Variable size → Sliding Window + HashMap
   → Sum related → Prefix Sum / Kadane's

৩. Pair/triplet খোঁজা?
   → Two Sum → HashMap
   → Three Sum → Sort + Two Pointer
   → k Sum → Reduce to Two Sum

৪. In-place পরিবর্তন?
   → Two Pointer (same direction or opposite)
   → Swap technique

৫. স্পেস O(1) constraint?
   → Bit manipulation
   → Math tricks (XOR, modulo)
   → Two Pointer
```

## স্ট্রিং সমস্যায় প্রচলিত প্যাটার্ন

```
১. ক্যারেক্টার frequency → HashMap/Array[26]
২. Substring সমস্যা → Sliding Window
৩. Palindrome → Two Pointer / Expand Around Center
৪. Pattern Matching → KMP / Rabin-Karp
৫. Prefix সমস্যা → Trie
৬. Anagram/Permutation → Sorting বা Frequency Counter
```

## সাধারণ ভুল এড়ানোর উপায়

| ভুল | সমাধান |
|-----|--------|
| Off-by-one error | Edge case চেক: empty array, single element |
| Integer overflow | বড় সংখ্যায় modulo ব্যবহার করো |
| Mutating while iterating | আলাদা result array ব্যবহার করো |
| Duplicate হ্যান্ডলিং ভুলে যাওয়া | Sort করে skip করো |
| Negative number ভুলে যাওয়া | Test case-এ negative সংখ্যা রাখো |
| UTF-8 encoding সমস্যা | PHP-তে mb_ ফাংশন ব্যবহার করো |

---

# 🔗 পরবর্তী টপিক

অ্যারে ও স্ট্রিং-এর ভিত্তি শক্ত হলে পরবর্তী ডাটা স্ট্রাকচার শিখুন:

**[লিংকড লিস্ট (Linked List) →](./linked-list.md)**

লিংকড লিস্ট অ্যারের সীমাবদ্ধতা (fixed size, costly insertion) কাটিয়ে ওঠে। Singly, Doubly, ও Circular Linked List-এর গভীর আলোচনা পরবর্তী গাইডে।
