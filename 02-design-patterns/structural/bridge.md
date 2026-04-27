# 🌉 Bridge প্যাটার্ন (ব্রিজ ডিজাইন প্যাটার্ন)

## 📌 সংজ্ঞা

> **GoF Definition:** *"Decouple an abstraction from its implementation so that the two can vary independently."*

Bridge প্যাটার্ন হলো একটি **Structural Design Pattern** যা একটি বৃহৎ ক্লাসকে বা ঘনিষ্ঠভাবে সম্পর্কিত ক্লাসগুলোর একটি সেটকে দুটি পৃথক হায়ারার্কিতে ভাগ করে — **Abstraction** এবং **Implementation** — যেগুলো একে অপরের থেকে স্বাধীনভাবে বিকশিত হতে পারে।

### মূল ধারণা

ধরুন আপনার কাছে একটি `Shape` ক্লাস আছে এবং এটি `Circle`, `Square` ইত্যাদি সাবক্লাস দিয়ে extend করা হয়েছে। এখন আপনি চান প্রতিটি shape-কে বিভিন্ন রঙে render করতে — `Red`, `Blue`। সাধারণ inheritance-এ আপনাকে `RedCircle`, `BlueCircle`, `RedSquare`, `BlueSquare` — এরকম **n × m** সংখ্যক ক্লাস বানাতে হবে। এটা হলো **class explosion** সমস্যা।

Bridge প্যাটার্ন এই সমস্যার সমাধান করে **composition over inheritance** নীতি প্রয়োগ করে। এটি একটি dimension-কে আলাদা ক্লাস হায়ারার্কিতে নিয়ে যায় এবং মূল ক্লাস থেকে সেটিকে **reference** হিসেবে ব্যবহার করে।

### চারটি মূল উপাদান

| উপাদান | ভূমিকা |
|---|---|
| **Abstraction** | উচ্চ-স্তরের control logic ধারণ করে; Implementation-এর reference রাখে |
| **Refined Abstraction** | Abstraction-এর বিশেষায়িত সংস্করণ |
| **Implementor (Interface)** | Implementation হায়ারার্কির জন্য interface সংজ্ঞায়িত করে |
| **Concrete Implementor** | Implementor interface-এর নির্দিষ্ট বাস্তবায়ন |

**সারকথা:** Abstraction এবং Implementation-এর মধ্যে একটি **সেতু** (bridge) তৈরি করা হয় যেখানে দুটোই স্বাধীনভাবে পরিবর্তিত হতে পারে।

---

## 🏠 বাস্তব উদাহরণ — রিমোট কন্ট্রোল ও টিভি

বাংলাদেশের প্রতিটি বাসায় টিভি আছে। ধরুন:

- **টিভি ব্র্যান্ড:** Samsung, Walton, Sony
- **রিমোট কন্ট্রোল ধরন:** Basic Remote (শুধু on/off, channel), Advanced Remote (volume control, mute, smart features)

এখন, যেকোনো রিমোট যেকোনো টিভির সাথে কাজ করতে পারে — এটাই Bridge প্যাটার্নের মূল ধারণা।

```
রিমোট (Abstraction)          টিভি (Implementation)
├── Basic Remote              ├── Samsung TV
└── Advanced Remote           ├── Walton TV
                              └── Sony TV
```

রিমোট টিভিকে "জানে" একটি interface-এর মাধ্যমে। রিমোট বলে "volume বাড়াও" — কীভাবে বাড়াবে সেটা টিভির নিজের ব্যাপার। Samsung-এ ভিন্নভাবে কাজ করে, Walton-এ ভিন্নভাবে। রিমোটের এটা নিয়ে মাথাব্যথা নেই।

---

## 📊 UML ডায়াগ্রাম

```
    ┌─────────────────────┐         ┌─────────────────────────┐
    │    Abstraction       │         │   Implementor           │
    │─────────────────────│         │  «interface»            │
    │ - impl: Implementor │────────▶│─────────────────────────│
    │─────────────────────│         │ + operationImpl(): void │
    │ + operation(): void  │         └───────────┬─────────────┘
    └──────────┬──────────┘                     │
               │                      ┌─────────┴──────────┐
    ┌──────────┴──────────┐  ┌────────┴───────┐ ┌──────────┴──────┐
    │ RefinedAbstraction   │  │ ConcreteImplA  │ │ ConcreteImplB   │
    │─────────────────────│  │────────────────│ │─────────────────│
    │ + operation(): void  │  │ + operationImpl│ │ + operationImpl │
    └──────────────────────┘  └────────────────┘ └─────────────────┘

    ═══════════════════════════════════════════════════════════════
    Composition (─────▶) = Bridge! Abstraction "has-a" Implementor
    ═══════════════════════════════════════════════════════════════
```

**ডেটা ফ্লো:**
```
Client → Abstraction.operation()
              │
              ├── নিজের logic চালায়
              └── impl.operationImpl() কল করে
                       │
                       └── ConcreteImplementor নিজের মতো execute করে
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Bridge — রিমোট কন্ট্রোল ও টিভি

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Implementor Interface — টিভি ডিভাইসের জন্য contract
interface Device
{
    public function isEnabled(): bool;
    public function enable(): void;
    public function disable(): void;
    public function getVolume(): int;
    public function setVolume(int $volume): void;
    public function getChannel(): int;
    public function setChannel(int $channel): void;
    public function getName(): string;
}

// Concrete Implementor A — Samsung TV
class SamsungTV implements Device
{
    private bool $on = false;
    private int $volume = 30;
    private int $channel = 1;

    public function isEnabled(): bool { return $this->on; }
    public function enable(): void { $this->on = true; }
    public function disable(): void { $this->on = false; }
    public function getVolume(): int { return $this->volume; }

    public function setVolume(int $volume): void
    {
        $this->volume = max(0, min(100, $volume));
    }

    public function getChannel(): int { return $this->channel; }

    public function setChannel(int $channel): void
    {
        $this->channel = max(1, $channel);
    }

    public function getName(): string { return 'Samsung Smart TV'; }
}

// Concrete Implementor B — Walton TV
class WaltonTV implements Device
{
    private bool $on = false;
    private int $volume = 20;
    private int $channel = 1;

    public function isEnabled(): bool { return $this->on; }
    public function enable(): void { $this->on = true; }
    public function disable(): void { $this->on = false; }
    public function getVolume(): int { return $this->volume; }

    public function setVolume(int $volume): void
    {
        $this->volume = max(0, min(80, $volume)); // Walton max volume 80
    }

    public function getChannel(): int { return $this->channel; }

    public function setChannel(int $channel): void
    {
        $this->channel = max(1, min(200, $channel));
    }

    public function getName(): string { return 'Walton Ultima TV'; }
}

// Abstraction — Remote Control
class RemoteControl
{
    public function __construct(
        protected Device $device
    ) {}

    public function togglePower(): void
    {
        if ($this->device->isEnabled()) {
            $this->device->disable();
        } else {
            $this->device->enable();
        }
    }

    public function volumeUp(): void
    {
        $this->device->setVolume($this->device->getVolume() + 10);
    }

    public function volumeDown(): void
    {
        $this->device->setVolume($this->device->getVolume() - 10);
    }

    public function channelUp(): void
    {
        $this->device->setChannel($this->device->getChannel() + 1);
    }

    public function channelDown(): void
    {
        $this->device->setChannel($this->device->getChannel() - 1);
    }
}

// Refined Abstraction — Advanced Remote (mute, device info)
class AdvancedRemote extends RemoteControl
{
    public function mute(): void
    {
        $this->device->setVolume(0);
    }

    public function getDeviceInfo(): string
    {
        return sprintf(
            "📺 %s | Power: %s | Volume: %d | Channel: %d",
            $this->device->getName(),
            $this->device->isEnabled() ? 'ON' : 'OFF',
            $this->device->getVolume(),
            $this->device->getChannel()
        );
    }
}

// ব্যবহার
$samsung = new SamsungTV();
$walton = new WaltonTV();

$basicRemote = new RemoteControl($samsung);
$basicRemote->togglePower();
$basicRemote->volumeUp();

$advancedRemote = new AdvancedRemote($walton);
$advancedRemote->togglePower();
$advancedRemote->volumeUp();
$advancedRemote->volumeUp();
echo $advancedRemote->getDeviceInfo();
// 📺 Walton Ultima TV | Power: ON | Volume: 40 | Channel: 1
$advancedRemote->mute();
echo $advancedRemote->getDeviceInfo();
// 📺 Walton Ultima TV | Power: ON | Volume: 0 | Channel: 1
```

#### JavaScript ES2022+

```javascript
// Implementor Interface — Symbol ব্যবহার করে private method simulate
class Device {
    isEnabled() { throw new Error('Must implement'); }
    enable() { throw new Error('Must implement'); }
    disable() { throw new Error('Must implement'); }
    getVolume() { throw new Error('Must implement'); }
    setVolume(_vol) { throw new Error('Must implement'); }
    getChannel() { throw new Error('Must implement'); }
    setChannel(_ch) { throw new Error('Must implement'); }
    getName() { throw new Error('Must implement'); }
}

// Concrete Implementor A
class SamsungTV extends Device {
    #on = false;
    #volume = 30;
    #channel = 1;

    isEnabled() { return this.#on; }
    enable() { this.#on = true; }
    disable() { this.#on = false; }
    getVolume() { return this.#volume; }
    setVolume(vol) { this.#volume = Math.max(0, Math.min(100, vol)); }
    getChannel() { return this.#channel; }
    setChannel(ch) { this.#channel = Math.max(1, ch); }
    getName() { return 'Samsung Smart TV'; }
}

// Concrete Implementor B
class WaltonTV extends Device {
    #on = false;
    #volume = 20;
    #channel = 1;

    isEnabled() { return this.#on; }
    enable() { this.#on = true; }
    disable() { this.#on = false; }
    getVolume() { return this.#volume; }
    setVolume(vol) { this.#volume = Math.max(0, Math.min(80, vol)); }
    getChannel() { return this.#channel; }
    setChannel(ch) { this.#channel = Math.max(1, Math.min(200, ch)); }
    getName() { return 'Walton Ultima TV'; }
}

// Abstraction
class RemoteControl {
    #device;
    constructor(device) { this.#device = device; }

    get device() { return this.#device; }

    togglePower() {
        this.#device.isEnabled() ? this.#device.disable() : this.#device.enable();
    }

    volumeUp() { this.#device.setVolume(this.#device.getVolume() + 10); }
    volumeDown() { this.#device.setVolume(this.#device.getVolume() - 10); }
    channelUp() { this.#device.setChannel(this.#device.getChannel() + 1); }
    channelDown() { this.#device.setChannel(this.#device.getChannel() - 1); }
}

// Refined Abstraction
class AdvancedRemote extends RemoteControl {
    mute() { this.device.setVolume(0); }

    getDeviceInfo() {
        const d = this.device;
        return `📺 ${d.getName()} | Power: ${d.isEnabled() ? 'ON' : 'OFF'} | Vol: ${d.getVolume()} | Ch: ${d.getChannel()}`;
    }
}

// ব্যবহার
const samsung = new SamsungTV();
const remote = new AdvancedRemote(samsung);
remote.togglePower();
remote.volumeUp();
console.log(remote.getDeviceInfo());
// 📺 Samsung Smart TV | Power: ON | Vol: 40 | Ch: 1
```

---

### 2️⃣ Notification System — NotificationType × Channel

এটি একটি বাস্তবসম্মত উদাহরণ। বাংলাদেশের একটি ই-কমার্স সাইটে notification পাঠাতে হবে:

- **Notification Type (Abstraction):** Urgent, Normal, Scheduled
- **Channel (Implementation):** Email, SMS, Push Notification

Bridge ছাড়া হলে: `UrgentEmail`, `UrgentSMS`, `UrgentPush`, `NormalEmail`... = **9টি ক্লাস!**
Bridge দিয়ে: **3 + 3 = 6টি ক্লাস।** ভবিষ্যতে নতুন channel যোগ করলে শুধু 1টি ক্লাস যোগ হবে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Implementation Interface — Notification Channel
interface NotificationChannel
{
    public function send(string $recipient, string $title, string $body): bool;
    public function getChannelName(): string;
}

// Concrete Implementation — Email
class EmailChannel implements NotificationChannel
{
    public function __construct(
        private readonly string $smtpHost = 'smtp.gmail.com',
        private readonly int $smtpPort = 587,
    ) {}

    public function send(string $recipient, string $title, string $body): bool
    {
        // বাস্তবে এখানে SMTP logic থাকবে
        echo "📧 Email → {$recipient}: [{$title}] {$body}\n";
        return true;
    }

    public function getChannelName(): string { return 'Email'; }
}

// Concrete Implementation — SMS (বাংলাদেশি SMS Gateway)
class SmsChannel implements NotificationChannel
{
    public function __construct(
        private readonly string $apiKey,
        private readonly string $gateway = 'bulksmsbd.com',
    ) {}

    public function send(string $recipient, string $title, string $body): bool
    {
        $message = "[{$title}] {$body}";
        // বাংলাদেশি নম্বর ফরম্যাট: +880XXXXXXXXXX
        if (!str_starts_with($recipient, '+880')) {
            throw new \InvalidArgumentException('বাংলাদেশি নম্বর +880 দিয়ে শুরু হতে হবে');
        }
        echo "📱 SMS → {$recipient}: {$message}\n";
        return true;
    }

    public function getChannelName(): string { return 'SMS'; }
}

// Concrete Implementation — Push Notification
class PushChannel implements NotificationChannel
{
    public function __construct(
        private readonly string $firebaseKey,
    ) {}

    public function send(string $recipient, string $title, string $body): bool
    {
        echo "🔔 Push → Device({$recipient}): [{$title}] {$body}\n";
        return true;
    }

    public function getChannelName(): string { return 'Push Notification'; }
}

// Abstraction — Notification
abstract class Notification
{
    protected array $channels = [];

    public function addChannel(NotificationChannel $channel): static
    {
        $this->channels[] = $channel;
        return $this;
    }

    abstract public function notify(string $recipient, string $message): void;
    abstract public function getType(): string;
}

// Refined Abstraction — Urgent Notification
class UrgentNotification extends Notification
{
    public function notify(string $recipient, string $message): void
    {
        $title = "🚨 জরুরি";

        // জরুরি notification সব channel-এ পাঠায়
        foreach ($this->channels as $channel) {
            $channel->send($recipient, $title, $message);
        }
    }

    public function getType(): string { return 'Urgent'; }
}

// Refined Abstraction — Normal Notification
class NormalNotification extends Notification
{
    public function notify(string $recipient, string $message): void
    {
        $title = "📌 বিজ্ঞপ্তি";

        // Normal notification শুধু প্রথম channel-এ পাঠায়
        if (!empty($this->channels)) {
            $this->channels[0]->send($recipient, $title, $message);
        }
    }

    public function getType(): string { return 'Normal'; }
}

// Refined Abstraction — Scheduled Notification
class ScheduledNotification extends Notification
{
    public function __construct(
        private readonly \DateTimeImmutable $scheduledAt,
    ) {}

    public function notify(string $recipient, string $message): void
    {
        $title = "⏰ নির্ধারিত ({$this->scheduledAt->format('h:i A')})";

        foreach ($this->channels as $channel) {
            echo "⏳ {$channel->getChannelName()} scheduled for {$this->scheduledAt->format('Y-m-d h:i A')}\n";
            $channel->send($recipient, $title, $message);
        }
    }

    public function getType(): string { return 'Scheduled'; }
}

// ব্যবহার — বাংলাদেশি ই-কমার্স অর্ডার notification
$email = new EmailChannel();
$sms = new SmsChannel(apiKey: 'bd-sms-key-123');
$push = new PushChannel(firebaseKey: 'fcm-key-456');

// অর্ডার কনফার্মেশন — Urgent, সব channel-এ
$urgent = new UrgentNotification();
$urgent->addChannel($email)->addChannel($sms)->addChannel($push);
$urgent->notify('+8801712345678', 'আপনার অর্ডার #BD-2024-001 কনফার্ম হয়েছে!');

echo "\n---\n\n";

// ডেলিভারি আপডেট — Normal, শুধু push
$normal = new NormalNotification();
$normal->addChannel($push);
$normal->notify('device_token_xyz', 'আপনার পার্সেল রাইডারের কাছে আছে।');
```

#### JavaScript ES2022+

```javascript
// Implementation Interface
class NotificationChannel {
    send(recipient, title, body) { throw new Error('Must implement'); }
    get channelName() { throw new Error('Must implement'); }
}

class EmailChannel extends NotificationChannel {
    #smtpHost;
    constructor(smtpHost = 'smtp.gmail.com') { super(); this.#smtpHost = smtpHost; }

    send(recipient, title, body) {
        console.log(`📧 Email → ${recipient}: [${title}] ${body}`);
        return true;
    }
    get channelName() { return 'Email'; }
}

class SmsChannel extends NotificationChannel {
    #apiKey;
    constructor(apiKey) { super(); this.#apiKey = apiKey; }

    send(recipient, title, body) {
        if (!recipient.startsWith('+880')) {
            throw new Error('বাংলাদেশি নম্বর +880 দিয়ে শুরু হতে হবে');
        }
        console.log(`📱 SMS → ${recipient}: [${title}] ${body}`);
        return true;
    }
    get channelName() { return 'SMS'; }
}

class PushChannel extends NotificationChannel {
    send(recipient, title, body) {
        console.log(`🔔 Push → ${recipient}: [${title}] ${body}`);
        return true;
    }
    get channelName() { return 'Push'; }
}

// Abstraction
class Notification {
    #channels = [];
    addChannel(channel) { this.#channels.push(channel); return this; }
    get channels() { return this.#channels; }
    notify(recipient, message) { throw new Error('Must implement'); }
}

class UrgentNotification extends Notification {
    notify(recipient, message) {
        this.channels.forEach(ch => ch.send(recipient, '🚨 জরুরি', message));
    }
}

class NormalNotification extends Notification {
    notify(recipient, message) {
        this.channels[0]?.send(recipient, '📌 বিজ্ঞপ্তি', message);
    }
}

// ব্যবহার
const urgent = new UrgentNotification();
urgent
    .addChannel(new EmailChannel())
    .addChannel(new SmsChannel('bd-key'))
    .addChannel(new PushChannel());

urgent.notify('+8801712345678', 'অর্ডার কনফার্ম হয়েছে!');
```

---

### 3️⃣ Payment Processing — PaymentMethod × Gateway

বাংলাদেশের পেমেন্ট ইকোসিস্টেম অত্যন্ত সমৃদ্ধ — bKash, Nagad, Rocket, কার্ড পেমেন্ট সব একসাথে handle করতে হয়। Bridge প্যাটার্ন এখানে চমৎকারভাবে কাজ করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Implementation Interface — Payment Gateway
interface PaymentGateway
{
    public function processPayment(float $amount, string $currency, array $metadata): PaymentResult;
    public function refund(string $transactionId, float $amount): bool;
    public function getGatewayName(): string;
}

// Value Object — Payment Result
readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public string $gateway,
        public float $amount,
        public string $message,
    ) {}
}

// Concrete Implementation — Stripe Gateway
class StripeGateway implements PaymentGateway
{
    public function __construct(
        private readonly string $secretKey,
    ) {}

    public function processPayment(float $amount, string $currency, array $metadata): PaymentResult
    {
        $txnId = 'stripe_' . bin2hex(random_bytes(8));
        // বাস্তবে Stripe API কল হবে
        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            gateway: 'Stripe',
            amount: $amount,
            message: "Stripe payment successful (${currency} {$amount})"
        );
    }

    public function refund(string $transactionId, float $amount): bool
    {
        echo "💰 Stripe Refund: {$transactionId} → ৳{$amount}\n";
        return true;
    }

    public function getGatewayName(): string { return 'Stripe'; }
}

// Concrete Implementation — bKash Gateway
class BkashGateway implements PaymentGateway
{
    public function __construct(
        private readonly string $appKey,
        private readonly string $appSecret,
    ) {}

    public function processPayment(float $amount, string $currency, array $metadata): PaymentResult
    {
        if ($currency !== 'BDT') {
            return new PaymentResult(false, '', 'bKash', $amount, 'bKash শুধু BDT সাপোর্ট করে');
        }

        if ($amount > 25000) {
            return new PaymentResult(false, '', 'bKash', $amount, 'bKash সর্বোচ্চ ৳25,000 পর্যন্ত');
        }

        $txnId = 'bkash_' . bin2hex(random_bytes(8));
        $phone = $metadata['phone'] ?? 'unknown';

        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            gateway: 'bKash',
            amount: $amount,
            message: "bKash payment from {$phone}: ৳{$amount}"
        );
    }

    public function refund(string $transactionId, float $amount): bool
    {
        echo "💰 bKash Refund: {$transactionId} → ৳{$amount}\n";
        return true;
    }

    public function getGatewayName(): string { return 'bKash'; }
}

// Concrete Implementation — Nagad Gateway
class NagadGateway implements PaymentGateway
{
    public function __construct(
        private readonly string $merchantId,
    ) {}

    public function processPayment(float $amount, string $currency, array $metadata): PaymentResult
    {
        if ($currency !== 'BDT') {
            return new PaymentResult(false, '', 'Nagad', $amount, 'Nagad শুধু BDT সাপোর্ট করে');
        }

        $txnId = 'nagad_' . bin2hex(random_bytes(8));
        return new PaymentResult(
            success: true,
            transactionId: $txnId,
            gateway: 'Nagad',
            amount: $amount,
            message: "Nagad payment: ৳{$amount}"
        );
    }

    public function refund(string $transactionId, float $amount): bool
    {
        echo "💰 Nagad Refund: {$transactionId} → ৳{$amount}\n";
        return true;
    }

    public function getGatewayName(): string { return 'Nagad'; }
}

// Abstraction — Payment Method
abstract class PaymentMethod
{
    public function __construct(
        protected PaymentGateway $gateway,
    ) {}

    abstract public function pay(float $amount, string $currency, array $metadata = []): PaymentResult;

    // Runtime-এ gateway switch করার ক্ষমতা — Bridge-এর শক্তি!
    public function switchGateway(PaymentGateway $gateway): void
    {
        $this->gateway = $gateway;
    }

    public function getGateway(): string
    {
        return $this->gateway->getGatewayName();
    }
}

// Refined Abstraction — Credit Card Payment
class CreditCardPayment extends PaymentMethod
{
    public function pay(float $amount, string $currency, array $metadata = []): PaymentResult
    {
        // ক্রেডিট কার্ডের জন্য অতিরিক্ত validation
        if (!isset($metadata['card_number'])) {
            return new PaymentResult(false, '', $this->getGateway(), $amount, 'কার্ড নম্বর প্রয়োজন');
        }

        // Surcharge যোগ (2.5% credit card fee)
        $totalAmount = $amount * 1.025;

        echo "💳 Credit Card Payment: ৳{$amount} + ৳" . round($amount * 0.025, 2) . " fee\n";
        return $this->gateway->processPayment($totalAmount, $currency, $metadata);
    }
}

// Refined Abstraction — Mobile Banking Payment (bKash/Nagad)
class MobileBankingPayment extends PaymentMethod
{
    public function pay(float $amount, string $currency, array $metadata = []): PaymentResult
    {
        if (!isset($metadata['phone'])) {
            return new PaymentResult(false, '', $this->getGateway(), $amount, 'মোবাইল নম্বর প্রয়োজন');
        }

        // মোবাইল ব্যাংকিং-এ সর্বদা BDT
        echo "📱 Mobile Banking ({$this->getGateway()}): ৳{$amount}\n";
        return $this->gateway->processPayment($amount, 'BDT', $metadata);
    }
}

// ব্যবহার — বাংলাদেশি ই-কমার্স checkout
$stripe = new StripeGateway('sk_test_xxx');
$bkash = new BkashGateway('bkash_app_key', 'bkash_app_secret');
$nagad = new NagadGateway('nagad_merchant_001');

// ক্রেডিট কার্ডে Stripe দিয়ে পেমেন্ট
$cardPayment = new CreditCardPayment($stripe);
$result = $cardPayment->pay(5000, 'BDT', ['card_number' => '4242424242424242']);
echo "✅ {$result->message}\n\n";

// bKash দিয়ে মোবাইল পেমেন্ট
$mobilePayment = new MobileBankingPayment($bkash);
$result = $mobilePayment->pay(1500, 'BDT', ['phone' => '+8801712345678']);
echo "✅ {$result->message}\n\n";

// Runtime-এ gateway switch — bKash থেকে Nagad-এ!
$mobilePayment->switchGateway($nagad);
$result = $mobilePayment->pay(2000, 'BDT', ['phone' => '+8801812345678']);
echo "✅ {$result->message}\n";
```

#### JavaScript ES2022+

```javascript
// Implementation — Payment Gateway
class PaymentGateway {
    processPayment(amount, currency, metadata) { throw new Error('Must implement'); }
    refund(txnId, amount) { throw new Error('Must implement'); }
    get name() { throw new Error('Must implement'); }
}

class BkashGateway extends PaymentGateway {
    #appKey;
    constructor(appKey) { super(); this.#appKey = appKey; }

    processPayment(amount, currency, metadata) {
        if (currency !== 'BDT') return { success: false, message: 'bKash শুধু BDT সাপোর্ট করে' };
        if (amount > 25000) return { success: false, message: 'সর্বোচ্চ ৳25,000' };

        return {
            success: true,
            transactionId: `bkash_${crypto.randomUUID().slice(0, 8)}`,
            gateway: 'bKash',
            amount,
            message: `bKash: ৳${amount} from ${metadata.phone}`,
        };
    }

    refund(txnId, amount) {
        console.log(`💰 bKash Refund: ${txnId} → ৳${amount}`);
        return true;
    }

    get name() { return 'bKash'; }
}

class NagadGateway extends PaymentGateway {
    processPayment(amount, currency, metadata) {
        return {
            success: true,
            transactionId: `nagad_${crypto.randomUUID().slice(0, 8)}`,
            gateway: 'Nagad',
            amount,
            message: `Nagad: ৳${amount}`,
        };
    }
    refund(txnId, amount) { return true; }
    get name() { return 'Nagad'; }
}

// Abstraction — Payment Method
class PaymentMethod {
    #gateway;
    constructor(gateway) { this.#gateway = gateway; }

    get gateway() { return this.#gateway; }
    switchGateway(gw) { this.#gateway = gw; }
    pay(amount, currency, metadata = {}) { throw new Error('Must implement'); }
}

class MobileBankingPayment extends PaymentMethod {
    pay(amount, currency, metadata = {}) {
        if (!metadata.phone) return { success: false, message: 'মোবাইল নম্বর দিন' };
        console.log(`📱 ${this.gateway.name}: ৳${amount}`);
        return this.gateway.processPayment(amount, 'BDT', metadata);
    }
}

// ব্যবহার
const payment = new MobileBankingPayment(new BkashGateway('key'));
console.log(payment.pay(1500, 'BDT', { phone: '+8801712345678' }));

// Runtime switch
payment.switchGateway(new NagadGateway());
console.log(payment.pay(2000, 'BDT', { phone: '+8801812345678' }));
```

---

## 🌍 Real-World Applicable Areas

### 1. Cross-Platform Rendering — Theme × Platform

```php
<?php

// Implementor — Theme
interface Theme
{
    public function getPrimaryColor(): string;
    public function getBackgroundColor(): string;
    public function getFontFamily(): string;
}

class LightTheme implements Theme
{
    public function getPrimaryColor(): string { return '#1a73e8'; }
    public function getBackgroundColor(): string { return '#ffffff'; }
    public function getFontFamily(): string { return 'SolaimanLipi, sans-serif'; }
}

class DarkTheme implements Theme
{
    public function getPrimaryColor(): string { return '#8ab4f8'; }
    public function getBackgroundColor(): string { return '#1e1e1e'; }
    public function getFontFamily(): string { return 'SolaimanLipi, sans-serif'; }
}

// Abstraction — UI Renderer
abstract class UIRenderer
{
    public function __construct(protected Theme $theme) {}

    abstract public function renderButton(string $text): string;
    abstract public function renderCard(string $title, string $body): string;
}

class WebRenderer extends UIRenderer
{
    public function renderButton(string $text): string
    {
        $bg = $this->theme->getPrimaryColor();
        $font = $this->theme->getFontFamily();
        return "<button style=\"background:{$bg};font-family:{$font}\">{$text}</button>";
    }

    public function renderCard(string $title, string $body): string
    {
        $bgColor = $this->theme->getBackgroundColor();
        return "<div class=\"card\" style=\"background:{$bgColor}\"><h3>{$title}</h3><p>{$body}</p></div>";
    }
}

class MobileRenderer extends UIRenderer
{
    public function renderButton(string $text): string
    {
        return json_encode([
            'type' => 'Button',
            'text' => $text,
            'style' => ['backgroundColor' => $this->theme->getPrimaryColor()],
        ]);
    }

    public function renderCard(string $title, string $body): string
    {
        return json_encode([
            'type' => 'Card',
            'title' => $title,
            'body' => $body,
            'style' => ['backgroundColor' => $this->theme->getBackgroundColor()],
        ]);
    }
}

// যেকোনো renderer + যেকোনো theme
$webDark = new WebRenderer(new DarkTheme());
echo $webDark->renderButton('কিনুন');

$mobilLight = new MobileRenderer(new LightTheme());
echo $mobilLight->renderCard('পণ্য', 'বাংলাদেশি পণ্য কিনুন');
```

### 2. Report Generation — Format × DataSource

```php
<?php

// Implementor — Data Source
interface ReportDataSource
{
    public function fetchData(string $query): array;
    public function getSourceName(): string;
}

class MySQLDataSource implements ReportDataSource
{
    public function fetchData(string $query): array
    {
        // বাস্তবে MySQL query run হবে
        return [
            ['name' => 'রহিম', 'sales' => 50000],
            ['name' => 'করিম', 'sales' => 75000],
        ];
    }
    public function getSourceName(): string { return 'MySQL'; }
}

class ApiDataSource implements ReportDataSource
{
    public function fetchData(string $query): array
    {
        // বাস্তবে API call হবে
        return [
            ['name' => 'ঢাকা শাখা', 'sales' => 500000],
            ['name' => 'চট্টগ্রাম শাখা', 'sales' => 350000],
        ];
    }
    public function getSourceName(): string { return 'REST API'; }
}

// Abstraction — Report Format
abstract class Report
{
    public function __construct(protected ReportDataSource $dataSource) {}

    abstract public function generate(string $query): string;
}

class PdfReport extends Report
{
    public function generate(string $query): string
    {
        $data = $this->dataSource->fetchData($query);
        $source = $this->dataSource->getSourceName();
        return "📄 PDF Report (Source: {$source})\n" . print_r($data, true);
    }
}

class ExcelReport extends Report
{
    public function generate(string $query): string
    {
        $data = $this->dataSource->fetchData($query);
        $source = $this->dataSource->getSourceName();
        $csv = implode("\n", array_map(fn($row) => implode(',', $row), $data));
        return "📊 Excel Report (Source: {$source})\n{$csv}";
    }
}

// যেকোনো format × যেকোনো source
$pdfFromApi = new PdfReport(new ApiDataSource());
echo $pdfFromApi->generate('SELECT * FROM sales');

$excelFromDb = new ExcelReport(new MySQLDataSource());
echo $excelFromDb->generate('SELECT * FROM employees');
```

### 3. Laravel-এ Bridge প্যাটার্নের ব্যবহার

Laravel-এর Database layer একটি চমৎকার Bridge প্যাটার্নের বাস্তব উদাহরণ:

```
Connection (Abstraction)              Grammar (Implementation)
├── MySqlConnection                   ├── MySqlGrammar
├── PostgresConnection                ├── PostgresGrammar
├── SQLiteConnection                  ├── SQLiteGrammar
└── SqlServerConnection               └── SqlServerGrammar
```

```php
// Laravel internally যেভাবে কাজ করে (সরলীকৃত রূপ):
abstract class Connection  // Abstraction
{
    protected Grammar $queryGrammar;  // ← Bridge!

    public function table(string $table): QueryBuilder
    {
        return new QueryBuilder($this, $this->queryGrammar);
    }

    public function select(string $query, array $bindings = []): array
    {
        // Grammar ব্যবহার করে SQL compile করে
        $compiled = $this->queryGrammar->compileSelect($query);
        return $this->run($compiled, $bindings);
    }
}

// Connection এবং Grammar স্বাধীনভাবে vary করে
// নতুন Database support যোগ করতে দুটো ক্লাস যোগ করলেই হয়
```

---

## 🔥 Advanced Deep Dive

### Bridge vs Adapter — গুরুত্বপূর্ণ পার্থক্য

এটি সবচেয়ে বেশি confused হওয়া প্রশ্ন। দুটো দেখতে একই রকম কিন্তু **উদ্দেশ্য সম্পূর্ণ ভিন্ন।**

| বিষয় | Bridge 🌉 | Adapter 🔌 |
|---|---|---|
| **উদ্দেশ্য** | Abstraction ও Implementation আলাদা করা | বিদ্যমান incompatible interface-কে compatible করা |
| **কখন ডিজাইন** | **আগে থেকে** (upfront design) | **পরে** (legacy code-এ) |
| **সম্পর্ক** | দুই হায়ারার্কি আগে থেকে আলাদা | একটি বিদ্যমান ক্লাসকে wrap করা |
| **Analogy** | সেতু — দুই পাড় সংযোগ করে | পাওয়ার অ্যাডাপ্টার — ভিন্ন plug মানানসই করে |

```php
<?php

// Adapter — বিদ্যমান legacy code-কে নতুন interface-এ মানানসই করা
class LegacySmsService  // পুরানো third-party library
{
    public function sendTextMessage(string $number, string $text): int
    {
        return 1; // success code
    }
}

class SmsAdapter implements NotificationChannel  // আমাদের interface
{
    public function __construct(private LegacySmsService $legacy) {}

    public function send(string $recipient, string $title, string $body): bool
    {
        // পুরানো API-কে নতুন interface-এ adapt করা
        $result = $this->legacy->sendTextMessage($recipient, "[{$title}] {$body}");
        return $result === 1;
    }

    public function getChannelName(): string { return 'SMS (Legacy Adapter)'; }
}

// Bridge — দুইটা dimension-ই নতুনভাবে ডিজাইন করা
// NotificationType × NotificationChannel (উপরের উদাহরণ দেখুন)
```

### Bridge vs Strategy

| বিষয় | Bridge | Strategy |
|---|---|---|
| **Focus** | Structure — দুই হায়ারার্কি আলাদা করা | Behavior — algorithm পরিবর্তন করা |
| **Complexity** | দুই পাশেই হায়ারার্কি থাকে | এক পাশে context, অন্য পাশে strategies |
| **Intent** | বিভিন্ন dimension-কে decouple করা | runtime-এ algorithm swap করা |

### Bridge + Abstract Factory — শক্তিশালী সমন্বয়

```php
<?php

// Abstract Factory যে সঠিক Implementation তৈরি করে দেয়
interface PaymentFactory
{
    public function createGateway(): PaymentGateway;
    public function createPaymentMethod(PaymentGateway $gateway): PaymentMethod;
}

class BangladeshPaymentFactory implements PaymentFactory
{
    public function createGateway(): PaymentGateway
    {
        return new BkashGateway('app_key', 'app_secret');
    }

    public function createPaymentMethod(PaymentGateway $gateway): PaymentMethod
    {
        return new MobileBankingPayment($gateway);
    }
}

class InternationalPaymentFactory implements PaymentFactory
{
    public function createGateway(): PaymentGateway
    {
        return new StripeGateway('sk_live_xxx');
    }

    public function createPaymentMethod(PaymentGateway $gateway): PaymentMethod
    {
        return new CreditCardPayment($gateway);
    }
}

// ব্যবহার — Factory + Bridge
function processCheckout(PaymentFactory $factory, float $amount): void
{
    $gateway = $factory->createGateway();
    $method = $factory->createPaymentMethod($gateway);

    $result = $method->pay($amount, 'BDT', [
        'phone' => '+8801712345678',
        'card_number' => '4242424242424242',
    ]);

    echo $result->success ? "✅ সফল" : "❌ ব্যর্থ";
}

// বাংলাদেশি customer
processCheckout(new BangladeshPaymentFactory(), 1500);

// আন্তর্জাতিক customer
processCheckout(new InternationalPaymentFactory(), 5000);
```

### Runtime Bridge Switching

```javascript
class DynamicPaymentProcessor {
    #method;

    constructor(method) { this.#method = method; }

    async processWithFallback(amount, currency, metadata, gateways) {
        for (const gateway of gateways) {
            this.#method.switchGateway(gateway);
            console.log(`🔄 চেষ্টা করা হচ্ছে: ${gateway.name}`);

            const result = this.#method.pay(amount, currency, metadata);

            if (result.success) {
                console.log(`✅ সফল: ${gateway.name}`);
                return result;
            }

            console.log(`❌ ব্যর্থ: ${gateway.name}, পরবর্তী gateway-এ যাচ্ছি...`);
        }

        throw new Error('সকল gateway ব্যর্থ হয়েছে!');
    }
}

// Fallback chain: bKash → Nagad
const processor = new DynamicPaymentProcessor(
    new MobileBankingPayment(new BkashGateway('key'))
);

await processor.processWithFallback(
    1500, 'BDT',
    { phone: '+8801712345678' },
    [new BkashGateway('key'), new NagadGateway()]
);
```

### Multi-dimensional Bridge

কখনো কখনো দুইয়ের বেশি dimension থাকে। তখন Bridge chain করা যায়:

```php
<?php

// তিনটি dimension: MessageType × Channel × Formatter
interface MessageFormatter
{
    public function format(string $title, string $body): string;
}

class BanglaFormatter implements MessageFormatter
{
    public function format(string $title, string $body): string
    {
        return "📜 শিরোনাম: {$title}\n📝 বিবরণ: {$body}";
    }
}

class EnglishFormatter implements MessageFormatter
{
    public function format(string $title, string $body): string
    {
        return "📜 Title: {$title}\n📝 Body: {$body}";
    }
}

// Channel এখন Formatter-ও ব্যবহার করে — দ্বিতীয় Bridge
class FormattedEmailChannel implements NotificationChannel
{
    public function __construct(
        private readonly MessageFormatter $formatter,
    ) {}

    public function send(string $recipient, string $title, string $body): bool
    {
        $formatted = $this->formatter->format($title, $body);
        echo "📧 → {$recipient}:\n{$formatted}\n";
        return true;
    }

    public function getChannelName(): string { return 'Formatted Email'; }
}

// NotificationType × FormattedChannel × Formatter — তিনটি dimension!
$banglaEmail = new FormattedEmailChannel(new BanglaFormatter());
$urgent = new UrgentNotification();
$urgent->addChannel($banglaEmail);
$urgent->notify('user@example.com', 'সার্ভার ডাউন হয়েছে!');
```

---

## ✅ Pros (সুবিধা)

| # | সুবিধা | ব্যাখ্যা |
|---|---|---|
| 1 | **Open/Closed Principle** | নতুন Abstraction বা Implementation যোগ করা যায় বিদ্যমান কোড পরিবর্তন ছাড়া |
| 2 | **Single Responsibility** | Abstraction উচ্চ-স্তরের logic, Implementation নিম্ন-স্তরের details — আলাদা |
| 3 | **Class Explosion রোধ** | n × m ক্লাসের বদলে n + m ক্লাস |
| 4 | **Runtime Switching** | চলমান অবস্থায় implementation পরিবর্তন করা যায় |
| 5 | **Platform Independence** | Abstraction platform-neutral থাকে |
| 6 | **Testability** | Mock implementation দিয়ে সহজে test করা যায় |

## ❌ Cons (অসুবিধা)

| # | অসুবিধা | ব্যাখ্যা |
|---|---|---|
| 1 | **Complexity বৃদ্ধি** | সাধারণ সমস্যায় অতিরিক্ত indirection তৈরি হয় |
| 2 | **Over-engineering** | যদি শুধু একটি implementation থাকে, Bridge অপ্রয়োজনীয় |
| 3 | **Learning Curve** | Junior developer-দের জন্য বোঝা কঠিন হতে পারে |
| 4 | **Initial Setup** | শুরুতে বেশি কোড লিখতে হয় |

---

## ⚠️ Common Mistakes (সাধারণ ভুল)

### ❌ ভুল ১: একটি মাত্র Implementation-এ Bridge ব্যবহার

```php
// ❌ ভুল — শুধু একটি implementation থাকলে Bridge লাগে না
interface Logger { public function log(string $msg): void; }
class FileLogger implements Logger { /* ... */ }

// এখানে শুধু FileLogger-ই আছে, অন্য কোনো Logger নেই
// সরাসরি FileLogger ব্যবহার করুন, Bridge অপ্রয়োজনীয়
```

### ❌ ভুল ২: Abstraction-এ Implementation-এর concrete type জানা

```php
// ❌ ভুল — Abstraction নির্দিষ্ট Implementation-এর উপর নির্ভর
class RemoteControl {
    public function __construct(private SamsungTV $tv) {} // ❌ Concrete!
}

// ✅ সঠিক — Interface-এর উপর নির্ভর
class RemoteControl {
    public function __construct(private Device $device) {} // ✅ Interface!
}
```

### ❌ ভুল ৩: Bridge এবং Inheritance মিশিয়ে ফেলা

```php
// ❌ ভুল — Implementation-কে Abstraction-এর ভিতর inherit করা
class EmailUrgentNotification extends EmailChannel { /* ... */ }

// ✅ সঠিক — Composition ব্যবহার করুন
class UrgentNotification {
    public function __construct(private NotificationChannel $channel) {}
}
```

### ❌ ভুল ৪: অত্যধিক Abstraction Layer

```php
// ❌ ভুল — যেখানে simple strategy কাজ করবে সেখানে Bridge জোর করা
// যদি শুধু algorithm swap করতে চান, Strategy ব্যবহার করুন
// Bridge ব্যবহার করুন যখন দুই পাশেই স্বতন্ত্র হায়ারার্কি আছে
```

---

## 🧪 টেস্টিং

### PHPUnit

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;

class BridgePatternTest extends TestCase
{
    // Mock Implementation দিয়ে Abstraction test করা
    public function test_urgent_notification_sends_to_all_channels(): void
    {
        $channel1 = $this->createMock(NotificationChannel::class);
        $channel2 = $this->createMock(NotificationChannel::class);

        $channel1->expects($this->once())
            ->method('send')
            ->with('+8801712345678', '🚨 জরুরি', 'টেস্ট মেসেজ')
            ->willReturn(true);

        $channel2->expects($this->once())
            ->method('send')
            ->with('+8801712345678', '🚨 জরুরি', 'টেস্ট মেসেজ')
            ->willReturn(true);

        $notification = new UrgentNotification();
        $notification->addChannel($channel1)->addChannel($channel2);
        $notification->notify('+8801712345678', 'টেস্ট মেসেজ');
    }

    public function test_normal_notification_sends_to_first_channel_only(): void
    {
        $channel1 = $this->createMock(NotificationChannel::class);
        $channel2 = $this->createMock(NotificationChannel::class);

        $channel1->expects($this->once())->method('send')->willReturn(true);
        $channel2->expects($this->never())->method('send');

        $notification = new NormalNotification();
        $notification->addChannel($channel1)->addChannel($channel2);
        $notification->notify('+8801712345678', 'টেস্ট');
    }

    public function test_bkash_rejects_non_bdt_currency(): void
    {
        $bkash = new BkashGateway('key', 'secret');
        $result = $bkash->processPayment(100, 'USD', []);

        $this->assertFalse($result->success);
        $this->assertStringContainsString('BDT', $result->message);
    }

    public function test_bkash_rejects_amount_over_limit(): void
    {
        $bkash = new BkashGateway('key', 'secret');
        $result = $bkash->processPayment(30000, 'BDT', ['phone' => '+8801712345678']);

        $this->assertFalse($result->success);
        $this->assertStringContainsString('25,000', $result->message);
    }

    public function test_credit_card_adds_surcharge(): void
    {
        $mockGateway = $this->createMock(PaymentGateway::class);
        $mockGateway->expects($this->once())
            ->method('processPayment')
            ->with(
                $this->equalTo(1025.0), // 1000 × 1.025
                $this->equalTo('BDT'),
                $this->anything()
            )
            ->willReturn(new PaymentResult(true, 'txn_1', 'Mock', 1025.0, 'OK'));

        $payment = new CreditCardPayment($mockGateway);
        $result = $payment->pay(1000, 'BDT', ['card_number' => '4242424242424242']);

        $this->assertTrue($result->success);
    }

    public function test_runtime_gateway_switch(): void
    {
        $bkash = $this->createMock(PaymentGateway::class);
        $nagad = $this->createMock(PaymentGateway::class);

        $bkash->method('getGatewayName')->willReturn('bKash');
        $nagad->method('getGatewayName')->willReturn('Nagad');

        $payment = new MobileBankingPayment($bkash);
        $this->assertEquals('bKash', $payment->getGateway());

        $payment->switchGateway($nagad);
        $this->assertEquals('Nagad', $payment->getGateway());
    }

    public function test_remote_control_with_different_devices(): void
    {
        $samsung = new SamsungTV();
        $walton = new WaltonTV();

        $remote1 = new AdvancedRemote($samsung);
        $remote1->togglePower();
        $remote1->volumeUp();

        $remote2 = new AdvancedRemote($walton);
        $remote2->togglePower();
        $remote2->volumeUp();

        // Samsung max vol 100, Walton max vol 80
        $this->assertEquals(40, $samsung->getVolume());
        $this->assertEquals(30, $walton->getVolume());
    }
}
```

### Jest (JavaScript)

```javascript
describe('Bridge Pattern Tests', () => {
    describe('Device + Remote', () => {
        test('AdvancedRemote should work with any Device', () => {
            const samsung = new SamsungTV();
            const remote = new AdvancedRemote(samsung);

            remote.togglePower();
            expect(samsung.isEnabled()).toBe(true);

            remote.volumeUp();
            expect(samsung.getVolume()).toBe(40);

            remote.mute();
            expect(samsung.getVolume()).toBe(0);
        });

        test('WaltonTV should cap volume at 80', () => {
            const walton = new WaltonTV();
            const remote = new RemoteControl(walton);

            remote.togglePower();
            for (let i = 0; i < 15; i++) remote.volumeUp(); // 20 + 150 = 170

            expect(walton.getVolume()).toBe(80);
        });
    });

    describe('Notification Bridge', () => {
        test('UrgentNotification sends to all channels', () => {
            const sends = [];
            const mockChannel = {
                send: (r, t, b) => { sends.push({ r, t, b }); return true; },
                channelName: 'Mock',
            };

            const urgent = new UrgentNotification();
            urgent.addChannel(mockChannel).addChannel({ ...mockChannel });
            urgent.notify('+8801712345678', 'টেস্ট');

            expect(sends).toHaveLength(2);
            expect(sends[0].t).toBe('🚨 জরুরি');
        });

        test('NormalNotification sends to first channel only', () => {
            let sendCount = 0;
            const mockChannel = {
                send: () => { sendCount++; return true; },
                channelName: 'Mock',
            };

            const normal = new NormalNotification();
            normal.addChannel(mockChannel).addChannel({ ...mockChannel });
            normal.notify('+8801712345678', 'টেস্ট');

            expect(sendCount).toBe(1);
        });
    });

    describe('Payment Bridge', () => {
        test('bKash rejects non-BDT currency', () => {
            const bkash = new BkashGateway('key');
            const result = bkash.processPayment(100, 'USD', {});

            expect(result.success).toBe(false);
        });

        test('runtime gateway switching works', () => {
            const bkash = new BkashGateway('key');
            const nagad = new NagadGateway();

            const payment = new MobileBankingPayment(bkash);
            expect(payment.gateway.name).toBe('bKash');

            payment.switchGateway(nagad);
            expect(payment.gateway.name).toBe('Nagad');

            const result = payment.pay(1000, 'BDT', { phone: '+8801712345678' });
            expect(result.success).toBe(true);
            expect(result.gateway).toBe('Nagad');
        });
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Adapter 🔌
- **সম্পর্ক:** Adapter বিদ্যমান incompatible interface-কে কাজের উপযোগী করে, Bridge শুরু থেকেই দুটো dimension আলাদা রাখে।
- **কখন Adapter:** Legacy system integrate করতে হলে। যেমন: পুরানো SMS gateway-র API-কে আপনার নতুন `NotificationChannel` interface-এ মানানসই করা।
- **কখন Bridge:** নতুন system design করার সময় যখন দুটো স্বতন্ত্র হায়ারার্কি দেখা যায়।

### Abstract Factory 🏭
- **সম্পর্ক:** Abstract Factory দিয়ে Bridge-এর সঠিক Abstraction + Implementation জোড়া তৈরি করা যায়।
- **ব্যবহার:** উপরের `BangladeshPaymentFactory` উদাহরণ দেখুন — Factory সিদ্ধান্ত নেয় কোন Gateway ও কোন PaymentMethod ব্যবহার হবে।

### Strategy 🎯
- **সম্পর্ক:** দুটোই composition ব্যবহার করে, কিন্তু Strategy শুধু behavior swap করে, Bridge দুটো সম্পূর্ণ হায়ারার্কি আলাদা রাখে।
- **চিন্তা করুন:** "আমার কি শুধু algorithm বদলাতে হবে (Strategy), নাকি দুটো আলাদা dimension manage করতে হবে (Bridge)?"

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন

1. **Class explosion দেখা দিচ্ছে** — যখন দুটো স্বাধীন dimension-এর জন্য n × m ক্লাস বানাতে হচ্ছে
2. **Runtime-এ implementation switch করতে হবে** — যেমন: bKash fail হলে Nagad-এ fallback
3. **Platform-independent abstraction দরকার** — যেমন: একই business logic Web, Mobile, Desktop-এ
4. **Implementation details লুকাতে হবে** — Client শুধু Abstraction জানবে
5. **দুটো হায়ারার্কিই ভবিষ্যতে বাড়বে** — নতুন payment method ও নতুন gateway দুটোই যোগ হবে

### ❌ ব্যবহার করবেন না

1. **শুধু একটি implementation আছে** — YAGNI (You Ain't Gonna Need It) নীতি মানুন
2. **সমস্যা শুধু behavior switching** — Strategy ব্যবহার করুন
3. **Legacy code-এ compatibility দরকার** — Adapter ব্যবহার করুন
4. **ছোট প্রজেক্ট** — অতিরিক্ত complexity আনবেন না
5. **দুটো dimension সত্যিই independent না** — Bridge তখন কৃত্রিম বিভাজন তৈরি করে

### সিদ্ধান্ত নেওয়ার ফ্লোচার্ট

```
আপনার কি দুটো স্বতন্ত্র dimension আছে?
├── হ্যাঁ → দুটোই কি ভবিষ্যতে বাড়বে?
│         ├── হ্যাঁ → ✅ Bridge ব্যবহার করুন
│         └── না → শুধু behavior বদলায়?
│                   ├── হ্যাঁ → Strategy ব্যবহার করুন
│                   └── না → Simple inheritance যথেষ্ট
└── না → Bridge লাগবে না
```

---

## 📋 সারসংক্ষেপ

| বিষয় | বিবরণ |
|---|---|
| **প্যাটার্ন ধরন** | Structural |
| **মূল সমস্যা** | Class explosion যখন দুটো স্বাধীন dimension থাকে |
| **সমাধান** | Composition দিয়ে দুটো হায়ারার্কি আলাদা করা |
| **GoF সূত্র** | Abstraction ও Implementation-কে decouple করা যেন দুটোই স্বাধীনভাবে vary করে |
| **মূল নীতি** | Composition over Inheritance, Open/Closed Principle |
| **বাংলাদেশি উদাহরণ** | bKash/Nagad × Credit/Mobile Payment, SMS/Email/Push × Urgent/Normal |
| **Laravel-এ** | Database Connection × Grammar |
| **সতর্কতা** | অতিরিক্ত abstraction করবেন না, YAGNI মানুন |
| **সেরা বন্ধু** | Abstract Factory (সঠিক জোড়া তৈরি), Strategy (behavior swap) |

### মনে রাখার সূত্র 🧠

> **"Bridge = দুটো আলাদা জিনিস, একটা সেতু দিয়ে সংযুক্ত।"**
>
> রিমোট আলাদা, টিভি আলাদা — কিন্তু যেকোনো রিমোট যেকোনো টিভিতে কাজ করে।
> Payment Method আলাদা, Gateway আলাদা — কিন্তু যেকোনো method যেকোনো gateway-এ কাজ করে।
>
> **যখন দেখবেন "X এর ধরন" × "Y এর ধরন" = অনেক ক্লাস — তখনই Bridge!**

---

*এই ডকুমেন্টটি বাংলাদেশি সফটওয়্যার ডেভেলপারদের জন্য তৈরি। উদাহরণগুলোতে বাংলাদেশের পেমেন্ট ইকোসিস্টেম (bKash, Nagad), টেলিকম, এবং সাধারণ প্রযুক্তি ব্যবহার করা হয়েছে।*
