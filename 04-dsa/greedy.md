# 🤑 গ্রিডি অ্যালগরিদম (Greedy Algorithm)

> _"প্রতিটি ধাপে সেই মুহূর্তে যেটা সবচেয়ে ভালো মনে হয়, সেটাই বেছে নাও — পেছনে ফিরে তাকিও না।"_ 💰

---

## 📖 সংজ্ঞা

**গ্রিডি অ্যালগরিদম** হলো এমন একটি কৌশল যেখানে প্রতিটি ধাপে **লোকালি অপটিমাল (সেই মুহূর্তে সেরা)** সিদ্ধান্ত নেওয়া হয়, এই আশায় যে এটি **গ্লোবালি অপটিমাল** সমাধানে পৌঁছাবে।

গ্রিডি সব সমস্যায় কাজ করে না — শুধুমাত্র **Greedy Choice Property** এবং **Optimal Substructure** থাকলেই কাজ করে।

## 🏠 বাস্তব উদাহরণ

| বাস্তব জগৎ | গ্রিডি-এর সাথে তুলনা |
|---|---|
| ভাংতি দেওয়া | সবচেয়ে বড় নোট/কয়েন আগে দাও |
| বুফে খাওয়া | সবচেয়ে পছন্দের আইটেম আগে নাও |
| ফ্রিল্যান্সার কাজ বাছাই | সবচেয়ে বেশি পে-এর কাজ আগে নাও |
| পরীক্ষায় সময় বণ্টন | সবচেয়ে সহজ প্রশ্ন আগে সমাধান |

---

## ১. গ্রিডি-এর মৌলিক ধারণা

### 🔑 দুটি শর্ত

```
১. Greedy Choice Property:
   প্রতিটি ধাপে লোকালি সেরা পছন্দ → গ্লোবালি সেরা ফলাফল

২. Optimal Substructure:
   সমস্যার অপটিমাল সমাধান = সাব-প্রবলেমগুলোর অপটিমাল সমাধানের সমন্বয়
```

### 🆚 Greedy vs Dynamic Programming

| বৈশিষ্ট্য | Greedy | DP |
|---|---|---|
| সিদ্ধান্ত | লোকাল সেরা | সব অপশন বিবেচনা |
| পেছনে যাওয়া | ❌ না | ✅ হ্যাঁ |
| গতি | সাধারণত দ্রুত | তুলনামূলক ধীর |
| সব সমস্যায় কাজ করে? | ❌ | ✅ |
| উদাহরণ | Huffman, Kruskal | Knapsack 0/1, LCS |

---

## ২. Activity Selection Problem — কার্যক্রম নির্বাচন

**সমস্যা**: অনেকগুলো কার্যক্রমের শুরু ও শেষ সময় দেওয়া আছে। সর্বাধিক কয়টি কার্যক্রম করা যায় যেন কোনোটি ওভারল্যাপ না করে?

**গ্রিডি কৌশল**: শেষ সময় অনুযায়ী সর্ট করো, আগে শেষ হয় এমনটি আগে বাছাই করো।

```php
<?php
function activitySelection(array $activities): array {
    // শেষ সময় অনুযায়ী সর্ট
    usort($activities, fn($a, $b) => $a['end'] - $b['end']);

    $selected = [$activities[0]];
    $lastEnd = $activities[0]['end'];

    for ($i = 1; $i < count($activities); $i++) {
        if ($activities[$i]['start'] >= $lastEnd) {
            $selected[] = $activities[$i];
            $lastEnd = $activities[$i]['end'];
        }
    }

    return $selected;
}

$activities = [
    ['name' => 'মিটিং-১', 'start' => 1, 'end' => 4],
    ['name' => 'মিটিং-২', 'start' => 3, 'end' => 5],
    ['name' => 'মিটিং-৩', 'start' => 0, 'end' => 6],
    ['name' => 'মিটিং-৪', 'start' => 5, 'end' => 7],
    ['name' => 'মিটিং-৫', 'start' => 8, 'end' => 9],
    ['name' => 'মিটিং-৬', 'start' => 5, 'end' => 9],
];

$result = activitySelection($activities);
foreach ($result as $a) {
    echo "{$a['name']} ({$a['start']}-{$a['end']})\n";
}
// মিটিং-১ (1-4), মিটিং-৪ (5-7), মিটিং-৫ (8-9)
```

```javascript
function activitySelection(activities) {
    activities.sort((a, b) => a.end - b.end);

    const selected = [activities[0]];
    let lastEnd = activities[0].end;

    for (let i = 1; i < activities.length; i++) {
        if (activities[i].start >= lastEnd) {
            selected.push(activities[i]);
            lastEnd = activities[i].end;
        }
    }

    return selected;
}

const activities = [
    { name: "ক্লাস", start: 1, end: 3 },
    { name: "ল্যাব", start: 2, end: 5 },
    { name: "সেমিনার", start: 4, end: 6 },
    { name: "খেলা", start: 6, end: 8 },
    { name: "পড়া", start: 5, end: 7 },
];

console.log(activitySelection(activities));
// ক্লাস (1-3), সেমিনার (4-6), খেলা (6-8)
```

---

## ৩. Fractional Knapsack — ভগ্নাংশ ন্যাপস্যাক

**সমস্যা**: একটি ব্যাগে নির্দিষ্ট ওজন ধরে। প্রতিটি জিনিসের ওজন ও মূল্য আছে। জিনিসকে ভাঙা (fraction) যায়। সর্বোচ্চ মূল্যের জিনিস ব্যাগে ভরো।

**গ্রিডি কৌশল**: প্রতি কেজিতে সবচেয়ে বেশি দামি জিনিস আগে নাও (value/weight অনুযায়ী সর্ট)।

```php
<?php
function fractionalKnapsack(array $items, int $capacity): float {
    // মূল্য/ওজন অনুপাত অনুযায়ী নিম্নক্রমে সর্ট
    usort($items, fn($a, $b) => ($b['value'] / $b['weight']) - ($a['value'] / $a['weight']));

    $totalValue = 0;
    $remaining = $capacity;

    foreach ($items as $item) {
        if ($remaining <= 0) break;

        if ($item['weight'] <= $remaining) {
            $totalValue += $item['value'];
            $remaining -= $item['weight'];
        } else {
            // ভগ্নাংশ নেওয়া
            $fraction = $remaining / $item['weight'];
            $totalValue += $item['value'] * $fraction;
            $remaining = 0;
        }
    }

    return $totalValue;
}

$items = [
    ['name' => 'স্বর্ণ', 'weight' => 10, 'value' => 500],
    ['name' => 'রূপা', 'weight' => 20, 'value' => 400],
    ['name' => 'তামা', 'weight' => 30, 'value' => 300],
];

echo "সর্বোচ্চ মূল্য: " . fractionalKnapsack($items, 35);
// স্বর্ণ পুরোটা (500) + রূপার 25kg এর 12.5kg (250) = 750
```

```javascript
function fractionalKnapsack(items, capacity) {
    items.sort((a, b) => (b.value / b.weight) - (a.value / a.weight));

    let totalValue = 0;
    let remaining = capacity;

    for (const item of items) {
        if (remaining <= 0) break;

        if (item.weight <= remaining) {
            totalValue += item.value;
            remaining -= item.weight;
        } else {
            const fraction = remaining / item.weight;
            totalValue += item.value * fraction;
            remaining = 0;
        }
    }

    return totalValue;
}

const items = [
    { name: "ল্যাপটপ", weight: 3, value: 2000 },
    { name: "ক্যামেরা", weight: 1, value: 1500 },
    { name: "বই", weight: 4, value: 300 },
];

console.log(fractionalKnapsack(items, 4));
// ক্যামেরা (1500) + ল্যাপটপের ৩ কেজির ৩ কেজি (2000) = 3500
```

---

## ৪. Huffman Coding — হাফম্যান কোডিং

**সমস্যা**: টেক্সট কম্প্রেশনের জন্য প্রতিটি ক্যারেক্টারকে ভ্যারিয়েবল-লেংথ বাইনারি কোড দেওয়া, যেখানে বেশি ফ্রিকোয়েন্সির অক্ষর কম বিট পায়।

```
উদাহরণ: "AABBBCCCC"
A: 2 বার, B: 3 বার, C: 4 বার

হাফম্যান ট্রি:
          [9]
         /   \
       C[4]  [5]
             / \
           A[2] B[3]

কোড: C = 0, A = 10, B = 11
এনকোডেড: 10 10 11 11 11 0 0 0 0 = 14 বিট
ফিক্সড কোডে: 2 বিট × 9 = 18 বিট (সাশ্রয়!)
```

```php
<?php
function huffmanCoding(array $freq): array {
    // Min-Heap সিমুলেশন (SplPriorityQueue ব্যবহার)
    $pq = new SplPriorityQueue();
    $pq->setExtractFlags(SplPriorityQueue::EXTR_BOTH);

    foreach ($freq as $char => $count) {
        $pq->insert(['char' => $char, 'left' => null, 'right' => null], -$count);
    }

    while ($pq->count() > 1) {
        $left = $pq->extract();
        $right = $pq->extract();

        $merged = [
            'char' => null,
            'left' => $left['data'],
            'right' => $right['data'],
        ];

        $pq->insert($merged, $left['priority'] + $right['priority']);
    }

    $root = $pq->extract()['data'];
    $codes = [];
    generateCodes($root, '', $codes);
    return $codes;
}

function generateCodes($node, string $code, array &$codes): void {
    if ($node['char'] !== null) {
        $codes[$node['char']] = $code ?: '0';
        return;
    }
    if ($node['left']) generateCodes($node['left'], $code . '0', $codes);
    if ($node['right']) generateCodes($node['right'], $code . '1', $codes);
}

$freq = ['A' => 5, 'B' => 9, 'C' => 12, 'D' => 13, 'E' => 16, 'F' => 45];
$codes = huffmanCoding($freq);
print_r($codes);
```

```javascript
class HuffmanNode {
    constructor(char, freq) {
        this.char = char;
        this.freq = freq;
        this.left = null;
        this.right = null;
    }
}

function huffmanCoding(freq) {
    let nodes = Object.entries(freq).map(([char, f]) => new HuffmanNode(char, f));

    while (nodes.length > 1) {
        nodes.sort((a, b) => a.freq - b.freq);

        const left = nodes.shift();
        const right = nodes.shift();

        const parent = new HuffmanNode(null, left.freq + right.freq);
        parent.left = left;
        parent.right = right;

        nodes.push(parent);
    }

    const codes = {};
    generateCodes(nodes[0], "", codes);
    return codes;
}

function generateCodes(node, code, codes) {
    if (!node) return;
    if (node.char !== null) {
        codes[node.char] = code || "0";
        return;
    }
    generateCodes(node.left, code + "0", codes);
    generateCodes(node.right, code + "1", codes);
}

const freq = { A: 5, B: 9, C: 12, D: 13, E: 16, F: 45 };
console.log(huffmanCoding(freq));
// F সবচেয়ে বেশি → সবচেয়ে কম বিট
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন গ্রিডি |
|---|---|
| Activity Selection | শেষ সময় দিয়ে সর্ট → অপটিমাল |
| Fractional Knapsack | ভাঙা যায় বলে গ্রিডি কাজ করে |
| Huffman Coding | ফ্রিকোয়েন্সি-ভিত্তিক কম্প্রেশন |
| Minimum Spanning Tree | Kruskal / Prim |
| Dijkstra's Algorithm | সবচেয়ে কাছের নোড আগে |
| ভাংতি দেওয়া (Coin Change — canonical) | বড় কয়েন আগে |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন না |
|---|---|
| 0/1 Knapsack | ভাঙা যায় না → DP দরকার |
| সব কম্বিনেশন বিবেচনা দরকার | গ্রিডি শুধু একটি পথ দেখে |
| সমস্যায় Greedy Choice Property নেই | ভুল ফলাফল দিতে পারে |
| Coin Change (non-canonical) | যেমন coins = [1, 3, 4], amount = 6 → গ্রিডি ভুল দেয় |

---

## 🧠 মনে রাখার টিপস

```
গ্রিডি = প্রতি ধাপে সেরা → পুরো সমাধান সেরা (সবসময় নয়!)
গ্রিডি কাজ করে কিনা প্রমাণ করো → তারপর কোড লেখো
সন্দেহ হলে → DP ব্যবহার করো (নিরাপদ)
Activity Selection → শেষ সময়ে সর্ট
Fractional Knapsack → value/weight-এ সর্ট
Huffman → কম ফ্রিকোয়েন্সি আগে merge
```
