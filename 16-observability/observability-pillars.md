# 🔭 Observability vs Monitoring — Three Pillars Deep Dive

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII ডায়াগ্রাম](#ascii-ডায়াগ্রাম)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🧠 সংজ্ঞা ও ধারণা

### Monitoring vs Observability — পার্থক্য কী?

**Monitoring (মনিটরিং):** "Known Unknowns" — আপনি আগে থেকে জানেন কী ভুল হতে পারে, তাই সেটার জন্য alert সেট করেন।

> "CPU 90% হলে alert দাও" — এটা monitoring

**Observability (অবজার্ভেবিলিটি):** "Unknown Unknowns" — আপনি জানেন না কী ভুল হবে, কিন্তু system-এর output দেখে যেকোনো প্রশ্নের উত্তর বের করতে পারেন।

> "কেন শুক্রবার দুপুর ২টায় চট্টগ্রামের user-রা slow response পাচ্ছে?" — এটা observability

### 🏛️ তিনটি স্তম্ভ (Three Pillars)

#### 1️⃣ Logs (লগ)

**কী:** সিস্টেমে ঘটা প্রতিটি event-এর discrete record।

**Structured Logging:** JSON format-এ log যাতে machine-readable হয়।

```json
{
  "timestamp": "2024-01-15T14:23:45Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "abc-123-def",
  "user_id": "U-98765",
  "message": "Payment failed",
  "error_code": "TIMEOUT",
  "latency_ms": 5200
}
```

**Log Correlation:** trace_id দিয়ে একই request-এর সব log একসাথে দেখা।

#### 2️⃣ Metrics (মেট্রিক্স)

**কী:** সময়ের সাথে পরিমাপযোগ্য সংখ্যাসূচক মান।

**চার ধরনের Metric:**

| টাইপ | বাংলায় | উদাহরণ |
|------|---------|---------|
| **Counter** | শুধু বাড়ে | মোট request সংখ্যা, মোট error |
| **Gauge** | বাড়ে/কমে | বর্তমান CPU%, active connections |
| **Histogram** | বিতরণ | Response time distribution (p50, p95, p99) |
| **Summary** | সারাংশ | Pre-calculated percentiles client-side |

#### 3️⃣ Traces (ট্রেস)

**কী:** একটি request সিস্টেমের ভিতর দিয়ে যে path follow করে তার সম্পূর্ণ চিত্র।

**মূল ধারণা:**
- **Trace:** একটি সম্পূর্ণ request journey (অনেক span নিয়ে গঠিত)
- **Span:** trace-র একটি unit of work (একটি service call, DB query ইত্যাদি)
- **Span Context:** trace_id + span_id + flags (propagation headers-এ থাকে)

### 🔗 Correlation — তিন পিলার সংযুক্ত করা

**Correlation ID (trace_id)** দিয়ে তিনটি pillar একসাথে কাজ করে:
- Log → trace_id field
- Metric → trace_id exemplar  
- Trace → trace_id root

### 💥 Cardinality Explosion Problem

**Cardinality** = একটি metric label-এ কতগুলো unique value থাকতে পারে।

```
✅ Low cardinality: status_code = {200, 201, 400, 404, 500}  → 5 values
❌ High cardinality: user_id = {U-1, U-2, ..., U-10000000}   → 10M values!
```

High cardinality metric → Prometheus/InfluxDB crash করতে পারে! 

**সমাধান:** High-cardinality data → Traces/Logs-এ রাখুন, Metrics-এ নয়।

### 🌐 OpenTelemetry (OTel)

OpenTelemetry হলো CNCF-এর open standard যা Logs, Metrics, এবং Traces — তিনটিকেই একীভূত করে।

**সুবিধা:**
- Vendor-neutral (Datadog, Grafana, Jaeger যেকোনোটায় পাঠানো যায়)
- Single SDK দিয়ে তিনটিই instrument করা যায়
- Automatic context propagation

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🏍️ Pathao Ride Booking — Slow Request Debugging

**সমস্যা:** User complain করছে ride booking-এ 8 সেকেন্ড লাগছে, কিন্তু average latency dashboard-এ দেখাচ্ছে 400ms!

**Service Architecture:**
```
User App → API Gateway → Booking Service → Driver Matching
                                         → Pricing Service → Surge Engine
                                         → Map Service → Google Maps API
                                         → Notification Service → FCM
```

**Step 1: Metrics দিয়ে শুরু**

Dashboard-এ দেখা গেলো:
- Average latency: 400ms ✅
- p99 latency: 7800ms ❌
- Error rate: 0.3%

**Step 2: Trace দিয়ে root cause খুঁজা**

Slow request-এর trace_id ধরে Jaeger-এ দেখা গেলো:

```
Total: 7800ms
├── API Gateway: 12ms
├── Booking Service: 45ms
├── Driver Matching: 6500ms  ← 🔴 BOTTLENECK!
│   ├── Redis Geo Query: 50ms
│   ├── Driver Filter: 6200ms ← 🔴
│   └── Response Build: 250ms
├── Pricing Service: 800ms
└── Notification Service: 443ms
```

**Step 3: Log correlation দিয়ে বিস্তারিত**

trace_id দিয়ে filter করে log-এ পাওয়া গেলো:

```json
{
  "trace_id": "abc-123",
  "service": "driver-matching",
  "message": "Retrying driver search, expanding radius",
  "attempt": 5,
  "radius_km": 15,
  "available_drivers": 0,
  "area": "Uttara Sector 10"
}
```

**Root Cause:** উত্তরা সেক্টর ১০-এ ঈদের দিনে driver নেই, তাই radius ৫ বার expand হচ্ছে!

**Fix:** Radius expansion-এ timeout যোগ করা হলো, 3 attempt-এর পর "No driver available" return করে।

### 📰 Prothom Alo — High Traffic Event Observability

ব্রেকিং নিউজে সার্ভার slow হয়ে যায়। Observability দিয়ে ধরা পড়লো:
- **Metrics:** CDN cache hit ratio 30%-এ নেমে গেছে (normally 95%)
- **Trace:** একটি uncached API call প্রতিটি page load-এ database hit করছে
- **Logs:** "Cache MISS for article_id=12345, TTL expired" — breaking news article-এর TTL ছিল 5 মিনিট

---

## 📊 ASCII ডায়াগ্রাম

### Three Pillars Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PLATFORM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   📝 LOGS    │  │  📈 METRICS  │  │    🔗 TRACES         │  │
│  │              │  │              │  │                      │  │
│  │ • Structured │  │ • Counters   │  │ • Distributed trace │  │
│  │ • Searchable │  │ • Gauges     │  │ • Span context      │  │
│  │ • Contextual │  │ • Histograms │  │ • Service map       │  │
│  │              │  │ • Summaries  │  │                      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                  │                     │               │
│         └──────────────────┼─────────────────────┘               │
│                            │                                     │
│                   ┌────────┴────────┐                            │
│                   │  Correlation ID │                            │
│                   │  (trace_id)     │                            │
│                   └─────────────────┘                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              OpenTelemetry Collector                        │ │
│  │  Receives → Processes → Exports to backends                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Distributed Trace Visualization

```
         Pathao Ride Booking — Distributed Trace
         trace_id: abc-123-def-456

Time →   0ms     200ms    400ms    600ms    800ms   1000ms
         |        |        |        |        |        |
Service  ├────────────────────────────────────────────┤
         │                                            │

API GW   ├──┤ 12ms
              │
Booking  .....├────┤ 45ms
                    │
Matching .........├────────────────────────────┤ 650ms
                  │   │                        │
  Redis   ........├─┤  │                        │
  Filter  ..........├──────────────────────┤    │
                                                │
Pricing  .......................├──────────┤ 180ms
                               │          │
  Surge   .....................├────┤      │
                                          │
Notify   .....................................├────┤ 90ms
                                               │
  FCM     .....................................├──┤

Legend: ├──┤ = span duration, ..... = waiting
```

### Monitoring vs Observability

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   📊 MONITORING              vs        🔭 OBSERVABILITY    │
│   (Known Unknowns)                    (Unknown Unknowns)   │
│                                                             │
│   "CPU > 90% হলে alert"              "কেন এই user-এর      │
│                                        request slow?"       │
│                                                             │
│   ┌─────────────────┐                ┌─────────────────┐   │
│   │ Dashboard দেখি  │                │ Question করি    │   │
│   │ Known patterns  │                │ Explore করি     │   │
│   │ Alert rule লিখি │                │ Correlation করি │   │
│   │ Predefined view │                │ Ad-hoc query    │   │
│   └─────────────────┘                └─────────────────┘   │
│                                                             │
│   Tools:                             Tools:                  │
│   • Nagios, Zabbix                   • Jaeger, Zipkin       │
│   • CloudWatch Alarms                • Grafana Tempo        │
│   • Uptime checks                    • Honeycomb            │
│                                      • OpenTelemetry        │
│                                                             │
│   আমার কাছে: ✅ Answer               আমার কাছে: ❓ Question│
│   System থেকে: 📌 Fixed view          System থেকে: 🔍 Data │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cardinality Explosion

```
           Cardinality Impact on Storage

Labels:    method × status × endpoint × user_id
           ──────   ──────   ────────   ───────
Values:      5    ×   5    ×   100    ×  10M

           ┌─────────────────────────────────────┐
Low Card:  │ 5 × 5 × 100 = 2,500 time series    │ ✅ Fine
           └─────────────────────────────────────┘

           ┌─────────────────────────────────────┐
High Card: │ 5 × 5 × 100 × 10M = 25 BILLION     │ ❌ CRASH!
           │ time series                          │
           └─────────────────────────────────────┘

Rule of thumb:
┌────────────────────────────────────────────────────┐
│  High-cardinality data → TRACES & LOGS             │
│  Low-cardinality aggregated data → METRICS         │
└────────────────────────────────────────────────────┘
```

### Sampling Strategies

```
┌─────────────────────────────────────────────────────────┐
│              Sampling Strategies                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Head-based Sampling (শুরুতেই সিদ্ধান্ত)             │
│     ┌──────────┐                                        │
│     │ Request  │──→ Random(10%) ──→ Sample? ─┬→ ✅ Keep │
│     │ arrives  │                             └→ ❌ Drop │
│     └──────────┘                                        │
│     সুবিধা: সহজ                                         │
│     অসুবিধা: Interesting request miss হতে পারে          │
│                                                          │
│  2. Tail-based Sampling (শেষে সিদ্ধান্ত)                │
│     ┌──────────┐        ┌───────────┐                   │
│     │ Request  │──→ ... │ Complete  │──→ Analyze ─┬→ ✅ │
│     │ arrives  │        │ Trace     │             └→ ❌ │
│     └──────────┘        └───────────┘                   │
│     সুবিধা: Error/Slow traces সব keep হয়               │
│     অসুবিধা: Memory intensive, complex                  │
│                                                          │
│  3. Priority Sampling                                    │
│     ├── Error traces → Always keep (100%)                │
│     ├── Slow traces (>2s) → Always keep (100%)           │
│     ├── Payment transactions → Always keep (100%)        │
│     └── Normal successful → Sample 5%                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Structured Logging, Metrics & Trace Correlation

```php
<?php

/**
 * Observability Implementation — Three Pillars
 * Pathao Ride Booking Service-এ Observability
 */

// === PILLAR 1: Structured Logging ===

class StructuredLogger
{
    private string $serviceName;
    private ?string $traceId = null;
    private ?string $spanId = null;

    public function __construct(string $serviceName)
    {
        $this->serviceName = $serviceName;
    }

    public function setTraceContext(string $traceId, string $spanId): void
    {
        $this->traceId = $traceId;
        $this->spanId = $spanId;
    }

    public function log(string $level, string $message, array $context = []): void
    {
        $entry = [
            'timestamp' => gmdate('Y-m-d\TH:i:s.v\Z'),
            'level' => $level,
            'service' => $this->serviceName,
            'trace_id' => $this->traceId,
            'span_id' => $this->spanId,
            'message' => $message,
            ...$context,
        ];

        // stdout-এ JSON output — log aggregator collect করবে
        fwrite(STDOUT, json_encode($entry, JSON_UNESCAPED_UNICODE) . "\n");
    }

    public function info(string $msg, array $ctx = []): void
    {
        $this->log('INFO', $msg, $ctx);
    }

    public function error(string $msg, array $ctx = []): void
    {
        $this->log('ERROR', $msg, $ctx);
    }

    public function warn(string $msg, array $ctx = []): void
    {
        $this->log('WARN', $msg, $ctx);
    }
}

// === PILLAR 2: Metrics ===

class MetricsCollector
{
    private array $counters = [];
    private array $gauges = [];
    private array $histograms = [];

    /**
     * Counter: শুধু বাড়ে (monotonically increasing)
     */
    public function incrementCounter(string $name, array $labels = [], float $value = 1): void
    {
        $key = $this->buildKey($name, $labels);
        if (!isset($this->counters[$key])) {
            $this->counters[$key] = ['name' => $name, 'labels' => $labels, 'value' => 0];
        }
        $this->counters[$key]['value'] += $value;
    }

    /**
     * Gauge: বাড়তে বা কমতে পারে (point-in-time value)
     */
    public function setGauge(string $name, float $value, array $labels = []): void
    {
        $key = $this->buildKey($name, $labels);
        $this->gauges[$key] = ['name' => $name, 'labels' => $labels, 'value' => $value];
    }

    /**
     * Histogram: distribution record করে (latency tracking-এ useful)
     */
    public function recordHistogram(string $name, float $value, array $labels = []): void
    {
        $key = $this->buildKey($name, $labels);
        if (!isset($this->histograms[$key])) {
            $this->histograms[$key] = [
                'name' => $name,
                'labels' => $labels,
                'values' => [],
                'sum' => 0,
                'count' => 0,
                'buckets' => [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
            ];
        }
        $this->histograms[$key]['values'][] = $value;
        $this->histograms[$key]['sum'] += $value;
        $this->histograms[$key]['count']++;
    }

    /**
     * Prometheus exposition format-এ export
     */
    public function export(): string
    {
        $output = "";

        foreach ($this->counters as $metric) {
            $labels = $this->formatLabels($metric['labels']);
            $output .= "{$metric['name']}{$labels} {$metric['value']}\n";
        }

        foreach ($this->gauges as $metric) {
            $labels = $this->formatLabels($metric['labels']);
            $output .= "{$metric['name']}{$labels} {$metric['value']}\n";
        }

        foreach ($this->histograms as $metric) {
            $labels = $this->formatLabels($metric['labels']);
            sort($metric['values']);
            foreach ($metric['buckets'] as $bucket) {
                $count = count(array_filter($metric['values'], fn($v) => $v <= $bucket));
                $output .= "{$metric['name']}_bucket{le=\"{$bucket}\"} {$count}\n";
            }
            $output .= "{$metric['name']}_sum {$metric['sum']}\n";
            $output .= "{$metric['name']}_count {$metric['count']}\n";
        }

        return $output;
    }

    private function buildKey(string $name, array $labels): string
    {
        return $name . ':' . json_encode($labels);
    }

    private function formatLabels(array $labels): string
    {
        if (empty($labels)) return '';
        $parts = array_map(fn($k, $v) => "$k=\"$v\"", array_keys($labels), $labels);
        return '{' . implode(',', $parts) . '}';
    }
}

// === PILLAR 3: Distributed Tracing ===

class Span
{
    public string $spanId;
    public string $traceId;
    public ?string $parentSpanId;
    public string $operationName;
    public float $startTime;
    public ?float $endTime = null;
    public array $tags = [];
    public array $logs = [];

    public function __construct(string $traceId, string $operationName, ?string $parentSpanId = null)
    {
        $this->spanId = bin2hex(random_bytes(8));
        $this->traceId = $traceId;
        $this->parentSpanId = $parentSpanId;
        $this->operationName = $operationName;
        $this->startTime = microtime(true) * 1000;
    }

    public function setTag(string $key, string $value): self
    {
        $this->tags[$key] = $value;
        return $this;
    }

    public function log(string $message, array $fields = []): self
    {
        $this->logs[] = [
            'timestamp' => microtime(true) * 1000,
            'message' => $message,
            'fields' => $fields,
        ];
        return $this;
    }

    public function finish(): void
    {
        $this->endTime = microtime(true) * 1000;
    }

    public function getDurationMs(): float
    {
        $end = $this->endTime ?? microtime(true) * 1000;
        return $end - $this->startTime;
    }
}

class Tracer
{
    private string $serviceName;
    private array $spans = [];

    public function __construct(string $serviceName)
    {
        $this->serviceName = $serviceName;
    }

    public function startTrace(string $operationName): Span
    {
        $traceId = bin2hex(random_bytes(16));
        $span = new Span($traceId, $operationName);
        $span->setTag('service', $this->serviceName);
        $this->spans[] = $span;
        return $span;
    }

    public function startSpan(string $operationName, Span $parent): Span
    {
        $span = new Span($parent->traceId, $operationName, $parent->spanId);
        $span->setTag('service', $this->serviceName);
        $this->spans[] = $span;
        return $span;
    }

    /**
     * Incoming request-এর header থেকে context extract
     */
    public function extractContext(array $headers): ?array
    {
        // W3C Trace Context format: traceparent header
        $traceparent = $headers['traceparent'] ?? null;
        if (!$traceparent) return null;

        // Format: version-trace_id-parent_id-flags
        $parts = explode('-', $traceparent);
        if (count($parts) !== 4) return null;

        return [
            'trace_id' => $parts[1],
            'parent_span_id' => $parts[2],
            'flags' => $parts[3],
        ];
    }

    /**
     * Outgoing request-এর header-এ context inject
     */
    public function injectContext(Span $span): array
    {
        return [
            'traceparent' => "00-{$span->traceId}-{$span->spanId}-01",
        ];
    }

    public function getSpans(): array
    {
        return $this->spans;
    }
}

// === Combined Observability: Pathao Ride Booking ===

class RideBookingService
{
    private StructuredLogger $logger;
    private MetricsCollector $metrics;
    private Tracer $tracer;

    public function __construct()
    {
        $this->logger = new StructuredLogger('ride-booking');
        $this->metrics = new MetricsCollector();
        $this->tracer = new Tracer('ride-booking');
    }

    public function bookRide(array $request): array
    {
        // Trace শুরু
        $rootSpan = $this->tracer->startTrace('book_ride');
        $rootSpan->setTag('user_id', $request['user_id']);
        $rootSpan->setTag('pickup', $request['pickup_area']);

        // Logger-এ trace context সেট
        $this->logger->setTraceContext($rootSpan->traceId, $rootSpan->spanId);

        $this->logger->info('Ride booking started', [
            'user_id' => $request['user_id'],
            'pickup' => $request['pickup_area'],
        ]);

        // Metrics: Request counter
        $this->metrics->incrementCounter('ride_booking_requests_total', [
            'area' => $request['pickup_area'],
        ]);

        try {
            // Driver matching span
            $matchSpan = $this->tracer->startSpan('find_driver', $rootSpan);
            $driver = $this->findNearestDriver($request, $matchSpan);
            $matchSpan->finish();

            // Pricing span
            $priceSpan = $this->tracer->startSpan('calculate_price', $rootSpan);
            $price = $this->calculatePrice($request, $priceSpan);
            $priceSpan->finish();

            $rootSpan->setTag('status', 'success');
            $rootSpan->finish();

            // Success metrics
            $this->metrics->incrementCounter('ride_booking_success_total');
            $this->metrics->recordHistogram(
                'ride_booking_duration_ms',
                $rootSpan->getDurationMs(),
                ['area' => $request['pickup_area']]
            );

            $this->logger->info('Ride booked successfully', [
                'driver_id' => $driver['id'],
                'price' => $price,
                'duration_ms' => $rootSpan->getDurationMs(),
            ]);

            return ['status' => 'confirmed', 'driver' => $driver, 'price' => $price];

        } catch (\Exception $e) {
            $rootSpan->setTag('status', 'error');
            $rootSpan->setTag('error.message', $e->getMessage());
            $rootSpan->finish();

            $this->metrics->incrementCounter('ride_booking_errors_total', [
                'error_type' => get_class($e),
            ]);

            $this->logger->error('Ride booking failed', [
                'error' => $e->getMessage(),
                'duration_ms' => $rootSpan->getDurationMs(),
            ]);

            throw $e;
        }
    }

    private function findNearestDriver(array $request, Span $parentSpan): array
    {
        $span = $this->tracer->startSpan('redis_geo_query', $parentSpan);
        // Simulate Redis geo query
        usleep(50000); // 50ms
        $span->setTag('radius_km', '5');
        $span->finish();

        return ['id' => 'D-123', 'name' => 'Rahim', 'distance_km' => 1.2];
    }

    private function calculatePrice(array $request, Span $parentSpan): float
    {
        $span = $this->tracer->startSpan('surge_check', $parentSpan);
        // Simulate surge pricing check
        usleep(30000); // 30ms
        $surgeMultiplier = 1.5;
        $span->setTag('surge', (string)$surgeMultiplier);
        $span->finish();

        return 150.0 * $surgeMultiplier;
    }

    public function getMetrics(): string
    {
        return $this->metrics->export();
    }
}

// === ব্যবহার ===

$service = new RideBookingService();

$result = $service->bookRide([
    'user_id' => 'U-98765',
    'pickup_area' => 'Uttara',
    'destination' => 'Gulshan',
    'ride_type' => 'bike',
]);

echo "\n=== 🏍️ Ride Booking Result ===\n";
echo json_encode($result, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE) . "\n";

echo "\n=== 📈 Metrics Export (Prometheus Format) ===\n";
echo $service->getMetrics();
```

---

## 🟨 JavaScript কোড উদাহরণ

### OpenTelemetry-based Observability with All Three Pillars

```javascript
/**
 * Three Pillars of Observability — JavaScript Implementation
 * Pathao Ride Service with Logs, Metrics, and Traces
 */

// === PILLAR 1: Structured Logger ===

class StructuredLogger {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.traceId = null;
    this.spanId = null;
  }

  setContext(traceId, spanId) {
    this.traceId = traceId;
    this.spanId = spanId;
  }

  _emit(level, message, context = {}) {
    const entry = {
      timestamp: new Date().toISOString(),
      level,
      service: this.serviceName,
      trace_id: this.traceId,
      span_id: this.spanId,
      message,
      ...context,
    };
    // Production-এ stdout → Fluentd/Vector → Elasticsearch/Loki
    console.log(JSON.stringify(entry));
  }

  info(msg, ctx) { this._emit('INFO', msg, ctx); }
  warn(msg, ctx) { this._emit('WARN', msg, ctx); }
  error(msg, ctx) { this._emit('ERROR', msg, ctx); }
  debug(msg, ctx) { this._emit('DEBUG', msg, ctx); }
}

// === PILLAR 2: Metrics Collector ===

class MetricsRegistry {
  constructor() {
    this.counters = new Map();
    this.gauges = new Map();
    this.histograms = new Map();
  }

  counter(name, labels = {}) {
    const key = this._key(name, labels);
    if (!this.counters.has(key)) {
      this.counters.set(key, { name, labels, value: 0 });
    }
    return {
      inc: (val = 1) => { this.counters.get(key).value += val; },
      get: () => this.counters.get(key).value,
    };
  }

  gauge(name, labels = {}) {
    const key = this._key(name, labels);
    if (!this.gauges.has(key)) {
      this.gauges.set(key, { name, labels, value: 0 });
    }
    return {
      set: (val) => { this.gauges.get(key).value = val; },
      inc: (val = 1) => { this.gauges.get(key).value += val; },
      dec: (val = 1) => { this.gauges.get(key).value -= val; },
      get: () => this.gauges.get(key).value,
    };
  }

  histogram(name, labels = {}, buckets = [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]) {
    const key = this._key(name, labels);
    if (!this.histograms.has(key)) {
      this.histograms.set(key, { name, labels, buckets, values: [], sum: 0, count: 0 });
    }
    return {
      observe: (val) => {
        const h = this.histograms.get(key);
        h.values.push(val);
        h.sum += val;
        h.count++;
      },
      percentile: (p) => {
        const h = this.histograms.get(key);
        const sorted = [...h.values].sort((a, b) => a - b);
        const idx = Math.ceil((p / 100) * sorted.length) - 1;
        return sorted[Math.max(0, idx)] || 0;
      },
    };
  }

  _key(name, labels) {
    return `${name}:${JSON.stringify(labels)}`;
  }

  // Prometheus format export
  toPrometheus() {
    let output = '';
    for (const [, m] of this.counters) {
      output += `${m.name}${this._fmtLabels(m.labels)} ${m.value}\n`;
    }
    for (const [, m] of this.gauges) {
      output += `${m.name}${this._fmtLabels(m.labels)} ${m.value}\n`;
    }
    for (const [, h] of this.histograms) {
      for (const bucket of h.buckets) {
        const count = h.values.filter(v => v <= bucket).length;
        output += `${h.name}_bucket{le="${bucket}"} ${count}\n`;
      }
      output += `${h.name}_sum ${h.sum}\n`;
      output += `${h.name}_count ${h.count}\n`;
    }
    return output;
  }

  _fmtLabels(labels) {
    const entries = Object.entries(labels);
    if (entries.length === 0) return '';
    return `{${entries.map(([k, v]) => `${k}="${v}"`).join(',')}}`;
  }
}

// === PILLAR 3: Distributed Tracer ===

class Span {
  constructor(traceId, operationName, parentSpanId = null) {
    this.traceId = traceId;
    this.spanId = this._generateId();
    this.parentSpanId = parentSpanId;
    this.operationName = operationName;
    this.startTime = performance.now();
    this.endTime = null;
    this.tags = {};
    this.events = [];
    this.status = 'OK';
  }

  setTag(key, value) {
    this.tags[key] = value;
    return this;
  }

  addEvent(name, attributes = {}) {
    this.events.push({
      timestamp: performance.now(),
      name,
      attributes,
    });
    return this;
  }

  setError(error) {
    this.status = 'ERROR';
    this.tags['error'] = true;
    this.tags['error.message'] = error.message;
    this.tags['error.stack'] = error.stack;
    return this;
  }

  end() {
    this.endTime = performance.now();
  }

  get durationMs() {
    return (this.endTime || performance.now()) - this.startTime;
  }

  _generateId() {
    return Array.from(crypto.getRandomValues(new Uint8Array(8)))
      .map(b => b.toString(16).padStart(2, '0')).join('');
  }
}

class DistributedTracer {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.spans = [];
  }

  startTrace(operationName) {
    const traceId = Array.from(crypto.getRandomValues(new Uint8Array(16)))
      .map(b => b.toString(16).padStart(2, '0')).join('');
    const span = new Span(traceId, operationName);
    span.setTag('service.name', this.serviceName);
    this.spans.push(span);
    return span;
  }

  startSpan(operationName, parentSpan) {
    const span = new Span(parentSpan.traceId, operationName, parentSpan.spanId);
    span.setTag('service.name', this.serviceName);
    this.spans.push(span);
    return span;
  }

  // W3C TraceContext propagation headers
  injectHeaders(span) {
    return {
      traceparent: `00-${span.traceId}-${span.spanId}-01`,
      tracestate: `service=${this.serviceName}`,
    };
  }

  extractFromHeaders(headers) {
    const traceparent = headers['traceparent'];
    if (!traceparent) return null;
    const [, traceId, parentId, flags] = traceparent.split('-');
    return { traceId, parentId, sampled: flags === '01' };
  }

  // Trace visualization
  printTrace() {
    console.log('\n=== 🔗 Distributed Trace ===');
    console.log(`Trace ID: ${this.spans[0]?.traceId}`);
    console.log('');

    const rootSpans = this.spans.filter(s => !s.parentSpanId);
    for (const root of rootSpans) {
      this._printSpan(root, 0);
    }
  }

  _printSpan(span, indent) {
    const prefix = '  '.repeat(indent);
    const duration = span.durationMs.toFixed(1);
    const status = span.status === 'ERROR' ? '❌' : '✅';
    console.log(`${prefix}${status} ${span.operationName} [${duration}ms]`);

    const children = this.spans.filter(s => s.parentSpanId === span.spanId);
    for (const child of children) {
      this._printSpan(child, indent + 1);
    }
  }
}

// === Combined: Observability-Driven Service ===

class ObservableRideService {
  constructor() {
    this.logger = new StructuredLogger('pathao-ride');
    this.metrics = new MetricsRegistry();
    this.tracer = new DistributedTracer('pathao-ride');

    // Pre-define metrics
    this.requestCounter = this.metrics.counter('http_requests_total');
    this.activeRequests = this.metrics.gauge('http_requests_active');
    this.requestDuration = this.metrics.histogram('http_request_duration_ms');
  }

  async handleBooking(request) {
    const rootSpan = this.tracer.startTrace('POST /api/v1/rides');
    rootSpan.setTag('http.method', 'POST');
    rootSpan.setTag('http.url', '/api/v1/rides');
    rootSpan.setTag('user.id', request.userId);

    this.logger.setContext(rootSpan.traceId, rootSpan.spanId);
    this.activeRequests.inc();
    this.requestCounter.inc();

    this.logger.info('Ride booking request received', {
      user_id: request.userId,
      pickup: request.pickup,
      destination: request.destination,
    });

    try {
      // Step 1: Find driver
      const driverSpan = this.tracer.startSpan('find_nearest_driver', rootSpan);
      const driver = await this.findDriver(request, driverSpan);
      driverSpan.setTag('driver.id', driver.id);
      driverSpan.end();

      // Step 2: Calculate price
      const priceSpan = this.tracer.startSpan('calculate_price', rootSpan);
      const price = await this.calculatePrice(request, priceSpan);
      priceSpan.setTag('price.amount', price);
      priceSpan.end();

      // Step 3: Send notification
      const notifySpan = this.tracer.startSpan('notify_driver', rootSpan);
      await this.notifyDriver(driver, notifySpan);
      notifySpan.end();

      rootSpan.setTag('http.status_code', '200');
      rootSpan.end();

      this.requestDuration.observe(rootSpan.durationMs);
      this.activeRequests.dec();

      this.logger.info('Ride booking successful', {
        driver_id: driver.id,
        price,
        duration_ms: rootSpan.durationMs,
      });

      return { status: 'confirmed', driver, price };
    } catch (error) {
      rootSpan.setError(error);
      rootSpan.setTag('http.status_code', '500');
      rootSpan.end();

      this.requestDuration.observe(rootSpan.durationMs);
      this.activeRequests.dec();
      this.metrics.counter('http_errors_total', { error_type: error.name }).inc();

      this.logger.error('Ride booking failed', {
        error: error.message,
        stack: error.stack,
        duration_ms: rootSpan.durationMs,
      });

      throw error;
    }
  }

  async findDriver(request, parentSpan) {
    const geoSpan = this.tracer.startSpan('redis_geo_search', parentSpan);
    // Redis GEORADIUS simulate
    await this._delay(45);
    geoSpan.setTag('redis.command', 'GEORADIUS');
    geoSpan.setTag('radius_km', 5);
    geoSpan.end();

    const filterSpan = this.tracer.startSpan('filter_available_drivers', parentSpan);
    await this._delay(20);
    filterSpan.setTag('candidates', 8);
    filterSpan.setTag('available', 3);
    filterSpan.end();

    return { id: 'D-456', name: 'Karim', vehicle: 'Bike', rating: 4.8 };
  }

  async calculatePrice(request, parentSpan) {
    const surgeSpan = this.tracer.startSpan('check_surge_pricing', parentSpan);
    await this._delay(15);
    surgeSpan.setTag('surge_multiplier', 1.3);
    surgeSpan.end();

    return 180; // BDT
  }

  async notifyDriver(driver, parentSpan) {
    const fcmSpan = this.tracer.startSpan('send_fcm_push', parentSpan);
    await this._delay(80);
    fcmSpan.setTag('fcm.success', true);
    fcmSpan.end();
  }

  _delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// === Execution ===

async function main() {
  console.log('=== 🏍️ Pathao Ride Service — Observability Demo ===\n');

  const service = new ObservableRideService();

  // Simulate a booking
  const result = await service.handleBooking({
    userId: 'U-12345',
    pickup: 'Dhanmondi 27',
    destination: 'Banani 11',
    rideType: 'bike',
  });

  console.log('\n=== 📋 Booking Result ===');
  console.log(JSON.stringify(result, null, 2));

  // Print distributed trace
  service.tracer.printTrace();

  // Print metrics
  console.log('\n=== 📈 Metrics (Prometheus Format) ===');
  console.log(service.metrics.toPrometheus());

  // Percentile latency
  console.log('\n=== ⏱️ Latency Percentiles ===');
  console.log(`p50: ${service.requestDuration.percentile(50).toFixed(1)}ms`);
  console.log(`p95: ${service.requestDuration.percentile(95).toFixed(1)}ms`);
  console.log(`p99: ${service.requestDuration.percentile(99).toFixed(1)}ms`);
}

main().catch(console.error);
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### Log vs Metric vs Trace — কোনটি কখন?

```
┌────────────────────────────────────────────────────────────────┐
│  প্রশ্ন                          │  ব্যবহার করুন              │
├──────────────────────────────────┼─────────────────────────────┤
│  "গত ঘণ্টায় কত error হয়েছে?"   │  📈 Metric (counter)       │
│  "কেন এই specific request fail?" │  📝 Log (structured)       │
│  "কোন service slow?"            │  🔗 Trace (span durations) │
│  "Error rate বাড়ছে কিনা?"       │  📈 Metric (rate)          │
│  "Error-টা কী message ছিল?"     │  📝 Log (error details)    │
│  "Request কোন path follow করে?" │  🔗 Trace (service map)    │
│  "CPU usage এখন কত?"           │  📈 Metric (gauge)         │
│  "User কী action করেছিল?"       │  📝 Log (audit trail)      │
│  "কোথায় bottleneck?"           │  🔗 Trace (waterfall)      │
└──────────────────────────────────┴─────────────────────────────┘
```

### ✅ কখন Full Observability দরকার

| পরিস্থিতি | কেন |
|-----------|-----|
| Microservices (5+ services) | Request flow track করতে trace লাগবে |
| High traffic (1000+ RPS) | Metrics দিয়ে trend বুঝতে হবে |
| SLA commitment আছে | SLI measure করতে সব pillar দরকার |
| Distributed team | Everyone-কে debug করতে পারতে হবে |
| Complex business logic | Structured log দিয়ে state track |

### ❌ কখন Overkill

| পরিস্থিতি | কেন |
|-----------|-----|
| Single monolith, 10 users | Simple logging যথেষ্ট |
| MVP/Prototype | Build first, observe later |
| Static website | Uptime check যথেষ্ট |
| Budget constraint | Start with logs, add gradually |

### 💰 Observability-র খরচ

```
Data Volume (প্রতিদিন):
├── Logs: 10-100 GB (সবচেয়ে বেশি volume)
├── Metrics: 1-10 GB (compact time-series)
└── Traces: 5-50 GB (sampled হলে কম)

খরচ কমানোর উপায়:
1. Sampling: 100% trace collect করবেন না, 5-10% enough
2. Log levels: Production-এ DEBUG log বন্ধ
3. Retention: 30 দিনের বেশি পুরনো detailed data delete
4. Aggregation: Raw metrics → pre-aggregated summaries
```
