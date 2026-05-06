# 🚢 Bulkhead Pattern (Resource Isolation)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Bulkhead-এর প্রকারভেদ](#bulkhead-এর-প্রকারভেদ)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Bulkhead Sizing ও Monitoring](#bulkhead-sizing-ও-monitoring)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Bulkhead Pattern** হলো একটি system-এর বিভিন্ন অংশকে আলাদা আলাদা resource pool-এ ভাগ করা, যাতে একটি অংশ ব্যর্থ হলে অন্য অংশগুলো প্রভাবিত না হয়।

### নামকরণের উৎস — জাহাজের Bulkhead:
জাহাজের hull-এ watertight compartment (bulkhead) থাকে। একটি compartment-এ পানি ঢুকলে শুধু সেই compartment-ই ভরে — পুরো জাহাজ ডোবে না! Titanic-এরও bulkhead ছিল, কিন্তু যথেষ্ট ছিল না।

### মূল ধারণা:
- Resource isolation: প্রতিটি service/functionality-র জন্য আলাদা resource pool
- Failure containment: একটি অংশের failure অন্য অংশে ছড়ায় না
- Graceful degradation: একটি feature down হলে বাকিগুলো চলতে থাকে

---

## 🌍 বাস্তব জীবনের উদাহরণ

### Pathao — Ride Booking vs Notification Isolation

**সমস্যা**:
Pathao-র ride booking এবং push notification একই thread pool ব্যবহার করছিল। একদিন notification service-এর third-party provider (Firebase) slow হয়ে গেল। সব thread notification-এ আটকে গেল। ফলে ride booking-ও বন্ধ হয়ে গেল! 😱

**সমাধান — Bulkhead**:
```
আগে (একটি pool):
[Thread 1: Ride] [Thread 2: Notify] [Thread 3: Notify-stuck] 
[Thread 4: Notify-stuck] [Thread 5: Notify-stuck] ... 
সব thread আটকে! Ride booking বন্ধ!

পরে (আলাদা pool):
Ride Pool (20 threads):    [T1: Ride] [T2: Ride] [T3: Ride] ... ✓ চলছে!
Notify Pool (5 threads):   [T1: stuck] [T2: stuck] ... ✗ কিন্তু ride চলছে!
```

### Daraz Bangladesh — Product API vs Review API:
- Product listing: dedicated connection pool (50 connections)
- User reviews: separate pool (10 connections)
- Search: separate pool (30 connections)
- Reviews slow হলেও product listing সচল থাকে!

### Prothom Alo — Content vs Ads:
- News content loading: 80% resources
- Advertisement loading: 20% resources
- Ad network slow হলেও খবর load হয়!

---

## 🔧 Bulkhead-এর প্রকারভেদ

### 1. Thread Pool Isolation (Hystrix-style)
- প্রতিটি dependency-র জন্য আলাদা thread pool
- সবচেয়ে শক্তিশালী isolation
- Overhead: প্রতিটি pool-এর জন্য thread তৈরি

### 2. Semaphore Isolation
- Thread pool তৈরি না করে counter-based limit
- কম overhead
- Timeout support দুর্বল

### 3. Connection Pool Isolation
- Database/HTTP connection আলাদা pool-এ
- Critical vs non-critical query আলাদা pool

### 4. Process Isolation
- সম্পূর্ণ আলাদা process/container
- সবচেয়ে শক্তিশালী কিন্তু expensive
- Kubernetes pod-level isolation

### তুলনা:

| Type | Isolation Level | Overhead | Use Case |
|------|----------------|----------|----------|
| Thread Pool | High | Medium | External service calls |
| Semaphore | Medium | Low | Internal fast calls |
| Connection Pool | Medium | Low | Database isolation |
| Process | Very High | High | Critical workloads |

---

## 📊 ASCII ডায়াগ্রাম

### Bulkhead ছাড়া (সমস্যা):

```
┌─────────────────────────────────────────────────────┐
│              Pathao Application Server                │
│                                                      │
│  ┌───────────────── Shared Thread Pool ────────────┐ │
│  │                                                  │ │
│  │  [Ride] [Ride] [Notify] [Notify] [Notify]       │ │
│  │  [Food] [Notify] [Notify] [Notify] [Notify]    │ │
│  │  [Notify] [Notify] [Notify] [Notify] [Notify]  │ │
│  │                                                  │ │
│  │  ⚠️ Notification service SLOW!                   │ │
│  │  সব thread notify-তে আটকে গেছে!                │ │
│  │  Ride booking ❌ Food order ❌                    │ │
│  │                                                  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                      │
│  Result: পুরো system DOWN! 💀                        │
└─────────────────────────────────────────────────────┘
```

### Bulkhead সহ (সমাধান):

```
┌─────────────────────────────────────────────────────────┐
│               Pathao Application Server                   │
│                                                          │
│  ┌─────────────────────┐  ┌──────────────────────┐      │
│  │  🚗 Ride Pool (20)   │  │  🍔 Food Pool (15)   │      │
│  │                      │  │                       │      │
│  │  [Ride] [Ride] [Ride]│  │  [Food] [Food] [Food]│      │
│  │  [Ride] [Ride] [Ride]│  │  [Food] [Food] [Food]│      │
│  │  [Ride] [Ride] ...   │  │  [Food] [Food] ...   │      │
│  │                      │  │                       │      │
│  │  ✅ সচল!             │  │  ✅ সচল!              │      │
│  └─────────────────────┘  └──────────────────────┘      │
│                                                          │
│  ┌─────────────────────┐  ┌──────────────────────┐      │
│  │  📱 Notify Pool (5)  │  │  🔍 Search Pool (10) │      │
│  │                      │  │                       │      │
│  │  [stuck][stuck]      │  │  [Search] [Search]   │      │
│  │  [stuck][stuck]      │  │  [Search] [Search]   │      │
│  │  [stuck]             │  │  [Search] ...        │      │
│  │                      │  │                       │      │
│  │  ❌ আটকে আছে         │  │  ✅ সচল!              │      │
│  │  (কিন্তু বাকি সব ✓)  │  │                       │      │
│  └─────────────────────┘  └──────────────────────┘      │
│                                                          │
│  Result: শুধু notification ডাউন, বাকি সব সচল! ✓          │
└─────────────────────────────────────────────────────────┘
```

### Thread Pool vs Semaphore Isolation:

```
    Thread Pool Isolation              Semaphore Isolation
    ═══════════════════════            ═════════════════════

    ┌──────────────────┐               ┌──────────────────┐
    │ Dedicated Pool    │               │ Shared Threads    │
    │                   │               │                   │
    │ ┌──┐ ┌──┐ ┌──┐  │               │ ┌──┐ ┌──┐ ┌──┐  │
    │ │T1│ │T2│ │T3│  │               │ │T1│ │T2│ │T3│  │
    │ └──┘ └──┘ └──┘  │               │ └──┘ └──┘ └──┘  │
    │ ┌──┐ ┌──┐       │               │                   │
    │ │T4│ │T5│       │               │ Semaphore: ███░░  │
    │ └──┘ └──┘       │               │ (3/5 permits used)│
    └──────────────────┘               └──────────────────┘

    ✓ Full isolation                   ✓ Low overhead
    ✓ Timeout support                  ✓ No thread creation
    ✗ Thread overhead                  ✗ No timeout control
    ✗ Context switch cost              ✗ Caller thread blocks
```

### Bulkhead + Circuit Breaker Combination:

```
┌─────────────────────────────────────────────────────┐
│                 Request Flow                          │
│                                                      │
│  Request ──► [Bulkhead] ──► [Circuit Breaker] ──► Service
│                  │                  │                 │
│                  │ Pool full?       │ Circuit open?   │
│                  ▼                  ▼                 │
│            ┌──────────┐      ┌──────────┐            │
│            │ Reject   │      │ Fail Fast│            │
│            │ (503)    │      │ (fallback│            │
│            └──────────┘      └──────────┘            │
│                                                      │
│  Bulkhead: resource limit enforce করে                │
│  Circuit Breaker: failure pattern detect করে         │
│  একসাথে: complete resilience! 💪                     │
└─────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Bulkhead Pattern Implementation
 * Pathao Service Resource Isolation
 */

class Bulkhead
{
    private string $name;
    private int $maxConcurrent;
    private int $maxWaitQueue;
    private int $currentActive = 0;
    private int $waitingCount = 0;
    private int $rejectedCount = 0;
    private int $completedCount = 0;
    private array $metrics = [];

    public function __construct(string $name, int $maxConcurrent, int $maxWaitQueue = 10)
    {
        $this->name = $name;
        $this->maxConcurrent = $maxConcurrent;
        $this->maxWaitQueue = $maxWaitQueue;
    }

    /**
     * Bulkhead-protected execution
     */
    public function execute(callable $task): mixed
    {
        // Pool full কিনা check
        if ($this->currentActive >= $this->maxConcurrent) {
            // Wait queue-তে জায়গা আছে কিনা
            if ($this->waitingCount >= $this->maxWaitQueue) {
                $this->rejectedCount++;
                $this->recordMetric('rejected');
                throw new BulkheadRejectedException(
                    "🚫 [{$this->name}] Bulkhead full! " .
                    "Active: {$this->currentActive}/{$this->maxConcurrent}, " .
                    "Waiting: {$this->waitingCount}/{$this->maxWaitQueue}"
                );
            }
            $this->waitingCount++;
            $this->recordMetric('queued');
        }

        // Execute task
        $this->currentActive++;
        $this->waitingCount = max(0, $this->waitingCount - 1);
        $startTime = microtime(true);

        try {
            $result = $task();
            $this->completedCount++;
            $this->recordMetric('completed', microtime(true) - $startTime);
            return $result;
        } catch (\Exception $e) {
            $this->recordMetric('failed', microtime(true) - $startTime);
            throw $e;
        } finally {
            $this->currentActive--;
        }
    }

    private function recordMetric(string $event, float $duration = 0): void
    {
        $this->metrics[] = [
            'event' => $event,
            'timestamp' => microtime(true),
            'duration' => $duration,
            'active' => $this->currentActive,
        ];
    }

    public function getStats(): array
    {
        return [
            'name' => $this->name,
            'maxConcurrent' => $this->maxConcurrent,
            'currentActive' => $this->currentActive,
            'waiting' => $this->waitingCount,
            'completed' => $this->completedCount,
            'rejected' => $this->rejectedCount,
            'utilization' => round(($this->currentActive / $this->maxConcurrent) * 100, 1) . '%',
        ];
    }
}

class BulkheadRejectedException extends \RuntimeException {}

/**
 * Semaphore-based Bulkhead (lightweight)
 */
class SemaphoreBulkhead
{
    private string $name;
    private int $maxPermits;
    private int $availablePermits;
    private int $timeoutMs;

    public function __construct(string $name, int $maxPermits, int $timeoutMs = 5000)
    {
        $this->name = $name;
        $this->maxPermits = $maxPermits;
        $this->availablePermits = $maxPermits;
        $this->timeoutMs = $timeoutMs;
    }

    public function tryAcquire(): bool
    {
        if ($this->availablePermits > 0) {
            $this->availablePermits--;
            return true;
        }
        return false;
    }

    public function release(): void
    {
        if ($this->availablePermits < $this->maxPermits) {
            $this->availablePermits++;
        }
    }

    public function execute(callable $task): mixed
    {
        if (!$this->tryAcquire()) {
            throw new BulkheadRejectedException(
                "🚫 [{$this->name}] Semaphore exhausted! " .
                "Permits: 0/{$this->maxPermits}"
            );
        }

        try {
            return $task();
        } finally {
            $this->release();
        }
    }
}

/**
 * Pathao Service with Bulkhead Isolation
 */
class PathaoServiceManager
{
    private Bulkhead $rideBulkhead;
    private Bulkhead $foodBulkhead;
    private Bulkhead $notificationBulkhead;
    private SemaphoreBulkhead $searchBulkhead;

    public function __construct()
    {
        // প্রতিটি service-এর জন্য আলাদা bulkhead
        $this->rideBulkhead = new Bulkhead('🚗 Ride-Booking', 20, 50);
        $this->foodBulkhead = new Bulkhead('🍔 Food-Order', 15, 30);
        $this->notificationBulkhead = new Bulkhead('📱 Notification', 5, 10);
        $this->searchBulkhead = new SemaphoreBulkhead('🔍 Search', 10);
    }

    /**
     * Ride booking — Critical, বেশি resource
     */
    public function bookRide(string $pickup, string $destination, string $userId): array
    {
        return $this->rideBulkhead->execute(function () use ($pickup, $destination, $userId) {
            echo "  🚗 Ride booking: {$pickup} → {$destination}\n";
            usleep(rand(500, 2000) * 1000); // 500ms-2s processing

            return [
                'status' => 'confirmed',
                'rideId' => 'RIDE_' . uniqid(),
                'pickup' => $pickup,
                'destination' => $destination,
                'estimatedFare' => rand(80, 500),
            ];
        });
    }

    /**
     * Food order — Medium priority
     */
    public function orderFood(string $restaurant, array $items, string $userId): array
    {
        return $this->foodBulkhead->execute(function () use ($restaurant, $items, $userId) {
            echo "  🍔 Food order: {$restaurant}\n";
            usleep(rand(300, 1500) * 1000);

            return [
                'status' => 'placed',
                'orderId' => 'FOOD_' . uniqid(),
                'restaurant' => $restaurant,
                'estimatedTime' => rand(20, 45) . ' min',
            ];
        });
    }

    /**
     * Notification — Low priority, failure acceptable
     */
    public function sendNotification(string $userId, string $message): array
    {
        try {
            return $this->notificationBulkhead->execute(function () use ($userId, $message) {
                echo "  📱 Sending notification to {$userId}\n";
                usleep(rand(100, 5000) * 1000); // Sometimes slow!

                return ['status' => 'sent', 'userId' => $userId];
            });
        } catch (BulkheadRejectedException $e) {
            // Notification failure is acceptable — queue for later
            echo "  ⚠️ Notification queued for later: {$e->getMessage()}\n";
            return ['status' => 'queued', 'userId' => $userId];
        }
    }

    /**
     * Search — Semaphore-based (lightweight)
     */
    public function searchDrivers(string $location): array
    {
        return $this->searchBulkhead->execute(function () use ($location) {
            echo "  🔍 Searching drivers near {$location}\n";
            usleep(rand(100, 500) * 1000);

            return [
                'driversFound' => rand(3, 15),
                'nearestEta' => rand(2, 10) . ' min',
            ];
        });
    }

    /**
     * সব bulkhead-এর অবস্থা দেখা
     */
    public function getSystemHealth(): void
    {
        echo "\n📊 ═══ System Health Dashboard ═══\n";
        $bulkheads = [
            $this->rideBulkhead->getStats(),
            $this->foodBulkhead->getStats(),
            $this->notificationBulkhead->getStats(),
        ];

        foreach ($bulkheads as $stats) {
            $status = $stats['rejected'] > 0 ? '⚠️' : '✅';
            echo "  {$status} {$stats['name']}: ";
            echo "Active={$stats['currentActive']}/{$stats['maxConcurrent']} ";
            echo "Util={$stats['utilization']} ";
            echo "Rejected={$stats['rejected']}\n";
        }
    }
}

// ========== ব্যবহার ==========

echo "🚢 ═══ Pathao Bulkhead Demo ═══\n\n";

$pathao = new PathaoServiceManager();

// Normal operation
echo "── Normal Operations ──\n";
$ride = $pathao->bookRide('মিরপুর-১০', 'গুলশান-১', 'user_001');
echo "  Result: {$ride['status']} (Fare: ৳{$ride['estimatedFare']})\n\n";

$food = $pathao->orderFood('Sultan\'s Dine', ['Kacchi', 'Borhani'], 'user_002');
echo "  Result: {$food['status']} (ETA: {$food['estimatedTime']})\n\n";

// Notification pool-এ load
echo "── Notification Stress Test ──\n";
for ($i = 0; $i < 8; $i++) {
    $pathao->sendNotification("user_00{$i}", "Your ride is arriving!");
}

echo "\n── Ride still works even if notification is overwhelmed ──\n";
$ride2 = $pathao->bookRide('ধানমন্ডি', 'বনানী', 'user_010');
echo "  ✅ Ride booking successful: {$ride2['rideId']}\n";

$pathao->getSystemHealth();
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Bulkhead Pattern — Thread Pool & Semaphore Isolation
 * Pathao Microservices Architecture
 */

class Bulkhead {
    constructor(name, options = {}) {
        this.name = name;
        this.maxConcurrent = options.maxConcurrent || 10;
        this.maxQueue = options.maxQueue || 20;
        this.timeout = options.timeout || 30000;

        this.activeCount = 0;
        this.queue = [];
        this.metrics = {
            totalExecuted: 0,
            totalRejected: 0,
            totalTimeout: 0,
            totalSuccess: 0,
            totalFailure: 0,
            avgDuration: 0,
        };
    }

    /**
     * Bulkhead-protected execution
     */
    async execute(task) {
        // Capacity check
        if (this.activeCount >= this.maxConcurrent) {
            if (this.queue.length >= this.maxQueue) {
                this.metrics.totalRejected++;
                throw new BulkheadFullError(
                    `🚫 [${this.name}] Full! ` +
                    `Active: ${this.activeCount}/${this.maxConcurrent}, ` +
                    `Queue: ${this.queue.length}/${this.maxQueue}`
                );
            }

            // Queue-তে অপেক্ষা
            await this.waitInQueue();
        }

        this.activeCount++;
        const startTime = Date.now();

        try {
            // Timeout সহ execution
            const result = await this.executeWithTimeout(task);
            this.metrics.totalSuccess++;
            this.updateAvgDuration(Date.now() - startTime);
            return result;
        } catch (error) {
            if (error instanceof TimeoutError) {
                this.metrics.totalTimeout++;
            } else {
                this.metrics.totalFailure++;
            }
            throw error;
        } finally {
            this.activeCount--;
            this.metrics.totalExecuted++;
            this.processQueue();
        }
    }

    /**
     * Queue-তে অপেক্ষা করা
     */
    waitInQueue() {
        return new Promise((resolve, reject) => {
            const queueItem = { resolve, reject };

            const timer = setTimeout(() => {
                const index = this.queue.indexOf(queueItem);
                if (index !== -1) {
                    this.queue.splice(index, 1);
                    reject(new TimeoutError(
                        `[${this.name}] Queue timeout after ${this.timeout}ms`
                    ));
                }
            }, this.timeout);

            queueItem.timer = timer;
            this.queue.push(queueItem);
        });
    }

    /**
     * Queue process করা (slot available হলে)
     */
    processQueue() {
        if (this.queue.length > 0 && this.activeCount < this.maxConcurrent) {
            const item = this.queue.shift();
            clearTimeout(item.timer);
            item.resolve();
        }
    }

    /**
     * Timeout সহ task execution
     */
    executeWithTimeout(task) {
        return new Promise(async (resolve, reject) => {
            const timer = setTimeout(() => {
                reject(new TimeoutError(`[${this.name}] Task timeout: ${this.timeout}ms`));
            }, this.timeout);

            try {
                const result = await task();
                clearTimeout(timer);
                resolve(result);
            } catch (error) {
                clearTimeout(timer);
                reject(error);
            }
        });
    }

    updateAvgDuration(duration) {
        const total = this.metrics.totalSuccess;
        this.metrics.avgDuration =
            (this.metrics.avgDuration * (total - 1) + duration) / total;
    }

    getStats() {
        return {
            name: this.name,
            active: this.activeCount,
            maxConcurrent: this.maxConcurrent,
            queueSize: this.queue.length,
            maxQueue: this.maxQueue,
            utilization: `${Math.round((this.activeCount / this.maxConcurrent) * 100)}%`,
            ...this.metrics,
        };
    }
}

class BulkheadFullError extends Error {
    constructor(message) {
        super(message);
        this.name = 'BulkheadFullError';
    }
}

class TimeoutError extends Error {
    constructor(message) {
        super(message);
        this.name = 'TimeoutError';
    }
}

/**
 * Bulkhead Registry — সব bulkhead manage করা
 */
class BulkheadRegistry {
    constructor() {
        this.bulkheads = new Map();
    }

    register(name, options) {
        const bulkhead = new Bulkhead(name, options);
        this.bulkheads.set(name, bulkhead);
        return bulkhead;
    }

    get(name) {
        return this.bulkheads.get(name);
    }

    getHealthReport() {
        const report = {};
        for (const [name, bulkhead] of this.bulkheads) {
            report[name] = bulkhead.getStats();
        }
        return report;
    }

    printDashboard() {
        console.log('\n📊 ═══ Bulkhead Health Dashboard ═══');
        console.log('─'.repeat(60));

        for (const [name, bulkhead] of this.bulkheads) {
            const stats = bulkhead.getStats();
            const bar = this.makeProgressBar(stats.active, stats.maxConcurrent);
            const status = stats.totalRejected > 0 ? '⚠️' : '✅';

            console.log(`${status} ${stats.name}`);
            console.log(`   ${bar} ${stats.utilization}`);
            console.log(`   Active: ${stats.active}/${stats.maxConcurrent} | ` +
                       `Queue: ${stats.queueSize}/${stats.maxQueue} | ` +
                       `Rejected: ${stats.totalRejected}`);
            console.log('');
        }
    }

    makeProgressBar(current, max, width = 20) {
        const filled = Math.round((current / max) * width);
        const empty = width - filled;
        return '█'.repeat(filled) + '░'.repeat(empty);
    }
}

/**
 * Pathao Service Manager with Bulkhead Isolation
 */
class PathaoServiceManager {
    constructor() {
        this.registry = new BulkheadRegistry();

        // Service-specific bulkheads
        this.rideBulkhead = this.registry.register('🚗 Ride-Booking', {
            maxConcurrent: 20,
            maxQueue: 50,
            timeout: 15000,
        });

        this.foodBulkhead = this.registry.register('🍔 Food-Order', {
            maxConcurrent: 15,
            maxQueue: 30,
            timeout: 20000,
        });

        this.notifyBulkhead = this.registry.register('📱 Notification', {
            maxConcurrent: 5,
            maxQueue: 10,
            timeout: 5000,
        });

        this.paymentBulkhead = this.registry.register('💳 Payment', {
            maxConcurrent: 25,
            maxQueue: 40,
            timeout: 30000,
        });
    }

    /**
     * Ride Booking — High Priority
     */
    async bookRide(pickup, destination) {
        return this.rideBulkhead.execute(async () => {
            await this.simulateWork(500, 2000);
            return {
                rideId: `RIDE_${Date.now()}`,
                pickup,
                destination,
                fare: Math.floor(Math.random() * 400) + 80,
                eta: `${Math.floor(Math.random() * 8) + 2} min`,
            };
        });
    }

    /**
     * Food Order — Medium Priority
     */
    async orderFood(restaurant, items) {
        return this.foodBulkhead.execute(async () => {
            await this.simulateWork(300, 1500);
            return {
                orderId: `FOOD_${Date.now()}`,
                restaurant,
                items,
                deliveryTime: `${Math.floor(Math.random() * 30) + 15} min`,
            };
        });
    }

    /**
     * Notification — Low Priority (graceful degradation)
     */
    async sendNotification(userId, message) {
        try {
            return await this.notifyBulkhead.execute(async () => {
                await this.simulateWork(100, 3000);
                return { status: 'sent', userId };
            });
        } catch (error) {
            if (error instanceof BulkheadFullError) {
                console.log(`   ⚠️ Notification queued async: ${userId}`);
                return { status: 'queued_async', userId };
            }
            throw error;
        }
    }

    /**
     * Payment — Critical (separate pool)
     */
    async processPayment(amount, method) {
        return this.paymentBulkhead.execute(async () => {
            await this.simulateWork(200, 1000);
            return {
                paymentId: `PAY_${Date.now()}`,
                amount,
                method,
                status: 'completed',
            };
        });
    }

    simulateWork(minMs, maxMs) {
        const duration = Math.floor(Math.random() * (maxMs - minMs)) + minMs;
        return new Promise(resolve => setTimeout(resolve, duration));
    }
}

// ========== ব্যবহার ==========

async function demo() {
    console.log('🚢 ═══ Pathao Bulkhead Pattern Demo ═══\n');

    const pathao = new PathaoServiceManager();

    // Concurrent requests simulate করা
    console.log('── Simulating 30 concurrent requests ──\n');

    const requests = [];

    // 10 ride bookings
    for (let i = 0; i < 10; i++) {
        requests.push(
            pathao.bookRide('মিরপুর', 'গুলশান')
                .then(r => console.log(`   ✅ Ride: ${r.rideId} (৳${r.fare})`))
                .catch(e => console.log(`   ❌ Ride failed: ${e.message}`))
        );
    }

    // 10 notifications (some will be rejected — bulkhead is small)
    for (let i = 0; i < 10; i++) {
        requests.push(
            pathao.sendNotification(`user_${i}`, 'Your driver is nearby!')
                .then(r => console.log(`   📱 Notify ${r.userId}: ${r.status}`))
                .catch(e => console.log(`   ❌ Notify failed: ${e.message}`))
        );
    }

    // 5 food orders
    for (let i = 0; i < 5; i++) {
        requests.push(
            pathao.orderFood("Sultan's Dine", ['Kacchi'])
                .then(r => console.log(`   ✅ Food: ${r.orderId}`))
                .catch(e => console.log(`   ❌ Food failed: ${e.message}`))
        );
    }

    // 5 payments
    for (let i = 0; i < 5; i++) {
        requests.push(
            pathao.processPayment(500 + i * 100, 'bKash')
                .then(r => console.log(`   ✅ Payment: ${r.paymentId} (৳${r.amount})`))
                .catch(e => console.log(`   ❌ Payment failed: ${e.message}`))
        );
    }

    await Promise.allSettled(requests);

    // Health Dashboard
    pathao.registry.printDashboard();
}

demo().catch(console.error);
```

---

## 📏 Bulkhead Sizing ও Monitoring

### Sizing Strategy:

```
Bulkhead Size = (Peak RPS × Average Latency) × Safety Factor

উদাহরণ — Pathao Ride Booking:
- Peak RPS: 100 requests/second
- Average Latency: 200ms (0.2s)
- Safety Factor: 1.5

Size = 100 × 0.2 × 1.5 = 30 concurrent slots

Queue Size = Size × 2 = 60 (burst handling)
```

### Monitoring Metrics:

| Metric | Warning Threshold | Critical Threshold |
|--------|------------------|--------------------|
| Utilization | > 70% | > 90% |
| Rejection Rate | > 1% | > 5% |
| Queue Wait Time | > 1s | > 5s |
| Timeout Rate | > 2% | > 10% |

### Alert Rules:
```
IF bulkhead.utilization > 80% FOR 5 minutes
  → ALERT: "Ride pool high utilization — consider scaling"

IF bulkhead.rejectionRate > 5%
  → ALERT: "Notification pool overflow — check Firebase"
  
IF bulkhead.avgWaitTime > 2s
  → ALERT: "Payment pool congested — investigate bank API"
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন:
- 🚗 Critical service isolation (Pathao ride vs notification)
- 🏦 Payment service আলাদা pool (bKash transactions)
- 📰 Content vs Ads isolation (Prothom Alo)
- 🗄️ Database connection isolation (read vs write pool)
- 🌐 Third-party API calls (Firebase, SMS gateway)
- 🔄 Async task processing (email, report generation)

### ❌ ব্যবহার করবেন না:
- 📦 Single-purpose microservice (একটাই কাজ করে)
- 🧮 CPU-bound tasks (thread pool help করবে না)
- 🔗 Tightly coupled components (isolation meaningless)
- 📊 Very low traffic systems (overhead > benefit)
- 🏠 Monolithic applications (process-level isolation better)

### 🎯 Bulkhead + Circuit Breaker কখন একসাথে:

```
Scenario: Pathao → Firebase Push Notification

┌─────────────┐     ┌───────────┐     ┌────────────────┐
│   Request   │────>│ Bulkhead  │────>│ Circuit Breaker│────> Firebase
│             │     │ (5 slots) │     │                │
└─────────────┘     └───────────┘     └────────────────┘

Bulkhead: সর্বোচ্চ ৫টি concurrent notification call
Circuit Breaker: ৫০% failure → circuit OPEN → fail fast

একসাথে: 
- Bulkhead = resource protection (অন্য service safe)
- CB = fast failure detection (অপ্রয়োজনীয় retry বন্ধ)
```

---

## 📚 সারসংক্ষেপ

Bulkhead Pattern হলো "damage control" — সমস্যা হবেই, কিন্তু সমস্যা যাতে ছড়িয়ে না পড়ে সেটা নিশ্চিত করে। জাহাজের মতো, আপনার system-কেও compartment-এ ভাগ করুন। একটি compartment-এ পানি ঢুকলে (failure হলে) বাকি compartment-গুলো (services) নিরাপদ থাকবে।

**মূল নীতি**: "একটি খারাপ dependency যেন পুরো system-কে ডোবাতে না পারে!" 🚢
