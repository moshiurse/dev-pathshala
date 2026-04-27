# 🪶 Flyweight প্যাটার্ন

## 📌 সংজ্ঞা

> **GoF Definition:** *"Use sharing to support large numbers of fine-grained objects efficiently."*

Flyweight প্যাটার্ন হলো একটি **Structural Design Pattern** যা বিপুল সংখ্যক ক্ষুদ্র অবজেক্ট দক্ষতার সাথে পরিচালনা করার জন্য **শেয়ারিং** মেকানিজম ব্যবহার করে। যখন আপনার অ্যাপ্লিকেশনে হাজার হাজার বা লক্ষ লক্ষ অবজেক্ট তৈরি হচ্ছে এবং প্রতিটি অবজেক্টের মধ্যে উল্লেখযোগ্য পরিমাণ **সাধারণ ডেটা (common data)** আছে, তখন এই প্যাটার্ন মেমোরি ব্যবহার নাটকীয়ভাবে কমিয়ে আনে।

### মূল ধারণা: Intrinsic vs Extrinsic State

Flyweight প্যাটার্নের **সবচেয়ে গুরুত্বপূর্ণ** ধারণা হলো অবজেক্টের state কে দুই ভাগে বিভক্ত করা:

| ধরন | বৈশিষ্ট্য | উদাহরণ |
|------|-----------|---------|
| **Intrinsic State** (অন্তর্নিহিত) | অবজেক্টের ভেতরে সংরক্ষিত, সব context-এ একই, শেয়ারযোগ্য, immutable | ফন্ট নাম, রঙ, টেক্সচার, আকৃতি |
| **Extrinsic State** (বাহ্যিক) | বাইরে থেকে সরবরাহ করা হয়, context-ভেদে ভিন্ন, শেয়ার করা যায় না | অবস্থান (x, y), সময়, velocity |

**গুরুত্বপূর্ণ উপলব্ধি:** Intrinsic state হলো সেই ডেটা যা **অবজেক্টের পরিচয়** নির্ধারণ করে — এটি পরিবর্তন হলে সেটি সম্পূর্ণ ভিন্ন অবজেক্ট হয়ে যায়। Extrinsic state হলো **ব্যবহারের প্রেক্ষাপট** — একই অবজেক্ট ভিন্ন ভিন্ন প্রেক্ষাপটে ভিন্নভাবে ব্যবহৃত হতে পারে।

ধরুন ঢাকা শহরের ম্যাপ রেন্ডার করছেন — "রাস্তা" টাইল টাইপটি একটাই (intrinsic), কিন্তু ম্যাপের হাজার হাজার স্থানে সেটি বসবে (extrinsic — প্রতিটি স্থানের x,y coordinate)।

---

## 🏠 বাস্তব উদাহরণ: দাবা খেলা (Chess Game)

দাবা খেলায় ৩২টি ঘুঁটি আছে, কিন্তু মাত্র **৬ ধরনের** ঘুঁটি (রাজা, রানী, হাতি, ঘোড়া, নৌকা, সৈন্য) এবং **২টি রঙ** (সাদা, কালো)।

```
প্রচলিত পদ্ধতি (Flyweight ছাড়া):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
৩২টি পৃথক অবজেক্ট, প্রতিটিতে:
  - ঘুঁটির ধরন (king, queen, ...)
  - রঙ (white/black)
  - আইকন/ছবি (ভারী ডেটা!) ← প্রতিটিতে পুনরায় সংরক্ষিত
  - অবস্থান (row, col)

মোট: ৩২ × (ধরন + রঙ + আইকন + অবস্থান)

Flyweight পদ্ধতি:
━━━━━━━━━━━━━━━━━━
১২টি শেয়ার্ড Flyweight (৬ ধরন × ২ রঙ):
  - ঘুঁটির ধরন      ┐
  - রঙ               ├─ Intrinsic (শেয়ার্ড)
  - আইকন/ছবি        ┘

৩২টি হালকা রেফারেন্স:
  - flyweight রেফ     ┐
  - অবস্থান (row, col) ├─ Extrinsic (ইউনিক)
                       ┘

মোট: ১২ × (ধরন + রঙ + আইকন) + ৩২ × (রেফ + অবস্থান)
```

এখানে **আইকন/ছবির ডেটা** সবচেয়ে ভারী অংশ — Flyweight ব্যবহারে সেটি মাত্র ১২ বার সংরক্ষিত হচ্ছে ৩২ বারের বদলে!

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client                                  │
│  (extrinsic state ধারণ করে, flyweight-এ পাঠায়)                  │
└──────────────────┬──────────────────────────────────────────────┘
                   │ uses
                   ▼
┌──────────────────────────────┐
│      FlyweightFactory        │
│──────────────────────────────│
│ - flyweights: Map<key, Fw>   │
│──────────────────────────────│
│ + getFlyweight(key): Fw      │
│ + listFlyweights(): int      │
└──────────────┬───────────────┘
               │ creates/returns
               ▼
┌──────────────────────────────┐
│    <<interface>> Flyweight    │
│──────────────────────────────│
│ + operation(extrinsicState)  │
└──────────────┬───────────────┘
               │
       ┌───────┴────────┐
       │                 │
       ▼                 ▼
┌──────────────┐  ┌───────────────────┐
│  Concrete    │  │  Unshared         │
│  Flyweight   │  │  ConcreteFlyweight│
│──────────────│  │───────────────────│
│ -intrinsic   │  │ -allState         │
│──────────────│  │───────────────────│
│ +operation() │  │ +operation()      │
└──────────────┘  └───────────────────┘

ডেটা প্রবাহ:
═══════════════
Client ──(extrinsic data)──► FlyweightFactory
                                    │
                              ┌─────┴─────┐
                              │ cache hit? │
                              └─────┬─────┘
                           yes/     \no
                          ▼          ▼
                    return        create new
                    existing      flyweight,
                    flyweight     cache it,
                                  return it
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Flyweight — ভিত্তি কাঠামো

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Flyweight ইন্টারফেস
interface Flyweight
{
    public function operation(array $extrinsicState): string;
}

// Concrete Flyweight — intrinsic state ধারণ করে
readonly class ConcreteFlyweight implements Flyweight
{
    public function __construct(
        private string $sharedData,
        private string $category,
    ) {}

    public function operation(array $extrinsicState): string
    {
        $intrinsic = "({$this->sharedData}, {$this->category})";
        $extrinsic = implode(', ', $extrinsicState);
        return "Flyweight: Intrinsic=[$intrinsic] | Extrinsic=[$extrinsic]";
    }

    public function getKey(): string
    {
        return "{$this->sharedData}_{$this->category}";
    }
}

// Flyweight Factory — ক্যাশিং ও ম্যানেজমেন্ট
class FlyweightFactory
{
    /** @var array<string, ConcreteFlyweight> */
    private array $flyweights = [];

    public function __construct(array $initialData = [])
    {
        foreach ($initialData as $data) {
            $fw = new ConcreteFlyweight($data[0], $data[1]);
            $this->flyweights[$fw->getKey()] = $fw;
        }
    }

    public function getFlyweight(string $sharedData, string $category): ConcreteFlyweight
    {
        $key = "{$sharedData}_{$category}";

        if (!isset($this->flyweights[$key])) {
            echo "Factory: নতুন flyweight তৈরি হচ্ছে — key: {$key}\n";
            $this->flyweights[$key] = new ConcreteFlyweight($sharedData, $category);
        } else {
            echo "Factory: বিদ্যমান flyweight পুনরায় ব্যবহৃত — key: {$key}\n";
        }

        return $this->flyweights[$key];
    }

    public function getCount(): int
    {
        return count($this->flyweights);
    }
}

// ব্যবহার
$factory = new FlyweightFactory([
    ['TypeA', 'Green'],
    ['TypeB', 'Red'],
]);

$fw1 = $factory->getFlyweight('TypeA', 'Green');   // পুনরায় ব্যবহৃত
$fw2 = $factory->getFlyweight('TypeC', 'Blue');     // নতুন তৈরি
$fw3 = $factory->getFlyweight('TypeA', 'Green');    // পুনরায় ব্যবহৃত

echo $fw1->operation(['posX:10', 'posY:20']) . "\n";
echo $fw2->operation(['posX:50', 'posY:60']) . "\n";
echo "মোট Flyweight সংখ্যা: {$factory->getCount()}\n"; // 3
```

#### JavaScript ES2022+

```javascript
// Flyweight class — intrinsic state ধারণ করে
class ConcreteFlyweight {
    #sharedData;
    #category;

    constructor(sharedData, category) {
        this.#sharedData = sharedData;
        this.#category = category;
        // freeze করে immutable বানানো — flyweight অবশ্যই immutable হতে হবে
        Object.freeze(this);
    }

    operation(extrinsicState) {
        const intrinsic = `(${this.#sharedData}, ${this.#category})`;
        const extrinsic = Object.entries(extrinsicState)
            .map(([k, v]) => `${k}:${v}`)
            .join(', ');
        return `Flyweight: Intrinsic=[${intrinsic}] | Extrinsic=[${extrinsic}]`;
    }

    get key() {
        return `${this.#sharedData}_${this.#category}`;
    }
}

// Flyweight Factory
class FlyweightFactory {
    #flyweights = new Map();

    constructor(initialData = []) {
        for (const [sharedData, category] of initialData) {
            const fw = new ConcreteFlyweight(sharedData, category);
            this.#flyweights.set(fw.key, fw);
        }
    }

    getFlyweight(sharedData, category) {
        const key = `${sharedData}_${category}`;

        if (!this.#flyweights.has(key)) {
            console.log(`Factory: নতুন flyweight তৈরি — key: ${key}`);
            this.#flyweights.set(key, new ConcreteFlyweight(sharedData, category));
        } else {
            console.log(`Factory: বিদ্যমান flyweight পুনরায় ব্যবহৃত — key: ${key}`);
        }

        return this.#flyweights.get(key);
    }

    get count() {
        return this.#flyweights.size;
    }
}

// ব্যবহার
const factory = new FlyweightFactory([
    ['TypeA', 'Green'],
    ['TypeB', 'Red'],
]);

const fw1 = factory.getFlyweight('TypeA', 'Green');
const fw2 = factory.getFlyweight('TypeC', 'Blue');
const fw3 = factory.getFlyweight('TypeA', 'Green');

console.log(fw1.operation({ posX: 10, posY: 20 }));
console.log(fw2.operation({ posX: 50, posY: 60 }));
console.log(`মোট Flyweight সংখ্যা: ${factory.count}`); // 3
console.log(`fw1 === fw3: ${fw1 === fw3}`); // true — একই রেফারেন্স!
```

---

### ২. Text Editor — ক্যারেক্টার রেন্ডারিং সিস্টেম

বিশাল ডকুমেন্টে লক্ষ লক্ষ ক্যারেক্টার থাকে। প্রতিটি ক্যারেক্টারের **ফন্ট, সাইজ, রঙ** (intrinsic) একই হতে পারে, কিন্তু **অবস্থান** (extrinsic) অবশ্যই ভিন্ন।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// CharacterStyle — Flyweight (intrinsic state)
readonly class CharacterStyle
{
    public function __construct(
        public string $fontFamily,
        public int    $fontSize,
        public string $color,
        public bool   $bold,
        public bool   $italic,
    ) {}

    public function render(string $char, int $row, int $col): string
    {
        $style = $this->bold ? 'Bold' : 'Normal';
        $style .= $this->italic ? '+Italic' : '';
        return "['{$char}' at ({$row},{$col}) | {$this->fontFamily} {$this->fontSize}px {$this->color} {$style}]";
    }

    public function getKey(): string
    {
        return md5(serialize([
            $this->fontFamily,
            $this->fontSize,
            $this->color,
            $this->bold,
            $this->italic,
        ]));
    }
}

// CharacterStyleFactory
class CharacterStyleFactory
{
    /** @var array<string, CharacterStyle> */
    private array $styles = [];

    public function getStyle(
        string $fontFamily,
        int $fontSize,
        string $color,
        bool $bold = false,
        bool $italic = false,
    ): CharacterStyle {
        $style = new CharacterStyle($fontFamily, $fontSize, $color, $bold, $italic);
        $key = $style->getKey();

        if (!isset($this->styles[$key])) {
            $this->styles[$key] = $style;
        }

        return $this->styles[$key];
    }

    public function getUniqueStyleCount(): int
    {
        return count($this->styles);
    }
}

// Document — প্রতিটি ক্যারেক্টার একটি লাইটওয়েট স্ট্রাকচার
class Document
{
    /** @var array{char: string, style: CharacterStyle, row: int, col: int}[] */
    private array $characters = [];

    public function __construct(
        private readonly CharacterStyleFactory $styleFactory,
    ) {}

    public function addCharacter(
        string $char,
        int $row,
        int $col,
        string $font = 'SolaimanLipi',
        int $size = 14,
        string $color = '#000000',
        bool $bold = false,
        bool $italic = false,
    ): void {
        $style = $this->styleFactory->getStyle($font, $size, $color, $bold, $italic);
        $this->characters[] = [
            'char'  => $char,
            'style' => $style, // রেফারেন্স — শেয়ার্ড flyweight
            'row'   => $row,
            'col'   => $col,
        ];
    }

    public function render(): void
    {
        foreach ($this->characters as $c) {
            echo $c['style']->render($c['char'], $c['row'], $c['col']) . "\n";
        }
    }

    public function getCharCount(): int
    {
        return count($this->characters);
    }
}

// === মেমোরি তুলনা ===
$factory = new CharacterStyleFactory();
$doc = new Document($factory);

// বাংলা টেক্সট যোগ করা — একই স্টাইলের অনেক ক্যারেক্টার
$text = 'বাংলাদেশ আমার প্রিয় মাতৃভূমি';
$col = 0;
foreach (mb_str_split($text) as $char) {
    $doc->addCharacter($char, 0, $col++, 'SolaimanLipi', 14, '#000000');
}

// শিরোনাম — ভিন্ন স্টাইল
$heading = 'স্বাগতম';
$col = 0;
foreach (mb_str_split($heading) as $char) {
    $doc->addCharacter($char, 1, $col++, 'Kalpurush', 24, '#1a73e8', bold: true);
}

echo "মোট ক্যারেক্টার: {$doc->getCharCount()}\n";
echo "ইউনিক স্টাইল (Flyweight): {$factory->getUniqueStyleCount()}\n";
// হাজার ক্যারেক্টার হলেও মাত্র কয়েকটি স্টাইল অবজেক্ট!
```

#### JavaScript ES2022+

```javascript
class CharacterStyle {
    #fontFamily;
    #fontSize;
    #color;
    #bold;
    #italic;

    constructor(fontFamily, fontSize, color, bold = false, italic = false) {
        this.#fontFamily = fontFamily;
        this.#fontSize = fontSize;
        this.#color = color;
        this.#bold = bold;
        this.#italic = italic;
        Object.freeze(this);
    }

    render(char, row, col) {
        const style = `${this.#bold ? 'Bold' : 'Normal'}${this.#italic ? '+Italic' : ''}`;
        return `['${char}' at (${row},${col}) | ${this.#fontFamily} ${this.#fontSize}px ${this.#color} ${style}]`;
    }

    get key() {
        return `${this.#fontFamily}_${this.#fontSize}_${this.#color}_${this.#bold}_${this.#italic}`;
    }
}

class CharacterStyleFactory {
    #styles = new Map();

    getStyle(fontFamily, fontSize, color, bold = false, italic = false) {
        const key = `${fontFamily}_${fontSize}_${color}_${bold}_${italic}`;

        if (!this.#styles.has(key)) {
            this.#styles.set(key, new CharacterStyle(fontFamily, fontSize, color, bold, italic));
        }
        return this.#styles.get(key);
    }

    get uniqueStyleCount() {
        return this.#styles.size;
    }
}

class TextDocument {
    #characters = [];
    #styleFactory;

    constructor(styleFactory) {
        this.#styleFactory = styleFactory;
    }

    addCharacter(char, row, col, font = 'SolaimanLipi', size = 14, color = '#000', bold = false, italic = false) {
        const style = this.#styleFactory.getStyle(font, size, color, bold, italic);
        this.#characters.push({ char, style, row, col });
    }

    render() {
        return this.#characters.map(c => c.style.render(c.char, c.row, c.col));
    }

    get charCount() {
        return this.#characters.length;
    }
}

// === মেমোরি তুলনা সিমুলেশন ===

// Flyweight ছাড়া — প্রতিটি ক্যারেক্টারে পৃথক স্টাইল অবজেক্ট
function withoutFlyweight(charCount) {
    const chars = [];
    for (let i = 0; i < charCount; i++) {
        chars.push({
            char: 'ক',
            font: 'SolaimanLipi',
            size: 14,
            color: '#000',
            bold: false,
            italic: false,
            row: Math.floor(i / 80),
            col: i % 80,
        });
    }
    return chars;
}

// Flyweight সহ
function withFlyweight(charCount) {
    const factory = new CharacterStyleFactory();
    const doc = new TextDocument(factory);
    for (let i = 0; i < charCount; i++) {
        doc.addCharacter('ক', Math.floor(i / 80), i % 80);
    }
    return { doc, factory };
}

const COUNT = 100_000;

console.time('Flyweight ছাড়া');
const without = withoutFlyweight(COUNT);
console.timeEnd('Flyweight ছাড়া');

console.time('Flyweight সহ');
const { doc, factory } = withFlyweight(COUNT);
console.timeEnd('Flyweight সহ');

console.log(`ক্যারেক্টার সংখ্যা: ${COUNT}`);
console.log(`ইউনিক স্টাইল অবজেক্ট (Flyweight): ${factory.uniqueStyleCount}`);
console.log(`Flyweight ছাড়া: ${COUNT}টি পূর্ণ অবজেক্ট (প্রতিটিতে style ডেটা)`);
console.log(`Flyweight সহ: ${factory.uniqueStyleCount}টি style + ${COUNT}টি লাইটওয়েট রেফারেন্স`);
```

---

### ৩. Game Particle System — পার্টিকেল ইফেক্ট

গেমে বিস্ফোরণ, আগুন, ধোঁয়া ইত্যাদি ইফেক্টে হাজার হাজার পার্টিকেল একসাথে রেন্ডার হয়। **টেক্সচার ও রঙ** শেয়ার করা যায়, কিন্তু **অবস্থান ও বেগ** প্রতিটির জন্য ইউনিক।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ParticleType — Flyweight (ভারী ডেটা এখানে — texture, color)
readonly class ParticleType
{
    public function __construct(
        public string $name,
        public string $texture,    // বড় ইমেজ ডেটা সিমুলেশন
        public string $color,
        public float  $baseSize,
        public int    $maxLifespan,
    ) {}

    public function draw(float $x, float $y, float $velocityX, float $velocityY, int $age): string
    {
        $remaining = $this->maxLifespan - $age;
        return sprintf(
            "[%s] pos(%.1f,%.1f) vel(%.1f,%.1f) age:%d/%d | tex:%s col:%s",
            $this->name, $x, $y, $velocityX, $velocityY,
            $age, $this->maxLifespan,
            substr($this->texture, 0, 10) . '...', $this->color
        );
    }

    public function estimateMemoryBytes(): int
    {
        // texture ডেটা সাধারণত কয়েক KB থেকে MB হতে পারে
        return strlen($this->texture) + strlen($this->color) + 64;
    }
}

// ParticleTypeFactory
class ParticleTypeFactory
{
    /** @var array<string, ParticleType> */
    private array $types = [];

    public function getType(
        string $name,
        string $texture,
        string $color,
        float $baseSize,
        int $maxLifespan,
    ): ParticleType {
        $key = $name;

        if (!isset($this->types[$key])) {
            $this->types[$key] = new ParticleType($name, $texture, $color, $baseSize, $maxLifespan);
        }

        return $this->types[$key];
    }

    public function getTypes(): array
    {
        return $this->types;
    }

    public function getTotalIntrinsicMemory(): int
    {
        return array_sum(array_map(
            fn(ParticleType $t) => $t->estimateMemoryBytes(),
            $this->types
        ));
    }
}

// Particle — extrinsic state ধারণ করে, flyweight-এ রেফারেন্স রাখে
class Particle
{
    public function __construct(
        private readonly ParticleType $type,
        private float $x,
        private float $y,
        private float $velocityX,
        private float $velocityY,
        private int $age = 0,
    ) {}

    public function update(): void
    {
        $this->x += $this->velocityX;
        $this->y += $this->velocityY;
        $this->velocityY += 0.1; // gravity
        $this->age++;
    }

    public function isAlive(): bool
    {
        return $this->age < $this->type->maxLifespan;
    }

    public function draw(): string
    {
        return $this->type->draw(
            $this->x, $this->y,
            $this->velocityX, $this->velocityY,
            $this->age
        );
    }
}

// ParticleSystem — পার্টিকেল ম্যানেজার
class ParticleSystem
{
    /** @var Particle[] */
    private array $particles = [];

    public function __construct(
        private readonly ParticleTypeFactory $factory,
    ) {}

    public function emit(string $typeName, float $x, float $y, int $count): void
    {
        $presets = [
            'fire'      => ['texture' => str_repeat('🔥', 500), 'color' => '#FF4500', 'size' => 3.0, 'life' => 60],
            'smoke'     => ['texture' => str_repeat('💨', 500), 'color' => '#888888', 'size' => 5.0, 'life' => 120],
            'spark'     => ['texture' => str_repeat('✨', 500), 'color' => '#FFD700', 'size' => 1.0, 'life' => 30],
            'explosion' => ['texture' => str_repeat('💥', 500), 'color' => '#FF0000', 'size' => 8.0, 'life' => 20],
        ];

        $preset = $presets[$typeName] ?? $presets['spark'];
        $type = $this->factory->getType(
            $typeName, $preset['texture'], $preset['color'],
            $preset['size'], $preset['life']
        );

        for ($i = 0; $i < $count; $i++) {
            $this->particles[] = new Particle(
                type: $type,
                x: $x + (mt_rand(-50, 50) / 10),
                y: $y + (mt_rand(-50, 50) / 10),
                velocityX: (mt_rand(-30, 30) / 10),
                velocityY: (mt_rand(-50, -10) / 10),
            );
        }
    }

    public function update(): void
    {
        foreach ($this->particles as $i => $particle) {
            $particle->update();
            if (!$particle->isAlive()) {
                unset($this->particles[$i]);
            }
        }
        $this->particles = array_values($this->particles);
    }

    public function getParticleCount(): int
    {
        return count($this->particles);
    }

    public function getMemoryReport(): string
    {
        $intrinsicMem = $this->factory->getTotalIntrinsicMemory();
        $extrinsicMem = count($this->particles) * 48; // আনুমানিক প্রতিটি particle-এর extrinsic
        $withoutFlyweight = count($this->particles) * ($intrinsicMem / max(1, count($this->factory->getTypes())) + 48);

        return sprintf(
            "=== মেমোরি রিপোর্ট ===\n" .
            "পার্টিকেল সংখ্যা: %d\n" .
            "ইউনিক টাইপ (Flyweight): %d\n" .
            "Intrinsic মেমোরি (শেয়ার্ড): ~%s\n" .
            "Extrinsic মেমোরি (ইউনিক): ~%s\n" .
            "Flyweight সহ মোট: ~%s\n" .
            "Flyweight ছাড়া মোট: ~%s\n" .
            "সাশ্রয়: ~%.1f%%",
            count($this->particles),
            count($this->factory->getTypes()),
            self::formatBytes($intrinsicMem),
            self::formatBytes($extrinsicMem),
            self::formatBytes($intrinsicMem + $extrinsicMem),
            self::formatBytes((int) $withoutFlyweight),
            (1 - ($intrinsicMem + $extrinsicMem) / max(1, $withoutFlyweight)) * 100
        );
    }

    private static function formatBytes(int $bytes): string
    {
        return match (true) {
            $bytes >= 1_048_576 => round($bytes / 1_048_576, 2) . ' MB',
            $bytes >= 1024      => round($bytes / 1024, 2) . ' KB',
            default             => $bytes . ' B',
        };
    }
}

// ব্যবহার
$factory = new ParticleTypeFactory();
$system = new ParticleSystem($factory);

$system->emit('fire', 100.0, 200.0, 5000);
$system->emit('smoke', 100.0, 200.0, 3000);
$system->emit('spark', 100.0, 200.0, 2000);
$system->emit('explosion', 150.0, 200.0, 1000);

echo $system->getMemoryReport();
```

#### JavaScript ES2022+

```javascript
class ParticleType {
    #name; #texture; #color; #baseSize; #maxLifespan;

    constructor(name, texture, color, baseSize, maxLifespan) {
        this.#name = name;
        this.#texture = texture;
        this.#color = color;
        this.#baseSize = baseSize;
        this.#maxLifespan = maxLifespan;
        Object.freeze(this);
    }

    draw(x, y, vx, vy, age) {
        return `[${this.#name}] pos(${x.toFixed(1)},${y.toFixed(1)}) vel(${vx.toFixed(1)},${vy.toFixed(1)}) age:${age}/${this.#maxLifespan}`;
    }

    get name() { return this.#name; }
    get maxLifespan() { return this.#maxLifespan; }
    get estimatedBytes() { return this.#texture.length * 2 + 64; }
}

class ParticleTypeFactory {
    #types = new Map();

    getType(name, texture, color, baseSize, maxLifespan) {
        if (!this.#types.has(name)) {
            this.#types.set(name, new ParticleType(name, texture, color, baseSize, maxLifespan));
        }
        return this.#types.get(name);
    }

    get typeCount() { return this.#types.size; }

    get totalIntrinsicBytes() {
        let total = 0;
        for (const t of this.#types.values()) total += t.estimatedBytes;
        return total;
    }
}

class Particle {
    constructor(type, x, y, vx, vy) {
        this.type = type;
        this.x = x; this.y = y;
        this.vx = vx; this.vy = vy;
        this.age = 0;
    }

    update() {
        this.x += this.vx;
        this.y += this.vy;
        this.vy += 0.1;
        this.age++;
    }

    get isAlive() { return this.age < this.type.maxLifespan; }
}

class ParticleSystem {
    #particles = [];
    #factory;

    constructor(factory) { this.#factory = factory; }

    emit(typeName, x, y, count) {
        const presets = {
            fire:      { texture: '🔥'.repeat(500), color: '#FF4500', size: 3, life: 60 },
            smoke:     { texture: '💨'.repeat(500), color: '#888',    size: 5, life: 120 },
            spark:     { texture: '✨'.repeat(500), color: '#FFD700', size: 1, life: 30 },
            explosion: { texture: '💥'.repeat(500), color: '#FF0000', size: 8, life: 20 },
        };
        const p = presets[typeName] ?? presets.spark;
        const type = this.#factory.getType(typeName, p.texture, p.color, p.size, p.life);

        for (let i = 0; i < count; i++) {
            this.#particles.push(new Particle(
                type,
                x + (Math.random() - 0.5) * 10,
                y + (Math.random() - 0.5) * 10,
                (Math.random() - 0.5) * 6,
                -(Math.random() * 4 + 1),
            ));
        }
    }

    update() {
        this.#particles = this.#particles.filter(p => {
            p.update();
            return p.isAlive;
        });
    }

    memoryReport() {
        const count = this.#particles.length;
        const intrinsic = this.#factory.totalIntrinsicBytes;
        const extrinsicPerParticle = 48;
        const extrinsic = count * extrinsicPerParticle;
        const withFw = intrinsic + extrinsic;
        const withoutFw = count * (intrinsic / Math.max(1, this.#factory.typeCount) + extrinsicPerParticle);
        const fmt = b => b >= 1_048_576 ? `${(b/1_048_576).toFixed(2)} MB` : `${(b/1024).toFixed(2)} KB`;

        return [
            `=== মেমোরি রিপোর্ট ===`,
            `পার্টিকেল: ${count}`,
            `ইউনিক টাইপ: ${this.#factory.typeCount}`,
            `Flyweight সহ: ~${fmt(withFw)}`,
            `Flyweight ছাড়া: ~${fmt(withoutFw)}`,
            `সাশ্রয়: ~${((1 - withFw / withoutFw) * 100).toFixed(1)}%`,
        ].join('\n');
    }
}

// ব্যবহার
const factory = new ParticleTypeFactory();
const system = new ParticleSystem(factory);

system.emit('fire', 100, 200, 5000);
system.emit('smoke', 100, 200, 3000);
system.emit('spark', 100, 200, 2000);
system.emit('explosion', 150, 200, 1000);

console.log(system.memoryReport());
```

---

### ৪. Tree Rendering — বনের গাছ রেন্ডারিং

একটি গেম বা সিমুলেশনে লক্ষ লক্ষ গাছ রেন্ডার করতে হয়। **TreeType** (নাম, রঙ, টেক্সচার) শেয়ার করা যায়, কিন্তু প্রতিটি গাছের **(x, y)** স্থানাঙ্ক ইউনিক।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// TreeType — Flyweight (intrinsic: নাম, রঙ, টেক্সচার)
readonly class TreeType
{
    public function __construct(
        public string $name,
        public string $color,
        public string $texture, // ভারী ইমেজ ডেটা
    ) {}

    public function draw(float $x, float $y, Canvas $canvas): void
    {
        $canvas->drawAt($x, $y, "{$this->name}({$this->color})");
    }
}

// TreeTypeFactory
class TreeTypeFactory
{
    /** @var array<string, TreeType> */
    private array $treeTypes = [];

    public function getTreeType(string $name, string $color, string $texture): TreeType
    {
        $key = "{$name}_{$color}";

        if (!isset($this->treeTypes[$key])) {
            $this->treeTypes[$key] = new TreeType($name, $color, $texture);
        }

        return $this->treeTypes[$key];
    }

    public function getCount(): int
    {
        return count($this->treeTypes);
    }
}

// Tree — extrinsic state (x, y) + flyweight রেফারেন্স
class Tree
{
    public function __construct(
        private readonly float $x,
        private readonly float $y,
        private readonly TreeType $type,
    ) {}

    public function draw(Canvas $canvas): void
    {
        $this->type->draw($this->x, $this->y, $canvas);
    }
}

// Canvas — সিম্পল রেন্ডারিং সিমুলেশন
class Canvas
{
    private array $rendered = [];

    public function drawAt(float $x, float $y, string $content): void
    {
        $this->rendered[] = "({$x},{$y}): {$content}";
    }

    public function getRenderedCount(): int
    {
        return count($this->rendered);
    }
}

// Forest — বন
class Forest
{
    /** @var Tree[] */
    private array $trees = [];
    private readonly TreeTypeFactory $factory;

    public function __construct()
    {
        $this->factory = new TreeTypeFactory();
    }

    public function plantTree(float $x, float $y, string $name, string $color, string $texture): void
    {
        $type = $this->factory->getTreeType($name, $color, $texture);
        $this->trees[] = new Tree($x, $y, $type);
    }

    public function draw(): Canvas
    {
        $canvas = new Canvas();
        foreach ($this->trees as $tree) {
            $tree->draw($canvas);
        }
        return $canvas;
    }

    public function getStats(): string
    {
        return sprintf(
            "গাছের সংখ্যা: %d | ইউনিক TreeType (Flyweight): %d",
            count($this->trees),
            $this->factory->getCount()
        );
    }
}

// === ব্যবহার ===
$forest = new Forest();
$treeData = [
    ['আম', '#228B22', str_repeat('🌳', 200)],
    ['কাঁঠাল', '#006400', str_repeat('🌲', 200)],
    ['তাল', '#32CD32', str_repeat('🌴', 200)],
    ['নারিকেল', '#2E8B57', str_repeat('🌴', 200)],
];

// ১০,০০০ গাছ লাগানো — মাত্র ৪টি TreeType flyweight
for ($i = 0; $i < 10_000; $i++) {
    [$name, $color, $texture] = $treeData[array_rand($treeData)];
    $forest->plantTree(
        x: mt_rand(0, 10000) / 10,
        y: mt_rand(0, 10000) / 10,
        name: $name,
        color: $color,
        texture: $texture,
    );
}

$canvas = $forest->draw();
echo $forest->getStats() . "\n";
echo "রেন্ডার হয়েছে: {$canvas->getRenderedCount()} গাছ\n";
```

#### JavaScript ES2022+

```javascript
class TreeType {
    #name; #color; #texture;

    constructor(name, color, texture) {
        this.#name = name;
        this.#color = color;
        this.#texture = texture;
        Object.freeze(this);
    }

    draw(x, y) {
        return `(${x.toFixed(1)},${y.toFixed(1)}): ${this.#name}(${this.#color})`;
    }

    get name() { return this.#name; }
}

class TreeTypeFactory {
    #types = new Map();

    getType(name, color, texture) {
        const key = `${name}_${color}`;
        if (!this.#types.has(key)) {
            this.#types.set(key, new TreeType(name, color, texture));
        }
        return this.#types.get(key);
    }

    get count() { return this.#types.size; }
}

class Forest {
    #trees = [];
    #factory = new TreeTypeFactory();

    plantTree(x, y, name, color, texture) {
        const type = this.#factory.getType(name, color, texture);
        this.#trees.push({ x, y, type });
    }

    draw() {
        return this.#trees.map(t => t.type.draw(t.x, t.y));
    }

    get stats() {
        return `গাছ: ${this.#trees.length} | ইউনিক TreeType: ${this.#factory.count}`;
    }
}

// === ঢাকা শহরের পার্কের বৃক্ষরোপণ সিমুলেশন ===
const forest = new Forest();
const treePresets = [
    ['আম',      '#228B22', '🌳'.repeat(200)],
    ['কাঁঠাল',  '#006400', '🌲'.repeat(200)],
    ['তাল',     '#32CD32', '🌴'.repeat(200)],
    ['নারিকেল', '#2E8B57', '🌴'.repeat(200)],
    ['বকুল',    '#3CB371', '🌳'.repeat(200)],
];

for (let i = 0; i < 50_000; i++) {
    const [name, color, texture] = treePresets[Math.floor(Math.random() * treePresets.length)];
    forest.plantTree(
        Math.random() * 1000,
        Math.random() * 1000,
        name, color, texture
    );
}

console.log(forest.stats);
// গাছ: 50000 | ইউনিক TreeType: 5
// ৫০,০০০ গাছের জন্য মাত্র ৫টি TreeType অবজেক্ট!
```

---

### ৫. 🇧🇩 বাংলাদেশ প্রসঙ্গ — ঢাকা শহরের ম্যাপ টাইল রেন্ডারিং

ঢাকা শহরের বিশাল ম্যাপ রেন্ডার করতে হাজার হাজার টাইল লাগে। **টাইল টাইপ** (রাস্তা, পানি, ভবন, সবুজ এলাকা) শেয়ার করা হবে, কিন্তু প্রতিটি টাইলের **গ্রিড অবস্থান** ইউনিক।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// MapTileType — Flyweight
readonly class MapTileType
{
    public function __construct(
        public string $name,
        public string $textureData,  // ভারী — ইমেজ ডেটা
        public string $color,
        public bool   $walkable,
        public float  $speedModifier,
    ) {}

    public function render(int $gridX, int $gridY, string $label = ''): string
    {
        $extra = $label ? " [{$label}]" : '';
        return "({$gridX},{$gridY}): {$this->name}{$extra}";
    }

    public function estimateBytes(): int
    {
        return strlen($this->textureData) * 2 + 128;
    }
}

class MapTileFactory
{
    /** @var array<string, MapTileType> */
    private array $types = [];

    public function getTileType(
        string $name,
        string $textureData,
        string $color,
        bool $walkable,
        float $speedModifier,
    ): MapTileType {
        if (!isset($this->types[$name])) {
            $this->types[$name] = new MapTileType($name, $textureData, $color, $walkable, $speedModifier);
        }
        return $this->types[$name];
    }

    public function getTypeCount(): int { return count($this->types); }

    public function getTotalIntrinsicBytes(): int
    {
        return array_sum(array_map(fn($t) => $t->estimateBytes(), $this->types));
    }
}

// ম্যাপ টাইল — extrinsic state
class MapTile
{
    public function __construct(
        public readonly int $gridX,
        public readonly int $gridY,
        public readonly MapTileType $type,
        public readonly string $label = '',
    ) {}

    public function render(): string
    {
        return $this->type->render($this->gridX, $this->gridY, $this->label);
    }
}

// DhakaMap — ঢাকা শহরের ম্যাপ
class DhakaMap
{
    /** @var MapTile[] */
    private array $tiles = [];
    private readonly MapTileFactory $factory;

    public function __construct()
    {
        $this->factory = new MapTileFactory();
    }

    public function addTile(int $x, int $y, string $typeName, string $label = ''): void
    {
        $presets = [
            'রাস্তা'     => ['tex' => str_repeat('░', 1000), 'color' => '#666', 'walk' => true,  'speed' => 1.0],
            'পানি'        => ['tex' => str_repeat('≈', 1000), 'color' => '#06F', 'walk' => false, 'speed' => 0.0],
            'ভবন'         => ['tex' => str_repeat('█', 1000), 'color' => '#888', 'walk' => false, 'speed' => 0.0],
            'সবুজ_এলাকা'  => ['tex' => str_repeat('▓', 1000), 'color' => '#0A0', 'walk' => true,  'speed' => 0.7],
            'বাজার'       => ['tex' => str_repeat('▒', 1000), 'color' => '#F90', 'walk' => true,  'speed' => 0.5],
            'রেললাইন'     => ['tex' => str_repeat('═', 1000), 'color' => '#333', 'walk' => false, 'speed' => 0.0],
        ];

        $p = $presets[$typeName] ?? $presets['রাস্তা'];
        $type = $this->factory->getTileType($typeName, $p['tex'], $p['color'], $p['walk'], $p['speed']);
        $this->tiles[] = new MapTile($x, $y, $type, $label);
    }

    public function render(int $startX, int $startY, int $width, int $height): array
    {
        $visible = [];
        foreach ($this->tiles as $tile) {
            if ($tile->gridX >= $startX && $tile->gridX < $startX + $width
                && $tile->gridY >= $startY && $tile->gridY < $startY + $height) {
                $visible[] = $tile->render();
            }
        }
        return $visible;
    }

    public function memoryReport(): string
    {
        $tileCount = count($this->tiles);
        $typeCount = $this->factory->getTypeCount();
        $intrinsic = $this->factory->getTotalIntrinsicBytes();
        $extrinsicPerTile = 32; // gridX, gridY, ref, label estimate
        $extrinsic = $tileCount * $extrinsicPerTile;

        $withFw = $intrinsic + $extrinsic;
        $avgIntrinsicPerType = $intrinsic / max(1, $typeCount);
        $withoutFw = $tileCount * ($avgIntrinsicPerType + $extrinsicPerTile);

        $fmt = fn(int $b) => match(true) {
            $b >= 1_048_576 => round($b / 1_048_576, 2) . ' MB',
            $b >= 1024      => round($b / 1024, 2) . ' KB',
            default         => $b . ' B',
        };

        return implode("\n", [
            "=== ঢাকা ম্যাপ মেমোরি রিপোর্ট ===",
            "মোট টাইল: {$tileCount}",
            "ইউনিক টাইল টাইপ (Flyweight): {$typeCount}",
            "Intrinsic (শেয়ার্ড): {$fmt($intrinsic)}",
            "Extrinsic (প্রতিটি টাইলের): {$fmt($extrinsic)}",
            "Flyweight সহ মোট: {$fmt($withFw)}",
            "Flyweight ছাড়া মোট: {$fmt((int)$withoutFw)}",
            sprintf("সাশ্রয়: ~%.1f%%", (1 - $withFw / max(1, $withoutFw)) * 100),
        ]);
    }
}

// === ঢাকা শহরের ম্যাপ তৈরি ===
$dhaka = new DhakaMap();

// গুলশান এলাকা
for ($x = 0; $x < 100; $x++) {
    for ($y = 0; $y < 100; $y++) {
        $type = match (true) {
            $x % 10 === 0 || $y % 10 === 0 => 'রাস্তা',
            $x > 40 && $x < 60 && $y > 40 && $y < 60 => 'সবুজ_এলাকা',
            $x > 70 && $y < 30 => 'বাজার',
            $y === 50 => 'রেললাইন',
            $x > 20 && $x < 25 && $y > 70 => 'পানি',
            default => 'ভবন',
        };
        $dhaka->addTile($x, $y, $type);
    }
}

echo $dhaka->memoryReport() . "\n";
// ১০,০০০ টাইলের জন্য মাত্র ৬টি টাইল টাইপ!
```

---

### ৬. 🇧🇩 বৃহৎ ইনভেন্টরি সিস্টেম অপটিমাইজেশন

বাংলাদেশের বড় ই-কমার্স প্ল্যাটফর্মে লক্ষ লক্ষ পণ্যের ভ্যারিয়েন্ট থাকতে পারে। **পণ্যের ভিত্তি তথ্য** (নাম, বিবরণ, ছবি) শেয়ার করে শুধু **ভ্যারিয়েন্ট-নির্দিষ্ট তথ্য** (সাইজ, রঙ, স্টক, দাম) আলাদা রাখা যায়।

#### JavaScript ES2022+

```javascript
// ProductBase — Flyweight (ভারী ডেটা: ছবি, বিবরণ)
class ProductBase {
    #name; #description; #images; #brand; #category;

    constructor(name, description, images, brand, category) {
        this.#name = name;
        this.#description = description;
        this.#images = images;
        this.#brand = brand;
        this.#category = category;
        Object.freeze(this);
    }

    get name() { return this.#name; }
    get description() { return this.#description; }
    get key() { return `${this.#brand}_${this.#name}`; }

    display(variant) {
        return `${this.#name} (${this.#brand}) — ${variant.size}/${variant.color} — ৳${variant.price} — স্টক: ${variant.stock}`;
    }

    get estimatedBytes() {
        return (this.#name.length + this.#description.length) * 2
            + this.#images.reduce((s, img) => s + img.length * 2, 0)
            + 128;
    }
}

class ProductBaseFactory {
    #bases = new Map();

    getBase(name, description, images, brand, category) {
        const key = `${brand}_${name}`;
        if (!this.#bases.has(key)) {
            this.#bases.set(key, new ProductBase(name, description, images, brand, category));
        }
        return this.#bases.get(key);
    }

    get count() { return this.#bases.size; }

    get totalIntrinsicBytes() {
        let total = 0;
        for (const b of this.#bases.values()) total += b.estimatedBytes;
        return total;
    }
}

// ProductVariant — extrinsic state
class ProductVariant {
    constructor(base, size, color, price, stock, sku) {
        this.base = base;     // flyweight রেফারেন্স
        this.size = size;
        this.color = color;
        this.price = price;
        this.stock = stock;
        this.sku = sku;
    }

    display() {
        return this.base.display(this);
    }
}

// === ইনভেন্টরি সিস্টেম ===
class InventorySystem {
    #variants = [];
    #factory;

    constructor() {
        this.#factory = new ProductBaseFactory();
    }

    addProduct(name, description, images, brand, category, variants) {
        const base = this.#factory.getBase(name, description, images, brand, category);
        for (const v of variants) {
            this.#variants.push(new ProductVariant(base, v.size, v.color, v.price, v.stock, v.sku));
        }
    }

    memoryReport() {
        const variantCount = this.#variants.length;
        const baseCount = this.#factory.count;
        const intrinsic = this.#factory.totalIntrinsicBytes;
        const extrinsicPer = 96;
        const extrinsic = variantCount * extrinsicPer;
        const withFw = intrinsic + extrinsic;
        const withoutFw = variantCount * (intrinsic / Math.max(1, baseCount) + extrinsicPer);
        const fmt = b => b >= 1_048_576 ? `${(b/1_048_576).toFixed(2)} MB` : `${(b/1024).toFixed(2)} KB`;

        return [
            `=== ইনভেন্টরি মেমোরি রিপোর্ট ===`,
            `মোট ভ্যারিয়েন্ট: ${variantCount}`,
            `ইউনিক প্রোডাক্ট বেস (Flyweight): ${baseCount}`,
            `Flyweight সহ: ~${fmt(withFw)}`,
            `Flyweight ছাড়া: ~${fmt(withoutFw)}`,
            `সাশ্রয়: ~${((1 - withFw / withoutFw) * 100).toFixed(1)}%`,
        ].join('\n');
    }
}

// === বাংলাদেশের ই-কমার্স সিমুলেশন ===
const inventory = new InventorySystem();

const sizes = ['S', 'M', 'L', 'XL', 'XXL'];
const colors = ['লাল', 'নীল', 'সবুজ', 'কালো', 'সাদা'];

const products = [
    { name: 'পাঞ্জাবি', brand: 'Aarong', desc: 'প্রিমিয়াম সুতি পাঞ্জাবি...'.repeat(50), cat: 'পোশাক' },
    { name: 'শাড়ি', brand: 'Aarong', desc: 'জামদানি সিল্ক শাড়ি...'.repeat(50), cat: 'পোশাক' },
    { name: 'টি-শার্ট', brand: 'Yellow', desc: 'ক্যাজুয়াল কটন টি-শার্ট...'.repeat(50), cat: 'পোশাক' },
    { name: 'জুতা', brand: 'Apex', desc: 'চামড়ার ফর্মাল জুতা...'.repeat(50), cat: 'জুতা' },
    { name: 'ব্যাগ', brand: 'Fortuna', desc: 'ল্যাপটপ ব্যাকপ্যাক...'.repeat(50), cat: 'এক্সেসরি' },
];

// প্রতিটি প্রোডাক্টের ২৫টি ভ্যারিয়েন্ট (5 সাইজ × 5 রঙ)
for (const p of products) {
    const variants = [];
    for (const size of sizes) {
        for (const color of colors) {
            variants.push({
                size, color,
                price: Math.floor(500 + Math.random() * 5000),
                stock: Math.floor(Math.random() * 100),
                sku: `${p.brand}-${p.name}-${size}-${color}`,
            });
        }
    }
    inventory.addProduct(
        p.name, p.desc,
        [`img1_${p.name}.jpg`, `img2_${p.name}.jpg`, `img3_${p.name}.jpg`],
        p.brand, p.cat, variants
    );
}

console.log(inventory.memoryReport());
```

---

## 🌍 Real-World Applicable Areas

### ১. String Interning (PHP/JS String Pools)

```php
// PHP — অভ্যন্তরীণভাবে string interning করে
$a = 'hello';
$b = 'hello';
// PHP engine একই স্ট্রিং ডেটা শেয়ার করে — এটি মূলত Flyweight প্যাটার্ন!
```

```javascript
// JS — string interning স্বয়ংক্রিয়
const a = 'hello';
const b = 'hello';
console.log(a === b); // true — একই মেমোরি রেফারেন্স

// Symbol দিয়ে explicit interning
const s1 = Symbol.for('shared_key');
const s2 = Symbol.for('shared_key');
console.log(s1 === s2); // true
```

### ২. Icon/Image Caching

```javascript
class IconCache {
    static #cache = new Map();

    static getIcon(name) {
        if (!this.#cache.has(name)) {
            // ভারী অপারেশন — ইমেজ লোড
            this.#cache.set(name, { name, data: `[${name} pixel data]`, loaded: Date.now() });
        }
        return this.#cache.get(name);
    }
}

// হাজার হাজার ফাইল লিস্ট আইটেমে একই আইকন শেয়ার করা হচ্ছে
const items = Array.from({ length: 10_000 }, (_, i) => ({
    name: `file_${i}.pdf`,
    icon: IconCache.getIcon('pdf_icon'), // সবগুলো একই রেফারেন্স!
}));
```

### ৩. Database Connection Pool (ধারণাগত সাদৃশ্য)

```php
// Connection pool — Flyweight ধারণার প্রয়োগ
class ConnectionPool
{
    /** @var array<string, PDO> */
    private static array $connections = [];

    public static function getConnection(string $dsn, string $user, string $pass): PDO
    {
        $key = md5("{$dsn}_{$user}");

        if (!isset(self::$connections[$key])) {
            self::$connections[$key] = new PDO($dsn, $user, $pass);
        }

        return self::$connections[$key];
    }
}
```

### ৪. Browser DOM Element Reuse (React Virtual DOM ধারণা)

```javascript
// React-এর মতো ভার্চুয়াল DOM — একই ধরনের element শেয়ার করা
class VirtualElement {
    #tagName; #className;

    constructor(tagName, className) {
        this.#tagName = tagName;
        this.#className = className;
        Object.freeze(this);
    }

    render(props) {
        return `<${this.#tagName} class="${this.#className}" ${
            Object.entries(props).map(([k,v]) => `${k}="${v}"`).join(' ')
        }/>`;
    }
}

class VirtualElementFactory {
    #elements = new Map();

    getElement(tagName, className) {
        const key = `${tagName}.${className}`;
        if (!this.#elements.has(key)) {
            this.#elements.set(key, new VirtualElement(tagName, className));
        }
        return this.#elements.get(key);
    }
}

// হাজার হাজার লিস্ট আইটেমে একই element type শেয়ার
const factory = new VirtualElementFactory();
const listItems = Array.from({ length: 5000 }, (_, i) => ({
    element: factory.getElement('li', 'list-item'), // শেয়ার্ড
    props: { key: i, text: `Item ${i}` },           // ইউনিক
}));
```

---

## 🔥 Advanced Deep Dive

### ১. Flyweight vs Object Pool vs Cache

```
┌─────────────────┬──────────────────────┬──────────────────┬─────────────────┐
│ বৈশিষ্ট্য        │ Flyweight            │ Object Pool      │ Cache           │
├─────────────────┼──────────────────────┼──────────────────┼─────────────────┤
│ উদ্দেশ্য         │ মেমোরি সাশ্রয়         │ তৈরির খরচ কমানো   │ দ্রুত অ্যাক্সেস    │
│ শেয়ারিং         │ একই সাথে বহু ক্লায়েন্ট│ একটি ক্লায়েন্ট    │ বহু ক্লায়েন্ট    │
│ Mutability      │ Immutable (অন্তর্নিহিত)│ Mutable          │ যেকোনো         │
│ জীবনকাল         │ অ্যাপ জীবনকাল         │ Checkout/Return  │ TTL/Eviction   │
│ State           │ Intrinsic শেয়ার্ড    │ Reset on return  │ সম্পূর্ণ সংরক্ষিত│
│ উদাহরণ          │ শেয়ার্ড TreeType      │ DB Connection    │ Redis/Memcached│
└─────────────────┴──────────────────────┴──────────────────┴─────────────────┘
```

**মূল পার্থক্য:** Flyweight-এ অবজেক্ট **একই সাথে** বহু জায়গায় ব্যবহৃত হয়। Pool-এ অবজেক্ট **ভাড়ায়** দেওয়া হয় — একবারে একজন ব্যবহার করে। Cache শুধু **দ্রুত অ্যাক্সেসের** জন্য।

### ২. Flyweight + Factory Method

```php
<?php

declare(strict_types=1);

interface Shape
{
    public function draw(float $x, float $y, float $scale): string;
}

readonly class Circle implements Shape
{
    public function __construct(
        private string $color,
        private float $radius,
    ) {}

    public function draw(float $x, float $y, float $scale): string
    {
        return "Circle({$this->color}, r={$this->radius}) at ({$x},{$y}) scale={$scale}";
    }
}

readonly class Square implements Shape
{
    public function __construct(
        private string $color,
        private float $side,
    ) {}

    public function draw(float $x, float $y, float $scale): string
    {
        return "Square({$this->color}, s={$this->side}) at ({$x},{$y}) scale={$scale}";
    }
}

// Factory Method + Flyweight
class ShapeFactory
{
    private array $shapes = [];

    public function getShape(string $type, string $color, float $dimension): Shape
    {
        $key = "{$type}_{$color}_{$dimension}";

        if (!isset($this->shapes[$key])) {
            $this->shapes[$key] = match ($type) {
                'circle' => new Circle($color, $dimension),
                'square' => new Square($color, $dimension),
                default  => throw new \InvalidArgumentException("Unknown: {$type}"),
            };
        }

        return $this->shapes[$key];
    }

    public function getCount(): int { return count($this->shapes); }
}
```

### ৩. Flyweight + Composite (শেয়ার্ড Leaf Nodes)

```javascript
// Composite প্যাটার্নে leaf node হিসেবে Flyweight ব্যবহার
class LeafStyle {
    #type; #color;
    constructor(type, color) {
        this.#type = type;
        this.#color = color;
        Object.freeze(this);
    }
    render(x, y) { return `${this.#type}(${this.#color}) at (${x},${y})`; }
    get key() { return `${this.#type}_${this.#color}`; }
}

class LeafStyleFactory {
    #pool = new Map();
    get(type, color) {
        const key = `${type}_${color}`;
        if (!this.#pool.has(key)) this.#pool.set(key, new LeafStyle(type, color));
        return this.#pool.get(key);
    }
}

// Composite node — flyweight leaf ধারণ করে
class CompositeNode {
    #children = [];
    add(child) { this.#children.push(child); return this; }
    render() { return this.#children.map(c => c.render()).flat(); }
}

// Leaf node — flyweight + extrinsic position
class LeafNode {
    #style; #x; #y;
    constructor(style, x, y) {
        this.#style = style;
        this.#x = x; this.#y = y;
    }
    render() { return [this.#style.render(this.#x, this.#y)]; }
}

// ব্যবহার
const leafFactory = new LeafStyleFactory();
const tree = new CompositeNode();

for (let i = 0; i < 1000; i++) {
    const style = leafFactory.get(
        i % 2 === 0 ? 'circle' : 'square',
        ['red', 'blue', 'green'][i % 3]
    );
    tree.add(new LeafNode(style, Math.random() * 100, Math.random() * 100));
}
// ১০০০ leaf node, মাত্র ৬টি LeafStyle flyweight (2 type × 3 color)
```

### ৪. Immutability এবং Flyweight

Flyweight **অবশ্যই** immutable হতে হবে! কারণ:

```
শেয়ার্ড Flyweight মিউটেবল হলে কী হয়:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Client A ──── uses ────┐
                        ├──► SharedFlyweight { color: 'red' }
Client B ──── uses ────┘

Client A → flyweight.color = 'blue'  // সমস্যা!

এখন Client B-ও 'blue' দেখবে! ← Race Condition / Bug
```

```php
// PHP 8.3 — readonly class ব্যবহার করে immutability নিশ্চিত
readonly class ImmutableFlyweight
{
    public function __construct(
        public string $data,
        public string $category,
    ) {}
    // কোনো setter নেই, property পরিবর্তন করা যাবে না
}
```

```javascript
// JS — Object.freeze + private fields
class ImmutableFlyweight {
    #data; #category;
    constructor(data, category) {
        this.#data = data;
        this.#category = category;
        Object.freeze(this); // বাহ্যিক property যোগ/পরিবর্তন অসম্ভব
    }
    get data() { return this.#data; }
    get category() { return this.#category; }
}
```

### ৫. PHP SplFixedArray সহ Flyweight

`SplFixedArray` নির্দিষ্ট আকারের অ্যারে — সাধারণ PHP array-এর চেয়ে **কম মেমোরি** ব্যবহার করে। Flyweight-এর সাথে মিলিয়ে extrinsic data সংরক্ষণ আরও দক্ষ হয়।

```php
<?php

declare(strict_types=1);

readonly class TileType
{
    public function __construct(
        public string $name,
        public string $texture,
    ) {}
}

class TileTypeFactory
{
    private array $types = [];

    public function get(string $name, string $texture): int
    {
        $key = $name;
        if (!isset($this->types[$key])) {
            $this->types[$key] = [
                'id'   => count($this->types),
                'type' => new TileType($name, $texture),
            ];
        }
        return $this->types[$key]['id'];
    }

    public function getType(int $id): TileType
    {
        foreach ($this->types as $entry) {
            if ($entry['id'] === $id) return $entry['type'];
        }
        throw new \RuntimeException("Unknown type ID: {$id}");
    }
}

// SplFixedArray দিয়ে মেমোরি-দক্ষ গ্রিড
class TileGrid
{
    private \SplFixedArray $typeIds;

    public function __construct(
        private readonly int $width,
        private readonly int $height,
        private readonly TileTypeFactory $factory,
    ) {
        $this->typeIds = new \SplFixedArray($width * $height);
    }

    public function setTile(int $x, int $y, string $name, string $texture): void
    {
        $index = $y * $this->width + $x;
        $this->typeIds[$index] = $this->factory->get($name, $texture);
    }

    public function getTile(int $x, int $y): TileType
    {
        $index = $y * $this->width + $x;
        return $this->factory->getType($this->typeIds[$index]);
    }

    public function getSize(): int
    {
        return $this->typeIds->getSize();
    }
}

// ব্যবহার
$factory = new TileTypeFactory();
$grid = new TileGrid(1000, 1000, $factory); // ১০ লক্ষ টাইল!

$grid->setTile(0, 0, 'grass', str_repeat('G', 5000));
$grid->setTile(1, 0, 'water', str_repeat('W', 5000));

echo "গ্রিড সাইজ: {$grid->getSize()} টাইল\n";
// SplFixedArray + Flyweight = মেমোরি-দক্ষতার সর্বোত্তম সমন্বয়
```

### ৬. JS WeakRef/WeakMap দিয়ে Flyweight Caching

`WeakRef` এবং `WeakMap` ব্যবহার করে flyweight cache **গার্বেজ কালেক্টর-বান্ধব** করা যায় — যখন কেউ ব্যবহার করছে না তখন মেমোরি স্বয়ংক্রিয়ভাবে মুক্ত হবে।

```javascript
class HeavyResource {
    #data;
    constructor(key) {
        this.key = key;
        this.#data = 'X'.repeat(100_000); // ভারী ডেটা সিমুলেশন
    }
    get data() { return this.#data; }
}

class WeakFlyweightFactory {
    #cache = new Map(); // key → WeakRef<HeavyResource>
    #finalizationRegistry;

    constructor() {
        // WeakRef-এ থাকা অবজেক্ট GC হলে cache থেকে মুছে ফেলা
        this.#finalizationRegistry = new FinalizationRegistry(key => {
            const ref = this.#cache.get(key);
            if (ref && !ref.deref()) {
                this.#cache.delete(key);
                console.log(`GC: '${key}' cache থেকে মুছে ফেলা হয়েছে`);
            }
        });
    }

    get(key) {
        const ref = this.#cache.get(key);
        const existing = ref?.deref();

        if (existing) {
            console.log(`Cache hit: '${key}'`);
            return existing;
        }

        console.log(`Cache miss: '${key}' — নতুন তৈরি হচ্ছে`);
        const resource = new HeavyResource(key);
        this.#cache.set(key, new WeakRef(resource));
        this.#finalizationRegistry.register(resource, key);
        return resource;
    }

    get cacheSize() { return this.#cache.size; }
}

// WeakMap দিয়ে অবজেক্ট-কীড flyweight
class WeakMapFlyweightCache {
    #cache = new WeakMap();

    getMetadata(obj) {
        if (!this.#cache.has(obj)) {
            // obj-এর জন্য মেটাডেটা তৈরি ও ক্যাশ
            this.#cache.set(obj, {
                computedAt: Date.now(),
                hash: Math.random().toString(36).slice(2),
            });
        }
        return this.#cache.get(obj);
    }
    // obj GC হলে সংশ্লিষ্ট entry স্বয়ংক্রিয়ভাবে মুছে যাবে
}
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | বিবরণ |
|---|--------|--------|
| 1 | **মেমোরি সাশ্রয়** | হাজার/লক্ষ অবজেক্টের intrinsic state শেয়ার করে বিপুল মেমোরি বাঁচায় |
| 2 | **পারফরম্যান্স উন্নতি** | কম অবজেক্ট = কম GC চাপ = দ্রুত অ্যাপ্লিকেশন |
| 3 | **স্কেলেবিলিটি** | সীমিত মেমোরিতে বেশি সংখ্যক অবজেক্ট পরিচালনা সম্ভব |
| 4 | **Centralized State** | শেয়ার্ড ডেটা এক জায়গায় — আপডেট সহজ |
| 5 | **Cache-friendly** | কম ইউনিক অবজেক্ট = বেশি CPU cache hit |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | বিবরণ |
|---|---------|--------|
| 1 | **জটিলতা বৃদ্ধি** | Intrinsic/Extrinsic state আলাদা করা কোডকে জটিল করে |
| 2 | **CPU trade-off** | Extrinsic state প্রতিবার পাস করতে হয় — অতিরিক্ত CPU সময় |
| 3 | **Thread Safety** | মাল্টি-থ্রেডেড পরিবেশে factory synchronization দরকার |
| 4 | **ডিবাগিং কঠিন** | শেয়ার্ড অবজেক্টের কারণে বাগ খুঁজে পাওয়া কঠিন হতে পারে |
| 5 | **অপ্রয়োজনীয় অপটিমাইজেশন** | কম সংখ্যক অবজেক্টে ব্যবহার করলে overhead বেশি হয় |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ১. Flyweight-কে Mutable রাখা

```php
// ❌ ভুল — mutable flyweight
class BadFlyweight
{
    public string $color = 'red'; // পরিবর্তনযোগ্য!

    public function setColor(string $c): void
    {
        $this->color = $c; // শেয়ার্ড অবজেক্ট পরিবর্তন — সব ক্লায়েন্ট প্রভাবিত!
    }
}

// ✅ সঠিক — immutable flyweight
readonly class GoodFlyweight
{
    public function __construct(
        public string $color,
    ) {} // readonly — পরিবর্তন অসম্ভব
}
```

### ২. Extrinsic State Flyweight-এ সংরক্ষণ করা

```javascript
// ❌ ভুল — position (extrinsic) flyweight-এর ভিতরে
class BadFlyweight {
    constructor(color, texture, x, y) { // x, y extrinsic — এখানে থাকা উচিত নয়!
        this.color = color;
        this.texture = texture;
        this.x = x;
        this.y = y;
    }
}

// ✅ সঠিক — শুধু intrinsic state flyweight-এ
class GoodFlyweight {
    #color; #texture;
    constructor(color, texture) {
        this.#color = color;
        this.#texture = texture;
        Object.freeze(this);
    }
    draw(x, y) { /* x, y বাইরে থেকে আসে */ }
}
```

### ৩. Factory ছাড়া Flyweight তৈরি করা

```php
// ❌ ভুল — সরাসরি new দিয়ে তৈরি, শেয়ারিং নেই
$fw1 = new TreeType('আম', '#228B22', '...');
$fw2 = new TreeType('আম', '#228B22', '...'); // ডুপ্লিকেট!

// ✅ সঠিক — Factory দিয়ে তৈরি
$fw1 = $factory->getTreeType('আম', '#228B22', '...');
$fw2 = $factory->getTreeType('আম', '#228B22', '...'); // $fw1 === $fw2
```

### ৪. অল্প সংখ্যক অবজেক্টে Flyweight ব্যবহার

```javascript
// ❌ ভুল — মাত্র ১০টি অবজেক্টের জন্য Flyweight
// Factory overhead > সাশ্রয় — অর্থহীন!

// ✅ সঠিক — হাজার/লক্ষ অবজেক্ট হলে Flyweight
// ১০,০০০+ অবজেক্টে উল্লেখযোগ্য সাশ্রয়
```

### ৫. ভুল Key দিয়ে ভিন্ন Flyweight শেয়ার করা

```php
// ❌ ভুল — শুধু নাম দিয়ে key তৈরি
$key = $name; // 'আম' — কিন্তু ভিন্ন রঙের আম গাছ থাকতে পারে!

// ✅ সঠিক — সব intrinsic property মিলিয়ে key
$key = "{$name}_{$color}_{$size}";
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class FlyweightTest extends TestCase
{
    private FlyweightFactory $factory;

    protected function setUp(): void
    {
        $this->factory = new FlyweightFactory();
    }

    public function testSameKeyReturnsSameInstance(): void
    {
        $fw1 = $this->factory->getFlyweight('TypeA', 'Red');
        $fw2 = $this->factory->getFlyweight('TypeA', 'Red');

        $this->assertSame($fw1, $fw2, 'একই key-তে একই instance ফেরত আসা উচিত');
    }

    public function testDifferentKeysReturnDifferentInstances(): void
    {
        $fw1 = $this->factory->getFlyweight('TypeA', 'Red');
        $fw2 = $this->factory->getFlyweight('TypeB', 'Blue');

        $this->assertNotSame($fw1, $fw2, 'ভিন্ন key-তে ভিন্ন instance আসা উচিত');
    }

    public function testFactoryCountIncrementsCorrectly(): void
    {
        $this->assertEquals(0, $this->factory->getCount());

        $this->factory->getFlyweight('A', 'X');
        $this->assertEquals(1, $this->factory->getCount());

        $this->factory->getFlyweight('A', 'X'); // পুনরায় ব্যবহৃত
        $this->assertEquals(1, $this->factory->getCount());

        $this->factory->getFlyweight('B', 'Y'); // নতুন
        $this->assertEquals(2, $this->factory->getCount());
    }

    public function testExtrinsicStateDoesNotAffectSharing(): void
    {
        $fw = $this->factory->getFlyweight('Type', 'Color');

        $result1 = $fw->operation(['pos' => '10,20']);
        $result2 = $fw->operation(['pos' => '30,40']);

        $this->assertStringContainsString('10,20', $result1);
        $this->assertStringContainsString('30,40', $result2);
        $this->assertNotEquals($result1, $result2);
    }

    public function testFlyweightIsImmutable(): void
    {
        $fw = new ConcreteFlyweight('data', 'category');

        // readonly class — property assign করলে Error হবে
        $this->expectException(\Error::class);
        $fw->sharedData = 'hacked'; // @phpstan-ignore-line
    }

    public function testTreeTypeFactorySharing(): void
    {
        $factory = new TreeTypeFactory();

        $type1 = $factory->getTreeType('আম', '#228B22', 'texture_data');
        $type2 = $factory->getTreeType('আম', '#228B22', 'texture_data');
        $type3 = $factory->getTreeType('কাঁঠাল', '#006400', 'texture_data');

        $this->assertSame($type1, $type2);
        $this->assertNotSame($type1, $type3);
        $this->assertEquals(2, $factory->getCount());
    }

    public function testParticleSystemMemoryEfficiency(): void
    {
        $factory = new ParticleTypeFactory();
        $system = new ParticleSystem($factory);

        $system->emit('fire', 0, 0, 1000);
        $system->emit('smoke', 0, 0, 1000);
        $system->emit('fire', 50, 50, 1000); // আবার fire — নতুন type তৈরি হবে না

        $this->assertEquals(3000, $system->getParticleCount());
        $this->assertEquals(2, count($factory->getTypes())); // মাত্র ২টি type
    }
}
```

### Jest (JavaScript)

```javascript
describe('Flyweight Pattern', () => {
    let factory;

    beforeEach(() => {
        factory = new FlyweightFactory();
    });

    test('একই key-তে একই instance ফেরত আসে', () => {
        const fw1 = factory.getFlyweight('TypeA', 'Red');
        const fw2 = factory.getFlyweight('TypeA', 'Red');
        expect(fw1).toBe(fw2); // strict reference equality
    });

    test('ভিন্ন key-তে ভিন্ন instance আসে', () => {
        const fw1 = factory.getFlyweight('TypeA', 'Red');
        const fw2 = factory.getFlyweight('TypeB', 'Blue');
        expect(fw1).not.toBe(fw2);
    });

    test('factory সঠিকভাবে count করে', () => {
        expect(factory.count).toBe(0);

        factory.getFlyweight('A', 'X');
        expect(factory.count).toBe(1);

        factory.getFlyweight('A', 'X');
        expect(factory.count).toBe(1); // পুনরায় ব্যবহৃত

        factory.getFlyweight('B', 'Y');
        expect(factory.count).toBe(2);
    });

    test('extrinsic state শেয়ারিং প্রভাবিত করে না', () => {
        const fw = factory.getFlyweight('T', 'C');
        const r1 = fw.operation({ pos: '10,20' });
        const r2 = fw.operation({ pos: '30,40' });

        expect(r1).toContain('10,20');
        expect(r2).toContain('30,40');
        expect(r1).not.toBe(r2);
    });

    test('flyweight immutable (Object.freeze)', () => {
        const fw = factory.getFlyweight('T', 'C');
        expect(() => { fw.newProp = 'test'; }).toThrow();
    });
});

describe('TreeType Factory', () => {
    test('একই ধরনের গাছ শেয়ার হয়', () => {
        const factory = new TreeTypeFactory();
        const t1 = factory.getType('আম', '#228B22', 'tex');
        const t2 = factory.getType('আম', '#228B22', 'tex');
        const t3 = factory.getType('কাঁঠাল', '#006400', 'tex');

        expect(t1).toBe(t2);
        expect(t1).not.toBe(t3);
        expect(factory.count).toBe(2);
    });
});

describe('Particle System', () => {
    test('একই particle type শেয়ার হয়', () => {
        const factory = new ParticleTypeFactory();
        const system = new ParticleSystem(factory);

        system.emit('fire', 0, 0, 1000);
        system.emit('smoke', 0, 0, 1000);
        system.emit('fire', 50, 50, 500);

        expect(factory.typeCount).toBe(2); // শুধু fire + smoke
    });
});

describe('Memory Comparison', () => {
    test('flyweight সহ কম মেমোরি ব্যবহৃত হয়', () => {
        const COUNT = 10_000;

        // Flyweight ছাড়া (সিমুলেশন)
        const withoutFw = Array.from({ length: COUNT }, () => ({
            font: 'SolaimanLipi', size: 14, color: '#000',
            bold: false, italic: false,
        }));

        // Flyweight সহ
        const styleFactory = new CharacterStyleFactory();
        for (let i = 0; i < COUNT; i++) {
            styleFactory.getStyle('SolaimanLipi', 14, '#000');
        }

        expect(styleFactory.uniqueStyleCount).toBe(1);
        expect(withoutFw.length).toBe(COUNT);
        // ১০,০০০ পৃথক style object vs মাত্র ১টি শেয়ার্ড style
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Flyweight ↔ Composite
**Composite** প্যাটার্নে ট্রি স্ট্রাকচারের **leaf node** হিসেবে Flyweight ব্যবহার করা যায়। উদাহরণ: গ্রাফিক্স এডিটরে শেয়ার্ড আকৃতি (leaf) + গ্রুপিং (composite)।

### Flyweight ↔ Singleton
**Singleton** একটি মাত্র instance তৈরি করে। **Flyweight Factory** একাধিক শেয়ার্ড instance ম্যানেজ করে — এটি মূলত "keyed Singleton"। Factory নিজে Singleton হতে পারে।

### Flyweight ↔ Factory
**Flyweight Factory** হলো Factory Method/Abstract Factory-এর বিশেষায়িত রূপ — এটি অবজেক্ট তৈরির পাশাপাশি **ক্যাশিং** করে।

### Flyweight ↔ State/Strategy
**State** বা **Strategy** অবজেক্টগুলো যদি stateless হয় (শুধু behavior ধারণ করে), তাহলে সেগুলো Flyweight হিসেবে শেয়ার করা যায়। একই strategy instance বহু context-এ ব্যবহৃত হতে পারে।

```
সম্পর্ক মানচিত্র:
━━━━━━━━━━━━━━━━━━━
                    ┌── Composite (leaf nodes হিসেবে)
                    │
Flyweight ──────────┼── Singleton (factory নিজে Singleton হতে পারে)
                    │
                    ├── Factory (তৈরি + ক্যাশিং)
                    │
                    └── State/Strategy (শেয়ার্ড behavior objects)
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **বিপুল সংখ্যক অবজেক্ট** তৈরি হচ্ছে (হাজার/লক্ষ)
2. অবজেক্টগুলোর মধ্যে **উল্লেখযোগ্য পরিমাণ সাধারণ ডেটা** আছে
3. মেমোরি **সীমিত** এবং অপটিমাইজেশন প্রয়োজন
4. Extrinsic state সহজে **আলাদা** করা যায়
5. অবজেক্টের **পরিচয় (identity)** গুরুত্বপূর্ণ নয় — শেয়ারিং গ্রহণযোগ্য
6. **প্রোফাইলিং** দেখাচ্ছে যে মেমোরি ব্যবহার সমস্যা তৈরি করছে

### ❌ ব্যবহার করবেন না যখন:

1. অবজেক্ট সংখ্যা **অল্প** (কয়েকটি থেকে কয়েকশ)
2. প্রতিটি অবজেক্টের state **সম্পূর্ণ ইউনিক** — শেয়ার করার কিছু নেই
3. অবজেক্টের **পরিচয়** গুরুত্বপূর্ণ (প্রতিটি আলাদাভাবে চিহ্নিত হওয়া দরকার)
4. **Premature optimization** — আগে প্রোফাইল করুন, তারপর Flyweight বিবেচনা করুন
5. কোডের **পঠনযোগ্যতা ও সরলতা** বেশি গুরুত্বপূর্ণ
6. Extrinsic state আলাদা করা **অত্যন্ত জটিল** বা অস্বাভাবিক

### সিদ্ধান্ত ফ্লোচার্ট:

```
হাজার হাজার সদৃশ অবজেক্ট আছে?
├── না → Flyweight দরকার নেই
└── হ্যাঁ → সাধারণ (intrinsic) ডেটা আলাদা করা সম্ভব?
            ├── না → Flyweight উপযুক্ত নয়
            └── হ্যাঁ → Intrinsic ডেটা কি ভারী? (ছবি, টেক্সচার, বড় স্ট্রিং)
                        ├── না → সাশ্রয় সামান্য হবে — বিবেচনা করুন
                        └── হ্যাঁ → ✅ Flyweight ব্যবহার করুন!
```

---

## 📋 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────────────┐
│                    🪶 Flyweight প্যাটার্ন সারসংক্ষেপ              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  উদ্দেশ্য: শেয়ারিং-এর মাধ্যমে বিপুল সংখ্যক ক্ষুদ্র অবজেক্ট       │
│          দক্ষতার সাথে পরিচালনা করা                                │
│                                                                  │
│  মূল ধারণা:                                                      │
│  ┌─────────────┐     ┌──────────────┐                            │
│  │  Intrinsic  │────►│  শেয়ার্ড      │  (immutable, factory-এ)   │
│  │  State      │     │  Flyweight   │                            │
│  └─────────────┘     └──────────────┘                            │
│  ┌─────────────┐     ┌──────────────┐                            │
│  │  Extrinsic  │────►│  Client-এ    │  (ইউনিক, বাইরে থাকে)       │
│  │  State      │     │  সংরক্ষিত     │                            │
│  └─────────────┘     └──────────────┘                            │
│                                                                  │
│  মনে রাখুন:                                                      │
│  ① Flyweight অবশ্যই immutable                                    │
│  ② Factory দিয়ে তৈরি ও ক্যাশ করুন                                │
│  ③ Intrinsic/Extrinsic সঠিকভাবে ভাগ করুন                         │
│  ④ আগে প্রোফাইল করুন, তারপর অপটিমাইজ করুন                       │
│  ⑤ হাজার+ অবজেক্ট হলেই বিবেচনা করুন                              │
│                                                                  │
│  PHP: readonly class + factory pattern                           │
│  JS: Object.freeze + private fields + Map/WeakMap                │
│                                                                  │
│  "প্রতিটি তুষারকণা অনন্য, কিন্তু বরফের স্ফটিকের গঠন একই —        │
│   সেই গঠনটাই আমাদের Flyweight।"                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

> **"Premature optimization is the root of all evil — কিন্তু যখন প্রোফাইলার বলছে মেমোরি ফুরিয়ে যাচ্ছে, তখন Flyweight আপনার সবচেয়ে বিশ্বস্ত মিত্র।"** — Donald Knuth (পরিবর্ধিত)
