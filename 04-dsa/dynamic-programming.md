# 🧠 ডাইনামিক প্রোগ্রামিং (Dynamic Programming)

> _"অতীতকে মনে রাখো, ভবিষ্যৎকে অপটিমাইজ করো।"_
> — প্রতিটি DP সমস্যার মূলমন্ত্র

---

## 📖 সংজ্ঞা

**ডাইনামিক প্রোগ্রামিং (DP)** হলো একটি অ্যালগরিদমিক কৌশল যেখানে একটি জটিল সমস্যাকে ছোট ছোট
**সাব-প্রবলেমে** ভেঙে সমাধান করা হয় এবং প্রতিটি সাব-প্রবলেমের ফলাফল **মনে রাখা (cache)** হয়
যেন একই সাব-প্রবলেম দ্বিতীয়বার সমাধান করতে না হয়।

## 🏠 বাস্তব উদাহরণ

- **GPS রুট খোঁজা**: ঢাকা থেকে চট্টগ্রাম যেতে সবচেয়ে কম সময়ের রাস্তা — প্রতিটি মধ্যবর্তী শহরের জন্য সেরা পথ মনে রাখা হয়
- **দাবা কৌশল**: প্রতিটি বোর্ড পজিশনের সেরা চাল ক্যাশ করা হয়
- **বিনিয়োগ পরিকল্পনা**: সীমিত টাকায় কোন কোন শেয়ারে বিনিয়োগ করলে সর্বোচ্চ লাভ হবে

---

# ১. DP-এর মৌলিক ধারণা

## 🔑 DP কী?

DP = **Recursion** + **Memoization** (অথবা **Tabulation**)

সহজ ভাষায়: যদি একটি সমস্যা সমাধান করতে গিয়ে একই ছোট সমস্যা বারবার আসে,
তাহলে সেই ছোট সমস্যার উত্তর একবার বের করে মনে রাখো — আবার হিসাব করো না।

## 📐 দুটি মূল বৈশিষ্ট্য

### ১. Overlapping Subproblems (ওভারল্যাপিং সাব-প্রবলেম)

একই সাব-প্রবলেম বারবার সমাধান করতে হয়।

```
ফিবোনাচি উদাহরণ: fib(5) হিসাব করতে গেলে —

                    fib(5)
                   /      \
              fib(4)        fib(3)      ← fib(3) দুইবার!
             /     \        /    \
         fib(3)   fib(2) fib(2) fib(1)  ← fib(2) তিনবার!
         /    \
     fib(2)  fib(1)

fib(2) তিনবার, fib(3) দুইবার হিসাব হচ্ছে — এটাই overlap!
```

### ২. Optimal Substructure (অপটিমাল সাব-স্ট্রাকচার)

বড় সমস্যার সেরা সমাধান ছোট সমস্যাগুলোর সেরা সমাধান থেকে তৈরি হয়।

```
উদাহরণ: A → D সর্বনিম্ন দূরত্ব

A → B → C → D  (মোট ১০ কিমি)

যদি A→D এর সেরা পথ B এর মধ্য দিয়ে যায়,
তাহলে A→B অংশটাও নিজে একটি সেরা পথ হবে।
```

## 📊 DP vs Greedy vs Divide & Conquer তুলনা

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য     │   DP             │   Greedy         │ Divide & Conquer │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ সাব-প্রবলেম      │ ওভারল্যাপিং      │ নেই              │ স্বতন্ত্র          │
│ অপটিমাল          │ গ্যারান্টি        │ সবসময় না         │ গ্যারান্টি        │
│ পদ্ধতি           │ সব অপশন দেখে     │ লোকাল সেরা নেয়   │ ভাগ+জোড়া         │
│ মেমরি            │ বেশি (টেবিল)     │ কম               │ মাঝামাঝি          │
│ উদাহরণ           │ Knapsack, LCS    │ Activity Sel.    │ Merge Sort       │
│ সময়             │ O(n²) বা O(nW)   │ O(n log n)       │ O(n log n)       │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

# ২. দুই ধরনের DP Approach

## 🔽 Top-Down (Memoization) — "উপর থেকে নিচে"

**ধারণা**: আগে recursion দিয়ে সমাধান লেখো, তারপর cache যোগ করো।

### ফিবোনাচি: Naive Recursion — O(2ⁿ) 😱

```
কল ট্রি (n=5):

                         fib(5)
                        /      \
                   fib(4)      fib(3)
                  /     \      /    \
             fib(3)  fib(2) fib(2) fib(1)
            /    \    / \    / \
        fib(2) fib(1) 1  0  1  0
        / \
       1   0

মোট কল: ১৫টি! (n বাড়লে exponentially বাড়ে)
```

**PHP — Naive Recursion:**

```php
<?php
// ❌ খারাপ পদ্ধতি — O(2^n) সময় লাগে
function fibNaive($n) {
    // ভিত্তি শর্ত
    if ($n <= 1) return $n;
    // একই জিনিস বারবার হিসাব হচ্ছে!
    return fibNaive($n - 1) + fibNaive($n - 2);
}

echo fibNaive(10); // ৫৫
// fibNaive(40) চালালে অনেক সময় লাগবে!
?>
```

**JS — Naive Recursion:**

```javascript
// ❌ খারাপ পদ্ধতি — O(2^n) সময় লাগে
function fibNaive(n) {
    // ভিত্তি শর্ত
    if (n <= 1) return n;
    // একই সাব-প্রবলেম বারবার সমাধান হচ্ছে
    return fibNaive(n - 1) + fibNaive(n - 2);
}

console.log(fibNaive(10)); // ৫৫
```

### ফিবোনাচি: Memoized — O(n) ✅

```
মেমোইজড কল ট্রি (n=5):

                     fib(5)
                    /      \
               fib(4)    fib(3) ← ক্যাশ থেকে! O(1)
              /     \
         fib(3)   fib(2) ← ক্যাশ থেকে! O(1)
        /     \
   fib(2)   fib(1)
   /    \
fib(1) fib(0)

মোট নতুন কল: মাত্র ৬টি! (n=5 এর জন্য)
ক্যাশ হিট: ২টি (fib(3), fib(2) আবার হিসাব করতে হয়নি)
```

**PHP — Memoization:**

```php
<?php
// ✅ ভালো পদ্ধতি — O(n) সময়, O(n) মেমরি
function fibMemo($n, &$memo = []) {
    // ক্যাশে আছে কিনা দেখো
    if (isset($memo[$n])) return $memo[$n];

    // ভিত্তি শর্ত
    if ($n <= 1) return $n;

    // ফলাফল ক্যাশে রাখো
    $memo[$n] = fibMemo($n - 1, $memo) + fibMemo($n - 2, $memo);
    return $memo[$n];
}

echo fibMemo(50); // ১২৫৮৬২৬৯০২৫ — তাৎক্ষণিক!
?>
```

**JS — Memoization:**

```javascript
// ✅ ভালো পদ্ধতি — O(n) সময়, O(n) মেমরি
function fibMemo(n, memo = {}) {
    // ক্যাশে আছে কিনা দেখো
    if (n in memo) return memo[n];

    // ভিত্তি শর্ত
    if (n <= 1) return n;

    // ফলাফল ক্যাশে রাখো
    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    return memo[n];
}

console.log(fibMemo(50)); // ১২৫৮৬২৬৯০২৫ — তাৎক্ষণিক!
```

---

## 🔼 Bottom-Up (Tabulation) — "নিচ থেকে উপরে"

**ধারণা**: ছোট সমস্যা থেকে শুরু করে ধাপে ধাপে বড় সমস্যার উত্তর তৈরি করো।

### ফিবোনাচি: Tabulation ট্রেস

```
n = 7 এর জন্য টেবিল পূরণ:

ধাপ ০: dp = [0, 1, _, _, _, _, _, _]
ধাপ ১: dp = [0, 1, 1, _, _, _, _, _]   ← dp[2] = dp[1] + dp[0] = 1+0 = 1
ধাপ ২: dp = [0, 1, 1, 2, _, _, _, _]   ← dp[3] = dp[2] + dp[1] = 1+1 = 2
ধাপ ৩: dp = [0, 1, 1, 2, 3, _, _, _]   ← dp[4] = dp[3] + dp[2] = 2+1 = 3
ধাপ ৪: dp = [0, 1, 1, 2, 3, 5, _, _]   ← dp[5] = dp[4] + dp[3] = 3+2 = 5
ধাপ ৫: dp = [0, 1, 1, 2, 3, 5, 8, _]   ← dp[6] = dp[5] + dp[4] = 5+3 = 8
ধাপ ৬: dp = [0, 1, 1, 2, 3, 5, 8, 13]  ← dp[7] = dp[6] + dp[5] = 8+5 = 13

উত্তর: dp[7] = 13 ✅
```

**PHP — Tabulation:**

```php
<?php
// ✅ Bottom-Up পদ্ধতি — O(n) সময়, O(n) মেমরি
function fibTab($n) {
    if ($n <= 1) return $n;

    // টেবিল তৈরি
    $dp = array_fill(0, $n + 1, 0);
    $dp[1] = 1;

    // ছোট থেকে বড় — ধাপে ধাপে পূরণ
    for ($i = 2; $i <= $n; $i++) {
        $dp[$i] = $dp[$i - 1] + $dp[$i - 2];
    }

    return $dp[$n];
}

echo fibTab(50); // ১২৫৮৬২৬৯০২৫
?>
```

**JS — Tabulation:**

```javascript
// ✅ Bottom-Up পদ্ধতি — O(n) সময়, O(n) মেমরি
function fibTab(n) {
    if (n <= 1) return n;

    // টেবিল তৈরি
    const dp = new Array(n + 1).fill(0);
    dp[1] = 1;

    // ছোট থেকে বড় — ধাপে ধাপে পূরণ
    for (let i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }

    return dp[n];
}

console.log(fibTab(50)); // ১২৫৮৬২৬৯০২৫
```

### 🚀 Space Optimized — O(1) মেমরি

```php
<?php
// ⭐ সেরা পদ্ধতি — O(n) সময়, O(1) মেমরি
function fibOptimized($n) {
    if ($n <= 1) return $n;

    // শুধু শেষ দুটি মান রাখলেই চলে!
    $prev2 = 0; // fib(0)
    $prev1 = 1; // fib(1)

    for ($i = 2; $i <= $n; $i++) {
        $current = $prev1 + $prev2;
        $prev2 = $prev1;  // সরিয়ে নাও
        $prev1 = $current;
    }

    return $prev1;
}
?>
```

```javascript
// ⭐ সেরা পদ্ধতি — O(n) সময়, O(1) মেমরি
function fibOptimized(n) {
    if (n <= 1) return n;

    // শুধু শেষ দুটি মান রাখলেই চলে!
    let prev2 = 0, prev1 = 1;

    for (let i = 2; i <= n; i++) {
        const current = prev1 + prev2;
        prev2 = prev1;  // সরিয়ে নাও
        prev1 = current;
    }

    return prev1;
}
```

## ⚖️ Top-Down vs Bottom-Up তুলনা

```
┌──────────────────┬────────────────────────┬────────────────────────┐
│     বৈশিষ্ট্য     │  Top-Down (Memoization)│  Bottom-Up (Tabulation)│
├──────────────────┼────────────────────────┼────────────────────────┤
│ পদ্ধতি           │ Recursion + Cache       │ Iterative + Table      │
│ কোডিং সহজতা     │ সহজ (recursive চিন্তা)  │ মাঝামাঝি               │
│ Stack Overflow   │ সম্ভব (গভীর recursion)  │ নেই                    │
│ মেমরি            │ শুধু দরকারি state       │ পুরো টেবিল             │
│ স্পেস অপটিমাইজ   │ কঠিন                   │ সহজ                    │
│ ডিবাগিং          │ কঠিন                   │ সহজ                    │
│ কখন ব্যবহার      │ সব state দরকার না       │ সব state দরকার         │
└──────────────────┴────────────────────────┴────────────────────────┘
```

---

# ৩. ক্লাসিক 1D DP সমস্যা

## 🪜 Climbing Stairs — সিঁড়ি আরোহণ

### 📖 সমস্যা

n ধাপের সিঁড়ি। প্রতিবার ১ বা ২ ধাপ উঠতে পারো। কতভাবে উপরে যাওয়া যায়?

### 🏠 বাস্তব উদাহরণ

তোমার বাসায় ৫ ধাপের সিঁড়ি আছে। তুমি প্রতিবার ১ ধাপ বা ২ ধাপ করে উঠতে পারো।
কতভাবে উপরে যেতে পারো?

### রিকার্সন ট্রি → মেমোইজেশন → ট্যাবুলেশন

```
n = 4 এর জন্য রিকার্সন ট্রি:

                    stairs(4)
                   /          \
            stairs(3)        stairs(2)
           /        \        /       \
      stairs(2)  stairs(1) stairs(1) stairs(0)
      /      \      |        |         |
 stairs(1) stairs(0) 1       1         1
    |         |
    1         1

সব পাতার যোগফল = ১+১+১+১+১ = ৫ ভাবে
```

```
ট্যাবুলেশন ট্রেস (n=5):

ধাপ:  0   1   2   3   4   5
dp:  [1] [1] [2] [3] [5] [8]
          ↑   ↑   ↑   ↑   ↑
          |  1+1 1+2 2+3 3+5
          |
      ১ ভাবে (এক ধাপেই!)

উত্তর: dp[5] = 8 ভাবে ✅
```

**PHP — Climbing Stairs:**

```php
<?php
// ১. Memoization পদ্ধতি
function climbStairsMemo($n, &$memo = []) {
    if (isset($memo[$n])) return $memo[$n];
    if ($n <= 1) return 1;

    // n-তম ধাপে যেতে পারি (n-1) থেকে ১ ধাপে বা (n-2) থেকে ২ ধাপে
    $memo[$n] = climbStairsMemo($n - 1, $memo) + climbStairsMemo($n - 2, $memo);
    return $memo[$n];
}

// ২. Tabulation পদ্ধতি
function climbStairsTab($n) {
    if ($n <= 1) return 1;
    $dp = array_fill(0, $n + 1, 0);
    $dp[0] = 1; // ০ ধাপ = ১ ভাবে (কিছু না করা)
    $dp[1] = 1; // ১ ধাপ = ১ ভাবে

    for ($i = 2; $i <= $n; $i++) {
        $dp[$i] = $dp[$i - 1] + $dp[$i - 2];
    }
    return $dp[$n];
}

// ৩. Space Optimized — O(1) মেমরি
function climbStairsOpt($n) {
    if ($n <= 1) return 1;
    $prev2 = 1; $prev1 = 1;

    for ($i = 2; $i <= $n; $i++) {
        $curr = $prev1 + $prev2;
        $prev2 = $prev1;
        $prev1 = $curr;
    }
    return $prev1;
}

echo climbStairsOpt(5); // ৮
?>
```

**JS — Climbing Stairs:**

```javascript
// ১. Memoization পদ্ধতি
function climbStairsMemo(n, memo = {}) {
    if (n in memo) return memo[n];
    if (n <= 1) return 1;

    memo[n] = climbStairsMemo(n - 1, memo) + climbStairsMemo(n - 2, memo);
    return memo[n];
}

// ২. Tabulation পদ্ধতি
function climbStairsTab(n) {
    if (n <= 1) return 1;
    const dp = new Array(n + 1).fill(0);
    dp[0] = 1;
    dp[1] = 1;

    for (let i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
}

// ৩. Space Optimized — O(1) মেমরি
function climbStairsOpt(n) {
    if (n <= 1) return 1;
    let prev2 = 1, prev1 = 1;

    for (let i = 2; i <= n; i++) {
        const curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

console.log(climbStairsOpt(5)); // ৮
```

### ⏱️ Big O বিশ্লেষণ

```
┌─────────────────┬──────────┬──────────┐
│ পদ্ধতি           │ সময়      │ মেমরি    │
├─────────────────┼──────────┼──────────┤
│ Naive Recursion │ O(2ⁿ)    │ O(n)     │
│ Memoization     │ O(n)     │ O(n)     │
│ Tabulation      │ O(n)     │ O(n)     │
│ Space Optimized │ O(n)     │ O(1)     │
└─────────────────┴──────────┴──────────┘
```

---

## 🏠 House Robber — বাড়ি ডাকাতি

### 📖 সমস্যা

একসারি বাড়ি আছে, প্রতিটিতে নির্দিষ্ট পরিমাণ টাকা। পাশাপাশি দুটি বাড়ি ডাকাতি করা যাবে না
(অ্যালার্ম বাজবে!)। সর্বোচ্চ কত টাকা নেওয়া যায়?

### 🏠 বাস্তব উদাহরণ

```
বাড়ি:    [২]  [৭]  [৯]  [৩]  [১]
ইনডেক্স:  0    1    2    3    4

❌ বাড়ি ০ এবং ১ একসাথে নেওয়া যাবে না (পাশাপাশি)
✅ বাড়ি ০, ২, ৪ নেওয়া যায় = ২+৯+১ = ১২
✅ বাড়ি ১, ৩ নেওয়া যায় = ৭+৩ = ১০
✅ বাড়ি ১, ৪ নেওয়া যায় = ৭+১ = ৮
```

### State ও Recurrence Relation

```
State: dp[i] = প্রথম i টি বাড়ি থেকে সর্বোচ্চ কত টাকা

প্রতিটি বাড়ির জন্য দুটি অপশন:
  ১. এই বাড়ি ডাকাতি করো  → nums[i] + dp[i-2]
  ২. এই বাড়ি ছেড়ে দাও    → dp[i-1]

Recurrence: dp[i] = max(nums[i] + dp[i-2], dp[i-1])
```

### ASCII ট্রেস

```
nums = [2, 7, 9, 3, 1]

i=0: dp[0] = 2                          (শুধু প্রথম বাড়ি)
i=1: dp[1] = max(7, 2) = 7              (বাড়ি ০ বা ১ এর মধ্যে সেরা)
i=2: dp[2] = max(9+2, 7) = max(11,7) = 11  (বাড়ি ২ নেও + dp[0])
i=3: dp[3] = max(3+7, 11) = max(10,11) = 11 (বাড়ি ৩ বাদ)
i=4: dp[4] = max(1+11, 11) = max(12,11) = 12 (বাড়ি ৪ নেও + dp[2])

ইনডেক্স:  0   1   2    3    4
dp:       [2] [7] [11] [11] [12]

উত্তর: ১২ (বাড়ি ০, ২, ৪ = ২+৯+১) ✅
```

**PHP — House Robber:**

```php
<?php
function houseRobber($nums) {
    $n = count($nums);
    if ($n === 0) return 0;
    if ($n === 1) return $nums[0];

    // dp[i] = i-তম বাড়ি পর্যন্ত সর্বোচ্চ লুট
    $prev2 = 0;           // dp[i-2]
    $prev1 = $nums[0];    // dp[i-1]

    for ($i = 1; $i < $n; $i++) {
        // এই বাড়ি নেব নাকি ছাড়ব?
        $current = max(
            $nums[$i] + $prev2,  // এই বাড়ি নিলে
            $prev1                // ছেড়ে দিলে
        );
        $prev2 = $prev1;
        $prev1 = $current;
    }

    return $prev1;
}

echo houseRobber([2, 7, 9, 3, 1]); // ১২
?>
```

**JS — House Robber:**

```javascript
function houseRobber(nums) {
    const n = nums.length;
    if (n === 0) return 0;
    if (n === 1) return nums[0];

    // শুধু আগের দুটি মান মনে রাখলেই চলে
    let prev2 = 0;
    let prev1 = nums[0];

    for (let i = 1; i < n; i++) {
        // এই বাড়ি নেব নাকি ছাড়ব?
        const current = Math.max(
            nums[i] + prev2, // এই বাড়ি নিলে
            prev1             // ছেড়ে দিলে
        );
        prev2 = prev1;
        prev1 = current;
    }

    return prev1;
}

console.log(houseRobber([2, 7, 9, 3, 1])); // ১২
```

---

## 🪙 Coin Change — মুদ্রা বিনিময়

### 📖 সমস্যা

নির্দিষ্ট কিছু মুদ্রা আছে (অসীম পরিমাণে)। একটি নির্দিষ্ট পরিমাণ টাকা বানাতে সর্বনিম্ন কয়টি মুদ্রা লাগবে?

### 🏠 বাস্তব উদাহরণ ও Greedy ব্যর্থতা

```
মুদ্রা: [1, 3, 4]   পরিমাণ: 6

❌ Greedy পদ্ধতি (সবচেয়ে বড় মুদ্রা আগে):
   4 + 1 + 1 = ৩টি মুদ্রা

✅ DP পদ্ধতি (সেরা সমাধান):
   3 + 3 = ২টি মুদ্রা

Greedy সবসময় কাজ করে না! DP দরকার।
```

### ২D টেবিল ট্রেস

```
মুদ্রা: [1, 3, 4],  পরিমাণ: 6

dp[i] = i টাকা বানাতে সর্বনিম্ন মুদ্রা সংখ্যা

পরিমাণ:  0   1   2   3   4   5   6
dp:     [0] [1] [2] [1] [1] [2] [2]

ধাপে ধাপে:
dp[0] = 0                          (০ টাকা = ০ মুদ্রা)
dp[1] = min(dp[1-1]+1) = 1        (১ টাকার ১টি মুদ্রা)
dp[2] = min(dp[2-1]+1) = 2        (১+১)
dp[3] = min(dp[3-1]+1, dp[3-3]+1) = min(3,1) = 1   (৩ টাকার ১টি মুদ্রা)
dp[4] = min(dp[4-1]+1, dp[4-3]+1, dp[4-4]+1) = min(2,2,1) = 1  (৪ টাকার ১টি)
dp[5] = min(dp[5-1]+1, dp[5-3]+1, dp[5-4]+1) = min(2,3,2) = 2  (৪+১ বা ৩+?)
dp[6] = min(dp[6-1]+1, dp[6-3]+1, dp[6-4]+1) = min(3,2,3) = 2  (৩+৩) ✅
```

**PHP — Coin Change:**

```php
<?php
function coinChange($coins, $amount) {
    // অসম্ভব বোঝাতে বড় সংখ্যা
    $dp = array_fill(0, $amount + 1, $amount + 1);
    $dp[0] = 0; // ০ টাকা বানাতে ০ মুদ্রা

    for ($i = 1; $i <= $amount; $i++) {
        foreach ($coins as $coin) {
            // এই মুদ্রা ব্যবহার করা সম্ভব হলে
            if ($coin <= $i) {
                $dp[$i] = min($dp[$i], $dp[$i - $coin] + 1);
            }
        }
    }

    // উত্তর সম্ভব কিনা যাচাই
    return $dp[$amount] > $amount ? -1 : $dp[$amount];
}

echo coinChange([1, 3, 4], 6); // ২
echo coinChange([2], 3);       // -১ (অসম্ভব)
?>
```

**JS — Coin Change:**

```javascript
function coinChange(coins, amount) {
    // অসম্ভব বোঝাতে বড় সংখ্যা
    const dp = new Array(amount + 1).fill(amount + 1);
    dp[0] = 0; // ০ টাকা বানাতে ০ মুদ্রা

    for (let i = 1; i <= amount; i++) {
        for (const coin of coins) {
            // এই মুদ্রা ব্যবহার করা সম্ভব হলে
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }

    return dp[amount] > amount ? -1 : dp[amount];
}

console.log(coinChange([1, 3, 4], 6)); // ২
console.log(coinChange([2], 3));       // -১ (অসম্ভব)
```

### ⏱️ Big O: সময় O(amount × coins), মেমরি O(amount)

---

## 📈 Longest Increasing Subsequence (LIS)

### 📖 সমস্যা

একটি অ্যারে থেকে সবচেয়ে দীর্ঘ ক্রমবর্ধমান সাবসিকোয়েন্স (পরপর থাকতে হবে না) বের করো।

### ধাপে ধাপে ট্রেস — O(n²) DP

```
nums = [10, 9, 2, 5, 3, 7, 101, 18]

dp[i] = i-তম ইলিমেন্টে শেষ হওয়া LIS-এর দৈর্ঘ্য

i=0: dp[0]=1  [10]
i=1: dp[1]=1  [9]          (৯<১০ তাই ১০ থেকে extend হবে না)
i=2: dp[2]=1  [2]          (২<৯, ২<১০)
i=3: dp[3]=2  [2,5]        (৫>২ তাই dp[2]+1=2)
i=4: dp[4]=2  [2,3]        (৩>২ তাই dp[2]+1=2)
i=5: dp[5]=3  [2,5,7]      (৭>৫ তাই dp[3]+1=3; ৭>৩ তাই dp[4]+1=3)
i=6: dp[6]=4  [2,5,7,101]  (১০১>৭ তাই dp[5]+1=4)
i=7: dp[7]=4  [2,5,7,18]   (১৮>৭ তাই dp[5]+1=4)

ইনডেক্স:  0  1  2  3  4  5  6  7
nums:    10  9  2  5  3  7 101 18
dp:       1  1  1  2  2  3  4  4

LIS দৈর্ঘ্য = max(dp) = 4 ✅
```

**PHP — LIS O(n²):**

```php
<?php
function lengthOfLIS($nums) {
    $n = count($nums);
    if ($n === 0) return 0;

    // প্রতিটি ইলিমেন্টে শেষ হওয়া LIS কমপক্ষে ১
    $dp = array_fill(0, $n, 1);

    for ($i = 1; $i < $n; $i++) {
        for ($j = 0; $j < $i; $j++) {
            // j-তম ইলিমেন্ট ছোট হলে extend করা যায়
            if ($nums[$j] < $nums[$i]) {
                $dp[$i] = max($dp[$i], $dp[$j] + 1);
            }
        }
    }

    return max($dp);
}

echo lengthOfLIS([10, 9, 2, 5, 3, 7, 101, 18]); // ৪
?>
```

**JS — LIS O(n²):**

```javascript
function lengthOfLIS(nums) {
    const n = nums.length;
    if (n === 0) return 0;

    const dp = new Array(n).fill(1);

    for (let i = 1; i < n; i++) {
        for (let j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
    }

    return Math.max(...dp);
}

console.log(lengthOfLIS([10, 9, 2, 5, 3, 7, 101, 18])); // ৪
```

### 🚀 LIS — O(n log n) Binary Search পদ্ধতি

```javascript
// ⭐ O(n log n) — patience sorting কৌশল
function lengthOfLIS_fast(nums) {
    // tails[i] = দৈর্ঘ্য i+1 এর LIS-এর সবচেয়ে ছোট শেষ ইলিমেন্ট
    const tails = [];

    for (const num of nums) {
        let lo = 0, hi = tails.length;
        // বাইনারি সার্চ — কোথায় বসাবো?
        while (lo < hi) {
            const mid = Math.floor((lo + hi) / 2);
            if (tails[mid] < num) lo = mid + 1;
            else hi = mid;
        }
        tails[lo] = num;
    }

    return tails.length;
}
```

```php
<?php
// ⭐ O(n log n) — patience sorting কৌশল
function lengthOfLIS_fast($nums) {
    $tails = [];

    foreach ($nums as $num) {
        $lo = 0;
        $hi = count($tails);
        // বাইনারি সার্চ — কোথায় বসাবো?
        while ($lo < $hi) {
            $mid = intdiv($lo + $hi, 2);
            if ($tails[$mid] < $num) $lo = $mid + 1;
            else $hi = $mid;
        }
        $tails[$lo] = $num;
    }

    return count($tails);
}
?>
```

### ⏱️ Big O: O(n²) DP, O(n log n) Binary Search

---

# ৪. ক্লাসিক 2D DP সমস্যা

## ⭐ 0/1 Knapsack — ০/১ ন্যাপস্যাক (গভীর আলোচনা)

### 📖 সমস্যা বিবরণ

একজন চোর একটি ব্যাগ নিয়ে বাড়িতে ঢুকেছে। ব্যাগে সর্বোচ্চ W কেজি ধরে।
প্রতিটি জিনিসের ওজন ও মূল্য আছে। সর্বোচ্চ কত মূল্যের জিনিস নিতে পারবে?

> **গুরুত্বপূর্ণ**: প্রতিটি জিনিস হয় পুরোটা নেবে, নয় নেবে না (0 বা 1)।
> অর্ধেক নেওয়া যাবে না (সেটা Fractional Knapsack — Greedy দিয়ে সমাধান হয়)।

### 🏠 বাস্তব উদাহরণ

```
চোরের ব্যাগ ধারণক্ষমতা: ৭ কেজি

জিনিসপত্র:
┌──────────┬────────┬────────┐
│  জিনিস    │ ওজন(কেজি)│ মূল্য(টাকা)│
├──────────┼────────┼────────┤
│ ল্যাপটপ   │   ৩    │  ৪০০০  │
│ ক্যামেরা  │   ১    │  ১৫০০  │
│ গহনা      │   ২    │  ৩০০০  │
│ ট্যাবলেট  │   ৪    │  ৩৫০০  │
└──────────┴────────┴────────┘

সর্বোচ্চ মূল্য কত?
```

### State ও Recurrence

```
State:  dp[i][w] = প্রথম i টি জিনিস এবং w ক্ষমতায় সর্বোচ্চ মূল্য

Recurrence:
  যদি wt[i] > w (জিনিস ব্যাগে ধরবে না):
      dp[i][w] = dp[i-1][w]

  অন্যথায় (নেওয়া বা না-নেওয়া — দুটোর মধ্যে সেরা):
      dp[i][w] = max(dp[i-1][w], val[i] + dp[i-1][w - wt[i]])
```

### সম্পূর্ণ 2D টেবিল ট্রেস

```
ওজন:  [3, 1, 2, 4]
মূল্য: [4000, 1500, 3000, 3500]
ক্ষমতা W = 7

      ক্ষমতা →  0     1      2      3      4      5      6      7
জিনিস ↓
  ০ (কিছুই)   0     0      0      0      0      0      0      0
  ১ (ল্যাপটপ)  0     0      0    4000   4000   4000   4000   4000
  ২ (ক্যামেরা) 0   1500   1500   4000   5500   5500   5500   5500
  ৩ (গহনা)    0   1500   3000   4500   5500   7000   8500   8500
  ৪ (ট্যাবলেট) 0   1500   3000   4500   5500   7000   8500   8500

ট্রেস (কিছু গুরুত্বপূর্ণ ঘর):

dp[1][3] = max(dp[0][3], 4000+dp[0][0]) = max(0, 4000) = 4000
    → ল্যাপটপ (ওজন ৩) নিলে ৪০০০ পাই

dp[2][4] = max(dp[1][4], 1500+dp[1][3]) = max(4000, 1500+4000) = 5500
    → ক্যামেরা (ওজন ১) + ল্যাপটপ = ৫৫০০

dp[3][5] = max(dp[2][5], 3000+dp[2][3]) = max(5500, 3000+4000) = 7000
    → গহনা + ল্যাপটপ = ৭০০০

dp[3][6] = max(dp[2][6], 3000+dp[2][4]) = max(5500, 3000+5500) = 8500
    → গহনা + ক্যামেরা + ল্যাপটপ = ৮৫০০

উত্তর: dp[4][7] = ৮৫০০ টাকা ✅
(ল্যাপটপ ৪০০০ + ক্যামেরা ১৫০০ + গহনা ৩০০০ = ৮৫০০)
```

**PHP — 0/1 Knapsack:**

```php
<?php
function knapsack01($weights, $values, $capacity) {
    $n = count($weights);

    // 2D টেবিল তৈরি
    $dp = [];
    for ($i = 0; $i <= $n; $i++) {
        $dp[$i] = array_fill(0, $capacity + 1, 0);
    }

    // টেবিল পূরণ
    for ($i = 1; $i <= $n; $i++) {
        for ($w = 0; $w <= $capacity; $w++) {
            // জিনিসটি ব্যাগে ধরবে না
            if ($weights[$i - 1] > $w) {
                $dp[$i][$w] = $dp[$i - 1][$w];
            } else {
                // নেওয়া বা না-নেওয়া — সেরাটা বেছে নাও
                $dp[$i][$w] = max(
                    $dp[$i - 1][$w],                                    // না নিলে
                    $values[$i - 1] + $dp[$i - 1][$w - $weights[$i - 1]] // নিলে
                );
            }
        }
    }

    return $dp[$n][$capacity];
}

$weights = [3, 1, 2, 4];
$values = [4000, 1500, 3000, 3500];
echo knapsack01($weights, $values, 7); // ৮৫০০
?>
```

**JS — 0/1 Knapsack:**

```javascript
function knapsack01(weights, values, capacity) {
    const n = weights.length;

    // 2D টেবিল তৈরি
    const dp = Array.from({ length: n + 1 }, () =>
        new Array(capacity + 1).fill(0)
    );

    // টেবিল পূরণ
    for (let i = 1; i <= n; i++) {
        for (let w = 0; w <= capacity; w++) {
            if (weights[i - 1] > w) {
                // জিনিসটি ব্যাগে ধরবে না
                dp[i][w] = dp[i - 1][w];
            } else {
                // নেওয়া বা না-নেওয়া — সেরাটা বেছে নাও
                dp[i][w] = Math.max(
                    dp[i - 1][w],
                    values[i - 1] + dp[i - 1][w - weights[i - 1]]
                );
            }
        }
    }

    return dp[n][capacity];
}

const weights = [3, 1, 2, 4];
const values = [4000, 1500, 3000, 3500];
console.log(knapsack01(weights, values, 7)); // ৮৫০০
```

### 🚀 Space Optimization — 1D Array

```php
<?php
// ⭐ O(W) মেমরিতে 0/1 Knapsack
function knapsack01_optimized($weights, $values, $capacity) {
    $n = count($weights);
    $dp = array_fill(0, $capacity + 1, 0);

    for ($i = 0; $i < $n; $i++) {
        // ⚠️ ডান থেকে বামে যেতে হবে (0/1 এর জন্য)
        for ($w = $capacity; $w >= $weights[$i]; $w--) {
            $dp[$w] = max($dp[$w], $values[$i] + $dp[$w - $weights[$i]]);
        }
    }

    return $dp[$capacity];
}
?>
```

```javascript
// ⭐ O(W) মেমরিতে 0/1 Knapsack
function knapsack01_optimized(weights, values, capacity) {
    const n = weights.length;
    const dp = new Array(capacity + 1).fill(0);

    for (let i = 0; i < n; i++) {
        // ⚠️ ডান থেকে বামে যেতে হবে (0/1 এর জন্য)
        for (let w = capacity; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
        }
    }

    return dp[capacity];
}
```

### ⏱️ Big O: সময় O(n×W), মেমরি O(n×W) → অপটিমাইজড O(W)

---

## ⭐ Longest Common Subsequence (LCS)

### 📖 সমস্যা

দুটি স্ট্রিং-এর মধ্যে সবচেয়ে দীর্ঘ কমন সাবসিকোয়েন্স বের করো।

### 2D টেবিল ট্রেস

```
s1 = "ABCBDAB"
s2 = "BDCAB"

     ""  B  D  C  A  B
  ""  0  0  0  0  0  0
  A   0  0  0  0  1  1
  B   0  1  1  1  1  2
  C   0  1  1  2  2  2
  B   0  1  1  2  2  3
  D   0  1  2  2  2  3
  A   0  1  2  2  3  3
  B   0  1  2  2  3  4

নিয়ম:
  যদি s1[i] == s2[j]:  dp[i][j] = dp[i-1][j-1] + 1  (↖ তীর)
  অন্যথায়:            dp[i][j] = max(dp[i-1][j], dp[i][j-1])

LCS দৈর্ঘ্য = dp[7][5] = 4

Backtrack (↖ তীর অনুসরণ করে LCS বের করো):
dp[7][5] → B (↖)
dp[6][4] → A (↖)
dp[4][2] → ... → "BCAB"

LCS = "BCAB" ✅
```

**PHP — LCS:**

```php
<?php
function lcs($s1, $s2) {
    $m = strlen($s1);
    $n = strlen($s2);

    // 2D টেবিল তৈরি
    $dp = [];
    for ($i = 0; $i <= $m; $i++) {
        $dp[$i] = array_fill(0, $n + 1, 0);
    }

    // টেবিল পূরণ
    for ($i = 1; $i <= $m; $i++) {
        for ($j = 1; $j <= $n; $j++) {
            if ($s1[$i - 1] === $s2[$j - 1]) {
                // অক্ষর মিলেছে — কোণাকুণি থেকে +১
                $dp[$i][$j] = $dp[$i - 1][$j - 1] + 1;
            } else {
                // মেলেনি — উপর বা বাম থেকে সেরাটা
                $dp[$i][$j] = max($dp[$i - 1][$j], $dp[$i][$j - 1]);
            }
        }
    }

    // ব্যাকট্র্যাক করে আসল LCS বের করো
    $lcsStr = '';
    $i = $m;
    $j = $n;
    while ($i > 0 && $j > 0) {
        if ($s1[$i - 1] === $s2[$j - 1]) {
            $lcsStr = $s1[$i - 1] . $lcsStr;
            $i--;
            $j--;
        } elseif ($dp[$i - 1][$j] > $dp[$i][$j - 1]) {
            $i--;
        } else {
            $j--;
        }
    }

    return ['দৈর্ঘ্য' => $dp[$m][$n], 'LCS' => $lcsStr];
}

$result = lcs("ABCBDAB", "BDCAB");
echo "LCS দৈর্ঘ্য: " . $result['দৈর্ঘ্য'] . "\n"; // ৪
echo "LCS: " . $result['LCS'] . "\n";               // BCAB
?>
```

**JS — LCS:**

```javascript
function lcs(s1, s2) {
    const m = s1.length, n = s2.length;

    // 2D টেবিল তৈরি
    const dp = Array.from({ length: m + 1 }, () =>
        new Array(n + 1).fill(0)
    );

    // টেবিল পূরণ
    for (let i = 1; i <= m; i++) {
        for (let j = 1; j <= n; j++) {
            if (s1[i - 1] === s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }

    // ব্যাকট্র্যাক করে আসল LCS বের করো
    let lcsStr = '';
    let i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1[i - 1] === s2[j - 1]) {
            lcsStr = s1[i - 1] + lcsStr;
            i--; j--;
        } else if (dp[i - 1][j] > dp[i][j - 1]) {
            i--;
        } else {
            j--;
        }
    }

    return { দৈর্ঘ্য: dp[m][n], LCS: lcsStr };
}

const result = lcs("ABCBDAB", "BDCAB");
console.log(`LCS দৈর্ঘ্য: ${result.দৈর্ঘ্য}`); // ৪
console.log(`LCS: ${result.LCS}`);               // BCAB
```

### ⏱️ Big O: সময় O(m×n), মেমরি O(m×n)

---

## ✏️ Edit Distance (Levenshtein Distance)

### 📖 সমস্যা

একটি শব্দকে অন্য শব্দে রূপান্তর করতে সর্বনিম্ন কয়টি অপারেশন দরকার?

তিনটি অপারেশন:
- **Insert** (ঢোকানো)
- **Delete** (মোছা)
- **Replace** (বদলানো)

### 🏠 বাস্তব উদাহরণ

```
"kitten" → "sitting" রূপান্তর:

kitten → sitten  (k → s বদলানো)
sitten → sittin  (e → i বদলানো)
sittin → sitting (g ঢোকানো)

মোট ৩টি অপারেশন
```

### 2D টেবিল ট্রেস

```
s1 = "CAT",  s2 = "CUT"

       ""   C    U    T
  ""    0    1    2    3
  C     1    0    1    2
  A     2    1    1    2
  T     3    2    2    1

নিয়ম:
  যদি s1[i] == s2[j]:  dp[i][j] = dp[i-1][j-1]     (কোনো পরিবর্তন নেই)
  অন্যথায়:            dp[i][j] = 1 + min(
                          dp[i-1][j-1],  ← বদলানো (Replace)
                          dp[i-1][j],    ← মোছা (Delete)
                          dp[i][j-1]     ← ঢোকানো (Insert)
                       )

dp[1][1] = 0  (C==C, কিছু করতে হবে না)
dp[2][2] = 1  (A≠U, min(0,1,1)+1 = 1, A→U বদলানো)
dp[3][3] = 1  (T==T, dp[2][2] = 1)

উত্তর: dp[3][3] = 1 ✅ (শুধু A→U বদলানো)
```

**PHP — Edit Distance:**

```php
<?php
function editDistance($s1, $s2) {
    $m = strlen($s1);
    $n = strlen($s2);

    $dp = [];
    for ($i = 0; $i <= $m; $i++) {
        $dp[$i] = array_fill(0, $n + 1, 0);
    }

    // ভিত্তি শর্ত: খালি স্ট্রিং থেকে রূপান্তর
    for ($i = 0; $i <= $m; $i++) $dp[$i][0] = $i; // সব মুছে ফেলো
    for ($j = 0; $j <= $n; $j++) $dp[0][$j] = $j; // সব ঢোকাও

    for ($i = 1; $i <= $m; $i++) {
        for ($j = 1; $j <= $n; $j++) {
            if ($s1[$i - 1] === $s2[$j - 1]) {
                // অক্ষর মিলেছে — কোনো খরচ নেই
                $dp[$i][$j] = $dp[$i - 1][$j - 1];
            } else {
                // তিনটি অপশনের মধ্যে সর্বনিম্ন + ১
                $dp[$i][$j] = 1 + min(
                    $dp[$i - 1][$j - 1], // বদলানো
                    $dp[$i - 1][$j],      // মোছা
                    $dp[$i][$j - 1]       // ঢোকানো
                );
            }
        }
    }

    return $dp[$m][$n];
}

echo editDistance("CAT", "CUT");       // ১
echo editDistance("kitten", "sitting"); // ৩
?>
```

**JS — Edit Distance:**

```javascript
function editDistance(s1, s2) {
    const m = s1.length, n = s2.length;

    const dp = Array.from({ length: m + 1 }, () =>
        new Array(n + 1).fill(0)
    );

    // ভিত্তি শর্ত
    for (let i = 0; i <= m; i++) dp[i][0] = i;
    for (let j = 0; j <= n; j++) dp[0][j] = j;

    for (let i = 1; i <= m; i++) {
        for (let j = 1; j <= n; j++) {
            if (s1[i - 1] === s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j - 1], // বদলানো
                    dp[i - 1][j],      // মোছা
                    dp[i][j - 1]       // ঢোকানো
                );
            }
        }
    }

    return dp[m][n];
}

console.log(editDistance("CAT", "CUT"));       // ১
console.log(editDistance("kitten", "sitting")); // ৩
```

---

## 🔗 Matrix Chain Multiplication

### 📖 সমস্যা

কয়েকটি ম্যাট্রিক্স পরপর গুণ করতে হবে। কোন ক্রমে গুণ করলে সবচেয়ে কম
স্কেলার গুণন (scalar multiplication) লাগবে?

### 🏠 বাস্তব উদাহরণ

```
ম্যাট্রিক্স: A(10×30), B(30×5), C(5×60)

❌ (A×B)×C:
   A×B = 10×30×5 = 1500 গুণন → ফলাফল 10×5
   (AB)×C = 10×5×60 = 3000 গুণন
   মোট = 4500 গুণন

✅ A×(B×C):
   B×C = 30×5×60 = 9000 গুণন → ফলাফল 30×60
   A×(BC) = 10×30×60 = 18000 গুণন
   মোট = 27000 গুণন

অবাক! (A×B)×C অনেক ভালো!
কিন্তু সবসময় এত সহজ না — DP দিয়ে সেরাটা খুঁজতে হয়।
```

### Interval DP ধারণা

```
dims = [10, 30, 5, 60]  → A(10×30), B(30×5), C(5×60)

dp[i][j] = i থেকে j পর্যন্ত ম্যাট্রিক্স গুণের সর্বনিম্ন খরচ

dp[i][j] = min over k { dp[i][k] + dp[k+1][j] + dims[i-1]×dims[k]×dims[j] }
           i ≤ k < j
```

**PHP — Matrix Chain Multiplication:**

```php
<?php
function matrixChainOrder($dims) {
    $n = count($dims) - 1; // ম্যাট্রিক্স সংখ্যা
    $dp = [];
    for ($i = 0; $i <= $n; $i++) {
        $dp[$i] = array_fill(0, $n + 1, 0);
    }

    // চেইন দৈর্ঘ্য ২ থেকে শুরু
    for ($len = 2; $len <= $n; $len++) {
        for ($i = 1; $i <= $n - $len + 1; $i++) {
            $j = $i + $len - 1;
            $dp[$i][$j] = PHP_INT_MAX;

            // সব সম্ভাব্য বিভাজন চেষ্টা করো
            for ($k = $i; $k < $j; $k++) {
                $cost = $dp[$i][$k] + $dp[$k + 1][$j]
                        + $dims[$i - 1] * $dims[$k] * $dims[$j];
                $dp[$i][$j] = min($dp[$i][$j], $cost);
            }
        }
    }

    return $dp[1][$n];
}

echo matrixChainOrder([10, 30, 5, 60]); // ৪৫০০
?>
```

**JS — Matrix Chain Multiplication:**

```javascript
function matrixChainOrder(dims) {
    const n = dims.length - 1;
    const dp = Array.from({ length: n + 1 }, () =>
        new Array(n + 1).fill(0)
    );

    for (let len = 2; len <= n; len++) {
        for (let i = 1; i <= n - len + 1; i++) {
            const j = i + len - 1;
            dp[i][j] = Infinity;

            for (let k = i; k < j; k++) {
                const cost = dp[i][k] + dp[k + 1][j]
                           + dims[i - 1] * dims[k] * dims[j];
                dp[i][j] = Math.min(dp[i][j], cost);
            }
        }
    }

    return dp[1][n];
}

console.log(matrixChainOrder([10, 30, 5, 60])); // ৪৫০০
```

### ⏱️ Big O: সময় O(n³), মেমরি O(n²)

---

# ৫. DP প্যাটার্ন চেনার কৌশল

## 🗂️ DP প্যাটার্ন ক্যাটালগ

### ১. Linear DP (রৈখিক DP)

```
📝 চেনার উপায়: 1D অ্যারে বা সিকোয়েন্সের উপর কাজ
🔑 State: dp[i] = i-তম পজিশন পর্যন্ত উত্তর
📐 Template:

for i = 0 to n:
    dp[i] = f(dp[i-1], dp[i-2], ...)

উদাহরণ: Fibonacci, Climbing Stairs, House Robber, Maximum Subarray
```

### ২. Grid DP (গ্রিড DP)

```
📝 চেনার উপায়: 2D গ্রিডে পথ খোঁজা, শুধু ডানে বা নিচে যাওয়া যায়
🔑 State: dp[i][j] = (i,j) ঘরে পৌঁছানোর উত্তর
📐 Template:

for i = 0 to rows:
    for j = 0 to cols:
        dp[i][j] = f(dp[i-1][j], dp[i][j-1])

উদাহরণ: Unique Paths, Minimum Path Sum
```

### ৩. String DP (স্ট্রিং DP)

```
📝 চেনার উপায়: দুটি স্ট্রিং তুলনা বা একটি স্ট্রিং-এ প্যাটার্ন খোঁজা
🔑 State: dp[i][j] = s1 এর প্রথম i ও s2 এর প্রথম j অক্ষর নিয়ে উত্তর
📐 Template:

for i = 0 to len(s1):
    for j = 0 to len(s2):
        if s1[i] == s2[j]:
            dp[i][j] = dp[i-1][j-1] + ...
        else:
            dp[i][j] = max/min(dp[i-1][j], dp[i][j-1])

উদাহরণ: LCS, Edit Distance, Longest Palindromic Subsequence
```

### ৪. Knapsack Variants (ন্যাপস্যাক ভ্যারিয়েন্ট)

```
📝 চেনার উপায়: সীমিত capacity-তে আইটেম select করা, maximize/minimize
🔑 State: dp[i][w] = i আইটেম ও w ক্ষমতায় উত্তর

প্রকারভেদ:
┌──────────────────┬────────────────────────┐
│ প্রকার           │ বৈশিষ্ট্য               │
├──────────────────┼────────────────────────┤
│ 0/1 Knapsack     │ প্রতিটি আইটেম ১বার      │
│ Unbounded        │ আইটেম অসীমবার          │
│ Subset Sum       │ নির্দিষ্ট যোগফল সম্ভব?  │
│ Partition Equal   │ দুই ভাগে সমান ভাগ       │
└──────────────────┴────────────────────────┘

উদাহরণ: Coin Change, Target Sum, Partition Problem
```

### ৫. Interval DP (ইন্টারভাল DP)

```
📝 চেনার উপায়: একটি রেঞ্জকে সব জায়গায় ভেঙে সেরাটা খোঁজা
🔑 State: dp[i][j] = i থেকে j রেঞ্জের উত্তর
📐 Template:

for len = 2 to n:
    for i = 0 to n-len:
        j = i + len
        for k = i to j:
            dp[i][j] = best(dp[i][k] + dp[k][j] + cost)

উদাহরণ: Matrix Chain, Burst Balloons, Stone Game
```

---

# ৬. Grid DP Problems

## 🛤️ Unique Paths — অনন্য পথ

### 📖 সমস্যা

m×n গ্রিডের উপর-বামে (0,0) থেকে নিচে-ডানে (m-1,n-1) যেতে হবে।
শুধু ডানে (→) বা নিচে (↓) যাওয়া যায়। কতটি অনন্য পথ আছে?

### ASCII গ্রিড ট্রেস

```
3×4 গ্রিড:

dp টেবিল:
     col0  col1  col2  col3
row0 [ 1 ] [ 1 ] [ 1 ] [ 1 ]
row1 [ 1 ] [ 2 ] [ 3 ] [ 4 ]
row2 [ 1 ] [ 3 ] [ 6 ] [10 ]

নিয়ম: dp[i][j] = dp[i-1][j] + dp[i][j-1]
       (উপর থেকে + বাম থেকে)

প্রথম সারি ও প্রথম কলাম সব ১ (একটিমাত্র পথ)

dp[1][1] = dp[0][1] + dp[1][0] = 1+1 = 2
dp[1][2] = dp[0][2] + dp[1][1] = 1+2 = 3
dp[2][2] = dp[1][2] + dp[2][1] = 3+3 = 6
dp[2][3] = dp[1][3] + dp[2][2] = 4+6 = 10

উত্তর: ১০টি অনন্য পথ ✅
```

**PHP — Unique Paths:**

```php
<?php
function uniquePaths($m, $n) {
    // 2D টেবিল
    $dp = [];
    for ($i = 0; $i < $m; $i++) {
        $dp[$i] = array_fill(0, $n, 0);
    }

    // প্রথম সারি ও কলাম = ১
    for ($i = 0; $i < $m; $i++) $dp[$i][0] = 1;
    for ($j = 0; $j < $n; $j++) $dp[0][$j] = 1;

    // বাকি ঘর পূরণ
    for ($i = 1; $i < $m; $i++) {
        for ($j = 1; $j < $n; $j++) {
            // উপরের ঘর + বামের ঘর
            $dp[$i][$j] = $dp[$i - 1][$j] + $dp[$i][$j - 1];
        }
    }

    return $dp[$m - 1][$n - 1];
}

echo uniquePaths(3, 4); // ১০
?>
```

**JS — Unique Paths:**

```javascript
function uniquePaths(m, n) {
    const dp = Array.from({ length: m }, () => new Array(n).fill(0));

    // প্রথম সারি ও কলাম = ১
    for (let i = 0; i < m; i++) dp[i][0] = 1;
    for (let j = 0; j < n; j++) dp[0][j] = 1;

    // বাকি ঘর পূরণ
    for (let i = 1; i < m; i++) {
        for (let j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }

    return dp[m - 1][n - 1];
}

console.log(uniquePaths(3, 4)); // ১০
```

---

## 🏔️ Minimum Path Sum — সর্বনিম্ন পথ খরচ

### 📖 সমস্যা

m×n গ্রিডে প্রতিটি ঘরে একটি সংখ্যা (খরচ) আছে। উপর-বাম থেকে নিচে-ডানে
সর্বনিম্ন খরচে যেতে হবে। শুধু ডানে বা নিচে যাওয়া যায়।

### ASCII গ্রিড ট্রেস

```
মূল গ্রিড:          dp টেবিল:
[1] [3] [1]         [1] [4] [5]
[1] [5] [1]    →    [2] [7] [6]
[4] [2] [1]         [6] [8] [7]

dp[0][0] = 1
dp[0][1] = 1+3 = 4
dp[0][2] = 4+1 = 5
dp[1][0] = 1+1 = 2
dp[1][1] = min(4,2)+5 = 2+5 = 7
dp[1][2] = min(5,7)+1 = 5+1 = 6
dp[2][0] = 2+4 = 6
dp[2][1] = min(7,6)+2 = 6+2 = 8
dp[2][2] = min(6,8)+1 = 6+1 = 7

সেরা পথ: 1 → 3 → 1 → 1 → 1 = 7 ✅
         অথবা: 1 → 1 → 5 → 1 → 1 ≠ (এটা ৯)
```

**PHP — Minimum Path Sum:**

```php
<?php
function minPathSum($grid) {
    $m = count($grid);
    $n = count($grid[0]);

    // প্রথম সারি — শুধু বাম থেকে আসতে পারে
    for ($j = 1; $j < $n; $j++) {
        $grid[0][$j] += $grid[0][$j - 1];
    }

    // প্রথম কলাম — শুধু উপর থেকে আসতে পারে
    for ($i = 1; $i < $m; $i++) {
        $grid[$i][0] += $grid[$i - 1][0];
    }

    // বাকি ঘর — উপর ও বামের মধ্যে ছোটটা + বর্তমান খরচ
    for ($i = 1; $i < $m; $i++) {
        for ($j = 1; $j < $n; $j++) {
            $grid[$i][$j] += min($grid[$i - 1][$j], $grid[$i][$j - 1]);
        }
    }

    return $grid[$m - 1][$n - 1];
}

$grid = [[1, 3, 1], [1, 5, 1], [4, 2, 1]];
echo minPathSum($grid); // ৭
?>
```

**JS — Minimum Path Sum:**

```javascript
function minPathSum(grid) {
    const m = grid.length, n = grid[0].length;

    // প্রথম সারি
    for (let j = 1; j < n; j++) {
        grid[0][j] += grid[0][j - 1];
    }

    // প্রথম কলাম
    for (let i = 1; i < m; i++) {
        grid[i][0] += grid[i - 1][0];
    }

    // বাকি ঘর
    for (let i = 1; i < m; i++) {
        for (let j = 1; j < n; j++) {
            grid[i][j] += Math.min(grid[i - 1][j], grid[i][j - 1]);
        }
    }

    return grid[m - 1][n - 1];
}

console.log(minPathSum([[1, 3, 1], [1, 5, 1], [4, 2, 1]])); // ৭
```

### ⏱️ Big O (Grid DP): সময় O(m×n), মেমরি O(1) (in-place) বা O(m×n)

---

# ৭. সম্পূর্ণ প্র্যাক্টিক্যাল প্রজেক্ট

## 🗺️ ট্রাভেল প্ল্যানার অপটিমাইজার

### 📖 সমস্যা বিবরণ

তুমি বাংলাদেশ ভ্রমণ করতে চাও। কয়েকটি শহর আছে, প্রতিটি শহরের মধ্যে দূরত্ব
এবং হোটেল খরচ আছে। সবগুলো শহর ঘুরে সর্বনিম্ন খরচে ফিরে আসতে হবে।

এটি TSP (Travelling Salesman Problem) এর একটি ভ্যারিয়েন্ট।
**DP with Bitmask** ব্যবহার করে সমাধান করবো।

### ⚙️ Bitmask DP ধারণা

```
৪টি শহর: ঢাকা(0), চট্টগ্রাম(1), সিলেট(2), রাজশাহী(3)

Bitmask দিয়ে কোন কোন শহর ভিজিট হয়েছে ট্র্যাক করি:

0000 (0)  = কোনো শহর ভিজিট হয়নি
0001 (1)  = শুধু ঢাকা
0011 (3)  = ঢাকা + চট্টগ্রাম
0111 (7)  = ঢাকা + চট্টগ্রাম + সিলেট
1111 (15) = সব শহর ভিজিট হয়েছে

State: dp[mask][i] = mask-এ থাকা শহরগুলো ভিজিট করে
                     শহর i-তে থাকলে সর্বনিম্ন খরচ
```

### ASCII ডায়াগ্রাম

```
শহরের মানচিত্র ও খরচ:

        ঢাকা (0)
       / |  \
   200/ 350\ \500
     /   |    \
চট্টগ্রাম(1)--সিলেট(2)
     \  150  /
   400\    /300
       \  /
    রাজশাহী(3)

দূরত্ব ম্যাট্রিক্স (ভ্রমণ + হোটেল খরচ):
      ঢাকা  চট্টগ্রাম সিলেট রাজশাহী
ঢাকা   ∞     200      350    500
চট্টগ্রাম200   ∞      150    400
সিলেট  350   150       ∞     300
রাজশাহী500   400      300     ∞
```

**PHP — Travel Planner (DP + Bitmask):**

```php
<?php
/**
 * ট্রাভেল প্ল্যানার অপটিমাইজার
 * TSP সমাধান — DP with Bitmask
 */
function travelPlanner($cost) {
    $n = count($cost);
    $allVisited = (1 << $n) - 1; // সব বিট ১ = সব শহর ভিজিট

    // dp[mask][i] = mask শহর ভিজিট করে i-তে আছি — সর্বনিম্ন খরচ
    $INF = PHP_INT_MAX;
    $dp = [];
    for ($mask = 0; $mask <= $allVisited; $mask++) {
        $dp[$mask] = array_fill(0, $n, $INF);
    }

    // শুরু: শহর ০ (ঢাকা) থেকে যাত্রা
    $dp[1][0] = 0; // শুধু শহর ০ ভিজিট, খরচ ০

    // সব সম্ভাব্য mask ও শহর চেষ্টা করো
    for ($mask = 1; $mask <= $allVisited; $mask++) {
        for ($u = 0; $u < $n; $u++) {
            // u শহরে আছি এবং mask-এ আছে
            if ($dp[$mask][$u] === $INF) continue;
            if (!(($mask >> $u) & 1)) continue;

            // পরবর্তী শহর v-তে যাও
            for ($v = 0; $v < $n; $v++) {
                // v ভিজিট হয়নি এবং পথ আছে
                if (($mask >> $v) & 1) continue;
                if ($cost[$u][$v] === $INF) continue;

                $newMask = $mask | (1 << $v);
                $newCost = $dp[$mask][$u] + $cost[$u][$v];

                if ($newCost < $dp[$newMask][$v]) {
                    $dp[$newMask][$v] = $newCost;
                }
            }
        }
    }

    // সব শহর ঘুরে শহর ০-তে ফেরত
    $minCost = $INF;
    for ($u = 1; $u < $n; $u++) {
        if ($dp[$allVisited][$u] !== $INF && $cost[$u][0] !== $INF) {
            $minCost = min($minCost, $dp[$allVisited][$u] + $cost[$u][0]);
        }
    }

    return $minCost === $INF ? -1 : $minCost;
}

// পথ বের করার ফাংশন
function findRoute($cost) {
    $n = count($cost);
    $allVisited = (1 << $n) - 1;
    $INF = PHP_INT_MAX;

    $dp = [];
    $parent = []; // পথ ট্র্যাকিং
    for ($mask = 0; $mask <= $allVisited; $mask++) {
        $dp[$mask] = array_fill(0, $n, $INF);
        $parent[$mask] = array_fill(0, $n, -1);
    }

    $dp[1][0] = 0;

    for ($mask = 1; $mask <= $allVisited; $mask++) {
        for ($u = 0; $u < $n; $u++) {
            if ($dp[$mask][$u] === $INF) continue;
            if (!(($mask >> $u) & 1)) continue;

            for ($v = 0; $v < $n; $v++) {
                if (($mask >> $v) & 1) continue;
                if ($cost[$u][$v] === $INF) continue;

                $newMask = $mask | (1 << $v);
                $newCost = $dp[$mask][$u] + $cost[$u][$v];

                if ($newCost < $dp[$newMask][$v]) {
                    $dp[$newMask][$v] = $newCost;
                    $parent[$newMask][$v] = $u;
                }
            }
        }
    }

    // সেরা শেষ শহর খোঁজো
    $minCost = $INF;
    $lastCity = -1;
    for ($u = 1; $u < $n; $u++) {
        if ($dp[$allVisited][$u] !== $INF && $cost[$u][0] !== $INF) {
            $total = $dp[$allVisited][$u] + $cost[$u][0];
            if ($total < $minCost) {
                $minCost = $total;
                $lastCity = $u;
            }
        }
    }

    // ব্যাকট্র্যাক করে রুট বের করো
    $route = [0]; // শহর ০ থেকে শুরু
    $mask = $allVisited;
    $city = $lastCity;

    $path = [];
    while ($city !== 0) {
        $path[] = $city;
        $prevCity = $parent[$mask][$city];
        $mask = $mask ^ (1 << $city);
        $city = $prevCity;
    }

    $route = array_merge($route, array_reverse($path));
    $route[] = 0; // শহর ০-তে ফেরত

    return ['খরচ' => $minCost, 'রুট' => $route];
}

// ========== পরীক্ষা ==========
$INF = PHP_INT_MAX;
$cityNames = ['ঢাকা', 'চট্টগ্রাম', 'সিলেট', 'রাজশাহী'];

$cost = [
    [  $INF,  200,  350,  500],
    [  200, $INF,  150,  400],
    [  350,  150, $INF,  300],
    [  500,  400,  300, $INF],
];

$result = findRoute($cost);

echo "সর্বনিম্ন ভ্রমণ খরচ: " . $result['খরচ'] . " টাকা\n";
echo "সেরা রুট: ";
foreach ($result['রুট'] as $idx => $city) {
    echo $cityNames[$city];
    if ($idx < count($result['রুট']) - 1) echo " → ";
}
echo "\n";

// প্রত্যাশিত আউটপুট:
// সর্বনিম্ন ভ্রমণ খরচ: ১০৫০ টাকা
// সেরা রুট: ঢাকা → চট্টগ্রাম → সিলেট → রাজশাহী → ঢাকা
?>
```

**JS — Travel Planner (DP + Bitmask):**

```javascript
/**
 * ট্রাভেল প্ল্যানার অপটিমাইজার
 * TSP সমাধান — DP with Bitmask
 */
function travelPlanner(cost) {
    const n = cost.length;
    const allVisited = (1 << n) - 1;
    const INF = Infinity;

    // dp[mask][i] = mask শহর ভিজিট করে i-তে আছি — সর্বনিম্ন খরচ
    const dp = Array.from({ length: allVisited + 1 }, () =>
        new Array(n).fill(INF)
    );
    const parent = Array.from({ length: allVisited + 1 }, () =>
        new Array(n).fill(-1)
    );

    dp[1][0] = 0; // শহর ০ থেকে শুরু

    for (let mask = 1; mask <= allVisited; mask++) {
        for (let u = 0; u < n; u++) {
            if (dp[mask][u] === INF) continue;
            if (!((mask >> u) & 1)) continue;

            for (let v = 0; v < n; v++) {
                if ((mask >> v) & 1) continue;
                if (cost[u][v] === INF) continue;

                const newMask = mask | (1 << v);
                const newCost = dp[mask][u] + cost[u][v];

                if (newCost < dp[newMask][v]) {
                    dp[newMask][v] = newCost;
                    parent[newMask][v] = u;
                }
            }
        }
    }

    // সেরা শেষ শহর ও রুট বের করো
    let minCost = INF, lastCity = -1;
    for (let u = 1; u < n; u++) {
        if (dp[allVisited][u] !== INF && cost[u][0] !== INF) {
            const total = dp[allVisited][u] + cost[u][0];
            if (total < minCost) {
                minCost = total;
                lastCity = u;
            }
        }
    }

    // ব্যাকট্র্যাক
    const path = [];
    let mask = allVisited, city = lastCity;
    while (city !== 0) {
        path.push(city);
        const prev = parent[mask][city];
        mask = mask ^ (1 << city);
        city = prev;
    }
    path.reverse();
    const route = [0, ...path, 0];

    return { খরচ: minCost, রুট: route };
}

// ========== পরীক্ষা ==========
const INF = Infinity;
const cityNames = ['ঢাকা', 'চট্টগ্রাম', 'সিলেট', 'রাজশাহী'];

const cost = [
    [INF, 200, 350, 500],
    [200, INF, 150, 400],
    [350, 150, INF, 300],
    [500, 400, 300, INF],
];

const result = travelPlanner(cost);
console.log(`সর্বনিম্ন ভ্রমণ খরচ: ${result.খরচ} টাকা`);
console.log(`সেরা রুট: ${result.রুট.map(i => cityNames[i]).join(' → ')}`);

// প্রত্যাশিত আউটপুট:
// সর্বনিম্ন ভ্রমণ খরচ: ১০৫০ টাকা
// সেরা রুট: ঢাকা → চট্টগ্রাম → সিলেট → রাজশাহী → ঢাকা
```

### ⏱️ Big O: সময় O(2ⁿ × n²), মেমরি O(2ⁿ × n)

```
⚠️ n বড় হলে (>20) এই পদ্ধতি ধীর।
বাস্তবে বড় TSP-এর জন্য approximation algorithm ব্যবহার করা হয়।
কিন্তু DP+Bitmask সঠিক উত্তর দেয় ছোট n-এর জন্য।
```

---

# 📋 DP প্যাটার্ন চিটশীট

```
┌────────────────────┬──────────────────────┬─────────────┬──────────────┐
│ প্যাটার্ন           │ উদাহরণ               │ সময়         │ মেমরি        │
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ Linear (1D)        │ Fibonacci            │ O(n)        │ O(n)→O(1)   │
│                    │ Climbing Stairs      │ O(n)        │ O(1)         │
│                    │ House Robber         │ O(n)        │ O(1)         │
│                    │ Coin Change          │ O(n×coins)  │ O(n)         │
│                    │ LIS                  │ O(n²)/O(nlogn)│ O(n)       │
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ Grid (2D)          │ Unique Paths         │ O(m×n)      │ O(m×n)→O(n) │
│                    │ Min Path Sum         │ O(m×n)      │ O(1) in-place│
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ String (2D)        │ LCS                  │ O(m×n)      │ O(m×n)       │
│                    │ Edit Distance        │ O(m×n)      │ O(m×n)→O(n) │
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ Knapsack           │ 0/1 Knapsack         │ O(n×W)      │ O(n×W)→O(W) │
│                    │ Unbounded Knapsack   │ O(n×W)      │ O(W)         │
│                    │ Subset Sum           │ O(n×S)      │ O(S)         │
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ Interval           │ Matrix Chain Mult.   │ O(n³)       │ O(n²)        │
│                    │ Burst Balloons       │ O(n³)       │ O(n²)        │
├────────────────────┼──────────────────────┼─────────────┼──────────────┤
│ Bitmask            │ TSP                  │ O(2ⁿ×n²)   │ O(2ⁿ×n)     │
└────────────────────┴──────────────────────┴─────────────┴──────────────┘
```

---

# 🎯 DP সমস্যা সমাধানের ৫-ধাপ ফ্রেমওয়ার্ক

## ধাপ ১: সমস্যা চিনতে পারো — "এটা কি DP?"

```
✅ DP হবে যদি:
  □ সর্বোচ্চ/সর্বনিম্ন বের করতে হয়
  □ কতভাবে করা যায় গুনতে হয়
  □ হ্যাঁ/না সম্ভব কিনা বের করতে হয়
  □ একই সাব-প্রবলেম বারবার আসে

❌ DP হবে না যদি:
  □ সব উপাদান প্রিন্ট করতে হয় (backtracking)
  □ গ্রাফ ট্রাভার্সাল (BFS/DFS)
  □ সর্টিং (Divide & Conquer)
```

## ধাপ ২: State সংজ্ঞায়িত করো — "dp[?] = কী?"

```
নিজেকে জিজ্ঞেস করো:
  "dp[i] বলতে কী বোঝায়?"
  "dp[i][j] বলতে কী বোঝায়?"

উদাহরণ:
  Climbing Stairs: dp[i] = i-তম ধাপে পৌঁছানোর উপায় সংখ্যা
  Knapsack:        dp[i][w] = i আইটেম ও w ক্ষমতায় সর্বোচ্চ মূল্য
  LCS:             dp[i][j] = s1[0..i] ও s2[0..j] এর LCS দৈর্ঘ্য
```

## ধাপ ৩: Recurrence Relation লেখো — "dp[i] কোথা থেকে আসে?"

```
প্রতিটি state-এর জন্য সম্ভাব্য transition চিন্তা করো:

উদাহরণ:
  Climbing Stairs: dp[i] = dp[i-1] + dp[i-2]
  House Robber:    dp[i] = max(dp[i-1], nums[i] + dp[i-2])
  Coin Change:     dp[i] = min(dp[i-coin] + 1) for each coin
  Knapsack:        dp[i][w] = max(dp[i-1][w], val[i]+dp[i-1][w-wt[i]])
```

## ধাপ ৪: Base Case ঠিক করো — "শুরুতে কী জানি?"

```
উদাহরণ:
  Climbing Stairs: dp[0] = 1, dp[1] = 1
  Coin Change:     dp[0] = 0 (০ টাকা = ০ মুদ্রা)
  Knapsack:        dp[0][w] = 0 (কোনো আইটেম নেই)
  LCS:             dp[0][j] = 0, dp[i][0] = 0 (একটি স্ট্রিং খালি)
```

## ধাপ ৫: অপটিমাইজ করো

```
① কোড চলছে? → ভালো!
② স্পেস কমানো যায়? → 2D → 1D, 1D → O(1)
③ সময় কমানো যায়? → Binary Search, Monotonic Stack
④ প্রতিটি ধাপে সিদ্ধান্ত (decision) কী ছিল সেটা রাখতে হলে
   → parent/choice array রাখো, ব্যাকট্র্যাক করো
```

---

## 📝 দ্রুত সারাংশ

```
DP = "মনে রাখো, আবার হিসাব করো না!"

১. Top-Down  = Recursion + Memo (সহজ চিন্তা, stack overflow ঝুঁকি)
২. Bottom-Up = Loop + Table    (দ্রুত, স্পেস অপটিমাইজ সহজ)

সমস্যা দেখলেই ভাবো:
  ① এটা DP?
  ② State কী?
  ③ Recurrence কী?
  ④ Base case কী?
  ⑤ অপটিমাইজ!
```

---

## 🔗 সম্পর্কিত নোট

- [Big O বিশ্লেষণ](./big-o.md) — সময় ও স্থান জটিলতা বিস্তারিত

---

> _"যে অতীত থেকে শেখে না, সে ভবিষ্যতে একই ভুল করে — recursion-এও, জীবনেও।"_ 🧠
