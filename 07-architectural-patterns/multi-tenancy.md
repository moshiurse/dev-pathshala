# 🏢 Multi-Tenancy Architecture Patterns (মাল্টি-টেন্যান্সি)

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Multi-Tenancy Models](#multi-tenancy-models)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Tenant Routing](#tenant-routing)
- [Data Isolation ও Security](#data-isolation-ও-security)
- [Migration Strategies](#migration-strategies)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Multi-Tenancy** হলো এমন একটি সফটওয়্যার আর্কিটেকচার যেখানে একটি single application instance একাধিক গ্রাহক (tenant) কে সেবা প্রদান করে। প্রতিটি tenant-এর ডেটা ও কনফিগারেশন আলাদা থাকে, কিন্তু তারা একই সফটওয়্যার ও infrastructure ভাগাভাগি করে ব্যবহার করে।

### 🏠 সহজ উদাহরণ:

```
Single-Tenant (একটি বাড়ি):
┌─────────────────┐
│  একটি পরিবার    │ ← শুধু তারাই থাকে
│  নিজস্ব বাড়ি   │ ← নিজস্ব রান্নাঘর, বাথরুম
└─────────────────┘

Multi-Tenant (অ্যাপার্টমেন্ট বিল্ডিং):
┌─────────────────┐
│  Flat 4 - করিম  │ ← আলাদা flat
│  Flat 3 - রহিম  │ ← আলাদা flat
│  Flat 2 - সালাম │ ← আলাদা flat
│  Flat 1 - জামাল │ ← আলাদা flat
├─────────────────┤
│  Shared: লিফট,  │ ← ভাগাভাগি infrastructure
│  জেনারেটর, গার্ড│
└─────────────────┘
```

### 🔑 মূল ধারণাসমূহ:
- **Tenant**: একজন গ্রাহক/প্রতিষ্ঠান যে সিস্টেম ব্যবহার করছে
- **Data Isolation**: এক tenant-এর ডেটা অন্য tenant দেখতে পারবে না
- **Tenant Context**: বর্তমান request কোন tenant-এর সেটা নির্ধারণ
- **Customization**: প্রতিটি tenant-এর জন্য আলাদা settings/branding
- **Resource Sharing**: infrastructure ভাগাভাগি করে cost কমানো

---

## 🗃️ Multi-Tenancy Models

### Model 1: Database-per-Tenant (সম্পূর্ণ আলাদা)
```
┌───────────────────────────────────────────────────┐
│                DATABASE PER TENANT                  │
├───────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐│
│  │  App Server │  │  App Server │  │App Server ││
│  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘│
│         │                │                │      │
│         ▼                ▼                ▼      │
│  ┌───────────┐    ┌───────────┐    ┌──────────┐ │
│  │  DB:      │    │  DB:      │    │  DB:     │ │
│  │  স্কুল-A │    │  স্কুল-B │    │  স্কুল-C│ │
│  └───────────┘    └───────────┘    └──────────┘ │
│                                                    │
│  ✅ সম্পূর্ণ data isolation                       │
│  ✅ সহজে backup/restore per tenant               │
│  ❌ ব্যয়বহুল (অনেক DB instance)                 │
│  ❌ Cross-tenant query কঠিন                      │
└───────────────────────────────────────────────────┘
```

### Model 2: Schema-per-Tenant (একই DB, আলাদা schema)
```
┌───────────────────────────────────────────────────┐
│              SCHEMA PER TENANT                      │
├───────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐│
│  │           Single Database Server              ││
│  │                                               ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    ││
│  │  │Schema:   │ │Schema:   │ │Schema:   │    ││
│  │  │school_a  │ │school_b  │ │school_c  │    ││
│  │  │          │ │          │ │          │    ││
│  │  │-students │ │-students │ │-students │    ││
│  │  │-teachers │ │-teachers │ │-teachers │    ││
│  │  │-classes  │ │-classes  │ │-classes  │    ││
│  │  └──────────┘ └──────────┘ └──────────┘    ││
│  └──────────────────────────────────────────────┘│
│                                                    │
│  ✅ ভালো isolation                                │
│  ✅ কম খরচ (single DB server)                    │
│  ❌ Schema migration সব tenant-এ করতে হয়        │
│  ❌ DB server-এ connection limit সমস্যা          │
└───────────────────────────────────────────────────┘
```

### Model 3: Shared Database with Row-Level Security
```
┌───────────────────────────────────────────────────┐
│          SHARED DATABASE (Row-Level Security)      │
├───────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐│
│  │              Single Database                   ││
│  │              Single Schema                     ││
│  │                                               ││
│  │  students table:                              ││
│  │  ┌────┬────────┬───────────┬──────────┐     ││
│  │  │ id │tenant_id│  name    │  class   │     ││
│  │  ├────┼────────┼───────────┼──────────┤     ││
│  │  │ 1  │ sch_a  │  রহিম    │  ৮ম     │     ││
│  │  │ 2  │ sch_b  │  করিম    │  ৯ম     │     ││
│  │  │ 3  │ sch_a  │  জামাল   │  ১০ম    │     ││
│  │  │ 4  │ sch_c  │  সালমা   │  ৭ম     │     ││
│  │  └────┴────────┴───────────┴──────────┘     ││
│  │                                               ││
│  │  WHERE tenant_id = 'sch_a'  ← সব query-তে   ││
│  └──────────────────────────────────────────────┘│
│                                                    │
│  ✅ সবচেয়ে কম খরচ                               │
│  ✅ সহজ maintenance                              │
│  ❌ Data leak-এর ঝুঁকি (WHERE ভুলে গেলে)       │
│  ❌ "Noisy neighbor" সমস্যা                      │
└───────────────────────────────────────────────────┘
```

### Model 4: Hybrid Approach
```
┌───────────────────────────────────────────────────┐
│               HYBRID APPROACH                      │
├───────────────────────────────────────────────────┤
│                                                    │
│  Premium Tenants          Standard Tenants         │
│  (বড় স্কুল)             (ছোট স্কুল)            │
│                                                    │
│  ┌───────────┐         ┌──────────────────┐      │
│  │Dedicated  │         │  Shared Database │      │
│  │Database   │         │  (Row-Level)     │      │
│  │(school_a) │         │                  │      │
│  └───────────┘         │ school_b         │      │
│  ┌───────────┐         │ school_c         │      │
│  │Dedicated  │         │ school_d         │      │
│  │Database   │         │ school_e         │      │
│  │(school_x) │         │ ...              │      │
│  └───────────┘         └──────────────────┘      │
│                                                    │
│  ✅ Premium = best performance & isolation         │
│  ✅ Standard = cost-effective                      │
│  ✅ Upgrade path: shared → dedicated              │
└───────────────────────────────────────────────────┘
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏫 "EduBD" - বাংলাদেশী School Management SaaS

ধরুন আমরা একটি School Management System তৈরি করছি যা বাংলাদেশের বিভিন্ন স্কুল ব্যবহার করবে:

```
Tenants (গ্রাহক স্কুলসমূহ):
├── রাজউক উত্তরা মডেল কলেজ (Premium - dedicated DB)
├── ভিকারুননিসা নূন স্কুল (Premium - dedicated DB)
├── মতিঝিল সরকারি প্রাথমিক (Standard - shared DB)
├── গ্রামীণ হাই স্কুল, সিলেট (Standard - shared DB)
└── ABC International School (Premium - dedicated DB)

Features per tenant:
├── Student Management (ছাত্র-ছাত্রী তথ্য)
├── Teacher Management (শিক্ষক তথ্য)
├── Attendance (উপস্থিতি)
├── Result Management (ফলাফল)
├── Fee Collection (bKash/Nagad integration)
├── SMS Notification (parents-কে SMS)
└── Custom Reports
```

### 💼 অন্যান্য বাংলাদেশী SaaS উদাহরণ:

```
"HishabPati" - Accounting Software:
├── Tenant: ছোট দোকান (Standard)
├── Tenant: মাঝারি ব্যবসা (Premium)
└── Tenant: বড় কারখানা (Enterprise - dedicated)

"JobsBD" - Job Portal:
├── Tenant: বিভিন্ন কোম্পানি job post করছে
├── Isolation: এক কোম্পানির applicant data অন্যটা দেখতে পারবে না
└── Shared: Job search, candidate pool

"CloudKitchen BD" - Restaurant Management:
├── Tenant: সুলতানস ডাইন
├── Tenant: স্টার কাবাব
├── Tenant: পিৎজা হাট বাংলাদেশ
└── প্রতিটির আলাদা menu, order, staff
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### সম্পূর্ণ Multi-Tenant Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                   MULTI-TENANT ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │  School A    │ │  School B    │ │  School C    │            │
│  │  (Browser)   │ │  (Browser)   │ │  (Mobile)    │            │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘            │
│         │                │                │                     │
│         │  schoola.edubd.com  schoolb.edubd.com                 │
│         │                │                │                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              TENANT RESOLVER                         │       │
│  │  ┌───────────────────────────────────────────────┐  │       │
│  │  │  subdomain → tenant_id mapping                │  │       │
│  │  │  schoola.edubd.com → tenant_sch_a             │  │       │
│  │  │  schoolb.edubd.com → tenant_sch_b             │  │       │
│  │  │  custom-domain.com → tenant_sch_c             │  │       │
│  │  └───────────────────────────────────────────────┘  │       │
│  └──────────────────────┬──────────────────────────────┘       │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              APPLICATION LAYER                       │       │
│  │                                                      │       │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐   │       │
│  │  │  Tenant    │  │  Auth &    │  │  Business  │   │       │
│  │  │  Context   │  │  RBAC      │  │  Logic     │   │       │
│  │  └────────────┘  └────────────┘  └────────────┘   │       │
│  │                                                      │       │
│  └──────────────────────┬──────────────────────────────┘       │
│                         │                                       │
│              ┌──────────┼──────────┐                           │
│              │          │          │                            │
│              ▼          ▼          ▼                            │
│  ┌──────────────┐ ┌─────────┐ ┌──────────────┐               │
│  │  Dedicated   │ │ Shared  │ │   Shared     │               │
│  │  DB (Sch A)  │ │   DB    │ │   Storage    │               │
│  │  (Premium)   │ │(B,C,D..)│ │  (S3/Minio)  │               │
│  └──────────────┘ └─────────┘ └──────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tenant Routing Flow:
```
   Request: https://vikarnissa.edubd.com/api/students
   
   ┌─────────────────────────────────────────────────┐
   │  Step 1: DNS → Load Balancer                     │
   │  *.edubd.com → 103.x.x.x                       │
   └─────────────────────┬───────────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────────┐
   │  Step 2: Tenant Resolution                       │
   │  subdomain "vikarnissa" → tenant_id: "VNS001"  │
   └─────────────────────┬───────────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────────┐
   │  Step 3: Tenant Context Set                      │
   │  TenantContext::current() = "VNS001"            │
   └─────────────────────┬───────────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────────┐
   │  Step 4: Database Connection                     │
   │  VNS001 → dedicated DB (premium tenant)         │
   │  বা shared DB + WHERE tenant_id = 'VNS001'     │
   └─────────────────────┬───────────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────────┐
   │  Step 5: Response (শুধু VNS001-এর students)     │
   └─────────────────────────────────────────────────┘
```

### Data Isolation Diagram:
```
┌────────────────────────────────────────────────────┐
│              DATA ISOLATION LAYERS                   │
├────────────────────────────────────────────────────┤
│                                                     │
│  Layer 1: Network Level                            │
│  ┌─────────────────────────────────────────┐      │
│  │ Tenant A requests → only Tenant A data  │      │
│  └─────────────────────────────────────────┘      │
│                                                     │
│  Layer 2: Application Level                        │
│  ┌─────────────────────────────────────────┐      │
│  │ Global Query Scope:                      │      │
│  │ WHERE tenant_id = current_tenant()      │      │
│  └─────────────────────────────────────────┘      │
│                                                     │
│  Layer 3: Database Level                           │
│  ┌─────────────────────────────────────────┐      │
│  │ Row-Level Security (RLS) in PostgreSQL  │      │
│  │ OR separate schema/database             │      │
│  └─────────────────────────────────────────┘      │
│                                                     │
│  Layer 4: Storage Level                            │
│  ┌─────────────────────────────────────────┐      │
│  │ Files: /storage/{tenant_id}/files/      │      │
│  │ S3: s3://bucket/{tenant_id}/            │      │
│  └─────────────────────────────────────────┘      │
│                                                     │
└────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Multi-Tenant Laravel-style Implementation:

```php
<?php

/**
 * Tenant Model
 */
class Tenant
{
    public string $id;
    public string $name;
    public string $slug;
    public string $domain;
    public string $plan; // free, standard, premium
    public string $databaseType; // shared, dedicated
    public ?string $databaseName;
    public array $settings;
    public bool $isActive;
    
    public function isDedicatedDatabase(): bool
    {
        return $this->databaseType === 'dedicated';
    }
}

/**
 * Tenant Context - Current tenant ধরে রাখে (per-request)
 */
class TenantContext
{
    private static ?Tenant $currentTenant = null;

    public static function set(Tenant $tenant): void
    {
        self::$currentTenant = $tenant;
    }

    public static function get(): ?Tenant
    {
        return self::$currentTenant;
    }

    public static function getTenantId(): string
    {
        if (!self::$currentTenant) {
            throw new \RuntimeException('Tenant context not set');
        }
        return self::$currentTenant->id;
    }

    public static function reset(): void
    {
        self::$currentTenant = null;
    }
}

/**
 * Tenant Resolver - Request থেকে tenant নির্ধারণ
 */
class TenantResolver
{
    private PDO $masterDb;
    private Redis $cache;

    public function __construct(PDO $masterDb, Redis $cache)
    {
        $this->masterDb = $masterDb;
        $this->cache = $cache;
    }

    public function resolve(Request $request): Tenant
    {
        // Strategy 1: Subdomain-based
        $host = $request->getHost();
        $subdomain = $this->extractSubdomain($host);

        if ($subdomain) {
            return $this->resolveBySubdomain($subdomain);
        }

        // Strategy 2: Custom domain
        $tenant = $this->resolveByDomain($host);
        if ($tenant) return $tenant;

        // Strategy 3: Header-based (API calls)
        $tenantId = $request->getHeader('X-Tenant-ID');
        if ($tenantId) {
            return $this->resolveById($tenantId);
        }

        // Strategy 4: Path-based (/tenant-slug/api/...)
        $slug = $this->extractSlugFromPath($request->getPath());
        if ($slug) {
            return $this->resolveBySlug($slug);
        }

        throw new TenantNotFoundException('Tenant নির্ধারণ করা যায়নি');
    }

    private function resolveBySubdomain(string $subdomain): Tenant
    {
        // Cache check
        $cached = $this->cache->get("tenant:subdomain:$subdomain");
        if ($cached) {
            return unserialize($cached);
        }

        $stmt = $this->masterDb->prepare(
            "SELECT * FROM tenants WHERE slug = ? AND is_active = 1"
        );
        $stmt->execute([$subdomain]);
        $data = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$data) {
            throw new TenantNotFoundException("'$subdomain' নামে কোনো tenant পাওয়া যায়নি");
        }

        $tenant = $this->hydrateTenant($data);

        // Cache for 5 minutes
        $this->cache->setex("tenant:subdomain:$subdomain", 300, serialize($tenant));

        return $tenant;
    }

    private function resolveByDomain(string $domain): ?Tenant
    {
        $stmt = $this->masterDb->prepare(
            "SELECT * FROM tenants WHERE custom_domain = ? AND is_active = 1"
        );
        $stmt->execute([$domain]);
        $data = $stmt->fetch(PDO::FETCH_ASSOC);

        return $data ? $this->hydrateTenant($data) : null;
    }

    private function resolveById(string $id): Tenant
    {
        $stmt = $this->masterDb->prepare("SELECT * FROM tenants WHERE id = ? AND is_active = 1");
        $stmt->execute([$id]);
        $data = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$data) throw new TenantNotFoundException("Tenant ID '$id' পাওয়া যায়নি");
        return $this->hydrateTenant($data);
    }

    private function resolveBySlug(string $slug): Tenant
    {
        return $this->resolveBySubdomain($slug);
    }

    private function extractSubdomain(string $host): ?string
    {
        // vikarnissa.edubd.com → vikarnissa
        if (preg_match('/^([a-z0-9-]+)\.edubd\.com$/', $host, $matches)) {
            return $matches[1];
        }
        return null;
    }

    private function extractSlugFromPath(string $path): ?string
    {
        if (preg_match('#^/t/([a-z0-9-]+)/#', $path, $matches)) {
            return $matches[1];
        }
        return null;
    }

    private function hydrateTenant(array $data): Tenant
    {
        $tenant = new Tenant();
        $tenant->id = $data['id'];
        $tenant->name = $data['name'];
        $tenant->slug = $data['slug'];
        $tenant->domain = $data['custom_domain'] ?? '';
        $tenant->plan = $data['plan'];
        $tenant->databaseType = $data['database_type'];
        $tenant->databaseName = $data['database_name'];
        $tenant->settings = json_decode($data['settings'] ?? '{}', true);
        $tenant->isActive = (bool) $data['is_active'];
        return $tenant;
    }
}

/**
 * Database Connection Manager
 * Tenant-based database connection switching
 */
class TenantDatabaseManager
{
    private array $connections = [];
    private array $config;

    public function __construct(array $config)
    {
        $this->config = $config;
    }

    public function getConnection(?Tenant $tenant = null): PDO
    {
        $tenant = $tenant ?? TenantContext::get();

        if ($tenant->isDedicatedDatabase()) {
            return $this->getDedicatedConnection($tenant);
        }

        return $this->getSharedConnection($tenant);
    }

    private function getDedicatedConnection(Tenant $tenant): PDO
    {
        $key = "dedicated:{$tenant->id}";

        if (!isset($this->connections[$key])) {
            $dbName = $tenant->databaseName;
            $this->connections[$key] = new PDO(
                "mysql:host={$this->config['host']};dbname=$dbName;charset=utf8mb4",
                $this->config['username'],
                $this->config['password'],
                [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
            );
        }

        return $this->connections[$key];
    }

    private function getSharedConnection(Tenant $tenant): PDO
    {
        if (!isset($this->connections['shared'])) {
            $this->connections['shared'] = new PDO(
                "mysql:host={$this->config['host']};dbname={$this->config['shared_db']};charset=utf8mb4",
                $this->config['username'],
                $this->config['password'],
                [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
            );
        }

        return $this->connections['shared'];
    }
}

/**
 * Tenant-Aware Query Builder
 * স্বয়ংক্রিয়ভাবে tenant_id filter যোগ করে
 */
class TenantAwareRepository
{
    private TenantDatabaseManager $dbManager;

    public function __construct(TenantDatabaseManager $dbManager)
    {
        $this->dbManager = $dbManager;
    }

    public function findAll(string $table, array $conditions = []): array
    {
        $tenant = TenantContext::get();
        $db = $this->dbManager->getConnection();

        $where = '';
        $params = [];

        // Shared DB-তে tenant_id filter যোগ করা
        if (!$tenant->isDedicatedDatabase()) {
            $conditions['tenant_id'] = $tenant->id;
        }

        if (!empty($conditions)) {
            $clauses = [];
            foreach ($conditions as $key => $value) {
                $clauses[] = "$key = ?";
                $params[] = $value;
            }
            $where = 'WHERE ' . implode(' AND ', $clauses);
        }

        $stmt = $db->prepare("SELECT * FROM $table $where");
        $stmt->execute($params);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function insert(string $table, array $data): string
    {
        $tenant = TenantContext::get();
        $db = $this->dbManager->getConnection();

        // Shared DB-তে tenant_id যোগ করা
        if (!$tenant->isDedicatedDatabase()) {
            $data['tenant_id'] = $tenant->id;
        }

        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $stmt = $db->prepare("INSERT INTO $table ($columns) VALUES ($placeholders)");
        $stmt->execute(array_values($data));

        return $db->lastInsertId();
    }
}

/**
 * School Management - Multi-Tenant Example
 */
class StudentService
{
    private TenantAwareRepository $repository;

    public function __construct(TenantAwareRepository $repository)
    {
        $this->repository = $repository;
    }

    public function getAllStudents(): array
    {
        // tenant_id filter স্বয়ংক্রিয়ভাবে যোগ হবে
        return $this->repository->findAll('students');
    }

    public function getStudentsByClass(string $className): array
    {
        return $this->repository->findAll('students', ['class_name' => $className]);
    }

    public function enrollStudent(array $data): string
    {
        return $this->repository->insert('students', [
            'name' => $data['name'],
            'class_name' => $data['class'],
            'roll_number' => $data['roll'],
            'guardian_phone' => $data['guardian_phone'],
            'enrolled_at' => date('Y-m-d'),
        ]);
    }
}

/**
 * Tenant-Specific Configuration
 */
class TenantConfigService
{
    private Redis $redis;

    public function __construct(Redis $redis)
    {
        $this->redis = $redis;
    }

    public function get(string $key, $default = null)
    {
        $tenantId = TenantContext::getTenantId();
        $value = $this->redis->hget("tenant_config:$tenantId", $key);
        return $value !== false ? json_decode($value, true) : $default;
    }

    public function set(string $key, $value): void
    {
        $tenantId = TenantContext::getTenantId();
        $this->redis->hset("tenant_config:$tenantId", $key, json_encode($value));
    }

    // প্রতিটি স্কুলের নিজস্ব branding/settings
    public function getBranding(): array
    {
        return [
            'school_name' => $this->get('school_name', 'EduBD School'),
            'logo_url' => $this->get('logo_url', '/default-logo.png'),
            'primary_color' => $this->get('primary_color', '#1a73e8'),
            'academic_year' => $this->get('academic_year', '2024'),
            'grading_system' => $this->get('grading_system', 'gpa_5'), // gpa_5, gpa_4, marks
        ];
    }
}

/**
 * Middleware: Tenant Resolution
 */
class TenantMiddleware
{
    private TenantResolver $resolver;

    public function __construct(TenantResolver $resolver)
    {
        $this->resolver = $resolver;
    }

    public function handle(Request $request, callable $next): Response
    {
        try {
            $tenant = $this->resolver->resolve($request);
            TenantContext::set($tenant);

            $response = $next($request);

            TenantContext::reset();
            return $response;

        } catch (TenantNotFoundException $e) {
            return new Response(404, json_encode([
                'error' => 'প্রতিষ্ঠান খুঁজে পাওয়া যায়নি',
                'message' => $e->getMessage(),
            ]));
        }
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Multi-Tenant Express.js Application:

```javascript
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

const app = express();
const redis = new Redis();

// ===== Tenant Resolution =====

class TenantResolver {
    constructor(masterPool, redis) {
        this.masterPool = masterPool;
        this.redis = redis;
    }

    async resolve(req) {
        // Strategy 1: Subdomain
        const host = req.hostname;
        const subdomain = this.extractSubdomain(host);

        if (subdomain) {
            return await this.resolveBySubdomain(subdomain);
        }

        // Strategy 2: JWT payload-এ tenant info
        if (req.user && req.user.tenantId) {
            return await this.resolveById(req.user.tenantId);
        }

        // Strategy 3: Header
        const tenantHeader = req.headers['x-tenant-id'];
        if (tenantHeader) {
            return await this.resolveById(tenantHeader);
        }

        throw new Error('Tenant নির্ধারণ করা যায়নি');
    }

    extractSubdomain(host) {
        // vikarnissa.edubd.com → vikarnissa
        const match = host.match(/^([a-z0-9-]+)\.edubd\.com$/);
        return match ? match[1] : null;
    }

    async resolveBySubdomain(subdomain) {
        // Redis cache
        const cached = await this.redis.get(`tenant:slug:${subdomain}`);
        if (cached) return JSON.parse(cached);

        const result = await this.masterPool.query(
            'SELECT * FROM tenants WHERE slug = $1 AND is_active = true',
            [subdomain]
        );

        if (result.rows.length === 0) {
            throw new Error(`"${subdomain}" নামে কোনো প্রতিষ্ঠান নেই`);
        }

        const tenant = result.rows[0];
        await this.redis.setex(`tenant:slug:${subdomain}`, 300, JSON.stringify(tenant));
        return tenant;
    }

    async resolveById(id) {
        const cached = await this.redis.get(`tenant:id:${id}`);
        if (cached) return JSON.parse(cached);

        const result = await this.masterPool.query(
            'SELECT * FROM tenants WHERE id = $1 AND is_active = true',
            [id]
        );

        if (result.rows.length === 0) {
            throw new Error(`Tenant "${id}" পাওয়া যায়নি`);
        }

        const tenant = result.rows[0];
        await this.redis.setex(`tenant:id:${id}`, 300, JSON.stringify(tenant));
        return tenant;
    }
}

// ===== Database Connection Manager =====

class TenantDatabaseManager {
    constructor(config) {
        this.config = config;
        this.pools = new Map();
        this.sharedPool = new Pool({
            host: config.host,
            database: config.sharedDatabase,
            user: config.user,
            password: config.password,
            max: 50,
        });
    }

    getPool(tenant) {
        if (tenant.database_type === 'dedicated') {
            return this.getDedicatedPool(tenant);
        }
        return this.sharedPool;
    }

    getDedicatedPool(tenant) {
        if (!this.pools.has(tenant.id)) {
            this.pools.set(tenant.id, new Pool({
                host: this.config.host,
                database: tenant.database_name,
                user: this.config.user,
                password: this.config.password,
                max: 20,
            }));
        }
        return this.pools.get(tenant.id);
    }

    // Tenant-aware query wrapper
    async query(tenant, sql, params = []) {
        const pool = this.getPool(tenant);

        if (tenant.database_type === 'shared') {
            // Row-level security: SET tenant context before query
            const client = await pool.connect();
            try {
                await client.query('SET app.current_tenant = $1', [tenant.id]);
                const result = await client.query(sql, params);
                return result;
            } finally {
                client.release();
            }
        }

        return await pool.query(sql, params);
    }
}

// ===== Tenant Middleware =====

const masterPool = new Pool({
    host: process.env.DB_HOST,
    database: 'edubd_master',
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
});

const tenantResolver = new TenantResolver(masterPool, redis);
const dbManager = new TenantDatabaseManager({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    sharedDatabase: 'edubd_shared',
});

// Middleware: Tenant context inject
app.use(async (req, res, next) => {
    try {
        const tenant = await tenantResolver.resolve(req);
        req.tenant = tenant;
        req.db = dbManager; // Pass DB manager
        next();
    } catch (error) {
        res.status(404).json({
            error: 'প্রতিষ্ঠান খুঁজে পাওয়া যায়নি',
            message: error.message,
        });
    }
});

// ===== Student API (Tenant-Scoped) =====

app.get('/api/students', async (req, res) => {
    const { tenant, db } = req;

    let result;
    if (tenant.database_type === 'dedicated') {
        // Dedicated DB - no tenant_id filter needed
        result = await db.query(tenant, 'SELECT * FROM students ORDER BY name');
    } else {
        // Shared DB - tenant_id filter
        result = await db.query(tenant,
            'SELECT * FROM students WHERE tenant_id = $1 ORDER BY name',
            [tenant.id]
        );
    }

    res.json({
        school: tenant.name,
        total: result.rows.length,
        students: result.rows,
    });
});

app.post('/api/students', async (req, res) => {
    const { tenant, db } = req;
    const { name, className, roll, guardianPhone } = req.body;

    const insertData = tenant.database_type === 'dedicated'
        ? {
            sql: 'INSERT INTO students (name, class_name, roll_number, guardian_phone) VALUES ($1, $2, $3, $4) RETURNING *',
            params: [name, className, roll, guardianPhone],
          }
        : {
            sql: 'INSERT INTO students (tenant_id, name, class_name, roll_number, guardian_phone) VALUES ($1, $2, $3, $4, $5) RETURNING *',
            params: [tenant.id, name, className, roll, guardianPhone],
          };

    const result = await db.query(tenant, insertData.sql, insertData.params);
    res.status(201).json({ student: result.rows[0], message: 'ছাত্র/ছাত্রী সফলভাবে ভর্তি হয়েছে' });
});

// ===== Fee Collection (bKash Integration) =====

app.post('/api/fees/collect', async (req, res) => {
    const { tenant, db } = req;
    const { studentId, amount, month, paymentMethod } = req.body;

    // Tenant-specific fee configuration
    const feeConfig = await redis.hgetall(`tenant_config:${tenant.id}:fees`);

    const feeRecord = {
        student_id: studentId,
        amount: parseFloat(amount),
        month,
        payment_method: paymentMethod, // bkash, nagad, cash
        status: 'pending',
        collected_at: new Date().toISOString(),
    };

    if (tenant.database_type !== 'dedicated') {
        feeRecord.tenant_id = tenant.id;
    }

    const columns = Object.keys(feeRecord).join(', ');
    const placeholders = Object.keys(feeRecord).map((_, i) => `$${i + 1}`).join(', ');

    const result = await db.query(tenant,
        `INSERT INTO fee_collections (${columns}) VALUES (${placeholders}) RETURNING *`,
        Object.values(feeRecord)
    );

    // SMS notification to guardian
    if (tenant.settings?.sms_enabled) {
        await sendSMS(
            feeRecord.guardian_phone,
            `${tenant.name}: ৳${amount} ফি জমা হয়েছে (${month})। ধন্যবাদ।`
        );
    }

    res.json({ fee: result.rows[0], message: 'ফি সংগ্রহ সফল' });
});

// ===== Tenant Admin: Custom Settings =====

app.put('/api/admin/settings', async (req, res) => {
    const { tenant } = req;
    const settings = req.body;

    // প্রতিটি school-এর নিজস্ব settings
    const allowedSettings = [
        'school_name', 'logo_url', 'primary_color',
        'academic_year', 'grading_system', 'sms_enabled',
        'attendance_start_time', 'fee_due_date',
    ];

    for (const [key, value] of Object.entries(settings)) {
        if (allowedSettings.includes(key)) {
            await redis.hset(`tenant_config:${tenant.id}`, key, JSON.stringify(value));
        }
    }

    res.json({ message: 'সেটিংস আপডেট হয়েছে', settings });
});

// ===== Tenant Provisioning (New School Onboarding) =====

class TenantProvisioner {
    constructor(masterPool, dbManager, redis) {
        this.masterPool = masterPool;
        this.dbManager = dbManager;
        this.redis = redis;
    }

    async createTenant(data) {
        const { name, slug, plan, adminEmail, adminPhone } = data;

        // Step 1: Tenant record create
        const tenant = await this.masterPool.query(
            `INSERT INTO tenants (id, name, slug, plan, database_type, is_active, created_at)
             VALUES ($1, $2, $3, $4, $5, true, NOW()) RETURNING *`,
            [
                `tenant_${slug}`,
                name,
                slug,
                plan,
                plan === 'premium' ? 'dedicated' : 'shared',
            ]
        );

        const newTenant = tenant.rows[0];

        // Step 2: Database setup
        if (plan === 'premium') {
            await this.createDedicatedDatabase(newTenant);
        } else {
            // Shared DB-তে কোনো extra setup লাগে না
            // RLS policy সব tenant-এর জন্য কাজ করবে
        }

        // Step 3: Admin user create
        await this.createAdminUser(newTenant, adminEmail, adminPhone);

        // Step 4: Default settings
        await this.setupDefaultSettings(newTenant);

        return newTenant;
    }

    async createDedicatedDatabase(tenant) {
        const dbName = `edubd_${tenant.slug}`;

        // Create database
        await this.masterPool.query(`CREATE DATABASE ${dbName}`);

        // Run migrations on new database
        const newPool = new Pool({
            host: process.env.DB_HOST,
            database: dbName,
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
        });

        await newPool.query(`
            CREATE TABLE students (
                id SERIAL PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                class_name VARCHAR(50),
                roll_number VARCHAR(20),
                guardian_phone VARCHAR(20),
                enrolled_at DATE DEFAULT CURRENT_DATE
            );
            
            CREATE TABLE teachers (
                id SERIAL PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                subject VARCHAR(100),
                phone VARCHAR(20)
            );
            
            CREATE TABLE fee_collections (
                id SERIAL PRIMARY KEY,
                student_id INTEGER REFERENCES students(id),
                amount DECIMAL(10, 2),
                month VARCHAR(20),
                payment_method VARCHAR(50),
                status VARCHAR(20) DEFAULT 'pending',
                collected_at TIMESTAMP
            );
            
            CREATE TABLE attendance (
                id SERIAL PRIMARY KEY,
                student_id INTEGER REFERENCES students(id),
                date DATE,
                status VARCHAR(10),
                UNIQUE(student_id, date)
            );
        `);

        // Update tenant record with database name
        await this.masterPool.query(
            'UPDATE tenants SET database_name = $1 WHERE id = $2',
            [dbName, tenant.id]
        );

        await newPool.end();
    }

    async createAdminUser(tenant, email, phone) {
        // Admin user creation logic
        console.log(`Admin created for ${tenant.name}: ${email}`);
    }

    async setupDefaultSettings(tenant) {
        const defaults = {
            school_name: tenant.name,
            academic_year: new Date().getFullYear().toString(),
            grading_system: 'gpa_5',
            sms_enabled: false,
            attendance_start_time: '08:00',
            fee_due_date: '10',
        };

        for (const [key, value] of Object.entries(defaults)) {
            await this.redis.hset(
                `tenant_config:${tenant.id}`,
                key,
                JSON.stringify(value)
            );
        }
    }
}

// Onboarding endpoint
const provisioner = new TenantProvisioner(masterPool, dbManager, redis);

app.post('/api/admin/tenants', async (req, res) => {
    try {
        const tenant = await provisioner.createTenant(req.body);
        res.status(201).json({
            message: 'নতুন প্রতিষ্ঠান সফলভাবে তৈরি হয়েছে',
            tenant,
            url: `https://${tenant.slug}.edubd.com`,
        });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// ===== Cross-Tenant Analytics (Admin Only) =====
app.get('/api/superadmin/analytics', async (req, res) => {
    // শুধুমাত্র platform admin - cross-tenant data
    const stats = await masterPool.query(`
        SELECT 
            t.name as school_name,
            t.plan,
            t.created_at,
            (SELECT COUNT(*) FROM edubd_shared.students s WHERE s.tenant_id = t.id) as student_count
        FROM tenants t
        WHERE t.is_active = true
        ORDER BY student_count DESC
    `);

    res.json({
        total_schools: stats.rows.length,
        schools: stats.rows,
    });
});

app.listen(3000, () => {
    console.log('🏫 EduBD Multi-Tenant System চালু হয়েছে');
});
```

---

## 🔀 Tenant Routing

### Routing Strategies:
```
┌─────────────────────────────────────────────────────────┐
│              TENANT ROUTING STRATEGIES                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Subdomain-based:                                    │
│     vikarnissa.edubd.com → Tenant: ভিকারুননিসা        │
│     rajuk.edubd.com → Tenant: রাজউক                    │
│                                                          │
│  2. Path-based:                                         │
│     edubd.com/t/vikarnissa/students → Tenant: VNS      │
│     edubd.com/t/rajuk/students → Tenant: RAJUK         │
│                                                          │
│  3. Header-based:                                       │
│     GET /api/students                                   │
│     X-Tenant-ID: vikarnissa                            │
│                                                          │
│  4. JWT Claim:                                          │
│     Token payload: { tenantId: "vns001", role: "admin" }│
│                                                          │
│  5. Custom Domain:                                      │
│     portal.vikarnissa.edu.bd → Tenant: VNS             │
│     (DNS CNAME → edubd.com)                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 🛡️ Data Isolation ও Security

### PostgreSQL Row-Level Security (RLS):
```sql
-- Shared database-এ RLS enable করা
ALTER TABLE students ENABLE ROW LEVEL SECURITY;

-- Policy: প্রতিটি tenant শুধু নিজের data দেখতে পারবে
CREATE POLICY tenant_isolation ON students
    USING (tenant_id = current_setting('app.current_tenant'));

-- Application-এ: প্রতি request-এ tenant set করা
SET app.current_tenant = 'vikarnissa';
SELECT * FROM students; -- শুধু vikarnissa-র students দেখাবে
```

---

## 🔄 Migration Strategies

### নতুন Tenant Onboarding:
```
1. Tenant record তৈরি (master DB)
2. Database/Schema তৈরি (if dedicated)
3. Schema migration চালানো
4. Default data seed করা
5. Admin user তৈরি
6. DNS/SSL setup (if custom domain)
7. Welcome email পাঠানো
```

### Tenant Plan Upgrade (Shared → Dedicated):
```
Step 1: নতুন dedicated DB তৈরি
Step 2: Shared DB থেকে data export (WHERE tenant_id = X)
Step 3: Dedicated DB-তে data import
Step 4: Tenant config update (database_type = 'dedicated')
Step 5: Shared DB থেকে পুরানো data delete
Step 6: Cache invalidate
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

| Multi-Tenancy ব্যবহার করবেন ✅ | করবেন না ❌ |
|---|---|
| SaaS application তৈরি করছেন | Single-client custom software |
| একই software অনেক গ্রাহক ব্যবহার করবে | প্রতিটি client-এর requirement সম্পূর্ণ আলাদা |
| Infrastructure cost কমাতে চান | Compliance-এ shared infrastructure নিষিদ্ধ |
| Centralized maintenance চান | Client-রা নিজেদের hosting চায় |
| Economies of scale দরকার | খুব কম সংখ্যক client (২-৩ জন) |

### Model নির্বাচন গাইড:

| Model | কখন ব্যবহার করবেন |
|---|---|
| Database-per-Tenant | High security/compliance (ব্যাংক, সরকারি) |
| Schema-per-Tenant | মাঝারি isolation, PostgreSQL ব্যবহার করলে |
| Shared DB (Row-Level) | ছোট tenant, cost-sensitive, basic isolation |
| Hybrid | বিভিন্ন tier-এর গ্রাহক (free/premium/enterprise) |

### ⚠️ চ্যালেঞ্জসমূহ:
1. **Data Leak Risk**: ভুলে tenant filter না দিলে অন্যের data দেখা যাবে
2. **Noisy Neighbor**: এক tenant-এর heavy query অন্যদের slow করবে
3. **Schema Migration**: সব tenant-এ একসাথে migration চালানো কঠিন
4. **Backup/Restore**: একটি tenant-এর data restore করা (shared DB-তে)
5. **Customization Limit**: খুব বেশি tenant-specific feature দেওয়া কঠিন

---

## 📚 সারসংক্ষেপ

```
┌────────────────────────────────────────────────────┐
│          MULTI-TENANCY সিদ্ধান্ত গাইড              │
├────────────────────────────────────────────────────┤
│                                                     │
│  "কত isolation দরকার?"                             │
│  ├── সর্বোচ্চ → Database-per-Tenant               │
│  ├── মাঝারি → Schema-per-Tenant                   │
│  └── ন্যূনতম → Shared + Row-Level Security        │
│                                                     │
│  "Budget কেমন?"                                    │
│  ├── বেশি → Dedicated (per tenant)                 │
│  ├── মাঝারি → Hybrid                              │
│  └── কম → Shared Database                          │
│                                                     │
│  "কত tenant থাকবে?"                               │
│  ├── ১০-এর কম → Database-per-Tenant সম্ভব        │
│  ├── ১০-১০০ → Schema বা Hybrid                    │
│  └── ১০০+ → Shared Database (must)                 │
│                                                     │
└────────────────────────────────────────────────────┘
```

> **মূল শিক্ষা:** Multi-Tenancy হলো SaaS-এর ভিত্তি। বাংলাদেশে যেকোনো B2B SaaS (School Management, HR System, Accounting Software) তৈরি করতে গেলে Multi-Tenancy architecture বুঝতেই হবে। সঠিক model নির্বাচন নির্ভর করে: isolation requirement, tenant সংখ্যা, এবং budget-এর উপর।
