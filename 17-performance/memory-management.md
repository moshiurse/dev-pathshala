# 🧠 মেমোরি ম্যানেজমেন্ট (Memory Management)

> **সফটওয়্যার পারফরম্যান্সের মূল ভিত্তি — মেমোরি কীভাবে কাজ করে, কীভাবে লিক হয়, এবং কীভাবে অপটিমাইজ করবেন**

---

## 📖 সূচিপত্র

- [মেমোরি অ্যালোকেশন — Stack vs Heap](#-মেমোরি-অ্যালোকেশন--stack-vs-heap)
- [গার্বেজ কালেকশন স্ট্র্যাটেজি](#-গার্বেজ-কালেকশন-স্ট্র্যাটেজি)
- [PHP মেমোরি ম্যানেজমেন্ট](#-php-মেমোরি-ম্যানেজমেন্ট)
- [JavaScript V8 মেমোরি](#-javascript-v8-মেমোরি)
- [মেমোরি লিক ডিটেকশন](#-মেমোরি-লিক-ডিটেকশন)
- [সাধারণ মেমোরি লিক প্যাটার্ন](#-সাধারণ-মেমোরি-লিক-প্যাটার্ন)
- [কনফিগারেশন ও টিউনিং](#-কনফিগারেশন-ও-টিউনিং)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)
- [বাস্তব উদাহরণ — বাংলাদেশ](#-বাস্তব-উদাহরণ--বাংলাদেশ)

---

## 📌 মেমোরি অ্যালোকেশন — Stack vs Heap

প্রোগ্রাম চলার সময় মেমোরি দুইভাবে বরাদ্দ হয় — **Stack** এবং **Heap**। এই দুটি ধারণা বুঝলে
আপনি বুঝবেন কেন কিছু কোড দ্রুত আর কিছু কোড ধীর।

```
┌─────────────────────────────────────────────────┐
│                  RAM (মেমোরি)                     │
├─────────────────────────────────────────────────┤
│                                                   │
│   ┌───────────────────┐                           │
│   │      STACK        │  ← ছোট, দ্রুত, LIFO        │
│   │  local variables  │  ← ফাংশন শেষে auto মুছে যায় │
│   │  function calls   │  ← সাইজ কম্পাইল টাইমে জানা  │
│   │  primitive types  │                           │
│   ├───────────────────┤                           │
│   │     ↕ ↕ ↕ ↕       │  ← দুটি বিপরীতে বাড়ে       │
│   ├───────────────────┤                           │
│   │      HEAP         │  ← বড়, ধীর, dynamic        │
│   │  objects          │  ← ম্যানুয়াল/GC ফ্রি করতে হয় │
│   │  arrays           │  ← রানটাইমে সাইজ ঠিক হয়     │
│   │  closures         │                           │
│   └───────────────────┘                           │
│                                                   │
└─────────────────────────────────────────────────┘
```

### Stack কীভাবে কাজ করে (LIFO):

```
ফাংশন কল হলে:                    ফাংশন শেষ হলে:

┌─────────────────┐               ┌─────────────────┐
│  inner()        │ ← TOP         │                 │ (inner মুছে গেছে)
├─────────────────┤               ├─────────────────┤
│  calculate()    │               │  calculate()    │ ← TOP
├─────────────────┤               ├─────────────────┤
│  main()         │               │  main()         │
└─────────────────┘               └─────────────────┘
```

### Stack vs Heap তুলনা:

| বৈশিষ্ট্য        | Stack              | Heap                    |
|------------------|--------------------|-------------------------|
| গতি              | ⚡ অনেক দ্রুত        | 🐢 তুলনামূলক ধীর          |
| সাইজ             | ছোট (সাধারণত 1-8MB) | বড় (GB পর্যন্ত)          |
| অ্যালোকেশন       | স্বয়ংক্রিয়          | ম্যানুয়াল / GC            |
| ডেটা টাইপ        | Primitives, refs   | Objects, Arrays          |
| লাইফটাইম         | ফাংশন স্কোপ          | ম্যানুয়াল নিয়ন্ত্রণ        |
| ফ্র্যাগমেন্টেশন   | নেই                | হতে পারে                 |

### কোড উদাহরণ:

```javascript
// Stack এ যায় — primitive types
let name = "Rahim";        // stack
let age = 25;              // stack
let isActive = true;       // stack

// Heap এ যায় — objects, arrays
let user = {               // reference stack এ, object heap এ
    name: "Rahim",
    orders: [1, 2, 3]
};

let items = [1, 2, 3, 4];  // reference stack এ, array heap এ
```

```php
<?php
// Stack allocation
$count = 100;              // stack
$name = "Karim";           // stack (PHP internally zval)

// Heap allocation
$users = [];               // heap
for ($i = 0; $i < 10000; $i++) {
    $users[] = [
        'id' => $i,
        'name' => "User $i",
        'email' => "user$i@example.com"
    ];
}
// $users array টি heap এ বড় জায়গা নিচ্ছে
```

---

## 🗑️ গার্বেজ কালেকশন স্ট্র্যাটেজি

গার্বেজ কালেকশন (GC) হলো অব্যবহৃত মেমোরি স্বয়ংক্রিয়ভাবে ফিরিয়ে আনার প্রক্রিয়া।
মনে করুন bKash-এ প্রতিদিন লক্ষ লক্ষ transaction হয় — পুরনো, সম্পন্ন হওয়া transaction এর
temporary data মুছে না ফেললে সার্ভার ক্র্যাশ করবে। GC ঠিক এই কাজটাই করে।

### 1️⃣ Reference Counting (রেফারেন্স কাউন্টিং):

প্রতিটি অবজেক্টের সাথে একটি কাউন্টার থাকে — কতজন সেটি ব্যবহার করছে।
কাউন্টার ০ হলে মেমোরি ফ্রি।

```
অবজেক্ট তৈরি হলে:        রেফারেন্স যোগ:          রেফারেন্স মুছলে:

┌─────────┐              ┌─────────┐             ┌─────────┐
│ Object  │              │ Object  │             │ Object  │
│ ref: 1  │              │ ref: 2  │             │ ref: 0  │ ← ফ্রি!
└─────────┘              └─────────┘             └─────────┘
    ↑                      ↑   ↑                   (কেউ নেই)
    │                      │   │
   var A                var A  var B
```

**সমস্যা — Circular Reference:**
```
┌─────────┐       ┌─────────┐
│ Object A│──────→│ Object B│
│ ref: 1  │←──────│ ref: 1  │
└─────────┘       └─────────┘

A → B কে রেফার করে (B এর ref = 1)
B → A কে রেফার করে (A এর ref = 1)
দুজনকেই unset করলেও ref কখনো 0 হবে না!
এটাই circular reference সমস্যা।
```

### 2️⃣ Mark-and-Sweep (মার্ক অ্যান্ড সুইপ):

দুই ধাপে কাজ করে:
- **Mark**: root থেকে শুরু করে সব reachable অবজেক্ট চিহ্নিত করে
- **Sweep**: চিহ্নিত নয় এমন সব অবজেক্ট মুছে ফেলে

```
Mark Phase:                          Sweep Phase:

  ROOT                                ROOT
   │                                   │
   ▼                                   ▼
┌──────┐    ┌──────┐              ┌──────┐    ┌──────┐
│  A ✓ │───→│  B ✓ │              │  A   │───→│  B   │
└──────┘    └──────┘              └──────┘    └──────┘
                                  
┌──────┐    ┌──────┐              ╳╳╳╳╳╳╳╳    ╳╳╳╳╳╳╳╳
│  C ✗ │───→│  D ✗ │              ╳ FREED ╳    ╳ FREED ╳
└──────┘    └──────┘              ╳╳╳╳╳╳╳╳    ╳╳╳╳╳╳╳╳

C, D root থেকে পৌঁছানো যায় না → মুছে ফেলা হবে
```

### 3️⃣ Generational GC (জেনারেশনাল):

ধারণা: বেশিরভাগ অবজেক্ট অল্প সময় বাঁচে ("infant mortality hypothesis")।

```
┌─────────────────────────────────────────────────┐
│              Generational Heap                    │
├─────────────────────────────────────────────────┤
│                                                   │
│  Generation 0 (Young/নতুন)     ← ঘন ঘন GC হয়     │
│  ┌──┬──┬──┬──┬──┬──┬──┬──┐                        │
│  │░░│██│░░│░░│██│░░│██│░░│   ░░ = alive            │
│  └──┴──┴──┴──┴──┴──┴──┴──┘   ██ = garbage          │
│          │ (survive করলে promote)                  │
│          ▼                                        │
│  Generation 1 (Old/পুরনো)     ← মাঝে মাঝে GC হয়   │
│  ┌──┬──┬──┬──┬──┐                                  │
│  │░░│░░│██│░░│░░│                                  │
│  └──┴──┴──┴──┴──┘                                  │
│          │ (survive করলে promote)                  │
│          ▼                                        │
│  Generation 2 (Permanent)     ← কদাচিৎ GC হয়      │
│  ┌──┬──┬──┐                                        │
│  │░░│░░│░░│                                        │
│  └──┴──┴──┘                                        │
│                                                   │
└─────────────────────────────────────────────────┘
```

**বাস্তব উপমা — Daraz গুদাম:**
- Gen 0 = নতুন অর্ডার — বেশিরভাগ দ্রুত deliver হয়ে যায়, প্রতিদিন চেক হয়
- Gen 1 = প্রসেসিং এ আছে — সপ্তাহে চেক করা হয়
- Gen 2 = দীর্ঘমেয়াদী স্টক — মাসে একবার অডিট হয়

---

## 🐘 PHP মেমোরি ম্যানেজমেন্ট

PHP অভ্যন্তরীণভাবে **zval** (Zend Value) স্ট্রাকচার ব্যবহার করে প্রতিটি variable ট্র্যাক করতে।

### zval স্ট্রাকচার:

```
┌──────────────────────────┐
│         zval              │
├──────────────────────────┤
│  value    → actual data  │
│  type     → IS_STRING,   │
│             IS_ARRAY...  │
│  refcount → 1            │  ← কতজন এটি ব্যবহার করছে
│  is_ref   → false        │  ← reference (&) কিনা
└──────────────────────────┘
```

### Reference Counting উদাহরণ:

```php
<?php
// Step 1: zval তৈরি হলো, refcount = 1
$a = "Hello Dhaka";

// Step 2: $b = $a → Copy-on-Write, refcount = 2
$b = $a;

// মেমোরি অবস্থা:
// ┌────────────────┐
// │ zval           │
// │ "Hello Dhaka"  │
// │ refcount: 2    │ ← $a এবং $b দুজনেই পয়েন্ট করছে
// └────────────────┘
//     ↑       ↑
//    $a      $b

// Step 3: $b পরিবর্তন → Copy-on-Write ট্রিগার হলো
$b = "Hello Chittagong";

// মেমোরি অবস্থা:
// ┌────────────────┐    ┌──────────────────┐
// │ zval           │    │ zval             │
// │ "Hello Dhaka"  │    │ "Hello Chittagong"│
// │ refcount: 1    │    │ refcount: 1      │
// └────────────────┘    └──────────────────┘
//     ↑                      ↑
//    $a                     $b
```

### মেমোরি ট্র্যাকিং:

```php
<?php
// মেমোরি ব্যবহার পরিমাপ
$before = memory_get_usage();

$array = range(1, 100000);

$after = memory_get_usage();
$peak = memory_get_peak_usage();

echo "ব্যবহৃত: " . number_format($after - $before) . " bytes\n";
echo "সর্বোচ্চ: " . number_format($peak) . " bytes\n";

// বড় array ফ্রি করা
unset($array);
echo "ফ্রি করার পর: " . number_format(memory_get_usage()) . " bytes\n";
```

### ❌ Circular Reference সমস্যা:

```php
<?php
// ❌ খারাপ — Circular Reference তৈরি হচ্ছে
class Node {
    public $data;
    public $ref;

    public function __construct($data) {
        $this->data = $data;
    }
}

$a = new Node("Pathao Rider");
$b = new Node("Pathao User");
$a->ref = $b;    // A → B
$b->ref = $a;    // B → A (circular!)

unset($a, $b);
// মেমোরি এখনও ফ্রি হয়নি!
// কারণ refcount কখনো 0 হবে না

// সমাধান: GC ম্যানুয়ালি চালান
$collected = gc_collect_cycles();
echo "পরিষ্কার হয়েছে: $collected টি cycle\n";
```

### ✅ সঠিক পদ্ধতি — WeakReference ব্যবহার (PHP 7.4+):

```php
<?php
// ✅ ভালো — WeakReference ব্যবহার করে circular reference এড়ানো
class TreeNode {
    public string $name;
    public ?WeakReference $parent = null;
    public array $children = [];

    public function __construct(string $name) {
        $this->name = $name;
    }

    public function addChild(TreeNode $child): void {
        $this->children[] = $child;
        // WeakReference ব্যবহার — GC কে ব্লক করবে না
        $child->parent = WeakReference::create($this);
    }
}

$root = new TreeNode("Grameenphone HQ");
$child = new TreeNode("GP Dhaka Office");
$root->addChild($child);

// parent access করতে:
$parentNode = $child->parent->get();
echo $parentNode->name; // "Grameenphone HQ"
```

### PHP মেমোরি অপটিমাইজেশন কৌশল:

```php
<?php
// ❌ খারাপ — পুরো ফাইল মেমোরিতে লোড
$lines = file('huge_log.txt'); // 500MB ফাইল = 500MB RAM!
foreach ($lines as $line) {
    process($line);
}

// ✅ ভালো — Generator ব্যবহার করে লাইন বাই লাইন পড়া
function readLines(string $file): Generator {
    $handle = fopen($file, 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;  // একবারে একটি লাইন মেমোরিতে
    }
    fclose($handle);
}

foreach (readLines('huge_log.txt') as $line) {
    process($line); // মেমোরি ব্যবহার সর্বদা কম থাকবে
}
```

---

## ⚡ JavaScript V8 মেমোরি

V8 ইঞ্জিন (Chrome, Node.js) একটি sophisticated generational GC ব্যবহার করে।

### V8 Heap Structure:

```
┌─────────────────────────────────────────────────────┐
│                    V8 Heap                            │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────┐         │
│  │     Young Generation (1-8 MB)            │         │
│  │  ┌──────────────┬──────────────┐         │         │
│  │  │  Semi-space   │  Semi-space   │         │         │
│  │  │   (From)      │   (To)        │         │         │
│  │  │  ░░░██░░██░░  │  (খালি)       │         │         │
│  │  └──────────────┴──────────────┘         │         │
│  │  Scavenger (Minor GC) — দ্রুত, ঘন ঘন      │         │
│  └─────────────────────────────────────────┘         │
│           │ survive করলে promote                      │
│           ▼                                          │
│  ┌─────────────────────────────────────────┐         │
│  │     Old Generation (শত MB পর্যন্ত)       │         │
│  │  ┌──────────────────────────────────┐    │         │
│  │  │  ░░░░░░░██░░░░░░██░░░░░░░░░░░░  │    │         │
│  │  └──────────────────────────────────┘    │         │
│  │  Mark-Compact (Major GC) — ধীর, কম ঘন ঘন │         │
│  └─────────────────────────────────────────┘         │
│                                                       │
│  ┌─────────────────────────────────────────┐         │
│  │     Large Object Space                    │         │
│  │  (বড় অবজেক্ট — সরাসরি এখানে)              │         │
│  └─────────────────────────────────────────┘         │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Scavenger (Minor GC) কীভাবে কাজ করে:

```
ধাপ ১: From space ভর্তি          ধাপ ২: Alive গুলো To তে কপি

From:                              From:              To:
┌──┬──┬──┬──┬──┬──┐               ┌──┬──┬──┬──┬──┬──┐  ┌──┬──┬──┐
│A │░░│B │░░│C │░░│               │  │  │  │  │  │  │  │A │B │C │
└──┴──┴──┴──┴──┴──┘               └──┴──┴──┴──┴──┴──┘  └──┴──┴──┘
 ✓  ✗  ✓  ✗  ✓  ✗                 (পরিষ্কার)             (কম্প্যাক্ট)

ধাপ ৩: From ↔ To অদলবদল → পুনরায় শুরু
```

### ❌ মেমোরি লিক — ক্রমবর্ধমান ক্যাশ:

```javascript
// ❌ খারাপ — cache সীমাহীনভাবে বাড়ছে
let cache = [];

setInterval(() => {
    // Nagad-এর প্রতিটি transaction ক্যাশ করছি
    cache.push({
        id: Date.now(),
        data: new Array(10000).fill('transaction-data'),
        timestamp: new Date()
    });
    // cache কখনো পরিষ্কার হচ্ছে না = মেমোরি লিক!
    console.log(`Cache size: ${cache.length}`);
}, 100);
```

### ✅ সমাধান — Bounded Cache:

```javascript
// ✅ ভালো — সীমিত আকারের ক্যাশ
class BoundedCache {
    constructor(maxSize = 100) {
        this.maxSize = maxSize;
        this.cache = new Map();
    }

    set(key, value) {
        // সীমা অতিক্রম করলে পুরনোটি মুছে ফেলো
        if (this.cache.size >= this.maxSize) {
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        this.cache.set(key, value);
    }

    get(key) {
        return this.cache.get(key);
    }

    get size() {
        return this.cache.size;
    }
}

// ব্যবহার — bKash transaction cache
const txnCache = new BoundedCache(1000);
txnCache.set('txn_001', { amount: 500, to: '01711XXXXXX' });
```

### WeakRef এবং FinalizationRegistry (ES2021):

```javascript
// ✅ ভালো — WeakRef ব্যবহার করে বড় অবজেক্ট ক্যাশ
class SmartCache {
    constructor() {
        this.cache = new Map();
        this.registry = new FinalizationRegistry((key) => {
            // GC যখন অবজেক্ট মুছবে, ক্যাশ থেকেও সরাও
            this.cache.delete(key);
            console.log(`GC cleaned: ${key}`);
        });
    }

    set(key, value) {
        const ref = new WeakRef(value);
        this.cache.set(key, ref);
        this.registry.register(value, key);
    }

    get(key) {
        const ref = this.cache.get(key);
        if (!ref) return undefined;
        const value = ref.deref();
        if (!value) {
            this.cache.delete(key);
            return undefined;
        }
        return value;
    }
}
```

---

## 🔍 মেমোরি লিক ডিটেকশন

মেমোরি লিক চিহ্নিত করার জন্য বিভিন্ন টুল ও কৌশল আছে।

### Chrome DevTools দিয়ে ডিটেকশন:

```
Chrome DevTools → Memory Tab

1. Heap Snapshot নিন:
   ┌─────────────────────────────────────┐
   │  Take Heap Snapshot                   │
   │  ┌────────────────────────────┐      │
   │  │ Snapshot 1: 5.2 MB         │      │
   │  │ Snapshot 2: 8.7 MB         │ ← !  │
   │  │ Snapshot 3: 12.1 MB        │ ← !! │
   │  └────────────────────────────┘      │
   │  ক্রমাগত বাড়ছে = সম্ভবত লিক আছে!     │
   └─────────────────────────────────────┘

2. Comparison View দিয়ে দেখুন কী বাড়ছে:
   ┌─────────────────────────────────────┐
   │ Constructor    │ # New  │ Size Delta │
   │────────────────┼────────┼───────────│
   │ Array          │ +5000  │ +3.5 MB   │ ← সন্দেহজনক!
   │ Object         │ +200   │ +0.1 MB   │
   │ EventListener  │ +50    │ +0.05 MB  │
   └─────────────────────────────────────┘
```

### Node.js মেমোরি প্রোফাইলিং:

```javascript
// Node.js এ মেমোরি মনিটরিং
function logMemory(label) {
    const usage = process.memoryUsage();
    console.log(`[${label}]`);
    console.log(`  RSS:       ${(usage.rss / 1024 / 1024).toFixed(2)} MB`);
    console.log(`  Heap Total: ${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`);
    console.log(`  Heap Used:  ${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`);
    console.log(`  External:   ${(usage.external / 1024 / 1024).toFixed(2)} MB`);
}

// ব্যবহার
logMemory('শুরু');
// ... কোড চালান ...
logMemory('শেষ');

// --inspect ফ্ল্যাগ দিয়ে চালান:
// node --inspect app.js
// Chrome এ chrome://inspect খুলুন
```

### PHP মেমোরি প্রোফাইলিং:

```php
<?php
// মেমোরি ব্যবহারের রিপোর্ট
function memoryReport(string $label): void {
    $usage = memory_get_usage(true);
    $peak = memory_get_peak_usage(true);
    $limit = ini_get('memory_limit');

    echo "=== [$label] ===\n";
    echo "বর্তমান: " . formatBytes($usage) . "\n";
    echo "সর্বোচ্চ: " . formatBytes($peak) . "\n";
    echo "সীমা:    $limit\n";
    echo "================\n";
}

function formatBytes(int $bytes): string {
    $units = ['B', 'KB', 'MB', 'GB'];
    $i = 0;
    while ($bytes >= 1024 && $i < count($units) - 1) {
        $bytes /= 1024;
        $i++;
    }
    return round($bytes, 2) . ' ' . $units[$i];
}

memoryReport('API শুরু');
$data = fetchLargeDataset();   // Daraz product catalog
memoryReport('ডেটা লোড');
processData($data);
unset($data);
memoryReport('প্রসেস শেষ');
```

---

## ⚠️ সাধারণ মেমোরি লিক প্যাটার্ন

### 1️⃣ ভুলে যাওয়া Event Listeners:

```javascript
// ❌ খারাপ — listener কখনো সরানো হচ্ছে না
class DarazProductPage {
    constructor() {
        this.data = new Array(100000).fill('product-info');
        window.addEventListener('scroll', this.onScroll);
        window.addEventListener('resize', this.onResize);
    }

    onScroll = () => { /* ... */ };
    onResize = () => { /* ... */ };

    // Page বদলালে এই instance GC হবে না
    // কারণ window এখনো listener ধরে রেখেছে!
}

// ✅ ভালো — cleanup method আছে
class DarazProductPage {
    constructor() {
        this.data = new Array(100000).fill('product-info');
        this.onScroll = this.onScroll.bind(this);
        this.onResize = this.onResize.bind(this);
        window.addEventListener('scroll', this.onScroll);
        window.addEventListener('resize', this.onResize);
    }

    onScroll() { /* ... */ }
    onResize() { /* ... */ }

    destroy() {
        // পরিষ্কারভাবে listener সরিয়ে ফেলুন
        window.removeEventListener('scroll', this.onScroll);
        window.removeEventListener('resize', this.onResize);
        this.data = null;
    }
}
```

### 2️⃣ Closure বড় অবজেক্ট ধরে রাখছে:

```javascript
// ❌ খারাপ — closure বিশাল array ধরে রাখছে
function processPathaoRides() {
    const allRides = new Array(1000000).fill({ from: 'Dhanmondi', to: 'Gulshan' });

    return function getCount() {
        // শুধু length দরকার, কিন্তু পুরো allRides ধরে আছে!
        return allRides.length;
    };
}

const counter = processPathaoRides(); // 1M rides মেমোরিতে আটকে আছে

// ✅ ভালো — শুধু যা দরকার তা রাখুন
function processPathaoRides() {
    const allRides = new Array(1000000).fill({ from: 'Dhanmondi', to: 'Gulshan' });
    const count = allRides.length; // শুধু count নিয়ে রাখুন

    return function getCount() {
        return count; // allRides এখন GC হতে পারবে
    };
}
```

### 3️⃣ Global Variable জমে যাওয়া:

```javascript
// ❌ খারাপ — global এ data জমছে
const userSessions = {};

function handleRequest(userId, data) {
    // প্রতিটি request এ data যোগ হচ্ছে, কিন্তু কখনো মুছছে না
    if (!userSessions[userId]) {
        userSessions[userId] = [];
    }
    userSessions[userId].push(data);
}

// ✅ ভালো — TTL সহ session management
class SessionManager {
    constructor(ttlMs = 30 * 60 * 1000) { // 30 মিনিট
        this.sessions = new Map();
        this.ttl = ttlMs;

        // পর্যায়ক্রমে মেয়াদোত্তীর্ণ session পরিষ্কার করো
        setInterval(() => this.cleanup(), 60 * 1000);
    }

    set(userId, data) {
        this.sessions.set(userId, {
            data,
            expiresAt: Date.now() + this.ttl
        });
    }

    get(userId) {
        const session = this.sessions.get(userId);
        if (!session || session.expiresAt < Date.now()) {
            this.sessions.delete(userId);
            return null;
        }
        return session.data;
    }

    cleanup() {
        const now = Date.now();
        for (const [key, session] of this.sessions) {
            if (session.expiresAt < now) {
                this.sessions.delete(key);
            }
        }
    }
}
```

### 4️⃣ বন্ধ না করা Database Connection:

```php
<?php
// ❌ খারাপ — connection কখনো বন্ধ হচ্ছে না
function getUsers(): array {
    $conn = new mysqli('localhost', 'root', '', 'bkash_db');
    $result = $conn->query("SELECT * FROM users");
    $users = $result->fetch_all(MYSQLI_ASSOC);
    // $conn বন্ধ করা হয়নি!
    return $users;
}

// লুপে কল করলে connection জমতে থাকবে
for ($i = 0; $i < 1000; $i++) {
    $users = getUsers(); // 1000 টি open connection!
}

// ✅ ভালো — try/finally দিয়ে নিশ্চিত করুন connection বন্ধ হবে
function getUsers(): array {
    $conn = new mysqli('localhost', 'root', '', 'bkash_db');
    try {
        $result = $conn->query("SELECT * FROM users");
        return $result->fetch_all(MYSQLI_ASSOC);
    } finally {
        $conn->close(); // সবসময় বন্ধ হবে
    }
}

// ✅ আরও ভালো — Connection Pool ব্যবহার করুন
// PDO persistent connection
$pdo = new PDO(
    'mysql:host=localhost;dbname=bkash_db',
    'root', '',
    [PDO::ATTR_PERSISTENT => true]
);
```

### 5️⃣ PHP Circular Reference (পুনরায়):

```php
<?php
// ❌ খারাপ — circular reference সহ long-running script
while (true) {
    $request = new Request();
    $response = new Response();
    $request->response = $response;
    $response->request = $request; // circular!

    processRequest($request);
    // unset করলেও মেমোরি ফ্রি হবে না
    unset($request, $response);
    // প্রতিটি iteration এ মেমোরি বাড়তে থাকবে
}

// ✅ ভালো — gc_collect_cycles() পর্যায়ক্রমে চালান
$iteration = 0;
while (true) {
    $request = new Request();
    $response = new Response();
    $request->response = $response;
    $response->request = $request;

    processRequest($request);
    unset($request, $response);

    if (++$iteration % 100 === 0) {
        gc_collect_cycles(); // প্রতি 100 iteration এ GC
    }
}
```

---

## ⚙️ কনফিগারেশন ও টিউনিং

### PHP memory_limit:

```php
<?php
// php.ini এ সেট করুন
// memory_limit = 256M

// অথবা রানটাইমে:
ini_set('memory_limit', '512M');

// বর্তমান সীমা দেখুন:
echo ini_get('memory_limit'); // "256M"
```

```
কখন memory_limit বাড়াবেন vs কোড ঠিক করবেন:

┌──────────────────────────────────────────────────┐
│                                                    │
│  memory_limit বাড়ান যখন:                           │
│  ├── সত্যিই বড় ডেটাসেট প্রসেস করতে হবে             │
│  ├── CSV import / image processing                │
│  └── একবারের batch job                             │
│                                                    │
│  কোড ঠিক করুন যখন:                                 │
│  ├── মেমোরি ক্রমাগত বাড়তেই থাকে (লিক)              │
│  ├── ছোট request এও বেশি মেমোরি লাগে               │
│  ├── Generator / streaming ব্যবহার করা যায়          │
│  └── অপ্রয়োজনীয় ডেটা মেমোরিতে রাখা হচ্ছে          │
│                                                    │
└──────────────────────────────────────────────────┘
```

### Node.js Heap Size:

```bash
# Default heap size: ~1.5 GB (64-bit)
# বাড়াতে চাইলে:
node --max-old-space-size=4096 app.js    # 4 GB

# Young generation size:
node --max-semi-space-size=64 app.js     # 64 MB

# GC logging চালু করুন (ডিবাগ এর জন্য):
node --trace-gc app.js

# Heap snapshot তৈরি করুন:
node --heapsnapshot-signal=SIGUSR2 app.js
# তারপর: kill -USR2 <pid>
```

### Node.js প্রোডাকশন কনফিগারেশন উদাহরণ:

```javascript
// Grameenphone API server — production config
// ecosystem.config.js (PM2)
module.exports = {
    apps: [{
        name: 'gp-api',
        script: 'dist/server.js',
        node_args: '--max-old-space-size=2048',
        instances: 4,             // 4 টি instance
        exec_mode: 'cluster',
        max_memory_restart: '1G', // 1GB অতিক্রম করলে restart
        env: {
            NODE_ENV: 'production'
        }
    }]
};
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন মেমোরি অপটিমাইজেশন করবেন:

| পরিস্থিতি | কৌশল |
|-----------|-------|
| বড় ফাইল প্রসেসিং (CSV, log) | Generator / streaming ব্যবহার করুন |
| Long-running process (daemon, queue worker) | পর্যায়ক্রমে GC ও মেমোরি মনিটরিং |
| ক্যাশিং ডেটা | Bounded cache + TTL |
| Event-driven architecture | Listener cleanup নিশ্চিত করুন |
| বড় API response | Pagination / cursor-based loading |
| Image / file processing | Stream-based processing |
| Database bulk operations | Chunked processing |

### ❌ কখন অপটিমাইজেশনে সময় নষ্ট করবেন না:

| পরিস্থিতি | কারণ |
|-----------|-------|
| ছোট script / CLI tool | PHP request শেষে মেমোরি ফ্রি হয় |
| অল্প ডেটা নিয়ে কাজ | অপটিমাইজেশনের সুবিধা নগণ্য |
| Premature optimization | "প্রথমে কাজ করাও, তারপর দ্রুত করো" |
| মেমোরি সমস্যা প্রমাণিত নয় | প্রোফাইলিং ছাড়া অনুমান করবেন না |

### 📊 সিদ্ধান্ত ফ্লোচার্ট:

```
মেমোরি সমস্যা আছে?
│
├── না → 🛑 কিছু করবেন না, অন্য কাজে মনোযোগ দিন
│
└── হ্যাঁ → প্রোফাইলিং করুন (DevTools / memory_get_usage)
            │
            ├── মেমোরি ক্রমাগত বাড়ছে?
            │   ├── হ্যাঁ → মেমোরি লিক! → লিক প্যাটার্ন চেক করুন
            │   │          └── Event listener? Closure? Global? Circular ref?
            │   │
            │   └── না → একবারে বড় allocation হচ্ছে
            │            └── Streaming / Generator / Chunking ব্যবহার করুন
            │
            └── কোন পদ্ধতিতেও কমছে না?
                └── memory_limit / --max-old-space-size বাড়ান
```

---

## 🏠 বাস্তব উদাহরণ — বাংলাদেশ

### bKash Transaction Processing:

```javascript
// bKash-এর মতো সিস্টেমে প্রতি সেকেন্ডে হাজার হাজার transaction আসে
// মেমোরি ম্যানেজমেন্ট অত্যন্ত গুরুত্বপূর্ণ

class TransactionProcessor {
    constructor() {
        // ✅ Bounded queue — সীমিত আকারের
        this.pendingQueue = [];
        this.maxQueueSize = 10000;

        // ✅ WeakMap দিয়ে metadata — GC friendly
        this.metadata = new WeakMap();
    }

    async processTransaction(txn) {
        if (this.pendingQueue.length >= this.maxQueueSize) {
            // Queue ভর্তি হলে পুরনোটি ফেলে দাও
            const dropped = this.pendingQueue.shift();
            console.warn(`Queue full, dropped: ${dropped.id}`);
        }

        this.pendingQueue.push(txn);

        try {
            const result = await this.execute(txn);
            return result;
        } finally {
            // প্রসেস শেষে queue থেকে সরাও
            const idx = this.pendingQueue.indexOf(txn);
            if (idx > -1) this.pendingQueue.splice(idx, 1);
        }
    }
}
```

### Pathao Ride Matching — Streaming Data:

```php
<?php
// Pathao-তে rider location ক্রমাগত আপডেট হয়
// সব location মেমোরিতে রাখলে server ক্র্যাশ করবে

// ❌ খারাপ — সব rider মেমোরিতে
$riders = Rider::all(); // 50,000 rider = বিশাল মেমোরি!

// ✅ ভালো — chunk করে প্রসেস
Rider::where('is_active', true)
    ->chunk(500, function ($riders) {
        foreach ($riders as $rider) {
            updateRiderLocation($rider);
        }
    }); // একবারে মাত্র 500 rider মেমোরিতে

// ✅ আরও ভালো — Lazy Collection (Laravel)
Rider::where('is_active', true)
    ->lazy()
    ->each(function ($rider) {
        updateRiderLocation($rider);
    }); // একবারে একটি করে, মেমোরি ব্যবহার সর্বনিম্ন
```

---

## 📊 মেমোরি ম্যানেজমেন্ট চেকলিস্ট

```
প্রোডাকশন ডিপ্লয়ের আগে চেক করুন:

□  মেমোরি লিক টেস্ট করা হয়েছে?
□  Event listeners সঠিকভাবে cleanup হচ্ছে?
□  Database connections বন্ধ হচ্ছে?
□  বড় ডেটাসেট streaming/chunking এ প্রসেস হচ্ছে?
□  Cache এ TTL এবং size limit আছে?
□  Circular references চেক করা হয়েছে?
□  memory_limit / --max-old-space-size সঠিকভাবে সেট?
□  মেমোরি মনিটরিং / alerting চালু আছে?
□  Long-running process এ periodic GC আছে?
□  Load test এ মেমোরি stable থাকছে?
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| Object Pool | অবজেক্ট পুনঃব্যবহার করে allocation কমায় |
| Flyweight Pattern | শেয়ার্ড state দিয়ে মেমোরি বাঁচায় |
| Iterator Pattern | Generator / lazy loading এর ভিত্তি |
| Observer Pattern | Event listener cleanup এর সাথে সম্পর্কিত |
| Singleton Pattern | একটি instance = কম মেমোরি, কিন্তু সাবধান |

---

> **মনে রাখবেন:** "Premature optimization is the root of all evil" — Donald Knuth।
> প্রথমে প্রোফাইল করুন, তারপর অপটিমাইজ করুন। অনুমান ভিত্তিক
> অপটিমাইজেশন প্রায়ই ভুল জায়গায় সময় নষ্ট করে। 📈
