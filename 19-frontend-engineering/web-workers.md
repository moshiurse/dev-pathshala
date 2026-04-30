# 🧵 Web Workers — ব্রাউজারে multi-threading

## 📌 সংজ্ঞা

JavaScript single-threaded — main thread একটা কাজ করতে গেলে UI freeze হয়ে যায়। **Web
Workers** browser-এ background thread চালানোর API দেয়, যাতে heavy computation main UI
thread block না করে।

```
  Without Workers
  ───────────────
  Main Thread:  [UI] [User click] [⏳ ৫MB CSV parse..............] [UI freeze!]
                              ↑
                              ৫ সেকেন্ড পেজ স্থির, ক্লিক কাজ করে না

  With Workers
  ────────────
  Main Thread:  [UI] [User click] → postMessage → [UI continues] [result render]
  Worker:                            [⏳ CSV parse ৫MB.......]    ↑ postMessage
```

---

## 🧬 ৩ ধরনের Worker

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  ১. Dedicated Worker                                        │
  │     এক page-এর সাথে এক worker — সবচেয়ে কমন                 │
  │     new Worker('./worker.js')                                │
  │                                                              │
  │  ২. Shared Worker                                           │
  │     এক origin-এর একাধিক page একটাই worker share              │
  │     new SharedWorker('./shared.js')                          │
  │     ⚠️ Safari-তে কম support                                   │
  │                                                              │
  │  ৩. Service Worker                                          │
  │     network proxy (PWA chapter দেখুন)                        │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

| Worker | Lifetime | Scope | Use case |
|--------|----------|-------|----------|
| Dedicated | পেজের সাথে | এক page | CSV parse, image filter, crypto |
| Shared | পেজ closed না হওয়া অবধি | এক origin-এর সব tab | shared cache, central WebSocket |
| Service | persistent (আলাদা lifecycle) | scope-based | offline, push, background sync |

---

## 🔧 Dedicated Worker — basic

### main.js

```javascript
// Worker create — module type recommended (modern)
const worker = new Worker(
  new URL('./csv-worker.js', import.meta.url),
  { type: 'module' }
);

worker.postMessage({ type: 'PARSE', csv: largeCsvString });

worker.onmessage = (e) => {
  const { type, payload } = e.data;
  if (type === 'PROGRESS') updateProgress(payload);
  if (type === 'DONE')     renderTable(payload);
  if (type === 'ERROR')    showError(payload);
};

worker.onerror = (err) => console.error('Worker error:', err);
```

### csv-worker.js

```javascript
self.onmessage = async (e) => {
  const { type, csv } = e.data;
  if (type !== 'PARSE') return;

  try {
    const lines = csv.split('\n');
    const total = lines.length;
    const result = [];

    for (let i = 0; i < total; i++) {
      result.push(parseLine(lines[i]));

      // প্রতি ১০০০ row-এ progress পাঠান
      if (i % 1000 === 0) {
        self.postMessage({ type: 'PROGRESS', payload: i / total });
      }
    }

    self.postMessage({ type: 'DONE', payload: result });
  } catch (err) {
    self.postMessage({ type: 'ERROR', payload: err.message });
  }
};

function parseLine(line) {
  const [date, type, amount, ref] = line.split(',');
  return { date, type, amount: parseFloat(amount), ref };
}
```

---

## 🔁 Communication — postMessage & Structured Clone

Main thread ও worker **মেমরি share করে না**। তাই যা পাঠান, browser **structured clone**
করে — অর্থাৎ deep copy।

```
  Main thread          Worker thread
  ─────────────        ──────────────
  obj { ... }   ──[deep clone]──▶  copy of obj { ... }
  
  ⚠️ বড় object copy = expensive
  ⚠️ functions, DOM nodes, Symbol clone হয় না
  ⚠️ class instance prototype হারায়
```

### Structured clone যা পাঠানো যায়

✅ primitives, Array, Object, Map, Set, Date, RegExp, Blob, File, ArrayBuffer, ImageData,
TypedArray
❌ Function, DOM node, Symbol (most), class instance methods

---

## 🚀 Transferable Objects — zero-copy

ArrayBuffer, MessagePort, ImageBitmap, OffscreenCanvas — এগুলো **transfer** করা যায়,
copy না। মূল thread থেকে ownership worker-এ চলে যায় (যেমন memory pointer pass)।

```javascript
const buffer = new ArrayBuffer(50 * 1024 * 1024); // 50MB

// ❌ copy — slow, memory doubled
worker.postMessage({ buffer });

// ✅ transfer — instant, memory ownership transferred
worker.postMessage({ buffer }, [buffer]);

// এখন main thread-এ:
console.log(buffer.byteLength);  // 0 (detached!)
```

```
  Copy (default)                    Transfer
  ──────────────                    ────────
  Main: [50MB buffer] → clone       Main: [50MB] →─────────▶ Worker
  Worker:              [50MB copy]  Main: [empty]  (detached)

  Time: 200ms                       Time: < 1ms
  Memory: 100MB peak                Memory: 50MB
```

বড় ImageData, audio buffer, ML tensor সবসময় transfer করুন।

---

## 🎨 OffscreenCanvas

Main thread ছাড়াই worker-এ canvas render — game, chart, image processing।

```javascript
// main.js
const canvas = document.getElementById('chart');
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker('./chart-worker.js');
worker.postMessage({ canvas: offscreen, data }, [offscreen]);
```

```javascript
// chart-worker.js
self.onmessage = (e) => {
  const { canvas, data } = e.data;
  const ctx = canvas.getContext('2d');
  
  // worker-এ পুরো render হবে — main thread blocked না
  drawComplexChart(ctx, data);
};
```

Pathao live tracking-এ ১০০+ driver pin animate করতে OffscreenCanvas আদর্শ।

---

## 🎯 Real Use Cases

### ১. Heavy computation

- **Sorting/filtering ১০ লাখ row** — Daraz product list
- **PDF generation** — bKash transaction statement
- **Cryptography** — encrypt/decrypt, hash
- **ML inference** — TensorFlow.js, ONNX runtime

### ২. Image / Video processing

- **Image filter** — Instagram-style filter
- **Video encoding** — TikTok-clone
- **Resize before upload** — Daraz seller product image

### ৩. Parsing / Data transformation

- **CSV / Excel** — bKash transaction export
- **JSON parsing** (huge) — analytics dashboard
- **Compression** — gzip before upload

### ৪. Real-time

- **WebSocket message processing** — chat, live tracking
- **Audio analysis** — voice recording app

---

## 🇧🇩 কেস স্টাডি — bKash transaction CSV

ইউজার ১ বছরের ৫০,০০০ transaction CSV download করে frontend-এ analyze করছে।

```
  ৫০,০০০ row × ৬ column ≈ ৫MB CSV
  
  Without worker:
  ─────────────
  • parseCSV() → main thread ৪-৬ সেকেন্ড freeze
  • UI dead — scroll না, click না
  • "Page Unresponsive" warning

  With worker:
  ────────────
  • Main thread free
  • UI smooth, progress bar render
  • Filter/sort instant feel
```

### main.js (React)

```javascript
import { useState, useRef, useEffect } from 'react';

function TransactionAnalyzer() {
  const [progress, setProgress] = useState(0);
  const [transactions, setTransactions] = useState([]);
  const workerRef = useRef();

  useEffect(() => {
    workerRef.current = new Worker(
      new URL('./tx-worker.js', import.meta.url),
      { type: 'module' }
    );
    workerRef.current.onmessage = (e) => {
      const { type, payload } = e.data;
      if (type === 'PROGRESS') setProgress(payload);
      if (type === 'DONE')     setTransactions(payload);
    };
    return () => workerRef.current.terminate();
  }, []);

  const handleFile = async (file) => {
    const buffer = await file.arrayBuffer();
    // transferable — zero copy
    workerRef.current.postMessage({ type: 'PARSE', buffer }, [buffer]);
  };

  return (
    <>
      <input type="file" accept=".csv"
             onChange={e => handleFile(e.target.files[0])} />
      <progress value={progress} max="1" />
      <p>{transactions.length} টি লেনদেন লোড হয়েছে</p>
    </>
  );
}
```

### tx-worker.js

```javascript
self.onmessage = async (e) => {
  const { type, buffer } = e.data;
  if (type !== 'PARSE') return;

  const text = new TextDecoder('utf-8').decode(buffer);
  const lines = text.split('\n');
  const total = lines.length;
  const result = new Array(total - 1);

  for (let i = 1; i < total; i++) {
    const [date, type, amount, fee, balance, ref] = lines[i].split(',');
    result[i - 1] = {
      date: new Date(date),
      type,
      amount: parseFloat(amount),
      fee: parseFloat(fee),
      balance: parseFloat(balance),
      ref
    };

    if (i % 5000 === 0) {
      self.postMessage({ type: 'PROGRESS', payload: i / total });
    }
  }

  // aggregations
  const summary = {
    totalSent:     result.filter(t => t.type === 'send').reduce((s, t) => s + t.amount, 0),
    totalReceived: result.filter(t => t.type === 'receive').reduce((s, t) => s + t.amount, 0),
    totalFee:      result.reduce((s, t) => s + t.fee, 0)
  };

  self.postMessage({ type: 'DONE', payload: { transactions: result, summary } });
};
```

---

## 🤝 Shared Worker

এক origin-এর একাধিক tab share করে।

```javascript
// shared-worker.js
const ports = [];
let cache = {};

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.push(port);

  port.onmessage = (msg) => {
    if (msg.data.type === 'SET') {
      cache[msg.data.key] = msg.data.value;
      // সব tab-এ broadcast
      ports.forEach(p => p.postMessage({ type: 'UPDATE', cache }));
    }
  };
};
```

```javascript
// main.js (in any tab)
const worker = new SharedWorker('./shared-worker.js');
worker.port.start();
worker.port.postMessage({ type: 'SET', key: 'cart', value: [...] });
worker.port.onmessage = (e) => updateUI(e.data.cache);
```

⚠️ Safari ১৬ থেকে আবার সাপোর্ট আসছে কিন্তু এখনো spotty। Production-এ BroadcastChannel
API বেশি reliable।

### BroadcastChannel — alternative

```javascript
const bc = new BroadcastChannel('daraz-cart');
bc.postMessage({ type: 'CART_UPDATED', items: [...] });
bc.onmessage = (e) => updateCart(e.data.items);
```

---

## ⚖️ Worker vs Node.js worker_threads

| বৈশিষ্ট্য | Browser Web Worker | Node.js worker_threads |
|----------|---------------------|------------------------|
| Module type | `type: 'module'` | ESM/CJS native |
| DOM access | ❌ | ❌ |
| File system | ❌ (limited via OPFS) | ✅ |
| Shared memory | SharedArrayBuffer | SharedArrayBuffer + workerData |
| Communication | postMessage | parentPort / MessageChannel |
| Use case | UI offload | CPU-bound server tasks |
| Crypto API | Web Crypto | crypto module |

কোডের বেশিরভাগ logic একই — postMessage pattern almost identical।

---

## 🔒 SharedArrayBuffer & Atomics

দুটো thread একই memory region share করতে পারে।

```javascript
// main.js
const sab = new SharedArrayBuffer(1024);
const view = new Int32Array(sab);

const worker = new Worker('./worker.js');
worker.postMessage({ sab });

// race condition prevent
Atomics.store(view, 0, 42);
const value = Atomics.load(view, 0);

// wait/notify (lock-free synchronization)
Atomics.wait(view, 0, 42);   // value 42 পরিবর্তন হওয়া পর্যন্ত wait
Atomics.notify(view, 0, 1);  // 1 জনকে wake
```

⚠️ **Spectre vulnerability**-এর কারণে SharedArrayBuffer চাইলে `Cross-Origin-Opener-Policy:
same-origin` ও `Cross-Origin-Embedder-Policy: require-corp` header লাগবে।

---

## 🛠️ Library সাহায্য — Comlink

postMessage boilerplate এড়িয়ে RPC-style API:

```javascript
// worker.js
import { expose } from 'comlink';

const api = {
  parseCSV(buffer) { /* ... */ return result; },
  filterByDate(txs, from, to) { /* ... */ }
};
expose(api);
```

```javascript
// main.js
import { wrap } from 'comlink';
const api = wrap(new Worker(...));
const result = await api.parseCSV(buffer);  // proxy call!
```

মনে হবে synchronous function call — পেছনে postMessage হচ্ছে।

---

## 🧩 Bundler integration (Vite, Webpack)

### Vite

```javascript
// auto-bundles
const worker = new Worker(
  new URL('./worker.js', import.meta.url),
  { type: 'module' }
);

// ?worker import syntax
import MyWorker from './worker.js?worker';
const w = new MyWorker();
```

### Webpack 5

```javascript
const worker = new Worker(
  new URL('./worker.js', import.meta.url)
);
```

Webpack 5 native worker support দেয় — manual loader দরকার নেই।

---

## ⚠️ Pitfalls

```
  ❌ DOM access কর‌তে চাওয়া — worker-এ document/window নেই
  ❌ Object large করে clone — performance killer; transferable ব্যবহার করুন
  ❌ Function পাঠানো — clone হবে না, error
  ❌ Worker terminate না করা → memory leak
  ❌ ৫০টা worker spawn করা → CPU overload (max 2-4 typical)
  ❌ Async error catch না করা → silent failure
  ❌ TypeScript-এ self typing miss → as any দরকার (or "lib": ["webworker"])
  ❌ Cross-origin worker — CORS rule
  ❌ Same-origin policy — file:// থেকে worker চলবে না
  ❌ Bundle dependency duplication — main + worker দুটোতেই lodash bundled
```

### TypeScript worker config

```json
// tsconfig.worker.json
{
  "compilerOptions": {
    "lib": ["ES2022", "WebWorker"],
    "types": []
  }
}
```

---

## ✅ যেখানে ব্যবহার করুন

- ✅ Computation > 50ms (UI jank threshold)
- ✅ ৫০ MB+ data parse (CSV, JSON, binary)
- ✅ Image / video / audio processing
- ✅ Cryptographic operation
- ✅ ML inference (TensorFlow.js)
- ✅ Real-time signal processing (WebRTC)

## ❌ যেখানে এড়ান

- ছোট কাজ (< 50ms) — postMessage overhead = ~১-২ms
- DOM-heavy logic (worker-এ DOM নেই)
- Worker-এ frequent state read (round-trip slow)
- Simple async I/O (fetch already non-blocking)

---

## 📋 সারসংক্ষেপ

```
  Web Worker — কখন কোনটি
  ════════════════════════════════════════════════
  Dedicated Worker  →  default, এক page-এর task
  Shared Worker     →  সব tab share, central cache
  Service Worker    →  network proxy, PWA
  
  Best practices:
  ─────────────────────────────────
  • postMessage payload ছোট রাখুন
  • Big data = Transferable Object
  • OffscreenCanvas = worker-এ render
  • Comlink = ergonomics বাড়ায়
  • SharedArrayBuffer = COOP/COEP লাগবে
  • Cleanup: worker.terminate()
  
  Rule of thumb:
  ─────────────────────────────────
  ৫০+ms blocking = worker-এ দিন
  ১০ ms = মূল thread-এ থাক
```

Web Worker দিয়ে frontend মূলত একটা **mini server** পায় — UI thread কে শুধু rendering-এ
focus রাখুন, heavy lifting অন্য thread-এ। ফলে আপনার অ্যাপ low-end Android-এও smooth
চলবে — যেটা বাংলাদেশি ইউজারদের জন্য সবচেয়ে গুরুত্বপূর্ণ।
