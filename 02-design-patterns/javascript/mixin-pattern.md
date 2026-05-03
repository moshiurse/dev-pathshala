# 🧬 মিক্সিন প্যাটার্ন (Mixin Pattern)

> **ক্যাটাগরি:** JavaScript Design Pattern
> **উৎস:** patterns.dev/vanilla/mixin-pattern

---

## সংজ্ঞা

**মিক্সিন** হলো এমন একটি অবজেক্ট/ক্লাস যা অন্য ক্লাসে তার মেথড যোগ করতে পারে — উত্তরাধিকার (inheritance) ছাড়াই। এটি "Composition over Inheritance" নীতির একটি বাস্তব প্রয়োগ।

```
   মিক্সিন vs ইনহেরিটেন্স
   ══════════════════════════════════════════════════════

   ❌ ইনহেরিটেন্স (শক্ত কাপলিং)
   Animal → Bird → FlyingBird → SwimmingFlyingBird ??? 🤯

   ✅ মিক্সিন (নমনীয় কম্পোজিশন)
   canFly   = { fly() }
   canSwim  = { swim() }
   canWalk  = { walk() }
   
   Duck = Class + canFly + canSwim + canWalk ✅
```

---

## 🏠 বাস্তব উদাহরণ (Bangladesh Context)

> Pathao-এর রাইডার-অ্যাপে `canReceiveOrder`, `canTrackLocation`, `canChat` — এগুলো আলাদা mixin।
> Rider, Courier, FoodDelivery — সবাই প্রয়োজন মতো মিক্সিন নেয়।

---

## ✅ Object.assign() দিয়ে মিক্সিন

```javascript
// মিক্সিনগুলো সাধারণ অবজেক্ট
const Serializable = {
  serialize() {
    return JSON.stringify(this);
  },
  deserialize(data) {
    return Object.assign(this, JSON.parse(data));
  },
};

const Validatable = {
  validate() {
    return Object.keys(this).every(key => this[key] !== null);
  },
};

const Timestampable = {
  setTimestamps() {
    this.createdAt = new Date().toISOString();
    this.updatedAt = new Date().toISOString();
  },
  touch() {
    this.updatedAt = new Date().toISOString();
  },
};

// User ক্লাসে মিক্সিন যোগ করা
class User {
  constructor(name, email) {
    this.name  = name;
    this.email = email;
  }
}

Object.assign(User.prototype, Serializable, Validatable, Timestampable);

const user = new User('আহমেদ', 'ahmed@example.com');
user.setTimestamps();
console.log(user.validate());  // true
console.log(user.serialize()); // JSON string
```

---

## ✅ ফাংশনাল মিক্সিন (আধুনিক পদ্ধতি)

```javascript
// ফাংশনাল মিক্সিন — ক্লাস থেকে ক্লাস তৈরি করে
const withLogging = (Base) => class extends Base {
  log(message) {
    console.log(`[${this.constructor.name}] ${message}`);
  }
};

const withRetry = (Base) => class extends Base {
  async retryAsync(fn, maxAttempts = 3) {
    for (let i = 0; i < maxAttempts; i++) {
      try {
        return await fn();
      } catch (err) {
        if (i === maxAttempts - 1) throw err;
        await new Promise(r => setTimeout(r, 1000 * (i + 1)));
      }
    }
  }
};

const withCache = (Base) => class extends Base {
  #cache = new Map();

  cached(key, fn) {
    if (this.#cache.has(key)) return this.#cache.get(key);
    const result = fn();
    this.#cache.set(key, result);
    return result;
  }
};

// বেস সার্ভিস
class PaymentService {
  async processPayment(amount) {
    // পেমেন্ট লজিক
    return { success: true, amount };
  }
}

// মিক্সিন কম্পোজ করুন
class EnhancedPaymentService extends withCache(withRetry(withLogging(PaymentService))) {
  async pay(amount) {
    this.log(`Processing payment: ৳${amount}`);
    return this.retryAsync(() => this.processPayment(amount));
  }
}

const service = new EnhancedPaymentService();
service.pay(500);
```

---

## ✅ React-এ মিক্সিন (Custom Hooks দিয়ে)

> React-এ class mixin বাতিল। এখন **Custom Hooks** মিক্সিনের কাজ করে।

```javascript
// useLocalStorage — স্টেট লোকাল স্টোরেজে সেভ করে
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  const setAndStore = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, setAndStore];
}

// useDebounce — ইনপুট ডিবাউন্স করে
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// যেকোনো কমপোনেন্টে মিক্সিনের মতো ব্যবহার
function SearchComponent() {
  const [query, setQuery]       = useLocalStorage('lastSearch', '');
  const debouncedQuery          = useDebounce(query, 400);

  useEffect(() => {
    if (debouncedQuery) fetchResults(debouncedQuery);
  }, [debouncedQuery]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

---

## ❌ সমস্যা vs ✅ সমাধান

```javascript
// ❌ গভীর ইনহেরিটেন্স চেইন
class Animal {}
class LivingThing extends Animal {}
class Creature extends LivingThing {}
class Pet extends Creature {}
class Dog extends Pet {} // কোনটা কোথায় আছে বোঝা কঠিন

// ✅ মিক্সিন দিয়ে কম্পোজিশন
const canEat   = { eat()   { return 'খাচ্ছি'; } };
const canSleep = { sleep() { return 'ঘুমাচ্ছি'; } };
const canBark  = { bark()  { return 'ঘেউ ঘেউ!'; } };

class Dog {
  constructor(name) { this.name = name; }
}

Object.assign(Dog.prototype, canEat, canSleep, canBark);

const dog = new Dog('রেক্স');
console.log(dog.bark()); // ঘেউ ঘেউ!
```

---

## 🔄 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | সম্পর্ক |
|----------|---------|
| Decorator | Decorator runtime-এ একটি অবজেক্টকে wrap করে; Mixin compile-time-এ prototype-এ যোগ করে |
| Strategy | Strategy behaviour swap করে; Mixin behaviour যোগ করে |
| Module | Mixin-গুলো Module হিসেবে এক্সপোর্ট করা হয় |
| Template Method | ফাংশনাল Mixin Template Method-এর মতো প্রসারিত করতে পারে |

---

## ✅ সুবিধা (Pros)

- **নমনীয়** — যেকোনো ক্লাসে মিক্সিন করা যায়
- **পুনঃব্যবহারযোগ্য** — একই মিক্সিন অনেক ক্লাসে
- **গভীর ইনহেরিটেন্স এড়ানো** — সমতল কোড স্ট্রাকচার

## ❌ অসুবিধা (Cons)

- **নামের সংঘর্ষ** — দুটো মিক্সিনে একই নাম হলে সমস্যা
- **উৎস অস্পষ্ট** — কোন মেথড কোন মিক্সিন থেকে এলো বোঝা কঠিন
- **টেস্টিং** — আলাদাভাবে টেস্ট করা কঠিন

---

## 📝 কখন ব্যবহার করবেন

- ✅ একাধিক ক্লাসে একই behaviour শেয়ার করতে হলে
- ✅ গভীর ইনহেরিটেন্স চেইন এড়াতে
- ✅ React-এ Custom Hooks হিসেবে
- ❌ যখন ইনহেরিটেন্স স্বাভাবিকভাবে ফিট হয়
- ❌ নামের সংঘর্ষের ঝুঁকি থাকলে সতর্ক থাকুন
