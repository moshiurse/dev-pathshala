# 🌳 ট্রি ও বাইনারি ট্রি (Tree & Binary Tree)

> _"ট্রি হলো প্রকৃতির সবচেয়ে সুন্দর ডেটা স্ট্রাকচার — শিকড় থেকে পাতা পর্যন্ত প্রতিটি পথ একটি গল্প বলে।"_ 🌿

---

## 📖 সংজ্ঞা

**ট্রি** হলো একটি **নন-লিনিয়ার**, **হায়ারার্কিক্যাল** ডেটা স্ট্রাকচার যেখানে নোডগুলো **প্যারেন্ট-চাইল্ড** সম্পর্কে যুক্ত থাকে। এটি একটি **সংযুক্ত** (connected) এবং **অচক্রীয়** (acyclic) গ্রাফ।

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | ট্রি-এর সাথে তুলনা |
|---|---|
| পারিবারিক বংশতালিকা | প্রতিটি ব্যক্তি → নোড, সম্পর্ক → এজ |
| ফাইল সিস্টেম | ফোল্ডার → ইন্টারনাল নোড, ফাইল → লিফ |
| HTML DOM | `<html>` → রুট, ট্যাগ → নোড |
| কোম্পানির অর্গানোগ্রাম | CEO → রুট, ম্যানেজার → ইন্টারনাল নোড |
| বই-এর সূচিপত্র | অধ্যায় → সাব-অধ্যায় → টপিক |

---

## ১. ট্রি-এর মৌলিক ধারণা

### 🔤 পরিভাষা (Terminology)

```
            [A]          ← রুট (Root): ট্রি-এর সর্বোচ্চ নোড, কোনো প্যারেন্ট নেই
           / | \
         [B] [C] [D]     ← A-এর চাইল্ড (Children), পরস্পর সিবলিং (Siblings)
        / \      |
      [E] [F]   [G]      ← B-এর চাইল্ড হলো E, F; D-এর চাইল্ড হলো G
     /
   [H]                   ← লিফ নোড (Leaf): কোনো চাইল্ড নেই (H, F, C, G)

   নোড (Node): প্রতিটি বৃত্ত (A, B, C...)
   এজ (Edge): নোডের মধ্যে সংযোগ রেখা
   প্যারেন্ট (Parent): B-এর প্যারেন্ট হলো A
   চাইল্ড (Child): A-এর চাইল্ড হলো B, C, D
   সিবলিং (Sibling): B, C, D পরস্পর সিবলিং (একই প্যারেন্ট)
```

### 📏 পরিমাপ সংক্রান্ত পরিভাষা

```
   Height, Depth, Level ব্যাখ্যা:

            [A]          Level 0  |  Depth 0  |  Height 3
           / | \
         [B] [C] [D]     Level 1  |  Depth 1  |  B: Height 2, C: Height 0, D: Height 1
        / \      |
      [E] [F]   [G]      Level 2  |  Depth 2  |  E: Height 1, F: Height 0, G: Height 0
     /
   [H]                   Level 3  |  Depth 3  |  H: Height 0

   ─────────────────────────────────────────────
   পরিভাষা         │ সংজ্ঞা
   ─────────────────────────────────────────────
   Height (উচ্চতা) │ নোড থেকে সবচেয়ে দূরবর্তী লিফ পর্যন্ত
                    │ দীর্ঘতম পথের এজ সংখ্যা
   Depth (গভীরতা)  │ রুট থেকে ঐ নোড পর্যন্ত এজ সংখ্যা
   Level (স্তর)    │ Depth-এর সমান (Level = Depth)
   Degree (মাত্রা) │ একটি নোডের চাইল্ড সংখ্যা
                    │ A-এর degree = 3, C-এর degree = 0
   Subtree (উপ-ট্রি)│ যেকোনো নোড ও তার সব বংশধর নিয়ে
                    │ গঠিত ট্রি
   ─────────────────────────────────────────────
```

### 🏠 বাস্তব উদাহরণ: ফাইল সিস্টেম

```
   ফাইল সিস্টেম ট্রি:

        [/]                      ← রুট ডিরেক্টরি
       / | \
    [home] [etc] [var]           ← সাব-ডিরেক্টরি
    /    \        |
  [user]  [docs] [log]
  /  \      |
[.bashrc] [pic] [app.log]       ← ফাইল (লিফ নোড)
            |
         [cat.jpg]
```

### 🌐 DOM Tree উদাহরণ

```
         [html]
        /      \
     [head]   [body]
      |       /    \
    [title] [div]  [footer]
            / \       |
          [h1] [p]   [a]
```

---

## ২. বাইনারি ট্রি (Binary Tree)

### 📖 সংজ্ঞা

**বাইনারি ট্রি** এমন একটি ট্রি যেখানে প্রতিটি নোডের **সর্বোচ্চ ২টি চাইল্ড** থাকে — **বাম চাইল্ড** (left) ও **ডান চাইল্ড** (right)।

### বাইনারি ট্রি-এর ধরন

#### ক) Full Binary Tree (পূর্ণ বাইনারি ট্রি)

```
   প্রতিটি নোডে হয় ০ অথবা ২টি চাইল্ড থাকে:

          [1]
         /   \
       [2]   [3]
      /   \
    [4]   [5]

   ✅ নোড 1: ২টি চাইল্ড
   ✅ নোড 2: ২টি চাইল্ড
   ✅ নোড 3: ০টি চাইল্ড (লিফ)
   ✅ নোড 4, 5: ০টি চাইল্ড (লিফ)
```

#### খ) Complete Binary Tree (সম্পূর্ণ বাইনারি ট্রি)

```
   সব লেভেল পূর্ণ, শেষ লেভেলে বাম থেকে ডানে ভর্তি:

          [1]
         /   \
       [2]   [3]
      /   \   /
    [4]  [5] [6]

   ✅ লেভেল ০, ১: পূর্ণ
   ✅ লেভেল ২: বাম থেকে ডানে ভর্তি (4, 5, 6)
```

#### গ) Perfect Binary Tree (নিখুঁত বাইনারি ট্রি)

```
   সব ইন্টারনাল নোডে ২টি চাইল্ড + সব লিফ একই লেভেলে:

          [1]
         /   \
       [2]   [3]
      /  \   / \
    [4] [5] [6] [7]

   ✅ সব ইন্টারনাল নোডে ২টি চাইল্ড
   ✅ সব লিফ (4,5,6,7) একই লেভেলে
   নোড সংখ্যা = 2^(h+1) - 1 = 2^3 - 1 = 7
```

#### ঘ) Balanced Binary Tree (সুষম বাইনারি ট্রি)

```
   প্রতিটি নোডে বাম ও ডান সাবট্রি-এর উচ্চতার পার্থক্য ≤ 1:

          [1]               [1]
         /   \             /
       [2]   [3]         [2]          ← ব্যালান্সড নয়!
      /                 /             (পার্থক্য = 2)
    [4]               [3]

   ✅ ব্যালান্সড            ❌ আনব্যালান্সড
```

#### ঙ) Degenerate (Skewed) Tree (অধঃপতিত ট্রি)

```
   প্রতিটি নোডে শুধু ১টি চাইল্ড — মূলত লিংকড লিস্ট:

   বামে স্কিউড:        ডানে স্কিউড:
      [1]                [1]
      /                    \
    [2]                    [2]
    /                        \
  [3]                        [3]
  /                            \
[4]                            [4]

   ⚠️ এই ধরনের ট্রি-তে সব অপারেশন O(n)!
```

### 📊 বাইনারি ট্রি-এর বৈশিষ্ট্য (Properties)

```
   ─────────────────────────────────────────────────────────
   বৈশিষ্ট্য                         │ সূত্র
   ─────────────────────────────────────────────────────────
   লেভেল i-তে সর্বোচ্চ নোড         │ 2^i
   উচ্চতা h-এর ট্রি-তে সর্বোচ্চ নোড │ 2^(h+1) - 1
   n নোডের ট্রি-তে এজ সংখ্যা       │ n - 1
   n লিফের Full BT-তে ইন্টারনাল নোড │ n - 1
   ন্যূনতম উচ্চতা (n নোড)           │ ⌊log₂(n)⌋
   সর্বোচ্চ উচ্চতা (n নোড)          │ n - 1
   ─────────────────────────────────────────────────────────
```

### 💻 নোড স্ট্রাকচার ও ইমপ্লিমেন্টেশন

**PHP:**

```php
<?php
// বাইনারি ট্রি নোড ক্লাস
class TreeNode {
    public $val;       // নোডের মান
    public $left;      // বাম চাইল্ড
    public $right;     // ডান চাইল্ড

    public function __construct($val) {
        $this->val = $val;
        $this->left = null;
        $this->right = null;
    }
}

// ট্রি তৈরি করা
$root = new TreeNode(1);
$root->left = new TreeNode(2);
$root->right = new TreeNode(3);
$root->left->left = new TreeNode(4);
$root->left->right = new TreeNode(5);

/*
   তৈরি হওয়া ট্রি:
        [1]
       /   \
     [2]   [3]
    /   \
  [4]   [5]
*/
?>
```

**JavaScript:**

```javascript
// বাইনারি ট্রি নোড ক্লাস
class TreeNode {
    constructor(val) {
        this.val = val;     // নোডের মান
        this.left = null;   // বাম চাইল্ড
        this.right = null;  // ডান চাইল্ড
    }
}

// ট্রি তৈরি করা
const root = new TreeNode(1);
root.left = new TreeNode(2);
root.right = new TreeNode(3);
root.left.left = new TreeNode(4);
root.left.right = new TreeNode(5);

/*
   তৈরি হওয়া ট্রি:
        [1]
       /   \
     [2]   [3]
    /   \
  [4]   [5]
*/
```

---

## ৩. ট্রি ট্রাভার্সাল (Tree Traversal)

### উদাহরণ ট্রি (সব ট্রাভার্সালে এটি ব্যবহৃত হবে):

```
            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]
     /
   [7]
```

---

### ক) Inorder ট্রাভার্সাল (Left → Root → Right)

```
   ধাপে ধাপে ট্রেস:

            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]
     /
   [7]

   ধাপ ১: বামে যাও → [1] → [2] → [4] → [7]
   ধাপ ২: [7] লিফ, প্রিন্ট: 7
   ধাপ ৩: উপরে [4], প্রিন্ট: 4
   ধাপ ৪: উপরে [2], প্রিন্ট: 2
   ধাপ ৫: [2]-এর ডানে [5], প্রিন্ট: 5
   ধাপ ৬: উপরে [1], প্রিন্ট: 1
   ধাপ ৭: [1]-এর ডানে [3], বামে কিছু নেই, প্রিন্ট: 3
   ধাপ ৮: [3]-এর ডানে [6], প্রিন্ট: 6

   ফলাফল: 7 → 4 → 2 → 5 → 1 → 3 → 6
```

**PHP — Recursive ও Iterative:**

```php
<?php
// ইনঅর্ডার — রিকার্সিভ পদ্ধতি
function inorderRecursive($node, &$result = []) {
    if ($node === null) return $result;

    inorderRecursive($node->left, $result);   // বামে যাও
    $result[] = $node->val;                   // রুট প্রিন্ট
    inorderRecursive($node->right, $result);  // ডানে যাও

    return $result;
}

// ইনঅর্ডার — ইটারেটিভ পদ্ধতি (স্ট্যাক ব্যবহার)
function inorderIterative($root) {
    $result = [];
    $stack = [];
    $current = $root;

    while ($current !== null || !empty($stack)) {
        // যতদূর সম্ভব বামে যাও
        while ($current !== null) {
            $stack[] = $current;
            $current = $current->left;
        }

        // স্ট্যাক থেকে পপ করো ও প্রসেস করো
        $current = array_pop($stack);
        $result[] = $current->val;

        // ডান সাবট্রি-তে যাও
        $current = $current->right;
    }

    return $result;
}

echo implode(' → ', inorderRecursive($root));
// ফলাফল: 7 → 4 → 2 → 5 → 1 → 3 → 6
?>
```

**JavaScript — Recursive ও Iterative:**

```javascript
// ইনঅর্ডার — রিকার্সিভ পদ্ধতি
function inorderRecursive(node, result = []) {
    if (node === null) return result;

    inorderRecursive(node.left, result);   // বামে যাও
    result.push(node.val);                 // রুট প্রিন্ট
    inorderRecursive(node.right, result);  // ডানে যাও

    return result;
}

// ইনঅর্ডার — ইটারেটিভ পদ্ধতি (স্ট্যাক ব্যবহার)
function inorderIterative(root) {
    const result = [];
    const stack = [];
    let current = root;

    while (current !== null || stack.length > 0) {
        // যতদূর সম্ভব বামে যাও
        while (current !== null) {
            stack.push(current);
            current = current.left;
        }

        // স্ট্যাক থেকে পপ করো ও প্রসেস করো
        current = stack.pop();
        result.push(current.val);

        // ডান সাবট্রি-তে যাও
        current = current.right;
    }

    return result;
}

console.log(inorderRecursive(root).join(' → '));
// ফলাফল: 7 → 4 → 2 → 5 → 1 → 3 → 6
```

---

### খ) Preorder ট্রাভার্সাল (Root → Left → Right)

```
   ধাপে ধাপে ট্রেস:

            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]
     /
   [7]

   ধাপ ১: রুট [1] প্রিন্ট: 1
   ধাপ ২: বামে [2], প্রিন্ট: 2
   ধাপ ৩: বামে [4], প্রিন্ট: 4
   ধাপ ৪: বামে [7], প্রিন্ট: 7
   ধাপ ৫: [7]-এর চাইল্ড নেই, ব্যাকট্র্যাক
   ধাপ ৬: [4]-এর ডানে নেই, [2]-এর ডানে [5], প্রিন্ট: 5
   ধাপ ৭: [1]-এর ডানে [3], প্রিন্ট: 3
   ধাপ ৮: [3]-এর ডানে [6], প্রিন্ট: 6

   ফলাফল: 1 → 2 → 4 → 7 → 5 → 3 → 6
```

**PHP:**

```php
<?php
// প্রিঅর্ডার — রিকার্সিভ পদ্ধতি
function preorderRecursive($node, &$result = []) {
    if ($node === null) return $result;

    $result[] = $node->val;                    // রুট প্রথমে
    preorderRecursive($node->left, $result);   // তারপর বামে
    preorderRecursive($node->right, $result);  // তারপর ডানে

    return $result;
}

// প্রিঅর্ডার — ইটারেটিভ পদ্ধতি
function preorderIterative($root) {
    if ($root === null) return [];

    $result = [];
    $stack = [$root];

    while (!empty($stack)) {
        $node = array_pop($stack);
        $result[] = $node->val;  // রুট প্রসেস করো

        // ডান আগে পুশ করো (যাতে বাম আগে পপ হয়)
        if ($node->right !== null) $stack[] = $node->right;
        if ($node->left !== null) $stack[] = $node->left;
    }

    return $result;
}

echo implode(' → ', preorderRecursive($root));
// ফলাফল: 1 → 2 → 4 → 7 → 5 → 3 → 6
?>
```

**JavaScript:**

```javascript
// প্রিঅর্ডার — রিকার্সিভ পদ্ধতি
function preorderRecursive(node, result = []) {
    if (node === null) return result;

    result.push(node.val);                    // রুট প্রথমে
    preorderRecursive(node.left, result);     // তারপর বামে
    preorderRecursive(node.right, result);    // তারপর ডানে

    return result;
}

// প্রিঅর্ডার — ইটারেটিভ পদ্ধতি
function preorderIterative(root) {
    if (root === null) return [];

    const result = [];
    const stack = [root];

    while (stack.length > 0) {
        const node = stack.pop();
        result.push(node.val);  // রুট প্রসেস করো

        // ডান আগে পুশ করো (যাতে বাম আগে পপ হয়)
        if (node.right !== null) stack.push(node.right);
        if (node.left !== null) stack.push(node.left);
    }

    return result;
}

console.log(preorderRecursive(root).join(' → '));
// ফলাফল: 1 → 2 → 4 → 7 → 5 → 3 → 6
```

---

### গ) Postorder ট্রাভার্সাল (Left → Right → Root)

```
   ধাপে ধাপে ট্রেস:

            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]
     /
   [7]

   ধাপ ১: বামে বামে বামে → [7] লিফ, প্রিন্ট: 7
   ধাপ ২: [4]-এর ডানে নেই, প্রিন্ট: 4
   ধাপ ৩: [2]-এর ডানে [5], লিফ, প্রিন্ট: 5
   ধাপ ৪: [2]-এর দুই চাইল্ড শেষ, প্রিন্ট: 2
   ধাপ ৫: [1]-এর ডানে [3], বামে নেই, ডানে [6]
   ধাপ ৬: [6] লিফ, প্রিন্ট: 6
   ধাপ ৭: [3] প্রিন্ট: 3
   ধাপ ৮: রুট [1] সবার শেষে, প্রিন্ট: 1

   ফলাফল: 7 → 4 → 5 → 2 → 6 → 3 → 1
```

**PHP:**

```php
<?php
// পোস্টঅর্ডার — রিকার্সিভ পদ্ধতি
function postorderRecursive($node, &$result = []) {
    if ($node === null) return $result;

    postorderRecursive($node->left, $result);   // বামে যাও
    postorderRecursive($node->right, $result);  // ডানে যাও
    $result[] = $node->val;                     // রুট সবার শেষে

    return $result;
}

// পোস্টঅর্ডার — ইটারেটিভ পদ্ধতি (দুটি স্ট্যাক)
function postorderIterative($root) {
    if ($root === null) return [];

    $result = [];
    $stack1 = [$root];
    $stack2 = [];

    while (!empty($stack1)) {
        $node = array_pop($stack1);
        $stack2[] = $node;  // দ্বিতীয় স্ট্যাকে রাখো

        if ($node->left !== null) $stack1[] = $node->left;
        if ($node->right !== null) $stack1[] = $node->right;
    }

    // দ্বিতীয় স্ট্যাক উল্টো করে পড়লেই পোস্টঅর্ডার
    while (!empty($stack2)) {
        $node = array_pop($stack2);
        $result[] = $node->val;
    }

    return $result;
}

echo implode(' → ', postorderRecursive($root));
// ফলাফল: 7 → 4 → 5 → 2 → 6 → 3 → 1
?>
```

**JavaScript:**

```javascript
// পোস্টঅর্ডার — রিকার্সিভ পদ্ধতি
function postorderRecursive(node, result = []) {
    if (node === null) return result;

    postorderRecursive(node.left, result);   // বামে যাও
    postorderRecursive(node.right, result);  // ডানে যাও
    result.push(node.val);                   // রুট সবার শেষে

    return result;
}

// পোস্টঅর্ডার — ইটারেটিভ পদ্ধতি (দুটি স্ট্যাক)
function postorderIterative(root) {
    if (root === null) return [];

    const result = [];
    const stack1 = [root];
    const stack2 = [];

    while (stack1.length > 0) {
        const node = stack1.pop();
        stack2.push(node);  // দ্বিতীয় স্ট্যাকে রাখো

        if (node.left !== null) stack1.push(node.left);
        if (node.right !== null) stack1.push(node.right);
    }

    // দ্বিতীয় স্ট্যাক উল্টো করে পড়লেই পোস্টঅর্ডার
    while (stack2.length > 0) {
        result.push(stack2.pop().val);
    }

    return result;
}

console.log(postorderRecursive(root).join(' → '));
// ফলাফল: 7 → 4 → 5 → 2 → 6 → 3 → 1
```

---

### ঘ) Level Order ট্রাভার্সাল (BFS — Breadth First Search)

```
   ধাপে ধাপে ট্রেস (কিউ ব্যবহার):

            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]
     /
   [7]

   কিউ: [1]           → ডিকিউ 1, এনকিউ 2, 3
   কিউ: [2, 3]        → ডিকিউ 2, এনকিউ 4, 5
   কিউ: [3, 4, 5]     → ডিকিউ 3, এনকিউ 6
   কিউ: [4, 5, 6]     → ডিকিউ 4, এনকিউ 7
   কিউ: [5, 6, 7]     → ডিকিউ 5 (লিফ)
   কিউ: [6, 7]        → ডিকিউ 6 (লিফ)
   কিউ: [7]           → ডিকিউ 7 (লিফ)
   কিউ: []            → খালি, শেষ!

   ফলাফল: 1 → 2 → 3 → 4 → 5 → 6 → 7
```

**PHP:**

```php
<?php
// লেভেল অর্ডার ট্রাভার্সাল (BFS) — কিউ ব্যবহার
function levelOrder($root) {
    if ($root === null) return [];

    $result = [];
    $queue = new SplQueue();  // কিউ তৈরি
    $queue->enqueue($root);

    while (!$queue->isEmpty()) {
        $levelSize = $queue->count();
        $currentLevel = [];

        for ($i = 0; $i < $levelSize; $i++) {
            $node = $queue->dequeue();  // সামনে থেকে বের করো
            $currentLevel[] = $node->val;

            // চাইল্ডদের কিউ-তে যোগ করো
            if ($node->left !== null) $queue->enqueue($node->left);
            if ($node->right !== null) $queue->enqueue($node->right);
        }

        $result[] = $currentLevel;  // এই লেভেলের সব নোড
    }

    return $result;
}

$levels = levelOrder($root);
foreach ($levels as $i => $level) {
    echo "লেভেল $i: " . implode(', ', $level) . "\n";
}
// লেভেল 0: 1
// লেভেল 1: 2, 3
// লেভেল 2: 4, 5, 6
// লেভেল 3: 7
?>
```

**JavaScript:**

```javascript
// লেভেল অর্ডার ট্রাভার্সাল (BFS) — কিউ ব্যবহার
function levelOrder(root) {
    if (root === null) return [];

    const result = [];
    const queue = [root];  // কিউ হিসেবে অ্যারে ব্যবহার

    while (queue.length > 0) {
        const levelSize = queue.length;
        const currentLevel = [];

        for (let i = 0; i < levelSize; i++) {
            const node = queue.shift();  // সামনে থেকে বের করো
            currentLevel.push(node.val);

            // চাইল্ডদের কিউ-তে যোগ করো
            if (node.left !== null) queue.push(node.left);
            if (node.right !== null) queue.push(node.right);
        }

        result.push(currentLevel);  // এই লেভেলের সব নোড
    }

    return result;
}

const levels = levelOrder(root);
levels.forEach((level, i) => {
    console.log(`লেভেল ${i}: ${level.join(', ')}`);
});
// লেভেল 0: 1
// লেভেল 1: 2, 3
// লেভেল 2: 4, 5, 6
// লেভেল 3: 7
```

---

### 📊 কখন কোন ট্রাভার্সাল ব্যবহার করবেন

```
   ─────────────────────────────────────────────────────────────────────
   ট্রাভার্সাল     │ ব্যবহারের ক্ষেত্র
   ─────────────────────────────────────────────────────────────────────
   Inorder         │ BST থেকে সর্টেড ক্রম পেতে
                   │ এক্সপ্রেশন ট্রি থেকে ইনফিক্স নোটেশন
   ─────────────────────────────────────────────────────────────────────
   Preorder        │ ট্রি কপি/ক্লোন করতে
                   │ প্রিফিক্স নোটেশন, সিরিয়ালাইজেশন
   ─────────────────────────────────────────────────────────────────────
   Postorder       │ ট্রি ডিলিট করতে (চাইল্ড আগে ডিলিট)
                   │ পোস্টফিক্স নোটেশন, ডিস্ক স্পেস গণনা
   ─────────────────────────────────────────────────────────────────────
   Level Order     │ সর্বনিম্ন পথ খুঁজতে (BFS)
                   │ লেভেল-ভিত্তিক প্রসেসিং
                   │ ট্রি-র প্রস্থ বা সর্বনিম্ন গভীরতা
   ─────────────────────────────────────────────────────────────────────
```

---

## ৪. Binary Search Tree (BST)

### 📖 সংজ্ঞা

**BST** হলো এমন একটি বাইনারি ট্রি যেখানে প্রতিটি নোডের জন্য:
- **বাম সাবট্রি-র** সব মান < নোডের মান
- **ডান সাবট্রি-র** সব মান > নোডের মান

```
   বৈধ BST:               অবৈধ BST:

        [8]                    [8]
       /   \                  /   \
     [3]   [10]             [3]   [10]
    /   \     \            /   \     \
  [1]   [6]  [14]       [1]   [9]  [14]
       /  \   /                ↑
     [4]  [7] [13]       9 > 8, তাই বাম সাবট্রি-তে
                          থাকতে পারে না!
```

### 🔍 BST অপারেশন: সার্চ (Search)

```
   ১৩ খুঁজছি:

        [8]         8 < 13 → ডানে যাও
       /   \
     [3]   [10]     10 < 13 → ডানে যাও
    /   \     \
  [1]   [6]  [14]   14 > 13 → বামে যাও
       /  \   /
     [4]  [7] [13]  ✅ পাওয়া গেছে!

   তুলনা সংখ্যা: ৪ (ট্রি-র উচ্চতা)
```

### ➕ BST অপারেশন: ইনসার্ট (Insert)

```
   ৫ ইনসার্ট করছি:

        [8]         8 > 5 → বামে
       /   \
     [3]   [10]     3 < 5 → ডানে
    /   \     \
  [1]   [6]  [14]   6 > 5 → বামে
       /  \   /
     [4]  [7] [13]  4 < 5 → ডানে

     [4]
       \
       [5] ← নতুন নোড এখানে বসলো!

   ইনসার্টের পর:
        [8]
       /   \
     [3]   [10]
    /   \     \
  [1]   [6]  [14]
       /  \   /
     [4]  [7] [13]
       \
       [5]
```

### ❌ BST অপারেশন: ডিলিট (Delete) — ৩টি কেস

```
   কেস ১: লিফ নোড ডিলিট (কোনো চাইল্ড নেই)
   ─────────────────────────────────────────
   [7] ডিলিট করো:

        [8]                [8]
       /   \              /   \
     [3]   [10]    →    [3]   [10]
    /   \     \        /   \     \
  [1]   [6]  [14]   [1]   [6]  [14]
       /  \                /
     [4]  [7]  ← ❌     [4]

   শুধু নোড মুছে ফেলো!

   কেস ২: একটি চাইল্ড আছে
   ─────────────────────────────────────────
   [10] ডিলিট করো (শুধু ডান চাইল্ড [14] আছে):

        [8]                [8]
       /   \              /   \
     [3]   [10] ← ❌    [3]   [14]
    /   \     \        /   \   /
  [1]   [6]  [14]   [1]   [6] [13]
       /  \   /           /  \
     [4]  [7] [13]      [4]  [7]

   চাইল্ডকে প্যারেন্টের জায়গায় বসাও!

   কেস ৩: দুটি চাইল্ড আছে
   ─────────────────────────────────────────
   [3] ডিলিট করো:

        [8]                  [8]
       /   \                /   \
     [3] ← ❌  [10]  →   [4]   [10]
    /   \        \       /   \     \
  [1]   [6]    [14]   [1]   [6]  [14]
       /  \    /            /  \   /
     [4]  [7] [13]        [5]  [7] [13]
       \
       [5]

   ইনঅর্ডার সাক্সেসর (ডান সাবট্রি-র সবচেয়ে ছোট)
   অথবা ইনঅর্ডার প্রিডেসেসর (বাম সাবট্রি-র সবচেয়ে বড়)
   দিয়ে প্রতিস্থাপন করো!
   এখানে সাক্সেসর = 4, তাই 3 → 4
```

### 📊 BST কমপ্লেক্সিটি

```
   ─────────────────────────────────────────────
   অপারেশন    │ গড় (Average)  │ সবচেয়ে খারাপ (Worst)
   ─────────────────────────────────────────────
   সার্চ      │ O(log n)       │ O(n)
   ইনসার্ট    │ O(log n)       │ O(n)
   ডিলিট      │ O(log n)       │ O(n)
   ─────────────────────────────────────────────

   কেন Worst Case O(n)?

   ক্রমানুসারে ইনসার্ট করলে (1, 2, 3, 4, 5):

   [1]
     \
     [2]
       \
       [3]        ← স্কিউড ট্রি!
         \           মূলত লিংকড লিস্ট
         [4]         তাই O(n)
           \
           [5]
```

### 💻 সম্পূর্ণ BST ইমপ্লিমেন্টেশন

**PHP:**

```php
<?php
class BSTNode {
    public $val;
    public $left;
    public $right;

    public function __construct($val) {
        $this->val = $val;
        $this->left = null;
        $this->right = null;
    }
}

class BST {
    private $root;

    public function __construct() {
        $this->root = null;
    }

    // ইনসার্ট অপারেশন
    public function insert($val) {
        $this->root = $this->_insert($this->root, $val);
    }

    private function _insert($node, $val) {
        // খালি জায়গায় নতুন নোড বসাও
        if ($node === null) return new BSTNode($val);

        if ($val < $node->val) {
            $node->left = $this->_insert($node->left, $val);  // বামে যাও
        } elseif ($val > $node->val) {
            $node->right = $this->_insert($node->right, $val); // ডানে যাও
        }
        // সমান হলে কিছু করো না (ডুপ্লিকেট অনুমোদিত নয়)

        return $node;
    }

    // সার্চ অপারেশন
    public function search($val) {
        return $this->_search($this->root, $val);
    }

    private function _search($node, $val) {
        if ($node === null) return false;     // পাওয়া যায়নি
        if ($val === $node->val) return true;  // পাওয়া গেছে!

        if ($val < $node->val) {
            return $this->_search($node->left, $val);   // বামে খোঁজো
        } else {
            return $this->_search($node->right, $val);  // ডানে খোঁজো
        }
    }

    // সবচেয়ে ছোট মান খুঁজে বের করো
    public function findMin($node = null) {
        $current = $node ?? $this->root;
        while ($current->left !== null) {
            $current = $current->left;  // সবচেয়ে বামের নোড = সবচেয়ে ছোট
        }
        return $current->val;
    }

    // সবচেয়ে বড় মান খুঁজে বের করো
    public function findMax($node = null) {
        $current = $node ?? $this->root;
        while ($current->right !== null) {
            $current = $current->right;  // সবচেয়ে ডানের নোড = সবচেয়ে বড়
        }
        return $current->val;
    }

    // ডিলিট অপারেশন
    public function delete($val) {
        $this->root = $this->_delete($this->root, $val);
    }

    private function _delete($node, $val) {
        if ($node === null) return null;

        if ($val < $node->val) {
            $node->left = $this->_delete($node->left, $val);
        } elseif ($val > $node->val) {
            $node->right = $this->_delete($node->right, $val);
        } else {
            // নোড পাওয়া গেছে — ডিলিট করো

            // কেস ১: লিফ নোড বা একটি চাইল্ড
            if ($node->left === null) return $node->right;
            if ($node->right === null) return $node->left;

            // কেস ৩: দুটি চাইল্ড — ইনঅর্ডার সাক্সেসর দিয়ে বদলাও
            $minRight = $this->findMin($node->right);
            $node->val = $minRight;
            $node->right = $this->_delete($node->right, $minRight);
        }

        return $node;
    }

    // ইনঅর্ডার ট্রাভার্সাল — সর্টেড আউটপুট
    public function inorder($node = null, &$result = []) {
        $node = $node ?? $this->root;
        if ($node === null) return $result;

        $this->inorder($node->left, $result);
        $result[] = $node->val;
        $this->inorder($node->right, $result);

        return $result;
    }

    // BST ভ্যালিডেশন
    public function isValidBST($node = null, $min = PHP_INT_MIN, $max = PHP_INT_MAX) {
        if ($node === null) {
            $node = $this->root;
            if ($node === null) return true;
        }

        if ($node->val <= $min || $node->val >= $max) return false;

        return $this->isValidBST($node->left, $min, $node->val)
            && $this->isValidBST($node->right, $node->val, $max);
    }

    public function getRoot() {
        return $this->root;
    }
}

// ব্যবহার
$bst = new BST();
foreach ([8, 3, 10, 1, 6, 14, 4, 7, 13] as $val) {
    $bst->insert($val);
}

echo "ইনঅর্ডার (সর্টেড): " . implode(', ', $bst->inorder()) . "\n";
// ফলাফল: 1, 3, 4, 6, 7, 8, 10, 13, 14

echo "সার্চ 6: " . ($bst->search(6) ? "পাওয়া গেছে" : "পাওয়া যায়নি") . "\n";
echo "ন্যূনতম: " . $bst->findMin() . "\n";  // 1
echo "সর্বোচ্চ: " . $bst->findMax() . "\n";  // 14

$bst->delete(3);
echo "৩ ডিলিটের পর: " . implode(', ', $bst->inorder()) . "\n";
// ফলাফল: 1, 4, 6, 7, 8, 10, 13, 14
?>
```

**JavaScript:**

```javascript
class BSTNode {
    constructor(val) {
        this.val = val;
        this.left = null;
        this.right = null;
    }
}

class BST {
    constructor() {
        this.root = null;
    }

    // ইনসার্ট অপারেশন
    insert(val) {
        this.root = this._insert(this.root, val);
    }

    _insert(node, val) {
        if (node === null) return new BSTNode(val);

        if (val < node.val) {
            node.left = this._insert(node.left, val);   // বামে যাও
        } else if (val > node.val) {
            node.right = this._insert(node.right, val);  // ডানে যাও
        }

        return node;
    }

    // সার্চ অপারেশন
    search(val) {
        return this._search(this.root, val);
    }

    _search(node, val) {
        if (node === null) return false;
        if (val === node.val) return true;

        if (val < node.val) return this._search(node.left, val);
        return this._search(node.right, val);
    }

    // সবচেয়ে ছোট মান
    findMin(node = this.root) {
        let current = node;
        while (current.left !== null) {
            current = current.left;
        }
        return current.val;
    }

    // সবচেয়ে বড় মান
    findMax(node = this.root) {
        let current = node;
        while (current.right !== null) {
            current = current.right;
        }
        return current.val;
    }

    // ডিলিট অপারেশন
    delete(val) {
        this.root = this._delete(this.root, val);
    }

    _delete(node, val) {
        if (node === null) return null;

        if (val < node.val) {
            node.left = this._delete(node.left, val);
        } else if (val > node.val) {
            node.right = this._delete(node.right, val);
        } else {
            // কেস ১: লিফ বা একটি চাইল্ড
            if (node.left === null) return node.right;
            if (node.right === null) return node.left;

            // কেস ৩: দুটি চাইল্ড
            const minRight = this.findMin(node.right);
            node.val = minRight;
            node.right = this._delete(node.right, minRight);
        }

        return node;
    }

    // ইনঅর্ডার (সর্টেড আউটপুট)
    inorder(node = this.root, result = []) {
        if (node === null) return result;

        this.inorder(node.left, result);
        result.push(node.val);
        this.inorder(node.right, result);

        return result;
    }

    // BST ভ্যালিডেশন
    isValidBST(node = this.root, min = -Infinity, max = Infinity) {
        if (node === null) return true;
        if (node.val <= min || node.val >= max) return false;

        return this.isValidBST(node.left, min, node.val)
            && this.isValidBST(node.right, node.val, max);
    }
}

// ব্যবহার
const bst = new BST();
[8, 3, 10, 1, 6, 14, 4, 7, 13].forEach(val => bst.insert(val));

console.log("ইনঅর্ডার (সর্টেড):", bst.inorder().join(", "));
// ফলাফল: 1, 3, 4, 6, 7, 8, 10, 13, 14

console.log("সার্চ 6:", bst.search(6) ? "পাওয়া গেছে" : "পাওয়া যায়নি");
console.log("ন্যূনতম:", bst.findMin());   // 1
console.log("সর্বোচ্চ:", bst.findMax());   // 14

bst.delete(3);
console.log("৩ ডিলিটের পর:", bst.inorder().join(", "));
// ফলাফল: 1, 4, 6, 7, 8, 10, 13, 14
```

---

## ৫. Self-Balancing BST

### কেন ব্যালান্সিং দরকার?

```
   ক্রমানুসারে ইনসার্ট: 1, 2, 3, 4, 5

   BST:                    AVL:
   [1]                       [2]
     \                      /   \
     [2]                  [1]   [4]
       \                       /   \
       [3]     ← O(n)       [3]   [5]   ← O(log n)
         \
         [4]
           \
           [5]

   BST-তে সার্চ: O(n) — লিংকড লিস্টের মতো!
   AVL-এ সার্চ: O(log n) — সবসময় ব্যালান্সড!
```

---

### ক) AVL Tree (অ্যাডেলসন-ভেলস্কি ও ল্যান্ডিস)

#### ব্যালান্স ফ্যাক্টর (Balance Factor)

```
   BF(node) = height(বাম সাবট্রি) - height(ডান সাবট্রি)

   ব্যালান্সড: BF ∈ {-1, 0, 1}
   আনব্যালান্সড: |BF| > 1 → রোটেশন দরকার!

        [30] BF=2         [20] BF=0
       /          →       /   \
     [20] BF=1          [10]  [30]
    /
  [10] BF=0
```

#### ৪ ধরনের রোটেশন

```
   ১) LL রোটেশন (Left-Left) — রাইট রোটেট করো
   ─────────────────────────────────────────

   সমস্যা: বাম-বাম দিকে ভারী

        [30]                 [20]
       /                    /    \
     [20]         →       [10]   [30]
    /
  [10]

   ধাপ: 30-এর বাম চাইল্ড (20) কে রুট বানাও
         30 কে 20-এর ডান চাইল্ড বানাও


   ২) RR রোটেশন (Right-Right) — লেফট রোটেট করো
   ─────────────────────────────────────────

   সমস্যা: ডান-ডান দিকে ভারী

   [10]                     [20]
     \                     /    \
     [20]        →       [10]   [30]
       \
       [30]

   ধাপ: 10-এর ডান চাইল্ড (20) কে রুট বানাও
         10 কে 20-এর বাম চাইল্ড বানাও


   ৩) LR রোটেশন (Left-Right) — প্রথমে লেফট, তারপর রাইট
   ─────────────────────────────────────────

   সমস্যা: বাম চাইল্ডের ডানে ভারী

      [30]            [30]            [20]
     /               /               /    \
   [10]      →     [20]      →    [10]   [30]
     \             /
     [20]        [10]

   ধাপ ১: বাম চাইল্ড (10)-এ লেফট রোটেট
   ধাপ ২: রুট (30)-এ রাইট রোটেট


   ৪) RL রোটেশন (Right-Left) — প্রথমে রাইট, তারপর লেফট
   ─────────────────────────────────────────

   সমস্যা: ডান চাইল্ডের বামে ভারী

   [10]           [10]               [20]
     \              \               /    \
     [30]    →     [20]     →    [10]   [30]
    /                 \
  [20]               [30]

   ধাপ ১: ডান চাইল্ড (30)-এ রাইট রোটেট
   ধাপ ২: রুট (10)-এ লেফট রোটেট
```

#### AVL ইনসার্ট উদাহরণ ট্রেস

```
   ক্রমে ইনসার্ট করি: 30, 20, 10, 25, 28

   ধাপ ১: Insert 30       [30]

   ধাপ ২: Insert 20       [30]
                          /
                        [20]

   ধাপ ৩: Insert 10       [30] BF=2 → LL রোটেশন!
                          /
                        [20]
                       /
                     [10]

   রোটেশনের পর:        [20]
                      /    \
                    [10]   [30]

   ধাপ ৪: Insert 25       [20]
                          /    \
                        [10]   [30]
                              /
                            [25]

   ধাপ ৫: Insert 28       [20]
                          /    \
                        [10]   [30] BF=2
                              /
                            [25] BF=-1 → RL!
                              \
                              [28]

   RL রোটেশন:
   প্রথমে 25-এ রাইট রোটেট, তারপর 30-এ লেফট রোটেট:

                          [20]
                         /    \
                       [10]   [28]
                             /    \
                           [25]   [30]
```

---

### খ) Red-Black Tree (লাল-কালো ট্রি) — ধারণাগত স্তর

```
   ৫টি বৈশিষ্ট্য:
   ─────────────────────────────────────────
   ১. প্রতিটি নোড হয় লাল (R) অথবা কালো (B)
   ২. রুট সবসময় কালো
   ৩. সব লিফ (NIL) কালো
   ৪. লাল নোডের চাইল্ড সবসময় কালো
      (পরপর দুটি লাল নোড হবে না)
   ৫. যেকোনো নোড থেকে লিফ পর্যন্ত সব পথে
      সমান সংখ্যক কালো নোড

   উদাহরণ:
          [13B]
         /     \
      [8R]     [17R]
      /  \      /   \
   [1B] [11B] [15B] [25B]
     \                /
    [6R]           [22R]

   B = কালো, R = লাল

   AVL vs Red-Black:
   ─────────────────────────────────────────
   │ বৈশিষ্ট্য    │ AVL        │ Red-Black │
   ─────────────────────────────────────────
   │ ব্যালান্সিং  │ কঠোর      │ শিথিল     │
   │ সার্চ        │ দ্রুততর   │ ধীরতর     │
   │ ইনসার্ট/ডিলিট│ বেশি রোটেশন│ কম রোটেশন │
   │ ব্যবহার     │ DB ইনডেক্স │ Map, Set  │
   ─────────────────────────────────────────
```

---

## ৬. Heap (হিপ)

### 📖 সংজ্ঞা

**হিপ** হলো একটি **Complete Binary Tree** যেখানে প্রতিটি নোড একটি নির্দিষ্ট ক্রম বজায় রাখে।

```
   Max-Heap:                    Min-Heap:
   (প্যারেন্ট ≥ চাইল্ড)      (প্যারেন্ট ≤ চাইল্ড)

        [90]                       [5]
       /    \                     /   \
     [80]   [70]               [10]  [15]
    /   \   /                 /   \   /
  [50] [60][40]             [20] [30][25]

   ✅ রুটে সর্বোচ্চ মান       ✅ রুটে সর্বনিম্ন মান
```

### 📊 অ্যারে রিপ্রেজেন্টেশন

```
   Max-Heap:
        [90]
       /    \
     [80]   [70]
    /   \   /
  [50] [60][40]

   অ্যারে (0-indexed):
   ইনডেক্স: [0]  [1]  [2]  [3]  [4]  [5]
   মান:     90   80   70   50   60   40

   ─────────────────────────────────────────
   সম্পর্ক             │ সূত্র (0-indexed)
   ─────────────────────────────────────────
   প্যারেন্ট ইনডেক্স   │ (i - 1) / 2    (ফ্লোর)
   বাম চাইল্ড ইনডেক্স  │ 2 * i + 1
   ডান চাইল্ড ইনডেক্স  │ 2 * i + 2
   ─────────────────────────────────────────

   উদাহরণ: ইনডেক্স 1 (মান 80)
   প্যারেন্ট: (1-1)/2 = 0 (মান 90) ✅
   বাম চাইল্ড: 2*1+1 = 3 (মান 50) ✅
   ডান চাইল্ড: 2*1+2 = 4 (মান 60) ✅
```

### ➕ হিপ অপারেশন: Insert (Heapify Up / Bubble Up)

```
   Max-Heap-এ 85 ইনসার্ট:

   ধাপ ১: শেষে যোগ করো
        [90]
       /    \
     [80]   [70]
    /   \   /  \
  [50] [60][40] [85] ← নতুন

   ধাপ ২: প্যারেন্টের (70) সাথে তুলনা: 85 > 70 → অদলবদল
        [90]
       /    \
     [80]   [85] ← উপরে উঠলো
    /   \   /  \
  [50] [60][40] [70]

   ধাপ ৩: প্যারেন্টের (90) সাথে তুলনা: 85 < 90 → থামো!
   চূড়ান্ত:
        [90]
       /    \
     [80]   [85]
    /   \   /  \
  [50] [60][40] [70]
```

### ❌ হিপ অপারেশন: Delete Root (Heapify Down / Bubble Down)

```
   Max-Heap থেকে রুট (90) ডিলিট:

   ধাপ ১: রুট ↔ শেষ নোড অদলবদল, শেষ নোড মুছো
        [40]             (90 মুছে গেছে)
       /    \
     [80]   [85]
    /   \   /
  [50] [60][70]

   ধাপ ২: চাইল্ডদের মধ্যে বড়টির (85) সাথে তুলনা
          40 < 85 → অদলবদল
        [85]
       /    \
     [80]   [40] ← নিচে নামলো
    /   \   /
  [50] [60][70]

   ধাপ ৩: চাইল্ড (70) এর সাথে তুলনা: 40 < 70 → অদলবদল
        [85]
       /    \
     [80]   [70]
    /   \   /
  [50] [60][40] ← সঠিক জায়গায়!
```

### 🏗️ Build Heap

```
   অ্যারে: [4, 10, 3, 5, 1]

   প্রাথমিক ট্রি (অ্যারে থেকে):
        [4]
       /   \
     [10]  [3]
    /   \
  [5]   [1]

   শেষ নন-লিফ নোড থেকে শুরু: ইনডেক্স (n/2 - 1) = 1

   ধাপ ১: heapify(1) → [10] vs [5],[1] → ঠিক আছে
   ধাপ ২: heapify(0) → [4] vs [10],[3] → 4 < 10 → অদলবদল
        [10]
       /   \
     [4]   [3]   → [4] vs [5],[1] → 4 < 5 → অদলবদল
    /   \
  [5]   [1]

   চূড়ান্ত Max-Heap:
        [10]
       /    \
      [5]   [3]
     /  \
   [4]  [1]

   ⏱️ Build Heap কমপ্লেক্সিটি: O(n) — O(n log n) নয়!
```

### 📊 Heap Sort অ্যালগরিদম

```
   ধাপ ১: Max-Heap তৈরি করো
   ধাপ ২: রুট (সর্বোচ্চ) কে শেষে পাঠাও
   ধাপ ৩: হিপ সাইজ কমাও, heapify করো
   ধাপ ৪: ২-৩ রিপিট

   অ্যারে: [4, 10, 3, 5, 1]

   Max-Heap:    [10, 5, 3, 4, 1]
   ← 10 শেষে →  [1, 5, 3, 4, | 10]    heapify → [5, 4, 3, 1, | 10]
   ← 5 শেষে →   [1, 4, 3, | 5, 10]    heapify → [4, 1, 3, | 5, 10]
   ← 4 শেষে →   [3, 1, | 4, 5, 10]    heapify → [3, 1, | 4, 5, 10]
   ← 3 শেষে →   [1, | 3, 4, 5, 10]

   সর্টেড: [1, 3, 4, 5, 10] ✅

   ⏱️ Heap Sort: O(n log n) — সবসময়!
   📦 স্পেস: O(1) — in-place সর্ট!
```

### 💻 হিপ ও প্রায়োরিটি কিউ ইমপ্লিমেন্টেশন

**PHP:**

```php
<?php
class MaxHeap {
    private $heap = [];

    // হিপের আকার
    public function size() {
        return count($this->heap);
    }

    // হিপ খালি কিনা
    public function isEmpty() {
        return empty($this->heap);
    }

    // সর্বোচ্চ মান দেখো (মুছো না)
    public function peek() {
        if ($this->isEmpty()) throw new Exception("হিপ খালি!");
        return $this->heap[0];
    }

    // ইনসার্ট — শেষে যোগ করো, তারপর উপরে তোলো
    public function insert($val) {
        $this->heap[] = $val;
        $this->heapifyUp(count($this->heap) - 1);
    }

    // রুট ডিলিট — শেষ দিয়ে বদলাও, তারপর নিচে নামাও
    public function extractMax() {
        if ($this->isEmpty()) throw new Exception("হিপ খালি!");

        $max = $this->heap[0];
        $last = array_pop($this->heap);

        if (!$this->isEmpty()) {
            $this->heap[0] = $last;
            $this->heapifyDown(0);
        }

        return $max;
    }

    // উপরে তোলো (Bubble Up)
    private function heapifyUp($index) {
        while ($index > 0) {
            $parent = intdiv($index - 1, 2);
            if ($this->heap[$index] <= $this->heap[$parent]) break;

            // প্যারেন্টের সাথে অদলবদল
            [$this->heap[$index], $this->heap[$parent]] =
                [$this->heap[$parent], $this->heap[$index]];
            $index = $parent;
        }
    }

    // নিচে নামাও (Bubble Down)
    private function heapifyDown($index) {
        $size = count($this->heap);

        while (true) {
            $largest = $index;
            $left = 2 * $index + 1;
            $right = 2 * $index + 2;

            // বাম চাইল্ড বড় হলে
            if ($left < $size && $this->heap[$left] > $this->heap[$largest]) {
                $largest = $left;
            }
            // ডান চাইল্ড বড় হলে
            if ($right < $size && $this->heap[$right] > $this->heap[$largest]) {
                $largest = $right;
            }

            if ($largest === $index) break;

            // অদলবদল
            [$this->heap[$index], $this->heap[$largest]] =
                [$this->heap[$largest], $this->heap[$index]];
            $index = $largest;
        }
    }
}

// হিপ সর্ট ফাংশন
function heapSort(&$arr) {
    $n = count($arr);

    // Max-Heap তৈরি করো
    for ($i = intdiv($n, 2) - 1; $i >= 0; $i--) {
        heapify($arr, $n, $i);
    }

    // একটি একটি করে এলিমেন্ট বের করো
    for ($i = $n - 1; $i > 0; $i--) {
        [$arr[0], $arr[$i]] = [$arr[$i], $arr[0]];
        heapify($arr, $i, 0);
    }
}

function heapify(&$arr, $n, $i) {
    $largest = $i;
    $left = 2 * $i + 1;
    $right = 2 * $i + 2;

    if ($left < $n && $arr[$left] > $arr[$largest]) $largest = $left;
    if ($right < $n && $arr[$right] > $arr[$largest]) $largest = $right;

    if ($largest !== $i) {
        [$arr[$i], $arr[$largest]] = [$arr[$largest], $arr[$i]];
        heapify($arr, $n, $largest);
    }
}

// প্রায়োরিটি কিউ ব্যবহার
$pq = new MaxHeap();
$pq->insert(30);
$pq->insert(50);
$pq->insert(10);
$pq->insert(40);

echo "সর্বোচ্চ: " . $pq->peek() . "\n";       // 50
echo "বের করো: " . $pq->extractMax() . "\n";    // 50
echo "পরবর্তী সর্বোচ্চ: " . $pq->peek() . "\n"; // 40

// হিপ সর্ট
$arr = [4, 10, 3, 5, 1];
heapSort($arr);
echo "সর্টেড: " . implode(', ', $arr) . "\n";   // 1, 3, 4, 5, 10
?>
```

**JavaScript:**

```javascript
class MaxHeap {
    constructor() {
        this.heap = [];
    }

    size() { return this.heap.length; }
    isEmpty() { return this.heap.length === 0; }

    // সর্বোচ্চ মান দেখো
    peek() {
        if (this.isEmpty()) throw new Error("হিপ খালি!");
        return this.heap[0];
    }

    // ইনসার্ট
    insert(val) {
        this.heap.push(val);
        this._heapifyUp(this.heap.length - 1);
    }

    // সর্বোচ্চ বের করো
    extractMax() {
        if (this.isEmpty()) throw new Error("হিপ খালি!");

        const max = this.heap[0];
        const last = this.heap.pop();

        if (!this.isEmpty()) {
            this.heap[0] = last;
            this._heapifyDown(0);
        }

        return max;
    }

    // উপরে তোলো
    _heapifyUp(index) {
        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (this.heap[index] <= this.heap[parent]) break;

            [this.heap[index], this.heap[parent]] =
                [this.heap[parent], this.heap[index]];
            index = parent;
        }
    }

    // নিচে নামাও
    _heapifyDown(index) {
        const size = this.heap.length;

        while (true) {
            let largest = index;
            const left = 2 * index + 1;
            const right = 2 * index + 2;

            if (left < size && this.heap[left] > this.heap[largest]) {
                largest = left;
            }
            if (right < size && this.heap[right] > this.heap[largest]) {
                largest = right;
            }

            if (largest === index) break;

            [this.heap[index], this.heap[largest]] =
                [this.heap[largest], this.heap[index]];
            index = largest;
        }
    }
}

// হিপ সর্ট
function heapSort(arr) {
    const n = arr.length;

    // Max-Heap তৈরি
    for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }

    // একটি একটি করে বের করো
    for (let i = n - 1; i > 0; i--) {
        [arr[0], arr[i]] = [arr[i], arr[0]];
        heapify(arr, i, 0);
    }

    return arr;
}

function heapify(arr, n, i) {
    let largest = i;
    const left = 2 * i + 1;
    const right = 2 * i + 2;

    if (left < n && arr[left] > arr[largest]) largest = left;
    if (right < n && arr[right] > arr[largest]) largest = right;

    if (largest !== i) {
        [arr[i], arr[largest]] = [arr[largest], arr[i]];
        heapify(arr, n, largest);
    }
}

// ব্যবহার
const pq = new MaxHeap();
pq.insert(30);
pq.insert(50);
pq.insert(10);
pq.insert(40);

console.log("সর্বোচ্চ:", pq.peek());         // 50
console.log("বের করো:", pq.extractMax());     // 50
console.log("পরবর্তী:", pq.peek());           // 40

const arr = [4, 10, 3, 5, 1];
console.log("সর্টেড:", heapSort(arr).join(", ")); // 1, 3, 4, 5, 10
```

---

## ৭. Advanced Tree Problems

### ক) Lowest Common Ancestor (LCA) — সর্বনিম্ন সাধারণ পূর্বপুরুষ

```
   LCA(4, 5) = 2
   LCA(4, 6) = 1

            [1]
           /   \
         [2]   [3]
        /   \     \
      [4]   [5]   [6]

   ব্যাখ্যা: 4 ও 5 — দুজনের সবচেয়ে কাছের সাধারণ পূর্বপুরুষ হলো 2
             4 ও 6 — একটি বামে, একটি ডানে, তাই LCA = 1 (রুট)
```

**PHP:**

```php
<?php
// LCA খুঁজে বের করো — রিকার্সিভ পদ্ধতি
function lowestCommonAncestor($root, $p, $q) {
    // বেস কেস: নোড null বা p বা q হলে নিজেকে রিটার্ন করো
    if ($root === null || $root->val === $p || $root->val === $q) {
        return $root;
    }

    // বাম ও ডান সাবট্রি-তে খোঁজো
    $left = lowestCommonAncestor($root->left, $p, $q);
    $right = lowestCommonAncestor($root->right, $p, $q);

    // দুই দিকেই পাওয়া গেলে → এই নোডই LCA
    if ($left !== null && $right !== null) return $root;

    // যেদিকে পাওয়া গেছে সেদিক রিটার্ন করো
    return $left !== null ? $left : $right;
}
?>
```

**JavaScript:**

```javascript
// LCA খুঁজে বের করো
function lowestCommonAncestor(root, p, q) {
    if (root === null || root.val === p || root.val === q) {
        return root;
    }

    const left = lowestCommonAncestor(root.left, p, q);
    const right = lowestCommonAncestor(root.right, p, q);

    if (left !== null && right !== null) return root;
    return left !== null ? left : right;
}
```

---

### খ) Diameter of Binary Tree (বাইনারি ট্রি-র ব্যাস)

```
   ব্যাস = সবচেয়ে দূরবর্তী দুই নোডের মধ্যে দীর্ঘতম পথ

            [1]
           /   \
         [2]   [3]
        /   \
      [4]   [5]

   ব্যাস = 3 (পথ: 4 → 2 → 1 → 3 অথবা 5 → 2 → 1 → 3)
```

**PHP:**

```php
<?php
function diameterOfBinaryTree($root) {
    $maxDiameter = 0;

    function height($node) use (&$maxDiameter) {
        if ($node === null) return 0;

        $leftHeight = height($node->left);
        $rightHeight = height($node->right);

        // এই নোড দিয়ে যাওয়া পথের দৈর্ঘ্য
        $maxDiameter = max($maxDiameter, $leftHeight + $rightHeight);

        return 1 + max($leftHeight, $rightHeight);
    }

    height($root);
    return $maxDiameter;
}
?>
```

**JavaScript:**

```javascript
function diameterOfBinaryTree(root) {
    let maxDiameter = 0;

    function height(node) {
        if (node === null) return 0;

        const leftHeight = height(node.left);
        const rightHeight = height(node.right);

        // এই নোড দিয়ে যাওয়া পথের দৈর্ঘ্য
        maxDiameter = Math.max(maxDiameter, leftHeight + rightHeight);

        return 1 + Math.max(leftHeight, rightHeight);
    }

    height(root);
    return maxDiameter;
}
```

---

### গ) Check if Tree is Balanced (ব্যালান্সড কিনা পরীক্ষা)

```
   ব্যালান্সড:             আনব্যালান্সড:
       [1]                    [1]
      /   \                  /
    [2]   [3]              [2]
   /  \                   /
  [4] [5]              [3]

   ব্যালান্সড = প্রতিটি নোডে |height(left) - height(right)| ≤ 1
```

**PHP:**

```php
<?php
function isBalanced($root) {
    // -1 রিটার্ন করলে আনব্যালান্সড
    function checkHeight($node) {
        if ($node === null) return 0;

        $left = checkHeight($node->left);
        if ($left === -1) return -1;  // বাম আনব্যালান্সড

        $right = checkHeight($node->right);
        if ($right === -1) return -1;  // ডান আনব্যালান্সড

        // উচ্চতার পার্থক্য ১-এর বেশি হলে আনব্যালান্সড
        if (abs($left - $right) > 1) return -1;

        return 1 + max($left, $right);
    }

    return checkHeight($root) !== -1;
}
?>
```

**JavaScript:**

```javascript
function isBalanced(root) {
    function checkHeight(node) {
        if (node === null) return 0;

        const left = checkHeight(node.left);
        if (left === -1) return -1;

        const right = checkHeight(node.right);
        if (right === -1) return -1;

        if (Math.abs(left - right) > 1) return -1;

        return 1 + Math.max(left, right);
    }

    return checkHeight(root) !== -1;
}
```

---

### ঘ) Serialize / Deserialize Binary Tree

```
   সিরিয়ালাইজ: ট্রি → স্ট্রিং (সংরক্ষণের জন্য)
   ডিসিরিয়ালাইজ: স্ট্রিং → ট্রি (পুনরুদ্ধার)

       [1]
      /   \
    [2]   [3]
          / \
        [4] [5]

   প্রিঅর্ডার সিরিয়ালাইজ: "1,2,#,#,3,4,#,#,5,#,#"
   (# = null নোড)
```

**PHP:**

```php
<?php
// সিরিয়ালাইজ — ট্রি থেকে স্ট্রিং
function serialize($root) {
    if ($root === null) return "#";

    return $root->val . ","
         . serialize($root->left) . ","
         . serialize($root->right);
}

// ডিসিরিয়ালাইজ — স্ট্রিং থেকে ট্রি
function deserialize($data) {
    $nodes = explode(",", $data);
    $index = 0;

    function build(&$nodes, &$index) {
        if ($index >= count($nodes) || $nodes[$index] === "#") {
            $index++;
            return null;
        }

        $node = new TreeNode((int)$nodes[$index]);
        $index++;
        $node->left = build($nodes, $index);
        $node->right = build($nodes, $index);

        return $node;
    }

    return build($nodes, $index);
}

// উদাহরণ
$serialized = serialize($root);
echo "সিরিয়ালাইজড: $serialized\n";

$restored = deserialize($serialized);
echo "পুনরুদ্ধার সফল: " . (serialize($restored) === $serialized ? "হ্যাঁ" : "না") . "\n";
?>
```

**JavaScript:**

```javascript
// সিরিয়ালাইজ
function serialize(root) {
    if (root === null) return "#";
    return root.val + "," + serialize(root.left) + "," + serialize(root.right);
}

// ডিসিরিয়ালাইজ
function deserialize(data) {
    const nodes = data.split(",");
    let index = 0;

    function build() {
        if (index >= nodes.length || nodes[index] === "#") {
            index++;
            return null;
        }

        const node = new TreeNode(parseInt(nodes[index]));
        index++;
        node.left = build();
        node.right = build();

        return node;
    }

    return build();
}
```

---

### ঙ) Construct BST from Preorder Traversal

```
   প্রিঅর্ডার: [8, 5, 1, 7, 10, 12]

   ধাপে ধাপে:
   8 → রুট
   5 < 8 → বামে
   1 < 5 → বামে
   7 > 5 কিন্তু < 8 → 5-এর ডানে
   10 > 8 → ডানে
   12 > 10 → ডানে

   ফলাফল:
        [8]
       /   \
     [5]   [10]
    /   \      \
  [1]   [7]   [12]
```

**PHP:**

```php
<?php
function bstFromPreorder($preorder) {
    $index = 0;

    function build(&$preorder, &$index, $min, $max) {
        if ($index >= count($preorder)) return null;

        $val = $preorder[$index];
        // মান সীমার বাইরে হলে null
        if ($val < $min || $val > $max) return null;

        $node = new TreeNode($val);
        $index++;

        $node->left = build($preorder, $index, $min, $val);   // বাম সীমা
        $node->right = build($preorder, $index, $val, $max);  // ডান সীমা

        return $node;
    }

    return build($preorder, $index, PHP_INT_MIN, PHP_INT_MAX);
}

$tree = bstFromPreorder([8, 5, 1, 7, 10, 12]);
?>
```

**JavaScript:**

```javascript
function bstFromPreorder(preorder) {
    let index = 0;

    function build(min, max) {
        if (index >= preorder.length) return null;

        const val = preorder[index];
        if (val < min || val > max) return null;

        const node = new TreeNode(val);
        index++;

        node.left = build(min, val);
        node.right = build(val, max);

        return node;
    }

    return build(-Infinity, Infinity);
}

const tree = bstFromPreorder([8, 5, 1, 7, 10, 12]);
```

---

## ৮. সম্পূর্ণ প্রজেক্ট: ফাইল সিস্টেম ট্রি 📁

N-ary tree ব্যবহার করে ফাইল সিস্টেম সিমুলেশন।

### সুবিধাসমূহ:

- `mkdir` — ডিরেক্টরি তৈরি
- `createFile` — ফাইল তৈরি (সাইজসহ)
- `ls` — ডিরেক্টরির বিষয়বস্তু দেখো
- `find` — নাম দিয়ে খোঁজো
- `du` — ডিস্ক ব্যবহার গণনা
- `tree` — ট্রি আকারে দেখাও

```
   উদাহরণ আউটপুট:

   /
   ├── home/
   │   ├── user/
   │   │   ├── .bashrc (1KB)
   │   │   └── docs/
   │   │       └── resume.pdf (50KB)
   │   └── shared/
   │       └── photo.jpg (200KB)
   └── etc/
       └── config.ini (2KB)

   মোট ডিস্ক ব্যবহার: 253KB
```

**PHP:**

```php
<?php
// ফাইল সিস্টেম নোড — ফাইল বা ডিরেক্টরি
class FSNode {
    public $name;        // নাম
    public $isDir;       // ডিরেক্টরি কিনা
    public $size;        // ফাইলের আকার (বাইটে)
    public $children;    // চাইল্ড নোডের তালিকা (ডিরেক্টরির জন্য)

    public function __construct($name, $isDir = true, $size = 0) {
        $this->name = $name;
        $this->isDir = $isDir;
        $this->size = $size;
        $this->children = [];
    }
}

class FileSystem {
    private $root;

    public function __construct() {
        $this->root = new FSNode("/");  // রুট ডিরেক্টরি
    }

    // পাথ অনুসরণ করে নোড খুঁজে বের করো
    private function navigate($path) {
        $parts = array_filter(explode("/", $path));
        $current = $this->root;

        foreach ($parts as $part) {
            $found = false;
            foreach ($current->children as $child) {
                if ($child->name === $part) {
                    $current = $child;
                    $found = true;
                    break;
                }
            }
            if (!$found) return null;  // পাথ পাওয়া যায়নি
        }

        return $current;
    }

    // ডিরেক্টরি তৈরি করো
    public function mkdir($path) {
        $parts = array_filter(explode("/", $path));
        $current = $this->root;

        // প্রতিটি অংশের জন্য ডিরেক্টরি তৈরি করো (যদি না থাকে)
        foreach ($parts as $part) {
            $found = false;
            foreach ($current->children as $child) {
                if ($child->name === $part && $child->isDir) {
                    $current = $child;
                    $found = true;
                    break;
                }
            }
            if (!$found) {
                $newDir = new FSNode($part);
                $current->children[] = $newDir;
                $current = $newDir;
            }
        }

        return "ডিরেক্টরি তৈরি হয়েছে: $path";
    }

    // ফাইল তৈরি করো
    public function createFile($path, $size = 0) {
        $parts = array_filter(explode("/", $path));
        $fileName = array_pop($parts);  // শেষ অংশ = ফাইলের নাম
        $dirPath = "/" . implode("/", $parts);

        // প্যারেন্ট ডিরেক্টরি তৈরি করো (যদি না থাকে)
        if (!empty($parts)) $this->mkdir($dirPath);

        $parent = $this->navigate($dirPath) ?? $this->root;
        $file = new FSNode($fileName, false, $size);
        $parent->children[] = $file;

        return "ফাইল তৈরি হয়েছে: $path ({$size}KB)";
    }

    // ls — ডিরেক্টরির বিষয়বস্তু
    public function ls($path = "/") {
        $node = $this->navigate($path) ?? $this->root;
        if (!$node->isDir) return [$node->name];

        $items = [];
        foreach ($node->children as $child) {
            $suffix = $child->isDir ? "/" : " ({$child->size}KB)";
            $items[] = $child->name . $suffix;
        }

        return $items;
    }

    // find — নাম দিয়ে খোঁজো (DFS)
    public function find($name, $node = null, $path = "") {
        $node = $node ?? $this->root;
        $results = [];
        $currentPath = $path . "/" . $node->name;

        if ($node->name === $name) {
            $results[] = $currentPath;
        }

        foreach ($node->children as $child) {
            $results = array_merge($results, $this->find($name, $child, $currentPath));
        }

        return $results;
    }

    // du — ডিস্ক ব্যবহার গণনা (Postorder ট্রাভার্সাল)
    public function du($path = "/") {
        $node = $this->navigate($path) ?? $this->root;
        return $this->_calculateSize($node);
    }

    private function _calculateSize($node) {
        if (!$node->isDir) return $node->size;

        $total = 0;
        foreach ($node->children as $child) {
            $total += $this->_calculateSize($child);  // চাইল্ডদের সাইজ যোগ করো
        }
        return $total;
    }

    // tree — ট্রি আকারে ডিরেক্টরি দেখাও
    public function tree($node = null, $prefix = "", $isLast = true) {
        $node = $node ?? $this->root;
        $output = "";

        // বর্তমান নোড প্রিন্ট
        $connector = $isLast ? "└── " : "├── ";
        $suffix = $node->isDir ? "/" : " ({$node->size}KB)";

        if ($node === $this->root) {
            $output .= $node->name . "\n";
        } else {
            $output .= $prefix . $connector . $node->name . $suffix . "\n";
        }

        // চাইল্ডদের জন্য রিকার্সিভ কল
        $children = $node->children;
        $count = count($children);

        for ($i = 0; $i < $count; $i++) {
            $childPrefix = $node === $this->root ? "" : $prefix . ($isLast ? "    " : "│   ");
            $output .= $this->tree($children[$i], $childPrefix, $i === $count - 1);
        }

        return $output;
    }
}

// ব্যবহার — ফাইল সিস্টেম তৈরি ও পরিচালনা
$fs = new FileSystem();

// ডিরেক্টরি তৈরি
echo $fs->mkdir("/home/user") . "\n";
echo $fs->mkdir("/home/user/docs") . "\n";
echo $fs->mkdir("/home/shared") . "\n";
echo $fs->mkdir("/etc") . "\n";

// ফাইল তৈরি
echo $fs->createFile("/home/user/.bashrc", 1) . "\n";
echo $fs->createFile("/home/user/docs/resume.pdf", 50) . "\n";
echo $fs->createFile("/home/shared/photo.jpg", 200) . "\n";
echo $fs->createFile("/etc/config.ini", 2) . "\n";

// ট্রি দেখাও
echo "\n📁 ফাইল সিস্টেম ট্রি:\n";
echo $fs->tree();

// ls
echo "\n📂 /home/user এর বিষয়বস্তু:\n";
echo implode("\n", $fs->ls("/home/user")) . "\n";

// du
echo "\nমোট ডিস্ক ব্যবহার: " . $fs->du() . "KB\n";
echo "/home এর ডিস্ক ব্যবহার: " . $fs->du("/home") . "KB\n";

// find
echo "\n🔍 'resume.pdf' খোঁজা:\n";
echo implode("\n", $fs->find("resume.pdf")) . "\n";
?>
```

**JavaScript:**

```javascript
// ফাইল সিস্টেম নোড
class FSNode {
    constructor(name, isDir = true, size = 0) {
        this.name = name;
        this.isDir = isDir;
        this.size = size;
        this.children = [];
    }
}

class FileSystem {
    constructor() {
        this.root = new FSNode("/");
    }

    // পাথ অনুসরণ করে নোড খোঁজো
    _navigate(path) {
        const parts = path.split("/").filter(Boolean);
        let current = this.root;

        for (const part of parts) {
            const child = current.children.find(c => c.name === part);
            if (!child) return null;
            current = child;
        }

        return current;
    }

    // ডিরেক্টরি তৈরি
    mkdir(path) {
        const parts = path.split("/").filter(Boolean);
        let current = this.root;

        for (const part of parts) {
            let child = current.children.find(c => c.name === part && c.isDir);
            if (!child) {
                child = new FSNode(part);
                current.children.push(child);
            }
            current = child;
        }

        return `ডিরেক্টরি তৈরি হয়েছে: ${path}`;
    }

    // ফাইল তৈরি
    createFile(path, size = 0) {
        const parts = path.split("/").filter(Boolean);
        const fileName = parts.pop();
        const dirPath = "/" + parts.join("/");

        if (parts.length > 0) this.mkdir(dirPath);

        const parent = this._navigate(dirPath) || this.root;
        parent.children.push(new FSNode(fileName, false, size));

        return `ফাইল তৈরি হয়েছে: ${path} (${size}KB)`;
    }

    // ls
    ls(path = "/") {
        const node = this._navigate(path) || this.root;
        if (!node.isDir) return [node.name];

        return node.children.map(c =>
            c.name + (c.isDir ? "/" : ` (${c.size}KB)`)
        );
    }

    // find — DFS দিয়ে খোঁজো
    find(name, node = this.root, path = "") {
        const results = [];
        const currentPath = path + "/" + node.name;

        if (node.name === name) results.push(currentPath);

        for (const child of node.children) {
            results.push(...this.find(name, child, currentPath));
        }

        return results;
    }

    // du — ডিস্ক ব্যবহার (Postorder)
    du(path = "/") {
        const node = this._navigate(path) || this.root;
        return this._calcSize(node);
    }

    _calcSize(node) {
        if (!node.isDir) return node.size;
        return node.children.reduce((sum, c) => sum + this._calcSize(c), 0);
    }

    // tree — সুন্দর আকারে দেখাও
    tree(node = this.root, prefix = "", isLast = true) {
        let output = "";

        const connector = isLast ? "└── " : "├── ";
        const suffix = node.isDir ? "/" : ` (${node.size}KB)`;

        if (node === this.root) {
            output += node.name + "\n";
        } else {
            output += prefix + connector + node.name + suffix + "\n";
        }

        const children = node.children;
        children.forEach((child, i) => {
            const childPrefix = node === this.root
                ? ""
                : prefix + (isLast ? "    " : "│   ");
            output += this.tree(child, childPrefix, i === children.length - 1);
        });

        return output;
    }
}

// ব্যবহার
const fs = new FileSystem();

fs.mkdir("/home/user");
fs.mkdir("/home/user/docs");
fs.mkdir("/home/shared");
fs.mkdir("/etc");

fs.createFile("/home/user/.bashrc", 1);
fs.createFile("/home/user/docs/resume.pdf", 50);
fs.createFile("/home/shared/photo.jpg", 200);
fs.createFile("/etc/config.ini", 2);

console.log("📁 ফাইল সিস্টেম ট্রি:");
console.log(fs.tree());

console.log("📂 /home/user:");
console.log(fs.ls("/home/user").join("\n"));

console.log(`\nমোট ডিস্ক ব্যবহার: ${fs.du()}KB`);
console.log(`/home: ${fs.du("/home")}KB`);

console.log("\n🔍 'resume.pdf' খোঁজা:");
console.log(fs.find("resume.pdf").join("\n"));
```

---

## 📋 ট্রি কমপ্লেক্সিটি চিটশিট

```
   ════════════════════════════════════════════════════════════════════
   ডেটা স্ট্রাকচার │  সার্চ      │  ইনসার্ট    │  ডিলিট      │ স্পেস
   ════════════════════════════════════════════════════════════════════
   BST (গড়)       │ O(log n)    │ O(log n)    │ O(log n)    │ O(n)
   BST (সর্বনিম্ন) │ O(n)        │ O(n)        │ O(n)        │ O(n)
   ────────────────────────────────────────────────────────────────────
   AVL Tree        │ O(log n)    │ O(log n)    │ O(log n)    │ O(n)
                   │ গ্যারান্টিড │ গ্যারান্টিড │ গ্যারান্টিড │
   ────────────────────────────────────────────────────────────────────
   Red-Black Tree  │ O(log n)    │ O(log n)    │ O(log n)    │ O(n)
                   │ গ্যারান্টিড │ গ্যারান্টিড │ গ্যারান্টিড │
   ────────────────────────────────────────────────────────────────────
   Max/Min Heap    │ O(n)        │ O(log n)    │ O(log n)    │ O(n)
   (peek: O(1))   │             │             │ (রুট ডিলিট)  │
   ────────────────────────────────────────────────────────────────────
   Heap Sort       │     —       │     —       │     —       │ O(1)
   সময়: O(n log n)│             │             │             │ in-place
   ────────────────────────────────────────────────────────────────────
   Build Heap      │     —       │     —       │     —       │ O(n)
   সময়: O(n)      │             │             │             │
   ════════════════════════════════════════════════════════════════════

   ট্রাভার্সাল কমপ্লেক্সিটি:
   ─────────────────────────────────────────
   │ সব ট্রাভার্সাল │ সময়: O(n) │ স্পেস: O(h) │
   │ (Inorder,       │           │ h = উচ্চতা   │
   │  Preorder,      │           │ worst: O(n)  │
   │  Postorder,     │           │ best: O(logn)│
   │  Level Order)   │           │ BFS: O(w)    │
   ─────────────────────────────────────────
   w = সর্বোচ্চ প্রস্থ (সবচেয়ে বেশি নোডের লেভেল)
```

---

## 🎯 ইন্টারভিউ টিপস

### ১. সবচেয়ে বেশি জিজ্ঞাসিত ট্রি প্রশ্ন

```
   ─────────────────────────────────────────────────────────────
   # │ প্রশ্ন                                │ কৌশল
   ─────────────────────────────────────────────────────────────
   ১ │ BST ভ্যালিডেশন                       │ min/max সীমা রিকার্সন
   ২ │ LCA খোঁজা                            │ রিকার্সিভ DFS
   ৩ │ ট্রি-র উচ্চতা/গভীরতা                 │ রিকার্সিভ — max(L,R)+1
   ৪ │ লেভেল অর্ডার ট্রাভার্সাল             │ কিউ (BFS)
   ৫ │ ব্যালান্সড কিনা পরীক্ষা              │ উচ্চতা গণনা + -1 ট্রিক
   ৬ │ ব্যাস বের করা                         │ height ফাংশনে ম্যাক্স আপডেট
   ৭ │ সিরিয়ালাইজ/ডিসিরিয়ালাইজ            │ প্রিঅর্ডার + null মার্কার
   ৮ │ BST-র k-তম ক্ষুদ্রতম                 │ ইনঅর্ডার + কাউন্টার
   ৯ │ ইনভার্ট বাইনারি ট্রি                  │ রিকার্সিভ swap
   ১০│ প্যাথ সাম (রুট → লিফ)               │ DFS + টার্গেট কমানো
   ─────────────────────────────────────────────────────────────
```

### ২. গুরুত্বপূর্ণ কৌশল

```
   🔑 মনে রাখার বিষয়:
   ─────────────────────────────────────────
   ✅ BST → ইনঅর্ডার = সর্টেড অ্যারে
   ✅ "সবচেয়ে কাছের" → BFS (লেভেল অর্ডার)
   ✅ "সব পথ/সম্ভাবনা" → DFS (প্রিঅর্ডার/পোস্টঅর্ডার)
   ✅ ট্রি সমস্যা → প্রায়ই রিকার্সিভ সমাধান সবচেয়ে সহজ
   ✅ "বটম-আপ" গণনা → পোস্টঅর্ডার
   ✅ "টপ-ডাউন" তথ্য পাস → প্রিঅর্ডার
   ✅ হিপ → "k-তম সবচেয়ে বড়/ছোট" সমস্যায়
   ✅ স্কিউড ট্রি এড়াতে → AVL/Red-Black ব্যবহার
```

### ৩. সাক্ষাৎকারে পদ্ধতি

```
   ১. প্রশ্ন বুঝো → কী ধরনের ট্রি? BST নাকি সাধারণ?
   ২. এজ কেস ভাবো → খালি ট্রি, একটি নোড, স্কিউড ট্রি
   ৩. ট্রাভার্সাল বেছে নাও → কোনটি এই সমস্যায় উপযুক্ত?
   ৪. রিকার্সিভ দিয়ে শুরু করো → সহজ ও পরিষ্কার
   ৫. প্রয়োজনে ইটারেটিভ দেখাও → স্ট্যাক ওভারফ্লো এড়াতে
   ৬. কমপ্লেক্সিটি বলো → সময় ও স্পেস উভয়ই
```

---

## 🔗 পরবর্তী অধ্যায়

গ্রাফ (Graph) — ট্রি-এর সাধারণীকৃত রূপ, সাইকেল, BFS/DFS বিস্তারিত → [graph.md](./graph.md)

---

> _"প্রতিটি বড় সমস্যাকে ছোট ছোট সাবট্রি-তে ভাঙো — রিকার্সন তোমার সেরা বন্ধু।"_ 🌳

