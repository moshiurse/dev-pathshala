# 📌 DDD with Microservices — ডোমেইন-ড্রিভেন ডিজাইন ও মাইক্রোসার্ভিসেস

> DDD এবং Microservices একসাথে কাজ করলে কীভাবে বড় সিস্টেমকে ছোট ছোট স্বাধীন সার্ভিসে ভাগ করা যায়, সেই সম্পূর্ণ গাইড।

---

## 📖 সূচিপত্র

1. [Bounded Context = Microservice Boundary](#-1-bounded-context--microservice-boundary)
2. [Anti-corruption Layer (ACL)](#-2-anti-corruption-layer-acl)
3. [Shared Kernel vs Separate Models](#-3-shared-kernel-vs-separate-models)
4. [Event-driven Communication](#-4-event-driven-communication)
5. [Practical Example: E-commerce Decomposition](#-5-practical-example-e-commerce-decomposition)
6. [কখন ব্যবহার করবেন / করবেন না](#-6-কখন-ব্যবহার-করবেন--করবেন-না)
7. [চেকলিস্ট](#-7-চেকলিস্ট)

---

## 📊 1. Bounded Context = Microservice Boundary

### মূল ধারণা

DDD-তে **Bounded Context** হলো একটি নির্দিষ্ট ডোমেইন মডেলের সীমানা। Microservices আর্কিটেকচারে প্রতিটি
Bounded Context স্বাভাবিকভাবেই একটি আলাদা Microservice হয়ে যায়। এটাই DDD আর Microservices-এর সবচেয়ে
শক্তিশালী সম্পর্ক — **প্রতিটি সার্ভিস একটি নির্দিষ্ট ব্যবসায়িক ক্ষমতার মালিক।**

ধরুন Daraz-এর মতো একটি ই-কমার্স সিস্টেম তৈরি করছেন। পুরো সিস্টেমকে একটি বিশাল Monolith হিসেবে না বানিয়ে,
DDD ব্যবহার করে আলাদা আলাদা Bounded Context চিহ্নিত করুন — প্রতিটিই একটি স্বতন্ত্র Microservice হবে।

### ASCII ডায়াগ্রাম: Bounded Context → Microservice

```
  ┌─────────────────────── DDD Bounded Contexts ───────────────────────┐
  │                                                                     │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
  │  │   Order       │  │   Product    │  │   Payment    │              │
  │  │   Context     │  │   Context    │  │   Context    │              │
  │  └──────────────┘  └──────────────┘  └──────────────┘              │
  │  ┌──────────────┐  ┌──────────────┐                                │
  │  │   Delivery    │  │   Identity   │                                │
  │  │   Context     │  │   Context    │                                │
  │  └──────────────┘  └──────────────┘                                │
  └─────────────────────────────────────────────────────────────────────┘

                          ║  DDD → Microservices  ║
                          ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼

  ┌──────────────────── Microservices Architecture ─────────────────────┐
  │                                                                     │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
  │  │ Order Service │  │Catalog Svc   │  │Payment Svc   │              │
  │  │  [Port 8001] │  │  [Port 8002] │  │  [Port 8003] │              │
  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │              │
  │  │  │ Own DB │  │  │  │ Own DB │  │  │  │ Own DB │  │              │
  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │              │
  │  └──────────────┘  └──────────────┘  └──────────────┘              │
  │  ┌──────────────┐  ┌──────────────┐                                │
  │  │Shipping Svc  │  │ User Service │                                │
  │  │  [Port 8004] │  │  [Port 8005] │                                │
  │  │  ┌────────┐  │  │  ┌────────┐  │                                │
  │  │  │ Own DB │  │  │  │ Own DB │  │                                │
  │  │  └────────┘  │  │  └────────┘  │                                │
  │  └──────────────┘  └──────────────┘                                │
  └─────────────────────────────────────────────────────────────────────┘
```

### সার্ভিস বাউন্ডারি কীভাবে চিহ্নিত করবেন?

ডোমেইন এক্সপার্টদের সাথে কথা বলুন। প্রতিটি দলের নিজস্ব **Ubiquitous Language** আলাদা হলে সেটাই
একটি আলাদা Bounded Context এবং তাই একটি আলাদা Microservice।

**Daraz-এর উদাহরণ:**

| Bounded Context | Microservice     | মূল Entity/Concept               |
|-----------------|------------------|-----------------------------------|
| Order           | Order Service    | Order, OrderItem, OrderStatus     |
| Product         | Catalog Service  | Product, Category, Brand, Review  |
| Payment         | Payment Service  | Payment, Refund, Transaction      |
| Delivery        | Shipping Service | Shipment, Tracking, Courier       |
| Identity        | User Service     | User, Address, Authentication     |

### ❌ Bad: টেকনিক্যাল লেয়ার দিয়ে ভাগ করা

```
  ❌ ভুল পদ্ধতি — Technical Layer দিয়ে ভাগ করা:

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  API Service │   │ Business     │   │  Database    │
  │  (সব API     │──▶│ Logic Svc    │──▶│  Service     │
  │   এখানে)    │   │ (সব লজিক    │   │ (সব DB      │
  │              │   │  এখানে)     │   │  এখানে)     │
  └──────────────┘   └──────────────┘   └──────────────┘

  সমস্যা:
  - সব কিছু tightly coupled
  - একটা পরিবর্তন সব সার্ভিসে ছড়িয়ে পড়ে
  - আলাদা টিম দিয়ে scale করা যায় না
```

### ✅ Good: Bounded Context / ব্যবসায়িক ক্ষমতা দিয়ে ভাগ করা

```
  ✅ সঠিক পদ্ধতি — Business Capability দিয়ে ভাগ:

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │Order Service │   │Catalog Svc   │   │Payment Svc   │
  │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
  │ │ API      │ │   │ │ API      │ │   │ │ API      │ │
  │ │ Logic    │ │   │ │ Logic    │ │   │ │ Logic    │ │
  │ │ Database │ │   │ │ Database │ │   │ │ Database │ │
  │ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │
  └──────────────┘   └──────────────┘   └──────────────┘

  সুবিধা:
  - প্রতিটি সার্ভিস স্বাধীন
  - আলাদা টিম, আলাদা deploy
  - একটি সার্ভিসের পরিবর্তন অন্যটিকে প্রভাবিত করে না
```

---

## 🔗 2. Anti-corruption Layer (ACL)

### ACL কী এবং কেন দরকার?

আপনার সুন্দর DDD মডেলকে বাইরের বিশৃঙ্খল সিস্টেম থেকে রক্ষা করতে ACL দরকার। এটি একটি **অনুবাদক স্তর**
যা বাইরের ডেটা ফরম্যাটকে আপনার অভ্যন্তরীণ মডেলে রূপান্তর করে।

বাংলাদেশের উদাহরণ: আপনার Payment Service-এ bKash, Nagad, এবং Stripe — তিনটি আলাদা পেমেন্ট গেটওয়ে
ইন্টিগ্রেট করতে হবে। প্রতিটির API আলাদা। ACL ছাড়া আপনার ডোমেইন কোড বাইরের API-র গঠন দ্বারা দূষিত হবে।

### ASCII ডায়াগ্রাম: ACL

```
  বাইরের সিস্টেম (External)              আপনার সিস্টেম (Internal)
  ──────────────────────────              ─────────────────────────

  ┌──────────────────┐          ┌─────────────────────────────────────┐
  │  bKash API       │          │         Payment Service             │
  │  {                │          │                                     │
  │   "trxID": "...",│    ┌─────┤  ┌───────────────────────────────┐  │
  │   "amount": 500, │───▶│ ACL │  │     Domain Model              │  │
  │   "currency":    │    │     │──▶  PaymentTransaction {         │  │
  │    "BDT"         │    │     │  │    transactionId: string      │  │
  │  }               │    │     │  │    amount: Money              │  │
  └──────────────────┘    │     │  │    status: PaymentStatus      │  │
                          │     │  │  }                            │  │
  ┌──────────────────┐    │     │  └───────────────────────────────┘  │
  │  Nagad API       │    │     │                                     │
  │  {                │───▶│     │                                     │
  │   "txn_id": "..",│    │     │                                     │
  │   "amt": "500",  │    └─────┤                                     │
  │  }               │          │                                     │
  └──────────────────┘          └─────────────────────────────────────┘

  ACL = Anti-corruption Layer
  বাইরের বিভিন্ন ফরম্যাটকে অভ্যন্তরীণ একক মডেলে রূপান্তর করে
```

### PHP Implementation: bKash/Nagad ACL

```php
<?php

// --- আপনার অভ্যন্তরীণ ডোমেইন মডেল ---
class Money
{
    public function __construct(
        public readonly float $amount,
        public readonly string $currency
    ) {}
}

class PaymentTransaction
{
    public function __construct(
        public readonly string $transactionId,
        public readonly Money $amount,
        public readonly string $status,
        public readonly string $gateway
    ) {}
}

// --- ACL Interface ---
interface PaymentGatewayACL
{
    public function translateResponse(array $externalResponse): PaymentTransaction;
    public function translateRequest(Money $amount, string $orderId): array;
}

// --- bKash ACL ---
class BkashACL implements PaymentGatewayACL
{
    public function translateResponse(array $externalResponse): PaymentTransaction
    {
        // bKash API response: {"trxID": "...", "amount": "500.00", "currency": "BDT", "transactionStatus": "Completed"}
        return new PaymentTransaction(
            transactionId: $externalResponse['trxID'],
            amount: new Money(
                amount: (float) $externalResponse['amount'],
                currency: $externalResponse['currency'] ?? 'BDT'
            ),
            status: $this->mapStatus($externalResponse['transactionStatus']),
            gateway: 'bkash'
        );
    }

    public function translateRequest(Money $amount, string $orderId): array
    {
        return [
            'amount'    => number_format($amount->amount, 2, '.', ''),
            'currency'  => $amount->currency,
            'merchantInvoiceNumber' => $orderId,
            'intent'    => 'sale',
        ];
    }

    private function mapStatus(string $bkashStatus): string
    {
        return match ($bkashStatus) {
            'Completed'  => 'completed',
            'Initiated'  => 'pending',
            'Authorized' => 'authorized',
            default      => 'failed',
        };
    }
}

// --- Nagad ACL ---
class NagadACL implements PaymentGatewayACL
{
    public function translateResponse(array $externalResponse): PaymentTransaction
    {
        // Nagad API response: {"issuerPaymentRefNo": "...", "amount": "500", "statusCode": "000", "orderID": "..."}
        return new PaymentTransaction(
            transactionId: $externalResponse['issuerPaymentRefNo'],
            amount: new Money(
                amount: (float) $externalResponse['amount'],
                currency: 'BDT'
            ),
            status: $this->mapStatus($externalResponse['statusCode']),
            gateway: 'nagad'
        );
    }

    public function translateRequest(Money $amount, string $orderId): array
    {
        return [
            'merchantId' => config('nagad.merchant_id'),
            'orderId'    => $orderId,
            'amount'     => (string) $amount->amount,
        ];
    }

    private function mapStatus(string $nagadCode): string
    {
        return match ($nagadCode) {
            '000' => 'completed',
            '001' => 'pending',
            default => 'failed',
        };
    }
}

// --- ব্যবহার ---
// আপনার ডোমেইন সার্ভিস বাইরের API সম্পর্কে কিছুই জানে না
class PaymentService
{
    public function processPayment(PaymentGatewayACL $acl, array $gatewayResponse): PaymentTransaction
    {
        $transaction = $acl->translateResponse($gatewayResponse);
        // এখানে শুধু আপনার নিজের PaymentTransaction মডেল দিয়ে কাজ করুন
        return $transaction;
    }
}
```

### JavaScript Implementation: ACL

```javascript
// --- অভ্যন্তরীণ ডোমেইন মডেল ---
class PaymentTransaction {
  constructor({ transactionId, amount, currency, status, gateway }) {
    this.transactionId = transactionId;
    this.amount = amount;
    this.currency = currency;
    this.status = status;
    this.gateway = gateway;
  }
}

// --- bKash ACL ---
class BkashACL {
  translateResponse(bkashResponse) {
    return new PaymentTransaction({
      transactionId: bkashResponse.trxID,
      amount: parseFloat(bkashResponse.amount),
      currency: bkashResponse.currency || "BDT",
      status: this.#mapStatus(bkashResponse.transactionStatus),
      gateway: "bkash",
    });
  }

  #mapStatus(bkashStatus) {
    const statusMap = {
      Completed: "completed",
      Initiated: "pending",
      Authorized: "authorized",
    };
    return statusMap[bkashStatus] || "failed";
  }
}

// --- Nagad ACL ---
class NagadACL {
  translateResponse(nagadResponse) {
    return new PaymentTransaction({
      transactionId: nagadResponse.issuerPaymentRefNo,
      amount: parseFloat(nagadResponse.amount),
      currency: "BDT",
      status: nagadResponse.statusCode === "000" ? "completed" : "failed",
      gateway: "nagad",
    });
  }
}
```

### Legacy System Integration: Inventory Example

```
  আপনার নতুন Order Service → পুরোনো Inventory System থেকে ডেটা দরকার:

  ┌─────────────────┐      ┌──────────┐      ┌──────────────────────┐
  │  Order Service   │      │   ACL    │      │  Legacy Inventory    │
  │  (নতুন DDD)     │─────▶│ (অনুবাদক)│─────▶│  (পুরোনো সিস্টেম)   │
  │                  │      │          │      │  - XML responses     │
  │  Product {       │      │ XML →    │      │  - SOAP API          │
  │   id, name,     │◀─────│ Object   │◀─────│  - অদ্ভুত field     │
  │   stock          │      │ mapping  │      │    নাম               │
  │  }               │      │          │      │                      │
  └─────────────────┘      └──────────┘      └──────────────────────┘

  ACL ছাড়া: পুরোনো সিস্টেমের জটিলতা নতুন কোডে ছড়িয়ে পড়ত
  ACL দিয়ে: নতুন কোড পরিষ্কার থাকে, শুধু ACL-এ translation logic থাকে
```

---

## 🏠 3. Shared Kernel vs Separate Models

### Shared Kernel কী?

Shared Kernel হলো দুই বা ততোধিক Bounded Context-এর মধ্যে ভাগ করা কোড বা মডেল। Microservices-এ এটি
অত্যন্ত সাবধানে ব্যবহার করতে হবে, কারণ এটি coupling তৈরি করে।

### কখন Share করবেন vs আলাদা রাখবেন

```
  Shared Kernel:                          Separate Models:
  ─────────────                           ─────────────────

  ┌───────────┐  shared  ┌───────────┐    ┌───────────┐       ┌───────────┐
  │ Service A │◄────────▶│ Service B │    │ Service A │       │ Service B │
  │           │  library │           │    │           │       │           │
  │  Money    │  ┌─────┐ │  Money    │    │  Money    │       │  Price    │
  │  Address  │──│Shared│─│  Address  │    │  Address  │       │  Location │
  │           │  │Kernel│ │           │    │ (নিজস্ব) │       │ (নিজস্ব) │
  └───────────┘  └─────┘ └───────────┘    └───────────┘       └───────────┘

  ⚠️ ঝুঁকি: একটি পরিবর্তনে       ✅ নিরাপদ: প্রতিটি সার্ভিস
     উভয় সার্ভিস প্রভাবিত হয়         স্বাধীনভাবে পরিবর্তন করতে পারে
```

### ❌ Bad: Microservices-এর মধ্যে Database শেয়ার করা

```
  ❌ ভুল — Shared Database:

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Order Service│     │Catalog Svc   │     │Payment Svc   │
  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
         │                    │                     │
         ▼                    ▼                     ▼
  ┌──────────────────────────────────────────────────────────┐
  │                    একটি বিশাল Database                   │
  │  orders | products | payments | users | shipping         │
  └──────────────────────────────────────────────────────────┘

  সমস্যা:
  - একটি সার্ভিসের schema পরিবর্তন অন্যগুলোকে ভেঙে দেয়
  - স্বাধীনভাবে deploy করা যায় না
  - সবকিছু tightly coupled
  - একটি table lock হলে সব সার্ভিস ক্ষতিগ্রস্ত
```

### ✅ Good: প্রতিটি সার্ভিস নিজের ডেটার মালিক

```
  ✅ সঠিক — Database per Service:

  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Order Service│     │Catalog Svc   │     │Payment Svc   │
  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
         │                    │                     │
         ▼                    ▼                     ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  Order DB    │     │  Catalog DB  │     │  Payment DB  │
  │  (MySQL)     │     │ (PostgreSQL) │     │  (MongoDB)   │
  └──────────────┘     └──────────────┘     └──────────────┘

       Events দিয়ে communicate করে, DB share করে না!
                    ┌────────────┐
            ───────▶│ Event Bus  │◀───────
                    └────────────┘
```

### PHP Example: Shared Value Objects (সাবধানে ব্যবহার)

```php
<?php
// shared-kernel/src/ValueObjects/Money.php
// এই প্যাকেজটি Composer দিয়ে আলাদা লাইব্রেরি হিসেবে publish করুন

namespace SharedKernel\ValueObjects;

class Money
{
    public function __construct(
        private readonly float $amount,
        private readonly string $currency = 'BDT'
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }
    }

    public function getAmount(): float
    {
        return $this->amount;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function add(Money $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    private function assertSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Currency mismatch');
        }
    }
}
```

### JavaScript Example: Shared Value Objects

```javascript
// @shared-kernel/money.js
// npm package হিসেবে publish করুন

export class Money {
  #amount;
  #currency;

  constructor(amount, currency = "BDT") {
    if (amount < 0) throw new Error("Amount cannot be negative");
    this.#amount = amount;
    this.#currency = currency;
  }

  get amount() {
    return this.#amount;
  }
  get currency() {
    return this.#currency;
  }

  add(other) {
    if (this.#currency !== other.currency) throw new Error("Currency mismatch");
    return new Money(this.#amount + other.amount, this.#currency);
  }

  equals(other) {
    return this.#amount === other.amount && this.#currency === other.currency;
  }
}
```

### 🎯 Decision Matrix: Share vs Separate

| বিষয়                    | Shared Kernel          | Separate Models        |
|--------------------------|------------------------|------------------------|
| Value Objects (Money)    | ✅ Share করা যায়       | ⚠️ দরকার হলে          |
| Entity                   | ❌ Share করবেন না      | ✅ আলাদা রাখুন          |
| Database                 | ❌ কখনোই না            | ✅ সবসময়               |
| Business Logic           | ❌ Share করবেন না      | ✅ আলাদা রাখুন          |
| DTO / API Contract       | ⚠️ সাবধানে            | ✅ ভালো                 |
| Enum / Constants         | ✅ Share করা যায়       | ⚠️ দরকার হলে          |

---

## 📊 4. Event-driven Communication

### কেন Synchronous REST Call সমস্যাজনক?

```
  ❌ Synchronous (REST) — চেইন ডিপেন্ডেন্সি:

  User ──▶ Order Service ──REST──▶ Payment Service ──REST──▶ Inventory
                                                        │
                                                        ▼
                                              Shipping Service

  সমস্যা:
  1. Payment Service ডাউন হলে Order Service-ও ডাউন (Cascading Failure)
  2. Response time = সব সার্ভিসের total time
  3. Tight coupling — একটি সার্ভিসের API পরিবর্তন সব caller ভেঙে দেয়

  বাংলা উপমা: ফোনে কথা বলা — দুজনকেই একই সময়ে available থাকতে হবে!
```

```
  ✅ Asynchronous (Events) — ঢিলে coupling:

  User ──▶ Order Service ──Event──▶ ┌─────────────┐ ──▶ Payment Svc
                                    │ Message      │ ──▶ Inventory Svc
                                    │ Broker       │ ──▶ Shipping Svc
                                    │ (RabbitMQ /  │ ──▶ Notification Svc
                                    │  Kafka)      │
                                    └─────────────┘

  সুবিধা:
  1. Payment Service ডাউন হলেও Order Service কাজ করবে
  2. Event queue-তে জমা থাকবে, পরে process হবে
  3. নতুন সার্ভিস যোগ করা সহজ — শুধু event subscribe করুন

  বাংলা উপমা: চিঠি পাঠানো — প্রাপককে সেই মুহূর্তে available থাকতে হয় না!
```

### ASCII ডায়াগ্রাম: Event-driven Architecture

```
  ┌────────────────────────────────────────────────────────────────┐
  │                        Event Bus (RabbitMQ / Kafka)            │
  │                                                                │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
  │  │OrderCreated  │  │PaymentDone   │  │InventoryReserved     │ │
  │  │OrderCancelled│  │PaymentFailed │  │InventoryReleased     │ │
  │  └──────────────┘  └──────────────┘  └──────────────────────┘ │
  └───────▲────────────────▲──────────────────▲───────────────────┘
          │ publish         │ publish          │ publish
          │                 │                  │
  ┌───────┴──────┐  ┌──────┴───────┐  ┌──────┴───────┐
  │Order Service │  │Payment Svc   │  │Inventory Svc │
  │              │  │              │  │              │
  │ subscribes:  │  │ subscribes:  │  │ subscribes:  │
  │ PaymentDone  │  │ OrderCreated │  │ OrderCreated │
  │ PaymentFailed│  │              │  │ PaymentFailed│
  └──────────────┘  └──────────────┘  └──────────────┘
```

### PHP Example: Event Publishing (RabbitMQ)

```php
<?php

// --- Domain Event ---
class OrderCreatedEvent
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $customerId,
        public readonly float $totalAmount,
        public readonly string $currency,
        public readonly array $items,
        public readonly \DateTimeImmutable $occurredAt
    ) {}

    public function toArray(): array
    {
        return [
            'event_type'  => 'order.created',
            'order_id'    => $this->orderId,
            'customer_id' => $this->customerId,
            'total'       => $this->totalAmount,
            'currency'    => $this->currency,
            'items'       => $this->items,
            'occurred_at' => $this->occurredAt->format('c'),
        ];
    }
}

// --- Event Publisher ---
class RabbitMQEventPublisher
{
    private \AMQPChannel $channel;

    public function __construct(\AMQPConnection $connection)
    {
        $this->channel = $connection->channel();
        $this->channel->exchange_declare('domain_events', 'topic', false, true, false);
    }

    public function publish(OrderCreatedEvent $event): void
    {
        $message = new \AMQPMessage(
            json_encode($event->toArray()),
            [
                'content_type'  => 'application/json',
                'delivery_mode' => 2, // persistent
                'message_id'    => uniqid('evt_', true),
                'timestamp'     => time(),
            ]
        );

        $this->channel->basic_publish(
            $message,
            'domain_events',
            'order.created'  // routing key
        );
    }
}

// --- Order Service-এ ব্যবহার ---
class PlaceOrderHandler
{
    public function __construct(
        private OrderRepository $orderRepo,
        private RabbitMQEventPublisher $publisher
    ) {}

    public function handle(PlaceOrderCommand $command): void
    {
        $order = Order::create(
            customerId: $command->customerId,
            items: $command->items
        );

        $this->orderRepo->save($order);

        // ডোমেইন ইভেন্ট publish করুন
        $this->publisher->publish(new OrderCreatedEvent(
            orderId: $order->getId(),
            customerId: $command->customerId,
            totalAmount: $order->getTotal()->getAmount(),
            currency: 'BDT',
            items: $command->items,
            occurredAt: new \DateTimeImmutable()
        ));
    }
}
```

### JavaScript Example: Event Consuming (RabbitMQ)

```javascript
// payment-service/src/consumers/orderCreatedConsumer.js

const amqp = require("amqplib");

class OrderCreatedConsumer {
  async start() {
    const connection = await amqp.connect("amqp://localhost");
    const channel = await connection.createChannel();

    await channel.assertExchange("domain_events", "topic", { durable: true });

    const q = await channel.assertQueue("payment_service.order_created", {
      durable: true,
    });

    await channel.bindQueue(q.queue, "domain_events", "order.created");

    console.log("💰 Payment Service: Waiting for order events...");

    channel.consume(q.queue, async (msg) => {
      try {
        const event = JSON.parse(msg.content.toString());
        console.log(`📦 Order received: ${event.order_id}`);

        // পেমেন্ট প্রসেস করুন
        await this.processPayment(event);

        channel.ack(msg); // সফল হলে acknowledge করুন
      } catch (error) {
        console.error("Payment processing failed:", error);
        channel.nack(msg, false, true); // retry এর জন্য requeue
      }
    });
  }

  async processPayment(orderEvent) {
    const payment = {
      orderId: orderEvent.order_id,
      amount: orderEvent.total,
      currency: orderEvent.currency,
      status: "processing",
    };

    // পেমেন্ট লজিক...
    // সফল হলে PaymentCompleted event publish করুন
  }
}

new OrderCreatedConsumer().start();
```

### Eventual Consistency: বাংলা উপমা

```
  Eventual Consistency = শেষ পর্যন্ত সবকিছু সামঞ্জস্যপূর্ণ হবে

  বাস্তব উদাহরণ — bKash টাকা পাঠানো:
  ─────────────────────────────────────────

  সময় t=0:  আপনি ৫০০ টাকা পাঠালেন
             আপনার ব্যালেন্স:  ৫০০ কমে গেছে  ✅
             প্রাপকের ব্যালেন্স: এখনও বাড়েনি  ⏳

  সময় t=1:  সিস্টেম প্রসেস করছে...
             অসামঞ্জস্যপূর্ণ অবস্থা (inconsistent)

  সময় t=2:  প্রাপকের ব্যালেন্স: ৫০০ বেড়ে গেছে  ✅
             এখন সবকিছু সামঞ্জস্যপূর্ণ (consistent)

  মাইক্রোসার্ভিসেও একই: Order তৈরি হলো কিন্তু Payment এখনও
  প্রসেস হয়নি — কিছুক্ষণ পর সবকিছু ঠিক হয়ে যাবে!
```

### Saga Pattern: Distributed Transaction

Microservices-এ একাধিক সার্ভিস জুড়ে transaction করতে Saga pattern ব্যবহার করুন। এটি step-by-step
এগিয়ে যায়, কোনো step fail করলে আগের সব step compensate (undo) করে।

```
  Saga: Order Placement Flow
  ═══════════════════════════

  সফল পথ (Happy Path):
  ──────────────────────

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Order   │───▶│ Payment  │───▶│Inventory │───▶│ Shipping │
  │ Created  │    │ Charged  │    │ Reserved │    │ Scheduled│
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
      Step 1          Step 2          Step 3          Step 4


  ব্যর্থ পথ (Failure + Compensation):
  ─────────────────────────────────────

  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Order   │───▶│ Payment  │───▶│Inventory │ ✖ FAIL!
  │ Created  │    │ Charged  │    │ Reserve  │   (স্টক নেই)
  └──────────┘    └──────────┘    └──────────┘
                                       │
                    Compensate!        │
                  ◀────────────────────┘
                       │
  ┌──────────┐    ┌──────────┐
  │  Order   │◀───│ Payment  │
  │ Cancelled│    │ Refunded │
  └──────────┘    └──────────┘
      Undo 1         Undo 2

  প্রতিটি step-এর একটি compensating action আছে:
  ┌─────────────────┬─────────────────────┐
  │ Action          │ Compensation        │
  ├─────────────────┼─────────────────────┤
  │ Create Order    │ Cancel Order        │
  │ Charge Payment  │ Refund Payment      │
  │ Reserve Stock   │ Release Stock       │
  │ Schedule Ship   │ Cancel Shipment     │
  └─────────────────┴─────────────────────┘
```

---

## 🎯 5. Practical Example: E-commerce Decomposition

### সম্পূর্ণ সিস্টেম আর্কিটেকচার: Daraz-এর মতো প্ল্যাটফর্ম

```
                              ┌─────────────┐
                              │   Client    │
                              │ (Browser /  │
                              │  Mobile)    │
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │ API Gateway │
                              │ (Kong/Nginx)│
                              │ - Auth      │
                              │ - Rate Limit│
                              │ - Routing   │
                              └──────┬──────┘
                                     │
          ┌──────────┬───────────┬───┴────┬──────────┬──────────┐
          │          │           │        │          │          │
   ┌──────▼────┐ ┌──▼──────┐ ┌─▼──────┐ │   ┌──────▼────┐ ┌──▼───────┐
   │  Order    │ │Catalog  │ │Payment │ │   │ Shipping  │ │  User    │
   │  Service  │ │Service  │ │Service │ │   │  Service  │ │ Service  │
   │           │ │         │ │        │ │   │           │ │          │
   │ ┌───────┐ │ │┌───────┐│ │┌──────┐│ │   │ ┌───────┐ │ │┌───────┐│
   │ │Domain │ │ ││Domain ││ ││Domain││ │   │ │Domain │ │ ││Domain ││
   │ │ Layer │ │ ││ Layer ││ ││Layer ││ │   │ │ Layer │ │ ││ Layer ││
   │ └───────┘ │ │└───────┘│ │└──────┘│ │   │ └───────┘ │ │└───────┘│
   │ ┌───────┐ │ │┌───────┐│ │┌──────┐│ │   │ ┌───────┐ │ │┌───────┐│
   │ │App    │ │ ││App    ││ ││App   ││ │   │ │App    │ │ ││App    ││
   │ │ Layer │ │ ││ Layer ││ ││Layer ││ │   │ │ Layer │ │ ││ Layer ││
   │ └───────┘ │ │└───────┘│ │└──────┘│ │   │ └───────┘ │ │└───────┘│
   │ ┌───────┐ │ │┌───────┐│ │┌──────┐│ │   │ ┌───────┐ │ │┌───────┐│
   │ │Infra  │ │ ││Infra  ││ ││Infra ││ │   │ │Infra  │ │ ││Infra  ││
   │ │ Layer │ │ ││ Layer ││ ││Layer ││ │   │ │ Layer │ │ ││ Layer ││
   │ └───────┘ │ │└───────┘│ │└──────┘│ │   │ └───────┘ │ │└───────┘│
   └─────┬─────┘ └────┬────┘ └───┬────┘ │   └─────┬─────┘ └────┬────┘
         │            │          │       │         │            │
         ▼            ▼          ▼       │         ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌────────┐ │   ┌──────────┐ ┌──────────┐
   │Order DB  │ │Catalog DB│ │Pay DB  │ │   │Ship DB   │ │ User DB  │
   │(MySQL)   │ │(Elastic +│ │(Postgres│ │   │(MySQL)   │ │(Postgres)│
   │          │ │ MySQL)   │ │       )│ │   │          │ │          │
   └──────────┘ └──────────┘ └────────┘ │   └──────────┘ └──────────┘
                                        │
                              ┌─────────▼────────┐
                              │ Notification Svc  │
                              │ (SMS/Email/Push)  │
                              │ ┌──────────────┐  │
                              │ │ Grameenphone │  │
                              │ │ SMS Gateway  │  │
                              │ └──────────────┘  │
                              └──────────────────┘

  ═══════════════════════════════════════════════════════════
          সব সার্ভিস Event Bus দিয়ে connected:
  ═══════════════════════════════════════════════════════════
  ┌──────────────────────────────────────────────────────────┐
  │              Event Bus (RabbitMQ / Kafka)                │
  │                                                          │
  │  OrderCreated → PaymentService, InventoryService         │
  │  PaymentCompleted → OrderService, ShippingService        │
  │  PaymentFailed → OrderService (cancel)                   │
  │  ShipmentDispatched → NotificationService                │
  │  ShipmentDelivered → OrderService (complete)             │
  └──────────────────────────────────────────────────────────┘
```

### Order Flow: Sequence Diagram (ASCII)

```
  Customer        API GW      Order Svc     Payment Svc   Inventory    Shipping
     │               │            │              │            │            │
     │  Place Order   │            │              │            │            │
     │──────────────▶│            │              │            │            │
     │               │ Route      │              │            │            │
     │               │───────────▶│              │            │            │
     │               │            │              │            │            │
     │               │            │ OrderCreated │            │            │
     │               │            │ (event)──────┼───────────▶│            │
     │               │            │              │            │            │
     │               │            │              │ Reserve    │            │
     │               │            │              │ Stock      │            │
     │               │            │              │◀───────────│            │
     │               │            │              │            │            │
     │               │            │ InventoryReserved         │            │
     │               │            │◀─────────────┼────────────│            │
     │               │            │              │            │            │
     │               │            │ RequestPayment            │            │
     │               │            │─────────────▶│            │            │
     │               │            │              │            │            │
     │               │            │              │ Process    │            │
     │               │            │              │ bKash/Nagad│            │
     │               │            │              │────────┐   │            │
     │               │            │              │        │   │            │
     │               │            │              │◀───────┘   │            │
     │               │            │              │            │            │
     │               │            │ PaymentCompleted          │            │
     │               │            │◀─────────────│            │            │
     │               │            │              │            │            │
     │               │            │ ScheduleShipment          │            │
     │               │            │──────────────┼────────────┼───────────▶│
     │               │            │              │            │            │
     │  Order         │            │              │            │  Dispatch  │
     │  Confirmed    │            │              │            │  Courier   │
     │◀──────────────│◀───────────│              │            │            │
     │               │            │              │            │            │
```

### PHP: Order Service Key Code

```php
<?php
// order-service/src/Domain/Order.php

namespace OrderService\Domain;

class OrderStatus
{
    const PENDING   = 'pending';
    const CONFIRMED = 'confirmed';
    const PAID      = 'paid';
    const SHIPPED   = 'shipped';
    const DELIVERED = 'delivered';
    const CANCELLED = 'cancelled';
}

class OrderItem
{
    public function __construct(
        public readonly string $productId,
        public readonly string $productName,
        public readonly int $quantity,
        public readonly float $unitPrice
    ) {}

    public function subtotal(): float
    {
        return $this->quantity * $this->unitPrice;
    }
}

class Order
{
    private string $id;
    private string $customerId;
    private array $items = [];
    private string $status;
    private float $totalAmount;
    private array $domainEvents = [];

    public static function create(string $customerId, array $items): self
    {
        $order = new self();
        $order->id = uniqid('ord_');
        $order->customerId = $customerId;
        $order->items = $items;
        $order->status = OrderStatus::PENDING;
        $order->totalAmount = array_sum(array_map(fn($i) => $i->subtotal(), $items));

        $order->domainEvents[] = new OrderCreatedEvent(
            orderId: $order->id,
            customerId: $customerId,
            totalAmount: $order->totalAmount,
            items: $items
        );

        return $order;
    }

    public function markAsPaid(string $transactionId): void
    {
        if ($this->status !== OrderStatus::PENDING) {
            throw new \DomainException('Only pending orders can be marked as paid');
        }
        $this->status = OrderStatus::PAID;
        $this->domainEvents[] = new OrderPaidEvent($this->id, $transactionId);
    }

    public function cancel(string $reason): void
    {
        if (in_array($this->status, [OrderStatus::SHIPPED, OrderStatus::DELIVERED])) {
            throw new \DomainException('Cannot cancel shipped/delivered orders');
        }
        $this->status = OrderStatus::CANCELLED;
        $this->domainEvents[] = new OrderCancelledEvent($this->id, $reason);
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    public function getId(): string { return $this->id; }
    public function getTotal(): float { return $this->totalAmount; }
    public function getStatus(): string { return $this->status; }
}
```

### JS: Catalog Service Key Code

```javascript
// catalog-service/src/domain/Product.js

class Product {
  #id;
  #name;
  #description;
  #price;
  #stock;
  #categoryId;
  #sellerId;

  constructor({ id, name, description, price, stock, categoryId, sellerId }) {
    this.#id = id;
    this.#name = name;
    this.#description = description;
    this.#price = price;
    this.#stock = stock;
    this.#categoryId = categoryId;
    this.#sellerId = sellerId;
  }

  reserveStock(quantity) {
    if (quantity > this.#stock) {
      throw new Error(
        `Insufficient stock: requested ${quantity}, available ${this.#stock}`
      );
    }
    this.#stock -= quantity;
    return {
      event: "inventory.reserved",
      productId: this.#id,
      quantity,
      remainingStock: this.#stock,
    };
  }

  releaseStock(quantity) {
    this.#stock += quantity;
    return {
      event: "inventory.released",
      productId: this.#id,
      quantity,
      remainingStock: this.#stock,
    };
  }

  get id() {
    return this.#id;
  }
  get name() {
    return this.#name;
  }
  get price() {
    return this.#price;
  }
  get stock() {
    return this.#stock;
  }
}

// catalog-service/src/consumers/orderEventConsumer.js
class OrderEventConsumer {
  constructor(productRepository, eventPublisher) {
    this.productRepo = productRepository;
    this.eventPublisher = eventPublisher;
  }

  async handleOrderCreated(event) {
    try {
      for (const item of event.items) {
        const product = await this.productRepo.findById(item.productId);
        const result = product.reserveStock(item.quantity);
        await this.productRepo.save(product);
        await this.eventPublisher.publish(result);
      }
    } catch (error) {
      // স্টক না থাকলে InventoryReservationFailed event publish করুন
      await this.eventPublisher.publish({
        event: "inventory.reservation_failed",
        orderId: event.order_id,
        reason: error.message,
      });
    }
  }

  async handlePaymentFailed(event) {
    // পেমেন্ট ফেইল হলে reserved stock ছেড়ে দিন
    for (const item of event.items) {
      const product = await this.productRepo.findById(item.productId);
      product.releaseStock(item.quantity);
      await this.productRepo.save(product);
    }
  }
}

module.exports = { Product, OrderEventConsumer };
```

### Database per Service Pattern

```
  প্রতিটি সার্ভিসের নিজস্ব ডেটা স্টোর — প্রয়োজন অনুযায়ী আলাদা DB টেকনোলজি:

  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Order Service          Catalog Service       Payment Service    │
  │  ┌──────────────┐       ┌──────────────┐      ┌──────────────┐  │
  │  │   MySQL      │       │ Elasticsearch│      │  PostgreSQL  │  │
  │  │              │       │ (Search)     │      │              │  │
  │  │ orders       │       │ products     │      │ transactions │  │
  │  │ order_items  │       │              │      │ refunds      │  │
  │  │ order_events │       │   + MySQL    │      │ ledger       │  │
  │  └──────────────┘       │ (Master data)│      └──────────────┘  │
  │                         └──────────────┘                         │
  │  Shipping Service       User Service          Notification Svc  │
  │  ┌──────────────┐       ┌──────────────┐      ┌──────────────┐  │
  │  │   MySQL      │       │  PostgreSQL  │      │   Redis      │  │
  │  │              │       │              │      │ (Queue +     │  │
  │  │ shipments    │       │ users        │      │  Templates)  │  │
  │  │ tracking     │       │ addresses    │      │              │  │
  │  │ couriers     │       │ auth_tokens  │      │   + MongoDB  │  │
  │  └──────────────┘       └──────────────┘      │ (Log/History)│  │
  │                                               └──────────────┘  │
  │                                                                  │
  │  প্রতিটি সার্ভিস তার ডেটার একমাত্র মালিক!                      │
  │  অন্য সার্ভিস সরাসরি DB access করতে পারবে না।                   │
  │  ডেটা দরকার হলে → API call অথবা Event subscribe করুন।          │
  └──────────────────────────────────────────────────────────────────┘
```

---

## ✅❌ 6. কখন ব্যবহার করবেন / করবেন না

### ✅ কখন DDD + Microservices ব্যবহার করবেন

```
  ✅ ব্যবহার করুন যখন:

  1. বড় ও জটিল ডোমেইন
     └─ উদাহরণ: Daraz, Pathao, bKash-এর মতো বড় সিস্টেম
     └─ একাধিক ব্যবসায়িক ক্ষমতা (ordering, payment, delivery...)

  2. বড় টিম (১৫+ ডেভেলপার)
     └─ প্রতিটি দল একটি সার্ভিসের মালিক
     └─ স্বাধীনভাবে develop, deploy, scale করতে পারে

  3. আলাদা আলাদা scaling দরকার
     └─ Catalog Service: বেশি read traffic → বেশি instance
     └─ Payment Service: কম traffic কিন্তু বেশি reliability

  4. আলাদা আলাদা টেকনোলজি দরকার
     └─ Search: Elasticsearch
     └─ Payment: PostgreSQL (ACID compliance)
     └─ Caching: Redis

  5. Continuous Deployment দরকার
     └─ একটি সার্ভিস deploy করলে পুরো সিস্টেম re-deploy লাগবে না
```

### ❌ কখন ব্যবহার করবেন না

```
  ❌ ব্যবহার করবেন না যখন:

  1. ছোট টিম (৫ জনের কম)
     └─ Microservices-এর operational overhead বিশাল
     └─ ছোট টিমে monolith অনেক বেশি productive

  2. সরল ডোমেইন
     └─ একটি সাধারণ CRUD অ্যাপ্লিকেশন
     └─ Blog, Portfolio, Simple e-commerce

  3. শুরুর দিকে / MVP পর্যায়ে
     └─ আগে ব্যবসা বুঝুন, পরে ভাগ করুন
     └─ "Premature decomposition is worse than premature optimization"

  4. DevOps দক্ষতা কম
     └─ Kubernetes, Docker, CI/CD, Monitoring
     └─ এগুলো ছাড়া microservices = বিপর্যয়

  5. ডোমেইন boundaries স্পষ্ট না
     └─ ভুল boundary দিয়ে ভাগ করলে পরে ঠিক করা কঠিন
```

### 🏠 "Monolith First" Approach

```
  প্রস্তাবিত পথ:
  ═══════════════

  Phase 1: Modular Monolith           Phase 2: Microservices
  ────────────────────────             ──────────────────────

  ┌────────────────────────┐           ┌──────────┐ ┌──────────┐
  │     Monolith App       │           │ Order    │ │ Catalog  │
  │  ┌──────┐ ┌──────┐    │           │ Service  │ │ Service  │
  │  │Order │ │Catalog│    │    ──▶    └──────────┘ └──────────┘
  │  │Module│ │Module │    │           ┌──────────┐ ┌──────────┐
  │  └──────┘ └──────┘    │           │ Payment  │ │ Shipping │
  │  ┌──────┐ ┌──────┐    │           │ Service  │ │ Service  │
  │  │Pay   │ │Ship  │    │           └──────────┘ └──────────┘
  │  │Module│ │Module │    │
  │  └──────┘ └──────┘    │
  └────────────────────────┘

  ধাপ ১: Monolith-এ DDD ব্যবহার করে Module boundary ঠিক করুন
  ধাপ ২: প্রতিটি Module আলাদা করে Microservice বানান
  ধাপ ৩: Event-driven communication যোগ করুন

  এই পদ্ধতিতে:
  - আগে ডোমেইন ভালো করে বুঝুন
  - সঠিক boundary চিহ্নিত করুন
  - তারপর ধীরে ধীরে ভাগ করুন
```

---

## 📌 7. চেকলিস্ট

### DDD + Microservices প্রয়োগের জন্য চেকলিস্ট

```
  ┌──────────────────────────────────────────────────────────────────┐
  │               DDD + Microservices চেকলিস্ট                      │
  ├──────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │  📋 প্রস্তুতি:                                                  │
  │  □ ডোমেইন এক্সপার্টদের সাথে Event Storming সেশন করেছি          │
  │  □ Bounded Context গুলো চিহ্নিত করেছি                           │
  │  □ Context Map তৈরি করেছি (কোন context কার সাথে কথা বলে)       │
  │  □ Ubiquitous Language প্রতিটি context-এ সংজ্ঞায়িত করেছি       │
  │                                                                  │
  │  🏗️ আর্কিটেকচার:                                               │
  │  □ প্রতিটি Bounded Context = একটি Microservice                  │
  │  □ প্রতিটি সার্ভিসের নিজস্ব Database আছে                       │
  │  □ সার্ভিসগুলো Event দিয়ে communicate করে                       │
  │  □ API Gateway সেটআপ করেছি                                      │
  │  □ ACL দিয়ে বাইরের সিস্টেম থেকে নিজেকে রক্ষা করেছি            │
  │                                                                  │
  │  📦 সার্ভিস ডিজাইন:                                             │
  │  □ প্রতিটি সার্ভিসে DDD layers আছে (Domain/App/Infra)          │
  │  □ Aggregate Root চিহ্নিত করেছি                                 │
  │  □ Domain Events সংজ্ঞায়িত করেছি                                │
  │  □ Value Objects ব্যবহার করেছি                                   │
  │  □ Repository Pattern ব্যবহার করেছি                              │
  │                                                                  │
  │  🔄 Communication:                                               │
  │  □ Synchronous call কমিয়েছি (শুধু query-র জন্য)                 │
  │  □ Asynchronous events বেশি ব্যবহার করেছি                       │
  │  □ Message Broker সেটআপ করেছি (RabbitMQ/Kafka)                  │
  │  □ Saga pattern দরকার হলে implement করেছি                       │
  │  □ Idempotency নিশ্চিত করেছি (duplicate event handle)           │
  │                                                                  │
  │  🛡️ Resilience:                                                  │
  │  □ Circuit Breaker pattern আছে                                  │
  │  □ Retry mechanism আছে                                          │
  │  □ Timeout সেট করেছি                                             │
  │  □ Fallback strategy আছে                                        │
  │  □ Dead Letter Queue আছে (failed events-এর জন্য)               │
  │                                                                  │
  │  📊 Monitoring:                                                  │
  │  □ Centralized logging আছে (ELK Stack)                         │
  │  □ Distributed tracing আছে (Jaeger/Zipkin)                     │
  │  □ Health checks আছে প্রতিটি সার্ভিসে                           │
  │  □ Alerting সেটআপ করেছি                                         │
  │                                                                  │
  │  🚀 Deployment:                                                  │
  │  □ Docker containerization করেছি                                 │
  │  □ CI/CD pipeline আছে প্রতিটি সার্ভিসের জন্য                   │
  │  □ স্বাধীনভাবে deploy করা যায়                                    │
  │  □ Blue-Green বা Canary deployment strategy আছে                 │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

### 🎯 সারসংক্ষেপ

```
  DDD + Microservices = শক্তিশালী কিন্তু জটিল

  মনে রাখুন:
  ┌─────────────────────────────────────────────────┐
  │  ১. Bounded Context = Service Boundary          │
  │  ২. ACL দিয়ে বাইরের দূষণ ঠেকান                 │
  │  ৩. Database share করবেন না                     │
  │  ৪. Event দিয়ে communicate করুন                 │
  │  ৫. Monolith দিয়ে শুরু করুন, পরে ভাগ করুন      │
  │  ৬. সঠিক boundary সবচেয়ে গুরুত্বপূর্ণ সিদ্ধান্ত│
  └─────────────────────────────────────────────────┘
```

---

> **"ভুল boundary দিয়ে ভাগ করা distributed monolith তৈরি করে — যেটা monolith আর microservices দুটোরই সবচেয়ে খারাপ দিক একসাথে পায়।"** — DDD Community
