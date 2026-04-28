# 📘 টেস্টিং

> সফটওয়্যার টেস্টিংয়ের বিভিন্ন স্তর ও কৌশল।
> প্রতিটি টপিকে PHPUnit (PHP) ও Jest (JS) উদাহরণ, Advanced patterns, এবং CI/CD integration সহ গভীর বিশ্লেষণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | ইউনিট টেস্টিং | [unit-testing.md](./unit-testing.md) | FIRST principles, PHPUnit Attributes, Jest matchers, DataProvider, Snapshot, Mutation Testing |
| ২ | ইন্টিগ্রেশন টেস্টিং | [integration-testing.md](./integration-testing.md) | API/DB/Queue/Cache Testing, Contract Testing (Pact), Docker Compose, CI/CD |
| ৩ | E2E টেস্টিং | [e2e-testing.md](./e2e-testing.md) | Cypress, Playwright, Laravel Dusk, Visual Regression, Accessibility (axe-core) |
| ৪ | TDD | [tdd.md](./tdd.md) | Red-Green-Refactor, Inside-Out vs Outside-In, BDD (Behat/Cucumber), TDD Kata |
| ৫ | মকিং ও স্টাবিং | [mocking.md](./mocking.md) | ৫ Test Doubles (Dummy/Stub/Spy/Mock/Fake), Mockery, Laravel Fakes, Over-mocking |
| ৬ | টেস্ট কভারেজ | [test-coverage.md](./test-coverage.md) | ৬ Coverage Types, Mutation Testing (Infection/Stryker), CI Thresholds, SonarQube |
| ৭ | পারফরম্যান্স টেস্টিং | [performance-testing.md](./performance-testing.md) | Load/Stress Testing, k6, JMeter, Artillery |
| ৮ | কন্ট্র্যাক্ট টেস্টিং | [contract-testing.md](./contract-testing.md) | Consumer-Driven Contracts, Pact Framework |
| ৯ | মিউটেশন টেস্টিং | [mutation-testing.md](./mutation-testing.md) | Mutators, Kill Score, Infection, Stryker |

---

## 🗺️ টেস্টিং পিরামিড

```
          ┌───────────┐
          │   E2E     │  ← কম সংখ্যক, ধীর, ব্যয়বহুল
          │ (Cypress/  │
          │ Playwright)│
          ├───────────┤
          │Integration │  ← মাঝারি সংখ্যক
          │ (API/DB    │
          │  Tests)    │
          ├───────────┤
          │   Unit     │  ← বেশি সংখ্যক, দ্রুত, সস্তা
          │ (PHPUnit/  │
          │   Jest)    │
          └───────────┘
```

---

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ বাংলায় বিস্তারিত ব্যাখ্যা
- ✅ PHPUnit (PHP) ও Jest (JS) কোড উদাহরণ
- ✅ Advanced patterns (TDD, Mutation Testing, Contract Testing)
- ✅ CI/CD integration (GitHub Actions)
- ✅ Best Practices ও Anti-patterns
- ✅ বাংলাদেশ context (bKash validation, BDT calculations)
