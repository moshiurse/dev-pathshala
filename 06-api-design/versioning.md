# 🔢 API ভার্সনিং (API Versioning)

## 📌 সংজ্ঞা ও প্রয়োজনীয়তা

API ভার্সনিং হলো একটি কৌশল যার মাধ্যমে আমরা API-এর বিভিন্ন সংস্করণ একই সময়ে চালু রাখতে পারি। যখন আপনার API-তে পরিবর্তন আনতে হয়, তখন পুরনো ক্লায়েন্টরা যেন ভেঙে না পড়ে সেটা নিশ্চিত করাই ভার্সনিং-এর মূল উদ্দেশ্য।

### কেন API ভার্সনিং দরকার?

ধরুন, bKash-এর Payment API v1.0 দিয়ে হাজার হাজার মার্চেন্ট ইন্টিগ্রেশন করেছে। এখন আপনি response structure পরিবর্তন করতে চাইছেন। ভার্সনিং ছাড়া এই পরিবর্তন সব মার্চেন্টের সিস্টেম ভেঙে দেবে।

**মূল কারণসমূহ:**
- **Backward Compatibility** — পুরনো ক্লায়েন্ট যেন কাজ করতে থাকে
- **Parallel Evolution** — নতুন ফিচার যোগ করার সময় পুরনো ভার্সন সচল রাখা
- **Controlled Migration** — ক্লায়েন্টদের নিজের সুবিধামতো নতুন ভার্সনে যাওয়ার সুযোগ
- **Contract Stability** — API consumer-দের সাথে চুক্তি রক্ষা

### Breaking vs Non-Breaking Changes

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANGE CLASSIFICATION                        │
├─────────────────────────────┬───────────────────────────────────┤
│   🔴 Breaking Changes       │   🟢 Non-Breaking Changes         │
│   (নতুন ভার্সন দরকার)       │   (ভার্সন না বাড়ালেও চলে)        │
├─────────────────────────────┼───────────────────────────────────┤
│ • Field মুছে ফেলা           │ • নতুন optional field যোগ         │
│ • Field-এর type পরিবর্তন   │ • নতুন endpoint যোগ               │
│ • Required field যোগ করা    │ • নতুন optional query param       │
│ • URL structure পরিবর্তন    │ • Response-এ নতুন field যোগ       │
│ • Response format পরিবর্তন  │ • নতুন HTTP method support        │
│ • Error code পরিবর্তন       │ • Performance উন্নতি              │
│ • Authentication পরিবর্তন   │ • Bug fix (contract অনুযায়ী)      │
│ • Enum value মুছে ফেলা      │ • নতুন enum value যোগ             │
└─────────────────────────────┴───────────────────────────────────┘
```

### বাস্তব উদাহরণ — bKash API Migration

```
bKash Payment API v1.0 Response:
{
    "txnId": "TXN123",
    "amount": "500",        ← string হিসেবে amount
    "status": "completed"
}

bKash Payment API v2.0 Response:
{
    "transaction_id": "TXN123",   ← field নাম পরিবর্তন (breaking!)
    "amount": 500,                ← number টাইপে পরিবর্তন (breaking!)
    "status": "completed",
    "currency": "BDT",            ← নতুন field (non-breaking)
    "metadata": {}                ← নতুন field (non-breaking)
}
```

---

## 📊 Versioning Decision Flow

```
                        API-তে পরিবর্তন দরকার?
                                │
                                ▼
                    ┌───────────────────────┐
                    │ পরিবর্তনটি কি         │
                    │ backward compatible?   │
                    └───────────┬───────────┘
                       ▼                ▼
                      হ্যাঁ             না
                       │                │
                       ▼                ▼
              ┌──────────────┐  ┌──────────────────┐
              │ বর্তমান       │  │ এটি কি শুধু       │
              │ ভার্সনেই      │  │ minor পরিবর্তন?   │
              │ deploy করুন   │  └────────┬─────────┘
              └──────────────┘      ▼           ▼
                                   হ্যাঁ         না
                                    │            │
                                    ▼            ▼
                          ┌────────────┐  ┌──────────────────┐
                          │ Minor       │  │ Major version     │
                          │ version     │  │ বাড়ান              │
                          │ বাড়ান       │  │ (v1 → v2)         │
                          │(v1.1→v1.2) │  └────────┬─────────┘
                          └────────────┘           │
                                                   ▼
                                        ┌──────────────────┐
                                        │ Deprecation       │
                                        │ timeline সেট করুন │
                                        │ পুরনো ভার্সনের জন্য│
                                        └──────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │          কোন Strategy ব্যবহার করবেন?                     │
    │                                                         │
    │  Public API? ──────────► URI Path Versioning             │
    │  Internal API? ────────► Header Versioning               │
    │  Microservice? ────────► Header / Content Negotiation    │
    │  Simple API? ──────────► Query Parameter                 │
    │  Flexible Evolution? ──► GraphQL (versionless)           │
    └─────────────────────────────────────────────────────────┘
```

---

## 💻 Versioning Strategies

### ১. URI Path Versioning

এটি সবচেয়ে জনপ্রিয় এবং সহজবোধ্য পদ্ধতি। URL path-এ সরাসরি version number থাকে।

```
GET /api/v1/users
GET /api/v2/users
GET /api/v1/payments/bkash
GET /api/v2/payments/bkash
```

#### Laravel Implementation

```php
// routes/api.php

// ভার্সন ১ — মূল route group
Route::prefix('v1')->group(function () {
    Route::apiResource('users', App\Http\Controllers\V1\UserController::class);
    Route::apiResource('payments', App\Http\Controllers\V1\PaymentController::class);

    Route::prefix('payments')->group(function () {
        Route::post('bkash/initiate', [App\Http\Controllers\V1\BkashController::class, 'initiate']);
        Route::post('bkash/execute/{paymentId}', [App\Http\Controllers\V1\BkashController::class, 'execute']);
    });
});

// ভার্সন ২ — নতুন features সহ
Route::prefix('v2')->group(function () {
    Route::apiResource('users', App\Http\Controllers\V2\UserController::class);
    Route::apiResource('payments', App\Http\Controllers\V2\PaymentController::class);

    Route::prefix('payments')->group(function () {
        Route::post('bkash/initiate', [App\Http\Controllers\V2\BkashController::class, 'initiate']);
        Route::post('bkash/execute/{paymentId}', [App\Http\Controllers\V2\BkashController::class, 'execute']);
        Route::post('bkash/refund/{paymentId}', [App\Http\Controllers\V2\BkashController::class, 'refund']);
    });
});
```

```php
// app/Http/Controllers/V1/UserController.php
namespace App\Http\Controllers\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\V1\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        $users = User::paginate(15);
        return UserResource::collection($users);
    }

    public function show(User $user)
    {
        return new UserResource($user);
    }
}

// app/Http/Resources/V1/UserResource.php
namespace App\Http\Resources\V1;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'    => $this->id,
            'name'  => $this->name,
            'email' => $this->email,
        ];
    }
}
```

```php
// app/Http/Controllers/V2/UserController.php
namespace App\Http\Controllers\V2;

use App\Http\Controllers\Controller;
use App\Http\Resources\V2\UserResource;
use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        $users = User::with(['profile', 'roles'])->paginate(20);
        return UserResource::collection($users);
    }

    public function show(User $user)
    {
        $user->load(['profile', 'roles', 'permissions']);
        return new UserResource($user);
    }
}

// app/Http/Resources/V2/UserResource.php
namespace App\Http\Resources\V2;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'         => $this->id,
            'full_name'  => $this->name,        // field নাম পরিবর্তন
            'email'      => $this->email,
            'avatar_url' => $this->profile?->avatar_url,
            'roles'      => $this->roles->pluck('name'),
            'metadata'   => [
                'created_at' => $this->created_at->toISOString(),
                'updated_at' => $this->updated_at->toISOString(),
            ],
        ];
    }
}
```

#### Express Implementation

```javascript
// routes/v1/users.js
const express = require('express');
const router = express.Router();
const UserControllerV1 = require('../../controllers/v1/UserController');

router.get('/', UserControllerV1.index);
router.get('/:id', UserControllerV1.show);
router.post('/', UserControllerV1.store);
router.put('/:id', UserControllerV1.update);
router.delete('/:id', UserControllerV1.destroy);

module.exports = router;

// routes/v2/users.js
const express = require('express');
const router = express.Router();
const UserControllerV2 = require('../../controllers/v2/UserController');

router.get('/', UserControllerV2.index);
router.get('/:id', UserControllerV2.show);
router.post('/', UserControllerV2.store);
router.put('/:id', UserControllerV2.update);
router.patch('/:id', UserControllerV2.patch);
router.delete('/:id', UserControllerV2.destroy);

module.exports = router;

// app.js — মূল router mounting
const express = require('express');
const app = express();

const v1UserRoutes = require('./routes/v1/users');
const v1PaymentRoutes = require('./routes/v1/payments');
const v2UserRoutes = require('./routes/v2/users');
const v2PaymentRoutes = require('./routes/v2/payments');

app.use('/api/v1/users', v1UserRoutes);
app.use('/api/v1/payments', v1PaymentRoutes);
app.use('/api/v2/users', v2UserRoutes);
app.use('/api/v2/payments', v2PaymentRoutes);

app.listen(3000, () => console.log('Server running on port 3000'));
```

```javascript
// controllers/v1/UserController.js
const User = require('../../models/User');

module.exports = {
    async index(req, res) {
        const page = parseInt(req.query.page) || 1;
        const limit = 15;
        const users = await User.find()
            .skip((page - 1) * limit)
            .limit(limit)
            .select('name email');

        res.json({
            data: users.map(u => ({
                id: u._id,
                name: u.name,
                email: u.email,
            })),
            page,
        });
    },

    async show(req, res) {
        const user = await User.findById(req.params.id).select('name email');
        if (!user) return res.status(404).json({ error: 'User not found' });

        res.json({ id: user._id, name: user.name, email: user.email });
    },
};

// controllers/v2/UserController.js
const User = require('../../models/User');

module.exports = {
    async index(req, res) {
        const page = parseInt(req.query.page) || 1;
        const limit = 20;
        const users = await User.find()
            .populate('profile roles')
            .skip((page - 1) * limit)
            .limit(limit);

        res.json({
            data: users.map(u => ({
                id: u._id,
                full_name: u.name,
                email: u.email,
                avatar_url: u.profile?.avatarUrl || null,
                roles: u.roles.map(r => r.name),
                metadata: {
                    created_at: u.createdAt.toISOString(),
                    updated_at: u.updatedAt.toISOString(),
                },
            })),
            meta: { page, limit, total: await User.countDocuments() },
        });
    },

    async show(req, res) {
        const user = await User.findById(req.params.id)
            .populate('profile roles permissions');
        if (!user) return res.status(404).json({ error: 'User not found' });

        res.json({
            id: user._id,
            full_name: user.name,
            email: user.email,
            avatar_url: user.profile?.avatarUrl || null,
            roles: user.roles.map(r => r.name),
            permissions: user.permissions.map(p => p.name),
            metadata: {
                created_at: user.createdAt.toISOString(),
                updated_at: user.updatedAt.toISOString(),
            },
        });
    },
};
```

**সুবিধা:** URL দেখেই version বোঝা যায়, caching সহজ, ডিবাগিং সুবিধাজনক
**অসুবিধা:** URL পরিবর্তন হয়, ক্লায়েন্ট URL আপডেট করতে হয়, resource URI-এর REST নীতি ভঙ্গ হতে পারে

---

### ২. Query Parameter Versioning

URL-এ query parameter হিসেবে version পাঠানো হয়।

```
GET /api/users?version=1
GET /api/users?version=2
GET /api/payments/bkash?version=2&type=initiate
```

#### Laravel Implementation

```php
// app/Http/Middleware/ApiVersionMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class ApiVersionMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $version = $request->query('version', '1'); // default v1

        if (!in_array($version, ['1', '2', '3'])) {
            return response()->json([
                'error' => 'Unsupported API version',
                'supported_versions' => ['1', '2', '3'],
            ], 400);
        }

        $request->attributes->set('api_version', (int) $version);
        return $next($request);
    }
}

// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index(Request $request)
    {
        $version = $request->attributes->get('api_version', 1);
        $users = User::paginate(15);

        return match ($version) {
            1 => $this->transformV1($users),
            2 => $this->transformV2($users),
            default => $this->transformV2($users),
        };
    }

    private function transformV1($users)
    {
        return response()->json([
            'data' => $users->map(fn ($u) => [
                'id'    => $u->id,
                'name'  => $u->name,
                'email' => $u->email,
            ]),
        ]);
    }

    private function transformV2($users)
    {
        return response()->json([
            'data' => $users->map(fn ($u) => [
                'id'        => $u->id,
                'full_name' => $u->name,
                'email'     => $u->email,
                'metadata'  => [
                    'created_at' => $u->created_at->toISOString(),
                ],
            ]),
            'meta' => [
                'current_page' => $users->currentPage(),
                'total'        => $users->total(),
            ],
        ]);
    }
}

// routes/api.php
Route::middleware(['api.version'])->group(function () {
    Route::apiResource('users', UserController::class);
    Route::apiResource('payments', PaymentController::class);
});
```

#### Express Implementation

```javascript
// middleware/apiVersion.js
function apiVersionMiddleware(req, res, next) {
    const version = parseInt(req.query.version) || 1;
    const supported = [1, 2, 3];

    if (!supported.includes(version)) {
        return res.status(400).json({
            error: 'Unsupported API version',
            supported_versions: supported,
        });
    }

    req.apiVersion = version;
    next();
}

module.exports = apiVersionMiddleware;

// controllers/UserController.js
const User = require('../models/User');

const transformers = {
    1: (user) => ({
        id: user._id,
        name: user.name,
        email: user.email,
    }),
    2: (user) => ({
        id: user._id,
        full_name: user.name,
        email: user.email,
        metadata: {
            created_at: user.createdAt.toISOString(),
            updated_at: user.updatedAt.toISOString(),
        },
    }),
};

module.exports = {
    async index(req, res) {
        const users = await User.find().limit(20);
        const transform = transformers[req.apiVersion] || transformers[2];
        res.json({ data: users.map(transform) });
    },
};

// app.js
const apiVersion = require('./middleware/apiVersion');
const userController = require('./controllers/UserController');

app.use('/api', apiVersion);
app.get('/api/users', userController.index);
```

**সুবিধা:** URL path পরিবর্তন হয় না, implement করা সহজ
**অসুবিধা:** query param ভুলে গেলে default version চলে, caching জটিল হতে পারে, URL কম পরিষ্কার

---

### ৩. Header Versioning (Custom Media Type)

HTTP header ব্যবহার করে version নির্ধারণ করা হয়। এটি REST-এর দৃষ্টিকোণ থেকে সবচেয়ে "সঠিক" পদ্ধতি।

```
GET /api/users
Accept: application/vnd.myapp.v1+json

GET /api/users
Accept: application/vnd.myapp.v2+json
```

#### Laravel Implementation

```php
// app/Http/Middleware/HeaderVersionMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class HeaderVersionMiddleware
{
    private const VERSION_PATTERN = '/application\/vnd\.myapp\.v(\d+)\+json/';
    private const DEFAULT_VERSION = 1;
    private const MAX_VERSION = 3;

    public function handle(Request $request, Closure $next)
    {
        $accept = $request->header('Accept', '');
        $version = self::DEFAULT_VERSION;

        if (preg_match(self::VERSION_PATTERN, $accept, $matches)) {
            $version = (int) $matches[1];
        }

        // Custom header fallback: X-API-Version
        if ($version === self::DEFAULT_VERSION && $request->hasHeader('X-API-Version')) {
            $version = (int) $request->header('X-API-Version');
        }

        if ($version < 1 || $version > self::MAX_VERSION) {
            return response()->json([
                'error'   => 'অসমর্থিত API সংস্করণ',
                'message' => "Version {$version} সমর্থিত নয়। v1-v" . self::MAX_VERSION . " ব্যবহার করুন।",
            ], 406);
        }

        $request->attributes->set('api_version', $version);

        $response = $next($request);
        $response->headers->set('X-API-Version', (string) $version);
        $response->headers->set('Content-Type', "application/vnd.myapp.v{$version}+json");

        return $response;
    }
}

// app/Providers/RouteServiceProvider.php — middleware নিবন্ধন
// Kernel.php বা bootstrap/app.php এ:
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'api.version.header' => \App\Http\Middleware\HeaderVersionMiddleware::class,
    ]);
})
```

#### Express Implementation

```javascript
// middleware/headerVersion.js
const VERSION_REGEX = /application\/vnd\.myapp\.v(\d+)\+json/;
const DEFAULT_VERSION = 1;
const MAX_VERSION = 3;

function headerVersionMiddleware(req, res, next) {
    const accept = req.get('Accept') || '';
    let version = DEFAULT_VERSION;

    const match = accept.match(VERSION_REGEX);
    if (match) {
        version = parseInt(match[1]);
    } else if (req.get('X-API-Version')) {
        version = parseInt(req.get('X-API-Version'));
    }

    if (version < 1 || version > MAX_VERSION) {
        return res.status(406).json({
            error: 'Unsupported API version',
            message: `Version ${version} is not supported. Use v1-v${MAX_VERSION}.`,
        });
    }

    req.apiVersion = version;

    // response header-এ version তথ্য যোগ
    res.set('X-API-Version', String(version));
    res.set('Content-Type', `application/vnd.myapp.v${version}+json`);

    next();
}

module.exports = headerVersionMiddleware;

// app.js
const headerVersion = require('./middleware/headerVersion');

app.use('/api', headerVersion);
app.get('/api/users', (req, res) => {
    const controller = require(`./controllers/v${req.apiVersion}/UserController`);
    controller.index(req, res);
});
```

**সুবিধা:** Clean URL, REST নীতি মেনে চলে, URL এবং resource representation আলাদা থাকে
**অসুবিধা:** ব্রাউজারে টেস্ট করা কঠিন, ডিবাগিং জটিল, ক্লায়েন্টদের header সেট করতে হয়

---

### ৪. Content Negotiation

Accept header ব্যবহার করে কোন format ও version-এ response চাই তা নির্ধারণ করা।

```
GET /api/users
Accept: application/json; version=2

GET /api/users
Accept: application/json; version=1; fields=id,name
```

#### Laravel Implementation

```php
// app/Http/Middleware/ContentNegotiationMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class ContentNegotiationMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $accept = $request->header('Accept', 'application/json');
        $version = 1;
        $format = 'json';

        // Accept: application/json; version=2 parse করা
        if (preg_match('/version=(\d+)/', $accept, $matches)) {
            $version = (int) $matches[1];
        }

        if (str_contains($accept, 'xml')) {
            $format = 'xml';
        }

        $request->attributes->set('api_version', $version);
        $request->attributes->set('response_format', $format);

        return $next($request);
    }
}

// app/Http/Controllers/PaymentController.php
namespace App\Http\Controllers;

use App\Models\Payment;
use Illuminate\Http\Request;

class PaymentController extends Controller
{
    public function show(Request $request, Payment $payment)
    {
        $version = $request->attributes->get('api_version');
        $format = $request->attributes->get('response_format');

        $data = match ($version) {
            1 => [
                'txnId'  => $payment->transaction_id,
                'amount' => (string) $payment->amount,
                'status' => $payment->status,
            ],
            2 => [
                'transaction_id' => $payment->transaction_id,
                'amount'         => (float) $payment->amount,
                'currency'       => $payment->currency ?? 'BDT',
                'status'         => $payment->status,
                'provider'       => $payment->provider,
                'metadata'       => $payment->metadata,
            ],
            default => abort(406, 'Unsupported version'),
        };

        if ($format === 'xml') {
            return response($this->toXml($data), 200)
                ->header('Content-Type', 'application/xml');
        }

        return response()->json($data);
    }

    private function toXml(array $data): string
    {
        $xml = new \SimpleXMLElement('<response/>');
        array_walk_recursive($data, function ($value, $key) use ($xml) {
            $xml->addChild($key, htmlspecialchars((string) $value));
        });
        return $xml->asXML();
    }
}
```

#### Express Implementation

```javascript
// middleware/contentNegotiation.js
function contentNegotiation(req, res, next) {
    const accept = req.get('Accept') || 'application/json';
    let version = 1;
    let format = 'json';

    const versionMatch = accept.match(/version=(\d+)/);
    if (versionMatch) version = parseInt(versionMatch[1]);

    if (accept.includes('xml')) format = 'xml';

    req.apiVersion = version;
    req.responseFormat = format;

    // response helper যোগ করা
    res.negotiate = (data) => {
        if (req.responseFormat === 'xml') {
            res.set('Content-Type', 'application/xml');
            return res.send(jsonToXml(data));
        }
        return res.json(data);
    };

    next();
}

function jsonToXml(obj, rootName = 'response') {
    let xml = `<${rootName}>`;
    for (const [key, value] of Object.entries(obj)) {
        if (typeof value === 'object' && value !== null) {
            xml += jsonToXml(value, key);
        } else {
            xml += `<${key}>${value}</${key}>`;
        }
    }
    xml += `</${rootName}>`;
    return xml;
}

module.exports = contentNegotiation;
```

**সুবিধা:** সবচেয়ে নমনীয়, format ও version একসাথে নিয়ন্ত্রণ করা যায়
**অসুবিধা:** বাস্তবায়ন জটিল, header parsing ভুল হওয়ার সম্ভাবনা

---

## 🔧 Implementation Deep Dive

### Laravel: সম্পূর্ণ Versioning Architecture

#### ফোল্ডার স্ট্রাকচার

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── V1/
│   │   │   ├── UserController.php
│   │   │   ├── PaymentController.php
│   │   │   └── BkashController.php
│   │   ├── V2/
│   │   │   ├── UserController.php
│   │   │   ├── PaymentController.php
│   │   │   └── BkashController.php
│   │   └── BaseController.php
│   ├── Middleware/
│   │   ├── ApiVersionMiddleware.php
│   │   └── DeprecationWarningMiddleware.php
│   └── Resources/
│       ├── V1/
│       │   ├── UserResource.php
│       │   └── PaymentResource.php
│       └── V2/
│           ├── UserResource.php
│           └── PaymentResource.php
├── Services/
│   ├── UserService.php          ← shared business logic
│   └── PaymentService.php
routes/
├── api_v1.php
├── api_v2.php
└── api.php
```

#### Route Provider সেটআপ

```php
// app/Providers/RouteServiceProvider.php
namespace App\Providers;

use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Route;

class RouteServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->routes(function () {
            // V1 Routes
            Route::middleware(['api', 'api.deprecation:v1'])
                ->prefix('api/v1')
                ->as('v1.')
                ->group(base_path('routes/api_v1.php'));

            // V2 Routes (সক্রিয়)
            Route::middleware('api')
                ->prefix('api/v2')
                ->as('v2.')
                ->group(base_path('routes/api_v2.php'));
        });
    }
}
```

#### Deprecation Warning Middleware

```php
// app/Http/Middleware/DeprecationWarningMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class DeprecationWarningMiddleware
{
    private const SUNSET_DATES = [
        'v1' => '2025-06-01',
        'v2' => null, // এখনও সক্রিয়
    ];

    public function handle(Request $request, Closure $next, string $version)
    {
        $response = $next($request);

        $sunset = self::SUNSET_DATES[$version] ?? null;
        if ($sunset) {
            $response->headers->set('Deprecation', 'true');
            $response->headers->set('Sunset', $sunset);
            $response->headers->set(
                'Link',
                '</api/v2>; rel="successor-version"'
            );
            $response->headers->set(
                'X-Deprecation-Notice',
                "{$version} will be retired on {$sunset}. Please migrate to the latest version."
            );
        }

        return $response;
    }
}
```

#### Shared Service Layer

```php
// app/Services/PaymentService.php
namespace App\Services;

use App\Models\Payment;

class PaymentService
{
    // উভয় version-ই একই business logic ব্যবহার করে
    public function initiateBkashPayment(array $data): Payment
    {
        $payment = Payment::create([
            'provider'       => 'bkash',
            'amount'         => $data['amount'],
            'currency'       => $data['currency'] ?? 'BDT',
            'status'         => 'initiated',
            'transaction_id' => $this->generateTransactionId(),
            'metadata'       => $data['metadata'] ?? [],
        ]);

        // bKash API কল...
        $this->callBkashGateway($payment);

        return $payment;
    }

    public function executeBkashPayment(string $paymentId): Payment
    {
        $payment = Payment::where('transaction_id', $paymentId)
            ->where('status', 'initiated')
            ->firstOrFail();

        $payment->update(['status' => 'completed']);
        return $payment;
    }

    private function generateTransactionId(): string
    {
        return 'TXN' . strtoupper(bin2hex(random_bytes(8)));
    }

    private function callBkashGateway(Payment $payment): void
    {
        // bKash sandbox/live API integration
    }
}
```

#### Version-specific Controller Pattern

```php
// app/Http/Controllers/V1/BkashController.php
namespace App\Http\Controllers\V1;

use App\Http\Controllers\Controller;
use App\Services\PaymentService;
use Illuminate\Http\Request;

class BkashController extends Controller
{
    public function __construct(private PaymentService $paymentService) {}

    public function initiate(Request $request)
    {
        $request->validate([
            'amount' => 'required|numeric|min:1',
        ]);

        $payment = $this->paymentService->initiateBkashPayment([
            'amount' => $request->amount,
        ]);

        // V1 response format
        return response()->json([
            'txnId'  => $payment->transaction_id,
            'amount' => (string) $payment->amount,
            'status' => $payment->status,
        ], 201);
    }
}

// app/Http/Controllers/V2/BkashController.php
namespace App\Http\Controllers\V2;

use App\Http\Controllers\Controller;
use App\Http\Resources\V2\PaymentResource;
use App\Services\PaymentService;
use Illuminate\Http\Request;

class BkashController extends Controller
{
    public function __construct(private PaymentService $paymentService) {}

    public function initiate(Request $request)
    {
        $request->validate([
            'amount'   => 'required|numeric|min:1',
            'currency' => 'sometimes|string|in:BDT,USD',
            'metadata' => 'sometimes|array',
        ]);

        $payment = $this->paymentService->initiateBkashPayment($request->all());

        // V2 response — Resource class ব্যবহার
        return (new PaymentResource($payment))
            ->response()
            ->setStatusCode(201);
    }

    public function refund(Request $request, string $paymentId)
    {
        // V2-তে নতুন endpoint
        $request->validate([
            'reason' => 'required|string|max:500',
        ]);

        $payment = $this->paymentService->refundPayment($paymentId, $request->reason);
        return new PaymentResource($payment);
    }
}
```

---

### Express: সম্পূর্ণ Versioning Architecture

#### ফোল্ডার স্ট্রাকচার

```
src/
├── controllers/
│   ├── v1/
│   │   ├── UserController.js
│   │   ├── PaymentController.js
│   │   └── BkashController.js
│   └── v2/
│       ├── UserController.js
│       ├── PaymentController.js
│       └── BkashController.js
├── middleware/
│   ├── versionRouter.js
│   ├── deprecation.js
│   └── versionNegotiator.js
├── routes/
│   ├── v1/
│   │   └── index.js
│   └── v2/
│   │   └── index.js
│   └── index.js
├── services/
│   ├── UserService.js
│   └── PaymentService.js
└── app.js
```

#### Dynamic Version Router

```javascript
// src/middleware/versionRouter.js
const fs = require('fs');
const path = require('path');

function versionRouter(routesDir) {
    const express = require('express');
    const router = express.Router();

    // routes ফোল্ডার থেকে সব version স্বয়ংক্রিয়ভাবে লোড
    const versions = fs.readdirSync(routesDir)
        .filter(f => fs.statSync(path.join(routesDir, f)).isDirectory())
        .sort();

    for (const version of versions) {
        const versionRoutes = require(path.join(routesDir, version));
        router.use(`/${version}`, versionRoutes);
        console.log(`✅ API ${version} routes loaded`);
    }

    return router;
}

module.exports = versionRouter;

// src/app.js
const express = require('express');
const versionRouter = require('./middleware/versionRouter');
const deprecation = require('./middleware/deprecation');
const path = require('path');

const app = express();
app.use(express.json());

// deprecation middleware সব deprecated version-এ
app.use('/api/v1', deprecation({
    version: 'v1',
    sunset: '2025-06-01',
    successor: '/api/v2',
}));

// dynamic version routing
app.use('/api', versionRouter(path.join(__dirname, 'routes')));

// fallback — version ছাড়া request এলে latest version-এ redirect
app.use('/api/users', (req, res) => {
    res.redirect(307, `/api/v2/users${req.url === '/' ? '' : req.url}`);
});

app.listen(3000);
```

#### Deprecation Middleware

```javascript
// src/middleware/deprecation.js
function deprecation({ version, sunset, successor }) {
    return (req, res, next) => {
        res.set('Deprecation', 'true');
        res.set('Sunset', sunset);
        res.set('Link', `<${successor}>; rel="successor-version"`);
        res.set(
            'X-Deprecation-Notice',
            `${version} will be retired on ${sunset}. Please migrate to ${successor}.`
        );
        next();
    };
}

module.exports = deprecation;
```

#### Version Negotiator (Advanced)

```javascript
// src/middleware/versionNegotiator.js
const semver = require('semver');

const VERSION_MAP = {
    'v1': '1.0.0',
    'v2': '2.0.0',
    'v3': '3.0.0',
};

const LATEST_VERSION = 'v2';

function versionNegotiator(req, res, next) {
    let version = null;

    // ১. URI path থেকে version চেক
    const pathMatch = req.path.match(/\/v(\d+)/);
    if (pathMatch) {
        version = `v${pathMatch[1]}`;
    }

    // ২. Header থেকে version চেক
    if (!version) {
        const acceptVersion = req.get('Accept-Version');
        if (acceptVersion) {
            const matched = Object.entries(VERSION_MAP)
                .find(([, semVer]) => semver.satisfies(semVer, acceptVersion));
            if (matched) version = matched[0];
        }
    }

    // ৩. Query param থেকে version চেক
    if (!version && req.query.version) {
        version = `v${req.query.version}`;
    }

    // ৪. Default version
    req.apiVersion = version || LATEST_VERSION;
    req.apiSemver = VERSION_MAP[req.apiVersion] || VERSION_MAP[LATEST_VERSION];

    res.set('X-API-Version', req.apiVersion);
    next();
}

module.exports = versionNegotiator;
```

---

## 🔥 Advanced Topics

### ১. Semantic Versioning for APIs

API-তে Semantic Versioning (SemVer) ব্যবহার করলে পরিবর্তনের ধরন স্পষ্ট হয়:

```
MAJOR.MINOR.PATCH
  │     │     │
  │     │     └── Bug fix, backward compatible
  │     └──────── নতুন feature, backward compatible
  └────────────── Breaking changes
```

```php
// app/Http/Middleware/SemverMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class SemverMiddleware
{
    private const VERSIONS = [
        '1.0.0' => ['status' => 'deprecated', 'sunset' => '2025-03-01'],
        '1.1.0' => ['status' => 'deprecated', 'sunset' => '2025-06-01'],
        '2.0.0' => ['status' => 'current'],
        '2.1.0' => ['status' => 'current'],
        '3.0.0' => ['status' => 'beta'],
    ];

    public function handle(Request $request, Closure $next)
    {
        $requested = $request->header('Accept-Version', '2.1.0');

        // exact match বা compatible version খোঁজা
        $resolved = $this->resolveVersion($requested);

        if (!$resolved) {
            return response()->json([
                'error'              => 'Version not found',
                'requested'          => $requested,
                'available_versions' => array_keys(self::VERSIONS),
            ], 406);
        }

        $request->attributes->set('api_semver', $resolved);
        $request->attributes->set('api_major', (int) explode('.', $resolved)[0]);

        $response = $next($request);
        $response->headers->set('X-API-Version', $resolved);

        $versionInfo = self::VERSIONS[$resolved];
        if ($versionInfo['status'] === 'deprecated') {
            $response->headers->set('Deprecation', 'true');
            $response->headers->set('Sunset', $versionInfo['sunset']);
        }

        return $response;
    }

    private function resolveVersion(string $requested): ?string
    {
        // exact match
        if (isset(self::VERSIONS[$requested])) {
            return $requested;
        }

        // major version match — সর্বোচ্চ minor version
        $major = explode('.', $requested)[0];
        $compatible = array_filter(
            array_keys(self::VERSIONS),
            fn ($v) => str_starts_with($v, "{$major}.")
        );

        return $compatible ? max($compatible) : null;
    }
}
```

---

### ২. Deprecation Strategy

সঠিক deprecation strategy ক্লায়েন্টদের সুষ্ঠু migration নিশ্চিত করে:

```
┌──────────────────────────────────────────────────────────────────┐
│                   API VERSION LIFECYCLE                          │
│                                                                  │
│  Beta ──► Current ──► Deprecated ──► Sunset ──► Removed          │
│   │         │            │             │           │              │
│   │      ৬-১২ মাস     ৩-৬ মাস      ১ মাস        │              │
│   │     সক্রিয় ব্যবহার  সতর্কতা      চূড়ান্ত       সম্পূর্ণ      │
│   │                    header       সতর্কতা       অপসারণ         │
│   │                                                              │
│  পরীক্ষামূলক,          Sunset header   ৪১০ Gone                  │
│  পরিবর্তন হতে পারে     Deprecation      response                 │
│                        header                                    │
└──────────────────────────────────────────────────────────────────┘
```

```javascript
// src/middleware/lifecycle.js
const VERSION_LIFECYCLE = {
    v1: {
        status: 'sunset',
        deprecatedAt: '2024-01-01',
        sunsetAt: '2025-01-01',
        successor: 'v2',
        migrationGuide: 'https://docs.example.com/migration/v1-to-v2',
    },
    v2: {
        status: 'current',
        releasedAt: '2024-01-01',
    },
    v3: {
        status: 'beta',
        releasedAt: '2024-12-01',
        notice: 'Beta — এই version পরিবর্তন হতে পারে।',
    },
};

function lifecycleMiddleware(req, res, next) {
    const version = req.apiVersion || 'v2';
    const lifecycle = VERSION_LIFECYCLE[version];

    if (!lifecycle) {
        return res.status(404).json({ error: `Version ${version} does not exist` });
    }

    switch (lifecycle.status) {
        case 'sunset':
            // Sunset date পেরিয়ে গেলে 410 Gone
            if (new Date() > new Date(lifecycle.sunsetAt)) {
                return res.status(410).json({
                    error: `${version} has been permanently removed.`,
                    migration_guide: lifecycle.migrationGuide,
                    successor: lifecycle.successor,
                });
            }
            res.set('Deprecation', `@${lifecycle.deprecatedAt}`);
            res.set('Sunset', lifecycle.sunsetAt);
            res.set('Link', `<${lifecycle.migrationGuide}>; rel="deprecation"`);
            break;

        case 'beta':
            res.set('X-API-Warn', lifecycle.notice);
            break;
    }

    next();
}

module.exports = lifecycleMiddleware;
```

---

### ৩. API Changelog Management

```php
// app/Services/ApiChangelog.php
namespace App\Services;

class ApiChangelog
{
    private const CHANGELOG = [
        '2.1.0' => [
            'date'    => '2025-01-15',
            'changes' => [
                ['type' => 'added', 'description' => 'Refund endpoint যোগ করা হয়েছে'],
                ['type' => 'added', 'description' => 'Pagination meta তথ্য যোগ'],
                ['type' => 'fixed', 'description' => 'Amount precision fix'],
            ],
        ],
        '2.0.0' => [
            'date'    => '2024-07-01',
            'changes' => [
                ['type' => 'breaking', 'description' => 'User response-এ name → full_name পরিবর্তন'],
                ['type' => 'breaking', 'description' => 'Amount string থেকে number-এ পরিবর্তন'],
                ['type' => 'added',    'description' => 'Metadata object যোগ'],
                ['type' => 'added',    'description' => 'Role-based response filtering'],
            ],
        ],
        '1.0.0' => [
            'date'    => '2023-01-01',
            'changes' => [
                ['type' => 'initial', 'description' => 'প্রাথমিক API release'],
            ],
        ],
    ];

    public static function getChangelog(?string $since = null): array
    {
        if (!$since) {
            return self::CHANGELOG;
        }

        return array_filter(
            self::CHANGELOG,
            fn ($_, $version) => version_compare($version, $since, '>'),
            ARRAY_FILTER_USE_BOTH
        );
    }

    public static function getBreakingChanges(string $fromVersion, string $toVersion): array
    {
        $changes = [];
        foreach (self::CHANGELOG as $version => $entry) {
            if (version_compare($version, $fromVersion, '>') &&
                version_compare($version, $toVersion, '<=')) {
                $breaking = array_filter(
                    $entry['changes'],
                    fn ($c) => $c['type'] === 'breaking'
                );
                if ($breaking) {
                    $changes[$version] = array_values($breaking);
                }
            }
        }
        return $changes;
    }
}

// Changelog endpoint
Route::get('api/changelog', function (Request $request) {
    $since = $request->query('since');
    return response()->json(ApiChangelog::getChangelog($since));
});

Route::get('api/changelog/breaking', function (Request $request) {
    return response()->json(
        ApiChangelog::getBreakingChanges(
            $request->query('from', '1.0.0'),
            $request->query('to', '2.1.0')
        )
    );
});
```

---

### ৪. Version Negotiation Middleware (Laravel)

```php
// app/Http/Middleware/VersionNegotiator.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class VersionNegotiator
{
    // অগ্রাধিকার ক্রম: URI > Header > Query > Default
    public function handle(Request $request, Closure $next)
    {
        $version = $this->fromUri($request)
            ?? $this->fromHeader($request)
            ?? $this->fromQuery($request)
            ?? $this->defaultVersion();

        $controllerNamespace = "App\\Http\\Controllers\\V{$version}";

        $request->attributes->set('api_version', $version);
        $request->attributes->set('controller_namespace', $controllerNamespace);

        return $next($request);
    }

    private function fromUri(Request $request): ?int
    {
        if (preg_match('/\/v(\d+)\//', $request->getPathInfo(), $m)) {
            return (int) $m[1];
        }
        return null;
    }

    private function fromHeader(Request $request): ?int
    {
        if (preg_match('/version=(\d+)/', $request->header('Accept', ''), $m)) {
            return (int) $m[1];
        }
        if ($v = $request->header('X-API-Version')) {
            return (int) $v;
        }
        return null;
    }

    private function fromQuery(Request $request): ?int
    {
        return $request->has('version') ? (int) $request->query('version') : null;
    }

    private function defaultVersion(): int
    {
        return (int) config('api.default_version', 2);
    }
}
```

---

### ৫. Database Schema Versioning with API Versions

API version পরিবর্তনের সাথে database schema কিভাবে evolve করবে:

```php
// database/migrations — ভার্সন অনুযায়ী migration

// V1 এর জন্য মূল migration
// 2023_01_01_create_payments_table.php
Schema::create('payments', function (Blueprint $table) {
    $table->id();
    $table->string('transaction_id')->unique();
    $table->string('amount');     // V1-এ string ছিল
    $table->string('status');
    $table->timestamps();
});

// V2 এর জন্য migration — column type পরিবর্তন না করে নতুন column যোগ
// 2024_07_01_add_v2_fields_to_payments.php
Schema::table('payments', function (Blueprint $table) {
    $table->decimal('amount_decimal', 10, 2)->nullable();
    $table->string('currency', 3)->default('BDT');
    $table->string('provider')->nullable();
    $table->json('metadata')->nullable();
});

// Model-এ accessor দিয়ে version অনুযায়ী data serve করা
// app/Models/Payment.php
class Payment extends Model
{
    // V2 amount accessor — decimal column ব্যবহার, fallback string column
    public function getAmountV2Attribute(): float
    {
        return $this->amount_decimal ?? (float) $this->amount;
    }

    // Data sync — পুরনো ও নতুন column সিঙ্ক রাখা
    protected static function booted()
    {
        static::saving(function (Payment $payment) {
            if ($payment->isDirty('amount')) {
                $payment->amount_decimal = (float) $payment->amount;
            }
            if ($payment->isDirty('amount_decimal')) {
                $payment->amount = (string) $payment->amount_decimal;
            }
        });
    }
}
```

---

### ৬. Backward Compatible Changes (Additive Pattern)

```javascript
// Tolerant Reader Pattern — ক্লায়েন্ট অজানা field উপেক্ষা করবে

// সার্ভার — নতুন field যোগ করা (non-breaking)
// controllers/v2/UserController.js
module.exports = {
    async show(req, res) {
        const user = await User.findById(req.params.id);

        const response = {
            id: user._id,
            full_name: user.name,
            email: user.email,
            // নতুন optional fields — পুরনো ক্লায়েন্ট এগুলো উপেক্ষা করবে
            phone: user.phone || null,
            preferences: user.preferences || {},
            verified: user.emailVerified || false,
        };

        res.json(response);
    },
};

// Robustness Principle অনুসরণ — "উদার হোন গ্রহণে, কঠোর হোন প্রেরণে"
// middleware/tolerantInput.js
function tolerantInput(schema) {
    return (req, res, next) => {
        // শুধু পরিচিত fields রাখা, বাকি সব উপেক্ষা
        const cleaned = {};
        for (const [key, rules] of Object.entries(schema)) {
            if (req.body[key] !== undefined) {
                cleaned[key] = req.body[key];
            } else if (rules.default !== undefined) {
                cleaned[key] = rules.default;
            }
        }
        req.cleanBody = cleaned;
        next();
    };
}

// ব্যবহার
app.post('/api/v2/users', tolerantInput({
    name:     { required: true },
    email:    { required: true },
    phone:    { default: null },
    role:     { default: 'user' },
}), userController.store);
```

---

### ৭. API Evolution Without Versioning

#### GraphQL Approach

```javascript
// GraphQL — ভার্সনিং ছাড়া API evolution

const { ApolloServer, gql } = require('apollo-server-express');

const typeDefs = gql`
    type User {
        id: ID!
        fullName: String!
        email: String!
        phone: String
        avatar: String

        # deprecated field — ক্লায়েন্টকে জানানো হচ্ছে
        name: String @deprecated(reason: "fullName ব্যবহার করুন")
    }

    type Payment {
        id: ID!
        transactionId: String!
        amount: Float!
        currency: String!
        status: PaymentStatus!
        provider: String!
        metadata: JSON

        # deprecated
        txnId: String @deprecated(reason: "transactionId ব্যবহার করুন")
    }

    enum PaymentStatus {
        INITIATED
        COMPLETED
        FAILED
        REFUNDED
    }

    type Query {
        user(id: ID!): User
        users(limit: Int = 20, offset: Int = 0): [User!]!
        payment(id: ID!): Payment
    }

    type Mutation {
        initiateBkashPayment(input: BkashPaymentInput!): Payment!
    }

    input BkashPaymentInput {
        amount: Float!
        currency: String = "BDT"
        metadata: JSON
    }
`;

const resolvers = {
    User: {
        // পুরনো field backward compatibility-র জন্য
        name: (user) => user.fullName,
    },
    Payment: {
        txnId: (payment) => payment.transactionId,
    },
};
```

#### Feature Flags Approach

```php
// config/api_features.php
return [
    'v2_response_format' => env('API_V2_RESPONSE', true),
    'enhanced_pagination' => env('API_ENHANCED_PAGINATION', true),
    'payment_refund' => env('API_PAYMENT_REFUND', false),
    'user_roles_in_response' => env('API_USER_ROLES', true),
];

// app/Http/Controllers/UserController.php
class UserController extends Controller
{
    public function index(Request $request)
    {
        $users = User::paginate(20);

        $data = $users->map(function ($user) {
            $item = [
                'id'    => $user->id,
                'email' => $user->email,
            ];

            // Feature flag দিয়ে ধীরে ধীরে নতুন format-এ migration
            if (config('api_features.v2_response_format')) {
                $item['full_name'] = $user->name;
                $item['metadata'] = ['created_at' => $user->created_at->toISOString()];
            } else {
                $item['name'] = $user->name;
            }

            if (config('api_features.user_roles_in_response')) {
                $item['roles'] = $user->roles->pluck('name');
            }

            return $item;
        });

        return response()->json(['data' => $data]);
    }
}
```

---

### ৮. Multi-Version Maintenance (Code Organization)

```
# Pattern ১: Inheritance-based (কোড পুনঃব্যবহার সর্বাধিক)

app/Http/Controllers/
├── BaseUserController.php      ← shared logic
├── V1/
│   └── UserController.php      ← extends Base, override শুধু response
└── V2/
    └── UserController.php      ← extends Base, নতুন method যোগ
```

```php
// app/Http/Controllers/BaseUserController.php
namespace App\Http\Controllers;

use App\Models\User;
use App\Services\UserService;

abstract class BaseUserController extends Controller
{
    public function __construct(protected UserService $userService) {}

    public function index()
    {
        $users = $this->userService->getPaginatedUsers();
        return $this->formatCollection($users);
    }

    public function show(int $id)
    {
        $user = $this->userService->findUser($id);
        return $this->formatSingle($user);
    }

    abstract protected function formatCollection($users);
    abstract protected function formatSingle(User $user);
}

// app/Http/Controllers/V1/UserController.php
namespace App\Http\Controllers\V1;

use App\Http\Controllers\BaseUserController;
use App\Http\Resources\V1\UserResource;
use App\Models\User;

class UserController extends BaseUserController
{
    protected function formatCollection($users)
    {
        return UserResource::collection($users);
    }

    protected function formatSingle(User $user)
    {
        return new UserResource($user);
    }
}

// app/Http/Controllers/V2/UserController.php
namespace App\Http\Controllers\V2;

use App\Http\Controllers\BaseUserController;
use App\Http\Resources\V2\UserResource;
use App\Models\User;

class UserController extends BaseUserController
{
    protected function formatCollection($users)
    {
        return UserResource::collection($users);
    }

    protected function formatSingle(User $user)
    {
        $user->load(['profile', 'roles']);
        return new UserResource($user);
    }

    // V2 exclusive method
    public function searchByRole(string $role)
    {
        $users = $this->userService->findByRole($role);
        return UserResource::collection($users);
    }
}
```

```javascript
// Express — Mixin-based multi-version pattern

// controllers/base/userMixin.js
const UserService = require('../../services/UserService');

const userMixin = {
    async getUsers(options = {}) {
        return UserService.paginate(options);
    },

    async getUser(id) {
        return UserService.findById(id);
    },

    async createUser(data) {
        return UserService.create(data);
    },
};

module.exports = userMixin;

// controllers/v1/UserController.js
const mixin = require('../base/userMixin');

module.exports = {
    async index(req, res) {
        const users = await mixin.getUsers({ limit: 15 });
        res.json({
            data: users.map(u => ({ id: u._id, name: u.name, email: u.email })),
        });
    },
};

// controllers/v2/UserController.js
const mixin = require('../base/userMixin');

module.exports = {
    async index(req, res) {
        const users = await mixin.getUsers({
            limit: 20,
            populate: ['profile', 'roles'],
        });
        res.json({
            data: users.map(u => ({
                id: u._id,
                full_name: u.name,
                email: u.email,
                roles: u.roles?.map(r => r.name) || [],
                metadata: {
                    created_at: u.createdAt,
                    updated_at: u.updatedAt,
                },
            })),
            meta: { total: await mixin.countUsers() },
        });
    },

    // V2 exclusive
    async searchByRole(req, res) {
        const users = await mixin.getUsers({
            filter: { 'roles.name': req.params.role },
        });
        res.json({ data: users });
    },
};
```

---

### ৯. API Version Lifecycle Management

```php
// app/Services/ApiVersionManager.php
namespace App\Services;

use Illuminate\Support\Carbon;

class ApiVersionManager
{
    private const LIFECYCLE = [
        'v1' => [
            'released'    => '2023-01-01',
            'deprecated'  => '2024-07-01',
            'sunset'      => '2025-06-01',
            'status'      => 'deprecated',
            'successor'   => 'v2',
            'support'     => 'security-only',
        ],
        'v2' => [
            'released'    => '2024-07-01',
            'deprecated'  => null,
            'sunset'      => null,
            'status'      => 'stable',
            'successor'   => null,
            'support'     => 'full',
        ],
        'v3' => [
            'released'    => '2025-01-01',
            'deprecated'  => null,
            'sunset'      => null,
            'status'      => 'beta',
            'successor'   => null,
            'support'     => 'best-effort',
        ],
    ];

    public function getVersionStatus(string $version): array
    {
        $info = self::LIFECYCLE[$version] ?? null;
        if (!$info) {
            return ['error' => 'Unknown version'];
        }

        $daysUntilSunset = null;
        if ($info['sunset']) {
            $daysUntilSunset = Carbon::now()->diffInDays(
                Carbon::parse($info['sunset']),
                false
            );
        }

        return [
            ...$info,
            'days_until_sunset' => $daysUntilSunset,
            'is_active'         => in_array($info['status'], ['stable', 'beta']),
            'is_recommended'    => $info['status'] === 'stable',
        ];
    }

    public function getAllVersions(): array
    {
        return array_map(
            fn ($v) => $this->getVersionStatus($v),
            array_keys(self::LIFECYCLE)
        );
    }

    public function getMigrationPath(string $from, string $to): array
    {
        $path = [];
        $current = $from;

        while ($current !== $to && isset(self::LIFECYCLE[$current])) {
            $successor = self::LIFECYCLE[$current]['successor'];
            if (!$successor) break;

            $path[] = [
                'from' => $current,
                'to'   => $successor,
                'guide' => "/docs/migration/{$current}-to-{$successor}",
            ];
            $current = $successor;
        }

        return $path;
    }
}

// API endpoint — version তথ্য
Route::get('api/versions', function (ApiVersionManager $manager) {
    return response()->json($manager->getAllVersions());
});

Route::get('api/versions/{version}/migration', function (
    ApiVersionManager $manager,
    string $version,
    Request $request
) {
    $target = $request->query('to', 'v2');
    return response()->json($manager->getMigrationPath($version, $target));
});
```

---

## 🆚 Strategy Comparison Table

```
┌──────────────────┬────────────┬──────────┬──────────┬──────────┬───────────┐
│ বৈশিষ্ট্য         │ URI Path   │ Query    │ Header   │ Content  │ GraphQL   │
│                  │            │ Param    │          │ Negot.   │ (No Ver.) │
├──────────────────┼────────────┼──────────┼──────────┼──────────┼───────────┤
│ বাস্তবায়ন সহজতা   │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐     │ ⭐⭐       │ ⭐⭐        │
│ URL পরিচ্ছন্নতা    │ ⭐⭐⭐      │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐   │
│ Cache সহজতা      │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐     │ ⭐⭐       │ ⭐⭐       │ ⭐⭐        │
│ Client সহজতা     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐     │ ⭐⭐       │ ⭐⭐⭐      │
│ Browser test     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐ │ ⭐⭐       │ ⭐⭐       │ ⭐⭐⭐      │
│ REST compliance  │ ⭐⭐⭐      │ ⭐⭐⭐     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │ N/A       │
│ নমনীয়তা          │ ⭐⭐⭐      │ ⭐⭐⭐     │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐   │
│ Discoverability  │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐     │ ⭐⭐       │ ⭐⭐       │ ⭐⭐⭐⭐    │
├──────────────────┼────────────┼──────────┼──────────┼──────────┼───────────┤
│ উপযুক্ত ক্ষেত্র    │ Public API │ Simple/  │ Internal │ Complex  │ Flexible  │
│                  │ bKash,     │ Internal │ Micro-   │ Multi-   │ Mobile    │
│                  │ Pathao     │ tools    │ services │ format   │ apps      │
└──────────────────┴────────────┴──────────┴──────────┴──────────┴───────────┘
```

---

## ✅ Best Practices

### ১. ভার্সনিং নীতিমালা

```
✅ করণীয়:
├── সর্বদা একটি default version রাখুন
├── Semantic versioning অনুসরণ করুন
├── Deprecation notice কমপক্ষে ৬ মাস আগে দিন
├── Sunset header ব্যবহার করুন
├── Migration guide তৈরি করুন
├── Changelog maintain করুন
├── ক্লায়েন্টদের email/webhook-এ জানান
├── API version discovery endpoint রাখুন
├── Breaking change ছাড়া version বাড়াবেন না
└── পুরনো version-এর automated test চালু রাখুন

✅ Bangladesh Context:
├── bKash/Nagad integration-এ সবসময় URI path versioning ব্যবহার
├── Payment gateway-তে backward compatibility সর্বোচ্চ গুরুত্ব
├── মার্চেন্টদের জন্য বাংলায় migration guide তৈরি করুন
└── SSL Commerz, ShurjoPay-এর মতো provider-এর version support যাচাই
```

### ২. Code Organization

```php
// config/api.php — কেন্দ্রীয় version configuration
return [
    'default_version' => env('API_DEFAULT_VERSION', 2),
    'supported_versions' => [1, 2],
    'latest_version' => 2,
    'deprecation_policy' => [
        'notice_period_months' => 6,
        'sunset_period_months' => 3,
    ],
];
```

### ৩. Testing Strategy

```php
// tests/Feature/ApiVersionTest.php
class ApiVersionTest extends TestCase
{
    /** @dataProvider versionProvider */
    public function test_each_version_returns_correct_format(int $version, array $expectedKeys): void
    {
        $user = User::factory()->create();

        $response = $this->getJson("/api/v{$version}/users/{$user->id}");

        $response->assertOk();
        foreach ($expectedKeys as $key) {
            $response->assertJsonStructure([$key]);
        }
    }

    public static function versionProvider(): array
    {
        return [
            'v1' => [1, ['id', 'name', 'email']],
            'v2' => [2, ['id', 'full_name', 'email', 'metadata']],
        ];
    }

    public function test_deprecated_version_returns_sunset_headers(): void
    {
        $response = $this->getJson('/api/v1/users');

        $response->assertHeader('Deprecation', 'true');
        $response->assertHeader('Sunset');
    }

    public function test_unsupported_version_returns_error(): void
    {
        $this->getJson('/api/v99/users')
            ->assertStatus(404);
    }

    public function test_backward_compatibility_v1_to_v2(): void
    {
        $user = User::factory()->create(['name' => 'রহিম']);

        $v1 = $this->getJson("/api/v1/users/{$user->id}")->json();
        $v2 = $this->getJson("/api/v2/users/{$user->id}")->json();

        // v1 ও v2 উভয়ই একই user-এর data দেয়
        $this->assertEquals($v1['id'], $v2['id']);
        $this->assertEquals($v1['name'], $v2['full_name']);
    }
}
```

---

## ⚠️ Common Mistakes

```
❌ ভুল                                    ✅ সঠিক
─────────────────────────────────────────────────────────────────────
❌ প্রতিটি ছোট পরিবর্তনে                    ✅ শুধু breaking change-এ
   version বাড়ানো                            version বাড়ান

❌ ভার্সন ছাড়া breaking                     ✅ সবসময় version bump করুন
   change deploy করা                         breaking change-এর আগে

❌ পুরনো version হঠাৎ                       ✅ Sunset header দিয়ে ধীরে
   বন্ধ করে দেওয়া                            ধীরে deprecate করুন

❌ সব version-এ আলাদা                      ✅ Shared service layer +
   business logic                            version-specific response

❌ Version-specific database                ✅ একটি schema, accessor/
   schema                                    transformer দিয়ে serve

❌ URL-এ date-based version                 ✅ Numeric versioning
   /api/2024-01-01/users                     /api/v2/users

❌ Deprecation notice ছাড়া                  ✅ কমপক্ষে ৬ মাস আগে
   version বন্ধ করা                           notice দিন

❌ Client-কে force update                   ✅ Gradual migration
   করতে বাধ্য করা                             support দিন

❌ শুধু latest version                       ✅ সব সমর্থিত version-এর
   test করা                                  test রাখুন
```

### Bangladesh-specific ভুল

```
❌ bKash sandbox ও live-এ আলাদা             ✅ উভয়ে একই version
   version ব্যবহার                            strategy

❌ Payment callback URL-এ                    ✅ Callback URL version-
   version না রাখা                            independent রাখুন

❌ মার্চেন্টদের না জানিয়ে                      ✅ Email + SMS + dashboard
   API পরিবর্তন                               notification দিন
```

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                    API VERSIONING সারসংক্ষেপ                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ১. URI Path (/api/v1/) — সবচেয়ে জনপ্রিয়, Public API-র জন্য    │
│     আদর্শ। bKash, Pathao-র মতো বাংলাদেশি API-তে ব্যবহৃত।       │
│                                                                 │
│  ২. Query Param (?version=2) — সহজ, কিন্তু caching সমস্যা।     │
│     ছোট internal tool-এ ব্যবহার করুন।                           │
│                                                                 │
│  ৩. Header (Accept: vnd.app.v2+json) — REST-compliant,          │
│     microservice-এ জনপ্রিয়।                                     │
│                                                                 │
│  ৪. Content Negotiation — সবচেয়ে নমনীয়, জটিল প্রয়োজনে।        │
│                                                                 │
│  ৫. GraphQL/Feature Flags — ভার্সনিং ছাড়া evolution।            │
│                                                                 │
│  মূল নীতি:                                                       │
│  • Breaking change = নতুন major version                         │
│  • Additive change = বর্তমান version-এ যোগ                      │
│  • Shared service layer, version-specific response               │
│  • ৬ মাস deprecation notice, Sunset header                      │
│  • সব version-এ automated test                                   │
│  • Bangladesh context: payment API-তে extra সতর্কতা              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **মনে রাখবেন:** সবচেয়ে ভালো API সেটাই যেটা version না বাড়িয়ে evolve করতে পারে। কিন্তু যখন breaking change অনিবার্য, তখন সুপরিকল্পিত versioning strategy আপনার consumer-দের বিশ্বাস ধরে রাখবে।
