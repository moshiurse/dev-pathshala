# ✅ ইনপুট ভ্যালিডেশন ও স্যানিটাইজেশন (Input Validation & Sanitization)

## 📌 ধারণা (Concept)

ইনপুট ভ্যালিডেশন হলো ইউজারের দেওয়া ডেটা গ্রহণ করার আগে সেটি সঠিক কিনা যাচাই করা। স্যানিটাইজেশন হলো ইনপুট থেকে ক্ষতিকর বা অনাকাঙ্ক্ষিত অংশ সরিয়ে দেওয়া।

**মূলনীতি:** কখনো ইউজারের ইনপুটকে বিশ্বাস করবেন না — সবসময় ভ্যালিডেট ও স্যানিটাইজ করুন।

- **Validation:** ইনপুট সঠিক ফরম্যাটে আছে কিনা যাচাই (গ্রহণ বা প্রত্যাখ্যান)
- **Sanitization:** ইনপুট থেকে ক্ষতিকর অংশ সরিয়ে পরিষ্কার করা

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন একটি পার্সেল ডেলিভারি সার্ভিসে:
- **Validation:** পার্সেলের ওজন ২০ কেজির বেশি হলে গ্রহণ করা হবে না — "না, এটি গ্রহণযোগ্য নয়"
- **Sanitization:** পার্সেলের গায়ে লেখা ঠিকানা থেকে অপ্রয়োজনীয় স্টিকার ছিঁড়ে ফেলা — পার্সেল গ্রহণ করা হচ্ছে, তবে পরিষ্কার করে

---

## 🔵 সার্ভার-সাইড ভ্যালিডেশন (PHP)

### ১. বেসিক ভ্যালিডেশন ক্লাস

```php
class Validator {
    private array $errors = [];
    private array $data;

    public function __construct(array $data) {
        $this->data = $data;
    }

    public function required(string $field, string $label): self {
        if (empty($this->data[$field]) && $this->data[$field] !== '0') {
            $this->errors[$field][] = "$label আবশ্যক";
        }
        return $this;
    }

    public function email(string $field): self {
        if (!empty($this->data[$field]) && !filter_var($this->data[$field], FILTER_VALIDATE_EMAIL)) {
            $this->errors[$field][] = "সঠিক ইমেইল দিন";
        }
        return $this;
    }

    public function minLength(string $field, int $min, string $label): self {
        if (!empty($this->data[$field]) && mb_strlen($this->data[$field]) < $min) {
            $this->errors[$field][] = "$label কমপক্ষে $min অক্ষর হতে হবে";
        }
        return $this;
    }

    public function maxLength(string $field, int $max, string $label): self {
        if (!empty($this->data[$field]) && mb_strlen($this->data[$field]) > $max) {
            $this->errors[$field][] = "$label সর্বোচ্চ $max অক্ষর হতে পারে";
        }
        return $this;
    }

    public function numeric(string $field, string $label): self {
        if (!empty($this->data[$field]) && !is_numeric($this->data[$field])) {
            $this->errors[$field][] = "$label সংখ্যা হতে হবে";
        }
        return $this;
    }

    public function in(string $field, array $allowed, string $label): self {
        if (!empty($this->data[$field]) && !in_array($this->data[$field], $allowed, true)) {
            $this->errors[$field][] = "$label অবৈধ মান";
        }
        return $this;
    }

    public function isValid(): bool {
        return empty($this->errors);
    }

    public function getErrors(): array {
        return $this->errors;
    }
}

// ব্যবহার
$validator = new Validator($_POST);
$validator
    ->required('name', 'নাম')
    ->minLength('name', 2, 'নাম')
    ->maxLength('name', 100, 'নাম')
    ->required('email', 'ইমেইল')
    ->email('email')
    ->required('age', 'বয়স')
    ->numeric('age', 'বয়স')
    ->in('role', ['user', 'admin', 'editor'], 'ভূমিকা');

if (!$validator->isValid()) {
    http_response_code(422);
    echo json_encode(['errors' => $validator->getErrors()]);
    exit;
}
```

### ২. PHP filter_var দিয়ে ভ্যালিডেশন ও স্যানিটাইজেশন

```php
// ভ্যালিডেশন
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);       // false বা পরিষ্কার email
$url = filter_var($_POST['website'], FILTER_VALIDATE_URL);          // false বা URL
$int = filter_var($_POST['age'], FILTER_VALIDATE_INT, [
    'options' => ['min_range' => 1, 'max_range' => 150]
]);

// স্যানিটাইজেশন
$cleanEmail = filter_var($_POST['email'], FILTER_SANITIZE_EMAIL);
$cleanString = filter_var($_POST['name'], FILTER_SANITIZE_SPECIAL_CHARS);
$cleanInt = filter_var($_POST['age'], FILTER_SANITIZE_NUMBER_INT);
$cleanUrl = filter_var($_POST['website'], FILTER_SANITIZE_URL);
```

---

## 🔵 সার্ভার-সাইড ভ্যালিডেশন (JavaScript / Node.js)

### ১. express-validator ব্যবহার

```javascript
import { body, param, query, validationResult } from 'express-validator';

// ভ্যালিডেশন রুলস
const registerValidation = [
    body('name')
        .trim()
        .notEmpty().withMessage('নাম আবশ্যক')
        .isLength({ min: 2, max: 100 }).withMessage('নাম ২-১০০ অক্ষরের মধ্যে হতে হবে')
        .escape(),

    body('email')
        .trim()
        .notEmpty().withMessage('ইমেইল আবশ্যক')
        .isEmail().withMessage('সঠিক ইমেইল দিন')
        .normalizeEmail(),

    body('password')
        .notEmpty().withMessage('পাসওয়ার্ড আবশ্যক')
        .isLength({ min: 8 }).withMessage('পাসওয়ার্ড কমপক্ষে ৮ অক্ষর')
        .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
        .withMessage('পাসওয়ার্ডে ছোট হাতের, বড় হাতের অক্ষর ও সংখ্যা থাকতে হবে'),

    body('age')
        .optional()
        .isInt({ min: 1, max: 150 }).withMessage('সঠিক বয়স দিন')
];

// ভ্যালিডেশন মিডলওয়্যার
const validate = (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(422).json({
            errors: errors.array().map(err => ({
                field: err.path,
                message: err.msg
            }))
        });
    }
    next();
};

app.post('/register', registerValidation, validate, registerUser);
```

### ২. Zod দিয়ে Schema Validation

```javascript
import { z } from 'zod';

const userSchema = z.object({
    name: z.string()
        .min(2, 'নাম কমপক্ষে ২ অক্ষর')
        .max(100, 'নাম সর্বোচ্চ ১০০ অক্ষর')
        .trim(),
    email: z.string()
        .email('সঠিক ইমেইল দিন')
        .toLowerCase(),
    age: z.number()
        .int('পূর্ণ সংখ্যা দিন')
        .min(1).max(150)
        .optional(),
    role: z.enum(['user', 'admin', 'editor'])
        .default('user')
});

// ব্যবহার
app.post('/users', (req, res) => {
    const result = userSchema.safeParse(req.body);

    if (!result.success) {
        return res.status(422).json({
            errors: result.error.issues.map(issue => ({
                field: issue.path.join('.'),
                message: issue.message
            }))
        });
    }

    createUser(result.data); // ভ্যালিডেটেড ও টাইপ-সেফ ডেটা
});
```

---

## 🟡 ক্লায়েন্ট-সাইড ভ্যালিডেশন

```javascript
// ক্লায়েন্ট-সাইড ভ্যালিডেশন শুধু UX উন্নতির জন্য — সিকিউরিটির জন্য নয়
class FormValidator {
    #errors = {};

    validateEmail(email) {
        const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!regex.test(email)) {
            this.#errors.email = 'সঠিক ইমেইল দিন';
        }
        return this;
    }

    validateRequired(field, value, label) {
        if (!value || value.trim() === '') {
            this.#errors[field] = `${label} আবশ্যক`;
        }
        return this;
    }

    validatePhone(phone) {
        const regex = /^(?:\+88)?01[3-9]\d{8}$/; // বাংলাদেশি ফোন নম্বর
        if (!regex.test(phone)) {
            this.#errors.phone = 'সঠিক ফোন নম্বর দিন';
        }
        return this;
    }

    isValid() { return Object.keys(this.#errors).length === 0; }
    getErrors() { return this.#errors; }
}
```

---

## ⚠️ সাধারণ ভুল ও সমাধান

```php
// ❌ শুধু ক্লায়েন্ট-সাইড ভ্যালিডেশনের উপর নির্ভর করা
// আক্রমণকারী সহজেই JavaScript বন্ধ করতে বা সরাসরি API কল করতে পারে

// ❌ Blacklist ভ্যালিডেশন
$forbidden = ['<script>', 'DROP TABLE'];
// আক্রমণকারী এনকোডিং বা ভিন্ন ফরম্যাট দিয়ে বাইপাস করতে পারে

// ✅ Whitelist ভ্যালিডেশন (সবসময় পছন্দনীয়)
$allowedRoles = ['user', 'admin', 'editor'];
if (!in_array($role, $allowedRoles, true)) {
    throw new ValidationException('অবৈধ ভূমিকা');
}
```

---

## ✅ কখন ব্যবহার করবেন

- **সার্ভার-সাইড ভ্যালিডেশন:** সবসময় — এটি বাধ্যতামূলক
- **ক্লায়েন্ট-সাইড ভ্যালিডেশন:** UX উন্নতির জন্য — দ্রুত ফিডব্যাক দিতে
- **Whitelist approach:** সবসময় — শুধু অনুমোদিত মান গ্রহণ করুন
- **Schema validation (Zod, Joi):** API এন্ডপয়েন্ট ও ফর্ম ডেটা ভ্যালিডেশনে

## ❌ কখন ব্যবহার করবেন না

- **শুধু ক্লায়েন্ট-সাইড ভ্যালিডেশন** দিয়ে সিকিউরিটি নিশ্চিত করার চেষ্টা করবেন না
- **Blacklist approach** (নির্দিষ্ট কিছু বাদ দেওয়া) — এটি সহজে বাইপাস করা যায়
- **অতিরিক্ত কঠোর ভ্যালিডেশন** যা বৈধ ইনপুট প্রত্যাখ্যান করে (যেমন: নামে unicode ক্যারেক্টার না নেওয়া)
- ভ্যালিডেশন এরর মেসেজে সিস্টেমের অভ্যন্তরীণ তথ্য দেবেন না
