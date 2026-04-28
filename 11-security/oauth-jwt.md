# 🔑 OAuth 2.0 ও JWT — টোকেন-ভিত্তিক অথেনটিকেশন

## 📌 ধারণা (Concept)

### OAuth 2.0
OAuth 2.0 হলো একটি অথরাইজেশন ফ্রেমওয়ার্ক যা থার্ড-পার্টি অ্যাপ্লিকেশনকে ইউজারের রিসোর্সে সীমিত অ্যাক্সেস দেয় — ইউজারের পাসওয়ার্ড শেয়ার না করেই। যেমন: "Google দিয়ে লগইন করুন"।

### JWT (JSON Web Token)
JWT হলো একটি কম্প্যাক্ট, self-contained টোকেন যা JSON ফরম্যাটে তথ্য বহন করে। এটি ডিজিটালি সাইন করা থাকে, তাই এর সত্যতা যাচাই করা যায়।

## 🏠 বাস্তব জীবনের উদাহরণ

### OAuth 2.0
ধরুন আপনি একটি হোটেলে গেলেন। রিসেপশন আপনাকে একটি কার্ড দিল যা দিয়ে শুধু আপনার রুম ও জিমে ঢোকা যায়, কিন্তু ম্যানেজারের অফিসে ঢোকা যায় না। OAuth ঠিক এভাবে কাজ করে — নির্দিষ্ট অনুমতি (scope) সহ অ্যাক্সেস দেওয়া।

### JWT
একটি এয়ারপোর্টের বোর্ডিং পাস — এতে আপনার নাম, ফ্লাইট নম্বর, সিট নম্বর সব লেখা আছে এবং এটি এয়ারলাইন সিল দিয়ে যাচাই করা। যে কেউ তথ্য পড়তে পারে, কিন্তু পরিবর্তন করতে পারে না।

---

## 📋 JWT এর গঠন (Structure)

```
xxxxx.yyyyy.zzzzz
  |      |      |
Header Payload Signature

Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "1234567890", "name": "রহিম", "role": "admin", "exp": 1700000000}
Signature: HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

### PHP — JWT তৈরি ও যাচাই

```php
// firebase/php-jwt লাইব্রেরি ব্যবহার
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

class JWTManager {
    private string $secretKey;
    private string $algorithm = 'HS256';

    public function __construct() {
        $this->secretKey = getenv('JWT_SECRET');
    }

    public function generateToken(array $userData): array {
        $now = time();

        // Access Token (সংক্ষিপ্ত মেয়াদ)
        $accessPayload = [
            'iss' => 'my-app',           // issuer
            'sub' => $userData['id'],     // subject (user id)
            'iat' => $now,                // issued at
            'exp' => $now + 900,          // ১৫ মিনিট পরে expire
            'role' => $userData['role'],
            'email' => $userData['email']
        ];

        // Refresh Token (দীর্ঘ মেয়াদ)
        $refreshPayload = [
            'sub' => $userData['id'],
            'iat' => $now,
            'exp' => $now + 604800, // ৭ দিন
            'type' => 'refresh'
        ];

        return [
            'access_token' => JWT::encode($accessPayload, $this->secretKey, $this->algorithm),
            'refresh_token' => JWT::encode($refreshPayload, $this->secretKey, $this->algorithm),
            'expires_in' => 900
        ];
    }

    public function verifyToken(string $token): object {
        try {
            return JWT::decode($token, new Key($this->secretKey, $this->algorithm));
        } catch (\Firebase\JWT\ExpiredException $e) {
            throw new \Exception('টোকেনের মেয়াদ শেষ হয়ে গেছে');
        } catch (\Exception $e) {
            throw new \Exception('অবৈধ টোকেন');
        }
    }

    public function refreshAccessToken(string $refreshToken): array {
        $payload = $this->verifyToken($refreshToken);
        if (($payload->type ?? '') !== 'refresh') {
            throw new \Exception('এটি রিফ্রেশ টোকেন নয়');
        }
        $user = DB::find('users', $payload->sub);
        return $this->generateToken((array) $user);
    }
}
```

### JavaScript (Node.js) — JWT তৈরি ও যাচাই

```javascript
import jwt from 'jsonwebtoken';

class JWTManager {
    #secret;

    constructor() {
        this.#secret = process.env.JWT_SECRET;
    }

    generateTokens(user) {
        const accessToken = jwt.sign(
            { sub: user.id, role: user.role, email: user.email },
            this.#secret,
            { expiresIn: '15m', issuer: 'my-app' }
        );

        const refreshToken = jwt.sign(
            { sub: user.id, type: 'refresh' },
            this.#secret,
            { expiresIn: '7d' }
        );

        return { accessToken, refreshToken, expiresIn: 900 };
    }

    verifyToken(token) {
        try {
            return jwt.verify(token, this.#secret);
        } catch (err) {
            if (err.name === 'TokenExpiredError') {
                throw new Error('টোকেনের মেয়াদ শেষ');
            }
            throw new Error('অবৈধ টোকেন');
        }
    }
}

// Express Middleware — JWT যাচাই
function authenticate(req, res, next) {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'টোকেন প্রদান করুন' });
    }

    const token = authHeader.split(' ')[1];
    try {
        const jwtManager = new JWTManager();
        req.user = jwtManager.verifyToken(token);
        next();
    } catch (err) {
        return res.status(401).json({ error: err.message });
    }
}

// ব্যবহার
app.get('/api/profile', authenticate, (req, res) => {
    res.json({ userId: req.user.sub, role: req.user.role });
});
```

---

## 🔄 OAuth 2.0 Authorization Code Flow

```
┌──────────┐     ১. লগইন বাটনে ক্লিক     ┌──────────────┐
│  ইউজার    │ ──────────────────────────▶  │  ক্লায়েন্ট    │
│ (ব্রাউজার) │                              │  অ্যাপ্লিকেশন  │
└──────────┘                              └──────┬───────┘
      │                                          │
      │  ২. Auth Server এ রিডাইরেক্ট             │
      ▼                                          │
┌──────────────┐                                 │
│ Authorization │  ৩. ইউজার অনুমতি দেয়           │
│    Server     │ ◀─────────────────────────────  │
│  (Google)     │                                 │
└──────┬───────┘                                 │
       │                                          │
       │  ৪. Authorization Code ফেরত দেয়         │
       │─────────────────────────────────────────▶│
       │                                          │
       │  ৫. Code + Client Secret দিয়ে            │
       │     Access Token নেয়                     │
       │◀─────────────────────────────────────────│
       │                                          │
       │  ৬. Access Token পাঠায়                   │
       │─────────────────────────────────────────▶│
```

### PHP — OAuth 2.0 Implementation

```php
class OAuthClient {
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;
    private string $authUrl = 'https://accounts.google.com/o/oauth2/auth';
    private string $tokenUrl = 'https://oauth2.googleapis.com/token';

    public function __construct() {
        $this->clientId = getenv('GOOGLE_CLIENT_ID');
        $this->clientSecret = getenv('GOOGLE_CLIENT_SECRET');
        $this->redirectUri = getenv('GOOGLE_REDIRECT_URI');
    }

    // ১. Auth URL তৈরি
    public function getAuthUrl(): string {
        $state = bin2hex(random_bytes(16));
        $_SESSION['oauth_state'] = $state;

        $params = http_build_query([
            'client_id' => $this->clientId,
            'redirect_uri' => $this->redirectUri,
            'response_type' => 'code',
            'scope' => 'openid email profile',
            'state' => $state,
            'access_type' => 'offline'
        ]);

        return $this->authUrl . '?' . $params;
    }

    // ২. Authorization Code দিয়ে Token নেওয়া
    public function getAccessToken(string $code, string $state): array {
        if ($state !== $_SESSION['oauth_state']) {
            throw new \Exception('CSRF আক্রমণ শনাক্ত হয়েছে');
        }

        $response = $this->httpPost($this->tokenUrl, [
            'code' => $code,
            'client_id' => $this->clientId,
            'client_secret' => $this->clientSecret,
            'redirect_uri' => $this->redirectUri,
            'grant_type' => 'authorization_code'
        ]);

        return json_decode($response, true);
    }

    private function httpPost(string $url, array $data): string {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true
        ]);
        $result = curl_exec($ch);
        curl_close($ch);
        return $result;
    }
}
```

### JavaScript — OAuth 2.0 with Passport.js

```javascript
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
    let user = await User.findOne({ googleId: profile.id });

    if (!user) {
        user = await User.create({
            googleId: profile.id,
            email: profile.emails[0].value,
            name: profile.displayName
        });
    }

    return done(null, user);
}));

// রাউটস
app.get('/auth/google',
    passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
    passport.authenticate('google', { failureRedirect: '/login' }),
    (req, res) => {
        const jwtManager = new JWTManager();
        const tokens = jwtManager.generateTokens(req.user);
        res.json(tokens);
    }
);
```

---

## ✅ কখন ব্যবহার করবেন

- **JWT:** Stateless API অথেনটিকেশন, মাইক্রোসার্ভিস আর্কিটেকচার, মোবাইল অ্যাপ
- **OAuth 2.0:** থার্ড-পার্টি লগইন (Google, Facebook, GitHub), API অ্যাক্সেস ডেলিগেশন
- **Refresh Token:** দীর্ঘমেয়াদী সেশন ম্যানেজমেন্ট — access token expire হলে নতুন নেওয়ার জন্য

## ❌ কখন ব্যবহার করবেন না

- **JWT তে সংবেদনশীল ডেটা রাখবেন না** — payload সবাই decode করতে পারে
- **JWT কে session replacement হিসেবে ব্যবহার করলে** logout কঠিন হয় — token blacklist দরকার
- **শুধু ক্লায়েন্ট-সাইড validation** যথেষ্ট নয় — সার্ভারে অবশ্যই token verify করুন
- **`alg: none`** কখনো accept করবেন না — এটি একটি পরিচিত আক্রমণ ভেক্টর
