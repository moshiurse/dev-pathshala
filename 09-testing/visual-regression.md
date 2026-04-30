# 🖼️ Visual Regression Testing — UI পরিবর্তন স্বয়ংক্রিয়ভাবে ধরা

## 📖 সংজ্ঞা ও মূল ধারণা

**Visual Regression Testing (VRT)** হলো একটি automated testing কৌশল যেখানে UI screenshot বা rendered DOM স্ক্রিনশট baseline এর সাথে compare করে unintended visual change detect করা হয়। Functional test pass করেও UI broken হতে পারে — button overlap, text overflow, color drift, font missing, responsive break — এগুলো শুধু চোখে দেখা যায়। VRT এই gap পূরণ করে।

```
Developer commits CSS change
            │
            ▼
   ┌────────────────┐
   │  CI build & run │
   │ E2E + Storybook │
   └────────┬───────┘
            │ capture screenshots
            ▼
   ┌────────────────┐
   │  Compare with   │
   │  baseline images│
   └────────┬───────┘
            │
       ┌────┴────┐
       │ Diff?   │
       ├────┬────┤
       NO   YES
        │    │
        ▼    ▼
      Pass  Reviewer-এ pixel-diff পাঠানো হয়:
            "Approve" → new baseline
            "Reject"  → bug, fail build
```

---

## 🎯 কেন প্রয়োজন

### Functional test যা ধরতে পারে না
- ✅ "Login button click works" — Cypress pass
- ❌ "Login button overlap with logo on iPhone SE" — Cypress assertion-এ ধরা পড়বে না

### Real-world UI bug examples (BD context)
- **Pathao app:** নতুন BDT formatter "৳1,000.00" pricing card overflow
- **Daraz product card:** Bangla long text "মাইক্রোসফ্ট সারফেস প্রো ৯ ইন্টেল ই্ভো প্ল্যাটফর্ম" নতুন font-এ truncation broken
- **bKash dashboard:** Dark mode toggle introduce — primary CTA invisible
- **Foodpanda menu:** Image lazy-load placeholder color scheme regression
- **Chaldal cart:** Quantity stepper button rounded → square accidentally

প্রতিটা defect manual QA-এ ধরা পড়ার আগে user-এ পৌঁছায়। VRT prod-এ যাওয়ার আগে CI-তেই ধরে।

---

## 🔬 Pixel-diff vs Perceptual diff

### Pixel-diff (naive)
প্রতিটা pixel-এর RGB compare; এক pixel ভিন্ন → fail।

**সমস্যা:**
- Anti-aliasing-এ subtle pixel difference (browser version)
- Font rendering OS-ভিন্ন (macOS subpixel vs Linux freetype)
- Animation/spinner frame-এ আলাদা
- Date/time dynamic content

### Perceptual diff (smarter)
Human vision model ব্যবহার (luminance, contrast, ssim, dssim)। ছোট difference (e.g., 0.1% area) ignore।

```
Tolerance options:
├─ pixelDiff: pixel-by-pixel exact (strictest)
├─ pixelMatchJS: with anti-aliasing detection
├─ SSIM (Structural Similarity): perceptual
├─ Looks-Same / odiff: handles rendering variance
└─ Resemble.js: ignoreColors, ignoreAntialiasing options
```

**Best practice:** Threshold 0.1-0.2% pixel difference allow; large region change = real bug।

---

## 🛠️ Tools Overview

| Tool | Type | Hosting | Best for |
|------|------|---------|----------|
| **Percy** (BrowserStack) | Page-level + Component | Cloud (paid) | Full-app E2E + cross-browser |
| **Chromatic** | Storybook component | Cloud (free tier) | Component library, design system |
| **BackstopJS** | Page-level | Self-hosted (free) | Self-hosted, simple CI |
| **Playwright snapshot** | Page-level | Self-hosted | Already using Playwright |
| **Cypress + Percy / @cypress/snapshot** | Page-level | Mix | Cypress-based suites |
| **Storybook + Loki** | Component | Self-hosted | Storybook without Chromatic cost |
| **Reg-Suit** | Generic | Self-hosted (S3 backend) | Custom workflow, OSS |
| **Applitools Eyes** | AI-powered | Cloud (enterprise) | Cross-browser, AI-driven |

---

## 💻 Playwright Snapshot Testing

```javascript
// tests/visual/product-page.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Daraz Product Page Visual', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('https://www.daraz.com.bd/products/iphone-15-i12345/');
    await page.waitForLoadState('networkidle');
    
    // 🔥 Stabilize dynamic content
    await page.addStyleTag({
      content: `
        *, *::before, *::after {
          animation-duration: 0s !important;
          transition-duration: 0s !important;
        }
        [data-testid="countdown"] { visibility: hidden !important; }
      `,
    });
  });

  test('product page — desktop', async ({ page }) => {
    await page.setViewportSize({ width: 1440, height: 900 });
    await expect(page).toHaveScreenshot('product-desktop.png', {
      fullPage: true,
      maxDiffPixelRatio: 0.005,    // 0.5% tolerance
      animations: 'disabled',
      mask: [
        page.locator('[data-testid="ad-banner"]'),
        page.locator('[data-testid="recently-viewed"]'),
      ],
    });
  });

  test('product page — mobile (iPhone 12)', async ({ page }) => {
    await page.setViewportSize({ width: 390, height: 844 });
    await expect(page).toHaveScreenshot('product-mobile.png', { fullPage: true });
  });

  test('add to cart button — focus state', async ({ page }) => {
    const btn = page.locator('[data-testid="add-to-cart"]');
    await btn.focus();
    await expect(btn).toHaveScreenshot('cta-focus.png');
  });

  test('Bangla locale rendering', async ({ page }) => {
    await page.goto('?lang=bn');
    await expect(page.locator('.product-title')).toHaveScreenshot('title-bn.png');
  });
});
```

```bash
# First run — generates baseline
npx playwright test --update-snapshots

# CI run — compares
npx playwright test
```

### Cross-browser/cross-device matrix

```javascript
// playwright.config.ts
export default {
  projects: [
    { name: 'chromium-desktop', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox-desktop',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit-desktop',   use: { ...devices['Desktop Safari'] } },
    { name: 'iphone-12',        use: { ...devices['iPhone 12'] } },
    { name: 'pixel-5',          use: { ...devices['Pixel 5'] } },
    { name: 'galaxy-s9',        use: { ...devices['Galaxy S9+'] } },
    { name: 'ipad-mini',        use: { ...devices['iPad Mini'] } },
  ],
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01,
      threshold: 0.2,             // perceptual sensitivity
    },
  },
};
```

Snapshots save হয় `tests/visual/__screenshots__/<browser>/<test>.png`।

---

## 💻 BackstopJS (PHP/Laravel app-এ popular)

```javascript
// backstop.json
{
  "id": "pathao_visual",
  "viewports": [
    { "label": "mobile",  "width":  375, "height": 667 },
    { "label": "tablet",  "width":  768, "height": 1024 },
    { "label": "desktop", "width": 1440, "height": 900 }
  ],
  "scenarios": [
    {
      "label": "Homepage",
      "url": "https://pathao.com",
      "delay": 1000,
      "removeSelectors": [".cookie-banner", "[data-dynamic]"],
      "hideSelectors":   [".live-counter"],
      "misMatchThreshold": 0.1
    },
    {
      "label": "Login Modal",
      "url": "https://pathao.com",
      "clickSelector": "[data-testid='login-btn']",
      "postInteractionWait": 500,
      "selectors": [".modal"]
    },
    {
      "label": "Ride booking flow",
      "url": "https://pathao.com/ride",
      "readyEvent": "appReady",
      "onReadyScript": "puppet/login.js"
    }
  ],
  "engine": "puppeteer",
  "report": ["browser", "CI"],
  "asyncCaptureLimit": 5,
  "asyncCompareLimit": 50,
  "paths": {
    "bitmaps_reference": "backstop_data/reference",
    "bitmaps_test":      "backstop_data/test",
    "ci_report":         "backstop_data/ci_report",
    "html_report":       "backstop_data/html_report"
  }
}
```

```bash
npm install -g backstopjs
backstop reference        # baseline
backstop test             # compare
backstop approve          # accept changes as new baseline
```

---

## 💻 Storybook + Chromatic (Component-level)

```javascript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Daraz/Button',
  component: Button,
  parameters: {
    chromatic: {
      viewports: [375, 768, 1440],
      diffThreshold: 0.05,
      modes: {
        light: { theme: 'light' },
        dark:  { theme: 'dark' },
        bn:    { locale: 'bn' },
      },
    },
  },
};
export default meta;

export const Primary: StoryObj<typeof Button> = {
  args: { variant: 'primary', children: 'কিনুন' },
};

export const LongBanglaText: StoryObj<typeof Button> = {
  args: { children: 'ক্যাশ অন ডেলিভারিতে অর্ডার করুন' },
};

export const Disabled: StoryObj<typeof Button> = {
  args: { variant: 'primary', disabled: true, children: 'লোড হচ্ছে...' },
};
```

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on: pull_request
jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: npm ci
      - uses: chromaui/action@v11
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitOnceUploaded: true
          onlyChanged: true     # turbosnap: শুধু changed component
```

PR-এ Chromatic UI review link comment করে; reviewer pixel-diff দেখে accept/reject।

---

## 💻 Cypress + Percy

```javascript
// cypress/e2e/checkout.cy.ts
describe('Foodpanda checkout VRT', () => {
  it('cart page', () => {
    cy.visit('/cart');
    cy.contains('৳').should('be.visible');
    cy.percySnapshot('Cart Page', {
      widths: [375, 768, 1280],
      minHeight: 1024,
    });
  });

  it('payment selection — bKash', () => {
    cy.visit('/checkout');
    cy.get('[data-test="payment-bkash"]').click();
    cy.percySnapshot('Payment - bKash selected');
  });
});
```

```bash
PERCY_TOKEN=xxx npx percy exec -- cypress run
```

---

## 🔧 Storybook + Loki (self-hosted, no Chromatic)

```javascript
// loki.config.json
{
  "configurations": {
    "chrome.laptop":  { "target": "chrome.docker", "width": 1366, "height": 768 },
    "chrome.iphone8": { "target": "chrome.docker", "preset": "iPhone 8" },
    "chrome.ipad":    { "target": "chrome.docker", "preset": "iPad" }
  },
  "diffingEngine": "looks-same",
  "looksSame": { "tolerance": 5, "antialiasingTolerance": 4 }
}
```

```bash
loki update    # baseline
loki test      # compare
```

---

## ⚠️ Flaky Tests — Stabilization Tricks

VRT-এর সবচেয়ে বড় শত্রু — flakiness। নিচের গুলো না করলে CI false-positive-এ ভরে যায়।

### 1. Disable animations
```css
*, *::before, *::after {
  animation-duration: 0s !important;
  animation-delay: 0s !important;
  transition-duration: 0s !important;
  transition-delay: 0s !important;
}
```

### 2. Wait for network idle
```javascript
await page.waitForLoadState('networkidle');
await page.waitForFunction(() => document.fonts.ready);
```

### 3. Mock dynamic data
- Date/time: freeze (`MockDate.set('2025-01-01')`)
- Random data: seeded faker
- Live counter, "Posted 5 minutes ago": mock or hide

### 4. Wait for fonts loaded
```javascript
await page.evaluate(() => document.fonts.ready);
```
বাংলা font (Hind Siliguri, Tiro Bangla) load না হলে fallback rendering — VRT fail।

### 5. Mask volatile region
```javascript
await expect(page).toHaveScreenshot({
  mask: [page.locator('.timestamp'), page.locator('.live-count')],
});
```

### 6. Stable image rendering
- Always WebP/optimized format
- Pre-load via `image.decode()` await
- Avatar/random image-এ placeholder

### 7. Browser deterministic flags
```javascript
launchOptions: {
  args: [
    '--font-render-hinting=none',
    '--disable-skia-runtime-opts',
    '--force-color-profile=srgb',
  ],
}
```

### 8. Run in Docker (host font ভিন্নতা remove)
```dockerfile
FROM mcr.microsoft.com/playwright:v1.44.0-jammy
# all Linux fonts identical across machines
```

---

## 📊 Component vs Page-level Trade-offs

| Aspect | Component (Storybook + Chromatic) | Page (Playwright/BackstopJS) |
|--------|-----------------------------------|-------------------------------|
| Speed | দ্রুত (isolated render) | ধীর (full page load) |
| Coverage | Component states (hover, error, disabled) সহজ | Real integration |
| Flakiness | কম | বেশি (network, ads, dynamic) |
| Cost | Storybook maintenance | Less infra |
| When to use | Design system, reusable component | Critical user flow (checkout, login) |

> **Best practice:** দুটোই — Storybook-এ atomic component, Playwright-এ key user journey।

---

## 🚀 CI Integration (GitHub Actions)

```yaml
name: Visual Regression
on:
  pull_request:
    paths: ['src/**', '*.css', '*.scss']

jobs:
  vrt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci

      # Run in pinned Docker for font consistency
      - name: Playwright tests
        run: npx playwright test --reporter=html
        env:
          CI: true

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: visual-diff
          path: test-results/

      # PR comment with diff images (optional)
      - name: Post diff to PR
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const body = `❌ Visual regression detected. [View report](${process.env.RUN_URL})`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body,
            });
```

### Approval workflow
- PR-এ VRT fail → reviewer diff দেখে
- Intentional change → `npx playwright test --update-snapshots` → commit baseline
- Bug → fix code

### Branch baseline strategy
- `main` branch baseline = source of truth
- Feature branch tests against `main` baseline
- Merge হলে baseline auto-updated

---

## ✅ কখন VRT ব্যবহার করবেন

- Design system / component library
- Marketing/landing page (CSS regression critical)
- E-commerce product/checkout pages (revenue impact)
- Multi-locale app (বাংলা layout break সহজে হয়)
- Cross-browser/device support critical (BD-তে old Android, low-end devices)
- Frequent CSS refactor / Tailwind migration
- A11y theme switching (dark mode, high contrast)

## ❌ কখন এড়াবেন / সীমিত করবেন

- Highly dynamic content (real-time dashboard, news feed) — flaky
- A/B test variation overload
- Solo dev / খুব ছোট app — ROI low
- Server-rendered এ massive content variation
- প্রতিটি minor change-এ baseline update দরকার এমন rapid prototyping

---

## 🇧🇩 Bangladesh Real-world VRT Wins

- **Pathao app** (web admin): ২০২৪-এ Tailwind v3 → v4 migration-এ ১২+ component-এ subtle padding/border-radius regression — Chromatic CI-এ ধরা পড়ে production-এ যাওয়ার আগে।
- **Daraz BD:** Festival sale (১১.১১, ১২.১২) banner-এ ১০-১৫টা campaign UI; প্রতিটার জন্য Storybook story ও Chromatic VRT — designer manual review-এ ১ দিন; CI-এ ১৫ মিনিট।
- **bKash app web:** Dark mode rollout-এ contrast ratio ১৭টা component-এ broken — Storybook + Loki ধরেছে।
- **Foodpanda BD:** Vendor dashboard-এ Bangla menu name long text overflow — Playwright screenshot test catch।
- **Chaldal:** Reorder card button hover state regression visible only on Firefox; cross-browser VRT না থাকলে miss হতো।
- **Sheba.xyz:** Service category card responsive break iPad portrait-এ; BackstopJS catch।
- **Bangladesh Bank portal:** Mostly static, কিন্তু font upgrade (SolaimanLipi → Tiro Bangla) সব page-এ subtle break — VRT essential।
- **Pathao Pay Merchant Portal:** QR code rendering library upgrade-এ ১ pixel offset broke scanner alignment; pixel-diff catch।

---

## ⚠️ Common Pitfalls

1. **Anti-aliasing flakiness ignore** — every CI run-এ random failures; team eventually disables VRT।
2. **Baseline drift** — careless `--update-snapshots` দিয়ে bug-ই baseline হয়ে যাওয়া।
3. **Auto-approve all** — reviewer cursorily click "approve all changes" → regression slip।
4. **No font loading wait** — bangla font load before screenshot capture না হলে fallback render।
5. **Animation/transition not disabled** — capture mid-animation।
6. **Dynamic content (date, random ad)** mask না করা।
7. **Same baseline cross-browser/cross-OS** — Linux Chrome ও macOS Chrome ভিন্ন → CI ও local mismatch।
8. **Massive baseline repo** — git history bloat; LFS or external store (S3) consider।
9. **Test runs only on main** — feature branch-এ regression detect হয় না।
10. **No threshold tuning** — too tight = flaky, too loose = miss bug। Per-component tuning দরকার।
11. **Only desktop tested** — BD-তে ৭০% mobile traffic; mobile snapshot mandatory।
12. **VRT replaces accessibility/functional test** — VRT pixel দেখে, screen reader experience দেখে না।

---

## 📊 Tooling Decision Matrix

| Need | Recommendation |
|------|----------------|
| OSS, self-hosted, page-level | BackstopJS / Playwright snapshot |
| Storybook component + cloud | Chromatic |
| Storybook component + self-hosted | Loki |
| Already on Cypress | Percy + Cypress |
| AI-driven, enterprise | Applitools Eyes |
| Generic CI image diff with S3 | reg-suit |
| Playwright user → built-in | Playwright `toHaveScreenshot` |

---

## 📝 সারসংক্ষেপ

- **VRT = automated screenshot comparison** যা functional test-এর gap পূরণ করে — UI broken হলে ধরে।
- **Pixel-diff vs Perceptual diff:** Anti-aliasing, font, animation-এর কারণে pure pixel-diff flaky; SSIM/looks-same/odiff perceptual ভালো।
- **Tools:** Percy, Chromatic (cloud), BackstopJS, Playwright snapshot, Loki, reg-suit (self-hosted)।
- **Component-level (Storybook) + Page-level (Playwright)** — দুটোই দরকার।
- **Flakiness handling:** disable animations, font ready wait, mask dynamic content, Docker for OS consistency।
- **CI integration:** PR fail → reviewer diff approve/reject; baseline updated on merge।
- **Cross-browser/device matrix** essential — BD-তে mobile + low-end Android coverage critical।
- **BD wins:** Daraz festival UI, bKash dark mode, Pathao Tailwind migration — VRT-এর জন্যই production save।
- **Pitfalls:** baseline drift, auto-approve, font/animation flakiness, mobile skip — disciplined process required।
