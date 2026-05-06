# 🔐 Workload Identity & mTLS (Mutual TLS)

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [আর্কিটেকচার ডায়াগ্রাম](#আর্কিটেকচার-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

### Workload Identity কী?

**Workload Identity** হলো একটি **non-human identity** যা সার্ভিস বা অ্যাপ্লিকেশনকে চিহ্নিত করে। মানুষের যেমন NID/পাসপোর্ট আছে, তেমনি সার্ভিসেরও একটি cryptographic identity থাকে যা দিয়ে সে নিজেকে প্রমাণ করে।

### মূল ধারণাসমূহ:

| ধারণা | ব্যাখ্যা |
|-------|----------|
| **Workload Identity** | সার্ভিসের জন্য unique cryptographic identity |
| **mTLS** | Client ও Server উভয়ই certificate দেখায় |
| **SPIFFE** | Secure Production Identity Framework for Everyone |
| **SPIRE** | SPIFFE Runtime Environment (identity issuer) |
| **Service Mesh** | Infrastructure layer যা mTLS auto-manage করে |
| **Zero Trust** | "Never trust, always verify" নীতি |

### Regular TLS vs mTLS:

```
Regular TLS (একমুখী):
- শুধু Server certificate দেখায়
- Client verify করে যে সে সঠিক server-এ connected
- Client-কে কেউ verify করে না

mTLS (দ্বিমুখী):
- Server certificate দেখায় → Client verify করে
- Client certificate দেখায় → Server verify করে
- উভয় পক্ষই নিশ্চিত যে অপর পক্ষ বৈধ
```

### SPIFFE Identity:

SPIFFE ID হলো একটি URI format identity:
```
spiffe://trust-domain/workload-identifier
spiffe://bkash.com/payment-service
spiffe://bkash.com/bank-gateway
```

### Zero Trust Principles:
1. **কাউকে বিশ্বাস করো না** — network location দিয়ে trust নির্ধারণ হয় না
2. **সবসময় verify করো** — প্রতিটি request-এ identity check
3. **Least privilege** — minimum necessary access
4. **Assume breach** — ধরে নাও network compromise হয়েছে

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏦 bKash Payment Service ↔ Bank Gateway mTLS

কল্পনা করুন bKash-এর payment service যখন Bangladesh Bank-এর gateway-তে connect করে:

```
সমস্যা:
- bKash-এর ১০০+ microservice আছে
- শুধুমাত্র "payment-service" ই bank gateway-তে call করতে পারবে
- কোনো rogue service যেন bank-এ access না পায়
- Network-এ কেউ traffic intercept করতে পারবে না

সমাধান: mTLS
- payment-service-এর নিজস্ব certificate আছে
- bank-gateway শুধু valid certificate-ওয়ালা client-কে accept করে
- certificate প্রতি ২৪ ঘণ্টায় auto-rotate হয়
```

### 🚗 Pathao-র Service Mesh উদাহরণ:

```
Pathao-র microservices:
├── rider-service      (spiffe://pathao.com/rider)
├── driver-service     (spiffe://pathao.com/driver)
├── payment-service    (spiffe://pathao.com/payment)
├── notification-svc   (spiffe://pathao.com/notification)
└── tracking-service   (spiffe://pathao.com/tracking)

নিয়ম:
- rider-service → payment-service ✅ (ride payment)
- driver-service → payment-service ✅ (earnings withdrawal)
- notification-svc → payment-service ❌ (কেন দরকার?)
- tracking-service → payment-service ❌ (কেন দরকার?)
```

### 📱 Grameenphone-র Kubernetes Service Accounts:

```
Namespace: gp-billing
├── Pod: invoice-generator
│   └── ServiceAccount: sa-invoice-gen
│       └── Permissions: read invoices, write PDF storage
│
├── Pod: payment-processor  
│   └── ServiceAccount: sa-payment-proc
│       └── Permissions: read/write transactions, call bank API
│
└── Pod: report-service
    └── ServiceAccount: sa-reports
        └── Permissions: read-only all billing data
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### mTLS Handshake Process:

```
┌─────────────────┐                          ┌─────────────────┐
│  bKash Payment  │                          │  Bank Gateway   │
│    Service      │                          │    Service      │
└────────┬────────┘                          └────────┬────────┘
         │                                            │
         │  1. ClientHello                            │
         │───────────────────────────────────────────>│
         │                                            │
         │  2. ServerHello + Server Certificate       │
         │<───────────────────────────────────────────│
         │                                            │
         │  3. Client verifies Server cert            │
         │  (Is this really Bank Gateway?)            │
         │                                            │
         │  4. Client Certificate                     │
         │───────────────────────────────────────────>│
         │                                            │
         │  5. Server verifies Client cert            │
         │  (Is this really bKash Payment?)           │
         │                                            │
         │  6. Encrypted Channel Established ✅       │
         │<══════════════════════════════════════════>│
         │                                            │
         │  7. Transfer funds (encrypted)             │
         │═══════════════════════════════════════════>│
         │                                            │
```

### SPIFFE/SPIRE Architecture:

```
┌─────────────────────────────────────────────────────┐
│                  SPIRE Server                         │
│  ┌───────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ CA (Root) │  │ Registration │  │  Datastore  │  │
│  │Certificate│  │   Entries    │  │  (Identity) │  │
│  └───────────┘  └──────────────┘  └─────────────┘  │
└──────────────────────┬──────────────────────────────┘
                       │ Issue SVIDs
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐┌──────────────┐┌──────────────┐
│ SPIRE Agent  ││ SPIRE Agent  ││ SPIRE Agent  │
│   (Node 1)   ││   (Node 2)   ││   (Node 3)   │
└──────┬───────┘└──────┬───────┘└──────┬───────┘
       │               │               │
  ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
  │ Workload│    │ Workload│    │ Workload│
  │(Payment)│    │ (User)  │    │ (Order) │
  │  SVID   │    │  SVID   │    │  SVID   │
  └─────────┘    └─────────┘    └─────────┘

SVID = SPIFFE Verifiable Identity Document
```

### Service Mesh Auto-mTLS (Istio):

```
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                   │
│                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    │
│  │   Pod A          │    │   Pod B          │    │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ │    │
│  │ │  App Container│ │    │ │  App Container│ │    │
│  │ │  (plaintext) │ │    │ │  (plaintext) │ │    │
│  │ └──────┬───────┘ │    │ └──────▲───────┘ │    │
│  │        │ localhost│    │        │ localhost│    │
│  │ ┌──────▼───────┐ │    │ ┌──────┴───────┐ │    │
│  │ │ Envoy Proxy  │ │    │ │ Envoy Proxy  │ │    │
│  │ │ (sidecar)    │ │    │ │ (sidecar)    │ │    │
│  │ │ + TLS cert   │ │    │ │ + TLS cert   │ │    │
│  │ └──────┬───────┘ │    │ └──────▲───────┘ │    │
│  └────────┼─────────┘    └────────┼─────────┘    │
│           │      mTLS encrypted    │              │
│           └────────────────────────┘              │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │         Istio Control Plane              │     │
│  │  (Certificate Authority + Config)        │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

### Certificate Rotation Timeline:

```
Certificate Lifecycle (Short-lived):

Time ──────────────────────────────────────────────────>

Cert v1: ████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░
         │← Active ─────→│← Grace period              │
         Issue            │                            │
                          │                            │
Cert v2: ░░░░░░░░░░░░░░░████████████████░░░░░░░░░░░░░
                         │← Active ──────→│            │
                         Issue            │            │
                                          │            │
Cert v3: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████████
                                         │← Active ──→│
                                         Issue

Overlap period ensures zero-downtime rotation!
```

---

## 💻 PHP কোড উদাহরণ

### mTLS Client (bKash Payment Service):

```php
<?php

namespace BKash\Security\MTLS;

/**
 * mTLS Client - Bank Gateway-তে secure connection
 */
class MTLSClient
{
    private string $clientCertPath;
    private string $clientKeyPath;
    private string $caCertPath;
    private string $serviceIdentity;

    public function __construct(
        string $clientCertPath,
        string $clientKeyPath,
        string $caCertPath,
        string $serviceIdentity
    ) {
        $this->clientCertPath = $clientCertPath;
        $this->clientKeyPath = $clientKeyPath;
        $this->caCertPath = $caCertPath;
        $this->serviceIdentity = $serviceIdentity;
    }

    /**
     * mTLS সহ HTTP request পাঠানো
     */
    public function sendRequest(string $url, array $data): array
    {
        $ch = curl_init($url);

        // Client certificate (আমি কে — bKash Payment Service)
        curl_setopt($ch, CURLOPT_SSLCERT, $this->clientCertPath);
        curl_setopt($ch, CURLOPT_SSLKEY, $this->clientKeyPath);

        // CA certificate (server-কে verify করতে)
        curl_setopt($ch, CURLOPT_CAINFO, $this->caCertPath);

        // Server certificate verify ON (must!)
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);

        // Request data
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Content-Type: application/json',
            'X-Service-Identity: ' . $this->serviceIdentity,
        ]);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \RuntimeException("mTLS connection failed: {$error}");
        }

        return [
            'status' => $httpCode,
            'body' => json_decode($response, true),
        ];
    }
}

/**
 * Certificate Manager — auto rotation পরিচালনা
 */
class CertificateManager
{
    private string $spireSocketPath;
    private string $certStorePath;
    private int $rotationIntervalHours;

    public function __construct(
        string $spireSocketPath = '/run/spire/sockets/agent.sock',
        string $certStorePath = '/var/run/secrets/certs',
        int $rotationIntervalHours = 24
    ) {
        $this->spireSocketPath = $spireSocketPath;
        $this->certStorePath = $certStorePath;
        $this->rotationIntervalHours = $rotationIntervalHours;
    }

    /**
     * SPIRE থেকে নতুন SVID (certificate) সংগ্রহ
     */
    public function fetchSVID(): WorkloadIdentity
    {
        // SPIRE Workload API call
        $svid = $this->callSpireAPI('/v1/x509-svid');

        return new WorkloadIdentity(
            spiffeId: $svid['spiffe_id'],
            certificate: $svid['x509_svid'],
            privateKey: $svid['x509_svid_key'],
            bundle: $svid['bundle'],
            expiresAt: new \DateTimeImmutable($svid['expires_at'])
        );
    }

    /**
     * Certificate expire হওয়ার আগে rotate করো
     */
    public function shouldRotate(WorkloadIdentity $identity): bool
    {
        $now = new \DateTimeImmutable();
        $expiresAt = $identity->expiresAt;
        $lifetime = $expiresAt->getTimestamp() - $now->getTimestamp();

        // ৫০% lifetime পার হলে rotate করো
        $halfLife = $lifetime / 2;
        $elapsed = $now->getTimestamp() - ($expiresAt->getTimestamp() - $lifetime);

        return $elapsed >= $halfLife;
    }

    /**
     * Certificate rotation loop
     */
    public function startRotationLoop(): void
    {
        while (true) {
            $identity = $this->fetchSVID();
            $this->storeCertificates($identity);

            // পরবর্তী check — half-life এ
            $sleepSeconds = $this->calculateSleepTime($identity);
            sleep($sleepSeconds);
        }
    }

    private function storeCertificates(WorkloadIdentity $identity): void
    {
        file_put_contents(
            $this->certStorePath . '/svid.pem',
            $identity->certificate
        );
        file_put_contents(
            $this->certStorePath . '/svid-key.pem',
            $identity->privateKey
        );
        file_put_contents(
            $this->certStorePath . '/bundle.pem',
            $identity->bundle
        );

        // File permissions — only service user can read
        chmod($this->certStorePath . '/svid-key.pem', 0600);
    }

    private function calculateSleepTime(WorkloadIdentity $identity): int
    {
        $now = new \DateTimeImmutable();
        $remaining = $identity->expiresAt->getTimestamp() - $now->getTimestamp();
        return (int)($remaining * 0.5); // half-life এ আবার check
    }

    private function callSpireAPI(string $endpoint): array
    {
        // Unix domain socket দিয়ে SPIRE Agent-এ call
        $ch = curl_init("http://localhost{$endpoint}");
        curl_setopt($ch, CURLOPT_UNIX_SOCKET_PATH, $this->spireSocketPath);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}

/**
 * Workload Identity data class
 */
class WorkloadIdentity
{
    public function __construct(
        public readonly string $spiffeId,
        public readonly string $certificate,
        public readonly string $privateKey,
        public readonly string $bundle,
        public readonly \DateTimeImmutable $expiresAt
    ) {}

    public function isExpired(): bool
    {
        return new \DateTimeImmutable() >= $this->expiresAt;
    }

    public function getRemainingLifetime(): int
    {
        $now = new \DateTimeImmutable();
        return max(0, $this->expiresAt->getTimestamp() - $now->getTimestamp());
    }
}

/**
 * mTLS Server Middleware — incoming request verify করো
 */
class MTLSVerificationMiddleware
{
    private array $allowedIdentities;
    private string $trustedCAPath;

    public function __construct(array $allowedIdentities, string $trustedCAPath)
    {
        $this->allowedIdentities = $allowedIdentities;
        $this->trustedCAPath = $trustedCAPath;
    }

    /**
     * Client certificate verify করো
     */
    public function verify(array $serverVars): bool
    {
        // Nginx/Apache client cert info pass করে
        $clientCert = $serverVars['SSL_CLIENT_CERT'] ?? null;
        $verifyStatus = $serverVars['SSL_CLIENT_VERIFY'] ?? 'NONE';

        if ($verifyStatus !== 'SUCCESS' || !$clientCert) {
            throw new \RuntimeException('Client certificate verification failed');
        }

        // Certificate থেকে SPIFFE ID extract করো
        $certInfo = openssl_x509_parse($clientCert);
        $spiffeId = $this->extractSpiffeId($certInfo);

        // Allowed list-এ আছে কিনা check করো
        if (!in_array($spiffeId, $this->allowedIdentities)) {
            throw new \RuntimeException(
                "Service '{$spiffeId}' is not authorized to access this endpoint"
            );
        }

        return true;
    }

    private function extractSpiffeId(array $certInfo): string
    {
        // SAN (Subject Alternative Name) থেকে SPIFFE URI extract
        $san = $certInfo['extensions']['subjectAltName'] ?? '';
        if (preg_match('/URI:spiffe:\/\/[^,]+/', $san, $matches)) {
            return str_replace('URI:', '', $matches[0]);
        }
        throw new \RuntimeException('No SPIFFE ID found in certificate');
    }
}

// === ব্যবহারের উদাহরণ ===

// bKash Payment → Bank Gateway mTLS call
$client = new MTLSClient(
    clientCertPath: '/var/run/secrets/certs/svid.pem',
    clientKeyPath: '/var/run/secrets/certs/svid-key.pem',
    caCertPath: '/var/run/secrets/certs/bundle.pem',
    serviceIdentity: 'spiffe://bkash.com/payment-service'
);

$response = $client->sendRequest(
    'https://bank-gateway.internal:8443/api/transfer',
    [
        'from_account' => 'BKASH_SETTLEMENT',
        'to_account' => 'USER_12345',
        'amount' => 5000.00,
        'currency' => 'BDT',
    ]
);

// Bank Gateway-এ incoming request verify
$middleware = new MTLSVerificationMiddleware(
    allowedIdentities: [
        'spiffe://bkash.com/payment-service',
        'spiffe://bkash.com/settlement-service',
    ],
    trustedCAPath: '/etc/ssl/trusted-ca/bkash-root.pem'
);

$middleware->verify($_SERVER);
```

---

## 🟨 JavaScript কোড উদাহরণ

### mTLS Client & Server (Node.js):

```javascript
const tls = require('tls');
const https = require('https');
const fs = require('fs');
const { X509Certificate } = require('crypto');

/**
 * mTLS Client — Service-to-Service secure communication
 * bKash Payment Service → Bank Gateway
 */
class MTLSClient {
    constructor(config) {
        this.certPath = config.certPath;
        this.keyPath = config.keyPath;
        this.caPath = config.caPath;
        this.serviceIdentity = config.serviceIdentity;
        this.currentCert = null;
        this.certWatcher = null;
    }

    /**
     * Certificate hot-reload setup — rotation এ নতুন cert auto-load
     */
    startCertWatcher() {
        this.loadCertificates();

        // Certificate file change detect করো
        this.certWatcher = fs.watch(this.certPath, () => {
            console.log('[mTLS] Certificate rotated, reloading...');
            this.loadCertificates();
        });
    }

    loadCertificates() {
        this.currentCert = {
            cert: fs.readFileSync(this.certPath),
            key: fs.readFileSync(this.keyPath),
            ca: fs.readFileSync(this.caPath),
        };
    }

    /**
     * mTLS সহ HTTPS request
     */
    async request(url, method, data) {
        const urlObj = new URL(url);

        const options = {
            hostname: urlObj.hostname,
            port: urlObj.port || 443,
            path: urlObj.pathname,
            method: method,
            // Client certificate (আমি কে)
            cert: this.currentCert.cert,
            key: this.currentCert.key,
            // Server verify করতে CA
            ca: this.currentCert.ca,
            // Server certificate must be valid
            rejectUnauthorized: true,
            headers: {
                'Content-Type': 'application/json',
                'X-Service-Identity': this.serviceIdentity,
            },
        };

        return new Promise((resolve, reject) => {
            const req = https.request(options, (res) => {
                let body = '';
                res.on('data', chunk => body += chunk);
                res.on('end', () => {
                    resolve({
                        status: res.statusCode,
                        headers: res.headers,
                        body: JSON.parse(body),
                    });
                });
            });

            req.on('error', (err) => {
                reject(new Error(`mTLS request failed: ${err.message}`));
            });

            if (data) {
                req.write(JSON.stringify(data));
            }
            req.end();
        });
    }

    stop() {
        if (this.certWatcher) {
            this.certWatcher.close();
        }
    }
}

/**
 * mTLS Server — শুধু verified client-দের accept করো
 * Bank Gateway Service
 */
class MTLSServer {
    constructor(config) {
        this.port = config.port;
        this.allowedSpiffeIds = config.allowedSpiffeIds;
        this.server = null;

        this.tlsOptions = {
            cert: fs.readFileSync(config.serverCertPath),
            key: fs.readFileSync(config.serverKeyPath),
            ca: fs.readFileSync(config.clientCAPath),
            // Client certificate MUST present করতে হবে
            requestCert: true,
            rejectUnauthorized: true,
        };
    }

    start() {
        this.server = https.createServer(this.tlsOptions, (req, res) => {
            try {
                const clientIdentity = this.verifyClientIdentity(req);
                console.log(`[mTLS] Verified client: ${clientIdentity}`);

                this.handleRequest(req, res, clientIdentity);
            } catch (err) {
                console.error(`[mTLS] Auth failed: ${err.message}`);
                res.writeHead(403, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'Unauthorized service' }));
            }
        });

        this.server.listen(this.port, () => {
            console.log(`[Bank Gateway] mTLS server listening on port ${this.port}`);
        });
    }

    /**
     * Client certificate থেকে SPIFFE ID verify করো
     */
    verifyClientIdentity(req) {
        const cert = req.socket.getPeerCertificate();

        if (!cert || !cert.subject) {
            throw new Error('No client certificate presented');
        }

        // SAN থেকে SPIFFE ID extract
        const spiffeId = this.extractSpiffeId(cert);

        if (!this.allowedSpiffeIds.includes(spiffeId)) {
            throw new Error(`Service '${spiffeId}' not in allowed list`);
        }

        // Certificate expiry check
        const notAfter = new Date(cert.valid_to);
        if (notAfter < new Date()) {
            throw new Error('Client certificate has expired');
        }

        return spiffeId;
    }

    extractSpiffeId(cert) {
        // subjectaltname: "URI:spiffe://bkash.com/payment-service"
        const san = cert.subjectaltname || '';
        const match = san.match(/URI:spiffe:\/\/[^,]+/);
        if (!match) {
            throw new Error('No SPIFFE ID in certificate SAN');
        }
        return match[0].replace('URI:', '');
    }

    handleRequest(req, res, clientIdentity) {
        // Route based on verified identity
        let body = '';
        req.on('data', chunk => body += chunk);
        req.on('end', () => {
            const data = JSON.parse(body);
            console.log(`[Bank Gateway] Transfer request from ${clientIdentity}:`, data);

            res.writeHead(200, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({
                status: 'success',
                transactionId: `TXN-${Date.now()}`,
                message: 'Fund transfer initiated',
            }));
        });
    }
}

/**
 * SPIRE Workload API Client
 * SPIRE Agent থেকে identity (SVID) সংগ্রহ
 */
class SpireWorkloadClient {
    constructor(socketPath = '/run/spire/sockets/agent.sock') {
        this.socketPath = socketPath;
    }

    /**
     * X.509 SVID সংগ্রহ করো
     */
    async fetchX509SVID() {
        return new Promise((resolve, reject) => {
            const options = {
                socketPath: this.socketPath,
                path: '/v1/x509-svid',
                method: 'GET',
            };

            const req = https.request(options, (res) => {
                let data = '';
                res.on('data', chunk => data += chunk);
                res.on('end', () => {
                    const svid = JSON.parse(data);
                    resolve({
                        spiffeId: svid.spiffe_id,
                        certificate: Buffer.from(svid.x509_svid, 'base64'),
                        privateKey: Buffer.from(svid.x509_svid_key, 'base64'),
                        bundle: Buffer.from(svid.bundle, 'base64'),
                        expiresAt: new Date(svid.expires_at),
                    });
                });
            });

            req.on('error', reject);
            req.end();
        });
    }

    /**
     * Auto-rotation with half-life renewal
     */
    async startAutoRotation(certDir, onRotation) {
        while (true) {
            const svid = await this.fetchX509SVID();

            // Certificate files save করো
            fs.writeFileSync(`${certDir}/svid.pem`, svid.certificate);
            fs.writeFileSync(`${certDir}/svid-key.pem`, svid.privateKey, { mode: 0o600 });
            fs.writeFileSync(`${certDir}/bundle.pem`, svid.bundle);

            console.log(`[SPIRE] New SVID issued for ${svid.spiffeId}`);
            console.log(`[SPIRE] Expires at: ${svid.expiresAt.toISOString()}`);

            if (onRotation) onRotation(svid);

            // Half-life sleep
            const remainingMs = svid.expiresAt.getTime() - Date.now();
            const sleepMs = remainingMs * 0.5;
            await new Promise(r => setTimeout(r, sleepMs));
        }
    }
}

// === ব্যবহার ===

// bKash Payment Service (Client)
const paymentClient = new MTLSClient({
    certPath: '/var/run/secrets/certs/svid.pem',
    keyPath: '/var/run/secrets/certs/svid-key.pem',
    caPath: '/var/run/secrets/certs/bundle.pem',
    serviceIdentity: 'spiffe://bkash.com/payment-service',
});

paymentClient.startCertWatcher();

// Bank Gateway-তে fund transfer
async function transferFunds() {
    const result = await paymentClient.request(
        'https://bank-gateway.internal:8443/api/transfer',
        'POST',
        {
            from: 'BKASH_POOL',
            to: 'USER_01911XXXXXX',
            amount: 2500,
            currency: 'BDT',
            purpose: 'cash_out',
        }
    );
    console.log('Transfer result:', result.body);
}

// Bank Gateway Server (mTLS enabled)
const bankGateway = new MTLSServer({
    port: 8443,
    serverCertPath: '/etc/certs/server.pem',
    serverKeyPath: '/etc/certs/server-key.pem',
    clientCAPath: '/etc/certs/trusted-clients-ca.pem',
    allowedSpiffeIds: [
        'spiffe://bkash.com/payment-service',
        'spiffe://bkash.com/settlement-service',
    ],
});

bankGateway.start();
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন ব্যবহার করবেন:

| পরিস্থিতি | কারণ |
|-----------|------|
| Service-to-service communication | Internal API calls secure করতে |
| Payment/Financial services | Bank gateway-তে connect করতে |
| Zero-trust environment | Network trust-এ নির্ভর না করে |
| Kubernetes/Cloud-native apps | Pod identity management |
| Compliance requirements (PCI-DSS) | Audit trail ও encryption mandatory |
| Multi-tenant platforms | Tenant isolation নিশ্চিত করতে |

### ❌ কখন ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|------|
| Public-facing APIs (end users) | Users-এর কাছে client cert manage করা অবাস্তব |
| Simple monolithic apps | Overhead বেশি, benefit কম |
| Development/local environment | Self-signed cert-এ complexity বাড়ে |
| Static websites | কোনো service communication নেই |
| Low-security internal tools | Simple API key যথেষ্ট |

### 🎯 Best Practices:

```
1. Short-lived certificates ব্যবহার করুন (24h বা কম)
2. Certificate rotation automate করুন (SPIRE/cert-manager)
3. Private key file permissions 0600 রাখুন
4. Certificate pinning avoid করুন (rotation break করে)
5. Service mesh ব্যবহার করুন (Istio/Linkerd) — app code-এ mTLS logic না রেখে
6. Identity allowlist maintain করুন (কোন service কোথায় call করতে পারবে)
7. Mutual TLS enforce করুন — optional TLS নয়
8. Certificate revocation plan রাখুন (CRL/OCSP)
```

### 🔄 Identity Bootstrapping Challenge:

```
সমস্যা: "প্রথম certificate কীভাবে পাবে?"

সমাধান ১ — Cloud Provider Identity:
  AWS → IAM Role → SPIRE attest → SVID issue
  
সমাধান ২ — Kubernetes Token:
  ServiceAccount Token → SPIRE attest → SVID issue

সমাধান ৩ — Join Token (one-time):
  Admin generates token → Agent uses once → SVID issue
  
সমাধান ৪ — TPM/Hardware attestation:
  Hardware identity → SPIRE attest → SVID issue
```

---

## 📚 সারসংক্ষেপ

```
Workload Identity + mTLS = সার্ভিসের জন্য NID + সুরক্ষিত যোগাযোগ

মনে রাখুন:
├── Identity = "আমি কে?" (SPIFFE ID)
├── Authentication = "প্রমাণ করো" (Certificate/SVID)
├── mTLS = "দুজনেই প্রমাণ করো" (Mutual verification)
├── Rotation = "নিয়মিত পরিবর্তন" (Short-lived certs)
└── Zero Trust = "কাউকে বিশ্বাস করো না, সবাইকে verify করো"
```
