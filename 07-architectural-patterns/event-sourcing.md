# 🎬 Event Sourcing Pattern

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Traditional CRUD vs Event Sourcing](#traditional-crud-vs-event-sourcing)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [CQRS Integration](#cqrs-integration)
- [Snapshots এবং Performance](#snapshots-এবং-performance)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Event Sourcing** হলো এমন একটি আর্কিটেকচারাল প্যাটার্ন যেখানে অ্যাপ্লিকেশনের **state** সরাসরি সংরক্ষণ করা হয় না, বরং state-এ যত পরিবর্তন (events) হয়েছে তার **সম্পূর্ণ ইতিহাস** সংরক্ষণ করা হয়।

### 🔑 মূল ধারণাসমূহ:
- **Event**: অতীতে ঘটে যাওয়া একটি ঘটনা (past tense: OrderPlaced, PaymentReceived)
- **Event Store**: সব events সংরক্ষণের স্থান (append-only log)
- **Projection/Read Model**: events থেকে তৈরি বর্তমান state
- **Snapshot**: performance-এর জন্য নির্দিষ্ট সময়ে state-এর ছবি
- **Replay**: পুরানো events আবার চালিয়ে state পুনর্গঠন

> 💡 মনে রাখুন: Event Sourcing-এ আমরা "কী হয়েছিল" সেটা সংরক্ষণ করি, "বর্তমান অবস্থা কী" সেটা নয়।

---

## 🔄 Traditional CRUD vs Event Sourcing

### Traditional CRUD:
```
অর্ডার টেবিল: { id: 1, status: "delivered", total: 500 }
→ শুধু বর্তমান অবস্থা জানা যায়
→ কিভাবে এই অবস্থায় এসেছে, জানা যায় না
```

### Event Sourcing:
```
Event 1: OrderPlaced { orderId: 1, items: [...], total: 450 }
Event 2: ItemAdded { orderId: 1, item: "চা", price: 50 }
Event 3: PaymentReceived { orderId: 1, amount: 500, method: "bKash" }
Event 4: OrderShipped { orderId: 1, courier: "Pathao" }
Event 5: OrderDelivered { orderId: 1, timestamp: "2024-01-15" }
→ সম্পূর্ণ ইতিহাস জানা যায়!
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏦 bKash Transaction System
বিকাশের প্রতিটি লেনদেন event হিসেবে চিন্তা করুন:

```
Event 1: AccountCreated { phone: "01711XXXXXX", name: "রহিম" }
Event 2: MoneyReceived { amount: 5000, from: "salary" }
Event 3: SendMoney { amount: 1000, to: "01812XXXXXX" }
Event 4: PayBill { amount: 800, biller: "DESCO" }
Event 5: CashOut { amount: 2000, agent: "01611XXXXXX" }

→ বর্তমান Balance: 5000 - 1000 - 800 - 2000 = 1200 টাকা
→ সম্পূর্ণ লেনদেনের ইতিহাস সংরক্ষিত!
```

### 📦 Pathao Delivery Tracking
```
Event 1: OrderReceived { orderId: "P001", restaurant: "সুলতানস ডাইন" }
Event 2: RiderAssigned { riderId: "R45", name: "করিম" }
Event 3: PickedUp { timestamp: "12:30 PM" }
Event 4: LocationUpdated { lat: 23.8103, lng: 90.4125 }
Event 5: Delivered { timestamp: "1:15 PM", rating: 5 }
```

### 📰 Prothom Alo Content Management
```
Event 1: ArticleCreated { id: "A1", title: "বাজেট ২০২৪" }
Event 2: ArticleEdited { changes: { title: "বাজেট ২০২৪-২৫" } }
Event 3: ArticlePublished { publishedAt: "2024-06-01" }
Event 4: ArticleUpdated { correction: "তথ্য সংশোধন" }
→ প্রতিটি সংশোধনের ইতিহাস রয়েছে (Audit Trail)
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Event Sourcing মূল কাঠামো:
```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING SYSTEM                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    Command     ┌──────────────────┐           │
│  │  Client  │ ──────────────▶│  Command Handler │           │
│  └──────────┘                └────────┬─────────┘           │
│                                       │                      │
│                                       ▼                      │
│                              ┌─────────────────┐            │
│                              │  Domain Logic   │            │
│                              │  (Aggregate)    │            │
│                              └────────┬────────┘            │
│                                       │                      │
│                                  New Events                  │
│                                       │                      │
│                                       ▼                      │
│  ┌────────────────────────────────────────────────────┐     │
│  │              EVENT STORE (Append-Only)              │     │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        │     │
│  │  │ E1  │ │ E2  │ │ E3  │ │ E4  │ │ E5  │ ...    │     │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘        │     │
│  └────────────────────────┬───────────────────────────┘     │
│                           │                                  │
│                     Event Published                          │
│                           │                                  │
│              ┌────────────┼────────────┐                    │
│              ▼            ▼            ▼                     │
│  ┌───────────────┐ ┌──────────┐ ┌──────────────┐          │
│  │  Projection 1 │ │Projection│ │  Projection 3│          │
│  │ (Order List)  │ │2 (Stats) │ │ (Search Index)│          │
│  └───────────────┘ └──────────┘ └──────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Replay প্রক্রিয়া:
```
   ┌─────────── EVENT STORE ───────────────────────┐
   │                                                │
   │  [E1]──▶[E2]──▶[E3]──▶[E4]──▶[E5]──▶[E6]   │
   │   │                     ▲                      │
   │   │                     │                      │
   │   │              ┌──────┴──────┐               │
   │   │              │  Snapshot   │               │
   │   │              │ (Event 4)   │               │
   │   │              └─────────────┘               │
   │   │                                            │
   └───┼────────────────────────────────────────────┘
       │
       ▼  Replay করলে:
   ┌──────────────────────────────────┐
   │  State₀ + E1 = State₁           │
   │  State₁ + E2 = State₂           │
   │  State₂ + E3 = State₃           │
   │  ...                             │
   │  StateN-1 + EN = Current State   │
   └──────────────────────────────────┘
```

### Temporal Query:
```
   Timeline:
   ─────────────────────────────────────────────▶ সময়
   │         │         │         │         │
   E1        E2        E3        E4        E5
   │                             │
   └─── "১ তারিখে balance       └─── "১৫ তারিখে
         কত ছিল?" ───────────          balance কত?"
         
   যেকোনো সময়ের state পুনর্গঠন সম্ভব!
```

---

## 💻 PHP কোড উদাহরণ

### Event Store Implementation:

```php
<?php

// Event Base Class
abstract class DomainEvent
{
    public readonly string $eventId;
    public readonly string $aggregateId;
    public readonly DateTimeImmutable $occurredAt;
    public readonly int $version;
    
    public function __construct(string $aggregateId, int $version)
    {
        $this->eventId = uniqid('evt_', true);
        $this->aggregateId = $aggregateId;
        $this->occurredAt = new DateTimeImmutable();
        $this->version = $version;
    }
    
    abstract public function getEventType(): string;
    abstract public function toPayload(): array;
}

// bKash Transaction Events
class AccountCreated extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $version,
        public readonly string $phoneNumber,
        public readonly string $ownerName
    ) {
        parent::__construct($aggregateId, $version);
    }
    
    public function getEventType(): string { return 'account.created'; }
    
    public function toPayload(): array
    {
        return [
            'phone_number' => $this->phoneNumber,
            'owner_name' => $this->ownerName,
        ];
    }
}

class MoneyTransferred extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $version,
        public readonly float $amount,
        public readonly string $recipientPhone,
        public readonly string $reference
    ) {
        parent::__construct($aggregateId, $version);
    }
    
    public function getEventType(): string { return 'money.transferred'; }
    
    public function toPayload(): array
    {
        return [
            'amount' => $this->amount,
            'recipient_phone' => $this->recipientPhone,
            'reference' => $this->reference,
        ];
    }
}

class PaymentReceived extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $version,
        public readonly float $amount,
        public readonly string $source,
        public readonly string $transactionId
    ) {
        parent::__construct($aggregateId, $version);
    }
    
    public function getEventType(): string { return 'payment.received'; }
    
    public function toPayload(): array
    {
        return [
            'amount' => $this->amount,
            'source' => $this->source,
            'transaction_id' => $this->transactionId,
        ];
    }
}

// Event Store Interface
interface EventStore
{
    public function append(string $streamId, array $events, int $expectedVersion): void;
    public function getStream(string $streamId): array;
    public function getStreamFromVersion(string $streamId, int $fromVersion): array;
}

// MySQL-based Event Store
class MySQLEventStore implements EventStore
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    public function append(string $streamId, array $events, int $expectedVersion): void
    {
        $this->pdo->beginTransaction();
        
        try {
            // Optimistic concurrency check
            $stmt = $this->pdo->prepare(
                "SELECT MAX(version) FROM events WHERE stream_id = ? FOR UPDATE"
            );
            $stmt->execute([$streamId]);
            $currentVersion = $stmt->fetchColumn() ?: 0;
            
            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException(
                    "Expected version $expectedVersion, got $currentVersion"
                );
            }
            
            $insertStmt = $this->pdo->prepare(
                "INSERT INTO events (event_id, stream_id, event_type, version, payload, occurred_at)
                 VALUES (?, ?, ?, ?, ?, ?)"
            );
            
            foreach ($events as $event) {
                $insertStmt->execute([
                    $event->eventId,
                    $streamId,
                    $event->getEventType(),
                    $event->version,
                    json_encode($event->toPayload()),
                    $event->occurredAt->format('Y-m-d H:i:s.u'),
                ]);
            }
            
            $this->pdo->commit();
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
    
    public function getStream(string $streamId): array
    {
        $stmt = $this->pdo->prepare(
            "SELECT * FROM events WHERE stream_id = ? ORDER BY version ASC"
        );
        $stmt->execute([$streamId]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getStreamFromVersion(string $streamId, int $fromVersion): array
    {
        $stmt = $this->pdo->prepare(
            "SELECT * FROM events WHERE stream_id = ? AND version > ? ORDER BY version ASC"
        );
        $stmt->execute([$streamId, $fromVersion]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}

// Aggregate - bKash Account
class BkashAccount
{
    private string $accountId;
    private string $phoneNumber;
    private float $balance = 0;
    private int $version = 0;
    private array $uncommittedEvents = [];
    
    public static function create(string $phoneNumber, string $ownerName): self
    {
        $account = new self();
        $account->apply(new AccountCreated(
            aggregateId: $phoneNumber,
            version: 1,
            phoneNumber: $phoneNumber,
            ownerName: $ownerName
        ));
        return $account;
    }
    
    public function receivePayment(float $amount, string $source, string $txnId): void
    {
        if ($amount <= 0) {
            throw new \InvalidArgumentException("পরিমাণ শূন্যের বেশি হতে হবে");
        }
        
        $this->apply(new PaymentReceived(
            aggregateId: $this->accountId,
            version: $this->version + 1,
            amount: $amount,
            source: $source,
            transactionId: $txnId
        ));
    }
    
    public function transferMoney(float $amount, string $recipientPhone): void
    {
        if ($amount > $this->balance) {
            throw new InsufficientBalanceException("অপর্যাপ্ত ব্যালেন্স");
        }
        
        $this->apply(new MoneyTransferred(
            aggregateId: $this->accountId,
            version: $this->version + 1,
            amount: $amount,
            recipientPhone: $recipientPhone,
            reference: uniqid('TXN')
        ));
    }
    
    private function apply(DomainEvent $event): void
    {
        $this->applyEvent($event);
        $this->uncommittedEvents[] = $event;
    }
    
    private function applyEvent(DomainEvent $event): void
    {
        match ($event::class) {
            AccountCreated::class => $this->onAccountCreated($event),
            PaymentReceived::class => $this->onPaymentReceived($event),
            MoneyTransferred::class => $this->onMoneyTransferred($event),
        };
        $this->version = $event->version;
    }
    
    private function onAccountCreated(AccountCreated $event): void
    {
        $this->accountId = $event->aggregateId;
        $this->phoneNumber = $event->phoneNumber;
    }
    
    private function onPaymentReceived(PaymentReceived $event): void
    {
        $this->balance += $event->amount;
    }
    
    private function onMoneyTransferred(MoneyTransferred $event): void
    {
        $this->balance -= $event->amount;
    }
    
    // Snapshot support
    public function takeSnapshot(): array
    {
        return [
            'account_id' => $this->accountId,
            'phone_number' => $this->phoneNumber,
            'balance' => $this->balance,
            'version' => $this->version,
        ];
    }
    
    public static function fromSnapshot(array $snapshot): self
    {
        $account = new self();
        $account->accountId = $snapshot['account_id'];
        $account->phoneNumber = $snapshot['phone_number'];
        $account->balance = $snapshot['balance'];
        $account->version = $snapshot['version'];
        return $account;
    }
    
    public function getUncommittedEvents(): array { return $this->uncommittedEvents; }
    public function getBalance(): float { return $this->balance; }
    public function getVersion(): int { return $this->version; }
}

// Projection - Read Model
class AccountBalanceProjection
{
    private PDO $pdo;
    
    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }
    
    public function handle(DomainEvent $event): void
    {
        match ($event->getEventType()) {
            'account.created' => $this->onAccountCreated($event),
            'payment.received' => $this->onPaymentReceived($event),
            'money.transferred' => $this->onMoneyTransferred($event),
        };
    }
    
    private function onAccountCreated(AccountCreated $event): void
    {
        $stmt = $this->pdo->prepare(
            "INSERT INTO account_balances (phone, owner_name, balance) VALUES (?, ?, 0)"
        );
        $stmt->execute([$event->phoneNumber, $event->ownerName]);
    }
    
    private function onPaymentReceived(PaymentReceived $event): void
    {
        $stmt = $this->pdo->prepare(
            "UPDATE account_balances SET balance = balance + ? WHERE phone = ?"
        );
        $stmt->execute([$event->amount, $event->aggregateId]);
    }
    
    private function onMoneyTransferred(MoneyTransferred $event): void
    {
        $stmt = $this->pdo->prepare(
            "UPDATE account_balances SET balance = balance - ? WHERE phone = ?"
        );
        $stmt->execute([$event->amount, $event->aggregateId]);
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Event Sourcing with Node.js:

```javascript
// Event Base
class DomainEvent {
    constructor(aggregateId, version) {
        this.eventId = `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
        this.aggregateId = aggregateId;
        this.version = version;
        this.occurredAt = new Date().toISOString();
    }
}

// Order Events (Pathao Food Order)
class OrderPlaced extends DomainEvent {
    constructor(aggregateId, version, { customerId, restaurantId, items, deliveryAddress }) {
        super(aggregateId, version);
        this.eventType = 'order.placed';
        this.payload = { customerId, restaurantId, items, deliveryAddress };
    }
}

class ItemAdded extends DomainEvent {
    constructor(aggregateId, version, { itemId, name, price, quantity }) {
        super(aggregateId, version);
        this.eventType = 'item.added';
        this.payload = { itemId, name, price, quantity };
    }
}

class PaymentProcessed extends DomainEvent {
    constructor(aggregateId, version, { amount, method, transactionId }) {
        super(aggregateId, version);
        this.eventType = 'payment.processed';
        this.payload = { amount, method, transactionId };
    }
}

class RiderAssigned extends DomainEvent {
    constructor(aggregateId, version, { riderId, riderName, estimatedTime }) {
        super(aggregateId, version);
        this.eventType = 'rider.assigned';
        this.payload = { riderId, riderName, estimatedTime };
    }
}

class OrderDelivered extends DomainEvent {
    constructor(aggregateId, version, { deliveredAt, rating }) {
        super(aggregateId, version);
        this.eventType = 'order.delivered';
        this.payload = { deliveredAt, rating };
    }
}

// Event Store (In-Memory for demonstration)
class EventStore {
    constructor() {
        this.streams = new Map();
        this.allEvents = [];
        this.subscribers = [];
    }

    async append(streamId, events, expectedVersion) {
        const stream = this.streams.get(streamId) || [];
        const currentVersion = stream.length;

        if (currentVersion !== expectedVersion) {
            throw new Error(
                `Concurrency conflict: expected v${expectedVersion}, got v${currentVersion}`
            );
        }

        stream.push(...events);
        this.streams.set(streamId, stream);
        this.allEvents.push(...events);

        // Event publish করা (Projections আপডেটের জন্য)
        for (const event of events) {
            this.subscribers.forEach(handler => handler(event));
        }
    }

    getStream(streamId) {
        return this.streams.get(streamId) || [];
    }

    getStreamFromVersion(streamId, fromVersion) {
        const stream = this.streams.get(streamId) || [];
        return stream.slice(fromVersion);
    }

    // Temporal Query - নির্দিষ্ট সময়ের state পেতে
    getStreamUntil(streamId, timestamp) {
        const stream = this.streams.get(streamId) || [];
        return stream.filter(e => new Date(e.occurredAt) <= new Date(timestamp));
    }

    subscribe(handler) {
        this.subscribers.push(handler);
    }
}

// Aggregate - Pathao Food Order
class FoodOrder {
    #id;
    #status;
    #items = [];
    #total = 0;
    #version = 0;
    #uncommittedEvents = [];

    static create(orderId, customerId, restaurantId, deliveryAddress) {
        const order = new FoodOrder();
        order.#applyNew(new OrderPlaced(orderId, 1, {
            customerId,
            restaurantId,
            items: [],
            deliveryAddress,
        }));
        return order;
    }

    addItem(itemId, name, price, quantity) {
        if (this.#status !== 'placed') {
            throw new Error('অর্ডার confirmed হয়ে গেলে আইটেম যোগ করা যাবে না');
        }
        this.#applyNew(new ItemAdded(this.#id, this.#version + 1, {
            itemId, name, price, quantity,
        }));
    }

    processPayment(amount, method, transactionId) {
        if (amount < this.#total) {
            throw new Error(`পেমেন্ট কম: প্রয়োজন ৳${this.#total}, পাওয়া গেছে ৳${amount}`);
        }
        this.#applyNew(new PaymentProcessed(this.#id, this.#version + 1, {
            amount, method, transactionId,
        }));
    }

    assignRider(riderId, riderName, estimatedTime) {
        if (this.#status !== 'paid') {
            throw new Error('পেমেন্ট ছাড়া রাইডার assign করা যাবে না');
        }
        this.#applyNew(new RiderAssigned(this.#id, this.#version + 1, {
            riderId, riderName, estimatedTime,
        }));
    }

    deliver(rating) {
        this.#applyNew(new OrderDelivered(this.#id, this.#version + 1, {
            deliveredAt: new Date().toISOString(),
            rating,
        }));
    }

    // Rebuild from events (Event Replay)
    static fromEvents(events) {
        const order = new FoodOrder();
        events.forEach(event => order.#applyExisting(event));
        return order;
    }

    #applyNew(event) {
        this.#applyExisting(event);
        this.#uncommittedEvents.push(event);
    }

    #applyExisting(event) {
        switch (event.eventType) {
            case 'order.placed':
                this.#id = event.aggregateId;
                this.#status = 'placed';
                break;
            case 'item.added':
                this.#items.push(event.payload);
                this.#total += event.payload.price * event.payload.quantity;
                break;
            case 'payment.processed':
                this.#status = 'paid';
                break;
            case 'rider.assigned':
                this.#status = 'in_delivery';
                break;
            case 'order.delivered':
                this.#status = 'delivered';
                break;
        }
        this.#version = event.version;
    }

    get uncommittedEvents() { return [...this.#uncommittedEvents]; }
    get version() { return this.#version; }
    get total() { return this.#total; }
    get status() { return this.#status; }
}

// Projection - Order Dashboard (Read Model)
class OrderDashboardProjection {
    constructor() {
        this.orders = new Map();
    }

    handle(event) {
        switch (event.eventType) {
            case 'order.placed':
                this.orders.set(event.aggregateId, {
                    id: event.aggregateId,
                    status: 'placed',
                    restaurant: event.payload.restaurantId,
                    items: [],
                    total: 0,
                    createdAt: event.occurredAt,
                });
                break;
            case 'item.added': {
                const order = this.orders.get(event.aggregateId);
                order.items.push(event.payload);
                order.total += event.payload.price * event.payload.quantity;
                break;
            }
            case 'payment.processed': {
                const order = this.orders.get(event.aggregateId);
                order.status = 'paid';
                order.paymentMethod = event.payload.method;
                break;
            }
            case 'rider.assigned': {
                const order = this.orders.get(event.aggregateId);
                order.status = 'in_delivery';
                order.rider = event.payload.riderName;
                order.eta = event.payload.estimatedTime;
                break;
            }
            case 'order.delivered': {
                const order = this.orders.get(event.aggregateId);
                order.status = 'delivered';
                order.deliveredAt = event.payload.deliveredAt;
                break;
            }
        }
    }

    getOrder(orderId) { return this.orders.get(orderId); }
    getActiveOrders() {
        return [...this.orders.values()].filter(o => o.status !== 'delivered');
    }
}

// ব্যবহার উদাহরণ
async function main() {
    const eventStore = new EventStore();
    const dashboard = new OrderDashboardProjection();
    eventStore.subscribe(event => dashboard.handle(event));

    // নতুন অর্ডার তৈরি
    const order = FoodOrder.create(
        'ORD-001',
        'CUST-123',
        'sultans-dine-gulshan',
        'বনানী, রোড ১১, বাড়ি ৪২'
    );

    order.addItem('ITM-1', 'কাচ্চি বিরিয়ানি', 350, 2);
    order.addItem('ITM-2', 'বোরহানি', 60, 2);
    order.processPayment(820, 'bKash', 'TXN-98765');
    order.assignRider('RDR-45', 'করিম ভাই', '30 মিনিট');

    await eventStore.append('ORD-001', order.uncommittedEvents, 0);

    // Dashboard থেকে অর্ডার দেখা (Read Model)
    console.log(dashboard.getOrder('ORD-001'));
    // Temporal Query: ১ ঘন্টা আগের state
    const pastEvents = eventStore.getStreamUntil('ORD-001', '2024-01-15T11:00:00Z');
    const pastState = FoodOrder.fromEvents(pastEvents);
    console.log('অতীতের অবস্থা:', pastState.status);
}
```

---

## 🔗 CQRS Integration

Event Sourcing সাধারণত CQRS (Command Query Responsibility Segregation) এর সাথে ব্যবহৃত হয়:

```
┌──────────────────────────────────────────────────────────┐
│                    CQRS + Event Sourcing                   │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  WRITE SIDE (Command)          READ SIDE (Query)         │
│  ┌─────────────────┐          ┌─────────────────┐       │
│  │ Command Handler │          │  Query Handler  │       │
│  └────────┬────────┘          └────────┬────────┘       │
│           │                            │                  │
│           ▼                            ▼                  │
│  ┌─────────────────┐          ┌─────────────────┐       │
│  │   Aggregate     │          │   Read Model    │       │
│  │ (Domain Logic)  │          │  (Denormalized) │       │
│  └────────┬────────┘          └─────────────────┘       │
│           │                            ▲                  │
│           ▼                            │                  │
│  ┌─────────────────┐         ┌────────┴────────┐       │
│  │  Event Store    │────────▶│   Projections   │       │
│  │ (Source of Truth)│  Events │  (Event Handlers)│       │
│  └─────────────────┘         └─────────────────┘       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

---

## 📸 Snapshots এবং Performance

যখন একটি Aggregate-এ হাজার হাজার event থাকে, প্রতিবার শুরু থেকে replay করা ব্যয়বহুল। Snapshot এই সমস্যা সমাধান করে:

```
   Events: [E1] [E2] [E3] ... [E100] [E101] ... [E200]
                                 ▲
                                 │
                          ┌──────┴──────┐
                          │  Snapshot   │
                          │ at Event 100│
                          └─────────────┘
                          
   পুনর্গঠনে: Snapshot(E100) + E101 + E102 + ... + E200
   বনাম: E1 + E2 + ... + E200 (অনেক ধীর!)
```

---

## ✅ কখন ব্যবহার করবেন

| ব্যবহার করবেন ✅ | ব্যবহার করবেন না ❌ |
|---|---|
| Complete audit trail প্রয়োজন (ব্যাংকিং, bKash) | সাধারণ CRUD অ্যাপ্লিকেশন |
| Time-travel debugging দরকার | State history প্রয়োজন নেই |
| Complex domain logic আছে | টিম Event Sourcing-এ অভিজ্ঞ নয় |
| Event-driven architecture তৈরি করছেন | ছোট প্রজেক্ট, সীমিত সময় |
| Regulatory compliance (আর্থিক সেবা) | Strong consistency জরুরি |
| Undo/Redo functionality দরকার | সহজ read-heavy অ্যাপ্লিকেশন |

### ⚠️ চ্যালেঞ্জসমূহ:
1. **Eventual Consistency**: Read model আপডেট হতে সময় লাগে
2. **Storage Growth**: Event store বড় হতে থাকে (archiving strategy দরকার)
3. **Schema Evolution**: পুরানো events-এর format পরিবর্তন কঠিন
4. **Complexity**: Traditional CRUD-এর চেয়ে অনেক বেশি জটিল
5. **Event Ordering**: Distributed system-এ event ordering নিশ্চিত করা কঠিন

---

## 📚 সারসংক্ষেপ

> Event Sourcing = **ঘটনার ইতিহাস সংরক্ষণ** (কী হয়েছিল তার সব রেকর্ড)
> Traditional CRUD = **বর্তমান অবস্থা সংরক্ষণ** (শুধু এখন কী আছে)

Event Sourcing বাংলাদেশের fintech সেক্টরে (bKash, Nagad, Rocket) এবং delivery tracking-এ (Pathao, Foodpanda) অত্যন্ত প্রাসঙ্গিক, কারণ এসব সিস্টেমে সম্পূর্ণ লেনদেনের ইতিহাস ও audit trail অপরিহার্য।
