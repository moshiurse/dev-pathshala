# 💾 Browser Storage — কোথায় কী রাখবেন

## 📌 সংক্ষেপ

Browser-এ ডেটা সংরক্ষণের অনেক option আছে — কিন্তু প্রত্যেকটার আলাদা **purpose, capacity,
persistence, security model**। ভুল choice = security breach বা data loss।

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  ১. Cookies         → server-এ ধাঁধাঁধি যায়, auth-এর জন্য   │
  │  ২. localStorage    → simple key-value, persistent          │
  │  ৩. sessionStorage  → tab-এর জীবন পর্যন্ত                   │
  │  ৪. IndexedDB       → বড়, structured, async, transactions   │
  │  ৫. Cache API       → HTTP response cache (Service Worker)  │
  │  ৬. OPFS            → ফাইল সিস্টেম (Chrome 102+)            │
  │  ৭. WebSQL          → ☠️ deprecated, ব্যবহার করবেন না      │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 📊 Comparison Table

| | Cookie | localStorage | sessionStorage | IndexedDB | Cache API | OPFS |
|--|--------|--------------|----------------|-----------|-----------|------|
| **Capacity** | ~4KB/cookie | ~5-10MB | ~5-10MB | ~50% disk | ~50% disk | ~ disk |
| **Persistence** | Expiry-নির্ভর | Forever | Tab close | Forever | Forever | Forever |
| **Server send** | ✅ auto | ❌ | ❌ | ❌ | ❌ | ❌ |
| **API** | sync | sync | sync | async | async | async |
| **Data type** | string | string | string | structured | Request/Response | binary file |
| **Worker access** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **HttpOnly** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Use case** | auth, CSRF | UI prefs | wizard step | offline DB | SW cache | files |

---

## 🍪 Cookies

### সংক্ষিপ্ত পরিচয়

Cookies প্রথম দিন থেকে web-এ আছে। প্রতিটা HTTP request-এ server-এ auto-যায়। তাই
authentication, session-এর জন্য default।

### Set করা — JS

```javascript
document.cookie = "lang=bn-BD; path=/; max-age=31536000; secure; samesite=lax";
```

### Set করা — Server (Express)

```javascript
res.cookie('session', token, {
  httpOnly: true,      // JS পড়তে পারবে না — XSS protection
  secure: true,        // HTTPS only
  sameSite: 'strict',  // CSRF protection
  maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days
  path: '/',
  domain: '.daraz.com.bd'
});
```

### গুরুত্বপূর্ণ flags

```
  HttpOnly    →  document.cookie দিয়ে read করা যায় না (XSS shield)
  Secure      →  HTTPS-এ only পাঠানো হবে
  SameSite    →  CSRF protection
                Strict — শুধু একই site থেকে request-এ যায়
                Lax    — top-level GET-এ যায় (default modern)
                None   — cross-site যায়, কিন্তু Secure লাগবে
  Domain      →  subdomain-এ share (.daraz.com.bd)
  Path        →  নির্দিষ্ট path-এ limit
  Max-Age     →  সেকেন্ডে expiry
```

### Cookie Prefixes

```
  __Secure-name=value; Secure        → অবশ্যই HTTPS
  __Host-name=value; Secure; Path=/  → Domain attribute নিষিদ্ধ, শুধু origin
```

### Authentication best practice

```javascript
// ✅ ভালো — server-only HttpOnly cookie
res.cookie('refreshToken', token, {
  httpOnly: true, secure: true, sameSite: 'strict'
});

// ❌ খারাপ — JS access করতে পারে, XSS = পুরো অ্যাকাউন্ট গেছে
localStorage.setItem('jwt', token);
```

> JWT localStorage-এ রাখা সহজ মনে হলেও XSS-এ পুরো credential চুরি হয়। HttpOnly cookie-ই
> সঠিক উত্তর। Daraz, bKash সবাই এটাই করে।

---

## 📦 localStorage

```javascript
// সব value string — JSON.stringify দিয়ে object রাখুন
localStorage.setItem('cart', JSON.stringify({ items: [...] }));
const cart = JSON.parse(localStorage.getItem('cart') || '{}');
localStorage.removeItem('cart');
localStorage.clear();
```

### বৈশিষ্ট্য

- ✅ persistent (manually clear না করলে থাকে)
- ✅ ৫-১০ MB
- ❌ **Synchronous** → বড় data পড়লে main thread block
- ❌ JS থেকে access → XSS = data leak
- ❌ string only

### কী রাখবেন

```
  ✅ UI preferences (theme, language)
  ✅ Last visited page
  ✅ Form draft (auto-save)
  ✅ Non-sensitive flags ("hasSeenOnboarding")

  ❌ Auth token / JWT (XSS risk)
  ❌ PII (নাম, ফোন, NID)
  ❌ Large data (> 100KB block UI)
  ❌ Encrypted blob without key management
```

### Storage event

অন্য tab-এ change-এ notify পেতে:

```javascript
window.addEventListener('storage', (e) => {
  if (e.key === 'cart') {
    updateCartUI(JSON.parse(e.newValue));
  }
});
```

---

## 📋 sessionStorage

API একই localStorage-এর মতো, কিন্তু:

- শুধু **এক tab-এ**
- Tab close = data gone
- Refresh-এ থাকে

### Use case

- Multi-step wizard (checkout step 1→2→3)
- "Continue where you left off" — current session-এ
- Tab-isolated experiment

```javascript
sessionStorage.setItem('checkoutStep', '2');
sessionStorage.setItem('orderDraft', JSON.stringify(order));
```

---

## 🗄️ IndexedDB — Browser-এর Real Database

IndexedDB হলো একটা **transactional, indexed, async** NoSQL key-value store। বড় data,
structured query, offline-first অ্যাপের ভিত্তি।

```
  ┌───────────────────────────────────────────────┐
  │   Database  →  Object Stores  →  Records      │
  │   ────────     ─────────────     ─────────    │
  │   pathao-db                                   │
  │     ├─ orders   (keyPath: 'id')               │
  │     ├─ cart     (keyPath: 'id')               │
  │     └─ sync     (keyPath: 'id')               │
  └───────────────────────────────────────────────┘
```

### Native API (verbose)

```javascript
const req = indexedDB.open('pathao-db', 1);

req.onupgradeneeded = (e) => {
  const db = e.target.result;
  const store = db.createObjectStore('orders', { keyPath: 'id', autoIncrement: true });
  store.createIndex('status', 'status', { unique: false });
  store.createIndex('createdAt', 'createdAt');
};

req.onsuccess = (e) => {
  const db = e.target.result;
  const tx = db.transaction('orders', 'readwrite');
  const store = tx.objectStore('orders');
  store.add({ items: [...], status: 'pending', createdAt: Date.now() });
};
```

### Dexie.js — IndexedDB এর recommended wrapper

```bash
npm install dexie
```

```javascript
import Dexie from 'dexie';

export const db = new Dexie('bkash-offline');
db.version(1).stores({
  // ++id = auto-increment, &phone = unique index, *tags = multi-entry
  transactions: '++id, &ref, status, type, createdAt',
  pendingTx:    '++id, payload, attempts, createdAt',
  contacts:     '&phone, name, lastUsed'
});

// ───── CRUD ─────
await db.transactions.add({
  ref: 'TX001',
  type: 'send',
  amount: 500,
  status: 'pending',
  createdAt: Date.now()
});

const recent = await db.transactions
  .where('createdAt').above(Date.now() - 7 * 86400_000)
  .reverse()
  .limit(20)
  .toArray();

const sent = await db.transactions
  .where('type').equals('send')
  .and(t => t.amount > 1000)
  .toArray();

await db.transactions.update(id, { status: 'completed' });
await db.transactions.delete(id);

// ───── Transaction (atomic) ─────
await db.transaction('rw', db.transactions, db.pendingTx, async () => {
  await db.pendingTx.delete(pendingId);
  await db.transactions.add(completedTx);
});
```

### Schema migration

```javascript
db.version(1).stores({ orders: '++id, status' });
db.version(2).stores({ orders: '++id, status, createdAt' }).upgrade(tx => {
  return tx.table('orders').toCollection().modify(o => {
    o.createdAt = Date.now();
  });
});
```

### Transactions, durability

IndexedDB ACID-ish:
- Atomic → entire tx commit বা rollback
- Concurrency → readwrite tx serialized per object store
- Durability → "relaxed" durability default; chrome flush optimization

---

## 🗂️ Cache API — Service Worker storage

Cache API specifically Request/Response pair store করার জন্য। Service Worker-এর সাথে যায়
(আলাদা bibliotheca-তে: `pwa-service-workers.md`)।

```javascript
const cache = await caches.open('static-v1');
await cache.add('/index.html');
await cache.addAll(['/css/app.css', '/js/app.js']);
const response = await cache.match('/index.html');
await cache.delete('/old.css');
```

Cookies/localStorage মূলত key-value, IndexedDB structured DB, **Cache API HTTP-aware**।

---

## 📁 OPFS (Origin Private File System)

Chrome 102+ — origin-এর জন্য sandboxed file system। বড় file, SQLite-Wasm এর জন্য
গুরুত্বপূর্ণ।

```javascript
const root = await navigator.storage.getDirectory();
const fileHandle = await root.getFileHandle('transactions.db', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(arrayBuffer);
await writable.close();

// Read
const file = await fileHandle.getFile();
const text = await file.text();
```

### SQLite-Wasm + OPFS

```javascript
import sqlite3InitModule from '@sqlite.org/sqlite-wasm';
const sqlite3 = await sqlite3InitModule();

const db = new sqlite3.oo1.OpfsDb('local.db');
db.exec('CREATE TABLE IF NOT EXISTS tx (id INTEGER PRIMARY KEY, ref TEXT, amount REAL)');
db.exec({ sql: 'INSERT INTO tx (ref, amount) VALUES (?, ?)', bind: ['TX001', 500] });
```

বড় relational data offline চালাতে SQLite + OPFS এখন viable।

---

## 🔢 Quotas & Persistence

### Storage estimate

```javascript
const estimate = await navigator.storage.estimate();
console.log(`Used ${estimate.usage} / ${estimate.quota} bytes`);
```

### Persistent storage (won't be evicted)

```javascript
const persistent = await navigator.storage.persist();
if (persistent) {
  console.log('Storage persistent — browser কখনো auto-clear করবে না');
}
```

Default-এ browser memory pressure-এ data evict করতে পারে। `persist()` request করলে
ব্রাউজার ইউজার থেকে অনুমতি চায় (PWA installed হলে auto-grant সাধারণত)।

### Quota limit (browser-specific)

```
  Chrome:   total ~80% of disk; per-origin proportional
  Firefox:  ~50% of disk; per-origin 10%
  Safari:   ~1GB initially, grows on prompt
  iOS:      strict ~200MB-1GB before user prompt
```

---

## 🔒 Security Implications

### XSS (Cross-Site Scripting)

```
  Attacker JS injected → can read:
  ──────────────────────────────────
  ✅ document.cookie (unless HttpOnly)
  ✅ localStorage / sessionStorage
  ✅ IndexedDB
  ❌ HttpOnly cookie

  Therefore:
  ▸ Auth token → HttpOnly cookie
  ▸ Sensitive data → never localStorage
  ▸ আপনার XSS prevention strategy দরকার
```

### CSRF

Cookie auto-send হয় cross-site request-এ। SameSite=Lax/Strict ব্যবহার করুন। বিস্তারিত
`11-security/xss-csrf.md`-এ।

### Encrypted storage

```javascript
// Web Crypto API দিয়ে encrypt
async function encrypt(data, key) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    new TextEncoder().encode(JSON.stringify(data))
  );
  return { iv, ciphertext };
}

const key = await crypto.subtle.generateKey(
  { name: 'AES-GCM', length: 256 },
  true,
  ['encrypt', 'decrypt']
);
```

⚠️ Key কোথায় রাখবেন? Browser-এই থাকলে XSS-এ ও যাবে। সাধারণত key server-derived (PBKDF2
পাসওয়ার্ড থেকে)।

---

## 🇧🇩 কেস স্টাডি — bKash অফলাইন Transaction Queue

ইউজার রাস্তায় চলতে চলতে টাকা পাঠাতে চান, কিন্তু network নেই।

```javascript
// db.js
import Dexie from 'dexie';
export const db = new Dexie('bkash-offline');
db.version(1).stores({
  pendingTx: '++id, status, createdAt, attempts',
  history:   '++id, ref, type, amount, createdAt'
});
```

```javascript
// send.js
async function sendMoney(payload) {
  // ১. Optimistic UI — local-এ "pending" দেখান
  const localId = await db.pendingTx.add({
    ...payload,
    status: 'pending',
    attempts: 0,
    createdAt: Date.now()
  });
  showToast('লেনদেন কিউতে যোগ হয়েছে');

  // ২. Network থাকলে এখনই পাঠান
  if (navigator.onLine) {
    return processPendingTx(localId);
  }

  // ৩. Service Worker-কে background sync register
  const reg = await navigator.serviceWorker.ready;
  if ('sync' in reg) await reg.sync.register('flush-tx');

  return { offline: true, localId };
}

async function processPendingTx(localId) {
  const tx = await db.pendingTx.get(localId);
  if (!tx) return;

  try {
    const res = await fetch('/api/bkash/send', {
      method: 'POST',
      headers: { 'Idempotency-Key': `tx-${localId}` },
      body: JSON.stringify(tx)
    });
    if (!res.ok) throw new Error('Server error');
    const result = await res.json();

    // ৪. Success — pending থেকে history-তে move (atomic)
    await db.transaction('rw', db.pendingTx, db.history, async () => {
      await db.pendingTx.delete(localId);
      await db.history.add({
        ref: result.ref,
        type: 'send',
        amount: tx.amount,
        createdAt: Date.now()
      });
    });
  } catch (err) {
    await db.pendingTx.update(localId, {
      attempts: tx.attempts + 1,
      lastError: err.message
    });
    if (tx.attempts >= 5) {
      await db.pendingTx.update(localId, { status: 'failed' });
    }
  }
}

// ৫. Online হলে auto retry
window.addEventListener('online', async () => {
  const pending = await db.pendingTx.where('status').equals('pending').toArray();
  for (const tx of pending) await processPendingTx(tx.id);
});
```

### Idempotency key — গুরুত্বপূর্ণ

Network unstable-এ একই রিকোয়েস্ট ২ বার গেলে ২ বার টাকা যাবে না — server idempotency-key
দেখে duplicate ignore করবে।

---

## 🔄 Data Sync Patterns

### ১. Last-Write-Wins (LWW)

`updatedAt` timestamp compare — newest জয়। সহজ, কিন্তু lost update problem।

### ২. Operation log (CRDT-style)

প্রতিটা মিউটেশন event হিসেবে log; server log merge করে। Real-time collaboration-এ
(Notion, Figma)।

### ৩. Conflict resolution UI

User-কে দেখান: "এই data offline-এ এক রকম, server-এ আরেক রকম, কোনটা রাখবেন?"

---

## ⚠️ Pitfalls

```
  ❌ JWT localStorage-এ — XSS = অ্যাকাউন্ট চুরি
  ❌ বড় JSON localStorage-এ — sync API, UI freeze
  ❌ IndexedDB transaction long-running — auto-abort
  ❌ document.cookie দিয়ে JSON parse করতে চাওয়া — string parsing painful
  ❌ Storage quota exceeded → unhandled QuotaExceededError
  ❌ Schema migration ভুল — পুরনো user data lost
  ❌ Private browsing mode — IndexedDB/localStorage সীমিত
  ❌ iOS Safari ITP — 7 day eviction non-persistent storage
  ❌ persist() না call করে critical data — evicted হতে পারে
  ❌ samesite=none secure ছাড়া — modern browser reject
  ❌ Encrypted blob-এ key co-located — security theater
```

---

## ✅ Decision Cheatsheet

```
  ════════════════════════════════════════════════════════════════
  কী ডেটা?              →  কোথায় রাখবেন?
  ════════════════════════════════════════════════════════════════
  Auth (JWT/session)    →  HttpOnly Secure SameSite cookie
  CSRF token            →  cookie + header
  UI theme/lang         →  localStorage
  Form auto-save draft  →  localStorage / IndexedDB
  Multi-step wizard     →  sessionStorage
  Cart (offline)        →  IndexedDB + sync
  Order history         →  IndexedDB
  Static asset cache    →  Cache API (Service Worker)
  HTTP response cache   →  Cache API
  Big files (PDF, video)→  OPFS / IndexedDB Blob
  SQLite local DB       →  OPFS + sqlite-wasm
  PII (name, NID)       →  ❌ ক্লায়েন্টে না; server only
  Payment data          →  ❌ ক্লায়েন্টে না; PCI compliance
  ════════════════════════════════════════════════════════════════
```

---

## 📋 সারসংক্ষেপ

```
  Browser Storage Decision Tree
  ════════════════════════════════════════════════
  
  ১. Server-এ যাবে?
     YES → Cookie (HttpOnly, Secure, SameSite)
  
  ২. Sensitive (auth, PII)?
     YES → HttpOnly cookie (never localStorage)
  
  ৩. ছোট key-value (< 1MB)?
     YES → localStorage / sessionStorage
  
  ৪. Structured, indexed, > 5MB?
     YES → IndexedDB (Dexie wrapper)
  
  ৫. HTTP Request/Response cache?
     YES → Cache API (in Service Worker)
  
  ৬. Files / SQLite?
     YES → OPFS
  
  Always:
  ────────
  • encrypt sensitive data (Web Crypto)
  • call navigator.storage.persist() for critical data
  • handle QuotaExceededError gracefully
  • migrate schemas safely
```

Browser storage ভুল choice মানে security breach বা UX disaster। **Auth = HttpOnly cookie,
preferences = localStorage, offline data = IndexedDB** — এই তিনটা rule মনে রাখলে ৯০%
mistake এড়ানো যায়। বাংলাদেশের context-এ unreliable network ও offline-first design
গুরুত্বপূর্ণ — তাই IndexedDB মাস্টার করা ফ্রন্টএন্ড ইঞ্জিনিয়ারের জন্য must-have skill।
