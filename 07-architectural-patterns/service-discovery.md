# 🔍 Service Discovery Pattern (সার্ভিস ডিসকভারি প্যাটার্ন)

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [Client-Side Discovery](#client-side-discovery)
- [Server-Side Discovery](#server-side-discovery)
- [Service Registry](#service-registry)
- [Health Check ও Heartbeat](#health-check-ও-heartbeat)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Service Discovery** হলো মাইক্রোসার্ভিস আর্কিটেকচারে একটি প্যাটার্ন যেখানে সার্ভিসগুলো স্বয়ংক্রিয়ভাবে অন্য সার্ভিসের নেটওয়ার্ক লোকেশন (IP address ও port) খুঁজে বের করে।

### 🤔 কেন Service Discovery দরকার?

মাইক্রোসার্ভিস environment-এ সার্ভিসের instance সংখ্যা ক্রমাগত পরিবর্তন হয়:
- Auto-scaling এ নতুন instance যোগ হয়
- কোনো instance crash করলে সরিয়ে ফেলা হয়
- নতুন deployment এ IP পরিবর্তন হয়
- Container orchestration এ dynamic port allocation হয়

**Static configuration** (হার্ডকোড IP) এই পরিস্থিতিতে কাজ করে না। তাই আমাদের **dynamic service discovery** প্রয়োজন।

### 📖 মূল ধারণাগুলো:

| ধারণা | বিবরণ |
|-------|--------|
| **Service Registry** | সার্ভিসের address book - কোন সার্ভিস কোথায় আছে তার তালিকা |
| **Registration** | সার্ভিস নিজেকে registry-তে নিবন্ধন করা |
| **Discovery** | অন্য সার্ভিসের location খুঁজে বের করা |
| **Health Check** | সার্ভিস সচল আছে কিনা যাচাই করা |
| **Heartbeat** | নিয়মিত সংকেত পাঠানো যে সার্ভিস জীবিত আছে |

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏍️ Pathao রাইড-ম্যাচিং সিস্টেম

ধরুন Pathao-র মাইক্রোসার্ভিস আর্কিটেকচার:

```
Pathao সিস্টেমের সার্ভিসগুলো:
├── 🏍️ Ride Matching Service (রাইডার-ড্রাইভার ম্যাচিং)
├── 💰 Payment Service (bKash, Nagad, Card পেমেন্ট)
├── 📱 Notification Service (Push notification, SMS)
├── 📍 Location Service (GPS tracking)
├── 👤 User Service (প্রোফাইল, রেটিং)
└── 📊 Pricing Service (ভাড়া হিসাব)
```

**সমস্যা:** Ride Matching Service-কে Payment Service-এর সাথে কথা বলতে হবে। কিন্তু Payment Service-এর ৫টি instance চলছে বিভিন্ন সার্ভারে। কোনটিতে request পাঠাবে?

**সমাধান:** Service Discovery! Payment Service নিজেকে registry-তে register করে, Ride Matching Service registry থেকে Payment Service-এর active instance খুঁজে নেয়।

### 📞 Grameenphone MyGP App

MyGP অ্যাপে যখন আপনি ব্যালেন্স চেক করেন:
1. **API Gateway** → Service Registry-তে Balance Service খোঁজে
2. **Balance Service** → Billing Service discover করে
3. **Billing Service** → Plan Service discover করে
4. প্রতিটি ধাপে dynamic discovery হয়

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Client-Side Discovery Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT-SIDE DISCOVERY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐         ┌──────────────────┐                   │
│  │              │  1.Query │                  │                   │
│  │   Ride       │────────►│  Service Registry │                   │
│  │   Matching   │◄────────│  (Consul/etcd)   │                   │
│  │   Service    │ 2.Return│                  │                   │
│  │              │  IPs    │  ┌────────────┐  │                   │
│  └──────┬───────┘         │  │ payment:   │  │                   │
│         │                 │  │ 10.0.1.5   │  │                   │
│         │ 3. Direct       │  │ 10.0.1.6   │  │                   │
│         │    Call          │  │ 10.0.1.7   │  │                   │
│         │                 │  └────────────┘  │                   │
│         ▼                 └──────────────────┘                   │
│  ┌──────────────────────────────────┐                            │
│  │      Payment Service Instances    │                            │
│  │  ┌────────┐ ┌────────┐ ┌────────┐│                           │
│  │  │10.0.1.5│ │10.0.1.6│ │10.0.1.7││                           │
│  │  │ :8080  │ │ :8080  │ │ :8080  ││                           │
│  │  └────────┘ └────────┘ └────────┘│                           │
│  └──────────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Server-Side Discovery Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVER-SIDE DISCOVERY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐         ┌──────────────────┐                   │
│  │              │ 1.Call   │                  │                   │
│  │   Ride       │────────►│   Load Balancer   │                   │
│  │   Matching   │         │   (AWS ALB /      │                   │
│  │   Service    │         │    K8s Service)   │                   │
│  │              │         │                  │                   │
│  └──────────────┘         └────────┬─────────┘                   │
│                                    │                             │
│                           2. LB queries                          │
│                              registry &                          │
│                              routes                              │
│                                    │                             │
│                                    ▼                             │
│  ┌──────────────────────────────────────────────┐               │
│  │         Payment Service Instances             │               │
│  │  ┌────────┐    ┌────────┐    ┌────────┐      │               │
│  │  │Instance│    │Instance│    │Instance│      │               │
│  │  │   1    │    │   2    │    │   3    │      │               │
│  │  └────────┘    └────────┘    └────────┘      │               │
│  └──────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### Self-Registration vs Third-Party Registration

```
┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│     SELF-REGISTRATION           │  │    THIRD-PARTY REGISTRATION     │
├─────────────────────────────────┤  ├─────────────────────────────────┤
│                                 │  │                                 │
│  ┌─────────┐    ┌──────────┐   │  │  ┌─────────┐    ┌──────────┐   │
│  │ Service │───►│ Registry │   │  │  │ Service │    │ Registry │   │
│  │         │    │          │   │  │  │         │    │          │   │
│  │ নিজেই   │    │          │   │  │  │         │    │     ▲    │   │
│  │ register│    │          │   │  │  │         │    │     │    │   │
│  │ করে     │    │          │   │  │  └─────────┘    └─────┼────┘   │
│  └─────────┘    └──────────┘   │  │       ▲               │        │
│                                 │  │       │ monitor  register      │
│  উদাহরণ: Netflix Eureka Client │  │       │               │        │
│                                 │  │  ┌────┴───────────────┴────┐   │
│                                 │  │  │      Registrar          │   │
│                                 │  │  │  (K8s, Docker Swarm)    │   │
│                                 │  │  └─────────────────────────┘   │
│                                 │  │                                 │
│                                 │  │  উদাহরণ: Kubernetes Service    │
└─────────────────────────────────┘  └─────────────────────────────────┘
```

### Heartbeat ও Stale Entry Handling

```
┌───────────────────────────────────────────────────────────────┐
│              HEARTBEAT MECHANISM                                │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Timeline:                                                     │
│  ═══════════════════════════════════════════════►              │
│                                                                │
│  Service A:  ♥───♥───♥───♥───♥───♥───♥  (সুস্থ - প্রতি 10s)  │
│                                                                │
│  Service B:  ♥───♥───♥───✗───✗───✗───☠  (মৃত - 30s পর remove)│
│                                                                │
│  Service C:  ♥───♥───♥───♥───♥  (সুস্থ)                       │
│                                                                │
│  Registry Actions:                                             │
│  ─────────────────                                             │
│  t=0s   : B registered ✓                                      │
│  t=10s  : B heartbeat received ✓                              │
│  t=20s  : B heartbeat received ✓                              │
│  t=30s  : B heartbeat MISSED ⚠️                                │
│  t=40s  : B heartbeat MISSED ⚠️⚠️                              │
│  t=50s  : B marked UNHEALTHY 🔴                               │
│  t=60s  : B REMOVED from registry ☠                           │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Multi-Datacenter Discovery

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-DATACENTER DISCOVERY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────┐         ┌─────────────────────┐        │
│  │   Dhaka DC (Primary) │         │  Singapore DC (DR)   │        │
│  │                     │  Sync   │                     │        │
│  │  ┌───────────────┐  │◄───────►│  ┌───────────────┐  │        │
│  │  │ Consul Server │  │         │  │ Consul Server │  │        │
│  │  │   (Leader)    │  │         │  │  (Follower)   │  │        │
│  │  └───────┬───────┘  │         │  └───────┬───────┘  │        │
│  │          │           │         │          │           │        │
│  │  ┌───────┴───────┐  │         │  ┌───────┴───────┐  │        │
│  │  │  Services:    │  │         │  │  Services:    │  │        │
│  │  │  - Payment    │  │         │  │  - Payment    │  │        │
│  │  │  - Ride       │  │         │  │  - Ride       │  │        │
│  │  │  - Notify     │  │         │  │  - Notify     │  │        │
│  │  └───────────────┘  │         │  └───────────────┘  │        │
│  └─────────────────────┘         └─────────────────────┘        │
│                                                                   │
│  Preference: Local DC first → Remote DC as fallback              │
└─────────────────────────────────────────────────────────────────┘
```

### DNS-Based Discovery

```
┌───────────────────────────────────────────────────────┐
│              DNS-BASED SERVICE DISCOVERY                │
├───────────────────────────────────────────────────────┤
│                                                        │
│  Client Query: payment.service.consul                  │
│                                                        │
│       ┌─────────┐  DNS Query  ┌──────────────┐        │
│       │ Client  │────────────►│  DNS Server  │        │
│       │         │◄────────────│  (Consul DNS)│        │
│       └────┬────┘  A Records  └──────────────┘        │
│            │                                           │
│            │  Returns:                                 │
│            │  payment.service.consul → 10.0.1.5        │
│            │  payment.service.consul → 10.0.1.6        │
│            │  payment.service.consul → 10.0.1.7        │
│            │                                           │
│            │  SRV Records (port সহ):                   │
│            │  _payment._tcp.service.consul              │
│            │    → 10.0.1.5:8080                        │
│            │    → 10.0.1.6:8081                        │
│            │    → 10.0.1.7:8082                        │
│            │                                           │
└───────────────────────────────────────────────────────┘
```

### Service Mesh Integration (Istio)

```
┌───────────────────────────────────────────────────────────────┐
│              SERVICE MESH (ISTIO) DISCOVERY                     │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────┐                               │
│  │         Pod A               │                               │
│  │  ┌─────────┐ ┌───────────┐ │     ┌───────────────┐        │
│  │  │ Ride    │ │  Envoy    │─┼────►│  Istio Pilot  │        │
│  │  │ Service │ │  Sidecar  │ │     │  (Control     │        │
│  │  │         │►│  Proxy    │ │     │   Plane)      │        │
│  │  └─────────┘ └─────┬─────┘ │     └───────┬───────┘        │
│  └─────────────────────┼───────┘             │                │
│                        │            Service Registry           │
│                        │            info pushed down           │
│                        ▼                     │                │
│  ┌─────────────────────────────┐             │                │
│  │         Pod B               │             │                │
│  │  ┌───────────┐ ┌─────────┐ │◄────────────┘                │
│  │  │  Envoy    │ │ Payment │ │                               │
│  │  │  Sidecar  │►│ Service │ │                               │
│  │  │  Proxy    │ │         │ │                               │
│  │  └───────────┘ └─────────┘ │                               │
│  └─────────────────────────────┘                               │
│                                                                │
│  সুবিধা: Application code-এ কোনো discovery logic নেই!         │
└───────────────────────────────────────────────────────────────┘
```

---

## 🖥️ PHP কোড উদাহরণ

### Service Registry Implementation

```php
<?php

/**
 * Service Discovery Pattern - PHP Implementation
 * Pathao-র মতো রাইড-শেয়ারিং সিস্টেমের জন্য
 */

// ===============================
// Service Registry (Consul-like)
// ===============================

class ServiceInstance
{
    public string $id;
    public string $serviceName;
    public string $host;
    public int $port;
    public string $status;
    public array $metadata;
    public int $lastHeartbeat;

    public function __construct(
        string $serviceName,
        string $host,
        int $port,
        array $metadata = []
    ) {
        $this->id = uniqid($serviceName . '_');
        $this->serviceName = $serviceName;
        $this->host = $host;
        $this->port = $port;
        $this->status = 'UP';
        $this->metadata = $metadata;
        $this->lastHeartbeat = time();
    }

    public function getUrl(): string
    {
        return "http://{$this->host}:{$this->port}";
    }

    public function isHealthy(int $timeoutSeconds = 30): bool
    {
        return (time() - $this->lastHeartbeat) < $timeoutSeconds;
    }
}

class ServiceRegistry
{
    private array $services = [];
    private int $heartbeatTimeout = 30; // সেকেন্ড
    private array $healthCheckCallbacks = [];

    /**
     * সার্ভিস রেজিস্ট্রেশন
     */
    public function register(ServiceInstance $instance): void
    {
        $serviceName = $instance->serviceName;

        if (!isset($this->services[$serviceName])) {
            $this->services[$serviceName] = [];
        }

        $this->services[$serviceName][$instance->id] = $instance;

        echo "✅ [{$instance->id}] registered: {$instance->getUrl()}\n";
    }

    /**
     * সার্ভিস ডিরেজিস্ট্রেশন
     */
    public function deregister(string $serviceId): void
    {
        foreach ($this->services as $serviceName => &$instances) {
            if (isset($instances[$serviceId])) {
                unset($instances[$serviceId]);
                echo "❌ [{$serviceId}] deregistered\n";
                return;
            }
        }
    }

    /**
     * Heartbeat গ্রহণ
     */
    public function heartbeat(string $serviceId): void
    {
        foreach ($this->services as &$instances) {
            if (isset($instances[$serviceId])) {
                $instances[$serviceId]->lastHeartbeat = time();
                $instances[$serviceId]->status = 'UP';
                return;
            }
        }
    }

    /**
     * সুস্থ সার্ভিস instance খুঁজে বের করা
     */
    public function discover(string $serviceName): array
    {
        if (!isset($this->services[$serviceName])) {
            return [];
        }

        // Stale entry গুলো সরিয়ে ফেলা
        $this->evictStaleEntries($serviceName);

        return array_values(
            array_filter(
                $this->services[$serviceName],
                fn($instance) => $instance->status === 'UP' && $instance->isHealthy($this->heartbeatTimeout)
            )
        );
    }

    /**
     * Stale entry eviction - মৃত সার্ভিস সরানো
     */
    private function evictStaleEntries(string $serviceName): void
    {
        if (!isset($this->services[$serviceName])) {
            return;
        }

        foreach ($this->services[$serviceName] as $id => $instance) {
            if (!$instance->isHealthy($this->heartbeatTimeout)) {
                $instance->status = 'DOWN';
                echo "⚠️ [{$id}] marked as DOWN (no heartbeat for {$this->heartbeatTimeout}s)\n";

                // ৬০ সেকেন্ড পর সম্পূর্ণ remove
                if ((time() - $instance->lastHeartbeat) > ($this->heartbeatTimeout * 2)) {
                    unset($this->services[$serviceName][$id]);
                    echo "☠️ [{$id}] evicted from registry\n";
                }
            }
        }
    }

    /**
     * Health check চালানো
     */
    public function runHealthChecks(): void
    {
        foreach ($this->services as $serviceName => $instances) {
            foreach ($instances as $instance) {
                $healthy = $this->performHealthCheck($instance);
                $instance->status = $healthy ? 'UP' : 'DOWN';
            }
        }
    }

    private function performHealthCheck(ServiceInstance $instance): bool
    {
        // HTTP health endpoint check
        $url = $instance->getUrl() . '/health';

        $ch = curl_init($url);
        curl_setopt($ch, CURLOPT_TIMEOUT, 5);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return $httpCode === 200;
    }

    /**
     * সকল registered সার্ভিসের তালিকা
     */
    public function getAllServices(): array
    {
        $result = [];
        foreach ($this->services as $name => $instances) {
            $result[$name] = count($instances);
        }
        return $result;
    }
}

// ===============================
// Client-Side Discovery (Load Balancer সহ)
// ===============================

interface LoadBalancerStrategy
{
    public function choose(array $instances): ?ServiceInstance;
}

class RoundRobinStrategy implements LoadBalancerStrategy
{
    private int $counter = 0;

    public function choose(array $instances): ?ServiceInstance
    {
        if (empty($instances)) {
            return null;
        }

        $instance = $instances[$this->counter % count($instances)];
        $this->counter++;
        return $instance;
    }
}

class WeightedStrategy implements LoadBalancerStrategy
{
    public function choose(array $instances): ?ServiceInstance
    {
        if (empty($instances)) {
            return null;
        }

        // Response time ভিত্তিক weight
        usort($instances, function ($a, $b) {
            $aWeight = $a->metadata['response_time'] ?? 100;
            $bWeight = $b->metadata['response_time'] ?? 100;
            return $aWeight - $bWeight;
        });

        return $instances[0];
    }
}

class ServiceDiscoveryClient
{
    private ServiceRegistry $registry;
    private LoadBalancerStrategy $loadBalancer;
    private array $cache = [];
    private int $cacheTTL = 10; // সেকেন্ড

    public function __construct(ServiceRegistry $registry, LoadBalancerStrategy $loadBalancer)
    {
        $this->registry = $registry;
        $this->loadBalancer = $loadBalancer;
    }

    /**
     * সার্ভিস discover করে একটি instance নির্বাচন করা
     */
    public function getServiceInstance(string $serviceName): ?ServiceInstance
    {
        // Cache check
        if ($this->isCacheValid($serviceName)) {
            $instances = $this->cache[$serviceName]['instances'];
        } else {
            $instances = $this->registry->discover($serviceName);
            $this->updateCache($serviceName, $instances);
        }

        if (empty($instances)) {
            throw new \RuntimeException("কোনো সুস্থ instance পাওয়া যায়নি: {$serviceName}");
        }

        return $this->loadBalancer->choose($instances);
    }

    private function isCacheValid(string $serviceName): bool
    {
        if (!isset($this->cache[$serviceName])) {
            return false;
        }

        return (time() - $this->cache[$serviceName]['timestamp']) < $this->cacheTTL;
    }

    private function updateCache(string $serviceName, array $instances): void
    {
        $this->cache[$serviceName] = [
            'instances' => $instances,
            'timestamp' => time()
        ];
    }
}

// ===============================
// Self-Registration Pattern
// ===============================

class SelfRegisteringService
{
    private ServiceRegistry $registry;
    private ServiceInstance $instance;
    private bool $running = false;

    public function __construct(ServiceRegistry $registry, string $serviceName, string $host, int $port)
    {
        $this->registry = $registry;
        $this->instance = new ServiceInstance($serviceName, $host, $port);
    }

    /**
     * সার্ভিস শুরু - নিজেকে register করা
     */
    public function start(): void
    {
        // Registry-তে নিবন্ধন
        $this->registry->register($this->instance);
        $this->running = true;

        // Heartbeat শুরু (প্রতি 10 সেকেন্ডে)
        $this->startHeartbeat();

        echo "🚀 Service started and registered: {$this->instance->getUrl()}\n";
    }

    /**
     * সার্ভিস বন্ধ - নিজেকে deregister করা
     */
    public function stop(): void
    {
        $this->running = false;
        $this->registry->deregister($this->instance->id);
        echo "🛑 Service stopped and deregistered\n";
    }

    private function startHeartbeat(): void
    {
        // বাস্তবে এটি একটি background timer হবে
        // এখানে simulation দেখানো হচ্ছে
        while ($this->running) {
            $this->registry->heartbeat($this->instance->id);
            sleep(10);
        }
    }
}

// ===============================
// ব্যবহারের উদাহরণ - Pathao System
// ===============================

echo "=== Pathao Service Discovery Demo ===\n\n";

// Registry তৈরি
$registry = new ServiceRegistry();

// Payment Service-এর instances register করা
$payment1 = new ServiceInstance('payment-service', '10.0.1.5', 8080, ['region' => 'dhaka']);
$payment2 = new ServiceInstance('payment-service', '10.0.1.6', 8080, ['region' => 'dhaka']);
$payment3 = new ServiceInstance('payment-service', '10.0.1.7', 8080, ['region' => 'chittagong']);

$registry->register($payment1);
$registry->register($payment2);
$registry->register($payment3);

// Notification Service register
$notify1 = new ServiceInstance('notification-service', '10.0.2.1', 9090);
$registry->register($notify1);

// Client-Side Discovery
$client = new ServiceDiscoveryClient($registry, new RoundRobinStrategy());

// Ride Matching Service Payment Service খুঁজছে
try {
    $paymentInstance = $client->getServiceInstance('payment-service');
    echo "\n🏍️ Ride Service → Payment Service found at: {$paymentInstance->getUrl()}\n";

    $notifyInstance = $client->getServiceInstance('notification-service');
    echo "🏍️ Ride Service → Notification Service found at: {$notifyInstance->getUrl()}\n";
} catch (\RuntimeException $e) {
    echo "❌ Error: " . $e->getMessage() . "\n";
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Service Registry with Health Checks

```javascript
/**
 * Service Discovery Pattern - JavaScript/Node.js Implementation
 * Pathao-র মাইক্রোসার্ভিস সিস্টেমের জন্য
 */

// ===============================
// Service Instance
// ===============================

class ServiceInstance {
    constructor(serviceName, host, port, metadata = {}) {
        this.id = `${serviceName}_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
        this.serviceName = serviceName;
        this.host = host;
        this.port = port;
        this.status = 'UP';
        this.metadata = metadata;
        this.lastHeartbeat = Date.now();
        this.registeredAt = Date.now();
    }

    getUrl() {
        return `http://${this.host}:${this.port}`;
    }

    isHealthy(timeoutMs = 30000) {
        return (Date.now() - this.lastHeartbeat) < timeoutMs;
    }
}

// ===============================
// Service Registry (Consul-like)
// ===============================

class ServiceRegistry {
    constructor(options = {}) {
        this.services = new Map();
        this.heartbeatTimeout = options.heartbeatTimeout || 30000; // 30s
        this.evictionTimeout = options.evictionTimeout || 60000;   // 60s
        this.healthCheckInterval = options.healthCheckInterval || 10000; // 10s

        // নিয়মিত health check চালানো
        this.healthCheckTimer = setInterval(
            () => this.runHealthChecks(),
            this.healthCheckInterval
        );
    }

    /**
     * সার্ভিস রেজিস্ট্রেশন
     */
    register(instance) {
        if (!this.services.has(instance.serviceName)) {
            this.services.set(instance.serviceName, new Map());
        }

        this.services.get(instance.serviceName).set(instance.id, instance);
        console.log(`✅ [${instance.serviceName}] registered: ${instance.getUrl()}`);

        return instance.id;
    }

    /**
     * সার্ভিস ডিরেজিস্ট্রেশন
     */
    deregister(serviceId) {
        for (const [serviceName, instances] of this.services) {
            if (instances.has(serviceId)) {
                instances.delete(serviceId);
                console.log(`❌ [${serviceId}] deregistered from ${serviceName}`);
                return true;
            }
        }
        return false;
    }

    /**
     * Heartbeat গ্রহণ করা
     */
    heartbeat(serviceId) {
        for (const [, instances] of this.services) {
            if (instances.has(serviceId)) {
                const instance = instances.get(serviceId);
                instance.lastHeartbeat = Date.now();
                instance.status = 'UP';
                return true;
            }
        }
        return false;
    }

    /**
     * সার্ভিস Discovery - সুস্থ instances ফেরত দেয়
     */
    discover(serviceName) {
        if (!this.services.has(serviceName)) {
            return [];
        }

        const instances = this.services.get(serviceName);
        const healthyInstances = [];

        for (const [id, instance] of instances) {
            if (instance.status === 'UP' && instance.isHealthy(this.heartbeatTimeout)) {
                healthyInstances.push(instance);
            }
        }

        return healthyInstances;
    }

    /**
     * Health Check ও Stale Entry Eviction
     */
    runHealthChecks() {
        const now = Date.now();

        for (const [serviceName, instances] of this.services) {
            for (const [id, instance] of instances) {
                const timeSinceHeartbeat = now - instance.lastHeartbeat;

                if (timeSinceHeartbeat > this.evictionTimeout) {
                    // সম্পূর্ণ evict করা
                    instances.delete(id);
                    console.log(`☠️ [${id}] evicted (no heartbeat for ${this.evictionTimeout}ms)`);
                } else if (timeSinceHeartbeat > this.heartbeatTimeout) {
                    // Unhealthy হিসেবে চিহ্নিত করা
                    instance.status = 'DOWN';
                    console.log(`⚠️ [${id}] marked DOWN (heartbeat timeout)`);
                }
            }
        }
    }

    /**
     * সকল সার্ভিসের সারাংশ
     */
    getSummary() {
        const summary = {};
        for (const [name, instances] of this.services) {
            const healthy = [...instances.values()].filter(i => i.status === 'UP').length;
            summary[name] = { total: instances.size, healthy };
        }
        return summary;
    }

    destroy() {
        clearInterval(this.healthCheckTimer);
    }
}

// ===============================
// Client-Side Discovery with Load Balancing
// ===============================

class LoadBalancer {
    static roundRobin(instances) {
        if (!this._counters) this._counters = {};
        const key = instances[0]?.serviceName || 'default';
        if (!this._counters[key]) this._counters[key] = 0;

        const instance = instances[this._counters[key] % instances.length];
        this._counters[key]++;
        return instance;
    }

    static random(instances) {
        return instances[Math.floor(Math.random() * instances.length)];
    }

    static leastConnections(instances) {
        return instances.reduce((min, inst) =>
            (inst.metadata.activeConnections || 0) < (min.metadata.activeConnections || 0) ? inst : min
        );
    }
}

class DiscoveryClient {
    constructor(registry, options = {}) {
        this.registry = registry;
        this.strategy = options.strategy || 'roundRobin';
        this.cache = new Map();
        this.cacheTTL = options.cacheTTL || 10000; // 10s
        this.retryCount = options.retryCount || 3;
    }

    /**
     * সার্ভিস খুঁজে instance নির্বাচন
     */
    async getServiceInstance(serviceName) {
        let instances;

        // Cache থেকে চেষ্টা
        const cached = this.cache.get(serviceName);
        if (cached && (Date.now() - cached.timestamp) < this.cacheTTL) {
            instances = cached.instances;
        } else {
            instances = this.registry.discover(serviceName);
            this.cache.set(serviceName, { instances, timestamp: Date.now() });
        }

        if (instances.length === 0) {
            throw new Error(`কোনো সুস্থ instance পাওয়া যায়নি: ${serviceName}`);
        }

        return LoadBalancer[this.strategy](instances);
    }

    /**
     * সার্ভিসে Request পাঠানো (retry সহ)
     */
    async callService(serviceName, path, options = {}) {
        let lastError;

        for (let attempt = 1; attempt <= this.retryCount; attempt++) {
            try {
                const instance = await this.getServiceInstance(serviceName);
                const url = `${instance.getUrl()}${path}`;

                console.log(`📡 [Attempt ${attempt}] Calling ${serviceName} at ${url}`);

                const response = await fetch(url, {
                    ...options,
                    signal: AbortSignal.timeout(options.timeout || 5000)
                });

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }

                return await response.json();
            } catch (error) {
                lastError = error;
                console.log(`⚠️ [Attempt ${attempt}] Failed: ${error.message}`);

                // Cache invalidate করা - পরবর্তী attempt-এ fresh data পাবে
                this.cache.delete(serviceName);
            }
        }

        throw new Error(`${serviceName} সার্ভিসে ${this.retryCount} বার চেষ্টার পরও ব্যর্থ: ${lastError.message}`);
    }
}

// ===============================
// Self-Registering Service
// ===============================

class MicroService {
    constructor(registry, name, host, port) {
        this.registry = registry;
        this.instance = new ServiceInstance(name, host, port);
        this.heartbeatTimer = null;
    }

    /**
     * সার্ভিস শুরু ও নিবন্ধন
     */
    async start() {
        // Registry-তে register
        this.registry.register(this.instance);

        // Heartbeat শুরু (প্রতি 10 সেকেন্ডে)
        this.heartbeatTimer = setInterval(() => {
            this.registry.heartbeat(this.instance.id);
        }, 10000);

        console.log(`🚀 ${this.instance.serviceName} started at ${this.instance.getUrl()}`);
    }

    /**
     * Graceful shutdown
     */
    async stop() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
        }

        this.registry.deregister(this.instance.id);
        console.log(`🛑 ${this.instance.serviceName} stopped`);
    }
}

// ===============================
// DNS-Based Discovery Simulation
// ===============================

class DNSServiceDiscovery {
    constructor() {
        this.records = new Map(); // service name → [addresses]
    }

    /**
     * SRV record যোগ করা
     */
    addSRVRecord(serviceName, host, port, priority = 10, weight = 100) {
        if (!this.records.has(serviceName)) {
            this.records.set(serviceName, []);
        }
        this.records.get(serviceName).push({ host, port, priority, weight });
    }

    /**
     * DNS query simulation
     */
    resolve(serviceName) {
        const records = this.records.get(serviceName) || [];

        // Priority অনুযায়ী sort, তারপর weighted random selection
        return records
            .sort((a, b) => a.priority - b.priority)
            .map(r => ({ host: r.host, port: r.port }));
    }
}

// ===============================
// ব্যবহারের উদাহরণ - Pathao System
// ===============================

async function pathaoServiceDiscoveryDemo() {
    console.log('=== 🏍️ Pathao Service Discovery Demo ===\n');

    // Service Registry তৈরি
    const registry = new ServiceRegistry({
        heartbeatTimeout: 30000,
        evictionTimeout: 60000,
        healthCheckInterval: 10000
    });

    // Payment Service instances
    const paymentInstances = [
        new ServiceInstance('payment-service', '10.0.1.5', 8080, { region: 'dhaka', provider: 'bKash' }),
        new ServiceInstance('payment-service', '10.0.1.6', 8080, { region: 'dhaka', provider: 'nagad' }),
        new ServiceInstance('payment-service', '10.0.1.7', 8080, { region: 'chittagong', provider: 'bKash' })
    ];

    // Notification Service instances
    const notifyInstances = [
        new ServiceInstance('notification-service', '10.0.2.1', 9090, { type: 'push' }),
        new ServiceInstance('notification-service', '10.0.2.2', 9090, { type: 'sms' })
    ];

    // Location Service
    const locationInstances = [
        new ServiceInstance('location-service', '10.0.3.1', 7070, { datacenter: 'dhaka-1' })
    ];

    // সকল সার্ভিস register করা
    [...paymentInstances, ...notifyInstances, ...locationInstances]
        .forEach(inst => registry.register(inst));

    console.log('\n📊 Registry Summary:', registry.getSummary());

    // Discovery Client তৈরি
    const discoveryClient = new DiscoveryClient(registry, {
        strategy: 'roundRobin',
        cacheTTL: 5000,
        retryCount: 3
    });

    // Ride Matching Service অন্য সার্ভিসগুলো খুঁজছে
    console.log('\n--- Ride Matching Service discovering other services ---\n');

    try {
        const payment = await discoveryClient.getServiceInstance('payment-service');
        console.log(`💰 Payment Service found: ${payment.getUrl()} (${payment.metadata.provider})`);

        const notify = await discoveryClient.getServiceInstance('notification-service');
        console.log(`📱 Notification Service found: ${notify.getUrl()} (${notify.metadata.type})`);

        const location = await discoveryClient.getServiceInstance('location-service');
        console.log(`📍 Location Service found: ${location.getUrl()}`);
    } catch (error) {
        console.error(`❌ Discovery failed: ${error.message}`);
    }

    // Simulate service failure
    console.log('\n--- Simulating payment-service-1 failure ---\n');
    paymentInstances[0].lastHeartbeat = Date.now() - 35000; // 35s আগের heartbeat
    registry.runHealthChecks();

    console.log('\n📊 Registry Summary after failure:', registry.getSummary());

    // Cleanup
    registry.destroy();
}

pathaoServiceDiscoveryDemo();
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

| পরিস্থিতি | কারণ |
|-----------|------|
| মাইক্রোসার্ভিস আর্কিটেকচারে | সার্ভিসের dynamic IP/port পরিবর্তন হয় |
| Container/Kubernetes environment | Pod-গুলো ephemeral, IP পরিবর্তন হয় |
| Auto-scaling ব্যবহার করলে | নতুন instance আসে-যায় |
| Multi-region deployment | বিভিন্ন datacenter-এ সার্ভিস আছে |
| Blue-Green / Canary deployment | নতুন version-এ traffic redirect করতে |
| Cloud-native applications | Elastic infrastructure-তে |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|------|
| Monolithic application | একটিই সার্ভিস, discover করার দরকার নেই |
| Static infrastructure | IP/port কখনো পরিবর্তন হয় না |
| ২-৩টি সার্ভিস | Configuration file-ই যথেষ্ট |
| Development environment | Docker Compose-এ service name-ই কাজ করে |
| সীমিত budget/team | Registry maintain করা overhead |

### 🔄 Client-Side vs Server-Side তুলনা:

```
┌────────────────────┬─────────────────────┬─────────────────────┐
│      বৈশিষ্ট্য     │   Client-Side       │    Server-Side      │
├────────────────────┼─────────────────────┼─────────────────────┤
│ Load Balancer      │ Client-এ থাকে       │ Infra-তে থাকে      │
│ Complexity         │ Client complex       │ Infra complex       │
│ Performance        │ Fewer hops           │ Extra hop (LB)      │
│ Language coupling  │ Library per language │ Language agnostic   │
│ উদাহরণ            │ Netflix Eureka       │ AWS ALB, K8s Service│
│ Bangladesh উদাহরণ │ Pathao app internal  │ Grameenphone API GW │
└────────────────────┴─────────────────────┴─────────────────────┘
```

### 🛠️ Registry তুলনা:

```
┌──────────────┬──────────────┬───────────┬──────────────┐
│   Feature    │    Consul    │   etcd    │  ZooKeeper   │
├──────────────┼──────────────┼───────────┼──────────────┤
│ Protocol     │ HTTP/DNS     │ gRPC      │ TCP          │
│ Health Check │ Built-in     │ External  │ Session      │
│ Multi-DC     │ ✅ Native    │ ❌        │ ❌           │
│ UI           │ ✅ Built-in  │ ❌        │ ❌           │
│ Consistency  │ CP (Raft)    │ CP (Raft) │ CP (ZAB)    │
│ Use Case     │ Service Mesh │ K8s       │ Hadoop      │
└──────────────┴──────────────┴───────────┴──────────────┘
```

### 💡 সেরা অনুশীলন (Best Practices):

1. **সর্বদা Health Check ব্যবহার করুন** - শুধু registration যথেষ্ট নয়
2. **Heartbeat timeout সঠিকভাবে সেট করুন** - খুব কম হলে false positive, বেশি হলে stale entry
3. **Client-side caching করুন** - প্রতিটি request-এ registry query না করে cache ব্যবহার
4. **Graceful shutdown implement করুন** - বন্ধ হওয়ার আগে deregister করা
5. **Circuit breaker সাথে ব্যবহার করুন** - unhealthy instance-এ repeated call এড়ান
6. **Multi-datacenter support** - local DC-কে preference দিন, remote-কে fallback হিসেবে ব্যবহার করুন

---

## 🔗 সম্পর্কিত প্যাটার্ন

- **Load Balancer Pattern** - Discovery-র পর traffic distribution
- **Circuit Breaker** - unhealthy service-এ call বন্ধ করা
- **API Gateway** - Server-side discovery-র সাথে ব্যবহার
- **Health Check Pattern** - সার্ভিস সচল কিনা যাচাই
- **Service Mesh** - Infrastructure-level discovery (Istio, Linkerd)
