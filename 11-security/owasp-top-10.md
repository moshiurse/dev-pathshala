# 🛡️ OWASP Top 10 — ওয়েব অ্যাপ্লিকেশনের শীর্ষ ১০ সিকিউরিটি ঝুঁকি

## 📌 ধারণা (Concept)

OWASP (Open Web Application Security Project) হলো একটি আন্তর্জাতিক অলাভজনক সংস্থা যারা ওয়েব অ্যাপ্লিকেশনের সিকিউরিটি নিয়ে কাজ করে। তাদের প্রকাশিত **OWASP Top 10** হলো ওয়েব অ্যাপ্লিকেশনের সবচেয়ে সাধারণ ও বিপজ্জনক ১০টি সিকিউরিটি ঝুঁকির তালিকা। এই তালিকা প্রতি কয়েক বছর আপডেট হয়।

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন আপনার একটি বাড়ি আছে। OWASP Top 10 হলো সেই বাড়ির সবচেয়ে সাধারণ ১০টি দুর্বলতার তালিকা — যেমন দরজায় তালা না থাকা, জানালা খোলা থাকা, চাবি দরজার নিচে রাখা ইত্যাদি। একজন নিরাপত্তা বিশেষজ্ঞ প্রথমে এই ১০টি দুর্বলতা চেক করবে।

---

## 📋 OWASP Top 10 (2021 সংস্করণ)

### ১. A01: Broken Access Control (ভাঙা অ্যাক্সেস কন্ট্রোল)

ইউজার তার অনুমোদিত সীমার বাইরে কাজ করতে পারে।

```php
// ❌ খারাপ উদাহরণ — কোনো অ্যাক্সেস চেক নেই
function getUserProfile($userId) {
    $query = "SELECT * FROM users WHERE id = $userId";
    return DB::query($query);
}

// ✅ ভালো উদাহরণ — অ্যাক্সেস চেক আছে
function getUserProfile($userId, $currentUser) {
    if ($currentUser->id !== $userId && !$currentUser->isAdmin()) {
        throw new ForbiddenException('আপনার এই প্রোফাইল দেখার অনুমতি নেই');
    }
    $query = "SELECT * FROM users WHERE id = ?";
    return DB::prepare($query, [$userId]);
}
```

```javascript
// ✅ JavaScript — Middleware দিয়ে অ্যাক্সেস কন্ট্রোল
const checkOwnership = async (req, res, next) => {
    const resource = await Resource.findById(req.params.id);
    if (resource.userId !== req.user.id && req.user.role !== 'admin') {
        return res.status(403).json({ error: 'অনুমতি নেই' });
    }
    next();
};

app.get('/api/profile/:id', authenticate, checkOwnership, getProfile);
```

### ২. A02: Cryptographic Failures (ক্রিপ্টোগ্রাফিক ব্যর্থতা)

সংবেদনশীল ডেটা সঠিকভাবে এনক্রিপ্ট না করা।

```php
// ❌ খারাপ — পাসওয়ার্ড প্লেইন টেক্সটে সংরক্ষণ
$password = $_POST['password'];
$query = "INSERT INTO users (password) VALUES ('$password')";

// ✅ ভালো — bcrypt দিয়ে হ্যাশ করা
$hashedPassword = password_hash($_POST['password'], PASSWORD_BCRYPT);
$stmt = $pdo->prepare("INSERT INTO users (password) VALUES (?)");
$stmt->execute([$hashedPassword]);
```

### ৩. A03: Injection (ইনজেকশন)

ইউজারের ইনপুট সরাসরি কোয়েরি বা কমান্ডে ব্যবহার করা। (বিস্তারিত: [sql-injection.md](./sql-injection.md))

### ৪. A04: Insecure Design (অনিরাপদ ডিজাইন)

ডিজাইন পর্যায়ে সিকিউরিটি বিবেচনা না করা।

```php
// ❌ অনিরাপদ ডিজাইন — পাসওয়ার্ড রিসেট লিংকে কোনো expiry নেই
function generateResetLink($email) {
    $token = bin2hex(random_bytes(32));
    DB::insert('password_resets', ['email' => $email, 'token' => $token]);
    return "/reset?token=$token";
}

// ✅ নিরাপদ ডিজাইন — expiry ও rate limiting সহ
function generateResetLink($email) {
    $recentAttempts = DB::count('password_resets', [
        'email' => $email,
        'created_at >' => date('Y-m-d H:i:s', strtotime('-1 hour'))
    ]);
    if ($recentAttempts >= 3) {
        throw new RateLimitException('অনুগ্রহ করে পরে চেষ্টা করুন');
    }

    $token = bin2hex(random_bytes(32));
    DB::insert('password_resets', [
        'email' => $email,
        'token' => $token,
        'expires_at' => date('Y-m-d H:i:s', strtotime('+15 minutes'))
    ]);
    return "/reset?token=$token";
}
```

### ৫. A05: Security Misconfiguration (সিকিউরিটি ভুল কনফিগারেশন)

ডিফল্ট সেটিংস, অপ্রয়োজনীয় ফিচার চালু থাকা, এরর মেসেজে তথ্য ফাঁস।

```javascript
// ❌ প্রোডাকশনে ডিবাগ মোড চালু
app.use(errorHandler({ showStack: true }));

// ✅ প্রোডাকশনে এরর মেসেজ সীমিত
app.use((err, req, res, next) => {
    console.error(err.stack); // শুধু লগে রাখা
    res.status(500).json({
        error: 'কিছু সমস্যা হয়েছে, অনুগ্রহ করে পরে চেষ্টা করুন'
    });
});
```

### ৬. A06: Vulnerable and Outdated Components (পুরোনো ও ঝুঁকিপূর্ণ কম্পোনেন্ট)

পুরোনো লাইব্রেরি বা ফ্রেমওয়ার্ক ব্যবহার যেখানে জানা দুর্বলতা আছে।

```bash
# নিয়মিত ডিপেন্ডেন্সি চেক করুন
composer audit          # PHP
npm audit               # JavaScript
npm audit fix           # স্বয়ংক্রিয় ফিক্স
```

### ৭. A07: Identification and Authentication Failures

দুর্বল অথেনটিকেশন সিস্টেম।

```php
// ✅ শক্তিশালী অথেনটিকেশন
function login($email, $password) {
    $user = DB::findByEmail($email);
    if (!$user || !password_verify($password, $user->password)) {
        // নির্দিষ্ট করে বলবেন না কোনটা ভুল
        throw new AuthException('ইমেইল বা পাসওয়ার্ড ভুল');
    }

    if ($user->two_factor_enabled) {
        return ['requires_2fa' => true, 'temp_token' => generateTempToken($user)];
    }

    return generateSession($user);
}
```

### ৮. A08: Software and Data Integrity Failures

সফটওয়্যার আপডেট বা CI/CD পাইপলাইনে integrity check না করা।

```javascript
// ✅ Subresource Integrity (SRI) ব্যবহার
// HTML এ CDN থেকে লোড করা স্ক্রিপ্টে integrity hash যোগ করুন
`<script
    src="https://cdn.example.com/lib.js"
    integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxAh..."
    crossorigin="anonymous">
</script>`;
```

### ৯. A09: Security Logging and Monitoring Failures

পর্যাপ্ত লগিং ও মনিটরিং না থাকা।

```php
// ✅ সিকিউরিটি ইভেন্ট লগিং
function logSecurityEvent($type, $details, $userId = null) {
    $log = [
        'type' => $type,
        'details' => $details,
        'user_id' => $userId,
        'ip' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'timestamp' => date('c')
    ];
    SecurityLogger::critical(json_encode($log));
}

// ব্যবহার
logSecurityEvent('LOGIN_FAILED', ['email' => $email], null);
logSecurityEvent('PRIVILEGE_ESCALATION_ATTEMPT', ['action' => $action], $userId);
```

### ১০. A10: Server-Side Request Forgery (SSRF)

সার্ভার থেকে অননুমোদিত HTTP রিকোয়েস্ট পাঠানো।

```javascript
// ❌ SSRF ঝুঁকি — ইউজারের দেওয়া URL সরাসরি ফেচ করা
app.get('/fetch', async (req, res) => {
    const response = await fetch(req.query.url);
    res.send(await response.text());
});

// ✅ URL হোয়াইটলিস্ট ও ভ্যালিডেশন
const ALLOWED_DOMAINS = ['api.example.com', 'cdn.example.com'];

app.get('/fetch', async (req, res) => {
    const url = new URL(req.query.url);
    if (!ALLOWED_DOMAINS.includes(url.hostname)) {
        return res.status(403).json({ error: 'এই ডোমেইন অনুমোদিত নয়' });
    }
    if (url.protocol !== 'https:') {
        return res.status(400).json({ error: 'শুধু HTTPS অনুমোদিত' });
    }
    const response = await fetch(url.toString());
    res.send(await response.text());
});
```

---

## ✅ কখন ব্যবহার করবেন

- প্রতিটি প্রজেক্টের সিকিউরিটি রিভিউতে OWASP Top 10 চেকলিস্ট হিসেবে ব্যবহার করুন
- নতুন ফিচার ডিজাইন করার সময় এই তালিকা মাথায় রাখুন
- কোড রিভিউতে এই ঝুঁকিগুলো চেক করুন
- সিকিউরিটি অডিটের ভিত্তি হিসেবে ব্যবহার করুন

## ❌ কখন যথেষ্ট নয়

- OWASP Top 10 সব ধরনের সিকিউরিটি ঝুঁকি কভার করে না
- ডোমেইন-স্পেসিফিক ঝুঁকির জন্য আলাদা বিশ্লেষণ দরকার
- এটি শুধু একটি শুরু, সম্পূর্ণ সিকিউরিটি প্রোগ্রামের বিকল্প নয়
