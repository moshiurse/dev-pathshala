# 🔍 সার্চিং অ্যালগরিদম (Searching Algorithms) - সম্পূর্ণ গাইড

## 📚 সূচিপত্র
1. [লিনিয়ার সার্চ (Linear Search)](#1-লিনিয়ার-সার্চ)
2. [বাইনারি সার্চ (Binary Search)](#2-বাইনারি-সার্চ)
3. [বাইনারি সার্চের বিভিন্ন ভেরিয়েশন](#3-বাইনারি-সার্চের-ভেরিয়েশন)
4. [ইন্টারপোলেশন সার্চ (Interpolation Search)](#4-ইন্টারপোলেশন-সার্চ)
5. [এক্সপোনেনশিয়াল সার্চ (Exponential Search)](#5-এক্সপোনেনশিয়াল-সার্চ)
6. [টার্নারি সার্চ (Ternary Search)](#6-টার্নারি-সার্চ)
7. [প্র্যাক্টিক্যাল প্রজেক্ট: স্মার্ট প্রোডাক্ট সার্চ ইঞ্জিন](#7-প্র্যাক্টিক্যাল-প্রজেক্ট)
8. [তুলনামূলক বিশ্লেষণ ও ইন্টারভিউ টিপস](#8-তুলনা-ও-টিপস)

---

## 🧠 সার্চিং কী?

সার্চিং হলো একটি ডেটা স্ট্রাকচারে নির্দিষ্ট একটি উপাদান (element) খুঁজে বের করার প্রক্রিয়া।
আমাদের দৈনন্দিন জীবনে আমরা প্রতিনিয়ত সার্চ করি — ফোনবুকে নাম খোঁজা, অভিধানে শব্দ খোঁজা,
গুগলে তথ্য খোঁজা — সবকিছুই সার্চিং অ্যালগরিদমের উপর নির্ভরশীল।

```
সার্চিং অ্যালগরিদমের শ্রেণীবিভাগ:

┌─────────────────────────────────────────┐
│         সার্চিং অ্যালগরিদম              │
├──────────────┬──────────────────────────┤
│  সিকোয়েন্সিয়াল │   ইন্টারভাল ভিত্তিক    │
│  (Sequential)  │   (Interval Based)     │
├──────────────┼──────────────────────────┤
│ • লিনিয়ার     │ • বাইনারি সার্চ          │
│   সার্চ        │ • ইন্টারপোলেশন সার্চ     │
│              │ • এক্সপোনেনশিয়াল সার্চ   │
│              │ • টার্নারি সার্চ           │
└──────────────┴──────────────────────────┘
```

---

## 1. লিনিয়ার সার্চ (Linear Search) {#1-লিনিয়ার-সার্চ}

### 📖 ধারণা

লিনিয়ার সার্চ হলো সবচেয়ে সরল সার্চিং অ্যালগরিদম। এটি অ্যারের প্রথম উপাদান থেকে শুরু করে
একটি একটি করে প্রতিটি উপাদান পরীক্ষা করে যতক্ষণ না টার্গেট পাওয়া যায় অথবা অ্যারে শেষ হয়।

### 🎯 কখন ব্যবহার করবেন?
- অ্যারে সর্টেড না হলে
- অ্যারে ছোট হলে (n < 100)
- শুধু একবার সার্চ করতে হলে
- লিংকড লিস্টে সার্চ করতে হলে

### 📊 ASCII ট্রেস ডায়াগ্রাম

```
অ্যারে: [৪, ৭, ২, ৯, ১, ৫, ৮, ৩]
টার্গেট: ৫

ধাপ ১: [৪] ৭  ২  ৯  ১  ৫  ৮  ৩   → ৪ ≠ ৫ ❌
         ↑

ধাপ ২:  ৪ [৭] ২  ৯  ১  ৫  ৮  ৩   → ৭ ≠ ৫ ❌
            ↑

ধাপ ৩:  ৪  ৭ [২] ৯  ১  ৫  ৮  ৩   → ২ ≠ ৫ ❌
               ↑

ধাপ ৪:  ৪  ৭  ২ [৯] ১  ৫  ৮  ৩   → ৯ ≠ ৫ ❌
                  ↑

ধাপ ৫:  ৪  ৭  ২  ৯ [১] ৫  ৮  ৩   → ১ ≠ ৫ ❌
                     ↑

ধাপ ৬:  ৪  ৭  ২  ৯  ১ [৫] ৮  ৩   → ৫ = ৫ ✅ পাওয়া গেছে! ইনডেক্স: ৫
                        ↑
```

### 💻 PHP কোড

```php
<?php
/**
 * লিনিয়ার সার্চ - অ্যারেতে একটি উপাদান খোঁজা
 * @param array $arr - যে অ্যারেতে খুঁজতে হবে
 * @param mixed $target - যে মানটি খুঁজছি
 * @return int - পাওয়া গেলে ইনডেক্স, না পেলে -১
 */
function linearSearch(array $arr, $target): int {
    $n = count($arr); // অ্যারের দৈর্ঘ্য বের করি

    // প্রতিটি উপাদান একে একে পরীক্ষা করি
    for ($i = 0; $i < $n; $i++) {
        if ($arr[$i] === $target) {
            return $i; // পাওয়া গেছে, ইনডেক্স ফেরত দিই
        }
    }

    return -1; // পুরো অ্যারেতে পাওয়া যায়নি
}

// ব্যবহারের উদাহরণ
$numbers = [4, 7, 2, 9, 1, 5, 8, 3];
$target = 5;
$result = linearSearch($numbers, $target);

if ($result !== -1) {
    echo "✅ {$target} পাওয়া গেছে ইনডেক্স {$result}-এ\n";
} else {
    echo "❌ {$target} অ্যারেতে নেই\n";
}
?>
```

### 💻 JavaScript কোড

```javascript
/**
 * লিনিয়ার সার্চ - অ্যারেতে একটি উপাদান খোঁজা
 * @param {Array} arr - যে অ্যারেতে খুঁজতে হবে
 * @param {*} target - যে মানটি খুঁজছি
 * @returns {number} - পাওয়া গেলে ইনডেক্স, না পেলে -১
 */
function linearSearch(arr, target) {
    // প্রতিটি উপাদান একে একে পরীক্ষা করি
    for (let i = 0; i < arr.length; i++) {
        if (arr[i] === target) {
            return i; // পাওয়া গেছে!
        }
    }
    return -1; // পাওয়া যায়নি
}

// ব্যবহারের উদাহরণ
const numbers = [4, 7, 2, 9, 1, 5, 8, 3];
const target = 5;
const result = linearSearch(numbers, target);

if (result !== -1) {
    console.log(`✅ ${target} পাওয়া গেছে ইনডেক্স ${result}-এ`);
} else {
    console.log(`❌ ${target} অ্যারেতে নেই`);
}
```

### ⏱️ Big O বিশ্লেষণ

```
┌──────────────────────────────────────────┐
│         লিনিয়ার সার্চ - জটিলতা          │
├──────────────┬───────────────────────────┤
│ সেরা ক্ষেত্র    │ O(1) - প্রথমেই পাওয়া    │
│ গড় ক্ষেত্র     │ O(n/2) ≈ O(n)           │
│ সবচেয়ে খারাপ  │ O(n) - শেষে বা নেই      │
│ স্পেস          │ O(1) - অতিরিক্ত মেমরি নেই│
└──────────────┴───────────────────────────┘
```

---

## 2. বাইনারি সার্চ (Binary Search) {#2-বাইনারি-সার্চ}

### 📖 ধারণা

বাইনারি সার্চ হলো একটি অত্যন্ত দক্ষ সার্চিং অ্যালগরিদম যা **সর্টেড অ্যারেতে** কাজ করে।
এটি প্রতিবার অ্যারেকে অর্ধেক ভাগ করে — যেমন আমরা অভিধানে শব্দ খুঁজি। মাঝখানের পাতা খুলে
দেখি, শব্দটি আগে না পরে আছে — তারপর সেই দিকে এগিয়ে যাই।

### 🔑 পূর্বশর্ত
- অ্যারে অবশ্যই **সর্টেড** (ক্রমানুসারে সাজানো) হতে হবে

### 📊 ASCII ট্রেস ডায়াগ্রাম

```
সর্টেড অ্যারে: [২, ৫, ৮, ১২, ১৬, ২৩, ৩৮, ৫৬, ৭২, ৯১]
টার্গেট: ২৩
ইনডেক্স:     ০  ১  ২   ৩   ৪   ৫   ৬   ৭   ৮   ৯

ধাপ ১: low=০, high=৯, mid=৪
  [২, ৫, ৮, ১২, ১৬, ২৩, ৩৮, ৫৬, ৭২, ৯১]
   L              M                    H
   ১৬ < ২৩ → ডান দিকে যাই (low = mid + 1 = ৫)

ধাপ ২: low=৫, high=৯, mid=৭
  [২, ৫, ৮, ১২, ১৬, ২৩, ৩৮, ৫৬, ৭২, ৯১]
                     L          M       H
   ৫৬ > ২৩ → বাম দিকে যাই (high = mid - 1 = ৬)

ধাপ ৩: low=৫, high=৬, mid=৫
  [২, ৫, ৮, ১২, ১৬, ২৩, ৩৮, ৫৬, ৭২, ৯১]
                     LM    H
   ২৩ = ২৩ → ✅ পাওয়া গেছে! ইনডেক্স: ৫

মোট তুলনা: ৩ বার (১০টি উপাদানে!)
লিনিয়ারে লাগতো: ৬ বার
```

### 🔄 রিকার্সিভ vs ইটারেটিভ পদ্ধতি

```
রিকার্সিভ পদ্ধতি:                  ইটারেটিভ পদ্ধতি:
┌─────────────────┐              ┌─────────────────┐
│ search(0, 9)    │              │ while(low<=high) │
│   mid = 4       │              │   mid = 4 → ডান │
│   ১৬ < ২৩      │              │   mid = 7 → বাম │
│   ↓             │              │   mid = 5 → ✅  │
│ search(5, 9)    │              └─────────────────┘
│   mid = 7       │
│   ৫৬ > ২৩      │              সুবিধা:
│   ↓             │              • কম মেমরি ব্যবহার
│ search(5, 6)    │              • স্ট্যাক ওভারফ্লো
│   mid = 5       │                ঝুঁকি নেই
│   ২৩ = ২৩ ✅   │              • প্র্যাকটিসে বেশি
└─────────────────┘                ব্যবহৃত
```

### 💻 PHP কোড (ইটারেটিভ)

```php
<?php
/**
 * বাইনারি সার্চ - ইটারেটিভ পদ্ধতি
 * সর্টেড অ্যারেতে একটি উপাদান দ্রুত খোঁজে
 */
function binarySearch(array $arr, $target): int {
    $low = 0;                    // শুরুর ইনডেক্স
    $high = count($arr) - 1;    // শেষের ইনডেক্স

    while ($low <= $high) {
        // মাঝের ইনডেক্স বের করি (ওভারফ্লো এড়াতে এই সূত্র)
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) {
            return $mid;          // পাওয়া গেছে!
        } elseif ($arr[$mid] < $target) {
            $low = $mid + 1;     // ডান অর্ধেকে খুঁজি
        } else {
            $high = $mid - 1;    // বাম অর্ধেকে খুঁজি
        }
    }

    return -1; // পাওয়া যায়নি
}

// রিকার্সিভ সংস্করণ
function binarySearchRecursive(array $arr, $target, int $low, int $high): int {
    // ভিত্তি শর্ত: সার্চ রেঞ্জ শেষ হয়ে গেছে
    if ($low > $high) {
        return -1;
    }

    $mid = $low + intdiv($high - $low, 2);

    if ($arr[$mid] === $target) {
        return $mid;
    } elseif ($arr[$mid] < $target) {
        // ডান অর্ধেকে রিকার্সিভলি খুঁজি
        return binarySearchRecursive($arr, $target, $mid + 1, $high);
    } else {
        // বাম অর্ধেকে রিকার্সিভলি খুঁজি
        return binarySearchRecursive($arr, $target, $low, $mid - 1);
    }
}

// পরীক্ষা
$sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91];
echo "ইটারেটিভ: " . binarySearch($sorted, 23) . "\n";        // ৫
echo "রিকার্সিভ: " . binarySearchRecursive($sorted, 23, 0, 9) . "\n"; // ৫
?>
```

### 💻 JavaScript কোড

```javascript
/**
 * বাইনারি সার্চ - ইটারেটিভ পদ্ধতি
 * সর্টেড অ্যারেতে একটি উপাদান দ্রুত খোঁজে
 */
function binarySearch(arr, target) {
    let low = 0;                    // শুরুর ইনডেক্স
    let high = arr.length - 1;     // শেষের ইনডেক্স

    while (low <= high) {
        // মাঝের ইনডেক্স বের করি
        const mid = Math.floor(low + (high - low) / 2);

        if (arr[mid] === target) {
            return mid;             // পাওয়া গেছে!
        } else if (arr[mid] < target) {
            low = mid + 1;         // ডান অর্ধেকে খুঁজি
        } else {
            high = mid - 1;        // বাম অর্ধেকে খুঁজি
        }
    }

    return -1; // পাওয়া যায়নি
}

// রিকার্সিভ সংস্করণ
function binarySearchRecursive(arr, target, low = 0, high = arr.length - 1) {
    if (low > high) return -1; // ভিত্তি শর্ত

    const mid = Math.floor(low + (high - low) / 2);

    if (arr[mid] === target) return mid;
    if (arr[mid] < target) {
        return binarySearchRecursive(arr, target, mid + 1, high);
    }
    return binarySearchRecursive(arr, target, low, mid - 1);
}

// পরীক্ষা
const sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91];
console.log("ইটারেটিভ:", binarySearch(sorted, 23));        // ৫
console.log("রিকার্সিভ:", binarySearchRecursive(sorted, 23)); // ৫
```

### ⏱️ Big O বিশ্লেষণ

```
┌──────────────────────────────────────────┐
│         বাইনারি সার্চ - জটিলতা           │
├──────────────┬───────────────────────────┤
│ সেরা ক্ষেত্র    │ O(1) - মাঝেই পেয়ে যাই  │
│ গড় ক্ষেত্র     │ O(log n)               │
│ সবচেয়ে খারাপ  │ O(log n)               │
│ স্পেস (ইটার.) │ O(1)                   │
│ স্পেস (রিকার.)│ O(log n) - কল স্ট্যাক   │
└──────────────┴───────────────────────────┘

কেন O(log n)?
━━━━━━━━━━━━━
প্রতি ধাপে অ্যারে অর্ধেক হয়:
n → n/2 → n/4 → n/8 → ... → 1

ধাপ সংখ্যা k হলে: n / 2^k = 1
∴ k = log₂(n)

উদাহরণ:
• n = 1,000 → প্রায় ১০ ধাপ
• n = 1,000,000 → প্রায় ২০ ধাপ
• n = 1,000,000,000 → প্রায় ৩০ ধাপ!
```

---

## 3. বাইনারি সার্চের ভেরিয়েশন {#3-বাইনারি-সার্চের-ভেরিয়েশন}

### 3.1 প্রথম অবস্থান খোঁজা (First Occurrence / Lower Bound)

একই মান একাধিকবার থাকলে **প্রথম** অবস্থান খুঁজে বের করা।

```
অ্যারে: [১, ৩, ৩, ৩, ৫, ৭, ৭, ৯]
টার্গেট: ৩

সাধারণ বাইনারি সার্চ → ইনডেক্স ২ (যেকোনো ৩)
ফার্স্ট অকারেন্স    → ইনডেক্স ১ (প্রথম ৩) ✅
```

#### PHP কোড

```php
<?php
/**
 * প্রথম অবস্থান খোঁজা - সর্টেড অ্যারেতে প্রথম occurrence
 */
function findFirstOccurrence(array $arr, $target): int {
    $low = 0;
    $high = count($arr) - 1;
    $result = -1; // ফলাফল সংরক্ষণ করি

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) {
            $result = $mid;      // পেয়েছি, কিন্তু আরো বামে থাকতে পারে
            $high = $mid - 1;    // বামে আরো খুঁজি
        } elseif ($arr[$mid] < $target) {
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    return $result;
}

$arr = [1, 3, 3, 3, 5, 7, 7, 9];
echo "প্রথম ৩ এর অবস্থান: " . findFirstOccurrence($arr, 3) . "\n"; // ১
?>
```

#### JavaScript কোড

```javascript
/**
 * প্রথম অবস্থান খোঁজা - সর্টেড অ্যারেতে প্রথম occurrence
 */
function findFirstOccurrence(arr, target) {
    let low = 0;
    let high = arr.length - 1;
    let result = -1;

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);

        if (arr[mid] === target) {
            result = mid;       // পেয়েছি, কিন্তু আরো বামে থাকতে পারে
            high = mid - 1;     // বামে আরো খুঁজি
        } else if (arr[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }

    return result;
}

console.log("প্রথম ৩:", findFirstOccurrence([1, 3, 3, 3, 5, 7, 7, 9], 3)); // ১
```

### 3.2 শেষ অবস্থান খোঁজা (Last Occurrence / Upper Bound)

```php
<?php
/**
 * শেষ অবস্থান খোঁজা - সর্টেড অ্যারেতে শেষ occurrence
 */
function findLastOccurrence(array $arr, $target): int {
    $low = 0;
    $high = count($arr) - 1;
    $result = -1;

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) {
            $result = $mid;      // পেয়েছি, কিন্তু আরো ডানে থাকতে পারে
            $low = $mid + 1;     // ডানে আরো খুঁজি
        } elseif ($arr[$mid] < $target) {
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    return $result;
}

$arr = [1, 3, 3, 3, 5, 7, 7, 9];
echo "শেষ ৩ এর অবস্থান: " . findLastOccurrence($arr, 3) . "\n"; // ৩
?>
```

```javascript
/**
 * শেষ অবস্থান খোঁজা
 */
function findLastOccurrence(arr, target) {
    let low = 0, high = arr.length - 1, result = -1;

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (arr[mid] === target) {
            result = mid;
            low = mid + 1;     // ডানে আরো খুঁজি
        } else if (arr[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return result;
}
```

### 3.3 কতবার আছে গণনা (Count Occurrences)

```
কৌশল: শেষ_অবস্থান - প্রথম_অবস্থান + ১

অ্যারে: [১, ৩, ৩, ৩, ৫, ৭, ৭, ৯]
৩ এর গণনা = শেষ(৩) - প্রথম(৩) + ১ = ৩ - ১ + ১ = ৩
```

```php
<?php
function countOccurrences(array $arr, $target): int {
    $first = findFirstOccurrence($arr, $target);
    if ($first === -1) return 0; // একটিও নেই

    $last = findLastOccurrence($arr, $target);
    return $last - $first + 1;
}

echo "৩ আছে " . countOccurrences([1,3,3,3,5,7,7,9], 3) . " বার\n"; // ৩ বার
?>
```

```javascript
function countOccurrences(arr, target) {
    const first = findFirstOccurrence(arr, target);
    if (first === -1) return 0;

    const last = findLastOccurrence(arr, target);
    return last - first + 1;
}

console.log("৩ আছে", countOccurrences([1,3,3,3,5,7,7,9], 3), "বার"); // ৩ বার
```

### 3.4 রোটেটেড সর্টেড অ্যারেতে সার্চ

```
মূল সর্টেড: [২, ৪, ৬, ৮, ১০, ১২, ১৪, ১৬]
৩ বার রোটেট: [১০, ১২, ১৪, ১৬, ২, ৪, ৬, ৮]
                              ↑ পিভট পয়েন্ট

বৈশিষ্ট্য: অন্তত একটি অর্ধেক সবসময় সর্টেড থাকে!

[১০, ১২, ১৪, ১৬, ২, ৪, ৬, ৮]
 └─── সর্টেড ───┘  └─সর্টেড─┘
```

#### PHP কোড

```php
<?php
/**
 * রোটেটেড সর্টেড অ্যারেতে সার্চ
 * কৌশল: যেকোনো মুহূর্তে একটি অর্ধেক অবশ্যই সর্টেড
 */
function searchRotatedArray(array $arr, $target): int {
    $low = 0;
    $high = count($arr) - 1;

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) {
            return $mid; // পাওয়া গেছে!
        }

        // বাম অর্ধেক সর্টেড কিনা পরীক্ষা
        if ($arr[$low] <= $arr[$mid]) {
            // টার্গেট বাম সর্টেড অংশে আছে?
            if ($target >= $arr[$low] && $target < $arr[$mid]) {
                $high = $mid - 1; // বামে যাই
            } else {
                $low = $mid + 1;  // ডানে যাই
            }
        } else {
            // ডান অর্ধেক সর্টেড
            if ($target > $arr[$mid] && $target <= $arr[$high]) {
                $low = $mid + 1;  // ডানে যাই
            } else {
                $high = $mid - 1; // বামে যাই
            }
        }
    }

    return -1;
}

$rotated = [10, 12, 14, 16, 2, 4, 6, 8];
echo "৬ আছে ইনডেক্স: " . searchRotatedArray($rotated, 6) . "\n"; // ৬
?>
```

#### JavaScript কোড

```javascript
/**
 * রোটেটেড সর্টেড অ্যারেতে সার্চ
 */
function searchRotatedArray(arr, target) {
    let low = 0, high = arr.length - 1;

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);

        if (arr[mid] === target) return mid;

        // বাম অর্ধেক সর্টেড?
        if (arr[low] <= arr[mid]) {
            if (target >= arr[low] && target < arr[mid]) {
                high = mid - 1;
            } else {
                low = mid + 1;
            }
        } else {
            // ডান অর্ধেক সর্টেড
            if (target > arr[mid] && target <= arr[high]) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }
    }

    return -1;
}

console.log("৬ আছে ইনডেক্স:", searchRotatedArray([10,12,14,16,2,4,6,8], 6)); // ৬
```

### 3.5 পিক এলিমেন্ট খোঁজা (Find Peak Element)

```
পিক এলিমেন্ট: যে উপাদানটি তার দুই পাশের উপাদান থেকে বড়

অ্যারে: [১, ৩, ৮, ১২, ৪, ২]
               ↑
             পিক (১২)

বাইনারি সার্চ কৌশল:
• mid যদি বামের চেয়ে ছোট হয় → পিক বামে আছে
• mid যদি ডানের চেয়ে ছোট হয় → পিক ডানে আছে
• অন্যথায় mid-ই পিক
```

```php
<?php
/**
 * পিক এলিমেন্ট খোঁজা - O(log n)
 */
function findPeakElement(array $arr): int {
    $low = 0;
    $high = count($arr) - 1;

    while ($low < $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] < $arr[$mid + 1]) {
            $low = $mid + 1;   // পিক ডানে আছে
        } else {
            $high = $mid;      // পিক বামে বা mid-এ আছে
        }
    }

    return $low; // low === high, এটাই পিক ইনডেক্স
}

echo "পিক ইনডেক্স: " . findPeakElement([1, 3, 8, 12, 4, 2]) . "\n"; // ৩
?>
```

```javascript
function findPeakElement(arr) {
    let low = 0, high = arr.length - 1;

    while (low < high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (arr[mid] < arr[mid + 1]) {
            low = mid + 1;   // পিক ডানে
        } else {
            high = mid;      // পিক বামে বা এখানে
        }
    }
    return low;
}

console.log("পিক ইনডেক্স:", findPeakElement([1, 3, 8, 12, 4, 2])); // ৩
```

### 3.6 বর্গমূল বের করা (Square Root using Binary Search)

```
√২৫ = ?

সার্চ রেঞ্জ: [১, ২৫]
mid=১৩ → ১৩²=১৬৯ > ২৫ → high=১২
mid=৬  → ৬²=৩৬ > ২৫    → high=৫
mid=৩  → ৩²=৯ < ২৫     → low=৪
mid=৪  → ৪²=১৬ < ২৫    → low=৫
mid=৫  → ৫²=২৫ = ২৫    → ✅ উত্তর: ৫
```

```php
<?php
/**
 * পূর্ণসংখ্যা বর্গমূল - বাইনারি সার্চ দিয়ে
 * floor(√n) বের করে
 */
function intSqrt(int $n): int {
    if ($n < 2) return $n;

    $low = 1;
    $high = $n;
    $result = 1;

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);
        $square = $mid * $mid;

        if ($square === $n) {
            return $mid;         // নিখুঁত বর্গমূল
        } elseif ($square < $n) {
            $result = $mid;      // এখন পর্যন্ত সেরা উত্তর
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    return $result;
}

echo "√২৫ = " . intSqrt(25) . "\n";  // ৫
echo "√২৭ = " . intSqrt(27) . "\n";  // ৫ (floor)
?>
```

```javascript
function intSqrt(n) {
    if (n < 2) return n;
    let low = 1, high = n, result = 1;

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        const square = mid * mid;

        if (square === n) return mid;
        if (square < n) {
            result = mid;
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return result;
}

console.log("√25 =", intSqrt(25)); // ৫
console.log("√27 =", intSqrt(27)); // ৫
```

### 3.7 ২D ম্যাট্রিক্সে সার্চ (Search in 2D Matrix)

```
ম্যাট্রিক্স (প্রতি সারি সর্টেড, পরের সারির প্রথম > আগের সারির শেষ):
┌────┬────┬────┬────┐
│  ১ │  ৩ │  ৫ │  ৭ │  সারি ০
├────┼────┼────┼────┤
│ ১০ │ ১১ │ ১৬ │ ২০ │  সারি ১
├────┼────┼────┼────┤
│ ২৩ │ ৩০ │ ৩৪ │ ৫০ │  সারি ২
└────┴────┴────┴────┘

কৌশল: পুরো ম্যাট্রিক্সকে একটি ১D সর্টেড অ্যারে হিসেবে ভাবি!
ইনডেক্স ম্যাপিং: row = index / cols, col = index % cols
```

```php
<?php
/**
 * ২D ম্যাট্রিক্সে বাইনারি সার্চ
 */
function searchMatrix(array $matrix, $target): bool {
    if (empty($matrix) || empty($matrix[0])) return false;

    $rows = count($matrix);
    $cols = count($matrix[0]);
    $low = 0;
    $high = $rows * $cols - 1; // মোট উপাদান - ১

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);
        // ১D ইনডেক্স থেকে ২D রো-কলাম বের করি
        $row = intdiv($mid, $cols);
        $col = $mid % $cols;
        $value = $matrix[$row][$col];

        if ($value === $target) {
            return true;
        } elseif ($value < $target) {
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    return false;
}

$matrix = [
    [1, 3, 5, 7],
    [10, 11, 16, 20],
    [23, 30, 34, 50]
];
echo searchMatrix($matrix, 16) ? "পাওয়া গেছে ✅\n" : "পাওয়া যায়নি ❌\n";
?>
```

```javascript
function searchMatrix(matrix, target) {
    if (!matrix.length || !matrix[0].length) return false;

    const rows = matrix.length;
    const cols = matrix[0].length;
    let low = 0, high = rows * cols - 1;

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        const row = Math.floor(mid / cols);
        const col = mid % cols;
        const value = matrix[row][col];

        if (value === target) return true;
        if (value < target) low = mid + 1;
        else high = mid - 1;
    }
    return false;
}

const matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,50]];
console.log("১৬:", searchMatrix(matrix, 16) ? "পাওয়া গেছে ✅" : "নেই ❌");
```

### 3.8 নিকটতম মান খোঁজা (Find Closest Element)

```php
<?php
/**
 * সর্টেড অ্যারেতে টার্গেটের নিকটতম মান খোঁজা
 */
function findClosest(array $arr, $target): int {
    $n = count($arr);
    if ($target <= $arr[0]) return $arr[0];
    if ($target >= $arr[$n - 1]) return $arr[$n - 1];

    $low = 0;
    $high = $n - 1;

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) return $arr[$mid];

        if ($arr[$mid] < $target) {
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    // low এবং high দুটোর মধ্যে কোনটি টার্গেটের কাছে?
    if (abs($arr[$low] - $target) < abs($arr[$high] - $target)) {
        return $arr[$low];
    }
    return $arr[$high];
}

echo "২৫ এর নিকটতম: " . findClosest([2, 5, 8, 12, 16, 23, 38], 25) . "\n"; // ২৩
?>
```

```javascript
function findClosest(arr, target) {
    let low = 0, high = arr.length - 1;

    if (target <= arr[0]) return arr[0];
    if (target >= arr[high]) return arr[high];

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (arr[mid] === target) return arr[mid];
        if (arr[mid] < target) low = mid + 1;
        else high = mid - 1;
    }

    // low ও high পজিশনের মধ্যে কোনটি কাছে?
    return Math.abs(arr[low] - target) < Math.abs(arr[high] - target)
        ? arr[low] : arr[high];
}

console.log("২৫ এর নিকটতম:", findClosest([2,5,8,12,16,23,38], 25)); // ২৩
```

---

## 4. ইন্টারপোলেশন সার্চ (Interpolation Search) {#4-ইন্টারপোলেশন-সার্চ}

### 📖 ধারণা

বাইনারি সার্চ সবসময় মাঝখানে যায়, কিন্তু ইন্টারপোলেশন সার্চ **আনুমানিক অবস্থানে** যায় —
যেমন আমরা অভিধানে "আ" দিয়ে শুরু হওয়া শব্দ খুঁজতে শুরুর দিকে খুলি, "য" দিয়ে শুরু হলে
শেষের দিকে খুলি। এটি সমানভাবে ছড়ানো (uniformly distributed) ডেটায় খুব দ্রুত কাজ করে।

### 📊 সূত্র ও ডায়াগ্রাম

```
সূত্র:
pos = low + ((target - arr[low]) * (high - low)) / (arr[high] - arr[low])

ব্যাখ্যা:
━━━━━━━━━
arr[low] = ১০, arr[high] = ১০০, target = ৭৫

সাধারণ বাইনারি: mid = (0 + 9) / 2 = ৪ (মাঝখানে)
ইন্টারপোলেশন: pos = 0 + ((৭৫-১০)*(৯-০))/(১০০-১০)
                    = 0 + (৬৫ * ৯) / ৯০
                    = 0 + ৬.৫ ≈ ৭ (শেষের দিকে — বুদ্ধিমান!)

ডায়াগ্রাম:
[১০, ২০, ৩০, ৪০, ৫০, ৬০, ৭০, ৮০, ৯০, ১০০]
  ↑                   ↑              ↑
 low              বাইনারি mid    ইন্টারপোলেশন pos
                  (সবসময় মাঝে)   (টার্গেটের কাছে!)
```

### 💻 PHP কোড

```php
<?php
/**
 * ইন্টারপোলেশন সার্চ
 * সমানভাবে ছড়ানো সর্টেড ডেটায় সবচেয়ে ভালো কাজ করে
 */
function interpolationSearch(array $arr, $target): int {
    $low = 0;
    $high = count($arr) - 1;

    while ($low <= $high && $target >= $arr[$low] && $target <= $arr[$high]) {
        // ভাগ শূন্য এড়ানো
        if ($arr[$high] === $arr[$low]) {
            if ($arr[$low] === $target) return $low;
            break;
        }

        // আনুমানিক অবস্থান বের করি
        $pos = $low + intdiv(
            ($target - $arr[$low]) * ($high - $low),
            $arr[$high] - $arr[$low]
        );

        if ($arr[$pos] === $target) {
            return $pos;           // পাওয়া গেছে!
        } elseif ($arr[$pos] < $target) {
            $low = $pos + 1;      // ডানে যাই
        } else {
            $high = $pos - 1;     // বামে যাই
        }
    }

    return -1;
}

$uniform = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100];
echo "৭০ আছে ইনডেক্স: " . interpolationSearch($uniform, 70) . "\n"; // ৬
?>
```

### 💻 JavaScript কোড

```javascript
/**
 * ইন্টারপোলেশন সার্চ
 */
function interpolationSearch(arr, target) {
    let low = 0, high = arr.length - 1;

    while (low <= high && target >= arr[low] && target <= arr[high]) {
        if (arr[high] === arr[low]) {
            if (arr[low] === target) return low;
            break;
        }

        // আনুমানিক অবস্থান গণনা
        const pos = Math.floor(
            low + ((target - arr[low]) * (high - low)) / (arr[high] - arr[low])
        );

        if (arr[pos] === target) return pos;
        if (arr[pos] < target) low = pos + 1;
        else high = pos - 1;
    }

    return -1;
}

const uniform = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100];
console.log("৭০ আছে ইনডেক্স:", interpolationSearch(uniform, 70)); // ৬
```

### ⏱️ Big O বিশ্লেষণ

```
┌──────────────────────────────────────────────┐
│       ইন্টারপোলেশন সার্চ - জটিলতা            │
├──────────────┬───────────────────────────────┤
│ সেরা ক্ষেত্র    │ O(1)                        │
│ গড় (সমবণ্টন) │ O(log log n) — অসাধারণ!     │
│ সবচেয়ে খারাপ  │ O(n) — অসম বণ্টনে           │
│ স্পেস          │ O(1)                        │
└──────────────┴───────────────────────────────┘

⚠️ সতর্কতা: ডেটা সমানভাবে ছড়ানো না হলে
বাইনারি সার্চের চেয়ে খারাপ পারফর্ম করতে পারে!
```

---

## 5. এক্সপোনেনশিয়াল সার্চ (Exponential Search) {#5-এক্সপোনেনশিয়াল-সার্চ}

### 📖 ধারণা

এক্সপোনেনশিয়াল সার্চ দুটি ধাপে কাজ করে:
1. **রেঞ্জ খোঁজা**: ১, ২, ৪, ৮, ১৬... এভাবে এক্সপোনেনশিয়ালি বাড়িয়ে রেঞ্জ নির্ধারণ
2. **বাইনারি সার্চ**: সেই রেঞ্জের মধ্যে বাইনারি সার্চ চালানো

### কখন ব্যবহার করবেন?
- অসীম/অজানা আকারের সর্টেড অ্যারেতে (Unbounded Search)
- টার্গেট শুরুর দিকে থাকলে খুব দ্রুত

### 📊 ASCII ট্রেস

```
অ্যারে: [২, ৩, ৪, ১০, ৪০, ৫০, ৫৫, ৬০, ৭০, ৮০, ৯১, ১০০]
টার্গেট: ৫৫

ধাপ ১ - রেঞ্জ খোঁজা (এক্সপোনেনশিয়াল জাম্প):
  i=১: arr[1]=৩ < ৫৫    → চালিয়ে যাই
  i=২: arr[2]=৪ < ৫৫    → চালিয়ে যাই
  i=৪: arr[4]=৪০ < ৫৫   → চালিয়ে যাই
  i=৮: arr[8]=৭০ > ৫৫   → থামো! রেঞ্জ পাওয়া গেছে [৪, ৮]

ধাপ ২ - বাইনারি সার্চ রেঞ্জ [৪, ৮] এ:
  [৪০, ৫০, ৫৫, ৬০, ৭০]
   ↑       ↑        ↑
  low     mid     high
  ৫৫ < ৬০ → high = mid - 1

  [৪০, ৫০, ৫৫]
   ↑   ↑    ↑
  low mid  high
  ৫০ < ৫৫ → low = mid + 1

  [৫৫]
   ↑
  low=high=mid
  ৫৫ = ৫৫ ✅ পাওয়া গেছে! ইনডেক্স: ৬
```

### 💻 PHP কোড

```php
<?php
/**
 * এক্সপোনেনশিয়াল সার্চ
 * প্রথমে রেঞ্জ খুঁজি, তারপর সেই রেঞ্জে বাইনারি সার্চ
 */
function exponentialSearch(array $arr, $target): int {
    $n = count($arr);

    // প্রথম উপাদান পরীক্ষা
    if ($arr[0] === $target) return 0;

    // রেঞ্জ খুঁজি: ১, ২, ৪, ৮... করে বাড়াই
    $i = 1;
    while ($i < $n && $arr[$i] <= $target) {
        $i *= 2; // দ্বিগুণ করে বাড়াই
    }

    // নির্ধারিত রেঞ্জে বাইনারি সার্চ
    $low = intdiv($i, 2);        // আগের জাম্প পয়েন্ট
    $high = min($i, $n - 1);    // বর্তমান জাম্প বা অ্যারের শেষ

    while ($low <= $high) {
        $mid = $low + intdiv($high - $low, 2);

        if ($arr[$mid] === $target) {
            return $mid;
        } elseif ($arr[$mid] < $target) {
            $low = $mid + 1;
        } else {
            $high = $mid - 1;
        }
    }

    return -1;
}

$arr = [2, 3, 4, 10, 40, 50, 55, 60, 70, 80, 91, 100];
echo "৫৫ আছে ইনডেক্স: " . exponentialSearch($arr, 55) . "\n"; // ৬
?>
```

### 💻 JavaScript কোড

```javascript
/**
 * এক্সপোনেনশিয়াল সার্চ
 */
function exponentialSearch(arr, target) {
    const n = arr.length;

    if (arr[0] === target) return 0;

    // রেঞ্জ খুঁজি
    let i = 1;
    while (i < n && arr[i] <= target) {
        i *= 2;
    }

    // রেঞ্জের মধ্যে বাইনারি সার্চ
    let low = Math.floor(i / 2);
    let high = Math.min(i, n - 1);

    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);

        if (arr[mid] === target) return mid;
        if (arr[mid] < target) low = mid + 1;
        else high = mid - 1;
    }

    return -1;
}

const arr = [2, 3, 4, 10, 40, 50, 55, 60, 70, 80, 91, 100];
console.log("৫৫ আছে ইনডেক্স:", exponentialSearch(arr, 55)); // ৬
```

### ⏱️ Big O বিশ্লেষণ

```
┌──────────────────────────────────────────┐
│      এক্সপোনেনশিয়াল সার্চ - জটিলতা      │
├──────────────┬───────────────────────────┤
│ সেরা ক্ষেত্র    │ O(1)                    │
│ গড় ক্ষেত্র     │ O(log n)               │
│ সবচেয়ে খারাপ  │ O(log n)               │
│ স্পেস          │ O(1)                   │
├──────────────┴───────────────────────────┤
│ বিশেষ সুবিধা: টার্গেট শুরুর দিকে থাকলে  │
│ O(log i) — যেখানে i হলো টার্গেটের        │
│ অবস্থান। অসীম অ্যারেতেও কাজ করে!       │
└──────────────────────────────────────────┘
```

---

## 6. টার্নারি সার্চ (Ternary Search) {#6-টার্নারি-সার্চ}

### 📖 ধারণা

বাইনারি সার্চ অ্যারেকে ২ ভাগে ভাগ করে, টার্নারি সার্চ **৩ ভাগে** ভাগ করে।
এটি বিশেষভাবে **ইউনিমোডাল ফাংশনে** (একটি সর্বোচ্চ বা সর্বনিম্ন বিন্দু আছে) সর্বোচ্চ/সর্বনিম্ন
মান খুঁজতে ব্যবহৃত হয়।

### 📊 ASCII ট্রেস

```
অ্যারে: [১, ২, ৫, ৯, ১২, ১৫, ১৮, ২১, ২৫, ২৮, ৩০]
টার্গেট: ১৮

low=০, high=১০
mid1 = 0 + (10-0)/3 = ৩     → arr[3] = ৯
mid2 = 10 - (10-0)/3 = ৭    → arr[7] = ২১

 [১, ২, ৫, ৯, ১২, ১৫, ১৮, ২১, ২৫, ২৮, ৩০]
  L        M1              M2              H
           ৯ < ১৮ < ২১ → মাঝের ভাগে আছে

low=৪, high=৬
mid1 = 4 + (6-4)/3 = ৪      → arr[4] = ১২
mid2 = 6 - (6-4)/3 = ৬      → arr[6] = ১৮

 [১২, ১৫, ১৮]
  M1       M2
  ১৮ = arr[mid2] ✅ পাওয়া গেছে! ইনডেক্স: ৬
```

### 💻 PHP কোড

```php
<?php
/**
 * টার্নারি সার্চ - সর্টেড অ্যারেকে ৩ ভাগে ভাগ করে খোঁজে
 */
function ternarySearch(array $arr, $target): int {
    $low = 0;
    $high = count($arr) - 1;

    while ($low <= $high) {
        // দুটি মধ্যবিন্দু বের করি — অ্যারে ৩ ভাগে বিভক্ত হবে
        $mid1 = $low + intdiv($high - $low, 3);
        $mid2 = $high - intdiv($high - $low, 3);

        if ($arr[$mid1] === $target) return $mid1;
        if ($arr[$mid2] === $target) return $mid2;

        if ($target < $arr[$mid1]) {
            $high = $mid1 - 1;    // প্রথম তৃতীয়াংশে
        } elseif ($target > $arr[$mid2]) {
            $low = $mid2 + 1;     // শেষ তৃতীয়াংশে
        } else {
            $low = $mid1 + 1;     // মাঝের তৃতীয়াংশে
            $high = $mid2 - 1;
        }
    }

    return -1;
}

$arr = [1, 2, 5, 9, 12, 15, 18, 21, 25, 28, 30];
echo "১৮ আছে ইনডেক্স: " . ternarySearch($arr, 18) . "\n"; // ৬
?>
```

### 💻 JavaScript কোড

```javascript
/**
 * টার্নারি সার্চ - অ্যারেকে ৩ ভাগে ভাগ করে খোঁজে
 */
function ternarySearch(arr, target) {
    let low = 0, high = arr.length - 1;

    while (low <= high) {
        const mid1 = Math.floor(low + (high - low) / 3);
        const mid2 = Math.floor(high - (high - low) / 3);

        if (arr[mid1] === target) return mid1;
        if (arr[mid2] === target) return mid2;

        if (target < arr[mid1]) {
            high = mid1 - 1;
        } else if (target > arr[mid2]) {
            low = mid2 + 1;
        } else {
            low = mid1 + 1;
            high = mid2 - 1;
        }
    }

    return -1;
}

console.log("১৮:", ternarySearch([1,2,5,9,12,15,18,21,25,28,30], 18)); // ৬
```

### 🎯 ইউনিমোডাল ফাংশনে সর্বোচ্চ মান খোঁজা

```javascript
/**
 * ইউনিমোডাল ফাংশনে সর্বোচ্চ মান খোঁজা
 * f(x) প্রথমে বাড়ে তারপর কমে — একটিমাত্র শীর্ষবিন্দু আছে
 */
function findMaxUnimodal(arr) {
    let low = 0, high = arr.length - 1;

    while (high - low > 2) {
        const mid1 = Math.floor(low + (high - low) / 3);
        const mid2 = Math.floor(high - (high - low) / 3);

        if (arr[mid1] < arr[mid2]) {
            low = mid1; // সর্বোচ্চ ডানে
        } else {
            high = mid2; // সর্বোচ্চ বামে
        }
    }

    // বাকি ২-৩টি উপাদানে সর্বোচ্চ খুঁজি
    let maxVal = arr[low];
    for (let i = low + 1; i <= high; i++) {
        maxVal = Math.max(maxVal, arr[i]);
    }
    return maxVal;
}

// ইউনিমোডাল: বাড়ে তারপর কমে
console.log(findMaxUnimodal([1, 4, 8, 15, 22, 19, 12, 7, 3])); // ২২
```

### ⏱️ Big O বিশ্লেষণ

```
┌──────────────────────────────────────────┐
│         টার্নারি সার্চ - জটিলতা          │
├──────────────┬───────────────────────────┤
│ সেরা ক্ষেত্র    │ O(1)                    │
│ গড় ক্ষেত্র     │ O(log₃ n)              │
│ সবচেয়ে খারাপ  │ O(log₃ n)              │
│ স্পেস          │ O(1)                   │
├──────────────┴───────────────────────────┤
│ ⚠️ নোট: log₃(n) < log₂(n), কিন্তু      │
│ প্রতি ধাপে ২টি তুলনা লাগে। তাই আসলে   │
│ বাইনারি সার্চের চেয়ে সামান্য ধীর!       │
│ তবে ইউনিমোডাল সমস্যায় অপরিহার্য।      │
└──────────────────────────────────────────┘
```

---

## 7. প্র্যাক্টিক্যাল প্রজেক্ট: স্মার্ট প্রোডাক্ট সার্চ ইঞ্জিন {#7-প্র্যাক্টিক্যাল-প্রজেক্ট}

### 📖 সমস্যা

একটি ই-কমার্স সাইটে হাজারো পণ্য আছে। ব্যবহারকারী নাম, দাম, ক্যাটাগরি দিয়ে পণ্য খুঁজতে চায়।
আমরা বিভিন্ন সার্চ অ্যালগরিদম ব্যবহার করে একটি স্মার্ট সার্চ সিস্টেম তৈরি করবো।

### 💻 PHP — সম্পূর্ণ প্রজেক্ট

```php
<?php
/**
 * স্মার্ট প্রোডাক্ট সার্চ ইঞ্জিন
 * বিভিন্ন সার্চ অ্যালগরিদম ব্যবহার করে পণ্য খোঁজে
 */
class ProductSearchEngine {
    private array $products = [];
    private array $sortedByPrice = [];   // দাম অনুসারে সর্টেড
    private array $sortedByName = [];    // নাম অনুসারে সর্টেড

    /**
     * পণ্য যোগ করি
     */
    public function addProduct(string $name, float $price, string $category): void {
        $this->products[] = [
            'name' => $name,
            'price' => $price,
            'category' => $category
        ];
    }

    /**
     * ইনডেক্স তৈরি করি — সর্টেড অ্যারে তৈরি করে দ্রুত সার্চের জন্য
     */
    public function buildIndex(): void {
        // দাম অনুসারে সর্ট
        $this->sortedByPrice = $this->products;
        usort($this->sortedByPrice, fn($a, $b) => $a['price'] <=> $b['price']);

        // নাম অনুসারে সর্ট
        $this->sortedByName = $this->products;
        usort($this->sortedByName, fn($a, $b) => strcmp($a['name'], $b['name']));
    }

    /**
     * ক্যাটাগরি দিয়ে খোঁজা — লিনিয়ার সার্চ
     * কারণ: ক্যাটাগরি সর্টেড নয়, একাধিক ফলাফল থাকতে পারে
     */
    public function searchByCategory(string $category): array {
        $results = [];
        foreach ($this->products as $product) {
            if (strtolower($product['category']) === strtolower($category)) {
                $results[] = $product;
            }
        }
        return $results;
    }

    /**
     * নির্দিষ্ট দামের পণ্য — বাইনারি সার্চ
     */
    public function searchByExactPrice(float $price): array {
        $results = [];
        $index = $this->binarySearchPrice($price);

        if ($index === -1) return $results;

        $results[] = $this->sortedByPrice[$index];

        // বামে একই দামের পণ্য খুঁজি
        $left = $index - 1;
        while ($left >= 0 && $this->sortedByPrice[$left]['price'] === $price) {
            $results[] = $this->sortedByPrice[$left--];
        }

        // ডানে একই দামের পণ্য খুঁজি
        $right = $index + 1;
        $n = count($this->sortedByPrice);
        while ($right < $n && $this->sortedByPrice[$right]['price'] === $price) {
            $results[] = $this->sortedByPrice[$right++];
        }

        return $results;
    }

    /**
     * দামের রেঞ্জে পণ্য খোঁজা — বাইনারি সার্চের ভেরিয়েশন
     */
    public function searchByPriceRange(float $minPrice, float $maxPrice): array {
        $results = [];
        $start = $this->lowerBoundPrice($minPrice);
        $n = count($this->sortedByPrice);

        for ($i = $start; $i < $n && $this->sortedByPrice[$i]['price'] <= $maxPrice; $i++) {
            $results[] = $this->sortedByPrice[$i];
        }

        return $results;
    }

    /**
     * নিকটতম দামের পণ্য — বাইনারি সার্চ ভেরিয়েশন
     */
    public function findClosestPrice(float $targetPrice): ?array {
        if (empty($this->sortedByPrice)) return null;

        $n = count($this->sortedByPrice);
        $low = 0;
        $high = $n - 1;

        while ($low <= $high) {
            $mid = $low + intdiv($high - $low, 2);
            if ($this->sortedByPrice[$mid]['price'] === $targetPrice) {
                return $this->sortedByPrice[$mid];
            }
            if ($this->sortedByPrice[$mid]['price'] < $targetPrice) {
                $low = $mid + 1;
            } else {
                $high = $mid - 1;
            }
        }

        if ($low >= $n) return $this->sortedByPrice[$n - 1];
        if ($high < 0) return $this->sortedByPrice[0];

        $diffLow = abs($this->sortedByPrice[$low]['price'] - $targetPrice);
        $diffHigh = abs($this->sortedByPrice[$high]['price'] - $targetPrice);

        return $diffLow < $diffHigh
            ? $this->sortedByPrice[$low]
            : $this->sortedByPrice[$high];
    }

    // --- সহায়ক ফাংশনগুলো ---

    private function binarySearchPrice(float $price): int {
        $low = 0;
        $high = count($this->sortedByPrice) - 1;

        while ($low <= $high) {
            $mid = $low + intdiv($high - $low, 2);
            if ($this->sortedByPrice[$mid]['price'] == $price) return $mid;
            if ($this->sortedByPrice[$mid]['price'] < $price) $low = $mid + 1;
            else $high = $mid - 1;
        }
        return -1;
    }

    private function lowerBoundPrice(float $price): int {
        $low = 0;
        $high = count($this->sortedByPrice) - 1;
        $result = count($this->sortedByPrice);

        while ($low <= $high) {
            $mid = $low + intdiv($high - $low, 2);
            if ($this->sortedByPrice[$mid]['price'] >= $price) {
                $result = $mid;
                $high = $mid - 1;
            } else {
                $low = $mid + 1;
            }
        }
        return $result;
    }
}

// --- ব্যবহারের উদাহরণ ---
$engine = new ProductSearchEngine();

$engine->addProduct("ল্যাপটপ", 65000, "ইলেকট্রনিক্স");
$engine->addProduct("মোবাইল", 25000, "ইলেকট্রনিক্স");
$engine->addProduct("শার্ট", 1200, "পোশাক");
$engine->addProduct("জিন্স", 2500, "পোশাক");
$engine->addProduct("হেডফোন", 3500, "ইলেকট্রনিক্স");
$engine->addProduct("ঘড়ি", 5000, "অ্যাক্সেসরিজ");
$engine->addProduct("জুতা", 3500, "পোশাক");
$engine->addProduct("ব্যাগ", 1800, "অ্যাক্সেসরিজ");

$engine->buildIndex();

// ক্যাটাগরি সার্চ (লিনিয়ার)
echo "=== ইলেকট্রনিক্স পণ্য ===\n";
foreach ($engine->searchByCategory("ইলেকট্রনিক্স") as $p) {
    echo "  {$p['name']} — ৳{$p['price']}\n";
}

// দামের রেঞ্জ সার্চ (বাইনারি)
echo "\n=== ৳১০০০-৳৪০০০ দামের পণ্য ===\n";
foreach ($engine->searchByPriceRange(1000, 4000) as $p) {
    echo "  {$p['name']} — ৳{$p['price']}\n";
}

// নিকটতম দাম (বাইনারি ভেরিয়েশন)
echo "\n=== ৳৪০০০ এর কাছাকাছি ===\n";
$closest = $engine->findClosestPrice(4000);
echo "  {$closest['name']} — ৳{$closest['price']}\n";
?>
```

### 💻 JavaScript — সম্পূর্ণ প্রজেক্ট

```javascript
/**
 * স্মার্ট প্রোডাক্ট সার্চ ইঞ্জিন
 */
class ProductSearchEngine {
    constructor() {
        this.products = [];
        this.sortedByPrice = [];
        this.sortedByName = [];
    }

    // পণ্য যোগ
    addProduct(name, price, category) {
        this.products.push({ name, price, category });
    }

    // ইনডেক্স তৈরি
    buildIndex() {
        this.sortedByPrice = [...this.products].sort((a, b) => a.price - b.price);
        this.sortedByName = [...this.products].sort((a, b) =>
            a.name.localeCompare(b.name, 'bn')
        );
    }

    // ক্যাটাগরি সার্চ — লিনিয়ার
    searchByCategory(category) {
        const cat = category.toLowerCase();
        return this.products.filter(p => p.category.toLowerCase() === cat);
    }

    // দামের রেঞ্জে সার্চ — বাইনারি lower bound ব্যবহার
    searchByPriceRange(minPrice, maxPrice) {
        const start = this._lowerBound(minPrice);
        const results = [];

        for (let i = start; i < this.sortedByPrice.length; i++) {
            if (this.sortedByPrice[i].price > maxPrice) break;
            results.push(this.sortedByPrice[i]);
        }
        return results;
    }

    // নিকটতম দামের পণ্য
    findClosestPrice(targetPrice) {
        const arr = this.sortedByPrice;
        if (!arr.length) return null;

        let low = 0, high = arr.length - 1;

        while (low <= high) {
            const mid = Math.floor(low + (high - low) / 2);
            if (arr[mid].price === targetPrice) return arr[mid];
            if (arr[mid].price < targetPrice) low = mid + 1;
            else high = mid - 1;
        }

        if (low >= arr.length) return arr[arr.length - 1];
        if (high < 0) return arr[0];

        return Math.abs(arr[low].price - targetPrice) <
               Math.abs(arr[high].price - targetPrice)
            ? arr[low] : arr[high];
    }

    // lower bound সহায়ক ফাংশন
    _lowerBound(price) {
        let low = 0, high = this.sortedByPrice.length - 1;
        let result = this.sortedByPrice.length;

        while (low <= high) {
            const mid = Math.floor(low + (high - low) / 2);
            if (this.sortedByPrice[mid].price >= price) {
                result = mid;
                high = mid - 1;
            } else {
                low = mid + 1;
            }
        }
        return result;
    }
}

// ব্যবহার
const engine = new ProductSearchEngine();

engine.addProduct("ল্যাপটপ", 65000, "ইলেকট্রনিক্স");
engine.addProduct("মোবাইল", 25000, "ইলেকট্রনিক্স");
engine.addProduct("শার্ট", 1200, "পোশাক");
engine.addProduct("জিন্স", 2500, "পোশাক");
engine.addProduct("হেডফোন", 3500, "ইলেকট্রনিক্স");
engine.addProduct("ঘড়ি", 5000, "অ্যাক্সেসরিজ");

engine.buildIndex();

console.log("=== ইলেকট্রনিক্স ===");
engine.searchByCategory("ইলেকট্রনিক্স").forEach(p =>
    console.log(`  ${p.name} — ৳${p.price}`)
);

console.log("\n=== ৳১০০০-৳৪০০০ ===");
engine.searchByPriceRange(1000, 4000).forEach(p =>
    console.log(`  ${p.name} — ৳${p.price}`)
);

console.log("\n=== ৳৪০০০ এর কাছাকাছি ===");
const closest = engine.findClosestPrice(4000);
console.log(`  ${closest.name} — ৳${closest.price}`);
```

---

## 8. তুলনামূলক বিশ্লেষণ ও ইন্টারভিউ টিপস {#8-তুলনা-ও-টিপস}

### 📊 সব অ্যালগরিদমের তুলনা

```
┌───────────────────┬──────────┬───────────┬───────────┬──────────┬────────────────┐
│    অ্যালগরিদম      │ সেরা      │ গড়        │ সবচেয়ে    │ স্পেস     │ পূর্বশর্ত         │
│                   │ ক্ষেত্র    │ ক্ষেত্র    │ খারাপ      │          │                │
├───────────────────┼──────────┼───────────┼───────────┼──────────┼────────────────┤
│ লিনিয়ার সার্চ      │ O(1)     │ O(n)      │ O(n)      │ O(1)    │ কোনোটি নেই      │
│ বাইনারি সার্চ       │ O(1)     │ O(log n)  │ O(log n)  │ O(1)    │ সর্টেড অ্যারে    │
│ ইন্টারপোলেশন       │ O(1)     │ O(log     │ O(n)      │ O(1)    │ সর্টেড +        │
│                   │          │ log n)    │           │          │ সমবণ্টন          │
│ এক্সপোনেনশিয়াল    │ O(1)     │ O(log n)  │ O(log n)  │ O(1)    │ সর্টেড অ্যারে    │
│ টার্নারি সার্চ      │ O(1)     │ O(log₃ n) │ O(log₃ n) │ O(1)    │ সর্টেড/          │
│                   │          │           │           │          │ ইউনিমোডাল        │
└───────────────────┴──────────┴───────────┴───────────┴──────────┴────────────────┘
```

### 🎯 কোন পরিস্থিতিতে কোনটি ব্যবহার করবেন?

```
সিদ্ধান্ত গ্রাফ:
━━━━━━━━━━━━━━

অ্যারে সর্টেড?
├── না → লিনিয়ার সার্চ ✅
│         (অথবা প্রথমে সর্ট করে নিন)
│
└── হ্যাঁ → অ্যারের আকার?
    ├── ছোট (< ১০০) → লিনিয়ার সার্চ ✅
    │                   (ওভারহেড কম)
    │
    └── বড় → ডেটা সমানভাবে ছড়ানো?
        ├── হ্যাঁ → ইন্টারপোলেশন সার্চ ✅
        │
        └── না → টার্গেট শুরুর দিকে?
            ├── হ্যাঁ → এক্সপোনেনশিয়াল সার্চ ✅
            │
            └── না → বাইনারি সার্চ ✅
                      (সবচেয়ে নির্ভরযোগ্য)

বিশেষ ক্ষেত্র:
• রোটেটেড অ্যারে → মডিফাইড বাইনারি সার্চ
• সর্বোচ্চ/সর্বনিম্ন খোঁজা → টার্নারি সার্চ
• অসীম অ্যারে → এক্সপোনেনশিয়াল সার্চ
• ২D ম্যাট্রিক্স → ম্যাট্রিক্স বাইনারি সার্চ
```

### 💡 ইন্টারভিউ টিপস ও সাধারণ প্রশ্ন

#### ১. "বাইনারি সার্চ কবে ব্যবহার করবেন?"
> যখন অ্যারে সর্টেড এবং আমাদের O(log n) সময়ে খুঁজতে হবে।

#### ২. "mid = (low + high) / 2 তে সমস্যা কী?"
> ইন্টিজার ওভারফ্লো! low ও high দুটোই বড় হলে যোগফল ওভারফ্লো করবে।
> সমাধান: `mid = low + (high - low) / 2`

#### ৩. "লিনিয়ার সার্চ কখন বাইনারি সার্চের চেয়ে ভালো?"
> - অ্যারে সর্টেড না হলে
> - অ্যারে খুব ছোট হলে (সর্টিং ওভারহেড বেশি)
> - শুধু একবার সার্চ করতে হলে (সর্ট O(n log n) + সার্চ O(log n) > O(n))

#### ৪. "রোটেটেড অ্যারেতে সার্চের কৌশল কী?"
> প্রতি ধাপে অন্তত একটি অর্ধেক সর্টেড থাকে। সেই সর্টেড অর্ধেকে টার্গেট আছে
> কিনা পরীক্ষা করে সঠিক দিকে এগিয়ে যাই।

#### ৫. "ইন্টারপোলেশন সার্চের সীমাবদ্ধতা কী?"
> ডেটা সমানভাবে ছড়ানো না হলে O(n) পর্যন্ত খারাপ হতে পারে।

### 📝 প্র্যাকটিস সমস্যা

```
সহজ:
━━━━━
১. সর্টেড অ্যারেতে একটি সংখ্যা আছে কিনা বলুন
২. একটি অ্যারেতে কোনো সংখ্যা কতবার আছে বের করুন
৩. √n এর পূর্ণসংখ্যা মান বের করুন

মাঝারি:
━━━━━━
৪. রোটেটেড সর্টেড অ্যারেতে সর্বনিম্ন মান খুঁজুন
৫. সর্টেড অ্যারেতে একটি সংখ্যার ঢোকার সঠিক অবস্থান বের করুন
৬. দুটি সর্টেড অ্যারের মিডিয়ান বের করুন

কঠিন:
━━━━━
৭. সর্টেড ম্যাট্রিক্সে k-তম ক্ষুদ্রতম সংখ্যা খুঁজুন
৮. একটি অ্যারেকে সর্বনিম্ন কতবার রোটেট করলে সর্টেড হবে?
৯. পিক এলিমেন্ট ভিত্তিক ম্যাট্রিক্স সার্চ
```

### 🔑 মনে রাখার সংক্ষিপ্ত সূত্র

```
"লিনিয়ার সোজা, বাইনারি ভাগ,
ইন্টারপোলেশন আন্দাজ,
এক্সপো লাফায়, টার্নারি তিন-ভাগ —
সব সার্চের আলাদা কাজ!"

মূল নিয়ম:
━━━━━━━━━
• সর্টেড → বাইনারি সার্চ (ডিফল্ট চয়েস)
• আনসর্টেড → লিনিয়ার সার্চ
• সমবণ্টিত → ইন্টারপোলেশন
• শুরুতে টার্গেট → এক্সপোনেনশিয়াল
• সর্বোচ্চ/সর্বনিম্ন → টার্নারি
• সবসময় mid = low + (high - low) / 2 ব্যবহার করুন!
```

---

## 🔗 পরবর্তী পড়ুন

- [ডাইনামিক প্রোগ্রামিং (Dynamic Programming)](dynamic-programming.md) — সার্চিংয়ের ধারণা
  কাজে লাগিয়ে জটিল সমস্যা সমাধান

---

> 📝 **নোট**: এই গাইডের সব কোড PHP 8.0+ এবং আধুনিক JavaScript (ES6+) এ লেখা।
> প্রতিটি অ্যালগরিদম নিজে কোড করে প্র্যাকটিস করুন — শুধু পড়লে হবে না! 🚀
