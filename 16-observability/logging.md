# 📌 লগিং (Logging) — Observability-র ভিত্তি

> "যদি তুমি দেখতে না পাও সিস্টেমে কী হচ্ছে, তাহলে তুমি ঠিক করতেও পারবে না।"

লগিং হলো সফটওয়্যার সিস্টেমের **চোখ ও কান**। প্রোডাকশনে কোনো সমস্যা হলে প্রথমে আমরা
লগ দেখি। bKash-এ একটা পেমেন্ট ফেইল হলে, Pathao-তে রাইড ম্যাচ না হলে, Daraz-এ অর্ডার
হারিয়ে গেলে — সব ক্ষেত্রেই লগই প্রথম সূত্র দেয়।

---

## 📖 সূচিপত্র

1. [স্ট্রাকচার্ড বনাম আনস্ট্রাকচার্ড লগিং](#-স্ট্রাকচার্ড-বনাম-আনস্ট্রাকচার্ড-লগিং)
2. [লগ লেভেল](#-লগ-লেভেল-log-levels)
3. [লগিং সেরা অনুশীলন](#-লগিং-সেরা-অনুশীলন-best-practices)
4. [কী লগ করবেন / করবেন না](#-কী-লগ-করবেন--কী-করবেন-না)
5. [Correlation ID](#-correlation-id--রিকোয়েস্ট-ট্রেসিং)
6. [সেন্ট্রালাইজড লগিং — ELK Stack](#-সেন্ট্রালাইজড-লগিং--elk-stack)
7. [Log Aggregation Patterns](#-log-aggregation-patterns)
8. [PHP — Monolog](#-php--monolog-সেটআপ)
9. [JS — Winston ও Pino](#-javascript--winston-ও-pino)
10. [বাস্তব উদাহরণ](#-বাস্তব-উদাহরণ-real-world-scenarios)

---

## 📊 স্ট্রাকচার্ড বনাম আনস্ট্রাকচার্ড লগিং

### আনস্ট্রাকচার্ড লগিং (পুরনো পদ্ধতি)

সাধারণ টেক্সট হিসেবে লগ লেখা। মানুষ পড়তে পারে কিন্তু মেশিন সহজে পার্স করতে পারে না।

```
[2024-01-15 10:30:45] ERROR: Payment failed for user 12345, amount 500 BDT
[2024-01-15 10:30:46] INFO: User logged in from Dhaka
[2024-01-15 10:30:47] WARN: Slow query detected, took 3.5 seconds
```

### স্ট্রাকচার্ড লগিং (আধুনিক পদ্ধতি)

JSON বা key-value ফরম্যাটে লগ লেখা। মেশিন সহজেই সার্চ, ফিল্টার, এবং অ্যানালাইজ করতে পারে।

```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "ERROR",
  "message": "Payment failed",
  "service": "payment-service",
  "userId": 12345,
  "amount": 500,
  "currency": "BDT",
  "gateway": "bKash",
  "errorCode": "INSUFFICIENT_BALANCE",
  "correlationId": "req-abc-123",
  "duration_ms": 1250
}
```

### তুলনা ডায়াগ্রাম

```
  আনস্ট্রাকচার্ড (Unstructured)          স্ট্রাকচার্ড (Structured)
  ================================         ================================

  "Payment failed for user 12345"          { "event": "payment_failed",
                                              "userId": 12345,
       ┌──────────────┐                       "amount": 500 }
       │  Plain Text   │
       │  Free-form    │                        ┌──────────────┐
       │  কোনো ফরম্যাট│                        │  JSON / KV    │
       │  নির্দিষ্ট নয়│                        │  পার্সযোগ্য   │
       └──────┬───────┘                        │  সার্চযোগ্য   │
              │                                 └──────┬───────┘
              ▼                                        ▼
     ❌ grep দিয়ে খুঁজতে               ✅ Elasticsearch-এ
        কষ্ট হয়                            সহজেই query করা যায়
     ❌ অ্যালার্ট সেট করা                ✅ ড্যাশবোর্ড বানানো
        কঠিন                                সহজ
     ❌ স্কেলে ব্যর্থ                    ✅ মিলিয়ন লগেও কাজ করে
```

### ❌ Bad: আনস্ট্রাকচার্ড লগিং

```php
// ❌ খারাপ — কোনো স্ট্রাকচার নেই, পার্স করা কঠিন
error_log("Payment failed for user " . $userId . " amount " . $amount);
error_log("ERROR at " . date('Y-m-d') . ": something went wrong");
echo "DEBUG: query took " . $time . " seconds";
```

### ✅ Good: স্ট্রাকচার্ড লগিং

```php
// ✅ ভালো — JSON স্ট্রাকচার্ড, সহজে পার্সযোগ্য
$logger->error('Payment failed', [
    'userId'        => $userId,
    'amount'        => $amount,
    'currency'      => 'BDT',
    'gateway'       => 'bKash',
    'errorCode'     => $errorCode,
    'correlationId' => $requestId,
]);
```

---

## 🎯 লগ লেভেল (Log Levels)

লগ লেভেল হলো লগের **গুরুত্ব নির্দেশক**। সঠিক লেভেল ব্যবহার না করলে হয় অতিরিক্ত
noise তৈরি হয়, নয়তো গুরুত্বপূর্ণ তথ্য হারিয়ে যায়।

```
  গুরুত্ব (Severity) ──────────────────────────────────────────▶

  ┌─────────┬─────────┬─────────┬─────────┬─────────┐
  │  DEBUG  │  INFO   │  WARN   │  ERROR  │  FATAL  │
  │         │         │         │         │         │
  │ সবচেয়ে │ সাধারণ  │ সতর্কতা │ ত্রুটি  │ সিস্টেম │
  │ বেশি    │ তথ্য    │         │         │  ক্র্যাশ│
  │ verbose │         │         │         │         │
  └─────────┴─────────┴─────────┴─────────┴─────────┘
      ▲                                       ▲
      │                                       │
  প্রোডাকশনে                           সবসময়
  বন্ধ রাখুন                           লগ করুন
```

### লগ লেভেল ডিসিশন টেবিল

```
┌──────────┬───────────────────────────────────┬──────────────────────────────┬────────────┐
│  Level   │  কখন ব্যবহার করবেন               │  উদাহরণ                      │ Production │
├──────────┼───────────────────────────────────┼──────────────────────────────┼────────────┤
│  DEBUG   │ ডেভেলপমেন্টে ডিবাগিং-এর জন্য     │ "SQL query: SELECT * FROM.." │  ❌ OFF    │
│          │ ভ্যারিয়েবল ভ্যালু, ফ্লো ট্রেসিং  │ "Request payload: {...}"     │            │
├──────────┼───────────────────────────────────┼──────────────────────────────┼────────────┤
│  INFO    │ স্বাভাবিক অপারেশন রেকর্ড করতে    │ "User logged in"             │  ✅ ON     │
│          │ গুরুত্বপূর্ণ বিজনেস ইভেন্ট       │ "Order placed #ORD-789"      │            │
├──────────┼───────────────────────────────────┼──────────────────────────────┼────────────┤
│  WARN    │ সমস্যা হতে পারে এমন পরিস্থিতি    │ "Disk usage 85%"             │  ✅ ON     │
│          │ এখনো কাজ করছে কিন্তু ঝুঁকিতে    │ "API response slow: 4.2s"    │            │
├──────────┼───────────────────────────────────┼──────────────────────────────┼────────────┤
│  ERROR   │ অপারেশন ব্যর্থ হয়েছে            │ "Payment gateway timeout"    │  ✅ ON     │
│          │ কিন্তু সিস্টেম চলছে              │ "Database connection lost"   │  + Alert   │
├──────────┼───────────────────────────────────┼──────────────────────────────┼────────────┤
│  FATAL   │ সিস্টেম বন্ধ হয়ে গেছে বা যাচ্ছে │ "Out of memory"              │  ✅ ON     │
│          │ তাৎক্ষণিক হস্তক্ষেপ দরকার       │ "Database server unreachable"│  + PagerDuty│
└──────────┴───────────────────────────────────┴──────────────────────────────┴────────────┘
```

### প্রতিটি লেভেলের বিস্তারিত উদাহরণ

```php
// 🔵 DEBUG — শুধু ডেভেলপমেন্টে
$logger->debug('bKash API request', [
    'endpoint' => '/checkout/payment/create',
    'payload'  => $requestData,
    'headers'  => $headers,
]);

// 🟢 INFO — স্বাভাবিক বিজনেস ইভেন্ট
$logger->info('Payment completed', [
    'orderId'   => 'ORD-2024-7890',
    'userId'    => 456,
    'amount'    => 1500.00,
    'gateway'   => 'Nagad',
    'method'    => 'mobile_wallet',
]);

// 🟡 WARN — সমস্যার আশঙ্কা
$logger->warning('bKash API response slow', [
    'endpoint'    => '/checkout/payment/create',
    'duration_ms' => 4200,
    'threshold'   => 2000,
]);

// 🔴 ERROR — অপারেশন ব্যর্থ
$logger->error('Payment processing failed', [
    'orderId'   => 'ORD-2024-7891',
    'gateway'   => 'bKash',
    'errorCode' => 'GATEWAY_TIMEOUT',
    'retryCount'=> 3,
    'exception' => $e->getMessage(),
]);

// 💀 FATAL — সিস্টেম ক্র্যাশ
$logger->critical('Database connection pool exhausted', [
    'activeConnections' => 100,
    'maxConnections'    => 100,
    'waitingRequests'   => 250,
    'service'           => 'order-service',
]);
```

---

## 📌 লগিং সেরা অনুশীলন (Best Practices)

### ১. প্রতিটি লগে Context দিন

```php
// ❌ Bad — কোনো context নেই
$logger->error('Payment failed');

// ✅ Good — পর্যাপ্ত context আছে
$logger->error('Payment failed', [
    'orderId'       => $orderId,
    'userId'        => $userId,
    'amount'        => $amount,
    'gateway'       => 'bKash',
    'failureReason' => $reason,
    'correlationId' => $correlationId,
]);
```

### ২. Sensitive ডেটা কখনো লগ করবেন না

```php
// ❌ Bad — পাসওয়ার্ড লগ করা হচ্ছে!
$logger->info('Login attempt', ['password' => $password]);

// ✅ Good — শুধু নিরাপদ তথ্য
$logger->info('Login attempt', ['username' => $username, 'ip' => $ip]);
```

### ৩. Exception-র পুরো Stack Trace লগ করুন

```php
// ❌ Bad — শুধু মেসেজ
$logger->error('Error: ' . $e->getMessage());

// ✅ Good — পুরো exception context
$logger->error('Payment processing exception', [
    'message'   => $e->getMessage(),
    'code'      => $e->getCode(),
    'file'      => $e->getFile(),
    'line'      => $e->getLine(),
    'trace'     => $e->getTraceAsString(),
    'orderId'   => $orderId,
]);
```

### ৪. একই ইভেন্ট বারবার লগ করবেন না

```php
// ❌ Bad — লুপে প্রতিটা আইটেমে লগ
foreach ($items as $item) {
    $logger->info('Processing item', ['id' => $item->id]);
    // ... process ...
    $logger->info('Item processed', ['id' => $item->id]);
}

// ✅ Good — সারাংশ লগ করুন
$logger->info('Batch processing started', ['totalItems' => count($items)]);
// ... process all ...
$logger->info('Batch processing completed', [
    'totalItems'  => count($items),
    'successful'  => $successCount,
    'failed'      => $failCount,
    'duration_ms' => $elapsed,
]);
```

### ৫. Performance-sensitive পাথে অতিরিক্ত লগ এড়িয়ে চলুন

```php
// ❌ Bad — হাই-ট্রাফিক এন্ডপয়েন্টে DEBUG লগ
function handleHealthCheck() {
    $logger->debug('Health check requested');       // প্রতি সেকেন্ডে হাজারবার!
    return response()->json(['status' => 'ok']);
}

// ✅ Good — শুধু সমস্যায় লগ করুন
function handleHealthCheck() {
    $status = checkAllServices();
    if (!$status->healthy) {
        $logger->warning('Health check degraded', [
            'failedServices' => $status->failed,
        ]);
    }
    return response()->json($status);
}
```

### ৬. Timestamp সবসময় UTC-তে রাখুন

```php
// ❌ Bad — লোকাল টাইমজোন
$logger->info('Event at ' . date('Y-m-d H:i:s'));  // BST? UTC? কোনটা?

// ✅ Good — ISO 8601 UTC ফরম্যাট (Monolog স্বয়ংক্রিয়ভাবে করে)
// Monolog JsonFormatter UTC timestamp ব্যবহার করে
```

---

## 🚫 কী লগ করবেন / কী করবেন না

### কী লগ করবেন ✅

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✅ অবশ্যই লগ করুন                            │
├─────────────────────────────────────────────────────────────────┤
│  ► Authentication ইভেন্ট (login, logout, failed attempts)      │
│  ► Authorization ব্যর্থতা (403 forbidden)                      │
│  ► বিজনেস ট্রানজ্যাকশন (অর্ডার, পেমেন্ট, রিফান্ড)            │
│  ► External API কল (request/response time, status)             │
│  ► Error ও Exception (পুরো stack trace সহ)                     │
│  ► Performance মেট্রিক (slow queries, high latency)            │
│  ► সিস্টেম স্টার্ট/শাটডাউন                                    │
│  ► Configuration পরিবর্তন                                      │
│  ► Background job execution (start, complete, fail)            │
│  ► Rate limiting ইভেন্ট                                        │
└─────────────────────────────────────────────────────────────────┘
```

### কী কখনো লগ করবেন না ❌

```
┌─────────────────────────────────────────────────────────────────┐
│                 ❌ কখনো লগ করবেন না                              │
├─────────────────────────────────────────────────────────────────┤
│  ✗ পাসওয়ার্ড (plain text বা hashed)                            │
│  ✗ API Keys / Secret tokens                                    │
│  ✗ ক্রেডিট কার্ড নম্বর                                        │
│  ✗ জাতীয় পরিচয়পত্র (NID) নম্বর                                │
│  ✗ ব্যক্তিগত ফোন নম্বর (সম্পূর্ণ)                              │
│  ✗ ইমেইল এড্রেস (সম্পূর্ণ — মাস্ক করুন)                       │
│  ✗ Session tokens / JWT tokens                                 │
│  ✗ ডাটাবেজ কানেকশন স্ট্রিং                                    │
│  ✗ ব্যাংক একাউন্ট নম্বর                                       │
│  ✗ bKash/Nagad পিন বা OTP                                      │
└─────────────────────────────────────────────────────────────────┘
```

### ❌/✅ তুলনা উদাহরণ

```php
// ❌ Bad: পাসওয়ার্ড লগ করা
$logger->info('User login', [
    'email'    => 'rahim@example.com',
    'password' => 'secret123',
]);

// ✅ Good: পাসওয়ার্ড বাদ, ইমেইল মাস্ক
$logger->info('User login', [
    'email' => 'ra***@example.com',
    'ip'    => $request->ip(),
]);
```

```php
// ❌ Bad: bKash পিন লগ করা
$logger->info('bKash payment', [
    'phone' => '01712345678',
    'pin'   => '12345',
    'amount'=> 5000,
]);

// ✅ Good: সংবেদনশীল তথ্য মাস্ক
$logger->info('bKash payment initiated', [
    'phone'   => '0171***5678',
    'amount'  => 5000,
    'txnId'   => $transactionId,
]);
```

```php
// ❌ Bad: পুরো API response বডি (সংবেদনশীল ডেটা থাকতে পারে)
$logger->debug('API response', ['body' => $fullResponseBody]);

// ✅ Good: শুধু প্রয়োজনীয় তথ্য
$logger->debug('API response', [
    'statusCode'  => $response->getStatusCode(),
    'duration_ms' => $elapsed,
    'endpoint'    => '/api/v1/payments',
]);
```

---

## 🔗 Correlation ID — রিকোয়েস্ট ট্রেসিং

Correlation ID হলো একটি **ইউনিক আইডেন্টিফায়ার** যা একটি রিকোয়েস্ট সব সার্ভিসের মধ্য দিয়ে
যাওয়ার সময় বহন করে। মাইক্রোসার্ভিসে একটি ইউজার রিকোয়েস্ট ৫-১০টি সার্ভিস হয়ে যেতে পারে।
Correlation ID ছাড়া এই সব লগ একসাথে জোড়া দেওয়া প্রায় অসম্ভব।

```
  Daraz-এ একটি অর্ডার প্লেসমেন্ট ফ্লো:

  ইউজার ──▶ API Gateway ──▶ Order Service ──▶ Payment Service ──▶ Inventory
              │                  │                   │                │
              │ correlationId:   │ correlationId:     │ correlationId: │ correlationId:
              │ "req-abc-123"    │ "req-abc-123"      │ "req-abc-123"  │ "req-abc-123"
              │                  │                   │                │
              ▼                  ▼                   ▼                ▼
           ┌──────┐          ┌──────┐            ┌──────┐         ┌──────┐
           │ LOG  │          │ LOG  │            │ LOG  │         │ LOG  │
           │ A    │          │ B    │            │ C    │         │ D    │
           └──────┘          └──────┘            └──────┘         └──────┘
              │                  │                   │                │
              └──────────────────┴───────────────────┴────────────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   Elasticsearch      │
                              │                     │
                              │  correlationId =    │
                              │  "req-abc-123"      │
                              │  দিয়ে সার্চ করলে   │
                              │  সব লগ একসাথে পাওয়া│
                              │  যাবে (A+B+C+D)    │
                              └─────────────────────┘
```

### Correlation ID প্রপাগেশন — বিস্তারিত

```
  ┌──────────────┐     HTTP Header                ┌──────────────┐
  │   Client     │ ──────────────────────────────▶ │  API Gateway │
  │  (Browser/   │   X-Correlation-ID:            │              │
  │   Mobile)    │   (gateway তৈরি করে            │  ID তৈরি:    │
  │              │    যদি না থাকে)                 │  "req-abc-123"│
  └──────────────┘                                 └──────┬───────┘
                                                          │
                    ┌─────────────────────────────────────┼───────────────┐
                    │                                     │               │
                    ▼                                     ▼               ▼
             ┌──────────────┐                     ┌──────────────┐ ┌───────────┐
             │ Order Service│                     │Payment Service│ │Notification│
             │              │                     │              │ │  Service  │
             │ Header থেকে │                     │ Header থেকে │ │           │
             │ ID পড়ে এবং  │                     │ ID পড়ে এবং  │ │ Queue msg │
             │ প্রতিটা লগে │                     │ প্রতিটা লগে │ │ থেকে ID  │
             │ যুক্ত করে   │                     │ যুক্ত করে   │ │ পড়ে      │
             └──────────────┘                     └──────────────┘ └───────────┘
```

### PHP — Correlation ID Middleware

```php
// Middleware: CorrelationIdMiddleware.php
class CorrelationIdMiddleware
{
    public function handle($request, Closure $next)
    {
        $correlationId = $request->header('X-Correlation-ID')
            ?: 'req-' . bin2hex(random_bytes(8));

        // পুরো request lifecycle-এ ব্যবহারের জন্য সংরক্ষণ
        $request->attributes->set('correlationId', $correlationId);

        // Monolog-এর সব লগে স্বয়ংক্রিয়ভাবে যুক্ত করতে processor ব্যবহার
        app('log')->getLogger()->pushProcessor(function ($record) use ($correlationId) {
            $record['extra']['correlationId'] = $correlationId;
            return $record;
        });

        $response = $next($request);

        // Response header-এও পাঠান (ডিবাগিং-এ সুবিধা)
        $response->headers->set('X-Correlation-ID', $correlationId);

        return $response;
    }
}
```

### JavaScript — Correlation ID Middleware (Express)

```javascript
const { v4: uuidv4 } = require('uuid');
const { AsyncLocalStorage } = require('async_hooks');

const asyncLocalStorage = new AsyncLocalStorage();

// Middleware: সব incoming request-এ correlation ID নিশ্চিত করে
function correlationMiddleware(req, res, next) {
    const correlationId = req.headers['x-correlation-id'] || `req-${uuidv4()}`;

    // Response header-এ সেট করুন
    res.setHeader('X-Correlation-ID', correlationId);

    // AsyncLocalStorage-এ সংরক্ষণ — যেকোনো async কোড থেকে অ্যাক্সেস করা যাবে
    asyncLocalStorage.run({ correlationId }, () => {
        next();
    });
}

// যেকোনো জায়গা থেকে correlation ID পেতে
function getCorrelationId() {
    const store = asyncLocalStorage.getStore();
    return store?.correlationId || 'unknown';
}

module.exports = { correlationMiddleware, getCorrelationId };
```

---

## 📊 সেন্ট্রালাইজড লগিং — ELK Stack

ELK Stack হলো **Elasticsearch + Logstash + Kibana** — তিনটি ওপেনসোর্স টুলের সমন্বয়
যা সেন্ট্রালাইজড লগিং-এর ইন্ডাস্ট্রি স্ট্যান্ডার্ড।

ধরুন Grameenphone-র ৫০টি মাইক্রোসার্ভিস আছে। প্রতিটি আলাদা আলাদা সার্ভারে চলছে।
কোনো সমস্যা হলে ৫০টি সার্ভারে SSH করে লগ দেখা অসম্ভব। ELK Stack সব লগ **এক জায়গায়**
নিয়ে আসে।

### ELK Stack আর্কিটেকচার ডায়াগ্রাম

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        ELK Stack Architecture                          │
  └─────────────────────────────────────────────────────────────────────────┘

  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐
  │  Order Service│  │Payment Service│  │  User Service  │  │ Rider App   │
  │  (PHP/Laravel)│  │  (Node.js)    │  │  (PHP)         │  │ (Node.js)   │
  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘  └──────┬──────┘
          │                  │                   │                 │
          │   JSON লগ        │   JSON লগ         │  JSON লগ       │  JSON লগ
          │   ফাইলে লেখে    │   stdout-এ লেখে  │  ফাইলে লেখে   │  stdout
          ▼                  ▼                   ▼                 ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │                        Filebeat / Fluentd                             │
  │        (Log Shipper — প্রতিটি সার্ভারে ইনস্টল থাকে)                  │
  │        লগ ফাইল পড়ে এবং Logstash-এ পাঠায়                             │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │                          Logstash                                     │
  │                                                                       │
  │   ┌──────────┐    ┌──────────────┐    ┌───────────┐                  │
  │   │  INPUT    │───▶│   FILTER     │───▶│  OUTPUT   │                  │
  │   │          │    │              │    │           │                  │
  │   │ Beats    │    │ Parse JSON   │    │ Elastic-  │                  │
  │   │ TCP/UDP  │    │ Add fields   │    │ search    │                  │
  │   │ Kafka    │    │ Remove PII   │    │ S3        │                  │
  │   │ HTTP     │    │ GeoIP lookup │    │ Kafka     │                  │
  │   └──────────┘    └──────────────┘    └───────────┘                  │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │                       Elasticsearch                                   │
  │                                                                       │
  │   ┌─────────────────────────────────────────────────────────────┐    │
  │   │  Index: logs-2024.01.15                                     │    │
  │   │                                                             │    │
  │   │  { "timestamp": "...", "level": "ERROR",                    │    │
  │   │    "service": "payment-service",                            │    │
  │   │    "message": "bKash timeout",                              │    │
  │   │    "correlationId": "req-abc-123" }                         │    │
  │   │                                                             │    │
  │   │  সার্চ, ফিল্টার, অ্যাগ্রিগেশন সব সম্ভব                    │    │
  │   └─────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │                          Kibana                                       │
  │                                                                       │
  │   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
  │   │   ড্যাশবোর্ড     │  │   লগ এক্সপ্লোরার  │  │   অ্যালার্ট      │  │
  │   │                  │  │                  │  │                  │  │
  │   │  Error rate      │  │  correlationId   │  │  ERROR > 100/min │  │
  │   │  গ্রাফ            │  │  দিয়ে সার্চ      │  │  হলে Slack-এ     │  │
  │   │                  │  │                  │  │  নোটিফিকেশন     │  │
  │   └──────────────────┘  └──────────────────┘  └──────────────────┘  │
  └───────────────────────────────────────────────────────────────────────┘
```

### ELK Stack — Docker Compose

```yaml
# docker-compose.elk.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log/app:/var/log/app:ro
    depends_on:
      - logstash

volumes:
  es-data:
```

### Logstash Pipeline কনফিগারেশন

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
  # JSON পার্স করুন
  json {
    source => "message"
    target => "log"
  }

  # Timestamp পার্স
  date {
    match => ["[log][timestamp]", "ISO8601"]
    target => "@timestamp"
  }

  # সংবেদনশীল ডেটা মাস্ক করুন
  mutate {
    gsub => [
      "[log][phone]", "(\d{4})\d{3}(\d{4})", "\1***\2",
      "[log][email]", "(.{2}).*(@.*)", "\1***\2"
    ]
  }

  # GeoIP থেকে লোকেশন নির্ধারণ
  if [log][ip] {
    geoip {
      source => "[log][ip]"
      target => "[log][geo]"
    }
  }

  # সার্ভিস অনুযায়ী ট্যাগ
  if [log][service] == "payment-service" {
    mutate { add_tag => ["payment", "critical"] }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

## 🏠 Log Aggregation Patterns

### প্যাটার্ন ১: Direct Shipping (সরাসরি পাঠানো)

```
  সহজ সেটআপ, ছোট অ্যাপ্লিকেশনের জন্য ভালো

  ┌───────────┐          ┌──────────────┐
  │    App     │ ──JSON──▶│ Elasticsearch│
  │  (Logger)  │  HTTP    │              │
  └───────────┘          └──────────────┘

  ⚠️ সমস্যা: App crash হলে লগ হারিয়ে যেতে পারে
```

### প্যাটার্ন ২: Agent-based (Filebeat/Fluentd)

```
  সবচেয়ে জনপ্রিয় — প্রোডাকশন রেডি

  ┌───────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │    App     │──▶│ Log File │──▶│ Filebeat │──▶│  Logstash    │
  │  (Logger)  │   │          │   │ (Agent)  │   │              │
  └───────────┘    └──────────┘   └──────────┘   └──────┬───────┘
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │Elasticsearch │
                                                  └──────────────┘

  ✅ App crash হলেও ফাইলে লগ থাকে
  ✅ Filebeat efficient — কম রিসোর্স খরচ
```

### প্যাটার্ন ৩: Queue-based (Kafka/Redis)

```
  হাই-ট্রাফিক সিস্টেমের জন্য (যেমন bKash, Grameenphone)

  ┌───────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │    App     │──▶│  Kafka   │──▶│ Logstash │──▶│ Elasticsearch│
  │  (Logger)  │   │  Queue   │   │(Consumer)│   │              │
  └───────────┘    └──────────┘   └──────────┘   └──────────────┘

  ✅ Burst traffic সামলাতে পারে
  ✅ কোনো লগ হারায় না (Kafka durable)
  ✅ Multiple consumer সম্ভব (archiving, alerting, analytics)
```

---

## 📦 PHP — Monolog সেটআপ

Monolog হলো PHP-র সবচেয়ে জনপ্রিয় লগিং লাইব্রেরি। Laravel, Symfony সহ
প্রায় সব ফ্রেমওয়ার্ক এটি ব্যবহার করে।

### বেসিক সেটআপ

```php
<?php
require 'vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Formatter\JsonFormatter;
use Monolog\Processor\UidProcessor;
use Monolog\Processor\WebProcessor;
use Monolog\Processor\MemoryUsageProcessor;
use Monolog\Processor\IntrospectionProcessor;

// Logger তৈরি করুন
$logger = new Logger('bkash-payment-service');

// JSON Formatter — স্ট্রাকচার্ড লগের জন্য
$jsonFormatter = new JsonFormatter();

// Handler ১: ফাইলে লগ (দৈনিক rotate হবে, ১৪ দিন রাখবে)
$fileHandler = new RotatingFileHandler(
    __DIR__ . '/logs/payment.log',
    14,                    // ১৪ দিন রাখবে
    Logger::INFO           // INFO এবং তার উপরে
);
$fileHandler->setFormatter($jsonFormatter);

// Handler ২: stderr-এ ERROR এবং তার উপরে
$stderrHandler = new StreamHandler('php://stderr', Logger::ERROR);
$stderrHandler->setFormatter($jsonFormatter);

// Processor: প্রতিটি লগে স্বয়ংক্রিয়ভাবে তথ্য যুক্ত করে
$logger->pushProcessor(new UidProcessor());           // unique request ID
$logger->pushProcessor(new WebProcessor());           // URL, IP, method
$logger->pushProcessor(new MemoryUsageProcessor());   // memory usage
$logger->pushProcessor(new IntrospectionProcessor()); // file, line, class

$logger->pushHandler($fileHandler);
$logger->pushHandler($stderrHandler);
```

### কাস্টম Processor — Correlation ID ও Service Info

```php
<?php
use Monolog\LogRecord;

// কাস্টম Processor: প্রতিটি লগে সার্ভিস তথ্য ও correlation ID যুক্ত করে
class ServiceContextProcessor
{
    private string $serviceName;
    private string $environment;

    public function __construct(string $serviceName, string $environment)
    {
        $this->serviceName = $serviceName;
        $this->environment = $environment;
    }

    public function __invoke(LogRecord $record): LogRecord
    {
        $record->extra['service']       = $this->serviceName;
        $record->extra['environment']   = $this->environment;
        $record->extra['hostname']      = gethostname();
        $record->extra['correlationId'] = $_SERVER['HTTP_X_CORRELATION_ID']
            ?? 'req-' . bin2hex(random_bytes(8));

        return $record;
    }
}

$logger->pushProcessor(new ServiceContextProcessor(
    'payment-service',
    getenv('APP_ENV') ?: 'production'
));
```

### কাস্টম Processor — PII মাস্কিং

```php
<?php
use Monolog\LogRecord;

// সংবেদনশীল তথ্য স্বয়ংক্রিয়ভাবে মাস্ক করে
class PiiMaskingProcessor
{
    private array $sensitiveKeys = [
        'password', 'pin', 'otp', 'secret',
        'token', 'authorization', 'credit_card',
        'nid', 'national_id',
    ];

    public function __invoke(LogRecord $record): LogRecord
    {
        $context = $record->context;
        $record = $record->with(context: $this->maskArray($context));
        return $record;
    }

    private function maskArray(array $data): array
    {
        foreach ($data as $key => $value) {
            if (is_array($value)) {
                $data[$key] = $this->maskArray($value);
            } elseif ($this->isSensitive($key)) {
                $data[$key] = '***MASKED***';
            } elseif ($this->isPhone($key) && is_string($value)) {
                $data[$key] = substr($value, 0, 4) . '***' . substr($value, -4);
            } elseif ($this->isEmail($key) && is_string($value)) {
                $parts = explode('@', $value);
                $data[$key] = substr($parts[0], 0, 2) . '***@' . ($parts[1] ?? '');
            }
        }
        return $data;
    }

    private function isSensitive(string $key): bool
    {
        return in_array(strtolower($key), $this->sensitiveKeys, true);
    }

    private function isPhone(string $key): bool
    {
        return in_array(strtolower($key), ['phone', 'mobile', 'msisdn'], true);
    }

    private function isEmail(string $key): bool
    {
        return strtolower($key) === 'email';
    }
}

$logger->pushProcessor(new PiiMaskingProcessor());
```

### Monolog ব্যবহার — সম্পূর্ণ উদাহরণ

```php
<?php
// bKash পেমেন্ট প্রসেসিং-এ Monolog ব্যবহার

class BkashPaymentService
{
    private Logger $logger;

    public function __construct(Logger $logger)
    {
        $this->logger = $logger;
    }

    public function processPayment(string $orderId, float $amount, string $phone): array
    {
        $this->logger->info('Payment processing started', [
            'orderId' => $orderId,
            'amount'  => $amount,
            'phone'   => $phone,    // PiiMaskingProcessor মাস্ক করবে
            'gateway' => 'bKash',
        ]);

        try {
            $startTime = microtime(true);

            $result = $this->callBkashApi($orderId, $amount, $phone);

            $duration = (microtime(true) - $startTime) * 1000;

            if ($duration > 2000) {
                $this->logger->warning('bKash API slow response', [
                    'orderId'     => $orderId,
                    'duration_ms' => round($duration),
                    'threshold'   => 2000,
                ]);
            }

            $this->logger->info('Payment completed successfully', [
                'orderId'        => $orderId,
                'transactionId'  => $result['txnId'],
                'amount'         => $amount,
                'duration_ms'    => round($duration),
            ]);

            return $result;

        } catch (\Exception $e) {
            $this->logger->error('Payment processing failed', [
                'orderId'   => $orderId,
                'amount'    => $amount,
                'error'     => $e->getMessage(),
                'errorCode' => $e->getCode(),
                'trace'     => $e->getTraceAsString(),
            ]);
            throw $e;
        }
    }
}
```

### Monolog আউটপুট উদাহরণ (JSON)

```json
{
  "message": "Payment completed successfully",
  "context": {
    "orderId": "ORD-2024-7890",
    "transactionId": "TXN-BK-456789",
    "amount": 1500,
    "duration_ms": 890
  },
  "level": 200,
  "level_name": "INFO",
  "channel": "bkash-payment-service",
  "datetime": "2024-01-15T10:30:45.123456+00:00",
  "extra": {
    "uid": "a1b2c3d4",
    "service": "payment-service",
    "environment": "production",
    "hostname": "prod-server-03",
    "correlationId": "req-abc-123",
    "memory_usage": "12 MB"
  }
}
```

---

## 🟢 JavaScript — Winston ও Pino

### Winston সেটআপ — সম্পূর্ণ উদাহরণ

```javascript
// logger.js — Winston কনফিগারেশন
const winston = require('winston');
const { combine, timestamp, json, errors, printf } = winston.format;

// কাস্টম ফরম্যাট: সার্ভিস তথ্য ও correlation ID যুক্ত
const serviceContext = winston.format((info) => {
    info.service = process.env.SERVICE_NAME || 'unknown-service';
    info.environment = process.env.NODE_ENV || 'development';
    info.hostname = require('os').hostname();
    return info;
});

// PII মাস্কিং ফরম্যাট
const maskPii = winston.format((info) => {
    const sensitiveKeys = ['password', 'pin', 'otp', 'secret', 'token'];

    for (const key of Object.keys(info)) {
        if (sensitiveKeys.includes(key.toLowerCase())) {
            info[key] = '***MASKED***';
        }
        if (['phone', 'mobile'].includes(key.toLowerCase()) && typeof info[key] === 'string') {
            const val = info[key];
            info[key] = val.slice(0, 4) + '***' + val.slice(-4);
        }
    }
    return info;
});

const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: combine(
        errors({ stack: true }),
        timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
        serviceContext(),
        maskPii(),
        json()
    ),
    defaultMeta: { service: 'pathao-ride-service' },
    transports: [
        // ফাইলে ERROR লগ
        new winston.transports.File({
            filename: 'logs/error.log',
            level: 'error',
            maxsize: 50 * 1024 * 1024,  // 50MB
            maxFiles: 14,
        }),

        // ফাইলে সব লগ
        new winston.transports.File({
            filename: 'logs/combined.log',
            maxsize: 100 * 1024 * 1024,  // 100MB
            maxFiles: 7,
        }),
    ],
});

// ডেভেলপমেন্টে console-এও দেখান
if (process.env.NODE_ENV !== 'production') {
    logger.add(new winston.transports.Console({
        format: combine(
            winston.format.colorize(),
            winston.format.simple()
        ),
    }));
}

module.exports = logger;
```

### Winston ব্যবহার — Pathao রাইড সার্ভিসে

```javascript
// rideService.js
const logger = require('./logger');
const { getCorrelationId } = require('./correlationMiddleware');

class RideService {
    async requestRide(userId, pickup, destination) {
        const correlationId = getCorrelationId();

        logger.info('Ride requested', {
            correlationId,
            userId,
            pickup: { lat: pickup.lat, lng: pickup.lng, area: pickup.area },
            destination: { lat: destination.lat, lng: destination.lng, area: destination.area },
        });

        try {
            const startTime = Date.now();

            // কাছের রাইডার খুঁজুন
            const nearbyRiders = await this.findNearbyRiders(pickup, 3);

            logger.info('Nearby riders found', {
                correlationId,
                riderCount: nearbyRiders.length,
                searchRadius_km: 3,
                duration_ms: Date.now() - startTime,
            });

            if (nearbyRiders.length === 0) {
                logger.warn('No riders available', {
                    correlationId,
                    userId,
                    area: pickup.area,
                    searchRadius_km: 3,
                });
                return { status: 'no_riders', retryAfter: 30 };
            }

            // রাইড ম্যাচিং
            const matchedRider = await this.matchRider(nearbyRiders, destination);

            logger.info('Ride matched successfully', {
                correlationId,
                userId,
                riderId: matchedRider.id,
                estimatedArrival_min: matchedRider.eta,
                estimatedFare: matchedRider.fare,
                duration_ms: Date.now() - startTime,
            });

            return { status: 'matched', rider: matchedRider };

        } catch (error) {
            logger.error('Ride request failed', {
                correlationId,
                userId,
                error: error.message,
                stack: error.stack,
            });
            throw error;
        }
    }
}
```

### Pino সেটআপ — হাই-পারফরম্যান্স লগিং

Pino হলো Winston-এর চেয়ে ৫-১০ গুণ **দ্রুত**। হাই-ট্রাফিক সার্ভিসের জন্য
(যেমন bKash API যেটা প্রতি সেকেন্ডে হাজার হাজার রিকোয়েস্ট সামলায়) Pino আদর্শ।

```javascript
// logger-pino.js
const pino = require('pino');

const logger = pino({
    level: process.env.LOG_LEVEL || 'info',

    // বেস তথ্য — প্রতিটি লগে থাকবে
    base: {
        service: process.env.SERVICE_NAME || 'bkash-txn-service',
        env: process.env.NODE_ENV || 'development',
        hostname: require('os').hostname(),
        pid: process.pid,
    },

    // Timestamp ISO format-এ
    timestamp: pino.stdTimeFunctions.isoTime,

    // Redaction — সংবেদনশীল ফিল্ড স্বয়ংক্রিয়ভাবে মাস্ক
    redact: {
        paths: [
            'password', 'pin', 'otp', 'secret',
            'token', 'authorization',
            'req.headers.authorization',
            'req.headers.cookie',
        ],
        censor: '***REDACTED***',
    },

    // Serializers — অবজেক্টকে লগ-ফ্রেন্ডলি ফরম্যাটে রূপান্তর
    serializers: {
        req: pino.stdSerializers.req,
        res: pino.stdSerializers.res,
        err: pino.stdSerializers.err,
    },

    // Transport — প্রোডাকশনে ফাইলে, ডেভে pretty print
    transport: process.env.NODE_ENV === 'production'
        ? {
            target: 'pino/file',
            options: { destination: './logs/app.log', mkdir: true },
        }
        : {
            target: 'pino-pretty',
            options: { colorize: true, translateTime: 'SYS:standard' },
        },
});

module.exports = logger;
```

### Pino ব্যবহার — bKash ট্রানজ্যাকশন সার্ভিসে

```javascript
// transactionService.js
const logger = require('./logger-pino');

async function processTransaction(txnData) {
    const child = logger.child({
        correlationId: txnData.correlationId,
        txnId: txnData.id,
    });

    child.info({ amount: txnData.amount, type: txnData.type }, 'Transaction started');

    const timer = Date.now();

    try {
        const result = await executePayment(txnData);

        child.info({
            duration_ms: Date.now() - timer,
            status: result.status,
        }, 'Transaction completed');

        return result;

    } catch (err) {
        child.error({
            err,
            duration_ms: Date.now() - timer,
            retryable: err.code !== 'INSUFFICIENT_BALANCE',
        }, 'Transaction failed');

        throw err;
    }
}
```

### Pino আউটপুট উদাহরণ

```json
{
  "level": 30,
  "time": "2024-01-15T10:30:45.123Z",
  "service": "bkash-txn-service",
  "env": "production",
  "hostname": "prod-node-07",
  "pid": 12345,
  "correlationId": "req-abc-123",
  "txnId": "TXN-2024-99001",
  "amount": 2500,
  "type": "send_money",
  "msg": "Transaction started"
}
```

### Winston vs Pino তুলনা

```
┌─────────────────┬────────────────────┬────────────────────┐
│   বৈশিষ্ট্য     │     Winston        │      Pino          │
├─────────────────┼────────────────────┼────────────────────┤
│  পারফরম্যান্স   │  ভালো              │  অনেক দ্রুত (5-10x)│
│  ফিচার          │  অনেক বেশি        │  মিনিমাল, ফোকাসড   │
│  কাস্টমাইজেশন   │  সহজ, flexible    │  সীমিত কিন্তু যথেষ্ট│
│  Transport      │  অনেক বিল্ট-ইন    │  pino-transport    │
│  Redaction      │  ম্যানুয়াল         │  বিল্ট-ইন          │
│  Child Logger   │  আছে              │  আরো efficient     │
│  কখন ব্যবহার   │  সাধারণ সার্ভিস    │  হাই-ট্রাফিক সার্ভিস│
│  উদাহরণ         │  Daraz backend    │  bKash core API    │
└─────────────────┴────────────────────┴────────────────────┘
```

---

## 🏠 বাস্তব উদাহরণ (Real-World Scenarios)

### উদাহরণ ১: bKash পেমেন্ট ট্রানজ্যাকশন লগিং

```
  ইউজার bKash দিয়ে Daraz-এ পেমেন্ট করছেন — লগিং ফ্লো:

  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
  │  Daraz   │───▶│ Order Service│───▶│Payment Service│───▶│ bKash API│
  │  App     │    │              │    │              │    │          │
  └──────────┘    └──────┬───────┘    └──────┬───────┘    └────┬─────┘
                         │                   │                 │
    LOG: "Order          │    LOG: "Payment   │   LOG: "bKash   │
    placed"              │    initiated"      │   callback      │
    INFO                 │    INFO            │   received"     │
    correlationId:       │    correlationId:  │   INFO          │
    "req-daraz-555"      │    "req-daraz-555" │   correlationId:│
                         ▼                   ▼   "req-daraz-555"│
                                                               ▼
```

```php
// bKash পেমেন্ট পুরো lifecycle লগিং
class BkashPaymentLogger
{
    private Logger $logger;

    // ধাপ ১: পেমেন্ট শুরু
    public function logPaymentInitiated(string $orderId, float $amount, string $userId): void
    {
        $this->logger->info('bKash payment initiated', [
            'orderId'  => $orderId,
            'amount'   => $amount,
            'userId'   => $userId,
            'gateway'  => 'bKash',
            'step'     => 'INITIATED',
        ]);
    }

    // ধাপ ২: bKash API কল
    public function logApiCall(string $orderId, string $endpoint, float $duration): void
    {
        $level = $duration > 3000 ? 'warning' : 'info';
        $this->logger->log($level, 'bKash API call completed', [
            'orderId'     => $orderId,
            'endpoint'    => $endpoint,
            'duration_ms' => $duration,
            'step'        => 'API_CALL',
        ]);
    }

    // ধাপ ৩: bKash callback
    public function logCallback(string $orderId, string $status, string $txnId): void
    {
        $level = $status === 'success' ? 'info' : 'error';
        $this->logger->log($level, 'bKash callback received', [
            'orderId'       => $orderId,
            'bkashTxnId'    => $txnId,
            'paymentStatus' => $status,
            'step'          => 'CALLBACK',
        ]);
    }

    // ধাপ ৪: পেমেন্ট সম্পন্ন / ব্যর্থ
    public function logPaymentResult(string $orderId, bool $success, array $meta): void
    {
        if ($success) {
            $this->logger->info('bKash payment successful', array_merge(
                ['orderId' => $orderId, 'step' => 'COMPLETED'],
                $meta
            ));
        } else {
            $this->logger->error('bKash payment failed', array_merge(
                ['orderId' => $orderId, 'step' => 'FAILED'],
                $meta
            ));
        }
    }
}
```

### উদাহরণ ২: Pathao রাইড ট্র্যাকিং লগ

```javascript
// Pathao রাইড-এর প্রতিটি স্টেপে লগিং
const logger = require('./logger');

class RideTracker {
    logRideRequested(rideId, userId, pickup, destination) {
        logger.info('Ride requested', {
            rideId, userId,
            pickup: `${pickup.area}, ${pickup.city}`,
            destination: `${destination.area}, ${destination.city}`,
            step: 'REQUESTED',
        });
    }

    logRiderSearching(rideId, searchRadius, nearbyCount) {
        logger.info('Searching for riders', {
            rideId, searchRadius,
            nearbyRiders: nearbyCount,
            step: 'SEARCHING',
        });
    }

    logRiderMatched(rideId, riderId, eta) {
        logger.info('Rider matched', {
            rideId, riderId,
            estimatedArrival_min: eta,
            step: 'MATCHED',
        });
    }

    logRideStarted(rideId, riderId) {
        logger.info('Ride started', {
            rideId, riderId,
            step: 'IN_PROGRESS',
            startedAt: new Date().toISOString(),
        });
    }

    logRideCompleted(rideId, fare, distance, duration) {
        logger.info('Ride completed', {
            rideId,
            fare,
            distance_km: distance,
            duration_min: duration,
            step: 'COMPLETED',
        });
    }

    logRideCancelled(rideId, cancelledBy, reason) {
        logger.warn('Ride cancelled', {
            rideId, cancelledBy, reason,
            step: 'CANCELLED',
        });
    }
}
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন স্ট্রাকচার্ড লগিং ব্যবহার করবেন

```
► প্রোডাকশন সিস্টেম (সবসময়)
► মাইক্রোসার্ভিস আর্কিটেকচার
► যেকোনো সিস্টেম যেখানে সেন্ট্রালাইজড লগিং আছে
► যেখানে লগ সার্চ ও অ্যানালাইসিস দরকার
► যেখানে অ্যালার্ট ও মনিটরিং সেটআপ করা হবে
```

### ❌ কখন ব্যবহার করবেন না (সরলতা ভালো)

```
► ছোট স্ক্রিপ্ট বা one-off টুল (echo/print যথেষ্ট)
► লোকাল ডেভেলপমেন্টে ডিবাগিং (var_dump / console.log ঠিক আছে)
► প্রোটোটাইপিং পর্যায়ে (পরে যুক্ত করুন)
```

### ✅ কখন ELK Stack ব্যবহার করবেন

```
► ৩+ মাইক্রোসার্ভিস আছে
► ডিস্ট্রিবিউটেড সিস্টেম (একাধিক সার্ভার)
► লগ সার্চ ও ভিজুয়ালাইজেশন দরকার
► দৈনিক ১ GB+ লগ তৈরি হয়
► অন-কল টিম আছে যারা প্রোডাকশন ইস্যু ডিবাগ করে
```

### ✅ কখন Correlation ID ব্যবহার করবেন

```
► সবসময়! যেকোনো সার্ভিস যেটা HTTP request হ্যান্ডেল করে
► মাইক্রোসার্ভিসে তো আবশ্যক
► মনোলিথেও উপকারী (async job ট্রেসিং-এ)
```

---

## 📊 JSON স্ট্রাকচার্ড লগ ফরম্যাট — স্ট্যান্ডার্ড স্কিমা

একটি প্রতিষ্ঠানের সব সার্ভিসে **একই লগ ফরম্যাট** ব্যবহার করা উচিত।
এতে Elasticsearch-এ সার্চ ও ড্যাশবোর্ড বানানো সহজ হয়।

```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "message": "Human-readable description of what happened",

  "service": {
    "name": "payment-service",
    "version": "2.3.1",
    "environment": "production",
    "hostname": "prod-server-03",
    "instanceId": "i-abc123"
  },

  "request": {
    "correlationId": "req-abc-123",
    "method": "POST",
    "path": "/api/v1/payments",
    "userAgent": "DarazApp/3.2.1",
    "ip": "103.48.16.xxx"
  },

  "context": {
    "userId": 12345,
    "orderId": "ORD-2024-7890",
    "amount": 1500.00,
    "currency": "BDT"
  },

  "performance": {
    "duration_ms": 890,
    "dbQueries": 3,
    "cacheHit": true
  },

  "error": {
    "type": "GatewayTimeoutException",
    "message": "bKash API did not respond within 5s",
    "code": "GATEWAY_TIMEOUT",
    "stack": "at PaymentService.process (payment.js:45)..."
  }
}
```

---

## 🔗 সারসংক্ষেপ

```
  ┌──────────────────────────────────────────────────────────────┐
  │              লগিং — মূল নীতিমালা                             │
  ├──────────────────────────────────────────────────────────────┤
  │                                                              │
  │  ১. সবসময় স্ট্রাকচার্ড (JSON) লগিং ব্যবহার করুন           │
  │  ২. সঠিক লগ লেভেল ব্যবহার করুন                              │
  │  ৩. প্রতিটি লগে পর্যাপ্ত context দিন                        │
  │  ৪. Correlation ID সবসময় ব্যবহার করুন                       │
  │  ৫. সংবেদনশীল ডেটা কখনো লগ করবেন না                         │
  │  ৬. সেন্ট্রালাইজড লগিং সেটআপ করুন (ELK/Loki)              │
  │  ৭. অ্যালার্ট সেটআপ করুন (ERROR spike, FATAL events)       │
  │  ৮. লগ rotation ও retention policy রাখুন                    │
  │  ৯. পারফরম্যান্স মেট্রিক (duration_ms) লগ করুন              │
  │  ১০. প্রোডাকশনে DEBUG বন্ধ রাখুন                            │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘

  মনে রাখুন: ভালো লগিং = দ্রুত ডিবাগিং = কম ডাউনটাইম = সুখী ইউজার 🎯
```
