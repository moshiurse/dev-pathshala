# 🌐 E2E টেস্টিং (End-to-End Testing) — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা

**End-to-End (E2E) টেস্টিং** হলো এমন একটি সফটওয়্যার টেস্টিং পদ্ধতি যেখানে পুরো অ্যাপ্লিকেশনকে শুরু থেকে শেষ পর্যন্ত একজন **বাস্তব ব্যবহারকারীর দৃষ্টিকোণ** থেকে পরীক্ষা করা হয়। এটি শুধু একটি ফাংশন বা একটি কম্পোনেন্ট নয় — বরং সম্পূর্ণ ব্যবসায়িক প্রবাহ (business flow) যাচাই করে।

### E2E টেস্টিং কেন দরকার?

```
ইউনিট টেস্ট বলে:    "প্রতিটি ইট ঠিক আছে"
ইন্টিগ্রেশন টেস্ট বলে: "দেয়ালগুলো ঠিকমতো দাঁড়িয়ে আছে"
E2E টেস্ট বলে:        "পুরো বাড়িতে মানুষ বাস করতে পারছে"
```

E2E টেস্ট নিশ্চিত করে যে:
- ইউজার সফলভাবে রেজিস্ট্রেশন ও লগইন করতে পারছে
- প্রোডাক্ট কার্টে যোগ করে চেকআউট সম্পন্ন করতে পারছে
- বিকাশ/নগদ পেমেন্ট গেটওয়ে সঠিকভাবে কাজ করছে
- ফর্ম ভ্যালিডেশন বাংলা টেক্সট সাপোর্ট করছে

---

## 📊 টেস্টিং পিরামিডে E2E-এর অবস্থান

```
                    ╱╲
                   ╱  ╲
                  ╱ E2E╲          ← কম সংখ্যক, ধীর, ব্যয়বহুল
                 ╱ Tests ╲          কিন্তু সর্বোচ্চ আত্মবিশ্বাস
                ╱──────────╲
               ╱ Integration╲     ← মাঝারি সংখ্যক
              ╱    Tests      ╲     API, DB, সার্ভিস সংযোগ পরীক্ষা
             ╱────────────────╲
            ╱   Unit Tests      ╲  ← সর্বাধিক সংখ্যক, দ্রুত, সস্তা
           ╱                      ╲   একক ফাংশন/মডিউল পরীক্ষা
          ╱────────────────────────╲

    দ্রুততা    ◄──────────────────► ধীরতা
    কম খরচ    ◄──────────────────► বেশি খরচ
    কম আত্মবিশ্বাস ◄────────────► বেশি আত্মবিশ্বাস
```

### টেস্টিং ট্রফি (Kent C. Dodds মডেল)

```
         ╭─────────╮
         │  E2E    │  ← অল্প কিন্তু গুরুত্বপূর্ণ
         ├─────────┤
         │Integration│ ← সবচেয়ে বেশি ফোকাস
         │  Tests   │
         ├─────────┤
         │  Unit   │  ← দ্রুত ফিডব্যাক
         ├─────────┤
         │ Static  │  ← TypeScript, ESLint
         ╰─────────╯
```

> **মূলনীতি**: E2E টেস্ট কম লিখুন, কিন্তু সবচেয়ে গুরুত্বপূর্ণ ব্যবসায়িক প্রবাহগুলো কভার করুন।

---

## 💻 Tools Deep Dive

### ১. Cypress — আধুনিক E2E টেস্টিং ফ্রেমওয়ার্ক

#### সেটআপ

```bash
# প্রজেক্টে Cypress ইনস্টল করুন
npm init -y
npm install cypress --save-dev

# Cypress চালু করুন (GUI মোড)
npx cypress open

# হেডলেস মোড (CI-এর জন্য)
npx cypress run
```

#### প্রজেক্ট স্ট্রাকচার

```
cypress/
├── e2e/                    # টেস্ট ফাইল
│   ├── auth/
│   │   ├── login.cy.js
│   │   └── register.cy.js
│   ├── checkout/
│   │   └── purchase.cy.js
│   └── bkash/
│       └── payment.cy.js
├── fixtures/               # টেস্ট ডেটা (JSON)
│   ├── users.json
│   └── products.json
├── support/
│   ├── commands.js         # কাস্টম কমান্ড
│   └── e2e.js             # গ্লোবাল কনফিগ
└── downloads/
cypress.config.js           # মূল কনফিগারেশন
```

#### `cypress.config.js` কনফিগারেশন

```javascript
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    viewportWidth: 1280,
    viewportHeight: 720,
    defaultCommandTimeout: 10000,
    requestTimeout: 15000,
    responseTimeout: 30000,
    video: true,
    screenshotOnRunFailure: true,
    retries: {
      runMode: 2,      // CI-তে ২ বার retry
      openMode: 0       // লোকাল GUI-তে retry নেই
    },
    env: {
      apiUrl: 'http://localhost:8000/api',
      bkashSandbox: 'https://sandbox.bka.sh/v1.2.0-beta'
    },
    setupNodeEvents(on, config) {
      // টেস্ট ডেটা সিডিং
      on('task', {
        seedDatabase() {
          // ডেটাবেজে টেস্ট ডেটা ঢোকানো
          return null;
        },
        clearDatabase() {
          return null;
        }
      });
      return config;
    }
  }
});
```

#### মৌলিক Cypress কমান্ডসমূহ

```javascript
// cy.visit — পেজে যাওয়া
cy.visit('/products');
cy.visit('https://daraz.com.bd');

// cy.get — এলিমেন্ট সিলেক্ট করা (CSS selector)
cy.get('#login-btn');
cy.get('[data-testid="cart-icon"]');
cy.get('.product-card').first();

// cy.contains — টেক্সট দিয়ে খুঁজে পাওয়া
cy.contains('কার্টে যোগ করুন');
cy.contains('button', 'সাবমিট');

// cy.click — ক্লিক করা
cy.get('#submit').click();
cy.get('.dropdown').click({ force: true });

// cy.type — টাইপ করা
cy.get('#email').type('user@example.com');
cy.get('#phone').type('01712345678');
cy.get('#name').type('মোহাম্মদ আলী');  // বাংলা টেক্সট

// cy.select — ড্রপডাউন সিলেক্ট
cy.get('#district').select('ঢাকা');

// cy.check / cy.uncheck — চেকবক্স
cy.get('#terms').check();

// cy.clear — ইনপুট ক্লিয়ার
cy.get('#search').clear().type('নতুন প্রোডাক্ট');
```

#### Cypress Assertions

```javascript
// Should-based assertions
cy.get('.cart-count').should('have.text', '3');
cy.get('#total').should('contain', '৳');
cy.get('.error').should('be.visible');
cy.get('.success-modal').should('not.exist');
cy.url().should('include', '/dashboard');
cy.get('input').should('have.value', 'মোহাম্মদ');

// Chained assertions
cy.get('.product-card')
  .should('have.length', 12)
  .first()
  .should('contain', 'Samsung')
  .and('be.visible');

// Expect-based assertions (BDD)
cy.get('.price').then(($el) => {
  const price = parseFloat($el.text().replace('৳', ''));
  expect(price).to.be.greaterThan(0);
  expect(price).to.be.lessThan(100000);
});
```

#### API Intercepting — `cy.intercept()`

```javascript
// GET রিকোয়েস্ট ইন্টারসেপ্ট করে mock ডেটা দেওয়া
cy.intercept('GET', '/api/products', {
  fixture: 'products.json'
}).as('getProducts');

cy.visit('/products');
cy.wait('@getProducts');

// POST রিকোয়েস্ট মনিটর করা
cy.intercept('POST', '/api/orders').as('createOrder');

cy.get('#checkout-btn').click();
cy.wait('@createOrder').then((interception) => {
  expect(interception.response.statusCode).to.equal(201);
  expect(interception.request.body).to.have.property('items');
});

// নেটওয়ার্ক ত্রুটি সিমুলেট করা
cy.intercept('POST', '/api/payment/bkash', {
  statusCode: 500,
  body: { error: 'বিকাশ সার্ভার সমস্যা' }
}).as('bkashError');

// বিলম্বিত রেসপন্স সিমুলেট
cy.intercept('GET', '/api/search*', (req) => {
  req.reply({
    delay: 3000,
    body: { results: [] }
  });
}).as('slowSearch');
```

#### Fixtures — টেস্ট ডেটা ব্যবস্থাপনা

```json
// cypress/fixtures/users.json
{
  "validUser": {
    "email": "test@example.com",
    "password": "SecurePass123!",
    "name": "টেস্ট ইউজার",
    "phone": "01712345678"
  },
  "invalidUser": {
    "email": "invalid-email",
    "password": "123"
  }
}
```

```javascript
// ফিক্সচার ব্যবহার
cy.fixture('users.json').then((users) => {
  cy.get('#email').type(users.validUser.email);
  cy.get('#password').type(users.validUser.password);
});
```

#### Custom Commands — পুনরায় ব্যবহারযোগ্য কমান্ড

```javascript
// cypress/support/commands.js

// লগইন কমান্ড — প্রতিটি টেস্টে UI লগইন এড়াতে API দিয়ে লগইন
Cypress.Commands.add('login', (email, password) => {
  cy.session([email, password], () => {
    cy.request('POST', '/api/auth/login', { email, password })
      .then((resp) => {
        window.localStorage.setItem('token', resp.body.token);
      });
  });
});

// বিকাশ পেমেন্ট সিমুলেট করার কমান্ড
Cypress.Commands.add('mockBkashPayment', (amount) => {
  cy.intercept('POST', '**/bkash/create', {
    statusCode: 200,
    body: {
      paymentID: 'TR001122334455',
      createTime: new Date().toISOString(),
      amount: amount,
      transactionStatus: 'Initiated'
    }
  });

  cy.intercept('POST', '**/bkash/execute', {
    statusCode: 200,
    body: {
      paymentID: 'TR001122334455',
      trxID: 'TRX9988776655',
      transactionStatus: 'Completed',
      amount: amount
    }
  });
});

// data-testid দিয়ে সহজে এলিমেন্ট খোঁজার কমান্ড
Cypress.Commands.add('getByTestId', (testId) => {
  return cy.get(`[data-testid="${testId}"]`);
});

// ব্যবহার
cy.login('admin@shop.com.bd', 'Admin123!');
cy.mockBkashPayment(1500);
cy.getByTestId('product-title').should('contain', 'Samsung Galaxy');
```

#### Cypress Cloud (Dashboard Service)

```javascript
// cypress.config.js
module.exports = defineConfig({
  projectId: 'abc123',
  e2e: {
    // ...
  }
});
```

```bash
# রেকর্ড সহ চালানো
npx cypress run --record --key YOUR_PROJECT_KEY

# প্যারালেল execution
npx cypress run --record --parallel --group "e2e-tests"
```

---

### ২. Playwright — Microsoft-এর ক্রস-ব্রাউজার E2E টুল

#### সেটআপ

```bash
# Playwright ইনস্টল
npm init playwright@latest

# নির্দিষ্ট ব্রাউজার ইনস্টল
npx playwright install chromium firefox webkit
```

#### `playwright.config.js`

```javascript
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './tests',
  timeout: 30000,
  expect: { timeout: 5000 },
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'results.xml' }]
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    locale: 'bn-BD',               // বাংলাদেশ লোকেল
    timezoneId: 'Asia/Dhaka'       // ঢাকা টাইমজোন
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    // মোবাইল ডিভাইস
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 13'] }
    }
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
});
```

#### মৌলিক Playwright API

```javascript
const { test, expect } = require('@playwright/test');

test.describe('প্রোডাক্ট পেজ', () => {
  test('প্রোডাক্ট লিস্ট লোড হওয়া উচিত', async ({ page }) => {
    // page.goto — পেজে যাওয়া
    await page.goto('/products');

    // page.locator — এলিমেন্ট খোঁজা (auto-waiting সহ)
    const heading = page.locator('h1');
    await expect(heading).toHaveText('সকল প্রোডাক্ট');

    // getByRole — accessibility role দিয়ে খোঁজা (সবচেয়ে ভালো পদ্ধতি)
    await page.getByRole('button', { name: 'কার্টে যোগ করুন' }).click();

    // getByTestId — data-testid দিয়ে খোঁজা
    await expect(page.getByTestId('cart-count')).toHaveText('1');

    // getByPlaceholder — placeholder দিয়ে ইনপুট খোঁজা
    await page.getByPlaceholder('অনুসন্ধান করুন...').fill('ল্যাপটপ');

    // getByText — টেক্সট দিয়ে খোঁজা
    await expect(page.getByText('কোনো ফলাফল নেই')).toBeHidden();
  });
});
```

#### Auto-Waiting — Playwright-এর সবচেয়ে শক্তিশালী ফিচার

```javascript
test('Auto-waiting ডেমো', async ({ page }) => {
  await page.goto('/dashboard');

  // Playwright স্বয়ংক্রিয়ভাবে অপেক্ষা করে:
  // ১. এলিমেন্ট DOM-এ আছে কিনা
  // ২. দৃশ্যমান কিনা
  // ৩. স্থির (stable) কিনা — অ্যানিমেশন শেষ হয়েছে কিনা
  // ৪. ক্লিকযোগ্য কিনা — অন্য এলিমেন্ট দ্বারা ঢাকা নেই

  // কোনো sleep/wait লাগে না!
  await page.getByRole('button', { name: 'ডেটা লোড করুন' }).click();
  await expect(page.locator('.data-table')).toBeVisible();

  // নির্দিষ্ট শর্তে অপেক্ষা করা
  await page.waitForResponse('**/api/data');
  await page.waitForLoadState('networkidle');
  await page.waitForURL('**/success');
});
```

#### মাল্টি-ব্রাউজার টেস্টিং

```bash
# সব ব্রাউজারে চালান
npx playwright test

# নির্দিষ্ট ব্রাউজার
npx playwright test --project=chromium
npx playwright test --project=firefox
npx playwright test --project="mobile-chrome"
```

#### Codegen — টেস্ট অটো-জেনারেট

```bash
# ব্রাউজার খুলে আপনার অ্যাকশন রেকর্ড করুন
npx playwright codegen http://localhost:3000

# বাংলাদেশি সাইটের জন্য
npx playwright codegen https://www.daraz.com.bd
```

> **Codegen** ব্রাউজারে আপনার প্রতিটি ক্লিক ও টাইপ ট্র্যাক করে স্বয়ংক্রিয়ভাবে টেস্ট কোড তৈরি করে। নতুনদের জন্য অসাধারণ!

#### Trace Viewer — ডিবাগিং

```bash
# trace সহ টেস্ট চালান
npx playwright test --trace on

# trace ফাইল দেখুন
npx playwright show-trace trace.zip
```

```javascript
test('trace ডেমো', async ({ page, context }) => {
  // প্রোগ্রাম্যাটিক trace শুরু
  await context.tracing.start({ screenshots: true, snapshots: true });

  await page.goto('/checkout');
  // ... টেস্ট স্টেপ ...

  // trace সংরক্ষণ
  await context.tracing.stop({ path: 'checkout-trace.zip' });
});
```

> Trace Viewer প্রতিটি স্টেপের স্ক্রিনশট, নেটওয়ার্ক কল, কনসোল লগ দেখায় — ডিবাগিংয়ে অতুলনীয়।

---

### ৩. Laravel Dusk — Laravel-এর নিজস্ব ব্রাউজার টেস্টিং

#### সেটআপ

```bash
# Dusk ইনস্টল
composer require laravel/dusk --dev

# Dusk সেটআপ
php artisan dusk:install

# Chrome driver আপডেট
php artisan dusk:chrome-driver --detect

# .env.dusk.local তৈরি করুন
cp .env .env.dusk.local
```

```env
# .env.dusk.local
APP_URL=http://localhost:8001
DB_DATABASE=testing_db
SESSION_DRIVER=file
```

#### প্রজেক্ট স্ট্রাকচার

```
tests/
└── Browser/
    ├── LoginTest.php
    ├── CheckoutTest.php
    ├── BkashPaymentTest.php
    ├── Pages/                  # Page Object Model
    │   ├── LoginPage.php
    │   └── CheckoutPage.php
    ├── Components/             # পুনরায় ব্যবহারযোগ্য কম্পোনেন্ট
    │   └── DatePicker.php
    └── screenshots/            # ব্যর্থ টেস্টের স্ক্রিনশট
```

#### মৌলিক Dusk API

```php
<?php

namespace Tests\Browser;

use App\Models\User;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ProductTest extends DuskTestCase
{
    /**
     * প্রোডাক্ট পেজ পরীক্ষা
     */
    public function test_user_can_view_products(): void
    {
        $user = User::factory()->create();

        $this->browse(function (Browser $browser) use ($user) {
            $browser->loginAs($user)
                    ->visit('/products')
                    ->assertSee('সকল প্রোডাক্ট')
                    ->assertPresent('.product-card')
                    ->screenshot('products-page');
        });
    }

    /**
     * প্রোডাক্ট সার্চ পরীক্ষা (বাংলা)
     */
    public function test_user_can_search_in_bangla(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/products')
                    ->type('#search', 'মোবাইল ফোন')
                    ->keys('#search', '{enter}')
                    ->waitForText('অনুসন্ধান ফলাফল')
                    ->assertSee('মোবাইল')
                    ->assertDontSee('কোনো ফলাফল নেই');
        });
    }

    /**
     * ফর্ম ফিলআপ পরীক্ষা
     */
    public function test_form_operations(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/register')
                    // type() — ইনপুটে টাইপ করা
                    ->type('name', 'মোহাম্মদ আলী')
                    ->type('email', 'ali@example.com')
                    ->type('phone', '01712345678')
                    // select() — ড্রপডাউন সিলেক্ট
                    ->select('district', 'dhaka')
                    // check() — চেকবক্স
                    ->check('#terms')
                    // click() — বাটন ক্লিক
                    ->click('@submit-btn')  // dusk="submit-btn"
                    // assertSee() — টেক্সট দেখা যাচ্ছে কিনা
                    ->assertSee('রেজিস্ট্রেশন সফল হয়েছে')
                    // assertPathIs() — URL যাচাই
                    ->assertPathIs('/dashboard');
        });
    }
}
```

#### Screenshots ও Console Logs

```php
public function test_with_debugging(): void
{
    $this->browse(function (Browser $browser) {
        $browser->visit('/checkout')
                ->screenshot('checkout-step-1')   // স্ক্রিনশট নেওয়া
                ->type('#address', 'মিরপুর, ঢাকা')
                ->screenshot('checkout-step-2')
                ->click('#next')
                ->storeConsoleLog('checkout-logs') // কনসোল লগ সংরক্ষণ
                ->storeSource('checkout-source');  // HTML সোর্স সংরক্ষণ
    });
}
```

#### Chrome Driver কাস্টমাইজেশন

```php
// tests/DuskTestCase.php
protected function driver(): RemoteWebDriver
{
    $options = (new ChromeOptions)->addArguments([
        '--disable-gpu',
        '--headless=new',
        '--window-size=1920,1080',
        '--lang=bn',                    // বাংলা ভাষা
        '--accept-lang=bn-BD,bn',
    ]);

    return RemoteWebDriver::create(
        'http://localhost:9515',
        DesiredCapabilities::chrome()->setCapability(
            ChromeOptions::CAPABILITY_W3C, $options
        )
    );
}
```

---

## 🔥 E2E টেস্ট সিনারিও

### ১. লগইন/রেজিস্ট্রেশন ফ্লো টেস্ট

#### Cypress

```javascript
describe('অথেন্টিকেশন ফ্লো', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('সঠিক তথ্য দিয়ে লগইন করতে পারবে', () => {
    cy.get('[data-testid="email"]').type('user@example.com');
    cy.get('[data-testid="password"]').type('Pass123!');
    cy.get('[data-testid="login-btn"]').click();

    cy.url().should('include', '/dashboard');
    cy.get('[data-testid="welcome"]').should('contain', 'স্বাগতম');
  });

  it('ভুল পাসওয়ার্ড দিলে ত্রুটি দেখাবে', () => {
    cy.get('[data-testid="email"]').type('user@example.com');
    cy.get('[data-testid="password"]').type('wrong');
    cy.get('[data-testid="login-btn"]').click();

    cy.get('.error-message')
      .should('be.visible')
      .and('contain', 'ইমেইল বা পাসওয়ার্ড ভুল');
  });

  it('রেজিস্ট্রেশন সম্পন্ন করতে পারবে', () => {
    cy.visit('/register');
    cy.get('#name').type('ফারহানা আক্তার');
    cy.get('#email').type(`test${Date.now()}@mail.com`);
    cy.get('#phone').type('01812345678');
    cy.get('#password').type('Secure123!');
    cy.get('#password_confirmation').type('Secure123!');
    cy.get('#district').select('চট্টগ্রাম');
    cy.get('#terms').check();
    cy.get('#register-btn').click();

    cy.url().should('include', '/verify-email');
    cy.contains('যাচাইকরণ ইমেইল পাঠানো হয়েছে');
  });
});
```

#### Playwright

```javascript
const { test, expect } = require('@playwright/test');

test.describe('অথেন্টিকেশন ফ্লো', () => {
  test('সফল লগইন', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('ইমেইল').fill('user@example.com');
    await page.getByLabel('পাসওয়ার্ড').fill('Pass123!');
    await page.getByRole('button', { name: 'লগইন' }).click();

    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.getByText('স্বাগতম')).toBeVisible();
  });

  test('ভুল তথ্যে ত্রুটি বার্তা', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('ইমেইল').fill('user@example.com');
    await page.getByLabel('পাসওয়ার্ড').fill('wrong');
    await page.getByRole('button', { name: 'লগইন' }).click();

    await expect(page.locator('.error-message')).toContainText(
      'ইমেইল বা পাসওয়ার্ড ভুল'
    );
  });
});
```

#### Laravel Dusk

```php
<?php

namespace Tests\Browser;

use Tests\DuskTestCase;
use Laravel\Dusk\Browser;
use App\Models\User;

class AuthenticationTest extends DuskTestCase
{
    public function test_successful_login(): void
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => bcrypt('Pass123!')
        ]);

        $this->browse(function (Browser $browser) {
            $browser->visit('/login')
                    ->type('email', 'test@example.com')
                    ->type('password', 'Pass123!')
                    ->press('লগইন')
                    ->assertPathIs('/dashboard')
                    ->assertSee('স্বাগতম');
        });
    }

    public function test_failed_login_shows_error(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/login')
                    ->type('email', 'fake@example.com')
                    ->type('password', 'wrong')
                    ->press('লগইন')
                    ->assertSee('ইমেইল বা পাসওয়ার্ড ভুল');
        });
    }

    public function test_registration_flow(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/register')
                    ->type('name', 'ফারহানা আক্তার')
                    ->type('email', 'farhana@example.com')
                    ->type('phone', '01812345678')
                    ->type('password', 'Secure123!')
                    ->type('password_confirmation', 'Secure123!')
                    ->select('district', 'চট্টগ্রাম')
                    ->check('#terms')
                    ->press('রেজিস্ট্রেশন করুন')
                    ->assertPathIs('/verify-email')
                    ->assertSee('যাচাইকরণ ইমেইল পাঠানো হয়েছে');
        });
    }
}
```

---

### ২. ই-কমার্স চেকআউট ফ্লো (Cart → Shipping → Payment → Confirmation)

```javascript
// Cypress — সম্পূর্ণ ক্রয় প্রবাহ
describe('ই-কমার্স চেকআউট', () => {
  beforeEach(() => {
    cy.login('buyer@shop.com.bd', 'Pass123!');
  });

  it('পণ্য কিনে অর্ডার নিশ্চিত করতে পারবে', () => {
    // ধাপ ১: পণ্য কার্টে যোগ
    cy.visit('/products');
    cy.contains('.product-card', 'Samsung Galaxy A54')
      .within(() => {
        cy.get('[data-testid="add-to-cart"]').click();
      });
    cy.getByTestId('cart-count').should('have.text', '1');

    // ধাপ ২: কার্ট পর্যালোচনা
    cy.visit('/cart');
    cy.get('.cart-item').should('have.length', 1);
    cy.getByTestId('cart-total').should('contain', '৳');
    cy.getByTestId('checkout-btn').click();

    // ধাপ ৩: শিপিং তথ্য
    cy.url().should('include', '/checkout/shipping');
    cy.get('#name').type('কামরুল হাসান');
    cy.get('#phone').type('01912345678');
    cy.get('#address').type('বাড়ি ১২, রোড ৫, ধানমন্ডি');
    cy.get('#city').select('ঢাকা');
    cy.get('#postal_code').type('1205');
    cy.get('#shipping-method').select('standard'); // ৳60
    cy.getByTestId('continue-btn').click();

    // ধাপ ৪: পেমেন্ট
    cy.url().should('include', '/checkout/payment');
    cy.mockBkashPayment(35999);
    cy.get('[data-testid="bkash-option"]').click();
    cy.get('#bkash-number').type('01712345678');
    cy.getByTestId('pay-btn').click();

    // ধাপ ৫: অর্ডার নিশ্চিতকরণ
    cy.url().should('include', '/order/confirmation');
    cy.contains('অর্ডার সফলভাবে সম্পন্ন হয়েছে');
    cy.getByTestId('order-id').should('exist');
    cy.getByTestId('order-total').should('contain', '৳36,059');
  });
});
```

```javascript
// Playwright — একই চেকআউট ফ্লো
const { test, expect } = require('@playwright/test');

test('সম্পূর্ণ চেকআউট ফ্লো', async ({ page }) => {
  // লগইন
  await page.goto('/login');
  await page.getByLabel('ইমেইল').fill('buyer@shop.com.bd');
  await page.getByLabel('পাসওয়ার্ড').fill('Pass123!');
  await page.getByRole('button', { name: 'লগইন' }).click();

  // পণ্য যোগ
  await page.goto('/products');
  await page.locator('.product-card')
    .filter({ hasText: 'Samsung Galaxy A54' })
    .getByRole('button', { name: 'কার্টে যোগ করুন' })
    .click();

  // চেকআউট
  await page.goto('/cart');
  await page.getByRole('button', { name: 'চেকআউট' }).click();

  // শিপিং
  await page.getByLabel('নাম').fill('কামরুল হাসান');
  await page.getByLabel('ফোন').fill('01912345678');
  await page.getByLabel('ঠিকানা').fill('বাড়ি ১২, রোড ৫, ধানমন্ডি');
  await page.getByLabel('শহর').selectOption('ঢাকা');
  await page.getByRole('button', { name: 'পরবর্তী' }).click();

  // পেমেন্ট (mock)
  await page.route('**/api/bkash/**', (route) => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({ trxID: 'TRX123', status: 'Completed' })
    });
  });

  await page.getByTestId('bkash-option').click();
  await page.getByPlaceholder('বিকাশ নম্বর').fill('01712345678');
  await page.getByRole('button', { name: 'পেমেন্ট করুন' }).click();

  // নিশ্চিতকরণ
  await expect(page).toHaveURL(/.*confirmation/);
  await expect(page.getByText('অর্ডার সফলভাবে সম্পন্ন হয়েছে')).toBeVisible();
});
```

---

### ৩. ফর্ম ভ্যালিডেশন টেস্টিং

```javascript
// Cypress — বিস্তারিত ফর্ম ভ্যালিডেশন
describe('ফর্ম ভ্যালিডেশন', () => {
  beforeEach(() => {
    cy.visit('/register');
  });

  it('খালি ফর্ম সাবমিট করলে সব ত্রুটি দেখাবে', () => {
    cy.get('#register-btn').click();

    cy.get('[data-error="name"]').should('contain', 'নাম আবশ্যক');
    cy.get('[data-error="email"]').should('contain', 'ইমেইল আবশ্যক');
    cy.get('[data-error="phone"]').should('contain', 'ফোন নম্বর আবশ্যক');
    cy.get('[data-error="password"]').should('contain', 'পাসওয়ার্ড আবশ্যক');
  });

  it('অবৈধ ইমেইলে ত্রুটি দেখাবে', () => {
    cy.get('#email').type('invalid-email');
    cy.get('#email').blur();
    cy.get('[data-error="email"]').should('contain', 'সঠিক ইমেইল দিন');
  });

  it('বাংলাদেশি ফোন নম্বর ভ্যালিডেশন', () => {
    // সঠিক নম্বর
    const validNumbers = ['01712345678', '01812345678', '01912345678'];
    validNumbers.forEach((num) => {
      cy.get('#phone').clear().type(num);
      cy.get('#phone').blur();
      cy.get('[data-error="phone"]').should('not.exist');
    });

    // ভুল নম্বর
    const invalidNumbers = ['0171234', '02812345678', '12345678901'];
    invalidNumbers.forEach((num) => {
      cy.get('#phone').clear().type(num);
      cy.get('#phone').blur();
      cy.get('[data-error="phone"]').should('contain', 'সঠিক ফোন নম্বর দিন');
    });
  });

  it('পাসওয়ার্ড শক্তি নির্দেশক কাজ করবে', () => {
    cy.get('#password').type('123');
    cy.get('.strength-meter').should('have.class', 'weak');
    cy.get('.strength-text').should('contain', 'দুর্বল');

    cy.get('#password').clear().type('MyPass123!');
    cy.get('.strength-meter').should('have.class', 'strong');
    cy.get('.strength-text').should('contain', 'শক্তিশালী');
  });

  it('বাংলা টেক্সট ইনপুট সঠিকভাবে কাজ করবে', () => {
    cy.get('#name').type('মোহাম্মদ আবদুল্লাহ');
    cy.get('#name').should('have.value', 'মোহাম্মদ আবদুল্লাহ');
    cy.get('#address').type('বাড়ি নং ১২/ক, সড়ক নং ৫');
    cy.get('#address').should('have.value', 'বাড়ি নং ১২/ক, সড়ক নং ৫');
  });
});
```

---

### ৪. ফাইল আপলোড টেস্টিং

```javascript
// Cypress — ফাইল আপলোড
describe('ফাইল আপলোড', () => {
  it('ছবি আপলোড করতে পারবে', () => {
    cy.visit('/profile/edit');

    // selectFile ব্যবহার (Cypress 12+)
    cy.get('input[type="file"]').selectFile('cypress/fixtures/avatar.jpg');

    // প্রিভিউ দেখা যাচ্ছে কিনা
    cy.get('.image-preview img').should('be.visible');
    cy.get('#upload-btn').click();
    cy.contains('ছবি সফলভাবে আপলোড হয়েছে');
  });

  it('অনুমোদিত ফরম্যাটের বাইরে ফাইল দিলে ত্রুটি দেখাবে', () => {
    cy.visit('/profile/edit');
    cy.get('input[type="file"]').selectFile('cypress/fixtures/document.exe');
    cy.get('.error').should('contain', 'শুধুমাত্র JPG, PNG, বা GIF অনুমোদিত');
  });

  it('ড্র্যাগ-অ্যান্ড-ড্রপ আপলোড', () => {
    cy.visit('/upload');
    cy.get('.dropzone').selectFile('cypress/fixtures/photo.png', {
      action: 'drag-drop'
    });
    cy.get('.file-list').should('contain', 'photo.png');
  });
});

// Playwright — ফাইল আপলোড
test('ফাইল আপলোড', async ({ page }) => {
  await page.goto('/profile/edit');

  // file chooser ইভেন্ট ধরা
  const fileChooserPromise = page.waitForEvent('filechooser');
  await page.getByText('ছবি নির্বাচন করুন').click();
  const fileChooser = await fileChooserPromise;
  await fileChooser.setFiles('tests/fixtures/avatar.jpg');

  await expect(page.locator('.image-preview img')).toBeVisible();
  await page.getByRole('button', { name: 'আপলোড' }).click();
  await expect(page.getByText('ছবি সফলভাবে আপলোড হয়েছে')).toBeVisible();
});
```

---

### ৫. রিয়েল-টাইম ফিচার টেস্টিং (WebSocket/Notifications)

```javascript
// Cypress — রিয়েল-টাইম নোটিফিকেশন টেস্ট
describe('রিয়েল-টাইম নোটিফিকেশন', () => {
  it('নতুন অর্ডার নোটিফিকেশন দেখাবে', () => {
    cy.login('admin@shop.com.bd', 'Admin123!');
    cy.visit('/admin/dashboard');

    // WebSocket ইভেন্ট সিমুলেট করা
    cy.window().then((win) => {
      win.Echo.channel('orders').listen('NewOrder', {
        order_id: 'ORD-12345',
        customer: 'মোহাম্মদ আলী',
        total: 5000
      });
    });

    cy.get('.notification-bell .badge').should('contain', '1');
    cy.get('.notification-bell').click();
    cy.get('.notification-item').first()
      .should('contain', 'নতুন অর্ডার')
      .and('contain', 'মোহাম্মদ আলী');
  });
});

// Playwright — WebSocket টেস্ট
test('চ্যাট মেসেজ রিয়েল-টাইমে আসবে', async ({ page, context }) => {
  // দুটি ব্রাউজার ট্যাব (দুই ব্যবহারকারী)
  const sender = await context.newPage();
  const receiver = page;

  // উভয় ইউজার লগইন
  await sender.goto('/login');
  await sender.getByLabel('ইমেইল').fill('user1@test.com');
  await sender.getByLabel('পাসওয়ার্ড').fill('Pass123!');
  await sender.getByRole('button', { name: 'লগইন' }).click();

  await receiver.goto('/login');
  await receiver.getByLabel('ইমেইল').fill('user2@test.com');
  await receiver.getByLabel('পাসওয়ার্ড').fill('Pass123!');
  await receiver.getByRole('button', { name: 'লগইন' }).click();

  // চ্যাট রুমে যাওয়া
  await sender.goto('/chat/room-1');
  await receiver.goto('/chat/room-1');

  // মেসেজ পাঠানো
  await sender.getByPlaceholder('বার্তা লিখুন...').fill('আসসালামু আলাইকুম');
  await sender.getByRole('button', { name: 'পাঠান' }).click();

  // রিসিভারের কাছে মেসেজ আসা
  await expect(
    receiver.locator('.message').last()
  ).toContainText('আসসালামু আলাইকুম');
});
```

---

### ৬. মাল্টি-ট্যাব/মাল্টি-ইউজার টেস্টিং (Playwright)

```javascript
const { test, expect } = require('@playwright/test');

test('একাধিক ট্যাবে কাজ করা', async ({ context }) => {
  // প্রথম ট্যাব
  const page1 = await context.newPage();
  await page1.goto('/products');

  // দ্বিতীয় ট্যাব (নতুন ট্যাবে লিঙ্ক খুলবে)
  const page2Promise = context.waitForEvent('page');
  await page1.getByText('বিস্তারিত দেখুন').click();
  const page2 = await page2Promise;
  await page2.waitForLoadState();

  await expect(page2).toHaveURL(/.*product\/\d+/);
  await expect(page2.locator('h1')).toBeVisible();
});

test('বিভিন্ন ইউজার দিয়ে সমান্তরাল টেস্ট', async ({ browser }) => {
  // প্রতিটি ব্যবহারকারীর জন্য আলাদা context (আলাদা সেশন)
  const adminContext = await browser.newContext();
  const buyerContext = await browser.newContext();

  const adminPage = await adminContext.newPage();
  const buyerPage = await buyerContext.newPage();

  // অ্যাডমিন একটি পণ্য যোগ করে
  await adminPage.goto('/admin/products/create');
  await adminPage.getByLabel('পণ্যের নাম').fill('নতুন ল্যাপটপ');
  await adminPage.getByLabel('দাম').fill('75000');
  await adminPage.getByRole('button', { name: 'প্রকাশ করুন' }).click();

  // ক্রেতা সেই পণ্য দেখতে পায়
  await buyerPage.goto('/products');
  await buyerPage.reload();
  await expect(buyerPage.getByText('নতুন ল্যাপটপ')).toBeVisible();

  // ক্লিনআপ
  await adminContext.close();
  await buyerContext.close();
});

test('একাধিক ব্রাউজারে একই সাথে বিড করা', async ({ browser }) => {
  const bidder1 = await browser.newContext();
  const bidder2 = await browser.newContext();

  const page1 = await bidder1.newPage();
  const page2 = await bidder2.newPage();

  // উভয় ইউজার একই নিলাম পেজে
  await page1.goto('/auction/item-100');
  await page2.goto('/auction/item-100');

  // বিডার ১ বিড করে
  await page1.getByLabel('বিডের পরিমাণ').fill('5000');
  await page1.getByRole('button', { name: 'বিড করুন' }).click();

  // বিডার ২ আপডেট দেখতে পায়
  await expect(page2.getByTestId('current-bid')).toContainText('৳5,000');

  await bidder1.close();
  await bidder2.close();
});
```

---

### ৭. ভিজ্যুয়াল রিগ্রেশন টেস্টিং

```javascript
// Playwright — স্ন্যাপশট টেস্টিং (built-in)
test('হোমপেজের ভিজ্যুয়াল তুলনা', async ({ page }) => {
  await page.goto('/');

  // পুরো পেজের স্ক্রিনশট তুলনা
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixelRatio: 0.01    // ১% পর্যন্ত পার্থক্য গ্রহণযোগ্য
  });

  // নির্দিষ্ট এলিমেন্টের স্ক্রিনশট
  await expect(page.locator('.hero-section')).toHaveScreenshot('hero.png');

  // অ্যানিমেশন বন্ধ করে তুলনা
  await expect(page).toHaveScreenshot('static-page.png', {
    animations: 'disabled',
    mask: [page.locator('.dynamic-banner')]  // পরিবর্তনশীল অংশ আড়াল
  });
});

// Cypress + Percy — ভিজ্যুয়াল রিভিউ
describe('ভিজ্যুয়াল রিগ্রেশন', () => {
  it('পণ্য কার্ড সঠিকভাবে রেন্ডার হচ্ছে', () => {
    cy.visit('/products');
    cy.get('.product-grid').should('be.visible');

    // Percy স্ন্যাপশট
    cy.percySnapshot('Products Page', {
      widths: [375, 768, 1280],     // মোবাইল, ট্যাবলেট, ডেস্কটপ
      minHeight: 1024
    });
  });

  it('বাংলা ফন্ট সঠিকভাবে রেন্ডার হচ্ছে', () => {
    cy.visit('/bn/about');
    // বাংলা কন্টেন্ট লোড হওয়ার জন্য অপেক্ষা
    cy.get('.bangla-content').should('be.visible');
    cy.percySnapshot('Bangla Content Page');
  });
});
```

```php
// Laravel Dusk — স্ক্রিনশট তুলনা
public function test_visual_regression(): void
{
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
                ->waitFor('.hero-section')
                ->assertScreenshot('homepage', tolerance: 0.01);
                // assertScreenshot (custom assertion) পিক্সেল-by-পিক্সেল তুলনা করে
    });
}
```

---

### ৮. API Mocking in E2E

```javascript
// Cypress — cy.intercept দিয়ে API mock করা
describe('API Mocking', () => {
  it('বিকাশ API ত্রুটিতে সঠিক বার্তা দেখাবে', () => {
    cy.intercept('POST', '/api/payment/bkash/create', {
      statusCode: 503,
      body: {
        statusCode: '2023',
        statusMessage: 'Insufficient Balance'
      }
    }).as('bkashCreate');

    cy.visit('/checkout/payment');
    cy.get('[data-testid="bkash-option"]').click();
    cy.get('#bkash-number').type('01712345678');
    cy.get('#pay-btn').click();

    cy.wait('@bkashCreate');
    cy.get('.payment-error')
      .should('contain', 'অপর্যাপ্ত ব্যালেন্স');
  });

  it('ধীর API-তে লোডিং স্পিনার দেখাবে', () => {
    cy.intercept('GET', '/api/products', (req) => {
      req.reply({
        delay: 5000,
        fixture: 'products.json'
      });
    }).as('slowProducts');

    cy.visit('/products');
    cy.get('.loading-spinner').should('be.visible');
    cy.wait('@slowProducts');
    cy.get('.loading-spinner').should('not.exist');
    cy.get('.product-card').should('have.length.gt', 0);
  });

  it('API কল-এর request body যাচাই', () => {
    cy.intercept('POST', '/api/orders', (req) => {
      // request body পরীক্ষা
      expect(req.body.items).to.have.length(2);
      expect(req.body.shipping.city).to.equal('ঢাকা');

      req.reply({
        statusCode: 201,
        body: { orderId: 'ORD-999' }
      });
    }).as('createOrder');

    // ... checkout steps ...
    cy.wait('@createOrder');
  });
});

// Playwright — page.route দিয়ে API mock করা
test('API mocking with Playwright', async ({ page }) => {
  // JSON ফাইল থেকে mock ডেটা
  await page.route('**/api/products', async (route) => {
    const json = require('./fixtures/products.json');
    await route.fulfill({ json });
  });

  // শর্তসাপেক্ষ mock
  await page.route('**/api/search**', async (route) => {
    const url = new URL(route.request().url());
    const query = url.searchParams.get('q');

    if (query === 'মোবাইল') {
      await route.fulfill({
        json: { results: [{ name: 'Samsung Galaxy', price: 25000 }] }
      });
    } else {
      await route.fulfill({
        json: { results: [] }
      });
    }
  });

  await page.goto('/products');
  await expect(page.locator('.product-card')).toHaveCount(10);
});
```

---

### ৯. মোবাইল রেসপন্সিভ টেস্টিং

```javascript
// Cypress — ভিউপোর্ট পরিবর্তন
describe('রেসপন্সিভ ডিজাইন', () => {
  const viewports = [
    { name: 'মোবাইল', width: 375, height: 667 },
    { name: 'ট্যাবলেট', width: 768, height: 1024 },
    { name: 'ডেস্কটপ', width: 1280, height: 720 }
  ];

  viewports.forEach(({ name, width, height }) => {
    it(`${name}-এ নেভিগেশন সঠিকভাবে কাজ করবে`, () => {
      cy.viewport(width, height);
      cy.visit('/');

      if (width < 768) {
        // মোবাইলে হ্যামবার্গার মেনু
        cy.get('.desktop-nav').should('not.be.visible');
        cy.get('.hamburger-btn').should('be.visible').click();
        cy.get('.mobile-nav').should('be.visible');
        cy.get('.mobile-nav').contains('পণ্যসমূহ').click();
      } else {
        // ডেস্কটপে সাধারণ নেভ
        cy.get('.desktop-nav').should('be.visible');
        cy.get('.desktop-nav').contains('পণ্যসমূহ').click();
      }

      cy.url().should('include', '/products');
    });
  });

  it('প্রোডাক্ট গ্রিড রেসপন্সিভ', () => {
    // মোবাইলে ১ কলাম
    cy.viewport('iphone-x');
    cy.visit('/products');
    cy.get('.product-grid').should('have.css', 'grid-template-columns')
      .and('match', /1fr/);

    // ডেস্কটপে ৪ কলাম
    cy.viewport(1280, 720);
    cy.get('.product-grid').should('have.css', 'grid-template-columns')
      .and('not.match', /^[^f]*1fr[^f]*$/);
  });
});

// Playwright — ডিভাইস ইমুলেশন (আরও সঠিক)
const { devices } = require('@playwright/test');

test.describe('মোবাইল টেস্ট', () => {
  test.use({ ...devices['iPhone 13'] });

  test('মোবাইল নেভিগেশন', async ({ page }) => {
    await page.goto('/');

    // touch ইভেন্ট, সঠিক viewport, user agent সব সিমুলেট হয়
    await page.getByRole('button', { name: 'মেনু' }).tap();
    await expect(page.locator('.mobile-nav')).toBeVisible();
  });
});
```

---

### ১০. অ্যাক্সেসিবিলিটি টেস্টিং (axe-core Integration)

```javascript
// Cypress + cypress-axe
// ইনস্টল: npm install cypress-axe axe-core --save-dev

// cypress/support/e2e.js
import 'cypress-axe';

describe('অ্যাক্সেসিবিলিটি', () => {
  it('হোমপেজ অ্যাক্সেসিবল', () => {
    cy.visit('/');
    cy.injectAxe();

    // সকল অ্যাক্সেসিবিলিটি লঙ্ঘন পরীক্ষা
    cy.checkA11y();
  });

  it('নির্দিষ্ট এলাকা পরীক্ষা', () => {
    cy.visit('/products');
    cy.injectAxe();

    // শুধু ন্যাভ পরীক্ষা
    cy.checkA11y('nav');

    // নির্দিষ্ট নিয়ম বাদ দিয়ে
    cy.checkA11y(null, {
      rules: {
        'color-contrast': { enabled: false }  // রং বৈসাদৃশ্য বাদ
      }
    });
  });

  it('ফর্ম অ্যাক্সেসিবিলিটি', () => {
    cy.visit('/register');
    cy.injectAxe();

    // গুরুতর (critical) সমস্যাগুলো পরীক্ষা
    cy.checkA11y(null, {
      includedImpacts: ['critical', 'serious']
    });
  });
});

// Playwright + @axe-core/playwright
// ইনস্টল: npm install @axe-core/playwright --save-dev
const { test, expect } = require('@playwright/test');
const AxeBuilder = require('@axe-core/playwright').default;

test('অ্যাক্সেসিবিলিটি অডিট', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])   // WCAG 2.0 Level AA
    .analyze();

  expect(results.violations).toEqual([]);
});

test('বাংলা কন্টেন্টের অ্যাক্সেসিবিলিটি', async ({ page }) => {
  await page.goto('/bn/about');

  const results = await new AxeBuilder({ page })
    .include('.main-content')          // শুধু মূল কন্টেন্ট পরীক্ষা
    .exclude('.third-party-widget')    // তৃতীয় পক্ষের উইজেট বাদ
    .withTags(['wcag2a', 'wcag2aa', 'best-practice'])
    .analyze();

  // লঙ্ঘনের বিস্তারিত রিপোর্ট
  results.violations.forEach((violation) => {
    console.log(`${violation.id}: ${violation.description}`);
    console.log(`  প্রভাব: ${violation.impact}`);
    console.log(`  এলিমেন্ট: ${violation.nodes.length}টি`);
  });

  expect(results.violations).toHaveLength(0);
});
```

---

## 🏗️ Advanced Topics

### CI/CD Integration (GitHub Actions)

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # ==================== Cypress ====================
  cypress-e2e:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        containers: [1, 2, 3]    # ৩টি প্যারালেল কন্টেইনার
    steps:
      - uses: actions/checkout@v4

      - name: Node.js সেটআপ
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: ডিপেন্ডেন্সি ইনস্টল
        run: npm ci

      - name: অ্যাপ বিল্ড
        run: npm run build

      - name: Cypress চালান
        uses: cypress-io/github-action@v6
        with:
          start: npm run start
          wait-on: 'http://localhost:3000'
          wait-on-timeout: 120
          record: true
          parallel: true
          group: 'E2E Tests'
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: ব্যর্থতায় আর্টিফ্যাক্ট সংরক্ষণ
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots-${{ matrix.containers }}
          path: cypress/screenshots
          retention-days: 7

  # ==================== Playwright ====================
  playwright-e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - name: Playwright ব্রাউজার ইনস্টল
        run: npx playwright install --with-deps

      - name: অ্যাপ বিল্ড
        run: npm run build

      - name: Playwright টেস্ট চালান
        run: npx playwright test
        env:
          CI: true

      - name: রিপোর্ট আপলোড
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14

  # ==================== Laravel Dusk ====================
  laravel-dusk:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: testing
          MYSQL_ROOT_PASSWORD: secret
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    steps:
      - uses: actions/checkout@v4

      - name: PHP সেটআপ
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, mysql, gd

      - name: Composer ইনস্টল
        run: composer install --no-progress

      - name: Chrome driver আপডেট
        run: php artisan dusk:chrome-driver --detect

      - name: Chrome শুরু
        run: google-chrome-stable --headless --disable-gpu --remote-debugging-port=9222 &

      - name: অ্যাপ সেটআপ
        run: |
          cp .env.dusk.ci .env
          php artisan key:generate
          php artisan migrate --seed

      - name: সার্ভার শুরু
        run: php artisan serve --port=8001 &

      - name: Dusk টেস্ট চালান
        run: php artisan dusk
        env:
          APP_URL: http://127.0.0.1:8001

      - name: স্ক্রিনশট সংরক্ষণ
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: dusk-screenshots
          path: tests/Browser/screenshots
```

### প্যারালেল E2E Execution

```javascript
// Playwright — fullyParallel: true কনফিগে ডিফল্ট
// শার্ডিং (CI-তে একাধিক মেশিনে ভাগ করে চালানো)

// কমান্ড:
// মেশিন ১: npx playwright test --shard=1/3
// মেশিন ২: npx playwright test --shard=2/3
// মেশিন ৩: npx playwright test --shard=3/3

// Cypress — cypress-parallel বা Cypress Cloud এর --parallel ফ্ল্যাগ ব্যবহার
// npx cypress run --record --parallel --group "e2e-full"
```

```yaml
# GitHub Actions-এ Playwright শার্ডিং
playwright-sharded:
  runs-on: ubuntu-latest
  strategy:
    fail-fast: false
    matrix:
      shard: [1/4, 2/4, 3/4, 4/4]
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
    - run: npm ci
    - run: npx playwright install --with-deps
    - run: npx playwright test --shard=${{ matrix.shard }}
    - uses: actions/upload-artifact@v4
      if: always()
      with:
        name: blob-report-${{ strategy.job-index }}
        path: blob-report

  # রিপোর্ট একত্রিতকরণ
  merge-reports:
    needs: playwright-sharded
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true
      - run: npx playwright merge-reports --reporter html ./all-blob-reports
```

### Flaky Test Handling — অস্থির টেস্ট সামলানো

```javascript
// Playwright — retry কনফিগ
// playwright.config.js
module.exports = defineConfig({
  retries: process.env.CI ? 2 : 0,
  use: {
    trace: 'on-first-retry'   // প্রথম retry-তে trace রেকর্ড
  }
});

// নির্দিষ্ট টেস্টে retry
test('অস্থির টেস্ট', async ({ page }) => {
  test.info().annotations.push({ type: 'flaky', description: 'নেটওয়ার্ক নির্ভর' });
  // ...
});

// Cypress — retry কনফিগ
// cypress.config.js
module.exports = defineConfig({
  retries: {
    runMode: 2,
    openMode: 0
  }
});
```

#### Flaky টেস্ট চিহ্নিত করা ও ঠিক করার কৌশল

```javascript
// ❌ Flaky — নির্দিষ্ট সময়ে অপেক্ষা (brittle)
cy.wait(5000);
cy.get('.data').should('exist');

// ✅ স্থিতিশীল — শর্তে অপেক্ষা
cy.get('.data', { timeout: 15000 }).should('be.visible');

// ❌ Flaky — অ্যানিমেশন চলাকালীন ক্লিক
cy.get('.modal-btn').click();

// ✅ স্থিতিশীল — অ্যানিমেশন শেষে ক্লিক
cy.get('.modal-btn').should('be.visible').and('not.have.class', 'animating').click();

// ❌ Flaky — test data collision (অন্য টেস্ট একই ডেটা ব্যবহার করছে)
cy.get('#email').type('user@test.com');

// ✅ স্থিতিশীল — ইউনিক ডেটা
const uniqueEmail = `user_${Date.now()}@test.com`;
cy.get('#email').type(uniqueEmail);
```

### Test Data Management — টেস্ট ডেটা ব্যবস্থাপনা

```javascript
// Cypress — beforeEach-এ ডেটা সিডিং
describe('অর্ডার ম্যানেজমেন্ট', () => {
  beforeEach(() => {
    // API দিয়ে ডেটা তৈরি (UI-এর চেয়ে দ্রুত)
    cy.request('POST', '/api/test/seed', {
      users: 5,
      products: 20,
      orders: 10
    });

    cy.login('admin@shop.com.bd', 'Admin123!');
  });

  afterEach(() => {
    // ডেটা পরিষ্কার করা
    cy.request('POST', '/api/test/cleanup');
  });
});

// Playwright — global setup/teardown
// global-setup.js
const { chromium } = require('@playwright/test');

module.exports = async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  // অ্যাডমিন হিসেবে লগইন করে storageState সংরক্ষণ
  await page.goto('http://localhost:3000/login');
  await page.getByLabel('ইমেইল').fill('admin@shop.com.bd');
  await page.getByLabel('পাসওয়ার্ড').fill('Admin123!');
  await page.getByRole('button', { name: 'লগইন' }).click();

  await page.context().storageState({ path: '.auth/admin.json' });
  await browser.close();
};

// playwright.config.js
module.exports = defineConfig({
  globalSetup: require.resolve('./global-setup'),
  projects: [
    {
      name: 'authenticated',
      use: { storageState: '.auth/admin.json' }
    }
  ]
});
```

```php
// Laravel Dusk — DatabaseMigrations trait ব্যবহার
use Illuminate\Foundation\Testing\DatabaseMigrations;

class OrderTest extends DuskTestCase
{
    use DatabaseMigrations;

    protected function setUp(): void
    {
        parent::setUp();
        // প্রতিটি টেস্টে fresh database + seed
        $this->artisan('db:seed', ['--class' => 'TestSeeder']);
    }

    public function test_admin_can_manage_orders(): void
    {
        $admin = User::factory()->admin()->create();
        $orders = Order::factory()->count(5)->create();

        $this->browse(function (Browser $browser) use ($admin) {
            $browser->loginAs($admin)
                    ->visit('/admin/orders')
                    ->assertSee('মোট অর্ডার: ৫');
        });
    }
}
```

---

## ✅ Best Practices — সেরা অনুশীলন

### ১. data-testid ব্যবহার করুন

```html
<!-- ❌ ভঙ্গুর সিলেক্টর -->
<button class="btn btn-primary mt-3">সাবমিট</button>

<!-- ✅ স্থিতিশীল সিলেক্টর -->
<button data-testid="submit-btn" class="btn btn-primary mt-3">সাবমিট</button>
```

```javascript
// ❌ এড়িয়ে চলুন
cy.get('.btn.btn-primary.mt-3').click();
cy.get('div > form > button:nth-child(3)').click();

// ✅ সুপারিশকৃত
cy.getByTestId('submit-btn').click();
// অথবা Playwright-এ
page.getByTestId('submit-btn').click();
page.getByRole('button', { name: 'সাবমিট' }).click();
```

### ২. Page Object Model (POM) ব্যবহার করুন

```javascript
// Playwright — Page Object
class CheckoutPage {
  constructor(page) {
    this.page = page;
    this.nameInput = page.getByLabel('নাম');
    this.phoneInput = page.getByLabel('ফোন');
    this.addressInput = page.getByLabel('ঠিকানা');
    this.nextButton = page.getByRole('button', { name: 'পরবর্তী' });
    this.bkashOption = page.getByTestId('bkash-option');
    this.payButton = page.getByRole('button', { name: 'পেমেন্ট করুন' });
  }

  async fillShipping(data) {
    await this.nameInput.fill(data.name);
    await this.phoneInput.fill(data.phone);
    await this.addressInput.fill(data.address);
    await this.nextButton.click();
  }

  async payWithBkash(number) {
    await this.bkashOption.click();
    await this.page.getByPlaceholder('বিকাশ নম্বর').fill(number);
    await this.payButton.click();
  }
}

// ব্যবহার
test('চেকআউট ফ্লো', async ({ page }) => {
  const checkout = new CheckoutPage(page);
  await page.goto('/checkout');

  await checkout.fillShipping({
    name: 'রাফি আহমেদ',
    phone: '01712345678',
    address: 'গুলশান, ঢাকা'
  });

  await checkout.payWithBkash('01712345678');
  await expect(page.getByText('অর্ডার সফল')).toBeVisible();
});
```

### ৩. গুরুত্বপূর্ণ ব্যবসায়িক ফ্লো-তে ফোকাস

```
✅ টেস্ট করুন:
├── রেজিস্ট্রেশন → লগইন → প্রোফাইল আপডেট
├── পণ্য খুঁজুন → কার্টে যোগ → চেকআউট → পেমেন্ট (বিকাশ/নগদ)
├── অ্যাডমিন → পণ্য যোগ → অর্ডার প্রসেস → ডেলিভারি
└── পাসওয়ার্ড রিসেট ফ্লো

❌ E2E তে টেস্ট করবেন না:
├── প্রতিটি ফর্ম ভ্যালিডেশন মেসেজ (ইউনিট টেস্টে করুন)
├── CSS স্টাইলিং বিস্তারিত
├── প্রতিটি API error code (ইন্টিগ্রেশন টেস্টে করুন)
└── তৃতীয় পক্ষের সার্ভিসের বিস্তারিত behavior
```

### ৪. API দিয়ে state সেটআপ করুন (UI নয়)

```javascript
// ❌ ধীর — UI দিয়ে লগইন
cy.visit('/login');
cy.get('#email').type('admin@test.com');
cy.get('#password').type('password');
cy.get('#login-btn').click();
cy.url().should('include', '/dashboard');

// ✅ দ্রুত — API দিয়ে লগইন
cy.login('admin@test.com', 'password');  // session/token সরাসরি সেট করে
cy.visit('/dashboard');                   // সরাসরি dashboard-এ যান
```

### ৫. টেস্ট পরস্পর স্বাধীন রাখুন

```javascript
// ❌ টেস্ট নির্ভরশীল
it('প্রোডাক্ট তৈরি করে', () => { /* ... */ });
it('তৈরি করা প্রোডাক্ট দেখে', () => { /* আগের টেস্ট দরকার */ });

// ✅ প্রতিটি টেস্ট স্বাধীন
it('প্রোডাক্ট দেখতে পারবে', () => {
  cy.request('POST', '/api/test/products', { name: 'Test Product' });
  cy.visit('/products');
  cy.contains('Test Product');
});
```

---

## ⚠️ Anti-patterns — যা এড়িয়ে চলবেন

### ১. হার্ড-কোডেড sleep/wait

```javascript
// ❌ কখনো করবেন না
cy.wait(5000);
await page.waitForTimeout(3000);
sleep(5);

// ✅ শর্তসাপেক্ষ অপেক্ষা
cy.get('.data', { timeout: 10000 }).should('be.visible');
await expect(page.locator('.data')).toBeVisible({ timeout: 10000 });
```

### ২. UI দিয়ে বারবার প্রি-কন্ডিশন সেটআপ

```javascript
// ❌ প্রতি টেস্টে UI দিয়ে লগইন + navigate
beforeEach(() => {
  cy.visit('/login');
  cy.get('#email').type('admin@test.com');
  cy.get('#password').type('Pass123!');
  cy.get('#login-btn').click();
  cy.visit('/admin');
  cy.get('.sidebar').contains('Products').click();
});

// ✅ API/session দিয়ে সরাসরি
beforeEach(() => {
  cy.login('admin@test.com', 'Pass123!');
  cy.visit('/admin/products');
});
```

### ৩. ভঙ্গুর CSS সিলেক্টর

```javascript
// ❌ সিএসএস ক্লাস পরিবর্তন হলে ভেঙে যাবে
cy.get('body > div:nth-child(2) > main > div.flex > button.bg-blue-500');

// ✅ অর্থপূর্ণ সিলেক্টর
cy.getByTestId('checkout-btn');
cy.contains('button', 'চেকআউট');
```

### ৪. সব কিছু E2E তে টেস্ট করা

```
❌ Anti-pattern: ৫০০+ E2E টেস্ট, ২ ঘণ্টা চলে
   - ছোট পরিবর্তনেও পুরো suite চালাতে হয়
   - Flaky টেস্ট বেশি হয়
   - ডেভেলপাররা টেস্ট ইগনোর করা শুরু করে

✅ সঠিক পদ্ধতি:
   - ২০-৫০টি critical path E2E টেস্ট
   - বাকিগুলো unit/integration-এ
   - E2E suite ১৫-২০ মিনিটে শেষ হওয়া উচিত
```

### ৫. টেস্টে ব্যবসায়িক লজিক

```javascript
// ❌ টেস্টে দাম গণনা করা
cy.get('.price').then(($price) => {
  const price = parseFloat($price.text());
  const tax = price * 0.15;
  const shipping = price > 1000 ? 0 : 60;
  const total = price + tax + shipping;
  cy.get('.total').should('contain', total);
});

// ✅ শুধু ফলাফল যাচাই করা
cy.get('.total').should('not.be.empty');
cy.get('.total').invoke('text').should('match', /৳[\d,]+/);
```

---

## 📋 তুলনা সারণী: Cypress vs Playwright vs Laravel Dusk

| বৈশিষ্ট্য | Cypress | Playwright | Laravel Dusk |
|---|---|---|---|
| **ভাষা** | JavaScript/TypeScript | JS/TS/Python/Java/C# | PHP |
| **ব্রাউজার সাপোর্ট** | Chrome, Edge, Firefox | Chromium, Firefox, WebKit | Chrome (ChromeDriver) |
| **মোবাইল টেস্টিং** | ভিউপোর্ট রিসাইজ | ডিভাইস ইমুলেশন (সেরা) | সীমিত |
| **Auto-waiting** | আছে (implicit) | আছে (সেরা) | আছে (waitFor*) |
| **প্যারালেল** | Cypress Cloud / plugin | Built-in (fullyParallel) | paratest plugin |
| **API Mocking** | cy.intercept (শক্তিশালী) | page.route (শক্তিশালী) | সীমিত |
| **ডিবাগিং** | Time Travel UI | Trace Viewer (সেরা) | Screenshots/Logs |
| **CI ইন্টিগ্রেশন** | সহজ (Docker image) | সহজ | সহজ |
| **শেখার বক্ররেখা** | সবচেয়ে সহজ | মাঝারি | সহজ (Laravel জানলে) |
| **মাল্টি-ট্যাব** | ❌ সাপোর্ট নেই | ✅ পূর্ণ সাপোর্ট | ❌ সীমিত |
| **iFrame সাপোর্ট** | সীমিত | ✅ পূর্ণ সাপোর্ট | সীমিত |
| **নেটওয়ার্ক ইন্টারসেপ্ট** | ✅ চমৎকার | ✅ চমৎকার | ❌ নেই |
| **ভিজ্যুয়াল টেস্ট** | Percy (তৃতীয় পক্ষ) | Built-in screenshot comparison | screenshot() |
| **কমিউনিটি** | বিশাল | দ্রুত বর্ধনশীল | Laravel কমিউনিটি |
| **বিনামূল্যে** | ✅ (Cloud পেইড) | ✅ সম্পূর্ণ বিনামূল্যে | ✅ |

### কখন কোনটি বেছে নেবেন?

```
┌─────────────────────────────────────────────────────────────┐
│ Cypress বেছে নিন যদি:                                       │
│  • দল JavaScript/React/Vue/Angular ব্যবহার করে              │
│  • শুধু Chrome/Edge-এ টেস্ট যথেষ্ট                          │
│  • সহজ শেখা ও চমৎকার DX চান                                │
│  • রিয়েল-টাইম ডিবাগিং GUI দরকার                            │
├─────────────────────────────────────────────────────────────┤
│ Playwright বেছে নিন যদি:                                    │
│  • ক্রস-ব্রাউজার টেস্টিং জরুরি (Chrome + Firefox + Safari) │
│  • মাল্টি-ট্যাব, মাল্টি-ইউজার টেস্ট দরকার                  │
│  • মোবাইল ইমুলেশন দরকার                                    │
│  • সর্বোচ্চ পারফরম্যান্স ও নির্ভরযোগ্যতা চান               │
│  • Python/Java/C# দলের জন্যও দরকার                         │
├─────────────────────────────────────────────────────────────┤
│ Laravel Dusk বেছে নিন যদি:                                  │
│  • Laravel অ্যাপ্লিকেশন টেস্ট করতে চান                      │
│  • PHP ইকোসিস্টেমে থাকতে চান                               │
│  • Eloquent/Factory দিয়ে ডেটা সেটআপ করতে চান               │
│  • Laravel-এর authentication system ব্যবহার করতে চান        │
└─────────────────────────────────────────────────────────────┘
```

---

## 🇧🇩 বাংলাদেশ প্রসঙ্গ

### বিকাশ চেকআউট ফ্লো টেস্টিং

```javascript
// Cypress — বিকাশ পেমেন্ট গেটওয়ে E2E টেস্ট
describe('বিকাশ পেমেন্ট ইন্টিগ্রেশন', () => {
  beforeEach(() => {
    cy.login('customer@test.com', 'Pass123!');
    // কার্টে পণ্য আছে এমন অবস্থায় শুরু
    cy.request('POST', '/api/test/add-to-cart', { productId: 1, qty: 1 });
  });

  it('বিকাশ দিয়ে সফলভাবে পেমেন্ট করতে পারবে', () => {
    // বিকাশ API mock
    cy.intercept('POST', '**/checkout/bkash/token', {
      statusCode: 200,
      body: {
        id_token: 'test_token_123',
        token_type: 'Bearer'
      }
    }).as('bkashToken');

    cy.intercept('POST', '**/checkout/bkash/create', {
      statusCode: 200,
      body: {
        paymentID: 'PAY123456',
        bkashURL: 'https://sandbox.payment.bkash.com/?paymentId=PAY123456',
        amount: '2500',
        merchantInvoiceNumber: 'INV-2024-001'
      }
    }).as('bkashCreate');

    cy.intercept('POST', '**/checkout/bkash/execute', {
      statusCode: 200,
      body: {
        paymentID: 'PAY123456',
        trxID: 'TRX9A8B7C6D',
        transactionStatus: 'Completed',
        amount: '2500',
        currency: 'BDT',
        customerMsisdn: '01712345678'
      }
    }).as('bkashExecute');

    cy.visit('/checkout');

    // বিকাশ নির্বাচন
    cy.getByTestId('payment-bkash').click();
    cy.get('#bkash-wallet').type('01712345678');
    cy.getByTestId('bkash-pay-btn').click();

    cy.wait('@bkashToken');
    cy.wait('@bkashCreate');

    // OTP পেজ (mock)
    cy.get('#bkash-otp').type('123456');
    cy.get('#bkash-confirm').click();

    cy.wait('@bkashExecute');

    // সফল পেমেন্ট যাচাই
    cy.url().should('include', '/order/success');
    cy.contains('পেমেন্ট সফল হয়েছে');
    cy.getByTestId('trx-id').should('contain', 'TRX9A8B7C6D');
    cy.getByTestId('amount').should('contain', '৳2,500');
  });

  it('বিকাশ ব্যালেন্স অপর্যাপ্ত হলে ত্রুটি দেখাবে', () => {
    cy.intercept('POST', '**/checkout/bkash/execute', {
      statusCode: 200,
      body: {
        statusCode: '2023',
        statusMessage: 'Insufficient Balance'
      }
    }).as('bkashFail');

    cy.visit('/checkout');
    cy.getByTestId('payment-bkash').click();
    cy.get('#bkash-wallet').type('01712345678');
    cy.getByTestId('bkash-pay-btn').click();

    cy.wait('@bkashFail');
    cy.get('.payment-error')
      .should('be.visible')
      .and('contain', 'অপর্যাপ্ত ব্যালেন্স। অনুগ্রহ করে বিকাশ অ্যাকাউন্টে টাকা যোগ করুন।');
  });

  it('বিকাশ টাইমআউটে সঠিক বার্তা দেখাবে', () => {
    cy.intercept('POST', '**/checkout/bkash/create', {
      statusCode: 408,
      body: { message: 'Request Timeout' },
      delay: 35000
    }).as('bkashTimeout');

    cy.visit('/checkout');
    cy.getByTestId('payment-bkash').click();
    cy.get('#bkash-wallet').type('01712345678');
    cy.getByTestId('bkash-pay-btn').click();

    // টাইমআউট বার্তা
    cy.get('.payment-error', { timeout: 40000 })
      .should('contain', 'সময়সীমা অতিক্রান্ত');
  });
});
```

### বাংলা টেক্সট ইনপুট টেস্টিং

```javascript
// Playwright — বাংলা ভাষা ও বিশেষ অক্ষর পরীক্ষা
test.describe('বাংলা ভাষা সমর্থন', () => {
  test('বাংলায় প্রোডাক্ট সার্চ', async ({ page }) => {
    await page.goto('/products');
    const searchBox = page.getByPlaceholder('অনুসন্ধান করুন...');

    await searchBox.fill('স্যামসাং গ্যালাক্সি');
    await page.keyboard.press('Enter');

    await expect(page.locator('.search-results')).toContainText('স্যামসাং');
  });

  test('বাংলা ঠিকানা সংরক্ষণ ও প্রদর্শন', async ({ page }) => {
    await page.goto('/profile/address');

    const address = 'বাড়ি নং ১২/ক, সড়ক নং ৫, ব্লক-ঘ, মিরপুর-১০, ঢাকা-১২১৬';
    await page.getByLabel('ঠিকানা').fill(address);
    await page.getByRole('button', { name: 'সংরক্ষণ করুন' }).click();

    await expect(page.getByText('ঠিকানা আপডেট হয়েছে')).toBeVisible();

    // পেজ রিলোড করে যাচাই — ডেটা ঠিকমতো সংরক্ষিত হয়েছে কিনা
    await page.reload();
    await expect(page.getByLabel('ঠিকানা')).toHaveValue(address);
  });

  test('বাংলা যুক্তাক্ষর সঠিকভাবে প্রদর্শিত হচ্ছে', async ({ page }) => {
    await page.goto('/bn/terms');

    // যুক্তাক্ষর সহ বাংলা টেক্সট যাচাই
    const complexWords = ['বিশ্ববিদ্যালয়', 'শিক্ষার্থী', 'রাষ্ট্র', 'যুক্তরাষ্ট্র'];
    for (const word of complexWords) {
      await expect(page.getByText(word).first()).toBeVisible();
    }
  });

  test('বাংলা সংখ্যা (১২৩) এবং ইংরেজি সংখ্যা (123) উভয়ই কাজ করবে', async ({ page }) => {
    await page.goto('/checkout');

    // বাংলা সংখ্যায় ফোন দিলে ত্রুটি/রূপান্তর হওয়া উচিত
    await page.getByLabel('ফোন').fill('০১৭১২৩৪৫৬৭৮');
    await page.getByLabel('ফোন').blur();

    // অ্যাপ হয় ইংরেজিতে রূপান্তর করবে, নয়তো বার্তা দেখাবে
    const phoneValue = await page.getByLabel('ফোন').inputValue();
    const isConverted = phoneValue === '01712345678';
    const hasError = await page.locator('[data-error="phone"]').isVisible();

    expect(isConverted || hasError).toBeTruthy();
  });
});
```

```php
// Laravel Dusk — বাংলা কন্টেন্ট পরীক্ষা
class BanglaContentTest extends DuskTestCase
{
    public function test_bangla_product_name_display(): void
    {
        $product = Product::factory()->create([
            'name' => 'হ্যান্ডমেইড জামদানি শাড়ি',
            'description' => 'ঐতিহ্যবাহী বাংলাদেশি জামদানি শাড়ি।'
        ]);

        $this->browse(function (Browser $browser) use ($product) {
            $browser->visit("/products/{$product->id}")
                    ->assertSee('হ্যান্ডমেইড জামদানি শাড়ি')
                    ->assertSee('ঐতিহ্যবাহী বাংলাদেশি জামদানি শাড়ি।');
        });
    }

    public function test_bangla_invoice_generation(): void
    {
        $order = Order::factory()->create(['total' => 5000]);

        $this->browse(function (Browser $browser) use ($order) {
            $browser->loginAs($order->user)
                    ->visit("/orders/{$order->id}/invoice")
                    ->assertSee('চালান')
                    ->assertSee('মোট মূল্য')
                    ->assertSee('৳৫,০০০');
        });
    }
}
```

---

## 🎯 সারসংক্ষেপ

```
E2E টেস্টিং সফল করার মূলমন্ত্র:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ কম কিন্তু গুরুত্বপূর্ণ টেস্ট লিখুন
✦ Critical business flows কভার করুন
✦ data-testid বা role-based সিলেক্টর ব্যবহার করুন
✦ API দিয়ে state সেটআপ করুন, UI দিয়ে নয়
✦ CI/CD-তে প্যারালেল চালান
✦ Flaky টেস্ট চিহ্নিত করে ঠিক করুন
✦ Page Object Model অনুসরণ করুন
✦ প্রতিটি টেস্ট স্বাধীন ও নির্ভরতামুক্ত রাখুন
```

---

> **লেখকের নোট**: এই গাইডটি বাংলাদেশি ডেভেলপারদের জন্য তৈরি, যেখানে বিকাশ/নগদ পেমেন্ট, বাংলা ইনপুট, এবং স্থানীয় প্রসঙ্গ অন্তর্ভুক্ত করা হয়েছে। প্রতিটি কোড উদাহরণ প্রোডাকশন-রেডি প্যাটার্ন অনুসরণ করে।
