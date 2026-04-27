# ⛓️ Chain of Responsibility প্যাটার্ন

## 📌 সংজ্ঞা ও মূল ধারণা

**Gang of Four (GoF) সংজ্ঞা:**
> *"Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it."*

**বাংলায়:** অনুরোধ প্রেরক (sender) এবং গ্রহীতাকে (receiver) সরাসরি যুক্ত না করে, একাধিক অবজেক্টকে অনুরোধ হ্যান্ডেল করার সুযোগ দেওয়া। গ্রহীতা অবজেক্টগুলোকে একটি চেইনে সাজানো হয় এবং অনুরোধটি চেইন বরাবর পাস করা হয় যতক্ষণ না কোনো অবজেক্ট সেটি হ্যান্ডেল করে।

### মূল নীতিমালা

Chain of Responsibility হলো একটি **Behavioral Design Pattern** যেখানে:

1. **Decoupling:** প্রেরক জানে না কে অনুরোধ হ্যান্ডেল করবে
2. **Dynamic Chain:** রানটাইমে চেইন পরিবর্তন/পুনর্গঠন করা যায়
3. **Single Responsibility:** প্রতিটি handler শুধু তার নিজের দায়িত্ব পালন করে
4. **Open/Closed Principle:** নতুন handler যোগ করতে বিদ্যমান কোড পরিবর্তন লাগে না

### প্যাটার্নের তিনটি রূপ

| রূপ | বর্ণনা | উদাহরণ |
|-----|---------|--------|
| **Pure CoR** | শুধুমাত্র একটি handler অনুরোধ প্রক্রিয়া করে | Exception handling |
| **Impure CoR (Pipeline)** | প্রতিটি handler কিছু প্রক্রিয়া করে ও পরবর্তীতে পাস করে | HTTP Middleware |
| **Bidirectional CoR** | চেইনের মধ্য দিয়ে যাওয়ার সময় ও ফেরার সময় দুটোতেই প্রক্রিয়া হয় | Laravel Middleware (before/after) |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### উদাহরণ ১: কাস্টমার সাপোর্ট এস্কেলেশন

কল্পনা করুন আপনি একটি টেলিকম কোম্পানিতে (যেমন গ্রামীণফোন) কল করেছেন:

```
আপনার সমস্যা: "আমার বিল ভুল এসেছে"

L1 সাপোর্ট (IVR/Bot) → সাধারণ FAQ দিয়ে সমাধান চেষ্টা
    ❌ সমাধান হয়নি
    ⬇️ এস্কেলেট

L2 সাপোর্ট (জুনিয়র এজেন্ট) → বিল চেক করে সাধারণ সমাধান দেওয়ার চেষ্টা
    ❌ জটিল সমস্যা, সমাধান হয়নি
    ⬇️ এস্কেলেট

L3 সাপোর্ট (সিনিয়র এজেন্ট) → বিস্তারিত তদন্ত, সিস্টেম অ্যাডজাস্টমেন্ট
    ❌ পলিসি-লেভেল সিদ্ধান্ত দরকার
    ⬇️ এস্কেলেট

ম্যানেজার → চূড়ান্ত সিদ্ধান্ত, বিল মওকুফ/সংশোধন
    ✅ সমাধান!
```

### উদাহরণ ২: বাংলাদেশে সরকারি অফিসের ফাইল চলাচল

```
নাগরিকের আবেদন (জমির মিউটেশন):

ইউনিয়ন পরিষদ → প্রাথমিক যাচাই, ফরওয়ার্ড
    ⬇️
উপজেলা ভূমি অফিস → জমির রেকর্ড যাচাই
    ⬇️
জেলা প্রশাসকের কার্যালয় (DC Office) → অনুমোদন
    ⬇️
ভূমি মন্ত্রণালয় → চূড়ান্ত নীতিগত সিদ্ধান্ত (প্রয়োজনে)
```

### উদাহরণ ৩: ব্যাংক লোন অনুমোদন (বাংলাদেশ প্রসঙ্গ)

```
লোনের আবেদন: ৫০ লক্ষ টাকা

ব্রাঞ্চ অফিসার → ৫ লক্ষ পর্যন্ত অনুমোদন করতে পারেন
    ❌ এই সীমার বাইরে
    ⬇️

ব্রাঞ্চ ম্যানেজার → ২০ লক্ষ পর্যন্ত অনুমোদন করতে পারেন
    ❌ এই সীমার বাইরে
    ⬇️

রিজিওনাল ম্যানেজার → ১ কোটি পর্যন্ত অনুমোদন করতে পারেন
    ✅ ৫০ লক্ষ অনুমোদিত!
```

---

## 📊 UML ডায়াগ্রাম

### ক্লাস ডায়াগ্রাম (ASCII)

```
┌─────────────────────────┐
│        Client            │
│─────────────────────────│
│ - handler: Handler       │
│─────────────────────────│
│ + sendRequest(req)       │
└──────────┬──────────────┘
           │ uses
           ▼
┌─────────────────────────────────┐
│    <<abstract>> Handler          │
│─────────────────────────────────│
│ # nextHandler: ?Handler          │
│─────────────────────────────────│
│ + setNext(handler): Handler      │
│ + handle(request): ?Response     │
└──────────┬──────────────────────┘
           │
     ┌─────┴──────────────┬────────────────────┐
     ▼                    ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ ConcreteH_A  │  │ ConcreteH_B  │  │ ConcreteH_C  │
│──────────────│  │──────────────│  │──────────────│
│ +handle(req) │  │ +handle(req) │  │ +handle(req) │
└──────────────┘  └──────────────┘  └──────────────┘
```

### সিকোয়েন্স ডায়াগ্রাম

```
Client        Handler_A      Handler_B      Handler_C
  │               │              │              │
  │──handle(req)─▶│              │              │
  │               │──canHandle?  │              │
  │               │  NO          │              │
  │               │──handle(req)─▶│              │
  │               │              │──canHandle?  │
  │               │              │  NO          │
  │               │              │──handle(req)─▶│
  │               │              │              │──canHandle?
  │               │              │              │  YES
  │◀──────────────response───────────────────────│
```

### Pipeline (Middleware) ডায়াগ্রাম

```
Request ──▶ [Auth] ──▶ [Authz] ──▶ [Validate] ──▶ [RateLimit] ──▶ [Handler]
                                                                       │
Response ◀── [Auth] ◀── [Authz] ◀── [Validate] ◀── [RateLimit] ◀──────┘
```

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Basic Chain — ভিত্তি কাঠামো

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Abstract Handler — চেইনের মূল কাঠামো
abstract class Handler
{
    private ?Handler $nextHandler = null;

    public function setNext(Handler $handler): Handler
    {
        $this->nextHandler = $handler;
        // fluent interface — চেইনিং এর জন্য handler রিটার্ন
        return $handler;
    }

    public function handle(Request $request): ?Response
    {
        // যদি পরবর্তী handler থাকে, তাকে দায়িত্ব দাও
        if ($this->nextHandler !== null) {
            return $this->nextHandler->handle($request);
        }

        // চেইনের শেষ — কেউ হ্যান্ডেল করতে পারেনি
        return null;
    }
}

// Value Objects
readonly class Request
{
    public function __construct(
        public string $type,
        public float $amount,
        public array $metadata = [],
    ) {}
}

readonly class Response
{
    public function __construct(
        public bool $approved,
        public string $handledBy,
        public string $message,
    ) {}
}

// Concrete Handlers — ব্যাংক লোন অনুমোদন চেইন
class BranchOfficerHandler extends Handler
{
    private const float MAX_AMOUNT = 500_000; // ৫ লক্ষ

    public function handle(Request $request): ?Response
    {
        if ($request->amount <= self::MAX_AMOUNT) {
            return new Response(
                approved: true,
                handledBy: 'ব্রাঞ্চ অফিসার',
                message: "৳{$request->amount} লোন অনুমোদিত (ব্রাঞ্চ লেভেল)"
            );
        }

        // নিজে হ্যান্ডেল করতে পারছে না — পরবর্তী handler এ পাঠাও
        return parent::handle($request);
    }
}

class BranchManagerHandler extends Handler
{
    private const float MAX_AMOUNT = 2_000_000; // ২০ লক্ষ

    public function handle(Request $request): ?Response
    {
        if ($request->amount <= self::MAX_AMOUNT) {
            return new Response(
                approved: true,
                handledBy: 'ব্রাঞ্চ ম্যানেজার',
                message: "৳{$request->amount} লোন অনুমোদিত (ব্রাঞ্চ ম্যানেজার লেভেল)"
            );
        }

        return parent::handle($request);
    }
}

class RegionalManagerHandler extends Handler
{
    private const float MAX_AMOUNT = 10_000_000; // ১ কোটি

    public function handle(Request $request): ?Response
    {
        if ($request->amount <= self::MAX_AMOUNT) {
            return new Response(
                approved: true,
                handledBy: 'রিজিওনাল ম্যানেজার',
                message: "৳{$request->amount} লোন অনুমোদিত (রিজিওনাল লেভেল)"
            );
        }

        return parent::handle($request);
    }
}

class HeadOfficeHandler extends Handler
{
    public function handle(Request $request): ?Response
    {
        // চেইনের শেষ handler — সব ক্ষেত্রেই সিদ্ধান্ত নেয়
        if ($request->amount <= 100_000_000) { // ১০ কোটি
            return new Response(
                approved: true,
                handledBy: 'প্রধান কার্যালয়',
                message: "৳{$request->amount} লোন অনুমোদিত (হেড অফিস লেভেল)"
            );
        }

        return new Response(
            approved: false,
            handledBy: 'প্রধান কার্যালয়',
            message: "৳{$request->amount} লোন প্রত্যাখ্যাত — বোর্ড মিটিং প্রয়োজন"
        );
    }
}

// ব্যবহার
$branchOfficer = new BranchOfficerHandler();
$branchManager = new BranchManagerHandler();
$regionalManager = new RegionalManagerHandler();
$headOffice = new HeadOfficeHandler();

// চেইন গঠন
$branchOfficer
    ->setNext($branchManager)
    ->setNext($regionalManager)
    ->setNext($headOffice);

// বিভিন্ন অনুরোধ পরীক্ষা
$requests = [
    new Request(type: 'personal', amount: 300_000),    // ৩ লক্ষ
    new Request(type: 'business', amount: 1_500_000),   // ১৫ লক্ষ
    new Request(type: 'corporate', amount: 50_000_000), // ৫ কোটি
];

foreach ($requests as $req) {
    $response = $branchOfficer->handle($req);
    echo "{$response->handledBy}: {$response->message}\n";
}
```

#### JavaScript (ES2022+)

```javascript
// Abstract Handler — ES2022 private fields ব্যবহার
class Handler {
    #nextHandler = null;

    setNext(handler) {
        this.#nextHandler = handler;
        return handler; // fluent chaining
    }

    handle(request) {
        if (this.#nextHandler) {
            return this.#nextHandler.handle(request);
        }
        return null;
    }
}

// Concrete Handlers — ব্যাংক লোন অনুমোদন
class BranchOfficerHandler extends Handler {
    static #MAX_AMOUNT = 500_000;

    handle(request) {
        if (request.amount <= BranchOfficerHandler.#MAX_AMOUNT) {
            return {
                approved: true,
                handledBy: 'ব্রাঞ্চ অফিসার',
                message: `৳${request.amount.toLocaleString('bn-BD')} লোন অনুমোদিত (ব্রাঞ্চ লেভেল)`,
            };
        }
        return super.handle(request);
    }
}

class BranchManagerHandler extends Handler {
    static #MAX_AMOUNT = 2_000_000;

    handle(request) {
        if (request.amount <= BranchManagerHandler.#MAX_AMOUNT) {
            return {
                approved: true,
                handledBy: 'ব্রাঞ্চ ম্যানেজার',
                message: `৳${request.amount.toLocaleString('bn-BD')} লোন অনুমোদিত`,
            };
        }
        return super.handle(request);
    }
}

class RegionalManagerHandler extends Handler {
    static #MAX_AMOUNT = 10_000_000;

    handle(request) {
        if (request.amount <= RegionalManagerHandler.#MAX_AMOUNT) {
            return {
                approved: true,
                handledBy: 'রিজিওনাল ম্যানেজার',
                message: `৳${request.amount.toLocaleString('bn-BD')} লোন অনুমোদিত`,
            };
        }
        return super.handle(request);
    }
}

// ব্যবহার
const branchOfficer = new BranchOfficerHandler();
const branchManager = new BranchManagerHandler();
const regionalManager = new RegionalManagerHandler();

branchOfficer.setNext(branchManager).setNext(regionalManager);

const requests = [
    { type: 'personal', amount: 300_000 },
    { type: 'business', amount: 1_500_000 },
    { type: 'corporate', amount: 50_000_000 },
];

for (const req of requests) {
    const response = branchOfficer.handle(req);
    console.log(response?.message ?? 'কেউ হ্যান্ডেল করতে পারেনি');
}
```

---

### ২. HTTP Middleware Pipeline

এটি Chain of Responsibility-এর সবচেয়ে জনপ্রিয় বাস্তব প্রয়োগ। প্রতিটি middleware অনুরোধ পরীক্ষা করে, প্রক্রিয়া করে এবং পরবর্তী middleware-এ পাস করে — অথবা চেইন ভেঙে দেয়।

#### PHP 8.3 — সম্পূর্ণ Middleware Pipeline

```php
<?php

declare(strict_types=1);

// Middleware Interface
interface Middleware
{
    public function handle(HttpRequest $request, Closure $next): HttpResponse;
}

// HTTP Value Objects
readonly class HttpRequest
{
    public function __construct(
        public string $method,
        public string $path,
        public array $headers = [],
        public array $body = [],
        public array $attributes = [], // middleware দ্বারা যোগকৃত ডেটা
    ) {}

    public function withAttribute(string $key, mixed $value): self
    {
        return new self(
            method: $this->method,
            path: $this->path,
            headers: $this->headers,
            body: $this->body,
            attributes: [...$this->attributes, $key => $value],
        );
    }
}

readonly class HttpResponse
{
    public function __construct(
        public int $statusCode,
        public string $body,
        public array $headers = [],
    ) {}
}

// ১. Authentication Middleware — টোকেন যাচাই
class AuthenticationMiddleware implements Middleware
{
    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        $token = $request->headers['Authorization'] ?? null;

        if ($token === null) {
            return new HttpResponse(
                statusCode: 401,
                body: json_encode(['error' => 'টোকেন প্রদান করা হয়নি']),
            );
        }

        // টোকেন ডিকোড করে user তথ্য সংযুক্ত করা
        $user = $this->decodeToken($token);

        if ($user === null) {
            return new HttpResponse(
                statusCode: 401,
                body: json_encode(['error' => 'অবৈধ টোকেন']),
            );
        }

        // request-এ user তথ্য যোগ করে পরবর্তী middleware-এ পাঠাও
        $request = $request->withAttribute('user', $user);
        return $next($request);
    }

    private function decodeToken(string $token): ?array
    {
        // সরলীকৃত — প্রোডাকশনে JWT verify করুন
        return str_starts_with($token, 'Bearer ')
            ? ['id' => 1, 'role' => 'admin', 'name' => 'রহিম']
            : null;
    }
}

// ২. Authorization Middleware — অনুমতি যাচাই
class AuthorizationMiddleware implements Middleware
{
    public function __construct(
        private readonly array $allowedRoles,
    ) {}

    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        $user = $request->attributes['user'] ?? null;

        if ($user === null || !in_array($user['role'], $this->allowedRoles, true)) {
            return new HttpResponse(
                statusCode: 403,
                body: json_encode(['error' => 'এই রিসোর্সে প্রবেশাধিকার নেই']),
            );
        }

        return $next($request);
    }
}

// ৩. Validation Middleware — ইনপুট যাচাই
class ValidationMiddleware implements Middleware
{
    public function __construct(
        private readonly array $rules,
    ) {}

    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        $errors = [];

        foreach ($this->rules as $field => $rule) {
            if ($rule === 'required' && empty($request->body[$field])) {
                $errors[] = "'{$field}' ফিল্ড আবশ্যক";
            }
        }

        if (!empty($errors)) {
            return new HttpResponse(
                statusCode: 422,
                body: json_encode(['errors' => $errors]),
            );
        }

        return $next($request);
    }
}

// ৪. Rate Limiting Middleware
class RateLimitMiddleware implements Middleware
{
    private array $requests = [];

    public function __construct(
        private readonly int $maxRequests = 60,
        private readonly int $windowSeconds = 60,
    ) {}

    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        $ip = $request->headers['X-Forwarded-For'] ?? '127.0.0.1';
        $now = time();

        // মেয়াদোত্তীর্ণ রেকর্ড মুছে ফেলো
        $this->requests[$ip] = array_filter(
            $this->requests[$ip] ?? [],
            fn(int $time) => ($now - $time) < $this->windowSeconds,
        );

        if (count($this->requests[$ip]) >= $this->maxRequests) {
            return new HttpResponse(
                statusCode: 429,
                body: json_encode(['error' => 'অনুরোধ সীমা অতিক্রম করেছে']),
                headers: ['Retry-After' => (string) $this->windowSeconds],
            );
        }

        $this->requests[$ip][] = $now;

        return $next($request);
    }
}

// Pipeline — চেইন নির্মাণ ও পরিচালনা
class Pipeline
{
    /** @var Middleware[] */
    private array $middlewares = [];

    public function pipe(Middleware $middleware): self
    {
        $this->middlewares[] = $middleware;
        return $this;
    }

    public function process(HttpRequest $request, Closure $handler): HttpResponse
    {
        // middleware চেইন রিভার্সে তৈরি করো — সবশেষেরটা আগে
        $pipeline = array_reduce(
            array_reverse($this->middlewares),
            fn(Closure $next, Middleware $middleware) =>
                fn(HttpRequest $req) => $middleware->handle($req, $next),
            $handler,
        );

        return $pipeline($request);
    }
}

// ব্যবহার
$pipeline = new Pipeline();
$pipeline
    ->pipe(new RateLimitMiddleware(maxRequests: 100))
    ->pipe(new AuthenticationMiddleware())
    ->pipe(new AuthorizationMiddleware(allowedRoles: ['admin', 'editor']))
    ->pipe(new ValidationMiddleware(rules: ['title' => 'required']));

$request = new HttpRequest(
    method: 'POST',
    path: '/api/articles',
    headers: ['Authorization' => 'Bearer valid-token-123'],
    body: ['title' => 'Chain of Responsibility বাংলায়'],
);

$response = $pipeline->process(
    $request,
    fn(HttpRequest $req) => new HttpResponse(
        statusCode: 201,
        body: json_encode(['message' => 'আর্টিকেল তৈরি হয়েছে', 'user' => $req->attributes['user']]),
    ),
);

echo "Status: {$response->statusCode}\nBody: {$response->body}\n";
```

#### JavaScript (ES2022+) — Middleware Pipeline

```javascript
// Pipeline ক্লাস — middleware চেইন পরিচালনা
class MiddlewarePipeline {
    #middlewares = [];

    use(middleware) {
        this.#middlewares.push(middleware);
        return this;
    }

    async process(request, finalHandler) {
        // reduce দিয়ে nested closure তৈরি
        const pipeline = this.#middlewares.reduceRight(
            (next, middleware) => async (req) => middleware(req, next),
            finalHandler,
        );
        return pipeline(request);
    }
}

// Middleware ফাংশন (Express.js স্টাইল)

const authMiddleware = async (request, next) => {
    const token = request.headers?.['authorization'];
    if (!token) {
        return { status: 401, body: { error: 'টোকেন প্রদান করা হয়নি' } };
    }
    // request-এ user যোগ
    request.user = { id: 1, role: 'admin', name: 'করিম' };
    return next(request);
};

const authzMiddleware = (allowedRoles) => async (request, next) => {
    if (!allowedRoles.includes(request.user?.role)) {
        return { status: 403, body: { error: 'অনুমতি নেই' } };
    }
    return next(request);
};

const validationMiddleware = (rules) => async (request, next) => {
    const errors = Object.entries(rules)
        .filter(([field, rule]) => rule === 'required' && !request.body?.[field])
        .map(([field]) => `'${field}' আবশ্যক`);

    if (errors.length > 0) {
        return { status: 422, body: { errors } };
    }
    return next(request);
};

const rateLimitMiddleware = (() => {
    const store = new Map();

    return async (request, next) => {
        const ip = request.ip ?? '127.0.0.1';
        const now = Date.now();
        const windowMs = 60_000;
        const maxReqs = 100;

        const timestamps = (store.get(ip) ?? []).filter((t) => now - t < windowMs);

        if (timestamps.length >= maxReqs) {
            return { status: 429, body: { error: 'অনুরোধ সীমা অতিক্রম' } };
        }

        timestamps.push(now);
        store.set(ip, timestamps);
        return next(request);
    };
})();

// ব্যবহার
const pipeline = new MiddlewarePipeline();
pipeline
    .use(rateLimitMiddleware)
    .use(authMiddleware)
    .use(authzMiddleware(['admin', 'editor']))
    .use(validationMiddleware({ title: 'required' }));

const request = {
    method: 'POST',
    path: '/api/articles',
    headers: { authorization: 'Bearer valid-token' },
    body: { title: 'CoR Pattern বাংলায়' },
};

const response = await pipeline.process(request, async (req) => ({
    status: 201,
    body: { message: 'তৈরি হয়েছে', user: req.user },
}));

console.log(response);
```

---

### ৩. Logging Chain — লগ লেভেল ভিত্তিক চেইন

একটি Pure CoR উদাহরণ যেখানে প্রতিটি handler নির্দিষ্ট লেভেলের লগ হ্যান্ডেল করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

enum LogLevel: int
{
    case DEBUG = 0;
    case INFO = 1;
    case WARNING = 2;
    case ERROR = 3;
    case CRITICAL = 4;
}

readonly class LogEntry
{
    public function __construct(
        public LogLevel $level,
        public string $message,
        public array $context = [],
        public DateTimeImmutable $timestamp = new DateTimeImmutable(),
    ) {}
}

abstract class LogHandler
{
    private ?LogHandler $next = null;

    public function __construct(
        protected readonly LogLevel $minimumLevel,
    ) {}

    public function setNext(LogHandler $handler): LogHandler
    {
        $this->next = $handler;
        return $handler;
    }

    public function log(LogEntry $entry): void
    {
        // এই handler-এর ন্যূনতম লেভেলে পৌঁছেছে কিনা
        if ($entry->level->value >= $this->minimumLevel->value) {
            $this->write($entry);
        }

        // Pipeline স্টাইল — সবাইকে সুযোগ দাও
        $this->next?->log($entry);
    }

    abstract protected function write(LogEntry $entry): void;
}

class ConsoleLogHandler extends LogHandler
{
    protected function write(LogEntry $entry): void
    {
        $time = $entry->timestamp->format('H:i:s');
        echo "[{$time}] [{$entry->level->name}] {$entry->message}\n";
    }
}

class FileLogHandler extends LogHandler
{
    public function __construct(
        LogLevel $minimumLevel,
        private readonly string $filePath,
    ) {
        parent::__construct($minimumLevel);
    }

    protected function write(LogEntry $entry): void
    {
        $line = sprintf(
            "[%s] [%s] %s %s\n",
            $entry->timestamp->format('Y-m-d H:i:s'),
            $entry->level->name,
            $entry->message,
            $entry->context ? json_encode($entry->context) : '',
        );
        file_put_contents($this->filePath, $line, FILE_APPEND);
    }
}

class SlackAlertHandler extends LogHandler
{
    protected function write(LogEntry $entry): void
    {
        // CRITICAL লেভেলে Slack notification পাঠাও
        echo "🚨 SLACK ALERT: [{$entry->level->name}] {$entry->message}\n";
    }
}

// চেইন গঠন
$console = new ConsoleLogHandler(LogLevel::DEBUG);
$file = new FileLogHandler(LogLevel::WARNING, '/var/log/app.log');
$slack = new SlackAlertHandler(LogLevel::CRITICAL);

$console->setNext($file)->setNext($slack);

// ব্যবহার — বিভিন্ন লেভেলের লগ
$console->log(new LogEntry(LogLevel::DEBUG, 'ডিবাগ তথ্য'));
$console->log(new LogEntry(LogLevel::ERROR, 'ডাটাবেজ সংযোগ ব্যর্থ', ['host' => 'db.local']));
$console->log(new LogEntry(LogLevel::CRITICAL, 'সার্ভার ডাউন!'));
```

#### JavaScript (ES2022+)

```javascript
const LogLevel = Object.freeze({
    DEBUG: 0,
    INFO: 1,
    WARNING: 2,
    ERROR: 3,
    CRITICAL: 4,
});

class LogHandler {
    #next = null;
    #minimumLevel;

    constructor(minimumLevel) {
        this.#minimumLevel = minimumLevel;
    }

    setNext(handler) {
        this.#next = handler;
        return handler;
    }

    log(entry) {
        if (entry.level >= this.#minimumLevel) {
            this.write(entry);
        }
        this.#next?.log(entry);
    }

    write(_entry) {
        throw new Error('Subclass must implement write()');
    }
}

class ConsoleLogHandler extends LogHandler {
    write(entry) {
        const levelName = Object.keys(LogLevel).find((k) => LogLevel[k] === entry.level);
        console.log(`[${new Date().toISOString()}] [${levelName}] ${entry.message}`);
    }
}

class WebhookAlertHandler extends LogHandler {
    write(entry) {
        // CRITICAL লেভেলে webhook call
        console.log(`🚨 WEBHOOK: ${entry.message}`);
    }
}

const console_ = new ConsoleLogHandler(LogLevel.DEBUG);
const webhook = new WebhookAlertHandler(LogLevel.CRITICAL);
console_.setNext(webhook);

console_.log({ level: LogLevel.DEBUG, message: 'ডিবাগ তথ্য' });
console_.log({ level: LogLevel.CRITICAL, message: 'সার্ভার ক্র্যাশ!' });
```

---

### ৪. Discount Calculation Chain — ছাড় গণনা

Impure CoR — প্রতিটি handler কিছু ছাড় প্রয়োগ করতে পারে এবং চেইন চলতে থাকে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

readonly class Order
{
    public function __construct(
        public float $originalPrice,
        public float $currentPrice,
        public string $customerType, // 'vip', 'regular', 'new'
        public ?string $couponCode = null,
        public int $quantity = 1,
        public bool $isSeasonal = false,
        public array $appliedDiscounts = [],
    ) {}

    public function withDiscount(float $newPrice, string $discountName): self
    {
        return new self(
            originalPrice: $this->originalPrice,
            currentPrice: $newPrice,
            customerType: $this->customerType,
            couponCode: $this->couponCode,
            quantity: $this->quantity,
            isSeasonal: $this->isSeasonal,
            appliedDiscounts: [...$this->appliedDiscounts, $discountName],
        );
    }
}

abstract class DiscountHandler
{
    private ?DiscountHandler $next = null;

    public function setNext(DiscountHandler $handler): DiscountHandler
    {
        $this->next = $handler;
        return $handler;
    }

    public function calculate(Order $order): Order
    {
        $order = $this->applyDiscount($order);
        return $this->next?->calculate($order) ?? $order;
    }

    abstract protected function applyDiscount(Order $order): Order;
}

class VipDiscountHandler extends DiscountHandler
{
    protected function applyDiscount(Order $order): Order
    {
        if ($order->customerType === 'vip') {
            $discounted = $order->currentPrice * 0.85; // ১৫% ছাড়
            return $order->withDiscount($discounted, 'VIP ছাড় (১৫%)');
        }
        return $order;
    }
}

class SeasonalDiscountHandler extends DiscountHandler
{
    protected function applyDiscount(Order $order): Order
    {
        if ($order->isSeasonal) {
            $discounted = $order->currentPrice * 0.90; // ১০% ছাড়
            return $order->withDiscount($discounted, 'সিজনাল ছাড় (১০%)');
        }
        return $order;
    }
}

class CouponDiscountHandler extends DiscountHandler
{
    private const array COUPONS = [
        'EID2024' => 0.20,    // ২০% ছাড়
        'WELCOME' => 0.10,    // ১০% ছাড়
        'POHELA' => 0.15,     // ১৫% ছাড় (পহেলা বৈশাখ)
    ];

    protected function applyDiscount(Order $order): Order
    {
        $discount = self::COUPONS[$order->couponCode] ?? null;

        if ($discount !== null) {
            $discounted = $order->currentPrice * (1 - $discount);
            $percent = (int) ($discount * 100);
            return $order->withDiscount($discounted, "কুপন '{$order->couponCode}' ({$percent}%)");
        }

        return $order;
    }
}

class BulkDiscountHandler extends DiscountHandler
{
    protected function applyDiscount(Order $order): Order
    {
        if ($order->quantity >= 10) {
            $discounted = $order->currentPrice * 0.92; // ৮% ছাড়
            return $order->withDiscount($discounted, 'বাল্ক ছাড় (৮% — ১০+ আইটেম)');
        }
        return $order;
    }
}

// চেইন গঠন
$vip = new VipDiscountHandler();
$seasonal = new SeasonalDiscountHandler();
$coupon = new CouponDiscountHandler();
$bulk = new BulkDiscountHandler();

$vip->setNext($seasonal)->setNext($coupon)->setNext($bulk);

// পরীক্ষা
$order = new Order(
    originalPrice: 10_000,
    currentPrice: 10_000,
    customerType: 'vip',
    couponCode: 'EID2024',
    quantity: 15,
    isSeasonal: true,
);

$result = $vip->calculate($order);

echo "মূল মূল্য: ৳{$result->originalPrice}\n";
echo "চূড়ান্ত মূল্য: ৳" . number_format($result->currentPrice, 2) . "\n";
echo "প্রয়োগকৃত ছাড়:\n";
foreach ($result->appliedDiscounts as $d) {
    echo "  ✓ {$d}\n";
}
```

#### JavaScript (ES2022+)

```javascript
class DiscountHandler {
    #next = null;

    setNext(handler) {
        this.#next = handler;
        return handler;
    }

    calculate(order) {
        order = this.applyDiscount(order);
        return this.#next?.calculate(order) ?? order;
    }

    applyDiscount(_order) {
        throw new Error('Must implement applyDiscount');
    }
}

class VipDiscountHandler extends DiscountHandler {
    applyDiscount(order) {
        if (order.customerType === 'vip') {
            return {
                ...order,
                currentPrice: order.currentPrice * 0.85,
                appliedDiscounts: [...order.appliedDiscounts, 'VIP ছাড় (১৫%)'],
            };
        }
        return order;
    }
}

class CouponDiscountHandler extends DiscountHandler {
    static #coupons = { EID2024: 0.2, WELCOME: 0.1, POHELA: 0.15 };

    applyDiscount(order) {
        const discount = CouponDiscountHandler.#coupons[order.couponCode];
        if (discount) {
            return {
                ...order,
                currentPrice: order.currentPrice * (1 - discount),
                appliedDiscounts: [
                    ...order.appliedDiscounts,
                    `কুপন '${order.couponCode}' (${discount * 100}%)`,
                ],
            };
        }
        return order;
    }
}

// ব্যবহার
const vip = new VipDiscountHandler();
const coupon = new CouponDiscountHandler();
vip.setNext(coupon);

const order = {
    originalPrice: 10_000,
    currentPrice: 10_000,
    customerType: 'vip',
    couponCode: 'EID2024',
    appliedDiscounts: [],
};

const result = vip.calculate(order);
console.log(`চূড়ান্ত মূল্য: ৳${result.currentPrice.toFixed(2)}`);
console.log('ছাড়সমূহ:', result.appliedDiscounts);
```

---

## 🌍 Real-World Applicable Areas

### ১. Laravel Middleware — এটাই Chain of Responsibility!

Laravel-এর সম্পূর্ণ HTTP middleware সিস্টেম CoR প্যাটার্নের উপর ভিত্তি করে নির্মিত:

```php
// app/Http/Middleware/EnsureBangladeshiIP.php
class EnsureBangladeshiIP
{
    private const array BD_IP_RANGES = ['103.', '114.130.', '27.147.'];

    public function handle(Request $request, Closure $next): Response
    {
        $ip = $request->ip();
        $isBD = array_any(
            self::BD_IP_RANGES,
            fn(string $range) => str_starts_with($ip, $range),
        );

        if (!$isBD) {
            abort(403, 'শুধুমাত্র বাংলাদেশ থেকে প্রবেশযোগ্য');
        }

        return $next($request); // পরবর্তী middleware-এ পাঠাও
    }
}
```

### ২. Express.js Middleware — `next()` ফাংশন

```javascript
// Express.js — next() হলো চেইনের পরবর্তী handler
app.use((req, res, next) => {
    console.log(`${req.method} ${req.path}`); // logging middleware
    next(); // পরবর্তী middleware-এ যাও
});

app.use((req, res, next) => {
    const token = req.headers.authorization;
    if (!token) return res.status(401).json({ error: 'Unauthorized' });
    req.user = verifyToken(token);
    next();
});
```

### ৩. DOM Event Bubbling

```javascript
// DOM-এ ইভেন্ট চাইল্ড → প্যারেন্ট চেইনে bubble করে
document.querySelector('#button').addEventListener('click', (e) => {
    console.log('Button clicked');
    // e.stopPropagation() — চেইন ভাঙো
});

document.querySelector('#container').addEventListener('click', (e) => {
    console.log('Container caught the event (bubbled up)');
});
```

### ৪. Exception Handling Chain

```php
// try-catch ব্লক নিজেই একটি chain of responsibility
try {
    riskyOperation();
} catch (DatabaseException $e) {
    // ডাটাবেজ সমস্যা হ্যান্ডেল
} catch (ValidationException $e) {
    // ভ্যালিডেশন সমস্যা হ্যান্ডেল
} catch (Exception $e) {
    // সাধারণ সমস্যা — চেইনের শেষ handler
}
```

### ৫. Approval Workflow — কর্মচারী ছুটি অনুমোদন

```
কর্মচারী (ছুটির আবেদন: ১৫ দিন)
    ↓
সুপারভাইজার → ৩ দিন পর্যন্ত অনুমোদন
    ↓ (বেশি হলে)
ম্যানেজার → ৭ দিন পর্যন্ত অনুমোদন
    ↓ (বেশি হলে)
ডিরেক্টর → ১৫ দিন পর্যন্ত অনুমোদন ✅
    ↓ (বেশি হলে)
HR প্রধান → চূড়ান্ত সিদ্ধান্ত
```

### ৬. Request Filtering/Sanitization Pipeline

```php
// ইনপুট পরিষ্কারকরণ — প্রতিটি ফিল্টার একটি ধাপ
$sanitized = Pipeline::send($userInput)
    ->through([
        TrimWhitespace::class,
        RemoveHtmlTags::class,
        NormalizeUnicode::class,    // বাংলা ইউনিকোড নরমালাইজ
        EscapeSqlCharacters::class,
    ])
    ->thenReturn();
```

---

## 🔥 Advanced Deep Dive

### Chain vs Decorator — তুলনা

উভয় প্যাটার্নই recursive composition ব্যবহার করে, কিন্তু উদ্দেশ্য ভিন্ন:

| বৈশিষ্ট্য | Chain of Responsibility | Decorator |
|-----------|------------------------|-----------|
| **উদ্দেশ্য** | অনুরোধ সঠিক handler-এ পৌঁছানো | অবজেক্টে নতুন আচরণ যোগ করা |
| **চেইন ভাঙা** | যেকোনো handler চেইন থামাতে পারে | সাধারণত সবাই প্রক্রিয়া করে |
| **প্রক্রিয়া** | একজন (বা pipeline-এ সবাই) হ্যান্ডেল করে | প্রতিটি decorator কিছু যোগ করে |
| **Interface** | handler-দের একই interface থাকে | Decorated object-এর interface মেনে চলে |
| **উদাহরণ** | Middleware, event handler | Stream wrappers, UI components |

### Chain vs Command

| Chain of Responsibility | Command |
|------------------------|---------|
| কে প্রক্রিয়া করবে জানা নেই | নির্দিষ্ট receiver আছে |
| অনুরোধ forward হয় | অনুরোধ encapsulate হয় |
| Runtime-এ handler ঠিক হয় | Compile-time-এ receiver জানা |

### Middleware Pipeline — স্ক্র্যাচ থেকে

```php
<?php

declare(strict_types=1);

// Laravel-স্টাইল Pipeline — বাস্তব বাস্তবায়ন
class Pipeline
{
    private array $pipes = [];
    private mixed $passable = null;

    public static function send(mixed $passable): self
    {
        $instance = new self();
        $instance->passable = $passable;
        return $instance;
    }

    public function through(array $pipes): self
    {
        $this->pipes = $pipes;
        return $this;
    }

    public function then(Closure $destination): mixed
    {
        // সবচেয়ে গুরুত্বপূর্ণ অংশ — reduce দিয়ে nested closures তৈরি
        $pipeline = array_reduce(
            array_reverse($this->pipes),
            $this->carry(),
            $this->prepareDestination($destination),
        );

        return $pipeline($this->passable);
    }

    public function thenReturn(): mixed
    {
        return $this->then(fn($passable) => $passable);
    }

    /**
     * carry() — প্রতিটি pipe-কে closure-এ মোড়ানোর কারখানা
     * এটাই Pipeline-এর হৃদপিণ্ড
     */
    private function carry(): Closure
    {
        return function (Closure $next, mixed $pipe): Closure {
            return function (mixed $passable) use ($next, $pipe): mixed {
                if (is_callable($pipe)) {
                    return $pipe($passable, $next);
                }

                if (is_string($pipe) && class_exists($pipe)) {
                    $instance = new $pipe();
                    return $instance->handle($passable, $next);
                }

                throw new InvalidArgumentException(
                    'Pipe অবশ্যই callable অথবা ক্লাস নাম হতে হবে'
                );
            };
        };
    }

    private function prepareDestination(Closure $destination): Closure
    {
        return fn(mixed $passable) => $destination($passable);
    }
}
```

### Bidirectional Chain — দ্বিমুখী প্রক্রিয়াকরণ

Laravel middleware-এর একটি শক্তিশালী বৈশিষ্ট্য — `$next()` কলের আগে ও পরে কোড চালানো:

```php
<?php

declare(strict_types=1);

class TimingMiddleware implements Middleware
{
    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        // ═══ BEFORE — অনুরোধ যাওয়ার আগে ═══
        $startTime = hrtime(true);

        // চেইনে পাঠাও — এবং response ফেরত আসবে
        $response = $next($request);

        // ═══ AFTER — response ফেরার পরে ═══
        $duration = (hrtime(true) - $startTime) / 1_000_000;

        echo "⏱️ Request took {$duration}ms\n";

        return $response;
    }
}

class CorsMiddleware implements Middleware
{
    public function handle(HttpRequest $request, Closure $next): HttpResponse
    {
        // BEFORE: preflight check
        if ($request->method === 'OPTIONS') {
            return new HttpResponse(
                statusCode: 204,
                body: '',
                headers: $this->corsHeaders(),
            );
        }

        $response = $next($request);

        // AFTER: CORS headers যোগ
        return new HttpResponse(
            statusCode: $response->statusCode,
            body: $response->body,
            headers: [...$response->headers, ...$this->corsHeaders()],
        );
    }

    private function corsHeaders(): array
    {
        return [
            'Access-Control-Allow-Origin' => '*',
            'Access-Control-Allow-Methods' => 'GET, POST, PUT, DELETE',
        ];
    }
}
```

```
অনুরোধ (Request) প্রবাহ:
═══════════════════════════

Request ──▶ [Timing:BEFORE] ──▶ [CORS:BEFORE] ──▶ [Auth] ──▶ Handler
                                                                 │
Response ◀── [Timing:AFTER] ◀── [CORS:AFTER] ◀── [Auth] ◀──────┘
              (সময় গণনা)       (headers যোগ)

Timing middleware:
  ├── BEFORE: startTime রেকর্ড
  └── AFTER: duration গণনা ও লগ

CORS middleware:
  ├── BEFORE: OPTIONS request হলে চেইন ভাঙো
  └── AFTER: response-এ CORS headers যোগ
```

### Priority-based Chain — অগ্রাধিকার ভিত্তিক চেইন

```php
<?php

declare(strict_types=1);

class PriorityChain
{
    /** @var array<int, array{priority: int, handler: Handler}> */
    private array $handlers = [];

    public function register(Handler $handler, int $priority = 0): self
    {
        $this->handlers[] = ['priority' => $priority, 'handler' => $handler];
        return $this;
    }

    public function buildChain(): Handler
    {
        // উচ্চ priority আগে execute হবে
        usort(
            $this->handlers,
            fn(array $a, array $b) => $b['priority'] <=> $a['priority'],
        );

        $handlers = array_column($this->handlers, 'handler');

        // চেইন গঠন
        for ($i = 0; $i < count($handlers) - 1; $i++) {
            $handlers[$i]->setNext($handlers[$i + 1]);
        }

        return $handlers[0]; // চেইনের শুরু
    }
}

// ব্যবহার
$chain = new PriorityChain();
$chain
    ->register(new AuthenticationMiddleware(), priority: 100)  // সর্বোচ্চ
    ->register(new LoggingMiddleware(), priority: 50)
    ->register(new CacheMiddleware(), priority: 30)
    ->register(new ValidationMiddleware(), priority: 80);

// গঠিত চেইন: Auth(100) → Validation(80) → Logging(50) → Cache(30)
$firstHandler = $chain->buildChain();
```

```javascript
// JavaScript — Priority Chain
class PriorityChain {
    #handlers = [];

    register(handler, priority = 0) {
        this.#handlers.push({ handler, priority });
        return this;
    }

    build() {
        const sorted = [...this.#handlers].sort((a, b) => b.priority - a.priority);
        for (let i = 0; i < sorted.length - 1; i++) {
            sorted[i].handler.setNext(sorted[i + 1].handler);
        }
        return sorted[0].handler;
    }
}
```

---

## ✅ Pros (সুবিধা)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| 1 | **Loose Coupling** | প্রেরক ও গ্রহীতা একে অপর সম্পর্কে জানে না |
| 2 | **Single Responsibility** | প্রতিটি handler একটি নির্দিষ্ট দায়িত্ব পালন করে |
| 3 | **Open/Closed** | নতুন handler যোগ করতে বিদ্যমান কোড পরিবর্তন লাগে না |
| 4 | **Dynamic Configuration** | রানটাইমে চেইন পরিবর্তন করা যায় |
| 5 | **Reusability** | handler-গুলো বিভিন্ন চেইনে পুনরায় ব্যবহারযোগ্য |
| 6 | **Flexibility** | চেইনের ক্রম সহজে পরিবর্তনযোগ্য |

## ❌ Cons (অসুবিধা)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| 1 | **কোনো গ্যারান্টি নেই** | অনুরোধ হ্যান্ডেল না হয়ে হারিয়ে যেতে পারে |
| 2 | **ডিবাগিং কঠিন** | লম্বা চেইনে কোথায় সমস্যা তা খুঁজে বের করা কঠিন |
| 3 | **পারফরম্যান্স** | খুব লম্বা চেইনে পারফরম্যান্স সমস্যা হতে পারে |
| 4 | **অনিশ্চয়তা** | কোন handler প্রক্রিয়া করবে তা আগে থেকে জানা যায় না |
| 5 | **অর্ডার নির্ভরতা** | চেইনের ক্রম গুরুত্বপূর্ণ, ভুল ক্রমে অপ্রত্যাশিত আচরণ |

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ভুল ১: চেইনের শেষে fallback handler না রাখা

```php
// ❌ ভুল — অনুরোধ হারিয়ে যেতে পারে
$handler1->setNext($handler2); // handler2 ও হ্যান্ডেল না করলে null

// ✅ সঠিক — সবসময় একটি fallback/default handler রাখুন
$handler1->setNext($handler2)->setNext(new DefaultHandler());

class DefaultHandler extends Handler
{
    public function handle(Request $request): Response
    {
        // চেইনে কেউ হ্যান্ডেল না করলে এটি কাজ করবে
        return new Response(
            approved: false,
            handledBy: 'সিস্টেম',
            message: 'কোনো handler অনুরোধ হ্যান্ডেল করতে পারেনি',
        );
    }
}
```

### ভুল ২: চেইনে circular reference তৈরি

```php
// ❌ ভুল — অসীম লুপ!
$handlerA->setNext($handlerB);
$handlerB->setNext($handlerA); // 💥 Stack overflow!

// ✅ সঠিক — চেইনে visited tracking রাখুন
abstract class SafeHandler extends Handler
{
    private static array $visited = [];

    public function handle(Request $request): ?Response
    {
        $id = spl_object_id($this);

        if (in_array($id, self::$visited, true)) {
            throw new RuntimeException('Circular chain detected!');
        }

        self::$visited[] = $id;

        try {
            return $this->process($request);
        } finally {
            self::$visited = []; // cleanup
        }
    }

    abstract protected function process(Request $request): ?Response;
}
```

### ভুল ৩: handler-এ state রাখা (thread-safety সমস্যা)

```php
// ❌ ভুল — handler-এ mutable state
class CountingHandler extends Handler
{
    private int $count = 0; // concurrent request-এ সমস্যা!

    public function handle(Request $request): ?Response
    {
        $this->count++;
        return parent::handle($request);
    }
}

// ✅ সঠিক — stateless handler, state বাইরে রাখুন
class CountingHandler extends Handler
{
    public function __construct(
        private readonly CounterService $counter,
    ) {}

    public function handle(Request $request): ?Response
    {
        $this->counter->increment(); // thread-safe service
        return parent::handle($request);
    }
}
```

### ভুল ৪: খুব লম্বা চেইন তৈরি করা

```php
// ❌ ভুল — ২০+ handler একটি চেইনে
$h1->setNext($h2)->setNext($h3)->...->>setNext($h20);

// ✅ সঠিক — সংশ্লিষ্ট handler-গুলো গ্রুপ করুন
$securityChain = buildSecurityChain();   // Auth → Authz → RateLimit
$validationChain = buildValidationChain(); // Input → Schema → Business
$processingChain = buildProcessingChain(); // Transform → Store → Notify

// গ্রুপ চেইন সংযুক্ত
$securityChain->setNext($validationChain)->setNext($processingChain);
```

### ভুল ৫: Middleware-এ `$next()` কল ভুলে যাওয়া

```php
// ❌ ভুল — $next() কল করা হয়নি, চেইন থেমে যাবে!
class LogMiddleware implements Middleware
{
    public function handle(Request $request, Closure $next): Response
    {
        Log::info('Request received');
        // return $next($request); ← এটা ভুলে গেছে!
        return new Response(200, 'OK'); // বাকি middleware skip!
    }
}
```

---

## 🧪 টেস্টিং

### PHPUnit টেস্ট

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class LoanApprovalChainTest extends TestCase
{
    private BranchOfficerHandler $chain;

    protected function setUp(): void
    {
        $this->chain = new BranchOfficerHandler();
        $branchManager = new BranchManagerHandler();
        $regionalManager = new RegionalManagerHandler();
        $headOffice = new HeadOfficeHandler();

        $this->chain
            ->setNext($branchManager)
            ->setNext($regionalManager)
            ->setNext($headOffice);
    }

    #[Test]
    public function branchOfficerHandlesSmallLoans(): void
    {
        $request = new Request(type: 'personal', amount: 300_000);
        $response = $this->chain->handle($request);

        $this->assertNotNull($response);
        $this->assertTrue($response->approved);
        $this->assertSame('ব্রাঞ্চ অফিসার', $response->handledBy);
    }

    #[Test]
    public function requestEscalatesToCorrectHandler(): void
    {
        $request = new Request(type: 'business', amount: 5_000_000);
        $response = $this->chain->handle($request);

        $this->assertNotNull($response);
        $this->assertTrue($response->approved);
        $this->assertSame('রিজিওনাল ম্যানেজার', $response->handledBy);
    }

    #[Test]
    public function veryLargeLoanIsRejected(): void
    {
        $request = new Request(type: 'corporate', amount: 500_000_000);
        $response = $this->chain->handle($request);

        $this->assertNotNull($response);
        $this->assertFalse($response->approved);
    }

    #[Test]
    #[DataProvider('loanAmountProvider')]
    public function correctHandlerProcessesEachAmount(
        float $amount,
        string $expectedHandler,
    ): void {
        $request = new Request(type: 'general', amount: $amount);
        $response = $this->chain->handle($request);

        $this->assertSame($expectedHandler, $response->handledBy);
    }

    public static function loanAmountProvider(): array
    {
        return [
            '৩ লক্ষ — ব্রাঞ্চ অফিসার' => [300_000, 'ব্রাঞ্চ অফিসার'],
            '১৫ লক্ষ — ব্রাঞ্চ ম্যানেজার' => [1_500_000, 'ব্রাঞ্চ ম্যানেজার'],
            '৫০ লক্ষ — রিজিওনাল' => [5_000_000, 'রিজিওনাল ম্যানেজার'],
            '৫ কোটি — হেড অফিস' => [50_000_000, 'প্রধান কার্যালয়'],
        ];
    }
}

class MiddlewarePipelineTest extends TestCase
{
    #[Test]
    public function pipelineExecutesAllMiddlewaresInOrder(): void
    {
        $order = [];

        $pipeline = new Pipeline();
        $pipeline
            ->pipe(new class implements Middleware {
                public function handle(HttpRequest $req, Closure $next): HttpResponse
                {
                    // before
                    $req = $req->withAttribute('step1', true);
                    $response = $next($req);
                    // after — bidirectional!
                    return $response;
                }
            })
            ->pipe(new class implements Middleware {
                public function handle(HttpRequest $req, Closure $next): HttpResponse
                {
                    $req = $req->withAttribute('step2', true);
                    return $next($req);
                }
            });

        $response = $pipeline->process(
            new HttpRequest(method: 'GET', path: '/test'),
            function (HttpRequest $req) {
                $this->assertTrue($req->attributes['step1']);
                $this->assertTrue($req->attributes['step2']);
                return new HttpResponse(200, 'OK');
            },
        );

        $this->assertSame(200, $response->statusCode);
    }

    #[Test]
    public function middlewareCanBreakChain(): void
    {
        $pipeline = new Pipeline();
        $pipeline
            ->pipe(new class implements Middleware {
                public function handle(HttpRequest $req, Closure $next): HttpResponse
                {
                    // চেইন ভাঙা — $next() কল করা হচ্ছে না
                    return new HttpResponse(401, 'Unauthorized');
                }
            })
            ->pipe(new class implements Middleware {
                public function handle(HttpRequest $req, Closure $next): HttpResponse
                {
                    throw new RuntimeException('এই middleware-এ কখনো পৌঁছানো উচিত না');
                }
            });

        $response = $pipeline->process(
            new HttpRequest(method: 'GET', path: '/secret'),
            fn() => new HttpResponse(200, 'OK'),
        );

        $this->assertSame(401, $response->statusCode);
    }
}
```

### Jest টেস্ট

```javascript
describe('Chain of Responsibility — Loan Approval', () => {
    let chain;

    beforeEach(() => {
        chain = new BranchOfficerHandler();
        const manager = new BranchManagerHandler();
        const regional = new RegionalManagerHandler();
        chain.setNext(manager).setNext(regional);
    });

    test('ব্রাঞ্চ অফিসার ছোট লোন হ্যান্ডেল করে', () => {
        const result = chain.handle({ type: 'personal', amount: 300_000 });
        expect(result.approved).toBe(true);
        expect(result.handledBy).toBe('ব্রাঞ্চ অফিসার');
    });

    test('বড় লোন সঠিক handler-এ escalate হয়', () => {
        const result = chain.handle({ type: 'business', amount: 5_000_000 });
        expect(result.approved).toBe(true);
        expect(result.handledBy).toBe('রিজিওনাল ম্যানেজার');
    });

    test('সীমার বাইরের লোনে null রিটার্ন', () => {
        const result = chain.handle({ type: 'mega', amount: 500_000_000 });
        expect(result).toBeNull();
    });

    test.each([
        [300_000, 'ব্রাঞ্চ অফিসার'],
        [1_500_000, 'ব্রাঞ্চ ম্যানেজার'],
        [5_000_000, 'রিজিওনাল ম্যানেজার'],
    ])('৳%i → %s', (amount, expectedHandler) => {
        const result = chain.handle({ type: 'test', amount });
        expect(result.handledBy).toBe(expectedHandler);
    });
});

describe('Middleware Pipeline', () => {
    test('সকল middleware ক্রমানুসারে execute হয়', async () => {
        const executionOrder = [];

        const pipeline = new MiddlewarePipeline();
        pipeline
            .use(async (req, next) => {
                executionOrder.push('A:before');
                const res = await next(req);
                executionOrder.push('A:after');
                return res;
            })
            .use(async (req, next) => {
                executionOrder.push('B:before');
                const res = await next(req);
                executionOrder.push('B:after');
                return res;
            });

        await pipeline.process({ path: '/test' }, async () => {
            executionOrder.push('handler');
            return { status: 200 };
        });

        expect(executionOrder).toEqual([
            'A:before',
            'B:before',
            'handler',
            'B:after',
            'A:after',
        ]);
    });

    test('middleware চেইন ভাঙতে পারে', async () => {
        const pipeline = new MiddlewarePipeline();
        pipeline
            .use(async (_req, _next) => {
                return { status: 401, body: 'Unauthorized' };
                // next() কল করা হয়নি — চেইন ভাঙা
            })
            .use(async () => {
                throw new Error('এখানে পৌঁছানো উচিত না');
            });

        const response = await pipeline.process({}, async () => ({ status: 200 }));
        expect(response.status).toBe(401);
    });

    test('middleware request-এ ডেটা যোগ করতে পারে', async () => {
        const pipeline = new MiddlewarePipeline();
        pipeline.use(async (req, next) => {
            req.user = { id: 1, name: 'করিম' };
            return next(req);
        });

        const response = await pipeline.process({}, async (req) => ({
            status: 200,
            user: req.user,
        }));

        expect(response.user.name).toBe('করিম');
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্নসমূহ

### Command Pattern

**সম্পর্ক:** Command অনুরোধকে অবজেক্টে encapsulate করে, CoR সেই অনুরোধ সঠিক handler-এ পৌঁছে দেয়। একসাথে ব্যবহার করলে Command অবজেক্ট চেইনে পাস হয়।

```
[Command Object] ──▶ [Handler A] ──▶ [Handler B] ──▶ [Handler C]
```

### Composite Pattern

**সম্পর্ক:** Composite-এর tree structure-এ child থেকে parent-এ অনুরোধ forward করা CoR-এর একটি রূপ (DOM event bubbling)।

### Decorator Pattern

**সম্পর্ক:** দুটোই linked objects ব্যবহার করে। Decorator সবসময় wrapped object-কে কল করে; CoR-এ handler চেইন থামাতে পারে।

### Mediator Pattern

**সম্পর্ক:** Mediator কেন্দ্রীয়ভাবে objects-এর মধ্যে যোগাযোগ পরিচালনা করে; CoR-এ অনুরোধ ক্রমানুসারে objects-এর মধ্য দিয়ে যায়। Mediator "star topology", CoR "linear chain"।

```
Mediator:                    Chain of Responsibility:
    ┌─── A                   A ──▶ B ──▶ C ──▶ D
    │
M ──┼─── B
    │
    └─── C
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

- **একাধিক অবজেক্ট** অনুরোধ হ্যান্ডেল করতে পারে, এবং কোনটি হ্যান্ডেল করবে তা আগে থেকে জানা যায় না
- **ক্রমানুসারে প্রক্রিয়াকরণ** প্রয়োজন (middleware pipeline)
- **Handler-দের সেট ডায়নামিকভাবে** পরিবর্তন করা দরকার
- **Sender-Receiver decoupling** প্রয়োজন
- **Approval workflow** — বিভিন্ন authority level-এ অনুমোদন
- **Validation chain** — একাধিক ধাপে ইনপুট যাচাই
- **Event processing** — বিভিন্ন handler বিভিন্ন ইভেন্ট প্রক্রিয়া করে

### ❌ ব্যবহার করবেন না যখন:

- **সরাসরি রাউটিং** সম্ভব — যদি if-else বা map দিয়ে সরাসরি handler ঠিক করা যায়
- **পারফরম্যান্স-ক্রিটিক্যাল** কোডে খুব লম্বা চেইন ব্যবহার করা ঠিক না
- **প্রতিটি অনুরোধ অবশ্যই হ্যান্ডেল হতে হবে** — CoR-এ অনুরোধ হারিয়ে যাওয়ার ঝুঁকি আছে (fallback ছাড়া)
- **চেইনের ক্রম গুরুত্বপূর্ণ নয়** — Strategy pattern বেশি উপযুক্ত
- **একটিমাত্র handler** — সরাসরি কল করুন, চেইনের দরকার নেই

---

## 📋 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────┐
│              Chain of Responsibility সারসংক্ষেপ            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ধরন: Behavioral Pattern                                 │
│  উদ্দেশ্য: Sender-Receiver decoupling                    │
│  মূলনীতি: অনুরোধ চেইন বরাবর পাস হয়                      │
│                                                          │
│  তিনটি রূপ:                                              │
│  ├── Pure CoR (একজন হ্যান্ডেল করে)                        │
│  ├── Pipeline (সবাই প্রক্রিয়া করে)                       │
│  └── Bidirectional (আসা-যাওয়া দুই পথে)                  │
│                                                          │
│  বাস্তব প্রয়োগ:                                          │
│  ├── Laravel Middleware                                   │
│  ├── Express.js next()                                   │
│  ├── DOM Event Bubbling                                  │
│  ├── Exception Handling                                  │
│  └── Approval Workflows                                  │
│                                                          │
│  মনে রাখুন:                                              │
│  ├── সবসময় fallback handler রাখুন                        │
│  ├── Circular reference এড়িয়ে চলুন                       │
│  ├── Handler stateless রাখুন                             │
│  ├── চেইন খুব লম্বা করবেন না                              │
│  └── $next() কল করতে ভুলবেন না (pipeline-এ)              │
│                                                          │
│  সম্পর্কিত: Command, Composite, Decorator, Mediator      │
│                                                          │
│  বাংলাদেশ প্রসঙ্গ:                                       │
│  ├── ব্যাংক লোন অনুমোদন (অফিসার→ম্যানেজার→রিজিওনাল)    │
│  ├── সরকারি ফাইল (ইউনিয়ন→উপজেলা→জেলা→মন্ত্রণালয়)       │
│  └── গ্রাহক সেবা (L1→L2→L3→ম্যানেজার)                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

> **"Chain of Responsibility মূলত বাস্তব জীবনের দায়িত্ব হস্তান্তরের ডিজিটাল রূপ — আপনার অভিযোগ যেভাবে এক ডেস্ক থেকে আরেক ডেস্কে যায়, ঠিক তেমনই অনুরোধ এক handler থেকে আরেক handler-এ যায়, যতক্ষণ না সঠিক কেউ সমাধান করে।"**

---

*📚 আরও পড়ুন: [Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns) — Gamma, Helm, Johnson, Vlissides (GoF)*
