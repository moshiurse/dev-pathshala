# 🏛️ Facade প্যাটার্ন (Facade Pattern)

## 📌 সংজ্ঞা (Definition)

> **GoF সংজ্ঞা:** *"Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use."*

**Facade** একটি **Structural Design Pattern** যা একটি জটিল সাবসিস্টেমের সামনে একটি **সরলীকৃত ইন্টারফেস** প্রদান করে। এটি ক্লায়েন্ট কোডকে সাবসিস্টেমের অভ্যন্তরীণ জটিলতা থেকে আলাদা রাখে — ক্লায়েন্টকে শুধু একটি মাত্র এন্ট্রি পয়েন্টের সাথে কথা বলতে হয়, বাকি সব কিছু Facade নিজেই সামলায়।

**মূল ধারণা:** আপনার সিস্টেমে যখন অনেকগুলো subsystem থাকে এবং প্রতিটির নিজস্ব API/interface থাকে, তখন ক্লায়েন্টকে সরাসরি সবার সাথে interact করতে হলে কোড জটিল ও tightly coupled হয়ে যায়। Facade একটি **simplified wrapper** তৈরি করে যেটা সব subsystem-এর কাজ orchestrate করে।

**Pattern Type:** Structural
**Complexity:** ★★☆☆☆ (সহজ কিন্তু প্রভাব বিশাল)
**Popularity:** ★★★★★ (সর্বাধিক ব্যবহৃত pattern-গুলোর একটি)

---

## 🏠 বাস্তব উদাহরণ (Real-World Analogy)

### হোটেল রিসেপশন ডেস্ক

ধরুন আপনি ঢাকার একটি ৫-তারা হোটেলে উঠেছেন। হোটেলে অনেকগুলো সার্ভিস আছে:

- 🍽️ **রেস্টুরেন্ট** — খাবার অর্ডার
- 🚗 **ট্রান্সপোর্ট** — গাড়ি বুকিং
- 🧹 **হাউসকিপিং** — রুম পরিষ্কার
- 🏊 **রিক্রিয়েশন** — পুল, জিম
- 💼 **কনসিয়ার্জ** — ট্যুর বুকিং, টিকেট

এখন আপনি কি প্রতিটি ডিপার্টমেন্টে আলাদা আলাদা গিয়ে কথা বলবেন? **না!** আপনি শুধু **রিসেপশন ডেস্কে** (Facade) ফোন করবেন, এবং তারা সব ব্যবস্থা করে দেবে।

```
আপনি (Client) → রিসেপশন ডেস্ক (Facade) → রেস্টুরেন্ট, ট্রান্সপোর্ট, হাউসকিপিং...
```

### বাংলাদেশ কনটেক্সট — bKash পেমেন্ট

bKash দিয়ে পেমেন্ট করার সময় আপনি শুধু **bKash অ্যাপে** টাকা পাঠান। কিন্তু অভ্যন্তরে:
- ব্যালেন্স চেক হয়
- ফ্রড ডিটেকশন চলে
- ব্যাংক API কল হয়
- ট্রানজেকশন লগ হয়
- SMS নোটিফিকেশন যায়

bKash অ্যাপটাই এখানে **Facade** — আপনাকে এসব জটিলতা জানতে হয় না।

---

## 📊 UML ডায়াগ্রাম

```
┌─────────────────────────────────────────────────────────┐
│                        Client                           │
│                          │                              │
│                          ▼                              │
│              ┌───────────────────────┐                  │
│              │       FACADE          │                  │
│              │                       │                  │
│              │  + operationA()       │                  │
│              │  + operationB()       │                  │
│              │  + complexWorkflow()  │                  │
│              └───────────┬───────────┘                  │
│                          │                              │
│            ┌─────────────┼─────────────┐                │
│            │             │             │                │
│            ▼             ▼             ▼                │
│   ┌──────────────┐ ┌──────────┐ ┌──────────────┐      │
│   │ SubsystemA   │ │SubsystemB│ │ SubsystemC   │      │
│   │              │ │          │ │              │      │
│   │ + methodA1() │ │+ methB1()│ │ + methodC1() │      │
│   │ + methodA2() │ │+ methB2()│ │ + methodC2() │      │
│   └──────────────┘ └──────────┘ └──────────────┘      │
│        Subsystem Layer (জটিল অভ্যন্তরীণ সিস্টেম)       │
└─────────────────────────────────────────────────────────┘

Sequence Diagram:

Client          Facade          SubA       SubB       SubC
  │                │              │          │          │
  │──doWork()────▶│              │          │          │
  │                │──methodA1()─▶│          │          │
  │                │◀─────result──│          │          │
  │                │──────────────methB1()──▶│          │
  │                │◀─────────────result─────│          │
  │                │──────────────────────methodC2()──▶│
  │                │◀─────────────────────────result───│
  │◀──final result─│              │          │          │
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Facade — প্রাথমিক কাঠামো

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ===== Subsystem Classes =====

class SubsystemA
{
    public function operationA1(): string
    {
        return "SubsystemA: অপারেশন A1 সম্পন্ন";
    }

    public function operationA2(): string
    {
        return "SubsystemA: অপারেশন A2 সম্পন্ন";
    }
}

class SubsystemB
{
    public function operationB1(): string
    {
        return "SubsystemB: অপারেশন B1 সম্পন্ন";
    }
}

class SubsystemC
{
    public function operationC1(): string
    {
        return "SubsystemC: অপারেশন C1 সম্পন্ন";
    }

    public function operationC2(string $data): string
    {
        return "SubsystemC: '{$data}' দিয়ে C2 সম্পন্ন";
    }
}

// ===== Facade =====

class Facade
{
    public function __construct(
        private readonly SubsystemA $subsystemA = new SubsystemA(),
        private readonly SubsystemB $subsystemB = new SubsystemB(),
        private readonly SubsystemC $subsystemC = new SubsystemC(),
    ) {}

    /**
     * Facade সাবসিস্টেমগুলোর জটিল কাজকে একটি সহজ মেথডে wrap করে
     */
    public function simpleOperation(): string
    {
        $results = [];
        $results[] = $this->subsystemA->operationA1();
        $results[] = $this->subsystemB->operationB1();
        $results[] = $this->subsystemC->operationC1();

        return implode("\n", $results);
    }

    public function complexOperation(string $data): string
    {
        $results = [];
        $results[] = $this->subsystemA->operationA1();
        $results[] = $this->subsystemA->operationA2();
        $results[] = $this->subsystemB->operationB1();
        $results[] = $this->subsystemC->operationC2($data);

        return implode("\n", $results);
    }
}

// ===== Client Code =====
$facade = new Facade();
echo $facade->simpleOperation();
echo "\n---\n";
echo $facade->complexOperation("টেস্ট ডেটা");
```

#### JavaScript (ES2022+)

```javascript
// ===== Subsystem Classes =====

class SubsystemA {
  operationA1() {
    return "SubsystemA: অপারেশন A1 সম্পন্ন";
  }

  operationA2() {
    return "SubsystemA: অপারেশন A2 সম্পন্ন";
  }
}

class SubsystemB {
  operationB1() {
    return "SubsystemB: অপারেশন B1 সম্পন্ন";
  }
}

class SubsystemC {
  operationC1() {
    return "SubsystemC: অপারেশন C1 সম্পন্ন";
  }

  operationC2(data) {
    return `SubsystemC: '${data}' দিয়ে C2 সম্পন্ন`;
  }
}

// ===== Facade =====

class Facade {
  #subsystemA;
  #subsystemB;
  #subsystemC;

  constructor(
    subsystemA = new SubsystemA(),
    subsystemB = new SubsystemB(),
    subsystemC = new SubsystemC()
  ) {
    this.#subsystemA = subsystemA;
    this.#subsystemB = subsystemB;
    this.#subsystemC = subsystemC;
  }

  simpleOperation() {
    return [
      this.#subsystemA.operationA1(),
      this.#subsystemB.operationB1(),
      this.#subsystemC.operationC1(),
    ].join("\n");
  }

  complexOperation(data) {
    return [
      this.#subsystemA.operationA1(),
      this.#subsystemA.operationA2(),
      this.#subsystemB.operationB1(),
      this.#subsystemC.operationC2(data),
    ].join("\n");
  }
}

// ===== Client Code =====
const facade = new Facade();
console.log(facade.simpleOperation());
console.log("---");
console.log(facade.complexOperation("টেস্ট ডেটা"));
```

---

### 2️⃣ Order Processing Facade — ই-কমার্স অর্ডার প্রসেসিং

এটি একটি বাস্তবসম্মত উদাহরণ যেখানে একটি অর্ডার প্রসেস করতে **Inventory, Payment, Shipping, এবং Notification** — এই চারটি সাবসিস্টেমকে সমন্বয় করতে হয়।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ===== DTO (Data Transfer Object) =====

readonly class OrderRequest
{
    public function __construct(
        public string $orderId,
        public string $customerId,
        public string $customerPhone,
        public array $items,        // ['product_id' => quantity]
        public float $totalAmount,
        public string $paymentMethod, // 'bkash', 'card', 'cod'
        public string $shippingAddress,
    ) {}
}

readonly class OrderResult
{
    public function __construct(
        public bool $success,
        public string $orderId,
        public string $trackingNumber,
        public string $message,
        public ?string $errorReason = null,
    ) {}
}

// ===== Subsystem: Inventory =====

class InventoryService
{
    private array $stock = [
        'PROD-001' => 50,
        'PROD-002' => 30,
        'PROD-003' => 0,
    ];

    public function checkAvailability(array $items): bool
    {
        foreach ($items as $productId => $quantity) {
            if (!isset($this->stock[$productId]) || $this->stock[$productId] < $quantity) {
                return false;
            }
        }
        return true;
    }

    public function reserveItems(array $items): string
    {
        foreach ($items as $productId => $quantity) {
            $this->stock[$productId] -= $quantity;
        }
        return 'RES-' . bin2hex(random_bytes(4));
    }

    public function releaseReservation(string $reservationId): void
    {
        // রিজার্ভেশন বাতিল করে স্টক ফেরত দেওয়া হয়
        echo "InventoryService: রিজার্ভেশন {$reservationId} রিলিজ করা হয়েছে\n";
    }
}

// ===== Subsystem: Payment =====

class PaymentGateway
{
    public function validatePaymentMethod(string $method, string $customerId): bool
    {
        // bKash, card, COD — সবই ভ্যালিড ধরছি
        return in_array($method, ['bkash', 'card', 'cod'], true);
    }

    public function processPayment(
        string $customerId,
        float $amount,
        string $method,
    ): array {
        // বাস্তবে এখানে bKash API / SSL Commerz / Stripe কল হবে
        return [
            'success' => true,
            'transactionId' => 'TXN-' . bin2hex(random_bytes(6)),
            'gateway' => strtoupper($method),
        ];
    }

    public function refund(string $transactionId, float $amount): bool
    {
        echo "PaymentGateway: {$transactionId} এর জন্য ৳{$amount} রিফান্ড করা হচ্ছে\n";
        return true;
    }
}

// ===== Subsystem: Shipping =====

class ShippingService
{
    public function calculateShippingCost(string $address): float
    {
        // ঢাকার ভেতরে কম, বাইরে বেশি
        return str_contains(strtolower($address), 'dhaka') ? 60.0 : 120.0;
    }

    public function createShipment(
        string $orderId,
        string $address,
        array $items,
    ): string {
        // Pathao / RedX / SteadFast কুরিয়ার API কল
        return 'TRACK-' . strtoupper(bin2hex(random_bytes(5)));
    }

    public function getEstimatedDelivery(string $address): string
    {
        return str_contains(strtolower($address), 'dhaka')
            ? '১-২ কার্যদিবস'
            : '৩-৫ কার্যদিবস';
    }
}

// ===== Subsystem: Notification =====

class NotificationService
{
    public function sendSms(string $phone, string $message): void
    {
        echo "SMS [{$phone}]: {$message}\n";
    }

    public function sendEmail(string $email, string $subject, string $body): void
    {
        echo "Email [{$email}]: {$subject}\n";
    }

    public function sendPushNotification(string $userId, string $message): void
    {
        echo "Push [{$userId}]: {$message}\n";
    }
}

// ===== ORDER PROCESSING FACADE =====

class OrderProcessingFacade
{
    public function __construct(
        private readonly InventoryService $inventory = new InventoryService(),
        private readonly PaymentGateway $payment = new PaymentGateway(),
        private readonly ShippingService $shipping = new ShippingService(),
        private readonly NotificationService $notification = new NotificationService(),
    ) {}

    /**
     * সম্পূর্ণ অর্ডার প্রসেসিং — একটি মাত্র মেথড কলে সব কিছু হ্যান্ডেল
     *
     * অভ্যন্তরে যা ঘটে:
     * 1. ইনভেন্টরি চেক → 2. পেমেন্ট ভ্যালিডেশন → 3. স্টক রিজার্ভ
     * → 4. পেমেন্ট প্রসেস → 5. শিপমেন্ট তৈরি → 6. নোটিফিকেশন
     */
    public function placeOrder(OrderRequest $request): OrderResult
    {
        // ধাপ ১: ইনভেন্টরি চেক
        if (!$this->inventory->checkAvailability($request->items)) {
            return new OrderResult(
                success: false,
                orderId: $request->orderId,
                trackingNumber: '',
                message: 'অর্ডার ব্যর্থ',
                errorReason: 'পণ্য স্টকে নেই',
            );
        }

        // ধাপ ২: পেমেন্ট মেথড ভ্যালিডেশন
        if (!$this->payment->validatePaymentMethod($request->paymentMethod, $request->customerId)) {
            return new OrderResult(
                success: false,
                orderId: $request->orderId,
                trackingNumber: '',
                message: 'অর্ডার ব্যর্থ',
                errorReason: 'অবৈধ পেমেন্ট পদ্ধতি',
            );
        }

        // ধাপ ৩: স্টক রিজার্ভ
        $reservationId = $this->inventory->reserveItems($request->items);

        // ধাপ ৪: পেমেন্ট প্রসেস
        $paymentResult = $this->payment->processPayment(
            $request->customerId,
            $request->totalAmount,
            $request->paymentMethod,
        );

        if (!$paymentResult['success']) {
            // পেমেন্ট ব্যর্থ হলে রিজার্ভেশন ছেড়ে দাও (rollback)
            $this->inventory->releaseReservation($reservationId);

            return new OrderResult(
                success: false,
                orderId: $request->orderId,
                trackingNumber: '',
                message: 'অর্ডার ব্যর্থ',
                errorReason: 'পেমেন্ট প্রসেস ব্যর্থ',
            );
        }

        // ধাপ ৫: শিপমেন্ট তৈরি
        $trackingNumber = $this->shipping->createShipment(
            $request->orderId,
            $request->shippingAddress,
            $request->items,
        );

        $estimatedDelivery = $this->shipping->getEstimatedDelivery($request->shippingAddress);

        // ধাপ ৬: নোটিফিকেশন পাঠাও
        $this->notification->sendSms(
            $request->customerPhone,
            "আপনার অর্ডার #{$request->orderId} নিশ্চিত হয়েছে। ট্র্যাকিং: {$trackingNumber}। আনুমানিক ডেলিভারি: {$estimatedDelivery}",
        );

        return new OrderResult(
            success: true,
            orderId: $request->orderId,
            trackingNumber: $trackingNumber,
            message: "অর্ডার সফলভাবে সম্পন্ন। ট্র্যাকিং: {$trackingNumber}",
        );
    }

    /**
     * শুধু শিপিং খরচ জানতে চাইলে — Facade-এর আরেকটি সরলীকৃত মেথড
     */
    public function getShippingEstimate(string $address): array
    {
        return [
            'cost' => $this->shipping->calculateShippingCost($address),
            'estimated_delivery' => $this->shipping->getEstimatedDelivery($address),
        ];
    }
}

// ===== ব্যবহার =====

$facade = new OrderProcessingFacade();

$order = new OrderRequest(
    orderId: 'ORD-2024-001',
    customerId: 'CUST-42',
    customerPhone: '+8801712345678',
    items: ['PROD-001' => 2, 'PROD-002' => 1],
    totalAmount: 2500.00,
    paymentMethod: 'bkash',
    shippingAddress: 'বনানী, Dhaka-1213',
);

$result = $facade->placeOrder($order);
echo $result->success ? "✅ {$result->message}" : "❌ {$result->errorReason}";
```

#### JavaScript (ES2022+)

```javascript
// ===== Subsystem: Inventory =====

class InventoryService {
  #stock = new Map([
    ["PROD-001", 50],
    ["PROD-002", 30],
    ["PROD-003", 0],
  ]);

  checkAvailability(items) {
    return Object.entries(items).every(
      ([id, qty]) => (this.#stock.get(id) ?? 0) >= qty
    );
  }

  reserveItems(items) {
    for (const [id, qty] of Object.entries(items)) {
      this.#stock.set(id, this.#stock.get(id) - qty);
    }
    return `RES-${crypto.randomUUID().slice(0, 8)}`;
  }

  releaseReservation(reservationId) {
    console.log(`InventoryService: রিজার্ভেশন ${reservationId} রিলিজ হয়েছে`);
  }
}

// ===== Subsystem: Payment =====

class PaymentGateway {
  #validMethods = new Set(["bkash", "card", "cod", "nagad"]);

  validatePaymentMethod(method) {
    return this.#validMethods.has(method);
  }

  async processPayment(customerId, amount, method) {
    // বাস্তবে bKash / Nagad / SSLCommerz async API কল
    await new Promise((r) => setTimeout(r, 100));
    return {
      success: true,
      transactionId: `TXN-${crypto.randomUUID().slice(0, 12)}`,
      gateway: method.toUpperCase(),
    };
  }

  async refund(transactionId, amount) {
    console.log(`PaymentGateway: ${transactionId} → ৳${amount} রিফান্ড`);
    return true;
  }
}

// ===== Subsystem: Shipping =====

class ShippingService {
  calculateCost(address) {
    return address.toLowerCase().includes("dhaka") ? 60 : 120;
  }

  async createShipment(orderId, address, items) {
    await new Promise((r) => setTimeout(r, 50));
    return `TRACK-${crypto.randomUUID().slice(0, 10).toUpperCase()}`;
  }

  getEstimatedDelivery(address) {
    return address.toLowerCase().includes("dhaka")
      ? "১-২ কার্যদিবস"
      : "৩-৫ কার্যদিবস";
  }
}

// ===== Subsystem: Notification =====

class NotificationService {
  async sendSms(phone, message) {
    console.log(`SMS [${phone}]: ${message}`);
  }

  async sendPush(userId, message) {
    console.log(`Push [${userId}]: ${message}`);
  }
}

// ===== ORDER PROCESSING FACADE =====

class OrderProcessingFacade {
  #inventory;
  #payment;
  #shipping;
  #notification;

  constructor({
    inventory = new InventoryService(),
    payment = new PaymentGateway(),
    shipping = new ShippingService(),
    notification = new NotificationService(),
  } = {}) {
    this.#inventory = inventory;
    this.#payment = payment;
    this.#shipping = shipping;
    this.#notification = notification;
  }

  async placeOrder({ orderId, customerId, customerPhone, items, totalAmount, paymentMethod, shippingAddress }) {
    // ধাপ ১: ইনভেন্টরি চেক
    if (!this.#inventory.checkAvailability(items)) {
      return { success: false, orderId, error: "পণ্য স্টকে নেই" };
    }

    // ধাপ ২: পেমেন্ট মেথড ভ্যালিডেশন
    if (!this.#payment.validatePaymentMethod(paymentMethod)) {
      return { success: false, orderId, error: "অবৈধ পেমেন্ট পদ্ধতি" };
    }

    // ধাপ ৩: স্টক রিজার্ভ
    const reservationId = this.#inventory.reserveItems(items);

    // ধাপ ৪: পেমেন্ট প্রসেস
    const paymentResult = await this.#payment.processPayment(
      customerId, totalAmount, paymentMethod
    );

    if (!paymentResult.success) {
      this.#inventory.releaseReservation(reservationId);
      return { success: false, orderId, error: "পেমেন্ট ব্যর্থ" };
    }

    // ধাপ ৫: শিপমেন্ট তৈরি
    const trackingNumber = await this.#shipping.createShipment(
      orderId, shippingAddress, items
    );
    const estimatedDelivery = this.#shipping.getEstimatedDelivery(shippingAddress);

    // ধাপ ৬: নোটিফিকেশন
    await this.#notification.sendSms(
      customerPhone,
      `অর্ডার #${orderId} নিশ্চিত। ট্র্যাকিং: ${trackingNumber}। ডেলিভারি: ${estimatedDelivery}`
    );

    return {
      success: true,
      orderId,
      trackingNumber,
      estimatedDelivery,
      transactionId: paymentResult.transactionId,
    };
  }

  getShippingEstimate(address) {
    return {
      cost: this.#shipping.calculateCost(address),
      estimatedDelivery: this.#shipping.getEstimatedDelivery(address),
    };
  }
}

// ===== ব্যবহার =====
const facade = new OrderProcessingFacade();

const result = await facade.placeOrder({
  orderId: "ORD-2024-001",
  customerId: "CUST-42",
  customerPhone: "+8801712345678",
  items: { "PROD-001": 2, "PROD-002": 1 },
  totalAmount: 2500,
  paymentMethod: "bkash",
  shippingAddress: "বনানী, Dhaka-1213",
});

console.log(result.success ? `✅ ট্র্যাকিং: ${result.trackingNumber}` : `❌ ${result.error}`);
```

---

### 3️⃣ Home Theater Facade — হোম থিয়েটার সিস্টেম

ক্লাসিক Facade উদাহরণ — একটি মুভি দেখতে গেলে TV চালু করা, স্পিকার সেট করা, লাইট কমানো, স্ট্রিমিং সার্ভিস চালু — সবকিছু একটি `watchMovie()` কলে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

class SmartTV
{
    public function on(): void { echo "📺 TV চালু হলো\n"; }
    public function off(): void { echo "📺 TV বন্ধ হলো\n"; }
    public function setInput(string $input): void { echo "📺 ইনপুট: {$input}\n"; }
    public function setResolution(string $res): void { echo "📺 রেজোলিউশন: {$res}\n"; }
}

class SoundSystem
{
    public function on(): void { echo "🔊 সাউন্ড সিস্টেম চালু\n"; }
    public function off(): void { echo "🔊 সাউন্ড সিস্টেম বন্ধ\n"; }
    public function setVolume(int $level): void { echo "🔊 ভলিউম: {$level}%\n"; }
    public function setSurroundMode(string $mode): void { echo "🔊 সারাউন্ড: {$mode}\n"; }
}

class SmartLights
{
    public function dim(int $level): void { echo "💡 লাইট: {$level}%\n"; }
    public function setColor(string $color): void { echo "💡 রং: {$color}\n"; }
    public function fullBright(): void { echo "💡 পূর্ণ আলো\n"; }
}

class StreamingService
{
    public function login(string $service): void { echo "🎬 {$service} লগইন সফল\n"; }
    public function searchMovie(string $title): string { echo "🎬 '{$title}' খুঁজে পাওয়া গেছে\n"; return $title; }
    public function play(string $title): void { echo "🎬 ▶️ '{$title}' চলছে...\n"; }
    public function stop(): void { echo "🎬 ⏹️ প্লেব্যাক বন্ধ\n"; }
}

// ===== HOME THEATER FACADE =====

class HomeTheaterFacade
{
    public function __construct(
        private readonly SmartTV $tv = new SmartTV(),
        private readonly SoundSystem $sound = new SoundSystem(),
        private readonly SmartLights $lights = new SmartLights(),
        private readonly StreamingService $streaming = new StreamingService(),
    ) {}

    public function watchMovie(string $title, string $service = 'Netflix'): void
    {
        echo "🎬 === মুভি নাইট শুরু === 🎬\n\n";

        $this->tv->on();
        $this->tv->setInput('HDMI-1');
        $this->tv->setResolution('4K HDR');

        $this->sound->on();
        $this->sound->setVolume(65);
        $this->sound->setSurroundMode('Dolby Atmos');

        $this->lights->dim(15);
        $this->lights->setColor('warm-amber');

        $this->streaming->login($service);
        $this->streaming->searchMovie($title);
        $this->streaming->play($title);

        echo "\n🍿 উপভোগ করুন!\n";
    }

    public function endMovie(): void
    {
        echo "\n🎬 === মুভি শেষ === 🎬\n\n";

        $this->streaming->stop();
        $this->sound->off();
        $this->lights->fullBright();
        $this->tv->off();
    }
}

// ===== ব্যবহার =====
$theater = new HomeTheaterFacade();
$theater->watchMovie('Oppenheimer');
$theater->endMovie();
```

#### JavaScript (ES2022+)

```javascript
class SmartTV {
  on() { console.log("📺 TV চালু"); }
  off() { console.log("📺 TV বন্ধ"); }
  setInput(input) { console.log(`📺 ইনপুট: ${input}`); }
  setResolution(res) { console.log(`📺 রেজোলিউশন: ${res}`); }
}

class SoundSystem {
  on() { console.log("🔊 সাউন্ড চালু"); }
  off() { console.log("🔊 সাউন্ড বন্ধ"); }
  setVolume(level) { console.log(`🔊 ভলিউম: ${level}%`); }
  setSurround(mode) { console.log(`🔊 সারাউন্ড: ${mode}`); }
}

class SmartLights {
  dim(level) { console.log(`💡 লাইট: ${level}%`); }
  setColor(color) { console.log(`💡 রং: ${color}`); }
  fullBright() { console.log("💡 পূর্ণ আলো"); }
}

class StreamingService {
  login(svc) { console.log(`🎬 ${svc} লগইন সফল`); }
  play(title) { console.log(`🎬 ▶️ '${title}' চলছে...`); }
  stop() { console.log("🎬 ⏹️ বন্ধ"); }
}

// ===== FACADE =====

class HomeTheaterFacade {
  #tv; #sound; #lights; #streaming;

  constructor({
    tv = new SmartTV(),
    sound = new SoundSystem(),
    lights = new SmartLights(),
    streaming = new StreamingService(),
  } = {}) {
    this.#tv = tv;
    this.#sound = sound;
    this.#lights = lights;
    this.#streaming = streaming;
  }

  watchMovie(title, service = "Netflix") {
    console.log("🎬 === মুভি নাইট শুরু ===\n");

    this.#tv.on();
    this.#tv.setInput("HDMI-1");
    this.#tv.setResolution("4K HDR");

    this.#sound.on();
    this.#sound.setVolume(65);
    this.#sound.setSurround("Dolby Atmos");

    this.#lights.dim(15);
    this.#lights.setColor("warm-amber");

    this.#streaming.login(service);
    this.#streaming.play(title);

    console.log("\n🍿 উপভোগ করুন!");
  }

  endMovie() {
    console.log("\n🎬 === মুভি শেষ ===\n");
    this.#streaming.stop();
    this.#sound.off();
    this.#lights.fullBright();
    this.#tv.off();
  }
}

// ===== ব্যবহার =====
const theater = new HomeTheaterFacade();
theater.watchMovie("Oppenheimer");
theater.endMovie();
```

---

## 🌍 Real-World Applicable Areas

### 1. Laravel Facades — Cache::, Auth::, Mail::

Laravel-এর Facade সিস্টেম একটি চমৎকার উদাহরণ। `Cache::get('key')` লেখার সময় আপনি আসলে Service Container থেকে resolve হওয়া একটি cache manager instance-এর `get()` মেথড কল করছেন।

```php
// আপনি লেখেন:
Cache::put('user_42', $userData, 3600);
$user = Cache::get('user_42');

// অভ্যন্তরে যা ঘটে:
// 1. Facade::__callStatic('get', ['user_42']) কল হয়
// 2. static::getFacadeRoot() → Service Container থেকে 'cache' resolve হয়
// 3. CacheManager->store()->get('user_42') কল হয়
// 4. Redis/Memcached/File driver অনুযায়ী ডেটা আসে

// Auth Facade:
Auth::attempt(['email' => $email, 'password' => $password]);
// → AuthManager → SessionGuard → UserProvider → Database query

// Mail Facade:
Mail::to('user@example.com')->send(new OrderConfirmation($order));
// → MailManager → Transport(SMTP/SES/Mailgun) → Message build → Send
```

### 2. Payment Processing Facade

```php
// bKash + Nagad + SSL Commerz — একটি Facade দিয়ে হ্যান্ডেল
class PaymentFacade
{
    public function charge(PaymentRequest $request): PaymentResponse
    {
        // ১. ভ্যালিডেশন (amount, currency, method)
        // ২. ফ্রড ডিটেকশন চেক
        // ৩. সঠিক গেটওয়ে সিলেক্ট (bKash/Nagad/Card)
        // ৪. পেমেন্ট প্রসেস
        // ৫. রসিদ জেনারেট
        // ৬. SMS/Email নোটিফিকেশন
        // ৭. অ্যাকাউন্টিং লেজার আপডেট
    }
}
```

### 3. File Upload Facade

```php
class FileUploadFacade
{
    public function upload(UploadedFile $file, string $directory): UploadResult
    {
        // ১. ভ্যালিডেশন (size, mime-type, malware scan)
        // ২. ইমেজ হলে রিসাইজ ও অপ্টিমাইজ
        // ৩. ইউনিক নাম জেনারেট
        // ৪. S3/DigitalOcean Spaces-এ আপলোড
        // ৫. CDN invalidation
        // ৬. ডেটাবেজে মেটাডেটা সেভ
        // ৭. থাম্বনেইল জেনারেট (queue-এ পাঠানো)
    }
}
```

### 4. Report Generation Facade

```php
class ReportFacade
{
    public function generateMonthlySalesReport(int $month, int $year): Report
    {
        // ১. ডেটাবেজ থেকে sales data query
        // ২. ক্যালকুলেশন engine দিয়ে aggregate
        // ৩. চার্ট/গ্রাফ জেনারেট
        // ৪. PDF/Excel তৈরি
        // ৫. ক্লাউড স্টোরেজে সেভ
        // ৬. ইমেইলে পাঠানো
    }
}
```

### 5. Third-Party API Integration Facade

```javascript
// বাংলাদেশী SMS গেটওয়ে — বিভিন্ন প্রোভাইডার
class SmsFacade {
  #providers; // BulkSMSBD, MiMSMS, SSL Wireless

  async send(phone, message, options = {}) {
    // ১. ফোন নম্বর ভ্যালিডেশন (বাংলাদেশী ফরম্যাট)
    // ২. মেসেজ encoding চেক (বাংলা/ইংলিশ)
    // ৩. সবচেয়ে সস্তা/available প্রোভাইডার সিলেক্ট
    // ৪. API কল
    // ৫. ডেলিভারি স্ট্যাটাস ট্র্যাক
    // ৬. ব্যালেন্স চেক ও alert
  }
}
```

### 6. Database Migration Facade

```php
class MigrationFacade
{
    public function migrateToNewSchema(): MigrationResult
    {
        // ১. বর্তমান স্কিমা ব্যাকআপ
        // ২. নতুন টেবিল তৈরি
        // ৩. ডেটা ট্রান্সফর্ম ও মাইগ্রেট
        // ৪. ইনডেক্স তৈরি
        // ৫. ফরেন কী যোগ
        // ৬. ভ্যালিডেশন (row count match)
        // ৭. পুরানো টেবিল আর্কাইভ
    }
}
```

### 7. E-Commerce Checkout Facade — সম্পূর্ণ উদাহরণ

```php
<?php

declare(strict_types=1);

/**
 * বাংলাদেশের ই-কমার্স সাইটের checkout প্রসেস
 * — দারাজ/চালডাল/ইভ্যালি টাইপ প্ল্যাটফর্ম
 */
class ECommerceCheckoutFacade
{
    public function __construct(
        private readonly CartService $cart,
        private readonly CouponService $coupon,
        private readonly TaxCalculator $tax,
        private readonly InventoryService $inventory,
        private readonly PaymentFacade $payment,      // নেস্টেড Facade!
        private readonly ShippingService $shipping,
        private readonly InvoiceGenerator $invoice,
        private readonly NotificationService $notifier,
        private readonly AnalyticsService $analytics,
    ) {}

    public function checkout(string $cartId, CheckoutRequest $request): CheckoutResult
    {
        // ১. কার্ট ভ্যালিডেট
        $cart = $this->cart->getValidatedCart($cartId);

        // ২. কুপন প্রয়োগ
        $discount = $request->couponCode
            ? $this->coupon->apply($request->couponCode, $cart)
            : 0.0;

        // ৩. ভ্যাট/ট্যাক্স ক্যালকুলেট (বাংলাদেশে ৫-১৫% VAT)
        $tax = $this->tax->calculate($cart, $request->shippingAddress);

        // ৪. শিপিং খরচ
        $shippingCost = $this->shipping->calculateCost(
            $cart->getTotalWeight(),
            $request->shippingAddress,
        );

        // ৫. চূড়ান্ত মূল্য
        $finalAmount = $cart->getSubtotal() - $discount + $tax + $shippingCost;

        // ৬. স্টক রিজার্ভ
        $reservation = $this->inventory->reserveAll($cart->getItems());

        // ৭. পেমেন্ট (নেস্টেড Facade ব্যবহার)
        $paymentResult = $this->payment->charge(new PaymentRequest(
            amount: $finalAmount,
            method: $request->paymentMethod,
            customerPhone: $request->phone,
        ));

        if (!$paymentResult->success) {
            $this->inventory->releaseAll($reservation);
            return CheckoutResult::failed($paymentResult->error);
        }

        // ৮. ইনভয়েস জেনারেট
        $invoicePdf = $this->invoice->generate($cart, $paymentResult, $tax);

        // ৯. নোটিফিকেশন
        $this->notifier->sendOrderConfirmation($request->phone, $cart->getOrderId());

        // ১০. অ্যানালিটিক্স ইভেন্ট
        $this->analytics->trackPurchase($cart->getOrderId(), $finalAmount);

        return CheckoutResult::success(
            orderId: $cart->getOrderId(),
            invoiceUrl: $invoicePdf->getUrl(),
            trackingNumber: $paymentResult->trackingNumber,
        );
    }
}
```

---

## 🔥 Advanced Deep Dive

### 1. Laravel Facade Deep Dive — ম্যাজিক __callStatic

Laravel-এর Facade আসলে GoF Facade pattern-এর সরাসরি বাস্তবায়ন নয় — এটি **Static Proxy Pattern**। তবে ধারণাগতভাবে একই কাজ করে: জটিলতা লুকানো।

```php
<?php

declare(strict_types=1);

/**
 * Laravel-এর Facade ক্লাসের সরলীকৃত অভ্যন্তরীণ কাঠামো
 */
abstract class LaravelStyleFacade
{
    /** Service Container-এর অনুকরণ */
    protected static array $resolvedInstances = [];
    protected static ?object $container = null;

    /**
     * ম্যাজিক! — যেকোনো static মেথড কল এখানে আসে
     * Cache::get('key') → __callStatic('get', ['key'])
     */
    public static function __callStatic(string $method, array $args): mixed
    {
        $instance = static::getFacadeRoot();

        if (!$instance) {
            throw new \RuntimeException('Facade root has not been set.');
        }

        // resolved instance-এর উপর মেথড কল forward করা হচ্ছে
        return $instance->$method(...$args);
    }

    /**
     * Service Container থেকে আসল instance resolve করে
     */
    public static function getFacadeRoot(): object
    {
        $name = static::getFacadeAccessor();

        if (!isset(static::$resolvedInstances[$name])) {
            // বাস্তব Laravel-এ: app()->make($name)
            static::$resolvedInstances[$name] = static::resolveFacadeInstance($name);
        }

        return static::$resolvedInstances[$name];
    }

    /**
     * প্রতিটি Facade ক্লাস এটি override করে
     * Cache → 'cache', Auth → 'auth', Mail → 'mailer'
     */
    abstract protected static function getFacadeAccessor(): string;

    protected static function resolveFacadeInstance(string $name): object
    {
        // Container থেকে resolve — এখানে সরলীকৃত
        return static::$container?->make($name)
            ?? throw new \RuntimeException("Cannot resolve [{$name}]");
    }

    /**
     * টেস্টিংয়ের জন্য — instance swap করা যায়
     */
    public static function swap(object $instance): void
    {
        $name = static::getFacadeAccessor();
        static::$resolvedInstances[$name] = $instance;
    }
}

// উদাহরণ: Cache Facade
class Cache extends LaravelStyleFacade
{
    protected static function getFacadeAccessor(): string
    {
        return 'cache';
    }
}

// এখন Cache::get('key') কল করলে:
// → __callStatic('get', ['key'])
// → getFacadeRoot() → container->make('cache') → CacheManager instance
// → CacheManager->get('key')
```

### 2. Facade vs Adapter vs Proxy — পার্থক্য

```
┌─────────────┬───────────────────────────────────────────────────────┐
│   Pattern   │ উদ্দেশ্য                                              │
├─────────────┼───────────────────────────────────────────────────────┤
│   Facade    │ জটিল সাবসিস্টেমকে সরলীকৃত ইন্টারফেস দেওয়া            │
│             │ → নতুন, সহজ API তৈরি করে (একাধিক ক্লাসের উপর)         │
│             │                                                       │
│   Adapter   │ incompatible ইন্টারফেসকে compatible করা               │
│             │ → একটি ক্লাসের ইন্টারফেস পরিবর্তন করে                  │
│             │                                                       │
│   Proxy     │ আসল অবজেক্টের access নিয়ন্ত্রণ করা                    │
│             │ → একই ইন্টারফেস রেখে behavior যোগ করে                  │
│             │ (lazy loading, caching, logging, access control)       │
└─────────────┴───────────────────────────────────────────────────────┘
```

**মনে রাখার সূত্র:**
- **Facade** = "আমি সবকিছু সহজ করে দিচ্ছি" (simplification)
- **Adapter** = "আমি ভাষা অনুবাদ করছি" (translation)
- **Proxy** = "আমি পাহারাদার / মধ্যস্থতাকারী" (control)

### 3. Facade vs Service Layer

```
┌─────────────────┬──────────────────────────────────────────┐
│    Facade        │    Service Layer                         │
├─────────────────┼──────────────────────────────────────────┤
│ সাবসিস্টেমের    │ ডোমেইন লজিক ও ব্যবসায়িক নিয়ম           │
│ উপর সরল wrapper  │ encapsulate করে                          │
│                  │                                          │
│ কোনো নতুন logic  │ নিজস্ব business logic                    │
│ যোগ করে না       │ থাকতে পারে                               │
│                  │                                          │
│ Subsystem-এর     │ Domain model ও                           │
│ মেথড delegate    │ repository-র মাঝে                        │
│ করে              │ orchestration করে                        │
└─────────────────┴──────────────────────────────────────────┘
```

বাস্তবে অনেক সময় Service Layer নিজেই একটি Facade হিসেবে কাজ করে। তবে **ধারণাগত পার্থক্য** হলো: Facade-এ নতুন business logic নেই — শুধু delegation আছে। Service Layer-এ business rule enforcement থাকে।

### 4. Anti-Corruption Layer (DDD) — Facade ব্যবহার

Domain-Driven Design-এ যখন আপনার bounded context বাইরের একটি legacy বা third-party সিস্টেমের সাথে যোগাযোগ করে, তখন **Anti-Corruption Layer** তৈরি করা হয় যাতে বাইরের সিস্টেমের ডেটা মডেল আপনার ডোমেইনকে "দূষিত" না করে। এটি মূলত Facade + Adapter-এর সমন্বয়।

```php
<?php

declare(strict_types=1);

/**
 * Legacy ERP সিস্টেমের Anti-Corruption Layer
 * — পুরানো সিস্টেমের জটিল API-কে আমাদের domain model-এ translate করে
 */
class LegacyErpFacade
{
    public function __construct(
        private readonly LegacyErpSoapClient $soapClient,
        private readonly ErpDataTransformer $transformer,
        private readonly ErpResponseValidator $validator,
    ) {}

    /**
     * Legacy ERP থেকে প্রোডাক্ট ডেটা আমাদের Product entity-তে রূপান্তর
     */
    public function getProduct(string $sku): Product
    {
        // Legacy SOAP কল — XML response
        $rawResponse = $this->soapClient->call('GetInventoryItem', [
            'ItemCode' => $sku,
            'WarehouseCode' => 'WH-DHK-01',
            'IncludePricing' => 1,
            'PriceListCode' => 'RETAIL-BD',
        ]);

        // Response ভ্যালিডেশন
        $this->validator->validate($rawResponse);

        // Legacy ডেটা → আমাদের domain model
        return $this->transformer->toProduct($rawResponse);
    }

    /**
     * আমাদের Order entity → Legacy ERP-এর ফরম্যাটে পাঠানো
     */
    public function syncOrder(Order $order): void
    {
        $erpOrder = $this->transformer->toLegacyOrderFormat($order);
        $this->soapClient->call('CreateSalesOrder', $erpOrder);
    }
}
```

### 5. Legacy System Wrapping

```javascript
/**
 * পুরানো jQuery/callback-based কোডকে আধুনিক Promise-based Facade দিয়ে wrap
 */
class LegacyApiFacade {
  #baseUrl;
  #legacyAuth;

  constructor(baseUrl, legacyAuth) {
    this.#baseUrl = baseUrl;
    this.#legacyAuth = legacyAuth;
  }

  async getUser(userId) {
    // Legacy সিস্টেম XML ফেরত দেয়, আমরা JSON-এ convert করি
    const xmlResponse = await this.#legacyXmlRequest("GET", `/users/${userId}`);
    const parsed = this.#parseXmlToJson(xmlResponse);

    // Legacy ফিল্ড নাম → আধুনিক ফিল্ড নাম
    return {
      id: parsed.USR_ID,
      name: parsed.USR_FULL_NM,
      email: parsed.USR_EMAIL_ADDR,
      phone: parsed.USR_PHONE_NUM,
      createdAt: new Date(parsed.CRT_DTTM),
    };
  }

  async #legacyXmlRequest(method, path) {
    const response = await fetch(`${this.#baseUrl}${path}`, {
      method,
      headers: {
        "X-Legacy-Token": this.#legacyAuth.getToken(),
        Accept: "application/xml",
      },
    });
    return response.text();
  }

  #parseXmlToJson(xml) {
    const parser = new DOMParser();
    const doc = parser.parseFromString(xml, "text/xml");
    // ... XML → JSON parsing logic
    return {};
  }
}
```

### 6. Nested/Layered Facades

বড় সিস্টেমে একটি Facade অন্য Facade ব্যবহার করতে পারে — এটাকে **Layered Facade** বলে।

```
┌──────────────────────────────────────────────────┐
│               Application Facade                  │
│         (checkout, dashboard, reporting)          │
│                      │                            │
│         ┌────────────┼────────────┐               │
│         ▼            ▼            ▼               │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│   │ Payment  │ │ Shipping │ │Inventory │        │
│   │ Facade   │ │ Facade   │ │ Facade   │        │
│   │    │     │ │    │     │ │    │     │        │
│   │    ▼     │ │    ▼     │ │    ▼     │        │
│   │ bKash    │ │ Pathao   │ │ MySQL    │        │
│   │ Nagad    │ │ RedX     │ │ Redis    │        │
│   │ SSL      │ │ SA       │ │ ES       │        │
│   └──────────┘ └──────────┘ └──────────┘        │
│              Domain/Infrastructure                │
└──────────────────────────────────────────────────┘
```

```php
<?php

declare(strict_types=1);

// নিচের স্তরের Facade
class PaymentFacade
{
    public function __construct(
        private readonly BkashGateway $bkash,
        private readonly NagadGateway $nagad,
        private readonly SslCommerzGateway $ssl,
        private readonly FraudDetector $fraud,
    ) {}

    public function charge(PaymentRequest $request): PaymentResult
    {
        $this->fraud->check($request);
        $gateway = $this->resolveGateway($request->method);
        return $gateway->process($request);
    }

    private function resolveGateway(string $method): PaymentGatewayInterface
    {
        return match($method) {
            'bkash' => $this->bkash,
            'nagad' => $this->nagad,
            default => $this->ssl,
        };
    }
}

// উপরের স্তরের Facade — নিচের Facade ব্যবহার করে
class CheckoutFacade
{
    public function __construct(
        private readonly PaymentFacade $payment,   // নেস্টেড Facade
        private readonly ShippingFacade $shipping,  // নেস্টেড Facade
        private readonly CartService $cart,
    ) {}

    public function processCheckout(string $cartId, CheckoutRequest $req): CheckoutResult
    {
        $cart = $this->cart->get($cartId);
        $shippingCost = $this->shipping->estimate($req->address); // Facade → Facade
        $paymentResult = $this->payment->charge(/*...*/);           // Facade → Facade
        // ...
    }
}
```

---

## ✅ Pros (সুবিধা)

| সুবিধা | বিস্তারিত |
|--------|----------|
| **সরলতা** | ক্লায়েন্ট কোড সহজ ও পরিষ্কার হয় — একটি মাত্র entry point |
| **Loose Coupling** | ক্লায়েন্ট সাবসিস্টেমের অভ্যন্তরীণ পরিবর্তন থেকে সুরক্ষিত |
| **Layered Architecture** | সিস্টেমকে স্তরে ভাগ করতে সাহায্য করে |
| **Legacy Wrapping** | পুরানো কোডকে নতুন ইন্টারফেসে wrap করা সহজ |
| **টেস্টিং সহজ** | Facade mock/stub করে ক্লায়েন্ট কোড সহজে টেস্ট করা যায় |
| **Onboarding** | নতুন ডেভেলপারদের সিস্টেম বুঝতে সহজ হয় |

## ❌ Cons (অসুবিধা)

| অসুবিধা | বিস্তারিত |
|---------|----------|
| **God Object ঝুঁকি** | Facade-এ অতিরিক্ত দায়িত্ব জমা হতে পারে |
| **অতিরিক্ত abstraction** | সহজ সিস্টেমে অপ্রয়োজনীয় জটিলতা যোগ হয় |
| **পারফরম্যান্স** | অতিরিক্ত layer-এ সামান্য overhead যোগ হয় |
| **Flexibility হ্রাস** | power user-দের সাবসিস্টেম সরাসরি access প্রয়োজন হতে পারে |
| **Hidden Complexity** | সমস্যা হলে ডিবাগিং কঠিন হতে পারে কারণ জটিলতা লুকানো |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ১. God Facade — সবকিছু এক জায়গায়

```php
// ❌ ভুল — একটি Facade-এ সবকিছু ঢুকিয়ে দেওয়া
class GodFacade
{
    public function processOrder() { /* ... */ }
    public function generateReport() { /* ... */ }
    public function sendNotification() { /* ... */ }
    public function handlePayment() { /* ... */ }
    public function manageInventory() { /* ... */ }
    public function processRefund() { /* ... */ }
    public function generateInvoice() { /* ... */ }
    public function syncWithErp() { /* ... */ }
    // ৫০+ মেথড... 😱
}

// ✅ সঠিক — আলাদা আলাদা Facade
class OrderFacade { /* অর্ডার সম্পর্কিত */ }
class PaymentFacade { /* পেমেন্ট সম্পর্কিত */ }
class ReportFacade { /* রিপোর্ট সম্পর্কিত */ }
class NotificationFacade { /* নোটিফিকেশন সম্পর্কিত */ }
```

### ২. Facade-এর পেছনে Tight Coupling

```php
// ❌ ভুল — Facade কংক্রিট ক্লাসের উপর নির্ভরশীল
class OrderFacade
{
    public function process(): void
    {
        $db = new MySQLDatabase();         // কংক্রিট!
        $mailer = new SmtpMailer();         // কংক্রিট!
        $logger = new FileLogger();         // কংক্রিট!
    }
}

// ✅ সঠিক — Dependency Injection + Interface
class OrderFacade
{
    public function __construct(
        private readonly DatabaseInterface $db,
        private readonly MailerInterface $mailer,
        private readonly LoggerInterface $logger,
    ) {}
}
```

### ৩. Facade-এ Business Logic রাখা

```php
// ❌ ভুল — Facade-এ business logic
class OrderFacade
{
    public function placeOrder(OrderRequest $req): OrderResult
    {
        // Facade-এ discount calculation — এটা এখানে থাকা উচিত নয়
        $discount = 0;
        if ($req->totalAmount > 5000) $discount = 0.1;
        if ($req->isVipCustomer) $discount += 0.05;
        $finalAmount = $req->totalAmount * (1 - $discount);

        // ... বাকি processing
    }
}

// ✅ সঠিক — Business logic আলাদা service-এ
class OrderFacade
{
    public function __construct(
        private readonly DiscountCalculator $discountCalc,
        // ...
    ) {}

    public function placeOrder(OrderRequest $req): OrderResult
    {
        $finalAmount = $this->discountCalc->calculate($req);
        // ... delegation only
    }
}
```

### ৪. সাবসিস্টেমে সরাসরি Access বন্ধ করে দেওয়া

```php
// ❌ ভুল — ধারণা: "Facade আছে তাই subsystem সরাসরি ব্যবহার করা যাবে না"
// Facade optional হওয়া উচিত — power user-রা সরাসরি subsystem ব্যবহার করতে পারবে

// ✅ সঠিক পন্থা:
// - সাধারণ কাজের জন্য → Facade ব্যবহার করুন
// - advanced/custom কাজের জন্য → সরাসরি subsystem ব্যবহার করুন
```

---

## 🧪 টেস্টিং (Testing)

### PHPUnit — Facade টেস্টিং

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\MockObject\MockObject;

class OrderProcessingFacadeTest extends TestCase
{
    private InventoryService&MockObject $inventory;
    private PaymentGateway&MockObject $payment;
    private ShippingService&MockObject $shipping;
    private NotificationService&MockObject $notification;
    private OrderProcessingFacade $facade;

    protected function setUp(): void
    {
        $this->inventory = $this->createMock(InventoryService::class);
        $this->payment = $this->createMock(PaymentGateway::class);
        $this->shipping = $this->createMock(ShippingService::class);
        $this->notification = $this->createMock(NotificationService::class);

        $this->facade = new OrderProcessingFacade(
            $this->inventory,
            $this->payment,
            $this->shipping,
            $this->notification,
        );
    }

    public function test_successful_order_processing(): void
    {
        // Arrange — সব subsystem সফল হবে
        $this->inventory->method('checkAvailability')->willReturn(true);
        $this->inventory->method('reserveItems')->willReturn('RES-123');
        $this->payment->method('validatePaymentMethod')->willReturn(true);
        $this->payment->method('processPayment')->willReturn([
            'success' => true,
            'transactionId' => 'TXN-456',
        ]);
        $this->shipping->method('createShipment')->willReturn('TRACK-789');
        $this->shipping->method('getEstimatedDelivery')->willReturn('১-২ দিন');

        $request = new OrderRequest(
            orderId: 'ORD-001',
            customerId: 'CUST-1',
            customerPhone: '+8801700000000',
            items: ['PROD-001' => 1],
            totalAmount: 1000.0,
            paymentMethod: 'bkash',
            shippingAddress: 'Dhaka',
        );

        // Act
        $result = $this->facade->placeOrder($request);

        // Assert
        $this->assertTrue($result->success);
        $this->assertEquals('TRACK-789', $result->trackingNumber);
    }

    public function test_order_fails_when_out_of_stock(): void
    {
        $this->inventory->method('checkAvailability')->willReturn(false);

        $request = new OrderRequest(
            orderId: 'ORD-002',
            customerId: 'CUST-1',
            customerPhone: '+8801700000000',
            items: ['PROD-003' => 1],
            totalAmount: 500.0,
            paymentMethod: 'bkash',
            shippingAddress: 'Dhaka',
        );

        $result = $this->facade->placeOrder($request);

        $this->assertFalse($result->success);
        $this->assertEquals('পণ্য স্টকে নেই', $result->errorReason);
    }

    public function test_inventory_released_on_payment_failure(): void
    {
        $this->inventory->method('checkAvailability')->willReturn(true);
        $this->inventory->method('reserveItems')->willReturn('RES-999');
        $this->payment->method('validatePaymentMethod')->willReturn(true);
        $this->payment->method('processPayment')->willReturn(['success' => false]);

        // রিজার্ভেশন রিলিজ হয়েছে কিনা verify
        $this->inventory
            ->expects($this->once())
            ->method('releaseReservation')
            ->with('RES-999');

        $request = new OrderRequest(
            orderId: 'ORD-003',
            customerId: 'CUST-1',
            customerPhone: '+8801700000000',
            items: ['PROD-001' => 1],
            totalAmount: 1000.0,
            paymentMethod: 'bkash',
            shippingAddress: 'Dhaka',
        );

        $result = $this->facade->placeOrder($request);
        $this->assertFalse($result->success);
    }

    public function test_notification_sent_on_success(): void
    {
        $this->inventory->method('checkAvailability')->willReturn(true);
        $this->inventory->method('reserveItems')->willReturn('RES-111');
        $this->payment->method('validatePaymentMethod')->willReturn(true);
        $this->payment->method('processPayment')->willReturn([
            'success' => true,
            'transactionId' => 'TXN-222',
        ]);
        $this->shipping->method('createShipment')->willReturn('TRACK-333');
        $this->shipping->method('getEstimatedDelivery')->willReturn('২-৩ দিন');

        // SMS পাঠানো হয়েছে কিনা verify
        $this->notification
            ->expects($this->once())
            ->method('sendSms')
            ->with(
                '+8801712345678',
                $this->stringContains('TRACK-333'),
            );

        $request = new OrderRequest(
            orderId: 'ORD-004',
            customerId: 'CUST-1',
            customerPhone: '+8801712345678',
            items: ['PROD-001' => 1],
            totalAmount: 1500.0,
            paymentMethod: 'bkash',
            shippingAddress: 'Chittagong',
        );

        $this->facade->placeOrder($request);
    }
}
```

### Jest — JavaScript Facade টেস্টিং

```javascript
import { jest, describe, it, expect, beforeEach } from "@jest/globals";

describe("OrderProcessingFacade", () => {
  let facade;
  let mockInventory, mockPayment, mockShipping, mockNotification;

  beforeEach(() => {
    mockInventory = {
      checkAvailability: jest.fn().mockReturnValue(true),
      reserveItems: jest.fn().mockReturnValue("RES-123"),
      releaseReservation: jest.fn(),
    };

    mockPayment = {
      validatePaymentMethod: jest.fn().mockReturnValue(true),
      processPayment: jest.fn().mockResolvedValue({
        success: true,
        transactionId: "TXN-456",
      }),
    };

    mockShipping = {
      createShipment: jest.fn().mockResolvedValue("TRACK-789"),
      getEstimatedDelivery: jest.fn().mockReturnValue("১-২ দিন"),
      calculateCost: jest.fn().mockReturnValue(60),
    };

    mockNotification = {
      sendSms: jest.fn().mockResolvedValue(undefined),
      sendPush: jest.fn().mockResolvedValue(undefined),
    };

    facade = new OrderProcessingFacade({
      inventory: mockInventory,
      payment: mockPayment,
      shipping: mockShipping,
      notification: mockNotification,
    });
  });

  it("সফল অর্ডার প্রসেসিং করতে পারে", async () => {
    const result = await facade.placeOrder({
      orderId: "ORD-001",
      customerId: "CUST-1",
      customerPhone: "+8801700000000",
      items: { "PROD-001": 2 },
      totalAmount: 2000,
      paymentMethod: "bkash",
      shippingAddress: "Dhaka",
    });

    expect(result.success).toBe(true);
    expect(result.trackingNumber).toBe("TRACK-789");
    expect(mockInventory.checkAvailability).toHaveBeenCalledWith({ "PROD-001": 2 });
    expect(mockPayment.processPayment).toHaveBeenCalledWith("CUST-1", 2000, "bkash");
    expect(mockNotification.sendSms).toHaveBeenCalled();
  });

  it("স্টক না থাকলে অর্ডার ব্যর্থ হয়", async () => {
    mockInventory.checkAvailability.mockReturnValue(false);

    const result = await facade.placeOrder({
      orderId: "ORD-002",
      customerId: "CUST-1",
      customerPhone: "+8801700000000",
      items: { "PROD-003": 1 },
      totalAmount: 500,
      paymentMethod: "bkash",
      shippingAddress: "Dhaka",
    });

    expect(result.success).toBe(false);
    expect(result.error).toBe("পণ্য স্টকে নেই");
    expect(mockPayment.processPayment).not.toHaveBeenCalled();
  });

  it("পেমেন্ট ব্যর্থ হলে রিজার্ভেশন রিলিজ হয়", async () => {
    mockPayment.processPayment.mockResolvedValue({ success: false });

    const result = await facade.placeOrder({
      orderId: "ORD-003",
      customerId: "CUST-1",
      customerPhone: "+8801700000000",
      items: { "PROD-001": 1 },
      totalAmount: 1000,
      paymentMethod: "bkash",
      shippingAddress: "Dhaka",
    });

    expect(result.success).toBe(false);
    expect(mockInventory.releaseReservation).toHaveBeenCalledWith("RES-123");
  });

  it("সাবসিস্টেমগুলো সঠিক ক্রমে কল হয়", async () => {
    const callOrder = [];
    mockInventory.checkAvailability.mockImplementation(() => { callOrder.push("inventory-check"); return true; });
    mockInventory.reserveItems.mockImplementation(() => { callOrder.push("inventory-reserve"); return "RES-1"; });
    mockPayment.validatePaymentMethod.mockImplementation(() => { callOrder.push("payment-validate"); return true; });
    mockPayment.processPayment.mockImplementation(async () => { callOrder.push("payment-process"); return { success: true, transactionId: "T-1" }; });
    mockShipping.createShipment.mockImplementation(async () => { callOrder.push("shipping-create"); return "TRK-1"; });
    mockNotification.sendSms.mockImplementation(async () => { callOrder.push("notify-sms"); });

    await facade.placeOrder({
      orderId: "ORD-004",
      customerId: "C-1",
      customerPhone: "+880170000",
      items: { "PROD-001": 1 },
      totalAmount: 500,
      paymentMethod: "bkash",
      shippingAddress: "Dhaka",
    });

    expect(callOrder).toEqual([
      "inventory-check",
      "payment-validate",
      "inventory-reserve",
      "payment-process",
      "shipping-create",
      "notify-sms",
    ]);
  });
});
```

### Laravel Facade টেস্টিং — `Facade::shouldReceive()`

```php
<?php

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;

class LaravelFacadeTestExample extends TestCase
{
    public function test_cache_facade_mocking(): void
    {
        // Laravel Facade-এর বিল্ট-ইন mocking (Mockery ব্যবহার করে)
        Cache::shouldReceive('get')
            ->once()
            ->with('user_42')
            ->andReturn(['name' => 'রহিম', 'email' => 'rahim@example.com']);

        Cache::shouldReceive('put')
            ->once()
            ->with('user_42', \Mockery::type('array'), 3600);

        $service = new UserService();
        $user = $service->getUser(42);

        $this->assertEquals('রহিম', $user['name']);
    }

    public function test_mail_facade_fake(): void
    {
        // Facade::fake() — পাঠানো মেইল ধরে রাখে, আসলে পাঠায় না
        Mail::fake();

        $orderService = new OrderService();
        $orderService->complete('ORD-100');

        Mail::assertSent(OrderConfirmationMail::class, function ($mail) {
            return $mail->hasTo('customer@example.com');
        });
    }

    public function test_notification_facade_fake(): void
    {
        Notification::fake();

        $orderService = new OrderService();
        $orderService->complete('ORD-200');

        Notification::assertSentTo(
            $user,
            OrderShippedNotification::class,
        );
    }
}
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Facade ↔ Adapter
- **Adapter**: একটি বিদ্যমান ইন্টারফেসকে ক্লায়েন্টের প্রত্যাশিত ইন্টারফেসে রূপান্তর করে
- **Facade**: একাধিক ইন্টারফেসকে একটি সরল ইন্টারফেসে রূপান্তর করে
- *সম্পর্ক*: Facade প্রায়ই অভ্যন্তরে Adapter ব্যবহার করে (Anti-Corruption Layer)

### Facade ↔ Mediator
- **Mediator**: অবজেক্টদের মধ্যে communication coordinate করে; সবাই Mediator-কে জানে
- **Facade**: একদিকী — ক্লায়েন্ট Facade-কে জানে, কিন্তু subsystem Facade-কে জানে না
- *সম্পর্ক*: উভয়ই জটিলতা কমায়, কিন্তু দিক ভিন্ন

### Facade ↔ Abstract Factory
- **Abstract Factory** Facade-এর বিকল্প হিসেবে ব্যবহার হতে পারে — subsystem object তৈরির জটিলতা লুকানোর জন্য
- *সম্পর্ক*: Facade Factory ব্যবহার করে subsystem component তৈরি করতে পারে

### Facade ↔ Singleton
- Facade প্রায়ই **Singleton** হিসেবে implement করা হয় কারণ সাধারণত একটি Facade instance-ই যথেষ্ট
- *সম্পর্ক*: Laravel-এর Facade-গুলো resolved instance cache করে (Singleton-এর মতো)

```
┌────────────────────────────────────────────────────────┐
│              Pattern Relationship Map                    │
│                                                         │
│   Abstract Factory ──creates──▶ Subsystem Objects       │
│         │                            │                  │
│         ▼                            ▼                  │
│      Facade ◀──uses── Adapter (interface translation)   │
│         │                                               │
│    Singleton ──ensures── single Facade instance         │
│         │                                               │
│    Mediator ──similar── bidirectional communication      │
└────────────────────────────────────────────────────────┘
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

1. **জটিল সাবসিস্টেম সরলীকরণ** — যখন অনেকগুলো ক্লাস/সার্ভিস একসাথে কাজ করে এবং ক্লায়েন্টকে সব জানার দরকার নেই
2. **Legacy system integration** — পুরানো কোডের উপর আধুনিক API দিতে
3. **Third-party library wrapping** — বাইরের লাইব্রেরির API পরিবর্তন হলে শুধু Facade আপডেট করলেই হবে
4. **Layered architecture** — প্রতিটি layer-এর entry point হিসেবে
5. **Team isolation** — একটি টিমের কোড অন্য টিম Facade দিয়ে ব্যবহার করবে
6. **API Gateway** — microservices-এর সামনে unified API দিতে

### ❌ কখন ব্যবহার করবেন না

1. **সাবসিস্টেম সহজ** — ২-৩টি ক্লাস হলে Facade অতিরিক্ত abstraction
2. **ক্লায়েন্টের সব feature দরকার** — যদি ক্লায়েন্ট সাবসিস্টেমের সব মেথড ব্যবহার করে, Facade মূল্য যোগ করে না
3. **Performance critical** — extra layer-এ overhead যোগ হয় (যদিও সামান্য)
4. **God Facade হওয়ার সম্ভাবনা** — যদি Facade-এ ৩০+ মেথড থাকে, ভেঙে ফেলুন

### সিদ্ধান্ত নেওয়ার চেকলিস্ট

```
□ সাবসিস্টেমে ৩+ ক্লাস/সার্ভিস আছে?
□ ক্লায়েন্ট কোড সাবসিস্টেমের সব কিছু জানার দরকার নেই?
□ সাবসিস্টেমের internal পরিবর্তন ক্লায়েন্টকে প্রভাবিত করবে?
□ একই workflow বারবার বিভিন্ন জায়গায় লেখা হচ্ছে?
□ নতুন ডেভেলপাররা সিস্টেম বুঝতে কষ্ট পাচ্ছে?

→ ৩+ টিক থাকলে Facade ব্যবহার করুন
```

---

## 📋 সারসংক্ষেপ (Summary)

```
┌──────────────────────────────────────────────────────────────┐
│                    FACADE PATTERN CHEAT SHEET                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🎯 উদ্দেশ্য: জটিল সাবসিস্টেমের সামনে সরল ইন্টারফেস          │
│                                                              │
│  📦 Type: Structural                                         │
│                                                              │
│  🔑 মূল নীতি:                                                 │
│     • ক্লায়েন্ট শুধু Facade-এর সাথে কথা বলে                    │
│     • Facade সাবসিস্টেমের কাজ orchestrate করে                  │
│     • সাবসিস্টেম Facade সম্পর্কে কিছু জানে না                  │
│     • Facade কোনো নতুন functionality যোগ করে না                │
│                                                              │
│  ✅ ব্যবহার করুন:                                              │
│     • জটিল সাবসিস্টেম সরলীকরণে                                │
│     • Legacy wrapping-এ                                       │
│     • Third-party integration-এ                               │
│     • Layered architecture-এ                                  │
│                                                              │
│  ❌ এড়িয়ে চলুন:                                                │
│     • সহজ সিস্টেমে                                             │
│     • God Facade তৈরি করা                                     │
│     • Facade-এ business logic রাখা                             │
│                                                              │
│  🇧🇩 বাংলাদেশ কনটেক্সট:                                       │
│     • bKash/Nagad payment integration facade                 │
│     • E-commerce checkout facade                             │
│     • SMS gateway (BulkSMS/SSL) facade                       │
│     • Pathao/RedX courier facade                             │
│                                                              │
│  🔗 সম্পর্কিত: Adapter, Mediator, Abstract Factory, Singleton │
│                                                              │
│  💡 মনে রাখুন: Facade = হোটেল রিসেপশন ডেস্ক                    │
│     আপনি শুধু রিসেপশনে বলবেন, তারা সব সার্ভিস coordinate করবে │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

> **"Any fool can write code that a computer can understand. Good programmers write code that humans can understand."** — Martin Fowler
>
> Facade প্যাটার্ন ঠিক এই কাজটিই করে — জটিল সিস্টেমকে মানুষের বোঝার উপযোগী করে তোলে। 🏛️
