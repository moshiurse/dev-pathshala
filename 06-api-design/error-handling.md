# ⚠️ এরর হ্যান্ডলিং (API Error Handling)

> **"একটি ভালো API শুধু সফল রেসপন্স দেয় না — ব্যর্থতাকেও সুন্দরভাবে উপস্থাপন করে।"**

---

## 📌 সংজ্ঞা ও গুরুত্ব

API Error Handling হলো এমন একটি systematic approach যেখানে API-তে ঘটা প্রতিটি ব্যর্থতা বা সমস্যাকে **consistent, predictable এবং actionable** ফরম্যাটে ক্লায়েন্টের কাছে পৌঁছানো হয়। এটি শুধু HTTP status code রিটার্ন করা নয় — এটি একটি সম্পূর্ণ error communication strategy।

### কেন Consistent Error Response গুরুত্বপূর্ণ?

১. **ডেভেলপার এক্সপেরিয়েন্স (DX):** ফ্রন্টএন্ড ডেভেলপাররা যখন জানে যে সব এরর একই ফরম্যাটে আসবে, তখন তারা একটি unified error handler লিখতে পারে। বারবার if-else লেখার প্রয়োজন হয় না।

২. **ডিবাগিং গতি:** Production-এ সমস্যা হলে structured error response থেকে দ্রুত root cause বোঝা যায়। bKash বা Nagad-এর মতো পেমেন্ট API-তে এটি অত্যন্ত জরুরি — একটি ভুল error message পুরো transaction flow ভেঙে দিতে পারে।

৩. **Security:** অসাবধানে stack trace বা database error ক্লায়েন্টে পাঠালে attacker-দের জন্য তথ্য ফাঁস হয়।

৪. **Automation:** CI/CD, monitoring tools (Sentry, Datadog) — সবাই structured error দিয়ে কাজ করে। Unstructured error গুলো alert system-কে বিভ্রান্ত করে।

### এরর ক্যাটাগরি (Error Categories)

```
┌─────────────────────────────────────────────────────┐
│              Error Categories                        │
├─────────────────┬───────────────────────────────────┤
│  Client Errors  │  ক্লায়েন্টের ভুল ইনপুট/অনুরোধ   │
│  (4xx)          │  → Validation, Auth, Not Found     │
├─────────────────┼───────────────────────────────────┤
│  Server Errors  │  সার্ভারের অভ্যন্তরীণ সমস্যা      │
│  (5xx)          │  → DB down, timeout, bug           │
├─────────────────┼───────────────────────────────────┤
│  Business Logic │  ব্যবসায়িক নিয়ম লঙ্ঘন             │
│  Errors         │  → Insufficient balance, limit     │
├─────────────────┼───────────────────────────────────┤
│  External       │  তৃতীয় পক্ষের সেবায় সমস্যা       │
│  Service Errors │  → Payment gateway, SMS API down   │
└─────────────────┴───────────────────────────────────┘
```

---

## 📊 Error Response Architecture

```
                    ┌──────────────┐
                    │   Client     │
                    │  (Mobile/Web)│
                    └──────┬───────┘
                           │ HTTP Request
                           ▼
                    ┌──────────────┐
                    │   API Gateway│ ← Rate Limit (429)
                    │   / Nginx    │ ← Bad Gateway (502)
                    └──────┬───────┘
                           │
                           ▼
               ┌───────────────────────┐
               │   Middleware Layer     │
               │                       │
               │  ┌─────────────────┐  │
               │  │ Auth Middleware  │──┼── 401 Unauthorized
               │  └─────────────────┘  │   403 Forbidden
               │  ┌─────────────────┐  │
               │  │ Validation      │──┼── 422 Unprocessable
               │  └─────────────────┘  │   400 Bad Request
               │  ┌─────────────────┐  │
               │  │ Rate Limiter    │──┼── 429 Too Many Requests
               │  └─────────────────┘  │
               └───────────┬───────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │   Controller Layer    │
               │                       │
               │  Business Logic ──────┼── 409 Conflict
               │  Resource CRUD ───────┼── 404 Not Found
               │  External API Call ───┼── 502/503 (upstream fail)
               └───────────┬───────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │  Global Error Handler │
               │                       │
               │  ┌─────────────────┐  │
               │  │ Catch Exception │  │
               │  │ Map to Response │  │
               │  │ Log + Monitor   │  │
               │  │ Sanitize Output │  │
               │  └─────────────────┘  │
               └───────────┬───────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │  Structured Response  │
               │  {                    │
               │    "status": 422,     │
               │    "code": "E2001",   │
               │    "message": "...",  │
               │    "errors": [...]    │
               │  }                    │
               └───────────────────────┘
```

এই architecture-এর মূল ধারণা হলো — **প্রতিটি layer-এ error ধরা হবে, কিন্তু final response তৈরি হবে একটি centralized handler থেকে।** এতে consistency বজায় থাকে।

---

## 💻 HTTP Status Codes Deep Dive

### 2xx — Success (সফল)

| Code | নাম | ব্যবহার | উদাহরণ |
|------|-----|---------|---------|
| `200` | OK | সফল GET, PUT, PATCH | `GET /api/users/1` — ইউজার ডেটা রিটার্ন |
| `201` | Created | নতুন রিসোর্স তৈরি | `POST /api/users` — নতুন ইউজার তৈরি হয়েছে |
| `202` | Accepted | অনুরোধ গ্রহণ করা হয়েছে, প্রসেসিং চলছে | বাল্ক ইমেইল পাঠানো, রিপোর্ট জেনারেশন |
| `204` | No Content | সফল কিন্তু কোনো body নেই | `DELETE /api/users/1` — ইউজার মুছে ফেলা হয়েছে |

```php
// Laravel — বিভিন্ন success response
// 200 — সাধারণ সফল রেসপন্স
return response()->json(['data' => $user], 200);

// 201 — রিসোর্স তৈরি
return response()->json(['data' => $user], 201)
    ->header('Location', "/api/users/{$user->id}");

// 202 — Async প্রসেস শুরু
return response()->json([
    'message' => 'রিপোর্ট তৈরি হচ্ছে, কিছুক্ষণ পর চেক করুন',
    'status_url' => "/api/reports/{$reportId}/status"
], 202);

// 204 — কোনো content নেই
return response()->noContent(); // 204
```

### 3xx — Redirect (পুনঃনির্দেশনা)

| Code | নাম | ব্যবহার |
|------|-----|---------|
| `301` | Moved Permanently | API endpoint স্থায়ীভাবে সরানো হয়েছে। ক্লায়েন্টকে নতুন URL ক্যাশ করতে হবে |
| `302` | Found (Temporary) | সাময়িক redirect। OAuth callback-এ বহুল ব্যবহৃত |
| `304` | Not Modified | ক্লায়েন্টের ক্যাশ এখনো valid, নতুন data পাঠানোর দরকার নেই |

```js
// Express — 304 Not Modified (ETag ভিত্তিক)
app.get('/api/products/:id', async (req, res) => {
    const product = await Product.findById(req.params.id);
    const etag = generateETag(product);

    if (req.headers['if-none-match'] === etag) {
        return res.status(304).end(); // ক্যাশ valid, data পাঠানোর দরকার নেই
    }

    res.set('ETag', etag).json({ data: product });
});
```

### 4xx — Client Error (ক্লায়েন্ট এরর)

এটি সবচেয়ে গুরুত্বপূর্ণ ক্যাটাগরি কারণ ক্লায়েন্টকে বলতে হবে **ঠিক কী ভুল হয়েছে এবং কীভাবে ঠিক করবে।**

| Code | নাম | কখন ব্যবহার করবেন |
|------|-----|-------------------|
| `400` | Bad Request | Malformed JSON, missing required headers, অবৈধ query param |
| `401` | Unauthorized | Authentication নেই বা token expired/invalid |
| `403` | Forbidden | Authenticated কিন্তু permission নেই |
| `404` | Not Found | রিসোর্স খুঁজে পাওয়া যায়নি |
| `405` | Method Not Allowed | `POST /api/users/1` যখন শুধু `GET`, `PUT` allowed |
| `409` | Conflict | Duplicate entry, optimistic locking failure, state conflict |
| `422` | Unprocessable Entity | JSON valid কিন্তু semantic ভুল — validation failure |
| `429` | Too Many Requests | Rate limit exceed হয়েছে |

```php
// Laravel — 401 vs 403 পার্থক্য বোঝা জরুরি
// 401: কে তুমি? (Identity অজানা)
if (!$request->bearerToken()) {
    return response()->json([
        'code'    => 'E1001',
        'message' => 'Authentication token প্রদান করুন'
    ], 401);
}

// 403: তোমাকে চিনি, কিন্তু তোমার অনুমতি নেই
if (!$user->can('delete', $post)) {
    return response()->json([
        'code'    => 'E1003',
        'message' => 'এই পোস্ট মুছে ফেলার অনুমতি আপনার নেই'
    ], 403);
}
```

```js
// Express — 409 Conflict উদাহরণ
app.post('/api/users', async (req, res, next) => {
    try {
        const existing = await User.findOne({ email: req.body.email });
        if (existing) {
            return res.status(409).json({
                code: 'E2005',
                message: 'এই ইমেইল দিয়ে ইতিমধ্যে একটি অ্যাকাউন্ট আছে',
                detail: `Email ${req.body.email} is already registered`,
                suggestion: 'লগইন করুন অথবা পাসওয়ার্ড রিসেট করুন'
            });
        }
        const user = await User.create(req.body);
        res.status(201).json({ data: user });
    } catch (err) {
        next(err);
    }
});

// Express — 429 Rate Limit
app.use('/api/', rateLimit({
    windowMs: 15 * 60 * 1000, // ১৫ মিনিট
    max: 100,
    handler: (req, res) => {
        res.status(429).json({
            code: 'E1010',
            message: 'অনুরোধের সীমা অতিক্রম করেছেন। কিছুক্ষণ পর চেষ্টা করুন।',
            retry_after: res.getHeader('Retry-After')
        });
    }
}));
```

### 5xx — Server Error (সার্ভার এরর)

| Code | নাম | পরিস্থিতি |
|------|-----|-----------|
| `500` | Internal Server Error | অপ্রত্যাশিত bug, unhandled exception |
| `502` | Bad Gateway | Upstream service (যেমন bKash API) ভুল response দিয়েছে |
| `503` | Service Unavailable | সার্ভার অস্থায়ীভাবে অক্ষম — maintenance, overloaded |
| `504` | Gateway Timeout | Upstream service সময়মতো response দেয়নি |

> ⚠️ **গুরুত্বপূর্ণ নিয়ম:** 5xx এররে কখনো internal details (stack trace, DB query, file path) ক্লায়েন্টে পাঠাবেন না। শুধু log-এ রাখুন।

```php
// Laravel — 503 Maintenance Mode
// 502 — Upstream failure
try {
    $response = Http::timeout(10)->post('https://tokenized.pay.bka.sh/v1.2.0-beta/checkout/payment/create', $payload);
    if ($response->failed()) {
        throw new UpstreamServiceException('bKash API response failed', $response->status());
    }
} catch (ConnectionException $e) {
    Log::critical('bKash API unreachable', ['error' => $e->getMessage()]);
    return response()->json([
        'code'    => 'E5002',
        'message' => 'পেমেন্ট সার্ভিসে সাময়িক সমস্যা হচ্ছে, কিছুক্ষণ পর আবার চেষ্টা করুন'
    ], 502);
}
```

### Status Code Decision Guide

```
ক্লায়েন্টের request কি valid JSON?
├── না → 400 Bad Request
└── হ্যাঁ
    ├── Authentication আছে?
    │   ├── না → 401 Unauthorized
    │   └── হ্যাঁ
    │       ├── Permission আছে?
    │       │   ├── না → 403 Forbidden
    │       │   └── হ্যাঁ
    │       │       ├── Resource খুঁজে পাওয়া গেছে?
    │       │       │   ├── না → 404 Not Found
    │       │       │   └── হ্যাঁ
    │       │       │       ├── HTTP Method সঠিক?
    │       │       │       │   ├── না → 405 Method Not Allowed
    │       │       │       │   └── হ্যাঁ
    │       │       │       │       ├── Validation পাস?
    │       │       │       │       │   ├── না → 422 Unprocessable Entity
    │       │       │       │       │   └── হ্যাঁ
    │       │       │       │       │       ├── Business rule conflict?
    │       │       │       │       │       │   ├── হ্যাঁ → 409 Conflict
    │       │       │       │       │       │   └── না
    │       │       │       │       │       │       ├── Rate limit?
    │       │       │       │       │       │       │   ├── হ্যাঁ → 429
    │       │       │       │       │       │       │   └── না
    │       │       │       │       │       │       │       ├── সার্ভার ঠিক আছে?
    │       │       │       │       │       │       │       │   ├── না → 500/503
    │       │       │       │       │       │       │       │   └── হ্যাঁ → 2xx ✅
```

---

## 🔧 Error Response Format

### RFC 7807 — Problem Details for HTTP APIs

RFC 7807 হলো IETF-এর একটি standard যা API error response-এর জন্য একটি নির্দিষ্ট structure দেয়। Content-Type হবে `application/problem+json`।

```json
{
    "type": "https://api.example.com/problems/validation-error",
    "title": "Validation Failed",
    "status": 422,
    "detail": "ফোন নম্বরটি সঠিক ফরম্যাটে নেই। বাংলাদেশী নম্বর +880 দিয়ে শুরু হতে হবে।",
    "instance": "/api/users/registration",
    "errors": [
        {
            "field": "phone",
            "message": "ফোন নম্বর +880XXXXXXXXXX ফরম্যাটে দিন"
        }
    ]
}
```

| ফিল্ড | বর্ণনা | আবশ্যক? |
|--------|--------|----------|
| `type` | সমস্যার ধরন বর্ণনা করা URI (মানুষের পড়ার উপযোগী ডকুমেন্টেশনের লিংক) | হ্যাঁ |
| `title` | সমস্যার সংক্ষিপ্ত বিবরণ (HTTP status থেকে আলাদা, human-readable) | হ্যাঁ |
| `status` | HTTP status code (response header-এর সাথে মিলবে) | হ্যাঁ |
| `detail` | এই নির্দিষ্ট ঘটনার বিস্তারিত ব্যাখ্যা | না |
| `instance` | সমস্যা ঘটা নির্দিষ্ট resource-এর URI | না |

### Custom Error Format

বাস্তবে অনেক API team নিজস্ব format তৈরি করে যা তাদের প্রয়োজন মেটায়:

```json
{
    "success": false,
    "code": "E2001",
    "message": "অনুরোধে ত্রুটি পাওয়া গেছে",
    "errors": [
        {
            "field": "amount",
            "code": "INVALID_RANGE",
            "message": "পরিমাণ ১০ থেকে ৫০,০০০ টাকার মধ্যে হতে হবে",
            "meta": {
                "min": 10,
                "max": 50000,
                "currency": "BDT"
            }
        },
        {
            "field": "receiver_phone",
            "code": "INVALID_FORMAT",
            "message": "গ্রাহকের ফোন নম্বরটি সঠিক নয়"
        }
    ],
    "meta": {
        "request_id": "req_abc123xyz",
        "timestamp": "2024-01-15T10:30:00+06:00",
        "documentation": "https://api.example.com/docs/errors/E2001"
    }
}
```

### Validation Error Format — Field-level Errors

Validation error-এ ক্লায়েন্টকে ঠিক কোন field-এ কী সমস্যা সেটা জানাতে হবে। এটি সবচেয়ে common pattern:

```json
{
    "status": 422,
    "code": "VALIDATION_FAILED",
    "message": "তথ্য যাচাইয়ে ত্রুটি পাওয়া গেছে",
    "errors": {
        "name": [
            "নাম আবশ্যক",
            "নাম কমপক্ষে ৩ অক্ষরের হতে হবে"
        ],
        "email": [
            "ইমেইল ঠিকানার ফরম্যাট সঠিক নয়"
        ],
        "address.city": [
            "শহরের নাম আবশ্যক"
        ],
        "items[0].quantity": [
            "পরিমাণ শূন্যের বেশি হতে হবে"
        ]
    }
}
```

### Laravel + Express — Standardized Error Response Implementation

```php
// Laravel — app/Http/Responses/ApiErrorResponse.php
<?php

namespace App\Http\Responses;

use Illuminate\Http\JsonResponse;

class ApiErrorResponse
{
    public static function make(
        string $code,
        string $message,
        int $status,
        array $errors = [],
        array $meta = []
    ): JsonResponse {
        $response = [
            'success' => false,
            'code'    => $code,
            'message' => $message,
        ];

        if (!empty($errors)) {
            $response['errors'] = $errors;
        }

        $response['meta'] = array_merge([
            'request_id' => request()->header('X-Request-ID', uniqid('req_')),
            'timestamp'  => now()->toIso8601String(),
        ], $meta);

        return response()->json($response, $status, [
            'Content-Type' => 'application/problem+json'
        ]);
    }
}
```

```js
// Express — utils/ApiErrorResponse.js
class ApiErrorResponse {
    constructor(code, message, status, errors = [], meta = {}) {
        this.success = false;
        this.code = code;
        this.message = message;
        this.status = status;
        this.errors = errors;
        this.meta = {
            request_id: meta.request_id || `req_${Date.now()}`,
            timestamp: new Date().toISOString(),
            ...meta
        };
    }

    toJSON() {
        const response = {
            success: this.success,
            code: this.code,
            message: this.message,
        };
        if (this.errors.length > 0) response.errors = this.errors;
        response.meta = this.meta;
        return response;
    }

    send(res) {
        return res.status(this.status)
            .set('Content-Type', 'application/problem+json')
            .json(this.toJSON());
    }
}

module.exports = ApiErrorResponse;
```

---

## 💻 Implementation

### Laravel — Exception Handler ও Custom Exceptions

```php
// app/Exceptions/ApiException.php
<?php

namespace App\Exceptions;

use Exception;

class ApiException extends Exception
{
    protected string $errorCode;
    protected array $errors;
    protected array $meta;

    public function __construct(
        string $errorCode,
        string $message,
        int $statusCode = 400,
        array $errors = [],
        array $meta = []
    ) {
        parent::__construct($message, $statusCode);
        $this->errorCode = $errorCode;
        $this->errors = $errors;
        $this->meta = $meta;
    }

    public function getErrorCode(): string { return $this->errorCode; }
    public function getErrors(): array { return $this->errors; }
    public function getStatusCode(): int { return $this->getCode(); }
    public function getMeta(): array { return $this->meta; }

    public function render($request)
    {
        return ApiErrorResponse::make(
            $this->errorCode,
            $this->getMessage(),
            $this->getStatusCode(),
            $this->errors,
            $this->meta
        );
    }
}

// নির্দিষ্ট Business Logic Exceptions
class InsufficientBalanceException extends ApiException
{
    public function __construct(float $currentBalance, float $requiredAmount)
    {
        parent::__construct(
            'E3001',
            'অপর্যাপ্ত ব্যালেন্স। আপনার অ্যাকাউন্টে পর্যাপ্ত টাকা নেই।',
            422,
            [],
            [
                'current_balance' => $currentBalance,
                'required_amount' => $requiredAmount,
                'deficit' => $requiredAmount - $currentBalance
            ]
        );
    }
}

class ResourceNotFoundException extends ApiException
{
    public function __construct(string $resource, string|int $id)
    {
        parent::__construct(
            'E4001',
            "{$resource} খুঁজে পাওয়া যায়নি (ID: {$id})",
            404
        );
    }
}

class DuplicateEntryException extends ApiException
{
    public function __construct(string $field, string $value)
    {
        parent::__construct(
            'E2005',
            "'{$field}' ফিল্ডে '{$value}' ইতিমধ্যে বিদ্যমান আছে",
            409
        );
    }
}
```

```php
// app/Exceptions/Handler.php — Global Exception Handler
<?php

namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\MethodNotAllowedHttpException;
use Throwable;

class Handler extends ExceptionHandler
{
    public function register(): void
    {
        // Validation ত্রুটি — Laravel-এর নিজস্ব validation system থেকে আসে
        $this->renderable(function (ValidationException $e, $request) {
            if ($request->expectsJson()) {
                return ApiErrorResponse::make(
                    'E2000',
                    'তথ্য যাচাইয়ে ত্রুটি পাওয়া গেছে',
                    422,
                    $this->formatValidationErrors($e)
                );
            }
        });

        // Authentication ত্রুটি
        $this->renderable(function (AuthenticationException $e, $request) {
            if ($request->expectsJson()) {
                return ApiErrorResponse::make(
                    'E1001',
                    'অনুগ্রহ করে লগইন করুন',
                    401
                );
            }
        });

        // Authorization ত্রুটি
        $this->renderable(function (AuthorizationException $e, $request) {
            if ($request->expectsJson()) {
                return ApiErrorResponse::make(
                    'E1003',
                    'এই কাজটি করার অনুমতি আপনার নেই',
                    403
                );
            }
        });

        // Model Not Found
        $this->renderable(function (ModelNotFoundException $e, $request) {
            if ($request->expectsJson()) {
                $model = class_basename($e->getModel());
                return ApiErrorResponse::make(
                    'E4001',
                    "{$model} খুঁজে পাওয়া যায়নি",
                    404
                );
            }
        });

        // Route Not Found
        $this->renderable(function (NotFoundHttpException $e, $request) {
            if ($request->expectsJson()) {
                return ApiErrorResponse::make(
                    'E4000',
                    'অনুরোধকৃত endpoint খুঁজে পাওয়া যায়নি',
                    404
                );
            }
        });

        // Method Not Allowed
        $this->renderable(function (MethodNotAllowedHttpException $e, $request) {
            if ($request->expectsJson()) {
                return ApiErrorResponse::make(
                    'E4005',
                    'এই endpoint-এ এই HTTP method সমর্থিত নয়',
                    405,
                    [],
                    ['allowed_methods' => $e->getHeaders()['Allow'] ?? '']
                );
            }
        });

        // সব অজানা exception — 500 Internal Server Error
        $this->renderable(function (Throwable $e, $request) {
            if ($request->expectsJson() && !($e instanceof ApiException)) {
                \Log::error('Unhandled Exception', [
                    'exception' => get_class($e),
                    'message'   => $e->getMessage(),
                    'file'      => $e->getFile(),
                    'line'      => $e->getLine(),
                    'trace'     => $e->getTraceAsString(),
                    'request'   => [
                        'url'     => $request->fullUrl(),
                        'method'  => $request->method(),
                        'ip'      => $request->ip(),
                    ]
                ]);

                // Production-এ কখনো internal details পাঠাবেন না
                $message = app()->environment('production')
                    ? 'সার্ভারে একটি অভ্যন্তরীণ সমস্যা হয়েছে'
                    : $e->getMessage();

                return ApiErrorResponse::make('E5000', $message, 500);
            }
        });
    }

    private function formatValidationErrors(ValidationException $e): array
    {
        $formatted = [];
        foreach ($e->errors() as $field => $messages) {
            $formatted[$field] = $messages;
        }
        return $formatted;
    }
}
```

```php
// app/Http/Requests/StoreTransactionRequest.php — FormRequest Validation
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class StoreTransactionRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'sender_phone'   => ['required', 'regex:/^\+880[0-9]{10}$/'],
            'receiver_phone' => ['required', 'regex:/^\+880[0-9]{10}$/'],
            'amount'         => ['required', 'numeric', 'min:10', 'max:50000'],
            'pin'            => ['required', 'digits:4'],
        ];
    }

    public function messages(): array
    {
        return [
            'sender_phone.required'   => 'প্রেরকের ফোন নম্বর আবশ্যক',
            'sender_phone.regex'      => 'ফোন নম্বর +880XXXXXXXXXX ফরম্যাটে দিন',
            'receiver_phone.required' => 'প্রাপকের ফোন নম্বর আবশ্যক',
            'amount.required'         => 'পরিমাণ উল্লেখ করুন',
            'amount.min'              => 'সর্বনিম্ন লেনদেনের পরিমাণ ১০ টাকা',
            'amount.max'              => 'সর্বোচ্চ লেনদেনের পরিমাণ ৫০,০০০ টাকা',
            'pin.required'            => 'পিন নম্বর দিন',
            'pin.digits'              => 'পিন অবশ্যই ৪ সংখ্যার হতে হবে',
        ];
    }

    protected function failedValidation(Validator $validator)
    {
        throw new HttpResponseException(
            ApiErrorResponse::make(
                'E2000',
                'লেনদেনের তথ্যে ত্রুটি আছে',
                422,
                $validator->errors()->toArray()
            )
        );
    }
}
```

### Express — Error Middleware ও Custom Error Classes

```js
// errors/AppError.js — Base Error Class
class AppError extends Error {
    constructor(code, message, statusCode, errors = [], meta = {}) {
        super(message);
        this.name = this.constructor.name;
        this.code = code;
        this.statusCode = statusCode;
        this.errors = errors;
        this.meta = meta;
        this.isOperational = true; // operational error (expected) vs programming error

        Error.captureStackTrace(this, this.constructor);
    }
}

class ValidationError extends AppError {
    constructor(errors, message = 'তথ্য যাচাইয়ে ত্রুটি পাওয়া গেছে') {
        super('E2000', message, 422, errors);
    }
}

class AuthenticationError extends AppError {
    constructor(message = 'অনুগ্রহ করে লগইন করুন') {
        super('E1001', message, 401);
    }
}

class ForbiddenError extends AppError {
    constructor(message = 'এই কাজটি করার অনুমতি আপনার নেই') {
        super('E1003', message, 403);
    }
}

class NotFoundError extends AppError {
    constructor(resource = 'Resource', id = '') {
        super('E4001', `${resource} খুঁজে পাওয়া যায়নি${id ? ` (ID: ${id})` : ''}`, 404);
    }
}

class ConflictError extends AppError {
    constructor(message) {
        super('E2005', message, 409);
    }
}

class UpstreamServiceError extends AppError {
    constructor(serviceName, originalError = null) {
        super(
            'E5002',
            `${serviceName} সার্ভিসে সাময়িক সমস্যা হচ্ছে`,
            502,
            [],
            { service: serviceName, upstream_error: originalError?.message }
        );
    }
}

module.exports = {
    AppError, ValidationError, AuthenticationError,
    ForbiddenError, NotFoundError, ConflictError, UpstreamServiceError
};
```

```js
// middleware/errorHandler.js — Global Error Handler
require('express-async-errors'); // async error গুলো auto-catch করবে

const { AppError } = require('../errors/AppError');
const logger = require('../utils/logger');

// express-validator errors format করা
function formatExpressValidatorErrors(errors) {
    const formatted = {};
    errors.forEach(err => {
        if (!formatted[err.path]) formatted[err.path] = [];
        formatted[err.path].push(err.msg);
    });
    return formatted;
}

// Global Error Handler Middleware (চারটি parameter = error middleware)
function errorHandler(err, req, res, next) {
    // Correlation ID — request tracking-এর জন্য
    const requestId = req.headers['x-request-id'] || `req_${Date.now()}`;

    // Operational error (আমাদের তৈরি — expected)
    if (err instanceof AppError) {
        logger.warn('Operational Error', {
            code: err.code,
            message: err.message,
            path: req.originalUrl,
            method: req.method,
            request_id: requestId,
        });

        return res.status(err.statusCode).json({
            success: false,
            code: err.code,
            message: err.message,
            ...(err.errors.length > 0 && { errors: err.errors }),
            meta: {
                request_id: requestId,
                timestamp: new Date().toISOString(),
                ...err.meta,
            },
        });
    }

    // Mongoose Validation Error
    if (err.name === 'ValidationError' && err.errors) {
        const errors = {};
        Object.keys(err.errors).forEach(key => {
            errors[key] = [err.errors[key].message];
        });

        return res.status(422).json({
            success: false,
            code: 'E2000',
            message: 'তথ্য যাচাইয়ে ত্রুটি পাওয়া গেছে',
            errors,
            meta: { request_id: requestId, timestamp: new Date().toISOString() },
        });
    }

    // Mongoose Duplicate Key Error
    if (err.code === 11000) {
        const field = Object.keys(err.keyPattern)[0];
        return res.status(409).json({
            success: false,
            code: 'E2005',
            message: `'${field}' ইতিমধ্যে বিদ্যমান আছে`,
            meta: { request_id: requestId, timestamp: new Date().toISOString() },
        });
    }

    // JSON Parse Error
    if (err.type === 'entity.parse.failed') {
        return res.status(400).json({
            success: false,
            code: 'E1000',
            message: 'অবৈধ JSON ফরম্যাট। অনুরোধের body পার্স করা যায়নি।',
            meta: { request_id: requestId, timestamp: new Date().toISOString() },
        });
    }

    // Programming error (unexpected) — log করুন, কিন্তু details ক্লায়েন্টে পাঠাবেন না
    logger.error('Unexpected Error', {
        error: err.message,
        stack: err.stack,
        path: req.originalUrl,
        method: req.method,
        body: req.body,
        request_id: requestId,
    });

    const message = process.env.NODE_ENV === 'production'
        ? 'সার্ভারে একটি অভ্যন্তরীণ সমস্যা হয়েছে'
        : err.message;

    res.status(500).json({
        success: false,
        code: 'E5000',
        message,
        meta: { request_id: requestId, timestamp: new Date().toISOString() },
    });
}

// 404 handler — কোনো route match না করলে
function notFoundHandler(req, res) {
    res.status(404).json({
        success: false,
        code: 'E4000',
        message: `${req.method} ${req.originalUrl} endpoint খুঁজে পাওয়া যায়নি`,
        meta: { timestamp: new Date().toISOString() },
    });
}

module.exports = { errorHandler, notFoundHandler, formatExpressValidatorErrors };
```

```js
// routes/users.js — express-validator ব্যবহার
const { body, validationResult } = require('express-validator');
const { ValidationError } = require('../errors/AppError');
const { formatExpressValidatorErrors } = require('../middleware/errorHandler');

const validateUser = [
    body('name').trim().notEmpty().withMessage('নাম আবশ্যক')
        .isLength({ min: 3 }).withMessage('নাম কমপক্ষে ৩ অক্ষরের হতে হবে'),
    body('email').isEmail().withMessage('সঠিক ইমেইল ঠিকানা দিন'),
    body('phone').matches(/^\+880[0-9]{10}$/).withMessage('ফোন নম্বর +880XXXXXXXXXX ফরম্যাটে দিন'),
];

router.post('/users', validateUser, async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        throw new ValidationError(formatExpressValidatorErrors(errors.array()));
    }

    const user = await User.create(req.body);
    res.status(201).json({ success: true, data: user });
});

// app.js setup
const { errorHandler, notFoundHandler } = require('./middleware/errorHandler');

app.use(express.json());
app.use('/api', routes);
app.use(notFoundHandler);    // সবার শেষে — কোনো route match না করলে
app.use(errorHandler);       // error handler সবার শেষে
```

---

## 🔥 Advanced Patterns

### ১. Error Codes System — শ্রেণীবদ্ধ এরর কোড

একটি সুসংগঠিত error code system ডিবাগিং অনেক সহজ করে। প্রতিটি error-এর একটি unique code থাকলে log search, monitoring alert, এবং documentation সব সহজ হয়।

```
Error Code Format: E[Category][Sequence]

Category:
  1xxx — Authentication & Authorization
  2xxx — Validation & Input
  3xxx — Business Logic
  4xxx — Resource (Not Found, Gone)
  5xxx — Server & Infrastructure
  6xxx — External Service
  7xxx — Rate Limiting & Throttling

উদাহরণ:
  E1001 — Token missing/invalid
  E1002 — Token expired
  E1003 — Insufficient permissions
  E2001 — Required field missing
  E2002 — Invalid format
  E2005 — Duplicate entry
  E3001 — Insufficient balance
  E3002 — Transaction limit exceeded
  E3003 — Account suspended
  E5001 — Database connection failed
  E6001 — bKash API timeout
  E6002 — SMS gateway failure
  E7001 — Rate limit exceeded
```

```php
// Laravel — Error Code Registry (config/error_codes.php)
<?php
return [
    'E1001' => [
        'title'   => 'Authentication Required',
        'message' => [
            'bn' => 'অনুগ্রহ করে লগইন করুন',
            'en' => 'Please log in to continue',
        ],
        'status'  => 401,
        'docs'    => 'https://api.example.com/docs/errors#E1001',
    ],
    'E2001' => [
        'title'   => 'Validation Failed',
        'message' => [
            'bn' => 'তথ্য যাচাইয়ে ত্রুটি পাওয়া গেছে',
            'en' => 'Validation failed',
        ],
        'status'  => 422,
        'docs'    => 'https://api.example.com/docs/errors#E2001',
    ],
    'E3001' => [
        'title'   => 'Insufficient Balance',
        'message' => [
            'bn' => 'অপর্যাপ্ত ব্যালেন্স। আপনার অ্যাকাউন্টে পর্যাপ্ত টাকা নেই।',
            'en' => 'Insufficient balance in your account.',
        ],
        'status'  => 422,
        'docs'    => 'https://api.example.com/docs/errors#E3001',
    ],
    'E6001' => [
        'title'   => 'Payment Gateway Timeout',
        'message' => [
            'bn' => 'পেমেন্ট সার্ভিসে সাময়িক সমস্যা। কিছুক্ষণ পর আবার চেষ্টা করুন।',
            'en' => 'Payment service is temporarily unavailable.',
        ],
        'status'  => 502,
        'docs'    => 'https://api.example.com/docs/errors#E6001',
    ],
];
```

### ২. Localized Error Messages — বহুভাষিক এরর মেসেজ

বাংলাদেশের API-গুলোতে বাংলা ও ইংরেজি দুটো ভাষায় error message দেওয়া উচিত। ক্লায়েন্ট `Accept-Language` header দিয়ে পছন্দের ভাষা জানাবে।

```php
// Laravel — Localized Error Middleware
<?php

namespace App\Http\Middleware;

class LocalizeApiErrors
{
    public function handle($request, \Closure $next)
    {
        $locale = $request->header('Accept-Language', 'bn');
        $supported = ['bn', 'en'];

        if (!in_array($locale, $supported)) {
            $locale = 'bn'; // default বাংলা
        }

        app()->setLocale($locale);
        return $next($request);
    }
}

// ব্যবহার — ApiException-এ
class ApiException extends Exception
{
    public function render($request)
    {
        $locale = app()->getLocale();
        $errorConfig = config("error_codes.{$this->errorCode}");
        $message = $errorConfig['message'][$locale] ?? $this->getMessage();

        return ApiErrorResponse::make(
            $this->errorCode,
            $message,
            $this->getStatusCode(),
            $this->errors
        );
    }
}
```

```js
// Express — Localized Messages
const errorMessages = {
    E1001: { bn: 'অনুগ্রহ করে লগইন করুন', en: 'Please log in' },
    E2000: { bn: 'তথ্য যাচাইয়ে ত্রুটি', en: 'Validation failed' },
    E3001: { bn: 'অপর্যাপ্ত ব্যালেন্স', en: 'Insufficient balance' },
    E6001: { bn: 'পেমেন্ট সেবা অস্থায়ীভাবে বন্ধ', en: 'Payment service temporarily down' },
};

function getLocalizedMessage(code, lang = 'bn') {
    return errorMessages[code]?.[lang] || errorMessages[code]?.['en'] || 'Unknown error';
}

// Middleware — Accept-Language থেকে locale নেওয়া
function localeMiddleware(req, res, next) {
    req.locale = ['bn', 'en'].includes(req.headers['accept-language'])
        ? req.headers['accept-language']
        : 'bn';
    next();
}
```

### ৩. Error Logging — Structured Logging with Correlation ID

প্রতিটি request-এর সাথে একটি unique correlation ID থাকলে distributed system-এ একটি request-এর সব log একসাথে খুঁজে পাওয়া যায়।

```js
// Express — Correlation ID Middleware + Structured Logging
const { v4: uuidv4 } = require('uuid');
const winston = require('winston');

const logger = winston.createLogger({
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
        new winston.transports.File({ filename: 'logs/combined.log' }),
    ],
});

// Correlation ID Middleware
function correlationId(req, res, next) {
    req.correlationId = req.headers['x-correlation-id'] || uuidv4();
    res.setHeader('X-Correlation-ID', req.correlationId);
    next();
}

// Error handler-এ structured log
function errorHandler(err, req, res, next) {
    const logContext = {
        correlation_id: req.correlationId,
        error_code: err.code || 'E5000',
        method: req.method,
        url: req.originalUrl,
        user_id: req.user?.id,
        ip: req.ip,
        user_agent: req.headers['user-agent'],
        stack: err.stack,
        request_body: sanitizeBody(req.body), // পাসওয়ার্ড, পিন লুকানো
    };

    if (err.isOperational) {
        logger.warn('Operational Error', logContext);
    } else {
        logger.error('Unexpected Error', logContext);
    }
    // ... response পাঠানো
}

function sanitizeBody(body) {
    if (!body) return {};
    const sanitized = { ...body };
    const sensitiveFields = ['password', 'pin', 'token', 'secret', 'credit_card'];
    sensitiveFields.forEach(field => {
        if (sanitized[field]) sanitized[field] = '***REDACTED***';
    });
    return sanitized;
}
```

### ৪. Error Monitoring — Sentry ও Bugsnag Integration

Production-এ error monitoring ছাড়া চলে না। Sentry বা Bugsnag — এরা real-time alert দেয়, error grouping করে, এবং trend analysis দেখায়।

```php
// Laravel — Sentry Integration (config/sentry.php সেটআপের পর)
// app/Exceptions/Handler.php-এ
public function register(): void
{
    $this->reportable(function (Throwable $e) {
        if ($this->shouldReport($e) && app()->bound('sentry')) {
            \Sentry\configureScope(function (\Sentry\State\Scope $scope) {
                $scope->setTag('error_code', $e instanceof ApiException ? $e->getErrorCode() : 'UNKNOWN');
                $scope->setUser([
                    'id'    => auth()->id(),
                    'email' => auth()->user()?->email,
                ]);
                $scope->setContext('request', [
                    'url'    => request()->fullUrl(),
                    'method' => request()->method(),
                    'ip'     => request()->ip(),
                ]);
            });

            app('sentry')->captureException($e);
        }
    });
}
```

```js
// Express — Sentry Integration
const Sentry = require('@sentry/node');

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV,
    tracesSampleRate: 0.2, // ২০% request trace করা
    beforeSend(event, hint) {
        const error = hint.originalException;
        // Operational error (4xx) Sentry-তে পাঠানোর দরকার নেই
        if (error?.isOperational && error?.statusCode < 500) {
            return null; // Sentry-তে পাঠাবে না
        }
        return event;
    },
});

// Sentry middleware (error handler-এর আগে)
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.errorHandler({
    shouldHandleError(error) {
        // শুধু 5xx error Sentry-তে পাঠাও
        return !error.statusCode || error.statusCode >= 500;
    }
}));
```

### ৫. Graceful Degradation — Fallback Responses

বাহ্যিক সার্ভিস (bKash, SMS gateway) ডাউন থাকলে পুরো API ভেঙে পড়া উচিত নয়। Fallback strategy রাখতে হবে।

```php
// Laravel — Circuit Breaker Pattern with Fallback
class PaymentService
{
    public function processPayment(array $data): PaymentResult
    {
        try {
            // প্রথমে bKash try করি
            return $this->bkashGateway->charge($data);
        } catch (UpstreamServiceException $e) {
            Log::warning('bKash unavailable, trying Nagad fallback', [
                'error' => $e->getMessage()
            ]);

            try {
                // bKash না হলে Nagad try করি
                return $this->nagadGateway->charge($data);
            } catch (UpstreamServiceException $e2) {
                Log::error('All payment gateways failed', [
                    'bkash_error' => $e->getMessage(),
                    'nagad_error' => $e2->getMessage(),
                ]);

                throw new ApiException(
                    'E6000',
                    'পেমেন্ট সার্ভিস সাময়িকভাবে অনুপলব্ধ। অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন।',
                    503,
                    [],
                    ['retry_after' => 300] // ৫ মিনিট পর retry
                );
            }
        }
    }
}
```

```js
// Express — Degraded response with cached data
async function getProductDetails(req, res, next) {
    try {
        const product = await ProductService.fetch(req.params.id);
        const reviews = await fetchWithFallback(
            () => ReviewService.fetch(req.params.id),  // primary
            () => cache.get(`reviews:${req.params.id}`), // fallback: cache
            { reviews: [], _degraded: true }              // last resort: empty
        );

        res.json({
            data: { ...product, reviews: reviews.data || reviews },
            ...(reviews._degraded && {
                warnings: [{
                    code: 'W6001',
                    message: 'রিভিউ সার্ভিস সাময়িকভাবে অনুপলব্ধ, ক্যাশ থেকে দেখানো হচ্ছে'
                }]
            })
        });
    } catch (err) {
        next(err);
    }
}

async function fetchWithFallback(primary, fallback, defaultValue) {
    try {
        return await primary();
    } catch {
        try {
            return await fallback();
        } catch {
            return defaultValue;
        }
    }
}
```

### ৬. Retry-friendly Errors — Retry-After Header ও Idempotency

কিছু error retry করলে সমাধান হতে পারে (503, 429)। ক্লায়েন্টকে জানাতে হবে কখন retry করবে এবং retry নিরাপদ কিনা।

```php
// Laravel — Retry-After Header
class RateLimitMiddleware
{
    public function handle($request, \Closure $next)
    {
        $key = 'rate_limit:' . $request->ip();
        $maxAttempts = 100;
        $decayMinutes = 15;

        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            $retryAfter = RateLimiter::availableIn($key);

            return response()->json([
                'code'    => 'E7001',
                'message' => "অনুরোধের সীমা অতিক্রম করেছেন। {$retryAfter} সেকেন্ড পর আবার চেষ্টা করুন।",
            ], 429)->header('Retry-After', $retryAfter);
        }

        RateLimiter::hit($key, $decayMinutes * 60);
        return $next($request);
    }
}

// Idempotency Key — duplicate request ঠেকানো
class IdempotencyMiddleware
{
    public function handle($request, \Closure $next)
    {
        if (!in_array($request->method(), ['POST', 'PATCH'])) {
            return $next($request);
        }

        $idempotencyKey = $request->header('Idempotency-Key');
        if (!$idempotencyKey) {
            return $next($request);
        }

        $cacheKey = "idempotency:{$idempotencyKey}";
        $cached = Cache::get($cacheKey);

        if ($cached) {
            return response()->json(
                json_decode($cached['body'], true),
                $cached['status']
            )->header('X-Idempotent-Replayed', 'true');
        }

        $response = $next($request);

        Cache::put($cacheKey, [
            'status' => $response->getStatusCode(),
            'body'   => $response->getContent(),
        ], now()->addHours(24));

        return $response;
    }
}
```

### ৭. GraphQL Error Handling

GraphQL-এ HTTP status code সবসময় `200` থাকে। Error গুলো response body-র `errors` array-তে আসে। এটি REST থেকে মৌলিকভাবে আলাদা।

```js
// Express + Apollo Server — GraphQL Error Handling
const { ApolloServer } = require('@apollo/server');
const { GraphQLError } = require('graphql');

// Custom GraphQL Error throw করা
function getUser(id) {
    const user = db.users.find(id);
    if (!user) {
        throw new GraphQLError('ইউজার খুঁজে পাওয়া যায়নি', {
            extensions: {
                code: 'USER_NOT_FOUND',
                error_code: 'E4001',
                http: { status: 404 },
                argumentName: 'id',
            },
        });
    }
    return user;
}

// GraphQL Response — partial errors (কিছু data সফল, কিছু ব্যর্থ)
// POST /graphql → 200 OK
/*
{
    "data": {
        "user": {
            "name": "রহিম",
            "email": "rahim@example.com",
            "recentOrders": null    // ← এই অংশটুকু ব্যর্থ
        }
    },
    "errors": [
        {
            "message": "অর্ডার সার্ভিস সাময়িকভাবে অনুপলব্ধ",
            "path": ["user", "recentOrders"],
            "extensions": {
                "code": "UPSTREAM_SERVICE_ERROR",
                "error_code": "E6001",
                "service": "order-service"
            }
        }
    ]
}
*/

// Apollo Server error formatting
const server = new ApolloServer({
    typeDefs,
    resolvers,
    formatError: (formattedError, error) => {
        // Production-এ internal error-এর বিস্তারিত লুকানো
        if (process.env.NODE_ENV === 'production' && !formattedError.extensions?.code) {
            return {
                message: 'সার্ভারে একটি সমস্যা হয়েছে',
                extensions: { code: 'INTERNAL_SERVER_ERROR' },
            };
        }

        // Stack trace সরানো
        delete formattedError.extensions?.stacktrace;
        return formattedError;
    },
});
```

### ৮. Validation Error Patterns — জটিল যাচাই

বাস্তব API-তে nested object এবং array validation অনেক common। এগুলোর error message clear ও navigable হতে হবে।

```php
// Laravel — Nested Object + Array Validation
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            // মূল অর্ডার তথ্য
            'customer_phone'          => 'required|regex:/^\+880[0-9]{10}$/',
            'delivery_address'        => 'required|array',
            'delivery_address.street' => 'required|string|min:5',
            'delivery_address.city'   => 'required|string',
            'delivery_address.zip'    => 'required|digits:4',

            // আইটেম array — কমপক্ষে ১টি আইটেম
            'items'                 => 'required|array|min:1|max:50',
            'items.*.product_id'    => 'required|exists:products,id',
            'items.*.quantity'      => 'required|integer|min:1|max:100',
            'items.*.variants'      => 'sometimes|array',
            'items.*.variants.size' => 'sometimes|in:S,M,L,XL,XXL',
        ];
    }

    public function messages(): array
    {
        return [
            'items.min'                    => 'কমপক্ষে ১টি আইটেম যোগ করুন',
            'items.max'                    => 'একবারে সর্বোচ্চ ৫০টি আইটেম অর্ডার করা যায়',
            'items.*.product_id.exists'    => ':position নম্বর আইটেমের product পাওয়া যায়নি',
            'items.*.quantity.min'         => 'আইটেমের পরিমাণ কমপক্ষে ১ হতে হবে',
            'delivery_address.zip.digits'  => 'পোস্ট কোড ৪ সংখ্যার হতে হবে',
        ];
    }
}

/* Error Response:
{
    "code": "E2000",
    "message": "অর্ডারের তথ্যে ত্রুটি আছে",
    "errors": {
        "delivery_address.zip": ["পোস্ট কোড ৪ সংখ্যার হতে হবে"],
        "items.0.quantity": ["আইটেমের পরিমাণ কমপক্ষে ১ হতে হবে"],
        "items.2.product_id": ["৩ নম্বর আইটেমের product পাওয়া যায়নি"]
    }
}
*/
```

```js
// Express — express-validator দিয়ে complex validation
const { body, checkSchema } = require('express-validator');

const orderValidation = checkSchema({
    customer_phone: {
        matches: { options: /^\+880[0-9]{10}$/, errorMessage: 'ফোন নম্বর +880XXXXXXXXXX ফরম্যাটে দিন' },
    },
    'delivery_address.street': {
        notEmpty: { errorMessage: 'রাস্তার ঠিকানা আবশ্যক' },
        isLength: { options: { min: 5 }, errorMessage: 'ঠিকানা কমপক্ষে ৫ অক্ষরের হতে হবে' },
    },
    'delivery_address.city': {
        notEmpty: { errorMessage: 'শহরের নাম আবশ্যক' },
    },
    items: {
        isArray: { options: { min: 1, max: 50 }, errorMessage: 'কমপক্ষে ১টি, সর্বোচ্চ ৫০টি আইটেম' },
    },
    'items.*.product_id': {
        notEmpty: { errorMessage: 'প্রোডাক্ট ID আবশ্যক' },
    },
    'items.*.quantity': {
        isInt: { options: { min: 1, max: 100 }, errorMessage: 'পরিমাণ ১-১০০ এর মধ্যে হতে হবে' },
    },
});
```

### ৯. Security-conscious Errors — প্রোডাকশনে তথ্য সুরক্ষা

Production API থেকে কোনো internal implementation details বের হওয়া উচিত নয়। এটি security vulnerability তৈরি করে।

```
❌ কখনো ক্লায়েন্টে পাঠাবেন না:
──────────────────────────────────
• Stack traces
• Database query / table names
• File paths (e.g., /var/www/app/...)
• Internal IP addresses
• Library/framework version numbers
• Raw SQL errors (SQLSTATE[42S02]...)
• Environment variables

✅ এর পরিবর্তে:
──────────────────
• Generic user-friendly message
• Error code (E5000) যা দিয়ে log search করা যাবে
• Correlation ID যা support-কে দিয়ে investigate করা যাবে
```

```php
// Laravel — Production Error Sanitization
class Handler extends ExceptionHandler
{
    public function render($request, Throwable $e)
    {
        if ($request->expectsJson()) {
            // QueryException — DB error কখনো ক্লায়েন্টে নয়
            if ($e instanceof \Illuminate\Database\QueryException) {
                Log::error('Database Error', [
                    'sql'       => $e->getSql(),    // log-এ রাখুন
                    'bindings'  => $e->getBindings(),
                    'message'   => $e->getMessage(),
                ]);

                return ApiErrorResponse::make(
                    'E5001',
                    'ডেটা প্রসেসিংয়ে সমস্যা হয়েছে। অনুগ্রহ করে পরে আবার চেষ্টা করুন।',
                    500
                    // ← SQL error, table name কিছুই পাঠাচ্ছি না
                );
            }

            // development-এ বিস্তারিত দেখান, production-এ লুকান
            if (!app()->environment('production')) {
                return ApiErrorResponse::make(
                    'E5000',
                    $e->getMessage(),
                    500,
                    [],
                    [
                        'exception' => get_class($e),
                        'file'      => $e->getFile(),
                        'line'      => $e->getLine(),
                        'trace'     => collect($e->getTrace())->take(5)->toArray(),
                    ]
                );
            }
        }

        return parent::render($request, $e);
    }
}
```

---

## ✅ Best Practices

### ১. সবসময় Consistent Format ব্যবহার করুন
প্রতিটি error response-এর structure একই হতে হবে — `code`, `message`, `errors`, `meta`। ক্লায়েন্ট যেন একটি generic handler লিখতে পারে।

### ২. সঠিক HTTP Status Code নির্বাচন করুন
`200` দিয়ে error body পাঠানো (যেমন `{"success": false}`) একটি anti-pattern। HTTP spec অনুযায়ী সঠিক status code ব্যবহার করুন।

### ৩. Actionable Error Message দিন
ইউজারকে বলুন **কী ভুল হয়েছে এবং কী করতে হবে**:
- ❌ `"Invalid input"`
- ✅ `"ফোন নম্বর +880XXXXXXXXXX ফরম্যাটে দিন"`

### ৪. Validation Error-এ Field-level তথ্য দিন
একটি generic "validation failed" যথেষ্ট নয়। প্রতিটি ভুল field আলাদা করে চিহ্নিত করুন।

### ৫. Production-এ Internal Details লুকান
Stack trace, SQL query, file path — সব production-এ লুকানো থাকবে। শুধু error code ও correlation ID দিন যা দিয়ে support team log থেকে বিস্তারিত দেখতে পারবে।

### ৬. Correlation ID ব্যবহার করুন
প্রতিটি request-এ unique ID থাকলে distributed system-এ debugging অনেক সহজ হয়। Response header-এ `X-Correlation-ID` পাঠান।

### ৭. Error Monitoring সেটআপ করুন
Sentry, Bugsnag, বা Datadog — যেকোনো একটি ব্যবহার করুন। ৫xx error-এর জন্য real-time alert সেটআপ করুন।

### ৮. Error Documentation তৈরি করুন
প্রতিটি error code-এর জন্য documentation থাকুক — কেন হয়, কীভাবে ঠিক করবে, কার দায়িত্ব (ক্লায়েন্ট নাকি সার্ভার)।

### ৯. Rate Limit Error-এ Retry-After দিন
429 response-এ `Retry-After` header দিন যাতে ক্লায়েন্ট জানে কতক্ষণ অপেক্ষা করতে হবে।

### ১০. Idempotency Key সমর্থন করুন
POST/PATCH request-এ `Idempotency-Key` header support করুন যাতে network failure-এ retry নিরাপদ হয়।

---

## ⚠️ Anti-patterns — যা করবেন না

### ১. সব কিছুতে 200 OK রিটার্ন করা
```json
// ❌ ভুল — HTTP 200 দিয়ে error body
{
    "success": false,
    "error": "User not found"
}

// ✅ সঠিক — HTTP 404 সহ structured error
// HTTP/1.1 404 Not Found
{
    "code": "E4001",
    "message": "ইউজার খুঁজে পাওয়া যায়নি"
}
```

### ২. সব কিছুতে 500 Internal Server Error
```
❌ ভুল: Validation fail → 500
❌ ভুল: Auth fail → 500
❌ ভুল: Not found → 500

✅ সঠিক: প্রতিটি error-এর জন্য সঠিক status code
```

### ৩. Inconsistent Error Format
```json
// ❌ এক endpoint থেকে:
{ "error": "Something went wrong" }

// ❌ আরেক endpoint থেকে:
{ "message": "Bad request", "statusCode": 400 }

// ❌ আরেক endpoint থেকে:
{ "errors": [{ "msg": "Invalid email" }] }

// ✅ সবখানে একই format:
{ "code": "EXXX", "message": "...", "errors": [...], "meta": {...} }
```

### ৪. Stack Trace ক্লায়েন্টে পাঠানো
```json
// ❌ মারাত্মক — attacker জানবে আপনার framework, file structure, DB
{
    "error": "SQLSTATE[42S02]: Base table or not found",
    "trace": "at /var/www/app/Models/User.php:45\nat vendor/laravel/..."
}

// ✅ নিরাপদ
{
    "code": "E5001",
    "message": "সার্ভারে একটি সমস্যা হয়েছে",
    "meta": { "request_id": "req_abc123", "support": "এই ID দিয়ে সাপোর্টে যোগাযোগ করুন" }
}
```

### ৫. Error Message-এ sensitive তথ্য
```
❌ "User admin@company.com not found"  → email leak
❌ "Password must not be 'abc123'"     → password hint
❌ "Table 'users' has no column 'x'"   → DB schema leak

✅ "ইউজার খুঁজে পাওয়া যায়নি"
✅ "পাসওয়ার্ড নীতিমালা পূরণ করেনি"
✅ "সার্ভারে সমস্যা হয়েছে"
```

### ৬. একই Error Code সবকিছুতে ব্যবহার
```
❌ সব error-এ E1000 ব্যবহার → ডিবাগিং অসম্ভব
✅ শ্রেণীবদ্ধ code: E1001, E2001, E3001 → log search সহজ
```

### ৭. Empty Error Body
```
❌ HTTP 422 (কোনো body নেই) → ক্লায়েন্ট জানে না কী ভুল
✅ HTTP 422 + { "errors": { "email": ["ইমেইল আবশ্যক"] } }
```

---

## 🇧🇩 বাংলাদেশ Context — bKash Error Pattern

বাংলাদেশে MFS (Mobile Financial Services) API-তে error handling বিশেষভাবে গুরুত্বপূর্ণ কারণ টাকা-পয়সা জড়িত। bKash API error pattern অনুসরণ করা যেতে পারে:

```js
// bKash-স্টাইল Error Code Pattern
const bkashStyleErrors = {
    // Transaction Errors
    2001: { bn: 'অপর্যাপ্ত ব্যালেন্স', en: 'Insufficient Balance' },
    2002: { bn: 'দৈনিক লেনদেনের সীমা অতিক্রম', en: 'Daily Transaction Limit Exceeded' },
    2003: { bn: 'ভুল পিন নম্বর', en: 'Invalid PIN' },
    2004: { bn: 'প্রাপকের অ্যাকাউন্ট নেই', en: 'Receiver Account Not Found' },
    2006: { bn: 'অ্যাকাউন্ট স্থগিত', en: 'Account Suspended' },

    // System Errors
    5001: { bn: 'সিস্টেমে সাময়িক সমস্যা', en: 'Temporary System Error' },
    5002: { bn: 'সার্ভিস অনুপলব্ধ', en: 'Service Unavailable' },
};

// বাংলা error message ইউজারকে দেখানো (mobile app-এ)
function getUserFriendlyError(errorCode, locale = 'bn') {
    const error = bkashStyleErrors[errorCode];
    if (!error) return locale === 'bn'
        ? 'একটি অপ্রত্যাশিত সমস্যা হয়েছে'
        : 'An unexpected error occurred';
    return error[locale];
}

// উদাহরণ: bKash-like transaction error response
/*
HTTP/1.1 422 Unprocessable Entity
{
    "statusCode": "2001",
    "statusMessage": "Insufficient Balance",
    "statusMessage_bn": "অপর্যাপ্ত ব্যালেন্স। আপনার অ্যাকাউন্টে পর্যাপ্ত টাকা নেই।",
    "trxID": null,
    "amount": "500.00",
    "currency": "BDT",
    "meta": {
        "current_balance": "350.00",
        "required_amount": "500.00",
        "charge": "5.00"
    }
}
*/
```

---

## 📋 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────────────┐
│                   API Error Handling সারসংক্ষেপ                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ১. Consistent Format     → সব error একই structure-এ            │
│  ২. সঠিক Status Code      → 4xx ক্লায়েন্ট, 5xx সার্ভার          │
│  ৩. Actionable Messages   → কী ভুল + কী করবে                    │
│  ৪. Error Codes (E1001)   → শ্রেণীবদ্ধ কোড, সহজ debugging       │
│  ৫. Field-level Errors    → ঠিক কোন field-এ কী সমস্যা           │
│  ৬. Security              → Production-এ details লুকান           │
│  ৭. Correlation ID        → Distributed tracing                  │
│  ৮. Monitoring            → Sentry/Bugsnag real-time alert       │
│  ৯. Retry-After           → ক্লায়েন্টকে retry-র সময় জানান       │
│  ১০. Localization         → বাংলা + ইংরেজি error messages        │
│  ১১. Graceful Degradation → Fallback response strategy           │
│  ১২. Idempotency          → নিরাপদ retry mechanism               │
│                                                                  │
│  মনে রাখুন: একটি ভালো error response হলো সেই response            │
│  যেটা পড়ে ডেভেলপার ঠিক বুঝতে পারে কী ভুল হয়েছে                 │
│  এবং কীভাবে ঠিক করবে — কোনো guesswork নেই।                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

> **"Error handling is not an afterthought — it's a first-class feature of your API."**

---

*তৈরি: Pattern Study — API Design Series*
