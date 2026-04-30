# 🎭 অ্যাক্টর মডেল (Actor Model)

## 📌 সূচিপত্র
- [অ্যাক্টর মডেল কী](#-অ্যাক্টর-মডেল-কী)
- [অ্যাক্টর = কম্পিউটেশনের একক](#-অ্যাক্টর--কম্পিউটেশনের-একক)
- [মেসেজ পাসিং](#️-মেসেজ-পাসিং)
- [Akka ও proto.actor কনসেপ্ট](#-akka-ও-protoactor-কনসেপ্ট)
- [PHP তে অ্যাক্টর মডেল](#-php-তে-অ্যাক্টর-মডেল)
- [JavaScript এ অ্যাক্টর মডেল](#-javascript-এ-অ্যাক্টর-মডেল)
- [থ্রেড-ভিত্তিক vs অ্যাক্টর মডেল](#️-থ্রেড-ভিত্তিক-vs-অ্যাক্টর-মডেল)
- [ব্যবহারের ক্ষেত্র](#-ব্যবহারের-ক্ষেত্র)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)
- [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 অ্যাক্টর মডেল কী?

### 📖 ইতিহাস

অ্যাক্টর মডেল **১৯৭৩ সালে Carl Hewitt** প্রস্তাব করেন। এটি concurrent computation-এর
একটি mathematical model। পরবর্তীতে **Gul Agha** এটিকে আরও formalize করেন। Erlang
ভাষা (১৯৮৬) এই মডেলের উপর ভিত্তি করে তৈরি — যা দিয়ে Ericsson তাদের telecom switch
বানিয়েছিল। আজকের দিনে **Akka (JVM)**, **Orleans (.NET)**, **proto.actor** সবই এই
মডেল অনুসরণ করে।

### 🏠 বাস্তব উদাহরণ — Grameenphone কল সেন্টার

কল্পনা করুন **Grameenphone-এর কল সেন্টার**:

```
📞 Customer Calls (Messages)
    |         |         |
    v         v         v
+--------+ +--------+ +--------+
| Agent  | | Agent  | | Agent  |
| রহিম   | | করিম   | | সালমা  |
| (Actor)| | (Actor)| | (Actor)|
+--------+ +--------+ +--------+
    |                      |
    | (জটিল সমস্যা)         | (নতুন agent দরকার)
    v                      v
+--------+           +--------+
| Senior |           | Tech   |
| Agent  |           | Agent  |
| (Actor)|           | (Actor)|
+--------+           +--------+
```

**কল সেন্টারের সাথে তুলনা:**

| কল সেন্টার             | অ্যাক্টর মডেল                    |
|------------------------|----------------------------------|
| প্রতিটি Agent           | একটি Actor                        |
| ফোন কল আসা              | Message receive করা               |
| Agent নিজের ডেস্কে কাজ   | Actor-এর private state           |
| কলার queue-তে থাকে      | Mailbox (message queue)          |
| Senior-কে ফরওয়ার্ড করা  | অন্য Actor-কে message পাঠানো     |
| নতুন Agent hire করা     | নতুন Actor create করা             |

> **মূল ধারণা:** প্রতিটি Actor স্বাধীনভাবে কাজ করে — কোনো shared state নেই, শুধু
> message-এর মাধ্যমে যোগাযোগ।

---

## 📊 অ্যাক্টর = কম্পিউটেশনের একক

একটি Actor হলো concurrency-এর সবচেয়ে ছোট একক। প্রতিটি Actor-এর তিনটি মূল
উপাদান আছে:

### Actor-এর গঠন

```
+----------------------------------------------------------+
|                        Actor                              |
|                                                          |
|  +------------+   +-------------+   +-----------------+  |
|  |   State    |   |  Behavior   |   |    Mailbox      |  |
|  |  (private) |   |  (process)  |   |  (msg queue)    |  |
|  |            |   |             |   |                 |  |
|  | - balance  |   | - onReceive |   | -> [msg1]       |  |
|  | - name     |   | - validate  |   | -> [msg2]       |  |
|  | - status   |   | - transform |   | -> [msg3]       |  |
|  |            |   |             |   | -> [msg4]       |  |
|  +------------+   +-------------+   +-----------------+  |
|                                            |              |
|                                     একটি একটি করে        |
|                                     process হয়           |
+----------------------------------------------------------+
```

### ১. State (ব্যক্তিগত অবস্থা)

- Actor-এর নিজস্ব ডেটা, বাইরে থেকে সরাসরি access করা যায় না
- শুধুমাত্র Actor নিজে তার state পরিবর্তন করতে পারে
- **উদাহরণ:** bKash account-এর balance — শুধু সেই account actor-ই balance পরিবর্তন করতে পারে

### ২. Behavior (আচরণ)

- Message পেলে Actor কী করবে তা define করে
- প্রতিটি message process করার পর Actor তার behavior পরিবর্তন করতে পারে
- **উদাহরণ:** Daraz-এ order actor — "pending" state-এ payment message পেলে "paid" হয়

### ৩. Mailbox (বার্তার সারি)

- Actor-এ আসা সকল message এখানে queue হয়
- **FIFO** (First In, First Out) অর্ডারে process হয়
- একবারে একটিই message process হয় — কোনো race condition নেই!

### Actor কী কী করতে পারে?

একটি message পাওয়ার পর Actor তিনটি কাজ করতে পারে:

```
Message পেলে Actor করতে পারে:
                |
    +-----------+-----------+
    |           |           |
    v           v           v
 নতুন Actor   অন্য Actor-  নিজের
 তৈরি করা    কে message   state/behavior
             পাঠানো      পরিবর্তন করা
```

**bKash উদাহরণ:**
- **নতুন Actor তৈরি:** একটি transaction request আসলে একটি নতুন TransactionActor তৈরি হয়
- **Message পাঠানো:** TransactionActor → NotificationActor-কে "SMS পাঠাও" message দেয়
- **State পরিবর্তন:** TransactionActor নিজের status "processing" → "completed" করে

---

## ✉️ মেসেজ পাসিং

### 📖 মূল নীতি: কোনো Shared State নেই

অ্যাক্টর মডেলে **কোনো shared memory নেই**। সবকিছু message-এর মাধ্যমে হয়:

```
  ❌ ভুল পদ্ধতি (Shared State)         ✅ সঠিক পদ্ধতি (Message Passing)

  +--------+    +--------+            +--------+         +--------+
  | Actor  |    | Actor  |            | Actor  | --msg--> | Actor  |
  |   A    |--->|   B    |            |   A    |          |   B    |
  +--------+  | +--------+            +--------+         +--------+
              |                           ^                   |
              v                           |                   |
         +----------+                    msg               msg
         | Shared   |                     |                   |
         | Memory   |   <-- Race         +--------+          |
         | (danger!)|       Condition!   | Actor  | <--------+
         +----------+                    |   C    |
                                         +--------+
```

### Fire-and-Forget Pattern (Tell)

Message পাঠিয়ে ভুলে যাও — response-এর অপেক্ষা নেই:

```
+----------+     "Process Order #456"      +----------+
|  Order   | ---------------------------> |  Payment |
|  Actor   |     (fire & forget)          |  Actor   |
+----------+                              +----------+
     |
     | পরের কাজে চলে যায়
     | response-এর জন্য
     | অপেক্ষা করে না
     v
 (অন্য message process)
```

**কখন ব্যবহার করবেন:**
- Notification পাঠানো (Grameenphone SMS)
- Log লেখা
- Analytics event পাঠানো

### Ask Pattern (Request-Reply)

Message পাঠিয়ে response-এর জন্য অপেক্ষা করা:

```
+----------+    "Check Balance"     +----------+
|  User    | ---------------------> | Account  |
|  Actor   |                        |  Actor   |
|          | <--------------------- |          |
+----------+    "Balance: ৳5000"    +----------+
       |
       | response পাওয়ার পর
       | পরের কাজ করে
       v
```

**কখন ব্যবহার করবেন:**
- bKash balance check
- Daraz-এ product price জানতে চাওয়া
- Pathao-তে nearest driver খোঁজা

### 🔄 মেসেজ ফ্লো — Nagad টাকা পাঠানো

```
+---------+   "Send ৳500    +------------+   "Deduct ৳500"  +----------+
|  User   |   to 01712..."  | Transaction|  ------------->  | Sender   |
|  API    | -------------->  |   Actor    |                  | Account  |
|  Actor  |                 |            |   "Add ৳500"     | Actor    |
+---------+                 |            |  ------------->  +----------+
     ^                      |            |                  +----------+
     |                      |            |                  | Receiver |
     |   "Success/Fail"    |            |   "Send SMS"     | Account  |
     | <------------------ |            |  ------------->  | Actor    |
     |                      +------------+                  +----------+
     |                                                     +----------+
     |                                                     |  SMS     |
     |                                                     |  Actor   |
     |                                                     +----------+
```

**প্রতিটি Actor স্বাধীন:**
- Transaction Actor শুধু orchestration করে
- Account Actor শুধু balance manage করে
- SMS Actor শুধু notification পাঠায়
- কেউ কারো internal state জানে না!

---

## 🔧 Akka ও proto.actor কনসেপ্ট

### Akka (JVM)

**Akka** হলো JVM-ভিত্তিক সবচেয়ে জনপ্রিয় Actor framework। Scala ও Java-তে ব্যবহার
করা যায়। Lightbend (আগের Typesafe) এটি maintain করে।

**বৈশিষ্ট্য:**
- Location transparency — actor local বা remote যেকোনো জায়গায় থাকতে পারে
- Supervision strategy — parent actor child-এর error handle করে
- Cluster support — distributed system সহজে বানানো যায়
- Persistence — actor state দীর্ঘস্থায়ীভাবে সংরক্ষণ করা যায়

### proto.actor

**proto.actor** হলো multi-language actor framework। Go, C#, Kotlin, Python-এ
ব্যবহার করা যায়। Akka-এর চেয়ে lightweight এবং cross-platform।

### Actor Hierarchy ও Supervision Tree

অ্যাক্টর মডেলে প্রতিটি actor একটি tree structure-তে থাকে। Parent actor তার
child-দের supervise করে:

```
                    +------------------+
                    |   /system        |
                    |  (Root Actor)    |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
    +---------+----------+      +-----------+---------+
    |  /order-service    |      |  /payment-service   |
    |  (Supervisor)      |      |  (Supervisor)       |
    +----+----------+----+      +-----+----------+----+
         |          |                 |          |
    +----+---+ +----+----+     +-----+---+ +----+----+
    | Order  | | Order   |     | Payment | | Payment |
    | Actor  | | Actor   |     | Actor   | | Actor   |
    | #1     | | #2      |     | #1      | | #2      |
    +--------+ +---------+     +---------+ +---------+
```

### Supervision Strategy (তত্ত্বাবধান কৌশল)

Parent actor child ব্যর্থ হলে কী করবে:

| Strategy        | কাজ                                      | উদাহরণ                          |
|----------------|-------------------------------------------|---------------------------------|
| Resume         | Error উপেক্ষা করে চালিয়ে যাও              | ছোট validation error           |
| Restart        | Actor বন্ধ করে আবার চালু করো               | Database connection lost        |
| Stop           | Actor স্থায়ীভাবে বন্ধ করো                 | Unrecoverable error            |
| Escalate       | Parent-এর parent-কে জানাও                | বোঝা যাচ্ছে না কী করবো          |

```
  Child Actor ব্যর্থ হলে:

  +------------------+
  | Supervisor Actor |
  +--------+---------+
           |
     কী করবো?
           |
   +-------+-------+-------+
   |       |       |       |
Resume  Restart  Stop  Escalate
   |       |       |       |
  চলো    নতুন    বন্ধ   উপরে
  থাকো   করে     করো   জানাও
         দাও
```

**Daraz উদাহরণ:**
- Order Actor crash করলে → **Restart** (নতুন করে order process করো)
- Payment Actor বারবার fail করলে → **Stop** + user-কে error দেখাও
- অজানা error → **Escalate** (system supervisor সিদ্ধান্ত নেবে)

---

## 🐘 PHP তে অ্যাক্টর মডেল

PHP মূলত synchronous ভাষা, কিন্তু **ReactPHP** এবং **Amphp** দিয়ে actor-like
pattern implement করা যায়।

### ❌ ভুল পদ্ধতি — Shared State দিয়ে concurrency

```php
<?php
// ❌ ভুল: Shared state ব্যবহার — Race condition হবে!
class UnsafeOrderProcessor
{
    public static int $totalOrders = 0; // shared state!

    public function process(): void
    {
        // Multiple process একই সাথে এটা change করলে data corrupt হবে
        self::$totalOrders++;
        // ৳ calculation ভুল হতে পারে!
    }
}
```

### ✅ সঠিক পদ্ধতি — ReactPHP দিয়ে Actor Pattern

```php
<?php
// ✅ সঠিক: Actor pattern — কোনো shared state নেই
use React\EventLoop\Loop;

class Actor
{
    private array $mailbox = [];
    private $behavior;

    public function __construct(callable $behavior)
    {
        $this->behavior = $behavior;
    }

    public function send(mixed $message): void
    {
        $this->mailbox[] = $message;
        Loop::futureTick(fn() => $this->processNext());
    }

    private function processNext(): void
    {
        if (!empty($this->mailbox)) {
            $message = array_shift($this->mailbox);
            ($this->behavior)($message, $this);
        }
    }
}

// === bKash Transaction Actor ===
$transactionActor = new Actor(function (mixed $msg, Actor $self) {
    match ($msg['type']) {
        'SEND_MONEY' => printf(
            "💸 ৳%d পাঠানো হচ্ছে %s-কে\n",
            $msg['amount'],
            $msg['to']
        ),
        'CASH_OUT' => printf(
            "🏧 ৳%d ক্যাশ আউট হচ্ছে\n",
            $msg['amount']
        ),
        default => printf("❓ অজানা message: %s\n", $msg['type']),
    };
});

// === Notification Actor ===
$notificationActor = new Actor(function (mixed $msg, Actor $self) {
    printf("📱 SMS পাঠানো হচ্ছে: %s\n", $msg['text']);
});

// Message পাঠানো
$transactionActor->send([
    'type'   => 'SEND_MONEY',
    'amount' => 500,
    'to'     => '01712345678',
]);

$transactionActor->send([
    'type'   => 'CASH_OUT',
    'amount' => 1000,
]);

$notificationActor->send([
    'text' => 'আপনার bKash একাউন্ট থেকে ৳500 পাঠানো হয়েছে।',
]);

Loop::get()->run();
```

### 🐘 PHP তে Actor System (উন্নত সংস্করণ)

```php
<?php
use React\EventLoop\Loop;

class ActorSystem
{
    private array $actors = [];

    public function createActor(string $name, callable $behavior): Actor
    {
        $actor = new Actor($behavior);
        $this->actors[$name] = $actor;
        return $actor;
    }

    public function getActor(string $name): ?Actor
    {
        return $this->actors[$name] ?? null;
    }

    public function run(): void
    {
        Loop::get()->run();
    }
}

// Daraz Order Processing System
$system = new ActorSystem();

// Order Actor
$system->createActor('order', function (mixed $msg, Actor $self) use ($system) {
    if ($msg['type'] === 'NEW_ORDER') {
        echo "📦 নতুন অর্ডার #{$msg['orderId']} প্রক্রিয়া হচ্ছে\n";

        // Payment actor-কে message পাঠানো
        $system->getActor('payment')?->send([
            'type'    => 'CHARGE',
            'orderId' => $msg['orderId'],
            'amount'  => $msg['amount'],
        ]);
    }
});

// Payment Actor
$system->createActor('payment', function (mixed $msg, Actor $self) use ($system) {
    if ($msg['type'] === 'CHARGE') {
        echo "💳 অর্ডার #{$msg['orderId']}-এর জন্য ৳{$msg['amount']} চার্জ হচ্ছে\n";

        // Notification actor-কে message পাঠানো
        $system->getActor('notification')?->send([
            'type' => 'SEND',
            'text' => "অর্ডার #{$msg['orderId']} সফলভাবে পেমেন্ট হয়েছে!",
        ]);
    }
});

// Notification Actor
$system->createActor('notification', function (mixed $msg, Actor $self) {
    if ($msg['type'] === 'SEND') {
        echo "📨 বিজ্ঞপ্তি: {$msg['text']}\n";
    }
});

// অর্ডার তৈরি
$system->getActor('order')?->send([
    'type'    => 'NEW_ORDER',
    'orderId' => 7890,
    'amount'  => 2500,
]);

$system->run();
```

---

## ⚡ JavaScript এ অ্যাক্টর মডেল

JavaScript-এ `worker_threads` ব্যবহার করে actor model implement করা যায়। প্রতিটি
Worker একটি actor — নিজস্ব memory, message-এর মাধ্যমে communication।

### worker_threads দিয়ে Actor

```javascript
// actor-worker.js — প্রতিটি worker একটি actor
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // === Main Thread: Actor System ===

    // Pathao Ride-Matching Actor System
    const rideActor = new Worker(__filename, {
        workerData: { role: 'ride-matcher' }
    });

    const driverActor = new Worker(__filename, {
        workerData: { role: 'driver-tracker' }
    });

    // Ride request পাঠানো
    rideActor.postMessage({
        type: 'MATCH_RIDE',
        userId: 'user_101',
        pickup: 'Dhanmondi 27',
        destination: 'Gulshan 2',
    });

    // Driver location update
    driverActor.postMessage({
        type: 'UPDATE_LOCATION',
        driverId: 'driver_55',
        lat: 23.7461,
        lng: 90.3742,
    });

    // Actor থেকে response শোনা
    rideActor.on('message', (msg) => {
        console.log('🚗 Ride Actor:', msg);

        // Ride match হলে driver actor-কে জানানো
        if (msg.status === 'matched') {
            driverActor.postMessage({
                type: 'ASSIGN_RIDE',
                driverId: msg.driverId,
                rideId: msg.rideId,
            });
        }
    });

    driverActor.on('message', (msg) => {
        console.log('📍 Driver Actor:', msg);
    });

} else {
    // === Worker Thread: Individual Actor ===

    const role = workerData.role;

    parentPort.on('message', (msg) => {
        switch (role) {
            case 'ride-matcher':
                handleRideMatching(msg);
                break;
            case 'driver-tracker':
                handleDriverTracking(msg);
                break;
        }
    });

    function handleRideMatching(msg) {
        if (msg.type === 'MATCH_RIDE') {
            // Simulate ride matching logic
            const rideId = `ride_${Date.now()}`;
            parentPort.postMessage({
                status: 'matched',
                rideId,
                driverId: 'driver_55',
                pickup: msg.pickup,
                destination: msg.destination,
                estimatedFare: '৳180',
            });
        }
    }

    function handleDriverTracking(msg) {
        if (msg.type === 'UPDATE_LOCATION') {
            parentPort.postMessage({
                status: 'location_updated',
                driverId: msg.driverId,
                position: { lat: msg.lat, lng: msg.lng },
            });
        }
        if (msg.type === 'ASSIGN_RIDE') {
            parentPort.postMessage({
                status: 'ride_assigned',
                driverId: msg.driverId,
                rideId: msg.rideId,
            });
        }
    }
}
```

### 📦 সরল Actor Class (JavaScript)

```javascript
// simple-actor.js — Lightweight actor pattern
class Actor {
    #state;
    #mailbox = [];
    #processing = false;

    constructor(initialState, handler) {
        this.#state = initialState;
        this.handler = handler;
    }

    send(message) {
        this.#mailbox.push(message);
        if (!this.#processing) {
            this.#processNext();
        }
    }

    async #processNext() {
        this.#processing = true;

        while (this.#mailbox.length > 0) {
            const msg = this.#mailbox.shift();
            this.#state = await this.handler(this.#state, msg);
        }

        this.#processing = false;
    }

    getState() {
        // শুধু debugging-এর জন্য — production-এ সরিয়ে দিন
        return { ...this.#state };
    }
}

// === Nagad Wallet Actor ===
const walletActor = new Actor(
    { balance: 5000, transactions: [] },
    async (state, msg) => {
        switch (msg.type) {
            case 'DEPOSIT':
                console.log(`💰 জমা: ৳${msg.amount}`);
                return {
                    balance: state.balance + msg.amount,
                    transactions: [
                        ...state.transactions,
                        { type: 'deposit', amount: msg.amount, time: Date.now() },
                    ],
                };

            case 'WITHDRAW':
                if (state.balance < msg.amount) {
                    console.log(`❌ অপর্যাপ্ত ব্যালেন্স! বর্তমান: ৳${state.balance}`);
                    return state;
                }
                console.log(`💸 উত্তোলন: ৳${msg.amount}`);
                return {
                    balance: state.balance - msg.amount,
                    transactions: [
                        ...state.transactions,
                        { type: 'withdraw', amount: msg.amount, time: Date.now() },
                    ],
                };

            case 'CHECK_BALANCE':
                console.log(`💳 বর্তমান ব্যালেন্স: ৳${state.balance}`);
                return state;

            default:
                console.log(`⚠️ অজানা মেসেজ: ${msg.type}`);
                return state;
        }
    }
);

walletActor.send({ type: 'DEPOSIT', amount: 1000 });
walletActor.send({ type: 'WITHDRAW', amount: 300 });
walletActor.send({ type: 'CHECK_BALANCE' });
walletActor.send({ type: 'WITHDRAW', amount: 99999 });
```

---

## ⚖️ থ্রেড-ভিত্তিক vs অ্যাক্টর মডেল

### তুলনা টেবিল

| বিষয়              | থ্রেড-ভিত্তিক (Threads)         | অ্যাক্টর মডেল (Actors)            |
|-------------------|---------------------------------|------------------------------------|
| State শেয়ারিং      | Shared memory                   | কোনো shared state নেই              |
| Communication     | Locks, Mutexes, Semaphores      | Asynchronous messages             |
| Deadlock          | সম্ভব এবং সাধারণ                 | সাধারণত হয় না                     |
| Race condition    | Mutex ছাড়া প্রায় নিশ্চিত        | অসম্ভব (single mailbox)           |
| Scaling           | একটি মেশিনে সীমাবদ্ধ             | একাধিক মেশিনে বিতরণযোগ্য           |
| ডিবাগিং            | কঠিন (non-deterministic)        | সহজতর (message trace করা যায়)     |
| Error handling    | try-catch (crash spreads)       | Supervision tree (isolated)       |
| মেমোরি ব্যবহার     | কম (shared memory)              | বেশি (প্রতিটি actor আলাদা memory)  |
| Performance       | CPU-bound কাজে ভালো              | I/O-bound ও distributed-এ ভালো    |

### ❌ থ্রেড দিয়ে সমস্যা

```
Thread A                    Thread B
   |                           |
   |--- Lock(balance) -------->|  ❌ Thread B অপেক্ষা করছে
   |    balance += 500         |
   |--- Unlock(balance) ------>|
   |                           |--- Lock(balance)
   |                           |    balance -= 200
   |                           |--- Unlock(balance)
   |
   |  যদি Lock ভুলে যাই?
   |  Race condition! 💥
```

### ✅ অ্যাক্টর দিয়ে সমাধান

```
Actor A                    Account Actor              Actor B
   |                           |                         |
   |--- "Add ৳500" msg ------->|                         |
   |                           |-- process msg           |
   |                           |   balance += 500        |
   |                           |                         |
   |                           |<-- "Deduct ৳200" msg ---|
   |                           |-- process msg           |
   |                           |   balance -= 200        |
   |                           |                         |
   |  কোনো Lock দরকার নেই!                               |
   |  একটি একটি message process হয় ✅                    |
```

---

## 🏠 ব্যবহারের ক্ষেত্র

### ১. 💬 চ্যাট সিস্টেম (WhatsApp-এর মতো)

প্রতিটি user conversation একটি actor:

```
+------------------+
| Chat System      |
+------------------+
        |
   +----+----+----+----+
   |    |    |    |    |
   v    v    v    v    v
 User  User  User Group Group
 Chat  Chat  Chat Chat  Chat
  A     B     C   "BFF" "Office"
(Actor)(Actor)(Actor)(Actor)(Actor)
   |
   | message আসলে
   v
 +-----------------------+
 | User A Chat Actor     |
 | State:                |
 |   - unread: 3         |
 |   - last_seen: 10:30  |
 |   - messages: [...]   |
 | Mailbox:              |
 |   -> [new msg from B] |
 |   -> [typing from C]  |
 +-----------------------+
```

**সুবিধা:** ১ কোটি user-এর জন্য ১ কোটি actor — প্রতিটি স্বাধীনভাবে কাজ করে।

### ২. 🎮 গেম সার্ভার

প্রতিটি game entity (player, NPC, item) একটি actor:

```
Game World Actor
      |
  +---+---+---+---+
  |   |   |   |   |
 P1  P2  NPC  NPC Item
 🧙  🗡️  👹   🐉  💎
```

### ৩. 🛒 Daraz Flash Sale সিস্টেম

```
                    +-------------------+
  হাজার হাজার       |  Load Balancer    |
  request আসছে ---> |                   |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------+---+  +------+-----+  +-----+------+
     | Product    |  | Product    |  | Product    |
     | Actor #1   |  | Actor #2   |  | Actor #3   |
     | iPhone 15  |  | Samsung S24|  | Pixel 8    |
     | stock: 50  |  | stock: 100 |  | stock: 30  |
     +-----+------+  +------+-----+  +-----+------+
           |                |               |
           v                v               v
     +----------+    +----------+    +----------+
     | Order    |    | Order    |    | Order    |
     | Actor    |    | Actor    |    | Order    |
     +----------+    +----------+    +----------+
```

**কেন actor model?**
- প্রতিটি product-এর stock আলাদা actor manage করে
- ১০,০০০ জন একসাথে iPhone কিনতে চাইলেও stock ভুল হবে না
- কোনো Lock/Mutex দরকার নেই — mailbox FIFO তে একটি একটি process হয়

### ৪. 🚗 Pathao রাইড ম্যাচিং

```
+---------------+     "I need a ride"     +---------------+
|  Rider Actor  | ----------------------> | Ride Matcher  |
|  (আব্দুল্লাহ)  |                         |    Actor      |
+---------------+                         +-------+-------+
                                                  |
                                    "কাছের driver কে?"
                                                  |
                      +---------------------------+---------------------------+
                      |                           |                           |
              +-------+-------+           +-------+-------+           +-------+-------+
              | Driver Actor  |           | Driver Actor  |           | Driver Actor  |
              | রফিক (1.2km) |           | সেলিম (3.5km) |           | নাসির (0.8km) |
              | status: free  |           | status: busy  |           | status: free  |
              +-------+-------+           +---------------+           +-------+-------+
                      |                                                       |
                      |              +------------------+                     |
                      +----------->  | নাসির সবচেয়ে কাছে |  <-----------------+
                                     | এবং free!         |
                                     +--------+---------+
                                              |
                                     "Ride assign করো"
                                              |
                                              v
                                     +--------+---------+
                                     |  Notification    |
                                     |  Actor           |
                                     | "নাসির, নতুন রাইড!"|
                                     +------------------+
```

### ৫. 🌐 IoT সিস্টেম

বাংলাদেশের smart factory-তে প্রতিটি sensor একটি actor:

```
Factory Floor
      |
  +---+---+---+---+
  |   |   |   |   |
 🌡️   💧  ⚡  🔧  📷
Temp  Water Power Motor Camera
Actor Actor Actor Actor Actor
```

প্রতিটি sensor actor স্বাধীনভাবে ডেটা পড়ে এবং alert actor-কে message পাঠায়।

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন অ্যাক্টর মডেল ব্যবহার করবেন

| পরিস্থিতি                                    | কেন Actor Model                            |
|---------------------------------------------|---------------------------------------------|
| অনেক স্বাধীন entity আছে                      | প্রতিটি entity একটি actor হতে পারে            |
| Shared state সমস্যা করছে                    | Actor-এ কোনো shared state নেই               |
| Distributed system দরকার                    | Actor location-transparent                  |
| Fault tolerance চাই                          | Supervision tree error isolate করে           |
| Real-time processing (chat, game, IoT)       | Message-driven, low latency                 |
| Event-driven architecture                    | Actor = natural event handler               |
| High concurrency (Daraz flash sale)          | কোনো Lock/Mutex ছাড়াই safe                 |

### ❌ কখন অ্যাক্টর মডেল ব্যবহার করবেন না

| পরিস্থিতি                                    | কেন না                                      |
|---------------------------------------------|---------------------------------------------|
| সাধারণ CRUD অ্যাপ                             | Overkill — একটি DB transaction-ই যথেষ্ট     |
| ভারী CPU গণনা (ML, video encoding)           | Thread pool বেশি efficient                   |
| Strong consistency চাই                       | Actor eventual consistency দেয়              |
| Request-response synchronous API             | Actor async — REST API-তে complexity বাড়ায়  |
| ছোট team, সহজ project                       | শেখার খরচ বেশি, debugging কঠিন হতে পারে      |
| Tight latency budget (< 1ms)                | Message passing overhead থাকে               |

### 🤔 সিদ্ধান্ত নেওয়ার ফ্লোচার্ট

```
তোমার সিস্টেমে কি অনেক
স্বাধীন entity আছে?
        |
    +---+---+
    |       |
   হ্যাঁ     না ----> ❌ Actor লাগবে না
    |                  সাধারণ approach ব্যবহার করো
    v
Shared state নিয়ে
সমস্যা হচ্ছে?
    |
  +-+---+
  |     |
 হ্যাঁ    না ----> 🤷 Actor optional
  |              Thread pool বা queue দিয়েও হতে পারে
  v
Distributed বা
fault-tolerant চাই?
  |
+-+---+
|     |
হ্যাঁ    না ----> ✅ Actor সম্ভব, কিন্তু simpler options দেখো
|                (Redis queue, database locks)
v
✅ Actor Model ব্যবহার করো!
Akka, proto.actor, বা নিজে implement করো
```

---

## 📖 সারসংক্ষেপ

### মূল পয়েন্ট

```
+------------------------------------------------------+
|              অ্যাক্টর মডেল মনে রাখার সূত্র               |
|                                                      |
|  ১. সবকিছু Actor                                     |
|  ২. কোনো Shared State নেই                            |
|  ৩. শুধু Message দিয়ে কথা বলো                         |
|  ৪. একবারে একটি Message process করো                   |
|  ৫. Actor নতুন Actor বানাতে পারে                      |
|  ৬. Actor নিজের Behavior বদলাতে পারে                  |
|  ৭. Supervision Tree দিয়ে error handle করো            |
+------------------------------------------------------+
```

### ❌ Bad vs ✅ Good Pattern

```
❌ BAD: Shared mutable state
---------------------------------
let sharedCounter = 0;
// Thread 1: sharedCounter++
// Thread 2: sharedCounter++
// Result: 1? 2? কেউ জানে না! 💥

✅ GOOD: Actor with private state
---------------------------------
counterActor.send({ type: 'INCREMENT' })
counterActor.send({ type: 'INCREMENT' })
// Result: সবসময় 2 ✅
// কারণ mailbox FIFO-তে একটি একটি process হয়
```

### 🔗 আরও জানতে

| রিসোর্স                        | লিংক / বিষয়                               |
|-------------------------------|--------------------------------------------|
| Carl Hewitt-এর মূল পেপার       | "A Universal Modular ACTOR Formalism" 1973 |
| Akka Documentation            | akka.io — JVM actor framework              |
| proto.actor                   | proto.actor — multi-language actors         |
| Erlang/OTP                    | erlang.org — actor model ভাষা              |
| "Reactive Messaging Patterns" | Vaughn Vernon-এর বই                        |
| Microsoft Orleans             | .NET virtual actor framework               |
| ReactPHP                      | reactphp.org — PHP async                   |

---

> **💡 মনে রাখো:** অ্যাক্টর মডেল হলো এমন একটি প্যাটার্ন যেখানে প্রতিটি কম্পোনেন্ট
> একটি স্বাধীন সত্তা — ঠিক যেমন একটি বড় অফিসে প্রতিটি কর্মচারী নিজের ডেস্কে
> বসে নিজের কাজ করে, এবং অন্যদের সাথে শুধু ইমেইল (message) দিয়ে যোগাযোগ করে।
> কেউ কারো ডেস্কের ড্রয়ার (state) খুলতে পারে না! 🔒
