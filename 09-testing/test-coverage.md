# 📊 টেস্ট কভারেজ (Test Coverage) — গভীর বিশ্লেষণ

## 📌 সংজ্ঞা — টেস্ট কভারেজ কী

**টেস্ট কভারেজ** হলো একটি মেট্রিক যা নির্দেশ করে আপনার টেস্ট সুইট সোর্স কোডের কতটুকু execute করেছে। শতকরা হারে প্রকাশ করা হয়।

**কেন প্রয়োজন:** ঝুঁকি চিহ্নিতকরণ, রিফ্যাক্টরিং-এ আত্মবিশ্বাস, রিগ্রেশন প্রতিরোধ, PR রিভিউ সহায়তা, এবং ব্যাংকিং/ফিনটেক কমপ্লায়েন্স।

### ⚠️ কেন ১০০% কভারেজ লক্ষ্য নয়

```
✗ কোড execute হওয়া ≠ সঠিকভাবে verify হওয়া
✗ Assertion ছাড়া টেস্ট কভারেজ বাড়ায় কিন্তু কিছু prove করে না
✗ Edge case, race condition ধরা পড়ে না
✗ Diminishing returns — ৮০% থেকে ১০০% যেতে অসম পরিশ্রম
```

```php
// ❌ ১০০% কভারেজ কিন্তু কিছুই verify হচ্ছে না
public function testCalculateDiscount(): void {
    $service = new PricingService();
    $service->calculateDiscount(1000, 'VIP'); // assertion নেই!
}

// ✅ আচরণ verify করুন
public function testVipGets20PercentDiscount(): void {
    $service = new PricingService();
    $this->assertSame(200.0, $service->calculateDiscount(1000, 'VIP'));
}
```

---

## 📊 কভারেজ টাইপসমূহ

### ১. লাইন কভারেজ (Line Coverage)

সবচেয়ে সাধারণ মেট্রিক। `(Execute হওয়া লাইন / মোট executable লাইন) × ১০০%`

```php
class TaxCalculator {
    public function calculate(float $amount, string $region): float
    {                                          // ✅ executed
        if ($amount <= 0) {                    // ✅ executed
            throw new \InvalidArgumentException('Amount must be positive');
        }
        return match ($region) {               // ✅ executed
            'BD' => $amount * 0.15,            // ✅ executed
            'US' => $amount * 0.08,            // ❌ NOT executed
            'UK' => $amount * 0.20,            // ❌ NOT executed
            default => $amount * 0.10,         // ❌ NOT executed
        };
    }
}
// Coverage: 5/8 executable lines = 62.5%
```

### ২. ব্রাঞ্চ কভারেজ (Branch Coverage)

প্রতিটি decision point-এর true/false উভয় শাখা execute হয়েছে কিনা পরিমাপ করে। **লাইন কভারেজের চেয়ে বেশি কার্যকর।**

```javascript
function processPayment(amount, currency, isVip) {
    let fee = 0;
    if (amount > 10000) { fee = amount * 0.02; }  // Branch 1: T✅ F✅
    else { fee = amount * 0.05; }
    if (isVip) { fee *= 0.5; }                     // Branch 2: T✅ F❌
    if (currency !== 'BDT') { fee += 50; }         // Branch 3: T❌ F✅
    return fee;
}
// Branch Coverage: 4/6 = 66.7%
// Line Coverage হয়তো 100% দেখাবে — কিন্তু branch বলছে ঝুঁকি আছে!
```

### ৩. ফাংশন/মেথড কভারেজ

`(কল হওয়া ফাংশন / মোট ফাংশন) × ১০০%` — Dead code চিহ্নিত করতে কার্যকর।

```
┌──────────────────┬────────┐
│ Method           │ Status │
├──────────────────┼────────┤
│ charge()         │   ✅   │
│ refund()         │   ✅   │
│ void()           │   ❌   │
│ getStatement()   │   ❌   │
└──────────────────┴────────┘  Function Coverage: 2/4 = 50%
```

### ৪. স্টেটমেন্ট কভারেজ

লাইন কভারেজের মতো কিন্তু একটি লাইনে একাধিক statement থাকতে পারে:

```javascript
const result = validate(input) && process(input) && save(input);
// validate() false হলে পরের দুটি execute হবে না — line covered কিন্তু ২টি statement missed
```

### ৫. কন্ডিশন কভারেজ ও MC/DC

**MC/DC** সবচেয়ে কঠোর মেট্রিক — এভিয়েশন (DO-178C) ও ক্রিটিক্যাল সিস্টেমে বাধ্যতামূলক।

```php
function isEligibleForLoan(bool $hasIncome, bool $goodCredit, bool $noDefault): bool {
    return $hasIncome && ($goodCredit || $noDefault);
}
```

```
MC/DC: প্রতিটি condition স্বাধীনভাবে decision-এর ফলাফল পরিবর্তন করে দেখাতে হবে
┌──────┬─────────┬─────────┬─────────┬────────┐
│ Test │ Income  │ Credit  │ NoDef   │ Result │
├──────┼─────────┼─────────┼─────────┼────────┤
│  T1  │  true   │  true   │  true   │  true  │
│  T2  │  false  │  true   │  true   │  false │  ← Income toggles
│  T3  │  true   │  false  │  false  │  false │  ← Both B,C off
│  T4  │  true   │  false  │  true   │  true  │  ← NoDef toggles
└──────┴─────────┴─────────┴─────────┴────────┘
ন্যূনতম N+1 টেস্ট কেস লাগে (N = condition সংখ্যা)
```

### ৬. পাথ কভারেজ

প্রতিটি সম্ভাব্য execution path কভার করতে হয়। ৩টি if-এ `2³ = ৮` পাথ, ১০টিতে ১০২৪টি — লুপে অসীম। তাত্ত্বিকভাবে সবচেয়ে শক্তিশালী কিন্তু বাস্তবে অসম্ভব।

---

## 💻 টুলস এবং সেটআপ

### PHPUnit + Xdebug/PCOV

```xml
<!-- phpunit.xml -->
<phpunit bootstrap="vendor/autoload.php" colors="true">
    <testsuites>
        <testsuite name="Unit"><directory>tests/Unit</directory></testsuite>
    </testsuites>
    <coverage>
        <report>
            <html outputDirectory="coverage/html" lowUpperBound="50" highLowerBound="80"/>
            <clover outputFile="coverage/clover.xml"/>
            <text outputFile="coverage/coverage.txt" showOnlySummary="true"/>
            <cobertura outputFile="coverage/cobertura.xml"/>
        </report>
        <include><directory suffix=".php">src</directory></include>
        <exclude>
            <directory>src/Migrations</directory>
            <file>src/Kernel.php</file>
        </exclude>
    </coverage>
</phpunit>
```

**PCOV vs Xdebug:** PCOV ~4x দ্রুত কিন্তু শুধু line coverage দেয়। CI-তে PCOV, branch analysis-এ Xdebug ব্যবহার করুন।

```php
// @covers অ্যানোটেশন — সুনির্দিষ্ট কভারেজ ম্যাপিং (PHPUnit 10+)
#[CoversClass(InvoiceService::class)]
class InvoiceServiceTest extends TestCase
{
    #[CoversMethod(InvoiceService::class, 'calculateTotal')]
    public function testCalculateTotalWithVat(): void
    {
        $service = new InvoiceService(new PdfRenderer(), new TaxCalculator());
        $total = $service->calculateTotal(
            items: [['name' => 'ল্যাপটপ', 'price' => 85000, 'qty' => 1]],
            vatRate: 0.15
        );
        $this->assertSame(97750.0, $total); // 85000 * 1.15
    }
}
```

### Jest + Istanbul (c8)

```javascript
// jest.config.js
module.exports = {
    collectCoverageFrom: [
        'src/**/*.{js,ts}',
        '!src/**/*.d.ts',
        '!src/migrations/**',
    ],
    coverageReporters: ['text', 'lcov', 'clover', 'json-summary'],
    coverageThreshold: {
        global: { branches: 75, functions: 80, lines: 80, statements: 80 },
        './src/services/payment/': {
            branches: 90, functions: 95, lines: 90, statements: 90,
        },
    },
};
```

```javascript
// tests/bkash-payment.test.js
const BkashPaymentService = require('../src/services/bkash-payment');

describe('BkashPaymentService', () => {
    let service, mockApi, mockLogger;

    beforeEach(() => {
        mockApi = {
            getToken: jest.fn().mockResolvedValue('token'),
            executePayment: jest.fn().mockResolvedValue({ trxId: 'TRX123' }),
        };
        mockLogger = { info: jest.fn(), error: jest.fn() };
        service = new BkashPaymentService(mockApi, mockLogger);
    });

    it('should process valid payment', async () => {
        const result = await service.processPayment('01712345678', 500, 'REF001');
        expect(result).toEqual({ success: true, transactionId: 'TRX123' });
    });

    it('should reject invalid phone', async () => {
        await expect(service.processPayment('02012345678', 500, 'REF002'))
            .rejects.toThrow('Invalid bKash number');
    });

    it('should handle API failure', async () => {
        mockApi.executePayment.mockRejectedValue(new Error('Timeout'));
        const result = await service.processPayment('01712345678', 500, 'REF003');
        expect(result).toEqual({ success: false, error: 'Timeout' });
    });
});
```

### SonarQube ইন্টিগ্রেশন

```properties
# sonar-project.properties
sonar.projectKey=my-fintech-app
sonar.sources=src
sonar.tests=tests
sonar.php.coverage.reportPaths=coverage/clover.xml
sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

### Codecov CI ইন্টিগ্রেশন

```yaml
# .github/workflows/coverage.yml
name: Coverage
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.3', coverage: pcov }
      - run: composer install && vendor/bin/phpunit --coverage-clover coverage.xml
      - uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
```

```yaml
# codecov.yml
coverage:
  status:
    project:
      default: { target: 80%, threshold: 2% }
    patch:
      default: { target: 85% }
```

---

## 🔥 অ্যাডভান্সড কনসেপ্ট

### ১. কভারেজ বনাম কনফিডেন্স

```
উচ্চ কভারেজ ≠ ভালো টেস্ট
কভারেজ পরিমাণ নির্দেশ করে, গুণমান নয়। মিউটেশন টেস্টিং গুণমান পরিমাপ করে।
```

### ২. মিউটেশন টেস্টিং

কোডে ইচ্ছাকৃত ত্রুটি (mutants) প্রবেশ করিয়ে দেখে টেস্ট সুইট ধরতে পারে কিনা।

```
মূল কোড: if (a > b)  →  Mutant: if (a >= b)  →  টেস্ট fail? হ্যাঁ=Killed✅ না=Survived❌
MSI = (Killed Mutants / Total Mutants) × ১০০%
```

**PHP — Infection:**

```json5
// infection.json5
{
    "source": { "directories": ["src"], "excludes": ["Migrations"] },
    "minMsi": 75,
    "minCoveredMsi": 90,
    "testFramework": "phpunit"
}
```

```bash
vendor/bin/infection --threads=4 --min-msi=75 --show-mutations
# শুধু পরিবর্তিত কোডে:
vendor/bin/infection --git-diff-filter=AM --git-diff-base=origin/main
```

**JavaScript — Stryker:**

```javascript
// stryker.config.js
module.exports = {
    testRunner: 'jest',
    coverageAnalysis: 'perTest',
    thresholds: { high: 80, low: 60, break: 50 },
    mutate: ['src/**/*.js', '!src/**/*.test.js'],
};
```

### ৩. CI/CD-তে কভারেজ থ্রেশহোল্ড

```yaml
- name: Check Coverage Threshold
  run: |
    COVERAGE=$(grep -oP 'Lines:\s+\K[\d.]+' phpunit-output.txt)
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "::error::Coverage ${COVERAGE}% is below 80% threshold!"
      exit 1
    fi
```

### ৪. ডিফারেনশিয়াল/PR কভারেজ

শুধু পরিবর্তিত ফাইলের কভারেজ চেক — `codecov.yml`-এ `patch` target ব্যবহার করুন। নতুন কোডে ৯০% enforce করলে legacy কোডের চাপ থাকে না।

### ৫. টেস্ট টাইপ অনুযায়ী কভারেজ

```
Unit Tests (৮০% effort)   → সর্বোচ্চ line/branch coverage, ≥85%
Integration (১৫%)         → module interaction, ≥65%
E2E (৫%)                  → system-level, কভারেজ সাধারণত নেওয়া হয় না
```

### ৬. কোড কভারেজ থেকে বাদ দেওয়া

```php
/** @codeCoverageIgnore — auto-generated migration */
class Version20240101 extends AbstractMigration { /* ... */ }

// নির্দিষ্ট ব্লক:
// @codeCoverageIgnoreStart
if (PHP_SAPI === 'cli') { exit(1); }
// @codeCoverageIgnoreEnd
```

```javascript
/* istanbul ignore next */
if (process.env.NODE_ENV === 'development') { enableDevTools(); }

/* c8 ignore start */
function debugOnly() { console.log('debug'); }
/* c8 ignore stop */
```

### ৭. GitHub Actions-এ কভারেজ রিপোর্ট

```yaml
- uses: irongut/CodeCoverageSummary@v1.3.0
  with:
    filename: coverage.xml
    format: markdown
    badge: true
    thresholds: '60 80'
- uses: marocchino/sticky-pull-request-comment@v2
  with: { path: code-coverage-results.md }
```

### ৮. কভারেজ ব্যাজ

```yaml
- uses: jaywcjlove/coverage-badges-cli@main
  with:
    source: coverage/coverage-summary.json
    output: coverage/badge.svg
```

```markdown
![Coverage](https://img.shields.io/badge/coverage-85%25-green)
```

### ৯. বাস্তব কভারেজ লক্ষ্যমাত্রা

```
┌─────────────────────┬──────────┬──────────┬───────────────────────┐
│ প্রজেক্ট টাইপ       │ Line     │ Branch   │ মন্তব্য               │
├─────────────────────┼──────────┼──────────┼───────────────────────┤
│ Startup MVP         │ 50-60%   │ 40-50%   │ ক্রিটিক্যাল পাথে ফোকাস│
│ SaaS Product        │ 70-80%   │ 60-70%   │ মূল business logic    │
│ Open Source Library  │ 85-95%   │ 80-90%   │ পাবলিক API ১০০%      │
│ FinTech/Banking(BD) │ 85-95%   │ 80-90%   │ BB/BFIU compliance    │
│ Safety-Critical     │ 100%MC/DC│ 100%     │ DO-178C / IEC 62304   │
└─────────────────────┴──────────┴──────────┴───────────────────────┘

🇧🇩 বাংলাদেশ কনটেক্সট:
• বাংলাদেশ ব্যাংকের ICT নির্দেশিকায় core banking-এ পর্যাপ্ত টেস্টিং বাধ্যতামূলক
• MFS (bKash, Nagad) পেমেন্ট কোডে ≥90% কভারেজ সুপারিশকৃত
• BFIU compliance-এ transaction monitoring কোডের কভারেজ audit হতে পারে
• PCI-DSS মেনে চলা প্রতিষ্ঠানে card processing-এ branch coverage সহ ≥85%
```

---

## ✅ বেস্ট প্র্যাকটিসেস

```php
// ❌ শুধু কভারেজ বাড়াতে getter/setter টেস্ট
$user->setName('আরিফ');
$this->assertSame('আরিফ', $user->getName());

// ✅ ব্যবসায়িক আচরণ টেস্ট করুন
$this->expectException(DuplicateEmailException::class);
$this->repo->save(new User(email: 'arif@test.com')); // duplicate
```

**মূল নীতিমালা:**
1. কভারেজকে গাইড হিসেবে ব্যবহার করুন, লক্ষ্য নয়
2. **Ratchet পদ্ধতি** — কভারেজ শুধু উপরে যাবে, নিচে নয়
3. নতুন কোডে ≥90% patch coverage enforce করুন
4. কভারেজ + মিউটেশন টেস্টিং একসাথে ব্যবহার করুন
5. `@codeCoverageIgnore` প্রতিটি ব্যবহারে কারণ লিখুন

---

## ⚠️ কভারেজ অ্যান্টি-প্যাটার্নস

| অ্যান্টি-প্যাটার্ন | সমস্যা | সমাধান |
|---|---|---|
| Assertion-less tests | কভারেজ বাড়ে কিন্তু কিছু verify হয় না | প্রতিটি টেস্টে meaningful assertion রাখুন |
| Getter/setter testing | কভারেজ ২০%+ বাড়ে কিন্তু শূন্য মূল্য | Business logic টেস্ট করুন |
| Tautological assertion | `expect(x).toBe(x)` সবসময় pass | Input-output relationship verify করুন |
| Coverage-driven dev | Uncovered line দেখে টেস্ট লেখা | Behavior-driven approach নিন |
| Excessive ignoring | কঠিন কোড ignore করা | Refactor to testable design |

---

## 📋 কভারেজ টাইপ তুলনা সারণী

```
┌──────────────┬────────────────┬──────────────────┬────────────────────┐
│ টাইপ         │ কী পরিমাপ করে  │ শক্তি            │ কখন ব্যবহার       │
├──────────────┼────────────────┼──────────────────┼────────────────────┤
│ Line         │ Execute হওয়া   │ সহজ, দ্রুত      │ দৈনন্দিন dev       │
│ Branch       │ true/false path│ if/else ধরে      │ CI/CD মেট্রিক     │
│ Function     │ কল হওয়া ফাংশন │ Dead code চিহ্নিত│ API surface        │
│ Statement    │ প্রতিটি stmt   │ Multi-stmt line  │ বিস্তারিত বিশ্লেষণ│
│ MC/DC        │ Sub-conditions │ সবচেয়ে নির্ভুল   │ Safety-critical    │
│ Path         │ সব exec path   │ সর্বোচ্চ শক্তি   │ তাত্ত্বিক (অসম্ভব)│
└──────────────┴────────────────┴──────────────────┴────────────────────┘

শক্তি ক্রম: Path > MC/DC > Branch > Condition > Statement/Line > Function
১০০% Branch → ১০০% Line (guaranteed), কিন্তু উল্টোটা সত্য নয়।
```

## 📝 সারসংক্ষেপ

- কভারেজ **ডায়াগনস্টিক টুল**, লক্ষ্য নয় — "কোথায় টেস্ট নেই?" জানতে ব্যবহার করুন
- **Branch > Line** — শুধু line coverage-এ সন্তুষ্ট হবেন না
- **কভারেজ + মিউটেশন = সম্পূর্ণ ছবি** — কভারেজ=পরিমাণ, মিউটেশন=গুণমান
- ক্রিটিক্যাল কোডে (পেমেন্ট, সিকিউরিটি) ≥৯০%, UI/config-এ ৬০-৭০% যথেষ্ট
- CI/CD-তে Codecov/SonarQube + threshold + PR comment স্বয়ংক্রিয় করুন
- 🇧🇩 FinTech-এ BB/BFIU compliance ও MFS audit readiness নিশ্চিত করুন

> **"কভারেজ বলে কোথায় টেস্ট নেই — মিউটেশন টেস্টিং বলে টেস্ট কতটা ভালো।"**
