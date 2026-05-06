# ⚖️ Stateless vs Stateful Service Design

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Stateless vs Stateful তুলনা](#stateless-vs-stateful-তুলনা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [Session Management](#session-management)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [12-Factor App Principles](#12-factor-app-principles)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

### Stateless Service 🔄
**Stateless Service** হলো এমন একটি সার্ভিস যা কোনো request-এর মধ্যে কোনো client-specific ডেটা সংরক্ষণ করে না। প্রতিটি request স্বাধীন এবং সম্পূর্ণ — সার্ভারের কাছে পূর্ববর্তী request-এর কোনো স্মৃতি থাকে না।

### Stateful Service 💾
**Stateful Service** হলো এমন একটি সার্ভিস যা request-এর মধ্যে client-specific state (ডেটা/session) মনে রাখে। সার্ভার জানে ক্লায়েন্ট আগে কী করেছিল।

### 🍽️ সহজ উদাহরণ:

**Stateless (ফাস্ট ফুডের কাউন্টার):**
```
গ্রাহক: "একটি বার্গার দিন" → অর্ডার সম্পন্ন
গ্রাহক: "একটি কোক দিন"   → নতুন অর্ডার (আগেরটা মনে নেই)
প্রতিটি অর্ডার স্বাধীন — যেকোনো কাউন্টারে যাওয়া যায়
```

**Stateful (রেস্টুরেন্টের ওয়েটার):**
```
ওয়েটার: "আপনি table 5-এ বসেছেন, আগে পানি চেয়েছিলেন"
ওয়েটার মনে রাখে আপনার পুরো visit-এর context
→ শুধু ঐ নির্দিষ্ট ওয়েটারই আপনাকে serve করতে পারবে
```

---

## 📊 Stateless vs Stateful তুলনা

```
┌────────────────────────────────────────────────────────────────┐
│            STATELESS vs STATEFUL COMPARISON                      │
├──────────────────────────┬─────────────────────────────────────┤
│       STATELESS          │           STATEFUL                   │
├──────────────────────────┼─────────────────────────────────────┤
│ ✅ Easy horizontal       │ ❌ Horizontal scaling কঠিন          │
│    scaling               │                                      │
│ ✅ কোনো instance-এ      │ ❌ নির্দিষ্ট instance-এ যেতে হবে    │
│    request পাঠানো যায়    │    (sticky session)                  │
│ ✅ Instance fail করলে    │ ❌ Instance fail = state হারানো      │
│    কোনো data হারায় না   │                                      │
│ ✅ Load balancing সহজ    │ ❌ Load balancing জটিল               │
│ ❌ প্রতিটি request-এ     │ ✅ দ্রুত response (local state)      │
│    auth check দরকার     │                                      │
│ ❌ External store দরকার  │ ✅ External dependency কম            │
│    (Redis, DB)           │                                      │
└──────────────────────────┴─────────────────────────────────────┘
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 💳 bKash Transaction Service (Stateless)

bKash-এর মতো সার্ভিস যেখানে প্রতি সেকেন্ডে হাজার হাজার transaction হয়, stateless design অত্যাবশ্যক:

```
ঈদের দিন bKash-এ:
- প্রতি সেকেন্ডে ১০,০০০+ transaction
- সাধারণ দিনের চেয়ে ১০ গুণ বেশি load
- সমাধান: আরো server instance যোগ করা (horizontal scaling)

Stateless হওয়ায়:
- যেকোনো server instance-এ request পাঠানো যায়
- Load balancer round-robin করতে পারে
- নতুন instance যোগ করলেই capacity বাড়ে
- কোনো instance down হলে অন্যটা কাজ করবে
```

### 🎮 Online Gaming Server (Stateful)

ধরুন একটি multiplayer game server (যেমন PUBG Mobile Bangladesh server):

```
Game Server (Stateful):
- প্রতিটি player-এর position মেমোরিতে রাখা হয়
- Game state প্রতি ফ্রেমে update হয়
- Player disconnect হলেও state কিছুক্ষণ ধরে রাখা হয়
- নির্দিষ্ট server-এই reconnect করতে হয়

কেন Stateful?
- Real-time performance (microsecond latency)
- প্রতিটি request-এ DB query করলে game lag করবে
- Game state বারবার serialize/deserialize করা expensive
```

### 🚗 Pathao Ride Service

```
Stateless অংশ:
├── Payment Processing → যেকোনো server-এ process হবে
├── Ride History API → DB থেকে পড়ে, state রাখে না
└── Notification Service → fire-and-forget

Stateful অংশ:
├── Rider Location Tracking → WebSocket connection
├── Real-time Ride Matching → in-memory proximity index
└── Surge Pricing Calculator → time-window aggregation
```

### 📰 Prothom Alo API

```
Stateless:
- Article API (GET /articles/:id)
- Search API
- Comment posting

Stateful:
- WebSocket for live score updates (ক্রিকেট)
- Online users count
- Editor collaboration (simultaneous editing)
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Stateless Architecture (Horizontal Scaling):
```
┌─────────────────────────────────────────────────────────────┐
│               STATELESS ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐                                              │
│   │  Client  │                                              │
│   └────┬─────┘                                              │
│        │  (JWT Token সহ প্রতিটি request)                    │
│        ▼                                                     │
│   ┌─────────────────────┐                                   │
│   │   Load Balancer     │ (Round Robin / Least Connection)  │
│   └──┬─────┬─────┬─────┘                                   │
│      │     │     │                                          │
│      ▼     ▼     ▼                                          │
│   ┌─────┐┌─────┐┌─────┐   ← যেকোনো instance-এ            │
│   │App 1││App 2││App 3│     যেকোনো request যেতে পারে       │
│   └──┬──┘└──┬──┘└──┬──┘                                    │
│      │      │      │                                        │
│      └──────┼──────┘                                        │
│             │                                                │
│             ▼                                                │
│   ┌───────────────────┐    ┌──────────────┐                │
│   │   Database        │    │    Redis     │                │
│   │  (PostgreSQL)     │    │  (Session/   │                │
│   │                   │    │   Cache)     │                │
│   └───────────────────┘    └──────────────┘                │
│                                                              │
│   💡 ঈদের সময়: App 4, App 5, App 6 যোগ করলেই হবে!         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Stateful Architecture (Sticky Sessions):
```
┌─────────────────────────────────────────────────────────────┐
│               STATEFUL ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌────────┐  ┌────────┐  ┌────────┐                       │
│   │Client A│  │Client B│  │Client C│                       │
│   └───┬────┘  └───┬────┘  └───┬────┘                       │
│       │            │            │                            │
│       ▼            ▼            ▼                            │
│   ┌─────────────────────────────────────┐                   │
│   │         Load Balancer               │                   │
│   │    (Sticky Session / IP Hash)       │                   │
│   └────┬───────────┬───────────┬────────┘                   │
│        │           │           │                             │
│   ┌────▼────┐ ┌────▼────┐ ┌───▼─────┐                      │
│   │ App 1   │ │ App 2   │ │ App 3   │                      │
│   │┌───────┐│ │┌───────┐│ │┌───────┐│                      │
│   ││State A││ ││State B││ ││State C││                      │
│   │└───────┘│ │└───────┘│ │└───────┘│                      │
│   └─────────┘ └─────────┘ └─────────┘                      │
│                                                              │
│   ⚠️ App 2 crash করলে Client B-এর state হারাবে!            │
│   ⚠️ নতুন instance যোগ করা সহজ নয়                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Hybrid Approach (External State):
```
┌─────────────────────────────────────────────────────────────┐
│            HYBRID: STATELESS APP + EXTERNAL STATE            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐    JWT Token                                 │
│   │  Client  │────────────────┐                             │
│   └──────────┘                │                             │
│                               ▼                              │
│                    ┌───────────────────┐                    │
│                    │   Load Balancer   │                    │
│                    └─────┬──────┬──────┘                    │
│                          │      │                           │
│              ┌───────────┘      └───────────┐              │
│              ▼                              ▼               │
│       ┌──────────┐                  ┌──────────┐           │
│       │  App 1   │                  │  App 2   │           │
│       │(Stateless│                  │(Stateless│           │
│       │ কোনো    │                  │ কোনো    │           │
│       │ local   │                  │ local   │           │
│       │ state   │                  │ state   │           │
│       │ নেই)    │                  │ নেই)    │           │
│       └────┬─────┘                  └────┬─────┘           │
│            │                             │                  │
│            └──────────┬──────────────────┘                  │
│                       │                                     │
│            ┌──────────▼──────────┐                         │
│            │    Redis Cluster    │                         │
│            │  (External State)   │                         │
│            │                     │                         │
│            │ Session: {userId,   │                         │
│            │  cart, preferences} │                         │
│            └─────────────────────┘                         │
│                                                              │
│   ✅ App stateless → easy scaling                           │
│   ✅ State centralized → no data loss                       │
│   ✅ যেকোনো App instance state access করতে পারে            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔐 Session Management

### JWT (Stateless) vs Server Session (Stateful):

```
┌─────────────── JWT (Stateless) ──────────────┐
│                                               │
│  Client                    Server             │
│    │                         │                │
│    │── Login Request ───────▶│                │
│    │                         │─┐ JWT তৈরি    │
│    │◀── JWT Token ───────────│─┘              │
│    │                         │                │
│    │── Request + JWT ───────▶│                │
│    │                         │─┐ JWT verify   │
│    │◀── Response ────────────│─┘ (DB call     │
│    │                         │    নেই!)       │
│                                               │
│  ✅ Server-এ কিছু store করতে হয় না           │
│  ✅ যেকোনো server verify করতে পারে           │
│  ❌ Token revoke করা কঠিন                    │
│  ❌ Token size বড় হতে পারে                   │
└───────────────────────────────────────────────┘

┌────────── Server Session (Stateful) ─────────┐
│                                               │
│  Client                    Server             │
│    │                         │                │
│    │── Login Request ───────▶│                │
│    │                         │─┐ Session      │
│    │◀── Session Cookie ──────│─┘ তৈরি ও      │
│    │                         │   store        │
│    │── Request + Cookie ────▶│                │
│    │                         │─┐ Session      │
│    │◀── Response ────────────│─┘ lookup       │
│    │                         │   (DB/Redis)   │
│                                               │
│  ✅ সহজে revoke/invalidate করা যায়           │
│  ✅ Server-side control বেশি                  │
│  ❌ Server-এ state store করতে হয়             │
│  ❌ Scaling-এ সমস্যা (sticky session)        │
└───────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Stateless API with JWT:

```php
<?php

/**
 * Stateless Transaction Service
 * bKash-এর মতো high-throughput stateless service
 */

// JWT Helper
class JWTService
{
    private string $secretKey;
    private string $algorithm = 'HS256';

    public function __construct(string $secretKey)
    {
        $this->secretKey = $secretKey;
    }

    public function generateToken(array $payload, int $expiresIn = 3600): string
    {
        $header = base64url_encode(json_encode([
            'alg' => $this->algorithm,
            'typ' => 'JWT',
        ]));

        $payload['iat'] = time();
        $payload['exp'] = time() + $expiresIn;
        $payloadEncoded = base64url_encode(json_encode($payload));

        $signature = hash_hmac('sha256', "$header.$payloadEncoded", $this->secretKey, true);
        $signatureEncoded = base64url_encode($signature);

        return "$header.$payloadEncoded.$signatureEncoded";
    }

    public function verifyToken(string $token): ?array
    {
        $parts = explode('.', $token);
        if (count($parts) !== 3) return null;

        [$header, $payload, $signature] = $parts;

        $expectedSignature = base64url_encode(
            hash_hmac('sha256', "$header.$payload", $this->secretKey, true)
        );

        if (!hash_equals($expectedSignature, $signature)) {
            return null;
        }

        $data = json_decode(base64url_decode($payload), true);

        if ($data['exp'] < time()) {
            return null; // Token expired
        }

        return $data;
    }
}

/**
 * Stateless Transaction Handler
 * প্রতিটি request স্বাধীন - কোনো server-side session নেই
 */
class StatelessTransactionService
{
    private PDO $db;
    private Redis $redis;
    private JWTService $jwt;

    public function __construct(PDO $db, Redis $redis, JWTService $jwt)
    {
        $this->db = $db;
        $this->redis = $redis;
        $this->jwt = $jwt;
    }

    /**
     * Send Money - সম্পূর্ণ stateless
     * Token-এ user info আছে, server-এ কিছু store করা হয়নি
     */
    public function sendMoney(string $authToken, array $request): array
    {
        // Step 1: Token verify (stateless - কোনো session lookup নেই)
        $user = $this->jwt->verifyToken($authToken);
        if (!$user) {
            return ['error' => 'অননুমোদিত অনুরোধ', 'code' => 401];
        }

        $senderPhone = $user['phone'];
        $recipientPhone = $request['recipient'];
        $amount = (float) $request['amount'];

        // Step 2: Idempotency check (distributed lock via Redis)
        $idempotencyKey = $request['idempotency_key'] ?? null;
        if ($idempotencyKey) {
            $existing = $this->redis->get("txn:idempotent:$idempotencyKey");
            if ($existing) {
                return json_decode($existing, true); // Already processed
            }
        }

        // Step 3: Transaction processing
        $this->db->beginTransaction();

        try {
            // Balance check
            $stmt = $this->db->prepare(
                "SELECT balance FROM accounts WHERE phone = ? FOR UPDATE"
            );
            $stmt->execute([$senderPhone]);
            $sender = $stmt->fetch();

            if ($sender['balance'] < $amount) {
                $this->db->rollBack();
                return ['error' => 'অপর্যাপ্ত ব্যালেন্স', 'code' => 400];
            }

            // Debit sender
            $this->db->prepare(
                "UPDATE accounts SET balance = balance - ? WHERE phone = ?"
            )->execute([$amount, $senderPhone]);

            // Credit recipient
            $this->db->prepare(
                "UPDATE accounts SET balance = balance + ? WHERE phone = ?"
            )->execute([$amount, $recipientPhone]);

            // Record transaction
            $txnId = 'TXN' . time() . rand(1000, 9999);
            $this->db->prepare(
                "INSERT INTO transactions (id, sender, recipient, amount, type, created_at)
                 VALUES (?, ?, ?, ?, 'send_money', NOW())"
            )->execute([$txnId, $senderPhone, $recipientPhone, $amount]);

            $this->db->commit();

            $result = [
                'success' => true,
                'transaction_id' => $txnId,
                'amount' => $amount,
                'recipient' => $recipientPhone,
                'message' => "৳{$amount} সফলভাবে পাঠানো হয়েছে",
            ];

            // Store idempotency result
            if ($idempotencyKey) {
                $this->redis->setex(
                    "txn:idempotent:$idempotencyKey",
                    86400,
                    json_encode($result)
                );
            }

            return $result;

        } catch (\Exception $e) {
            $this->db->rollBack();
            return ['error' => 'লেনদেন ব্যর্থ হয়েছে', 'code' => 500];
        }
    }
}

/**
 * External Session Store
 * Stateless app + Redis-এ centralized session
 */
class ExternalSessionStore
{
    private Redis $redis;
    private int $ttl;

    public function __construct(Redis $redis, int $ttl = 3600)
    {
        $this->redis = $redis;
        $this->ttl = $ttl;
    }

    public function createSession(string $userId, array $data): string
    {
        $sessionId = bin2hex(random_bytes(32));
        $sessionData = array_merge($data, [
            'user_id' => $userId,
            'created_at' => time(),
            'last_activity' => time(),
        ]);

        $this->redis->setex(
            "session:$sessionId",
            $this->ttl,
            json_encode($sessionData)
        );

        return $sessionId;
    }

    public function getSession(string $sessionId): ?array
    {
        $data = $this->redis->get("session:$sessionId");
        if (!$data) return null;

        $session = json_decode($data, true);

        // Update last activity
        $session['last_activity'] = time();
        $this->redis->setex(
            "session:$sessionId",
            $this->ttl,
            json_encode($session)
        );

        return $session;
    }

    public function destroySession(string $sessionId): void
    {
        $this->redis->del("session:$sessionId");
    }

    // Cart management (stateless app-এ external state)
    public function addToCart(string $sessionId, array $item): void
    {
        $session = $this->getSession($sessionId);
        $session['cart'][] = $item;
        $this->redis->setex(
            "session:$sessionId",
            $this->ttl,
            json_encode($session)
        );
    }
}

/**
 * Stateful WebSocket Handler
 * Real-time features-এর জন্য stateful approach
 */
class StatefulRideTracker
{
    // In-memory state - এই server instance-এ
    private array $activeRides = [];
    private array $riderLocations = [];
    private array $connections = [];

    public function onRiderLocationUpdate(string $riderId, float $lat, float $lng): void
    {
        // In-memory update (কোনো DB call নেই - fast!)
        $this->riderLocations[$riderId] = [
            'lat' => $lat,
            'lng' => $lng,
            'updated_at' => microtime(true),
        ];

        // এই rider-এর active ride-এর customer-কে notify
        $rideId = $this->findActiveRide($riderId);
        if ($rideId && isset($this->connections[$rideId])) {
            $this->connections[$rideId]->send(json_encode([
                'type' => 'location_update',
                'rider_lat' => $lat,
                'rider_lng' => $lng,
                'eta' => $this->calculateETA($riderId, $rideId),
            ]));
        }
    }

    public function onCustomerConnect(string $rideId, $connection): void
    {
        $this->connections[$rideId] = $connection;
    }

    private function findActiveRide(string $riderId): ?string
    {
        foreach ($this->activeRides as $rideId => $ride) {
            if ($ride['rider_id'] === $riderId) return $rideId;
        }
        return null;
    }

    private function calculateETA(string $riderId, string $rideId): string
    {
        // In-memory calculation - fast!
        return '৫ মিনিট';
    }
}

function base64url_encode(string $data): string
{
    return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
}

function base64url_decode(string $data): string
{
    return base64_decode(strtr($data, '-_', '+/'));
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Stateless REST API + Stateful WebSocket:

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');
const WebSocket = require('ws');

const app = express();
const redis = new Redis({ host: 'redis-cluster', port: 6379 });

const JWT_SECRET = process.env.JWT_SECRET;

// ===== STATELESS API Section =====

/**
 * Stateless Middleware - JWT Authentication
 * কোনো server-side session lookup নেই
 * Token-ই সব information বহন করে
 */
function statelessAuth(req, res, next) {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) {
        return res.status(401).json({ error: 'টোকেন প্রদান করুন' });
    }

    try {
        // Stateless verification - শুধু signature check
        const decoded = jwt.verify(token, JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        return res.status(401).json({ error: 'অবৈধ টোকেন' });
    }
}

/**
 * Stateless Transaction API
 * bKash-style - প্রতিটি request স্বাধীন
 */
app.post('/api/transactions/send', statelessAuth, async (req, res) => {
    const { recipient, amount, pin } = req.body;
    const sender = req.user.phone; // JWT থেকে

    // Idempotency (stateless - Redis-এ check)
    const idempotencyKey = req.headers['x-idempotency-key'];
    if (idempotencyKey) {
        const cached = await redis.get(`idempotent:${idempotencyKey}`);
        if (cached) return res.json(JSON.parse(cached));
    }

    try {
        // Transaction logic (database handles state)
        const result = await processTransaction(sender, recipient, amount);

        if (idempotencyKey) {
            await redis.setex(`idempotent:${idempotencyKey}`, 86400, JSON.stringify(result));
        }

        res.json(result);
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

/**
 * Stateless Balance Check
 * যেকোনো server instance handle করতে পারবে
 */
app.get('/api/balance', statelessAuth, async (req, res) => {
    const phone = req.user.phone;

    // Cache check (external state in Redis)
    const cached = await redis.get(`balance:${phone}`);
    if (cached) {
        return res.json({ balance: parseFloat(cached), cached: true });
    }

    // DB query
    const balance = await getBalanceFromDB(phone);
    await redis.setex(`balance:${phone}`, 30, balance.toString());

    res.json({ balance, cached: false });
});

// ===== EXTERNAL STATE MANAGEMENT =====

/**
 * Shopping Cart - Stateless App + Redis State
 * Chaldal/Daraz-style cart management
 */
class CartService {
    constructor(redis) {
        this.redis = redis;
        this.cartTTL = 7 * 24 * 3600; // ৭ দিন
    }

    async addItem(userId, item) {
        const cartKey = `cart:${userId}`;
        const cart = await this.getCart(userId);

        const existingIndex = cart.items.findIndex(i => i.productId === item.productId);
        if (existingIndex >= 0) {
            cart.items[existingIndex].quantity += item.quantity;
        } else {
            cart.items.push(item);
        }

        cart.total = cart.items.reduce((sum, i) => sum + (i.price * i.quantity), 0);
        cart.updatedAt = new Date().toISOString();

        await this.redis.setex(cartKey, this.cartTTL, JSON.stringify(cart));
        return cart;
    }

    async getCart(userId) {
        const cartKey = `cart:${userId}`;
        const data = await this.redis.get(cartKey);
        return data ? JSON.parse(data) : { items: [], total: 0, createdAt: new Date().toISOString() };
    }

    async removeItem(userId, productId) {
        const cart = await this.getCart(userId);
        cart.items = cart.items.filter(i => i.productId !== productId);
        cart.total = cart.items.reduce((sum, i) => sum + (i.price * i.quantity), 0);
        await this.redis.setex(`cart:${userId}`, this.cartTTL, JSON.stringify(cart));
        return cart;
    }
}

// ===== STATEFUL WebSocket Section =====

/**
 * Stateful Real-Time Tracking
 * Pathao-style ride tracking with WebSocket
 */
class RealTimeTrackingServer {
    constructor() {
        // In-memory state (stateful!)
        this.connections = new Map(); // rideId → WebSocket connection
        this.riderLocations = new Map(); // riderId → {lat, lng, timestamp}
        this.activeRides = new Map(); // rideId → ride info
    }

    initialize(server) {
        this.wss = new WebSocket.Server({ server, path: '/ws/tracking' });

        this.wss.on('connection', (ws, req) => {
            const rideId = new URL(req.url, 'http://localhost').searchParams.get('rideId');
            const role = new URL(req.url, 'http://localhost').searchParams.get('role');

            console.log(`[WebSocket] ${role} connected for ride: ${rideId}`);

            // Stateful: connection মেমোরিতে রাখা হচ্ছে
            if (role === 'customer') {
                this.connections.set(rideId, ws);
            }

            ws.on('message', (data) => {
                const message = JSON.parse(data);
                this.handleMessage(rideId, role, message);
            });

            ws.on('close', () => {
                if (role === 'customer') {
                    this.connections.delete(rideId);
                }
                console.log(`[WebSocket] ${role} disconnected from ride: ${rideId}`);
            });
        });
    }

    handleMessage(rideId, role, message) {
        switch (message.type) {
            case 'location_update':
                // Rider-এর location update (stateful - in-memory)
                this.riderLocations.set(message.riderId, {
                    lat: message.lat,
                    lng: message.lng,
                    timestamp: Date.now(),
                    heading: message.heading,
                });

                // Customer-কে real-time notify
                const customerWs = this.connections.get(rideId);
                if (customerWs && customerWs.readyState === WebSocket.OPEN) {
                    customerWs.send(JSON.stringify({
                        type: 'rider_location',
                        lat: message.lat,
                        lng: message.lng,
                        eta: this.calculateETA(rideId),
                    }));
                }
                break;

            case 'ride_completed':
                this.activeRides.delete(rideId);
                this.connections.delete(rideId);
                break;
        }
    }

    calculateETA(rideId) {
        const ride = this.activeRides.get(rideId);
        const riderLoc = this.riderLocations.get(ride?.riderId);
        if (!ride || !riderLoc) return 'গণনা করা হচ্ছে...';

        const distance = this.haversineDistance(
            riderLoc.lat, riderLoc.lng,
            ride.destinationLat, ride.destinationLng
        );
        const etaMinutes = Math.ceil(distance / 0.5); // ০.৫ km/min avg speed
        return `${etaMinutes} মিনিট`;
    }

    haversineDistance(lat1, lng1, lat2, lng2) {
        const R = 6371;
        const dLat = (lat2 - lat1) * Math.PI / 180;
        const dLng = (lng2 - lng1) * Math.PI / 180;
        const a = Math.sin(dLat / 2) ** 2 +
                  Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
                  Math.sin(dLng / 2) ** 2;
        return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    }

    // Scaling challenge: state অন্য instance-এ transfer করা
    getState() {
        return {
            activeConnections: this.connections.size,
            trackedRiders: this.riderLocations.size,
            activeRides: this.activeRides.size,
        };
    }
}

// ===== 12-Factor App: Config from Environment =====
const config = {
    port: process.env.PORT || 3000,
    redisUrl: process.env.REDIS_URL || 'redis://localhost:6379',
    dbUrl: process.env.DATABASE_URL,
    jwtSecret: process.env.JWT_SECRET,
    // Stateless: কোনো local file/state dependency নেই
};

// ===== Server Setup =====
const cartService = new CartService(redis);

app.post('/api/cart/add', statelessAuth, async (req, res) => {
    const cart = await cartService.addItem(req.user.id, req.body.item);
    res.json(cart);
});

app.get('/api/cart', statelessAuth, async (req, res) => {
    const cart = await cartService.getCart(req.user.id);
    res.json(cart);
});

const server = app.listen(config.port, () => {
    console.log(`✅ Stateless API চালু: port ${config.port}`);
    console.log(`📡 Stateful WebSocket ready: ws://localhost:${config.port}/ws/tracking`);
});

// Stateful WebSocket server (same process)
const tracker = new RealTimeTrackingServer();
tracker.initialize(server);
```

---

## 📖 12-Factor App Principles

Stateless design-এর জন্য 12-Factor App-এর গুরুত্বপূর্ণ নীতিসমূহ:

```
┌─────────────────────────────────────────────────────────┐
│              12-FACTOR APP (Stateless Focus)              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Factor III: Config                                      │
│  ─────────────────                                       │
│  Environment variables ব্যবহার করুন                     │
│  ❌ config.php-এ hardcode                               │
│  ✅ process.env.DATABASE_URL                            │
│                                                          │
│  Factor VI: Processes                                    │
│  ─────────────────                                       │
│  "Execute the app as stateless processes"               │
│  ❌ in-memory session                                   │
│  ✅ Redis/Database-এ session                            │
│                                                          │
│  Factor VIII: Concurrency                                │
│  ─────────────────────                                   │
│  Process model দিয়ে scale করুন                          │
│  ❌ একটি বড় process                                    │
│  ✅ অনেক ছোট stateless process                         │
│                                                          │
│  Factor IX: Disposability                                │
│  ─────────────────────                                   │
│  "Maximize robustness with fast startup/graceful shutdown"│
│  ❌ startup-এ state load করা                           │
│  ✅ instant start, কোনো warmup নেই                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### Stateless ব্যবহার করুন যখন:

| Scenario | কারণ |
|---|---|
| REST API / Microservices | Horizontal scaling সহজ |
| High-traffic services (bKash) | Load balancing সরল |
| Cloud deployment (AWS, GCP) | Auto-scaling কাজ করবে |
| Payment processing | কোনো instance fail করলেও সমস্যা নেই |
| Content delivery | Cache-friendly |

### Stateful ব্যবহার করুন যখন:

| Scenario | কারণ |
|---|---|
| WebSocket real-time (Pathao tracking) | Connection state রাখতে হয় |
| Game servers | Frame-by-frame state দরকার |
| In-memory cache/computation | DB roundtrip avoid করতে |
| Collaborative editing | Real-time sync দরকার |
| Live streaming/chat | Persistent connection দরকার |

### ⚠️ চ্যালেঞ্জসমূহ:

**Stateless চ্যালেঞ্জ:**
- প্রতি request-এ authentication overhead
- External state store (Redis) এর latency
- Token size বড় হলে bandwidth বাড়ে

**Stateful চ্যালেঞ্জ:**
- Horizontal scaling কঠিন
- Server failure = state loss
- Sticky session load balancing জটিল
- Deployment/restart-এ state migration

---

## 📚 সারসংক্ষেপ

```
┌────────────────────────────────────────┐
│         সিদ্ধান্ত নেওয়ার গাইড          │
├────────────────────────────────────────┤
│                                        │
│  প্রশ্ন: "আমার কি scaling দরকার?"     │
│     ├── হ্যাঁ → Stateless ✅           │
│     └── না → Stateful ঠিক আছে        │
│                                        │
│  প্রশ্ন: "Real-time দরকার?"           │
│     ├── হ্যাঁ → Stateful (WebSocket)   │
│     └── না → Stateless ✅              │
│                                        │
│  প্রশ্ন: "Server crash হলে কী হবে?"   │
│     ├── সমস্যা → Stateless ✅          │
│     └── acceptable → Stateful ok       │
│                                        │
│  Best Practice:                        │
│  → API layer: Stateless               │
│  → Real-time layer: Stateful          │
│  → State: External store (Redis)      │
│                                        │
└────────────────────────────────────────┘
```

> **মূল শিক্ষা:** বেশিরভাগ ক্ষেত্রে **Stateless + External State Store** (Redis/Database) হলো সর্বোত্তম পদ্ধতি। শুধুমাত্র real-time, low-latency requirement থাকলে Stateful ব্যবহার করুন।
