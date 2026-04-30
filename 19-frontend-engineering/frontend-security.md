# 🛡️ Frontend Security — সম্পূর্ণ চিত্র

> এই ফাইলটি **`11-security/xss-csrf.md`-কে extend করে**, duplicate করে না। মূল XSS/CSRF
> ধারণা সেখানে; এখানে আমরা frontend-specific বিস্তৃত threat surface, defense-in-depth ও
> modern browser security primitives নিয়ে আলোচনা করব।

## 📌 Frontend Threat Landscape

```
  ┌──────────────────────────────────────────────────────────────┐
  │           Frontend Attack Surface — ২০২৪                    │
  │                                                              │
  │   📥 INPUT             📤 OUTPUT          📦 SUPPLY CHAIN     │
  │   ─────────            ────────           ──────────────     │
  │   • Form               • DOM XSS          • npm packages     │
  │   • URL params         • Innerhtml        • CDN scripts      │
  │   • postMessage        • dangerouslySet   • Type confusion   │
  │   • Storage            • document.write   • Typosquatting    │
  │                                            • Dependency      │
  │                                              confusion       │
  │   🌐 NETWORK           🪟 IFRAME           👤 SESSION         │
  │   ──────────           ─────────           ──────────         │
  │   • MITM               • Clickjacking     • Cookie theft     │
  │   • CORS bypass        • Tabnabbing       • CSRF             │
  │   • DNS rebind         • Frame busting    • Session fixation │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

---

## 🧨 ১. XSS — beyond basics

(stored, reflected, DOM-based-এর basic আছে `11-security/xss-csrf.md`-এ)

### DOM XSS sinks (যা attacker exploit করে)

```javascript
// ❌ এসব সব ঝুঁকিপূর্ণ — user input এখানে দেবেন না
element.innerHTML = userInput;
element.outerHTML = userInput;
document.write(userInput);
eval(userInput);
new Function(userInput);
setTimeout(userInput, ...);
setInterval(userInput, ...);
location = userInput;     // javascript:alert()
location.href = userInput;
element.setAttribute('onclick', userInput);
window.open(userInput);   // javascript: scheme
iframe.src = userInput;
```

### Sources (যেখান থেকে input আসে)

```javascript
location.search        // ?q=...
location.hash          // #...
document.referrer
window.name
postMessage event.data
localStorage.getItem()
fetch().then(r => r.json())  // API থেকেও untrusted
URLSearchParams
```

### React-এ DOM XSS এর ফাঁদ

```jsx
// ❌ XSS open
<div dangerouslySetInnerHTML={{ __html: userBio }} />

// ❌ href="javascript:..."
<a href={userProvidedUrl}>Click</a>

// ❌ এই pattern ও বিপজ্জনক
<a href={`mailto:${email}`} />  // email-এ ?cc=evil থাকতে পারে
```

```jsx
// ✅ ভালো
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userBio) }} />

// ✅ URL validate
function safeUrl(url) {
  try {
    const u = new URL(url);
    return ['http:', 'https:', 'mailto:'].includes(u.protocol) ? url : '#';
  } catch { return '#'; }
}
<a href={safeUrl(userProvidedUrl)} />
```

---

## 🛡️ ২. Content Security Policy (CSP) — Deep Dive

CSP হলো একটা HTTP header যা browser-কে বলে কোন source থেকে script/style/image load করা
যাবে। XSS-এর সবচেয়ে শক্তিশালী defense।

### সাধারণ CSP

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.daraz.com.bd;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https://images.daraz.com.bd;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.daraz.com.bd wss://chat.daraz.com.bd;
  frame-src https://payment.bkash.com;
  frame-ancestors 'none';
  form-action 'self';
  upgrade-insecure-requests;
  report-uri https://csp.daraz.com.bd/report
```

### ডিরেক্টিভ ব্যাখ্যা

| Directive | কী restrict করে |
|-----------|-----------------|
| `default-src` | অন্য সব directive-এর fallback |
| `script-src` | JS source |
| `style-src` | CSS source |
| `img-src` | image |
| `font-src` | web font |
| `connect-src` | fetch/XHR/WebSocket |
| `frame-src` | iframe child |
| `frame-ancestors` | কে আপনাকে frame করতে পারবে (clickjacking) |
| `form-action` | form-এর action URL |
| `base-uri` | `<base>` tag |
| `object-src` | plugin (deprecated; `'none'` দিন) |
| `upgrade-insecure-requests` | HTTP→HTTPS auto |

### Nonce-based CSP (recommended)

`'unsafe-inline'` দিলে CSP-র অর্ধেক benefit চলে যায়। Nonce ব্যবহার করুন:

```http
Content-Security-Policy:
  script-src 'self' 'nonce-XYZ123' 'strict-dynamic';
  base-uri 'none';
  object-src 'none';
```

```html
<!-- প্রতিটি page-এ নতুন random nonce server inject করে -->
<script nonce="XYZ123">
  // এই inline script approved
</script>
<script nonce="XYZ123" src="/app.js"></script>
```

`'strict-dynamic'` মানে: nonce-approved script যা যা load করবে সেগুলোও trusted।
Whitelist maintain করতে হবে না।

### Hash-based

```http
script-src 'sha256-AbCdEfGh...';
```

### Report-only mode (rollout-এর জন্য)

```http
Content-Security-Policy-Report-Only: ...
```

ভঙ্গ করলে block করবে না, শুধু `report-uri`-তে রিপোর্ট পাঠাবে — production-এ deploy
করার আগে এই mode-এ test করুন।

### Express middleware

```javascript
import helmet from 'helmet';
app.use(helmet.contentSecurityPolicy({
  useDefaults: true,
  directives: {
    "script-src": ["'self'", (req, res) => `'nonce-${res.locals.cspNonce}'`],
    "frame-ancestors": ["'none'"]
  }
}));
```

---

## 🔒 ৩. Trusted Types — XSS-কে শেকলবন্দী করা

Chrome 83+ — DOM injection-এ raw string allow না, শুধু "trusted" wrapper allow।

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types default daraz-policy;
```

```javascript
const policy = trustedTypes.createPolicy('daraz-policy', {
  createHTML: (input) => DOMPurify.sanitize(input),
  createScriptURL: (url) => {
    const u = new URL(url);
    if (u.origin === 'https://cdn.daraz.com.bd') return url;
    throw new Error('Untrusted URL');
  }
});

// এখন থেকে innerHTML-এ string লিখলে error
element.innerHTML = userInput;          // ❌ throws
element.innerHTML = policy.createHTML(userInput);  // ✅
```

ফলে দুর্ঘটনাবশত কেউ unsafe sink ব্যবহার করলেই runtime error — test-এ ধরা পড়বে।

---

## 🎣 ৪. Clickjacking

Attacker-এর সাইটে আপনার অ্যাপ invisible iframe-এ load করে — ইউজার মনে করে বাটন ক্লিক
করছে, আসলে আপনার অ্যাপের critical action ক্লিক হচ্ছে।

### Defense

```http
X-Frame-Options: DENY                    # legacy header
Content-Security-Policy: frame-ancestors 'none';   # modern
```

```http
X-Frame-Options: SAMEORIGIN              # শুধু নিজের সাইট frame করতে পারবে
```

### Frame-busting (legacy)

```javascript
// ❌ পুরনো method, modern attack-এ কাজ করে না
if (top !== self) top.location = self.location;
```

CSP `frame-ancestors` সবসময় preferred।

### iframe sandbox

আপনি যদি 3rd party content (payment widget, ad) embed করেন:

```html
<iframe
  src="https://payment.bkash.com/widget"
  sandbox="allow-scripts allow-forms allow-same-origin"
  referrerpolicy="strict-origin"
></iframe>
```

Sandbox token:
```
  allow-scripts        → JS চলবে
  allow-forms          → form submit
  allow-popups         → window.open
  allow-same-origin    → cookies, storage access
  allow-top-navigation → top frame change (dangerous)
  allow-modals         → alert/confirm
```

কম দেওয়া = বেশি secure।

---

## 🪟 ৫. Tabnabbing

`window.open` ও `<a target="_blank">` দিয়ে opened page parent-কে navigate করতে পারে
(`window.opener.location = "phishing.com"`)।

```html
<!-- ❌ -->
<a href="https://external.com" target="_blank">Visit</a>

<!-- ✅ -->
<a href="https://external.com" target="_blank" rel="noopener noreferrer">Visit</a>
```

`noopener` — opener access বন্ধ করে (modern browser-এ এখন default)।

---

## 🍪 ৬. Cookie & Session Security

(`browser-storage.md`-এও আছে — এখানে security focus)

### Recipe — secure session cookie

```javascript
res.cookie('session', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',  // CSRF
  path: '/',
  maxAge: 7 * 86400_000,
  domain: '.daraz.com.bd'
});
```

### Cookie prefixes

```
  __Secure-token=...; Secure
  __Host-csrf=...; Secure; Path=/  (Domain attribute নিষেধ — host-locked)
```

### Session fixation

Login-এর পর session ID **regenerate** করুন — না হলে attacker pre-set ID দিয়ে hijack
করতে পারে।

```javascript
req.session.regenerate(err => {
  req.session.userId = user.id;
  res.redirect('/dashboard');
});
```

---

## 📦 ৭. Supply Chain Security

modern frontend = ৫০০+ npm dependencies। এক compromised package = পুরো অ্যাপ
compromised।

### Threats

```
  ১. Typosquatting
     npm install reactt  → malicious "reactt" package install
  
  ২. Dependency Confusion
     internal "@daraz/utils" — public registry-এ একই নাম published
     misconfigured registry → public version install
  
  ৩. Account Takeover
     popular maintainer-এর account হাইজ্যাক → malicious update
     উদাহরণ: event-stream (2018), node-ipc (2022)
  
  ৪. Post-install scripts
     npm install দিলে arbitrary script run
     ─ npm install --ignore-scripts
  
  ৫. Build-time attack
     malicious package build process থেকেই তথ্য পাঠায়
```

### Defenses

```bash
# ১. Lockfile commit করুন
git commit package-lock.json

# ২. Audit
npm audit
npm audit fix

# ৩. Pin versions (or use ranges with caution)
"react": "18.2.0"  // exact

# ৪. Disable post-install scripts
npm install --ignore-scripts
# বা package.json
"scripts": { "preinstall": "npx only-allow pnpm" }

# ৫. Provenance verify (npm 9+)
npm install --foreground-scripts=false

# ৬. SCA tools
# - Snyk
# - Socket.dev  
# - GitHub Dependabot
# - Renovate
```

### .npmrc — internal package safety

```
# .npmrc
@daraz:registry=https://npm.daraz.com.bd
//npm.daraz.com.bd/:_authToken=${NPM_TOKEN}
```

`@daraz/*` সব package internal registry থেকে আসবে — public-এ থাকলেও না।

### Lockfile lint

```bash
npx lockfile-lint --path package-lock.json --validate-https \
  --allowed-hosts npm registry.npmjs.org
```

---

## 🗺️ ৮. Source Map Exposure

Production-এ source map deploy হলে অ্যাটাকার আপনার unminified code পেয়ে যাবে — internal
API endpoint, business logic, comment সব।

```bash
# webpack
devtool: process.env.NODE_ENV === 'production' ? 'hidden-source-map' : 'eval-cheap-source-map'

# vite
build: {
  sourcemap: false,  // production
  // OR upload to Sentry, don't serve publicly
}
```

`hidden-source-map` — source map তৈরি হয় কিন্তু `//# sourceMappingURL` reference থাকে না।
Sentry-এ upload করতে পারেন error tracking-এর জন্য।

---

## 🔗 ৯. Subresource Integrity (SRI)

External CDN script verify করার method — script-এর hash mismatch হলে browser load করবে
না।

```html
<script
  src="https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js"
  integrity="sha384-tsQFqpEReu7ZLhBV2VZlAu7zcOV+rXbYlF2cqB8txI/8aZajjp4Bqd+V6D5IgvKT"
  crossorigin="anonymous"
></script>
```

CDN compromised হলেও hash mismatch → browser block।

CSP-র সাথে combine: `require-sri-for script style;` (deprecated, কিন্তু concept matter)।

---

## 🛂 ১০. Modern Security Headers

```http
# Server response headers — recommended baseline
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: ... (above)
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(self), payment=(self)
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
```

### Permissions-Policy (formerly Feature-Policy)

কোন powerful API allow:

```http
Permissions-Policy:
  camera=(self "https://video-call.daraz.com.bd"),
  microphone=(),
  geolocation=(self),
  payment=(self),
  usb=()
```

### COOP / COEP — process isolation

SharedArrayBuffer, high-resolution timer ব্যবহার করতে হলে দরকার:

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

ফলে আপনার অ্যাপ alone process-এ চলে — Spectre side-channel mitigated।

### Referrer-Policy

কী referrer leak হবে control:

```
  no-referrer                    → কিছুই না
  strict-origin                  → শুধু origin, HTTPS-এ HTTPS-এ
  strict-origin-when-cross-origin → same origin: full URL; cross: origin only
  no-referrer-when-downgrade     → default; HTTP-এ পাঠাবে না
```

---

## 🐛 ১১. Prototype Pollution

```javascript
// ❌ vulnerable merge function
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === 'object') {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
}

// attacker payload
merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'));

// এখন:
const u = {};
console.log(u.isAdmin);  // true!  সব Object instance polluted
```

### Defense

```javascript
// ১. __proto__, constructor, prototype keys reject
function safeMerge(target, source) {
  for (const key in source) {
    if (['__proto__', 'constructor', 'prototype'].includes(key)) continue;
    // ...
  }
}

// ২. Object.create(null) — prototype-less object
const map = Object.create(null);

// ৩. Library: lodash _.merge() ২০১৯-এ patched, কিন্তু update থাকতে হবে
// ৪. Map ব্যবহার করুন object-এর বদলে
const cache = new Map();
```

JSON-এ `__proto__` শুধু own property হিসেবে create — তাই সাবধান parse-merge logic-এ।

---

## 🔐 ১২. CORS Misconceptions

```
  ❌ "CORS is security"
  ✅ CORS is a relaxation of Same-Origin Policy
  
  CORS attacker থেকে browser-কে রক্ষা করে না।
  Server response header দিয়ে browser-কে বলে "এই origin-কে read access দাও"।
  
  Cross-origin attacker সরাসরি curl দিয়ে আপনার API call করতে পারবে।
  CORS শুধু victim-এর browser থেকে evil.com-এর JS আপনার API read করতে দেয় না।
```

### সঠিক CORS

```javascript
app.use(cors({
  origin: ['https://daraz.com.bd', 'https://m.daraz.com.bd'],
  credentials: true,    // cookie-সহ
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));

// ❌ ভুল
app.use(cors({ origin: '*', credentials: true }));  // browser reject করবে
```

`Access-Control-Allow-Origin: *` ও `credentials: true` একসাথে যায় না।

---

## 🧪 ১৩. Security Testing

```
  ১. Static — ESLint plugins
     - eslint-plugin-security
     - eslint-plugin-no-unsanitized
     - eslint-plugin-react-security
  
  ২. Dependency — Snyk, Dependabot, npm audit
  
  ৩. Dynamic
     - OWASP ZAP
     - Burp Suite
     - browser DevTools Security panel
  
  ৪. CSP testing
     - Report-only mode in staging
     - csp-evaluator.withgoogle.com
  
  ৫. Penetration test (production-grade)
```

---

## 🇧🇩 কেস স্টাডি — bKash Web Security Stack

```
  Layer 1: Network
  ─────────────────
  ✅ HSTS preload
  ✅ TLS 1.3, weak cipher disabled
  ✅ DNSSEC

  Layer 2: HTTP Headers
  ──────────────────────
  ✅ Strict CSP with nonce
  ✅ Trusted Types
  ✅ COOP/COEP
  ✅ Permissions-Policy (block camera, USB except where needed)
  ✅ X-Content-Type-Options: nosniff

  Layer 3: Cookies / Auth
  ────────────────────────
  ✅ HttpOnly + Secure + SameSite=Strict
  ✅ __Host- prefix
  ✅ Session regeneration on login
  ✅ Short refresh token TTL

  Layer 4: Code
  ──────────────
  ✅ DOMPurify wrap dangerouslySetInnerHTML
  ✅ React + Trusted Types policy
  ✅ ESLint security rules
  ✅ pre-commit hook: secret scan

  Layer 5: Supply chain
  ──────────────────────
  ✅ Internal npm registry
  ✅ Lockfile committed + lint
  ✅ Renovate auto-update
  ✅ ignore-scripts default

  Layer 6: Monitoring
  ────────────────────
  ✅ CSP reports → SIEM
  ✅ Sentry source maps (private)
  ✅ Anomaly detection
```

এতগুলো layer থাকার পরেও ১০০% নিরাপদ না — defense in depth-এ "every layer matters"।

---

## ⚠️ Common Mistakes

```
  ❌ JWT in localStorage          → use HttpOnly cookie
  ❌ "trust the API response"     → API থেকেও XSS আসতে পারে
  ❌ CSP-এ 'unsafe-inline'        → nonce/hash use করুন
  ❌ target="_blank" no rel        → tabnabbing
  ❌ Source map prod-এ exposed     → hidden-source-map
  ❌ npm install -g random pkg     → audit before install
  ❌ Inline event handler (onclick)→ CSP ভাঙে
  ❌ document.write প্রোডাকশনে    → never use
  ❌ Trust user-supplied URL       → protocol validate
  ❌ window.postMessage origin check skip → cross-origin XSS
  ❌ "API key in JS"               → server-side only
  ❌ Session cookie HTTPS ছাড়া    → MITM
```

---

## ✅ Frontend Security Checklist

```
  □ HTTPS everywhere + HSTS preload
  □ Strict CSP with nonce + Trusted Types
  □ HttpOnly Secure SameSite cookies
  □ X-Frame-Options / frame-ancestors 'none'
  □ Permissions-Policy minimal
  □ COOP/COEP if SAB needed
  □ DOMPurify on any HTML render
  □ rel="noopener noreferrer" on external links
  □ SRI for external scripts
  □ Source maps not publicly exposed
  □ npm audit clean, lockfile committed
  □ Internal scoped packages
  □ Pre-commit secret scanning
  □ Regular pen test
  □ Bug bounty program (mature)
  □ CSP report endpoint monitored
```

---

## 📋 সারসংক্ষেপ

Frontend security = **defense in depth**। কোনো একটা defense ১০০% guaranteed না — তাই
layered approach চাই:

1. **Network layer** — HTTPS, HSTS
2. **Header layer** — CSP, Trusted Types, COOP/COEP, Permissions-Policy
3. **Cookie/Auth layer** — HttpOnly, SameSite, prefixes
4. **Code layer** — input validation, output encoding, DOMPurify
5. **Dependency layer** — audit, lockfile, internal registry
6. **Runtime layer** — CSP reports, error monitoring
7. **Process layer** — code review, pen test, bug bounty

বাংলাদেশের ফিনটেক (bKash, Nagad, Rocket) থেকে শুরু করে e-commerce (Daraz, Chaldal) — সব
জায়গায় একটাই XSS = বড় ক্ষতি (টাকা চুরি, account takeover)। Modern browser-এ CSP, Trusted
Types, ও SameSite cookie ৯০% common attack বন্ধ করে দেয় — কিন্তু সেগুলো **ঠিকঠাক
configure করতে হবে**। Default secure এখনো নয়।
