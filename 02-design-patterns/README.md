# 📘 ডিজাইন প্যাটার্ন (GoF + JavaScript + React)

> **GoF ২৩টি** প্যাটার্ন + **JavaScript-specific** প্যাটার্ন + **React** প্যাটার্ন
> প্রতিটি প্যাটার্নে PHP 8.3 ও JavaScript ES2022+ উদাহরণ, real-world applicable areas, 
> deep dive analysis, অন্যান্য প্যাটার্নের সাথে সম্পর্ক, এবং বাংলাদেশ context অন্তর্ভুক্ত।

---

## 🎯 দ্রুত নেভিগেশন

| ক্যাটাগরি | লিংক | বর্ণনা |
|----------|------|--------|
| **জাভাস্ক্রিপ্ট** | [`javascript/`](./javascript/) | Module Pattern, Mixin Pattern |
| **রিঅ্যাক্ট** | [`react/`](./react/) | Provider Pattern + Component Design Patterns |
| **ক্রিয়েশনাল** | [`creational/`](./creational/) | ৫টি প্যাটার্ন — Singleton, Factory, Builder, Prototype, Abstract Factory |
| **স্ট্রাকচারাল** | [`structural/`](./structural/) | ৭টি প্যাটার্ন — Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| **বিহেভিয়রাল** | [`behavioral/`](./behavioral/) | ১১টি প্যাটার্ন — Observer, Strategy, State, Command, Iterator, Mediator, Template Method, Chain of Responsibility, Visitor, Interpreter, Memento |

---

## 📂 বিস্তারিত ক্যাটাগরি সূচি

### 🏗️ ক্রিয়েশনাল প্যাটার্ন (Creational) — [`creational/`](./creational/)

> অবজেক্ট তৈরির কৌশল সম্পর্কিত প্যাটার্ন।

| # | প্যাটার্ন | ফাইল | মূল বিষয়বস্তু |
|---|----------|------|--------------|
| ১ | Singleton | [singleton.md](./creational/singleton.md) | Thread-safe, Enum, Multiton, DI Container, Anti-pattern বিশ্লেষণ |
| ২ | Factory Method | [factory-method.md](./creational/factory-method.md) | Payment Gateway (bKash/Nagad), Laravel Manager, Parameterized/Registration Factory |
| ৩ | Abstract Factory | [abstract-factory.md](./creational/abstract-factory.md) | UI Theme, Multi-DB, Payment Ecosystem families, Laravel DatabaseManager |
| ৪ | Builder | [builder.md](./creational/builder.md) | Fluent/Step Builder, Query Builder, Email Builder, Immutable Builder |
| ৫ | Prototype | [prototype.md](./creational/prototype.md) | Shallow vs Deep Copy, Registry, Clone, structuredClone, Copy-on-Write |

---

### 🧱 স্ট্রাকচারাল প্যাটার্ন (Structural) — [`structural/`](./structural/)

> ক্লাস ও অবজেক্টের কম্পোজিশন সম্পর্কিত প্যাটার্ন।

| # | প্যাটার্ন | ফাইল | মূল বিষয়বস্তু |
|---|----------|------|--------------|
| ১ | Adapter | [adapter.md](./structural/adapter.md) | Object/Class Adapter, Payment/SMS Gateway, Laravel Filesystem, Legacy Migration |
| ২ | Bridge | [bridge.md](./structural/bridge.md) | Notification×Channel, Payment×Gateway, Multi-dimensional, Runtime Switching |
| ৩ | Composite | [composite.md](./structural/composite.md) | FileSystem, Menu, Organization, DFS/BFS Iterator, React Component Tree |
| ৪ | Decorator | [decorator.md](./structural/decorator.md) | Laravel Middleware, Caching/Logging Decorator, PHP Attributes, TC39 Decorators |
| ৫ | Facade | [facade.md](./structural/facade.md) | Laravel Facades internals, Order Processing, Anti-Corruption Layer, Nested Facade |
| ৬ | Flyweight | [flyweight.md](./structural/flyweight.md) | Intrinsic/Extrinsic, Particle System, Map Tiles, WeakRef/WeakMap, Memory Profiling |
| ৭ | Proxy | [proxy.md](./structural/proxy.md) | ৬ Proxy Types, JS Proxy Traps, Rate Limiting, Proxy Chain, Laravel Lazy Loading |

---

### 🎭 বিহেভিয়রাল প্যাটার্ন (Behavioral) — [`behavioral/`](./behavioral/)

> অবজেক্টের মধ্যে যোগাযোগ ও দায়িত্ব বণ্টন সম্পর্কিত প্যাটার্ন।

| # | প্যাটার্ন | ফাইল | মূল বিষয়বস্তু |
|---|----------|------|--------------|
| ১ | Chain of Responsibility | [chain-of-responsibility.md](./behavioral/chain-of-responsibility.md) | HTTP Middleware Pipeline, Laravel/Express, Bidirectional Chain, Priority-based |
| ২ | Command | [command.md](./behavioral/command.md) | Undo/Redo, Macro, Queue, Laravel Jobs/Artisan, Command Bus, Saga |
| ৩ | Iterator | [iterator.md](./behavioral/iterator.md) | PHP Generator/SPL, JS Symbol.iterator, Async Iterator, Lazy Evaluation, Pipeline |
| ৪ | Mediator | [mediator.md](./behavioral/mediator.md) | ChatRoom, UI Dialog, Event Dispatcher, Laravel Events, API Gateway as Mediator |
| ৫ | Memento | [memento.md](./behavioral/memento.md) | Undo Stack, Form Draft, Game Checkpoint, Delta Memento, Event Sourcing |
| ৬ | Observer | [observer.md](./behavioral/observer.md) | Laravel Observers, Node EventEmitter, Reactive Observable, Push vs Pull, WebSocket |
| ৭ | State | [state.md](./behavioral/state.md) | Order/Payment/Document FSM, bKash Transaction States, HSM, Transition Table |
| ৮ | Strategy | [strategy.md](./behavioral/strategy.md) | Payment/Auth/Caching Strategy, Lambda/Closure, Functional, Laravel Guards |
| ৯ | Template Method | [template-method.md](./behavioral/template-method.md) | Data Import Pipeline, Report Generator, Hollywood Principle, Hook Methods |
| ১০ | Visitor | [visitor.md](./behavioral/visitor.md) | Double Dispatch, AST Visitor, Tax Calculator, Document Exporter, Acyclic Visitor |
| ১১ | Interpreter | [interpreter.md](./behavioral/interpreter.md) | Math/Boolean Evaluator, Rule Engine, DSL Builder, Laravel Validation Internals |

---

## 🗺️ প্যাটার্ন সম্পর্ক ডায়াগ্রাম

```
            ┌─────────────────── Creational ───────────────────┐
            │                                                   │
    Singleton ◄──── Abstract Factory ────► Factory Method       │
        │                  │                    │               │
        │                  ▼                    ▼               │
        │              Builder ◄──────── Prototype              │
        │                                                       │
        ├─────────────────── Structural ───────────────────────┤
        │                                                       │
    Facade ────► Adapter ◄──► Bridge                            │
        │           │            │                              │
        │           ▼            ▼                              │
        │       Decorator ◄──► Composite ──► Flyweight          │
        │           │                                           │
        │           ▼                                           │
        │        Proxy                                          │
        │                                                       │
        ├─────────────────── Behavioral ───────────────────────┤
        │                                                       │
    Chain of Resp. ◄──► Command ◄──► Memento                   │
        │                  │                                    │
        ▼                  ▼                                    │
    Mediator ◄────► Observer                                    │
        │                  │                                    │
        ▼                  ▼                                    │
    Strategy ◄────► State                                       │
        │                                                       │
        ▼                                                       │
    Template Method ──► Iterator ◄──► Visitor ──► Interpreter  │
        │                                                       │
        └───────────────────────────────────────────────────────┘
```

---

## 📖 প্রতিটি প্যাটার্নে যা পাবেন

- ✅ বাংলায় GoF সংজ্ঞা ও বিস্তারিত ব্যাখ্যা
- ✅ বাস্তব জীবনের উদাহরণ (Analogy)
- ✅ ASCII UML ও Sequence ডায়াগ্রাম
- ✅ PHP 8.3 (readonly, enums, match, constructor promotion) কোড
- ✅ JavaScript ES2022+ (private #fields, static, optional chaining) কোড
- ✅ Real-World Applicable Areas (Laravel, Express, React framework উদাহরণ)
- ✅ Advanced Deep Dive (প্যাটার্ন তুলনা, ফ্রেমওয়ার্ক internals)
- ✅ সুবিধা (Pros) ও অসুবিধা (Cons) টেবিল
- ✅ সাধারণ ভুল ও Anti-Patterns (❌ Bad → ✅ Good কোড)
- ✅ টেস্টিং (PHPUnit + Jest)
- ✅ সম্পর্কিত প্যাটার্নের বিশ্লেষণ
- ✅ কখন ব্যবহার করবেন / করবেন না
- ✅ বাংলাদেশ context (bKash, Nagad, Pathao, Daraz উদাহরণ)

---

> **মোট:** ৫ ক্রিয়েশনাল + ৭ স্ট্রাকচারাল + ১১ বিহেভিয়রাল = **২৩টি GoF প্যাটার্ন** ✅ সম্পূর্ণ

---

## 🟨 জাভাস্ক্রিপ্ট প্যাটার্ন — [`javascript/`](./javascript/)

> GoF-এর বাইরে JavaScript-specific ডিজাইন প্যাটার্ন (patterns.dev)

| # | প্যাটার্ন | ফাইল | মূল বিষয়বস্তু |
|---|----------|------|--------------|
| ১ | Module Pattern | [module-pattern.md](./javascript/module-pattern.md) | ES Modules, এনক্যাপসুলেশন, Named/Default Export, IIFE, Dynamic Import |
| ২ | Mixin Pattern | [mixin-pattern.md](./javascript/mixin-pattern.md) | Composition over Inheritance, Object.assign, Functional Mixin, Custom Hooks |

---

## ⚛️ রিঅ্যাক্ট প্যাটার্ন — [`react/`](./react/)

> React-specific কম্পোনেন্ট ডিজাইন প্যাটার্ন (patterns.dev)
>
> **দ্রষ্টব্য:** Compound Component, Render Props, HOC, Hooks → [`19-frontend-engineering/component-design.md`](../../19-frontend-engineering/component-design.md)

| # | প্যাটার্ন | ফাইল | মূল বিষয়বস্তু |
|---|----------|------|--------------|
| ১ | Provider Pattern | [provider-pattern.md](./react/provider-pattern.md) | React Context API, Prop Drilling সমাধান, Auth/Theme/Cart Provider, useMemo অপ্টিমাইজেশন |
