# 👁️ Observer প্যাটার্ন (Observer Pattern)

## 📌 সংজ্ঞা (Definition)

**Gang of Four (GoF) সংজ্ঞা:**

> *"Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically."*

Observer প্যাটার্ন হলো একটি **Behavioral Design Pattern** যা অবজেক্টদের মধ্যে **one-to-many** সম্পর্ক স্থাপন করে। যখন একটি অবজেক্ট (Subject/Publisher) তার state পরিবর্তন করে, তখন তার উপর নির্ভরশীল সকল অবজেক্ট (Observers/Subscribers) **স্বয়ংক্রিয়ভাবে** জানানো হয় এবং আপডেট হয়।

### মূল ধারণা:

এই প্যাটার্নের দুটি প্রধান ভূমিকা আছে:

- **Subject (Publisher/Observable):** যে অবজেক্ট state ধারণ করে এবং পরিবর্তন হলে সবাইকে জানায়
- **Observer (Subscriber/Listener):** যে অবজেক্টগুলো Subject-এর পরিবর্তন শুনতে চায় এবং সেই অনুযায়ী react করে

এটি **loose coupling** নিশ্চিত করে — Subject জানে না Observer কী করবে, শুধু জানায়। Observer জানে না আর কে কে শুনছে। প্রত্যেকে স্বাধীনভাবে কাজ করে।

### বাংলাদেশ কনটেক্সটে উদাহরণ:

ধরুন আপনি **bKash** দিয়ে একটি ট্রানজ্যাকশন করলেন। তখন কী হয়?

1. আপনার ফোনে **SMS notification** যায়
2. bKash অ্যাপে **push notification** আসে
3. ব্যাংক অ্যাকাউন্টে **balance update** হয়
4. **Transaction log** তৈরি হয়
5. প্রয়োজনে **fraud detection system** alert পায়

এখানে bKash-এর ট্রানজ্যাকশন সিস্টেম হলো **Subject**, আর SMS, push notification, bank, log, fraud detection — এগুলো সবই **Observer**।

---

## 🏠 বাস্তব উদাহরণ (Real-World Analogy)

### 📰 সংবাদপত্র সাবস্ক্রিপশন (Newspaper Subscription)

এটি Observer প্যাটার্নের সবচেয়ে ক্লাসিক উদাহরণ:

```
📰 প্রথম আলো (Publisher/Subject)
    │
    ├──► 👤 গ্রাহক ১ (Observer) — সকালে পেপার পায়
    ├──► 👤 গ্রাহক ২ (Observer) — সকালে পেপার পায়
    ├──► 👤 গ্রাহক ৩ (Observer) — সকালে পেপার পায়
    └──► 👤 গ্রাহক ৪ (Observer) — সকালে পেপার পায়

কীভাবে কাজ করে:
✅ নতুন গ্রাহক → subscribe() → তালিকায় যুক্ত হয়
❌ গ্রাহক বাদ → unsubscribe() → তালিকা থেকে সরানো হয়
📢 নতুন সংবাদ → notify() → সব গ্রাহককে জানানো হয়
```

### 🏍️ Pathao রাইড স্ট্যাটাস (Bangladesh Context)

```
🏍️ Pathao Ride (Subject)
    │
    │ State Change: "Rider Assigned" → "On the Way" → "Arrived" → "Trip Started" → "Completed"
    │
    ├──► 📱 Rider App (Observer) — UI আপডেট, নেভিগেশন শুরু
    ├──► 📱 Passenger App (Observer) — ম্যাপে ট্র্যাকিং, ETA দেখায়
    ├──► 💰 Billing System (Observer) — ভাড়া ক্যালকুলেশন শুরু/শেষ
    ├──► 📊 Analytics (Observer) — রাইড ডেটা সংগ্রহ
    └──► 🔔 Notification Service (Observer) — SMS/Push পাঠায়
```

---

## 📊 UML ডায়াগ্রাম (ASCII)

### ক্লাসিক Observer Pattern Structure:

```
┌─────────────────────────────────┐
│        <<interface>>            │
│          Subject                │
├─────────────────────────────────┤
│ + attach(observer: Observer)    │
│ + detach(observer: Observer)    │
│ + notify(): void                │
└──────────────┬──────────────────┘
               │ implements
               ▼
┌─────────────────────────────────┐         ┌─────────────────────────────┐
│      ConcreteSubject            │         │      <<interface>>          │
├─────────────────────────────────┤         │        Observer             │
│ - state: mixed                  │         ├─────────────────────────────┤
│ - observers: Observer[]         │◆───────►│ + update(subject): void     │
├─────────────────────────────────┤   *     └──────────────┬──────────────┘
│ + getState(): mixed             │                        │ implements
│ + setState(state): void         │                        ▼
│ + attach(observer): void        │         ┌─────────────────────────────┐
│ + detach(observer): void        │         │    ConcreteObserver         │
│ + notify(): void                │         ├─────────────────────────────┤
└─────────────────────────────────┘         │ - observerState: mixed      │
                                            ├─────────────────────────────┤
                                            │ + update(subject): void     │
                                            └─────────────────────────────┘
```

### ইভেন্ট ফ্লো ডায়াগ্রাম:

```
  Subject              Observer A          Observer B          Observer C
    │                     │                    │                    │
    │  setState("new")    │                    │                    │
    ├────────┐            │                    │                    │
    │        │ notify()   │                    │                    │
    │◄───────┘            │                    │                    │
    │                     │                    │                    │
    │   update(this)      │                    │                    │
    ├────────────────────►│                    │                    │
    │                     │                    │                    │
    │   update(this)      │                    │                    │
    ├─────────────────────┼───────────────────►│                    │
    │                     │                    │                    │
    │   update(this)      │                    │                    │
    ├─────────────────────┼────────────────────┼───────────────────►│
    │                     │                    │                    │
```

---

## 💻 ইমপ্লিমেন্টেশন (Implementation)

### 1️⃣ Basic Observer — PHP (SplObserver/SplSubject)

PHP-তে বিল্ট-ইন `SplObserver` এবং `SplSubject` ইন্টারফেস আছে। PHP 8.3-এ এগুলো ব্যবহার করে ক্লিন ইমপ্লিমেন্টেশন করা যায়:

```php
<?php

declare(strict_types=1);

// PHP 8.3 — SplSubject/SplObserver ব্যবহার করে Observer Pattern

/**
 * বিকাশ ট্রানজ্যাকশন সিস্টেম — Subject হিসেবে কাজ করবে
 */
class BkashTransaction implements \SplSubject
{
    private \SplObjectStorage $observers;
    private string $status = 'initiated';

    public function __construct(
        private readonly string $transactionId,
        private readonly float $amount,
        private readonly string $senderPhone,
        private readonly string $receiverPhone,
    ) {
        $this->observers = new \SplObjectStorage();
    }

    public function attach(\SplObserver $observer): void
    {
        $this->observers->attach($observer);
    }

    public function detach(\SplObserver $observer): void
    {
        $this->observers->detach($observer);
    }

    public function notify(): void
    {
        foreach ($this->observers as $observer) {
            $observer->update($this);
        }
    }

    public function processTransaction(): void
    {
        $this->status = 'processing';
        $this->notify();

        // ট্রানজ্যাকশন লজিক...
        $this->status = 'completed';
        $this->notify();
    }

    // Readonly accessors
    public function getStatus(): string { return $this->status; }
    public function getTransactionId(): string { return $this->transactionId; }
    public function getAmount(): float { return $this->amount; }
    public function getSenderPhone(): string { return $this->senderPhone; }
    public function getReceiverPhone(): string { return $this->receiverPhone; }
}

/**
 * SMS নোটিফিকেশন Observer
 */
class SmsNotificationObserver implements \SplObserver
{
    public function update(\SplSubject $subject): void
    {
        if (!$subject instanceof BkashTransaction) return;

        match ($subject->getStatus()) {
            'processing' => printf(
                "[SMS → %s] আপনার bKash ট্রানজ্যাকশন %s প্রক্রিয়াধীন...\n",
                $subject->getSenderPhone(),
                $subject->getTransactionId()
            ),
            'completed' => printf(
                "[SMS → %s] ৳%.2f সফলভাবে %s-এ পাঠানো হয়েছে। TrxID: %s\n",
                $subject->getSenderPhone(),
                $subject->getAmount(),
                $subject->getReceiverPhone(),
                $subject->getTransactionId()
            ),
            default => null,
        };
    }
}

/**
 * Fraud Detection Observer
 */
class FraudDetectionObserver implements \SplObserver
{
    private const float SUSPICIOUS_AMOUNT = 50_000.00;

    public function update(\SplSubject $subject): void
    {
        if (!$subject instanceof BkashTransaction) return;
        if ($subject->getStatus() !== 'processing') return;

        if ($subject->getAmount() > self::SUSPICIOUS_AMOUNT) {
            printf(
                "🚨 [FRAUD ALERT] সন্দেহজনক ট্রানজ্যাকশন: ৳%.2f — TrxID: %s\n",
                $subject->getAmount(),
                $subject->getTransactionId()
            );
        }
    }
}

/**
 * Transaction Log Observer
 */
class TransactionLogObserver implements \SplObserver
{
    private array $logs = [];

    public function update(\SplSubject $subject): void
    {
        if (!$subject instanceof BkashTransaction) return;

        $this->logs[] = [
            'trxId'     => $subject->getTransactionId(),
            'status'    => $subject->getStatus(),
            'amount'    => $subject->getAmount(),
            'timestamp' => (new \DateTimeImmutable())->format('Y-m-d H:i:s'),
        ];

        printf("[LOG] TrxID: %s — Status: %s\n", $subject->getTransactionId(), $subject->getStatus());
    }

    public function getLogs(): array { return $this->logs; }
}

// === ব্যবহার ===
$transaction = new BkashTransaction(
    transactionId: 'TRX' . random_int(100000, 999999),
    amount: 75_000.00,
    senderPhone: '01711XXXXXX',
    receiverPhone: '01812XXXXXX',
);

$transaction->attach(new SmsNotificationObserver());
$transaction->attach(new FraudDetectionObserver());
$transaction->attach($logger = new TransactionLogObserver());

$transaction->processTransaction();

// Output:
// [LOG] TrxID: TRX847293 — Status: processing
// 🚨 [FRAUD ALERT] সন্দেহজনক ট্রানজ্যাকশন: ৳75000.00 — TrxID: TRX847293
// [SMS → 01711XXXXXX] আপনার bKash ট্রানজ্যাকশন TRX847293 প্রক্রিয়াধীন...
// [LOG] TrxID: TRX847293 — Status: completed
// [SMS → 01711XXXXXX] ৳75000.00 সফলভাবে 01812XXXXXX-এ পাঠানো হয়েছে।
```

### 2️⃣ Basic Observer — JavaScript ES2022+

```javascript
// JavaScript ES2022+ — Observer Pattern (Custom Implementation)

/**
 * Subject Base Class — যেকোনো Observable এটি extend করবে
 */
class Subject {
    #observers = new Map(); // event → Set<callback>

    subscribe(event, observer) {
        if (!this.#observers.has(event)) {
            this.#observers.set(event, new Set());
        }
        this.#observers.get(event).add(observer);

        // Unsubscribe ফাংশন রিটার্ন — মেমরি লিক প্রতিরোধ
        return () => this.#observers.get(event)?.delete(observer);
    }

    unsubscribe(event, observer) {
        this.#observers.get(event)?.delete(observer);
    }

    notify(event, data) {
        this.#observers.get(event)?.forEach(observer => {
            try {
                observer(data);
            } catch (error) {
                console.error(`Observer error on "${event}":`, error);
            }
        });
    }

    get observerCount() {
        let count = 0;
        for (const set of this.#observers.values()) count += set.size;
        return count;
    }
}

/**
 * Pathao রাইড — Subject হিসেবে
 */
class PathaoRide extends Subject {
    #status = 'searching';
    #riderInfo = null;

    constructor(passengerId, pickup, destination) {
        super();
        this.passengerId = passengerId;
        this.pickup = pickup;
        this.destination = destination;
        this.createdAt = new Date();
    }

    get status() { return this.#status; }

    assignRider(riderInfo) {
        this.#riderInfo = riderInfo;
        this.#status = 'rider_assigned';
        this.notify('statusChange', {
            status: this.#status,
            rider: this.#riderInfo,
            message: `রাইডার ${riderInfo.name} আসছেন আপনার কাছে`,
        });
    }

    startTrip() {
        this.#status = 'trip_started';
        this.notify('statusChange', {
            status: this.#status,
            message: 'যাত্রা শুরু হয়েছে!',
        });
    }

    completeTrip(fare) {
        this.#status = 'completed';
        this.notify('statusChange', {
            status: this.#status,
            fare,
            message: `যাত্রা সম্পন্ন। ভাড়া: ৳${fare}`,
        });
        this.notify('tripCompleted', { fare, ride: this });
    }
}

// === Observers ===
const passengerAppUI = (data) => {
    console.log(`📱 [Passenger App] ${data.message}`);
};

const riderAppUI = (data) => {
    console.log(`🏍️ [Rider App] স্ট্যাটাস: ${data.status}`);
};

const analyticsTracker = (data) => {
    console.log(`📊 [Analytics] ইভেন্ট ট্র্যাক: ${JSON.stringify(data)}`);
};

const billingSystem = ({ fare, ride }) => {
    const commission = fare * 0.20; // ২০% কমিশন
    console.log(`💰 [Billing] ভাড়া: ৳${fare}, কমিশন: ৳${commission}`);
};

// === ব্যবহার ===
const ride = new PathaoRide('P123', 'ধানমন্ডি ২৭', 'গুলশান ১');

const unsubPassenger = ride.subscribe('statusChange', passengerAppUI);
ride.subscribe('statusChange', riderAppUI);
ride.subscribe('statusChange', analyticsTracker);
ride.subscribe('tripCompleted', billingSystem);

ride.assignRider({ name: 'করিম', phone: '01911XXXXXX', rating: 4.8 });
ride.startTrip();
ride.completeTrip(180);

// Unsubscribe example — মেমরি লিক প্রতিরোধ
unsubPassenger();
console.log(`Active observers: ${ride.observerCount}`);
```

---

### 3️⃣ Event Emitter Implementation

#### PHP 8.3 — Type-safe Event Emitter

```php
<?php

declare(strict_types=1);

/**
 * Generic Event Emitter — PHP 8.3 (Typed Properties + Enums + First-class Callables)
 */
enum EventPriority: int
{
    case LOW    = 0;
    case NORMAL = 50;
    case HIGH   = 100;
    case URGENT = 200;
}

readonly class EventListener
{
    public function __construct(
        public \Closure $callback,
        public EventPriority $priority = EventPriority::NORMAL,
        public bool $once = false,
    ) {}
}

class EventEmitter
{
    /** @var array<string, \SplPriorityQueue<EventListener>> */
    private array $listeners = [];
    private array $onceFlags = [];

    public function on(
        string $event,
        \Closure $callback,
        EventPriority $priority = EventPriority::NORMAL,
    ): self {
        $this->addListener($event, new EventListener($callback, $priority));
        return $this;
    }

    public function once(string $event, \Closure $callback): self
    {
        $this->addListener($event, new EventListener($callback, once: true));
        return $this;
    }

    public function off(string $event, ?\Closure $callback = null): self
    {
        if ($callback === null) {
            unset($this->listeners[$event]);
            return $this;
        }

        if (!isset($this->listeners[$event])) return $this;

        $this->listeners[$event] = array_filter(
            $this->listeners[$event],
            fn(EventListener $l) => $l->callback !== $callback
        );
        return $this;
    }

    public function emit(string $event, mixed ...$args): self
    {
        if (!isset($this->listeners[$event])) return $this;

        // Priority অনুযায়ী sort করে execute করা
        $sorted = $this->listeners[$event];
        usort($sorted, fn(EventListener $a, EventListener $b) =>
            $b->priority->value <=> $a->priority->value
        );

        $toRemove = [];
        foreach ($sorted as $index => $listener) {
            ($listener->callback)(...$args);
            if ($listener->once) {
                $toRemove[] = $index;
            }
        }

        // once listener সরিয়ে ফেলা
        foreach (array_reverse($toRemove) as $idx) {
            array_splice($this->listeners[$event], $idx, 1);
        }

        return $this;
    }

    public function listenerCount(string $event): int
    {
        return count($this->listeners[$event] ?? []);
    }

    private function addListener(string $event, EventListener $listener): void
    {
        $this->listeners[$event] ??= [];
        $this->listeners[$event][] = $listener;
    }
}

// === ব্যবহার: E-commerce Order System ===
$emitter = new EventEmitter();

// High priority — ইনভেন্টরি প্রথমে আপডেট হবে
$emitter->on('order.placed', function (array $order) {
    printf("📦 [Inventory] %d টি পণ্য ইনভেন্টরি থেকে কমানো হলো\n", count($order['items']));
}, EventPriority::HIGH);

// Normal priority — ইমেইল
$emitter->on('order.placed', function (array $order) {
    printf("📧 [Email] অর্ডার কনফার্মেশন পাঠানো হলো: %s-এ\n", $order['email']);
});

// Once — প্রথম অর্ডারে স্বাগতম বোনাস
$emitter->once('order.placed', function (array $order) {
    printf("🎁 [Bonus] প্রথম অর্ডার বোনাস: ৳50 ক্যাশব্যাক!\n");
});

$emitter->emit('order.placed', [
    'id'    => 'ORD-001',
    'items' => ['shirt', 'pants'],
    'email' => 'user@example.com',
    'total' => 2500.00,
]);
```

#### JavaScript ES2022+ — Advanced Event Emitter

```javascript
// JavaScript ES2022+ — Advanced EventEmitter with AbortController, Async, WeakRef

class AdvancedEventEmitter {
    #listeners = new Map();
    #maxListeners = 10;

    on(event, callback, { priority = 0, signal } = {}) {
        if (!this.#listeners.has(event)) {
            this.#listeners.set(event, []);
        }

        const listener = { callback, priority, once: false };
        this.#listeners.get(event).push(listener);
        this.#listeners.get(event).sort((a, b) => b.priority - a.priority);

        // AbortController সাপোর্ট — ক্লিনআপ সহজ করে
        signal?.addEventListener('abort', () => this.off(event, callback), { once: true });

        this.#warnIfTooMany(event);

        return () => this.off(event, callback);
    }

    once(event, callback) {
        const wrapper = (...args) => {
            this.off(event, wrapper);
            return callback(...args);
        };
        wrapper._original = callback;
        return this.on(event, wrapper);
    }

    off(event, callback) {
        const listeners = this.#listeners.get(event);
        if (!listeners) return;
        const idx = listeners.findIndex(
            l => l.callback === callback || l.callback._original === callback
        );
        if (idx !== -1) listeners.splice(idx, 1);
    }

    async emit(event, ...args) {
        const listeners = this.#listeners.get(event);
        if (!listeners?.length) return false;

        const results = [];
        for (const { callback } of [...listeners]) {
            try {
                results.push(await callback(...args));
            } catch (error) {
                this.emit('error', { event, error, args });
            }
        }
        return results;
    }

    // Promise-based — একবারের জন্য ইভেন্ট অপেক্ষা
    waitFor(event, { timeout = 5000 } = {}) {
        return new Promise((resolve, reject) => {
            const timer = setTimeout(() => {
                this.off(event, handler);
                reject(new Error(`"${event}" ইভেন্ট টাইমআউট (${timeout}ms)`));
            }, timeout);

            const handler = (data) => {
                clearTimeout(timer);
                resolve(data);
            };
            this.once(event, handler);
        });
    }

    #warnIfTooMany(event) {
        const count = this.#listeners.get(event)?.length ?? 0;
        if (count > this.#maxListeners) {
            console.warn(
                `⚠️ "${event}" ইভেন্টে ${count} টি listener আছে। মেমরি লিক হতে পারে!`
            );
        }
    }
}

// === ব্যবহার ===
const emitter = new AdvancedEventEmitter();
const controller = new AbortController();

emitter.on('order:created', async (order) => {
    console.log(`📦 ইনভেন্টরি আপডেট: অর্ডার ${order.id}`);
}, { priority: 10, signal: controller.signal });

emitter.on('order:created', async (order) => {
    // সিমুলেটেড async অপারেশন
    await new Promise(r => setTimeout(r, 100));
    console.log(`📧 ইমেইল পাঠানো হয়েছে: ${order.email}`);
}, { signal: controller.signal });

await emitter.emit('order:created', { id: 'ORD-42', email: 'buyer@bd.com' });

// সব listener একবারে cleanup — AbortController দিয়ে
controller.abort();
```

---

### 4️⃣ Reactive Streams — Basic Observable

#### PHP 8.3 — Reactive Observable

```php
<?php

declare(strict_types=1);

/**
 * Lightweight Reactive Observable — PHP 8.3
 * RxPHP-এর সরলীকৃত সংস্করণ
 */
class Observable
{
    private \Closure $subscribeFn;

    public function __construct(\Closure $subscribeFn)
    {
        $this->subscribeFn = $subscribeFn;
    }

    public function subscribe(
        ?\Closure $onNext = null,
        ?\Closure $onError = null,
        ?\Closure $onComplete = null,
    ): Subscription {
        $subscriber = new Subscriber(
            onNext: $onNext ?? fn() => null,
            onError: $onError ?? fn(\Throwable $e) => throw $e,
            onComplete: $onComplete ?? fn() => null,
        );

        ($this->subscribeFn)($subscriber);
        return new Subscription($subscriber);
    }

    // --- Operators ---

    public function map(\Closure $transform): self
    {
        return new self(function (Subscriber $subscriber) use ($transform) {
            $this->subscribe(
                onNext: fn($value) => $subscriber->next($transform($value)),
                onError: fn($e) => $subscriber->error($e),
                onComplete: fn() => $subscriber->complete(),
            );
        });
    }

    public function filter(\Closure $predicate): self
    {
        return new self(function (Subscriber $subscriber) use ($predicate) {
            $this->subscribe(
                onNext: function ($value) use ($subscriber, $predicate) {
                    if ($predicate($value)) {
                        $subscriber->next($value);
                    }
                },
                onError: fn($e) => $subscriber->error($e),
                onComplete: fn() => $subscriber->complete(),
            );
        });
    }

    public static function fromArray(array $items): self
    {
        return new self(function (Subscriber $subscriber) use ($items) {
            foreach ($items as $item) {
                $subscriber->next($item);
            }
            $subscriber->complete();
        });
    }
}

class Subscriber
{
    private bool $closed = false;

    public function __construct(
        private readonly \Closure $onNext,
        private readonly \Closure $onError,
        private readonly \Closure $onComplete,
    ) {}

    public function next(mixed $value): void
    {
        if ($this->closed) return;
        ($this->onNext)($value);
    }

    public function error(\Throwable $e): void
    {
        if ($this->closed) return;
        $this->closed = true;
        ($this->onError)($e);
    }

    public function complete(): void
    {
        if ($this->closed) return;
        $this->closed = true;
        ($this->onComplete)();
    }

    public function unsubscribe(): void { $this->closed = true; }
}

class Subscription
{
    public function __construct(private Subscriber $subscriber) {}
    public function unsubscribe(): void { $this->subscriber->unsubscribe(); }
}

// === ব্যবহার: Stock Price Stream ===
$stockPrices = Observable::fromArray([
    ['symbol' => 'BRAC', 'price' => 42.50],
    ['symbol' => 'GP',   'price' => 320.75],
    ['symbol' => 'BRAC', 'price' => 43.10],
    ['symbol' => 'GP',   'price' => 318.20],
    ['symbol' => 'BRAC', 'price' => 44.00],
]);

$stockPrices
    ->filter(fn($stock) => $stock['symbol'] === 'BRAC')
    ->map(fn($stock) => sprintf("BRAC Bank: ৳%.2f", $stock['price']))
    ->subscribe(
        onNext: fn($msg) => print("📈 $msg\n"),
        onComplete: fn() => print("✅ স্ট্রিম শেষ\n"),
    );

// Output:
// 📈 BRAC Bank: ৳42.50
// 📈 BRAC Bank: ৳43.10
// 📈 BRAC Bank: ৳44.00
// ✅ স্ট্রিম শেষ
```

#### JavaScript ES2022+ — Reactive Observable

```javascript
// JavaScript ES2022+ — Reactive Observable with Symbol.observable

class Observable {
    #subscribeFn;

    constructor(subscribeFn) {
        this.#subscribeFn = subscribeFn;
    }

    subscribe(observerOrNext, error, complete) {
        const observer = typeof observerOrNext === 'function'
            ? { next: observerOrNext, error: error ?? (e => { throw e; }), complete: complete ?? (() => {}) }
            : observerOrNext;

        const subscriber = new Subscriber(observer);
        this.#subscribeFn(subscriber);
        return subscriber;
    }

    // Operators
    pipe(...operators) {
        return operators.reduce((obs, op) => op(obs), this);
    }

    static from(iterable) {
        return new Observable(subscriber => {
            try {
                for (const item of iterable) {
                    if (subscriber.closed) return;
                    subscriber.next(item);
                }
                subscriber.complete();
            } catch (e) {
                subscriber.error(e);
            }
        });
    }

    static interval(ms) {
        return new Observable(subscriber => {
            let count = 0;
            const id = setInterval(() => subscriber.next(count++), ms);
            subscriber.add(() => clearInterval(id));
        });
    }
}

class Subscriber {
    #closed = false;
    #teardowns = [];
    #observer;

    constructor(observer) { this.#observer = observer; }

    get closed() { return this.#closed; }

    next(value) {
        if (!this.#closed) this.#observer.next(value);
    }

    error(err) {
        if (this.#closed) return;
        this.#closed = true;
        this.#observer.error(err);
        this.#runTeardowns();
    }

    complete() {
        if (this.#closed) return;
        this.#closed = true;
        this.#observer.complete();
        this.#runTeardowns();
    }

    add(teardown) { this.#teardowns.push(teardown); }
    unsubscribe() { this.#closed = true; this.#runTeardowns(); }
    #runTeardowns() { this.#teardowns.forEach(fn => fn()); }
}

// Operators (RxJS-স্টাইল)
const map = (fn) => (source) =>
    new Observable(subscriber => {
        source.subscribe({
            next: val => subscriber.next(fn(val)),
            error: err => subscriber.error(err),
            complete: () => subscriber.complete(),
        });
    });

const filter = (predicate) => (source) =>
    new Observable(subscriber => {
        source.subscribe({
            next: val => predicate(val) && subscriber.next(val),
            error: err => subscriber.error(err),
            complete: () => subscriber.complete(),
        });
    });

const take = (count) => (source) =>
    new Observable(subscriber => {
        let taken = 0;
        source.subscribe({
            next: val => { if (++taken <= count) subscriber.next(val); if (taken >= count) subscriber.complete(); },
            error: err => subscriber.error(err),
            complete: () => subscriber.complete(),
        });
    });

// === ব্যবহার: ঢাকা স্টক এক্সচেঞ্জ (DSE) Price Stream ===
const dseStocks = Observable.from([
    { symbol: 'GP', price: 320.5, change: +2.3 },
    { symbol: 'BRAC', price: 42.1, change: -0.5 },
    { symbol: 'GP', price: 322.0, change: +1.5 },
    { symbol: 'SQUARE', price: 198.0, change: +5.0 },
    { symbol: 'GP', price: 325.0, change: +3.0 },
]);

dseStocks.pipe(
    filter(s => s.symbol === 'GP'),
    map(s => `GP: ৳${s.price} (${s.change > 0 ? '📈' : '📉'} ${s.change})`),
    take(2),
).subscribe({
    next: msg => console.log(msg),
    complete: () => console.log('✅ ২টি GP আপডেট দেখানো হলো'),
});
```

---

## 🌍 Real-World Applicable Areas

### 1. Laravel Model Observers

Laravel-এ Model Observer ব্যবহার করে Eloquent model-এর lifecycle events-এ hook করা যায়:

```php
<?php

// php artisan make:observer OrderObserver --model=Order

namespace App\Observers;

use App\Models\Order;
use App\Services\InventoryService;
use App\Services\NotificationService;
use App\Jobs\SendOrderConfirmationEmail;
use Illuminate\Support\Facades\Log;

class OrderObserver
{
    public function __construct(
        private readonly InventoryService $inventory,
        private readonly NotificationService $notifications,
    ) {}

    /**
     * creating — ডাটাবেসে save হওয়ার আগে
     * Validation, default value সেট করা
     */
    public function creating(Order $order): void
    {
        $order->order_number = 'ORD-' . now()->format('Ymd') . '-' . str_pad(
            (string) Order::whereDate('created_at', today())->count() + 1,
            4, '0', STR_PAD_LEFT
        );
        $order->status = 'pending';

        Log::info("Order creating: {$order->order_number}");
    }

    /**
     * created — ডাটাবেসে save হওয়ার পরে
     * Side effects: notification, queue jobs
     */
    public function created(Order $order): void
    {
        // ইনভেন্টরি রিজার্ভ
        $this->inventory->reserveItems($order->items);

        // ইমেইল queue-তে পাঠানো
        SendOrderConfirmationEmail::dispatch($order);

        // রিয়েল-টাইম নোটিফিকেশন
        $this->notifications->push(
            userId: $order->user_id,
            message: "আপনার অর্ডার #{$order->order_number} গ্রহণ করা হয়েছে!",
        );
    }

    /**
     * updating — আপডেট হওয়ার আগে
     */
    public function updating(Order $order): void
    {
        // status পরিবর্তন ট্র্যাক করা
        if ($order->isDirty('status')) {
            $order->status_changed_at = now();

            Log::info("Order {$order->order_number}: {$order->getOriginal('status')} → {$order->status}");
        }
    }

    /**
     * updated — আপডেট হওয়ার পরে
     */
    public function updated(Order $order): void
    {
        if ($order->wasChanged('status') && $order->status === 'shipped') {
            $this->notifications->sms(
                phone: $order->user->phone,
                message: "আপনার অর্ডার #{$order->order_number} শিপ করা হয়েছে!",
            );
        }
    }

    /**
     * deleting — মুছে ফেলার আগে
     */
    public function deleting(Order $order): void
    {
        // Soft delete হলে ইনভেন্টরি ফেরত
        $this->inventory->releaseReservedItems($order->items);
    }
}

// === AppServiceProvider-এ রেজিস্টার ===
// Order::observe(OrderObserver::class);
```

### 2. Laravel Events & Listeners

```php
<?php

// Event — কী ঘটেছে তা describe করে
namespace App\Events;

use App\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\InteractsWithSockets;

class OrderPlaced
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order,
        public readonly float $totalAmount,
    ) {}
}

// Listener ১ — ইমেইল
namespace App\Listeners;

use App\Events\OrderPlaced;
use App\Mail\OrderConfirmation;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;

class SendOrderConfirmationEmail implements ShouldQueue
{
    public int $tries = 3;
    public int $backoff = 60;

    public function handle(OrderPlaced $event): void
    {
        Mail::to($event->order->user->email)
            ->send(new OrderConfirmation($event->order));
    }

    public function failed(OrderPlaced $event, \Throwable $exception): void
    {
        // ফেইল হলে SMS পাঠানো
        app('sms')->send(
            $event->order->user->phone,
            "অর্ডার কনফার্মেশন ইমেইল পাঠাতে সমস্যা হচ্ছে। অর্ডার নম্বর: {$event->order->order_number}"
        );
    }
}

// Listener ২ — Analytics
class TrackOrderAnalytics implements ShouldQueue
{
    public function handle(OrderPlaced $event): void
    {
        analytics()->track('order_placed', [
            'order_id'   => $event->order->id,
            'amount'     => $event->totalAmount,
            'items'      => $event->order->items->count(),
            'user_type'  => $event->order->user->is_premium ? 'premium' : 'regular',
        ]);
    }
}

// EventServiceProvider-এ রেজিস্টার:
// OrderPlaced::class => [
//     SendOrderConfirmationEmail::class,
//     TrackOrderAnalytics::class,
//     UpdateInventory::class,
//     NotifyWarehouse::class,
// ],

// ডিসপ্যাচ:
// OrderPlaced::dispatch($order, $order->total);
```

### 3. Node.js EventEmitter

```javascript
// Node.js EventEmitter — Core Module ব্যবহার

import { EventEmitter } from 'node:events';

class OrderService extends EventEmitter {
    #orders = new Map();

    async placeOrder(orderData) {
        const order = {
            id: `ORD-${Date.now()}`,
            ...orderData,
            status: 'placed',
            createdAt: new Date(),
        };

        this.#orders.set(order.id, order);

        // সব observers-কে জানানো হচ্ছে
        this.emit('order:placed', order);
        return order;
    }

    async shipOrder(orderId) {
        const order = this.#orders.get(orderId);
        if (!order) throw new Error(`অর্ডার ${orderId} পাওয়া যায়নি`);

        order.status = 'shipped';
        order.shippedAt = new Date();

        this.emit('order:shipped', order);
        return order;
    }
}

// === Observers রেজিস্টার ===
const orderService = new OrderService();

// ইনভেন্টরি
orderService.on('order:placed', (order) => {
    console.log(`📦 ইনভেন্টরি আপডেট: ${order.items.length} টি আইটেম রিজার্ভ`);
});

// ইমেইল
orderService.on('order:placed', async (order) => {
    // await sendEmail(order.email, 'Order Confirmation', ...);
    console.log(`📧 কনফার্মেশন ইমেইল: ${order.email}`);
});

// শিপিং নোটিফিকেশন
orderService.on('order:shipped', (order) => {
    console.log(`🚚 শিপিং SMS পাঠানো হচ্ছে: ${order.phone}`);
});

// Error handling — গুরুত্বপূর্ণ!
orderService.on('error', (err) => {
    console.error('⚠️ OrderService Error:', err.message);
});

// ব্যবহার
await orderService.placeOrder({
    items: [{ name: 'পাঞ্জাবি', qty: 1 }],
    email: 'buyer@example.com',
    phone: '01711XXXXXX',
});
```

### 4. DOM Event Listeners

```javascript
// DOM Observer Pattern — ব্রাউজার Environment

// ===  Shopping Cart Price Update ===
class ShoppingCart extends EventTarget {
    #items = [];

    addItem(item) {
        this.#items.push(item);
        this.dispatchEvent(new CustomEvent('cart:updated', {
            detail: { action: 'add', item, total: this.total },
        }));
    }

    removeItem(itemId) {
        this.#items = this.#items.filter(i => i.id !== itemId);
        this.dispatchEvent(new CustomEvent('cart:updated', {
            detail: { action: 'remove', itemId, total: this.total },
        }));
    }

    get total() {
        return this.#items.reduce((sum, item) => sum + item.price * item.qty, 0);
    }
}

const cart = new ShoppingCart();
const controller = new AbortController();

// UI Observer
cart.addEventListener('cart:updated', (e) => {
    document.querySelector('#cart-total').textContent = `৳${e.detail.total}`;
    document.querySelector('#cart-badge').textContent = e.detail.action === 'add' ? '+1' : '-1';
}, { signal: controller.signal });

// Analytics Observer
cart.addEventListener('cart:updated', (e) => {
    gtag('event', 'cart_update', { action: e.detail.action, value: e.detail.total });
}, { signal: controller.signal });

// ক্লিনআপ — পেজ ছেড়ে গেলে সব listener বাতিল
window.addEventListener('beforeunload', () => controller.abort());
```

### 5. WebSocket Real-Time Notifications

```javascript
// WebSocket + Observer — রিয়েল-টাইম নোটিফিকেশন সিস্টেম

class WebSocketObservable {
    #ws = null;
    #emitter = new EventTarget();
    #reconnectAttempts = 0;
    #maxReconnect = 5;

    constructor(url) {
        this.url = url;
        this.connect();
    }

    connect() {
        this.#ws = new WebSocket(this.url);

        this.#ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.#emitter.dispatchEvent(
                new CustomEvent(data.type, { detail: data.payload })
            );
        };

        this.#ws.onclose = () => {
            if (this.#reconnectAttempts < this.#maxReconnect) {
                const delay = Math.min(1000 * 2 ** this.#reconnectAttempts, 30000);
                console.log(`🔄 ${delay}ms পরে পুনরায় সংযোগ করা হচ্ছে...`);
                setTimeout(() => { this.#reconnectAttempts++; this.connect(); }, delay);
            }
        };
    }

    on(eventType, callback) {
        const handler = (e) => callback(e.detail);
        this.#emitter.addEventListener(eventType, handler);
        return () => this.#emitter.removeEventListener(eventType, handler);
    }

    disconnect() {
        this.#maxReconnect = 0;
        this.#ws?.close();
    }
}

// === ব্যবহার ===
const socket = new WebSocketObservable('wss://api.example.com/ws');

const unsubOrder = socket.on('order:status', (data) => {
    console.log(`📦 অর্ডার ${data.orderId}: ${data.status}`);
});

socket.on('notification', (data) => {
    showToast(data.message);
});

// ক্লিনআপ
// unsubOrder();
// socket.disconnect();
```

### 6. Full Working Example: E-commerce Order Observer

```php
<?php

declare(strict_types=1);

// === সম্পূর্ণ E-commerce Order Observer System — PHP 8.3 ===

interface OrderEventInterface
{
    public function getOrderId(): string;
    public function getEventName(): string;
    public function getPayload(): array;
}

readonly class OrderEvent implements OrderEventInterface
{
    public function __construct(
        private string $orderId,
        private string $eventName,
        private array $payload,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}

    public function getOrderId(): string { return $this->orderId; }
    public function getEventName(): string { return $this->eventName; }
    public function getPayload(): array { return $this->payload; }
}

interface OrderObserverInterface
{
    public function handle(OrderEventInterface $event): void;
    public function subscribedEvents(): array;
}

class OrderSubject
{
    /** @var array<string, OrderObserverInterface[]> */
    private array $observers = [];

    public function attach(OrderObserverInterface $observer): void
    {
        foreach ($observer->subscribedEvents() as $event) {
            $this->observers[$event] ??= [];
            $this->observers[$event][] = $observer;
        }
    }

    public function detach(OrderObserverInterface $observer): void
    {
        foreach ($this->observers as $event => &$list) {
            $list = array_filter($list, fn($o) => $o !== $observer);
        }
    }

    public function dispatch(OrderEventInterface $event): void
    {
        $eventName = $event->getEventName();
        foreach ($this->observers[$eventName] ?? [] as $observer) {
            try {
                $observer->handle($event);
            } catch (\Throwable $e) {
                error_log("Observer error [{$eventName}]: {$e->getMessage()}");
            }
        }
    }
}

// --- Concrete Observers ---

class InventoryObserver implements OrderObserverInterface
{
    public function subscribedEvents(): array
    {
        return ['order.placed', 'order.cancelled'];
    }

    public function handle(OrderEventInterface $event): void
    {
        $payload = $event->getPayload();
        match ($event->getEventName()) {
            'order.placed' => printf(
                "📦 [Inventory] %d আইটেম রিজার্ভ করা হলো — অর্ডার: %s\n",
                count($payload['items']),
                $event->getOrderId()
            ),
            'order.cancelled' => printf(
                "📦 [Inventory] আইটেম রিলিজ করা হলো — অর্ডার: %s\n",
                $event->getOrderId()
            ),
        };
    }
}

class EmailObserver implements OrderObserverInterface
{
    public function subscribedEvents(): array
    {
        return ['order.placed', 'order.shipped', 'order.delivered'];
    }

    public function handle(OrderEventInterface $event): void
    {
        $email = $event->getPayload()['customer_email'] ?? 'unknown';
        printf("📧 [Email] %s → %s — অর্ডার: %s\n",
            $event->getEventName(), $email, $event->getOrderId());
    }
}

class AnalyticsObserver implements OrderObserverInterface
{
    private array $metrics = [];

    public function subscribedEvents(): array
    {
        return ['order.placed', 'order.shipped', 'order.delivered', 'order.cancelled'];
    }

    public function handle(OrderEventInterface $event): void
    {
        $this->metrics[] = [
            'event'   => $event->getEventName(),
            'orderId' => $event->getOrderId(),
            'time'    => $event->occurredAt->format('H:i:s'),
        ];
        printf("📊 [Analytics] ট্র্যাক: %s — অর্ডার: %s\n",
            $event->getEventName(), $event->getOrderId());
    }

    public function getMetrics(): array { return $this->metrics; }
}

// === চালানো ===
$subject = new OrderSubject();
$subject->attach(new InventoryObserver());
$subject->attach(new EmailObserver());
$subject->attach($analytics = new AnalyticsObserver());

echo "=== অর্ডার প্লেস ===\n";
$subject->dispatch(new OrderEvent('ORD-001', 'order.placed', [
    'items'          => ['laptop', 'mouse'],
    'customer_email' => 'rahim@bd.com',
    'total'          => 85_000.00,
]));

echo "\n=== অর্ডার শিপ ===\n";
$subject->dispatch(new OrderEvent('ORD-001', 'order.shipped', [
    'customer_email' => 'rahim@bd.com',
    'tracking'       => 'BD-TRACK-12345',
]));
```

---

## 🔥 Advanced Deep Dive

### 1. Observer vs Pub/Sub — তুলনা

```
┌─────────────────────────┬──────────────────────────────────────┐
│       Observer           │           Pub/Sub                    │
├─────────────────────────┼──────────────────────────────────────┤
│ Subject Observer-কে     │ Publisher Subscriber-কে              │
│ সরাসরি জানে            │ জানে না (Broker মধ্যস্থতা করে)      │
│                         │                                      │
│ Tight coupling          │ Loose coupling                       │
│ (Subject → Observer)    │ (Publisher → Broker → Subscriber)    │
│                         │                                      │
│ Synchronous             │ Synchronous বা Asynchronous          │
│ (সাধারণত)              │                                      │
│                         │                                      │
│ Same process            │ Cross-process / Cross-service        │
│                         │ সম্ভব                               │
│                         │                                      │
│ উদাহরণ: DOM events,    │ উদাহরণ: Redis Pub/Sub,              │
│ Laravel Observer        │ RabbitMQ, Kafka, Laravel Events      │
└─────────────────────────┴──────────────────────────────────────┘
```

```
Observer Pattern:
    Subject ──────► Observer A
       │──────────► Observer B
       └──────────► Observer C
    (Subject সরাসরি Observer-দের reference রাখে)

Pub/Sub Pattern:
    Publisher ──► [Message Broker/Event Bus] ──► Subscriber A
                                             ──► Subscriber B
                                             ──► Subscriber C
    (Publisher এবং Subscriber একে অপরকে জানে না)
```

### 2. Observer vs Mediator

```
Observer:
    বহু Observer এক Subject-এর পরিবর্তন শোনে।
    Subject জানে observers আছে, কিন্তু কী করবে জানে না।
    Communication: One-to-Many, একমুখী

Mediator:
    বহু অবজেক্ট একটি Mediator-এর মাধ্যমে যোগাযোগ করে।
    কেউ কাউকে সরাসরি জানে না।
    Communication: Many-to-Many, দ্বিমুখী

Observer → "আমি পরিবর্তন হয়েছি, সবাই জানো!"
Mediator → "হে মধ্যস্থতাকারী, B-কে বলো আমি ready!"
```

### 3. Push vs Pull Observer

```php
<?php

// === Push Model ===
// Subject পরিবর্তিত ডেটা সরাসরি Observer-কে পাঠিয়ে দেয়
interface PushObserver
{
    public function update(string $event, array $data): void;
}

class PushSubject
{
    private array $observers = [];

    public function notify(string $event, array $data): void
    {
        foreach ($this->observers as $observer) {
            $observer->update($event, $data); // ডেটা push করা হচ্ছে
        }
    }
}

// === Pull Model ===
// Subject শুধু জানায়, Observer নিজে এসে ডেটা নিয়ে যায়
interface PullObserver
{
    public function update(PullSubject $subject): void;
}

class PullSubject
{
    private array $observers = [];
    private string $state = '';

    public function getState(): string { return $this->state; }

    public function notify(): void
    {
        foreach ($this->observers as $observer) {
            $observer->update($this); // Observer নিজে state pull করবে
        }
    }
}
```

```
Push Model:
  ✅ Observer-কে Subject-এর ইন্টারনাল জানতে হয় না
  ✅ দ্রুত — ডেটা সাথে সাথে পৌঁছে যায়
  ❌ অপ্রয়োজনীয় ডেটাও পাঠাতে পারে
  ❌ Subject-কে জানতে হয় Observer কী চায়

Pull Model:
  ✅ Observer শুধু যা দরকার তাই নেয়
  ✅ Subject সহজ — শুধু নোটিফাই করে
  ❌ Observer-কে Subject-এর API জানতে হয়
  ❌ অতিরিক্ত coupling তৈরি হতে পারে
```

### 4. Async Observer

```javascript
// JavaScript — Async Observer Pattern

class AsyncEventBus {
    #handlers = new Map();

    on(event, handler) {
        if (!this.#handlers.has(event)) this.#handlers.set(event, []);
        this.#handlers.get(event).push(handler);
        return () => this.off(event, handler);
    }

    off(event, handler) {
        const handlers = this.#handlers.get(event);
        if (handlers) {
            const idx = handlers.indexOf(handler);
            if (idx !== -1) handlers.splice(idx, 1);
        }
    }

    // Parallel — সব observer একসাথে চলে
    async emitParallel(event, data) {
        const handlers = this.#handlers.get(event) ?? [];
        const results = await Promise.allSettled(
            handlers.map(h => h(data))
        );

        // ফেইল হওয়া handler-গুলো লগ করা
        results.forEach((result, i) => {
            if (result.status === 'rejected') {
                console.error(`Handler ${i} failed for "${event}":`, result.reason);
            }
        });

        return results;
    }

    // Sequential — একটার পর একটা চলে (ordering গুরুত্বপূর্ণ হলে)
    async emitSequential(event, data) {
        const handlers = this.#handlers.get(event) ?? [];
        const results = [];
        for (const handler of handlers) {
            try {
                results.push({ status: 'fulfilled', value: await handler(data) });
            } catch (error) {
                results.push({ status: 'rejected', reason: error });
            }
        }
        return results;
    }
}

// === ব্যবহার ===
const bus = new AsyncEventBus();

bus.on('payment:received', async (data) => {
    await new Promise(r => setTimeout(r, 200));
    console.log(`💰 পেমেন্ট প্রসেস: ৳${data.amount}`);
});

bus.on('payment:received', async (data) => {
    await new Promise(r => setTimeout(r, 100));
    console.log(`📧 রিসিট ইমেইল পাঠানো হচ্ছে`);
});

bus.on('payment:received', async (data) => {
    console.log(`📊 Analytics ট্র্যাক হচ্ছে`);
});

// Parallel: তিনটাই একসাথে শুরু হবে (~200ms)
await bus.emitParallel('payment:received', { amount: 5000 });

// Sequential: একটার পর একটা (~300ms)
await bus.emitSequential('payment:received', { amount: 5000 });
```

### 5. PHP Fibers + Observer

```php
<?php

declare(strict_types=1);

// PHP 8.3 Fibers — Non-blocking Observer

class FiberObserverBus
{
    private array $listeners = [];
    private array $fibers = [];

    public function on(string $event, \Closure $handler): void
    {
        $this->listeners[$event] ??= [];
        $this->listeners[$event][] = $handler;
    }

    /**
     * Fiber-based dispatch — প্রতিটি observer আলাদা Fiber-এ চলবে
     * Non-blocking: একটা observer block করলে অন্যটায় সুইচ করা যাবে
     */
    public function dispatch(string $event, mixed $data): void
    {
        foreach ($this->listeners[$event] ?? [] as $handler) {
            $fiber = new \Fiber(function () use ($handler, $data) {
                $handler($data);
            });

            $fiber->start();
            $this->fibers[] = $fiber;
        }

        // সব fiber শেষ করা
        $this->drainFibers();
    }

    private function drainFibers(): void
    {
        while (!empty($this->fibers)) {
            foreach ($this->fibers as $key => $fiber) {
                if ($fiber->isTerminated()) {
                    unset($this->fibers[$key]);
                    continue;
                }
                if ($fiber->isSuspended()) {
                    $fiber->resume();
                }
            }
            $this->fibers = array_values($this->fibers);
        }
    }
}

// Simulated I/O operation যা Fiber.suspend() করে
function simulateAsyncIO(string $label, int $steps = 3): void
{
    for ($i = 1; $i <= $steps; $i++) {
        printf("  [%s] ধাপ %d/%d\n", $label, $i, $steps);
        \Fiber::suspend(); // অন্য fiber-কে সুযোগ দেওয়া
    }
}

$bus = new FiberObserverBus();

$bus->on('order.created', function ($order) {
    echo "📦 Inventory শুরু...\n";
    simulateAsyncIO('Inventory', 2);
    echo "📦 Inventory সম্পন্ন\n";
});

$bus->on('order.created', function ($order) {
    echo "📧 Email শুরু...\n";
    simulateAsyncIO('Email', 3);
    echo "📧 Email সম্পন্ন\n";
});

$bus->dispatch('order.created', ['id' => 'ORD-001']);

// Output (interleaved):
// 📦 Inventory শুরু...
//   [Inventory] ধাপ 1/2
// 📧 Email শুরু...
//   [Email] ধাপ 1/3
//   [Inventory] ধাপ 2/2
//   [Email] ধাপ 2/3
// 📦 Inventory সম্পন্ন
//   [Email] ধাপ 3/3
// 📧 Email সম্পন্ন
```

### 6. Memory Leaks — ভুলে যাওয়া সাবস্ক্রিপশন

মেমরি লিক হলো Observer প্যাটার্নের **সবচেয়ে বড় সমস্যা**। Observer রেজিস্টার করে unsubscribe না করলে:

```javascript
// ❌ মেমরি লিক — খুবই সাধারণ ভুল!
class UserDashboard {
    constructor(eventBus) {
        // প্রতিবার নতুন Dashboard তৈরি হলে listener যোগ হয়
        // কিন্তু পুরনোটা সরানো হয় না!
        eventBus.on('price:update', (data) => {
            this.updatePriceUI(data);
        });
    }

    updatePriceUI(data) { /* ... */ }
}

// ✅ সমাধান ১: Unsubscribe ফাংশন ব্যবহার
class FixedDashboard {
    #unsubscribers = [];

    constructor(eventBus) {
        this.#unsubscribers.push(
            eventBus.on('price:update', (data) => this.updatePriceUI(data))
        );
    }

    destroy() {
        this.#unsubscribers.forEach(unsub => unsub());
        this.#unsubscribers = [];
    }
}

// ✅ সমাধান ২: AbortController (আধুনিক পদ্ধতি)
class ModernDashboard {
    #controller = new AbortController();

    constructor(eventBus) {
        eventBus.on('price:update', (data) => this.updatePriceUI(data), {
            signal: this.#controller.signal,
        });
    }

    destroy() {
        this.#controller.abort(); // সব listener একবারে বাতিল!
    }
}

// ✅ সমাধান ৩: WeakRef (Observer গার্বেজ কালেক্ট হতে পারে)
class WeakObserverStore {
    #observers = new Set();

    add(observer) {
        this.#observers.add(new WeakRef(observer));
    }

    notify(data) {
        for (const ref of this.#observers) {
            const observer = ref.deref();
            if (observer) {
                observer.update(data);
            } else {
                this.#observers.delete(ref); // মৃত reference পরিষ্কার
            }
        }
    }
}
```

### 7. Event Sourcing Connection

Observer প্যাটার্ন এবং Event Sourcing ঘনিষ্ঠভাবে সম্পর্কিত:

```
Event Sourcing:
    প্রতিটি state change একটি immutable event হিসেবে সংরক্ষিত হয়।
    বর্তমান state = সব events replay করলে যা পাওয়া যায়।

    OrderPlaced → OrderPaid → OrderShipped → OrderDelivered
         │              │            │               │
         ▼              ▼            ▼               ▼
    [Event Store — সব events চিরকাল সংরক্ষিত]

    Observer Pattern সেই events-এ react করে:
    - Projection: events থেকে read model তৈরি
    - Side effects: notification, email, analytics
    - Saga/Process Manager: long-running business process
```

```php
<?php

// Event Sourcing-এর সাথে Observer Pattern

readonly class DomainEvent
{
    public function __construct(
        public string $aggregateId,
        public string $type,
        public array $data,
        public int $version,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}
}

class EventStore
{
    private array $events = [];
    private array $projectors = [];

    public function append(DomainEvent $event): void
    {
        $this->events[] = $event;

        // সব projector-কে নোটিফাই (Observer Pattern!)
        foreach ($this->projectors as $projector) {
            $projector->handle($event);
        }
    }

    public function subscribe(object $projector): void
    {
        $this->projectors[] = $projector;
    }

    public function getEvents(string $aggregateId): array
    {
        return array_filter(
            $this->events,
            fn(DomainEvent $e) => $e->aggregateId === $aggregateId
        );
    }
}

// Projector — events থেকে read model তৈরি করে
class OrderSummaryProjector
{
    private array $summaries = [];

    public function handle(DomainEvent $event): void
    {
        match ($event->type) {
            'OrderPlaced' => $this->summaries[$event->aggregateId] = [
                'status' => 'placed',
                'total'  => $event->data['total'],
            ],
            'OrderShipped' => $this->summaries[$event->aggregateId]['status'] = 'shipped',
            default => null,
        };
    }

    public function getSummary(string $orderId): ?array
    {
        return $this->summaries[$orderId] ?? null;
    }
}
```

---

## ✅ Pros (সুবিধা)

| সুবিধা | বিবরণ |
|---------|--------|
| **Loose Coupling** | Subject এবং Observer স্বাধীনভাবে পরিবর্তন করা যায় |
| **Open/Closed Principle** | নতুন Observer যোগ করতে Subject পরিবর্তন লাগে না |
| **Runtime Flexibility** | রানটাইমে Observer যোগ/সরানো যায় |
| **Broadcast Communication** | একটি ইভেন্টে বহু receiver — কোড ডুপ্লিকেশন কমায় |
| **SRP Support** | প্রতিটি Observer একটি কাজ করে — Single Responsibility |
| **Reusability** | একই Observer বিভিন্ন Subject-এ ব্যবহারযোগ্য |

## ❌ Cons (অসুবিধা)

| অসুবিধা | বিবরণ |
|----------|--------|
| **Memory Leaks** | Unsubscribe না করলে Observer GC হয় না |
| **Unexpected Updates** | কোন ক্রমে Observer চলবে তার নিশ্চয়তা নেই |
| **Cascade Effects** | একটি Observer আরেকটি ইভেন্ট ট্রিগার করলে chain reaction হতে পারে |
| **Debugging Difficulty** | কোন Observer কখন, কেন চলেছে — ট্র্যাক করা কঠিন |
| **Performance** | বহু Observer থাকলে notification loop ধীর হতে পারে |
| **Complexity** | ছোট সিস্টেমে অতিরিক্ত complexity যোগ করতে পারে |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### 1. Cascade/Circular Updates

```javascript
// ❌ Observer A → ইভেন্ট ট্রিগার → Observer B → ইভেন্ট ট্রিগার → Observer A → ∞ LOOP!
class PriceService extends Subject {
    updatePrice(price) {
        this.price = price;
        this.notify('price:changed', price);
    }
}

class TaxCalculator {
    constructor(priceService) {
        priceService.subscribe('price:changed', (price) => {
            const withTax = price * 1.15;
            priceService.updatePrice(withTax); // ❌ INFINITE LOOP!
        });
    }
}

// ✅ সমাধান: আলাদা ইভেন্ট ব্যবহার করুন বা guard condition রাখুন
class FixedTaxCalculator {
    constructor(priceService) {
        priceService.subscribe('price:changed', (price) => {
            const withTax = price * 1.15;
            priceService.notify('price:withTax', withTax); // আলাদা ইভেন্ট
        });
    }
}
```

### 2. Observer-এ Heavy Operation

```php
<?php
// ❌ Observer-এ ভারী কাজ — পুরো সিস্টেম ব্লক হয়
class BadObserver implements \SplObserver
{
    public function update(\SplSubject $subject): void
    {
        // ❌ এখানে HTTP call, file I/O, heavy DB query করবেন না!
        $response = file_get_contents('https://api.external.com/notify');
        sleep(5); // পুরো notification chain ব্লক!
    }
}

// ✅ Queue/Async-তে পাঠান
class GoodObserver implements \SplObserver
{
    public function update(\SplSubject $subject): void
    {
        // Queue-তে dispatch করুন
        dispatch(new SendNotificationJob($subject->getState()));
    }
}
```

### 3. Order Dependency

```javascript
// ❌ Observer execution order-এ নির্ভর করা
emitter.on('save', validateData);   // এটা আগে চলবে?
emitter.on('save', transformData);  // নাকি এটা?
emitter.on('save', persistData);    // এটা শেষে চলতেই হবে!

// ✅ সমাধান: Priority system ব্যবহার করুন অথবা আলাদা ইভেন্ট চেইন করুন
emitter.on('save:validate', validateData);     // ১ম ধাপ
emitter.on('save:transform', transformData);   // ২য় ধাপ
emitter.on('save:persist', persistData);       // ৩য় ধাপ
```

---

## 🧪 টেস্টিং (Testing)

### PHPUnit — Observer Testing

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class OrderSubjectTest extends TestCase
{
    private OrderSubject $subject;

    protected function setUp(): void
    {
        $this->subject = new OrderSubject();
    }

    #[Test]
    public function observer_receives_notification_on_event(): void
    {
        $observer = $this->createMock(OrderObserverInterface::class);
        $observer->method('subscribedEvents')->willReturn(['order.placed']);

        $observer->expects($this->once())
            ->method('handle')
            ->with($this->callback(function (OrderEventInterface $event) {
                return $event->getEventName() === 'order.placed'
                    && $event->getOrderId() === 'ORD-001';
            }));

        $this->subject->attach($observer);
        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', []));
    }

    #[Test]
    public function detached_observer_does_not_receive_notification(): void
    {
        $observer = $this->createMock(OrderObserverInterface::class);
        $observer->method('subscribedEvents')->willReturn(['order.placed']);
        $observer->expects($this->never())->method('handle');

        $this->subject->attach($observer);
        $this->subject->detach($observer);
        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', []));
    }

    #[Test]
    public function multiple_observers_all_get_notified(): void
    {
        $observers = [];
        for ($i = 0; $i < 3; $i++) {
            $observer = $this->createMock(OrderObserverInterface::class);
            $observer->method('subscribedEvents')->willReturn(['order.placed']);
            $observer->expects($this->once())->method('handle');
            $observers[] = $observer;
            $this->subject->attach($observer);
        }

        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', []));
    }

    #[Test]
    public function observer_only_receives_subscribed_events(): void
    {
        $observer = $this->createMock(OrderObserverInterface::class);
        $observer->method('subscribedEvents')->willReturn(['order.shipped']);
        $observer->expects($this->never())->method('handle');

        $this->subject->attach($observer);
        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', []));
    }

    #[Test]
    public function failing_observer_does_not_break_others(): void
    {
        $failingObserver = $this->createMock(OrderObserverInterface::class);
        $failingObserver->method('subscribedEvents')->willReturn(['order.placed']);
        $failingObserver->method('handle')
            ->willThrowException(new \RuntimeException('DB Error'));

        $workingObserver = $this->createMock(OrderObserverInterface::class);
        $workingObserver->method('subscribedEvents')->willReturn(['order.placed']);
        $workingObserver->expects($this->once())->method('handle');

        $this->subject->attach($failingObserver);
        $this->subject->attach($workingObserver);
        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', []));
    }

    #[Test]
    public function event_carries_correct_payload(): void
    {
        $capturedEvent = null;
        $observer = new class($capturedEvent) implements OrderObserverInterface {
            public function __construct(private ?OrderEventInterface &$ref) {}
            public function subscribedEvents(): array { return ['order.placed']; }
            public function handle(OrderEventInterface $event): void {
                $this->ref = $event;
            }
        };

        $this->subject->attach($observer);
        $this->subject->dispatch(new OrderEvent('ORD-001', 'order.placed', [
            'total' => 5000.00,
            'items' => ['shirt'],
        ]));

        $this->assertNotNull($capturedEvent);
        $this->assertEquals(5000.00, $capturedEvent->getPayload()['total']);
        $this->assertCount(1, $capturedEvent->getPayload()['items']);
    }
}
```

### Jest — Observer Testing

```javascript
// Jest — Observer Pattern Testing

import { describe, test, expect, jest, beforeEach } from '@jest/globals';

class EventEmitter {
    #listeners = new Map();

    on(event, callback) {
        if (!this.#listeners.has(event)) this.#listeners.set(event, []);
        this.#listeners.get(event).push(callback);
        return () => this.off(event, callback);
    }

    off(event, callback) {
        const list = this.#listeners.get(event);
        if (list) {
            const idx = list.indexOf(callback);
            if (idx !== -1) list.splice(idx, 1);
        }
    }

    emit(event, ...args) {
        const list = this.#listeners.get(event) ?? [];
        list.forEach(cb => cb(...args));
    }

    listenerCount(event) {
        return this.#listeners.get(event)?.length ?? 0;
    }
}

describe('EventEmitter (Observer Pattern)', () => {
    let emitter;

    beforeEach(() => {
        emitter = new EventEmitter();
    });

    test('observer কে সঠিক ডেটা সহ নোটিফাই করে', () => {
        const handler = jest.fn();
        emitter.on('order:placed', handler);

        emitter.emit('order:placed', { id: 'ORD-1', total: 1500 });

        expect(handler).toHaveBeenCalledTimes(1);
        expect(handler).toHaveBeenCalledWith({ id: 'ORD-1', total: 1500 });
    });

    test('একাধিক observer সবাই নোটিফিকেশন পায়', () => {
        const handler1 = jest.fn();
        const handler2 = jest.fn();
        const handler3 = jest.fn();

        emitter.on('update', handler1);
        emitter.on('update', handler2);
        emitter.on('update', handler3);

        emitter.emit('update', 'test-data');

        expect(handler1).toHaveBeenCalledWith('test-data');
        expect(handler2).toHaveBeenCalledWith('test-data');
        expect(handler3).toHaveBeenCalledWith('test-data');
    });

    test('unsubscribe করলে আর নোটিফিকেশন পায় না', () => {
        const handler = jest.fn();
        const unsubscribe = emitter.on('event', handler);

        emitter.emit('event', 'first');
        unsubscribe();
        emitter.emit('event', 'second');

        expect(handler).toHaveBeenCalledTimes(1);
        expect(handler).toHaveBeenCalledWith('first');
    });

    test('ভুল ইভেন্টে subscribe করলে নোটিফিকেশন পায় না', () => {
        const handler = jest.fn();
        emitter.on('event-a', handler);

        emitter.emit('event-b', 'data');

        expect(handler).not.toHaveBeenCalled();
    });

    test('listener count সঠিকভাবে ট্র্যাক হয়', () => {
        const h1 = jest.fn();
        const h2 = jest.fn();

        emitter.on('test', h1);
        emitter.on('test', h2);

        expect(emitter.listenerCount('test')).toBe(2);

        emitter.off('test', h1);
        expect(emitter.listenerCount('test')).toBe(1);
    });

    test('observer-এ error হলে অন্যরা প্রভাবিত হয় না', () => {
        const errorHandler = jest.fn(() => { throw new Error('Boom!'); });
        const normalHandler = jest.fn();

        // Error-safe emit
        const safeEmit = (event, ...args) => {
            const list = [];
            emitter.on(event, (...a) => {
                try { errorHandler(...a); } catch (e) { /* log */ }
            });
            emitter.on(event, normalHandler);
            emitter.emit(event, ...args);
        };

        safeEmit('test', 'data');
        expect(normalHandler).toHaveBeenCalledWith('data');
    });

    test('কোনো subscriber না থাকলেও emit করা যায়', () => {
        expect(() => emitter.emit('nonexistent', 'data')).not.toThrow();
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন (Related Patterns)

### Observer ↔ Mediator
- **Mediator** অবজেক্টদের মধ্যে communication centralize করে — সবাই mediator-এর মাধ্যমে কথা বলে
- **Observer** এক-থেকে-বহু broadcast করে — Subject পরিবর্তন হলে সবাই জানে
- দুটো একসাথে ব্যবহার করা যায়: Mediator নিজে Observer Pattern ব্যবহার করতে পারে

### Observer ↔ Singleton
- Event Bus বা Event Dispatcher সাধারণত **Singleton** হিসেবে implement করা হয়
- Laravel-এর `Event` facade একটি Singleton Event Dispatcher

### Observer ↔ Command
- **Command** একটি request-কে অবজেক্ট হিসেবে encapsulate করে
- Observer notification-এ Command objects পাঠানো যায়
- Event Sourcing-এ events = commands-এর ফলাফল

### Observer ↔ Chain of Responsibility
- **Chain of Responsibility**: একটি request handler chain-এ যায়, একজন handle করে
- **Observer**: একটি event সব subscriber-এর কাছে যায়, সবাই handle করে
- একটি ইভেন্টে Chain of Responsibility ব্যবহার করে ফিল্টার করা যায়

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **একটি অবজেক্টের পরিবর্তনে অন্যরা আপডেট হতে হবে** — এবং কতজন আপডেট হবে তা আগে থেকে জানা নেই
2. **Loose coupling দরকার** — Subject জানবে না কে কী করবে
3. **Runtime-এ subscriber যোগ/সরানো দরকার** — dynamic relationship
4. **Event-driven architecture** — মাইক্রোসার্ভিস, real-time systems
5. **UI frameworks** — Model পরিবর্তনে View আপডেট (MVC/MVVM)
6. **Plugin system** — তৃতীয় পক্ষ hook করতে পারবে

### ❌ ব্যবহার করবেন না যখন:

1. **শুধু ২-৩ টি অবজেক্ট জড়িত** — সরাসরি method call যথেষ্ট, Observer overhead অপ্রয়োজনীয়
2. **Execution order গুরুত্বপূর্ণ** — Observer ক্রম guaranteed নয় (priority system ছাড়া)
3. **Synchronous response দরকার** — Observer fire-and-forget, return value নেই
4. **Simple one-time callback** — Promise বা callback যথেষ্ট
5. **Circular dependency সম্ভাবনা** — A → B → A cascade হতে পারে

---

## 📋 সারসংক্ষেপ (Summary)

```
┌─────────────────────────────────────────────────────────┐
│              Observer Pattern — সারসংক্ষেপ              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  📖 কী: One-to-Many dependency — state পরিবর্তনে       │
│         সব dependents স্বয়ংক্রিয়ভাবে নোটিফাই হয়     │
│                                                         │
│  🎯 উদ্দেশ্য: Loose coupling-এ event-based              │
│              communication নিশ্চিত করা                  │
│                                                         │
│  🏗️ মূল উপাদান:                                        │
│     • Subject — state ধারণ করে, observer পরিচালনা করে  │
│     • Observer — Subject-এর পরিবর্তনে react করে        │
│     • attach/detach — যোগ/সরানো                        │
│     • notify — সবাইকে জানানো                           │
│                                                         │
│  💡 মূল নীতি:                                           │
│     • Subject Observer-এর concrete type জানে না          │
│     • Observer যেকোনো সময় যোগ/সরানো যায়               │
│     • Unsubscribe করতে ভুলবেন না — মেমরি লিক!          │
│                                                         │
│  🌍 ব্যবহার:                                            │
│     • Laravel Events, Model Observers                   │
│     • Node.js EventEmitter                              │
│     • DOM addEventListener                              │
│     • WebSocket real-time                               │
│     • Message Queues (Pub/Sub)                          │
│     • Reactive Programming (RxJS, RxPHP)                │
│                                                         │
│  🇧🇩 বাংলাদেশ কনটেক্সট:                               │
│     • bKash ট্রানজ্যাকশন → SMS, Push, Log, Fraud       │
│     • Pathao রাইড → Rider, Passenger, Billing           │
│     • DSE Stock Ticker → Multiple Displays              │
│                                                         │
│  ⚠️ সতর্কতা:                                           │
│     • Memory leak (unsubscribe ভুলে গেলে)               │
│     • Cascade updates (infinite loop)                   │
│     • Order dependency (execution ক্রম অনিশ্চিত)       │
│     • Heavy observers (async/queue ব্যবহার করুন)        │
│                                                         │
│  🔗 সম্পর্কিত: Mediator, Command, Chain of             │
│     Responsibility, Singleton, Event Sourcing           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> **"Don't call us, we'll call you."** — Hollywood Principle
> Observer প্যাটার্নের মূলমন্ত্র: Subject যখন তৈরি, তখনই Observer-দের জানাবে। Observer-দের বারবার জিজ্ঞেস করতে হবে না।

---

*এই ডকুমেন্ট PHP 8.3 এবং JavaScript ES2022+ ব্যবহার করে তৈরি। সব কোড production-ready pattern অনুসরণ করে।*
