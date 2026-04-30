# 🔁 Polling Patterns — Short Polling, Long Polling এবং Real-time এর পরিবার

> **"Server-এ কিছু ঘটলে ক্লায়েন্ট কিভাবে জানবে?" — এই সাধারণ প্রশ্নের পেছনে আছে এক বিশাল ডিজাইন স্পেস।"**

Browser-এ "real-time update" দরকার — Foodpanda-তে delivery rider কোথায়, bKash-এ payment confirm
হলো কিনা, Pathao chat-এ নতুন message এসেছে কিনা। এই need মেটানোর সবচেয়ে পুরনো এবং সবচেয়ে firewall-
friendly পদ্ধতি হলো **polling**। আধুনিক যুগে SSE, WebSocket, WebTransport, gRPC streaming এসেছে —
কিন্তু polling এখনো অপ্রাসঙ্গিক হয়নি, বিশেষ করে Bangladesh-এর 3G/4G heterogeneous network ও
corporate firewall-এর প্রেক্ষিতে।

এই ডকুমেন্টে আমরা short polling থেকে শুরু করে long polling-এর deep mechanics, সম্পূর্ণ Node.js
implementation, PHP-FPM caveats, hybrid fallback, mobile NAT timeout, এবং Foodpanda/bKash/Pathao
সিনারিও পর্যন্ত গভীরে যাব।

---

## 📑 সূচিপত্র

1. [Polling কী এবং কেন এখনো প্রাসঙ্গিক](#-polling-কী-এবং-কেন-এখনো-প্রাসঙ্গিক)
2. [Short Polling — সরল কিন্তু costly](#-short-polling--সরল-কিন্তু-costly)
3. [Long Polling — server holds connection](#-long-polling--server-holds-connection)
4. [Comparison Matrix — সব real-time technique](#-comparison-matrix)
5. [Decision Tree — কোনটা কখন](#-decision-tree)
6. [Node.js Long Polling — full implementation](#-nodejs-long-polling--full-implementation)
7. [PHP Short Polling + PHP-FPM caveats](#-php-short-polling--php-fpm-caveats)
8. [Hybrid Fallback — SSE/WS fail হলে long-poll](#-hybrid-fallback)
9. [Operational Concerns — proxy, CDN, mobile NAT](#-operational-concerns)
10. [Push Notifications — out-of-band](#-push-notifications--out-of-band)
11. [BD Real-world Scenarios](#-bd-real-world-scenarios)
12. [Anti-patterns](#️-anti-patterns)
13. [Checklist](#-checklist)

---

## 📌 Polling কী এবং কেন এখনো প্রাসঙ্গিক

**Polling** = client periodically server-কে জিজ্ঞেস করে "কোনো update আছে?"। দু'ধরনের:

```
   Short Polling                    Long Polling
   ─────────────────────            ─────────────────────
   Client: GET?                     Client: GET (wait)
   Server: nothing.                 Server: ......
   (3s wait)                        (event আসা পর্যন্ত wait)
   Client: GET?                     Server: HERE'S DATA!
   Server: nothing.                 Client: GET (wait)
   (3s wait)                        Server: ......
   Client: GET?
   Server: HERE'S DATA!
```

### কেন এখনো প্রাসঙ্গিক?

```
  ✅ Stateless HTTP — কোনো বিশেষ protocol নয়
  ✅ Firewall/proxy traverse করে (corporate, BD office firewall)
  ✅ Mobile network NAT timeout-এর সাথে বন্ধুত্বপূর্ণ
  ✅ Load balancer-এর সাথে সহজে কাজ করে (no sticky session)
  ✅ Implementation trivial (২০ লাইন)
  ✅ SSE/WS fail হলে fallback
  ✅ Battery-friendly version ডিজাইন সম্ভব (interval জানা)
```

---

## 🔄 Short Polling — সরল কিন্তু costly

### Concept

```
   Client                           Server
     │                                │
     │── GET /orders/123/status ─────▶│
     │◄───── { status: "pending" } ───│
     │  (sleep 3s)                    │
     │── GET /orders/123/status ─────▶│
     │◄───── { status: "pending" } ───│
     │  (sleep 3s)                    │
     │── GET /orders/123/status ─────▶│
     │◄───── { status: "delivered" }──│
```

### Pros

```
  ✅ সরল implementation (client + server উভয়পক্ষে)
  ✅ Stateless — যেকোনো server reply করতে পারে
  ✅ Client offline হলে state নষ্ট হয় না
  ✅ Cache-friendly (ETag/Last-Modified)
  ✅ Battery-friendly যদি interval বড় হয়
```

### Cons

```
  ❌ Latency vs cost trade-off
     Interval ২s → 30 req/min/client
     ১M user × 30 req/min = ৫ লক্ষ req/sec → costly!
  ❌ অপ্রয়োজনীয় round-trip (৯৯% time "nothing new")
  ❌ Real-time না (interval-bound latency)
  ❌ Server load wasted CPU/IO
```

### Trade-off math

```
   Polling interval বনাম resources
   ─────────────────────────────────
   1s    → 60 req/min/user, latency P99 = 1s   (costly)
   3s    → 20 req/min/user, latency P99 = 3s
   10s   → 6 req/min/user, latency P99 = 10s
   60s   → 1 req/min/user, latency P99 = 60s   (real-time না)
```

**Rule of thumb:** Short polling তখনই appropriate যখন:
- Update infrequent (e.g., once-a-minute price update)
- Low concurrent user count (admin dashboard)
- Latency tolerance বেশি (10s+)

---

## ⏳ Long Polling — server holds connection

### Concept

```
   Client                           Server
     │                                │
     │── GET /events?since=42 ──────▶│
     │                                │ ── এই request open রাখা
     │                                │    event এর জন্য wait
     │                                │
     │  (no traffic, connection idle) │
     │                                │
     │                                │ ◄── event arrives!
     │◄── { events: [...], cursor:43}─│
     │                                │
     │── GET /events?since=43 ──────▶│  (immediate reconnect)
```

Server event আসা পর্যন্ত request hold করে — অথবা **maximum timeout** (e.g., 30s) পরে empty response।

### Pros

```
  ✅ Lower latency than short polling (event ↔ delivery)
  ✅ কম redundant request (90%+ reduction)
  ✅ HTTP-based — firewall/proxy friendly
  ✅ SSE/WS-এর fallback হিসেবে আদর্শ
```

### Cons

```
  ❌ Server-এ open connection ধরে রাখতে হয়
     (PHP-FPM এর সাথে worker exhaustion সমস্যা)
  ❌ Head-of-line blocking — সব update একসাথে আসে না
  ❌ Edge proxy timeout (Nginx default 60s)
  ❌ Mobile NAT কতক্ষণ idle TCP conn সহ্য করবে — অজানা
  ❌ Reconnect storm (server restart হলে সবাই একসাথে reconnect)
```

### Why server holds connection?

Server side-এ event-driven ভাবে event এর জন্য listen — Node.js EventEmitter, Redis pub/sub, Kafka
consumer। যখন event আসে, hold-করা request resolve করা হয়।

---

## 📊 Comparison Matrix

```
┌──────────────────┬─────────┬─────────┬────────┬────────┬──────────┬─────────┐
│ Feature          │ Short   │ Long    │ SSE    │ WebSkt │ WebTrans │ gRPC ss │
│                  │ Polling │ Polling │        │        │ port     │ stream  │
├──────────────────┼─────────┼─────────┼────────┼────────┼──────────┼─────────┤
│ Latency          │ Poll    │ ~event  │ ~event │ ~event │ ~event   │ ~event  │
│                  │ interval│         │        │        │          │         │
│ Server cost      │ High    │ Med     │ Low    │ Low    │ Low      │ Low     │
│ Bidirectional    │ ✓ (req) │ ✓ (req) │ ❌     │ ✅     │ ✅       │ ❌      │
│ Browser support  │ All     │ All     │ Modern │ All    │ Limited  │ N/A    │
│ Firewall friendly│ ✅      │ ✅      │ ✅     │ 🟡     │ ❌       │ 🟡      │
│ Reconnect logic  │ Trivial │ Easy    │ Built  │ Manual │ Manual   │ Manual  │
│                  │         │         │ -in    │        │          │         │
│ Message size     │ Any     │ Any     │ Text   │ Any    │ Any      │ Any     │
│ Backpressure     │ N/A     │ N/A     │ Limit. │ Manual │ Built-in │ Built-in│
│ Compression      │ gzip    │ gzip    │ gzip   │ permsg │ QUIC     │ HPACK   │
│                  │         │         │        │ deflat │          │         │
│ Mobile-friendly  │ ✅      │ 🟡      │ 🟡     │ 🟡     │ ✅ (UDP) │ 🟡      │
│ Stateful?        │ No      │ Per-req │ Yes    │ Yes    │ Yes      │ Yes     │
│ HTTP/2 multiplex │ Yes     │ Yes     │ Yes    │ Yes (h2│ N/A      │ Native  │
│                  │         │         │        │ ws)    │          │         │
│ Implementation   │ ★       │ ★★      │ ★★     │ ★★★    │ ★★★★     │ ★★★    │
│ complexity       │         │         │        │        │          │         │
└──────────────────┴─────────┴─────────┴────────┴────────┴──────────┴─────────┘
```

আরও বিস্তারিত:
- [WebSockets](../14-networking/websockets.md)
- [Server-Sent Events](./server-sent-events.md)
- [gRPC](./grpc.md)

---

## 🌳 Decision Tree

```
                    ┌────────────────────────────────────┐
                    │ Real-time update দরকার?            │
                    └─────────────┬──────────────────────┘
                                  │
                            Yes ──┘
                                  │
                    ┌─────────────▼──────────────────────┐
                    │ Bidirectional communication লাগবে? │
                    └────┬────────────────────┬──────────┘
                    Yes  │                    │ No
                         ▼                    ▼
                  ┌──────────────┐    ┌────────────────┐
                  │ Browser      │    │ One-way (server│
                  │ priority?    │    │ → client)      │
                  └──┬───────┬───┘    └────┬───────────┘
                Yes  │       │ No          │
                     ▼       ▼             ▼
                ┌────────┐ ┌──────┐  ┌─────────────────┐
                │ Web    │ │ gRPC │  │ Update frequency │
                │ Socket │ │ bidi │  │ ও latency budget?│
                └────────┘ └──────┘  └────┬─────────────┘
                                          │
                       ┌──────────────────┼──────────────┐
                       │                  │              │
                  high freq.         med freq.       low freq.
                  low latency        mod latency     hi latency OK
                       │                  │              │
                       ▼                  ▼              ▼
                  ┌────────┐      ┌──────────┐   ┌──────────┐
                  │ SSE    │      │ Long Poll│   │ Short    │
                  │        │      │          │   │ Polling  │
                  └────────┘      └──────────┘   └──────────┘
```

### Practical guidelines

| Use Case | Recommendation |
|---|---|
| Live stock price | SSE বা WebSocket |
| Chat (typing indicator + msg) | WebSocket |
| Order status | Long polling বা SSE |
| Daily price refresh | Short polling (60s) |
| File upload progress | SSE |
| Multiplayer game | WebSocket / WebTransport |
| Admin dashboard refresh | Short polling (10-30s) |
| Mobile push (background) | Push notification (APNs/FCM) |
| Microservice streaming | gRPC server-streaming |
| Corporate firewall stuck | Short polling বা long polling |

---

## 💻 Node.js Long Polling — full implementation

পূর্ণাঙ্গ example: order status update। EventEmitter-based, timeout-bounded, idempotent cursor-aware:

```javascript
// long-poll-server.js
const express = require('express');
const { EventEmitter } = require('node:events');
const Redis = require('ioredis');

const app = express();
app.use(express.json());

const redis = new Redis(process.env.REDIS_URL);
const orderBus = new EventEmitter();
orderBus.setMaxListeners(10000);    // many concurrent waiters

// In-memory event log (production-এ Redis Stream / Kafka)
// orderId → [{ seq, status, timestamp }, ...]
const eventLog = new Map();

function appendEvent(orderId, status) {
  if (!eventLog.has(orderId)) eventLog.set(orderId, []);
  const log = eventLog.get(orderId);
  const seq = log.length + 1;
  const event = { seq, orderId, status, timestamp: Date.now() };
  log.push(event);
  orderBus.emit(`order:${orderId}`, event);
  // টাইম-ভিত্তিক cleanup আলাদা job-এ
  return event;
}

// Subscribe to Redis pub/sub if multi-instance
redis.duplicate().subscribe('order-updates', () => {});
redis.duplicate().on('message', (_ch, payload) => {
  const ev = JSON.parse(payload);
  orderBus.emit(`order:${ev.orderId}`, ev);
});

// ─── Long-poll endpoint ───
const MAX_HOLD_MS = 25_000;        // < CDN/LB timeout (default 30-60s)
const MIN_HOLD_MS = 1_000;
const ALLOWED_HOLD_MAX = 60_000;

app.get('/api/orders/:id/events', async (req, res) => {
  const { id } = req.params;
  const sinceSeq = parseInt(req.query.since, 10) || 0;
  const holdMs = Math.min(
    ALLOWED_HOLD_MAX,
    Math.max(MIN_HOLD_MS, parseInt(req.query.holdMs, 10) || MAX_HOLD_MS)
  );

  // ১. Fast path: ইতিমধ্যে event আছে কিনা
  const log = eventLog.get(id) || [];
  const newEvents = log.filter(e => e.seq > sinceSeq);
  if (newEvents.length > 0) {
    return res.json({ events: newEvents, cursor: newEvents.at(-1).seq });
  }

  // ২. Slow path: hold করুন event arrival পর্যন্ত / timeout পর্যন্ত
  let resolved = false;
  const channel = `order:${id}`;

  const onEvent = (event) => {
    if (resolved) return;
    if (event.seq <= sinceSeq) return;
    cleanup();
    res.json({ events: [event], cursor: event.seq });
  };

  const onTimeout = () => {
    if (resolved) return;
    cleanup();
    res.json({ events: [], cursor: sinceSeq, timedOut: true });
  };

  const onClientAbort = () => {
    if (resolved) return;
    cleanup();
    // no response — already disconnected
  };

  const cleanup = () => {
    resolved = true;
    clearTimeout(timer);
    orderBus.off(channel, onEvent);
    req.off('close', onClientAbort);
  };

  const timer = setTimeout(onTimeout, holdMs);
  orderBus.on(channel, onEvent);
  req.on('close', onClientAbort);
});

// ─── Webhook → event source ───
app.post('/internal/order-event', (req, res) => {
  const ev = appendEvent(req.body.orderId, req.body.status);
  redis.publish('order-updates', JSON.stringify(ev));    // broadcast to other instances
  res.status(202).send();
});

// Health check
app.get('/health', (_req, res) => res.json({ ok: true, listeners: orderBus.eventNames().length }));

app.listen(3000, () => console.log('long-poll on :3000'));
```

### Client-side reconnect logic (browser)

```javascript
// long-poll-client.js
class OrderEventStream {
  constructor(orderId, onEvent, opts = {}) {
    this.orderId = orderId;
    this.onEvent = onEvent;
    this.cursor = opts.startCursor || 0;
    this.holdMs = opts.holdMs || 25000;
    this.maxBackoff = opts.maxBackoff || 30000;
    this.backoff = 500;
    this.aborted = false;
    this._loop();
  }

  abort() {
    this.aborted = true;
    if (this.controller) this.controller.abort();
  }

  async _loop() {
    while (!this.aborted) {
      this.controller = new AbortController();
      const url = `/api/orders/${this.orderId}/events?since=${this.cursor}&holdMs=${this.holdMs}`;
      try {
        const res = await fetch(url, { signal: this.controller.signal });
        if (res.status === 504 || res.status === 502) throw new Error('proxy_timeout');
        if (!res.ok) throw new Error(`status_${res.status}`);
        const data = await res.json();
        for (const ev of (data.events || [])) {
          this.onEvent(ev);
          this.cursor = Math.max(this.cursor, ev.seq);
        }
        this.backoff = 500;        // success → reset
      } catch (err) {
        if (this.aborted) return;
        if (err.name === 'AbortError') return;
        // jittered backoff
        const sleep = this.backoff / 2 + Math.random() * (this.backoff / 2);
        await new Promise(r => setTimeout(r, sleep));
        this.backoff = Math.min(this.backoff * 2, this.maxBackoff);
      }
    }
  }
}

// ব্যবহার (Foodpanda customer order tracking)
const stream = new OrderEventStream('ORD-9988', (ev) => {
  console.log('Order event:', ev);
  updateOrderUI(ev.status);
});

window.addEventListener('beforeunload', () => stream.abort());
```

### Key design points

```
  ✅ holdMs < proxy timeout (CDN default 30s)
  ✅ Cursor-based (since=N) — duplicate event এড়াতে
  ✅ Timeout-এ empty response (504 না)
  ✅ req.on('close') cleanup → leak prevention
  ✅ Redis pub/sub for multi-instance broadcast
  ✅ Client: jittered exponential backoff
  ✅ Client: proxy timeout (502/504) detect করে quick reconnect
```

---

## 🐘 PHP Short Polling + PHP-FPM caveats

### সাধারণ short polling endpoint

```php
<?php
// app/Http/Controllers/OrderStatusController.php
namespace App\Http\Controllers;

use App\Models\Order;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class OrderStatusController extends Controller
{
    public function status(Request $request, string $orderId): JsonResponse
    {
        $order = Order::select('id', 'status', 'updated_at')
            ->findOrFail($orderId);

        $etag = sha1("{$order->id}:{$order->status}:{$order->updated_at}");

        if ($request->header('If-None-Match') === $etag) {
            return response()->json(null, 304)->header('ETag', $etag);
        }

        return response()
            ->json([
                'id' => $order->id,
                'status' => $order->status,
                'updatedAt' => $order->updated_at->toIso8601String(),
            ])
            ->header('ETag', $etag)
            ->header('Cache-Control', 'no-cache, must-revalidate')
            ->header('X-Poll-Interval', $this->suggestedInterval($order));
    }

    private function suggestedInterval(Order $order): int
    {
        // adaptive: terminal status হলে stop polling
        return match ($order->status) {
            'delivered', 'cancelled', 'failed' => 0,    // client should stop
            'preparing', 'on_the_way' => 3,
            default => 10,
        };
    }
}
```

### Client (vanilla JS)

```javascript
async function pollStatus(orderId, onUpdate) {
  let etag = null;
  let interval = 5000;
  let stop = false;

  while (!stop) {
    try {
      const res = await fetch(`/api/orders/${orderId}/status`, {
        headers: etag ? { 'If-None-Match': etag } : {},
      });
      if (res.status === 304) {
        // no change
      } else if (res.ok) {
        etag = res.headers.get('ETag');
        const data = await res.json();
        onUpdate(data);
        const suggested = parseInt(res.headers.get('X-Poll-Interval'), 10);
        if (suggested === 0) stop = true;
        else if (!isNaN(suggested)) interval = suggested * 1000;
      } else if (res.status === 404) {
        return;
      }
    } catch (e) { /* network — retry */ }
    await new Promise(r => setTimeout(r, interval));
  }
}
```

### ⚠️ PHP-FPM Long Polling Caveats

PHP-FPM-এ long polling **না করার** কারণ:

```
   PHP-FPM pool: 50 worker
   Long polling 30s hold each
   ──────────────────────────────
   50 user simultaneously polling = সব worker locked
   51-th user → 502 Bad Gateway
   
   কেউ যদি API browse করতে চায় → cannot serve
   (worker exhaustion)
```

### সমাধান: Swoole / ReactPHP

```php
<?php
// Swoole long-poll server
$server = new Swoole\HTTP\Server('0.0.0.0', 9501);
$server->set([
    'worker_num' => 4,
    'task_worker_num' => 0,
    'max_request' => 0,
]);

$waiters = [];      // orderId → [response, timer]

$server->on('request', function ($req, $res) use ($server, &$waiters) {
    $path = $req->server['request_uri'] ?? '/';
    if (preg_match('#^/api/orders/(\w+)/events$#', $path, $m)) {
        $orderId = $m[1];
        $sinceSeq = (int) ($req->get['since'] ?? 0);

        // Fast path
        $events = OrderEventStore::since($orderId, $sinceSeq);
        if (!empty($events)) {
            $res->header('Content-Type', 'application/json');
            $res->end(json_encode(['events' => $events]));
            return;
        }

        // Slow path: register waiter
        $waiters[$orderId][] = $res;
        $timer = Swoole\Timer::after(25_000, function () use (&$waiters, $orderId, $res) {
            $waiters[$orderId] = array_filter($waiters[$orderId] ?? [], fn($r) => $r !== $res);
            if (!$res->isWritable()) return;
            $res->end(json_encode(['events' => [], 'timedOut' => true]));
        });
    }
});

// Subscribe to event source (Redis)
$redis = new Redis();
$redis->subscribe(['order-updates'], function ($_, $channel, $msg) use (&$waiters) {
    $event = json_decode($msg, true);
    $list = $waiters[$event['orderId']] ?? [];
    foreach ($list as $res) {
        if ($res->isWritable()) $res->end(json_encode(['events' => [$event]]));
    }
    unset($waiters[$event['orderId']]);
});

$server->start();
```

> **Recommendation:** PHP shop-এ long polling দরকার হলে Swoole/ReactPHP/Workerman (কোনো async runtime)
> ব্যবহার করুন। PHP-FPM-এ short polling ব্যবহার করুন।

---

## 🔁 Hybrid Fallback

Corporate network, BD-র কিছু ISP, পুরনো proxy — WebSocket/SSE block করতে পারে। Production-এ **graceful
fallback**:

```javascript
// universal-events.js
class UniversalEventStream {
  constructor(url, onEvent) {
    this.url = url;
    this.onEvent = onEvent;
    this._tryWebSocket() || this._trySSE() || this._fallbackLongPoll();
  }

  _tryWebSocket() {
    if (typeof WebSocket === 'undefined') return false;
    try {
      const ws = new WebSocket(this.url.replace(/^http/, 'ws') + '?transport=ws');
      let ok = false;
      ws.onopen = () => { ok = true; this.transport = 'ws'; };
      ws.onmessage = (e) => this.onEvent(JSON.parse(e.data));
      ws.onerror = () => { if (!ok) this._trySSE() || this._fallbackLongPoll(); };
      // 5s within open না হলে fallback
      setTimeout(() => { if (!ok) ws.close(); }, 5000);
      return true;
    } catch { return false; }
  }

  _trySSE() {
    if (typeof EventSource === 'undefined') return false;
    try {
      const es = new EventSource(this.url + '?transport=sse');
      es.onmessage = (e) => { this.transport = 'sse'; this.onEvent(JSON.parse(e.data)); };
      es.onerror = () => { es.close(); this._fallbackLongPoll(); };
      return true;
    } catch { return false; }
  }

  _fallbackLongPoll() {
    this.transport = 'long-poll';
    new OrderEventStream(this.url, this.onEvent);
  }
}
```

Socket.io-র আদি motivation এই — multi-transport graceful fallback।

---

## 🏭 Operational Concerns

### Nginx proxy timeouts

```nginx
location /api/orders/ {
    proxy_pass http://upstream;
    proxy_http_version 1.1;

    # Long polling endpoint-এর জন্য
    proxy_read_timeout    60s;     # default 60s — হোল্ড > এটার চেয়ে কম রাখুন (e.g., 25s)
    proxy_send_timeout    60s;
    proxy_buffering       off;     # SSE/long-poll-এ buffering bad
    proxy_cache           off;

    # Connection: keep-alive
    proxy_set_header Connection "";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

আরও: [Nginx Deep Dive](../14-networking/nginx-deep-dive.md) (যদি বিদ্যমান)

### CDN behavior

CDN (Cloudflare, AWS CloudFront) by default:
- Connection idle timeout: 30-60s
- Long-held connection cap: per-edge
- WebSocket: needs explicit upgrade allow

> **Tip:** Cloudflare-এ free tier-এ idle 100s, paid plan কম। `holdMs` সেটিকে 25-30s রাখুন।

### Mobile network NAT timeout (BD context)

বাংলাদেশে Grameenphone/Robi/Banglalink mobile network-এ NAT translation table-এর idle timeout
সাধারণত:

```
   3G/HSPA NAT timeout:    ~3-5 min idle
   4G/LTE NAT timeout:     ~10-30 min idle (carrier-specific)
   WiFi-via-router NAT:    ~30 min - 24 hr
```

ফলাফল: WebSocket idle থাকলে hidden disconnection (TCP RST আসে না)। সমাধান:
- WebSocket: ping/pong every 30-60s
- SSE: server-side keepalive comment `: ping\n\n` every 15s
- Long polling: holdMs < NAT timeout (problem solved naturally)

### Connection limits (LB)

```
   AWS ALB:        per-target HTTP connection limit
   GCP LB:         backend service max-connections
   Browser:        per-host concurrent connection 6 (HTTP/1.1), 100s (HTTP/2 multiplex)
```

Long polling = প্রতি tab-এ ১টি persistent connection। ১M user × 1 tab = ১M open connection — single
node handle করতে পারবে না (file descriptor limit, RAM)।

```
   Node.js single process:    ~50-100K concurrent (tuned)
   Go server:                 ~500K-1M
   Erlang (WhatsApp legacy):  ~2M+
```

---

## 📲 Push Notifications — out-of-band

App background-এ থাকলে polling অর্থহীন (battery drain, OS kill করে)। Solution: **push notification**।

```
   Server                   Push Service              Device
     │                          │                       │
     │── send (token, msg) ───▶│                       │
     │                          │── deliver ──────────▶│
     │                          │                       │
     │                          │  (battery-friendly,    │
     │                          │   OS-managed)          │
```

### Channels (BD-relevant)

```
   Android:    FCM (Firebase Cloud Messaging)
   iOS:        APNs (Apple Push Notification service)
   Web:        Web Push API (VAPID)
   SMS:        SSL Wireless, Banglalink, GP API → SMS gateway
   In-app:     polling/SSE when app foreground
```

### When to push vs poll

| Scenario | Recommendation |
|---|---|
| App in foreground | SSE / WebSocket / long-poll |
| App in background | Push (FCM/APNs) |
| User logged out | SMS / Email |
| Critical (payment) | Push + SMS (redundancy) |
| Non-critical (promo) | Push only |

---

## 🇧🇩 BD Real-world Scenarios

### Foodpanda customer order tracking

WebSocket আগে — orders flooding করছিল WS server-এ।

```
   Architecture (legacy, long polling):
   ───────────────────────────────────
   Customer App
        │
        │ GET /orders/{id}/track?since=N (holdMs=25s)
        ▼
   API Gateway → Order-Tracking Service (Node.js)
                       │
                       │ (subscribe Redis pub/sub: order:{id})
                       │
                ┌──────┴──────┐
                │             │
                ▼             ▼
              Driver       Kitchen
              GPS push     status push
```

Update flow:
- Order placed → "preparing" event → push to all subscribers
- Rider picks up → "on_the_way" + GPS lat/lng → push every 10s
- Delivered → "delivered" → terminal event, client stops polling

### bKash merchant transaction status (short polling)

merchant-এ payment confirm হলো কিনা, এটা merchant's backend POS short-polls bKash:

```
   POS Backend                bKash API
       │                          │
       │── GET /trx/TRX123/status ▶│
       │◄── { status: "PENDING" } │
       │  (sleep 5s)               │
       │── GET /trx/TRX123/status ▶│
       │◄── { status: "COMPLETED",│
       │      amount: 500, ... }   │
       │  (stop polling)           │
```

Production-এ webhook preferred, কিন্তু merchant-এর internal network webhook receive করতে পারে না
(no public IP) → polling fallback।

### Pathao chat — short polling fallback for poor 3G

Pathao chat default WebSocket। কিন্তু rural BD-তে 3G unstable:

```javascript
// Pathao chat client
let chat = new WebSocket(WS_URL);
let useFallback = false;

chat.onerror = chat.onclose = () => {
  if (!useFallback) {
    useFallback = true;
    // 3-attempt WS retry-এর পর short polling fallback
    startShortPoll();
  }
};

function startShortPoll() {
  const interval = setInterval(async () => {
    const since = localStorage.getItem('chat_cursor') || 0;
    try {
      const res = await fetch(`/api/chat/messages?since=${since}`, { timeout: 5000 });
      const data = await res.json();
      if (data.messages?.length) {
        localStorage.setItem('chat_cursor', data.cursor);
        data.messages.forEach(displayMessage);
      }
    } catch { /* network — keep trying */ }
  }, 4000);
}
```

---

## ⚠️ Anti-patterns

### ১. Short polling at 1s/client
```
   ❌ 1M user × 60 req/min = 1M req/sec → DDoS yourself
   ✅ Adaptive interval (3-30s based on activity); SSE/WS-এ migrate
```

### ২. Long polling without timeout cap
```
   ❌ Server forever holds → file descriptor exhaustion
   ✅ holdMs ≤ 30s; client reconnect on timeout
```

### ৩. Missing reconnection on 504
```
   ❌ Proxy 504 → client gives up → lost user
   ✅ jittered exponential backoff, transparent retry
```

### ৪. Polling deleted resource forever
```
   ❌ Order deleted → 404 → infinite poll
   ✅ Terminal status detect (delivered/cancelled/404) → stop
```

### ৫. Same interval for all resources
```
   ❌ active vs idle order একই 5s interval
   ✅ Adaptive: active 2s, idle 30s, terminal stop
```

### ৬. PHP-FPM + long polling
```
   ❌ Worker pool exhaustion
   ✅ Swoole/ReactPHP, বা short polling, বা SSE-via-Nginx-push
```

### ৭. No cursor / since parameter
```
   ❌ প্রতি poll-এ পুরো dataset → bandwidth waste, race condition
   ✅ since=N বা If-Modified-Since cursor
```

### ৮. CDN caching long-poll endpoint
```
   ❌ /api/poll cached → সবাই একই response
   ✅ Cache-Control: no-cache, no-store
```

### ৯. Battery-killer mobile polling
```
   ❌ 1s polling → mobile battery 30% in 1 hr
   ✅ Push notification + foreground polling only
```

### ১০. Holding connection longer than CDN timeout
```
   ❌ holdMs=120s, CDN timeout=60s → 502 every poll
   ✅ holdMs = CDN_timeout − 5s (safety margin)
```

---

## ✅ Checklist

```
Strategy:
[ ] Polling vs SSE vs WS vs gRPC — decision documented
[ ] Adaptive interval (active/idle/terminal)
[ ] Cursor-based (since/seq)
[ ] ETag / If-None-Match support (short polling)
[ ] Terminal status → client stops polling

Long Polling:
[ ] holdMs < proxy/CDN timeout (typically 25-30s)
[ ] Server-side cleanup on req.close
[ ] Multi-instance: Redis pub/sub broadcast
[ ] Empty-result on timeout (with timedOut flag)
[ ] Cursor persistence on reconnect

PHP:
[ ] PHP-FPM-এ long polling avoided OR Swoole/ReactPHP
[ ] Short polling caching (ETag)
[ ] Adaptive interval header (X-Poll-Interval)

Client:
[ ] Reconnect logic with jittered backoff
[ ] Abort on page unload
[ ] Multi-transport fallback (WS → SSE → long-poll → short-poll)
[ ] Network change handler (online/offline event)
[ ] Cursor stored locally (resume after refresh)

Infrastructure:
[ ] Nginx/LB proxy_read_timeout > holdMs
[ ] proxy_buffering off for streaming
[ ] CDN cache disabled for poll endpoint
[ ] Connection limit budget calculated (file descriptors)
[ ] Mobile NAT timeout: keepalive ping configured (WS/SSE)

Observability:
[ ] Active polling connection count
[ ] Avg hold time (long poll)
[ ] Reconnect rate
[ ] 504 rate (proxy timeout)
[ ] Battery usage (mobile telemetry)

Security:
[ ] Auth on every poll (or session token)
[ ] Rate limit per user (prevent runaway clients)
[ ] CSRF protection
[ ] Stop polling on auth-fail (no infinite 401 loop)

Testing:
[ ] Slow network simulation (3G profile)
[ ] Proxy 504 simulation
[ ] Server restart → reconnect storm test
[ ] Mobile background → foreground transition test
```

---

## 📚 আরও পড়ুন

- [Server-Sent Events](./server-sent-events.md)
- [WebSockets](../14-networking/websockets.md)
- [gRPC](./grpc.md)
- [API Gateway](./api-gateway.md)
- [Webhook](./webhook.md)
- [REST API](./rest-api.md)
- [Rate Limiting](../03-system-design/rate-limiting.md)
- [Load Shedding](../03-system-design/load-shedding.md)

> **চূড়ান্ত কথা:** Polling "primitive" বলে অনেকে অবজ্ঞা করেন — কিন্তু **সঠিক জায়গায় polling-ই সবচেয়ে
> robust solution**। Bangladesh-এর mobile network heterogeneity, corporate firewall, IoT device
> battery — এই সব context-এ polling-এর জায়গা আছে। Modern stack-এ সাধারণত **SSE/WS as primary
> + long-poll as fallback** — সেটাই production-grade approach।
