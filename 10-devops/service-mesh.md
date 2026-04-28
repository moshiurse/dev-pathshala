# 📘 Service Mesh

> **Service Mesh — মাইক্রোসার্ভিস আর্কিটেকচারে সার্ভিস-টু-সার্ভিস কমিউনিকেশন পরিচালনার ডেডিকেটেড ইনফ্রাস্ট্রাকচার লেয়ার।**

---

## 📖 সংজ্ঞা ও মূল ধারণা

Service Mesh হলো একটি **ইনফ্রাস্ট্রাকচার লেয়ার** যা মাইক্রোসার্ভিসগুলোর মধ্যে নেটওয়ার্ক কমিউনিকেশন পরিচালনা করে। এটি অ্যাপ্লিকেশন কোড পরিবর্তন না করেই **traffic management, security, observability** এবং **reliability** নিশ্চিত করে।

### মূল উপাদানসমূহ

- **Data Plane**: প্রতিটি সার্ভিসের পাশে চলা **sidecar proxy** (যেমন Envoy) যা সকল নেটওয়ার্ক ট্রাফিক intercept ও manage করে
- **Control Plane**: সকল sidecar proxy কনফিগার ও পরিচালনা করে (যেমন Istio এর istiod)
- **Sidecar Pattern**: প্রতিটি সার্ভিস Pod-এ একটি করে proxy container চলে যা মূল সার্ভিসের সাথে localhost-এ যুক্ত থাকে

### জনপ্রিয় Service Mesh সলিউশন

| সলিউশন | Control Plane | Data Plane | বৈশিষ্ট্য |
|---------|---------------|------------|-----------|
| **Istio** | istiod | Envoy | সবচেয়ে feature-rich, complex |
| **Linkerd** | Go-based | Rust proxy | লাইটওয়েট, সহজ |
| **Consul Connect** | Consul | Envoy/Built-in | HashiCorp ইকোসিস্টেম |
| **AWS App Mesh** | AWS managed | Envoy | AWS-native integration |

```
Service Mesh আর্কিটেকচার:

┌─────────────────────── Control Plane ───────────────────────┐
│                         istiod                               │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  Pilot   │  │  Citadel     │  │  Galley (Config)      │  │
│  │ (Traffic)│  │ (Security)   │  │  (Validation)         │  │
│  └──────────┘  └──────────────┘  └───────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ Configuration
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Pod A      │  │   Pod B      │  │   Pod C      │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Service  │ │  │ │ Service  │ │  │ │ Service  │ │
│ │   A      │ │  │ │   B      │ │  │ │   C      │ │
│ └────┬─────┘ │  │ └────┬─────┘ │  │ └────┬─────┘ │
│      │       │  │      │       │  │      │       │
│ ┌────▼─────┐ │  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │
│ │ Envoy    │◄├──├─► Envoy    │◄├──├─► Envoy    │ │
│ │ Sidecar  │ │  │ │ Sidecar  │ │  │ │ Sidecar  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
       ◄──── Data Plane (mTLS encrypted) ────►
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 🏙️ শহরের ট্রাফিক ম্যানেজমেন্ট সিস্টেম

কল্পনা করুন একটি বড় শহর যেখানে হাজারো গাড়ি চলাচল করে:

- **গাড়িগুলো = মাইক্রোসার্ভিস**: প্রতিটি গাড়ি নিজের গন্তব্যে যেতে চায়
- **ট্রাফিক সিগন্যাল ও পুলিশ = Sidecar Proxy**: প্রতিটি চৌরাস্তায় থাকা ট্রাফিক কন্ট্রোলার — গাড়িকে কোন রাস্তায় যেতে হবে তা নির্দেশ করে
- **কেন্দ্রীয় ট্রাফিক কন্ট্রোল রুম = Control Plane**: পুরো শহরের ট্রাফিক পরিস্থিতি মনিটর করে এবং সিগন্যালগুলোকে কনফিগার করে
- **Road block / One-way = Traffic Rules**: কোন রাস্তা দিয়ে কোন গাড়ি যেতে পারবে তা নিয়ন্ত্রণ (Authorization Policy)

গাড়ির ড্রাইভারকে (অ্যাপ্লিকেশন কোড) পুরো শহরের ট্রাফিক সিস্টেম জানতে হয় না — শুধু গন্তব্য জানলেই চলে। বাকি সব ট্রাফিক ম্যানেজমেন্ট সিস্টেম সামলায়।

---

## 🔧 Istio কনফিগারেশন উদাহরণ

### VirtualService — Traffic Routing

```yaml
# traffic-routing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
          weight: 100
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10
```

### DestinationRule — Load Balancing ও Circuit Breaking

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## 🐘 PHP উদাহরণ — Service Mesh-compatible Microservice

```php
<?php
/**
 * Service Mesh পরিবেশে চলার জন্য তৈরি একটি PHP মাইক্রোসার্ভিস।
 * Sidecar proxy (Envoy) সকল networking সামলায়,
 * তাই অ্যাপ্লিকেশন কোডে retry/circuit-breaker লাগে না।
 */

class OrderService
{
    private string $paymentServiceUrl;
    private string $inventoryServiceUrl;

    public function __construct()
    {
        // Service mesh-এ সার্ভিস নাম দিয়ে কল করা যায়
        // DNS resolution ও load balancing Envoy sidecar করে
        $this->paymentServiceUrl = 'http://payment-service:8080';
        $this->inventoryServiceUrl = 'http://inventory-service:8080';
    }

    public function createOrder(array $orderData): array
    {
        // Distributed tracing headers propagate করা — mesh observability-র জন্য গুরুত্বপূর্ণ
        $traceHeaders = $this->extractTraceHeaders();

        // Inventory চেক — retry/timeout Envoy সামলাবে
        $inventory = $this->callService(
            "{$this->inventoryServiceUrl}/api/check",
            ['product_id' => $orderData['product_id'], 'quantity' => $orderData['quantity']],
            $traceHeaders
        );

        if (!$inventory['available']) {
            return ['success' => false, 'error' => 'পণ্য স্টকে নেই'];
        }

        // Payment process — mTLS encryption Envoy sidecar করে
        $payment = $this->callService(
            "{$this->paymentServiceUrl}/api/charge",
            ['amount' => $orderData['total'], 'currency' => 'BDT'],
            $traceHeaders
        );

        return [
            'success' => true,
            'order_id' => uniqid('ORD-'),
            'payment_id' => $payment['transaction_id'],
            'trace_id' => $traceHeaders['x-request-id'] ?? 'unknown',
        ];
    }

    /**
     * Istio/Envoy distributed tracing-এর জন্য header propagation।
     * এই headers sidecar থেকে আসে এবং পরবর্তী সার্ভিসে পাঠাতে হয়।
     */
    private function extractTraceHeaders(): array
    {
        $tracingHeaders = [
            'x-request-id', 'x-b3-traceid', 'x-b3-spanid',
            'x-b3-parentspanid', 'x-b3-sampled', 'x-b3-flags',
        ];

        $headers = [];
        foreach ($tracingHeaders as $header) {
            $serverKey = 'HTTP_' . strtoupper(str_replace('-', '_', $header));
            if (isset($_SERVER[$serverKey])) {
                $headers[$header] = $_SERVER[$serverKey];
            }
        }
        return $headers;
    }

    private function callService(string $url, array $data, array $headers): array
    {
        $ch = curl_init($url);
        $curlHeaders = ['Content-Type: application/json'];
        foreach ($headers as $key => $value) {
            $curlHeaders[] = "{$key}: {$value}";
        }

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => $curlHeaders,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new RuntimeException("Service call failed: HTTP {$httpCode}");
        }
        return json_decode($response, true);
    }
}

// Health check endpoint — Kubernetes liveness/readiness probe
if ($_SERVER['REQUEST_URI'] === '/health') {
    header('Content-Type: application/json');
    echo json_encode(['status' => 'healthy', 'timestamp' => time()]);
    exit;
}
```

---

## 🟨 JavaScript উদাহরণ — Node.js Service with Mesh Awareness

```javascript
/**
 * Service Mesh (Istio) পরিবেশে চলার জন্য তৈরি Node.js সার্ভিস।
 * Envoy sidecar সকল networking concern সামলায়।
 */

const express = require('express');
const axios = require('axios');

const app = express();
app.use(express.json());

// Service mesh-এ Kubernetes service name ব্যবহার করা হয়
const SERVICES = {
  payment: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:8080',
  inventory: process.env.INVENTORY_SERVICE_URL || 'http://inventory-service:8080',
  notification: process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:8080',
};

// Istio tracing headers — প্রতিটি downstream call-এ propagate করতে হবে
const TRACE_HEADERS = [
  'x-request-id', 'x-b3-traceid', 'x-b3-spanid',
  'x-b3-parentspanid', 'x-b3-sampled', 'x-b3-flags',
  'x-ot-span-context',
];

/**
 * Incoming request থেকে tracing headers extract করে।
 * এটা service mesh observability-র জন্য অপরিহার্য।
 */
function extractTraceHeaders(req) {
  const headers = {};
  TRACE_HEADERS.forEach(header => {
    if (req.headers[header]) {
      headers[header] = req.headers[header];
    }
  });
  return headers;
}

/**
 * অন্য সার্ভিসে কল করার সময় trace headers সহ পাঠানো হয়।
 * Retry, timeout, circuit breaking — সবকিছু Envoy sidecar করবে।
 */
async function callService(serviceUrl, path, data, traceHeaders) {
  const response = await axios.post(`${serviceUrl}${path}`, data, {
    headers: {
      'Content-Type': 'application/json',
      ...traceHeaders,
    },
    // timeout কম রাখা হয়েছে কারণ Envoy-level timeout আছে
    timeout: 10000,
  });
  return response.data;
}

// অর্ডার তৈরির endpoint
app.post('/api/orders', async (req, res) => {
  const traceHeaders = extractTraceHeaders(req);
  const { productId, quantity, amount } = req.body;

  try {
    // ১. Inventory চেক
    const inventory = await callService(
      SERVICES.inventory, '/api/check',
      { product_id: productId, quantity },
      traceHeaders
    );

    if (!inventory.available) {
      return res.status(400).json({ error: 'পণ্য স্টকে নেই' });
    }

    // ২. Payment প্রসেস
    const payment = await callService(
      SERVICES.payment, '/api/charge',
      { amount, currency: 'BDT' },
      traceHeaders
    );

    // ৩. Notification পাঠানো (fire-and-forget)
    callService(
      SERVICES.notification, '/api/notify',
      { type: 'order_created', orderId: payment.transaction_id },
      traceHeaders
    ).catch(err => console.warn('Notification failed:', err.message));

    res.json({
      success: true,
      orderId: `ORD-${Date.now()}`,
      paymentId: payment.transaction_id,
      traceId: traceHeaders['x-request-id'] || 'unknown',
    });
  } catch (error) {
    console.error('Order creation failed:', error.message);
    res.status(500).json({ error: 'অর্ডার তৈরি ব্যর্থ হয়েছে' });
  }
});

// Health ও readiness endpoints — Kubernetes probes
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', uptime: process.uptime() });
});

app.get('/ready', (req, res) => {
  res.json({ status: 'ready', timestamp: new Date().toISOString() });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Order service running on port ${PORT}`);
});
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|-----------|------|
| **বড় মাইক্রোসার্ভিস আর্কিটেকচার** (১০+ সার্ভিস) | সার্ভিস-টু-সার্ভিস কমিউনিকেশন ম্যানেজমেন্ট জটিল হয়ে যায় |
| **mTLS ও zero-trust security দরকার** | প্রতিটি সার্ভিসের মধ্যে encrypted communication নিশ্চিত করে |
| **Advanced traffic management** প্রয়োজন | Canary deploy, traffic splitting, A/B testing সহজ হয় |
| **Distributed tracing ও observability** চাই | সকল সার্ভিসের মধ্যে request flow ট্র্যাক করা যায় |
| **Multi-language মাইক্রোসার্ভিস** | PHP, Node.js, Go, Java — সব ভাষায় একই networking policy প্রয়োগ হয় |
| **Compliance requirement** আছে | Audit trail, encryption, access control কেন্দ্রীয়ভাবে পরিচালনা |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| **ছোট অ্যাপ্লিকেশন বা মনোলিথ** | অপ্রয়োজনীয় complexity যোগ হয় |
| **৫-৬টির কম সার্ভিস** | সাধারণ HTTP client library দিয়েই কাজ চলে |
| **টিমের Kubernetes অভিজ্ঞতা কম** | Service mesh শেখার curve অনেক বেশি |
| **Resource-constrained পরিবেশ** | প্রতিটি sidecar proxy অতিরিক্ত CPU ও memory খরচ করে |
| **Latency-critical অ্যাপ্লিকেশন** | প্রতিটি hop-এ sidecar proxy ~1-3ms latency যোগ করে |
| **Simple request-response pattern** | ওভারহেড justify হয় না |

---

## 📊 Service Mesh ছাড়া vs সহ তুলনা

```
Service Mesh ছাড়া:
┌──────────┐    Direct call    ┌──────────┐
│ Service A│ ─────────────────►│ Service B│
│          │  No encryption    │          │
│          │  No retry logic   │          │
│          │  No tracing       │          │
└──────────┘                   └──────────┘
    সমস্যা: প্রতিটি সার্ভিসে retry, timeout, auth কোড লিখতে হয়

Service Mesh সহ:
┌──────────────────┐           ┌──────────────────┐
│  Pod A           │           │  Pod B           │
│ ┌──────┐ ┌─────┐│  mTLS     │┌─────┐ ┌──────┐ │
│ │Svc A │→│Envoy│├───────────►│Envoy│→│Svc B │ │
│ └──────┘ └─────┘│  Retry     │└─────┘ └──────┘ │
│                  │  Tracing   │                  │
└──────────────────┘  Metrics  └──────────────────┘
    সুবিধা: সকল networking concern infrastructure-এ চলে যায়
```

---

## 🔑 মূল শিক্ষা

1. **Service Mesh অ্যাপ্লিকেশন কোড থেকে networking logic আলাদা করে** — এটাই এর সবচেয়ে বড় সুবিধা
2. **Sidecar pattern** বোঝা অত্যন্ত গুরুত্বপূর্ণ — প্রতিটি সার্ভিসের পাশে একটি proxy চলে
3. **Trace header propagation** অ্যাপ্লিকেশন কোডে করতে হয় — এটা mesh নিজে করতে পারে না
4. **Resource overhead** সবসময় হিসাব করুন — ছোট ক্লাস্টারে mesh ব্যবহার না করাই ভালো
5. **শুরুতে Linkerd** ব্যবহার করুন — Istio-র চেয়ে সহজ ও লাইটওয়েট
