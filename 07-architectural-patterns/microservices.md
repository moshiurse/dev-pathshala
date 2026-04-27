# 🔬 মাইক্রোসার্ভিস আর্কিটেকচার (Microservices Architecture)

## 📌 সংজ্ঞা ও মূল ধারণা

**Microservices Architecture** হলো একটি software design approach যেখানে একটি বড় application-কে ছোট ছোট, স্বাধীনভাবে deployable service-এ ভেঙে ফেলা হয়। প্রতিটি service নিজস্ব business capability handle করে, নিজস্ব database রাখে এবং lightweight protocol (HTTP/gRPC/messaging) দিয়ে অন্য service-এর সাথে communicate করে।

**Martin Fowler-এর সংজ্ঞা:**
> "The microservice architectural style is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms."

### ইতিহাস ও বিবর্তন

২০১১-১২ সালে Netflix, Amazon, এবং eBay-এর মতো বড় কোম্পানি monolithic architecture-এর সীমাবদ্ধতা অনুভব করে। তারা তাদের system-কে ছোট ছোট service-এ ভাঙতে শুরু করে। ২০১৪ সালে Martin Fowler এবং James Lewis আনুষ্ঠানিকভাবে "Microservices" term-টি define করেন।

### Monolith vs Microservices তুলনা

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MONOLITH vs MICROSERVICES                        │
├──────────────────────┬──────────────────────────────────────────────┤
│      MONOLITH        │              MICROSERVICES                   │
│                      │                                              │
│  ┌──────────────┐    │    ┌─────┐  ┌─────┐  ┌─────┐               │
│  │  UI Layer    │    │    │User │  │Order│  │Pay- │               │
│  ├──────────────┤    │    │Svc  │  │Svc  │  │ment │               │
│  │Business Logic│    │    │ DB  │  │ DB  │  │ DB  │               │
│  ├──────────────┤    │    └─────┘  └─────┘  └─────┘               │
│  │  Data Layer  │    │    ┌─────┐  ┌─────┐  ┌─────┐               │
│  ├──────────────┤    │    │Notif│  │Inven│  │Ship-│               │
│  │  Single DB   │    │    │Svc  │  │tory │  │ping │               │
│  └──────────────┘    │    │ DB  │  │ DB  │  │ DB  │               │
│                      │    └─────┘  └─────┘  └─────┘               │
├──────────────────────┼──────────────────────────────────────────────┤
│ একটি codebase       │ প্রতিটি service আলাদা codebase              │
│ একটি database        │ প্রতি service-এ আলাদা database              │
│ একসাথে deploy        │ independently deploy করা যায়               │
│ একটি tech stack      │ polyglot (বিভিন্ন language ব্যবহার সম্ভব)  │
│ সহজে শুরু করা যায়   │ প্রাথমিক complexity বেশি                   │
│ scaling পুরো app-এ   │ individual service scale করা যায়            │
└──────────────────────┴──────────────────────────────────────────────┘
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

ধরুন ঢাকা শহরকে একটি software system হিসেবে চিন্তা করুন:

**Monolith পদ্ধতি:** একটি বিশাল ভবনে হাসপাতাল, স্কুল, ফায়ার স্টেশন, পুলিশ স্টেশন — সব একসাথে। ভবনের লিফট নষ্ট হলে সব কিছু বন্ধ।

**Microservices পদ্ধতি:** প্রতিটি department আলাদা:
- 🏥 **হাসপাতাল** → User Service (নিজস্ব staff, database, ভবন)
- 🚒 **ফায়ার স্টেশন** → Notification Service (স্বাধীনভাবে কাজ করে)
- 🏫 **স্কুল** → Education/Content Service
- 🏦 **ব্যাংক** → Payment Service (bKash/Nagad-এর মতো)

প্রতিটি department নিজস্ব resource নিয়ে স্বাধীনভাবে চলে। ফায়ার স্টেশনে সমস্যা হলে হাসপাতাল ঠিকই চালু থাকে। তারা phone (API), চিঠি (message queue), বা মিটিং (synchronous call) দিয়ে communicate করে।

**বাংলাদেশের Pathao উদাহরণ:**
- **Ride Service** → রাইড বুকিং ও ম্যাচিং
- **Payment Service** → bKash/card payment process
- **Notification Service** → push notification, SMS
- **Driver Service** → driver tracking, availability
- **Pricing Service** → surge pricing, discount calculation

প্রতিটি service আলাদাভাবে scale করা যায় — পিক আওয়ারে শুধু Ride Service ও Pricing Service scale করলেই হয়।

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

```
                            ┌──────────────┐
                            │   Client     │
                            │ (Mobile/Web) │
                            └──────┬───────┘
                                   │
                            ┌──────▼───────┐
                            │  API Gateway │
                            │ (Kong/Nginx) │
                            │  - Auth      │
                            │  - Rate Limit│
                            │  - Routing   │
                            └──────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
             ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼───────┐
             │ User Svc   │ │ Order Svc  │ │ Payment Svc│
             │ (Laravel)  │ │ (Node.js)  │ │ (Laravel)  │
             └──────┬─────┘ └─────┬──────┘ └────┬───────┘
                    │             │              │
             ┌──────▼─────┐ ┌────▼───────┐ ┌───▼────────┐
             │ PostgreSQL │ │  MongoDB   │ │   MySQL    │
             └────────────┘ └────────────┘ └────────────┘
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                          ┌────────▼────────┐
                          │  Message Broker  │
                          │ (RabbitMQ/Kafka) │
                          └────────┬────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
             ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼───────┐
             │ Notif Svc  │ │Inventory   │ │ Shipping   │
             │            │ │  Svc       │ │   Svc      │
             └────────────┘ └────────────┘ └────────────┘

        ┌─────────────────────────────────────────────┐
        │          Supporting Infrastructure           │
        │                                             │
        │  ┌────────────┐  ┌───────────┐  ┌────────┐ │
        │  │  Consul    │  │  Jaeger   │  │ Redis  │ │
        │  │ (Discovery)│  │ (Tracing) │  │(Cache) │ │
        │  └────────────┘  └───────────┘  └────────┘ │
        └─────────────────────────────────────────────┘
```

### Request Flow (অর্ডার তৈরির ধাপ)

```
Client ──► API Gateway ──► Order Service ──► Payment Service
                                │                   │
                                │    ◄── Success ────┘
                                │
                                ▼ (Event: OrderCreated)
                          Message Queue
                           /        \
                          ▼          ▼
                   Inventory     Notification
                    Service       Service
                      │              │
                      ▼              ▼
                  Stock Update   SMS/Email Send
```

---

## 💻 মাইক্রোসার্ভিস ডিজাইন প্রিন্সিপল

### ১. Single Responsibility per Service
প্রতিটি service শুধু একটি নির্দিষ্ট business domain handle করবে। User service শুধু user management করবে, payment নয়।

### ২. Database per Service
প্রতিটি service-এর নিজস্ব database থাকবে। কোনো service অন্য service-এর database সরাসরি access করবে না।

### ৩. API First Design
Service তৈরির আগে API contract (OpenAPI/Swagger) define করতে হবে। এটি team-গুলোকে parallel-এ কাজ করতে সাহায্য করে।

### ৪. Decentralized Data Management
কোনো central database নেই। প্রতিটি service নিজের data নিজে manage করে। Data consistency-র জন্য eventual consistency pattern ব্যবহার করা হয়।

### ৫. Infrastructure Automation
CI/CD pipeline, containerization (Docker), orchestration (Kubernetes) — সব automated হতে হবে। Manual deployment microservices-এ অবাস্তব।

---

## 🔥 Complex Implementation Scenarios

### ১. API Gateway Pattern

API Gateway সমস্ত client request-এর single entry point হিসেবে কাজ করে। এটি authentication, rate limiting, request routing, এবং response aggregation handle করে।

**PHP (Laravel) — Gateway Service:**

```php
<?php
// app/Services/ApiGateway.php

declare(strict_types=1);

namespace App\Services;

use App\Enums\ServiceName;
use App\DTOs\GatewayRequest;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\RateLimiter;

enum ServiceName: string
{
    case USER = 'user-service';
    case ORDER = 'order-service';
    case PAYMENT = 'payment-service';
    case NOTIFICATION = 'notification-service';
}

final readonly class ServiceRegistry
{
    public function __construct(
        private array $services = [
            'user-service'    => 'http://user-svc:8001',
            'order-service'   => 'http://order-svc:8002',
            'payment-service' => 'http://payment-svc:8003',
        ]
    ) {}

    public function resolve(ServiceName $service): string
    {
        return $this->services[$service->value]
            ?? throw new \RuntimeException("Service not found: {$service->value}");
    }
}

final readonly class ApiGatewayService
{
    public function __construct(
        private ServiceRegistry $registry,
        private CircuitBreakerManager $circuitBreaker,
    ) {}

    public function forward(
        ServiceName $service,
        string $method,
        string $path,
        array $payload = [],
        array $headers = [],
    ): array {
        // Rate limiting check
        $key = 'gateway:' . request()->ip();
        if (RateLimiter::tooManyAttempts($key, maxAttempts: 100)) {
            throw new \App\Exceptions\RateLimitExceededException(
                retryAfter: RateLimiter::availableIn($key)
            );
        }
        RateLimiter::hit($key, decaySeconds: 60);

        $baseUrl = $this->registry->resolve($service);
        $url = "{$baseUrl}/{$path}";

        // Correlation ID propagation
        $correlationId = $headers['X-Correlation-ID']
            ?? request()->header('X-Correlation-ID')
            ?? (string) \Illuminate\Support\Str::uuid();

        $requestHeaders = [
            ...$headers,
            'X-Correlation-ID' => $correlationId,
            'X-Gateway-Time'   => now()->toISOString(),
        ];

        return $this->circuitBreaker->execute(
            service: $service->value,
            action: fn () => Http::withHeaders($requestHeaders)
                ->timeout(seconds: 5)
                ->retry(times: 2, sleepMilliseconds: 100)
                ->{$method}($url, $payload)
                ->throw()
                ->json(),
            fallback: fn () => ['error' => 'Service unavailable', 'cached' => true],
        );
    }
}

// app/Http/Controllers/GatewayController.php
final class GatewayController extends Controller
{
    public function __construct(
        private readonly ApiGatewayService $gateway,
    ) {}

    public function __invoke(Request $request, string $service, string $path): JsonResponse
    {
        $serviceName = ServiceName::tryFrom($service)
            ?? throw new NotFoundHttpException("Unknown service: {$service}");

        $result = $this->gateway->forward(
            service: $serviceName,
            method: strtolower($request->method()),
            path: $path,
            payload: $request->all(),
            headers: $request->headers->all(),
        );

        return response()->json($result);
    }
}
```

**Node.js (Express) — API Gateway:**

```javascript
// src/gateway/ApiGateway.js
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';
import rateLimit from 'express-rate-limit';
import { v4 as uuidv4 } from 'uuid';

class ServiceRegistry {
    #services = new Map([
        ['user-service',    'http://user-svc:8001'],
        ['order-service',   'http://order-svc:8002'],
        ['payment-service', 'http://payment-svc:8003'],
    ]);

    resolve(serviceName) {
        const url = this.#services.get(serviceName);
        if (!url) throw new Error(`Service not found: ${serviceName}`);
        return url;
    }

    register(name, url) {
        this.#services.set(name, url);
    }
}

class ApiGateway {
    #app;
    #registry;
    #circuitBreakers = new Map();

    constructor() {
        this.#app = express();
        this.#registry = new ServiceRegistry();
        this.#setupMiddleware();
        this.#setupRoutes();
    }

    #setupMiddleware() {
        // Rate limiting — প্রতি মিনিটে সর্বোচ্চ ১০০ request
        this.#app.use(rateLimit({
            windowMs: 60 * 1000,
            max: 100,
            standardHeaders: true,
            message: { error: 'Too many requests', retryAfter: '60s' },
        }));

        // Correlation ID injection
        this.#app.use((req, _res, next) => {
            req.correlationId = req.headers['x-correlation-id'] ?? uuidv4();
            req.headers['x-correlation-id'] = req.correlationId;
            req.headers['x-gateway-time'] = new Date().toISOString();
            next();
        });

        // Authentication middleware
        this.#app.use(async (req, res, next) => {
            const publicPaths = ['/api/auth/login', '/api/auth/register'];
            if (publicPaths.includes(req.path)) return next();

            const token = req.headers.authorization?.split(' ')[1];
            if (!token) return res.status(401).json({ error: 'Token required' });

            try {
                const user = await this.#verifyToken(token);
                req.headers['x-user-id'] = user.id;
                req.headers['x-user-role'] = user.role;
                next();
            } catch {
                res.status(401).json({ error: 'Invalid token' });
            }
        });
    }

    #setupRoutes() {
        const services = ['user-service', 'order-service', 'payment-service'];

        for (const service of services) {
            const prefix = `/api/${service.replace('-service', '')}`;
            const target = this.#registry.resolve(service);

            this.#app.use(prefix, createProxyMiddleware({
                target,
                changeOrigin: true,
                pathRewrite: { [`^${prefix}`]: '' },
                onError: (err, _req, res) => {
                    console.error(`[Gateway] ${service} error:`, err.message);
                    res.status(503).json({
                        error: 'Service temporarily unavailable',
                        service,
                    });
                },
            }));
        }
    }

    async #verifyToken(token) {
        // JWT verification logic
        const jwt = await import('jsonwebtoken');
        return jwt.default.verify(token, process.env.JWT_SECRET);
    }

    listen(port = 3000) {
        this.#app.listen(port, () =>
            console.log(`API Gateway running on port ${port}`)
        );
    }
}

const gateway = new ApiGateway();
gateway.listen(process.env.PORT ?? 3000);
```

---

### ২. Service Discovery

Service Discovery হলো এমন একটি mechanism যেখানে service-গুলো dynamically একে অপরের location (IP/port) খুঁজে পায়। Production-এ service instance-গুলো ক্রমাগত তৈরি ও ধ্বংস হয়, তাই hardcoded URL ব্যবহার করা সম্ভব নয়।

**Client-side Discovery:** Client নিজেই service registry (Consul/etcd) থেকে service location বের করে।  
**Server-side Discovery:** Load balancer (Nginx/AWS ALB) registry থেকে location বের করে client-এর হয়ে route করে।

**PHP — Health Check ও Service Registration:**

```php
<?php
// app/Services/ServiceDiscovery.php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Http;

final readonly class ConsulServiceDiscovery
{
    public function __construct(
        private string $consulUrl = 'http://consul:8500',
    ) {}

    public function register(
        string $serviceName,
        string $serviceId,
        string $address,
        int $port,
        array $tags = [],
    ): void {
        Http::put("{$this->consulUrl}/v1/agent/service/register", [
            'ID'      => $serviceId,
            'Name'    => $serviceName,
            'Address' => $address,
            'Port'    => $port,
            'Tags'    => $tags,
            'Check'   => [
                'HTTP'                           => "http://{$address}:{$port}/health",
                'Interval'                       => '10s',
                'Timeout'                        => '3s',
                'DeregisterCriticalServiceAfter' => '30s',
            ],
        ]);
    }

    public function discover(string $serviceName): array
    {
        $response = Http::get(
            "{$this->consulUrl}/v1/health/service/{$serviceName}",
            ['passing' => 'true']
        );

        return collect($response->json())
            ->map(fn (array $entry) => [
                'id'      => $entry['Service']['ID'],
                'address' => $entry['Service']['Address'],
                'port'    => $entry['Service']['Port'],
                'tags'    => $entry['Service']['Tags'],
            ])
            ->toArray();
    }

    public function resolveOne(string $serviceName): string
    {
        $instances = $this->discover($serviceName);

        if (empty($instances)) {
            throw new \RuntimeException("No healthy instance for: {$serviceName}");
        }

        // সহজ round-robin — random instance select
        $instance = $instances[array_rand($instances)];
        return "http://{$instance['address']}:{$instance['port']}";
    }
}

// app/Http/Controllers/HealthController.php
final class HealthController extends Controller
{
    public function __invoke(): JsonResponse
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'redis'    => $this->checkRedis(),
            'disk'     => disk_free_space('/') > 100 * 1024 * 1024,
        ];

        $healthy = !in_array(false, $checks, true);

        return response()->json([
            'status'  => $healthy ? 'healthy' : 'degraded',
            'checks'  => $checks,
            'version' => config('app.version'),
            'uptime'  => now()->diffInSeconds(LARAVEL_START),
        ], $healthy ? 200 : 503);
    }

    private function checkDatabase(): bool
    {
        try {
            \DB::connection()->getPdo();
            return true;
        } catch (\Throwable) {
            return false;
        }
    }

    private function checkRedis(): bool
    {
        try {
            \Illuminate\Support\Facades\Redis::ping();
            return true;
        } catch (\Throwable) {
            return false;
        }
    }
}
```

**Node.js — Service Discovery:**

```javascript
// src/discovery/ConsulDiscovery.js
import Consul from 'consul';

class ServiceDiscovery {
    #consul;
    #serviceId;
    #cache = new Map();

    constructor(consulHost = 'consul', consulPort = 8500) {
        this.#consul = new Consul({ host: consulHost, port: consulPort });
    }

    async register({ name, id, address, port, tags = [] }) {
        this.#serviceId = id;

        await this.#consul.agent.service.register({
            id,
            name,
            address,
            port,
            tags,
            check: {
                http: `http://${address}:${port}/health`,
                interval: '10s',
                timeout: '3s',
                deregistercriticalserviceafter: '30s',
            },
        });

        // Graceful deregister on shutdown
        for (const signal of ['SIGINT', 'SIGTERM']) {
            process.on(signal, async () => {
                await this.deregister();
                process.exit(0);
            });
        }
    }

    async discover(serviceName) {
        const cached = this.#cache.get(serviceName);
        if (cached && Date.now() - cached.timestamp < 5000) {
            return cached.instances;
        }

        const result = await this.#consul.health.service({
            service: serviceName,
            passing: true,
        });

        const instances = result.map(entry => ({
            id: entry.Service.ID,
            address: entry.Service.Address,
            port: entry.Service.Port,
        }));

        this.#cache.set(serviceName, { instances, timestamp: Date.now() });
        return instances;
    }

    async resolveOne(serviceName) {
        const instances = await this.discover(serviceName);
        if (!instances.length) throw new Error(`No instance: ${serviceName}`);
        const pick = instances[Math.floor(Math.random() * instances.length)];
        return `http://${pick.address}:${pick.port}`;
    }

    async deregister() {
        if (this.#serviceId) {
            await this.#consul.agent.service.deregister(this.#serviceId);
        }
    }
}

// Health check endpoint
import express from 'express';
const app = express();

app.get('/health', async (_req, res) => {
    const checks = {
        database: await checkMongo(),
        memory: process.memoryUsage().heapUsed < 500 * 1024 * 1024,
        uptime: process.uptime(),
    };

    const healthy = Object.values(checks).every(Boolean);
    res.status(healthy ? 200 : 503).json({
        status: healthy ? 'healthy' : 'degraded',
        checks,
        version: process.env.APP_VERSION ?? '1.0.0',
    });
});

export { ServiceDiscovery };
```

---

### ৩. Inter-Service Communication

Microservices-এ service-গুলো দুইভাবে communicate করে:

**Synchronous (তাৎক্ষণিক উত্তর দরকার):** REST API call, gRPC  
**Asynchronous (পরে process হলেও চলবে):** Message Queue (RabbitMQ, Kafka)

#### Synchronous Communication — gRPC Example

**PHP (gRPC Client):**

```php
<?php
// app/Services/GrpcUserClient.php

declare(strict_types=1);

namespace App\Services;

use App\Protos\UserServiceClient;
use App\Protos\GetUserRequest;
use Grpc\ChannelCredentials;

final readonly class GrpcUserClient
{
    private UserServiceClient $client;

    public function __construct(string $host = 'user-service:50051')
    {
        $this->client = new UserServiceClient($host, [
            'credentials' => ChannelCredentials::createInsecure(),
        ]);
    }

    public function getUser(int $userId): array
    {
        $request = new GetUserRequest();
        $request->setUserId($userId);

        [$response, $status] = $this->client->GetUser($request)->wait();

        if ($status->code !== \Grpc\STATUS_OK) {
            throw new \RuntimeException("gRPC error: {$status->details}");
        }

        return [
            'id'    => $response->getId(),
            'name'  => $response->getName(),
            'email' => $response->getEmail(),
        ];
    }
}
```

#### Asynchronous Communication — RabbitMQ

**PHP (Laravel Queue — Producer/Consumer):**

```php
<?php
// app/Events/OrderCreatedEvent.php
declare(strict_types=1);

namespace App\Events;

final readonly class OrderCreatedEvent
{
    public function __construct(
        public string $orderId,
        public string $userId,
        public float $totalAmount,
        public array $items,
        public string $correlationId,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}

    public function toArray(): array
    {
        return [
            'event_type'     => 'order.created',
            'order_id'       => $this->orderId,
            'user_id'        => $this->userId,
            'total_amount'   => $this->totalAmount,
            'items'          => $this->items,
            'correlation_id' => $this->correlationId,
            'occurred_at'    => $this->occurredAt->format('c'),
        ];
    }
}

// app/Jobs/ProcessPaymentJob.php — Consumer
namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;

final class ProcessPaymentJob implements ShouldQueue
{
    use Queueable, InteractsWithQueue;

    public int $tries = 3;
    public int $backoff = 30;

    public function __construct(
        private readonly string $orderId,
        private readonly string $userId,
        private readonly float $amount,
        private readonly string $correlationId,
    ) {}

    public function handle(PaymentService $paymentService): void
    {
        logger()->info("Processing payment", [
            'order_id'       => $this->orderId,
            'correlation_id' => $this->correlationId,
        ]);

        $result = $paymentService->charge(
            userId: $this->userId,
            amount: $this->amount,
            reference: $this->orderId,
        );

        if ($result->successful()) {
            // Payment success event publish করো
            event(new PaymentCompletedEvent(
                orderId: $this->orderId,
                transactionId: $result->transactionId,
            ));
        }
    }

    public function failed(\Throwable $exception): void
    {
        logger()->error("Payment failed for order: {$this->orderId}", [
            'error' => $exception->getMessage(),
        ]);
        // Compensation: Order cancel event publish
        event(new PaymentFailedEvent(orderId: $this->orderId));
    }
}

// Publishing — OrderService
$event = new OrderCreatedEvent(
    orderId: $order->id,
    userId: auth()->id(),
    totalAmount: $order->total,
    items: $order->items->toArray(),
    correlationId: request()->header('X-Correlation-ID'),
);

ProcessPaymentJob::dispatch(
    $event->orderId,
    $event->userId,
    $event->totalAmount,
    $event->correlationId,
)->onQueue('payments');
```

**Node.js (RabbitMQ — amqplib):**

```javascript
// src/messaging/RabbitMQBroker.js
import amqp from 'amqplib';

class MessageBroker {
    #connection;
    #channel;
    #exchange = 'microservices.events';

    async connect(url = 'amqp://rabbitmq:5672') {
        this.#connection = await amqp.connect(url);
        this.#channel = await this.#connection.createChannel();

        await this.#channel.assertExchange(this.#exchange, 'topic', {
            durable: true,
        });

        // Graceful shutdown
        process.once('SIGINT', () => this.close());
    }

    async publish(routingKey, message, correlationId) {
        const payload = JSON.stringify({
            ...message,
            correlation_id: correlationId,
            published_at: new Date().toISOString(),
        });

        this.#channel.publish(
            this.#exchange,
            routingKey,
            Buffer.from(payload),
            {
                persistent: true,
                contentType: 'application/json',
                correlationId,
            }
        );

        console.log(`[Published] ${routingKey}:`, message.event_type);
    }

    async subscribe(routingKey, queueName, handler) {
        const { queue } = await this.#channel.assertQueue(queueName, {
            durable: true,
            deadLetterExchange: `${this.#exchange}.dlx`,
        });

        await this.#channel.bindQueue(queue, this.#exchange, routingKey);
        await this.#channel.prefetch(10);

        this.#channel.consume(queue, async (msg) => {
            if (!msg) return;

            try {
                const data = JSON.parse(msg.content.toString());
                await handler(data);
                this.#channel.ack(msg);
            } catch (error) {
                console.error(`[Consumer Error] ${queueName}:`, error.message);
                // ৩ বার retry-র পর dead letter queue-তে পাঠাও
                const retries = (msg.properties.headers?.['x-retry-count'] ?? 0);
                if (retries < 3) {
                    this.#channel.nack(msg, false, true);
                } else {
                    this.#channel.nack(msg, false, false); // DLQ-তে যাবে
                }
            }
        });
    }

    async close() {
        await this.#channel?.close();
        await this.#connection?.close();
    }
}

// ব্যবহার — Payment Service Consumer
const broker = new MessageBroker();
await broker.connect();

await broker.subscribe('order.created', 'payment-queue', async (data) => {
    console.log(`Processing payment for order: ${data.order_id}`);
    const result = await paymentService.charge({
        userId: data.user_id,
        amount: data.total_amount,
        reference: data.order_id,
    });

    await broker.publish('payment.completed', {
        event_type: 'payment.completed',
        order_id: data.order_id,
        transaction_id: result.transactionId,
    }, data.correlation_id);
});
```

---

### ৪. Saga Pattern (Distributed Transactions)

Traditional database transaction (ACID) microservices-এ কাজ করে না কারণ প্রতিটি service-এর আলাদা database। Saga pattern distributed transaction manage করে — প্রতিটি step-এ একটি local transaction চালায় এবং ব্যর্থ হলে compensation (rollback) action চালায়।

#### E-commerce Order Flow:
```
Order Created → Payment Charged → Inventory Reserved → Shipping Initiated
     ↓                ↓                  ↓                    ↓
  (Cancel)     (Refund Payment)   (Release Stock)     (Cancel Shipment)
```

**PHP — Orchestration-based Saga:**

```php
<?php
// app/Sagas/OrderSaga.php

declare(strict_types=1);

namespace App\Sagas;

enum SagaStepStatus: string
{
    case PENDING    = 'pending';
    case COMPLETED  = 'completed';
    case FAILED     = 'failed';
    case COMPENSATED = 'compensated';
}

final readonly class SagaStep
{
    public function __construct(
        public string $name,
        public \Closure $execute,
        public \Closure $compensate,
        public SagaStepStatus $status = SagaStepStatus::PENDING,
    ) {}
}

final class OrderSagaOrchestrator
{
    private array $steps = [];
    private array $completedSteps = [];
    private array $sagaLog = [];

    public function __construct(
        private readonly string $orderId,
        private readonly string $correlationId,
    ) {}

    public function addStep(string $name, \Closure $execute, \Closure $compensate): self
    {
        $this->steps[] = new SagaStep($name, $execute, $compensate);
        return $this;
    }

    public function execute(): SagaResult
    {
        foreach ($this->steps as $step) {
            try {
                $this->log($step->name, 'executing');
                ($step->execute)($this->orderId);
                $this->completedSteps[] = $step;
                $this->log($step->name, 'completed');
            } catch (\Throwable $e) {
                $this->log($step->name, "failed: {$e->getMessage()}");
                $this->compensate();

                return new SagaResult(
                    success: false,
                    failedStep: $step->name,
                    error: $e->getMessage(),
                    log: $this->sagaLog,
                );
            }
        }

        return new SagaResult(
            success: true,
            log: $this->sagaLog,
        );
    }

    private function compensate(): void
    {
        // বিপরীত ক্রমে compensation চালাও
        foreach (array_reverse($this->completedSteps) as $step) {
            try {
                $this->log($step->name, 'compensating');
                ($step->compensate)($this->orderId);
                $this->log($step->name, 'compensated');
            } catch (\Throwable $e) {
                $this->log($step->name, "compensation failed: {$e->getMessage()}");
                // Manual intervention দরকার — alert পাঠাও
            }
        }
    }

    private function log(string $step, string $message): void
    {
        $this->sagaLog[] = [
            'step'           => $step,
            'message'        => $message,
            'correlation_id' => $this->correlationId,
            'timestamp'      => now()->toISOString(),
        ];
    }
}

// ব্যবহার — Order Creation Saga
$saga = new OrderSagaOrchestrator($orderId, $correlationId);

$saga
    ->addStep(
        name: 'create_order',
        execute: fn (string $id) => $orderService->create($id),
        compensate: fn (string $id) => $orderService->cancel($id),
    )
    ->addStep(
        name: 'process_payment',
        execute: fn (string $id) => $paymentService->charge($id),
        compensate: fn (string $id) => $paymentService->refund($id),
    )
    ->addStep(
        name: 'reserve_inventory',
        execute: fn (string $id) => $inventoryService->reserve($id),
        compensate: fn (string $id) => $inventoryService->release($id),
    )
    ->addStep(
        name: 'initiate_shipping',
        execute: fn (string $id) => $shippingService->initiate($id),
        compensate: fn (string $id) => $shippingService->cancel($id),
    );

$result = $saga->execute();

if (!$result->success) {
    logger()->error("Saga failed at step: {$result->failedStep}");
}
```

**Node.js — Choreography-based Saga:**

```javascript
// src/sagas/OrderSaga.js

class SagaStep {
    #name;
    #execute;
    #compensate;

    constructor(name, executeFn, compensateFn) {
        this.#name = name;
        this.#execute = executeFn;
        this.#compensate = compensateFn;
    }

    get name() { return this.#name; }

    async execute(context) {
        return this.#execute(context);
    }

    async compensate(context) {
        return this.#compensate(context);
    }
}

class SagaOrchestrator {
    #steps = [];
    #completedSteps = [];
    #sagaLog = [];

    addStep(name, executeFn, compensateFn) {
        this.#steps.push(new SagaStep(name, executeFn, compensateFn));
        return this;
    }

    async execute(context) {
        for (const step of this.#steps) {
            try {
                this.#log(step.name, 'executing');
                const result = await step.execute(context);
                context = { ...context, ...result };
                this.#completedSteps.push(step);
                this.#log(step.name, 'completed');
            } catch (error) {
                this.#log(step.name, `failed: ${error.message}`);
                await this.#compensate(context);

                return {
                    success: false,
                    failedStep: step.name,
                    error: error.message,
                    log: this.#sagaLog,
                };
            }
        }

        return { success: true, context, log: this.#sagaLog };
    }

    async #compensate(context) {
        const reversed = [...this.#completedSteps].reverse();
        for (const step of reversed) {
            try {
                this.#log(step.name, 'compensating');
                await step.compensate(context);
                this.#log(step.name, 'compensated');
            } catch (error) {
                this.#log(step.name, `compensation failed: ${error.message}`);
            }
        }
    }

    #log(step, message) {
        this.#sagaLog.push({
            step,
            message,
            timestamp: new Date().toISOString(),
        });
    }
}

// ব্যবহার — bKash Payment সহ Order Saga
const saga = new SagaOrchestrator();

saga
    .addStep('create_order',
        async (ctx) => {
            const order = await orderService.create(ctx.userId, ctx.items);
            return { orderId: order.id };
        },
        async (ctx) => orderService.cancel(ctx.orderId),
    )
    .addStep('charge_bkash',
        async (ctx) => {
            const txn = await bkashService.charge(ctx.userId, ctx.totalAmount);
            return { transactionId: txn.id };
        },
        async (ctx) => bkashService.refund(ctx.transactionId),
    )
    .addStep('reserve_inventory',
        async (ctx) => {
            await inventoryService.reserve(ctx.orderId, ctx.items);
            return {};
        },
        async (ctx) => inventoryService.release(ctx.orderId),
    )
    .addStep('schedule_delivery',
        async (ctx) => {
            const delivery = await deliveryService.schedule(ctx.orderId);
            return { deliveryId: delivery.id };
        },
        async (ctx) => deliveryService.cancel(ctx.deliveryId),
    );

const result = await saga.execute({
    userId: 'user-123',
    items: [{ productId: 'p1', quantity: 2 }],
    totalAmount: 1500.00,
});
```

---

### ৫. Circuit Breaker Pattern

Circuit Breaker downstream service ব্যর্থ হলে ক্রমাগত request পাঠানো বন্ধ করে। তিনটি state আছে:

```
    ┌────────────────────────────────────────────┐
    │                                            │
    │  CLOSED ──(failures > threshold)──► OPEN   │
    │    ▲                                  │    │
    │    │                          (timeout)│    │
    │    │                                  ▼    │
    │    └──(success)──── HALF-OPEN ◄───────┘    │
    │                        │                   │
    │                   (failure)──► OPEN         │
    └────────────────────────────────────────────┘
```

**CLOSED:** সব request pass হয়। Failure count track করা হয়।  
**OPEN:** সব request তাৎক্ষণিক fail/fallback। কোনো call downstream-এ যায় না।  
**HALF-OPEN:** সীমিত request পাঠানো হয় পরীক্ষার জন্য। সফল হলে CLOSED, ব্যর্থ হলে আবার OPEN।

**PHP — Circuit Breaker:**

```php
<?php
declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Cache;

enum CircuitState: string
{
    case CLOSED    = 'closed';
    case OPEN      = 'open';
    case HALF_OPEN = 'half_open';
}

final class CircuitBreaker
{
    private const CACHE_PREFIX = 'circuit_breaker:';

    public function __construct(
        private readonly string $serviceName,
        private readonly int $failureThreshold = 5,
        private readonly int $recoveryTimeout = 30,
        private readonly int $halfOpenMaxAttempts = 3,
    ) {}

    public function execute(\Closure $action, ?\Closure $fallback = null): mixed
    {
        $state = $this->getState();

        if ($state === CircuitState::OPEN) {
            if ($this->shouldAttemptReset()) {
                $this->transitionTo(CircuitState::HALF_OPEN);
            } else {
                return $fallback
                    ? $fallback()
                    : throw new CircuitOpenException($this->serviceName);
            }
        }

        try {
            $result = $action();
            $this->onSuccess();
            return $result;
        } catch (\Throwable $e) {
            $this->onFailure();

            if ($this->getFailureCount() >= $this->failureThreshold) {
                $this->transitionTo(CircuitState::OPEN);
            }

            return $fallback ? $fallback() : throw $e;
        }
    }

    private function getState(): CircuitState
    {
        $state = Cache::get(self::CACHE_PREFIX . "{$this->serviceName}:state", 'closed');
        return CircuitState::from($state);
    }

    private function transitionTo(CircuitState $state): void
    {
        Cache::put(
            self::CACHE_PREFIX . "{$this->serviceName}:state",
            $state->value,
            now()->addMinutes(5),
        );

        if ($state === CircuitState::OPEN) {
            Cache::put(
                self::CACHE_PREFIX . "{$this->serviceName}:opened_at",
                now()->timestamp,
                now()->addMinutes(5),
            );
        }
    }

    private function onSuccess(): void
    {
        Cache::put(self::CACHE_PREFIX . "{$this->serviceName}:failures", 0, now()->addMinutes(5));

        if ($this->getState() === CircuitState::HALF_OPEN) {
            $this->transitionTo(CircuitState::CLOSED);
        }
    }

    private function onFailure(): void
    {
        Cache::increment(self::CACHE_PREFIX . "{$this->serviceName}:failures");
    }

    private function getFailureCount(): int
    {
        return (int) Cache::get(self::CACHE_PREFIX . "{$this->serviceName}:failures", 0);
    }

    private function shouldAttemptReset(): bool
    {
        $openedAt = Cache::get(self::CACHE_PREFIX . "{$this->serviceName}:opened_at", 0);
        return (now()->timestamp - $openedAt) >= $this->recoveryTimeout;
    }
}

// ব্যবহার
$cb = new CircuitBreaker(serviceName: 'payment-service', failureThreshold: 5);

$result = $cb->execute(
    action: fn () => Http::timeout(3)->get('http://payment-svc/api/charge'),
    fallback: fn () => ['status' => 'queued', 'message' => 'Payment will be retried'],
);
```

**Node.js — Circuit Breaker:**

```javascript
// src/resilience/CircuitBreaker.js

class CircuitBreaker {
    static State = Object.freeze({
        CLOSED: 'closed',
        OPEN: 'open',
        HALF_OPEN: 'half_open',
    });

    #state = CircuitBreaker.State.CLOSED;
    #failureCount = 0;
    #successCount = 0;
    #lastFailureTime = null;
    #failureThreshold;
    #recoveryTimeout;
    #halfOpenMaxAttempts;

    constructor({
        failureThreshold = 5,
        recoveryTimeout = 30_000,
        halfOpenMaxAttempts = 3,
    } = {}) {
        this.#failureThreshold = failureThreshold;
        this.#recoveryTimeout = recoveryTimeout;
        this.#halfOpenMaxAttempts = halfOpenMaxAttempts;
    }

    get state() { return this.#state; }

    async execute(action, fallback) {
        if (this.#state === CircuitBreaker.State.OPEN) {
            if (this.#shouldAttemptReset()) {
                this.#state = CircuitBreaker.State.HALF_OPEN;
                this.#successCount = 0;
            } else {
                if (fallback) return fallback();
                throw new Error('Circuit is OPEN');
            }
        }

        try {
            const result = await action();
            this.#onSuccess();
            return result;
        } catch (error) {
            this.#onFailure();
            if (fallback) return fallback();
            throw error;
        }
    }

    #onSuccess() {
        this.#failureCount = 0;
        if (this.#state === CircuitBreaker.State.HALF_OPEN) {
            this.#successCount++;
            if (this.#successCount >= this.#halfOpenMaxAttempts) {
                this.#state = CircuitBreaker.State.CLOSED;
            }
        }
    }

    #onFailure() {
        this.#failureCount++;
        this.#lastFailureTime = Date.now();
        if (this.#failureCount >= this.#failureThreshold) {
            this.#state = CircuitBreaker.State.OPEN;
        }
    }

    #shouldAttemptReset() {
        return Date.now() - this.#lastFailureTime >= this.#recoveryTimeout;
    }
}

// ব্যবহার
const paymentBreaker = new CircuitBreaker({
    failureThreshold: 5,
    recoveryTimeout: 30_000,
});

const result = await paymentBreaker.execute(
    () => fetch('http://payment-svc/api/charge', { method: 'POST', body }),
    () => ({ status: 'queued', message: 'Payment will be retried' }),
);
```

---

### ৬. Service Mesh

Service Mesh হলো infrastructure layer যা service-to-service communication manage করে। প্রতিটি service-এর পাশে একটি sidecar proxy (Envoy) বসে যা traffic management, security (mTLS), ও observability handle করে। Application code-এ কোনো পরিবর্তন করতে হয় না।

```
┌──────────────────────────────────────────────┐
│              Service Mesh (Istio)             │
│                                              │
│  ┌─────────────────┐  ┌─────────────────┐   │
│  │  Order Service  │  │ Payment Service │   │
│  │  ┌───────────┐  │  │  ┌───────────┐  │   │
│  │  │ App Code  │  │  │  │ App Code  │  │   │
│  │  └─────┬─────┘  │  │  └─────┬─────┘  │   │
│  │  ┌─────▼─────┐  │  │  ┌─────▼─────┐  │   │
│  │  │  Envoy    │◄─┼──┼─►│  Envoy    │  │   │
│  │  │  Sidecar  │  │  │  │  Sidecar  │  │   │
│  │  └───────────┘  │  │  └───────────┘  │   │
│  └─────────────────┘  └─────────────────┘   │
│                                              │
│  Control Plane: Istiod (config, certs, etc.) │
└──────────────────────────────────────────────┘
```

**মূল সুবিধা:**
- **mTLS:** Service-এর মধ্যে encrypted communication, zero-trust security
- **Traffic Management:** Canary deployment, A/B testing, rate limiting
- **Observability:** Automatic metrics, tracing, logging — কোডে কিছু লিখতে হয় না

---

### ৭. Database per Service ও CQRS

প্রতিটি microservice-এর নিজস্ব database থাকা বাধ্যতামূলক। Shared database হলো anti-pattern কারণ এটি tight coupling তৈরি করে।

#### CQRS (Command Query Responsibility Segregation)

```
                    ┌──────────────┐
    Write ─────────►│ Command Side │──────► Write DB (PostgreSQL)
    (Create/Update) │  (Master)    │            │
                    └──────────────┘            │ Event Stream
                                                ▼
                    ┌──────────────┐       ┌──────────┐
    Read ──────────►│  Query Side  │◄──────│ Read DB  │
    (Search/List)   │  (Replica)   │       │(Elastic/ │
                    └──────────────┘       │ Redis)   │
                                           └──────────┘
```

**PHP — CQRS Implementation:**

```php
<?php
declare(strict_types=1);

namespace App\CQRS;

// Command side
interface Command {}

final readonly class CreateOrderCommand implements Command
{
    public function __construct(
        public string $userId,
        public array $items,
        public float $totalAmount,
    ) {}
}

final class CreateOrderHandler
{
    public function __construct(
        private readonly OrderWriteRepository $writeRepo,
        private readonly EventBus $eventBus,
    ) {}

    public function handle(CreateOrderCommand $command): string
    {
        $order = Order::create(
            userId: $command->userId,
            items: $command->items,
            total: $command->totalAmount,
        );

        $this->writeRepo->save($order);

        // Event publish করো — read side update হবে
        $this->eventBus->publish(new OrderCreatedEvent(
            orderId: $order->id,
            data: $order->toArray(),
        ));

        return $order->id;
    }
}

// Query side — আলাদা optimized read model
final readonly class OrderQueryService
{
    public function __construct(
        private \Elasticsearch\Client $elastic,
    ) {}

    public function search(string $query, array $filters = []): array
    {
        $result = $this->elastic->search([
            'index' => 'orders',
            'body'  => [
                'query' => [
                    'bool' => [
                        'must'   => [['match' => ['_all' => $query]]],
                        'filter' => $this->buildFilters($filters),
                    ],
                ],
            ],
        ]);

        return array_map(
            fn ($hit) => $hit['_source'],
            $result['hits']['hits'],
        );
    }

    private function buildFilters(array $filters): array
    {
        return array_map(
            fn ($key, $value) => ['term' => [$key => $value]],
            array_keys($filters),
            array_values($filters),
        );
    }
}

// Event Handler — Read model update
final class OrderProjection
{
    public function __construct(private \Elasticsearch\Client $elastic) {}

    public function onOrderCreated(OrderCreatedEvent $event): void
    {
        $this->elastic->index([
            'index' => 'orders',
            'id'    => $event->orderId,
            'body'  => $event->data,
        ]);
    }
}
```

---

### ৮. Distributed Tracing

Microservices-এ একটি request অনেকগুলো service-এর মধ্য দিয়ে যায়। কোথায় bottleneck বা error হচ্ছে তা বোঝার জন্য distributed tracing অত্যাবশ্যক। OpenTelemetry হলো industry standard।

**PHP — OpenTelemetry Tracing:**

```php
<?php
declare(strict_types=1);

namespace App\Tracing;

use OpenTelemetry\API\Trace\TracerInterface;
use OpenTelemetry\API\Trace\SpanKind;
use OpenTelemetry\SDK\Trace\TracerProvider;
use OpenTelemetry\Contrib\Jaeger\Exporter as JaegerExporter;

final class TracingService
{
    private TracerInterface $tracer;

    public function __construct()
    {
        $exporter = new JaegerExporter('order-service', 'http://jaeger:14268');
        $provider = new TracerProvider($exporter);
        $this->tracer = $provider->getTracer('order-service', '1.0.0');
    }

    public function startSpan(
        string $name,
        SpanKind $kind = SpanKind::KIND_INTERNAL,
        array $attributes = [],
    ): \OpenTelemetry\API\Trace\SpanInterface {
        $span = $this->tracer
            ->spanBuilder($name)
            ->setSpanKind($kind)
            ->startSpan();

        foreach ($attributes as $key => $value) {
            $span->setAttribute($key, $value);
        }

        return $span;
    }

    public function traceHttpRequest(string $method, string $url, \Closure $action): mixed
    {
        $span = $this->startSpan(
            name: "{$method} {$url}",
            kind: SpanKind::KIND_CLIENT,
            attributes: [
                'http.method' => $method,
                'http.url'    => $url,
            ],
        );

        try {
            $result = $action($span);
            $span->setAttribute('http.status_code', 200);
            return $result;
        } catch (\Throwable $e) {
            $span->recordException($e);
            $span->setAttribute('http.status_code', 500);
            throw $e;
        } finally {
            $span->end();
        }
    }
}

// Middleware — Correlation ID ও Trace Context propagation
final class TracingMiddleware
{
    public function handle(Request $request, \Closure $next)
    {
        $correlationId = $request->header('X-Correlation-ID', (string) Str::uuid());
        $request->headers->set('X-Correlation-ID', $correlationId);

        // Log context-এ correlation ID add করো
        Log::shareContext(['correlation_id' => $correlationId]);

        $response = $next($request);
        $response->headers->set('X-Correlation-ID', $correlationId);

        return $response;
    }
}
```

**Node.js — OpenTelemetry:**

```javascript
// src/tracing/setup.js
import { NodeSDK } from '@opentelemetry/sdk-node';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';
import { trace, SpanKind, context, propagation } from '@opentelemetry/api';

const sdk = new NodeSDK({
    serviceName: 'order-service',
    traceExporter: new JaegerExporter({
        endpoint: 'http://jaeger:14268/api/traces',
    }),
    instrumentations: [
        new HttpInstrumentation(),
        new ExpressInstrumentation(),
    ],
});

sdk.start();

// Custom tracing helper
class Tracer {
    static #tracer = trace.getTracer('order-service', '1.0.0');

    static async withSpan(name, attributes, fn) {
        return Tracer.#tracer.startActiveSpan(name, {
            kind: SpanKind.INTERNAL,
            attributes,
        }, async (span) => {
            try {
                const result = await fn(span);
                span.setStatus({ code: 1 }); // OK
                return result;
            } catch (error) {
                span.recordException(error);
                span.setStatus({ code: 2, message: error.message });
                throw error;
            } finally {
                span.end();
            }
        });
    }
}

// Middleware — correlation ID propagation
function correlationMiddleware(req, res, next) {
    req.correlationId = req.headers['x-correlation-id'] ?? crypto.randomUUID();
    res.setHeader('X-Correlation-ID', req.correlationId);
    next();
}

export { Tracer, correlationMiddleware, sdk };
```

---

### ৯. Strangler Fig Pattern (Migration)

Monolith থেকে microservices-এ একবারে migrate করা ঝুঁকিপূর্ণ। Strangler Fig pattern-এ ধীরে ধীরে monolith-এর functionality microservices-এ সরানো হয়, যতক্ষণ না monolith সম্পূর্ণ replace হয়।

```
Phase 1: Facade/Proxy বসাও
┌────────────────────────────────────────┐
│  Client → Proxy/Facade → Monolith     │
└────────────────────────────────────────┘

Phase 2: একটি feature microservice-এ সরাও
┌────────────────────────────────────────┐
│            ┌──► User Microservice     │
│  Client → Proxy                       │
│            └──► Monolith (বাকি সব)    │
└────────────────────────────────────────┘

Phase 3: আরো feature সরাও
┌────────────────────────────────────────┐
│            ┌──► User Microservice     │
│  Client → Proxy──► Order Microservice │
│            └──► Monolith (শুধু legacy) │
└────────────────────────────────────────┘

Phase 4: Monolith সম্পূর্ণ retire
┌────────────────────────────────────────┐
│         ┌──► User Microservice        │
│  Client → Gateway──► Order Service    │
│         └──► Payment Service          │
└────────────────────────────────────────┘
```

**PHP Laravel — Strangler Proxy Middleware:**

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use Illuminate\Support\Facades\Http;

final class StranglerProxyMiddleware
{
    // কোন route microservice-এ গেছে তার mapping
    private array $migratedRoutes = [
        '/api/users/*'        => 'http://user-service:8001',
        '/api/notifications/*' => 'http://notification-service:8004',
    ];

    public function handle(Request $request, \Closure $next)
    {
        $targetService = $this->findMigratedService($request->path());

        if ($targetService) {
            // Microservice-এ forward করো
            $response = Http::withHeaders($request->headers->all())
                ->send(
                    $request->method(),
                    $targetService . '/' . $request->path(),
                    ['body' => $request->getContent()]
                );

            return response($response->body(), $response->status())
                ->withHeaders($response->headers());
        }

        // Monolith-এ handle করো (এখনও migrate হয়নি)
        return $next($request);
    }

    private function findMigratedService(string $path): ?string
    {
        foreach ($this->migratedRoutes as $pattern => $service) {
            if (fnmatch($pattern, "/{$path}")) {
                return $service;
            }
        }
        return null;
    }
}
```

---

## 🐳 Deployment Patterns

### Docker Compose — Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  api-gateway:
    build: ./gateway
    ports: ["3000:3000"]
    depends_on: [user-service, order-service, rabbitmq]
    environment:
      - JWT_SECRET=super-secret-key
    networks: [microservices]

  user-service:
    build: ./services/user
    environment:
      - DB_HOST=user-db
      - DB_NAME=users
      - CONSUL_HOST=consul
    depends_on: [user-db, consul]
    networks: [microservices]

  order-service:
    build: ./services/order
    environment:
      - MONGO_URL=mongodb://order-db:27017/orders
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on: [order-db, rabbitmq]
    networks: [microservices]

  payment-service:
    build: ./services/payment
    environment:
      - DB_HOST=payment-db
      - BKASH_API_KEY=${BKASH_API_KEY}
    depends_on: [payment-db, rabbitmq]
    networks: [microservices]

  # Databases — প্রতি service-এ আলাদা
  user-db:
    image: postgres:16
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: secret
    volumes: [user-data:/var/lib/postgresql/data]
    networks: [microservices]

  order-db:
    image: mongo:7
    volumes: [order-data:/data/db]
    networks: [microservices]

  payment-db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: payments
      MYSQL_ROOT_PASSWORD: secret
    networks: [microservices]

  # Infrastructure
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["15672:15672"]
    networks: [microservices]

  consul:
    image: consul:1.15
    ports: ["8500:8500"]
    networks: [microservices]

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: ["16686:16686", "14268:14268"]
    networks: [microservices]

networks:
  microservices:
    driver: bridge

volumes:
  user-data:
  order-data:
```

### Kubernetes Deployment

```yaml
# k8s/order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        sidecar.istio.io/inject: "true"  # Istio sidecar inject
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.2.0
          ports:
            - containerPort: 8002
          env:
            - name: MONGO_URL
              valueFrom:
                secretKeyRef:
                  name: order-secrets
                  key: mongo-url
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8002
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8002
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8002
      targetPort: 8002
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## ✅ সুবিধা ও ❌ অসুবিধা

| দিক | ✅ সুবিধা | ❌ অসুবিধা |
|------|-----------|------------|
| **Scaling** | Individual service scale করা যায় | Infrastructure complexity অনেক বেশি |
| **Deployment** | Independent deployment, zero-downtime | CI/CD pipeline প্রতি service-এ আলাদা |
| **Technology** | Polyglot — প্রতি service-এ ভিন্ন stack | Team-কে অনেক technology জানতে হয় |
| **Fault Isolation** | একটি service fail হলে বাকিরা চলে | Distributed system debugging কঠিন |
| **Team** | ছোট team স্বাধীনভাবে কাজ করতে পারে | Inter-team communication overhead |
| **Data** | Service-specific data optimization | Data consistency চ্যালেঞ্জিং (eventual) |
| **Development** | দ্রুত feature delivery | Initial setup ও learning curve বেশি |
| **Testing** | Unit test সহজ | Integration ও E2E test জটিল |
| **Security** | Granular security per service | Service-to-service auth (mTLS) দরকার |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Distributed Monolith
**সমস্যা:** Service-গুলো নামে আলাদা কিন্তু tightly coupled — একটি deploy করতে হলে বাকিগুলোও deploy করতে হয়।  
**সমাধান:** Service boundary সঠিকভাবে define করুন। Domain-Driven Design (DDD) ব্যবহার করুন।

### ২. Shared Database
**সমস্যা:** একাধিক service একই database ব্যবহার করে।  
**সমাধান:** Database per service pattern অনুসরণ করুন। প্রয়োজনে data duplication মেনে নিন।

### ৩. Synchronous Chain Calls
**সমস্যা:** Service A → B → C → D synchronously call করে। একটি slow হলে সব slow হয়।  
**সমাধান:** Asynchronous messaging ব্যবহার করুন। যেখানে sync দরকার, সেখানে timeout ও circuit breaker রাখুন।

### ৪. Circuit Breaker না রাখা
**সমস্যা:** Downstream service fail হলে cascade failure ঘটে — পুরো system crash করে।  
**সমাধান:** প্রতিটি external call-এ circuit breaker, timeout, ও retry with backoff ব্যবহার করুন।

### ৫. Nano-services (অতি ছোট Service)
**সমস্যা:** খুব ছোট ছোট service তৈরি করা — operational overhead সুবিধার চেয়ে বেশি হয়ে যায়।  
**সমাধান:** Bounded Context অনুযায়ী service size ঠিক করুন। একটি team যদি একটি service manage করতে পারে তাহলে সেটা সঠিক size।

### ৬. সকল Communication Synchronous রাখা
**সমস্যা:** সবকিছু REST call দিয়ে করলে latency বাড়ে, coupling তৈরি হয়।  
**সমাধান:** Event-driven architecture adopt করুন। যেখানে immediate response দরকার নেই, message queue ব্যবহার করুন।

---

## 🧪 টেস্টিং মাইক্রোসার্ভিস

Microservices-এ testing strategy multi-layered হতে হবে:

```
         ┌───────────────┐
         │   E2E Tests   │  ← সবচেয়ে কম, সবচেয়ে ধীর
         ├───────────────┤
         │Contract Tests │  ← Service boundary verify
         ├───────────────┤
         │Integration    │  ← DB, Queue সহ test
         ├───────────────┤
         │  Unit Tests   │  ← সবচেয়ে বেশি, সবচেয়ে দ্রুত
         └───────────────┘
```

### Consumer-Driven Contract Testing (Pact)

**PHP — Provider Test:**

```php
<?php
// tests/Contract/UserServiceProviderTest.php

declare(strict_types=1);

namespace Tests\Contract;

use PHPUnit\Framework\TestCase;
use PhpPact\Standalone\ProviderVerifier\Model\VerifierConfig;
use PhpPact\Standalone\ProviderVerifier\Verifier;

final class UserServiceProviderTest extends TestCase
{
    public function testPactWithOrderService(): void
    {
        $config = new VerifierConfig();
        $config
            ->setProviderName('UserService')
            ->setProviderBaseUrl('http://localhost:8001')
            ->setPactBrokerUri('http://pact-broker:9292')
            ->setConsumerName('OrderService')
            ->setPublishResults(true)
            ->setProviderVersion('1.0.0');

        $verifier = new Verifier($config);
        $verifier->verifyProvider();

        $this->assertTrue(true, 'Pact verification passed');
    }
}
```

**Node.js — Consumer Test:**

```javascript
// tests/contract/orderConsumer.pact.test.js
import { PactV3 } from '@pact-foundation/pact';
import { describe, it, expect } from 'vitest';
import { UserClient } from '../../src/clients/UserClient.js';

const provider = new PactV3({
    consumer: 'OrderService',
    provider: 'UserService',
});

describe('Order Service → User Service Contract', () => {
    it('should return user details by ID', async () => {
        // Consumer-এর expectation define করো
        provider
            .given('user with ID user-123 exists')
            .uponReceiving('a request for user details')
            .withRequest({
                method: 'GET',
                path: '/api/users/user-123',
                headers: { Accept: 'application/json' },
            })
            .willRespondWith({
                status: 200,
                headers: { 'Content-Type': 'application/json' },
                body: {
                    id: 'user-123',
                    name: 'Rahim Uddin',
                    email: 'rahim@example.com',
                },
            });

        await provider.executeTest(async (mockServer) => {
            const client = new UserClient(mockServer.url);
            const user = await client.getUser('user-123');

            expect(user.id).toBe('user-123');
            expect(user.name).toBe('Rahim Uddin');
            expect(user.email).toContain('@');
        });
    });
});
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:
- **বড় team (20+ developer)** — ছোট team independently কাজ করতে পারবে
- **Complex domain** — বিভিন্ন business capability স্পষ্টভাবে আলাদা করা যায়
- **Scale প্রয়োজন** — নির্দিষ্ট component বেশি load পায় (যেমন Pathao-র ride matching)
- **Polyglot দরকার** — বিভিন্ন service-এ ভিন্ন technology ব্যবহার করতে চান
- **Rapid deployment** — প্রতিদিন একাধিকবার deploy করতে চান

### ❌ ব্যবহার করবেন না যখন:
- **ছোট team (< 5-8 জন)** — operational overhead manage করা কঠিন হবে
- **সহজ application** — CRUD app-এর জন্য monolith যথেষ্ট
- **Startup MVP** — প্রথমে monolith বানান, পরে প্রয়োজনে migrate করুন
- **Domain unclear** — Business boundary পরিষ্কার না হলে ভুল boundary তৈরি হবে
- **DevOps maturity কম** — CI/CD, container orchestration ছাড়া microservices manage করা অসম্ভব

### Conway's Law বিবেচনা
> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations."

Team structure microservices boundary-র সাথে মিলতে হবে। যদি তিনটি team থাকে, তাহলে তিনটি logical service group হওয়া উচিত। Team structure ও architecture mismatch হলে communication overhead বিপুল হয়ে যায়।

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | সম্পর্ক |
|-----------|---------|
| **API Gateway** | Microservices-এর সামনে বসে single entry point হিসেবে |
| **Circuit Breaker** | Service failure resilience-এর জন্য অপরিহার্য |
| **Saga Pattern** | Distributed transaction management |
| **CQRS** | Read/Write আলাদা করে performance optimize |
| **Event Sourcing** | State change history track ও service sync |
| **Service Mesh** | Infrastructure-level communication management |
| **Strangler Fig** | Monolith থেকে microservices-এ migration strategy |
| **Sidecar Pattern** | Cross-cutting concerns (logging, monitoring) handle |
| **BFF (Backend for Frontend)** | প্রতিটি client type-এর জন্য আলাদা backend layer |
| **Domain-Driven Design** | Bounded Context দিয়ে service boundary নির্ধারণ |

---

## 📋 সারসংক্ষেপ

```
Microservices Architecture মনে রাখার সূত্র:

🔹 প্রতিটি service একটি business capability — Single Responsibility
🔹 নিজস্ব database — Data isolation
🔹 Independently deployable — স্বাধীনভাবে deploy
🔹 Lightweight communication — REST/gRPC/Message Queue
🔹 Failure isolation — Circuit Breaker, Retry, Fallback
🔹 Observability — Tracing, Logging, Metrics
🔹 Automation — CI/CD, Container, Orchestration

শুরু করুন:
  ১. Domain analysis (DDD Bounded Context)
  ২. Monolith-first approach
  ৩. ধীরে ধীরে Strangler Fig দিয়ে extract
  ৪. Service Mesh ও observability যোগ করুন
  ৫. Team structure align করুন (Conway's Law)

⚠️ মনে রাখবেন: Microservices কোনো silver bullet নয়।
   এটি complexity trade-off — distributed system-এর
   জটিলতা বিনিময়ে flexibility ও scalability পাওয়া।
```

> **"Start with a monolith, and extract microservices as you learn your domain."** — Sam Newman

---

*এই document-টি senior engineer-দের জন্য তৈরি। Production-এ microservices implement করার আগে অবশ্যই আপনার team-এর DevOps maturity, domain knowledge, এবং organizational readiness মূল্যায়ন করুন।*
