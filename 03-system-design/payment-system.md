# 💳 পেমেন্ট সিস্টেম ডিজাইন (Payment System Design)

> বাংলাদেশের প্রেক্ষাপটে স্কেলেবল ও নির্ভরযোগ্য পেমেন্ট সিস্টেম — bKash, Nagad, SSLCommerz, Daraz, Pathao-র উদাহরণসহ

---

## 📌 পেমেন্ট সিস্টেম আর্কিটেকচার ওভারভিউ

Daraz-এ অর্ডার করলে বা Pathao-তে রাইড শেষ করলে পেমেন্ট প্রসেসের পেছনে একটি জটিল সিস্টেম কাজ করে।

```
┌───────────┐    ┌────────────┐    ┌─────────────────┐
│  Client   │───▶│ API Gateway│───▶│ Payment Service  │
│ (App/Web) │    │ (nginx/LB) │    │  (Core Logic)    │
└───────────┘    └────────────┘    └───────┬─────────┘
                                           │
                     ┌─────────────────────┼─────────────────────┐
                     ▼                     ▼                     ▼
            ┌──────────────┐    ┌──────────────┐    ┌───────────────┐
            │  bKash API   │    │  Nagad API   │    │  SSLCommerz   │
            │ (MFS Gateway)│    │ (MFS Gateway)│    │ (Card Gateway)│
            └──────┬───────┘    └──────┬───────┘    └──────┬────────┘
                   │                   │                    │
                   ▼                   ▼                    ▼
            ┌───────────────────────────────────────────────────┐
            │        ব্যাংক / MFS সেটেলমেন্ট লেয়ার              │
            │     Bangladesh Bank · DBBL · bKash · Nagad        │
            └───────────────────────────────────────────────────┘
```

| কম্পোনেন্ট | দায়িত্ব | উদাহরণ |
|---|---|---|
| **Client** | পেমেন্ট মেথড সিলেক্ট, UI | Daraz App, Pathao |
| **Payment Service** | পেমেন্ট লজিক, অর্ডার ম্যানেজমেন্ট | Custom Microservice |
| **Payment Gateway** | থার্ড-পার্টি পেমেন্ট প্রসেসর | SSLCommerz, bKash API |
| **Ledger Service** | ডাবল-এন্ট্রি বুককিপিং | Custom Service |

---

## 📊 পেমেন্ট ফ্লো (Payment Flow)

### সফল পেমেন্ট ফ্লো

```
 Customer        Daraz Server      SSLCommerz       bKash Server
    │                │                │                 │
    │ ১. অর্ডার দিলো  │                │                 │
    │───────────────▶│                │                 │
    │                │ ২. পেমেন্ট      │                 │
    │                │  ইনিশিয়েট      │                 │
    │                │───────────────▶│                 │
    │ ৩. রিডাইরেক্ট  │                │                 │
    │◀───────────────│                │                 │
    │ ৪. bKash পে    │                │                 │
    │────────────────────────────────────────────────▶│
    │                │                │ ৫. ভেরিফাই      │
    │                │                │◀────────────────│
    │                │ ৬. ওয়েবহুক     │                 │
    │                │◀───────────────│ (success)       │
    │ ৭. কনফার্মেশন  │                │                 │
    │◀───────────────│                │                 │
```

### পেমেন্ট স্ট্যাটাস মেশিন

```
              ┌──────────┐
              │ CREATED  │
              └────┬─────┘
                   ▼
              ┌──────────┐     টাইমআউট
              │ PENDING  │──────────────▶ EXPIRED
              └────┬─────┘
            ┌──────┼──────┐
            ▼             ▼
      ┌──────────┐  ┌──────────┐
      │ SUCCESS  │  │  FAILED  │──▶ RETRYING
      └────┬─────┘  └──────────┘
           ▼
      ┌──────────┐       ┌──────────┐
      │ SETTLED  │       │ REFUNDED │
      └──────────┘       └──────────┘
```

---

## 🔗 পেমেন্ট গেটওয়ে ইন্টিগ্রেশন

### SSLCommerz ইন্টিগ্রেশন (PHP)

```php
// ✅ ভালো: SSLCommerz পেমেন্ট ইনিশিয়েশন
class SSLCommerzPayment
{
    public function initiatePayment(Order $order): array
    {
        $transactionId = 'TXN_' . uniqid() . '_' . time();

        $postData = [
            'store_id'     => config('payment.sslcommerz.store_id'),
            'store_passwd' => config('payment.sslcommerz.store_password'),
            'total_amount' => $order->total,
            'currency'     => 'BDT',
            'tran_id'      => $transactionId,
            'success_url'  => route('payment.success'),
            'fail_url'     => route('payment.fail'),
            'cancel_url'   => route('payment.cancel'),
            'ipn_url'      => route('payment.ipn'),
            'cus_name'     => $order->customer->name,
            'cus_phone'    => $order->customer->phone,
        ];

        // ডাটাবেসে ট্রানজেকশন সেভ (PENDING স্ট্যাটাসে)
        Payment::create([
            'transaction_id' => $transactionId,
            'order_id'       => $order->id,
            'amount'         => $order->total,
            'status'         => 'PENDING',
            'gateway'        => 'sslcommerz',
        ]);

        return Http::asForm()->post($this->apiUrl, $postData)->json();
    }
}
```

```php
// ❌ খারাপ: সিক্রেট হার্ডকোড, ইনপুট ভ্যালিডেশন নেই, ট্রানজেকশন সেভ নেই
$postData = [
    'store_id'     => 'mystore123',        // ❌ হার্ডকোডেড!
    'store_passwd' => 'mypassword@123',    // ❌ পাসওয়ার্ড এক্সপোজড!
    'total_amount' => $_POST['amount'],    // ❌ ভ্যালিডেট করা হয়নি!
];
```

### Stripe ইন্টিগ্রেশন (JavaScript)

```javascript
// ✅ ভালো: Stripe Payment Intent
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

async function createPaymentIntent(order) {
  const paymentIntent = await stripe.paymentIntents.create(
    {
      amount: order.totalInPaisa,
      currency: 'bdt',
      metadata: { orderId: order.id },
      payment_method_types: ['card'],
    },
    { idempotencyKey: `order_${order.id}` }
  );

  await Payment.create({
    transactionId: paymentIntent.id,
    orderId: order.id,
    amount: order.totalInPaisa,
    status: 'PENDING',
  });

  return { clientSecret: paymentIntent.client_secret };
}
```

---

## 🔑 আইডেম্পোটেন্সি (Idempotency)

একই রিকোয়েস্ট বারবার পাঠালেও ফলাফল একবারই কার্যকর হবে। পেমেন্টে এটি অত্যন্ত
গুরুত্বপূর্ণ — নেটওয়ার্ক সমস্যায় ডাবল চার্জ হয়ে যেতে পারে!

```
  কাস্টমার "Pay" ক্লিক → রিকোয়েস্ট গেলো → নেটওয়ার্ক স্লো → আবার ক্লিক
  ❌ আইডেম্পোটেন্সি ছাড়া: ৫,০০০ × ২ = ১০,০০০ টাকা কেটে গেছে!
  ✅ আইডেম্পোটেন্সি সহ: দ্বিতীয়বার আগের রেসপন্স ফেরত দেয়
```

```php
// ✅ PHP আইডেম্পোটেন্সি হ্যান্ডলার
public function processPayment(Request $request): JsonResponse
{
    $idempotencyKey = $request->header('Idempotency-Key');
    if (!$idempotencyKey) {
        return response()->json(['error' => 'Idempotency-Key আবশ্যক'], 400);
    }

    $existing = DB::table('idempotency_keys')->where('key', $idempotencyKey)->first();
    if ($existing) {
        return response()->json(json_decode($existing->response), $existing->status_code);
    }

    DB::beginTransaction();
    try {
        DB::table('idempotency_keys')->insert([
            'key' => $idempotencyKey, 'status' => 'processing', 'created_at' => now(),
        ]);

        $result = $this->chargeCustomer($request);

        DB::table('idempotency_keys')->where('key', $idempotencyKey)->update([
            'response' => json_encode($result), 'status_code' => 200, 'status' => 'completed',
        ]);

        DB::commit();
        return response()->json($result, 200);
    } catch (\Exception $e) {
        DB::rollBack();
        throw $e;
    }
}
```

```sql
CREATE TABLE idempotency_keys (
    key         VARCHAR(255) PRIMARY KEY,
    status      ENUM('processing', 'completed', 'failed') NOT NULL,
    response    JSON,
    status_code INT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at  TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL 24 HOUR)
);
```

---

## 📖 ডাবল-এন্ট্রি বুককিপিং (Double-Entry Bookkeeping)

প্রতিটি লেনদেনে **ডেবিট** ও **ক্রেডিট** — হিসাব সবসময় ব্যালেন্সড থাকে।

### উদাহরণ: bKash সেন্ড মানি (রহিম → করিম ১,০০০ টাকা)

```
┌──────────────────┬──────────┬──────────┬──────────────┐
│  Account         │  Debit   │  Credit  │  বিবরণ        │
├──────────────────┼──────────┼──────────┼──────────────┤
│  রহিম (Sender)   │  ১,০০০   │          │  সেন্ড মানি    │
│  করিম (Receiver) │          │  ১,০০০   │  ক্যাশ ইন      │
│  রহিম (Sender)   │  ৫       │          │  সার্ভিস চার্জ  │
│  bKash Revenue   │          │  ৫       │  চার্জ আয়      │
├──────────────────┼──────────┼──────────┼──────────────┤
│  মোট             │  ১,০০৫   │  ১,০০৫   │  ✅ ব্যালেন্সড  │
└──────────────────┴──────────┴──────────┴──────────────┘
```

```sql
CREATE TABLE ledger_entries (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_id  VARCHAR(50) NOT NULL,
    account_id      BIGINT NOT NULL,
    entry_type      ENUM('DEBIT', 'CREDIT') NOT NULL,
    amount          DECIMAL(15,2) NOT NULL,
    balance_after   DECIMAL(15,2) NOT NULL,
    description     VARCHAR(255),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_transaction (transaction_id)
);
```

```php
// ✅ ডাবল-এন্ট্রি ট্রানজেকশন
public function transferMoney(Account $sender, Account $receiver, float $amount, float $fee = 0): Transaction
{
    return DB::transaction(function () use ($sender, $receiver, $amount, $fee) {
        $txnId = 'TXN_' . date('Ymd') . '_' . uniqid();

        $senderAcct = Account::lockForUpdate()->find($sender->id);
        if ($senderAcct->balance < ($amount + $fee)) {
            throw new InsufficientBalanceException('পর্যাপ্ত ব্যালেন্স নেই');
        }

        // ডেবিট: সেন্ডার থেকে কাটো
        $senderAcct->balance -= ($amount + $fee);
        $senderAcct->save();
        LedgerEntry::create([
            'transaction_id' => $txnId, 'account_id' => $senderAcct->id,
            'entry_type' => 'DEBIT', 'amount' => $amount + $fee,
            'balance_after' => $senderAcct->balance,
        ]);

        // ক্রেডিট: রিসিভারে যোগ করো
        $receiverAcct = Account::lockForUpdate()->find($receiver->id);
        $receiverAcct->balance += $amount;
        $receiverAcct->save();
        LedgerEntry::create([
            'transaction_id' => $txnId, 'account_id' => $receiverAcct->id,
            'entry_type' => 'CREDIT', 'amount' => $amount,
            'balance_after' => $receiverAcct->balance,
        ]);

        return Transaction::create([
            'id' => $txnId, 'type' => 'SEND_MONEY', 'status' => 'COMPLETED',
            'sender_id' => $sender->user_id, 'receiver_id' => $receiver->user_id,
            'amount' => $amount, 'fee' => $fee,
        ]);
    });
}
```

---

## ⚠️ ফেইলিওর হ্যান্ডলিং ও রিকনসিলিয়েশন

### সম্ভাব্য ফেইলিওর পয়েন্ট

```
  Client ──▶ API ──▶ Payment Svc ──▶ Gateway ──▶ Bank
    ❌         ❌          ❌             ❌         ❌
  নেটওয়ার্ক  সার্ভার    DB ফেইল        টাইমআউট   ব্যালেন্স নেই
```

### রিকনসিলিয়েশন ফ্লো

আমাদের রেকর্ড ও গেটওয়ে/ব্যাংকের রেকর্ড প্রতিদিন মিলিয়ে দেখা:

```
  ┌──────────────────┐        ┌──────────────────┐
  │ আমাদের ডাটাবেস   │        │ গেটওয়ে রিপোর্ট   │
  │ (Payment Records)│        │ (SSLCommerz CSV) │
  └────────┬─────────┘        └────────┬─────────┘
           └──────────┬───────────────┘
                      ▼
              ┌───────────────┐
              │  Reconciler   │
              └───────┬───────┘
          ┌───────────┼───────────┐
          ▼           ▼           ▼
     ✅ ম্যাচ     ⚠️ মিসম্যাচ    ❌ মিসিং
     (OK)       (তদন্ত)       (ম্যানুয়াল রিভিউ)
```

| ক্যাটাগরি | ব্যাখ্যা | অ্যাকশন |
|---|---|---|
| **ম্যাচ** | আমাদের রেকর্ড = গেটওয়ে রেকর্ড | কিছু করার দরকার নেই |
| **Amount Mismatch** | টাকার পরিমাণ আলাদা | ম্যানুয়াল রিভিউ |
| **Missing in Gateway** | আমাদের আছে, গেটওয়েতে নেই | রিফান্ড বিবেচনা |
| **Missing in System** | গেটওয়েতে আছে, আমাদের নেই | ট্রানজেকশন তৈরি করতে হবে |

---

## 🛡️ PCI DSS কমপ্লায়েন্স

### ✅ যা করবেন / ❌ যা করবেন না

```
  ✅ করবেন:                              ❌ করবেন না:
  ├── টোকেনাইজেশন ব্যবহার করুন            ├── CVV কখনো স্টোর করবেন না!
  ├── HTTPS সবখানে বাধ্যতামূলক             ├── পূর্ণ কার্ড নম্বর DB-তে রাখবেন না
  ├── কার্ড মাস্ক করুন (**** 4242)         ├── কার্ড ডেটা লগে লিখবেন না
  ├── Hosted checkout page ব্যবহার করুন   ├── প্লেইন টেক্সটে সেনসিটিভ ডেটা না
  └── নিয়মিত সিকিউরিটি অডিট               └── কার্ড ডেটা ইমেইল/SMS-এ না
```

| লেভেল | বার্ষিক ট্রানজেকশন | প্রয়োজনীয়তা |
|---|---|---|
| **Level 1** | ৬০ লক্ষ+ | বাহ্যিক অডিটর (QSA) |
| **Level 2** | ১০ - ৬০ লক্ষ | Self-Assessment Questionnaire |
| **Level 3** | ২০K - ১০ লক্ষ | SAQ + নেটওয়ার্ক স্ক্যান |
| **Level 4** | ২০K-এর কম | সরলীকৃত SAQ |

> 💡 SSLCommerz-এর hosted checkout ব্যবহার করলে PCI বোঝা তারাই বহন করে (SAQ-A)।

---

## 🔍 ফ্রড ডিটেকশন (Fraud Detection)

```
                    ┌─────────────────────┐
                    │  নতুন ট্রানজেকশন     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
               ┌────│ ১ ঘণ্টায় ৫+ টিক্সেন?│────┐
             হ্যাঁ   └─────────────────────┘   না
               │                                │
          🚫 ব্লক                    ┌──────────▼──────────┐
                                ┌───│ দৈনিক লিমিট ৮০%+?  │───┐
                              হ্যাঁ  └─────────────────────┘  না
                                │                              │
                           ⚠️ OTP                  ┌──────────▼──────────┐
                                              ┌───│ স্বাভাবিক লোকেশন?   │───┐
                                            না   └─────────────────────┘  হ্যাঁ
                                              │                              │
                                         🚫 ব্লক                   ┌────────▼────────┐
                                                               ┌───│ পরিচিত ডিভাইস? │───┐
                                                             না   └─────────────────┘  হ্যাঁ
                                                              │                          │
                                                         ⚠️ OTP               ✅ অনুমোদিত
```

| রুল | বিবরণ | অ্যাকশন |
|---|---|---|
| **Velocity** | ১ ঘণ্টায় ৫+ ট্রানজেকশন | ব্লক + রিভিউ |
| **Amount** | একক ট্রানজেকশন ৫০,০০০+ টাকা | OTP ভেরিফিকেশন |
| **Geo-mismatch** | ঢাকা লগইন, চট্টগ্রাম পেমেন্ট | ব্লক + ভেরিফাই |
| **Device** | নতুন ডিভাইস + বড় অ্যামাউন্ট | চ্যালেঞ্জ + OTP |
| **Smurfing** | একই রিসিভারে বারবার ছোট অ্যামাউন্ট | ফ্ল্যাগ |

---

## 📬 ওয়েবহুক হ্যান্ডলিং (Webhook Handling)

গেটওয়ে পেমেন্ট সম্পন্ন হলে IPN/Webhook পাঠায়। সঠিকভাবে হ্যান্ডেল করতে হবে:
**১.** সিগনেচার ভেরিফাই **২.** আইডেম্পোটেন্ট প্রসেসিং **৩.** দ্রুত ২০০ রেসপন্স **৪.** কিউতে ভারী কাজ

```php
// ✅ SSLCommerz IPN হ্যান্ডলার (PHP)
public function handleIPN(Request $request): JsonResponse
{
    if (!$this->verifySignature($request)) {
        return response()->json(['status' => 'INVALID_SIGNATURE'], 403);
    }

    $transactionId = $request->input('tran_id');
    $payment = Payment::where('transaction_id', $transactionId)->first();

    if (!$payment || $payment->status === 'COMPLETED') {
        return response()->json(['status' => 'ALREADY_PROCESSED']);
    }

    // সার্ভার-টু-সার্ভার ভেরিফিকেশন
    $validation = Http::get(config('sslcommerz.validation_url'), [
        'val_id' => $request->input('val_id'),
        'store_id' => config('sslcommerz.store_id'),
        'store_passwd' => config('sslcommerz.store_password'),
    ]);

    if ($validation->json('status') !== 'VALID') {
        $payment->update(['status' => 'VALIDATION_FAILED']);
        return response()->json(['status' => 'VALIDATION_FAILED']);
    }

    // অ্যামাউন্ট ম্যাচ চেক
    if ((float) $request->input('amount') !== (float) $payment->amount) {
        $payment->update(['status' => 'AMOUNT_MISMATCH']);
        return response()->json(['status' => 'AMOUNT_MISMATCH']);
    }

    $payment->update(['status' => 'COMPLETED']);
    ProcessOrderAfterPayment::dispatch($payment);
    return response()->json(['status' => 'SUCCESS']);
}
```

```javascript
// ✅ Stripe Webhook হ্যান্ডলার (JS)
const endpointSecret = process.env.STRIPE_WEBHOOK_SECRET;

app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, req.headers['stripe-signature'], endpointSecret);
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  res.status(200).json({ received: true }); // দ্রুত রেসপন্স

  switch (event.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSuccess(event.data.object);
      break;
    case 'payment_intent.payment_failed':
      await handlePaymentFailure(event.data.object);
      break;
  }
});
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কাস্টম পেমেন্ট সিস্টেম বানাবেন:

```
  ✅ বড় ফিনটেক প্ল্যাটফর্ম (bKash/Nagad-এর মতো)
  ✅ মাল্টিপল গেটওয়ে সাপোর্ট দরকার
  ✅ স্প্লিট পেমেন্ট, সাবস্ক্রিপশন লজিক
  ✅ বিস্তারিত লেজার ও রিপোর্টিং
  ✅ প্রতিদিন লক্ষাধিক ট্রানজেকশন
```

### ❌ কাস্টম পেমেন্ট সিস্টেম বানাবেন না:

```
  ❌ ছোট ই-কমার্স (SSLCommerz Plugin ব্যবহার করুন)
  ❌ MVP / প্রোটোটাইপ স্টেজে
  ❌ টিমে পেমেন্ট সিস্টেমের অভিজ্ঞতা নেই
  ❌ PCI DSS-এর বাজেট/রিসোর্স নেই
  ❌ সময় কম — ৩ মাসের কম সময়ে লঞ্চ
```

### সিদ্ধান্ত ফ্লোচার্ট

```
  মাসিক ট্রানজেকশন ১ লক্ষ+?
  ├── না → SSLCommerz Plugin ✅
  └── হ্যাঁ → মাল্টিপল গেটওয়ে দরকার?
              ├── না → গেটওয়ের SDK ✅
              └── হ্যাঁ → স্প্লিট পেমেন্ট / লেজার দরকার?
                          ├── না → Thin Payment Layer ✅
                          └── হ্যাঁ → কাস্টম সিস্টেম বানান ✅
```

---

## 📊 বাংলাদেশের পেমেন্ট গেটওয়ে তুলনা

| বৈশিষ্ট্য | SSLCommerz | bKash PGW | Nagad PGW | AmarPay | PortWallet |
|---|---|---|---|---|---|
| **কার্ড পেমেন্ট** | ✅ Visa/Master | ❌ | ❌ | ✅ Visa/Master | ✅ Visa/Master |
| **MFS সাপোর্ট** | ✅ সব MFS | ✅ শুধু bKash | ✅ শুধু Nagad | ✅ bKash/Nagad | ✅ bKash/Nagad |
| **ইন্টারনেট ব্যাংকিং** | ✅ | ❌ | ❌ | ✅ | ✅ |
| **সেটআপ ফি** | ১০-২৫K৳ | বিনামূল্যে | বিনামূল্যে | ৫K৳ | ১০K৳ |
| **ট্রানজেকশন ফি** | ১.৮-২.৫% | ১.৫-১.৮% | ১.৫% | ১.৫-২% | ২-২.৫% |
| **সেটেলমেন্ট** | T+2 | T+1 | T+1 | T+2 | T+2 |
| **API Docs** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Sandbox** | ✅ | ✅ | ✅ | ✅ | ✅ |

```
  কোনটা কখন?
  বড় ই-কমার্স (Daraz)       → SSLCommerz (সব মেথড)
  ফুড ডেলিভারি অ্যাপ          → bKash PGW (দ্রুত MFS)
  ফ্রিল্যান্সার পেমেন্ট         → Nagad PGW (কম ফি)
  ছোট ই-কমার্স               → AmarPay (সহজ সেটআপ)
  এন্টারপ্রাইজ                 → PortWallet (কাস্টমাইজযোগ্য)
```

---

## 💡 মূল শিক্ষণীয় বিষয়

```
  ১. 🔑 আইডেম্পোটেন্সি — ডাবল চার্জ কখনো গ্রহণযোগ্য নয়
  ২. 📖 ডাবল-এন্ট্রি — প্রতিটি পয়সার হিসাব রাখুন
  ৩. ⚠️ ফেইলিওর হ্যান্ডলিং ডিজাইনের অংশ
  ৪. 🛡️ সিকিউরিটি — PCI DSS মানুন, CVV সেভ না
  ৫. 📬 ওয়েবহুক — সিগনেচার যাচাই + আইডেম্পোটেন্সি
  ৬. 🔍 ফ্রড ডিটেকশন শুরু থেকেই রাখুন
  ৭. 📊 রিকনসিলিয়েশন প্রতিদিন চালান
```

---

> **"পেমেন্ট সিস্টেমে ভুলের জায়গা নেই — প্রতিটি পয়সা ট্র্যাক করুন, প্রতিটি ফেইলিওর হ্যান্ডেল করুন।"** 💳
