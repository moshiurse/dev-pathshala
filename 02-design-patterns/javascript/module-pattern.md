# 📦 মডিউল প্যাটার্ন (Module Pattern)

> **ক্যাটাগরি:** JavaScript Design Pattern
> **উৎস:** patterns.dev/vanilla/module-pattern

---

## সংজ্ঞা

**মডিউল প্যাটার্ন** কোডকে ছোট, পুনঃব্যবহারযোগ্য এবং _এনক্যাপসুলেটেড_ অংশে ভাগ করে। মডিউলের ভেতরে ঘোষিত ভ্যারিয়েবল ও ফাংশন ডিফল্টভাবে সেই মডিউলের মধ্যেই সীমাবদ্ধ — যা গ্লোবাল স্কোপ দূষণ রোধ করে।

```
   মডিউল প্যাটার্নের মূল ধারণা
   ══════════════════════════════════════════════════

   ┌─────────────────────────────────────────────┐
   │              math.js (মডিউল)               │
   │                                             │
   │  🔒 privateValue = "শুধু এখানে"            │
   │  ✅ export function add() { }              │
   │  ✅ export function multiply() { }         │
   │  🔒 export না করলে বাইরে যাবে না           │
   └─────────────────────────────────────────────┘
              ↓ import
   ┌─────────────────────────────────────────────┐
   │              index.js                       │
   │                                             │
   │  import { add, multiply } from './math.js' │
   │  ❌ privateValue → ReferenceError           │
   └─────────────────────────────────────────────┘
```

---

## 🏠 বাস্তব উদাহরণ (Bangladesh Context)

> bKash অ্যাপে পেমেন্ট লজিক, ভ্যালিডেশন, ফরম্যাটিং — প্রতিটি আলাদা মডিউলে থাকে।
> একটা মডিউলের ভেতরের API key বা secret অন্য মডিউল দেখতে পারে না।

---

## ✅ ES2015 মডিউল — Named Export

```javascript
// math.js
const PI = 3.14159; // প্রাইভেট — export নেই

export function add(x, y) {
  return x + y;
}

export function multiply(x, y) {
  return x * y;
}

export function circleArea(r) {
  return PI * r * r; // ভেতরে PI ব্যবহার হচ্ছে, কিন্তু বাইরে যাচ্ছে না
}
```

```javascript
// index.js
import { add, multiply, circleArea } from './math.js';

console.log(add(5, 3));       // 8
console.log(multiply(4, 2));  // 8
console.log(circleArea(7));   // 153.938...

// console.log(PI); // ❌ ReferenceError: PI is not defined
```

---

## ✅ Default Export

```javascript
// logger.js
class Logger {
  #prefix;

  constructor(prefix) {
    this.#prefix = prefix;
  }

  log(message) {
    console.log(`[${this.#prefix}] ${message}`);
  }
}

export default Logger;
```

```javascript
// app.js
import Logger from './logger.js';

const logger = new Logger('bKash');
logger.log('ট্রানজেকশন সম্পন্ন'); // [bKash] ট্রানজেকশন সম্পন্ন
```

---

## ✅ Rename & Re-export (Barrel Pattern)

```javascript
// services/index.js — সব সার্ভিস এক জায়গা থেকে এক্সপোর্ট
export { PaymentService } from './payment.js';
export { UserService }    from './user.js';
export { SMSService }     from './sms.js';
```

```javascript
// app.js — এক import থেকে সব পাওয়া যাচ্ছে
import { PaymentService, UserService } from './services/index.js';
```

---

## ✅ Dynamic Import (Lazy Loading)

```javascript
// ভারী মডিউল শুধু প্রয়োজনে লোড করুন
async function loadChart() {
  const { Chart } = await import('./charts.js');
  return new Chart();
}

// React-এ lazy()
const Dashboard = React.lazy(() => import('./Dashboard'));
```

---

## ✅ IIFE Module (পুরনো প্যাটার্ন, ES Modules আগে)

```javascript
// ES Modules আগে এভাবে প্রাইভেসি রাখা হতো
const CounterModule = (() => {
  let count = 0; // প্রাইভেট

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
})();

CounterModule.increment();
CounterModule.increment();
console.log(CounterModule.getCount()); // 2
console.log(CounterModule.count);      // undefined (প্রাইভেট)
```

---

## ❌ সমস্যা vs ✅ সমাধান

```javascript
// ❌ গ্লোবাল স্কোপে সব রাখা
var TAX_RATE = 0.15;
var calculateTax = function(amount) { return amount * TAX_RATE; };
var formatBDT  = function(amount) { return '৳' + amount; };
// যেকোনো ফাইল TAX_RATE পরিবর্তন করতে পারবে!

// ✅ মডিউলে এনক্যাপসুলেট করুন
// tax.js
const TAX_RATE = 0.15; // প্রাইভেট

export function calculateTax(amount) {
  return amount * TAX_RATE;
}

export function formatBDT(amount) {
  return `৳${amount.toLocaleString('bn-BD')}`;
}
```

---

## 🔄 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | সম্পর্ক |
|----------|---------|
| Singleton | মডিউল ফাইলই একটি Singleton — একবার লোড হলে ক্যাশ হয় |
| Facade | মডিউল একটি Facade হিসেবে কাজ করতে পারে |
| Factory | মডিউল থেকে Factory ফাংশন এক্সপোর্ট করা যায় |
| Observer | EventEmitter মডিউল হিসেবে এক্সপোর্ট করা যায় |

---

## ✅ সুবিধা (Pros)

- **এনক্যাপসুলেশন** — প্রাইভেট স্কোপ নিশ্চিত
- **নামের সংঘর্ষ নেই** — গ্লোবাল দূষণ রোধ
- **রিউজেবল** — একাধিক ফাইলে ইম্পোর্ট করা যায়
- **ট্রি-শেকিং** — অব্যবহৃত এক্সপোর্ট বান্ডেল থেকে বাদ যায়

## ❌ অসুবিধা (Cons)

- **Circular dependency** হতে পারে
- **Barrel ফাইল** অতিরিক্ত ব্যবহারে slow bundling

---

## 📝 কখন ব্যবহার করবেন

- ✅ সব জায়গায় — ES Modules হলো আধুনিক JavaScript-এর ভিত্তি
- ✅ প্রাইভেট হেল্পার ফাংশন বা কনস্ট্যান্ট লুকাতে
- ✅ লাইব্রেরি বা ইউটিলিটি তৈরিতে
- ❌ খুব ছোট স্ক্রিপ্টে (single file project)
