# 📊 গ্রাফ (Graph) - সম্পূর্ণ বাংলা গাইড

## সূচিপত্র
- [১. গ্রাফের মৌলিক ধারণা](#১-গ্রাফের-মৌলিক-ধারণা)
- [২. গ্রাফ রিপ্রেজেন্টেশন](#২-গ্রাফ-রিপ্রেজেন্টেশন)
- [৩. BFS (Breadth-First Search)](#৩-bfs-breadth-first-search)
- [৪. DFS (Depth-First Search)](#৪-dfs-depth-first-search)
- [৫. শর্টেস্ট পাথ অ্যালগরিদম](#৫-শর্টেস্ট-পাথ-অ্যালগরিদম)
- [৬. টপোলজিক্যাল সর্ট](#৬-টপোলজিক্যাল-সর্ট)
- [৭. মিনিমাম স্প্যানিং ট্রি](#৭-মিনিমাম-স্প্যানিং-ট্রি)
- [৮. অ্যাডভান্সড সমস্যা](#৮-অ্যাডভান্সড-সমস্যা)
- [৯. সম্পূর্ণ প্রজেক্ট: রুট ফাইন্ডার](#৯-সম্পূর্ণ-প্রজেক্ট-রুট-ফাইন্ডার)
- [চিটশিট ও ইন্টারভিউ টিপস](#চিটশিট-ও-ইন্টারভিউ-টিপস)

---

## ১. গ্রাফের মৌলিক ধারণা

### গ্রাফ কী?

গ্রাফ হলো একটি নন-লিনিয়ার ডেটা স্ট্রাকচার যেখানে **নোড** (Vertex) এবং **এজ** (Edge) থাকে।
নোডগুলো বিভিন্ন সত্তা (entity) প্রকাশ করে এবং এজগুলো তাদের মধ্যে সম্পর্ক দেখায়।

```
গাণিতিক সংজ্ঞা: G = (V, E)
যেখানে V = নোডের সেট, E = এজের সেট
```

### বাস্তব জীবনে গ্রাফের উদাহরণ

| উদাহরণ | নোড | এজ |
|---------|------|-----|
| সোশ্যাল নেটওয়ার্ক | ব্যবহারকারী | বন্ধুত্ব |
| গুগল ম্যাপ | শহর/স্থান | রাস্তা |
| ওয়েবসাইট | পেজ | লিংক |
| বিদ্যুৎ গ্রিড | সাবস্টেশন | তার |

### গ্রাফের প্রকারভেদ

```
১. আনডাইরেক্টেড গ্রাফ (Undirected Graph)
   - এজের কোনো দিক নেই
   - A --- B মানে A থেকে B এবং B থেকে A উভয়ই যাওয়া যায়

    A ---- B
    |      |
    |      |
    C ---- D

২. ডাইরেক্টেড গ্রাফ (Directed Graph / Digraph)
   - এজের নির্দিষ্ট দিক আছে
   - A --> B মানে শুধু A থেকে B যাওয়া যায়

    A ----> B
    |       |
    v       v
    C ----> D

৩. ওয়েটেড গ্রাফ (Weighted Graph)
   - প্রতিটি এজের একটি ওজন/খরচ আছে

    A --5-- B
    |       |
    3       7
    |       |
    C --2-- D

৪. আনওয়েটেড গ্রাফ (Unweighted Graph)
   - সব এজের ওজন সমান (সাধারণত ১)

৫. সাইক্লিক গ্রাফ (Cyclic Graph)
   - গ্রাফে চক্র আছে: A -> B -> C -> A

৬. অ্যাসাইক্লিক গ্রাফ (Acyclic Graph)
   - কোনো চক্র নেই
   - DAG (Directed Acyclic Graph) খুবই গুরুত্বপূর্ণ

৭. কানেক্টেড গ্রাফ (Connected Graph)
   - যেকোনো নোড থেকে অন্য যেকোনো নোডে যাওয়া যায়

৮. ডিসকানেক্টেড গ্রাফ (Disconnected Graph)
   - কিছু নোড বিচ্ছিন্ন

    A --- B     D --- E
    |           |
    C           F
```

### মূল পরিভাষা

| পরিভাষা | ইংরেজি | বিবরণ |
|----------|---------|--------|
| শীর্ষবিন্দু | Vertex/Node | গ্রাফের প্রতিটি উপাদান |
| ধার/সংযোগ | Edge | দুটি নোডের মধ্যে সংযোগ |
| ডিগ্রি | Degree | একটি নোডের সাথে কতটি এজ যুক্ত |
| ইন-ডিগ্রি | In-degree | কতটি এজ নোডে আসছে (ডাইরেক্টেড) |
| আউট-ডিগ্রি | Out-degree | কতটি এজ নোড থেকে যাচ্ছে (ডাইরেক্টেড) |
| পাথ | Path | এক নোড থেকে অন্য নোডে যাওয়ার পথ |
| সাইকেল | Cycle | যে পাথ শুরুর নোডে ফিরে আসে |
| প্রতিবেশী | Neighbor/Adjacent | সরাসরি সংযুক্ত নোড |

---

## ২. গ্রাফ রিপ্রেজেন্টেশন

গ্রাফকে কম্পিউটারে তিনভাবে উপস্থাপন করা যায়।

### উদাহরণ গ্রাফ

```
    0 ---- 1
    |  \   |
    |   \  |
    3 ---- 2
```

### ২.১ অ্যাডজেসেন্সি ম্যাট্রিক্স (Adjacency Matrix)

একটি 2D অ্যারে যেখানে `matrix[i][j] = 1` মানে i থেকে j তে এজ আছে।

```
     0  1  2  3
  0 [0, 1, 1, 1]
  1 [1, 0, 1, 0]
  2 [1, 1, 0, 1]
  3 [1, 0, 1, 0]
```

**PHP বাস্তবায়ন:**

```php
<?php
// গ্রাফ ক্লাস - অ্যাডজেসেন্সি ম্যাট্রিক্স দিয়ে
class GraphMatrix {
    private $matrix;  // ম্যাট্রিক্স সংরক্ষণ
    private $size;    // নোড সংখ্যা

    // কনস্ট্রাক্টর - নোড সংখ্যা নিয়ে ম্যাট্রিক্স তৈরি
    public function __construct($size) {
        $this->size = $size;
        // সব ঘরে ০ বসাও
        $this->matrix = array_fill(0, $size, array_fill(0, $size, 0));
    }

    // এজ যোগ করো (আনডাইরেক্টেড)
    public function addEdge($u, $v) {
        $this->matrix[$u][$v] = 1;
        $this->matrix[$v][$u] = 1; // উভয় দিকে
    }

    // এজ আছে কিনা দেখো
    public function hasEdge($u, $v) {
        return $this->matrix[$u][$v] === 1;
    }

    // ম্যাট্রিক্স প্রিন্ট করো
    public function display() {
        echo "অ্যাডজেসেন্সি ম্যাট্রিক্স:\n";
        for ($i = 0; $i < $this->size; $i++) {
            echo implode(", ", $this->matrix[$i]) . "\n";
        }
    }

    // নির্দিষ্ট নোডের প্রতিবেশী খোঁজো
    public function getNeighbors($node) {
        $neighbors = [];
        for ($i = 0; $i < $this->size; $i++) {
            if ($this->matrix[$node][$i] === 1) {
                $neighbors[] = $i;
            }
        }
        return $neighbors;
    }
}

// ব্যবহার
$graph = new GraphMatrix(4);
$graph->addEdge(0, 1);
$graph->addEdge(0, 2);
$graph->addEdge(0, 3);
$graph->addEdge(1, 2);
$graph->addEdge(2, 3);

$graph->display();
echo "নোড ০ এর প্রতিবেশী: " . implode(", ", $graph->getNeighbors(0)) . "\n";
?>
```

**JavaScript বাস্তবায়ন:**

```javascript
// গ্রাফ ক্লাস - অ্যাডজেসেন্সি ম্যাট্রিক্স দিয়ে
class GraphMatrix {
    constructor(size) {
        this.size = size;
        // size x size ম্যাট্রিক্স তৈরি, সব ০ দিয়ে ভরা
        this.matrix = Array.from({ length: size }, () => Array(size).fill(0));
    }

    // এজ যোগ করো (আনডাইরেক্টেড)
    addEdge(u, v) {
        this.matrix[u][v] = 1;
        this.matrix[v][u] = 1; // উভয় দিকে
    }

    // এজ আছে কিনা পরীক্ষা
    hasEdge(u, v) {
        return this.matrix[u][v] === 1;
    }

    // ম্যাট্রিক্স দেখাও
    display() {
        console.log("অ্যাডজেসেন্সি ম্যাট্রিক্স:");
        this.matrix.forEach((row, i) => console.log(`${i}: [${row.join(", ")}]`));
    }

    // প্রতিবেশী খোঁজো
    getNeighbors(node) {
        const neighbors = [];
        for (let i = 0; i < this.size; i++) {
            if (this.matrix[node][i] === 1) neighbors.push(i);
        }
        return neighbors;
    }
}

// ব্যবহার
const graph = new GraphMatrix(4);
graph.addEdge(0, 1);
graph.addEdge(0, 2);
graph.addEdge(0, 3);
graph.addEdge(1, 2);
graph.addEdge(2, 3);

graph.display();
console.log("নোড ০ এর প্রতিবেশী:", graph.getNeighbors(0));
```

### ২.২ অ্যাডজেসেন্সি লিস্ট (Adjacency List)

প্রতিটি নোডের জন্য তার প্রতিবেশীদের একটি তালিকা।

```
0 -> [1, 2, 3]
1 -> [0, 2]
2 -> [0, 1, 3]
3 -> [0, 2]
```

**PHP বাস্তবায়ন:**

```php
<?php
// গ্রাফ ক্লাস - অ্যাডজেসেন্সি লিস্ট দিয়ে
class GraphList {
    private $adjList = []; // প্রতিবেশী তালিকা

    // নোড যোগ করো
    public function addNode($node) {
        if (!isset($this->adjList[$node])) {
            $this->adjList[$node] = [];
        }
    }

    // এজ যোগ করো (আনডাইরেক্টেড)
    public function addEdge($u, $v) {
        $this->addNode($u);
        $this->addNode($v);
        $this->adjList[$u][] = $v;
        $this->adjList[$v][] = $u;
    }

    // প্রতিবেশী নাও
    public function getNeighbors($node) {
        return $this->adjList[$node] ?? [];
    }

    // গ্রাফ দেখাও
    public function display() {
        echo "অ্যাডজেসেন্সি লিস্ট:\n";
        foreach ($this->adjList as $node => $neighbors) {
            echo "$node -> [" . implode(", ", $neighbors) . "]\n";
        }
    }
}

$graph = new GraphList();
$graph->addEdge(0, 1);
$graph->addEdge(0, 2);
$graph->addEdge(0, 3);
$graph->addEdge(1, 2);
$graph->addEdge(2, 3);
$graph->display();
?>
```

**JavaScript বাস্তবায়ন:**

```javascript
// গ্রাফ ক্লাস - অ্যাডজেসেন্সি লিস্ট দিয়ে
class GraphList {
    constructor() {
        this.adjList = new Map(); // ম্যাপ দিয়ে তালিকা সংরক্ষণ
    }

    // নোড যোগ করো
    addNode(node) {
        if (!this.adjList.has(node)) {
            this.adjList.set(node, []);
        }
    }

    // এজ যোগ করো (আনডাইরেক্টেড)
    addEdge(u, v) {
        this.addNode(u);
        this.addNode(v);
        this.adjList.get(u).push(v);
        this.adjList.get(v).push(u);
    }

    // প্রতিবেশী নাও
    getNeighbors(node) {
        return this.adjList.get(node) || [];
    }

    // গ্রাফ দেখাও
    display() {
        console.log("অ্যাডজেসেন্সি লিস্ট:");
        for (const [node, neighbors] of this.adjList) {
            console.log(`${node} -> [${neighbors.join(", ")}]`);
        }
    }
}

const graph = new GraphList();
graph.addEdge(0, 1);
graph.addEdge(0, 2);
graph.addEdge(0, 3);
graph.addEdge(1, 2);
graph.addEdge(2, 3);
graph.display();
```

### ২.৩ এজ লিস্ট (Edge List)

সব এজকে একটি তালিকায় রাখা হয়।

```
এজ লিস্ট: [(0,1), (0,2), (0,3), (1,2), (2,3)]
```

**PHP বাস্তবায়ন:**

```php
<?php
// এজ লিস্ট ভিত্তিক গ্রাফ
class GraphEdgeList {
    private $edges = []; // এজের তালিকা

    // এজ যোগ করো (ওজনসহ ঐচ্ছিক)
    public function addEdge($u, $v, $weight = 1) {
        $this->edges[] = [$u, $v, $weight];
    }

    // সব এজ নাও
    public function getEdges() {
        return $this->edges;
    }

    // এজ দেখাও
    public function display() {
        echo "এজ লিস্ট:\n";
        foreach ($this->edges as $edge) {
            echo "({$edge[0]}, {$edge[1]}) ওজন: {$edge[2]}\n";
        }
    }
}

$graph = new GraphEdgeList();
$graph->addEdge(0, 1, 5);
$graph->addEdge(0, 2, 3);
$graph->addEdge(1, 2, 7);
$graph->display();
?>
```

### রিপ্রেজেন্টেশন তুলনা

| বৈশিষ্ট্য | ম্যাট্রিক্স | লিস্ট | এজ লিস্ট |
|-----------|-------------|--------|----------|
| স্পেস | O(V²) | O(V + E) | O(E) |
| এজ চেক | O(1) | O(V) | O(E) |
| প্রতিবেশী খোঁজা | O(V) | O(ডিগ্রি) | O(E) |
| এজ যোগ | O(1) | O(1) | O(1) |
| ঘন গ্রাফে ভালো | ✅ | ❌ | ❌ |
| বিরল গ্রাফে ভালো | ❌ | ✅ | ✅ |

---

## ৩. BFS (Breadth-First Search)

### ধারণা

BFS হলো **স্তরভিত্তিক** (level-by-level) গ্রাফ ট্রাভার্সাল। এটি **কিউ (Queue)** ব্যবহার করে।
প্রথমে শুরু নোড ভিজিট করে, তারপর তার সব প্রতিবেশী, তারপর প্রতিবেশীদের প্রতিবেশী — এভাবে চলতে থাকে।

### ASCII ট্রেস

```
গ্রাফ:
    0 ---- 1
    |  \   |
    |   \  |
    3 ---- 2

BFS শুরু নোড: 0

ধাপ ১: কিউ = [0]          ভিজিটেড = {0}
        প্রসেস: 0
        প্রতিবেশী 1, 2, 3 কিউতে যোগ

ধাপ ২: কিউ = [1, 2, 3]    ভিজিটেড = {0, 1}
        প্রসেস: 1
        প্রতিবেশী 2 ইতিমধ্যে কিউতে আছে

ধাপ ৩: কিউ = [2, 3]       ভিজিটেড = {0, 1, 2}
        প্রসেস: 2
        সব প্রতিবেশী ভিজিটেড

ধাপ ৪: কিউ = [3]           ভিজিটেড = {0, 1, 2, 3}
        প্রসেস: 3
        সব প্রতিবেশী ভিজিটেড

ফলাফল: 0 -> 1 -> 2 -> 3

স্তর দেখানো:
    স্তর ০:    [0]
    স্তর ১:    [1, 2, 3]
```

### PHP বাস্তবায়ন

```php
<?php
// BFS ট্রাভার্সাল - অ্যাডজেসেন্সি লিস্ট দিয়ে
function bfs($graph, $start) {
    $visited = [];      // ভিজিটেড নোড ট্র্যাক
    $queue = [$start];  // কিউ তৈরি, শুরু নোড দিয়ে
    $visited[$start] = true;
    $result = [];       // ট্রাভার্সাল ক্রম

    while (!empty($queue)) {
        // কিউ থেকে প্রথম নোড বের করো
        $node = array_shift($queue);
        $result[] = $node;

        // প্রতিবেশীদের দেখো
        foreach ($graph[$node] as $neighbor) {
            if (!isset($visited[$neighbor])) {
                $visited[$neighbor] = true;
                $queue[] = $neighbor; // কিউতে যোগ করো
            }
        }
    }

    return $result;
}

// BFS দিয়ে শর্টেস্ট পাথ (আনওয়েটেড গ্রাফে)
function bfsShortestPath($graph, $start, $end) {
    $visited = [$start => true];
    $queue = [[$start, [$start]]]; // [নোড, পাথ]

    while (!empty($queue)) {
        [$node, $path] = array_shift($queue);

        if ($node === $end) {
            return $path; // পাথ পাওয়া গেছে!
        }

        foreach ($graph[$node] as $neighbor) {
            if (!isset($visited[$neighbor])) {
                $visited[$neighbor] = true;
                $newPath = array_merge($path, [$neighbor]);
                $queue[] = [$neighbor, $newPath];
            }
        }
    }

    return null; // পাথ নেই
}

// উদাহরণ গ্রাফ
$graph = [
    0 => [1, 2, 3],
    1 => [0, 2],
    2 => [0, 1, 3],
    3 => [0, 2],
];

echo "BFS ট্রাভার্সাল: " . implode(" -> ", bfs($graph, 0)) . "\n";
$path = bfsShortestPath($graph, 1, 3);
echo "১ থেকে ৩ এর শর্টেস্ট পাথ: " . implode(" -> ", $path) . "\n";
?>
```

### JavaScript বাস্তবায়ন

```javascript
// BFS ট্রাভার্সাল
function bfs(graph, start) {
    const visited = new Set([start]); // ভিজিটেড সেট
    const queue = [start];            // কিউ
    const result = [];                // ফলাফল

    while (queue.length > 0) {
        const node = queue.shift(); // কিউ থেকে বের করো
        result.push(node);

        // প্রতিবেশী ঘুরে দেখো
        for (const neighbor of graph.get(node) || []) {
            if (!visited.has(neighbor)) {
                visited.add(neighbor);
                queue.push(neighbor); // কিউতে ঢোকাও
            }
        }
    }

    return result;
}

// BFS দিয়ে শর্টেস্ট পাথ (আনওয়েটেড)
function bfsShortestPath(graph, start, end) {
    const visited = new Set([start]);
    const queue = [[start, [start]]]; // [নোড, পাথ]

    while (queue.length > 0) {
        const [node, path] = queue.shift();

        if (node === end) return path; // পাওয়া গেছে!

        for (const neighbor of graph.get(node) || []) {
            if (!visited.has(neighbor)) {
                visited.add(neighbor);
                queue.push([neighbor, [...path, neighbor]]);
            }
        }
    }

    return null; // পাথ নেই
}

// উদাহরণ
const graph = new Map([
    [0, [1, 2, 3]],
    [1, [0, 2]],
    [2, [0, 1, 3]],
    [3, [0, 2]],
]);

console.log("BFS ট্রাভার্সাল:", bfs(graph, 0).join(" -> "));
console.log("১ থেকে ৩ পাথ:", bfsShortestPath(graph, 1, 3).join(" -> "));
```

### BFS জটিলতা বিশ্লেষণ

| অপারেশন | টাইম | স্পেস |
|----------|-------|-------|
| ট্রাভার্সাল | O(V + E) | O(V) |
| শর্টেস্ট পাথ (আনওয়েটেড) | O(V + E) | O(V) |

---

## ৪. DFS (Depth-First Search)

### ধারণা

DFS হলো **গভীরতা-প্রথম** ট্রাভার্সাল। এটি **স্ট্যাক** বা **রিকার্শন** ব্যবহার করে।
একটি পথ ধরে যতদূর সম্ভব যায়, তারপর ব্যাকট্র্যাক করে অন্য পথ দেখে।

### ASCII ট্রেস

```
গ্রাফ:
    0 ---- 1
    |  \   |
    |   \  |
    3 ---- 2

DFS শুরু নোড: 0

রিকার্সিভ DFS ট্রেস:
dfs(0)
  ভিজিট: 0
  -> dfs(1)
       ভিজিট: 1
       -> dfs(2)
            ভিজিট: 2
            -> dfs(3)
                 ভিজিট: 3
                 সব প্রতিবেশী ভিজিটেড, ব্যাকট্র্যাক
            <- ব্যাকট্র্যাক
       <- ব্যাকট্র্যাক
  <- ব্যাকট্র্যাক

ফলাফল: 0 -> 1 -> 2 -> 3
```

### PHP বাস্তবায়ন (রিকার্সিভ ও ইটারেটিভ)

```php
<?php
// রিকার্সিভ DFS
function dfsRecursive($graph, $node, &$visited, &$result) {
    $visited[$node] = true;
    $result[] = $node;

    // প্রতিটি প্রতিবেশীতে যাও
    foreach ($graph[$node] as $neighbor) {
        if (!isset($visited[$neighbor])) {
            dfsRecursive($graph, $neighbor, $visited, $result);
        }
    }
}

// ইটারেটিভ DFS (স্ট্যাক দিয়ে)
function dfsIterative($graph, $start) {
    $visited = [];
    $stack = [$start]; // স্ট্যাক ব্যবহার
    $result = [];

    while (!empty($stack)) {
        $node = array_pop($stack); // স্ট্যাক থেকে বের করো

        if (isset($visited[$node])) continue;

        $visited[$node] = true;
        $result[] = $node;

        // প্রতিবেশী স্ট্যাকে পুশ করো (উল্টো ক্রমে)
        foreach (array_reverse($graph[$node]) as $neighbor) {
            if (!isset($visited[$neighbor])) {
                $stack[] = $neighbor;
            }
        }
    }

    return $result;
}

// DFS দিয়ে পাথ খোঁজা
function dfsPath($graph, $start, $end, &$visited = [], $path = []) {
    $visited[$start] = true;
    $path[] = $start;

    if ($start === $end) return $path;

    foreach ($graph[$start] as $neighbor) {
        if (!isset($visited[$neighbor])) {
            $found = dfsPath($graph, $neighbor, $end, $visited, $path);
            if ($found !== null) return $found;
        }
    }

    return null; // এই পথে পাওয়া যায়নি
}

// উদাহরণ
$graph = [
    0 => [1, 2, 3],
    1 => [0, 2],
    2 => [0, 1, 3],
    3 => [0, 2],
];

// রিকার্সিভ
$visited = [];
$result = [];
dfsRecursive($graph, 0, $visited, $result);
echo "রিকার্সিভ DFS: " . implode(" -> ", $result) . "\n";

// ইটারেটিভ
echo "ইটারেটিভ DFS: " . implode(" -> ", dfsIterative($graph, 0)) . "\n";

// পাথ খোঁজা
$visited = [];
$path = dfsPath($graph, 0, 3, $visited);
echo "০ থেকে ৩ পাথ: " . implode(" -> ", $path) . "\n";
?>
```

### JavaScript বাস্তবায়ন

```javascript
// রিকার্সিভ DFS
function dfsRecursive(graph, node, visited = new Set(), result = []) {
    visited.add(node);
    result.push(node);

    for (const neighbor of graph.get(node) || []) {
        if (!visited.has(neighbor)) {
            dfsRecursive(graph, neighbor, visited, result);
        }
    }

    return result;
}

// ইটারেটিভ DFS (স্ট্যাক দিয়ে)
function dfsIterative(graph, start) {
    const visited = new Set();
    const stack = [start]; // স্ট্যাক
    const result = [];

    while (stack.length > 0) {
        const node = stack.pop(); // উপর থেকে বের করো

        if (visited.has(node)) continue;

        visited.add(node);
        result.push(node);

        // উল্টো ক্রমে পুশ করো যাতে ক্রম ঠিক থাকে
        const neighbors = graph.get(node) || [];
        for (let i = neighbors.length - 1; i >= 0; i--) {
            if (!visited.has(neighbors[i])) {
                stack.push(neighbors[i]);
            }
        }
    }

    return result;
}

// DFS দিয়ে সব পাথ খোঁজা
function dfsAllPaths(graph, start, end, visited = new Set(), path = []) {
    visited.add(start);
    path.push(start);

    const allPaths = [];

    if (start === end) {
        allPaths.push([...path]); // পাথের কপি সংরক্ষণ
    } else {
        for (const neighbor of graph.get(start) || []) {
            if (!visited.has(neighbor)) {
                allPaths.push(...dfsAllPaths(graph, neighbor, end, visited, path));
            }
        }
    }

    // ব্যাকট্র্যাক
    path.pop();
    visited.delete(start);

    return allPaths;
}

// উদাহরণ
const graph = new Map([
    [0, [1, 2, 3]],
    [1, [0, 2]],
    [2, [0, 1, 3]],
    [3, [0, 2]],
]);

console.log("রিকার্সিভ DFS:", dfsRecursive(graph, 0).join(" -> "));
console.log("ইটারেটিভ DFS:", dfsIterative(graph, 0).join(" -> "));
console.log("সব পাথ (0->3):", dfsAllPaths(graph, 0, 3));
```

### BFS বনাম DFS তুলনা

| বৈশিষ্ট্য | BFS | DFS |
|-----------|-----|-----|
| ডেটা স্ট্রাকচার | কিউ | স্ট্যাক/রিকার্শন |
| ট্রাভার্সাল ধরন | স্তরভিত্তিক | গভীরতা-প্রথম |
| শর্টেস্ট পাথ | ✅ (আনওয়েটেড) | ❌ |
| স্পেস | O(V) (প্রশস্ত গ্রাফে বেশি) | O(V) (গভীর গ্রাফে বেশি) |
| সাইকেল ডিটেকশন | ✅ | ✅ |
| টপোলজিক্যাল সর্ট | ✅ (কান'স) | ✅ |
| কানেক্টেড কম্পোনেন্ট | ✅ | ✅ |

---

## ৫. শর্টেস্ট পাথ অ্যালগরিদম

### ৫.১ ডায়াক্সট্রা অ্যালগরিদম (Dijkstra's Algorithm)

**কখন ব্যবহার:** ওয়েটেড গ্রাফে শর্টেস্ট পাথ (নন-নেগেটিভ ওজন)

### ASCII ট্রেস

```
ওয়েটেড গ্রাফ:
    A --4-- B
    |       |
    2       1
    |       |
    C --3-- D --5-- E

ডায়াক্সট্রা (A থেকে শুরু):

ধাপ  | প্রসেস | দূরত্ব                              | আপডেট
-----|--------|--------------------------------------|--------
 ১   | A      | A=0, B=∞, C=∞, D=∞, E=∞           | B=4, C=2
 ২   | C      | A=0, B=4, C=2, D=∞, E=∞           | D=5 (2+3)
 ৩   | B      | A=0, B=4, C=2, D=5, E=∞           | D=5 (min(5,4+1)=5)
 ৪   | D      | A=0, B=4, C=2, D=5, E=∞           | E=10 (5+5)
 ৫   | E      | A=0, B=4, C=2, D=5, E=10          | শেষ

চূড়ান্ত শর্টেস্ট দূরত্ব A থেকে:
A=0, B=4, C=2, D=5, E=10
```

### PHP বাস্তবায়ন - ডায়াক্সট্রা

```php
<?php
// ডায়াক্সট্রা অ্যালগরিদম
function dijkstra($graph, $start) {
    $dist = [];     // দূরত্ব
    $prev = [];     // পূর্ববর্তী নোড (পাথ ট্র্যাকিং)
    $visited = [];  // ভিজিটেড

    // সব নোডের দূরত্ব অসীম সেট করো
    foreach (array_keys($graph) as $node) {
        $dist[$node] = PHP_INT_MAX;
        $prev[$node] = null;
    }
    $dist[$start] = 0;

    // সিম্পল প্রায়োরিটি কিউ (অ্যারে দিয়ে)
    $pq = [[$start, 0]]; // [নোড, দূরত্ব]

    while (!empty($pq)) {
        // সবচেয়ে কম দূরত্বের নোড বের করো
        usort($pq, fn($a, $b) => $a[1] - $b[1]);
        [$node, $d] = array_shift($pq);

        if (isset($visited[$node])) continue;
        $visited[$node] = true;

        // প্রতিবেশীদের দূরত্ব আপডেট করো
        foreach ($graph[$node] as [$neighbor, $weight]) {
            $newDist = $dist[$node] + $weight;
            if ($newDist < $dist[$neighbor]) {
                $dist[$neighbor] = $newDist;
                $prev[$neighbor] = $node;
                $pq[] = [$neighbor, $newDist];
            }
        }
    }

    return ['dist' => $dist, 'prev' => $prev];
}

// পাথ রিকনস্ট্রাক্ট করো
function reconstructPath($prev, $end) {
    $path = [];
    $current = $end;
    while ($current !== null) {
        array_unshift($path, $current);
        $current = $prev[$current];
    }
    return $path;
}

// উদাহরণ ওয়েটেড গ্রাফ
$graph = [
    'A' => [['B', 4], ['C', 2]],
    'B' => [['A', 4], ['D', 1]],
    'C' => [['A', 2], ['D', 3]],
    'D' => [['B', 1], ['C', 3], ['E', 5]],
    'E' => [['D', 5]],
];

$result = dijkstra($graph, 'A');
echo "A থেকে শর্টেস্ট দূরত্ব:\n";
foreach ($result['dist'] as $node => $dist) {
    $path = implode(" -> ", reconstructPath($result['prev'], $node));
    echo "  $node: দূরত্ব = $dist, পাথ = $path\n";
}
?>
```

### JavaScript বাস্তবায়ন - ডায়াক্সট্রা

```javascript
// ডায়াক্সট্রা অ্যালগরিদম
function dijkstra(graph, start) {
    const dist = {};  // দূরত্ব
    const prev = {};  // পূর্ববর্তী নোড
    const visited = new Set();

    // সব নোডের দূরত্ব অসীম
    for (const node of graph.keys()) {
        dist[node] = Infinity;
        prev[node] = null;
    }
    dist[start] = 0;

    // সিম্পল প্রায়োরিটি কিউ
    const pq = [[start, 0]]; // [নোড, দূরত্ব]

    while (pq.length > 0) {
        // সবচেয়ে কম দূরত্ব খোঁজো
        pq.sort((a, b) => a[1] - b[1]);
        const [node, d] = pq.shift();

        if (visited.has(node)) continue;
        visited.add(node);

        // প্রতিবেশী আপডেট
        for (const [neighbor, weight] of graph.get(node) || []) {
            const newDist = dist[node] + weight;
            if (newDist < dist[neighbor]) {
                dist[neighbor] = newDist;
                prev[neighbor] = node;
                pq.push([neighbor, newDist]);
            }
        }
    }

    return { dist, prev };
}

// পাথ বের করো
function reconstructPath(prev, end) {
    const path = [];
    let current = end;
    while (current !== null) {
        path.unshift(current);
        current = prev[current];
    }
    return path;
}

// উদাহরণ
const graph = new Map([
    ["A", [["B", 4], ["C", 2]]],
    ["B", [["A", 4], ["D", 1]]],
    ["C", [["A", 2], ["D", 3]]],
    ["D", [["B", 1], ["C", 3], ["E", 5]]],
    ["E", [["D", 5]]],
]);

const result = dijkstra(graph, "A");
console.log("A থেকে শর্টেস্ট দূরত্ব:");
for (const [node, dist] of Object.entries(result.dist)) {
    const path = reconstructPath(result.prev, node).join(" -> ");
    console.log(`  ${node}: দূরত্ব = ${dist}, পাথ = ${path}`);
}
```

### ৫.২ বেলম্যান-ফোর্ড অ্যালগরিদম (Bellman-Ford)

**কখন ব্যবহার:** নেগেটিভ ওজনের এজ থাকলে, নেগেটিভ সাইকেল ডিটেকশন।

```php
<?php
// বেলম্যান-ফোর্ড অ্যালগরিদম
function bellmanFord($vertices, $edges, $start) {
    $dist = array_fill_keys($vertices, PHP_INT_MAX);
    $dist[$start] = 0;

    // V-1 বার সব এজ রিল্যাক্স করো
    for ($i = 0; $i < count($vertices) - 1; $i++) {
        foreach ($edges as [$u, $v, $w]) {
            if ($dist[$u] !== PHP_INT_MAX && $dist[$u] + $w < $dist[$v]) {
                $dist[$v] = $dist[$u] + $w;
            }
        }
    }

    // নেগেটিভ সাইকেল চেক
    foreach ($edges as [$u, $v, $w]) {
        if ($dist[$u] !== PHP_INT_MAX && $dist[$u] + $w < $dist[$v]) {
            echo "⚠️ নেগেটিভ সাইকেল পাওয়া গেছে!\n";
            return null;
        }
    }

    return $dist;
}

$vertices = ['A', 'B', 'C', 'D'];
$edges = [
    ['A', 'B', 4],
    ['A', 'C', 2],
    ['B', 'D', 3],
    ['C', 'D', 1],
    ['C', 'B', -1], // নেগেটিভ ওজন
];

$dist = bellmanFord($vertices, $edges, 'A');
echo "বেলম্যান-ফোর্ড দূরত্ব:\n";
foreach ($dist as $node => $d) {
    echo "  $node: $d\n";
}
?>
```

### ৫.৩ ফ্লয়েড-ওয়ারশাল অ্যালগরিদম (Floyd-Warshall)

**কখন ব্যবহার:** সব জোড়া নোডের মধ্যে শর্টেস্ট পাথ।

```javascript
// ফ্লয়েড-ওয়ারশাল অ্যালগরিদম
function floydWarshall(n, edges) {
    // দূরত্ব ম্যাট্রিক্স তৈরি
    const dist = Array.from({ length: n }, (_, i) =>
        Array.from({ length: n }, (_, j) => (i === j ? 0 : Infinity))
    );

    // এজের ওজন বসাও
    for (const [u, v, w] of edges) {
        dist[u][v] = w;
    }

    // মূল অ্যালগরিদম - তিনটি নেস্টেড লুপ
    for (let k = 0; k < n; k++) {         // মধ্যবর্তী নোড
        for (let i = 0; i < n; i++) {     // উৎস
            for (let j = 0; j < n; j++) { // গন্তব্য
                if (dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }

    return dist;
}

// উদাহরণ
const edges = [
    [0, 1, 4],
    [0, 2, 2],
    [1, 3, 3],
    [2, 1, 1],
    [2, 3, 5],
];

const dist = floydWarshall(4, edges);
console.log("সব জোড়ার শর্টেস্ট দূরত্ব:");
dist.forEach((row, i) => {
    row.forEach((d, j) => {
        if (d !== Infinity) console.log(`  ${i} -> ${j}: ${d}`);
    });
});
```

### শর্টেস্ট পাথ অ্যালগরিদম তুলনা

| অ্যালগরিদম | টাইম | স্পেস | নেগেটিভ ওজন | সব জোড়া | ব্যবহার |
|------------|-------|-------|-------------|---------|---------|
| ডায়াক্সট্রা | O((V+E)logV) | O(V) | ❌ | ❌ | একক উৎস, নন-নেগেটিভ |
| বেলম্যান-ফোর্ড | O(V·E) | O(V) | ✅ | ❌ | একক উৎস, নেগেটিভ ওজন |
| ফ্লয়েড-ওয়ারশাল | O(V³) | O(V²) | ✅ | ✅ | সব জোড়ার শর্টেস্ট পাথ |

---

## ৬. টপোলজিক্যাল সর্ট

### ধারণা

**টপোলজিক্যাল সর্ট** শুধুমাত্র **DAG** (Directed Acyclic Graph) এ কাজ করে।
এটি নোডগুলোকে এমনভাবে সাজায় যাতে প্রতিটি এজ (u, v) এর জন্য u, v এর আগে আসে।

### ব্যবহারের ক্ষেত্র

- কোর্স প্রিরিকুইজিট নির্ধারণ
- বিল্ড সিস্টেম (ডিপেন্ডেন্সি রেজলভ)
- টাস্ক শিডিউলিং

### ASCII ট্রেস

```
DAG:
    A ----> B ----> D
    |       ^       ^
    |       |       |
    +-----> C ------+

ইন-ডিগ্রি: A=0, B=1, C=1, D=2

কান'স অ্যালগরিদম ট্রেস:
ধাপ ১: কিউ = [A] (ইন-ডিগ্রি ০)
        প্রসেস A -> B, C এর ইন-ডিগ্রি কমাও

ধাপ ২: কিউ = [B, C]  (B=0, C=0)
        প্রসেস B -> D এর ইন-ডিগ্রি কমাও

ধাপ ৩: কিউ = [C]
        প্রসেস C -> D এর ইন-ডিগ্রি কমাও

ধাপ ৪: কিউ = [D]  (D=0)
        প্রসেস D

ফলাফল: A -> B -> C -> D (বা A -> C -> B -> D)
```

### ৬.১ কান'স অ্যালগরিদম (BFS ভিত্তিক)

```php
<?php
// কান'স অ্যালগরিদম - BFS ভিত্তিক টপোলজিক্যাল সর্ট
function kahnsTopologicalSort($graph) {
    // ইন-ডিগ্রি গণনা
    $inDegree = [];
    foreach (array_keys($graph) as $node) {
        $inDegree[$node] = 0;
    }
    foreach ($graph as $node => $neighbors) {
        foreach ($neighbors as $neighbor) {
            $inDegree[$neighbor] = ($inDegree[$neighbor] ?? 0) + 1;
        }
    }

    // ইন-ডিগ্রি ০ এমন নোড কিউতে ঢোকাও
    $queue = [];
    foreach ($inDegree as $node => $deg) {
        if ($deg === 0) $queue[] = $node;
    }

    $result = [];
    while (!empty($queue)) {
        $node = array_shift($queue);
        $result[] = $node;

        foreach ($graph[$node] as $neighbor) {
            $inDegree[$neighbor]--;
            if ($inDegree[$neighbor] === 0) {
                $queue[] = $neighbor;
            }
        }
    }

    // সাইকেল চেক
    if (count($result) !== count($graph)) {
        echo "⚠️ সাইকেল আছে! টপোলজিক্যাল সর্ট সম্ভব নয়।\n";
        return null;
    }

    return $result;
}

$graph = [
    'A' => ['B', 'C'],
    'B' => ['D'],
    'C' => ['B', 'D'],
    'D' => [],
];

$sorted = kahnsTopologicalSort($graph);
echo "কান'স টপোলজিক্যাল সর্ট: " . implode(" -> ", $sorted) . "\n";
?>
```

### ৬.২ DFS ভিত্তিক টপোলজিক্যাল সর্ট

```javascript
// DFS ভিত্তিক টপোলজিক্যাল সর্ট
function topologicalSortDFS(graph) {
    const visited = new Set();
    const stack = []; // ফলাফল স্ট্যাক

    function dfs(node) {
        visited.add(node);
        for (const neighbor of graph.get(node) || []) {
            if (!visited.has(neighbor)) {
                dfs(neighbor);
            }
        }
        stack.push(node); // পোস্ট-অর্ডারে যোগ
    }

    // সব নোডে DFS চালাও
    for (const node of graph.keys()) {
        if (!visited.has(node)) {
            dfs(node);
        }
    }

    return stack.reverse(); // উল্টো করলে টপোলজিক্যাল অর্ডার পাওয়া যায়
}

// কান'স অ্যালগরিদম (JavaScript)
function kahnsSort(graph) {
    const inDegree = new Map();
    for (const node of graph.keys()) inDegree.set(node, 0);

    for (const [_, neighbors] of graph) {
        for (const n of neighbors) {
            inDegree.set(n, (inDegree.get(n) || 0) + 1);
        }
    }

    const queue = [];
    for (const [node, deg] of inDegree) {
        if (deg === 0) queue.push(node);
    }

    const result = [];
    while (queue.length > 0) {
        const node = queue.shift();
        result.push(node);
        for (const neighbor of graph.get(node) || []) {
            inDegree.set(neighbor, inDegree.get(neighbor) - 1);
            if (inDegree.get(neighbor) === 0) queue.push(neighbor);
        }
    }

    return result.length === graph.size ? result : null;
}

const graph = new Map([
    ["A", ["B", "C"]],
    ["B", ["D"]],
    ["C", ["B", "D"]],
    ["D", []],
]);

console.log("DFS টপোসর্ট:", topologicalSortDFS(graph).join(" -> "));
console.log("কান'স টপোসর্ট:", kahnsSort(graph).join(" -> "));
```

### টপোলজিক্যাল সর্ট জটিলতা

| অ্যালগরিদম | টাইম | স্পেস |
|------------|-------|-------|
| কান'স (BFS) | O(V + E) | O(V) |
| DFS ভিত্তিক | O(V + E) | O(V) |

---

## ৭. মিনিমাম স্প্যানিং ট্রি (MST)

### ধারণা

MST হলো একটি সাবগ্রাফ যা সব নোড সংযুক্ত রাখে এবং মোট এজ ওজন সর্বনিম্ন।

### ASCII ট্রেস

```
মূল গ্রাফ:
    A --4-- B
    |  \    |
    2   3   1
    |    \  |
    C --5-- D

ক্রাস্কাল'স MST তৈরি (ওজন অনুসারে এজ সাজানো):
১. B-D (ওজন 1) ✅ যোগ
২. A-C (ওজন 2) ✅ যোগ
৩. A-D (ওজন 3) ✅ যোগ
৪. A-B (ওজন 4) ❌ সাইকেল তৈরি হবে
৫. C-D (ওজন 5) ❌ সাইকেল তৈরি হবে

MST:
    A       B
    |       |
    2       1
    |       |
    C   3   D
      A---D

MST মোট ওজন: 1 + 2 + 3 = 6
```

### ৭.১ ক্রাস্কাল'স অ্যালগরিদম (Kruskal's)

**কৌশল:** এজগুলো ওজন অনুসারে সাজাও, তারপর ছোট থেকে বড় ক্রমে যোগ করো (সাইকেল না হলে)।

```php
<?php
// ইউনিয়ন-ফাইন্ড (ডিসজয়েন্ট সেট)
class UnionFind {
    private $parent = [];
    private $rank = [];

    public function makeSet($x) {
        $this->parent[$x] = $x;
        $this->rank[$x] = 0;
    }

    // পাথ কম্প্রেশনসহ ফাইন্ড
    public function find($x) {
        if ($this->parent[$x] !== $x) {
            $this->parent[$x] = $this->find($this->parent[$x]);
        }
        return $this->parent[$x];
    }

    // র্যাংক দিয়ে ইউনিয়ন
    public function union($x, $y) {
        $rootX = $this->find($x);
        $rootY = $this->find($y);

        if ($rootX === $rootY) return false; // একই সেটে আছে

        if ($this->rank[$rootX] < $this->rank[$rootY]) {
            $this->parent[$rootX] = $rootY;
        } elseif ($this->rank[$rootX] > $this->rank[$rootY]) {
            $this->parent[$rootY] = $rootX;
        } else {
            $this->parent[$rootY] = $rootX;
            $this->rank[$rootX]++;
        }
        return true;
    }
}

// ক্রাস্কাল'স অ্যালগরিদম
function kruskal($vertices, $edges) {
    // এজ ওজন অনুসারে সাজাও
    usort($edges, fn($a, $b) => $a[2] - $b[2]);

    $uf = new UnionFind();
    foreach ($vertices as $v) $uf->makeSet($v);

    $mst = [];
    $totalWeight = 0;

    foreach ($edges as [$u, $v, $w]) {
        // সাইকেল না হলে যোগ করো
        if ($uf->union($u, $v)) {
            $mst[] = [$u, $v, $w];
            $totalWeight += $w;
        }
    }

    return ['mst' => $mst, 'weight' => $totalWeight];
}

$vertices = ['A', 'B', 'C', 'D'];
$edges = [
    ['A', 'B', 4],
    ['A', 'C', 2],
    ['A', 'D', 3],
    ['B', 'D', 1],
    ['C', 'D', 5],
];

$result = kruskal($vertices, $edges);
echo "ক্রাস্কাল'স MST:\n";
foreach ($result['mst'] as [$u, $v, $w]) {
    echo "  $u -- $v (ওজন: $w)\n";
}
echo "মোট ওজন: {$result['weight']}\n";
?>
```

### ৭.২ প্রিম'স অ্যালগরিদম (Prim's)

**কৌশল:** একটি নোড থেকে শুরু করে সবচেয়ে কম ওজনের এজ দিয়ে MST বাড়াও।

```javascript
// প্রিম'স অ্যালগরিদম
function prim(graph, start) {
    const mst = [];          // MST এজ
    const visited = new Set();
    let totalWeight = 0;

    // [ওজন, উৎস, গন্তব্য] হিসেবে প্রায়োরিটি কিউ
    let pq = [[0, null, start]];

    while (pq.length > 0 && visited.size < graph.size) {
        // সবচেয়ে কম ওজন বের করো
        pq.sort((a, b) => a[0] - b[0]);
        const [weight, from, node] = pq.shift();

        if (visited.has(node)) continue;
        visited.add(node);

        if (from !== null) {
            mst.push([from, node, weight]);
            totalWeight += weight;
        }

        // প্রতিবেশীদের কিউতে যোগ
        for (const [neighbor, w] of graph.get(node) || []) {
            if (!visited.has(neighbor)) {
                pq.push([w, node, neighbor]);
            }
        }
    }

    return { mst, totalWeight };
}

// উদাহরণ
const graph = new Map([
    ["A", [["B", 4], ["C", 2], ["D", 3]]],
    ["B", [["A", 4], ["D", 1]]],
    ["C", [["A", 2], ["D", 5]]],
    ["D", [["A", 3], ["B", 1], ["C", 5]]],
]);

const result = prim(graph, "A");
console.log("প্রিম'স MST:");
result.mst.forEach(([u, v, w]) => console.log(`  ${u} -- ${v} (ওজন: ${w})`));
console.log("মোট ওজন:", result.totalWeight);
```

### MST অ্যালগরিদম তুলনা

| বৈশিষ্ট্য | ক্রাস্কাল | প্রিম |
|-----------|----------|-------|
| কৌশল | এজ-ভিত্তিক (গ্রিডি) | নোড-ভিত্তিক (গ্রিডি) |
| ডেটা স্ট্রাকচার | ইউনিয়ন-ফাইন্ড | প্রায়োরিটি কিউ |
| টাইম | O(E log E) | O((V+E) log V) |
| বিরল গ্রাফে ভালো | ✅ | ❌ |
| ঘন গ্রাফে ভালো | ❌ | ✅ |

---

## ৮. অ্যাডভান্সড সমস্যা

### ৮.১ সাইকেল ডিটেকশন

#### আনডাইরেক্টেড গ্রাফে সাইকেল (DFS)

```php
<?php
// আনডাইরেক্টেড গ্রাফে সাইকেল ডিটেকশন
function hasCycleUndirected($graph) {
    $visited = [];

    function dfs($node, $parent, $graph, &$visited) {
        $visited[$node] = true;

        foreach ($graph[$node] as $neighbor) {
            if (!isset($visited[$neighbor])) {
                if (dfs($neighbor, $node, $graph, $visited)) {
                    return true; // সাইকেল পাওয়া গেছে
                }
            } elseif ($neighbor !== $parent) {
                return true; // ব্যাক এজ = সাইকেল!
            }
        }
        return false;
    }

    foreach (array_keys($graph) as $node) {
        if (!isset($visited[$node])) {
            if (dfs($node, -1, $graph, $visited)) {
                return true;
            }
        }
    }
    return false;
}

$graphWithCycle = [
    0 => [1, 2],
    1 => [0, 2],
    2 => [0, 1],
];

$graphNoCycle = [
    0 => [1],
    1 => [0, 2],
    2 => [1],
];

echo "সাইকেলযুক্ত: " . (hasCycleUndirected($graphWithCycle) ? "হ্যাঁ" : "না") . "\n";
echo "সাইকেলবিহীন: " . (hasCycleUndirected($graphNoCycle) ? "হ্যাঁ" : "না") . "\n";
?>
```

#### ডাইরেক্টেড গ্রাফে সাইকেল (তিন রঙের পদ্ধতি)

```javascript
// ডাইরেক্টেড গ্রাফে সাইকেল ডিটেকশন
// তিন রঙ: WHITE=অনভিজিটেড, GRAY=প্রসেসিং, BLACK=সম্পন্ন
function hasCycleDirected(graph) {
    const WHITE = 0, GRAY = 1, BLACK = 2;
    const color = new Map();

    for (const node of graph.keys()) color.set(node, WHITE);

    function dfs(node) {
        color.set(node, GRAY); // প্রসেসিং শুরু

        for (const neighbor of graph.get(node) || []) {
            if (color.get(neighbor) === GRAY) {
                return true; // ব্যাক এজ = সাইকেল!
            }
            if (color.get(neighbor) === WHITE && dfs(neighbor)) {
                return true;
            }
        }

        color.set(node, BLACK); // প্রসেসিং শেষ
        return false;
    }

    for (const node of graph.keys()) {
        if (color.get(node) === WHITE && dfs(node)) {
            return true;
        }
    }
    return false;
}

// সাইকেলযুক্ত ডাইরেক্টেড গ্রাফ
const graphCycle = new Map([
    ["A", ["B"]],
    ["B", ["C"]],
    ["C", ["A"]], // সাইকেল: A -> B -> C -> A
]);

// সাইকেলবিহীন DAG
const dag = new Map([
    ["A", ["B", "C"]],
    ["B", ["D"]],
    ["C", ["D"]],
    ["D", []],
]);

console.log("সাইকেলযুক্ত:", hasCycleDirected(graphCycle) ? "হ্যাঁ" : "না");
console.log("DAG:", hasCycleDirected(dag) ? "হ্যাঁ" : "না");
```

### ৮.২ কানেক্টেড কম্পোনেন্ট

```php
<?php
// কানেক্টেড কম্পোনেন্ট খোঁজা
function findConnectedComponents($graph) {
    $visited = [];
    $components = [];

    foreach (array_keys($graph) as $node) {
        if (!isset($visited[$node])) {
            // নতুন কম্পোনেন্ট - BFS দিয়ে খোঁজো
            $component = [];
            $queue = [$node];
            $visited[$node] = true;

            while (!empty($queue)) {
                $curr = array_shift($queue);
                $component[] = $curr;

                foreach ($graph[$curr] as $neighbor) {
                    if (!isset($visited[$neighbor])) {
                        $visited[$neighbor] = true;
                        $queue[] = $neighbor;
                    }
                }
            }
            $components[] = $component;
        }
    }
    return $components;
}

// ডিসকানেক্টেড গ্রাফ
$graph = [
    0 => [1],
    1 => [0],
    2 => [3],
    3 => [2],
    4 => [],
];

$components = findConnectedComponents($graph);
echo "কানেক্টেড কম্পোনেন্ট সংখ্যা: " . count($components) . "\n";
foreach ($components as $i => $comp) {
    echo "  কম্পোনেন্ট " . ($i + 1) . ": [" . implode(", ", $comp) . "]\n";
}
// ফলাফল: ৩টি কম্পোনেন্ট - [0,1], [2,3], [4]
?>
```

### ৮.৩ বাইপার্টাইট গ্রাফ চেক

একটি গ্রাফ বাইপার্টাইট যদি নোডগুলোকে দুটি দলে ভাগ করা যায় যেন একই দলের দুটি নোডের মধ্যে কোনো এজ না থাকে।

```javascript
// বাইপার্টাইট চেক (BFS দিয়ে ২-কালারিং)
function isBipartite(graph) {
    const color = new Map(); // রঙ সংরক্ষণ (0 বা 1)

    for (const node of graph.keys()) {
        if (color.has(node)) continue; // ইতিমধ্যে রঙ করা

        // BFS দিয়ে রঙ করো
        const queue = [node];
        color.set(node, 0);

        while (queue.length > 0) {
            const curr = queue.shift();
            const currColor = color.get(curr);

            for (const neighbor of graph.get(curr) || []) {
                if (!color.has(neighbor)) {
                    // বিপরীত রঙ দাও
                    color.set(neighbor, 1 - currColor);
                    queue.push(neighbor);
                } else if (color.get(neighbor) === currColor) {
                    // একই রঙ = বাইপার্টাইট নয়!
                    return false;
                }
            }
        }
    }
    return true;
}

// বাইপার্টাইট গ্রাফ (দুটি দলে ভাগ করা যায়)
const bipartite = new Map([
    [1, [2, 4]],
    [2, [1, 3]],
    [3, [2, 4]],
    [4, [3, 1]],
]);

// নন-বাইপার্টাইট (বিজোড় সাইকেল আছে)
const nonBipartite = new Map([
    [1, [2, 3]],
    [2, [1, 3]],
    [3, [1, 2]],
]);

console.log("বাইপার্টাইট:", isBipartite(bipartite) ? "হ্যাঁ" : "না");
console.log("ত্রিভুজ:", isBipartite(nonBipartite) ? "হ্যাঁ" : "না");
```

### অ্যাডভান্সড সমস্যা জটিলতা

| সমস্যা | টাইম | স্পেস | অ্যালগরিদম |
|---------|-------|-------|-----------|
| সাইকেল ডিটেকশন | O(V + E) | O(V) | DFS |
| কানেক্টেড কম্পোনেন্ট | O(V + E) | O(V) | BFS/DFS |
| বাইপার্টাইট চেক | O(V + E) | O(V) | BFS (2-কালার) |

---

## ৯. সম্পূর্ণ প্রজেক্ট: রুট ফাইন্ডার

### প্রজেক্ট বিবরণ

একটি শহরের মানচিত্রে ডায়াক্সট্রা দিয়ে শর্টেস্ট পাথ এবং DFS দিয়ে সব সম্ভব পাথ বের করার সিস্টেম।

```
শহরের মানচিত্র:
    ঢাকা --10-- চট্টগ্রাম
    |    \        |
    5     8       3
    |      \      |
    সিলেট --4-- রাজশাহী --7-- খুলনা
```

### PHP - সম্পূর্ণ রুট ফাইন্ডার

```php
<?php
// ===== রুট ফাইন্ডার - শহরের মানচিত্র =====

class RouteFinder {
    private $graph = [];  // শহরের সংযোগ
    private $cities = []; // শহরের তালিকা

    // শহর যোগ করো
    public function addCity($city) {
        if (!isset($this->graph[$city])) {
            $this->graph[$city] = [];
            $this->cities[] = $city;
        }
    }

    // রাস্তা যোগ করো (দূরত্বসহ)
    public function addRoad($city1, $city2, $distance) {
        $this->addCity($city1);
        $this->addCity($city2);
        $this->graph[$city1][] = [$city2, $distance];
        $this->graph[$city2][] = [$city1, $distance];
    }

    // ডায়াক্সট্রা দিয়ে শর্টেস্ট পাথ
    public function shortestPath($start, $end) {
        $dist = [];
        $prev = [];
        $visited = [];

        foreach ($this->cities as $city) {
            $dist[$city] = PHP_INT_MAX;
            $prev[$city] = null;
        }
        $dist[$start] = 0;

        $pq = [[$start, 0]];

        while (!empty($pq)) {
            usort($pq, fn($a, $b) => $a[1] - $b[1]);
            [$city, $d] = array_shift($pq);

            if (isset($visited[$city])) continue;
            $visited[$city] = true;

            if ($city === $end) break; // গন্তব্যে পৌঁছেছি

            foreach ($this->graph[$city] as [$neighbor, $weight]) {
                $newDist = $dist[$city] + $weight;
                if ($newDist < $dist[$neighbor]) {
                    $dist[$neighbor] = $newDist;
                    $prev[$neighbor] = $city;
                    $pq[] = [$neighbor, $newDist];
                }
            }
        }

        // পাথ তৈরি
        $path = [];
        $current = $end;
        while ($current !== null) {
            array_unshift($path, $current);
            $current = $prev[$current];
        }

        return [
            'distance' => $dist[$end],
            'path' => $path,
        ];
    }

    // DFS দিয়ে সব পাথ খোঁজো
    public function allPaths($start, $end) {
        $allPaths = [];
        $this->dfsAllPaths($start, $end, [$start => true], [$start], 0, $allPaths);

        // দূরত্ব অনুসারে সাজাও
        usort($allPaths, fn($a, $b) => $a['distance'] - $b['distance']);
        return $allPaths;
    }

    private function dfsAllPaths($current, $end, $visited, $path, $dist, &$allPaths) {
        if ($current === $end) {
            $allPaths[] = ['path' => $path, 'distance' => $dist];
            return;
        }

        foreach ($this->graph[$current] as [$neighbor, $weight]) {
            if (!isset($visited[$neighbor])) {
                $visited[$neighbor] = true;
                $path[] = $neighbor;

                $this->dfsAllPaths($neighbor, $end, $visited, $path, $dist + $weight, $allPaths);

                // ব্যাকট্র্যাক
                array_pop($path);
                unset($visited[$neighbor]);
            }
        }
    }

    // মানচিত্র দেখাও
    public function displayMap() {
        echo "🗺️  শহরের মানচিত্র:\n";
        echo str_repeat("-", 40) . "\n";
        $shown = [];
        foreach ($this->graph as $city => $roads) {
            foreach ($roads as [$neighbor, $dist]) {
                $key = min($city, $neighbor) . "-" . max($city, $neighbor);
                if (!isset($shown[$key])) {
                    echo "  $city <--$dist কি.মি.--> $neighbor\n";
                    $shown[$key] = true;
                }
            }
        }
        echo str_repeat("-", 40) . "\n";
    }
}

// ===== ব্যবহার =====
$finder = new RouteFinder();

// শহর ও রাস্তা যোগ
$finder->addRoad("ঢাকা", "চট্টগ্রাম", 10);
$finder->addRoad("ঢাকা", "সিলেট", 5);
$finder->addRoad("ঢাকা", "রাজশাহী", 8);
$finder->addRoad("চট্টগ্রাম", "রাজশাহী", 3);
$finder->addRoad("সিলেট", "রাজশাহী", 4);
$finder->addRoad("রাজশাহী", "খুলনা", 7);

$finder->displayMap();

// শর্টেস্ট পাথ
echo "\n📍 ঢাকা থেকে খুলনা - শর্টেস্ট পাথ:\n";
$result = $finder->shortestPath("ঢাকা", "খুলনা");
echo "  পাথ: " . implode(" -> ", $result['path']) . "\n";
echo "  দূরত্ব: {$result['distance']} কি.মি.\n";

// সব পাথ
echo "\n📍 ঢাকা থেকে খুলনা - সব সম্ভব পাথ:\n";
$allPaths = $finder->allPaths("ঢাকা", "খুলনা");
foreach ($allPaths as $i => $p) {
    echo "  " . ($i + 1) . ". " . implode(" -> ", $p['path']);
    echo " (দূরত্ব: {$p['distance']} কি.মি.)\n";
}
?>
```

### JavaScript - সম্পূর্ণ রুট ফাইন্ডার

```javascript
// ===== রুট ফাইন্ডার - শহরের মানচিত্র =====

class RouteFinder {
    constructor() {
        this.graph = new Map(); // শহরের সংযোগ
    }

    // শহর যোগ করো
    addCity(city) {
        if (!this.graph.has(city)) {
            this.graph.set(city, []);
        }
    }

    // রাস্তা যোগ করো
    addRoad(city1, city2, distance) {
        this.addCity(city1);
        this.addCity(city2);
        this.graph.get(city1).push([city2, distance]);
        this.graph.get(city2).push([city1, distance]);
    }

    // ডায়াক্সট্রা দিয়ে শর্টেস্ট পাথ
    shortestPath(start, end) {
        const dist = new Map();
        const prev = new Map();
        const visited = new Set();

        for (const city of this.graph.keys()) {
            dist.set(city, Infinity);
            prev.set(city, null);
        }
        dist.set(start, 0);

        const pq = [[start, 0]];

        while (pq.length > 0) {
            pq.sort((a, b) => a[1] - b[1]);
            const [city, d] = pq.shift();

            if (visited.has(city)) continue;
            visited.add(city);

            if (city === end) break;

            for (const [neighbor, weight] of this.graph.get(city)) {
                const newDist = dist.get(city) + weight;
                if (newDist < dist.get(neighbor)) {
                    dist.set(neighbor, newDist);
                    prev.set(neighbor, city);
                    pq.push([neighbor, newDist]);
                }
            }
        }

        // পাথ তৈরি
        const path = [];
        let current = end;
        while (current !== null) {
            path.unshift(current);
            current = prev.get(current);
        }

        return { distance: dist.get(end), path };
    }

    // DFS দিয়ে সব পাথ
    allPaths(start, end) {
        const allPaths = [];
        this._dfsAllPaths(start, end, new Set([start]), [start], 0, allPaths);
        allPaths.sort((a, b) => a.distance - b.distance);
        return allPaths;
    }

    _dfsAllPaths(current, end, visited, path, dist, allPaths) {
        if (current === end) {
            allPaths.push({ path: [...path], distance: dist });
            return;
        }

        for (const [neighbor, weight] of this.graph.get(current)) {
            if (!visited.has(neighbor)) {
                visited.add(neighbor);
                path.push(neighbor);

                this._dfsAllPaths(neighbor, end, visited, path, dist + weight, allPaths);

                // ব্যাকট্র্যাক
                path.pop();
                visited.delete(neighbor);
            }
        }
    }

    // মানচিত্র দেখাও
    displayMap() {
        console.log("🗺️  শহরের মানচিত্র:");
        console.log("-".repeat(40));
        const shown = new Set();
        for (const [city, roads] of this.graph) {
            for (const [neighbor, dist] of roads) {
                const key = [city, neighbor].sort().join("-");
                if (!shown.has(key)) {
                    console.log(`  ${city} <--${dist} কি.মি.--> ${neighbor}`);
                    shown.add(key);
                }
            }
        }
        console.log("-".repeat(40));
    }
}

// ===== ব্যবহার =====
const finder = new RouteFinder();

finder.addRoad("ঢাকা", "চট্টগ্রাম", 10);
finder.addRoad("ঢাকা", "সিলেট", 5);
finder.addRoad("ঢাকা", "রাজশাহী", 8);
finder.addRoad("চট্টগ্রাম", "রাজশাহী", 3);
finder.addRoad("সিলেট", "রাজশাহী", 4);
finder.addRoad("রাজশাহী", "খুলনা", 7);

finder.displayMap();

// শর্টেস্ট পাথ
console.log("\n📍 ঢাকা থেকে খুলনা - শর্টেস্ট পাথ:");
const result = finder.shortestPath("ঢাকা", "খুলনা");
console.log(`  পাথ: ${result.path.join(" -> ")}`);
console.log(`  দূরত্ব: ${result.distance} কি.মি.`);

// সব পাথ
console.log("\n📍 ঢাকা থেকে খুলনা - সব সম্ভব পাথ:");
const allPaths = finder.allPaths("ঢাকা", "খুলনা");
allPaths.forEach((p, i) => {
    console.log(`  ${i + 1}. ${p.path.join(" -> ")} (দূরত্ব: ${p.distance} কি.মি.)`);
});
```

---

## চিটশিট ও ইন্টারভিউ টিপস

### 📋 গ্রাফ অ্যালগরিদম চিটশিট

```
╔══════════════════════════════════════════════════════════════════╗
║                    গ্রাফ চিটশিট                                ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  রিপ্রেজেন্টেশন:                                               ║
║  • ম্যাট্রিক্স → ঘন গ্রাফ, O(1) এজ লুকআপ                    ║
║  • লিস্ট → বিরল গ্রাফ, O(V+E) স্পেস ✅                       ║
║  • এজ লিস্ট → ক্রাস্কাল, O(E) স্পেস                          ║
║                                                                  ║
║  ট্রাভার্সাল:                                                   ║
║  • BFS → কিউ, স্তরভিত্তিক, শর্টেস্ট পাথ (আনওয়েটেড)         ║
║  • DFS → স্ট্যাক/রিকার্শন, সাইকেল, টপোসর্ট                  ║
║                                                                  ║
║  শর্টেস্ট পাথ:                                                  ║
║  • ডায়াক্সট্রা → নন-নেগেটিভ ওজন, O((V+E)logV)               ║
║  • বেলম্যান-ফোর্ড → নেগেটিভ ওজন, O(V·E)                      ║
║  • ফ্লয়েড-ওয়ারশাল → সব জোড়া, O(V³)                          ║
║                                                                  ║
║  MST:                                                            ║
║  • ক্রাস্কাল → এজ সর্ট + ইউনিয়ন-ফাইন্ড, O(ElogE)            ║
║  • প্রিম → প্রায়োরিটি কিউ, O((V+E)logV)                      ║
║                                                                  ║
║  টপোলজিক্যাল সর্ট:                                             ║
║  • কান'স → BFS + ইন-ডিগ্রি, O(V+E)                            ║
║  • DFS → পোস্ট-অর্ডার রিভার্স, O(V+E)                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### 💡 ইন্টারভিউ টিপস

1. **সঠিক রিপ্রেজেন্টেশন বেছে নাও:**
   - বেশিরভাগ সময় অ্যাডজেসেন্সি লিস্ট ব্যবহার করো
   - ঘন গ্রাফে ম্যাট্রিক্স ভালো

2. **BFS বনাম DFS বোঝো:**
   - শর্টেস্ট পাথ (আনওয়েটেড) → BFS
   - পাথ খোঁজা, সাইকেল → DFS
   - কানেক্টেড কম্পোনেন্ট → যেকোনোটা

3. **এজ কেস মনে রাখো:**
   - ডিসকানেক্টেড গ্রাফ
   - সেল্ফ-লুপ
   - প্যারালেল এজ
   - খালি গ্রাফ

4. **জটিলতা বিশ্লেষণ জানো:**
   - V = নোড সংখ্যা, E = এজ সংখ্যা
   - বেশিরভাগ গ্রাফ অ্যালগরিদম O(V + E)

5. **সাধারণ প্যাটার্ন:**
   - `visited` সেট সবসময় ব্যবহার করো
   - ডাইরেক্টেড vs আনডাইরেক্টেড পার্থক্য বোঝো
   - ওয়েটেড গ্রাফে BFS কাজ করবে না

6. **প্র্যাকটিস সমস্যা:**
   - LeetCode: Number of Islands (200)
   - LeetCode: Course Schedule (207)
   - LeetCode: Network Delay Time (743)
   - LeetCode: Clone Graph (133)

### 📊 সম্পূর্ণ Big O তুলনা

| অ্যালগরিদম | টাইম কমপ্লেক্সিটি | স্পেস কমপ্লেক্সিটি |
|------------|-------------------|---------------------|
| BFS | O(V + E) | O(V) |
| DFS | O(V + E) | O(V) |
| ডায়াক্সট্রা | O((V+E) log V) | O(V) |
| বেলম্যান-ফোর্ড | O(V · E) | O(V) |
| ফ্লয়েড-ওয়ারশাল | O(V³) | O(V²) |
| কান'স সর্ট | O(V + E) | O(V) |
| ক্রাস্কাল | O(E log E) | O(V) |
| প্রিম | O((V+E) log V) | O(V) |
| সাইকেল ডিটেকশন | O(V + E) | O(V) |
| বাইপার্টাইট চেক | O(V + E) | O(V) |

---

> 📖 **পরবর্তী বিষয়:** [হ্যাশ টেবিল (Hash Table)](./hash-table.md) - কী-ভ্যালু ডেটা স্ট্রাকচার শিখুন
