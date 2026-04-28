# 🔍 সার্চ অটোকমপ্লিট (Search Autocomplete / Typeahead)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**সার্চ অটোকমপ্লিট** (Typeahead) হলো এমন একটি সিস্টেম যা ইউজার টাইপ করার সাথে সাথে সম্ভাব্য সার্চ সাজেশন দেখায়। এটি ইউজার এক্সপেরিয়েন্স উন্নত করে, টাইপিং কমায় এবং জনপ্রিয় সার্চ টার্মগুলো দ্রুত খুঁজে পেতে সাহায্য করে।

### কেন অটোকমপ্লিট দরকার?

```
Daraz.com.bd-তে ইউজার টাইপ করছে: "মোবা..."

অটোকমপ্লিট সাজেশন:
  ┌──────────────────────────────────┐
  │  🔍 মোবা                         │
  ├──────────────────────────────────┤
  │  📱 মোবাইল ফোন                   │
  │  📱 মোবাইল চার্জার                │
  │  📱 মোবাইল কভার                  │
  │  📱 মোবাইল স্ক্রিন প্রটেক্টর       │
  │  📱 মোবাইল ব্যাটারি               │
  └──────────────────────────────────┘

  → ইউজার "মোবাইল ফোন" সিলেক্ট করলো — ৭টি অক্ষর বাঁচলো! ✅
```

---

## 🎯 বাস্তব উদাহরণ — ডাকঘরের ঠিকানা বই

```
কল্পনা করুন, ডাকঘরে একটি বিশাল ঠিকানা বই আছে।
কেউ "ঢাকা..." বললেই কর্মচারী বলে:

  "ঢাকা" দিয়ে শুরু হওয়া এলাকা:
   → ঢাকা মিরপুর
   → ঢাকা গুলশান
   → ঢাকা ধানমন্ডি
   → ঢাকা উত্তরা

  কর্মচারী এই সাজেশন দ্রুত দিতে পারে কারণ
  ঠিকানাগুলো বর্ণানুক্রমে (Trie) সাজানো আছে!
```

---

## 🌳 Trie ডেটা স্ট্রাকচার

**Trie** (প্রিফিক্স ট্রি) হলো অটোকমপ্লিটের মূল ডেটা স্ট্রাকচার। প্রতিটি নোড একটি অক্ষর ধারণ করে এবং রুট থেকে পাতা পর্যন্ত পথ একটি সম্পূর্ণ শব্দ তৈরি করে।

```
          Trie Structure ("mobile", "money", "motor")
          =============================================

                       (root)
                         │
                         m
                        ╱ ╲
                       o    (অন্যান্য)
                      ╱ ╲
                     b    n    t
                     │    │    │
                     i    e    o
                     │    │    │
                     l    y    r
                     │   ✅    │
                     e         (...)
                     ✅

  "mo" টাইপ করলে → b, n, t শাখায় যাও → mobile, money, motor সাজেশন
```

### Trie-এ প্রিফিক্স সার্চের সময় জটিলতা

```
  সার্চ: O(L) — L হলো প্রিফিক্সের দৈর্ঘ্য
  ইনসার্ট: O(L)
  স্পেস: O(ALPHABET_SIZE × L × N) — N হলো মোট শব্দ সংখ্যা

  তুলনা:
    ডেটাবেস LIKE "%মোবা%" → O(N) — সব রেকর্ড স্ক্যান 😰
    Trie "মোবা"           → O(4)  — মাত্র ৪ ধাপ ✅
```

---

## 📊 সিস্টেম আর্কিটেকচার

```
        অটোকমপ্লিট সিস্টেম আর্কিটেকচার
        ==================================

  ইউজার ──→ API Gateway ──→ Autocomplete Service
   │                              │
   │ "মোবা"                       ├──→ Trie Cache (Redis)
   │                              │     (দ্রুত প্রিফিক্স সার্চ)
   │                              │
   │                              └──→ Elasticsearch
   │                                    (ফাজি সার্চ, টাইপো সহনশীল)
   │
   │         ┌────────────────────────┐
   └────────→│ Analytics Service      │
             │ (সার্চ ফ্রিকোয়েন্সি ট্র্যাক) │
             └────────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │ Trie Rebuild  │ (প্রতি ১৫ মিনিটে জনপ্রিয়তা অনুযায়ী)
              │ Worker        │
              └───────────────┘
```

---

## 🔍 Elasticsearch ধারণা

Elasticsearch হলো ডিস্ট্রিবিউটেড সার্চ ইঞ্জিন যা Trie-এর চেয়ে বেশি ফিচার দেয়:

```
সুবিধা:
  ✅ ফাজি সার্চ — "mobail" লিখলেও "mobile" খুঁজে পাবে
  ✅ টোকেনাইজেশন — "red mobile phone" → "red", "mobile", "phone"
  ✅ স্কোরিং — জনপ্রিয়তা অনুযায়ী র‍্যাংকিং
  ✅ ডিস্ট্রিবিউটেড — বিলিয়ন ডকুমেন্ট হ্যান্ডেল করে

তুলনা:
  Trie           → দ্রুত প্রিফিক্স ম্যাচ, সাধারণ ব্যবহার
  Elasticsearch  → ফুল-টেক্সট সার্চ, ফাজি ম্যাচ, জটিল কোয়েরি
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

// Trie নোড
class TrieNode
{
    public array $children = [];
    public bool $isEnd = false;
    public int $frequency = 0;
    public string $word = '';
}

// Trie ভিত্তিক অটোকমপ্লিট
class AutocompleteService
{
    private TrieNode $root;

    public function __construct()
    {
        $this->root = new TrieNode();
    }

    public function insert(string $word, int $frequency = 1): void
    {
        $node = $this->root;
        $chars = mb_str_split($word);
        foreach ($chars as $char) {
            if (!isset($node->children[$char])) {
                $node->children[$char] = new TrieNode();
            }
            $node = $node->children[$char];
        }
        $node->isEnd = true;
        $node->frequency += $frequency;
        $node->word = $word;
    }

    public function search(string $prefix, int $limit = 5): array
    {
        $node = $this->root;
        $chars = mb_str_split($prefix);

        // প্রিফিক্স নোডে পৌঁছাও
        foreach ($chars as $char) {
            if (!isset($node->children[$char])) {
                return [];
            }
            $node = $node->children[$char];
        }

        // DFS দিয়ে সব সাজেশন খুঁজো
        $results = [];
        $this->dfs($node, $results);

        // জনপ্রিয়তা অনুযায়ী সাজাও
        usort($results, fn($a, $b) => $b['frequency'] - $a['frequency']);
        return array_slice($results, 0, $limit);
    }

    private function dfs(TrieNode $node, array &$results): void
    {
        if ($node->isEnd) {
            $results[] = [
                'word' => $node->word,
                'frequency' => $node->frequency,
            ];
        }
        foreach ($node->children as $child) {
            $this->dfs($child, $results);
        }
    }
}

// ব্যবহার
$autocomplete = new AutocompleteService();

// জনপ্রিয় সার্চ টার্ম ইনসার্ট
$autocomplete->insert('মোবাইল ফোন', 5000);
$autocomplete->insert('মোবাইল চার্জার', 3000);
$autocomplete->insert('মোবাইল কভার', 2500);
$autocomplete->insert('মোটরসাইকেল', 1500);

// সার্চ
$suggestions = $autocomplete->search('মোবা', 3);
// ফলাফল: মোবাইল ফোন (5000), মোবাইল চার্জার (3000), মোবাইল কভার (2500)

// Redis ক্যাশিং সহ API এন্ডপয়েন্ট
function handleAutocomplete(string $prefix): string
{
    $redis = new Redis();
    $redis->connect('127.0.0.1');

    $cacheKey = "autocomplete:" . md5($prefix);
    $cached = $redis->get($cacheKey);

    if ($cached) {
        return $cached;
    }

    $autocomplete = new AutocompleteService();
    // ডেটাবেস থেকে Trie তৈরি করো...
    $results = $autocomplete->search($prefix);
    $json = json_encode($results);

    $redis->setex($cacheKey, 300, $json); // ৫ মিনিট ক্যাশ
    return $json;
}
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
// Trie ভিত্তিক অটোকমপ্লিট (ক্লায়েন্ট সাইড)
class TrieNode {
  constructor() {
    this.children = new Map();
    this.isEnd = false;
    this.frequency = 0;
    this.word = '';
  }
}

class AutocompleteClient {
  constructor() {
    this.root = new TrieNode();
    this.cache = new Map();
  }

  insert(word, frequency = 1) {
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char);
    }
    node.isEnd = true;
    node.frequency += frequency;
    node.word = word;
  }

  search(prefix, limit = 5) {
    // ক্যাশে আছে কিনা দেখো
    if (this.cache.has(prefix)) return this.cache.get(prefix);

    let node = this.root;
    for (const char of prefix) {
      if (!node.children.has(char)) return [];
      node = node.children.get(char);
    }

    const results = [];
    this._dfs(node, results);
    results.sort((a, b) => b.frequency - a.frequency);
    const topResults = results.slice(0, limit);

    this.cache.set(prefix, topResults);
    return topResults;
  }

  _dfs(node, results) {
    if (node.isEnd) {
      results.push({ word: node.word, frequency: node.frequency });
    }
    for (const child of node.children.values()) {
      this._dfs(child, results);
    }
  }
}

// ডিবাউন্স সহ UI ইন্টিগ্রেশন
class SearchBox {
  constructor(inputElement, resultsElement) {
    this.input = inputElement;
    this.results = resultsElement;
    this.debounceTimer = null;
    this.trie = new AutocompleteClient();

    this.input.addEventListener('input', (e) => this._onInput(e));
  }

  _onInput(event) {
    const query = event.target.value.trim();
    clearTimeout(this.debounceTimer);

    if (query.length < 2) {
      this.results.innerHTML = '';
      return;
    }

    // ২০০ms ডিবাউন্স — প্রতিটি কীস্ট্রোকে API কল এড়াও
    this.debounceTimer = setTimeout(async () => {
      const suggestions = await this._fetchSuggestions(query);
      this._renderSuggestions(suggestions);
    }, 200);
  }

  async _fetchSuggestions(query) {
    // প্রথমে লোকাল Trie থেকে চেষ্টা
    const local = this.trie.search(query);
    if (local.length > 0) return local;

    // না পেলে সার্ভার থেকে আনো
    const res = await fetch(`/api/autocomplete?q=${encodeURIComponent(query)}`);
    const data = await res.json();

    // ভবিষ্যতের জন্য Trie-তে যোগ করো
    data.forEach((item) => this.trie.insert(item.word, item.frequency));
    return data;
  }

  _renderSuggestions(suggestions) {
    this.results.innerHTML = suggestions
      .map((s) => `<div class="suggestion" data-word="${s.word}">${s.word}</div>`)
      .join('');
  }
}

// ব্যবহার
const searchBox = new SearchBox(
  document.getElementById('search-input'),
  document.getElementById('search-results')
);
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | পদ্ধতি |
|---|---|
| সাধারণ প্রিফিক্স সার্চ | Trie |
| টাইপো সহনশীল সার্চ | Elasticsearch |
| খুব দ্রুত রেসপন্স (<৫০ms) | Redis Sorted Set + Trie |
| ই-কমার্স প্রোডাক্ট সার্চ | Elasticsearch + Trie Hybrid |
| ছোট ডেটাসেট (<১০K শব্দ) | ক্লায়েন্ট-সাইড Trie |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| ফুল-টেক্সট সার্চ | Trie শুধু প্রিফিক্স ম্যাচ করে |
| ডিবাউন্স ছাড়া API কল | প্রতি কীস্ট্রোকে সার্ভারে লোড |
| বিশাল ডেটাসেটে শুধু Trie | মেমরি বেশি লাগে, Elasticsearch ভালো |
| সিকিউরিটি সেনসিটিভ সার্চ | ইউজারের সার্চ হিস্ট্রি লিক হতে পারে |

---

## 🔑 মনে রাখার পয়েন্ট

1. **Trie** — প্রিফিক্স সার্চের জন্য সবচেয়ে দক্ষ ডেটা স্ট্রাকচার
2. **ডিবাউন্স** — প্রতি কীস্ট্রোকে API কল করো না, ২০০-৩০০ms অপেক্ষা করো
3. **ফ্রিকোয়েন্সি র‍্যাংকিং** — জনপ্রিয় সার্চ আগে দেখাও
4. **ক্যাশিং** — সাধারণ প্রিফিক্স ক্যাশ করো (Redis/ক্লায়েন্ট)
5. ইন্টারভিউতে — Trie, Debounce, Ranking, Data Collection Pipeline আলোচনা করুন
