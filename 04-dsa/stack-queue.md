# 📚 স্ট্যাক ও কিউ (Stack & Queue) — সম্পূর্ণ গাইড

> "তথ্য সংরক্ষণ ও পরিচালনার দুটি মৌলিক ভিত্তি — স্ট্যাক ও কিউ বুঝলে, প্রোগ্রামিংয়ের অর্ধেক সমস্যা সহজ হয়ে যায়।"

---

## 📑 সূচিপত্র

1. [স্ট্যাক (Stack) — LIFO নীতি](#স্ট্যাক-stack--lifo-নীতি)
2. [স্ট্যাকের সমস্যাসমূহ](#স্ট্যাকের-সমস্যাসমূহ)
3. [কিউ (Queue) — FIFO নীতি](#কিউ-queue--fifo-নীতি)
4. [কিউয়ের বিভিন্ন ধরন](#কিউয়ের-বিভিন্ন-ধরন)
5. [কিউয়ের সমস্যাসমূহ](#কিউয়ের-সমস্যাসমূহ)
6. [ব্যবহারিক প্রকল্প — মেসেজ কিউ সিস্টেম](#ব্যবহারিক-প্রকল্প--মেসেজ-কিউ-সিস্টেম)
7. [স্ট্যাক বনাম কিউ তুলনা](#স্ট্যাক-বনাম-কিউ-তুলনা)
8. [ইন্টারভিউ টিপস](#ইন্টারভিউ-টিপস)

---

## স্ট্যাক (Stack) — LIFO নীতি

### 📖 সংজ্ঞা

স্ট্যাক হলো একটি **রৈখিক তথ্য কাঠামো** (Linear Data Structure) যেটি **LIFO (Last In, First Out)** নীতিতে কাজ করে।
অর্থাৎ, **সর্বশেষ যে উপাদান প্রবেশ করে, সেটিই সর্বপ্রথম বের হয়**।

### 🏠 বাস্তব উদাহরণ

- 🍽️ **থালার স্তূপ** — সবার উপরের থালা আগে তোলা হয়
- 📚 **বইয়ের স্তূপ** — উপরের বই আগে সরানো হয়
- ↩️ **ব্রাউজারের Back বোতাম** — সর্বশেষ পৃষ্ঠায় ফিরে যাওয়া
- 🔄 **Ctrl+Z (Undo)** — সর্বশেষ কাজ পূর্বাবস্থায় ফেরানো

### 📐 ASCII চিত্র

```
    ┌─────────┐
    │    40   │  ← শীর্ষ (Top) — এখান থেকে বের হয়
    ├─────────┤
    │    30   │
    ├─────────┤
    │    20   │
    ├─────────┤
    │    10   │  ← তলা (Bottom)
    └─────────┘

    Push(50):           Pop():
    ┌─────────┐         ┌─────────┐
    │    50   │ ← নতুন  │ ▒▒▒▒▒▒▒ │ ← 40 বের হলো
    ├─────────┤         ├─────────┤
    │    40   │         │    30   │ ← নতুন শীর্ষ
    ├─────────┤         ├─────────┤
    │    30   │         │    20   │
    ├─────────┤         ├─────────┤
    │    20   │         │    10   │
    ├─────────┤         └─────────┘
    │    10   │
    └─────────┘
```

### ⚙️ মূল অপারেশনসমূহ

| অপারেশন | বর্ণনা | সময় জটিলতা |
|----------|--------|-------------|
| `push(item)` | শীর্ষে উপাদান যোগ করা | O(1) |
| `pop()` | শীর্ষ থেকে উপাদান সরানো ও ফেরত দেওয়া | O(1) |
| `peek() / top()` | শীর্ষের উপাদান দেখা (সরানো হয় না) | O(1) |
| `isEmpty()` | স্ট্যাক খালি কিনা পরীক্ষা | O(1) |
| `size()` | স্ট্যাকে কতটি উপাদান আছে | O(1) |

---

### 🔧 অ্যারে-ভিত্তিক স্ট্যাক বাস্তবায়ন

#### PHP বাস্তবায়ন

```php
<?php
class Stack {
    private array $items = [];  // স্ট্যাকের উপাদান রাখার অ্যারে

    // স্ট্যাকে উপাদান যোগ করা
    public function push(mixed $item): void {
        $this->items[] = $item;
    }

    // শীর্ষ থেকে উপাদান বের করা
    public function pop(): mixed {
        if ($this->isEmpty()) {
            throw new UnderflowException("স্ট্যাক খালি! Pop করা সম্ভব নয়।");
        }
        return array_pop($this->items);
    }

    // শীর্ষের উপাদান দেখা (সরানো হয় না)
    public function peek(): mixed {
        if ($this->isEmpty()) {
            throw new UnderflowException("স্ট্যাক খালি! Peek করা সম্ভব নয়।");
        }
        return end($this->items);
    }

    // স্ট্যাক খালি কিনা পরীক্ষা
    public function isEmpty(): bool {
        return empty($this->items);
    }

    // স্ট্যাকের আকার
    public function size(): int {
        return count($this->items);
    }
}

// ব্যবহার
$stack = new Stack();
$stack->push(10);
$stack->push(20);
$stack->push(30);
echo $stack->peek();  // ফলাফল: 30
echo $stack->pop();   // ফলাফল: 30
echo $stack->size();  // ফলাফল: 2
```

#### JavaScript বাস্তবায়ন

```javascript
class Stack {
    constructor() {
        this.items = [];  // স্ট্যাকের উপাদান রাখার অ্যারে
    }

    // স্ট্যাকে উপাদান যোগ করা
    push(item) {
        this.items.push(item);
    }

    // শীর্ষ থেকে উপাদান বের করা
    pop() {
        if (this.isEmpty()) {
            throw new Error("স্ট্যাক খালি! Pop করা সম্ভব নয়।");
        }
        return this.items.pop();
    }

    // শীর্ষের উপাদান দেখা
    peek() {
        if (this.isEmpty()) {
            throw new Error("স্ট্যাক খালি! Peek করা সম্ভব নয়।");
        }
        return this.items[this.items.length - 1];
    }

    // স্ট্যাক খালি কিনা
    isEmpty() {
        return this.items.length === 0;
    }

    // স্ট্যাকের আকার
    size() {
        return this.items.length;
    }
}

// ব্যবহার
const stack = new Stack();
stack.push(10);
stack.push(20);
stack.push(30);
console.log(stack.peek());  // ফলাফল: 30
console.log(stack.pop());   // ফলাফল: 30
console.log(stack.size());  // ফলাফল: 2
```

---

### 🔗 লিংকড লিস্ট-ভিত্তিক স্ট্যাক বাস্তবায়ন

#### PHP বাস্তবায়ন

```php
<?php
// নোড ক্লাস — প্রতিটি উপাদান এক একটি নোড
class Node {
    public mixed $data;
    public ?Node $next;

    public function __construct(mixed $data) {
        $this->data = $data;
        $this->next = null;  // পরবর্তী নোডের রেফারেন্স
    }
}

class LinkedListStack {
    private ?Node $top = null;  // শীর্ষ নোড
    private int $count = 0;      // মোট উপাদান সংখ্যা

    // নতুন উপাদান শীর্ষে যোগ করা
    public function push(mixed $data): void {
        $newNode = new Node($data);
        $newNode->next = $this->top;  // নতুন নোড বর্তমান শীর্ষকে নির্দেশ করে
        $this->top = $newNode;        // নতুন নোড এখন শীর্ষ
        $this->count++;
    }

    // শীর্ষ থেকে উপাদান সরানো
    public function pop(): mixed {
        if ($this->isEmpty()) {
            throw new UnderflowException("স্ট্যাক খালি!");
        }
        $data = $this->top->data;
        $this->top = $this->top->next;  // পরবর্তী নোড শীর্ষ হয়
        $this->count--;
        return $data;
    }

    // শীর্ষের উপাদান দেখা
    public function peek(): mixed {
        if ($this->isEmpty()) {
            throw new UnderflowException("স্ট্যাক খালি!");
        }
        return $this->top->data;
    }

    public function isEmpty(): bool {
        return $this->top === null;
    }

    public function size(): int {
        return $this->count;
    }
}
```

#### JavaScript বাস্তবায়ন

```javascript
// নোড ক্লাস — প্রতিটি উপাদান রাখার জন্য
class Node {
    constructor(data) {
        this.data = data;
        this.next = null;  // পরবর্তী নোডের রেফারেন্স
    }
}

class LinkedListStack {
    constructor() {
        this.top = null;   // শীর্ষ নোড
        this.count = 0;    // মোট উপাদান
    }

    // শীর্ষে উপাদান যোগ
    push(data) {
        const newNode = new Node(data);
        newNode.next = this.top;  // নতুন নোড বর্তমান শীর্ষকে নির্দেশ করে
        this.top = newNode;       // নতুন নোড শীর্ষ হলো
        this.count++;
    }

    // শীর্ষ থেকে সরানো
    pop() {
        if (this.isEmpty()) {
            throw new Error("স্ট্যাক খালি!");
        }
        const data = this.top.data;
        this.top = this.top.next;  // পরবর্তী নোড নতুন শীর্ষ
        this.count--;
        return data;
    }

    // শীর্ষ দেখা
    peek() {
        if (this.isEmpty()) throw new Error("স্ট্যাক খালি!");
        return this.top.data;
    }

    isEmpty() {
        return this.top === null;
    }

    size() {
        return this.count;
    }
}
```

### 📊 জটিলতা তুলনা সারণি

| অপারেশন | অ্যারে-ভিত্তিক | লিংকড লিস্ট-ভিত্তিক |
|----------|----------------|----------------------|
| Push | O(1)* | O(1) |
| Pop | O(1) | O(1) |
| Peek | O(1) | O(1) |
| Space | O(n) | O(n) |

> *অ্যারে পূর্ণ হলে রিসাইজে O(n) লাগতে পারে, কিন্তু amortized O(1)।

---

## স্ট্যাকের সমস্যাসমূহ

### ১. ভারসাম্যপূর্ণ বন্ধনী (Balanced Parentheses)

**সমস্যা:** একটি স্ট্রিংয়ে সব বন্ধনী `()`, `{}`, `[]` সঠিকভাবে জোড়া আছে কিনা যাচাই করো।

```
সঠিক:   "{[()]}"   "([{}])"   "()()"
ভুল:     "{[}]"     "(()"      ")("
```

#### PHP সমাধান

```php
<?php
// ভারসাম্যপূর্ণ বন্ধনী পরীক্ষা করার ফাংশন
function isBalanced(string $s): bool {
    $stack = new SplStack();
    // প্রতিটি বন্ধনীর জোড়া ম্যাপ
    $matching = [')' => '(', '}' => '{', ']' => '['];

    for ($i = 0; $i < strlen($s); $i++) {
        $char = $s[$i];

        // খোলা বন্ধনী হলে স্ট্যাকে রাখো
        if (in_array($char, ['(', '{', '['])) {
            $stack->push($char);
        }
        // বন্ধ বন্ধনী হলে জোড়া মিলাও
        elseif (isset($matching[$char])) {
            // স্ট্যাক খালি বা শীর্ষে সঠিক জোড়া নেই
            if ($stack->isEmpty() || $stack->top() !== $matching[$char]) {
                return false;
            }
            $stack->pop();
        }
    }

    // সব বন্ধনী জোড়া হলে স্ট্যাক খালি থাকবে
    return $stack->isEmpty();
}

echo isBalanced("{[()]}") ? "সঠিক" : "ভুল";  // ফলাফল: সঠিক
echo isBalanced("{[}]") ? "সঠিক" : "ভুল";    // ফলাফল: ভুল
```

#### JavaScript সমাধান

```javascript
// ভারসাম্যপূর্ণ বন্ধনী পরীক্ষা
function isBalanced(s) {
    const stack = [];
    // জোড়া ম্যাপ
    const matching = { ')': '(', '}': '{', ']': '[' };

    for (const char of s) {
        if (['(', '{', '['].includes(char)) {
            stack.push(char);  // খোলা বন্ধনী স্ট্যাকে রাখো
        } else if (matching[char]) {
            // স্ট্যাক খালি বা জোড়া মিলছে না
            if (stack.length === 0 || stack[stack.length - 1] !== matching[char]) {
                return false;
            }
            stack.pop();  // জোড়া মিললে সরাও
        }
    }

    return stack.length === 0;  // সব জোড়া হলে খালি
}

console.log(isBalanced("{[()]}")); // true
console.log(isBalanced("{[}]"));   // false
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n)

---

### ২. ন্যূনতম স্ট্যাক (Min Stack)

**সমস্যা:** এমন একটি স্ট্যাক ডিজাইন করো যেটি push, pop, top এবং **O(1) তে সর্বনিম্ন মান** ফেরত দেয়।

#### PHP সমাধান

```php
<?php
class MinStack {
    private array $stack = [];     // মূল স্ট্যাক
    private array $minStack = [];  // প্রতিটি স্তরে ন্যূনতম মান ট্র্যাক করে

    public function push(int $val): void {
        $this->stack[] = $val;
        // ন্যূনতম স্ট্যাকে বর্তমান ন্যূনতম মান রাখা
        $currentMin = empty($this->minStack)
            ? $val
            : min($val, end($this->minStack));
        $this->minStack[] = $currentMin;
    }

    public function pop(): void {
        array_pop($this->stack);
        array_pop($this->minStack);  // দুটি স্ট্যাক সমান্তরালে চলে
    }

    public function top(): int {
        return end($this->stack);
    }

    // O(1) তে ন্যূনতম মান
    public function getMin(): int {
        return end($this->minStack);
    }
}

$ms = new MinStack();
$ms->push(5);
$ms->push(3);
$ms->push(7);
echo $ms->getMin(); // ফলাফল: 3
$ms->pop();
echo $ms->getMin(); // ফলাফল: 3
$ms->pop();
echo $ms->getMin(); // ফলাফল: 5
```

#### JavaScript সমাধান

```javascript
class MinStack {
    constructor() {
        this.stack = [];     // মূল স্ট্যাক
        this.minStack = [];  // ন্যূনতম মান ট্র্যাকার
    }

    push(val) {
        this.stack.push(val);
        // ন্যূনতম স্ট্যাকে বর্তমান ন্যূনতম রাখা
        const currentMin = this.minStack.length === 0
            ? val
            : Math.min(val, this.minStack[this.minStack.length - 1]);
        this.minStack.push(currentMin);
    }

    pop() {
        this.stack.pop();
        this.minStack.pop();
    }

    top() {
        return this.stack[this.stack.length - 1];
    }

    // O(1) তে ন্যূনতম
    getMin() {
        return this.minStack[this.minStack.length - 1];
    }
}
```

**⏱️ জটিলতা:** সব অপারেশন O(1) সময়, O(n) স্থান

---

### ৩. পোস্টফিক্স মূল্যায়ন (Postfix Evaluation)

**সমস্যা:** পোস্টফিক্স (Reverse Polish Notation) রাশি মূল্যায়ন করো।

```
ইনপুট:  ["2", "3", "+", "4", "*"]
ধাপ:    2 3 + → 5, তারপর 5 4 * → 20
ফলাফল:  20
```

#### PHP সমাধান

```php
<?php
// পোস্টফিক্স রাশি মূল্যায়ন
function evalPostfix(array $tokens): int {
    $stack = new SplStack();

    foreach ($tokens as $token) {
        // অপারেটর হলে দুটি সংখ্যা বের করে হিসাব করো
        if (in_array($token, ['+', '-', '*', '/'])) {
            $b = $stack->pop();  // দ্বিতীয় অপারেন্ড
            $a = $stack->pop();  // প্রথম অপারেন্ড
            $result = match($token) {
                '+' => $a + $b,
                '-' => $a - $b,
                '*' => $a * $b,
                '/' => intdiv($a, $b),  // পূর্ণসংখ্যা ভাগ
            };
            $stack->push($result);  // ফলাফল স্ট্যাকে রাখো
        } else {
            $stack->push((int)$token);  // সংখ্যা হলে সরাসরি রাখো
        }
    }

    return $stack->pop();  // চূড়ান্ত ফলাফল
}

echo evalPostfix(["2", "3", "+", "4", "*"]); // ফলাফল: 20
```

#### JavaScript সমাধান

```javascript
// পোস্টফিক্স রাশি মূল্যায়ন
function evalPostfix(tokens) {
    const stack = [];
    const ops = { '+': (a, b) => a + b, '-': (a, b) => a - b,
                  '*': (a, b) => a * b, '/': (a, b) => Math.trunc(a / b) };

    for (const token of tokens) {
        if (ops[token]) {
            const b = stack.pop();  // দ্বিতীয় অপারেন্ড
            const a = stack.pop();  // প্রথম অপারেন্ড
            stack.push(ops[token](a, b));  // ফলাফল স্ট্যাকে
        } else {
            stack.push(parseInt(token));  // সংখ্যা স্ট্যাকে
        }
    }

    return stack.pop();  // চূড়ান্ত ফলাফল
}

console.log(evalPostfix(["2", "3", "+", "4", "*"])); // 20
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n)

---

### ৪. পরবর্তী বড় উপাদান (Next Greater Element)

**সমস্যা:** প্রতিটি উপাদানের জন্য তার ডানদিকের প্রথম বড় উপাদান খোঁজো।

```
ইনপুট:  [4, 5, 2, 10, 8]
ফলাফল:  [5, 10, 10, -1, -1]
```

#### JavaScript সমাধান

```javascript
// পরবর্তী বড় উপাদান খোঁজা — ডান থেকে বামে স্ক্যান
function nextGreaterElement(arr) {
    const result = new Array(arr.length).fill(-1);
    const stack = [];  // ইনডেক্স রাখার স্ট্যাক

    // ডান থেকে বামে চলো
    for (let i = arr.length - 1; i >= 0; i--) {
        // স্ট্যাকের শীর্ষে ছোট বা সমান মান থাকলে সরাও
        while (stack.length > 0 && stack[stack.length - 1] <= arr[i]) {
            stack.pop();
        }
        // স্ট্যাক খালি না হলে শীর্ষই পরবর্তী বড় উপাদান
        if (stack.length > 0) {
            result[i] = stack[stack.length - 1];
        }
        stack.push(arr[i]);  // বর্তমান উপাদান স্ট্যাকে
    }

    return result;
}

console.log(nextGreaterElement([4, 5, 2, 10, 8]));
// ফলাফল: [5, 10, 10, -1, -1]
```

#### PHP সমাধান

```php
<?php
// পরবর্তী বড় উপাদান — ডান থেকে বাম
function nextGreaterElement(array $arr): array {
    $n = count($arr);
    $result = array_fill(0, $n, -1);
    $stack = [];

    for ($i = $n - 1; $i >= 0; $i--) {
        // ছোট মান সরাও
        while (!empty($stack) && end($stack) <= $arr[$i]) {
            array_pop($stack);
        }
        if (!empty($stack)) {
            $result[$i] = end($stack);
        }
        $stack[] = $arr[$i];
    }

    return $result;
}

print_r(nextGreaterElement([4, 5, 2, 10, 8]));
// ফলাফল: [5, 10, 10, -1, -1]
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n) — প্রতিটি উপাদান সর্বাধিক একবার push ও একবার pop হয়

---

### ৫. দৈনিক তাপমাত্রা (Daily Temperatures)

**সমস্যা:** প্রতিদিনের জন্য কতদিন পর উষ্ণতর দিন আসবে তা বের করো।

```
ইনপুট:  [73, 74, 75, 71, 69, 72, 76, 73]
ফলাফল:  [1,  1,  4,  2,  1,  1,  0,  0]
```

#### JavaScript সমাধান

```javascript
// দৈনিক তাপমাত্রা — মনোটনিক স্ট্যাক ব্যবহার
function dailyTemperatures(temperatures) {
    const n = temperatures.length;
    const result = new Array(n).fill(0);
    const stack = [];  // ইনডেক্সের স্ট্যাক (হ্রাসমান তাপমাত্রা)

    for (let i = 0; i < n; i++) {
        // বর্তমান তাপমাত্রা স্ট্যাকের শীর্ষের চেয়ে বেশি হলে
        while (stack.length > 0 && temperatures[i] > temperatures[stack[stack.length - 1]]) {
            const prevIndex = stack.pop();
            result[prevIndex] = i - prevIndex;  // দিনের ব্যবধান
        }
        stack.push(i);  // বর্তমান ইনডেক্স স্ট্যাকে
    }

    return result;
}

console.log(dailyTemperatures([73, 74, 75, 71, 69, 72, 76, 73]));
// ফলাফল: [1, 1, 4, 2, 1, 1, 0, 0]
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n)

---

### ৬. ইনফিক্স থেকে পোস্টফিক্স রূপান্তর (Shunting Yard অ্যালগরিদম)

**সমস্যা:** `3 + 4 * 2` → `3 4 2 * +`

```
অপারেটর অগ্রাধিকার:
  * , / → উচ্চ (2)
  + , - → নিম্ন (1)
```

#### PHP সমাধান

```php
<?php
// ইনফিক্স থেকে পোস্টফিক্স — Shunting Yard অ্যালগরিদম
function infixToPostfix(string $expression): string {
    // অপারেটরের অগ্রাধিকার
    $precedence = ['+' => 1, '-' => 1, '*' => 2, '/' => 2];
    $output = [];  // ফলাফল কিউ
    $stack = [];   // অপারেটর স্ট্যাক
    $tokens = preg_split('/\s+/', trim($expression));

    foreach ($tokens as $token) {
        if (is_numeric($token)) {
            $output[] = $token;  // সংখ্যা সরাসরি আউটপুটে
        } elseif ($token === '(') {
            $stack[] = $token;   // খোলা বন্ধনী স্ট্যাকে
        } elseif ($token === ')') {
            // বন্ধ বন্ধনী — খোলা বন্ধনী না পাওয়া পর্যন্ত pop
            while (end($stack) !== '(') {
                $output[] = array_pop($stack);
            }
            array_pop($stack);  // '(' সরাও
        } else {
            // অপারেটর — অগ্রাধিকার মেনে pop করো
            while (!empty($stack) && end($stack) !== '('
                   && ($precedence[end($stack)] ?? 0) >= ($precedence[$token] ?? 0)) {
                $output[] = array_pop($stack);
            }
            $stack[] = $token;
        }
    }

    // বাকি অপারেটর আউটপুটে
    while (!empty($stack)) {
        $output[] = array_pop($stack);
    }

    return implode(' ', $output);
}

echo infixToPostfix("3 + 4 * 2");         // ফলাফল: 3 4 2 * +
echo infixToPostfix("( 3 + 4 ) * 2");     // ফলাফল: 3 4 + 2 *
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n)

---

### ৭. কল স্ট্যাক ও রিকার্সন (Call Stack & Recursion)

প্রতিটি ফাংশন কলে একটি **স্ট্যাক ফ্রেম** তৈরি হয়। রিকার্সন স্বয়ংক্রিয়ভাবে কল স্ট্যাক ব্যবহার করে।

```
factorial(4) এর কল স্ট্যাক:

    ┌────────────────┐
    │ factorial(1)=1 │  ← বেস কেস — ফেরত শুরু
    ├────────────────┤
    │ factorial(2)   │  → 2 * 1 = 2
    ├────────────────┤
    │ factorial(3)   │  → 3 * 2 = 6
    ├────────────────┤
    │ factorial(4)   │  → 4 * 6 = 24
    └────────────────┘
```

#### JavaScript সমাধান — রিকার্সন বনাম স্ট্যাক

```javascript
// রিকার্সিভ ফ্যাক্টোরিয়াল — কল স্ট্যাক ব্যবহার করে
function factorialRecursive(n) {
    if (n <= 1) return 1;       // বেস কেস
    return n * factorialRecursive(n - 1);  // রিকার্সিভ কল
}

// ইটারেটিভ — নিজস্ব স্ট্যাক ব্যবহার করে (স্ট্যাক ওভারফ্লো এড়ানো)
function factorialIterative(n) {
    const stack = [];
    // স্ট্যাকে সব মান রাখো
    for (let i = n; i >= 1; i--) {
        stack.push(i);
    }
    // একে একে গুণ করো
    let result = 1;
    while (stack.length > 0) {
        result *= stack.pop();
    }
    return result;
}

console.log(factorialRecursive(5));  // 120
console.log(factorialIterative(5));  // 120
```

> 💡 **গুরুত্বপূর্ণ:** গভীর রিকার্সনে Stack Overflow হতে পারে। সেক্ষেত্রে নিজস্ব স্ট্যাক ব্যবহার করো।

---

## কিউ (Queue) — FIFO নীতি

### 📖 সংজ্ঞা

কিউ হলো একটি **রৈখিক তথ্য কাঠামো** যেটি **FIFO (First In, First Out)** নীতিতে কাজ করে।
অর্থাৎ, **সর্বপ্রথম যে উপাদান প্রবেশ করে, সেটিই সর্বপ্রথম বের হয়**।

### 🏠 বাস্তব উদাহরণ

- 🏦 **ব্যাংকের লাইন** — আগে আসলে আগে সেবা পাওয়া
- 🖨️ **প্রিন্টার কিউ** — প্রথমে পাঠানো ফাইল আগে প্রিন্ট হয়
- 📞 **কল সেন্টার** — আগে কল করলে আগে উত্তর পাওয়া
- 🚗 **টোল বুথ** — আগে আসা গাড়ি আগে যায়

### 📐 ASCII চিত্র

```
    বের হয় (Dequeue)                        প্রবেশ করে (Enqueue)
    ←──────────                              ──────────→
    ┌────┬────┬────┬────┬────┐
    │ 10 │ 20 │ 30 │ 40 │ 50 │
    └────┴────┴────┴────┴────┘
    ↑ সামনে (Front)          ↑ পেছনে (Rear)

    Enqueue(60):
    ┌────┬────┬────┬────┬────┬────┐
    │ 10 │ 20 │ 30 │ 40 │ 50 │ 60 │  ← 60 পেছনে যোগ
    └────┴────┴────┴────┴────┴────┘

    Dequeue():
    ┌────┬────┬────┬────┬────┐
    │ 20 │ 30 │ 40 │ 50 │ 60 │  ← 10 সামনে থেকে বের হলো
    └────┴────┴────┴────┴────┘
```

### ⚙️ মূল অপারেশনসমূহ

| অপারেশন | বর্ণনা | সময় জটিলতা |
|----------|--------|-------------|
| `enqueue(item)` | পেছনে উপাদান যোগ | O(1) |
| `dequeue()` | সামনে থেকে উপাদান সরানো ও ফেরত দেওয়া | O(1) |
| `front() / peek()` | সামনের উপাদান দেখা | O(1) |
| `isEmpty()` | কিউ খালি কিনা | O(1) |
| `size()` | কিউয়ের আকার | O(1) |

---

### 🔧 লিংকড লিস্ট-ভিত্তিক কিউ বাস্তবায়ন

#### PHP বাস্তবায়ন

```php
<?php
class QueueNode {
    public mixed $data;
    public ?QueueNode $next = null;

    public function __construct(mixed $data) {
        $this->data = $data;
    }
}

class LinkedListQueue {
    private ?QueueNode $front = null;  // সামনের নোড
    private ?QueueNode $rear = null;   // পেছনের নোড
    private int $count = 0;

    // পেছনে উপাদান যোগ
    public function enqueue(mixed $data): void {
        $newNode = new QueueNode($data);
        if ($this->isEmpty()) {
            $this->front = $this->rear = $newNode;
        } else {
            $this->rear->next = $newNode;  // পেছনে যুক্ত
            $this->rear = $newNode;
        }
        $this->count++;
    }

    // সামনে থেকে উপাদান সরানো
    public function dequeue(): mixed {
        if ($this->isEmpty()) {
            throw new UnderflowException("কিউ খালি!");
        }
        $data = $this->front->data;
        $this->front = $this->front->next;
        if ($this->front === null) {
            $this->rear = null;  // কিউ খালি হলে rear ও null
        }
        $this->count--;
        return $data;
    }

    // সামনের উপাদান দেখা
    public function peek(): mixed {
        if ($this->isEmpty()) throw new UnderflowException("কিউ খালি!");
        return $this->front->data;
    }

    public function isEmpty(): bool {
        return $this->front === null;
    }

    public function size(): int {
        return $this->count;
    }
}

// ব্যবহার
$queue = new LinkedListQueue();
$queue->enqueue(10);
$queue->enqueue(20);
$queue->enqueue(30);
echo $queue->peek();     // ফলাফল: 10
echo $queue->dequeue();  // ফলাফল: 10
echo $queue->peek();     // ফলাফল: 20
```

#### JavaScript বাস্তবায়ন

```javascript
class QueueNode {
    constructor(data) {
        this.data = data;
        this.next = null;
    }
}

class LinkedListQueue {
    constructor() {
        this.front = null;  // সামনে
        this.rear = null;   // পেছনে
        this.count = 0;
    }

    // পেছনে যোগ
    enqueue(data) {
        const newNode = new QueueNode(data);
        if (this.isEmpty()) {
            this.front = this.rear = newNode;
        } else {
            this.rear.next = newNode;
            this.rear = newNode;
        }
        this.count++;
    }

    // সামনে থেকে সরানো
    dequeue() {
        if (this.isEmpty()) throw new Error("কিউ খালি!");
        const data = this.front.data;
        this.front = this.front.next;
        if (!this.front) this.rear = null;
        this.count--;
        return data;
    }

    peek() {
        if (this.isEmpty()) throw new Error("কিউ খালি!");
        return this.front.data;
    }

    isEmpty() { return this.front === null; }
    size() { return this.count; }
}
```

---

### 🔄 বৃত্তাকার কিউ (Circular Queue) — অ্যারে-ভিত্তিক

বৃত্তাকার কিউতে অ্যারের শেষে পৌঁছালে আবার শুরুতে ফিরে যায়। **মডুলার গাণিতিক** ব্যবহৃত হয়।

```
    সূচি:     0    1    2    3    4
            ┌────┬────┬────┬────┬────┐
            │ __ │ 20 │ 30 │ 40 │ __ │
            └────┴────┴────┴────┴────┘
              ↑              ↑
              rear=0        front=1

    মডুলার সূত্র:
    পরবর্তী সূচি = (বর্তমান সূচি + 1) % ধারণক্ষমতা
```

#### PHP বাস্তবায়ন

```php
<?php
class CircularQueue {
    private array $items;
    private int $front = 0;
    private int $rear = -1;
    private int $size = 0;
    private int $capacity;

    public function __construct(int $capacity) {
        $this->capacity = $capacity;
        $this->items = array_fill(0, $capacity, null);
    }

    // পেছনে যোগ — মডুলার গণিত ব্যবহার
    public function enqueue(mixed $data): bool {
        if ($this->isFull()) return false;  // পূর্ণ হলে ব্যর্থ
        $this->rear = ($this->rear + 1) % $this->capacity;  // বৃত্তাকার সূচি
        $this->items[$this->rear] = $data;
        $this->size++;
        return true;
    }

    // সামনে থেকে সরানো — মডুলার গণিত
    public function dequeue(): mixed {
        if ($this->isEmpty()) return null;
        $data = $this->items[$this->front];
        $this->items[$this->front] = null;
        $this->front = ($this->front + 1) % $this->capacity;  // বৃত্তাকার সূচি
        $this->size--;
        return $data;
    }

    public function peek(): mixed {
        return $this->isEmpty() ? null : $this->items[$this->front];
    }

    public function isEmpty(): bool { return $this->size === 0; }
    public function isFull(): bool { return $this->size === $this->capacity; }
}
```

#### JavaScript বাস্তবায়ন

```javascript
class CircularQueue {
    constructor(capacity) {
        this.items = new Array(capacity).fill(null);
        this.capacity = capacity;
        this.front = 0;
        this.rear = -1;
        this.size = 0;
    }

    // পেছনে যোগ — মডুলার গণিত
    enqueue(data) {
        if (this.isFull()) return false;
        this.rear = (this.rear + 1) % this.capacity;
        this.items[this.rear] = data;
        this.size++;
        return true;
    }

    // সামনে থেকে সরানো
    dequeue() {
        if (this.isEmpty()) return null;
        const data = this.items[this.front];
        this.items[this.front] = null;
        this.front = (this.front + 1) % this.capacity;
        this.size--;
        return data;
    }

    peek() { return this.isEmpty() ? null : this.items[this.front]; }
    isEmpty() { return this.size === 0; }
    isFull() { return this.size === this.capacity; }
}
```

---

## কিউয়ের বিভিন্ন ধরন

### ১. ডেক (Deque — Double Ended Queue)

উভয় প্রান্ত থেকে যোগ ও সরানো যায়।

```
    সামনে যোগ →  ┌────┬────┬────┬────┐  ← পেছনে যোগ
    সামনে সরানো ← │ 10 │ 20 │ 30 │ 40 │ → পেছনে সরানো
                  └────┴────┴────┴────┘
```

#### JavaScript বাস্তবায়ন

```javascript
class Deque {
    constructor() {
        this.items = {};  // অবজেক্ট-ভিত্তিক — O(1) উভয় প্রান্তে
        this.frontIndex = 0;
        this.rearIndex = -1;
    }

    // সামনে যোগ
    addFront(item) {
        this.frontIndex--;
        this.items[this.frontIndex] = item;
    }

    // পেছনে যোগ
    addRear(item) {
        this.rearIndex++;
        this.items[this.rearIndex] = item;
    }

    // সামনে থেকে সরানো
    removeFront() {
        if (this.isEmpty()) throw new Error("ডেক খালি!");
        const item = this.items[this.frontIndex];
        delete this.items[this.frontIndex];
        this.frontIndex++;
        return item;
    }

    // পেছন থেকে সরানো
    removeRear() {
        if (this.isEmpty()) throw new Error("ডেক খালি!");
        const item = this.items[this.rearIndex];
        delete this.items[this.rearIndex];
        this.rearIndex--;
        return item;
    }

    peekFront() { return this.items[this.frontIndex]; }
    peekRear() { return this.items[this.rearIndex]; }
    isEmpty() { return this.rearIndex < this.frontIndex; }
    size() { return this.rearIndex - this.frontIndex + 1; }
}
```

---

### ২. অগ্রাধিকার কিউ (Priority Queue)

প্রতিটি উপাদানের একটি **অগ্রাধিকার** থাকে। উচ্চ অগ্রাধিকারের উপাদান আগে বের হয়।

```
    সাধারণ কিউ:         অগ্রাধিকার কিউ:
    A → B → C → D       B(3) → D(2) → A(1) → C(1)
    (আসার ক্রমে)         (অগ্রাধিকার অনুসারে)
```

#### PHP বাস্তবায়ন

```php
<?php
class PriorityQueue {
    private array $heap = [];  // মিন-হিপ — ছোট সংখ্যা = উচ্চ অগ্রাধিকার

    // উপাদান যোগ — হিপে সঠিক স্থানে বসানো
    public function enqueue(mixed $data, int $priority): void {
        $this->heap[] = ['data' => $data, 'priority' => $priority];
        $this->bubbleUp(count($this->heap) - 1);  // উপরে তোলা
    }

    // সর্বোচ্চ অগ্রাধিকারের উপাদান বের করা
    public function dequeue(): mixed {
        if ($this->isEmpty()) throw new UnderflowException("কিউ খালি!");
        $min = $this->heap[0];
        $last = array_pop($this->heap);
        if (!empty($this->heap)) {
            $this->heap[0] = $last;
            $this->bubbleDown(0);  // নিচে নামানো
        }
        return $min['data'];
    }

    // উপরে তোলা — পিতামাতার চেয়ে ছোট হলে অদলবদল
    private function bubbleUp(int $i): void {
        while ($i > 0) {
            $parent = intdiv($i - 1, 2);
            if ($this->heap[$i]['priority'] < $this->heap[$parent]['priority']) {
                [$this->heap[$i], $this->heap[$parent]] = [$this->heap[$parent], $this->heap[$i]];
                $i = $parent;
            } else break;
        }
    }

    // নিচে নামানো — সন্তানের চেয়ে বড় হলে অদলবদল
    private function bubbleDown(int $i): void {
        $n = count($this->heap);
        while (true) {
            $smallest = $i;
            $left = 2 * $i + 1;
            $right = 2 * $i + 2;
            if ($left < $n && $this->heap[$left]['priority'] < $this->heap[$smallest]['priority']) {
                $smallest = $left;
            }
            if ($right < $n && $this->heap[$right]['priority'] < $this->heap[$smallest]['priority']) {
                $smallest = $right;
            }
            if ($smallest !== $i) {
                [$this->heap[$i], $this->heap[$smallest]] = [$this->heap[$smallest], $this->heap[$i]];
                $i = $smallest;
            } else break;
        }
    }

    public function isEmpty(): bool { return empty($this->heap); }
    public function peek(): mixed { return $this->heap[0]['data'] ?? null; }
}

// ব্যবহার — হাসপাতালের জরুরি বিভাগ
$pq = new PriorityQueue();
$pq->enqueue("সাধারণ জ্বর", 3);       // নিম্ন অগ্রাধিকার
$pq->enqueue("হার্ট অ্যাটাক", 1);     // সর্বোচ্চ অগ্রাধিকার
$pq->enqueue("হাড় ভাঙা", 2);          // মধ্যম
echo $pq->dequeue(); // "হার্ট অ্যাটাক" — আগে সেবা পায়
echo $pq->dequeue(); // "হাড় ভাঙা"
echo $pq->dequeue(); // "সাধারণ জ্বর"
```

---

### ৩. মনোটনিক কিউ (Monotonic Queue)

কিউয়ের মধ্যে উপাদানগুলো সর্বদা **বৃদ্ধি বা হ্রাস ক্রমে** থাকে। Sliding Window Maximum সমস্যায় অত্যন্ত কার্যকর।

---

## কিউয়ের সমস্যাসমূহ

### ১. দুটি স্ট্যাক দিয়ে কিউ (Queue using Two Stacks)

**ধারণা:** দুটি স্ট্যাক ব্যবহার করে FIFO আচরণ তৈরি করা।

```
    pushStack:  [1, 2, 3]  ← push এখানে
    popStack:   []

    dequeue চাইলে:
    pushStack থেকে popStack এ ঢালো (উল্টে যায়):
    pushStack:  []
    popStack:   [3, 2, 1]  ← pop করলে 1 আসে (FIFO!)
```

#### PHP সমাধান

```php
<?php
class QueueUsingStacks {
    private array $pushStack = [];  // enqueue এর জন্য
    private array $popStack = [];   // dequeue এর জন্য

    // enqueue — সর্বদা pushStack এ রাখো
    public function enqueue(mixed $item): void {
        $this->pushStack[] = $item;
    }

    // dequeue — popStack খালি হলে pushStack থেকে ঢালো
    public function dequeue(): mixed {
        if (empty($this->popStack)) {
            // pushStack এর সব উপাদান popStack এ স্থানান্তর (ক্রম উল্টে যায়)
            while (!empty($this->pushStack)) {
                $this->popStack[] = array_pop($this->pushStack);
            }
        }
        if (empty($this->popStack)) {
            throw new UnderflowException("কিউ খালি!");
        }
        return array_pop($this->popStack);
    }

    public function peek(): mixed {
        if (empty($this->popStack)) {
            while (!empty($this->pushStack)) {
                $this->popStack[] = array_pop($this->pushStack);
            }
        }
        return end($this->popStack);
    }

    public function isEmpty(): bool {
        return empty($this->pushStack) && empty($this->popStack);
    }
}

$q = new QueueUsingStacks();
$q->enqueue(1);
$q->enqueue(2);
$q->enqueue(3);
echo $q->dequeue(); // 1 (FIFO!)
echo $q->dequeue(); // 2
```

#### JavaScript সমাধান

```javascript
class QueueUsingStacks {
    constructor() {
        this.pushStack = [];  // enqueue এর জন্য
        this.popStack = [];   // dequeue এর জন্য
    }

    enqueue(item) {
        this.pushStack.push(item);
    }

    // popStack খালি হলে pushStack থেকে স্থানান্তর
    dequeue() {
        if (this.popStack.length === 0) {
            while (this.pushStack.length > 0) {
                this.popStack.push(this.pushStack.pop());
            }
        }
        if (this.popStack.length === 0) throw new Error("কিউ খালি!");
        return this.popStack.pop();
    }

    peek() {
        if (this.popStack.length === 0) {
            while (this.pushStack.length > 0) {
                this.popStack.push(this.pushStack.pop());
            }
        }
        return this.popStack[this.popStack.length - 1];
    }

    isEmpty() {
        return this.pushStack.length === 0 && this.popStack.length === 0;
    }
}
```

**⏱️ জটিলতা:** enqueue O(1), dequeue **amortized O(1)** — প্রতিটি উপাদান সর্বাধিক ২ বার সরে

---

### ২. দুটি কিউ দিয়ে স্ট্যাক (Stack using Two Queues)

#### JavaScript সমাধান

```javascript
class StackUsingQueues {
    constructor() {
        this.q1 = [];  // মূল কিউ
        this.q2 = [];  // সাহায্যকারী কিউ
    }

    // push — নতুন উপাদান q2 তে রাখো, তারপর q1 এর সব q2 তে ঢালো
    push(item) {
        this.q2.push(item);
        // q1 এর সব q2 তে স্থানান্তর
        while (this.q1.length > 0) {
            this.q2.push(this.q1.shift());
        }
        // q1 ও q2 অদলবদল
        [this.q1, this.q2] = [this.q2, this.q1];
    }

    // pop — q1 এর সামনে থেকে সরানো (এটাই সর্বশেষ push করা)
    pop() {
        if (this.q1.length === 0) throw new Error("স্ট্যাক খালি!");
        return this.q1.shift();
    }

    top() {
        if (this.q1.length === 0) throw new Error("স্ট্যাক খালি!");
        return this.q1[0];
    }

    isEmpty() { return this.q1.length === 0; }
}
```

**⏱️ জটিলতা:** push O(n), pop O(1)

---

### ৩. স্লাইডিং উইন্ডো ম্যাক্সিমাম (Sliding Window Maximum)

**সমস্যা:** k আকারের প্রতিটি উইন্ডোতে সর্বাধিক মান খোঁজো।

```
ইনপুট:  nums = [1,3,-1,-3,5,3,6,7], k = 3
উইন্ডো:   [1,3,-1] → 3
           [3,-1,-3] → 3
           [-1,-3,5] → 5
           [-3,5,3] → 5
           [5,3,6] → 6
           [3,6,7] → 7
ফলাফল:  [3,3,5,5,6,7]
```

#### JavaScript সমাধান — মনোটনিক ডেক ব্যবহার

```javascript
// স্লাইডিং উইন্ডো ম্যাক্সিমাম — হ্রাসমান মনোটনিক ডেক
function maxSlidingWindow(nums, k) {
    const result = [];
    const deque = [];  // ইনডেক্সের ডেক (হ্রাসমান মান ক্রমে)

    for (let i = 0; i < nums.length; i++) {
        // উইন্ডোর বাইরের ইনডেক্স সামনে থেকে সরাও
        if (deque.length > 0 && deque[0] <= i - k) {
            deque.shift();
        }

        // ডেকের পেছনে ছোট মান সরাও (মনোটনিক রক্ষা)
        while (deque.length > 0 && nums[deque[deque.length - 1]] <= nums[i]) {
            deque.pop();
        }

        deque.push(i);  // বর্তমান ইনডেক্স যোগ

        // প্রথম সম্পূর্ণ উইন্ডো থেকে ফলাফল সংগ্রহ
        if (i >= k - 1) {
            result.push(nums[deque[0]]);  // সামনের ইনডেক্সই সর্বাধিক
        }
    }

    return result;
}

console.log(maxSlidingWindow([1,3,-1,-3,5,3,6,7], 3));
// ফলাফল: [3,3,5,5,6,7]
```

#### PHP সমাধান

```php
<?php
function maxSlidingWindow(array $nums, int $k): array {
    $result = [];
    $deque = [];  // ইনডেক্সের ডেক

    for ($i = 0; $i < count($nums); $i++) {
        // উইন্ডোর বাইরে চলে গেছে
        if (!empty($deque) && $deque[0] <= $i - $k) {
            array_shift($deque);
        }
        // পেছনে ছোট মান সরাও
        while (!empty($deque) && $nums[end($deque)] <= $nums[$i]) {
            array_pop($deque);
        }
        $deque[] = $i;
        if ($i >= $k - 1) {
            $result[] = $nums[$deque[0]];
        }
    }

    return $result;
}

print_r(maxSlidingWindow([1,3,-1,-3,5,3,6,7], 3));
// ফলাফল: [3,3,5,5,6,7]
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(k)

---

### ৪. BFS লেভেল অর্ডার ট্রাভার্সাল

**সমস্যা:** বাইনারি ট্রি-তে প্রতিটি স্তরের নোডগুলো আলাদাভাবে ফেরত দাও।

```
        3
       / \
      9   20
         / \
        15   7

    ফলাফল: [[3], [9, 20], [15, 7]]
```

#### JavaScript সমাধান

```javascript
// BFS লেভেল অর্ডার — কিউ ব্যবহার
function levelOrder(root) {
    if (!root) return [];
    const result = [];
    const queue = [root];  // কিউতে রুট দিয়ে শুরু

    while (queue.length > 0) {
        const levelSize = queue.length;  // বর্তমান স্তরের নোড সংখ্যা
        const currentLevel = [];

        for (let i = 0; i < levelSize; i++) {
            const node = queue.shift();  // সামনে থেকে নোড বের করো
            currentLevel.push(node.val);

            // সন্তান নোড কিউতে যোগ করো (পরবর্তী স্তরের জন্য)
            if (node.left) queue.push(node.left);
            if (node.right) queue.push(node.right);
        }

        result.push(currentLevel);  // স্তরটি ফলাফলে যোগ
    }

    return result;
}
```

**⏱️ জটিলতা:** সময় O(n), স্থান O(n)

---

### ৫. টাস্ক শিডিউলার (LeetCode 621)

**সমস্যা:** CPU টাস্কগুলো এমনভাবে সাজাও যেন একই টাস্কের মধ্যে কমপক্ষে `n` সময় ব্যবধান থাকে।

```
ইনপুট:  tasks = ["A","A","A","B","B","B"], n = 2
সাজানো: A → B → idle → A → B → idle → A → B
ফলাফল:  8 (মোট সময়)
```

#### JavaScript সমাধান

```javascript
// টাস্ক শিডিউলার — ম্যাক্স হিপ + কুলডাউন কিউ
function leastInterval(tasks, n) {
    // প্রতিটি টাস্কের ফ্রিকোয়েন্সি গণনা
    const freq = {};
    for (const task of tasks) {
        freq[task] = (freq[task] || 0) + 1;
    }

    // সর্বাধিক ফ্রিকোয়েন্সি অনুসারে সাজানো (ম্যাক্স হিপ অনুকরণ)
    const maxHeap = Object.values(freq).sort((a, b) => b - a);
    const cooldown = [];  // [ফ্রিকোয়েন্সি, প্রস্তুত_সময়] — কুলডাউনে থাকা টাস্ক
    let time = 0;

    while (maxHeap.length > 0 || cooldown.length > 0) {
        time++;

        if (maxHeap.length > 0) {
            // সর্বাধিক ফ্রিকোয়েন্সির টাস্ক নাও
            const count = maxHeap.shift() - 1;
            if (count > 0) {
                cooldown.push([count, time + n]);  // কুলডাউনে পাঠাও
            }
        }

        // কুলডাউন শেষ হলে হিপে ফেরত দাও
        if (cooldown.length > 0 && cooldown[0][1] === time) {
            const [cnt] = cooldown.shift();
            maxHeap.push(cnt);
            maxHeap.sort((a, b) => b - a);  // পুনরায় সাজানো
        }
    }

    return time;
}

console.log(leastInterval(["A","A","A","B","B","B"], 2)); // ফলাফল: 8
```

**⏱️ জটিলতা:** সময় O(n × m) যেখানে m = ভিন্ন টাস্ক সংখ্যা, স্থান O(m)

---

## ব্যবহারিক প্রকল্প — মেসেজ কিউ সিস্টেম

### 📋 প্রকল্প বর্ণনা

একটি **মেসেজ কিউ সিস্টেম** তৈরি করো যেখানে:
- **প্রডিউসার** মেসেজ পাঠায়
- **কনজ্যুমার** মেসেজ গ্রহণ করে
- **অগ্রাধিকার কিউ** — গুরুত্বপূর্ণ মেসেজ আগে প্রক্রিয়া হয়
- **ডেড লেটার কিউ** — ব্যর্থ মেসেজ সংরক্ষণ
- **পুনঃচেষ্টা** — ব্যর্থ মেসেজ পুনরায় পাঠানো

### PHP বাস্তবায়ন

```php
<?php
// মেসেজ — প্রতিটি মেসেজের তথ্য
class Message {
    public string $id;
    public mixed $payload;     // মেসেজের মূল তথ্য
    public int $priority;       // অগ্রাধিকার (কম = বেশি গুরুত্বপূর্ণ)
    public int $retryCount = 0; // পুনঃচেষ্টার সংখ্যা
    public int $maxRetries;     // সর্বাধিক পুনঃচেষ্টা
    public float $createdAt;

    public function __construct(mixed $payload, int $priority = 5, int $maxRetries = 3) {
        $this->id = uniqid('msg_', true);
        $this->payload = $payload;
        $this->priority = $priority;
        $this->maxRetries = $maxRetries;
        $this->createdAt = microtime(true);
    }
}

// ডেড লেটার কিউ — সম্পূর্ণ ব্যর্থ মেসেজ রাখে
class DeadLetterQueue {
    private array $messages = [];

    public function add(Message $msg, string $reason): void {
        $this->messages[] = [
            'message' => $msg,
            'reason' => $reason,      // ব্যর্থতার কারণ
            'failedAt' => date('Y-m-d H:i:s'),
        ];
    }

    public function getAll(): array { return $this->messages; }
    public function count(): int { return count($this->messages); }
}

// মূল মেসেজ কিউ — অগ্রাধিকার সহ
class MessageQueue {
    private array $heap = [];             // অগ্রাধিকার হিপ
    private DeadLetterQueue $dlq;         // ডেড লেটার কিউ
    private array $consumers = [];        // কনজ্যুমার তালিকা

    public function __construct() {
        $this->dlq = new DeadLetterQueue();
    }

    // প্রডিউসার — মেসেজ কিউতে পাঠানো
    public function publish(Message $msg): void {
        $this->heap[] = $msg;
        $this->bubbleUp(count($this->heap) - 1);
        echo "📤 প্রকাশিত: [{$msg->id}] অগ্রাধিকার: {$msg->priority}\n";
    }

    // কনজ্যুমার নিবন্ধন
    public function subscribe(callable $handler): void {
        $this->consumers[] = $handler;
    }

    // মেসেজ প্রক্রিয়া — সর্বোচ্চ অগ্রাধিকারের মেসেজ আগে
    public function consume(): void {
        while (!empty($this->heap)) {
            $msg = $this->dequeueMin();

            foreach ($this->consumers as $handler) {
                try {
                    $handler($msg);
                    echo "✅ প্রক্রিয়াকৃত: [{$msg->id}]\n";
                } catch (\Exception $e) {
                    $this->handleFailure($msg, $e->getMessage());
                }
            }
        }
    }

    // ব্যর্থতা পরিচালনা — পুনঃচেষ্টা বা ডেড লেটার কিউ
    private function handleFailure(Message $msg, string $error): void {
        $msg->retryCount++;
        if ($msg->retryCount <= $msg->maxRetries) {
            echo "🔄 পুনঃচেষ্টা ({$msg->retryCount}/{$msg->maxRetries}): [{$msg->id}]\n";
            $this->publish($msg);  // আবার কিউতে রাখো
        } else {
            echo "💀 ডেড লেটার কিউতে: [{$msg->id}] কারণ: {$error}\n";
            $this->dlq->add($msg, $error);
        }
    }

    // হিপ থেকে ন্যূনতম অগ্রাধিকারের মেসেজ বের করা
    private function dequeueMin(): Message {
        $min = $this->heap[0];
        $last = array_pop($this->heap);
        if (!empty($this->heap)) {
            $this->heap[0] = $last;
            $this->bubbleDown(0);
        }
        return $min;
    }

    private function bubbleUp(int $i): void {
        while ($i > 0) {
            $parent = intdiv($i - 1, 2);
            if ($this->heap[$i]->priority < $this->heap[$parent]->priority) {
                [$this->heap[$i], $this->heap[$parent]] = [$this->heap[$parent], $this->heap[$i]];
                $i = $parent;
            } else break;
        }
    }

    private function bubbleDown(int $i): void {
        $n = count($this->heap);
        while (true) {
            $smallest = $i;
            $l = 2 * $i + 1;
            $r = 2 * $i + 2;
            if ($l < $n && $this->heap[$l]->priority < $this->heap[$smallest]->priority) $smallest = $l;
            if ($r < $n && $this->heap[$r]->priority < $this->heap[$smallest]->priority) $smallest = $r;
            if ($smallest !== $i) {
                [$this->heap[$i], $this->heap[$smallest]] = [$this->heap[$smallest], $this->heap[$i]];
                $i = $smallest;
            } else break;
        }
    }

    public function getDeadLetterQueue(): DeadLetterQueue { return $this->dlq; }
}

// === ব্যবহার ===
$mq = new MessageQueue();

// কনজ্যুমার নিবন্ধন — এলোমেলোভাবে ব্যর্থ হয় (পরীক্ষার জন্য)
$mq->subscribe(function(Message $msg) {
    if (rand(0, 3) === 0) {
        throw new \Exception("প্রক্রিয়াকরণ ত্রুটি!");
    }
    echo "  📨 মেসেজ: " . json_encode($msg->payload) . "\n";
});

// প্রডিউসার — বিভিন্ন অগ্রাধিকারে মেসেজ পাঠানো
$mq->publish(new Message(["type" => "log", "text" => "সাধারণ লগ"], 5));
$mq->publish(new Message(["type" => "alert", "text" => "জরুরি সতর্কতা!"], 1));
$mq->publish(new Message(["type" => "email", "text" => "ইমেইল পাঠাও"], 3));

// প্রক্রিয়া শুরু
$mq->consume();

echo "\n💀 ডেড লেটার কিউতে মোট: " . $mq->getDeadLetterQueue()->count() . " টি মেসেজ\n";
```

### JavaScript বাস্তবায়ন

```javascript
// মেসেজ ক্লাস
class Message {
    constructor(payload, priority = 5, maxRetries = 3) {
        this.id = `msg_${Date.now()}_${Math.random().toString(36).substr(2, 5)}`;
        this.payload = payload;       // মূল তথ্য
        this.priority = priority;     // অগ্রাধিকার
        this.retryCount = 0;          // পুনঃচেষ্টার সংখ্যা
        this.maxRetries = maxRetries;  // সর্বাধিক পুনঃচেষ্টা
        this.createdAt = new Date();
    }
}

// ডেড লেটার কিউ — সম্পূর্ণ ব্যর্থ মেসেজ
class DeadLetterQueue {
    constructor() { this.messages = []; }

    add(msg, reason) {
        this.messages.push({
            message: msg,
            reason,
            failedAt: new Date().toISOString(),
        });
    }

    getAll() { return this.messages; }
    count() { return this.messages.length; }
}

// মূল মেসেজ কিউ সিস্টেম
class MessageQueueSystem {
    constructor() {
        this.heap = [];                // অগ্রাধিকার হিপ
        this.dlq = new DeadLetterQueue();
        this.consumers = [];
    }

    // প্রডিউসার — মেসেজ প্রকাশ
    publish(msg) {
        this.heap.push(msg);
        this._bubbleUp(this.heap.length - 1);
        console.log(`📤 প্রকাশিত: [${msg.id}] অগ্রাধিকার: ${msg.priority}`);
    }

    // কনজ্যুমার নিবন্ধন
    subscribe(handler) {
        this.consumers.push(handler);
    }

    // সব মেসেজ প্রক্রিয়া
    async consume() {
        while (this.heap.length > 0) {
            const msg = this._dequeueMin();

            for (const handler of this.consumers) {
                try {
                    await handler(msg);
                    console.log(`✅ প্রক্রিয়াকৃত: [${msg.id}]`);
                } catch (err) {
                    this._handleFailure(msg, err.message);
                }
            }
        }
    }

    // ব্যর্থতা — পুনঃচেষ্টা বা DLQ
    _handleFailure(msg, error) {
        msg.retryCount++;
        if (msg.retryCount <= msg.maxRetries) {
            console.log(`🔄 পুনঃচেষ্টা (${msg.retryCount}/${msg.maxRetries}): [${msg.id}]`);
            this.publish(msg);
        } else {
            console.log(`💀 ডেড লেটার কিউতে: [${msg.id}] কারণ: ${error}`);
            this.dlq.add(msg, error);
        }
    }

    _dequeueMin() {
        const min = this.heap[0];
        const last = this.heap.pop();
        if (this.heap.length > 0) {
            this.heap[0] = last;
            this._bubbleDown(0);
        }
        return min;
    }

    _bubbleUp(i) {
        while (i > 0) {
            const parent = Math.floor((i - 1) / 2);
            if (this.heap[i].priority < this.heap[parent].priority) {
                [this.heap[i], this.heap[parent]] = [this.heap[parent], this.heap[i]];
                i = parent;
            } else break;
        }
    }

    _bubbleDown(i) {
        const n = this.heap.length;
        while (true) {
            let smallest = i;
            const l = 2 * i + 1, r = 2 * i + 2;
            if (l < n && this.heap[l].priority < this.heap[smallest].priority) smallest = l;
            if (r < n && this.heap[r].priority < this.heap[smallest].priority) smallest = r;
            if (smallest !== i) {
                [this.heap[i], this.heap[smallest]] = [this.heap[smallest], this.heap[i]];
                i = smallest;
            } else break;
        }
    }

    getDeadLetterQueue() { return this.dlq; }
}

// === ব্যবহার ===
const mq = new MessageQueueSystem();

// কনজ্যুমার — এলোমেলোভাবে ব্যর্থ হয়
mq.subscribe(async (msg) => {
    if (Math.random() < 0.25) throw new Error("প্রক্রিয়াকরণ ত্রুটি!");
    console.log(`  📨 মেসেজ: ${JSON.stringify(msg.payload)}`);
});

// মেসেজ পাঠানো
mq.publish(new Message({ type: "log", text: "সাধারণ লগ" }, 5));
mq.publish(new Message({ type: "alert", text: "জরুরি সতর্কতা!" }, 1));
mq.publish(new Message({ type: "email", text: "ইমেইল পাঠাও" }, 3));

// প্রক্রিয়া
mq.consume().then(() => {
    console.log(`\n💀 DLQ তে মোট: ${mq.getDeadLetterQueue().count()} টি মেসেজ`);
});
```

---

## স্ট্যাক বনাম কিউ তুলনা

| বৈশিষ্ট্য | স্ট্যাক (Stack) | কিউ (Queue) |
|-----------|----------------|-------------|
| **নীতি** | LIFO (শেষে আসা আগে যায়) | FIFO (আগে আসা আগে যায়) |
| **যোগ** | push (শীর্ষে) | enqueue (পেছনে) |
| **সরানো** | pop (শীর্ষ থেকে) | dequeue (সামনে থেকে) |
| **দেখা** | peek/top | front/peek |
| **বাস্তব উদাহরণ** | থালার স্তূপ, Undo | ব্যাংকের লাইন, প্রিন্টার |
| **ব্যবহার** | DFS, Expression Eval, Backtracking | BFS, Scheduling, Buffering |
| **পুনরাবৃত্তি** | DFS (গভীরতা প্রথম) | BFS (প্রশস্ততা প্রথম) |

### 📊 Big O তুলনা

| অপারেশন | স্ট্যাক | কিউ | ডেক | অগ্রাধিকার কিউ |
|----------|--------|------|------|---------------|
| Insert | O(1) | O(1) | O(1) | O(log n) |
| Delete | O(1) | O(1) | O(1) | O(log n) |
| Peek | O(1) | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(n) | O(n) |

---

## ইন্টারভিউ টিপস

### 🎯 স্ট্যাক সম্পর্কিত মনে রাখো

1. **মনোটনিক স্ট্যাক** — Next Greater/Smaller Element সমস্যায় ব্যবহার করো
2. **বন্ধনী সমস্যা** — সর্বদা স্ট্যাক মনে করো
3. **রিকার্সন → ইটারেশন** রূপান্তরে স্ট্যাক ব্যবহার করো
4. **DFS** ট্রাভার্সালে স্ট্যাক (বা রিকার্সন) ব্যবহৃত হয়
5. **Expression মূল্যায়ন** — পোস্টফিক্স / প্রিফিক্সে স্ট্যাক লাগে

### 🎯 কিউ সম্পর্কিত মনে রাখো

1. **BFS** ট্রাভার্সালে কিউ অপরিহার্য
2. **স্লাইডিং উইন্ডো** সমস্যায় ডেক/মনোটনিক কিউ ভাবো
3. **দুটি স্ট্যাক দিয়ে কিউ** — amortized O(1) dequeue মনে রাখো
4. **সার্কুলার কিউ** — মডুলার গণিত `(i + 1) % n` ব্যবহার করো
5. **প্রসেস শিডিউলিং** ও **ক্যাশিং** সমস্যায় কিউ ভাবো

### ⚠️ সাধারণ ভুল এড়াও

- খালি স্ট্যাক/কিউতে pop/dequeue করার আগে isEmpty চেক করো
- সার্কুলার কিউতে মডুলার গণিত ভুলে যাওয়া
- Priority Queue তে হিপ ব্যবহার না করে O(n) অপারেশন করা
- স্ট্যাক ওভারফ্লো এড়াতে গভীর রিকার্সনে ইটারেটিভ পদ্ধতি ব্যবহার করো

---

> 📌 **পরবর্তী বিষয়:** [ট্রি (Tree) — বাইনারি ট্রি, BST, হিপ →](./tree.md)
