# 📌 OpenTelemetry (OTel) — ওপেনটেলিমেট্রি কমপ্লিট গাইড

## 📖 সূচিপত্র
- [ওপেনটেলিমেট্রি কী?](#-ওপেনটেলিমেট্রি-কী)
- [তিনটি পিলার একত্রীকরণ](#-তিনটি-পিলার-একত্রীকরণ)
- [OTel আর্কিটেকচার](#-otel-আর্কিটেকচার)
- [অটো-ইন্সট্রুমেন্টেশন](#-অটো-ইন্সট্রুমেন্টেশন)
- [ম্যানুয়াল ইন্সট্রুমেন্টেশন](#-ম্যানুয়াল-ইন্সট্রুমেন্টেশন)
- [OTel Collector কনফিগারেশন](#-otel-collector-কনফিগারেশন)
- [ভেন্ডর-অ্যাগনস্টিক পদ্ধতি](#-ভেন্ডর-অ্যাগনস্টিক-পদ্ধতি)
- [PHP OpenTelemetry SDK](#-php-opentelemetry-sdk)
- [Node.js OpenTelemetry](#-nodejs-opentelemetry)
- [Jaeger, Prometheus, Grafana ইন্টিগ্রেশন](#-jaeger-prometheus-grafana-ইন্টিগ্রেশন)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📖 ওপেনটেলিমেট্রি কী?

OpenTelemetry (সংক্ষেপে OTel) হলো একটি **ওপেন-সোর্স observability ফ্রেমওয়ার্ক** যা
traces, metrics, এবং logs — এই তিনটি observability সিগন্যালকে **একটি মানদণ্ডে (standard)**
একত্রিত করে। এটি CNCF (Cloud Native Computing Foundation) এর দ্বিতীয় সবচেয়ে সক্রিয়
প্রজেক্ট (Kubernetes এর পরে)।

```
ধরুন bKash এর মাইক্রোসার্ভিস আর্কিটেকচারে ২০টি সার্ভিস আছে।
আগে প্রতিটি সার্ভিসে আলাদা আলাদা টুল ব্যবহার করতে হতো:
  - Traces → Jaeger SDK
  - Metrics → Prometheus client
  - Logs → Fluentd/ELK

OTel এসে বলল: "একটাই SDK দিয়ে তিনটাই পাঠাও!"
```

### আগে vs এখন

```
 ╔═══════════════════════════════════════════════════════╗
 ║              আগে (Before OTel)                       ║
 ╠═══════════════════════════════════════════════════════╣
 ║                                                       ║
 ║  App ──► Jaeger SDK ──────────► Jaeger                ║
 ║   │                                                   ║
 ║   ├──► Prometheus Client ─────► Prometheus             ║
 ║   │                                                   ║
 ║   └──► Fluentd Agent ────────► Elasticsearch           ║
 ║                                                       ║
 ║  সমস্যা: ৩টি আলাদা SDK, ৩টি আলাদা কনফিগ,           ║
 ║          ৩টি আলাদা প্রোটোকল, কোনো correlation নেই   ║
 ╚═══════════════════════════════════════════════════════╝

 ╔═══════════════════════════════════════════════════════╗
 ║              এখন (With OTel)                          ║
 ╠═══════════════════════════════════════════════════════╣
 ║                                                       ║
 ║                    ┌──────────► Jaeger                 ║
 ║                    │                                   ║
 ║  App ──► OTel ─────┼──────────► Prometheus             ║
 ║          SDK       │                                   ║
 ║                    └──────────► Elasticsearch           ║
 ║                                                       ║
 ║  সমাধান: ১টি SDK, ১টি কনফিগ, OTLP প্রোটোকল,        ║
 ║          trace-metric-log সব correlated               ║
 ╚═══════════════════════════════════════════════════════╝
```

---

## 📊 তিনটি পিলার একত্রীকরণ

OpenTelemetry তিনটি observability সিগন্যালকে **একটি ইউনিফাইড মডেলে** নিয়ে আসে।
প্রতিটি সিগন্যালে একই `trace_id` এবং `span_id` থাকায় এরা পরস্পর **correlated**।

```
 ┌─────────────────────────────────────────────────────────────┐
 │                  OpenTelemetry Unified Model                 │
 ├─────────────────────────────────────────────────────────────┤
 │                                                             │
 │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
 │   │   TRACES      │  │   METRICS    │  │    LOGS      │     │
 │   │              │  │              │  │              │     │
 │   │  কী হলো?     │  │  কতটুকু?     │  │  কেন হলো?    │     │
 │   │  কতক্ষণ?     │  │  কত বার?     │  │  কী error?   │     │
 │   │  কোন পথে?    │  │  কত দ্রুত?   │  │  কী context? │     │
 │   │              │  │              │  │              │     │
 │   │ trace_id: abc│  │ trace_id: abc│  │ trace_id: abc│     │
 │   │ span_id: 123 │  │ exemplar: 123│  │ span_id: 123 │     │
 │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
 │          │                 │                 │              │
 │          └────────┬────────┴────────┬────────┘              │
 │                   │                 │                        │
 │              trace_id দিয়ে সব connected!                    │
 │                                                             │
 └─────────────────────────────────────────────────────────────┘
```

### বাস্তব উদাহরণ — Pathao রাইড রিকোয়েস্ট

```
Trace:  User ──► API Gateway ──► Ride Service ──► Driver Match ──► Payment
        |__________________________________________________________|
                          trace_id: "abc-123"

Metric: ride.request.count = 1
        ride.request.duration = 2.3s
        (exemplar trace_id: "abc-123" — click করলে trace দেখায়!)

Log:    [INFO] trace_id=abc-123 span_id=xyz-456
        "Driver matched: driver_id=D-789, ETA=5min"
```

---

## 🏠 OTel আর্কিটেকচার

OpenTelemetry চারটি মূল কম্পোনেন্ট নিয়ে গঠিত:

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                    OTel Architecture Overview                    │
 ├──────────────────────────────────────────────────────────────────┤
 │                                                                  │
 │  ┌─────────────────────────────────────────┐                     │
 │  │         YOUR APPLICATION                 │                     │
 │  │                                         │                     │
 │  │  ┌──────────┐    ┌──────────────────┐   │                     │
 │  │  │ OTel API │───►│   OTel SDK       │   │                     │
 │  │  │          │    │                  │   │                     │
 │  │  │ (ইন্টার- │    │ • TracerProvider │   │                     │
 │  │  │  ফেস)    │    │ • MeterProvider  │   │                     │
 │  │  │          │    │ • LoggerProvider │   │                     │
 │  │  │          │    │ • Sampler        │   │                     │
 │  │  │          │    │ • SpanProcessor  │   │                     │
 │  │  └──────────┘    │ • Exporter       │   │                     │
 │  │                  └────────┬─────────┘   │                     │
 │  └───────────────────────────┼─────────────┘                     │
 │                              │ OTLP (gRPC/HTTP)                  │
 │                              ▼                                   │
 │  ┌─────────────────────────────────────────┐                     │
 │  │         OTel COLLECTOR                   │                     │
 │  │                                         │                     │
 │  │  Receivers ──► Processors ──► Exporters │                     │
 │  │                                         │                     │
 │  │  • OTLP        • Batch       • Jaeger   │                     │
 │  │  • Prometheus   • Filter      • Prometheus│                    │
 │  │  • Kafka        • Attributes  • OTLP     │                     │
 │  │  • Zipkin       • Tail Sample • Loki     │                     │
 │  └──────────┬──────────┬──────────┬────────┘                     │
 │             │          │          │                               │
 │             ▼          ▼          ▼                               │
 │         ┌───────┐ ┌────────┐ ┌────────┐                          │
 │         │Jaeger │ │Promethe│ │Grafana │                          │
 │         │       │ │us      │ │Loki    │                          │
 │         └───────┘ └────────┘ └────────┘                          │
 │                    BACKENDS                                      │
 └──────────────────────────────────────────────────────────────────┘
```

### কম্পোনেন্ট ব্যাখ্যা

| কম্পোনেন্ট | ভূমিকা | উদাহরণ |
|---|---|---|
| **API** | ইন্সট্রুমেন্টেশন ইন্টারফেস, vendor-neutral | `tracer.startSpan()` |
| **SDK** | API এর implementation, কনফিগারেশন, processing | `TracerProvider`, `Sampler` |
| **Collector** | টেলিমেট্রি receive, process, export করে | OTel Collector binary |
| **Exporter** | নির্দিষ্ট backend এ ডেটা পাঠায় | Jaeger, Prometheus, OTLP |

### Resource ও Semantic Conventions

Resource হলো আপনার সার্ভিসের **পরিচয়**। Semantic Conventions হলো **নামকরণের মানদণ্ড**।

```
Resource Attributes (সার্ভিসের পরিচয়):
┌──────────────────────────────────────────┐
│  service.name      = "bkash-payment"     │
│  service.version   = "2.1.0"            │
│  deployment.env    = "production"        │
│  host.name         = "prod-server-03"   │
│  cloud.provider    = "aws"              │
│  cloud.region      = "ap-southeast-1"   │
└──────────────────────────────────────────┘

Semantic Conventions (নামকরণ):
┌──────────────────────────────────────────┐
│  http.request.method  = "POST"           │
│  http.response.status_code = 200         │
│  url.path             = "/api/payment"   │
│  db.system            = "mysql"          │
│  db.statement         = "SELECT ..."     │
│  messaging.system     = "rabbitmq"       │
└──────────────────────────────────────────┘
```

bKash এর প্রতিটি সার্ভিস যখন telemetry পাঠায়, Resource Attributes দিয়ে বোঝা যায়
এটা কোন সার্ভিস, কোন ভার্সন, কোন environment থেকে এসেছে। Semantic Conventions
ব্যবহার করলে সব সার্ভিসের attribute name **একই রকম** থাকে — ফলে Grafana dashboard
তৈরি করা সহজ হয়।

---

## ⚡ অটো-ইন্সট্রুমেন্টেশন

অটো-ইন্সট্রুমেন্টেশন হলো **কোড না লিখেই** আপনার অ্যাপ্লিকেশনে observability যোগ করা।
OTel এর instrumentation library গুলো popular ফ্রেমওয়ার্ক (Laravel, Express, MySQL, Redis)
স্বয়ংক্রিয়ভাবে instrument করে।

```
 ┌──────────────────────────────────────────────────────────────┐
 │             Auto-Instrumentation Flow                        │
 ├──────────────────────────────────────────────────────────────┤
 │                                                              │
 │  ┌──────────────────────────────────────────┐                │
 │  │          আপনার অ্যাপ্লিকেশন              │                │
 │  │  (কোনো OTel কোড নেই)                    │                │
 │  └──────────────────┬───────────────────────┘                │
 │                     │ runtime এ inject হয়                   │
 │                     ▼                                        │
 │  ┌──────────────────────────────────────────┐                │
 │  │     Auto-Instrumentation Agent           │                │
 │  │                                          │                │
 │  │  ┌────────────┐  ┌────────────────┐      │                │
 │  │  │ HTTP hooks  │  │ DB query hooks │      │                │
 │  │  └────────────┘  └────────────────┘      │                │
 │  │  ┌────────────┐  ┌────────────────┐      │                │
 │  │  │ Redis hooks │  │ Queue hooks    │      │                │
 │  │  └────────────┘  └────────────────┘      │                │
 │  └──────────────────┬───────────────────────┘                │
 │                     │ স্বয়ংক্রিয়ভাবে spans/metrics তৈরি   │
 │                     ▼                                        │
 │  ┌──────────────────────────────────────────┐                │
 │  │         OTel Collector                   │                │
 │  └──────────────────────────────────────────┘                │
 │                                                              │
 │  ✅ কোনো কোড পরিবর্তন দরকার নেই!                           │
 │  ✅ HTTP, DB, Cache, Queue — সব ক্যাপচার হয়                │
 │  ❌ কিন্তু custom business logic spans পাবেন না             │
 └──────────────────────────────────────────────────────────────┘
```

### PHP অটো-ইন্সট্রুমেন্টেশন সেটআপ

```bash
# Step 1: OTel PHP extension ইনস্টল করুন
pecl install opentelemetry

# Step 2: php.ini এ enable করুন
# php.ini:
extension=opentelemetry.so

# Step 3: Composer packages ইনস্টল করুন
composer require \
  open-telemetry/sdk \
  open-telemetry/exporter-otlp \
  open-telemetry/auto-laravel \
  open-telemetry/auto-pdo \
  open-telemetry/auto-curl

# Step 4: Environment variables সেট করুন
export OTEL_SERVICE_NAME="daraz-product-service"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_PHP_AUTOLOAD_ENABLED=true
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp

# এখন Laravel অ্যাপ চালান — সব HTTP request, DB query,
# cache call স্বয়ংক্রিয়ভাবে trace হবে!
php artisan serve
```

### Node.js অটো-ইন্সট্রুমেন্টেশন সেটআপ

```bash
# Step 1: Packages ইনস্টল করুন
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http
```

```javascript
// tracing.js — এই ফাইলটি সবার আগে লোড হবে
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-http');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');

const sdk = new NodeSDK({
    serviceName: 'pathao-ride-service',
    traceExporter: new OTLPTraceExporter({
        url: 'http://otel-collector:4318/v1/traces',
    }),
    metricReader: new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter({
            url: 'http://otel-collector:4318/v1/metrics',
        }),
        exportIntervalMillis: 15000,
    }),
    instrumentations: [
        getNodeAutoInstrumentations({
            // Express, HTTP, MySQL, Redis, gRPC সব auto-instrument হবে
            '@opentelemetry/instrumentation-express': { enabled: true },
            '@opentelemetry/instrumentation-http': { enabled: true },
            '@opentelemetry/instrumentation-mysql2': { enabled: true },
            '@opentelemetry/instrumentation-redis-4': { enabled: true },
        }),
    ],
});

sdk.start();
console.log('🔭 OTel auto-instrumentation started for Pathao Ride Service');

process.on('SIGTERM', () => {
    sdk.shutdown().then(() => process.exit(0));
});
```

```bash
# --require flag দিয়ে সবার আগে tracing.js লোড করুন
node --require ./tracing.js app.js

# অথবা NODE_OPTIONS দিয়ে:
export NODE_OPTIONS="--require ./tracing.js"
node app.js
```

---

## 🔧 ম্যানুয়াল ইন্সট্রুমেন্টেশন

অটো-ইন্সট্রুমেন্টেশন সব কিছু ধরতে পারে না। **Business logic** এর জন্য manual
instrumentation প্রয়োজন। যেমন — bKash এ "টাকা ট্রান্সফার" একটি business operation,
এটা শুধু HTTP request বা DB query না।

### PHP ম্যানুয়াল ইন্সট্রুমেন্টেশন

```php
<?php
// bKash Money Transfer — Manual Instrumentation

use OpenTelemetry\API\Globals;
use OpenTelemetry\API\Trace\SpanKind;
use OpenTelemetry\API\Trace\StatusCode;

class MoneyTransferService
{
    private $tracer;
    private $meter;

    public function __construct()
    {
        // Tracer ও Meter পাওয়া — global provider থেকে
        $this->tracer = Globals::tracerProvider()
            ->getTracer('bkash-transfer-service', '1.0.0');
        $this->meter = Globals::meterProvider()
            ->getMeter('bkash-transfer-service', '1.0.0');
    }

    public function transfer(string $from, string $to, float $amount): bool
    {
        // Custom span তৈরি করুন — এটাই manual instrumentation
        $span = $this->tracer->spanBuilder('money.transfer')
            ->setSpanKind(SpanKind::KIND_INTERNAL)
            ->setAttribute('transfer.from', $from)
            ->setAttribute('transfer.to', $to)
            ->setAttribute('transfer.amount', $amount)
            ->setAttribute('transfer.currency', 'BDT')
            ->startSpan();

        $scope = $span->activate();

        try {
            // Step 1: Balance check
            $this->checkBalance($from, $amount);

            // Step 2: Debit
            $this->debit($from, $amount);

            // Step 3: Credit
            $this->credit($to, $amount);

            // Span event — গুরুত্বপূর্ণ ঘটনা লগ করুন
            $span->addEvent('transfer.completed', [
                'transfer.transaction_id' => uniqid('TXN-'),
            ]);

            $span->setStatus(StatusCode::STATUS_OK);

            // Custom metric — ট্রান্সফার count ও amount ট্র্যাক করুন
            $transferCounter = $this->meter->createCounter(
                'bkash.transfer.count',
                'transfers',
                'Number of money transfers'
            );
            $transferCounter->add(1, ['status' => 'success']);

            $amountHistogram = $this->meter->createHistogram(
                'bkash.transfer.amount',
                'BDT',
                'Transfer amounts'
            );
            $amountHistogram->record($amount, ['currency' => 'BDT']);

            return true;

        } catch (\Exception $e) {
            // Error তেও span এ রেকর্ড করুন
            $span->recordException($e);
            $span->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());

            $transferCounter = $this->meter->createCounter(
                'bkash.transfer.count',
                'transfers',
                'Number of money transfers'
            );
            $transferCounter->add(1, ['status' => 'failed']);

            throw $e;

        } finally {
            $scope->detach();
            $span->end();
        }
    }

    private function checkBalance(string $account, float $amount): void
    {
        // Nested child span — parent span এর ভিতরে
        $span = $this->tracer->spanBuilder('balance.check')
            ->setAttribute('account.id', $account)
            ->setAttribute('required.amount', $amount)
            ->startSpan();

        $scope = $span->activate();

        try {
            // DB query (auto-instrumented) — কিন্তু business context manual
            $balance = $this->getAccountBalance($account);
            $span->setAttribute('account.balance', $balance);

            if ($balance < $amount) {
                throw new \RuntimeException('Insufficient balance');
            }

            $span->setStatus(StatusCode::STATUS_OK);
        } finally {
            $scope->detach();
            $span->end();
        }
    }

    private function debit(string $account, float $amount): void
    {
        $span = $this->tracer->spanBuilder('account.debit')
            ->setAttribute('account.id', $account)
            ->setAttribute('debit.amount', $amount)
            ->startSpan();

        $scope = $span->activate();
        try {
            // ... DB update logic
            $span->addEvent('debit.processed');
            $span->setStatus(StatusCode::STATUS_OK);
        } finally {
            $scope->detach();
            $span->end();
        }
    }

    private function credit(string $to, float $amount): void
    {
        $span = $this->tracer->spanBuilder('account.credit')
            ->setAttribute('account.id', $to)
            ->setAttribute('credit.amount', $amount)
            ->startSpan();

        $scope = $span->activate();
        try {
            // ... DB update logic
            $span->addEvent('credit.processed');
            $span->setStatus(StatusCode::STATUS_OK);
        } finally {
            $scope->detach();
            $span->end();
        }
    }
}
```

### Node.js ম্যানুয়াল ইন্সট্রুমেন্টেশন

```javascript
// Nagad Payment Processing — Manual Instrumentation
const { trace, metrics, SpanStatusCode, SpanKind } = require('@opentelemetry/api');

const tracer = trace.getTracer('nagad-payment-service', '1.0.0');
const meter = metrics.getMeter('nagad-payment-service', '1.0.0');

// Custom metrics তৈরি
const paymentCounter = meter.createCounter('nagad.payment.count', {
    description: 'Total payment transactions',
});
const paymentDuration = meter.createHistogram('nagad.payment.duration_ms', {
    description: 'Payment processing duration',
    unit: 'ms',
});
const activePayments = meter.createUpDownCounter('nagad.payment.active', {
    description: 'Currently processing payments',
});

async function processPayment(userId, merchantId, amount) {
    const startTime = Date.now();
    activePayments.add(1);

    // Manual span তৈরি
    return tracer.startActiveSpan('payment.process', {
        kind: SpanKind.INTERNAL,
        attributes: {
            'payment.user_id': userId,
            'payment.merchant_id': merchantId,
            'payment.amount': amount,
            'payment.currency': 'BDT',
            'payment.method': 'nagad_wallet',
        },
    }, async (parentSpan) => {
        try {
            // Step 1: Fraud check — nested span
            await tracer.startActiveSpan('payment.fraud_check', async (span) => {
                const riskScore = await checkFraudRisk(userId, amount);
                span.setAttribute('fraud.risk_score', riskScore);
                span.setAttribute('fraud.approved', riskScore < 0.7);

                if (riskScore >= 0.7) {
                    span.setStatus({ code: SpanStatusCode.ERROR, message: 'High fraud risk' });
                    span.end();
                    throw new Error('Payment blocked: high fraud risk');
                }

                span.addEvent('fraud_check.passed');
                span.setStatus({ code: SpanStatusCode.OK });
                span.end();
            });

            // Step 2: Balance verification
            await tracer.startActiveSpan('payment.verify_balance', async (span) => {
                const balance = await getWalletBalance(userId);
                span.setAttribute('wallet.balance', balance);
                span.setAttribute('wallet.sufficient', balance >= amount);

                if (balance < amount) {
                    throw new Error('Insufficient wallet balance');
                }

                span.setStatus({ code: SpanStatusCode.OK });
                span.end();
            });

            // Step 3: Execute transaction
            const txnId = await tracer.startActiveSpan('payment.execute', async (span) => {
                const result = await executeTransaction(userId, merchantId, amount);
                span.setAttribute('payment.transaction_id', result.txnId);

                span.addEvent('payment.completed', {
                    'transaction.id': result.txnId,
                    'merchant.name': result.merchantName,
                });

                span.setStatus({ code: SpanStatusCode.OK });
                span.end();
                return result.txnId;
            });

            parentSpan.setAttribute('payment.transaction_id', txnId);
            parentSpan.setStatus({ code: SpanStatusCode.OK });

            // Metrics record
            paymentCounter.add(1, { status: 'success', merchant: merchantId });
            paymentDuration.record(Date.now() - startTime, { status: 'success' });

            return { success: true, txnId };

        } catch (error) {
            parentSpan.recordException(error);
            parentSpan.setStatus({ code: SpanStatusCode.ERROR, message: error.message });

            paymentCounter.add(1, { status: 'failed', merchant: merchantId });
            paymentDuration.record(Date.now() - startTime, { status: 'failed' });

            throw error;
        } finally {
            activePayments.add(-1);
            parentSpan.end();
        }
    });
}
```

---

## 📦 OTel Collector কনফিগারেশন

OTel Collector হলো একটি **প্রক্সি/পাইপলাইন** যা telemetry data receive করে, process করে,
এবং বিভিন্ন backend এ export করে। এটি আপনার application ও backend এর মধ্যে বসে।

### Collector Pipeline আর্কিটেকচার

```
 ┌──────────────────────────────────────────────────────────────────┐
 │              OTel Collector Pipeline                              │
 ├──────────────────────────────────────────────────────────────────┤
 │                                                                  │
 │  RECEIVERS            PROCESSORS           EXPORTERS             │
 │  (ডেটা গ্রহণ)         (ডেটা প্রক্রিয়া)     (ডেটা পাঠানো)       │
 │                                                                  │
 │  ┌────────────┐      ┌──────────────┐     ┌──────────────┐       │
 │  │ otlp       │─────►│ batch        │────►│ otlp         │       │
 │  │ (gRPC:4317)│      │ (bundle করে) │     │ (Tempo/etc)  │       │
 │  ├────────────┤      ├──────────────┤     ├──────────────┤       │
 │  │ otlp       │─────►│ attributes   │────►│ prometheus   │       │
 │  │ (HTTP:4318)│      │ (env tag add)│     │ (port:8889)  │       │
 │  ├────────────┤      ├──────────────┤     ├──────────────┤       │
 │  │ prometheus │─────►│ filter       │────►│ jaeger       │       │
 │  │ (scrape)   │      │ (unwanted    │     │ (port:14250) │       │
 │  ├────────────┤      │  drop করে)   │     ├──────────────┤       │
 │  │ filelog    │      ├──────────────┤     │ loki         │       │
 │  │ (log file) │      │ tail_sampling│     │ (logs)       │       │
 │  └────────────┘      │ (smart sample│     └──────────────┘       │
 │                      │  করে — সব    │                            │
 │                      │  রাখে না)    │                            │
 │                      └──────────────┘                            │
 │                                                                  │
 │  pipeline: traces    pipeline: metrics    pipeline: logs         │
 │  receivers → proc →  receivers → proc →   receivers → proc →    │
 │  exporters           exporters            exporters              │
 └──────────────────────────────────────────────────────────────────┘
```

### সম্পূর্ণ Collector YAML কনফিগারেশন

```yaml
# otel-collector-config.yaml
# Daraz Bangladesh মাইক্রোসার্ভিস এর জন্য OTel Collector কনফিগ

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

  # Prometheus metrics scrape করুন (legacy services থেকে)
  prometheus:
    config:
      scrape_configs:
        - job_name: 'daraz-legacy-services'
          scrape_interval: 15s
          static_configs:
            - targets: ['product-service:9090', 'order-service:9090']

  # Application log files পড়ুন
  filelog:
    include: ['/var/log/daraz/*.log']
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'

processors:
  # ডেটা batch করে পাঠান — performance এর জন্য
  batch:
    send_batch_size: 1024
    send_batch_max_size: 2048
    timeout: 5s

  # Resource attributes যোগ করুন
  resource:
    attributes:
      - key: deployment.environment
        value: "production"
        action: upsert
      - key: cloud.region
        value: "ap-southeast-1"
        action: upsert

  # সব trace রাখার দরকার নেই — smart sampling
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: error-traces
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 2000
      - name: percentage-sample
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

  # অপ্রয়োজনীয় attributes ফিল্টার করুন
  attributes:
    actions:
      - key: http.user_agent
        action: delete
      - key: http.request.header.cookie
        action: delete

exporters:
  # Jaeger — distributed traces দেখতে
  otlp/jaeger:
    endpoint: "jaeger:4317"
    tls:
      insecure: true

  # Prometheus — metrics expose করতে
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"

  # Loki — logs পাঠাতে
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

  # Debug — development এ console এ দেখতে
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource, tail_sampling, attributes]
      exporters: [otlp/jaeger]

    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, resource]
      exporters: [prometheusremotewrite]

    logs:
      receivers: [otlp, filelog]
      processors: [batch, resource, attributes]
      exporters: [loki]

  telemetry:
    logs:
      level: info
    metrics:
      address: ":8888"
```

---

## 🔀 ভেন্ডর-অ্যাগনস্টিক পদ্ধতি

OTel এর সবচেয়ে শক্তিশালী দিক হলো **vendor lock-in থেকে মুক্তি**। আপনি কোড না বদলে
backend পাল্টাতে পারবেন!

### Vendor-Specific vs OTel পদ্ধতি

```
 ╔═══════════════════════════════════════════════════════════════╗
 ║          ❌ Vendor-Specific (আগের পদ্ধতি)                    ║
 ╠═══════════════════════════════════════════════════════════════╣
 ║                                                               ║
 ║  ┌──────────┐   Datadog SDK        ┌──────────┐              ║
 ║  │ Service A │──────────────────────│ Datadog  │              ║
 ║  └──────────┘   (code coupled)     └──────────┘              ║
 ║                                                               ║
 ║  Backend বদলাতে চাইলে? সব সার্ভিসের কোড বদলাতে হবে!       ║
 ║  ২০টি সার্ভিস × কোড পরিবর্তন = 😱                          ║
 ╚═══════════════════════════════════════════════════════════════╝

 ╔═══════════════════════════════════════════════════════════════╗
 ║          ✅ OTel (ভেন্ডর-অ্যাগনস্টিক)                      ║
 ╠═══════════════════════════════════════════════════════════════╣
 ║                                                               ║
 ║  ┌──────────┐                     ┌──────────┐               ║
 ║  │ Service A │──► OTel ──► Collector ──►│ Jaeger │           ║
 ║  └──────────┘    SDK      (config  │  └──────────┘           ║
 ║                           change   │  ┌──────────┐           ║
 ║                           only!)   ├─►│ Datadog  │           ║
 ║                                    │  └──────────┘           ║
 ║                                    │  ┌──────────┐           ║
 ║  Backend বদলাতে চাইলে?            └─►│ New Tool │           ║
 ║  শুধু Collector config বদলান!         └──────────┘           ║
 ║  কোড বদলানো লাগবে না! 🎉                                    ║
 ╚═══════════════════════════════════════════════════════════════╝
```

### Vendor Lock-in তুলনা টেবিল

| বিষয় | ❌ Vendor-Specific | ✅ OTel Approach |
|---|---|---|
| SDK পরিবর্তন | প্রতি সার্ভিসে কোড বদলাতে হয় | শুধু Collector YAML বদলান |
| Multiple backend | আলাদা SDK দরকার | একই SDK, multiple exporter |
| নতুন ভাষা সাপোর্ট | vendor এর উপর নির্ভরশীল | community maintained |
| ডেটা format | proprietary | OTLP (open standard) |
| Migration cost | সপ্তাহ/মাস | ঘণ্টা |
| Community | vendor controlled | CNCF open-source |

### বাস্তব উদাহরণ — Grameenphone

```
Grameenphone এর API Platform:

Phase 1: Datadog ব্যবহার করছিল (মাসে $15,000)

Phase 2: OTel adopt করল
  - সব সার্ভিসে OTel SDK লাগালো
  - Collector config এ Datadog exporter রাখলো
  - কোনো downtime নেই!

Phase 3: Jaeger + Grafana তে migrate করলো
  - শুধু Collector config এ exporter বদলালো
  - সার্ভিসের কোড একটুও বদলায়নি!
  - মাসিক খরচ: $15,000 → $2,000 (self-hosted)

  ┌──────────────┐     ┌───────────┐     ┌──────────┐
  │ GP Services  │────►│ Collector │──X──│ Datadog  │ (সরিয়ে দিলো)
  │ (OTel SDK)   │     │           │     └──────────┘
  │ কোড একই!    │     │  config   │──►  ┌──────────┐
  └──────────────┘     │  বদলালো   │     │ Jaeger + │
                       │  শুধু!    │     │ Grafana  │ (নতুন)
                       └───────────┘     └──────────┘
```

---

## 🐘 PHP OpenTelemetry SDK

### সম্পূর্ণ Laravel সেটআপ

```php
<?php
// config/opentelemetry.php — Laravel OTel Configuration

return [
    'service_name' => env('OTEL_SERVICE_NAME', 'daraz-catalog-service'),
    'service_version' => env('APP_VERSION', '1.0.0'),
    'collector_endpoint' => env('OTEL_EXPORTER_OTLP_ENDPOINT', 'http://localhost:4318'),
    'sampler_ratio' => env('OTEL_TRACES_SAMPLER_ARG', 1.0),
];
```

```php
<?php
// app/Providers/OpenTelemetryServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use OpenTelemetry\API\Globals;
use OpenTelemetry\SDK\Trace\TracerProviderBuilder;
use OpenTelemetry\SDK\Trace\SpanProcessor\BatchSpanProcessor;
use OpenTelemetry\SDK\Resource\ResourceInfoFactory;
use OpenTelemetry\SDK\Resource\ResourceInfo;
use OpenTelemetry\SDK\Common\Attribute\Attributes;
use OpenTelemetry\Contrib\Otlp\SpanExporter as OtlpSpanExporter;
use OpenTelemetry\SDK\Trace\Sampler\ParentBased;
use OpenTelemetry\SDK\Trace\Sampler\TraceIdRatioBasedSampler;
use OpenTelemetry\SemConv\ResourceAttributes;

class OpenTelemetryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $resource = ResourceInfoFactory::defaultResource()->merge(
            ResourceInfo::create(Attributes::create([
                ResourceAttributes::SERVICE_NAME => config('opentelemetry.service_name'),
                ResourceAttributes::SERVICE_VERSION => config('opentelemetry.service_version'),
                ResourceAttributes::DEPLOYMENT_ENVIRONMENT => app()->environment(),
            ]))
        );

        $exporter = new OtlpSpanExporter(
            config('opentelemetry.collector_endpoint') . '/v1/traces'
        );

        $sampler = new ParentBased(
            new TraceIdRatioBasedSampler(config('opentelemetry.sampler_ratio'))
        );

        $tracerProvider = (new TracerProviderBuilder())
            ->setResource($resource)
            ->setSampler($sampler)
            ->addSpanProcessor(new BatchSpanProcessor($exporter))
            ->build();

        Globals::registerInitialTracerProvider($tracerProvider);

        // Shutdown hook — graceful termination
        $this->app->terminating(function () use ($tracerProvider) {
            $tracerProvider->shutdown();
        });
    }
}
```

### ❌ Bad vs ✅ Good — PHP Instrumentation

```php
<?php
// ❌ BAD: অতিরিক্ত span তৈরি — performance নষ্ট
class ProductController
{
    public function show($id)
    {
        $span1 = $this->tracer->spanBuilder('get.id')->startSpan();
        $productId = (int) $id; // এটার span দরকার নেই!
        $span1->end();

        $span2 = $this->tracer->spanBuilder('build.query')->startSpan();
        $query = "SELECT * FROM products WHERE id = ?"; // এটারও না!
        $span2->end();

        $span3 = $this->tracer->spanBuilder('execute.query')->startSpan();
        $product = DB::select($query, [$productId]); // auto-instrumented!
        $span3->end();

        $span4 = $this->tracer->spanBuilder('format.response')->startSpan();
        return response()->json($product); // এটারও span দরকার নেই!
        $span4->end();
    }
}

// ✅ GOOD: অর্থবহ business span শুধু
class ProductController
{
    public function show($id)
    {
        return $this->tracer->spanBuilder('product.get_details')
            ->setAttribute('product.id', $id)
            ->startSpan()
            ->scopeAndEnd(function ($span) use ($id) {
                $product = $this->productService->findWithRecommendations($id);

                $span->setAttribute('product.category', $product->category);
                $span->setAttribute('product.in_stock', $product->inStock());
                $span->addEvent('product.viewed', [
                    'product.name' => $product->name,
                ]);

                return response()->json($product);
            });
    }
}
```

---

## 🟢 Node.js OpenTelemetry

### সম্পূর্ণ Express.js সেটআপ

```javascript
// instrumentation.js — Daraz Node.js Service

const { NodeSDK } = require('@opentelemetry/sdk-node');
const { Resource } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } = require('@opentelemetry/semantic-conventions');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-http');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { ParentBasedSampler, TraceIdRatioBasedSampler } = require('@opentelemetry/sdk-trace-base');

const resource = new Resource({
    [ATTR_SERVICE_NAME]: 'daraz-order-service',
    [ATTR_SERVICE_VERSION]: '2.3.1',
    'deployment.environment': process.env.NODE_ENV || 'development',
});

const sdk = new NodeSDK({
    resource,
    spanProcessors: [
        new BatchSpanProcessor(
            new OTLPTraceExporter({
                url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/traces',
            })
        ),
    ],
    sampler: new ParentBasedSampler({
        root: new TraceIdRatioBasedSampler(
            parseFloat(process.env.OTEL_TRACES_SAMPLER_ARG || '1.0')
        ),
    }),
    metricReader: new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter({
            url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/metrics',
        }),
        exportIntervalMillis: 30000,
    }),
    instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### ❌ Bad vs ✅ Good — Node.js Instrumentation

```javascript
// ❌ BAD: Sensitive data span এ রাখা — security risk!
async function processOrder(order) {
    return tracer.startActiveSpan('order.process', (span) => {
        span.setAttribute('customer.email', order.email);       // PII!
        span.setAttribute('customer.phone', order.phone);       // PII!
        span.setAttribute('payment.card_number', order.cardNo); // 😱 NEVER!
        span.setAttribute('customer.address', order.address);   // PII!

        // 100+ attributes set করা — cardinality explosion!
        order.items.forEach((item, i) => {
            span.setAttribute(`item.${i}.name`, item.name);
            span.setAttribute(`item.${i}.sku`, item.sku);
            span.setAttribute(`item.${i}.price`, item.price);
            span.setAttribute(`item.${i}.qty`, item.qty);
        });

        span.end();
    });
}

// ✅ GOOD: শুধু প্রয়োজনীয় ও নিরাপদ তথ্য
async function processOrder(order) {
    return tracer.startActiveSpan('order.process', {
        attributes: {
            'order.id': order.id,
            'order.item_count': order.items.length,
            'order.total_amount': order.totalAmount,
            'order.currency': 'BDT',
            'order.payment_method': order.paymentMethod,
            'customer.tier': order.customerTier,  // gold/silver, PII না
        }
    }, async (span) => {
        try {
            const result = await executeOrder(order);
            span.setAttribute('order.status', result.status);
            span.addEvent('order.confirmed', {
                'order.confirmation_id': result.confirmationId,
            });
            span.setStatus({ code: SpanStatusCode.OK });
            return result;
        } catch (err) {
            span.recordException(err);
            span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
            throw err;
        } finally {
            span.end();
        }
    });
}
```

---

## 🔗 Jaeger, Prometheus, Grafana ইন্টিগ্রেশন

### Docker Compose — সম্পূর্ণ Observability Stack

```yaml
# docker-compose.observability.yaml
# bKash Microservices Observability Stack

version: '3.8'

services:
  # OTel Collector — সব telemetry এখানে আসে
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
      - "8888:8888"   # Collector metrics
      - "8889:8889"   # Prometheus exporter
    depends_on:
      - jaeger
      - prometheus

  # Jaeger — Distributed Tracing UI
  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # Jaeger UI
      - "14268:14268"  # Jaeger collector HTTP
      - "14250:14250"  # Jaeger collector gRPC

  # Prometheus — Metrics Storage
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-remote-write-receiver'

  # Grafana — Visualization Dashboard
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=bkash_observability
      - GF_AUTH_ANONYMOUS_ENABLED=true
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - jaeger

  # Loki — Log Aggregation
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  # ──────── bKash Services (উদাহরণ) ────────

  # Payment Service (PHP/Laravel)
  bkash-payment:
    build: ./services/payment
    environment:
      - OTEL_SERVICE_NAME=bkash-payment-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
      - OTEL_PHP_AUTOLOAD_ENABLED=true
      - OTEL_TRACES_EXPORTER=otlp
      - OTEL_METRICS_EXPORTER=otlp
      - OTEL_LOGS_EXPORTER=otlp
    depends_on:
      - otel-collector

  # User Service (Node.js)
  bkash-user:
    build: ./services/user
    environment:
      - OTEL_SERVICE_NAME=bkash-user-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
      - NODE_OPTIONS=--require ./instrumentation.js
    depends_on:
      - otel-collector

  # Notification Service (Node.js)
  bkash-notification:
    build: ./services/notification
    environment:
      - OTEL_SERVICE_NAME=bkash-notification-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
      - NODE_OPTIONS=--require ./instrumentation.js
    depends_on:
      - otel-collector
```

### সম্পূর্ণ ফ্লো — bKash টাকা পাঠানো

```
 ┌─────────────────────────────────────────────────────────────────┐
 │       bKash Send Money — Full Observability Flow                │
 ├─────────────────────────────────────────────────────────────────┤
 │                                                                 │
 │  User App                                                       │
 │    │                                                            │
 │    │ POST /api/transfer                                         │
 │    ▼                                                            │
 │  ┌──────────────────┐  trace: transfer.initiate                 │
 │  │  API Gateway      │  metric: api.request.count++             │
 │  │  (Span 1)        │  log: "Transfer request received"        │
 │  └────────┬─────────┘                                           │
 │           │                                                     │
 │           ▼                                                     │
 │  ┌──────────────────┐  trace: auth.verify_pin                   │
 │  │  Auth Service     │  metric: auth.verification.duration      │
 │  │  (Span 2)        │  log: "PIN verified for user U-123"      │
 │  └────────┬─────────┘                                           │
 │           │                                                     │
 │           ▼                                                     │
 │  ┌──────────────────┐  trace: transfer.process                  │
 │  │  Payment Service  │  metric: transfer.amount.histogram       │
 │  │  (Span 3)        │  log: "Debit ৳500 from A, Credit to B"  │
 │  └────────┬─────────┘                                           │
 │           │                                                     │
 │           ▼                                                     │
 │  ┌──────────────────┐  trace: notification.send                 │
 │  │  Notification Svc │  metric: sms.sent.count++                │
 │  │  (Span 4)        │  log: "SMS sent to 01712XXXXXX"          │
 │  └──────────────────┘                                           │
 │                                                                 │
 │  ═══════════════════════════════════════════════════            │
 │  সব spans একই trace_id: "abc-123-def-456" বহন করে              │
 │  Jaeger তে দেখলে সম্পূর্ণ journey দেখা যাবে!                  │
 │                                                                 │
 │  Grafana Dashboard:                                             │
 │  ┌─────────────────────────────────────┐                        │
 │  │ Transfer Success Rate:  99.2%       │                        │
 │  │ Avg Latency:           1.3s         │                        │
 │  │ Error Rate:            0.8%         │                        │
 │  │ Active Transfers:      342          │                        │
 │  │ [Click trace_id to see in Jaeger]   │                        │
 │  └─────────────────────────────────────┘                        │
 └─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন OpenTelemetry ব্যবহার করবেন

```
✅ মাইক্রোসার্ভিস আর্কিটেকচার (৩+ সার্ভিস)
   → bKash, Pathao, Daraz এর মতো distributed system

✅ Multiple observability backend ব্যবহার করতে চাইলে
   → Traces → Jaeger, Metrics → Prometheus, Logs → Loki

✅ Vendor lock-in এড়াতে চাইলে
   → আজ Datadog, কাল self-hosted — কোড বদলানো লাগবে না

✅ Unified correlation চাইলে
   → একটি error trace থেকে related logs ও metrics দেখতে

✅ Multi-language environment
   → PHP, Node.js, Go, Python — সবার জন্য একই standard

✅ Cloud-native / Kubernetes environment
   → OTel Collector DaemonSet হিসেবে deploy করা যায়

✅ Standardized telemetry চাইলে
   → Semantic conventions সব team কে একই ভাষায় কথা বলতে দেয়
```

### ❌ কখন OpenTelemetry ব্যবহার করবেন না

```
❌ একটি মাত্র monolith application
   → সাধারণ APM tool (New Relic, Scout) যথেষ্ট

❌ খুব ছোট প্রজেক্ট / MVP
   → console.log আর basic monitoring দিয়ে শুরু করুন

❌ ইতোমধ্যে vendor SDK deeply integrated
   → migration cost >> benefit হলে এখন করবেন না

❌ Team এ observability experience কম
   → আগে basic monitoring শিখুন, তারপর OTel

❌ শুধু simple uptime monitoring চাইলে
   → Uptime Robot / Pingdom যথেষ্ট, OTel overkill
```

### ❌ Bad Instrumentation Practices

```
❌ BAD: সব function এ span তৈরি করা
   → হাজার হাজার span = performance hit + বিশাল storage cost

❌ BAD: Sensitive data attributes এ রাখা
   → credit card number, password, personal info কখনো না

❌ BAD: High cardinality attributes
   → user_id attribute (লক্ষ লক্ষ unique value) = metric explosion

❌ BAD: Sampling না করা production এ
   → 100% traces রাখা = অতিরিক্ত cost, Collector overload

❌ BAD: Context propagation ভুলে যাওয়া
   → সার্ভিসের মধ্যে trace_id forward না করলে trace ভেঙে যায়
```

### ✅ Good Instrumentation Practices

```
✅ GOOD: Business-meaningful spans তৈরি করুন
   → "payment.process", "order.validate", "driver.match"

✅ GOOD: Tail-based sampling ব্যবহার করুন
   → শুধু errors + slow requests ১০০% রাখুন, বাকি ১০% sample

✅ GOOD: Semantic conventions অনুসরণ করুন
   → http.request.method, db.system — standard নাম ব্যবহার করুন

✅ GOOD: Resource attributes ঠিকভাবে সেট করুন
   → service.name, deployment.environment, service.version

✅ GOOD: Collector ব্যবহার করুন, সরাসরি backend এ পাঠাবেন না
   → Collector buffering, retry, processing সব সামলায়

✅ GOOD: Graceful shutdown implement করুন
   → app বন্ধ হওয়ার আগে pending telemetry flush করুন
```

---

## 📊 বাস্তব পরিস্থিতি — bKash মাইক্রোসার্ভিস Observability

```
 ┌──────────────────────────────────────────────────────────────┐
 │        bKash Production Observability Architecture            │
 ├──────────────────────────────────────────────────────────────┤
 │                                                              │
 │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
 │  │ Payment │ │  Auth   │ │  User   │ │  Notif  │           │
 │  │ Service │ │ Service │ │ Service │ │ Service │           │
 │  │ (PHP)   │ │ (Node)  │ │ (Node)  │ │ (Node)  │           │
 │  │ +OTel   │ │ +OTel   │ │ +OTel   │ │ +OTel   │           │
 │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │
 │       │           │           │           │                  │
 │       └───────────┴─────┬─────┴───────────┘                  │
 │                         │ OTLP                               │
 │                         ▼                                    │
 │              ┌─────────────────────┐                         │
 │              │   OTel Collector    │                         │
 │              │   (DaemonSet on     │                         │
 │              │    each K8s node)   │                         │
 │              └──────────┬──────────┘                         │
 │                    ┌────┼────┐                               │
 │                    │    │    │                                │
 │                    ▼    ▼    ▼                                │
 │              ┌────┐ ┌────┐ ┌────┐                            │
 │              │Jaeg│ │Prom│ │Loki│                            │
 │              │er  │ │ethe│ │    │                            │
 │              │    │ │us  │ │    │                            │
 │              └──┬─┘ └──┬─┘ └──┬─┘                            │
 │                 │      │      │                               │
 │                 └──────┼──────┘                               │
 │                        ▼                                     │
 │              ┌─────────────────────┐                         │
 │              │      Grafana        │                         │
 │              │                     │                         │
 │              │  Dashboard:         │                         │
 │              │  • Transfer SLA     │                         │
 │              │  • Error rates      │                         │
 │              │  • P99 latency     │                         │
 │              │  • Active users    │                         │
 │              │                     │                         │
 │              │  Alert Rules:       │                         │
 │              │  • Error > 1%      │                         │
 │              │  • P99 > 3s        │                         │
 │              │  • Transfer fail   │                         │
 │              └─────────────────────┘                         │
 │                                                              │
 │  On-call engineer alert পেলে:                               │
 │  1. Grafana metric দেখে কোথায় spike                        │
 │  2. Exemplar click করে Jaeger trace দেখে                    │
 │  3. Trace থেকে span_id নিয়ে Loki তে log search করে        │
 │  4. Root cause পেয়ে fix করে — MTTR ৭০% কমলো! 🎉            │
 └──────────────────────────────────────────────────────────────┘
```

---

## 📌 সারাংশ

```
 ╔══════════════════════════════════════════════════════════════╗
 ║               OpenTelemetry মনে রাখার চেকলিস্ট             ║
 ╠══════════════════════════════════════════════════════════════╣
 ║                                                              ║
 ║  ☐ OTel = Traces + Metrics + Logs একত্রে                   ║
 ║  ☐ OTLP = Open standard protocol                            ║
 ║  ☐ Vendor-agnostic: কোড না বদলে backend বদলানো যায়        ║
 ║  ☐ Auto-instrumentation = zero-code observability            ║
 ║  ☐ Manual instrumentation = business logic spans             ║
 ║  ☐ Collector = receive → process → export pipeline           ║
 ║  ☐ Resource = সার্ভিসের পরিচয়                              ║
 ║  ☐ Semantic Conventions = standard attribute নাম             ║
 ║  ☐ Sampling = production এ ১০০% trace রাখবেন না            ║
 ║  ☐ Correlation = trace_id দিয়ে trace-metric-log যোগ        ║
 ║                                                              ║
 ║  Production Checklist:                                       ║
 ║  ☐ Resource attributes সেট করুন                             ║
 ║  ☐ Collector deploy করুন (সরাসরি backend এ পাঠাবেন না)     ║
 ║  ☐ Tail-based sampling কনফিগার করুন                        ║
 ║  ☐ Sensitive data filter করুন (Collector processor এ)       ║
 ║  ☐ Graceful shutdown implement করুন                         ║
 ║  ☐ Dashboard + Alert rules তৈরি করুন                        ║
 ║  ☐ Runbook লিখুন — alert পেলে কী করবেন                    ║
 ║                                                              ║
 ╚══════════════════════════════════════════════════════════════╝
```

---

> **"Observability ছাড়া distributed system চালানো হলো অন্ধকারে গাড়ি চালানোর মতো।
> OpenTelemetry হলো সেই হেডলাইট যা পথ দেখায় — vendor lock-in ছাড়াই।"**
> — bKash SRE Team এর মতো ভাবুন 🔭
