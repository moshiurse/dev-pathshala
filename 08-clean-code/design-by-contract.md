# 🤝 ডিজাইন বাই কন্ট্র্যাক্ট (Design by Contract - DbC)

> "সফটওয়্যার কম্পোনেন্টগুলোর মধ্যে একটি আনুষ্ঠানিক চুক্তি থাকা উচিত — ঠিক যেমন ব্যবসায়িক চুক্তিতে দুই পক্ষের দায়িত্ব ও অধিকার স্পষ্ট থাকে।"
> — **বার্ট্রান্ড মেয়ার (Bertrand Meyer)**, Eiffel প্রোগ্রামিং ভাষার স্রষ্টা

---

## 📖 সংজ্ঞা (Definition)

**ডিজাইন বাই কন্ট্র্যাক্ট (DbC)** হলো একটি সফটওয়্যার ডিজাইন পদ্ধতি যেখানে সফটওয়্যার কম্পোনেন্টগুলোর মধ্যে
**আনুষ্ঠানিক, সুনির্দিষ্ট এবং যাচাইযোগ্য ইন্টারফেস স্পেসিফিকেশন** নির্ধারণ করা হয়। এই স্পেসিফিকেশনগুলো
তিনটি মূল উপাদান নিয়ে গঠিত:

| উপাদান | ইংরেজি | বর্ণনা |
|--------|---------|--------|
| **পূর্বশর্ত** | Precondition | ফাংশন কল করার আগে কী সত্য হতে হবে |
| **পরশর্ত** | Postcondition | ফাংশন শেষ হওয়ার পর কী সত্য হবে |
| **ক্লাস ইনভ্যারিয়েন্ট** | Class Invariant | অবজেক্টের জীবনকালে সবসময় কী সত্য থাকবে |

### DbC-এর মূল ধারণা (ASCII ডায়াগ্রাম):

```
┌─────────────────────────────────────────────────┐
│              কন্ট্র্যাক্ট (Contract)              │
│                                                   │
│  ┌───────────────┐     ┌───────────────┐         │
│  │   কলার        │     │   সার্ভিস      │         │
│  │  (Caller)     │────▶│  (Supplier)    │         │
│  │               │     │               │         │
│  │ দায়িত্ব:      │     │ দায়িত্ব:      │         │
│  │ পূর্বশর্ত     │     │ পরশর্ত        │         │
│  │ পূরণ করা     │     │ পূরণ করা     │         │
│  └───────────────┘     └───────────────┘         │
│                                                   │
│  ক্লাস ইনভ্যারিয়েন্ট: সবসময় সত্য থাকবে          │
└─────────────────────────────────────────────────┘
```

---

## 🏠 বাস্তব জীবনের উদাহরণ (Real-life Analogy)

### 🏦 ব্যাংক থেকে টাকা তোলার চুক্তি

ধরুন আপনি ব্যাংকে গেলেন টাকা তুলতে। এখানে একটি অদৃশ্য চুক্তি কাজ করে:

```
┌─────────────────────────────────────────────────────┐
│           ব্যাংক থেকে টাকা তোলার চুক্তি             │
├─────────────────────────────────────────────────────┤
│                                                       │
│  📋 পূর্বশর্ত (আপনার দায়িত্ব):                       │
│  ├── ✅ বৈধ অ্যাকাউন্ট নম্বর দিতে হবে                │
│  ├── ✅ অ্যাকাউন্টে পর্যাপ্ত টাকা থাকতে হবে          │
│  ├── ✅ তোলার পরিমাণ ধনাত্মক হতে হবে                │
│  └── ✅ সঠিক পরিচয়পত্র দেখাতে হবে                  │
│                                                       │
│  📋 পরশর্ত (ব্যাংকের দায়িত্ব):                       │
│  ├── ✅ সঠিক পরিমাণ টাকা দিতে হবে                   │
│  ├── ✅ অ্যাকাউন্ট থেকে সঠিক টাকা কাটতে হবে        │
│  └── ✅ রসিদ/স্লিপ দিতে হবে                          │
│                                                       │
│  📋 ইনভ্যারিয়েন্ট (সবসময় সত্য):                     │
│  ├── ✅ ব্যালেন্স কখনো ঋণাত্মক হবে না               │
│  └── ✅ সকল লেনদেন রেকর্ড থাকবে                     │
│                                                       │
└─────────────────────────────────────────────────────┘
```

> 💡 **মূল শিক্ষা:** যদি আপনি (কলার) পূর্বশর্ত পূরণ করেন, তাহলে ব্যাংক (সার্ভিস) গ্যারান্টি দেয় যে
> পরশর্ত পূরণ হবে। আর ইনভ্যারিয়েন্ট সবসময় বজায় থাকবে।

---

## ❌ খারাপ উদাহরণ (কন্ট্র্যাক্ট ছাড়া)

### PHP — কন্ট্র্যাক্ট ছাড়া ব্যাংক অ্যাকাউন্ট:

```php
<?php
// ❌ খারাপ: কোনো চুক্তি/যাচাই নেই
class BankAccount
{
    public $balance; // পাবলিক — যে কেউ পরিবর্তন করতে পারে!

    public function withdraw($amount)
    {
        // কোনো পূর্বশর্ত যাচাই নেই
        // টাকার পরিমাণ ঋণাত্মক হতে পারে!
        // ব্যালেন্সের চেয়ে বেশি তুলতে পারে!
        $this->balance -= $amount;
        return $amount;
    }

    public function deposit($amount)
    {
        // কোনো যাচাই নেই — ঋণাত্মক ডিপোজিট সম্ভব!
        $this->balance += $amount;
    }
}

// সমস্যা দেখুন:
$account = new BankAccount();
$account->balance = 1000;
$account->withdraw(-500);   // ব্যালেন্স বেড়ে যাবে! বাগ!
$account->withdraw(2000);   // ব্যালেন্স ঋণাত্মক হবে! বাগ!
$account->balance = -99999; // সরাসরি পরিবর্তন সম্ভব! বাগ!
```

### JavaScript — কন্ট্র্যাক্ট ছাড়া ব্যাংক অ্যাকাউন্ট:

```javascript
// ❌ খারাপ: কোনো চুক্তি/যাচাই নেই
class BankAccount {
    constructor() {
        this.balance = 0; // সরাসরি অ্যাক্সেসযোগ্য
    }

    withdraw(amount) {
        // কোনো পূর্বশর্ত যাচাই নেই
        this.balance -= amount;
        return amount;
    }

    deposit(amount) {
        // কোনো যাচাই নেই
        this.balance += amount;
    }
}

// সমস্যা দেখুন:
const account = new BankAccount();
account.deposit(1000);
account.withdraw(-500);   // ব্যালেন্স বেড়ে যাবে! বাগ!
account.withdraw(2000);   // ব্যালেন্স ঋণাত্মক! বাগ!
account.balance = -99999; // সরাসরি পরিবর্তন! বাগ!
```

---

## ✅ ভালো উদাহরণ (কন্ট্র্যাক্ট সহ)

### PHP — কন্ট্র্যাক্ট সহ ব্যাংক অ্যাকাউন্ট:

```php
<?php
// ✅ ভালো: সম্পূর্ণ কন্ট্র্যাক্ট সহ ব্যাংক অ্যাকাউন্ট

/**
 * কন্ট্র্যাক্ট লঙ্ঘনের জন্য কাস্টম এক্সেপশন
 */
class ContractViolationException extends RuntimeException {}
class PreconditionException extends ContractViolationException {}
class PostconditionException extends ContractViolationException {}
class InvariantException extends ContractViolationException {}

class BankAccount
{
    // ব্যালেন্স প্রাইভেট — বাইরে থেকে সরাসরি পরিবর্তন অসম্ভব
    private float $balance;
    private string $accountNumber;

    // ইনভ্যারিয়েন্ট: ব্যালেন্স কখনো ঋণাত্মক হবে না
    // ইনভ্যারিয়েন্ট: অ্যাকাউন্ট নম্বর খালি হবে না

    public function __construct(string $accountNumber, float $initialBalance = 0.0)
    {
        // পূর্বশর্ত যাচাই
        if (empty($accountNumber)) {
            throw new PreconditionException('অ্যাকাউন্ট নম্বর খালি হতে পারে না');
        }
        if ($initialBalance < 0) {
            throw new PreconditionException('প্রারম্ভিক ব্যালেন্স ঋণাত্মক হতে পারে না');
        }

        $this->accountNumber = $accountNumber;
        $this->balance = $initialBalance;

        // ইনভ্যারিয়েন্ট যাচাই
        $this->checkInvariant();
    }

    /**
     * টাকা তোলা
     *
     * পূর্বশর্ত: পরিমাণ ০-এর বেশি হতে হবে
     * পূর্বশর্ত: পরিমাণ ব্যালেন্সের চেয়ে বেশি হতে পারবে না
     * পরশর্ত: ব্যালেন্স সঠিকভাবে কমবে
     * পরশর্ত: সঠিক পরিমাণ ফেরত আসবে
     */
    public function withdraw(float $amount): float
    {
        // পূর্বশর্ত যাচাই
        if ($amount <= 0) {
            throw new PreconditionException(
                "তোলার পরিমাণ ধনাত্মক হতে হবে। পাওয়া গেছে: {$amount}"
            );
        }
        if ($amount > $this->balance) {
            throw new PreconditionException(
                "অপর্যাপ্ত ব্যালেন্স। চাওয়া: {$amount}, আছে: {$this->balance}"
            );
        }

        // আগের ব্যালেন্স সংরক্ষণ (পরশর্ত যাচাইয়ের জন্য)
        $previousBalance = $this->balance;

        // মূল কাজ
        $this->balance -= $amount;

        // পরশর্ত যাচাই
        $expectedBalance = $previousBalance - $amount;
        if (abs($this->balance - $expectedBalance) > 0.001) {
            throw new PostconditionException(
                "ব্যালেন্স সঠিকভাবে আপডেট হয়নি। প্রত্যাশিত: {$expectedBalance}, পাওয়া: {$this->balance}"
            );
        }

        // ইনভ্যারিয়েন্ট যাচাই
        $this->checkInvariant();

        return $amount;
    }

    /**
     * টাকা জমা
     *
     * পূর্বশর্ত: পরিমাণ ০-এর বেশি হতে হবে
     * পরশর্ত: ব্যালেন্স সঠিকভাবে বাড়বে
     */
    public function deposit(float $amount): void
    {
        // পূর্বশর্ত যাচাই
        if ($amount <= 0) {
            throw new PreconditionException(
                "জমার পরিমাণ ধনাত্মক হতে হবে। পাওয়া গেছে: {$amount}"
            );
        }

        $previousBalance = $this->balance;

        // মূল কাজ
        $this->balance += $amount;

        // পরশর্ত যাচাই
        if (abs($this->balance - ($previousBalance + $amount)) > 0.001) {
            throw new PostconditionException('ব্যালেন্স সঠিকভাবে আপডেট হয়নি');
        }

        // ইনভ্যারিয়েন্ট যাচাই
        $this->checkInvariant();
    }

    /**
     * ব্যালেন্স দেখা (শুধু পড়া — কোনো পরিবর্তন নেই)
     */
    public function getBalance(): float
    {
        return $this->balance;
    }

    /**
     * ক্লাস ইনভ্যারিয়েন্ট যাচাই
     * এই শর্তগুলো অবজেক্টের জীবনকালে সবসময় সত্য থাকবে
     */
    private function checkInvariant(): void
    {
        if ($this->balance < 0) {
            throw new InvariantException(
                "ইনভ্যারিয়েন্ট লঙ্ঘন: ব্যালেন্স ঋণাত্মক ({$this->balance})"
            );
        }
        if (empty($this->accountNumber)) {
            throw new InvariantException(
                'ইনভ্যারিয়েন্ট লঙ্ঘন: অ্যাকাউন্ট নম্বর খালি'
            );
        }
    }
}

// ✅ সঠিক ব্যবহার:
$account = new BankAccount('ACC-001', 1000.0);
$account->deposit(500);        // ব্যালেন্স: ১৫০০
$account->withdraw(200);       // ব্যালেন্স: ১৩০০
echo $account->getBalance();   // ১৩০০

// ❌ এগুলো এক্সেপশন ছুড়বে:
// $account->withdraw(-100);   // PreconditionException!
// $account->withdraw(5000);   // PreconditionException!
// $account->deposit(-50);     // PreconditionException!
```

### JavaScript — কন্ট্র্যাক্ট সহ ব্যাংক অ্যাকাউন্ট:

```javascript
// ✅ ভালো: সম্পূর্ণ কন্ট্র্যাক্ট সহ ব্যাংক অ্যাকাউন্ট

// কন্ট্র্যাক্ট লঙ্ঘনের জন্য কাস্টম এরর ক্লাস
class ContractViolationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'ContractViolationError';
    }
}

class PreconditionError extends ContractViolationError {
    constructor(message) {
        super(message);
        this.name = 'PreconditionError';
    }
}

class PostconditionError extends ContractViolationError {
    constructor(message) {
        super(message);
        this.name = 'PostconditionError';
    }
}

class InvariantError extends ContractViolationError {
    constructor(message) {
        super(message);
        this.name = 'InvariantError';
    }
}

class BankAccount {
    // প্রাইভেট ফিল্ড — বাইরে থেকে অ্যাক্সেস অসম্ভব
    #balance;
    #accountNumber;

    constructor(accountNumber, initialBalance = 0) {
        // পূর্বশর্ত যাচাই
        if (!accountNumber || typeof accountNumber !== 'string') {
            throw new PreconditionError('অ্যাকাউন্ট নম্বর একটি অ-খালি স্ট্রিং হতে হবে');
        }
        if (typeof initialBalance !== 'number' || initialBalance < 0) {
            throw new PreconditionError('প্রারম্ভিক ব্যালেন্স একটি অ-ঋণাত্মক সংখ্যা হতে হবে');
        }

        this.#accountNumber = accountNumber;
        this.#balance = initialBalance;

        // ইনভ্যারিয়েন্ট যাচাই
        this.#checkInvariant();
    }

    /**
     * টাকা তোলা
     * পূর্বশর্ত: পরিমাণ ধনাত্মক সংখ্যা এবং ব্যালেন্সের চেয়ে কম বা সমান
     * পরশর্ত: ব্যালেন্স সঠিকভাবে হ্রাস পাবে
     */
    withdraw(amount) {
        // পূর্বশর্ত যাচাই
        if (typeof amount !== 'number' || amount <= 0) {
            throw new PreconditionError(
                `তোলার পরিমাণ ধনাত্মক সংখ্যা হতে হবে। পাওয়া গেছে: ${amount}`
            );
        }
        if (amount > this.#balance) {
            throw new PreconditionError(
                `অপর্যাপ্ত ব্যালেন্স। চাওয়া: ${amount}, আছে: ${this.#balance}`
            );
        }

        const previousBalance = this.#balance;

        // মূল কাজ
        this.#balance -= amount;

        // পরশর্ত যাচাই
        const expectedBalance = previousBalance - amount;
        if (Math.abs(this.#balance - expectedBalance) > 0.001) {
            throw new PostconditionError(
                `ব্যালেন্স সঠিকভাবে আপডেট হয়নি`
            );
        }

        // ইনভ্যারিয়েন্ট যাচাই
        this.#checkInvariant();

        return amount;
    }

    /**
     * টাকা জমা
     * পূর্বশর্ত: পরিমাণ ধনাত্মক সংখ্যা
     * পরশর্ত: ব্যালেন্স সঠিকভাবে বৃদ্ধি পাবে
     */
    deposit(amount) {
        // পূর্বশর্ত যাচাই
        if (typeof amount !== 'number' || amount <= 0) {
            throw new PreconditionError(
                `জমার পরিমাণ ধনাত্মক সংখ্যা হতে হবে। পাওয়া গেছে: ${amount}`
            );
        }

        const previousBalance = this.#balance;

        // মূল কাজ
        this.#balance += amount;

        // পরশর্ত যাচাই
        if (Math.abs(this.#balance - (previousBalance + amount)) > 0.001) {
            throw new PostconditionError('ব্যালেন্স সঠিকভাবে আপডেট হয়নি');
        }

        // ইনভ্যারিয়েন্ট যাচাই
        this.#checkInvariant();
    }

    get balance() {
        return this.#balance;
    }

    // ক্লাস ইনভ্যারিয়েন্ট — সবসময় সত্য থাকতে হবে
    #checkInvariant() {
        if (this.#balance < 0) {
            throw new InvariantError(
                `ইনভ্যারিয়েন্ট লঙ্ঘন: ব্যালেন্স ঋণাত্মক (${this.#balance})`
            );
        }
        if (!this.#accountNumber) {
            throw new InvariantError(
                'ইনভ্যারিয়েন্ট লঙ্ঘন: অ্যাকাউন্ট নম্বর খালি'
            );
        }
    }
}

// ✅ সঠিক ব্যবহার:
const account = new BankAccount('ACC-001', 1000);
account.deposit(500);         // ব্যালেন্স: ১৫০০
account.withdraw(200);        // ব্যালেন্স: ১৩০০
console.log(account.balance); // ১৩০০

// ❌ এগুলো এরর ছুড়বে:
// account.withdraw(-100);    // PreconditionError!
// account.withdraw(5000);    // PreconditionError!
// account.deposit(-50);      // PreconditionError!
```

---

## 📚 গভীর আলোচনা (Deep Dive)

---

### 🔹 ১. DbC-এর মূল ধারণা — বার্ট্রান্ড মেয়ার এবং Eiffel

**ডিজাইন বাই কন্ট্র্যাক্ট** ধারণাটি ১৯৮৬ সালে **বার্ট্রান্ড মেয়ার** তাঁর **Eiffel** প্রোগ্রামিং ভাষায় প্রথম
আনুষ্ঠানিকভাবে উপস্থাপন করেন। এটি **হোয়ার লজিক (Hoare Logic)** থেকে অনুপ্রাণিত, যেখানে প্রতিটি
প্রোগ্রাম স্টেটমেন্টের জন্য তিনটি জিনিস সংজ্ঞায়িত করা হয়:

```
{ P } S { Q }

যেখানে:
  P = পূর্বশর্ত (Precondition) — S চালানোর আগে কী সত্য
  S = স্টেটমেন্ট — কাজটি যা করা হবে
  Q = পরশর্ত (Postcondition) — S চালানোর পরে কী সত্য হবে
```

#### DbC-এর মূল নীতিমালা:

| নীতি | বর্ণনা |
|------|--------|
| **দায়িত্ব বণ্টন** | কলার ও সার্ভিসের মধ্যে দায়িত্ব স্পষ্টভাবে ভাগ |
| **ব্যর্থতায় দ্রুত সনাক্তকরণ** | ভুল ইনপুট তাৎক্ষণিক ধরা পড়ে |
| **ডকুমেন্টেশন হিসেবে কন্ট্র্যাক্ট** | কোড নিজেই ডকুমেন্টেশন হিসেবে কাজ করে |
| **ডিবাগিং সহজতর** | কোন চুক্তি ভেঙেছে তা স্পষ্ট থাকে |

---

### 🔹 ২. পূর্বশর্ত (Preconditions) — বিস্তারিত

পূর্বশর্ত হলো সেই শর্তগুলো যা একটি ফাংশন বা মেথড কল করার **আগে** সত্য হতে হবে।
এটি **কলারের দায়িত্ব** পূরণ করা।

#### PHP উদাহরণ — পূর্বশর্ত প্যাটার্ন:

```php
<?php
// পূর্বশর্ত যাচাইয়ের বিভিন্ন পদ্ধতি

class UserService
{
    /**
     * ব্যবহারকারী নিবন্ধন
     *
     * পূর্বশর্ত:
     * - ইমেইল অবশ্যই বৈধ ফরম্যাটে হতে হবে
     * - পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে
     * - নাম খালি হতে পারবে না
     */
    public function register(string $name, string $email, string $password): User
    {
        // পূর্বশর্ত ১: নাম খালি কিনা
        if (trim($name) === '') {
            throw new PreconditionException('নাম খালি হতে পারে না');
        }

        // পূর্বশর্ত ২: ইমেইল বৈধতা
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new PreconditionException("অবৈধ ইমেইল ফরম্যাট: {$email}");
        }

        // পূর্বশর্ত ৩: পাসওয়ার্ডের দৈর্ঘ্য
        if (strlen($password) < 8) {
            throw new PreconditionException(
                'পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে'
            );
        }

        // সব পূর্বশর্ত পূরণ — মূল কাজ চালানো নিরাপদ
        $user = new User($name, $email, $password);
        $this->repository->save($user);

        return $user;
    }
}
```

#### JavaScript উদাহরণ — পূর্বশর্ত হেল্পার:

```javascript
// পুনরায় ব্যবহারযোগ্য পূর্বশর্ত যাচাই হেল্পার

class Contract {
    // পূর্বশর্ত যাচাই — ব্যর্থ হলে PreconditionError ছুড়বে
    static requires(condition, message) {
        if (!condition) {
            throw new PreconditionError(message);
        }
    }

    // পরশর্ত যাচাই — ব্যর্থ হলে PostconditionError ছুড়বে
    static ensures(condition, message) {
        if (!condition) {
            throw new PostconditionError(message);
        }
    }

    // ইনভ্যারিয়েন্ট যাচাই
    static invariant(condition, message) {
        if (!condition) {
            throw new InvariantError(message);
        }
    }
}

class UserService {
    register(name, email, password) {
        // পূর্বশর্ত যাচাই — সংক্ষিপ্ত ও পরিষ্কার
        Contract.requires(
            typeof name === 'string' && name.trim().length > 0,
            'নাম একটি অ-খালি স্ট্রিং হতে হবে'
        );
        Contract.requires(
            /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email),
            `অবৈধ ইমেইল ফরম্যাট: ${email}`
        );
        Contract.requires(
            typeof password === 'string' && password.length >= 8,
            'পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে'
        );

        // মূল কাজ
        const user = { name: name.trim(), email, createdAt: new Date() };
        this.repository.save(user);

        // পরশর্ত যাচাই
        Contract.ensures(user.createdAt !== null, 'ব্যবহারকারীর তৈরির সময় সেট হয়নি');

        return user;
    }
}
```

---

### 🔹 ৩. পরশর্ত (Postconditions) — বিস্তারিত

পরশর্ত হলো সেই শর্তগুলো যা ফাংশন বা মেথড সফলভাবে সম্পন্ন হওয়ার **পরে** সত্য হতে হবে।
এটি **সার্ভিসের দায়িত্ব** পূরণ করা।

#### PHP উদাহরণ — পরশর্ত:

```php
<?php
class SortingService
{
    /**
     * অ্যারে সর্ট করা
     *
     * পূর্বশর্ত: ইনপুট অ্যারে খালি নয়
     * পরশর্ত: ফলাফল অ্যারে ছোট থেকে বড় ক্রমে সাজানো
     * পরশর্ত: ফলাফল অ্যারের দৈর্ঘ্য ইনপুটের সমান
     * পরশর্ত: ইনপুটের সব উপাদান ফলাফলে আছে
     */
    public function sort(array $input): array
    {
        // পূর্বশর্ত
        if (empty($input)) {
            throw new PreconditionException('অ্যারে খালি হতে পারে না');
        }

        $originalCount = count($input);

        // মূল কাজ — সর্টিং
        $result = $input;
        sort($result);

        // পরশর্ত ১: দৈর্ঘ্য সমান
        if (count($result) !== $originalCount) {
            throw new PostconditionException(
                'সর্ট করা অ্যারের দৈর্ঘ্য ইনপুটের সাথে মেলে না'
            );
        }

        // পরশর্ত ২: সঠিকভাবে সাজানো হয়েছে
        for ($i = 1; $i < count($result); $i++) {
            if ($result[$i] < $result[$i - 1]) {
                throw new PostconditionException(
                    "অ্যারে সঠিকভাবে সাজানো হয়নি: ইন্ডেক্স {$i}"
                );
            }
        }

        return $result;
    }
}
```

#### JavaScript উদাহরণ — পরশর্ত:

```javascript
class MathService {
    /**
     * বর্গমূল বের করা
     *
     * পূর্বশর্ত: সংখ্যা অ-ঋণাত্মক হতে হবে
     * পরশর্ত: ফলাফল * ফলাফল ≈ ইনপুট (ভাসমান বিন্দু সহনশীলতা সহ)
     * পরশর্ত: ফলাফল অ-ঋণাত্মক
     */
    squareRoot(number) {
        // পূর্বশর্ত
        Contract.requires(
            typeof number === 'number' && number >= 0,
            `ইনপুট অ-ঋণাত্মক সংখ্যা হতে হবে। পাওয়া গেছে: ${number}`
        );

        // মূল কাজ
        const result = Math.sqrt(number);

        // পরশর্ত ১: ফলাফল অ-ঋণাত্মক
        Contract.ensures(
            result >= 0,
            'বর্গমূল ঋণাত্মক হতে পারে না'
        );

        // পরশর্ত ২: result² ≈ number
        Contract.ensures(
            Math.abs(result * result - number) < 0.0001,
            `বর্গমূল সঠিক নয়: ${result}² = ${result * result}, প্রত্যাশিত ≈ ${number}`
        );

        return result;
    }
}
```

---

### 🔹 ৪. ক্লাস ইনভ্যারিয়েন্ট (Class Invariants) — বিস্তারিত

ক্লাস ইনভ্যারিয়েন্ট হলো এমন শর্ত যা একটি অবজেক্টের **সম্পূর্ণ জীবনকালে** সত্য থাকতে হবে —
কনস্ট্রাক্টরের পরে এবং প্রতিটি পাবলিক মেথডের আগে ও পরে।

```
অবজেক্ট জীবনচক্র:

  ┌──────────┐
  │ তৈরি হওয়া│──▶ ইনভ্যারিয়েন্ট সত্য ✅
  └────┬─────┘
       │
  ┌────▼──────┐
  │ মেথড কল ১ │──▶ আগে: ইনভ্যারিয়েন্ট সত্য ✅
  │           │──▶ পরে: ইনভ্যারিয়েন্ট সত্য ✅
  └────┬──────┘
       │
  ┌────▼──────┐
  │ মেথড কল ২ │──▶ আগে: ইনভ্যারিয়েন্ট সত্য ✅
  │           │──▶ পরে: ইনভ্যারিয়েন্ট সত্য ✅
  └────┬──────┘
       │
  ┌────▼──────┐
  │  ধ্বংস    │──▶ ইনভ্যারিয়েন্ট সত্য ✅
  └───────────┘
```

#### PHP উদাহরণ — তারিখের পরিসর ক্লাস:

```php
<?php
/**
 * তারিখের পরিসর (DateRange)
 *
 * ইনভ্যারিয়েন্ট: শুরুর তারিখ সবসময় শেষ তারিখের আগে বা সমান
 */
class DateRange
{
    private DateTimeImmutable $start;
    private DateTimeImmutable $end;

    public function __construct(DateTimeImmutable $start, DateTimeImmutable $end)
    {
        $this->start = $start;
        $this->end = $end;

        // ইনভ্যারিয়েন্ট যাচাই
        $this->checkInvariant();
    }

    /**
     * পরিসর প্রসারিত করা
     * ইনভ্যারিয়েন্ট বজায় রাখতে হবে
     */
    public function extendTo(DateTimeImmutable $newEnd): self
    {
        // পূর্বশর্ত: নতুন শেষ তারিখ বর্তমান শেষের পরে হতে হবে
        if ($newEnd <= $this->end) {
            throw new PreconditionException(
                'নতুন শেষ তারিখ বর্তমান শেষ তারিখের পরে হতে হবে'
            );
        }

        $newRange = new self($this->start, $newEnd);

        // পরশর্ত: পরিসর বড় হয়েছে
        // ইনভ্যারিয়েন্ট: কনস্ট্রাক্টরে যাচাই হয়ে গেছে

        return $newRange;
    }

    /**
     * একটি তারিখ এই পরিসরের মধ্যে আছে কিনা
     */
    public function contains(DateTimeImmutable $date): bool
    {
        return $date >= $this->start && $date <= $this->end;
    }

    // ইনভ্যারিয়েন্ট: শুরু <= শেষ
    private function checkInvariant(): void
    {
        if ($this->start > $this->end) {
            throw new InvariantException(
                'ইনভ্যারিয়েন্ট লঙ্ঘন: শুরুর তারিখ শেষ তারিখের পরে'
            );
        }
    }
}
```

#### JavaScript উদাহরণ — স্ট্যাক ডেটা স্ট্রাকচার:

```javascript
/**
 * সীমিত আকারের স্ট্যাক
 *
 * ইনভ্যারিয়েন্ট:
 * - আকার কখনো ঋণাত্মক নয়
 * - আকার কখনো সর্বোচ্চ ক্ষমতার বেশি নয়
 * - আকার সর্বদা অভ্যন্তরীণ অ্যারের দৈর্ঘ্যের সমান
 */
class BoundedStack {
    #items;
    #maxCapacity;

    constructor(maxCapacity) {
        Contract.requires(
            Number.isInteger(maxCapacity) && maxCapacity > 0,
            'সর্বোচ্চ ক্ষমতা একটি ধনাত্মক পূর্ণসংখ্যা হতে হবে'
        );

        this.#items = [];
        this.#maxCapacity = maxCapacity;

        this.#checkInvariant();
    }

    push(item) {
        // পূর্বশর্ত: স্ট্যাক পূর্ণ নয়
        Contract.requires(
            !this.isFull(),
            `স্ট্যাক পূর্ণ (ক্ষমতা: ${this.#maxCapacity})`
        );

        const previousSize = this.size;

        // মূল কাজ
        this.#items.push(item);

        // পরশর্ত: আকার ১ বেড়েছে
        Contract.ensures(
            this.size === previousSize + 1,
            'পুশের পর আকার সঠিকভাবে বাড়েনি'
        );

        // পরশর্ত: শীর্ষ উপাদান পুশ করা আইটেম
        Contract.ensures(
            this.peek() === item,
            'শীর্ষ উপাদান পুশ করা আইটেম নয়'
        );

        this.#checkInvariant();
    }

    pop() {
        // পূর্বশর্ত: স্ট্যাক খালি নয়
        Contract.requires(
            !this.isEmpty(),
            'খালি স্ট্যাক থেকে পপ করা যায় না'
        );

        const previousSize = this.size;

        // মূল কাজ
        const item = this.#items.pop();

        // পরশর্ত: আকার ১ কমেছে
        Contract.ensures(
            this.size === previousSize - 1,
            'পপের পর আকার সঠিকভাবে কমেনি'
        );

        this.#checkInvariant();
        return item;
    }

    peek() {
        Contract.requires(!this.isEmpty(), 'খালি স্ট্যাকের শীর্ষ দেখা যায় না');
        return this.#items[this.#items.length - 1];
    }

    get size() { return this.#items.length; }
    isFull() { return this.#items.length >= this.#maxCapacity; }
    isEmpty() { return this.#items.length === 0; }

    // ইনভ্যারিয়েন্ট যাচাই
    #checkInvariant() {
        Contract.invariant(
            this.#items.length >= 0,
            'ইনভ্যারিয়েন্ট লঙ্ঘন: আকার ঋণাত্মক'
        );
        Contract.invariant(
            this.#items.length <= this.#maxCapacity,
            `ইনভ্যারিয়েন্ট লঙ্ঘন: আকার (${this.#items.length}) সর্বোচ্চ ক্ষমতা (${this.#maxCapacity}) ছাড়িয়ে গেছে`
        );
    }
}
```

---

### 🔹 ৫. ডিফেন্সিভ প্রোগ্রামিং বনাম DbC

অনেকে **ডিফেন্সিভ প্রোগ্রামিং** এবং **DbC** কে গুলিয়ে ফেলেন। এদের মধ্যে গুরুত্বপূর্ণ পার্থক্য আছে:

```
┌─────────────────────────────────────────────────────────────┐
│          ডিফেন্সিভ প্রোগ্রামিং বনাম DbC                     │
├───────────────────────┬─────────────────────────────────────┤
│ ডিফেন্সিভ প্রোগ্রামিং │ ডিজাইন বাই কন্ট্র্যাক্ট           │
├───────────────────────┼─────────────────────────────────────┤
│ "কারো উপর বিশ্বাস     │ "দায়িত্ব স্পষ্টভাবে ভাগ করো"      │
│  করো না"               │                                    │
├───────────────────────┼─────────────────────────────────────┤
│ সব জায়গায় যাচাই করে │ নির্দিষ্ট জায়গায় যাচাই করে        │
├───────────────────────┼─────────────────────────────────────┤
│ ভুল ইনপুটে ডিফল্ট    │ ভুল ইনপুটে এক্সেপশন ছোড়ে          │
│ মান ফেরত দেয়          │                                    │
├───────────────────────┼─────────────────────────────────────┤
│ বাগ লুকায়             │ বাগ তাৎক্ষণিক প্রকাশ করে           │
├───────────────────────┼─────────────────────────────────────┤
│ কোড ফুলে যায়          │ কোড পরিষ্কার থাকে                  │
├───────────────────────┼─────────────────────────────────────┤
│ দায়িত্ব অস্পষ্ট       │ দায়িত্ব সুনির্দিষ্ট                │
└───────────────────────┴─────────────────────────────────────┘
```

#### PHP তুলনা:

```php
<?php
// ❌ ডিফেন্সিভ প্রোগ্রামিং — বাগ লুকায়!
class DefensiveCalculator
{
    public function divide($a, $b)
    {
        // সমস্যা: শূন্য দিয়ে ভাগ করলে ০ ফেরত দেয় — ভুল উত্তর!
        if ($b == 0) {
            return 0; // নীরবে ভুল ফলাফল দেয়
        }
        // সমস্যা: সংখ্যা না হলেও কাজ করতে চেষ্টা করে
        if (!is_numeric($a)) {
            $a = 0;
        }
        if (!is_numeric($b)) {
            $b = 1;
        }
        return $a / $b;
    }
}

// ✅ DbC পদ্ধতি — বাগ তাৎক্ষণিক ধরা পড়ে
class ContractCalculator
{
    /**
     * ভাগ করা
     *
     * পূর্বশর্ত: $a এবং $b সংখ্যা হতে হবে
     * পূর্বশর্ত: $b শূন্য হতে পারবে না
     * পরশর্ত: ফলাফল * $b ≈ $a
     */
    public function divide(float $a, float $b): float
    {
        // পূর্বশর্ত
        if ($b == 0) {
            throw new PreconditionException('শূন্য দিয়ে ভাগ করা যায় না');
        }

        $result = $a / $b;

        // পরশর্ত: ফলাফল যাচাই
        if (abs($result * $b - $a) > 0.0001) {
            throw new PostconditionException(
                "ভাগফল সঠিক নয়: {$result} * {$b} ≠ {$a}"
            );
        }

        return $result;
    }
}
```

---

### 🔹 ৬. PHP এবং JavaScript-এ DbC প্যাটার্নসমূহ

#### PHP — Assert ব্যবহার করে DbC:

```php
<?php
/**
 * PHP-তে assert() ব্যবহার করে কন্ট্র্যাক্ট
 * ডেভেলপমেন্টে assert সক্রিয়, প্রোডাকশনে নিষ্ক্রিয় করা যায়
 */
class ProductCatalog
{
    private array $products = [];

    /**
     * পণ্য যোগ করা
     *
     * পূর্বশর্ত: নাম খালি নয়, দাম ধনাত্মক
     * পরশর্ত: পণ্য ক্যাটালগে যোগ হয়েছে
     * ইনভ্যারিয়েন্ট: সব পণ্যের দাম ধনাত্মক
     */
    public function addProduct(string $name, float $price): void
    {
        // পূর্বশর্ত
        assert(trim($name) !== '', 'পণ্যের নাম খালি হতে পারে না');
        assert($price > 0, "পণ্যের দাম ধনাত্মক হতে হবে, পাওয়া গেছে: {$price}");

        $previousCount = count($this->products);

        // মূল কাজ
        $this->products[] = ['name' => $name, 'price' => $price];

        // পরশর্ত
        assert(
            count($this->products) === $previousCount + 1,
            'পণ্য সঠিকভাবে যোগ হয়নি'
        );

        // ইনভ্যারিয়েন্ট
        $this->checkInvariant();
    }

    /**
     * পণ্যের দাম আপডেট
     *
     * পূর্বশর্ত: ইন্ডেক্স বৈধ, নতুন দাম ধনাত্মক
     * পরশর্ত: শুধুমাত্র নির্দিষ্ট পণ্যের দাম পরিবর্তন হয়েছে
     */
    public function updatePrice(int $index, float $newPrice): void
    {
        // পূর্বশর্ত
        assert(
            $index >= 0 && $index < count($this->products),
            "অবৈধ ইন্ডেক্স: {$index}"
        );
        assert($newPrice > 0, "দাম ধনাত্মক হতে হবে: {$newPrice}");

        // মূল কাজ
        $this->products[$index]['price'] = $newPrice;

        // পরশর্ত
        assert(
            $this->products[$index]['price'] === $newPrice,
            'দাম সঠিকভাবে আপডেট হয়নি'
        );

        // ইনভ্যারিয়েন্ট
        $this->checkInvariant();
    }

    // ইনভ্যারিয়েন্ট: সব পণ্যের দাম ধনাত্মক
    private function checkInvariant(): void
    {
        foreach ($this->products as $index => $product) {
            assert(
                $product['price'] > 0,
                "ইনভ্যারিয়েন্ট লঙ্ঘন: ইন্ডেক্স {$index}-এ ঋণাত্মক দাম"
            );
        }
    }
}
```

#### JavaScript — Proxy ব্যবহার করে স্বয়ংক্রিয় DbC:

```javascript
/**
 * JavaScript Proxy ব্যবহার করে স্বয়ংক্রিয়ভাবে ইনভ্যারিয়েন্ট যাচাই
 * প্রতিটি মেথড কলের পরে ইনভ্যারিয়েন্ট স্বয়ংক্রিয়ভাবে চেক হয়
 */
function withInvariant(obj, invariantCheck) {
    return new Proxy(obj, {
        get(target, prop) {
            const value = target[prop];
            if (typeof value === 'function') {
                return function (...args) {
                    // মেথড কল
                    const result = value.apply(target, args);
                    // মেথডের পরে ইনভ্যারিয়েন্ট যাচাই
                    invariantCheck(target);
                    return result;
                };
            }
            return value;
        }
    });
}

// ব্যবহার:
class Temperature {
    constructor(celsius) {
        this.celsius = celsius;
    }

    // সেলসিয়াস সেট করা
    setCelsius(value) {
        Contract.requires(
            typeof value === 'number',
            'তাপমাত্রা সংখ্যা হতে হবে'
        );
        this.celsius = value;
    }

    // ফারেনহাইটে রূপান্তর
    toFahrenheit() {
        return this.celsius * 9/5 + 32;
    }
}

// ইনভ্যারিয়েন্ট সহ Temperature তৈরি
const temp = withInvariant(
    new Temperature(25),
    (obj) => {
        // ইনভ্যারিয়েন্ট: তাপমাত্রা পরম শূন্যের (-273.15°C) নিচে যেতে পারে না
        Contract.invariant(
            obj.celsius >= -273.15,
            `ইনভ্যারিয়েন্ট লঙ্ঘন: তাপমাত্রা পরম শূন্যের নিচে (${obj.celsius}°C)`
        );
    }
);

temp.setCelsius(100);  // ✅ কাজ করবে
// temp.setCelsius(-300); // ❌ InvariantError!
```

---

### 🔹 ৭. সম্পূর্ণ উদাহরণ — শপিং কার্ট

এখন আমরা একটি সম্পূর্ণ শপিং কার্ট সিস্টেম তৈরি করব যেখানে DbC-এর সব দিক প্রয়োগ করা হবে।

#### PHP — সম্পূর্ণ শপিং কার্ট:

```php
<?php
/**
 * শপিং কার্ট — সম্পূর্ণ DbC উদাহরণ
 *
 * ক্লাস ইনভ্যারিয়েন্ট:
 * - কার্টে আইটেমের সংখ্যা কখনো ঋণাত্মক নয়
 * - প্রতিটি আইটেমের পরিমাণ কমপক্ষে ১
 * - প্রতিটি আইটেমের দাম ধনাত্মক
 * - মোট মূল্য সবসময় পৃথক আইটেমের যোগফলের সমান
 */
class ShoppingCart
{
    /** @var array<string, array{name: string, price: float, quantity: int}> */
    private array $items = [];

    /**
     * কার্টে আইটেম যোগ করা
     *
     * পূর্বশর্ত: আইটেম আইডি খালি নয়
     * পূর্বশর্ত: নাম খালি নয়
     * পূর্বশর্ত: দাম ধনাত্মক
     * পূর্বশর্ত: পরিমাণ কমপক্ষে ১
     * পরশর্ত: আইটেম কার্টে আছে
     * পরশর্ত: মোট আইটেম সংখ্যা বেড়েছে
     */
    public function addItem(
        string $itemId,
        string $name,
        float $price,
        int $quantity = 1
    ): void {
        // পূর্বশর্ত যাচাই
        if (empty($itemId)) {
            throw new PreconditionException('আইটেম আইডি খালি হতে পারে না');
        }
        if (empty(trim($name))) {
            throw new PreconditionException('আইটেমের নাম খালি হতে পারে না');
        }
        if ($price <= 0) {
            throw new PreconditionException("দাম ধনাত্মক হতে হবে: {$price}");
        }
        if ($quantity < 1) {
            throw new PreconditionException("পরিমাণ কমপক্ষে ১ হতে হবে: {$quantity}");
        }

        $previousTotal = $this->getTotalItems();

        // মূল কাজ
        if (isset($this->items[$itemId])) {
            // বিদ্যমান আইটেমের পরিমাণ বাড়ানো
            $this->items[$itemId]['quantity'] += $quantity;
        } else {
            // নতুন আইটেম যোগ
            $this->items[$itemId] = [
                'name' => $name,
                'price' => $price,
                'quantity' => $quantity,
            ];
        }

        // পরশর্ত যাচাই
        if (!isset($this->items[$itemId])) {
            throw new PostconditionException("আইটেম কার্টে যোগ হয়নি: {$itemId}");
        }
        if ($this->getTotalItems() !== $previousTotal + $quantity) {
            throw new PostconditionException('মোট আইটেম সংখ্যা সঠিকভাবে বাড়েনি');
        }

        // ইনভ্যারিয়েন্ট যাচাই
        $this->checkInvariant();
    }

    /**
     * কার্ট থেকে আইটেম সরানো
     *
     * পূর্বশর্ত: আইটেম কার্টে আছে
     * পূর্বশর্ত: পরিমাণ বৈধ
     * পরশর্ত: আইটেমের পরিমাণ সঠিকভাবে কমেছে
     */
    public function removeItem(string $itemId, int $quantity = 1): void
    {
        // পূর্বশর্ত
        if (!isset($this->items[$itemId])) {
            throw new PreconditionException("আইটেম কার্টে নেই: {$itemId}");
        }
        if ($quantity < 1) {
            throw new PreconditionException("পরিমাণ কমপক্ষে ১ হতে হবে: {$quantity}");
        }
        if ($quantity > $this->items[$itemId]['quantity']) {
            throw new PreconditionException(
                "সরানোর পরিমাণ ({$quantity}) বর্তমান পরিমাণের ({$this->items[$itemId]['quantity']}) চেয়ে বেশি"
            );
        }

        $previousTotal = $this->getTotalItems();

        // মূল কাজ
        $this->items[$itemId]['quantity'] -= $quantity;
        if ($this->items[$itemId]['quantity'] === 0) {
            unset($this->items[$itemId]);
        }

        // পরশর্ত
        if ($this->getTotalItems() !== $previousTotal - $quantity) {
            throw new PostconditionException('আইটেম সংখ্যা সঠিকভাবে কমেনি');
        }

        // ইনভ্যারিয়েন্ট
        $this->checkInvariant();
    }

    /**
     * মোট মূল্য গণনা
     *
     * পরশর্ত: মোট মূল্য অ-ঋণাত্মক
     * পরশর্ত: মোট = প্রতিটি আইটেমের (দাম × পরিমাণ) এর যোগফল
     */
    public function getTotalPrice(): float
    {
        $total = 0.0;
        foreach ($this->items as $item) {
            $total += $item['price'] * $item['quantity'];
        }

        // পরশর্ত
        if ($total < 0) {
            throw new PostconditionException("মোট মূল্য ঋণাত্মক: {$total}");
        }

        return round($total, 2);
    }

    /**
     * মোট আইটেম সংখ্যা
     */
    public function getTotalItems(): int
    {
        $total = 0;
        foreach ($this->items as $item) {
            $total += $item['quantity'];
        }
        return $total;
    }

    /**
     * কার্ট খালি কিনা
     */
    public function isEmpty(): bool
    {
        return empty($this->items);
    }

    /**
     * ক্লাস ইনভ্যারিয়েন্ট যাচাই
     */
    private function checkInvariant(): void
    {
        // ইনভ্যারিয়েন্ট ১: প্রতিটি আইটেমের পরিমাণ ≥ ১
        foreach ($this->items as $id => $item) {
            if ($item['quantity'] < 1) {
                throw new InvariantException(
                    "ইনভ্যারিয়েন্ট লঙ্ঘন: আইটেম '{$id}'-এর পরিমাণ {$item['quantity']} (< ১)"
                );
            }
            // ইনভ্যারিয়েন্ট ২: দাম ধনাত্মক
            if ($item['price'] <= 0) {
                throw new InvariantException(
                    "ইনভ্যারিয়েন্ট লঙ্ঘন: আইটেম '{$id}'-এর দাম {$item['price']} (≤ ০)"
                );
            }
        }

        // ইনভ্যারিয়েন্ট ৩: মোট মূল্য অ-ঋণাত্মক
        $calculatedTotal = 0.0;
        foreach ($this->items as $item) {
            $calculatedTotal += $item['price'] * $item['quantity'];
        }
        if ($calculatedTotal < 0) {
            throw new InvariantException(
                "ইনভ্যারিয়েন্ট লঙ্ঘন: মোট মূল্য ঋণাত্মক ({$calculatedTotal})"
            );
        }
    }
}

// ===== ব্যবহার =====
$cart = new ShoppingCart();

// আইটেম যোগ
$cart->addItem('BOOK-001', 'ক্লিন কোড বই', 450.00, 2);
$cart->addItem('PEN-001', 'কলম সেট', 120.50, 3);

echo "মোট আইটেম: " . $cart->getTotalItems() . "\n";  // ৫
echo "মোট মূল্য: ৳" . $cart->getTotalPrice() . "\n";  // ৳১২৬১.৫০

// আইটেম সরানো
$cart->removeItem('PEN-001', 1);
echo "মোট আইটেম: " . $cart->getTotalItems() . "\n";  // ৪

// ❌ এগুলো এক্সেপশন ছুড়বে:
// $cart->addItem('', 'টেস্ট', 100);        // PreconditionException: খালি আইডি
// $cart->addItem('X', 'টেস্ট', -50);       // PreconditionException: ঋণাত্মক দাম
// $cart->removeItem('INVALID-001');         // PreconditionException: আইটেম নেই
// $cart->removeItem('BOOK-001', 100);      // PreconditionException: বেশি পরিমাণ
```

#### JavaScript — সম্পূর্ণ শপিং কার্ট:

```javascript
/**
 * শপিং কার্ট — সম্পূর্ণ DbC উদাহরণ (JavaScript)
 *
 * ক্লাস ইনভ্যারিয়েন্ট:
 * - প্রতিটি আইটেমের পরিমাণ ≥ ১
 * - প্রতিটি আইটেমের দাম > ০
 * - মোট মূল্য সর্বদা সঠিক
 */
class ShoppingCart {
    #items;

    constructor() {
        this.#items = new Map();
        this.#checkInvariant();
    }

    addItem(itemId, name, price, quantity = 1) {
        // পূর্বশর্ত
        Contract.requires(
            typeof itemId === 'string' && itemId.length > 0,
            'আইটেম আইডি একটি অ-খালি স্ট্রিং হতে হবে'
        );
        Contract.requires(
            typeof name === 'string' && name.trim().length > 0,
            'আইটেমের নাম খালি হতে পারে না'
        );
        Contract.requires(
            typeof price === 'number' && price > 0,
            `দাম ধনাত্মক হতে হবে: ${price}`
        );
        Contract.requires(
            Number.isInteger(quantity) && quantity >= 1,
            `পরিমাণ কমপক্ষে ১ হতে হবে: ${quantity}`
        );

        const previousTotal = this.totalItems;

        // মূল কাজ
        if (this.#items.has(itemId)) {
            const existing = this.#items.get(itemId);
            existing.quantity += quantity;
        } else {
            this.#items.set(itemId, { name, price, quantity });
        }

        // পরশর্ত
        Contract.ensures(
            this.#items.has(itemId),
            `আইটেম কার্টে যোগ হয়নি: ${itemId}`
        );
        Contract.ensures(
            this.totalItems === previousTotal + quantity,
            'মোট আইটেম সংখ্যা সঠিকভাবে বাড়েনি'
        );

        this.#checkInvariant();
    }

    removeItem(itemId, quantity = 1) {
        // পূর্বশর্ত
        Contract.requires(
            this.#items.has(itemId),
            `আইটেম কার্টে নেই: ${itemId}`
        );
        Contract.requires(
            Number.isInteger(quantity) && quantity >= 1,
            `পরিমাণ কমপক্ষে ১ হতে হবে: ${quantity}`
        );

        const item = this.#items.get(itemId);
        Contract.requires(
            quantity <= item.quantity,
            `সরানোর পরিমাণ (${quantity}) বর্তমান পরিমাণের (${item.quantity}) চেয়ে বেশি`
        );

        const previousTotal = this.totalItems;

        // মূল কাজ
        item.quantity -= quantity;
        if (item.quantity === 0) {
            this.#items.delete(itemId);
        }

        // পরশর্ত
        Contract.ensures(
            this.totalItems === previousTotal - quantity,
            'আইটেম সংখ্যা সঠিকভাবে কমেনি'
        );

        this.#checkInvariant();
    }

    get totalPrice() {
        let total = 0;
        for (const item of this.#items.values()) {
            total += item.price * item.quantity;
        }

        Contract.ensures(total >= 0, `মোট মূল্য ঋণাত্মক: ${total}`);
        return Math.round(total * 100) / 100;
    }

    get totalItems() {
        let count = 0;
        for (const item of this.#items.values()) {
            count += item.quantity;
        }
        return count;
    }

    get isEmpty() {
        return this.#items.size === 0;
    }

    // ক্লাস ইনভ্যারিয়েন্ট
    #checkInvariant() {
        for (const [id, item] of this.#items) {
            Contract.invariant(
                item.quantity >= 1,
                `ইনভ্যারিয়েন্ট লঙ্ঘন: '${id}'-এর পরিমাণ ${item.quantity}`
            );
            Contract.invariant(
                item.price > 0,
                `ইনভ্যারিয়েন্ট লঙ্ঘন: '${id}'-এর দাম ${item.price}`
            );
        }
    }
}

// ===== ব্যবহার =====
const cart = new ShoppingCart();

cart.addItem('BOOK-001', 'ক্লিন কোড বই', 450.00, 2);
cart.addItem('PEN-001', 'কলম সেট', 120.50, 3);

console.log(`মোট আইটেম: ${cart.totalItems}`);   // ৫
console.log(`মোট মূল্য: ৳${cart.totalPrice}`);   // ৳১২৬১.৫

cart.removeItem('PEN-001', 1);
console.log(`মোট আইটেম: ${cart.totalItems}`);   // ৪

// ❌ এগুলো এরর ছুড়বে:
// cart.addItem('', 'টেস্ট', 100);         // PreconditionError
// cart.addItem('X', 'টেস্ট', -50);        // PreconditionError
// cart.removeItem('INVALID');              // PreconditionError
```

---

## 📋 DbC চেকলিস্ট

আপনার কোডে DbC প্রয়োগ করার সময় এই চেকলিস্ট অনুসরণ করুন:

### পূর্বশর্ত (Preconditions):
- [ ] ফাংশনের সব প্যারামিটারের জন্য টাইপ যাচাই আছে?
- [ ] ব্যবসায়িক নিয়ম অনুযায়ী মান যাচাই আছে? (ধনাত্মক সংখ্যা, খালি নয়, ইত্যাদি)
- [ ] সীমার মধ্যে আছে কিনা যাচাই আছে? (অ্যারে ইন্ডেক্স, তারিখ পরিসর)
- [ ] null/undefined যাচাই আছে?
- [ ] ভুল ইনপুটে স্পষ্ট এরর মেসেজ আছে?

### পরশর্ত (Postconditions):
- [ ] ফলাফলের টাইপ সঠিক?
- [ ] ফলাফল প্রত্যাশিত সীমার মধ্যে?
- [ ] স্টেট পরিবর্তন সঠিকভাবে হয়েছে?
- [ ] কোনো পার্শ্বপ্রতিক্রিয়া প্রত্যাশিত?

### ক্লাস ইনভ্যারিয়েন্ট:
- [ ] অবজেক্টের অবস্থা সবসময় বৈধ?
- [ ] কনস্ট্রাক্টরে ইনভ্যারিয়েন্ট যাচাই আছে?
- [ ] প্রতিটি পাবলিক মেথডের পরে ইনভ্যারিয়েন্ট যাচাই আছে?
- [ ] প্রাইভেট ফিল্ড ব্যবহার করে এনক্যাপসুলেশন বজায় রাখা হয়েছে?

### সাধারণ:
- [ ] এক্সেপশন/এরর ক্লাস আলাদা (Precondition, Postcondition, Invariant)?
- [ ] এরর মেসেজে কী ভুল হয়েছে ও কী প্রত্যাশিত ছিল তা আছে?
- [ ] ডেভেলপমেন্ট ও প্রোডাকশনে assert-এর আচরণ ভিন্ন?

---

## 📊 তুলনা সারণি — DbC বনাম অন্যান্য পদ্ধতি

| বৈশিষ্ট্য | DbC | ডিফেন্সিভ প্রোগ্রামিং | টেস্ট-ড্রিভেন (TDD) | টাইপ সিস্টেম |
|-----------|-----|----------------------|---------------------|-------------|
| বাগ সনাক্তকরণ | রানটাইমে তাৎক্ষণিক | নীরবে হ্যান্ডেল | টেস্ট চালানোর সময় | কম্পাইল টাইমে |
| দায়িত্ব বণ্টন | ✅ স্পষ্ট | ❌ অস্পষ্ট | ⚠️ আংশিক | ⚠️ আংশিক |
| ডকুমেন্টেশন | ✅ কোডে স্বয়ংক্রিয় | ❌ আলাদা | ✅ টেস্টে | ✅ টাইপে |
| পারফরম্যান্স প্রভাব | ⚠️ রানটাইম খরচ | ⚠️ রানটাইম খরচ | ✅ নেই | ✅ নেই |
| প্রোডাকশনে ব্যবহার | ⚠️ বন্ধ করা যায় | ✅ সবসময় সক্রিয় | ✅ নেই | ✅ সবসময় |
| শেখার বক্ররেখা | মাঝারি | সহজ | মাঝারি | কঠিন |

---

## 🔑 মূল শিক্ষা সংক্ষেপ

```
┌───────────────────────────────────────────────────┐
│           DbC মনে রাখার সূত্র                     │
│                                                     │
│  ১. পূর্বশর্ত = কলারের দায়িত্ব                    │
│     "আমাকে সঠিক জিনিস দাও"                       │
│                                                     │
│  ২. পরশর্ত = সার্ভিসের দায়িত্ব                    │
│     "আমি সঠিক ফলাফল দেব"                          │
│                                                     │
│  ৩. ইনভ্যারিয়েন্ট = সবসময় সত্য                   │
│     "এই নিয়ম কখনো ভাঙবে না"                       │
│                                                     │
│  ৪. Fail Fast = দ্রুত ব্যর্থ হও                    │
│     "সমস্যা লুকিও না, চিৎকার করো"                  │
│                                                     │
│  ৫. স্পষ্ট দায়িত্ব = পরিষ্কার কোড                 │
│     "কে কী করবে তা সবাই জানে"                      │
└───────────────────────────────────────────────────┘
```

---

## 🔗 পরবর্তী বিষয়

DbC বোঝার পর, আপনি আরও ভালোভাবে বুঝতে পারবেন কিভাবে অবজেক্টগুলো একে অপরের সাথে
সম্পর্কিত এবং কিভাবে সেই সম্পর্কগুলো পরিচালনা করতে হয়:

➡️ [কম্পোজিশন বনাম ইনহেরিটেন্স (Composition vs Inheritance)](./composition-vs-inheritance.md)

---

> 💬 **"ভালো কন্ট্র্যাক্ট ভালো সফটওয়্যার তৈরি করে — ঠিক যেমন ভালো চুক্তি ভালো ব্যবসা তৈরি করে।"**
