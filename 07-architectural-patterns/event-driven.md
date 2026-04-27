# ⚡ ইভেন্ট-ড্রিভেন আর্কিটেকচার (Event-Driven Architecture - EDA)

## 📌 সংজ্ঞা ও মূল ধারণা

**Event-Driven Architecture (EDA)** হলো একটি সফটওয়্যার আর্কিটেকচার প্যাটার্ন যেখানে সিস্টেমের বিভিন্ন component-গুলো **event**-এর মাধ্যমে একে অপরের সাথে যোগাযোগ করে। কোনো component সরাসরি অন্য component-কে call করে না — বরং একটি event publish করে, এবং যেসব component সেই event-এ আগ্রহী তারা সেটি consume করে।

EDA-র মূল দর্শন হলো **temporal decoupling** এবং **spatial decoupling**। Producer জানে না কে consume করবে, কখন করবে, বা আদৌ কেউ করবে কিনা। এই decoupling-ই EDA-কে highly scalable এবং resilient করে তোলে।

### Event vs Message vs Command — পার্থক্য

```
┌─────────────┬──────────────────────────────────────────────────────────┐
│   ধরন       │  বৈশিষ্ট্য                                              │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Event       │ অতীতে ঘটে যাওয়া কিছু (past tense): OrderPlaced,       │
│             │ PaymentCompleted। Immutable। Publisher জানে না কে       │
│             │ listen করছে। Fire-and-forget।                           │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Command     │ কাউকে কিছু করতে বলা (imperative): PlaceOrder,          │
│             │ ProcessPayment। একটি specific receiver আছে।            │
│             │ Sender response আশা করে।                                │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Message     │ Event ও Command দুটোই message। Message হলো generic     │
│             │ container যা broker-এর মধ্য দিয়ে যায়।                  │
└─────────────┴──────────────────────────────────────────────────────────┘
```

### Event-এর তিনটি প্রধান ধরন

**১. Domain Event:** একটি bounded context-এর ভেতরে ঘটা ব্যবসায়িক ঘটনা। যেমন `OrderPlaced`, `AccountDebited`। এগুলো domain model-এর অংশ এবং ubiquitous language অনুসরণ করে।

**২. Integration Event:** বিভিন্ন bounded context বা microservice-এর মধ্যে যোগাযোগের জন্য ব্যবহৃত event। যেমন bKash service থেকে Order service-এ `PaymentConfirmed` event পাঠানো। এগুলোতে schema versioning অত্যন্ত গুরুত্বপূর্ণ।

**৩. Notification Event:** শুধু জানানোর জন্য — "কিছু ঘটেছে, বিস্তারিত জানতে চাইলে query করো"। Payload ছোট রাখে, কিন্তু consumer-কে callback করতে হয়।

---

## 🏠 বাস্তব জীবনের উদাহরণ

**পত্রিকার সাবস্ক্রিপশন (Publisher-Subscriber) অ্যানালজি:**

ধরুন আপনি প্রথম আলো পত্রিকার সাবস্ক্রাইবার। প্রথম আলো (Publisher) প্রতিদিন পত্রিকা ছাপায়। তারা জানে না আপনি সেটা পড়বেন কিনা, কখন পড়বেন, বা আদৌ পড়বেন কিনা। তারা শুধু আপনার ঠিকানায় deliver করে।

- **Publisher** = প্রথম আলো (event producer)
- **Event** = আজকের পত্রিকা (data payload)
- **Event Channel** = ডাক বিভাগ / হকার (message broker)
- **Subscriber** = আপনি (event consumer)
- **Subscribe** = সাবস্ক্রিপশন নেওয়া (event registration)

আপনি unsubscribe করলে প্রথম আলোর কোনো কিছু বদলায় না — তারা আগের মতোই পত্রিকা ছাপাতে থাকে। এটাই **decoupling**।

**bKash Transaction Event Flow:**

```
ব্যবহারকারী Send Money করলো
    → TransactionInitiated event fire
        → Fraud Detection service শোনে → চেক করে
        → Balance service শোনে → ব্যালেন্স কমায়
        → Notification service শোনে → SMS পাঠায়
        → Analytics service শোনে → report-এ যোগ করে
```

bKash-এর Send Money feature কখনো সরাসরি SMS service-কে call করে না। সে শুধু `TransactionCompleted` event publish করে, আর SMS service সেটা subscribe করে নিজের কাজ করে।

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Mediator vs Broker Topology

```
                    ╔══════════════════════════════════════════════╗
                    ║         MEDIATOR TOPOLOGY                    ║
                    ╠══════════════════════════════════════════════╣
                    ║                                              ║
                    ║   ┌─────────┐    Event    ┌──────────────┐  ║
                    ║   │ Client  │───────────→│  Event        │  ║
                    ║   └─────────┘            │  Mediator     │  ║
                    ║                          │ (Orchestrator)│  ║
                    ║                          └──────┬───────┘  ║
                    ║                     ┌───────┬───┴───┬────┐  ║
                    ║                     ▼       ▼       ▼    ▼  ║
                    ║                  ┌─────┐┌─────┐┌─────┐┌───┐ ║
                    ║                  │Proc ││Proc ││Proc ││Pro│ ║
                    ║                  │ A   ││ B   ││ C   ││ D │ ║
                    ║                  └─────┘└─────┘└─────┘└───┘ ║
                    ╚══════════════════════════════════════════════╝

                    ╔══════════════════════════════════════════════╗
                    ║          BROKER TOPOLOGY                     ║
                    ╠══════════════════════════════════════════════╣
                    ║                                              ║
                    ║   ┌─────┐  event  ┌─────────────┐  event   ║
                    ║   │Svc A│────────→│ Event       │────────→ ║
                    ║   └─────┘         │ Channel     │  ┌─────┐ ║
                    ║   ┌─────┐  event  │ (Broker)    │→│Svc C│  ║
                    ║   │Svc B│────────→│             │  └─────┘ ║
                    ║   └─────┘         └─────────────┘  ┌─────┐ ║
                    ║                        │ event      │Svc D│ ║
                    ║                        └──────────→└─────┘  ║
                    ╚══════════════════════════════════════════════╝
```

### Event Flow — Pub/Sub Architecture

```
    ┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
    │   Producer   │         │   Message Broker  │         │   Consumer   │
    │              │  publish │                  │subscribe│              │
    │  Order Svc   │────────→│  Topic/Exchange   │────────→│ Payment Svc  │
    │              │         │                  │         │              │
    └──────────────┘         │  ┌────────────┐  │         └──────────────┘
                             │  │  Queue A   │  │         ┌──────────────┐
                             │  ├────────────┤  │────────→│Inventory Svc │
                             │  │  Queue B   │  │         └──────────────┘
                             │  ├────────────┤  │         ┌──────────────┐
                             │  │  Queue C   │  │────────→│Notification  │
                             │  └────────────┘  │         └──────────────┘
                             └──────────────────┘
```

---

## 💻 EDA Topologies

### ১. Mediator Topology

Mediator topology-তে একটি central **event mediator** থাকে যে initial event গ্রহণ করে এবং সেটিকে কয়েকটি asynchronous step-এ ভাগ করে বিভিন্ন event processor-কে পাঠায়। Mediator পুরো workflow-এর orchestration করে — কোন step কোন order-এ হবে, কোনটা parallel হবে, কোনটা sequential হবে — সব সে নিয়ন্ত্রণ করে।

**কখন ব্যবহার করবেন:** যখন event processing-এ multiple ordered steps থাকে এবং error handling ও compensation প্রয়োজন। যেমন: e-commerce order fulfillment।

#### PHP (Laravel) — Mediator Topology

```php
<?php

declare(strict_types=1);

// Event Mediator — orchestrates the order fulfillment workflow
final class OrderFulfillmentMediator
{
    public function __construct(
        private readonly EventBus $eventBus,
        private readonly LoggerInterface $logger,
    ) {}

    public function handle(OrderPlaced $initialEvent): void
    {
        $orderId = $initialEvent->orderId;
        $this->logger->info("Mediator: Order fulfillment started", ['orderId' => $orderId]);

        try {
            // Step 1: Payment validation (sequential)
            $this->eventBus->dispatch(new ValidatePayment(
                orderId: $orderId,
                amount: $initialEvent->totalAmount,
                paymentMethod: $initialEvent->paymentMethod,
            ));

            // Step 2: Parallel steps after payment
            $this->eventBus->dispatchParallel([
                new ReserveInventory(orderId: $orderId, items: $initialEvent->items),
                new CalculateShipping(orderId: $orderId, address: $initialEvent->shippingAddress),
                new NotifyWarehouse(orderId: $orderId, items: $initialEvent->items),
            ]);

            // Step 3: Final confirmation
            $this->eventBus->dispatch(new ConfirmOrder(orderId: $orderId));

        } catch (PaymentFailedException $e) {
            $this->eventBus->dispatch(new OrderRejected(
                orderId: $orderId,
                reason: $e->getMessage(),
            ));
        } catch (InventoryException $e) {
            // Compensation: refund payment
            $this->eventBus->dispatch(new RefundPayment(orderId: $orderId));
            $this->eventBus->dispatch(new OrderCancelled(orderId: $orderId, reason: 'Out of stock'));
        }
    }
}

// Event Processor — handles a specific step
final class PaymentValidationProcessor
{
    public function __construct(
        private readonly PaymentGateway $gateway,
    ) {}

    public function handle(ValidatePayment $event): void
    {
        $result = $this->gateway->charge(
            orderId: $event->orderId,
            amount: $event->amount,
            method: $event->paymentMethod,
        );

        if (!$result->isSuccessful()) {
            throw new PaymentFailedException($result->errorMessage);
        }
    }
}
```

#### JavaScript (Node.js) — Mediator Topology

```javascript
// Event Mediator — orchestrates order fulfillment
class OrderFulfillmentMediator {
    #eventBus;
    #logger;

    constructor(eventBus, logger) {
        this.#eventBus = eventBus;
        this.#logger = logger;
    }

    async handle(initialEvent) {
        const { orderId, totalAmount, paymentMethod, items, shippingAddress } = initialEvent;
        this.#logger.info(`Mediator: Order fulfillment started`, { orderId });

        try {
            // Step 1: Payment validation (sequential)
            await this.#eventBus.dispatch({
                type: 'ValidatePayment',
                payload: { orderId, amount: totalAmount, paymentMethod },
            });

            // Step 2: Parallel steps
            await Promise.all([
                this.#eventBus.dispatch({
                    type: 'ReserveInventory',
                    payload: { orderId, items },
                }),
                this.#eventBus.dispatch({
                    type: 'CalculateShipping',
                    payload: { orderId, address: shippingAddress },
                }),
                this.#eventBus.dispatch({
                    type: 'NotifyWarehouse',
                    payload: { orderId, items },
                }),
            ]);

            // Step 3: Confirm
            await this.#eventBus.dispatch({ type: 'ConfirmOrder', payload: { orderId } });

        } catch (error) {
            if (error instanceof PaymentFailedError) {
                await this.#eventBus.dispatch({
                    type: 'OrderRejected',
                    payload: { orderId, reason: error.message },
                });
            } else if (error instanceof InventoryError) {
                await this.#eventBus.dispatch({ type: 'RefundPayment', payload: { orderId } });
                await this.#eventBus.dispatch({
                    type: 'OrderCancelled',
                    payload: { orderId, reason: 'Out of stock' },
                });
            }
        }
    }
}
```

---

### ২. Broker Topology

Broker topology-তে কোনো central mediator নেই। প্রতিটি event processor সরাসরি event broker (যেমন RabbitMQ, Kafka) এর সাথে connected। একটি processor event receive করে, process করে, এবং নতুন event publish করে — যেটা অন্য processor consume করে। এটা একটা **chain reaction**-এর মতো।

**কখন ব্যবহার করবেন:** যখন event flow সহজ এবং central orchestration-এর দরকার নেই। যেমন: real-time notification system।

#### PHP — Broker Topology

```php
<?php

declare(strict_types=1);

// প্রতিটি service নিজে event publish ও consume করে — কোনো central coordinator নেই

final class OrderService
{
    public function __construct(
        private readonly MessageBroker $broker,
    ) {}

    public function placeOrder(OrderDTO $dto): void
    {
        $order = Order::create($dto);
        $order->save();

        // Broker-এ event publish — কে শুনবে জানা নেই
        $this->broker->publish('orders', new OrderPlaced(
            orderId: $order->id,
            items: $order->items->toArray(),
            customerId: $order->customer_id,
            totalAmount: $order->total,
            occurredAt: new DateTimeImmutable(),
        ));
    }
}

// Payment service স্বাধীনভাবে OrderPlaced শোনে
final class PaymentEventHandler
{
    public function __construct(
        private readonly PaymentProcessor $processor,
        private readonly MessageBroker $broker,
    ) {}

    #[EventListener('orders.OrderPlaced')]
    public function onOrderPlaced(OrderPlaced $event): void
    {
        $result = $this->processor->charge($event->customerId, $event->totalAmount);

        // নতুন event publish — chain reaction
        $this->broker->publish('payments', $result->isSuccessful()
            ? new PaymentCompleted(orderId: $event->orderId, transactionId: $result->txnId)
            : new PaymentFailed(orderId: $event->orderId, reason: $result->error)
        );
    }
}

// Inventory service PaymentCompleted শোনে
final class InventoryEventHandler
{
    #[EventListener('payments.PaymentCompleted')]
    public function onPaymentCompleted(PaymentCompleted $event): void
    {
        $this->reserveStock($event->orderId);
        $this->broker->publish('inventory', new StockReserved(orderId: $event->orderId));
    }
}
```

#### JavaScript — Broker Topology

```javascript
// Order Service — broker-এ event publish করে
class OrderService {
    #broker;

    constructor(broker) {
        this.#broker = broker;
    }

    async placeOrder(dto) {
        const order = await Order.create(dto);

        await this.#broker.publish('orders', {
            type: 'OrderPlaced',
            payload: {
                orderId: order.id,
                items: order.items,
                customerId: order.customerId,
                totalAmount: order.total,
            },
            metadata: { occurredAt: new Date().toISOString(), version: 1 },
        });
    }
}

// Payment service — chain-এ পরবর্তী link
class PaymentEventHandler {
    #processor;
    #broker;

    constructor(processor, broker) {
        this.#processor = processor;
        this.#broker = broker;
    }

    async onOrderPlaced(event) {
        const { customerId, totalAmount, orderId } = event.payload;
        const result = await this.#processor.charge(customerId, totalAmount);

        await this.#broker.publish('payments', result.success
            ? { type: 'PaymentCompleted', payload: { orderId, txnId: result.txnId } }
            : { type: 'PaymentFailed', payload: { orderId, reason: result.error } }
        );
    }
}

// Inventory service — PaymentCompleted শোনে
class InventoryEventHandler {
    async onPaymentCompleted(event) {
        await this.reserveStock(event.payload.orderId);
        await this.broker.publish('inventory', {
            type: 'StockReserved',
            payload: { orderId: event.payload.orderId },
        });
    }
}
```

---

## 🔥 Complex Implementation Scenarios

### ১. Event Sourcing

Event Sourcing হলো এমন একটি পদ্ধতি যেখানে application-এর state সরাসরি সংরক্ষণ করা হয় না। বরং state-এ পরিবর্তন আনা **প্রতিটি event** ক্রমানুসারে সংরক্ষণ করা হয়। বর্তমান state পেতে হলে শুরু থেকে সব event **replay** করতে হয়।

**Traditional CRUD বনাম Event Sourcing:**

```
CRUD:                                Event Sourcing:
┌──────────────────┐                 ┌──────────────────────────────┐
│ Account Table    │                 │ Event Store                  │
│ id: 1            │                 │ 1. AccountOpened(bal: 0)     │
│ balance: 1500    │                 │ 2. MoneyDeposited(amt: 2000) │
│ (শুধু final state)│                 │ 3. MoneyWithdrawn(amt: 500)  │
└──────────────────┘                 │ (পুরো ইতিহাস সংরক্ষিত!)      │
                                     │ Current state = replay all   │
                                     └──────────────────────────────┘
```

CRUD-এ আপনি শুধু জানেন balance 1500। কিন্তু Event Sourcing-এ আপনি জানেন কীভাবে 1500 হলো — কোন কোন transaction হয়েছে, কখন হয়েছে।

#### PHP — Event Sourcing (Bank Account)

```php
<?php

declare(strict_types=1);

// Domain Events
final readonly class AccountOpened
{
    public function __construct(
        public string $accountId,
        public string $holderName,
        public DateTimeImmutable $occurredAt = new DateTimeImmutable(),
    ) {}
}

final readonly class MoneyDeposited
{
    public function __construct(
        public string $accountId,
        public float $amount,
        public string $reference,
        public DateTimeImmutable $occurredAt = new DateTimeImmutable(),
    ) {}
}

final readonly class MoneyWithdrawn
{
    public function __construct(
        public string $accountId,
        public float $amount,
        public string $reference,
        public DateTimeImmutable $occurredAt = new DateTimeImmutable(),
    ) {}
}

// Aggregate — state rebuild from events
final class BankAccountAggregate
{
    private string $accountId;
    private string $holderName;
    private float $balance = 0.0;
    private int $version = 0;

    /** @var list<object> */
    private array $uncommittedEvents = [];

    public static function open(string $accountId, string $holderName): self
    {
        $account = new self();
        $account->apply(new AccountOpened($accountId, $holderName));
        return $account;
    }

    public function deposit(float $amount, string $reference): void
    {
        if ($amount <= 0) {
            throw new InvalidArgumentException('Deposit amount must be positive');
        }
        $this->apply(new MoneyDeposited($this->accountId, $amount, $reference));
    }

    public function withdraw(float $amount, string $reference): void
    {
        if ($amount > $this->balance) {
            throw new InsufficientFundsException(
                "Cannot withdraw {$amount}, balance is {$this->balance}"
            );
        }
        $this->apply(new MoneyWithdrawn($this->accountId, $amount, $reference));
    }

    public function getBalance(): float
    {
        return $this->balance;
    }

    // Event apply — uncommitted events track করে
    private function apply(object $event): void
    {
        $this->when($event);
        $this->uncommittedEvents[] = $event;
        $this->version++;
    }

    // State mutation — শুধুমাত্র event থেকে state change হয়
    private function when(object $event): void
    {
        match ($event::class) {
            AccountOpened::class => $this->whenAccountOpened($event),
            MoneyDeposited::class => $this->whenMoneyDeposited($event),
            MoneyWithdrawn::class => $this->whenMoneyWithdrawn($event),
            default => throw new UnknownEventException($event::class),
        };
    }

    private function whenAccountOpened(AccountOpened $e): void
    {
        $this->accountId = $e->accountId;
        $this->holderName = $e->holderName;
        $this->balance = 0.0;
    }

    private function whenMoneyDeposited(MoneyDeposited $e): void
    {
        $this->balance += $e->amount;
    }

    private function whenMoneyWithdrawn(MoneyWithdrawn $e): void
    {
        $this->balance -= $e->amount;
    }

    // Event Store থেকে aggregate rebuild
    public static function reconstituteFromHistory(array $events): self
    {
        $account = new self();
        foreach ($events as $event) {
            $account->when($event);
            $account->version++;
        }
        return $account;
    }

    public function getUncommittedEvents(): array
    {
        return $this->uncommittedEvents;
    }

    public function clearUncommittedEvents(): void
    {
        $this->uncommittedEvents = [];
    }
}

// Event Store — events persist করে
final class EventStore
{
    public function __construct(
        private readonly PDO $pdo,
        private readonly EventSerializer $serializer,
    ) {}

    public function append(string $aggregateId, array $events, int $expectedVersion): void
    {
        $this->pdo->beginTransaction();

        try {
            // Optimistic concurrency check
            $stmt = $this->pdo->prepare(
                'SELECT MAX(version) FROM event_store WHERE aggregate_id = ?'
            );
            $stmt->execute([$aggregateId]);
            $currentVersion = (int) $stmt->fetchColumn();

            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException(
                    "Expected version {$expectedVersion}, but found {$currentVersion}"
                );
            }

            $insert = $this->pdo->prepare(
                'INSERT INTO event_store (aggregate_id, version, event_type, payload, occurred_at)
                 VALUES (?, ?, ?, ?, ?)'
            );

            foreach ($events as $i => $event) {
                $insert->execute([
                    $aggregateId,
                    $expectedVersion + $i + 1,
                    $event::class,
                    $this->serializer->serialize($event),
                    $event->occurredAt->format('Y-m-d H:i:s.u'),
                ]);
            }

            $this->pdo->commit();
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }

    public function getEvents(string $aggregateId): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT event_type, payload FROM event_store
             WHERE aggregate_id = ? ORDER BY version ASC'
        );
        $stmt->execute([$aggregateId]);

        return array_map(
            fn(array $row) => $this->serializer->deserialize($row['event_type'], $row['payload']),
            $stmt->fetchAll(PDO::FETCH_ASSOC),
        );
    }

    // Snapshot — অনেক event থাকলে performance-এর জন্য
    public function getEventsAfterSnapshot(string $aggregateId): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT version, snapshot_data FROM snapshots
             WHERE aggregate_id = ? ORDER BY version DESC LIMIT 1'
        );
        $stmt->execute([$aggregateId]);
        $snapshot = $stmt->fetch(PDO::FETCH_ASSOC);

        $fromVersion = $snapshot ? (int) $snapshot['version'] : 0;

        $stmt = $this->pdo->prepare(
            'SELECT event_type, payload FROM event_store
             WHERE aggregate_id = ? AND version > ? ORDER BY version ASC'
        );
        $stmt->execute([$aggregateId, $fromVersion]);

        return [
            'snapshot' => $snapshot ? json_decode($snapshot['snapshot_data'], true) : null,
            'events' => array_map(
                fn(array $row) => $this->serializer->deserialize($row['event_type'], $row['payload']),
                $stmt->fetchAll(PDO::FETCH_ASSOC),
            ),
        ];
    }
}
```

#### JavaScript (Node.js) — Event Sourcing (Bank Account)

```javascript
// Domain Events
class AccountOpened {
    constructor(accountId, holderName) {
        this.type = 'AccountOpened';
        this.accountId = accountId;
        this.holderName = holderName;
        this.occurredAt = new Date().toISOString();
    }
}

class MoneyDeposited {
    constructor(accountId, amount, reference) {
        this.type = 'MoneyDeposited';
        this.accountId = accountId;
        this.amount = amount;
        this.reference = reference;
        this.occurredAt = new Date().toISOString();
    }
}

class MoneyWithdrawn {
    constructor(accountId, amount, reference) {
        this.type = 'MoneyWithdrawn';
        this.accountId = accountId;
        this.amount = amount;
        this.reference = reference;
        this.occurredAt = new Date().toISOString();
    }
}

// Aggregate
class BankAccountAggregate {
    #accountId;
    #holderName;
    #balance = 0;
    #version = 0;
    #uncommittedEvents = [];

    static open(accountId, holderName) {
        const account = new BankAccountAggregate();
        account.#applyEvent(new AccountOpened(accountId, holderName));
        return account;
    }

    deposit(amount, reference) {
        if (amount <= 0) throw new Error('Deposit amount must be positive');
        this.#applyEvent(new MoneyDeposited(this.#accountId, amount, reference));
    }

    withdraw(amount, reference) {
        if (amount > this.#balance) {
            throw new InsufficientFundsError(
                `Cannot withdraw ${amount}, balance is ${this.#balance}`
            );
        }
        this.#applyEvent(new MoneyWithdrawn(this.#accountId, amount, reference));
    }

    get balance() { return this.#balance; }
    get version() { return this.#version; }
    get uncommittedEvents() { return [...this.#uncommittedEvents]; }

    clearUncommittedEvents() { this.#uncommittedEvents = []; }

    #applyEvent(event) {
        this.#when(event);
        this.#uncommittedEvents.push(event);
        this.#version++;
    }

    #when(event) {
        const handlers = {
            AccountOpened: (e) => {
                this.#accountId = e.accountId;
                this.#holderName = e.holderName;
                this.#balance = 0;
            },
            MoneyDeposited: (e) => { this.#balance += e.amount; },
            MoneyWithdrawn: (e) => { this.#balance -= e.amount; },
        };

        const handler = handlers[event.type];
        if (!handler) throw new Error(`Unknown event type: ${event.type}`);
        handler(event);
    }

    static reconstituteFromHistory(events) {
        const account = new BankAccountAggregate();
        for (const event of events) {
            account.#when(event);
            account.#version++;
        }
        return account;
    }
}

// Event Store (PostgreSQL-backed)
class EventStore {
    #pool; // pg Pool

    constructor(pool) { this.#pool = pool; }

    async append(aggregateId, events, expectedVersion) {
        const client = await this.#pool.connect();
        try {
            await client.query('BEGIN');

            // Optimistic concurrency
            const { rows } = await client.query(
                'SELECT COALESCE(MAX(version), 0) as ver FROM event_store WHERE aggregate_id = $1',
                [aggregateId]
            );

            if (Number(rows[0].ver) !== expectedVersion) {
                throw new ConcurrencyError(
                    `Expected version ${expectedVersion}, found ${rows[0].ver}`
                );
            }

            for (let i = 0; i < events.length; i++) {
                await client.query(
                    `INSERT INTO event_store (aggregate_id, version, event_type, payload, occurred_at)
                     VALUES ($1, $2, $3, $4, $5)`,
                    [
                        aggregateId,
                        expectedVersion + i + 1,
                        events[i].type,
                        JSON.stringify(events[i]),
                        events[i].occurredAt,
                    ]
                );
            }

            await client.query('COMMIT');
        } catch (err) {
            await client.query('ROLLBACK');
            throw err;
        } finally {
            client.release();
        }
    }

    async getEvents(aggregateId) {
        const { rows } = await this.#pool.query(
            'SELECT event_type, payload FROM event_store WHERE aggregate_id = $1 ORDER BY version',
            [aggregateId]
        );
        return rows.map((r) => JSON.parse(r.payload));
    }
}
```

---

### ২. Message Brokers

#### RabbitMQ

RabbitMQ হলো AMQP (Advanced Message Queuing Protocol) ভিত্তিক একটি message broker। এটি **Exchange** এবং **Queue** concept ব্যবহার করে।

**Exchange Types:**

```
┌─────────────┬──────────────────────────────────────────────────────┐
│ Exchange     │ আচরণ                                                │
├─────────────┼──────────────────────────────────────────────────────┤
│ Direct      │ Routing key exact match করে নির্দিষ্ট queue-তে পাঠায়│
│ Fanout      │ সব bound queue-তে broadcast করে (routing key ignore)│
│ Topic       │ Routing key pattern match (*.order.#)                │
│ Headers     │ Message header attributes দিয়ে match করে             │
└─────────────┴──────────────────────────────────────────────────────┘
```

#### PHP — RabbitMQ Producer/Consumer

```php
<?php

declare(strict_types=1);

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
use PhpAmqpLib\Exchange\AMQPExchangeType;

// Producer — event publish করে
final class RabbitMQEventPublisher
{
    private AMQPStreamConnection $connection;
    private \PhpAmqpLib\Channel\AMQPChannel $channel;

    public function __construct(
        private readonly string $host = 'localhost',
        private readonly int $port = 5672,
    ) {
        $this->connection = new AMQPStreamConnection($this->host, $this->port, 'guest', 'guest');
        $this->channel = $this->connection->channel();

        // Topic exchange — flexible routing
        $this->channel->exchange_declare(
            exchange: 'domain_events',
            type: AMQPExchangeType::TOPIC,
            durable: true,
        );

        // Dead Letter Exchange — failed messages-এর জন্য
        $this->channel->exchange_declare(
            exchange: 'domain_events.dlx',
            type: AMQPExchangeType::FANOUT,
            durable: true,
        );

        $this->channel->queue_declare(
            queue: 'domain_events.dead_letter',
            durable: true,
            arguments: ['x-message-ttl' => ['I', 86400000]], // 24 hour TTL
        );

        $this->channel->queue_bind('domain_events.dead_letter', 'domain_events.dlx');
    }

    public function publish(string $routingKey, object $event): void
    {
        $message = new AMQPMessage(
            body: json_encode([
                'event_type' => $event::class,
                'payload' => $event,
                'metadata' => [
                    'event_id' => Str::uuid()->toString(),
                    'occurred_at' => now()->toIso8601String(),
                    'correlation_id' => CorrelationContext::getId(),
                ],
            ]),
            properties: [
                'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
                'content_type' => 'application/json',
                'message_id' => Str::uuid()->toString(),
            ],
        );

        $this->channel->basic_publish($message, 'domain_events', $routingKey);
    }
}

// Consumer — with retry and dead letter handling
final class RabbitMQEventConsumer
{
    private const int MAX_RETRIES = 3;

    public function consume(string $queueName, callable $handler): void
    {
        // Queue with DLX configuration
        $this->channel->queue_declare(
            queue: $queueName,
            durable: true,
            arguments: [
                'x-dead-letter-exchange' => ['S', 'domain_events.dlx'],
                'x-dead-letter-routing-key' => ['S', $queueName],
            ],
        );

        $this->channel->basic_qos(prefetch_count: 10);

        $this->channel->basic_consume(
            queue: $queueName,
            callback: function (AMQPMessage $msg) use ($handler): void {
                try {
                    $event = json_decode($msg->getBody(), true, flags: JSON_THROW_ON_ERROR);
                    $handler($event);
                    $msg->ack();
                } catch (\Throwable $e) {
                    $retryCount = $this->getRetryCount($msg);

                    if ($retryCount >= self::MAX_RETRIES) {
                        // Dead Letter Queue-তে পাঠাও
                        $msg->nack(requeue: false);
                        Log::error('Message moved to DLQ', [
                            'queue' => $msg->getRoutingKey(),
                            'error' => $e->getMessage(),
                            'retries' => $retryCount,
                        ]);
                    } else {
                        // Requeue with delay
                        $msg->nack(requeue: true);
                    }
                }
            },
        );

        while ($this->channel->is_open()) {
            $this->channel->wait();
        }
    }

    private function getRetryCount(AMQPMessage $msg): int
    {
        $headers = $msg->get('application_headers')?->getNativeData() ?? [];
        return $headers['x-retry-count'] ?? 0;
    }
}
```

#### Node.js — RabbitMQ Producer/Consumer

```javascript
import amqplib from 'amqplib';
import { randomUUID } from 'node:crypto';

class RabbitMQEventPublisher {
    #connection;
    #channel;

    async connect() {
        this.#connection = await amqplib.connect('amqp://localhost');
        this.#channel = await this.#connection.createConfirmChannel();

        // Topic exchange + Dead Letter Exchange
        await this.#channel.assertExchange('domain_events', 'topic', { durable: true });
        await this.#channel.assertExchange('domain_events.dlx', 'fanout', { durable: true });
        await this.#channel.assertQueue('domain_events.dead_letter', {
            durable: true,
            arguments: { 'x-message-ttl': 86_400_000 },
        });
        await this.#channel.bindQueue('domain_events.dead_letter', 'domain_events.dlx');
    }

    async publish(routingKey, event) {
        const message = Buffer.from(JSON.stringify({
            eventType: event.type,
            payload: event,
            metadata: {
                eventId: randomUUID(),
                occurredAt: new Date().toISOString(),
                correlationId: AsyncLocalStorage.getStore()?.correlationId,
            },
        }));

        await new Promise((resolve, reject) => {
            this.#channel.publish('domain_events', routingKey, message, {
                persistent: true,
                contentType: 'application/json',
                messageId: randomUUID(),
            }, (err) => err ? reject(err) : resolve());
        });
    }
}

// Consumer with retry logic
class RabbitMQEventConsumer {
    static #MAX_RETRIES = 3;
    #channel;

    async consume(queueName, handler) {
        await this.#channel.assertQueue(queueName, {
            durable: true,
            arguments: {
                'x-dead-letter-exchange': 'domain_events.dlx',
                'x-dead-letter-routing-key': queueName,
            },
        });

        await this.#channel.prefetch(10);

        await this.#channel.consume(queueName, async (msg) => {
            if (!msg) return;

            try {
                const event = JSON.parse(msg.content.toString());
                await handler(event);
                this.#channel.ack(msg);
            } catch (error) {
                const retryCount = msg.properties.headers?.['x-retry-count'] ?? 0;

                if (retryCount >= RabbitMQEventConsumer.#MAX_RETRIES) {
                    this.#channel.nack(msg, false, false); // DLQ-তে যাবে
                    console.error(`Message moved to DLQ`, { queue: queueName, error: error.message });
                } else {
                    this.#channel.nack(msg, false, true); // requeue
                }
            }
        });
    }
}
```

#### Apache Kafka

Kafka হলো distributed event streaming platform। এটি **Topic**, **Partition**, এবং **Consumer Group** concept ব্যবহার করে। Kafka-তে message দীর্ঘদিন persist থাকে (configurable retention) এবং **replay** করা যায়।

```
Topic: order-events
┌─────────────────────────────────────────┐
│ Partition 0: [e1] [e4] [e7] [e10]      │  ← Consumer Group A, Consumer 1
│ Partition 1: [e2] [e5] [e8] [e11]      │  ← Consumer Group A, Consumer 2
│ Partition 2: [e3] [e6] [e9] [e12]      │  ← Consumer Group A, Consumer 3
└─────────────────────────────────────────┘
    Partition key (orderId) → same partition → ordering guarantee
```

#### Node.js — Kafka (KafkaJS)

```javascript
import { Kafka, Partitioners, logLevel } from 'kafkajs';

const kafka = new Kafka({
    clientId: 'order-service',
    brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
    logLevel: logLevel.WARN,
    retry: { initialRetryTime: 100, retries: 8 },
});

// Producer — ordering guarantee with partition key
class KafkaEventProducer {
    #producer;

    async connect() {
        this.#producer = kafka.producer({
            createPartitioner: Partitioners.DefaultPartitioner,
            idempotent: true, // exactly-once semantics
            maxInFlightRequests: 5,
        });
        await this.#producer.connect();
    }

    async publishOrderEvent(event) {
        await this.#producer.send({
            topic: 'order-events',
            messages: [{
                key: event.orderId,  // same orderId → same partition → ordering
                value: JSON.stringify(event),
                headers: {
                    'event-type': event.type,
                    'schema-version': '1',
                    'correlation-id': event.correlationId ?? randomUUID(),
                },
                timestamp: Date.now().toString(),
            }],
        });
    }

    async publishBatch(events) {
        await this.#producer.sendBatch({
            topicMessages: [{
                topic: 'order-events',
                messages: events.map((e) => ({
                    key: e.orderId,
                    value: JSON.stringify(e),
                    headers: { 'event-type': e.type },
                })),
            }],
        });
    }
}

// Consumer Group — parallel processing with auto-rebalancing
class KafkaEventConsumer {
    #consumer;

    async connect(groupId) {
        this.#consumer = kafka.consumer({
            groupId,
            sessionTimeout: 30_000,
            heartbeatInterval: 3_000,
            maxBytesPerPartition: 1_048_576,
        });

        await this.#consumer.connect();
        await this.#consumer.subscribe({ topics: ['order-events'], fromBeginning: false });
    }

    async startConsuming(handlers) {
        await this.#consumer.run({
            eachMessage: async ({ topic, partition, message, heartbeat }) => {
                const eventType = message.headers['event-type']?.toString();
                const payload = JSON.parse(message.value.toString());

                const handler = handlers.get(eventType);
                if (handler) {
                    await handler(payload);
                }

                await heartbeat(); // দীর্ঘ processing-এ session alive রাখে
            },
        });
    }
}

// ব্যবহার
const consumer = new KafkaEventConsumer();
await consumer.connect('payment-service-group');

await consumer.startConsuming(new Map([
    ['OrderPlaced', async (event) => { await processPayment(event); }],
    ['OrderCancelled', async (event) => { await refundPayment(event); }],
]));
```

#### PHP — Kafka (RdKafka)

```php
<?php

declare(strict_types=1);

final class KafkaEventProducer
{
    private \RdKafka\Producer $producer;
    private \RdKafka\ProducerTopic $topic;

    public function __construct(string $topicName)
    {
        $conf = new \RdKafka\Conf();
        $conf->set('metadata.broker.list', 'kafka-1:9092,kafka-2:9092');
        $conf->set('enable.idempotence', 'true');
        $conf->set('acks', 'all');

        $this->producer = new \RdKafka\Producer($conf);
        $this->topic = $this->producer->newTopic($topicName);
    }

    public function publish(string $key, object $event): void
    {
        $this->topic->producev(
            partition: RD_KAFKA_PARTITION_UA,
            msgflags: 0,
            payload: json_encode($event),
            key: $key,  // partition key — ordering guarantee
            headers: [
                'event-type' => $event::class,
                'schema-version' => '1',
                'correlation-id' => CorrelationContext::getId(),
            ],
        );

        $this->producer->flush(timeoutMs: 5000);
    }
}

final class KafkaEventConsumer
{
    private \RdKafka\KafkaConsumer $consumer;

    public function __construct(string $groupId, array $topics)
    {
        $conf = new \RdKafka\Conf();
        $conf->set('group.id', $groupId);
        $conf->set('metadata.broker.list', 'kafka-1:9092,kafka-2:9092');
        $conf->set('auto.offset.reset', 'earliest');
        $conf->set('enable.auto.commit', 'false'); // manual commit

        $this->consumer = new \RdKafka\KafkaConsumer($conf);
        $this->consumer->subscribe($topics);
    }

    public function consume(array $handlers): never
    {
        while (true) {
            $message = $this->consumer->consume(timeoutMs: 1000);

            match ($message->err) {
                RD_KAFKA_RESP_ERR_NO_ERROR => $this->processMessage($message, $handlers),
                RD_KAFKA_RESP_ERR__PARTITION_EOF => null, // End of partition
                RD_KAFKA_RESP_ERR__TIMED_OUT => null,     // No message
                default => throw new \RuntimeException($message->errstr()),
            };
        }
    }

    private function processMessage(\RdKafka\Message $message, array $handlers): void
    {
        $eventType = $message->headers['event-type'] ?? null;
        $handler = $handlers[$eventType] ?? null;

        if ($handler !== null) {
            $payload = json_decode($message->payload, true);
            ($handler)($payload);
        }

        $this->consumer->commit($message); // manual commit
    }
}
```

---

### ৩. CQRS + Event Sourcing Combined

CQRS (Command Query Responsibility Segregation) event sourcing-এর সাথে মিলিয়ে ব্যবহার করলে write model (command side) event store-এ events সংরক্ষণ করে, আর read model (query side) সেই events থেকে optimized **projection** তৈরি করে।

```
Command Side (Write)                    Query Side (Read)
┌───────────────┐                       ┌────────────────┐
│  Command      │                       │  Query         │
│  Handler      │                       │  Handler       │
└──────┬────────┘                       └──────┬─────────┘
       │                                       │
       ▼                                       ▼
┌──────────────┐    Projection          ┌──────────────┐
│ Event Store  │──────────────────────→│ Read Database │
│ (append only)│    (async event        │ (denormalized)│
└──────────────┘     processing)        └──────────────┘
```

#### PHP — CQRS + Event Sourcing

```php
<?php

declare(strict_types=1);

// Command Side
final readonly class PlaceOrderCommand
{
    public function __construct(
        public string $orderId,
        public string $customerId,
        public array $items,
        public string $shippingAddress,
    ) {}
}

final class PlaceOrderHandler
{
    public function __construct(
        private readonly EventStore $eventStore,
        private readonly EventBus $eventBus,
    ) {}

    public function handle(PlaceOrderCommand $command): void
    {
        $order = OrderAggregate::create(
            orderId: $command->orderId,
            customerId: $command->customerId,
            items: $command->items,
        );

        $this->eventStore->append(
            aggregateId: $command->orderId,
            events: $order->getUncommittedEvents(),
            expectedVersion: 0,
        );

        // Event bus-এ publish করে read model update trigger করে
        foreach ($order->getUncommittedEvents() as $event) {
            $this->eventBus->publish($event);
        }
    }
}

// Read Side — Projection (denormalized read model)
final class OrderProjection
{
    public function __construct(
        private readonly \PDO $readDb,
    ) {}

    // Event listener — event store থেকে event আসলে read model update হয়
    public function onOrderPlaced(OrderPlaced $event): void
    {
        $this->readDb->prepare(
            'INSERT INTO order_summary (order_id, customer_id, status, total, item_count, created_at)
             VALUES (?, ?, ?, ?, ?, ?)'
        )->execute([
            $event->orderId,
            $event->customerId,
            'placed',
            $event->totalAmount,
            count($event->items),
            $event->occurredAt->format('Y-m-d H:i:s'),
        ]);
    }

    public function onOrderShipped(OrderShipped $event): void
    {
        $this->readDb->prepare(
            'UPDATE order_summary SET status = ?, shipped_at = ?, tracking_number = ? WHERE order_id = ?'
        )->execute(['shipped', $event->shippedAt->format('Y-m-d H:i:s'), $event->trackingNumber, $event->orderId]);
    }

    // Rebuild projection from scratch — event replay
    public function rebuild(EventStore $eventStore): void
    {
        $this->readDb->exec('TRUNCATE TABLE order_summary');

        $allEvents = $eventStore->getAllEvents();
        foreach ($allEvents as $event) {
            match ($event::class) {
                OrderPlaced::class => $this->onOrderPlaced($event),
                OrderShipped::class => $this->onOrderShipped($event),
                OrderCancelled::class => $this->onOrderCancelled($event),
                default => null,
            };
        }
    }
}

// Query Side
final class OrderQueryService
{
    public function __construct(
        private readonly \PDO $readDb,
    ) {}

    public function getOrderSummary(string $orderId): ?array
    {
        $stmt = $this->readDb->prepare('SELECT * FROM order_summary WHERE order_id = ?');
        $stmt->execute([$orderId]);
        return $stmt->fetch(\PDO::FETCH_ASSOC) ?: null;
    }

    public function getCustomerOrders(string $customerId, int $limit = 20): array
    {
        $stmt = $this->readDb->prepare(
            'SELECT * FROM order_summary WHERE customer_id = ? ORDER BY created_at DESC LIMIT ?'
        );
        $stmt->execute([$customerId, $limit]);
        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
}
```

#### JavaScript — CQRS + Event Sourcing

```javascript
// Command Side
class PlaceOrderHandler {
    #eventStore;
    #eventBus;

    constructor(eventStore, eventBus) {
        this.#eventStore = eventStore;
        this.#eventBus = eventBus;
    }

    async handle(command) {
        const order = OrderAggregate.create(command);

        await this.#eventStore.append(
            command.orderId,
            order.uncommittedEvents,
            0
        );

        for (const event of order.uncommittedEvents) {
            await this.#eventBus.publish(event);
        }
    }
}

// Read Side — Projection
class OrderProjection {
    #readDb;

    constructor(readDb) { this.#readDb = readDb; }

    async onOrderPlaced(event) {
        await this.#readDb.query(
            `INSERT INTO order_summary (order_id, customer_id, status, total, item_count, created_at)
             VALUES ($1, $2, $3, $4, $5, $6)`,
            [event.orderId, event.customerId, 'placed', event.totalAmount,
             event.items.length, event.occurredAt]
        );
    }

    async onOrderShipped(event) {
        await this.#readDb.query(
            `UPDATE order_summary SET status = $1, shipped_at = $2, tracking_number = $3
             WHERE order_id = $4`,
            ['shipped', event.shippedAt, event.trackingNumber, event.orderId]
        );
    }

    async rebuild(eventStore) {
        await this.#readDb.query('TRUNCATE TABLE order_summary');
        const events = await eventStore.getAllEvents();

        for (const event of events) {
            const handler = {
                OrderPlaced: (e) => this.onOrderPlaced(e),
                OrderShipped: (e) => this.onOrderShipped(e),
                OrderCancelled: (e) => this.onOrderCancelled(e),
            }[event.type];

            if (handler) await handler(event);
        }
    }
}

// Query Service
class OrderQueryService {
    #readDb;

    constructor(readDb) { this.#readDb = readDb; }

    async getCustomerOrders(customerId, limit = 20) {
        const { rows } = await this.#readDb.query(
            'SELECT * FROM order_summary WHERE customer_id = $1 ORDER BY created_at DESC LIMIT $2',
            [customerId, limit]
        );
        return rows;
    }
}
```

---

### ৪. Saga Pattern with Events

Saga Pattern হলো distributed transaction-এর alternative। যেখানে traditional 2PC (Two-Phase Commit) ব্যবহার করা সম্ভব না (microservices-এ), সেখানে Saga pattern ব্যবহার করা হয়। **Choreography-based Saga**-তে প্রতিটি service নিজের কাজ শেষে event publish করে, এবং পরের service সেটা শুনে নিজের কাজ শুরু করে। Failure হলে **compensation event** publish হয়।

```
সফল flow:
OrderCreated → PaymentProcessed → InventoryReserved → ShippingScheduled → OrderCompleted

ব্যর্থ flow (Inventory-তে সমস্যা):
OrderCreated → PaymentProcessed → InventoryFailed
                                       ↓
                               PaymentRefunded ← (compensation)
                                       ↓
                               OrderCancelled  ← (compensation)
```

#### PHP — Choreography Saga (bKash Payment + Pathao Delivery)

```php
<?php

declare(strict_types=1);

// Saga Events
final readonly class OrderCreated
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public string $customerId,
        public float $amount,
        public array $items,
        public DateTimeImmutable $occurredAt = new DateTimeImmutable(),
    ) {}
}

final readonly class PaymentProcessed
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public string $transactionId,
        public float $amount,
    ) {}
}

final readonly class PaymentFailed
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public string $reason,
    ) {}
}

final readonly class InventoryReserved
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public array $reservedItems,
    ) {}
}

final readonly class InventoryReservationFailed
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public string $reason,
    ) {}
}

// Compensation Events
final readonly class PaymentRefunded
{
    public function __construct(
        public string $sagaId,
        public string $orderId,
        public string $originalTransactionId,
        public float $amount,
    ) {}
}

// Payment Service — OrderCreated শুনে payment process করে
final class PaymentSagaHandler
{
    public function __construct(
        private readonly BkashGateway $bkash,
        private readonly MessageBroker $broker,
    ) {}

    public function onOrderCreated(OrderCreated $event): void
    {
        try {
            $result = $this->bkash->executePayment(
                amount: $event->amount,
                customerId: $event->customerId,
                reference: "ORDER-{$event->orderId}",
            );

            $this->broker->publish('saga.payments', new PaymentProcessed(
                sagaId: $event->sagaId,
                orderId: $event->orderId,
                transactionId: $result->transactionId,
                amount: $event->amount,
            ));
        } catch (BkashPaymentException $e) {
            $this->broker->publish('saga.payments', new PaymentFailed(
                sagaId: $event->sagaId,
                orderId: $event->orderId,
                reason: $e->getMessage(),
            ));
        }
    }

    // Compensation — InventoryFailed হলে payment refund
    public function onInventoryReservationFailed(InventoryReservationFailed $event): void
    {
        $sagaState = SagaStateStore::get($event->sagaId);

        $this->bkash->refundPayment($sagaState->transactionId);

        $this->broker->publish('saga.payments', new PaymentRefunded(
            sagaId: $event->sagaId,
            orderId: $event->orderId,
            originalTransactionId: $sagaState->transactionId,
            amount: $sagaState->amount,
        ));
    }
}

// Inventory Service — PaymentProcessed শুনে stock reserve করে
final class InventorySagaHandler
{
    public function __construct(
        private readonly InventoryRepository $inventory,
        private readonly MessageBroker $broker,
    ) {}

    public function onPaymentProcessed(PaymentProcessed $event): void
    {
        $sagaState = SagaStateStore::get($event->sagaId);

        try {
            $reserved = $this->inventory->reserveItems($sagaState->items);

            $this->broker->publish('saga.inventory', new InventoryReserved(
                sagaId: $event->sagaId,
                orderId: $event->orderId,
                reservedItems: $reserved,
            ));
        } catch (OutOfStockException $e) {
            $this->broker->publish('saga.inventory', new InventoryReservationFailed(
                sagaId: $event->sagaId,
                orderId: $event->orderId,
                reason: $e->getMessage(),
            ));
        }
    }
}

// Saga State Tracker — saga-র current state track করে
final class OrderSagaTracker
{
    public function __construct(
        private readonly SagaStateStore $store,
    ) {}

    public function onPaymentProcessed(PaymentProcessed $event): void
    {
        $this->store->update($event->sagaId, [
            'status' => 'payment_completed',
            'transactionId' => $event->transactionId,
        ]);
    }

    public function onPaymentRefunded(PaymentRefunded $event): void
    {
        $this->store->update($event->sagaId, [
            'status' => 'cancelled',
            'cancellationReason' => 'Inventory reservation failed, payment refunded',
        ]);
    }
}
```

#### JavaScript — Choreography Saga

```javascript
// Saga Events (same pattern — Node.js)
class PaymentSagaHandler {
    #bkashGateway;
    #broker;

    constructor(bkashGateway, broker) {
        this.#bkashGateway = bkashGateway;
        this.#broker = broker;
    }

    async onOrderCreated(event) {
        const { sagaId, orderId, customerId, amount } = event;

        try {
            const result = await this.#bkashGateway.executePayment({
                amount,
                customerId,
                reference: `ORDER-${orderId}`,
            });

            await this.#broker.publish('saga.payments', {
                type: 'PaymentProcessed',
                sagaId, orderId,
                transactionId: result.transactionId,
                amount,
            });
        } catch (error) {
            await this.#broker.publish('saga.payments', {
                type: 'PaymentFailed',
                sagaId, orderId,
                reason: error.message,
            });
        }
    }

    async onInventoryReservationFailed(event) {
        const sagaState = await SagaStateStore.get(event.sagaId);
        await this.#bkashGateway.refundPayment(sagaState.transactionId);

        await this.#broker.publish('saga.payments', {
            type: 'PaymentRefunded',
            sagaId: event.sagaId,
            orderId: event.orderId,
            originalTransactionId: sagaState.transactionId,
            amount: sagaState.amount,
        });
    }
}

// Saga State Tracker
class OrderSagaTracker {
    #store;

    constructor(store) { this.#store = store; }

    async onEvent(event) {
        const stateTransitions = {
            OrderCreated: { status: 'initiated' },
            PaymentProcessed: { status: 'payment_completed', transactionId: event.transactionId },
            InventoryReserved: { status: 'inventory_reserved' },
            ShippingScheduled: { status: 'shipping_scheduled' },
            OrderCompleted: { status: 'completed' },
            PaymentFailed: { status: 'failed', reason: event.reason },
            PaymentRefunded: { status: 'cancelled', reason: 'Refunded due to downstream failure' },
        };

        const transition = stateTransitions[event.type];
        if (transition) {
            await this.#store.update(event.sagaId, transition);
        }
    }
}
```

---

### ৫. Event-Driven Microservices Communication

বিভিন্ন bounded context-এর মধ্যে integration event-এর মাধ্যমে যোগাযোগ করলে **schema versioning** অত্যন্ত গুরুত্বপূর্ণ। একটি service-এর event schema পরিবর্তন করলে অন্য সব consumer service ভেঙে যেতে পারে।

#### Event Schema Versioning

```javascript
// Version 1 — initial
const orderPlacedV1 = {
    type: 'OrderPlaced',
    version: 1,
    payload: {
        orderId: 'ORD-123',
        amount: 1500,
        currency: 'BDT',
    },
};

// Version 2 — backward compatible (new field added with default)
const orderPlacedV2 = {
    type: 'OrderPlaced',
    version: 2,
    payload: {
        orderId: 'ORD-123',
        amount: 1500,
        currency: 'BDT',
        discountApplied: false, // নতুন field — default value আছে
    },
};

// Event Upcaster — পুরানো event-কে নতুন version-এ convert করে
class EventUpcaster {
    static upcast(event) {
        const upcasters = {
            OrderPlaced: {
                1: (e) => ({
                    ...e,
                    version: 2,
                    payload: { ...e.payload, discountApplied: false },
                }),
            },
        };

        let currentEvent = { ...event };
        const typeUpcasters = upcasters[currentEvent.type] ?? {};

        while (typeUpcasters[currentEvent.version]) {
            currentEvent = typeUpcasters[currentEvent.version](currentEvent);
        }

        return currentEvent;
    }
}
```

```php
<?php

// PHP — Event Upcaster
final class EventUpcaster
{
    private array $upcasters = [];

    public function register(string $eventType, int $fromVersion, \Closure $transformer): void
    {
        $this->upcasters[$eventType][$fromVersion] = $transformer;
    }

    public function upcast(array $event): array
    {
        $type = $event['type'];
        $version = $event['version'];

        while (isset($this->upcasters[$type][$version])) {
            $event = ($this->upcasters[$type][$version])($event);
            $version = $event['version'];
        }

        return $event;
    }
}

// ব্যবহার
$upcaster = new EventUpcaster();
$upcaster->register('OrderPlaced', 1, fn(array $e) => [
    ...$e,
    'version' => 2,
    'payload' => [...$e['payload'], 'discountApplied' => false],
]);
```

---

### ৬. Real-time Event Processing

#### Laravel Broadcasting + Echo (WebSocket)

```php
<?php

declare(strict_types=1);

// Laravel Event — broadcast করার জন্য ShouldBroadcast implement করতে হয়
final class RideLocationUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly string $rideId,
        public readonly float $latitude,
        public readonly float $longitude,
        public readonly float $speed,
        public readonly string $driverName,
    ) {}

    // Channel — কোন channel-এ broadcast হবে
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("ride.{$this->rideId}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'location.updated';
    }

    public function broadcastWith(): array
    {
        return [
            'rideId' => $this->rideId,
            'lat' => $this->latitude,
            'lng' => $this->longitude,
            'speed' => $this->speed,
            'driver' => $this->driverName,
            'timestamp' => now()->toIso8601String(),
        ];
    }
}

// Controller — ride-sharing app (Pathao-style)
final class RideTrackingController extends Controller
{
    public function updateLocation(
        UpdateLocationRequest $request,
        string $rideId,
    ): JsonResponse {
        $ride = Ride::findOrFail($rideId);

        $ride->update([
            'current_lat' => $request->latitude,
            'current_lng' => $request->longitude,
        ]);

        // Real-time event broadcast
        event(new RideLocationUpdated(
            rideId: $rideId,
            latitude: $request->latitude,
            longitude: $request->longitude,
            speed: $request->speed,
            driverName: $ride->driver->name,
        ));

        return response()->json(['status' => 'ok']);
    }
}
```

#### Node.js — Socket.io Real-time Events

```javascript
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

// Socket.io setup with Redis adapter (multi-instance scaling)
const io = new Server(httpServer, {
    cors: { origin: process.env.CLIENT_URL },
    transports: ['websocket', 'polling'],
});

// Redis adapter — multiple Node.js instance-এ scale করার জন্য
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Authentication middleware
io.use(async (socket, next) => {
    const token = socket.handshake.auth.token;
    try {
        const user = await verifyJWT(token);
        socket.data.user = user;
        next();
    } catch {
        next(new Error('Authentication failed'));
    }
});

// Real-time ride tracking (Pathao-style)
io.on('connection', (socket) => {
    const user = socket.data.user;

    // Rider joins their ride room
    socket.on('ride:join', (rideId) => {
        socket.join(`ride:${rideId}`);
    });

    // Driver sends location updates
    socket.on('driver:location', async (data) => {
        const { rideId, lat, lng, speed } = data;

        // সব rider যারা এই ride track করছে তাদের কাছে broadcast
        io.to(`ride:${rideId}`).emit('location:updated', {
            rideId, lat, lng, speed,
            driver: user.name,
            timestamp: new Date().toISOString(),
        });

        // Analytics service-এও event publish
        await eventBus.publish('ride-events', {
            type: 'DriverLocationUpdated',
            payload: { rideId, lat, lng, speed, driverId: user.id },
        });
    });

    // Real-time notification system
    socket.on('notification:subscribe', () => {
        socket.join(`user:${user.id}:notifications`);
    });
});

// Server-Sent Events (SSE) — simpler alternative
import { Router } from 'express';

const sseRouter = Router();

sseRouter.get('/events/orders/:orderId', authenticate, (req, res) => {
    const { orderId } = req.params;

    res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        Connection: 'keep-alive',
    });

    // Event listener setup
    const handler = (event) => {
        res.write(`event: ${event.type}\n`);
        res.write(`data: ${JSON.stringify(event.payload)}\n`);
        res.write(`id: ${event.id}\n\n`);
    };

    eventEmitter.on(`order:${orderId}`, handler);

    req.on('close', () => {
        eventEmitter.off(`order:${orderId}`, handler);
    });
});
```

---

### ৭. Event Replay and Temporal Queries

Event Sourcing-এর সবচেয়ে শক্তিশালী সুবিধা হলো **event replay**। Production-এ bug পাওয়া গেলে, আপনি নির্দিষ্ট সময়ের events replay করে দেখতে পারেন ঠিক কী ঘটেছিল — এটাকে বলে **time-travel debugging**।

```php
<?php

declare(strict_types=1);

final class EventReplayService
{
    public function __construct(
        private readonly EventStore $eventStore,
        private readonly ProjectionManager $projectionManager,
    ) {}

    // নির্দিষ্ট সময়ে system-এর state কী ছিল — temporal query
    public function getStateAt(string $aggregateId, DateTimeImmutable $pointInTime): object
    {
        $events = $this->eventStore->getEventsUpTo($aggregateId, $pointInTime);
        return BankAccountAggregate::reconstituteFromHistory($events);
    }

    // Bug fix-এর পরে projection rebuild
    public function rebuildProjection(string $projectionName): void
    {
        $projection = $this->projectionManager->get($projectionName);
        $projection->reset();

        $batchSize = 1000;
        $lastPosition = 0;

        while (true) {
            $events = $this->eventStore->getEventsFromPosition($lastPosition, $batchSize);

            if (empty($events)) break;

            foreach ($events as $event) {
                $projection->handle($event);
                $lastPosition = $event->position;
            }
        }

        Log::info("Projection '{$projectionName}' rebuilt up to position {$lastPosition}");
    }

    // Audit trail — নির্দিষ্ট aggregate-এর সম্পূর্ণ ইতিহাস
    public function getAuditTrail(string $aggregateId): array
    {
        $events = $this->eventStore->getEvents($aggregateId);

        return array_map(fn(object $event) => [
            'event_type' => $event::class,
            'occurred_at' => $event->occurredAt->format('Y-m-d H:i:s'),
            'data' => $event,
        ], $events);
    }
}
```

```javascript
// Node.js — Event Replay
class EventReplayService {
    #eventStore;

    constructor(eventStore) { this.#eventStore = eventStore; }

    async getStateAt(aggregateId, pointInTime) {
        const events = await this.#eventStore.getEventsUpTo(aggregateId, pointInTime);
        return BankAccountAggregate.reconstituteFromHistory(events);
    }

    async rebuildProjection(projection) {
        await projection.reset();
        let lastPosition = 0;
        const batchSize = 1000;

        while (true) {
            const events = await this.#eventStore.getEventsFromPosition(lastPosition, batchSize);
            if (events.length === 0) break;

            for (const event of events) {
                await projection.handle(event);
                lastPosition = event.position;
            }
        }

        console.log(`Projection rebuilt to position ${lastPosition}`);
    }

    // bKash transaction audit trail
    async getTransactionAuditTrail(accountId) {
        const events = await this.#eventStore.getEvents(accountId);
        return events.map((e) => ({
            type: e.type,
            occurredAt: e.occurredAt,
            amount: e.amount,
            balance: e.balanceAfter,
            reference: e.reference,
        }));
    }
}
```

---

## 📦 Event Schema Design

### Event Naming Conventions

```
✅ ভালো নামকরণ:
- Past tense (অতীতকাল): OrderPlaced, PaymentCompleted, ItemShipped
- Domain-specific: bKashPaymentReceived, PathaoRideStarted
- Namespace: ecommerce.order.OrderPlaced, payment.bkash.PaymentReceived

❌ খারাপ নামকরণ:
- Imperative: PlaceOrder (এটা Command, Event না)
- Generic: DataUpdated, ThingHappened
- Technical: DatabaseRowInserted, CacheInvalidated
```

### Event Envelope Pattern

প্রতিটি event একটি standard **envelope**-এ wrap করা উচিত যাতে metadata থাকে:

```javascript
// CloudEvents specification compatible envelope
const eventEnvelope = {
    // Required (CloudEvents spec)
    specversion: '1.0',
    id: 'evt_01HQ3K5M7T8X2Y3Z',         // unique event ID
    source: '/services/order-service',     // কোন service থেকে এসেছে
    type: 'com.company.order.OrderPlaced', // fully qualified event type
    time: '2024-01-15T10:30:00.000Z',     // কখন ঘটেছে

    // Optional (CloudEvents extensions)
    datacontenttype: 'application/json',
    dataschema: 'https://schema-registry/OrderPlaced/v2',
    subject: 'ORD-12345',                 // affected resource

    // Custom extensions
    correlationid: 'corr_abc123',          // request tracing
    causationid: 'evt_previous_xyz',       // কোন event-এর কারণে এটি ঘটেছে

    // Actual event data
    data: {
        orderId: 'ORD-12345',
        customerId: 'CUST-789',
        items: [{ sku: 'PROD-001', qty: 2, price: 500 }],
        totalAmount: 1000,
        currency: 'BDT',
    },
};
```

```php
<?php

// PHP — Event Envelope
final readonly class EventEnvelope
{
    public function __construct(
        public string $id,
        public string $source,
        public string $type,
        public string $time,
        public string $specversion = '1.0',
        public string $datacontenttype = 'application/json',
        public ?string $correlationId = null,
        public ?string $causationId = null,
        public array $data = [],
    ) {}

    public static function wrap(object $event, string $source): self
    {
        return new self(
            id: Str::uuid()->toString(),
            source: $source,
            type: $event::class,
            time: now()->toIso8601String(),
            correlationId: CorrelationContext::getId(),
            data: (array) $event,
        );
    }
}
```

---

## ✅ সুবিধা ও ❌ অসুবিধা

```
┌────────────────────────────────┬───────────────────────────────────────┐
│ ✅ সুবিধা (Pros)              │ ❌ অসুবিধা (Cons)                    │
├────────────────────────────────┼───────────────────────────────────────┤
│ Loose coupling — services     │ Eventual consistency — data          │
│ স্বাধীনভাবে deploy/scale করা  │ immediately consistent নয়,           │
│ যায়                           │ debugging কঠিন                       │
├────────────────────────────────┼───────────────────────────────────────┤
│ Scalability — event consumers │ Complexity — event ordering,         │
│ independently scale হয়        │ duplicate handling, schema evolution  │
│                                │ পরিচালনা করতে হয়                     │
├────────────────────────────────┼───────────────────────────────────────┤
│ Resilience — একটি service     │ Debugging difficulty — distributed   │
│ down থাকলেও অন্যগুলো চলে     │ tracing ছাড়া event flow track করা   │
│                                │ অত্যন্ত কঠিন                         │
├────────────────────────────────┼───────────────────────────────────────┤
│ Audit trail — Event Sourcing  │ Event schema management — version    │
│ দিয়ে পুরো ইতিহাস পাওয়া যায়   │ compatibility maintain করা জটিল       │
├────────────────────────────────┼───────────────────────────────────────┤
│ Extensibility — নতুন consumer │ Infrastructure overhead — message    │
│ যোগ করতে producer বদলাতে হয়  │ broker operate করতে DevOps দক্ষতা    │
│ না                             │ প্রয়োজন                               │
├────────────────────────────────┼───────────────────────────────────────┤
│ Real-time processing — event  │ Testing complexity — integration     │
│ আসামাত্র process হয়           │ test করা কঠিন, mocking জটিল          │
└────────────────────────────────┴───────────────────────────────────────┘
```

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Event Storm — অতিরিক্ত event

```javascript
// ❌ খারাপ — প্রতিটি ছোট পরিবর্তনে event
class UserService {
    async updateProfile(userId, data) {
        if (data.name) await emit('UserNameChanged', { userId, name: data.name });
        if (data.email) await emit('UserEmailChanged', { userId, email: data.email });
        if (data.phone) await emit('UserPhoneChanged', { userId, phone: data.phone });
        if (data.address) await emit('UserAddressChanged', { userId, address: data.address });
        // প্রতিটি field-এর জন্য আলাদা event — consumer overwhelmed হয়ে যায়!
    }
}

// ✅ ভালো — meaningful business event
class UserService {
    async updateProfile(userId, data) {
        const updatedFields = Object.keys(data);
        await emit('UserProfileUpdated', {
            userId,
            changes: data,
            updatedFields,
        });
        // একটি event — consumer জানে কী কী বদলেছে
    }
}
```

### ২. Missing Idempotency

Network issue-এর কারণে একই event দুইবার deliver হতে পারে। Idempotency না থাকলে bKash-এ দুইবার টাকা কেটে যেতে পারে!

```php
// ❌ খারাপ — idempotency নেই
final class PaymentHandler
{
    public function handle(OrderPlaced $event): void
    {
        // দুইবার call হলে দুইবার charge হবে!
        $this->gateway->charge($event->amount);
    }
}

// ✅ ভালো — idempotency key ব্যবহার
final class PaymentHandler
{
    public function __construct(
        private readonly IdempotencyStore $idempotencyStore,
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $idempotencyKey = "payment:{$event->orderId}";

        if ($this->idempotencyStore->hasProcessed($idempotencyKey)) {
            Log::info('Duplicate event ignored', ['key' => $idempotencyKey]);
            return;
        }

        $this->gateway->charge($event->amount);
        $this->idempotencyStore->markProcessed($idempotencyKey);
    }
}
```

### ৩. Dual-Write Problem এবং Outbox Pattern

**Dual-write problem:** database-এ data সংরক্ষণ এবং message broker-এ event publish — এই দুটি কাজ atomic নয়। Database save হলো কিন্তু event publish fail হলো, অথবা উল্টোটা। এর সমাধান হলো **Outbox Pattern**।

```php
// ❌ খারাপ — dual-write problem
final class OrderService
{
    public function placeOrder(OrderDTO $dto): void
    {
        $order = Order::create($dto);   // 1. DB write
        $this->broker->publish(          // 2. Broker write — এটা fail হলে?
            new OrderPlaced($order->id)  // DB-তে order আছে কিন্তু event নেই!
        );
    }
}

// ✅ ভালো — Outbox Pattern
final class OrderService
{
    public function placeOrder(OrderDTO $dto): void
    {
        DB::transaction(function () use ($dto) {
            $order = Order::create($dto);

            // Outbox table-এও একই transaction-এ insert
            OutboxMessage::create([
                'aggregate_id' => $order->id,
                'event_type' => 'OrderPlaced',
                'payload' => json_encode(new OrderPlaced($order->id, $order->total)),
                'published' => false,
            ]);
        });
        // এখন দুটোই atomic — হয় দুটোই save হবে, না হলে কোনোটাই না
    }
}

// Background worker — outbox থেকে event publish করে
final class OutboxPublisher
{
    public function run(): void
    {
        $messages = OutboxMessage::where('published', false)
            ->orderBy('created_at')
            ->limit(100)
            ->get();

        foreach ($messages as $message) {
            try {
                $this->broker->publish($message->event_type, json_decode($message->payload));
                $message->update(['published' => true, 'published_at' => now()]);
            } catch (\Throwable $e) {
                Log::error('Outbox publish failed', ['id' => $message->id]);
            }
        }
    }
}
```

```javascript
// Node.js — Outbox Pattern
class OrderService {
    async placeOrder(dto) {
        const client = await pool.connect();
        try {
            await client.query('BEGIN');

            const { rows: [order] } = await client.query(
                'INSERT INTO orders (customer_id, total) VALUES ($1, $2) RETURNING *',
                [dto.customerId, dto.total]
            );

            // Outbox — same transaction
            await client.query(
                `INSERT INTO outbox (aggregate_id, event_type, payload)
                 VALUES ($1, $2, $3)`,
                [order.id, 'OrderPlaced', JSON.stringify({ orderId: order.id, total: order.total })]
            );

            await client.query('COMMIT');
        } catch (err) {
            await client.query('ROLLBACK');
            throw err;
        } finally {
            client.release();
        }
    }
}

// Outbox Poller
class OutboxPublisher {
    async poll() {
        const { rows: messages } = await pool.query(
            'SELECT * FROM outbox WHERE published = false ORDER BY created_at LIMIT 100'
        );

        for (const msg of messages) {
            try {
                await this.broker.publish(msg.event_type, JSON.parse(msg.payload));
                await pool.query('UPDATE outbox SET published = true WHERE id = $1', [msg.id]);
            } catch (err) {
                console.error(`Outbox publish failed: ${msg.id}`, err);
            }
        }
    }
}
```

### ৪. Event Ordering Issues

```javascript
// ❌ সমস্যা — event ভুল ক্রমে process হলে
// OrderShipped আগে process হলো, OrderPlaced পরে — state ভুল হয়ে যায়

// ✅ সমাধান — ordering guarantee
// 1. Kafka: same partition key → same partition → ordered
// 2. Version number: event-এ sequence number রাখা
// 3. Causal ordering: causation_id দিয়ে dependency track করা

class OrderedEventProcessor {
    #lastProcessedVersion = new Map();

    async process(event) {
        const { aggregateId, version } = event;
        const lastVersion = this.#lastProcessedVersion.get(aggregateId) ?? 0;

        if (version <= lastVersion) {
            return; // duplicate বা out-of-order — skip
        }

        if (version !== lastVersion + 1) {
            // Gap detected — missing event, wait or fetch
            await this.#fetchMissingEvents(aggregateId, lastVersion + 1, version - 1);
        }

        await this.#handle(event);
        this.#lastProcessedVersion.set(aggregateId, version);
    }
}
```

---

## 🧪 টেস্টিং EDA

### Unit Testing Event Handlers

#### PHP (PHPUnit)

```php
<?php

declare(strict_types=1);

final class PaymentHandlerTest extends TestCase
{
    public function test_payment_processed_on_order_placed(): void
    {
        $gateway = $this->createMock(PaymentGateway::class);
        $broker = $this->createMock(MessageBroker::class);

        $gateway->expects($this->once())
            ->method('charge')
            ->with('CUST-1', 1500.0)
            ->willReturn(new PaymentResult(success: true, txnId: 'TXN-123'));

        $broker->expects($this->once())
            ->method('publish')
            ->with('saga.payments', $this->callback(
                fn($event) => $event instanceof PaymentProcessed
                    && $event->transactionId === 'TXN-123'
            ));

        $handler = new PaymentSagaHandler($gateway, $broker);
        $handler->onOrderCreated(new OrderCreated(
            sagaId: 'SAGA-1',
            orderId: 'ORD-1',
            customerId: 'CUST-1',
            amount: 1500.0,
            items: [],
        ));
    }

    public function test_idempotent_handler_ignores_duplicate(): void
    {
        $store = new InMemoryIdempotencyStore();
        $gateway = $this->createMock(PaymentGateway::class);

        $handler = new PaymentHandler($store, $gateway);
        $event = new OrderPlaced(orderId: 'ORD-1', amount: 1000);

        $handler->handle($event); // প্রথমবার — process হবে
        $handler->handle($event); // দ্বিতীয়বার — ignore হবে

        $gateway->expects($this->once())->method('charge'); // শুধু একবার call হবে
    }
}

// Event Sourcing Aggregate টেস্ট
final class BankAccountAggregateTest extends TestCase
{
    public function test_deposit_increases_balance(): void
    {
        $account = BankAccountAggregate::open('ACC-1', 'রহিম');
        $account->deposit(5000.0, 'bKash-TXN-001');

        $this->assertSame(5000.0, $account->getBalance());
        $this->assertCount(2, $account->getUncommittedEvents()); // Open + Deposit
    }

    public function test_withdraw_fails_with_insufficient_funds(): void
    {
        $account = BankAccountAggregate::open('ACC-1', 'করিম');
        $account->deposit(1000.0, 'REF-1');

        $this->expectException(InsufficientFundsException::class);
        $account->withdraw(2000.0, 'REF-2');
    }

    public function test_reconstitute_from_history(): void
    {
        $events = [
            new AccountOpened('ACC-1', 'রহিম'),
            new MoneyDeposited('ACC-1', 5000.0, 'REF-1'),
            new MoneyWithdrawn('ACC-1', 2000.0, 'REF-2'),
        ];

        $account = BankAccountAggregate::reconstituteFromHistory($events);
        $this->assertSame(3000.0, $account->getBalance());
    }
}
```

#### JavaScript (Jest)

```javascript
describe('PaymentSagaHandler', () => {
    test('processes payment on OrderCreated', async () => {
        const gateway = { executePayment: jest.fn().mockResolvedValue({ transactionId: 'TXN-1' }) };
        const broker = { publish: jest.fn().mockResolvedValue(undefined) };

        const handler = new PaymentSagaHandler(gateway, broker);
        await handler.onOrderCreated({
            sagaId: 'SAGA-1', orderId: 'ORD-1', customerId: 'C-1', amount: 1500,
        });

        expect(gateway.executePayment).toHaveBeenCalledWith(
            expect.objectContaining({ amount: 1500, customerId: 'C-1' })
        );
        expect(broker.publish).toHaveBeenCalledWith(
            'saga.payments',
            expect.objectContaining({ type: 'PaymentProcessed', transactionId: 'TXN-1' })
        );
    });
});

describe('BankAccountAggregate', () => {
    test('deposit increases balance', () => {
        const account = BankAccountAggregate.open('ACC-1', 'রহিম');
        account.deposit(5000, 'bKash-TXN-001');
        expect(account.balance).toBe(5000);
    });

    test('withdraw throws on insufficient funds', () => {
        const account = BankAccountAggregate.open('ACC-1', 'করিম');
        account.deposit(1000, 'REF-1');
        expect(() => account.withdraw(2000, 'REF-2')).toThrow(InsufficientFundsError);
    });

    test('reconstitutes from event history', () => {
        const events = [
            new AccountOpened('ACC-1', 'রহিম'),
            new MoneyDeposited('ACC-1', 5000, 'REF-1'),
            new MoneyWithdrawn('ACC-1', 2000, 'REF-2'),
        ];
        const account = BankAccountAggregate.reconstituteFromHistory(events);
        expect(account.balance).toBe(3000);
    });
});
```

### Integration Testing with Test Broker

```javascript
// In-memory broker for integration tests
class InMemoryEventBroker {
    #handlers = new Map();
    #publishedEvents = [];

    subscribe(eventType, handler) {
        if (!this.#handlers.has(eventType)) this.#handlers.set(eventType, []);
        this.#handlers.get(eventType).push(handler);
    }

    async publish(channel, event) {
        this.#publishedEvents.push({ channel, event });
        const handlers = this.#handlers.get(event.type) ?? [];
        for (const handler of handlers) {
            await handler(event);
        }
    }

    getPublishedEvents(type) {
        return this.#publishedEvents.filter((e) => e.event.type === type);
    }

    reset() {
        this.#publishedEvents = [];
        this.#handlers.clear();
    }
}

// Integration test
describe('Order Saga Integration', () => {
    let broker;

    beforeEach(() => { broker = new InMemoryEventBroker(); });

    test('complete order saga flows correctly', async () => {
        const paymentHandler = new PaymentSagaHandler(mockGateway, broker);
        const inventoryHandler = new InventorySagaHandler(mockInventory, broker);

        broker.subscribe('OrderCreated', (e) => paymentHandler.onOrderCreated(e));
        broker.subscribe('PaymentProcessed', (e) => inventoryHandler.onPaymentProcessed(e));

        await broker.publish('orders', {
            type: 'OrderCreated',
            sagaId: 'S-1', orderId: 'O-1', customerId: 'C-1', amount: 500, items: [],
        });

        // Eventually consistent — সব step complete হওয়া পর্যন্ত wait
        await new Promise((r) => setTimeout(r, 100));

        expect(broker.getPublishedEvents('PaymentProcessed')).toHaveLength(1);
        expect(broker.getPublishedEvents('InventoryReserved')).toHaveLength(1);
    });
});
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

- **Loose coupling প্রয়োজন** — microservices-এর মধ্যে সরাসরি dependency কমাতে চান
- **Real-time processing** — bKash transaction notification, ride-sharing live tracking
- **Audit trail দরকার** — financial system-এ প্রতিটি পরিবর্তনের ইতিহাস রাখতে হবে
- **Scalability** — বিভিন্ন service আলাদা আলাদা scale করতে হবে
- **Multiple consumers** — একটি event-এ অনেক service আগ্রহী (notification, analytics, billing)
- **Async processing** — দীর্ঘ সময়ের কাজ background-এ করতে চান

### ❌ ব্যবহার করবেন না যখন:

- **Simple CRUD application** — ছোট monolithic app-এ EDA unnecessary complexity যোগ করে
- **Strong consistency প্রয়োজন** — bank balance check-এ eventual consistency চলবে না
- **Request-response pattern** — synchronous response দরকার হলে EDA overhead
- **Small team** — distributed system operate করার দক্ষতা নেই
- **Low event volume** — দিনে ১০০টি event-এর জন্য Kafka চালানো overkill

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

```
┌────────────────────────────┬────────────────────────────────────────┐
│ প্যাটার্ন                   │ EDA-র সাথে সম্পর্ক                     │
├────────────────────────────┼────────────────────────────────────────┤
│ CQRS                       │ পরিপূরক — EDA read/write model আলাদা  │
│                            │ করতে সাহায্য করে                       │
├────────────────────────────┼────────────────────────────────────────┤
│ Event Sourcing             │ সহচর — EDA-র সাথে naturally fit করে,  │
│                            │ event store থেকে events publish হয়    │
├────────────────────────────┼────────────────────────────────────────┤
│ Saga Pattern               │ নির্ভরশীল — distributed transaction   │
│                            │ EDA events দিয়ে orchestrate করে        │
├────────────────────────────┼────────────────────────────────────────┤
│ Microservices              │ সক্ষমকারী — EDA microservices-এর      │
│                            │ মধ্যে async communication enable করে   │
├────────────────────────────┼────────────────────────────────────────┤
│ Observer Pattern           │ ভিত্তি — EDA হলো distributed Observer │
│                            │ pattern-এর বড় পরিসরে প্রয়োগ          │
├────────────────────────────┼────────────────────────────────────────┤
│ Pub/Sub                    │ উপাদান — EDA-র messaging layer        │
│                            │ হিসেবে Pub/Sub ব্যবহৃত হয়              │
├────────────────────────────┼────────────────────────────────────────┤
│ Domain-Driven Design       │ অংশীদার — DDD-র Domain Events EDA-র  │
│                            │ fundamental building block              │
└────────────────────────────┴────────────────────────────────────────┘
```

---

## 📋 সারসংক্ষেপ

Event-Driven Architecture আধুনিক distributed system-এর ভিত্তিপ্রস্তর। এটি services-কে loosely coupled রাখে, independent scaling সম্ভব করে, এবং real-time processing enable করে। তবে এটি significant complexity নিয়ে আসে — eventual consistency, event ordering, schema evolution, idempotency — এগুলো সঠিকভাবে handle না করলে system unreliable হয়ে যায়।

**মূল শিক্ষা:**

1. **Event হলো immutable fact** — অতীতে যা ঘটেছে তার রেকর্ড, পরিবর্তনযোগ্য নয়
2. **Idempotency সবসময়** — প্রতিটি event handler idempotent হতে হবে, কারণ duplicate delivery হতে পারে
3. **Outbox Pattern ব্যবহার করুন** — dual-write problem এড়াতে, database আর broker-এ একসাথে write নয়
4. **Schema versioning থেকে শুরু করুন** — backward compatibility maintain না করলে পুরো system ভেঙে যাবে
5. **Correlation ID সবসময় রাখুন** — distributed tracing ছাড়া production debugging অসম্ভব
6. **Mediator topology** — complex orchestration দরকার হলে; **Broker topology** — simple event chain হলে
7. **Event Sourcing বিবেচনা করুন** — যেখানে audit trail, temporal query, বা state rebuild প্রয়োজন

> **"বড় সিস্টেম synchronous coupling-এ ভেঙে পড়ে। Event-Driven Architecture সেই coupling ভাঙার সবচেয়ে কার্যকর উপায় — তবে এটি একটি trade-off, কোনো silver bullet নয়।"**
