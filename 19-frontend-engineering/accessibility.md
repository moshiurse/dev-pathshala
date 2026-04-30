# ♿ Accessibility (a11y) — সবার জন্য ওয়েব

## 📌 সংজ্ঞা

**Accessibility (a11y)** হলো এমনভাবে ওয়েব অ্যাপ্লিকেশন তৈরি করা, যাতে **প্রতিবন্ধী ব্যবহারকারী**সহ
সবাই সেটি ব্যবহার করতে পারেন — দৃষ্টিপ্রতিবন্ধী (visually impaired), শ্রবণপ্রতিবন্ধী
(hearing impaired), মোটর-সংক্রান্ত সমস্যা (motor impairment), জ্ঞানীয় প্রতিবন্ধকতা
(cognitive disability) — সবাই।

> "a11y" = a + ১১ অক্ষর + y → accessibility-এর সংক্ষিপ্ত রূপ।

বাংলাদেশে প্রায় **১.৬ কোটি** মানুষ কোনো না কোনোভাবে প্রতিবন্ধী (WHO/BBS)। bKash, Nagad,
Pathao, Daraz — যেকোনো জনপ্রিয় অ্যাপের ০.১% ইউজারও যদি দৃষ্টিপ্রতিবন্ধী হন, সেটা লক্ষ লক্ষ
মানুষ। অথচ বেশিরভাগ বাংলাদেশি অ্যাপ স্ক্রিন-রিডার দিয়ে অনুভবযোগ্য নয়।

---

## 🎯 কেন গুরুত্বপূর্ণ?

```
  Accessibility কেন? — ৪টি কারণ
  ════════════════════════════════════════════════════

  ① মানবিক (Ethical)
     প্রতিবন্ধী ব্যক্তিরও ডিজিটাল সেবা পাওয়ার অধিকার আছে
     bKash দিয়ে টাকা পাঠানো একজন অন্ধ ব্যক্তিরও দরকার

  ② আইনি (Legal)
     - ADA (USA), EAA (EU), AODA (Canada)
     - বাংলাদেশ: প্রতিবন্ধী ব্যক্তির অধিকার ও সুরক্ষা আইন ২০১৩
     - সরকারি সাইটে BTRC গাইডলাইন

  ③ ব্যবসায়িক (Business)
     - বড় বাজার (১৫% বিশ্বব্যাপী জনসংখ্যা)
     - SEO ভালো হয় (Google semantic HTML পছন্দ করে)
     - বয়স্ক ইউজার ও সাময়িক প্রতিবন্ধকতা (broken hand, low light)

  ④ গুণগত (Quality)
     a11y ভালো হলে → keyboard, voice, automation সবাই কাজ করে
     টেস্টেবলিটি বাড়ে, কোড গুণগত মান বাড়ে
```

---

## 🏛️ WCAG — Web Content Accessibility Guidelines

WCAG হলো W3C-র তৈরি accessibility-র আন্তর্জাতিক স্ট্যান্ডার্ড।

| ভার্সন | প্রকাশ | মূল ফোকাস |
|--------|--------|-----------|
| WCAG 2.0 | 2008 | Web content মূলনীতি |
| WCAG 2.1 | 2018 | Mobile, low vision, cognitive |
| WCAG 2.2 | 2023 | Focus visibility, dragging, target size |

### Conformance Level

```
  Level A   →  ন্যূনতম (legally required basics)
  Level AA  →  ইন্ডাস্ট্রি স্ট্যান্ডার্ড (most apps target this)
  Level AAA →  সর্বোচ্চ (specialized, gov sometimes)
```

### POUR — চারটি মূলনীতি

```
  ┌─────────────────────────────────────────────────────────┐
  │  P  →  Perceivable    (অনুভবযোগ্য)                       │
  │       তথ্য এমনভাবে দিতে হবে যাতে ইউজার অনুভব করতে পারে │
  │       উদাহরণ: ইমেজে alt text, ভিডিওতে caption          │
  │                                                          │
  │  O  →  Operable        (পরিচালনাযোগ্য)                   │
  │       কীবোর্ড দিয়ে সব কাজ করা যাবে, মাউস বাধ্যতামূলক না │
  │       উদাহরণ: Tab দিয়ে নেভিগেশন, Esc দিয়ে modal বন্ধ   │
  │                                                          │
  │  U  →  Understandable  (বোধগম্য)                         │
  │       কন্টেন্ট ও UI predictable হবে                     │
  │       উদাহরণ: পরিষ্কার error message, consistent nav   │
  │                                                          │
  │  R  →  Robust          (দৃঢ়)                            │
  │       বিভিন্ন assistive tech-এ চলবে                     │
  │       উদাহরণ: valid HTML, ARIA সঠিক ব্যবহার            │
  └─────────────────────────────────────────────────────────┘
```

---

## 🏷️ Semantic HTML — a11y-র ভিত্তি

**সবচেয়ে গুরুত্বপূর্ণ a11y নিয়ম:** যেখানে native HTML element আছে, সেখানে `<div>`/`<span>`
ব্যবহার করবেন না। semantic HTML browser ও screen reader-কে অনেক কিছু **বিনামূল্যে** দেয়।

```html
<!-- ❌ খারাপ — অর্থহীন div -->
<div class="button" onclick="submit()">পাঠান</div>

<!-- ✅ ভালো — semantic button -->
<button type="submit">পাঠান</button>
<!-- 
  স্বয়ংক্রিয়ভাবে পায়:
  - keyboard focusable
  - Enter / Space দিয়ে activate
  - screen reader-এ "button" announce
  - disabled state support
-->
```

### সঠিক semantic ট্যাগ

```html
<header>     <!-- পেজ হেডার -->
<nav>        <!-- নেভিগেশন -->
<main>       <!-- মূল কন্টেন্ট (পেজে একটিই) -->
<article>    <!-- স্বনির্ভর কন্টেন্ট (e.g. blog post) -->
<section>    <!-- কন্টেন্টের section (heading সহ) -->
<aside>      <!-- পার্শ্ব কন্টেন্ট -->
<footer>     <!-- ফুটার -->
<button>     <!-- ক্লিকযোগ্য বাটন -->
<a href>     <!-- লিংক (নেভিগেশনের জন্য) -->
<form>       <!-- ফর্ম -->
<label>      <!-- ফিল্ডের লেবেল -->
```

### Heading hierarchy

```html
<!-- ✅ সঠিক — হেডিং স্কিপ করবেন না -->
<h1>Daraz</h1>
  <h2>ইলেকট্রনিক্স</h2>
    <h3>মোবাইল ফোন</h3>
    <h3>ল্যাপটপ</h3>
  <h2>ফ্যাশন</h2>

<!-- ❌ খারাপ — h1 থেকে সরাসরি h4 -->
<h1>Daraz</h1>
<h4>মোবাইল ফোন</h4>
```

স্ক্রিন-রিডার ইউজার সাধারণত H কী চাপ দিয়ে heading-এর মাধ্যমে নেভিগেট করে। heading skip
করলে document outline ভেঙে পড়ে।

---

## 🎭 ARIA — Accessible Rich Internet Applications

**ARIA** হলো HTML attribute সেট যা element-এর role, state, properties বর্ণনা করে। যখন
native HTML যথেষ্ট নয় (e.g. custom dropdown), তখন ARIA ব্যবহার করতে হয়।

### ARIA-র ৫টি নিয়ম (W3C)

```
  ১. পারলে ARIA ব্যবহার করবেন না — native HTML ব্যবহার করুন
  ২. native semantics পরিবর্তন করবেন না (e.g. <button role="link">)
  ৩. সব interactive ARIA widget keyboard-accessible হতে হবে
  ৪. focusable element-কে role="presentation" বা aria-hidden করবেন না
  ৫. সব interactive element-এর accessible name থাকতে হবে
```

### Roles, States, Properties

```html
<!-- Role: element কী -->
<div role="button" tabindex="0">Click</div>
<div role="alert">Network error</div>
<div role="dialog" aria-modal="true">...</div>

<!-- State: পরিবর্তনশীল অবস্থা -->
<button aria-pressed="true">Bold</button>
<div aria-expanded="false">Menu</div>
<input aria-invalid="true" />
<div aria-busy="true">Loading...</div>

<!-- Property: স্থিতিশীল বৈশিষ্ট্য -->
<input aria-label="Search products" />
<input aria-labelledby="search-heading" />
<button aria-describedby="help-text">Submit</button>
<div role="region" aria-labelledby="cart-heading">
  <h2 id="cart-heading">আপনার কার্ট</h2>
</div>
```

### সাধারণ ARIA roles

| Role | কখন |
|------|-----|
| `button` | clickable widget যা button না |
| `dialog` | modal/popup |
| `alert` | জরুরি বার্তা (auto-announced) |
| `status` | অগ্রাধিকারহীন আপডেট |
| `tablist`, `tab`, `tabpanel` | ট্যাব UI |
| `menu`, `menuitem` | অ্যাপ্লিকেশন menu |
| `combobox`, `listbox`, `option` | autocomplete |
| `tree`, `treeitem` | hierarchical list |
| `progressbar` | progress indicator |

### aria-live — dynamic content

```html
<!-- bKash transaction status -->
<div aria-live="polite" id="status">
  <!-- JS দিয়ে আপডেট হলে স্ক্রিন রিডার পড়বে -->
</div>

<div aria-live="assertive" role="alert">
  <!-- জরুরি, সবকিছু থামিয়ে পড়বে -->
</div>
```

- `polite` — ইউজারের কাজ শেষে পড়বে
- `assertive` — এখনই পড়বে (শুধু critical)
- `off` — পড়বে না (default)

---

## ⌨️ Keyboard Navigation

**সব কিছু কীবোর্ড দিয়ে চালানো যাবে** — এটি WCAG-র Level A requirement।

### Keyboard interaction patterns

| কী | কাজ |
|----|-----|
| `Tab` | পরের focusable element |
| `Shift+Tab` | আগের focusable element |
| `Enter` | বাটন/লিংক activate |
| `Space` | বাটন activate, checkbox toggle |
| `Esc` | modal/dropdown বন্ধ |
| `↑/↓` | menu/list-এ navigation |
| `Home/End` | প্রথম/শেষ item |

### tabindex — কখন কী

```html
<!-- tabindex="0" — natural tab order-এ যোগ করে -->
<div tabindex="0" role="button">Custom button</div>

<!-- tabindex="-1" — programmatically focusable, tab-এ আসে না -->
<div tabindex="-1" id="error">Error</div>
<!-- JS দিয়ে: errorEl.focus() -->

<!-- tabindex="1+" — ❌ ব্যবহার করবেন না, natural order ভাঙে -->
```

### Focus visibility

```css
/* ❌ কখনো এটা করবেন না */
*:focus { outline: none; }

/* ✅ ভালো — focus-visible দিয়ে keyboard-only */
:focus-visible {
  outline: 3px solid #e2136e; /* bKash brand color */
  outline-offset: 2px;
}

/* mouse click-এ outline দেখাবে না, keyboard-এ দেখাবে */
:focus:not(:focus-visible) {
  outline: none;
}
```

### Skip link — Daraz-এর হোমপেজে

```html
<a href="#main" class="skip-link">মূল কন্টেন্টে যান</a>
<nav>
  <!-- ১০০টি ক্যাটাগরি লিংক -->
</nav>
<main id="main" tabindex="-1">
  ...
</main>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  padding: 8px;
  background: #000;
  color: #fff;
}
.skip-link:focus {
  top: 0;
}
</style>
```

কীবোর্ড ইউজার Tab চাপলেই skip link দেখা যায়, এক ক্লিকে main content-এ চলে যেতে পারে।

---

## 🎯 Focus Management — Modal উদাহরণ

Modal খুললে focus modal-এ যাবে, modal বন্ধ হলে focus আগের জায়গায় ফিরবে। এর মধ্যে focus
modal-এর বাইরে যাবে না — এটাকে **focus trap** বলে।

### Accessible Modal (vanilla JS)

```javascript
class AccessibleModal {
  constructor(modalEl) {
    this.modal = modalEl;
    this.previousFocus = null;
    this.focusableSelectors = [
      'a[href]',
      'button:not([disabled])',
      'input:not([disabled])',
      'select:not([disabled])',
      'textarea:not([disabled])',
      '[tabindex]:not([tabindex="-1"])'
    ].join(',');
  }

  open() {
    this.previousFocus = document.activeElement;
    this.modal.removeAttribute('hidden');
    this.modal.setAttribute('aria-modal', 'true');
    this.modal.setAttribute('role', 'dialog');

    // Background-এর সব কিছু hide
    document.querySelectorAll('main, header, footer').forEach(el => {
      el.setAttribute('aria-hidden', 'true');
      el.setAttribute('inert', '');
    });

    // প্রথম focusable element-এ focus
    const firstFocusable = this.modal.querySelector(this.focusableSelectors);
    firstFocusable?.focus();

    document.addEventListener('keydown', this.handleKeydown);
  }

  close() {
    this.modal.setAttribute('hidden', '');
    document.querySelectorAll('main, header, footer').forEach(el => {
      el.removeAttribute('aria-hidden');
      el.removeAttribute('inert');
    });
    document.removeEventListener('keydown', this.handleKeydown);

    // আগের focus-এ ফিরে যান
    this.previousFocus?.focus();
  }

  handleKeydown = (e) => {
    if (e.key === 'Escape') {
      this.close();
      return;
    }

    if (e.key !== 'Tab') return;

    // Focus trap
    const focusable = this.modal.querySelectorAll(this.focusableSelectors);
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
  };
}
```

### Modal HTML

```html
<button id="open-btn">পেমেন্ট নিশ্চিত করুন</button>

<div
  id="payment-modal"
  hidden
  aria-labelledby="modal-title"
  aria-describedby="modal-desc"
>
  <h2 id="modal-title">পেমেন্ট নিশ্চিতকরণ</h2>
  <p id="modal-desc">আপনি কি ৫০০ টাকা পাঠাতে চান?</p>
  <button id="confirm">হ্যাঁ, পাঠান</button>
  <button id="cancel">বাতিল</button>
</div>
```

> 💡 আধুনিক ব্রাউজারে `<dialog>` element ব্যবহার করলে অনেক কিছু free পাওয়া যায়:
> `dialog.showModal()` automatic focus trap, Esc handling, backdrop দেয়।

---

## 📝 Accessible Form

ফর্ম a11y-র সবচেয়ে গুরুত্বপূর্ণ অংশ — সবাই ফর্ম পূরণ করে (login, payment, order)।

```html
<form novalidate id="signup-form">
  <!-- প্রতিটি input-এর label থাকতে হবে -->
  <div class="field">
    <label for="phone">মোবাইল নম্বর <span aria-hidden="true">*</span></label>
    <input
      id="phone"
      name="phone"
      type="tel"
      required
      autocomplete="tel"
      aria-required="true"
      aria-describedby="phone-help phone-error"
      inputmode="numeric"
      pattern="01[0-9]{9}"
    />
    <small id="phone-help">উদাহরণ: 01712345678</small>
    <span id="phone-error" class="error" role="alert" aria-live="polite"></span>
  </div>

  <!-- ভুল হলে clearly জানাতে হবে -->
  <div class="field">
    <label for="pin">PIN নম্বর</label>
    <input
      id="pin"
      type="password"
      autocomplete="current-password"
      aria-invalid="false"
      aria-describedby="pin-error"
    />
    <span id="pin-error" class="error" aria-live="assertive"></span>
  </div>

  <!-- Fieldset দিয়ে related field group -->
  <fieldset>
    <legend>পেমেন্ট পদ্ধতি</legend>
    <label><input type="radio" name="payment" value="bkash"> bKash</label>
    <label><input type="radio" name="payment" value="nagad"> Nagad</label>
    <label><input type="radio" name="payment" value="cod"> ক্যাশ অন ডেলিভারি</label>
  </fieldset>

  <button type="submit">নিবন্ধন করুন</button>
</form>
```

### JavaScript validation (accessible)

```javascript
const form = document.getElementById('signup-form');

form.addEventListener('submit', (e) => {
  e.preventDefault();
  const phone = form.phone;
  const phoneError = document.getElementById('phone-error');

  if (!/^01[0-9]{9}$/.test(phone.value)) {
    phone.setAttribute('aria-invalid', 'true');
    phoneError.textContent = 'বৈধ বাংলাদেশি মোবাইল নম্বর দিন';
    phone.focus(); // error হলে সেই field-এ focus নিয়ে যান
    return;
  }

  phone.setAttribute('aria-invalid', 'false');
  phoneError.textContent = '';
  // submit ...
});
```

### ফর্মে যা মনে রাখবেন

| ✅ করুন | ❌ করবেন না |
|---------|------------|
| প্রতিটি input-এ `<label>` | placeholder-কে label হিসেবে |
| `autocomplete` attribute | autocomplete বন্ধ করা |
| Error-এ `aria-invalid` + message | শুধু রঙ দিয়ে error |
| Error হলে field-এ focus | শুধু banner দিয়ে error |
| `inputmode` (mobile keyboard) | type="text" সবকিছুতে |

---

## 🎨 Color Contrast

দৃষ্টিপ্রতিবন্ধী, বয়স্ক ব্যক্তি, বা রোদে মোবাইল ব্যবহারকারীদের জন্য contrast গুরুত্বপূর্ণ।

### WCAG AA contrast ratio

| টেক্সট সাইজ | ন্যূনতম ratio |
|-------------|---------------|
| Normal (< 18pt) | **4.5:1** |
| Large (≥ 18pt বা ১৪pt bold) | **3:1** |
| UI components (button border) | **3:1** |

```css
/* ❌ খারাপ — ratio 2.3:1 */
.text { color: #999; background: #fff; }

/* ✅ ভালো — ratio 4.6:1 */
.text { color: #767676; background: #fff; }

/* ✅ bKash brand-এ accessible */
.btn-primary {
  background: #e2136e;  /* bKash pink */
  color: #fff;          /* contrast 5.8:1 */
}
```

**টুল:** Chrome DevTools → Inspect → Color picker → contrast ratio দেখায়।

### রঙের উপর শুধু নির্ভর করবেন না

```html
<!-- ❌ খারাপ — শুধু রঙ দিয়ে error -->
<input style="border: red"> 

<!-- ✅ ভালো — রঙ + আইকন + টেক্সট -->
<input aria-invalid="true" style="border: red">
<span class="error">⚠️ ভুল নম্বর</span>
```

---

## 🎬 prefers-reduced-motion

কিছু ইউজারের vestibular disorder (vertigo) আছে — animation দেখলে অসুস্থ লাগে।

```css
.banner {
  animation: slide 0.5s ease;
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

```javascript
// JS-এও চেক করুন
const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
if (!prefersReduced) {
  startCarousel();
}
```

অন্যান্য preference media query: `prefers-color-scheme`, `prefers-contrast`,
`prefers-reduced-transparency`.

---

## 🔊 Screen Reader

Screen reader স্ক্রিনের কন্টেন্ট পড়ে শোনায়। বিভিন্ন প্ল্যাটফর্মে বিভিন্ন reader:

| OS | Screen Reader | Free? |
|----|---------------|-------|
| Windows | **NVDA** | ✅ Free, popular |
| Windows | JAWS | ❌ Paid, enterprise |
| macOS/iOS | **VoiceOver** | ✅ Built-in |
| Android | **TalkBack** | ✅ Built-in |
| Linux | Orca | ✅ Free |

### বাংলা ভাষায় screen reader

NVDA + eSpeak NG-তে বাংলা ভয়েস আছে। iOS VoiceOver-এ Bangla (Bangladesh) সাপোর্ট। অ্যাপের
HTML lang সঠিক হলে সঠিক উচ্চারণ হয়:

```html
<html lang="bn-BD">
<!-- বা multi-language পেজে -->
<p lang="en">Welcome</p>
<p lang="bn-BD">স্বাগতম</p>
```

### NVDA দিয়ে টেস্ট (Windows)

```
  Ctrl                → Stop reading
  Insert + Down       → Read all
  H / Shift+H         → next/prev heading
  K / Shift+K         → next/prev link
  F / Shift+F         → next/prev form field
  B / Shift+B         → next/prev button
  ;                   → ARIA landmarks list
  Insert + F7         → Elements list (links/headings/landmarks)
```

প্রতিটি ফ্রন্টএন্ড ডেভেলপারের NVDA ব্যবহার শেখা উচিত।

---

## 🧪 Accessibility Testing

### Automated tools

```bash
# axe-core (Deque) — সবচেয়ে নির্ভরযোগ্য
npm install --save-dev @axe-core/playwright @axe-core/react

# eslint-plugin-jsx-a11y — কোড-লেভেল চেক
npm install --save-dev eslint-plugin-jsx-a11y
```

### Jest + axe

```javascript
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('Checkout form should have no a11y violations', async () => {
  const { container } = render(<CheckoutForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### Playwright + axe

```javascript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('Daraz product page a11y', async ({ page }) => {
  await page.goto('https://daraz.com.bd/products/iphone-15');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

### Lighthouse a11y audit

Chrome DevTools → Lighthouse → Accessibility → score (0–100)। কিন্তু Lighthouse শুধু ৩০%
সমস্যা ধরে — manual testing লাগবেই।

### Manual testing checklist

```
  □ keyboard-only দিয়ে পুরো অ্যাপ ব্যবহার করুন (mouse unplug করে)
  □ Tab ক্রম logical
  □ focus সবসময় visible
  □ NVDA/VoiceOver দিয়ে পড়ে দেখুন
  □ ব্রাউজার zoom 200% — পেজ ভাঙে?
  □ contrast ratio চেক
  □ slow 3G + screen reader একসাথে
  □ মোবাইল touch target ≥ 44x44 px
  □ form error চেক
  □ Color blindness simulator (Chrome DevTools)
```

> ⚠️ Automated tool সর্বোচ্চ ৩০-৪০% সমস্যা ধরে। বাকি ৬০-৭০% শুধু **manual** testing-এ ধরা পড়ে।

---

## 🇧🇩 বাংলাদেশ কেস স্টাডি — bKash a11y

bKash বাংলাদেশে ৭ কোটি+ ব্যবহারকারী। যদি ১% দৃষ্টিপ্রতিবন্ধী হন — সেটা ৭ লক্ষ মানুষ।

### সমস্যার উদাহরণ

```
  ❌ বর্তমানে যা দেখা যায়:
  ─────────────────────────
  • PIN keypad শুধু ছবি (alt text নেই)
  • "Send Money" বাটনে role/label নেই — শুধু div
  • OTP input-এ autocomplete="one-time-code" নেই
  • Transaction list-এ heading hierarchy ভাঙা
  • Modal-এ focus trap নেই — Tab চাপলে পেছনের পেজে focus যায়
  • Color-only error indication
  • bn-BD lang attribute অনুপস্থিত — screen reader ভুল উচ্চারণ
```

### সমাধান

```html
<html lang="bn-BD">
<head>
  <title>bKash — সেন্ড মানি</title>
</head>
<body>
  <header>
    <h1>bKash</h1>
    <nav aria-label="মূল মেনু">...</nav>
  </header>

  <main>
    <h2>টাকা পাঠান</h2>
    <form>
      <label for="recipient">প্রাপকের নম্বর</label>
      <input
        id="recipient"
        type="tel"
        autocomplete="tel"
        inputmode="numeric"
        aria-describedby="recipient-help"
      />
      <small id="recipient-help">১১ ডিজিটের bKash নম্বর</small>

      <label for="amount">পরিমাণ (টাকা)</label>
      <input id="amount" type="number" min="10" max="25000" />

      <fieldset>
        <legend>PIN দিন</legend>
        <input type="password" autocomplete="current-password"
               aria-label="৫ ডিজিটের PIN" />
      </fieldset>

      <button type="submit">পাঠান</button>
    </form>

    <div role="status" aria-live="polite" id="tx-status"></div>
  </main>
</body>
</html>
```

### Voice + Haptic feedback

বাংলাদেশে অনেক বয়স্ক ও কম-শিক্ষিত ইউজার আছেন। শুধু visual UI যথেষ্ট না:

```javascript
// সাকসেসে haptic + voice
async function onTransactionSuccess(amount, recipient) {
  // Haptic (mobile)
  if ('vibrate' in navigator) navigator.vibrate([100, 50, 100]);

  // ARIA live region আপডেট
  document.getElementById('tx-status').textContent =
    `${amount} টাকা ${recipient} কে সফলভাবে পাঠানো হয়েছে`;

  // Speech synthesis (optional, ইউজারের কনফিগারে)
  if (userSettings.voiceFeedback) {
    const utter = new SpeechSynthesisUtterance(
      `${amount} টাকা পাঠানো হয়েছে`
    );
    utter.lang = 'bn-BD';
    speechSynthesis.speak(utter);
  }
}
```

---

## ⚠️ সাধারণ ভুল (Pitfalls)

```
  ❌ <div onclick> বাটন হিসেবে — keyboard ও screen reader কাজ করবে না
  ❌ outline: none — keyboard ইউজার হারিয়ে যাবেন
  ❌ aria-label="click here" — context নেই
  ❌ alt="image" — অর্থহীন alt
  ❌ <h1>...<h4> — heading skip
  ❌ "form is invalid" - কোন field, কী সমস্যা বলেননি
  ❌ placeholder = label — placeholder মানে hint, label না
  ❌ tabindex="5" — natural order ভাঙে
  ❌ aria-hidden="true" focusable element-এ — keyboard trap
  ❌ শুধু color দিয়ে info — color-blind ইউজার বুঝবেন না
  ❌ auto-playing video/carousel — vestibular issue
  ❌ Toast notification 2s-এ চলে যায় — slow reader পড়তেই পারে না
```

---

## ✅ যখন বিশেষ মনোযোগ দিতে হবে

- **Public-facing app** (Daraz, Pathao, bKash) — আইনগত ও ব্যবসায়িক প্রয়োজন
- **Government সাইট** — universal access বাধ্যতামূলক
- **Banking/Fintech** — visually impaired ইউজার অর্থ লেনদেন করেন
- **Education platform** — different ability ইউজার
- **B2B SaaS enterprise client** — বড় কোম্পানি accessibility চায়

## ❌ যেখানে full WCAG AAA দরকার নেই

- Internal admin panel (employees only, controlled access)
- Personal blog (still target AA)
- POC/MVP — কিন্তু basic semantic HTML ছাড়া যাবে না

---

## 📋 সারসংক্ষেপ

```
  Accessibility — ১০ সোনালী নিয়ম
  ════════════════════════════════════════════════

  ১. Semantic HTML (button, nav, main, h1-h6)
  ২. সব image-এ অর্থপূর্ণ alt
  ৩. সব input-এ label
  ৪. সব কাজ keyboard দিয়ে চলবে
  ৫. focus সবসময় visible
  ৬. Color contrast ≥ 4.5:1
  ৭. Modal-এ focus trap + Esc
  ৮. Error message clear, field-এ focus
  ৯. ARIA শুধু যখন native HTML যথেষ্ট নয়
  ১০. NVDA/VoiceOver দিয়ে নিজে টেস্ট করুন
```

a11y শুধু "compliance" না — এটা **মানবিক ডিজাইন**। আপনার তৈরি অ্যাপ যেন বাংলাদেশের
প্রতিটি মানুষ — দৃষ্টিপ্রতিবন্ধী, বয়স্ক, কম-শিক্ষিত — সবাই ব্যবহার করতে পারে। সেটাই
ভালো ইঞ্জিনিয়ারিং।
