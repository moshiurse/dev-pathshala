# 🗂️ হ্যাশ টেবিল (Hash Table) — সম্পূর্ণ বাংলা গাইড

> **ডেটা স্ট্রাকচার সিরিজ** | পূর্ববর্তী: [sorting.md](./sorting.md)

---

## 📑 সূচিপত্র

1. [হ্যাশ টেবিলের মৌলিক ধারণা](#১-হ্যাশ-টেবিলের-মৌলিক-ধারণা)
2. [হ্যাশ ফাংশন](#২-হ্যাশ-ফাংশন)
3. [কলিশন ও কলিশন হ্যান্ডলিং](#৩-কলিশন-ও-কলিশন-হ্যান্ডলিং)
4. [অপারেশন ও কমপ্লেক্সিটি](#৪-অপারেশন-ও-কমপ্লেক্সিটি)
5. [ভেরিয়েশন ও বিল্ট-ইন স্ট্রাকচার](#৫-ভেরিয়েশন-ও-বিল্ট-ইন-স্ট্রাকচার)
6. [জটিল সমস্যা ও সমাধান](#৬-জটিল-সমস্যা-ও-সমাধান)
7. [প্রজেক্ট: সিম্পল ডেটাবেস ইনডেক্স](#৭-প্রজেক্ট-সিম্পল-ডেটাবেস-ইনডেক্স)
8. [চিটশিট ও ইন্টারভিউ টিপস](#৮-চিটশিট-ও-ইন্টারভিউ-টিপস)

---

## ১. হ্যাশ টেবিলের মৌলিক ধারণা

### হ্যাশ টেবিল কী?

হ্যাশ টেবিল হলো এমন একটি ডেটা স্ট্রাকচার যেখানে **কী-ভ্যালু জোড়া (key-value pair)** সংরক্ষণ করা হয়।
একটি **হ্যাশ ফাংশন** কী (key) কে একটি ইনডেক্সে রূপান্তর করে, এবং সেই ইনডেক্সে ভ্যালু সংরক্ষিত হয়।

### কেন হ্যাশ টেবিল দরকার?

- অ্যারেতে সার্চ করতে O(n) সময় লাগে
- সর্টেড অ্যারেতে বাইনারি সার্চে O(log n) লাগে
- কিন্তু হ্যাশ টেবিলে **গড়ে O(1)** সময়ে ডেটা খুঁজে পাওয়া যায়!

### মূল ধারণার ASCII চিত্র

```
    কী (Key)          হ্যাশ ফাংশন          ইনডেক্স        অ্যারে (Bucket)
  ┌──────────┐      ┌──────────────┐      ┌───┐       ┌──────────────────┐
  │ "নাম"    │ ──▶  │  h("নাম")    │ ──▶  │ 2 │  ──▶  │ [2] = "রহিম"    │
  └──────────┘      └──────────────┘      └───┘       └──────────────────┘
  ┌──────────┐      ┌──────────────┐      ┌───┐       ┌──────────────────┐
  │ "বয়স"   │ ──▶  │  h("বয়স")   │ ──▶  │ 5 │  ──▶  │ [5] = 25         │
  └──────────┘      └──────────────┘      └───┘       └──────────────────┘
  ┌──────────┐      ┌──────────────┐      ┌───┐       ┌──────────────────┐
  │ "শহর"   │ ──▶  │  h("শহর")   │ ──▶  │ 0 │  ──▶  │ [0] = "ঢাকা"    │
  └──────────┘      └──────────────┘      └───┘       └──────────────────┘
```

### কী-ভ্যালু জোড়ার উদাহরণ

```
  ┌─────────────────────────────────────────────┐
  │         হ্যাশ টেবিল (আকার = ৭)             │
  ├─────────┬───────────────────────────────────┤
  │ ইনডেক্স │           ডেটা                    │
  ├─────────┼───────────────────────────────────┤
  │    0    │  "শহর" → "ঢাকা"                  │
  │    1    │  (খালি)                           │
  │    2    │  "নাম" → "রহিম"                   │
  │    3    │  (খালি)                           │
  │    4    │  "ফোন" → "০১৭১২৩৪৫৬৭৮"          │
  │    5    │  "বয়স" → ২৫                       │
  │    6    │  (খালি)                           │
  └─────────┴───────────────────────────────────┘
```

### PHP তে সরল উদাহরণ

```php
<?php
// হ্যাশ টেবিলের মৌলিক ব্যবহার — PHP এর অ্যাসোসিয়েটিভ অ্যারে

// কী-ভ্যালু জোড়া তৈরি
$ছাত্র = [
    "নাম"  => "রহিম",
    "বয়স"  => 25,
    "শহর"  => "ঢাকা",
    "ফোন"  => "০১৭১২৩৪৫৬৭৮"
];

// ভ্যালু অ্যাক্সেস — O(1) গড় সময়
echo $ছাত্র["নাম"];   // আউটপুট: রহিম
echo $ছাত্র["বয়স"];   // আউটপুট: 25

// কী আছে কিনা পরীক্ষা
if (isset($ছাত্র["শহর"])) {
    echo "শহর পাওয়া গেছে: " . $ছাত্র["শহর"];
}

// নতুন কী-ভ্যালু যোগ
$ছাত্র["বিভাগ"] = "কম্পিউটার সায়েন্স";

// কী মুছে ফেলা
unset($ছাত্র["ফোন"]);
?>
```

### JavaScript এ সরল উদাহরণ

```javascript
// হ্যাশ টেবিলের মৌলিক ব্যবহার — JS এর Map ব্যবহার করে

// Map তৈরি (হ্যাশ টেবিলের বিল্ট-ইন রূপ)
const ছাত্র = new Map();

// কী-ভ্যালু জোড়া সেট করা
ছাত্র.set("নাম", "রহিম");
ছাত্র.set("বয়স", 25);
ছাত্র.set("শহর", "ঢাকা");

// ভ্যালু পড়া — O(1) গড় সময়
console.log(ছাত্র.get("নাম"));  // আউটপুট: রহিম

// কী আছে কিনা যাচাই
if (ছাত্র.has("শহর")) {
    console.log("শহর পাওয়া গেছে:", ছাত্র.get("শহর"));
}

// কী মুছে ফেলা
ছাত্র.delete("শহর");

// মোট এন্ট্রি সংখ্যা
console.log("মোট:", ছাত্র.size); // আউটপুট: 2
```

---

## ২. হ্যাশ ফাংশন

হ্যাশ ফাংশন হলো এমন একটি ফাংশন যা যেকোনো আকারের ইনপুটকে একটি **নির্দিষ্ট পরিসরের সংখ্যায়** রূপান্তর করে।

### ভালো হ্যাশ ফাংশনের বৈশিষ্ট্য

1. **ডিটারমিনিস্টিক**: একই ইনপুটে সবসময় একই আউটপুট
2. **ইউনিফর্ম ডিস্ট্রিবিউশন**: বিভিন্ন ইনডেক্সে সমানভাবে বিতরণ
3. **দ্রুত গণনা**: O(1) সময়ে হ্যাশ ভ্যালু বের করা
4. **কম কলিশন**: ভিন্ন ভিন্ন কী যেন একই ইনডেক্সে না পড়ে

### ২.১ ডিভিশন মেথড

সবচেয়ে সরল পদ্ধতি: `h(k) = k mod m` (যেখানে m = টেবিলের আকার)

```
  h(k) = k % m

  উদাহরণ: m = 7 (টেবিলের আকার)
  h(10) = 10 % 7 = 3
  h(22) = 22 % 7 = 1
  h(31) = 31 % 7 = 3  ← কলিশন! (10 এর সাথে)
  h(4)  =  4 % 7 = 4
  h(15) = 15 % 7 = 1  ← কলিশন! (22 এর সাথে)
```

```php
<?php
// ডিভিশন মেথড হ্যাশ ফাংশন
function divisionHash($key, $tableSize) {
    // কী কে টেবিলের আকার দিয়ে ভাগশেষ বের করি
    return $key % $tableSize;
}

// উদাহরণ — টেবিলের আকার ৭
$আকার = 7;
echo divisionHash(10, $আকার); // আউটপুট: 3
echo divisionHash(22, $আকার); // আউটপুট: 1
echo divisionHash(31, $আকার); // আউটপুট: 3 (কলিশন!)
?>
```

```javascript
// ডিভিশন মেথড হ্যাশ ফাংশন
function divisionHash(key, tableSize) {
    // কী কে টেবিলের আকার দিয়ে মড করি
    return key % tableSize;
}

// উদাহরণ
console.log(divisionHash(10, 7)); // 3
console.log(divisionHash(22, 7)); // 1
console.log(divisionHash(31, 7)); // 3 (কলিশন!)
```

> 💡 **টিপ**: m এর মান মৌলিক সংখ্যা (prime) হলে কলিশন কম হয়। ২ এর পাওয়ার এড়িয়ে চলুন।

### ২.২ মাল্টিপ্লিকেশন মেথড

`h(k) = floor(m * (k * A mod 1))` — যেখানে A হলো ০ থেকে ১ এর মধ্যে একটি ধ্রুবক।

ন্যুথ (Knuth) এর প্রস্তাবিত মান: **A ≈ 0.6180339887** (সুবর্ণ অনুপাতের বিপরীত)

```php
<?php
// মাল্টিপ্লিকেশন মেথড হ্যাশ ফাংশন
function multiplicationHash($key, $tableSize) {
    // সুবর্ণ অনুপাতের বিপরীত ধ্রুবক
    $A = 0.6180339887;
    // k * A এর দশমিক অংশ নিয়ে m দিয়ে গুণ করি
    return floor($tableSize * fmod($key * $A, 1));
}

echo multiplicationHash(10, 7);  // ইনডেক্স বের হবে
echo multiplicationHash(22, 7);
echo multiplicationHash(31, 7);
?>
```

### ২.৩ স্ট্রিং হ্যাশিং

স্ট্রিং কী এর জন্য প্রতিটি অক্ষরের ASCII মান ব্যবহার করে হ্যাশ তৈরি করা হয়।

```
  "cat" → 'c'=99, 'a'=97, 't'=116
  সরল যোগফল: 99 + 97 + 116 = 312
  h("cat") = 312 % 7 = 4
```

```php
<?php
// সরল স্ট্রিং হ্যাশ — অক্ষরগুলোর ASCII মান যোগ করে
function simpleStringHash($str, $tableSize) {
    $hash = 0;
    $len = strlen($str);
    for ($i = 0; $i < $len; $i++) {
        // প্রতিটি অক্ষরের ASCII মান যোগ করি
        $hash += ord($str[$i]);
    }
    // টেবিলের আকার দিয়ে মড করি
    return $hash % $tableSize;
}

echo simpleStringHash("cat", 7);  // 4
echo simpleStringHash("act", 7);  // 4 ← কলিশন! (অক্ষর একই, ক্রম ভিন্ন)
?>
```

### ২.৪ পলিনোমিয়াল রোলিং হ্যাশ

অক্ষরের অবস্থান (position) গুরুত্বপূর্ণ করে তোলে — "cat" এবং "act" ভিন্ন হ্যাশ দেবে।

```
  h(s) = (s[0]*p^0 + s[1]*p^1 + s[2]*p^2 + ...) mod m

  যেখানে:
    p = মৌলিক সংখ্যা (সাধারণত 31 বা 37)
    m = বড় মৌলিক সংখ্যা (যেমন 10^9 + 7)
```

```php
<?php
// পলিনোমিয়াল রোলিং হ্যাশ — অবস্থান-সচেতন হ্যাশিং
function polynomialHash($str, $tableSize) {
    $hash = 0;
    $p = 31;        // ভিত্তি মৌলিক সংখ্যা
    $m = 1000000007; // বড় মৌলিক সংখ্যা (ওভারফ্লো রোধে)
    $pPow = 1;       // p এর ক্রমবর্ধমান পাওয়ার

    $len = strlen($str);
    for ($i = 0; $i < $len; $i++) {
        // প্রতিটি অক্ষরকে তার অবস্থান অনুযায়ী ওজন দিই
        $hash = ($hash + (ord($str[$i]) - ord('a') + 1) * $pPow) % $m;
        $pPow = ($pPow * $p) % $m;
    }

    return $hash % $tableSize;
}

echo polynomialHash("cat", 7); // "cat" ও "act" এখন ভিন্ন হ্যাশ দেবে
echo polynomialHash("act", 7);
?>
```

```javascript
// পলিনোমিয়াল রোলিং হ্যাশ — JavaScript বাস্তবায়ন
function polynomialHash(str, tableSize) {
    let hash = 0;
    const p = 31;           // ভিত্তি মৌলিক সংখ্যা
    const m = 1000000007;   // বড় মৌলিক সংখ্যা
    let pPow = 1;

    for (let i = 0; i < str.length; i++) {
        // অক্ষরের মান × অবস্থানের ওজন
        hash = (hash + (str.charCodeAt(i) - 96) * pPow) % m;
        pPow = (pPow * p) % m;
    }

    return hash % tableSize;
}

console.log(polynomialHash("cat", 7));
console.log(polynomialHash("act", 7)); // ভিন্ন ফলাফল!
```

---

## ৩. কলিশন ও কলিশন হ্যান্ডলিং

### কলিশন কী?

যখন দুটি ভিন্ন কী একই ইনডেক্সে হ্যাশ হয়, তখন **কলিশন** ঘটে।

```
  কী "cat" ──▶ h("cat") = 4 ──┐
                                ├──▶ ইনডেক্স 4 ← সংঘর্ষ!
  কী "act" ──▶ h("act") = 4 ──┘
```

### ৩.১ চেইনিং (Separate Chaining)

প্রতিটি ইনডেক্সে একটি **লিংকড লিস্ট** রাখা হয়। কলিশন হলে নতুন আইটেম লিস্টে যোগ হয়।

```
  ইনডেক্স    চেইন (লিংকড লিস্ট)
  ┌───┐
  │ 0 │ → [("শহর","ঢাকা")] → NULL
  ├───┤
  │ 1 │ → [("বয়স",22)] → [("ফোন","০১৭...")] → NULL
  ├───┤
  │ 2 │ → NULL (খালি)
  ├───┤
  │ 3 │ → [("নাম","রহিম")] → [("পেশা","ছাত্র")] → NULL
  ├───┤
  │ 4 │ → [("ঠিকানা","মিরপুর")] → NULL
  ├───┤
  │ 5 │ → NULL (খালি)
  ├───┤
  │ 6 │ → [("ইমেইল","r@g.com")] → NULL
  └───┘
```

```php
<?php
// চেইনিং পদ্ধতিতে হ্যাশ টেবিল বাস্তবায়ন
class ChainingHashTable {
    private $buckets;   // বাকেট অ্যারে
    private $size;      // টেবিলের আকার
    private $count;     // মোট আইটেম সংখ্যা

    public function __construct($size = 7) {
        $this->size = $size;
        $this->count = 0;
        // প্রতিটি বাকেটে খালি অ্যারে রাখি (চেইনের জন্য)
        $this->buckets = array_fill(0, $size, []);
    }

    // হ্যাশ ফাংশন — কী থেকে ইনডেক্স বের করে
    private function hash($key) {
        $hash = 0;
        $len = strlen((string)$key);
        for ($i = 0; $i < $len; $i++) {
            $hash = ($hash * 31 + ord(((string)$key)[$i])) % $this->size;
        }
        return $hash;
    }

    // ইনসার্ট — নতুন কী-ভ্যালু যোগ বা আপডেট
    public function put($key, $value) {
        $index = $this->hash($key);

        // চেইনে আগে থেকে কী আছে কিনা দেখি
        foreach ($this->buckets[$index] as &$pair) {
            if ($pair[0] === $key) {
                // কী পাওয়া গেলে ভ্যালু আপডেট করি
                $pair[1] = $value;
                return;
            }
        }

        // নতুন কী-ভ্যালু চেইনে যোগ করি
        $this->buckets[$index][] = [$key, $value];
        $this->count++;
    }

    // সার্চ — কী দিয়ে ভ্যালু খোঁজা
    public function get($key) {
        $index = $this->hash($key);

        // চেইনে কী খুঁজি
        foreach ($this->buckets[$index] as $pair) {
            if ($pair[0] === $key) {
                return $pair[1]; // পেলে ভ্যালু ফেরত দিই
            }
        }

        return null; // না পেলে null ফেরত
    }

    // ডিলিট — কী মুছে ফেলা
    public function delete($key) {
        $index = $this->hash($key);

        foreach ($this->buckets[$index] as $i => $pair) {
            if ($pair[0] === $key) {
                // চেইন থেকে আইটেম সরাই
                array_splice($this->buckets[$index], $i, 1);
                $this->count--;
                return true;
            }
        }

        return false; // কী পাওয়া যায়নি
    }

    // লোড ফ্যাক্টর — চেইনের গড় দৈর্ঘ্য
    public function loadFactor() {
        return $this->count / $this->size;
    }
}

// ব্যবহার
$ht = new ChainingHashTable(7);
$ht->put("নাম", "রহিম");
$ht->put("বয়স", 25);
$ht->put("শহর", "ঢাকা");

echo $ht->get("নাম");  // আউটপুট: রহিম
echo $ht->get("বয়স");  // আউটপুট: 25

$ht->delete("শহর");
echo $ht->get("শহর");   // আউটপুট: null
?>
```

```javascript
// চেইনিং পদ্ধতিতে হ্যাশ টেবিল — JavaScript বাস্তবায়ন
class ChainingHashTable {
    constructor(size = 7) {
        this.size = size;
        this.count = 0;
        // প্রতিটি বাকেটে খালি অ্যারে রাখি
        this.buckets = new Array(size).fill(null).map(() => []);
    }

    // হ্যাশ ফাংশন
    _hash(key) {
        let hash = 0;
        const str = String(key);
        for (let i = 0; i < str.length; i++) {
            hash = (hash * 31 + str.charCodeAt(i)) % this.size;
        }
        return hash;
    }

    // কী-ভ্যালু জোড়া ইনসার্ট বা আপডেট
    put(key, value) {
        const index = this._hash(key);
        const chain = this.buckets[index];

        // চেইনে কী আগে থেকে আছে কিনা দেখি
        for (let i = 0; i < chain.length; i++) {
            if (chain[i][0] === key) {
                chain[i][1] = value; // আপডেট
                return;
            }
        }

        // নতুন এন্ট্রি যোগ
        chain.push([key, value]);
        this.count++;
    }

    // কী দিয়ে ভ্যালু খোঁজা
    get(key) {
        const index = this._hash(key);
        const chain = this.buckets[index];

        for (const [k, v] of chain) {
            if (k === key) return v; // পেলে ফেরত দিই
        }

        return undefined; // না পেলে undefined
    }

    // কী মুছে ফেলা
    delete(key) {
        const index = this._hash(key);
        const chain = this.buckets[index];

        for (let i = 0; i < chain.length; i++) {
            if (chain[i][0] === key) {
                chain.splice(i, 1); // চেইন থেকে সরাই
                this.count--;
                return true;
            }
        }
        return false;
    }
}

// ব্যবহার
const ht = new ChainingHashTable(7);
ht.put("নাম", "রহিম");
ht.put("বয়স", 25);
console.log(ht.get("নাম"));  // রহিম
ht.delete("বয়স");
console.log(ht.get("বয়স"));  // undefined
```

### ৩.২ ওপেন অ্যাড্রেসিং (Open Addressing)

কলিশন হলে **অন্য একটি খালি স্লটে** ডেটা রাখা হয়। তিনটি প্রধান পদ্ধতি:

#### ক) লিনিয়ার প্রোবিং (Linear Probing)

পরের ঘরে চেষ্টা করা: `h(k, i) = (h(k) + i) mod m`

```
  ইনসার্ট ক্রম: 10, 22, 31, 4, 15  (m=7)

  h(10) = 10%7 = 3  → স্লট[3] ← 10
  h(22) = 22%7 = 1  → স্লট[1] ← 22
  h(31) = 31%7 = 3  → কলিশন! → চেষ্টা 4 → স্লট[4] ← 31
  h(4)  =  4%7 = 4  → কলিশন! → চেষ্টা 5 → স্লট[5] ← 4
  h(15) = 15%7 = 1  → কলিশন! → চেষ্টা 2 → স্লট[2] ← 15

  ┌───┬────┬────┬────┬────┬────┬────┐
  │ 0 │  1 │  2 │  3 │  4 │  5 │  6 │
  ├───┼────┼────┼────┼────┼────┼────┤
  │   │ 22 │ 15 │ 10 │ 31 │  4 │    │
  └───┴────┴────┴────┴────┴────┴────┘
```

> ⚠️ **ক্লাস্টারিং সমস্যা**: পরপর ভরা স্লট তৈরি হলে সার্চ ধীর হয়ে যায়।

#### খ) কোয়াড্রাটিক প্রোবিং (Quadratic Probing)

বর্গাকার ধাপে এগিয়ে যাওয়া: `h(k, i) = (h(k) + i²) mod m`

```
  কলিশন হলে চেষ্টা:
  ১ম: h(k) + 1² = h(k) + 1
  ২য়: h(k) + 2² = h(k) + 4
  ৩য়: h(k) + 3² = h(k) + 9
  ...

  এতে ক্লাস্টারিং কমে কিন্তু সম্পূর্ণ দূর হয় না।
```

#### গ) ডাবল হ্যাশিং (Double Hashing)

দ্বিতীয় হ্যাশ ফাংশন দিয়ে ধাপের আকার নির্ধারণ: `h(k, i) = (h₁(k) + i * h₂(k)) mod m`

```
  h₁(k) = k mod 7          (প্রাথমিক হ্যাশ)
  h₂(k) = 5 - (k mod 5)    (ধাপের আকার)

  k=10: h₁=3, h₂=5  → 3, 8%7=1, 6, 4, ...
  k=31: h₁=3, h₂=4  → 3(ভরা!), 7%7=0, 4, ...
```

```php
<?php
// ওপেন অ্যাড্রেসিং হ্যাশ টেবিল — লিনিয়ার প্রোবিং
class OpenAddressingHashTable {
    private $keys;      // কী রাখার অ্যারে
    private $values;    // ভ্যালু রাখার অ্যারে
    private $size;      // টেবিলের আকার
    private $count;     // মোট আইটেম
    private $deleted;   // মুছে ফেলা চিহ্ন

    public function __construct($size = 7) {
        $this->size = $size;
        $this->count = 0;
        $this->keys = array_fill(0, $size, null);
        $this->values = array_fill(0, $size, null);
        $this->deleted = array_fill(0, $size, false);
    }

    private function hash($key) {
        $hash = 0;
        $str = (string)$key;
        for ($i = 0; $i < strlen($str); $i++) {
            $hash = ($hash * 31 + ord($str[$i])) % $this->size;
        }
        return $hash;
    }

    // দ্বিতীয় হ্যাশ — ডাবল হ্যাশিং এর জন্য
    private function hash2($key) {
        $hash = 0;
        $str = (string)$key;
        for ($i = 0; $i < strlen($str); $i++) {
            $hash = ($hash * 37 + ord($str[$i]));
        }
        // মৌলিক সংখ্যা ব্যবহার করি
        return 5 - ($hash % 5);
    }

    // ইনসার্ট — লিনিয়ার প্রোবিং সহ
    public function put($key, $value) {
        // লোড ফ্যাক্টর ০.৭ এর বেশি হলে রিসাইজ করি
        if ($this->count >= $this->size * 0.7) {
            $this->resize();
        }

        $index = $this->hash($key);

        for ($i = 0; $i < $this->size; $i++) {
            // লিনিয়ার প্রোবিং: পরের ঘরে চেষ্টা
            $probeIndex = ($index + $i) % $this->size;

            // খালি ঘর বা মুছে ফেলা ঘর পেলে সেখানে রাখি
            if ($this->keys[$probeIndex] === null || $this->deleted[$probeIndex]) {
                $this->keys[$probeIndex] = $key;
                $this->values[$probeIndex] = $value;
                $this->deleted[$probeIndex] = false;
                $this->count++;
                return;
            }

            // একই কী পেলে ভ্যালু আপডেট করি
            if ($this->keys[$probeIndex] === $key) {
                $this->values[$probeIndex] = $value;
                return;
            }
        }
    }

    // সার্চ
    public function get($key) {
        $index = $this->hash($key);

        for ($i = 0; $i < $this->size; $i++) {
            $probeIndex = ($index + $i) % $this->size;

            // সম্পূর্ণ খালি ঘর — কী নেই
            if ($this->keys[$probeIndex] === null && !$this->deleted[$probeIndex]) {
                return null;
            }

            // কী মিললো
            if ($this->keys[$probeIndex] === $key && !$this->deleted[$probeIndex]) {
                return $this->values[$probeIndex];
            }
        }

        return null;
    }

    // ডিলিট — সফট ডিলিট (মুছে ফেলা চিহ্ন রাখি)
    public function delete($key) {
        $index = $this->hash($key);

        for ($i = 0; $i < $this->size; $i++) {
            $probeIndex = ($index + $i) % $this->size;

            if ($this->keys[$probeIndex] === null && !$this->deleted[$probeIndex]) {
                return false;
            }

            if ($this->keys[$probeIndex] === $key && !$this->deleted[$probeIndex]) {
                // সফট ডিলিট — চিহ্ন রাখি যাতে প্রোবিং চেইন না ভাঙে
                $this->deleted[$probeIndex] = true;
                $this->count--;
                return true;
            }
        }

        return false;
    }

    // রিসাইজ — টেবিল বড় করা ও রিহ্যাশ করা
    private function resize() {
        $oldKeys = $this->keys;
        $oldValues = $this->values;
        $oldDeleted = $this->deleted;
        $oldSize = $this->size;

        // নতুন আকার: প্রায় দ্বিগুণ মৌলিক সংখ্যা
        $this->size = $this->nextPrime($oldSize * 2);
        $this->keys = array_fill(0, $this->size, null);
        $this->values = array_fill(0, $this->size, null);
        $this->deleted = array_fill(0, $this->size, false);
        $this->count = 0;

        // পুরানো ডেটা নতুন টেবিলে রিহ্যাশ করি
        for ($i = 0; $i < $oldSize; $i++) {
            if ($oldKeys[$i] !== null && !$oldDeleted[$i]) {
                $this->put($oldKeys[$i], $oldValues[$i]);
            }
        }
    }

    // পরবর্তী মৌলিক সংখ্যা খুঁজে বের করা
    private function nextPrime($n) {
        while (!$this->isPrime($n)) {
            $n++;
        }
        return $n;
    }

    private function isPrime($n) {
        if ($n < 2) return false;
        for ($i = 2; $i * $i <= $n; $i++) {
            if ($n % $i === 0) return false;
        }
        return true;
    }
}

// ব্যবহার
$ht = new OpenAddressingHashTable(7);
$ht->put("রহিম", 90);
$ht->put("করিম", 85);
$ht->put("জামিল", 92);

echo $ht->get("রহিম");   // 90
$ht->delete("করিম");
echo $ht->get("করিম");   // null
?>
```

---

## ৪. অপারেশন ও কমপ্লেক্সিটি

### ৪.১ Big O বিশ্লেষণ

```
  ┌─────────────────────────────────────────────────────────────────┐
  │              হ্যাশ টেবিল — টাইম কমপ্লেক্সিটি                  │
  ├──────────────┬────────────────┬────────────────┬───────────────┤
  │  অপারেশন     │   গড় (Average) │ সর্বোত্তম(Best)│ সর্বনিম্ন    │
  │              │                │                │  (Worst)      │
  ├──────────────┼────────────────┼────────────────┼───────────────┤
  │  ইনসার্ট     │     O(1)       │     O(1)       │    O(n)       │
  │  সার্চ       │     O(1)       │     O(1)       │    O(n)       │
  │  ডিলিট       │     O(1)       │     O(1)       │    O(n)       │
  ├──────────────┼────────────────┼────────────────┼───────────────┤
  │  স্পেস       │     O(n)       │       —        │    O(n)       │
  └──────────────┴────────────────┴────────────────┴───────────────┘

  সর্বনিম্ন O(n) কখন হয়?
  → সব কী একই ইনডেক্সে হ্যাশ হলে (সবচেয়ে খারাপ হ্যাশ ফাংশন)
  → তখন চেইনিং-এ লিংকড লিস্ট সার্চ হয়
```

### ৪.২ লোড ফ্যাক্টর (Load Factor)

```
  α = n / m

  যেখানে:
    n = মোট সংরক্ষিত আইটেম সংখ্যা
    m = টেবিলের আকার (বাকেট সংখ্যা)

  ┌─────────────┬───────────────────────────────────────────────┐
  │ লোড ফ্যাক্টর │           তাৎপর্য                             │
  ├─────────────┼───────────────────────────────────────────────┤
  │   α < 0.5   │ খুব কম ভরা — মেমোরি অপচয়                    │
  │   α ≈ 0.7   │ আদর্শ — ভালো পারফরম্যান্স ও মেমোরি ব্যালেন্স│
  │   α > 0.8   │ অতিরিক্ত ভরা — কলিশন বাড়ে, রিসাইজ দরকার    │
  │   α > 1.0   │ শুধু চেইনিং-এ সম্ভব — গড় চেইন দৈর্ঘ্য > ১  │
  └─────────────┴───────────────────────────────────────────────┘
```

### ৪.৩ রিসাইজিং ও রিহ্যাশিং

```
  রিসাইজিং প্রক্রিয়া:

  ১. লোড ফ্যাক্টর থ্রেশহোল্ড (যেমন ০.৭) অতিক্রম করলে:
     ┌───────────┐
     │ পুরানো    │   আকার: 7, আইটেম: 5
     │ টেবিল     │   α = 5/7 ≈ 0.71 ← থ্রেশহোল্ড পার!
     └─────┬─────┘
           │
           ▼
  ২. নতুন বড় টেবিল তৈরি (সাধারণত ২× মৌলিক সংখ্যা):
     ┌───────────────────┐
     │ নতুন টেবিল         │   আকার: 17 (পরবর্তী মৌলিক)
     │ (খালি)             │
     └─────┬─────────────┘
           │
           ▼
  ৩. সব আইটেম নতুন হ্যাশ ফাংশনে রিহ্যাশ:
     প্রতিটি আইটেমের নতুন ইনডেক্স = h(key) % 17

  ⏱️ রিসাইজিং-এর খরচ: O(n)
  কিন্তু অ্যামোর্টাইজড (গড়ে): O(1) প্রতি ইনসার্ট
```

### ৪.৪ অ্যামোর্টাইজড বিশ্লেষণ

```
  ধরি, প্রতিটি সাধারণ ইনসার্টের খরচ = ১
  রিসাইজিং-এর খরচ = n (সব আইটেম রিহ্যাশ)

  n টি ইনসার্টে মোট খরচ:
  = n (সাধারণ ইনসার্ট) + n + n/2 + n/4 + ... (রিসাইজিং)
  = n + 2n = 3n

  প্রতি ইনসার্টে গড় খরচ = 3n / n = O(1) ← অ্যামোর্টাইজড O(1)

  ┌────────┬───────┬───────────────────────────────────┐
  │ ইনসার্ট │ খরচ   │ ব্যাখ্যা                           │
  │ নম্বর  │       │                                   │
  ├────────┼───────┼───────────────────────────────────┤
  │   1    │   1   │ সাধারণ ইনসার্ট                     │
  │   2    │   1   │ সাধারণ ইনসার্ট                     │
  │   3    │   1   │ সাধারণ ইনসার্ট                     │
  │   4    │   1   │ সাধারণ ইনসার্ট                     │
  │   5    │ 1+4   │ ইনসার্ট + রিসাইজ (৪টি রিহ্যাশ)    │
  │   6    │   1   │ সাধারণ ইনসার্ট                     │
  │  ...   │  ...  │                                   │
  │   9    │ 1+8   │ ইনসার্ট + রিসাইজ (৮টি রিহ্যাশ)    │
  └────────┴───────┴───────────────────────────────────┘
```

---

## ৫. ভেরিয়েশন ও বিল্ট-ইন স্ট্রাকচার

### ৫.১ তুলনা টেবিল

```
  ┌──────────────┬──────────┬────────────┬──────────────────────────┐
  │ স্ট্রাকচার   │ ক্রমানুসারে│ ডুপ্লিকেট  │ মূল বৈশিষ্ট্য              │
  ├──────────────┼──────────┼────────────┼──────────────────────────┤
  │ HashSet      │   না     │    না      │ শুধু কী, কোনো ভ্যালু নেই  │
  │ HashMap      │   না     │ কী: না     │ কী-ভ্যালু, null কী হতে পারে│
  │ LinkedHashMap│   হ্যাঁ   │ কী: না     │ ইনসার্ট ক্রম মনে রাখে     │
  │ TreeMap      │   হ্যাঁ   │ কী: না     │ সর্টেড কী, O(log n)       │
  └──────────────┴──────────┴────────────┴──────────────────────────┘
```

### ৫.২ PHP এর অ্যারে — লুকানো হ্যাশ টেবিল

PHP এর অ্যারে আসলে একটি **অর্ডারড হ্যাশ ম্যাপ** — এটি ইনসার্ট ক্রম মনে রাখে।

```php
<?php
// PHP অ্যারে — অভ্যন্তরে হ্যাশ টেবিল ব্যবহার করে

// অ্যাসোসিয়েটিভ অ্যারে (হ্যাশ ম্যাপ)
$ফল = [
    "আম"    => 120,
    "কাঁঠাল" => 80,
    "লিচু"   => 200
];

// কী আছে কিনা পরীক্ষা — O(1)
var_dump(array_key_exists("আম", $ফল));     // true
var_dump(in_array(200, $ফল));               // true (ভ্যালু সার্চ O(n))

// PHP তে সেট হিসেবে ব্যবহার — কী ব্যবহার করে
$দেখাHoyeche = [];
$সংখ্যা = [1, 2, 3, 2, 4, 1, 5];

foreach ($সংখ্যা as $num) {
    if (isset($দেখাHoyeche[$num])) {
        echo "$num আগেই দেখা হয়েছে!\n";
    } else {
        $দেখাHoyeche[$num] = true;
    }
}

// array_count_values — ফ্রিকোয়েন্সি ম্যাপ
$ফলের_তালিকা = ["আম", "লিচু", "আম", "কলা", "আম", "লিচু"];
$গণনা = array_count_values($ফলের_তালিকা);
// ফলাফল: ["আম" => 3, "লিচু" => 2, "কলা" => 1]
print_r($গণনা);
?>
```

### ৫.৩ JavaScript — Map, Set, WeakMap, WeakRef

```javascript
// ──────── Map ────────
// যেকোনো টাইপের কী গ্রহণ করে (অবজেক্ট সহ)
const মানচিত্র = new Map();

// অবজেক্ট কী হিসেবে ব্যবহার করা যায়
const ব্যক্তি = { নাম: "রহিম" };
মানচিত্র.set(ব্যক্তি, { বয়স: 25 });
console.log(মানচিত্র.get(ব্যক্তি)); // { বয়স: 25 }

// ইটারেশন — ইনসার্ট ক্রমে
মানচিত্র.set("ক", 1);
মানচিত্র.set("খ", 2);
for (const [কী, মান] of মানচিত্র) {
    console.log(`${কী} → ${মান}`);
}

// ──────── Set ────────
// ইউনিক ভ্যালুর সেট
const অনন্য = new Set([1, 2, 3, 2, 4, 1, 5]);
console.log(অনন্য.size);       // 5
console.log(অনন্য.has(3));     // true
অনন্য.add(6);
অনন্য.delete(1);

// অ্যারে থেকে ডুপ্লিকেট সরানো
const তালিকা = [1, 2, 2, 3, 3, 4];
const ইউনিক = [...new Set(তালিকা)]; // [1, 2, 3, 4]

// ──────── WeakMap ────────
// শুধু অবজেক্ট কী হিসেবে গ্রহণ করে
// গার্বেজ কালেকশনে বাধা দেয় না
const ক্যাশ = new WeakMap();

let ডেটা = { id: 1 };
ক্যাশ.set(ডেটা, "গুরুত্বপূর্ণ তথ্য");
console.log(ক্যাশ.get(ডেটা)); // "গুরুত্বপূর্ণ তথ্য"

// ডেটা রেফারেন্স মুছে ফেললে WeakMap থেকেও সরে যায়
ডেটা = null; // গার্বেজ কালেক্ট হবে

// ──────── Object বনাম Map ────────
// Object: কী শুধু স্ট্রিং বা Symbol
// Map: যেকোনো টাইপের কী, .size প্রপার্টি, ইটারেবল
console.log("Map বনাম Object:");
console.log("Map আকার:", মানচিত্র.size);               // সরাসরি আকার পাওয়া যায়
console.log("Object আকার:", Object.keys({}).length);     // ম্যানুয়ালি গুণতে হয়
```

---

## ৬. জটিল সমস্যা ও সমাধান

### ৬.১ Two Sum (দুটি সংখ্যার যোগফল)

**সমস্যা**: একটি অ্যারে ও একটি লক্ষ্য সংখ্যা দেওয়া আছে। দুটি সংখ্যা খুঁজুন যাদের যোগফল = লক্ষ্য।

```
  ইনপুট: nums = [2, 7, 11, 15], target = 9
  আউটপুট: [0, 1]   (কারণ nums[0] + nums[1] = 2 + 7 = 9)
```

```php
<?php
// Two Sum — হ্যাশ ম্যাপ ব্যবহার করে O(n) সমাধান
function twoSum($nums, $target) {
    $map = []; // সংখ্যা → ইনডেক্স ম্যাপ

    foreach ($nums as $i => $num) {
        // পরিপূরক সংখ্যা (target - বর্তমান) ম্যাপে আছে কিনা দেখি
        $complement = $target - $num;

        if (isset($map[$complement])) {
            // পেয়ে গেছি! দুই ইনডেক্স ফেরত দিই
            return [$map[$complement], $i];
        }

        // বর্তমান সংখ্যা ও তার ইনডেক্স ম্যাপে রাখি
        $map[$num] = $i;
    }

    return []; // সমাধান নেই
}

print_r(twoSum([2, 7, 11, 15], 9)); // [0, 1]
?>
```

```javascript
// Two Sum — JavaScript সমাধান
function twoSum(nums, target) {
    const map = new Map(); // সংখ্যা → ইনডেক্স

    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];

        // পরিপূরক সংখ্যা আগেই দেখেছি কিনা
        if (map.has(complement)) {
            return [map.get(complement), i];
        }

        // বর্তমান সংখ্যা সংরক্ষণ
        map.set(nums[i], i);
    }

    return [];
}

console.log(twoSum([2, 7, 11, 15], 9)); // [0, 1]
```

### ৬.২ Group Anagrams (অ্যানাগ্রাম গ্রুপিং)

**সমস্যা**: একগুচ্ছ শব্দ দেওয়া আছে। যেসব শব্দ একই অক্ষর দিয়ে গঠিত তাদের গ্রুপ করুন।

```
  ইনপুট: ["eat","tea","tan","ate","nat","bat"]
  আউটপুট: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

```php
<?php
// Group Anagrams — সর্টেড স্ট্রিং কে কী হিসেবে ব্যবহার
function groupAnagrams($strs) {
    $গ্রুপ = []; // সর্টেড_স্ট্রিং → শব্দের তালিকা

    foreach ($strs as $word) {
        // শব্দের অক্ষরগুলো সর্ট করে কী বানাই
        $chars = str_split($word);
        sort($chars);
        $key = implode('', $chars);

        // একই কী তে শব্দ যোগ করি
        $গ্রুপ[$key][] = $word;
    }

    return array_values($গ্রুপ);
}

$ফলাফল = groupAnagrams(["eat","tea","tan","ate","nat","bat"]);
print_r($ফলাফল);
// [["eat","tea","ate"], ["tan","nat"], ["bat"]]
?>
```

```javascript
// Group Anagrams — JavaScript সমাধান
function groupAnagrams(strs) {
    const groups = new Map();

    for (const word of strs) {
        // অক্ষরগুলো সর্ট করে কী তৈরি
        const key = word.split('').sort().join('');

        if (!groups.has(key)) {
            groups.set(key, []);
        }
        groups.get(key).push(word);
    }

    return [...groups.values()];
}

console.log(groupAnagrams(["eat","tea","tan","ate","nat","bat"]));
```

### ৬.৩ Longest Consecutive Sequence (দীর্ঘতম ক্রমিক ধারা)

**সমস্যা**: একটি আনসর্টেড অ্যারেতে দীর্ঘতম পরপর সংখ্যার ধারার দৈর্ঘ্য বের করুন। O(n) সময়ে।

```
  ইনপুট: [100, 4, 200, 1, 3, 2]
  আউটপুট: 4   (ধারা: 1, 2, 3, 4)
```

```php
<?php
// Longest Consecutive Sequence — HashSet ব্যবহারে O(n)
function longestConsecutive($nums) {
    if (empty($nums)) return 0;

    // সব সংখ্যা সেটে রাখি — O(1) লুকআপের জন্য
    $numSet = [];
    foreach ($nums as $num) {
        $numSet[$num] = true;
    }

    $সর্বোচ্চ = 0;

    foreach ($numSet as $num => $_) {
        // শুধু ধারার শুরু থেকে গণনা করি
        // num-1 সেটে থাকলে এটা শুরু নয়, এড়িয়ে যাই
        if (!isset($numSet[$num - 1])) {
            $বর্তমান = $num;
            $দৈর্ঘ্য = 1;

            // পরের সংখ্যাগুলো সেটে আছে কিনা দেখি
            while (isset($numSet[$বর্তমান + 1])) {
                $বর্তমান++;
                $দৈর্ঘ্য++;
            }

            $সর্বোচ্চ = max($সর্বোচ্চ, $দৈর্ঘ্য);
        }
    }

    return $সর্বোচ্চ;
}

echo longestConsecutive([100, 4, 200, 1, 3, 2]); // 4
?>
```

```javascript
// Longest Consecutive Sequence — JavaScript
function longestConsecutive(nums) {
    const numSet = new Set(nums);
    let longest = 0;

    for (const num of numSet) {
        // ধারার শুরু হলেই গণনা করি
        if (!numSet.has(num - 1)) {
            let current = num;
            let length = 1;

            while (numSet.has(current + 1)) {
                current++;
                length++;
            }

            longest = Math.max(longest, length);
        }
    }

    return longest;
}

console.log(longestConsecutive([100, 4, 200, 1, 3, 2])); // 4
```

### ৬.৪ Subarray Sum Equals K (সাবঅ্যারে যোগফল = K)

**সমস্যা**: একটি অ্যারেতে কতগুলো সাবঅ্যারের যোগফল K এর সমান?

```
  ইনপুট: nums = [1, 1, 1], k = 2
  আউটপুট: 2   (সাবঅ্যারে: [1,1] (0-1), [1,1] (1-2))
```

```php
<?php
// Subarray Sum = K — প্রিফিক্স সাম + হ্যাশ ম্যাপ
function subarraySum($nums, $k) {
    // প্রিফিক্স সাম → কতবার দেখা গেছে
    $prefixCount = [0 => 1]; // ০ যোগফল ১বার (শুরুতে)
    $sum = 0;
    $count = 0;

    foreach ($nums as $num) {
        $sum += $num; // চলমান যোগফল

        // (sum - k) আগে দেখা গেলে, ওই পয়েন্ট থেকে এখন পর্যন্ত = k
        if (isset($prefixCount[$sum - $k])) {
            $count += $prefixCount[$sum - $k];
        }

        // বর্তমান প্রিফিক্স সাম ম্যাপে যোগ
        if (!isset($prefixCount[$sum])) {
            $prefixCount[$sum] = 0;
        }
        $prefixCount[$sum]++;
    }

    return $count;
}

echo subarraySum([1, 1, 1], 2); // 2
echo subarraySum([1, 2, 3, -3, 1, 1, 1], 3); // 5
?>
```

```javascript
// Subarray Sum = K — JavaScript
function subarraySum(nums, k) {
    const prefixCount = new Map();
    prefixCount.set(0, 1); // খালি সাবঅ্যারে

    let sum = 0;
    let count = 0;

    for (const num of nums) {
        sum += num;

        // (sum - k) প্রিফিক্স আগে দেখা গেলে
        if (prefixCount.has(sum - k)) {
            count += prefixCount.get(sum - k);
        }

        prefixCount.set(sum, (prefixCount.get(sum) || 0) + 1);
    }

    return count;
}

console.log(subarraySum([1, 1, 1], 2)); // 2
```

### ৬.৫ LRU Cache (সর্বশেষ ব্যবহৃত ক্যাশ)

**সমস্যা**: একটি ক্যাশ ডিজাইন করুন যেটা ক্ষমতা পূর্ণ হলে সবচেয়ে পুরানো (Least Recently Used) আইটেম বাদ দেয়।

```
  ক্ষমতা: 3

  put(1,"ক") → [1]
  put(2,"খ") → [1, 2]
  put(3,"গ") → [1, 2, 3]
  get(1)     → "ক", ক্রম: [2, 3, 1] (1 সবচেয়ে সাম্প্রতিক)
  put(4,"ঘ") → ক্ষমতা পূর্ণ! 2 বাদ → [3, 1, 4]
```

```javascript
// LRU Cache — Map ব্যবহার করে (Map ইনসার্ট ক্রম রাখে)
class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map(); // ইনসার্ট ক্রম সংরক্ষিত থাকে
    }

    get(key) {
        if (!this.cache.has(key)) return -1;

        // কী পেলে সবচেয়ে সাম্প্রতিক করি
        const value = this.cache.get(key);
        this.cache.delete(key);    // মুছে
        this.cache.set(key, value); // আবার যোগ (শেষে যায়)
        return value;
    }

    put(key, value) {
        // আগে থাকলে মুছে ফেলি (ক্রম আপডেটের জন্য)
        if (this.cache.has(key)) {
            this.cache.delete(key);
        }

        this.cache.set(key, value);

        // ক্ষমতা অতিক্রম করলে সবচেয়ে পুরানো আইটেম বাদ
        if (this.cache.size > this.capacity) {
            // Map এর প্রথম কী = সবচেয়ে পুরানো
            const oldestKey = this.cache.keys().next().value;
            this.cache.delete(oldestKey);
        }
    }
}

// ব্যবহার
const cache = new LRUCache(3);
cache.put(1, "ক");
cache.put(2, "খ");
cache.put(3, "গ");
console.log(cache.get(1));  // "ক" (1 সাম্প্রতিক হলো)
cache.put(4, "ঘ");          // 2 বাদ গেলো (সবচেয়ে পুরানো)
console.log(cache.get(2));  // -1 (বাদ পড়েছে)
```

```php
<?php
// LRU Cache — PHP বাস্তবায়ন (অর্ডারড অ্যারে ব্যবহারে)
class LRUCache {
    private $capacity;
    private $cache; // কী → ভ্যালু (ইনসার্ট ক্রমে)

    public function __construct($capacity) {
        $this->capacity = $capacity;
        $this->cache = [];
    }

    public function get($key) {
        if (!isset($this->cache[$key])) return -1;

        // সবচেয়ে সাম্প্রতিক করতে: মুছে আবার যোগ
        $value = $this->cache[$key];
        unset($this->cache[$key]);
        $this->cache[$key] = $value;
        return $value;
    }

    public function put($key, $value) {
        // আগে থাকলে সরাই
        if (isset($this->cache[$key])) {
            unset($this->cache[$key]);
        }

        $this->cache[$key] = $value;

        // ক্ষমতা অতিক্রম করলে প্রথমটি (সবচেয়ে পুরানো) বাদ
        if (count($this->cache) > $this->capacity) {
            // PHP অ্যারের প্রথম কী = সবচেয়ে পুরানো
            reset($this->cache);
            $oldestKey = key($this->cache);
            unset($this->cache[$oldestKey]);
        }
    }
}

$cache = new LRUCache(3);
$cache->put(1, "ক");
$cache->put(2, "খ");
$cache->put(3, "গ");
echo $cache->get(1) . "\n";   // "ক"
$cache->put(4, "ঘ");          // 2 বাদ
echo $cache->get(2) . "\n";   // -1
?>
```

### ৬.৬ Top K Frequent Elements (সবচেয়ে ঘন K সংখ্যা)

**সমস্যা**: একটি অ্যারেতে সবচেয়ে বেশিবার আসা K টি সংখ্যা বের করুন।

```
  ইনপুট: nums = [1,1,1,2,2,3], k = 2
  আউটপুট: [1, 2]
```

```php
<?php
// Top K Frequent — ফ্রিকোয়েন্সি ম্যাপ + বাকেট সর্ট
function topKFrequent($nums, $k) {
    // ধাপ ১: ফ্রিকোয়েন্সি গণনা — O(n)
    $freq = [];
    foreach ($nums as $num) {
        $freq[$num] = ($freq[$num] ?? 0) + 1;
    }

    // ধাপ ২: বাকেট সর্ট — ফ্রিকোয়েন্সি অনুযায়ী বাকেটে রাখি
    $n = count($nums);
    $buckets = array_fill(0, $n + 1, []);
    foreach ($freq as $num => $count) {
        $buckets[$count][] = $num;
    }

    // ধাপ ৩: উচ্চ ফ্রিকোয়েন্সি থেকে K টি বের করি
    $result = [];
    for ($i = $n; $i >= 0 && count($result) < $k; $i--) {
        foreach ($buckets[$i] as $num) {
            $result[] = $num;
            if (count($result) === $k) break;
        }
    }

    return $result;
}

print_r(topKFrequent([1,1,1,2,2,3], 2)); // [1, 2]
?>
```

```javascript
// Top K Frequent — JavaScript (বাকেট সর্ট)
function topKFrequent(nums, k) {
    // ফ্রিকোয়েন্সি ম্যাপ তৈরি
    const freq = new Map();
    for (const num of nums) {
        freq.set(num, (freq.get(num) || 0) + 1);
    }

    // বাকেট: ইনডেক্স = ফ্রিকোয়েন্সি
    const buckets = new Array(nums.length + 1).fill(null).map(() => []);
    for (const [num, count] of freq) {
        buckets[count].push(num);
    }

    // উচ্চ ফ্রিকোয়েন্সি থেকে সংগ্রহ
    const result = [];
    for (let i = buckets.length - 1; i >= 0 && result.length < k; i--) {
        result.push(...buckets[i]);
    }

    return result.slice(0, k);
}

console.log(topKFrequent([1,1,1,2,2,3], 2)); // [1, 2]
```

---

## ৭. প্রজেক্ট: সিম্পল ডেটাবেস ইনডেক্স

### লক্ষ্য

হ্যাশ-ভিত্তিক ইনডেক্সিং ব্যবহার করে একটি সরল ডেটাবেস তৈরি করুন — রেকর্ড ইনসার্ট, সার্চ, ডিলিট করতে পারবেন।

### আর্কিটেকচার

```
  ┌──────────────────────────────────────────────────────┐
  │                  ডেটাবেস ইনডেক্স                      │
  │                                                      │
  │  কমান্ড ──▶ ┌──────────┐    ┌──────────────────┐     │
  │             │ হ্যাশ     │    │   রেকর্ড স্টোর    │     │
  │  INSERT ──▶ │ ইনডেক্স   │◀──▶│  (অ্যারে)         │     │
  │  SEARCH ──▶ │           │    │                  │     │
  │  DELETE ──▶ │ কী→অবস্থান │    │ [রেকর্ড১,রেকর্ড২] │     │
  │             └──────────┘    └──────────────────┘     │
  │                                                      │
  │  ইনডেক্স ম্যাপ:                                      │
  │    "id:1001"  → অবস্থান 0                             │
  │    "name:রহিম" → [অবস্থান 0, অবস্থান 3]               │
  │    "id:1002"  → অবস্থান 1                             │
  └──────────────────────────────────────────────────────┘
```

### PHP বাস্তবায়ন

```php
<?php
// সিম্পল ডেটাবেস — হ্যাশ ইনডেক্স সহ
class SimpleDatabase {
    private $records;       // রেকর্ড সংরক্ষণ (অ্যারে)
    private $primaryIndex;  // প্রাইমারি কী ইনডেক্স (id → অবস্থান)
    private $secondaryIndex; // সেকেন্ডারি ইনডেক্স (ফিল্ড:মান → [অবস্থান])
    private $nextId;        // পরবর্তী স্বয়ংক্রিয় আইডি

    public function __construct() {
        $this->records = [];
        $this->primaryIndex = [];
        $this->secondaryIndex = [];
        $this->nextId = 1;
    }

    // রেকর্ড ইনসার্ট — O(1) গড়
    public function insert($data) {
        // স্বয়ংক্রিয় আইডি যোগ
        $data['id'] = $this->nextId++;
        $position = count($this->records);

        // রেকর্ড সংরক্ষণ
        $this->records[$position] = $data;

        // প্রাইমারি ইনডেক্স আপডেট
        $this->primaryIndex[$data['id']] = $position;

        // সেকেন্ডারি ইনডেক্স আপডেট — প্রতিটি ফিল্ডের জন্য
        foreach ($data as $field => $value) {
            $key = $field . ':' . $value;
            if (!isset($this->secondaryIndex[$key])) {
                $this->secondaryIndex[$key] = [];
            }
            $this->secondaryIndex[$key][] = $position;
        }

        return $data['id'];
    }

    // আইডি দিয়ে খোঁজা — O(1)
    public function findById($id) {
        if (!isset($this->primaryIndex[$id])) {
            return null; // পাওয়া যায়নি
        }

        $position = $this->primaryIndex[$id];
        $record = $this->records[$position];

        // মুছে ফেলা হয়েছে কিনা পরীক্ষা
        if ($record === null) return null;

        return $record;
    }

    // ফিল্ড ও মান দিয়ে খোঁজা — O(k) যেখানে k = মিলে যাওয়া রেকর্ড সংখ্যা
    public function findByField($field, $value) {
        $key = $field . ':' . $value;

        if (!isset($this->secondaryIndex[$key])) {
            return []; // কিছু পাওয়া যায়নি
        }

        $results = [];
        foreach ($this->secondaryIndex[$key] as $position) {
            if ($this->records[$position] !== null) {
                $results[] = $this->records[$position];
            }
        }

        return $results;
    }

    // রেকর্ড মুছে ফেলা — O(1) গড়
    public function deleteById($id) {
        if (!isset($this->primaryIndex[$id])) {
            return false;
        }

        $position = $this->primaryIndex[$id];
        $record = $this->records[$position];

        if ($record === null) return false;

        // সেকেন্ডারি ইনডেক্স পরিষ্কার
        foreach ($record as $field => $value) {
            $key = $field . ':' . $value;
            if (isset($this->secondaryIndex[$key])) {
                $this->secondaryIndex[$key] = array_filter(
                    $this->secondaryIndex[$key],
                    fn($pos) => $pos !== $position
                );
            }
        }

        // রেকর্ড ও প্রাইমারি ইনডেক্স মুছি
        $this->records[$position] = null;
        unset($this->primaryIndex[$id]);

        return true;
    }

    // রেকর্ড আপডেট — O(1) গড়
    public function update($id, $newData) {
        if (!isset($this->primaryIndex[$id])) {
            return false;
        }

        $position = $this->primaryIndex[$id];
        $oldRecord = $this->records[$position];

        if ($oldRecord === null) return false;

        // পুরানো সেকেন্ডারি ইনডেক্স সরাই
        foreach ($oldRecord as $field => $value) {
            $key = $field . ':' . $value;
            if (isset($this->secondaryIndex[$key])) {
                $this->secondaryIndex[$key] = array_filter(
                    $this->secondaryIndex[$key],
                    fn($pos) => $pos !== $position
                );
            }
        }

        // নতুন ডেটা মার্জ (আইডি অপরিবর্তিত)
        $newData['id'] = $id;
        $this->records[$position] = $newData;

        // নতুন সেকেন্ডারি ইনডেক্স তৈরি
        foreach ($newData as $field => $value) {
            $key = $field . ':' . $value;
            if (!isset($this->secondaryIndex[$key])) {
                $this->secondaryIndex[$key] = [];
            }
            $this->secondaryIndex[$key][] = $position;
        }

        return true;
    }

    // সব রেকর্ড দেখা
    public function getAll() {
        return array_filter($this->records, fn($r) => $r !== null);
    }

    // রেকর্ড সংখ্যা
    public function count() {
        return count(array_filter($this->records, fn($r) => $r !== null));
    }
}

// ============================
// ব্যবহারের উদাহরণ
// ============================
$db = new SimpleDatabase();

// রেকর্ড ইনসার্ট
$id1 = $db->insert(["নাম" => "রহিম", "বয়স" => 25, "শহর" => "ঢাকা"]);
$id2 = $db->insert(["নাম" => "করিম", "বয়স" => 30, "শহর" => "চট্টগ্রাম"]);
$id3 = $db->insert(["নাম" => "জামিল", "বয়স" => 25, "শহর" => "ঢাকা"]);

echo "=== আইডি দিয়ে খোঁজা ===\n";
print_r($db->findById($id1));
// ["id" => 1, "নাম" => "রহিম", "বয়স" => 25, "শহর" => "ঢাকা"]

echo "\n=== শহর দিয়ে খোঁজা ===\n";
print_r($db->findByField("শহর", "ঢাকা"));
// রহিম ও জামিল দুজনেই পাওয়া যাবে

echo "\n=== বয়স দিয়ে খোঁজা ===\n";
print_r($db->findByField("বয়স", 25));
// বয়স ২৫ এর সবাই

echo "\n=== আপডেট ===\n";
$db->update($id2, ["নাম" => "করিম", "বয়স" => 31, "শহর" => "সিলেট"]);
print_r($db->findById($id2));

echo "\n=== ডিলিট ===\n";
$db->deleteById($id1);
echo "মোট রেকর্ড: " . $db->count() . "\n"; // 2
?>
```

### JavaScript বাস্তবায়ন

```javascript
// সিম্পল ডেটাবেস — হ্যাশ ইনডেক্স সহ
class SimpleDatabase {
    constructor() {
        this.records = [];           // রেকর্ড সংরক্ষণ
        this.primaryIndex = new Map(); // id → অবস্থান
        this.secondaryIndex = new Map(); // "ফিল্ড:মান" → Set(অবস্থান)
        this.nextId = 1;
    }

    // রেকর্ড ইনসার্ট
    insert(data) {
        data.id = this.nextId++;
        const position = this.records.length;
        this.records.push(data);

        // প্রাইমারি ইনডেক্স
        this.primaryIndex.set(data.id, position);

        // সেকেন্ডারি ইনডেক্স — প্রতিটি ফিল্ডের জন্য
        for (const [field, value] of Object.entries(data)) {
            const key = `${field}:${value}`;
            if (!this.secondaryIndex.has(key)) {
                this.secondaryIndex.set(key, new Set());
            }
            this.secondaryIndex.get(key).add(position);
        }

        return data.id;
    }

    // আইডি দিয়ে খোঁজা — O(1)
    findById(id) {
        if (!this.primaryIndex.has(id)) return null;
        const pos = this.primaryIndex.get(id);
        return this.records[pos];
    }

    // ফিল্ড দিয়ে খোঁজা
    findByField(field, value) {
        const key = `${field}:${value}`;
        if (!this.secondaryIndex.has(key)) return [];

        const results = [];
        for (const pos of this.secondaryIndex.get(key)) {
            if (this.records[pos] !== null) {
                results.push(this.records[pos]);
            }
        }
        return results;
    }

    // রেকর্ড মুছে ফেলা
    deleteById(id) {
        if (!this.primaryIndex.has(id)) return false;

        const pos = this.primaryIndex.get(id);
        const record = this.records[pos];
        if (!record) return false;

        // সেকেন্ডারি ইনডেক্স পরিষ্কার
        for (const [field, value] of Object.entries(record)) {
            const key = `${field}:${value}`;
            this.secondaryIndex.get(key)?.delete(pos);
        }

        this.records[pos] = null;
        this.primaryIndex.delete(id);
        return true;
    }

    // রেকর্ড সংখ্যা
    count() {
        return this.records.filter(r => r !== null).length;
    }
}

// ব্যবহার
const db = new SimpleDatabase();

const id1 = db.insert({ নাম: "রহিম", বয়স: 25, শহর: "ঢাকা" });
const id2 = db.insert({ নাম: "করিম", বয়স: 30, শহর: "চট্টগ্রাম" });
const id3 = db.insert({ নাম: "জামিল", বয়স: 25, শহর: "ঢাকা" });

console.log("আইডি দিয়ে:", db.findById(id1));
console.log("শহর দিয়ে:", db.findByField("শহর", "ঢাকা"));

db.deleteById(id1);
console.log("মোট রেকর্ড:", db.count()); // 2
```

---

## ৮. চিটশিট ও ইন্টারভিউ টিপস

### 📋 চিটশিট

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    হ্যাশ টেবিল চিটশিট                          │
  ├─────────────────────────────────────────────────────────────────┤
  │                                                                 │
  │  🔑 মূল ধারণা:                                                  │
  │  • কী → হ্যাশ ফাংশন → ইনডেক্স → ভ্যালু                        │
  │  • গড় O(1) ইনসার্ট, সার্চ, ডিলিট                              │
  │  • সর্বনিম্ন O(n) — সব কী একই ইনডেক্সে হ্যাশ হলে              │
  │                                                                 │
  │  ⚡ কলিশন হ্যান্ডলিং:                                           │
  │  • চেইনিং: প্রতি বাকেটে লিংকড লিস্ট                           │
  │  • ওপেন অ্যাড্রেসিং: লিনিয়ার/কোয়াড্রাটিক/ডাবল হ্যাশিং      │
  │                                                                 │
  │  📊 লোড ফ্যাক্টর (α):                                           │
  │  • α = আইটেম সংখ্যা / টেবিলের আকার                             │
  │  • আদর্শ: ০.৬ — ০.৭৫                                           │
  │  • α > থ্রেশহোল্ড → রিসাইজ (২× করো) + রিহ্যাশ                 │
  │                                                                 │
  │  🛠️ কখন হ্যাশ ম্যাপ ব্যবহার করবেন:                             │
  │  • O(1) লুকআপ দরকার                                            │
  │  • ফ্রিকোয়েন্সি গণনা                                           │
  │  • ডুপ্লিকেট চেক                                                │
  │  • দুটি ডেটাসেটের মধ্যে সম্পর্ক রাখা                           │
  │  • ক্যাশিং                                                      │
  │                                                                 │
  │  🚫 কখন ব্যবহার করবেন না:                                      │
  │  • সর্টেড ডেটা দরকার → TreeMap ব্যবহার করুন                    │
  │  • রেঞ্জ কোয়েরি দরকার → BST বা সেগমেন্ট ট্রি                 │
  │  • মেমোরি সীমিত → অ্যারে যথেষ্ট হতে পারে                      │
  └─────────────────────────────────────────────────────────────────┘
```

### 💡 ইন্টারভিউ প্যাটার্ন ও টিপস

```
  ┌──────────────────────────────────────────────────────────────┐
  │              হ্যাশ টেবিল ইন্টারভিউ প্যাটার্ন                 │
  ├──────────────────────────────────────────────────────────────┤
  │                                                              │
  │  ১. ফ্রিকোয়েন্সি কাউন্টার প্যাটার্ন:                       │
  │     → Top K, Valid Anagram, Most Common Word                 │
  │     → ম্যাপে গণনা রাখুন, তারপর ফিল্টার করুন               │
  │                                                              │
  │  ২. কমপ্লিমেন্ট প্যাটার্ন:                                  │
  │     → Two Sum, Three Sum, Pair Difference                    │
  │     → target - current ম্যাপে আছে কিনা দেখুন               │
  │                                                              │
  │  ৩. প্রিফিক্স সাম + হ্যাশ ম্যাপ:                            │
  │     → Subarray Sum = K, Contiguous Array                     │
  │     → চলমান যোগফল ম্যাপে রাখুন                              │
  │                                                              │
  │  ৪. গ্রুপিং প্যাটার্ন:                                      │
  │     → Group Anagrams, Group Shifted Strings                  │
  │     → সাধারণ বৈশিষ্ট্যকে কী বানান                          │
  │                                                              │
  │  ৫. সেট অপারেশন:                                            │
  │     → Intersection, Union, Difference                        │
  │     → Set/HashSet দিয়ে O(n) সমাধান                          │
  │                                                              │
  │  ৬. ক্যাশিং/মেমোইজেশন:                                      │
  │     → LRU Cache, LFU Cache                                  │
  │     → HashMap + Doubly Linked List                           │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 🎯 সচরাচর জিজ্ঞাসিত প্রশ্ন

**প্রশ্ন: হ্যাশ টেবিল বনাম BST — কোনটা কখন?**
- হ্যাশ টেবিল: O(1) লুকআপ দরকার, ক্রম গুরুত্বপূর্ণ নয়
- BST: সর্টেড ডেটা দরকার, রেঞ্জ কোয়েরি, min/max খুঁজতে হবে

**প্রশ্ন: চেইনিং বনাম ওপেন অ্যাড্রেসিং — কোনটা ভালো?**
- চেইনিং: সরল বাস্তবায়ন, ডিলিট সহজ, α > 1 হতে পারে
- ওপেন অ্যাড্রেসিং: ক্যাশ-ফ্রেন্ডলি, কম মেমোরি ওভারহেড, α < 1 রাখতে হবে

**প্রশ্ন: কেন রিসাইজিং-এ মৌলিক সংখ্যা ব্যবহার করা হয়?**
- মৌলিক সংখ্যা আকার ব্যবহারে হ্যাশ ভ্যালু আরো সমানভাবে বিতরিত হয়
- ২ এর পাওয়ার আকারে নিচের বিটগুলো বেশি প্রভাবিত করে — প্যাটার্ন তৈরি হয়

**প্রশ্ন: PHP এর array কি হ্যাশ টেবিল?**
- হ্যাঁ! PHP অ্যারে অভ্যন্তরে একটি অর্ডারড হ্যাশ ম্যাপ — ইনসার্ট ক্রম সংরক্ষিত

**প্রশ্ন: JS এ Object বনাম Map?**
- Map: যেকোনো টাইপের কী, .size, ইটারেবল, বেশি পারফরম্যান্ট (ঘন ইনসার্ট/ডিলিটে)
- Object: শুধু স্ট্রিং/Symbol কী, প্রোটোটাইপ চেইন সমস্যা

---

### 🔗 পরবর্তী পড়ুন

- [sorting.md](./sorting.md) — সর্টিং অ্যালগরিদম বিস্তারিত

---

> 📝 **শেষ কথা**: হ্যাশ টেবিল হলো প্রোগ্রামিং-এর সবচেয়ে গুরুত্বপূর্ণ ডেটা স্ট্রাকচারগুলোর একটি।
> ইন্টারভিউতে প্রায় ৩০-৪০% সমস্যায় হ্যাশ ম্যাপ ব্যবহার হয়। উপরের প্যাটার্নগুলো ভালোভাবে
> অনুশীলন করুন এবং কলিশন হ্যান্ডলিং ও রিসাইজিং-এর ধারণা স্পষ্ট রাখুন।
