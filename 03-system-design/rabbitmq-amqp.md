# 🐰 RabbitMQ & AMQP — Deep Dive

## 📌 সংজ্ঞা ও মূল ধারণা

**RabbitMQ** হলো একটি ওপেন-সোর্স **message broker**, যা মূলত **AMQP 0-9-1** (Advanced Message Queuing Protocol) implement করে। Erlang/OTP-তে লেখা — যার fault tolerance ও lightweight process model RabbitMQ-কে দিয়েছে rock-solid stability।

> RabbitMQ-এর মূল দর্শন: **Smart broker, dumb consumer।** Broker নিজেই decide করে কোন message কোথায় যাবে — routing logic broker-এ।
>
> Kafka-র দর্শন উল্টো: **Dumb broker, smart consumer।** Broker শুধু log রাখে; consumer নিজে offset ম্যানেজ করে।

### কেন RabbitMQ?

```
┌────────────────────────────────────────────────────────────┐
│  bKash — Send Money সম্পন্ন হওয়ার পর কী কী হবে?           │
│                                                            │
│  ❌ Synchronous (সব HTTP):                                │
│     SMS API   (200ms) ─┐                                  │
│     Push API  (300ms) ─┤                                  │
│     Email     (500ms) ─┼─→ User wait 1.2s + any failure   │
│     Ledger    (100ms) ─┘    means request fails 💀        │
│                                                            │
│  ✅ RabbitMQ topic exchange:                              │
│     Core service publishes 1 event in 5ms                 │
│     Topic routing → SMS / Push / Email / Ledger queues   │
│     Each consumer processes independently                 │
│     User gets response in 50ms ✅                         │
└────────────────────────────────────────────────────────────┘
```

RabbitMQ shines যখন:
- Flexible routing দরকার (topic patterns, headers)।
- Per-message ack, retry, DLX-এর মতো reliability primitives চাই।
- Throughput moderate (১০-৫০K msg/s per queue, classic; quorum-এ কম)।
- Multi-protocol (AMQP, MQTT, STOMP) দরকার।
- Operational simplicity > distributed log replay।

> 🔗 যদি বিশাল throughput বা long retention/replay দরকার, Kafka বেছে নাও — দেখো `/18-kafka/`।

---

## 🏗️ AMQP 0-9-1 Model — সম্পূর্ণ Conceptual Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│                       AMQP 0-9-1 Model                              │
│                                                                     │
│   ┌──────────┐                                                      │
│   │ Producer │                                                      │
│   └────┬─────┘                                                      │
│        │ publish(exchange, routing_key, body, props)                │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────┐       │
│   │                       BROKER                            │       │
│   │                                                         │       │
│   │   ┌─────────────────┐         ┌──────────────────────┐  │       │
│   │   │   EXCHANGE      │ binding │       QUEUE          │  │       │
│   │   │  (router)       │────────▶│  ┌──┬──┬──┬──┬──┐    │  │       │
│   │   │ direct/topic/   │ binding │  │M1│M2│M3│M4│M5│ →  │  │       │
│   │   │ fanout/headers  │────────▶│  └──┴──┴──┴──┴──┘    │  │       │
│   │   └─────────────────┘         └──────────────────────┘  │       │
│   │                                            │            │       │
│   └────────────────────────────────────────────┼────────────┘       │
│                                                ▼                    │
│                                          ┌──────────┐               │
│                                          │ Consumer │               │
│                                          │  (ack)   │               │
│                                          └──────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

### Connection, Channel, VHost

| Concept | কী | Notes |
|---------|------|-------|
| **Connection** | TCP/TLS connection broker-এ | Heavy — 1 per process recommended |
| **Channel** | Connection-এর ভেতরে virtual lightweight connection | Multiplex; thousands per connection; **thread-unsafe** — per-thread channel |
| **VHost** | Logical broker partition — exchanges/queues isolated | Multi-tenant; permissions per vhost |

```
                Connection (TCP)
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Channel    Channel    Channel
        (publish) (consume) (admin)
```

> ⚠️ Channels share thread না করার rule: `BasicPublish` + `BasicConsume` একই channel-এ multi-thread থেকে call করলে frame interleave-এ corruption হবে।

### Frame Types

AMQP wire protocol-এ চার ধরনের frame:

| Frame | Purpose |
|-------|---------|
| **METHOD** | RPC-style command (publish, ack, declare) |
| **HEADER** | Message properties (content-type, delivery_mode, priority...) |
| **BODY** | Actual payload bytes (chunked if > frame-max) |
| **HEARTBEAT** | Keep-alive (default 60s); broker drops idle conns |

```ini
# rabbitmq.conf
heartbeat = 60
frame_max = 131072        # 128 KB
```

---

## 🔀 Exchange Types — Routing-এর হৃদপিণ্ড

### 1. Direct Exchange — Exact routing key match

```
publish(routing_key="order.bd")
    │
    ▼
[Direct Exchange]
    ├─binding "order.bd"   ──▶ [Queue: BD orders]
    ├─binding "order.pk"   ──▶ [Queue: PK orders]
    └─binding "order.bd"   ──▶ [Queue: Audit (multi-bind ok)]
```

```bash
rabbitmqadmin declare exchange name=orders type=direct durable=true
rabbitmqadmin declare queue name=orders.bd durable=true
rabbitmqadmin declare binding source=orders destination=orders.bd routing_key=order.bd
```

**Use case:** Daraz-এ region-wise order routing — `order.bd`, `order.pk`, `order.mm`।

### 2. Topic Exchange — Wildcard pattern

- `*` = exactly one word
- `#` = zero or more words

```
publish(routing_key="payment.bd.bkash.success")
       ▼
[Topic Exchange]
       ├─bind "payment.bd.*.success"     ──▶ [BD success notifications]
       ├─bind "payment.#"                ──▶ [Audit catch-all]
       ├─bind "payment.*.bkash.*"        ──▶ [bKash analytics]
       └─bind "payment.bd.nagad.success" ──▶ [Nagad — won't match]
```

**Use case:** bKash event streaming, Pathao city-wise notification (`ride.dhaka.completed`, `ride.ctg.cancelled`)।

### 3. Fanout Exchange — Broadcast to all bound queues

```
publish(routing_key="ignored")
       ▼
[Fanout Exchange]
   ├──▶ [Push notification queue]
   ├──▶ [Email queue]
   ├──▶ [Analytics queue]
   └──▶ [Audit queue]
```

Routing key-কে ignore করে — সব bound queue-এ copy যায়।

**Use case:** Daraz price-change event → cache invalidation, search reindex, recommendation update — সব একসাথে।

### 4. Headers Exchange — Match by message headers

```
publish(headers={region: "bd", channel: "sms"})
       ▼
[Headers Exchange]
   binding x-match=all, region=bd, channel=sms ──▶ [BD SMS queue] ✅
   binding x-match=any, region=bd              ──▶ [BD anything]  ✅
```

`x-match=all` মানে সব header match করতে হবে; `any` মানে যেকোনো একটি। কম ব্যবহৃত — topic exchange সাধারণত যথেষ্ট।

### 5. Default Exchange (`""`) — Implicit direct

প্রতিটি queue auto-bound to default exchange with routing key = queue name।

```js
ch.publish('', 'orders.bd', Buffer.from('...'));   // → orders.bd queue
```

Quick prototype-এ ব্যবহার হয়, প্রোডাকশনে explicit exchange ভালো।

---

## 📥 Queue Features — সব option

### Basic flags

```js
ch.assertQueue('orders.bd', {
    durable: true,         // broker restart-এ queue থাকবে
    exclusive: false,      // শুধু এই connection-এই, disconnect-এ delete
    autoDelete: false,     // last consumer চলে গেলে delete
    arguments: {
        'x-message-ttl': 86400000,           // queue-level TTL: 24h
        'x-max-length': 100000,              // overflow → drop-head/reject-publish
        'x-max-length-bytes': 1073741824,    // 1 GB
        'x-overflow': 'reject-publish',      // or drop-head, reject-publish-dlx
        'x-dead-letter-exchange': 'orders.dlx',
        'x-dead-letter-routing-key': 'orders.failed',
        'x-queue-type': 'quorum',            // classic / quorum / stream
        'x-queue-mode': 'lazy',              // disk-first (classic only)
        'x-max-priority': 10,                // priority queue (classic)
        'x-single-active-consumer': true,    // ordered processing
    }
});
```

### Persistent vs Transient Messages

```js
ch.publish('orders', 'order.bd', Buffer.from(JSON.stringify(order)), {
    persistent: true,            // delivery_mode=2 → fsync to disk
    contentType: 'application/json',
    messageId: order.id,
    timestamp: Date.now(),
    headers: { traceId: ctx.traceId },
});
```

> ⚠️ **Durable queue + Persistent message + Publisher confirm** — তিনটাই দরকার full reliability-এর জন্য। শুধু durable queue declare করলে message disk-এ যায় না; `persistent: true` লাগবে।

### Lazy Queue (classic only)

```js
arguments: { 'x-queue-mode': 'lazy' }
```

Message সাথে সাথে disk-এ লেখে, RAM-এ minimum রাখে। মিলিয়ন message-এর backlog handle করতে — Foodpanda invoice batch generation।

### Priority Queue (classic only)

```js
ch.assertQueue('payments', { arguments: { 'x-max-priority': 10 } });
ch.publish('', 'payments', body, { priority: 9 });   // VIP user
ch.publish('', 'payments', body, { priority: 0 });   // free tier
```

> ⚠️ Quorum queue priority queue support করে না — classic queue লাগবে।

### Single Active Consumer

```js
arguments: { 'x-single-active-consumer': true }
```

একসাথে শুধু একজন consumer message পায়; সে disconnect হলে next consumer activate। **Order-preserving processing** চাইলে।

### Quorum Queues (RabbitMQ 3.8+, default in 4.x)

Raft-based replicated queue — Mirrored queue-এর modern replacement।

```js
arguments: { 'x-queue-type': 'quorum', 'x-quorum-initial-group-size': 3 }
```

| Feature | Classic Mirrored | Quorum |
|---------|------------------|--------|
| Replication | Eventual, leader-follower | Strong consistency (Raft) |
| Failover | Manual coordination, can lose msgs | Automatic, no message loss |
| Performance | Faster but unsafe | Slightly slower, safer |
| Memory | RAM-heavy | Disk-first by default |
| Priorities/TTL | ✅ | ⚠️ Partial (no priority, TTL ok) |
| Status | **Deprecated** | **Recommended** |

### Stream Queues (RabbitMQ 3.9+)

Kafka-like append-only log; replay support।

```js
ch.assertQueue('events', {
    durable: true,
    arguments: {
        'x-queue-type': 'stream',
        'x-max-length-bytes': 10737418240,   // 10 GB cap
        'x-max-age': '7D',
    }
});
// consumer offset
ch.consume('events', handler, {
    arguments: { 'x-stream-offset': 'first' /* or timestamp/last/N */ }
});
```

Stream queues use case: event sourcing, replay debugging — Kafka-র চেয়ে ছোট স্কেলে।

### Dead Letter Exchange (DLX)

Message DLX-এ যায় যখন:
1. Consumer reject/nack with `requeue: false`।
2. Message TTL expire।
3. Queue length limit hit (overflow)।

```
[Main Queue: orders]
    │ reject(requeue=false) / TTL expire
    ▼
[DLX: orders.dlx]
    │ binding routing_key="orders.failed"
    ▼
[Queue: orders.dlq]   ── alarmed when count > 0
```

#### Retry Ladder Pattern (1m → 5m → 30m → DLQ)

```
order.queue ──reject──▶ retry.1m  ──TTL 60s──▶ DLX ──▶ order.queue
                          (or)
order.queue ──reject──▶ retry.5m  ──TTL 300s──▶ DLX ──▶ order.queue
                          (or, after N attempts)
order.queue ──reject──▶ parking.lot  ── manual review
```

Header `x-death` count পরীক্ষা করে কোন retry queue-এ পাঠাতে হবে decide করো।

```js
// Consumer logic
const xDeath = msg.properties.headers?.['x-death'] ?? [];
const attempts = xDeath[0]?.count ?? 0;

if (attempts >= 5) {
    ch.publish('parking.lot.exchange', '', msg.content, msg.properties);
    ch.ack(msg);
    return;
}
const ttlQueue = ['retry.1m', 'retry.5m', 'retry.30m'][Math.min(attempts, 2)];
ch.publish('', ttlQueue, msg.content, msg.properties);
ch.ack(msg);
```

---

## ✅ Reliability — End-to-End Guarantees

### Publisher Confirms

Broker accept করার পর producer-কে ack পাঠায়।

```js
const ch = await conn.createConfirmChannel();
ch.publish('orders', 'order.bd', Buffer.from(json), { persistent: true });
await ch.waitForConfirms();   // throws on nack
```

PHP php-amqplib:
```php
$ch->confirm_select();
$ch->set_ack_handler(fn($msg) => log("ACK"));
$ch->set_nack_handler(fn($msg) => log("NACK — retry"));
$ch->basic_publish($amqpMsg, 'orders', 'order.bd');
$ch->wait_for_pending_acks(5.0);
```

> 💡 **Confirms vs Transactions:** AMQP `tx.select` ১০০x slower (full sync per publish)। Always use confirms in modern code।

### Consumer Acknowledgements

```js
ch.consume('orders.bd', async (msg) => {
    try {
        await processOrder(JSON.parse(msg.content));
        ch.ack(msg);
    } catch (e) {
        if (isTransient(e)) {
            ch.nack(msg, false, false);   // requeue=false → DLX
        } else {
            ch.reject(msg, false);        // poison → DLX
        }
    }
}, { noAck: false });
```

| Method | Effect |
|--------|--------|
| `ack(msg)` | Confirm processed, broker deletes |
| `nack(msg, multiple, requeue)` | Negative ack; requeue=true → back to queue |
| `reject(msg, requeue)` | Single-message nack |
| `noAck: true` (autoAck) | ❌ Avoid — message lost on consumer crash |

### Prefetch — Backpressure

```js
ch.prefetch(20);   // max 20 unacked messages per consumer
```

```
prefetch=1   → strict round-robin, slowest, fairest
prefetch=10  → balanced (default recommendation)
prefetch=100+ → high throughput, risk of slow consumer hoarding
prefetch=0   → unlimited (DANGER — RAM blowup)
```

> 🔗 Backpressure বিস্তারিত: `/12-concurrency/backpressure.md`।

### Delivery Guarantees

| Guarantee | How |
|-----------|-----|
| **At-most-once** | autoAck or noAck consumer |
| **At-least-once** | manual ack + persistent + confirms (**default for prod**) |
| **Exactly-once** | At-least-once + **idempotent consumer** (dedup by message_id in DB/Redis) |

> 🚨 RabbitMQ broker exactly-once দেয় না (Kafka-ও দেয় না full sense-এ)। Producer retry → duplicate possible → consumer-এ idempotency check করো।

### Mandatory + basic.return

```js
ch.publish('orders', 'unrouted.key', body, { mandatory: true });
ch.on('return', (msg) => {
    // routing fail করেছে — কোনো queue match হয়নি
});
```

`immediate` deprecated AMQP 0-9-1-এ।

---

## 🌐 Clustering & HA

### Erlang Cluster

RabbitMQ nodes Erlang clustering-এ যুক্ত। সবাই একই Erlang cookie share করে।

```bash
# Node 2 join Node 1
echo "$(cat /var/lib/rabbitmq/.erlang.cookie)" | ssh node2 "sudo tee /var/lib/rabbitmq/.erlang.cookie"
ssh node2 'rabbitmqctl stop_app && rabbitmqctl join_cluster rabbit@node1 && rabbitmqctl start_app'
```

Metadata (exchanges, queues, bindings, users) — সব node-এ replicated। Queue data — only on declared node, **unless** mirrored/quorum।

### Mirrored Queues (deprecated)

```
ha-mode: all/exactly/nodes
ha-sync-mode: automatic/manual
```

Master-slave; eventual consistency; can lose messages on partition। **3.8+ থেকে discouraged।**

### Quorum Queues (Raft)

Strong consistency; minimum 3 replicas। Leader হ্যান্ডল করে publish/consume; followers replay log।

```js
arguments: { 'x-queue-type': 'quorum' }
```

```bash
rabbitmq-diagnostics quorum_status orders.bd
```

### Federation vs Shovel vs Cluster

| Mechanism | Topology | Latency | Network | Use case |
|-----------|----------|---------|---------|----------|
| **Cluster** | Same LAN, single logical broker | Low | High (Erlang messaging) | Single DC HA |
| **Federation** | Loose coupling between brokers; per-exchange/queue | Higher | WAN-friendly | Multi-DC pub/sub |
| **Shovel** | Move messages from queue → another broker | Configurable | WAN-friendly | One-way bridge, migrations |

### Network Partitions

```
cluster_partition_handling = pause_minority
# alternatives: ignore, autoheal
```

| Strategy | Behavior |
|----------|----------|
| `ignore` | উভয় partition চলবে — split-brain risk |
| `autoheal` | Partition resolve হলে winner side ঠিক করে, loser-কে restart |
| `pause_minority` | **Recommended** — minority side pause, majority কাজ চালু রাখে |

> 💡 ৩-নোড cluster + `pause_minority` quorum-এর মতো কাজ করে metadata-র জন্য।

### Multi-DC Strategies

- **Federation:** Active-active producer; consumers federated downstream।
- **Shovel:** Warm standby DC — queues mirror via shovel; failover manual।
- **Two clusters + app-level dual-publish:** Most robust but expensive।

---

## 🔌 Plugins

| Plugin | Port/Use |
|--------|----------|
| **rabbitmq_management** | Web UI :15672, REST API |
| **rabbitmq_prometheus** | Metrics :15692 |
| **rabbitmq_shovel** + UI | Move messages |
| **rabbitmq_federation** + UI | Loose multi-broker linking |
| **rabbitmq_delayed_message_exchange** | Per-message delay |
| **rabbitmq_consistent_hash_exchange** | Hash-based partitioning across queues |
| **rabbitmq_mqtt** | MQTT 3.1.1 :1883 |
| **rabbitmq_stomp** | STOMP :61613 |
| **rabbitmq_amqp1_0** | AMQP 1.0 (different from 0-9-1) |
| **rabbitmq_auth_backend_ldap** | LDAP auth |
| **rabbitmq_auth_backend_oauth2** | OAuth2/JWT |

```bash
rabbitmq-plugins enable rabbitmq_management rabbitmq_prometheus rabbitmq_delayed_message_exchange
```

---

## 🔒 Security

### TLS

```ini
listeners.ssl.default = 5671
listeners.tcp = none

ssl_options.cacertfile = /etc/rabbitmq/ca.crt
ssl_options.certfile   = /etc/rabbitmq/server.crt
ssl_options.keyfile    = /etc/rabbitmq/server.key
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

### x509 client auth

```bash
rabbitmq-plugins enable rabbitmq_auth_mechanism_ssl
```

### Permissions per vhost

```bash
rabbitmqctl add_vhost /bkash
rabbitmqctl add_user payments StrongPass
rabbitmqctl set_permissions -p /bkash payments \
    "^payments\..*"   "^payments\..*"   "^payments\..*"
#    configure          write             read
```

Three regex patterns: configure (declare/delete), write (publish), read (consume/get/bind)।

### Resource limits

```ini
disk_free_limit.absolute = 5GB
vm_memory_high_watermark.relative = 0.6      # 60% RAM
vm_memory_high_watermark.paging_ratio = 0.5  # paging starts at 50% of HW
max_connections = 5000
max_channels = 100000
```

---

## 🛠️ Operations

### CLI tools

```bash
rabbitmqctl status
rabbitmqctl cluster_status
rabbitmqctl list_queues name messages consumers memory state
rabbitmqctl list_connections
rabbitmqctl list_exchanges name type
rabbitmqctl list_bindings
rabbitmqctl list_consumers

rabbitmqctl set_policy ha-quorum "^orders\." \
    '{"queue-type":"quorum"}' --apply-to queues

rabbitmq-diagnostics ping
rabbitmq-diagnostics check_running
rabbitmq-diagnostics check_local_alarms
rabbitmq-diagnostics memory_breakdown
rabbitmq-diagnostics observer
```

### Memory & disk alarms

Memory > `vm_memory_high_watermark` → publishers blocked (consumers চলবে)। Disk < `disk_free_limit` → একই।

```bash
rabbitmqctl status | grep alarm
```

### Upgrade Strategy

- **Rolling upgrade:** ১ node-এ একবার, mixed-version cluster supported within minor releases।
- **Blue-green vhost:** নতুন cluster, federation/shovel দিয়ে drain, app cutover।
- Mirrored queue থেকে quorum-এ migration: queue declare new, drain old।

---

## 💻 Patterns — PHP (php-amqplib) + Node (amqplib)

### 1. Work Queue (load balancing)

**Producer (PHP):**
```php
<?php
require 'vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$conn = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest', '/');
$ch = $conn->channel();
$ch->confirm_select();
$ch->queue_declare('invoice.gen', false, true, false, false, false,
    new \PhpAmqpLib\Wire\AMQPTable(['x-queue-type' => 'quorum']));

for ($i = 1; $i <= 1000; $i++) {
    $msg = new AMQPMessage(
        json_encode(['orderId' => $i]),
        ['delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
         'content_type'  => 'application/json',
         'message_id'    => uniqid('inv_', true)]
    );
    $ch->basic_publish($msg, '', 'invoice.gen');
}
$ch->wait_for_pending_acks(10);
$ch->close(); $conn->close();
```

**Consumer (PHP):**
```php
$ch->basic_qos(null, 20, null);   // prefetch 20
$ch->basic_consume('invoice.gen', '', false, false, false, false,
    function ($msg) use ($ch) {
        try {
            $data = json_decode($msg->getBody(), true);
            generateInvoice($data['orderId']);   // your logic
            $ch->basic_ack($msg->getDeliveryTag());
        } catch (\Throwable $e) {
            error_log("fail: " . $e->getMessage());
            $ch->basic_nack($msg->getDeliveryTag(), false, false);  // → DLX
        }
    });
while ($ch->is_consuming()) $ch->wait();
```

**Producer (Node):**
```js
const amqp = require('amqplib');
const conn = await amqp.connect('amqp://guest:guest@localhost');
const ch = await conn.createConfirmChannel();
await ch.assertQueue('invoice.gen', {
    durable: true, arguments: { 'x-queue-type': 'quorum' },
});
for (let i = 1; i <= 1000; i++) {
    ch.sendToQueue('invoice.gen', Buffer.from(JSON.stringify({ orderId: i })),
        { persistent: true, messageId: `inv_${i}`, contentType: 'application/json' });
}
await ch.waitForConfirms();
await ch.close(); await conn.close();
```

**Consumer (Node):**
```js
await ch.prefetch(20);
ch.consume('invoice.gen', async (msg) => {
    try {
        const data = JSON.parse(msg.content);
        await generateInvoice(data.orderId);
        ch.ack(msg);
    } catch (e) {
        ch.nack(msg, false, false);
    }
}, { noAck: false });
```

### 2. Pub/Sub via Fanout (Daraz price-change)

```js
await ch.assertExchange('price.changes', 'fanout', { durable: true });
ch.publish('price.changes', '', Buffer.from(JSON.stringify({ sku, price })), { persistent: true });

// Each subscriber declares its own queue and binds
const { queue } = await ch.assertQueue('cache.invalidate', { durable: true });
await ch.bindQueue(queue, 'price.changes', '');
```

### 3. Direct Routing — Region-wise

```js
await ch.assertExchange('orders', 'direct', { durable: true });
await ch.assertQueue('orders.bd');
await ch.bindQueue('orders.bd', 'orders', 'order.bd');

ch.publish('orders', 'order.bd', body, { persistent: true });
```

### 4. Topic — Pathao city notification

```js
await ch.assertExchange('rides', 'topic', { durable: true });

// Dhaka SMS service
await ch.assertQueue('sms.dhaka');
await ch.bindQueue('sms.dhaka', 'rides', 'ride.dhaka.*');

// All cities push
await ch.assertQueue('push.all');
await ch.bindQueue('push.all', 'rides', 'ride.#');

ch.publish('rides', 'ride.dhaka.completed', Buffer.from('...'));
```

### 5. RPC Pattern — Sync over Async

**Server:**
```js
await ch.assertQueue('rpc.fxRate', { durable: false });
await ch.prefetch(1);
ch.consume('rpc.fxRate', async (msg) => {
    const { from, to } = JSON.parse(msg.content);
    const rate = await fetchFx(from, to);
    ch.sendToQueue(msg.properties.replyTo, Buffer.from(JSON.stringify({ rate })),
        { correlationId: msg.properties.correlationId });
    ch.ack(msg);
});
```

**Client:**
```js
const { queue: replyQ } = await ch.assertQueue('', { exclusive: true });
const corr = crypto.randomUUID();
const promise = new Promise((res) => {
    ch.consume(replyQ, (msg) => {
        if (msg.properties.correlationId === corr) {
            res(JSON.parse(msg.content));
        }
    }, { noAck: true });
});
ch.sendToQueue('rpc.fxRate', Buffer.from(JSON.stringify({ from: 'USD', to: 'BDT' })),
    { correlationId: corr, replyTo: replyQ });
const { rate } = await promise;
```

> ⚠️ RPC pattern সাবধান — sync expectation distributed system-এ failure mode বাড়ায়। Timeout, circuit breaker দরকার।

### 6. Saga Choreography via Topic

```
orders.exchange (topic)
  ├─routing_key="order.created"      → payment.svc subscribes
  ├─routing_key="payment.completed"  → inventory.svc subscribes
  ├─routing_key="payment.failed"     → order.svc compensate
  ├─routing_key="inventory.reserved" → shipping.svc subscribes
  └─routing_key="inventory.failed"   → payment.svc refund
```

> 🔗 Saga compensation logic বিস্তারিত: `/12-concurrency/saga-pattern.md`।

### 7. Retry Ladder + DLX

```js
// Three retry queues with TTLs
for (const [name, ttl] of [['retry.1m', 60000], ['retry.5m', 300000], ['retry.30m', 1800000]]) {
    await ch.assertQueue(name, {
        durable: true,
        arguments: {
            'x-message-ttl': ttl,
            'x-dead-letter-exchange': 'orders',
            'x-dead-letter-routing-key': 'order.bd',   // back to main
        },
    });
}
await ch.assertQueue('orders.parking', { durable: true });

// In consumer on failure:
const attempts = (msg.properties.headers?.['x-retry-count'] ?? 0) + 1;
if (attempts > 3) {
    ch.sendToQueue('orders.parking', msg.content, msg.properties);
} else {
    const target = ['retry.1m', 'retry.5m', 'retry.30m'][attempts - 1];
    ch.sendToQueue(target, msg.content, {
        ...msg.properties,
        headers: { ...msg.properties.headers, 'x-retry-count': attempts },
    });
}
ch.ack(msg);
```

### 8. Delayed Messages (plugin)

```js
await ch.assertExchange('schedule', 'x-delayed-message', {
    durable: true, arguments: { 'x-delayed-type': 'direct' },
});
ch.publish('schedule', 'reminder', body, {
    headers: { 'x-delay': 3600000 },   // 1 hour
});
```

**Use case:** Foodpanda — delivery feedback prompt 30 minutes after delivery।

### 9. Outbox Pattern Integration

```
DB transaction:
  1. INSERT order
  2. INSERT outbox event
  COMMIT

Outbox poller / Debezium:
  reads outbox table → publishes to RabbitMQ
  marks event as published
```

এতে DB commit + message publish একসাথে atomic হয় (without distributed transaction)।

### 10. Priority Queue — bKash VIP

```js
await ch.assertQueue('payments', {
    durable: true, arguments: { 'x-max-priority': 10 },   // classic queue
});
ch.sendToQueue('payments', body, { persistent: true, priority: 9 });    // VIP merchant
ch.sendToQueue('payments', body, { persistent: true, priority: 1 });    // regular
```

---

## ⚖️ RabbitMQ vs Kafka vs Redis Streams vs SQS/SNS

| Aspect | **RabbitMQ** | **Kafka** | **Redis Streams** | **AWS SQS/SNS** |
|--------|--------------|-----------|-------------------|------------------|
| **Model** | Smart broker, push | Distributed log, pull | In-memory log, pull | Managed queue/topic |
| **Throughput per partition** | 10-50K msg/s | 100K-1M msg/s | 50-100K msg/s | ~3K (standard), 300 FIFO |
| **Latency** | 1-10 ms | 5-20 ms | <1 ms | 10-100 ms |
| **Ordering** | Per-queue (1 consumer) | Per-partition | Per-stream | FIFO queue only |
| **Persistence** | Disk + replication | Disk + replication | RAM + AOF | Managed, durable |
| **Retention** | Until consumed | Configurable (days/forever) | XTRIM-bound | 14 days max |
| **Replay** | ❌ (Streams plugin yes) | ✅ Native | ✅ Native | ❌ |
| **Fan-out** | Exchange bindings | Multiple consumer groups | Multiple groups | SNS → multiple SQS |
| **Routing logic** | Rich (direct/topic/headers) | Topic + key partitioning | Single stream | Topic + filter policies |
| **Push vs Pull** | Push (consumer subscribes) | Pull | Pull (XREAD) | Pull (long poll) |
| **Exactly-once** | At-least-once + idempotency | Idempotent producer + EOS | At-least-once + idemp. | At-least-once (FIFO with dedup) |
| **Ops complexity** | Medium | High | Low (if Redis already) | Zero (managed) |
| **Ecosystem** | Plugins (MQTT, STOMP) | Connect, Streams, ksqlDB | Modules | AWS-only |
| **Best for** | Task queues, RPC, complex routing | Event streaming, big data | Lightweight stream, low latency | Cloud-native simple queue |

### Decision Matrix

```
┌──────────────────────────────────────────────────────────────────┐
│  Need                              │  Pick                        │
├──────────────────────────────────────────────────────────────────┤
│  Replay last 7 days for analytics  │  Kafka                       │
│  Per-message routing & priorities  │  RabbitMQ                    │
│  Existing Redis + small scale      │  Redis Streams               │
│  Ordered task queue per tenant     │  RabbitMQ (1 queue/tenant)   │
│  Microservice events <50K msg/s    │  RabbitMQ (topic exchange)   │
│  Microservice events >100K msg/s   │  Kafka                       │
│  Mobile push fan-out (millions)    │  Kafka or SNS                │
│  RPC over async                    │  RabbitMQ (reply_to)         │
│  Cloud-native, no ops              │  SQS/SNS or CloudAMQP        │
│  CDC / event sourcing              │  Kafka + Debezium            │
└──────────────────────────────────────────────────────────────────┘
```

> 🔗 আরও তুলনা: `/03-system-design/message-queue.md`, `/18-kafka/README.md`, `/05-database/redis-deep-dive.md` (Streams section)।

---

## 🔄 Migration

### Kafka → RabbitMQ (rare)
- সাধারণত যখন routing complexity বাড়ে এবং throughput কম।
- Strategy: dual-publish বা MirrorMaker → Shovel।

### RabbitMQ → Kafka (common at scale)
- Reason: replay, retention, throughput beyond 100K msg/s।
- Strategy:
  1. Kafka cluster দাঁড় করানো।
  2. Shovel/dual-publish — RabbitMQ exchange-এর সব message Kafka topic-এ mirror।
  3. Consumers একে একে Kafka-এ migrate।
  4. Old RabbitMQ producer cut।
- Pathao একসময় RabbitMQ-এ ছিল; trip event volume বাড়ার সাথে Kafka-এ migrate করেছে event sourcing-এর জন্য।

---

## 🇧🇩 Bangladesh Case Studies

### bKash — Payment Notification Fan-out (Topic Exchange)
```
Topic: notifications.exchange
  ├─"payment.bd.bkash.success" → SMS queue (immediate)
  ├─"payment.bd.bkash.success" → Push queue
  ├─"payment.bd.bkash.success" → Email queue (priority for merchants)
  ├─"payment.bd.#"             → Audit/compliance queue
  └─"payment.#.failed"         → Alerting queue (PagerDuty bridge)
```
- Quorum queues, persistent messages, AOF।
- Idempotency via `transaction_id` in Redis SET NX (24h TTL)।

### Daraz — Seller Portal Order Events with Priority
- VIP sellers (top 1%) — `priority: 9`; regular `priority: 0`।
- Single queue with `x-max-priority: 10` (classic), prefetch 50।
- Order processing SLA: VIP <30s, regular <5min।

### Pathao Foods — Order Lifecycle Saga (Direct Exchange + DLX)
```
order.created    → kitchen.queue
kitchen.accepted → rider.assignment.queue
rider.picked     → eta.compute.queue
order.delivered  → settlement.queue + feedback.delayed (1hr)

Failure: any → compensate.exchange → respective rollback queue
DLX: orders.dlx → orders.parking → manual ops review
```

### Foodpanda — Invoice Generation Work Queue
- Lazy queue (millions of monthly invoices), `x-queue-mode: lazy`।
- 50 consumer instances, `prefetch: 5` each।
- DLX retry ladder 1m → 5m → 30m → parking।
- Off-hour batch generation reduces DB pressure।

### Small BD Startup — CloudAMQP LittleLemur (Free)
- 1M messages/month free tier।
- 20 connections, 100 queues — যথেষ্ট MVP-র জন্য।
- HTTPS API + management UI সহ — DevOps team লাগে না।
- Outgrow করলে → upgrade Tough Tiger বা self-host।

---

## 🚫 Anti-patterns

```
┌────────────────────────────────────────────────────────────────┐
│ ❌ RabbitMQ for stream replay/analytics                          │
│    → Use Kafka; RabbitMQ deletes after ack                      │
│ ❌ No DLX → poison messages requeue forever (CPU loop)          │
│ ❌ No prefetch → consumer overwhelmed, RAM blowup                │
│ ❌ Mirrored queues at scale → use quorum                        │
│ ❌ Sharing connection across threads → race, frame interleave   │
│ ❌ AutoAck (noAck=true) for important messages → message loss   │
│ ❌ Mixing RPC + work queues on same connection → starvation     │
│ ❌ Polling get_one in loop → kills broker (push + consume)      │
│ ❌ Unbounded queue length → broker OOM under producer surge     │
│ ❌ No publisher confirms → silent message drops                 │
│ ❌ Putting business logic in routing keys → hard to evolve      │
│ ❌ One huge vhost → noisy neighbour problem                     │
│ ❌ Default guest:guest in prod → SECURITY BREACH                │
│ ❌ Forgetting heartbeat → idle connections dropped, surprises   │
└────────────────────────────────────────────────────────────────┘
```

---

## 📝 Summary

RabbitMQ = mature AMQP broker, smart routing (direct/topic/fanout/headers), strong reliability primitives (confirms, ack, DLX, retry, prefetch), HA via quorum queues + Erlang clustering, federation/shovel for multi-DC। Kafka-র চেয়ে কম throughput কিন্তু operational simplicity ও routing flexibility বেশি। Microservice task queue, RPC, notification fan-out — সব default pick হিসেবে ভালো।

---

## ✅ Production Checklist

- [ ] **Quorum queues** as default (not classic mirrored)।
- [ ] **Persistent messages** (`delivery_mode=2`) on durable queues।
- [ ] **Publisher confirms** enabled in producer code।
- [ ] **Manual ack** in consumers (no autoAck)।
- [ ] **Prefetch** tuned per consumer (start 10-20)।
- [ ] **DLX** configured on every business queue।
- [ ] **Retry ladder** (TTL queues) before parking lot।
- [ ] **Idempotency** in consumers (message_id dedup in Redis/DB)।
- [ ] **Connection per process**, channel per thread/coroutine।
- [ ] **Heartbeat** 30-60s; client reconnect logic।
- [ ] **Cluster ≥3 nodes**, `cluster_partition_handling=pause_minority`।
- [ ] **TLS (5671)**, plaintext disabled in prod।
- [ ] **Default `guest`** user removed; per-app users with vhost-scoped regex।
- [ ] **Memory watermark** ≤60%, paging_ratio 0.5।
- [ ] **Disk free limit** ≥5GB।
- [ ] **Max connections / channels** caps set।
- [ ] **Lazy queues** for huge backlogs (>100K messages)।
- [ ] **Priority queue** only when really needed (classic only)।
- [ ] **Single Active Consumer** for ordered processing per queue।
- [ ] **Streams plugin** for replay use cases (or migrate to Kafka)।
- [ ] **Delayed messages plugin** for scheduled tasks।
- [ ] **Federation/Shovel** for multi-DC strategy chosen and tested।
- [ ] **Backup:** definitions export weekly (`rabbitmqctl export_definitions`)।
- [ ] **Monitoring:** `rabbitmq_prometheus` + Grafana (dashboard 10991)।
- [ ] **Alerts:** queue depth, consumer count, unacked, memory, disk, partition state, churn rate।
- [ ] **Health checks:** `rabbitmq-diagnostics check_running` + custom DLQ depth।
- [ ] **Load test** (PerfTest tool) before prod scale-up।
- [ ] **Schema** per message: `content_type`, `message_id`, `correlation_id`, `traceId` header।
- [ ] **Versioning** in routing keys or headers (e.g., `order.v2.created`)।
- [ ] **Outbox pattern** for DB-broker consistency।
- [ ] **Circuit breaker** in producers to broker outage tolerance।
- [ ] **Graceful shutdown** in consumers — finish current message, then `cancel`।
- [ ] **Plugins audit:** only enable what's used (attack surface)।
- [ ] **Erlang cookie** secure (`chmod 400`); not committed to git।
- [ ] **Resource policies** per vhost (max-length, max-bytes, ttl)।
- [ ] **Documentation:** queue-owner team mapping, runbook for poison messages।
- [ ] **DR drill:** quarterly broker failover test।
- [ ] **Migration plan** documented if scaling toward Kafka territory।
- [ ] **Cost monitoring** — managed (CloudAMQP) vs self-host TCO review।

---

> 🔗 সংশ্লিষ্ট ফাইল:
> - `/03-system-design/message-queue.md` — broker landscape।
> - `/18-kafka/kafka-fundamentals.md`, `/18-kafka/README.md` — Kafka comparison।
> - `/05-database/redis-deep-dive.md` — Redis Streams alternative।
> - `/12-concurrency/saga-pattern.md` — saga choreography।
> - `/12-concurrency/backpressure.md` — prefetch tuning theory।
