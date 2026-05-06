# 🔄 Secrets Rotation & Management Patterns

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

### Secrets Rotation কী?

**Secrets Rotation** হলো নিয়মিত সময় অন্তর secrets (passwords, API keys, certificates) পরিবর্তন করার প্রক্রিয়া — কোনো downtime ছাড়াই। এটি credential leakage-এর impact কমায় এবং compliance requirements পূরণ করে।

### কেন Secrets Rotation জরুরি?

```
১. Credential Leakage → যদি কোনো secret leak হয়, rotation করলে পুরানো secret অকেজো হয়
২. Compliance → PCI-DSS, SOC2, ISO 27001 — সবাই regular rotation দাবি করে
৩. Blast Radius কমানো → Compromised secret এর lifetime সীমিত
৪. Insider Threat → প্রাক্তন কর্মচারী/vendor-এর access বন্ধ
৫. Audit Trail → কে, কখন, কোন secret access করেছে — ট্র্যাক করা যায়
```

### Secret-এর প্রকারভেদ:

| Secret Type | উদাহরণ | Rotation Period |
|-------------|---------|-----------------|
| **Database Password** | PostgreSQL/MySQL credentials | ৩০ দিন |
| **API Keys** | Payment gateway, SMS API | ৯০ দিন |
| **TLS Certificates** | HTTPS, mTLS certs | ৯০ দিন বা কম |
| **Encryption Keys** | AES keys for data at rest | ১ বছর |
| **OAuth Secrets** | Client secret for OAuth apps | ৬০ দিন |
| **SSH Keys** | Server access keys | ৯০ দিন |
| **JWT Signing Keys** | Token signing private keys | ৩০ দিন |

### Rotation Strategies:

```
Strategy 1: Dual-Secret (Overlap Period)
─────────────────────────────────────────
Secret v1: ████████████████████████░░░░░░░
                        │ overlap │
Secret v2: ░░░░░░░░░░░░████████████████████
           ↑            ↑         ↑
        v1 active   both valid  v1 revoked

Strategy 2: Blue-Green Rotation
─────────────────────────────────
Blue  (active):  [Secret A] ──→ [Secret A] (revoke)
Green (standby): [Secret B] ──→ [Secret B] (activate)
                     ↕ switch

Strategy 3: Dynamic Secrets (Vault)
─────────────────────────────────────
Request → Vault generates new credential → Use → Auto-expire
          (short-lived, unique per request)
```

### Tools Overview:

| Tool | বৈশিষ্ট্য |
|------|-----------|
| **HashiCorp Vault** | Dynamic secrets, encryption as service, full audit |
| **AWS Secrets Manager** | Auto-rotation Lambda, RDS integration |
| **Azure Key Vault** | HSM-backed, Azure service integration |
| **GCP Secret Manager** | IAM-based access, version management |
| **Doppler** | Developer-friendly, environment sync |
| **SOPS** | Encrypted files in Git, multiple KMS support |

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 📡 Grameenphone — ৫০টি Microservice-এ Database Credential Rotation

```
সমস্যা:
- Grameenphone-র ৫০টি microservice একই PostgreSQL cluster ব্যবহার করে
- প্রতিটি service-এর আলাদা DB user আছে
- Compliance team বলেছে প্রতি ৩০ দিনে password change করতে হবে
- একসাথে ৫০টি service restart করা impossible — 5 কোটি user affected হবে

সমাধান: HashiCorp Vault Dynamic Secrets
- প্রতিটি service চালু হলে Vault থেকে temporary DB credential নেয়
- Credential ১ ঘণ্টা পর auto-expire হয়
- Service আবার নতুন credential নেয় (lease renewal)
- কোনো manual rotation দরকার নেই!
```

### 🛒 Daraz — Payment Gateway API Key Rotation:

```
Daraz-এর payment flow:
1. Customer → Daraz checkout → Payment Gateway (bKash/Nagad API)
2. API Key leak হলে attacker Daraz-এর নামে transaction করতে পারে

Rotation Plan:
- Primary Key: bkash_api_key_v1 (active)
- Secondary Key: bkash_api_key_v2 (standby)

Step 1: bKash portal-এ নতুন key generate (v2)
Step 2: Vault-এ v2 store করো
Step 3: Daraz services ধীরে ধীরে v2 তে switch করো
Step 4: সব service v2 তে switch হলে v1 revoke করো

Result: Zero downtime! 🎉
```

### 📰 Prothom Alo — Emergency Rotation Scenario:

```
ঘটনা: একজন developer ভুলে Git-এ DB password push করেছে

Emergency Response:
T+0min:  Alert — GitHub secret scanning detected exposed credential
T+2min:  Vault-এ emergency rotation trigger
T+3min:  নতুন password set in PostgreSQL (old still valid — grace period)
T+5min:  সব service নতুন credential নিয়েছে (Vault lease refresh)
T+10min: পুরানো password revoke
T+15min: Git history থেকে secret remove (git filter-branch)
T+30min: Incident report তৈরি

সময়: মোট ৩০ মিনিটে সমস্যা সমাধান (manual হলে ৪-৫ ঘণ্টা লাগতো)
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### Secret Management Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Grameenphone Platform                      │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Billing  │  │  CRM     │  │  SMS     │  │ Recharge │   │
│  │ Service  │  │ Service  │  │ Gateway  │  │ Service  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │          │
│       └──────────────┴──────┬───────┴──────────────┘          │
│                             │                                 │
│                             ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              HashiCorp Vault Cluster                  │    │
│  │                                                      │    │
│  │  ┌──────────┐ ┌───────────┐ ┌──────────────────┐   │    │
│  │  │  Secret  │ │  Dynamic  │ │   PKI Engine     │   │    │
│  │  │  Engine  │ │  DB Creds │ │ (Certificates)   │   │    │
│  │  └──────────┘ └───────────┘ └──────────────────┘   │    │
│  │                                                      │    │
│  │  ┌──────────┐ ┌───────────┐ ┌──────────────────┐   │    │
│  │  │  Audit   │ │  Policy   │ │  Auth Methods    │   │    │
│  │  │  Logs    │ │  Engine   │ │ (K8s/AppRole)    │   │    │
│  │  └──────────┘ └───────────┘ └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                             │                                 │
│                             ▼                                 │
│  ┌──────────────────────────────────────────────────┐       │
│  │           PostgreSQL / MySQL Cluster               │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Dual-Secret Rotation Flow:

```
Timeline ─────────────────────────────────────────────────────>

Phase 1: Normal Operation
┌─────────┐     ┌───────┐
│ Service │────▶│  DB   │  Using: password_v1
└─────────┘     └───────┘

Phase 2: New Secret Created (Overlap Starts)
┌─────────┐     ┌───────┐
│  Vault  │────▶│  DB   │  password_v2 created
└─────────┘     └───────┘              (both v1 & v2 valid)

Phase 3: Services Migrate
┌─────────┐     ┌───────┐
│ Service │────▶│  DB   │  Gradually switching to v2
│  (v1→v2)│     └───────┘
└─────────┘

Phase 4: Old Secret Revoked
┌─────────┐     ┌───────┐
│ Service │────▶│  DB   │  Only v2 valid, v1 revoked
│   (v2)  │     └───────┘
└─────────┘

Total downtime: 0 seconds ✅
```

### Secret Injection Patterns:

```
Pattern 1: Environment Variables
┌─────────────────────────────────────┐
│  Pod                                 │
│  ┌───────────────────────────────┐  │
│  │  Container                     │  │
│  │  ENV: DB_PASS=xK9#mP2$        │  │
│  │  (injected at startup)         │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
⚠️ Risk: env vars in process list, crash dumps

Pattern 2: Mounted Volume
┌─────────────────────────────────────┐
│  Pod                                 │
│  ┌────────────┐  ┌──────────────┐  │
│  │ Container  │  │ Secret Volume│  │
│  │            │──│ /secrets/    │  │
│  │ reads file │  │  db-pass.txt │  │
│  └────────────┘  └──────────────┘  │
└─────────────────────────────────────┘
✅ Better: file-based, can be rotated

Pattern 3: Vault Sidecar/Init Container
┌─────────────────────────────────────┐
│  Pod                                 │
│  ┌────────────┐  ┌──────────────┐  │
│  │    App     │  │ Vault Agent  │  │
│  │ Container  │  │  (sidecar)   │  │
│  │            │◀─│ Fetches +    │  │
│  │ reads      │  │ renews       │  │
│  │ /secrets/  │  │ secrets      │  │
│  └────────────┘  └──────┬───────┘  │
└──────────────────────────┼──────────┘
                           │
                    ┌──────▼──────┐
                    │    Vault    │
                    └─────────────┘
✅ Best: auto-rotation, lease management
```

### Dynamic Secrets Flow (Vault):

```
┌──────────┐                    ┌───────────┐                ┌──────────┐
│ Service  │                    │   Vault   │                │ Database │
└────┬─────┘                    └─────┬─────┘                └────┬─────┘
     │                                │                           │
     │ 1. "I need DB access"          │                           │
     │ (with auth token)              │                           │
     │───────────────────────────────▶│                           │
     │                                │                           │
     │                                │ 2. CREATE USER temp_abc   │
     │                                │    WITH PASSWORD 'xyz'    │
     │                                │    VALID FOR 1 HOUR       │
     │                                │──────────────────────────▶│
     │                                │                           │
     │                                │ 3. User created ✅        │
     │                                │◀──────────────────────────│
     │                                │                           │
     │ 4. Here's your credential:     │                           │
     │    user=temp_abc, pass=xyz     │                           │
     │    lease=1hr, renewable        │                           │
     │◀───────────────────────────────│                           │
     │                                │                           │
     │ ... 55 minutes later ...       │                           │
     │                                │                           │
     │ 5. Renew my lease please       │                           │
     │───────────────────────────────▶│                           │
     │                                │                           │
     │ 6. Extended to +1hr            │                           │
     │◀───────────────────────────────│                           │
     │                                │                           │
     │ ... service stops ...          │                           │
     │                                │                           │
     │                                │ 7. Lease expired →        │
     │                                │    DROP USER temp_abc     │
     │                                │──────────────────────────▶│
     │                                │                           │
```

---

## 💻 PHP কোড উদাহরণ

### Complete Secrets Rotation System:

```php
<?php

namespace Grameenphone\Secrets;

/**
 * Vault Client — HashiCorp Vault-এর সাথে communicate
 */
class VaultClient
{
    private string $vaultAddr;
    private string $token;
    private int $timeout;

    public function __construct(string $vaultAddr, string $token, int $timeout = 30)
    {
        $this->vaultAddr = rtrim($vaultAddr, '/');
        $this->token = $token;
        $this->timeout = $timeout;
    }

    /**
     * Static secret পড়া (KV v2 engine)
     */
    public function readSecret(string $path): array
    {
        $response = $this->request('GET', "/v1/secret/data/{$path}");
        return $response['data']['data'] ?? [];
    }

    /**
     * Dynamic database credential নেওয়া
     */
    public function getDatabaseCredential(string $role): DatabaseLease
    {
        $response = $this->request('GET', "/v1/database/creds/{$role}");

        return new DatabaseLease(
            username: $response['data']['username'],
            password: $response['data']['password'],
            leaseId: $response['lease_id'],
            leaseDuration: $response['lease_duration'],
            renewable: $response['renewable']
        );
    }

    /**
     * Lease renew করা
     */
    public function renewLease(string $leaseId, int $increment = 3600): array
    {
        return $this->request('PUT', '/v1/sys/leases/renew', [
            'lease_id' => $leaseId,
            'increment' => $increment,
        ]);
    }

    /**
     * Lease revoke করা (emergency rotation)
     */
    public function revokeLease(string $leaseId): void
    {
        $this->request('PUT', '/v1/sys/leases/revoke', [
            'lease_id' => $leaseId,
        ]);
    }

    /**
     * Secret-এর নতুন version তৈরি (rotation)
     */
    public function writeSecret(string $path, array $data): array
    {
        return $this->request('POST', "/v1/secret/data/{$path}", [
            'data' => $data,
        ]);
    }

    private function request(string $method, string $path, array $data = null): array
    {
        $ch = curl_init($this->vaultAddr . $path);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, $this->timeout);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'X-Vault-Token: ' . $this->token,
            'Content-Type: application/json',
        ]);

        if ($data) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new \RuntimeException("Vault error ({$httpCode}): {$response}");
        }

        return json_decode($response, true);
    }
}

/**
 * Database Lease — Vault থেকে পাওয়া temporary credential
 */
class DatabaseLease
{
    public readonly \DateTimeImmutable $expiresAt;

    public function __construct(
        public readonly string $username,
        public readonly string $password,
        public readonly string $leaseId,
        public readonly int $leaseDuration,
        public readonly bool $renewable
    ) {
        $this->expiresAt = new \DateTimeImmutable("+{$leaseDuration} seconds");
    }

    public function isExpiringSoon(int $thresholdSeconds = 300): bool
    {
        $now = new \DateTimeImmutable();
        $remaining = $this->expiresAt->getTimestamp() - $now->getTimestamp();
        return $remaining <= $thresholdSeconds;
    }

    public function getDSN(string $host, int $port, string $database): string
    {
        return "pgsql:host={$host};port={$port};dbname={$database};user={$this->username};password={$this->password}";
    }
}

/**
 * Secret Rotation Manager — automated rotation orchestration
 */
class SecretRotationManager
{
    private VaultClient $vault;
    private array $rotationPolicies;
    private AuditLogger $auditLogger;

    public function __construct(
        VaultClient $vault,
        array $rotationPolicies,
        AuditLogger $auditLogger
    ) {
        $this->vault = $vault;
        $this->rotationPolicies = $rotationPolicies;
        $this->auditLogger = $auditLogger;
    }

    /**
     * Dual-secret rotation — zero downtime
     */
    public function rotateDualSecret(string $secretPath, callable $generateNew): void
    {
        $this->auditLogger->log('rotation_started', $secretPath);

        // Step 1: বর্তমান secret পড়ো
        $current = $this->vault->readSecret($secretPath);
        $currentVersion = $current['_version'] ?? 1;

        // Step 2: নতুন secret generate করো
        $newSecret = $generateNew();

        // Step 3: নতুন secret কে primary করো, পুরানোটা secondary
        $rotatedData = [
            'primary' => $newSecret,
            'secondary' => $current['primary'] ?? $current,
            '_version' => $currentVersion + 1,
            '_rotated_at' => date('c'),
            '_next_rotation' => $this->calculateNextRotation($secretPath),
        ];

        // Step 4: Vault-এ save করো
        $this->vault->writeSecret($secretPath, $rotatedData);

        $this->auditLogger->log('rotation_completed', $secretPath, [
            'old_version' => $currentVersion,
            'new_version' => $currentVersion + 1,
        ]);
    }

    /**
     * Emergency rotation — credential leak হলে
     */
    public function emergencyRotate(string $secretPath, string $reason): void
    {
        $this->auditLogger->log('emergency_rotation_triggered', $secretPath, [
            'reason' => $reason,
            'severity' => 'CRITICAL',
        ]);

        // তাৎক্ষণিক rotation — overlap period ছাড়া
        $newSecret = bin2hex(random_bytes(32));

        $this->vault->writeSecret($secretPath, [
            'primary' => $newSecret,
            'secondary' => null, // পুরানো secret তাৎক্ষণিক বাতিল
            '_version' => 'emergency_' . time(),
            '_rotated_at' => date('c'),
            '_emergency' => true,
            '_reason' => $reason,
        ]);

        // সব active lease revoke করো
        $this->revokeAllLeases($secretPath);

        // Alert পাঠাও
        $this->sendAlert('CRITICAL', "Emergency rotation: {$secretPath} — {$reason}");

        $this->auditLogger->log('emergency_rotation_completed', $secretPath);
    }

    /**
     * Rotation schedule check — কোন secrets rotate করা দরকার
     */
    public function checkRotationSchedule(): array
    {
        $dueForRotation = [];

        foreach ($this->rotationPolicies as $path => $policy) {
            $secret = $this->vault->readSecret($path);
            $lastRotated = $secret['_rotated_at'] ?? null;

            if (!$lastRotated) {
                $dueForRotation[] = ['path' => $path, 'reason' => 'never_rotated'];
                continue;
            }

            $lastRotatedTime = new \DateTimeImmutable($lastRotated);
            $now = new \DateTimeImmutable();
            $daysSinceRotation = $now->diff($lastRotatedTime)->days;

            if ($daysSinceRotation >= $policy['max_age_days']) {
                $dueForRotation[] = [
                    'path' => $path,
                    'reason' => 'max_age_exceeded',
                    'days_since_rotation' => $daysSinceRotation,
                    'max_age' => $policy['max_age_days'],
                ];
            }
        }

        return $dueForRotation;
    }

    private function calculateNextRotation(string $secretPath): string
    {
        $policy = $this->rotationPolicies[$secretPath] ?? ['max_age_days' => 30];
        $nextDate = new \DateTimeImmutable("+{$policy['max_age_days']} days");
        return $nextDate->format('c');
    }

    private function revokeAllLeases(string $secretPath): void
    {
        $this->vault->request('PUT', '/v1/sys/leases/revoke-prefix/' . $secretPath);
    }

    private function sendAlert(string $severity, string $message): void
    {
        // Slack/PagerDuty/SMS alert
        error_log("[{$severity}] {$message}");
    }
}

/**
 * Dynamic DB Connection — auto-renewing credential
 */
class DynamicDatabaseConnection
{
    private VaultClient $vault;
    private string $vaultRole;
    private string $dbHost;
    private int $dbPort;
    private string $dbName;
    private ?DatabaseLease $currentLease = null;
    private ?\PDO $connection = null;

    public function __construct(
        VaultClient $vault,
        string $vaultRole,
        string $dbHost,
        int $dbPort,
        string $dbName
    ) {
        $this->vault = $vault;
        $this->vaultRole = $vaultRole;
        $this->dbHost = $dbHost;
        $this->dbPort = $dbPort;
        $this->dbName = $dbName;
    }

    /**
     * Connection পাওয়া — auto credential management সহ
     */
    public function getConnection(): \PDO
    {
        // Lease নেই বা expire হচ্ছে → নতুন credential নাও
        if (!$this->currentLease || $this->currentLease->isExpiringSoon()) {
            $this->refreshCredential();
        }

        return $this->connection;
    }

    private function refreshCredential(): void
    {
        // আগের lease renew try করো
        if ($this->currentLease && $this->currentLease->renewable) {
            try {
                $this->vault->renewLease($this->currentLease->leaseId);
                return;
            } catch (\Exception $e) {
                // Renewal failed — নতুন credential নাও
            }
        }

        // নতুন dynamic credential নাও
        $this->currentLease = $this->vault->getDatabaseCredential($this->vaultRole);

        // নতুন connection তৈরি করো
        $dsn = $this->currentLease->getDSN($this->dbHost, $this->dbPort, $this->dbName);
        $this->connection = new \PDO($dsn, $this->currentLease->username, $this->currentLease->password, [
            \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
            \PDO::ATTR_TIMEOUT => 5,
        ]);
    }

    public function __destruct()
    {
        // Service বন্ধ হলে lease revoke করো
        if ($this->currentLease) {
            try {
                $this->vault->revokeLease($this->currentLease->leaseId);
            } catch (\Exception $e) {
                // Vault will auto-revoke on expiry
            }
        }
    }
}

/**
 * Secret Leakage Detector
 */
class SecretLeakageDetector
{
    private array $patterns = [
        'aws_key' => '/AKIA[0-9A-Z]{16}/',
        'private_key' => '/-----BEGIN (RSA |EC )?PRIVATE KEY-----/',
        'password_in_url' => '/:\/\/[^:]+:[^@]+@/',
        'jwt_token' => '/eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+/',
        'generic_secret' => '/(password|secret|token|api_key)\s*[:=]\s*["\'][^"\']{8,}["\']/',
    ];

    /**
     * Code-এ secret leak হয়েছে কিনা scan করো
     */
    public function scanFile(string $filePath): array
    {
        $content = file_get_contents($filePath);
        $findings = [];

        foreach ($this->patterns as $type => $pattern) {
            if (preg_match_all($pattern, $content, $matches, PREG_OFFSET_CAPTURE)) {
                foreach ($matches[0] as $match) {
                    $lineNumber = substr_count(substr($content, 0, $match[1]), "\n") + 1;
                    $findings[] = [
                        'type' => $type,
                        'file' => $filePath,
                        'line' => $lineNumber,
                        'snippet' => $this->maskSecret($match[0]),
                    ];
                }
            }
        }

        return $findings;
    }

    private function maskSecret(string $secret): string
    {
        $length = strlen($secret);
        if ($length <= 8) return str_repeat('*', $length);
        return substr($secret, 0, 4) . str_repeat('*', $length - 8) . substr($secret, -4);
    }
}

/**
 * Audit Logger
 */
class AuditLogger
{
    public function log(string $action, string $resource, array $metadata = []): void
    {
        $entry = [
            'timestamp' => date('c'),
            'action' => $action,
            'resource' => $resource,
            'metadata' => $metadata,
            'actor' => $this->getCurrentActor(),
        ];
        // Append to immutable audit log
        file_put_contents(
            '/var/log/secrets-audit.jsonl',
            json_encode($entry) . "\n",
            FILE_APPEND | LOCK_EX
        );
    }

    private function getCurrentActor(): string
    {
        return getenv('SERVICE_NAME') ?: 'unknown-service';
    }
}

// === ব্যবহারের উদাহরণ ===

// Vault client setup
$vault = new VaultClient(
    vaultAddr: 'https://vault.gp.internal:8200',
    token: getenv('VAULT_TOKEN')
);

// Dynamic DB connection (Grameenphone Billing Service)
$db = new DynamicDatabaseConnection(
    vault: $vault,
    vaultRole: 'billing-service-readonly',
    dbHost: 'postgres-cluster.gp.internal',
    dbPort: 5432,
    dbName: 'billing'
);

$pdo = $db->getConnection();
$stmt = $pdo->query("SELECT * FROM invoices WHERE month = '2024-01'");

// Rotation schedule check
$rotationManager = new SecretRotationManager(
    vault: $vault,
    rotationPolicies: [
        'services/billing/api-key' => ['max_age_days' => 90],
        'services/sms-gateway/password' => ['max_age_days' => 30],
        'services/payment/bkash-secret' => ['max_age_days' => 60],
    ],
    auditLogger: new AuditLogger()
);

$dueSecrets = $rotationManager->checkRotationSchedule();
foreach ($dueSecrets as $secret) {
    echo "⚠️ Rotation needed: {$secret['path']} — {$secret['reason']}\n";
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Complete Secrets Management System (Node.js):

```javascript
const crypto = require('crypto');
const https = require('https');

/**
 * Vault Client — HashiCorp Vault API wrapper
 */
class VaultClient {
    constructor(config) {
        this.address = config.address;
        this.token = config.token;
        this.namespace = config.namespace || '';
    }

    /**
     * KV v2 secret পড়া
     */
    async readSecret(path) {
        const response = await this._request('GET', `/v1/secret/data/${path}`);
        return response.data.data;
    }

    /**
     * KV v2 secret লেখা
     */
    async writeSecret(path, data) {
        return this._request('POST', `/v1/secret/data/${path}`, { data });
    }

    /**
     * Dynamic database credential
     */
    async getDatabaseCredential(role) {
        const response = await this._request('GET', `/v1/database/creds/${role}`);
        return {
            username: response.data.username,
            password: response.data.password,
            leaseId: response.lease_id,
            leaseDuration: response.lease_duration,
            renewable: response.renewable,
            expiresAt: new Date(Date.now() + response.lease_duration * 1000),
        };
    }

    /**
     * Lease renew
     */
    async renewLease(leaseId, increment = 3600) {
        return this._request('PUT', '/v1/sys/leases/renew', {
            lease_id: leaseId,
            increment,
        });
    }

    /**
     * Lease revoke
     */
    async revokeLease(leaseId) {
        return this._request('PUT', '/v1/sys/leases/revoke', {
            lease_id: leaseId,
        });
    }

    async _request(method, path, body = null) {
        const url = new URL(path, this.address);

        const options = {
            method,
            hostname: url.hostname,
            port: url.port,
            path: url.pathname,
            headers: {
                'X-Vault-Token': this.token,
                'Content-Type': 'application/json',
                ...(this.namespace && { 'X-Vault-Namespace': this.namespace }),
            },
        };

        return new Promise((resolve, reject) => {
            const req = https.request(options, (res) => {
                let data = '';
                res.on('data', chunk => data += chunk);
                res.on('end', () => {
                    if (res.statusCode >= 400) {
                        reject(new Error(`Vault error ${res.statusCode}: ${data}`));
                    } else {
                        resolve(JSON.parse(data));
                    }
                });
            });
            req.on('error', reject);
            if (body) req.write(JSON.stringify(body));
            req.end();
        });
    }
}

/**
 * Secret Rotation Orchestrator
 * Grameenphone-র ৫০টি microservice-এ credential rotation
 */
class SecretRotationOrchestrator {
    constructor(vault, config) {
        this.vault = vault;
        this.config = config;
        this.rotationTimers = new Map();
        this.auditLog = [];
    }

    /**
     * Dual-secret rotation — zero downtime guarantee
     */
    async rotateDualSecret(secretPath, generateNewFn) {
        const startTime = Date.now();
        this.audit('rotation_start', secretPath);

        try {
            // Step 1: বর্তমান secret পড়ো
            const current = await this.vault.readSecret(secretPath);
            const currentVersion = current._version || 1;

            // Step 2: নতুন secret generate
            const newSecret = await generateNewFn();

            // Step 3: Dual-secret format-এ save (overlap period)
            const rotatedData = {
                primary: newSecret,
                secondary: current.primary || current.value,
                _version: currentVersion + 1,
                _rotated_at: new Date().toISOString(),
                _rotation_duration_ms: Date.now() - startTime,
            };

            await this.vault.writeSecret(secretPath, rotatedData);

            this.audit('rotation_complete', secretPath, {
                oldVersion: currentVersion,
                newVersion: currentVersion + 1,
                durationMs: Date.now() - startTime,
            });

            return rotatedData;
        } catch (error) {
            this.audit('rotation_failed', secretPath, { error: error.message });
            throw error;
        }
    }

    /**
     * Emergency rotation — leaked credential তাৎক্ষণিক বাতিল
     */
    async emergencyRotate(secretPath, reason) {
        console.error(`🚨 EMERGENCY ROTATION: ${secretPath} — ${reason}`);
        this.audit('emergency_rotation', secretPath, { reason, severity: 'CRITICAL' });

        // নতুন secret generate
        const newSecret = crypto.randomBytes(32).toString('hex');

        // Overlap period ছাড়া — পুরানো তাৎক্ষণিক বাতিল
        await this.vault.writeSecret(secretPath, {
            primary: newSecret,
            secondary: null,
            _version: `emergency_${Date.now()}`,
            _rotated_at: new Date().toISOString(),
            _emergency: true,
            _reason: reason,
        });

        // Alert team
        await this.sendCriticalAlert({
            type: 'EMERGENCY_ROTATION',
            path: secretPath,
            reason,
            timestamp: new Date().toISOString(),
        });

        return newSecret;
    }

    /**
     * Scheduled rotation setup
     */
    startScheduledRotation(policies) {
        for (const [path, policy] of Object.entries(policies)) {
            const intervalMs = policy.maxAgeDays * 24 * 60 * 60 * 1000;

            const timer = setInterval(async () => {
                try {
                    await this.rotateDualSecret(path, policy.generator);
                    console.log(`✅ Auto-rotated: ${path}`);
                } catch (err) {
                    console.error(`❌ Rotation failed: ${path}`, err.message);
                }
            }, intervalMs);

            this.rotationTimers.set(path, timer);
        }
    }

    /**
     * কোন secrets rotation-এ due তা check
     */
    async getRotationStatus(policies) {
        const status = [];

        for (const [path, policy] of Object.entries(policies)) {
            try {
                const secret = await this.vault.readSecret(path);
                const lastRotated = secret._rotated_at
                    ? new Date(secret._rotated_at)
                    : null;
                const daysSince = lastRotated
                    ? Math.floor((Date.now() - lastRotated.getTime()) / 86400000)
                    : Infinity;

                status.push({
                    path,
                    lastRotated: lastRotated?.toISOString() || 'never',
                    daysSinceRotation: daysSince,
                    maxAgeDays: policy.maxAgeDays,
                    isDue: daysSince >= policy.maxAgeDays,
                    urgency: daysSince >= policy.maxAgeDays * 1.5 ? 'CRITICAL' : 
                             daysSince >= policy.maxAgeDays ? 'WARNING' : 'OK',
                });
            } catch (err) {
                status.push({ path, error: err.message, urgency: 'ERROR' });
            }
        }

        return status;
    }

    audit(action, resource, metadata = {}) {
        const entry = {
            timestamp: new Date().toISOString(),
            action,
            resource,
            ...metadata,
        };
        this.auditLog.push(entry);
        console.log(`[AUDIT] ${action}: ${resource}`, metadata);
    }

    async sendCriticalAlert(alert) {
        // Slack webhook / PagerDuty
        console.error('🚨 ALERT:', JSON.stringify(alert));
    }

    stopAll() {
        for (const [path, timer] of this.rotationTimers) {
            clearInterval(timer);
        }
        this.rotationTimers.clear();
    }
}

/**
 * Dynamic Database Pool — auto-renewing credentials
 */
class DynamicDatabasePool {
    constructor(vault, config) {
        this.vault = vault;
        this.role = config.role;
        this.host = config.host;
        this.port = config.port;
        this.database = config.database;
        this.currentLease = null;
        this.pool = null;
        this.renewalTimer = null;
    }

    /**
     * Connection pool initialize — Vault থেকে credential নিয়ে
     */
    async initialize() {
        await this.refreshCredentials();
        this.startLeaseRenewal();
    }

    async refreshCredentials() {
        // পুরানো lease revoke
        if (this.currentLease) {
            try {
                await this.vault.revokeLease(this.currentLease.leaseId);
            } catch (e) { /* ignore */ }
        }

        // নতুন dynamic credential নাও
        this.currentLease = await this.vault.getDatabaseCredential(this.role);

        console.log(`[DB Pool] New credential: user=${this.currentLease.username}, ` +
            `expires=${this.currentLease.expiresAt.toISOString()}`);

        // Connection pool recreate
        // (pseudo-code — use pg or mysql2 pool)
        this.pool = {
            host: this.host,
            port: this.port,
            database: this.database,
            user: this.currentLease.username,
            password: this.currentLease.password,
        };
    }

    /**
     * Lease auto-renewal — expire হওয়ার আগে renew
     */
    startLeaseRenewal() {
        // Lease-এর ৭০% সময়ে renew করো
        const renewalInterval = this.currentLease.leaseDuration * 0.7 * 1000;

        this.renewalTimer = setInterval(async () => {
            try {
                if (this.currentLease.renewable) {
                    await this.vault.renewLease(this.currentLease.leaseId);
                    console.log('[DB Pool] Lease renewed successfully');
                } else {
                    await this.refreshCredentials();
                }
            } catch (err) {
                console.error('[DB Pool] Lease renewal failed, getting new credential');
                await this.refreshCredentials();
            }
        }, renewalInterval);
    }

    getPool() {
        return this.pool;
    }

    async shutdown() {
        if (this.renewalTimer) clearInterval(this.renewalTimer);
        if (this.currentLease) {
            await this.vault.revokeLease(this.currentLease.leaseId);
        }
    }
}

/**
 * Secret Leakage Scanner
 */
class SecretLeakageScanner {
    constructor() {
        this.patterns = {
            aws_access_key: /AKIA[0-9A-Z]{16}/g,
            aws_secret_key: /[A-Za-z0-9/+=]{40}/g,
            private_key: /-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----/g,
            generic_password: /(password|passwd|pwd)\s*[:=]\s*['"][^'"]{8,}['"]/gi,
            api_token: /(token|api[_-]?key)\s*[:=]\s*['"][^'"]{16,}['"]/gi,
            connection_string: /mongodb(\+srv)?:\/\/[^:]+:[^@]+@/g,
            jwt: /eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}/g,
        };
    }

    /**
     * File content scan for leaked secrets
     */
    scanContent(content, filename = 'unknown') {
        const findings = [];

        for (const [type, pattern] of Object.entries(this.patterns)) {
            const matches = content.matchAll(pattern);
            for (const match of matches) {
                const line = content.substring(0, match.index).split('\n').length;
                findings.push({
                    type,
                    file: filename,
                    line,
                    snippet: this.maskValue(match[0]),
                    severity: this.getSeverity(type),
                });
            }
        }

        return findings;
    }

    maskValue(value) {
        if (value.length <= 8) return '*'.repeat(value.length);
        return value.slice(0, 4) + '*'.repeat(Math.min(value.length - 8, 20)) + value.slice(-4);
    }

    getSeverity(type) {
        const critical = ['private_key', 'aws_secret_key', 'connection_string'];
        const high = ['aws_access_key', 'api_token', 'jwt'];
        if (critical.includes(type)) return 'CRITICAL';
        if (high.includes(type)) return 'HIGH';
        return 'MEDIUM';
    }
}

// === ব্যবহার ===

// Vault setup
const vault = new VaultClient({
    address: 'https://vault.gp.internal:8200',
    token: process.env.VAULT_TOKEN,
});

// Rotation orchestrator
const rotator = new SecretRotationOrchestrator(vault, {});

// Grameenphone-র scheduled rotation policies
rotator.startScheduledRotation({
    'services/billing/db-password': {
        maxAgeDays: 30,
        generator: () => crypto.randomBytes(24).toString('base64url'),
    },
    'services/sms-api/key': {
        maxAgeDays: 90,
        generator: () => crypto.randomBytes(32).toString('hex'),
    },
    'services/payment-gateway/secret': {
        maxAgeDays: 60,
        generator: () => crypto.randomBytes(32).toString('hex'),
    },
});

// Dynamic DB connection (Billing Service)
const dbPool = new DynamicDatabasePool(vault, {
    role: 'billing-service-rw',
    host: 'postgres.gp.internal',
    port: 5432,
    database: 'billing',
});

await dbPool.initialize();

// Emergency rotation example
// (developer ভুলে secret push করেছে)
await rotator.emergencyRotate(
    'services/payment-gateway/secret',
    'Secret exposed in public GitHub repository'
);
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

| পরিস্থিতি | Strategy |
|-----------|----------|
| Database credentials (multiple services) | Dynamic secrets (Vault) |
| API keys (third-party services) | Dual-secret rotation |
| TLS certificates | Auto-renewal (cert-manager) |
| Encryption keys | Key versioning + gradual migration |
| After employee departure | Emergency rotation |
| Compliance audit preparation | Scheduled rotation + audit trail |
| After security breach suspected | Emergency rotation all affected |

### ❌ কখন ব্যবহার করবেন না / সতর্কতা:

| পরিস্থিতি | কারণ |
|-----------|------|
| Single developer project | Overhead বেশি, simple .env যথেষ্ট |
| Hardcoded secrets in rotation | আগে secret management setup করো |
| Without testing | Rotation break করলে outage হবে |
| Without monitoring | Rotation fail হলে জানা দরকার |
| All secrets একসাথে rotate | Cascading failure-র risk |

### 🎯 Best Practices:

```
Secret Storage:
├── ❌ Code-এ hardcode করবেন না
├── ❌ .env file Git-এ commit করবেন না
├── ❌ Plain text-এ store করবেন না
├── ✅ Vault/Secret Manager ব্যবহার করুন
├── ✅ Environment variable দিয়ে inject করুন
└── ✅ Mounted volume (tmpfs) ব্যবহার করুন

Rotation:
├── ✅ Always test rotation in staging first
├── ✅ Overlap period রাখুন (dual-secret)
├── ✅ Automated rotation setup করুন
├── ✅ Monitoring + alerting রাখুন
├── ✅ Audit log সবসময় রাখুন
├── ✅ Emergency rotation procedure document করুন
└── ✅ Rotation failure → auto-alert → manual intervention
```

### 🔑 Secret Versioning Strategy:

```
Version  State     Purpose
──────── ───────── ─────────────────────────────
v1       revoked   আর ব্যবহারযোগ্য নয়
v2       revoked   আর ব্যবহারযোগ্য নয়
v3       secondary পুরানো client-দের জন্য (grace period)
v4       primary   বর্তমান active secret
v5       pending   পরবর্তী rotation-এ activate হবে
```

---

## 📚 সারসংক্ষেপ

```
Secrets Rotation = নিয়মিত তালা বদলানো 🔑

মনে রাখুন:
├── Dynamic Secrets → সবচেয়ে secure (temporary, unique)
├── Dual-Secret → zero-downtime rotation guarantee
├── Emergency Rotation → breach detect হলে তাৎক্ষণিক
├── Audit Trail → কে, কখন, কোন secret — সব record
├── Automation → manual rotation error-prone
└── Detection → leak হলে দ্রুত জানা দরকার

Tools: HashiCorp Vault > AWS Secrets Manager > Azure Key Vault
Pattern: Sidecar injection > Mounted volume > Environment variable
```
