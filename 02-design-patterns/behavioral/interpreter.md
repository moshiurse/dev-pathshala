# 🔤 Interpreter প্যাটার্ন

## 📌 সংজ্ঞা

**GoF Definition:** *"Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language."*

**বাংলায়:** কোনো ভাষা দেওয়া থাকলে, সেই ভাষার ব্যাকরণের একটি রিপ্রেজেন্টেশন তৈরি করুন এবং একটি ইন্টারপ্রেটার তৈরি করুন যা সেই রিপ্রেজেন্টেশন ব্যবহার করে ভাষার বাক্যগুলো ব্যাখ্যা করবে।

Interpreter প্যাটার্ন হলো একটি **Behavioral Design Pattern** যা কোনো নির্দিষ্ট ভাষার (Domain Specific Language) ব্যাকরণকে ক্লাস হায়ারার্কি হিসেবে মডেল করে। প্রতিটি ব্যাকরণের নিয়ম একটি ক্লাস দ্বারা উপস্থাপিত হয়। এই প্যাটার্নটি মূলত **Abstract Syntax Tree (AST)** তৈরি করে এবং সেই ট্রি traverse করে expression evaluate করে।

এটি সবচেয়ে কম ব্যবহৃত GoF প্যাটার্নগুলোর মধ্যে একটি, কিন্তু যখন ব্যবহার করা হয় — তখন এটি অত্যন্ত শক্তিশালী। SQL পার্সার, রেগুলার এক্সপ্রেশন ইঞ্জিন, টেমপ্লেট ইঞ্জিন, ভ্যালিডেশন রুল পার্সিং — সবকিছুতেই এর ছাপ রয়েছে।

### মূল ধারণা

ধরুন, আপনার একটি ই-কমার্স সাইট আছে বাংলাদেশে। আপনি চান ব্যবসায়িক নিয়মগুলো কোডে হার্ডকোড না করে একটি **রুল ল্যাঙ্গুয়েজ** দিয়ে লিখতে:

```
IF total > 5000 AND member = true THEN discount 10%
IF category = "electronics" AND total > 10000 THEN discount 15%
```

এই ধরনের "mini language" বা DSL ইন্টারপ্রেট করতেই Interpreter প্যাটার্ন ব্যবহৃত হয়।

---

## 🏠 বাস্তব উদাহরণ — মিউজিক্যাল নোটেশন

কল্পনা করুন একটি **সঙ্গীতের স্বরলিপি** (sheet music)। একই শীট মিউজিক বিভিন্ন মিউজিশিয়ান পড়তে পারেন:

- **গিটারিস্ট** সেই নোট গিটারে বাজাবেন
- **পিয়ানিস্ট** সেই নোট পিয়ানোতে বাজাবেন
- **বাঁশিবাদক** সেই নোট বাঁশিতে বাজাবেন

প্রতিটি মিউজিশিয়ান একটি **ইন্টারপ্রেটার** — তারা একই ব্যাকরণ (musical notation) পড়ছেন, কিন্তু নিজস্ব context অনুযায়ী interpret করছেন।

**আরেকটি বাংলাদেশি উদাহরণ:** বাংলা ভাষার সংখ্যা পদ্ধতি। "পাঁচ হাজার তিনশত বিশ" — এই বাক্যটিকে ইন্টারপ্রেট করে `5320` বের করা। এখানে "হাজার", "শত" হলো NonTerminal expressions এবং "পাঁচ", "তিন", "বিশ" হলো Terminal expressions।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────┐
│           Context                │
│─────────────────────────────────│
│ - input: string                  │
│ - variables: Map<string, any>    │
│─────────────────────────────────│
│ + lookup(name): any              │
│ + assign(name, value): void      │
└─────────────────────────────────┘
              │ uses
              ▼
┌─────────────────────────────────┐
│    <<interface>>                 │
│    AbstractExpression            │
│─────────────────────────────────│
│ + interpret(context): any        │
└──────────┬──────────────────────┘
           │
     ┌─────┴──────────────────┐
     │                        │
     ▼                        ▼
┌──────────────┐   ┌────────────────────────┐
│  Terminal     │   │   NonTerminal           │
│  Expression   │   │   Expression            │
│──────────────│   │────────────────────────│
│ - value       │   │ - left: AbstractExpr    │
│──────────────│   │ - right: AbstractExpr   │
│ + interpret() │   │────────────────────────│
└──────────────┘   │ + interpret()           │
                    └────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐  ┌──────────┐   ┌──────────┐
        │ AddExpr  │  │ AndExpr  │   │ OrExpr   │
        └──────────┘  └──────────┘   └──────────┘

┌──────────────────────────────────────────────────────┐
│                      Client                           │
│──────────────────────────────────────────────────────│
│  1. Builds Abstract Syntax Tree (AST)                │
│  2. Creates Context with variables                    │
│  3. Calls interpret() on root of AST                 │
└──────────────────────────────────────────────────────┘
```

### কম্পোনেন্ট ব্যাখ্যা

| কম্পোনেন্ট | ভূমিকা |
|---|---|
| **AbstractExpression** | সকল expression-এর জন্য সাধারণ ইন্টারফেস। `interpret()` মেথড ডিফাইন করে |
| **TerminalExpression** | ব্যাকরণের terminal symbol। আর decompose করা যায় না (যেমন: সংখ্যা, ভেরিয়েবল) |
| **NonTerminalExpression** | অন্যান্য expression-এর সমন্বয়ে তৈরি (যেমন: `Add`, `And`, `Or`) |
| **Context** | ইন্টারপ্রেটারের জন্য global তথ্য ধারণ করে (ভেরিয়েবল, ইনপুট ইত্যাদি) |
| **Client** | AST তৈরি করে এবং `interpret()` কল করে |

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Interpreter — গাণিতিক Expression মূল্যায়ন

প্রথমে একটি সহজ উদাহরণ দেখি যেখানে `2 + 3 * 4` এর মতো expression evaluate করা হবে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Abstract Expression
interface Expression
{
    public function interpret(Context $context): int|float;
}

// Context - গ্লোবাল তথ্য ধারণ করে
class Context
{
    private array $variables = [];

    public function assign(string $name, int|float $value): void
    {
        $this->variables[$name] = $value;
    }

    public function lookup(string $name): int|float
    {
        if (!array_key_exists($name, $this->variables)) {
            throw new RuntimeException("অসংজ্ঞায়িত ভেরিয়েবল: {$name}");
        }
        return $this->variables[$name];
    }
}

// Terminal Expression — সংখ্যা
class NumberExpression implements Expression
{
    public function __construct(
        private readonly int|float $value
    ) {}

    public function interpret(Context $context): int|float
    {
        return $this->value;
    }
}

// Terminal Expression — ভেরিয়েবল
class VariableExpression implements Expression
{
    public function __construct(
        private readonly string $name
    ) {}

    public function interpret(Context $context): int|float
    {
        return $context->lookup($this->name);
    }
}

// NonTerminal Expression — যোগ
class AddExpression implements Expression
{
    public function __construct(
        private readonly Expression $left,
        private readonly Expression $right
    ) {}

    public function interpret(Context $context): int|float
    {
        return $this->left->interpret($context) + $this->right->interpret($context);
    }
}

// NonTerminal Expression — গুণ
class MultiplyExpression implements Expression
{
    public function __construct(
        private readonly Expression $left,
        private readonly Expression $right
    ) {}

    public function interpret(Context $context): int|float
    {
        return $this->left->interpret($context) * $this->right->interpret($context);
    }
}

// NonTerminal Expression — বিয়োগ
class SubtractExpression implements Expression
{
    public function __construct(
        private readonly Expression $left,
        private readonly Expression $right
    ) {}

    public function interpret(Context $context): int|float
    {
        return $this->left->interpret($context) - $this->right->interpret($context);
    }
}

// ব্যবহার: 2 + 3 * 4 = 14
$context = new Context();

// AST তৈরি (operator precedence বজায় রেখে)
// 3 * 4 আগে, তারপর 2 + (3*4)
$expression = new AddExpression(
    new NumberExpression(2),
    new MultiplyExpression(
        new NumberExpression(3),
        new NumberExpression(4)
    )
);

echo $expression->interpret($context); // 14

// ভেরিয়েবল সহ: x + y * 2
$context->assign('x', 10);
$context->assign('y', 5);

$exprWithVars = new AddExpression(
    new VariableExpression('x'),
    new MultiplyExpression(
        new VariableExpression('y'),
        new NumberExpression(2)
    )
);

echo $exprWithVars->interpret($context); // 20
```

#### JavaScript (ES2022+)

```javascript
// Abstract Expression — interpret() মেথড থাকতে হবে

class Context {
    #variables = new Map();

    assign(name, value) {
        this.#variables.set(name, value);
    }

    lookup(name) {
        if (!this.#variables.has(name)) {
            throw new Error(`অসংজ্ঞায়িত ভেরিয়েবল: ${name}`);
        }
        return this.#variables.get(name);
    }
}

class NumberExpression {
    #value;
    constructor(value) {
        this.#value = value;
    }
    interpret(_context) {
        return this.#value;
    }
}

class VariableExpression {
    #name;
    constructor(name) {
        this.#name = name;
    }
    interpret(context) {
        return context.lookup(this.#name);
    }
}

class AddExpression {
    #left; #right;
    constructor(left, right) {
        this.#left = left;
        this.#right = right;
    }
    interpret(context) {
        return this.#left.interpret(context) + this.#right.interpret(context);
    }
}

class MultiplyExpression {
    #left; #right;
    constructor(left, right) {
        this.#left = left;
        this.#right = right;
    }
    interpret(context) {
        return this.#left.interpret(context) * this.#right.interpret(context);
    }
}

// ব্যবহার: 2 + 3 * 4 = 14
const context = new Context();

const expression = new AddExpression(
    new NumberExpression(2),
    new MultiplyExpression(
        new NumberExpression(3),
        new NumberExpression(4)
    )
);

console.log(expression.interpret(context)); // 14
```

---

### 2️⃣ Boolean Expression Interpreter

`true AND (false OR true)` — এই ধরনের লজিক্যাল expression ইন্টারপ্রেট করা। এটি অনুমতি সিস্টেম বা বিজনেস রুল ইঞ্জিনে অত্যন্ত কার্যকর।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface BooleanExpression
{
    public function interpret(array $context): bool;
}

// Terminal — ধ্রুবক মান
class BooleanConstant implements BooleanExpression
{
    public function __construct(
        private readonly bool $value
    ) {}

    public function interpret(array $context): bool
    {
        return $this->value;
    }
}

// Terminal — ভেরিয়েবল
class BooleanVariable implements BooleanExpression
{
    public function __construct(
        private readonly string $name
    ) {}

    public function interpret(array $context): bool
    {
        return (bool) ($context[$this->name] ?? false);
    }
}

// NonTerminal — AND
class AndExpression implements BooleanExpression
{
    public function __construct(
        private readonly BooleanExpression $left,
        private readonly BooleanExpression $right
    ) {}

    public function interpret(array $context): bool
    {
        return $this->left->interpret($context) && $this->right->interpret($context);
    }
}

// NonTerminal — OR
class OrExpression implements BooleanExpression
{
    public function __construct(
        private readonly BooleanExpression $left,
        private readonly BooleanExpression $right
    ) {}

    public function interpret(array $context): bool
    {
        return $this->left->interpret($context) || $this->right->interpret($context);
    }
}

// NonTerminal — NOT
class NotExpression implements BooleanExpression
{
    public function __construct(
        private readonly BooleanExpression $expression
    ) {}

    public function interpret(array $context): bool
    {
        return !$this->expression->interpret($context);
    }
}

// true AND (false OR true)
$expression = new AndExpression(
    new BooleanConstant(true),
    new OrExpression(
        new BooleanConstant(false),
        new BooleanConstant(true)
    )
);

var_dump($expression->interpret([])); // true

// ভেরিয়েবল সহ: isPremium AND (hasDiscount OR isNewUser)
$accessRule = new AndExpression(
    new BooleanVariable('isPremium'),
    new OrExpression(
        new BooleanVariable('hasDiscount'),
        new BooleanVariable('isNewUser')
    )
);

$userContext = [
    'isPremium'   => true,
    'hasDiscount' => false,
    'isNewUser'   => true,
];

var_dump($accessRule->interpret($userContext)); // true
```

#### JavaScript (ES2022+)

```javascript
class BooleanConstant {
    #value;
    constructor(value) { this.#value = value; }
    interpret(_ctx) { return this.#value; }
}

class BooleanVariable {
    #name;
    constructor(name) { this.#name = name; }
    interpret(ctx) { return Boolean(ctx[this.#name]); }
}

class AndExpression {
    #left; #right;
    constructor(left, right) { this.#left = left; this.#right = right; }
    interpret(ctx) {
        return this.#left.interpret(ctx) && this.#right.interpret(ctx);
    }
}

class OrExpression {
    #left; #right;
    constructor(left, right) { this.#left = left; this.#right = right; }
    interpret(ctx) {
        return this.#left.interpret(ctx) || this.#right.interpret(ctx);
    }
}

class NotExpression {
    #expr;
    constructor(expr) { this.#expr = expr; }
    interpret(ctx) { return !this.#expr.interpret(ctx); }
}

// isPremium AND (hasDiscount OR isNewUser)
const accessRule = new AndExpression(
    new BooleanVariable('isPremium'),
    new OrExpression(
        new BooleanVariable('hasDiscount'),
        new BooleanVariable('isNewUser')
    )
);

console.log(accessRule.interpret({
    isPremium: true,
    hasDiscount: false,
    isNewUser: true
})); // true
```

---

### 3️⃣ SQL-like Query Interpreter — সহজ WHERE ক্লজ পার্সিং

বাংলাদেশি ই-কমার্স প্রোডাক্ট ডেটার উপর SQL-এর মতো কোয়েরি চালানো:
`WHERE price > 500 AND category = "electronics"`

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface QueryExpression
{
    public function interpret(array $row): bool;
}

// Terminal — একটি ফিল্ড তুলনা
class ComparisonExpression implements QueryExpression
{
    public function __construct(
        private readonly string $field,
        private readonly string $operator,
        private readonly mixed  $value,
    ) {}

    public function interpret(array $row): bool
    {
        $fieldValue = $row[$this->field] ?? null;

        return match ($this->operator) {
            '='  => $fieldValue == $this->value,
            '!=' => $fieldValue != $this->value,
            '>'  => $fieldValue > $this->value,
            '<'  => $fieldValue < $this->value,
            '>=' => $fieldValue >= $this->value,
            '<=' => $fieldValue <= $this->value,
            'LIKE' => str_contains(
                strtolower((string) $fieldValue),
                strtolower(trim((string) $this->value, '%'))
            ),
            default => throw new RuntimeException("অজানা অপারেটর: {$this->operator}"),
        };
    }
}

// NonTerminal — AND
class QueryAndExpression implements QueryExpression
{
    public function __construct(
        private readonly QueryExpression $left,
        private readonly QueryExpression $right
    ) {}

    public function interpret(array $row): bool
    {
        return $this->left->interpret($row) && $this->right->interpret($row);
    }
}

// NonTerminal — OR
class QueryOrExpression implements QueryExpression
{
    public function __construct(
        private readonly QueryExpression $left,
        private readonly QueryExpression $right
    ) {}

    public function interpret(array $row): bool
    {
        return $this->left->interpret($row) || $this->right->interpret($row);
    }
}

// সাধারণ WHERE পার্সার
class SimpleWhereParser
{
    public function parse(string $where): QueryExpression
    {
        // AND দিয়ে বিভক্ত (সরলীকৃত পার্সার)
        if (stripos($where, ' AND ') !== false) {
            $parts = preg_split('/\s+AND\s+/i', $where, 2);
            return new QueryAndExpression(
                $this->parse($parts[0]),
                $this->parse($parts[1])
            );
        }

        if (stripos($where, ' OR ') !== false) {
            $parts = preg_split('/\s+OR\s+/i', $where, 2);
            return new QueryOrExpression(
                $this->parse($parts[0]),
                $this->parse($parts[1])
            );
        }

        // একক comparison পার্স
        if (preg_match('/(\w+)\s*(>=|<=|!=|=|>|<|LIKE)\s*(.+)/i', trim($where), $matches)) {
            $value = trim($matches[3], " \"'");
            return new ComparisonExpression(
                $matches[1],
                strtoupper($matches[2]),
                is_numeric($value) ? (float) $value : $value
            );
        }

        throw new RuntimeException("পার্স করা যায়নি: {$where}");
    }
}

// বাংলাদেশি ই-কমার্স ডেটা
$products = [
    ['name' => 'স্যামসাং গ্যালাক্সি', 'price' => 25000, 'category' => 'electronics'],
    ['name' => 'পাঞ্জাবি',           'price' => 1200,  'category' => 'clothing'],
    ['name' => 'ল্যাপটপ ব্যাগ',      'price' => 800,   'category' => 'accessories'],
    ['name' => 'শাওমি ফোন',          'price' => 15000, 'category' => 'electronics'],
    ['name' => 'জামদানি শাড়ি',       'price' => 8000,  'category' => 'clothing'],
];

$parser = new SimpleWhereParser();

$query = $parser->parse('price > 5000 AND category = "electronics"');

$results = array_filter($products, fn(array $row) => $query->interpret($row));

foreach ($results as $product) {
    echo "{$product['name']} — ৳{$product['price']}\n";
}
// স্যামসাং গ্যালাক্সি — ৳25000
// শাওমি ফোন — ৳15000
```

#### JavaScript (ES2022+)

```javascript
class ComparisonExpression {
    #field; #operator; #value;

    constructor(field, operator, value) {
        this.#field = field;
        this.#operator = operator;
        this.#value = value;
    }

    interpret(row) {
        const fieldVal = row[this.#field];
        switch (this.#operator) {
            case '=':  return fieldVal == this.#value;
            case '!=': return fieldVal != this.#value;
            case '>':  return fieldVal > this.#value;
            case '<':  return fieldVal < this.#value;
            case '>=': return fieldVal >= this.#value;
            case '<=': return fieldVal <= this.#value;
            case 'LIKE':
                return String(fieldVal).toLowerCase()
                    .includes(String(this.#value).replace(/%/g, '').toLowerCase());
            default: throw new Error(`অজানা অপারেটর: ${this.#operator}`);
        }
    }
}

class QueryAndExpr {
    #left; #right;
    constructor(left, right) { this.#left = left; this.#right = right; }
    interpret(row) {
        return this.#left.interpret(row) && this.#right.interpret(row);
    }
}

class SimpleWhereParser {
    parse(where) {
        const andIdx = where.toUpperCase().indexOf(' AND ');
        if (andIdx !== -1) {
            return new QueryAndExpr(
                this.parse(where.slice(0, andIdx)),
                this.parse(where.slice(andIdx + 5))
            );
        }

        const match = where.trim().match(/(\w+)\s*(>=|<=|!=|=|>|<|LIKE)\s*(.+)/i);
        if (!match) throw new Error(`পার্স করা যায়নি: ${where}`);

        let value = match[3].trim().replace(/["']/g, '');
        if (!isNaN(value)) value = Number(value);

        return new ComparisonExpression(match[1], match[2].toUpperCase(), value);
    }
}

const products = [
    { name: 'স্যামসাং গ্যালাক্সি', price: 25000, category: 'electronics' },
    { name: 'পাঞ্জাবি',           price: 1200,  category: 'clothing' },
    { name: 'শাওমি ফোন',          price: 15000, category: 'electronics' },
];

const parser = new SimpleWhereParser();
const query = parser.parse('price > 5000 AND category = "electronics"');

const results = products.filter(p => query.interpret(p));
results.forEach(p => console.log(`${p.name} — ৳${p.price}`));
```

---

### 4️⃣ Rule Engine — বাংলাদেশি ই-কমার্স ডিসকাউন্ট সিস্টেম

এটি সবচেয়ে বাস্তবসম্মত উদাহরণ। ব্যবসায়িক নিয়মগুলো DSL আকারে লেখা হয় এবং ইন্টারপ্রেটার সেগুলো execute করে।

```
IF total > 5000 AND member = true THEN discount 10%
IF category = "electronics" AND total > 10000 THEN discount 15%
IF isRamadan = true THEN discount 5%
```

#### PHP 8.3 — সম্পূর্ণ Rule Engine

```php
<?php

declare(strict_types=1);

// === Condition Expressions ===

interface ConditionExpression
{
    public function evaluate(array $facts): bool;
}

class FieldComparison implements ConditionExpression
{
    public function __construct(
        private readonly string $field,
        private readonly string $operator,
        private readonly mixed  $value,
    ) {}

    public function evaluate(array $facts): bool
    {
        $actual = $facts[$this->field] ?? null;

        return match ($this->operator) {
            '>'  => $actual > $this->value,
            '<'  => $actual < $this->value,
            '>=' => $actual >= $this->value,
            '<=' => $actual <= $this->value,
            '='  => $actual == $this->value,
            '!=' => $actual != $this->value,
            default => false,
        };
    }
}

class AndCondition implements ConditionExpression
{
    /** @param ConditionExpression[] $conditions */
    public function __construct(
        private readonly array $conditions
    ) {}

    public function evaluate(array $facts): bool
    {
        foreach ($this->conditions as $condition) {
            if (!$condition->evaluate($facts)) return false;
        }
        return true;
    }
}

class OrCondition implements ConditionExpression
{
    /** @param ConditionExpression[] $conditions */
    public function __construct(
        private readonly array $conditions
    ) {}

    public function evaluate(array $facts): bool
    {
        foreach ($this->conditions as $condition) {
            if ($condition->evaluate($facts)) return true;
        }
        return false;
    }
}

// === Action Expressions ===

interface ActionExpression
{
    public function execute(array &$facts): void;
}

class DiscountAction implements ActionExpression
{
    public function __construct(
        private readonly float $percentage
    ) {}

    public function execute(array &$facts): void
    {
        $currentDiscount = $facts['_appliedDiscount'] ?? 0;
        // সর্বোচ্চ ডিসকাউন্ট প্রয়োগ
        $facts['_appliedDiscount'] = max($currentDiscount, $this->percentage);
        $facts['_discountLog'][] = "ডিসকাউন্ট {$this->percentage}% প্রয়োগযোগ্য";
    }
}

class FreeShippingAction implements ActionExpression
{
    public function execute(array &$facts): void
    {
        $facts['_freeShipping'] = true;
        $facts['_discountLog'][] = 'ফ্রি শিপিং প্রযোজ্য';
    }
}

// === Rule ===

class Rule
{
    public function __construct(
        private readonly string              $name,
        private readonly ConditionExpression $condition,
        private readonly ActionExpression    $action,
        private readonly int                 $priority = 0,
    ) {}

    public function apply(array &$facts): bool
    {
        if ($this->condition->evaluate($facts)) {
            $this->action->execute($facts);
            return true;
        }
        return false;
    }

    public function getPriority(): int
    {
        return $this->priority;
    }

    public function getName(): string
    {
        return $this->name;
    }
}

// === Rule Engine ===

class RuleEngine
{
    /** @var Rule[] */
    private array $rules = [];

    public function addRule(Rule $rule): self
    {
        $this->rules[] = $rule;
        // উচ্চ priority আগে evaluate হবে
        usort($this->rules, fn(Rule $a, Rule $b) => $b->getPriority() <=> $a->getPriority());
        return $this;
    }

    public function evaluate(array &$facts): array
    {
        $facts['_appliedDiscount'] = 0;
        $facts['_freeShipping']    = false;
        $facts['_discountLog']     = [];

        $appliedRules = [];
        foreach ($this->rules as $rule) {
            if ($rule->apply($facts)) {
                $appliedRules[] = $rule->getName();
            }
        }

        return $appliedRules;
    }
}

// === Rule পার্সার — DSL থেকে Rule তৈরি ===

class RuleParser
{
    public function parse(string $ruleText): Rule
    {
        // IF ... THEN ... ফরম্যাট
        if (!preg_match('/^IF\s+(.+?)\s+THEN\s+(.+)$/i', trim($ruleText), $matches)) {
            throw new RuntimeException("রুল পার্স করা যায়নি: {$ruleText}");
        }

        $conditionStr = $matches[1];
        $actionStr    = $matches[2];

        return new Rule(
            name: $ruleText,
            condition: $this->parseCondition($conditionStr),
            action: $this->parseAction($actionStr),
        );
    }

    private function parseCondition(string $str): ConditionExpression
    {
        // AND দিয়ে বিভক্ত
        if (stripos($str, ' AND ') !== false) {
            $parts = preg_split('/\s+AND\s+/i', $str);
            $conditions = array_map(
                fn(string $part) => $this->parseSingleCondition(trim($part)),
                $parts
            );
            return new AndCondition($conditions);
        }

        return $this->parseSingleCondition($str);
    }

    private function parseSingleCondition(string $str): FieldComparison
    {
        if (!preg_match('/(\w+)\s*(>=|<=|!=|=|>|<)\s*(.+)/', trim($str), $m)) {
            throw new RuntimeException("শর্ত পার্স ত্রুটি: {$str}");
        }

        $value = trim($m[3], " \"'");

        // টাইপ রূপান্তর
        if ($value === 'true')  $value = true;
        elseif ($value === 'false') $value = false;
        elseif (is_numeric($value)) $value = (float) $value;

        return new FieldComparison($m[1], $m[2], $value);
    }

    private function parseAction(string $str): ActionExpression
    {
        if (preg_match('/discount\s+([\d.]+)%/i', $str, $m)) {
            return new DiscountAction((float) $m[1]);
        }
        if (stripos($str, 'free shipping') !== false) {
            return new FreeShippingAction();
        }
        throw new RuntimeException("অ্যাকশন পার্স ত্রুটি: {$str}");
    }
}

// === ব্যবহার ===

$parser = new RuleParser();
$engine = new RuleEngine();

// DSL থেকে রুল তৈরি
$engine->addRule($parser->parse('IF total > 5000 AND member = true THEN discount 10%'));
$engine->addRule($parser->parse('IF category = "electronics" AND total > 10000 THEN discount 15%'));
$engine->addRule($parser->parse('IF total > 20000 THEN free shipping'));

// বাংলাদেশি গ্রাহকের অর্ডার
$order = [
    'total'    => 25000,
    'member'   => true,
    'category' => 'electronics',
];

$appliedRules = $engine->evaluate($order);

echo "প্রযোজ্য ডিসকাউন্ট: {$order['_appliedDiscount']}%\n"; // 15%
echo "ফ্রি শিপিং: " . ($order['_freeShipping'] ? 'হ্যাঁ' : 'না') . "\n"; // হ্যাঁ
echo "চূড়ান্ত মূল্য: ৳" . ($order['total'] * (1 - $order['_appliedDiscount'] / 100)) . "\n";

foreach ($order['_discountLog'] as $log) {
    echo "  → {$log}\n";
}
```

#### JavaScript (ES2022+) — সম্পূর্ণ Rule Engine

```javascript
class FieldComparison {
    #field; #operator; #value;

    constructor(field, operator, value) {
        this.#field = field;
        this.#operator = operator;
        this.#value = value;
    }

    evaluate(facts) {
        const actual = facts[this.#field];
        switch (this.#operator) {
            case '>':  return actual > this.#value;
            case '<':  return actual < this.#value;
            case '>=': return actual >= this.#value;
            case '<=': return actual <= this.#value;
            case '=':  return actual == this.#value;
            case '!=': return actual != this.#value;
            default: return false;
        }
    }
}

class AndCondition {
    #conditions;
    constructor(conditions) { this.#conditions = conditions; }
    evaluate(facts) { return this.#conditions.every(c => c.evaluate(facts)); }
}

class DiscountAction {
    #percentage;
    constructor(pct) { this.#percentage = pct; }
    execute(facts) {
        facts._appliedDiscount = Math.max(facts._appliedDiscount ?? 0, this.#percentage);
    }
}

class FreeShippingAction {
    execute(facts) { facts._freeShipping = true; }
}

class Rule {
    #name; #condition; #action;

    constructor(name, condition, action) {
        this.#name = name;
        this.#condition = condition;
        this.#action = action;
    }

    apply(facts) {
        if (this.#condition.evaluate(facts)) {
            this.#action.execute(facts);
            return this.#name;
        }
        return null;
    }
}

class RuleParser {
    parse(text) {
        const match = text.match(/^IF\s+(.+?)\s+THEN\s+(.+)$/i);
        if (!match) throw new Error(`রুল পার্স ত্রুটি: ${text}`);

        return new Rule(text, this.#parseCondition(match[1]), this.#parseAction(match[2]));
    }

    #parseCondition(str) {
        if (str.toUpperCase().includes(' AND ')) {
            const parts = str.split(/\s+AND\s+/i);
            return new AndCondition(parts.map(p => this.#parseSingle(p.trim())));
        }
        return this.#parseSingle(str);
    }

    #parseSingle(str) {
        const m = str.trim().match(/(\w+)\s*(>=|<=|!=|=|>|<)\s*(.+)/);
        if (!m) throw new Error(`শর্ত পার্স ত্রুটি: ${str}`);

        let value = m[3].trim().replace(/["']/g, '');
        if (value === 'true') value = true;
        else if (value === 'false') value = false;
        else if (!isNaN(value)) value = Number(value);

        return new FieldComparison(m[1], m[2], value);
    }

    #parseAction(str) {
        const discountMatch = str.match(/discount\s+([\d.]+)%/i);
        if (discountMatch) return new DiscountAction(Number(discountMatch[1]));
        if (str.toLowerCase().includes('free shipping')) return new FreeShippingAction();
        throw new Error(`অ্যাকশন পার্স ত্রুটি: ${str}`);
    }
}

// ব্যবহার
const parser = new RuleParser();
const rules = [
    parser.parse('IF total > 5000 AND member = true THEN discount 10%'),
    parser.parse('IF category = "electronics" AND total > 10000 THEN discount 15%'),
    parser.parse('IF total > 20000 THEN free shipping'),
];

const order = { total: 25000, member: true, category: 'electronics', _appliedDiscount: 0 };

const applied = rules.map(r => r.apply(order)).filter(Boolean);
console.log(`ডিসকাউন্ট: ${order._appliedDiscount}%`);   // 15%
console.log(`ফ্রি শিপিং: ${order._freeShipping ?? false}`); // true

const finalPrice = order.total * (1 - order._appliedDiscount / 100);
console.log(`চূড়ান্ত মূল্য: ৳${finalPrice}`); // ৳21250
```

---

## 🌍 Real-World Applicable Areas

### ১. Regular Expression Engine
রেগুলার এক্সপ্রেশন হলো Interpreter প্যাটার্নের সবচেয়ে পরিচিত উদাহরণ। `/[a-z]+@[a-z]+\.com/` — এই প্যাটার্নটি একটি মিনি ভাষা যা ইন্টারপ্রেট হয়।

### ২. SQL Parser
প্রতিটি SQL কোয়েরি আসলে একটি ভাষা যা ডেটাবেস ইঞ্জিন ইন্টারপ্রেট করে। `SELECT * FROM users WHERE age > 25` — এখানে `WHERE` ক্লজটি একটি expression tree তৈরি করে।

### ৩. Template Engine (Blade, Twig, Handlebars)
```php
// Blade: {{ $user->name }} কে interpret করে PHP echo-তে রূপান্তর
// Twig: {% for item in items %} — loop expression interpret করে
```

### ৪. Configuration Parser (JSON, YAML, TOML)
YAML ফাইল পার্স করা মূলত একটি ইন্টারপ্রেটেশন প্রক্রিয়া — indentation-based grammar কে data structure-এ রূপান্তর।

### ৫. Domain Specific Languages (DSLs)
- **Dockerfile** — কন্টেইনার বিল্ডের ভাষা
- **Makefile** — বিল্ড অটোমেশনের ভাষা
- **GitHub Actions YAML** — CI/CD পাইপলাইনের ভাষা

### ৬. Business Rule Engine
আমাদের উপরের ডিসকাউন্ট সিস্টেমের মতো — ব্যবসায়িক নিয়ম কোডের বাইরে DSL আকারে লেখা হয়।

### ৭. Math Formula Evaluator
স্প্রেডশীট অ্যাপে `=SUM(A1:A10) * 1.15` — এই ফর্মুলা ইন্টারপ্রেট করা।

### ৮. Routing Pattern Matching (Laravel)
```php
// Route::get('/user/{id}/post/{postId}', ...);
// '/user/42/post/7' কে interpret করে id=42, postId=7 বের করা
```

### ৯. Validation Rule Parsing
```php
// Laravel: 'required|string|max:255|email'
// প্রতিটি pipe-separated অংশ একটি expression যা interpret হয়
```

### ১০. Bangla Text Number Parsing
```
"পাঁচ হাজার তিনশত বিশ" → 5320
"দুই লক্ষ পঞ্চাশ হাজার" → 250000
```

---

## 🔥 Advanced Deep Dive

### Interpreter vs Visitor — AST প্রসেসিংয়ের জন্য তুলনা

| বৈশিষ্ট্য | Interpreter | Visitor |
|---|---|---|
| **অপারেশনের অবস্থান** | প্রতিটি নোডে `interpret()` থাকে | আলাদা Visitor ক্লাসে অপারেশন থাকে |
| **নতুন নোড যোগ** | সহজ — নতুন ক্লাস তৈরি | কঠিন — সকল Visitor আপডেট করতে হয় |
| **নতুন অপারেশন যোগ** | কঠিন — সকল নোড আপডেট | সহজ — নতুন Visitor তৈরি |
| **ব্যবহারের ক্ষেত্র** | ব্যাকরণ স্থিতিশীল, অপারেশন একটি | ব্যাকরণ স্থিতিশীল, অপারেশন অনেক |
| **জটিলতা** | সহজ | জটিল (double dispatch) |

```php
<?php
// Visitor পদ্ধতি — যখন একাধিক operation দরকার (evaluate, print, optimize)
interface ExpressionVisitor
{
    public function visitNumber(NumberNode $node): mixed;
    public function visitAdd(AddNode $node): mixed;
    public function visitMultiply(MultiplyNode $node): mixed;
}

interface VisitableExpression
{
    public function accept(ExpressionVisitor $visitor): mixed;
}

class NumberNode implements VisitableExpression
{
    public function __construct(public readonly float $value) {}

    public function accept(ExpressionVisitor $visitor): mixed
    {
        return $visitor->visitNumber($this);
    }
}

class AddNode implements VisitableExpression
{
    public function __construct(
        public readonly VisitableExpression $left,
        public readonly VisitableExpression $right,
    ) {}

    public function accept(ExpressionVisitor $visitor): mixed
    {
        return $visitor->visitAdd($this);
    }
}

class MultiplyNode implements VisitableExpression
{
    public function __construct(
        public readonly VisitableExpression $left,
        public readonly VisitableExpression $right,
    ) {}

    public function accept(ExpressionVisitor $visitor): mixed
    {
        return $visitor->visitMultiply($this);
    }
}

// Evaluate Visitor
class EvaluateVisitor implements ExpressionVisitor
{
    public function visitNumber(NumberNode $node): float
    {
        return $node->value;
    }

    public function visitAdd(AddNode $node): float
    {
        return $node->left->accept($this) + $node->right->accept($this);
    }

    public function visitMultiply(MultiplyNode $node): float
    {
        return $node->left->accept($this) * $node->right->accept($this);
    }
}

// Print Visitor — একই AST, ভিন্ন অপারেশন
class PrintVisitor implements ExpressionVisitor
{
    public function visitNumber(NumberNode $node): string
    {
        return (string) $node->value;
    }

    public function visitAdd(AddNode $node): string
    {
        return "({$node->left->accept($this)} + {$node->right->accept($this)})";
    }

    public function visitMultiply(MultiplyNode $node): string
    {
        return "({$node->left->accept($this)} × {$node->right->accept($this)})";
    }
}
```

**সিদ্ধান্ত:** যদি শুধু evaluate করতে হয় → **Interpreter**। যদি evaluate, print, optimize, compile সব দরকার → **Visitor**।

---

### Interpreter + Composite — Expression Tree

Interpreter প্যাটার্ন প্রকৃতপক্ষে **Composite প্যাটার্নের একটি বিশেষায়িত রূপ**। NonTerminalExpression গুলো Composite node আর TerminalExpression গুলো Leaf node।

```
          AddExpr          ← Composite (NonTerminal)
         /       \
    NumExpr(2)   MulExpr   ← Composite (NonTerminal)
                /      \
          NumExpr(3)  NumExpr(4)  ← Leaf (Terminal)
```

---

### Interpreter + Flyweight — শেয়ার্ড Terminal Expression

যখন একই terminal expression বারবার ব্যবহৃত হয়, Flyweight প্যাটার্ন মেমোরি সাশ্রয় করে:

```php
<?php

class ExpressionFactory
{
    private static array $pool = [];

    public static function number(int|float $value): NumberExpression
    {
        $key = "num_{$value}";
        if (!isset(self::$pool[$key])) {
            self::$pool[$key] = new NumberExpression($value);
        }
        return self::$pool[$key];
    }

    public static function variable(string $name): VariableExpression
    {
        $key = "var_{$name}";
        if (!isset(self::$pool[$key])) {
            self::$pool[$key] = new VariableExpression($name);
        }
        return self::$pool[$key];
    }

    public static function poolSize(): int
    {
        return count(self::$pool);
    }
}

// NumberExpression(0) এবং NumberExpression(1) বারবার ব্যবহৃত হলে
// একবারই তৈরি হবে, বাকি সময় ক্যাশ থেকে আসবে
$zero = ExpressionFactory::number(0);
$one  = ExpressionFactory::number(1);
$sameZero = ExpressionFactory::number(0); // ক্যাশ থেকে

var_dump($zero === $sameZero); // true — একই অবজেক্ট
```

---

### Recursive Descent Parsing — সম্পূর্ণ Math Parser

প্রকৃত expression parser তৈরি করি যা `"2 + 3 * (4 - 1)"` স্ট্রিং থেকে AST তৈরি করে:

```php
<?php

declare(strict_types=1);

class Token
{
    public function __construct(
        public readonly string $type,
        public readonly string $value,
    ) {}
}

class Lexer
{
    private int $pos = 0;

    public function __construct(private readonly string $input) {}

    /** @return Token[] */
    public function tokenize(): array
    {
        $tokens = [];
        while ($this->pos < strlen($this->input)) {
            $ch = $this->input[$this->pos];

            if (ctype_space($ch)) {
                $this->pos++;
                continue;
            }

            if (ctype_digit($ch) || $ch === '.') {
                $num = '';
                while ($this->pos < strlen($this->input)
                    && (ctype_digit($this->input[$this->pos]) || $this->input[$this->pos] === '.')) {
                    $num .= $this->input[$this->pos++];
                }
                $tokens[] = new Token('NUMBER', $num);
                continue;
            }

            $tokens[] = match ($ch) {
                '+' => new Token('PLUS', $ch),
                '-' => new Token('MINUS', $ch),
                '*' => new Token('MUL', $ch),
                '/' => new Token('DIV', $ch),
                '(' => new Token('LPAREN', $ch),
                ')' => new Token('RPAREN', $ch),
                default => throw new RuntimeException("অজানা ক্যারেক্টার: {$ch}"),
            };
            $this->pos++;
        }

        $tokens[] = new Token('EOF', '');
        return $tokens;
    }
}

class RecursiveDescentParser
{
    private int $pos = 0;
    /** @var Token[] */
    private array $tokens;

    public function parse(string $input): Expression
    {
        $this->tokens = (new Lexer($input))->tokenize();
        $this->pos = 0;

        $result = $this->parseExpression();

        if ($this->current()->type !== 'EOF') {
            throw new RuntimeException('অপ্রত্যাশিত টোকেন: ' . $this->current()->value);
        }

        return $result;
    }

    // expression = term (('+' | '-') term)*
    private function parseExpression(): Expression
    {
        $left = $this->parseTerm();

        while (in_array($this->current()->type, ['PLUS', 'MINUS'])) {
            $op = $this->current()->type;
            $this->advance();
            $right = $this->parseTerm();

            $left = $op === 'PLUS'
                ? new AddExpression($left, $right)
                : new SubtractExpression($left, $right);
        }

        return $left;
    }

    // term = factor (('*' | '/') factor)*
    private function parseTerm(): Expression
    {
        $left = $this->parseFactor();

        while (in_array($this->current()->type, ['MUL', 'DIV'])) {
            $op = $this->current()->type;
            $this->advance();
            $right = $this->parseFactor();

            $left = $op === 'MUL'
                ? new MultiplyExpression($left, $right)
                : new DivideExpression($left, $right);
        }

        return $left;
    }

    // factor = NUMBER | '(' expression ')'
    private function parseFactor(): Expression
    {
        $token = $this->current();

        if ($token->type === 'NUMBER') {
            $this->advance();
            return new NumberExpression((float) $token->value);
        }

        if ($token->type === 'LPAREN') {
            $this->advance();
            $expr = $this->parseExpression();
            if ($this->current()->type !== 'RPAREN') {
                throw new RuntimeException('বন্ধনী বন্ধ হয়নি');
            }
            $this->advance();
            return $expr;
        }

        throw new RuntimeException("অপ্রত্যাশিত: {$token->value}");
    }

    private function current(): Token
    {
        return $this->tokens[$this->pos];
    }

    private function advance(): void
    {
        $this->pos++;
    }
}

class DivideExpression implements Expression
{
    public function __construct(
        private readonly Expression $left,
        private readonly Expression $right,
    ) {}

    public function interpret(Context $context): int|float
    {
        $divisor = $this->right->interpret($context);
        if ($divisor == 0) throw new RuntimeException('শূন্য দিয়ে ভাগ করা যায় না');
        return $this->left->interpret($context) / $divisor;
    }
}

// ব্যবহার
$parser = new RecursiveDescentParser();
$context = new Context();

$expr = $parser->parse('2 + 3 * (4 - 1)');
echo $expr->interpret($context); // 11

$expr2 = $parser->parse('(10 + 5) * 2 / 3');
echo "\n" . $expr2->interpret($context); // 10
```

```javascript
// JavaScript Recursive Descent Parser
class Lexer {
    #input; #pos = 0;

    constructor(input) { this.#input = input; }

    tokenize() {
        const tokens = [];
        while (this.#pos < this.#input.length) {
            const ch = this.#input[this.#pos];
            if (/\s/.test(ch)) { this.#pos++; continue; }

            if (/[\d.]/.test(ch)) {
                let num = '';
                while (this.#pos < this.#input.length && /[\d.]/.test(this.#input[this.#pos])) {
                    num += this.#input[this.#pos++];
                }
                tokens.push({ type: 'NUMBER', value: num });
                continue;
            }

            const typeMap = {
                '+': 'PLUS', '-': 'MINUS', '*': 'MUL',
                '/': 'DIV',  '(': 'LPAREN', ')': 'RPAREN'
            };
            if (typeMap[ch]) {
                tokens.push({ type: typeMap[ch], value: ch });
                this.#pos++;
            } else {
                throw new Error(`অজানা ক্যারেক্টার: ${ch}`);
            }
        }
        tokens.push({ type: 'EOF', value: '' });
        return tokens;
    }
}

class MathParser {
    #tokens; #pos = 0;

    parse(input) {
        this.#tokens = new Lexer(input).tokenize();
        this.#pos = 0;
        const result = this.#parseExpr();
        if (this.#current().type !== 'EOF') throw new Error('অপ্রত্যাশিত টোকেন');
        return result;
    }

    #parseExpr() {
        let left = this.#parseTerm();
        while (['PLUS', 'MINUS'].includes(this.#current().type)) {
            const op = this.#current().type;
            this.#pos++;
            const right = this.#parseTerm();
            left = op === 'PLUS'
                ? new AddExpression(left, right)
                : { interpret: (ctx) => left.interpret(ctx) - right.interpret(ctx) };
        }
        return left;
    }

    #parseTerm() {
        let left = this.#parseFactor();
        while (['MUL', 'DIV'].includes(this.#current().type)) {
            const op = this.#current().type;
            this.#pos++;
            const right = this.#parseFactor();
            left = op === 'MUL'
                ? new MultiplyExpression(left, right)
                : { interpret: (ctx) => left.interpret(ctx) / right.interpret(ctx) };
        }
        return left;
    }

    #parseFactor() {
        const tok = this.#current();
        if (tok.type === 'NUMBER') {
            this.#pos++;
            return new NumberExpression(Number(tok.value));
        }
        if (tok.type === 'LPAREN') {
            this.#pos++;
            const expr = this.#parseExpr();
            if (this.#current().type !== 'RPAREN') throw new Error('বন্ধনী বন্ধ হয়নি');
            this.#pos++;
            return expr;
        }
        throw new Error(`অপ্রত্যাশিত: ${tok.value}`);
    }

    #current() { return this.#tokens[this.#pos]; }
}

const mathParser = new MathParser();
const ctx = new Context();
console.log(mathParser.parse('2 + 3 * (4 - 1)').interpret(ctx)); // 11
```

---

### Building a Simple DSL — বাংলাদেশি ব্যবসার জন্য

একটি সরল DSL তৈরি করি যা বাংলাদেশি ই-কমার্সে ব্যবহার করা যায়:

```
WHEN order.total >= 5000
  AND order.items_count > 3
  AND customer.tier = "gold"
APPLY discount 12%
  AND free_shipping
```

```php
<?php

declare(strict_types=1);

enum TokenType: string
{
    case WHEN    = 'WHEN';
    case AND     = 'AND';
    case APPLY   = 'APPLY';
    case IDENT   = 'IDENT';
    case NUMBER  = 'NUMBER';
    case STRING  = 'STRING';
    case OP      = 'OP';
    case KEYWORD = 'KEYWORD';
    case EOF     = 'EOF';
}

class DSLInterpreter
{
    public function execute(string $dsl, array $context): array
    {
        $lines = array_filter(
            array_map('trim', explode("\n", $dsl)),
            fn(string $line) => $line !== ''
        );

        $conditions = [];
        $actions    = [];
        $section    = 'condition';

        foreach ($lines as $line) {
            $cleaned = preg_replace('/^(WHEN|AND)\s+/i', '', $line);

            if (str_starts_with(strtoupper($line), 'APPLY')) {
                $section = 'action';
                $cleaned = preg_replace('/^APPLY\s+/i', '', $line);
            }

            if ($section === 'condition') {
                $conditions[] = $this->parseCondition($cleaned);
            } else {
                $cleanedAction = preg_replace('/^AND\s+/i', '', $cleaned);
                $actions[] = $this->parseAction($cleanedAction);
            }
        }

        // সকল শর্ত পরীক্ষা
        $allMet = true;
        foreach ($conditions as $condition) {
            if (!$condition($context)) {
                $allMet = false;
                break;
            }
        }

        $result = ['matched' => $allMet, 'actions' => []];

        if ($allMet) {
            foreach ($actions as $action) {
                $result['actions'][] = $action($context);
            }
        }

        return $result;
    }

    private function parseCondition(string $str): Closure
    {
        if (!preg_match('/([\w.]+)\s*(>=|<=|!=|=|>|<)\s*(.+)/', $str, $m)) {
            throw new RuntimeException("শর্ত পার্স ত্রুটি: {$str}");
        }

        $path = $m[1];
        $op   = $m[2];
        $val  = trim($m[3], " \"'");
        if (is_numeric($val)) $val = (float) $val;

        return function (array $ctx) use ($path, $op, $val): bool {
            $actual = $this->resolvePath($ctx, $path);
            return match ($op) {
                '>'  => $actual > $val,
                '>=' => $actual >= $val,
                '<'  => $actual < $val,
                '<=' => $actual <= $val,
                '='  => $actual == $val,
                '!=' => $actual != $val,
            };
        };
    }

    private function parseAction(string $str): Closure
    {
        if (preg_match('/discount\s+([\d.]+)%/i', $str, $m)) {
            $pct = (float) $m[1];
            return fn(array $ctx) => ['type' => 'discount', 'value' => $pct];
        }
        if (stripos($str, 'free_shipping') !== false) {
            return fn(array $ctx) => ['type' => 'free_shipping'];
        }
        throw new RuntimeException("অ্যাকশন পার্স ত্রুটি: {$str}");
    }

    private function resolvePath(array $ctx, string $path): mixed
    {
        $parts = explode('.', $path);
        $current = $ctx;
        foreach ($parts as $part) {
            $current = $current[$part] ?? null;
        }
        return $current;
    }
}

// ব্যবহার
$dsl = <<<DSL
WHEN order.total >= 5000
AND order.items_count > 3
AND customer.tier = "gold"
APPLY discount 12%
AND free_shipping
DSL;

$interpreter = new DSLInterpreter();

$result = $interpreter->execute($dsl, [
    'order' => ['total' => 8500, 'items_count' => 5],
    'customer' => ['tier' => 'gold', 'name' => 'রহিম উদ্দিন'],
]);

print_r($result);
// matched => true, actions => [discount 12%, free_shipping]
```

---

### Performance Considerations — কখন Interpreter ব্যবহার করবেন না

**⚠️ Interpreter প্যাটার্নের সীমাবদ্ধতা:**

1. **জটিল ব্যাকরণ:** যদি ব্যাকরণ বড় হয় (যেমন: পূর্ণাঙ্গ প্রোগ্রামিং ভাষা), তাহলে ক্লাসের সংখ্যা অনিয়ন্ত্রিতভাবে বাড়বে। এক্ষেত্রে parser generator (ANTLR, PEG.js) ব্যবহার করুন।

2. **পারফরম্যান্স:** প্রতিটি expression evaluate-এ virtual method call হয়। হাজার হাজার expression-এ এটি ধীর হতে পারে। বিকল্প: expression কে bytecode-এ কম্পাইল করুন।

3. **গভীর recursion:** বড় expression tree-তে stack overflow হতে পারে। বিকল্প: iterative evaluation ব্যবহার করুন।

```php
// ❌ খারাপ — প্রতিটি request-এ রুল পার্স করা
function processOrder(array $order): float {
    $parser = new RuleParser();
    $rule = $parser->parse('IF total > 5000 THEN discount 10%');  // প্রতিবার পার্স!
    // ...
}

// ✅ ভালো — একবার পার্স, বারবার ব্যবহার (cache)
class CachedRuleEngine {
    private static array $parsedRules = [];

    public static function getRule(string $ruleText): Rule {
        return self::$parsedRules[$ruleText]
            ??= (new RuleParser())->parse($ruleText);
    }
}
```

---

### Laravel Validation Rule Parsing — অভ্যন্তরীণ কার্যপ্রণালী

Laravel-এর validation system আসলে Interpreter প্যাটার্নের একটি চমৎকার বাস্তবায়ন:

```php
// Laravel ভ্যালিডেশন রুল:
// 'email' => 'required|string|email|max:255|unique:users'

// অভ্যন্তরীণভাবে যা ঘটে:
// 1. স্ট্রিং টোকেনাইজ হয় — pipe (|) দিয়ে বিভক্ত
// 2. প্রতিটি টোকেন একটি Rule object-এ রূপান্তরিত হয়
// 3. প্রতিটি Rule object-এর passes() মেথড কল হয়

// সরলীকৃত রূপ:
interface ValidationRule
{
    public function passes(string $attribute, mixed $value): bool;
    public function message(): string;
}

class RequiredRule implements ValidationRule
{
    public function passes(string $attribute, mixed $value): bool
    {
        return $value !== null && $value !== '';
    }

    public function message(): string
    {
        return ':attribute ফিল্ডটি আবশ্যক';
    }
}

class MaxRule implements ValidationRule
{
    public function __construct(private readonly int $max) {}

    public function passes(string $attribute, mixed $value): bool
    {
        return is_string($value)
            ? mb_strlen($value) <= $this->max
            : $value <= $this->max;
    }

    public function message(): string
    {
        return ":attribute সর্বোচ্চ {$this->max} হতে পারবে";
    }
}

class ValidationRuleParser
{
    public static function parse(string $rules): array
    {
        return array_map(function (string $rule) {
            $parts = explode(':', $rule, 2);
            $name = $parts[0];
            $params = isset($parts[1]) ? explode(',', $parts[1]) : [];

            return match ($name) {
                'required' => new RequiredRule(),
                'max'      => new MaxRule((int) $params[0]),
                'string'   => new class implements ValidationRule {
                    public function passes(string $attr, mixed $val): bool {
                        return is_string($val);
                    }
                    public function message(): string {
                        return ':attribute অবশ্যই স্ট্রিং হতে হবে';
                    }
                },
                default => throw new RuntimeException("অজানা রুল: {$name}"),
            };
        }, explode('|', $rules));
    }
}

// ব্যবহার
$rules = ValidationRuleParser::parse('required|string|max:255');
$value = 'রহিম উদ্দিন';

foreach ($rules as $rule) {
    echo $rule->passes('name', $value) ? "✅ পাস\n" : "❌ " . $rule->message() . "\n";
}
```

---

## ✅ Pros ও ❌ Cons

### ✅ সুবিধাসমূহ

| সুবিধা | ব্যাখ্যা |
|---|---|
| **ব্যাকরণ সহজে পরিবর্তনযোগ্য** | নতুন expression যোগ করতে শুধু একটি নতুন ক্লাস তৈরি করলেই হয় |
| **Open/Closed Principle** | বিদ্যমান কোড পরিবর্তন না করে নতুন ব্যাকরণ যোগ করা যায় |
| **সহজ ব্যাকরণের জন্য চমৎকার** | ছোট DSL, রুল ইঞ্জিন, ভ্যালিডেশন রুলের জন্য আদর্শ |
| **টেস্ট করা সহজ** | প্রতিটি expression স্বতন্ত্রভাবে টেস্ট করা যায় |
| **কনফিগারেশন-চালিত** | ব্যবসায়িক নিয়ম কোডে হার্ডকোড না করে বাইরে রাখা যায় |

### ❌ অসুবিধাসমূহ

| অসুবিধা | ব্যাখ্যা |
|---|---|
| **জটিল ব্যাকরণে অকার্যকর** | ব্যাকরণ বড় হলে ক্লাস বিস্ফোরণ ঘটে |
| **পারফরম্যান্স** | প্রতিটি নোডে method call — বড় tree-তে ধীর |
| **ডিবাগিং কঠিন** | গভীর recursion trace করা জটিল |
| **পার্সিং আলাদা সমস্যা** | প্যাটার্ন শুধু interpret করে — parse করা ক্লায়েন্টের দায়িত্ব |
| **মেমোরি ব্যবহার** | প্রতিটি expression-এর জন্য আলাদা অবজেক্ট তৈরি হয় |

---

## ⚠️ Common Mistakes

### ১. সবকিছুতে Interpreter ব্যবহার করা
```php
// ❌ সাধারণ if-else-এর জন্য interpreter দরকার নেই
$expr = new AndExpression(
    new ComparisonExpression('age', '>', 18),
    new ComparisonExpression('country', '=', 'BD'),
);
if ($expr->interpret($data)) { ... }

// ✅ সরাসরি if-else ব্যবহার করুন
if ($data['age'] > 18 && $data['country'] === 'BD') { ... }
```
**নিয়ম:** Interpreter ব্যবহার করুন শুধুমাত্র যখন expression **ডায়নামিক** — অর্থাৎ রানটাইমে ব্যবহারকারী বা কনফিগারেশন থেকে আসে।

### ২. পার্সিং ও ইন্টারপ্রেটিং মিশ্রিত করা
```php
// ❌ একই ক্লাসে পার্স ও ইন্টারপ্রেট
class BadExpression {
    public function evaluate(string $rawExpression): mixed {
        // পার্স এবং evaluate একসাথে — Separation of Concerns ভঙ্গ
    }
}

// ✅ আলাদা করুন: Parser → AST → Interpreter
$ast = $parser->parse($rawExpression);    // স্ট্রিং → AST
$result = $ast->interpret($context);       // AST → ফলাফল
```

### ৩. Context ছাড়া কাজ করা
```php
// ❌ Context ব্যবহার না করা — ভেরিয়েবল ম্যানেজমেন্ট কঠিন
class NumberExpr {
    public function interpret(): int { ... }  // Context নেই!
}

// ✅ সবসময় Context ব্যবহার করুন
class NumberExpr {
    public function interpret(Context $ctx): int { ... }
}
```

### ৪. Error handling বাদ দেওয়া
```php
// ❌ কোনো ভুল ইনপুটে crash
$parser->parse('invalid!!!');

// ✅ সুন্দর error message সহ ব্যতিক্রম
try {
    $ast = $parser->parse($input);
} catch (ParseException $e) {
    echo "সিনট্যাক্স ত্রুটি: {$e->getMessage()} (অবস্থান: {$e->getPosition()})";
}
```

### ৫. বারবার পার্স করা (ক্যাশ না করা)
প্রতিটি request-এ একই রুল বারবার পার্স করা অপচয়। একবার পার্স করে AST ক্যাশ করুন।

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class MathInterpreterTest extends TestCase
{
    private Context $context;

    protected function setUp(): void
    {
        $this->context = new Context();
    }

    #[Test]
    public function numberExpressionReturnsValue(): void
    {
        $expr = new NumberExpression(42);
        $this->assertSame(42, $expr->interpret($this->context));
    }

    #[Test]
    public function addExpressionSumsTwoNumbers(): void
    {
        $expr = new AddExpression(
            new NumberExpression(10),
            new NumberExpression(20)
        );
        $this->assertSame(30, $expr->interpret($this->context));
    }

    #[Test]
    public function operatorPrecedenceIsRespected(): void
    {
        // 2 + 3 * 4 = 14 (গুণ আগে)
        $expr = new AddExpression(
            new NumberExpression(2),
            new MultiplyExpression(
                new NumberExpression(3),
                new NumberExpression(4)
            )
        );
        $this->assertEquals(14, $expr->interpret($this->context));
    }

    #[Test]
    public function variableExpressionLooksUpContext(): void
    {
        $this->context->assign('x', 100);
        $expr = new VariableExpression('x');
        $this->assertSame(100, $expr->interpret($this->context));
    }

    #[Test]
    public function undefinedVariableThrowsException(): void
    {
        $this->expectException(RuntimeException::class);
        $this->expectExceptionMessage('অসংজ্ঞায়িত ভেরিয়েবল');

        $expr = new VariableExpression('unknown');
        $expr->interpret($this->context);
    }

    #[Test]
    public function complexNestedExpression(): void
    {
        // (x + 5) * (y - 2) where x=10, y=7
        $this->context->assign('x', 10);
        $this->context->assign('y', 7);

        $expr = new MultiplyExpression(
            new AddExpression(
                new VariableExpression('x'),
                new NumberExpression(5)
            ),
            new SubtractExpression(
                new VariableExpression('y'),
                new NumberExpression(2)
            )
        );

        $this->assertEquals(75, $expr->interpret($this->context)); // 15 * 5
    }

    #[Test]
    public function recursiveDescentParserProducesCorrectAST(): void
    {
        $parser = new RecursiveDescentParser();

        $this->assertEquals(11, $parser->parse('2 + 3 * (4 - 1)')->interpret($this->context));
        $this->assertEquals(14, $parser->parse('2 + 3 * 4')->interpret($this->context));
        $this->assertEquals(20, $parser->parse('(2 + 3) * 4')->interpret($this->context));
    }
}

class BooleanInterpreterTest extends TestCase
{
    #[Test]
    public function andExpressionReturnsTrueWhenBothTrue(): void
    {
        $expr = new AndExpression(
            new BooleanConstant(true),
            new BooleanConstant(true)
        );
        $this->assertTrue($expr->interpret([]));
    }

    #[Test]
    public function complexBooleanExpression(): void
    {
        // true AND (false OR true) = true
        $expr = new AndExpression(
            new BooleanConstant(true),
            new OrExpression(
                new BooleanConstant(false),
                new BooleanConstant(true)
            )
        );
        $this->assertTrue($expr->interpret([]));
    }

    #[Test]
    public function notExpressionInvertsBooleanValue(): void
    {
        $expr = new NotExpression(new BooleanConstant(true));
        $this->assertFalse($expr->interpret([]));
    }

    #[Test]
    public function booleanVariablesFromContext(): void
    {
        $ctx = ['isPremium' => true, 'isBlocked' => false];

        $expr = new AndExpression(
            new BooleanVariable('isPremium'),
            new NotExpression(new BooleanVariable('isBlocked'))
        );

        $this->assertTrue($expr->interpret($ctx));
    }
}

class RuleEngineTest extends TestCase
{
    #[Test]
    public function discountRuleAppliesCorrectly(): void
    {
        $parser = new RuleParser();
        $engine = new RuleEngine();

        $engine->addRule($parser->parse('IF total > 5000 AND member = true THEN discount 10%'));

        $order = ['total' => 8000, 'member' => true];
        $engine->evaluate($order);

        $this->assertEquals(10, $order['_appliedDiscount']);
    }

    #[Test]
    public function higherDiscountWins(): void
    {
        $parser = new RuleParser();
        $engine = new RuleEngine();

        $engine->addRule($parser->parse('IF total > 5000 THEN discount 10%'));
        $engine->addRule($parser->parse('IF total > 10000 THEN discount 15%'));

        $order = ['total' => 15000];
        $engine->evaluate($order);

        $this->assertEquals(15, $order['_appliedDiscount']);
    }

    #[Test]
    public function noRuleMatchedMeansZeroDiscount(): void
    {
        $parser = new RuleParser();
        $engine = new RuleEngine();

        $engine->addRule($parser->parse('IF total > 5000 THEN discount 10%'));

        $order = ['total' => 2000];
        $engine->evaluate($order);

        $this->assertEquals(0, $order['_appliedDiscount']);
    }

    #[Test]
    public function invalidRuleThrowsException(): void
    {
        $parser = new RuleParser();
        $this->expectException(RuntimeException::class);
        $parser->parse('INVALID RULE TEXT');
    }
}
```

### Jest (JavaScript)

```javascript
import { describe, it, expect, beforeEach } from '@jest/globals';

describe('Math Interpreter', () => {
    let context;

    beforeEach(() => {
        context = new Context();
    });

    it('NumberExpression মান ফেরত দেয়', () => {
        const expr = new NumberExpression(42);
        expect(expr.interpret(context)).toBe(42);
    });

    it('AddExpression দুটি সংখ্যা যোগ করে', () => {
        const expr = new AddExpression(
            new NumberExpression(10),
            new NumberExpression(20)
        );
        expect(expr.interpret(context)).toBe(30);
    });

    it('operator precedence সঠিকভাবে কাজ করে', () => {
        // 2 + 3 * 4 = 14
        const expr = new AddExpression(
            new NumberExpression(2),
            new MultiplyExpression(
                new NumberExpression(3),
                new NumberExpression(4)
            )
        );
        expect(expr.interpret(context)).toBe(14);
    });

    it('VariableExpression context থেকে মান খোঁজে', () => {
        context.assign('x', 100);
        const expr = new VariableExpression('x');
        expect(expr.interpret(context)).toBe(100);
    });

    it('অসংজ্ঞায়িত ভেরিয়েবলে Error throw হয়', () => {
        const expr = new VariableExpression('unknown');
        expect(() => expr.interpret(context)).toThrow('অসংজ্ঞায়িত ভেরিয়েবল');
    });
});

describe('Boolean Interpreter', () => {
    it('AND expression — দুটি true হলে true', () => {
        const expr = new AndExpression(
            new BooleanConstant(true),
            new BooleanConstant(true)
        );
        expect(expr.interpret({})).toBe(true);
    });

    it('true AND (false OR true) = true', () => {
        const expr = new AndExpression(
            new BooleanConstant(true),
            new OrExpression(
                new BooleanConstant(false),
                new BooleanConstant(true)
            )
        );
        expect(expr.interpret({})).toBe(true);
    });

    it('NOT expression মান উল্টে দেয়', () => {
        const expr = new NotExpression(new BooleanConstant(true));
        expect(expr.interpret({})).toBe(false);
    });
});

describe('Rule Engine', () => {
    it('ডিসকাউন্ট রুল সঠিকভাবে প্রয়োগ হয়', () => {
        const parser = new RuleParser();
        const rules = [
            parser.parse('IF total > 5000 AND member = true THEN discount 10%'),
        ];

        const order = { total: 8000, member: true, _appliedDiscount: 0 };
        rules.forEach(r => r.apply(order));

        expect(order._appliedDiscount).toBe(10);
    });

    it('শর্ত পূরণ না হলে কোনো ডিসকাউন্ট নেই', () => {
        const parser = new RuleParser();
        const rules = [
            parser.parse('IF total > 5000 THEN discount 10%'),
        ];

        const order = { total: 2000, _appliedDiscount: 0 };
        rules.forEach(r => r.apply(order));

        expect(order._appliedDiscount).toBe(0);
    });

    it('ভুল রুল টেক্সটে Error throw হয়', () => {
        const parser = new RuleParser();
        expect(() => parser.parse('INVALID RULE')).toThrow();
    });
});

describe('Recursive Descent Parser', () => {
    it('2 + 3 * (4 - 1) = 11', () => {
        const parser = new MathParser();
        const ctx = new Context();
        expect(parser.parse('2 + 3 * (4 - 1)').interpret(ctx)).toBe(11);
    });

    it('(2 + 3) * 4 = 20', () => {
        const parser = new MathParser();
        const ctx = new Context();
        expect(parser.parse('(2 + 3) * 4').interpret(ctx)).toBe(20);
    });

    it('ভুল expression-এ Error throw হয়', () => {
        const parser = new MathParser();
        expect(() => parser.parse('2 ++ 3')).toThrow();
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Composite Pattern
Interpreter প্যাটার্নের AST মূলত একটি **Composite structure**। NonTerminalExpression হলো Composite node যা অন্যান্য expression ধারণ করে, আর TerminalExpression হলো Leaf node।

### Visitor Pattern
যখন একই AST-এর উপর **একাধিক অপারেশন** দরকার (evaluate, print, optimize, compile), তখন Interpreter-এর বদলে Visitor ব্যবহার করুন। Visitor নতুন অপারেশন যোগ করা সহজ করে।

### Flyweight Pattern
Terminal expression গুলো প্রায়ই পুনরাবৃত্ত হয়। Flyweight প্যাটার্ন ব্যবহার করে এগুলো শেয়ার করা যায়, যা মেমোরি সাশ্রয় করে।

### Iterator Pattern
AST traverse করতে Iterator ব্যবহার করা যায়। বিশেষত যখন tree-র উপর বিভিন্ন traversal strategy (in-order, pre-order, post-order) দরকার।

```
┌────────────────┐     ┌─────────────────┐
│   Composite    │────▶│   Interpreter   │
│  (Tree গঠন)    │     │  (Tree মূল্যায়ন) │
└────────────────┘     └────────┬────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
             ┌──────────┐ ┌──────────┐ ┌──────────┐
             │ Visitor  │ │ Flyweight│ │ Iterator │
             │(বহু ops) │ │(শেয়ার্ড) │ │(traverse)│
             └──────────┘ └──────────┘ └──────────┘
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

1. **সহজ ব্যাকরণ আছে** — ব্যাকরণের নিয়ম কম এবং expression ছোট
2. **দক্ষতা সমালোচনামূলক নয়** — পারফরম্যান্স প্রধান উদ্বেগ নয়
3. **Expression ডায়নামিক** — ব্যবহারকারী বা কনফিগারেশন থেকে expression আসে
4. **ব্যবসায়িক নিয়ম ঘন ঘন পরিবর্তন হয়** — কোড ডিপ্লয় ছাড়াই নিয়ম আপডেট করতে চান
5. **DSL তৈরি করতে চান** — ডোমেইন-নির্দিষ্ট ভাষা যেমন রুল ইঞ্জিন, কোয়েরি বিল্ডার

### ❌ ব্যবহার করবেন না যখন:

1. **ব্যাকরণ জটিল** — পূর্ণাঙ্গ প্রোগ্রামিং ভাষার জন্য ANTLR/PEG ব্যবহার করুন
2. **পারফরম্যান্স গুরুত্বপূর্ণ** — hot path-এ বারবার interpret করতে হলে কম্পাইলার পদ্ধতি ভালো
3. **নিয়ম স্থিতিশীল** — যদি নিয়ম কখনো পরিবর্তন না হয়, সরাসরি কোডে লিখুন
4. **শুধু একটি expression type** — একটিমাত্র তুলনার জন্য পুরো প্যাটার্ন অতিরিক্ত
5. **টিম অপরিচিত** — যদি দল এই প্যাটার্নে অভ্যস্ত না হয়, কোড বোঝা কঠিন হবে

### সিদ্ধান্ত ফ্লোচার্ট

```
Expression কি রানটাইমে ডায়নামিক?
├─ না → সরাসরি কোডে লিখুন
└─ হ্যাঁ
   └─ ব্যাকরণ কি সহজ? (< 15 rules)
      ├─ না → Parser Generator ব্যবহার করুন (ANTLR/PEG)
      └─ হ্যাঁ
         └─ পারফরম্যান্স কি critical?
            ├─ হ্যাঁ → Compile to bytecode / cache AST
            └─ না → ✅ Interpreter প্যাটার্ন ব্যবহার করুন
```

---

## 📋 সারসংক্ষেপ

| বিষয় | বিবরণ |
|---|---|
| **ধরন** | Behavioral Design Pattern |
| **উদ্দেশ্য** | একটি ভাষার ব্যাকরণ উপস্থাপন ও ব্যাখ্যা করা |
| **মূল অংশ** | AbstractExpression, TerminalExpression, NonTerminalExpression, Context |
| **সম্পর্কিত প্যাটার্ন** | Composite, Visitor, Flyweight, Iterator |
| **শক্তি** | ডায়নামিক expression, DSL, রুল ইঞ্জিন |
| **দুর্বলতা** | জটিল ব্যাকরণে অকার্যকর, পারফরম্যান্স |
| **আদর্শ ব্যবহার** | ব্যবসায়িক রুল ইঞ্জিন, ভ্যালিডেশন পার্সিং, কোয়েরি বিল্ডার |
| **এড়িয়ে চলুন** | পূর্ণ ভাষা পার্সার, hot path, স্থিতিশীল নিয়ম |

### মনে রাখার সূত্র

> **"ব্যাকরণ যদি ছোট হয়, নিয়ম যদি বদলায়,**
> **Interpreter প্যাটার্ন তখনই কাজে লাগায়।"**

Interpreter প্যাটার্ন বাংলাদেশি সফটওয়্যার শিল্পে বিশেষভাবে কার্যকর — ই-কমার্স ডিসকাউন্ট সিস্টেম, ব্যাংকিং লোন ক্যালকুলেটর, কাস্টম রিপোর্টিং কোয়েরি, এবং ব্যবসায়িক নিয়ম ইঞ্জিনে। মূল কথা হলো — যখনই আপনি দেখবেন ব্যবসায়িক নিয়ম বারবার পরিবর্তন হচ্ছে এবং সেগুলো কোডে হার্ডকোড করা যন্ত্রণাদায়ক, তখনই Interpreter প্যাটার্ন বিবেচনা করুন।
