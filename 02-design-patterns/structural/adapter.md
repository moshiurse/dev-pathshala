# 🔌 Adapter প্যাটার্ন (Adapter Pattern)

> **"Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces."**
> — Gang of Four (GoF), *Design Patterns: Elements of Reusable Object-Oriented Software*

---

## 📌 সংজ্ঞা

Adapter প্যাটার্ন একটি **Structural Design Pattern** যা দুটি incompatible interface-এর মধ্যে সেতুবন্ধন তৈরি করে। এই প্যাটার্নের মূল কাজ হলো — একটি ক্লাসের interface-কে অন্য একটি interface-এ রূপান্তর করা যাতে client তার প্রত্যাশিত interface-এর মাধ্যমে কাজ করতে পারে।

ধরুন আপনার সিস্টেমে একটি নির্দিষ্ট interface আছে (`PaymentInterface`), কিন্তু আপনি একটি third-party library ব্যবহার করতে চান যার interface সম্পূর্ণ আলাদা। Adapter প্যাটার্ন এই দুটি incompatible interface-কে একসাথে কাজ করার সুযোগ দেয় — কোনো পক্ষের কোড পরিবর্তন না করেই।

### মূল ধারণা:
- **Target**: Client যে interface আশা করে (যেমন `PaymentInterface`)
- **Adaptee**: যে ক্লাসের interface adapt করতে হবে (যেমন `BkashPaymentSDK`)
- **Adapter**: Target interface implement করে এবং Adaptee-কে wrap করে
- **Client**: শুধুমাত্র Target interface-এর সাথে কাজ করে

### GoF Classification:
| বিষয় | মান |
|--------|------|
| **ক্যাটাগরি** | Structural Pattern |
| **স্কোপ** | Class (inheritance) অথবা Object (composition) |
| **উদ্দেশ্য** | Incompatible interface-কে compatible করা |
| **অন্য নাম** | Wrapper |

---

## 🏠 বাস্তব উদাহরণ (Real-World Analogy)

### ⚡ পাওয়ার প্লাগ অ্যাডাপ্টার

সবচেয়ে সহজ উদাহরণ হলো বিদ্যুৎ প্লাগ অ্যাডাপ্টার:

```
🔌 US ডিভাইস (2-pin flat plug) ──→ 🔄 অ্যাডাপ্টার ──→ 🏠 BD সকেট (3-pin round)

US ল্যাপটপ চার্জার:       অ্যাডাপ্টার:              বাংলাদেশের সকেট:
┌─────────┐            ┌──────────┐           ┌──────────┐
│  ||     │ ──────→    │ || → ●●● │ ──────→   │  ●●●     │
│  (flat) │            │ converter│           │ (round)  │
└─────────┘            └──────────┘           └──────────┘
```

**বাংলাদেশ প্রেক্ষাপটে চিন্তা করুন:**
- আপনি Amazon থেকে একটি US ল্যাপটপ কিনলেন
- চার্জারের প্লাগ flat 2-pin (Type A)
- বাংলাদেশের সকেট round 3-pin (Type D/G)
- সমাধান: একটি **প্লাগ অ্যাডাপ্টার** যা flat pin গ্রহণ করে round pin-এ রূপান্তর করে

এখানে:
- **Target** = বাংলাদেশের সকেট (round pin আশা করে)
- **Adaptee** = US প্লাগ (flat pin দেয়)
- **Adapter** = প্লাগ কনভার্টার (মাঝখানে বসে translate করে)
- **Client** = বিদ্যুৎ সরবরাহ ব্যবস্থা

### 💰 bKash/Nagad পেমেন্ট উদাহরণ

আপনার ই-কমার্স সাইটে একটি `PaymentInterface` আছে যেখানে `charge()` মেথড আছে। কিন্তু bKash SDK-তে আছে `createPayment()`, Nagad SDK-তে আছে `initiateTransaction()`। Adapter প্যাটার্ন দিয়ে প্রতিটি SDK-কে আপনার `PaymentInterface`-এর সাথে মানানসই করা যায়।

---

## 📊 UML ডায়াগ্রাম (ASCII)

### Object Adapter (Composition-based) — বেশি ব্যবহৃত

```
    ┌──────────────────┐         ┌───────────────────┐
    │     Client       │         │   <<interface>>    │
    │                  │────────>│      Target        │
    │                  │         │───────────────────│
    └──────────────────┘         │ + request()        │
                                 └───────────────────┘
                                          ▲
                                          │ implements
                                 ┌───────────────────┐       ┌───────────────────┐
                                 │     Adapter        │       │     Adaptee       │
                                 │───────────────────│       │───────────────────│
                                 │ - adaptee: Adaptee│──────>│ + specificRequest()│
                                 │───────────────────│       │                   │
                                 │ + request()       │       └───────────────────┘
                                 │   // adaptee->    │
                                 │   // specificReq() │
                                 └───────────────────┘

    ★ Composition ব্যবহার করে — adaptee-কে property হিসেবে রাখে
    ★ Runtime-এ adaptee পরিবর্তন সম্ভব
    ★ SOLID-এর Dependency Inversion মানে
```

### Class Adapter (Inheritance-based)

```
    ┌──────────────────┐         ┌───────────────────┐
    │     Client       │         │   <<interface>>    │
    │                  │────────>│      Target        │
    │                  │         │───────────────────│
    └──────────────────┘         │ + request()        │
                                 └───────────────────┘
                                          ▲
                                          │ implements
                                 ┌───────────────────┐
                                 │     Adapter        │
                                 │───────────────────│
                                 │ + request()       │
                                 │   // this->       │
                                 │   // specificReq() │
                                 └───────────────────┘
                                          ▲
                                          │ extends
                                 ┌───────────────────┐
                                 │     Adaptee       │
                                 │───────────────────│
                                 │ + specificRequest()│
                                 └───────────────────┘

    ★ Multiple inheritance প্রয়োজন (PHP-তে interface + extends দিয়ে সম্ভব)
    ★ Compile-time binding — runtime-এ adaptee পরিবর্তন অসম্ভব
    ★ Adaptee-এর behavior override করার সুবিধা
```

### দুই ধরনের তুলনা:

| বৈশিষ্ট্য | Object Adapter | Class Adapter |
|-----------|---------------|---------------|
| **সম্পর্ক** | Composition (has-a) | Inheritance (is-a) |
| **Flexibility** | ✅ বেশি (runtime swap) | ❌ কম (compile-time) |
| **Adaptee Access** | শুধু public API | Protected members-ও পায় |
| **Multiple Adaptees** | ✅ একটি adapter-এ একাধিক | ❌ একটিতে একটি |
| **Override** | ❌ Adaptee behavior override কঠিন | ✅ সহজে override সম্ভব |
| **SOLID** | ✅ ভালো মানে | ⚠️ Tight coupling |
| **সুপারিশ** | ✅ **বেশিরভাগ ক্ষেত্রে এটি ব্যবহার করুন** | নির্দিষ্ট পরিস্থিতিতে |

---

## 💻 ইমপ্লিমেন্টেশন

### ১. Object Adapter (Composition-based) — সবচেয়ে বেশি ব্যবহৃত

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// ========================
// Target Interface — Client এটি আশা করে
// ========================
interface PaymentGateway
{
    public function charge(float $amount, string $currency, string $customerRef): PaymentResult;
    public function refund(string $transactionId, float $amount): RefundResult;
    public function getStatus(string $transactionId): TransactionStatus;
}

// ========================
// Value Objects — Type safety নিশ্চিত করে
// ========================
readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public float $amount,
        public string $currency,
        public string $message,
        public \DateTimeImmutable $processedAt = new \DateTimeImmutable(),
    ) {}
}

readonly class RefundResult
{
    public function __construct(
        public bool $success,
        public string $refundId,
        public float $amount,
        public string $message,
    ) {}
}

enum TransactionStatus: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    case Failed = 'failed';
    case Refunded = 'refunded';
}

// ========================
// Adaptee ১: bKash SDK (তাদের নিজস্ব interface)
// ========================
class BkashPaymentSDK
{
    public function createPayment(array $paymentData): array
    {
        // bKash API কল সিমুলেশন
        return [
            'paymentID' => 'BK' . uniqid(),
            'trxID' => 'TRX' . random_int(100000, 999999),
            'status' => 'Completed',
            'amount' => $paymentData['amount'],
            'merchantInvoiceNumber' => $paymentData['invoice'],
        ];
    }

    public function refundTransaction(string $paymentId, string $trxId, float $amount): array
    {
        return [
            'refundTrxID' => 'RF' . uniqid(),
            'transactionStatus' => 'Completed',
            'amount' => $amount,
            'completedTime' => date('Y-m-d H:i:s'),
        ];
    }

    public function queryPayment(string $paymentId): array
    {
        return [
            'paymentID' => $paymentId,
            'transactionStatus' => 'Completed',
        ];
    }
}

// ========================
// Adaptee ২: Nagad SDK (সম্পূর্ণ ভিন্ন interface)
// ========================
class NagadPaymentSDK
{
    public function initiateTransaction(
        string $merchantId,
        string $orderId,
        float $amount,
        string $callbackUrl
    ): object {
        return (object) [
            'issuerPaymentRefNo' => 'NG' . uniqid(),
            'status' => 'Success',
            'orderID' => $orderId,
            'amount' => $amount,
        ];
    }

    public function verifyTransaction(string $paymentRefNo): object
    {
        return (object) [
            'issuerPaymentRefNo' => $paymentRefNo,
            'statusCode' => '000',
            'status' => 'Success',
        ];
    }

    public function refundPayment(string $paymentRefNo, float $refundAmount, string $reason): object
    {
        return (object) [
            'refundId' => 'NRF' . uniqid(),
            'status' => 'Refunded',
            'amount' => $refundAmount,
        ];
    }
}

// ========================
// Object Adapter: bKash → PaymentGateway
// ========================
class BkashPaymentAdapter implements PaymentGateway
{
    public function __construct(
        private readonly BkashPaymentSDK $bkash,
        private readonly string $merchantInvoicePrefix = 'INV',
    ) {}

    public function charge(float $amount, string $currency, string $customerRef): PaymentResult
    {
        // bKash SDK-এর format-এ ডেটা রূপান্তর
        $response = $this->bkash->createPayment([
            'amount' => $amount,
            'invoice' => "{$this->merchantInvoicePrefix}-{$customerRef}",
        ]);

        // bKash response → আমাদের unified PaymentResult-এ রূপান্তর
        return new PaymentResult(
            success: $response['status'] === 'Completed',
            transactionId: $response['trxID'],
            amount: (float) $response['amount'],
            currency: $currency,
            message: "bKash payment {$response['status']}. PaymentID: {$response['paymentID']}",
        );
    }

    public function refund(string $transactionId, float $amount): RefundResult
    {
        $response = $this->bkash->refundTransaction(
            paymentId: $transactionId,
            trxId: $transactionId,
            amount: $amount,
        );

        return new RefundResult(
            success: $response['transactionStatus'] === 'Completed',
            refundId: $response['refundTrxID'],
            amount: (float) $response['amount'],
            message: "bKash refund processed at {$response['completedTime']}",
        );
    }

    public function getStatus(string $transactionId): TransactionStatus
    {
        $response = $this->bkash->queryPayment($transactionId);

        return match ($response['transactionStatus']) {
            'Completed' => TransactionStatus::Completed,
            'Pending', 'Initiated' => TransactionStatus::Pending,
            default => TransactionStatus::Failed,
        };
    }
}

// ========================
// Object Adapter: Nagad → PaymentGateway
// ========================
class NagadPaymentAdapter implements PaymentGateway
{
    public function __construct(
        private readonly NagadPaymentSDK $nagad,
        private readonly string $merchantId,
        private readonly string $callbackUrl,
    ) {}

    public function charge(float $amount, string $currency, string $customerRef): PaymentResult
    {
        $response = $this->nagad->initiateTransaction(
            merchantId: $this->merchantId,
            orderId: $customerRef,
            amount: $amount,
            callbackUrl: $this->callbackUrl,
        );

        return new PaymentResult(
            success: $response->status === 'Success',
            transactionId: $response->issuerPaymentRefNo,
            amount: (float) $response->amount,
            currency: $currency,
            message: "Nagad payment {$response->status}. OrderID: {$response->orderID}",
        );
    }

    public function refund(string $transactionId, float $amount): RefundResult
    {
        $response = $this->nagad->refundPayment(
            paymentRefNo: $transactionId,
            refundAmount: $amount,
            reason: 'Customer requested refund',
        );

        return new RefundResult(
            success: $response->status === 'Refunded',
            refundId: $response->refundId,
            amount: (float) $response->amount,
            message: "Nagad refund {$response->status}",
        );
    }

    public function getStatus(string $transactionId): TransactionStatus
    {
        $response = $this->nagad->verifyTransaction($transactionId);

        return match ($response->statusCode) {
            '000' => TransactionStatus::Completed,
            '001' => TransactionStatus::Pending,
            default => TransactionStatus::Failed,
        };
    }
}

// ========================
// Client Code — কোনো concrete adapter সম্পর্কে জানে না
// ========================
class CheckoutService
{
    public function __construct(
        private readonly PaymentGateway $gateway,
    ) {}

    public function processPayment(float $amount, string $customerRef): PaymentResult
    {
        // Client শুধু PaymentGateway interface জানে
        // bKash না Nagad — সেটা জানার দরকার নেই
        $result = $this->gateway->charge($amount, 'BDT', $customerRef);

        if ($result->success) {
            // order confirm logic...
            echo "✅ পেমেন্ট সফল: {$result->transactionId} — {$result->amount} BDT\n";
        }

        return $result;
    }
}

// ব্যবহার:
$bkashAdapter = new BkashPaymentAdapter(new BkashPaymentSDK());
$nagadAdapter = new NagadPaymentAdapter(
    new NagadPaymentSDK(),
    merchantId: 'MERCHANT_001',
    callbackUrl: 'https://myshop.com.bd/callback',
);

$checkout = new CheckoutService($bkashAdapter); // অথবা $nagadAdapter
$checkout->processPayment(1500.00, 'ORD-2024-001');
```

#### JavaScript (ES2022+)

```javascript
// ========================
// Target Interface (JSDoc + Symbol দিয়ে enforce)
// ========================
const PaymentGateway = Symbol('PaymentGateway');

class PaymentResult {
    /** @readonly */ success;
    /** @readonly */ transactionId;
    /** @readonly */ amount;
    /** @readonly */ currency;
    /** @readonly */ message;
    /** @readonly */ processedAt;

    constructor({ success, transactionId, amount, currency, message }) {
        this.success = success;
        this.transactionId = transactionId;
        this.amount = amount;
        this.currency = currency;
        this.message = message;
        this.processedAt = new Date();
        Object.freeze(this);
    }
}

class RefundResult {
    constructor({ success, refundId, amount, message }) {
        Object.assign(this, { success, refundId, amount, message });
        Object.freeze(this);
    }
}

// ========================
// Adaptee ১: bKash SDK
// ========================
class BkashPaymentSDK {
    async createPayment(paymentData) {
        // bKash API কল সিমুলেশন
        return {
            paymentID: `BK${crypto.randomUUID().slice(0, 8)}`,
            trxID: `TRX${Math.floor(Math.random() * 900000 + 100000)}`,
            status: 'Completed',
            amount: paymentData.amount,
            merchantInvoiceNumber: paymentData.invoice,
        };
    }

    async refundTransaction(paymentId, trxId, amount) {
        return {
            refundTrxID: `RF${crypto.randomUUID().slice(0, 8)}`,
            transactionStatus: 'Completed',
            amount,
            completedTime: new Date().toISOString(),
        };
    }

    async queryPayment(paymentId) {
        return { paymentID: paymentId, transactionStatus: 'Completed' };
    }
}

// ========================
// Adaptee ২: Nagad SDK
// ========================
class NagadPaymentSDK {
    async initiateTransaction(merchantId, orderId, amount, callbackUrl) {
        return {
            issuerPaymentRefNo: `NG${crypto.randomUUID().slice(0, 8)}`,
            status: 'Success',
            orderID: orderId,
            amount,
        };
    }

    async verifyTransaction(paymentRefNo) {
        return {
            issuerPaymentRefNo: paymentRefNo,
            statusCode: '000',
            status: 'Success',
        };
    }

    async refundPayment(paymentRefNo, refundAmount, reason) {
        return {
            refundId: `NRF${crypto.randomUUID().slice(0, 8)}`,
            status: 'Refunded',
            amount: refundAmount,
        };
    }
}

// ========================
// Object Adapter: bKash → PaymentGateway
// ========================
class BkashPaymentAdapter {
    #bkash;
    #invoicePrefix;

    constructor(bkashSdk, invoicePrefix = 'INV') {
        this.#bkash = bkashSdk;
        this.#invoicePrefix = invoicePrefix;
        this[PaymentGateway] = true; // interface marker
    }

    async charge(amount, currency, customerRef) {
        const response = await this.#bkash.createPayment({
            amount,
            invoice: `${this.#invoicePrefix}-${customerRef}`,
        });

        return new PaymentResult({
            success: response.status === 'Completed',
            transactionId: response.trxID,
            amount: response.amount,
            currency,
            message: `bKash payment ${response.status}. PaymentID: ${response.paymentID}`,
        });
    }

    async refund(transactionId, amount) {
        const response = await this.#bkash.refundTransaction(
            transactionId, transactionId, amount,
        );

        return new RefundResult({
            success: response.transactionStatus === 'Completed',
            refundId: response.refundTrxID,
            amount: response.amount,
            message: `bKash refund processed at ${response.completedTime}`,
        });
    }

    async getStatus(transactionId) {
        const response = await this.#bkash.queryPayment(transactionId);
        const statusMap = { Completed: 'completed', Pending: 'pending', Initiated: 'pending' };
        return statusMap[response.transactionStatus] ?? 'failed';
    }
}

// ========================
// Object Adapter: Nagad → PaymentGateway
// ========================
class NagadPaymentAdapter {
    #nagad;
    #merchantId;
    #callbackUrl;

    constructor(nagadSdk, merchantId, callbackUrl) {
        this.#nagad = nagadSdk;
        this.#merchantId = merchantId;
        this.#callbackUrl = callbackUrl;
        this[PaymentGateway] = true;
    }

    async charge(amount, currency, customerRef) {
        const response = await this.#nagad.initiateTransaction(
            this.#merchantId, customerRef, amount, this.#callbackUrl,
        );

        return new PaymentResult({
            success: response.status === 'Success',
            transactionId: response.issuerPaymentRefNo,
            amount: response.amount,
            currency,
            message: `Nagad payment ${response.status}. OrderID: ${response.orderID}`,
        });
    }

    async refund(transactionId, amount) {
        const response = await this.#nagad.refundPayment(
            transactionId, amount, 'Customer requested refund',
        );

        return new RefundResult({
            success: response.status === 'Refunded',
            refundId: response.refundId,
            amount: response.amount,
            message: `Nagad refund ${response.status}`,
        });
    }

    async getStatus(transactionId) {
        const response = await this.#nagad.verifyTransaction(transactionId);
        return response.statusCode === '000' ? 'completed' : 'failed';
    }
}

// ========================
// Client Code
// ========================
class CheckoutService {
    #gateway;

    constructor(gateway) {
        if (!gateway[PaymentGateway]) {
            throw new TypeError('Gateway must implement PaymentGateway interface');
        }
        this.#gateway = gateway;
    }

    async processPayment(amount, customerRef) {
        const result = await this.#gateway.charge(amount, 'BDT', customerRef);

        if (result.success) {
            console.log(`✅ পেমেন্ট সফল: ${result.transactionId} — ${result.amount} BDT`);
        }

        return result;
    }
}

// ব্যবহার:
const bkashAdapter = new BkashPaymentAdapter(new BkashPaymentSDK());
const nagadAdapter = new NagadPaymentAdapter(
    new NagadPaymentSDK(), 'MERCHANT_001', 'https://myshop.com.bd/callback',
);

const checkout = new CheckoutService(bkashAdapter);
await checkout.processPayment(1500.00, 'ORD-2024-001');
```

---

### ২. Class Adapter (Inheritance-based)

Class Adapter inheritance ব্যবহার করে adaptee-এর functionality পায়। PHP-তে single inheritance limitation আছে, তাই interface implement করে class extend করতে হয়।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Target Interface
interface NotificationSender
{
    public function send(string $recipient, string $message): bool;
    public function getDeliveryStatus(string $messageId): string;
}

// Adaptee: Legacy BD SMS Gateway (পুরনো API)
class BangladeshSmsGateway
{
    protected string $apiKey;

    public function __construct(string $apiKey)
    {
        $this->apiKey = $apiKey;
    }

    public function sendSms(string $mobileNumber, string $textContent, string $maskName): array
    {
        // বাংলাদেশি SMS gateway API কল
        return [
            'response_code' => 202,
            'message_id' => 'BDSMS_' . uniqid(),
            'success_message' => 'SMS sent successfully',
        ];
    }

    public function checkStatus(string $messageId): string
    {
        return 'DELIVERED';
    }

    protected function formatBdPhoneNumber(string $number): string
    {
        // +880 prefix নিশ্চিত করা
        if (str_starts_with($number, '01')) {
            return '+880' . substr($number, 1);
        }
        return $number;
    }
}

// Class Adapter — Inheritance ব্যবহার করে
class BdSmsNotificationAdapter extends BangladeshSmsGateway implements NotificationSender
{
    public function __construct(
        string $apiKey,
        private readonly string $senderMask = 'MyShopBD',
    ) {
        parent::__construct($apiKey);
    }

    public function send(string $recipient, string $message): bool
    {
        // parent class-এর protected method সরাসরি access করতে পারছি
        // Object Adapter-এ এটা সম্ভব হতো না
        $formattedNumber = $this->formatBdPhoneNumber($recipient);

        $response = $this->sendSms($formattedNumber, $message, $this->senderMask);

        return $response['response_code'] === 202;
    }

    public function getDeliveryStatus(string $messageId): string
    {
        $status = $this->checkStatus($messageId);

        return match ($status) {
            'DELIVERED' => 'delivered',
            'PENDING' => 'pending',
            'FAILED', 'EXPIRED' => 'failed',
            default => 'unknown',
        };
    }
}

// ব্যবহার:
$smsAdapter = new BdSmsNotificationAdapter(
    apiKey: 'BD_SMS_API_KEY_123',
    senderMask: 'DhakaShop',
);

$sent = $smsAdapter->send('01712345678', 'আপনার অর্ডার কনফার্ম হয়েছে!');
echo $sent ? '✅ SMS পাঠানো হয়েছে' : '❌ SMS পাঠানো ব্যর্থ';
```

#### JavaScript (ES2022+)

```javascript
// Adaptee: Legacy BD SMS Gateway
class BangladeshSmsGateway {
    #apiKey;

    constructor(apiKey) {
        this.#apiKey = apiKey;
    }

    async sendSms(mobileNumber, textContent, maskName) {
        return {
            response_code: 202,
            message_id: `BDSMS_${Date.now()}`,
            success_message: 'SMS sent successfully',
        };
    }

    async checkStatus(messageId) {
        return 'DELIVERED';
    }

    // JS-তে protected নেই, তাই naming convention ব্যবহার
    _formatBdPhoneNumber(number) {
        if (number.startsWith('01')) {
            return `+880${number.slice(1)}`;
        }
        return number;
    }
}

// Class Adapter — Inheritance ব্যবহার
class BdSmsNotificationAdapter extends BangladeshSmsGateway {
    #senderMask;

    constructor(apiKey, senderMask = 'MyShopBD') {
        super(apiKey);
        this.#senderMask = senderMask;
    }

    async send(recipient, message) {
        const formattedNumber = this._formatBdPhoneNumber(recipient);
        const response = await this.sendSms(formattedNumber, message, this.#senderMask);
        return response.response_code === 202;
    }

    async getDeliveryStatus(messageId) {
        const status = await this.checkStatus(messageId);
        const statusMap = {
            DELIVERED: 'delivered',
            PENDING: 'pending',
            FAILED: 'failed',
            EXPIRED: 'failed',
        };
        return statusMap[status] ?? 'unknown';
    }
}

// ব্যবহার:
const smsAdapter = new BdSmsNotificationAdapter('BD_SMS_API_KEY_123', 'DhakaShop');
const sent = await smsAdapter.send('01712345678', 'আপনার অর্ডার কনফার্ম হয়েছে!');
console.log(sent ? '✅ SMS পাঠানো হয়েছে' : '❌ SMS পাঠানো ব্যর্থ');
```

---

### ৩. Two-Way Adapter (দ্বিমুখী অ্যাডাপ্টার)

Two-way adapter দুটি ভিন্ন interface-কে পরস্পরের সাথে কাজ করার সুযোগ দেয়। এটি বিশেষভাবে কাজে লাগে যখন দুটি সিস্টেমের মধ্যে দ্বিমুখী যোগাযোগ প্রয়োজন — যেমন legacy system ও new system একসাথে চালানোর সময়।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// New System Interface
interface ModernLogger
{
    public function log(string $level, string $message, array $context = []): void;
    public function emergency(string $message, array $context = []): void;
    public function error(string $message, array $context = []): void;
}

// Legacy System Interface
interface LegacyLogger
{
    public function writeLog(string $message, int $severity): void;
    public function writeError(string $message): void;
}

// Modern Logger Implementation
class PsrLogger implements ModernLogger
{
    public function log(string $level, string $message, array $context = []): void
    {
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        echo "[PSR][{$level}] {$message}{$contextStr}\n";
    }

    public function emergency(string $message, array $context = []): void
    {
        $this->log('EMERGENCY', $message, $context);
    }

    public function error(string $message, array $context = []): void
    {
        $this->log('ERROR', $message, $context);
    }
}

// Legacy Logger Implementation
class FileLogger implements LegacyLogger
{
    public function writeLog(string $message, int $severity): void
    {
        echo "[FILE][Severity:{$severity}] {$message}\n";
    }

    public function writeError(string $message): void
    {
        $this->writeLog("ERROR: {$message}", 1);
    }
}

// Two-Way Adapter — দুটি interface-ই implement করে
class TwoWayLoggerAdapter implements ModernLogger, LegacyLogger
{
    private const array SEVERITY_MAP = [
        'emergency' => 0,
        'error' => 1,
        'warning' => 2,
        'info' => 3,
        'debug' => 4,
    ];

    private const array LEVEL_MAP = [
        0 => 'emergency',
        1 => 'error',
        2 => 'warning',
        3 => 'info',
        4 => 'debug',
    ];

    public function __construct(
        private readonly ModernLogger $modernLogger,
        private readonly LegacyLogger $legacyLogger,
    ) {}

    // ModernLogger interface → LegacyLogger-এ delegate
    public function log(string $level, string $message, array $context = []): void
    {
        $severity = self::SEVERITY_MAP[strtolower($level)] ?? 3;
        $fullMessage = !empty($context)
            ? "{$message} | " . json_encode($context)
            : $message;

        $this->legacyLogger->writeLog($fullMessage, $severity);
    }

    public function emergency(string $message, array $context = []): void
    {
        $this->log('emergency', $message, $context);
        $this->modernLogger->emergency($message, $context); // উভয়তেই পাঠাই
    }

    public function error(string $message, array $context = []): void
    {
        $this->log('error', $message, $context);
        $this->modernLogger->error($message, $context);
    }

    // LegacyLogger interface → ModernLogger-এ delegate
    public function writeLog(string $message, int $severity): void
    {
        $level = self::LEVEL_MAP[$severity] ?? 'info';
        $this->modernLogger->log($level, $message);
    }

    public function writeError(string $message): void
    {
        $this->modernLogger->error($message);
    }
}

// ব্যবহার — উভয় সিস্টেমই এই adapter ব্যবহার করতে পারে
$adapter = new TwoWayLoggerAdapter(new PsrLogger(), new FileLogger());

// New code এটিকে ModernLogger হিসেবে ব্যবহার করে
$adapter->error('পেমেন্ট ব্যর্থ হয়েছে', ['orderId' => 'ORD-001']);

// Legacy code এটিকে LegacyLogger হিসেবে ব্যবহার করে
$adapter->writeLog('Legacy system log entry', 2);
```

#### JavaScript (ES2022+)

```javascript
// Two-Way Adapter — দুটি system-কে bridge করে
class TwoWayLoggerAdapter {
    #modernLogger;
    #legacyLogger;

    static #severityMap = { emergency: 0, error: 1, warning: 2, info: 3, debug: 4 };
    static #levelMap = { 0: 'emergency', 1: 'error', 2: 'warning', 3: 'info', 4: 'debug' };

    constructor(modernLogger, legacyLogger) {
        this.#modernLogger = modernLogger;
        this.#legacyLogger = legacyLogger;
    }

    // Modern interface methods
    log(level, message, context = {}) {
        const severity = TwoWayLoggerAdapter.#severityMap[level.toLowerCase()] ?? 3;
        const fullMessage = Object.keys(context).length
            ? `${message} | ${JSON.stringify(context)}`
            : message;
        this.#legacyLogger.writeLog(fullMessage, severity);
    }

    error(message, context = {}) {
        this.log('error', message, context);
        this.#modernLogger.error(message, context);
    }

    // Legacy interface methods
    writeLog(message, severity) {
        const level = TwoWayLoggerAdapter.#levelMap[severity] ?? 'info';
        this.#modernLogger.log(level, message);
    }

    writeError(message) {
        this.#modernLogger.error(message);
    }
}
```

---

## 🌍 Real-World Applicable Areas

### ১. Payment Gateway Adapter (পূর্ণ উদাহরণ উপরে)

বাংলাদেশের ই-কমার্স সাইটগুলোতে একাধিক পেমেন্ট গেটওয়ে ব্যবহার করা হয়: bKash, Nagad, Rocket, SSLCommerz, AamarPay। প্রতিটির API ভিন্ন, কিন্তু Adapter দিয়ে একটি unified `PaymentGateway` interface-এ আনা যায়।

### ২. SMS Provider Adapter

```php
<?php

// বাংলাদেশে জনপ্রিয় SMS Provider: BulkSMSBD, Infobip, SSL Wireless, Alpha Net
interface SmsProvider
{
    public function sendSms(string $to, string $body): SmsResponse;
    public function checkBalance(): float;
}

readonly class SmsResponse
{
    public function __construct(
        public bool $sent,
        public string $messageId,
        public string $provider,
    ) {}
}

// SSL Wireless SMS Adaptee (তাদের নিজস্ব API)
class SslWirelessApi
{
    public function pushSingleSms(
        string $msisdn,
        string $sms,
        string $csmsId,
        string $smsType = 'TEXT'
    ): array {
        return ['status' => 'SUCCESS', 'status_code' => 200, 'smsid' => 'SSL_' . uniqid()];
    }

    public function getBalance(): array
    {
        return ['balance' => 5000.50, 'currency' => 'BDT'];
    }
}

// Alpha Net SMS Adaptee
class AlphaNetSmsApi
{
    public function send_sms(string $mobile, string $message_body): string
    {
        // XML response সিমুলেশন
        return '<response><msgid>ALPHA_' . uniqid() . '</msgid><status>1</status></response>';
    }

    public function check_credit(): string
    {
        return '<response><credit>3200.75</credit></response>';
    }
}

// SSL Wireless Adapter
class SslWirelessSmsAdapter implements SmsProvider
{
    public function __construct(
        private readonly SslWirelessApi $api,
    ) {}

    public function sendSms(string $to, string $body): SmsResponse
    {
        $result = $this->api->pushSingleSms(
            msisdn: $to,
            sms: $body,
            csmsId: uniqid('csms_'),
        );

        return new SmsResponse(
            sent: $result['status_code'] === 200,
            messageId: $result['smsid'],
            provider: 'SSLWireless',
        );
    }

    public function checkBalance(): float
    {
        return $this->api->getBalance()['balance'];
    }
}

// Alpha Net Adapter — XML response parse করতে হয়
class AlphaNetSmsAdapter implements SmsProvider
{
    public function __construct(
        private readonly AlphaNetSmsApi $api,
    ) {}

    public function sendSms(string $to, string $body): SmsResponse
    {
        $xmlResponse = $this->api->send_sms($to, $body);
        $xml = simplexml_load_string($xmlResponse);

        return new SmsResponse(
            sent: (string) $xml->status === '1',
            messageId: (string) $xml->msgid,
            provider: 'AlphaNet',
        );
    }

    public function checkBalance(): float
    {
        $xmlResponse = $this->api->check_credit();
        $xml = simplexml_load_string($xmlResponse);

        return (float) $xml->credit;
    }
}
```

### ৩. File Storage Adapter

```php
<?php

// Unified Storage Interface
interface StorageAdapter
{
    public function put(string $path, string $contents): bool;
    public function get(string $path): string;
    public function delete(string $path): bool;
    public function exists(string $path): bool;
    public function url(string $path): string;
}

// S3 SDK Adaptee
class AwsS3Client
{
    public function putObject(array $args): object { /* ... */ }
    public function getObject(array $args): object { /* ... */ }
    public function deleteObject(array $args): object { /* ... */ }
    public function doesObjectExist(string $bucket, string $key): bool { return true; }
}

// S3 → StorageAdapter
class S3StorageAdapter implements StorageAdapter
{
    public function __construct(
        private readonly AwsS3Client $s3,
        private readonly string $bucket,
        private readonly string $cdnDomain = '',
    ) {}

    public function put(string $path, string $contents): bool
    {
        $this->s3->putObject([
            'Bucket' => $this->bucket,
            'Key'    => $path,
            'Body'   => $contents,
        ]);
        return true;
    }

    public function get(string $path): string
    {
        $result = $this->s3->getObject([
            'Bucket' => $this->bucket,
            'Key'    => $path,
        ]);
        return (string) $result;
    }

    public function delete(string $path): bool
    {
        $this->s3->deleteObject([
            'Bucket' => $this->bucket,
            'Key'    => $path,
        ]);
        return true;
    }

    public function exists(string $path): bool
    {
        return $this->s3->doesObjectExist($this->bucket, $path);
    }

    public function url(string $path): string
    {
        return $this->cdnDomain
            ? "https://{$this->cdnDomain}/{$path}"
            : "https://{$this->bucket}.s3.amazonaws.com/{$path}";
    }
}
```

### ৪. Laravel-এ Adapter Pattern-এর ব্যবহার

Laravel framework নিজেই Adapter pattern ব্যাপকভাবে ব্যবহার করে:

```
┌─────────────────────────────────────────────────────────────┐
│                    Laravel Adapters                          │
├────────────────┬────────────────────────────────────────────┤
│ Filesystem     │ LocalAdapter, S3Adapter, FtpAdapter        │
│                │ (Flysystem library-র মাধ্যমে)              │
├────────────────┼────────────────────────────────────────────┤
│ Cache          │ FileStore, RedisStore, MemcachedStore,     │
│                │ DatabaseStore, ArrayStore                   │
├────────────────┼────────────────────────────────────────────┤
│ Mail           │ SmtpTransport, SesTransport,               │
│                │ MailgunTransport, PostmarkTransport         │
├────────────────┼────────────────────────────────────────────┤
│ Queue          │ SyncQueue, DatabaseQueue, RedisQueue,      │
│                │ SqsQueue, BeanstalkdQueue                  │
├────────────────┼────────────────────────────────────────────┤
│ Session        │ FileHandler, DatabaseHandler,              │
│                │ RedisHandler, CookieHandler                │
├────────────────┼────────────────────────────────────────────┤
│ Broadcasting   │ PusherBroadcaster, RedisBroadcaster,      │
│                │ AblyBroadcaster                             │
└────────────────┴────────────────────────────────────────────┘
```

Laravel Filesystem উদাহরণ:

```php
// config/filesystems.php
'disks' => [
    'local' => ['driver' => 'local', 'root' => storage_path('app')],
    's3'    => ['driver' => 's3', 'key' => env('AWS_ACCESS_KEY_ID'), /*...*/],
];

// ব্যবহার — driver পরিবর্তন করলেও কোড একই থাকে
Storage::disk('local')->put('file.txt', 'Hello');
Storage::disk('s3')->put('file.txt', 'Hello');
// Adapter pattern-এর কারণে client code পরিবর্তন করতে হয়নি!
```

### ৫. Database Driver Adapter

```javascript
// Unified Database Interface
class DatabaseAdapter {
    async query(sql, params) { throw new Error('Not implemented'); }
    async insert(table, data) { throw new Error('Not implemented'); }
    async close() { throw new Error('Not implemented'); }
}

// MySQL Adaptee (mysql2 package)
class MySql2Adapter extends DatabaseAdapter {
    #pool;

    constructor(config) {
        super();
        // mysql2 pool ব্যবহার করে
        this.#pool = { execute: async (sql, params) => [[{ id: 1 }]] }; // সিমুলেশন
    }

    async query(sql, params = []) {
        const [rows] = await this.#pool.execute(sql, params);
        return rows;
    }

    async insert(table, data) {
        const keys = Object.keys(data).join(', ');
        const placeholders = Object.keys(data).map(() => '?').join(', ');
        const sql = `INSERT INTO ${table} (${keys}) VALUES (${placeholders})`;
        const [result] = await this.#pool.execute(sql, Object.values(data));
        return result;
    }

    async close() { /* pool.end() */ }
}

// PostgreSQL Adaptee (pg package)
class PgAdapter extends DatabaseAdapter {
    #client;

    constructor(config) {
        super();
        this.#client = { query: async (sql, params) => ({ rows: [{ id: 1 }] }) };
    }

    async query(sql, params = []) {
        // pg uses $1, $2 style params — adapt from ? style
        const pgSql = sql.replace(/\?/g, (() => { let i = 0; return () => `$${++i}`; })());
        const result = await this.#client.query(pgSql, params);
        return result.rows;
    }

    async insert(table, data) {
        const keys = Object.keys(data).join(', ');
        const placeholders = Object.keys(data).map((_, i) => `$${i + 1}`).join(', ');
        const sql = `INSERT INTO ${table} (${keys}) VALUES (${placeholders}) RETURNING *`;
        const result = await this.#client.query(sql, Object.values(data));
        return result.rows[0];
    }

    async close() { /* client.end() */ }
}
```

### ৬. ORM Adapter (Eloquent ↔ Doctrine)

```php
<?php

interface RepositoryInterface
{
    public function find(int $id): ?array;
    public function findBy(array $criteria): array;
    public function save(array $data): int;
    public function delete(int $id): bool;
}

// Eloquent-based Repository
class EloquentUserRepository implements RepositoryInterface
{
    public function __construct(private readonly object $model) {}

    public function find(int $id): ?array
    {
        // $this->model::find($id)?->toArray();
        return ['id' => $id, 'name' => 'Test User'];
    }

    public function findBy(array $criteria): array
    {
        // Eloquent where clause build
        return [];
    }

    public function save(array $data): int
    {
        // $this->model::create($data)->id;
        return 1;
    }

    public function delete(int $id): bool
    {
        // $this->model::destroy($id);
        return true;
    }
}

// Doctrine-based Adapter — একই interface, ভিন্ন ORM
class DoctrineUserRepository implements RepositoryInterface
{
    public function __construct(private readonly object $entityManager) {}

    public function find(int $id): ?array
    {
        // $entity = $this->entityManager->find(User::class, $id);
        return ['id' => $id, 'name' => 'Test User'];
    }

    public function findBy(array $criteria): array
    {
        // $this->entityManager->getRepository(User::class)->findBy($criteria);
        return [];
    }

    public function save(array $data): int
    {
        // $this->entityManager->persist($entity); $this->entityManager->flush();
        return 1;
    }

    public function delete(int $id): bool
    {
        // $this->entityManager->remove($entity); $this->entityManager->flush();
        return true;
    }
}
```

---

## 🔥 Advanced Deep Dive

### ১. Adapter vs Facade vs Proxy vs Decorator — তুলনা

এই চারটি pattern প্রায়ই গুলিয়ে যায় কারণ সবাই একটি object-কে "wrap" করে। কিন্তু তাদের **উদ্দেশ্য** সম্পূর্ণ আলাদা:

| বৈশিষ্ট্য | Adapter | Facade | Proxy | Decorator |
|-----------|---------|--------|-------|-----------|
| **উদ্দেশ্য** | Interface রূপান্তর | জটিল সিস্টেমের সরল interface | Access control / lazy load | নতুন behavior যোগ |
| **Interface** | ভিন্ন interface-কে expected-এ রূপান্তর | নতুন সরলীকৃত interface তৈরি | **একই** interface বজায় রাখে | **একই** interface বজায় রাখে |
| **Wraps** | একটি object | একাধিক subsystem | একটি object | একটি object |
| **Client জানে?** | হ্যাঁ, adapter ব্যবহার করছে জানে | হ্যাঁ, facade ব্যবহার করছে জানে | না, proxy transparent | না (অথবা হ্যাঁ) |
| **কখন** | Third-party API integrate | Complex library simplify | Caching, logging, access | Runtime-এ feature যোগ |
| **BD উদাহরণ** | bKash SDK → PaymentInterface | Pathao API (ride+food+parcel) | Image proxy for CDN | Logging decorator for SMS |

```
Adapter:    [Client] → [Target Interface] → [Adapter] → [Adaptee]
                        (expected)            (converts)   (incompatible)

Facade:     [Client] → [Facade] → [SubsystemA]
                                 → [SubsystemB]
                                 → [SubsystemC]

Proxy:      [Client] → [Proxy] → [RealSubject]
                        (same interface, controls access)

Decorator:  [Client] → [DecoratorB] → [DecoratorA] → [Component]
                        (adds behavior)  (adds behavior)  (original)
```

### ২. Adapter with Dependency Injection

Adapter pattern-এর আসল শক্তি প্রকাশ পায় Dependency Injection-এর সাথে:

```php
<?php

// Laravel Service Provider-এ binding
class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Config-এর উপর ভিত্তি করে adapter নির্বাচন
        $this->app->bind(PaymentGateway::class, function ($app) {
            return match (config('payment.default')) {
                'bkash' => new BkashPaymentAdapter(
                    new BkashPaymentSDK(),
                ),
                'nagad' => new NagadPaymentAdapter(
                    new NagadPaymentSDK(),
                    config('payment.nagad.merchant_id'),
                    config('payment.nagad.callback_url'),
                ),
                'sslcommerz' => new SslCommerzAdapter(
                    new SslCommerzApi(config('payment.sslcommerz')),
                ),
                default => throw new \InvalidArgumentException(
                    'Unsupported payment gateway: ' . config('payment.default')
                ),
            };
        });
    }
}

// Controller-এ inject — কোন adapter ব্যবহার হচ্ছে সেটা জানার দরকার নেই
class OrderController
{
    public function __construct(
        private readonly PaymentGateway $payment,
    ) {}

    public function checkout(Request $request): JsonResponse
    {
        $result = $this->payment->charge(
            amount: $request->float('amount'),
            currency: 'BDT',
            customerRef: $request->string('order_id'),
        );

        return response()->json($result);
    }
}
```

```javascript
// JavaScript — DI Container দিয়ে
class Container {
    #bindings = new Map();

    bind(key, factory) {
        this.#bindings.set(key, factory);
    }

    resolve(key) {
        const factory = this.#bindings.get(key);
        if (!factory) throw new Error(`No binding for ${key}`);
        return factory(this);
    }
}

const container = new Container();

// Config-এর উপর ভিত্তি করে adapter bind
container.bind('PaymentGateway', () => {
    const provider = process.env.PAYMENT_PROVIDER ?? 'bkash';

    switch (provider) {
        case 'bkash':
            return new BkashPaymentAdapter(new BkashPaymentSDK());
        case 'nagad':
            return new NagadPaymentAdapter(
                new NagadPaymentSDK(),
                process.env.NAGAD_MERCHANT_ID,
                process.env.NAGAD_CALLBACK_URL,
            );
        default:
            throw new Error(`Unsupported provider: ${provider}`);
    }
});

const gateway = container.resolve('PaymentGateway');
```

### ৩. Adapter for Legacy Code Migration

ধাপে ধাপে legacy system migrate করার কৌশল:

```php
<?php

// ধাপ ১: Legacy code-এর জন্য interface তৈরি
interface UserService
{
    public function getUser(int $id): array;
    public function createUser(array $data): int;
    public function updateUser(int $id, array $data): bool;
}

// ধাপ ২: Legacy code-কে adapter দিয়ে wrap
class LegacyUserServiceAdapter implements UserService
{
    public function __construct(
        private readonly LegacyUserManager $legacy,
    ) {}

    public function getUser(int $id): array
    {
        // Legacy method যা MySQL procedural call ব্যবহার করে
        $rawData = $this->legacy->fetch_user_data($id);

        // নতুন format-এ রূপান্তর
        return [
            'id' => $rawData['usr_id'],
            'name' => $rawData['usr_fname'] . ' ' . $rawData['usr_lname'],
            'email' => $rawData['usr_email'],
        ];
    }

    public function createUser(array $data): int
    {
        return $this->legacy->insert_user_record(
            $data['name'],
            $data['email'],
            $data['password'] ?? '',
        );
    }

    public function updateUser(int $id, array $data): bool
    {
        return $this->legacy->modify_user_record($id, $data);
    }
}

// ধাপ ৩: নতুন implementation তৈরি (একই interface)
class ModernUserService implements UserService
{
    public function __construct(
        private readonly object $userRepository,
    ) {}

    public function getUser(int $id): array { /* নতুন Eloquent/Doctrine ব্যবহার */ return []; }
    public function createUser(array $data): int { return 1; }
    public function updateUser(int $id, array $data): bool { return true; }
}

// ধাপ ৪: Feature flag দিয়ে ধীরে ধীরে switch করা
class GradualMigrationAdapter implements UserService
{
    public function __construct(
        private readonly LegacyUserServiceAdapter $legacy,
        private readonly ModernUserService $modern,
        private readonly FeatureFlags $flags,
    ) {}

    public function getUser(int $id): array
    {
        if ($this->flags->isEnabled('modern_user_read')) {
            return $this->modern->getUser($id);
        }
        return $this->legacy->getUser($id);
    }

    public function createUser(array $data): int
    {
        if ($this->flags->isEnabled('modern_user_write')) {
            return $this->modern->createUser($data);
        }
        return $this->legacy->createUser($data);
    }

    public function updateUser(int $id, array $data): bool
    {
        if ($this->flags->isEnabled('modern_user_write')) {
            return $this->modern->updateUser($id, $data);
        }
        return $this->legacy->updateUser($id, $data);
    }
}
```

### ৪. Adapter with Generics/Templates (PHP 8.3 Template Pattern)

```php
<?php

// Generic Adapter Pattern — PHP-তে Template দিয়ে
// PHP-তে true generics নেই, তাই convention + PHPStan annotation ব্যবহার

/**
 * @template TInput
 * @template TOutput
 */
interface DataTransformerAdapter
{
    /**
     * @param TInput $input
     * @return TOutput
     */
    public function transform(mixed $input): mixed;

    /**
     * @param TOutput $output
     * @return TInput
     */
    public function reverseTransform(mixed $output): mixed;
}

/**
 * @implements DataTransformerAdapter<array, object>
 */
class ArrayToObjectAdapter implements DataTransformerAdapter
{
    public function __construct(
        private readonly string $className,
    ) {}

    public function transform(mixed $input): mixed
    {
        // Array → Object রূপান্তর
        $reflection = new \ReflectionClass($this->className);
        return $reflection->newInstanceArgs($input);
    }

    public function reverseTransform(mixed $output): mixed
    {
        // Object → Array রূপান্তর
        return (array) $output;
    }
}
```

```javascript
// JavaScript — Generic-like adapter using TypeScript-style JSDoc

/**
 * @template TSource
 * @template TTarget
 */
class GenericAdapter {
    #transformFn;
    #reverseTransformFn;

    /**
     * @param {(source: TSource) => TTarget} transformFn
     * @param {(target: TTarget) => TSource} reverseTransformFn
     */
    constructor(transformFn, reverseTransformFn) {
        this.#transformFn = transformFn;
        this.#reverseTransformFn = reverseTransformFn;
    }

    /** @param {TSource} input @returns {TTarget} */
    transform(input) {
        return this.#transformFn(input);
    }

    /** @param {TTarget} input @returns {TSource} */
    reverseTransform(input) {
        return this.#reverseTransformFn(input);
    }
}

// API Response → Internal Model adapter
const apiResponseAdapter = new GenericAdapter(
    // API format → Internal format
    (apiData) => ({
        id: apiData.user_id,
        fullName: `${apiData.first_name} ${apiData.last_name}`,
        email: apiData.email_address,
        isActive: apiData.status === 'active',
    }),
    // Internal format → API format
    (internal) => ({
        user_id: internal.id,
        first_name: internal.fullName.split(' ')[0],
        last_name: internal.fullName.split(' ').slice(1).join(' '),
        email_address: internal.email,
        status: internal.isActive ? 'active' : 'inactive',
    }),
);

const internalUser = apiResponseAdapter.transform({
    user_id: 42,
    first_name: 'রহিম',
    last_name: 'উদ্দিন',
    email_address: 'rahim@example.com',
    status: 'active',
});
// → { id: 42, fullName: 'রহিম উদ্দিন', email: 'rahim@example.com', isActive: true }
```

---

## ✅ Pros (সুবিধাসমূহ)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| 1 | **Single Responsibility (SRP)** | Interface রূপান্তরের কোড business logic থেকে আলাদা থাকে |
| 2 | **Open/Closed Principle (OCP)** | নতুন adapter যোগ করলে বিদ্যমান কোড পরিবর্তন হয় না |
| 3 | **Reusability** | বিদ্যমান class-কে modification ছাড়াই নতুন system-এ ব্যবহার করা যায় |
| 4 | **Testability** | Mock adapter তৈরি করে সহজে unit test লেখা যায় |
| 5 | **Loose Coupling** | Client এবং adaptee-এর মধ্যে সরাসরি dependency থাকে না |
| 6 | **Legacy Integration** | পুরনো কোড পরিবর্তন না করেই নতুন system-এ integrate করা যায় |
| 7 | **Vendor Lock-in Prevention** | Adapter swap করে সহজে vendor পরিবর্তন করা যায় |

## ❌ Cons (অসুবিধাসমূহ)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| 1 | **Complexity বৃদ্ধি** | নতুন class এবং interface যোগ হওয়ায় কোডবেস বড় হয় |
| 2 | **Indirection** | Debug করার সময় adapter layer-এ trace করতে হয়, সময় বেশি লাগে |
| 3 | **Performance Overhead** | অতিরিক্ত method call এবং data transformation-এর কারণে সামান্য overhead |
| 4 | **Incomplete Adaptation** | কিছু ক্ষেত্রে adaptee-এর সব functionality adapt করা সম্ভব হয় না |
| 5 | **Over-engineering** | সাধারণ ক্ষেত্রে adapter ব্যবহার করলে unnecessary complexity তৈরি হয় |

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ❌ ভুল ১: Adapter-এ Business Logic রাখা

```php
// ❌ ভুল — Adapter-এর কাজ শুধু interface রূপান্তর, business logic নয়
class BadBkashAdapter implements PaymentGateway
{
    public function charge(float $amount, string $currency, string $ref): PaymentResult
    {
        // ❌ Adapter-এ discount calculation করা উচিত নয়!
        if ($amount > 5000) {
            $amount *= 0.95; // 5% discount
        }

        // ❌ Adapter-এ email পাঠানোও উচিত নয়!
        mail('admin@shop.com', 'New Payment', "Amount: $amount");

        $result = $this->bkash->createPayment([...]);
        // ...
    }
}

// ✅ সঠিক — Adapter শুধু interface translate করে
class GoodBkashAdapter implements PaymentGateway
{
    public function charge(float $amount, string $currency, string $ref): PaymentResult
    {
        $response = $this->bkash->createPayment([
            'amount' => $amount,
            'invoice' => $ref,
        ]);

        return new PaymentResult(
            success: $response['status'] === 'Completed',
            transactionId: $response['trxID'],
            amount: (float) $response['amount'],
            currency: $currency,
            message: "bKash: {$response['status']}",
        );
    }
}
```

### ❌ ভুল ২: God Adapter (একটি adapter-এ অনেক adaptee)

```php
// ❌ ভুল — একটি adapter-এ সব payment gateway
class MegaPaymentAdapter implements PaymentGateway
{
    public function charge(float $amount, string $currency, string $ref): PaymentResult
    {
        // ❌ এটি Adapter নয়, এটি একটি mess
        if ($this->provider === 'bkash') { /* ... */ }
        elseif ($this->provider === 'nagad') { /* ... */ }
        elseif ($this->provider === 'sslcommerz') { /* ... */ }
    }
}

// ✅ সঠিক — প্রতিটি adaptee-এর জন্য আলাদা adapter
class BkashPaymentAdapter implements PaymentGateway { /* ... */ }
class NagadPaymentAdapter implements PaymentGateway { /* ... */ }
class SslCommerzAdapter implements PaymentGateway { /* ... */ }
```

### ❌ ভুল ৩: Exception হ্যান্ডলিং ভুলে যাওয়া

```php
// ❌ ভুল — Adaptee-এর exception client-এ leak হচ্ছে
class BadAdapter implements PaymentGateway
{
    public function charge(float $amount, string $currency, string $ref): PaymentResult
    {
        // BkashApiException client পর্যন্ত পৌঁছে যাবে!
        return $this->bkash->createPayment([...]);
    }
}

// ✅ সঠিক — Adaptee exception catch করে নিজস্ব exception বা result দেওয়া
class GoodAdapter implements PaymentGateway
{
    public function charge(float $amount, string $currency, string $ref): PaymentResult
    {
        try {
            $response = $this->bkash->createPayment([
                'amount' => $amount,
                'invoice' => $ref,
            ]);

            return new PaymentResult(
                success: $response['status'] === 'Completed',
                transactionId: $response['trxID'] ?? '',
                amount: $amount,
                currency: $currency,
                message: "Payment processed",
            );
        } catch (\BkashApiException $e) {
            // Adaptee-specific exception → generic PaymentResult
            return new PaymentResult(
                success: false,
                transactionId: '',
                amount: $amount,
                currency: $currency,
                message: "Payment failed: {$e->getMessage()}",
            );
        }
    }
}
```

### ❌ ভুল ৪: Adaptee-কে Expose করা

```php
// ❌ ভুল — Adaptee-কে public করে দেওয়া
class BadAdapter implements PaymentGateway
{
    public BkashPaymentSDK $bkash; // ❌ Client সরাসরি access পায়!

    public function getBkashSdk(): BkashPaymentSDK  // ❌ Leaky abstraction
    {
        return $this->bkash;
    }
}

// ✅ সঠিক — Adaptee সম্পূর্ণ encapsulated
class GoodAdapter implements PaymentGateway
{
    public function __construct(
        private readonly BkashPaymentSDK $bkash, // ✅ private readonly
    ) {}
}
```

---

## 🧪 টেস্টিং (Testing)

### PHPUnit (PHP 8.3)

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\CoversClass;

#[CoversClass(BkashPaymentAdapter::class)]
class BkashPaymentAdapterTest extends TestCase
{
    private BkashPaymentAdapter $adapter;
    private BkashPaymentSDK&\PHPUnit\Framework\MockObject\MockObject $mockBkash;

    protected function setUp(): void
    {
        $this->mockBkash = $this->createMock(BkashPaymentSDK::class);
        $this->adapter = new BkashPaymentAdapter($this->mockBkash);
    }

    #[Test]
    public function charge_returns_successful_payment_result(): void
    {
        // Arrange — Mock adaptee response
        $this->mockBkash
            ->expects($this->once())
            ->method('createPayment')
            ->with($this->callback(fn(array $data) =>
                $data['amount'] === 1500.0 && str_contains($data['invoice'], 'ORD-001')
            ))
            ->willReturn([
                'paymentID' => 'BK123',
                'trxID' => 'TRX456',
                'status' => 'Completed',
                'amount' => 1500.0,
                'merchantInvoiceNumber' => 'INV-ORD-001',
            ]);

        // Act
        $result = $this->adapter->charge(1500.0, 'BDT', 'ORD-001');

        // Assert — Adapter সঠিকভাবে format রূপান্তর করেছে
        $this->assertTrue($result->success);
        $this->assertSame('TRX456', $result->transactionId);
        $this->assertSame(1500.0, $result->amount);
        $this->assertSame('BDT', $result->currency);
        $this->assertStringContainsString('bKash', $result->message);
    }

    #[Test]
    public function charge_returns_failed_result_on_bkash_failure(): void
    {
        $this->mockBkash
            ->method('createPayment')
            ->willReturn([
                'paymentID' => 'BK789',
                'trxID' => 'TRX000',
                'status' => 'Failed',
                'amount' => 500.0,
                'merchantInvoiceNumber' => 'INV-ORD-002',
            ]);

        $result = $this->adapter->charge(500.0, 'BDT', 'ORD-002');

        $this->assertFalse($result->success);
    }

    #[Test]
    public function refund_correctly_adapts_bkash_response(): void
    {
        $this->mockBkash
            ->expects($this->once())
            ->method('refundTransaction')
            ->willReturn([
                'refundTrxID' => 'RF001',
                'transactionStatus' => 'Completed',
                'amount' => 750.0,
                'completedTime' => '2024-01-15 10:30:00',
            ]);

        $result = $this->adapter->refund('TRX456', 750.0);

        $this->assertTrue($result->success);
        $this->assertSame('RF001', $result->refundId);
        $this->assertSame(750.0, $result->amount);
    }

    #[Test]
    #[DataProvider('statusMappingProvider')]
    public function getStatus_maps_bkash_status_correctly(
        string $bkashStatus,
        TransactionStatus $expected
    ): void {
        $this->mockBkash
            ->method('queryPayment')
            ->willReturn([
                'paymentID' => 'BK001',
                'transactionStatus' => $bkashStatus,
            ]);

        $result = $this->adapter->getStatus('BK001');

        $this->assertSame($expected, $result);
    }

    public static function statusMappingProvider(): array
    {
        return [
            'completed' => ['Completed', TransactionStatus::Completed],
            'pending'   => ['Pending', TransactionStatus::Pending],
            'initiated' => ['Initiated', TransactionStatus::Pending],
            'unknown'   => ['SomethingElse', TransactionStatus::Failed],
        ];
    }

    #[Test]
    public function adapter_implements_payment_gateway_interface(): void
    {
        $this->assertInstanceOf(PaymentGateway::class, $this->adapter);
    }
}

// Integration Test — Adapter সঠিকভাবে Client-এর সাথে কাজ করে
#[CoversClass(CheckoutService::class)]
class CheckoutServiceIntegrationTest extends TestCase
{
    #[Test]
    public function checkout_works_with_bkash_adapter(): void
    {
        $mockBkash = $this->createMock(BkashPaymentSDK::class);
        $mockBkash->method('createPayment')->willReturn([
            'paymentID' => 'BK_TEST',
            'trxID' => 'TRX_TEST',
            'status' => 'Completed',
            'amount' => 2000.0,
            'merchantInvoiceNumber' => 'INV-TEST',
        ]);

        $adapter = new BkashPaymentAdapter($mockBkash);
        $checkout = new CheckoutService($adapter);

        $result = $checkout->processPayment(2000.0, 'TEST-001');

        $this->assertTrue($result->success);
    }

    #[Test]
    public function checkout_works_with_nagad_adapter(): void
    {
        $mockNagad = $this->createMock(NagadPaymentSDK::class);
        $mockNagad->method('initiateTransaction')->willReturn(
            (object) [
                'issuerPaymentRefNo' => 'NG_TEST',
                'status' => 'Success',
                'orderID' => 'TEST-001',
                'amount' => 2000.0,
            ]
        );

        $adapter = new NagadPaymentAdapter($mockNagad, 'M001', 'https://test.com/cb');
        $checkout = new CheckoutService($adapter);

        $result = $checkout->processPayment(2000.0, 'TEST-001');

        $this->assertTrue($result->success);
    }
}
```

### Jest (JavaScript ES2022+)

```javascript
import { describe, test, expect, jest, beforeEach } from '@jest/globals';

describe('BkashPaymentAdapter', () => {
    let mockBkash;
    let adapter;

    beforeEach(() => {
        mockBkash = {
            createPayment: jest.fn(),
            refundTransaction: jest.fn(),
            queryPayment: jest.fn(),
        };
        adapter = new BkashPaymentAdapter(mockBkash);
    });

    test('charge() সঠিকভাবে bKash response-কে PaymentResult-এ রূপান্তর করে', async () => {
        // Arrange
        mockBkash.createPayment.mockResolvedValue({
            paymentID: 'BK123',
            trxID: 'TRX456',
            status: 'Completed',
            amount: 1500,
            merchantInvoiceNumber: 'INV-ORD-001',
        });

        // Act
        const result = await adapter.charge(1500, 'BDT', 'ORD-001');

        // Assert
        expect(result).toBeInstanceOf(PaymentResult);
        expect(result.success).toBe(true);
        expect(result.transactionId).toBe('TRX456');
        expect(result.amount).toBe(1500);
        expect(result.currency).toBe('BDT');
        expect(result.message).toContain('bKash');
    });

    test('charge() ব্যর্থ হলে success: false রিটার্ন করে', async () => {
        mockBkash.createPayment.mockResolvedValue({
            paymentID: 'BK789',
            trxID: 'TRX000',
            status: 'Failed',
            amount: 500,
        });

        const result = await adapter.charge(500, 'BDT', 'ORD-002');

        expect(result.success).toBe(false);
    });

    test('refund() সঠিকভাবে bKash refund response adapt করে', async () => {
        mockBkash.refundTransaction.mockResolvedValue({
            refundTrxID: 'RF001',
            transactionStatus: 'Completed',
            amount: 750,
            completedTime: '2024-01-15T10:30:00Z',
        });

        const result = await adapter.refund('TRX456', 750);

        expect(result).toBeInstanceOf(RefundResult);
        expect(result.success).toBe(true);
        expect(result.refundId).toBe('RF001');
        expect(result.amount).toBe(750);
    });

    test('getStatus() bKash status-কে সঠিকভাবে map করে', async () => {
        const statusMappings = [
            ['Completed', 'completed'],
            ['Pending', 'pending'],
            ['Initiated', 'pending'],
            ['RandomStatus', 'failed'],
        ];

        for (const [bkashStatus, expected] of statusMappings) {
            mockBkash.queryPayment.mockResolvedValue({
                paymentID: 'BK001',
                transactionStatus: bkashStatus,
            });

            const result = await adapter.getStatus('BK001');
            expect(result).toBe(expected);
        }
    });

    test('createPayment() সঠিক parameter সহ call হয়', async () => {
        mockBkash.createPayment.mockResolvedValue({
            paymentID: 'BK1', trxID: 'T1', status: 'Completed', amount: 100,
        });

        await adapter.charge(100, 'BDT', 'REF-123');

        expect(mockBkash.createPayment).toHaveBeenCalledWith({
            amount: 100,
            invoice: 'INV-REF-123',
        });
    });
});

describe('CheckoutService Integration', () => {
    test('bKash adapter-এর সাথে সঠিকভাবে কাজ করে', async () => {
        const mockBkash = {
            createPayment: jest.fn().mockResolvedValue({
                paymentID: 'BK_T', trxID: 'TRX_T',
                status: 'Completed', amount: 2000,
            }),
            refundTransaction: jest.fn(),
            queryPayment: jest.fn(),
        };

        const adapter = new BkashPaymentAdapter(mockBkash);
        const checkout = new CheckoutService(adapter);

        const result = await checkout.processPayment(2000, 'TEST-001');

        expect(result.success).toBe(true);
    });

    test('PaymentGateway interface ছাড়া reject করে', () => {
        const fakeGateway = {};

        expect(() => new CheckoutService(fakeGateway)).toThrow(TypeError);
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Bridge Pattern
- **পার্থক্য**: Bridge abstraction এবং implementation-কে **শুরু থেকেই** আলাদা করে design করা হয়। Adapter **পরে** incompatible interface-কে সংযুক্ত করে।
- **কখন Bridge**: যখন আপনি design phase-এ আছেন এবং multiple dimension-এ vary করতে চান।
- **কখন Adapter**: যখন existing incompatible class-কে ব্যবহার করতে চান।

### Decorator Pattern
- **পার্থক্য**: Decorator **একই interface** রেখে behavior যোগ করে। Adapter **ভিন্ন interface**-এ রূপান্তর করে।
- **মিল**: উভয়ই wrapping technique ব্যবহার করে।
- **একসাথে**: Adapter-কে Decorator দিয়ে wrap করা যায় (যেমন logging decorator on payment adapter)।

### Facade Pattern
- **পার্থক্য**: Facade একাধিক subsystem-এর জন্য **একটি সরলীকৃত interface** দেয়। Adapter **একটি** existing interface-কে অন্যটিতে রূপান্তর করে।
- **কখন Facade**: জটিল library-র সরল API চাইলে।
- **কখন Adapter**: Incompatible interface-কে compatible করতে চাইলে।

### Proxy Pattern
- **পার্থক্য**: Proxy **একই interface** রেখে access control, caching, বা lazy loading যোগ করে। Adapter interface রূপান্তর করে।
- **মিল**: উভয়ই একটি object-কে wrap করে।

```
সম্পর্ক ডায়াগ্রাম:

Bridge ←── (design-time separation) ──→ Adapter (runtime adaptation)
   │                                         │
   │ "আগে থেকেই আলাদা করি"         "পরে জোড়া লাগাই"
   │                                         │
Decorator ←── (same interface, add) ──→ Adapter (different interface, convert)
   │                                         │
   │ "behavior যোগ করি"             "interface translate করি"
   │                                         │
Facade ←── (simplify multiple) ───→ Adapter (convert single)
   │                                         │
   │ "জটিল → সরল"                  "incompatible → compatible"
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

1. **Third-party library integration**: বাহ্যিক SDK/API-র interface আপনার system-এর সাথে মেলে না
2. **Legacy system migration**: পুরনো কোড পরিবর্তন না করেই নতুন system-এ integrate করতে চান
3. **Multiple vendor support**: একই কাজের জন্য একাধিক vendor (bKash, Nagad, Rocket) সাপোর্ট করতে হবে
4. **Testing**: External dependency-কে mock করার জন্য adapter layer রাখলে test করা সহজ হয়
5. **Interface standardization**: বিভিন্ন format-এর data source-কে একটি unified interface-এ আনতে চান
6. **Vendor lock-in এড়াতে**: Adapter layer থাকলে vendor পরিবর্তন করতে শুধু নতুন adapter লিখলেই হয়

### ❌ ব্যবহার করবেন না যখন:

1. **Interface ইতিমধ্যে compatible**: যদি existing class আপনার interface-এর সাথে মিলে যায়, adapter অপ্রয়োজনীয়
2. **Simple cases**: শুধু একটি method name পরিবর্তনের জন্য পুরো adapter class তৈরি করা over-engineering
3. **Adaptee পরিবর্তন সম্ভব**: যদি adaptee-এর source code আপনার নিয়ন্ত্রণে থাকে, সরাসরি interface implement করুন
4. **Performance-critical path**: High-throughput loop-এ adapter-এর indirection overhead সমস্যা হতে পারে
5. **একটিমাত্র implementation**: যদি ভবিষ্যতে কখনোই দ্বিতীয় adapter লাগবে না, তাহলে abstraction অপ্রয়োজনীয় (YAGNI)

### সিদ্ধান্ত নেওয়ার Flowchart:

```
existing class-এর interface কি আপনার system-এর সাথে মেলে?
├── হ্যাঁ → ❌ Adapter লাগবে না, সরাসরি ব্যবহার করুন
└── না → class-এর source code কি আপনার নিয়ন্ত্রণে?
    ├── হ্যাঁ → সরাসরি interface implement করান (Adapter লাগবে না)
    └── না → ভবিষ্যতে কি alternative implementation আসতে পারে?
        ├── হ্যাঁ → ✅ Adapter Pattern ব্যবহার করুন
        └── না → Adapter তৈরি করুন তবে simpler version-এ
```

---

## 📋 সারসংক্ষেপ (Summary)

### মূল বিষয়সমূহ:

| বিষয় | সারাংশ |
|--------|---------|
| **কী করে** | Incompatible interface-কে compatible করে |
| **কেন দরকার** | Third-party code modify না করেই integrate করার জন্য |
| **দুই ধরন** | Object Adapter (composition — ✅ বেশি ব্যবহৃত) এবং Class Adapter (inheritance) |
| **SOLID** | SRP ও OCP চমৎকারভাবে মানে; DIP-এর সাথে ব্যবহার করলে সর্বোচ্চ ফলাফল |
| **সতর্কতা** | Adapter-এ business logic রাখবেন না; exception handle করুন; adaptee expose করবেন না |
| **BD প্রেক্ষাপট** | bKash/Nagad/Rocket adapter, BD SMS gateway adapter, SSLCommerz integration |

### Quick Reference:

```php
// ১. Target Interface তৈরি করুন
interface PaymentGateway {
    public function charge(float $amount): PaymentResult;
}

// ২. Adapter class তৈরি করুন — Target implement + Adaptee wrap
class BkashAdapter implements PaymentGateway {
    public function __construct(private readonly BkashSDK $bkash) {}

    public function charge(float $amount): PaymentResult {
        $response = $this->bkash->createPayment($amount); // adaptee call
        return new PaymentResult($response);               // format রূপান্তর
    }
}

// ৩. Client শুধু Target Interface ব্যবহার করে
class Checkout {
    public function __construct(private readonly PaymentGateway $gw) {}
    public function pay(float $amount): void {
        $this->gw->charge($amount); // কোন adapter — জানার দরকার নেই
    }
}
```

### মনে রাখার সূত্র:

> **"Adapter = Translator"** — দুটি ভিন্ন ভাষায় কথা বলা মানুষের মধ্যে একজন দোভাষীর মতো কাজ করে। দোভাষী (adapter) নিজে কোনো নতুন কথা বলে না — শুধু এক পক্ষের কথা অন্য পক্ষের বোধগম্য ভাষায় অনুবাদ করে দেয়।

---

> **📚 আরও পড়ুন:**
> - *Design Patterns: Elements of Reusable Object-Oriented Software* — GoF (Chapter: Adapter)
> - *Head First Design Patterns* — Chapter 7
> - *Refactoring to Patterns* — Joshua Kerievsky
> - PHP: [Flysystem](https://flysystem.thephpleague.com/) — File storage adapter-এর চমৎকার বাস্তব উদাহরণ
> - Laravel Source: `Illuminate\Filesystem\FilesystemAdapter`
