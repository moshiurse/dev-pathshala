# 📘 Secrets Management

> **Secrets Management — সংবেদনশীল তথ্য (credentials, API keys, certificates) নিরাপদে সংরক্ষণ, বিতরণ ও rotation-এর কেন্দ্রীয় ব্যবস্থাপনা।**

---

## 📖 সংজ্ঞা ও মূল ধারণা

Secrets Management হলো অ্যাপ্লিকেশন ও ইনফ্রাস্ট্রাকচারের **সংবেদনশীল তথ্য** (passwords, API keys, tokens, certificates, encryption keys) **নিরাপদে সংরক্ষণ, অ্যাক্সেস নিয়ন্ত্রণ ও স্বয়ংক্রিয় rotation** করার পদ্ধতি।

### কেন দরকার?

```
❌ সাধারণ ভুল পদ্ধতিগুলো:

1. কোডে হার্ডকোড:     DB_PASS = "mypassword123"     ← Git history-তে চিরকাল থাকবে
2. .env ফাইলে:         .env ফাইল ভুলে commit হয়      ← GitHub-এ public হয়ে যায়
3. Environment Variable: CI/CD-তে plain text          ← Log-এ expose হতে পারে
4. Config ফাইলে:       config.php-তে credentials     ← সার্ভার হ্যাক হলেই লিক

✅ সঠিক পদ্ধতি:
   Centralized Secrets Manager → Encrypted Storage → Access Control → Auto Rotation
```

### জনপ্রিয় Secrets Management টুলস

| টুল | ধরন | বৈশিষ্ট্য |
|-----|------|-----------|
| **HashiCorp Vault** | Self-hosted / Cloud | Dynamic secrets, PKI, transit encryption |
| **AWS Secrets Manager** | Cloud-managed | Auto rotation, RDS integration |
| **Azure Key Vault** | Cloud-managed | HSM-backed, Azure-native |
| **Google Secret Manager** | Cloud-managed | IAM-integrated, versioning |
| **Kubernetes Secrets** | Built-in | Base64 encoded (encrypted at rest সহ) |
| **SOPS** | File-based | Git-friendly encrypted files |

### Secrets Management আর্কিটেকচার

```
┌─────────────────────── Secrets Manager (Vault) ──────────────────────┐
│                                                                       │
│  ┌─────────────┐  ┌──────────────────┐  ┌─────────────────────────┐  │
│  │ Auth Backend│  │ Secrets Engine   │  │ Audit Log               │  │
│  │             │  │                  │  │                         │  │
│  │ • AppRole   │  │ • KV (key/value) │  │ • কে কখন কোন secret   │  │
│  │ • Kubernetes│  │ • Database       │  │   অ্যাক্সেস করেছে     │  │
│  │ • LDAP      │  │ • PKI            │  │ • সকল operation logged │  │
│  │ • Token     │  │ • Transit        │  │                         │  │
│  └──────┬──────┘  └────────┬─────────┘  └─────────────────────────┘  │
│         │                  │                                          │
│         │     ACL Policy   │                                          │
│         └──────────────────┘                                          │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
   ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
   │ PHP Service  │ │ Node Service │  │ Go Service   │
   │ (Vault SDK)  │ │ (Vault SDK)  │  │ (Vault SDK)  │
   └──────────────┘ └──────────────┘  └──────────────┘
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 🏦 ব্যাংকের ভল্ট সিস্টেম

একটি ব্যাংকের নিরাপত্তা ব্যবস্থা চিন্তা করুন:

- **ব্যাংক ভল্ট = HashiCorp Vault**: সকল মূল্যবান জিনিস (secrets) এখানে সুরক্ষিত থাকে
- **লকার চাবি ও বায়োমেট্রিক = Authentication**: শুধু অনুমোদিত ব্যক্তিরা ভল্টে ঢুকতে পারে
- **বিভিন্ন লকার = Secrets Engines**: ভিন্ন ভিন্ন ধরনের secret আলাদা আলাদা engine-এ থাকে
- **CCTV ও লগ বই = Audit Trail**: কে কখন কোন লকার খুলেছে সব রেকর্ড থাকে
- **চাবি বদলানো = Secret Rotation**: নির্দিষ্ট সময় পর পর লকার কোড বদলানো হয়

কেউ চাইলেই ভল্টে ঢুকতে পারে না — আগে **পরিচয় যাচাই** হবে, তারপর **অনুমতি চেক** হবে, তারপর শুধু নির্দিষ্ট লকারেই অ্যাক্সেস পাবে।

---

## 🔧 Environment Variables vs Secrets Manager তুলনা

```
Environment Variables:
┌────────────────────────────────────┐
│  ❌ Plain text-এ থাকে             │
│  ❌ Rotation কঠিন                 │
│  ❌ Audit trail নেই               │
│  ❌ Access control সীমিত          │
│  ❌ সকল env var সকল process দেখে │
│  ✅ সহজ ও দ্রুত সেটআপ            │
└────────────────────────────────────┘

Secrets Manager (Vault):
┌────────────────────────────────────┐
│  ✅ Encrypted at rest ও in transit│
│  ✅ Auto rotation সম্ভব           │
│  ✅ সম্পূর্ণ audit trail          │
│  ✅ Fine-grained access control   │
│  ✅ Dynamic secrets সম্ভব         │
│  ❌ অতিরিক্ত infrastructure      │
└────────────────────────────────────┘
```

---

## 🐘 PHP উদাহরণ — Vault Client Integration

```php
<?php
/**
 * HashiCorp Vault থেকে secrets পড়ার জন্য PHP client।
 * Production-এ সকল sensitive configuration Vault থেকে আসবে।
 */

class VaultClient
{
    private string $vaultAddr;
    private string $token;
    private array $secretCache = [];

    public function __construct(?string $vaultAddr = null, ?string $token = null)
    {
        $this->vaultAddr = $vaultAddr ?? getenv('VAULT_ADDR') ?: 'http://vault:8200';
        $this->token = $token ?? $this->authenticateWithAppRole();
    }

    /**
     * AppRole authentication — Kubernetes বা CI/CD পরিবেশে ব্যবহৃত হয়।
     * Role ID ও Secret ID দিয়ে Vault token পাওয়া যায়।
     */
    private function authenticateWithAppRole(): string
    {
        $roleId = getenv('VAULT_ROLE_ID');
        $secretId = getenv('VAULT_SECRET_ID');

        if (!$roleId || !$secretId) {
            throw new RuntimeException('VAULT_ROLE_ID এবং VAULT_SECRET_ID সেট করা আবশ্যক');
        }

        $response = $this->httpRequest('POST', '/v1/auth/approle/login', [
            'role_id' => $roleId,
            'secret_id' => $secretId,
        ]);

        if (!isset($response['auth']['client_token'])) {
            throw new RuntimeException('Vault authentication ব্যর্থ হয়েছে');
        }

        return $response['auth']['client_token'];
    }

    /**
     * KV v2 secrets engine থেকে secret পড়ে।
     * ফলাফল cache করে যাতে বারবার Vault call না হয়।
     */
    public function getSecret(string $path, string $key): string
    {
        $cacheKey = "{$path}:{$key}";

        if (isset($this->secretCache[$cacheKey])) {
            return $this->secretCache[$cacheKey];
        }

        $response = $this->httpRequest('GET', "/v1/secret/data/{$path}");

        if (!isset($response['data']['data'][$key])) {
            throw new RuntimeException("Secret '{$key}' পাওয়া যায়নি path '{$path}'-এ");
        }

        $value = $response['data']['data'][$key];
        $this->secretCache[$cacheKey] = $value;
        return $value;
    }

    /**
     * Dynamic database credentials তৈরি করে।
     * প্রতিবার নতুন username/password পাওয়া যায় যা নির্দিষ্ট সময় পর expire হয়।
     */
    public function getDatabaseCredentials(string $roleName): array
    {
        $response = $this->httpRequest('GET', "/v1/database/creds/{$roleName}");

        return [
            'username' => $response['data']['username'],
            'password' => $response['data']['password'],
            'lease_id' => $response['lease_id'],
            'lease_duration' => $response['lease_duration'],
        ];
    }

    private function httpRequest(string $method, string $endpoint, ?array $data = null): array
    {
        $ch = curl_init("{$this->vaultAddr}{$endpoint}");
        $headers = ['Content-Type: application/json'];

        if ($this->token) {
            $headers[] = "X-Vault-Token: {$this->token}";
        }

        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 10,
        ]);

        if ($data !== null) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new RuntimeException("Vault API error: HTTP {$httpCode}");
        }

        return json_decode($response, true);
    }
}

/**
 * ব্যবহার — Database connection-এ Vault থেকে credentials আনা:
 */
class DatabaseFactory
{
    private VaultClient $vault;

    public function __construct(VaultClient $vault)
    {
        $this->vault = $vault;
    }

    /**
     * Dynamic credentials দিয়ে PDO connection তৈরি।
     * Credentials নির্দিষ্ট সময় পর expire হয় — connection pool refresh করতে হবে।
     */
    public function createConnection(): PDO
    {
        $creds = $this->vault->getDatabaseCredentials('php-app-role');
        $host = $this->vault->getSecret('config/database', 'host');
        $dbName = $this->vault->getSecret('config/database', 'name');

        $dsn = "mysql:host={$host};dbname={$dbName};charset=utf8mb4";

        return new PDO($dsn, $creds['username'], $creds['password'], [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        ]);
    }
}

// ব্যবহার:
$vault = new VaultClient();
$dbFactory = new DatabaseFactory($vault);
$pdo = $dbFactory->createConnection();
```

---

## 🟨 JavaScript উদাহরণ — AWS Secrets Manager ও Vault Integration

```javascript
/**
 * Node.js অ্যাপ্লিকেশনে Secrets Management।
 * AWS Secrets Manager ও HashiCorp Vault — দুটোর integration দেখানো হয়েছে।
 */

const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');
const axios = require('axios');

// ==================== AWS Secrets Manager ====================

class AWSSecretsProvider {
  constructor(region = 'ap-southeast-1') {
    this.client = new SecretsManagerClient({ region });
    this.cache = new Map();
    this.cacheTTL = 5 * 60 * 1000; // ৫ মিনিট cache
  }

  /**
   * AWS Secrets Manager থেকে secret পড়ে।
   * TTL-ভিত্তিক caching আছে যাতে API call কমে।
   */
  async getSecret(secretName) {
    const cached = this.cache.get(secretName);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }

    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await this.client.send(command);

    let secretValue;
    if (response.SecretString) {
      secretValue = JSON.parse(response.SecretString);
    } else {
      const buff = Buffer.from(response.SecretBinary, 'base64');
      secretValue = JSON.parse(buff.toString('utf8'));
    }

    this.cache.set(secretName, { value: secretValue, timestamp: Date.now() });
    return secretValue;
  }

  async getDatabaseConfig() {
    const secret = await this.getSecret('production/database/credentials');
    return {
      host: secret.host,
      port: secret.port || 3306,
      database: secret.dbname,
      user: secret.username,
      password: secret.password,
      ssl: { rejectUnauthorized: true },
    };
  }
}

// ==================== HashiCorp Vault ====================

class VaultSecretsProvider {
  constructor(config = {}) {
    this.vaultAddr = config.vaultAddr || process.env.VAULT_ADDR || 'http://vault:8200';
    this.token = null;
    this.cache = new Map();
  }

  /**
   * Kubernetes Service Account দিয়ে Vault-এ authenticate।
   * Pod-এর SA token স্বয়ংক্রিয়ভাবে mount হয়।
   */
  async authenticateWithKubernetes() {
    const fs = require('fs');
    const jwt = fs.readFileSync(
      '/var/run/secrets/kubernetes.io/serviceaccount/token', 'utf8'
    );

    const response = await axios.post(`${this.vaultAddr}/v1/auth/kubernetes/login`, {
      role: process.env.VAULT_ROLE || 'app-role',
      jwt: jwt,
    });

    this.token = response.data.auth.client_token;
    this.tokenExpiry = Date.now() + (response.data.auth.lease_duration * 1000);
    return this.token;
  }

  async ensureAuthenticated() {
    if (!this.token || Date.now() >= this.tokenExpiry - 60000) {
      await this.authenticateWithKubernetes();
    }
  }

  async getSecret(path, key) {
    await this.ensureAuthenticated();

    const cacheKey = `${path}:${key}`;
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < 300000) {
      return cached.value;
    }

    const response = await axios.get(`${this.vaultAddr}/v1/secret/data/${path}`, {
      headers: { 'X-Vault-Token': this.token },
    });

    const value = response.data.data.data[key];
    this.cache.set(cacheKey, { value, timestamp: Date.now() });
    return value;
  }

  /**
   * Dynamic database credentials — প্রতিবার নতুন credentials পাওয়া যায়।
   */
  async getDynamicDatabaseCreds(roleName) {
    await this.ensureAuthenticated();

    const response = await axios.get(
      `${this.vaultAddr}/v1/database/creds/${roleName}`,
      { headers: { 'X-Vault-Token': this.token } }
    );

    return {
      username: response.data.data.username,
      password: response.data.data.password,
      leaseDuration: response.data.lease_duration,
      leaseId: response.data.lease_id,
    };
  }
}

// ==================== Unified Secrets Factory ====================

class SecretsFactory {
  static create(provider = process.env.SECRETS_PROVIDER || 'vault') {
    switch (provider) {
      case 'aws':
        return new AWSSecretsProvider();
      case 'vault':
        return new VaultSecretsProvider();
      default:
        throw new Error(`অজানা secrets provider: ${provider}`);
    }
  }
}

// ==================== ব্যবহার ====================

async function initializeApp() {
  const secrets = SecretsFactory.create();
  const dbConfig = await secrets.getDatabaseConfig?.()
    || { host: await secrets.getSecret('config/db', 'host') };

  console.log('✅ Database config loaded from secrets manager');
  console.log(`📌 Host: ${dbConfig.host}`);

  return dbConfig;
}

module.exports = { AWSSecretsProvider, VaultSecretsProvider, SecretsFactory };
```

---

## 🔒 Security Best Practices

### করণীয় ✅

1. **Secrets কখনো Git-এ commit করবেন না** — `.gitignore`-এ `.env`, `*.key`, `*.pem` যোগ করুন
2. **Dynamic secrets ব্যবহার করুন** — static password-এর বদলে auto-generated, time-limited credentials
3. **Least privilege principle** মানুন — প্রতিটি সার্ভিস শুধু তার প্রয়োজনীয় secrets পাবে
4. **Auto rotation সেটআপ করুন** — নিয়মিত বিরতিতে credentials বদলানো হবে
5. **Audit logging চালু রাখুন** — কে কখন কোন secret অ্যাক্সেস করেছে
6. **Encryption at rest ও in transit** নিশ্চিত করুন
7. **Secret versioning** ব্যবহার করুন — রোলব্যাক করা সহজ হয়

### বর্জনীয় ❌

1. **Environment variable-এ sensitive data রাখবেন না** production-এ — process listing-এ দেখা যেতে পারে
2. **Shared credentials ব্যবহার করবেন না** — প্রতিটি সার্ভিস/ব্যক্তির আলাদা credentials থাকবে
3. **Log-এ secrets print করবেন না** — `console.log(config)` করলে password দেখা যেতে পারে
4. **Long-lived tokens ব্যবহার এড়িয়ে চলুন** — short TTL ও renewal ব্যবহার করুন
5. **Client-side-এ secrets রাখবেন না** — browser/mobile app-এ secrets embed করবেন না

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|-----------|------|
| **Production পরিবেশ** | সকল sensitive data কেন্দ্রীয়ভাবে পরিচালিত হবে |
| **Multi-service architecture** | প্রতিটি সার্ভিসের credentials আলাদাভাবে ম্যানেজ করা যায় |
| **Compliance requirement** (PCI-DSS, HIPAA, SOC2) | Audit trail ও access control বাধ্যতামূলক |
| **Team-এ একাধিক ডেভেলপার** | কে কোন secret অ্যাক্সেস করতে পারবে তা নিয়ন্ত্রণ |
| **Database credentials rotation** দরকার | Dynamic secrets দিয়ে স্বয়ংক্রিয় rotation |
| **Multi-cloud বা hybrid setup** | একই Vault সকল cloud-এর secrets পরিচালনা করে |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| **ছোট personal প্রজেক্ট** | `.env` ফাইল ও environment variables যথেষ্ট |
| **Non-sensitive configuration** | App name, log level ইত্যাদি ConfigMap/env-এ রাখুন |
| **Development environment** | লোকাল ডেভে .env ফাইল ব্যবহার করুন |
| **Budget constraint** | Vault চালাতে অতিরিক্ত infrastructure লাগে |
| **একক সার্ভার অ্যাপ** যেখানে কোনো compliance নেই | ওভারহেড justify হয় না |

---

## 🔑 মূল শিক্ষা

1. **Secrets ≠ Configuration** — sensitive তথ্য আর সাধারণ config আলাদাভাবে ম্যানেজ করুন
2. **Dynamic secrets সবচেয়ে নিরাপদ** — প্রতিটি credentials short-lived ও unique
3. **Zero-trust approach** নিন — কোনো সার্ভিসকে বিশ্বাস করবেন না, সবসময় verify করুন
4. **Secret sprawl এড়ান** — সকল secrets এক জায়গায় রাখুন, বিভিন্ন জায়গায় ছড়িয়ে রাখবেন না
5. **Rotation plan থাকা বাধ্যতামূলক** — "set and forget" মানসিকতা security breach ডেকে আনে
