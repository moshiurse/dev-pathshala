# 🔄 State প্যাটার্ন (State Design Pattern)

## 📌 সংজ্ঞা

> **GoF সংজ্ঞা:** "Allow an object to alter its behavior when its internal state changes. The object will appear to change its class."
>
> একটি অবজেক্টকে তার অভ্যন্তরীণ state পরিবর্তনের সাথে সাথে নিজের behavior পরিবর্তন করতে দিন। বাইরে থেকে মনে হবে অবজেক্টের ক্লাসই যেন পরিবর্তন হয়ে গেছে।

State প্যাটার্ন হলো একটি **Behavioral Design Pattern** যেটি কোনো অবজেক্টের বিভিন্ন state-কে আলাদা আলাদা ক্লাসে encapsulate করে। প্রতিটি state ক্লাস সেই নির্দিষ্ট state-এ অবজেক্টের behavior define করে। যখন অবজেক্টের state পরিবর্তন হয়, তখন অবজেক্ট নিজেই যেন একটি ভিন্ন ক্লাসের ইনস্ট্যান্সে রূপান্তরিত হয়েছে — এমনটি মনে হয়।

### মূল ধারণা

State প্যাটার্নের পেছনের মূল ধারণা হলো **Finite State Machine (FSM)**। যেকোনো সময়ে একটি অবজেক্ট সসীম সংখ্যক state-এর মধ্যে যেকোনো একটিতে থাকতে পারে। প্রতিটি state-এ অবজেক্টের behavior আলাদা এবং একটি state থেকে আরেকটি state-এ যাওয়ার নির্দিষ্ট নিয়ম (transition rules) থাকে।

### প্রচলিত সমস্যা — কেন State প্যাটার্ন দরকার?

ধরুন, আপনি একটি e-commerce অর্ডার সিস্টেম বানাচ্ছেন। অর্ডারের বিভিন্ন অবস্থা আছে: Pending, Confirmed, Processing, Shipped, Delivered, Cancelled। প্রতিটি অবস্থায় অর্ডারের উপর বিভিন্ন action আলাদা ভাবে কাজ করে। `if/else` বা `switch` দিয়ে এটা handle করলে কোড দ্রুত অপঠনীয় ও unmaintainable হয়ে যায়:

```php
// ❌ খারাপ পদ্ধতি — এভাবে করবেন না!
class Order {
    public function cancel(): void {
        if ($this->status === 'pending') {
            $this->status = 'cancelled';
        } elseif ($this->status === 'confirmed') {
            $this->status = 'cancelled';
            $this->refundPayment();
        } elseif ($this->status === 'shipped') {
            throw new Exception('Shipped অর্ডার cancel করা যায় না');
        }
        // আরো state যোগ হলে এই if/else বাড়তেই থাকবে...
    }
}
```

State প্যাটার্ন এই সমস্যার চমৎকার সমাধান দেয়।

---

## 🏠 বাস্তব উদাহরণ

### ট্রাফিক লাইট সিস্টেম 🚦

ট্রাফিক লাইট State প্যাটার্নের সবচেয়ে স্বাভাবিক উদাহরণ:

- **🔴 লাল (Red):** গাড়ি থামবে। পরবর্তী state → সবুজ
- **🟡 হলুদ (Yellow):** গাড়ি ধীর হবে / প্রস্তুত হবে। পরবর্তী state → লাল
- **🟢 সবুজ (Green):** গাড়ি চলবে। পরবর্তী state → হলুদ

প্রতিটি রঙে (state-এ) ট্রাফিক লাইটের behavior সম্পূর্ণ আলাদা। কিন্তু ট্রাফিক লাইট একটাই — শুধু তার অভ্যন্তরীণ state পরিবর্তন হচ্ছে।

### বাংলাদেশ context: bKash লেনদেন 💰

bKash-এ টাকা পাঠানোর সময় লেনদেনের বিভিন্ন state থাকে:

- **Initiated:** ব্যবহারকারী Send Money-তে ক্লিক করেছে
- **PIN Verification:** PIN দেওয়া হচ্ছে / যাচাই করা হচ্ছে
- **Processing:** ব্যাকএন্ডে লেনদেন প্রসেস হচ্ছে
- **Success:** টাকা সফলভাবে পাঠানো হয়েছে
- **Failed:** লেনদেন ব্যর্থ হয়েছে (ব্যালেন্স কম, সার্ভার সমস্যা)
- **Reversed:** ব্যর্থ লেনদেনে টাকা ফেরত দেওয়া হয়েছে

প্রতিটি state-এ সিস্টেমের behavior আলাদা — Failed state-এ retry করা যায়, Success state-এ receipt দেখানো হয়।

### ডেলিভারি ট্র্যাকিং (Pathao/Foodpanda) 🛵

- **Order Placed** → **Restaurant Accepted** → **Preparing** → **Rider Assigned** → **Picked Up** → **On The Way** → **Delivered**

---

## 📊 UML ডায়াগ্রাম

### ক্লাস ডায়াগ্রাম (ASCII)

```
┌─────────────────────────┐         ┌──────────────────────────┐
│        Context           │         │    <<interface>>          │
│─────────────────────────│         │       State               │
│ - state: State           │────────>│──────────────────────────│
│─────────────────────────│         │ + handle(ctx): void       │
│ + setState(s: State)     │         │ + getStatus(): string     │
│ + request(): void        │         └──────────┬───────────────┘
│   // state.handle(this)  │                     │
└─────────────────────────┘                     │
                                    ┌───────────┼───────────┐
                                    │           │           │
                              ┌─────┴─────┐ ┌──┴────┐ ┌───┴──────┐
                              │ConcreteA   │ │Concr B│ │Concrete C│
                              │───────────│ │───────│ │──────────│
                              │+handle()  │ │+handle│ │+handle() │
                              │+getStatus │ │+getSta│ │+getStatus│
                              └───────────┘ └───────┘ └──────────┘
```

### State Machine ডায়াগ্রাম — অর্ডার সিস্টেম

```
                    ┌──────────┐
        ┌──────────>│ CANCELLED│
        │           └──────────┘
        │                ^
        │                │ cancel()
        │                │
   ┌────┴───┐     ┌─────┴─────┐     ┌───────────┐
   │PENDING │────>│ CONFIRMED │────>│PROCESSING │
   └────────┘     └───────────┘     └─────┬─────┘
    confirm()      process()               │
                                    ship() │
                                           v
                                    ┌──────┴──┐     ┌───────────┐
                                    │ SHIPPED  │────>│ DELIVERED │
                                    └─────────┘     └───────────┘
                                     deliver()
```

### bKash Transaction State Machine

```
  ┌──────────┐  verify  ┌────────────┐  process  ┌────────────┐
  │INITIATED │────────> │ VERIFYING  │─────────> │ PROCESSING │
  └──────────┘          └─────┬──────┘           └──┬────┬────┘
                              │ fail                │    │ fail
                              v                     │    v
                        ┌──────────┐          success  ┌────────┐
                        │ FAILED   │<─────────────────│ FAILED  │
                        └────┬─────┘           │       └───┬────┘
                             │                 v           │
                             │ reverse   ┌─────────┐      │ reverse
                             v           │ SUCCESS │      v
                        ┌──────────┐     └─────────┘ ┌──────────┐
                        │ REVERSED │                  │ REVERSED │
                        └──────────┘                  └──────────┘
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic State Pattern — ট্রাফিক লাইট

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// State ইন্টারফেস — সব state ক্লাস এটা implement করবে
interface TrafficLightState
{
    public function handle(TrafficLight $light): void;
    public function getColor(): string;
    public function canCross(): bool;
}

// Concrete State: লাল বাতি
class RedState implements TrafficLightState
{
    public function handle(TrafficLight $light): void
    {
        echo "🔴 লাল বাতি — সবাই থামুন! ৬০ সেকেন্ড অপেক্ষা করুন।\n";
        $light->setState(new GreenState());
    }

    public function getColor(): string { return 'RED'; }
    public function canCross(): bool { return false; }
}

// Concrete State: সবুজ বাতি
class GreenState implements TrafficLightState
{
    public function handle(TrafficLight $light): void
    {
        echo "🟢 সবুজ বাতি — যানবাহন চলতে পারে। ৪৫ সেকেন্ড।\n";
        $light->setState(new YellowState());
    }

    public function getColor(): string { return 'GREEN'; }
    public function canCross(): bool { return true; }
}

// Concrete State: হলুদ বাতি
class YellowState implements TrafficLightState
{
    public function handle(TrafficLight $light): void
    {
        echo "🟡 হলুদ বাতি — সতর্ক! গতি কমান। ১৫ সেকেন্ড।\n";
        $light->setState(new RedState());
    }

    public function getColor(): string { return 'YELLOW'; }
    public function canCross(): bool { return false; }
}

// Context ক্লাস
class TrafficLight
{
    public function __construct(
        private TrafficLightState $state = new RedState()
    ) {}

    public function setState(TrafficLightState $state): void
    {
        $this->state = $state;
    }

    public function change(): void
    {
        $this->state->handle($this);
    }

    public function getColor(): string
    {
        return $this->state->getColor();
    }
}

// ব্যবহার
$light = new TrafficLight();
for ($i = 0; $i < 6; $i++) {
    echo "বর্তমান: {$light->getColor()} | ";
    $light->change();
}
```

#### JavaScript (ES2022+)

```javascript
// State ইন্টারফেস — JS-তে informal contract
class TrafficLightState {
  handle(light) { throw new Error('Subclass must implement handle()'); }
  getColor() { throw new Error('Subclass must implement getColor()'); }
  get canCross() { throw new Error('Subclass must implement canCross'); }
}

class RedState extends TrafficLightState {
  handle(light) {
    console.log('🔴 লাল বাতি — সবাই থামুন! ৬০ সেকেন্ড অপেক্ষা করুন।');
    light.setState(new GreenState());
  }
  getColor() { return 'RED'; }
  get canCross() { return false; }
}

class GreenState extends TrafficLightState {
  handle(light) {
    console.log('🟢 সবুজ বাতি — যানবাহন চলতে পারে। ৪৫ সেকেন্ড।');
    light.setState(new YellowState());
  }
  getColor() { return 'GREEN'; }
  get canCross() { return true; }
}

class YellowState extends TrafficLightState {
  handle(light) {
    console.log('🟡 হলুদ বাতি — সতর্ক! গতি কমান। ১৫ সেকেন্ড।');
    light.setState(new RedState());
  }
  getColor() { return 'YELLOW'; }
  get canCross() { return false; }
}

class TrafficLight {
  #state;

  constructor(initialState = new RedState()) {
    this.#state = initialState;
  }

  setState(state) { this.#state = state; }
  change() { this.#state.handle(this); }
  getColor() { return this.#state.getColor(); }
}

// ব্যবহার
const light = new TrafficLight();
for (let i = 0; i < 6; i++) {
  console.log(`বর্তমান: ${light.getColor()} |`);
  light.change();
}
```

---

### 2️⃣ Order State Machine — সম্পূর্ণ ই-কমার্স অর্ডার সিস্টেম

#### PHP 8.3 — Full Implementation

```php
<?php

declare(strict_types=1);

// PHP 8.1+ Enum দিয়ে state identifier
enum OrderStatus: string
{
    case Pending    = 'pending';
    case Confirmed  = 'confirmed';
    case Processing = 'processing';
    case Shipped    = 'shipped';
    case Delivered  = 'delivered';
    case Cancelled  = 'cancelled';
}

// State ইন্টারফেস — প্রতিটি possible action define করে
interface OrderState
{
    public function confirm(Order $order): void;
    public function process(Order $order): void;
    public function ship(Order $order): void;
    public function deliver(Order $order): void;
    public function cancel(Order $order): void;
    public function getStatus(): OrderStatus;
}

// Base State — default behavior (exception throw) দেয়
abstract class BaseOrderState implements OrderState
{
    public function confirm(Order $order): void
    {
        throw new InvalidStateTransitionException(
            "'{$this->getStatus()->value}' state থেকে confirm করা যায় না"
        );
    }

    public function process(Order $order): void
    {
        throw new InvalidStateTransitionException(
            "'{$this->getStatus()->value}' state থেকে process করা যায় না"
        );
    }

    public function ship(Order $order): void
    {
        throw new InvalidStateTransitionException(
            "'{$this->getStatus()->value}' state থেকে ship করা যায় না"
        );
    }

    public function deliver(Order $order): void
    {
        throw new InvalidStateTransitionException(
            "'{$this->getStatus()->value}' state থেকে deliver করা যায় না"
        );
    }

    public function cancel(Order $order): void
    {
        throw new InvalidStateTransitionException(
            "'{$this->getStatus()->value}' state থেকে cancel করা যায় না"
        );
    }
}

class InvalidStateTransitionException extends \RuntimeException {}

// ─── Concrete States ───

class PendingState extends BaseOrderState
{
    public function confirm(Order $order): void
    {
        echo "✅ অর্ডার confirm হচ্ছে। পেমেন্ট যাচাই সম্পন্ন।\n";
        $order->setState(new ConfirmedState());
    }

    public function cancel(Order $order): void
    {
        echo "❌ Pending অর্ডার বাতিল করা হলো। কোনো চার্জ নেই।\n";
        $order->setState(new CancelledState());
    }

    public function getStatus(): OrderStatus { return OrderStatus::Pending; }
}

class ConfirmedState extends BaseOrderState
{
    public function process(Order $order): void
    {
        echo "⚙️ অর্ডার processing শুরু। গুদাম থেকে পণ্য সংগ্রহ হচ্ছে।\n";
        $order->setState(new ProcessingState());
    }

    public function cancel(Order $order): void
    {
        echo "❌ Confirmed অর্ডার বাতিল। পেমেন্ট refund প্রক্রিয়া শুরু।\n";
        $order->setState(new CancelledState());
        $order->initiateRefund();
    }

    public function getStatus(): OrderStatus { return OrderStatus::Confirmed; }
}

class ProcessingState extends BaseOrderState
{
    public function ship(Order $order): void
    {
        if (!$order->hasTrackingNumber()) {
            throw new \LogicException('Tracking number ছাড়া ship করা যাবে না');
        }
        echo "🚚 অর্ডার shipped! ট্র্যাকিং: {$order->getTrackingNumber()}\n";
        $order->setState(new ShippedState());
    }

    public function getStatus(): OrderStatus { return OrderStatus::Processing; }
}

class ShippedState extends BaseOrderState
{
    public function deliver(Order $order): void
    {
        echo "📦 অর্ডার সফলভাবে ডেলিভারি হয়েছে!\n";
        $order->setState(new DeliveredState());
    }

    public function getStatus(): OrderStatus { return OrderStatus::Shipped; }
}

class DeliveredState extends BaseOrderState
{
    public function getStatus(): OrderStatus { return OrderStatus::Delivered; }
    // Delivered state-এ কোনো transition নেই — এটা terminal state
}

class CancelledState extends BaseOrderState
{
    public function getStatus(): OrderStatus { return OrderStatus::Cancelled; }
    // Cancelled-ও terminal state
}

// ─── Context: Order ───

class Order
{
    private OrderState $state;
    private array $history = [];
    private ?string $trackingNumber = null;

    public function __construct(
        private readonly string $orderId,
        private readonly float $amount,
    ) {
        $this->state = new PendingState();
        $this->logTransition('Order তৈরি হয়েছে');
    }

    public function setState(OrderState $state): void
    {
        $previousStatus = $this->state->getStatus()->value;
        $this->state = $state;
        $this->logTransition("{$previousStatus} → {$state->getStatus()->value}");
    }

    public function confirm(): void { $this->state->confirm($this); }
    public function process(): void { $this->state->process($this); }
    public function ship(): void { $this->state->ship($this); }
    public function deliver(): void { $this->state->deliver($this); }
    public function cancel(): void { $this->state->cancel($this); }

    public function getStatus(): string { return $this->state->getStatus()->value; }

    public function setTrackingNumber(string $tracking): void
    {
        $this->trackingNumber = $tracking;
    }

    public function getTrackingNumber(): ?string { return $this->trackingNumber; }
    public function hasTrackingNumber(): bool { return $this->trackingNumber !== null; }

    public function initiateRefund(): void
    {
        echo "💰 ৳{$this->amount} refund প্রক্রিয়া শুরু হয়েছে।\n";
    }

    private function logTransition(string $message): void
    {
        $this->history[] = [
            'time' => date('H:i:s'),
            'message' => $message,
            'status' => $this->state->getStatus()->value,
        ];
    }

    public function getHistory(): array { return $this->history; }
}

// ─── ব্যবহার ───

$order = new Order('ORD-2024-001', 2500.00);
echo "Status: {$order->getStatus()}\n";  // pending

$order->confirm();
echo "Status: {$order->getStatus()}\n";  // confirmed

$order->process();
$order->setTrackingNumber('BD-TRACK-98765');
$order->ship();
$order->deliver();

echo "\n📜 অর্ডার হিস্ট্রি:\n";
foreach ($order->getHistory() as $entry) {
    echo "[{$entry['time']}] {$entry['message']} ({$entry['status']})\n";
}

// Invalid transition test
try {
    $order->cancel(); // Delivered অর্ডার cancel হবে না!
} catch (InvalidStateTransitionException $e) {
    echo "⚠️ ত্রুটি: {$e->getMessage()}\n";
}
```

#### JavaScript (ES2022+)

```javascript
// State identifier
const OrderStatus = Object.freeze({
  PENDING: 'pending',
  CONFIRMED: 'confirmed',
  PROCESSING: 'processing',
  SHIPPED: 'shipped',
  DELIVERED: 'delivered',
  CANCELLED: 'cancelled',
});

class InvalidStateTransition extends Error {
  constructor(message) {
    super(message);
    this.name = 'InvalidStateTransition';
  }
}

class BaseOrderState {
  confirm(order)  { this.#deny('confirm'); }
  process(order)  { this.#deny('process'); }
  ship(order)     { this.#deny('ship'); }
  deliver(order)  { this.#deny('deliver'); }
  cancel(order)   { this.#deny('cancel'); }
  get status()    { throw new Error('Subclass must define status'); }

  #deny(action) {
    throw new InvalidStateTransition(
      `'${this.status}' state থেকে ${action} করা যায় না`
    );
  }
}

class PendingState extends BaseOrderState {
  get status() { return OrderStatus.PENDING; }

  confirm(order) {
    console.log('✅ অর্ডার confirm হচ্ছে।');
    order.setState(new ConfirmedState());
  }

  cancel(order) {
    console.log('❌ Pending অর্ডার বাতিল।');
    order.setState(new CancelledState());
  }
}

class ConfirmedState extends BaseOrderState {
  get status() { return OrderStatus.CONFIRMED; }

  process(order) {
    console.log('⚙️ অর্ডার processing শুরু।');
    order.setState(new ProcessingState());
  }

  cancel(order) {
    console.log('❌ Confirmed অর্ডার বাতিল। Refund শুরু।');
    order.setState(new CancelledState());
    order.initiateRefund();
  }
}

class ProcessingState extends BaseOrderState {
  get status() { return OrderStatus.PROCESSING; }

  ship(order) {
    if (!order.trackingNumber) {
      throw new Error('Tracking number ছাড়া ship করা যাবে না');
    }
    console.log(`🚚 Shipped! ট্র্যাকিং: ${order.trackingNumber}`);
    order.setState(new ShippedState());
  }
}

class ShippedState extends BaseOrderState {
  get status() { return OrderStatus.SHIPPED; }
  deliver(order) {
    console.log('📦 সফলভাবে ডেলিভারি হয়েছে!');
    order.setState(new DeliveredState());
  }
}

class DeliveredState extends BaseOrderState {
  get status() { return OrderStatus.DELIVERED; }
}

class CancelledState extends BaseOrderState {
  get status() { return OrderStatus.CANCELLED; }
}

// Context
class Order {
  #state;
  #history = [];
  trackingNumber = null;

  constructor(orderId, amount) {
    this.orderId = orderId;
    this.amount = amount;
    this.#state = new PendingState();
    this.#log('Order তৈরি হয়েছে');
  }

  setState(state) {
    const prev = this.#state.status;
    this.#state = state;
    this.#log(`${prev} → ${state.status}`);
  }

  confirm()  { this.#state.confirm(this); }
  process()  { this.#state.process(this); }
  ship()     { this.#state.ship(this); }
  deliver()  { this.#state.deliver(this); }
  cancel()   { this.#state.cancel(this); }

  get status() { return this.#state.status; }

  initiateRefund() {
    console.log(`💰 ৳${this.amount} refund প্রক্রিয়া শুরু।`);
  }

  #log(msg) {
    this.#history.push({ time: new Date().toISOString(), message: msg, status: this.#state.status });
  }

  get history() { return [...this.#history]; }
}

// ব্যবহার
const order = new Order('ORD-2024-001', 2500);
order.confirm();
order.process();
order.trackingNumber = 'BD-TRACK-98765';
order.ship();
order.deliver();
console.log('History:', order.history);
```

---

### 3️⃣ Document Workflow — Draft → Review → Approved → Published

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface DocumentState
{
    public function edit(Document $doc): void;
    public function submitForReview(Document $doc): void;
    public function approve(Document $doc): void;
    public function reject(Document $doc): void;
    public function publish(Document $doc): void;
    public function getName(): string;
}

abstract class BaseDocumentState implements DocumentState
{
    public function edit(Document $doc): void
    {
        throw new \RuntimeException("{$this->getName()} state-এ edit করা যায় না");
    }

    public function submitForReview(Document $doc): void
    {
        throw new \RuntimeException("{$this->getName()} state-এ review-তে পাঠানো যায় না");
    }

    public function approve(Document $doc): void
    {
        throw new \RuntimeException("{$this->getName()} state-এ approve করা যায় না");
    }

    public function reject(Document $doc): void
    {
        throw new \RuntimeException("{$this->getName()} state-এ reject করা যায় না");
    }

    public function publish(Document $doc): void
    {
        throw new \RuntimeException("{$this->getName()} state-এ publish করা যায় না");
    }
}

class DraftState extends BaseDocumentState
{
    public function edit(Document $doc): void
    {
        echo "📝 ড্রাফটে পরিবর্তন সংরক্ষিত হয়েছে।\n";
    }

    public function submitForReview(Document $doc): void
    {
        if (empty($doc->getContent())) {
            throw new \LogicException('খালি ডকুমেন্ট review-তে পাঠানো যাবে না');
        }
        echo "📤 ডকুমেন্ট review-এ পাঠানো হয়েছে।\n";
        $doc->setState(new ReviewState());
    }

    public function getName(): string { return 'Draft'; }
}

class ReviewState extends BaseDocumentState
{
    public function approve(Document $doc): void
    {
        echo "✅ ডকুমেন্ট অনুমোদিত হয়েছে।\n";
        $doc->setState(new ApprovedState());
    }

    public function reject(Document $doc): void
    {
        echo "🔙 ডকুমেন্ট প্রত্যাখ্যান। ড্রাফটে ফেরত।\n";
        $doc->setState(new DraftState());
    }

    public function getName(): string { return 'Review'; }
}

class ApprovedState extends BaseDocumentState
{
    public function publish(Document $doc): void
    {
        echo "🌐 ডকুমেন্ট প্রকাশিত হয়েছে!\n";
        $doc->setPublishedAt(new \DateTimeImmutable());
        $doc->setState(new PublishedState());
    }

    public function getName(): string { return 'Approved'; }
}

class PublishedState extends BaseDocumentState
{
    public function getName(): string { return 'Published'; }
}

class Document
{
    private DocumentState $state;
    private ?\DateTimeImmutable $publishedAt = null;

    public function __construct(
        private string $title,
        private string $content = '',
    ) {
        $this->state = new DraftState();
    }

    public function setState(DocumentState $state): void { $this->state = $state; }
    public function getContent(): string { return $this->content; }
    public function setContent(string $c): void { $this->content = $c; }
    public function setPublishedAt(\DateTimeImmutable $d): void { $this->publishedAt = $d; }
    public function getStateName(): string { return $this->state->getName(); }

    public function edit(): void { $this->state->edit($this); }
    public function submitForReview(): void { $this->state->submitForReview($this); }
    public function approve(): void { $this->state->approve($this); }
    public function reject(): void { $this->state->reject($this); }
    public function publish(): void { $this->state->publish($this); }
}

// ব্যবহার
$doc = new Document('State Pattern গাইড');
$doc->setContent('State Pattern হলো...');
$doc->edit();              // ✅ Draft-এ edit চলে
$doc->submitForReview();   // ✅ Review-তে পাঠানো হলো
$doc->reject();            // 🔙 Draft-এ ফেরত
$doc->submitForReview();   // আবার review-তে
$doc->approve();           // ✅ অনুমোদিত
$doc->publish();           // 🌐 প্রকাশিত
```

---

### 4️⃣ Vending Machine States

#### JavaScript (ES2022+)

```javascript
class VendingMachineState {
  insertCoin(machine)   { throw new Error('Not allowed in this state'); }
  selectProduct(machine, product) { throw new Error('Not allowed'); }
  dispense(machine)     { throw new Error('Not allowed in this state'); }
  get name() { throw new Error('Must implement'); }
}

class IdleState extends VendingMachineState {
  get name() { return 'IDLE'; }

  insertCoin(machine) {
    console.log(`💰 ৳${machine.insertedAmount} জমা হয়েছে।`);
    machine.setState(new HasMoneyState());
  }
}

class HasMoneyState extends VendingMachineState {
  get name() { return 'HAS_MONEY'; }

  insertCoin(machine) {
    console.log(`💰 আরো টাকা জমা। মোট: ৳${machine.insertedAmount}`);
  }

  selectProduct(machine, product) {
    if (machine.insertedAmount < product.price) {
      console.log(`❌ পর্যাপ্ত টাকা নেই। দরকার: ৳${product.price}, আছে: ৳${machine.insertedAmount}`);
      return;
    }
    if (product.stock <= 0) {
      console.log('❌ পণ্য স্টকে নেই।');
      return;
    }
    console.log(`🛒 "${product.name}" নির্বাচিত। বিতরণ হচ্ছে...`);
    machine.selectedProduct = product;
    machine.setState(new DispensingState());
    machine.dispense();
  }
}

class DispensingState extends VendingMachineState {
  get name() { return 'DISPENSING'; }

  dispense(machine) {
    const product = machine.selectedProduct;
    product.stock--;
    const change = machine.insertedAmount - product.price;
    console.log(`📦 "${product.name}" বের হচ্ছে...`);
    if (change > 0) console.log(`💵 ফেরত: ৳${change}`);
    machine.insertedAmount = 0;
    machine.selectedProduct = null;
    machine.setState(new IdleState());
  }
}

class VendingMachine {
  #state;
  insertedAmount = 0;
  selectedProduct = null;
  products = new Map();

  constructor() {
    this.#state = new IdleState();
  }

  addProduct(id, name, price, stock) {
    this.products.set(id, { name, price, stock });
  }

  setState(state) { this.#state = state; }
  get stateName() { return this.#state.name; }

  insertCoin(amount) {
    this.insertedAmount += amount;
    this.#state.insertCoin(this);
  }

  selectProduct(productId) {
    const product = this.products.get(productId);
    if (!product) throw new Error('পণ্য পাওয়া যায়নি');
    this.#state.selectProduct(this, product);
  }

  dispense() { this.#state.dispense(this); }
}

// ব্যবহার
const vm = new VendingMachine();
vm.addProduct('chips', 'চিপস', 30, 5);
vm.addProduct('drink', 'কোল্ড ড্রিংক', 50, 3);

vm.insertCoin(20);
vm.insertCoin(20);
vm.selectProduct('chips');  // ৳40 দিয়ে ৳30 এর চিপস → ৳10 ফেরত
```

---

## 🌍 Real-World Applicable Areas

### 1. Payment Transaction States (bKash/Nagad Context)

```php
<?php

declare(strict_types=1);

enum TransactionStatus: string
{
    case Initiated  = 'initiated';
    case Processing = 'processing';
    case Success    = 'success';
    case Failed     = 'failed';
    case Refunded   = 'refunded';
}

interface TransactionState
{
    public function process(Transaction $tx): void;
    public function complete(Transaction $tx): void;
    public function fail(Transaction $tx, string $reason): void;
    public function refund(Transaction $tx): void;
    public function getStatus(): TransactionStatus;
}

abstract class BaseTransactionState implements TransactionState
{
    public function process(Transaction $tx): void
    {
        throw new \RuntimeException("এই state-এ process করা যায় না");
    }

    public function complete(Transaction $tx): void
    {
        throw new \RuntimeException("এই state-এ complete করা যায় না");
    }

    public function fail(Transaction $tx, string $reason): void
    {
        throw new \RuntimeException("এই state-এ fail করা যায় না");
    }

    public function refund(Transaction $tx): void
    {
        throw new \RuntimeException("এই state-এ refund করা যায় না");
    }
}

class InitiatedState extends BaseTransactionState
{
    public function process(Transaction $tx): void
    {
        echo "⏳ লেনদেন প্রসেসিং শুরু: {$tx->getId()}\n";
        $tx->setState(new ProcessingTransactionState());
    }

    public function getStatus(): TransactionStatus
    {
        return TransactionStatus::Initiated;
    }
}

class ProcessingTransactionState extends BaseTransactionState
{
    public function complete(Transaction $tx): void
    {
        echo "✅ লেনদেন সফল! TxID: {$tx->getId()}\n";
        $tx->setState(new SuccessState());
    }

    public function fail(Transaction $tx, string $reason): void
    {
        echo "❌ লেনদেন ব্যর্থ: {$reason}\n";
        $tx->setFailureReason($reason);
        $tx->setState(new FailedState());
    }

    public function getStatus(): TransactionStatus
    {
        return TransactionStatus::Processing;
    }
}

class SuccessState extends BaseTransactionState
{
    public function refund(Transaction $tx): void
    {
        echo "💸 ৳{$tx->getAmount()} refund হচ্ছে...\n";
        $tx->setState(new RefundedState());
    }

    public function getStatus(): TransactionStatus
    {
        return TransactionStatus::Success;
    }
}

class FailedState extends BaseTransactionState
{
    public function getStatus(): TransactionStatus
    {
        return TransactionStatus::Failed;
    }
}

class RefundedState extends BaseTransactionState
{
    public function getStatus(): TransactionStatus
    {
        return TransactionStatus::Refunded;
    }
}

class Transaction
{
    private TransactionState $state;
    private ?string $failureReason = null;

    public function __construct(
        private readonly string $id,
        private readonly float $amount,
        private readonly string $sender,
        private readonly string $receiver,
    ) {
        $this->state = new InitiatedState();
    }

    public function setState(TransactionState $state): void { $this->state = $state; }
    public function getId(): string { return $this->id; }
    public function getAmount(): float { return $this->amount; }
    public function setFailureReason(string $r): void { $this->failureReason = $r; }

    public function process(): void { $this->state->process($this); }
    public function complete(): void { $this->state->complete($this); }
    public function fail(string $reason): void { $this->state->fail($this, $reason); }
    public function refund(): void { $this->state->refund($this); }
}

// bKash লেনদেন সিমুলেশন
$tx = new Transaction('TXN-BK-001', 500.00, '01712345678', '01887654321');
$tx->process();
$tx->complete();
$tx->refund();
```

### 2. Game Character States

```javascript
class CharacterState {
  enter(character)  {}
  exit(character)   {}
  update(character) {}
  handleInput(character, input) {}
  get name() { return 'BASE'; }
}

class IdleCharState extends CharacterState {
  get name() { return 'IDLE'; }

  enter(character) { console.log('🧍 চরিত্র দাঁড়িয়ে আছে'); }

  handleInput(character, input) {
    switch (input) {
      case 'MOVE':   character.setState(new WalkingState()); break;
      case 'JUMP':   character.setState(new JumpingState()); break;
      case 'ATTACK': character.setState(new AttackingState()); break;
    }
  }
}

class WalkingState extends CharacterState {
  get name() { return 'WALKING'; }
  enter(character) { console.log('🚶 চরিত্র হাঁটছে'); character.speed = 5; }
  exit(character)  { character.speed = 0; }

  handleInput(character, input) {
    switch (input) {
      case 'STOP':   character.setState(new IdleCharState()); break;
      case 'RUN':    character.setState(new RunningState()); break;
      case 'JUMP':   character.setState(new JumpingState()); break;
    }
  }
}

class RunningState extends CharacterState {
  get name() { return 'RUNNING'; }
  enter(character) { console.log('🏃 চরিত্র দৌড়াচ্ছে!'); character.speed = 12; }
  exit(character)  { character.speed = 0; }

  handleInput(character, input) {
    if (input === 'STOP') character.setState(new WalkingState());
    if (input === 'JUMP') character.setState(new JumpingState());
  }
}

class JumpingState extends CharacterState {
  get name() { return 'JUMPING'; }
  enter(character) { console.log('🦘 চরিত্র লাফাচ্ছে!'); }

  update(character) {
    // মাটিতে নামলে idle-এ ফেরত
    if (character.isGrounded) character.setState(new IdleCharState());
  }
}

class AttackingState extends CharacterState {
  #attackDuration = 500; // ms

  get name() { return 'ATTACKING'; }
  enter(character) { console.log('⚔️ আক্রমণ!'); character.damage = 25; }
  exit(character)  { character.damage = 0; }

  update(character) {
    // আক্রমণ শেষে idle-এ ফেরত
    character.setState(new IdleCharState());
  }
}

class GameCharacter {
  #state;
  speed = 0;
  damage = 0;
  isGrounded = true;

  constructor(name) {
    this.name = name;
    this.#state = new IdleCharState();
    this.#state.enter(this);
  }

  setState(newState) {
    this.#state.exit(this);
    this.#state = newState;
    this.#state.enter(this);
  }

  handleInput(input) { this.#state.handleInput(this, input); }
  update() { this.#state.update(this); }
  get currentState() { return this.#state.name; }
}

const hero = new GameCharacter('বীর যোদ্ধা');
hero.handleInput('MOVE');
hero.handleInput('RUN');
hero.handleInput('JUMP');
```

### 3. অন্যান্য Real-World ব্যবহার ক্ষেত্র

| ক্ষেত্র | States | বিবরণ |
|---------|--------|--------|
| **TCP Connection** | CLOSED, LISTEN, SYN_SENT, ESTABLISHED, FIN_WAIT, CLOSE_WAIT | নেটওয়ার্ক কানেকশন ম্যানেজমেন্ট |
| **User Account** | Active, Suspended, Banned, PendingVerification, Deleted | ব্যবহারকারী জীবনচক্র |
| **Workflow Engine** | Draft, Submitted, UnderReview, Approved, Rejected | অনুমোদন প্রক্রিয়া |
| **Laravel Model** | Eloquent Model States (spatie/laravel-model-states) | মডেল স্টেট ম্যানেজমেন্ট |
| **Media Player** | Playing, Paused, Stopped, Buffering | মিডিয়া প্লেব্যাক কন্ট্রোল |
| **Elevator** | Idle, MovingUp, MovingDown, DoorOpen, Maintenance | লিফট নিয়ন্ত্রণ সিস্টেম |

---

## 🔥 Advanced Deep Dive

### State vs Strategy — তুলনামূলক আলোচনা

এই দুটি প্যাটার্ন দেখতে প্রায় একই রকম, কিন্তু উদ্দেশ্য সম্পূর্ণ আলাদা:

| বিষয় | State | Strategy |
|--------|-------|----------|
| **উদ্দেশ্য** | অবজেক্টের অভ্যন্তরীণ অবস্থার উপর নির্ভর করে behavior পরিবর্তন | একটি নির্দিষ্ট কাজের জন্য বিভিন্ন algorithm প্রদান |
| **State পরিবর্তন** | State নিজেই পরবর্তী state-এ transition করে | Client বাইরে থেকে strategy সেট করে |
| **সচেতনতা** | State ক্লাসগুলো একে অপরকে চেনে | Strategy ক্লাসগুলো একে অপরকে চেনে না |
| **সংখ্যা** | সময়ের সাথে অনেক state পরিবর্তন হয় | সাধারণত একবার সেট করা হয় |
| **উদাহরণ** | অর্ডারের Pending → Shipped → Delivered | পেমেন্টে bKash/Nagad/Card strategy |

```php
// State — state নিজেই transition করে
class PendingState implements OrderState {
    public function confirm(Order $order): void {
        $order->setState(new ConfirmedState()); // state নিজে পরবর্তী state সেট করে
    }
}

// Strategy — client বাইরে থেকে সেট করে
$payment->setStrategy(new BkashPayment()); // client নিজে strategy বাছাই করে
$payment->pay(500);
```

### State vs if/else/switch — কেন State প্যাটার্ন ভালো?

```php
// ❌ if/else পদ্ধতি — Open/Closed Principle লঙ্ঘন
class Order {
    private string $status = 'pending';

    public function ship(): void {
        // নতুন state যোগ করতে হলে এই method পরিবর্তন করতে হবে
        match ($this->status) {
            'pending'    => throw new Exception('আগে confirm করুন'),
            'confirmed'  => throw new Exception('আগে process করুন'),
            'processing' => $this->status = 'shipped',
            'shipped'    => throw new Exception('ইতিমধ্যে shipped'),
            'delivered'  => throw new Exception('ইতিমধ্যে delivered'),
            'cancelled'  => throw new Exception('বাতিল অর্ডার ship করা যায় না'),
        };
        // প্রতিটি method-এ এই বিশাল match/switch থাকবে!
    }
}

// ✅ State Pattern পদ্ধতি — নতুন state যোগে existing কোড পরিবর্তন লাগে না
class ProcessingState extends BaseOrderState {
    public function ship(Order $order): void {
        $order->setState(new ShippedState()); // পরিষ্কার, focused
    }
}
```

**State Pattern-এর সুবিধা:**
- ✅ **Single Responsibility** — প্রতিটি state-এর logic আলাদা ক্লাসে
- ✅ **Open/Closed** — নতুন state যোগ করতে existing কোড পরিবর্তন লাগে না
- ✅ **পরীক্ষাযোগ্য** — প্রতিটি state আলাদা ভাবে test করা যায়
- ✅ **Readable** — বড় switch/if-else এর বদলে পরিষ্কার ক্লাস

### Finite State Machine (FSM) — Generic Implementation

```php
<?php

declare(strict_types=1);

class StateMachine
{
    private string $currentState;

    /** @var array<string, array<string, array{to: string, guard?: callable, action?: callable}>> */
    private array $transitions = [];

    /** @var array<string, callable[]> */
    private array $listeners = [];

    public function __construct(
        private readonly string $initialState,
        private readonly array $validStates,
    ) {
        if (!in_array($initialState, $validStates, true)) {
            throw new \InvalidArgumentException("Invalid initial state: {$initialState}");
        }
        $this->currentState = $initialState;
    }

    public function addTransition(
        string $from,
        string $event,
        string $to,
        ?callable $guard = null,
        ?callable $action = null,
    ): self {
        $this->transitions[$from][$event] = [
            'to' => $to,
            'guard' => $guard,
            'action' => $action,
        ];
        return $this;
    }

    public function onTransition(string $event, callable $listener): self
    {
        $this->listeners[$event][] = $listener;
        return $this;
    }

    public function trigger(string $event, array $context = []): bool
    {
        $transition = $this->transitions[$this->currentState][$event] ?? null;

        if ($transition === null) {
            throw new \RuntimeException(
                "'{$this->currentState}' state-এ '{$event}' event প্রযোজ্য নয়"
            );
        }

        // Guard condition চেক
        if ($transition['guard'] !== null && !($transition['guard'])($context)) {
            return false;
        }

        $from = $this->currentState;
        $this->currentState = $transition['to'];

        // Action execute
        if ($transition['action'] !== null) {
            ($transition['action'])($from, $this->currentState, $context);
        }

        // Listeners notify
        foreach ($this->listeners[$event] ?? [] as $listener) {
            $listener($from, $this->currentState, $context);
        }

        return true;
    }

    public function getCurrentState(): string { return $this->currentState; }

    public function canTrigger(string $event): bool
    {
        return isset($this->transitions[$this->currentState][$event]);
    }

    public function getAvailableEvents(): array
    {
        return array_keys($this->transitions[$this->currentState] ?? []);
    }
}

// ─── ব্যবহার: ডেলিভারি ট্র্যাকিং FSM ───

$deliveryFSM = new StateMachine('placed', [
    'placed', 'accepted', 'preparing', 'rider_assigned',
    'picked_up', 'on_the_way', 'delivered', 'cancelled',
]);

$deliveryFSM
    ->addTransition('placed', 'accept', 'accepted')
    ->addTransition('placed', 'cancel', 'cancelled')
    ->addTransition('accepted', 'start_prepare', 'preparing')
    ->addTransition('preparing', 'assign_rider', 'rider_assigned',
        guard: fn($ctx) => !empty($ctx['rider_id']), // guard: rider_id থাকতে হবে
        action: fn($from, $to, $ctx) => print("🛵 রাইডার #{$ctx['rider_id']} নিয়োগ\n"),
    )
    ->addTransition('rider_assigned', 'pickup', 'picked_up')
    ->addTransition('picked_up', 'start_delivery', 'on_the_way')
    ->addTransition('on_the_way', 'complete', 'delivered',
        action: fn($from, $to, $ctx) => print("✅ ডেলিভারি সম্পন্ন!\n"),
    );

// Observer: state পরিবর্তনে notification
$deliveryFSM->onTransition('complete', function ($from, $to, $ctx) {
    echo "📱 SMS পাঠানো হচ্ছে: আপনার অর্ডার ডেলিভারি হয়েছে!\n";
});

$deliveryFSM->trigger('accept');
$deliveryFSM->trigger('start_prepare');
$deliveryFSM->trigger('assign_rider', ['rider_id' => 'R-042']);
$deliveryFSM->trigger('pickup');
$deliveryFSM->trigger('start_delivery');
$deliveryFSM->trigger('complete');

echo "Available: " . implode(', ', $deliveryFSM->getAvailableEvents()) . "\n"; // কোনো event নেই, terminal
```

### State Transition Table

Transition table হলো সমস্ত valid state transition-কে একটি data structure-এ সাজানো:

```javascript
// Declarative State Machine — Transition Table ভিত্তিক
class TableDrivenStateMachine {
  #state;
  #table;
  #listeners = new Map();

  constructor(initialState, transitionTable) {
    this.#state = initialState;
    this.#table = transitionTable;
  }

  get state() { return this.#state; }

  on(event, listener) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, []);
    this.#listeners.get(event).push(listener);
    return this;
  }

  send(event, payload = {}) {
    const stateTransitions = this.#table[this.#state];
    if (!stateTransitions || !stateTransitions[event]) {
      throw new Error(`'${this.#state}' এ '${event}' event অবৈধ`);
    }

    const { target, guard, action } = stateTransitions[event];

    if (guard && !guard(payload)) return false;

    const prev = this.#state;
    this.#state = target;

    action?.(prev, target, payload);
    this.#listeners.get(event)?.forEach(fn => fn(prev, target, payload));

    return true;
  }

  get availableEvents() {
    return Object.keys(this.#table[this.#state] || {});
  }
}

// ─── bKash Transaction State Machine ───
const bkashFSM = new TableDrivenStateMachine('initiated', {
  initiated: {
    verify_pin: { target: 'verifying' },
    cancel:     { target: 'cancelled' },
  },
  verifying: {
    pin_ok:   { target: 'processing' },
    pin_fail: { target: 'failed', action: (f, t) => console.log('❌ PIN ভুল!') },
  },
  processing: {
    success: {
      target: 'success',
      action: (from, to, { amount, receiver }) => {
        console.log(`✅ ৳${amount} সফলভাবে ${receiver}-এ পাঠানো হয়েছে`);
      },
    },
    fail: { target: 'failed' },
  },
  success: {
    refund: { target: 'refunded' },
  },
  failed: {
    retry: { target: 'initiated' },
  },
  cancelled: {},
  refunded: {},
});

bkashFSM.on('success', (from, to, { amount }) => {
  console.log(`📱 SMS: আপনার ৳${amount} Send Money সফল হয়েছে`);
});

bkashFSM.send('verify_pin');
bkashFSM.send('pin_ok');
bkashFSM.send('success', { amount: 1000, receiver: '01887654321' });
console.log(`Available events: ${bkashFSM.availableEvents}`);
```

### State + Observer Pattern — State পরিবর্তনে Notification

```php
<?php

declare(strict_types=1);

interface StateChangeObserver
{
    public function onStateChange(string $entity, string $from, string $to, array $context): void;
}

class EmailNotifier implements StateChangeObserver
{
    public function onStateChange(string $entity, string $from, string $to, array $context): void
    {
        echo "📧 ইমেইল পাঠানো হচ্ছে: {$entity} এর status '{$from}' থেকে '{$to}' হয়েছে\n";
    }
}

class SmsNotifier implements StateChangeObserver
{
    public function onStateChange(string $entity, string $from, string $to, array $context): void
    {
        echo "📱 SMS: {$entity} আপডেট — নতুন status: {$to}\n";
    }
}

class AuditLogger implements StateChangeObserver
{
    public function onStateChange(string $entity, string $from, string $to, array $context): void
    {
        $time = date('Y-m-d H:i:s');
        echo "📝 Audit: [{$time}] {$entity}: {$from} → {$to}\n";
    }
}

// Observable State Machine
class ObservableOrder
{
    private OrderState $state;

    /** @var StateChangeObserver[] */
    private array $observers = [];

    public function __construct(private readonly string $orderId)
    {
        $this->state = new PendingState();
    }

    public function addObserver(StateChangeObserver $observer): void
    {
        $this->observers[] = $observer;
    }

    public function setState(OrderState $newState): void
    {
        $from = $this->state->getStatus()->value;
        $to   = $newState->getStatus()->value;
        $this->state = $newState;

        foreach ($this->observers as $observer) {
            $observer->onStateChange($this->orderId, $from, $to, []);
        }
    }

    // ... বাকি methods আগের মতো
}
```

### PHP Enums দিয়ে State Identifier + Transition Rules

```php
<?php

declare(strict_types=1);

enum OrderStep: string
{
    case Pending    = 'pending';
    case Confirmed  = 'confirmed';
    case Processing = 'processing';
    case Shipped    = 'shipped';
    case Delivered  = 'delivered';
    case Cancelled  = 'cancelled';

    /**
     * এই state থেকে কোন কোন state-এ যাওয়া যায়
     * @return self[]
     */
    public function allowedTransitions(): array
    {
        return match ($this) {
            self::Pending    => [self::Confirmed, self::Cancelled],
            self::Confirmed  => [self::Processing, self::Cancelled],
            self::Processing => [self::Shipped],
            self::Shipped    => [self::Delivered],
            self::Delivered  => [],
            self::Cancelled  => [],
        };
    }

    public function canTransitionTo(self $target): bool
    {
        return in_array($target, $this->allowedTransitions(), true);
    }

    public function isTerminal(): bool
    {
        return empty($this->allowedTransitions());
    }

    public function label(): string
    {
        return match ($this) {
            self::Pending    => '⏳ অপেক্ষমাণ',
            self::Confirmed  => '✅ নিশ্চিত',
            self::Processing => '⚙️ প্রক্রিয়াধীন',
            self::Shipped    => '🚚 প্রেরিত',
            self::Delivered  => '📦 বিতরিত',
            self::Cancelled  => '❌ বাতিল',
        };
    }
}

// Enum-ভিত্তিক transition ব্যবহার
$current = OrderStep::Pending;
$target  = OrderStep::Confirmed;

if ($current->canTransitionTo($target)) {
    echo "{$current->label()} → {$target->label()} ✅ অনুমোদিত\n";
} else {
    echo "⚠️ এই transition অবৈধ\n";
}
```

### Hierarchical State Machine (HSM) — সংক্ষিপ্ত ধারণা

HSM-এ state-এর মধ্যে আরো sub-state থাকতে পারে। যেমন, একটি গেমের `Active` state-এর মধ্যে `Walking`, `Running`, `Jumping` sub-state থাকতে পারে:

```javascript
// Hierarchical State Machine — সরলীকৃত
class HierarchicalState {
  #parent;
  #children = new Map();
  #activeChild = null;

  constructor(name, parent = null) {
    this.name = name;
    this.#parent = parent;
    parent?.addChild(this);
  }

  addChild(child) { this.#children.set(child.name, child); }

  enter(context) {
    console.log(`↳ ${this.name} state-এ প্রবেশ`);
    // প্রথম child-কে default active করো
    if (this.#children.size > 0) {
      this.#activeChild = this.#children.values().next().value;
      this.#activeChild.enter(context);
    }
  }

  exit(context) {
    this.#activeChild?.exit(context);
    console.log(`↲ ${this.name} state থেকে বের`);
  }

  handleEvent(event, context) {
    // আগে active child-কে handle করতে দাও
    if (this.#activeChild?.handleEvent(event, context)) return true;
    // child handle না করলে নিজে handle করো
    return false;
  }
}

/*
  GameState (root)
  ├── Menu
  ├── Playing
  │   ├── Exploring
  │   ├── Combat
  │   │   ├── Attacking
  │   │   └── Defending
  │   └── Paused
  └── GameOver
*/
```

### State Machine Libraries

| ভাষা | লাইব্রেরি | বৈশিষ্ট্য |
|------|----------|----------|
| **PHP** | `spatie/laravel-model-states` | Laravel Eloquent মডেলে state management |
| **PHP** | `winzou/state-machine` | Symfony-friendly, config-driven FSM |
| **PHP** | `asantibanez/laravel-eloquent-state-machines` | Eloquent event hooks সহ |
| **JS** | `xstate` | সবচেয়ে জনপ্রিয়, visualizer আছে, statecharts সাপোর্ট |
| **JS** | `robot3` | হালকা, functional approach |
| **JS** | `javascript-state-machine` | সহজ, declarative configuration |

---

## ✅ Pros (সুবিধা)

1. **Single Responsibility Principle** — প্রতিটি state-এর code আলাদা ক্লাসে, পরিষ্কার ও organized
2. **Open/Closed Principle** — নতুন state যোগ করতে existing state ক্লাস পরিবর্তন লাগে না
3. **বিশাল conditional logic দূর** — if/else/switch-এর জটিল জঞ্জাল থেকে মুক্তি
4. **State transition স্পষ্ট** — কোন state থেকে কোন state-এ যাওয়া যায় তা স্পষ্ট ও documented
5. **টেস্টিং সহজ** — প্রতিটি state আলাদাভাবে unit test করা যায়
6. **ডিবাগিং সহজ** — সমস্যা হলে নির্দিষ্ট state ক্লাসে দেখলেই হয়
7. **Context class সরল** — Context শুধু state-এ delegate করে, নিজে কোনো business logic রাখে না

## ❌ Cons (অসুবিধা)

1. **ক্লাস সংখ্যা বৃদ্ধি** — প্রতিটি state-এর জন্য আলাদা ক্লাস, ছোট প্রজেক্টে overkill হতে পারে
2. **Over-engineering** — মাত্র ২-৩টি state থাকলে simple if/else যথেষ্ট
3. **State ক্লাসগুলোর মধ্যে coupling** — একটি state আরেকটি state ক্লাসকে জানে (transition-এ)
4. **Global/shared state** — একাধিক Context একই state share করলে জটিলতা বাড়ে
5. **শেখার বক্ররেখা** — নতুন ডেভেলপারদের বুঝতে সময় লাগে

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ১. Context-এ state logic রাখা

```php
// ❌ ভুল — Context-এ business logic
class Order {
    public function ship(): void {
        if ($this->state instanceof ProcessingState) {
            // business logic এখানে থাকা উচিত না!
            $this->validateShipping();
            $this->state = new ShippedState();
        }
    }
}

// ✅ সঠিক — State ক্লাসে logic
class ProcessingState extends BaseOrderState {
    public function ship(Order $order): void {
        $order->validateShipping();
        $order->setState(new ShippedState());
    }
}
```

### ২. State ক্লাসে mutable data রাখা

```php
// ❌ ভুল — State-এ mutable data রাখলে shared state সমস্যা হয়
class ProcessingState {
    private int $retryCount = 0; // একাধিক অর্ডার share করলে সমস্যা!
}

// ✅ সঠিক — Data Context-এ রাখুন, State শুধু behavior handle করুক
class Order {
    private int $retryCount = 0;
}
```

### ৩. Transition validation না করা

```php
// ❌ ভুল — যেকোনো state-এ যেকোনো transition
class Order {
    public function setState(OrderState $state): void {
        $this->state = $state; // কোনো যাচাই নেই!
    }
}

// ✅ সঠিক — Transition validation
class Order {
    public function setState(OrderState $newState): void {
        $allowed = $this->state->getStatus()->allowedTransitions();
        if (!in_array($newState->getStatus(), $allowed, true)) {
            throw new InvalidStateTransitionException('অবৈধ transition');
        }
        $this->state = $newState;
    }
}
```

### ৪. Terminal state-এ transition ভুলে যাওয়া

Terminal state (Delivered, Cancelled) থেকে অন্য কোথাও যাওয়া যাবে না — এটা enforce করতে ভুলবেন না।

### ৫. State history track না করা

Production সিস্টেমে state transition-এর history রাখা অত্যন্ত গুরুত্বপূর্ণ — debugging ও audit-এর জন্য।

---

## 🧪 টেস্টিং

### PHPUnit — State Transition টেস্ট

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class OrderStateTest extends TestCase
{
    private Order $order;

    protected function setUp(): void
    {
        $this->order = new Order('TEST-001', 1500.00);
    }

    #[Test]
    public function নতুন_অর্ডার_pending_state_এ_থাকে(): void
    {
        $this->assertSame('pending', $this->order->getStatus());
    }

    #[Test]
    public function pending_থেকে_confirmed_এ_transition_হয়(): void
    {
        $this->order->confirm();
        $this->assertSame('confirmed', $this->order->getStatus());
    }

    #[Test]
    public function confirmed_থেকে_cancelled_এ_transition_হয়(): void
    {
        $this->order->confirm();
        $this->order->cancel();
        $this->assertSame('cancelled', $this->order->getStatus());
    }

    #[Test]
    public function সম্পূর্ণ_happy_path_transition(): void
    {
        $this->order->confirm();
        $this->order->process();
        $this->order->setTrackingNumber('TRACK-123');
        $this->order->ship();
        $this->order->deliver();

        $this->assertSame('delivered', $this->order->getStatus());
    }

    #[Test]
    public function pending_থেকে_সরাসরি_ship_করলে_exception(): void
    {
        $this->expectException(InvalidStateTransitionException::class);
        $this->order->ship();
    }

    #[Test]
    public function delivered_অর্ডার_cancel_করলে_exception(): void
    {
        $this->order->confirm();
        $this->order->process();
        $this->order->setTrackingNumber('TRACK-123');
        $this->order->ship();
        $this->order->deliver();

        $this->expectException(InvalidStateTransitionException::class);
        $this->order->cancel();
    }

    #[Test]
    public function tracking_ছাড়া_ship_করলে_exception(): void
    {
        $this->order->confirm();
        $this->order->process();

        $this->expectException(\LogicException::class);
        $this->order->ship();
    }

    #[Test]
    public function history_সঠিকভাবে_track_হয়(): void
    {
        $this->order->confirm();
        $this->order->process();

        $history = $this->order->getHistory();
        $this->assertCount(3, $history); // তৈরি + confirm + process
    }

    #[DataProvider('invalidTransitionsProvider')]
    #[Test]
    public function অবৈধ_transition_এ_exception(string $method, array $setup): void
    {
        foreach ($setup as $step) {
            $this->order->{$step}();
        }

        $this->expectException(InvalidStateTransitionException::class);
        $this->order->{$method}();
    }

    public static function invalidTransitionsProvider(): array
    {
        return [
            'pending → ship'    => ['ship', []],
            'pending → deliver' => ['deliver', []],
            'pending → process' => ['process', []],
            'shipped → confirm' => ['confirm', ['confirm', 'process']],
        ];
    }
}
```

### Jest — JavaScript State Transition টেস্ট

```javascript
describe('Order State Machine', () => {
  let order;

  beforeEach(() => {
    order = new Order('TEST-001', 1500);
  });

  test('নতুন অর্ডার pending state-এ থাকে', () => {
    expect(order.status).toBe('pending');
  });

  test('সম্পূর্ণ happy path transition কাজ করে', () => {
    order.confirm();
    order.process();
    order.trackingNumber = 'TRACK-123';
    order.ship();
    order.deliver();

    expect(order.status).toBe('delivered');
  });

  test('অবৈধ transition-এ error throw হয়', () => {
    expect(() => order.ship()).toThrow(InvalidStateTransition);
    expect(() => order.deliver()).toThrow(InvalidStateTransition);
    expect(() => order.process()).toThrow(InvalidStateTransition);
  });

  test('pending থেকে cancel করা যায়', () => {
    order.cancel();
    expect(order.status).toBe('cancelled');
  });

  test('delivered অর্ডার cancel করা যায় না', () => {
    order.confirm();
    order.process();
    order.trackingNumber = 'TRACK-123';
    order.ship();
    order.deliver();

    expect(() => order.cancel()).toThrow(InvalidStateTransition);
  });

  test('tracking ছাড়া ship করলে error', () => {
    order.confirm();
    order.process();

    expect(() => order.ship()).toThrow();
  });

  test('state transition history সঠিক', () => {
    order.confirm();
    order.process();

    expect(order.history).toHaveLength(3);
    expect(order.history[1].message).toContain('pending → confirmed');
  });

  describe('Table-driven FSM', () => {
    let fsm;

    beforeEach(() => {
      fsm = new TableDrivenStateMachine('initiated', {
        initiated:  { process: { target: 'processing' } },
        processing: {
          success: { target: 'success' },
          fail:    { target: 'failed' },
        },
        success:  { refund: { target: 'refunded' } },
        failed:   { retry:  { target: 'initiated' } },
        refunded: {},
      });
    });

    test('সঠিক initial state', () => {
      expect(fsm.state).toBe('initiated');
    });

    test('valid transition কাজ করে', () => {
      fsm.send('process');
      expect(fsm.state).toBe('processing');
    });

    test('invalid event-এ error throw হয়', () => {
      expect(() => fsm.send('refund')).toThrow();
    });

    test('available events সঠিক', () => {
      expect(fsm.availableEvents).toEqual(['process']);
    });

    test('failure ও retry path', () => {
      fsm.send('process');
      fsm.send('fail');
      expect(fsm.state).toBe('failed');

      fsm.send('retry');
      expect(fsm.state).toBe('initiated');
    });
  });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| **Strategy** | কাঠামো একই কিন্তু উদ্দেশ্য ভিন্ন। Strategy — algorithm বদল, State — আচরণ বদল (state-এর উপর নির্ভর করে) |
| **Singleton** | State অবজেক্টগুলো প্রায়ই stateless হয়, তাই Singleton হিসেবে share করা যায় (memory সাশ্রয়) |
| **Flyweight** | অনেক Context একই state অবজেক্ট share করলে Flyweight ব্যবহার করা যায় |
| **Observer** | State পরিবর্তনের সাথে সাথে interested parties-কে notify করতে Observer ব্যবহার করা যায় |
| **Command** | State transition-কে Command হিসেবে encapsulate করলে undo/redo সম্ভব |
| **Mediator** | জটিল state machine-এ state-গুলোর মধ্যে communication Mediator দিয়ে manage করা যায় |
| **Template Method** | BaseState ক্লাসে common behavior রেখে subclass-এ specific behavior — Template Method pattern |

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

- অবজেক্টের **বিভিন্ন state-এ আচরণ উল্লেখযোগ্যভাবে ভিন্ন** এবং state সংখ্যা ৩+ হয়
- **বড় conditional block** (if/else/switch) state-এর উপর ভিত্তি করে behavior নির্ধারণ করে
- **State transition-এ নির্দিষ্ট নিয়ম** আছে (সব state থেকে সব state-এ যাওয়া যায় না)
- State-ভিত্তিক behavior-এ **ঘন ঘন পরিবর্তন** বা **নতুন state যোগ** হওয়ার সম্ভাবনা আছে
- **Domain model-এ** স্পষ্ট lifecycle আছে (Order, Payment, Document, Ticket)
- **Finite State Machine** model আপনার সমস্যার সাথে মানানসই

### ❌ ব্যবহার করবেন না যখন:

- মাত্র **২টি state** (active/inactive) — simple boolean যথেষ্ট
- State-ভেদে **behavior খুব একটা আলাদা না** — if/else সহজ ও পরিষ্কার
- **State transition নিয়ম সরল** — কোনো complex guard condition নেই
- **কম state, কম action** — pattern-এর overhead ন্যায্য নয়
- আপনার প্রজেক্ট **ছোট ও স্বল্পমেয়াদী** — over-engineering হবে

### সিদ্ধান্ত নেওয়ার চেকলিস্ট:

```
☐ ৩+ state আছে?
☐ প্রতিটি state-এ behavior আলাদা?
☐ State transition-এ নিয়ম আছে?
☐ ভবিষ্যতে নতুন state আসতে পারে?
☐ বর্তমান conditional code জটিল/লম্বা?

≥ ৩টি হ্যাঁ = State Pattern ব্যবহার করুন
< ৩টি হ্যাঁ = সাধারণ approach যথেষ্ট
```

---

## 📋 সারসংক্ষেপ

State Design Pattern আপনার অবজেক্টের state-ভিত্তিক behavior-কে পরিষ্কার, maintainable, ও extensible ভাবে সংগঠিত করে। এটি **Finite State Machine** ধারণার উপর ভিত্তি করে তৈরি এবং বিশেষত সেই সব ক্ষেত্রে কার্যকর যেখানে অবজেক্টের lifecycle-এ একাধিক state থাকে এবং প্রতিটি state-এ আচরণ ভিন্ন।

**মনে রাখুন:**

| বিষয় | মূল কথা |
|--------|---------|
| **কী** | অবজেক্টের state-কে আলাদা ক্লাসে encapsulate করা |
| **কেন** | বড় if/else/switch দূর করা, OCP মেনে চলা |
| **কখন** | ৩+ state, ভিন্ন behavior, নির্দিষ্ট transition নিয়ম |
| **কখন না** | ২টি state, সরল transition, ছোট প্রজেক্ট |
| **সতর্কতা** | Over-engineering, ক্লাস সংখ্যা বৃদ্ধি |
| **Bangladesh context** | bKash/Nagad লেনদেন, ডেলিভারি ট্র্যাকিং, অর্ডার ম্যানেজমেন্ট |
| **সেরা সঙ্গী** | Observer (notify), Command (undo), Strategy (তুলনা) |

> 💡 **চূড়ান্ত পরামর্শ:** State Pattern শেখার সেরা উপায় হলো আপনার প্রজেক্টের একটি entity (যেমন Order, Payment, Ticket) বেছে নিয়ে তার lifecycle-কে State Pattern দিয়ে model করা। প্রথমে state diagram আঁকুন, তারপর transition table লিখুন, তারপর কোড করুন।

---

*"State Pattern হলো সেই ডিজাইন প্যাটার্ন যা আপনার কোডকে বলে — প্রতিটি অবস্থায় নিজের কাজ নিজে সামলাও, অন্যের কাজে নাক গলানোর দরকার নেই।"* 🔄
