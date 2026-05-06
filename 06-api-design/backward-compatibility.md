# 🔄 Backward Compatibility & API Evolution

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Breaking vs Non-Breaking Changes](#breaking-vs-non-breaking-changes)
- [Postel's Law](#postels-law)
- [API Versioning Strategies](#api-versioning-strategies)
- [Deprecation Workflow](#deprecation-workflow)
- [Consumer-Driven Contracts](#consumer-driven-contracts)
- [Schema Evolution](#schema-evolution)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Backward Compatibility** হলো এমন একটি নীতি যেখানে নতুন version এর API পুরানো client দের সাথে সঠিকভাবে কাজ করতে পারে। অর্থাৎ, আপনি API তে নতুন feature যোগ করলেও পুরানো consumer রা কোনো সমস্যা ছাড়াই তাদের কাজ চালিয়ে যেতে পারবে।

**API Evolution** হলো সময়ের সাথে সাথে API কে উন্নত করার প্রক্রিয়া — নতুন feature যোগ করা, পুরানো feature বাদ দেওয়া, এবং performance উন্নত করা — সবই consumer দের ক্ষতি না করে।

### মূল নীতিগুলো:
- ✅ নতুন field যোগ করা নিরাপদ (Additive Change)
- ❌ বিদ্যমান field মুছে ফেলা বিপজ্জনক (Breaking Change)
- ⚠️ field rename করা breaking change
- ✅ নতুন endpoint যোগ করা নিরাপদ
- ❌ বিদ্যমান endpoint এর response structure পরিবর্তন করা বিপজ্জনক

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏦 bKash API Evolution (v1 → v3)

bKash যখন প্রথম তাদের Payment API চালু করেছিল, তখন শুধু basic payment ছিল। সময়ের সাথে সাথে তারা নতুন feature যোগ করেছে:

```
bKash API Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2018 (v1): Basic Send Money, Payment
2020 (v2): + Tokenized Payment, + Subscription, + Refund
2023 (v3): + QR Payment, + Split Payment, + Scheduled Payment
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**সমস্যা:** হাজার হাজার merchant v1 ব্যবহার করছে। তারা যদি হঠাৎ v1 বন্ধ করে দেয়, তাহলে সব merchant এর integration ভেঙে যাবে।

**সমাধান:** Backward compatible evolution — v1 চালু রেখেই v2, v3 release করা।

### 🚗 Pathao API উদাহরণ

Pathao তাদের Ride API তে driver location tracking যোগ করেছে:

```
// v1 Response (পুরানো)
{
  "ride_id": "R123",
  "status": "ongoing",
  "driver_phone": "01712345678"
}

// v2 Response (নতুন — backward compatible!)
{
  "ride_id": "R123",
  "status": "ongoing",
  "driver_phone": "01712345678",
  "driver_location": {          // ← নতুন field (additive)
    "lat": 23.8103,
    "lng": 90.4125
  },
  "estimated_arrival": "5 min"  // ← নতুন field (additive)
}
```

পুরানো client শুধু `ride_id`, `status`, `driver_phone` পড়বে — নতুন field গুলো ignore করবে। কোনো সমস্যা নেই! ✅

---

## ⚡ Breaking vs Non-Breaking Changes

### ASCII Diagram: Change Types

```
┌─────────────────────────────────────────────────────────────┐
│                    API Changes Classification                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ NON-BREAKING (Safe)          ❌ BREAKING (Dangerous)    │
│  ─────────────────────           ──────────────────────     │
│  • নতুন field যোগ                • field মুছে ফেলা          │
│  • নতুন endpoint যোগ             • field rename করা         │
│  • নতুন optional parameter       • type পরিবর্তন            │
│  • নতুন enum value যোগ           • required field যোগ       │
│  • response এ data যোগ           • URL path পরিবর্তন        │
│  • নতুন HTTP method support      • HTTP method পরিবর্তন     │
│  • error message উন্নত            • status code পরিবর্তন     │
│  • performance improvement       • authentication পরিবর্তন  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Breaking Change এর প্রভাব:

```
┌──────────┐         ┌──────────┐         ┌──────────────┐
│  bKash   │         │   API    │         │   Merchant   │
│  Server  │────────▶│ Gateway  │────────▶│   (Daraz)    │
└──────────┘         └──────────┘         └──────────────┘
     │                                          │
     │  Breaking Change!                        │
     │  "amount" → "total_amount"               │
     ▼                                          ▼
┌──────────┐                             ┌──────────────┐
│  নতুন    │                             │   ERROR!     │
│ Response │                             │ "amount"     │
│  পাঠানো  │                             │  field নেই!  │
└──────────┘                             └──────────────┘
```

---

## 📜 Postel's Law (Robustness Principle)

> **"Be conservative in what you send, be liberal in what you accept"**
> — Jon Postel (RFC 761)

### বাংলায়:
> **"যা পাঠাবেন তাতে কঠোর হন, যা গ্রহণ করবেন তাতে উদার হন"**

### ব্যাখ্যা:

```
┌─────────────────────────────────────────────────────┐
│              Postel's Law in Practice                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  API Server (Conservative in Sending):              │
│  ─────────────────────────────────────              │
│  • সবসময় documented format এ response দিন          │
│  • extra field যোগ করলেও structure ঠিক রাখুন       │
│  • null এর বদলে default value দিন                   │
│                                                     │
│  API Client (Liberal in Accepting):                 │
│  ─────────────────────────────────────              │
│  • অচেনা field ignore করুন                          │
│  • missing optional field handle করুন              │
│  • extra data তে crash করবেন না                    │
│  • flexible parsing ব্যবহার করুন                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Grameenphone API Example:

```
// GP যা পাঠায় (Conservative):
{
  "subscriber": "01711234567",
  "balance": 150.50,
  "currency": "BDT"      // সবসময় include করে
}

// Client যা accept করে (Liberal):
// - "bonus_balance" field থাকলেও সমস্যা নেই
// - "currency" না থাকলেও default "BDT" ধরে নেয়
// - extra whitespace, different casing handle করে
```

---

## 🔢 API Versioning Strategies

### Strategy Comparison Diagram:

```
┌────────────────────────────────────────────────────────────────────┐
│                  API Versioning Strategies তুলনা                    │
├──────────────┬──────────────┬───────────────┬─────────────────────┤
│   Strategy   │  Complexity  │  Cacheability │    Use Case          │
├──────────────┼──────────────┼───────────────┼─────────────────────┤
│ URL Path     │    Low ⭐     │   High ⭐⭐⭐   │ Public API (bKash)  │
│ /api/v1/pay  │              │               │                     │
├──────────────┼──────────────┼───────────────┼─────────────────────┤
│ Query Param  │    Low ⭐     │   Medium ⭐⭐  │ Simple APIs         │
│ ?version=1   │              │               │                     │
├──────────────┼──────────────┼───────────────┼─────────────────────┤
│ Header       │   Medium ⭐⭐  │   Low ⭐      │ Internal APIs       │
│ X-API-Ver: 1 │              │               │ (Pathao microserv.) │
├──────────────┼──────────────┼───────────────┼─────────────────────┤
│ Content      │   High ⭐⭐⭐  │   Low ⭐      │ Enterprise APIs     │
│ Negotiation  │              │               │ (Banking APIs)      │
└──────────────┴──────────────┴───────────────┴─────────────────────┘
```

### 1️⃣ URL Path Versioning:
```
GET /api/v1/payments/create
GET /api/v2/payments/create
GET /api/v3/payments/create
```

### 2️⃣ Query Parameter Versioning:
```
GET /api/payments/create?version=1
GET /api/payments/create?version=2
```

### 3️⃣ Header Versioning:
```
GET /api/payments/create
Header: X-API-Version: 2
Header: Accept-Version: v2
```

### 4️⃣ Content Negotiation:
```
GET /api/payments/create
Accept: application/vnd.bkash.v2+json
```

---

## 🚨 Deprecation Workflow

### Deprecation Timeline:

```
┌─────────────────────────────────────────────────────────────────┐
│                    API Deprecation Workflow                       │
│                                                                 │
│  Phase 1          Phase 2          Phase 3         Phase 4      │
│  ────────         ────────         ────────        ────────     │
│  Announce         Deprecation      Migration       Sunset       │
│                   Warning          Period                        │
│                                                                 │
│  ┌───┐           ┌───┐           ┌───┐           ┌───┐        │
│  │ 📢│──────────▶│ ⚠️│──────────▶│ 🔄│──────────▶│ 🌅│        │
│  └───┘           └───┘           └───┘           └───┘        │
│   │               │               │               │            │
│   │ 6 months      │ 3 months      │ 3 months      │            │
│   │ before        │ before        │ before        │ API        │
│   │               │               │               │ বন্ধ       │
│   ▼               ▼               ▼               ▼            │
│  Blog post       Sunset header   Email alerts    404/410       │
│  Email notice    Response warn   Dashboard       Response      │
│  Docs update     Log warnings    Direct calls                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sunset Header Example:

```
HTTP/1.1 200 OK
Sunset: Sat, 01 Jun 2025 00:00:00 GMT
Deprecation: true
Link: <https://api.bkash.com/migration-guide>; rel="deprecation"
X-API-Warn: "v1 will be removed on 2025-06-01. Please migrate to v3."
```

---

## 🤝 Consumer-Driven Contracts

### ধারণা:

Consumer-Driven Contracts (CDC) হলো এমন একটি pattern যেখানে API consumer রা তাদের expectations define করে, এবং provider সেগুলো satisfy করছে কিনা তা verify করে।

```
┌──────────────┐     Contract      ┌──────────────┐
│              │    ┌────────┐     │              │
│   Daraz      │───▶│  CDC   │◀────│   bKash      │
│  (Consumer)  │    │  Test  │     │  (Provider)  │
│              │    └────────┘     │              │
└──────────────┘         │        └──────────────┘
                         │
┌──────────────┐         │        ┌──────────────┐
│   Prothom    │─────────┘        │              │
│    Alo       │                  │   CI/CD      │
│  (Consumer)  │                  │  Pipeline    │
└──────────────┘                  └──────────────┘
                                         │
                                         ▼
                                  ┌──────────────┐
                                  │  All tests   │
                                  │   pass? ✅   │
                                  │  Deploy! 🚀  │
                                  └──────────────┘
```

---

## 🧬 Schema Evolution

### Avro / Protobuf Compatibility:

```
┌────────────────────────────────────────────────────────────┐
│                Schema Compatibility Types                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  BACKWARD Compatible:                                      │
│  ─────────────────────                                     │
│  নতুন schema পুরানো data পড়তে পারে                        │
│  Example: নতুন optional field যোগ (with default)           │
│                                                            │
│  FORWARD Compatible:                                       │
│  ─────────────────────                                     │
│  পুরানো schema নতুন data পড়তে পারে                        │
│  Example: field remove করা (যদি optional হয়)              │
│                                                            │
│  FULL Compatible:                                          │
│  ─────────────────────                                     │
│  উভয় দিকেই compatible                                     │
│  Example: optional field যোগ/বাদ (with defaults)           │
│                                                            │
│  NONE:                                                     │
│  ─────────────────────                                     │
│  কোনো guarantee নেই                                        │
│  Example: field type পরিবর্তন (int → string)              │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Backward Compatible API Controller:

```php
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

/**
 * bKash Payment API - Backward Compatible Implementation
 * v1, v2, v3 সবই support করে
 */
class PaymentController extends Controller
{
    /**
     * API Version Detection
     * Multiple versioning strategies support করে
     */
    private function detectApiVersion(Request $request): int
    {
        // Strategy 1: URL Path (/api/v2/payments)
        $urlVersion = $request->segment(2); // "v1", "v2", "v3"
        if ($urlVersion && preg_match('/^v(\d+)$/', $urlVersion, $matches)) {
            return (int) $matches[1];
        }

        // Strategy 2: Header (X-API-Version: 2)
        $headerVersion = $request->header('X-API-Version');
        if ($headerVersion) {
            return (int) $headerVersion;
        }

        // Strategy 3: Query Parameter (?version=2)
        $queryVersion = $request->query('version');
        if ($queryVersion) {
            return (int) $queryVersion;
        }

        // Strategy 4: Content Negotiation
        $accept = $request->header('Accept', '');
        if (preg_match('/application\/vnd\.bkash\.v(\d+)\+json/', $accept, $matches)) {
            return (int) $matches[1];
        }

        // Default: latest stable version
        return 2;
    }

    /**
     * Payment Create - সব version support করে
     */
    public function createPayment(Request $request): JsonResponse
    {
        $version = $this->detectApiVersion($request);

        // Postel's Law: Liberal in accepting
        $amount = $request->input('amount') 
                  ?? $request->input('total_amount')  // v2 name
                  ?? $request->input('payment_amount'); // v3 name

        $phone = $request->input('phone')
                 ?? $request->input('sender_phone')
                 ?? $request->input('customer_phone');

        // Process payment
        $payment = $this->processPayment($amount, $phone);

        // Postel's Law: Conservative in sending (version-specific response)
        return match($version) {
            1 => $this->v1Response($payment),
            2 => $this->v2Response($payment),
            3 => $this->v3Response($payment),
            default => $this->v3Response($payment),
        };
    }

    /**
     * v1 Response - শুধু basic fields
     */
    private function v1Response(array $payment): JsonResponse
    {
        return response()->json([
            'payment_id' => $payment['id'],
            'amount' => $payment['amount'],
            'status' => $payment['status'],
            'trx_id' => $payment['transaction_id'],
        ], 200, $this->deprecationHeaders(1));
    }

    /**
     * v2 Response - + tokenization, refund info
     */
    private function v2Response(array $payment): JsonResponse
    {
        return response()->json([
            'payment_id' => $payment['id'],
            'amount' => $payment['amount'],
            'currency' => 'BDT',
            'status' => $payment['status'],
            'transaction_id' => $payment['transaction_id'],
            'token' => $payment['token'] ?? null,
            'refundable' => $payment['refundable'] ?? true,
            'created_at' => $payment['created_at'],
        ]);
    }

    /**
     * v3 Response - + split payment, QR, scheduling
     */
    private function v3Response(array $payment): JsonResponse
    {
        return response()->json([
            'data' => [
                'payment_id' => $payment['id'],
                'amount' => [
                    'value' => $payment['amount'],
                    'currency' => 'BDT',
                ],
                'status' => $payment['status'],
                'transaction_id' => $payment['transaction_id'],
                'token' => $payment['token'] ?? null,
                'splits' => $payment['splits'] ?? [],
                'qr_code' => $payment['qr_code'] ?? null,
                'scheduled_at' => $payment['scheduled_at'] ?? null,
                'metadata' => $payment['metadata'] ?? [],
            ],
            'links' => [
                'self' => "/api/v3/payments/{$payment['id']}",
                'refund' => "/api/v3/payments/{$payment['id']}/refund",
                'receipt' => "/api/v3/payments/{$payment['id']}/receipt",
            ],
        ]);
    }

    /**
     * Deprecation Headers - পুরানো version ব্যবহারকারীদের জানানো
     */
    private function deprecationHeaders(int $version): array
    {
        if ($version >= 3) {
            return [];
        }

        $sunsetDates = [
            1 => 'Sat, 01 Mar 2025 00:00:00 GMT',
            2 => 'Sat, 01 Dec 2025 00:00:00 GMT',
        ];

        return [
            'Sunset' => $sunsetDates[$version] ?? '',
            'Deprecation' => 'true',
            'X-API-Warn' => "Version {$version} is deprecated. Please migrate to v3.",
            'Link' => '<https://developer.bkash.com/migration>; rel="deprecation"',
        ];
    }

    private function processPayment(float $amount, string $phone): array
    {
        // Payment processing logic...
        return [
            'id' => 'PAY-' . uniqid(),
            'amount' => $amount,
            'status' => 'completed',
            'transaction_id' => 'TRX-' . strtoupper(bin2hex(random_bytes(8))),
            'token' => bin2hex(random_bytes(16)),
            'refundable' => true,
            'created_at' => now()->toISOString(),
            'splits' => [],
            'qr_code' => null,
            'scheduled_at' => null,
            'metadata' => [],
        ];
    }
}

/**
 * Feature Flags for API Versions
 * কোন version active, deprecated, বা sunset হয়েছে তা manage করে
 */
class ApiVersionManager
{
    private array $versions = [
        1 => ['status' => 'sunset', 'sunset_date' => '2025-03-01'],
        2 => ['status' => 'deprecated', 'sunset_date' => '2025-12-01'],
        3 => ['status' => 'active', 'sunset_date' => null],
    ];

    public function isVersionActive(int $version): bool
    {
        return isset($this->versions[$version]) 
            && $this->versions[$version]['status'] !== 'sunset';
    }

    public function getActiveVersions(): array
    {
        return array_filter($this->versions, fn($v) => $v['status'] === 'active');
    }

    /**
     * Version Usage Monitoring
     * কোন version কতটা ব্যবহৃত হচ্ছে তা track করে
     */
    public function trackUsage(int $version, string $consumerId): void
    {
        // Redis-based tracking
        $key = "api_version_usage:{$version}:" . date('Y-m-d');
        // Redis::hincrby($key, $consumerId, 1);
        // Redis::expire($key, 86400 * 90); // 90 days retention
    }

    /**
     * Migration Progress Report
     */
    public function getMigrationReport(): array
    {
        return [
            'v1_consumers' => 12,    // এখনো v1 ব্যবহার করছে
            'v2_consumers' => 856,   // v2 ব্যবহার করছে
            'v3_consumers' => 2341,  // v3 তে migrate করেছে
            'migration_rate' => '73%',
            'target_sunset_v2' => '2025-12-01',
        ];
    }
}

/**
 * Gradual Migration Middleware
 * ধীরে ধীরে traffic নতুন version এ redirect করে
 */
class GradualMigrationMiddleware
{
    public function handle(Request $request, \Closure $next)
    {
        $version = $this->detectVersion($request);
        $consumer = $request->header('X-Consumer-Id', 'unknown');

        // Sunset হয়ে গেছে — 410 Gone
        if ($version === 1) {
            return response()->json([
                'error' => 'API_VERSION_SUNSET',
                'message' => 'API v1 is no longer available. Please use v3.',
                'migration_guide' => 'https://developer.bkash.com/migrate/v1-to-v3',
                'support' => 'developer-support@bkash.com',
            ], 410);
        }

        // Deprecated — warning header সহ response
        if ($version === 2) {
            $response = $next($request);
            $response->headers->set('X-API-Warn', 'v2 deprecated. Migrate to v3 by 2025-12-01.');
            $response->headers->set('Sunset', 'Mon, 01 Dec 2025 00:00:00 GMT');

            // Usage log করা — কে এখনো পুরানো version ব্যবহার করছে
            logger()->info('Deprecated API usage', [
                'version' => $version,
                'consumer' => $consumer,
                'endpoint' => $request->path(),
            ]);

            return $response;
        }

        return $next($request);
    }

    private function detectVersion(Request $request): int
    {
        // URL path থেকে version detect
        if (preg_match('/\/api\/v(\d+)\//', $request->path(), $matches)) {
            return (int) $matches[1];
        }
        return 3;
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Backward Compatible API Client (Consumer Side):

```javascript
/**
 * bKash API Client - Backward Compatible
 * Postel's Law অনুসরণ করে: Liberal in accepting
 */
class BkashApiClient {
    constructor(config) {
        this.baseUrl = config.baseUrl || 'https://api.bkash.com';
        this.version = config.version || 3;
        this.apiKey = config.apiKey;
        this.timeout = config.timeout || 30000;
    }

    /**
     * Payment তৈরি করা — সব version handle করে
     */
    async createPayment(paymentData) {
        const url = `${this.baseUrl}/api/v${this.version}/payments`;
        
        const response = await fetch(url, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.apiKey}`,
                'X-API-Version': String(this.version),
                'Accept': `application/vnd.bkash.v${this.version}+json`,
            },
            body: JSON.stringify(paymentData),
        });

        // Deprecation warning check করা
        this.checkDeprecationWarnings(response);

        const data = await response.json();

        // Postel's Law: Liberal in accepting
        // যেকোনো version এর response normalize করে
        return this.normalizePaymentResponse(data);
    }

    /**
     * Response Normalization
     * v1, v2, v3 — যেকোনো version এর response একই format এ আনা
     */
    normalizePaymentResponse(data) {
        // v3 response structure: { data: { ... }, links: { ... } }
        if (data.data && data.data.payment_id) {
            return {
                paymentId: data.data.payment_id,
                amount: data.data.amount?.value || data.data.amount,
                currency: data.data.amount?.currency || 'BDT',
                status: data.data.status,
                transactionId: data.data.transaction_id,
                token: data.data.token || null,
            };
        }

        // v2 response: { payment_id, amount, currency, ... }
        if (data.payment_id && data.currency) {
            return {
                paymentId: data.payment_id,
                amount: data.amount,
                currency: data.currency,
                status: data.status,
                transactionId: data.transaction_id,
                token: data.token || null,
            };
        }

        // v1 response: { payment_id, amount, status, trx_id }
        return {
            paymentId: data.payment_id,
            amount: data.amount,
            currency: 'BDT', // v1 তে currency ছিল না — default ধরে নিচ্ছি
            status: data.status,
            transactionId: data.trx_id || data.transaction_id,
            token: null,
        };
    }

    /**
     * Deprecation Warning Handler
     * Server থেকে deprecation notice আসলে alert করা
     */
    checkDeprecationWarnings(response) {
        const sunset = response.headers.get('Sunset');
        const deprecation = response.headers.get('Deprecation');
        const warning = response.headers.get('X-API-Warn');

        if (deprecation === 'true' || sunset) {
            console.warn(`⚠️ API Deprecation Warning!`);
            console.warn(`  Message: ${warning}`);
            console.warn(`  Sunset Date: ${sunset}`);
            console.warn(`  Migration: https://developer.bkash.com/migration`);

            // Monitoring system এ alert পাঠানো
            this.sendDeprecationAlert({
                version: this.version,
                sunsetDate: sunset,
                message: warning,
            });
        }
    }

    async sendDeprecationAlert(details) {
        // Slack/Teams/Email notification
        console.log('📧 Deprecation alert sent to engineering team:', details);
    }
}

/**
 * Consumer-Driven Contract Test
 * bKash API response আমাদের expectation মেটাচ্ছে কিনা verify করা
 */
class ConsumerContractTest {
    constructor(apiClient) {
        this.client = apiClient;
    }

    /**
     * Daraz এর payment contract verify
     * Daraz এর জন্য যে fields দরকার সেগুলো আছে কিনা check
     */
    async verifyDarazPaymentContract() {
        const testPayment = {
            amount: 1500,
            currency: 'BDT',
            customer_phone: '01712345678',
            merchant_id: 'DARAZ_001',
            order_id: 'ORD-2024-001',
        };

        const result = await this.client.createPayment(testPayment);

        // Contract assertions
        const requiredFields = ['paymentId', 'amount', 'status', 'transactionId'];
        const missingFields = requiredFields.filter(field => !(field in result));

        if (missingFields.length > 0) {
            throw new Error(
                `❌ Contract Violation! Missing fields: ${missingFields.join(', ')}\n` +
                `   bKash API response does not satisfy Daraz consumer contract.`
            );
        }

        // Type checks
        if (typeof result.amount !== 'number') {
            throw new Error('❌ Contract Violation! "amount" must be a number.');
        }

        if (!['pending', 'completed', 'failed'].includes(result.status)) {
            throw new Error(`❌ Contract Violation! Unknown status: "${result.status}"`);
        }

        console.log('✅ Daraz consumer contract satisfied!');
        return true;
    }
}

/**
 * API Version Feature Flags (Client Side)
 * Feature flag দিয়ে নতুন version এ gradually migrate করা
 */
class ApiFeatureFlags {
    constructor() {
        this.flags = {
            'use_v3_payments': { enabled: true, rollout: 100 },
            'use_v3_refunds': { enabled: true, rollout: 75 },
            'use_v3_subscriptions': { enabled: false, rollout: 0 },
            'use_v3_split_payments': { enabled: true, rollout: 50 },
        };
    }

    /**
     * Feature flag check করা
     * Percentage-based rollout support করে
     */
    isEnabled(flagName, userId = null) {
        const flag = this.flags[flagName];
        if (!flag || !flag.enabled) return false;

        if (flag.rollout >= 100) return true;
        if (flag.rollout <= 0) return false;

        // User-based consistent rollout
        if (userId) {
            const hash = this.hashCode(userId + flagName);
            return (hash % 100) < flag.rollout;
        }

        return Math.random() * 100 < flag.rollout;
    }

    /**
     * কোন API version ব্যবহার করবে তা decide করা
     */
    getApiVersion(feature, userId) {
        const flagName = `use_v3_${feature}`;
        return this.isEnabled(flagName, userId) ? 3 : 2;
    }

    hashCode(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash |= 0;
        }
        return Math.abs(hash);
    }
}

/**
 * Usage Monitoring - পুরানো version কে ব্যবহার করছে track করা
 */
class ApiVersionMonitor {
    constructor() {
        this.usageLog = new Map();
    }

    /**
     * Version usage record করা
     */
    recordUsage(version, consumerId, endpoint) {
        const key = `${version}:${consumerId}:${endpoint}`;
        const current = this.usageLog.get(key) || { count: 0, lastUsed: null };
        current.count++;
        current.lastUsed = new Date().toISOString();
        this.usageLog.set(key, current);
    }

    /**
     * Migration report generate করা
     * কোন consumer এখনো পুরানো version ব্যবহার করছে
     */
    generateMigrationReport() {
        const report = {
            generatedAt: new Date().toISOString(),
            versions: {},
        };

        for (const [key, data] of this.usageLog) {
            const [version] = key.split(':');
            if (!report.versions[version]) {
                report.versions[version] = { totalCalls: 0, consumers: new Set() };
            }
            report.versions[version].totalCalls += data.count;
            report.versions[version].consumers.add(key.split(':')[1]);
        }

        console.log('📊 API Version Usage Report:');
        console.log('═══════════════════════════════════════');
        for (const [ver, data] of Object.entries(report.versions)) {
            const status = ver === '3' ? '✅ Active' : ver === '2' ? '⚠️ Deprecated' : '🚫 Sunset';
            console.log(`  v${ver} ${status}: ${data.totalCalls} calls, ${data.consumers.size} consumers`);
        }
        console.log('═══════════════════════════════════════');

        return report;
    }
}

// ব্যবহার:
const client = new BkashApiClient({
    baseUrl: 'https://api.bkash.com',
    version: 3,
    apiKey: 'your-api-key',
});

const featureFlags = new ApiFeatureFlags();
const monitor = new ApiVersionMonitor();

// Daraz payment integration
async function processOrder(orderId, userId) {
    const version = featureFlags.getApiVersion('payments', userId);
    
    monitor.recordUsage(version, 'DARAZ', '/payments');

    const payment = await client.createPayment({
        amount: 2500,
        customer_phone: '01712345678',
        order_id: orderId,
    });

    console.log(`✅ Payment created: ${payment.transactionId}`);
    return payment;
}
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Backward Compatibility দরকার:

| পরিস্থিতি | কারণ |
|-----------|------|
| Public API (bKash, Pathao) | হাজার হাজার third-party consumer |
| Mobile App API | App update force করা যায় না |
| Microservices communication | একসাথে সব service deploy হয় না |
| Partner integrations (Daraz + bKash) | Partner দের update time দিতে হয় |
| Enterprise B2B APIs | Contract ভাঙলে legal issue হতে পারে |

### ❌ কখন এত strict হওয়ার দরকার নেই:

| পরিস্থিতি | কারণ |
|-----------|------|
| Internal team API (same deploy) | একসাথে deploy হয় |
| Early stage startup (Prothom Alo নতুন feature) | Consumer কম, দ্রুত iterate করা দরকার |
| Pre-release / Beta API | Consumer জানে যে breaking change হতে পারে |
| GraphQL (intrinsically evolvable) | Field addition breaking নয় |
| Event-driven systems (Kafka) | Schema registry handle করে |

### ⚖️ Best Practices সারসংক্ষেপ:

```
┌─────────────────────────────────────────────────────────┐
│           Backward Compatibility Checklist               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ DO:                                                 │
│  • নতুন field যোগ করুন, পুরানো রাখুন                    │
│  • Deprecation header ব্যবহার করুন                      │
│  • Migration guide লিখুন                               │
│  • Usage monitoring করুন                                │
│  • Sunset date আগে থেকে জানান                          │
│  • Consumer contract test লিখুন                        │
│  • Feature flags দিয়ে gradual rollout করুন             │
│                                                         │
│  ❌ DON'T:                                              │
│  • হঠাৎ field মুছবেন না                                 │
│  • Notice না দিয়ে breaking change করবেন না             │
│  • সব consumer একসাথে migrate করতে বাধ্য করবেন না       │
│  • Sunset date ignore করবেন না                         │
│  • Version monitoring ছাড়া old version বন্ধ করবেন না   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📚 সংক্ষিপ্ত রেফারেন্স

| বিষয় | মনে রাখার কথা |
|-------|---------------|
| Postel's Law | পাঠাতে কঠোর, গ্রহণে উদার |
| Additive Change | নতুন field যোগ = নিরাপদ ✅ |
| Breaking Change | field মুছা/rename = বিপজ্জনক ❌ |
| Sunset Header | কবে বন্ধ হবে HTTP header এ জানান |
| CDC | Consumer expectations verify করুন |
| Feature Flags | ধীরে ধীরে নতুন version এ migrate করুন |
| Monitoring | কে পুরানো version ব্যবহার করছে track করুন |

---

*"ভালো API evolution মানে — নতুন feature আনা, পুরানো consumer ভাঙা নয়।"* 🚀
