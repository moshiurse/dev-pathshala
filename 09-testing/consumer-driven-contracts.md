# 📜 Consumer-Driven Contract Testing (CDC Testing)

## 📖 সংজ্ঞা ও ধারণা

Consumer-Driven Contract (CDC) Testing হলো এমন একটি testing পদ্ধতি যেখানে **consumer (API ব্যবহারকারী) নির্ধারণ করে provider (API প্রদানকারী) কী response দেবে**। এটি microservices architecture-এ integration testing-এর একটি স্মার্ট বিকল্প।

### 🤔 কেন Integration Testing যথেষ্ট নয়?

```
সমস্যা ১: Microservices-এ E2E test চালাতে সব service একসাথে চালু রাখতে হয়
সমস্যা ২: একটি service change করলে ১০টি integration test ভেঙে যায়
সমস্যা ৩: Test environment maintain করা expensive
সমস্যা ৪: Flaky tests — network issue, timing issue
সমস্যা ৫: Slow feedback — ২০ মিনিট পরে জানা যায় কিছু ভেঙেছে
```

### 📋 CDC Testing কীভাবে কাজ করে

CDC Testing-এ দুটি পক্ষ থাকে:
- **Consumer:** যে service API call করে (যেমন: Mobile App)
- **Provider:** যে service API response দেয় (যেমন: Transaction API)

Consumer তার প্রত্যাশা (expectation) একটি **contract** হিসেবে লিখে রাখে। Provider সেই contract অনুযায়ী তার API verify করে।

---

## 🌍 বাস্তব জীবনের উদাহরণ (বাংলাদেশ প্রসঙ্গ)

### 📱 bKash Mobile App ↔ Transaction API

bKash-এর Mobile App (consumer) যখন Transaction API (provider) call করে:

**Consumer-এর প্রত্যাশা (Contract):**
```
- POST /api/v1/send-money
- Request: { receiver: "01712345678", amount: 500 }
- Response: { status: "success", transactionId: "TXN123", balance: 4500 }
- Status Code: 200
```

যদি Transaction API team এই response structure change করে (যেমন `transactionId` → `txnId`), তাহলে contract test **fail** হবে এবং deploy আটকে যাবে।

### 🛒 Daraz — Multiple Consumers

```
┌──────────────────────────────────────────────────────────┐
│                    DARAZ ECOSYSTEM                         │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────┐                                         │
│  │ Android App │──┐                                      │
│  └─────────────┘  │     ┌─────────────────────┐         │
│                    ├────▶│   Product Catalog   │         │
│  ┌─────────────┐  │     │       API           │         │
│  │   iOS App   │──┤     │   (Provider)        │         │
│  └─────────────┘  │     └─────────────────────┘         │
│                    │                                      │
│  ┌─────────────┐  │     প্রতিটি consumer-এর             │
│  │   Web App   │──┤     আলাদা contract আছে              │
│  └─────────────┘  │                                      │
│                    │                                      │
│  ┌─────────────┐  │                                      │
│  │ Seller App  │──┘                                      │
│  └─────────────┘                                         │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### 📡 Grameenphone — Internal Microservices

Grameenphone-এর Recharge API consume করে:
- MyGP App
- USSD Service
- Third-party recharge partners (bKash, Nagad)

প্রতিটি consumer আলাদা field ব্যবহার করে — contract testing নিশ্চিত করে কেউ ক্ষতিগ্রস্ত না হয়।

---

## 📊 ASCII Diagram — CDC Testing Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│              CONSUMER-DRIVEN CONTRACT TESTING WORKFLOW                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CONSUMER SIDE                           PROVIDER SIDE               │
│  ────────────                            ─────────────              │
│                                                                      │
│  ┌──────────────────┐                                               │
│  │ 1. Consumer test │                                               │
│  │    লিখুন         │                                               │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐                                               │
│  │ 2. Mock Provider │                                               │
│  │    এর বিরুদ্ধে   │                                               │
│  │    test চালান    │                                               │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐          ┌──────────────────┐                 │
│  │ 3. Contract      │─────────▶│ 4. Contract      │                 │
│  │    (Pact) file   │  Broker  │    Broker-এ      │                 │
│  │    generate হয়  │          │    publish হয়    │                 │
│  └──────────────────┘          └────────┬─────────┘                 │
│                                         │                            │
│                                         ▼                            │
│                                ┌──────────────────┐                  │
│                                │ 5. Provider      │                  │
│                                │    contract      │                  │
│                                │    verify করে   │                  │
│                                └────────┬─────────┘                  │
│                                         │                            │
│                              ┌──────────┴──────────┐                 │
│                              ▼                     ▼                  │
│                     ┌─────────────┐       ┌─────────────┐           │
│                     │  PASS ✅    │       │  FAIL ❌    │           │
│                     │  Deploy OK  │       │  Breaking   │           │
│                     │             │       │  Change!    │           │
│                     └─────────────┘       └─────────────┘           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 🔄 Bilateral vs Consumer-Driven Contracts

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  BILATERAL CONTRACT              CONSUMER-DRIVEN CONTRACT     │
│  (দুই পক্ষের সম্মতি)            (Consumer-ই সিদ্ধান্ত নেয়)  │
│                                                               │
│  ┌──────────┐  negotiate  ┌──────────┐                       │
│  │ Consumer │◀───────────▶│ Provider │                       │
│  └──────────┘             └──────────┘                       │
│       ↕                                                       │
│  দুজনেই contract          ┌──────────┐   defines             │
│  define করে               │ Consumer │──────────┐            │
│                            └──────────┘          │            │
│  সমস্যা: কে lead নেবে?                          ▼            │
│  সমস্যা: versioning কঠিন   ┌──────────────────────────┐     │
│                             │  CONTRACT (Consumer-এর   │     │
│                             │  প্রত্যাশা অনুযায়ী)      │     │
│                             └────────────┬─────────────┘     │
│                                          │                    │
│                                          ▼  verifies         │
│                                   ┌──────────┐               │
│                                   │ Provider │               │
│                                   └──────────┘               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Consumer Side — Pact Test (PHP)

```php
<?php

namespace Tests\Contract\Consumer;

use PhpPact\Consumer\InteractionBuilder;
use PhpPact\Consumer\Model\ConsumerRequest;
use PhpPact\Consumer\Model\ProviderResponse;
use PhpPact\Standalone\MockService\MockServerConfig;
use PHPUnit\Framework\TestCase;
use App\Services\BkashTransactionClient;

/**
 * bKash Mobile App (Consumer) → Transaction API (Provider) Contract Test
 */
class BkashTransactionContractTest extends TestCase
{
    private InteractionBuilder $builder;
    private MockServerConfig $config;

    protected function setUp(): void
    {
        // Mock server config — Provider-এর mock
        $this->config = new MockServerConfig();
        $this->config->setHost('localhost');
        $this->config->setPort(7203);
        $this->config->setConsumer('BkashMobileApp');
        $this->config->setProvider('TransactionAPI');

        $this->builder = new InteractionBuilder($this->config);
    }

    /**
     * Test: Send Money — সফল transaction
     */
    public function testSendMoney_Success(): void
    {
        // Consumer-এর প্রত্যাশা define করা
        $request = new ConsumerRequest();
        $request
            ->setMethod('POST')
            ->setPath('/api/v1/send-money')
            ->addHeader('Content-Type', 'application/json')
            ->addHeader('Authorization', 'Bearer valid-token')
            ->setBody([
                'receiver' => '01712345678',
                'amount' => 500,
                'pin' => '12345',
                'reference' => 'Eid Salami',
            ]);

        $response = new ProviderResponse();
        $response
            ->setStatus(200)
            ->addHeader('Content-Type', 'application/json')
            ->setBody([
                'status' => 'success',
                'transactionId' => 'TXN-2024-ABC123',
                'sender' => [
                    'number' => '01812345678',
                    'name' => 'Rahim Uddin',
                    'newBalance' => 4500.00,
                ],
                'receiver' => [
                    'number' => '01712345678',
                    'name' => 'Karim Ahmed',
                ],
                'amount' => 500.00,
                'fee' => 5.00,
                'timestamp' => '2024-04-10T10:30:00Z',
            ]);

        // Interaction define করা
        $this->builder
            ->given('User has sufficient balance') // Provider State
            ->uponReceiving('A request to send money')
            ->with($request)
            ->willRespondWith($response);

        // Consumer code test করা (mock provider-এর বিরুদ্ধে)
        $client = new BkashTransactionClient($this->config->getBaseUri());
        $result = $client->sendMoney('01712345678', 500, '12345', 'Eid Salami');

        // Assertions
        $this->assertEquals('success', $result->status);
        $this->assertNotEmpty($result->transactionId);
        $this->assertEquals(4500.00, $result->sender->newBalance);
        $this->assertEquals(5.00, $result->fee);

        // Contract file generate হবে (pact JSON)
        $this->builder->verify();
    }

    /**
     * Test: Send Money — অপর্যাপ্ত ব্যালেন্স
     */
    public function testSendMoney_InsufficientBalance(): void
    {
        $request = new ConsumerRequest();
        $request
            ->setMethod('POST')
            ->setPath('/api/v1/send-money')
            ->addHeader('Content-Type', 'application/json')
            ->setBody([
                'receiver' => '01712345678',
                'amount' => 50000, // বেশি টাকা
                'pin' => '12345',
            ]);

        $response = new ProviderResponse();
        $response
            ->setStatus(422)
            ->setBody([
                'status' => 'failed',
                'error' => [
                    'code' => 'INSUFFICIENT_BALANCE',
                    'message' => 'আপনার পর্যাপ্ত ব্যালেন্স নেই',
                    'currentBalance' => 5000.00,
                    'requiredAmount' => 50005.00, // amount + fee
                ],
            ]);

        $this->builder
            ->given('User has balance of 5000 BDT')
            ->uponReceiving('A send money request with insufficient balance')
            ->with($request)
            ->willRespondWith($response);

        $client = new BkashTransactionClient($this->config->getBaseUri());
        $result = $client->sendMoney('01712345678', 50000, '12345');

        $this->assertEquals('failed', $result->status);
        $this->assertEquals('INSUFFICIENT_BALANCE', $result->error->code);

        $this->builder->verify();
    }

    /**
     * Test: Transaction History — পেজিনেশন সহ
     */
    public function testGetTransactionHistory_WithPagination(): void
    {
        $request = new ConsumerRequest();
        $request
            ->setMethod('GET')
            ->setPath('/api/v1/transactions')
            ->addQueryParameter('page', '1')
            ->addQueryParameter('limit', '20')
            ->addHeader('Authorization', 'Bearer valid-token');

        $response = new ProviderResponse();
        $response
            ->setStatus(200)
            ->setBody([
                'transactions' => [
                    [
                        'id' => 'TXN-001',
                        'type' => 'send_money',
                        'amount' => 500,
                        'receiver' => '01712345678',
                        'timestamp' => '2024-04-10T10:30:00Z',
                    ],
                ],
                'pagination' => [
                    'currentPage' => 1,
                    'totalPages' => 5,
                    'totalItems' => 98,
                    'hasNext' => true,
                ],
            ]);

        $this->builder
            ->given('User has transaction history')
            ->uponReceiving('A request for transaction history page 1')
            ->with($request)
            ->willRespondWith($response);

        $client = new BkashTransactionClient($this->config->getBaseUri());
        $history = $client->getTransactionHistory(page: 1, limit: 20);

        $this->assertNotEmpty($history->transactions);
        $this->assertTrue($history->pagination->hasNext);

        $this->builder->verify();
    }
}

/**
 * Provider Side — Contract Verification (PHP)
 */
class BkashTransactionProviderVerificationTest extends TestCase
{
    /**
     * Provider State Setup — Consumer-এর given() conditions পূরণ করা
     */
    public function providerStates(): array
    {
        return [
            'User has sufficient balance' => function () {
                // Test database-এ user তৈরি + balance set
                $this->createUser('01812345678', balance: 5000);
            },
            'User has balance of 5000 BDT' => function () {
                $this->createUser('01812345678', balance: 5000);
            },
            'User has transaction history' => function () {
                $this->createUser('01812345678', balance: 5000);
                $this->createTransactions('01812345678', count: 98);
            },
        ];
    }

    /**
     * Pact Verifier চালানো
     */
    public function testProviderSatisfiesConsumerContracts(): void
    {
        $config = new VerifierConfig();
        $config
            ->setProviderName('TransactionAPI')
            ->setProviderBaseUrl('http://localhost:8080')
            ->setBrokerUri('https://pact-broker.bkash.internal')
            ->setPublishResults(true)
            ->setProviderVersion(getenv('GIT_COMMIT_SHA'));

        // Provider states callback register
        $config->setStateChangeUrl('http://localhost:8080/_pact/state');

        $verifier = new Verifier($config);
        $verifier->verify(); // সব consumer contracts verify হবে
    }

    private function createUser(string $phone, float $balance): void
    {
        // Database seeding for test
    }

    private function createTransactions(string $phone, int $count): void
    {
        // Transaction history seed
    }
}

/**
 * Contract Broker Integration — CI/CD এ can-i-deploy check
 */
class ContractDeploymentChecker
{
    private string $brokerUrl;
    private string $brokerToken;

    public function __construct(string $brokerUrl, string $brokerToken)
    {
        $this->brokerUrl = $brokerUrl;
        $this->brokerToken = $brokerToken;
    }

    /**
     * Deploy করার আগে check — কোনো contract ভাঙবে না তো?
     */
    public function canIDeploy(string $serviceName, string $version): DeployDecision
    {
        $response = $this->callBroker(
            "/can-i-deploy?pacticipant={$serviceName}&version={$version}&to=production"
        );

        return new DeployDecision(
            allowed: $response['summary']['deployable'],
            reason: $response['summary']['reason'],
            verificationResults: $response['matrix'],
        );
    }

    /**
     * Pending Pacts — নতুন consumer-এর contract যা এখনো verify হয়নি
     */
    public function getPendingPacts(string $providerName): array
    {
        return $this->callBroker(
            "/pacts/provider/{$providerName}/for-verification"
        )['_embedded']['pacts'] ?? [];
    }

    private function callBroker(string $path): array
    {
        // HTTP call to Pact Broker
        return [];
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Consumer Test — Pact.js (JavaScript)

```javascript
/**
 * bKash Mobile App (Consumer) Contract Test
 * Framework: Pact JS
 */

const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { like, eachLike, string, integer, decimal, datetime } = MatchersV3;
const { BkashTransactionClient } = require('../src/clients/BkashTransactionClient');

const provider = new PactV3({
    consumer: 'BkashMobileApp',
    provider: 'TransactionAPI',
    logLevel: 'warn',
});

describe('bKash Transaction API Contract', () => {

    describe('Send Money - সফল', () => {
        it('should return success response with transaction details', async () => {
            // Contract define করা (Consumer-এর প্রত্যাশা)
            await provider
                .given('User 01812345678 has balance 5000 BDT')
                .uponReceiving('a successful send money request')
                .withRequest({
                    method: 'POST',
                    path: '/api/v1/send-money',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': like('Bearer eyJhbGciOiJIUzI1NiJ9...'),
                    },
                    body: {
                        receiver: '01712345678',
                        amount: 500,
                        pin: string('12345'),
                        reference: like('Eid Salami'),
                    },
                })
                .willRespondWith({
                    status: 200,
                    headers: { 'Content-Type': 'application/json' },
                    body: {
                        status: 'success',
                        transactionId: like('TXN-2024-ABC123'),
                        sender: {
                            number: like('01812345678'),
                            name: like('Rahim Uddin'),
                            newBalance: decimal(4500.00),
                        },
                        receiver: {
                            number: '01712345678',
                            name: like('Karim Ahmed'),
                        },
                        amount: decimal(500.00),
                        fee: decimal(5.00),
                        timestamp: datetime("yyyy-MM-dd'T'HH:mm:ss'Z'"),
                    },
                })
                .executeTest(async (mockServer) => {
                    // Consumer code test — mock provider-এর বিরুদ্ধে
                    const client = new BkashTransactionClient(mockServer.url);
                    const result = await client.sendMoney({
                        receiver: '01712345678',
                        amount: 500,
                        pin: '12345',
                        reference: 'Eid Salami',
                    });

                    // Assertions
                    expect(result.status).toBe('success');
                    expect(result.transactionId).toBeDefined();
                    expect(result.sender.newBalance).toBe(4500.00);
                    expect(result.fee).toBeGreaterThan(0);
                });
        });
    });

    describe('Send Money - ব্যর্থ (Insufficient Balance)', () => {
        it('should return error when balance is not enough', async () => {
            await provider
                .given('User 01812345678 has balance 100 BDT')
                .uponReceiving('a send money request with insufficient balance')
                .withRequest({
                    method: 'POST',
                    path: '/api/v1/send-money',
                    headers: { 'Content-Type': 'application/json' },
                    body: {
                        receiver: '01712345678',
                        amount: 50000,
                        pin: string(),
                    },
                })
                .willRespondWith({
                    status: 422,
                    body: {
                        status: 'failed',
                        error: {
                            code: 'INSUFFICIENT_BALANCE',
                            message: like('আপনার পর্যাপ্ত ব্যালেন্স নেই'),
                            currentBalance: decimal(),
                            requiredAmount: decimal(),
                        },
                    },
                })
                .executeTest(async (mockServer) => {
                    const client = new BkashTransactionClient(mockServer.url);

                    await expect(
                        client.sendMoney({ receiver: '01712345678', amount: 50000, pin: '12345' })
                    ).rejects.toThrow('INSUFFICIENT_BALANCE');
                });
        });
    });

    describe('Transaction History', () => {
        it('should return paginated transaction list', async () => {
            await provider
                .given('User has 98 transactions')
                .uponReceiving('a request for transaction history')
                .withRequest({
                    method: 'GET',
                    path: '/api/v1/transactions',
                    query: { page: '1', limit: '20' },
                    headers: { 'Authorization': like('Bearer token') },
                })
                .willRespondWith({
                    status: 200,
                    body: {
                        transactions: eachLike({
                            id: like('TXN-001'),
                            type: like('send_money'),
                            amount: decimal(500),
                            counterparty: like('01712345678'),
                            timestamp: datetime("yyyy-MM-dd'T'HH:mm:ss'Z'"),
                            status: like('completed'),
                        }),
                        pagination: {
                            currentPage: integer(1),
                            totalPages: integer(5),
                            totalItems: integer(98),
                            hasNext: like(true),
                        },
                    },
                })
                .executeTest(async (mockServer) => {
                    const client = new BkashTransactionClient(mockServer.url);
                    const history = await client.getHistory({ page: 1, limit: 20 });

                    expect(history.transactions.length).toBeGreaterThan(0);
                    expect(history.pagination.hasNext).toBe(true);
                    expect(history.pagination.totalItems).toBe(98);
                });
        });
    });
});

/**
 * Provider Verification — Provider side
 * Transaction API team এই test চালাবে
 */
const { Verifier } = require('@pact-foundation/pact');

describe('Transaction API Provider Verification', () => {
    let server;

    beforeAll(async () => {
        // Real provider service start করা (test mode)
        server = await startTransactionAPI({ port: 8080, env: 'test' });
    });

    afterAll(async () => {
        await server.close();
    });

    it('validates all consumer contracts', async () => {
        const verifier = new Verifier({
            providerBaseUrl: 'http://localhost:8080',
            provider: 'TransactionAPI',

            // Pact Broker থেকে contracts fetch
            pactBrokerUrl: 'https://pact-broker.bkash.internal',
            pactBrokerToken: process.env.PACT_BROKER_TOKEN,

            // Provider states handle করা
            stateHandlers: {
                'User 01812345678 has balance 5000 BDT': async () => {
                    await seedDatabase({
                        phone: '01812345678',
                        name: 'Rahim Uddin',
                        balance: 5000,
                    });
                },
                'User 01812345678 has balance 100 BDT': async () => {
                    await seedDatabase({
                        phone: '01812345678',
                        name: 'Rahim Uddin',
                        balance: 100,
                    });
                },
                'User has 98 transactions': async () => {
                    await seedDatabase({ phone: '01812345678', balance: 5000 });
                    await seedTransactions('01812345678', 98);
                },
            },

            // Publish verification results
            publishVerificationResult: true,
            providerVersion: process.env.GIT_SHA,
            providerVersionBranch: process.env.GIT_BRANCH,

            // Pending pacts — নতুন consumer-এর contract (fail해도 OK)
            enablePending: true,

            // WIP pacts — Work In Progress contracts include করা
            includeWipPactsSince: '2024-01-01',
        });

        await verifier.verifyProvider();
    });
});

/**
 * CI/CD Integration — can-i-deploy check
 */
class PactDeploymentGate {
    constructor(brokerUrl, brokerToken) {
        this.brokerUrl = brokerUrl;
        this.brokerToken = brokerToken;
    }

    /**
     * Deploy করার আগে safety check
     * CI/CD pipeline-এ এই step mandatory
     */
    async canIDeploy(participant, version, environment = 'production') {
        const url = `${this.brokerUrl}/can-i-deploy` +
            `?pacticipant=${participant}` +
            `&version=${version}` +
            `&to=${environment}`;

        const response = await fetch(url, {
            headers: { 'Authorization': `Bearer ${this.brokerToken}` },
        });

        const data = await response.json();

        console.log('\n📋 Can-I-Deploy Result:');
        console.log(`   Service: ${participant}`);
        console.log(`   Version: ${version}`);
        console.log(`   Target:  ${environment}`);
        console.log(`   Result:  ${data.summary.deployable ? '✅ SAFE' : '❌ BLOCKED'}`);
        console.log(`   Reason:  ${data.summary.reason}`);

        if (!data.summary.deployable) {
            console.log('\n❌ Breaking contracts detected:');
            data.matrix.forEach(item => {
                if (!item.verificationResult?.success) {
                    console.log(`   - ${item.consumer.name} → ${item.provider.name}`);
                    console.log(`     Contract: ${item.pact.version}`);
                }
            });
        }

        return {
            deployable: data.summary.deployable,
            reason: data.summary.reason,
            matrix: data.matrix,
        };
    }

    /**
     * Deploy সফল হলে tag করা
     */
    async recordDeployment(participant, version, environment) {
        await fetch(`${this.brokerUrl}/pacticipants/${participant}/versions/${version}/deployed`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${this.brokerToken}`,
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ environment }),
        });
        console.log(`✅ ${participant}@${version} deployed to ${environment}`);
    }
}

// === CI/CD Pipeline Script ===
async function deploymentPipeline() {
    const gate = new PactDeploymentGate(
        'https://pact-broker.bkash.internal',
        process.env.PACT_TOKEN
    );

    const service = 'TransactionAPI';
    const version = process.env.GIT_SHA;

    // Step 1: Can I deploy?
    const result = await gate.canIDeploy(service, version);

    if (!result.deployable) {
        console.error('🚫 Deploy blocked! Contract ভাঙবে।');
        process.exit(1);
    }

    // Step 2: Deploy
    console.log('🚀 Deploying...');
    // ... actual deployment ...

    // Step 3: Record deployment
    await gate.recordDeployment(service, version, 'production');
}
```

---

## 📊 Contract Versioning ও Pending Pacts

```
┌─────────────────────────────────────────────────────────────┐
│                 CONTRACT LIFECYCLE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐   publish   ┌──────────────┐               │
│  │  Consumer  │────────────▶│    PENDING   │               │
│  │  writes    │             │    PACT      │               │
│  │  new pact  │             │  (New, not   │               │
│  └────────────┘             │   verified)  │               │
│                             └──────┬───────┘               │
│                                    │                        │
│                                    ▼ provider verifies      │
│                             ┌──────────────┐               │
│                             │   VERIFIED   │               │
│                             │    PACT      │               │
│                             │  (Tested ✅) │               │
│                             └──────┬───────┘               │
│                                    │                        │
│                                    ▼ both deployed          │
│                             ┌──────────────┐               │
│                             │   DEPLOYED   │               │
│                             │    PACT      │               │
│                             │ (Production) │               │
│                             └──────────────┘               │
│                                                              │
│  WIP Pacts: Pending + recently added                        │
│  → Provider verify করলে fail হলেও build break হয় না      │
│  → Provider ধীরে ধীরে support যোগ করতে পারে               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🆚 Pact vs Spring Cloud Contract

```
┌────────────────┬─────────────────────┬─────────────────────┐
│ বৈশিষ্ট্য      │ Pact                │ Spring Cloud Contract│
├────────────────┼─────────────────────┼─────────────────────┤
│ Language       │ Polyglot (JS, Java, │ Java/Kotlin only    │
│                │ PHP, Go, Python)    │                     │
├────────────────┼─────────────────────┼─────────────────────┤
│ Contract Owner │ Consumer            │ Provider            │
├────────────────┼─────────────────────┼─────────────────────┤
│ Broker         │ Pactflow (SaaS)     │ Git repo            │
├────────────────┼─────────────────────┼─────────────────────┤
│ Best For       │ Multi-language      │ Spring Boot         │
│                │ microservices       │ ecosystem           │
├────────────────┼─────────────────────┼─────────────────────┤
│ bKash-এ       │ ✅ (Multi-platform) │ ❌ (Only Java)      │
│ উপযুক্ত       │                     │                     │
└────────────────┴─────────────────────┴─────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | করবেন? |
|-----------|--------|
| Microservices architecture | ✅ অবশ্যই |
| Multiple teams working on different services | ✅ অবশ্যই |
| API versioning complex হচ্ছে | ✅ হ্যাঁ |
| Mobile + Web + Partner — multiple consumers | ✅ হ্যাঁ |
| Service অনেক frequently deploy হয় | ✅ হ্যাঁ |
| Third-party integration (bKash → Bank) | ✅ হ্যাঁ |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| Monolithic application | Service boundary নেই |
| একই team সব service maintain করে | Direct communication সহজ |
| Internal library/SDK | Contract testing-এর scope না |
| GraphQL (consumer query নিজে define করে) | Schema validation যথেষ্ট |
| শুধুমাত্র ১টি consumer আছে | Overhead বেশি, লাভ কম |

---

## 📝 সারসংক্ষেপ

> **CDC Testing = Consumer বলে "আমার এটা লাগবে", Provider নিশ্চিত করে "আমি এটা দিতে পারবো"**

মনে রাখুন:
1. Consumer-ই contract define করে (তার যা লাগে শুধু সেটাই)
2. Provider verify করে (সব consumer-এর contract)
3. Pact Broker হলো single source of truth
4. `can-i-deploy` CI/CD-তে mandatory gate
5. Pending pacts নতুন consumer-কে gradually onboard করতে দেয়
