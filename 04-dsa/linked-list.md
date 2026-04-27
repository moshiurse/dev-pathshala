# 🔗 লিংকড লিস্ট (Linked List) — সম্পূর্ণ গাইড

> _"শিকলের প্রতিটি আংটা আলাদা, তবুও একসাথে বাঁধা — লিংকড লিস্টও তেমনি, প্রতিটি নোড আলাদা ঠিকানায় থাকলেও পয়েন্টারের সুতোয় গাঁথা।"_ 🪢

---

## 📖 সংজ্ঞা

**লিংকড লিস্ট** হলো একটি লিনিয়ার ডেটা স্ট্রাকচার যেখানে উপাদানগুলো (নোড) মেমোরিতে ক্রমানুসারে
সাজানো থাকে না। প্রতিটি নোড দুটি জিনিস ধারণ করে:

1. **ডেটা (Data)** — প্রকৃত মান
2. **পয়েন্টার (Pointer/Reference)** — পরবর্তী নোডের ঠিকানা

অ্যারের মতো ইনডেক্স দিয়ে সরাসরি অ্যাক্সেস করা যায় না — প্রথম নোড (head) থেকে শুরু করে
একটার পর একটা ধরে ধরে যেতে হয়।

---

## 🏠 বাস্তব উদাহরণ

```
🚂 ট্রেনের বগি:
  প্রতিটি বগি (নোড) আলাদা, কিন্তু কাপলিং (পয়েন্টার) দিয়ে সংযুক্ত।
  ইঞ্জিন = Head, শেষ বগি = Tail (পরবর্তী = NULL)

  [ইঞ্জিন]--->[বগি-১]--->[বগি-২]--->[বগি-৩]--->NULL

📿 জপমালা:
  প্রতিটি পুঁতি (নোড) সুতো (পয়েন্টার) দিয়ে সংযুক্ত।

🎵 প্লেলিস্ট:
  একটি গান শেষ হলে পরের গানে যায় — Singly Linked List
  আগের গানেও ফেরত যেতে পারে — Doubly Linked List
  শেষ গানের পর আবার প্রথম গান — Circular Linked List
```

---

## 📑 সূচিপত্র

1. [Singly Linked List](#১-singly-linked-list)
2. [Doubly Linked List](#২-doubly-linked-list)
3. [Circular Linked List](#৩-circular-linked-list)
4. [লিংকড লিস্ট টেকনিক](#৪-লিংকড-লিস্ট-টেকনিক)
5. [ক্লাসিক সমস্যা](#৫-ক্লাসিক-সমস্যা)
6. [অ্যারে vs লিংকড লিস্ট তুলনা](#৬-অ্যারে-vs-লিংকড-লিস্ট-তুলনা)

---

## ১. Singly Linked List

### 📐 নোড স্ট্রাকচার

প্রতিটি নোডে দুটি ফিল্ড:

```
┌──────────┬──────────┐
│   data   │   next   │
│  (মান)   │(পরবর্তী)│
└──────────┴──────────┘
```

**সম্পূর্ণ লিংকড লিস্ট:**

```
  head
   │
   ▼
┌──────┬───┐    ┌──────┬───┐    ┌──────┬───┐    ┌──────┬──────┐
│  10  │ ──┼───►│  20  │ ──┼───►│  30  │ ──┼───►│  40  │ NULL │
└──────┴───┘    └──────┴───┘    └──────┴───┘    └──────┴──────┘
  নোড ১          নোড ২          নোড ৩          নোড ৪ (tail)
```

### 🧠 মেমোরি লেআউট — অ্যারে vs লিংকড লিস্ট

**অ্যারে (Contiguous মেমোরি):**

```
মেমোরি ঠিকানা:  100   104   108   112   116
              ┌─────┬─────┬─────┬─────┬─────┐
              │  10 │  20 │  30 │  40 │  50 │
              └─────┴─────┴─────┴─────┴─────┘
              পরপর সাজানো — ইনডেক্স দিয়ে সরাসরি অ্যাক্সেস
```

**লিংকড লিস্ট (Non-Contiguous মেমোরি):**

```
মেমোরি ঠিকানা:   200        508        312        720
               ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
               │  10  │   │  20  │   │  30  │   │  40  │
               │ @508 │   │ @312 │   │ @720 │   │ NULL │
               └──────┘   └──────┘   └──────┘   └──────┘
               এলোমেলো ঠিকানায় — পয়েন্টার দিয়ে সংযুক্ত

  ✦ অ্যারে: একটানা মেমোরি ব্লক প্রয়োজন
  ✦ লিংকড লিস্ট: মেমোরিতে যেখানে জায়গা পায় সেখানেই বসে
```

---

### ⚙️ অপারেশন সমূহ

#### 🔹 ১. Head-এ Insert (Prepend)

```
আগে:
  head -> [20] -> [30] -> [40] -> NULL

ধাপ ১: নতুন নোড তৈরি [10]
  [10]    head -> [20] -> [30] -> [40] -> NULL

ধাপ ২: নতুন নোডের next = head
  [10] -> [20] -> [30] -> [40] -> NULL
   ↑
  head (আপডেট)

পরে:
  head -> [10] -> [20] -> [30] -> [40] -> NULL

⏱ Time: O(1)
```

#### 🔹 ২. Tail-এ Insert (Append)

```
আগে:
  head -> [10] -> [20] -> [30] -> NULL

ধাপ ১: নতুন নোড তৈরি [40]

ধাপ ২: শেষ নোড পর্যন্ত traverse
  head -> [10] -> [20] -> [30] -> NULL
                            ↑
                          current (শেষ নোড পাওয়া গেছে)

ধাপ ৩: শেষ নোডের next = নতুন নোড
  head -> [10] -> [20] -> [30] -> [40] -> NULL

⏱ Time: O(n) — শেষ পর্যন্ত হাঁটতে হয়
⏱ Time: O(1) — যদি tail পয়েন্টার রাখা হয়
```

#### 🔹 ৩. মাঝে Insert (position k-তে)

```
আগে:
  head -> [10] -> [30] -> [40] -> NULL
  position 1-এ 20 insert করতে হবে (0-indexed)

ধাপ ১: position-1 পর্যন্ত traverse (নোড [10]-এ পৌঁছানো)
  head -> [10] -> [30] -> [40] -> NULL
           ↑
         current

ধাপ ২: নতুন নোড তৈরি [20]
ধাপ ৩: newNode.next = current.next
  [20] -> [30]

ধাপ ৪: current.next = newNode
  head -> [10] -> [20] -> [30] -> [40] -> NULL

⏱ Time: O(n)
```

#### 🔹 ৪. Delete (Head)

```
আগে:
  head -> [10] -> [20] -> [30] -> NULL

ধাপ ১: temp = head
  temp -> [10]

ধাপ ২: head = head.next
  head -> [20] -> [30] -> NULL

ধাপ ৩: temp মুক্ত করুন (PHP/JS-এ garbage collected)

⏱ Time: O(1)
```

#### 🔹 ৫. Delete (নির্দিষ্ট মান)

```
আগে:
  head -> [10] -> [20] -> [30] -> [40] -> NULL
  মান 30 মুছতে হবে

ধাপ ১: traverse করে 30 এর আগের নোড (20) খুঁজুন
  head -> [10] -> [20] -> [30] -> [40] -> NULL
                   ↑       ↑
                  prev   current

ধাপ ২: prev.next = current.next
  head -> [10] -> [20] ────────> [40] -> NULL
                          [30] (বিচ্ছিন্ন)

⏱ Time: O(n)
```

#### 🔹 ৬. Search (অনুসন্ধান)

```
head -> [10] -> [20] -> [30] -> [40] -> NULL
মান 30 খুঁজতে হবে

ধাপ ১: current = head, current.data (10) ≠ 30, এগিয়ে যাও
ধাপ ২: current.data (20) ≠ 30, এগিয়ে যাও
ধাপ ৩: current.data (30) = 30 ✅ পাওয়া গেছে! (index 2)

⏱ Time: O(n)
```

#### 🔹 ৭. Traverse (সমস্ত নোড প্রদর্শন)

```
head -> [10] -> [20] -> [30] -> NULL

current = head
প্রিন্ট: 10 -> current = current.next
প্রিন্ট: 20 -> current = current.next
প্রিন্ট: 30 -> current = current.next
current = NULL -> থামো

আউটপুট: 10 -> 20 -> 30 -> NULL

⏱ Time: O(n)
```

---

### 📊 কমপ্লেক্সিটি টেবিল (Singly Linked List)

```
┌───────────────────────┬──────────────┬──────────────┐
│      অপারেশন          │   সময় (Time) │   স্থান (Space) │
├───────────────────────┼──────────────┼──────────────┤
│ Head-এ Insert         │     O(1)     │     O(1)     │
│ Tail-এ Insert         │     O(n)*    │     O(1)     │
│ মাঝে Insert           │     O(n)     │     O(1)     │
│ Head Delete           │     O(1)     │     O(1)     │
│ Tail Delete           │     O(n)     │     O(1)     │
│ নির্দিষ্ট মান Delete    │     O(n)     │     O(1)     │
│ Search                │     O(n)     │     O(1)     │
│ Traverse              │     O(n)     │     O(1)     │
│ Access (index)        │     O(n)     │     O(1)     │
└───────────────────────┴──────────────┴──────────────┘
* tail পয়েন্টার রাখলে O(1)
```

---

### 💻 সম্পূর্ণ Implementation — PHP

```php
<?php
/**
 * সিঙ্গলি লিংকড লিস্ট — PHP Implementation
 * প্রতিটি নোডে data এবং next পয়েন্টার থাকে
 */

// নোড ক্লাস — লিংকড লিস্টের প্রতিটি উপাদান
class Node {
    public $data;   // নোডের মান
    public $next;   // পরবর্তী নোডের রেফারেন্স

    public function __construct($data) {
        $this->data = $data;
        $this->next = null; // প্রাথমিকভাবে পরবর্তী নোড নেই
    }
}

// সিঙ্গলি লিংকড লিস্ট ক্লাস
class SinglyLinkedList {
    private $head;  // প্রথম নোডের রেফারেন্স
    private $size;  // লিস্টের মোট নোড সংখ্যা

    public function __construct() {
        $this->head = null;
        $this->size = 0;
    }

    // শুরুতে নোড যোগ — O(1)
    public function prepend($data) {
        $newNode = new Node($data);
        $newNode->next = $this->head; // নতুন নোড বর্তমান head-কে দেখায়
        $this->head = $newNode;       // head আপডেট
        $this->size++;
    }

    // শেষে নোড যোগ — O(n)
    public function append($data) {
        $newNode = new Node($data);

        // লিস্ট খালি হলে head-ই হবে নতুন নোড
        if ($this->head === null) {
            $this->head = $newNode;
            $this->size++;
            return;
        }

        // শেষ নোড পর্যন্ত হাঁটো
        $current = $this->head;
        while ($current->next !== null) {
            $current = $current->next;
        }

        $current->next = $newNode; // শেষ নোডের next-এ নতুন নোড বসাও
        $this->size++;
    }

    // নির্দিষ্ট পজিশনে নোড যোগ — O(n)
    public function insertAt($position, $data) {
        // অবৈধ পজিশন চেক
        if ($position < 0 || $position > $this->size) {
            echo "❌ অবৈধ পজিশন!\n";
            return;
        }

        // শুরুতে insert
        if ($position === 0) {
            $this->prepend($data);
            return;
        }

        $newNode = new Node($data);
        $current = $this->head;

        // (position - 1) পর্যন্ত হাঁটো
        for ($i = 0; $i < $position - 1; $i++) {
            $current = $current->next;
        }

        // নতুন নোডের next = current এর next
        $newNode->next = $current->next;
        // current এর next = নতুন নোড
        $current->next = $newNode;
        $this->size++;
    }

    // শুরু থেকে নোড মুছুন — O(1)
    public function deleteFirst() {
        if ($this->head === null) {
            echo "❌ লিস্ট খালি!\n";
            return null;
        }

        $deletedData = $this->head->data;
        $this->head = $this->head->next; // head এগিয়ে দাও
        $this->size--;
        return $deletedData;
    }

    // শেষ থেকে নোড মুছুন — O(n)
    public function deleteLast() {
        if ($this->head === null) {
            echo "❌ লিস্ট খালি!\n";
            return null;
        }

        // শুধু একটি নোড থাকলে
        if ($this->head->next === null) {
            $deletedData = $this->head->data;
            $this->head = null;
            $this->size--;
            return $deletedData;
        }

        // শেষের আগের নোড পর্যন্ত হাঁটো
        $current = $this->head;
        while ($current->next->next !== null) {
            $current = $current->next;
        }

        $deletedData = $current->next->data;
        $current->next = null; // শেষ নোড বিচ্ছিন্ন
        $this->size--;
        return $deletedData;
    }

    // নির্দিষ্ট মান মুছুন — O(n)
    public function deleteValue($value) {
        if ($this->head === null) {
            return false;
        }

        // head-ই যদি সেই মান হয়
        if ($this->head->data === $value) {
            $this->head = $this->head->next;
            $this->size--;
            return true;
        }

        $current = $this->head;
        // মানটি খুঁজে তার আগের নোডে থামো
        while ($current->next !== null && $current->next->data !== $value) {
            $current = $current->next;
        }

        // মান পাওয়া গেলে
        if ($current->next !== null) {
            $current->next = $current->next->next; // মাঝের নোড বাদ
            $this->size--;
            return true;
        }

        return false; // মান পাওয়া যায়নি
    }

    // মান অনুসন্ধান — O(n)
    public function search($value) {
        $current = $this->head;
        $index = 0;

        while ($current !== null) {
            if ($current->data === $value) {
                return $index; // পাওয়া গেছে, ইনডেক্স ফেরত দাও
            }
            $current = $current->next;
            $index++;
        }

        return -1; // পাওয়া যায়নি
    }

    // সমস্ত নোড প্রদর্শন — O(n)
    public function display() {
        $current = $this->head;
        $result = "";

        while ($current !== null) {
            $result .= $current->data;
            if ($current->next !== null) {
                $result .= " -> ";
            }
            $current = $current->next;
        }

        echo $result . " -> NULL\n";
    }

    // লিস্টের দৈর্ঘ্য
    public function getSize() {
        return $this->size;
    }

    // লিস্ট খালি কিনা
    public function isEmpty() {
        return $this->head === null;
    }

    // head নোড ফেরত দাও (অন্যান্য অপারেশনের জন্য)
    public function getHead() {
        return $this->head;
    }
}

// ===== ব্যবহার =====
$list = new SinglyLinkedList();

$list->append(10);     // 10 -> NULL
$list->append(20);     // 10 -> 20 -> NULL
$list->append(30);     // 10 -> 20 -> 30 -> NULL
$list->prepend(5);     // 5 -> 10 -> 20 -> 30 -> NULL
$list->insertAt(2, 15); // 5 -> 10 -> 15 -> 20 -> 30 -> NULL

echo "লিস্ট: ";
$list->display();
// আউটপুট: 5 -> 10 -> 15 -> 20 -> 30 -> NULL

echo "মান 15 এর ইনডেক্স: " . $list->search(15) . "\n"; // 2
echo "সাইজ: " . $list->getSize() . "\n"; // 5

$list->deleteFirst();   // 10 -> 15 -> 20 -> 30 -> NULL
$list->deleteLast();    // 10 -> 15 -> 20 -> NULL
$list->deleteValue(15); // 10 -> 20 -> NULL

echo "মুছে ফেলার পর: ";
$list->display();
?>
```

---

### 💻 সম্পূর্ণ Implementation — JavaScript

```javascript
/**
 * সিঙ্গলি লিংকড লিস্ট — JavaScript Implementation
 * প্রতিটি নোডে data এবং next পয়েন্টার থাকে
 */

// নোড ক্লাস
class Node {
    constructor(data) {
        this.data = data;   // নোডের মান
        this.next = null;   // পরবর্তী নোডের রেফারেন্স
    }
}

// সিঙ্গলি লিংকড লিস্ট ক্লাস
class SinglyLinkedList {
    constructor() {
        this.head = null;  // প্রথম নোডের রেফারেন্স
        this.size = 0;     // লিস্টের মোট নোড সংখ্যা
    }

    // শুরুতে নোড যোগ — O(1)
    prepend(data) {
        const newNode = new Node(data);
        newNode.next = this.head; // নতুন নোড বর্তমান head-কে দেখায়
        this.head = newNode;      // head আপডেট
        this.size++;
    }

    // শেষে নোড যোগ — O(n)
    append(data) {
        const newNode = new Node(data);

        // লিস্ট খালি হলে
        if (!this.head) {
            this.head = newNode;
            this.size++;
            return;
        }

        // শেষ নোড পর্যন্ত হাঁটো
        let current = this.head;
        while (current.next) {
            current = current.next;
        }

        current.next = newNode; // শেষে সংযুক্ত
        this.size++;
    }

    // নির্দিষ্ট পজিশনে নোড যোগ — O(n)
    insertAt(position, data) {
        if (position < 0 || position > this.size) {
            console.log("❌ অবৈধ পজিশন!");
            return;
        }

        if (position === 0) {
            this.prepend(data);
            return;
        }

        const newNode = new Node(data);
        let current = this.head;

        // (position - 1) পর্যন্ত হাঁটো
        for (let i = 0; i < position - 1; i++) {
            current = current.next;
        }

        newNode.next = current.next;
        current.next = newNode;
        this.size++;
    }

    // শুরু থেকে নোড মুছুন — O(1)
    deleteFirst() {
        if (!this.head) {
            console.log("❌ লিস্ট খালি!");
            return null;
        }

        const deletedData = this.head.data;
        this.head = this.head.next;
        this.size--;
        return deletedData;
    }

    // শেষ থেকে নোড মুছুন — O(n)
    deleteLast() {
        if (!this.head) {
            console.log("❌ লিস্ট খালি!");
            return null;
        }

        if (!this.head.next) {
            const deletedData = this.head.data;
            this.head = null;
            this.size--;
            return deletedData;
        }

        let current = this.head;
        while (current.next.next) {
            current = current.next;
        }

        const deletedData = current.next.data;
        current.next = null;
        this.size--;
        return deletedData;
    }

    // নির্দিষ্ট মান মুছুন — O(n)
    deleteValue(value) {
        if (!this.head) return false;

        if (this.head.data === value) {
            this.head = this.head.next;
            this.size--;
            return true;
        }

        let current = this.head;
        while (current.next && current.next.data !== value) {
            current = current.next;
        }

        if (current.next) {
            current.next = current.next.next;
            this.size--;
            return true;
        }

        return false;
    }

    // মান অনুসন্ধান — O(n)
    search(value) {
        let current = this.head;
        let index = 0;

        while (current) {
            if (current.data === value) return index;
            current = current.next;
            index++;
        }

        return -1; // পাওয়া যায়নি
    }

    // সমস্ত নোড প্রদর্শন — O(n)
    display() {
        let current = this.head;
        const parts = [];

        while (current) {
            parts.push(current.data);
            current = current.next;
        }

        console.log(parts.join(" -> ") + " -> NULL");
    }

    // লিস্টের দৈর্ঘ্য
    getSize() {
        return this.size;
    }

    // লিস্ট খালি কিনা
    isEmpty() {
        return this.head === null;
    }
}

// ===== ব্যবহার =====
const list = new SinglyLinkedList();

list.append(10);       // 10 -> NULL
list.append(20);       // 10 -> 20 -> NULL
list.append(30);       // 10 -> 20 -> 30 -> NULL
list.prepend(5);       // 5 -> 10 -> 20 -> 30 -> NULL
list.insertAt(2, 15);  // 5 -> 10 -> 15 -> 20 -> 30 -> NULL

console.log("লিস্ট:");
list.display();
// আউটপুট: 5 -> 10 -> 15 -> 20 -> 30 -> NULL

console.log(`মান 15 এর ইনডেক্স: ${list.search(15)}`); // 2
console.log(`সাইজ: ${list.getSize()}`);                 // 5

list.deleteFirst();     // 10 -> 15 -> 20 -> 30 -> NULL
list.deleteLast();      // 10 -> 15 -> 20 -> NULL
list.deleteValue(15);   // 10 -> 20 -> NULL

console.log("মুছে ফেলার পর:");
list.display();
```

---

## ২. Doubly Linked List

### 📐 নোড স্ট্রাকচার

প্রতিটি নোডে তিনটি ফিল্ড:

```
┌──────────┬──────────┬──────────┐
│   prev   │   data   │   next   │
│(পূর্ববর্তী)│  (মান)   │(পরবর্তী) │
└──────────┴──────────┴──────────┘
```

**সম্পূর্ণ ডাবলি লিংকড লিস্ট:**

```
  head                                                        tail
   │                                                           │
   ▼                                                           ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ NULL ← 10 → ─┼────►│ ← 20 → ─────┼────►│ ← 30 → NULL │
│              │◄────┼──────        │◄────┼──────        │
└──────────────┘     └──────────────┘     └──────────────┘

বিস্তারিত ভিউ:
NULL ◄──[prev|10|next]──► ◄──[prev|20|next]──► ◄──[prev|30|next]──► NULL
```

### 💡 সুবিধা

```
✅ Bidirectional Traversal — সামনে ও পেছনে দুদিকে যাওয়া যায়
✅ O(1) Delete — নোডের রেফারেন্স জানা থাকলে সরাসরি মুছতে পারি
✅ Tail থেকে পেছনে হাঁটা সহজ

❌ বেশি মেমোরি লাগে — প্রতিটি নোডে একটি extra pointer (prev)
❌ Insert/Delete-এ বেশি pointer update করতে হয়
```

### 🌐 Use Case: ব্রাউজার হিস্টোরি (Back/Forward)

```
  ◄── Back                                    Forward ──►

  [google.com] ◄──► [github.com] ◄──► [stackoverflow.com]
                                           ↑
                                       বর্তমান পেজ

  Back চাপলে:  current = current.prev  → github.com
  Forward চাপলে: current = current.next → stackoverflow.com
```

---

### 💻 Doubly Linked List — PHP

```php
<?php
/**
 * ডাবলি লিংকড লিস্ট — PHP Implementation
 * প্রতিটি নোডে prev, data এবং next পয়েন্টার থাকে
 */

// ডাবলি নোড — দুদিকে পয়েন্টার আছে
class DoublyNode {
    public $data;
    public $prev;  // পূর্ববর্তী নোড
    public $next;  // পরবর্তী নোড

    public function __construct($data) {
        $this->data = $data;
        $this->prev = null;
        $this->next = null;
    }
}

class DoublyLinkedList {
    private $head;  // প্রথম নোড
    private $tail;  // শেষ নোড
    private $size;

    public function __construct() {
        $this->head = null;
        $this->tail = null;
        $this->size = 0;
    }

    // শুরুতে নোড যোগ — O(1)
    public function prepend($data) {
        $newNode = new DoublyNode($data);

        if ($this->head === null) {
            // লিস্ট খালি — নতুন নোডই head এবং tail
            $this->head = $newNode;
            $this->tail = $newNode;
        } else {
            $newNode->next = $this->head;  // নতুন নোড পুরানো head-কে দেখায়
            $this->head->prev = $newNode;  // পুরানো head পেছনে নতুন নোডকে দেখায়
            $this->head = $newNode;        // head আপডেট
        }
        $this->size++;
    }

    // শেষে নোড যোগ — O(1) কারণ tail পয়েন্টার আছে
    public function append($data) {
        $newNode = new DoublyNode($data);

        if ($this->tail === null) {
            $this->head = $newNode;
            $this->tail = $newNode;
        } else {
            $newNode->prev = $this->tail;  // নতুন নোড পুরানো tail-কে দেখায়
            $this->tail->next = $newNode;  // পুরানো tail সামনে নতুন নোডকে দেখায়
            $this->tail = $newNode;        // tail আপডেট
        }
        $this->size++;
    }

    // নির্দিষ্ট পজিশনে insert — O(n)
    public function insertAt($position, $data) {
        if ($position < 0 || $position > $this->size) {
            echo "❌ অবৈধ পজিশন!\n";
            return;
        }

        if ($position === 0) {
            $this->prepend($data);
            return;
        }

        if ($position === $this->size) {
            $this->append($data);
            return;
        }

        $newNode = new DoublyNode($data);
        $current = $this->head;

        for ($i = 0; $i < $position; $i++) {
            $current = $current->next;
        }

        // current এর আগে newNode বসাও
        $newNode->prev = $current->prev;
        $newNode->next = $current;
        $current->prev->next = $newNode;
        $current->prev = $newNode;
        $this->size++;
    }

    // নির্দিষ্ট নোড মুছুন — O(1) যদি নোড রেফারেন্স জানা থাকে
    public function deleteNode($node) {
        if ($node === null) return;

        // head মুছলে
        if ($node === $this->head) {
            $this->head = $node->next;
            if ($this->head) {
                $this->head->prev = null;
            } else {
                $this->tail = null; // লিস্ট খালি হয়ে গেছে
            }
        }
        // tail মুছলে
        elseif ($node === $this->tail) {
            $this->tail = $node->prev;
            $this->tail->next = null;
        }
        // মাঝের নোড মুছলে
        else {
            $node->prev->next = $node->next;
            $node->next->prev = $node->prev;
        }

        $this->size--;
    }

    // মান দিয়ে মুছুন — O(n)
    public function deleteValue($value) {
        $current = $this->head;

        while ($current !== null) {
            if ($current->data === $value) {
                $this->deleteNode($current);
                return true;
            }
            $current = $current->next;
        }

        return false;
    }

    // সামনে থেকে প্রদর্শন
    public function displayForward() {
        $current = $this->head;
        $parts = [];
        while ($current !== null) {
            $parts[] = $current->data;
            $current = $current->next;
        }
        echo "সামনে: NULL ◄── " . implode(" ◄──► ", $parts) . " ──► NULL\n";
    }

    // পেছন থেকে প্রদর্শন
    public function displayBackward() {
        $current = $this->tail;
        $parts = [];
        while ($current !== null) {
            $parts[] = $current->data;
            $current = $current->prev;
        }
        echo "পেছনে: NULL ◄── " . implode(" ◄──► ", $parts) . " ──► NULL\n";
    }

    public function getSize() { return $this->size; }
    public function getHead() { return $this->head; }
    public function getTail() { return $this->tail; }
}

// ===== ব্যবহার =====
$dll = new DoublyLinkedList();
$dll->append(10);
$dll->append(20);
$dll->append(30);
$dll->prepend(5);

$dll->displayForward();
// সামনে: NULL ◄── 5 ◄──► 10 ◄──► 20 ◄──► 30 ──► NULL

$dll->displayBackward();
// পেছনে: NULL ◄── 30 ◄──► 20 ◄──► 10 ◄──► 5 ──► NULL

$dll->deleteValue(20);
$dll->displayForward();
// সামনে: NULL ◄── 5 ◄──► 10 ◄──► 30 ──► NULL
?>
```

---

### 💻 Doubly Linked List — JavaScript

```javascript
/**
 * ডাবলি লিংকড লিস্ট — JavaScript Implementation
 */

class DoublyNode {
    constructor(data) {
        this.data = data;
        this.prev = null;  // পূর্ববর্তী নোড
        this.next = null;  // পরবর্তী নোড
    }
}

class DoublyLinkedList {
    constructor() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }

    // শুরুতে নোড যোগ — O(1)
    prepend(data) {
        const newNode = new DoublyNode(data);

        if (!this.head) {
            this.head = newNode;
            this.tail = newNode;
        } else {
            newNode.next = this.head;
            this.head.prev = newNode;
            this.head = newNode;
        }
        this.size++;
    }

    // শেষে নোড যোগ — O(1)
    append(data) {
        const newNode = new DoublyNode(data);

        if (!this.tail) {
            this.head = newNode;
            this.tail = newNode;
        } else {
            newNode.prev = this.tail;
            this.tail.next = newNode;
            this.tail = newNode;
        }
        this.size++;
    }

    // নির্দিষ্ট পজিশনে insert — O(n)
    insertAt(position, data) {
        if (position < 0 || position > this.size) {
            console.log("❌ অবৈধ পজিশন!");
            return;
        }

        if (position === 0) { this.prepend(data); return; }
        if (position === this.size) { this.append(data); return; }

        const newNode = new DoublyNode(data);
        let current = this.head;

        for (let i = 0; i < position; i++) {
            current = current.next;
        }

        newNode.prev = current.prev;
        newNode.next = current;
        current.prev.next = newNode;
        current.prev = newNode;
        this.size++;
    }

    // নোড মুছুন — O(1) যদি নোড রেফারেন্স জানা থাকে
    deleteNode(node) {
        if (!node) return;

        if (node === this.head) {
            this.head = node.next;
            if (this.head) this.head.prev = null;
            else this.tail = null;
        } else if (node === this.tail) {
            this.tail = node.prev;
            this.tail.next = null;
        } else {
            node.prev.next = node.next;
            node.next.prev = node.prev;
        }
        this.size--;
    }

    // মান দিয়ে মুছুন — O(n)
    deleteValue(value) {
        let current = this.head;
        while (current) {
            if (current.data === value) {
                this.deleteNode(current);
                return true;
            }
            current = current.next;
        }
        return false;
    }

    // সামনে থেকে প্রদর্শন
    displayForward() {
        let current = this.head;
        const parts = [];
        while (current) {
            parts.push(current.data);
            current = current.next;
        }
        console.log("সামনে: NULL ◄── " + parts.join(" ◄──► ") + " ──► NULL");
    }

    // পেছন থেকে প্রদর্শন
    displayBackward() {
        let current = this.tail;
        const parts = [];
        while (current) {
            parts.push(current.data);
            current = current.prev;
        }
        console.log("পেছনে: NULL ◄── " + parts.join(" ◄──► ") + " ──► NULL");
    }

    getSize() { return this.size; }
}

// ===== ব্রাউজার হিস্টোরি সিমুলেশন =====
class BrowserHistory {
    constructor(homepage) {
        this.list = new DoublyLinkedList();
        this.list.append(homepage);
        this.current = this.list.head; // বর্তমান পেজ
    }

    // নতুন পেজে যাও — forward হিস্টোরি মুছে যায়
    visit(url) {
        const newNode = new DoublyNode(url);
        newNode.prev = this.current;
        this.current.next = newNode;
        this.current = newNode;
        console.log(`🌐 পরিদর্শন: ${url}`);
    }

    // পেছনে যাও
    back(steps) {
        while (steps > 0 && this.current.prev) {
            this.current = this.current.prev;
            steps--;
        }
        console.log(`◄ পেছনে: ${this.current.data}`);
        return this.current.data;
    }

    // সামনে যাও
    forward(steps) {
        while (steps > 0 && this.current.next) {
            this.current = this.current.next;
            steps--;
        }
        console.log(`► সামনে: ${this.current.data}`);
        return this.current.data;
    }
}

// ব্যবহার
const browser = new BrowserHistory("google.com");
browser.visit("github.com");        // 🌐 পরিদর্শন: github.com
browser.visit("stackoverflow.com"); // 🌐 পরিদর্শন: stackoverflow.com
browser.back(1);                     // ◄ পেছনে: github.com
browser.back(1);                     // ◄ পেছনে: google.com
browser.forward(1);                  // ► সামনে: github.com
```

---

## ৩. Circular Linked List

### 📐 Singly Circular Linked List

শেষ নোডের `next` → NULL নয়, বরং `head`-কে দেখায়:

```
        head
         │
         ▼
      ┌──────┐     ┌──────┐     ┌──────┐
  ┌──►│  10  │────►│  20  │────►│  30  │───┐
  │   └──────┘     └──────┘     └──────┘   │
  │                                         │
  └─────────────────────────────────────────┘
         শেষ নোড আবার head-এ ফিরে আসে
```

### 📐 Doubly Circular Linked List

```
         head
          │
          ▼
  ┌───[prev|10|next]◄──►[prev|20|next]◄──►[prev|30|next]───┐
  │                                                          │
  └──────────────────────────────────────────────────────────┘
  head.prev = শেষ নোড, শেষ নোড.next = head
```

### 🎯 Use Cases

```
🔄 Round-Robin Scheduling:
  প্রতিটি প্রসেসকে একটি নির্দিষ্ট সময় (quantum) দেওয়া হয়,
  শেষ হলে পরের প্রসেসে যায়, সবার শেষে আবার প্রথম থেকে শুরু।

  [P1] -> [P2] -> [P3] -> [P1] -> [P2] -> ...

🎵 Music Playlist (Repeat):
  শেষ গানের পর আবার প্রথম গান বাজে।

  [Song1] -> [Song2] -> [Song3] -> [Song1] -> ...

♠️ Card Game:
  খেলোয়াড়দের মধ্যে ঘুরে ঘুরে পালা আসে।
```

---

### 💻 Circular Linked List — PHP

```php
<?php
/**
 * সার্কুলার লিংকড লিস্ট — PHP Implementation
 * শেষ নোডের next আবার head-কে দেখায়
 */

class CircularNode {
    public $data;
    public $next;

    public function __construct($data) {
        $this->data = $data;
        $this->next = null;
    }
}

class CircularLinkedList {
    private $head;
    private $tail;  // শেষ নোডের রেফারেন্স (সুবিধার জন্য)
    private $size;

    public function __construct() {
        $this->head = null;
        $this->tail = null;
        $this->size = 0;
    }

    // শেষে নোড যোগ — O(1)
    public function append($data) {
        $newNode = new CircularNode($data);

        if ($this->head === null) {
            $this->head = $newNode;
            $this->tail = $newNode;
            $newNode->next = $newNode; // নিজেকেই দেখায় (একমাত্র নোড)
        } else {
            $newNode->next = $this->head;   // নতুন নোড head-কে দেখায়
            $this->tail->next = $newNode;   // পুরানো tail নতুন নোডকে দেখায়
            $this->tail = $newNode;         // tail আপডেট
        }
        $this->size++;
    }

    // শুরুতে নোড যোগ — O(1)
    public function prepend($data) {
        $newNode = new CircularNode($data);

        if ($this->head === null) {
            $this->head = $newNode;
            $this->tail = $newNode;
            $newNode->next = $newNode;
        } else {
            $newNode->next = $this->head;
            $this->tail->next = $newNode;   // tail এখন নতুন head-কে দেখায়
            $this->head = $newNode;
        }
        $this->size++;
    }

    // মান মুছুন — O(n)
    public function deleteValue($value) {
        if ($this->head === null) return false;

        // শুধু একটি নোড থাকলে
        if ($this->size === 1 && $this->head->data === $value) {
            $this->head = null;
            $this->tail = null;
            $this->size--;
            return true;
        }

        // head মুছলে
        if ($this->head->data === $value) {
            $this->head = $this->head->next;
            $this->tail->next = $this->head;
            $this->size--;
            return true;
        }

        // মাঝে বা শেষে খুঁজে মুছুন
        $current = $this->head;
        while ($current->next !== $this->head) {
            if ($current->next->data === $value) {
                if ($current->next === $this->tail) {
                    $this->tail = $current;
                }
                $current->next = $current->next->next;
                $this->size--;
                return true;
            }
            $current = $current->next;
        }

        return false;
    }

    // সমস্ত নোড প্রদর্শন
    public function display() {
        if ($this->head === null) {
            echo "(খালি)\n";
            return;
        }

        $current = $this->head;
        $parts = [];

        do {
            $parts[] = $current->data;
            $current = $current->next;
        } while ($current !== $this->head);

        echo implode(" -> ", $parts) . " -> [HEAD:" . $this->head->data . "] (বৃত্তাকার)\n";
    }

    public function getSize() { return $this->size; }
    public function getHead() { return $this->head; }
}

// ===== Round-Robin সিমুলেশন =====
function roundRobinDemo() {
    $cll = new CircularLinkedList();
    $cll->append("P1");
    $cll->append("P2");
    $cll->append("P3");

    echo "🔄 Round-Robin প্রসেস শিডিউলিং:\n";
    $cll->display();

    // ৩ রাউন্ড সিমুলেট
    $current = $cll->getHead();
    for ($round = 1; $round <= 9; $round++) {
        echo "রাউন্ড $round: {$current->data} চলছে...\n";
        $current = $current->next;
    }
}

roundRobinDemo();
?>
```

---

### 💻 Circular Linked List — JavaScript

```javascript
/**
 * সার্কুলার লিংকড লিস্ট — JavaScript Implementation
 */

class CircularNode {
    constructor(data) {
        this.data = data;
        this.next = null;
    }
}

class CircularLinkedList {
    constructor() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }

    // শেষে নোড যোগ — O(1)
    append(data) {
        const newNode = new CircularNode(data);

        if (!this.head) {
            this.head = newNode;
            this.tail = newNode;
            newNode.next = newNode; // নিজেকেই দেখায়
        } else {
            newNode.next = this.head;
            this.tail.next = newNode;
            this.tail = newNode;
        }
        this.size++;
    }

    // শুরুতে নোড যোগ — O(1)
    prepend(data) {
        const newNode = new CircularNode(data);

        if (!this.head) {
            this.head = newNode;
            this.tail = newNode;
            newNode.next = newNode;
        } else {
            newNode.next = this.head;
            this.tail.next = newNode;
            this.head = newNode;
        }
        this.size++;
    }

    // মান মুছুন — O(n)
    deleteValue(value) {
        if (!this.head) return false;

        if (this.size === 1 && this.head.data === value) {
            this.head = null;
            this.tail = null;
            this.size--;
            return true;
        }

        if (this.head.data === value) {
            this.head = this.head.next;
            this.tail.next = this.head;
            this.size--;
            return true;
        }

        let current = this.head;
        while (current.next !== this.head) {
            if (current.next.data === value) {
                if (current.next === this.tail) {
                    this.tail = current;
                }
                current.next = current.next.next;
                this.size--;
                return true;
            }
            current = current.next;
        }

        return false;
    }

    // প্রদর্শন
    display() {
        if (!this.head) {
            console.log("(খালি)");
            return;
        }

        let current = this.head;
        const parts = [];

        do {
            parts.push(current.data);
            current = current.next;
        } while (current !== this.head);

        console.log(parts.join(" -> ") + ` -> [HEAD:${this.head.data}] (বৃত্তাকার)`);
    }
}

// ===== 🎵 মিউজিক প্লেলিস্ট সিমুলেশন =====
class MusicPlaylist {
    constructor() {
        this.list = new CircularLinkedList();
        this.current = null; // বর্তমানে বাজছে
    }

    addSong(name) {
        this.list.append(name);
        if (!this.current) {
            this.current = this.list.head;
        }
    }

    // পরের গান
    playNext() {
        if (!this.current) {
            console.log("🔇 প্লেলিস্ট খালি!");
            return;
        }
        this.current = this.current.next;
        console.log(`🎵 এখন বাজছে: ${this.current.data}`);
    }

    // বর্তমান গান
    nowPlaying() {
        if (this.current) {
            console.log(`🎵 এখন বাজছে: ${this.current.data}`);
        }
    }
}

// ব্যবহার
const playlist = new MusicPlaylist();
playlist.addSong("আমার সোনার বাংলা");
playlist.addSong("একটি বাংলাদেশ");
playlist.addSong("আমি বাংলায় গান গাই");

playlist.nowPlaying();  // 🎵 এখন বাজছে: আমার সোনার বাংলা
playlist.playNext();    // 🎵 এখন বাজছে: একটি বাংলাদেশ
playlist.playNext();    // 🎵 এখন বাজছে: আমি বাংলায় গান গাই
playlist.playNext();    // 🎵 এখন বাজছে: আমার সোনার বাংলা (আবার শুরু!)
```

---

## ৪. লিংকড লিস্ট টেকনিক

### 🏃 ৪.১ Runner (Fast/Slow Pointer) টেকনিক

দুটি পয়েন্টার ব্যবহার করা হয় — একটি ধীরে (slow, ১ ধাপ), একটি দ্রুত (fast, ২ ধাপ)।

#### মাঝের নোড খুঁজুন

```
লিস্ট: 1 -> 2 -> 3 -> 4 -> 5 -> NULL

ধাপ ০:  slow=1, fast=1
         ↓S         ↓F
         [1] -> [2] -> [3] -> [4] -> [5] -> NULL

ধাপ ১:  slow=2, fast=3
                ↓S          ↓F
         [1] -> [2] -> [3] -> [4] -> [5] -> NULL

ধাপ ২:  slow=3, fast=5
                       ↓S                ↓F
         [1] -> [2] -> [3] -> [4] -> [5] -> NULL

fast শেষে পৌঁছেছে → slow-ই মাঝের নোড = 3 ✅

যুক্তি: fast দ্বিগুণ গতিতে চলে, তাই fast শেষে পৌঁছালে
        slow ঠিক মাঝখানে থাকবে।
```

#### সাইকেল ডিটেকশন (Floyd's Algorithm)

```
সাইকেল আছে:
  [1] -> [2] -> [3] -> [4] -> [5]
                  ↑                │
                  └────────────────┘

ধাপ ০: slow=1, fast=1
ধাপ ১: slow=2, fast=3
ধাপ ২: slow=3, fast=5
ধাপ ৩: slow=4, fast=4  ← মিলে গেছে! সাইকেল আছে ✅

সাইকেল না থাকলে fast NULL-এ পৌঁছে যাবে।
```

---

### 🤖 ৪.২ Dummy Node টেকনিক

**সমস্যা:** head-এ insert/delete করলে বিশেষ case handle করতে হয়।

**সমাধান:** একটি "ডামি" নোড head-এর আগে রাখো — এতে সব অপারেশন একই নিয়মে চলে।

```
ডামি নোড ছাড়া:
  if (head === null) { ... }   // বিশেষ case
  if (position === 0) { ... }  // বিশেষ case

ডামি নোড সহ:
  dummy -> [real nodes...]

  dummy.next = head
  সব সময় prev নোড পাওয়া যায় (dummy নিজেই prev হতে পারে)
  কোনো বিশেষ case নেই!

উদাহরণ — মান 10 মুছুন:
  আগে: dummy -> [10] -> [20] -> [30] -> NULL
  পরে: dummy -> [20] -> [30] -> NULL
  return dummy.next (প্রকৃত head)
```

---

### 🔄 ৪.৩ Recursive Approach

লিংকড লিস্টের প্রকৃতি recursive — প্রতিটি নোড "নিজে + বাকি লিস্ট"।

```
কখন Recursion ব্যবহার করবেন:
✅ Reverse — সুন্দর recursive সমাধান
✅ Merge — দুটি sorted লিস্ট merge
✅ Print backward — পেছন থেকে প্রিন্ট

কখন এড়িয়ে চলবেন:
❌ অনেক বড় লিস্ট — stack overflow হতে পারে
❌ Simple traverse — iterative ভালো

মূল ধারণা:
  function process(node):
      if node === null: return  // Base case
      // node নিয়ে কিছু করো
      process(node.next)        // বাকি লিস্ট process করো
```

---

## ৫. ক্লাসিক সমস্যা (Complex Problems)

### 🔃 ৫.১ Reverse Linked List

#### ASCII Step-by-Step Trace (Iterative)

```
আগে:  1 -> 2 -> 3 -> 4 -> NULL

ধাপ ০: prev=NULL, current=1, next=?
        NULL    [1] -> [2] -> [3] -> [4] -> NULL
         ↑       ↑
        prev   current

ধাপ ১: next = current.next (2)
        current.next = prev (NULL)
        prev = current (1)
        current = next (2)

        NULL <- [1]    [2] -> [3] -> [4] -> NULL
                 ↑      ↑
               prev   current

ধাপ ২: next = current.next (3)
        current.next = prev (1)
        prev = current (2)
        current = next (3)

        NULL <- [1] <- [2]    [3] -> [4] -> NULL
                        ↑      ↑
                      prev   current

ধাপ ৩: next = current.next (4)
        current.next = prev (2)
        prev = current (3)
        current = next (4)

        NULL <- [1] <- [2] <- [3]    [4] -> NULL
                                ↑      ↑
                              prev   current

ধাপ ৪: next = current.next (NULL)
        current.next = prev (3)
        prev = current (4)
        current = next (NULL)

        NULL <- [1] <- [2] <- [3] <- [4]    NULL
                                      ↑      ↑
                                    prev   current

current === NULL → থামো, head = prev

পরে:  4 -> 3 -> 2 -> 1 -> NULL ✅
```

#### PHP — Iterative + Recursive

```php
<?php
// Reverse Linked List — Iterative পদ্ধতি — O(n) time, O(1) space
function reverseIterative($head) {
    $prev = null;
    $current = $head;

    while ($current !== null) {
        $next = $current->next;     // পরের নোড সংরক্ষণ
        $current->next = $prev;     // লিংক উল্টাও
        $prev = $current;           // prev এগিয়ে দাও
        $current = $next;           // current এগিয়ে দাও
    }

    return $prev; // নতুন head
}

// Reverse Linked List — Recursive পদ্ধতি — O(n) time, O(n) space (call stack)
function reverseRecursive($head) {
    // Base case: খালি বা একটি মাত্র নোড
    if ($head === null || $head->next === null) {
        return $head;
    }

    // বাকি লিস্ট reverse করো
    $newHead = reverseRecursive($head->next);

    // বর্তমান নোডের পরের নোডকে বর্তমানের দিকে ঘোরাও
    $head->next->next = $head;
    $head->next = null;

    return $newHead;
}
?>
```

#### JavaScript — Iterative + Recursive

```javascript
// Reverse Linked List — Iterative — O(n) time, O(1) space
function reverseIterative(head) {
    let prev = null;
    let current = head;

    while (current) {
        const next = current.next;  // পরের নোড সংরক্ষণ
        current.next = prev;        // লিংক উল্টাও
        prev = current;             // prev এগিয়ে দাও
        current = next;             // current এগিয়ে দাও
    }

    return prev; // নতুন head
}

// Reverse Linked List — Recursive — O(n) time, O(n) space
function reverseRecursive(head) {
    if (!head || !head.next) return head;

    const newHead = reverseRecursive(head.next);
    head.next.next = head;
    head.next = null;

    return newHead;
}
```

---

### 🔍 ৫.২ Detect Cycle — Floyd's Tortoise and Hare

#### গাণিতিক প্রমাণ

```
ধরি:
  - সাইকেলের শুরু পর্যন্ত দূরত্ব = L
  - সাইকেলের দৈর্ঘ্য = C
  - slow এবং fast যেখানে মিলেছে, সেটা সাইকেলের শুরু থেকে K দূরে

  slow চলেছে: L + K ধাপ
  fast চলেছে: L + K + nC ধাপ (n বার সাইকেল ঘুরেছে)

  fast = 2 × slow (দ্বিগুণ গতি):
    L + K + nC = 2(L + K)
    nC = L + K
    L = nC - K

  অর্থাৎ: মিলনস্থল থেকে L ধাপ হাঁটলে সাইকেলের শুরুতে পৌঁছায়!
  একইসাথে head থেকেও L ধাপ হাঁটলে সাইকেলের শুরুতে পৌঁছায়!

  তাহলে: একটি পয়েন্টার head-এ, আরেকটি মিলনস্থলে রেখে
          দুজনে ১ ধাপ করে হাঁটলে সাইকেলের শুরুতে মিলবে!
```

```
উদাহরণ:
    1 -> 2 -> 3 -> 4 -> 5
                   ↑         │
                   └─────────┘

    L = 2 (1,2 — সাইকেলের আগে)
    C = 3 (3,4,5 — সাইকেলের দৈর্ঘ্য)

    slow: 1,2,3,4,5,3,4  (৬ ধাপ)
    fast: 1,3,5,4,3,5,4  (১২ ধাপ, ৬ বার ২ ধাপ)
    মিলনস্থল: 4

    এখন: ptr1 = head (1), ptr2 = মিলনস্থল (4)
    ধাপ ১: ptr1=2, ptr2=5
    ধাপ ২: ptr1=3, ptr2=3 ← মিলেছে! সাইকেলের শুরু = 3 ✅
```

#### PHP Implementation

```php
<?php
// সাইকেল আছে কিনা — O(n) time, O(1) space
function hasCycle($head) {
    $slow = $head;
    $fast = $head;

    while ($fast !== null && $fast->next !== null) {
        $slow = $slow->next;         // ১ ধাপ
        $fast = $fast->next->next;   // ২ ধাপ

        if ($slow === $fast) {
            return true; // সাইকেল পাওয়া গেছে
        }
    }

    return false; // fast NULL-এ পৌঁছেছে — সাইকেল নেই
}

// সাইকেলের শুরু খুঁজুন — O(n) time, O(1) space
function detectCycleStart($head) {
    $slow = $head;
    $fast = $head;

    // প্রথমে মিলনস্থল খুঁজুন
    while ($fast !== null && $fast->next !== null) {
        $slow = $slow->next;
        $fast = $fast->next->next;

        if ($slow === $fast) {
            // মিলনস্থল পাওয়া গেছে
            // এখন head থেকে এবং মিলনস্থল থেকে ১ ধাপে হাঁটো
            $ptr1 = $head;
            $ptr2 = $slow;

            while ($ptr1 !== $ptr2) {
                $ptr1 = $ptr1->next;
                $ptr2 = $ptr2->next;
            }

            return $ptr1; // সাইকেলের শুরু নোড
        }
    }

    return null; // সাইকেল নেই
}
?>
```

#### JavaScript Implementation

```javascript
// সাইকেল আছে কিনা — O(n) time, O(1) space
function hasCycle(head) {
    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow === fast) return true;
    }

    return false;
}

// সাইকেলের শুরু খুঁজুন
function detectCycleStart(head) {
    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow === fast) {
            let ptr1 = head;
            let ptr2 = slow;

            while (ptr1 !== ptr2) {
                ptr1 = ptr1.next;
                ptr2 = ptr2.next;
            }
            return ptr1;
        }
    }

    return null;
}
```

---

### 🔀 ৫.৩ Merge Two Sorted Lists

```
লিস্ট ১: 1 -> 3 -> 5 -> NULL
লিস্ট ২: 2 -> 4 -> 6 -> NULL

মার্জ:   1 -> 2 -> 3 -> 4 -> 5 -> 6 -> NULL

ধাপে ধাপে:
  তুলনা 1 vs 2: 1 ছোট → নাও     → [1]
  তুলনা 3 vs 2: 2 ছোট → নাও     → [1,2]
  তুলনা 3 vs 4: 3 ছোট → নাও     → [1,2,3]
  তুলনা 5 vs 4: 4 ছোট → নাও     → [1,2,3,4]
  তুলনা 5 vs 6: 5 ছোট → নাও     → [1,2,3,4,5]
  লিস্ট ১ শেষ, বাকি লিস্ট ২ যোগ → [1,2,3,4,5,6]
```

#### PHP

```php
<?php
// Iterative — Dummy Node টেকনিক ব্যবহার — O(n+m) time, O(1) space
function mergeTwoListsIterative($l1, $l2) {
    $dummy = new Node(0);   // ডামি নোড — edge case সহজ করে
    $current = $dummy;

    while ($l1 !== null && $l2 !== null) {
        if ($l1->data <= $l2->data) {
            $current->next = $l1;  // l1 ছোট — l1 নাও
            $l1 = $l1->next;
        } else {
            $current->next = $l2;  // l2 ছোট — l2 নাও
            $l2 = $l2->next;
        }
        $current = $current->next;
    }

    // যে লিস্ট বাকি আছে সেটা জুড়ে দাও
    $current->next = $l1 !== null ? $l1 : $l2;

    return $dummy->next; // ডামির পরবর্তী = প্রকৃত head
}

// Recursive — O(n+m) time, O(n+m) space (call stack)
function mergeTwoListsRecursive($l1, $l2) {
    if ($l1 === null) return $l2; // l1 শেষ — বাকি l2 ফেরত দাও
    if ($l2 === null) return $l1; // l2 শেষ — বাকি l1 ফেরত দাও

    if ($l1->data <= $l2->data) {
        $l1->next = mergeTwoListsRecursive($l1->next, $l2);
        return $l1;
    } else {
        $l2->next = mergeTwoListsRecursive($l1, $l2->next);
        return $l2;
    }
}
?>
```

#### JavaScript

```javascript
// Iterative — O(n+m) time, O(1) space
function mergeTwoListsIterative(l1, l2) {
    const dummy = new Node(0);
    let current = dummy;

    while (l1 && l2) {
        if (l1.data <= l2.data) {
            current.next = l1;
            l1 = l1.next;
        } else {
            current.next = l2;
            l2 = l2.next;
        }
        current = current.next;
    }

    current.next = l1 || l2;
    return dummy.next;
}

// Recursive — O(n+m) time, O(n+m) space
function mergeTwoListsRecursive(l1, l2) {
    if (!l1) return l2;
    if (!l2) return l1;

    if (l1.data <= l2.data) {
        l1.next = mergeTwoListsRecursive(l1.next, l2);
        return l1;
    } else {
        l2.next = mergeTwoListsRecursive(l1, l2.next);
        return l2;
    }
}
```

---

### 🎯 ৫.৪ Remove Nth Node from End — Two Pointer, One Pass

```
লিস্ট: 1 -> 2 -> 3 -> 4 -> 5 -> NULL
শেষ থেকে ২য় নোড (4) মুছতে হবে (n=2)

কৌশল: দুটি পয়েন্টার — fast আগে n ধাপ এগিয়ে যাবে,
       তারপর দুজনে একসাথে হাঁটবে। fast শেষে পৌঁছালে
       slow শেষ থেকে n-তম নোডের ঠিক আগে থাকবে।

ধাপ ০: dummy -> [1] -> [2] -> [3] -> [4] -> [5] -> NULL
        ↑s        ↑f

ধাপ ১: fast ২ ধাপ এগিয়ে যাক (n=2)
        dummy -> [1] -> [2] -> [3] -> [4] -> [5] -> NULL
        ↑s                      ↑f

ধাপ ২: দুজনে একসাথে হাঁটো
        dummy -> [1] -> [2] -> [3] -> [4] -> [5] -> NULL
                  ↑s                   ↑f

ধাপ ৩:
        dummy -> [1] -> [2] -> [3] -> [4] -> [5] -> NULL
                         ↑s                   ↑f

ধাপ ৪:
        dummy -> [1] -> [2] -> [3] -> [4] -> [5] -> NULL
                                ↑s                    ↑f(NULL)

fast = NULL → থামো
slow.next = slow.next.next (3 এর next = 5, 4 বাদ)

ফল: 1 -> 2 -> 3 -> 5 -> NULL ✅
```

#### PHP

```php
<?php
// শেষ থেকে n-তম নোড মুছুন — O(n) time, O(1) space, একবার traverse
function removeNthFromEnd($head, $n) {
    $dummy = new Node(0);
    $dummy->next = $head;
    $fast = $dummy;
    $slow = $dummy;

    // fast-কে n+1 ধাপ এগিয়ে দাও
    for ($i = 0; $i <= $n; $i++) {
        $fast = $fast->next;
    }

    // দুজনে একসাথে হাঁটো
    while ($fast !== null) {
        $slow = $slow->next;
        $fast = $fast->next;
    }

    // slow->next হলো মুছতে হবে এমন নোড
    $slow->next = $slow->next->next;

    return $dummy->next;
}
?>
```

#### JavaScript

```javascript
// শেষ থেকে n-তম নোড মুছুন — O(n) time, O(1) space
function removeNthFromEnd(head, n) {
    const dummy = new Node(0);
    dummy.next = head;
    let fast = dummy;
    let slow = dummy;

    // fast-কে n+1 ধাপ এগিয়ে দাও
    for (let i = 0; i <= n; i++) {
        fast = fast.next;
    }

    // দুজনে একসাথে হাঁটো
    while (fast) {
        slow = slow.next;
        fast = fast.next;
    }

    slow.next = slow.next.next;
    return dummy.next;
}
```

---

### 🧠 ৫.৫ LRU Cache — Doubly Linked List + HashMap

#### ধারণা

**LRU (Least Recently Used) Cache** — সবচেয়ে কম সম্প্রতি ব্যবহৃত আইটেম প্রথমে বাদ দেয়।

```
প্রয়োজনীয়তা:
  get(key)  → O(1) — মান ফেরত দাও, আইটেমকে "সম্প্রতি ব্যবহৃত" হিসেবে চিহ্নিত করো
  put(key, value) → O(1) — নতুন আইটেম যোগ করো, ক্যাপাসিটি ভরে গেলে LRU বাদ দাও

কৌশল:
  HashMap → O(1) lookup (key → node)
  Doubly Linked List → O(1) insert/delete, সর্বশেষ ব্যবহৃত ক্রম রক্ষা
```

#### বিস্তারিত ASCII ডায়াগ্রাম

```
ক্যাপাসিটি: 3

HashMap:                     Doubly Linked List:
┌─────┬───────┐
│ key │ node* │             HEAD ◄──► [সবচেয়ে সম্প্রতি]
├─────┼───────┤                        ↕
│  1  │   ──────────►       [মাঝে]
│  2  │   ──────────►         ↕
│  3  │   ──────────►       [সবচেয়ে পুরানো] ◄──► TAIL
└─────┴───────┘

get(2) করলে:
  ১. HashMap থেকে node পাও — O(1)
  ২. node-কে বর্তমান জায়গা থেকে সরাও — O(1)
  ৩. node-কে head-এর পরে বসাও (সবচেয়ে সম্প্রতি) — O(1)

put(4, value) করলে (ক্যাপাসিটি ভরা):
  ১. tail-এর আগের নোড (LRU) মুছে দাও — O(1)
  ২. HashMap থেকে সেই key মুছে দাও — O(1)
  ৩. নতুন নোড head-এর পরে বসাও — O(1)
  ৪. HashMap-এ নতুন entry যোগ করো — O(1)
```

```
ধাপে ধাপে উদাহরণ (capacity=3):

ধাপ ১: put(1, "A")
  HashMap: {1: nodeA}
  List: HEAD ◄──► [1:A] ◄──► TAIL

ধাপ ২: put(2, "B")
  HashMap: {1: nodeA, 2: nodeB}
  List: HEAD ◄──► [2:B] ◄──► [1:A] ◄──► TAIL

ধাপ ৩: put(3, "C")
  HashMap: {1: nodeA, 2: nodeB, 3: nodeC}
  List: HEAD ◄──► [3:C] ◄──► [2:B] ◄──► [1:A] ◄──► TAIL

ধাপ ৪: get(1)  → "A" ফেরত দাও, 1 কে সামনে আনো
  List: HEAD ◄──► [1:A] ◄──► [3:C] ◄──► [2:B] ◄──► TAIL

ধাপ ৫: put(4, "D")  → ক্যাপাসিটি ভরা, LRU (2:B) বাদ দাও
  HashMap: {1: nodeA, 3: nodeC, 4: nodeD}  // 2 মুছে গেছে
  List: HEAD ◄──► [4:D] ◄──► [1:A] ◄──► [3:C] ◄──► TAIL

ধাপ ৬: get(2)  → -1 (পাওয়া যায়নি, বাদ পড়েছে)
```

#### PHP — সম্পূর্ণ LRU Cache

```php
<?php
/**
 * LRU Cache — PHP Implementation
 * Doubly Linked List + HashMap
 * get O(1), put O(1)
 */

class LRUNode {
    public $key;
    public $value;
    public $prev;
    public $next;

    public function __construct($key, $value) {
        $this->key = $key;
        $this->value = $value;
        $this->prev = null;
        $this->next = null;
    }
}

class LRUCache {
    private $capacity;  // সর্বোচ্চ ধারণক্ষমতা
    private $map;       // HashMap — key থেকে node-এ দ্রুত পৌঁছানোর জন্য
    private $head;      // ডামি head — সবচেয়ে সম্প্রতি ব্যবহৃত আইটেমের আগে
    private $tail;      // ডামি tail — সবচেয়ে পুরানো আইটেমের পরে

    public function __construct($capacity) {
        $this->capacity = $capacity;
        $this->map = [];

        // ডামি head ও tail তৈরি (sentinel nodes)
        // এতে edge case handle করতে হয় না
        $this->head = new LRUNode(0, 0);
        $this->tail = new LRUNode(0, 0);
        $this->head->next = $this->tail;
        $this->tail->prev = $this->head;
    }

    // মান পড়ুন — O(1)
    public function get($key) {
        if (!isset($this->map[$key])) {
            return -1; // পাওয়া যায়নি
        }

        $node = $this->map[$key];
        $this->removeNode($node);       // বর্তমান জায়গা থেকে সরাও
        $this->addToFront($node);       // সামনে (সম্প্রতি ব্যবহৃত) রাখো
        return $node->value;
    }

    // মান লিখুন/আপডেট করুন — O(1)
    public function put($key, $value) {
        // key আগে থেকে আছে — আপডেট করো
        if (isset($this->map[$key])) {
            $node = $this->map[$key];
            $node->value = $value;
            $this->removeNode($node);
            $this->addToFront($node);
            return;
        }

        // ক্যাপাসিটি ভরে গেলে LRU বাদ দাও
        if (count($this->map) >= $this->capacity) {
            $lru = $this->tail->prev;       // সবচেয়ে পুরানো নোড
            $this->removeNode($lru);
            unset($this->map[$lru->key]);   // HashMap থেকেও মুছে দাও
        }

        // নতুন নোড তৈরি ও সামনে রাখো
        $newNode = new LRUNode($key, $value);
        $this->addToFront($newNode);
        $this->map[$key] = $newNode;
    }

    // ---- সাহায্যকারী ফাংশন ----

    // নোড সরাও (Doubly LL থেকে) — O(1)
    private function removeNode($node) {
        $node->prev->next = $node->next;
        $node->next->prev = $node->prev;
    }

    // নোড সামনে (head-এর পরে) রাখো — O(1)
    private function addToFront($node) {
        $node->next = $this->head->next;
        $node->prev = $this->head;
        $this->head->next->prev = $node;
        $this->head->next = $node;
    }

    // ক্যাশের বর্তমান অবস্থা দেখাও
    public function display() {
        $current = $this->head->next;
        $parts = [];
        while ($current !== $this->tail) {
            $parts[] = "[{$current->key}:{$current->value}]";
            $current = $current->next;
        }
        echo "LRU Cache: HEAD ◄──► " . implode(" ◄──► ", $parts) . " ◄──► TAIL\n";
    }
}

// ===== ব্যবহার =====
$cache = new LRUCache(3);

$cache->put(1, "A");
$cache->put(2, "B");
$cache->put(3, "C");
$cache->display();
// LRU Cache: HEAD ◄──► [3:C] ◄──► [2:B] ◄──► [1:A] ◄──► TAIL

echo "get(1): " . $cache->get(1) . "\n"; // "A" — 1 সামনে চলে আসবে
$cache->display();
// LRU Cache: HEAD ◄──► [1:A] ◄──► [3:C] ◄──► [2:B] ◄──► TAIL

$cache->put(4, "D"); // ক্যাপাসিটি ভরা — LRU (2:B) বাদ
$cache->display();
// LRU Cache: HEAD ◄──► [4:D] ◄──► [1:A] ◄──► [3:C] ◄──► TAIL

echo "get(2): " . $cache->get(2) . "\n"; // -1 (বাদ পড়েছে)
?>
```

#### JavaScript — সম্পূর্ণ LRU Cache

```javascript
/**
 * LRU Cache — JavaScript Implementation
 * Doubly Linked List + Map
 * get O(1), put O(1)
 */

class LRUNode {
    constructor(key, value) {
        this.key = key;
        this.value = value;
        this.prev = null;
        this.next = null;
    }
}

class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.map = new Map(); // key → node ম্যাপিং

        // ডামি sentinel nodes — edge case এড়াতে
        this.head = new LRUNode(0, 0);
        this.tail = new LRUNode(0, 0);
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }

    // মান পড়ুন — O(1)
    get(key) {
        if (!this.map.has(key)) return -1;

        const node = this.map.get(key);
        this._removeNode(node);    // বর্তমান জায়গা থেকে সরাও
        this._addToFront(node);    // সামনে রাখো
        return node.value;
    }

    // মান লিখুন — O(1)
    put(key, value) {
        if (this.map.has(key)) {
            const node = this.map.get(key);
            node.value = value;
            this._removeNode(node);
            this._addToFront(node);
            return;
        }

        // ক্যাপাসিটি চেক
        if (this.map.size >= this.capacity) {
            const lru = this.tail.prev; // সবচেয়ে পুরানো
            this._removeNode(lru);
            this.map.delete(lru.key);
        }

        const newNode = new LRUNode(key, value);
        this._addToFront(newNode);
        this.map.set(key, newNode);
    }

    // নোড সরাও — O(1)
    _removeNode(node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    // সামনে রাখো — O(1)
    _addToFront(node) {
        node.next = this.head.next;
        node.prev = this.head;
        this.head.next.prev = node;
        this.head.next = node;
    }

    // বর্তমান অবস্থা দেখাও
    display() {
        let current = this.head.next;
        const parts = [];
        while (current !== this.tail) {
            parts.push(`[${current.key}:${current.value}]`);
            current = current.next;
        }
        console.log("LRU: HEAD ◄──► " + parts.join(" ◄──► ") + " ◄──► TAIL");
    }
}

// ===== ব্যবহার =====
const cache = new LRUCache(3);

cache.put(1, "A");
cache.put(2, "B");
cache.put(3, "C");
cache.display();
// LRU: HEAD ◄──► [3:C] ◄──► [2:B] ◄──► [1:A] ◄──► TAIL

console.log(`get(1): ${cache.get(1)}`); // "A"
cache.display();
// LRU: HEAD ◄──► [1:A] ◄──► [3:C] ◄──► [2:B] ◄──► TAIL

cache.put(4, "D"); // LRU (2:B) বাদ
cache.display();
// LRU: HEAD ◄──► [4:D] ◄──► [1:A] ◄──► [3:C] ◄──► TAIL

console.log(`get(2): ${cache.get(2)}`); // -1
```

---

### 🪞 ৫.৬ Palindrome Linked List

```
লিস্ট: 1 -> 2 -> 3 -> 2 -> 1 -> NULL
প্যালিনড্রোম কিনা চেক করো

কৌশল:
  ১. Fast/Slow pointer দিয়ে মাঝ খুঁজুন
  ২. দ্বিতীয় অর্ধেক reverse করুন
  ৩. প্রথম অর্ধেক ও reversed দ্বিতীয় অর্ধেক তুলনা করুন

ধাপ ১: মাঝ খুঁজুন
  1 -> 2 -> 3 -> 2 -> 1 -> NULL
                 ↑S              ↑F(NULL)
  slow = 3 (মাঝের নোড)

ধাপ ২: দ্বিতীয় অর্ধেক (slow.next থেকে) reverse করুন
  আগে:  2 -> 1 -> NULL
  পরে:  1 -> 2 -> NULL

ধাপ ৩: তুলনা
  1 -> 2 -> 3     (প্রথম অর্ধেক)
  1 -> 2          (reversed দ্বিতীয় অর্ধেক)

  1 == 1 ✅
  2 == 2 ✅
  → প্যালিনড্রোম! ✅
```

#### PHP

```php
<?php
// প্যালিনড্রোম চেক — O(n) time, O(1) space
function isPalindrome($head) {
    if ($head === null || $head->next === null) return true;

    // ধাপ ১: মাঝ খুঁজুন (fast/slow)
    $slow = $head;
    $fast = $head;

    while ($fast->next !== null && $fast->next->next !== null) {
        $slow = $slow->next;
        $fast = $fast->next->next;
    }

    // ধাপ ২: দ্বিতীয় অর্ধেক reverse
    $secondHalf = reverseIterative($slow->next);

    // ধাপ ৩: তুলনা
    $firstHalf = $head;
    $secondPtr = $secondHalf;

    while ($secondPtr !== null) {
        if ($firstHalf->data !== $secondPtr->data) {
            return false; // মিলছে না
        }
        $firstHalf = $firstHalf->next;
        $secondPtr = $secondPtr->next;
    }

    // (ঐচ্ছিক) দ্বিতীয় অর্ধেক আবার reverse করে আসল অবস্থায় ফেরাও
    $slow->next = reverseIterative($secondHalf);

    return true; // প্যালিনড্রোম ✅
}
?>
```

#### JavaScript

```javascript
// প্যালিনড্রোম চেক — O(n) time, O(1) space
function isPalindrome(head) {
    if (!head || !head.next) return true;

    // ধাপ ১: মাঝ খুঁজুন
    let slow = head;
    let fast = head;

    while (fast.next && fast.next.next) {
        slow = slow.next;
        fast = fast.next.next;
    }

    // ধাপ ২: দ্বিতীয় অর্ধেক reverse
    let secondHalf = reverseIterative(slow.next);

    // ধাপ ৩: তুলনা
    let p1 = head;
    let p2 = secondHalf;

    while (p2) {
        if (p1.data !== p2.data) return false;
        p1 = p1.next;
        p2 = p2.next;
    }

    // আসল অবস্থায় ফেরাও
    slow.next = reverseIterative(secondHalf);

    return true;
}
```

---

## ৬. অ্যারে vs লিংকড লিস্ট তুলনা

### 📊 বিস্তারিত তুলনা টেবিল

```
┌──────────────────────┬────────────────────┬────────────────────────┐
│       বৈশিষ্ট্য        │      অ্যারে         │     লিংকড লিস্ট         │
├──────────────────────┼────────────────────┼────────────────────────┤
│ মেমোরি বিন্যাস       │ Contiguous         │ Non-contiguous         │
│                      │ (একটানা)            │ (বিচ্ছিন্ন)              │
├──────────────────────┼────────────────────┼────────────────────────┤
│ র‍্যান্ডম অ্যাক্সেস    │ O(1) ✅            │ O(n) ❌                │
│ (index দিয়ে)         │ সরাসরি             │ traverse করতে হয়       │
├──────────────────────┼────────────────────┼────────────────────────┤
│ শুরুতে Insert        │ O(n) ❌            │ O(1) ✅                │
│                      │ সব সরাতে হয়        │ শুধু pointer পরিবর্তন    │
├──────────────────────┼────────────────────┼────────────────────────┤
│ শেষে Insert          │ O(1)* amortized    │ O(1) (tail থাকলে)      │
│                      │                    │ O(n) (tail না থাকলে)    │
├──────────────────────┼────────────────────┼────────────────────────┤
│ মাঝে Insert          │ O(n) ❌            │ O(n) (traverse) +      │
│                      │ shift প্রয়োজন       │ O(1) (insert)          │
├──────────────────────┼────────────────────┼────────────────────────┤
│ শুরু থেকে Delete     │ O(n) ❌            │ O(1) ✅                │
├──────────────────────┼────────────────────┼────────────────────────┤
│ শেষ থেকে Delete      │ O(1)               │ O(n) (singly)          │
│                      │                    │ O(1) (doubly)          │
├──────────────────────┼────────────────────┼────────────────────────┤
│ Search               │ O(n) linear        │ O(n) linear            │
│                      │ O(log n) sorted    │ binary search নেই ❌    │
├──────────────────────┼────────────────────┼────────────────────────┤
│ মেমোরি ব্যবহার       │ কম (শুধু data)      │ বেশি (data + pointer)  │
│                      │                    │ Doubly: data+2 pointer │
├──────────────────────┼────────────────────┼────────────────────────┤
│ Cache Friendliness   │ ✅ খুব ভালো         │ ❌ খারাপ               │
│                      │ Locality of        │ নোড মেমোরিতে          │
│                      │ reference          │ ছড়িয়ে থাকে            │
├──────────────────────┼────────────────────┼────────────────────────┤
│ সাইজ পরিবর্তন        │ Fixed / costly     │ Dynamic ✅             │
│                      │ resize             │ সহজেই বাড়ে/কমে        │
├──────────────────────┼────────────────────┼────────────────────────┤
│ মেমোরি অপচয়         │ পূর্ব-বরাদ্দকৃত      │ কোনো অপচয় নেই         │
│                      │ অতিরিক্ত স্থান       │ প্রয়োজনমতো বরাদ্দ      │
└──────────────────────┴────────────────────┴────────────────────────┘
```

### 🧭 কখন কোনটা ব্যবহার করবেন — সিদ্ধান্ত গাইড

```
📌 অ্যারে ব্যবহার করুন যখন:
  ✅ ইনডেক্স দিয়ে দ্রুত অ্যাক্সেস দরকার
  ✅ ডেটার সাইজ আগে থেকে জানা
  ✅ ক্যাশ পারফরম্যান্স গুরুত্বপূর্ণ
  ✅ Binary search প্রয়োজন
  ✅ কম মেমোরি ব্যবহার করতে চান

📌 লিংকড লিস্ট ব্যবহার করুন যখন:
  ✅ ঘন ঘন শুরুতে/মাঝে insert/delete
  ✅ ডেটার সাইজ অজানা বা পরিবর্তনশীল
  ✅ ধারাবাহিক অ্যাক্সেস (sequential) যথেষ্ট
  ✅ Stack/Queue implement করতে
  ✅ LRU Cache-এর মতো জটিল ডেটা স্ট্রাকচার

📌 সহজ নিয়ম:
  "বেশিরভাগ ক্ষেত্রে অ্যারে ব্যবহার করুন।
   লিংকড লিস্ট তখনই ব্যবহার করুন যখন ঘন ঘন
   insert/delete প্রয়োজন এবং random access না লাগে।"
```

---

## 📋 কমপ্লেক্সিটি চিটশিট

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    লিংকড লিস্ট কমপ্লেক্সিটি চিটশিট                    ║
╠═══════════════════════════════╦═══════════╦═══════════╦═════════════╣
║          অপারেশন              ║  Singly   ║  Doubly   ║  Circular   ║
╠═══════════════════════════════╬═══════════╬═══════════╬═════════════╣
║ Head-এ Insert                ║   O(1)    ║   O(1)    ║    O(1)     ║
║ Tail-এ Insert                ║   O(n)*   ║   O(1)    ║    O(1)     ║
║ মাঝে Insert                  ║   O(n)    ║   O(n)    ║    O(n)     ║
║ Head Delete                  ║   O(1)    ║   O(1)    ║    O(1)     ║
║ Tail Delete                  ║   O(n)    ║   O(1)    ║    O(1)     ║
║ নোড Delete (ref জানা)         ║   O(n)    ║   O(1)    ║    O(1)     ║
║ Search                       ║   O(n)    ║   O(n)    ║    O(n)     ║
║ Access (index)               ║   O(n)    ║   O(n)    ║    O(n)     ║
╠═══════════════════════════════╬═══════════╬═══════════╬═════════════╣
║ Space (per node)             ║ data+1ptr ║ data+2ptr ║  data+1/2   ║
╚═══════════════════════════════╩═══════════╩═══════════╩═════════════╝
* tail পয়েন্টার রাখলে O(1)

╔═══════════════════════════════════════════════════════════════╗
║                  ক্লাসিক সমস্যা চিটশিট                        ║
╠═══════════════════════════╦════════════╦══════════════════════╣
║         সমস্যা             ║    সময়     ║       টেকনিক          ║
╠═══════════════════════════╬════════════╬══════════════════════╣
║ Reverse List              ║   O(n)     ║ Iterative / Recursive║
║ Detect Cycle              ║   O(n)     ║ Floyd's (fast/slow)  ║
║ Find Cycle Start          ║   O(n)     ║ Floyd's + reset      ║
║ Find Middle               ║   O(n)     ║ Fast/Slow pointer    ║
║ Merge Two Sorted          ║   O(n+m)   ║ Dummy node           ║
║ Remove Nth from End       ║   O(n)     ║ Two pointer gap      ║
║ Check Palindrome          ║   O(n)     ║ Fast/Slow + reverse  ║
║ LRU Cache (get/put)       ║   O(1)     ║ DLL + HashMap        ║
╚═══════════════════════════╩════════════╩══════════════════════╝
```

---

## 🎯 ইন্টারভিউ টিপস

```
💡 ১. NULL চেক — সবসময় head === null এবং node.next === null চেক করুন।
      NullPointerException সবচেয়ে সাধারণ ভুল!

💡 ২. Edge Cases সবসময় বিবেচনা করুন:
      - খালি লিস্ট (head = null)
      - একটি মাত্র নোড
      - দুটি নোড
      - Head বা Tail-এ অপারেশন

💡 ৩. Dummy Node ব্যবহার করুন — merge, delete, partition-এর মতো
      সমস্যায় edge case handling সহজ হয়।

💡 ৪. Fast/Slow Pointer মনে রাখুন:
      - মাঝ খুঁজুন → slow=1x, fast=2x
      - সাইকেল খুঁজুন → slow=1x, fast=2x
      - শেষ থেকে nth → gap=n রেখে দুজনে হাঁটো

💡 ৫. Drawing করুন! কাগজে/হোয়াইটবোর্ডে ছবি আঁকুন।
      Pointer manipulation ভিজ্যুয়ালাইজ না করলে ভুল হওয়ার সম্ভাবনা বেশি।

💡 ৬. Space Complexity:
      - ইন্টারভিউয়ার O(1) space চাইবেন → iterative ব্যবহার করুন
      - Recursive = O(n) space (call stack)

💡 ৭. Singly vs Doubly সিদ্ধান্ত:
      - Singly যথেষ্ট হলে Doubly ব্যবহার করবেন না (অতিরিক্ত মেমোরি)
      - Doubly দরকার: LRU Cache, Browser History, O(1) delete

💡 ৮. সাধারণ ভুল এড়িয়ে চলুন:
      ❌ next pointer আপডেটের আগে সংরক্ষণ না করা
      ❌ লুপ কন্ডিশনে current vs current.next বিভ্রান্তি
      ❌ Circular list-এ অসীম লুপ
      ❌ head আপডেট করতে ভুলে যাওয়া
```

---

## 🔗 পরবর্তী পড়ুন

লিংকড লিস্ট শিখলে এখন **স্ট্যাক ও কিউ** শিখুন — এগুলো লিংকড লিস্ট দিয়ে implement করা যায়:

➡️ [স্ট্যাক ও কিউ (Stack & Queue)](./stack-queue.md)

---

> _"লিংকড লিস্ট বুঝলে pointer বোঝা হয়, pointer বুঝলে programming-এর অর্ধেক বোঝা হয়।"_ 🎓
