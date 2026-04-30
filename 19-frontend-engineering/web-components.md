# 🧩 Web Components — ব্রাউজার-নেটিভ কম্পোনেন্ট

## 📌 সংজ্ঞা

**Web Components** হলো W3C/WHATWG-এর তৈরি ৪টি browser API-র সেট, যা framework ছাড়াই
**reusable, encapsulated UI component** তৈরি করতে দেয়:

```
  ┌──────────────────────────────────────────────────────────┐
  │  ① Custom Elements   →  নিজের HTML tag (<bkash-button>)  │
  │  ② Shadow DOM        →  CSS/DOM encapsulation             │
  │  ③ HTML Templates    →  <template> / <slot>               │
  │  ④ ES Modules        →  মডিউল system (import/export)     │
  └──────────────────────────────────────────────────────────┘
```

React/Vue framework-এর মতো — কিন্তু **browser-native**, কোনো runtime/build tool বাধ্য না।

বাংলাদেশের context-এ গুরুত্বপূর্ণ: যদি Daraz, Pathao, bKash কে এক ডিজাইন সিস্টেম
share করতে হয় — তাহলে Web Components universal solution। React, Angular, Vue, plain
HTML — সবেতেই চলবে।

---

## 🤔 কেন Web Components?

```
  Framework Component vs Web Component
  ═════════════════════════════════════════════════════════════

                  React/Vue        Web Component
                  ──────────       ──────────────
  Runtime         ৩০-৪০ KB        ০ (browser-native)
  ফ্রেমওয়ার্ক      বাধ্যতামূলক     কোনোটাই না (interop যেকোনো)
  CSS scope       library/build    Shadow DOM (browser)
  Reusable        framework-only   universal
  Browser support বছরভিত্তিক       Chrome/Edge/Safari/FF সব
  Lifecycle       framework        browser API
```

> 💡 Daraz যদি Lazada-র সাথে design system share করতে চায় (Daraz হলো Lazada-র sister
> brand) — Daraz React, Lazada Vue হলেও একই `<lzd-button>` Web Component দুজনেই use করতে
> পারে।

---

## 🏷️ ১. Custom Elements

নিজের HTML tag তৈরি করা যায় — তবে নাম-এ অবশ্যই **dash (`-`)** থাকতে হবে (browser native
tag-এর সাথে collision এড়াতে)।

### সাধারণ syntax

```javascript
class BkashButton extends HTMLElement {
  // ── attributes observe ──
  static get observedAttributes() {
    return ['variant', 'disabled', 'loading'];
  }

  constructor() {
    super();
    // এখানে DOM access করবেন না — element হয়তো এখনো document-এ যোগ হয়নি
  }

  // ── element document-এ যোগ হলো ──
  connectedCallback() {
    this.render();
    this.addEventListener('click', this.handleClick);
  }

  // ── element document থেকে remove হলো ──
  disconnectedCallback() {
    this.removeEventListener('click', this.handleClick);
  }

  // ── attribute পরিবর্তিত হলো ──
  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue === newValue) return;
    this.render();
  }

  // ── অন্য document-এ adopt হলো (rare) ──
  adoptedCallback() {}

  handleClick = (e) => {
    if (this.hasAttribute('disabled') || this.hasAttribute('loading')) {
      e.stopPropagation();
      return;
    }
    this.dispatchEvent(new CustomEvent('bkash-click', {
      bubbles: true,
      composed: true,  // shadow DOM cross করে
      detail: { timestamp: Date.now() }
    }));
  };

  render() {
    const variant = this.getAttribute('variant') || 'primary';
    const loading = this.hasAttribute('loading');
    this.innerHTML = `
      <button class="bkash-btn bkash-btn--${variant}" ${loading ? 'disabled' : ''}>
        ${loading ? '⏳ লোড হচ্ছে...' : '<slot></slot>'}
      </button>
    `;
  }
}

customElements.define('bkash-button', BkashButton);
```

### ব্যবহার

```html
<bkash-button variant="primary">টাকা পাঠান</bkash-button>
<bkash-button variant="ghost" loading>প্রসেসিং</bkash-button>
<bkash-button disabled>OTP দিন</bkash-button>
```

### Lifecycle Callbacks বিস্তারিত

```
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  constructor()              ← element create হলো       │
  │       ↓                                                │
  │  connectedCallback()        ← document-এ যুক্ত হলো    │
  │       ↓                                                │
  │  attributeChangedCallback() ← attribute বদলায়          │
  │       ↓                                                │
  │  adoptedCallback()          ← document.adoptNode()    │
  │       ↓                                                │
  │  disconnectedCallback()     ← document থেকে removed   │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

| Callback | কখন | DOM access? |
|----------|-----|-------------|
| `constructor` | element instantiate হলে | ❌ না |
| `connectedCallback` | document-এ যুক্ত | ✅ হ্যাঁ |
| `disconnectedCallback` | remove | ✅ (cleanup-এর জন্য) |
| `attributeChangedCallback` | observed attribute change | ✅ হ্যাঁ |
| `adoptedCallback` | iframe-এ move হলে | ✅ |

---

## 🌑 ২. Shadow DOM

Shadow DOM হলো **encapsulated DOM tree** — element-এর ভেতরে আরেকটা DOM, যেটা বাইরের
CSS/JS থেকে আলাদা।

```
  ┌─────────────────────────────────────────────────┐
  │            Document (Light DOM)                 │
  │  ┌───────────────────────────────────────────┐  │
  │  │ <bkash-button>                            │  │
  │  │   #shadow-root (closed/open)              │  │
  │  │   ┌─────────────────────────────────────┐ │  │
  │  │   │ <style> ... </style>                │ │  │
  │  │   │ <button class="btn">                │ │  │
  │  │   │   <slot></slot>                     │ │  │
  │  │   │ </button>                           │ │  │
  │  │   └─────────────────────────────────────┘ │  │
  │  │ </bkash-button>                           │  │
  │  └───────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────┘

  বাইরের CSS:  button { color: red; }    ← shadow-এ ঢুকবে না
  ভেতরের CSS:  .btn { color: pink; }      ← শুধু এখানে কাজ করবে
```

### Shadow DOM সহ component

```javascript
class BkashCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' }); // 'closed' হলে JS থেকে inaccessible
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          border: 1px solid #e2136e;
          border-radius: 8px;
          padding: 16px;
          font-family: 'Noto Sans Bengali', sans-serif;
        }
        :host([elevated]) {
          box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        ::slotted(h2) {
          color: #e2136e;
          margin: 0 0 8px;
        }
        .body {
          color: #333;
        }
      </style>

      <slot name="title"></slot>
      <div class="body">
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('bkash-card', BkashCard);
```

```html
<bkash-card elevated>
  <h2 slot="title">আজকের লেনদেন</h2>
  <p>আপনি ৫,০০০ টাকা পাঠিয়েছেন</p>
</bkash-card>
```

### Shadow DOM-এর CSS selector

| Selector | কী |
|----------|----|
| `:host` | host element নিজে |
| `:host(.active)` | host-এ class থাকলে |
| `:host([disabled])` | host-এ attribute থাকলে |
| `:host-context(dark-theme)` | parent context-এ class থাকলে |
| `::slotted(p)` | slot-এ আসা element |
| `::part(label)` | exposed part style |

### ::part — selective exposure

```javascript
// component-এ
this.shadowRoot.innerHTML = `
  <button part="button">
    <span part="label"><slot></slot></span>
  </button>
`;
```

```css
/* বাইরে থেকে স্টাইল */
bkash-button::part(button) {
  background: #e2136e;
}
bkash-button::part(label) {
  font-weight: bold;
}
```

### CSS Custom Properties — theming

```css
/* component-এ */
:host {
  --bkash-color: var(--brand-color, #e2136e);
}
.btn { background: var(--bkash-color); }

/* parent-এ */
bkash-button { --bkash-color: #ff5a5f; }
```

CSS variables shadow boundary cross করে — এটিই official theming mechanism।

---

## 📋 ৩. HTML Templates & Slots

`<template>` — inert (inactive) HTML যা JS দিয়ে clone করে ব্যবহার করা যায়।

```html
<template id="product-card-template">
  <style>
    .card { border: 1px solid #ddd; padding: 12px; }
    .price { color: green; font-weight: bold; }
  </style>
  <article class="card">
    <img class="img" />
    <h3 class="name"></h3>
    <span class="price"></span>
    <slot name="actions"></slot>
  </article>
</template>
```

```javascript
class ProductCard extends HTMLElement {
  constructor() {
    super();
    const template = document.getElementById('product-card-template');
    const node = template.content.cloneNode(true);
    this.attachShadow({ mode: 'open' }).appendChild(node);
  }

  connectedCallback() {
    this.shadowRoot.querySelector('.name').textContent = this.getAttribute('name');
    this.shadowRoot.querySelector('.price').textContent = '৳' + this.getAttribute('price');
    this.shadowRoot.querySelector('.img').src = this.getAttribute('image');
  }
}
customElements.define('product-card', ProductCard);
```

```html
<product-card name="iPhone 15" price="125000" image="/iphone.jpg">
  <button slot="actions">কার্টে যোগ</button>
</product-card>
```

### Named slots vs default slot

```html
<my-modal>
  <h2 slot="header">পেমেন্ট নিশ্চিতকরণ</h2>     <!-- named slot -->
  <p>আপনি কি ৫০০ টাকা পাঠাতে চান?</p>          <!-- default slot -->
  <button slot="footer">নিশ্চিত করুন</button>   <!-- named slot -->
</my-modal>
```

```javascript
this.shadowRoot.innerHTML = `
  <header><slot name="header"></slot></header>
  <main><slot></slot></main>
  <footer><slot name="footer"></slot></footer>
`;
```

---

## 📦 ৪. ES Modules

```javascript
// components/bkash-button.js
export class BkashButton extends HTMLElement { /* ... */ }
customElements.define('bkash-button', BkashButton);
```

```html
<script type="module" src="/components/bkash-button.js"></script>
<!-- বা -->
<script type="module">
  import './components/bkash-button.js';
</script>
```

---

## ⚡ Lit — Web Components-এর জন্য micro-framework

vanilla Web Components লেখা boilerplate-heavy। Google-এর **Lit** (~5KB) declarative
template ও reactive update দেয়।

```bash
npm install lit
```

### Lit-এ component

```javascript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('bkash-button')
export class BkashButton extends LitElement {
  static styles = css`
    :host { display: inline-block; }
    button {
      background: var(--bkash-color, #e2136e);
      color: white;
      padding: 10px 20px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-family: 'Noto Sans Bengali', sans-serif;
    }
    button[disabled] { opacity: 0.5; cursor: not-allowed; }
  `;

  @property({ type: String }) variant = 'primary';
  @property({ type: Boolean }) loading = false;
  @property({ type: Boolean }) disabled = false;

  render() {
    return html`
      <button ?disabled=${this.disabled || this.loading} @click=${this._onClick}>
        ${this.loading ? html`⏳ লোড হচ্ছে...` : html`<slot></slot>`}
      </button>
    `;
  }

  _onClick(e) {
    this.dispatchEvent(new CustomEvent('bkash-click', {
      bubbles: true, composed: true
    }));
  }
}
```

```html
<bkash-button variant="primary" @bkash-click=${handler}>
  টাকা পাঠান
</bkash-button>
```

### Lit-এর সুবিধা

- Reactive properties (auto re-render)
- Tagged template literals (efficient diff)
- Decorator support
- TypeScript-friendly
- SSR support (`@lit-labs/ssr`)
- ~5KB minified

---

## 🤝 Framework Interop

### React-এ Web Component

```jsx
function App() {
  const ref = useRef();

  useEffect(() => {
    // events DOM way দিয়ে subscribe
    const handler = (e) => console.log(e.detail);
    ref.current.addEventListener('bkash-click', handler);
    return () => ref.current.removeEventListener('bkash-click', handler);
  }, []);

  return (
    <bkash-button ref={ref} variant="primary">
      টাকা পাঠান
    </bkash-button>
  );
}
```

> ⚠️ React 18 ও তার আগে: custom event subscribe & non-string props ম্যানুয়াল করতে হয়।
> **React 19** Web Components সরাসরি সাপোর্ট করে — events ও properties auto-bind।

### Vue-এ

```vue
<template>
  <bkash-button :variant="'primary'" @bkash-click="onClick">
    টাকা পাঠান
  </bkash-button>
</template>

<script setup>
function onClick(e) { console.log(e.detail); }
</script>

<!-- vite.config.js-এ -->
<!-- vue: { template: { compilerOptions: { isCustomElement: (tag) => tag.includes('-') } } } -->
```

### Angular-এ

```typescript
// app.module.ts
@NgModule({
  schemas: [CUSTOM_ELEMENTS_SCHEMA]
})
```

```html
<bkash-button [attr.variant]="'primary'" (bkash-click)="onClick($event)">
  টাকা পাঠান
</bkash-button>
```

---

## 🏢 কেস স্টাডি — Daraz Group Design System

Daraz/Lazada/Lazada Indonesia-র মধ্যে ৬টি দেশ। প্রতিটি দেশের team আলাদা framework
(some React, some Vue)।

```
  ┌──────────────────────────────────────────────────────────┐
  │     লাজাদা গ্রুপের ডিজাইন সিস্টেম: <lzd-*>               │
  │                                                          │
  │     ┌────────────┐  ┌────────────┐  ┌────────────┐      │
  │     │ Daraz BD   │  │ Lazada SG  │  │ Lazada ID  │      │
  │     │  React     │  │   Vue      │  │  Next.js   │      │
  │     └─────┬──────┘  └──────┬─────┘  └──────┬─────┘      │
  │           │                │               │            │
  │           └────────────────┼───────────────┘            │
  │                            ▼                            │
  │             ┌─────────────────────────────┐             │
  │             │   @lzd/web-components       │             │
  │             │   ──────────────────        │             │
  │             │   <lzd-button>              │             │
  │             │   <lzd-card>                │             │
  │             │   <lzd-modal>               │             │
  │             │   <lzd-toast>               │             │
  │             │   <lzd-i18n>                │             │
  │             └─────────────────────────────┘             │
  │                                                          │
  │     এক component lib — সব framework support              │
  │     CSS Custom Property দিয়ে theming (RTL-arabic, dark) │
  └──────────────────────────────────────────────────────────┘
```

### Theming RTL/LTR

```css
/* package design-tokens.css */
:root {
  --lzd-color-primary: #f57224;
  --lzd-direction: ltr;
  --lzd-font: 'SF Pro Display', sans-serif;
}
[dir="rtl"] {
  --lzd-direction: rtl;
}
[lang="bn-BD"] {
  --lzd-font: 'Noto Sans Bengali', sans-serif;
}
```

---

## ⚖️ React vs Web Components — কখন কোনটা?

| পরিস্থিতি | React/Vue | Web Components |
|----------|-----------|----------------|
| একটাই অ্যাপ, একটাই team | ✅ React | ❌ overhead বেশি |
| Multi-team, multi-framework | ❌ | ✅ universal |
| Design system shared with 3rd party | ❌ | ✅ |
| Heavy SPA | ✅ | ⚠️ extra glue lagbe |
| CMS-এ embed (WordPress, Webflow) | ❌ awkward | ✅ perfect |
| Server components / SSR | ✅ React | ⚠️ Lit SSR experimental |
| Small widget on legacy page | ❌ overkill | ✅ |

### Hybrid approach

বেশিরভাগ বড় কোম্পানি দুটো ব্যবহার করে:
- **App-level**: React/Vue (routing, data, SPA logic)
- **Design system primitives**: Web Components (button, input, modal)

---

## ⚠️ Pitfalls

```
  ❌ tag name-এ dash নেই → invalid → throw
  ❌ constructor-এ DOM access → element-এর attribute এখনো নেই
  ❌ Shadow DOM ছাড়া CSS leak হয় (host's CSS এসে ঢুকবে)
  ❌ Custom event-এ composed: false → shadow boundary cross করবে না
  ❌ FOUC: registration-এর আগে element render → :defined দিয়ে hide করুন
  ❌ Form participation নেই — formAssociated ও ElementInternals লাগবে
  ❌ React 18-এ non-string props প্যাচানো — wrapper বানান
  ❌ aria role/state স্বয়ংক্রিয় না — accessibility manually যোগ করতে হবে
  ❌ SEO: Shadow DOM-এর content Googlebot পারে কিন্তু কিছু crawler পারে না
```

### FOUC fix

```css
/* element define না হওয়া পর্যন্ত hide */
bkash-button:not(:defined) {
  visibility: hidden;
}
```

```javascript
await customElements.whenDefined('bkash-button');
// SSR/dynamic load-এর জন্য
```

### Form-associated custom element

```javascript
class BkashInput extends HTMLElement {
  static formAssociated = true;

  constructor() {
    super();
    this._internals = this.attachInternals();
  }

  set value(v) {
    this._value = v;
    this._internals.setFormValue(v);
  }
  get value() { return this._value; }
}
```

ElementInternals API দিয়ে form, validation, ARIA সবই participate করানো যায়।

---

## ✅ যেখানে ব্যবহার করবেন

- **Cross-framework design system** (Daraz Group, Pathao Group)
- **Embeddable widget** (live chat, payment button)
- **Browser extension** content script
- **Static site (legacy)**-এ rich UI inject
- **Long-lived enterprise app** যেখানে framework migration হবে

## ❌ যেখানে এড়ান

- ছোট, single-framework SPA
- SSR-heavy SEO অ্যাপ (Lit SSR এখনো immature)
- Form-heavy app (form participation extra কাজ)
- Real-time collaborative app যেখানে state বেশি

---

## 📋 সারসংক্ষেপ

```
  Web Components — ৪টি স্তম্ভ
  ════════════════════════════════════════════════
  ① Custom Elements    →  নিজের HTML tag (lifecycle)
  ② Shadow DOM         →  CSS/DOM encapsulation
  ③ HTML Templates     →  reusable inert markup + slots
  ④ ES Modules         →  modular distribution

  ব্যবহার:
  • plain JS → vanilla pattern
  • Production → Lit (5KB, declarative)
  • Theming → CSS custom properties
  • Communication → Custom Events (composed: true)
```

Web Components ফ্রেমওয়ার্ক war-এর বাইরে দাঁড়িয়ে থাকা **browser-native standard**। যখন
আপনার component বছরের পর বছর — এমনকি React/Vue অপ্রচলিত হয়ে গেলেও — কাজ করতে হবে, তখন
এটাই সবচেয়ে নিরাপদ choice।
