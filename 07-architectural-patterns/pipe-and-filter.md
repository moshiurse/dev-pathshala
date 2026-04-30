# 🔧 পাইপ অ্যান্ড ফিল্টার প্যাটার্ন (Pipe and Filter Architecture Pattern)

## 📌 সংজ্ঞা ও মূল ধারণা

**Pipe and Filter** হলো একটি আর্কিটেকচারাল প্যাটার্ন যেখানে ডেটা প্রসেসিংকে একাধিক **স্বতন্ত্র ধাপে (Filter)** ভাগ করা হয় এবং এই ধাপগুলো **Pipe** দিয়ে সংযুক্ত থাকে। প্রতিটি Filter একটি নির্দিষ্ট কাজ করে — সে input গ্রহণ করে, transform করে, এবং output হিসেবে পরবর্তী Filter-এ পাঠায়।

এই প্যাটার্নের মূল দর্শন হলো **separation of concerns** এবং **composability**। প্রতিটি Filter আলাদাভাবে develop, test, এবং maintain করা যায়। নতুন Filter যোগ করা, পুরনো Filter সরানো, বা Filter-এর ক্রম পরিবর্তন করা অত্যন্ত সহজ।

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                    PIPE AND FILTER — মূল কাঠামো                         ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                         ║
║   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐  ║
║   │  Data    │ Pipe │ Filter A │ Pipe │ Filter B │ Pipe │ Filter C │  ║
║   │  Source  │─────→│(Validate)│─────→│(Transform│─────→│ (Format) │  ║
║   │          │      │          │      │          │      │          │  ║
║   └──────────┘      └──────────┘      └──────────┘      └──────────┘  ║
║        ↑                                                      │        ║
║     Input                                                  Output      ║
║    (Raw Data)                                         (Processed Data) ║
║                                                                         ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

### Unix Philosophy — এই প্যাটার্নের জন্মস্থান

Unix command line-ই এই প্যাটার্নের সবচেয়ে পরিচিত উদাহরণ:

```bash
cat access.log | grep "404" | awk '{print $7}' | sort | uniq -c | sort -rn | head -10
```

এখানে প্রতিটি command একটি Filter, আর `|` হলো Pipe। প্রতিটি command স্বতন্ত্র — সে জানে না তার আগে বা পরে কে আছে।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### উদাহরণ ১: বাংলাদেশের পানি বিশুদ্ধকরণ (RO Filter System)

```
╔═══════════════════════════════════════════════════════════════════════╗
║              🚰 পানি বিশুদ্ধকরণ — Pipe and Filter অ্যানালজি        ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  নলকূপের       Sediment      Carbon       RO            UV           ║
║  কাঁচা পানি    Filter        Filter       Membrane      Filter       ║
║     │          ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐   ║
║     │          │ বালি  │      │ ক্লোরিন│     │ ভারী  │      │ ব্যাক │   ║
║     ├────────→│ কাদা  │────→│ গন্ধ  │────→│ ধাতু  │────→│ টেরিয়া│   ║
║     │          │ মরিচা │      │ রাসায়ন│     │ আর্সে │      │ ভাইরাস│   ║
║     │          │ দূর   │      │ দূর   │      │ নিক   │      │ দূর   │   ║
║     │          └──────┘      └──────┘      └──────┘      └──────┘   ║
║                                                               │       ║
║                                                          বিশুদ্ধ      ║
║                                                          পানি ✅      ║
╚═══════════════════════════════════════════════════════════════════════╝

প্রতিটি Filter:
  ✅ একটি নির্দিষ্ট দূষণকারী দূর করে (Single Responsibility)
  ✅ স্বতন্ত্রভাবে কাজ করে (Independent)
  ✅ প্রতিস্থাপনযোগ্য (Replaceable)
  ✅ যেকোনো ক্রমে সাজানো যায় (Reorderable — তবে কিছু ক্রম ভালো কাজ করে)
```

### উদাহরণ ২: বাংলাদেশের গার্মেন্টস ফ্যাক্টরির Assembly Line

```
কাপড় কাটা → সেলাই → বোতাম লাগানো → ইস্ত্রি → Quality Check → প্যাকেজিং
   [Cut]  →  [Sew]  →  [Button]   → [Iron] →    [QC]     →  [Pack]

প্রতিটি স্টেশন (Filter) একটি নির্দিষ্ট কাজ করে।
একটি স্টেশনের কাজ শেষ হলে পরের স্টেশনে পাঠায় (Pipe)।
কোনো স্টেশন বুঝতে পারে না আগের স্টেশনে কী হয়েছে — সে শুধু তার কাজটুকু করে।
```

---

## 📖 Components বিস্তারিত

### ১. Filter (ফিল্টার)

Filter হলো একটি **স্বতন্ত্র প্রসেসিং ইউনিট** যা:
- Input গ্রহণ করে
- কোনো transformation বা computation করে
- Output তৈরি করে

```
┌──────────────────────────────────────────────┐
│              Filter-এর বৈশিষ্ট্য             │
├──────────────────────────────────────────────┤
│ ✅ Independent — অন্য Filter-এর উপর নির্ভরশীল নয়  │
│ ✅ Stateless — নিজের মধ্যে কোনো state রাখে না     │
│ ✅ Reusable — বিভিন্ন pipeline-এ ব্যবহারযোগ্য     │
│ ✅ Testable — আলাদাভাবে test করা যায়             │
│ ✅ Replaceable — সহজে প্রতিস্থাপনযোগ্য           │
│ ✅ Single Responsibility — একটি মাত্র কাজ করে      │
└──────────────────────────────────────────────┘
```

#### PHP Filter Interface ও Implementation

```php
<?php

interface Filter
{
    /**
     * ডেটা প্রসেস করে পরবর্তী ধাপে পাঠায়
     */
    public function handle(mixed $data): mixed;
}

// ──────────── Concrete Filter গুলো ────────────

class TrimFilter implements Filter
{
    public function handle(mixed $data): mixed
    {
        if (is_string($data)) {
            return trim($data);
        }
        if (is_array($data)) {
            return array_map('trim', $data);
        }
        return $data;
    }
}

class LowercaseFilter implements Filter
{
    public function handle(mixed $data): mixed
    {
        if (is_string($data)) {
            return strtolower($data);
        }
        return $data;
    }
}

class RemoveSpecialCharsFilter implements Filter
{
    public function handle(mixed $data): mixed
    {
        if (is_string($data)) {
            return preg_replace('/[^a-zA-Z0-9\s\p{Bengali}]/u', '', $data);
        }
        return $data;
    }
}
```

#### JavaScript Filter Examples

```javascript
// প্রতিটি Filter একটি pure function
const trimFilter = (data) => {
    if (typeof data === 'string') return data.trim();
    if (Array.isArray(data)) return data.map(item => item.trim());
    return data;
};

const lowercaseFilter = (data) => {
    if (typeof data === 'string') return data.toLowerCase();
    return data;
};

const removeSpecialCharsFilter = (data) => {
    if (typeof data === 'string') return data.replace(/[^a-zA-Z0-9\s]/g, '');
    return data;
};
```

### ২. Pipe (পাইপ)

Pipe হলো **Filter-দের মধ্যে সংযোগকারী** — এটি একটি Filter-এর output নিয়ে পরবর্তী Filter-এর input হিসেবে পাঠায়।

```
┌──────────────────────────────────────────────────────────┐
│                    Pipe-এর ধরন                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ১. Synchronous Pipe (সিঙ্ক্রোনাস)                       │
│     Filter A ──→ তৎক্ষণাৎ ──→ Filter B                   │
│     সরাসরি function call, কোনো buffering নেই             │
│                                                          │
│  ২. Asynchronous Pipe (অ্যাসিঙ্ক্রোনাস)                   │
│     Filter A ──→ Queue/Buffer ──→ Filter B               │
│     Message queue ব্যবহার করে, Filter আলাদা গতিতে চলে    │
│                                                          │
│  ৩. Buffered Pipe (বাফার্ড)                               │
│     Filter A ──→ [Buffer] ──→ Filter B                   │
│     নির্দিষ্ট পরিমাণ ডেটা জমা হলে পরবর্তী Filter-এ পাঠায়  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### ৩. Data Source ও Data Sink

```
┌───────────┐                                              ┌───────────┐
│   Data    │   যেখান থেকে ডেটা আসে                        │   Data    │
│   Source  │   - Database query                           │   Sink    │
│           │   - API request                              │           │
│  (উৎস)    │   - File read (CSV, JSON)                    │  (গন্তব্য) │
│           │   - User input                               │           │
│           │   - Message queue                            │           │
└───────────┘                                              └───────────┘
                                                           যেখানে প্রসেসড
                                                           ডেটা যায়:
                                                           - Database save
                                                           - API response
                                                           - File write
                                                           - Email/SMS send
```

---

## 📊 Use Cases (ব্যবহারের ক্ষেত্র)

### ১. ডেটা Transformation Pipeline (ETL)

```
  CSV File        Parse         Validate       Transform       Load
  ┌──────┐      ┌──────┐      ┌──────┐       ┌──────┐      ┌──────┐
  │ Raw  │─────→│ CSV  │─────→│ Data │──────→│Format│─────→│ Save │
  │ Data │      │Parser│      │Check │       │Change│      │to DB │
  └──────┘      └──────┘      └──────┘       └──────┘      └──────┘

Daraz-এ প্রতিদিন হাজার হাজার product CSV আপলোড হয়।
প্রতিটি ধাপ আলাদা Filter হিসেবে কাজ করে।
```

### ২. Image Processing Pipeline

```
  Original      Resize        Crop         Watermark     Compress
  ┌──────┐    ┌──────┐     ┌──────┐      ┌──────┐     ┌──────┐
  │ 📷   │───→│ 800x │────→│Center│─────→│ Logo │────→│ WebP │
  │Image │    │ 600  │     │ Crop │      │ যোগ  │     │ 80%  │
  └──────┘    └──────┘     └──────┘      └──────┘     └──────┘

Daraz/Bikroy-তে seller product ছবি আপলোড করলে
এই pipeline দিয়ে ছবি প্রসেস হয়।
```

### ৩. Order Processing (Daraz/Chaldal)

```
  New Order → [Validate] → [CalcPrice] → [Discount] → [Tax] → [Invoice]
                  │              │            │          │         │
              অর্ডার তথ্য    মূল্য গণনা    কুপন/অফার  ভ্যাট    চালান তৈরি
              যাচাই         করা          প্রয়োগ     হিসাব
```

### ৪. bKash Payment Processing

```
  ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Validate │──→│  Fraud    │──→│ Process  │──→│  Update  │──→│  Send    │
  │ Request  │   │  Check    │   │ Payment  │   │ Balance  │   │  SMS     │
  │          │   │           │   │          │   │          │   │          │
  │ পিন যাচাই│   │ সন্দেহজনক │   │ টাকা     │   │ ব্যালেন্স │   │ নোটিফি-  │
  │ ব্যালেন্স │   │ লেনদেন   │   │ স্থানান্তর│   │ আপডেট   │   │ কেশন    │
  │ চেক     │   │ শনাক্ত    │   │          │   │          │   │ পাঠানো   │
  └──────────┘   └───────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## 🎯 Laravel Pipeline — পারফেক্ট Implementation

Laravel-এর `Pipeline` ক্লাস হলো Pipe and Filter প্যাটার্নের একটি চমৎকার বাস্তবায়ন। Laravel-এর middleware system এই Pipeline ব্যবহার করে।

### Laravel Pipeline দিয়ে Order Processing

```php
<?php

use Illuminate\Pipeline\Pipeline;

// ──────────── Pipe Classes ────────────

class ValidateOrder
{
    public function handle($order, Closure $next)
    {
        if (empty($order->items)) {
            throw new \Exception('অর্ডারে কোনো পণ্য নেই!');
        }

        if (!$order->customer_id) {
            throw new \Exception('কাস্টমার আইডি প্রয়োজন!');
        }

        return $next($order);
    }
}

class CalculateTotal
{
    public function handle($order, Closure $next)
    {
        $total = 0;
        foreach ($order->items as $item) {
            $total += $item->price * $item->quantity;
        }
        $order->subtotal = $total;

        return $next($order);
    }
}

class ApplyDiscount
{
    public function handle($order, Closure $next)
    {
        if ($order->coupon_code) {
            $discount = Coupon::find($order->coupon_code);
            if ($discount && $discount->isValid()) {
                $order->discount = $order->subtotal * ($discount->percentage / 100);
                $order->subtotal -= $order->discount;
            }
        }

        return $next($order);
    }
}

class CalculateVAT
{
    public function handle($order, Closure $next)
    {
        // বাংলাদেশে VAT ১৫%
        $order->vat = $order->subtotal * 0.15;
        $order->total = $order->subtotal + $order->vat;

        return $next($order);
    }
}

class CalculateShipping
{
    public function handle($order, Closure $next)
    {
        // ঢাকার ভেতরে ৬০ টাকা, বাইরে ১২০ টাকা
        $order->shipping = $order->is_dhaka ? 60 : 120;
        $order->total += $order->shipping;

        return $next($order);
    }
}

class ProcessPayment
{
    public function handle($order, Closure $next)
    {
        $paymentResult = PaymentGateway::charge(
            $order->payment_method,
            $order->total
        );

        $order->payment_status = $paymentResult->success ? 'paid' : 'failed';
        $order->transaction_id = $paymentResult->transaction_id;

        if (!$paymentResult->success) {
            throw new PaymentFailedException('পেমেন্ট ব্যর্থ হয়েছে!');
        }

        return $next($order);
    }
}

// ──────────── Pipeline ব্যবহার ────────────

$processedOrder = app(Pipeline::class)
    ->send($order)
    ->through([
        ValidateOrder::class,
        CalculateTotal::class,
        ApplyDiscount::class,
        CalculateVAT::class,
        CalculateShipping::class,
        ProcessPayment::class,
    ])
    ->thenReturn();
```

### Custom Pipeline Implementation (Laravel ছাড়া)

```php
<?php

class Pipeline
{
    private mixed $passable;
    private array $pipes = [];

    public function send(mixed $passable): self
    {
        $this->passable = $passable;
        return $this;
    }

    public function through(array $pipes): self
    {
        $this->pipes = $pipes;
        return $this;
    }

    public function thenReturn(): mixed
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes),
            function ($next, $pipe) {
                return function ($passable) use ($next, $pipe) {
                    if (is_callable($pipe)) {
                        return $pipe($passable, $next);
                    }
                    $instance = new $pipe();
                    return $instance->handle($passable, $next);
                };
            },
            fn($passable) => $passable // শেষ ধাপ: ডেটা return করে
        );

        return $pipeline($this->passable);
    }
}

// ব্যবহার:
$result = (new Pipeline())
    ->send($order)
    ->through([
        ValidateOrder::class,
        CalculateTotal::class,
        ApplyDiscount::class,
    ])
    ->thenReturn();
```

---

## ⚡ Express Middleware — Pipe and Filter in Action

Express.js-এর middleware system **হুবহু** Pipe and Filter প্যাটার্ন। প্রতিটি middleware হলো একটি Filter, আর Express-এর `next()` হলো Pipe।

```
Request → [auth] → [validate] → [rateLimit] → [process] → [format] → Response
            │          │            │             │            │
         পরিচয়     ইনপুট         সীমা          মূল          রেসপন্স
         যাচাই     যাচাই         নিয়ন্ত্রণ      কাজ          ফরম্যাট
```

### Express Middleware Pipeline

```javascript
const express = require('express');
const app = express();

// ──────────── Filter 1: Authentication ────────────
const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];

    if (!token) {
        return res.status(401).json({ error: 'টোকেন প্রদান করুন' });
    }

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next(); // পরবর্তী Filter-এ পাঠাও
    } catch (err) {
        return res.status(401).json({ error: 'অবৈধ টোকেন' });
    }
};

// ──────────── Filter 2: Input Validation ────────────
const validateInput = (req, res, next) => {
    const { amount, recipient } = req.body;

    if (!amount || amount <= 0) {
        return res.status(400).json({ error: 'সঠিক পরিমাণ দিন' });
    }
    if (!recipient) {
        return res.status(400).json({ error: 'প্রাপকের নম্বর দিন' });
    }

    req.validatedData = { amount, recipient };
    next();
};

// ──────────── Filter 3: Rate Limiting ────────────
const rateLimit = (req, res, next) => {
    const userId = req.user.id;
    const requests = rateLimitStore.get(userId) || 0;

    if (requests >= 100) {
        return res.status(429).json({ error: 'অনেক বেশি রিকুয়েস্ট! পরে চেষ্টা করুন' });
    }

    rateLimitStore.set(userId, requests + 1);
    next();
};

// ──────────── Filter 4: Process Transaction ────────────
const processTransaction = async (req, res, next) => {
    const { amount, recipient } = req.validatedData;

    const transaction = await Transaction.create({
        sender: req.user.id,
        recipient,
        amount,
        status: 'completed',
    });

    req.transaction = transaction;
    next();
};

// ──────────── Filter 5: Format Response ────────────
const formatResponse = (req, res) => {
    res.json({
        success: true,
        message: 'লেনদেন সফল হয়েছে',
        data: {
            transactionId: req.transaction.id,
            amount: req.transaction.amount,
            recipient: req.transaction.recipient,
        },
    });
};

// Pipeline সাজানো — প্রতিটি middleware একটি Filter
app.post(
    '/api/send-money',
    authenticate,         // Filter 1
    validateInput,        // Filter 2
    rateLimit,            // Filter 3
    processTransaction,   // Filter 4
    formatResponse        // Filter 5 (Sink)
);
```

---

## 🔀 Advanced Patterns

### ১. Parallel Filters (সমান্তরাল ফিল্টার)

কখনো কখনো একাধিক Filter একই ডেটার উপর **একসাথে** কাজ করতে পারে:

```
                          ┌──────────────┐
                     ┌───→│ Image Resize │───┐
                     │    └──────────────┘   │
  ┌──────────┐      │    ┌──────────────┐   │     ┌──────────┐
  │ Upload   │──────┼───→│  Virus Scan  │───┼────→│  Save    │
  │ Image    │      │    └──────────────┘   │     │  Result  │
  └──────────┘      │    ┌──────────────┐   │     └──────────┘
                     └───→│ Extract EXIF │───┘
                          └──────────────┘

তিনটি Filter একসাথে কাজ করে — Resize, Virus Scan, EXIF Extract
সবার কাজ শেষ হলে ফলাফল merge হয়ে Save হয়।
```

```javascript
// JavaScript — Parallel Filter Implementation
async function parallelPipeline(data, parallelFilters) {
    const results = await Promise.all(
        parallelFilters.map(filter => filter(data))
    );

    // সব Filter-এর ফলাফল merge করো
    return results.reduce((merged, result) => ({
        ...merged,
        ...result,
    }), {});
}

// ব্যবহার:
const result = await parallelPipeline(uploadedImage, [
    resizeFilter,
    virusScanFilter,
    extractExifFilter,
]);
```

### ২. Branching Pipes (শাখা পাইপ)

ডেটার ধরন অনুযায়ী বিভিন্ন পথে পাঠানো:

```
                              ┌──────────────┐    ┌──────────┐
                         ┌───→│ bKash Filter │───→│ bKash    │
                         │    └──────────────┘    │ Gateway  │
  ┌──────────┐    ┌─────┴─┐  ┌──────────────┐    ├──────────┤
  │ Payment  │───→│ Route │──→│ Nagad Filter │───→│ Nagad    │
  │ Request  │    │Filter │  └──────────────┘    │ Gateway  │
  └──────────┘    └─────┬─┘  ┌──────────────┐    ├──────────┤
                         └───→│ Card Filter  │───→│ SSL      │
                              └──────────────┘    │ Commerz  │
                                                  └──────────┘
```

```php
<?php

class PaymentRouterFilter
{
    private array $routes = [
        'bkash' => BkashPaymentFilter::class,
        'nagad' => NagadPaymentFilter::class,
        'card'  => CardPaymentFilter::class,
    ];

    public function handle($payment, Closure $next)
    {
        $method = $payment->payment_method;

        if (!isset($this->routes[$method])) {
            throw new \Exception("অজানা পেমেন্ট মেথড: {$method}");
        }

        $filter = new $this->routes[$method]();
        $payment = $filter->handle($payment, $next);

        return $next($payment);
    }
}
```

### ৩. Error Handling in Pipelines

```
  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
  │Filter A │────→│Filter B │────→│Filter C │────→│Filter D │
  └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
       │               │               │               │
       ▼               ▼               ▼               ▼
  ┌──────────────────────────────────────────────────────────┐
  │              Error Handler (সকল ত্রুটি এখানে আসে)        │
  │   - Log the error                                        │
  │   - Notify admin                                         │
  │   - Return meaningful error message                      │
  └──────────────────────────────────────────────────────────┘
```

```php
<?php

class SafePipeline
{
    private mixed $passable;
    private array $pipes = [];
    private ?Closure $errorHandler = null;

    public function send(mixed $passable): self
    {
        $this->passable = $passable;
        return $this;
    }

    public function through(array $pipes): self
    {
        $this->pipes = $pipes;
        return $this;
    }

    public function onError(Closure $handler): self
    {
        $this->errorHandler = $handler;
        return $this;
    }

    public function thenReturn(): mixed
    {
        $result = $this->passable;

        foreach ($this->pipes as $pipe) {
            try {
                $instance = is_string($pipe) ? new $pipe() : $pipe;
                $result = $instance->handle($result, fn($data) => $data);
            } catch (\Throwable $e) {
                if ($this->errorHandler) {
                    return ($this->errorHandler)($e, $result, $pipe);
                }
                throw $e;
            }
        }

        return $result;
    }
}

// ব্যবহার:
$result = (new SafePipeline())
    ->send($order)
    ->through([
        ValidateOrder::class,
        CalculateTotal::class,
        ProcessPayment::class,
    ])
    ->onError(function (\Throwable $e, $order, $failedPipe) {
        Log::error("Pipeline ব্যর্থ হয়েছে: {$e->getMessage()}", [
            'order_id'    => $order->id,
            'failed_pipe' => get_class($failedPipe),
        ]);
        return $order; // আংশিক প্রসেসড অর্ডার return
    })
    ->thenReturn();
```

---

## 🛒 Complete PHP Implementation — Daraz Product Import

Daraz-এর মতো একটি ই-কমার্স প্ল্যাটফর্মে seller CSV আপলোড করে product যোগ করেন। এই প্রসেসটি Pipe and Filter দিয়ে implement করা যায়:

```
CSV File → [Read] → [Validate] → [Transform] → [Deduplicate] → [Save] → [Report]
```

```php
<?php

// ──────────── Filter Interface ────────────
interface ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed;
}

// ──────────── Filter 1: CSV পড়া ────────────
class ReadCsvFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        // এই ক্ষেত্রে $products-এ file path থাকে
        $filePath = $products['file_path'];
        $rows = [];
        $handle = fopen($filePath, 'r');
        $headers = fgetcsv($handle);

        while (($row = fgetcsv($handle)) !== false) {
            $rows[] = array_combine($headers, $row);
        }
        fclose($handle);

        return $next($rows);
    }
}

// ──────────── Filter 2: ডেটা যাচাই ────────────
class ValidateProductFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        $valid = [];
        $errors = [];

        foreach ($products as $index => $product) {
            $productErrors = [];

            if (empty($product['name'])) {
                $productErrors[] = "পণ্যের নাম খালি";
            }
            if (!is_numeric($product['price']) || $product['price'] <= 0) {
                $productErrors[] = "মূল্য সঠিক নয়";
            }
            if (empty($product['sku'])) {
                $productErrors[] = "SKU খালি";
            }

            if (empty($productErrors)) {
                $valid[] = $product;
            } else {
                $errors[] = [
                    'row'    => $index + 2,
                    'errors' => $productErrors,
                ];
            }
        }

        // ত্রুটি তথ্য metadata হিসেবে সংরক্ষণ
        $valid['_metadata']['validation_errors'] = $errors;

        return $next($valid);
    }
}

// ──────────── Filter 3: ফরম্যাট রূপান্তর ────────────
class TransformProductFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        $metadata = $products['_metadata'] ?? [];
        unset($products['_metadata']);

        $transformed = array_map(function ($product) {
            return [
                'name'        => trim($product['name']),
                'slug'        => Str::slug($product['name']),
                'price'       => (float) $product['price'],
                'sku'         => strtoupper(trim($product['sku'])),
                'category_id' => $this->mapCategory($product['category'] ?? ''),
                'stock'       => (int) ($product['stock'] ?? 0),
                'status'      => 'pending_review',
                'created_at'  => now(),
            ];
        }, $products);

        $transformed['_metadata'] = $metadata;

        return $next($transformed);
    }

    private function mapCategory(string $categoryName): int
    {
        $map = [
            'electronics' => 1,
            'clothing'    => 2,
            'books'       => 3,
        ];
        return $map[strtolower($categoryName)] ?? 0;
    }
}

// ──────────── Filter 4: ডুপ্লিকেট চেক ────────────
class DeduplicateFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        $metadata = $products['_metadata'] ?? [];
        unset($products['_metadata']);

        $existingSkus = Product::whereIn(
            'sku',
            array_column($products, 'sku')
        )->pluck('sku')->toArray();

        $unique = array_filter($products, function ($product) use ($existingSkus) {
            return !in_array($product['sku'], $existingSkus);
        });

        $metadata['duplicates_removed'] = count($products) - count($unique);
        $unique['_metadata'] = $metadata;

        return $next($unique);
    }
}

// ──────────── Filter 5: ডাটাবেসে সংরক্ষণ ────────────
class SaveToDbFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        $metadata = $products['_metadata'] ?? [];
        unset($products['_metadata']);

        $saved = 0;
        foreach (array_chunk($products, 100) as $chunk) {
            Product::insert($chunk);
            $saved += count($chunk);
        }

        $metadata['saved_count'] = $saved;
        $products['_metadata'] = $metadata;

        return $next($products);
    }
}

// ──────────── Filter 6: রিপোর্ট তৈরি ────────────
class GenerateReportFilter implements ProductImportFilter
{
    public function handle(array $products, Closure $next): mixed
    {
        $metadata = $products['_metadata'] ?? [];

        $report = [
            'total_processed'     => $metadata['saved_count'] ?? 0,
            'validation_errors'   => count($metadata['validation_errors'] ?? []),
            'duplicates_removed'  => $metadata['duplicates_removed'] ?? 0,
            'completed_at'        => now()->toDateTimeString(),
        ];

        // Seller-কে email পাঠাও
        Mail::to($products['seller_email'] ?? '')
            ->send(new ProductImportReport($report));

        return $next($report);
    }
}

// ──────────── Pipeline চালানো ────────────
$report = app(Pipeline::class)
    ->send(['file_path' => storage_path('imports/products.csv')])
    ->through([
        ReadCsvFilter::class,
        ValidateProductFilter::class,
        TransformProductFilter::class,
        DeduplicateFilter::class,
        SaveToDbFilter::class,
        GenerateReportFilter::class,
    ])
    ->thenReturn();

echo "আমদানি সম্পন্ন! মোট {$report['total_processed']} পণ্য সংরক্ষিত হয়েছে।";
```

---

## 🖥️ Complete JS Implementation — User Registration Pipeline

Node.js-এ class-based এবং functional দুইভাবেই pipeline তৈরি করা যায়।

### Functional Approach (ফাংশনাল পদ্ধতি)

```javascript
// ──────────── Pipeline Builder (Functional) ────────────
const pipe = (...filters) => async (input) => {
    let result = input;
    for (const filter of filters) {
        result = await filter(result);
    }
    return result;
};

// ──────────── Filters ────────────

const validateInput = async (data) => {
    const errors = [];

    if (!data.email || !data.email.includes('@')) {
        errors.push('সঠিক ইমেইল দিন');
    }
    if (!data.password || data.password.length < 8) {
        errors.push('পাসওয়ার্ড কমপক্ষে ৮ অক্ষরের হতে হবে');
    }
    if (!data.phone || !/^01[3-9]\d{8}$/.test(data.phone)) {
        errors.push('সঠিক বাংলাদেশি মোবাইল নম্বর দিন');
    }

    if (errors.length > 0) {
        throw new Error(`যাচাইকরণ ব্যর্থ: ${errors.join(', ')}`);
    }

    return { ...data, validated: true };
};

const hashPassword = async (data) => {
    const bcrypt = require('bcrypt');
    const hashedPassword = await bcrypt.hash(data.password, 12);
    return {
        ...data,
        password: hashedPassword,
        originalPassword: undefined, // raw password মুছে দাও
    };
};

const createUser = async (data) => {
    const user = await User.create({
        name: data.name,
        email: data.email,
        phone: data.phone,
        password: data.password,
    });
    return { ...data, userId: user.id, user };
};

const sendWelcomeEmail = async (data) => {
    await emailService.send({
        to: data.email,
        subject: 'স্বাগতম! আপনার অ্যাকাউন্ট তৈরি হয়েছে',
        template: 'welcome',
        context: { name: data.name },
    });
    return { ...data, welcomeEmailSent: true };
};

const logActivity = async (data) => {
    await ActivityLog.create({
        user_id: data.userId,
        action: 'USER_REGISTERED',
        metadata: {
            email: data.email,
            phone: data.phone,
            timestamp: new Date().toISOString(),
        },
    });
    return { ...data, logged: true };
};

// ──────────── Pipeline চালানো ────────────
const registerUser = pipe(
    validateInput,
    hashPassword,
    createUser,
    sendWelcomeEmail,
    logActivity
);

// ব্যবহার:
try {
    const result = await registerUser({
        name: 'রহিম উদ্দিন',
        email: 'rahim@example.com',
        phone: '01712345678',
        password: 'securePass123',
    });
    console.log('রেজিস্ট্রেশন সফল! User ID:', result.userId);
} catch (error) {
    console.error('রেজিস্ট্রেশন ব্যর্থ:', error.message);
}
```

### Class-Based Approach (ক্লাস ভিত্তিক পদ্ধতি)

```javascript
class Pipeline {
    #passable = null;
    #pipes = [];

    send(passable) {
        this.#passable = passable;
        return this;
    }

    through(pipes) {
        this.#pipes = pipes;
        return this;
    }

    async thenReturn() {
        let result = this.#passable;

        for (const Pipe of this.#pipes) {
            if (typeof Pipe === 'function' && !Pipe.prototype?.handle) {
                // Plain function filter
                result = await Pipe(result);
            } else {
                // Class-based filter
                const instance = new Pipe();
                result = await instance.handle(result);
            }
        }

        return result;
    }
}

// ──────────── Filter Classes ────────────

class ValidatePhoneFilter {
    async handle(data) {
        // Grameenphone, Robi, Banglalink, Teletalk — সব নম্বর চেক
        const bdPhoneRegex = /^(?:\+88)?01[3-9]\d{8}$/;

        if (!bdPhoneRegex.test(data.phone)) {
            throw new Error('সঠিক বাংলাদেশি মোবাইল নম্বর দিন');
        }

        return { ...data, phoneValidated: true };
    }
}

class NormalizeDataFilter {
    async handle(data) {
        return {
            ...data,
            email: data.email.toLowerCase().trim(),
            name: data.name.trim(),
            phone: data.phone.replace(/^\+88/, ''),
        };
    }
}

// ব্যবহার:
const result = await new Pipeline()
    .send({ name: ' রহিম ', email: ' RAHIM@EXAMPLE.COM ', phone: '+8801712345678' })
    .through([
        NormalizeDataFilter,
        ValidatePhoneFilter,
    ])
    .thenReturn();
```

---

## ❌ Bad vs ✅ Good Patterns

### ❌ Bad: Filter-গুলো একে অপরের internal state-এ নির্ভরশীল

```php
<?php
// ❌ খারাপ — Filter অন্য Filter-এর internal state access করে
class PriceCalculator
{
    public float $tax = 0;        // public state!
    public float $discount = 0;   // অন্য Filter এটা access করে!

    public function handle($order, Closure $next)
    {
        $this->tax = $order->subtotal * 0.15;
        $this->discount = $order->subtotal > 5000 ? 500 : 0;
        return $next($order);
    }
}

class InvoiceGenerator
{
    public function handle($order, Closure $next)
    {
        // ❌ অন্য Filter-এর instance-এ নির্ভরশীল!
        $calculator = app(PriceCalculator::class);
        $order->tax = $calculator->tax;         // Tight coupling!
        $order->discount = $calculator->discount; // Tight coupling!
        return $next($order);
    }
}
```

### ✅ Good: Filter-গুলো সম্পূর্ণ স্বতন্ত্র — শুধু data দিয়ে যোগাযোগ

```php
<?php
// ✅ ভালো — প্রতিটি Filter শুধু ডেটা transform করে
class PriceCalculator
{
    public function handle($order, Closure $next)
    {
        // ডেটার মধ্যেই সব তথ্য রাখো
        $order->tax = $order->subtotal * 0.15;
        $order->discount = $order->subtotal > 5000 ? 500 : 0;
        $order->total = $order->subtotal + $order->tax - $order->discount;

        return $next($order); // পরবর্তী Filter ডেটা থেকেই সব পাবে
    }
}

class InvoiceGenerator
{
    public function handle($order, Closure $next)
    {
        // ✅ শুধু $order ডেটা ব্যবহার করে — অন্য Filter-এর কথা জানে না
        $order->invoice = [
            'subtotal' => $order->subtotal,
            'tax'      => $order->tax,
            'discount' => $order->discount,
            'total'    => $order->total,
        ];

        return $next($order);
    }
}
```

### ❌ Bad: একটি বিশাল monolithic function সব কাজ করে

```javascript
// ❌ খারাপ — একটি ফাংশনে সব কিছু
async function processOrder(orderData) {
    // যাচাই
    if (!orderData.items?.length) throw new Error('No items');
    if (!orderData.customerId) throw new Error('No customer');

    // মূল্য গণনা
    let total = orderData.items.reduce((sum, i) => sum + i.price * i.qty, 0);

    // ডিসকাউন্ট
    if (orderData.coupon) {
        const coupon = await Coupon.findOne({ code: orderData.coupon });
        if (coupon) total -= total * (coupon.percent / 100);
    }

    // ভ্যাট
    total += total * 0.15;

    // শিপিং
    total += orderData.isDhaka ? 60 : 120;

    // পেমেন্ট
    const payment = await gateway.charge(total);
    if (!payment.success) throw new Error('Payment failed');

    // সংরক্ষণ
    const order = await Order.create({ ...orderData, total });

    // নোটিফিকেশন
    await sendSMS(orderData.phone, `অর্ডার #${order.id} সফল!`);
    await sendEmail(orderData.email, 'Order Confirmed', order);

    return order;
}
// সমস্যা: test করা কঠিন, পরিবর্তন করলে সব ভেঙে যেতে পারে,
// কোনো ধাপ reuse করা যায় না
```

### ✅ Good: ছোট ছোট Filter-এর Pipeline

```javascript
// ✅ ভালো — প্রতিটি ধাপ আলাদা, testable, reusable
const processOrder = pipe(
    validateOrder,        // শুধু যাচাই করে
    calculateSubtotal,    // শুধু মূল্য গণনা করে
    applyDiscount,        // শুধু ডিসকাউন্ট প্রয়োগ করে
    calculateVAT,         // শুধু ভ্যাট হিসাব করে
    addShipping,          // শুধু শিপিং যোগ করে
    processPayment,       // শুধু পেমেন্ট প্রসেস করে
    saveOrder,            // শুধু ডাটাবেসে সংরক্ষণ করে
    sendNotifications     // শুধু নোটিফিকেশন পাঠায়
);

// প্রতিটি Filter আলাদাভাবে test করা যায়:
// test('calculateVAT adds 15% tax', () => { ... })
// test('applyDiscount reduces price', () => { ... })
```

---

## 🤔 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন

```
┌────────────────────────────────────────────────────────────────────┐
│  ✅ Pipe and Filter ব্যবহার করুন যখন:                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ১. ডেটা sequential ভাবে process করতে হয়                          │
│     → CSV import, ETL, data migration                             │
│                                                                    │
│  ২. প্রতিটি processing step স্বতন্ত্র                               │
│     → প্রতিটি ধাপ আলাদাভাবে test/deploy করা যায়                    │
│                                                                    │
│  ৩. ধাপ যোগ/সরানো/পুনর্বিন্যাস করার flexibility দরকার              │
│     → আজ ৫টা ধাপ, কাল ৭টা — সহজেই পরিবর্তনযোগ্য                  │
│                                                                    │
│  ৪. একই Filter বিভিন্ন pipeline-এ reuse করতে চান                  │
│     → ValidateEmail filter ১০টা pipeline-এ ব্যবহার করুন             │
│                                                                    │
│  ৫. Middleware chain দরকার                                         │
│     → HTTP request processing, authorization layers                │
│                                                                    │
│  ৬. জটিল business logic কে ছোট ছোট ধাপে ভাঙতে চান                │
│     → Order processing, payment flow, registration                 │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### ❌ কখন ব্যবহার করবেন না

```
┌────────────────────────────────────────────────────────────────────┐
│  ❌ Pipe and Filter ব্যবহার করবেন না যখন:                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ১. ধাপগুলো একে অপরের উপর heavily নির্ভরশীল                       │
│     → Filter A-র result ছাড়া Filter C কাজ করতে পারবে না,          │
│       আবার Filter B-র result-ও লাগবে — জটিল dependency graph      │
│                                                                    │
│  ২. Shared mutable state দরকার                                    │
│     → সব Filter-কে একই database connection/transaction             │
│       share করতে হবে                                               │
│                                                                    │
│  ৩. অত্যন্ত সাধারণ operation                                       │
│     → ২-৩ লাইনের কাজের জন্য pipeline overhead যুক্তিসঙ্গত নয়       │
│                                                                    │
│  ৪. Real-time interactive processing                               │
│     → যেখানে user-এর সাথে ক্রমাগত interaction দরকার                │
│     → Pipeline একবার শুরু হলে শেষ পর্যন্ত চলে                      │
│                                                                    │
│  ৫. যখন performance-ই সর্বোচ্চ অগ্রাধিকার                          │
│     → প্রতিটি Pipe-এ data copy/transfer হয়, latency যোগ হয়         │
│     → Ultra-low-latency system-এ overhead সমস্যা হতে পারে          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Decision Guide (সিদ্ধান্ত গাইড)

```
আপনার সমস্যা কি sequential data processing?
    │
    ├── হ্যাঁ → ধাপগুলো কি স্বতন্ত্র?
    │           │
    │           ├── হ্যাঁ → ধাপ কি ৩টির বেশি?
    │           │           │
    │           │           ├── হ্যাঁ → ✅ Pipe and Filter ব্যবহার করুন!
    │           │           │
    │           │           └── না → সাধারণ function call-ই যথেষ্ট হতে পারে
    │           │
    │           └── না → ধাপগুলো কি tightly coupled?
    │                       │
    │                       ├── হ্যাঁ → ❌ অন্য প্যাটার্ন বিবেচনা করুন
    │                       │
    │                       └── আংশিক → কিছু ধাপ group করে Filter বানান
    │
    └── না → ❌ এই প্যাটার্ন আপনার জন্য নয়
```

---

## 🔗 Pipe and Filter vs অন্যান্য প্যাটার্ন

```
┌──────────────────┬────────────────────┬──────────────────┬──────────────────┐
│                  │ Pipe and Filter    │ Chain of         │ Middleware       │
│     বৈশিষ্ট্য    │                    │ Responsibility   │                  │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ উদ্দেশ্য         │ ডেটা transform     │ Request handle   │ Request/Response │
│                  │ করা                │ করা              │ intercept করা    │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ সব handler কি   │ ✅ হ্যাঁ — সবাই     │ ❌ না — একজন     │ ✅ হ্যাঁ — সবাই   │
│ execute হয়?     │ execute হয়        │ handle করলেই     │ execute হয়      │
│                  │                    │ থেমে যায়         │                  │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ Data flow        │ একমুখী            │ একমুখী           │ দ্বিমুখী          │
│                  │ Input → Output     │ Request →        │ Request →        │
│                  │                    │                  │ ← Response       │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ ডেটা পরিবর্তন   │ ✅ প্রতিটি ধাপে    │ ❌ সাধারণত না    │ ✅ পরিবর্তন      │
│                  │ transform হয়      │                  │ করতে পারে        │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ বাংলাদেশি        │ bKash payment      │ Customer support │ Laravel/Express  │
│ উদাহরণ          │ processing         │ escalation       │ HTTP middleware  │
│                  │ pipeline           │ (agent→supervisor│                  │
│                  │                    │ →manager)        │                  │
├──────────────────┼────────────────────┼──────────────────┼──────────────────┤
│ কখন ব্যবহার     │ ডেটা ধাপে ধাপে    │ কে handle করবে   │ Cross-cutting    │
│ করবেন           │ process করতে       │ জানা নেই         │ concerns যোগ     │
│                  │                    │                  │ করতে (auth,      │
│                  │                    │                  │ logging)         │
└──────────────────┴────────────────────┴──────────────────┴──────────────────┘
```

### Pipe and Filter vs Strategy Pattern

```
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Pipe and Filter:  সব Filter একের পর এক চলে                      │
│                                                                   │
│    [A] ──→ [B] ──→ [C] ──→ [D]   (সবাই চলবে, ক্রমানুসারে)        │
│                                                                   │
│  Strategy:  একটি মাত্র strategy select হয়                         │
│                                                                   │
│            ┌── [Strategy A]                                       │
│    Context─┼── [Strategy B]  ← এদের মধ্যে একটি select হবে         │
│            └── [Strategy C]                                       │
│                                                                   │
│  পার্থক্য:                                                         │
│  - Pipe and Filter = "সব ধাপ চলবে, ক্রমে" (composition)           │
│  - Strategy = "একটি বেছে নাও" (selection)                         │
│                                                                   │
│  Pathao-র উদাহরণ:                                                 │
│  - Payment Pipeline (Pipe & Filter):                              │
│    Validate → FraudCheck → ProcessPayment → Notify                │
│                                                                   │
│  - Payment Method Selection (Strategy):                           │
│    bKash / Nagad / Card — একটি বেছে নিয়ে সেটা দিয়ে pay করো       │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

```
╔═══════════════════════════════════════════════════════════════════╗
║              PIPE AND FILTER — মনে রাখার বিষয়গুলো               ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  ১. প্রতিটি Filter = একটি কাজ (Single Responsibility)             ║
║  ২. Filter-গুলো স্বতন্ত্র — একে অপরের কথা জানে না                 ║
║  ৩. Pipe = শুধু ডেটা পাস করে, কোনো logic নেই                     ║
║  ৪. ডেটা একমুখী প্রবাহিত হয়: Source → Filters → Sink             ║
║  ৫. নতুন Filter যোগ/সরানো অত্যন্ত সহজ                           ║
║  ৬. প্রতিটি Filter আলাদাভাবে test করা যায়                         ║
║  ৭. Laravel Pipeline, Express Middleware — বাস্তব উদাহরণ          ║
║  ৮. Unix pipe (`|`) — এই প্যাটার্নের আদি রূপ                     ║
║                                                                   ║
║  মনে রাখুন: পানি বিশুদ্ধকরণ সিস্টেমের মতো —                      ║
║  প্রতিটি ফিল্টার একটি নির্দিষ্ট দূষণকারী দূর করে,                 ║
║  এবং পরিষ্কার পানি পরের ফিল্টারে যায়।                            ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```
