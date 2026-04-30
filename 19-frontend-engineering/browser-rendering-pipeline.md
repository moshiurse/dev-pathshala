# 🖼️ ব্রাউজার রেন্ডারিং পাইপলাইন (Critical Rendering Path)

## ভূমিকা

আপনি যখন `https://daraz.com.bd` টাইপ করে Enter চাপেন, পরবর্তী কয়েকশ মিলিসেকেন্ডের মধ্যে ব্রাউজার একটি জটিল পাইপলাইন রান করে — HTML বাইট থেকে স্ক্রিনে পিক্সেল পর্যন্ত। এই পাইপলাইনের নাম **Critical Rendering Path (CRP)**। সিনিয়র ফ্রন্টএন্ড ইঞ্জিনিয়ার হিসেবে এটি না বুঝলে কেন একটা পেজ লো-এন্ড অ্যান্ড্রয়েডে ৮ সেকেন্ড নেয়, কেন `box-shadow` অ্যানিমেট করা খারাপ, কেন `transform` ভালো — কিছুই ব্যাখ্যা করতে পারবেন না।

```
URL → Network → HTML bytes → Parser → DOM
                              ↓
              CSS bytes → Parser → CSSOM
                              ↓
                        Render Tree
                              ↓
                          Layout
                              ↓
                           Paint
                              ↓
                        Composite → 🖥️ Pixels
```

---

## 🔄 সম্পূর্ণ পাইপলাইন (ধাপে ধাপে)

```
       ব্রাউজার রেন্ডারিং পাইপলাইন — Full View
═══════════════════════════════════════════════════════════════

  1. NETWORK
     │  TCP/TLS handshake → HTTP request → response bytes
     ▼
  2. PARSING (HTML)
     │  bytes → characters → tokens → nodes → DOM Tree
     │  (encounters <link>, <script>, <img> → parallel fetches)
     ▼
  3. CSSOM CONSTRUCTION
     │  CSS bytes → tokens → CSSOM Tree (render-blocking!)
     ▼
  4. JAVASCRIPT EXECUTION
     │  <script> blocks parser unless async/defer/module
     ▼
  5. RENDER TREE
     │  DOM ⊗ CSSOM = Render Tree (display:none বাদ যায়)
     ▼
  6. LAYOUT (Reflow)
     │  geometry calculation: x, y, width, height
     ▼
  7. PAINT
     │  pixels: colors, borders, shadows, text → layers
     ▼
  8. COMPOSITE
     │  GPU combines layers → final frame on screen
     ▼
  🖥️  PIXELS VISIBLE TO USER
```

প্রতিটি ধাপ আলাদাভাবে দেখা যাক।

---

## ১. DOM (Document Object Model) তৈরি

HTML বাইটগুলো প্রথমে characters → tokens (start tag, attribute, text) → nodes-এ রূপান্তরিত হয়। তারপর parent-child সম্পর্ক অনুযায়ী একটি **DOM Tree** তৈরি হয়।

```html
<!DOCTYPE html>
<html>
  <head><title>Daraz</title></head>
  <body>
    <header>...</header>
    <main><h1>মেগা সেল</h1></main>
  </body>
</html>
```

**DOM Tree:**
```
Document
└── html
    ├── head
    │   └── title → "Daraz"
    └── body
        ├── header
        └── main
            └── h1 → "মেগা সেল"
```

**গুরুত্বপূর্ণ পয়েন্ট:**
- HTML parser **incremental** — যতটুকু ডাউনলোড হয়েছে, ততটুকু পার্স করে।
- `<script>` ট্যাগ পেলে parser থামে (যদি না `async`/`defer`/`type="module"`)। এটাকে বলে **parser-blocking script**।
- `<link rel="stylesheet">` parser থামায় না কিন্তু **render-blocking** — CSSOM তৈরি না হলে রেন্ডার শুরু হবে না।

---

## ২. CSSOM (CSS Object Model) তৈরি

CSS bytes একইভাবে tokens → nodes → tree-তে রূপান্তরিত হয়। কিন্তু DOM-এর মতো incremental নয় — **CSSOM সম্পূর্ণ তৈরি না হওয়া পর্যন্ত ব্রাউজার কিছুই রেন্ডার করে না**। কারণ একটি লেট-আসা CSS rule আগের সব স্টাইল বদলে দিতে পারে (cascade)।

```css
body { font-size: 16px; }
main { font-size: 1.5em; }   /* 24px = 16 * 1.5 */
h1 { font-weight: bold; }
```

**CSSOM Tree** computed values ধারণ করে — অর্থাৎ `1.5em` already `24px` হিসেবে resolve হয়।

> ⚠️ **পরিণতি**: CSS render-blocking। তাই critical CSS inline করা, non-critical CSS `media="print"` বা JS-এ async load — performance optimization-এর মূল কৌশল।

---

## ৩. Render Tree

DOM + CSSOM একসাথে মিশিয়ে **Render Tree** তৈরি হয়। এতে শুধু **visible** elements থাকে।

```
DOM Tree         +  CSSOM         =   Render Tree
─────────────       ─────────────      ─────────────
html                html { ... }       html
  body                body { ... }       body
    div.hidden        .hidden {            header
    header              display:none}      main
    main              header { ... }         h1 (font: 24px bold)
      h1              main { ... }
                      h1 { bold }
```

`display: none` element render tree-তে আসে না। কিন্তু `visibility: hidden` আসে (জায়গা দখল করে, শুধু invisible)।

---

## ৪. Layout (Reflow)

Render Tree-তে প্রতিটি node-এর exact geometry (x, y, width, height) ক্যালকুলেট হয়। ব্রাউজার root থেকে শুরু করে recursive ভাবে box model হিসাব করে।

```
       Layout = Geometry calculation
       ─────────────────────────────
       <html> 1440x900
         <body> 1440x900, padding:0
           <header> 1440x80
           <main>   1440x820
             <h1>   1440x40, x=24, y=100
```

**Reflow trigger** এমন ঘটনা যা layout আবার চালাতে বাধ্য করে:
- DOM পরিবর্তন (`appendChild`, `removeChild`)
- জ্যামিতিক CSS পরিবর্তন: `width`, `height`, `padding`, `margin`, `border`, `top`, `left`, `font-size`
- ক্লাস পরিবর্তন যা layout বদলায়
- উইন্ডো resize, font load
- পড়া (read) properties যা synchronous layout force করে: `offsetWidth`, `offsetHeight`, `getBoundingClientRect()`, `scrollTop`, `clientWidth`

---

## ৫. Paint (Rasterization)

Layout শেষ হলে ব্রাউজার প্রতিটি element-কে pixels-এ "আঁকে": background, color, border, shadow, text, image। এই ধাপে **paint layers** তৈরি হয় — পেজ এক ছবি নয়, বরং একাধিক layer যা পরে একসাথে কম্পোজ হয়।

**Repaint trigger** (layout বদলায় না, শুধু চেহারা):
- `color`, `background-color`, `visibility`, `box-shadow`, `outline`

```
Paint vs Reflow Cost (relative)
─────────────────────────────────
  Layout (Reflow)  ████████████ 100%  (most expensive)
  Paint            ████████      60%
  Composite        ██            10%  (cheapest)
```

---

## ৬. Composite

প্রতিটি paint layer GPU-তে texture হিসেবে আপলোড হয়। GPU layers-কে z-order অনুযায়ী একসাথে combine করে final frame তৈরি করে স্ক্রিনে দেখায়।

**যে CSS properties শুধু composite ট্রিগার করে (সবচেয়ে সস্তা):**
- `transform` (translate, scale, rotate)
- `opacity`
- `filter` (কখনো কখনো)

**এই কারণেই** `transform: translateX(100px)` দিয়ে অ্যানিমেশন `left: 100px` থেকে অনেক দ্রুত — শেষেরটা layout + paint + composite, আগেরটা শুধু composite।

---

## 🔁 Reflow vs Repaint — তুলনা টেবিল

| ঘটনা                 | Reflow? | Repaint? | Composite? | Cost |
|----------------------|---------|----------|-----------|------|
| `width` change       | ✅      | ✅       | ✅        | High |
| `margin` / `padding` | ✅      | ✅       | ✅        | High |
| `display: none`      | ✅      | ✅       | ✅        | High |
| `font-size`          | ✅      | ✅       | ✅        | High |
| `color`              | ❌      | ✅       | ✅        | Med  |
| `background`         | ❌      | ✅       | ✅        | Med  |
| `box-shadow`         | ❌      | ✅       | ✅        | Med  |
| `transform`          | ❌      | ❌       | ✅        | Low  |
| `opacity`            | ❌      | ❌       | ✅        | Low  |

---

## 🎬 Frame Budget — 16.6ms রুল

60 FPS = প্রতি ফ্রেমে মাত্র **16.6 ms**। এই সময়ের মধ্যে JS execution + style + layout + paint + composite — সব শেষ করতে হবে।

```
  16.6 ms Frame Budget
  ┌──────────────────────────────────────────────┐
  │ JS  │ Style │ Layout │ Paint │ Composite │   │
  │ ~5ms│  ~2ms │  ~3ms  │  ~3ms │   ~2ms    │1ms│
  └──────────────────────────────────────────────┘
                                              ▲
                                       safety margin

  ➡️ JS যদি ১০ms+ নেয় → frame drop → jank!
```

মোবাইল (Pathao app, MyGP) লো-এন্ড ডিভাইসে CPU ৩-৪ গুণ ধীর। তাই দেস্কটপে ৫ms = মোবাইলে ১৫-২০ms। সহজেই frame drop হয়।

---

## 💻 কোড উদাহরণ — Reflow trigger

### ❌ খারাপ: Layout thrashing

```javascript
// ❌ লুপের মধ্যে বারবার read + write → প্রতিবার synchronous layout force করে
const items = document.querySelectorAll('.product-card');
for (const item of items) {
  // READ → layout force
  const w = item.offsetWidth;
  // WRITE → layout invalidate
  item.style.width = (w * 1.1) + 'px';
}
// যদি 100 কার্ড থাকে → 100টা reflow!
```

### ✅ ভালো: Read এবং Write আলাদা batch

```javascript
const items = document.querySelectorAll('.product-card');

// 1) সব READ একসাথে (একটাই layout)
const widths = [...items].map(it => it.offsetWidth);

// 2) সব WRITE একসাথে (একটাই reflow শেষে)
items.forEach((item, i) => {
  item.style.width = (widths[i] * 1.1) + 'px';
});
```

### ✅ আরও ভালো: `requestAnimationFrame` দিয়ে frame-aligned কাজ

```javascript
function smoothlyResize() {
  // READ phase
  const items = [...document.querySelectorAll('.card')];
  const widths = items.map(it => it.getBoundingClientRect().width);

  // WRITE phase — পরের ফ্রেমে
  requestAnimationFrame(() => {
    items.forEach((it, i) => {
      it.style.transform = `scaleX(${widths[i] * 0.01})`;
    });
  });
}
```

---

## 🎨 কোড উদাহরণ — Animation: ভালো বনাম খারাপ

```javascript
// ❌ খারাপ: প্রতিটি ফ্রেমে layout + paint
function moveBox(el) {
  let x = 0;
  setInterval(() => {
    x += 2;
    el.style.left = x + 'px';   // reflow!
  }, 16);
}

// ✅ ভালো: শুধু composite
function moveBoxFast(el) {
  let x = 0;
  function tick() {
    x += 2;
    el.style.transform = `translateX(${x}px)`;  // GPU only
    requestAnimationFrame(tick);
  }
  requestAnimationFrame(tick);
}
```

```css
/* ✅ ব্রাউজারকে hint দিন: এই element animate হবে → আগেই own layer দাও */
.product-card {
  will-change: transform;
}

/* ⚠️ অপব্যবহার করবেন না — সব element-এ will-change দিলে GPU memory খেয়ে ফেলবে */
```

---

## 🧱 GPU Compositing Layers

ব্রাউজার নির্দিষ্ট কিছু কন্ডিশনে একটি element-কে নিজের layer-এ promote করে — যাকে বলে **compositor layer / GPU layer**। এই layer GPU-তে texture হিসেবে থাকে এবং swap-only animations সম্ভব হয়।

**Layer promote-এর কারণ:**
- `transform: translateZ(0)` বা `translate3d(0,0,0)` (পুরোনো hack)
- `will-change: transform` / `will-change: opacity`
- `position: fixed`
- `<video>`, `<canvas>` (WebGL)
- CSS animations on transform/opacity
- `filter` properties

```
Without layers          With layers (GPU)
──────────────          ──────────────────
[Single bitmap]         [Layer 1: header  ]
   ↓ paint all          [Layer 2: cards   ]  ← translate only
                        [Layer 3: modal   ]
                              ↓
                        Composite on GPU
```

**সতর্কতা**: প্রতিটি layer GPU memory নেয়। ১০০টা card-এ `will-change: transform` দিলে মোবাইলে memory crash হতে পারে।

---

## 🛠️ DevTools Performance Panel দিয়ে ডিবাগিং

Chrome DevTools → **Performance** ট্যাব → Record → ৫-১০ সেকেন্ড → Stop।

```
Performance Timeline (যা দেখবেন)
────────────────────────────────────────────────────
  Main Thread:
  ┌────────────────────────────────────────────┐
  │ Task | Parse HTML | Recalc Style | Layout  │
  │ JS   |            |              |  Paint  │
  └────────────────────────────────────────────┘
  
  ⚠️  বড় হলুদ block = long JS task (>50ms)
  🔴  লাল চিহ্নিত frame = dropped frame (>16ms)
  
  Bottom-Up tab → কোন function সবচেয়ে বেশি সময় নিল
  Call Tree    → invocation tree
  Event Log    → timestamp-wise সব event
```

**Performance Insights** (নতুন panel) সরাসরি LCP, INP, CLS issues দেখায় — কোন element layout shift করছে, কোন interaction slow।

**Rendering tab tips:**
- "Paint flashing" enable → repaint area সবুজ flash করবে
- "Layer borders" → কোন element আলাদা layer-এ আছে দেখা যাবে
- "FPS meter" → real-time frame rate

---

## 🇧🇩 বাস্তব উদাহরণ — Daraz/Pathao মোবাইল অপ্টিমাইজেশন

### Daraz product list scroll

**সমস্যা**: ৫০০+ product card scroll করলে jank।

**সমাধান কৌশল:**
1. **Virtual scrolling** — শুধু viewport-এ যা দেখা যায় তার DOM রাখা (react-window, virtual-scroller)।
2. **`content-visibility: auto`** — viewport-এর বাইরে থাকা card layout/paint skip করে।
   ```css
   .product-card {
     content-visibility: auto;
     contain-intrinsic-size: 280px;  /* placeholder height */
   }
   ```
3. **Image lazy loading**: `<img loading="lazy" decoding="async">`
4. **Skeleton screens** — LCP candidate তাড়াতাড়ি paint করে CLS কমাতে।

### Pathao map + ride card animation

ম্যাপ animation transform দিয়ে — `top/left` ব্যবহার করলে ম্যাপের পুরো canvas reflow হবে। তাই:
```css
.ride-card-enter {
  transform: translateY(100%);
  opacity: 0;
  will-change: transform, opacity;
}
.ride-card-enter-active {
  transform: translateY(0);
  opacity: 1;
  transition: transform 250ms ease-out, opacity 200ms;
}
```

### bKash transaction list

পেমেন্ট কনফার্মেশনের পর list-এ নতুন item যোগ হয়। যদি পুরো list re-render হয় (React এ key ভুল হলে), তাহলে ৫০টা row-এর পুনরায় layout হয়। সঠিক `key` (`tx_id`) এবং memoization (`React.memo`) দিয়ে শুধু নতুন row paint হবে।

### Foodpanda restaurant list

Image-heavy। তাই `<img srcset>` + WebP/AVIF + lazy load + responsive sizing — যাতে 3G-তে অপ্রয়োজনীয় বড় image ডাউনলোড না হয়। `aspect-ratio` CSS দিয়ে CLS prevent।

---

## ⚠️ সাধারণ pitfall

| ভুল | সমস্যা | সঠিক সমাধান |
|------|--------|--------------|
| `<script>` head-এ blocking | Render delay | `defer` / নিচে রাখা / `async` |
| বড় CSS bundle | CSSOM blocking | Critical CSS inline, বাকি defer |
| `top/left` দিয়ে animation | Reflow + paint প্রতি frame | `transform` ব্যবহার |
| সব element-এ `will-change` | GPU memory পূর্ণ | শুধু animating element-এ |
| Synchronous DOM read in loop | Layout thrashing | Read/Write batch |
| Web font FOIT | Text invisible | `font-display: swap` |
| বড় hero image, no width/height | CLS | `width`/`height` attr দেওয়া |
| `box-shadow` heavy animation | Paint overhead | Pre-render shadow as image বা layer |

---

## ✅ চেকলিস্ট — Production-ready Rendering

- [ ] Critical CSS inline `<head>`-এ
- [ ] Non-critical CSS async load
- [ ] `<script>` `defer` বা `module`
- [ ] Image: `width`/`height`, `loading="lazy"`, modern format
- [ ] Animation শুধু `transform`/`opacity`-এ
- [ ] `will-change` শুধু প্রয়োজনীয় element-এ
- [ ] Long list-এ virtualization বা `content-visibility`
- [ ] Web fonts: `font-display: swap` + preload
- [ ] DevTools Performance audit মাসে একবার
- [ ] Lighthouse mobile + slow 4G simulation

---

## 📝 সারসংক্ষেপ

ব্রাউজার রেন্ডারিং পাইপলাইন = **HTML→DOM, CSS→CSSOM → Render Tree → Layout → Paint → Composite**। প্রতিটি ধাপের cost আলাদা — Layout সবচেয়ে দামি, Composite সবচেয়ে সস্তা। `transform`/`opacity` দিয়ে animate করলে শুধু composite ঘটে — এই কারণেই smooth ৬০ FPS সম্ভব। `width`/`top`/`margin` পরিবর্তন reflow ট্রিগার করে — যা ১৬.৬ms frame budget ভেঙে দেয়। DevTools Performance panel + Rendering tab দিয়ে মাপতে শিখুন। বাংলাদেশের লো-এন্ড ডিভাইসে এই অপ্টিমাইজেশনগুলো optional নয় — অনিবার্য।

> 💡 মনে রাখুন: **"যা দেখা যায় না, তা যেন রেন্ডারও না হয়। যা সরে, সেটা যেন GPU layer-এ থাকে।"**
