# I — Interface Segregation Principle (ISP)

> **"ক্লায়েন্টকে এমন ইন্টারফেসের উপর নির্ভর করতে বাধ্য করা উচিত নয় যা সে ব্যবহার করে না।"**
> — Robert C. Martin

---

## 📖 সংজ্ঞা

Interface Segregation Principle বলে যে, **একটি বড় ইন্টারফেসের বদলে একাধিক ছোট, নির্দিষ্ট ইন্টারফেস তৈরি করা উচিত।** প্রতিটি ক্লাস শুধুমাত্র তার প্রয়োজনীয় মেথডগুলো ইমপ্লিমেন্ট করবে।

সহজ ভাষায়: **ক্লাসকে এমন কোনো মেথড ইমপ্লিমেন্ট করতে বাধ্য করবেন না যা সে ব্যবহার করবে না।**

### "Fat Interface" সমস্যা

যখন একটি ইন্টারফেসে অনেক মেথড থাকে এবং কোনো ক্লাসের শুধু কয়েকটি দরকার, তখন:
- বাকি মেথডগুলো empty body বা `throw Exception` দিয়ে ইমপ্লিমেন্ট করতে হয়
- এটি LSP ও ভঙ্গ করে
- এটি কোডকে বিভ্রান্তিকর করে

---

## 🏠 বাস্তব জীবনের উদাহরণ

### রিমোট কন্ট্রোলের উদাহরণ

একটি সিম্পল রিমোট চিন্তা করুন:
- **টিভি রিমোট** — Power, Volume, Channel, Smart Menu দরকার
- **ফ্যানের রিমোট** — শুধু Power আর Speed দরকার

যদি একটি "সুপার রিমোট" বানান যেখানে সব বাটন আছে:
- ফ্যানের জন্য Channel বাটন অর্থহীন
- টিভির জন্য Speed বাটন অর্থহীন

**সমাধান:** আলাদা রিমোট — প্রতিটি শুধু প্রয়োজনীয় বাটন নিয়ে। এটাই ISP!

---

## ❌ খারাপ উদাহরণ — ISP ভঙ্গ

### PHP (❌ ভুল) — Fat Interface

```php
<?php

// ❌ "Fat Interface" — সব ধরনের Worker এর জন্য একটি ইন্টারফেস
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
    public function attendMeeting(): void;
    public function writeReport(): void;
    public function codeReview(): void;
}

// মানুষ কর্মী — সব মেথড অর্থবহ ✅
class HumanWorker implements Worker
{
    public function work(): void
    {
        echo "কাজ করছে\n";
    }

    public function eat(): void
    {
        echo "খাচ্ছে\n";
    }

    public function sleep(): void
    {
        echo "ঘুমাচ্ছে\n";
    }

    public function attendMeeting(): void
    {
        echo "মিটিংয়ে আছে\n";
    }

    public function writeReport(): void
    {
        echo "রিপোর্ট লিখছে\n";
    }

    public function codeReview(): void
    {
        echo "কোড রিভিউ করছে\n";
    }
}

// ❌ রোবট কর্মী — eat, sleep অর্থহীন!
class RobotWorker implements Worker
{
    public function work(): void
    {
        echo "রোবট কাজ করছে\n";
    }

    public function eat(): void
    {
        // ❌ রোবট খায় না! কিন্তু ইমপ্লিমেন্ট করতে বাধ্য
        throw new BadMethodCallException("রোবট খায় না!");
    }

    public function sleep(): void
    {
        // ❌ রোবট ঘুমায় না!
        throw new BadMethodCallException("রোবট ঘুমায় না!");
    }

    public function attendMeeting(): void
    {
        // ❌ রোবট মিটিংয়ে যায় না!
        throw new BadMethodCallException("রোবট মিটিংয়ে যায় না!");
    }

    public function writeReport(): void
    {
        echo "রোবট রিপোর্ট জেনারেট করছে\n";
    }

    public function codeReview(): void
    {
        // ❌ রোবট কোড রিভিউ করে না!
        throw new BadMethodCallException("রোবট কোড রিভিউ করে না!");
    }
}

// ❌ ইন্টার্ন — codeReview করে না কিন্তু ইমপ্লিমেন্ট করতে বাধ্য
class InternWorker implements Worker
{
    public function work(): void { echo "ইন্টার্ন কাজ করছে\n"; }
    public function eat(): void { echo "ইন্টার্ন খাচ্ছে\n"; }
    public function sleep(): void { echo "ইন্টার্ন ঘুমাচ্ছে\n"; }
    public function attendMeeting(): void { echo "ইন্টার্ন মিটিংয়ে আছে\n"; }
    public function writeReport(): void { echo "ইন্টার্ন রিপোর্ট লিখছে\n"; }

    public function codeReview(): void
    {
        // ❌ ইন্টার্ন কোড রিভিউ করে না!
        throw new BadMethodCallException("ইন্টার্নের কোড রিভিউ অনুমতি নেই!");
    }
}
```

### JavaScript (❌ ভুল)

```javascript
// ❌ Fat Interface — সব ফিচার একটি ক্লাসে
class MultiFunctionPrinter {
    print(document) { /* ... */ }
    scan(document) { /* ... */ }
    fax(document) { /* ... */ }
    staple(document) { /* ... */ }
    photocopy(document) { /* ... */ }
}

// ❌ সিম্পল প্রিন্টার — শুধু print দরকার, কিন্তু বাকি সব ইমপ্লিমেন্ট করতে হবে
class SimplePrinter extends MultiFunctionPrinter {
    print(document) {
        console.log(`প্রিন্ট করছে: ${document}`);
    }

    scan(document) {
        throw new Error("এই প্রিন্টারে স্ক্যানার নেই!"); // ❌
    }

    fax(document) {
        throw new Error("এই প্রিন্টারে ফ্যাক্স নেই!"); // ❌
    }

    staple(document) {
        throw new Error("এই প্রিন্টারে স্ট্যাপলার নেই!"); // ❌
    }

    photocopy(document) {
        throw new Error("এই প্রিন্টারে ফটোকপি নেই!"); // ❌
    }
}
```

---

## ✅ ভালো উদাহরণ — ISP মেনে

### PHP (✅ সঠিক) — ছোট, নির্দিষ্ট ইন্টারফেস

```php
<?php

// ✅ ছোট, নির্দিষ্ট ইন্টারফেস — প্রতিটি একটি capability প্রকাশ করে
interface Workable
{
    public function work(): void;
}

interface Eatable
{
    public function eat(): void;
}

interface Sleepable
{
    public function sleep(): void;
}

interface MeetingAttendable
{
    public function attendMeeting(): void;
}

interface ReportWritable
{
    public function writeReport(): void;
}

interface CodeReviewable
{
    public function codeReview(): void;
}

// ✅ মানুষ কর্মী — সব capability ইমপ্লিমেন্ট করে (কারণ তার সব দরকার)
class HumanDeveloper implements
    Workable,
    Eatable,
    Sleepable,
    MeetingAttendable,
    ReportWritable,
    CodeReviewable
{
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function work(): void
    {
        echo "{$this->name} কোড লিখছে\n";
    }

    public function eat(): void
    {
        echo "{$this->name} লাঞ্চ করছে\n";
    }

    public function sleep(): void
    {
        echo "{$this->name} বিশ্রাম নিচ্ছে\n";
    }

    public function attendMeeting(): void
    {
        echo "{$this->name} স্ট্যান্ডআপ মিটিংয়ে আছে\n";
    }

    public function writeReport(): void
    {
        echo "{$this->name} স্প্রিন্ট রিপোর্ট লিখছে\n";
    }

    public function codeReview(): void
    {
        echo "{$this->name} পুল রিকোয়েস্ট রিভিউ করছে\n";
    }
}

// ✅ রোবট — শুধু Workable আর ReportWritable ইমপ্লিমেন্ট করে
class RobotWorker implements Workable, ReportWritable
{
    public function work(): void
    {
        echo "রোবট অটোমেটেড টাস্ক চালাচ্ছে\n";
    }

    public function writeReport(): void
    {
        echo "রোবট অটো-রিপোর্ট জেনারেট করছে\n";
    }
    // ✅ eat(), sleep(), attendMeeting(), codeReview() নেই — দরকার নেই!
}

// ✅ ইন্টার্ন — কোড রিভিউ ছাড়া বাকি সব পারে
class InternDeveloper implements
    Workable,
    Eatable,
    Sleepable,
    MeetingAttendable,
    ReportWritable
{
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function work(): void { echo "{$this->name} শিখছে ও কোড করছে\n"; }
    public function eat(): void { echo "{$this->name} খাচ্ছে\n"; }
    public function sleep(): void { echo "{$this->name} ঘুমাচ্ছে\n"; }
    public function attendMeeting(): void { echo "{$this->name} মিটিংয়ে শুনছে\n"; }
    public function writeReport(): void { echo "{$this->name} ডেইলি রিপোর্ট লিখছে\n"; }
    // ✅ codeReview() নেই — ইন্টার্নের এই অনুমতি নেই
}

// ✅ ফাংশনগুলো শুধু প্রয়োজনীয় ইন্টারফেস ব্যবহার করে
function assignTask(Workable $worker): void
{
    $worker->work();
}

function lunchBreak(Eatable $worker): void
{
    $worker->eat();
}

function dailyStandup(MeetingAttendable $attendee): void
{
    $attendee->attendMeeting();
}

// ✅ সব টাইপে কাজ করে — কোনো exception নেই
$dev = new HumanDeveloper("রহিম");
$robot = new RobotWorker();
$intern = new InternDeveloper("করিম");

assignTask($dev);    // ✅
assignTask($robot);  // ✅
assignTask($intern); // ✅

lunchBreak($dev);    // ✅
lunchBreak($intern); // ✅
// lunchBreak($robot); — কম্পাইল এরর! RobotWorker Eatable না ✅
```

### JavaScript (✅ সঠিক) — Mixin/Composition দিয়ে

```javascript
// ✅ JavaScript এ ইন্টারফেস নেই — তাই Mixin/Composition ব্যবহার করি

// ছোট ছোট capability mixin
const Printable = {
    print(document) {
        console.log(`প্রিন্ট করছে: ${document}`);
    }
};

const Scannable = {
    scan(document) {
        console.log(`স্ক্যান করছে: ${document}`);
        return `scanned_${document}`;
    }
};

const Faxable = {
    fax(document, number) {
        console.log(`ফ্যাক্স পাঠাচ্ছে: ${document} → ${number}`);
    }
};

const Photocopiable = {
    photocopy(document, copies = 1) {
        console.log(`ফটোকপি করছে: ${document} x${copies}`);
    }
};

// ✅ হেল্পার ফাংশন — mixin যোগ করার জন্য
function createDevice(name, ...mixins) {
    const device = { name };
    for (const mixin of mixins) {
        Object.assign(device, mixin);
    }
    return device;
}

// ✅ প্রতিটি ডিভাইস শুধু তার capability পায়
const simplePrinter = createDevice('সিম্পল প্রিন্টার', Printable);
const officePrinter = createDevice('অফিস প্রিন্টার', Printable, Scannable, Photocopiable);
const fullPrinter = createDevice('ফুল প্রিন্টার', Printable, Scannable, Faxable, Photocopiable);

simplePrinter.print("রিপোর্ট.pdf");       // ✅
officePrinter.print("রিপোর্ট.pdf");       // ✅
officePrinter.scan("ডকুমেন্ট.pdf");       // ✅
fullPrinter.fax("চুক্তি.pdf", "01711.."); // ✅

// simplePrinter.scan() → undefined — কিন্তু Error throw করে না

// ✅ Class-based approach with composition
class SimplePrinter {
    print(document) {
        console.log(`প্রিন্ট করছে: ${document}`);
    }
}

class AllInOnePrinter {
    #printer;
    #scanner;

    constructor() {
        this.#printer = new PrinterModule();
        this.#scanner = new ScannerModule();
    }

    print(document) { this.#printer.print(document); }
    scan(document) { return this.#scanner.scan(document); }
}

class PrinterModule {
    print(document) {
        console.log(`প্রিন্ট করছে: ${document}`);
    }
}

class ScannerModule {
    scan(document) {
        console.log(`স্ক্যান করছে: ${document}`);
        return `scanned_${document}`;
    }
}

// ✅ ফাংশন শুধু প্রয়োজনীয় capability চেক করে
function printDocument(printer, document) {
    if (typeof printer.print !== 'function') {
        throw new Error("এই ডিভাইসে প্রিন্ট capability নেই");
    }
    printer.print(document);
}

function scanDocument(scanner, document) {
    if (typeof scanner.scan !== 'function') {
        throw new Error("এই ডিভাইসে স্ক্যান capability নেই");
    }
    return scanner.scan(document);
}
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### বাস্তব প্রজেক্টে ISP — Laravel উদাহরণ

```php
<?php

// ❌ Fat Repository Interface
interface UserRepository
{
    public function find(int $id): ?User;
    public function findAll(): array;
    public function findByEmail(string $email): ?User;
    public function create(array $data): User;
    public function update(int $id, array $data): User;
    public function delete(int $id): void;
    public function softDelete(int $id): void;
    public function restore(int $id): void;
    public function export(): string;
    public function import(string $csv): void;
    public function generateReport(): string;
}

// ✅ আলাদা ইন্টারফেস — প্রতিটি একটি concern
interface ReadableRepository
{
    public function find(int $id): ?User;
    public function findAll(): array;
    public function findByEmail(string $email): ?User;
}

interface WritableRepository
{
    public function create(array $data): User;
    public function update(int $id, array $data): User;
    public function delete(int $id): void;
}

interface SoftDeletable
{
    public function softDelete(int $id): void;
    public function restore(int $id): void;
}

interface Exportable
{
    public function export(): string;
}

interface Importable
{
    public function import(string $csv): void;
}

interface Reportable
{
    public function generateReport(): string;
}

// ✅ Eloquent Repository — সব পারে
class EloquentUserRepository implements
    ReadableRepository,
    WritableRepository,
    SoftDeletable,
    Exportable,
    Importable,
    Reportable
{
    // সব মেথড ইমপ্লিমেন্ট...
}

// ✅ API Controller — শুধু Read + Write দরকার
class UserApiController
{
    public function __construct(
        private readonly ReadableRepository $reader,
        private readonly WritableRepository $writer
    ) {}

    public function index(): array
    {
        return $this->reader->findAll();
    }

    public function store(array $data): User
    {
        return $this->writer->create($data);
    }
}

// ✅ Report Controller — শুধু Reportable দরকার
class ReportController
{
    public function __construct(
        private readonly Reportable $reportable
    ) {}

    public function generate(): string
    {
        return $this->reportable->generateReport();
    }
}

// ✅ Read-only API — শুধু ReadableRepository দরকার
class PublicApiController
{
    public function __construct(
        private readonly ReadableRepository $reader
    ) {}

    public function show(int $id): ?User
    {
        return $this->reader->find($id);
    }
}
```

### Role Interface vs Header Interface

```
❌ Header Interface (ISP ভঙ্গ):
┌─────────────────────────────┐
│  UserRepository              │
│  ─────────────────────────── │
│  find()                      │
│  findAll()                   │
│  create()                    │
│  update()                    │
│  delete()                    │
│  export()                    │
│  import()                    │
│  generateReport()            │
│  softDelete()                │
│  restore()                   │
└─────────────────────────────┘
     ↑ সবাই এটাই ইমপ্লিমেন্ট করে

✅ Role Interface (ISP মেনে):
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Readable │ │ Writable │ │Exportable│
│──────────│ │──────────│ │──────────│
│ find()   │ │ create() │ │ export() │
│ findAll()│ │ update() │ └──────────┘
└──────────┘ │ delete() │
             └──────────┘
     ↑ প্রতিটি ক্লাস শুধু প্রয়োজনীয় role ইমপ্লিমেন্ট করে
```

### TypeScript/JavaScript এ ISP

```javascript
// TypeScript-style ISP (জেনেরিক ইন্টারফেস চিন্তা)

// ✅ ছোট ছোট "ক্ষমতা" ক্লাস
class EventEmitter {
    #listeners = new Map();

    on(event, callback) {
        if (!this.#listeners.has(event)) {
            this.#listeners.set(event, []);
        }
        this.#listeners.get(event).push(callback);
    }

    emit(event, data) {
        const callbacks = this.#listeners.get(event) || [];
        callbacks.forEach(cb => cb(data));
    }
}

class Serializable {
    toJSON() {
        throw new Error('toJSON() ইমপ্লিমেন্ট করুন');
    }

    static fromJSON(json) {
        throw new Error('fromJSON() ইমপ্লিমেন্ট করুন');
    }
}

class Validatable {
    validate() {
        throw new Error('validate() ইমপ্লিমেন্ট করুন');
    }

    get isValid() {
        try {
            this.validate();
            return true;
        } catch {
            return false;
        }
    }
}

// ✅ প্রতিটি মডেল শুধু প্রয়োজনীয় capability গ্রহণ করে
class User {
    #name;
    #email;
    #events = new EventEmitter();

    constructor(name, email) {
        this.#name = name;
        this.#email = email;
    }

    get name() { return this.#name; }
    get email() { return this.#email; }

    // Serializable behavior
    toJSON() {
        return { name: this.#name, email: this.#email };
    }

    // Validatable behavior
    validate() {
        if (!this.#name) throw new Error("নাম আবশ্যক");
        if (!this.#email?.includes('@')) throw new Error("ইমেইল অবৈধ");
    }

    // EventEmitter behavior
    on(event, callback) { this.#events.on(event, callback); }
    emit(event, data) { this.#events.emit(event, data); }

    save() {
        this.validate();
        this.emit('saving', this);
        // সেভ লজিক...
        this.emit('saved', this);
    }
}

// ✅ Log Entry — শুধু Serializable, Validatable দরকার নেই
class LogEntry {
    #message;
    #timestamp;

    constructor(message) {
        this.#message = message;
        this.#timestamp = new Date();
    }

    toJSON() {
        return { message: this.#message, timestamp: this.#timestamp.toISOString() };
    }
    // ✅ validate(), EventEmitter কিছুই নেই — দরকার নেই!
}
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **কম অপ্রয়োজনীয় dependency** | ক্লাস শুধু যা দরকার তা জানে |
| ২ | **সহজ ইমপ্লিমেন্টেশন** | ছোট ইন্টারফেস ইমপ্লিমেন্ট করা সহজ |
| ৩ | **LSP সমর্থন** | empty method বা exception throw হবে না |
| ৪ | **সহজ মকিং** | টেস্টে ছোট ইন্টারফেস মক করা সহজ |
| ৫ | **ফ্লেক্সিবল কম্পোজিশন** | ক্লাস প্রয়োজনমতো capability মিক্স করতে পারে |
| ৬ | **কম পরিবর্তন প্রভাব** | একটি ইন্টারফেস বদলালে শুধু সংশ্লিষ্ট ক্লাস প্রভাবিত |
| ৭ | **স্পষ্ট কন্ট্র্যাক্ট** | ইন্টারফেসের নাম দেখেই বোঝা যায় কী capability |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **ইন্টারফেস সংখ্যা বাড়ে** | অনেক ছোট ইন্টারফেস ম্যানেজ করতে হয় |
| ২ | **ইন্টারফেস বিভ্রান্তি** | কোন ক্লাস কোন ইন্টারফেস ইমপ্লিমেন্ট করে তা ট্র্যাক করা কঠিন |
| ৩ | **অতি-বিভাজন** | খুব বেশি ভাঙলে ইন্টারফেস অর্থহীন হয়ে যেতে পারে |
| ৪ | **JavaScript এ প্রয়োগ কঠিন** | JS তে formal interface নেই, convention follow করতে হয় |
| ৫ | **বয়লারপ্লেট** | implements লিস্ট লম্বা হতে পারে |

---

## 🚫 সাধারণ ভুল

### ভুল ১: প্রতিটি মেথডের জন্য আলাদা ইন্টারফেস

```php
<?php

// ❌ অতি-বিভাজন — একটি মেথড, একটি ইন্টারফেস
interface HasName { public function getName(): string; }
interface HasEmail { public function getEmail(): string; }
interface HasPhone { public function getPhone(): string; }
interface HasAddress { public function getAddress(): string; }
// এভাবে ১০০টা ইন্টারফেস!

// ✅ সঠিক — সম্পর্কিত মেথড একসাথে
interface Contactable
{
    public function getEmail(): string;
    public function getPhone(): string;
}

interface Addressable
{
    public function getAddress(): string;
    public function getCity(): string;
    public function getCountry(): string;
}

interface Identifiable
{
    public function getId(): int;
    public function getName(): string;
}
```

### ভুল ২: ক্লায়েন্টের কথা না ভেবে ইন্টারফেস ডিজাইন

```php
<?php

// ❌ ইন্টারফেস ডিজাইন করেছি ক্লাসের দিক থেকে চিন্তা করে
// কিন্তু ক্লায়েন্টের কী দরকার সেটা ভাবিনি

// ✅ ক্লায়েন্টের দৃষ্টিকোণ থেকে ভাবুন:
// - কোন কোড এই ইন্টারফেস ব্যবহার করবে?
// - তার কী কী মেথড দরকার?
// - সেই মেথডগুলোই একটি ইন্টারফেসে রাখুন
```

---

## 📏 ISP ভঙ্গের লক্ষণ ও কখন প্রয়োগ করবেন

### ⚠️ ISP ভঙ্গের লক্ষণ:

- ইমপ্লিমেন্টেশনে **empty method body** আছে
- ইমপ্লিমেন্টেশনে **NotImplementedException** throw হচ্ছে
- ক্লাস ইন্টারফেসের **অর্ধেক মেথড** ব্যবহার করে না
- ইন্টারফেসে **১০+ মেথড** আছে
- বিভিন্ন ক্লায়েন্ট ইন্টারফেসের **ভিন্ন ভিন্ন অংশ** ব্যবহার করে

### ✅ করবেন যখন:

- ইন্টারফেস **অনেক বড়** হয়ে যাচ্ছে
- বিভিন্ন ইমপ্লিমেন্টেশনে **empty/throw মেথড** দেখা যাচ্ছে
- বিভিন্ন ক্লায়েন্টের **ভিন্ন প্রয়োজন** — কেউ read চায়, কেউ write

### ❌ করবেন না যখন:

- ইন্টারফেসের **সব মেথড সব ইমপ্লিমেন্টেশনে** অর্থবহ
- ইন্টারফেসে **৩-৫টি** সম্পর্কিত মেথড
- শুধু **একটি ইমপ্লিমেন্টেশন** আছে

---

## 🔗 অন্যান্য SOLID প্রিন্সিপলের সাথে সম্পর্ক

```
ISP ──► SRP: ছোট ইন্টারফেস = একটি দায়িত্ব/role
ISP ──► LSP: ছোট ইন্টারফেস মানে empty method দরকার নেই = LSP ভঙ্গ হয় না
ISP ──► OCP: ছোট ইন্টারফেস সহজে এক্সটেন্ড করা যায়
ISP ──► DIP: ছোট, নির্দিষ্ট abstraction এর উপর নির্ভর করা সহজ
```

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **মূল ধারণা** | বড় ইন্টারফেস ভেঙে ছোট, নির্দিষ্ট ইন্টারফেস তৈরি করুন |
| **লক্ষ্য** | ক্লাস শুধু প্রয়োজনীয় মেথড ইমপ্লিমেন্ট করবে |
| **চেক করুন** | "কোনো ইমপ্লিমেন্টেশনে empty/throw মেথড আছে কি?" |
| **সাবধান** | অতি-বিভাজন এড়িয়ে চলুন — সম্পর্কিত মেথড একসাথে রাখুন |
| **মনে রাখুন** | ক্লায়েন্টের দৃষ্টিকোণ থেকে ইন্টারফেস ডিজাইন করুন |
