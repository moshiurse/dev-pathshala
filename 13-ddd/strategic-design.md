# 📌 DDD Strategic Design (কৌশলগত নকশা)

> **"ভালো সফটওয়্যার তৈরি হয় ডোমেইন বোঝার মাধ্যমে, শুধু কোড লেখার মাধ্যমে নয়।"**
> — Eric Evans, Domain-Driven Design

Strategic Design হলো DDD-এর সবচেয়ে গুরুত্বপূর্ণ অংশ। এটি আমাদের শেখায় কীভাবে একটি
বড় সিস্টেমকে ছোট ছোট অংশে ভাগ করতে হয়, কীভাবে টিমের সবাই একই ভাষায় কথা বলবে,
এবং কীভাবে বিভিন্ন অংশের মধ্যে সম্পর্ক তৈরি করতে হয়।

```
┌─────────────────────────────────────────────────────┐
│              DDD Strategic Design                    │
│                                                     │
│   ┌──────────────┐  ┌──────────────┐                │
│   │  Ubiquitous  │  │   Bounded    │                │
│   │  Language    │  │   Contexts   │                │
│   └──────┬───────┘  └──────┬───────┘                │
│          │                 │                        │
│          ▼                 ▼                        │
│   ┌──────────────┐  ┌──────────────┐                │
│   │   Context    │  │  Subdomains  │                │
│   │   Maps       │  │              │                │
│   └──────────────┘  └──────────────┘                │
└─────────────────────────────────────────────────────┘
```

---

## 📖 সূচিপত্র

1. [Ubiquitous Language (সর্বজনীন ভাষা)](#-ubiquitous-language-সর্বজনীন-ভাষা)
2. [Bounded Contexts (সীমানাবদ্ধ প্রসঙ্গ)](#-bounded-contexts-সীমানাবদ্ধ-প্রসঙ্গ)
3. [Context Maps ও Relationships](#-context-maps-ও-relationships)
4. [Subdomains (সাবডোমেইন)](#-subdomains-সাবডোমেইন)
5. [Real-world E-commerce Example](#-real-world-e-commerce-example)
6. [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📖 Ubiquitous Language (সর্বজনীন ভাষা)

### এটা কী?

Ubiquitous Language হলো এমন একটি **সাধারণ ভাষা** যা ডেভেলপার, প্রোডাক্ট ম্যানেজার,
বিজনেস অ্যানালিস্ট — টিমের **প্রত্যেকে** ব্যবহার করে। কোডে যে নাম থাকবে, মিটিংয়েও সেই
একই নাম ব্যবহার হবে। কোনো অনুবাদ বা রূপান্তর লাগবে না।

### কেন গুরুত্বপূর্ণ?

ধরুন bKash-এর মতো একটি সিস্টেমে টাকা পাঠানোর ফিচার তৈরি হচ্ছে:

```
❌ খারাপ — বিভিন্ন জায়গায় বিভিন্ন শব্দ:

  বিজনেস টিম বলে:    "Send Money"
  ডেভেলপার বলে:       "Transfer"
  ডাটাবেজে আছে:       "transaction"
  API-তে আছে:         "remittance"
  UI-তে দেখায়:         "টাকা পাঠান"

  ফলাফল → বিভ্রান্তি, বাগ, ভুল বোঝাবুঝি

✅ ভালো — সবাই একই শব্দ ব্যবহার করে:

  বিজনেস টিম বলে:    "SendMoney"
  ডেভেলপার বলে:       "SendMoney"
  ডাটাবেজে আছে:       "send_money"
  API-তে আছে:         "/send-money"
  কোডে ক্লাস:          SendMoneyRequest

  ফলাফল → সবাই একই পৃষ্ঠায়, কম ভুল
```

### ❌ Bad: বিভিন্ন টার্ম ব্যবহার (PHP)

```php
// ❌ খারাপ — বিজনেস বলে "SendMoney" কিন্তু কোডে আছে "Transfer"
class TransferService
{
    // বিজনেস বলে "sender" কিন্তু কোডে আছে "fromUser"
    public function makeTransaction(int $fromUser, int $toUser, float $amt): void
    {
        // "amt" কী? amount? বিজনেস টিম বুঝবে না
        $txn = new Transaction(); // "txn" — সংক্ষেপে লেখা, অস্পষ্ট
        $txn->src = $fromUser;    // "src" মানে কী?
        $txn->dst = $toUser;      // "dst" মানে কী?
        $txn->amt = $amt;
        $txn->save();
    }
}
```

### ✅ Good: Ubiquitous Language মেনে চলা (PHP)

```php
// ✅ ভালো — বিজনেস যা বলে, কোডেও তাই
class SendMoneyService
{
    public function sendMoney(
        Sender $sender,
        Receiver $receiver,
        Money $amount
    ): SendMoneyConfirmation {
        $request = new SendMoneyRequest(
            sender: $sender,
            receiver: $receiver,
            amount: $amount
        );

        // বিজনেস রুল: দৈনিক লিমিট চেক
        $this->dailyLimitPolicy->verify($request);

        $confirmation = $this->wallet->debitAndCredit($request);

        return $confirmation;
    }
}
```

### ✅ Good: Ubiquitous Language (JavaScript)

```javascript
// ✅ ভালো — বিজনেস ল্যাঙ্গুয়েজ সরাসরি কোডে
class SendMoneyService {
  async sendMoney(sender, receiver, amount) {
    const sendMoneyRequest = new SendMoneyRequest({
      sender,
      receiver,
      amount,
    });

    // বিজনেস রুল ভেরিফিকেশন — নামগুলো বিজনেসের সাথে মেলে
    await this.dailyLimitPolicy.verify(sendMoneyRequest);
    await this.fraudDetection.screen(sendMoneyRequest);

    const confirmation = await this.wallet.debitAndCredit(sendMoneyRequest);

    await this.notificationService.notifySender(confirmation);
    await this.notificationService.notifyReceiver(confirmation);

    return confirmation;
  }
}
```

### 🎯 Ubiquitous Language তৈরির পদ্ধতি

```
  ১. ডোমেইন এক্সপার্টদের সাথে বসুন
              │
              ▼
  ২. তাঁরা যে শব্দ ব্যবহার করেন, লিখে রাখুন
              │
              ▼
  ৩. গ্লসারি/অভিধান তৈরি করুন
              │
              ▼
  ৪. কোডে সেই শব্দগুলোই ব্যবহার করুন
              │
              ▼
  ৫. নিয়মিত রিভিউ ও আপডেট করুন
```

---

## 📊 Bounded Contexts (সীমানাবদ্ধ প্রসঙ্গ)

### এটা কী?

Bounded Context হলো একটি **স্পষ্ট সীমানা** যার ভেতরে একটি নির্দিষ্ট ডোমেইন মডেল
প্রযোজ্য। একই শব্দ বিভিন্ন Bounded Context-এ **ভিন্ন অর্থ** বহন করতে পারে।

### বাস্তব উদাহরণ: "Product" শব্দটি ভিন্ন Context-এ

একটি Daraz-এর মতো ই-কমার্স সিস্টেমে "Product" শব্দটি বিভিন্ন জায়গায় ভিন্ন জিনিস বোঝায়:

```
┌────────────────────────────────────────────────────────────────┐
│                    একটি ই-কমার্স সিস্টেম                        │
│                                                                │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │  📦 Catalog BC   │  │  📋 Inventory BC │                    │
│  │                  │  │                  │                    │
│  │  Product:        │  │  Product:        │                    │
│  │  - name          │  │  - sku           │                    │
│  │  - description   │  │  - quantity      │                    │
│  │  - images        │  │  - warehouse     │                    │
│  │  - price         │  │  - reorderLevel  │                    │
│  │  - reviews       │  │  - supplier      │                    │
│  └──────────────────┘  └──────────────────┘                    │
│                                                                │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │  🚚 Shipping BC  │  │  💰 Billing BC   │                    │
│  │                  │  │                  │                    │
│  │  Product:        │  │  Product:        │                    │
│  │  - weight        │  │  - unitPrice     │                    │
│  │  - dimensions    │  │  - taxCategory   │                    │
│  │  - isFragile     │  │  - discount      │                    │
│  │  - destination   │  │  - vatRate       │                    │
│  └──────────────────┘  └──────────────────┘                    │
│                                                                │
│  💡 একই "Product" — কিন্তু প্রতিটি Context-এ ভিন্ন অর্থ!      │
└────────────────────────────────────────────────────────────────┘
```

### ❌ Bad: একটি God Model সব Context-এ (PHP)

```php
// ❌ খারাপ — একটি বিশাল Product ক্লাস সবকিছু করে
class Product
{
    public int $id;
    public string $name;
    public string $description;
    public array $images;
    public float $price;
    public float $weight;
    public string $dimensions;
    public bool $isFragile;
    public int $stockQuantity;
    public string $warehouse;
    public int $reorderLevel;
    public float $vatRate;
    public string $taxCategory;
    // ... আরো ৫০টি প্রপার্টি

    // Catalog-এর কাজ
    public function updateDescription(): void { /* ... */ }

    // Inventory-এর কাজ
    public function adjustStock(): void { /* ... */ }

    // Shipping-এর কাজ
    public function calculateShippingCost(): float { /* ... */ }

    // Billing-এর কাজ
    public function applyDiscount(): void { /* ... */ }
}
// সমস্যা: ক্লাসটি ক্রমাগত বড় হতে থাকে, সব টিম একই ফাইল এডিট করে
```

### ✅ Good: আলাদা Bounded Context-এ আলাদা মডেল (PHP)

```php
// ✅ ভালো — প্রতিটি Bounded Context-এর নিজস্ব Product মডেল

// --- Catalog Bounded Context ---
namespace App\Catalog\Domain;

class Product
{
    public function __construct(
        private ProductId $id,
        private string $name,
        private string $description,
        private Money $price,
        private array $images
    ) {}

    public function updateDescription(string $newDescription): void
    {
        $this->description = $newDescription;
    }
}

// --- Inventory Bounded Context ---
namespace App\Inventory\Domain;

class StockItem
{
    public function __construct(
        private ProductId $productId,
        private Sku $sku,
        private int $quantity,
        private Warehouse $warehouse,
        private int $reorderLevel
    ) {}

    public function adjustQuantity(int $adjustment): void
    {
        $this->quantity += $adjustment;

        if ($this->quantity <= $this->reorderLevel) {
            DomainEventPublisher::publish(
                new LowStockAlert($this->productId, $this->quantity)
            );
        }
    }
}

// --- Shipping Bounded Context ---
namespace App\Shipping\Domain;

class Parcel
{
    public function __construct(
        private ProductId $productId,
        private Weight $weight,
        private Dimensions $dimensions,
        private bool $isFragile
    ) {}

    public function calculateShippingCost(Address $destination): Money
    {
        // শিপিং-এর নিজস্ব লজিক
    }
}
```

### ✅ Good: Bounded Context (JavaScript)

```javascript
// --- Catalog Context ---
// src/catalog/domain/Product.js
class CatalogProduct {
  constructor({ id, name, description, price, images }) {
    this.id = id;
    this.name = name;
    this.description = description;
    this.price = price;
    this.images = images;
  }

  updatePrice(newPrice) {
    if (newPrice <= 0) throw new Error("মূল্য শূন্যের বেশি হতে হবে");
    this.price = newPrice;
  }
}

// --- Inventory Context ---
// src/inventory/domain/StockItem.js
class StockItem {
  constructor({ productId, sku, quantity, warehouse, reorderLevel }) {
    this.productId = productId;
    this.sku = sku;
    this.quantity = quantity;
    this.warehouse = warehouse;
    this.reorderLevel = reorderLevel;
  }

  adjustQuantity(amount) {
    this.quantity += amount;
    if (this.quantity <= this.reorderLevel) {
      this.addDomainEvent(new LowStockAlert(this.productId, this.quantity));
    }
  }
}
```

### 📁 ফোল্ডার স্ট্রাকচার

```
src/
├── catalog/              ← Catalog Bounded Context
│   ├── domain/
│   │   ├── Product.php
│   │   └── Review.php
│   ├── application/
│   │   └── AddProductService.php
│   └── infrastructure/
│       └── ProductRepository.php
│
├── inventory/            ← Inventory Bounded Context
│   ├── domain/
│   │   ├── StockItem.php
│   │   └── Warehouse.php
│   ├── application/
│   │   └── AdjustStockService.php
│   └── infrastructure/
│       └── StockRepository.php
│
├── shipping/             ← Shipping Bounded Context
│   ├── domain/
│   │   ├── Parcel.php
│   │   └── ShippingRoute.php
│   └── ...
│
└── billing/              ← Billing Bounded Context
    ├── domain/
    │   ├── Invoice.php
    │   └── LineItem.php
    └── ...
```

---

## 🔗 Context Maps ও Relationships

### Context Map কী?

Context Map হলো একটি **ভিজ্যুয়াল ডায়াগ্রাম** যা দেখায় বিভিন্ন Bounded Context
একে অপরের সাথে কীভাবে সম্পর্কিত। এটি টিমগুলোর মধ্যকার সম্পর্কও প্রকাশ করে।

### 🏠 সম্পূর্ণ Context Map (Daraz-সদৃশ সিস্টেম)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTEXT MAP                                  │
│                  (Daraz-সদৃশ ই-কমার্স)                               │
│                                                                     │
│                                                                     │
│   ┌──────────┐    Partnership     ┌──────────┐                      │
│   │ Catalog  │◄══════════════════►│  Search  │                      │
│   │ Context  │    (অংশীদারিত্ব)    │ Context  │                      │
│   └────┬─────┘                    └──────────┘                      │
│        │                                                            │
│        │ Customer-Supplier                                          │
│        │ (কাস্টমার-সাপ্লায়ার)                                        │
│        ▼                                                            │
│   ┌──────────┐   Shared Kernel   ┌──────────┐                      │
│   │  Order   │◄═════════════════►│ Payment  │                      │
│   │ Context  │  (শেয়ার্ড কার্নেল)  │ Context  │                      │
│   └────┬─────┘                   └────┬─────┘                      │
│        │                              │                            │
│        │ Conformist                   │ ACL                        │
│        │ (কনফর্মিস্ট)                  │ (দূষণরোধী স্তর)              │
│        ▼                              ▼                            │
│   ┌──────────┐                  ┌──────────────┐                   │
│   │Inventory │                  │   bKash/     │                   │
│   │ Context  │                  │ Nagad Gateway│                   │
│   └────┬─────┘                  │  (External)  │                   │
│        │                        └──────────────┘                   │
│        │ Open Host Service                                         │
│        │ (উন্মুক্ত হোস্ট সেবা)                                       │
│        ▼                                                           │
│   ┌──────────┐                  ┌──────────┐                       │
│   │ Shipping │                  │Notification│                     │
│   │ Context  │                  │  Context   │                     │
│   └──────────┘                  └────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### সম্পর্কের ধরনসমূহ বিস্তারিত

---

#### ১. Partnership (অংশীদারিত্ব)

দুটি টিম **সমানভাবে** সহযোগিতা করে। কোনো একটি টিম আরেকটির উপর নির্ভরশীল নয়।
তারা একসাথে পরিকল্পনা করে এবং একসাথে ডেলিভার করে।

**বাস্তব উদাহরণ:** Grameenphone-এ MyGP অ্যাপ টিম এবং Offer Engine টিম — দুজনেই
সমানভাবে কাজ করে নতুন অফার ফিচার ডেলিভার করতে।

```
  ┌──────────┐         ┌──────────┐
  │ Team A   │◄═══════►│ Team B   │
  │ (Catalog)│  সমান    │ (Search) │
  └──────────┘ সহযোগিতা └──────────┘
```

---

#### ২. Shared Kernel (শেয়ার্ড কার্নেল)

দুটি Bounded Context একটি **ছোট, শেয়ার করা মডেল** ব্যবহার করে। এই শেয়ার করা অংশ
পরিবর্তন করতে হলে **দুই টিমকেই রাজি হতে হবে**।

**বাস্তব উদাহরণ:** bKash-এ Order ও Payment Context — দুটোই `Money` Value Object
এবং `TransactionId` শেয়ার করে।

```
  ┌──────────┐                    ┌──────────┐
  │  Order   │                    │ Payment  │
  │ Context  │                    │ Context  │
  └────┬─────┘                    └────┬─────┘
       │    ┌────────────────┐         │
       └───►│ Shared Kernel  │◄────────┘
            │  - Money       │
            │  - Transaction │
            └────────────────┘
```

---

#### ৩. Customer-Supplier (কাস্টমার-সাপ্লায়ার)

Upstream (সাপ্লায়ার) টিম API/ডেটা সরবরাহ করে, Downstream (কাস্টমার) টিম সেটা
ব্যবহার করে। Upstream টিমের কিছু দায়িত্ব আছে Downstream-এর চাহিদা পূরণ করার।

**বাস্তব উদাহরণ:** Pathao-তে Ride Context (upstream) রাইডের তথ্য সরবরাহ করে
এবং Billing Context (downstream) সেই তথ্য থেকে বিল তৈরি করে।

```
  ┌──────────────┐
  │   Catalog    │  ← Upstream (Supplier)
  │   Context    │
  └──────┬───────┘
         │  API / Events
         ▼
  ┌──────────────┐
  │    Order     │  ← Downstream (Customer)
  │   Context    │
  └──────────────┘
```

---

#### ৪. Conformist (কনফর্মিস্ট)

Downstream টিম Upstream-এর মডেল **হুবহু** মেনে চলে। Upstream টিম Downstream-এর
চাহিদা অনুযায়ী কিছু পরিবর্তন করবে না।

**বাস্তব উদাহরণ:** একটি ছোট স্টার্টআপ যখন Facebook Login API ব্যবহার করে — Facebook
তাদের জন্য API বদলাবে না, তাই স্টার্টআপকে Facebook-এর মডেল হুবহু মেনে চলতে হয়।

```
  ┌──────────────┐
  │  Facebook    │  ← Upstream (তারা পরিবর্তন করবে না)
  │  Auth API    │
  └──────┬───────┘
         │
         ▼  হুবহু মেনে চলো
  ┌──────────────┐
  │  আমাদের     │  ← Downstream (আমরা মানিয়ে নিই)
  │  Login       │
  └──────────────┘
```

---

#### ৫. Anti-corruption Layer — ACL (দূষণরোধী স্তর)

বাইরের সিস্টেমের মডেল আমাদের সিস্টেমে **দূষণ** ছড়াতে পারে। ACL হলো একটি **অনুবাদ
স্তর** যা বাইরের মডেলকে আমাদের নিজস্ব মডেলে রূপান্তর করে।

**বাস্তব উদাহরণ:** Daraz যখন bKash/Nagad পেমেন্ট গেটওয়ে ইন্টিগ্রেট করে — bKash-এর
API রেসপন্স সরাসরি ব্যবহার না করে, একটি ACL দিয়ে নিজেদের Payment মডেলে রূপান্তর করে।

```
  ┌─────────────┐     ┌──────────────┐     ┌──────────────┐
  │  bKash API  │────►│     ACL      │────►│  আমাদের     │
  │  (External) │     │  (অনুবাদক)   │     │  Payment     │
  │             │     │              │     │  Model       │
  │ txnId       │     │ txnId →      │     │ paymentId    │
  │ txnStatus   │     │  paymentId   │     │ status       │
  │ amount      │     │ txnStatus →  │     │ amount       │
  │ currency    │     │  status      │     │              │
  └─────────────┘     └──────────────┘     └──────────────┘
```

### ✅ Anti-corruption Layer (PHP)

```php
// বাইরের bKash API-এর রেসপন্স
// {
//   "trxID": "ABC123",
//   "trxStatus": "Completed",
//   "amount": "500.00",
//   "currency": "BDT",
//   "merchantInvoiceNumber": "INV-001"
// }

// --- ACL: বাইরের মডেলকে আমাদের মডেলে রূপান্তর ---
namespace App\Payment\Infrastructure\ACL;

class BkashPaymentTranslator
{
    public function translateResponse(array $bkashResponse): PaymentConfirmation
    {
        return new PaymentConfirmation(
            paymentId: new PaymentId($bkashResponse['trxID']),
            status: $this->mapStatus($bkashResponse['trxStatus']),
            amount: Money::BDT((float) $bkashResponse['amount']),
            orderId: new OrderId($bkashResponse['merchantInvoiceNumber'])
        );
    }

    private function mapStatus(string $bkashStatus): PaymentStatus
    {
        return match ($bkashStatus) {
            'Completed' => PaymentStatus::CONFIRMED,
            'Failed'    => PaymentStatus::FAILED,
            'Cancelled' => PaymentStatus::CANCELLED,
            default     => PaymentStatus::UNKNOWN,
        };
    }
}

// --- ব্যবহার ---
class BkashPaymentGateway implements PaymentGateway
{
    public function __construct(
        private BkashApiClient $client,
        private BkashPaymentTranslator $translator // ACL
    ) {}

    public function processPayment(PaymentRequest $request): PaymentConfirmation
    {
        $bkashResponse = $this->client->executePayment(
            amount: $request->amount->value(),
            invoice: $request->orderId->value()
        );

        // ACL দিয়ে অনুবাদ — bKash-এর মডেল আমাদের কোডে ঢুকছে না
        return $this->translator->translateResponse($bkashResponse);
    }
}
```

### ✅ Anti-corruption Layer (JavaScript)

```javascript
// --- ACL: Nagad API রেসপন্স থেকে আমাদের মডেলে রূপান্তর ---
class NagadPaymentTranslator {
  translateResponse(nagadResponse) {
    return new PaymentConfirmation({
      paymentId: nagadResponse.issuerPaymentRefNo,
      status: this.#mapStatus(nagadResponse.status),
      amount: Money.bdt(parseFloat(nagadResponse.amount)),
      orderId: nagadResponse.orderId,
      paidAt: new Date(nagadResponse.issuerPaymentDateTime),
    });
  }

  #mapStatus(nagadStatus) {
    const statusMap = {
      Success: "CONFIRMED",
      Failed: "FAILED",
      Aborted: "CANCELLED",
    };
    return statusMap[nagadStatus] ?? "UNKNOWN";
  }
}

// --- ব্যবহার ---
class NagadPaymentGateway {
  #client;
  #translator;

  constructor(client, translator) {
    this.#client = client;
    this.#translator = translator; // ACL
  }

  async processPayment(paymentRequest) {
    const nagadResponse = await this.#client.completePayment({
      amount: paymentRequest.amount.value,
      orderId: paymentRequest.orderId,
    });

    // ACL: বাইরের API-এর structure আমাদের domain-এ ঢুকছে না
    return this.#translator.translateResponse(nagadResponse);
  }
}
```

---

#### ৬. Open Host Service (উন্মুক্ত হোস্ট সেবা)

একটি Bounded Context তার সেবা **পাবলিক API** হিসেবে প্রকাশ করে, যাতে যেকোনো
Context সেটা ব্যবহার করতে পারে। সাধারণত REST API বা Event হিসেবে থাকে।

**বাস্তব উদাহরণ:** Pathao-র Ride Service একটি Open Host Service — Food, Courier
সবাই এটি ব্যবহার করে rider availability জানতে।

```
                      ┌──────────────┐
            ┌────────►│              │
            │         │  Inventory   │
  ┌─────────┴──┐      │  Open Host   │
  │   Order    │      │  Service     │
  │  Context   │      │  (REST API)  │
  └─────────┬──┘      │              │
            │         └──────┬───────┘
            │                │
            │         ┌──────┴───────┐
            └────────►│  Shipping    │
                      │  Context     │
                      └──────────────┘
```

---

## 🏠 Subdomains (সাবডোমেইন)

### সাবডোমেইন কী?

সাবডোমেইন হলো বিজনেস ডোমেইনের **প্রাকৃতিক বিভাজন**। প্রতিটি সাবডোমেইন ব্যবসার
একটি নির্দিষ্ট ক্ষেত্র বা সমস্যা সমাধান করে।

### তিন ধরনের সাবডোমেইন

```
┌──────────────────────────────────────────────────────────────┐
│                    সাবডোমেইন শ্রেণিবিভাগ                       │
│                                                              │
│  ┌─────────────────┐                                         │
│  │   🏆 Core       │  প্রতিযোগিতামূলক সুবিধা                  │
│  │   Domain        │  সবচেয়ে বেশি বিনিয়োগ করুন               │
│  │                 │  সেরা ডেভেলপাররা এখানে কাজ করবে          │
│  └─────────────────┘                                         │
│           │                                                  │
│  ┌─────────────────┐                                         │
│  │  🔧 Supporting  │  প্রয়োজনীয় কিন্তু পার্থক্য তৈরি করে না │
│  │   Domain        │  ইন-হাউজ তৈরি করা দরকার                 │
│  │                 │  কিন্তু সেরা ডেভেলপার দরকার নেই          │
│  └─────────────────┘                                         │
│           │                                                  │
│  ┌─────────────────┐                                         │
│  │  📦 Generic     │  সব ব্যবসায় সাধারণ                      │
│  │   Domain        │  কিনে নিন বা ওপেন সোর্স ব্যবহার করুন    │
│  │                 │  নিজে তৈরি করার দরকার নেই                │
│  └─────────────────┘                                         │
└──────────────────────────────────────────────────────────────┘
```

### বাস্তব উদাহরণ: Pathao

```
┌───────────────────────────────────────────────────────────────┐
│                    Pathao সাবডোমেইন ম্যাপ                      │
│                                                               │
│  🏆 Core Domain (মূল — প্রতিযোগিতামূলক সুবিধা):               │
│  ┌──────────────────────────────────────────┐                 │
│  │  • Ride Matching Algorithm               │                 │
│  │    (কোন রাইডার কোন যাত্রীকে পাবে)        │                 │
│  │  • Dynamic Pricing Engine                │                 │
│  │    (চাহিদা অনুযায়ী ভাড়া নির্ধারণ)         │                 │
│  │  • Route Optimization                    │                 │
│  │    (সেরা রাস্তা খুঁজে বের করা)             │                 │
│  └──────────────────────────────────────────┘                 │
│                                                               │
│  🔧 Supporting Domain (সহায়ক — দরকার কিন্তু মূল না):          │
│  ┌──────────────────────────────────────────┐                 │
│  │  • Driver Verification                   │                 │
│  │    (ড্রাইভারের লাইসেন্স/NID যাচাই)        │                 │
│  │  • Trip History                          │                 │
│  │    (আগের ট্রিপের ইতিহাস)                 │                 │
│  │  • Rating & Review                       │                 │
│  │    (রেটিং ও রিভিউ সিস্টেম)               │                 │
│  └──────────────────────────────────────────┘                 │
│                                                               │
│  📦 Generic Domain (সাধারণ — কিনে নিন):                       │
│  ┌──────────────────────────────────────────┐                 │
│  │  • Authentication (Firebase Auth)        │                 │
│  │  • Email/SMS Sending (Twilio/SSL)        │                 │
│  │  • Payment Gateway (bKash/Nagad)         │                 │
│  │  • Push Notification (Firebase/OneSignal)│                 │
│  └──────────────────────────────────────────┘                 │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### তুলনামূলক টেবিল

```
┌──────────────┬───────────────────┬──────────────────┬──────────────────┐
│    বৈশিষ্ট্য  │  🏆 Core Domain   │ 🔧 Supporting    │ 📦 Generic       │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ গুরুত্ব      │ সর্বোচ্চ            │ মাঝারি            │ কম               │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ প্রতিযোগিতা  │ পার্থক্য তৈরি করে  │ করে না            │ করে না            │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ জটিলতা      │ উচ্চ               │ মাঝারি            │ কম               │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ বিনিয়োগ     │ সবচেয়ে বেশি       │ মাঝারি            │ সবচেয়ে কম        │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ তৈরি করবে কে│ সেরা ডেভেলপাররা   │ ইন-হাউজ টিম      │ তৃতীয় পক্ষ/SaaS │
├──────────────┼───────────────────┼──────────────────┼──────────────────┤
│ উদাহরণ      │ Ride Matching     │ Driver           │ Auth, Email,     │
│ (Pathao)    │ Dynamic Pricing   │ Verification     │ SMS Gateway      │
└──────────────┴───────────────────┴──────────────────┴──────────────────┘
```

### 🎯 কীভাবে চিনবেন কোন ডোমেইন কোন ধরনের?

```
প্রশ্ন করুন:
                                    ┌─────┐
  "এটা কি আমাদের ব্যবসাকে          │     │
   প্রতিযোগীদের থেকে আলাদা করে?"  │     │
                                    └──┬──┘
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                        হ্যাঁ                        না
                          │                         │
                          ▼                         ▼
                   🏆 Core Domain        "এটা কি আমাদের ব্যবসার
                                         জন্য বিশেষভাবে তৈরি
                                         করতে হবে?"
                                                │
                                   ┌────────────┴────────────┐
                                   │                         │
                                 হ্যাঁ                        না
                                   │                         │
                                   ▼                         ▼
                            🔧 Supporting              📦 Generic
                               Domain                    Domain
```

---

## 🎯 Real-world E-commerce Example

### Daraz-সদৃশ ই-কমার্স সিস্টেম

নিচে একটি পূর্ণাঙ্গ ই-কমার্স সিস্টেমের Strategic Design দেখানো হলো:

```
┌─────────────────────────────────────────────────────────────────────┐
│                Daraz-সদৃশ সিস্টেম — Strategic Design Map            │
│                                                                     │
│  🏆 CORE DOMAINS:                                                   │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐     │
│  │  Product        │  │  Recommendation  │  │  Pricing &     │     │
│  │  Discovery &    │  │  Engine          │  │  Promotions    │     │
│  │  Search         │  │  (ব্যক্তিগতকৃত   │  │  (ক্যাম্পেইন,  │     │
│  │  (খোঁজা ও       │  │   সুপারিশ)       │  │   ফ্ল্যাশ সেল) │     │
│  │   আবিষ্কার)     │  └──────────────────┘  └────────────────┘     │
│  └─────────────────┘                                                │
│                                                                     │
│  🔧 SUPPORTING DOMAINS:                                             │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  Order      │ │  Inventory   │ │  Shipping   │ │  Seller     │ │
│  │  Management │ │  Management  │ │  & Delivery │ │  Management │ │
│  └─────────────┘ └──────────────┘ └─────────────┘ └─────────────┘ │
│                                                                     │
│  📦 GENERIC DOMAINS:                                                │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  Auth &     │ │  Payment     │ │  Notification│ │  Reporting  │ │
│  │  Identity   │ │  (bKash,     │ │  (SMS/Email) │ │  & Analytics│ │
│  │             │ │   Nagad)     │ │              │ │             │ │
│  └─────────────┘ └──────────────┘ └─────────────┘ └─────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### PHP: Bounded Context আলাদাকরণ

```php
// === Order Bounded Context ===
namespace App\Order\Domain;

class Order
{
    private OrderId $id;
    private CustomerId $customerId;
    private array $lineItems = [];
    private OrderStatus $status;
    private Money $totalAmount;

    public static function place(
        CustomerId $customerId,
        array $lineItems
    ): self {
        $order = new self();
        $order->id = OrderId::generate();
        $order->customerId = $customerId;
        $order->lineItems = $lineItems;
        $order->status = OrderStatus::PLACED;
        $order->totalAmount = self::calculateTotal($lineItems);

        $order->recordEvent(new OrderPlaced(
            orderId: $order->id,
            customerId: $order->customerId,
            totalAmount: $order->totalAmount
        ));

        return $order;
    }

    public function confirm(): void
    {
        if ($this->status !== OrderStatus::PLACED) {
            throw new InvalidOrderStateException(
                "শুধুমাত্র PLACED অবস্থার অর্ডার confirm করা যায়"
            );
        }
        $this->status = OrderStatus::CONFIRMED;
        $this->recordEvent(new OrderConfirmed($this->id));
    }

    private static function calculateTotal(array $lineItems): Money
    {
        return array_reduce(
            $lineItems,
            fn(Money $total, LineItem $item) => $total->add($item->subtotal()),
            Money::zero('BDT')
        );
    }
}

// === Inventory Bounded Context ===
// Order Context-এর OrderPlaced ইভেন্ট শুনে স্টক রিজার্ভ করে
namespace App\Inventory\Application;

class ReserveStockOnOrderPlaced
{
    public function __construct(private StockRepository $stockRepo) {}

    public function handle(OrderPlacedEvent $event): void
    {
        foreach ($event->lineItems as $item) {
            $stock = $this->stockRepo->findByProductId($item['productId']);
            $stock->reserve($item['quantity']);
            $this->stockRepo->save($stock);
        }
    }
}
```

### JavaScript: Bounded Context আলাদাকরণ

```javascript
// === Order Bounded Context ===
// src/order/domain/Order.js
class Order {
  #id;
  #customerId;
  #lineItems;
  #status;
  #events = [];

  static place(customerId, lineItems) {
    const order = new Order();
    order.#id = OrderId.generate();
    order.#customerId = customerId;
    order.#lineItems = lineItems;
    order.#status = "PLACED";

    order.#events.push(
      new OrderPlaced({
        orderId: order.#id,
        customerId: order.#customerId,
        items: lineItems,
        totalAmount: order.totalAmount,
      })
    );

    return order;
  }

  get totalAmount() {
    return this.#lineItems.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );
  }

  confirm() {
    if (this.#status !== "PLACED") {
      throw new Error("শুধুমাত্র PLACED অবস্থার অর্ডার confirm করা যায়");
    }
    this.#status = "CONFIRMED";
    this.#events.push(new OrderConfirmed({ orderId: this.#id }));
  }

  get domainEvents() {
    return [...this.#events];
  }
}

// === Inventory Bounded Context ===
// src/inventory/application/ReserveStockOnOrderPlaced.js
class ReserveStockOnOrderPlaced {
  #stockRepo;

  constructor(stockRepo) {
    this.#stockRepo = stockRepo;
  }

  async handle(orderPlacedEvent) {
    for (const item of orderPlacedEvent.items) {
      const stock = await this.#stockRepo.findByProductId(item.productId);
      stock.reserve(item.quantity);
      await this.#stockRepo.save(stock);
    }
  }
}
```

### Context-এর মধ্যে যোগাযোগ

```
┌──────────────┐     Domain Event      ┌───────────────┐
│    Order     │ ── OrderPlaced ──────► │   Inventory   │
│   Context    │                        │   Context     │
│              │ ── OrderCancelled ───► │               │
└──────┬───────┘                        └───────────────┘
       │
       │  Domain Event
       │  OrderPlaced
       ▼
┌──────────────┐     API Call           ┌───────────────┐
│   Payment    │ ◄── processPayment ── │    Order      │
│   Context    │                        │   Context     │
│              │     ACL                │               │
│   (bKash/   │ ── translateResponse ─►│               │
│    Nagad)    │                        │               │
└──────────────┘                        └───────────────┘

💡 নিয়ম: Context-এর মধ্যে যোগাযোগ হয় Domain Events অথবা
   well-defined API দিয়ে — সরাসরি ডাটাবেজ শেয়ার করা যাবে না!
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Strategic DDD ব্যবহার করবেন

```
✅ বড় ও জটিল ডোমেইন
   → যেমন: Daraz, Pathao, bKash — অনেক ফিচার, অনেক টিম

✅ একাধিক টিম একই সিস্টেমে কাজ করে
   → প্রতিটি টিমের নিজস্ব Bounded Context থাকলে conflict কমে

✅ ডোমেইন এক্সপার্ট আছেন
   → Ubiquitous Language তৈরি করতে তাদের সহায়তা দরকার

✅ সিস্টেম দীর্ঘমেয়াদে চলবে
   → ৫-১০ বছর মেইনটেইন করতে হবে, ভালো structure দরকার

✅ Microservices আর্কিটেকচার
   → Bounded Context = Microservice boundary
```

### ❌ কখন ব্যবহার করবেন না

```
❌ ছোট CRUD অ্যাপ্লিকেশন
   → একটি ব্লগ বা টোডো অ্যাপে DDD overkill

❌ একজন ডেভেলপারের প্রজেক্ট
   → Context map, team boundaries — এগুলো একজনের জন্য অপ্রয়োজনীয়

❌ প্রোটোটাইপ বা MVP
   → দ্রুত বাজারে যেতে হলে আগে সিম্পল রাখুন, পরে refactor করুন

❌ সিম্পল বিজনেস লজিক
   → যদি পুরো সিস্টেম শুধু CRUD হয়, DDD value add করবে না

❌ টিম DDD সম্পর্কে জানে না
   → আগে শিখুন, তারপর প্রয়োগ করুন — না বুঝে করলে ক্ষতি
```

### 🎯 সিদ্ধান্ত নেওয়ার গাইড

```
আপনার প্রজেক্ট কি:

  ১. ৩+ টিমের বেশি?                    হ্যাঁ → Strategic DDD বিবেচনা করুন
  ২. ডোমেইন জটিল?                     হ্যাঁ → Strategic DDD দরকার
  ৩. ৩ বছরের বেশি চলবে?               হ্যাঁ → Strategic DDD দরকার
  ৪. বিভিন্ন অংশে ভিন্ন ভাষা ব্যবহৃত হয়?  হ্যাঁ → Bounded Context দরকার

  সব "না" → Simple architecture যথেষ্ট
  ১-২ টা "হ্যাঁ" → আংশিক DDD (শুধু Ubiquitous Language + Bounded Context)
  ৩+ "হ্যাঁ" → পূর্ণ Strategic DDD প্রয়োগ করুন
```

---

## 📊 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────────┐
│                Strategic Design — মূল ধারণাসমূহ               │
│                                                              │
│  📖 Ubiquitous Language                                      │
│     → টিমের সবাই একই শব্দ ব্যবহার করবে                      │
│     → কোড, মিটিং, ডকুমেন্ট — সব জায়গায় একই টার্ম            │
│                                                              │
│  📊 Bounded Contexts                                         │
│     → বড় সিস্টেমকে ছোট ছোট অংশে ভাগ করুন                   │
│     → প্রতিটি অংশের নিজস্ব মডেল, নিজস্ব ভাষা                 │
│                                                              │
│  🔗 Context Maps                                             │
│     → Context-এর মধ্যে সম্পর্ক নির্ধারণ করুন                  │
│     → ACL দিয়ে বাইরের সিস্টেম থেকে নিজেকে রক্ষা করুন        │
│                                                              │
│  🏠 Subdomains                                               │
│     → Core: সবচেয়ে বেশি বিনিয়োগ, সেরা ডেভেলপার              │
│     → Supporting: ইন-হাউজ তৈরি, কিন্তু Core-এর মতো গুরুত্ব নয়│
│     → Generic: কিনে নিন বা ওপেন সোর্স ব্যবহার করুন           │
│                                                              │
│  💡 মনে রাখুন: Strategic Design হলো টেকনিক্যাল সিদ্ধান্ত নয়,  │
│     এটি বিজনেস সিদ্ধান্ত। ডোমেইন বুঝুন, তারপর কোড লিখুন।    │
└──────────────────────────────────────────────────────────────┘
```

---

> **পরবর্তী পড়ুন:** Tactical Design — Entity, Value Object, Aggregate, Repository
