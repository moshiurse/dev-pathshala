# 🏔️ হিপ ও প্রায়োরিটি কিউ (Heap & Priority Queue)

> _"হিপ হলো এমন একটি গাছ যেখানে সবসময় সবচেয়ে গুরুত্বপূর্ণ জিনিসটা ওপরে থাকে।"_ 🔝

---

## 📖 সংজ্ঞা

**হিপ (Heap)** হলো একটি **কমপ্লিট বাইনারি ট্রি** ভিত্তিক ডেটা স্ট্রাকচার যেখানে প্রতিটি নোড তার চাইল্ডদের থেকে একটি নির্দিষ্ট ক্রম (বড় বা ছোট) বজায় রাখে। হিপ মূলত দুই ধরনের:

- **Min-Heap**: প্যারেন্ট সবসময় চাইল্ডদের থেকে **ছোট বা সমান**
- **Max-Heap**: প্যারেন্ট সবসময় চাইল্ডদের থেকে **বড় বা সমান**

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | হিপ-এর সাথে তুলনা |
|---|---|
| হাসপাতালের ইমার্জেন্সি | সবচেয়ে জরুরি রোগী আগে দেখা হয় → Min-Heap (priority) |
| VIP লাউঞ্জ | সবচেয়ে গুরুত্বপূর্ণ ব্যক্তি সবার আগে প্রবেশ → Max-Heap |
| প্রিন্টার কিউ | জরুরি ডকুমেন্ট আগে প্রিন্ট হয় → Priority Queue |
| টাস্ক শিডিউলার (OS) | সর্বোচ্চ প্রায়োরিটির প্রসেস আগে চলে |

---

## ১. হিপ-এর মৌলিক ধারণা

### 🌲 কমপ্লিট বাইনারি ট্রি

```
Max-Heap উদাহরণ:

            [90]              ← রুট: সর্বোচ্চ মান
           /    \
        [70]    [80]          ← দ্বিতীয় স্তর
       /   \    /
     [40] [50] [60]           ← তৃতীয় স্তর

অ্যারে রিপ্রেজেন্টেশন: [90, 70, 80, 40, 50, 60]
ইনডেক্স:                 0   1   2   3   4   5

প্যারেন্ট(i) = floor((i-1)/2)
বাম চাইল্ড(i) = 2*i + 1
ডান চাইল্ড(i) = 2*i + 2
```

### 📏 টাইম কমপ্লেক্সিটি

| অপারেশন | কমপ্লেক্সিটি |
|---|---|
| Insert | O(log n) |
| Extract Min/Max | O(log n) |
| Peek (Top) | O(1) |
| Heapify | O(n) |
| Heap Sort | O(n log n) |

---

## ২. Heapify — হিপ তৈরির প্রক্রিয়া

**Heapify** হলো একটি অ্যারেকে হিপ প্রোপার্টি অনুযায়ী পুনর্বিন্যাস করার প্রক্রিয়া।

```
Heapify-Down উদাহরণ (Max-Heap):

ধরি অ্যারে: [10, 70, 80, 40, 50, 60]

ধাপ ১: ইনডেক্স 2 (80) — চাইল্ড নেই যে বড়, ঠিক আছে
ধাপ ২: ইনডেক্স 1 (70) — ঠিক আছে
ধাপ ৩: ইনডেক্স 0 (10) — 80 বড়, swap!

        [10]           [80]           [80]
       /    \    →    /    \    →    /    \
     [70]  [80]    [70]  [10]    [70]  [60]
    /  \   /      /  \   /      /  \   /
  [40][50][60]  [40][50][60]  [40][50][10]
```

---

## ৩. PHP-তে Heap ইমপ্লিমেন্টেশন

```php
<?php

class MinHeap {
    private array $heap = [];

    public function insert(int $value): void {
        $this->heap[] = $value;
        $this->heapifyUp(count($this->heap) - 1);
    }

    public function extractMin(): ?int {
        if (empty($this->heap)) return null;

        $min = $this->heap[0];
        $last = array_pop($this->heap);

        if (!empty($this->heap)) {
            $this->heap[0] = $last;
            $this->heapifyDown(0);
        }

        return $min;
    }

    public function peek(): ?int {
        return $this->heap[0] ?? null;
    }

    private function heapifyUp(int $index): void {
        while ($index > 0) {
            $parent = intdiv($index - 1, 2);
            if ($this->heap[$parent] <= $this->heap[$index]) break;

            [$this->heap[$parent], $this->heap[$index]] = [$this->heap[$index], $this->heap[$parent]];
            $index = $parent;
        }
    }

    private function heapifyDown(int $index): void {
        $size = count($this->heap);

        while (true) {
            $smallest = $index;
            $left = 2 * $index + 1;
            $right = 2 * $index + 2;

            if ($left < $size && $this->heap[$left] < $this->heap[$smallest]) {
                $smallest = $left;
            }
            if ($right < $size && $this->heap[$right] < $this->heap[$smallest]) {
                $smallest = $right;
            }

            if ($smallest === $index) break;

            [$this->heap[$index], $this->heap[$smallest]] = [$this->heap[$smallest], $this->heap[$index]];
            $index = $smallest;
        }
    }

    public function size(): int {
        return count($this->heap);
    }
}

// ব্যবহার
$heap = new MinHeap();
$heap->insert(30);
$heap->insert(10);
$heap->insert(20);
$heap->insert(5);

echo $heap->extractMin(); // 5
echo $heap->extractMin(); // 10

// PHP-তে বিল্ট-ইন SplMinHeap ও SplMaxHeap আছে
$splHeap = new SplMinHeap();
$splHeap->insert(30);
$splHeap->insert(10);
$splHeap->insert(5);
echo $splHeap->extract(); // 5
```

---

## ৪. JavaScript-এ Heap ইমপ্লিমেন্টেশন

```javascript
class MaxHeap {
    constructor() {
        this.heap = [];
    }

    insert(value) {
        this.heap.push(value);
        this._heapifyUp(this.heap.length - 1);
    }

    extractMax() {
        if (this.heap.length === 0) return null;

        const max = this.heap[0];
        const last = this.heap.pop();

        if (this.heap.length > 0) {
            this.heap[0] = last;
            this._heapifyDown(0);
        }

        return max;
    }

    peek() {
        return this.heap[0] ?? null;
    }

    _heapifyUp(index) {
        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (this.heap[parent] >= this.heap[index]) break;

            [this.heap[parent], this.heap[index]] = [this.heap[index], this.heap[parent]];
            index = parent;
        }
    }

    _heapifyDown(index) {
        const size = this.heap.length;

        while (true) {
            let largest = index;
            const left = 2 * index + 1;
            const right = 2 * index + 2;

            if (left < size && this.heap[left] > this.heap[largest]) largest = left;
            if (right < size && this.heap[right] > this.heap[largest]) largest = right;

            if (largest === index) break;

            [this.heap[index], this.heap[largest]] = [this.heap[largest], this.heap[index]];
            index = largest;
        }
    }

    get size() {
        return this.heap.length;
    }
}

// Heap Sort ইমপ্লিমেন্টেশন
function heapSort(arr) {
    const heap = new MaxHeap();
    arr.forEach(val => heap.insert(val));

    const sorted = [];
    while (heap.size > 0) {
        sorted.unshift(heap.extractMax());
    }
    return sorted;
}

// ব্যবহার
const heap = new MaxHeap();
heap.insert(10);
heap.insert(40);
heap.insert(20);
heap.insert(50);

console.log(heap.extractMax()); // 50
console.log(heap.extractMax()); // 40

console.log(heapSort([38, 27, 43, 3, 9, 82, 10]));
// [3, 9, 10, 27, 38, 43, 82]
```

---

## ৫. Priority Queue — প্রায়োরিটি কিউ

**Priority Queue** হলো এমন একটি কিউ যেখানে প্রতিটি এলিমেন্টের একটি **প্রায়োরিটি** থাকে এবং সর্বোচ্চ (বা সর্বনিম্ন) প্রায়োরিটির এলিমেন্ট সবার আগে বের হয়।

```php
<?php
// PHP-তে SplPriorityQueue ব্যবহার
$pq = new SplPriorityQueue();
$pq->insert('সাধারণ রোগী', 1);
$pq->insert('জরুরি রোগী', 3);
$pq->insert('মাঝারি জরুরি', 2);

while (!$pq->isEmpty()) {
    echo $pq->extract() . "\n";
}
// জরুরি রোগী → মাঝারি জরুরি → সাধারণ রোগী
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন হিপ |
|---|---|
| সর্বোচ্চ/সর্বনিম্ন দ্রুত বের করতে | O(1) peek |
| ক্রমাগত সর্টেড ডেটা দরকার | Priority Queue |
| মিডিয়ান ফাইন্ডিং (স্ট্রিমে) | দুটি হিপ একসাথে |
| Dijkstra / Prim's অ্যালগরিদম | Min-Heap দরকার |
| টাস্ক শিডিউলিং | Priority Queue |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না |
|---|---|
| র‍্যান্ডম অ্যাক্সেস দরকার | হিপে O(n) লাগে |
| সম্পূর্ণ সর্টেড ডেটা দরকার | সরাসরি সর্ট ভালো |
| ছোট ডেটাসেট | সিম্পল অ্যারেই যথেষ্ট |
| নির্দিষ্ট এলিমেন্ট খোঁজা | Hash Table বা BST ব্যবহার করুন |

---

## 🧠 মনে রাখার টিপস

```
হিপ = কমপ্লিট বাইনারি ট্রি + অর্ডার প্রোপার্টি
Min-Heap → ছোটটা ওপরে → "ছোটরা আগে"
Max-Heap → বড়টা ওপরে → "বড়রা আগে"
Priority Queue = হিপ-ভিত্তিক ADT
Heap Sort = Max-Heap থেকে একটা একটা করে বের করা
```
