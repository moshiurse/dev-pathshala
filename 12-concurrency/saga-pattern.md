# 🔄 সাগা প্যাটার্ন (Saga Pattern)

## ধারণা

সাগা প্যাটার্ন হলো ডিস্ট্রিবিউটেড সিস্টেমে দীর্ঘমেয়াদী ট্রানজ্যাকশন পরিচালনার একটি কৌশল। যেখানে একটি বড় ট্রানজ্যাকশনকে একাধিক ছোট ছোট লোকাল ট্রানজ্যাকশনে ভাগ করা হয়। প্রতিটি ধাপ সফল হলে পরবর্তী ধাপ চলে, ব্যর্থ হলে পূর্ববর্তী ধাপগুলোর **কম্পেনসেটিং ট্রানজ্যাকশন** চালিয়ে রোলব্যাক করা হয়।

### দুই ধরনের সাগা

1. **Choreography (কোরিওগ্রাফি)**: প্রতিটি সার্ভিস নিজেই ইভেন্ট পাবলিশ করে এবং অন্য সার্ভিস সেটি শোনে। কোনো কেন্দ্রীয় নিয়ন্ত্রক নেই।
2. **Orchestration (অর্কেস্ট্রেশন)**: একটি কেন্দ্রীয় অর্কেস্ট্রেটর সব ধাপ নিয়ন্ত্রণ করে, প্রতিটি সার্ভিসকে কখন কী করতে হবে বলে দেয়।

---

## 🏠 বাস্তব জীবনের উদাহরণ

### ই-কমার্স অর্ডার প্রসেসিং

অনলাইনে কিছু অর্ডার করলে:
1. **অর্ডার তৈরি** → ২. **পেমেন্ট চার্জ** → ৩. **ইনভেন্টরি কমানো** → ৪. **শিপিং শিডিউল**

যদি ৩ নম্বর ধাপে পণ্য স্টকে না থাকে:
- ৩. ইনভেন্টরি → ব্যর্থ
- ২. পেমেন্ট → রিফান্ড (কম্পেনসেশন)
- ১. অর্ডার → ক্যান্সেল (কম্পেনসেশন)

**কোরিওগ্রাফি** = বিয়ের অনুষ্ঠানে প্রতিটি দল নিজেরাই জানে কখন কী করতে হবে (গায়ে হলুদ শেষ → মেহেদী শুরু)।
**অর্কেস্ট্রেশন** = বিয়ের প্ল্যানার সবকিছু নিয়ন্ত্রণ করে, প্রতিটি দলকে নির্দেশ দেয়।

---

## 💻 PHP কোড উদাহরণ

### Orchestration সাগা

```php
<?php

class OrderSagaOrchestrator
{
    private array $completedSteps = [];

    public function execute(array $orderData): bool
    {
        try {
            // ধাপ ১: অর্ডার তৈরি
            $orderId = $this->createOrder($orderData);
            $this->completedSteps[] = ['action' => 'order', 'id' => $orderId];

            // ধাপ ২: পেমেন্ট প্রসেস
            $paymentId = $this->processPayment($orderId, $orderData['amount']);
            $this->completedSteps[] = ['action' => 'payment', 'id' => $paymentId];

            // ধাপ ৩: ইনভেন্টরি রিজার্ভ
            $reservationId = $this->reserveInventory($orderId, $orderData['items']);
            $this->completedSteps[] = ['action' => 'inventory', 'id' => $reservationId];

            // ধাপ ৪: শিপিং শিডিউল
            $shippingId = $this->scheduleShipping($orderId, $orderData['address']);
            $this->completedSteps[] = ['action' => 'shipping', 'id' => $shippingId];

            echo "✅ অর্ডার সাগা সম্পূর্ণ সফল!\n";
            return true;

        } catch (\Exception $e) {
            echo "❌ ব্যর্থ: {$e->getMessage()}\n";
            $this->compensate();
            return false;
        }
    }

    private function compensate(): void
    {
        echo "🔄 কম্পেনসেশন শুরু...\n";
        foreach (array_reverse($this->completedSteps) as $step) {
            match ($step['action']) {
                'shipping' => $this->cancelShipping($step['id']),
                'inventory' => $this->releaseInventory($step['id']),
                'payment'   => $this->refundPayment($step['id']),
                'order'     => $this->cancelOrder($step['id']),
            };
            echo "↩️ {$step['action']} কম্পেনসেট করা হয়েছে\n";
        }
    }

    private function createOrder(array $data): string
    {
        echo "📦 অর্ডার তৈরি হচ্ছে...\n";
        return 'ORD-' . uniqid();
    }

    private function processPayment(string $orderId, float $amount): string
    {
        echo "💳 পেমেন্ট প্রসেস হচ্ছে: {$amount} টাকা\n";
        return 'PAY-' . uniqid();
    }

    private function reserveInventory(string $orderId, array $items): string
    {
        echo "📋 ইনভেন্টরি রিজার্ভ হচ্ছে...\n";
        if (rand(0, 1) === 0) {
            throw new \RuntimeException('স্টকে পণ্য নেই!');
        }
        return 'RES-' . uniqid();
    }

    private function scheduleShipping(string $orderId, string $address): string
    {
        echo "🚚 শিপিং শিডিউল হচ্ছে: {$address}\n";
        return 'SHIP-' . uniqid();
    }

    private function cancelOrder(string $id): void { /* অর্ডার ক্যান্সেল */ }
    private function refundPayment(string $id): void { /* রিফান্ড */ }
    private function releaseInventory(string $id): void { /* ইনভেন্টরি ফেরত */ }
    private function cancelShipping(string $id): void { /* শিপিং ক্যান্সেল */ }
}

// ব্যবহার
$saga = new OrderSagaOrchestrator();
$saga->execute([
    'items'   => ['laptop', 'mouse'],
    'amount'  => 85000,
    'address' => 'ঢাকা, বাংলাদেশ',
]);
```

---

## 💻 JavaScript কোড উদাহরণ

### Choreography সাগা (Event-Driven)

```javascript
const EventEmitter = require('events');

class SagaEventBus extends EventEmitter {}
const bus = new SagaEventBus();

// অর্ডার সার্ভিস
bus.on('order.create', async (data) => {
    console.log('📦 অর্ডার তৈরি হচ্ছে...');
    const orderId = `ORD-${Date.now()}`;
    bus.emit('order.created', { ...data, orderId });
});

bus.on('payment.failed', async ({ orderId }) => {
    console.log(`↩️ অর্ডার ${orderId} ক্যান্সেল হচ্ছে`);
});

// পেমেন্ট সার্ভিস
bus.on('order.created', async ({ orderId, amount }) => {
    console.log(`💳 পেমেন্ট প্রসেস হচ্ছে: ${amount} টাকা`);
    try {
        if (amount > 100000) throw new Error('লিমিট অতিক্রম!');
        const paymentId = `PAY-${Date.now()}`;
        bus.emit('payment.completed', { orderId, paymentId });
    } catch (err) {
        console.log(`❌ পেমেন্ট ব্যর্থ: ${err.message}`);
        bus.emit('payment.failed', { orderId });
    }
});

// ইনভেন্টরি সার্ভিস
bus.on('payment.completed', async ({ orderId, paymentId }) => {
    console.log('📋 ইনভেন্টরি রিজার্ভ হচ্ছে...');
    const available = Math.random() > 0.3;
    if (available) {
        bus.emit('inventory.reserved', { orderId, paymentId });
    } else {
        console.log('❌ স্টক নেই — কম্পেনসেশন শুরু');
        bus.emit('inventory.failed', { orderId, paymentId });
    }
});

bus.on('inventory.failed', async ({ orderId, paymentId }) => {
    console.log(`↩️ পেমেন্ট ${paymentId} রিফান্ড হচ্ছে`);
    bus.emit('payment.failed', { orderId });
});

// শিপিং সার্ভিস
bus.on('inventory.reserved', async ({ orderId }) => {
    console.log(`🚚 শিপিং শিডিউল হচ্ছে — অর্ডার ${orderId}`);
    console.log('✅ সাগা সম্পূর্ণ!');
});

// সাগা শুরু
bus.emit('order.create', { items: ['phone'], amount: 45000 });
```

### Orchestration সাগা (JavaScript)

```javascript
class SagaOrchestrator {
    constructor() {
        this.steps = [];
    }

    addStep(name, execute, compensate) {
        this.steps.push({ name, execute, compensate });
        return this;
    }

    async run(context) {
        const completed = [];
        for (const step of this.steps) {
            try {
                console.log(`▶️ ${step.name} শুরু...`);
                const result = await step.execute(context);
                context = { ...context, ...result };
                completed.push(step);
            } catch (err) {
                console.log(`❌ ${step.name} ব্যর্থ: ${err.message}`);
                for (const done of completed.reverse()) {
                    console.log(`↩️ ${done.name} কম্পেনসেট হচ্ছে`);
                    await done.compensate(context);
                }
                throw err;
            }
        }
        return context;
    }
}

// ব্যবহার
const saga = new SagaOrchestrator()
    .addStep('অর্ডার তৈরি',
        async (ctx) => ({ orderId: `ORD-${Date.now()}` }),
        async (ctx) => console.log(`অর্ডার ${ctx.orderId} মুছে ফেলা হয়েছে`)
    )
    .addStep('পেমেন্ট',
        async (ctx) => ({ paymentId: `PAY-${Date.now()}` }),
        async (ctx) => console.log(`পেমেন্ট ${ctx.paymentId} রিফান্ড`)
    )
    .addStep('ইনভেন্টরি',
        async (ctx) => {
            if (Math.random() < 0.5) throw new Error('স্টক শেষ');
            return { reservationId: `RES-${Date.now()}` };
        },
        async (ctx) => console.log(`ইনভেন্টরি ${ctx.reservationId} মুক্ত`)
    );

saga.run({ amount: 5000 })
    .then(() => console.log('✅ সম্পূর্ণ!'))
    .catch(() => console.log('🔄 রোলব্যাক সম্পন্ন'));
```

---

## ⚖️ Choreography vs Orchestration

| বৈশিষ্ট্য | Choreography | Orchestration |
|-----------|-------------|---------------|
| নিয়ন্ত্রণ | বিকেন্দ্রীভূত | কেন্দ্রীভূত |
| কাপলিং | লুজ কাপলিং | অর্কেস্ট্রেটরের উপর নির্ভরশীল |
| জটিলতা | সহজ ফ্লোতে ভালো | জটিল ফ্লোতে ভালো |
| ডিবাগিং | কঠিন (ইভেন্ট ট্রেসিং দরকার) | সহজ (কেন্দ্রীয় লজিক) |
| স্কেলেবিলিটি | বেশি | অর্কেস্ট্রেটর বটলনেক হতে পারে |
| উদাহরণ | Kafka ইভেন্ট | Temporal, Camunda |

---

## ✅ কখন ব্যবহার করবেন

- মাইক্রোসার্ভিস আর্কিটেকচারে ক্রস-সার্ভিস ট্রানজ্যাকশনে
- দীর্ঘমেয়াদী বিজনেস প্রসেস যেখানে অনেক ধাপ আছে
- যখন ACID ট্রানজ্যাকশন সম্ভব নয় (ভিন্ন ডাটাবেজ, ভিন্ন সার্ভিস)
- ই-কমার্স অর্ডার, ট্রাভেল বুকিং, ব্যাংকিং ট্রান্সফার

## ❌ কখন ব্যবহার করবেন না

- সিঙ্গেল ডাটাবেজ ট্রানজ্যাকশনে (সাধারণ ACID যথেষ্ট)
- সিঙ্ক্রোনাস রেসপন্স প্রয়োজন হলে (সাগা ইভেঞ্চুয়ালি কনসিস্টেন্ট)
- খুব সাধারণ CRUD অপারেশনে
- যখন সিস্টেমে ২-৩টি সার্ভিসের বেশি নেই এবং একটি শেয়ারড ডাটাবেজ ব্যবহার করা যায়
