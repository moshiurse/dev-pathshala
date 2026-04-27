# 🧹 DRY, KISS, YAGNI — ক্লিন কোডের তিন স্তম্ভ

> "সরলতা হলো চূড়ান্ত পরিশীলতা।" — লিওনার্দো দা ভিঞ্চি
>
> "ভালো কোড লেখার মানে হলো — কম কোড লেখা, সহজ কোড লেখা, এবং শুধু প্রয়োজনীয় কোড লেখা।"

---

## 📑 সূচিপত্র

| ক্রম | বিষয় |
|------|-------|
| ১ | [DRY — Don't Repeat Yourself](#১-dry--dont-repeat-yourself) |
| ২ | [KISS — Keep It Simple, Stupid](#২-kiss--keep-it-simple-stupid) |
| ৩ | [YAGNI — You Aren't Gonna Need It](#৩-yagni--you-arent-gonna-need-it) |
| ৪ | [তিনটি নীতির মধ্যে সম্পর্ক ও দ্বন্দ্ব](#৪-তিনটি-নীতির-মধ্যে-সম্পর্ক-ও-দ্বন্দ্ব) |
| ৫ | [সম্পূর্ণ প্র্যাক্টিক্যাল উদাহরণ — ব্লগ প্ল্যাটফর্ম](#৫-সম্পূর্ণ-প্র্যাক্টিক্যাল-উদাহরণ--ব্লগ-প্ল্যাটফর্ম) |
| ৬ | [চেকলিস্ট ও তুলনামূলক টেবিল](#৬-চেকলিস্ট-ও-তুলনামূলক-টেবিল) |

---

## ১. DRY — Don't Repeat Yourself

### 📖 সংজ্ঞা

**DRY (Don't Repeat Yourself)** নীতি বলে:

> "প্রতিটি জ্ঞানের (knowledge) একটি একক, দ্ব্যর্থহীন, এবং প্রামাণিক উপস্থাপনা থাকা উচিত সিস্টেমের মধ্যে।"
>
> — Andrew Hunt ও David Thomas, "The Pragmatic Programmer" (১৯৯৯)

DRY শুধু "কোড কপি-পেস্ট করো না" এতটুকু না। এটি আরও গভীর — এটি বলে যে **একটি তথ্য বা লজিক শুধুমাত্র একটি জায়গায় থাকবে**। যদি সেই তথ্য পরিবর্তন করতে হয়, তাহলে শুধু একটি জায়গায় পরিবর্তন করলেই হবে।

```
┌─────────────────────────────────────────────────────┐
│                  DRY নীতি                            │
│                                                     │
│   "একটি তথ্য = একটি জায়গা"                          │
│                                                     │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│   │ মডিউল A  │    │ মডিউল B  │    │ মডিউল C  │      │
│   │          │    │          │    │          │      │
│   │  ব্যবহার ──────▶ একক    ◀────── ব্যবহার │      │
│   │  করে     │    │  উৎস    │    │  করে     │      │
│   └──────────┘    └──────────┘    └──────────┘      │
│                                                     │
│   পরিবর্তন দরকার? শুধু "একক উৎস" পরিবর্তন করো!    │
└─────────────────────────────────────────────────────┘
```

---

### 🏠 বাস্তব জীবনের উদাহরণ

**📱 ফোন নম্বর পরিবর্তন:**

ধরো, তোমার ফোন নম্বর পরিবর্তন হয়েছে।

- **WET পদ্ধতি (খারাপ):** তুমি তোমার ফোন নম্বর ১০টা জায়গায় আলাদা আলাদা ভাবে লিখে রেখেছো — ব্যাংকে, অফিসে, বন্ধুদের কাছে, ডাক্তারের কাছে ইত্যাদি। এখন ১০টা জায়গায় গিয়ে আপডেট করতে হবে। যদি একটা মিস হয়, সমস্যা!
- **DRY পদ্ধতি (ভালো):** তুমি একটা Google Contacts-এ নম্বর রাখো, সবাই সেখান থেকে দেখে। পরিবর্তন করলে সবাই আপডেটেড নম্বর পায়।

**🍳 রান্নার রেসিপি:**

- **WET:** প্রতিটি ডিশের রেসিপিতে "পেঁয়াজ কাটার নিয়ম" আলাদা করে লেখা।
- **DRY:** একটি "পেঁয়াজ কাটার নিয়ম" পেজ বানিয়ে, প্রতিটি ডিশ থেকে সেই পেজে রেফারেন্স দেওয়া।

---

### 🔍 DRY ভঙ্গের ধরন

DRY ভঙ্গ শুধু কোড কপি-পেস্ট না। তিনটি প্রধান ধরন আছে:

```
┌──────────────────────────────────────────────────────────────────┐
│              DRY ভঙ্গের তিনটি স্তর                               │
│                                                                  │
│  স্তর ১: কোড ডুপ্লিকেশন                                         │
│  ─────────────────────                                           │
│  • হুবহু একই কোড একাধিক জায়গায়                                   │
│  • সবচেয়ে সহজে চিহ্নিত করা যায়                                  │
│  • IDE "Extract Method" দিয়ে সমাধান                              │
│                                                                  │
│  স্তর ২: লজিক ডুপ্লিকেশন                                        │
│  ─────────────────────                                           │
│  • ভিন্ন কোড, কিন্তু একই কাজ করে                                 │
│  • চিহ্নিত করা কঠিন — দেখতে আলাদা মনে হয়                        │
│  • গভীর বিশ্লেষণ দরকার                                           │
│                                                                  │
│  স্তর ৩: নলেজ ডুপ্লিকেশন                                        │
│  ─────────────────────                                           │
│  • একই তথ্য কোডে এবং ডকুমেন্টেশনে                               │
│  • বা একই বিজনেস রুল দুই জায়গায়                                  │
│  • সবচেয়ে বিপজ্জনক — আউট অফ সিংক হলে বাগ                       │
└──────────────────────────────────────────────────────────────────┘
```

---

#### স্তর ১: কোড ডুপ্লিকেশন (সবচেয়ে সাধারণ)

হুবহু একই কোড একাধিক জায়গায় কপি-পেস্ট করা।

**❌ খারাপ উদাহরণ (PHP) — কোড ডুপ্লিকেশন:**

```php
<?php
// ❌ খারাপ: একই ভ্যালিডেশন লজিক দুই জায়গায়

class UserController
{
    public function register(Request $request)
    {
        // ইমেইল ভ্যালিডেশন — এখানে লেখা
        $email = $request->input('email');
        if (empty($email)) {
            return response()->json(['error' => 'ইমেইল দিতে হবে'], 400);
        }
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return response()->json(['error' => 'ইমেইল সঠিক নয়'], 400);
        }
        if (strlen($email) > 255) {
            return response()->json(['error' => 'ইমেইল অনেক বড়'], 400);
        }

        // ব্যবহারকারী তৈরি করো...
    }

    public function updateProfile(Request $request)
    {
        // ইমেইল ভ্যালিডেশন — আবার একই কোড!
        $email = $request->input('email');
        if (empty($email)) {
            return response()->json(['error' => 'ইমেইল দিতে হবে'], 400);
        }
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return response()->json(['error' => 'ইমেইল সঠিক নয়'], 400);
        }
        if (strlen($email) > 255) {
            return response()->json(['error' => 'ইমেইল অনেক বড়'], 400);
        }

        // প্রোফাইল আপডেট করো...
    }
}
```

**✅ ভালো উদাহরণ (PHP) — DRY প্রয়োগ:**

```php
<?php
// ✅ ভালো: ইমেইল ভ্যালিডেশন একটি জায়গায়

class EmailValidator
{
    /**
     * ইমেইল ভ্যালিডেট করো — একক উৎস
     */
    public static function validate(string $email): ?string
    {
        if (empty($email)) {
            return 'ইমেইল দিতে হবে';
        }
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return 'ইমেইল সঠিক নয়';
        }
        if (strlen($email) > 255) {
            return 'ইমেইল অনেক বড়';
        }

        return null; // কোনো ত্রুটি নেই
    }
}

class UserController
{
    public function register(Request $request)
    {
        // শেয়ার্ড ভ্যালিডেটর ব্যবহার
        $error = EmailValidator::validate($request->input('email'));
        if ($error) {
            return response()->json(['error' => $error], 400);
        }

        // ব্যবহারকারী তৈরি করো...
    }

    public function updateProfile(Request $request)
    {
        // একই ভ্যালিডেটর পুনরায় ব্যবহার
        $error = EmailValidator::validate($request->input('email'));
        if ($error) {
            return response()->json(['error' => $error], 400);
        }

        // প্রোফাইল আপডেট করো...
    }
}
```

---

#### স্তর ২: লজিক ডুপ্লিকেশন (ভিন্ন কোড, একই লজিক)

দেখতে আলাদা কোড, কিন্তু আসলে একই কাজ করছে। এটি চিহ্নিত করা বেশি কঠিন।

**❌ খারাপ উদাহরণ (JavaScript) — লজিক ডুপ্লিকেশন:**

```javascript
// ❌ খারাপ: ভিন্ন কোড, কিন্তু একই "বয়স যাচাই" লজিক

// প্রথম জায়গা — ইউজার রেজিস্ট্রেশনে
function canRegister(user) {
    const birthDate = new Date(user.dateOfBirth);
    const today = new Date();
    // বছরের পার্থক্য হিসাব করে বয়স বের করা
    let age = today.getFullYear() - birthDate.getFullYear();
    const monthDiff = today.getMonth() - birthDate.getMonth();
    if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
        age--;
    }
    return age >= 18;
}

// দ্বিতীয় জায়গা — মদ কেনার অনুমতিতে
function canBuyAlcohol(customer) {
    // মিলিসেকেন্ড থেকে বয়স বের করা — ভিন্ন পদ্ধতি, একই লজিক!
    const ageInMs = Date.now() - new Date(customer.dob).getTime();
    const ageInYears = Math.floor(ageInMs / (365.25 * 24 * 60 * 60 * 1000));
    return ageInYears >= 18;
}
```

**✅ ভালো উদাহরণ (JavaScript) — DRY প্রয়োগ:**

```javascript
// ✅ ভালো: বয়স গণনা একটি ফাংশনে

/**
 * জন্মতারিখ থেকে বয়স গণনা — একক উৎস
 */
function calculateAge(dateOfBirth) {
    const birthDate = new Date(dateOfBirth);
    const today = new Date();
    let age = today.getFullYear() - birthDate.getFullYear();
    const monthDiff = today.getMonth() - birthDate.getMonth();
    if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
        age--;
    }
    return age;
}

/**
 * নির্দিষ্ট বয়সসীমা পার হয়েছে কিনা যাচাই
 */
function isAboveAge(dateOfBirth, minimumAge) {
    return calculateAge(dateOfBirth) >= minimumAge;
}

// এখন সব জায়গায় একই ফাংশন ব্যবহার
function canRegister(user) {
    return isAboveAge(user.dateOfBirth, 18);
}

function canBuyAlcohol(customer) {
    return isAboveAge(customer.dob, 18);
}
```

---

#### স্তর ৩: নলেজ ডুপ্লিকেশন (ডকুমেন্টেশন ও কোডে একই তথ্য)

**❌ খারাপ উদাহরণ (PHP) — নলেজ ডুপ্লিকেশন:**

```php
<?php
// ❌ খারাপ: ট্যাক্স রেট কোডে এবং ডকুমেন্টেশনেও আছে

/**
 * ট্যাক্স হিসাব করে।
 * বর্তমান ট্যাক্স রেট ১৫%।    <--- ডকুমেন্টেশনে রেট লেখা
 */
function calculateTax(float $amount): float
{
    return $amount * 0.15;        // <--- কোডেও রেট আছে
}

// সমস্যা: রেট ২০% হলে দুই জায়গায় আপডেট করতে হবে।
// ডকুমেন্টেশন আপডেট ভুলে গেলে — ভুল তথ্য থাকবে!
```

**✅ ভালো উদাহরণ (PHP) — DRY প্রয়োগ:**

```php
<?php
// ✅ ভালো: ট্যাক্স রেট একটি কনস্ট্যান্টে, ডক এটি রেফার করে

class TaxConfig
{
    // একক উৎস — শুধু এখানে পরিবর্তন করলেই হবে
    public const RATE = 0.15;
}

/**
 * ট্যাক্স হিসাব করে।
 * TaxConfig::RATE ব্যবহার করে বর্তমান রেট প্রয়োগ করে।
 */
function calculateTax(float $amount): float
{
    return $amount * TaxConfig::RATE;
}
```

---

### 💧 WET কোড — Write Everything Twice

**WET** হলো DRY-এর বিপরীত। এর মানে "Write Everything Twice" বা "We Enjoy Typing"।

WET কোডের সমস্যা:

```
┌────────────────────────────────────────────────────────────┐
│                WET কোডের বিপদ                               │
│                                                            │
│  ১. বাগ ফিক্স একাধিক জায়গায় করতে হয়                       │
│     └── একটি মিস করলে → নতুন বাগ!                          │
│                                                            │
│  ২. পরিবর্তন ব্যয়বহুল                                     │
│     └── ১০ জায়গায় একই পরিবর্তন → সময় নষ্ট                │
│                                                            │
│  ৩. অসামঞ্জস্যতা (Inconsistency)                           │
│     └── একই লজিকের ভিন্ন ভার্সন → অদ্ভুত বাগ              │
│                                                            │
│  ৪. টেস্ট করা কঠিন                                         │
│     └── একই জিনিস বারবার টেস্ট করতে হয়                    │
│                                                            │
│  ৫. কোড বোঝা কঠিন                                          │
│     └── "এই ফাংশনটা কি ওইটার মতো?"                         │
└────────────────────────────────────────────────────────────┘
```

---

### ⚠️ DRY অতিরিক্ত প্রয়োগ — কখন ডুপ্লিকেশন ভালো

DRY অতিরিক্ত প্রয়োগ করলে কোড আরও জটিল হতে পারে! **Rule of Three** মনে রাখো:

> "প্রথমবার লেখো, দ্বিতীয়বার অস্বস্তি অনুভব করো, তৃতীয়বার রিফ্যাক্টর করো।"

```
┌────────────────────────────────────────────────────────────┐
│              Rule of Three সিদ্ধান্ত চার্ট                  │
│                                                            │
│  কোড কি ডুপ্লিকেট?                                        │
│       │                                                    │
│       ▼                                                    │
│  ┌────────────┐                                            │
│  │ প্রথমবার?  │──── হ্যাঁ ──▶ রাখো, চিন্তা নেই           │
│  └────────────┘                                            │
│       │ না                                                  │
│       ▼                                                    │
│  ┌────────────┐                                            │
│  │ দ্বিতীয়বার?│──── হ্যাঁ ──▶ নোট করো, এখনো রাখো         │
│  └────────────┘                                            │
│       │ না                                                  │
│       ▼                                                    │
│  ┌────────────┐                                            │
│  │ তৃতীয়বার? │──── হ্যাঁ ──▶ 🔧 রিফ্যাক্টর করো!          │
│  └────────────┘                                            │
└────────────────────────────────────────────────────────────┘
```

**কখন ডুপ্লিকেশন গ্রহণযোগ্য:**

| পরিস্থিতি | কারণ |
|-----------|------|
| ভিন্ন বাউন্ডেড কনটেক্সট | অর্ডার সিস্টেম ও শিপিং সিস্টেমে `Address` ক্লাস আলাদা রাখা ভালো |
| প্রোটোটাইপ/MVP | দ্রুত শিপ করা বেশি গুরুত্বপূর্ণ |
| ভিন্ন পরিবর্তনের হার | একটি প্রায়ই বদলায়, অন্যটি স্থির |
| DRY করলে জটিলতা অনেক বাড়ে | KISS নীতির সাথে দ্বন্দ্ব হয় |

---

### 🛒 কমপ্লেক্স উদাহরণ: ই-কমার্স ডিসকাউন্ট সিস্টেম

একটি ই-কমার্স সাইটে বিভিন্ন জায়গায় ডিসকাউন্ট ভ্যালিডেশন লজিক পুনরাবৃত্তি হচ্ছে।

**❌ খারাপ উদাহরণ (PHP) — WET ডিসকাউন্ট সিস্টেম:**

```php
<?php
// ❌ খারাপ: ডিসকাউন্ট লজিক তিন জায়গায় ছড়িয়ে আছে

class CartController
{
    public function applyDiscount(Request $request)
    {
        $code = $request->input('discount_code');
        $cart = $this->getCart($request->user());

        // ডিসকাউন্ট ভ্যালিডেশন — প্রথম কপি
        $discount = Discount::where('code', $code)->first();
        if (!$discount) {
            return response()->json(['error' => 'কুপন পাওয়া যায়নি'], 404);
        }
        if ($discount->expires_at < now()) {
            return response()->json(['error' => 'কুপনের মেয়াদ শেষ'], 400);
        }
        if ($discount->usage_count >= $discount->max_usage) {
            return response()->json(['error' => 'কুপন ব্যবহারের সীমা শেষ'], 400);
        }
        if ($cart->total < $discount->minimum_order) {
            return response()->json([
                'error' => "ন্যূনতম অর্ডার {$discount->minimum_order} টাকা হতে হবে"
            ], 400);
        }

        // ডিসকাউন্ট প্রয়োগ
        $discountAmount = $cart->total * ($discount->percentage / 100);
        $cart->discount = min($discountAmount, $discount->max_discount);
        $cart->save();
    }
}

class CheckoutController
{
    public function processOrder(Request $request)
    {
        $cart = $this->getCart($request->user());

        if ($cart->discount_code) {
            // ডিসকাউন্ট ভ্যালিডেশন — দ্বিতীয় কপি (চেকআউটের সময় আবার যাচাই)
            $discount = Discount::where('code', $cart->discount_code)->first();
            if (!$discount) {
                return response()->json(['error' => 'কুপন পাওয়া যায়নি'], 404);
            }
            if ($discount->expires_at < now()) {
                return response()->json(['error' => 'কুপনের মেয়াদ শেষ'], 400);
            }
            if ($discount->usage_count >= $discount->max_usage) {
                return response()->json(['error' => 'কুপন ব্যবহারের সীমা শেষ'], 400);
            }
            if ($cart->total < $discount->minimum_order) {
                return response()->json([
                    'error' => "ন্যূনতম অর্ডার {$discount->minimum_order} টাকা হতে হবে"
                ], 400);
            }

            // ডিসকাউন্ট হিসাব — আবার একই কোড!
            $discountAmount = $cart->total * ($discount->percentage / 100);
            $cart->discount = min($discountAmount, $discount->max_discount);
        }

        // অর্ডার প্রসেস করো...
    }
}

class ApiController
{
    public function validateCoupon(Request $request)
    {
        $code = $request->input('code');
        $orderTotal = $request->input('total');

        // ডিসকাউন্ট ভ্যালিডেশন — তৃতীয় কপি (API-এর জন্য)
        $discount = Discount::where('code', $code)->first();
        if (!$discount) {
            return response()->json(['valid' => false, 'reason' => 'কুপন পাওয়া যায়নি']);
        }
        if ($discount->expires_at < now()) {
            return response()->json(['valid' => false, 'reason' => 'মেয়াদ শেষ']);
        }
        if ($discount->usage_count >= $discount->max_usage) {
            return response()->json(['valid' => false, 'reason' => 'সীমা শেষ']);
        }
        if ($orderTotal < $discount->minimum_order) {
            return response()->json(['valid' => false, 'reason' => 'ন্যূনতম অর্ডার পূরণ হয়নি']);
        }

        // ডিসকাউন্ট হিসাব — আবার!
        $discountAmount = $orderTotal * ($discount->percentage / 100);
        $finalDiscount = min($discountAmount, $discount->max_discount);

        return response()->json(['valid' => true, 'discount' => $finalDiscount]);
    }
}
```

**✅ ভালো উদাহরণ (PHP) — DRY ডিসকাউন্ট সিস্টেম:**

```php
<?php
// ✅ ভালো: একটি DiscountService সব লজিক ধারণ করে

class DiscountValidationResult
{
    public bool $isValid;
    public ?string $error;
    public ?float $discountAmount;

    public function __construct(bool $isValid, ?string $error = null, ?float $discountAmount = null)
    {
        $this->isValid = $isValid;
        $this->error = $error;
        $this->discountAmount = $discountAmount;
    }

    public static function success(float $amount): self
    {
        return new self(true, null, $amount);
    }

    public static function failure(string $error): self
    {
        return new self(false, $error);
    }
}

class DiscountService
{
    /**
     * কুপন ভ্যালিডেট ও ডিসকাউন্ট হিসাব করো — একক উৎস
     */
    public function validateAndCalculate(string $code, float $orderTotal): DiscountValidationResult
    {
        $discount = Discount::where('code', $code)->first();

        if (!$discount) {
            return DiscountValidationResult::failure('কুপন পাওয়া যায়নি');
        }

        if ($discount->expires_at < now()) {
            return DiscountValidationResult::failure('কুপনের মেয়াদ শেষ');
        }

        if ($discount->usage_count >= $discount->max_usage) {
            return DiscountValidationResult::failure('কুপন ব্যবহারের সীমা শেষ');
        }

        if ($orderTotal < $discount->minimum_order) {
            return DiscountValidationResult::failure(
                "ন্যূনতম অর্ডার {$discount->minimum_order} টাকা হতে হবে"
            );
        }

        // ডিসকাউন্ট হিসাব — শুধু এখানে
        $discountAmount = $orderTotal * ($discount->percentage / 100);
        $finalAmount = min($discountAmount, $discount->max_discount);

        return DiscountValidationResult::success($finalAmount);
    }
}

// এখন সব কন্ট্রোলার একই সার্ভিস ব্যবহার করে
class CartController
{
    public function __construct(private DiscountService $discountService) {}

    public function applyDiscount(Request $request)
    {
        $cart = $this->getCart($request->user());
        $result = $this->discountService->validateAndCalculate(
            $request->input('discount_code'),
            $cart->total
        );

        if (!$result->isValid) {
            return response()->json(['error' => $result->error], 400);
        }

        $cart->discount = $result->discountAmount;
        $cart->save();
    }
}

class CheckoutController
{
    public function __construct(private DiscountService $discountService) {}

    public function processOrder(Request $request)
    {
        $cart = $this->getCart($request->user());

        if ($cart->discount_code) {
            $result = $this->discountService->validateAndCalculate(
                $cart->discount_code,
                $cart->total
            );

            if (!$result->isValid) {
                return response()->json(['error' => $result->error], 400);
            }
            $cart->discount = $result->discountAmount;
        }

        // অর্ডার প্রসেস করো...
    }
}
```

---

## ২. KISS — Keep It Simple, Stupid

### 📖 সংজ্ঞা

**KISS (Keep It Simple, Stupid)** নীতি বলে:

> "বেশিরভাগ সিস্টেম সবচেয়ে ভালো কাজ করে যদি সেগুলো সরল রাখা হয়, জটিল না করা হয়।
> তাই, ডিজাইনের মূল লক্ষ্য হওয়া উচিত সরলতা, এবং অপ্রয়োজনীয় জটিলতা এড়িয়ে চলা।"
>
> — Kelly Johnson, Lockheed Martin (১৯৬০ এর দশক)

KISS নীতির মূল কথা: **সবচেয়ে সরল সমাধান যা কাজ করে, সেটাই সেরা সমাধান।**

```
┌───────────────────────────────────────────────────────────┐
│                    KISS নীতি                               │
│                                                           │
│   সমস্যা ──▶ সমাধান খোঁজো ──▶ সবচেয়ে সরলটা বেছে নাও   │
│                                                           │
│   ┌───────────────────────────────────────────┐           │
│   │  "ডিবাগিং কোড লেখার চেয়ে দ্বিগুণ কঠিন। │           │
│   │  তাই, তুমি যদি কোড লেখার সময় সর্বোচ্চ   │           │
│   │  চালাকি দেখাও, তাহলে সেটা ডিবাগ করার    │           │
│   │  মতো বুদ্ধি তোমার নেই।"                   │           │
│   │                   — Brian Kernighan        │           │
│   └───────────────────────────────────────────┘           │
│                                                           │
│   সরলতা ≠ সহজ                                             │
│   সরলতা = জটিলতার অনুপস্থিতি                              │
└───────────────────────────────────────────────────────────┘
```

---

### 🏠 বাস্তব জীবনের উদাহরণ

**🚪 দরজা খোলা:**

- **KISS ভঙ্গ:** দরজায় ফেস রিকগনিশন + ফিঙ্গারপ্রিন্ট + ভয়েস কমান্ড + OTP — শুধু ঘরে ঢুকতে!
- **KISS মেনে চলা:** একটি চাবি দিয়ে দরজা খোলো। কাজ হচ্ছে, সরল।

**🍳 ডিম ভাজা:**

- **KISS ভঙ্গ:** sous vide মেশিনে সঠিক তাপমাত্রায় ৪৫ মিনিট রান্না, তারপর blow torch দিয়ে ফিনিশ।
- **KISS মেনে চলা:** ফ্রাইপ্যানে তেল গরম করো, ডিম ভাঙো, ভাজো। ব্যস!

---

### 🏗️ Over-engineering উদাহরণ

**❌ খারাপ উদাহরণ (JavaScript) — Over-engineered Authentication:**

```javascript
// ❌ খারাপ: ইউজার অথেন্টিকেশনের জন্য অতিরিক্ত জটিল আর্কিটেকচার

// পদক্ষেপ ১: অ্যাবস্ট্রাক্ট ফ্যাক্টরি
class AbstractAuthenticationStrategyFactory {
    createStrategy() {
        throw new Error('সাবক্লাসে ইমপ্লিমেন্ট করতে হবে');
    }
}

// পদক্ষেপ ২: কনক্রিট ফ্যাক্টরি
class ConcretePasswordAuthenticationStrategyFactory extends AbstractAuthenticationStrategyFactory {
    createStrategy() {
        return new PasswordAuthenticationStrategy(
            new PasswordHashingService(
                new BCryptAdapter(
                    new CryptoConfigurationProvider()
                )
            )
        );
    }
}

// পদক্ষেপ ৩: স্ট্র্যাটেজি ইন্টারফেস
class AuthenticationStrategy {
    authenticate(credentials) {
        throw new Error('সাবক্লাসে ইমপ্লিমেন্ট করতে হবে');
    }
}

// পদক্ষেপ ৪: কনক্রিট স্ট্র্যাটেজি
class PasswordAuthenticationStrategy extends AuthenticationStrategy {
    constructor(hashingService) {
        super();
        this.hashingService = hashingService;
    }

    authenticate(credentials) {
        return this.hashingService.verify(
            credentials.password,
            credentials.hashedPassword
        );
    }
}

// পদক্ষেপ ৫: হ্যাশিং সার্ভিস
class PasswordHashingService {
    constructor(adapter) {
        this.adapter = adapter;
    }

    verify(plain, hashed) {
        return this.adapter.compare(plain, hashed);
    }
}

// পদক্ষেপ ৬: অ্যাডাপ্টার
class BCryptAdapter {
    constructor(configProvider) {
        this.config = configProvider.getConfig();
    }

    compare(plain, hashed) {
        // অবশেষে bcrypt কল হচ্ছে...
        return bcrypt.compareSync(plain, hashed);
    }
}

// পদক্ষেপ ৭: কনফিগ প্রোভাইডার
class CryptoConfigurationProvider {
    getConfig() {
        return { rounds: 10 };
    }
}

// ব্যবহার — ৭টি ক্লাস পার হয়ে শুধু পাসওয়ার্ড চেক!
const factory = new ConcretePasswordAuthenticationStrategyFactory();
const strategy = factory.createStrategy();
const isValid = strategy.authenticate({
    password: 'user123',
    hashedPassword: '$2b$10$...'
});
```

**✅ ভালো উদাহরণ (JavaScript) — KISS প্রয়োগ:**

```javascript
// ✅ ভালো: সরল AuthService — যা দরকার শুধু তাই

const bcrypt = require('bcrypt');

class AuthService {
    // পাসওয়ার্ড হ্যাশ করো
    static async hashPassword(password) {
        return bcrypt.hash(password, 10);
    }

    // পাসওয়ার্ড যাচাই করো
    static async verifyPassword(password, hashedPassword) {
        return bcrypt.compare(password, hashedPassword);
    }

    // লগইন করো
    static async login(email, password) {
        const user = await User.findByEmail(email);
        if (!user) {
            return { success: false, error: 'ব্যবহারকারী পাওয়া যায়নি' };
        }

        const isValid = await this.verifyPassword(password, user.hashedPassword);
        if (!isValid) {
            return { success: false, error: 'ভুল পাসওয়ার্ড' };
        }

        const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET);
        return { success: true, token };
    }
}

// ব্যবহার — পরিষ্কার এবং সরল
const result = await AuthService.login('user@example.com', 'password123');
```

---

### 🔬 সরল সমাধান vs জটিল সমাধান

**❌ খারাপ উদাহরণ (PHP) — জটিল অ্যারে ফিল্টারিং:**

```php
<?php
// ❌ খারাপ: "চালাক" কিন্তু দুর্বোধ্য কোড

function getActiveAdminEmails(array $users): array
{
    // একটি লাইনে সব করার চেষ্টা — বোঝা কঠিন
    return array_values(
        array_map(
            fn($u) => $u['email'],
            array_filter(
                $users,
                fn($u) => ($u['role'] ?? '') === 'admin'
                    && ($u['status'] ?? '') === 'active'
                    && isset($u['email'])
                    && filter_var($u['email'], FILTER_VALIDATE_EMAIL)
            )
        )
    );
}
```

**✅ ভালো উদাহরণ (PHP) — KISS প্রয়োগ:**

```php
<?php
// ✅ ভালো: ধাপে ধাপে, পড়তে সহজ

function getActiveAdminEmails(array $users): array
{
    $emails = [];

    foreach ($users as $user) {
        // শর্তগুলো আলাদা করে পড়া যাচ্ছে
        $isAdmin = ($user['role'] ?? '') === 'admin';
        $isActive = ($user['status'] ?? '') === 'active';
        $hasValidEmail = isset($user['email'])
            && filter_var($user['email'], FILTER_VALIDATE_EMAIL);

        if ($isAdmin && $isActive && $hasValidEmail) {
            $emails[] = $user['email'];
        }
    }

    return $emails;
}
```

---

### 🐢 Premature Optimization vs KISS

> "অকাল অপ্টিমাইজেশন হলো সমস্ত মন্দের মূল।" — Donald Knuth

**❌ খারাপ উদাহরণ (JavaScript) — Premature Optimization:**

```javascript
// ❌ খারাপ: অপ্রয়োজনীয় ক্যাশিং ও অপ্টিমাইজেশন

class UserNameFormatter {
    constructor() {
        // ক্যাশ — আসলে প্রয়োজন নেই (১০০ জন ইউজার!)
        this._cache = new Map();
        this._cacheHits = 0;
        this._cacheMisses = 0;
    }

    formatName(user) {
        const cacheKey = `${user.id}_${user.firstName}_${user.lastName}`;

        if (this._cache.has(cacheKey)) {
            this._cacheHits++;
            return this._cache.get(cacheKey);
        }

        this._cacheMisses++;

        // বিট ম্যানিপুলেশন দিয়ে স্ট্রিং — কেন?!
        const result = `${user.firstName} ${user.lastName}`.trim();

        this._cache.set(cacheKey, result);

        // ক্যাশ সাইজ সীমিত রাখা (LRU) — অপ্রয়োজনীয়!
        if (this._cache.size > 1000) {
            const firstKey = this._cache.keys().next().value;
            this._cache.delete(firstKey);
        }

        return result;
    }

    getCacheStats() {
        return {
            hits: this._cacheHits,
            misses: this._cacheMisses,
            ratio: this._cacheHits / (this._cacheHits + this._cacheMisses)
        };
    }
}
```

**✅ ভালো উদাহরণ (JavaScript) — KISS প্রয়োগ:**

```javascript
// ✅ ভালো: সরল, পরিষ্কার — যা দরকার শুধু তাই

function formatUserName(user) {
    return `${user.firstName} ${user.lastName}`.trim();
}

// ব্যস! ১০০ জন ইউজারের জন্য ক্যাশিংয়ের দরকার নেই।
// যদি ভবিষ্যতে পার্ফরম্যান্স সমস্যা হয়, তখন অপ্টিমাইজ করবে।
```

---

## ৩. YAGNI — You Aren't Gonna Need It

### 📖 সংজ্ঞা

**YAGNI (You Aren't Gonna Need It)** নীতি বলে:

> "কোনো ফিচার ততক্ষণ যোগ করো না, যতক্ষণ না সত্যিই দরকার হচ্ছে।"
>
> — Ron Jeffries, Extreme Programming (XP) এর অন্যতম প্রতিষ্ঠাতা

YAGNI এর মূল কথা: **"হয়তো ভবিষ্যতে লাগবে" — এই চিন্তা করে কোড লিখো না। যখন লাগবে, তখন লিখো।**

```
┌────────────────────────────────────────────────────────────┐
│                     YAGNI নীতি                              │
│                                                            │
│   ডেভেলপার: "ভবিষ্যতে XML সাপোর্টও লাগতে পারে..."        │
│                                                            │
│   YAGNI:    "এখন কি XML দরকার?"                            │
│                                                            │
│   ডেভেলপার: "না, কিন্তু..."                                │
│                                                            │
│   YAGNI:    "তাহলে লিখো না! যখন দরকার হবে, তখন লিখো।"   │
│                                                            │
│   ┌──────────────────────────────────────────────┐         │
│   │  YAGNI = বর্তমান প্রয়োজন মেটাও              │         │
│   │                                              │         │
│   │  ❌ "হয়তো লাগবে" কোড                         │         │
│   │  ❌ "কেউ চাইতে পারে" ফিচার                   │         │
│   │  ❌ "ভবিষ্যতে কাজে আসবে" অ্যাবস্ট্রাকশন      │         │
│   │                                              │         │
│   │  ✅ শুধু এখন যা দরকার                         │         │
│   │  ✅ সবচেয়ে সহজ যা কাজ করে                    │         │
│   └──────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────┘
```

---

### 🏠 বাস্তব জীবনের উদাহরণ

**🏠 বাড়ি বানানো:**

- **YAGNI ভঙ্গ:** তুমি ২ জনের পরিবারের জন্য বাড়ি বানাচ্ছো, কিন্তু "হয়তো ভবিষ্যতে ১০ জন থাকবে" ভেবে ১০টা বেডরুম বানিয়ে ফেলো। বাড়তি খরচ, বাড়তি রক্ষণাবেক্ষণ — আর ১০ জন কখনোই আসে না।
- **YAGNI মেনে চলা:** ২টা বেডরুম বানাও। পরে দরকার হলে এক্সটেনশন করো।

**🧳 ভ্রমণ:**

- **YAGNI ভঙ্গ:** ৩ দিনের ট্রিপে ৩টা বড় ব্যাগ — "যদি বৃষ্টি হয়, যদি পাহাড়ে যাই, যদি সমুদ্রে যাই..."
- **YAGNI মেনে চলা:** আবহাওয়া দেখে যতটুকু লাগবে ঠিক ততটুকু নাও।

---

### 🔮 Speculative Generality

**Speculative Generality** হলো এমন কোড লেখা যা "হয়তো ভবিষ্যতে কাজে আসবে" চিন্তা থেকে আসে:

```
┌────────────────────────────────────────────────────────────┐
│          Speculative Generality এর লক্ষণ                    │
│                                                            │
│  🚩 অব্যবহৃত প্যারামিটার                                   │
│     function doSomething(a, b, c, futureParam) { ... }     │
│                                                            │
│  🚩 খালি ইন্টারফেস/ক্লাস                                   │
│     class FutureFeatureHandler { /* পরে লিখবো */ }         │
│                                                            │
│  🚩 অতিরিক্ত অ্যাবস্ট্রাকশন                                │
│     সহজ কাজে ৫-৬ লেয়ারের আর্কিটেকচার                      │
│                                                            │
│  🚩 কনফিগারেশনের অতিরিক্ত অপশন                            │
│     ১০০টা সেটিংস, মাত্র ৩টা ব্যবহৃত                       │
│                                                            │
│  🚩 "এক্সটেনসিবিলিটি পয়েন্ট"                               │
│     প্লাগইন সিস্টেম — কিন্তু কোনো প্লাগইন নেই             │
└────────────────────────────────────────────────────────────┘
```

---

### 💰 ভবিষ্যতের জন্য কোড লেখার সমস্যা

```
┌─────────────────────────────────────────────────────────────┐
│        "হয়তো লাগবে" কোডের প্রকৃত খরচ                       │
│                                                             │
│   লেখার সময়:     ████████████  (৪ ঘণ্টা)                   │
│   টেস্ট করা:      ██████████    (৩ ঘণ্টা)                   │
│   ডকুমেন্টেশন:   ██████        (২ ঘণ্টা)                   │
│   রক্ষণাবেক্ষণ:   █████████████████████████  (চলমান!)       │
│   রিফ্যাক্টরিং:  ████████████  (৪ ঘণ্টা, বারবার)          │
│   বাগ ফিক্স:      ██████████    (অব্যবহৃত কোডেও বাগ হয়!) │
│                                                             │
│   মোট খরচ: অনেক!                                            │
│   ব্যবহৃত: ০%                                                │
│                                                             │
│   ❗ গবেষণায় দেখা গেছে: "হয়তো লাগবে" ফিচারের              │
│      মাত্র ~১০% আসলে কখনো ব্যবহৃত হয়!                     │
└─────────────────────────────────────────────────────────────┘
```

---

### 🗃️ কমপ্লেক্স উদাহরণ: CMS সিস্টেম — অপ্রয়োজনীয় ডেটাবেস সাপোর্ট

একটি CMS সিস্টেম যেখানে শুধু MySQL দরকার, কিন্তু ডেভেলপার ১০টি ডেটাবেস সাপোর্ট করার জন্য কোড লিখেছে।

**❌ খারাপ উদাহরণ (PHP) — YAGNI ভঙ্গ:**

```php
<?php
// ❌ খারাপ: শুধু MySQL লাগবে, কিন্তু ১০টি ডেটাবেসের জন্য কোড!

interface DatabaseDriverInterface
{
    public function connect(array $config): void;
    public function query(string $sql, array $params = []): array;
    public function insert(string $table, array $data): int;
    public function update(string $table, array $data, array $where): int;
    public function delete(string $table, array $where): int;
    public function beginTransaction(): void;
    public function commit(): void;
    public function rollback(): void;
}

// MySQL ড্রাইভার — এটাই আসলে দরকার
class MySQLDriver implements DatabaseDriverInterface
{
    private $pdo;

    public function connect(array $config): void
    {
        $dsn = "mysql:host={$config['host']};dbname={$config['database']}";
        $this->pdo = new PDO($dsn, $config['username'], $config['password']);
    }

    public function query(string $sql, array $params = []): array
    {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    // ... বাকি মেথডগুলো
    public function insert(string $table, array $data): int { /* ... */ return 0; }
    public function update(string $table, array $data, array $where): int { /* ... */ return 0; }
    public function delete(string $table, array $where): int { /* ... */ return 0; }
    public function beginTransaction(): void { /* ... */ }
    public function commit(): void { /* ... */ }
    public function rollback(): void { /* ... */ }
}

// PostgreSQL ড্রাইভার — "হয়তো লাগবে" (কখনো লাগেনি!)
class PostgreSQLDriver implements DatabaseDriverInterface
{
    public function connect(array $config): void { /* ... */ }
    public function query(string $sql, array $params = []): array { return []; }
    public function insert(string $table, array $data): int { return 0; }
    public function update(string $table, array $data, array $where): int { return 0; }
    public function delete(string $table, array $where): int { return 0; }
    public function beginTransaction(): void { /* ... */ }
    public function commit(): void { /* ... */ }
    public function rollback(): void { /* ... */ }
}

// SQLite ড্রাইভার — "হয়তো লাগবে" (কখনো লাগেনি!)
class SQLiteDriver implements DatabaseDriverInterface
{
    public function connect(array $config): void { /* ... */ }
    public function query(string $sql, array $params = []): array { return []; }
    public function insert(string $table, array $data): int { return 0; }
    public function update(string $table, array $data, array $where): int { return 0; }
    public function delete(string $table, array $where): int { return 0; }
    public function beginTransaction(): void { /* ... */ }
    public function commit(): void { /* ... */ }
    public function rollback(): void { /* ... */ }
}

// MongoDB ড্রাইভার — "হয়তো লাগবে" (কখনো লাগেনি!)
class MongoDBDriver implements DatabaseDriverInterface
{
    // SQL ইন্টারফেস MongoDB-তে জোর করে ফিট করা হচ্ছে!
    public function connect(array $config): void { /* ... */ }
    public function query(string $sql, array $params = []): array { return []; }
    public function insert(string $table, array $data): int { return 0; }
    public function update(string $table, array $data, array $where): int { return 0; }
    public function delete(string $table, array $where): int { return 0; }
    public function beginTransaction(): void { /* MongoDB-তে ট্রানজেকশন ভিন্ন! */ }
    public function commit(): void { /* ... */ }
    public function rollback(): void { /* ... */ }
}

// ড্রাইভার ফ্যাক্টরি — আরও জটিলতা!
class DatabaseDriverFactory
{
    private static array $drivers = [
        'mysql'      => MySQLDriver::class,
        'postgresql' => PostgreSQLDriver::class,
        'sqlite'     => SQLiteDriver::class,
        'mongodb'    => MongoDBDriver::class,
        // আরও ৬টি ড্রাইভার পরে যোগ হবে "কখনো"...
    ];

    public static function create(string $driver): DatabaseDriverInterface
    {
        if (!isset(self::$drivers[$driver])) {
            throw new \Exception("অসমর্থিত ড্রাইভার: {$driver}");
        }

        $class = self::$drivers[$driver];
        return new $class();
    }
}

// কনফিগ — ৪টি ড্রাইভারের কনফিগ, শুধু ১টি ব্যবহৃত
$config = [
    'default' => 'mysql',
    'drivers' => [
        'mysql'      => ['host' => 'localhost', 'database' => 'cms', /* ... */],
        'postgresql' => ['host' => 'localhost', 'database' => 'cms', /* ... */],
        'sqlite'     => ['path' => '/var/db/cms.sqlite'],
        'mongodb'    => ['host' => 'localhost', 'database' => 'cms'],
    ]
];
```

**✅ ভালো উদাহরণ (PHP) — YAGNI মেনে চলা:**

```php
<?php
// ✅ ভালো: শুধু MySQL, কারণ শুধু MySQL-ই দরকার!

class Database
{
    private PDO $pdo;

    public function __construct(array $config)
    {
        $dsn = "mysql:host={$config['host']};dbname={$config['database']};charset=utf8mb4";
        $this->pdo = new PDO($dsn, $config['username'], $config['password'], [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        ]);
    }

    public function query(string $sql, array $params = []): array
    {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll();
    }

    public function insert(string $table, array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $this->pdo->prepare("INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})")
            ->execute(array_values($data));

        return (int) $this->pdo->lastInsertId();
    }

    public function execute(string $sql, array $params = []): int
    {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt->rowCount();
    }
}

// ব্যবহার — সরল এবং পরিষ্কার
$db = new Database([
    'host'     => 'localhost',
    'database' => 'cms',
    'username' => 'root',
    'password' => 'secret',
]);

$posts = $db->query('SELECT * FROM posts WHERE status = ?', ['published']);

// ভবিষ্যতে যদি সত্যিই PostgreSQL লাগে?
// তখন ইন্টারফেস এক্সট্র্যাক্ট করো — ৩০ মিনিটের কাজ।
// এখন থেকে সেটা রেডি রাখার মানে হয় না!
```

---

### 💸 অব্যবহৃত কোডের রক্ষণাবেক্ষণ বোঝা

**❌ খারাপ উদাহরণ (JavaScript) — অব্যবহৃত ফিচার:**

```javascript
// ❌ খারাপ: "ভবিষ্যতে লাগবে" ফিচারগুলো — কখনো ব্যবহৃত হয়নি

class NotificationService {
    constructor() {
        // এখন শুধু ইমেইল ব্যবহার হচ্ছে
        this.emailProvider = new EmailProvider();

        // "হয়তো লাগবে" — ৬ মাস ধরে অব্যবহৃত
        this.smsProvider = new SMSProvider();
        this.pushProvider = new PushNotificationProvider();
        this.slackProvider = new SlackProvider();
        this.telegramProvider = new TelegramProvider();
        this.webhookProvider = new WebhookProvider();
    }

    // এটাই শুধু ব্যবহৃত হচ্ছে
    async sendEmail(to, subject, body) {
        return this.emailProvider.send(to, subject, body);
    }

    // এগুলো কখনো কল হয় না — কিন্তু মেইনটেইন করতে হচ্ছে!
    async sendSMS(to, message) {
        return this.smsProvider.send(to, message);
    }

    async sendPush(userId, notification) {
        return this.pushProvider.send(userId, notification);
    }

    async sendSlack(channel, message) {
        return this.slackProvider.send(channel, message);
    }

    async sendTelegram(chatId, message) {
        return this.telegramProvider.send(chatId, message);
    }

    async sendWebhook(url, payload) {
        return this.webhookProvider.send(url, payload);
    }

    // "ইউনিভার্সাল সেন্ডার" — অতিরিক্ত জটিল
    async send(type, recipient, content) {
        switch (type) {
            case 'email':    return this.sendEmail(recipient, content.subject, content.body);
            case 'sms':      return this.sendSMS(recipient, content.message);
            case 'push':     return this.sendPush(recipient, content);
            case 'slack':    return this.sendSlack(recipient, content.message);
            case 'telegram': return this.sendTelegram(recipient, content.message);
            case 'webhook':  return this.sendWebhook(recipient, content);
            default:         throw new Error(`অসমর্থিত নোটিফিকেশন টাইপ: ${type}`);
        }
    }
}
```

**✅ ভালো উদাহরণ (JavaScript) — YAGNI মেনে চলা:**

```javascript
// ✅ ভালো: শুধু ইমেইল, কারণ শুধু ইমেইলই দরকার

class NotificationService {
    constructor(emailConfig) {
        this.emailProvider = new EmailProvider(emailConfig);
    }

    async sendEmail(to, subject, body) {
        return this.emailProvider.send(to, subject, body);
    }
}

// ব্যাস! যখন SMS লাগবে, তখন যোগ করো।
// ৫টা অব্যবহৃত প্রোভাইডার মেইনটেইন করার চেয়ে
// ১টা কাজের প্রোভাইডার ভালো করে লেখা শ্রেয়।
```

---

## ৪. তিনটি নীতির মধ্যে সম্পর্ক ও দ্বন্দ্ব

### 🔄 তিন নীতির পারস্পরিক সম্পর্ক

```
┌──────────────────────────────────────────────────────────────┐
│            DRY, KISS, YAGNI এর সম্পর্ক                       │
│                                                              │
│                     ┌───────┐                                │
│                     │  DRY  │                                │
│                     │পুনরাবৃ│                                │
│                     │ত্তি নয়│                                │
│                     └───┬───┘                                │
│                    ╱         ╲                                │
│                  ╱             ╲                              │
│           সহযোগী             দ্বন্দ্ব                        │
│               ╱                 ╲                             │
│         ╱                         ╲                          │
│   ┌────────┐                 ┌────────┐                      │
│   │  YAGNI │ ◄── সহযোগী ──▶ │  KISS  │                      │
│   │ শুধু   │                 │ সরলতা  │                      │
│   │ দরকারি │                 │        │                      │
│   └────────┘                 └────────┘                      │
│                                                              │
│   🤝 সহযোগী: সাধারণত একসাথে কাজ করে                        │
│   ⚡ দ্বন্দ্ব: কখনো কখনো একে অপরের বিরোধী                  │
└──────────────────────────────────────────────────────────────┘
```

---

### ⚡ কখন DRY vs KISS দ্বন্দ্ব হয়

DRY মেনে চলতে গিয়ে কোড এত জটিল হয় যে KISS ভঙ্গ হয়:

**❌ খারাপ উদাহরণ (JavaScript) — DRY অতিরিক্ত, KISS ভঙ্গ:**

```javascript
// ❌ খারাপ: DRY মানতে গিয়ে জটিল "ইউনিভার্সাল" ফাংশন

// দুটো মিলতা কোড DRY করতে গিয়ে...
function processEntity(entity, type, options = {}) {
    const config = getConfigForType(type);
    const validator = options.customValidator || config.defaultValidator;
    const transformer = options.preTransform || (x => x);
    const postProcessor = options.postProcess || config.defaultPostProcessor;

    const transformed = transformer(entity);

    if (!validator(transformed)) {
        const errorHandler = options.onError || config.defaultErrorHandler;
        return errorHandler(new Error(`${type} ভ্যালিডেশন ব্যর্থ`));
    }

    const processed = config.processor(transformed);
    return postProcessor(processed);
}

// ব্যবহার — কী হচ্ছে বুঝতে কষ্ট!
processEntity(user, 'user', {
    customValidator: validateUser,
    preTransform: normalizeUserData,
    postProcess: enrichUserProfile
});

processEntity(product, 'product', {
    preTransform: normalizeProductData
});
```

**✅ ভালো উদাহরণ (JavaScript) — KISS কে প্রাধান্য:**

```javascript
// ✅ ভালো: সামান্য ডুপ্লিকেশন, কিন্তু পরিষ্কার ও বোধগম্য

function processUser(user) {
    const normalized = normalizeUserData(user);

    if (!validateUser(normalized)) {
        throw new Error('ইউজার ডেটা সঠিক নয়');
    }

    const saved = saveUser(normalized);
    return enrichUserProfile(saved);
}

function processProduct(product) {
    const normalized = normalizeProductData(product);

    if (!validateProduct(normalized)) {
        throw new Error('প্রোডাক্ট ডেটা সঠিক নয়');
    }

    return saveProduct(normalized);
}

// কিছু কোড ডুপ্লিকেট? হ্যাঁ। কিন্তু—
// ১. পড়তে সহজ
// ২. ডিবাগ করতে সহজ
// ৩. প্রতিটি ফাংশন স্বাধীনভাবে পরিবর্তন করা যায়
```

---

### ⚡ কখন YAGNI vs DRY দ্বন্দ্ব হয়

DRY মেনে চলতে গিয়ে এমন অ্যাবস্ট্রাকশন তৈরি হয় যা YAGNI ভঙ্গ করে:

**❌ খারাপ উদাহরণ (PHP) — DRY + YAGNI দ্বন্দ্ব:**

```php
<?php
// ❌ খারাপ: দুটো ফর্ম্যাটার DRY করতে গিয়ে
// একটি "ইউনিভার্সাল ফর্ম্যাটিং ফ্রেমওয়ার্ক" বানানো হয়েছে

interface FormatterInterface
{
    public function format($data): string;
}

interface FormatterFactoryInterface
{
    public function create(string $type): FormatterInterface;
}

abstract class AbstractFormatter implements FormatterInterface
{
    protected array $options;

    public function __construct(array $options = [])
    {
        $this->options = array_merge($this->getDefaultOptions(), $options);
    }

    abstract protected function getDefaultOptions(): array;
}

class FormatterRegistry
{
    private array $formatters = [];

    public function register(string $name, string $class): void
    {
        $this->formatters[$name] = $class;
    }

    public function get(string $name): FormatterInterface
    {
        return new $this->formatters[$name]();
    }
}

// ... আরও ৫-৬টি ক্লাস — শুধু দুটো ফর্ম্যাটারের জন্য!
```

**✅ ভালো উদাহরণ (PHP) — ভারসাম্য বজায়:**

```php
<?php
// ✅ ভালো: দুটো ফর্ম্যাটার? দুটো ফাংশন যথেষ্ট!

function formatCurrency(float $amount, string $currency = 'BDT'): string
{
    return number_format($amount, 2) . ' ' . $currency;
}

function formatDate(\DateTimeInterface $date, string $format = 'd/m/Y'): string
{
    return $date->format($format);
}

// ভবিষ্যতে ১০টি ফর্ম্যাটার লাগলে? তখন ভাবা যাবে।
```

---

### 🌳 সিদ্ধান্ত গাছ (Decision Tree)

কোন নীতি কখন প্রাধান্য পাবে, তার জন্য এই সিদ্ধান্ত গাছ অনুসরণ করো:

```
┌─────────────────────────────────────────────────────────────────┐
│               সিদ্ধান্ত গাছ: কোন নীতি কখন?                     │
│                                                                 │
│  এই ফিচার/কোড কি এখনই দরকার?                                   │
│       │                                                         │
│       ├── না ──▶ 🛑 লিখো না! (YAGNI)                           │
│       │                                                         │
│       └── হ্যাঁ                                                  │
│            │                                                    │
│            ▼                                                    │
│  একই লজিক কি ইতোমধ্যে আছে?                                    │
│       │                                                         │
│       ├── না ──▶ সবচেয়ে সরলভাবে লেখো (KISS)                   │
│       │                                                         │
│       └── হ্যাঁ                                                  │
│            │                                                    │
│            ▼                                                    │
│  এটা কি তৃতীয়বার পুনরাবৃত্তি?                                  │
│       │                                                         │
│       ├── না ──▶ ডুপ্লিকেট রাখো, নোট করো (Rule of Three)      │
│       │                                                         │
│       └── হ্যাঁ                                                  │
│            │                                                    │
│            ▼                                                    │
│  DRY করলে কি KISS ভঙ্গ হবে?                                    │
│       │                                                         │
│       ├── হ্যাঁ ──▶ সরলতা বজায় রেখে যতটুকু সম্ভব DRY করো     │
│       │            (সম্পূর্ণ DRY-এর চেয়ে পড়ার সুবিধা বেশি    │
│       │             গুরুত্বপূর্ণ)                                │
│       │                                                         │
│       └── না ──▶ ✅ DRY করো! Extract Method/Class              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 📊 বাস্তব প্রজেক্ট সিদ্ধান্তের উদাহরণ

| পরিস্থিতি | সিদ্ধান্ত | কারণ |
|-----------|----------|------|
| ২টি কন্ট্রোলারে একই ভ্যালিডেশন | ডুপ্লিকেট রাখো | Rule of Three — এখনো ২ বার |
| ৫টি জায়গায় একই তারিখ ফর্ম্যাটিং | DRY করো — ইউটিলিটি ফাংশন বানাও | ৩+ বার, এবং সরলভাবে DRY সম্ভব |
| "হয়তো GraphQL API লাগবে" | লিখো না | YAGNI — এখন REST কাজ করছে |
| জটিল জেনেরিক অ্যাবস্ট্রাকশন দিয়ে DRY | সরল ডুপ্লিকেশন রাখো | KISS > DRY — পড়ার সুবিধা বেশি গুরুত্বপূর্ণ |
| ১০+ জায়গায় একই এরর হ্যান্ডলিং প্যাটার্ন | DRY করো — মিডলওয়্যার বানাও | স্পষ্টভাবে পুনরাবৃত্তি, সরলভাবে সমাধান সম্ভব |
| "ভবিষ্যতে মাল্টি-টেন্যান্ট লাগতে পারে" | এখন লিখো না | YAGNI — প্রয়োজন হলে রিফ্যাক্টর করো |

---

## ৫. সম্পূর্ণ প্র্যাক্টিক্যাল উদাহরণ — ব্লগ প্ল্যাটফর্ম

### সমস্যা বর্ণনা

একটি ব্লগ প্ল্যাটফর্মের কোড যেখানে **DRY, KISS, এবং YAGNI** — তিনটিই ভঙ্গ হয়েছে। আমরা দেখবো কীভাবে এটি ঠিক করা যায়।

---

### ❌ খারাপ ভার্সন (PHP) — তিনটি নীতিই ভঙ্গ

```php
<?php
// ❌ খারাপ: ব্লগ প্ল্যাটফর্ম — DRY, KISS, YAGNI সব ভঙ্গ!

// [YAGNI ভঙ্গ] অতিরিক্ত অ্যাবস্ট্রাকশন — কখনো একাধিক ইমপ্লিমেন্টেশন হবে না
interface ContentRepositoryInterface
{
    public function find(int $id): ?array;
    public function findAll(array $criteria = []): array;
    public function save(array $data): int;
    public function update(int $id, array $data): bool;
    public function delete(int $id): bool;
}

// [YAGNI ভঙ্গ] "হয়তো ভবিষ্যতে ক্যাশিং লাগবে" — লাগেনি!
interface CacheInterface
{
    public function get(string $key): mixed;
    public function set(string $key, mixed $value, int $ttl = 3600): void;
    public function delete(string $key): void;
    public function flush(): void;
}

// [YAGNI ভঙ্গ] ক্যাশ ডেকোরেটর — অব্যবহৃত
class CachedContentRepository implements ContentRepositoryInterface
{
    public function __construct(
        private ContentRepositoryInterface $inner,
        private CacheInterface $cache
    ) {}

    public function find(int $id): ?array
    {
        $key = "content_{$id}";
        $cached = $this->cache->get($key);
        if ($cached !== null) return $cached;

        $result = $this->inner->find($id);
        if ($result) $this->cache->set($key, $result);
        return $result;
    }

    public function findAll(array $criteria = []): array { return $this->inner->findAll($criteria); }
    public function save(array $data): int { $this->cache->flush(); return $this->inner->save($data); }
    public function update(int $id, array $data): bool { $this->cache->delete("content_{$id}"); return $this->inner->update($id, $data); }
    public function delete(int $id): bool { $this->cache->delete("content_{$id}"); return $this->inner->delete($id); }
}

// [KISS ভঙ্গ] অতিরিক্ত জটিল পোস্ট ক্লাস
class BlogPostManager
{
    private ContentRepositoryInterface $repository;
    private CacheInterface $cache;
    private array $middlewares = [];          // কখনো ব্যবহৃত হয়নি
    private array $eventListeners = [];       // কখনো ব্যবহৃত হয়নি
    private ?string $currentLocale = null;    // বহুভাষা সাপোর্ট — দরকার নেই

    public function __construct(
        ContentRepositoryInterface $repository,
        CacheInterface $cache
    ) {
        $this->repository = $repository;
        $this->cache = $cache;
    }

    // [YAGNI] মিডলওয়্যার সিস্টেম — কখনো ব্যবহৃত হয়নি
    public function addMiddleware(callable $middleware): void
    {
        $this->middlewares[] = $middleware;
    }

    // [YAGNI] ইভেন্ট সিস্টেম — কখনো ব্যবহৃত হয়নি
    public function addEventListener(string $event, callable $listener): void
    {
        $this->eventListeners[$event][] = $listener;
    }

    public function createPost(array $data): int
    {
        // [DRY ভঙ্গ] ভ্যালিডেশন — createPost এ
        if (empty($data['title'])) {
            throw new \Exception('শিরোনাম দিতে হবে');
        }
        if (strlen($data['title']) > 200) {
            throw new \Exception('শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না');
        }
        if (empty($data['content'])) {
            throw new \Exception('বিষয়বস্তু দিতে হবে');
        }
        if (strlen($data['content']) < 50) {
            throw new \Exception('বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে');
        }

        // [KISS ভঙ্গ] স্লাগ জেনারেট — অতিরিক্ত জটিল
        $slug = $this->generateSlug($data['title']);

        $data['slug'] = $slug;
        $data['created_at'] = date('Y-m-d H:i:s');

        return $this->repository->save($data);
    }

    public function updatePost(int $id, array $data): bool
    {
        // [DRY ভঙ্গ] একই ভ্যালিডেশন আবার!
        if (empty($data['title'])) {
            throw new \Exception('শিরোনাম দিতে হবে');
        }
        if (strlen($data['title']) > 200) {
            throw new \Exception('শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না');
        }
        if (empty($data['content'])) {
            throw new \Exception('বিষয়বস্তু দিতে হবে');
        }
        if (strlen($data['content']) < 50) {
            throw new \Exception('বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে');
        }

        $data['slug'] = $this->generateSlug($data['title']);
        $data['updated_at'] = date('Y-m-d H:i:s');

        return $this->repository->update($id, $data);
    }

    // [KISS ভঙ্গ] অতিরিক্ত জটিল স্লাগ জেনারেটর
    private function generateSlug(string $title): string
    {
        // ইউনিকোড ট্রান্সলিটারেশন — বাংলা ব্লগে দরকার নেই
        $transliterator = \Transliterator::create('Any-Latin; Latin-ASCII');
        if ($transliterator) {
            $slug = $transliterator->transliterate($title);
        } else {
            $slug = $title;
        }

        $slug = strtolower($slug);
        $slug = preg_replace('/[^a-z0-9]+/', '-', $slug);
        $slug = trim($slug, '-');

        // ইউনিকনেস চেক — ৫ বার পর্যন্ত রিট্রাই
        $originalSlug = $slug;
        $counter = 1;
        while ($this->repository->findAll(['slug' => $slug])) {
            $slug = $originalSlug . '-' . $counter;
            $counter++;
            if ($counter > 5) break;
        }

        return $slug;
    }

    // [DRY ভঙ্গ] পোস্ট ফর্ম্যাটিং — getPost এ
    public function getPost(int $id): ?array
    {
        $post = $this->repository->find($id);
        if (!$post) return null;

        // ফর্ম্যাটিং — এখানে করা হচ্ছে
        $post['formatted_date'] = date('d M, Y', strtotime($post['created_at']));
        $post['reading_time'] = ceil(str_word_count($post['content']) / 200) . ' মিনিট';
        $post['excerpt'] = mb_substr(strip_tags($post['content']), 0, 150) . '...';

        return $post;
    }

    // [DRY ভঙ্গ] একই ফর্ম্যাটিং — getAllPosts এও!
    public function getAllPosts(): array
    {
        $posts = $this->repository->findAll();

        return array_map(function ($post) {
            // একই ফর্ম্যাটিং কোড আবার!
            $post['formatted_date'] = date('d M, Y', strtotime($post['created_at']));
            $post['reading_time'] = ceil(str_word_count($post['content']) / 200) . ' মিনিট';
            $post['excerpt'] = mb_substr(strip_tags($post['content']), 0, 150) . '...';

            return $post;
        }, $posts);
    }
}
```

---

### ✅ ভালো ভার্সন (PHP) — তিনটি নীতি অনুসরণ

```php
<?php
// ✅ ভালো: ব্লগ প্ল্যাটফর্ম — DRY, KISS, YAGNI সব মেনে চলা

/**
 * পোস্ট ভ্যালিডেটর — একক উৎস (DRY)
 */
class PostValidator
{
    public static function validate(array $data): ?string
    {
        if (empty($data['title'])) {
            return 'শিরোনাম দিতে হবে';
        }
        if (mb_strlen($data['title']) > 200) {
            return 'শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না';
        }
        if (empty($data['content'])) {
            return 'বিষয়বস্তু দিতে হবে';
        }
        if (mb_strlen($data['content']) < 50) {
            return 'বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে';
        }

        return null;
    }
}

/**
 * পোস্ট ফর্ম্যাটার — একক উৎস (DRY)
 */
class PostFormatter
{
    public static function format(array $post): array
    {
        $post['formatted_date'] = date('d M, Y', strtotime($post['created_at']));
        $post['reading_time'] = ceil(mb_substr_count($post['content'], ' ') / 200) . ' মিনিট';
        $post['excerpt'] = mb_substr(strip_tags($post['content']), 0, 150) . '...';

        return $post;
    }
}

/**
 * ব্লগ সার্ভিস — সরল ও প্রয়োজনীয় (KISS + YAGNI)
 *
 * কোনো অতিরিক্ত অ্যাবস্ট্রাকশন, ক্যাশ, মিডলওয়্যার, বা ইভেন্ট নেই।
 * যখন দরকার হবে, তখন যোগ করা যাবে।
 */
class BlogService
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function createPost(array $data): int
    {
        // শেয়ার্ড ভ্যালিডেটর (DRY)
        $error = PostValidator::validate($data);
        if ($error) {
            throw new \InvalidArgumentException($error);
        }

        // সরল স্লাগ জেনারেশন (KISS)
        $slug = $this->makeSlug($data['title']);

        $stmt = $this->db->prepare(
            'INSERT INTO posts (title, content, slug, created_at) VALUES (?, ?, ?, NOW())'
        );
        $stmt->execute([$data['title'], $data['content'], $slug]);

        return (int) $this->db->lastInsertId();
    }

    public function updatePost(int $id, array $data): void
    {
        // একই ভ্যালিডেটর পুনরায় ব্যবহার (DRY)
        $error = PostValidator::validate($data);
        if ($error) {
            throw new \InvalidArgumentException($error);
        }

        $slug = $this->makeSlug($data['title']);

        $stmt = $this->db->prepare(
            'UPDATE posts SET title = ?, content = ?, slug = ?, updated_at = NOW() WHERE id = ?'
        );
        $stmt->execute([$data['title'], $data['content'], $slug, $id]);
    }

    public function getPost(int $id): ?array
    {
        $stmt = $this->db->prepare('SELECT * FROM posts WHERE id = ?');
        $stmt->execute([$id]);
        $post = $stmt->fetch(PDO::FETCH_ASSOC);

        // শেয়ার্ড ফর্ম্যাটার (DRY)
        return $post ? PostFormatter::format($post) : null;
    }

    public function getAllPosts(int $page = 1, int $perPage = 10): array
    {
        $offset = ($page - 1) * $perPage;
        $stmt = $this->db->prepare('SELECT * FROM posts ORDER BY created_at DESC LIMIT ? OFFSET ?');
        $stmt->execute([$perPage, $offset]);
        $posts = $stmt->fetchAll(PDO::FETCH_ASSOC);

        // একই ফর্ম্যাটার (DRY)
        return array_map([PostFormatter::class, 'format'], $posts);
    }

    // সরল স্লাগ মেকার (KISS)
    private function makeSlug(string $title): string
    {
        $slug = mb_strtolower(trim($title));
        $slug = preg_replace('/[^\p{L}\p{N}]+/u', '-', $slug);
        return trim($slug, '-');
    }
}
```

---

### ❌ খারাপ ভার্সন (JavaScript) — তিনটি নীতিই ভঙ্গ

```javascript
// ❌ খারাপ: ব্লগ প্ল্যাটফর্ম — DRY, KISS, YAGNI সব ভঙ্গ!

// [YAGNI] ইভেন্ট বাস — কখনো ব্যবহৃত হয়নি
class EventBus {
    constructor() {
        this.listeners = {};
    }
    on(event, callback) {
        if (!this.listeners[event]) this.listeners[event] = [];
        this.listeners[event].push(callback);
    }
    emit(event, data) {
        (this.listeners[event] || []).forEach(cb => cb(data));
    }
}

// [YAGNI] প্লাগইন সিস্টেম — কোনো প্লাগইন নেই!
class PluginManager {
    constructor() {
        this.plugins = [];
    }
    register(plugin) {
        this.plugins.push(plugin);
    }
    executeHook(hookName, data) {
        return this.plugins.reduce((result, plugin) => {
            if (typeof plugin[hookName] === 'function') {
                return plugin[hookName](result);
            }
            return result;
        }, data);
    }
}

// [KISS ভঙ্গ + YAGNI] অতিরিক্ত জটিল ব্লগ সার্ভিস
class BlogPostService {
    constructor(db) {
        this.db = db;
        this.eventBus = new EventBus();           // অব্যবহৃত
        this.pluginManager = new PluginManager();   // অব্যবহৃত
        this.revisionHistory = [];                 // "হয়তো লাগবে"
    }

    async createPost(data) {
        // [DRY ভঙ্গ] ভ্যালিডেশন — create এ
        if (!data.title || data.title.trim() === '') {
            throw new Error('শিরোনাম দিতে হবে');
        }
        if (data.title.length > 200) {
            throw new Error('শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না');
        }
        if (!data.content || data.content.trim() === '') {
            throw new Error('বিষয়বস্তু দিতে হবে');
        }
        if (data.content.length < 50) {
            throw new Error('বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে');
        }

        const slug = this._generateSlug(data.title);

        const result = await this.db.query(
            'INSERT INTO posts (title, content, slug) VALUES (?, ?, ?)',
            [data.title, data.content, slug]
        );

        // [YAGNI] রিভিশন হিস্ট্রি — কখনো ব্যবহৃত হয়নি
        this.revisionHistory.push({
            postId: result.insertId,
            action: 'create',
            data: { ...data },
            timestamp: new Date()
        });

        return result.insertId;
    }

    async updatePost(id, data) {
        // [DRY ভঙ্গ] একই ভ্যালিডেশন আবার!
        if (!data.title || data.title.trim() === '') {
            throw new Error('শিরোনাম দিতে হবে');
        }
        if (data.title.length > 200) {
            throw new Error('শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না');
        }
        if (!data.content || data.content.trim() === '') {
            throw new Error('বিষয়বস্তু দিতে হবে');
        }
        if (data.content.length < 50) {
            throw new Error('বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে');
        }

        const slug = this._generateSlug(data.title);

        await this.db.query(
            'UPDATE posts SET title = ?, content = ?, slug = ? WHERE id = ?',
            [data.title, data.content, slug, id]
        );
    }

    // [DRY ভঙ্গ] ফর্ম্যাটিং getPost এ
    async getPost(id) {
        const [post] = await this.db.query('SELECT * FROM posts WHERE id = ?', [id]);
        if (!post) return null;

        post.formattedDate = new Date(post.created_at).toLocaleDateString('bn-BD');
        post.readingTime = Math.ceil(post.content.split(' ').length / 200) + ' মিনিট';
        post.excerpt = post.content.replace(/<[^>]*>/g, '').substring(0, 150) + '...';

        return post;
    }

    // [DRY ভঙ্গ] একই ফর্ম্যাটিং getAllPosts এও!
    async getAllPosts() {
        const posts = await this.db.query('SELECT * FROM posts ORDER BY created_at DESC');

        return posts.map(post => {
            post.formattedDate = new Date(post.created_at).toLocaleDateString('bn-BD');
            post.readingTime = Math.ceil(post.content.split(' ').length / 200) + ' মিনিট';
            post.excerpt = post.content.replace(/<[^>]*>/g, '').substring(0, 150) + '...';
            return post;
        });
    }

    // [KISS ভঙ্গ] অতিরিক্ত জটিল স্লাগ জেনারেটর
    _generateSlug(title) {
        const charMap = {
            'à': 'a', 'á': 'a', 'â': 'a', 'ã': 'a', 'ä': 'a', 'å': 'a',
            'è': 'e', 'é': 'e', 'ê': 'e', 'ë': 'e',
            'ì': 'i', 'í': 'i', 'î': 'i', 'ï': 'i',
            'ò': 'o', 'ó': 'o', 'ô': 'o', 'õ': 'o', 'ö': 'o',
            'ù': 'u', 'ú': 'u', 'û': 'u', 'ü': 'u',
            'ñ': 'n', 'ç': 'c', 'ß': 'ss',
            // ... আরও ৫০+ ম্যাপিং — বাংলা ব্লগে দরকার নেই!
        };

        let slug = title.toLowerCase();
        for (const [from, to] of Object.entries(charMap)) {
            slug = slug.replace(new RegExp(from, 'g'), to);
        }
        slug = slug.replace(/[^a-z0-9\u0980-\u09FF]+/g, '-');
        return slug.replace(/^-+|-+$/g, '');
    }
}
```

---

### ✅ ভালো ভার্সন (JavaScript) — তিনটি নীতি অনুসরণ

```javascript
// ✅ ভালো: ব্লগ প্ল্যাটফর্ম — DRY, KISS, YAGNI সব মেনে চলা

/**
 * পোস্ট ভ্যালিডেটর — একক উৎস (DRY)
 */
function validatePost(data) {
    if (!data.title?.trim()) {
        return 'শিরোনাম দিতে হবে';
    }
    if (data.title.length > 200) {
        return 'শিরোনাম ২০০ অক্ষরের বেশি হতে পারবে না';
    }
    if (!data.content?.trim()) {
        return 'বিষয়বস্তু দিতে হবে';
    }
    if (data.content.length < 50) {
        return 'বিষয়বস্তু কমপক্ষে ৫০ অক্ষর হতে হবে';
    }
    return null;
}

/**
 * পোস্ট ফর্ম্যাটার — একক উৎস (DRY)
 */
function formatPost(post) {
    return {
        ...post,
        formattedDate: new Date(post.created_at).toLocaleDateString('bn-BD'),
        readingTime: Math.ceil(post.content.split(' ').length / 200) + ' মিনিট',
        excerpt: post.content.replace(/<[^>]*>/g, '').substring(0, 150) + '...',
    };
}

/**
 * সরল স্লাগ মেকার (KISS)
 */
function makeSlug(title) {
    return title
        .toLowerCase()
        .trim()
        .replace(/[^a-z0-9\u0980-\u09FF]+/g, '-')
        .replace(/^-+|-+$/g, '');
}

/**
 * ব্লগ সার্ভিস — সরল, DRY, এবং শুধু প্রয়োজনীয় (KISS + YAGNI)
 *
 * কোনো ইভেন্ট বাস, প্লাগইন সিস্টেম, বা রিভিশন হিস্ট্রি নেই।
 * যখন সত্যিই দরকার হবে, তখন যোগ করা যাবে।
 */
class BlogService {
    constructor(db) {
        this.db = db;
    }

    async createPost(data) {
        // শেয়ার্ড ভ্যালিডেটর (DRY)
        const error = validatePost(data);
        if (error) throw new Error(error);

        const slug = makeSlug(data.title);
        const result = await this.db.query(
            'INSERT INTO posts (title, content, slug) VALUES (?, ?, ?)',
            [data.title, data.content, slug]
        );
        return result.insertId;
    }

    async updatePost(id, data) {
        // একই ভ্যালিডেটর (DRY)
        const error = validatePost(data);
        if (error) throw new Error(error);

        const slug = makeSlug(data.title);
        await this.db.query(
            'UPDATE posts SET title = ?, content = ?, slug = ? WHERE id = ?',
            [data.title, data.content, slug, id]
        );
    }

    async getPost(id) {
        const [post] = await this.db.query('SELECT * FROM posts WHERE id = ?', [id]);
        // শেয়ার্ড ফর্ম্যাটার (DRY)
        return post ? formatPost(post) : null;
    }

    async getAllPosts(page = 1, perPage = 10) {
        const offset = (page - 1) * perPage;
        const posts = await this.db.query(
            'SELECT * FROM posts ORDER BY created_at DESC LIMIT ? OFFSET ?',
            [perPage, offset]
        );
        // একই ফর্ম্যাটার (DRY)
        return posts.map(formatPost);
    }
}

module.exports = { BlogService, validatePost, formatPost, makeSlug };
```

---

## ৬. চেকলিস্ট ও তুলনামূলক টেবিল

### 📋 চেকলিস্ট — নীতি ভঙ্গের লক্ষণ ও সমাধান

| নীতি | ভঙ্গের লক্ষণ | সমাধান |
|------|-------------|--------|
| **DRY** | একই কোড কপি-পেস্ট হয়েছে | Extract Method / Extract Class |
| **DRY** | একই লজিক ভিন্ন ভাবে লেখা | একটি শেয়ার্ড ফাংশন বানাও |
| **DRY** | কোড আর ডকুমেন্টেশনে একই তথ্য | তথ্য কোডে রাখো, ডক রেফার করুক |
| **DRY** | বাগ ফিক্স একাধিক জায়গায় করতে হচ্ছে | লজিক কেন্দ্রীভূত করো |
| **DRY** | একটি পরিবর্তনে অনেক ফাইল পরিবর্তন লাগে | Single Source of Truth তৈরি করো |
| **KISS** | কোড বুঝতে ৫+ মিনিট সময় লাগে | সরল করো, নাম পরিষ্কার করো |
| **KISS** | একটি ক্লাসে ১০+ ডিপেন্ডেন্সি | ভেঙে ছোট করো, বা সরলতর পদ্ধতি ব্যবহার করো |
| **KISS** | জেনেরিক সমাধান যখন নির্দিষ্ট সমাধান যথেষ্ট | নির্দিষ্ট সমাধান ব্যবহার করো |
| **KISS** | "চালাক" কোড — অন্যরা বুঝতে পারে না | সরল, স্পষ্ট কোড লেখো |
| **KISS** | Premature optimization | "প্রথমে কাজ করাও, তারপর দ্রুত করো" |
| **YAGNI** | অব্যবহৃত কোড / ডেড কোড | মুছে ফেলো |
| **YAGNI** | "হয়তো লাগবে" ফিচার | মুছো — Git-এ থাকবে |
| **YAGNI** | খালি ইন্টারফেস / অ্যাবস্ট্রাক্ট ক্লাস | একটাই ইমপ্লিমেন্টেশন থাকলে ইন্টারফেস লাগে না |
| **YAGNI** | কনফিগারেশন অপশনের ৮০%+ অব্যবহৃত | শুধু ব্যবহৃত অপশন রাখো |
| **YAGNI** | "এক্সটেনসিবিলিটি পয়েন্ট" যা কেউ ব্যবহার করে না | সরিয়ে ফেলো |

---

### 📊 তুলনামূলক টেবিল — DRY vs KISS vs YAGNI

| বৈশিষ্ট্য | DRY | KISS | YAGNI |
|----------|-----|------|-------|
| **মূল বক্তব্য** | পুনরাবৃত্তি এড়াও | সরল রাখো | শুধু দরকারি লেখো |
| **উৎস** | The Pragmatic Programmer (১৯৯৯) | Kelly Johnson, Lockheed (১৯৬০) | Extreme Programming (১৯৯০) |
| **বিপরীত** | WET (Write Everything Twice) | Over-engineering | Speculative Generality |
| **মূল প্রশ্ন** | "এই লজিক কি অন্য কোথাও আছে?" | "এটা কি সবচেয়ে সরল সমাধান?" | "এটা কি এখনই দরকার?" |
| **অতিরিক্ত প্রয়োগের ঝুঁকি** | জটিল অ্যাবস্ট্রাকশন | অপর্যাপ্ত ডিজাইন | প্রয়োজনীয় ফিচারও বাদ দেওয়া |
| **কখন ভাঙা যায়** | ভিন্ন কনটেক্সট, Rule of Three | পার্ফরম্যান্স ক্রিটিক্যাল কোড | নিরাপত্তা, মূল আর্কিটেকচার |
| **টুল সাপোর্ট** | IDE "Extract" রিফ্যাক্টরিং | কোড রিভিউ, লিন্টার | কোড কভারেজ, ডেড কোড অ্যানালাইসিস |
| **শেখার কঠিনতা** | মাঝারি | সহজ (বোঝা সহজ, মানা কঠিন) | সহজ (বোঝা সহজ, মানা কঠিন) |
| **সেরা অভ্যাস** | শেয়ার্ড সার্ভিস/ইউটিলিটি | পড়ার যোগ্য, সরল কোড | ক্রমবর্ধমান উন্নয়ন (Iterative) |

---

### 🎯 সারসংক্ষেপ — তিন নীতির সমন্বয়

```
┌──────────────────────────────────────────────────────────────────┐
│                     ক্লিন কোডের সূত্র                            │
│                                                                  │
│   ┌──────────────────────────────────────────────────────┐       │
│   │                                                      │       │
│   │   ১. প্রথমে জিজ্ঞেস করো: "এটা কি দরকার?"  (YAGNI)  │       │
│   │          │                                           │       │
│   │          ▼  হ্যাঁ                                     │       │
│   │                                                      │       │
│   │   ২. তারপর জিজ্ঞেস করো: "সবচেয়ে সরল               │       │
│   │      সমাধান কী?"                           (KISS)    │       │
│   │          │                                           │       │
│   │          ▼  লেখা হলো                                  │       │
│   │                                                      │       │
│   │   ৩. শেষে জিজ্ঞেস করো: "এটা কি আগে                 │       │
│   │      লেখা আছে?"                           (DRY)     │       │
│   │          │                                           │       │
│   │          ▼  হ্যাঁ → রিফ্যাক্টর (Rule of Three)       │       │
│   │                                                      │       │
│   │   🎉 ক্লিন কোড!                                      │       │
│   │                                                      │       │
│   └──────────────────────────────────────────────────────┘       │
│                                                                  │
│   মনে রাখো:                                                      │
│   • কোনো নীতিই পরম সত্য নয় — প্রসঙ্গ অনুযায়ী সিদ্ধান্ত নাও   │
│   • পড়ার সুবিধা (Readability) সবসময় সর্বোচ্চ অগ্রাধিকার       │
│   • নিয়ম শেখো, তারপর কখন ভাঙতে হয় সেটাও শেখো                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔗 পরবর্তী টপিক

➡️ [Design by Contract — চুক্তি ভিত্তিক ডিজাইন](./design-by-contract.md)

---

> "সবকিছু যতটা সম্ভব সরল করা উচিত, কিন্তু সরলতর নয়।" — আলবার্ট আইনস্টাইন
