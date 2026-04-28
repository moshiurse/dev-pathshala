# 📜 কন্ট্র্যাক্ট টেস্টিং (Contract Testing) — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা ও মূল ধারণা

**কন্ট্র্যাক্ট টেস্টিং** হলো এমন একটি টেস্টিং পদ্ধতি যেখানে দুইটি সার্ভিসের (consumer ও provider) মধ্যে যোগাযোগের "চুক্তি" (contract) যাচাই করা হয়। এই চুক্তি নিশ্চিত করে যে consumer যেভাবে API কল করে, provider ঠিক সেই ফরম্যাটে রেসপন্স দেয়।

### কেন Integration Testing যথেষ্ট নয়?

Microservice architecture-এ প্রতিটি সার্ভিস স্বতন্ত্রভাবে deploy হয়। Integration test চালাতে সব সার্ভিস একসাথে চালু রাখতে হয় — যা ধীর, ভঙ্গুর ও ব্যয়বহুল। Contract testing এই সমস্যার সমাধান করে:

```
╔═════════════════════════════════════════════════════════════════════════╗
║          Integration Testing vs Contract Testing                       ║
╠═════════════════════════════════════════════════════════════════════════╣
║                                                                         ║
║  Integration Testing:                                                   ║
║  ┌──────────┐    actual call    ┌──────────┐                            ║
║  │ Consumer ├──────────────────►│ Provider │  ← দুইটাই চালু থাকতে হবে  ║
║  └──────────┘                   └──────────┘                            ║
║  • ধীর, environment-নির্ভর, flaky                                      ║
║                                                                         ║
║  Contract Testing:                                                      ║
║  ┌──────────┐  generates   ┌──────────┐  verified by  ┌──────────┐     ║
║  │ Consumer ├─────────────►│ Contract ├──────────────►│ Provider │     ║
║  └──────────┘              └──────────┘               └──────────┘     ║
║  • দ্রুত, স্বাধীন, নির্ভরযোগ্য                                        ║
║                                                                         ║
╚═════════════════════════════════════════════════════════════════════════╝
```

### Consumer-Driven Contract Testing (CDCT)

CDCT-তে **consumer** (যে সার্ভিস API ব্যবহার করে) নির্ধারণ করে তার কী দরকার। এই প্রত্যাশাগুলো একটি contract ফাইলে (সাধারণত JSON) সংরক্ষিত হয়, এবং **provider** সেই contract-এর বিপরীতে verify করে।

```
╔══════════════════════════════════════════════════════════════════╗
║                    CDCT Workflow                                 ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ১. Consumer টেস্ট লেখে → Contract (Pact file) তৈরি হয়         ║
║  ২. Contract একটি Pact Broker-এ আপলোড হয়                       ║
║  ৩. Provider সেই Contract ডাউনলোড করে verify করে                ║
║  ৪. Verification ফলাফল Broker-এ publish হয়                      ║
║  ৫. CI/CD pipeline-এ "can-i-deploy" চেক চলে                     ║
║                                                                  ║
║  Consumer ──► Pact Broker ◄── Provider                           ║
║     (publish)             (verify)                               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🔑 মূল ধারণাসমূহ

| ধারণা | বর্ণনা |
|-------|--------|
| **Consumer** | যে সার্ভিস API কল করে (client) |
| **Provider** | যে সার্ভিস API সরবরাহ করে (server) |
| **Pact** | Consumer-এর প্রত্যাশা ও provider-এর রেসপন্সের একটি JSON contract |
| **Interaction** | একটি নির্দিষ্ট request-response জোড়া |
| **Pact Broker** | Contract সংরক্ষণ ও শেয়ার করার কেন্দ্রীয় সার্ভার |
| **Provider State** | Provider-কে নির্দিষ্ট অবস্থায় আনা (যেমন: "user 1 exists") |
| **Can-I-Deploy** | deploy করার আগে compatibility যাচাই করার CLI টুল |
| **Matchers** | exact value-র বদলে type/regex/range দিয়ে match করা |

---

## 🏗️ বাস্তব উদাহরণ (Real-life Analogy)

ধরুন আপনি একটি **রেস্টুরেন্ট সাপ্লাই চেইন** চালাচ্ছেন:

- **Consumer** = রেস্তোরাঁ (যারা মাছ, সবজি অর্ডার করে)
- **Provider** = পাইকারি বাজার (যারা সরবরাহ করে)
- **Contract** = অর্ডার ফর্ম যেখানে লেখা আছে: "প্রতি সোমবার ৫ কেজি ইলিশ, ১০ কেজি আলু, দাম X টাকার মধ্যে"

রেস্তোরাঁ (consumer) তার চাহিদা অনুযায়ী ফর্ম তৈরি করে। পাইকার (provider) সেই ফর্ম দেখে নিশ্চিত করে যে তারা এই চাহিদা পূরণ করতে পারবে। যদি পাইকার হঠাৎ আলুর বদলে মিষ্টি আলু পাঠায় — contract ভঙ্গ হবে এবং আগেই ধরা পড়বে।

**এটাই contract testing-এর মূল শক্তি** — প্রোডাকশনে যাওয়ার আগেই incompatibility ধরা পড়ে।

---

## 🐘 PHP Pact উদাহরণ

### Consumer Side (PHP — Order Service)

```php
<?php

declare(strict_types=1);

use PhpPact\Consumer\InteractionBuilder;
use PhpPact\Consumer\Model\ConsumerRequest;
use PhpPact\Consumer\Model\ProviderResponse;
use PhpPact\Standalone\MockService\MockServerConfig;
use PHPUnit\Framework\TestCase;
use GuzzleHttp\Client;

class UserServiceConsumerTest extends TestCase
{
    private InteractionBuilder $builder;
    private MockServerConfig $config;
    private Client $httpClient;

    protected function setUp(): void
    {
        // Mock Server কনফিগারেশন
        $this->config = new MockServerConfig();
        $this->config->setConsumer('OrderService');
        $this->config->setProvider('UserService');
        $this->config->setHost('localhost');
        $this->config->setPort(7203);
        $this->config->setPactDir(__DIR__ . '/pacts');

        $this->builder = new InteractionBuilder($this->config);
        $this->httpClient = new Client(['base_uri' => "http://localhost:7203"]);
    }

    public function testGetUserById(): void
    {
        // Consumer-এর প্রত্যাশা নির্ধারণ
        $request = new ConsumerRequest();
        $request->setMethod('GET')
                ->setPath('/api/users/1')
                ->addHeader('Accept', 'application/json');

        $response = new ProviderResponse();
        $response->setStatus(200)
                 ->addHeader('Content-Type', 'application/json')
                 ->setBody([
                     'id'    => 1,
                     'name'  => 'Rahim Uddin',
                     'email' => 'rahim@example.com',
                     'active' => true,
                 ]);

        // Interaction তৈরি
        $this->builder
            ->given('user with id 1 exists')       // Provider State
            ->uponReceiving('a request for user 1') // Interaction বর্ণনা
            ->with($request)
            ->willRespondWith($response);

        // প্রকৃত API কল (mock server-এ)
        $result = $this->httpClient->get('/api/users/1', [
            'headers' => ['Accept' => 'application/json'],
        ]);

        $body = json_decode($result->getBody()->getContents(), true);

        // Assertion
        $this->assertEquals(200, $result->getStatusCode());
        $this->assertEquals('Rahim Uddin', $body['name']);
        $this->assertTrue($body['active']);

        // Pact ফাইল তৈরি হবে
        $this->builder->verify();
    }

    public function testGetUserNotFound(): void
    {
        $request = new ConsumerRequest();
        $request->setMethod('GET')
                ->setPath('/api/users/999')
                ->addHeader('Accept', 'application/json');

        $response = new ProviderResponse();
        $response->setStatus(404)
                 ->addHeader('Content-Type', 'application/json')
                 ->setBody([
                     'error'   => 'User not found',
                     'code'    => 'USER_NOT_FOUND',
                 ]);

        $this->builder
            ->given('user with id 999 does not exist')
            ->uponReceiving('a request for non-existent user')
            ->with($request)
            ->willRespondWith($response);

        $result = $this->httpClient->get('/api/users/999', [
            'headers' => ['Accept' => 'application/json'],
            'http_errors' => false,
        ]);

        $this->assertEquals(404, $result->getStatusCode());
        $this->builder->verify();
    }
}
```

### Provider Side (PHP — User Service Verification)

```php
<?php

declare(strict_types=1);

use PhpPact\Standalone\ProviderVerifier\Model\VerifierConfig;
use PhpPact\Standalone\ProviderVerifier\Verifier;
use PHPUnit\Framework\TestCase;

class UserServiceProviderTest extends TestCase
{
    public function testPactVerification(): void
    {
        $config = new VerifierConfig();
        $config->setProviderName('UserService')
               ->setProviderBaseUrl('http://localhost:8080')     // আসল provider URL
               ->setProviderStatesSetupUrl('http://localhost:8080/_pact/state')
               ->setBrokerUri('https://pact-broker.example.com') // Pact Broker
               ->setPublishResults(true)
               ->setProviderVersion(getenv('GIT_COMMIT') ?: '1.0.0');

        $verifier = new Verifier($config);

        // Broker থেকে সব contract verify করা
        $verifier->verifyFromBroker();

        $this->assertTrue(true, 'Pact verification successful');
    }
}
```

---

## 🟨 JavaScript Pact উদাহরণ

### Consumer Side (Node.js)

```javascript
// userService.consumer.pact.test.js
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { like, eachLike, regex } = MatchersV3;
const axios = require('axios');
const path = require('path');

const provider = new PactV3({
    consumer: 'OrderService',
    provider: 'UserService',
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'warn',
});

describe('User Service - Consumer Contract Tests', () => {
    // সফল রেসপন্স টেস্ট
    it('should return user details by ID', async () => {
        // Interaction সংজ্ঞায়িত করা
        provider
            .given('user with id 1 exists')
            .uponReceiving('a request for user details')
            .withRequest({
                method: 'GET',
                path: '/api/users/1',
                headers: { Accept: 'application/json' },
            })
            .willRespondWith({
                status: 200,
                headers: { 'Content-Type': 'application/json' },
                body: {
                    id: like(1),                            // type matching — যেকোনো integer হলেই হবে
                    name: like('Rahim Uddin'),               // type matching
                    email: regex(/^\S+@\S+\.\S+$/, 'rahim@example.com'), // regex matching
                    roles: eachLike('admin'),                // array-র প্রতিটি element string
                    createdAt: like('2024-01-15T10:30:00Z'), // ISO date format
                },
            });

        // Mock provider-এ প্রকৃত API কল
        await provider.executeTest(async (mockServer) => {
            const response = await axios.get(`${mockServer.url}/api/users/1`, {
                headers: { Accept: 'application/json' },
            });

            expect(response.status).toBe(200);
            expect(response.data.name).toBeDefined();
            expect(response.data.email).toContain('@');
            expect(Array.isArray(response.data.roles)).toBe(true);
        });
    });

    // ব্যবহারকারীদের তালিকা
    it('should return a list of users', async () => {
        provider
            .given('users exist in the system')
            .uponReceiving('a request for user list')
            .withRequest({
                method: 'GET',
                path: '/api/users',
                query: { page: '1', limit: '10' },
            })
            .willRespondWith({
                status: 200,
                body: {
                    data: eachLike({
                        id: like(1),
                        name: like('User Name'),
                        email: like('user@example.com'),
                    }),
                    meta: {
                        total: like(100),
                        page: like(1),
                        limit: like(10),
                    },
                },
            });

        await provider.executeTest(async (mockServer) => {
            const response = await axios.get(`${mockServer.url}/api/users`, {
                params: { page: 1, limit: 10 },
            });

            expect(response.status).toBe(200);
            expect(response.data.data.length).toBeGreaterThan(0);
            expect(response.data.meta.total).toBeDefined();
        });
    });

    // Error response
    it('should return 404 for non-existent user', async () => {
        provider
            .given('user with id 999 does not exist')
            .uponReceiving('a request for non-existent user')
            .withRequest({
                method: 'GET',
                path: '/api/users/999',
            })
            .willRespondWith({
                status: 404,
                body: {
                    error: like('User not found'),
                    code: like('USER_NOT_FOUND'),
                },
            });

        await provider.executeTest(async (mockServer) => {
            try {
                await axios.get(`${mockServer.url}/api/users/999`);
            } catch (error) {
                expect(error.response.status).toBe(404);
                expect(error.response.data.code).toBe('USER_NOT_FOUND');
            }
        });
    });
});
```

### Provider Side (Node.js Verification)

```javascript
// userService.provider.pact.test.js
const { Verifier } = require('@pact-foundation/pact');
const path = require('path');
const app = require('../src/app'); // Express/Fastify app

describe('User Service - Provider Verification', () => {
    let server;

    beforeAll(async () => {
        server = app.listen(8081);
    });

    afterAll(() => server.close());

    it('should validate the contract against the provider', async () => {
        const opts = {
            providerBaseUrl: 'http://localhost:8081',
            provider: 'UserService',

            // Pact Broker থেকে verify
            pactBrokerUrl: process.env.PACT_BROKER_URL || 'https://pact-broker.example.com',
            publishVerificationResult: process.env.CI === 'true',
            providerVersion: process.env.GIT_SHA || '1.0.0',

            // Provider States সেটআপ
            stateHandlers: {
                'user with id 1 exists': async () => {
                    await db.users.create({ id: 1, name: 'Rahim Uddin', email: 'rahim@example.com' });
                },
                'user with id 999 does not exist': async () => {
                    await db.users.deleteWhere({ id: 999 });
                },
                'users exist in the system': async () => {
                    await db.users.bulkCreate([
                        { id: 1, name: 'User 1', email: 'u1@example.com' },
                        { id: 2, name: 'User 2', email: 'u2@example.com' },
                    ]);
                },
            },
        };

        await new Verifier(opts).verifyProvider();
    });
});
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন দরকার |
|-----------|----------|
| Microservice architecture-তে | সার্ভিসগুলো স্বাধীনভাবে deploy হয়, compatibility নিশ্চিত করতে হবে |
| বিভিন্ন দল আলাদা সার্ভিস maintain করলে | দুই দলের মধ্যে যোগাযোগের "চুক্তি" হিসেবে কাজ করে |
| API versioning-এর সময় | পুরনো consumer-রা ভাঙবে না তা নিশ্চিত করতে |
| CI/CD pipeline-এ fast feedback দরকার হলে | Integration test-এর চেয়ে অনেক দ্রুত |
| Third-party API ব্যবহার করলে | তাদের API পরিবর্তন হলে আগেই জানা যাবে |
| Frontend-Backend আলাদা দল হলে | API-র shape নিয়ে দুই দলের সম্মতি থাকবে |

## ❌ কখন করবেন না / সীমাবদ্ধতা

| পরিস্থিতি | কারণ |
|-----------|------|
| Monolith application-এ | একই codebase-এ সব আছে, contract-এর প্রয়োজন নেই |
| Business logic verification-এর জন্য | Contract শুধু API-র shape যাচাই করে, logic নয় |
| End-to-end flow টেস্ট করতে | E2E testing আলাদাভাবে করুন |
| শুধু একটি consumer থাকলে | জটিলতা বনাম সুবিধার হিসাবে মূল্যায়ন করুন |
| Performance/load testing-এর বিকল্প হিসেবে | এটি functional compatibility-র জন্য, performance-এর জন্য নয় |
| যেসব API কদাচিৎ পরিবর্তন হয় | রক্ষণাবেক্ষণের খরচ সুবিধার চেয়ে বেশি হতে পারে |

---

## 🧠 সিনিয়র ইঞ্জিনিয়ারদের জন্য পরামর্শ

১. **Pact Broker ব্যবহার করুন** — লোকাল ফাইল শেয়ারিং-এর বদলে centralized broker ব্যবহার করুন। এটি versioning, tagging ও can-i-deploy সুবিধা দেয়।

২. **Matcher ব্যবহার করুন, exact value নয়** — `like()`, `eachLike()`, `regex()` ব্যবহার করলে contract কম ভঙ্গুর হয়। আপনি data-র type ও shape যাচাই করতে চান, exact value নয়।

৩. **Provider States সঠিকভাবে সেটআপ করুন** — প্রতিটি interaction-এর জন্য provider-কে সঠিক অবস্থায় আনা অত্যন্ত গুরুত্বপূর্ণ।

৪. **Contract Test ≠ Integration Test** — Contract test শুধু API-র interface যাচাই করে। ব্যবসায়িক logic ও data flow-এর জন্য আলাদা integration test রাখুন।

৫. **CI/CD-তে can-i-deploy ব্যবহার করুন** — deploy-এর আগে `pact-broker can-i-deploy --pacticipant OrderService --version <sha> --to production` চালান।

> **মনে রাখবেন**: "Contract testing সার্ভিসগুলোর মধ্যে বিশ্বাসের সেতু তৈরি করে — যাতে প্রত্যেকে স্বাধীনভাবে deploy করতে পারে।" 🤝
