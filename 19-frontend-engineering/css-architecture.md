# 🎨 CSS আর্কিটেকচার (CSS Architecture)

## ভূমিকা

ছোট প্রজেক্টে যেকোনো ভাবে CSS লেখা যায়। কিন্তু Daraz/Foodpanda-র মতো ১০+ developer, ১০০+ component, একাধিক theme, dark mode, RTL support — এমন প্রজেক্টে disciplined architecture ছাড়া CSS ৬ মাসে অপ্রবেশ্য হয়ে পড়ে। **CSS Architecture** = scalable, maintainable, predictable styles লেখার রীতি ও pattern। এই ফাইলে আমরা দেখব BEM, OOCSS, SMACSS, ITCSS, Atomic CSS philosophy; CSS Modules, CSS-in-JS, Tailwind comparison; design tokens, theming, dark mode; এবং modern features — `@layer`, container queries, `:has()`।

```
   বড় প্রজেক্টে CSS-এর ৪ শত্রু
   ──────────────────────────────
   1. Specificity wars  (!important everywhere)
   2. Global namespace  (class clash)
   3. Dead CSS         (কেউ জানে না delete safe কিনা)
   4. Inconsistency    (১০ shade নীল রঙ)
```

---

## 🧱 Naming Methodologies

### ১. BEM (Block, Element, Modifier)

সবচেয়ে জনপ্রিয় naming convention। Block = standalone component, Element = block-এর অংশ, Modifier = variant।

```
   block__element--modifier
   ────────────────────────
   .card                    (block)
   .card__title             (element)
   .card__image             (element)
   .card--featured          (modifier)
   .card__title--large      (element + modifier)
```

```html
<article class="card card--featured">
  <img class="card__image" src="...">
  <h2 class="card__title card__title--large">মেগা সেল</h2>
  <p class="card__price">৳ ৪৯৯</p>
</article>
```

```css
.card { padding: 16px; border-radius: 8px; }
.card--featured { border: 2px solid gold; }
.card__title { font-size: 18px; }
.card__title--large { font-size: 24px; }
```

✅ **লাভ**: low specificity (always single class), no nesting needed, predictable।
❌ **ক্ষতি**: long class names, verbose HTML।

### ২. OOCSS (Object-Oriented CSS)

Nicole Sullivan-এর নীতি: **structure** এবং **skin** আলাদা।

```css
/* Object: layout structure */
.media { display: flex; gap: 12px; }
.media__img { flex-shrink: 0; }
.media__body { flex: 1; }

/* Skin: visual style — যেকোনো object-এ ব্যবহার */
.btn-primary { background: #f57224; color: #fff; }
.btn-secondary { background: #eee; color: #333; }
```

```html
<div class="media">
  <img class="media__img" src="...">
  <div class="media__body">
    <button class="btn-primary">কিনুন</button>
  </div>
</div>
```

### ৩. SMACSS (Scalable & Modular Architecture for CSS)

৫ category-তে CSS ভাগ:
1. **Base** — element selectors (h1, p, body)
2. **Layout** — `.l-header`, `.l-sidebar`, `.l-grid`
3. **Module** — reusable component (card, btn)
4. **State** — `.is-active`, `.is-hidden`, `.is-loading`
5. **Theme** — `.theme-dark`, `.theme-eid`

### ৪. ITCSS (Inverted Triangle CSS)

Specificity বাড়ার ক্রমে CSS organize:

```
        Specificity →  low ─────────────────▶ high
                       reach →    wide ─────▶ narrow
        ┌─────────────────────────────────────────┐
   1   │ Settings  (variables, no CSS output)    │
   2   │ Tools     (mixins, functions)            │
   3   │ Generic   (reset, normalize, box-sizing) │
   4   │ Elements  (h1, a, p — no class)          │
   5   │ Objects   (.o-container, .o-grid — OOCSS)│
   6   │ Components(.c-card, .c-btn)              │
   7   │ Utilities (.u-text-center, .u-mt-2)      │
        └─────────────────────────────────────────┘
   8   │ Trumps   (!important, hacks — last)      │
```

বাস্তব Sass structure:
```
styles/
  ├── 01-settings/
  ├── 02-tools/
  ├── 03-generic/
  ├── 04-elements/
  ├── 05-objects/
  ├── 06-components/
  └── 07-utilities/
```

### ৫. Atomic CSS / Functional CSS

প্রতিটি class একটি কাজ করে। **Tailwind** এই philosophy-র জনপ্রিয় বাস্তবায়ন।

```html
<button class="bg-orange-500 hover:bg-orange-600 text-white font-bold
               py-2 px-4 rounded shadow-md">
  কিনুন
</button>
```

❌ "ugly HTML"? হ্যাঁ। ✅ কিন্তু zero CSS file growth, locality of behavior, easy refactor।

---

## ⚖️ Methodology Comparison

| Approach    | Class structure | CSS file size | Refactor cost | Learning |
|-------------|-----------------|---------------|---------------|----------|
| BEM         | Long, semantic | Linear growth | Medium        | Easy     |
| OOCSS       | Composable     | Slow growth   | Low           | Medium   |
| SMACSS      | Categorized    | Linear        | Low           | Medium   |
| ITCSS       | Layered        | Linear        | Low           | Hard     |
| Atomic/TW   | Utility chain  | Constant (purged) | Very low  | Medium   |

---

## 🎯 Styling Approaches — Modern Choices

### CSS Modules

প্রতিটি file local-scope class বানায়। Webpack/Vite hash যোগ করে।

```css
/* Card.module.css */
.card { padding: 16px; }
.title { font-size: 18px; }
.featured { border: 2px solid gold; }
```

```jsx
import styles from './Card.module.css';

export function Card({ featured }) {
  return (
    <article className={`${styles.card} ${featured ? styles.featured : ''}`}>
      <h2 className={styles.title}>...</h2>
    </article>
  );
}
```

✅ Build-time, zero runtime cost, automatic scoping।

### CSS-in-JS (styled-components, Emotion)

```jsx
import styled from 'styled-components';

const Card = styled.article`
  padding: 16px;
  border-radius: 8px;
  ${(p) => p.featured && `border: 2px solid gold;`}
`;

const Title = styled.h2`
  font-size: ${(p) => (p.large ? '24px' : '18px')};
  color: ${(p) => p.theme.text};
`;

export default function Product({ featured }) {
  return (
    <Card featured={featured}>
      <Title large>{name}</Title>
    </Card>
  );
}
```

✅ Dynamic styles based on props, theme via Provider, dead-code elimination।
❌ Runtime cost (parse + insert styles), bundle size, SSR complexity।

**Modern alternatives**: zero-runtime CSS-in-JS — **Linaria**, **vanilla-extract**, **Panda CSS**। Build time-এ extract করে।

### Tailwind / Utility-first

```jsx
function ProductCard({ featured, name, price }) {
  return (
    <article className={`
      p-4 rounded-lg shadow-sm
      ${featured ? 'border-2 border-amber-400' : 'border border-gray-200'}
      hover:shadow-md transition-shadow
    `}>
      <h2 className="text-lg font-semibold">{name}</h2>
      <p className="text-orange-600 font-bold">৳ {price}</p>
    </article>
  );
}
```

`tailwind.config.js`:
```javascript
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#fff7ed',
          500: '#f57224',  // Daraz orange
          900: '#7c2d12',
        },
      },
      fontFamily: {
        bangla: ['Hind Siliguri', 'sans-serif'],
      },
    },
  },
  plugins: [require('@tailwindcss/forms')],
};
```

### Comparison Table

| Approach          | Runtime cost | Bundle | Dynamic | DX     | SSR |
|-------------------|--------------|--------|---------|--------|------|
| Plain CSS         | None         | Static | Hard    | OK     | Easy |
| CSS Modules       | None         | Static | Limited | Good   | Easy |
| Sass / Less       | None         | Static | Build-time | Good | Easy |
| styled-components | Medium       | +12KB  | ✅      | ⭐⭐⭐  | Tricky |
| Emotion           | Medium       | +8KB   | ✅      | ⭐⭐⭐  | Tricky |
| vanilla-extract   | None         | Static | Theme   | ⭐⭐⭐  | Easy |
| Tailwind CSS      | None         | Tiny (purged) | Class swap | ⭐⭐⭐⭐ | Easy |

---

## 🎨 Design Tokens

Design token = design decisions in data form। কোনো hard-coded `#f57224` নয় — token name।

```javascript
// tokens.js (single source of truth)
export const tokens = {
  color: {
    brand: {
      primary:   '#f57224',   // Daraz orange
      secondary: '#0f146d',
    },
    text: {
      primary:   'hsl(0 0% 12%)',
      secondary: 'hsl(0 0% 40%)',
      inverse:   'hsl(0 0% 98%)',
    },
    surface: {
      base:  '#ffffff',
      raised: '#f7f7f7',
    },
  },
  spacing: {
    xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px',
  },
  radius: { sm: '4px', md: '8px', lg: '12px', full: '9999px' },
  font: {
    family: { bangla: '"Hind Siliguri", sans-serif' },
    size:   { sm: '14px', md: '16px', lg: '18px', xl: '24px' },
    weight: { regular: 400, medium: 500, bold: 700 },
  },
  shadow: {
    sm: '0 1px 2px rgba(0,0,0,.05)',
    md: '0 4px 6px rgba(0,0,0,.07)',
  },
};
```

CSS-এ:
```css
:root {
  --color-brand-primary: #f57224;
  --color-text-primary: hsl(0 0% 12%);
  --space-md: 16px;
  --radius-md: 8px;
}

.btn-primary {
  background: var(--color-brand-primary);
  padding: var(--space-md);
  border-radius: var(--radius-md);
}
```

**Style Dictionary** / **Tokens Studio** এই tokens-কে CSS, JS, iOS, Android — সব platform-এর জন্য build করে।

---

## 🌗 Dark Mode + Theming

### CSS Custom Properties + `prefers-color-scheme`

```css
:root {
  --bg: #ffffff;
  --text: #111111;
  --surface: #f7f7f7;
  --border: #e5e5e5;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #0b0b0d;
    --text: #f5f5f5;
    --surface: #18181b;
    --border: #27272a;
  }
}

/* manual toggle override */
[data-theme="dark"] {
  --bg: #0b0b0d;
  --text: #f5f5f5;
  /* ... */
}

body { background: var(--bg); color: var(--text); }
```

```javascript
// localStorage + system preference combined
function applyTheme() {
  const stored = localStorage.getItem('theme');
  const system = window.matchMedia('(prefers-color-scheme: dark)').matches
    ? 'dark' : 'light';
  document.documentElement.dataset.theme = stored ?? system;
}
applyTheme();
```

### Multi-theme (Eid theme, Pohela Boishakh)

```css
[data-theme="eid"] {
  --color-brand-primary: #1f8b4c;     /* green */
  --color-accent: #ffd700;            /* gold */
}

[data-theme="boishakh"] {
  --color-brand-primary: #d63384;     /* lal */
  --pattern-bg: url('/patterns/alpana.svg');
}
```

Daraz BD-এর campaign-specific theme এই pattern দিয়ে এক click-এ swap।

---

## ⚔️ Specificity & Cascade Pitfalls

```
   Specificity Calculator
   ───────────────────────
   Inline style       1 0 0 0
   #id                0 1 0 0
   .class, [attr], :pseudo  0 0 1 0
   element, ::pseudo  0 0 0 1
   * (universal)      0 0 0 0
   !important         💀 (overrides everything except later !important)
```

**সমস্যা example:**
```css
.card .title { color: blue; }              /* (0,0,2,0) */
.title-red { color: red; }                 /* (0,0,1,0) — হারে! */

#main .title-red { color: red; }           /* (0,1,1,0) — জিতে কিন্তু ID বাজে */
```

**সমাধান rules:**
1. **শূন্য বা একটি class-এ থামুন** (BEM helps)
2. **ID selector ব্যবহার করবেন না styling-এ**
3. **`!important` শুধু utility class-এ** (Tailwind করে)
4. **Source order matters** — পরে আসা rule আগের same-specificity rule overrides

---

## 🆕 Modern CSS Features

### `@layer` — Cascade Layers (২০২২+)

ITCSS-এর native সমাধান। Layer order define করে cascade পুরো নিয়ন্ত্রণ।

```css
@layer reset, base, layout, components, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; margin: 0; }
}

@layer components {
  .btn { padding: 8px 16px; background: var(--brand); }
}

@layer utilities {
  .text-center { text-align: center; }
}
```

মূল magic: `@layer utilities`-এর `.text-center` সবসময় `@layer components`-এর `.btn` থেকে stronger — even though specificity equal। Layer order সিদ্ধান্ত নেয়।

### Container Queries

Component-level responsive — viewport নয়, parent container দেখে।

```css
.product-grid {
  container-type: inline-size;
  container-name: grid;
}

.card { display: block; }

@container grid (min-width: 600px) {
  .card { display: flex; }
}

@container grid (min-width: 900px) {
  .card { padding: 24px; }
}
```

একই `.card` component sidebar-এ vertical, main-এ horizontal — media query কখনো এটা সমাধান করতে পারত না।

### `:has()` — Parent Selector

```css
/* card-এ image থাকলে আলাদা padding */
.card:has(img) {
  padding-top: 0;
}

/* form-এ invalid input থাকলে submit disable look */
form:has(input:invalid) .submit {
  opacity: 0.5;
  pointer-events: none;
}

/* Bangla text থাকলে font switch */
:has(> :lang(bn)) {
  font-family: 'Hind Siliguri', sans-serif;
}
```

### আরও modern features

- `aspect-ratio: 16 / 9;` — image/video CLS prevent
- `gap` (flexbox + grid)
- `clamp(min, preferred, max)` — fluid typography
- `color-mix(in oklch, var(--brand) 80%, white)`
- `inset: 0` (= top:0; right:0; bottom:0; left:0)
- `accent-color: var(--brand)` — checkbox/radio recolor
- Logical properties: `margin-inline`, `padding-block` (RTL friendly)
- `@scope` (২০২৩+)

---

## 🇧🇩 বাস্তব উদাহরণ

### Daraz BD — Campaign theming

```css
/* base brand */
:root {
  --brand: #f57224;
  --brand-text: white;
}

/* 11.11 sale */
[data-campaign="11-11"] {
  --brand: #d50000;
  --bg-pattern: url('/11-11-bg.svg');
}

/* Eid */
[data-campaign="eid"] {
  --brand: #1f8b4c;
  --accent: #ffd700;
}
```

Backend `data-campaign` attribute set করে, CSS auto-switch।

### Foodpanda — Atomic Design + Tailwind

Foodpanda-র মতো বহু-country app: Tailwind tokens প্রতি market customizable। Bangla typography, RTL (Arabic), price formatting — সব design token দিয়ে।

### Pathao — Container queries (ride card)

Pathao-র সাইডবারে ride card narrow, main view-এ wide। Container query দিয়ে একই component দুই layout-এ adapt করে — JavaScript prop পরিবর্তন ছাড়াই।

### Bangla typography concerns

- Font: **Hind Siliguri**, **Noto Sans Bengali**, **Kalpurush**
- `line-height` Bangla-তে English-এর চেয়ে বেশি দরকার (কার, যফলা, রেফ — ascender/descender বেশি)
- `font-display: swap` + preload critical font subset
- Text size — Bangla character ৬০-৭০% width English-এর তুলনায়, তাই size সামান্য বাড়ান

```css
@font-face {
  font-family: 'Hind Siliguri';
  src: url('/fonts/HindSiliguri.woff2') format('woff2');
  font-display: swap;
  unicode-range: U+0980-09FF, U+200C-200D;  /* শুধু Bangla */
}

:lang(bn) {
  font-family: 'Hind Siliguri', sans-serif;
  line-height: 1.7;
  font-size: 1.05em;
}
```

---

## ⚠️ সাধারণ Pitfalls

| ভুল                              | পরিণতি                       | সমাধান                       |
|-----------------------------------|------------------------------|------------------------------|
| `!important` overuse              | Specificity war              | BEM, low specificity         |
| Deeply nested SCSS (`.a .b .c .d`)| Specificity, fragile         | Flat structure, BEM          |
| Hard-coded color/spacing          | Inconsistency                | Design tokens (CSS vars)     |
| Global class names (`.title`)     | Naming clash                 | CSS Modules / BEM / Tailwind |
| Media query everywhere            | Component-level bugs        | Container queries            |
| Dark mode = duplicate CSS         | 2x maintenance               | CSS variables                |
| Tailwind class spaghetti          | Hard to read                 | `@apply` for repeated, components |
| `min-height: 100vh` mobile        | Address bar bug              | `100dvh`                     |
| `*` reset heavy                   | Browser default loss         | Modern reset (Andy Bell)     |
| Unused CSS bundled                | Big payload                  | Tailwind purge / PurgeCSS    |

---

## ✅ চেকলিস্ট

- [ ] একটা naming methodology চয়েস করেছেন (BEM/Tailwind/CSS Modules)?
- [ ] Design tokens আছে — colors, spacing, fonts?
- [ ] Specificity flat — কোনো ID selector নেই?
- [ ] Dark mode CSS variables দিয়ে?
- [ ] `@layer` দিয়ে cascade নিয়ন্ত্রণ?
- [ ] Critical CSS inline?
- [ ] Unused CSS purged?
- [ ] Bangla font subset + `font-display: swap`?
- [ ] Container queries needed component-level হলে?
- [ ] RTL support (logical properties) যদি Arabic market থাকে?

---

## 📝 সারসংক্ষেপ

CSS architecture = বড় টিমে predictable, scalable styles লেখার discipline। **BEM** naming, **OOCSS/SMACSS/ITCSS** organization, **Atomic/Tailwind** utility-first — প্রতিটির trade-off আছে। আধুনিক default: **CSS Modules + design tokens** বা **Tailwind + tokens config**। **Design tokens** color/spacing/typography-কে data-as-code বানায় — multi-theme, dark mode, campaign skin সব এক variable swap। **Modern CSS** (`@layer`, container queries, `:has()`, logical properties) আগের অনেক hack remove করে দিয়েছে। বাংলাদেশী context-এ Bangla font, `:lang(bn)` styling, campaign theme (Eid/Boishakh/11-11) বাস্তব requirement।

> 💡 মনে রাখুন: **"Specificity flat রাখুন, tokens দিয়ে design করুন, layer দিয়ে cascade নিয়ন্ত্রণ করুন।"**
