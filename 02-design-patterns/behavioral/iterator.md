# 🔄 Iterator প্যাটার্ন (Iterator Pattern)

## 📌 সংজ্ঞা

> **GoF Definition:** "Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation."

Iterator প্যাটার্ন হলো একটি **Behavioral Design Pattern** যা কোনো কালেকশনের অভ্যন্তরীণ গঠন (internal structure) প্রকাশ না করে তার উপাদানগুলোকে ক্রমানুসারে (sequentially) অ্যাক্সেস করার একটি সুনির্দিষ্ট উপায় প্রদান করে।

### মূল ধারণা

ধরুন আপনার কাছে একটি কালেকশন আছে — সেটা Array হতে পারে, Linked List হতে পারে, Tree হতে পারে, অথবা Graph। প্রতিটির অভ্যন্তরীণ গঠন সম্পূর্ণ আলাদা। কিন্তু আপনি চান একটি **সার্বজনীন পদ্ধতিতে** (uniform interface) এদের সব উপাদানে ঘুরে ঘুরে যেতে। Iterator প্যাটার্ন ঠিক এই কাজটিই করে — **traversal logic কে কালেকশন থেকে আলাদা করে** একটি স্বতন্ত্র অবজেক্টে রাখে।

### সমস্যা যা সমাধান করে

```
সমস্যা:
├── কালেকশনের বিভিন্ন ধরন (Array, Tree, Graph, HashMap)
├── প্রতিটিতে ভিন্ন ভিন্ন traversal যুক্তি
├── ক্লায়েন্ট কোডে কালেকশনের internal structure জানতে হয়
├── একাধিক traversal strategy (DFS, BFS, in-order) সাপোর্ট করতে কষ্ট
└── Single Responsibility Principle লঙ্ঘন হয়

সমাধান (Iterator):
├── একটি সার্বজনীন ইন্টারফেস (next, hasNext, current)
├── Traversal যুক্তি আলাদা অবজেক্টে
├── ক্লায়েন্ট কোড internal structure থেকে মুক্ত
├── একই কালেকশনে একাধিক Iterator সম্ভব
└── Open/Closed Principle মেনে চলে
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### টিভি রিমোট কন্ট্রোল 📺

টিভি রিমোটের **Next Channel** ও **Previous Channel** বোতাম চিন্তা করুন:

```
আপনি (Client)
    │
    ▼
┌─────────────┐     ┌──────────────────────────┐
│  রিমোট      │────▶│  চ্যানেল তালিকা           │
│  (Iterator)  │     │  (Collection)             │
│             │     │                           │
│  ▶ Next     │     │  BTV, NTV, ATN, RTV...    │
│  ◀ Prev     │     │  (Array? LinkedList?      │
│  ■ Current  │     │   কোনটি — আপনার জানার     │
│             │     │   দরকার নেই!)              │
└─────────────┘     └──────────────────────────┘
```

আপনি শুধু "Next" চাপেন — পরের চ্যানেলে যায়। চ্যানেলগুলো কীভাবে সংরক্ষিত (array, linked list, database), কীভাবে সাজানো (নম্বর অনুযায়ী, নাম অনুযায়ী) — এসব আপনার জানার দরকার নেই।

### বাংলাদেশ প্রসঙ্গ 🇧🇩

**দারাজ/ইভ্যালি প্রোডাক্ট লিস্টিং:** হাজার হাজার প্রোডাক্ট আছে। আপনি পেজে পেজে দেখেন (Page 1, Page 2...)। প্রতিটি পেজ একটি "iteration step" — আপনি জানেন না ডাটাবেজে কীভাবে প্রোডাক্টগুলো সাজানো, শুধু "পরবর্তী পেজ" চাপেন।

**বড় CSV ইম্পোর্ট:** ১০ লাখ সারির CSV ফাইল। একবারে মেমোরিতে লোড করলে সার্ভার ক্র্যাশ। Iterator দিয়ে একটি একটি সারি পড়ে প্রসেস করেন — মেমোরি ব্যবহার ন্যূনতম।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────┐         ┌─────────────────────────┐
│   <<interface>>      │         │    <<interface>>          │
│   IterableAggregate  │         │    Iterator               │
│─────────────────────│         │─────────────────────────│
│ + createIterator():  │────────▶│ + current(): mixed        │
│     Iterator         │         │ + key(): mixed            │
│                      │         │ + next(): void            │
└──────────┬──────────┘         │ + rewind(): void          │
           │                     │ + valid(): bool           │
           │                     └───────────┬──────────────┘
           │                                 │
           │ implements                      │ implements
           │                                 │
┌──────────▼──────────┐         ┌───────────▼──────────────┐
│  ConcreteCollection  │         │  ConcreteIterator          │
│─────────────────────│         │─────────────────────────  │
│ - items: array       │────────▶│ - collection: Collection   │
│─────────────────────│  has-a  │ - position: int            │
│ + createIterator()   │         │─────────────────────────  │
│ + getItems()         │         │ + current(): mixed         │
│ + addItem(item)      │         │ + key(): int               │
│ + count(): int       │         │ + next(): void             │
└─────────────────────┘         │ + rewind(): void           │
                                 │ + valid(): bool            │
                                 └────────────────────────────┘

সম্পর্ক:
  ──────▶  Association (uses / has-a)
  ───┬───  Inheritance / Implementation
      │
```

### JS ভার্সন (Iterable Protocol)

```
┌────────────────────────┐       ┌──────────────────────────┐
│  Iterable Object       │       │  Iterator Object          │
│────────────────────────│       │──────────────────────────│
│ [Symbol.iterator]():   │──────▶│ next(): {value, done}     │
│   Iterator             │       │ return?(): IteratorResult  │
│                        │       │ throw?(): IteratorResult   │
└────────────────────────┘       └──────────────────────────┘
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Iterator

#### PHP 8.3 — `Iterator` ও `IteratorAggregate` ইন্টারফেস

```php
<?php

declare(strict_types=1);

// --- ১ক. PHP Iterator ইন্টারফেস সরাসরি ইমপ্লিমেন্ট ---

final class ProductCollection implements \IteratorAggregate, \Countable
{
    /** @var Product[] */
    private array $products = [];

    public function addProduct(Product $product): void
    {
        $this->products[] = $product;
    }

    public function getProducts(): array
    {
        return $this->products;
    }

    public function count(): int
    {
        return count($this->products);
    }

    public function getIterator(): ProductIterator
    {
        return new ProductIterator($this);
    }
}

final readonly class Product
{
    public function __construct(
        public string $name,
        public float $price,
        public string $category,
    ) {}
}

final class ProductIterator implements \Iterator
{
    private int $position = 0;

    public function __construct(
        private readonly ProductCollection $collection,
    ) {}

    public function current(): Product
    {
        return $this->collection->getProducts()[$this->position];
    }

    public function key(): int
    {
        return $this->position;
    }

    public function next(): void
    {
        $this->position++;
    }

    public function rewind(): void
    {
        $this->position = 0;
    }

    public function valid(): bool
    {
        return isset($this->collection->getProducts()[$this->position]);
    }
}

// ব্যবহার:
$collection = new ProductCollection();
$collection->addProduct(new Product('লুঙ্গি', 450.0, 'পোশাক'));
$collection->addProduct(new Product('ইলিশ মাছ', 1200.0, 'খাবার'));
$collection->addProduct(new Product('জামদানি শাড়ি', 8500.0, 'পোশাক'));

// foreach সরাসরি কাজ করে কারণ IteratorAggregate ইমপ্লিমেন্ট করা
foreach ($collection as $index => $product) {
    echo "{$index}: {$product->name} — ৳{$product->price}\n";
}
```

#### JS ES2022+ — `Symbol.iterator` প্রোটোকল

```javascript
// --- ১খ. JS Iterable Protocol ---

class Product {
  #name;
  #price;
  #category;

  constructor(name, price, category) {
    this.#name = name;
    this.#price = price;
    this.#category = category;
  }

  get name() { return this.#name; }
  get price() { return this.#price; }
  get category() { return this.#category; }

  toString() {
    return `${this.#name} — ৳${this.#price}`;
  }
}

class ProductCollection {
  #products = [];

  addProduct(product) {
    this.#products.push(product);
    return this; // fluent interface
  }

  get length() {
    return this.#products.length;
  }

  // Symbol.iterator ইমপ্লিমেন্ট করলে for...of কাজ করবে
  [Symbol.iterator]() {
    let index = 0;
    const products = this.#products;

    return {
      next() {
        if (index < products.length) {
          return { value: products[index++], done: false };
        }
        return { value: undefined, done: true };
      },

      // optional: for...of থেকে break করলে cleanup
      return() {
        console.log('Iterator বন্ধ হলো (break/return)');
        return { value: undefined, done: true };
      }
    };
  }
}

// ব্যবহার:
const collection = new ProductCollection();
collection
  .addProduct(new Product('লুঙ্গি', 450, 'পোশাক'))
  .addProduct(new Product('ইলিশ মাছ', 1200, 'খাবার'))
  .addProduct(new Product('জামদানি শাড়ি', 8500, 'পোশাক'));

for (const product of collection) {
  console.log(product.toString());
}

// spread ও destructuring কাজ করে
const allProducts = [...collection];
const [first, ...rest] = collection;
```

---

### ২. Custom Collection with Filtered Iterator

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ক্যাটাগরি অনুযায়ী ফিল্টার করা Iterator
final class CategoryFilterIterator extends \FilterIterator
{
    public function __construct(
        \Iterator $iterator,
        private readonly string $category,
    ) {
        parent::__construct($iterator);
    }

    public function accept(): bool
    {
        /** @var Product $product */
        $product = $this->current();
        return $product->category === $this->category;
    }
}

// দাম অনুযায়ী সাজানো Iterator
final class PriceSortedIterator extends \IteratorIterator
{
    private array $sortedItems = [];
    private int $position = 0;

    public function __construct(\Traversable $iterator)
    {
        parent::__construct($iterator);
        $this->sortedItems = iterator_to_array($iterator);
        usort($this->sortedItems, fn(Product $a, Product $b) => $a->price <=> $b->price);
    }

    public function current(): Product
    {
        return $this->sortedItems[$this->position];
    }

    public function key(): int
    {
        return $this->position;
    }

    public function next(): void
    {
        $this->position++;
    }

    public function rewind(): void
    {
        $this->position = 0;
    }

    public function valid(): bool
    {
        return isset($this->sortedItems[$this->position]);
    }
}

// ব্যবহার:
$collection = new ProductCollection();
$collection->addProduct(new Product('লুঙ্গি', 450.0, 'পোশাক'));
$collection->addProduct(new Product('ইলিশ মাছ', 1200.0, 'খাবার'));
$collection->addProduct(new Product('জামদানি শাড়ি', 8500.0, 'পোশাক'));
$collection->addProduct(new Product('পাঞ্জাবি', 1800.0, 'পোশাক'));
$collection->addProduct(new Product('চিংড়ি', 900.0, 'খাবার'));

// শুধু পোশাক ক্যাটাগরির প্রোডাক্ট
$clothingIterator = new CategoryFilterIterator(
    $collection->getIterator(),
    'পোশাক'
);

echo "=== পোশাক ===\n";
foreach ($clothingIterator as $product) {
    echo "  {$product->name}: ৳{$product->price}\n";
}

// দাম অনুযায়ী সাজানো
$sortedIterator = new PriceSortedIterator($collection->getIterator());

echo "\n=== দাম অনুযায়ী ===\n";
foreach ($sortedIterator as $product) {
    echo "  {$product->name}: ৳{$product->price}\n";
}
```

#### JS ES2022+

```javascript
class FilteredIterator {
  #source;
  #predicate;

  constructor(source, predicate) {
    this.#source = source;
    this.#predicate = predicate;
  }

  *[Symbol.iterator]() {
    for (const item of this.#source) {
      if (this.#predicate(item)) {
        yield item;
      }
    }
  }
}

class SortedIterator {
  #source;
  #comparator;

  constructor(source, comparator) {
    this.#source = source;
    this.#comparator = comparator;
  }

  *[Symbol.iterator]() {
    const items = [...this.#source].sort(this.#comparator);
    yield* items;
  }
}

// ব্যবহার:
const clothingOnly = new FilteredIterator(
  collection,
  (p) => p.category === 'পোশাক'
);

for (const product of clothingOnly) {
  console.log(`পোশাক: ${product}`);
}

const byPrice = new SortedIterator(
  collection,
  (a, b) => a.price - b.price
);

for (const product of byPrice) {
  console.log(`সাজানো: ${product}`);
}
```

---

### ৩. PHP Generators (`yield`) — Lazy Iterator

Generator হলো PHP-তে Iterator তৈরির সবচেয়ে সংক্ষিপ্ত ও মেমোরি-সাশ্রয়ী পদ্ধতি। `yield` কীওয়ার্ড ব্যবহার করলে ফাংশন একটি `Generator` অবজেক্ট রিটার্ন করে যা `Iterator` ইন্টারফেস ইমপ্লিমেন্ট করে।

```php
<?php

declare(strict_types=1);

// --- ৩ক. বেসিক Generator ---
function rangeGenerator(int $start, int $end, int $step = 1): Generator
{
    for ($i = $start; $i <= $end; $i += $step) {
        yield $i; // প্রতিটি মান lazy ভাবে তৈরি হয়
    }
}

// ১ কোটি সংখ্যা — কিন্তু মেমোরিতে একটিমাত্র সংখ্যা থাকে!
foreach (rangeGenerator(1, 10_000_000) as $number) {
    if ($number > 5) break;
    echo $number . "\n";
}

// --- ৩খ. বড় CSV ফাইল পড়া (বাংলাদেশ ব্যবসা: বড় CSV ইম্পোর্ট) ---
function readCsvLines(string $filePath): Generator
{
    $handle = fopen($filePath, 'r');
    if ($handle === false) {
        throw new RuntimeException("ফাইল খুলতে পারলাম না: {$filePath}");
    }

    try {
        $header = fgetcsv($handle);
        while (($row = fgetcsv($handle)) !== false) {
            // key-value pair হিসেবে yield
            yield array_combine($header, $row);
        }
    } finally {
        fclose($handle);
    }
}

// ১০ লাখ সারির CSV — মেমোরি ব্যবহার ন্যূনতম
foreach (readCsvLines('/data/products.csv') as $row) {
    // প্রতিটি সারি একটি associative array
    processProduct($row['name'], (float) $row['price']);
}

// --- ৩গ. yield from — Generator Delegation ---
function innerGenerator(): Generator
{
    yield 'ক';
    yield 'খ';
}

function outerGenerator(): Generator
{
    yield 'অ';
    yield from innerGenerator(); // delegate
    yield 'গ';
}

// ফলাফল: অ, ক, খ, গ
foreach (outerGenerator() as $value) {
    echo $value . "\n";
}

// --- ৩ঘ. Bidirectional Generator (send/receive) ---
function accumulator(): Generator
{
    $total = 0;
    while (true) {
        $value = yield $total; // yield করে, আবার বাইরে থেকে মান নেয়
        if ($value === null) break;
        $total += $value;
    }
}

$acc = accumulator();
$acc->current();        // Initialize: 0
$acc->send(100);        // total = 100
$acc->send(250);        // total = 350
echo $acc->send(50);    // 400
```

---

### ৪. JS Generators (`function*`) — Lazy Iterator

```javascript
// --- ৪ক. বেসিক Generator ---
function* rangeGenerator(start, end, step = 1) {
  for (let i = start; i <= end; i += step) {
    yield i;
  }
}

for (const num of rangeGenerator(1, 10_000_000)) {
  if (num > 5) break; // lazy — শুধু ৫টি মান তৈরি হয়
  console.log(num);
}

// --- ৪খ. Infinite Sequence ---
function* fibonacci() {
  let [a, b] = [0n, 1n]; // BigInt ব্যবহার
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// প্রথম ২০টি ফিবোনাচ্চি সংখ্যা নাও
function take(iterable, count) {
  const result = [];
  for (const item of iterable) {
    result.push(item);
    if (result.length >= count) break;
  }
  return result;
}

console.log(take(fibonacci(), 20));

// --- ৪গ. Generator Composition (yield*) ---
function* concat(...iterables) {
  for (const iterable of iterables) {
    yield* iterable;
  }
}

const merged = concat([1, 2], rangeGenerator(3, 5), [6, 7]);
console.log([...merged]); // [1, 2, 3, 4, 5, 6, 7]

// --- ৪ঘ. Pipeline Pattern with Generators ---
function* filter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}

function* map(iterable, transform) {
  for (const item of iterable) {
    yield transform(item);
  }
}

// Pipeline: ১-১০০ থেকে → জোড় সংখ্যা → বর্গ → প্রথম ৫টি
const pipeline = take(
  map(
    filter(rangeGenerator(1, 100), n => n % 2 === 0),
    n => n * n
  ),
  5
);
console.log(pipeline); // [4, 16, 36, 64, 100]
```

---

### ৫. Async Iterator

#### JS — Async Iteration (`for await...of`)

```javascript
// --- ৫ক. Async Generator — API Pagination ---
async function* fetchPaginatedProducts(baseUrl, pageSize = 20) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(
      `${baseUrl}?page=${page}&limit=${pageSize}`
    );
    const data = await response.json();

    for (const product of data.items) {
      yield product;
    }

    hasMore = data.hasNextPage;
    page++;
  }
}

// ব্যবহার — সমস্ত প্রোডাক্ট একটি একটি করে আসবে, পেজিনেশন স্বয়ংক্রিয়
async function displayAllProducts() {
  const products = fetchPaginatedProducts('https://api.daraz.bd/products');

  for await (const product of products) {
    console.log(`${product.name}: ৳${product.price}`);

    // যেকোনো সময় থামাতে পারি
    if (product.price > 50000) break;
  }
}

// --- ৫খ. Async Iterable Class ---
class AsyncFileReader {
  #filePath;
  #chunkSize;

  constructor(filePath, chunkSize = 1024) {
    this.#filePath = filePath;
    this.#chunkSize = chunkSize;
  }

  async *[Symbol.asyncIterator]() {
    const fs = await import('node:fs/promises');
    const handle = await fs.open(this.#filePath, 'r');

    try {
      const buffer = Buffer.alloc(this.#chunkSize);
      let bytesRead;

      do {
        ({ bytesRead } = await handle.read(buffer, 0, this.#chunkSize));
        if (bytesRead > 0) {
          yield buffer.subarray(0, bytesRead).toString('utf-8');
        }
      } while (bytesRead === this.#chunkSize);
    } finally {
      await handle.close();
    }
  }
}

// ব্যবহার:
const reader = new AsyncFileReader('./large-data.csv');
for await (const chunk of reader) {
  process(chunk);
}
```

#### PHP 8.3 — Fiber-Based Async Iterator

```php
<?php

declare(strict_types=1);

// PHP-তে native async iterator নেই, কিন্তু Fiber দিয়ে সিমুলেট করা যায়
final class AsyncApiIterator implements \IteratorAggregate
{
    public function __construct(
        private readonly string $baseUrl,
        private readonly int $pageSize = 20,
    ) {}

    public function getIterator(): Generator
    {
        $page = 1;
        $hasMore = true;

        while ($hasMore) {
            // Fiber ব্যবহার করে non-blocking HTTP call সিমুলেশন
            $fiber = new Fiber(function () use ($page): array {
                // বাস্তবে এখানে async HTTP client ব্যবহার হবে
                $url = "{$this->baseUrl}?page={$page}&limit={$this->pageSize}";
                $response = file_get_contents($url);
                return json_decode($response, true);
            });

            $fiber->start();
            $data = $fiber->getReturn();

            foreach ($data['items'] as $item) {
                yield $item;
            }

            $hasMore = $data['has_next_page'] ?? false;
            $page++;
        }
    }
}

// আরো বাস্তবসম্মত: Generator-based Async Pattern
function asyncPaginatedFetch(string $baseUrl, int $pageSize = 20): Generator
{
    $page = 1;

    while (true) {
        $url = "{$baseUrl}?page={$page}&limit={$pageSize}";
        $context = stream_context_create([
            'http' => ['timeout' => 30],
        ]);

        $json = file_get_contents($url, false, $context);
        $data = json_decode($json, true, 512, JSON_THROW_ON_ERROR);

        if (empty($data['items'])) {
            return; // Generator শেষ
        }

        yield from $data['items'];

        if (!($data['has_next_page'] ?? false)) {
            return;
        }

        $page++;
    }
}
```

---

## 🌍 Real-World Applicable Areas

### ১. Database Cursor / Pagination (Laravel)

```php
<?php

// --- Laravel cursor() — একটি একটি করে রেকর্ড আনে, মেমোরি সাশ্রয়ী ---

// ❌ খারাপ: ১০ লাখ রেকর্ড একবারে মেমোরিতে
$products = Product::where('active', true)->get();

// ✅ ভালো: cursor() — Generator ব্যবহার করে, একটি করে আনে
foreach (Product::where('active', true)->cursor() as $product) {
    $this->processProduct($product);
}

// ✅ chunk() — নির্দিষ্ট সংখ্যক ব্যাচে আনে
Product::where('active', true)->chunk(1000, function ($products) {
    foreach ($products as $product) {
        $this->processProduct($product);
    }
});

// ✅ lazy() — cursor() এর উন্নত সংস্করণ, LazyCollection রিটার্ন করে
Product::where('active', true)
    ->lazy(1000)
    ->filter(fn($p) => $p->price > 500)
    ->each(fn($p) => $this->export($p));
```

### ২. File Line-by-Line Reading

```php
<?php

// PHP — মেমোরি-সাশ্রয়ী ফাইল পড়া
function readLines(string $path): Generator
{
    $handle = fopen($path, 'r');
    try {
        while (($line = fgets($handle)) !== false) {
            yield trim($line);
        }
    } finally {
        fclose($handle);
    }
}

// ২GB লগ ফাইল থেকে error খুঁজুন — মেমোরি ব্যবহার ~কয়েক KB
$errorCount = 0;
foreach (readLines('/var/log/app.log') as $lineNumber => $line) {
    if (str_contains($line, 'ERROR')) {
        $errorCount++;
        echo "Line {$lineNumber}: {$line}\n";
    }
}
```

```javascript
// JS — ReadableStream to AsyncIterator
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

async function* readLines(filePath) {
  const rl = createInterface({
    input: createReadStream(filePath),
    crlfDelay: Infinity,
  });

  for await (const line of rl) {
    yield line;
  }
}

// ব্যবহার:
for await (const line of readLines('./server.log')) {
  if (line.includes('ERROR')) {
    console.error(line);
  }
}
```

### ৩. Tree/Graph Traversal (DFS/BFS Iterator)

```php
<?php

declare(strict_types=1);

final readonly class TreeNode
{
    /** @param TreeNode[] $children */
    public function __construct(
        public string $name,
        public array $children = [],
    ) {}
}

// DFS Iterator (Pre-order) — Generator দিয়ে
function dfsTraversal(TreeNode $root): Generator
{
    yield $root;
    foreach ($root->children as $child) {
        yield from dfsTraversal($child);
    }
}

// BFS Iterator — Generator দিয়ে
function bfsTraversal(TreeNode $root): Generator
{
    $queue = new SplQueue();
    $queue->enqueue($root);

    while (!$queue->isEmpty()) {
        $node = $queue->dequeue();
        yield $node;

        foreach ($node->children as $child) {
            $queue->enqueue($child);
        }
    }
}

// বাংলাদেশ প্রসঙ্গ: অফিসের বিভাগীয় কাঠামো
$company = new TreeNode('সিইও', [
    new TreeNode('সিটিও', [
        new TreeNode('ব্যাকএন্ড লিড', [
            new TreeNode('সিনিয়র ডেভ'),
            new TreeNode('জুনিয়র ডেভ'),
        ]),
        new TreeNode('ফ্রন্টএন্ড লিড'),
    ]),
    new TreeNode('সিএফও', [
        new TreeNode('হিসাবরক্ষক'),
    ]),
]);

echo "=== DFS ===\n";
foreach (dfsTraversal($company) as $node) {
    echo "  {$node->name}\n";
}

echo "=== BFS ===\n";
foreach (bfsTraversal($company) as $node) {
    echo "  {$node->name}\n";
}
```

```javascript
// JS Tree Traversal
class TreeNode {
  constructor(name, children = []) {
    this.name = name;
    this.children = children;
  }

  // DFS — default iterator
  *[Symbol.iterator]() {
    yield this;
    for (const child of this.children) {
      yield* child; // recursive yield delegation
    }
  }

  // BFS
  *bfs() {
    const queue = [this];
    while (queue.length > 0) {
      const node = queue.shift();
      yield node;
      queue.push(...node.children);
    }
  }
}

const company = new TreeNode('সিইও', [
  new TreeNode('সিটিও', [
    new TreeNode('ব্যাকএন্ড লিড', [
      new TreeNode('সিনিয়র ডেভ'),
      new TreeNode('জুনিয়র ডেভ'),
    ]),
    new TreeNode('ফ্রন্টএন্ড লিড'),
  ]),
  new TreeNode('সিএফও', [
    new TreeNode('হিসাবরক্ষক'),
  ]),
]);

// DFS (for...of default)
for (const node of company) {
  console.log(`DFS: ${node.name}`);
}

// BFS
for (const node of company.bfs()) {
  console.log(`BFS: ${node.name}`);
}
```

### ৪. Full Working Example: Paginated API Iterator

```javascript
// সম্পূর্ণ কার্যকরী উদাহরণ — Paginated API Iterator

class PaginatedApiIterator {
  #baseUrl;
  #params;
  #pageSize;

  constructor(baseUrl, params = {}, pageSize = 25) {
    this.#baseUrl = baseUrl;
    this.#params = params;
    this.#pageSize = pageSize;
  }

  async *[Symbol.asyncIterator]() {
    let cursor = null;
    let totalFetched = 0;

    while (true) {
      const url = new URL(this.#baseUrl);
      url.searchParams.set('limit', this.#pageSize);
      Object.entries(this.#params).forEach(([k, v]) =>
        url.searchParams.set(k, v)
      );
      if (cursor) url.searchParams.set('cursor', cursor);

      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`API Error: ${response.status}`);
      }

      const { data, nextCursor, total } = await response.json();

      for (const item of data) {
        yield item;
        totalFetched++;
      }

      if (!nextCursor || data.length < this.#pageSize) break;
      cursor = nextCursor;
    }
  }

  // Utility: সব ডাটা একটি array-তে
  async toArray() {
    const items = [];
    for await (const item of this) {
      items.push(item);
    }
    return items;
  }

  // Utility: প্রথম N টি আইটেম
  async take(count) {
    const items = [];
    for await (const item of this) {
      items.push(item);
      if (items.length >= count) break;
    }
    return items;
  }
}

// ব্যবহার — দারাজ-স্টাইল API
const products = new PaginatedApiIterator(
  'https://api.example.bd/products',
  { category: 'electronics', sort: 'price_asc' }
);

// সব প্রোডাক্ট iterate
for await (const product of products) {
  console.log(`${product.name}: ৳${product.price}`);
}

// অথবা শুধু প্রথম ১০টি
const top10 = await products.take(10);
```

### ৫. PHP SPL Iterators

```php
<?php

declare(strict_types=1);

// --- ArrayIterator ---
$items = new ArrayIterator(['আম', 'জাম', 'কাঁঠাল', 'লিচু']);

// --- LimitIterator: শুধু প্রথম ২টি ---
$limited = new LimitIterator($items, 0, 2);
foreach ($limited as $fruit) {
    echo $fruit . "\n"; // আম, জাম
}

// --- FilterIterator: কাস্টম ফিল্টার ---
$filtered = new CallbackFilterIterator(
    $items,
    fn(string $fruit): bool => mb_strlen($fruit) > 2
);

// --- RecursiveDirectoryIterator: ডিরেক্টরি traverse ---
$dir = new RecursiveDirectoryIterator('/var/www/project/src');
$flatDir = new RecursiveIteratorIterator(
    $dir,
    RecursiveIteratorIterator::SELF_FIRST
);

foreach ($flatDir as $file) {
    if ($file->isFile() && $file->getExtension() === 'php') {
        echo $file->getPathname() . "\n";
    }
}

// --- AppendIterator: একাধিক Iterator জোড়া ---
$combined = new AppendIterator();
$combined->append(new ArrayIterator([1, 2, 3]));
$combined->append(new ArrayIterator([4, 5, 6]));
// ফলাফল: 1, 2, 3, 4, 5, 6

// --- CachingIterator: look-ahead capability ---
$caching = new CachingIterator(
    new ArrayIterator(['প্রথম', 'দ্বিতীয়', 'তৃতীয়']),
    CachingIterator::FULL_CACHE
);

foreach ($caching as $item) {
    $hasNext = $caching->hasNext();
    echo "{$item}" . ($hasNext ? ' → ' : "\n");
}
// ফলাফল: প্রথম → দ্বিতীয় → তৃতীয়
```

### ৬. Infinite Sequences

```php
<?php

// PHP — অসীম ফিবোনাচ্চি Generator
function fibonacci(): Generator
{
    [$a, $b] = [0, 1];
    while (true) {
        yield $a;
        [$a, $b] = [$b, $a + $b];
    }
}

// মৌলিক সংখ্যা Generator
function primes(): Generator
{
    yield 2;
    $n = 3;
    while (true) {
        $isPrime = true;
        for ($i = 2; $i <= (int) sqrt($n); $i++) {
            if ($n % $i === 0) {
                $isPrime = false;
                break;
            }
        }
        if ($isPrime) yield $n;
        $n += 2;
    }
}

// প্রথম ১০টি মৌলিক সংখ্যা
$count = 0;
foreach (primes() as $prime) {
    echo $prime . ' ';
    if (++$count >= 10) break;
}
// 2 3 5 7 11 13 17 19 23 29
```

---

## 🔥 Advanced Deep Dive

### Internal vs External Iterator

```
┌──────────────────────────────────────────────────────────────┐
│                   Iterator-এর দুই ধরন                        │
├─────────────────────────┬────────────────────────────────────┤
│   Internal Iterator     │   External Iterator                │
├─────────────────────────┼────────────────────────────────────┤
│ কালেকশন নিজেই iterate  │ ক্লায়েন্ট iterate নিয়ন্ত্রণ করে  │
│ করে, callback পায়       │ (next, hasNext কল করে)             │
│                         │                                    │
│ PHP: array_map,         │ PHP: Iterator interface,           │
│ array_filter,           │ Generator, foreach                 │
│ Collection::each()      │                                    │
│                         │                                    │
│ JS: forEach, map,       │ JS: for...of, Symbol.iterator,    │
│ filter, reduce          │ Generator                          │
│                         │                                    │
│ সুবিধা: সরল সিনট্যাক্স │ সুবিধা: সম্পূর্ণ নিয়ন্ত্রণ,     │
│                         │ যেকোনো সময় থামানো যায়,            │
│                         │ একাধিক iterator পাশাপাশি           │
│ অসুবিধা: থামানো যায়    │                                    │
│ না (early break কঠিন)  │ অসুবিধা: বেশি কোড লিখতে হয়       │
└─────────────────────────┴────────────────────────────────────┘
```

### PHP `yield` / Generator গভীর বিশ্লেষণ

```php
<?php

declare(strict_types=1);

// Generator আসলে কী?
// যখন একটি ফাংশনে yield থাকে, PHP সেই ফাংশনকে সরাসরি execute করে না।
// পরিবর্তে একটি Generator অবজেক্ট রিটার্ন করে যা Iterator ইমপ্লিমেন্ট করে।

function exampleGenerator(): Generator
{
    echo "yield এর আগে\n";   // current() বা foreach-এ প্রথমবার execute হবে
    yield 'প্রথম';            // এখানে pause হবে
    echo "দুই yield এর মাঝে\n";
    yield 'দ্বিতীয়';          // আবার pause
    echo "yield এর পরে\n";    // last next() এ execute হবে
}

$gen = exampleGenerator(); // কোনো কিছুই execute হয় না!
echo "Generator তৈরি হলো\n";

// এখন execute শুরু — প্রথম yield পর্যন্ত
$gen->current(); // আউটপুট: "yield এর আগে" → return: "প্রথম"
$gen->next();    // "দুই yield এর মাঝে" → return: "দ্বিতীয়"
$gen->next();    // "yield এর পরে" → valid() = false

// Generator-এর return value
function withReturn(): Generator
{
    yield 'a';
    yield 'b';
    return 'শেষ মান'; // foreach এ আসবে না, getReturn() দিয়ে নিতে হবে
}

$gen = withReturn();
foreach ($gen as $val) { /* a, b */ }
echo $gen->getReturn(); // "শেষ মান"
```

### JS `for...of` প্রোটোকল বিস্তারিত

```javascript
// for...of কীভাবে কাজ করে — ভেতরের গল্প

const iterable = {
  [Symbol.iterator]() {
    let step = 0;
    return {
      next() {
        step++;
        switch (step) {
          case 1: return { value: 'ক', done: false };
          case 2: return { value: 'খ', done: false };
          case 3: return { value: 'গ', done: false };
          default: return { value: undefined, done: true };
        }
      }
    };
  }
};

// for...of আসলে এটাই করে:
const iterator = iterable[Symbol.iterator]();
let result = iterator.next();
while (!result.done) {
  const value = result.value;
  console.log(value);          // ক, খ, গ
  result = iterator.next();
}

// Well-known iterables যারা for...of সাপোর্ট করে:
// String, Array, Map, Set, TypedArray, NodeList, arguments,
// Generator objects, যেকোনো [Symbol.iterator] আছে এমন object
```

### Lazy Evaluation with Generators

```javascript
// Lazy Pipeline — প্রতিটি উপাদান সম্পূর্ণ pipeline দিয়ে যায়
// আগে পরেরটি শুরু হয় না

function* naturals() {
  let n = 1;
  while (true) yield n++;
}

function* filterGen(source, pred) {
  for (const item of source) {
    if (pred(item)) yield item;
  }
}

function* mapGen(source, fn) {
  for (const item of source) {
    yield fn(item);
  }
}

function* takeGen(source, n) {
  let count = 0;
  for (const item of source) {
    yield item;
    if (++count >= n) return;
  }
}

// Pipeline: অসীম সংখ্যা → ৩ দিয়ে ভাগ যায় → বর্গ → প্রথম ৫টি
const result = takeGen(
  mapGen(
    filterGen(naturals(), n => n % 3 === 0),
    n => n ** 2
  ),
  5
);

console.log([...result]); // [9, 36, 81, 144, 225]
// শুধুমাত্র ১৫টি সংখ্যা (1-15) পরীক্ষা করেছে — অসীমের মধ্যে!
```

### Iterator + Composite প্যাটার্ন

```php
<?php

declare(strict_types=1);

// মেনু সিস্টেম — Composite + Iterator
interface MenuComponent
{
    public function getName(): string;
    public function getIterator(): Generator;
}

final readonly class MenuItem implements MenuComponent
{
    public function __construct(
        private string $name,
        private float $price,
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getPrice(): float
    {
        return $this->price;
    }

    public function getIterator(): Generator
    {
        yield $this;
    }
}

final class Menu implements MenuComponent
{
    /** @var MenuComponent[] */
    private array $components = [];

    public function __construct(
        private readonly string $name,
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function add(MenuComponent $component): void
    {
        $this->components[] = $component;
    }

    // Composite Iterator — সব নেস্টেড আইটেম flat করে
    public function getIterator(): Generator
    {
        foreach ($this->components as $component) {
            yield from $component->getIterator();
        }
    }
}

// বাংলাদেশী রেস্টুরেন্ট মেনু
$mainMenu = new Menu('প্রধান মেনু');

$breakfast = new Menu('সকালের নাস্তা');
$breakfast->add(new MenuItem('পরোটা + ডাল', 60.0));
$breakfast->add(new MenuItem('খিচুড়ি', 80.0));

$lunch = new Menu('দুপুরের খাবার');
$lunch->add(new MenuItem('ভাত + ইলিশ', 350.0));
$lunch->add(new MenuItem('বিরিয়ানি', 250.0));

$mainMenu->add($breakfast);
$mainMenu->add($lunch);

// flat iteration — সব আইটেম
foreach ($mainMenu->getIterator() as $item) {
    /** @var MenuItem $item */
    echo "{$item->getName()}: ৳{$item->getPrice()}\n";
}
```

### Pipeline of Iterators (filter → map → take)

```php
<?php

declare(strict_types=1);

// Fluent Iterator Pipeline (Laravel Collection স্টাইল)
final class IteratorPipeline
{
    private iterable $source;

    private function __construct(iterable $source)
    {
        $this->source = $source;
    }

    public static function from(iterable $source): self
    {
        return new self($source);
    }

    public function filter(callable $predicate): self
    {
        $source = $this->source;
        $this->source = (function () use ($source, $predicate): Generator {
            foreach ($source as $key => $value) {
                if ($predicate($value, $key)) {
                    yield $key => $value;
                }
            }
        })();
        return $this;
    }

    public function map(callable $transform): self
    {
        $source = $this->source;
        $this->source = (function () use ($source, $transform): Generator {
            foreach ($source as $key => $value) {
                yield $key => $transform($value, $key);
            }
        })();
        return $this;
    }

    public function take(int $limit): self
    {
        $source = $this->source;
        $this->source = (function () use ($source, $limit): Generator {
            $count = 0;
            foreach ($source as $key => $value) {
                yield $key => $value;
                if (++$count >= $limit) return;
            }
        })();
        return $this;
    }

    public function toArray(): array
    {
        return iterator_to_array($this->source, preserve_keys: false);
    }

    /** @return Generator */
    public function getIterator(): Generator
    {
        yield from $this->source;
    }
}

// ব্যবহার — সম্পূর্ণ lazy, একটি একটি উপাদান pipeline দিয়ে যায়
$result = IteratorPipeline::from(range(1, 1_000_000))
    ->filter(fn(int $n): bool => $n % 2 === 0)    // জোড়
    ->map(fn(int $n): int => $n ** 2)               // বর্গ
    ->filter(fn(int $n): bool => $n > 100)          // ১০০ এর বেশি
    ->take(5)                                        // প্রথম ৫টি
    ->toArray();

var_dump($result); // [64 → skip, 144, 196, 256, 324, 400]
```

---

## ✅ Pros (সুবিধা)

| সুবিধা | ব্যাখ্যা |
|---------|---------|
| **Single Responsibility** | Traversal যুক্তি কালেকশন থেকে আলাদা — প্রতিটি ক্লাসের একটিই দায়িত্ব |
| **Open/Closed** | নতুন ধরনের Iterator যোগ করা যায় কালেকশন পরিবর্তন না করে |
| **Uniform Interface** | যেকোনো কালেকশনে একই `foreach`/`for...of` কাজ করে |
| **Lazy Evaluation** | Generator দিয়ে অসীম সিকোয়েন্স ও বিশাল ডাটাসেট মেমোরি-সাশ্রয়ীভাবে প্রসেস |
| **Multiple Traversal** | একই কালেকশনে একসাথে একাধিক Iterator চালানো যায় |
| **Encapsulation** | কালেকশনের internal structure সুরক্ষিত থাকে |

## ❌ Cons (অসুবিধা)

| অসুবিধা | ব্যাখ্যা |
|----------|---------|
| **Overhead** | সাধারণ array-র জন্য Iterator তৈরি করা অতিরিক্ত জটিলতা |
| **Performance** | Direct index access-এর চেয়ে Iterator ধীর হতে পারে |
| **Debugging কঠিন** | Generator-এর lazy execution trace করা কঠিন |
| **State Management** | External Iterator-এ current position সামলানো ভুলপ্রবণ |
| **One-way** | বেশিরভাগ Iterator শুধু সামনে যায়, পেছনে ফেরা যায় না (বিশেষত Generator) |

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ভুল ১: Generator পুনরায় ব্যবহার

```php
<?php
// ❌ ভুল — Generator একবারই ব্যবহার করা যায়!
function numbers(): Generator
{
    yield 1;
    yield 2;
    yield 3;
}

$gen = numbers();
foreach ($gen as $n) { echo $n; }  // 1, 2, 3 ✅
foreach ($gen as $n) { echo $n; }  // কিছুই আসবে না! ❌

// ✅ সঠিক — প্রতিবার নতুন Generator তৈরি করুন
foreach (numbers() as $n) { echo $n; }  // 1, 2, 3
foreach (numbers() as $n) { echo $n; }  // 1, 2, 3
```

```javascript
// ❌ JS-তেও একই সমস্যা
function* nums() { yield 1; yield 2; yield 3; }
const gen = nums();
console.log([...gen]); // [1, 2, 3]
console.log([...gen]); // [] — খালি!

// ✅ প্রতিবার নতুন generator কল করুন
console.log([...nums()]); // [1, 2, 3]
console.log([...nums()]); // [1, 2, 3]
```

### ভুল ২: Iteration-এর সময় কালেকশন পরিবর্তন

```php
<?php
// ❌ ভুল — iterate করতে করতে পরিবর্তন (undefined behavior)
$items = ['a', 'b', 'c', 'd'];
foreach ($items as $i => $item) {
    if ($item === 'b') {
        unset($items[$i]); // বিপদজনক!
    }
}

// ✅ সঠিক — আলাদা array-তে রাখুন, পরে মুছুন
$toRemove = [];
foreach ($items as $i => $item) {
    if ($item === 'b') $toRemove[] = $i;
}
foreach ($toRemove as $i) unset($items[$i]);
```

### ভুল ৩: Async Iterator-এ error handling ভুলে যাওয়া

```javascript
// ❌ ভুল — error handle করা হয়নি
for await (const item of fetchPaginated(url)) {
  process(item);
}

// ✅ সঠিক — error handling সহ
try {
  for await (const item of fetchPaginated(url)) {
    try {
      await process(item);
    } catch (itemError) {
      console.error(`আইটেম প্রসেসিং ব্যর্থ: ${itemError.message}`);
      // continue অথবা break সিদ্ধান্ত নিন
    }
  }
} catch (iterationError) {
  console.error(`Iteration ব্যর্থ: ${iterationError.message}`);
}
```

### ভুল ৪: অপ্রয়োজনীয় Iterator তৈরি

```php
<?php
// ❌ অতিরিক্ত — সাধারণ array-র জন্য Iterator ক্লাস বানানো
class NumbersIterator implements Iterator { /* ৫০ লাইন কোড */ }

// ✅ সরল — array নিজেই iterable!
$numbers = [1, 2, 3, 4, 5];
foreach ($numbers as $n) { /* কাজ হয়ে গেল */ }
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

final class ProductCollectionTest extends TestCase
{
    #[Test]
    public function iterates_over_all_products(): void
    {
        $collection = new ProductCollection();
        $collection->addProduct(new Product('লুঙ্গি', 450.0, 'পোশাক'));
        $collection->addProduct(new Product('ইলিশ', 1200.0, 'খাবার'));

        $names = [];
        foreach ($collection as $product) {
            $names[] = $product->name;
        }

        $this->assertSame(['লুঙ্গি', 'ইলিশ'], $names);
    }

    #[Test]
    public function empty_collection_yields_nothing(): void
    {
        $collection = new ProductCollection();
        $items = iterator_to_array($collection);

        $this->assertEmpty($items);
    }

    #[Test]
    public function supports_multiple_iterations(): void
    {
        $collection = new ProductCollection();
        $collection->addProduct(new Product('পাঞ্জাবি', 1800.0, 'পোশাক'));

        $first = iterator_to_array($collection);
        $second = iterator_to_array($collection);

        $this->assertEquals($first, $second);
    }

    #[Test]
    public function filter_iterator_filters_by_category(): void
    {
        $collection = new ProductCollection();
        $collection->addProduct(new Product('লুঙ্গি', 450.0, 'পোশাক'));
        $collection->addProduct(new Product('ইলিশ', 1200.0, 'খাবার'));
        $collection->addProduct(new Product('শাড়ি', 8500.0, 'পোশাক'));

        $filtered = new CategoryFilterIterator(
            $collection->getIterator(),
            'পোশাক'
        );

        $names = array_map(
            fn(Product $p) => $p->name,
            iterator_to_array($filtered, preserve_keys: false),
        );

        $this->assertSame(['লুঙ্গি', 'শাড়ি'], $names);
    }

    #[Test]
    public function generator_yields_lazily(): void
    {
        $callCount = 0;
        $generator = (function () use (&$callCount): Generator {
            for ($i = 0; $i < 1000; $i++) {
                $callCount++;
                yield $i;
            }
        })();

        // শুধু ৩টি আইটেম নিই
        $results = [];
        foreach ($generator as $value) {
            $results[] = $value;
            if (count($results) >= 3) break;
        }

        $this->assertSame([0, 1, 2], $results);
        $this->assertSame(3, $callCount); // শুধু ৩ বার execute হয়েছে!
    }

    #[Test]
    public function tree_dfs_traversal_order(): void
    {
        $root = new TreeNode('A', [
            new TreeNode('B', [
                new TreeNode('D'),
                new TreeNode('E'),
            ]),
            new TreeNode('C'),
        ]);

        $names = array_map(
            fn(TreeNode $n) => $n->name,
            iterator_to_array(dfsTraversal($root), preserve_keys: false),
        );

        $this->assertSame(['A', 'B', 'D', 'E', 'C'], $names);
    }

    #[Test]
    public function tree_bfs_traversal_order(): void
    {
        $root = new TreeNode('A', [
            new TreeNode('B', [
                new TreeNode('D'),
                new TreeNode('E'),
            ]),
            new TreeNode('C'),
        ]);

        $names = array_map(
            fn(TreeNode $n) => $n->name,
            iterator_to_array(bfsTraversal($root), preserve_keys: false),
        );

        $this->assertSame(['A', 'B', 'C', 'D', 'E'], $names);
    }

    #[Test]
    public function pipeline_processes_lazily(): void
    {
        $result = IteratorPipeline::from(range(1, 100))
            ->filter(fn(int $n): bool => $n % 2 === 0)
            ->map(fn(int $n): int => $n * 10)
            ->take(3)
            ->toArray();

        $this->assertSame([20, 40, 60], $result);
    }
}
```

### Jest (JS)

```javascript
// iterator.test.js
import { describe, it, expect } from '@jest/globals';

describe('ProductCollection Iterator', () => {
  it('should iterate over all products with for...of', () => {
    const collection = new ProductCollection();
    collection
      .addProduct(new Product('লুঙ্গি', 450, 'পোশাক'))
      .addProduct(new Product('ইলিশ', 1200, 'খাবার'));

    const names = [];
    for (const product of collection) {
      names.push(product.name);
    }

    expect(names).toEqual(['লুঙ্গি', 'ইলিশ']);
  });

  it('should support spread operator', () => {
    const collection = new ProductCollection();
    collection.addProduct(new Product('পাঞ্জাবি', 1800, 'পোশাক'));

    const products = [...collection];
    expect(products).toHaveLength(1);
    expect(products[0].name).toBe('পাঞ্জাবি');
  });

  it('should support destructuring', () => {
    const collection = new ProductCollection();
    collection
      .addProduct(new Product('A', 100, 'x'))
      .addProduct(new Product('B', 200, 'y'));

    const [first, second] = collection;
    expect(first.name).toBe('A');
    expect(second.name).toBe('B');
  });

  it('empty collection yields nothing', () => {
    const collection = new ProductCollection();
    expect([...collection]).toEqual([]);
  });
});

describe('Generator Functions', () => {
  it('fibonacci generates correct sequence', () => {
    const fib = take(fibonacci(), 8);
    expect(fib).toEqual([0n, 1n, 1n, 2n, 3n, 5n, 8n, 13n]);
  });

  it('generators are lazy - only compute needed values', () => {
    let computeCount = 0;

    function* tracked() {
      for (let i = 0; i < 1000; i++) {
        computeCount++;
        yield i;
      }
    }

    const result = take(tracked(), 3);
    expect(result).toEqual([0, 1, 2]);
    expect(computeCount).toBe(3);
  });

  it('generator pipeline works correctly', () => {
    const result = [
      ...takeGen(
        mapGen(
          filterGen(naturals(), n => n % 2 === 0),
          n => n * n
        ),
        3
      )
    ];

    expect(result).toEqual([4, 16, 36]);
  });
});

describe('TreeNode Iterator', () => {
  const tree = new TreeNode('A', [
    new TreeNode('B', [
      new TreeNode('D'),
      new TreeNode('E'),
    ]),
    new TreeNode('C'),
  ]);

  it('DFS via Symbol.iterator (default)', () => {
    const names = [...tree].map(n => n.name);
    expect(names).toEqual(['A', 'B', 'D', 'E', 'C']);
  });

  it('BFS traversal', () => {
    const names = [...tree.bfs()].map(n => n.name);
    expect(names).toEqual(['A', 'B', 'C', 'D', 'E']);
  });
});

describe('Async Iterator', () => {
  it('should iterate over async paginated data', async () => {
    // Mock fetch
    const mockData = [
      { items: [{ id: 1 }, { id: 2 }], nextCursor: 'abc', total: 3 },
      { items: [{ id: 3 }], nextCursor: null, total: 3 },
    ];

    let callIndex = 0;
    global.fetch = async () => ({
      ok: true,
      json: async () => mockData[callIndex++],
    });

    const iterator = new PaginatedApiIterator('https://api.test.com/items');
    const items = await iterator.toArray();

    expect(items).toEqual([{ id: 1 }, { id: 2 }, { id: 3 }]);
  });

  it('take() should limit results', async () => {
    global.fetch = async () => ({
      ok: true,
      json: async () => ({
        data: Array.from({ length: 25 }, (_, i) => ({ id: i + 1 })),
        nextCursor: 'next',
      }),
    });

    const iterator = new PaginatedApiIterator('https://api.test.com/items');
    const items = await iterator.take(5);

    expect(items).toHaveLength(5);
    expect(items[0].id).toBe(1);
  });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| **Composite** | Iterator দিয়ে Composite tree structure traverse করা হয়। `yield from` দিয়ে recursive traversal অত্যন্ত সহজ। |
| **Factory Method** | কালেকশন ক্লাস Factory Method ব্যবহার করে উপযুক্ত Iterator তৈরি করে (`createIterator()`)। |
| **Memento** | Iterator-এর বর্তমান অবস্থান (position) Memento হিসেবে সংরক্ষণ করে পরে সেই অবস্থান থেকে শুরু করা যায়। |
| **Visitor** | Iterator কালেকশনের উপাদানগুলোতে পৌঁছায়, Visitor সেগুলোতে operation চালায়। একসাথে ব্যবহারে শক্তিশালী combination তৈরি হয়। |
| **Strategy** | বিভিন্ন traversal অ্যালগরিদম (DFS, BFS) কে Strategy হিসেবে Iterator-এ inject করা যায়। |
| **Observer** | Iterator যখন কোনো উপাদানে পৌঁছায় তখন Observer-কে notify করা যায়। |

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

- **জটিল কালেকশন গঠন** (Tree, Graph, Custom Data Structure) — ক্লায়েন্টকে internal structure জানতে হবে না
- **একাধিক traversal পদ্ধতি দরকার** — DFS, BFS, reverse, filtered — প্রতিটির জন্য আলাদা Iterator
- **বিশাল ডাটাসেট** — Generator দিয়ে lazy evaluation, মেমোরি সাশ্রয়ী (বড় CSV, database cursor)
- **অসীম সিকোয়েন্স** — Fibonacci, prime numbers, event stream
- **Uniform interface চান** — বিভিন্ন ডাটা সোর্স (DB, API, file) একই `foreach`/`for...of` দিয়ে প্রসেস
- **API pagination** — স্বয়ংক্রিয়ভাবে পেজ থেকে পেজে যাওয়া
- **Pipeline processing** — filter → map → reduce চেইন

### ❌ ব্যবহার করবেন না যখন:

- **সাধারণ array/list** — PHP array বা JS Array নিজেই iterable, আলাদা Iterator অপ্রয়োজনীয়
- **Random access দরকার** — Iterator sequential, নির্দিষ্ট index-এ সরাসরি যেতে পারে না
- **ছোট ডাটাসেট** — ১০-২০টি আইটেমের জন্য Iterator overhead অর্থহীন
- **Bidirectional traversal** — Iterator সাধারণত একমুখী; doubly-linked list-এর জন্য সরাসরি traversal ভালো
- **Simple CRUD** — ডাটাবেজ থেকে সব রেকর্ড এনে দেখানোতে জটিল Iterator লাগে না

---

## 📋 সারসংক্ষেপ

```
Iterator প্যাটার্ন — মূল বিষয়সমূহ:

১. কালেকশনের internal structure লুকিয়ে uniform traversal interface দেয়
২. PHP: Iterator/IteratorAggregate interface, Generator (yield), SPL Iterators
৩. JS: Symbol.iterator protocol, Generator (function*), Async Iterator
৪. Lazy evaluation — Generator দিয়ে অসীম সিকোয়েন্স ও বিশাল ডাটাসেট হ্যান্ডল
৫. Pipeline — filter → map → take চেইন করে declarative data processing
৬. Real-world: DB cursor, file reading, tree traversal, API pagination
৭. সতর্কতা: Generator একবারই ব্যবহারযোগ্য, iteration-কালে modification বিপদজনক
```

### এক নজরে PHP vs JS তুলনা

```
┌──────────────────┬─────────────────────────┬──────────────────────────┐
│ বিষয়             │ PHP 8.3                 │ JS ES2022+               │
├──────────────────┼─────────────────────────┼──────────────────────────┤
│ Interface        │ Iterator,               │ Symbol.iterator,         │
│                  │ IteratorAggregate       │ Symbol.asyncIterator     │
│ Generator        │ yield, yield from       │ yield, yield*            │
│ Lazy Loop        │ foreach                 │ for...of                 │
│ Async            │ Fiber (manual)          │ for await...of (native)  │
│ Built-in         │ SPL Iterators           │ Array, Map, Set, String  │
│ Pipeline         │ Generator chaining      │ Generator chaining,      │
│                  │                         │ Iterator Helpers (Stage3)│
│ Memory trick     │ Generator = ~KB         │ Generator = ~KB          │
│ Reusable?        │ Generator: ❌ একবার     │ Generator: ❌ একবার      │
│                  │ IteratorAggregate: ✅   │ Iterable class: ✅       │
└──────────────────┴─────────────────────────┴──────────────────────────┘
```

> **মনে রাখুন:** Iterator প্যাটার্ন শুধু একটি design pattern নয় — এটি আধুনিক প্রোগ্রামিং ভাষাগুলোর মূল ভিত্তি। PHP-র `foreach`, JS-এর `for...of`, Python-এর `for...in` — সবই Iterator protocol-এর উপর দাঁড়িয়ে। এই প্যাটার্ন বুঝলে আপনি ভাষার গভীরতা বুঝবেন। 🚀
