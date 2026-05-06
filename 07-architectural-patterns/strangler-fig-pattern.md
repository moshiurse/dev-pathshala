# 🌳 Strangler Fig Pattern (স্ট্র্যাংলার ফিগ প্যাটার্ন)

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [Migration Phases](#migration-phases)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Anti-Corruption Layer](#anti-corruption-layer)
- [Rollback Strategies](#rollback-strategies)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Strangler Fig Pattern** হলো একটি legacy system modernization কৌশল যেখানে পুরানো সিস্টেমকে ধীরে ধীরে, feature-by-feature নতুন সিস্টেম দিয়ে প্রতিস্থাপন করা হয়। এটি অস্ট্রেলিয়ার **Strangler Fig গাছ** থেকে নামকরণ করা হয়েছে, যা অন্য গাছকে ধীরে ধীরে ঘিরে ফেলে এবং শেষে পুরানো গাছটি মারা যায়।

### 🌱 Strangler Fig গাছের সাদৃশ্য:
```
   পর্যায় ১:              পর্যায় ২:              পর্যায় ৩:
   বীজ রোপণ             বৃদ্ধি                 সম্পূর্ণ আবরণ
   
   ┌───────┐            ╔═══════╗            ╔═══════════╗
   │Legacy │            ║New ┌──║──┐         ║           ║
   │System │            ║    │Le║ga│         ║    New    ║
   │       │            ║    │cy║  │         ║  System   ║
   │       │            ║    └──║──┘         ║           ║
   └───────┘            ╚═══════╝            ╚═══════════╝
   
   পুরানো সিস্টেম        নতুন সিস্টেম           পুরানো সিস্টেম
   সম্পূর্ণ কাজ করছে     ধীরে ধীরে ঘিরে       সম্পূর্ণ প্রতিস্থাপিত
                         ফেলছে
```

### 🔑 মূল ধারণাসমূহ:
- **Facade/Proxy Layer**: ট্র্যাফিক রাউটিং-এর জন্য মধ্যবর্তী স্তর
- **Feature-by-Feature Migration**: একবারে একটি feature migrate করা
- **Coexistence Period**: পুরানো ও নতুন সিস্টেম একসাথে চলা
- **Anti-Corruption Layer**: নতুন ও পুরানো সিস্টেমের মধ্যে অনুবাদ স্তর
- **Incremental Rollout**: ধাপে ধাপে নতুন সিস্টেমে ট্র্যাফিক পাঠানো

> 💡 মূলমন্ত্র: "Big Bang rewrite" এর বদলে ক্রমান্বয়ে modernization — ঝুঁকি কম, ব্যবসা চলমান থাকে।

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏢 Scenario: একটি বাংলাদেশী ই-কমার্স কোম্পানির Legacy Migration

ধরুন **"বাংলা শপ"** একটি পুরানো PHP monolithic অ্যাপ্লিকেশন চালাচ্ছে যেটি ১০ বছর ধরে চলছে:

**পুরানো সিস্টেম (Legacy Monolith):**
- PHP 5.6, CodeIgniter framework
- একটি বিশাল MySQL database
- সব feature একসাথে: পণ্য ব্যবস্থাপনা, অর্ডার, পেমেন্ট, ডেলিভারি, রিপোর্টিং
- প্রতিদিন ৫০,০০০+ অর্ডার পরিচালনা করে

**সমস্যা:**
- নতুন feature যোগ করতে মাসখানেক লাগে
- একটি bug fix করলে অন্য জায়গায় সমস্যা হয়
- Scaling কঠিন — Eid/Sale-এর সময় সিস্টেম ক্র্যাশ করে
- নতুন developer hire করা কঠিন (কেউ পুরানো stack-এ কাজ করতে চায় না)

**সমাধান: Strangler Fig Pattern ব্যবহার করে ধাপে ধাপে Microservices-এ Migration**

### 📱 Pathao-র উদাহরণ:
ধরুন Pathao তাদের পুরানো monolithic ride-sharing system modernize করতে চায়:

```
Migration Plan:
Phase 1: Payment Service আলাদা করা (bKash/Nagad integration)
Phase 2: Rider Matching Service আলাদা করা
Phase 3: Location/Tracking Service আলাদা করা
Phase 4: Notification Service আলাদা করা
Phase 5: Legacy monolith বন্ধ করা
```

### 📰 Prothom Alo Digital Transformation:
```
Legacy: WordPress-based monolith
Target: Microservices (Content, User, Ad, Analytics)

Phase 1: Content API আলাদা করা → নতুন Mobile App এটা ব্যবহার করবে
Phase 2: User/Subscription service আলাদা করা
Phase 3: Ad management system আলাদা করা
Phase 4: Analytics service আলাদা করা
Phase 5: পুরানো WordPress বন্ধ করা
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### সম্পূর্ণ Strangler Fig Architecture:
```
┌────────────────────────────────────────────────────────────────┐
│                    STRANGLER FIG ARCHITECTURE                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐                                                 │
│   │  Client  │ (Mobile App / Web Browser)                      │
│   └────┬─────┘                                                 │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────────────────────────────┐                  │
│   │         FACADE / API GATEWAY            │                  │
│   │   (Routing Layer - nginx/Kong/Custom)   │                  │
│   │                                         │                  │
│   │  /products/*  → New Product Service     │                  │
│   │  /orders/*    → New Order Service       │                  │
│   │  /payments/*  → New Payment Service     │                  │
│   │  /reports/*   → Legacy System (এখনো)    │                  │
│   │  /admin/*     → Legacy System (এখনো)    │                  │
│   └───────┬──────────────┬──────────────────┘                  │
│           │              │                                      │
│     ┌─────┴──────┐  ┌───┴────────────────────────┐            │
│     │            │  │                             │            │
│     ▼            ▼  ▼                             ▼            │
│  ┌────────┐ ┌────────┐ ┌────────┐    ┌────────────────────┐  │
│  │Product │ │ Order  │ │Payment │    │                    │  │
│  │Service │ │Service │ │Service │    │   LEGACY SYSTEM    │  │
│  │(Node.js)│ │(Go)   │ │(Java)  │    │   (PHP Monolith)  │  │
│  └───┬────┘ └───┬────┘ └───┬────┘    │                    │  │
│      │          │          │         │  - Reports         │  │
│      ▼          ▼          ▼         │  - Admin Panel     │  │
│  ┌───────┐ ┌───────┐ ┌───────┐     │  - Old Features    │  │
│  │MongoDB│ │ MySQL │ │ Redis │     │                    │  │
│  └───────┘ └───────┘ └───────┘     └────────┬───────────┘  │
│                                              │               │
│                    ┌─────────────────────┐   │               │
│                    │ ANTI-CORRUPTION     │◄──┘               │
│                    │ LAYER               │                    │
│                    │ (Data Translation)  │                    │
│                    └─────────────────────┘                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Migration Timeline:
```
সময়রেখা ──────────────────────────────────────────────────▶

Phase 1 (মাস ১-৩):        Phase 2 (মাস ৪-৬):       Phase 3 (মাস ৭-৯):
┌────────────────────┐    ┌────────────────────┐    ┌────────────────────┐
│████████████████████│    │████████████████░░░░│    │████████░░░░░░░░░░░░│
│█ Legacy (100%)    █│    │█ Legacy (80%)  ░░░░│    │█Legacy░░░░░░░░░░░░░│
│████████████████████│    │████████████████░░░░│    │█(40%)░░░░░░░░░░░░░░│
│                    │    │░░░░ New (20%) ░░░░░│    │░░░░░░ New (60%) ░░░│
└────────────────────┘    └────────────────────┘    └────────────────────┘

Phase 4 (মাস ১০-১২):     Phase 5 (মাস ১৩+):
┌────────────────────┐    ┌────────────────────┐
│███░░░░░░░░░░░░░░░░░│    │░░░░░░░░░░░░░░░░░░░░│
│█L░░░░░░░░░░░░░░░░░░│    │░░░░░░░░░░░░░░░░░░░░│
│█░░░░░░░░░░░░░░░░░░░│    │░░░ New (100%) ░░░░░│
│░░░░ New (85%) ░░░░░│    │░░░░░░░░░░░░░░░░░░░░│
└────────────────────┘    └────────────────────┘

█ = Legacy System          ░ = New Microservices
```

### Routing Strategy:
```
                    ┌─────────────┐
   Request ────────▶│   Router    │
                    │  Decision   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐  ┌───▼───┐  ┌────▼────┐
         │ Feature │  │Feature│  │ Feature │
         │   Flag  │  │ Path  │  │ Header  │
         │ Based   │  │ Based │  │ Based   │
         └────┬────┘  └───┬───┘  └────┬────┘
              │            │            │
              ▼            ▼            ▼
    ┌──────────────┐ ┌─────────┐ ┌──────────┐
    │ if flag=true │ │ /api/v2 │ │ X-Version│
    │  → new       │ │ → new   │ │ : 2      │
    │ else → old   │ │ /api/v1 │ │ → new    │
    └──────────────┘ │ → old   │ └──────────┘
                     └─────────┘
```

---

## 🔄 Migration Phases

### Phase 1: Facade/Proxy স্থাপন
```
Before:                          After Phase 1:
                                 
Client ──▶ Legacy               Client ──▶ Proxy ──▶ Legacy
                                            │
                                            └──▶ (Ready for routing)
```

### Phase 2: প্রথম Service বের করা
```
Client ──▶ Proxy ──┬──▶ New Product Service
                   │
                   └──▶ Legacy (বাকি সব)
```

### Phase 3: আরো Services
```
Client ──▶ Proxy ──┬──▶ Product Service
                   ├──▶ Order Service
                   ├──▶ Payment Service
                   │
                   └──▶ Legacy (Reports, Admin)
```

### Phase 4: Legacy Decommission
```
Client ──▶ Proxy ──┬──▶ Product Service
                   ├──▶ Order Service
                   ├──▶ Payment Service
                   ├──▶ Report Service
                   └──▶ Admin Service
                   
                   Legacy: ❌ বন্ধ
```

---

## 💻 PHP কোড উদাহরণ

### Strangler Facade (API Gateway/Proxy):

```php
<?php

/**
 * Strangler Facade - Request Router
 * এই class legacy ও নতুন services-এর মধ্যে routing পরিচালনা করে
 */
class StranglerFacade
{
    private array $routingRules;
    private LegacyProxy $legacyProxy;
    private ServiceRegistry $serviceRegistry;
    private FeatureFlagService $featureFlags;

    public function __construct(
        LegacyProxy $legacyProxy,
        ServiceRegistry $serviceRegistry,
        FeatureFlagService $featureFlags
    ) {
        $this->legacyProxy = $legacyProxy;
        $this->serviceRegistry = $serviceRegistry;
        $this->featureFlags = $featureFlags;
        $this->loadRoutingRules();
    }

    private function loadRoutingRules(): void
    {
        $this->routingRules = [
            '/api/products' => [
                'service' => 'product-service',
                'feature_flag' => 'new_product_service',
                'fallback' => 'legacy',
            ],
            '/api/orders' => [
                'service' => 'order-service',
                'feature_flag' => 'new_order_service',
                'fallback' => 'legacy',
            ],
            '/api/payments' => [
                'service' => 'payment-service',
                'feature_flag' => 'new_payment_service',
                'fallback' => 'legacy',
            ],
            // এখনো migrate হয়নি - সরাসরি legacy-তে যাবে
            '/api/reports' => [
                'service' => 'legacy',
                'feature_flag' => null,
                'fallback' => 'legacy',
            ],
        ];
    }

    public function handleRequest(Request $request): Response
    {
        $path = $request->getPath();
        $rule = $this->findMatchingRule($path);

        if (!$rule) {
            return $this->legacyProxy->forward($request);
        }

        // Feature flag check - ধাপে ধাপে rollout
        if ($rule['feature_flag'] && !$this->featureFlags->isEnabled($rule['feature_flag'], $request)) {
            return $this->legacyProxy->forward($request);
        }

        try {
            $service = $this->serviceRegistry->getService($rule['service']);
            $response = $service->handle($request);

            // Response validation
            if ($this->isValidResponse($response)) {
                return $response;
            }

            // Invalid response - fallback to legacy
            $this->logMigrationIssue($path, 'invalid_response', $response);
            return $this->legacyProxy->forward($request);

        } catch (\Exception $e) {
            // নতুন service fail করলে legacy-তে fallback
            $this->logMigrationIssue($path, 'service_error', $e->getMessage());
            return $this->legacyProxy->forward($request);
        }
    }

    private function findMatchingRule(string $path): ?array
    {
        foreach ($this->routingRules as $pattern => $rule) {
            if (str_starts_with($path, $pattern)) {
                return $rule;
            }
        }
        return null;
    }

    private function isValidResponse(Response $response): bool
    {
        return $response->getStatusCode() < 500;
    }

    private function logMigrationIssue(string $path, string $type, $details): void
    {
        // Migration issues log করা - পরে analyze করার জন্য
        error_log(json_encode([
            'type' => 'migration_fallback',
            'path' => $path,
            'issue_type' => $type,
            'details' => $details,
            'timestamp' => date('c'),
        ]));
    }
}

/**
 * Feature Flag Service - ধাপে ধাপে নতুন service-এ traffic পাঠানো
 */
class FeatureFlagService
{
    private array $flags;

    public function __construct(private readonly Redis $redis)
    {
        $this->loadFlags();
    }

    public function isEnabled(string $flagName, Request $request): bool
    {
        $flag = $this->flags[$flagName] ?? null;
        if (!$flag) return false;

        return match ($flag['strategy']) {
            'percentage' => $this->percentageRollout($flag, $request),
            'user_list' => $this->userListCheck($flag, $request),
            'header' => $this->headerCheck($flag, $request),
            'all' => true,
            default => false,
        };
    }

    private function percentageRollout(array $flag, Request $request): bool
    {
        $userId = $request->getUserId() ?? $request->getIp();
        $hash = crc32($userId . $flag['name']);
        $percentage = abs($hash) % 100;
        return $percentage < $flag['rollout_percentage'];
    }

    private function userListCheck(array $flag, Request $request): bool
    {
        $userId = $request->getUserId();
        return in_array($userId, $flag['allowed_users'] ?? []);
    }

    private function headerCheck(array $flag, Request $request): bool
    {
        return $request->getHeader('X-Use-New-Service') === 'true';
    }

    private function loadFlags(): void
    {
        $this->flags = [
            'new_product_service' => [
                'name' => 'new_product_service',
                'strategy' => 'percentage',
                'rollout_percentage' => 50, // ৫০% ট্র্যাফিক নতুন service-এ
            ],
            'new_order_service' => [
                'name' => 'new_order_service',
                'strategy' => 'percentage',
                'rollout_percentage' => 25, // ২৫% ট্র্যাফিক নতুন service-এ
            ],
            'new_payment_service' => [
                'name' => 'new_payment_service',
                'strategy' => 'user_list',
                'allowed_users' => ['internal_test_1', 'internal_test_2'],
            ],
        ];
    }
}

/**
 * Anti-Corruption Layer
 * Legacy ও নতুন system-এর data format-এর মধ্যে অনুবাদ করে
 */
class AntiCorruptionLayer
{
    // Legacy format → New service format
    public function translateOrderToNew(array $legacyOrder): array
    {
        return [
            'id' => $legacyOrder['order_id'],
            'customer' => [
                'id' => $legacyOrder['cust_id'],
                'name' => $legacyOrder['cust_name'],
                'phone' => $legacyOrder['cust_phone'],
            ],
            'items' => array_map(fn($item) => [
                'productId' => $item['prod_id'],
                'name' => $item['prod_name'],
                'quantity' => (int) $item['qty'],
                'unitPrice' => (float) $item['price'],
            ], $legacyOrder['items'] ?? []),
            'total' => (float) $legacyOrder['total_amount'],
            'status' => $this->mapOrderStatus($legacyOrder['status_code']),
            'createdAt' => date('c', strtotime($legacyOrder['created_date'])),
        ];
    }

    // New format → Legacy format (backward compatibility)
    public function translateOrderToLegacy(array $newOrder): array
    {
        return [
            'order_id' => $newOrder['id'],
            'cust_id' => $newOrder['customer']['id'],
            'cust_name' => $newOrder['customer']['name'],
            'cust_phone' => $newOrder['customer']['phone'],
            'total_amount' => $newOrder['total'],
            'status_code' => $this->reverseMapStatus($newOrder['status']),
            'created_date' => date('Y-m-d H:i:s', strtotime($newOrder['createdAt'])),
        ];
    }

    private function mapOrderStatus(int $code): string
    {
        return match ($code) {
            0 => 'pending',
            1 => 'confirmed',
            2 => 'processing',
            3 => 'shipped',
            4 => 'delivered',
            5 => 'cancelled',
            default => 'unknown',
        };
    }

    private function reverseMapStatus(string $status): int
    {
        return match ($status) {
            'pending' => 0,
            'confirmed' => 1,
            'processing' => 2,
            'shipped' => 3,
            'delivered' => 4,
            'cancelled' => 5,
            default => -1,
        };
    }
}

/**
 * Legacy Proxy - পুরানো system-এ request forward করে
 */
class LegacyProxy
{
    private string $legacyBaseUrl;

    public function __construct(string $legacyBaseUrl)
    {
        $this->legacyBaseUrl = $legacyBaseUrl;
    }

    public function forward(Request $request): Response
    {
        $url = $this->legacyBaseUrl . $request->getPath();

        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $request->getMethod(),
            CURLOPT_HTTPHEADER => $request->getHeaders(),
            CURLOPT_POSTFIELDS => $request->getBody(),
            CURLOPT_TIMEOUT => 30,
        ]);

        $body = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return new Response($statusCode, $body);
    }
}

/**
 * Data Sync Service
 * Migration চলাকালীন legacy ও new system-এর মধ্যে data sync রাখে
 */
class DataSyncService
{
    public function __construct(
        private readonly PDO $legacyDb,
        private readonly HttpClient $newServiceClient,
        private readonly AntiCorruptionLayer $acl
    ) {}

    // Legacy-তে নতুন order তৈরি হলে new service-এও sync করা
    public function syncNewOrder(int $legacyOrderId): void
    {
        $stmt = $this->legacyDb->prepare(
            "SELECT * FROM orders WHERE order_id = ?"
        );
        $stmt->execute([$legacyOrderId]);
        $legacyOrder = $stmt->fetch(PDO::FETCH_ASSOC);

        $newFormat = $this->acl->translateOrderToNew($legacyOrder);

        $this->newServiceClient->post('/api/orders/sync', $newFormat);
    }

    // নতুন service-এ order update হলে legacy-তেও sync করা
    public function syncOrderUpdate(array $newOrder): void
    {
        $legacyFormat = $this->acl->translateOrderToLegacy($newOrder);

        $stmt = $this->legacyDb->prepare(
            "UPDATE orders SET status_code = ?, total_amount = ? WHERE order_id = ?"
        );
        $stmt->execute([
            $legacyFormat['status_code'],
            $legacyFormat['total_amount'],
            $legacyFormat['order_id'],
        ]);
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### API Gateway with Express.js:

```javascript
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const Redis = require('ioredis');

const app = express();
const redis = new Redis();

// ===== Feature Flag Manager =====
class FeatureFlagManager {
    constructor(redis) {
        this.redis = redis;
        this.flags = new Map();
        this.loadFlags();
    }

    async loadFlags() {
        const flagsData = await this.redis.get('strangler:feature_flags');
        if (flagsData) {
            const flags = JSON.parse(flagsData);
            flags.forEach(flag => this.flags.set(flag.name, flag));
        }
    }

    isEnabled(flagName, request) {
        const flag = this.flags.get(flagName);
        if (!flag || !flag.enabled) return false;

        switch (flag.strategy) {
            case 'percentage':
                return this.checkPercentage(flag, request);
            case 'user_list':
                return flag.allowedUsers.includes(request.userId);
            case 'canary':
                return request.headers['x-canary'] === 'true';
            case 'all':
                return true;
            default:
                return false;
        }
    }

    checkPercentage(flag, request) {
        const identifier = request.userId || request.ip;
        const hash = this.simpleHash(identifier + flag.name);
        return (hash % 100) < flag.percentage;
    }

    simpleHash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash; // Convert to 32bit integer
        }
        return Math.abs(hash);
    }

    // Admin API - percentage পরিবর্তন
    async updateRolloutPercentage(flagName, newPercentage) {
        const flag = this.flags.get(flagName);
        if (flag) {
            flag.percentage = Math.min(100, Math.max(0, newPercentage));
            this.flags.set(flagName, flag);
            await this.saveFlags();
        }
    }

    async saveFlags() {
        await this.redis.set(
            'strangler:feature_flags',
            JSON.stringify([...this.flags.values()])
        );
    }
}

// ===== Strangler Router =====
class StranglerRouter {
    constructor(featureFlags) {
        this.featureFlags = featureFlags;
        this.metrics = { legacy: 0, new: 0, fallback: 0 };

        this.routes = [
            {
                path: '/api/products',
                newService: 'http://product-service:3001',
                featureFlag: 'new_product_service',
                status: 'migrated', // migrated | migrating | legacy
            },
            {
                path: '/api/orders',
                newService: 'http://order-service:3002',
                featureFlag: 'new_order_service',
                status: 'migrating',
            },
            {
                path: '/api/payments',
                newService: 'http://payment-service:3003',
                featureFlag: 'new_payment_service',
                status: 'migrating',
            },
            {
                path: '/api/reports',
                newService: null,
                featureFlag: null,
                status: 'legacy',
            },
        ];
    }

    getTarget(request) {
        const route = this.routes.find(r => request.path.startsWith(r.path));

        if (!route || route.status === 'legacy') {
            this.metrics.legacy++;
            return { target: 'legacy', reason: 'no_migration' };
        }

        if (route.status === 'migrated') {
            this.metrics.new++;
            return { target: route.newService, reason: 'fully_migrated' };
        }

        // migrating - feature flag check
        if (this.featureFlags.isEnabled(route.featureFlag, request)) {
            this.metrics.new++;
            return { target: route.newService, reason: 'feature_flag_enabled' };
        }

        this.metrics.legacy++;
        return { target: 'legacy', reason: 'feature_flag_disabled' };
    }
}

// ===== Anti-Corruption Layer =====
class AntiCorruptionLayer {
    // Legacy response → New format (client-facing)
    transformProductResponse(legacyProduct) {
        return {
            id: legacyProduct.prod_id,
            name: legacyProduct.prod_name_bn || legacyProduct.prod_name,
            nameBn: legacyProduct.prod_name_bn,
            nameEn: legacyProduct.prod_name,
            price: {
                amount: parseFloat(legacyProduct.price),
                currency: 'BDT',
                formatted: `৳${parseFloat(legacyProduct.price).toLocaleString('bn-BD')}`,
            },
            stock: parseInt(legacyProduct.qty_available),
            category: this.mapCategory(legacyProduct.cat_id),
            images: this.parseImages(legacyProduct.img_urls),
            createdAt: new Date(legacyProduct.create_dt).toISOString(),
        };
    }

    // New format → Legacy format (for sync)
    transformProductToLegacy(newProduct) {
        return {
            prod_id: newProduct.id,
            prod_name: newProduct.nameEn || newProduct.name,
            prod_name_bn: newProduct.nameBn || newProduct.name,
            price: newProduct.price.amount.toString(),
            qty_available: newProduct.stock.toString(),
            cat_id: this.reverseCategoryMap(newProduct.category),
            img_urls: newProduct.images.join(','),
        };
    }

    mapCategory(catId) {
        const categories = {
            1: 'electronics',
            2: 'clothing',
            3: 'groceries',
            4: 'books',
            5: 'beauty',
        };
        return categories[catId] || 'other';
    }

    reverseCategoryMap(category) {
        const map = {
            electronics: 1,
            clothing: 2,
            groceries: 3,
            books: 4,
            beauty: 5,
        };
        return map[category] || 0;
    }

    parseImages(imgUrls) {
        if (!imgUrls) return [];
        return imgUrls.split(',').map(url => url.trim());
    }
}

// ===== Express App Setup =====
const featureFlags = new FeatureFlagManager(redis);
const router = new StranglerRouter(featureFlags);
const acl = new AntiCorruptionLayer();

const LEGACY_URL = 'http://legacy-php-app:8080';

// Middleware: Request enrichment
app.use((req, res, next) => {
    req.userId = req.headers['x-user-id'] || null;
    req.requestId = `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    req.startTime = Date.now();
    next();
});

// Main routing middleware
app.use('/api', async (req, res, next) => {
    const routing = router.getTarget(req);

    // Add routing info to response headers (debugging)
    res.setHeader('X-Routed-To', routing.target === 'legacy' ? 'legacy' : 'new');
    res.setHeader('X-Route-Reason', routing.reason);

    if (routing.target === 'legacy') {
        // Legacy system-এ proxy করা
        return proxyToLegacy(req, res, next);
    }

    // নতুন service-এ forward করা
    try {
        const response = await forwardToService(routing.target, req);

        // Response logging
        logRouting(req, routing, response.status);

        res.status(response.status).json(response.data);
    } catch (error) {
        // Fallback to legacy on error
        console.error(`[Strangler] Service error, falling back to legacy:`, error.message);
        router.metrics.fallback++;
        logRouting(req, { ...routing, fallback: true }, 500);
        return proxyToLegacy(req, res, next);
    }
});

function proxyToLegacy(req, res, next) {
    const proxy = httpProxy.createProxyMiddleware({
        target: LEGACY_URL,
        changeOrigin: true,
        onProxyRes: (proxyRes, req) => {
            // Legacy response-কে transform করা (যদি দরকার)
            proxyRes.headers['X-Source'] = 'legacy';
        },
    });
    return proxy(req, res, next);
}

async function forwardToService(targetUrl, req) {
    const url = `${targetUrl}${req.path}`;
    const response = await fetch(url, {
        method: req.method,
        headers: {
            'Content-Type': 'application/json',
            'X-Request-Id': req.requestId,
            'X-User-Id': req.userId || '',
        },
        body: ['POST', 'PUT', 'PATCH'].includes(req.method)
            ? JSON.stringify(req.body) : undefined,
    });
    const data = await response.json();
    return { status: response.status, data };
}

function logRouting(req, routing, statusCode) {
    console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        requestId: req.requestId,
        path: req.path,
        method: req.method,
        routedTo: routing.target,
        reason: routing.reason,
        fallback: routing.fallback || false,
        statusCode,
        duration: Date.now() - req.startTime,
    }));
}

// ===== Admin Endpoints - Migration Control =====
app.post('/admin/rollout/:flag', async (req, res) => {
    const { flag } = req.params;
    const { percentage } = req.body;
    await featureFlags.updateRolloutPercentage(flag, percentage);
    res.json({ success: true, flag, newPercentage: percentage });
});

app.get('/admin/metrics', (req, res) => {
    res.json({
        routing: router.metrics,
        routes: router.routes.map(r => ({
            path: r.path,
            status: r.status,
        })),
    });
});

// ===== Comparison Testing (Shadow Mode) =====
class ShadowTester {
    constructor(legacyUrl, newServiceUrl, acl) {
        this.legacyUrl = legacyUrl;
        this.newServiceUrl = newServiceUrl;
        this.acl = acl;
        this.discrepancies = [];
    }

    async compareResponses(req) {
        const [legacyRes, newRes] = await Promise.allSettled([
            fetch(`${this.legacyUrl}${req.path}`),
            fetch(`${this.newServiceUrl}${req.path}`),
        ]);

        if (legacyRes.status === 'fulfilled' && newRes.status === 'fulfilled') {
            const legacyData = await legacyRes.value.json();
            const newData = await newRes.value.json();

            // Legacy data transform করে compare করা
            const transformedLegacy = this.acl.transformProductResponse(legacyData);

            const isEqual = JSON.stringify(transformedLegacy) === JSON.stringify(newData);
            if (!isEqual) {
                this.discrepancies.push({
                    path: req.path,
                    timestamp: new Date().toISOString(),
                    legacy: transformedLegacy,
                    new: newData,
                });
            }
            return { match: isEqual };
        }
        return { match: false, error: 'One service failed' };
    }
}

app.listen(3000, () => {
    console.log('🌳 Strangler Facade চালু হয়েছে port 3000-এ');
});
```

---

## 🛡️ Anti-Corruption Layer

Anti-Corruption Layer (ACL) নতুন ও পুরানো সিস্টেমের মধ্যে **অনুবাদক** হিসেবে কাজ করে:

```
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│   New Service   │     │  Anti-Corruption    │     │  Legacy System  │
│                 │     │      Layer          │     │                 │
│  Modern Format  │◄───▶│  ┌──────────────┐  │◄───▶│  Old Format     │
│  {              │     │  │  Translator  │  │     │  {              │
│    id: "P001", │     │  │              │  │     │    prod_id: 1,  │
│    price: {    │     │  │ New ◄──► Old │  │     │    price: "500",│
│      amount:500│     │  │              │  │     │    cat_id: 3    │
│    }           │     │  └──────────────┘  │     │  }              │
│  }             │     │                     │     │                 │
└─────────────────┘     └─────────────────────┘     └─────────────────┘
```

---

## 🔙 Rollback Strategies

### Strategy 1: Feature Flag Rollback (তাৎক্ষণিক)
```
সমস্যা দেখা দিলে → Feature Flag percentage 0% করুন
→ সব ট্র্যাফিক Legacy-তে ফিরে যাবে
→ Zero downtime
```

### Strategy 2: Circuit Breaker
```
নতুন service-এ error rate বাড়লে:
  → Circuit OPEN করুন
  → সব request legacy-তে redirect
  → কিছুক্ষণ পর আবার try করুন (Half-Open)
```

### Strategy 3: Blue-Green Deployment
```
Blue (Legacy) ◄─── Active
Green (New)   ◄─── Standby

সমস্যা হলে: Traffic switch করুন Blue-তে
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

| ব্যবহার করবেন ✅ | ব্যবহার করবেন না ❌ |
|---|---|
| বড় legacy system আছে | ছোট অ্যাপ্লিকেশন, সহজেই rewrite হবে |
| "Big Bang" rewrite ঝুঁকিপূর্ণ | নতুন প্রজেক্ট তৈরি করছেন |
| ব্যবসা চলমান রাখতে হবে migration-এর সময় | Legacy system সহজ ও ছোট |
| দলে legacy ও modern উভয় দক্ষতা আছে | পুরানো system-এর code বোঝার সুযোগ নেই |
| Feature-by-feature আলাদা করা সম্ভব | Tight coupling - আলাদা করা প্রায় অসম্ভব |
| ধীরে ধীরে migrate করার সময় আছে | জরুরি ভিত্তিতে সম্পূর্ণ পরিবর্তন দরকার |

### ⚠️ ঝুঁকি ও চ্যালেঞ্জ:
1. **Dual Maintenance**: Migration চলাকালীন দুটো system maintain করতে হয়
2. **Data Consistency**: দুটো system-এ data sync রাখা কঠিন
3. **Increased Complexity**: Proxy layer, ACL - অতিরিক্ত complexity
4. **Team Coordination**: Legacy ও New team-এর মধ্যে সমন্বয় দরকার
5. **Testing Complexity**: দুটো system-ই test করতে হয়

---

## 📚 সারসংক্ষেপ

> Strangler Fig Pattern = পুরানো গাছকে (legacy) নতুন গাছ (microservices) ধীরে ধীরে ঘিরে ফেলা — **কখনো পুরানো গাছ কাটা হয় না** (until নতুন গাছ সম্পূর্ণ প্রস্তুত)।

বাংলাদেশের অনেক tech কোম্পানি (bKash, Pathao, Chaldal, 10 Minute School) তাদের প্রাথমিক monolithic system থেকে microservices-এ যাওয়ার সময় এই pattern ব্যবহার করে, কারণ এটি business continuity নিশ্চিত করে।
