# 🔄 API ইডেম্পোটেন্সি (Idempotency in APIs)

> **"একটি অপারেশন ইডেম্পোটেন্ট হলে, সেটি একবার বা হাজারবার চালালেও ফলাফল একই থাকবে।"**

---

## 📑 সূচিপত্র

1. [ইডেম্পোটেন্সি কী?](#-ইডেম্পোটেন্সি-কী)
2. [কেন গুরুত্বপূর্ণ?](#-কেন-গুরুত্বপূর্ণ)
3. [HTTP মেথডের ইডেম্পোটেন্সি](#-http-মেথডের-ইডেম্পোটেন্সি)
4. [ইডেম্পোটেন্সি কী (Idempotency Key)](#-ইডেম্পোটেন্সি-কী-idempotency-key)
5. [ইমপ্লিমেন্টেশন প্যাটার্ন](#-ইমপ্লিমেন্টেশন-প্যাটার্ন)
6. [PHP Laravel ইমপ্লিমেন্টেশন](#-php-laravel-ইমপ্লিমেন্টেশন)
7. [JavaScript Express ইমপ্লিমেন্টেশন](#-javascript-express-ইমপ্লিমেন্টেশন)
8. [পেমেন্ট সিস্টেমে ইডেম্পোটেন্সি](#-পেমেন্ট-সিস্টেমে-ইডেম্পোটেন্সি)
9. [Retry সাথে ইডেম্পোটেন্সি](#-retry-সাথে-ইডেম্পোটেন্সি)
10. [সুবিধা ও অসুবিধা](#-সুবিধা-ও-অসুবিধা)
11. [সাধারণ ভুল (Anti-patterns)](#-সাধারণ-ভুল-anti-patterns)
12. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 ইডেম্পোটেন্সি কী?

ইডেম্পোটেন্সি (Idempotency) হলো এমন একটি বৈশিষ্ট্য যেখানে কোনো অপারেশন **একবার বা একাধিকবার** চালালেও সিস্টেমের অবস্থা (state) **একই থাকে**। গাণিতিকভাবে, `f(x) = f(f(x))`।

### বাস্তব উদাহরণ — ATM থেকে টাকা তোলা

কল্পনা করুন, আপনি ATM থেকে ৫,০০০ টাকা তুলছেন। বোতাম চাপার পর নেটওয়ার্ক সমস্যায় কোনো response আসলো না। আপনি আবার বোতাম চাপলেন। এখন দুটি জিনিস হতে পারে:

```
❌ ইডেম্পোটেন্ট না হলে:
┌──────────┐         ┌──────────┐         ┌──────────┐
│  আপনি    │ ৫০০০৳  │   ATM    │ ৫০০০৳  │   ব্যাংক  │
│  (গ্রাহক) │───────→│  মেশিন   │───────→│  সার্ভার  │
│          │ ৫০০০৳  │          │ ৫০০০৳  │          │
│          │───────→│          │───────→│ ব্যালেন্স: │
│          │        │          │        │ ১০,০০০৳   │
│          │        │          │        │ কাটলো!    │
└──────────┘        └──────────┘        └──────────┘
     ফলাফল: ৫,০০০ এর বদলে ১০,০০০ টাকা কেটে গেলো! 😱

✅ ইডেম্পোটেন্ট হলে:
┌──────────┐         ┌──────────┐         ┌──────────┐
│  আপনি    │ ৫০০০৳  │   ATM    │ ৫০০০৳  │   ব্যাংক  │
│  (গ্রাহক) │───────→│  মেশিন   │───────→│  সার্ভার  │
│          │ ৫০০০৳  │          │ ৫০০০৳  │          │
│          │───────→│          │───────→│ একই txn!  │
│          │        │          │        │ ৫,০০০৳    │
│          │        │          │        │ কাটলো।    │
└──────────┘        └──────────┘        └──────────┘
     ফলাফল: শুধু ৫,০০০ টাকাই কাটলো। ✅
```

### বাস্তব উদাহরণ — bKash Send Money

ধরুন আপনি bKash দিয়ে বন্ধুকে ১,০০০ টাকা পাঠাচ্ছেন। আপনার ফোনের নেটওয়ার্ক slow, তাই "Send" বোতাম দুইবার চাপলেন:

```
আপনার bKash App                    bKash সার্ভার
      │                                  │
      │── Send ৳1000 (txn: abc123) ─────→│  ✅ প্রথমবার প্রসেস
      │                                  │     ব্যালেন্স: ৳5000 → ৳4000
      │   (নেটওয়ার্ক timeout)             │
      │                                  │
      │── Send ৳1000 (txn: abc123) ─────→│  🔍 txn abc123 আগেই আছে!
      │                                  │     ব্যালেন্স পরিবর্তন হবে না
      │←── ✅ সফল (cached response) ─────│
      │                                  │

    একই transaction ID থাকায় দ্বিতীয়বার টাকা কাটে না!
```

---

## 🎯 কেন গুরুত্বপূর্ণ?

### ১. নেটওয়ার্ক অনির্ভরযোগ্যতা (Network Unreliability)

বাংলাদেশের মোবাইল নেটওয়ার্ক সবসময় স্থিতিশীল থাকে না। গ্রামীণ এলাকায় 2G/3G তে API call timeout হওয়া খুবই সাধারণ ব্যাপার। Client retry করবে — তখন ইডেম্পোটেন্সি না থাকলে বিপদ।

### ২. ডুপ্লিকেট পেমেন্ট প্রতিরোধ

```
সমস্যা — ডুপ্লিকেট পেমেন্ট:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  গ্রাহক: ৳৫০০ পেমেন্ট করলো (Daraz অর্ডার)               │
│                                                         │
│  ১ম Request ──→ SSLCommerz ──→ ✅ পেমেন্ট সফল            │
│                              (Response হারিয়ে গেলো)      │
│                                                         │
│  ২য় Request ──→ SSLCommerz ──→ ❌ আবার ৳৫০০ কাটলো!      │
│                                                         │
│  মোট কাটলো: ৳১,০০০ (কিন্তু অর্ডার মাত্র ৳৫০০ এর!)      │
│                                                         │
│  গ্রাহক রাগ করলো → কাস্টমার সাপোর্টে ফোন → রিফান্ড      │
│  → ব্যবসার খরচ + গ্রাহকের বিশ্বাস কমলো                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### ৩. ডেটা কনসিস্টেন্সি

Microservices architecture-এ একটি request একাধিক সার্ভিসে যায়। যেকোনো পয়েন্টে failure হলে retry দরকার — ইডেম্পোটেন্সি ছাড়া ডেটা inconsistent হয়ে যায়।

### ৪. Distributed System-এ Message Delivery

```
Message Delivery গ্যারান্টি:

┌──────────────┬─────────────────────────────────────────┐
│   ধরন        │  বর্ণনা                                  │
├──────────────┼─────────────────────────────────────────┤
│ At-most-once │ সর্বোচ্চ ১ বার → মেসেজ হারাতে পারে      │
│ At-least-once│ কমপক্ষে ১ বার → ডুপ্লিকেট হতে পারে      │
│ Exactly-once │ ঠিক ১ বার → ইডেম্পোটেন্সি দিয়ে অর্জন   │
└──────────────┴─────────────────────────────────────────┘

At-least-once + Idempotency = Exactly-once (কার্যকরভাবে)
```

---

## 📊 HTTP মেথডের ইডেম্পোটেন্সি

HTTP specification অনুযায়ী কিছু মেথড naturally ইডেম্পোটেন্ট এবং কিছু নয়:

```
┌──────────┬───────────────┬────────────┬──────────────────────────────────┐
│  মেথড    │ ইডেম্পোটেন্ট?│   Safe?    │  উদাহরণ                          │
├──────────┼───────────────┼────────────┼──────────────────────────────────┤
│  GET     │    ✅ হ্যাঁ    │   ✅ হ্যাঁ  │ প্রোডাক্ট লিস্ট দেখা             │
│  HEAD    │    ✅ হ্যাঁ    │   ✅ হ্যাঁ  │ রিসোর্স আছে কিনা চেক            │
│  PUT     │    ✅ হ্যাঁ    │   ❌ না    │ প্রোফাইল আপডেট (পুরোটা)          │
│  DELETE  │    ✅ হ্যাঁ    │   ❌ না    │ অর্ডার ক্যান্সেল                  │
│  OPTIONS │    ✅ হ্যাঁ    │   ✅ হ্যাঁ  │ CORS preflight                   │
│  POST    │    ❌ না      │   ❌ না    │ নতুন অর্ডার তৈরি                 │
│  PATCH   │    ❌ না      │   ❌ না    │ ব্যালেন্স ৳৫০০ যোগ করো           │
└──────────┴───────────────┴────────────┴──────────────────────────────────┘
```

### কেন POST ইডেম্পোটেন্ট না?

```
POST /api/orders (অর্ডার তৈরি):

১ম call → Order #101 তৈরি হলো     → ১টি অর্ডার
২য় call → Order #102 তৈরি হলো     → ২টি অর্ডার  ← ভিন্ন ফলাফল!

PUT /api/orders/101 (অর্ডার আপডেট):

১ম call → Order #101 আপডেট হলো    → status: "confirmed"
২য় call → Order #101 আবার আপডেট  → status: "confirmed"  ← একই ফলাফল!
```

### PATCH কেন ইডেম্পোটেন্ট না?

```
PATCH /api/wallet/balance
Body: { "add": 500 }

১ম call → ব্যালেন্স: ১০০০ + ৫০০ = ১৫০০
২য় call → ব্যালেন্স: ১৫০০ + ৫০০ = ২০০০  ← ভিন্ন!

কিন্তু এভাবে করলে ইডেম্পোটেন্ট:
PATCH /api/wallet/balance
Body: { "set": 1500 }

১ম call → ব্যালেন্স: ১৫০০
২য় call → ব্যালেন্স: ১৫০০  ← একই!
```

---

## 🔑 ইডেম্পোটেন্সি কী (Idempotency Key)

POST-এর মতো naturally non-idempotent মেথডকে ইডেম্পোটেন্ট করতে **Idempotency Key** ব্যবহার করা হয়। এটি প্রতিটি unique অপারেশনের জন্য একটি unique identifier — সাধারণত UUID v4।

### কিভাবে কাজ করে?

```
Client                           Server
  │                                 │
  │  POST /api/payments             │
  │  Idempotency-Key: 550e8400...   │
  │  Body: { amount: 1000 }         │
  │────────────────────────────────→│
  │                                 │  ১. Key চেক করো store-এ
  │                                 │  ২. নেই? → প্রসেস করো
  │                                 │  ৩. Response + Key সেভ করো
  │←────────────────────────────────│
  │  201 Created                    │
  │                                 │
  │  (Network timeout — retry)      │
  │                                 │
  │  POST /api/payments             │
  │  Idempotency-Key: 550e8400...   │  ← একই key!
  │  Body: { amount: 1000 }         │
  │────────────────────────────────→│
  │                                 │  ১. Key চেক করো store-এ
  │                                 │  ২. আছে! → cached response দাও
  │←────────────────────────────────│
  │  201 Created (cached)           │
  │                                 │
```

### Stripe-এর Approach

Stripe পেমেন্ট API তে `Idempotency-Key` হেডার বাধ্যতামূলক:

```
curl https://api.stripe.com/v1/charges \
  -H "Idempotency-Key: KG5LxwFBepaKHyUD" \
  -d amount=2000 \
  -d currency=bdt \
  -d source=tok_visa

# একই key দিয়ে আবার পাঠালে → আগের response-ই ফিরে আসবে
# নতুন চার্জ হবে না!
```

### Key Generation Best Practices

```
✅ ভালো Idempotency Key:
├── UUID v4: "550e8400-e29b-41d4-a716-446655440000"
├── কম্বিনেশন: "user_123_order_456_payment_789"
├── Hash: SHA256(user_id + amount + timestamp_minute)
└── Client-generated unique ID

❌ খারাপ Idempotency Key:
├── Sequential: "1", "2", "3" (সহজে guessable)
├── Timestamp only: "1703001234" (collision সম্ভব)
├── Empty/null key
└── Server-generated (client retry-তে আলাদা হবে)
```

---

## 🏗️ ইমপ্লিমেন্টেশন প্যাটার্ন

### প্যাটার্ন ১: Database Unique Constraint

সবচেয়ে সহজ পদ্ধতি — database-এ unique constraint দিয়ে ডুপ্লিকেট ঠেকানো:

```
┌─────────────────────────────────────────────────┐
│  transactions টেবিল                              │
├──────────────┬──────────┬───────┬───────────────┤
│ id           │ txn_ref  │ amount│ status        │
│              │ (UNIQUE) │       │               │
├──────────────┼──────────┼───────┼───────────────┤
│ 1            │ bkash_001│ 1000  │ completed     │
│ 2            │ bkash_002│ 500   │ completed     │
│ —            │ bkash_001│ 1000  │ ← DUPLICATE!  │
│              │          │       │   INSERT FAIL  │
└──────────────┴──────────┴───────┴───────────────┘

SQL: INSERT INTO transactions (txn_ref, amount)
     VALUES ('bkash_001', 1000)
     → ERROR: Duplicate entry 'bkash_001' for key 'txn_ref_unique'
```

### প্যাটার্ন ২: Idempotency Key Store (Redis/DB)

Dedicated store-এ idempotency key ও response সেভ রাখা:

```
┌──────────────────────────────────────────────────────────┐
│                 Idempotency Key Flow                     │
│                                                          │
│  Request ──→ ┌───────────────┐                           │
│              │ Key Store     │                           │
│              │ (Redis/DB)    │                           │
│              └───────┬───────┘                           │
│                      │                                   │
│              ┌───────┴───────┐                           │
│              │               │                           │
│          Key আছে?       Key নেই                         │
│              │               │                           │
│              ▼               ▼                           │
│     ┌─────────────┐  ┌──────────────┐                   │
│     │ Status চেক  │  │ Key সেভ করো  │                   │
│     │             │  │ status:      │                   │
│     │             │  │ "processing" │                   │
│     └──────┬──────┘  └──────┬───────┘                   │
│            │                │                            │
│     ┌──────┴──────┐        ▼                            │
│     │             │  ┌──────────────┐                   │
│ completed?   processing? │ Business    │                   │
│     │             │  │ Logic চালাও │                   │
│     ▼             ▼  └──────┬───────┘                   │
│  Cached      409 or        │                            │
│  Response    Wait    ┌─────┴──────┐                     │
│  ফেরত দাও          │ Response   │                     │
│                     │ সেভ করো,   │                     │
│                     │ status:    │                     │
│                     │ "completed"│                     │
│                     └────────────┘                     │
└──────────────────────────────────────────────────────────┘
```

### প্যাটার্ন ৩: Request Fingerprinting

Request body-র hash তৈরি করে ডুপ্লিকেট চেক করা:

```
Request ──→ SHA256(method + url + body + user_id)
         = fingerprint "a1b2c3d4..."

┌────────────────────────────────────────┐
│  Request Fingerprint Cache (Redis)     │
├─────────────────────┬──────────────────┤
│  Fingerprint        │  TTL             │
├─────────────────────┼──────────────────┤
│  a1b2c3d4...        │  ৫ মিনিট         │
│  e5f6g7h8...        │  ৫ মিনিট         │
└─────────────────────┴──────────────────┘

সুবিধা: Client-কে key পাঠাতে হয় না
অসুবিধা: Intentional duplicate ধরতে পারে না
         (দুইবার একই জিনিস অর্ডার করা valid হতে পারে)
```

### প্যাটার্ন ৪: Conditional Requests (ETags, If-Match)

```
Step 1: GET /api/products/42
        Response: ETag: "v5"

Step 2: PUT /api/products/42
        If-Match: "v5"
        Body: { name: "নতুন নাম", price: 500 }

        → Server চেক: বর্তমান ETag "v5" কি?
          ✅ হ্যাঁ → আপডেট করো, নতুন ETag: "v6"
          ❌ না  → 412 Precondition Failed

Concurrent update থেকে রক্ষা করে:
User A: If-Match: "v5" → ✅ আপডেট → ETag "v6"
User B: If-Match: "v5" → ❌ 412! (stale data)
```

---

## 💻 PHP Laravel ইমপ্লিমেন্টেশন

### Approach ১: Middleware with Redis

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Redis;
use Symfony\Component\HttpFoundation\Response;

class IdempotencyMiddleware
{
    private const KEY_PREFIX = 'idempotency:';
    private const TTL_SECONDS = 86400; // ২৪ ঘণ্টা

    public function handle(Request $request, Closure $next): Response
    {
        // শুধু POST, PATCH মেথডে কাজ করবে
        if (!in_array($request->method(), ['POST', 'PATCH'])) {
            return $next($request);
        }

        $idempotencyKey = $request->header('Idempotency-Key');

        if (empty($idempotencyKey)) {
            return response()->json([
                'error' => 'Idempotency-Key হেডার আবশ্যক',
                'message' => 'POST/PATCH request-এ Idempotency-Key হেডার পাঠান।',
            ], 400);
        }

        $cacheKey = self::KEY_PREFIX . $idempotencyKey;
        $cached = Redis::get($cacheKey);

        // আগে থেকেই প্রসেস হয়ে থাকলে cached response দাও
        if ($cached) {
            $cachedData = json_decode($cached, true);

            // এখনো processing চলছে কিনা চেক
            if ($cachedData['status'] === 'processing') {
                return response()->json([
                    'error' => 'Request এখনো প্রসেস হচ্ছে',
                    'message' => 'কিছুক্ষণ পরে আবার চেষ্টা করুন।',
                ], 409);
            }

            // Request body মিলছে কিনা verify
            if ($cachedData['fingerprint'] !== $this->fingerprint($request)) {
                return response()->json([
                    'error' => 'Idempotency Key conflict',
                    'message' => 'এই key আগে ভিন্ন request-এ ব্যবহৃত হয়েছে।',
                ], 422);
            }

            return response()->json(
                $cachedData['body'],
                $cachedData['status_code'],
                ['X-Idempotent-Replayed' => 'true']
            );
        }

        // Processing শুরুর চিহ্ন রাখো (race condition এড়াতে)
        $locked = Redis::set($cacheKey, json_encode([
            'status' => 'processing',
            'fingerprint' => $this->fingerprint($request),
        ]), 'EX', 60, 'NX'); // NX = শুধু নতুন হলে সেট করো

        if (!$locked) {
            return response()->json([
                'error' => 'Concurrent request',
                'message' => 'এই key দিয়ে আরেকটি request চলছে।',
            ], 409);
        }

        // মূল request প্রসেস করো
        $response = $next($request);

        // সফল response cache করো
        Redis::setex($cacheKey, self::TTL_SECONDS, json_encode([
            'status' => 'completed',
            'fingerprint' => $this->fingerprint($request),
            'status_code' => $response->getStatusCode(),
            'body' => json_decode($response->getContent(), true),
        ]));

        return $response;
    }

    private function fingerprint(Request $request): string
    {
        return hash('sha256', $request->method() . $request->url() . $request->getContent());
    }
}
```

### Route Registration

```php
// routes/api.php
Route::middleware(['idempotency'])->group(function () {
    Route::post('/payments', [PaymentController::class, 'store']);
    Route::post('/orders', [OrderController::class, 'store']);
    Route::post('/transfers', [TransferController::class, 'store']);
});
```

### Approach ২: Database Table

```php
<?php

// Migration
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('idempotency_keys', function (Blueprint $table) {
            $table->id();
            $table->string('key', 255)->unique();
            $table->string('request_path');
            $table->string('request_fingerprint', 64);
            $table->unsignedSmallInteger('response_code');
            $table->json('response_body');
            $table->enum('status', ['processing', 'completed', 'failed']);
            $table->timestamp('expires_at');
            $table->timestamps();

            $table->index('expires_at'); // cleanup query-র জন্য
        });
    }
};
```

```php
<?php

namespace App\Services;

use App\Models\IdempotencyKey;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;

class IdempotencyService
{
    public function check(Request $request): ?array
    {
        $key = $request->header('Idempotency-Key');
        if (!$key) return null;

        $record = IdempotencyKey::where('key', $key)
            ->where('expires_at', '>', Carbon::now())
            ->first();

        if (!$record) return null;

        if ($record->status === 'processing') {
            return ['status' => 'processing'];
        }

        // Fingerprint verify
        $fingerprint = hash('sha256', $request->getContent());
        if ($record->request_fingerprint !== $fingerprint) {
            return ['status' => 'conflict'];
        }

        return [
            'status' => 'completed',
            'response_code' => $record->response_code,
            'response_body' => $record->response_body,
        ];
    }

    public function lock(Request $request): bool
    {
        try {
            IdempotencyKey::create([
                'key' => $request->header('Idempotency-Key'),
                'request_path' => $request->path(),
                'request_fingerprint' => hash('sha256', $request->getContent()),
                'response_code' => 0,
                'response_body' => [],
                'status' => 'processing',
                'expires_at' => Carbon::now()->addHours(24),
            ]);
            return true;
        } catch (\Illuminate\Database\QueryException $e) {
            return false; // unique constraint violation
        }
    }

    public function complete(string $key, int $statusCode, array $body): void
    {
        IdempotencyKey::where('key', $key)->update([
            'status' => 'completed',
            'response_code' => $statusCode,
            'response_body' => $body,
        ]);
    }

    // পুরাতন key পরিষ্কার করার জন্য (Schedule এ চালাতে হবে)
    public function cleanup(): int
    {
        return IdempotencyKey::where('expires_at', '<', Carbon::now())->delete();
    }
}
```

### Controller-এ ব্যবহার — bKash পেমেন্ট উদাহরণ

```php
<?php

namespace App\Http\Controllers;

use App\Services\IdempotencyService;
use App\Services\BkashPaymentService;
use Illuminate\Http\Request;

class PaymentController extends Controller
{
    public function store(Request $request, BkashPaymentService $bkash)
    {
        $validated = $request->validate([
            'amount' => 'required|numeric|min:10|max:25000',
            'receiver' => 'required|regex:/^01[3-9]\d{8}$/',
            'pin' => 'required|digits:5',
        ]);

        // পেমেন্ট প্রসেস — ইডেম্পোটেন্সি middleware handle করছে
        $result = $bkash->sendMoney(
            receiver: $validated['receiver'],
            amount: $validated['amount'],
            reference: $request->header('Idempotency-Key'),
        );

        return response()->json([
            'transaction_id' => $result->transactionId,
            'amount' => $result->amount,
            'receiver' => $result->receiver,
            'status' => 'success',
            'message' => 'টাকা সফলভাবে পাঠানো হয়েছে।',
        ], 201);
    }
}
```

---

## 🟢 JavaScript Express ইমপ্লিমেন্টেশন

### Middleware with Redis

```javascript
const crypto = require("crypto");
const Redis = require("ioredis");

const redis = new Redis({
  host: process.env.REDIS_HOST || "127.0.0.1",
  port: process.env.REDIS_PORT || 6379,
});

const KEY_PREFIX = "idempotency:";
const TTL_SECONDS = 86400; // ২৪ ঘণ্টা
const LOCK_TTL = 60; // ১ মিনিট lock

function fingerprint(req) {
  return crypto
    .createHash("sha256")
    .update(req.method + req.originalUrl + JSON.stringify(req.body))
    .digest("hex");
}

function idempotencyMiddleware(req, res, next) {
  // GET, PUT, DELETE ইতিমধ্যে idempotent
  if (!["POST", "PATCH"].includes(req.method)) {
    return next();
  }

  const idempotencyKey = req.headers["idempotency-key"];

  if (!idempotencyKey) {
    return res.status(400).json({
      error: "Idempotency-Key হেডার আবশ্যক",
      message: "POST/PATCH request-এ Idempotency-Key হেডার পাঠান।",
    });
  }

  const cacheKey = KEY_PREFIX + idempotencyKey;
  const reqFingerprint = fingerprint(req);

  redis
    .get(cacheKey)
    .then((cached) => {
      if (cached) {
        const data = JSON.parse(cached);

        // এখনো প্রসেস হচ্ছে
        if (data.status === "processing") {
          return res.status(409).json({
            error: "Request প্রসেস হচ্ছে",
            message: "কিছুক্ষণ পরে আবার চেষ্টা করুন।",
          });
        }

        // Fingerprint মেলে না → ভিন্ন request-এ একই key!
        if (data.fingerprint !== reqFingerprint) {
          return res.status(422).json({
            error: "Idempotency Key conflict",
            message: "এই key আগে ভিন্ন request-এ ব্যবহৃত হয়েছে।",
          });
        }

        // Cached response ফেরত দাও
        res.set("X-Idempotent-Replayed", "true");
        return res.status(data.statusCode).json(data.body);
      }

      // NX দিয়ে atomic lock — race condition এড়াতে
      return redis
        .set(
          cacheKey,
          JSON.stringify({ status: "processing", fingerprint: reqFingerprint }),
          "EX",
          LOCK_TTL,
          "NX"
        )
        .then((lockResult) => {
          if (!lockResult) {
            return res.status(409).json({
              error: "Concurrent request",
              message: "এই key দিয়ে আরেকটি request চলছে।",
            });
          }

          // Original response.json() override করে response ক্যাপচার করি
          const originalJson = res.json.bind(res);
          res.json = (body) => {
            // সফল response cache করো
            redis.setex(
              cacheKey,
              TTL_SECONDS,
              JSON.stringify({
                status: "completed",
                fingerprint: reqFingerprint,
                statusCode: res.statusCode,
                body: body,
              })
            );
            return originalJson(body);
          };

          next();
        });
    })
    .catch((err) => {
      console.error("Idempotency middleware error:", err);
      next(); // Redis down হলেও request ব্লক করবো না
    });
}

module.exports = idempotencyMiddleware;
```

### Route-এ ব্যবহার

```javascript
const express = require("express");
const idempotency = require("./middleware/idempotency");

const app = express();
app.use(express.json());

// পেমেন্ট route-এ ইডেম্পোটেন্সি middleware লাগাও
app.post("/api/payments", idempotency, async (req, res) => {
  const { amount, receiver, method } = req.body;

  // SSLCommerz পেমেন্ট প্রসেস (উদাহরণ)
  const transaction = await processPayment({
    amount,
    receiver,
    reference: req.headers["idempotency-key"],
  });

  res.status(201).json({
    transaction_id: transaction.id,
    amount: transaction.amount,
    status: "success",
    message: "পেমেন্ট সফলভাবে সম্পন্ন হয়েছে।",
  });
});

// Pathao রাইড বুকিং
app.post("/api/rides", idempotency, async (req, res) => {
  const { pickup, destination, rideType } = req.body;

  const ride = await bookRide({ pickup, destination, rideType });

  res.status(201).json({
    ride_id: ride.id,
    driver: ride.driver,
    estimated_fare: ride.fare,
    message: "রাইড বুক করা হয়েছে।",
  });
});

app.listen(3000, () => console.log("Server চালু হয়েছে port 3000-এ"));
```

---

## 💳 পেমেন্ট সিস্টেমে ইডেম্পোটেন্সি

পেমেন্ট সিস্টেমে ইডেম্পোটেন্সি সবচেয়ে গুরুত্বপূর্ণ — কারণ এখানে ভুল মানেই আর্থিক ক্ষতি।

### bKash Double-Charge Prevention

```
গ্রাহকের অ্যাপ                  bKash API                    ব্যাংক
     │                            │                            │
     │  POST /payment             │                            │
     │  Key: pay_abc123           │                            │
     │  Amount: ৳500              │                            │
     │───────────────────────────→│                            │
     │                            │  ১. pay_abc123 চেক         │
     │                            │  ২. নেই → lock করো         │
     │                            │                            │
     │                            │  Debit ৳500               │
     │                            │───────────────────────────→│
     │                            │                            │
     │                            │←── ✅ Debited ────────────│
     │                            │                            │
     │   ✗ Network Timeout        │  ৩. Response সেভ করো       │
     │   (response হারিয়ে গেলো)   │     key: pay_abc123       │
     │                            │     status: completed      │
     │                            │                            │
     │  POST /payment (retry)     │                            │
     │  Key: pay_abc123           │                            │
     │  Amount: ৳500              │                            │
     │───────────────────────────→│                            │
     │                            │  ১. pay_abc123 চেক         │
     │                            │  ২. আছে! completed!        │
     │                            │  ৩. ব্যাংকে যাবে না        │
     │                            │                            │
     │←── ✅ Success (cached) ────│                            │
     │                            │                            │
     │  গ্রাহক দেখলো: সফল!         │     ব্যাংকে মাত্র ১ বার     │
     │  টাকা ১ বারই কাটলো ✅       │     ৳500 কাটলো ✅           │
```

### SSLCommerz Integration Pattern

```php
<?php

class SSLCommerzPaymentService
{
    public function initiatePayment(array $data): array
    {
        $idempotencyKey = sprintf(
            'ssl_%s_%s_%s',
            $data['order_id'],
            $data['amount'],
            date('Ymd')
        );

        // আগে কি এই অর্ডারের পেমেন্ট হয়েছে?
        $existing = Payment::where('idempotency_key', $idempotencyKey)
            ->whereIn('status', ['initiated', 'completed'])
            ->first();

        if ($existing) {
            return [
                'status' => 'already_exists',
                'payment' => $existing,
                'message' => 'এই অর্ডারের পেমেন্ট আগেই শুরু হয়েছে।',
            ];
        }

        // নতুন পেমেন্ট শুরু করো
        $payment = Payment::create([
            'idempotency_key' => $idempotencyKey,
            'order_id' => $data['order_id'],
            'amount' => $data['amount'],
            'status' => 'initiated',
        ]);

        $sslResponse = $this->callSSLCommerzAPI($payment);

        return [
            'status' => 'initiated',
            'payment' => $payment,
            'redirect_url' => $sslResponse['GatewayPageURL'],
        ];
    }
}
```

---

## 🔁 Retry সাথে ইডেম্পোটেন্সি

ইডেম্পোটেন্সি retry-কে নিরাপদ করে তোলে। Retry ছাড়া ইডেম্পোটেন্সি অর্থহীন, ইডেম্পোটেন্সি ছাড়া retry বিপজ্জনক।

### Retry + Idempotency Flow

```
Client                                Server
  │                                      │
  │── Request (Key: abc) ──────────────→ │ ✅ প্রসেস
  │                                      │
  │  ✗ timeout (response পায়নি)          │
  │                                      │
  │  ⏱️ 1s পরে retry                     │
  │── Request (Key: abc) ──────────────→ │ 🔍 abc আছে → cached
  │←── ✅ Response ────────────────────── │
  │                                      │
```

### Exponential Backoff with Idempotency (JavaScript)

```javascript
async function retryableRequest(url, data, maxRetries = 3) {
  // একবার key তৈরি করো — সব retry-তে একই key ব্যবহার হবে
  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(data),
      });

      if (response.ok) return await response.json();

      // 4xx error → retry করে লাভ নেই (client error)
      if (response.status >= 400 && response.status < 500) {
        throw new Error(`Client error: ${response.status}`);
      }

      // 5xx → retry করো
    } catch (err) {
      if (attempt === maxRetries) throw err;

      // Exponential backoff: 1s, 2s, 4s...
      const delay = Math.pow(2, attempt) * 1000;
      const jitter = Math.random() * 1000;
      console.log(`Retry ${attempt + 1}/${maxRetries} — ${delay + jitter}ms পরে`);
      await new Promise((r) => setTimeout(r, delay + jitter));
    }
  }
}

// ব্যবহার: Daraz অর্ডার তৈরি
retryableRequest("/api/orders", {
  product_id: 12345,
  quantity: 1,
  shipping_address: "মিরপুর-১০, ঢাকা",
}).then((order) => console.log("অর্ডার হয়েছে:", order.id));
```

### Retry Strategy তুলনা

```
┌───────────────────┬──────────────────────────────────────┐
│ Strategy          │ বিবরণ                                 │
├───────────────────┼──────────────────────────────────────┤
│ Immediate Retry   │ সাথে সাথে retry → সার্ভারে চাপ বাড়ে  │
│ Fixed Delay       │ প্রতিবার ২ সেকেন্ড → সহজ কিন্তু slow │
│ Exponential       │ 1s, 2s, 4s, 8s → ভালো balance       │
│ Exp + Jitter      │ Random delay যোগ → thundering herd   │
│                   │ সমস্যা এড়ায় ✅ (সবচেয়ে ভালো)        │
└───────────────────┴──────────────────────────────────────┘
```

---

## ⚖️ সুবিধা ও অসুবিধা

```
┌──────────────────────────────────┬──────────────────────────────────────┐
│         ✅ সুবিধা                │          ❌ অসুবিধা                  │
├──────────────────────────────────┼──────────────────────────────────────┤
│ ডুপ্লিকেট অপারেশন প্রতিরোধ      │ অতিরিক্ত storage দরকার (Redis/DB)   │
│                                  │                                      │
│ নিরাপদ retry সম্ভব হয়           │ সিস্টেমে complexity বাড়ে             │
│                                  │                                      │
│ ডেটা consistency বজায় থাকে      │ Key management দরকার (TTL, cleanup)  │
│                                  │                                      │
│ গ্রাহকের আর্থিক ক্ষতি রোধ       │ Response caching-এ memory লাগে      │
│                                  │                                      │
│ Distributed system-এ নির্ভরযোগ্য │ Race condition handle করতে হয়       │
│                                  │                                      │
│ Client-Server trust বাড়ে        │ Clock/TTL synchronization সমস্যা     │
│                                  │                                      │
│ Payment gateway সুরক্ষা          │ সব endpoint-এ দরকার নেই, বেছে       │
│                                  │ বেছে লাগাতে হয়                       │
└──────────────────────────────────┴──────────────────────────────────────┘
```

---

## 🚫 সাধারণ ভুল (Anti-patterns)

### ভুল ১: সার্ভারে Idempotency Key তৈরি করা

```
❌ ভুল:
Client ─── POST /pay ──→ Server generates key "xyz"
Client ─── POST /pay ──→ Server generates key "abc"  ← আলাদা key!
(retry হিসেবে কাজ করলো না)

✅ সঠিক:
Client generates key "xyz"
Client ─── POST /pay (Key: xyz) ──→ Server stores "xyz"
Client ─── POST /pay (Key: xyz) ──→ Server finds "xyz" → cached!
```

### ভুল ২: Response cache না করা

```
❌ ভুল:
Request 1: Key=abc → Process → Return { id: 1 }  (response সেভ হলো না)
Request 2: Key=abc → "Already processed" → 409 Error?

গ্রাহক জানতে পারলো না আসল result কী ছিলো!

✅ সঠিক:
Request 1: Key=abc → Process → { id: 1 } সেভ → Return { id: 1 }
Request 2: Key=abc → Cached { id: 1 } → Return { id: 1 }

গ্রাহক আসল result পেলো, যেন প্রথমবারই হয়েছে!
```

### ভুল ৩: TTL না দেওয়া

```
❌ ভুল:
Idempotency keys চিরকাল থেকে যায়
→ Storage ক্রমাগত বাড়তে থাকে
→ Redis/DB ফুলে যায়
→ এক সময় সিস্টেম crash!

✅ সঠিক:
Key TTL: ২৪ ঘণ্টা (পেমেন্টের জন্য)
Key TTL: ১ ঘণ্টা (সাধারণ API-র জন্য)
Scheduled cleanup job চালানো
```

### ভুল ৪: Request Body verify না করা

```
❌ ভুল:
Request 1: Key=abc, Body: { amount: 1000 } → Processed
Request 2: Key=abc, Body: { amount: 5000 } → Returns cached (1000 এর response!)

Client ভুল key reuse করলে ভুল response পেতে পারে!

✅ সঠিক:
Request 1: Key=abc, Fingerprint=hash1 → Processed
Request 2: Key=abc, Fingerprint=hash2 → 422 Error!
"এই key ভিন্ন request body-র সাথে ব্যবহৃত হয়েছে"
```

### ভুল ৫: সব endpoint-এ Idempotency বাধ্যতামূলক করা

```
❌ ভুল:
GET /products ← Idempotency key লাগবে? (GET ইতিমধ্যে idempotent!)
GET /users/me ← অপ্রয়োজনীয়!

✅ সঠিক:
শুধু state-changing, non-idempotent endpoint-এ:
POST /payments  ← ✅ টাকা কাটে, দরকার
POST /orders    ← ✅ অর্ডার তৈরি হয়, দরকার
PATCH /wallet   ← ✅ ব্যালেন্স বদলায়, দরকার
GET /products   ← ❌ দরকার নেই
DELETE /cart/5   ← ❌ DELETE ইতিমধ্যে idempotent
```

---

## 🤔 কখন ব্যবহার করবেন / করবেন না

```
✅ অবশ্যই ব্যবহার করবেন:
┌──────────────────────────────────────────────────────┐
│ • পেমেন্ট প্রসেসিং (bKash, SSLCommerz, Stripe)      │
│ • অর্ডার তৈরি (Daraz, Foodpanda)                     │
│ • ফান্ড ট্রান্সফার (ব্যাংক → ব্যাংক)                 │
│ • SMS/Email পাঠানো (ডুপ্লিকেট notification আটকাতে)   │
│ • রাইড বুকিং (Pathao, Uber)                         │
│ • Inventory আপডেট (স্টক কমানো/বাড়ানো)               │
│ • সাবস্ক্রিপশন তৈরি/বাতিল                           │
│ • Webhook processing (তৃতীয় পক্ষ থেকে আসা event)    │
└──────────────────────────────────────────────────────┘

❌ দরকার নেই:
┌──────────────────────────────────────────────────────┐
│ • ডেটা পড়া (GET requests)                           │
│ • Search / Filter queries                            │
│ • Logging / Analytics event                          │
│ • Cache invalidation                                 │
│ • Health check endpoints                             │
│ • Static file serving                                │
│ • PUT দিয়ে সম্পূর্ণ resource replace                  │
│   (PUT নিজেই idempotent)                             │
└──────────────────────────────────────────────────────┘
```

### সারসংক্ষেপ Decision Tree

```
                    নতুন API endpoint?
                          │
                    ┌─────┴──────┐
                    │            │
              State পরিবর্তন   শুধু Read
              করে? (POST/PATCH)  (GET/HEAD)
                    │            │
                    │            └──→ ইডেম্পোটেন্সি
                    │                 দরকার নেই ✅
                    ▼
              আর্থিক লেনদেন
              বা গুরুত্বপূর্ণ
              অপারেশন?
                    │
              ┌─────┴──────┐
              │            │
             হ্যাঁ         না
              │            │
              ▼            ▼
         Idempotency    ডুপ্লিকেট
         Key + Store    কি সমস্যা
         বাধ্যতামূলক    তৈরি করবে?
                          │
                    ┌─────┴──────┐
                    │            │
                   হ্যাঁ         না
                    │            │
                    ▼            ▼
              Idempotency   Optional
              Key ব্যবহার   বা Database
              করুন         Constraint
                           যথেষ্ট
```

---

## 📚 আরও পড়ুন

- [Stripe Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [HTTP Specification — Idempotent Methods (RFC 7231)](https://tools.ietf.org/html/rfc7231#section-4.2.2)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)

---

> **"ইডেম্পোটেন্সি হলো distributed system-এর সিটবেল্ট — দুর্ঘটনা ঘটার আগেই পরে নিতে হয়।"** 🔄
