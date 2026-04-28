# 🔤 ট্রাই (Trie — Prefix Tree)

> _"ট্রাই হলো এমন একটি গাছ যেখানে শব্দের প্রতিটি অক্ষর একটি করে ধাপ — শিকড় থেকে পাতা পর্যন্ত পড়লেই শব্দ পাওয়া যায়।"_ 🌳

---

## 📖 সংজ্ঞা

**ট্রাই (Trie)** হলো একটি **ট্রি-ভিত্তিক ডেটা স্ট্রাকচার** যা মূলত **স্ট্রিং সংরক্ষণ ও অনুসন্ধানের** জন্য ব্যবহৃত হয়। প্রতিটি নোড একটি **অক্ষর (character)** ধারণ করে এবং রুট থেকে যেকোনো নোড পর্যন্ত পথ একটি **prefix** তৈরি করে।

"Trie" শব্দটি এসেছে "re**trie**val" থেকে।

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | ট্রাই-এর সাথে তুলনা |
|---|---|
| মোবাইলের Autocomplete | টাইপ করতে করতে সাজেশন আসে |
| ডিকশনারি | শব্দ প্রেফিক্স ধরে দ্রুত খোঁজা |
| ফোনবুকের নাম খোঁজা | প্রথম কয়েকটি অক্ষর লিখলেই ফিল্টার হয় |
| IP রাউটিং টেবিল | Longest Prefix Matching |
| স্পেল চেকার | শব্দ সঠিক কিনা যাচাই |

---

## ১. ট্রাই-এর গঠন

```
শব্দ: "cat", "car", "card", "dog", "do"

          [root]
         /      \
       [c]      [d]
        |         |
       [a]      [o]✓
       / \        |
     [t]✓ [r]✓  [g]✓
            |
           [d]✓

✓ = শব্দের শেষ (isEndOfWord = true)

"cat" → root → c → a → t ✓
"car" → root → c → a → r ✓
"card"→ root → c → a → r → d ✓
"do"  → root → d → o ✓
"dog" → root → d → o → g ✓
```

### 📏 টাইম কমপ্লেক্সিটি

| অপারেশন | কমপ্লেক্সিটি |
|---|---|
| Insert | O(m) — m = শব্দের দৈর্ঘ্য |
| Search | O(m) |
| Delete | O(m) |
| Prefix Search | O(p + k) — p = prefix দৈর্ঘ্য, k = ফলাফল সংখ্যা |

### 🆚 Hash Table vs Trie

| বৈশিষ্ট্য | Hash Table | Trie |
|---|---|---|
| Exact Search | O(1) avg | O(m) |
| Prefix Search | ❌ সম্ভব নয় | ✅ দ্রুত |
| Sorted Order | ❌ | ✅ |
| মেমোরি | কম | বেশি (পয়েন্টার) |

---

## ২. PHP-তে Trie ইমপ্লিমেন্টেশন

```php
<?php

class TrieNode {
    public array $children = [];
    public bool $isEndOfWord = false;
}

class Trie {
    private TrieNode $root;

    public function __construct() {
        $this->root = new TrieNode();
    }

    // শব্দ ইনসার্ট
    public function insert(string $word): void {
        $node = $this->root;
        for ($i = 0; $i < strlen($word); $i++) {
            $char = $word[$i];
            if (!isset($node->children[$char])) {
                $node->children[$char] = new TrieNode();
            }
            $node = $node->children[$char];
        }
        $node->isEndOfWord = true;
    }

    // শব্দ খোঁজা
    public function search(string $word): bool {
        $node = $this->findNode($word);
        return $node !== null && $node->isEndOfWord;
    }

    // প্রেফিক্স আছে কিনা
    public function startsWith(string $prefix): bool {
        return $this->findNode($prefix) !== null;
    }

    // শব্দ ডিলিট
    public function delete(string $word): void {
        $this->deleteHelper($this->root, $word, 0);
    }

    private function deleteHelper(TrieNode $node, string $word, int $depth): bool {
        if ($depth === strlen($word)) {
            if (!$node->isEndOfWord) return false;
            $node->isEndOfWord = false;
            return empty($node->children);
        }

        $char = $word[$depth];
        if (!isset($node->children[$char])) return false;

        $shouldDelete = $this->deleteHelper($node->children[$char], $word, $depth + 1);

        if ($shouldDelete) {
            unset($node->children[$char]);
            return empty($node->children) && !$node->isEndOfWord;
        }

        return false;
    }

    // Autocomplete — নির্দিষ্ট prefix-এর সব শব্দ
    public function autocomplete(string $prefix): array {
        $node = $this->findNode($prefix);
        if ($node === null) return [];

        $results = [];
        $this->collectWords($node, $prefix, $results);
        return $results;
    }

    private function findNode(string $prefix): ?TrieNode {
        $node = $this->root;
        for ($i = 0; $i < strlen($prefix); $i++) {
            $char = $prefix[$i];
            if (!isset($node->children[$char])) return null;
            $node = $node->children[$char];
        }
        return $node;
    }

    private function collectWords(TrieNode $node, string $prefix, array &$results): void {
        if ($node->isEndOfWord) {
            $results[] = $prefix;
        }
        foreach ($node->children as $char => $child) {
            $this->collectWords($child, $prefix . $char, $results);
        }
    }
}

// ব্যবহার
$trie = new Trie();
$trie->insert("bangladesh");
$trie->insert("bangla");
$trie->insert("bank");
$trie->insert("banner");

echo $trie->search("bangla") ? "পাওয়া গেছে" : "নেই"; // পাওয়া গেছে
echo $trie->startsWith("ban") ? "prefix আছে" : "নেই"; // prefix আছে

print_r($trie->autocomplete("ban"));
// ["bangladesh", "bangla", "bank", "banner"]

$trie->delete("bank");
echo $trie->search("bank") ? "আছে" : "নেই"; // নেই
```

---

## ৩. JavaScript-এ Trie ইমপ্লিমেন্টেশন

```javascript
class TrieNode {
    constructor() {
        this.children = {};
        this.isEndOfWord = false;
    }
}

class Trie {
    constructor() {
        this.root = new TrieNode();
    }

    insert(word) {
        let node = this.root;
        for (const char of word) {
            if (!node.children[char]) {
                node.children[char] = new TrieNode();
            }
            node = node.children[char];
        }
        node.isEndOfWord = true;
    }

    search(word) {
        const node = this._findNode(word);
        return node !== null && node.isEndOfWord;
    }

    startsWith(prefix) {
        return this._findNode(prefix) !== null;
    }

    delete(word) {
        this._deleteHelper(this.root, word, 0);
    }

    _deleteHelper(node, word, depth) {
        if (depth === word.length) {
            if (!node.isEndOfWord) return false;
            node.isEndOfWord = false;
            return Object.keys(node.children).length === 0;
        }

        const char = word[depth];
        if (!node.children[char]) return false;

        const shouldDelete = this._deleteHelper(node.children[char], word, depth + 1);

        if (shouldDelete) {
            delete node.children[char];
            return Object.keys(node.children).length === 0 && !node.isEndOfWord;
        }

        return false;
    }

    // Autocomplete ফিচার
    autocomplete(prefix) {
        const node = this._findNode(prefix);
        if (!node) return [];

        const results = [];
        this._collectWords(node, prefix, results);
        return results;
    }

    _findNode(prefix) {
        let node = this.root;
        for (const char of prefix) {
            if (!node.children[char]) return null;
            node = node.children[char];
        }
        return node;
    }

    _collectWords(node, prefix, results) {
        if (node.isEndOfWord) results.push(prefix);
        for (const [char, child] of Object.entries(node.children)) {
            this._collectWords(child, prefix + char, results);
        }
    }
}

// ব্যবহার
const trie = new Trie();
["dhaka", "dinajpur", "dhanmondi", "delhi", "dubai"].forEach(w => trie.insert(w));

console.log(trie.search("dhaka"));       // true
console.log(trie.startsWith("dh"));      // true
console.log(trie.autocomplete("dh"));    // ["dhaka", "dhanmondi"]
console.log(trie.autocomplete("d"));     // ["dhaka", "dinajpur", "dhanmondi", "delhi", "dubai"]

trie.delete("dhaka");
console.log(trie.search("dhaka"));       // false
console.log(trie.startsWith("dh"));      // true (dhanmondi এখনো আছে)
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন ট্রাই |
|---|---|
| Autocomplete / Typeahead | প্রেফিক্স সার্চ অত্যন্ত দ্রুত |
| স্পেল চেকার | শব্দ আছে কিনা দ্রুত যাচাই |
| IP রাউটিং | Longest Prefix Matching |
| ওয়ার্ড গেম (Scrabble, Boggle) | শব্দ ভ্যালিডেশন |
| সার্চ ইঞ্জিন সাজেশন | কীওয়ার্ড প্রেফিক্স ম্যাচ |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না |
|---|---|
| Exact match-ই যথেষ্ট | Hash Table দ্রুততর |
| মেমোরি সীমিত | ট্রাই অনেক পয়েন্টার রাখে |
| সংখ্যাভিত্তিক ডেটা | অন্য ডেটা স্ট্রাকচার ভালো |
| খুব কম ডেটা | সিম্পল অ্যারে/অবজেক্ট যথেষ্ট |

---

## 🧠 মনে রাখার টিপস

```
Trie = Prefix Tree = রিট্রিভাল ট্রি
প্রতিটি এজ = একটি অক্ষর
রুট = খালি
পাতা (বা isEndOfWord) = শব্দ শেষ
Autocomplete = prefix node থেকে DFS
```
