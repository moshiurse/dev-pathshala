# 🔐 API অথেনটিকেশন (API Authentication Deep Dive)

> **লেভেল:** Advanced | **ভাষা:** PHP (Laravel) + JavaScript (Express/Node.js)  
> **প্রেক্ষাপট:** বাংলাদেশ — bKash API, SSLCommerz, বাংলাদেশি মোবাইল OTP

---

## 📌 সংজ্ঞা ও মৌলিক ধারণা

### Authentication vs Authorization

**Authentication (অথেনটিকেশন)** এবং **Authorization (অথোরাইজেশন)** দুটি সম্পূর্ণ ভিন্ন ধারণা, কিন্তু প্রায়ই এদের গুলিয়ে ফেলা হয়।

| বিষয় | Authentication (AuthN) | Authorization (AuthZ) |
|--------|----------------------|----------------------|
| প্রশ্ন | "তুমি কে?" | "তোমার কী করার অনুমতি আছে?" |
| উদ্দেশ্য | পরিচয় যাচাই | অনুমতি যাচাই |
| সময় | সবার আগে ঘটে | Authentication-এর পরে ঘটে |
| উদাহরণ | লগইন করা | Admin panel অ্যাক্সেস |
| ব্যর্থতা কোড | `401 Unauthorized` | `403 Forbidden` |

**বাস্তব উদাহরণ:** bKash অ্যাপে ফোন নম্বর + PIN দিয়ে লগইন করা হলো **Authentication**। লগইন করার পর merchant dashboard দেখার অনুমতি আছে কি না — সেটা **Authorization**।

### Stateless vs Stateful Authentication

```
Stateful (Session-based):
┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client  │──login──▶│  Server  │──store──▶│ Session  │
│          │◀─cookie──│          │         │  Store   │
│          │──req+────▶│          │──lookup─▶│ (Redis/  │
│          │  cookie  │          │         │  DB)     │
└──────────┘         └──────────┘         └──────────┘
→ সার্ভার সেশন সংরক্ষণ করে, প্রতিটি রিকোয়েস্টে lookup হয়

Stateless (Token-based):
┌──────────┐         ┌──────────┐
│  Client  │──login──▶│  Server  │  (কোনো সেশন স্টোর নেই)
│          │◀─token───│          │
│          │──req+────▶│  verify  │  (token নিজেই তথ্য বহন করে)
│          │  token   │  token   │
└──────────┘         └──────────┘
→ সার্ভার কোনো state রাখে না, token-এ সব তথ্য encoded থাকে
```

**Stateless-এর সুবিধা:** Horizontal scaling সহজ — যেকোনো সার্ভার instance token verify করতে পারে। Microservice architecture-এ এটি অপরিহার্য।

**Stateful-এর সুবিধা:** তাৎক্ষণিক revocation সম্ভব — সেশন মুছে দিলেই হলো। JWT-তে এটি সমস্যাজনক কারণ token expiry পর্যন্ত valid থাকে।

---

## 📊 Auth Flow Diagrams

### JWT Authentication Flow
```
Client                    Server                   Database
  │                         │                         │
  │── POST /login ─────────▶│                         │
  │   {email, password}     │── verify credentials ──▶│
  │                         │◀── user data ───────────│
  │                         │                         │
  │                         │── sign JWT ──┐          │
  │                         │◀─────────────┘          │
  │◀── {access_token,  ────│                         │
  │     refresh_token}      │                         │
  │                         │                         │
  │── GET /api/resource ───▶│                         │
  │   Authorization:        │── verify JWT ──┐        │
  │   Bearer <token>        │◀───────────────┘        │
  │◀── 200 {data} ─────────│                         │
  │                         │                         │
  │── POST /refresh ───────▶│                         │
  │   {refresh_token}       │── verify + rotate ─────▶│
  │◀── {new_access,  ──────│◀────────────────────────│
  │     new_refresh}        │                         │
```

### OAuth 2.0 Authorization Code Flow
```
User        Client App       Auth Server      Resource Server
 │              │                  │                  │
 │──click───▶  │                  │                  │
 │  "Login"    │──redirect───────▶│                  │
 │             │  /authorize?      │                  │
 │             │  client_id=X&     │                  │
 │             │  redirect_uri=Y&  │                  │
 │             │  scope=Z&state=S  │                  │
 │◀────────────────login form─────│                  │
 │──credentials────────────────▶  │                  │
 │◀────────────────redirect───────│                  │
 │  /callback?code=ABC&state=S    │                  │
 │──follow────▶│                  │                  │
 │             │──POST /token─────▶│                  │
 │             │  {code, secret}   │                  │
 │             │◀──{access_token}──│                  │
 │             │                   │                  │
 │             │──GET /resource────────────────────▶  │
 │             │  Authorization:   │                  │
 │             │  Bearer <token>   │                  │
 │             │◀──{data}──────────────────────────── │
```

### HMAC Request Signing Flow (bKash/SSLCommerz Webhook)
```
Sender (bKash)              Receiver (Your Server)
     │                              │
     │── Create payload ──┐        │
     │                    │        │
     │── HMAC-SHA256(     │        │
     │   payload +        │        │
     │   timestamp +      │        │
     │   secret_key) ─────┘        │
     │                              │
     │── POST /webhook ────────────▶│
     │   Headers:                   │
     │   X-Signature: <hmac>        │── Recalculate HMAC ──┐
     │   X-Timestamp: <ts>         │                      │
     │   Body: {payload}           │◀─────────────────────┘
     │                              │── Compare signatures
     │                              │── Check timestamp (±5min)
     │◀── 200 OK ──────────────────│
```

---

## 💻 Auth Methods Deep Dive

### ১. API Key Authentication

API Key হলো সবচেয়ে সরল authentication পদ্ধতি — একটি unique string যা প্রতিটি রিকোয়েস্টের সাথে পাঠানো হয়। এটি মূলত machine-to-machine communication-এ ব্যবহৃত হয়।

**কোথায় পাঠাবেন:**
- **Header (recommended):** `X-API-Key: abc123` — URL-এ expose হয় না
- **Query Parameter:** `?api_key=abc123` — server logs-এ ফাঁস হতে পারে, **এড়িয়ে চলুন**

#### PHP (Laravel) — API Key System

```php
// database/migrations/create_api_keys_table.php
Schema::create('api_keys', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('key', 64)->unique();
    $table->string('hashed_key', 128)->index();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->json('permissions')->nullable();
    $table->integer('rate_limit')->default(1000); // প্রতি ঘণ্টায়
    $table->timestamp('last_used_at')->nullable();
    $table->timestamp('expires_at')->nullable();
    $table->timestamp('rotated_at')->nullable();
    $table->timestamps();
    $table->softDeletes();
});

// app/Models/ApiKey.php
class ApiKey extends Model
{
    use SoftDeletes;

    protected $casts = [
        'permissions' => 'array',
        'expires_at' => 'datetime',
    ];

    // Key তৈরি — plain key একবারই দেখানো হবে
    public static function generate(User $user, string $name, array $permissions = []): array
    {
        $plainKey = Str::random(48);

        $apiKey = self::create([
            'name' => $name,
            'key' => substr($plainKey, 0, 8) . '...',  // prefix for identification
            'hashed_key' => hash('sha256', $plainKey),
            'user_id' => $user->id,
            'permissions' => $permissions,
            'expires_at' => now()->addYear(),
        ]);

        return ['api_key' => $apiKey, 'plain_key' => $plainKey];
    }

    // Key rotation — পুরনো key revoke করে নতুন key তৈরি
    public function rotate(): array
    {
        $newPlainKey = Str::random(48);

        $this->update([
            'key' => substr($newPlainKey, 0, 8) . '...',
            'hashed_key' => hash('sha256', $newPlainKey),
            'rotated_at' => now(),
        ]);

        return ['plain_key' => $newPlainKey];
    }

    public function hasPermission(string $permission): bool
    {
        return in_array($permission, $this->permissions ?? []);
    }

    public function isExpired(): bool
    {
        return $this->expires_at && $this->expires_at->isPast();
    }
}

// app/Http/Middleware/AuthenticateApiKey.php
class AuthenticateApiKey
{
    public function handle(Request $request, Closure $next, ?string $permission = null)
    {
        $plainKey = $request->header('X-API-Key');

        if (!$plainKey) {
            return response()->json(['error' => 'API key অনুপস্থিত'], 401);
        }

        $hashedKey = hash('sha256', $plainKey);
        $apiKey = ApiKey::where('hashed_key', $hashedKey)->first();

        if (!$apiKey || $apiKey->isExpired()) {
            return response()->json(['error' => 'অবৈধ বা মেয়াদোত্তীর্ণ API key'], 401);
        }

        if ($permission && !$apiKey->hasPermission($permission)) {
            return response()->json(['error' => 'অনুমতি নেই'], 403);
        }

        // Rate limiting — প্রতিটি key-এর জন্য আলাদা limit
        $rateLimitKey = 'api_key_rate:' . $apiKey->id;
        if (RateLimiter::tooManyAttempts($rateLimitKey, $apiKey->rate_limit)) {
            $retryAfter = RateLimiter::availableIn($rateLimitKey);
            return response()->json([
                'error' => 'Rate limit অতিক্রম করেছে',
                'retry_after' => $retryAfter
            ], 429);
        }
        RateLimiter::hit($rateLimitKey, 3600);

        $apiKey->update(['last_used_at' => now()]);
        $request->merge(['api_key' => $apiKey]);

        return $next($request);
    }
}
```

#### JavaScript (Express/Node.js) — API Key System

```javascript
// models/ApiKey.js
const crypto = require('crypto');
const mongoose = require('mongoose');

const apiKeySchema = new mongoose.Schema({
  name: String,
  prefix: String,
  hashedKey: { type: String, index: true },
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  permissions: [String],
  rateLimit: { type: Number, default: 1000 },
  expiresAt: Date,
  lastUsedAt: Date,
}, { timestamps: true });

apiKeySchema.statics.generate = async function(userId, name, permissions = []) {
  const plainKey = crypto.randomBytes(36).toString('hex');
  const hashedKey = crypto.createHash('sha256').update(plainKey).digest('hex');

  const apiKey = await this.create({
    name,
    prefix: plainKey.substring(0, 8),
    hashedKey,
    userId,
    permissions,
    expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000),
  });

  // plain key শুধুমাত্র এই একবারই return হবে
  return { apiKey, plainKey };
};

module.exports = mongoose.model('ApiKey', apiKeySchema);

// middleware/apiKeyAuth.js
const Redis = require('ioredis');
const redis = new Redis();
const ApiKey = require('../models/ApiKey');

async function apiKeyAuth(requiredPermission) {
  return async (req, res, next) => {
    const plainKey = req.headers['x-api-key'];
    if (!plainKey) return res.status(401).json({ error: 'API key অনুপস্থিত' });

    const hashedKey = crypto.createHash('sha256').update(plainKey).digest('hex');
    const apiKey = await ApiKey.findOne({ hashedKey });

    if (!apiKey || (apiKey.expiresAt && apiKey.expiresAt < new Date())) {
      return res.status(401).json({ error: 'অবৈধ বা মেয়াদোত্তীর্ণ API key' });
    }

    if (requiredPermission && !apiKey.permissions.includes(requiredPermission)) {
      return res.status(403).json({ error: 'অনুমতি নেই' });
    }

    // Redis-ভিত্তিক rate limiting
    const rateLimitKey = `rate:apikey:${apiKey._id}`;
    const current = await redis.incr(rateLimitKey);
    if (current === 1) await redis.expire(rateLimitKey, 3600);

    if (current > apiKey.rateLimit) {
      const ttl = await redis.ttl(rateLimitKey);
      return res.status(429).json({ error: 'Rate limit অতিক্রম', retry_after: ttl });
    }

    apiKey.lastUsedAt = new Date();
    await apiKey.save();
    req.apiKey = apiKey;
    next();
  };
}
```

---

### ২. JWT (JSON Web Token)

JWT হলো modern API authentication-এর মেরুদণ্ড। এটি একটি self-contained token যা নিজের মধ্যেই user information বহন করে।

#### JWT Structure

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.     ← Header (Base64)
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ik.     ← Payload (Base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQ.     ← Signature

Header:  {"alg": "RS256", "typ": "JWT"}
Payload: {"sub": "1234", "name": "রহিম", "role": "admin", "iat": 1700000, "exp": 1700900}
Signature: RS256(base64(header) + "." + base64(payload), private_key)
```

#### HS256 vs RS256

| বিষয় | HS256 (Symmetric) | RS256 (Asymmetric) |
|--------|-------------------|-------------------|
| Key | একটি shared secret | Public + Private key pair |
| Sign | secret দিয়ে | Private key দিয়ে |
| Verify | একই secret দিয়ে | Public key দিয়ে |
| ব্যবহার | Monolith | Microservices (যেকোনো service public key দিয়ে verify করতে পারে) |
| নিরাপত্তা | Secret ফাঁস হলে sign ও verify দুটোই সম্ভব | Private key ফাঁস না হলে forge অসম্ভব |

#### PHP (Laravel Sanctum + Custom JWT)

```php
// ১. Laravel Sanctum — SPA ও Mobile-এর জন্য সেরা
// config/sanctum.php
'expiration' => 60, // মিনিটে

// app/Http/Controllers/AuthController.php
class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'phone' => 'required|regex:/^01[3-9]\d{8}$/', // BD মোবাইল নম্বর
            'password' => 'required|min:6',
        ]);

        $user = User::where('phone', $request->phone)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json(['error' => 'ভুল ফোন নম্বর বা পাসওয়ার্ড'], 401);
        }

        // Access token — স্বল্পকালীন
        $accessToken = $user->createToken('access', ['*'], now()->addMinutes(15));

        // Refresh token — দীর্ঘকালীন
        $refreshToken = $user->createToken('refresh', ['token:refresh'], now()->addDays(30));

        return response()->json([
            'access_token' => $accessToken->plainTextToken,
            'refresh_token' => $refreshToken->plainTextToken,
            'token_type' => 'Bearer',
            'expires_in' => 900,
            'user' => new UserResource($user),
        ]);
    }

    public function refresh(Request $request)
    {
        $request->validate(['refresh_token' => 'required|string']);

        // Refresh token verify করুন
        $token = PersonalAccessToken::findToken($request->refresh_token);

        if (!$token || !$token->can('token:refresh') || $token->expires_at->isPast()) {
            return response()->json(['error' => 'অবৈধ refresh token'], 401);
        }

        $user = $token->tokenable;

        // Token rotation — পুরনো refresh token মুছে দিন
        $token->delete();

        $newAccess = $user->createToken('access', ['*'], now()->addMinutes(15));
        $newRefresh = $user->createToken('refresh', ['token:refresh'], now()->addDays(30));

        return response()->json([
            'access_token' => $newAccess->plainTextToken,
            'refresh_token' => $newRefresh->plainTextToken,
            'expires_in' => 900,
        ]);
    }

    public function logout(Request $request)
    {
        // বর্তমান token revoke
        $request->user()->currentAccessToken()->delete();

        // অথবা সব token revoke করতে:
        // $request->user()->tokens()->delete();

        return response()->json(['message' => 'সফলভাবে লগআউট হয়েছে']);
    }
}

// ২. Custom JWT with RS256 — Microservice-এর জন্য
// app/Services/JwtService.php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

class JwtService
{
    private string $privateKey;
    private string $publicKey;

    public function __construct()
    {
        $this->privateKey = file_get_contents(storage_path('keys/private.pem'));
        $this->publicKey = file_get_contents(storage_path('keys/public.pem'));
    }

    public function generateTokenPair(User $user): array
    {
        $now = time();

        $accessPayload = [
            'iss' => config('app.url'),
            'sub' => $user->id,
            'aud' => 'api',
            'iat' => $now,
            'exp' => $now + 900, // ১৫ মিনিট
            'jti' => Str::uuid()->toString(),
            'role' => $user->role,
            'permissions' => $user->getAllPermissions()->pluck('name'),
        ];

        $refreshPayload = [
            'iss' => config('app.url'),
            'sub' => $user->id,
            'iat' => $now,
            'exp' => $now + 2592000, // ৩০ দিন
            'jti' => Str::uuid()->toString(),
            'type' => 'refresh',
        ];

        // Refresh token-এর jti ডাটাবেসে সংরক্ষণ (revocation-এর জন্য)
        RefreshToken::create([
            'jti' => $refreshPayload['jti'],
            'user_id' => $user->id,
            'expires_at' => Carbon::createFromTimestamp($refreshPayload['exp']),
        ]);

        return [
            'access_token' => JWT::encode($accessPayload, $this->privateKey, 'RS256'),
            'refresh_token' => JWT::encode($refreshPayload, $this->privateKey, 'RS256'),
            'expires_in' => 900,
        ];
    }

    public function verifyAccessToken(string $token): object
    {
        $decoded = JWT::decode($token, new Key($this->publicKey, 'RS256'));

        // Blacklist চেক (Redis)
        if (Cache::has("jwt_blacklist:{$decoded->jti}")) {
            throw new \Exception('Token revoked');
        }

        return $decoded;
    }

    public function blacklistToken(string $jti, int $expiresAt): void
    {
        $ttl = $expiresAt - time();
        if ($ttl > 0) {
            Cache::put("jwt_blacklist:{$jti}", true, $ttl);
        }
    }
}
```

#### JavaScript (Node.js/Express) — JWT Implementation

```javascript
// services/jwtService.js
const jwt = require('jsonwebtoken');
const fs = require('fs');
const { v4: uuidv4 } = require('uuid');
const Redis = require('ioredis');
const redis = new Redis();

const PRIVATE_KEY = fs.readFileSync('./keys/private.pem', 'utf8');
const PUBLIC_KEY = fs.readFileSync('./keys/public.pem', 'utf8');

class JwtService {
  static generateTokenPair(user) {
    const accessJti = uuidv4();
    const refreshJti = uuidv4();

    const accessToken = jwt.sign(
      {
        sub: user.id,
        role: user.role,
        permissions: user.permissions,
        jti: accessJti,
      },
      PRIVATE_KEY,
      { algorithm: 'RS256', expiresIn: '15m', issuer: 'api.example.com' }
    );

    const refreshToken = jwt.sign(
      { sub: user.id, type: 'refresh', jti: refreshJti },
      PRIVATE_KEY,
      { algorithm: 'RS256', expiresIn: '30d', issuer: 'api.example.com' }
    );

    // Refresh token hash Redis-এ সংরক্ষণ
    redis.setex(`refresh:${user.id}:${refreshJti}`, 30 * 86400, 'valid');

    return { accessToken, refreshToken, expiresIn: 900 };
  }

  static async verifyAccessToken(token) {
    const decoded = jwt.verify(token, PUBLIC_KEY, {
      algorithms: ['RS256'],
      issuer: 'api.example.com',
    });

    // Blacklist চেক
    const isBlacklisted = await redis.exists(`blacklist:${decoded.jti}`);
    if (isBlacklisted) throw new Error('Token revoked');

    return decoded;
  }

  static async revokeToken(jti, exp) {
    const ttl = exp - Math.floor(Date.now() / 1000);
    if (ttl > 0) await redis.setex(`blacklist:${jti}`, ttl, '1');
  }
}

// middleware/jwtAuth.js
async function jwtAuth(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token অনুপস্থিত' });
  }

  try {
    const token = authHeader.split(' ')[1];
    req.user = await JwtService.verifyAccessToken(token);
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token মেয়াদোত্তীর্ণ', code: 'TOKEN_EXPIRED' });
    }
    return res.status(401).json({ error: 'অবৈধ token' });
  }
}

// routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcrypt');

router.post('/login', async (req, res) => {
  const { phone, password } = req.body;

  // বাংলাদেশি মোবাইল নম্বর যাচাই
  if (!/^01[3-9]\d{8}$/.test(phone)) {
    return res.status(400).json({ error: 'সঠিক বাংলাদেশি মোবাইল নম্বর দিন' });
  }

  const user = await User.findOne({ phone });
  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).json({ error: 'ভুল ফোন নম্বর বা পাসওয়ার্ড' });
  }

  const tokens = JwtService.generateTokenPair(user);
  res.json(tokens);
});

router.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  try {
    const decoded = jwt.verify(refreshToken, PUBLIC_KEY, { algorithms: ['RS256'] });

    if (decoded.type !== 'refresh') {
      return res.status(401).json({ error: 'অবৈধ token ধরন' });
    }

    // পুরনো refresh token বাতিল করুন (rotation)
    const key = `refresh:${decoded.sub}:${decoded.jti}`;
    const isValid = await redis.get(key);
    if (!isValid) return res.status(401).json({ error: 'Token ইতিমধ্যে ব্যবহৃত' });
    await redis.del(key);

    const user = await User.findById(decoded.sub);
    const tokens = JwtService.generateTokenPair(user);
    res.json(tokens);
  } catch (err) {
    res.status(401).json({ error: 'অবৈধ refresh token' });
  }
});

router.post('/logout', jwtAuth, async (req, res) => {
  await JwtService.revokeToken(req.user.jti, req.user.exp);
  res.json({ message: 'সফলভাবে লগআউট হয়েছে' });
});
```

**Token Storage নির্দেশিকা:**

| স্থান | সুবিধা | ঝুঁকি | সুপারিশ |
|--------|--------|-------|---------|
| `httpOnly` Cookie | XSS-proof | CSRF ঝুঁকি | ✅ Web app-এর জন্য সেরা |
| `localStorage` | সহজ | XSS-এ চুরি সম্ভব | ❌ এড়িয়ে চলুন |
| Memory (JS variable) | সবচেয়ে নিরাপদ | পেজ রিফ্রেশে হারায় | ✅ + silent refresh |
| Secure Cookie + CSRF | XSS + CSRF protected | জটিল | ✅ সেরা সমাধান |

---

### ৩. OAuth 2.0

OAuth 2.0 হলো industry-standard authorization framework। এটি third-party অ্যাপ্লিকেশনকে ব্যবহারকারীর তথ্যে সীমিত অ্যাক্সেস দেয়, পাসওয়ার্ড শেয়ার ছাড়াই।

#### চারটি Grant Type

**১. Authorization Code (+ PKCE) — সবচেয়ে নিরাপদ, SPA ও Mobile-এর জন্য**
```
User → App → Auth Server (/authorize)
Auth Server → User (login page)
User → Auth Server (credentials)
Auth Server → App (authorization code)
App → Auth Server (/token + code + code_verifier)
Auth Server → App (access_token)
```

**২. Client Credentials — Machine-to-Machine (bKash API integration)**
```
App → Auth Server (/token + client_id + client_secret)
Auth Server → App (access_token)
→ কোনো user involvement নেই, শুধু app-to-app communication
```

**৩. Device Code — Smart TV, IoT devices**
```
Device → Auth Server (/device/code)
Auth Server → Device (device_code + user_code + verification_uri)
Device → User: "go.bkash.com এ গিয়ে ABC123 কোড দিন"
User → Browser → Auth Server (enter code + login)
Device → Auth Server (poll /token with device_code)
Auth Server → Device (access_token)
```

**৪. PKCE (Proof Key for Code Exchange) — Public client-এর জন্য**
```
code_verifier = random(43-128 chars)
code_challenge = BASE64URL(SHA256(code_verifier))

/authorize?...&code_challenge=X&code_challenge_method=S256
/token?...&code_verifier=Y  → সার্ভার SHA256(Y) == X যাচাই করে
```

#### PHP (Laravel Passport) — OAuth 2.0 Server

```php
// Laravel Passport সেটআপ
// config/auth.php
'guards' => [
    'api' => ['driver' => 'passport', 'provider' => 'users'],
],

// app/Http/Controllers/OAuthController.php
class OAuthController extends Controller
{
    // Social Login — Google, Facebook, GitHub
    public function redirectToProvider(string $provider)
    {
        $validProviders = ['google', 'facebook', 'github'];
        if (!in_array($provider, $validProviders)) {
            return response()->json(['error' => 'অসমর্থিত প্রোভাইডার'], 400);
        }

        return Socialite::driver($provider)
            ->scopes(['email', 'profile'])
            ->stateless()
            ->redirect();
    }

    public function handleProviderCallback(string $provider)
    {
        try {
            $socialUser = Socialite::driver($provider)->stateless()->user();
        } catch (\Exception $e) {
            return response()->json(['error' => 'OAuth যাচাই ব্যর্থ'], 401);
        }

        $user = User::firstOrCreate(
            ['email' => $socialUser->getEmail()],
            [
                'name' => $socialUser->getName(),
                'provider' => $provider,
                'provider_id' => $socialUser->getId(),
                'avatar' => $socialUser->getAvatar(),
                'email_verified_at' => now(),
            ]
        );

        // JWT জেনারেট করুন
        $jwtService = app(JwtService::class);
        $tokens = $jwtService->generateTokenPair($user);

        // Frontend-এ redirect with tokens
        $frontendUrl = config('app.frontend_url');
        return redirect("{$frontendUrl}/auth/callback?" . http_build_query($tokens));
    }

    // Client Credentials — bKash API Integration
    public function getBkashToken()
    {
        $response = Http::withHeaders([
            'Content-Type' => 'application/json',
            'username' => config('bkash.username'),
            'password' => config('bkash.password'),
        ])->post(config('bkash.base_url') . '/tokenized/checkout/token/grant', [
            'app_key' => config('bkash.app_key'),
            'app_secret' => config('bkash.app_secret'),
        ]);

        if ($response->successful()) {
            // Token ক্যাশ করুন (expiry - 60 সেকেন্ড buffer)
            $data = $response->json();
            Cache::put('bkash_token', $data['id_token'], $data['expires_in'] - 60);
            return $data;
        }

        throw new \Exception('bKash token প্রাপ্তি ব্যর্থ');
    }
}
```

#### JavaScript (Express + Passport.js) — OAuth 2.0

```javascript
// config/passport.js
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const GitHubStrategy = require('passport-github2').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback',
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await User.findOne({ 'providers.google': profile.id });
      if (!user) {
        user = await User.create({
          name: profile.displayName,
          email: profile.emails[0].value,
          avatar: profile.photos[0]?.value,
          providers: { google: profile.id },
          emailVerified: true,
        });
      }
      done(null, user);
    } catch (err) {
      done(err);
    }
  }
));

passport.use(new GitHubStrategy({
    clientID: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    callbackURL: '/auth/github/callback',
    scope: ['user:email'],
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await User.findOne({ 'providers.github': profile.id });
      if (!user) {
        const email = profile.emails?.[0]?.value;
        user = await User.create({
          name: profile.displayName || profile.username,
          email,
          providers: { github: profile.id },
        });
      }
      done(null, user);
    } catch (err) {
      done(err);
    }
  }
));

// routes/oauth.js
const router = require('express').Router();

router.get('/auth/google', passport.authenticate('google', { scope: ['profile', 'email'] }));

router.get('/auth/google/callback',
  passport.authenticate('google', { session: false }),
  (req, res) => {
    const tokens = JwtService.generateTokenPair(req.user);
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?` +
      `access_token=${tokens.accessToken}&refresh_token=${tokens.refreshToken}`);
  }
);

router.get('/auth/github', passport.authenticate('github', { scope: ['user:email'] }));

router.get('/auth/github/callback',
  passport.authenticate('github', { session: false }),
  (req, res) => {
    const tokens = JwtService.generateTokenPair(req.user);
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?` +
      `access_token=${tokens.accessToken}&refresh_token=${tokens.refreshToken}`);
  }
);
```

---

### ৪. Session-based Authentication

Traditional web application-এর জন্য session-based auth এখনও প্রাসঙ্গিক, বিশেষত server-rendered পেজে।

#### PHP (Laravel Session)

```php
// app/Http/Controllers/SessionAuthController.php
class SessionAuthController extends Controller
{
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'phone' => 'required|regex:/^01[3-9]\d{8}$/',
            'password' => 'required',
        ]);

        if (!Auth::attempt($credentials, $request->boolean('remember'))) {
            return back()->withErrors(['phone' => 'ভুল তথ্য দেওয়া হয়েছে']);
        }

        $request->session()->regenerate(); // Session fixation প্রতিরোধ

        return redirect()->intended('/dashboard');
    }

    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken(); // CSRF token রিফ্রেশ
        return redirect('/');
    }
}

// CSRF সুরক্ষা — Laravel স্বয়ংক্রিয়ভাবে পরিচালনা করে
// Blade template-এ: @csrf
// AJAX-এ: X-CSRF-TOKEN header
// SPA-তে: /sanctum/csrf-cookie endpoint কল করুন
```

#### JavaScript (express-session)

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const csrf = require('csurf');

app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  name: 'sid',
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000, // ১ দিন
  },
}));

app.use(csrf());

app.post('/login', async (req, res) => {
  const { phone, password } = req.body;
  const user = await User.findOne({ phone });

  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).json({ error: 'ভুল তথ্য' });
  }

  // Session regeneration — fixation attack প্রতিরোধ
  req.session.regenerate((err) => {
    if (err) return res.status(500).json({ error: 'Session ত্রুটি' });
    req.session.userId = user._id;
    req.session.role = user.role;
    res.json({ message: 'লগইন সফল', csrfToken: req.csrfToken() });
  });
});

// CSRF token endpoint — SPA-এর জন্য
app.get('/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
```

---

### ৫. Basic Authentication

HTTP Basic Auth সবচেয়ে সহজ কিন্তু সবচেয়ে কম নিরাপদ পদ্ধতি। শুধুমাত্র internal API, development environment, বা TLS-এর উপর ব্যবহার করুন।

```
Authorization: Basic base64(username:password)
উদাহরণ: Authorization: Basic cmFoaW06MTIzNDU2  ← "rahim:123456"
```

```javascript
// Express Basic Auth Middleware
function basicAuth(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Basic ')) {
    res.setHeader('WWW-Authenticate', 'Basic realm="Internal API"');
    return res.status(401).json({ error: 'Authentication প্রয়োজন' });
  }

  const base64 = authHeader.split(' ')[1];
  const [username, password] = Buffer.from(base64, 'base64').toString().split(':');

  // timing-safe comparison ব্যবহার করুন — timing attack প্রতিরোধ
  const validUser = crypto.timingSafeEqual(
    Buffer.from(username), Buffer.from(process.env.INTERNAL_USER)
  );
  const validPass = crypto.timingSafeEqual(
    Buffer.from(password), Buffer.from(process.env.INTERNAL_PASS)
  );

  if (!validUser || !validPass) {
    return res.status(401).json({ error: 'অবৈধ credentials' });
  }
  next();
}
```

---

### ৬. HMAC Authentication (Webhook Verification)

HMAC (Hash-based Message Authentication Code) ব্যবহার করে request-এর অখণ্ডতা (integrity) এবং প্রামাণিকতা (authenticity) যাচাই করা হয়। **bKash, SSLCommerz, Stripe** — সবাই webhook-এ HMAC ব্যবহার করে।

#### PHP (Laravel) — SSLCommerz/bKash Webhook Verification

```php
// app/Http/Middleware/VerifyWebhookSignature.php
class VerifyWebhookSignature
{
    public function handle(Request $request, Closure $next, string $provider)
    {
        $secret = config("services.{$provider}.webhook_secret");

        switch ($provider) {
            case 'bkash':
                return $this->verifyBkash($request, $next, $secret);
            case 'sslcommerz':
                return $this->verifySslCommerz($request, $next, $secret);
            case 'stripe':
                return $this->verifyStripe($request, $next, $secret);
            default:
                abort(400, 'অজানা প্রোভাইডার');
        }
    }

    private function verifyBkash(Request $request, Closure $next, string $secret)
    {
        $signature = $request->header('X-Signature');
        $timestamp = $request->header('X-Timestamp');

        // Timestamp যাচাই — replay attack প্রতিরোধ (±৫ মিনিট)
        if (abs(time() - (int)$timestamp) > 300) {
            Log::warning('bKash webhook: মেয়াদোত্তীর্ণ timestamp', [
                'timestamp' => $timestamp
            ]);
            return response()->json(['error' => 'মেয়াদোত্তীর্ণ অনুরোধ'], 403);
        }

        $payload = $timestamp . '.' . $request->getContent();
        $expectedSignature = hash_hmac('sha256', $payload, $secret);

        if (!hash_equals($expectedSignature, $signature)) {
            Log::warning('bKash webhook: অবৈধ signature');
            return response()->json(['error' => 'অবৈধ signature'], 403);
        }

        return $next($request);
    }

    private function verifySslCommerz(Request $request, Closure $next, string $secret)
    {
        $data = $request->all();
        $receivedHash = $data['verify_sign'] ?? '';
        $storePasswd = $secret;

        // SSLCommerz-এর নিজস্ব verification যুক্তি
        unset($data['verify_sign'], $data['verify_key']);
        $keys = explode(',', $request->input('verify_key', ''));

        $toHash = '';
        foreach ($keys as $key) {
            $toHash .= $key . '=' . ($data[$key] ?? '') . '&';
        }
        $toHash = rtrim($toHash, '&');

        $generatedHash = md5($toHash . '&store_passwd=' . md5($storePasswd));

        if ($generatedHash !== $receivedHash) {
            return response()->json(['error' => 'অবৈধ SSLCommerz signature'], 403);
        }

        return $next($request);
    }
}

// routes/api.php — Webhook routes (CSRF বাদ দিতে হবে)
Route::post('/webhooks/bkash', [WebhookController::class, 'bkash'])
    ->middleware('verify.webhook:bkash')
    ->withoutMiddleware(['csrf']);
```

#### JavaScript (Express) — HMAC Webhook Verification

```javascript
// middleware/webhookVerify.js
const crypto = require('crypto');

function verifyWebhook(provider) {
  return (req, res, next) => {
    const secret = process.env[`${provider.toUpperCase()}_WEBHOOK_SECRET`];

    // Raw body প্রয়োজন — JSON parse-এর আগে
    const rawBody = req.rawBody || req.body;

    switch (provider) {
      case 'bkash': {
        const signature = req.headers['x-signature'];
        const timestamp = req.headers['x-timestamp'];

        // Replay attack প্রতিরোধ
        if (Math.abs(Date.now() / 1000 - parseInt(timestamp)) > 300) {
          return res.status(403).json({ error: 'মেয়াদোত্তীর্ণ অনুরোধ' });
        }

        const payload = `${timestamp}.${rawBody}`;
        const expected = crypto.createHmac('sha256', secret).update(payload).digest('hex');

        if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature))) {
          return res.status(403).json({ error: 'অবৈধ signature' });
        }
        break;
      }

      case 'stripe': {
        const signature = req.headers['stripe-signature'];
        const elements = Object.fromEntries(
          signature.split(',').map(s => s.split('='))
        );

        const payload = `${elements.t}.${rawBody}`;
        const expected = crypto.createHmac('sha256', secret).update(payload).digest('hex');

        if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(elements.v1))) {
          return res.status(403).json({ error: 'অবৈধ Stripe signature' });
        }
        break;
      }
    }

    next();
  };
}

// Express-এ raw body ক্যাপচার করুন
app.use(express.json({
  verify: (req, res, buf) => { req.rawBody = buf.toString(); }
}));

app.post('/webhooks/bkash', verifyWebhook('bkash'), bkashWebhookHandler);
app.post('/webhooks/stripe', verifyWebhook('stripe'), stripeWebhookHandler);
```

---

### ৭. mTLS (Mutual TLS) — Certificate-based Authentication

mTLS-এ client ও server উভয়েই certificate দিয়ে একে অপরকে যাচাই করে। Microservice-to-microservice communication-এ এটি সবচেয়ে নিরাপদ।

```
সাধারণ TLS:
Client ──────────────▶ Server
        "তুমি কে?"     → Server certificate দেখায়
        "OK, বিশ্বাস"  → Encrypted connection

Mutual TLS:
Client ◀─────────────▶ Server
        "তুমি কে?"     → Server certificate দেখায়
        "তুমি কে?"     ← Client certificate দেখায়
        উভয়ে যাচাই     → Encrypted connection
```

```javascript
// Node.js mTLS Server
const https = require('https');
const fs = require('fs');

const server = https.createServer({
  key: fs.readFileSync('./certs/server-key.pem'),
  cert: fs.readFileSync('./certs/server-cert.pem'),
  ca: fs.readFileSync('./certs/ca-cert.pem'),
  requestCert: true,      // Client certificate চাই
  rejectUnauthorized: true // অবৈধ cert reject করো
}, app);

// Client certificate থেকে service identity বের করুন
app.use((req, res, next) => {
  const cert = req.socket.getPeerCertificate();
  if (!cert || !cert.subject) {
    return res.status(401).json({ error: 'Client certificate অনুপস্থিত' });
  }

  req.serviceIdentity = {
    commonName: cert.subject.CN,    // e.g., "payment-service"
    org: cert.subject.O,            // e.g., "MyCompany"
    fingerprint: cert.fingerprint,
  };
  next();
});

// mTLS Client — অন্য microservice-এ কল করুন
const axios = require('axios');
const httpsAgent = new https.Agent({
  key: fs.readFileSync('./certs/client-key.pem'),
  cert: fs.readFileSync('./certs/client-cert.pem'),
  ca: fs.readFileSync('./certs/ca-cert.pem'),
});

async function callPaymentService(data) {
  return axios.post('https://payment-service:8443/process', data, { httpsAgent });
}
```

---

## 🔥 Advanced Topics

### ১. Token Refresh Strategy

#### Silent Refresh (SPA-এর জন্য)

```javascript
// frontend/authService.js
class AuthService {
  #accessToken = null;
  #refreshTimer = null;

  async login(phone, password) {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ phone, password }),
    });

    const data = await res.json();
    this.#setTokens(data);
    return data;
  }

  #setTokens({ accessToken, refreshToken, expiresIn }) {
    this.#accessToken = accessToken;

    // httpOnly cookie-তে refresh token সেট (সার্ভার থেকে Set-Cookie)
    // অথবা memory-তে রাখুন (সবচেয়ে নিরাপদ)

    // Expiry-এর ১ মিনিট আগে silent refresh schedule করুন
    clearTimeout(this.#refreshTimer);
    this.#refreshTimer = setTimeout(
      () => this.#silentRefresh(),
      (expiresIn - 60) * 1000
    );
  }

  async #silentRefresh() {
    try {
      const res = await fetch('/api/refresh', {
        method: 'POST',
        credentials: 'include', // cookie পাঠান
      });

      if (!res.ok) throw new Error('Refresh ব্যর্থ');
      const data = await res.json();
      this.#setTokens(data);
    } catch {
      this.logout();
      window.location.href = '/login';
    }
  }

  getAuthHeader() {
    return this.#accessToken ? { Authorization: `Bearer ${this.#accessToken}` } : {};
  }

  logout() {
    this.#accessToken = null;
    clearTimeout(this.#refreshTimer);
  }
}
```

#### Refresh Token Rotation

```
1st refresh:  RT_v1 → (valid) → AT_new + RT_v2,  RT_v1 বাতিল
2nd refresh:  RT_v2 → (valid) → AT_new + RT_v3,  RT_v2 বাতিল
Reuse attack: RT_v1 → (অবৈধ!) → সব token বাতিল (token family invalidation)
```

---

### ২. Token Blacklisting/Revocation (Redis)

```php
// app/Services/TokenBlacklistService.php (Laravel)
class TokenBlacklistService
{
    public function revoke(string $jti, int $expiresAt): void
    {
        $ttl = $expiresAt - time();
        if ($ttl > 0) {
            Redis::setex("jwt:blacklist:{$jti}", $ttl, '1');
        }
    }

    public function revokeAllForUser(int $userId): void
    {
        // User-এর সব refresh token বাতিল
        RefreshToken::where('user_id', $userId)->delete();

        // একটি "revoked after" timestamp সেট করুন
        Redis::set("jwt:revoked_after:{$userId}", time());
    }

    public function isRevoked(string $jti, int $userId, int $iat): bool
    {
        // Individual token চেক
        if (Redis::exists("jwt:blacklist:{$jti}")) return true;

        // "Revoke all" চেক — iat যদি revocation-এর আগে হয়
        $revokedAfter = Redis::get("jwt:revoked_after:{$userId}");
        return $revokedAfter && $iat < (int)$revokedAfter;
    }
}
```

---

### ৩. Multi-factor Authentication (MFA)

#### TOTP (Time-based One-Time Password)

```php
// Laravel — TOTP with Google Authenticator
use PragmaRX\Google2FA\Google2FA;

class MfaController extends Controller
{
    public function enableMfa(Request $request)
    {
        $google2fa = new Google2FA();
        $secret = $google2fa->generateSecretKey();

        $request->user()->update(['mfa_secret' => encrypt($secret)]);

        $qrCodeUrl = $google2fa->getQRCodeUrl(
            config('app.name'),
            $request->user()->email,
            $secret
        );

        return response()->json([
            'secret' => $secret,
            'qr_url' => $qrCodeUrl,
        ]);
    }

    public function verifyMfa(Request $request)
    {
        $request->validate(['otp' => 'required|digits:6']);

        $google2fa = new Google2FA();
        $secret = decrypt($request->user()->mfa_secret);

        if (!$google2fa->verifyKey($secret, $request->otp)) {
            return response()->json(['error' => 'অবৈধ OTP'], 401);
        }

        // MFA সফল — full access token জেনারেট করুন
        $request->user()->update(['mfa_verified_at' => now()]);

        return response()->json(['message' => 'MFA সফল']);
    }
}
```

#### SMS OTP (বাংলাদেশি মোবাইল নম্বর)

```javascript
// services/otpService.js — BD SMS Provider Integration
const Redis = require('ioredis');
const redis = new Redis();
const crypto = require('crypto');

class OtpService {
  static async send(phone) {
    // বাংলাদেশি নম্বর যাচাই
    if (!/^(?:\+880|880|0)1[3-9]\d{8}$/.test(phone)) {
      throw new Error('সঠিক বাংলাদেশি মোবাইল নম্বর দিন');
    }

    // নম্বর normalize করুন
    const normalized = phone.replace(/^(?:\+880|880|0)/, '0');

    // Rate limit — ৬০ সেকেন্ডে ১টি OTP
    const cooldownKey = `otp:cooldown:${normalized}`;
    if (await redis.exists(cooldownKey)) {
      const ttl = await redis.ttl(cooldownKey);
      throw new Error(`${ttl} সেকেন্ড পর আবার চেষ্টা করুন`);
    }

    // দৈনিক limit — ৫টি OTP
    const dailyKey = `otp:daily:${normalized}`;
    const dailyCount = await redis.incr(dailyKey);
    if (dailyCount === 1) await redis.expire(dailyKey, 86400);
    if (dailyCount > 5) throw new Error('দৈনিক OTP সীমা অতিক্রম');

    const otp = crypto.randomInt(100000, 999999).toString();
    const hashedOtp = crypto.createHash('sha256').update(otp).digest('hex');

    // ৫ মিনিট মেয়াদ
    await redis.setex(`otp:${normalized}`, 300, hashedOtp);
    await redis.setex(cooldownKey, 60, '1');

    // BD SMS Gateway-এ পাঠান (উদাহরণ: SSLWireless/BulkSMS BD)
    await sendSms(normalized, `আপনার OTP: ${otp}। ৫ মিনিটের মধ্যে ব্যবহার করুন।`);

    return { message: 'OTP পাঠানো হয়েছে', expiresIn: 300 };
  }

  static async verify(phone, otp) {
    const normalized = phone.replace(/^(?:\+880|880|0)/, '0');
    const key = `otp:${normalized}`;

    const storedHash = await redis.get(key);
    if (!storedHash) throw new Error('OTP মেয়াদোত্তীর্ণ বা পাওয়া যায়নি');

    const otpHash = crypto.createHash('sha256').update(otp).digest('hex');

    if (!crypto.timingSafeEqual(Buffer.from(storedHash), Buffer.from(otpHash))) {
      // Brute force প্রতিরোধ — ৩ বার ভুল হলে OTP বাতিল
      const attemptKey = `otp:attempts:${normalized}`;
      const attempts = await redis.incr(attemptKey);
      await redis.expire(attemptKey, 300);

      if (attempts >= 3) {
        await redis.del(key, attemptKey);
        throw new Error('অনেকবার ভুল। নতুন OTP নিন।');
      }
      throw new Error(`ভুল OTP। ${3 - attempts} বার সুযোগ আছে।`);
    }

    await redis.del(key, `otp:attempts:${normalized}`);
    return true;
  }
}
```

---

### ৪. API Gateway Authentication (Centralized)

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Auth + Rate   │
Client ────────────▶│   Limiting)     │
                    │                 │
                    └──────┬──────────┘
                           │ (verified request + user context)
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ User     │ │ Payment  │ │ Order    │
        │ Service  │ │ Service  │ │ Service  │
        └──────────┘ └──────────┘ └──────────┘
        (কোনো service আলাদাভাবে auth করে না)
```

```javascript
// gateway/authMiddleware.js
const axios = require('axios');

async function gatewayAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Token অনুপস্থিত' });

  try {
    // কেন্দ্রীয়ভাবে token verify
    const decoded = await JwtService.verifyAccessToken(token);

    // Downstream service-এ user context forward করুন
    req.headers['x-user-id'] = decoded.sub;
    req.headers['x-user-role'] = decoded.role;
    req.headers['x-user-permissions'] = JSON.stringify(decoded.permissions);

    // Internal request signature — downstream service যাচাই করবে
    const internalSecret = process.env.INTERNAL_SECRET;
    const timestamp = Date.now().toString();
    const hmac = crypto.createHmac('sha256', internalSecret)
      .update(`${timestamp}.${decoded.sub}`)
      .digest('hex');

    req.headers['x-internal-timestamp'] = timestamp;
    req.headers['x-internal-signature'] = hmac;

    next();
  } catch (err) {
    res.status(401).json({ error: 'অবৈধ token' });
  }
}
```

---

### ৫. RBAC (Role-Based Access Control)

```php
// app/Models/Role.php (Laravel)
class Role extends Model
{
    public function permissions()
    {
        return $this->belongsToMany(Permission::class);
    }
}

// app/Models/User.php
class User extends Authenticatable
{
    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }

    public function hasRole(string $role): bool
    {
        return $this->roles->contains('name', $role);
    }

    public function hasPermission(string $permission): bool
    {
        return $this->roles->flatMap->permissions->contains('name', $permission);
    }
}

// app/Http/Middleware/CheckPermission.php
class CheckPermission
{
    public function handle(Request $request, Closure $next, string $permission)
    {
        if (!$request->user()->hasPermission($permission)) {
            return response()->json([
                'error' => 'অনুমতি নেই',
                'required_permission' => $permission,
            ], 403);
        }
        return $next($request);
    }
}

// routes/api.php
Route::middleware(['auth:sanctum', 'permission:manage-payments'])->group(function () {
    Route::post('/payments/refund', [PaymentController::class, 'refund']);
});

Route::middleware(['auth:sanctum', 'permission:view-reports'])->group(function () {
    Route::get('/reports/sales', [ReportController::class, 'sales']);
});
```

```javascript
// middleware/rbac.js (Express)
function requirePermission(...permissions) {
  return (req, res, next) => {
    const userPermissions = req.user.permissions || [];
    const hasAll = permissions.every(p => userPermissions.includes(p));

    if (!hasAll) {
      return res.status(403).json({
        error: 'অনুমতি নেই',
        required: permissions,
        your_permissions: userPermissions,
      });
    }
    next();
  };
}

// ব্যবহার
app.delete('/api/users/:id', jwtAuth, requirePermission('users:delete'), deleteUser);
app.get('/api/reports', jwtAuth, requirePermission('reports:view'), getReports);
```

---

### ৬. ABAC (Attribute-Based Access Control)

RBAC-এর চেয়ে বেশি granular — user, resource, এবং environment attributes-এর উপর ভিত্তি করে সিদ্ধান্ত।

```javascript
// policies/abac.js
class AbacEngine {
  constructor() {
    this.policies = [];
  }

  addPolicy(policy) {
    this.policies.push(policy);
  }

  evaluate(subject, action, resource, environment = {}) {
    for (const policy of this.policies) {
      const result = policy({ subject, action, resource, environment });
      if (result === 'deny') return false;
      if (result === 'allow') return true;
    }
    return false; // default deny
  }
}

const abac = new AbacEngine();

// নীতি: শুধুমাত্র নিজের ডেটা সম্পাদনা করা যায়
abac.addPolicy(({ subject, action, resource }) => {
  if (action === 'edit' && resource.ownerId !== subject.id) return 'deny';
});

// নীতি: অফিস সময়ে (BD সময় ৯AM-৬PM) শুধুমাত্র financial data অ্যাক্সেস
abac.addPolicy(({ action, resource, environment }) => {
  if (resource.type === 'financial') {
    const bdHour = new Date(environment.timestamp)
      .toLocaleString('en', { timeZone: 'Asia/Dhaka', hour: 'numeric', hour12: false });
    if (bdHour < 9 || bdHour >= 18) return 'deny';
  }
});

// নীতি: Admin সবকিছু করতে পারে
abac.addPolicy(({ subject }) => {
  if (subject.role === 'super_admin') return 'allow';
});

// Middleware
function abacMiddleware(action, getResource) {
  return async (req, res, next) => {
    const resource = await getResource(req);
    const allowed = abac.evaluate(req.user, action, resource, {
      timestamp: Date.now(),
      ip: req.ip,
    });

    if (!allowed) return res.status(403).json({ error: 'ABAC নীতি অনুসারে অনুমতি নেই' });
    next();
  };
}
```

---

### ৭. JWT Security — হুমকি ও প্রতিরক্ষা

| হুমকি | বিবরণ | প্রতিরক্ষা |
|--------|--------|-----------|
| **Token Theft (XSS)** | XSS দিয়ে localStorage থেকে token চুরি | `httpOnly` cookie-তে রাখুন, CSP header ব্যবহার করুন |
| **CSRF** | Cookie-ভিত্তিক token-এ CSRF attack | `SameSite=Strict`, CSRF token, Double Submit Cookie |
| **Token Replay** | চুরি করা token পুনরায় ব্যবহার | ছোট expiry (১৫ মিনিট), token binding (IP/fingerprint) |
| **Algorithm Confusion** | `alg: "none"` বা HS256↔RS256 | Algorithm whitelist, library-তে explicitly algorithm সেট করুন |
| **JWK Injection** | Header-এ malicious JWK | Server-side key ব্যবহার করুন, header-এর key বিশ্বাস করবেন না |
| **Secret Brute Force** | দুর্বল HS256 secret | ≥256-bit random secret অথবা RS256 ব্যবহার করুন |

```javascript
// নিরাপদ JWT verification — সব সুরক্ষা একসাথে
function secureVerify(token) {
  return jwt.verify(token, PUBLIC_KEY, {
    algorithms: ['RS256'],          // শুধুমাত্র RS256 গ্রহণ করুন
    issuer: 'api.example.com',      // issuer যাচাই
    audience: 'web-app',            // audience যাচাই
    clockTolerance: 30,             // ৩০ সেকেন্ড tolerance
    maxAge: '15m',                  // সর্বোচ্চ age
  });
}
```

---

### ৮. OAuth2 Security

#### PKCE Implementation

```javascript
// Frontend — PKCE flow
const crypto = require('crypto');

function generatePKCE() {
  const verifier = crypto.randomBytes(32)
    .toString('base64url')
    .substring(0, 128);

  const challenge = crypto.createHash('sha256')
    .update(verifier)
    .digest('base64url');

  return { verifier, challenge };
}

// Authorization request
const { verifier, challenge } = generatePKCE();
sessionStorage.setItem('pkce_verifier', verifier);

const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', REDIRECT_URI);
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('scope', 'openid profile');
authUrl.searchParams.set('state', crypto.randomBytes(16).toString('hex'));
authUrl.searchParams.set('code_challenge', challenge);
authUrl.searchParams.set('code_challenge_method', 'S256');

window.location.href = authUrl.toString();

// Callback-এ token exchange
async function handleCallback(code) {
  const verifier = sessionStorage.getItem('pkce_verifier');
  sessionStorage.removeItem('pkce_verifier');

  const res = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: REDIRECT_URI,
      client_id: CLIENT_ID,
      code_verifier: verifier,
    }),
  });

  return res.json();
}
```

#### State Parameter — CSRF প্রতিরোধ

```javascript
// state parameter তৈরি ও যাচাই
function generateState() {
  const state = crypto.randomBytes(32).toString('hex');
  sessionStorage.setItem('oauth_state', state);
  return state;
}

function verifyState(receivedState) {
  const savedState = sessionStorage.getItem('oauth_state');
  sessionStorage.removeItem('oauth_state');

  if (!savedState || savedState !== receivedState) {
    throw new Error('State mismatch — সম্ভাব্য CSRF attack!');
  }
}
```

---

### ৯. Rate Limiting per Auth Method

```javascript
// middleware/rateLimiter.js — Auth method অনুসারে ভিন্ন rate limit
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

const authRateLimits = {
  apiKey: (req) => ({
    windowMs: 60 * 60 * 1000,
    max: req.apiKey?.rateLimit || 1000,
    keyGenerator: () => `apikey:${req.apiKey?._id}`,
  }),

  jwt: () => ({
    windowMs: 15 * 60 * 1000,
    max: 500,
    keyGenerator: (req) => `jwt:${req.user?.sub}`,
  }),

  basic: () => ({
    windowMs: 60 * 60 * 1000,
    max: 100,
    keyGenerator: (req) => `basic:${req.ip}`,
  }),

  // Login endpoint — brute force প্রতিরোধ
  login: () => ({
    windowMs: 15 * 60 * 1000,
    max: 5,
    keyGenerator: (req) => `login:${req.body.phone || req.ip}`,
    handler: (req, res) => {
      res.status(429).json({
        error: 'অনেক বেশি লগইন প্রচেষ্টা। ১৫ মিনিট পর চেষ্টা করুন।',
      });
    },
  }),

  // OTP endpoint — আরও কঠোর
  otp: () => ({
    windowMs: 60 * 1000,
    max: 1,
    keyGenerator: (req) => `otp:${req.body.phone}`,
  }),
};

function createLimiter(type) {
  return (req, res, next) => {
    const config = authRateLimits[type](req);
    const limiter = rateLimit({
      store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
      ...config,
      standardHeaders: true,
      legacyHeaders: false,
    });
    limiter(req, res, next);
  };
}

app.post('/login', createLimiter('login'), loginHandler);
app.post('/otp/send', createLimiter('otp'), otpHandler);
app.use('/api', jwtAuth, createLimiter('jwt'));
```

---

## 🆚 Auth Method Comparison Table

| মানদণ্ড | API Key | JWT | OAuth 2.0 | Session | Basic Auth | HMAC | mTLS |
|---------|---------|-----|-----------|---------|-----------|------|------|
| **জটিলতা** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **নিরাপত্তা** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Stateless** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Scalability** | উচ্চ | উচ্চ | উচ্চ | মধ্যম | উচ্চ | উচ্চ | উচ্চ |
| **Revocation** | সহজ | কঠিন | সহজ | সহজ | N/A | N/A | Certificate revoke |
| **Third-party** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **User context** | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Mobile-friendly** | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Microservice** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ব্যবহার** | Public API | SPA, Mobile | Social Login | Web App | Internal | Webhook | Service Mesh |
| **BD উদাহরণ** | বাংলাদেশ Bank API | bKash App | Google Login | Prothom Alo | Staging API | SSLCommerz IPN | Bank-to-Bank |
| **Token Storage** | Server DB | Client-side | Server + Client | Server (Redis) | N/A | N/A | Certificate Store |

---

## ✅ Best Practices

### ১. সাধারণ নিয়ম
- **HTTPS সর্বদা** — কোনো exception নেই। Let's Encrypt বিনামূল্যে certificate দেয়
- **Secrets `.env`-তে** — কখনো codebase-এ hardcode করবেন না
- **Principle of Least Privilege** — ন্যূনতম অনুমতি দিন
- **Defense in Depth** — একাধিক স্তরে সুরক্ষা রাখুন

### ২. Token নিয়ম
- Access token: **১৫ মিনিট** বা কম
- Refresh token: **৩০ দিন**, rotation সহ
- API Key: **১ বছর**, নিয়মিত rotation
- **`httpOnly`, `Secure`, `SameSite=Strict`** cookie flag ব্যবহার করুন

### ৩. Password ও Secret
- Password hashing: **bcrypt** (cost ১২+) অথবা **Argon2id**
- API secret: কমপক্ষে **256-bit** random
- **timing-safe comparison** সর্বদা ব্যবহার করুন

### ৪. Rate Limiting
- Login: **৫ attempts / ১৫ মিনিট**
- OTP: **১ / মিনিট**, **৫ / দিন**
- API: ব্যবহারকারী/key অনুসারে ভিন্ন limit

### ৫. Logging ও Monitoring
- সব authentication event log করুন (সফল + ব্যর্থ)
- অস্বাভাবিক pattern detect করুন (ভিন্ন IP থেকে login, অনেক ব্যর্থ প্রচেষ্টা)
- **কখনো token বা password log করবেন না**

### ৬. বাংলাদেশ-নির্দিষ্ট
- মোবাইল নম্বর validation: `^01[3-9]\d{8}$`
- OTP-তে Bangla SMS: `"আপনার কোড: ১২৩৪৫৬"`
- bKash/Nagad integration-এ Client Credentials flow ব্যবহার করুন
- SSLCommerz IPN-এ HMAC verify অবশ্যই করুন

---

## ⚠️ Common Vulnerabilities

### ১. Broken Authentication (OWASP Top 10)
```
❌ ভুল: JWT secret হার্ডকোড
const token = jwt.sign(data, 'my-secret-123');

✅ সঠিক: Environment variable থেকে শক্তিশালী secret
const token = jwt.sign(data, process.env.JWT_SECRET); // ≥256-bit random
```

### ২. Insecure Token Storage
```
❌ ভুল: localStorage-এ sensitive token
localStorage.setItem('access_token', token);  // XSS-এ চুরি হবে

✅ সঠিক: httpOnly cookie
res.cookie('token', token, { httpOnly: true, secure: true, sameSite: 'strict' });
```

### ৩. Missing Rate Limiting
```
❌ ভুল: কোনো rate limit নেই
app.post('/login', loginHandler);  // Brute force সম্ভব

✅ সঠিক: Rate limit + account lockout
app.post('/login', rateLimit({ max: 5, windowMs: 900000 }), loginHandler);
```

### ৪. JWT Algorithm Confusion
```
❌ ভুল: যেকোনো algorithm গ্রহণ
jwt.verify(token, secret);  // "alg: none" attack সম্ভব

✅ সঠিক: Algorithm whitelist
jwt.verify(token, key, { algorithms: ['RS256'] });
```

### ৫. CSRF Token অনুপস্থিত
```
❌ ভুল: Cookie-ভিত্তিক auth-এ CSRF protection নেই

✅ সঠিক: SameSite + CSRF token + Origin header যাচাই
```

### ৬. Timing Attack
```
❌ ভুল: সাধারণ comparison
if (providedKey === storedKey) { ... }

✅ সঠিক: Constant-time comparison
if (crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))) { ... }
```

---

## 📋 সারসংক্ষেপ

### কখন কী ব্যবহার করবেন — সিদ্ধান্ত গাইড

```
তোমার API কোন ধরনের?
│
├── Public API (third-party developers)?
│   └── OAuth 2.0 + API Key (quota tracking)
│
├── SPA / Mobile App?
│   └── JWT (access + refresh) with httpOnly cookies
│       └── Social login দরকার? → OAuth 2.0 (PKCE)
│
├── Server-to-Server / Microservice?
│   ├── Internal? → mTLS
│   └── External? → Client Credentials (OAuth 2.0)
│
├── Webhook?
│   └── HMAC Signature Verification
│
├── Traditional Web App (SSR)?
│   └── Session-based + CSRF token
│
└── Internal/Development API?
    └── Basic Auth (over HTTPS only)
```

### বাংলাদেশ প্রেক্ষাপটে Auth Stack

| সেবা | Auth পদ্ধতি |
|------|------------|
| **bKash API Integration** | Client Credentials → Token → HMAC Webhook |
| **SSLCommerz Payment** | API Key + HMAC IPN Verification |
| **বাংলাদেশি মোবাইল OTP** | SMS OTP (01X pattern) + Rate Limiting |
| **সরকারি NID Verification** | mTLS + API Key |
| **E-commerce App** | JWT (Phone+OTP Login) + OAuth (Google/Facebook) |
| **Banking API** | mTLS + HMAC + MFA (TOTP) |

### মূল শিক্ষা

1. **Authentication ≠ Authorization** — প্রথমে পরিচয় যাচাই, তারপর অনুমতি যাচাই
2. **Stateless (JWT) বনাম Stateful (Session)** — use case অনুসারে বেছে নিন
3. **Defense in Depth** — একটি পদ্ধতিতে নির্ভর করবেন না
4. **Token Rotation** — refresh token অবশ্যই rotate করুন
5. **HMAC Verification** — সব webhook-এ signature যাচাই করুন
6. **Rate Limiting** — প্রতিটি endpoint-এ যুক্তিসঙ্গত limit রাখুন
7. **Bangladesh Context** — মোবাইল-কেন্দ্রিক auth (OTP), bKash/SSLCommerz integration জানুন

> **"নিরাপত্তা কোনো feature নয় — এটি একটি অবিচ্ছিন্ন প্রক্রিয়া।"**

---

*এই ডকুমেন্টের কোড উদাহরণগুলো production-ready ভিত্তি হিসেবে ব্যবহার করা যায়, তবে আপনার নির্দিষ্ট ব্যবহারক্ষেত্র অনুসারে security audit এবং কাস্টমাইজেশন অবশ্যই করবেন।*
