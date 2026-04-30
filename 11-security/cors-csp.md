# 🛡️ CORS ও CSP — Cross-Origin Resource Sharing ও Content Security Policy

## 📖 ভূমিকা

Browser-এ দুটি বড় security mechanism — **CORS** এবং **CSP** — দুটোই HTTP header-ভিত্তিক, কিন্তু দুটো ভিন্ন সমস্যার সমাধান করে। অনেকে এদের গুলিয়ে ফেলে।

| বিষয় | CORS | CSP |
|------|------|-----|
| উদ্দেশ্য | Same-Origin Policy এর exception দেওয়া | Browser-এ unsafe content load/execute রোধ |
| কে enforce করে | Browser | Browser |
| Direction | Server → Browser যেতে দিচ্ছে কিনা | Browser কী load করতে পারবে তার allowlist |
| Mitigates | Cross-site data leak | XSS, clickjacking, data exfiltration |
| Header | `Access-Control-Allow-*` | `Content-Security-Policy` |

---

# Part 1: CORS

## 📖 Same-Origin Policy (SOP)

ব্রাউজারের core security model — **একটি origin-এর script অন্য origin-এর resource (response body) পড়তে পারে না**। Origin = `protocol + host + port` এর tuple।

```
https://daraz.com.bd/page.html  ←─ same origin
https://daraz.com.bd/api/data   ←─ same origin

http://daraz.com.bd     ← different (protocol)
https://m.daraz.com.bd  ← different (subdomain)
https://daraz.com.bd:8080 ← different (port)
```

**SOP কী রক্ষা করে?**
- আপনি bKash-এ logged in, attacker.com সেই tab-এ JS দিয়ে bKash-এর `/balance` API call করে balance পড়তে পারবে না।
- কিন্তু `<img>`, `<script>`, `<form>` সাধারণ resource (read করা ছাড়া) load করতে পারে — এজন্য CSRF-এর প্রয়োজন।

## 🌐 CORS — কখন প্রয়োজন

আপনার frontend `https://app.daraz.com.bd` থেকে API `https://api.daraz.com.bd`-এ AJAX call করতে চান — দুটো origin আলাদা। CORS এর header না থাকলে browser response block করবে।

### Simple Request (no preflight)

Browser preflight skip করে যদি **সব** শর্ত মেনে চলা হয়:
- Method: `GET`, `HEAD`, `POST`
- Header শুধু: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`
- `Content-Type` শুধু: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`
- No `ReadableStream` upload, no event listener on `XMLHttpRequest.upload`

```
Browser ────► Server
GET /products HTTP/1.1
Origin: https://app.daraz.com.bd

Server ────► Browser
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.daraz.com.bd
Content-Type: application/json
{...}
```

### Preflight Request (OPTIONS)

`PUT`, `DELETE`, custom header (`Authorization`, `X-API-Key`), বা `Content-Type: application/json` দিলে browser আগে `OPTIONS` পাঠায়।

```
Browser ────► Server (preflight)
OPTIONS /orders HTTP/1.1
Origin: https://app.daraz.com.bd
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

Server ────► Browser
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.daraz.com.bd
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400              ← cache preflight 24h

Browser ────► Server (actual request)
POST /orders HTTP/1.1
Origin: https://app.daraz.com.bd
Authorization: Bearer eyJ...
Content-Type: application/json
{...}
```

### Credentialed Request

Cookie/Auth header পাঠাতে হলে:

```javascript
fetch('https://api.daraz.com.bd/profile', {
  credentials: 'include',  // send cookies
});
```

Server-এ MUST থাকবে:
```
Access-Control-Allow-Origin: https://app.daraz.com.bd  ← exact, NO wildcard
Access-Control-Allow-Credentials: true
```

⚠️ **Wildcard (`*`) credentialed request-এ illegal।** Browser block করবে।

### Response Headers Summary

| Header | কাজ |
|--------|-----|
| `Access-Control-Allow-Origin` | কোন origin allow (specific URL or `*`) |
| `Access-Control-Allow-Methods` | কোন method (preflight response) |
| `Access-Control-Allow-Headers` | কোন request header allow |
| `Access-Control-Allow-Credentials` | Cookie/auth allowed (`true`) |
| `Access-Control-Expose-Headers` | কোন response header JS পড়তে পারবে (e.g., `X-Total-Count`) |
| `Access-Control-Max-Age` | Preflight cache সেকেন্ড |

---

## 💻 PHP (Laravel) CORS

```php
// config/cors.php (Laravel 9+)
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],

    // ✅ Exact origins for credentialed
    'allowed_origins' => [
        'https://app.daraz.com.bd',
        'https://m.daraz.com.bd',
    ],

    // For dev — pattern
    'allowed_origins_patterns' => ['/^https:\/\/.*\.daraz\.com\.bd$/'],

    'allowed_headers' => ['Authorization', 'Content-Type', 'X-Requested-With', 'X-CSRF-TOKEN'],
    'exposed_headers' => ['X-Total-Count', 'X-Request-Id'],
    'max_age' => 86400,
    'supports_credentials' => true,
];
```

### Manual middleware

```php
namespace App\Http\Middleware;
use Closure;

class CorsMiddleware
{
    private array $allowedOrigins = [
        'https://app.daraz.com.bd',
        'https://m.daraz.com.bd',
    ];

    public function handle($request, Closure $next)
    {
        $origin = $request->headers->get('Origin');

        // ⚠️ Echo origin only if in allowlist (avoid reflective wildcard)
        if (!in_array($origin, $this->allowedOrigins, true)) {
            // Origin not allowed — don't add CORS headers
            if ($request->isMethod('OPTIONS')) {
                return response('', 403);
            }
            return $next($request);
        }

        if ($request->isMethod('OPTIONS')) {
            return response('', 204)
                ->header('Access-Control-Allow-Origin', $origin)
                ->header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')
                ->header('Access-Control-Allow-Headers', 'Authorization, Content-Type, X-CSRF-TOKEN')
                ->header('Access-Control-Allow-Credentials', 'true')
                ->header('Access-Control-Max-Age', '86400')
                ->header('Vary', 'Origin');                // 🔥 critical for caching
        }

        $response = $next($request);
        $response->headers->set('Access-Control-Allow-Origin', $origin);
        $response->headers->set('Access-Control-Allow-Credentials', 'true');
        $response->headers->set('Vary', 'Origin');
        return $response;
    }
}
```

---

## 💻 Node.js (Express) CORS

```javascript
import express from 'express';
import cors from 'cors';

const app = express();

const allowedOrigins = [
  'https://app.daraz.com.bd',
  'https://m.daraz.com.bd',
];

app.use(cors({
  origin: (origin, callback) => {
    // Allow no-origin (mobile apps, curl)
    if (!origin) return callback(null, true);
    if (allowedOrigins.includes(origin)) return callback(null, true);
    callback(new Error('CORS: origin not allowed'));
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Authorization', 'Content-Type', 'X-CSRF-Token'],
  exposedHeaders: ['X-Total-Count'],
  maxAge: 86400,
}));
```

---

## ⚠️ CORS Misconfigurations & Bypasses

### 1. Reflective `Access-Control-Allow-Origin`
```php
// ❌ DANGEROUS
header('Access-Control-Allow-Origin: ' . $_SERVER['HTTP_ORIGIN']);
header('Access-Control-Allow-Credentials: true');
```
যেকোনো site (attacker.com) request পাঠালেই allow। কখনই reflect করার আগে allowlist check করুন।

### 2. Null origin
`Origin: null` (sandboxed iframe, file://, redirect chain)। কেউ allowlist-এ `null` রাখলে attacker iframe থেকে exploit করতে পারে।

### 3. Subdomain takeover
`*.daraz.com.bd` allow করেছেন কিন্তু `abandoned.daraz.com.bd` DNS-এ orphan → attacker register করে exploit।

### 4. Wildcard with credentials
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
Browser এটা reject করে, কিন্তু কিছু cache/CDN এ মিশে যায় — confusion।

### 5. `Vary: Origin` ভুলে যাওয়া
CDN/proxy cache-এ এক origin-এর response অন্য origin-কে serve হতে পারে → security/data leak।

### 6. Whitelisting `<your-domain>.attacker.com`
Regex `^.*daraz\.com\.bd` মতো loose pattern attacker-এর `daraz.com.bd.attacker.com` match করে।

### 7. CORS ≠ CSRF protection
CORS request-পাঠানো block করে না (browser request পাঠাবে), শুধু response read block করে। POST form submit-এ CORS-এর ভূমিকা নেই — তাই CSRF token আলাদাভাবে দরকার।

---

# Part 2: Content Security Policy (CSP)

## 📖 ধারণা

**CSP** browser-কে বলে কোন source থেকে script, style, image, frame, font, ইত্যাদি load/execute করা allowed। XSS-এর ultimate defense layer। `Content-Security-Policy` header (or `<meta>` tag, কিন্তু header preferred)।

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.daraz.com.bd; img-src 'self' data: https:;
```

XSS attacker `<script>alert(1)</script>` inject করেও browser CSP-এর কারণে execute করবে না।

## 📜 CSP Directives

| Directive | কী control করে |
|-----------|---------------|
| `default-src` | অন্যান্য `*-src` set না থাকলে fallback |
| `script-src` | JavaScript sources |
| `script-src-elem` | `<script>` tag specifically |
| `script-src-attr` | inline event handlers (`onclick=`) |
| `style-src` | CSS sources |
| `img-src` | `<img>`, CSS `url()` images |
| `font-src` | `@font-face` |
| `connect-src` | `fetch`, `XHR`, `WebSocket`, EventSource, `sendBeacon` |
| `media-src` | `<audio>`, `<video>` |
| `frame-src` | `<iframe>` |
| `frame-ancestors` | যারা embed করতে পারবে (X-Frame-Options এর replacement) |
| `form-action` | `<form action=...>` |
| `object-src` | `<object>`, `<embed>` (Flash etc.) — set `'none'` |
| `base-uri` | `<base href>` |
| `worker-src` | Web Workers, Service Workers |
| `manifest-src` | PWA manifest |
| `report-uri` / `report-to` | Violation report endpoint |
| `upgrade-insecure-requests` | http → https auto upgrade |
| `block-all-mixed-content` | Mixed content block |
| `require-trusted-types-for 'script'` | Trusted Types enforce |
| `trusted-types` | Trusted Types policy names |

## 🎯 Source Values

| Value | অর্থ |
|-------|------|
| `'self'` | Same origin |
| `'none'` | কিছুই allow না |
| `'unsafe-inline'` | Inline script/style allow (⚠️ XSS risk) |
| `'unsafe-eval'` | `eval()`, `new Function()` allow |
| `'strict-dynamic'` | Loaded script-এর descendants trust |
| `'nonce-XXX'` | Specific nonce-যুক্ত inline allow |
| `'sha256-XXX'` | Specific hash-এর inline allow |
| `https:` | যেকোনো HTTPS source |
| `data:` | data: URI (img-এ ব্যবহার, script-এ never) |
| `*.daraz.com.bd` | wildcard subdomain |

---

## 🔐 Nonce-based CSP (recommended)

প্রতিটি page load-এ unique random nonce generate করুন; `<script>` tag-এ `nonce` attribute এবং header-এ `'nonce-XXX'` মিলালে script run করবে।

### PHP

```php
// middleware
$nonce = base64_encode(random_bytes(16));
$request->attributes->set('csp_nonce', $nonce);

$csp = "default-src 'self'; "
     . "script-src 'self' 'nonce-{$nonce}' 'strict-dynamic' https:; "
     . "style-src 'self' 'nonce-{$nonce}'; "
     . "img-src 'self' data: https://cdn.daraz.com.bd; "
     . "connect-src 'self' https://api.daraz.com.bd https://www.google-analytics.com; "
     . "font-src 'self' https://fonts.gstatic.com; "
     . "frame-ancestors 'none'; "
     . "base-uri 'self'; "
     . "object-src 'none'; "
     . "form-action 'self'; "
     . "upgrade-insecure-requests; "
     . "report-to csp-endpoint;";

$response->headers->set('Content-Security-Policy', $csp);
$response->headers->set('Reporting-Endpoints', 'csp-endpoint="https://daraz.com.bd/csp-report"');
```

```blade
{{-- Blade template --}}
<script nonce="{{ request()->attributes->get('csp_nonce') }}">
  window.appConfig = @json($config);
</script>
```

### Node.js (Express + helmet)

```javascript
import helmet from 'helmet';
import crypto from 'node:crypto';

app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
  next();
});

app.use(helmet.contentSecurityPolicy({
  useDefaults: false,
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.cspNonce}'`, "'strict-dynamic'", 'https:'],
    styleSrc:  ["'self'", (req, res) => `'nonce-${res.locals.cspNonce}'`],
    imgSrc:    ["'self'", 'data:', 'https://cdn.daraz.com.bd'],
    connectSrc:["'self'", 'https://api.daraz.com.bd', 'wss://socket.daraz.com.bd'],
    fontSrc:   ["'self'", 'https://fonts.gstatic.com'],
    objectSrc: ["'none'"],
    frameAncestors: ["'none'"],
    baseUri:   ["'self'"],
    formAction:["'self'"],
    upgradeInsecureRequests: [],
    reportTo:  ['csp-endpoint'],
  },
}));
```

---

## 🌟 `'strict-dynamic'`

পুরাতন CSP-এ allowlist (`https://cdn.example.com`) maintain করা কষ্টকর — bypass-prone। `strict-dynamic` দিলে nonced/hashed script-এর descendants automatically trusted; allowlist ignored।

```
script-src 'nonce-RANDOM' 'strict-dynamic' https: 'unsafe-inline';
        ↑              ↑                    ↑       ↑
   primary trust  trust descendants      old      old
                                       browser   browser
                                       fallback  fallback
```

Modern browser `'strict-dynamic'` দেখলে `https:` ও `'unsafe-inline'` ignore করে — backward compat-এর জন্যই রাখা।

---

## 🔒 Hash-based CSP

Nonce-এর বদলে inline content-এর SHA256 hash:

```html
<script>console.log('hello')</script>
```

Hash:
```
echo -n "console.log('hello')" | openssl dgst -sha256 -binary | openssl base64
→ Pj/L+1QO1wNmeI6ZSt2wmwS1xvQk/5Rl6CnWXaIPBWk=
```

Header:
```
Content-Security-Policy: script-src 'sha256-Pj/L+1QO1wNmeI6ZSt2wmwS1xvQk/5Rl6CnWXaIPBWk='
```

Static inline (analytics snippet) থাকলে practical।

---

## 📨 Reporting (report-uri / report-to)

CSP violation হলে browser report POST করে।

```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

`Report-Only` mode-এ block হয় না, শুধু report — production-এ deploy করার আগে monitoring।

```javascript
// Node.js endpoint
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  console.log('CSP violation:', req.body);
  // ship to Sentry/Datadog
  res.status(204).end();
});
```

Modern `Reporting-API`:
```
Reporting-Endpoints: csp-endpoint="https://daraz.com.bd/csp-report"
Content-Security-Policy: ...; report-to csp-endpoint;
```

---

## 🔐 Trusted Types (DOM XSS এর শেষ defense)

Browser-কে বলুন যে `innerHTML`, `eval()`, `setTimeout(string)` ইত্যাদি sink-এ raw string accept করবে না — শুধু `TrustedHTML`/`TrustedScript` object।

```
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types daraz-policy default;
```

```javascript
// Define policy
const policy = trustedTypes.createPolicy('daraz-policy', {
  createHTML: (input) => DOMPurify.sanitize(input),
});

// Now innerHTML must use policy
element.innerHTML = policy.createHTML(userInput);  // ✅
element.innerHTML = userInput;                      // ❌ TypeError
```

---

## 🎯 Real-world CSP Migration Strategy

```
Step 1: Audit (Report-Only)
   Content-Security-Policy-Report-Only: default-src 'self'; ...; report-to csp;
   ↓ collect 1-2 weeks of violations
Step 2: Refactor inline scripts
   - move to external files
   - or apply nonce/hash
Step 3: Tighten allowlist
   - remove unused CDN
   - replace 'unsafe-inline' with nonce
Step 4: Enforce
   Content-Security-Policy: ...; (no Report-Only)
Step 5: Add Trusted Types
   require-trusted-types-for 'script'
```

---

## 🇧🇩 Bangladesh Real-world Examples

- **Daraz BD:** Strict CSP, `nonce` per page, third-party (Google Tag Manager, Facebook Pixel) explicitly allowed। Festival sale-এ inline script auto-generated nonce দিয়ে।
- **bKash web portal:** `frame-ancestors 'none'` clickjacking রোধ; merchant payment iframe-এ specific origin allow।
- **Pathao web:** `connect-src` Google Maps API ও internal API restrict; XSS report Datadog-এ।
- **Chaldal:** Image CDN (`cdn.chaldal.com`) সহ tight CSP; user review section-এ DOMPurify + Trusted Types।
- **Prothom Alo:** Comment section-এ XSS prone; Report-Only mode থেকে enforce-এ migration।
- **SSLCommerz / ShurjoPay:** Payment gateway iframe-এ strict `frame-ancestors`-merchant whitelist।
- **Bangladesh Bank** (BACPS portal): conservative CSP, কোনো third-party script নেই, `default-src 'self'`।

---

## ⚠️ Common Misconfigurations & Bypasses

### CSP bypasses
1. **`'unsafe-inline'` রাখা** — CSP কার্যত অকার্যকর XSS-এর বিরুদ্ধে।
2. **JSONP endpoint allowlist** — `https://cdn.example.com` allow করেছেন; cdn-এ JSONP endpoint থাকলে attacker callback name দিয়ে arbitrary JS execute।
3. **AngularJS/Old framework on allowlist** — Angular sandbox bypass dating CSP allowlist-এ থাকলে।
4. **`base-uri` ভুলে যাওয়া** — attacker `<base href>` inject করে relative URL hijack।
5. **`'unsafe-eval'`** — JS template engine (Vue inline template, Handlebars eval) require করে; safe alternative use করুন।
6. **Wildcard `*` script-src** — সমস্ত source allow → CSP useless।
7. **No `object-src 'none'`** — Flash file দিয়ে XSS bypass (পুরাতন)।
8. **`script-src 'self'` + user upload** — user `.js` upload করতে পারলে same origin, CSP allow করবে।

### CORS misconfigurations recap
- Reflective ACAO with credentials
- Null origin allow
- Subdomain wildcard sans verification
- Forgot `Vary: Origin`
- Loose regex matching attacker domain

---

## ✅ Best Practices Checklist

**CORS:**
- [ ] Exact origin allowlist; never reflect blindly
- [ ] `Vary: Origin` always
- [ ] `credentials: true` শুধু necessary endpoints-এ
- [ ] Preflight cache `Access-Control-Max-Age` set
- [ ] CORS ≠ CSRF; token আলাদা

**CSP:**
- [ ] `default-src 'self'`; tight per-directive
- [ ] No `'unsafe-inline'`, no `'unsafe-eval'`
- [ ] Nonce per request + `'strict-dynamic'`
- [ ] `object-src 'none'`, `base-uri 'self'`
- [ ] `frame-ancestors` set (X-Frame-Options replacement)
- [ ] `upgrade-insecure-requests` for HTTPS migration
- [ ] Report-Only mode-এ shake out violations first
- [ ] Trusted Types চালু DOM XSS-এর জন্য
- [ ] CSP-report monitoring (Sentry/Datadog)

---

## 📝 সারসংক্ষেপ

- **CORS** = Same-Origin Policy এর exception মেকানিজম। Browser response read করতে দিবে কিনা decide করে।
- **CSP** = Browser-এ load/execute হওয়া content-এর allowlist। XSS-এর সবচেয়ে শক্তিশালী mitigation।
- দুটো ভিন্ন সমস্যা; দুটোই দরকার।
- CORS-এ wildcard + credentials illegal; reflective origin dangerous।
- CSP-এ `'unsafe-inline'` এড়ান; nonce + `'strict-dynamic'` use করুন।
- Trusted Types DOM XSS-এর finishing layer।
- Report-Only mode দিয়ে production-এ migrate; violations monitor করুন।
- Bangladesh-এ Daraz, bKash, Pathao production-এ এ best practices follow করে; misconfigured site-এ XSS exploit common।
