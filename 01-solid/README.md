# 📘 SOLID প্রিন্সিপল

> অবজেক্ট-ওরিয়েন্টেড ডিজাইনের ৫টি মূলনীতি যা মেইনটেইনেবল ও স্কেলেবল কোড তৈরিতে সাহায্য করে।
> সব টপিক বাংলায়, PHP ও JavaScript উদাহরণসহ।

---

## 📂 টপিক সূচি

| # | প্রিন্সিপল | ফাইল | সংক্ষেপ |
|---|-----------|------|--------|
| ১ | **S** — Single Responsibility Principle | [single-responsibility.md](./single-responsibility.md) | একটি ক্লাসের একটাই দায়িত্ব থাকবে |
| ২ | **O** — Open/Closed Principle | [open-closed.md](./open-closed.md) | এক্সটেনশনের জন্য খোলা, মডিফিকেশনের জন্য বন্ধ |
| ৩ | **L** — Liskov Substitution Principle | [liskov-substitution.md](./liskov-substitution.md) | চাইল্ড ক্লাস প্যারেন্টের জায়গায় ব্যবহারযোগ্য হবে |
| ৪ | **I** — Interface Segregation Principle | [interface-segregation.md](./interface-segregation.md) | ছোট ছোট নির্দিষ্ট ইন্টারফেস তৈরি করুন |
| ৫ | **D** — Dependency Inversion Principle | [dependency-inversion.md](./dependency-inversion.md) | কনক্রিটের বদলে অ্যাবস্ট্রাকশনের উপর নির্ভর করুন |

---

## 🔄 SOLID প্রিন্সিপলগুলোর পারস্পরিক সম্পর্ক

```
    SRP ────────────► OCP
  (আলাদা দায়িত্ব)    (আলাদাভাবে এক্সটেন্ড)
    │                   ▲
    │                   │
    ▼                   │
   ISP ◄────────────── LSP
 (ছোট ইন্টারফেস)    (সঠিক substitution)
    │                   │
    └────► DIP ◄────────┘
       (abstraction এর উপর নির্ভর)
```

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ সংজ্ঞা ও বিস্তারিত ব্যাখ্যা (বাংলায়)
- ✅ বাস্তব জীবনের উদাহরণ (analogy)
- ✅ ❌ খারাপ কোড উদাহরণ (PHP + JS) — কেন ভুল তার ব্যাখ্যা
- ✅ ✅ ভালো কোড উদাহরণ (PHP + JS) — সঠিক পদ্ধতি
- ✅ গভীর বিশ্লেষণ (Deep Dive) — অ্যাডভান্সড কৌশল
- ✅ সুবিধা (Pros) ও অসুবিধা (Cons) টেবিল
- ✅ সাধারণ ভুল (Common Mistakes)
- ✅ ইউনিট টেস্ট উদাহরণ (PHPUnit + Jest)
- ✅ কখন করবেন / করবেন না
- ✅ অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক
