# ⚡ WebAssembly (Wasm) — Web-এ Native গতি

## 📌 সংজ্ঞা

**WebAssembly (Wasm)** হলো একটি **portable binary instruction format** যা ব্রাউজার ও
অন্যান্য runtime-এ near-native গতিতে চলে। C, C++, Rust, Go, AssemblyScript, Zig — এসব
ভাষায় কোড লিখে Wasm-এ compile করা যায়, এবং সেটি JavaScript-এর পাশাপাশি browser-এ চলে।

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   C / C++ / Rust / Go     ──compile──▶   .wasm binary    │
  │                                          ────────────    │
  │                                          • compact       │
  │                                          • portable      │
  │                                          • sandboxed     │
  │                                          • near-native   │
  │                                                          │
  │   Browser engines (V8, SpiderMonkey, JavaScriptCore)     │
  │   সরাসরি Wasm execute করে — JIT/AOT optimization        │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

> 💡 ২০১৭ থেকে Wasm সব major browser-এ stable। ২০১৯-এ W3C Recommendation।

---

## 🎯 কেন Wasm?

```
  JavaScript-এর সীমাবদ্ধতা
  ───────────────────────
  • Dynamic typing → JIT-এর কাজ কঠিন
  • GC pauses
  • Some workloads (video codec, crypto) JS-এ ৫-১০x ধীর
  • Existing C/C++ codebase rewrite করা impractical

  Wasm-এর শক্তি
  ─────────────
  • Static types → predictable performance
  • Compact binary (gzip-এ ছোট)
  • Sandboxed (browser security model মেনে)
  • Language portability (Rust → Wasm → Web)
  • Near-native speed (often 80-95% of native C)
```

| বিষয় | JavaScript | Wasm |
|------|-----------|------|
| Parsing | source code | binary (faster) |
| Type | dynamic | static |
| Memory | GC | manual / linear |
| Threads | Web Worker | Wasm Threads (via WW) |
| Speed | varies | predictable, fast |
| Source size | text | binary (compact) |

---

## 📚 Wasm-এর দুটি format

### বাইনারি (`.wasm`)

```
  00 61 73 6d 01 00 00 00  →  "\0asm" magic + version
  ...
```

### টেক্সট (`.wat` — WebAssembly Text)

```wat
;; দুটি integer যোগ করার Wasm module
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add)
  (export "add" (func $add)))
```

`.wat` মূলত debug/learning-এর জন্য — actual deployment-এ `.wasm` যায়।

### Stack-based VM

Wasm স্ট্যাক-ভিত্তিক ভার্চুয়াল মেশিন। প্রতিটি instruction stack-এ push/pop করে:

```
  i32.const 5      →  stack: [5]
  i32.const 3      →  stack: [5, 3]
  i32.add          →  stack: [8]
```

---

## 🔧 Toolchain — কীভাবে compile করব?

### ১. Rust → Wasm (`wasm-pack`) — সবচেয়ে recommended

```bash
# Install
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
cargo install wasm-pack

# Project create
cargo new --lib bkash-crypto
cd bkash-crypto
```

```toml
# Cargo.toml
[package]
name = "bkash-crypto"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
sha2 = "0.10"
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;
use sha2::{Sha256, Digest};

#[wasm_bindgen]
pub fn hash_transaction(data: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data.as_bytes());
    let result = hasher.finalize();
    format!("{:x}", result)
}

#[wasm_bindgen]
pub struct Calculator {
    value: f64,
}

#[wasm_bindgen]
impl Calculator {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Calculator { Calculator { value: 0.0 } }

    pub fn add(&mut self, x: f64) -> f64 {
        self.value += x;
        self.value
    }
}
```

```bash
wasm-pack build --target web
# pkg/ folder — JS bindings + .wasm + .d.ts
```

```javascript
// app.js
import init, { hash_transaction, Calculator } from './pkg/bkash_crypto.js';

await init();  // Wasm load
const hash = hash_transaction('TX-12345-500-bdt');
const calc = new Calculator();
calc.add(100);
calc.add(50);
```

### ২. C/C++ → Wasm (Emscripten)

```bash
# emsdk install
git clone https://github.com/emscripten-core/emsdk
cd emsdk && ./emsdk install latest && ./emsdk activate latest
source ./emsdk_env.sh

# compile
emcc image_filter.c -o filter.js -s EXPORTED_FUNCTIONS='["_apply_grayscale"]'
```

Emscripten C standard lib, OpenGL→WebGL, SDL→Canvas — সব এমুলেট করে। বড় গেম port করতে
উপযুক্ত (Unity, Unreal Wasm export এই system-এ)।

### ৩. AssemblyScript — TypeScript সিনট্যাক্স

```typescript
// assembly/index.ts
export function fibonacci(n: i32): i32 {
  if (n < 2) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

```bash
npm install -g assemblyscript
asc assembly/index.ts -b build/index.wasm
```

JavaScript developer-এর জন্য সবচেয়ে সহজ entry point।

### ৪. Go → Wasm

```bash
GOOS=js GOARCH=wasm go build -o main.wasm
```

Go runtime ভারী (~2MB)। সাধারণত Rust preferred।

---

## 🤝 JS ↔ Wasm Interop

### ১. সরাসরি instantiate

```javascript
const response = await fetch('/calc.wasm');
const { instance } = await WebAssembly.instantiateStreaming(response, {
  env: {
    // JS function-কে Wasm-এ inject
    log: (n) => console.log('Wasm says:', n)
  }
});

const result = instance.exports.add(5, 3);
console.log(result); // 8
```

### ২. Memory share

Wasm-এর memory একটি linear ArrayBuffer:

```javascript
const memory = instance.exports.memory; // WebAssembly.Memory
const view = new Uint8Array(memory.buffer);

// JS থেকে data লিখা
view[0] = 65; view[1] = 66;

// Wasm function পড়তে পারে memory[0]
```

### ৩. String pass করা

```rust
// Rust + wasm-bindgen এসব auto করে
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("নমস্কার {}", name)
}
```

```javascript
greet('আকাশ');  // wasm-bindgen automatically encodes string
```

ম্যানুয়ালি করতে হলে: JS UTF-8 encode → memory-তে copy → Wasm-এ pointer + length pass →
return-এ uncopy।

### ৪. Numeric types

| JS | Wasm |
|----|------|
| `Number` (small int) | `i32`, `i64` (BigInt) |
| `Number` (float) | `f32`, `f64` |
| `BigInt` | `i64` |

### ৫. Memory model

```
  ┌────────────────────────────────────────────────────────────┐
  │              Wasm Linear Memory (one big array)            │
  │   [0x00] [0x01] [0x02] ... [n] ... [0xFFFF...]            │
  │                                                            │
  │   Wasm: pointer (i32) দিয়ে address করে                    │
  │   JS:   new Uint8Array(memory.buffer) দিয়ে read/write     │
  │                                                            │
  │   Pages of 64KB; auto-growable                             │
  └────────────────────────────────────────────────────────────┘
```

---

## 🎯 Real Use Cases

### ১. Codecs

- **Image:** WebP, AVIF decoder (Squoosh.app — Wasm)
- **Video:** ffmpeg.wasm — browser-এ video transcode
- **Audio:** Opus, MP3 codec

### ২. Crypto

- **libsodium-wasm** — secure crypto in browser
- **Argon2 password hashing**
- **End-to-end encrypted chat (Signal Web)**

### ৩. Games

- **Unity WebGL** — pure Wasm
- **Unreal Engine 5** — Wasm export
- **Doom**, **Quake** — Wasm port
- **Figma** — entire design canvas in C++ → Wasm

### ৪. Productivity & Office

- **AutoCAD Web** — C++ → Wasm
- **Google Earth** — C++ → Wasm
- **Photoshop Web** — Adobe ported via Emscripten

### ৫. Data / ML

- **TensorFlow.js Wasm backend** — CPU inference faster
- **DuckDB-Wasm** — SQL analytics in browser
- **SQLite-Wasm** — offline DB

### ৬. Programming language runtime

- **Pyodide** — Python in browser (CPython → Wasm)
- **Ruby** — RubyVM Wasm
- **PHP-Wasm** — WordPress in browser

---

## 🇧🇩 কেস স্টাডি — bKash QR scanner

bKash অ্যাপের web version-এ QR code scanner চাই — barcode parsing JS-এ ৩০০ms,
Wasm-এ ৩০ms।

```javascript
// app.js
import init, { decode_qr } from './qr-wasm/pkg/qr_decoder.js';
await init();

navigator.mediaDevices.getUserMedia({ video: true }).then(stream => {
  const video = document.getElementById('cam');
  video.srcObject = stream;

  setInterval(() => {
    const canvas = document.createElement('canvas');
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(video, 0, 0);
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    const result = decode_qr(imgData.data, canvas.width, canvas.height);
    if (result) {
      processBkashQR(result);  // bkash:merchant=...
    }
  }, 100);
});
```

```rust
// qr_decoder Rust
use wasm_bindgen::prelude::*;
use rqrr::PreparedImage;

#[wasm_bindgen]
pub fn decode_qr(data: &[u8], width: u32, height: u32) -> Option<String> {
    let img = image::ImageBuffer::from_raw(width, height, data.to_vec())?;
    let mut prepared = PreparedImage::prepare(img);
    let grids = prepared.detect_grids();
    for grid in grids {
        if let Ok((_, content)) = grid.decode() {
            return Some(content);
        }
    }
    None
}
```

ফলাফল: 30fps real-time QR detection low-end Android-এ — JS পারে না।

---

## 🚀 Advanced Features

### SIMD (Single Instruction, Multiple Data)

একটি instruction-এ ৪/৮টি data parallel process।

```rust
#![feature(portable_simd)]
use std::simd::f32x4;

#[wasm_bindgen]
pub fn vec_add(a: &[f32], b: &[f32], out: &mut [f32]) {
    for i in (0..a.len()).step_by(4) {
        let va = f32x4::from_slice(&a[i..]);
        let vb = f32x4::from_slice(&b[i..]);
        (va + vb).copy_to_slice(&mut out[i..]);
    }
}
```

Image filtering, audio mixing, ML — SIMD ৪x পর্যন্ত speed-up।

### Wasm Threads

Web Worker + SharedArrayBuffer + Atomics দিয়ে multi-threaded Wasm।

```bash
# Rust
RUSTFLAGS='-C target-feature=+atomics,+bulk-memory' \
  wasm-pack build --target web -- -Z build-std=panic_abort,std
```

⚠️ COOP/COEP header লাগবে।

### Garbage Collection (WasmGC)

২০২৩-এ ship — Wasm directly use করতে পারে browser-এর GC, যা Java/Kotlin/Dart-এর জন্য
গুরুত্বপূর্ণ। Flutter Web এখন WasmGC ব্যবহার করছে।

### Component Model

বিভিন্ন language-এ লেখা Wasm modules-এর interop standard।

---

## 🌐 WASI — WebAssembly System Interface

Browser-এর বাইরে Wasm চালানোর standard — file I/O, network, env।

```
  Wasm + WASI  →  server-side, edge, IoT, plugin system
  ───────────────────────────────────────────────────
  • Cloudflare Workers (V8 isolates + Wasm)
  • Fastly Compute@Edge
  • Fermyon Spin
  • Docker Desktop runs Wasm
```

server-side use case ক্রমশ বড় হচ্ছে — ১ মিলি‌সেকেন্ড cold start, sandboxed,
language-agnostic plugin।

---

## 📦 Bundling & Loading

### Vite

```javascript
// .wasm import as URL
import wasmUrl from './module.wasm?url';
const wasm = await WebAssembly.instantiateStreaming(fetch(wasmUrl));
```

### ES Module Integration (proposal)

```javascript
// Future: direct import
import { add } from './math.wasm';
```

### Streaming compilation

`WebAssembly.instantiateStreaming` — download + compile একসাথে, fastest।

```javascript
// ❌ slow
const bytes = await fetch('m.wasm').then(r => r.arrayBuffer());
const m = await WebAssembly.instantiate(bytes);

// ✅ fast
const m = await WebAssembly.instantiateStreaming(fetch('m.wasm'));
```

Server-এ correct MIME type পাঠাতে হবে: `Content-Type: application/wasm`।

---

## ⚠️ কখন Wasm ব্যবহার **করবেন না**

```
  ❌ ছোট DOM update — JS-ই দ্রুত (interop overhead আছে)
  ❌ String-heavy logic — JS string optimized
  ❌ Async I/O ভিত্তিক কোড — তেমন benefit নেই
  ❌ Network-bound কাজ — bottleneck network
  ❌ ছোট bundle — Wasm overhead (loader + binary) >  benefit
  ❌ DOM access intensive — Wasm-এ DOM API নেই, প্রতিবার JS bridge
  ❌ Quick prototype — toolchain learning curve বেশি
```

### ⚖️ JS vs Wasm — Decision matrix

| পরিস্থিতি | উপযুক্ত |
|-----------|---------|
| CRUD form / dashboard | JS |
| ৫% হটস্পট compute-heavy | Wasm (just that part) |
| Existing C/C++/Rust codebase port | Wasm |
| Crypto / codec | Wasm |
| Image/video editor | Wasm |
| Game engine | Wasm |
| Plugin system | Wasm + WASI |

---

## 🔒 Security

Wasm sandboxed:

- **Memory:** শুধু own linear memory access করতে পারে
- **No DOM:** সরাসরি DOM access নেই (must go through JS)
- **No filesystem:** browser-এ সাধারণত নেই
- **Imports controlled:** যা import করবেন তা আপনি দেন

কিন্তু **side-channel attack** (Spectre, Meltdown) Wasm-কেও affect করে — তাই
SharedArrayBuffer ব্যবহারে COOP/COEP লাগে।

---

## ⚠️ Pitfalls

```
  ❌ DOM access expectation — Wasm থেকে DOM-এ যেতে JS bridge call
  ❌ String passing overhead — encode/decode round-trip
  ❌ Memory leak — Rust drop না হলে memory growing
  ❌ Wasm size ভুলে যাওয়া — ১৫MB Wasm = mobile-এ disaster
  ❌ Source map ছাড়া debug — কোডের কোন লাইন বুঝবেন না
  ❌ instantiate() vs instantiateStreaming() — wrong choice
  ❌ Content-Type ভুল server-এ — "magic number" mismatch error
  ❌ Memory.grow() না call করে large allocate — out of memory
  ❌ ১টা JS function call থেকে ১০০০ Wasm call — overhead dominate
  ❌ Different versions of bindgen client/server — runtime crash
```

---

## ✅ যেখানে ব্যবহার করুন

- 🎮 Game / 3D editor
- 🖼️ Image / video / audio processing
- 🔐 Crypto operations
- 📊 Heavy data analytics (DuckDB-Wasm)
- 🤖 ML inference (TensorFlow.js WASM)
- 🔁 Existing native code reuse (legacy C/C++)
- 🧩 Plugin systems (Figma plugins, Cloudflare Workers)
- ⚡ Performance-critical hotspot (5% of code, 80% of time)

## ❌ যেখানে এড়ান

- সাধারণ web app (form, list, dashboard)
- DOM-heavy interaction
- Small bundle size critical (mobile, low bandwidth)
- Quick prototype
- Team-এর Rust/C++ knowledge নেই

---

## 📋 সারসংক্ষেপ

```
  WebAssembly একনজরে
  ════════════════════════════════════════════════
  
  কী?       Compact, sandboxed binary format
  কোথায়?    Browser, Node.js, Cloudflare Workers, edge
  ভাষা?     C/C++, Rust, Go, AssemblyScript, Zig, Swift...
  
  Toolchains:
  ────────────
  • Rust + wasm-pack       → most modern, ergonomic
  • C/C++ + Emscripten      → legacy port, games
  • AssemblyScript          → TypeScript-like, easy entry
  • Go                      → simple, but heavy runtime
  
  Performance:
  ────────────
  • ৮০-৯৫% native C speed
  • Predictable (no GC pauses)
  • SIMD = 4x more
  • Threads via SharedArrayBuffer
  
  Use:
  ────
  • Hotspots, codecs, crypto, games, ML
  • Reusing native libraries
  • Plugin system
```

Wasm JavaScript-কে replace করতে আসেনি — **complement** করতে এসেছে। সাধারণ অ্যাপ
JavaScript-এ চলবে, ৫% performance hotspot Wasm-এ চলবে। বাংলাদেশের ক্ষেত্রে যেখানে
low-end device-এ heavy কম্পিউটেশন (image upload আগে compress, QR scan, video filter)
করতে হবে — Wasm-ই সমাধান।
