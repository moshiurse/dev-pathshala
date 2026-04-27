# 🔄 রিফ্যাক্টরিং কৌশল (Refactoring Techniques)

> _"কোড লেখা শিল্প, কিন্তু কোড পরিষ্কার করা মহাশিল্প।"_
> — পরিচ্ছন্ন কোডের দর্শন

---

## 📖 সংজ্ঞা

**রিফ্যাক্টরিং** হলো বিদ্যমান কোডের **বাহ্যিক আচরণ অপরিবর্তিত** রেখে তার **অভ্যন্তরীণ কাঠামো উন্নত** করার প্রক্রিয়া। এটি কোনো নতুন ফিচার যোগ করে না — বরং কোডকে আরও পরিষ্কার, পাঠযোগ্য এবং রক্ষণাবেক্ষণযোগ্য করে তোলে।

```
রিফ্যাক্টরিং = একই ফলাফল + উন্নত কাঠামো

┌─────────────────────────────────────────────────────────┐
│                    রিফ্যাক্টরিং                          │
│                                                         │
│   আগের কোড ──────────────► পরের কোড                    │
│   (জটিল, অগোছালো)          (পরিষ্কার, সুসংগঠিত)        │
│                                                         │
│   ┌───────────┐            ┌───────────┐                │
│   │ ইনপুট: ৫  │ ──► ১০    │ ইনপুট: ৫  │ ──► ১০         │
│   └───────────┘            └───────────┘                │
│   আচরণ একই থাকে, গঠন বদলায়                             │
└─────────────────────────────────────────────────────────┘
```

### কেন রিফ্যাক্টরিং দরকার?

| সমস্যা | রিফ্যাক্টরিং-এর ফলে সমাধান |
|---------|---------------------------|
| কোড বোঝা কঠিন | পাঠযোগ্যতা বাড়ে |
| বাগ খুঁজে পেতে সময় লাগে | ডিবাগিং সহজ হয় |
| নতুন ফিচার যোগ করতে ভয় | পরিবর্তন নিরাপদ হয় |
| কোড ডুপ্লিকেশন | পুনঃব্যবহারযোগ্যতা বাড়ে |
| টেস্ট লেখা কঠিন | টেস্টযোগ্যতা বাড়ে |

---

## 🏠 বাস্তব জীবনের উদাহরণ

কল্পনা করুন আপনার একটি **পুরনো বাড়ি** আছে। বাড়িটা দাঁড়িয়ে আছে, মানুষ থাকছে (কাজ করছে)। কিন্তু ভেতরের তারের সংযোগ জটবদ্ধ, পাইপ পুরনো, রুমের বিন্যাস অদ্ভুত।

**রিফ্যাক্টরিং** হলো বাড়ি ভাঙা ছাড়াই ভেতরের তার ঠিক করা, পাইপ বদলানো, রুম পুনর্বিন্যাস করা — যাতে বাসিন্দাদের কোনো অসুবিধা না হয়, কিন্তু বাড়ি আরও ভালো হয়।

```
🏚️ আগের বাড়ি                    🏡 পরের বাড়ি
┌──────────────────┐             ┌──────────────────┐
│ জটবদ্ধ তার ⚡    │             │ পরিষ্কার তার ⚡   │
│ পুরনো পাইপ 🔧   │  ──রিফ্যাক──►│ নতুন পাইপ 🔧     │
│ অগোছালো রুম 🚪  │             │ সুসজ্জিত রুম 🚪  │
│                  │             │                  │
│ বাসিন্দা: সুখী ✅ │             │ বাসিন্দা: সুখী ✅  │
└──────────────────┘             └──────────────────┘
        বাহ্যিক আচরণ একই থাকে!
```

---

## ১. 🔬 Extract Method / ফাংশন বের করা

### ধারণা

একটি বড় ফাংশনের ভেতরে যদি কোনো কোড ব্লক **একটি নির্দিষ্ট কাজ** করে, তাহলে সেটাকে আলাদা ফাংশনে বের করে আনুন। নতুন ফাংশনের নামটি হবে **সেই কাজের বর্ণনা**।

### কখন Extract করবেন — চিহ্ন ও নিয়ম

```
Extract Method-এর চিহ্ন:
┌─────────────────────────────────────────────┐
│ ✦ ফাংশন ২০+ লাইনের বেশি                     │
│ ✦ কমেন্ট দিয়ে কোড ব্লক আলাদা করা হচ্ছে      │
│ ✦ একই কোড একাধিক জায়গায় আছে               │
│ ✦ ফাংশনের নাম দেখে বোঝা যায় না কী করে      │
│ ✦ একটি ফাংশনে একাধিক abstraction level      │
│ ✦ নেস্টেড if/loop ৩ স্তরের বেশি               │
└─────────────────────────────────────────────┘
```

### ❌ আগের কোড (PHP) — পেমেন্ট প্রসেসিং

```php
<?php
// ❌ খারাপ: একটি বিশাল ফাংশন যা সব কিছু করে
class PaymentProcessor
{
    public function processPayment($orderId, $amount, $cardNumber, $cvv, $expiry)
    {
        // কার্ড যাচাই করা
        if (strlen($cardNumber) !== 16) {
            throw new Exception("কার্ড নম্বর অবৈধ");
        }
        if (strlen($cvv) !== 3) {
            throw new Exception("CVV অবৈধ");
        }
        $expiryParts = explode('/', $expiry);
        if (count($expiryParts) !== 2) {
            throw new Exception("মেয়াদ অবৈধ");
        }
        $month = (int) $expiryParts[0];
        $year = (int) $expiryParts[1];
        if ($month < 1 || $month > 12) {
            throw new Exception("মেয়াদের মাস অবৈধ");
        }
        if ($year < date('Y')) {
            throw new Exception("কার্ডের মেয়াদ শেষ");
        }

        // ডিসকাউন্ট হিসাব করা
        $discount = 0;
        if ($amount > 10000) {
            $discount = $amount * 0.1;
        } elseif ($amount > 5000) {
            $discount = $amount * 0.05;
        } elseif ($amount > 1000) {
            $discount = $amount * 0.02;
        }
        $finalAmount = $amount - $discount;

        // ট্যাক্স হিসাব করা
        $tax = $finalAmount * 0.15;
        $totalAmount = $finalAmount + $tax;

        // পেমেন্ট গেটওয়ে কল করা
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, "https://payment-gateway.com/charge");
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
            'amount' => $totalAmount,
            'card' => $cardNumber,
            'cvv' => $cvv,
        ]));
        $response = curl_exec($ch);
        curl_close($ch);

        // রেসপন্স প্রসেস করা
        $result = json_decode($response, true);
        if ($result['status'] !== 'success') {
            // ব্যর্থতা লগ করা
            $logFile = fopen('payment_errors.log', 'a');
            fwrite($logFile, date('Y-m-d H:i:s') . " - অর্ডার: $orderId - ব্যর্থ\n");
            fclose($logFile);
            throw new Exception("পেমেন্ট ব্যর্থ হয়েছে");
        }

        // সফলতা লগ করা
        $logFile = fopen('payment_success.log', 'a');
        fwrite($logFile, date('Y-m-d H:i:s') . " - অর্ডার: $orderId - সফল - পরিমাণ: $totalAmount\n");
        fclose($logFile);

        // ইমেইল পাঠানো
        mail("customer@example.com", "পেমেন্ট সফল",
            "আপনার $totalAmount টাকা পেমেন্ট সফল হয়েছে। অর্ডার: $orderId");

        return [
            'status' => 'সফল',
            'orderId' => $orderId,
            'amount' => $totalAmount,
            'transactionId' => $result['transaction_id'],
        ];
    }
}
```

### ✅ পরের কোড (PHP) — Extract Method প্রয়োগ

```php
<?php
// ✅ ভালো: প্রতিটি দায়িত্ব আলাদা মেথডে
class PaymentProcessor
{
    /**
     * মূল পেমেন্ট প্রসেসিং — পরিষ্কার ও পাঠযোগ্য
     */
    public function processPayment(string $orderId, float $amount, string $cardNumber, string $cvv, string $expiry): array
    {
        $this->validateCard($cardNumber, $cvv, $expiry);

        $finalAmount = $this->calculateFinalAmount($amount);
        $gatewayResponse = $this->chargePaymentGateway($finalAmount, $cardNumber, $cvv);

        $this->handleGatewayResponse($gatewayResponse, $orderId);
        $this->logSuccess($orderId, $finalAmount);
        $this->sendConfirmationEmail($orderId, $finalAmount);

        return $this->buildSuccessResponse($orderId, $finalAmount, $gatewayResponse);
    }

    // কার্ড যাচাইকরণ — শুধুমাত্র যাচাইয়ের দায়িত্ব
    private function validateCard(string $cardNumber, string $cvv, string $expiry): void
    {
        $this->validateCardNumber($cardNumber);
        $this->validateCvv($cvv);
        $this->validateExpiry($expiry);
    }

    private function validateCardNumber(string $cardNumber): void
    {
        if (strlen($cardNumber) !== 16) {
            throw new Exception("কার্ড নম্বর অবৈধ");
        }
    }

    private function validateCvv(string $cvv): void
    {
        if (strlen($cvv) !== 3) {
            throw new Exception("CVV অবৈধ");
        }
    }

    private function validateExpiry(string $expiry): void
    {
        $parts = explode('/', $expiry);
        if (count($parts) !== 2) {
            throw new Exception("মেয়াদ অবৈধ");
        }

        $month = (int) $parts[0];
        $year = (int) $parts[1];

        if ($month < 1 || $month > 12) {
            throw new Exception("মেয়াদের মাস অবৈধ");
        }
        if ($year < (int) date('Y')) {
            throw new Exception("কার্ডের মেয়াদ শেষ");
        }
    }

    // মোট অর্থ হিসাব — ডিসকাউন্ট + ট্যাক্স
    private function calculateFinalAmount(float $amount): float
    {
        $discount = $this->calculateDiscount($amount);
        $afterDiscount = $amount - $discount;
        $tax = $this->calculateTax($afterDiscount);

        return $afterDiscount + $tax;
    }

    private function calculateDiscount(float $amount): float
    {
        if ($amount > 10000) return $amount * 0.10;
        if ($amount > 5000)  return $amount * 0.05;
        if ($amount > 1000)  return $amount * 0.02;
        return 0;
    }

    private function calculateTax(float $amount): float
    {
        return $amount * 0.15;
    }

    // গেটওয়ে থেকে চার্জ করা
    private function chargePaymentGateway(float $amount, string $card, string $cvv): array
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, "https://payment-gateway.com/charge");
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
            'amount' => $amount,
            'card' => $card,
            'cvv' => $cvv,
        ]));
        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }

    // গেটওয়ে রেসপন্স হ্যান্ডেল করা
    private function handleGatewayResponse(array $response, string $orderId): void
    {
        if ($response['status'] !== 'success') {
            $this->logFailure($orderId);
            throw new Exception("পেমেন্ট ব্যর্থ হয়েছে");
        }
    }

    private function logSuccess(string $orderId, float $amount): void
    {
        $message = date('Y-m-d H:i:s') . " - অর্ডার: $orderId - সফল - পরিমাণ: $amount\n";
        file_put_contents('payment_success.log', $message, FILE_APPEND);
    }

    private function logFailure(string $orderId): void
    {
        $message = date('Y-m-d H:i:s') . " - অর্ডার: $orderId - ব্যর্থ\n";
        file_put_contents('payment_errors.log', $message, FILE_APPEND);
    }

    private function sendConfirmationEmail(string $orderId, float $amount): void
    {
        mail("customer@example.com", "পেমেন্ট সফল",
            "আপনার $amount টাকা পেমেন্ট সফল হয়েছে। অর্ডার: $orderId");
    }

    private function buildSuccessResponse(string $orderId, float $amount, array $gateway): array
    {
        return [
            'status' => 'সফল',
            'orderId' => $orderId,
            'amount' => $amount,
            'transactionId' => $gateway['transaction_id'],
        ];
    }
}
```

### ✅ পরের কোড (JavaScript) — Extract Method

```javascript
// ✅ জাভাস্ক্রিপ্ট — Extract Method প্রয়োগ
class PaymentProcessor {
    // মূল মেথড — প্রতিটি ধাপ পরিষ্কার
    async processPayment(orderId, amount, cardInfo) {
        this.#validateCard(cardInfo);

        const finalAmount = this.#calculateFinalAmount(amount);
        const gatewayResponse = await this.#chargeGateway(finalAmount, cardInfo);

        this.#ensurePaymentSuccess(gatewayResponse, orderId);
        this.#logSuccess(orderId, finalAmount);
        await this.#sendConfirmation(orderId, finalAmount);

        return this.#buildResponse(orderId, finalAmount, gatewayResponse);
    }

    // কার্ড যাচাই
    #validateCard({ number, cvv, expiry }) {
        if (number.length !== 16) throw new Error("কার্ড নম্বর অবৈধ");
        if (cvv.length !== 3) throw new Error("CVV অবৈধ");

        const [month, year] = expiry.split('/').map(Number);
        if (month < 1 || month > 12) throw new Error("মেয়াদের মাস অবৈধ");
        if (year < new Date().getFullYear()) throw new Error("কার্ডের মেয়াদ শেষ");
    }

    // চূড়ান্ত অর্থ = মূল - ডিসকাউন্ট + ট্যাক্স
    #calculateFinalAmount(amount) {
        const discount = this.#calculateDiscount(amount);
        const afterDiscount = amount - discount;
        const tax = afterDiscount * 0.15;
        return afterDiscount + tax;
    }

    #calculateDiscount(amount) {
        if (amount > 10000) return amount * 0.10;
        if (amount > 5000) return amount * 0.05;
        if (amount > 1000) return amount * 0.02;
        return 0;
    }

    // গেটওয়ে API কল
    async #chargeGateway(amount, cardInfo) {
        const response = await fetch("https://payment-gateway.com/charge", {
            method: "POST",
            body: JSON.stringify({ amount, card: cardInfo.number, cvv: cardInfo.cvv }),
        });
        return response.json();
    }

    #ensurePaymentSuccess(response, orderId) {
        if (response.status !== "success") {
            this.#logFailure(orderId);
            throw new Error("পেমেন্ট ব্যর্থ হয়েছে");
        }
    }

    #logSuccess(orderId, amount) {
        console.log(`[সফল] অর্ডার: ${orderId}, পরিমাণ: ${amount}`);
    }

    #logFailure(orderId) {
        console.error(`[ব্যর্থ] অর্ডার: ${orderId}`);
    }

    async #sendConfirmation(orderId, amount) {
        // ইমেইল সার্ভিস কল
        console.log(`ইমেইল পাঠানো হচ্ছে — অর্ডার: ${orderId}`);
    }

    #buildResponse(orderId, amount, gateway) {
        return {
            status: "সফল",
            orderId,
            amount,
            transactionId: gateway.transaction_id,
        };
    }
}
```

---

## ২. ✏️ Rename Method / Variable — অর্থবহ নামকরণ

### ধারণা

কোডে ব্যবহৃত ভ্যারিয়েবল, ফাংশন এবং ক্লাসের নাম যদি **তাদের উদ্দেশ্য প্রকাশ না করে**, তাহলে সেগুলো পরিবর্তন করুন। ভালো নাম হলো কোডের সেরা ডকুমেন্টেশন।

```
নামকরণের নিয়ম:
┌──────────────────────────────────────────────────────┐
│ ১. নাম পড়ে বোঝা যাবে কী করে / কী ধারণ করে          │
│ ২. সংক্ষেপ পরিহার করুন (usr → user)                 │
│ ৩. বুলিয়ান: is/has/can/should দিয়ে শুরু              │
│ ৪. ফাংশন: ক্রিয়াপদ দিয়ে শুরু (get, calculate, send) │
│ ৫. ক্লাস: বিশেষ্য (Invoice, UserRepository)          │
│ ৬. কনটেক্সট থেকে পুনরাবৃত্তি বাদ দিন                │
└──────────────────────────────────────────────────────┘
```

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: নামগুলো কিছুই বলে না
function calc($d, $t) {
    $r = 0;
    $tmp = [];
    foreach ($d as $x) {
        $v = $x['p'] * $x['q'];
        if ($v > 500) {
            $v = $v * 0.9; // কী হচ্ছে বোঝা যায় না
        }
        $tmp[] = $v;
        $r += $v;
    }
    $tx = $r * $t;
    $f = $r + $tx;
    return ['i' => $tmp, 's' => $r, 't' => $tx, 'f' => $f];
}
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: প্রতিটি নাম তার উদ্দেশ্য প্রকাশ করে
function calculateInvoiceTotal(array $lineItems, float $taxRate): array
{
    $subtotal = 0;
    $itemTotals = [];

    foreach ($lineItems as $item) {
        $itemTotal = $item['price'] * $item['quantity'];

        $hasBulkDiscount = $itemTotal > 500;
        if ($hasBulkDiscount) {
            $itemTotal = $itemTotal * 0.9; // ১০% বাল্ক ডিসকাউন্ট
        }

        $itemTotals[] = $itemTotal;
        $subtotal += $itemTotal;
    }

    $taxAmount = $subtotal * $taxRate;
    $grandTotal = $subtotal + $taxAmount;

    return [
        'itemTotals'  => $itemTotals,
        'subtotal'    => $subtotal,
        'taxAmount'   => $taxAmount,
        'grandTotal'  => $grandTotal,
    ];
}
```

### ❌ আগের কোড (JavaScript)

```javascript
// ❌ খারাপ: অর্থহীন নাম
function proc(u) {
    let a = u.fn + " " + u.ln;
    let b = new Date().getFullYear() - u.by;
    let c = b >= 18;
    let d = u.e.includes("@");
    if (c && d) {
        return { n: a, a: b, s: "ok" };
    }
    return { s: "fail" };
}
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ ভালো: নাম পড়েই বোঝা যায় কী হচ্ছে
function processUserRegistration(userInput) {
    const fullName = `${userInput.firstName} ${userInput.lastName}`;
    const age = new Date().getFullYear() - userInput.birthYear;

    const isEligibleAge = age >= 18;
    const hasValidEmail = userInput.email.includes("@");

    if (isEligibleAge && hasValidEmail) {
        return { name: fullName, age, status: "অনুমোদিত" };
    }

    return { status: "প্রত্যাখ্যাত" };
}
```

### IDE রিফ্যাক্টরিং টুলস

| IDE | শর্টকাট | বিবরণ |
|-----|---------|--------|
| VS Code | `F2` | Rename Symbol — সব রেফারেন্স বদলায় |
| PhpStorm | `Shift + F6` | Rename — পুরো প্রজেক্টে বদলায় |
| WebStorm | `Shift + F6` | JavaScript/TypeScript rename |
| Vim | `:s/old/new/g` | ম্যানুয়াল (সতর্কতা দরকার) |

---

## ৩. 🚚 Move Method / Move Field — সঠিক জায়গায় স্থানান্তর

### ধারণা

যখন একটি মেথড তার নিজের ক্লাসের চেয়ে **অন্য ক্লাসের ডেটা বেশি ব্যবহার** করে, তখন সেই মেথডকে যথাযথ ক্লাসে সরিয়ে নিন। এটি **Feature Envy** কোড স্মেল-এর সমাধান।

```
Feature Envy চিহ্নিতকরণ:
┌────────────────────────────────────────────────────┐
│                                                    │
│   ক্লাস A                     ক্লাস B              │
│   ┌────────────┐              ┌────────────┐       │
│   │ মেথড X()   │──────────►  │ ডেটা ১     │       │
│   │            │──────────►  │ ডেটা ২     │       │
│   │            │──────────►  │ ডেটা ৩     │       │
│   └────────────┘              └────────────┘       │
│                                                    │
│   মেথড X ক্লাস B-র ডেটা বেশি ব্যবহার করে           │
│   → মেথড X-কে ক্লাস B-তে সরান!                    │
└────────────────────────────────────────────────────┘
```

### ❌ আগের কোড (PHP) — Address যাচাই Order-এ আছে

```php
<?php
// ❌ খারাপ: Order ক্লাসে Address যাচাইকরণ — Feature Envy!
class Order
{
    private string $customerName;
    private Address $shippingAddress;

    // এই মেথড Address-এর ডেটা বেশি ব্যবহার করছে
    public function validateShippingAddress(): bool
    {
        $address = $this->shippingAddress;

        if (empty($address->street)) return false;
        if (empty($address->city)) return false;
        if (empty($address->zipCode)) return false;
        if (strlen($address->zipCode) !== 5) return false;
        if (!in_array($address->country, ['BD', 'IN', 'US'])) return false;

        // বাংলাদেশের জন্য বিশেষ যাচাই
        if ($address->country === 'BD') {
            if (!preg_match('/^\d{4}$/', $address->zipCode)) return false;
        }

        return true;
    }
}

class Address
{
    public string $street;
    public string $city;
    public string $zipCode;
    public string $country;
}
```

### ✅ পরের কোড (PHP) — Address যাচাই Address-এ সরানো হয়েছে

```php
<?php
// ✅ ভালো: যাচাইকরণ সঠিক ক্লাসে
class Address
{
    public function __construct(
        private string $street,
        private string $city,
        private string $zipCode,
        private string $country,
    ) {}

    // এখন Address নিজেই নিজেকে যাচাই করে
    public function isValid(): bool
    {
        if (empty($this->street) || empty($this->city)) return false;
        if (empty($this->zipCode)) return false;
        if (!$this->isSupportedCountry()) return false;

        return $this->isValidZipCode();
    }

    private function isSupportedCountry(): bool
    {
        return in_array($this->country, ['BD', 'IN', 'US']);
    }

    private function isValidZipCode(): bool
    {
        // দেশ অনুযায়ী জিপকোড যাচাই
        return match ($this->country) {
            'BD' => preg_match('/^\d{4}$/', $this->zipCode),
            'US' => strlen($this->zipCode) === 5,
            default => !empty($this->zipCode),
        };
    }
}

class Order
{
    private string $customerName;
    private Address $shippingAddress;

    // Order শুধু Address-কে জিজ্ঞেস করে — দায়িত্ব ভাগ হয়েছে
    public function canShip(): bool
    {
        return $this->shippingAddress->isValid();
    }
}
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — Move Method প্রয়োগ
class Address {
    #street; #city; #zipCode; #country;

    constructor(street, city, zipCode, country) {
        this.#street = street;
        this.#city = city;
        this.#zipCode = zipCode;
        this.#country = country;
    }

    // ঠিকানা নিজেই নিজেকে যাচাই করে
    isValid() {
        if (!this.#street || !this.#city || !this.#zipCode) return false;
        if (!this.#isSupportedCountry()) return false;
        return this.#isValidZipCode();
    }

    #isSupportedCountry() {
        return ["BD", "IN", "US"].includes(this.#country);
    }

    #isValidZipCode() {
        const rules = {
            BD: /^\d{4}$/,
            US: /^\d{5}$/,
            IN: /^\d{6}$/,
        };
        return rules[this.#country]?.test(this.#zipCode) ?? true;
    }
}

class Order {
    #customerName;
    #shippingAddress;

    // Order শুধু প্রশ্ন করে, যাচাই নিজে করে না
    canShip() {
        return this.#shippingAddress.isValid();
    }
}
```

---

## ৪. 🔀 Replace Conditional with Polymorphism — শর্ত → বহুরূপিতা

### ধারণা

যদি কোনো `switch` বা `if-else` শৃঙ্খলে **টাইপ অনুযায়ী ভিন্ন আচরণ** থাকে, তাহলে সেটাকে **Strategy বা State Pattern** দিয়ে প্রতিস্থাপন করুন।

```
শর্ত → বহুরূপিতা রূপান্তর:

আগে:                              পরে:
┌──────────────┐                  ┌─────────────────┐
│ if (ধরন A)   │                  │ <<interface>>    │
│   কাজ A      │                  │ ShippingCarrier  │
│ elseif (B)   │    ──────►       │ + calculate()    │
│   কাজ B      │                  └──────┬──────────┘
│ elseif (C)   │                    ┌────┼────┐
│   কাজ C      │                    ▼    ▼    ▼
└──────────────┘                  ┌──┐ ┌──┐ ┌──┐
                                  │A │ │B │ │C │
                                  └──┘ └──┘ └──┘
```

### ❌ আগের কোড (PHP) — শিপিং খরচ ক্যালকুলেটর

```php
<?php
// ❌ খারাপ: নতুন ক্যারিয়ার যোগ করতে এই ফাংশন বদলাতে হবে
class ShippingCalculator
{
    public function calculateCost(string $carrier, float $weight, float $distance): float
    {
        if ($carrier === 'sundorbon') {
            // সুন্দরবন কুরিয়ার — ওজন অনুযায়ী
            $baseCost = 60;
            if ($weight > 1) {
                $baseCost += ($weight - 1) * 20;
            }
            if ($distance > 100) {
                $baseCost += 50; // দূরবর্তী চার্জ
            }
            return $baseCost;
        } elseif ($carrier === 'sa_paribahan') {
            // এসএ পরিবহন — দূরত্ব অনুযায়ী
            $costPerKm = 2.5;
            $baseCost = $distance * $costPerKm;
            if ($weight > 5) {
                $baseCost *= 1.5; // ভারী পার্সেল
            }
            return max($baseCost, 100); // সর্বনিম্ন ১০০ টাকা
        } elseif ($carrier === 'pathao') {
            // পাঠাও — ফ্ল্যাট রেট + ওজন
            $baseCost = 45;
            $baseCost += $weight * 15;
            // ঢাকার ভেতরে ফ্রি
            if ($distance <= 20) {
                return $baseCost;
            }
            return $baseCost + ($distance - 20) * 3;
        } elseif ($carrier === 'redx') {
            // রেডএক্স — জোন ভিত্তিক
            if ($distance <= 50) {
                return 70 + ($weight * 10);
            } elseif ($distance <= 200) {
                return 120 + ($weight * 15);
            } else {
                return 180 + ($weight * 20);
            }
        }

        throw new Exception("অজানা ক্যারিয়ার: $carrier");
    }
}
```

### ✅ পরের কোড (PHP) — Polymorphism প্রয়োগ

```php
<?php
// ✅ ভালো: প্রতিটি ক্যারিয়ার নিজের হিসাব নিজে করে

interface ShippingCarrier
{
    public function calculateCost(float $weight, float $distance): float;
    public function getName(): string;
}

class SundorbonCourier implements ShippingCarrier
{
    private const BASE_COST = 60;
    private const EXTRA_WEIGHT_RATE = 20;      // প্রতি কেজি
    private const LONG_DISTANCE_CHARGE = 50;
    private const LONG_DISTANCE_THRESHOLD = 100; // কিমি

    public function calculateCost(float $weight, float $distance): float
    {
        $cost = self::BASE_COST;

        if ($weight > 1) {
            $cost += ($weight - 1) * self::EXTRA_WEIGHT_RATE;
        }
        if ($distance > self::LONG_DISTANCE_THRESHOLD) {
            $cost += self::LONG_DISTANCE_CHARGE;
        }

        return $cost;
    }

    public function getName(): string
    {
        return 'সুন্দরবন কুরিয়ার';
    }
}

class SaParibahan implements ShippingCarrier
{
    private const COST_PER_KM = 2.5;
    private const HEAVY_PARCEL_MULTIPLIER = 1.5;
    private const HEAVY_THRESHOLD = 5;  // কেজি
    private const MINIMUM_COST = 100;

    public function calculateCost(float $weight, float $distance): float
    {
        $cost = $distance * self::COST_PER_KM;

        if ($weight > self::HEAVY_THRESHOLD) {
            $cost *= self::HEAVY_PARCEL_MULTIPLIER;
        }

        return max($cost, self::MINIMUM_COST);
    }

    public function getName(): string
    {
        return 'এসএ পরিবহন';
    }
}

class PathaoCourier implements ShippingCarrier
{
    private const BASE_COST = 45;
    private const WEIGHT_RATE = 15;
    private const FREE_DISTANCE = 20; // ঢাকার ভেতরে
    private const EXTRA_DISTANCE_RATE = 3;

    public function calculateCost(float $weight, float $distance): float
    {
        $cost = self::BASE_COST + ($weight * self::WEIGHT_RATE);

        if ($distance <= self::FREE_DISTANCE) {
            return $cost;
        }

        return $cost + ($distance - self::FREE_DISTANCE) * self::EXTRA_DISTANCE_RATE;
    }

    public function getName(): string
    {
        return 'পাঠাও';
    }
}

class RedxCourier implements ShippingCarrier
{
    // জোন ভিত্তিক রেট
    private const ZONES = [
        ['maxDistance' => 50,  'baseCost' => 70,  'weightRate' => 10],
        ['maxDistance' => 200, 'baseCost' => 120, 'weightRate' => 15],
        ['maxDistance' => PHP_INT_MAX, 'baseCost' => 180, 'weightRate' => 20],
    ];

    public function calculateCost(float $weight, float $distance): float
    {
        foreach (self::ZONES as $zone) {
            if ($distance <= $zone['maxDistance']) {
                return $zone['baseCost'] + ($weight * $zone['weightRate']);
            }
        }

        return 0; // কখনো এখানে আসবে না
    }

    public function getName(): string
    {
        return 'রেডএক্স';
    }
}

// ফ্যাক্টরি — ক্যারিয়ার তৈরি
class ShippingCarrierFactory
{
    private static array $carriers = [
        'sundorbon'    => SundorbonCourier::class,
        'sa_paribahan' => SaParibahan::class,
        'pathao'       => PathaoCourier::class,
        'redx'         => RedxCourier::class,
    ];

    public static function create(string $carrierName): ShippingCarrier
    {
        if (!isset(self::$carriers[$carrierName])) {
            throw new Exception("অজানা ক্যারিয়ার: $carrierName");
        }
        $class = self::$carriers[$carrierName];
        return new $class();
    }
}

// ব্যবহার — নতুন ক্যারিয়ার যোগ করতে শুধু নতুন ক্লাস + ফ্যাক্টরিতে এন্ট্রি
$carrier = ShippingCarrierFactory::create('pathao');
echo $carrier->getName() . ": " . $carrier->calculateCost(2.5, 30) . " টাকা";
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — Strategy Pattern দিয়ে শিপিং ক্যালকুলেটর

// প্রতিটি কৌশল আলাদা ক্লাস
class SundorbonCourier {
    static BASE_COST = 60;

    calculateCost(weight, distance) {
        let cost = SundorbonCourier.BASE_COST;
        if (weight > 1) cost += (weight - 1) * 20;
        if (distance > 100) cost += 50;
        return cost;
    }
}

class PathaoCourier {
    calculateCost(weight, distance) {
        let cost = 45 + weight * 15;
        if (distance <= 20) return cost; // ঢাকার ভেতরে
        return cost + (distance - 20) * 3;
    }
}

// ফ্যাক্টরি ম্যাপ
const carrierMap = {
    sundorbon: SundorbonCourier,
    pathao: PathaoCourier,
};

function getShippingCost(carrierName, weight, distance) {
    const CarrierClass = carrierMap[carrierName];
    if (!CarrierClass) throw new Error(`অজানা ক্যারিয়ার: ${carrierName}`);

    const carrier = new CarrierClass();
    return carrier.calculateCost(weight, distance);
}

// ব্যবহার
console.log(getShippingCost("pathao", 3, 15)); // ঢাকার ভেতরে
```

---

## ৫. 📦 Introduce Parameter Object — প্যারামিটার অবজেক্ট প্রবর্তন

### ধারণা

যখন একটি ফাংশনে **৩-এর বেশি প্যারামিটার** থাকে, তখন সংশ্লিষ্ট প্যারামিটারগুলোকে একটি অবজেক্টে (DTO) গুছিয়ে নিন।

```
প্যারামিটার সমস্যা:

আগে:                                    পরে:
createUser(                              createUser(UserDTO $dto)
    $name,        ┐                      
    $email,       │ ৬টি প্যারামিটার!     ┌─────────────┐
    $phone,       │ ───────────►        │   UserDTO    │
    $address,     │                      │ + name      │
    $city,        │                      │ + email     │
    $country      ┘                      │ + phone     │
)                                        │ + address   │
                                         │ + city      │
                                         │ + country   │
                                         └─────────────┘
```

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: অনেক বেশি প্যারামিটার — মনে রাখা কঠিন
function createUser(
    string $firstName,
    string $lastName,
    string $email,
    string $phone,
    string $street,
    string $city,
    string $zipCode,
    string $country,
    string $role,
    bool $isActive
): array {
    // ব্যবহারকারী তৈরি
    $fullName = $firstName . ' ' . $lastName;

    // ঠিকানা তৈরি
    $fullAddress = "$street, $city - $zipCode, $country";

    return [
        'name'    => $fullName,
        'email'   => $email,
        'phone'   => $phone,
        'address' => $fullAddress,
        'role'    => $role,
        'active'  => $isActive,
    ];
}

// কল করার সময় কোন পজিশনে কী — ভুল হওয়া স্বাভাবিক!
createUser('রহিম', 'উদ্দিন', 'rahim@email.com', '01712345678',
    'মিরপুর ১০', 'ঢাকা', '1216', 'BD', 'admin', true);
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: সুসংগঠিত DTO ক্লাস

class UserAddress
{
    public function __construct(
        public readonly string $street,
        public readonly string $city,
        public readonly string $zipCode,
        public readonly string $country,
    ) {}

    public function getFullAddress(): string
    {
        return "{$this->street}, {$this->city} - {$this->zipCode}, {$this->country}";
    }
}

class CreateUserDTO
{
    public function __construct(
        public readonly string $firstName,
        public readonly string $lastName,
        public readonly string $email,
        public readonly string $phone,
        public readonly UserAddress $address,
        public readonly string $role = 'user',
        public readonly bool $isActive = true,
    ) {}

    public function getFullName(): string
    {
        return "{$this->firstName} {$this->lastName}";
    }
}

class UserService
{
    // এখন মাত্র ১টি প্যারামিটার — পরিষ্কার!
    public function createUser(CreateUserDTO $dto): array
    {
        return [
            'name'    => $dto->getFullName(),
            'email'   => $dto->email,
            'phone'   => $dto->phone,
            'address' => $dto->address->getFullAddress(),
            'role'    => $dto->role,
            'active'  => $dto->isActive,
        ];
    }
}

// ব্যবহার — পরিষ্কার ও ভুলের সম্ভাবনা কম
$address = new UserAddress('মিরপুর ১০', 'ঢাকা', '1216', 'BD');
$dto = new CreateUserDTO(
    firstName: 'রহিম',
    lastName: 'উদ্দিন',
    email: 'rahim@email.com',
    phone: '01712345678',
    address: $address,
    role: 'admin',
);

$service = new UserService();
$user = $service->createUser($dto);
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — Parameter Object

class UserAddress {
    constructor({ street, city, zipCode, country }) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
        this.country = country;
    }

    getFullAddress() {
        return `${this.street}, ${this.city} - ${this.zipCode}, ${this.country}`;
    }
}

class CreateUserRequest {
    constructor({ firstName, lastName, email, phone, address, role = "user", isActive = true }) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.phone = phone;
        this.address = new UserAddress(address);
        this.role = role;
        this.isActive = isActive;
    }

    get fullName() {
        return `${this.firstName} ${this.lastName}`;
    }
}

// ব্যবহার — নামসহ প্যারামিটার, ক্রম গুরুত্বপূর্ণ নয়
const request = new CreateUserRequest({
    firstName: "রহিম",
    lastName: "উদ্দিন",
    email: "rahim@email.com",
    phone: "01712345678",
    address: { street: "মিরপুর ১০", city: "ঢাকা", zipCode: "1216", country: "BD" },
    role: "admin",
});

console.log(request.fullName); // রহিম উদ্দিন
console.log(request.address.getFullAddress()); // মিরপুর ১০, ঢাকা - 1216, BD
```

---

## ৬. 🔢 Replace Magic Number with Constant — ম্যাজিক নম্বর প্রতিস্থাপন

### ধারণা

কোডে সরাসরি সংখ্যা (ম্যাজিক নম্বর) ব্যবহার না করে **নামযুক্ত কনস্ট্যান্ট** ব্যবহার করুন।

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: ০.১৫ কী? ৩ কী? ৫০০০ কী?
function calculateEmployeeSalary(float $baseSalary, int $yearsWorked): array
{
    $bonus = 0;
    if ($yearsWorked > 5) {
        $bonus = $baseSalary * 0.15;
    } elseif ($yearsWorked > 3) {
        $bonus = $baseSalary * 0.10;
    }

    $tax = $baseSalary * 0.12;
    $insurance = 5000;
    $providentFund = $baseSalary * 0.08;

    if ($baseSalary > 50000) {
        $tax = $baseSalary * 0.20;
    }

    return [
        'gross' => $baseSalary + $bonus,
        'tax' => $tax,
        'insurance' => $insurance,
        'pf' => $providentFund,
        'net' => $baseSalary + $bonus - $tax - $insurance - $providentFund,
    ];
}
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: প্রতিটি সংখ্যার একটি অর্থবহ নাম আছে
class SalaryConstants
{
    // বোনাস হার
    public const SENIOR_BONUS_RATE = 0.15;
    public const MID_LEVEL_BONUS_RATE = 0.10;
    public const SENIOR_THRESHOLD_YEARS = 5;
    public const MID_LEVEL_THRESHOLD_YEARS = 3;

    // ট্যাক্স হার
    public const STANDARD_TAX_RATE = 0.12;
    public const HIGH_INCOME_TAX_RATE = 0.20;
    public const HIGH_INCOME_THRESHOLD = 50000;

    // কর্তন
    public const MONTHLY_INSURANCE = 5000;
    public const PROVIDENT_FUND_RATE = 0.08;
}

function calculateEmployeeSalary(float $baseSalary, int $yearsWorked): array
{
    $bonus = calculateBonus($baseSalary, $yearsWorked);
    $tax = calculateTax($baseSalary);
    $providentFund = $baseSalary * SalaryConstants::PROVIDENT_FUND_RATE;

    $grossSalary = $baseSalary + $bonus;
    $totalDeductions = $tax + SalaryConstants::MONTHLY_INSURANCE + $providentFund;

    return [
        'gross'     => $grossSalary,
        'tax'       => $tax,
        'insurance' => SalaryConstants::MONTHLY_INSURANCE,
        'pf'        => $providentFund,
        'net'       => $grossSalary - $totalDeductions,
    ];
}

function calculateBonus(float $baseSalary, int $yearsWorked): float
{
    if ($yearsWorked > SalaryConstants::SENIOR_THRESHOLD_YEARS) {
        return $baseSalary * SalaryConstants::SENIOR_BONUS_RATE;
    }
    if ($yearsWorked > SalaryConstants::MID_LEVEL_THRESHOLD_YEARS) {
        return $baseSalary * SalaryConstants::MID_LEVEL_BONUS_RATE;
    }
    return 0;
}

function calculateTax(float $baseSalary): float
{
    if ($baseSalary > SalaryConstants::HIGH_INCOME_THRESHOLD) {
        return $baseSalary * SalaryConstants::HIGH_INCOME_TAX_RATE;
    }
    return $baseSalary * SalaryConstants::STANDARD_TAX_RATE;
}
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — কনস্ট্যান্ট ব্যবহার
const SALARY = Object.freeze({
    SENIOR_BONUS_RATE: 0.15,
    MID_LEVEL_BONUS_RATE: 0.10,
    SENIOR_YEARS: 5,
    MID_LEVEL_YEARS: 3,
    STANDARD_TAX: 0.12,
    HIGH_TAX: 0.20,
    HIGH_INCOME_LIMIT: 50000,
    INSURANCE: 5000,
    PF_RATE: 0.08,
});

function calculateSalary(baseSalary, yearsWorked) {
    const bonus = yearsWorked > SALARY.SENIOR_YEARS
        ? baseSalary * SALARY.SENIOR_BONUS_RATE
        : yearsWorked > SALARY.MID_LEVEL_YEARS
        ? baseSalary * SALARY.MID_LEVEL_BONUS_RATE
        : 0;

    const tax = baseSalary > SALARY.HIGH_INCOME_LIMIT
        ? baseSalary * SALARY.HIGH_TAX
        : baseSalary * SALARY.STANDARD_TAX;

    const pf = baseSalary * SALARY.PF_RATE;
    const gross = baseSalary + bonus;
    const net = gross - tax - SALARY.INSURANCE - pf;

    return { gross, tax, insurance: SALARY.INSURANCE, pf, net };
}
```

---

## ৭. 🏗️ Extract Class / Inline Class — ক্লাস বিভাজন

### ধারণা

**God Class** (যে ক্লাস সবকিছু করে) কে ভেঙে একাধিক ছোট, ফোকাসড ক্লাসে রূপান্তর করুন। প্রতিটি ক্লাসের **একটিই দায়িত্ব** থাকবে (Single Responsibility Principle)।

```
God Class ভাঙার প্রক্রিয়া:

    ┌──────────────────────────────────┐
    │         UserManager (God)        │
    │                                  │
    │ + createUser()                   │
    │ + updateUser()                   │
    │ + deleteUser()                   │
    │ + validateEmail()                │
    │ + validatePassword()             │
    │ + hashPassword()                 │
    │ + findById()                     │
    │ + findByEmail()                  │
    │ + saveToDatabase()               │
    │ + sendWelcomeEmail()             │
    │ + sendPasswordResetEmail()       │
    │ + generateReport()               │
    └──────────┬───────────────────────┘
               │ Extract Class
               ▼
    ┌──────────────┐  ┌──────────────────┐
    │    User       │  │  UserValidator   │
    │ + name        │  │ + validateEmail()│
    │ + email       │  │ + validatePass() │
    │ + getFullName │  │ + hashPassword() │
    └──────────────┘  └──────────────────┘

    ┌──────────────────┐  ┌──────────────────┐
    │ UserRepository   │  │  UserNotifier    │
    │ + findById()     │  │ + sendWelcome()  │
    │ + findByEmail()  │  │ + sendPassReset()│
    │ + save()         │  │ + sendReport()   │
    │ + delete()       │  └──────────────────┘
    └──────────────────┘
```

### ❌ আগের কোড (PHP) — God Class

```php
<?php
// ❌ খারাপ: একটি ক্লাসে ৪-৫টি দায়িত্ব!
class UserManager
{
    private $db;

    // ১. ব্যবহারকারী তৈরি + যাচাই + সংরক্ষণ + নোটিফিকেশন — সব একসাথে!
    public function createUser(string $name, string $email, string $password): array
    {
        // যাচাই
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new Exception("অবৈধ ইমেইল");
        }
        if (strlen($password) < 8) {
            throw new Exception("পাসওয়ার্ড কমপক্ষে ৮ অক্ষর");
        }
        if (!preg_match('/[A-Z]/', $password)) {
            throw new Exception("বড় হাতের অক্ষর দরকার");
        }

        // পাসওয়ার্ড হ্যাশ
        $hashedPassword = password_hash($password, PASSWORD_BCRYPT);

        // ডেটাবেজে সংরক্ষণ
        $stmt = $this->db->prepare("INSERT INTO users (name, email, password) VALUES (?, ?, ?)");
        $stmt->execute([$name, $email, $hashedPassword]);
        $userId = $this->db->lastInsertId();

        // স্বাগত ইমেইল
        $subject = "স্বাগতম, $name!";
        $body = "আপনার অ্যাকাউন্ট তৈরি হয়েছে।";
        mail($email, $subject, $body);

        return ['id' => $userId, 'name' => $name, 'email' => $email];
    }

    public function findById(int $id): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        return $stmt->fetch() ?: null;
    }

    public function findByEmail(string $email): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE email = ?");
        $stmt->execute([$email]);
        return $stmt->fetch() ?: null;
    }

    public function sendPasswordResetEmail(string $email): void
    {
        $token = bin2hex(random_bytes(32));
        $this->db->prepare("UPDATE users SET reset_token = ? WHERE email = ?")
            ->execute([$token, $email]);
        mail($email, "পাসওয়ার্ড রিসেট", "টোকেন: $token");
    }

    public function generateMonthlyReport(): string
    {
        $stmt = $this->db->query("SELECT COUNT(*) as total FROM users");
        $total = $stmt->fetch()['total'];
        return "মোট ব্যবহারকারী: $total";
    }
}
```

### ✅ পরের কোড (PHP) — Extract Class প্রয়োগ

```php
<?php
// ✅ ভালো: প্রতিটি ক্লাসের একটি দায়িত্ব

// ১. ডেটা ক্লাস
class User
{
    public function __construct(
        public readonly ?int $id,
        public readonly string $name,
        public readonly string $email,
        private string $hashedPassword,
    ) {}

    public function getHashedPassword(): string
    {
        return $this->hashedPassword;
    }
}

// ২. যাচাইকরণ ক্লাস
class UserValidator
{
    public function validate(string $email, string $password): void
    {
        $this->validateEmail($email);
        $this->validatePassword($password);
    }

    private function validateEmail(string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new Exception("অবৈধ ইমেইল");
        }
    }

    private function validatePassword(string $password): void
    {
        if (strlen($password) < 8) {
            throw new Exception("পাসওয়ার্ড কমপক্ষে ৮ অক্ষর");
        }
        if (!preg_match('/[A-Z]/', $password)) {
            throw new Exception("বড় হাতের অক্ষর দরকার");
        }
    }

    public function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_BCRYPT);
    }
}

// ৩. ডেটাবেজ ক্লাস
class UserRepository
{
    public function __construct(private PDO $db) {}

    public function save(User $user): int
    {
        $stmt = $this->db->prepare("INSERT INTO users (name, email, password) VALUES (?, ?, ?)");
        $stmt->execute([$user->name, $user->email, $user->getHashedPassword()]);
        return (int) $this->db->lastInsertId();
    }

    public function findById(int $id): ?User
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        $row = $stmt->fetch();
        return $row ? new User($row['id'], $row['name'], $row['email'], $row['password']) : null;
    }

    public function findByEmail(string $email): ?User
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE email = ?");
        $stmt->execute([$email]);
        $row = $stmt->fetch();
        return $row ? new User($row['id'], $row['name'], $row['email'], $row['password']) : null;
    }
}

// ৪. নোটিফিকেশন ক্লাস
class UserNotifier
{
    public function sendWelcomeEmail(User $user): void
    {
        mail($user->email, "স্বাগতম, {$user->name}!", "আপনার অ্যাকাউন্ট তৈরি হয়েছে।");
    }

    public function sendPasswordResetEmail(string $email, string $token): void
    {
        mail($email, "পাসওয়ার্ড রিসেট", "আপনার রিসেট টোকেন: $token");
    }
}

// ৫. সমন্বয়কারী সার্ভিস — সবাইকে একত্রিত করে
class UserService
{
    public function __construct(
        private UserValidator $validator,
        private UserRepository $repository,
        private UserNotifier $notifier,
    ) {}

    public function register(string $name, string $email, string $password): User
    {
        $this->validator->validate($email, $password);
        $hashedPassword = $this->validator->hashPassword($password);

        $user = new User(null, $name, $email, $hashedPassword);
        $userId = $this->repository->save($user);

        $savedUser = new User($userId, $name, $email, $hashedPassword);
        $this->notifier->sendWelcomeEmail($savedUser);

        return $savedUser;
    }
}
```

### ✅ পরের কোড (JavaScript) — Extract Class

```javascript
// ✅ জাভাস্ক্রিপ্ট — প্রতিটি দায়িত্ব আলাদা ক্লাসে

class UserValidator {
    validate(email, password) {
        if (!email.includes("@")) throw new Error("অবৈধ ইমেইল");
        if (password.length < 8) throw new Error("পাসওয়ার্ড কমপক্ষে ৮ অক্ষর");
    }
}

class UserRepository {
    #users = new Map();
    #nextId = 1;

    save(userData) {
        const id = this.#nextId++;
        const user = { id, ...userData };
        this.#users.set(id, user);
        return user;
    }

    findById(id) {
        return this.#users.get(id) ?? null;
    }
}

class UserNotifier {
    async sendWelcome(user) {
        console.log(`স্বাগত ইমেইল পাঠানো হচ্ছে: ${user.email}`);
    }
}

// সমন্বয়কারী সার্ভিস
class UserService {
    #validator;
    #repository;
    #notifier;

    constructor(validator, repository, notifier) {
        this.#validator = validator;
        this.#repository = repository;
        this.#notifier = notifier;
    }

    async register(name, email, password) {
        this.#validator.validate(email, password);
        const user = this.#repository.save({ name, email, password });
        await this.#notifier.sendWelcome(user);
        return user;
    }
}
```

---

## ৮. 🔄 Replace Temp with Query — টেম্প ভ্যারিয়েবল → মেথড কল

### ধারণা

যদি একটি টেম্পোরারি ভ্যারিয়েবল **শুধু একটি এক্সপ্রেশনের ফলাফল ধরে** রাখে এবং **একাধিক জায়গায়** সেই হিসাব দরকার হয়, তাহলে সেটাকে একটি মেথডে রূপান্তর করুন।

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: অনেক টেম্প ভ্যারিয়েবল
class Product
{
    private float $price;
    private int $quantity;

    public function getInvoiceLine(): string
    {
        $basePrice = $this->price * $this->quantity;
        $discountFactor = ($basePrice > 1000) ? 0.95 : 0.98;
        $discountedPrice = $basePrice * $discountFactor;
        $tax = $discountedPrice * 0.15;
        $total = $discountedPrice + $tax;

        return "মূল্য: $total টাকা";
    }
}
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: প্রতিটি হিসাব একটি মেথড — পুনঃব্যবহারযোগ্য
class Product
{
    private float $price;
    private int $quantity;

    public function getInvoiceLine(): string
    {
        return "মূল্য: {$this->getTotalWithTax()} টাকা";
    }

    public function getBasePrice(): float
    {
        return $this->price * $this->quantity;
    }

    public function getDiscountFactor(): float
    {
        return ($this->getBasePrice() > 1000) ? 0.95 : 0.98;
    }

    public function getDiscountedPrice(): float
    {
        return $this->getBasePrice() * $this->getDiscountFactor();
    }

    public function getTax(): float
    {
        return $this->getDiscountedPrice() * 0.15;
    }

    public function getTotalWithTax(): float
    {
        return $this->getDiscountedPrice() + $this->getTax();
    }
}
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — Replace Temp with Query
class Product {
    #price;
    #quantity;

    constructor(price, quantity) {
        this.#price = price;
        this.#quantity = quantity;
    }

    get basePrice() {
        return this.#price * this.#quantity;
    }

    get discountFactor() {
        return this.basePrice > 1000 ? 0.95 : 0.98;
    }

    get discountedPrice() {
        return this.basePrice * this.discountFactor;
    }

    get tax() {
        return this.discountedPrice * 0.15;
    }

    get total() {
        return this.discountedPrice + this.tax;
    }

    getInvoiceLine() {
        return `মূল্য: ${this.total} টাকা`;
    }
}
```

---

## ৯. 🎵 Compose Method — সমান Abstraction Level

### ধারণা

একটি মেথডের সব ধাপ **একই abstraction level**-এ থাকা উচিত। উচ্চ-স্তরের মেথড শুধু "কী হচ্ছে" বলবে, "কীভাবে হচ্ছে" নিম্ন-স্তরের মেথডে থাকবে।

```
Abstraction Level মিশ্রণ সমস্যা:

❌ খারাপ:                          ✅ ভালো:
┌──────────────────────┐          ┌──────────────────────┐
│ processOrder()       │          │ processOrder()       │
│                      │          │   validateOrder()    │ ← সমান level
│ // যাচাই (উচ্চ)      │          │   calculateTotal()   │ ← সমান level
│ if (!$order->items)  │          │   chargePayment()    │ ← সমান level
│   throw ...          │          │   sendConfirmation() │ ← সমান level
│                      │          └──────────────────────┘
│ // হিসাব (নিম্ন)     │
│ $sum = 0;            │
│ foreach ($items ...) │
│   $sum += ...        │
│                      │
│ // পেমেন্ট (উচ্চ)    │
│ $gateway->charge()   │
│                      │
│ // ইমেইল (নিম্ন)     │
│ $headers = "From:..."│
│ mail(...)            │
└──────────────────────┘
```

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: বিভিন্ন abstraction level মিশ্রিত
class OrderProcessor
{
    public function processOrder(array $order): void
    {
        // উচ্চ স্তর: যাচাই
        if (empty($order['items'])) {
            throw new Exception("অর্ডারে আইটেম নেই");
        }

        // নিম্ন স্তর: হিসাব (লুপ, গণনা)
        $total = 0;
        foreach ($order['items'] as $item) {
            $itemPrice = $item['price'] * $item['qty'];
            if ($item['qty'] > 10) {
                $itemPrice *= 0.9;
            }
            $total += $itemPrice;
        }
        $tax = $total * 0.15;
        $grandTotal = $total + $tax;

        // উচ্চ স্তর: পেমেন্ট
        $gateway = new PaymentGateway();
        $gateway->charge($order['card'], $grandTotal);

        // নিম্ন স্তর: ইমেইল (হেডার, বডি তৈরি)
        $headers = "From: shop@example.com\r\nContent-Type: text/html";
        $body = "<h1>অর্ডার নিশ্চিত</h1><p>মোট: $grandTotal টাকা</p>";
        mail($order['email'], "অর্ডার নিশ্চিতকরণ", $body, $headers);
    }
}
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: প্রতিটি ধাপ সমান abstraction level-এ
class OrderProcessor
{
    // মূল মেথড — শুধু "কী হচ্ছে" বলে
    public function processOrder(array $order): void
    {
        $this->validateOrder($order);
        $grandTotal = $this->calculateGrandTotal($order['items']);
        $this->chargePayment($order['card'], $grandTotal);
        $this->sendConfirmation($order['email'], $grandTotal);
    }

    private function validateOrder(array $order): void
    {
        if (empty($order['items'])) {
            throw new Exception("অর্ডারে আইটেম নেই");
        }
    }

    private function calculateGrandTotal(array $items): float
    {
        $subtotal = $this->calculateSubtotal($items);
        $tax = $subtotal * 0.15;
        return $subtotal + $tax;
    }

    private function calculateSubtotal(array $items): float
    {
        $total = 0;
        foreach ($items as $item) {
            $total += $this->calculateItemPrice($item);
        }
        return $total;
    }

    private function calculateItemPrice(array $item): float
    {
        $price = $item['price'] * $item['qty'];
        $hasBulkDiscount = $item['qty'] > 10;
        return $hasBulkDiscount ? $price * 0.9 : $price;
    }

    private function chargePayment(string $card, float $amount): void
    {
        $gateway = new PaymentGateway();
        $gateway->charge($card, $amount);
    }

    private function sendConfirmation(string $email, float $total): void
    {
        $body = "<h1>অর্ডার নিশ্চিত</h1><p>মোট: $total টাকা</p>";
        mail($email, "অর্ডার নিশ্চিতকরণ", $body, "From: shop@example.com\r\nContent-Type: text/html");
    }
}
```

---

## ১০. 🔗 Replace Inheritance with Composition — উত্তরাধিকার → সংমিশ্রণ

### ধারণা

যখন একটি ক্লাস আরেকটি ক্লাসকে **extends** করে শুধুমাত্র কিছু ফাংশনালিটি পুনঃব্যবহার করতে (সত্যিকারের "is-a" সম্পর্ক নেই), তখন **composition** ("has-a") ব্যবহার করুন।

```
is-a বনাম has-a:

❌ Stack extends ArrayList         ✅ Stack has-a ArrayList
┌────────────────────┐            ┌────────────────────┐
│     ArrayList      │            │      Stack         │
│ + add()            │            │ - list: ArrayList  │
│ + remove()         │            │ + push()           │
│ + get()            │            │ + pop()            │
│ + sort()  ← ভুল!  │            │ + peek()           │
└────────┬───────────┘            └────────────────────┘
         │ extends                Stack ArrayList-এর
┌────────┴───────────┐            ফাংশন "ব্যবহার" করে,
│      Stack         │            কিন্তু sort(), get()
│ + push()           │            বাইরে দেয় না
│ + pop()            │
│ + sort() ← উত্তরাধিকারে │
│           পেয়ে গেছে!    │
└────────────────────┘
```

### ❌ আগের কোড (PHP)

```php
<?php
// ❌ খারাপ: Logger কোনো DatabaseConnection "নয়"
class DatabaseConnection
{
    protected $connection;

    public function connect(string $host, string $db): void
    {
        $this->connection = new PDO("mysql:host=$host;dbname=$db", 'user', 'pass');
    }

    public function query(string $sql): array
    {
        return $this->connection->query($sql)->fetchAll();
    }

    public function execute(string $sql, array $params): bool
    {
        return $this->connection->prepare($sql)->execute($params);
    }
}

// Logger DatabaseConnection extends করেছে শুধু execute() ব্যবহার করতে
// কিন্তু Logger কোনো Database "নয়"!
class Logger extends DatabaseConnection
{
    public function log(string $level, string $message): void
    {
        $this->execute(
            "INSERT INTO logs (level, message, created_at) VALUES (?, ?, NOW())",
            [$level, $message]
        );
    }
}
```

### ✅ পরের কোড (PHP)

```php
<?php
// ✅ ভালো: Logger "ব্যবহার করে" DatabaseConnection — has-a সম্পর্ক
class DatabaseConnection
{
    private PDO $connection;

    public function connect(string $host, string $db): void
    {
        $this->connection = new PDO("mysql:host=$host;dbname=$db", 'user', 'pass');
    }

    public function execute(string $sql, array $params): bool
    {
        return $this->connection->prepare($sql)->execute($params);
    }
}

// Logger-এর কাছে Database আছে, Logger Database "নয়"
class Logger
{
    public function __construct(
        private DatabaseConnection $db
    ) {}

    public function log(string $level, string $message): void
    {
        $this->db->execute(
            "INSERT INTO logs (level, message, created_at) VALUES (?, ?, NOW())",
            [$level, $message]
        );
    }

    public function info(string $message): void
    {
        $this->log('INFO', $message);
    }

    public function error(string $message): void
    {
        $this->log('ERROR', $message);
    }
}

// ব্যবহার
$db = new DatabaseConnection();
$db->connect('localhost', 'myapp');
$logger = new Logger($db); // Composition — ইনজেক্ট করা হচ্ছে
$logger->info("সিস্টেম চালু হয়েছে");
```

### ✅ পরের কোড (JavaScript)

```javascript
// ✅ জাভাস্ক্রিপ্ট — Composition
class DatabaseConnection {
    async execute(sql, params) {
        console.log(`SQL: ${sql}`, params);
        return true;
    }
}

// Logger "has-a" DatabaseConnection
class Logger {
    #db;

    constructor(db) {
        this.#db = db;
    }

    async log(level, message) {
        await this.#db.execute(
            "INSERT INTO logs (level, message) VALUES (?, ?)",
            [level, message]
        );
    }

    info(msg) { return this.log("INFO", msg); }
    error(msg) { return this.log("ERROR", msg); }
}

// ব্যবহার
const db = new DatabaseConnection();
const logger = new Logger(db);
logger.info("অ্যাপ্লিকেশন শুরু হয়েছে");
```

---

## 🏥 বিশাল উদাহরণ: হাসপাতাল ম্যানেজমেন্ট সিস্টেম

### একাধিক রিফ্যাক্টরিং কৌশল একত্রে প্রয়োগ

এই উদাহরণে আমরা একটি অগোছালো হাসপাতাল সিস্টেমকে একাধিক কৌশল ব্যবহার করে রিফ্যাক্টর করব।

```
প্রয়োগকৃত কৌশলসমূহ:
┌──────────────────────────────────────────────────┐
│ ✦ Extract Method        — বড় ফাংশন ভাঙা         │
│ ✦ Extract Class         — God Class ভাঙা         │
│ ✦ Parameter Object      — অনেক প্যারামিটার গুছানো │
│ ✦ Replace Conditional   — switch → Polymorphism  │
│ ✦ Rename                — অর্থবহ নামকরণ          │
│ ✦ Move Method           — সঠিক ক্লাসে সরানো      │
│ ✦ Magic Number → Const  — হার্ডকোড ভ্যালু সরানো   │
└──────────────────────────────────────────────────┘
```

### ❌ আগের কোড (PHP) — অগোছালো হাসপাতাল সিস্টেম

```php
<?php
// ❌ ভয়াবহ God Class — সব কিছু একটি ক্লাসে!
class HospitalSystem
{
    private $db;

    // একটি বিশাল মেথড — রোগী ভর্তি থেকে বিল পর্যন্ত সব করে
    public function proc($n, $a, $g, $t, $d, $r, $ins)
    {
        // রোগী যাচাই
        if (empty($n)) throw new Exception("নাম দিন");
        if ($a < 0 || $a > 150) throw new Exception("বয়স ভুল");
        if (!in_array($g, ['M', 'F', 'O'])) throw new Exception("লিঙ্গ ভুল");

        // ডাক্তার খুঁজে পাওয়া
        $doc = null;
        if ($t === 'cardiac') {
            $doc = $this->db->query("SELECT * FROM doctors WHERE dept='cardiology' AND available=1 LIMIT 1")->fetch();
        } elseif ($t === 'neuro') {
            $doc = $this->db->query("SELECT * FROM doctors WHERE dept='neurology' AND available=1 LIMIT 1")->fetch();
        } elseif ($t === 'ortho') {
            $doc = $this->db->query("SELECT * FROM doctors WHERE dept='orthopedics' AND available=1 LIMIT 1")->fetch();
        } elseif ($t === 'general') {
            $doc = $this->db->query("SELECT * FROM doctors WHERE dept='general' AND available=1 LIMIT 1")->fetch();
        }

        if (!$doc) throw new Exception("ডাক্তার নেই");

        // রুম খুঁজে পাওয়া
        $room = null;
        if ($r === 'general') {
            $room = $this->db->query("SELECT * FROM rooms WHERE type='general' AND occupied=0 LIMIT 1")->fetch();
            $roomCost = 500;
        } elseif ($r === 'semi_private') {
            $room = $this->db->query("SELECT * FROM rooms WHERE type='semi_private' AND occupied=0 LIMIT 1")->fetch();
            $roomCost = 1500;
        } elseif ($r === 'private') {
            $room = $this->db->query("SELECT * FROM rooms WHERE type='private' AND occupied=0 LIMIT 1")->fetch();
            $roomCost = 3000;
        } elseif ($r === 'icu') {
            $room = $this->db->query("SELECT * FROM rooms WHERE type='icu' AND occupied=0 LIMIT 1")->fetch();
            $roomCost = 8000;
        }

        if (!$room) throw new Exception("রুম নেই");

        // রোগী সংরক্ষণ
        $this->db->prepare("INSERT INTO patients (name, age, gender, type, doctor_id, room_id) VALUES (?,?,?,?,?,?)")
            ->execute([$n, $a, $g, $t, $doc['id'], $room['id']]);
        $pid = $this->db->lastInsertId();

        // রুম দখল করা
        $this->db->prepare("UPDATE rooms SET occupied=1 WHERE id=?")->execute([$room['id']]);

        // বিল হিসাব
        $consultFee = 0;
        if ($t === 'cardiac') $consultFee = 2000;
        elseif ($t === 'neuro') $consultFee = 2500;
        elseif ($t === 'ortho') $consultFee = 1500;
        else $consultFee = 800;

        $admissionFee = 500;
        $totalBill = $consultFee + $roomCost + $admissionFee;

        // বীমা ছাড়
        if ($ins) {
            if ($ins === 'full') {
                $totalBill = $totalBill * 0.1; // ৯০% কভার
            } elseif ($ins === 'partial') {
                $totalBill = $totalBill * 0.5; // ৫০% কভার
            }
        }

        // বিল সংরক্ষণ
        $this->db->prepare("INSERT INTO bills (patient_id, amount) VALUES (?, ?)")
            ->execute([$pid, $totalBill]);

        // SMS পাঠানো
        $msg = "প্রিয় $n, আপনি ভর্তি হয়েছেন। বিল: $totalBill টাকা।";
        // sms_send($phone, $msg);

        return ['patient_id' => $pid, 'bill' => $totalBill, 'room' => $room['id']];
    }
}
```

### ✅ পরের কোড (PHP) — সম্পূর্ণ রিফ্যাক্টরড

```php
<?php
// ✅ একাধিক রিফ্যাক্টরিং কৌশল প্রয়োগ করা হয়েছে

// ═══════════════════════════════════════════
// ১. Parameter Object — রোগীর তথ্য গুছানো
// ═══════════════════════════════════════════
class PatientAdmissionRequest
{
    public function __construct(
        public readonly string $name,
        public readonly int $age,
        public readonly string $gender,
        public readonly string $treatmentType,  // Rename: $t → $treatmentType
        public readonly string $roomType,        // Rename: $r → $roomType
        public readonly ?string $insuranceType = null,
    ) {}
}

// ═══════════════════════════════════════════
// ২. Extract Class — যাচাইকরণ আলাদা
// ═══════════════════════════════════════════
class PatientValidator
{
    private const VALID_GENDERS = ['M', 'F', 'O'];

    public function validate(PatientAdmissionRequest $request): void
    {
        if (empty($request->name)) {
            throw new InvalidArgumentException("রোগীর নাম দিন");
        }
        if ($request->age < 0 || $request->age > 150) {
            throw new InvalidArgumentException("বয়স ০-১৫০ এর মধ্যে হতে হবে");
        }
        if (!in_array($request->gender, self::VALID_GENDERS)) {
            throw new InvalidArgumentException("লিঙ্গ M, F বা O হতে হবে");
        }
    }
}

// ═══════════════════════════════════════════
// ৩. Replace Conditional — বিভাগ অনুযায়ী Polymorphism
// ═══════════════════════════════════════════
interface Department
{
    public function getDepartmentName(): string;
    public function getConsultationFee(): float;
    public function findAvailableDoctor(PDO $db): ?array;
}

class CardiologyDepartment implements Department
{
    private const CONSULTATION_FEE = 2000;

    public function getDepartmentName(): string { return 'cardiology'; }
    public function getConsultationFee(): float { return self::CONSULTATION_FEE; }

    public function findAvailableDoctor(PDO $db): ?array
    {
        $stmt = $db->query("SELECT * FROM doctors WHERE dept='cardiology' AND available=1 LIMIT 1");
        return $stmt->fetch() ?: null;
    }
}

class NeurologyDepartment implements Department
{
    private const CONSULTATION_FEE = 2500;

    public function getDepartmentName(): string { return 'neurology'; }
    public function getConsultationFee(): float { return self::CONSULTATION_FEE; }

    public function findAvailableDoctor(PDO $db): ?array
    {
        $stmt = $db->query("SELECT * FROM doctors WHERE dept='neurology' AND available=1 LIMIT 1");
        return $stmt->fetch() ?: null;
    }
}

class OrthopedicsDepartment implements Department
{
    private const CONSULTATION_FEE = 1500;

    public function getDepartmentName(): string { return 'orthopedics'; }
    public function getConsultationFee(): float { return self::CONSULTATION_FEE; }

    public function findAvailableDoctor(PDO $db): ?array
    {
        $stmt = $db->query("SELECT * FROM doctors WHERE dept='orthopedics' AND available=1 LIMIT 1");
        return $stmt->fetch() ?: null;
    }
}

class GeneralDepartment implements Department
{
    private const CONSULTATION_FEE = 800;

    public function getDepartmentName(): string { return 'general'; }
    public function getConsultationFee(): float { return self::CONSULTATION_FEE; }

    public function findAvailableDoctor(PDO $db): ?array
    {
        $stmt = $db->query("SELECT * FROM doctors WHERE dept='general' AND available=1 LIMIT 1");
        return $stmt->fetch() ?: null;
    }
}

// ফ্যাক্টরি — বিভাগ তৈরি
class DepartmentFactory
{
    private static array $map = [
        'cardiac' => CardiologyDepartment::class,
        'neuro'   => NeurologyDepartment::class,
        'ortho'   => OrthopedicsDepartment::class,
        'general' => GeneralDepartment::class,
    ];

    public static function create(string $type): Department
    {
        if (!isset(self::$map[$type])) {
            throw new InvalidArgumentException("অজানা চিকিৎসা ধরন: $type");
        }
        $class = self::$map[$type];
        return new $class();
    }
}

// ═══════════════════════════════════════════
// ৪. Replace Conditional — রুম ধরন Polymorphism
// ═══════════════════════════════════════════
interface RoomType
{
    public function getTypeName(): string;
    public function getDailyCost(): float;
}

class GeneralRoom implements RoomType
{
    public function getTypeName(): string { return 'general'; }
    public function getDailyCost(): float { return 500; }
}

class SemiPrivateRoom implements RoomType
{
    public function getTypeName(): string { return 'semi_private'; }
    public function getDailyCost(): float { return 1500; }
}

class PrivateRoom implements RoomType
{
    public function getTypeName(): string { return 'private'; }
    public function getDailyCost(): float { return 3000; }
}

class IcuRoom implements RoomType
{
    public function getTypeName(): string { return 'icu'; }
    public function getDailyCost(): float { return 8000; }
}

class RoomTypeFactory
{
    private static array $map = [
        'general'      => GeneralRoom::class,
        'semi_private' => SemiPrivateRoom::class,
        'private'      => PrivateRoom::class,
        'icu'          => IcuRoom::class,
    ];

    public static function create(string $type): RoomType
    {
        if (!isset(self::$map[$type])) {
            throw new InvalidArgumentException("অজানা রুম ধরন: $type");
        }
        $class = self::$map[$type];
        return new $class();
    }
}

// ═══════════════════════════════════════════
// ৫. Extract Class — রুম সংরক্ষণ আলাদা
// ═══════════════════════════════════════════
class RoomRepository
{
    public function __construct(private PDO $db) {}

    public function findAvailableRoom(RoomType $roomType): ?array
    {
        $type = $roomType->getTypeName();
        $stmt = $this->db->query("SELECT * FROM rooms WHERE type='$type' AND occupied=0 LIMIT 1");
        return $stmt->fetch() ?: null;
    }

    public function markOccupied(int $roomId): void
    {
        $this->db->prepare("UPDATE rooms SET occupied=1 WHERE id=?")->execute([$roomId]);
    }
}

// ═══════════════════════════════════════════
// ৬. Extract Class — বিল হিসাব আলাদা
// ═══════════════════════════════════════════
class BillingCalculator
{
    private const ADMISSION_FEE = 500;

    // বীমা ছাড় হার
    private const INSURANCE_RATES = [
        'full'    => 0.10,  // ৯০% কভারেজ, রোগী ১০% দেয়
        'partial' => 0.50,  // ৫০% কভারেজ
    ];

    public function calculate(
        Department $department,
        RoomType $roomType,
        ?string $insuranceType = null
    ): float {
        $consultationFee = $department->getConsultationFee();
        $roomCost = $roomType->getDailyCost();

        $totalBill = $consultationFee + $roomCost + self::ADMISSION_FEE;
        $totalBill = $this->applyInsuranceDiscount($totalBill, $insuranceType);

        return $totalBill;
    }

    private function applyInsuranceDiscount(float $amount, ?string $insuranceType): float
    {
        if ($insuranceType === null) return $amount;

        $rate = self::INSURANCE_RATES[$insuranceType] ?? 1.0;
        return $amount * $rate;
    }
}

// ═══════════════════════════════════════════
// ৭. Extract Class — রোগী সংরক্ষণ আলাদা
// ═══════════════════════════════════════════
class PatientRepository
{
    public function __construct(private PDO $db) {}

    public function save(PatientAdmissionRequest $request, int $doctorId, int $roomId): int
    {
        $stmt = $this->db->prepare(
            "INSERT INTO patients (name, age, gender, type, doctor_id, room_id) VALUES (?,?,?,?,?,?)"
        );
        $stmt->execute([
            $request->name,
            $request->age,
            $request->gender,
            $request->treatmentType,
            $doctorId,
            $roomId,
        ]);
        return (int) $this->db->lastInsertId();
    }
}

class BillRepository
{
    public function __construct(private PDO $db) {}

    public function save(int $patientId, float $amount): void
    {
        $this->db->prepare("INSERT INTO bills (patient_id, amount) VALUES (?, ?)")
            ->execute([$patientId, $amount]);
    }
}

// ═══════════════════════════════════════════
// ৮. Extract Class — নোটিফিকেশন আলাদা
// ═══════════════════════════════════════════
class PatientNotifier
{
    public function sendAdmissionNotification(string $name, float $bill): void
    {
        $message = "প্রিয় $name, আপনি সফলভাবে ভর্তি হয়েছেন। বিল: $bill টাকা।";
        // SMS/ইমেইল সার্ভিস কল
    }
}

// ═══════════════════════════════════════════
// ৯. Compose Method — মূল সমন্বয়কারী সার্ভিস
// ═══════════════════════════════════════════
class HospitalAdmissionService
{
    public function __construct(
        private PatientValidator $validator,
        private PatientRepository $patientRepo,
        private RoomRepository $roomRepo,
        private BillRepository $billRepo,
        private BillingCalculator $billing,
        private PatientNotifier $notifier,
        private PDO $db,
    ) {}

    // মূল মেথড — সমান abstraction level, পরিষ্কার প্রবাহ
    public function admitPatient(PatientAdmissionRequest $request): array
    {
        $this->validator->validate($request);

        $department = DepartmentFactory::create($request->treatmentType);
        $roomType = RoomTypeFactory::create($request->roomType);

        $doctor = $this->findDoctor($department);
        $room = $this->assignRoom($roomType);

        $patientId = $this->patientRepo->save($request, $doctor['id'], $room['id']);

        $bill = $this->billing->calculate($department, $roomType, $request->insuranceType);
        $this->billRepo->save($patientId, $bill);

        $this->notifier->sendAdmissionNotification($request->name, $bill);

        return [
            'patient_id' => $patientId,
            'bill'       => $bill,
            'room'       => $room['id'],
            'doctor'     => $doctor['name'],
        ];
    }

    private function findDoctor(Department $department): array
    {
        $doctor = $department->findAvailableDoctor($this->db);
        if (!$doctor) {
            throw new RuntimeException("কোনো ডাক্তার পাওয়া যায়নি");
        }
        return $doctor;
    }

    private function assignRoom(RoomType $roomType): array
    {
        $room = $this->roomRepo->findAvailableRoom($roomType);
        if (!$room) {
            throw new RuntimeException("কোনো রুম পাওয়া যায়নি");
        }
        $this->roomRepo->markOccupied($room['id']);
        return $room;
    }
}

// ═══════════════════════════════════════════
// ব্যবহার
// ═══════════════════════════════════════════
/*
$service = new HospitalAdmissionService(
    new PatientValidator(),
    new PatientRepository($db),
    new RoomRepository($db),
    new BillRepository($db),
    new BillingCalculator(),
    new PatientNotifier(),
    $db,
);

$request = new PatientAdmissionRequest(
    name: 'করিম সাহেব',
    age: 45,
    gender: 'M',
    treatmentType: 'cardiac',
    roomType: 'private',
    insuranceType: 'full',
);

$result = $service->admitPatient($request);
*/
```

### ✅ পরের কোড (JavaScript) — রিফ্যাক্টরড হাসপাতাল সিস্টেম

```javascript
// ✅ জাভাস্ক্রিপ্ট — রিফ্যাক্টরড হাসপাতাল ম্যানেজমেন্ট সিস্টেম

// ═══════════ Parameter Object ═══════════
class AdmissionRequest {
    constructor({ name, age, gender, treatmentType, roomType, insuranceType = null }) {
        this.name = name;
        this.age = age;
        this.gender = gender;
        this.treatmentType = treatmentType;
        this.roomType = roomType;
        this.insuranceType = insuranceType;
    }
}

// ═══════════ যাচাইকরণ ═══════════
class PatientValidator {
    static VALID_GENDERS = ["M", "F", "O"];

    validate(request) {
        if (!request.name?.trim()) throw new Error("রোগীর নাম দিন");
        if (request.age < 0 || request.age > 150) throw new Error("বয়স সীমার বাইরে");
        if (!PatientValidator.VALID_GENDERS.includes(request.gender)) {
            throw new Error("অবৈধ লিঙ্গ");
        }
    }
}

// ═══════════ বিভাগ — Strategy Pattern ═══════════
class CardiologyDept {
    get name() { return "cardiology"; }
    get consultationFee() { return 2000; }
}

class NeurologyDept {
    get name() { return "neurology"; }
    get consultationFee() { return 2500; }
}

class OrthopedicsDept {
    get name() { return "orthopedics"; }
    get consultationFee() { return 1500; }
}

class GeneralDept {
    get name() { return "general"; }
    get consultationFee() { return 800; }
}

const departmentMap = {
    cardiac: CardiologyDept,
    neuro: NeurologyDept,
    ortho: OrthopedicsDept,
    general: GeneralDept,
};

function createDepartment(type) {
    const Dept = departmentMap[type];
    if (!Dept) throw new Error(`অজানা বিভাগ: ${type}`);
    return new Dept();
}

// ═══════════ রুম — Strategy Pattern ═══════════
const ROOM_COSTS = Object.freeze({
    general: 500,
    semi_private: 1500,
    private: 3000,
    icu: 8000,
});

// ═══════════ বিলিং — Magic Number → Constant ═══════════
const BILLING = Object.freeze({
    ADMISSION_FEE: 500,
    INSURANCE: { full: 0.10, partial: 0.50 },
});

class BillingCalculator {
    calculate(department, roomType, insuranceType) {
        const roomCost = ROOM_COSTS[roomType] ?? 0;
        let total = department.consultationFee + roomCost + BILLING.ADMISSION_FEE;

        if (insuranceType && BILLING.INSURANCE[insuranceType]) {
            total *= BILLING.INSURANCE[insuranceType];
        }

        return total;
    }
}

// ═══════════ নোটিফিকেশন ═══════════
class PatientNotifier {
    sendAdmission(name, bill) {
        console.log(`📱 SMS: প্রিয় ${name}, ভর্তি সফল। বিল: ${bill} টাকা`);
    }
}

// ═══════════ মূল সার্ভিস — Compose Method ═══════════
class HospitalAdmissionService {
    #validator;
    #billing;
    #notifier;

    constructor() {
        this.#validator = new PatientValidator();
        this.#billing = new BillingCalculator();
        this.#notifier = new PatientNotifier();
    }

    // প্রতিটি ধাপ সমান abstraction level-এ
    async admitPatient(request) {
        this.#validator.validate(request);

        const department = createDepartment(request.treatmentType);
        const bill = this.#billing.calculate(department, request.roomType, request.insuranceType);

        const patientId = await this.#savePatient(request);
        this.#notifier.sendAdmission(request.name, bill);

        return { patientId, bill, department: department.name };
    }

    async #savePatient(request) {
        // ডেটাবেজে সংরক্ষণ
        const id = Math.floor(Math.random() * 10000);
        console.log(`রোগী সংরক্ষিত: ${request.name} (ID: ${id})`);
        return id;
    }
}

// ═══════════ ব্যবহার ═══════════
async function main() {
    const service = new HospitalAdmissionService();

    const result = await service.admitPatient(new AdmissionRequest({
        name: "করিম সাহেব",
        age: 45,
        gender: "M",
        treatmentType: "cardiac",
        roomType: "private",
        insuranceType: "full",
    }));

    console.log("ভর্তি ফলাফল:", result);
}

main();
```

---

## 📊 রিফ্যাক্টরিং আগে-পরে তুলনা সারণি

| বিষয় | আগে (❌) | পরে (✅) |
|--------|----------|---------|
| ক্লাসের সংখ্যা | ১টি God Class | ৮-১০টি ফোকাসড ক্লাস |
| মেথডের আকার | ১০০+ লাইন | ৫-১৫ লাইন |
| প্যারামিটার সংখ্যা | ৭+ | ১-২ (DTO সহ) |
| if/else শৃঙ্খল | ২০+ শাখা | Polymorphism |
| ম্যাজিক নম্বর | সরাসরি সংখ্যা | নামযুক্ত কনস্ট্যান্ট |
| টেস্টযোগ্যতা | কঠিন | সহজ (Mock করা যায়) |
| নতুন ফিচার যোগ | বিদ্যমান কোড বদলাতে হয় | নতুন ক্লাস যোগ করলেই হয় |
| নামকরণ | `proc`, `$t`, `$r` | `admitPatient`, `$treatmentType` |

---

## 📋 রিফ্যাক্টরিং চেকলিস্ট

| # | চেকলিস্ট আইটেম | বিবরণ | অবস্থা |
|---|----------------|--------|--------|
| ১ | টেস্ট আছে কি? | রিফ্যাক্টরিং-এর আগে অবশ্যই টেস্ট থাকতে হবে | ⬜ |
| ২ | ফাংশন ২০+ লাইন? | Extract Method প্রয়োগ করুন | ⬜ |
| ৩ | ৩+ প্যারামিটার? | Introduce Parameter Object | ⬜ |
| ৪ | ভ্যারিয়েবল নাম অর্থহীন? | Rename Variable/Method | ⬜ |
| ৫ | switch/if-else টাইপ চেক? | Replace Conditional with Polymorphism | ⬜ |
| ৬ | ম্যাজিক নম্বর আছে? | Replace Magic Number with Constant | ⬜ |
| ৭ | God Class আছে? | Extract Class — SRP অনুসরণ করুন | ⬜ |
| ৮ | Feature Envy? | Move Method সঠিক ক্লাসে | ⬜ |
| ৯ | টেম্প ভ্যারিয়েবল অনেক? | Replace Temp with Query | ⬜ |
| ১০ | মিশ্রিত abstraction level? | Compose Method | ⬜ |
| ১১ | ভুল inheritance? | Replace Inheritance with Composition | ⬜ |
| ১২ | ডুপ্লিকেট কোড? | Extract Method / Extract Class | ⬜ |
| ১৩ | রিফ্যাক্টরিং-এর পর টেস্ট পাস? | সব টেস্ট সবুজ হতে হবে | ⬜ |
| ১৪ | কোড রিভিউ হয়েছে? | দলের সদস্যদের মতামত নিন | ⬜ |
| ১৫ | ডকুমেন্টেশন আপডেট? | পরিবর্তন অনুযায়ী ডক আপডেট | ⬜ |

---

## ⚠️ কখন রিফ্যাক্টরিং করবেন না

সবসময় রিফ্যাক্টরিং করা উচিত নয়। নিচের পরিস্থিতিতে **সাবধান** হোন:

```
রিফ্যাক্টরিং এড়ানোর পরিস্থিতি:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ❌ ১. ডেডলাইন খুব কাছে                                        │
│     → শিপ করুন, পরে রিফ্যাক্টর করুন                             │
│     → "প্রথমে কাজ করাও, তারপর সুন্দর করো"                      │
│                                                                │
│  ❌ ২. টেস্ট নেই                                                │
│     → আগে টেস্ট লিখুন, তারপর রিফ্যাক্টর করুন                   │
│     → টেস্ট ছাড়া রিফ্যাক্টরিং = অন্ধকারে হাঁটা                 │
│                                                                │
│  ❌ ৩. পুনর্লিখন (Rewrite) দরকার                                │
│     → কোড এতটাই খারাপ যে ঠিক করার চেয়ে                        │
│       নতুন করে লেখা সহজ                                        │
│                                                                │
│  ❌ ৪. কোড শীঘ্রই বাতিল হবে                                     │
│     → যদি ফিচার/মডিউল বন্ধ হওয়ার পরিকল্পনা থাকে                │
│     → পরিত্যক্ত কোড রিফ্যাক্টর করা সময়ের অপচয়                 │
│                                                                │
│  ❌ ৫. প্রোডাকশন ক্রিটিক্যাল সময়                                │
│     → বড় সেল/ইভেন্টের আগে রিফ্যাক্টরিং নয়                    │
│     → স্থিতিশীলতা > পরিচ্ছন্নতা                                │
│                                                                │
│  ❌ ৬. শুধু "সুন্দর" করতে                                        │
│     → রিফ্যাক্টরিং-এর পেছনে ব্যবসায়িক কারণ থাকা উচিত          │
│     → "পরিবর্তনযোগ্যতা বাড়ানো" হলো ভালো কারণ                  │
│     → "আমার পছন্দের প্যাটার্ন ব্যবহার করা" ভালো কারণ নয়       │
│                                                                │
│  ❌ ৭. দলের সম্মতি ছাড়া                                         │
│     → বড় রিফ্যাক্টরিং দলগত সিদ্ধান্ত হওয়া উচিত                │
│     → একা করলে merge conflict এবং বিভ্রান্তি হবে               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### রিফ্যাক্টরিং-এর সেরা সময়

| সময় | বিবরণ |
|------|--------|
| নতুন ফিচারের আগে | ফিচার যোগ করতে গেলে পুরনো কোড পরিষ্কার করুন |
| বাগ ফিক্সের সময় | বাগের মূল কারণ জটিল কোড হলে রিফ্যাক্টর করুন |
| কোড রিভিউতে | রিভিউয়ার-এর পরামর্শ অনুযায়ী |
| বোঝার সময় | কোড বুঝতে গেলে বুঝে পরিষ্কার করুন |
| Sprint-এর শেষে | অবশিষ্ট সময়ে টেকনিক্যাল ডেট কমান |

---

## 🎯 সারসংক্ষেপ

```
রিফ্যাক্টরিং কৌশলের মূলমন্ত্র:

    ছোট ছোট পদক্ষেপে পরিবর্তন করুন
              │
              ▼
    প্রতিটি পদক্ষেপের পর টেস্ট চালান
              │
              ▼
    বাহ্যিক আচরণ অপরিবর্তিত রাখুন
              │
              ▼
    কোড পরিষ্কার, পাঠযোগ্য ও রক্ষণাবেক্ষণযোগ্য হবে ✅
```

| কৌশল | এক কথায় |
|------|---------|
| Extract Method | বড় → ছোট ছোট ফাংশন |
| Rename | অর্থহীন → অর্থবহ নাম |
| Move Method | ভুল জায়গা → সঠিক জায়গা |
| Replace Conditional | if/switch → Polymorphism |
| Parameter Object | অনেক প্যারামিটার → ১টি অবজেক্ট |
| Magic Number → Const | ৫০০ → MAX_DISCOUNT_LIMIT |
| Extract Class | God Class → একাধিক ছোট ক্লাস |
| Temp → Query | ভ্যারিয়েবল → মেথড |
| Compose Method | মিশ্রিত → সমান level |
| Composition > Inheritance | is-a → has-a |

---

## 🔗 পরবর্তী টপিক

> পরবর্তী অধ্যায়ে আমরা শিখব **DRY, KISS, YAGNI নীতিমালা** — যেগুলো রিফ্যাক্টরিং-এর সময় সিদ্ধান্ত নিতে সাহায্য করে।
>
> ➡️ [DRY, KISS, YAGNI নীতিমালা](./dry-kiss-yagni.md)
