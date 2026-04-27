# 🌳 Composite প্যাটার্ন (Composite Pattern)

## 📌 সংজ্ঞা ও উদ্দেশ্য

> **GoF Definition:** *"Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly."*

Composite প্যাটার্ন হলো একটি **Structural Design Pattern** যা অবজেক্টগুলোকে **ট্রি স্ট্রাকচারে** সাজিয়ে part-whole (অংশ-সমগ্র) hierarchy তৈরি করে। এই প্যাটার্নের মূল শক্তি হলো — ক্লায়েন্ট কোড **একটি single object** এবং **একটি composite object** (যেটার ভিতরে অনেক অবজেক্ট আছে) — দুটোকেই **একইভাবে** treat করতে পারে।

### মূল ধারণা

ধরুন, আপনার কাছে একটি বাক্স আছে। সেই বাক্সের ভিতরে আরো ছোট বাক্স আছে, এবং সেই ছোট বাক্সগুলোর ভিতরে পণ্য আছে। এখন আপনি যদি পুরো বাক্সের মোট দাম বের করতে চান, তাহলে আপনাকে প্রতিটি nested বাক্সের ভিতরে ঢুকে সব পণ্যের দাম যোগ করতে হবে। Composite প্যাটার্ন এই কাজটাকে **uniform interface** দিয়ে সহজ করে দেয়।

### তিনটি মূল Component:

| Component | ভূমিকা | উদাহরণ |
|-----------|--------|--------|
| **Component** | সকল element-এর common interface ঘোষণা করে | `FileSystemNode` |
| **Leaf** | ট্রি-র শেষ প্রান্তের element, এর কোনো child নেই | `File` |
| **Composite** | Leaf ও অন্যান্য Composite ধারণ করে, child management করে | `Directory` |

---

## 🏠 বাস্তব উদাহরণ

### ফাইল সিস্টেম — সবচেয়ে ক্লাসিক উদাহরণ

আপনার কম্পিউটারে `/home/user/projects/` ফোল্ডার আছে। এর ভিতরে কিছু ফাইল আছে, কিছু সাব-ফোল্ডার আছে, সেগুলোর ভিতরেও আবার ফাইল ও ফোল্ডার আছে। কিন্তু আপনি যখন `ls` কমান্ড চালান বা ফাইল ম্যানেজারে দেখেন — **ফাইল ও ফোল্ডার দুটোই একই লিস্টে দেখায়**। দুটোরই নাম আছে, সাইজ আছে, পারমিশন আছে। ফোল্ডার শুধু অতিরিক্তভাবে অন্যান্য আইটেম ধারণ করে — এটাই Composite প্যাটার্ন!

### বাংলাদেশের প্রেক্ষাপটে উদাহরণ

**১. Daraz প্রোডাক্ট ক্যাটাগরি:**
```
ইলেকট্রনিক্স (Category - Composite)
├── মোবাইল ফোন (SubCategory - Composite)
│   ├── Samsung Galaxy S24 (Product - Leaf)
│   ├── iPhone 15 (Product - Leaf)
│   └── বাজেট ফোন (SubCategory - Composite)
│       ├── Redmi Note 13 (Product - Leaf)
│       └── Realme C67 (Product - Leaf)
├── ল্যাপটপ (SubCategory - Composite)
│   ├── HP Pavilion (Product - Leaf)
│   └── Lenovo IdeaPad (Product - Leaf)
└── হেডফোন (SubCategory - Composite)
    └── Sony WH-1000XM5 (Product - Leaf)
```

**২. বাংলাদেশ সরকারের প্রশাসনিক কাঠামো:**
```
জাতীয় সরকার (Composite)
├── বিভাগ - ঢাকা (Composite)
│   ├── জেলা - ঢাকা (Composite)
│   │   ├── উপজেলা - সাভার (Composite)
│   │   │   ├── কর্মকর্তা: আহমেদ (Leaf)
│   │   │   └── কর্মকর্তা: করিম (Leaf)
│   │   └── উপজেলা - ধামরাই (Composite)
│   │       └── কর্মকর্তা: রহিম (Leaf)
│   └── জেলা - গাজীপুর (Composite)
│       └── কর্মকর্তা: সালাম (Leaf)
└── বিভাগ - চট্টগ্রাম (Composite)
    └── জেলা - চট্টগ্রাম (Composite)
        └── কর্মকর্তা: জামাল (Leaf)
```

---

## 📊 UML ডায়াগ্রাম

```
                    ┌─────────────────────────┐
                    │    <<interface>>         │
                    │      Component           │
                    ├─────────────────────────┤
                    │ + operation(): void      │
                    │ + getSize(): number      │
                    │ + getName(): string      │
                    └────────┬────────────────┘
                             │
                 ┌───────────┴────────────┐
                 │                        │
       ┌─────────────────┐    ┌──────────────────────────┐
       │      Leaf        │    │       Composite           │
       ├─────────────────┤    ├──────────────────────────┤
       │ - name: string  │    │ - children: Component[]   │
       │ - size: number  │    │ - name: string            │
       ├─────────────────┤    ├──────────────────────────┤
       │ + operation()   │    │ + operation()             │
       │ + getSize()     │    │ + getSize()               │
       │ + getName()     │    │ + add(c: Component)       │
       └─────────────────┘    │ + remove(c: Component)    │
                              │ + getChild(i): Component  │
                              │ + getName()               │
                              └──────────────────────────┘

    ক্লায়েন্ট শুধু Component interface জানে।
    Leaf → শেষ প্রান্তের আইটেম (ফাইল, প্রোডাক্ট, কর্মচারী)
    Composite → container যেটা অন্যান্য Component ধারণ করে
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Composite — ভিত্তিমূলক উদাহরণ

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

// Component Interface — সকল element-এর জন্য common contract
interface Renderable
{
    public function render(int $depth = 0): string;
    public function getSize(): int;
}

// Leaf — শেষ প্রান্তের element, কোনো child নেই
final readonly class TextBlock implements Renderable
{
    public function __construct(
        private string $content,
    ) {}

    public function render(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        return "{$indent}{$this->content}";
    }

    public function getSize(): int
    {
        return mb_strlen($this->content);
    }
}

// Composite — অন্যান্য Component ধারণ করে
final class Container implements Renderable
{
    /** @var list<Renderable> */
    private array $children = [];

    public function __construct(
        private readonly string $title,
    ) {}

    public function add(Renderable ...$components): self
    {
        foreach ($components as $component) {
            $this->children[] = $component;
        }
        return $this;
    }

    public function remove(Renderable $component): self
    {
        $this->children = array_values(
            array_filter($this->children, fn(Renderable $c) => $c !== $component)
        );
        return $this;
    }

    public function render(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        $output = "{$indent}[{$this->title}]" . PHP_EOL;

        foreach ($this->children as $child) {
            $output .= $child->render($depth + 1) . PHP_EOL;
        }

        return rtrim($output);
    }

    // Composite-এর getSize() সব children-এর size-এর যোগফল
    public function getSize(): int
    {
        return array_reduce(
            $this->children,
            fn(int $total, Renderable $child) => $total + $child->getSize(),
            0
        );
    }
}

// ব্যবহার
$root = new Container('মূল পাতা');
$section1 = new Container('প্রথম অংশ');
$section1->add(
    new TextBlock('এটি প্রথম প্যারাগ্রাফ'),
    new TextBlock('এটি দ্বিতীয় প্যারাগ্রাফ'),
);

$section2 = new Container('দ্বিতীয় অংশ');
$nested = new Container('নেস্টেড অংশ');
$nested->add(new TextBlock('গভীরে লুকানো কন্টেন্ট'));
$section2->add(new TextBlock('সাধারণ কন্টেন্ট'), $nested);

$root->add($section1, $section2);

echo $root->render();
echo PHP_EOL . "মোট সাইজ: {$root->getSize()} characters" . PHP_EOL;
```

**JavaScript (ES2022+):**

```javascript
// Component Interface — Symbol ব্যবহার করে type identification
const COMPONENT = Symbol('composite-component');

class TextBlock {
    [COMPONENT] = true;

    #content;

    constructor(content) {
        this.#content = content;
    }

    render(depth = 0) {
        const indent = '  '.repeat(depth);
        return `${indent}${this.#content}`;
    }

    getSize() {
        return this.#content.length;
    }
}

class Container {
    [COMPONENT] = true;

    #title;
    #children = [];

    constructor(title) {
        this.#title = title;
    }

    add(...components) {
        for (const component of components) {
            if (!component[COMPONENT]) {
                throw new TypeError('শুধুমাত্র Component টাইপ যোগ করা যাবে');
            }
            this.#children.push(component);
        }
        return this;
    }

    remove(component) {
        this.#children = this.#children.filter(c => c !== component);
        return this;
    }

    render(depth = 0) {
        const indent = '  '.repeat(depth);
        const childOutput = this.#children
            .map(child => child.render(depth + 1))
            .join('\n');
        return `${indent}[${this.#title}]\n${childOutput}`;
    }

    getSize() {
        return this.#children.reduce((total, child) => total + child.getSize(), 0);
    }

    // Iterator protocol — for...of দিয়ে iterate করা যাবে
    *[Symbol.iterator]() {
        yield* this.#children;
    }
}

// ব্যবহার
const root = new Container('মূল পাতা');
const section1 = new Container('প্রথম অংশ');
section1.add(new TextBlock('স্বাগতম'), new TextBlock('এটি Composite প্যাটার্ন'));

root.add(section1, new TextBlock('সরাসরি টেক্সট'));

console.log(root.render());
console.log(`মোট সাইজ: ${root.getSize()}`);

// Iterator ব্যবহার
for (const child of root) {
    console.log(`Child size: ${child.getSize()}`);
}
```

---

### ২. File System — ফাইল ও ডিরেক্টরি

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

interface FileSystemNode
{
    public function getName(): string;
    public function getSize(): int;
    public function display(string $prefix = '', bool $isLast = true): string;
    public function find(string $name): ?FileSystemNode;
}

final readonly class File implements FileSystemNode
{
    public function __construct(
        private string $name,
        private int $size,        // bytes
        private string $extension,
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getSize(): int
    {
        return $this->size;
    }

    public function getExtension(): string
    {
        return $this->extension;
    }

    public function display(string $prefix = '', bool $isLast = true): string
    {
        $connector = $isLast ? '└── ' : '├── ';
        $sizeFormatted = $this->formatSize($this->size);
        return "{$prefix}{$connector}📄 {$this->name} ({$sizeFormatted})";
    }

    public function find(string $name): ?FileSystemNode
    {
        return $this->name === $name ? $this : null;
    }

    private function formatSize(int $bytes): string
    {
        return match (true) {
            $bytes >= 1_073_741_824 => round($bytes / 1_073_741_824, 2) . ' GB',
            $bytes >= 1_048_576     => round($bytes / 1_048_576, 2) . ' MB',
            $bytes >= 1_024         => round($bytes / 1_024, 2) . ' KB',
            default                 => $bytes . ' B',
        };
    }
}

final class Directory implements FileSystemNode
{
    /** @var list<FileSystemNode> */
    private array $children = [];

    public function __construct(
        private readonly string $name,
    ) {}

    public function add(FileSystemNode ...$nodes): self
    {
        foreach ($nodes as $node) {
            // duplicate check
            foreach ($this->children as $existing) {
                if ($existing->getName() === $node->getName()) {
                    throw new \InvalidArgumentException(
                        "'{$node->getName()}' ইতোমধ্যে '{$this->name}' ডিরেক্টরিতে আছে"
                    );
                }
            }
            $this->children[] = $node;
        }
        return $this;
    }

    public function getName(): string
    {
        return $this->name;
    }

    // Recursive — সকল children-এর সাইজের যোগফল
    public function getSize(): int
    {
        return array_reduce(
            $this->children,
            fn(int $total, FileSystemNode $node) => $total + $node->getSize(),
            0
        );
    }

    public function getChildCount(): int
    {
        return count($this->children);
    }

    public function display(string $prefix = '', bool $isLast = true): string
    {
        $connector = $isLast ? '└── ' : '├── ';
        $output = "{$prefix}{$connector}📁 {$this->name}/";
        $childPrefix = $prefix . ($isLast ? '    ' : '│   ');

        foreach ($this->children as $i => $child) {
            $isChildLast = ($i === count($this->children) - 1);
            $output .= PHP_EOL . $child->display($childPrefix, $isChildLast);
        }

        return $output;
    }

    // Recursive search — ট্রি-র মধ্যে নাম দিয়ে খোঁজা
    public function find(string $name): ?FileSystemNode
    {
        if ($this->name === $name) {
            return $this;
        }
        foreach ($this->children as $child) {
            $found = $child->find($name);
            if ($found !== null) {
                return $found;
            }
        }
        return null;
    }
}

// বাস্তব ব্যবহার — একটি প্রজেক্ট স্ট্রাকচার তৈরি
$project = new Directory('my-laravel-app');

$app = new Directory('app');
$models = new Directory('Models');
$models->add(
    new File('User.php', 2_048, 'php'),
    new File('Product.php', 1_536, 'php'),
);

$controllers = new Directory('Controllers');
$controllers->add(
    new File('UserController.php', 4_096, 'php'),
    new File('ProductController.php', 3_584, 'php'),
);

$app->add($models, $controllers);

$public = new Directory('public');
$public->add(
    new File('index.php', 512, 'php'),
    new File('style.css', 8_192, 'css'),
);

$project->add(
    $app,
    $public,
    new File('composer.json', 1_024, 'json'),
    new File('.env', 256, 'env'),
);

echo $project->display() . PHP_EOL;
echo "প্রজেক্টের মোট সাইজ: " . $project->getSize() . " bytes" . PHP_EOL;

// খোঁজা
$found = $project->find('User.php');
echo $found ? "পাওয়া গেছে: {$found->getName()}" : "পাওয়া যায়নি";
```

**JavaScript (ES2022+):**

```javascript
class FileNode {
    #name;
    #size;

    constructor(name, size) {
        this.#name = name;
        this.#size = size;
    }

    get name() { return this.#name; }
    get size() { return this.#size; }

    display(prefix = '', isLast = true) {
        const connector = isLast ? '└── ' : '├── ';
        return `${prefix}${connector}📄 ${this.#name} (${this.#formatSize()})`;
    }

    *find(name) {
        if (this.#name === name) yield this;
    }

    #formatSize() {
        const units = ['B', 'KB', 'MB', 'GB'];
        let size = this.#size;
        let unitIndex = 0;
        while (size >= 1024 && unitIndex < units.length - 1) {
            size /= 1024;
            unitIndex++;
        }
        return `${size.toFixed(unitIndex ? 2 : 0)} ${units[unitIndex]}`;
    }
}

class DirectoryNode {
    #name;
    #children = [];

    constructor(name) {
        this.#name = name;
    }

    get name() { return this.#name; }

    // Computed property — সব children-এর size recursive ভাবে যোগ
    get size() {
        return this.#children.reduce((sum, child) => sum + child.size, 0);
    }

    add(...nodes) {
        for (const node of nodes) {
            if (this.#children.some(c => c.name === node.name)) {
                throw new Error(`'${node.name}' ইতোমধ্যে বিদ্যমান`);
            }
            this.#children.push(node);
        }
        return this;
    }

    display(prefix = '', isLast = true) {
        const connector = isLast ? '└── ' : '├── ';
        let output = `${prefix}${connector}📁 ${this.#name}/`;
        const childPrefix = prefix + (isLast ? '    ' : '│   ');

        this.#children.forEach((child, i) => {
            const isChildLast = i === this.#children.length - 1;
            output += '\n' + child.display(childPrefix, isChildLast);
        });

        return output;
    }

    // Generator-based recursive find
    *find(name) {
        if (this.#name === name) yield this;
        for (const child of this.#children) {
            yield* child.find(name);
        }
    }

    *[Symbol.iterator]() {
        yield* this.#children;
    }
}

// ব্যবহার
const project = new DirectoryNode('ecommerce-app');
const src = new DirectoryNode('src');
const components = new DirectoryNode('components');
components.add(
    new FileNode('Header.jsx', 2048),
    new FileNode('Footer.jsx', 1536),
);
src.add(components, new FileNode('App.jsx', 4096));
project.add(src, new FileNode('package.json', 1024));

console.log(project.display());
console.log(`মোট: ${project.size} bytes`);

// Generator দিয়ে search
const [result] = [...project.find('Header.jsx')];
console.log(result ? `পাওয়া: ${result.name}` : 'পাওয়া যায়নি');
```

---

### ৩. Menu System — মেনু ও সাবমেনু

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

interface MenuComponent
{
    public function render(int $depth = 0): string;
    public function isActive(): bool;
    public function getUrl(): ?string;
}

final readonly class MenuItem implements MenuComponent
{
    public function __construct(
        private string $label,
        private string $url,
        private bool $active = false,
        private ?string $icon = null,
        private ?string $badge = null,
    ) {}

    public function render(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        $icon = $this->icon ? "{$this->icon} " : '';
        $badge = $this->badge ? " [{$this->badge}]" : '';
        $marker = $this->active ? ' ◀ (সক্রিয়)' : '';
        return "{$indent}- {$icon}{$this->label}{$badge}{$marker}";
    }

    public function isActive(): bool
    {
        return $this->active;
    }

    public function getUrl(): string
    {
        return $this->url;
    }
}

final class SubMenu implements MenuComponent
{
    /** @var list<MenuComponent> */
    private array $items = [];

    public function __construct(
        private readonly string $label,
        private readonly ?string $icon = null,
    ) {}

    public function add(MenuComponent ...$items): self
    {
        array_push($this->items, ...$items);
        return $this;
    }

    public function render(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        $icon = $this->icon ? "{$this->icon} " : '';
        $activeMarker = $this->isActive() ? ' ▼' : ' ▶';
        $output = "{$indent}- {$icon}{$this->label}{$activeMarker}";

        foreach ($this->items as $item) {
            $output .= PHP_EOL . $item->render($depth + 1);
        }

        return $output;
    }

    // যেকোনো child active থাকলে SubMenu-ও active
    public function isActive(): bool
    {
        return array_any($this->items, fn(MenuComponent $item) => $item->isActive());
    }

    public function getUrl(): ?string
    {
        return null;
    }
}

// Daraz-style নেভিগেশন মেনু তৈরি
$mainMenu = new SubMenu('প্রধান মেনু', '☰');

$electronics = new SubMenu('ইলেকট্রনিক্স', '📱');
$electronics->add(
    new MenuItem('মোবাইল ফোন', '/mobile', true, '📞', 'নতুন'),
    new MenuItem('ল্যাপটপ', '/laptop', icon: '💻'),
    new MenuItem('ট্যাবলেট', '/tablet', icon: '📟'),
);

$fashion = new SubMenu('ফ্যাশন', '👔');
$fashion->add(
    new MenuItem('পুরুষ', '/men'),
    new MenuItem('মহিলা', '/women'),
    new MenuItem('শিশু', '/kids', badge: '৫০% ছাড়'),
);

$mainMenu->add(
    new MenuItem('হোম', '/', icon: '🏠'),
    $electronics,
    $fashion,
    new MenuItem('আমাদের সম্পর্কে', '/about', icon: 'ℹ️'),
);

echo $mainMenu->render() . PHP_EOL;
```

**JavaScript (ES2022+):**

```javascript
class MenuItem {
    #label; #url; #active; #icon; #badge;

    constructor({ label, url, active = false, icon = null, badge = null }) {
        this.#label = label;
        this.#url = url;
        this.#active = active;
        this.#icon = icon;
        this.#badge = badge;
    }

    get isActive() { return this.#active; }
    get url() { return this.#url; }

    render(depth = 0) {
        const indent = '  '.repeat(depth);
        const icon = this.#icon ? `${this.#icon} ` : '';
        const badge = this.#badge ? ` [${this.#badge}]` : '';
        const marker = this.#active ? ' ◀ (সক্রিয়)' : '';
        return `${indent}- ${icon}${this.#label}${badge}${marker}`;
    }
}

class SubMenu {
    #label; #icon; #items = [];

    constructor({ label, icon = null }) {
        this.#label = label;
        this.#icon = icon;
    }

    add(...items) {
        this.#items.push(...items);
        return this;
    }

    get isActive() {
        return this.#items.some(item => item.isActive);
    }

    get url() { return null; }

    render(depth = 0) {
        const indent = '  '.repeat(depth);
        const icon = this.#icon ? `${this.#icon} ` : '';
        const arrow = this.isActive ? ' ▼' : ' ▶';
        const children = this.#items.map(i => i.render(depth + 1)).join('\n');
        return `${indent}- ${icon}${this.#label}${arrow}\n${children}`;
    }

    *[Symbol.iterator]() { yield* this.#items; }
}

// ব্যবহার
const menu = new SubMenu({ label: 'নেভিগেশন', icon: '☰' });
const products = new SubMenu({ label: 'পণ্যসমূহ', icon: '🛒' });
products.add(
    new MenuItem({ label: 'ফোন', url: '/phone', active: true, icon: '📱' }),
    new MenuItem({ label: 'ল্যাপটপ', url: '/laptop', icon: '💻' }),
);
menu.add(
    new MenuItem({ label: 'হোম', url: '/', icon: '🏠' }),
    products,
);

console.log(menu.render());
```

---

### ৪. Organization Hierarchy — প্রতিষ্ঠানের কাঠামো

**PHP 8.3:**

```php
<?php

declare(strict_types=1);

interface OrganizationUnit
{
    public function getName(): string;
    public function getSalary(): float;
    public function getHeadCount(): int;
    public function display(int $depth = 0): string;
}

final readonly class Employee implements OrganizationUnit
{
    public function __construct(
        private string $name,
        private string $designation,
        private float $salary,
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getSalary(): float
    {
        return $this->salary;
    }

    public function getHeadCount(): int
    {
        return 1;
    }

    public function display(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        $salaryFormatted = number_format($this->salary);
        return "{$indent}👤 {$this->name} ({$this->designation}) - ৳{$salaryFormatted}";
    }
}

final class Department implements OrganizationUnit
{
    /** @var list<OrganizationUnit> */
    private array $units = [];

    public function __construct(
        private readonly string $name,
        private readonly ?string $head = null,
    ) {}

    public function add(OrganizationUnit ...$units): self
    {
        array_push($this->units, ...$units);
        return $this;
    }

    public function getName(): string
    {
        return $this->name;
    }

    // মোট বেতন — সব unit-এর recursive যোগফল
    public function getSalary(): float
    {
        return array_reduce(
            $this->units,
            fn(float $total, OrganizationUnit $unit) => $total + $unit->getSalary(),
            0.0
        );
    }

    // মোট কর্মী সংখ্যা
    public function getHeadCount(): int
    {
        return array_reduce(
            $this->units,
            fn(int $total, OrganizationUnit $unit) => $total + $unit->getHeadCount(),
            0
        );
    }

    public function display(int $depth = 0): string
    {
        $indent = str_repeat('  ', $depth);
        $headInfo = $this->head ? " (প্রধান: {$this->head})" : '';
        $salaryFormatted = number_format($this->getSalary());
        $output = "{$indent}🏢 {$this->name}{$headInfo}";
        $output .= " [কর্মী: {$this->getHeadCount()}, বেতন: ৳{$salaryFormatted}]";

        foreach ($this->units as $unit) {
            $output .= PHP_EOL . $unit->display($depth + 1);
        }

        return $output;
    }
}

// বাংলাদেশি সফটওয়্যার কোম্পানির কাঠামো
$company = new Department('TechBangla Ltd.', 'CEO সাহেব');

$engineering = new Department('ইঞ্জিনিয়ারিং', 'CTO করিম');
$backend = new Department('ব্যাকএন্ড টিম');
$backend->add(
    new Employee('আরিফ', 'Senior Developer', 120_000),
    new Employee('সুমাইয়া', 'Mid Developer', 80_000),
    new Employee('তানভীর', 'Junior Developer', 45_000),
);

$frontend = new Department('ফ্রন্টএন্ড টিম');
$frontend->add(
    new Employee('নাফিসা', 'Lead Developer', 110_000),
    new Employee('রাকিব', 'Mid Developer', 75_000),
);

$engineering->add($backend, $frontend);

$hr = new Department('মানবসম্পদ');
$hr->add(
    new Employee('ফারহানা', 'HR Manager', 90_000),
    new Employee('শাকিল', 'HR Executive', 50_000),
);

$company->add($engineering, $hr);

echo $company->display() . PHP_EOL;
echo PHP_EOL . "মোট কর্মী: {$company->getHeadCount()} জন" . PHP_EOL;
echo "মোট মাসিক বেতন: ৳" . number_format($company->getSalary()) . PHP_EOL;
```

---

## 🌍 Real-World Applicable Areas

### ১. UI Component Trees (React/Vue)

React-এর পুরো architecture-ই Composite প্যাটার্নের উপর গড়া। প্রতিটি component হয় একটি leaf (যেমন `<button>`) অথবা composite (যেমন `<Form>` যার ভিতরে অনেক child component)।

```javascript
// React-এ Composite প্যাটার্ন স্বাভাবিকভাবেই কাজ করে
// প্রতিটি JSX element হলো React.createElement() এর composite tree

class FormField {
    #name; #value;
    constructor(name, value = '') {
        this.#name = name;
        this.#value = value;
    }
    validate() { return this.#value.length > 0; }
    getData() { return { [this.#name]: this.#value }; }
    setValue(v) { this.#value = v; return this; }
}

class FormGroup {
    #name; #fields = [];
    constructor(name) { this.#name = name; }

    add(...fields) {
        this.#fields.push(...fields);
        return this;
    }

    // সব field valid হলেই group valid
    validate() {
        return this.#fields.every(f => f.validate());
    }

    // সব field-এর data merge করে
    getData() {
        return this.#fields.reduce(
            (data, field) => ({ ...data, ...field.getData() }),
            {}
        );
    }
}

// ব্যবহার
const registrationForm = new FormGroup('registration');
const personalInfo = new FormGroup('personal');
personalInfo.add(
    new FormField('name').setValue('রহিম'),
    new FormField('email').setValue('rahim@example.com'),
);
const addressInfo = new FormGroup('address');
addressInfo.add(
    new FormField('city').setValue('ঢাকা'),
    new FormField('area').setValue('মিরপুর'),
);
registrationForm.add(personalInfo, addressInfo);

console.log('Valid:', registrationForm.validate());
console.log('Data:', registrationForm.getData());
```

### ২. E-Commerce Category Tree — দাম হিসাব সহ

**PHP 8.3 — পূর্ণাঙ্গ উদাহরণ:**

```php
<?php

declare(strict_types=1);

interface CatalogItem
{
    public function getName(): string;
    public function getTotalPrice(): float;
    public function getItemCount(): int;
    public function toArray(): array;
}

final readonly class Product implements CatalogItem
{
    public function __construct(
        private string $name,
        private float $price,
        private int $stock,
        private ?float $discount = null, // শতাংশে
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getTotalPrice(): float
    {
        $effectivePrice = $this->discount
            ? $this->price * (1 - $this->discount / 100)
            : $this->price;
        return $effectivePrice * $this->stock;
    }

    public function getItemCount(): int
    {
        return $this->stock > 0 ? 1 : 0;
    }

    public function toArray(): array
    {
        return [
            'type'     => 'product',
            'name'     => $this->name,
            'price'    => $this->price,
            'stock'    => $this->stock,
            'discount' => $this->discount,
            'total'    => $this->getTotalPrice(),
        ];
    }
}

final class Category implements CatalogItem
{
    /** @var list<CatalogItem> */
    private array $children = [];

    public function __construct(
        private readonly string $name,
        private readonly ?string $slug = null,
    ) {}

    public function add(CatalogItem ...$items): self
    {
        array_push($this->children, ...$items);
        return $this;
    }

    public function getName(): string
    {
        return $this->name;
    }

    // Recursive মোট দাম — সব sub-category ও product-এর যোগফল
    public function getTotalPrice(): float
    {
        return array_reduce(
            $this->children,
            fn(float $sum, CatalogItem $item) => $sum + $item->getTotalPrice(),
            0.0
        );
    }

    public function getItemCount(): int
    {
        return array_reduce(
            $this->children,
            fn(int $count, CatalogItem $item) => $count + $item->getItemCount(),
            0
        );
    }

    public function toArray(): array
    {
        return [
            'type'       => 'category',
            'name'       => $this->name,
            'slug'       => $this->slug,
            'itemCount'  => $this->getItemCount(),
            'totalValue' => $this->getTotalPrice(),
            'children'   => array_map(fn(CatalogItem $c) => $c->toArray(), $this->children),
        ];
    }
}

// Daraz-style ক্যাটালগ তৈরি
$root = new Category('সকল পণ্য', 'all');

$electronics = new Category('ইলেকট্রনিক্স', 'electronics');
$mobiles = new Category('মোবাইল ফোন', 'mobiles');
$mobiles->add(
    new Product('Samsung Galaxy S24', 119_999, 50, discount: 10),
    new Product('iPhone 15', 154_999, 30),
    new Product('Xiaomi Redmi Note 13', 24_999, 200, discount: 15),
);
$electronics->add($mobiles);
$electronics->add(new Product('Sony WH-1000XM5', 34_999, 25));

$fashion = new Category('ফ্যাশন', 'fashion');
$fashion->add(
    new Product('পাঞ্জাবি (প্রিমিয়াম)', 2_500, 100),
    new Product('শাড়ি (জামদানি)', 8_500, 40, discount: 20),
);

$root->add($electronics, $fashion);

echo "মোট পণ্য সংখ্যা: {$root->getItemCount()}" . PHP_EOL;
echo "মোট ইনভেন্টরি মূল্য: ৳" . number_format($root->getTotalPrice(), 2) . PHP_EOL;
echo PHP_EOL . json_encode($root->toArray(), JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
```

### ৩. Permission/Role Hierarchies

```php
<?php

interface Permission
{
    public function getName(): string;
    public function hasAccess(string $resource): bool;
}

final readonly class SinglePermission implements Permission
{
    public function __construct(
        private string $name,
        private string $resource,
        private bool $granted,
    ) {}

    public function getName(): string { return $this->name; }

    public function hasAccess(string $resource): bool
    {
        return $this->resource === $resource && $this->granted;
    }
}

final class PermissionGroup implements Permission
{
    /** @var list<Permission> */
    private array $permissions = [];

    public function __construct(private readonly string $name) {}

    public function add(Permission ...$perms): self
    {
        array_push($this->permissions, ...$perms);
        return $this;
    }

    public function getName(): string { return $this->name; }

    // যেকোনো child permission-এ access থাকলেই group-এ access আছে
    public function hasAccess(string $resource): bool
    {
        return array_any(
            $this->permissions,
            fn(Permission $p) => $p->hasAccess($resource)
        );
    }
}
```

### ৪. Laravel Blade Component Nesting

Laravel-এর Blade component system-ও Composite প্যাটার্ন follow করে। `<x-layout>` component-এর ভিতরে `<x-card>`, তার ভিতরে `<x-button>` — সবাই একই `render()` interface মানে।

---

## 🔥 Advanced Deep Dive

### Transparent vs Safe Composite

Composite প্যাটার্নের দুটি variation আছে:

**Transparent Composite:** `add()`, `remove()`, `getChild()` — এগুলো Component interface-এই declare করা হয়। ক্লায়েন্ট সব element-কে একইভাবে treat করতে পারে, কিন্তু Leaf-এ `add()` call করলে runtime error হয়।

**Safe Composite:** `add()`, `remove()` শুধু Composite class-এ থাকে। Type-safe কিন্তু ক্লায়েন্টকে Composite কিনা check করতে হয়।

```php
<?php

// Transparent Approach — Component interface-এ সব method
interface TransparentComponent
{
    public function operation(): string;
    public function add(TransparentComponent $child): void;
    public function remove(TransparentComponent $child): void;
    public function isComposite(): bool;
}

// Leaf-কে add() call করলে exception
final class TransparentLeaf implements TransparentComponent
{
    public function __construct(private readonly string $name) {}

    public function operation(): string { return $this->name; }

    public function add(TransparentComponent $child): void
    {
        throw new \LogicException("Leaf-এ child যোগ করা যায় না");
    }

    public function remove(TransparentComponent $child): void
    {
        throw new \LogicException("Leaf থেকে child সরানো যায় না");
    }

    public function isComposite(): bool { return false; }
}

// Safe Approach — শুধু Composite-এ child management
interface SafeComponent
{
    public function operation(): string;
}

final readonly class SafeLeaf implements SafeComponent
{
    public function __construct(private string $name) {}
    public function operation(): string { return $this->name; }
}

final class SafeComposite implements SafeComponent
{
    /** @var list<SafeComponent> */
    private array $children = [];

    // add/remove শুধু Composite-এই আছে — type-safe
    public function add(SafeComponent $child): void
    {
        $this->children[] = $child;
    }

    public function operation(): string
    {
        return implode(', ', array_map(
            fn(SafeComponent $c) => $c->operation(),
            $this->children
        ));
    }
}
```

> **সুপারিশ:** বেশিরভাগ ক্ষেত্রে **Safe Composite** ব্যবহার করুন। PHP ও TypeScript-এর মতো strongly-typed ভাষায় এটি অনেক বেশি নিরাপদ।

---

### Composite + Iterator — Tree Traversal (DFS ও BFS)

```javascript
class TreeNode {
    #value; #children;

    constructor(value, children = []) {
        this.#value = value;
        this.#children = children;
    }

    get value() { return this.#value; }
    get children() { return [...this.#children]; }

    add(...nodes) {
        this.#children.push(...nodes);
        return this;
    }

    // DFS — Depth-First Search (গভীরতা-প্রথম অনুসন্ধান)
    *dfs() {
        yield this;
        for (const child of this.#children) {
            yield* child.dfs();
        }
    }

    // BFS — Breadth-First Search (প্রস্থ-প্রথম অনুসন্ধান)
    *bfs() {
        const queue = [this];
        while (queue.length > 0) {
            const node = queue.shift();
            yield node;
            queue.push(...node.children);
        }
    }

    // Post-order DFS — children আগে, তারপর parent
    *postOrder() {
        for (const child of this.#children) {
            yield* child.postOrder();
        }
        yield this;
    }
}

// ব্যবহার — বাংলাদেশের প্রশাসনিক কাঠামো
const bangladesh = new TreeNode('বাংলাদেশ');
const dhaka = new TreeNode('ঢাকা বিভাগ');
const chittagong = new TreeNode('চট্টগ্রাম বিভাগ');

dhaka.add(new TreeNode('ঢাকা জেলা'), new TreeNode('গাজীপুর জেলা'));
chittagong.add(new TreeNode('চট্টগ্রাম জেলা'), new TreeNode("কক্সবাজার জেলা"));
bangladesh.add(dhaka, chittagong);

console.log('--- DFS (গভীরতা-প্রথম) ---');
for (const node of bangladesh.dfs()) {
    console.log(node.value);
}

console.log('\n--- BFS (প্রস্থ-প্রথম) ---');
for (const node of bangladesh.bfs()) {
    console.log(node.value);
}
```

---

### Composite + Visitor প্যাটার্ন

```php
<?php

interface Visitor
{
    public function visitFile(File $file): void;
    public function visitDirectory(Directory $dir): void;
}

// Composite element-গুলো visitor গ্রহণ করে
interface Visitable
{
    public function accept(Visitor $visitor): void;
}

// SizeCalculator Visitor — সাইজ হিসাব করে
final class SizeCalculatorVisitor implements Visitor
{
    private int $totalSize = 0;

    public function visitFile(File $file): void
    {
        $this->totalSize += $file->getSize();
    }

    public function visitDirectory(Directory $dir): void
    {
        // Directory নিজে কোনো size যোগ করে না,
        // কিন্তু children-দের visit করায়
    }

    public function getTotalSize(): int
    {
        return $this->totalSize;
    }
}

// SearchVisitor — নির্দিষ্ট extension-এর ফাইল খুঁজে বের করে
final class ExtensionSearchVisitor implements Visitor
{
    /** @var list<string> */
    private array $results = [];

    public function __construct(
        private readonly string $targetExtension,
    ) {}

    public function visitFile(File $file): void
    {
        if ($file->getExtension() === $this->targetExtension) {
            $this->results[] = $file->getName();
        }
    }

    public function visitDirectory(Directory $dir): void {}

    /** @return list<string> */
    public function getResults(): array
    {
        return $this->results;
    }
}
```

---

### Caching in Composite — Memoized Calculations

বড় ট্রি-তে প্রতিবার recursive calculation ব্যয়বহুল। Cache ব্যবহার করে performance উন্নত করা যায়:

```php
<?php

final class CachedDirectory implements FileSystemNode
{
    /** @var list<FileSystemNode> */
    private array $children = [];
    private ?int $cachedSize = null;
    private ?int $cachedCount = null;

    public function __construct(private readonly string $name) {}

    public function add(FileSystemNode $node): self
    {
        $this->children[] = $node;
        $this->invalidateCache(); // নতুন child যোগ হলে cache মুছে ফেলো
        return $this;
    }

    public function remove(FileSystemNode $node): self
    {
        $this->children = array_filter($this->children, fn($c) => $c !== $node);
        $this->invalidateCache();
        return $this;
    }

    public function getSize(): int
    {
        // Cache-এ থাকলে সরাসরি return করো — recalculate করতে হবে না
        if ($this->cachedSize !== null) {
            return $this->cachedSize;
        }

        $this->cachedSize = array_reduce(
            $this->children,
            fn(int $sum, FileSystemNode $n) => $sum + $n->getSize(),
            0
        );

        return $this->cachedSize;
    }

    private function invalidateCache(): void
    {
        $this->cachedSize = null;
        $this->cachedCount = null;
    }

    // ... বাকি methods
}
```

```javascript
// JavaScript — WeakMap দিয়ে external cache
const sizeCache = new WeakMap();

class CachedDirectory {
    #name;
    #children = [];

    constructor(name) {
        this.#name = name;
    }

    add(node) {
        this.#children.push(node);
        sizeCache.delete(this); // cache invalidate
        return this;
    }

    get size() {
        if (sizeCache.has(this)) {
            return sizeCache.get(this);
        }
        const total = this.#children.reduce((sum, c) => sum + c.size, 0);
        sizeCache.set(this, total);
        return total;
    }
}
```

---

### Composite vs Decorator

| বিষয় | Composite | Decorator |
|-------|-----------|-----------|
| **উদ্দেশ্য** | Part-whole hierarchy তৈরি | অবজেক্টে নতুন behavior যোগ |
| **Structure** | Tree (এক parent, অনেক child) | Chain (wrapper-এর পর wrapper) |
| **Children** | একাধিক child থাকে | ঠিক একটি wrapped object |
| **Focus** | Uniform treatment of parts | Dynamic responsibility addition |
| **পরিবর্তন** | ট্রি structure পরিবর্তন | Behavior পরিবর্তন/বৃদ্ধি |

> দুটো প্যাটার্ন একসাথে ব্যবহার করা যায়! Decorator দিয়ে Composite tree-র node-এ নতুন feature যোগ করা সম্ভব — যেমন logging, caching, validation ইত্যাদি।

---

### Composite in React/Vue Architecture

```javascript
// React-এর component tree মূলত Composite প্যাটার্ন
// প্রতিটি component-এর render() method আছে
// Composite component children render করে
// Leaf component নিজের markup render করে

// Vue.js-এও একই concept:
// <template>
//   <div>                     ← Composite
//     <Header />              ← Composite (nested)
//       <Logo />              ← Leaf
//       <Navigation />        ← Composite
//         <NavItem />         ← Leaf
//     <Content />             ← Composite
//       <Article />           ← Leaf
//     <Footer />              ← Leaf
//   </div>
// </template>

// Virtual DOM নিজেই একটি Composite tree
class VNode {
    #tag; #props; #children;

    constructor(tag, props = {}, children = []) {
        this.#tag = tag;
        this.#props = props;
        this.#children = children;
    }

    render() {
        const childHtml = this.#children
            .map(c => typeof c === 'string' ? c : c.render())
            .join('');
        const attrs = Object.entries(this.#props)
            .map(([k, v]) => `${k}="${v}"`)
            .join(' ');
        return `<${this.#tag}${attrs ? ' ' + attrs : ''}>${childHtml}</${this.#tag}>`;
    }
}

// ব্যবহার — Virtual DOM tree তৈরি
const vdom = new VNode('div', { class: 'app' }, [
    new VNode('h1', {}, ['স্বাগতম']),
    new VNode('ul', {}, [
        new VNode('li', {}, ['প্রথম আইটেম']),
        new VNode('li', {}, ['দ্বিতীয় আইটেম']),
    ]),
]);

console.log(vdom.render());
// <div class="app"><h1>স্বাগতম</h1><ul><li>প্রথম আইটেম</li><li>দ্বিতীয় আইটেম</li></ul></div>
```

---

## ✅ Pros (সুবিধাসমূহ)

1. **Open/Closed Principle** — নতুন component type যোগ করতে existing code পরিবর্তন করতে হয় না
2. **Uniform Interface** — ক্লায়েন্ট কোড simple থাকে — Leaf ও Composite-কে আলাদাভাবে handle করতে হয় না
3. **Recursive Composition** — অসীম গভীরতার nested structure সহজে তৈরি করা যায়
4. **Single Responsibility** — প্রতিটি class তার নিজের কাজ করে
5. **Polymorphism** — ক্লায়েন্ট কোড `Component` interface-এর সাথে কাজ করে, concrete class জানতে হয় না

## ❌ Cons (অসুবিধাসমূহ)

1. **Overly General Interface** — সব component-কে একই interface দিলে কিছু method Leaf-এ অর্থহীন হয়ে যায়
2. **Type Safety হ্রাস** — Transparent approach-এ runtime error-এর ঝুঁকি থাকে
3. **Complexity** — ছোট বা flat structure-এর জন্য অতিরিক্ত complexity যোগ করে
4. **Debugging কঠিন** — গভীর nested tree-তে সমস্যা খুঁজে পাওয়া কঠিন হতে পারে
5. **Performance** — খুব বড় ট্রি-তে recursive operation slow হতে পারে (caching ছাড়া)
6. **Constraint প্রয়োগ কঠিন** — নির্দিষ্ট ধরনের child restrict করা Composite-এর interface দিয়ে কঠিন

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ১. Type Checking করা — সবচেয়ে বড় ভুল

```php
// ❌ ভুল — instanceof দিয়ে check করা Composite-এর উদ্দেশ্য নষ্ট করে
function processNode(FileSystemNode $node): void
{
    if ($node instanceof File) {
        echo "ফাইল প্রসেস: " . $node->getName();
    } elseif ($node instanceof Directory) {
        echo "ডিরেক্টরি প্রসেস: " . $node->getName();
        foreach ($node->getChildren() as $child) {
            processNode($child); // recursive, কিন্তু polymorphism নষ্ট
        }
    }
}

// ✅ সঠিক — polymorphism ব্যবহার করুন
function processNode(FileSystemNode $node): void
{
    // uniform treatment — node নিজেই জানে কীভাবে process হতে হবে
    echo $node->display();
}
```

### ২. Overly General Interface

```php
// ❌ ভুল — Leaf-এ অর্থহীন method রাখা
interface Component
{
    public function operation(): string;
    public function add(Component $c): void;     // Leaf-এ এর কোনো মানে নেই
    public function remove(Component $c): void;  // Leaf-এ এর কোনো মানে নেই
    public function getChild(int $i): Component; // Leaf-এ এর কোনো মানে নেই
}

// ✅ সঠিক — Safe Composite approach ব্যবহার করুন
interface Component
{
    public function operation(): string; // শুধু common method
}
```

### ৩. Circular Reference তৈরি করা

```javascript
// ❌ বিপজ্জনক — নিজেকে নিজের child হিসেবে যোগ করা
const dir = new DirectoryNode('test');
dir.add(dir); // infinite recursion! getSize() কখনো শেষ হবে না

// ✅ সুরক্ষা — add()-এ circular reference check যোগ করুন
add(node) {
    if (node === this) {
        throw new Error('নিজেকে নিজের child করা যাবে না');
    }
    if (this.#isDescendant(node)) {
        throw new Error('Circular reference সনাক্ত হয়েছে');
    }
    this.#children.push(node);
}
```

### ৪. Parent Reference না রাখা

কিছু ক্ষেত্রে child থেকে parent-এ navigate করতে হয়। সেক্ষেত্রে parent reference রাখা উচিত, তবে memory leak-এর ব্যাপারে সতর্ক থাকতে হবে।

---

## 🧪 টেস্টিং

### PHPUnit Tests:

```php
<?php

declare(strict_types=1);

namespace Tests\Composite;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

final class FileSystemTest extends TestCase
{
    #[Test]
    public function leaf_returns_its_own_size(): void
    {
        $file = new File('test.txt', 1024, 'txt');

        $this->assertSame(1024, $file->getSize());
        $this->assertSame('test.txt', $file->getName());
    }

    #[Test]
    public function empty_directory_has_zero_size(): void
    {
        $dir = new Directory('empty');

        $this->assertSame(0, $dir->getSize());
        $this->assertSame(0, $dir->getChildCount());
    }

    #[Test]
    public function directory_calculates_total_size_recursively(): void
    {
        $root = new Directory('root');
        $sub = new Directory('sub');
        $sub->add(new File('a.txt', 100, 'txt'), new File('b.txt', 200, 'txt'));
        $root->add($sub, new File('c.txt', 300, 'txt'));

        // 100 + 200 + 300 = 600
        $this->assertSame(600, $root->getSize());
    }

    #[Test]
    public function find_returns_node_by_name(): void
    {
        $root = new Directory('root');
        $file = new File('target.txt', 50, 'txt');
        $sub = new Directory('sub');
        $sub->add($file);
        $root->add($sub);

        $found = $root->find('target.txt');

        $this->assertNotNull($found);
        $this->assertSame('target.txt', $found->getName());
    }

    #[Test]
    public function find_returns_null_when_not_found(): void
    {
        $root = new Directory('root');
        $root->add(new File('exists.txt', 10, 'txt'));

        $this->assertNull($root->find('missing.txt'));
    }

    #[Test]
    public function duplicate_name_throws_exception(): void
    {
        $dir = new Directory('test');
        $dir->add(new File('same.txt', 10, 'txt'));

        $this->expectException(\InvalidArgumentException::class);
        $dir->add(new File('same.txt', 20, 'txt'));
    }

    #[Test]
    public function deeply_nested_structure_calculates_correctly(): void
    {
        // ৫ স্তর গভীর nested structure
        $level1 = new Directory('L1');
        $level2 = new Directory('L2');
        $level3 = new Directory('L3');
        $level4 = new Directory('L4');
        $level5 = new Directory('L5');

        $level5->add(new File('deep.txt', 1, 'txt'));
        $level4->add($level5);
        $level3->add($level4, new File('mid.txt', 2, 'txt'));
        $level2->add($level3);
        $level1->add($level2, new File('top.txt', 4, 'txt'));

        $this->assertSame(7, $level1->getSize()); // 1 + 2 + 4
    }

    #[Test]
    public function composite_treats_leaf_and_composite_uniformly(): void
    {
        $root = new Directory('root');
        $file = new File('file.txt', 100, 'txt');
        $dir = new Directory('dir');
        $dir->add(new File('inner.txt', 50, 'txt'));
        $root->add($file, $dir);

        // দুটোই FileSystemNode — uniform treatment
        $nodes = [$file, $dir];
        foreach ($nodes as $node) {
            $this->assertIsInt($node->getSize());
            $this->assertIsString($node->getName());
            $this->assertIsString($node->display());
        }
    }
}

final class OrganizationTest extends TestCase
{
    #[Test]
    public function department_salary_is_sum_of_all_employees(): void
    {
        $dept = new Department('ইঞ্জিনিয়ারিং');
        $dept->add(
            new Employee('আরিফ', 'Dev', 100_000),
            new Employee('সুমা', 'Dev', 80_000),
        );

        $this->assertSame(180_000.0, $dept->getSalary());
        $this->assertSame(2, $dept->getHeadCount());
    }

    #[Test]
    public function nested_departments_aggregate_correctly(): void
    {
        $company = new Department('কোম্পানি');
        $eng = new Department('ইঞ্জিনিয়ারিং');
        $eng->add(new Employee('আরিফ', 'Dev', 100_000));
        $hr = new Department('HR');
        $hr->add(new Employee('ফারহানা', 'Manager', 90_000));
        $company->add($eng, $hr);

        $this->assertSame(190_000.0, $company->getSalary());
        $this->assertSame(2, $company->getHeadCount());
    }
}
```

### Jest Tests:

```javascript
import { describe, it, expect } from '@jest/globals';

describe('FileSystem Composite', () => {
    it('leaf node returns its own size', () => {
        const file = new FileNode('test.txt', 1024);
        expect(file.size).toBe(1024);
        expect(file.name).toBe('test.txt');
    });

    it('empty directory has zero size', () => {
        const dir = new DirectoryNode('empty');
        expect(dir.size).toBe(0);
    });

    it('directory recursively calculates total size', () => {
        const root = new DirectoryNode('root');
        const sub = new DirectoryNode('sub');
        sub.add(new FileNode('a.txt', 100), new FileNode('b.txt', 200));
        root.add(sub, new FileNode('c.txt', 300));

        expect(root.size).toBe(600);
    });

    it('find returns matching node via generator', () => {
        const root = new DirectoryNode('root');
        const target = new FileNode('target.txt', 50);
        const sub = new DirectoryNode('sub');
        sub.add(target);
        root.add(sub);

        const [found] = [...root.find('target.txt')];
        expect(found).toBe(target);
    });

    it('find yields nothing when not found', () => {
        const root = new DirectoryNode('root');
        root.add(new FileNode('other.txt', 10));

        const results = [...root.find('missing.txt')];
        expect(results).toHaveLength(0);
    });

    it('throws on duplicate names', () => {
        const dir = new DirectoryNode('test');
        dir.add(new FileNode('same.txt', 10));

        expect(() => dir.add(new FileNode('same.txt', 20)))
            .toThrow('ইতোমধ্যে বিদ্যমান');
    });

    it('supports iteration via Symbol.iterator', () => {
        const dir = new DirectoryNode('root');
        const f1 = new FileNode('a.txt', 10);
        const f2 = new FileNode('b.txt', 20);
        dir.add(f1, f2);

        const children = [...dir];
        expect(children).toEqual([f1, f2]);
    });

    it('DFS traversal visits nodes depth-first', () => {
        const root = new TreeNode('A');
        const b = new TreeNode('B');
        const c = new TreeNode('C');
        b.add(new TreeNode('D'), new TreeNode('E'));
        root.add(b, c);

        const order = [...root.dfs()].map(n => n.value);
        expect(order).toEqual(['A', 'B', 'D', 'E', 'C']);
    });

    it('BFS traversal visits nodes breadth-first', () => {
        const root = new TreeNode('A');
        const b = new TreeNode('B');
        const c = new TreeNode('C');
        b.add(new TreeNode('D'), new TreeNode('E'));
        root.add(b, c);

        const order = [...root.bfs()].map(n => n.value);
        expect(order).toEqual(['A', 'B', 'C', 'D', 'E']);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্নসমূহ

| প্যাটার্ন | সম্পর্ক | একসাথে ব্যবহার |
|-----------|--------|----------------|
| **Decorator** | Composite-র node-এ dynamic behavior যোগ করে | Composite tree-র node-কে decorate করুন (logging, caching) |
| **Iterator** | Composite tree traverse করতে Iterator ব্যবহার হয় | DFS, BFS traversal implement করুন |
| **Visitor** | Composite tree-র element-এ operation define করে — element পরিবর্তন ছাড়াই | নতুন operation যোগ করতে Visitor ব্যবহার করুন |
| **Chain of Responsibility** | দুটোই recursive structure ব্যবহার করে | Request-কে parent পর্যন্ত propagate করুন |
| **Flyweight** | Composite tree-তে অনেক similar leaf থাকলে Flyweight দিয়ে memory বাঁচানো যায় | অনেক shared leaf-এর জন্য Flyweight ব্যবহার করুন |
| **Builder** | জটিল Composite tree তৈরিতে Builder ব্যবহার করা যায় | Step-by-step tree construction |

### Composite + Chain of Responsibility উদাহরণ:

```php
// Event propagation — child থেকে parent পর্যন্ত
// DOM-এর event bubbling ঠিক এভাবে কাজ করে
interface UIComponent
{
    public function handleEvent(string $event): bool;
    public function setParent(?UIComponent $parent): void;
}
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

1. **Part-whole hierarchy আছে** — ট্রি structure-এ অবজেক্ট সাজাতে হবে
2. **Uniform treatment দরকার** — ক্লায়েন্ট কোডকে leaf ও composite আলাদা করতে হবে না
3. **Recursive structure আছে** — ফাইল সিস্টেম, মেনু, ক্যাটাগরি, org chart
4. **নতুন component type সহজে যোগ করতে চান** — OCP মেনে
5. **Tree-wide operation দরকার** — পুরো ট্রি-তে aggregate calculation (total size, total price)

### ❌ ব্যবহার করবেন না যখন:

1. **Flat structure** — কোনো nesting নেই, সব element এক স্তরে
2. **Leaf ও Composite-এর interface অনেক আলাদা** — uniform interface তৈরি করা কঠিন
3. **Performance critical** — লক্ষ লক্ষ node-এর ট্রি-তে recursive operation ধীর হতে পারে
4. **Simple collection** — শুধু একটি list বা array-ই যথেষ্ট, ট্রি দরকার নেই
5. **Type-specific behavior দরকার** — প্রতিটি element-এর আলাদা treatment প্রয়োজন হলে Composite সঠিক নয়

### সিদ্ধান্ত গাছ (Decision Tree):

```
আপনার কি tree/hierarchical structure আছে?
├── না → Composite লাগবে না
└── হ্যাঁ → Leaf ও Container-কে একইভাবে treat করতে চান?
    ├── না → সাধারণ tree data structure ব্যবহার করুন
    └── হ্যাঁ → Composite প্যাটার্ন ব্যবহার করুন!
        ├── Type safety গুরুত্বপূর্ণ? → Safe Composite
        └── Maximum transparency চান? → Transparent Composite
```

---

## 📋 সারসংক্ষেপ

Composite প্যাটার্ন হলো **tree structure**-এ অবজেক্ট সাজানোর এবং **individual ও composite** অবজেক্টকে **uniformly** treat করার একটি শক্তিশালী কৌশল।

### মূল বিষয়গুলো:

- **Component** — common interface (PHP interface / JS abstract class)
- **Leaf** — ট্রি-র শেষ প্রান্ত, কোনো child নেই
- **Composite** — container, অন্যান্য Component ধারণ করে
- **Recursive** — Composite-এর operation children-দের operation call করে
- **Transparency vs Safety** — দুটি approach, প্রয়োজন অনুসারে বেছে নিন

### মনে রাখার সূত্র:

> 🌳 "ফাইল আর ফোল্ডার — দুটোই `ls` কমান্ডে দেখা যায়। ফোল্ডার শুধু অন্যান্য আইটেম ধারণ করে — এটাই Composite।"

### চেকলিস্ট:

- [ ] Common `Component` interface তৈরি করেছেন?
- [ ] `Leaf` class সরাসরি operation implement করে?
- [ ] `Composite` class children-দের কাছে delegate করে?
- [ ] Circular reference protection আছে?
- [ ] বড় ট্রি-তে caching ব্যবহার করছেন?
- [ ] Uniform treatment নিশ্চিত — type checking এড়িয়ে চলছেন?
- [ ] Safe vs Transparent approach সচেতনভাবে বেছে নিয়েছেন?

> **"যেখানে গাছ আছে, সেখানে Composite আছে।"** — ফাইল সিস্টেম থেকে React component tree, Daraz-এর ক্যাটাগরি থেকে সরকারি কাঠামো — সর্বত্র এই প্যাটার্ন কাজ করে। 🌳
