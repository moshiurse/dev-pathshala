# 📘 মনিটরিং ও লগিং

> **"যা পরিমাপ করা যায় না, তা উন্নতি করা যায় না।" — সফটওয়্যার সিস্টেমের স্বাস্থ্য পর্যবেক্ষণ।**

---

## 📖 সংজ্ঞা

### মনিটরিং কী?

মনিটরিং হলো সিস্টেমের **মেট্রিক্স সংগ্রহ, বিশ্লেষণ ও অ্যালার্ট** করার প্রক্রিয়া — যাতে সমস্যা হওয়ার **আগে বা তাৎক্ষণিকভাবে** জানা যায়।

### লগিং কী?

লগিং হলো সিস্টেমের **ইভেন্ট ও কার্যকলাপ রেকর্ড** করা — যাতে সমস্যা **ডিবাগ ও বিশ্লেষণ** করা যায়।

### তিনটি স্তম্ভ (Three Pillars of Observability)

```
        Observability (পর্যবেক্ষণযোগ্যতা)
              /        |         \
             /         |          \
       Metrics      Logs       Traces
       (মেট্রিক্স)  (লগ)      (ট্রেস)

Metrics: সংখ্যাভিত্তিক ডাটা — CPU 85%, Response Time 200ms
Logs:    ইভেন্ট রেকর্ড — "ইউজার রহিম লগইন করেছে 10:30 AM"
Traces:  রিকোয়েস্ট জার্নি — API → Auth → DB → Cache → Response
```

---

## 📊 Prometheus ও Grafana

### Prometheus — মেট্রিক্স সংগ্রহ ও সংরক্ষণ

```yaml
# prometheus.yml — Prometheus কনফিগারেশন
global:
  scrape_interval: 15s           # প্রতি ১৫ সেকেন্ডে মেট্রিক্স সংগ্রহ
  evaluation_interval: 15s

# অ্যালার্ট রুল ফাইল
rule_files:
  - "alert_rules.yml"

# Alertmanager কনফিগারেশন
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# মেট্রিক্স সোর্স (Scrape Targets)
scrape_configs:
  # Prometheus নিজের মেট্রিক্স
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node.js অ্যাপ
  - job_name: 'node-api'
    static_configs:
      - targets: ['api:3000']
    metrics_path: '/metrics'
    scrape_interval: 10s

  # PHP-FPM
  - job_name: 'php-fpm'
    static_configs:
      - targets: ['php-fpm-exporter:9253']

  # MySQL
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # Nginx
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']

  # Node Exporter (সার্ভার মেট্রিক্স)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Kubernetes Service Discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### অ্যালার্ট রুলস

```yaml
# alert_rules.yml
groups:
  - name: অ্যাপ্লিকেশন অ্যালার্ট
    rules:
      # হাই এরর রেট
      - alert: HighErrorRate
        expr: |
          (sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))) * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "এরর রেট ৫% এর উপরে!"
          description: "গত ৫ মিনিটে {{ $value }}% রিকোয়েস্ট ব্যর্থ হচ্ছে।"

      # ধীর রেসপন্স টাইম
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 রেসপন্স টাইম ২ সেকেন্ডের উপরে!"

      # হাই CPU
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU ব্যবহার ৮৫% এর উপরে!"

      # হাই মেমোরি
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "মেমোরি ব্যবহার ৯০% এর উপরে!"

      # ডিস্ক স্পেস কম
      - alert: LowDiskSpace
        expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "ডিস্ক স্পেস ৮৫% ভর্তি!"

      # Pod ক্র্যাশ লুপ
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} বারবার ক্র্যাশ করছে!"
```

### Node.js এ কাস্টম মেট্রিক্স

```javascript
// metrics.js — Prometheus মেট্রিক্স (prom-client লাইব্রেরি)
const promClient = require('prom-client');

// ডিফল্ট মেট্রিক্স (CPU, মেমোরি, ইভেন্ট লুপ)
promClient.collectDefaultMetrics({ prefix: 'app_' });

// কাস্টম মেট্রিক্স

// ১. Counter — শুধু বাড়ে (মোট রিকোয়েস্ট সংখ্যা)
const httpRequestsTotal = new promClient.Counter({
    name: 'http_requests_total',
    help: 'মোট HTTP রিকোয়েস্ট',
    labelNames: ['method', 'path', 'status']
});

// ২. Histogram — বিতরণ (রেসপন্স টাইম)
const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP রিকোয়েস্ট সময়কাল (সেকেন্ড)',
    labelNames: ['method', 'path', 'status'],
    buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});

// ৩. Gauge — বাড়ে ও কমে (সক্রিয় কানেকশন)
const activeConnections = new promClient.Gauge({
    name: 'active_connections',
    help: 'বর্তমান সক্রিয় কানেকশন সংখ্যা'
});

// ৪. Summary — percentiles (DB কোয়েরি সময়)
const dbQueryDuration = new promClient.Summary({
    name: 'db_query_duration_seconds',
    help: 'ডাটাবেস কোয়েরি সময়কাল',
    labelNames: ['operation', 'collection'],
    percentiles: [0.5, 0.9, 0.95, 0.99]
});

// Express Middleware — অটো মেট্রিক্স সংগ্রহ
function metricsMiddleware(req, res, next) {
    const start = process.hrtime.bigint();
    activeConnections.inc();

    res.on('finish', () => {
        const duration = Number(process.hrtime.bigint() - start) / 1e9;
        const path = req.route?.path || req.path;

        httpRequestsTotal.inc({
            method: req.method,
            path: path,
            status: res.statusCode
        });

        httpRequestDuration.observe({
            method: req.method,
            path: path,
            status: res.statusCode
        }, duration);

        activeConnections.dec();
    });

    next();
}

// /metrics endpoint
function metricsEndpoint(req, res) {
    res.set('Content-Type', promClient.register.contentType);
    promClient.register.metrics().then(metrics => res.end(metrics));
}

module.exports = {
    metricsMiddleware,
    metricsEndpoint,
    httpRequestsTotal,
    httpRequestDuration,
    activeConnections,
    dbQueryDuration
};

// app.js
const express = require('express');
const { metricsMiddleware, metricsEndpoint } = require('./metrics');

const app = express();

app.use(metricsMiddleware);
app.get('/metrics', metricsEndpoint);

app.get('/api/users', async (req, res) => {
    // ... API লজিক
    res.json({ users: [] });
});
```

### PHP Laravel এ মনিটরিং

```php
<?php

// app/Http/Middleware/MetricsMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class MetricsMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        $startTime = microtime(true);

        $response = $next($request);

        $duration = microtime(true) - $startTime;
        $statusCode = $response->getStatusCode();

        // স্ট্রাকচার্ড লগ (JSON)
        Log::channel('metrics')->info('http_request', [
            'method' => $request->method(),
            'path' => $request->path(),
            'status' => $statusCode,
            'duration_ms' => round($duration * 1000, 2),
            'ip' => $request->ip(),
            'user_id' => $request->user()?->id,
            'memory_mb' => round(memory_get_peak_usage(true) / 1048576, 2),
        ]);

        // ধীর রিকোয়েস্ট অ্যালার্ট
        if ($duration > 2.0) {
            Log::channel('slack')->warning('ধীর রিকোয়েস্ট!', [
                'path' => $request->fullUrl(),
                'duration_s' => round($duration, 2),
                'method' => $request->method(),
            ]);
        }

        return $response;
    }
}

// config/logging.php — লগ চ্যানেল কনফিগারেশন
'channels' => [
    'metrics' => [
        'driver' => 'daily',
        'path' => storage_path('logs/metrics.log'),
        'days' => 30,
    ],
    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'level' => 'warning',
    ],
    'elasticsearch' => [
        'driver' => 'monolog',
        'handler' => \Monolog\Handler\ElasticsearchHandler::class,
        'handler_with' => [
            'client' => \Elasticsearch\ClientBuilder::create()
                ->setHosts([env('ELASTICSEARCH_HOST', 'localhost:9200')])
                ->build(),
        ],
    ],
],
```

---

## 📝 ELK Stack — লগ ম্যানেজমেন্ট

### ELK কী?

```
E = Elasticsearch — লগ সংরক্ষণ ও সার্চ
L = Logstash      — লগ সংগ্রহ ও প্রসেসিং
K = Kibana        — লগ ভিজুয়ালাইজেশন ও ড্যাশবোর্ড

ফ্লো:
App → Filebeat/Fluentd → Logstash → Elasticsearch → Kibana
                                                       ↑
                                                 ড্যাশবোর্ড ও সার্চ
```

### Docker Compose — ELK Stack

```yaml
# docker-compose.elk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      elasticsearch:
        condition: service_healthy

  # Filebeat — লগ ফাইল থেকে সংগ্রহ
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.12.0
    container_name: filebeat
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log:/var/log:ro
      - ./storage/logs:/app/logs:ro  # Laravel লগ
    depends_on:
      - logstash

volumes:
  es-data:
```

### Logstash Pipeline

```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }

  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Laravel লগ পার্সিং
  if [fields][type] == "laravel" {
    grok {
      match => {
        "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{DATA:environment}\.%{DATA:level}: %{GREEDYDATA:log_message}"
      }
    }

    date {
      match => ["timestamp", "ISO8601"]
      target => "@timestamp"
    }

    # JSON লগ মেসেজ পার্স
    if [log_message] =~ /^\{/ {
      json {
        source => "log_message"
        target => "log_data"
      }
    }
  }

  # Node.js লগ পার্সিং
  if [fields][type] == "nodejs" {
    json {
      source => "message"
    }
  }

  # Geo IP (IP থেকে লোকেশন)
  if [client_ip] {
    geoip {
      source => "client_ip"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[fields][type]}-%{+YYYY.MM.dd}"
  }
}
```

---

## 🔬 স্ট্রাকচার্ড লগিং (Structured Logging)

### কেন স্ট্রাকচার্ড লগিং?

```
❌ আনস্ট্রাকচার্ড (পার্স করা কঠিন):
"User rahim logged in from 192.168.1.1 at 2024-01-15 10:30:00"

✅ স্ট্রাকচার্ড JSON (সহজে সার্চ ও ফিল্টার):
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "info",
  "event": "user_login",
  "user_id": 42,
  "user_name": "rahim",
  "ip": "192.168.1.1",
  "user_agent": "Chrome/120"
}
```

### Node.js — Winston Logger

```javascript
// logger.js
const winston = require('winston');

const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
    ),
    defaultMeta: {
        service: 'api-service',
        environment: process.env.NODE_ENV
    },
    transports: [
        // ফাইলে লগ
        new winston.transports.File({
            filename: 'logs/error.log',
            level: 'error',
            maxsize: 10485760,  // 10MB
            maxFiles: 5
        }),
        new winston.transports.File({
            filename: 'logs/combined.log',
            maxsize: 10485760,
            maxFiles: 10
        }),
    ]
});

// ডেভেলপমেন্টে কনসোলে রঙিন আউটপুট
if (process.env.NODE_ENV !== 'production') {
    logger.add(new winston.transports.Console({
        format: winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
        )
    }));
}

// ব্যবহার
logger.info('ইউজার লগইন সফল', {
    userId: 42,
    email: 'rahim@example.com',
    ip: '192.168.1.1',
    method: 'password'
});

logger.error('পেমেন্ট ব্যর্থ', {
    orderId: 'ORD-123',
    amount: 1500,
    gateway: 'bkash',
    error: 'Insufficient balance'
});

logger.warn('রেট লিমিট কাছে', {
    ip: '192.168.1.1',
    currentRate: 95,
    limit: 100
});

module.exports = logger;
```

### PHP — Monolog

```php
<?php

// app/Logging/CustomFormatter.php
namespace App\Logging;

use Monolog\Formatter\JsonFormatter;

class CustomFormatter
{
    public function __invoke($logger)
    {
        foreach ($logger->getHandlers() as $handler) {
            $handler->setFormatter(new JsonFormatter());
        }
    }
}

// ব্যবহার
Log::info('ইউজার লগইন সফল', [
    'user_id' => $user->id,
    'email' => $user->email,
    'ip' => request()->ip(),
    'user_agent' => request()->userAgent(),
]);

Log::error('পেমেন্ট ব্যর্থ', [
    'order_id' => $order->id,
    'amount' => $order->total,
    'gateway' => 'bkash',
    'error' => $exception->getMessage(),
    'trace' => $exception->getTraceAsString(),
]);
```

---

## 🔔 অ্যালার্টিং ও নোটিফিকেশন

### Alertmanager কনফিগারেশন

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical → Slack + PagerDuty
    - match:
        severity: critical
      receiver: 'critical-alerts'
      repeat_interval: 1h

    # Warning → শুধু Slack
    - match:
        severity: warning
      receiver: 'warning-alerts'
      repeat_interval: 4h

receivers:
  - name: 'default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts'

  - name: 'critical-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts-critical'
        title: '🔴 ক্রিটিক্যাল অ্যালার্ট!'
        text: '{{ .CommonAnnotations.summary }}'
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'

  - name: 'warning-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
        channel: '#alerts-warning'
        title: '🟡 ওয়ার্নিং'
        text: '{{ .CommonAnnotations.summary }}'
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **সমস্যা দ্রুত ধরা** | অ্যালার্ট দিয়ে মিনিটে জানা যায় |
| ২ | **প্রোঅ্যাক্টিভ** | সমস্যা বড় হওয়ার আগেই সমাধান |
| ৩ | **ডিবাগিং সহজ** | স্ট্রাকচার্ড লগ ও ট্রেসিং দিয়ে |
| ৪ | **পারফরম্যান্স বিশ্লেষণ** | বটলনেক খুঁজে বের করা যায় |
| ৫ | **ক্যাপাসিটি প্ল্যানিং** | ট্রেন্ড দেখে ভবিষ্যৎ রিসোর্স পরিকল্পনা |
| ৬ | **SLA ট্র্যাকিং** | আপটাইম ও পারফরম্যান্স পরিমাপ |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **অ্যালার্ট ফ্যাটিগ** | বেশি অ্যালার্ট = উপেক্ষা করা শুরু হয় |
| ২ | **স্টোরেজ খরচ** | লগ ও মেট্রিক্স অনেক জায়গা নেয় |
| ৩ | **জটিল সেটআপ** | Prometheus + Grafana + ELK সেটআপ কঠিন |
| ৪ | **পারফরম্যান্স ইম্প্যাক্ট** | অতিরিক্ত লগিং অ্যাপ ধীর করতে পারে |
| ৫ | **ডাটা ওভারলোড** | অনেক ডাটা = অর্থপূর্ণ insight খুঁজে পাওয়া কঠিন |

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **Metrics** | সংখ্যাভিত্তিক ডাটা — Prometheus, Grafana |
| **Logs** | ইভেন্ট রেকর্ড — ELK Stack, Loki |
| **Traces** | রিকোয়েস্ট জার্নি — Jaeger, Zipkin |
| **অ্যালার্ট** | Alertmanager, PagerDuty, Slack |
| **মূলনীতি** | স্ট্রাকচার্ড লগিং, অর্থপূর্ণ মেট্রিক্স, কম অ্যালার্ট ফ্যাটিগ |
