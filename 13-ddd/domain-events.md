# 📌 ডোমেইন ইভেন্ট (Domain Events) — DDD

> "Something happened that domain experts care about."
> ডোমেইনে এমন কিছু ঘটেছে যা ডোমেইন এক্সপার্টদের কাছে গুরুত্বপূর্ণ।

---

## 📖 সূচিপত্র

1. [ডোমেইন ইভেন্ট কী?](#-ডোমেইন-ইভেন্ট-কী)
2. [ইভেন্ট নেমিং কনভেনশন](#-ইভেন্ট-নেমিং-কনভেনশন)
3. [ইভেন্ট স্ট্রাকচার](#-ইভেন্ট-স্ট্রাকচার)
4. [ইভেন্ট ডিসপ্যাচিং](#-ইভেন্ট-ডিসপ্যাচিং)
5. [ইভেন্ট হ্যান্ডলার](#-ইভেন্ট-হ্যান্ডলার)
6. [ইভেন্ট সোর্সিং](#-ইভেন্ট-সোর্সিং)
7. [Integration Events vs Domain Events](#-integration-events-vs-domain-events)
8. [প্র্যাক্টিক্যাল উদাহরণ — Pathao রাইড সিস্টেম](#-প্র্যাক্টিক্যাল-উদাহরণ--pathao-রাইড-সিস্টেম)
9. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📖 ডোমেইন ইভেন্ট কী?

ডোমেইন ইভেন্ট হলো ডোমেইনে ঘটে যাওয়া একটি ঘটনা — **যা ইতিমধ্যে ঘটেছে** (past tense)।
এটি সিস্টেমের অন্য অংশকে জানায় যে কিছু একটা গুরুত্বপূর্ণ ঘটনা ঘটেছে।

### 🏠 বাস্তব উদাহরণ

bKash-এ টাকা পাঠানোর পর আপনি SMS পান — এটি একটি ডোমেইন ইভেন্টের প্রতিক্রিয়া:

```
আপনি bKash-এ ৫০০ টাকা পাঠালেন
        │
        ▼
  MoneyTransferred (ডোমেইন ইভেন্ট ঘটলো)
        │
        ├──▶ SMS পাঠানো হলো (Handler 1)
        ├──▶ Transaction History আপডেট হলো (Handler 2)
        └──▶ Balance আপডেট হলো (Handler 3)
```

### ইভেন্ট ফ্লো ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────┐
│                    EVENT FLOW                            │
│                                                         │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────┐  │
│  │  Action   │───▶│  Domain   │───▶│  Event Handlers  │  │
│  │ (Command) │    │  Event    │    │  (Side Effects)  │  │
│  └──────────┘    └───────────┘    └──────────────────┘  │
│                                                         │
│  "টাকা পাঠাও"   "MoneyTransferred"  "SMS, Log, etc."  │
│                                                         │
│  ⚡ কিছু করো     📢 কিছু ঘটেছে       👂 প্রতিক্রিয়া   │
└─────────────────────────────────────────────────────────┘
```

**মনে রাখুন:** Command = "কিছু করো" (ভবিষ্যৎ), Event = "কিছু ঘটেছে" (অতীত)

---

## 📊 ইভেন্ট নেমিং কনভেনশন

ডোমেইন ইভেন্টের নাম সবসময় **past tense** (অতীতকাল) এ হবে — কারণ ইভেন্ট বলছে "এটা ইতিমধ্যে ঘটে গেছে"।

### ❌ ভুল নাম (Bad — Imperative / Present Tense)

```
CreateOrder        ← এটা command, event না
SendPayment        ← ভবিষ্যতে কিছু করতে বলছে
UpdateInventory    ← এটা instruction
RegisterUser       ← "ব্যবহারকারী নিবন্ধন করো" — event না
```

### ✅ সঠিক নাম (Good — Past Tense)

```
OrderCreated       ← "অর্ডার তৈরি হয়েছে"
PaymentSent        ← "পেমেন্ট পাঠানো হয়েছে"
InventoryUpdated   ← "ইনভেন্টরি আপডেট হয়েছে"
UserRegistered     ← "ব্যবহারকারী নিবন্ধিত হয়েছে"
```

### Daraz-সদৃশ সিস্টেমের ইভেন্ট টেবিল

| Aggregate  | Event Name              | বাংলা অর্থ                        |
|------------|-------------------------|------------------------------------|
| Order      | `OrderPlaced`           | অর্ডার দেওয়া হয়েছে              |
| Order      | `OrderCancelled`        | অর্ডার বাতিল হয়েছে              |
| Order      | `OrderShipped`          | অর্ডার শিপ করা হয়েছে            |
| Order      | `OrderDelivered`        | অর্ডার ডেলিভারি হয়েছে           |
| Payment    | `PaymentReceived`       | পেমেন্ট গৃহীত হয়েছে             |
| Payment    | `PaymentRefunded`       | পেমেন্ট ফেরত দেওয়া হয়েছে       |
| Product    | `ProductAddedToCart`    | পণ্য কার্টে যোগ হয়েছে           |
| Product    | `ProductOutOfStock`     | পণ্য স্টকে নেই                    |
| User       | `UserRegistered`        | ব্যবহারকারী নিবন্ধিত হয়েছে     |
| Review     | `ReviewSubmitted`       | রিভিউ জমা দেওয়া হয়েছে          |
| Delivery   | `DeliveryAttemptFailed` | ডেলিভারি চেষ্টা ব্যর্থ হয়েছে   |

---

## 📊 ইভেন্ট স্ট্রাকচার

প্রতিটি ডোমেইন ইভেন্টে কী কী ডেটা থাকা উচিত:

```
┌────────────────────────────────────────┐
│          Domain Event Structure         │
├────────────────────────────────────────┤
│  eventId      : unique identifier      │
│  eventName    : "OrderPlaced"          │
│  occurredAt   : when it happened       │
│  aggregateId  : which aggregate        │
│  payload      : minimal relevant data  │
│  version      : event schema version   │
└────────────────────────────────────────┘
```

### PHP — Base Domain Event ক্লাস

```php
<?php

abstract class DomainEvent
{
    private string $eventId;
    private DateTimeImmutable $occurredAt;

    public function __construct()
    {
        $this->eventId = Uuid::uuid4()->toString();
        $this->occurredAt = new DateTimeImmutable();
    }

    public function eventId(): string
    {
        return $this->eventId;
    }

    public function occurredAt(): DateTimeImmutable
    {
        return $this->occurredAt;
    }

    abstract public function eventName(): string;
}
```

### PHP — Concrete Event (OrderPlaced)

```php
<?php

// ✅ Good: ন্যূনতম প্রয়োজনীয় ডেটা বহন করে
class OrderPlaced extends DomainEvent
{
    public function __construct(
        private string $orderId,
        private string $customerId,
        private float $totalAmount,
        private string $currency
    ) {
        parent::__construct();
    }

    public function eventName(): string
    {
        return 'order.placed';
    }

    public function orderId(): string { return $this->orderId; }
    public function customerId(): string { return $this->customerId; }
    public function totalAmount(): float { return $this->totalAmount; }
    public function currency(): string { return $this->currency; }
}
```

### ❌ Bad: ইভেন্টে পুরো Aggregate State বহন করা

```php
<?php

// ❌ Bad: সম্পূর্ণ order object ইভেন্টে পাঠাচ্ছে
class OrderPlacedBad extends DomainEvent
{
    public function __construct(
        private Order $order,          // পুরো aggregate!
        private array $orderItems,     // সব items!
        private Customer $customer,    // পুরো customer!
        private Address $address       // পুরো address!
    ) {
        parent::__construct();
    }
    // ... সমস্যা: tight coupling, বড় payload, privacy risk
}
```

### ✅ Good: ন্যূনতম ডেটা বহন করা

```php
<?php

// ✅ Good: শুধু যতটুকু দরকার ততটুকু
class OrderPlacedGood extends DomainEvent
{
    public function __construct(
        private string $orderId,
        private string $customerId,
        private float $totalAmount,
        private string $currency
    ) {
        parent::__construct();
    }
    // handler-এর আরো ডেটা লাগলে সে নিজে query করবে
}
```

### JavaScript — Domain Event ক্লাস

```javascript
// Base Domain Event
class DomainEvent {
    constructor(aggregateId, payload = {}) {
        this.eventId = crypto.randomUUID();
        this.occurredAt = new Date().toISOString();
        this.aggregateId = aggregateId;
        this.payload = payload;
    }

    get eventName() {
        return this.constructor.name;
    }
}

// Concrete Event
class OrderPlaced extends DomainEvent {
    constructor(orderId, customerId, totalAmount) {
        super(orderId, { customerId, totalAmount });
    }
}

// ব্যবহার
const event = new OrderPlaced('order-123', 'cust-456', 1500.00);
console.log(event.eventName);   // "OrderPlaced"
console.log(event.occurredAt);  // "2024-01-15T10:30:00.000Z"
```

---

## 🎯 ইভেন্ট ডিসপ্যাচিং

ইভেন্ট ডিসপ্যাচিং-এর সঠিক পদ্ধতি হলো: আগে Aggregate-এ ইভেন্ট জমা করা,
তারপর ডাটাবেসে সেভ করা, এবং **সেভ সফল হলে** ইভেন্ট dispatch করা।

### ডিসপ্যাচ ফ্লো ডায়াগ্রাম

```
┌───────────────────────────────────────────────────────────────────┐
│                   EVENT DISPATCH FLOW                              │
│                                                                   │
│  ┌───────────┐   ┌──────────┐   ┌─────────┐   ┌──────────────┐  │
│  │ Aggregate │──▶│ Collect  │──▶│ Save to │──▶│  Dispatch    │  │
│  │ (Domain   │   │ Events   │   │   DB    │   │  Events to   │  │
│  │  Logic)   │   │ internally│   │         │   │  Handlers    │  │
│  └───────────┘   └──────────┘   └─────────┘   └──────────────┘  │
│                                                                   │
│  ১. Order তৈরি  ২. OrderPlaced   ৩. DB তে    ৪. Handler দের    │
│     হলো           ইভেন্ট জমা       persist     জানানো হলো       │
│                    হলো              হলো                           │
│                                                                   │
│  ⚠️ গুরুত্বপূর্ণ: DB save ব্যর্থ হলে event dispatch হবে না!    │
└───────────────────────────────────────────────────────────────────┘
```

### PHP — Aggregate with Event Collection

```php
<?php

abstract class AggregateRoot
{
    private array $domainEvents = [];

    protected function recordEvent(DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}

class Order extends AggregateRoot
{
    private string $id;
    private string $status;

    public static function place(string $customerId, array $items, float $total): self
    {
        $order = new self();
        $order->id = Uuid::uuid4()->toString();
        $order->status = 'placed';

        // ইভেন্ট রেকর্ড করা হচ্ছে — এখনো dispatch হয়নি
        $order->recordEvent(new OrderPlaced(
            $order->id,
            $customerId,
            $total,
            'BDT'
        ));

        return $order;
    }
}
```

### PHP — Event Dispatcher

```php
<?php

interface EventDispatcher
{
    public function dispatch(DomainEvent $event): void;
}

class SimpleEventDispatcher implements EventDispatcher
{
    private array $handlers = [];

    public function subscribe(string $eventClass, callable $handler): void
    {
        $this->handlers[$eventClass][] = $handler;
    }

    public function dispatch(DomainEvent $event): void
    {
        $eventClass = get_class($event);
        foreach ($this->handlers[$eventClass] ?? [] as $handler) {
            $handler($event);
        }
    }
}

// Application Service — সব কিছু একসাথে কাজ করছে
class PlaceOrderService
{
    public function __construct(
        private OrderRepository $repository,
        private EventDispatcher $dispatcher
    ) {}

    public function execute(PlaceOrderCommand $command): void
    {
        // ১. Domain logic
        $order = Order::place(
            $command->customerId,
            $command->items,
            $command->totalAmount
        );

        // ২. Persist (save to DB)
        $this->repository->save($order);

        // ৩. Dispatch events (save সফল হওয়ার পরে)
        foreach ($order->pullDomainEvents() as $event) {
            $this->dispatcher->dispatch($event);
        }
    }
}
```

### JavaScript — EventEmitter-based Dispatch

```javascript
const EventEmitter = require('events');

class DomainEventBus extends EventEmitter {
    dispatch(event) {
        this.emit(event.eventName, event);
    }

    subscribe(eventName, handler) {
        this.on(eventName, handler);
    }
}

// Aggregate
class Order {
    #events = [];

    static place(customerId, items, total) {
        const order = new Order();
        order.id = crypto.randomUUID();
        order.status = 'placed';

        order.#events.push(new OrderPlaced(order.id, customerId, total));
        return order;
    }

    pullEvents() {
        const events = [...this.#events];
        this.#events = [];
        return events;
    }
}

// Application Service
class PlaceOrderService {
    constructor(orderRepo, eventBus) {
        this.orderRepo = orderRepo;
        this.eventBus = eventBus;
    }

    async execute(command) {
        const order = Order.place(command.customerId, command.items, command.total);

        await this.orderRepo.save(order);

        for (const event of order.pullEvents()) {
            this.eventBus.dispatch(event);
        }
    }
}
```

---

## 🔗 ইভেন্ট হ্যান্ডলার

একটি ডোমেইন ইভেন্ট থেকে **একাধিক হ্যান্ডলার** কাজ করতে পারে।
প্রতিটি হ্যান্ডলার একটি নির্দিষ্ট side effect সম্পন্ন করে।

### একটি ইভেন্ট → একাধিক হ্যান্ডলার

```
                    ┌──────────────────────────────────┐
                    │        OrderPlaced Event          │
                    │  (Daraz-এ অর্ডার দেওয়া হয়েছে)  │
                    └──────────────┬───────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
              ▼                    ▼                     ▼
   ┌──────────────────┐ ┌─────────────────┐ ┌────────────────────┐
   │ Handler 1:       │ │ Handler 2:      │ │ Handler 3:         │
   │ Send Email       │ │ Update          │ │ Notify             │
   │ Confirmation     │ │ Inventory       │ │ Warehouse          │
   │                  │ │                 │ │                    │
   │ "অর্ডার কনফার্ম │ │ "স্টক কমাও"    │ │ "গুদামকে জানাও"   │
   │  ইমেইল পাঠাও"   │ │                 │ │                    │
   └──────────────────┘ └─────────────────┘ └────────────────────┘
```

### PHP — Event Handlers

```php
<?php

// Handler 1: কনফার্মেশন ইমেইল পাঠানো
class SendOrderConfirmationEmail
{
    public function __construct(private Mailer $mailer) {}

    public function handle(OrderPlaced $event): void
    {
        $this->mailer->send(
            to: $event->customerId(),
            subject: 'অর্ডার কনফার্ম হয়েছে!',
            body: "আপনার অর্ডার #{$event->orderId()} গৃহীত হয়েছে।"
        );
    }
}

// Handler 2: ইনভেন্টরি আপডেট
class UpdateInventoryOnOrderPlaced
{
    public function __construct(private InventoryService $inventory) {}

    public function handle(OrderPlaced $event): void
    {
        $this->inventory->decreaseStock($event->orderId());
    }
}

// Handler 3: ওয়্যারহাউস নোটিফিকেশন
class NotifyWarehouse
{
    public function __construct(private WarehouseGateway $warehouse) {}

    public function handle(OrderPlaced $event): void
    {
        $this->warehouse->prepareForShipping($event->orderId());
    }
}

// হ্যান্ডলার রেজিস্ট্রেশন
$dispatcher = new SimpleEventDispatcher();
$dispatcher->subscribe(OrderPlaced::class, [new SendOrderConfirmationEmail($mailer), 'handle']);
$dispatcher->subscribe(OrderPlaced::class, [new UpdateInventoryOnOrderPlaced($inventory), 'handle']);
$dispatcher->subscribe(OrderPlaced::class, [new NotifyWarehouse($warehouse), 'handle']);
```

### JavaScript — Event Handlers

```javascript
// Handler 1: কনফার্মেশন ইমেইল
function sendOrderConfirmation(event) {
    console.log(`ইমেইল পাঠানো হচ্ছে — Order #${event.aggregateId}`);
    mailer.send({
        to: event.payload.customerId,
        subject: 'অর্ডার কনফার্ম হয়েছে!',
        body: `আপনার অর্ডার #${event.aggregateId} গৃহীত হয়েছে।`
    });
}

// Handler 2: ইনভেন্টরি আপডেট
function updateInventory(event) {
    inventoryService.decreaseStock(event.aggregateId);
}

// Handler 3: ওয়্যারহাউস নোটিফিকেশন
function notifyWarehouse(event) {
    warehouseGateway.prepareShipping(event.aggregateId);
}

// সব হ্যান্ডলার রেজিস্টার
const eventBus = new DomainEventBus();
eventBus.subscribe('OrderPlaced', sendOrderConfirmation);
eventBus.subscribe('OrderPlaced', updateInventory);
eventBus.subscribe('OrderPlaced', notifyWarehouse);

// ইভেন্ট dispatch হলে তিনটি হ্যান্ডলারই কাজ করবে
```

---

## 📊 ইভেন্ট সোর্সিং (Event Sourcing)

ইভেন্ট সোর্সিং-এ বর্তমান state সরাসরি সংরক্ষণ না করে,
**সমস্ত ঘটনার ইতিহাস** সংরক্ষণ করা হয় এবং সেগুলো replay করে বর্তমান state তৈরি করা হয়।

### Traditional DB vs Event Sourcing

```
┌──────────────────────────────────┐  ┌──────────────────────────────────────┐
│      TRADITIONAL (CRUD)          │  │        EVENT SOURCING                │
├──────────────────────────────────┤  ├──────────────────────────────────────┤
│                                  │  │                                      │
│  accounts table:                 │  │  events table:                       │
│  ┌────────┬─────────┐           │  │  ┌────┬──────────────────┬─────────┐│
│  │ id     │ balance  │           │  │  │ #  │ event            │ amount  ││
│  ├────────┼─────────┤           │  │  ├────┼──────────────────┼─────────┤│
│  │ acc-1  │ 3000    │           │  │  │ 1  │ AccountOpened    │ 0       ││
│  └────────┴─────────┘           │  │  │ 2  │ MoneyDeposited   │ +5000   ││
│                                  │  │  │ 3  │ MoneyWithdrawn   │ -2000   ││
│  শুধু বর্তমান state দেখা যায়   │  │  │ 4  │ MoneyDeposited   │ +1000   ││
│  ইতিহাস হারিয়ে গেছে!          │  │  │ 5  │ MoneyWithdrawn   │ -1000   ││
│                                  │  │  └────┴──────────────────┴─────────┘│
│                                  │  │                                      │
│                                  │  │  Replay: 0+5000-2000+1000-1000      │
│                                  │  │  Balance = ৩০০০ টাকা ✅             │
│                                  │  │                                      │
│                                  │  │  সম্পূর্ণ ইতিহাস সংরক্ষিত!        │
└──────────────────────────────────┘  └──────────────────────────────────────┘
```

### PHP — Event Sourcing উদাহরণ (bKash অ্যাকাউন্ট)

```php
<?php

// ইভেন্টগুলো
class AccountOpened extends DomainEvent
{
    public function __construct(private string $accountId, private string $ownerName) {
        parent::__construct();
    }
    public function eventName(): string { return 'account.opened'; }
    public function accountId(): string { return $this->accountId; }
}

class MoneyDeposited extends DomainEvent
{
    public function __construct(private string $accountId, private float $amount) {
        parent::__construct();
    }
    public function eventName(): string { return 'money.deposited'; }
    public function accountId(): string { return $this->accountId; }
    public function amount(): float { return $this->amount; }
}

class MoneyWithdrawn extends DomainEvent
{
    public function __construct(private string $accountId, private float $amount) {
        parent::__construct();
    }
    public function eventName(): string { return 'money.withdrawn'; }
    public function accountId(): string { return $this->accountId; }
    public function amount(): float { return $this->amount; }
}

// Event Sourced Aggregate
class BkashAccount
{
    private string $id;
    private float $balance = 0;
    private array $uncommittedEvents = [];

    // ইভেন্ট replay করে state পুনরুদ্ধার
    public static function reconstitute(array $events): self
    {
        $account = new self();
        foreach ($events as $event) {
            $account->apply($event);
        }
        return $account;
    }

    public static function open(string $ownerName): self
    {
        $account = new self();
        $account->recordAndApply(new AccountOpened(
            Uuid::uuid4()->toString(),
            $ownerName
        ));
        return $account;
    }

    public function deposit(float $amount): void
    {
        if ($amount <= 0) {
            throw new InvalidArgumentException('পরিমাণ ধনাত্মক হতে হবে');
        }
        $this->recordAndApply(new MoneyDeposited($this->id, $amount));
    }

    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new DomainException('অপর্যাপ্ত ব্যালেন্স!');
        }
        $this->recordAndApply(new MoneyWithdrawn($this->id, $amount));
    }

    private function apply(DomainEvent $event): void
    {
        match (true) {
            $event instanceof AccountOpened => $this->id = $event->accountId(),
            $event instanceof MoneyDeposited => $this->balance += $event->amount(),
            $event instanceof MoneyWithdrawn => $this->balance -= $event->amount(),
        };
    }

    private function recordAndApply(DomainEvent $event): void
    {
        $this->apply($event);
        $this->uncommittedEvents[] = $event;
    }

    public function balance(): float { return $this->balance; }

    public function pullUncommittedEvents(): array
    {
        $events = $this->uncommittedEvents;
        $this->uncommittedEvents = [];
        return $events;
    }
}

// ব্যবহার
$account = BkashAccount::open('রহিম');
$account->deposit(5000);   // +৫০০০
$account->withdraw(2000);  // -২০০০
echo $account->balance();  // ৩০০০
```

### JavaScript — Event Sourcing উদাহরণ

```javascript
class BkashAccount {
    #id;
    #balance = 0;
    #events = [];

    static open(ownerName) {
        const account = new BkashAccount();
        account.#recordAndApply({
            type: 'AccountOpened',
            accountId: crypto.randomUUID(),
            ownerName
        });
        return account;
    }

    // ইভেন্ট থেকে state পুনরুদ্ধার
    static reconstitute(events) {
        const account = new BkashAccount();
        events.forEach(e => account.#apply(e));
        return account;
    }

    deposit(amount) {
        if (amount <= 0) throw new Error('পরিমাণ ধনাত্মক হতে হবে');
        this.#recordAndApply({ type: 'MoneyDeposited', accountId: this.#id, amount });
    }

    withdraw(amount) {
        if (amount > this.#balance) throw new Error('অপর্যাপ্ত ব্যালেন্স!');
        this.#recordAndApply({ type: 'MoneyWithdrawn', accountId: this.#id, amount });
    }

    #apply(event) {
        switch (event.type) {
            case 'AccountOpened': this.#id = event.accountId; break;
            case 'MoneyDeposited': this.#balance += event.amount; break;
            case 'MoneyWithdrawn': this.#balance -= event.amount; break;
        }
    }

    #recordAndApply(event) {
        this.#apply(event);
        this.#events.push({ ...event, occurredAt: new Date().toISOString() });
    }

    get balance() { return this.#balance; }

    pullEvents() {
        const events = [...this.#events];
        this.#events = [];
        return events;
    }
}

// ব্যবহার
const account = BkashAccount.open('করিম');
account.deposit(5000);
account.withdraw(2000);
console.log(account.balance); // 3000
```

### ইভেন্ট সোর্সিং — সুবিধা ও অসুবিধা

| সুবিধা ✅                              | অসুবিধা ❌                             |
|-----------------------------------------|-----------------------------------------|
| সম্পূর্ণ audit trail পাওয়া যায়        | complexity বাড়ে                        |
| যেকোনো সময়ের state দেখা সম্ভব         | eventual consistency বুঝতে হবে         |
| debugging সহজ হয়                       | storage বেশি লাগতে পারে                |
| temporal queries সম্ভব                  | event versioning চ্যালেঞ্জিং          |
| bKash-এর মতো transaction history        | simple CRUD-এ overkill                 |

### কখন Event Sourcing ব্যবহার করবেন?

bKash/Nagad-এর transaction history চিন্তা করুন — প্রতিটি লেনদেনের বিস্তারিত ইতিহাস রাখা দরকার। এটি event sourcing-এর উত্তম ব্যবহার।

---

## 🔗 Integration Events vs Domain Events

### পার্থক্য ডায়াগ্রাম

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌─────────────── Bounded Context: Order ──────────────────┐        │
│  │                                                          │        │
│  │  Order Aggregate                                         │        │
│  │       │                                                  │        │
│  │       ▼                                                  │        │
│  │  OrderPlaced (Domain Event)  ◄── অভ্যন্তরীণ ইভেন্ট     │        │
│  │       │                                                  │        │
│  │       ├──▶ SendConfirmationEmail (Handler)               │        │
│  │       │                                                  │        │
│  │       └──▶ PublishIntegrationEvent (Handler)             │        │
│  │                │                                         │        │
│  └────────────────┼─────────────────────────────────────────┘        │
│                   │                                                  │
│                   ▼                                                  │
│     OrderPlacedIntegrationEvent  ◄── বাইরের ইভেন্ট                 │
│          (Message Bus / Queue)                                       │
│                   │                                                  │
│       ┌───────────┴───────────┐                                      │
│       ▼                       ▼                                      │
│  ┌──────────┐          ┌────────────┐                                │
│  │Inventory │          │ Shipping   │                                │
│  │ Service  │          │ Service    │                                │
│  └──────────┘          └────────────┘                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### তুলনা টেবিল

| বৈশিষ্ট্য          | Domain Event              | Integration Event             |
|---------------------|---------------------------|-------------------------------|
| স্কোপ              | একটি Bounded Context-এর মধ্যে | Bounded Context-এর বাইরে    |
| পরিবহন             | In-process (মেমোরিতে)     | Message Bus (RabbitMQ, Kafka) |
| Consistency         | Immediate                 | Eventual                      |
| ব্যর্থতা হ্যান্ডলিং| Exception                 | Retry, Dead Letter Queue      |
| Coupling            | অভ্যন্তরীণ               | ন্যূনতম (loose)              |
| উদাহরণ             | OrderPlaced               | OrderPlacedIntegrationEvent   |

### PHP — Domain Event ও Integration Event

```php
<?php

// Domain Event — অভ্যন্তরীণ (Internal)
class OrderPlaced extends DomainEvent
{
    public function __construct(
        private string $orderId,
        private string $customerId,
        private float $totalAmount,
        private string $currency
    ) {
        parent::__construct();
    }

    public function eventName(): string { return 'order.placed'; }
    public function orderId(): string { return $this->orderId; }
    public function customerId(): string { return $this->customerId; }
    public function totalAmount(): float { return $this->totalAmount; }
}

// Integration Event — বাইরের সার্ভিসের জন্য
class OrderPlacedIntegrationEvent
{
    public function __construct(
        private string $orderId,
        private float $totalAmount,
        private string $currency,
        private DateTimeImmutable $occurredAt
    ) {}

    public function toArray(): array
    {
        return [
            'event_type' => 'order_service.order.placed',
            'order_id'   => $this->orderId,
            'total'      => $this->totalAmount,
            'currency'   => $this->currency,
            'occurred_at'=> $this->occurredAt->format('c'),
        ];
    }
}

// Domain Event Handler যে Integration Event publish করে
class PublishOrderPlacedIntegration
{
    public function __construct(private MessageBus $bus) {}

    public function handle(OrderPlaced $domainEvent): void
    {
        $integrationEvent = new OrderPlacedIntegrationEvent(
            $domainEvent->orderId(),
            $domainEvent->totalAmount(),
            'BDT',
            $domainEvent->occurredAt()
        );

        // RabbitMQ / Kafka-তে পাঠানো হচ্ছে
        $this->bus->publish('orders', $integrationEvent->toArray());
    }
}
```

### JavaScript — Domain Event ও Integration Event

```javascript
// Domain Event (অভ্যন্তরীণ)
class OrderPlaced extends DomainEvent {
    constructor(orderId, customerId, totalAmount) {
        super(orderId, { customerId, totalAmount });
    }
}

// Integration Event publisher (বাইরের সার্ভিসে পাঠানো)
class PublishOrderPlacedIntegration {
    constructor(messageBus) {
        this.messageBus = messageBus;
    }

    handle(domainEvent) {
        const integrationEvent = {
            eventType: 'order_service.order.placed',
            orderId: domainEvent.aggregateId,
            totalAmount: domainEvent.payload.totalAmount,
            occurredAt: domainEvent.occurredAt
        };

        // RabbitMQ / Kafka-তে publish
        this.messageBus.publish('orders', integrationEvent);
    }
}

// ব্যবহার
eventBus.subscribe('OrderPlaced', (event) => {
    // Domain handler — অভ্যন্তরীণ
    sendConfirmationEmail(event);
});

eventBus.subscribe('OrderPlaced', (event) => {
    // Integration — বাইরের সার্ভিসে
    new PublishOrderPlacedIntegration(kafkaBus).handle(event);
});
```

---

## 🎯 প্র্যাক্টিক্যাল উদাহরণ — Pathao রাইড সিস্টেম

একটি সম্পূর্ণ Pathao-সদৃশ রাইড সিস্টেমের ইভেন্ট ফ্লো:

### সম্পূর্ণ ইভেন্ট ফ্লো ডায়াগ্রাম

```
 যাত্রী রাইড           ড্রাইভার              রাইড              পেমেন্ট
 রিকোয়েস্ট করলো       অ্যাসাইন হলো          শুরু হলো          প্রসেস হলো
      │                    │                    │                   │
      ▼                    ▼                    ▼                   ▼
 ┌──────────┐      ┌──────────────┐     ┌────────────┐     ┌──────────────┐
 │  Ride     │      │   Driver     │     │   Ride     │     │   Payment    │
 │ Requested │─────▶│  Assigned    │────▶│  Started   │────▶│  Processed   │
 └──────────┘      └──────────────┘     └────────────┘     └──────────────┘
      │                    │                    │                   │
      ▼                    ▼                    ▼                   ▼
 ┌──────────┐      ┌──────────────┐     ┌────────────┐     ┌──────────────┐
 │• ড্রাইভার│      │• যাত্রীকে   │     │• GPS       │     │• ড্রাইভারকে │
 │  খোঁজ    │      │  SMS পাঠাও  │     │  tracking  │     │  টাকা দাও   │
 │  শুরু    │      │• ETA দেখাও  │     │  শুরু করো  │     │• রিসিপ্ট    │
 │• Push    │      │              │     │            │     │  পাঠাও      │
 │  notify  │      │              │     │            │     │• Rating চাও │
 └──────────┘      └──────────────┘     └────────────┘     └──────────────┘
```

### Ride Completed ইভেন্ট — সম্পূর্ণ ফ্লো

```
                    ┌──────────────────────┐
                    │   RideCompleted      │
                    │   (ডোমেইন ইভেন্ট)   │
                    └──────────┬───────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                      │
         ▼                     ▼                      ▼
  ┌──────────────┐   ┌────────────────┐   ┌────────────────────┐
  │ Calculate    │   │ Update Driver  │   │  Request Rating    │
  │ Fare &      │   │ Availability   │   │  from Passenger    │
  │ Process     │   │ (available     │   │  (Push             │
  │ Payment     │   │  again)        │   │   notification)    │
  └──────────────┘   └────────────────┘   └────────────────────┘
```

### PHP — Pathao রাইড সিস্টেমের ইভেন্ট ও হ্যান্ডলার

```php
<?php

// ── Events ──────────────────────────────────────────
class RideRequested extends DomainEvent
{
    public function __construct(
        private string $rideId,
        private string $passengerId,
        private string $pickupLocation,
        private string $dropoffLocation
    ) {
        parent::__construct();
    }
    public function eventName(): string { return 'ride.requested'; }
    public function rideId(): string { return $this->rideId; }
    public function passengerId(): string { return $this->passengerId; }
}

class DriverAssigned extends DomainEvent
{
    public function __construct(
        private string $rideId,
        private string $driverId,
        private int $estimatedMinutes
    ) {
        parent::__construct();
    }
    public function eventName(): string { return 'ride.driver_assigned'; }
    public function rideId(): string { return $this->rideId; }
    public function driverId(): string { return $this->driverId; }
}

class RideCompleted extends DomainEvent
{
    public function __construct(
        private string $rideId,
        private string $passengerId,
        private string $driverId,
        private float $fareAmount,
        private float $distanceKm
    ) {
        parent::__construct();
    }
    public function eventName(): string { return 'ride.completed'; }
    public function rideId(): string { return $this->rideId; }
    public function fareAmount(): float { return $this->fareAmount; }
    public function driverId(): string { return $this->driverId; }
    public function passengerId(): string { return $this->passengerId; }
}

// ── Aggregate ───────────────────────────────────────
class Ride extends AggregateRoot
{
    private string $id;
    private string $status;

    public static function request(
        string $passengerId,
        string $pickup,
        string $dropoff
    ): self {
        $ride = new self();
        $ride->id = Uuid::uuid4()->toString();
        $ride->status = 'requested';

        $ride->recordEvent(new RideRequested(
            $ride->id, $passengerId, $pickup, $dropoff
        ));
        return $ride;
    }

    public function assignDriver(string $driverId, int $eta): void
    {
        $this->status = 'assigned';
        $this->recordEvent(new DriverAssigned($this->id, $driverId, $eta));
    }

    public function complete(string $passengerId, string $driverId, float $fare, float $km): void
    {
        $this->status = 'completed';
        $this->recordEvent(new RideCompleted(
            $this->id, $passengerId, $driverId, $fare, $km
        ));
    }
}

// ── Handlers ────────────────────────────────────────
class FindNearbyDrivers
{
    public function handle(RideRequested $event): void
    {
        // কাছাকাছি ড্রাইভার খুঁজে assign করো
        echo "রাইড #{$event->rideId()} — কাছের ড্রাইভার খোঁজা হচ্ছে...\n";
    }
}

class NotifyPassengerOfDriver
{
    public function handle(DriverAssigned $event): void
    {
        echo "রাইড #{$event->rideId()} — যাত্রীকে জানানো হচ্ছে: ড্রাইভার আসছে!\n";
    }
}

class ProcessPaymentOnCompletion
{
    public function handle(RideCompleted $event): void
    {
        echo "রাইড #{$event->rideId()} — ভাড়া ৳{$event->fareAmount()} প্রসেস হচ্ছে\n";
    }
}

class RequestRatingOnCompletion
{
    public function handle(RideCompleted $event): void
    {
        echo "রাইড #{$event->rideId()} — যাত্রীকে রেটিং দিতে বলা হচ্ছে\n";
    }
}

// ── Wire Everything ─────────────────────────────────
$dispatcher = new SimpleEventDispatcher();
$dispatcher->subscribe(RideRequested::class, [new FindNearbyDrivers(), 'handle']);
$dispatcher->subscribe(DriverAssigned::class, [new NotifyPassengerOfDriver(), 'handle']);
$dispatcher->subscribe(RideCompleted::class, [new ProcessPaymentOnCompletion(), 'handle']);
$dispatcher->subscribe(RideCompleted::class, [new RequestRatingOnCompletion(), 'handle']);
```

### JavaScript — Pathao রাইড সিস্টেম

```javascript
// Events
class RideRequested extends DomainEvent {
    constructor(rideId, passengerId, pickup, dropoff) {
        super(rideId, { passengerId, pickup, dropoff });
    }
}

class DriverAssigned extends DomainEvent {
    constructor(rideId, driverId, eta) {
        super(rideId, { driverId, eta });
    }
}

class RideCompleted extends DomainEvent {
    constructor(rideId, passengerId, driverId, fare) {
        super(rideId, { passengerId, driverId, fare });
    }
}

// Aggregate
class Ride {
    #events = [];

    static request(passengerId, pickup, dropoff) {
        const ride = new Ride();
        ride.id = crypto.randomUUID();
        ride.status = 'requested';
        ride.#events.push(new RideRequested(ride.id, passengerId, pickup, dropoff));
        return ride;
    }

    assignDriver(driverId, eta) {
        this.status = 'assigned';
        this.#events.push(new DriverAssigned(this.id, driverId, eta));
    }

    complete(passengerId, driverId, fare) {
        this.status = 'completed';
        this.#events.push(new RideCompleted(this.id, passengerId, driverId, fare));
    }

    pullEvents() {
        const events = [...this.#events];
        this.#events = [];
        return events;
    }
}

// Handlers
const eventBus = new DomainEventBus();

eventBus.subscribe('RideRequested', (event) => {
    console.log(`🔍 রাইড #${event.aggregateId} — কাছের ড্রাইভার খোঁজা হচ্ছে...`);
});

eventBus.subscribe('DriverAssigned', (event) => {
    console.log(`📱 রাইড #${event.aggregateId} — যাত্রীকে জানানো: ড্রাইভার ${event.payload.eta} মিনিটে আসবে`);
});

eventBus.subscribe('RideCompleted', (event) => {
    console.log(`💰 রাইড #${event.aggregateId} — ভাড়া ৳${event.payload.fare} প্রসেস হচ্ছে`);
});

eventBus.subscribe('RideCompleted', (event) => {
    console.log(`⭐ রাইড #${event.aggregateId} — যাত্রীকে রেটিং দিতে অনুরোধ`);
});

// সম্পূর্ণ ফ্লো চালানো
const ride = Ride.request('passenger-1', 'ধানমন্ডি', 'গুলশান');
for (const e of ride.pullEvents()) eventBus.dispatch(e);

ride.assignDriver('driver-42', 5);
for (const e of ride.pullEvents()) eventBus.dispatch(e);

ride.complete('passenger-1', 'driver-42', 250);
for (const e of ride.pullEvents()) eventBus.dispatch(e);
```

---

## ✅❌ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Domain Events ব্যবহার করবেন

```
✅ যখন একটি action-এর পর একাধিক side effect দরকার
   উদাহরণ: অর্ডার হলে → ইমেইল + ইনভেন্টরি + নোটিফিকেশন

✅ যখন Bounded Context-এর মধ্যে loosely coupled যোগাযোগ দরকার
   উদাহরণ: Order context → Notification context

✅ যখন audit trail / ইতিহাস রাখা দরকার
   উদাহরণ: bKash transaction log

✅ যখন async processing দরকার
   উদাহরণ: অর্ডার confirm → background-এ ইনভয়েস তৈরি

✅ যখন আপনি চান aggregate একে অপরকে সরাসরি call না করুক
   উদাহরণ: Order aggregate সরাসরি Inventory aggregate call করবে না
```

### ❌ কখন Domain Events ব্যবহার করবেন না

```
❌ সাধারণ CRUD অপারেশনে যেখানে কোনো side effect নেই
   উদাহরণ: ব্যবহারকারীর প্রোফাইল ফটো আপডেট

❌ যখন synchronous response জরুরি
   উদাহরণ: পেমেন্ট গেটওয়ে validation — তৎক্ষণাৎ ফলাফল দরকার

❌ ছোট প্রজেক্টে যেখানে complexity যোগ করার দরকার নেই
   উদাহরণ: সাধারণ ব্লগ অ্যাপ্লিকেশন

❌ যখন একটি মাত্র handler থাকবে এবং সেটি synchronous
   সরাসরি মেথড call-ই যথেষ্ট
```

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট

```
              একটি action-এর পর কী হবে?
                       │
            ┌──────────┴──────────┐
            │                     │
       একটি মাত্র              একাধিক
       কাজ হবে                কাজ হবে
            │                     │
            ▼                     ▼
    সরাসরি method          Domain Event
    call করুন ✅            ব্যবহার করুন ✅
                                  │
                          ┌───────┴───────┐
                          │               │
                     একই context      অন্য context
                          │               │
                          ▼               ▼
                    Domain Event    Integration Event
                                   (Message Bus দিয়ে)
```

---

## 📌 সারসংক্ষেপ (Quick Reference)

```
┌────────────────────────────────────────────────────────────────┐
│                  DOMAIN EVENTS CHEAT SHEET                     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  নামকরণ:     সবসময় past tense → OrderPlaced, PaymentSent    │
│  ডেটা:       ন্যূনতম প্রয়োজনীয় → ID, timestamp, payload    │
│  Dispatch:   DB save-এর পরে → Aggregate → Save → Dispatch    │
│  Handler:    একটি ইভেন্ট → একাধিক handler হতে পারে           │
│  Sourcing:   ইভেন্ট সংরক্ষণ করে state পুনরুদ্ধার            │
│  Domain:     অভ্যন্তরীণ (in-process)                         │
│  Integration: বাহ্যিক (message bus / queue)                   │
│                                                                │
│  মনে রাখুন:                                                   │
│  ─────────                                                     │
│  Command = "কিছু করো"  (TransferMoney)                        │
│  Event   = "কিছু ঘটেছে" (MoneyTransferred)                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

> **পরবর্তী টপিক:** CQRS (Command Query Responsibility Segregation) — ডোমেইন ইভেন্ট এবং CQRS একসাথে দারুণ কাজ করে!
