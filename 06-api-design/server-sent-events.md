# 📡 Server-Sent Events (SSE) — One-way Real-time Streaming

## 📖 সংজ্ঞা ও মূল ধারণা

**Server-Sent Events (SSE)** হলো একটি HTML5 standard যেখানে server থেকে client-এ একমুখী (unidirectional) real-time data stream পাঠানো হয় একটি single long-lived HTTP connection-এ। MIME type হলো `text/event-stream`। ব্রাউজারে built-in `EventSource` API দিয়ে consume করা যায়।

**মূল বৈশিষ্ট্য:**
- HTTP/1.1 ও HTTP/2 এর উপর কাজ করে (no special protocol)
- Plain text format, debug সহজ
- Auto-reconnect, `Last-Event-ID` দিয়ে resume
- Server → Client only (unidirectional)
- Browser native support (no library needed)

```
SSE Connection Lifecycle:

Client                                Server
  │                                     │
  │ ── GET /events HTTP/1.1 ──────────► │
  │    Accept: text/event-stream        │
  │                                     │
  │ ◄── 200 OK ────────────────────────│
  │    Content-Type: text/event-stream  │
  │    Cache-Control: no-cache          │
  │    Connection: keep-alive           │
  │                                     │
  │ ◄── data: {"order":"out_for_dlv"} │
  │     (event ১)                       │
  │                                     │
  │ ◄── data: {"order":"delivered"}   │
  │     (event ২)                       │
  │                                     │
  │ (connection স্থায়ীভাবে open)         │
  │                                     │
  │ X── connection drop ────────────── │
  │ ── auto reconnect with             │
  │    Last-Event-ID: 42 ─────────────► │
  │                                     │
```

---

## 🎯 SSE বনাম WebSocket বনাম Long Polling বনাম HTTP/2 Push

| বৈশিষ্ট্য | SSE | WebSocket | Long Polling | HTTP/2 Push (deprecated) |
|----------|-----|-----------|--------------|--------------------------|
| Direction | Server → Client | Bi-directional | Server → Client | Server → Client (resource) |
| Protocol | HTTP/1.1+ | ws://, wss:// (upgraded HTTP) | HTTP | HTTP/2 |
| Format | text/event-stream | binary or text frames | JSON over HTTP | Any |
| Auto-reconnect | ✅ Built-in | ❌ Manual | N/A | N/A |
| Last-Event-ID | ✅ Built-in | ❌ Manual | ❌ Manual | N/A |
| Browser API | EventSource | WebSocket | fetch/XHR | (deprecated) |
| Proxy/CDN friendly | ✅ (HTTP) | ⚠️ Needs WS support | ✅ | ✅ |
| Firewall | ✅ Port 80/443 | ⚠️ Sometimes blocked | ✅ | ✅ |
| Scalability | High (one-way) | Medium (stateful) | Low (high overhead) | N/A |
| Overhead per msg | ~10 bytes header | ~2 bytes frame | Full HTTP request | Full HTTP/2 frame |
| Use case | Notifications, ticker, log stream | Chat, gaming, collab | Legacy fallback | Server-pushing assets |
| Complexity | কম | বেশি | মাঝারি | High |

> **Decision rule:** যদি data flow শুধু Server → Client হয় (notification, live tracking, stock ticker, AI streaming), SSE-ই best। Bi-directional চাইলে WebSocket।

---

## 📜 Event Stream Format (Spec)

```
event: order_status        ← optional event name (default: "message")
id: 12345                  ← event ID (Last-Event-ID তে use)
retry: 3000                ← reconnect delay (ms)
data: {"orderId":"O-001",  ← actual payload (multi-line allowed)
data: "status":"picked"}

                           ← double newline = end of event
event: heartbeat
data: ping

```

**Rules:**
- লাইন `field: value` format-এ
- `data:` multiple line হতে পারে (joined with `\n`)
- Double `\n\n` = event boundary
- Comment line `:` দিয়ে শুরু (heartbeat-এ ব্যবহার)
- UTF-8 encoded

---

## 🔬 Deep Dive — Reconnection ও Last-Event-ID

```
Browser auto-reconnect flow:

1. Connection drop detect হলে ব্রাউজার retry করে
2. Default retry = 3 seconds (server `retry:` field দিয়ে override করা যায়)
3. শেষ পাওয়া event-এর `id` ব্রাউজার মনে রাখে
4. Reconnect-এ header পাঠায়:  Last-Event-ID: 12345
5. Server এই ID থেকে পরের event পাঠাবে (server-এর responsibility)
```

**Server-এ implement করতে হবে:**
- Event ID monotonically increasing রাখা
- Recent events buffer/persist করা (Redis stream, Kafka, in-memory ring buffer)
- `Last-Event-ID` header দেখে সেই point থেকে replay

---

## 💻 Node.js (Express) SSE Server — Pathao Foods Order Tracking

```javascript
// pathao-foods/sse-server.js
import express from 'express';
import { createClient } from 'redis';

const app = express();
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

// In-memory client registry (production-এ Redis Pub/Sub)
const clients = new Map(); // orderId -> Set<{res, lastEventId}>

function sseHeaders(res) {
  res.writeHead(200, {
    'Content-Type'      : 'text/event-stream',
    'Cache-Control'     : 'no-cache, no-transform',
    'Connection'        : 'keep-alive',
    'X-Accel-Buffering' : 'no',           // 🔥 Nginx buffering disable
    'Access-Control-Allow-Origin': '*',
  });
  res.flushHeaders();
}

// 📍 Order tracking SSE endpoint
app.get('/sse/orders/:orderId/track', async (req, res) => {
  const { orderId } = req.params;
  const lastEventId = req.headers['last-event-id'];
  const userId = req.user?.id; // assume auth middleware

  sseHeaders(res);

  // Initial state
  const order = await redis.hGetAll(`order:${orderId}`);
  if (!order || order.userId !== userId) {
    res.write(`event: error\ndata: ${JSON.stringify({ msg: 'Forbidden' })}\n\n`);
    return res.end();
  }

  // Tell client to retry after 3s if disconnected
  res.write('retry: 3000\n\n');

  // Replay missed events (if Last-Event-ID provided)
  if (lastEventId) {
    const missed = await redis.xRange(`order:${orderId}:events`, `(${lastEventId}`, '+');
    for (const ev of missed) {
      res.write(formatEvent(ev.id, ev.message.type, ev.message.payload));
    }
  } else {
    // Send current snapshot
    res.write(formatEvent(order.lastEventId || '0', 'snapshot', order));
  }

  // Register client
  if (!clients.has(orderId)) clients.set(orderId, new Set());
  const client = { res, userId };
  clients.get(orderId).add(client);

  // 💓 Heartbeat every 25s (keep-alive, prevent proxy timeout)
  const heartbeat = setInterval(() => {
    res.write(`: heartbeat ${Date.now()}\n\n`);
  }, 25_000);

  // Cleanup on disconnect
  req.on('close', () => {
    clearInterval(heartbeat);
    clients.get(orderId)?.delete(client);
    console.log(`Client disconnected from order ${orderId}`);
  });
});

function formatEvent(id, eventName, payload) {
  let out = '';
  if (id) out += `id: ${id}\n`;
  if (eventName) out += `event: ${eventName}\n`;
  out += `data: ${JSON.stringify(payload)}\n\n`;
  return out;
}

// Subscribe to Redis Pub/Sub for order events (from rider app, kitchen, etc.)
const subscriber = redis.duplicate();
await subscriber.connect();
await subscriber.pSubscribe('order.*.update', (message, channel) => {
  const orderId = channel.split('.')[1];
  const event = JSON.parse(message);

  // Persist to stream (for replay)
  redis.xAdd(`order:${orderId}:events`, '*', {
    type: event.type,
    payload: JSON.stringify(event.data),
  });

  // Push to all SSE clients of this order
  const orderClients = clients.get(orderId);
  if (!orderClients) return;
  for (const c of orderClients) {
    c.res.write(formatEvent(Date.now(), event.type, event.data));
  }
});

app.listen(4000, () => console.log('SSE server on :4000'));
```

### Browser EventSource (Frontend)

```javascript
// Pathao Foods customer app
function trackOrder(orderId) {
  const source = new EventSource(`/sse/orders/${orderId}/track`, {
    withCredentials: true, // send cookies for auth
  });

  source.addEventListener('snapshot', (e) => {
    const order = JSON.parse(e.data);
    updateUI(order); // initial state
  });

  source.addEventListener('status_change', (e) => {
    const data = JSON.parse(e.data);
    showNotification(`আপনার অর্ডার এখন: ${translateStatus(data.status)}`);
    updateMap(data.riderLocation);
  });

  source.addEventListener('rider_location', (e) => {
    const { lat, lng, eta } = JSON.parse(e.data);
    moveRiderMarker(lat, lng);
    document.querySelector('#eta').textContent = `${eta} মিনিট`;
  });

  source.addEventListener('error', (e) => {
    if (source.readyState === EventSource.CLOSED) {
      console.log('Connection closed by server');
    } else {
      console.log('Reconnecting...'); // browser auto-reconnects
    }
  });

  // Manual close (যেমন order delivered হলে)
  return () => source.close();
}

function translateStatus(s) {
  return {
    'placed'        : 'অর্ডার গৃহীত',
    'preparing'     : 'রান্না হচ্ছে',
    'picked'        : 'রাইডার তুলেছে',
    'on_the_way'    : 'পথে',
    'delivered'     : 'ডেলিভারি সম্পন্ন',
  }[s] || s;
}
```

---

## 🐘 PHP SSE — Raw + Laravel Examples

### Raw PHP SSE (no framework)

```php
<?php
// public/sse/price-ticker.php
header('Content-Type: text/event-stream');
header('Cache-Control: no-cache, no-transform');
header('Connection: keep-alive');
header('X-Accel-Buffering: no'); // Nginx

// PHP buffer disable (critical!)
@ob_end_flush();
@ob_implicit_flush(true);
ignore_user_abort(false); // detect client disconnect
set_time_limit(0);

$lastEventId = $_SERVER['HTTP_LAST_EVENT_ID'] ?? 0;

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

echo "retry: 3000\n\n";

while (true) {
    if (connection_aborted()) {
        break;
    }

    // Fetch latest stock prices (DSE — Dhaka Stock Exchange)
    $prices = $redis->hGetAll('dse:prices');
    $eventId = (int) $redis->get('dse:price_version');

    if ($eventId > $lastEventId) {
        echo "id: {$eventId}\n";
        echo "event: price_update\n";
        echo "data: " . json_encode($prices, JSON_UNESCAPED_UNICODE) . "\n\n";
        $lastEventId = $eventId;
        @ob_flush();
        @flush();
    } else {
        // heartbeat (comment line)
        echo ": heartbeat " . time() . "\n\n";
        @ob_flush();
        @flush();
    }

    sleep(1);
}
```

### Laravel SSE (Notifications)

```php
// app/Http/Controllers/NotificationStreamController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Illuminate\Support\Facades\Redis;

class NotificationStreamController extends Controller
{
    public function stream(Request $request): StreamedResponse
    {
        $userId      = $request->user()->id;
        $lastEventId = (int) ($request->header('Last-Event-ID') ?? 0);

        $response = new StreamedResponse(function () use ($userId, $lastEventId) {
            // Replay missed events from DB
            $missed = \App\Models\Notification::where('user_id', $userId)
                        ->where('id', '>', $lastEventId)
                        ->orderBy('id')
                        ->limit(50)
                        ->get();

            foreach ($missed as $n) {
                echo "id: {$n->id}\n";
                echo "event: notification\n";
                echo 'data: ' . json_encode($n->only(['id', 'title', 'body', 'type'])) . "\n\n";
                $lastEventId = $n->id;
            }

            @ob_flush(); @flush();

            // Subscribe to Redis Pub/Sub
            Redis::subscribe(["user.{$userId}.notifications"], function ($message) use (&$lastEventId) {
                if (connection_aborted()) {
                    exit; // important to break the subscribe loop
                }

                $data = json_decode($message, true);
                $lastEventId = $data['id'];

                echo "id: {$data['id']}\n";
                echo "event: notification\n";
                echo 'data: ' . json_encode($data, JSON_UNESCAPED_UNICODE) . "\n\n";
                @ob_flush(); @flush();
            });
        });

        $response->headers->set('Content-Type', 'text/event-stream');
        $response->headers->set('Cache-Control', 'no-cache, no-transform');
        $response->headers->set('X-Accel-Buffering', 'no');
        $response->headers->set('Connection', 'keep-alive');

        return $response;
    }
}
```

### Symfony Mercure (managed SSE hub)

Mercure হলো Symfony-এর তৈরি SSE-based protocol + reverse proxy (`mercure` binary)। Single hub dynamically multiplex করে publishers থেকে subscribers-এ। বেশি client (10K+) হলে নিজের SSE server না লিখে Mercure ব্যবহার করুন।

```php
// publisher (publish থেকে hub-এ)
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

public function placeOrder(HubInterface $hub) {
    // ...order create logic
    $update = new Update(
        "https://daraz.com.bd/orders/{$order->id}",
        json_encode(['status' => 'placed', 'orderId' => $order->id]),
        private: true,                 // JWT-protected topic
    );
    $hub->publish($update);
}
```

```javascript
// frontend (browser)
const url = new URL('https://hub.daraz.com.bd/.well-known/mercure');
url.searchParams.append('topic', `https://daraz.com.bd/orders/${orderId}`);
const es = new EventSource(url, { withCredentials: true });
```

---

## ⚠️ Proxy/Nginx Buffering Pitfall

SSE-এর সবচেয়ে famous bug — message client-এ পৌঁছায় না, কারণ Nginx response buffer করে।

### Nginx config (proper SSE):

```nginx
location /sse/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;

    # 🔥 SSE-এর জন্য critical
    proxy_set_header Connection '';
    proxy_buffering off;              # buffer disable
    proxy_cache off;
    proxy_read_timeout 86400s;        # 24h, default 60s
    proxy_send_timeout 86400s;
    chunked_transfer_encoding on;

    # Pass Last-Event-ID
    proxy_set_header Last-Event-ID $http_last_event_id;
}
```

### Server-এ `X-Accel-Buffering: no` header
Nginx এই header দেখলে নির্দিষ্ট response-এ buffering off করে — config ছুঁতে না পারলেও কাজ করে।

### CDN
- **CloudFlare:** SSE support আছে কিন্তু Free plan-এ proxy 100s timeout। Enterprise plan দরকার long-lived এর জন্য।
- **AWS CloudFront:** SSE works, কিন্তু `Origin Response Timeout` 60s default → বাড়ান।

### Other gotchas
- HTTP/1.1-এ একই domain-এ ব্রাউজার max 6 SSE connection (per origin) — multi-tab এ blocking। HTTP/2 দিয়ে এটা multiplexed (100+ streams)।
- Compression (`Content-Encoding: gzip`) চালু থাকলে buffering হবে — disable করুন।
- PHP-FPM `output_buffering` setting check করুন (`Off` করুন)।

---

## ✅ কখন SSE ব্যবহার করবেন

- **One-way real-time updates** — notification, live order tracking, stock price ticker, log streaming, AI token streaming (ChatGPT-style)
- **Browser native support চান**, library ছাড়া
- **HTTP infrastructure (proxy, auth, CORS) reuse** করতে চান
- **Auto-reconnect** চান built-in
- Connection count মাঝারি (1K-50K per server, tuning দরকার)

## ❌ কখন SSE এড়াবেন

- **Bi-directional communication** দরকার (chat, multiplayer game) → WebSocket
- **Binary data** (audio, video) → WebSocket / WebRTC
- **Mobile native app** — NSURLSession/OkHttp-এ SSE স্বাভাবিকভাবে নেই; library দরকার (LaunchDarkly EventSource, ReactiveX)
- IE 11 support দরকার (EventSource নাই; polyfill দরকার)
- High-frequency tick (1000+ events/sec) — overhead বেশি, WebSocket binary frame ভালো

---

## 🛡️ Production Concerns

### 1. Connection limits
- Linux default: `ulimit -n 1024` → বাড়ান (`65536`)
- Node.js: একটা process ~10K SSE connection handle করতে পারে; বেশি হলে cluster/horizontal scaling
- PHP-FPM: প্রতিটা connection-এ একটা PHP process — expensive। ReactPHP/Swoole/RoadRunner ভালো।

### 2. Authentication
- `EventSource` constructor custom header support করে না (cookies শুধু)
- `withCredentials: true` দিলে cookies/auth header propagate
- Token auth দরকার হলে query param বা polyfill (`event-source-polyfill`) ব্যবহার

### 3. Scaling — Pub/Sub backbone
Single-server SSE simple, কিন্তু multi-instance হলে Redis Pub/Sub / Kafka / NATS দরকার যেন এক instance-এর publish সব instance-এর subscriber-এ যায়।

```
   Web Server 1 (SSE)  ◄── Redis Pub/Sub ──► Web Server 2 (SSE)
        ▲                                          ▲
    Client A                                    Client B
   (Order #100 listener)                  (Order #100 listener)
```

### 4. Backpressure
Slow client → write buffer পূর্ণ হয়ে memory leak। Node.js-এ `res.write()` return value check করুন; `false` হলে drain event-এর জন্য অপেক্ষা।

```javascript
function safeWrite(res, chunk) {
  if (!res.write(chunk)) {
    // backpressure — wait for drain or drop
    return new Promise(resolve => res.once('drain', resolve));
  }
}
```

### 5. Heartbeat
Idle connection 30-60s পর proxy/firewall close করে। প্রতি 15-25s comment line `:` পাঠান।

---

## 🇧🇩 বাংলাদেশ Real-world Use Cases

- **Pathao Foods:** Live order tracking — placed → confirmed → preparing → picked → on the way → delivered। Customer-এর map-এ rider location SSE দিয়ে stream।
- **Foodpanda BD:** Restaurant dashboard-এ নতুন order এলে instant pop-up — SSE notification stream।
- **DSE (Dhaka Stock Exchange) তৃতীয় পক্ষ data provider:** Stock price ticker SSE দিয়ে publish।
- **bKash Merchant App:** Settlement update, dispute notification।
- **Daraz seller center:** নতুন order এলে browser tab-এ live count update।
- **Chaldal:** Delivery status update — "আপনার rider 5 মিনিটের মধ্যে আসছে"।
- **BRAC Bank Astha App (web):** Transaction notification stream।
- **e-Cab/Uber-style ride apps:** Driver location, fare estimate update।
- **NewsBangla24/Prothom Alo live blog:** Live election result, খেলার scoreboard।
- **AI chatbot (Sheba.xyz support):** Token-by-token streaming response।

---

## ⚠️ সাধারণ Pitfalls

1. **Buffering bug** — Nginx/PHP-FPM buffer; `X-Accel-Buffering: no` ও `proxy_buffering off` mandatory।
2. **No heartbeat** — proxy 60s পর kill করে; client reconnect storm।
3. **`Last-Event-ID` ignore** — server replay implement না করলে disconnect-এ data হারায়।
4. **Memory leak** — disconnected client cleanup না করলে clients map বাড়তে থাকে।
5. **Browser 6-connection limit (HTTP/1.1)** — same origin-এ ৬টা tab খুললে ৭ম tab-এর SSE pending; HTTP/2 use করুন।
6. **Sending huge payload** — SSE per-event-এ ছোট data; বড় payload হলে এটা ভুল choice।
7. **CORS** — `Access-Control-Allow-Origin` ও `Access-Control-Allow-Credentials` ঠিকঠাক না থাকলে browser block করে।
8. **PHP `set_time_limit(0)` ভুলে যাওয়া** — PHP default 30s execution timeout-এ connection মরে।
9. **Compression on** — gzip buffer; SSE-তে compression disable।

---

## 📊 SSE Capacity Benchmark (illustrative)

| Setup | Concurrent connections | RAM | Notes |
|-------|------------------------|-----|-------|
| Node.js + Nginx, idle heartbeat | ~50K | 4 GB | Tuned ulimit |
| Go SSE server | ~200K | 4 GB | Lightweight goroutines |
| PHP-FPM (process per conn) | ~500 | 4 GB | Use Swoole/RoadRunner instead |
| Symfony Mercure hub | ~40K | 2 GB | Managed reverse proxy |

---

## 📝 সারসংক্ষেপ

- **SSE = HTTP-based one-way real-time stream।** Plain text `text/event-stream`, browser native `EventSource`।
- WebSocket-এর তুলনায় simpler, HTTP-friendly, auto-reconnect built-in; কিন্তু bi-directional নয়।
- **Format:** `id:`, `event:`, `data:`, `retry:` field; double `\n\n` event separator।
- **Reconnection:** Browser automatic; `Last-Event-ID` header দিয়ে server replay implement করতে হবে।
- **Pitfall #1:** Proxy/Nginx buffering। `X-Accel-Buffering: no` ও `proxy_buffering off` mandatory।
- **Heartbeat** comment line `:` দিয়ে 15-25s interval-এ — না হলে proxy timeout।
- **Use cases:** notification, order tracking, price ticker, log stream, AI streaming।
- **PHP-এ ReactPHP/Swoole/Mercure** — না হলে FPM process explosion।
- **Scaling:** Redis Pub/Sub / Kafka backbone, multi-instance fan-out।
- **Bangladesh-এ** Pathao Foods order tracking, Daraz seller dashboard, bKash merchant alerts — SSE আদর্শ।
