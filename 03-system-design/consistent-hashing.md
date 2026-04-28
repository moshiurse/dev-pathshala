# 🔄 কনসিস্টেন্ট হ্যাশিং (Consistent Hashing)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**কনসিস্টেন্ট হ্যাশিং** হলো একটি বিশেষ হ্যাশিং কৌশল যেখানে হ্যাশ টেবিলের আকার পরিবর্তন হলেও গড়ে শুধুমাত্র `K/N` কী রি-ম্যাপ করতে হয় (K = মোট কী, N = মোট নোড)। ডিস্ট্রিবিউটেড সিস্টেমে ডেটা পার্টিশনিং ও লোড ব্যালেন্সিংয়ের জন্য এটি অত্যন্ত গুরুত্বপূর্ণ।

### কেন সাধারণ হ্যাশিং যথেষ্ট নয়?

```
সাধারণ হ্যাশিং (Modular Hashing):
  server = hash(key) % N

সমস্যা — ধরুন bKash-এর ক্যাশ সার্ভার ৪টি থেকে ৫টি হলো:
  hash("user_123") % 4 = 3  →  Server 3
  hash("user_123") % 5 = 1  →  Server 1  😱

  ফলাফল: প্রায় সব কী রি-ম্যাপ হয়! → ক্যাশ মিস ঝড় (Cache Stampede) 💀
```

---

## 🎯 বাস্তব উদাহরণ — চট্টগ্রাম বন্দরের গুদাম

```
কল্পনা করুন, চট্টগ্রাম বন্দরে ৪টি গুদাম আছে।
প্রতিটি পণ্যের নাম হ্যাশ করে গুদাম ঠিক করা হয়।

সাধারণ পদ্ধতি:
  ৫ম গুদাম যোগ হলে → প্রায় সব পণ্য সরাতে হবে 😰

কনসিস্টেন্ট হ্যাশিং:
  ৫ম গুদাম যোগ হলে → শুধু প্রতিবেশী গুদামের কিছু পণ্য সরালেই হবে ✅
  বাকি গুদামের পণ্য যেখানে আছে সেখানেই থাকবে
```

---

## 🔵 হ্যাশ রিং (Hash Ring) ধারণা

```
              কনসিস্টেন্ট হ্যাশিং রিং
              =========================

                    0 / 2^32
                     ╱╲
                   ╱    ╲
                 ╱        ╲
          Node A ●          ● Node B
               ╱              ╲
              │                 │
              │    Hash Ring    │
              │                 │
               ╲              ╱
          Node D ●          ● Node C
                 ╲        ╱
                   ╲    ╱
                     ╲╱

  Key "user_1" → হ্যাশ করে রিংয়ে বসান → ঘড়ির কাঁটার দিকে
                  প্রথম যে নোড পাবে, সেটাই দায়িত্বপ্রাপ্ত
```

### কী ম্যাপিং প্রক্রিয়া

```
  hash("user_1")   = 15000  → Node B (অবস্থান: 20000) এ যাবে
  hash("order_42") = 55000  → Node C (অবস্থান: 60000) এ যাবে
  hash("txn_99")   = 85000  → Node D (অবস্থান: 90000) এ যাবে
```

---

## 🌀 ভার্চুয়াল নোড (Virtual Nodes)

ভার্চুয়াল নোড ছাড়া নোডগুলো রিংয়ে অসমানভাবে বসতে পারে, ফলে লোড ভারসাম্যহীন হয়।

```
সমস্যা: ২টি নোড কাছাকাছি বসলে —
  Node A: ৮০% ডেটা 😫
  Node B: ২০% ডেটা 😴

সমাধান — ভার্চুয়াল নোড:
  প্রতিটি ফিজিক্যাল নোডের ১০০-২০০টি ভার্চুয়াল কপি রিংয়ে ছড়িয়ে দাও

  Node A → vA1, vA2, vA3, ..., vA150
  Node B → vB1, vB2, vB3, ..., vB150

  ফলাফল: লোড প্রায় সমানভাবে ভাগ হয় ✅
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

class ConsistentHash
{
    private array $ring = [];
    private array $nodes = [];
    private int $virtualNodes;

    public function __construct(int $virtualNodes = 150)
    {
        $this->virtualNodes = $virtualNodes;
    }

    public function addNode(string $node): void
    {
        $this->nodes[] = $node;
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualKey = $node . '#' . $i;
            $hash = crc32($virtualKey);
            $this->ring[$hash] = $node;
        }
        ksort($this->ring);
    }

    public function removeNode(string $node): void
    {
        $this->nodes = array_filter($this->nodes, fn($n) => $n !== $node);
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $virtualKey = $node . '#' . $i;
            $hash = crc32($virtualKey);
            unset($this->ring[$hash]);
        }
    }

    public function getNode(string $key): ?string
    {
        if (empty($this->ring)) {
            return null;
        }

        $hash = crc32($key);
        // ঘড়ির কাঁটার দিকে প্রথম নোড খুঁজি
        foreach ($this->ring as $nodeHash => $node) {
            if ($nodeHash >= $hash) {
                return $node;
            }
        }
        // রিংয়ের শেষে পৌঁছালে প্রথম নোডে ফিরে যাই
        return reset($this->ring);
    }
}

// ব্যবহার — ক্যাশ সার্ভার নির্বাচন
$ch = new ConsistentHash(virtualNodes: 150);
$ch->addNode('cache-server-1');
$ch->addNode('cache-server-2');
$ch->addNode('cache-server-3');

echo $ch->getNode('user:1001');  // cache-server-2
echo $ch->getNode('order:5042'); // cache-server-1

// নতুন সার্ভার যোগ — শুধু কিছু কী রি-ম্যাপ হবে
$ch->addNode('cache-server-4');
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
const crypto = require('crypto');

class ConsistentHash {
  constructor(virtualNodes = 150) {
    this.virtualNodes = virtualNodes;
    this.ring = new Map();
    this.sortedKeys = [];
  }

  _hash(key) {
    return parseInt(
      crypto.createHash('md5').update(key).digest('hex').slice(0, 8),
      16
    );
  }

  addNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = `${node}#${i}`;
      const hash = this._hash(virtualKey);
      this.ring.set(hash, node);
    }
    this.sortedKeys = [...this.ring.keys()].sort((a, b) => a - b);
  }

  removeNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = `${node}#${i}`;
      const hash = this._hash(virtualKey);
      this.ring.delete(hash);
    }
    this.sortedKeys = [...this.ring.keys()].sort((a, b) => a - b);
  }

  getNode(key) {
    if (this.sortedKeys.length === 0) return null;

    const hash = this._hash(key);
    // বাইনারি সার্চ দিয়ে ঘড়ির কাঁটার দিকে নিকটতম নোড খুঁজি
    let low = 0, high = this.sortedKeys.length - 1;
    while (low < high) {
      const mid = (low + high) >> 1;
      if (this.sortedKeys[mid] < hash) low = mid + 1;
      else high = mid;
    }

    const idx = this.sortedKeys[low] >= hash ? low : 0;
    return this.ring.get(this.sortedKeys[idx]);
  }
}

// ব্যবহার
const ch = new ConsistentHash(150);
ch.addNode('redis-1.bkash.internal');
ch.addNode('redis-2.bkash.internal');
ch.addNode('redis-3.bkash.internal');

console.log(ch.getNode('user:1001'));   // redis-2.bkash.internal
console.log(ch.getNode('wallet:5042')); // redis-1.bkash.internal

// নতুন নোড যোগ করলে মিনিমাল ডিসরাপশন
ch.addNode('redis-4.bkash.internal');
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন |
|---|---|
| ডিস্ট্রিবিউটেড ক্যাশ (Redis Cluster, Memcached) | নোড যোগ/বাদে ক্যাশ ইনভ্যালিডেশন কমানো |
| ডেটাবেস শার্ডিং | ডেটা সমানভাবে পার্টিশন করা |
| CDN নোড সিলেকশন | ইউজারকে নিকটতম সার্ভারে রাউট করা |
| ডিস্ট্রিবিউটেড ফাইল সিস্টেম | ফাইল চাঙ্ক বিতরণ (HDFS, Cassandra) |
| লোড ব্যালেন্সিং | স্টিকি সেশনের বিকল্প হিসেবে |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| ছোট/ফিক্সড সার্ভার সেট | সাধারণ মডুলার হ্যাশিংই যথেষ্ট |
| ডেটা লোকালিটি জরুরি | কনসিস্টেন্ট হ্যাশিং ডেটা ছড়িয়ে দেয় |
| অর্ডারড ডেটা অ্যাক্সেস | রেঞ্জ কোয়েরি সাপোর্ট করে না |
| সিঙ্গেল নোড সিস্টেম | অপ্রয়োজনীয় জটিলতা |

---

## 🏭 বাস্তব ব্যবহারের ক্ষেত্র

```
Amazon DynamoDB     → পার্টিশন কী ম্যাপিং
Apache Cassandra    → ডেটা পার্টিশনিং
Discord             → গিল্ড/সার্ভার শার্ডিং
Akamai CDN          → কন্টেন্ট রাউটিং
```

---

## 🔑 মনে রাখার পয়েন্ট

1. **সাধারণ হ্যাশিং** — নোড পরিবর্তনে প্রায় সব কী রি-ম্যাপ হয়
2. **কনসিস্টেন্ট হ্যাশিং** — শুধু `K/N` কী রি-ম্যাপ হয়
3. **ভার্চুয়াল নোড** — লোড ব্যালেন্স নিশ্চিত করে (১৫০-২০০ ভার্চুয়াল নোড আদর্শ)
4. **রিং স্ট্রাকচার** — ঘড়ির কাঁটার দিকে ঘুরে নিকটতম নোড খুঁজে
5. ইন্টারভিউতে — DynamoDB, Cassandra, Redis Cluster এর রেফারেন্স দিন
