# 🔢 Big O নোটেশন — টাইম ও স্পেস কমপ্লেক্সিটি সম্পূর্ণ গাইড

> "অ্যালগরিদমের দক্ষতা বোঝা মানেই Big O বোঝা — এটাই প্রোগ্রামিংয়ের প্রথম ধাপ।"

---

## 📑 সূচিপত্র

| # | বিষয় |
|---|-------|
| ১ | [Big O কী ও কেন দরকার](#১--big-o-কী-ও-কেন-দরকার) |
| ২ | [Asymptotic Notation (O, Ω, Θ)](#২--asymptotic-notation-o-ω-θ) |
| ৩ | [সাধারণ Big O ক্লাস](#৩--সাধারণ-big-o-ক্লাস) |
| ৪ | [Growth Rate তুলনা](#৪--growth-rate-তুলনা) |
| ৫ | [Big O হিসাব করার নিয়ম](#৫--big-o-হিসাব-করার-নিয়ম) |
| ৬ | [কমপ্লেক্সিটি বিশ্লেষণ প্র্যাক্টিস](#৬--কমপ্লেক্সিটি-বিশ্লেষণ-প্র্যাক্টিস) |
| ৭ | [Space Complexity](#৭--space-complexity) |
| ৮ | [Amortized Analysis](#৮--amortized-analysis) |
| ৯ | [Best, Average, Worst Case](#৯--best-average-worst-case) |
| ১০ | [প্র্যাক্টিক্যাল প্রজেক্ট](#১০--প্র্যাক্টিক্যাল-প্রজেক্ট) |
| ১১ | [চিট শিট মাস্টার টেবিল](#১১--চিট-শিট-মাস্টার-টেবিল) |
| ১২ | [ইন্টারভিউ টিপস](#১২--ইন্টারভিউ-টিপস) |
| ১৩ | [সম্পর্কিত টপিক লিংক](#১৩--সম্পর্কিত-টপিক-লিংক) |

---

## ১ — 🎯 Big O কী ও কেন দরকার

> "Big O হলো অ্যালগরিদমের পারফরম্যান্সের ভাষা।"

### 📖 সংজ্ঞা

**Big O নোটেশন** হলো একটি গাণিতিক চিহ্ন যা বর্ণনা করে ইনপুট সাইজ (n) বাড়লে
একটি অ্যালগরিদমের **সময়** বা **মেমরি** কতটা বাড়ে — worst case-এ।

```
f(n) = O(g(n))

মানে: এমন ধনাত্মক ধ্রুবক c ও n₀ আছে যেন
      সব n ≥ n₀ এর জন্য f(n) ≤ c·g(n)
```

### 🏠 বাস্তব উদাহরণ

কল্পনা করুন আপনি একটি **ফোনবুক** থেকে নাম খুঁজছেন:

| পদ্ধতি | বাস্তব কাজ | Big O |
|--------|-----------|-------|
| প্রথম পৃষ্ঠা থেকে একে একে দেখা | Linear Search | O(n) |
| মাঝখান থেকে অর্ধেক বাদ দেওয়া | Binary Search | O(log n) |
| সরাসরি ইনডেক্স দিয়ে খোলা | Hash Lookup | O(1) |

### কেন দরকার?

```
সমস্যা: ১০ টি ডাটায় সব কোড ভালো চলে।
         কিন্তু ১০ লক্ষ ডাটায়?

✅ Big O জানলে → আগেই বুঝবেন কোড স্কেল করবে কিনা
✅ ইন্টারভিউতে → প্রতিটি সমাধানের কমপ্লেক্সিটি জানতে চায়
✅ প্রোডাকশনে → ইউজার বাড়লে সার্ভার ক্র্যাশ ঠেকাতে
```

---

## ২ — 📐 Asymptotic Notation (O, Ω, Θ)

> "Big O শুধু একটি নোটেশন — আসলে তিনটি আছে।"

### 📖 তিনটি নোটেশন

```
╔══════════════════════════════════════════════════════════════╗
║  নোটেশন     বাংলা অর্থ              ব্যাখ্যা               ║
╠══════════════════════════════════════════════════════════════╣
║  O (Big O)   উপরের সীমা (Upper)      সর্বোচ্চ কত সময় লাগবে ║
║  Ω (Omega)   নিচের সীমা (Lower)      সর্বনিম্ন কত সময় লাগবে║
║  Θ (Theta)   আঁটসাঁট সীমা (Tight)    গড় বা সঠিক পরিমাপ     ║
╚══════════════════════════════════════════════════════════════╝
```

### ASCII ডায়াগ্রাম — সীমার ধারণা

```
সময়
 ▲
 │        ╱ O(n²) — Upper Bound (সর্বোচ্চ)
 │       ╱
 │      ╱  ── Θ(n log n) — Tight Bound (প্রকৃত)
 │     ╱  ╱
 │    ╱  ╱
 │   ╱  ╱   ── Ω(n) — Lower Bound (সর্বনিম্ন)
 │  ╱  ╱   ╱
 │ ╱  ╱   ╱
 │╱  ╱   ╱
 ┼──╱───╱──────────▶ ইনপুট সাইজ (n)
```

### PHP উদাহরণ

```php
<?php
// === Asymptotic Notation বোঝা ===

// লিনিয়ার সার্চ → সবগুলো কেসে বিশ্লেষণ
function linearSearch(array $arr, int $target): int {
    // সেরা কেস Ω(1): প্রথম ইলিমেন্টেই পাওয়া
    // গড় কেস Θ(n/2) ≈ Θ(n): মাঝামাঝি কোথাও পাওয়া
    // খারাপ কেস O(n): শেষে পাওয়া বা নেই

    foreach ($arr as $i => $val) {
        if ($val === $target) {
            return $i; // পাওয়া গেছে — ইনডেক্স ফেরত
        }
    }
    return -1; // পাওয়া যায়নি
}

$data = [5, 3, 8, 1, 9, 2, 7];
echo linearSearch($data, 9); // ফলাফল: 4
echo linearSearch($data, 5); // ফলাফল: 0 (সেরা কেস)
```

### JavaScript উদাহরণ

```javascript
// === Asymptotic Notation বোঝা ===

function linearSearch(arr, target) {
    // সেরা কেস Ω(1): প্রথম ইলিমেন্টেই মিলে গেলে
    // খারাপ কেস O(n): পুরো অ্যারে ঘুরতে হলে
    for (let i = 0; i < arr.length; i++) {
        if (arr[i] === target) {
            return i; // পাওয়া গেছে
        }
    }
    return -1; // পাওয়া যায়নি
}

const data = [5, 3, 8, 1, 9, 2, 7];
console.log(linearSearch(data, 9)); // 4
console.log(linearSearch(data, 5)); // 0 (সেরা কেস)
```

---

## ৩ — 📊 সাধারণ Big O ক্লাস

> "প্রতিটি কমপ্লেক্সিটি ক্লাস একটি গতির গল্প বলে।"

### ৩.১ — O(1) — কনস্ট্যান্ট টাইম ⚡

📖 **সংজ্ঞা**: ইনপুট যতই বড় হোক, সময় একই থাকে।

🏠 **বাস্তব**: লাইটের সুইচ টিপুন — ১টা বাল্ব থাকুক বা ১০০টা সুইচ, একটা সুইচ টিপতে একই সময়।

```php
<?php
// O(1) — কনস্ট্যান্ট টাইম
// অ্যারের প্রথম ইলিমেন্ট অ্যাক্সেস

function getFirst(array $arr): mixed {
    return $arr[0]; // সরাসরি ইনডেক্স — সবসময় একই সময়
}

// হ্যাশম্যাপ লুকআপও O(1)
function checkExists(array $map, string $key): bool {
    return isset($map[$key]); // হ্যাশ ফাংশন দিয়ে সরাসরি অ্যাক্সেস
}

$arr = range(1, 1000000);
echo getFirst($arr); // 1 — ১০ লক্ষ ইলিমেন্ট হলেও একই সময়
```

```javascript
// O(1) — কনস্ট্যান্ট টাইম
// অ্যারের যেকোনো ইনডেক্স অ্যাক্সেস

function getFirst(arr) {
    return arr[0]; // সরাসরি মেমরি অ্যাড্রেস — তাৎক্ষণিক
}

// অবজেক্ট প্রপার্টি অ্যাক্সেসও O(1)
function getValue(obj, key) {
    return obj[key]; // হ্যাশ টেবিল লুকআপ
}

const arr = Array.from({length: 1000000}, (_, i) => i);
console.log(getFirst(arr)); // 0 — সাইজ যাই হোক, একই সময়
```

### ৩.২ — O(log n) — লগারিদমিক টাইম 🔍

📖 **সংজ্ঞা**: প্রতিটি ধাপে সমস্যার আকার **অর্ধেক** হয়ে যায়।

🏠 **বাস্তব**: ডিকশনারিতে শব্দ খোঁজা — মাঝখান থেকে শুরু করে অর্ধেক বাদ দেন।

```
ধাপে ধাপে কমে যাওয়া:
n = 16 → 8 → 4 → 2 → 1   (৪ ধাপ = log₂16)
n = 1024 → 512 → ... → 1  (১০ ধাপ = log₂1024)
```

```php
<?php
// O(log n) — বাইনারি সার্চ
// শর্ত: অ্যারে সর্টেড হতে হবে

function binarySearch(array $arr, int $target): int {
    $left = 0;
    $right = count($arr) - 1;

    while ($left <= $right) {
        $mid = intdiv($left + $right, 2); // মাঝের ইনডেক্স
        
        if ($arr[$mid] === $target) {
            return $mid; // পাওয়া গেছে!
        } elseif ($arr[$mid] < $target) {
            $left = $mid + 1; // ডান অর্ধে খুঁজুন
        } else {
            $right = $mid - 1; // বাম অর্ধে খুঁজুন
        }
    }
    return -1; // পাওয়া যায়নি
}

$sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91];
echo binarySearch($sorted, 23); // ফলাফল: 5
// ১০টি ইলিমেন্টে সর্বোচ্চ ⌈log₂10⌉ = 4 বার চেক
```

```javascript
// O(log n) — বাইনারি সার্চ
// সর্টেড অ্যারেতে খুব দ্রুত খোঁজা

function binarySearch(arr, target) {
    let left = 0;
    let right = arr.length - 1;

    while (left <= right) {
        const mid = Math.floor((left + right) / 2); // মাঝখান
        
        if (arr[mid] === target) {
            return mid; // পাওয়া গেছে
        } else if (arr[mid] < target) {
            left = mid + 1; // ডান দিকে যাও
        } else {
            right = mid - 1; // বাম দিকে যাও
        }
    }
    return -1; // নেই
}

const sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91];
console.log(binarySearch(sorted, 23)); // 5
```

### ৩.৩ — O(n) — লিনিয়ার টাইম 📏

📖 **সংজ্ঞা**: সময় ইনপুট সাইজের সাথে সমানুপাতিক হারে বাড়ে।

🏠 **বাস্তব**: বইয়ের প্রতিটি পৃষ্ঠা পড়া — ২০০ পৃষ্ঠা হলে ২০০ বার পড়তে হবে।

```php
<?php
// O(n) — লিনিয়ার টাইম
// অ্যারের সবগুলো ইলিমেন্টের যোগফল

function sumArray(array $arr): int {
    $total = 0;
    foreach ($arr as $num) {
        $total += $num; // প্রতিটি ইলিমেন্ট একবার টাচ
    }
    return $total;
}

// সর্বোচ্চ মান খোঁজা — একবার পুরো অ্যারে ঘুরতে হবে
function findMax(array $arr): int {
    $max = $arr[0]; // প্রথমটাকে সর্বোচ্চ ধরি
    for ($i = 1; $i < count($arr); $i++) {
        if ($arr[$i] > $max) {
            $max = $arr[$i]; // নতুন সর্বোচ্চ পেলাম
        }
    }
    return $max;
}

echo sumArray([1, 2, 3, 4, 5]); // 15
echo findMax([3, 7, 2, 9, 1]); // 9
```

```javascript
// O(n) — লিনিয়ার টাইম
// অ্যারেতে নির্দিষ্ট শর্ত পূরণকারী ইলিমেন্ট ফিল্টার

function filterEven(arr) {
    const result = []; // নতুন অ্যারে
    for (const num of arr) {
        if (num % 2 === 0) {
            result.push(num); // জোড় সংখ্যা রাখো
        }
    }
    return result;
}

// স্ট্রিংয়ে নির্দিষ্ট অক্ষর গণনা
function countChar(str, ch) {
    let count = 0;
    for (const c of str) {
        if (c === ch) count++; // প্রতিটি অক্ষর চেক
    }
    return count;
}

console.log(filterEven([1, 2, 3, 4, 5, 6])); // [2, 4, 6]
console.log(countChar("bangladesh", "a")); // 2
```

### ৩.৪ — O(n log n) — লিনিয়ারিদমিক টাইম 🔀

📖 **সংজ্ঞা**: লিনিয়ার ও লগারিদমিকের গুণফল — দক্ষ সর্টিংয়ের ক্লাস।

🏠 **বাস্তব**: তাসের স্তূপ সর্ট করা Merge Sort পদ্ধতিতে — ভাগ করো ও জয় করো।

```php
<?php
// O(n log n) — মার্জ সর্ট
// ভাগ করো ও জয় করো (Divide & Conquer)

function mergeSort(array $arr): array {
    $n = count($arr);
    if ($n <= 1) return $arr; // বেস কেস — ১টি ইলিমেন্ট ইতিমধ্যে সর্টেড

    $mid = intdiv($n, 2);
    // বাম ও ডান অর্ধ আলাদা করো — log n বার ভাগ হয়
    $left = mergeSort(array_slice($arr, 0, $mid));
    $right = mergeSort(array_slice($arr, $mid));

    return merge($left, $right); // মার্জ করো — প্রতি স্তরে n কাজ
}

function merge(array $left, array $right): array {
    $result = [];
    $i = $j = 0;

    // দুটি সর্টেড অ্যারে মিলিয়ে একটি বানাও
    while ($i < count($left) && $j < count($right)) {
        if ($left[$i] <= $right[$j]) {
            $result[] = $left[$i++]; // বাম থেকে নাও
        } else {
            $result[] = $right[$j++]; // ডান থেকে নাও
        }
    }

    // বাকিগুলো যোগ করো
    while ($i < count($left)) $result[] = $left[$i++];
    while ($j < count($right)) $result[] = $right[$j++];

    return $result;
}

$unsorted = [38, 27, 43, 3, 9, 82, 10];
print_r(mergeSort($unsorted)); // [3, 9, 10, 27, 38, 43, 82]
```

```javascript
// O(n log n) — মার্জ সর্ট (JavaScript)

function mergeSort(arr) {
    if (arr.length <= 1) return arr; // বেস কেস

    const mid = Math.floor(arr.length / 2);
    const left = mergeSort(arr.slice(0, mid));   // বাম অর্ধ
    const right = mergeSort(arr.slice(mid));       // ডান অর্ধ

    return merge(left, right);
}

function merge(left, right) {
    const result = [];
    let i = 0, j = 0;

    // ছোট ইলিমেন্ট আগে বসাও
    while (i < left.length && j < right.length) {
        if (left[i] <= right[j]) {
            result.push(left[i++]);
        } else {
            result.push(right[j++]);
        }
    }

    // বাকি ইলিমেন্ট যোগ করো
    return [...result, ...left.slice(i), ...right.slice(j)];
}

console.log(mergeSort([38, 27, 43, 3, 9, 82, 10]));
// [3, 9, 10, 27, 38, 43, 82]
```

### ৩.৫ — O(n²) — কোয়াড্রাটিক টাইম 🐌

📖 **সংজ্ঞা**: নেস্টেড লুপ — প্রতিটি ইলিমেন্টের জন্য পুরো অ্যারে ঘোরা।

🏠 **বাস্তব**: ক্লাসের সবার সাথে সবার হ্যান্ডশেক — n জন থাকলে n×(n-1)/2 বার।

```php
<?php
// O(n²) — বাবল সর্ট
// প্রতিবার পাশাপাশি তুলনা করে বড়টা ডানে সরাও

function bubbleSort(array $arr): array {
    $n = count($arr);
    for ($i = 0; $i < $n - 1; $i++) {           // বাইরের লুপ: n বার
        for ($j = 0; $j < $n - $i - 1; $j++) {  // ভেতরের লুপ: n বার
            if ($arr[$j] > $arr[$j + 1]) {
                // অদলবদল (swap)
                [$arr[$j], $arr[$j + 1]] = [$arr[$j + 1], $arr[$j]];
            }
        }
    }
    return $arr;
}

// ডুপ্লিকেট খোঁজা — নেস্টেড লুপ পদ্ধতি O(n²)
function hasDuplicate(array $arr): bool {
    $n = count($arr);
    for ($i = 0; $i < $n; $i++) {
        for ($j = $i + 1; $j < $n; $j++) {
            if ($arr[$i] === $arr[$j]) return true; // ডুপ্লিকেট পাওয়া গেছে
        }
    }
    return false;
}

print_r(bubbleSort([64, 34, 25, 12, 22, 11, 90]));
```

```javascript
// O(n²) — সিলেকশন সর্ট
// প্রতিবার সবচেয়ে ছোটটা খুঁজে সামনে আনো

function selectionSort(arr) {
    const n = arr.length;
    for (let i = 0; i < n - 1; i++) {
        let minIdx = i; // ধরে নিই i-তম ইলিমেন্ট সবচেয়ে ছোট
        for (let j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) {
                minIdx = j; // নতুন ছোট পেলাম
            }
        }
        // অদলবদল
        [arr[i], arr[minIdx]] = [arr[minIdx], arr[i]];
    }
    return arr;
}

console.log(selectionSort([64, 25, 12, 22, 11]));
// [11, 12, 22, 25, 64]
```

### ৩.৬ — O(n³) — কিউবিক টাইম 🧊

📖 **সংজ্ঞা**: তিনটি নেস্টেড লুপ — সাধারণত ম্যাট্রিক্স অপারেশনে দেখা যায়।

```php
<?php
// O(n³) — ম্যাট্রিক্স গুণ (Matrix Multiplication)

function matrixMultiply(array $a, array $b): array {
    $n = count($a);
    $result = array_fill(0, $n, array_fill(0, $n, 0));

    for ($i = 0; $i < $n; $i++) {           // ১ম লুপ: সারি
        for ($j = 0; $j < $n; $j++) {       // ২য় লুপ: কলাম
            for ($k = 0; $k < $n; $k++) {   // ৩য় লুপ: গুণ ও যোগ
                $result[$i][$j] += $a[$i][$k] * $b[$k][$j];
            }
        }
    }
    return $result;
}

$a = [[1, 2], [3, 4]];
$b = [[5, 6], [7, 8]];
print_r(matrixMultiply($a, $b));
// [[19, 22], [43, 50]]
```

```javascript
// O(n³) — ম্যাট্রিক্স গুণ (JavaScript)

function matrixMultiply(a, b) {
    const n = a.length;
    // শূন্য দিয়ে ভরা রেজাল্ট ম্যাট্রিক্স
    const result = Array.from({length: n}, () => Array(n).fill(0));

    for (let i = 0; i < n; i++) {       // সারি
        for (let j = 0; j < n; j++) {   // কলাম
            for (let k = 0; k < n; k++) { // গুণফলের যোগ
                result[i][j] += a[i][k] * b[k][j];
            }
        }
    }
    return result;
}

console.log(matrixMultiply([[1,2],[3,4]], [[5,6],[7,8]]));
// [[19, 22], [43, 50]]
```

### ৩.৭ — O(2ⁿ) — এক্সপোনেনশিয়াল টাইম 💥

📖 **সংজ্ঞা**: প্রতিটি ইনপুটে সময় দ্বিগুণ — রিকার্সিভ সমাধানে দেখা যায়।

🏠 **বাস্তব**: প্রতিটি জিনিস নেওয়া বা না-নেওয়ার সিদ্ধান্ত — n টি জিনিসে 2ⁿ সমন্বয়।

```php
<?php
// O(2ⁿ) — ফিবোনাচ্চি (নেইভ রিকার্সিভ)
// প্রতিটি কলে দুটি নতুন কল — এক্সপোনেনশিয়াল বিস্ফোরণ!

function fibonacci(int $n): int {
    if ($n <= 1) return $n; // বেস কেস: fib(0)=0, fib(1)=1
    
    return fibonacci($n - 1) + fibonacci($n - 2); // দুটি শাখা
}

// কল ট্রি (n=5):
//                  fib(5)
//                /        \
//           fib(4)        fib(3)
//          /     \        /     \
//       fib(3)  fib(2)  fib(2)  fib(1)
//      /   \    /  \    /   \
//   fib(2) fib(1) ...  ...  ...

echo fibonacci(10); // 55
// n=40 হলে অনেক সময় লাগবে!
```

```javascript
// O(2ⁿ) — সাবসেট জেনারেশন (Power Set)
// n ইলিমেন্টের সবগুলো সাবসেট বানাও

function subsets(arr) {
    const result = [];
    
    function backtrack(start, current) {
        result.push([...current]); // বর্তমান সাবসেট সেভ করো
        
        for (let i = start; i < arr.length; i++) {
            current.push(arr[i]);       // নাও
            backtrack(i + 1, current);  // পরেরটা ট্রাই করো
            current.pop();              // বাদ দাও (ব্যাকট্র্যাক)
        }
    }
    
    backtrack(0, []);
    return result;
}

console.log(subsets([1, 2, 3]));
// [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
// মোট 2³ = 8 টি সাবসেট
```

### ৩.৮ — O(n!) — ফ্যাক্টোরিয়াল টাইম 🤯

📖 **সংজ্ঞা**: সবচেয়ে ধীর — সব পারমুটেশন চেক করা।

🏠 **বাস্তব**: n শহরে ঘুরে আসার সব সম্ভাব্য রুট ট্রাই করা (TSP)।

```php
<?php
// O(n!) — পারমুটেশন জেনারেশন

function permutations(array $arr, int $start = 0): array {
    $result = [];
    
    if ($start === count($arr) - 1) {
        $result[] = $arr; // একটি সম্পূর্ণ পারমুটেশন পাওয়া গেছে
        return $result;
    }
    
    for ($i = $start; $i < count($arr); $i++) {
        // অদলবদল
        [$arr[$start], $arr[$i]] = [$arr[$i], $arr[$start]];
        // বাকি অংশের পারমুটেশন
        $result = array_merge($result, permutations($arr, $start + 1));
        // আবার আগের জায়গায় ফেরত (ব্যাকট্র্যাক)
        [$arr[$start], $arr[$i]] = [$arr[$i], $arr[$start]];
    }
    
    return $result;
}

$perms = permutations([1, 2, 3]);
// মোট 3! = 6 টি পারমুটেশন
// [1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,2,1], [3,1,2]
echo count($perms); // 6
```

```javascript
// O(n!) — পারমুটেশন (JavaScript)

function permutations(arr) {
    const result = [];
    
    function permute(current, remaining) {
        if (remaining.length === 0) {
            result.push([...current]); // সম্পূর্ণ পারমুটেশন
            return;
        }
        
        for (let i = 0; i < remaining.length; i++) {
            current.push(remaining[i]); // একটা বাছো
            // বাকিগুলো নিয়ে এগিয়ে যাও
            permute(current, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
            current.pop(); // ব্যাকট্র্যাক
        }
    }
    
    permute([], arr);
    return result;
}

console.log(permutations([1, 2, 3]).length); // 6
// n=10 হলে 3,628,800 টি পারমুটেশন — অত্যন্ত ধীর!
```

---

## ৪ — 📈 Growth Rate তুলনা

> "সংখ্যা দিয়ে দেখলেই বুঝবেন পার্থক্যটা কত বিশাল।"

### সংখ্যাসূচক তুলনা টেবিল

```
╔═══════════╤══════════╤══════════╤══════════════╤══════════════╤═══════════════╤══════════════╗
║ Big O     │ n=10     │ n=100    │ n=1,000      │ n=10,000     │ n=100,000     │ n=1,000,000  ║
╠═══════════╪══════════╪══════════╪══════════════╪══════════════╪═══════════════╪══════════════╣
║ O(1)      │ 1        │ 1        │ 1            │ 1            │ 1             │ 1            ║
║ O(log n)  │ 3        │ 7        │ 10           │ 13           │ 17            │ 20           ║
║ O(n)      │ 10       │ 100      │ 1,000        │ 10,000       │ 100,000       │ 1,000,000    ║
║ O(n log n)│ 33       │ 664      │ 9,966        │ 132,877      │ 1,660,964     │ 19,931,569   ║
║ O(n²)     │ 100      │ 10,000   │ 1,000,000    │ 100,000,000  │ 10,000,000,000│ 10¹²         ║
║ O(n³)     │ 1,000    │ 1,000,000│ 10⁹          │ 10¹²         │ 10¹⁵          │ 10¹⁸         ║
║ O(2ⁿ)     │ 1,024    │ 10³⁰     │ 10³⁰¹        │ ∞            │ ∞             │ ∞            ║
║ O(n!)     │ 3,628,800│ ∞        │ ∞            │ ∞            │ ∞             │ ∞            ║
╚═══════════╧══════════╧══════════╧══════════════╧══════════════╧═══════════════╧══════════════╝
```

### ASCII Growth Chart

```
অপারেশন
সংখ্যা ▲
         │                                          ╱ O(n!)
         │                                        ╱
    10⁶  │                                      ╱   ╱ O(2ⁿ)
         │                                    ╱   ╱
         │                                  ╱   ╱
         │                                ╱   ╱
    10⁴  │                              ╱   ╱
         │                     ╱──── O(n²)
         │                   ╱  ╱
         │                 ╱  ╱
    10²  │               ╱  ╱
         │        ╱──── O(n log n)
         │      ╱ ╱
         │    ╱ ╱──── O(n)
    10   │  ╱ ╱
         │╱ ╱────── O(log n)
         │╱
      1  │━━━━━━━━━ O(1)
         ┼────┬────┬────┬────┬────▶ ইনপুট সাইজ (n)
              10   100  1K   10K
```

### কমপ্লেক্সিটি জোন

```
╔════════════════════════════════════════════════════════╗
║  🟢 চমৎকার    │ O(1), O(log n)                        ║
║  🟡 ভালো      │ O(n)                                  ║
║  🟠 মোটামুটি  │ O(n log n)                             ║
║  🔴 খারাপ     │ O(n²), O(n³)                          ║
║  ☠️  বিপর্যয়  │ O(2ⁿ), O(n!)                          ║
╚════════════════════════════════════════════════════════╝
```

---

## ৫ — 🧮 Big O হিসাব করার নিয়ম

> "চারটি নিয়ম মনে রাখলেই Big O হিসাব করা সোজা।"

### নিয়ম ১: ধ্রুবক বাদ দাও (Drop Constants)

```php
<?php
// এই ফাংশনটি কি O(2n)?
function printTwice(array $arr): void {
    // প্রথম লুপ — n বার
    foreach ($arr as $val) {
        echo $val . " ";
    }
    // দ্বিতীয় লুপ — আরো n বার
    foreach ($arr as $val) {
        echo $val * 2 . " ";
    }
}
// মোট: n + n = 2n → কিন্তু Big O তে ধ্রুবক বাদ → O(n)
// কারণ: n যখন অনেক বড়, 2 এর গুণক তুচ্ছ
```

### নিয়ম ২: ছোট পদ বাদ দাও (Drop Non-Dominant Terms)

```javascript
// এই ফাংশনটি কি O(n² + n)?
function processData(arr) {
    // O(n²) — নেস্টেড লুপ
    for (let i = 0; i < arr.length; i++) {
        for (let j = 0; j < arr.length; j++) {
            console.log(arr[i] + arr[j]); // জোড়া তৈরি
        }
    }
    
    // O(n) — সিঙ্গেল লুপ
    for (let k = 0; k < arr.length; k++) {
        console.log(arr[k]); // একক প্রিন্ট
    }
}
// মোট: O(n² + n) → ছোট পদ বাদ → O(n²)
// কারণ: n বড় হলে n² এর কাছে n তুচ্ছ
```

### নিয়ম ৩: ভিন্ন ইনপুটে ভিন্ন ভেরিয়েবল

```php
<?php
// দুটি ভিন্ন অ্যারে — একটি n, আরেকটি m
function printBoth(array $arrA, array $arrB): void {
    // প্রথম অ্যারে ঘোরা — O(n)
    foreach ($arrA as $a) {
        echo $a . " ";
    }
    // দ্বিতীয় অ্যারে ঘোরা — O(m)
    foreach ($arrB as $b) {
        echo $b . " ";
    }
}
// ⚠️ ভুল: O(n) নয়!
// ✅ সঠিক: O(n + m) — কারণ দুটি আলাদা ইনপুট

// নেস্টেড হলে?
function compare(array $arrA, array $arrB): void {
    foreach ($arrA as $a) {        // O(n)
        foreach ($arrB as $b) {    // O(m)
            if ($a === $b) echo "মিল!";
        }
    }
}
// এটি O(n × m) — O(n²) নয়! (যদি না n = m হয়)
```

### নিয়ম ৪: লুপ নেস্টিং গুণ করো

```javascript
// নেস্টেড লুপ = গুণ, পরপর লুপ = যোগ

// পরপর (Sequential) → যোগ
function sequential(arr) {
    for (const x of arr) { /* O(n) */ }  // ← n
    for (const x of arr) { /* O(n) */ }  // + n
}                                         // = O(2n) = O(n)

// নেস্টেড → গুণ
function nested(arr) {
    for (const x of arr) {         // ← n
        for (const y of arr) {     // × n
            console.log(x, y);
        }
    }
}                                   // = O(n × n) = O(n²)

// তিন স্তর নেস্টেড → ত্রিগুণ
function tripleNested(arr) {
    for (const x of arr) {              // n
        for (const y of arr) {          // × n
            for (const z of arr) {      // × n
                console.log(x, y, z);
            }
        }
    }
}                                        // = O(n³)
```

### সারসংক্ষেপ ফর্মুলা

```
╔══════════════════════════════════════════════════╗
║  নিয়ম                    উদাহরণ → Big O         ║
╠══════════════════════════════════════════════════╣
║  ধ্রুবক বাদ দাও         5n → O(n)               ║
║  ছোট পদ বাদ দাও        n² + n → O(n²)           ║
║  ভিন্ন ইনপুট আলাদা      n + m → O(n + m)        ║
║  নেস্টেড = গুণ          n × n → O(n²)            ║
║  পরপর = যোগ            n + n → O(n)              ║
║  লুপের ভেতর লগ         n × log n → O(n log n)   ║
╚══════════════════════════════════════════════════╝
```

---

## ৬ — 🏋️ কমপ্লেক্সিটি বিশ্লেষণ প্র্যাক্টিস

> "প্র্যাক্টিস ছাড়া কমপ্লেক্সিটি বিশ্লেষণ আয়ত্ত হবে না।"

### প্র্যাক্টিস ১: অ্যারে রিভার্স

```php
<?php
function reverseArray(array &$arr): void {
    $left = 0;
    $right = count($arr) - 1;
    while ($left < $right) {
        [$arr[$left], $arr[$right]] = [$arr[$right], $arr[$left]];
        $left++;
        $right--;
    }
}
// বিশ্লেষণ: একটি while লুপ, n/2 বার চলে → O(n/2) → O(n)
// স্পেস: ইন-প্লেস, অতিরিক্ত অ্যারে নেই → O(1)
```

### প্র্যাক্টিস ২: টু সাম (Two Sum)

```javascript
// পদ্ধতি ১: ব্রুট ফোর্স — O(n²)
function twoSumBrute(arr, target) {
    for (let i = 0; i < arr.length; i++) {
        for (let j = i + 1; j < arr.length; j++) {
            if (arr[i] + arr[j] === target) {
                return [i, j]; // জোড়া পাওয়া গেছে
            }
        }
    }
    return null;
}

// পদ্ধতি ২: হ্যাশম্যাপ — O(n) ✅ অনেক ভালো!
function twoSumHash(arr, target) {
    const map = {}; // পরিপূরক সংখ্যা সেভ রাখি
    for (let i = 0; i < arr.length; i++) {
        const complement = target - arr[i];
        if (complement in map) {
            return [map[complement], i]; // পরিপূরক আগেই সেভ ছিল
        }
        map[arr[i]] = i; // বর্তমান সংখ্যা ও ইনডেক্স সেভ করো
    }
    return null;
}
// বিশ্লেষণ: একটি লুপ + হ্যাশম্যাপ O(1) লুকআপ → O(n)
// স্পেস: হ্যাশম্যাপে n টি এন্ট্রি → O(n)
```

### প্র্যাক্টিস ৩: স্ট্রিং ম্যাচিং

```php
<?php
function containsSubstring(string $text, string $pattern): bool {
    $n = strlen($text);
    $m = strlen($pattern);
    
    for ($i = 0; $i <= $n - $m; $i++) {       // (n - m + 1) বার
        $match = true;
        for ($j = 0; $j < $m; $j++) {         // m বার
            if ($text[$i + $j] !== $pattern[$j]) {
                $match = false;
                break;
            }
        }
        if ($match) return true;
    }
    return false;
}
// বিশ্লেষণ: বাইরের লুপ O(n), ভেতরের O(m) → O(n × m)
// সেরা কেস: প্রথম অক্ষরেই মিল নেই → O(n)
```

### প্র্যাক্টিস ৪: ফিবোনাচ্চি — তিন পদ্ধতি

```javascript
// পদ্ধতি ১: রিকার্সিভ — O(2ⁿ) ❌
function fibRecursive(n) {
    if (n <= 1) return n;
    return fibRecursive(n - 1) + fibRecursive(n - 2);
}

// পদ্ধতি ২: মেমোইজেশন — O(n) ✅ (টাইম-স্পেস ট্রেডঅফ)
function fibMemo(n, memo = {}) {
    if (n in memo) return memo[n]; // আগে হিসাব করা থাকলে সরাসরি ফেরত
    if (n <= 1) return n;
    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    return memo[n];
}

// পদ্ধতি ৩: ইটারেটিভ — O(n) টাইম, O(1) স্পেস ✅✅
function fibIterative(n) {
    if (n <= 1) return n;
    let prev = 0, curr = 1;
    for (let i = 2; i <= n; i++) {
        [prev, curr] = [curr, prev + curr]; // শুধু আগের দুটো মনে রাখো
    }
    return curr;
}

console.log(fibIterative(50)); // 12586269025 — তাৎক্ষণিক!
```

### প্র্যাক্টিস ৫: লগ বেসড প্যাটার্ন

```php
<?php
// লুপ ভেরিয়েবল দ্বিগুণ হচ্ছে → O(log n)
function logPattern(int $n): void {
    $i = 1;
    while ($i < $n) {
        echo $i . " "; // 1, 2, 4, 8, 16, ...
        $i *= 2;       // প্রতিবার দ্বিগুণ
    }
}
// বিশ্লেষণ: i দ্বিগুণ হচ্ছে → log₂n ধাপ → O(log n)

// লুপ ভেরিয়েবল অর্ধেক হচ্ছে → O(log n)
function logPatternReverse(int $n): void {
    $i = $n;
    while ($i >= 1) {
        echo $i . " "; // n, n/2, n/4, ..., 1
        $i = intdiv($i, 2); // প্রতিবার অর্ধেক
    }
}
```

### প্র্যাক্টিস ৬: নেস্টেড লুপে ভিন্ন সীমা

```javascript
// ভেতরের লুপ বাইরের উপর নির্ভরশীল
function trianglePattern(n) {
    for (let i = 0; i < n; i++) {
        for (let j = 0; j < i; j++) {  // j চলে 0 থেকে i পর্যন্ত
            process(i, j);
        }
    }
}
// বিশ্লেষণ: 0 + 1 + 2 + ... + (n-1) = n(n-1)/2 → O(n²)

function process(i, j) { /* কিছু কাজ */ }
```

### প্র্যাক্টিস ৭: রিকার্সিভ ফাংশন

```php
<?php
// রিকার্সিভ যোগফল
function recursiveSum(array $arr, int $n): int {
    if ($n <= 0) return 0; // বেস কেস
    return $arr[$n - 1] + recursiveSum($arr, $n - 1); // একটি রিকার্সিভ কল
}
// বিশ্লেষণ: n বার কল → O(n) টাইম
// স্ট্যাক: n ফ্রেম → O(n) স্পেস
```

### প্র্যাক্টিস ৮: স্ট্রিং বিল্ডিং

```javascript
// ❌ অদক্ষ — O(n²)
function buildStringBad(n) {
    let str = "";
    for (let i = 0; i < n; i++) {
        str += "a"; // প্রতিবার নতুন স্ট্রিং কপি হচ্ছে!
    }
    return str;
}
// 1 + 2 + 3 + ... + n কপি → O(n²)

// ✅ দক্ষ — O(n)
function buildStringGood(n) {
    const parts = [];
    for (let i = 0; i < n; i++) {
        parts.push("a"); // অ্যারেতে যোগ — O(1) amortized
    }
    return parts.join(""); // একবারে জোড়া — O(n)
}
```

### প্র্যাক্টিস ৯: সর্ট করে বাইনারি সার্চ

```php
<?php
function sortAndSearch(array $arr, int $target): bool {
    sort($arr);                    // সর্ট → O(n log n)
    // তারপর বাইনারি সার্চ → O(log n)
    $left = 0;
    $right = count($arr) - 1;
    while ($left <= $right) {
        $mid = intdiv($left + $right, 2);
        if ($arr[$mid] === $target) return true;
        elseif ($arr[$mid] < $target) $left = $mid + 1;
        else $right = $mid - 1;
    }
    return false;
}
// মোট: O(n log n) + O(log n) → O(n log n) (বড় পদ রাখো)
```

### প্র্যাক্টিস ১০: গ্রাফ BFS

```javascript
// BFS — O(V + E) যেখানে V = নোড, E = এজ
function bfs(graph, start) {
    const visited = new Set();
    const queue = [start];
    visited.add(start);
    
    while (queue.length > 0) {
        const node = queue.shift(); // প্রতিটি নোড একবার — O(V)
        console.log(node);
        
        for (const neighbor of graph[node] || []) { // প্রতিটি এজ একবার — O(E)
            if (!visited.has(neighbor)) {
                visited.add(neighbor);
                queue.push(neighbor);
            }
        }
    }
}
// বিশ্লেষণ: প্রতিটি নোড ও এজ একবার করে ভিজিট → O(V + E)
```

---

## ৭ — 💾 Space Complexity

> "সময় ছাড়াও মেমরি খরচ বোঝা সমান গুরুত্বপূর্ণ।"

### 📖 সংজ্ঞা

**Space Complexity** = অ্যালগরিদম চলাকালীন কতটুকু **অতিরিক্ত মেমরি** লাগে।

### Auxiliary Space vs Total Space

```
╔═══════════════════════════════════════════════════════╗
║  ধরন              ব্যাখ্যা                            ║
╠═══════════════════════════════════════════════════════╣
║  Auxiliary Space   শুধু অতিরিক্ত মেমরি (ইনপুট বাদে)  ║
║  Total Space       ইনপুট + অতিরিক্ত মেমরি             ║
╚═══════════════════════════════════════════════════════╝

উদাহরণ: Merge Sort
  - ইনপুট: n সাইজের অ্যারে → O(n)
  - অতিরিক্ত: মার্জের সময় আরো n → O(n)
  - Total Space: O(2n) = O(n)
  - Auxiliary Space: O(n)
```

### স্পেস কমপ্লেক্সিটি উদাহরণ

```php
<?php
// O(1) স্পেস — ইন-প্লেস অদলবদল
function swap(array &$arr, int $i, int $j): void {
    $temp = $arr[$i];       // মাত্র ১টি অতিরিক্ত ভেরিয়েবল
    $arr[$i] = $arr[$j];
    $arr[$j] = $temp;
}

// O(n) স্পেস — নতুন অ্যারে তৈরি
function doubleValues(array $arr): array {
    $result = [];            // n সাইজের নতুন অ্যারে
    foreach ($arr as $val) {
        $result[] = $val * 2;
    }
    return $result;
}

// O(n²) স্পেস — 2D অ্যারে তৈরি
function create2DMatrix(int $n): array {
    $matrix = [];
    for ($i = 0; $i < $n; $i++) {
        $matrix[$i] = array_fill(0, $n, 0); // n × n ম্যাট্রিক্স
    }
    return $matrix;
}
```

### রিকার্শন স্ট্যাক স্পেস

```javascript
// রিকার্শনে কল স্ট্যাকও মেমরি খরচ করে!

// O(n) স্ট্যাক স্পেস
function factorial(n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
// কল স্ট্যাক: factorial(5) → factorial(4) → ... → factorial(1)
// ৫টি ফ্রেম মেমরিতে একসাথে থাকে → O(n)

// O(log n) স্ট্যাক স্পেস
function binarySearchRecursive(arr, target, left = 0, right = arr.length - 1) {
    if (left > right) return -1;
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, right);
    return binarySearchRecursive(arr, target, left, mid - 1);
}
// প্রতিটি কলে অর্ধেক → log n গভীরতা → O(log n) স্ট্যাক

// O(n) স্ট্যাক (ট্রি রিকার্শন — কিন্তু একসময়ে সর্বোচ্চ n গভীর)
function treeHeight(node) {
    if (!node) return 0;
    return 1 + Math.max(treeHeight(node.left), treeHeight(node.right));
}
```

### 🏠 Space-Time Tradeoff

```
╔══════════════════════════════════════════════════════════════╗
║  কৌশল         সময়          স্পেস      ব্যাখ্যা              ║
╠══════════════════════════════════════════════════════════════╣
║  ব্রুট ফোর্স  O(n²)        O(1)      মেমরি কম, সময় বেশি    ║
║  হ্যাশম্যাপ   O(n)         O(n)      মেমরি বেশি, সময় কম    ║
║  সর্টিং       O(n log n)   O(1)*     মাঝামাঝি               ║
║  ক্যাশিং      O(1) lookup  O(n)      পুনরায় হিসাব বাঁচায়    ║
╚══════════════════════════════════════════════════════════════╝
* ইন-প্লেস সর্ট ধরে

💡 মনে রাখুন: সময় ও স্পেস দুটোই শূন্য করা যায় না — একটা বাড়ালে
   অন্যটা কমে। এটাই ট্রেডঅফ!
```

---

## ৮ — ⚖️ Amortized Analysis

> "মাঝে মাঝে ব্যয়বহুল অপারেশন হলেও গড়ে সস্তা — এটাই অ্যামোর্টাইজড।"

### 📖 সংজ্ঞা

**Amortized Analysis** = একগুচ্ছ অপারেশনের **গড় খরচ** হিসাব করা,
যেখানে কিছু অপারেশন সস্তা আর কিছু দামি।

### 🏠 বাস্তব উদাহরণ

কল্পনা করুন আপনি **কয়েন জমা** করছেন:
- প্রতিদিন ১ টাকা রাখেন (সস্তা)
- ৩০ দিনে একবার ব্যাংকে যান (দামি)
- গড়ে: প্রতিদিন ≈ ১.০৩ টাকা খরচ

### Dynamic Array (ArrayList) রিসাইজিং

```
অপারেশন:  [1] [2] [3] [4] [5] [6] [7] [8] [9]
সাইজ:      1   2   4   4   8   8   8   8   16
কপি খরচ:   -   1   2   -   4   -   -   -   8

╔══════════════════════════════════════════════════╗
║  পুশ #  ক্যাপাসিটি  কপি খরচ  ইনসার্ট  মোট খরচ   ║
╠══════════════════════════════════════════════════╣
║  1      1          0       1       1            ║
║  2      2          1       1       2            ║
║  3      4          2       1       3            ║
║  4      4          0       1       1            ║
║  5      8          4       1       5            ║
║  6      8          0       1       1            ║
║  7      8          0       1       1            ║
║  8      8          0       1       1            ║
║  9      16         8       1       9            ║
╠══════════════════════════════════════════════════╣
║  মোট: 24 অপারেশনে ≈ 9 পুশ → গড় ≈ 2.67        ║
║  অ্যামোর্টাইজড: O(1) প্রতি পুশে                 ║
╚══════════════════════════════════════════════════╝
```

### PHP তে ডাইনামিক অ্যারে

```php
<?php
// PHP এর array স্বয়ংক্রিয়ভাবে রিসাইজ হয়
// কিন্তু ধারণাটি বোঝার জন্য ম্যানুয়াল ইমপ্লিমেন্টেশন:

class DynamicArray {
    private array $data;
    private int $size;
    private int $capacity;

    public function __construct() {
        $this->data = [];
        $this->size = 0;
        $this->capacity = 1; // শুরুতে ১ জায়গা
    }

    public function push(mixed $value): void {
        // ক্যাপাসিটি ভর্তি? → দ্বিগুণ করো (দামি অপারেশন — O(n))
        if ($this->size === $this->capacity) {
            $this->resize();
        }
        $this->data[$this->size] = $value; // সস্তা অপারেশন — O(1)
        $this->size++;
    }

    private function resize(): void {
        $this->capacity *= 2; // দ্বিগুণ ক্যাপাসিটি
        $newData = [];
        // পুরানো ডাটা কপি — O(n)
        for ($i = 0; $i < $this->size; $i++) {
            $newData[$i] = $this->data[$i];
        }
        $this->data = $newData;
        echo "রিসাইজ হলো! নতুন ক্যাপাসিটি: {$this->capacity}\n";
    }

    public function getSize(): int {
        return $this->size;
    }
}

$arr = new DynamicArray();
for ($i = 0; $i < 20; $i++) {
    $arr->push($i);
}
// রিসাইজ ঘটবে: 1→2, 2→4, 4→8, 8→16, 16→32
// ২০ পুশে মাত্র ৫ বার রিসাইজ → অ্যামোর্টাইজড O(1)
```

```javascript
// JavaScript তে অ্যামোর্টাইজড বিশ্লেষণ ডেমো

class DynamicArray {
    constructor() {
        this.data = new Array(1); // শুরুতে ১ জায়গা
        this.size = 0;
        this.capacity = 1;
        this.resizeCount = 0; // কতবার রিসাইজ হলো
    }

    push(value) {
        if (this.size === this.capacity) {
            this._resize(); // দ্বিগুণ করো
        }
        this.data[this.size] = value;
        this.size++;
    }

    _resize() {
        this.capacity *= 2;
        const newData = new Array(this.capacity);
        for (let i = 0; i < this.size; i++) {
            newData[i] = this.data[i]; // পুরানো ডাটা কপি
        }
        this.data = newData;
        this.resizeCount++;
    }
}

const arr = new DynamicArray();
for (let i = 0; i < 1000; i++) {
    arr.push(i);
}
console.log(`সাইজ: ${arr.size}, রিসাইজ: ${arr.resizeCount} বার`);
// সাইজ: 1000, রিসাইজ: 10 বার (log₂1000 ≈ 10)
// ১০০০ পুশে মাত্র ১০ বার রিসাইজ!
```

---

## ৯ — 🎲 Best, Average, Worst Case

> "একই অ্যালগরিদম ভিন্ন ইনপুটে ভিন্ন আচরণ করে।"

### 📖 তিনটি কেস

```
╔══════════════════════════════════════════════════════════╗
║  কেস          বাংলা          ব্যাখ্যা                    ║
╠══════════════════════════════════════════════════════════╣
║  Best Case    সেরা কেস       সবচেয়ে ভালো ইনপুটে         ║
║  Average Case গড় কেস        সাধারণ/র্যান্ডম ইনপুটে       ║
║  Worst Case   খারাপ কেস     সবচেয়ে খারাপ ইনপুটে         ║
╚══════════════════════════════════════════════════════════╝
```

### Quick Sort — তিন কেসের আদর্শ উদাহরণ

```
Best Case — O(n log n):
    পিভট সবসময় মাঝখানে পড়ে → সমান ভাগ

    [3, 1, 4, 1, 5, 9, 2, 6]
         পিভট = 4
    [3,1,1,2] [4] [5,9,6]    ← সমান ভাগ
       ↙  ↘       ↙  ↘
    [1,1][3,2] [5,6][9]      ← log n স্তর

Average Case — O(n log n):
    র্যান্ডম ইনপুটে সাধারণত ভালো ভাগ হয়

Worst Case — O(n²):
    ইতিমধ্যে সর্টেড অ্যারে + প্রথম/শেষ ইলিমেন্ট পিভট

    [1, 2, 3, 4, 5, 6, 7, 8]
     পিভট = 1
    [] [1] [2,3,4,5,6,7,8]   ← একদিকে সব!
          পিভট = 2
       [] [2] [3,4,5,6,7,8]  ← আবার একদিকে!
    ... n স্তর × n কাজ = O(n²)
```

### PHP তে Quick Sort

```php
<?php
// Quick Sort — গড়ে O(n log n), খারাপ কেসে O(n²)

function quickSort(array $arr): array {
    $n = count($arr);
    if ($n <= 1) return $arr; // বেস কেস

    // পিভট বাছাই — মাঝখানের ইলিমেন্ট (worst case এড়াতে)
    $pivotIndex = intdiv($n, 2);
    $pivot = $arr[$pivotIndex];
    
    $left = $right = $equal = [];
    
    foreach ($arr as $val) {
        if ($val < $pivot) {
            $left[] = $val;      // পিভটের চেয়ে ছোট → বামে
        } elseif ($val > $pivot) {
            $right[] = $val;     // পিভটের চেয়ে বড় → ডানে
        } else {
            $equal[] = $val;     // পিভটের সমান → মাঝে
        }
    }

    // বাম সর্ট + মাঝ + ডান সর্ট
    return array_merge(quickSort($left), $equal, quickSort($right));
}

// সেরা কেস: সুষম ভাগ হওয়া
$best = [5, 3, 8, 1, 9, 2, 7, 4, 6];
echo implode(", ", quickSort($best)) . "\n";

// খারাপ কেস: ইতিমধ্যে সর্টেড (যদি প্রথম ইলিমেন্ট পিভট হতো)
$worst = range(1, 100);
echo implode(", ", quickSort($worst)) . "\n";
// মাঝখানের পিভট বেছে নেওয়ায় worst case এড়ানো সম্ভব
```

```javascript
// Quick Sort — JavaScript ইমপ্লিমেন্টেশন

function quickSort(arr) {
    if (arr.length <= 1) return arr;

    // র্যান্ডম পিভট — worst case এড়ানোর সেরা উপায়
    const pivotIdx = Math.floor(Math.random() * arr.length);
    const pivot = arr[pivotIdx];
    
    const left = [];   // পিভটের চেয়ে ছোট
    const right = [];  // পিভটের চেয়ে বড়
    const equal = [];  // পিভটের সমান

    for (const val of arr) {
        if (val < pivot) left.push(val);
        else if (val > pivot) right.push(val);
        else equal.push(val);
    }

    return [...quickSort(left), ...equal, ...quickSort(right)];
}

// পারফরম্যান্স টেস্ট
function measureSort(arr, name) {
    const start = performance.now();
    quickSort([...arr]); // কপি করে সর্ট
    const end = performance.now();
    console.log(`${name}: ${(end - start).toFixed(2)}ms`);
}

// বিভিন্ন কেস টেস্ট
const random = Array.from({length: 10000}, () => Math.floor(Math.random() * 10000));
const sorted = Array.from({length: 10000}, (_, i) => i);
const reversed = Array.from({length: 10000}, (_, i) => 10000 - i);

measureSort(random, "র্যান্ডম (গড় কেস)");
measureSort(sorted, "সর্টেড");
measureSort(reversed, "রিভার্সড");
```

### সাধারণ অ্যালগরিদমের তিন কেস তুলনা

```
╔══════════════════════════════════════════════════════════════════╗
║  অ্যালগরিদম       সেরা         গড়          খারাপ       স্পেস  ║
╠══════════════════════════════════════════════════════════════════╣
║  Linear Search    O(1)        O(n)        O(n)        O(1)    ║
║  Binary Search    O(1)        O(log n)    O(log n)    O(1)    ║
║  Bubble Sort      O(n)        O(n²)       O(n²)       O(1)    ║
║  Insertion Sort   O(n)        O(n²)       O(n²)       O(1)    ║
║  Merge Sort       O(n log n)  O(n log n)  O(n log n)  O(n)    ║
║  Quick Sort       O(n log n)  O(n log n)  O(n²)       O(log n)║
║  Heap Sort        O(n log n)  O(n log n)  O(n log n)  O(1)    ║
║  Hash Table Get   O(1)        O(1)        O(n)        O(n)    ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## ১০ — 🛠️ প্র্যাক্টিক্যাল প্রজেক্ট: পারফরম্যান্স অ্যানালাইজার

> "তত্ত্ব জানলেই হবে না — নিজে পরিমাপ করতে শিখুন!"

### প্রজেক্ট: কোড পারফরম্যান্স তুলনাকারী

```php
<?php
// === পারফরম্যান্স অ্যানালাইজার — PHP ===

class PerformanceAnalyzer {
    
    // সময় পরিমাপ করার সাধারণ মেথড
    public static function measure(callable $fn, string $label): float {
        $start = microtime(true);
        $fn(); // ফাংশন চালাও
        $end = microtime(true);
        $elapsed = ($end - $start) * 1000; // মিলিসেকেন্ডে
        echo sprintf("⏱️ %s: %.4f ms\n", $label, $elapsed);
        return $elapsed;
    }

    // বিভিন্ন সাইজে পরীক্ষা করো
    public static function benchmark(callable $fn, string $label, array $sizes = [100, 1000, 10000]): void {
        echo "\n📊 বেঞ্চমার্ক: $label\n";
        echo str_repeat("─", 50) . "\n";

        $results = [];
        foreach ($sizes as $n) {
            $data = range(1, $n);
            shuffle($data); // র্যান্ডম অর্ডার

            $start = microtime(true);
            $fn($data);
            $end = microtime(true);
            
            $time = ($end - $start) * 1000;
            $results[] = ['size' => $n, 'time' => $time];
            echo sprintf("  n=%-8d → %10.4f ms\n", $n, $time);
        }

        // গ্রোথ রেট বিশ্লেষণ
        echo "\n📈 গ্রোথ রেট:\n";
        for ($i = 1; $i < count($results); $i++) {
            $ratio = $results[$i]['time'] / max($results[$i-1]['time'], 0.0001);
            $sizeRatio = $results[$i]['size'] / $results[$i-1]['size'];
            echo sprintf("  n×%d → সময় ×%.1f\n", $sizeRatio, $ratio);
        }
    }
}

// ========== পরীক্ষা চালানো ==========

// O(n) — লিনিয়ার সার্চ
PerformanceAnalyzer::benchmark(
    fn($data) => array_search(count($data), $data),
    "লিনিয়ার সার্চ (Worst Case)"
);

// O(n²) — বাবল সর্ট
PerformanceAnalyzer::benchmark(
    function($data) {
        $n = count($data);
        for ($i = 0; $i < $n; $i++) {
            for ($j = 0; $j < $n - $i - 1; $j++) {
                if ($data[$j] > $data[$j+1]) {
                    [$data[$j], $data[$j+1]] = [$data[$j+1], $data[$j]];
                }
            }
        }
    },
    "বাবল সর্ট O(n²)",
    [100, 500, 1000, 2000]
);

// O(n log n) — বিল্ট-ইন সর্ট
PerformanceAnalyzer::benchmark(
    fn($data) => sort($data),
    "PHP sort() — O(n log n)"
);
```

```javascript
// === পারফরম্যান্স অ্যানালাইজার — JavaScript ===

class PerformanceAnalyzer {
    
    // সময় পরিমাপ
    static measure(fn, label) {
        const start = performance.now();
        const result = fn();
        const end = performance.now();
        const elapsed = end - start;
        console.log(`⏱️ ${label}: ${elapsed.toFixed(4)} ms`);
        return { elapsed, result };
    }

    // বিভিন্ন সাইজে বেঞ্চমার্ক
    static benchmark(fn, label, sizes = [100, 1000, 10000]) {
        console.log(`\n📊 বেঞ্চমার্ক: ${label}`);
        console.log("─".repeat(50));

        const results = [];
        for (const n of sizes) {
            // র্যান্ডম ডাটা তৈরি
            const data = Array.from({length: n}, () => Math.floor(Math.random() * n));

            const start = performance.now();
            fn([...data]); // কপি দিয়ে চালাও
            const end = performance.now();

            const time = end - start;
            results.push({ size: n, time });
            console.log(`  n=${String(n).padEnd(8)} → ${time.toFixed(4).padStart(10)} ms`);
        }

        // গ্রোথ রেট বিশ্লেষণ
        console.log("\n📈 গ্রোথ রেট:");
        for (let i = 1; i < results.length; i++) {
            const ratio = results[i].time / Math.max(results[i-1].time, 0.0001);
            const sizeRatio = results[i].size / results[i-1].size;
            console.log(`  n×${sizeRatio} → সময় ×${ratio.toFixed(1)}`);
        }
    }

    // কমপ্লেক্সিটি অনুমান করো
    static guessComplexity(fn, sizes = [100, 500, 1000, 5000, 10000]) {
        console.log("\n🔍 কমপ্লেক্সিটি অনুমান:");
        const times = [];

        for (const n of sizes) {
            const data = Array.from({length: n}, () => Math.floor(Math.random() * n));
            const start = performance.now();
            fn([...data]);
            const elapsed = performance.now() - start;
            times.push({ n, time: elapsed });
        }

        // O(1) চেক: সময় প্রায় একই
        // O(n) চেক: সময়/n প্রায় একই
        // O(n²) চেক: সময়/n² প্রায় একই
        // O(n log n) চেক: সময়/(n log n) প্রায় একই

        const ratios = {
            "O(1)": times.map(t => t.time),
            "O(n)": times.map(t => t.time / t.n),
            "O(n log n)": times.map(t => t.time / (t.n * Math.log2(t.n))),
            "O(n²)": times.map(t => t.time / (t.n * t.n)),
        };

        let bestGuess = "অজানা";
        let bestVariance = Infinity;

        for (const [name, vals] of Object.entries(ratios)) {
            const avg = vals.reduce((a, b) => a + b) / vals.length;
            const variance = vals.reduce((sum, v) => sum + (v - avg) ** 2, 0) / vals.length;
            const cv = Math.sqrt(variance) / Math.max(avg, 0.0001); // ভ্যারিয়েশন সহগ

            if (cv < bestVariance) {
                bestVariance = cv;
                bestGuess = name;
            }
        }

        console.log(`  🎯 অনুমান: ${bestGuess}`);
        return bestGuess;
    }
}

// ========== পরীক্ষা ==========

// বাবল সর্ট
PerformanceAnalyzer.benchmark(
    (arr) => {
        for (let i = 0; i < arr.length; i++)
            for (let j = 0; j < arr.length - i - 1; j++)
                if (arr[j] > arr[j+1]) [arr[j], arr[j+1]] = [arr[j+1], arr[j]];
    },
    "বাবল সর্ট O(n²)",
    [100, 500, 1000, 2000]
);

// বিল্ট-ইন সর্ট
PerformanceAnalyzer.benchmark(
    (arr) => arr.sort((a, b) => a - b),
    "Array.sort() — O(n log n)"
);

// কমপ্লেক্সিটি অনুমান
PerformanceAnalyzer.guessComplexity(
    (arr) => arr.sort((a, b) => a - b)
);
```

---

## ১১ — 📋 চিট শিট মাস্টার টেবিল

> "এক পৃষ্ঠায় সব কমপ্লেক্সিটি!"

### ডাটা স্ট্রাকচার অপারেশন

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ডাটা স্ট্রাকচার    অ্যাক্সেস  সার্চ   ইনসার্ট  ডিলিট   স্পেস            ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  Array              O(1)      O(n)    O(n)     O(n)    O(n)              ║
║  Linked List        O(n)      O(n)    O(1)     O(1)    O(n)              ║
║  Stack              O(n)      O(n)    O(1)     O(1)    O(n)              ║
║  Queue              O(n)      O(n)    O(1)     O(1)    O(n)              ║
║  Hash Table         N/A       O(1)*   O(1)*    O(1)*   O(n)              ║
║  BST (সুষম)         O(log n)  O(log n)O(log n) O(log n)O(n)              ║
║  BST (অসুষম)        O(n)      O(n)    O(n)     O(n)    O(n)              ║
║  Heap               O(1)**    O(n)    O(log n) O(log n)O(n)              ║
║  Trie               O(k)      O(k)    O(k)     O(k)    O(n×k)            ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  * গড় কেস, worst case O(n)    ** শুধু min/max                            ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

### সর্টিং অ্যালগরিদম

```
╔══════════════════════════════════════════════════════════════════════╗
║  অ্যালগরিদম      সেরা          গড়           খারাপ        স্পেস    ║
╠══════════════════════════════════════════════════════════════════════╣
║  Bubble Sort     O(n)         O(n²)        O(n²)       O(1)      ║
║  Selection Sort  O(n²)        O(n²)        O(n²)       O(1)      ║
║  Insertion Sort  O(n)         O(n²)        O(n²)       O(1)      ║
║  Merge Sort      O(n log n)   O(n log n)   O(n log n)  O(n)      ║
║  Quick Sort      O(n log n)   O(n log n)   O(n²)       O(log n)  ║
║  Heap Sort       O(n log n)   O(n log n)   O(n log n)  O(1)      ║
║  Counting Sort   O(n + k)     O(n + k)     O(n + k)    O(k)      ║
║  Radix Sort      O(n × d)     O(n × d)     O(n × d)    O(n + k)  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### দ্রুত মনে রাখার সূত্র

```
╔════════════════════════════════════════════════════════════════════╗
║  🧠 মনে রাখার কৌশল                                               ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  O(1)       → "সুইচ টিপুন" — সবসময় একই সময়                     ║
║  O(log n)   → "অর্ধেক করো" — প্রতিবার সমস্যা ভাগ হয়              ║
║  O(n)       → "একবার দেখো" — সবকিছু একবার স্পর্শ                  ║
║  O(n log n) → "ভাগ করো ও জোড়ো" — দক্ষ সর্টিং                    ║
║  O(n²)      → "সবার সাথে তুলনা" — নেস্টেড লুপ                    ║
║  O(2ⁿ)      → "নাও বা রাখো" — প্রতি ইলিমেন্টে দুটি পথ            ║
║  O(n!)      → "সব সাজাও" — সব পারমুটেশন                          ║
║                                                                   ║
║  📌 পরপর লুপ = যোগ: O(n) + O(n) = O(n)                           ║
║  📌 নেস্টেড লুপ = গুণ: O(n) × O(n) = O(n²)                       ║
║  📌 অর্ধেক করা = লগ: while(n > 0) n /= 2 → O(log n)             ║
║  📌 দুই শাখা রিকার্শন = এক্সপোনেনশিয়াল: T(n) = 2T(n-1) → O(2ⁿ)  ║
║                                                                   ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## ১২ — 🎤 ইন্টারভিউ টিপস

> "ইন্টারভিউতে কমপ্লেক্সিটি প্রশ্ন এড়িয়ে যাওয়ার উপায় নেই!"

### ইন্টারভিউতে কমপ্লেক্সিটি বিশ্লেষণের ধাপ

```
╔════════════════════════════════════════════════════════════════════╗
║  ধাপ ১: সমাধান লেখার আগেই ব্রুট ফোর্স কমপ্লেক্সিটি বলুন        ║
║         "ব্রুট ফোর্সে O(n²) লাগবে..."                             ║
║                                                                   ║
║  ধাপ ২: অপটিমাইজেশনের সুযোগ চিহ্নিত করুন                        ║
║         "হ্যাশম্যাপ ব্যবহার করলে O(n) এ নামানো যায়"              ║
║                                                                   ║
║  ধাপ ৩: টাইম ও স্পেস দুটোই বলুন                                  ║
║         "টাইম O(n), স্পেস O(n) — হ্যাশম্যাপের জন্য"              ║
║                                                                   ║
║  ধাপ ৪: ট্রেডঅফ আলোচনা করুন                                     ║
║         "সময় কমাতে O(n) অতিরিক্ত মেমরি লাগছে"                   ║
╚════════════════════════════════════════════════════════════════════╝
```

### সাধারণ ইন্টারভিউ প্রশ্ন ও উত্তর

```
প্রশ্ন ১: "এই কোডের Time Complexity কত?"
→ লুপ গণনা করুন, নেস্টিং দেখুন, বড় পদ রাখুন

প্রশ্ন ২: "O(n) আর O(n²) এর পার্থক্য কী?"
→ "n বাড়লে O(n²) অনেক বেশি ধীর হয়। n=1000 এ O(n)=1000,
   কিন্তু O(n²)=1,000,000 — হাজার গুণ বেশি!"

প্রশ্ন ৩: "কীভাবে O(n²) কে O(n) এ নামাবেন?"
→ হ্যাশম্যাপ, সর্টিং + টু পয়েন্টার, প্রিপ্রসেসিং

প্রশ্ন ৪: "Space-Time Tradeoff ব্যাখ্যা করুন"
→ "মেমোইজেশনে O(n) মেমরি খরচ করে O(2ⁿ) থেকে O(n) এ নামানো যায়"

প্রশ্ন ৫: "Amortized O(1) মানে কী?"
→ "বেশিরভাগ অপারেশন O(1), মাঝে মাঝে O(n) — গড়ে O(1)"
```

### ভুল এড়ানোর চেকলিস্ট

```
✅ করণীয়:
  □ সবসময় worst case বলুন (যদি না জিজ্ঞেস করে)
  □ ভিন্ন ইনপুটে ভিন্ন ভেরিয়েবল ব্যবহার করুন (n, m)
  □ রিকার্শনের স্ট্যাক স্পেস ভুলবেন না
  □ বিল্ট-ইন ফাংশনের কমপ্লেক্সিটি জানুন
     - sort() → O(n log n)
     - array_search() → O(n)
     - in_array() → O(n)
     - isset() → O(1)

❌ বর্জনীয়:
  □ "O(2n)" বলবেন না → বলুন "O(n)"
  □ "O(n² + n)" বলবেন না → বলুন "O(n²)"
  □ দুটি আলাদা অ্যারেকে একই n ধরবেন না
  □ "O(n) space" ভুলে যাবেন না রিকার্শনে
```

---

## ১৩ — 🔗 সম্পর্কিত টপিক লিংক

> "Big O শেখার পর এগুলো শিখুন — পরবর্তী ধাপ।"

### পরবর্তী টপিক ম্যাপ

```
                    Big O নোটেশন
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    ডাটা স্ট্রাকচার  অ্যালগরিদম    ম্যাথ
    ├─ Array/List     ├─ সর্টিং      ├─ লগারিদম
    ├─ Stack/Queue    ├─ সার্চিং     ├─ সিরিজ ও ধারা
    ├─ Hash Table     ├─ রিকার্শন    ├─ প্রোবাবিলিটি
    ├─ Tree/Graph     ├─ DP          └─ কম্বিনেটরিক্স
    └─ Heap/Trie      ├─ Greedy
                      └─ Divide & Conquer
```

### সম্পর্কিত ফাইল লিংক

```
📁 04-dsa/
├── 📄 big-o.md              ← আপনি এখানে আছেন
├── 📄 arrays-strings.md      → অ্যারে ও স্ট্রিং
├── 📄 linked-list.md         → লিংকড লিস্ট
├── 📄 stack-queue.md         → স্ট্যাক ও কিউ
├── 📄 hash-table.md          → হ্যাশ টেবিল
├── 📄 trees.md               → ট্রি ডাটা স্ট্রাকচার
├── 📄 graphs.md              → গ্রাফ অ্যালগরিদম
├── 📄 sorting.md             → সর্টিং অ্যালগরিদম
├── 📄 searching.md           → সার্চিং অ্যালগরিদম
├── 📄 recursion-dp.md        → রিকার্শন ও ডাইনামিক প্রোগ্রামিং
└── 📄 greedy-backtracking.md → গ্রিডি ও ব্যাকট্র্যাকিং
```

### অনলাইন রিসোর্স

```
📚 বই:
  • "Introduction to Algorithms" (CLRS) — অধ্যায় ৩: Growth of Functions
  • "Grokking Algorithms" — অধ্যায় ১: Big O

🌐 ওয়েবসাইট:
  • bigocheatsheet.com — ভিজুয়াল চিট শিট
  • visualgo.net — অ্যানিমেটেড ভিজুয়ালাইজেশন
  • leetcode.com — প্র্যাক্টিস প্রবলেম

🎥 ভিডিও:
  • CS50 Big O লেকচার (Harvard)
  • Abdul Bari — Algorithms Playlist (YouTube)
```

---

## 🎯 সর্বশেষ কথা

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                 ║
║   🏆 Big O আয়ত্ত করার ৩ ধাপ:                                   ║
║                                                                 ║
║   ১. মুখস্থ করুন: সাধারণ প্যাটার্নগুলো (লুপ, রিকার্শন)        ║
║   ২. প্র্যাক্টিস করুন: প্রতিদিন ২-৩ টি কোড বিশ্লেষণ করুন      ║
║   ৩. পরিমাপ করুন: বেঞ্চমার্ক চালিয়ে তত্ত্ব যাচাই করুন         ║
║                                                                 ║
║   "প্রতিটি লাইন কোড লেখার আগে ভাবুন —                          ║
║    এটি কত দ্রুত? এটি কত মেমরি নেবে?"                            ║
║                                                                 ║
║   শুভকামনা! 🚀                                                   ║
║                                                                 ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*তৈরি: Big O নোটেশন সম্পূর্ণ বাংলা গাইড | সময় ও স্পেস কমপ্লেক্সিটি*
