# 📨 মেসেজ কিউ (Message Queue) — সম্পূর্ণ গভীর বিশ্লেষণ

## 📌 সংজ্ঞা ও মূল ধারণা

**মেসেজ কিউ** হলো একটি **asynchronous communication mechanism** যেখানে একটি সিস্টেমের বিভিন্ন কম্পোনেন্ট একে অপরের সাথে সরাসরি কথা না বলে একটি মধ্যবর্তী **broker** এর মাধ্যমে মেসেজ আদান-প্রদান করে। এটি সিস্টেম ডিজাইনের সবচেয়ে শক্তিশালী প্যাটার্নগুলোর একটি।

### Message vs Event — পার্থক্য বোঝা জরুরি

| বৈশিষ্ট্য | Message | Event |
|-----------|---------|-------|
| উদ্দেশ্য | কাউকে কিছু করতে বলা (command) | কিছু ঘটেছে জানানো (notification) |
| Consumer | নির্দিষ্ট একজন | যে কেউ subscribe করতে পারে |
| উদাহরণ | "এই অর্ডারটি প্রসেস করো" | "অর্ডার #123 সফল হয়েছে" |
| Coupling | কিছুটা coupled | সম্পূর্ণ decoupled |

### Synchronous vs Asynchronous Communication

```
Synchronous (সরাসরি):
  User → [Order Service] ──HTTP──▶ [Payment Service] ──HTTP──▶ [Notification]
         ⏳ ব্লক হয়ে অপেক্ষা   ⏳ ব্লক হয়ে অপেক্ষা    ⏳ ধীরগতি

Asynchronous (কিউ দিয়ে):
  User → [Order Service] ──▶ |  QUEUE  | ──▶ [Payment Service]
         ✅ তাৎক্ষণিক রেসপন্স  |_________|     (নিজের সময়ে প্রসেস)
```

### Decoupling কেন দরকার?

বাস্তব উদাহরণ — **bKash লেনদেন সিস্টেম**: যখন কেউ Send Money করে, তখন একসাথে SMS পাঠানো, ব্যালেন্স আপডেট, ট্রানজ্যাকশন লগ — সব synchronous করলে API টাইমআউট হবে। মেসেজ কিউ দিয়ে মূল লেনদেন দ্রুত সম্পন্ন হয়, বাকি কাজ queue-তে যায়।

---

## 📊 Queue Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              MESSAGE BROKER                  │
                        │                                             │
  ┌──────────┐         │  ┌─────────┐    ┌──────────────────────┐   │         ┌──────────┐
  │ Producer │────▶    │  │Exchange/│───▶│    Queue / Topic     │   │   ──▶  │ Consumer │
  │ (App A)  │  publish│  │ Router  │    │ ┌──┬──┬──┬──┬──┬──┐ │   │consume  │ (App B)  │
  └──────────┘         │  └─────────┘    │ │M1│M2│M3│M4│M5│M6│ │   │         └──────────┘
                        │                 │ └──┴──┴──┴──┴──┴──┘ │   │
  ┌──────────┐         │  ┌─────────┐    └──────────────────────┘   │         ┌──────────┐
  │ Producer │────▶    │  │  Dead   │    ┌──────────────────────┐   │   ──▶  │ Consumer │
  │ (App C)  │         │  │ Letter  │◀───│   Failed Messages    │   │         │ (App D)  │
  └──────────┘         │  │  Queue  │    └──────────────────────┘   │         └──────────┘
                        │  └─────────┘                               │
                        └─────────────────────────────────────────────┘

  মেসেজ ফ্লো:
  ─────────────────────────────────────────────────────────────
  ১. Producer মেসেজ তৈরি করে Broker-এ পাঠায়
  ২. Broker মেসেজ সংরক্ষণ করে ও routing সিদ্ধান্ত নেয়
  ৩. Consumer মেসেজ গ্রহণ করে ও প্রসেস করে
  ৪. সফল হলে ACK পাঠায়, ব্যর্থ হলে DLQ-তে যায়
```

---

## 💻 Queue Technologies Deep Dive

### ১. RabbitMQ — AMQP ভিত্তিক মেসেজ ব্রোকার

RabbitMQ হলো সবচেয়ে জনপ্রিয় **traditional message broker**। এটি AMQP (Advanced Message Queuing Protocol) ব্যবহার করে এবং **smart broker, dumb consumer** মডেল অনুসরণ করে — অর্থাৎ broker নিজেই সিদ্ধান্ত নেয় কোন মেসেজ কোথায় যাবে।

#### Exchange Types — মেসেজ রাউটিংয়ের হৃদপিণ্ড

```
┌─────────────────────────────────────────────────────────────────┐
│                    RabbitMQ Exchange Types                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① Direct Exchange         ② Fanout Exchange                   │
│  ─────────────────         ──────────────────                   │
│  routing_key="sms"         সব queue-তে broadcast               │
│                                                                 │
│  Producer ──▶ [Exchange]   Producer ──▶ [Exchange]              │
│                  │                       │ │ │                   │
│           key="sms"               ┌──────┘ │ └──────┐           │
│                  │                ▼        ▼        ▼           │
│                  ▼           [Queue A] [Queue B] [Queue C]      │
│             [SMS Queue]                                         │
│                                                                 │
│  ③ Topic Exchange          ④ Headers Exchange                  │
│  ─────────────────         ───────────────────                  │
│  pattern matching          header attributes দিয়ে match        │
│                                                                 │
│  key="order.*.bd"          x-match: all/any                     │
│  ──▶ matches:              headers: {region:"bd", type:"sms"}   │
│    order.created.bd ✅                                          │
│    order.shipped.bd ✅                                          │
│    order.created.us ❌                                          │
└─────────────────────────────────────────────────────────────────┘
```

#### PHP (php-amqplib) — RabbitMQ Producer ও Consumer

```php
<?php
// Producer — bKash ট্রানজ্যাকশন নোটিফিকেশন পাঠানো
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
use PhpAmqpLib\Exchange\AMQPExchangeType;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

// Dead Letter Exchange সেটআপ
$channel->exchange_declare('dlx.notifications', AMQPExchangeType::DIRECT, false, true, false);
$channel->queue_declare('dlq.notifications', false, true, false, false);
$channel->queue_bind('dlq.notifications', 'dlx.notifications', 'failed');

// মূল exchange ও queue সেটআপ (DLQ সংযুক্ত)
$channel->exchange_declare('ex.notifications', AMQPExchangeType::TOPIC, false, true, false);
$channel->queue_declare('q.sms', false, true, false, false, false, [
    'x-dead-letter-exchange'    => ['S', 'dlx.notifications'],
    'x-dead-letter-routing-key' => ['S', 'failed'],
    'x-message-ttl'             => ['I', 60000], // ৬০ সেকেন্ড TTL
]);
$channel->queue_bind('q.sms', 'ex.notifications', 'notification.sms.*');

// মেসেজ পাঠানো
$payload = json_encode([
    'transaction_id' => 'TXN_' . uniqid(),
    'phone'          => '+8801712345678',
    'amount'         => 5000,
    'type'           => 'send_money',
    'timestamp'      => time(),
]);

$message = new AMQPMessage($payload, [
    'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
    'content_type'  => 'application/json',
    'message_id'    => uniqid('msg_'),
    'timestamp'     => time(),
    'expiration'    => '300000', // ৫ মিনিট TTL
]);

$channel->basic_publish($message, 'ex.notifications', 'notification.sms.bd');
echo "মেসেজ পাঠানো হয়েছে!\n";

$channel->close();
$connection->close();
```

```php
<?php
// Consumer — Prefetch ও ACK/NACK সহ
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

// Prefetch — একবারে সর্বোচ্চ ১০টি মেসেজ নেবে (backpressure নিয়ন্ত্রণ)
$channel->basic_qos(null, 10, null);

$callback = function ($msg) {
    $data = json_decode($msg->body, true);
    echo "প্রসেস হচ্ছে: TXN {$data['transaction_id']}\n";

    try {
        sendSms($data['phone'], "৳{$data['amount']} Send Money সফল হয়েছে।");
        // সফল — ACK পাঠাও, broker থেকে মেসেজ মুছে যাবে
        $msg->ack();
    } catch (\Exception $e) {
        // ব্যর্থ — NACK পাঠাও, requeue=false মানে DLQ-তে যাবে
        $msg->nack(false);
        error_log("SMS ব্যর্থ: {$e->getMessage()}");
    }
};

// Consumer শুরু — no_ack=false মানে manual acknowledgement
$channel->basic_consume('q.sms', '', false, false, false, false, $callback);

echo "Consumer চালু আছে। বন্ধ করতে Ctrl+C\n";
while ($channel->is_open()) {
    $channel->wait();
}
```

#### JavaScript (amqplib) — Node.js RabbitMQ

```javascript
// producer.js — অর্ডার প্রসেসিং সিস্টেম
const amqp = require('amqplib');

async function setupAndPublish() {
    const conn = await amqp.connect('amqp://localhost');
    const channel = await conn.createChannel();

    // Exchange ও Queue সেটআপ
    await channel.assertExchange('ex.orders', 'topic', { durable: true });
    await channel.assertQueue('q.order.processing', {
        durable: true,
        arguments: {
            'x-dead-letter-exchange': 'dlx.orders',
            'x-dead-letter-routing-key': 'order.failed',
            'x-max-length': 100000,        // সর্বোচ্চ ১ লক্ষ মেসেজ
            'x-overflow': 'reject-publish', // পূর্ণ হলে নতুন মেসেজ প্রত্যাখ্যান
        }
    });
    await channel.bindQueue('q.order.processing', 'ex.orders', 'order.new.#');

    const order = {
        orderId: `ORD_${Date.now()}`,
        customer: { phone: '+8801812345678', name: 'রহিম উদ্দিন' },
        items: [{ product: 'Smartphone', qty: 1, price: 25000 }],
        totalAmount: 25000,
        createdAt: new Date().toISOString(),
    };

    channel.publish('ex.orders', 'order.new.dhaka', Buffer.from(JSON.stringify(order)), {
        persistent: true,
        messageId: order.orderId,
        contentType: 'application/json',
        headers: { 'x-retry-count': 0, 'x-source': 'web-app' },
    });

    console.log(`অর্ডার পাঠানো হয়েছে: ${order.orderId}`);
    setTimeout(() => conn.close(), 500);
}

setupAndPublish().catch(console.error);
```

```javascript
// consumer.js — Retry logic সহ
const amqp = require('amqplib');

const MAX_RETRIES = 3;

async function startConsumer() {
    const conn = await amqp.connect('amqp://localhost');
    const channel = await conn.createChannel();

    await channel.prefetch(5); // একবারে ৫টি মেসেজ

    await channel.consume('q.order.processing', async (msg) => {
        if (!msg) return;
        const order = JSON.parse(msg.content.toString());
        const retryCount = msg.properties.headers['x-retry-count'] || 0;

        try {
            console.log(`প্রসেস হচ্ছে: ${order.orderId}`);
            await processOrder(order);
            channel.ack(msg);
            console.log(`সফল: ${order.orderId}`);
        } catch (err) {
            console.error(`ব্যর্থ (চেষ্টা ${retryCount + 1}/${MAX_RETRIES}): ${err.message}`);
            if (retryCount < MAX_RETRIES) {
                // পুনরায় চেষ্টার জন্য নতুন মেসেজ তৈরি (delay সহ)
                channel.publish('ex.orders', 'order.new.retry', msg.content, {
                    persistent: true,
                    headers: { ...msg.properties.headers, 'x-retry-count': retryCount + 1 },
                    expiration: String(Math.pow(2, retryCount) * 1000), // exponential backoff
                });
                channel.ack(msg);
            } else {
                channel.nack(msg, false, false); // DLQ-তে পাঠাও
            }
        }
    }, { noAck: false });

    console.log('অর্ডার Consumer চালু আছে...');
}

async function processOrder(order) {
    // অর্ডার প্রসেসিং লজিক
    if (Math.random() < 0.1) throw new Error('পেমেন্ট গেটওয়ে সমস্যা');
}

startConsumer().catch(console.error);
```

---

### ২. Apache Kafka — Distributed Event Streaming Platform

Kafka একটি **distributed commit log** — এটি traditional queue নয়। মূল পার্থক্য হলো Kafka মেসেজ **consume করার পরেও মুছে ফেলে না**, বরং retention period পর্যন্ত রেখে দেয়। এটি **dumb broker, smart consumer** মডেল — consumer নিজেই ট্র্যাক রাখে কতদূর পড়েছে (offset)।

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA ARCHITECTURE                            │
│                                                                 │
│  Topic: "bkash-transactions"                                    │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ Partition 0: [msg0|msg1|msg2|msg3|msg4|msg5|...]     │       │
│  │              ▲ offset=3 (Consumer Group A)           │       │
│  │              ▲ offset=1 (Consumer Group B)           │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │ Partition 1: [msg0|msg1|msg2|msg3|...]               │       │
│  ├──────────────────────────────────────────────────────┤       │
│  │ Partition 2: [msg0|msg1|msg2|...]                    │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  Consumer Group A              Consumer Group B                 │
│  ┌────────────┐                ┌────────────┐                   │
│  │Consumer A1 │◀── P0          │Consumer B1 │◀── P0, P1        │
│  │Consumer A2 │◀── P1          │Consumer B2 │◀── P2             │
│  │Consumer A3 │◀── P2          └────────────┘                   │
│  └────────────┘                                                 │
│                                                                 │
│  * প্রতিটি partition একটি consumer group-এ একজনই পড়ে           │
│  * ভিন্ন group স্বাধীনভাবে পড়তে পারে                           │
│  * Partition সংখ্যা = সর্বোচ্চ parallelism                      │
└─────────────────────────────────────────────────────────────────┘
```

#### PHP (rdkafka) — Kafka Producer ও Consumer

```php
<?php
// Kafka Producer — bKash ট্রানজ্যাকশন ইভেন্ট
$config = new RdKafka\Conf();
$config->set('metadata.broker.list', 'kafka-broker1:9092,kafka-broker2:9092');
$config->set('acks', 'all');                    // সব replica-তে লেখা নিশ্চিত
$config->set('enable.idempotence', 'true');      // exactly-once producer
$config->set('max.in.flight.requests.per.connection', '5');
$config->set('retries', '2147483647');           // অসীম retry
$config->set('compression.type', 'snappy');      // ডেটা কমপ্রেশন

$producer = new RdKafka\Producer($config);

$topic = $producer->newTopic('bkash-transactions', (new RdKafka\TopicConf()));

$transaction = [
    'txn_id'     => 'TXN_20240115_001',
    'sender'     => '+8801712345678',
    'receiver'   => '+8801898765432',
    'amount'     => 1500.00,
    'type'       => 'send_money',
    'timestamp'  => microtime(true),
    'region'     => 'dhaka',
];

// partition key হিসেবে sender নম্বর ব্যবহার
// একই sender-এর সব ট্রানজ্যাকশন একই partition-এ যাবে → ordering গ্যারান্টি
$topic->produce(
    RD_KAFKA_PARTITION_UA,       // auto partition assignment
    0,
    json_encode($transaction),
    $transaction['sender']       // partition key
);

$producer->flush(5000); // ৫ সেকেন্ডের মধ্যে সব মেসেজ পাঠাও
echo "Kafka-তে ইভেন্ট পাঠানো হয়েছে\n";
```

```php
<?php
// Kafka Consumer — Consumer Group সহ
$config = new RdKafka\Conf();
$config->set('metadata.broker.list', 'kafka-broker1:9092');
$config->set('group.id', 'transaction-processor-group');
$config->set('auto.offset.reset', 'earliest');
$config->set('enable.auto.commit', 'false'); // manual commit (exactly-once এর জন্য)

$config->setRebalanceCb(function (RdKafka\KafkaConsumer $consumer, $err, array $partitions = null) {
    switch ($err) {
        case RD_KAFKA_RESP_ERR__ASSIGN_PARTITIONS:
            echo "নতুন partition বরাদ্দ হয়েছে\n";
            $consumer->assign($partitions);
            break;
        case RD_KAFKA_RESP_ERR__REVOKE_PARTITIONS:
            echo "Partition প্রত্যাহার হয়েছে\n";
            $consumer->assign(null);
            break;
    }
});

$consumer = new RdKafka\KafkaConsumer($config);
$consumer->subscribe(['bkash-transactions']);

echo "Kafka consumer চালু...\n";
while (true) {
    $message = $consumer->consume(10000); // ১০ সেকেন্ড অপেক্ষা
    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            $data = json_decode($message->payload, true);
            echo "ট্রানজ্যাকশন প্রসেস: {$data['txn_id']} — ৳{$data['amount']}\n";

            processTransaction($data);
            $consumer->commit($message); // offset commit — এই পর্যন্ত পড়া হয়েছে
            break;

        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            // এই partition-এ আর মেসেজ নেই
            break;
        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            break;
        default:
            error_log("Kafka error: {$message->errstr()}");
    }
}
```

#### JavaScript (kafkajs) — Node.js Kafka

```javascript
// kafka-producer.js — Exactly-once semantics সহ
const { Kafka, Partitioners, CompressionTypes } = require('kafkajs');

const kafka = new Kafka({
    clientId: 'order-service',
    brokers: ['kafka1:9092', 'kafka2:9092'],
    retry: { initialRetryTime: 100, retries: 8 },
});

const producer = kafka.producer({
    createPartitioner: Partitioners.DefaultPartitioner,
    idempotent: true,          // exactly-once producer guarantee
    maxInFlightRequests: 5,
    transactionalId: 'order-producer-txn-01', // transactional producer
});

async function publishOrderEvents(orders) {
    await producer.connect();
    const transaction = await producer.transaction();

    try {
        // একই transaction-এ একাধিক topic-এ লেখা
        await transaction.send({
            topic: 'order-events',
            compression: CompressionTypes.Snappy,
            messages: orders.map(order => ({
                key: order.customerId,    // partitioning key
                value: JSON.stringify(order),
                headers: {
                    'event-type': 'ORDER_CREATED',
                    'source': 'order-service',
                    'correlation-id': order.orderId,
                },
                timestamp: Date.now().toString(),
            })),
        });

        await transaction.send({
            topic: 'analytics-events',
            messages: orders.map(o => ({
                key: o.orderId,
                value: JSON.stringify({ orderId: o.orderId, amount: o.total }),
            })),
        });

        await transaction.commit();
        console.log(`${orders.length}টি অর্ডার ইভেন্ট পাঠানো হয়েছে (atomic)`);
    } catch (err) {
        await transaction.abort();
        console.error('Transaction ব্যর্থ:', err.message);
    }
}

// ব্যবহার
publishOrderEvents([
    { orderId: 'ORD001', customerId: 'CUST_BD_01', total: 15000, items: ['Laptop Cover'] },
]);
```

```javascript
// kafka-consumer.js — Consumer Group, offset management
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ clientId: 'notification-service', brokers: ['kafka1:9092'] });
const consumer = kafka.consumer({ groupId: 'notification-group' });

async function startConsumer() {
    await consumer.connect();
    await consumer.subscribe({ topics: ['order-events'], fromBeginning: false });

    await consumer.run({
        partitionsConsumedConcurrently: 3, // ৩টি partition একসাথে প্রসেস
        eachMessage: async ({ topic, partition, message, heartbeat }) => {
            const order = JSON.parse(message.value.toString());
            const eventType = message.headers['event-type']?.toString();

            console.log(`[P${partition}] ${eventType}: ${order.orderId}`);

            await sendNotification(order);
            await heartbeat(); // দীর্ঘ প্রসেসিংয়ে session timeout এড়াতে
        },
    });
}

// Graceful shutdown
['SIGTERM', 'SIGINT'].forEach(signal => {
    process.on(signal, async () => {
        console.log('Consumer বন্ধ হচ্ছে...');
        await consumer.disconnect();
        process.exit(0);
    });
});

startConsumer().catch(console.error);
```

---

### ৩. AWS SQS/SNS — Managed Queue Service

AWS SQS হলো fully managed queue service — কোনো সার্ভার ম্যানেজ করতে হয় না। **SNS (Simple Notification Service)** হলো pub/sub সিস্টেম যা SQS-এর সাথে মিলে **fan-out pattern** তৈরি করে।

```
┌─────────────────────────────────────────────────────────┐
│            SNS + SQS Fan-out Pattern                     │
│                                                         │
│                    ┌──────────┐                          │
│                    │ SNS Topic│                          │
│  Order Service ──▶ │ "orders" │                          │
│                    └────┬─────┘                          │
│               ┌─────────┼─────────┐                     │
│               ▼         ▼         ▼                     │
│         ┌─────────┐ ┌────────┐ ┌──────────┐            │
│         │SQS:     │ │SQS:    │ │SQS:      │            │
│         │inventory│ │billing │ │analytics │            │
│         └────┬────┘ └───┬────┘ └─────┬────┘            │
│              ▼          ▼            ▼                  │
│         [Inventory] [Billing]  [Analytics]              │
│         [Service  ] [Service]  [Service  ]              │
│                                                         │
│  Standard Queue          vs       FIFO Queue            │
│  ─────────────                    ──────────            │
│  • সীমাহীন throughput             • ৩০০ msg/sec         │
│  • At-least-once delivery         • Exactly-once        │
│  • ক্রম গ্যারান্টি নেই            • FIFO গ্যারান্টি     │
│  • কদাচিৎ duplicate হয়           • deduplication আছে   │
└─────────────────────────────────────────────────────────┘
```

#### PHP (AWS SDK) — SQS

```php
<?php
use Aws\Sqs\SqsClient;

$sqs = new SqsClient([
    'region'  => 'ap-southeast-1',
    'version' => '2012-11-05',
]);

$queueUrl = 'https://sqs.ap-southeast-1.amazonaws.com/123456789/order-processing';

// মেসেজ পাঠানো (FIFO queue-তে)
$result = $sqs->sendMessage([
    'QueueUrl'               => $queueUrl . '.fifo',
    'MessageBody'            => json_encode([
        'orderId' => 'ORD_BD_001',
        'amount'  => 8500,
        'items'   => [['name' => 'চাল', 'qty' => 10, 'price' => 850]],
    ]),
    'MessageGroupId'         => 'order-group-dhaka',      // FIFO grouping
    'MessageDeduplicationId' => 'ORD_BD_001_' . time(),   // duplicate prevention
    'MessageAttributes'      => [
        'Priority' => ['DataType' => 'String', 'StringValue' => 'high'],
        'Region'   => ['DataType' => 'String', 'StringValue' => 'dhaka'],
    ],
]);

echo "মেসেজ পাঠানো হয়েছে — ID: {$result['MessageId']}\n";

// মেসেজ গ্রহণ (long polling সহ — খরচ কমায়)
while (true) {
    $result = $sqs->receiveMessage([
        'QueueUrl'            => $queueUrl,
        'MaxNumberOfMessages' => 10,
        'WaitTimeSeconds'     => 20,     // long polling — ২০ সেকেন্ড অপেক্ষা
        'VisibilityTimeout'   => 60,     // ৬০ সেকেন্ড প্রসেসিং সময়
        'MessageAttributeNames' => ['All'],
    ]);

    if (!empty($result['Messages'])) {
        foreach ($result['Messages'] as $message) {
            $body = json_decode($message['Body'], true);
            echo "অর্ডার প্রসেস: {$body['orderId']}\n";

            try {
                processOrder($body);
                // সফল — মেসেজ মুছে ফেলো
                $sqs->deleteMessage([
                    'QueueUrl'      => $queueUrl,
                    'ReceiptHandle' => $message['ReceiptHandle'],
                ]);
            } catch (\Exception $e) {
                // ব্যর্থ — visibility timeout শেষে আবার দেখা যাবে
                echo "ব্যর্থ: {$e->getMessage()}\n";
            }
        }
    }
}
```

#### JavaScript (aws-sdk v3) — SQS/SNS

```javascript
// sqs-sns.js — SNS fan-out ও SQS consumer
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');

const sns = new SNSClient({ region: 'ap-southeast-1' });
const sqs = new SQSClient({ region: 'ap-southeast-1' });

// SNS-এ ইভেন্ট পাবলিশ (fan-out)
async function publishOrderEvent(order) {
    await sns.send(new PublishCommand({
        TopicArn: 'arn:aws:sns:ap-southeast-1:123456789:order-events',
        Message: JSON.stringify(order),
        MessageAttributes: {
            eventType: { DataType: 'String', StringValue: 'ORDER_CREATED' },
            region: { DataType: 'String', StringValue: order.region },
        },
        // FIFO topic-এর জন্য
        MessageGroupId: order.customerId,
        MessageDeduplicationId: order.orderId,
    }));
    console.log(`SNS-এ পাবলিশ: ${order.orderId}`);
}

// SQS Consumer — graceful polling loop
async function pollMessages(queueUrl) {
    let isRunning = true;
    process.on('SIGTERM', () => { isRunning = false; });

    while (isRunning) {
        const { Messages = [] } = await sqs.send(new ReceiveMessageCommand({
            QueueUrl: queueUrl,
            MaxNumberOfMessages: 10,
            WaitTimeSeconds: 20,
            VisibilityTimeout: 120,
        }));

        await Promise.all(Messages.map(async (msg) => {
            const order = JSON.parse(msg.Body);
            console.log(`প্রসেস: ${order.orderId}`);

            await handleOrder(order);

            await sqs.send(new DeleteMessageCommand({
                QueueUrl: queueUrl,
                ReceiptHandle: msg.ReceiptHandle,
            }));
        }));
    }
}

pollMessages('https://sqs.ap-southeast-1.amazonaws.com/123456789/order-processing');
```

---

### ৪. Redis Streams — লাইটওয়েট ইভেন্ট স্ট্রিমিং

Redis Streams হলো Kafka-র মতো **append-only log** কিন্তু Redis-এর ভিতরে। ছোট-মাঝারি সিস্টেমে Kafka-র বিকল্প হিসেবে চমৎকার।

```
  XADD stream * field value     →  Stream-এ নতুন entry যোগ
  XREAD COUNT 10 STREAMS s 0    →  Stream থেকে পড়া
  XGROUP CREATE s group 0       →  Consumer Group তৈরি
  XREADGROUP GROUP g c COUNT 5  →  Group-এ consumer হিসেবে পড়া
  XACK stream group id          →  প্রসেস সম্পন্ন নিশ্চিত করা
```

#### PHP — Redis Streams (Predis)

```php
<?php
use Predis\Client;

$redis = new Client(['scheme' => 'tcp', 'host' => '127.0.0.1', 'port' => 6379]);

// Consumer Group তৈরি
try {
    $redis->xgroup('CREATE', 'notifications', 'sms-workers', '0', true);
} catch (\Exception $e) {
    // group আগে থেকে থাকলে সমস্যা নেই
}

// Producer — নোটিফিকেশন stream-এ push
$entryId = $redis->xadd('notifications', '*', [
    'type'    => 'sms',
    'phone'   => '+8801712345678',
    'message' => 'আপনার অর্ডার #ORD001 কনফার্ম হয়েছে',
    'priority' => 'high',
]);
echo "Entry ID: {$entryId}\n";

// Consumer — Group-ভিত্তিক consuming
while (true) {
    $messages = $redis->xreadgroup(
        'sms-workers',         // group name
        'worker-1',            // consumer name
        ['notifications' => '>'], // '>' = শুধু নতুন মেসেজ
        5,                     // সর্বোচ্চ ৫টি
        2000                   // ২ সেকেন্ড block
    );

    if ($messages) {
        foreach ($messages['notifications'] as $id => $data) {
            echo "SMS পাঠানো হচ্ছে: {$data['phone']}\n";

            try {
                sendSms($data['phone'], $data['message']);
                $redis->xack('notifications', 'sms-workers', $id);
            } catch (\Exception $e) {
                // ACK না করলে pending list-এ থাকবে, পরে অন্য worker নিতে পারবে
                echo "ব্যর্থ: {$e->getMessage()}\n";
            }
        }
    }
}
```

#### JavaScript — Redis Streams (ioredis)

```javascript
const Redis = require('ioredis');
const redis = new Redis();

// Consumer Group তৈরি ও consuming
async function startStreamConsumer() {
    try {
        await redis.xgroup('CREATE', 'notifications', 'email-workers', '0', 'MKSTREAM');
    } catch (e) { /* group exists */ }

    while (true) {
        const results = await redis.xreadgroup(
            'GROUP', 'email-workers', 'worker-1',
            'COUNT', 10,
            'BLOCK', 2000,
            'STREAMS', 'notifications', '>'
        );

        if (results) {
            for (const [stream, messages] of results) {
                for (const [id, fields] of messages) {
                    const data = {};
                    for (let i = 0; i < fields.length; i += 2) {
                        data[fields[i]] = fields[i + 1];
                    }
                    console.log(`ইমেইল পাঠানো হচ্ছে: ${data.phone}`);
                    await redis.xack('notifications', 'email-workers', id);
                }
            }
        }
    }
}

startStreamConsumer();
```

---

### ৫. Laravel Queue — PHP-র সেরা Queue ইকোসিস্টেম

Laravel Queue হলো PHP জগতে সবচেয়ে শক্তিশালী ও সুসংগঠিত queue system। এটি বিভিন্ন backend driver সাপোর্ট করে (Redis, SQS, Database, Beanstalkd) এবং **unified API** দেয়।

```php
<?php
// ১. Job তৈরি — বিকাশ ক্যাশআউট নোটিফিকেশন
// php artisan make:job ProcessCashoutNotification

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\Middleware\WithoutOverlapping;
use Illuminate\Queue\Middleware\RateLimited;

class ProcessCashoutNotification implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;           // সর্বোচ্চ ৩ বার চেষ্টা
    public int $timeout = 30;        // ৩০ সেকেন্ড timeout
    public int $maxExceptions = 2;   // ২টি exception-এর পর বন্ধ
    public array $backoff = [10, 30, 60]; // retry delay (সেকেন্ড)

    public function __construct(
        public readonly string $transactionId,
        public readonly string $phone,
        public readonly float $amount,
    ) {}

    // Middleware — একই transaction-এ overlapping রোধ + rate limiting
    public function middleware(): array
    {
        return [
            new WithoutOverlapping($this->transactionId),
            new RateLimited('sms-notifications'),
        ];
    }

    public function handle(): void
    {
        $message = "৳{$this->amount} ক্যাশআউট সফল। TXN: {$this->transactionId}";
        SmsService::send($this->phone, $message);
    }

    // সব retry ব্যর্থ হলে
    public function failed(\Throwable $exception): void
    {
        Log::critical("SMS ব্যর্থ — TXN: {$this->transactionId}", [
            'error' => $exception->getMessage(),
        ]);
        AlertService::notifyOps("Critical: SMS delivery failure for {$this->transactionId}");
    }
}
```

```php
<?php
// ২. Job Dispatch — বিভিন্ন কৌশল

// সাধারণ dispatch
ProcessCashoutNotification::dispatch('TXN001', '+8801712345678', 5000);

// নির্দিষ্ট queue-তে পাঠানো
ProcessCashoutNotification::dispatch('TXN002', '+8801812345678', 10000)
    ->onQueue('high-priority')
    ->delay(now()->addSeconds(5)); // ৫ সেকেন্ড পর

// ৩. Job Chain — ক্রমানুসারে একটির পর একটি
use Illuminate\Support\Facades\Bus;

Bus::chain([
    new ValidateTransaction($orderId),
    new ProcessPayment($orderId),
    new SendConfirmationSms($orderId),
    new UpdateInventory($orderId),
])->onQueue('orders')
  ->catch(fn (\Throwable $e) => Log::error("Chain ব্যর্থ: {$e->getMessage()}"))
  ->dispatch();

// ৪. Job Batch — সমান্তরালে একাধিক job, progress tracking সহ
$batch = Bus::batch([
    new SendPromotionalSms('+8801711111111', 'ঈদ অফার!'),
    new SendPromotionalSms('+8801722222222', 'ঈদ অফার!'),
    new SendPromotionalSms('+8801733333333', 'ঈদ অফার!'),
    // হাজার হাজার SMS job...
])
->then(fn ($batch) => Log::info("সব SMS পাঠানো সম্পন্ন! মোট: {$batch->totalJobs}"))
->catch(fn ($batch, $e) => Log::error("কিছু SMS ব্যর্থ: {$batch->failedJobs}"))
->finally(fn ($batch) => NotifyAdmin::dispatch("Batch {$batch->id} শেষ"))
->allowFailures()
->onQueue('bulk-sms')
->dispatch();

echo "Batch ID: {$batch->id} — Progress: {$batch->progress()}%\n";

// ৫. Rate Limiting (AppServiceProvider-এ)
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('sms-notifications', function ($job) {
    return Limit::perMinute(100); // মিনিটে সর্বোচ্চ ১০০ SMS
});
```

```
Laravel Horizon — Queue Monitoring Dashboard

  horizon.conf:
  ┌─────────────────────────────────────────────┐
  │  Environment: production                     │
  │  ├── supervisor: default                     │
  │  │   ├── queue: default, high-priority       │
  │  │   ├── processes: 10                       │
  │  │   ├── tries: 3                            │
  │  │   └── balance: auto (auto-scaling)        │
  │  ├── supervisor: bulk                        │
  │  │   ├── queue: bulk-sms, bulk-email         │
  │  │   ├── processes: 20                       │
  │  │   └── balance: simple                     │
  │  └── metrics retention: 7 days               │
  └─────────────────────────────────────────────┘
```

---

### ৬. BullMQ (Node.js) — আধুনিক Queue Library

BullMQ হলো Node.js ইকোসিস্টেমের সবচেয়ে শক্তিশালী Redis-ভিত্তিক queue library। এটি Laravel Queue-র Node.js সমতুল্য।

```javascript
// bullmq-setup.js — সম্পূর্ণ সেটআপ
const { Queue, Worker, QueueScheduler, FlowProducer } = require('bullmq');

const connection = { host: '127.0.0.1', port: 6379 };

// ১. Queue তৈরি
const orderQueue = new Queue('order-processing', {
    connection,
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 2000 },
        removeOnComplete: { age: 3600 * 24 },  // ২৪ ঘণ্টা পর মুছে যাবে
        removeOnFail: { age: 3600 * 24 * 7 },  // ৭ দিন পর মুছবে
    },
});

// ২. বিভিন্ন ধরনের Job যোগ করা
async function addJobs() {
    // সাধারণ job
    await orderQueue.add('process-order', {
        orderId: 'ORD_001',
        customer: 'রহিম',
        amount: 25000,
    });

    // বিলম্বিত job — ৫ মিনিট পর চলবে
    await orderQueue.add('send-reminder', {
        orderId: 'ORD_001',
        message: 'আপনার অর্ডার কনফার্ম করুন',
    }, { delay: 5 * 60 * 1000 });

    // অগ্রাধিকার job — ১ (সর্বোচ্চ) থেকে ২০ (সর্বনিম্ন)
    await orderQueue.add('urgent-refund', {
        orderId: 'ORD_002',
        refundAmount: 5000,
    }, { priority: 1 });

    // পুনরাবৃত্তি job — প্রতি ৫ মিনিটে
    await orderQueue.add('check-pending-orders', {}, {
        repeat: { every: 5 * 60 * 1000 },
        jobId: 'pending-check', // unique ID → duplicate রোধ
    });
}

// ৩. Worker — কাজ প্রসেস করা
const worker = new Worker('order-processing', async (job) => {
    console.log(`কাজ শুরু: ${job.name} — ${job.id}`);

    switch (job.name) {
        case 'process-order':
            await processOrder(job.data);
            break;
        case 'send-reminder':
            await sendReminder(job.data);
            break;
        case 'urgent-refund':
            await processRefund(job.data);
            break;
        case 'check-pending-orders':
            await checkPending();
            break;
    }

    // Progress report (UI-তে দেখানো যায়)
    await job.updateProgress(100);
    return { success: true, processedAt: new Date().toISOString() };
}, {
    connection,
    concurrency: 5,                // একসাথে ৫টি job
    limiter: { max: 100, duration: 60000 }, // মিনিটে সর্বোচ্চ ১০০
});

// ৪. Worker ইভেন্ট
worker.on('completed', (job, result) => {
    console.log(`সম্পন্ন: ${job.id}`);
});

worker.on('failed', (job, err) => {
    console.error(`ব্যর্থ: ${job.id} — ${err.message}`);
});

worker.on('stalled', (jobId) => {
    console.warn(`আটকে গেছে: ${jobId} — অন্য worker নেবে`);
});

// ৫. Flow — parent-child সম্পর্কযুক্ত job
const flowProducer = new FlowProducer({ connection });

await flowProducer.add({
    name: 'complete-order',
    queueName: 'order-processing',
    data: { orderId: 'ORD_003' },
    children: [
        { name: 'validate-payment', queueName: 'order-processing', data: { orderId: 'ORD_003' } },
        { name: 'check-inventory', queueName: 'order-processing', data: { orderId: 'ORD_003' } },
        { name: 'calculate-shipping', queueName: 'order-processing', data: { orderId: 'ORD_003' } },
    ],
});
// children সব সম্পন্ন হলেই parent চলবে

addJobs().catch(console.error);
```

---

## 🔥 Advanced Patterns ও Concepts

### ১. Message Ordering Guarantees

```
┌─────────────────────────────────────────────────────────────┐
│              Ordering Strategies                             │
│                                                             │
│  ① Global FIFO (SQS FIFO, RabbitMQ single queue)           │
│     msg1 → msg2 → msg3 → msg4                              │
│     সুবিধা: সম্পূর্ণ ক্রম গ্যারান্টি                        │
│     অসুবিধা: throughput সীমিত                                │
│                                                             │
│  ② Partition-based Ordering (Kafka)                         │
│     Partition 0: [user_A:msg1] → [user_A:msg3]             │
│     Partition 1: [user_B:msg2] → [user_B:msg4]             │
│     সুবিধা: একই key-র মেসেজে ক্রম + উচ্চ throughput        │
│     অসুবিধা: শুধু partition-এর ভিতরে ক্রম গ্যারান্টি        │
│                                                             │
│  ③ No Ordering (SQS Standard)                               │
│     msg3 → msg1 → msg4 → msg2 (যেকোনো ক্রমে)              │
│     সুবিধা: সর্বোচ্চ throughput                               │
│     অসুবিধা: কোনো ক্রম গ্যারান্টি নেই                       │
│                                                             │
│  ✦ বাস্তব সিদ্ধান্ত: bKash-এ একই ইউজারের ট্রানজ্যাকশন     │
│    ক্রমানুসারে হতে হবে — Kafka partition key = user_id      │
└─────────────────────────────────────────────────────────────┘
```

### ২. Delivery Semantics — তিনটি গ্যারান্টি স্তর

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  At-most-once (সর্বোচ্চ একবার)                                  │
│  ─────────────────────────────                                   │
│  Fire-and-forget। মেসেজ হারাতে পারে, duplicate হবে না।            │
│  ব্যবহার: লগিং, মেট্রিক্স — হারালে বড় ক্ষতি নেই                │
│  কৌশল: ACK আগে পাঠাও, তারপর প্রসেস করো                         │
│                                                                  │
│  Producer ──▶ Broker ──▶ Consumer                                │
│                          │ ACK ✓                                 │
│                          │ Process... (ব্যর্থ হলে মেসেজ হারাবে)  │
│                                                                  │
│  At-least-once (ন্যূনতম একবার) ← সবচেয়ে সাধারণ                  │
│  ──────────────────────────────                                  │
│  মেসেজ হারাবে না, কিন্তু duplicate আসতে পারে।                    │
│  ব্যবহার: অধিকাংশ সিস্টেম — idempotent handler দরকার           │
│  কৌশল: প্রসেস আগে করো, তারপর ACK পাঠাও                         │
│                                                                  │
│  Producer ──▶ Broker ──▶ Consumer                                │
│                          │ Process... ✓                           │
│                          │ ACK... (নেটওয়ার্ক সমস্যা!)            │
│                          │ Broker আবার পাঠায় → duplicate          │
│                                                                  │
│  Exactly-once (ঠিক একবার) ← সবচেয়ে কঠিন                        │
│  ─────────────────────────                                       │
│  না হারাবে, না duplicate হবে।                                    │
│  ব্যবহার: আর্থিক লেনদেন — bKash, পেমেন্ট                       │
│  কৌশল: Kafka transactions + idempotent producer                  │
│         অথবা, consumer-side deduplication                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```php
<?php
// Idempotent Consumer Pattern — duplicate মেসেজ সামলানো
class IdempotentOrderProcessor
{
    public function handle(array $message): void
    {
        $messageId = $message['message_id'];

        // ডাটাবেসে atomic check-and-insert
        $alreadyProcessed = DB::table('processed_messages')
            ->where('message_id', $messageId)
            ->exists();

        if ($alreadyProcessed) {
            Log::info("Duplicate মেসেজ উপেক্ষা: {$messageId}");
            return;
        }

        DB::transaction(function () use ($message, $messageId) {
            // মূল কাজ
            Order::create($message['order_data']);

            // প্রসেস রেকর্ড রাখা
            DB::table('processed_messages')->insert([
                'message_id'   => $messageId,
                'processed_at' => now(),
            ]);
        });
    }
}
```

### ৩. Dead Letter Queue (DLQ) Patterns

```
┌────────────────────────────────────────────────────────────┐
│              Dead Letter Queue Strategy                      │
│                                                            │
│  Main Queue                                                │
│  ┌──────────┐     ব্যর্থ (N বার retry-র পর)                │
│  │  msg A   │──────────────────────────┐                   │
│  │  msg B ✗ │──retry──▶ [Retry Queue]  │                   │
│  │  msg C   │          delay: 1s,5s,30s │                   │
│  └──────────┘                 │         ▼                   │
│                               │   ┌───────────┐            │
│                               └──▶│    DLQ    │            │
│                                   │ (বিষ মেসেজ)│            │
│                                   └─────┬─────┘            │
│                                         │                  │
│                            ┌────────────┼────────────┐     │
│                            ▼            ▼            ▼     │
│                      [AlertOps]   [Dashboard]   [Manual    │
│                                                 Replay]    │
│                                                            │
│  DLQ মেসেজ পরীক্ষা করার পর:                                │
│  ① বাগ ঠিক করে আবার main queue-তে পাঠানো                  │
│  ② মেসেজ স্থায়ীভাবে বাতিল করা                              │
│  ③ ম্যানুয়ালি প্রসেস করা                                    │
└────────────────────────────────────────────────────────────┘
```

```javascript
// DLQ monitor ও replay — Node.js
const { Queue, QueueEvents } = require('bullmq');

const mainQueue = new Queue('orders', { connection });
const dlq = new Queue('orders-dlq', { connection });

// ব্যর্থ job গুলো DLQ-তে স্থানান্তর
const queueEvents = new QueueEvents('orders', { connection });
queueEvents.on('failed', async ({ jobId, failedReason }) => {
    const job = await mainQueue.getJob(jobId);
    if (job && job.attemptsMade >= job.opts.attempts) {
        await dlq.add('failed-order', {
            originalData: job.data,
            failedReason,
            failedAt: new Date().toISOString(),
            originalJobId: jobId,
        });
        console.log(`DLQ-তে স্থানান্তরিত: ${jobId}`);
    }
});

// DLQ থেকে replay
async function replayDlqMessages(count = 10) {
    const jobs = await dlq.getJobs(['waiting', 'delayed'], 0, count);
    for (const job of jobs) {
        await mainQueue.add('process-order', job.data.originalData, {
            attempts: 3,
            jobId: `replay_${job.data.originalJobId}`,
        });
        await job.remove();
        console.log(`Replay: ${job.data.originalJobId}`);
    }
}
```

### ৪. Message Schema Evolution — সিস্টেম বড় হলে জরুরি

ডেটা ফরম্যাট সময়ের সাথে বদলায়। পুরাতন consumer যেন নতুন ফরম্যাটের মেসেজ পড়তে পারে, সেজন্য **schema evolution** দরকার।

```
Schema Evolution Strategies:
──────────────────────────────────────────
V1: { orderId, amount }
V2: { orderId, amount, currency }         ← নতুন ফিল্ড (backward compatible)
V3: { orderId, totalAmount, currency }    ← ফিল্ড নাম পরিবর্তন (breaking!)

✅ Backward Compatible: পুরাতন consumer নতুন মেসেজ পড়তে পারে
✅ Forward Compatible: নতুন consumer পুরাতন মেসেজ পড়তে পারে
❌ Breaking Change: কেউই কারোটা পড়তে পারে না

সমাধান:
① JSON — সহজ, কিন্তু schema enforce হয় না
② Avro + Schema Registry — Kafka-র সাথে সবচেয়ে ভালো
③ Protobuf — gRPC দিয়ে ব্যবহারে চমৎকার
```

```javascript
// Schema Registry সহ Avro encoding — kafkajs
const { SchemaRegistry, SchemaType } = require('@kafkajs/confluent-schema-registry');

const registry = new SchemaRegistry({ host: 'http://schema-registry:8081' });

// Schema register
const schema = {
    type: 'record',
    name: 'OrderEvent',
    namespace: 'com.ecommerce',
    fields: [
        { name: 'orderId', type: 'string' },
        { name: 'amount', type: 'double' },
        { name: 'currency', type: 'string', default: 'BDT' }, // default = backward compatible
        { name: 'region', type: ['null', 'string'], default: null },
    ],
};

const { id: schemaId } = await registry.register({
    type: SchemaType.AVRO,
    schema: JSON.stringify(schema),
});

// Encode ও produce
const encodedValue = await registry.encode(schemaId, {
    orderId: 'ORD_001',
    amount: 25000.0,
    currency: 'BDT',
    region: 'dhaka',
});

await producer.send({
    topic: 'order-events',
    messages: [{ key: 'ORD_001', value: encodedValue }],
});
```

### ৫. Backpressure Handling

**Backpressure** তখন হয় যখন producer মেসেজ পাঠায় consumer-এর প্রসেসিং ক্ষমতার চেয়ে দ্রুত। এটি সামলানো না হলে মেমরি শেষ হয়ে সিস্টেম ক্র্যাশ করে।

```
  সমস্যা:
  Producer (1000 msg/s) ═══▶ Queue (ভরে যাচ্ছে!) ═══▶ Consumer (100 msg/s)

  সমাধান কৌশল:
  ──────────────────────────────────────────────────────
  ① Prefetch/QoS Limit
     Consumer বলে: "আমাকে একবারে ১০টির বেশি দিও না"
     → RabbitMQ: channel.prefetch(10)
     → BullMQ: concurrency: 5

  ② Queue Size Limit
     Queue পূর্ণ হলে নতুন মেসেজ প্রত্যাখ্যান
     → RabbitMQ: x-max-length, x-overflow: reject-publish
     → Producer-কে জানানো হয় → সে ধীরে পাঠায়

  ③ Rate Limiting
     → BullMQ: limiter: { max: 100, duration: 60000 }
     → Laravel: RateLimiter::for('queue', ...)

  ④ Auto-scaling Consumers
     Queue depth বাড়লে নতুন consumer চালু হয়
     → Laravel Horizon: balance: 'auto'
     → Kubernetes HPA + queue metrics

  ⑤ Batching
     ছোট ছোট মেসেজ একত্র করে বড় batch-এ প্রসেস
     → Kafka: linger.ms, batch.size
```

### ৬. Priority Queues

```php
<?php
// Laravel-এ Priority Queue কৌশল — আলাদা queue ব্যবহার
// কনফিগ: config/horizon.php
return [
    'environments' => [
        'production' => [
            'supervisor-critical' => [
                'queue' => ['critical'],
                'processes' => 5,
            ],
            'supervisor-high' => [
                'queue' => ['high', 'default'],
                'processes' => 10,
            ],
            'supervisor-low' => [
                'queue' => ['low', 'bulk'],
                'processes' => 3,
            ],
        ],
    ],
];

// Dispatch সময় queue নির্দিষ্ট করা
ProcessRefund::dispatch($refund)->onQueue('critical');
SendOrderConfirmation::dispatch($order)->onQueue('high');
GenerateReport::dispatch($params)->onQueue('low');
```

### ৭. Queue-based Load Leveling Pattern

```
  ট্র্যাফিক স্পাইক (যেমন: ঈদের দিন bKash-এ):
  ─────────────────────────────────────────────

  সরাসরি (Queue ছাড়া):
  Request ████████████████████████ → [Server] 💥 ক্র্যাশ!
  Spike!   ████████████████████████

  Queue দিয়ে Load Leveling:
  Request ████████████████████████ → [Queue ████████] → [Server] ✅ স্থিতিশীল
  Spike!   ████████████████████████   buffer হিসেবে      ধীরে কিন্তু
                                      কাজ করে            ক্র্যাশ হয় না

  Queue গভীরতা অনুযায়ী auto-scaling:
  ┌──────────────────────────┐
  │ Queue Depth │ Consumers  │
  ├──────────────────────────┤
  │ 0-100       │ 2          │
  │ 100-1000    │ 5          │
  │ 1000-5000   │ 10         │
  │ 5000+       │ 20 + alert │
  └──────────────────────────┘
```

### ৮. Competing Consumers Pattern

```
  একই queue থেকে একাধিক consumer একই মেসেজ প্রসেস করে
  (কিন্তু প্রতিটি মেসেজ শুধু একজনই পায়):

  ┌─────────┐     ┌────────────────┐     ┌──────────────┐
  │         │     │    Queue       │     │ Consumer #1  │◀── msg1, msg4
  │Producer │────▶│ [m1|m2|m3|m4] │────▶│ Consumer #2  │◀── msg2, msg5
  │         │     │                │     │ Consumer #3  │◀── msg3, msg6
  └─────────┘     └────────────────┘     └──────────────┘

  সুবিধা:
  • অনুভূমিক scaling — consumer যোগ করলেই throughput বাড়ে
  • একটি consumer ব্যর্থ হলে অন্যরা চালু থাকে
  • কোনো মেসেজ হারায় না (unacked মেসেজ অন্যকে দেওয়া হয়)

  ⚠ সতর্কতা:
  • মেসেজ ক্রম গ্যারান্টি নেই (ভিন্ন consumer ভিন্ন সময়ে শেষ করে)
  • Idempotent handler আবশ্যক
```

---

## 🆚 প্রযুক্তি তুলনা

| বৈশিষ্ট্য | RabbitMQ | Kafka | AWS SQS | Redis Streams |
|-----------|----------|-------|---------|---------------|
| **ধরন** | Message Broker | Event Streaming | Managed Queue | In-memory Stream |
| **Protocol** | AMQP | Custom (TCP) | HTTP/REST | Redis Protocol |
| **Ordering** | Queue-level FIFO | Partition-level | FIFO queue-তে | Stream-level |
| **Retention** | ACK-এর পর মুছে যায় | Configurable (দিন/সপ্তাহ) | ১৪ দিন সর্বোচ্চ | Memory/disk সীমিত |
| **Throughput** | ~50K msg/s | ~1M+ msg/s | সীমাহীন (managed) | ~100K msg/s |
| **Delivery** | At-least-once | Exactly-once সম্ভব | At-least-once/Exactly (FIFO) | At-least-once |
| **Replay** | ❌ না | ✅ হ্যাঁ (offset reset) | ❌ না | ⚠ সীমিত |
| **Routing** | শক্তিশালী (exchange) | Topic/partition | সীমিত | সীমিত |
| **Scaling** | Vertical + clustering | Horizontal (partition) | Auto (AWS managed) | Vertical |
| **জটিলতা** | মাঝারি | উচ্চ | সহজ | সহজ |
| **খরচ** | Self-hosted/Cloud | Self-hosted/Confluent | Pay-per-use | Self-hosted |
| **আদর্শ ব্যবহার** | Task queue, RPC | Event sourcing, log | Serverless, AWS | Cache + simple queue |
| **বাংলাদেশ উদাহরণ** | SMS queue | ট্রানজ্যাকশন লগ | Serverless API | Session/cache queue |

---

## ✅ সুবিধা ও ❌ অসুবিধা

### সুবিধা (Pros)

| # | সুবিধা | বিস্তারিত |
|---|--------|----------|
| ১ | **Decoupling** | সার্ভিস একে অপরকে না জেনেও যোগাযোগ করতে পারে |
| ২ | **Resilience** | একটি সার্ভিস ডাউন হলে মেসেজ queue-তে জমা থাকে |
| ৩ | **Load Leveling** | ট্র্যাফিক স্পাইকে সার্ভার ক্র্যাশ হয় না |
| ৪ | **Scalability** | Consumer যোগ করলেই throughput বাড়ে |
| ৫ | **Async Processing** | ইউজারকে অপেক্ষা করতে হয় না — দ্রুত response |
| ৬ | **Retry/DLQ** | ব্যর্থ কাজ স্বয়ংক্রিয়ভাবে পুনরায় চেষ্টা হয় |
| ৭ | **Audit Trail** | Kafka-তে সব ইভেন্টের ইতিহাস থাকে |

### অসুবিধা (Cons)

| # | অসুবিধা | বিস্তারিত |
|---|---------|----------|
| ১ | **Complexity** | সিস্টেমে নতুন কম্পোনেন্ট — monitoring, debugging কঠিন |
| ২ | **Eventual Consistency** | ডেটা তাৎক্ষণিক সবখানে আপডেট হয় না |
| ৩ | **Message Ordering** | বিশেষ ব্যবস্থা ছাড়া ক্রম নষ্ট হতে পারে |
| ৪ | **Duplicate Handling** | Idempotent consumer লিখতে হয় |
| ৫ | **Operational Overhead** | Broker পরিচালনা, monitoring, alerting দরকার |
| ৬ | **Latency** | Synchronous-এর তুলনায় কিছুটা বিলম্ব |
| ৭ | **Debugging** | বিতরিত সিস্টেমে trace করা কঠিন — correlation ID দরকার |

---

## ⚠️ সাধারণ ভুল ও সমাধান

### ১. ACK না পাঠানো / ভুল সময়ে পাঠানো
```
❌ ভুল: Auto-ack চালু, কিন্তু process ব্যর্থ → মেসেজ হারিয়ে গেলো
✅ সঠিক: Manual ack — প্রসেস সম্পন্ন হলেই ack পাঠাও
```

### ২. Idempotency ভুলে যাওয়া
```
❌ ভুল: একই পেমেন্ট দুইবার প্রসেস → গ্রাহক দুইবার চার্জ!
✅ সঠিক: message_id দিয়ে deduplication, DB-তে unique constraint
```

### ৩. Queue-কে Database হিসেবে ব্যবহার
```
❌ ভুল: লক্ষ লক্ষ মেসেজ queue-তে জমিয়ে রাখা, মাঝে মাঝে query করা
✅ সঠিক: Queue হলো transit — প্রসেস করো এবং result DB-তে রাখো
```

### ৪. বিশাল মেসেজ পাঠানো
```
❌ ভুল: ১০ MB ফাইল সরাসরি queue-তে
✅ সঠিক: ফাইল S3/MinIO-তে আপলোড, queue-তে শুধু reference পাঠাও
   { "file_url": "s3://bucket/file.pdf", "action": "process" }
```

### ৫. Monitoring না করা
```
❌ ভুল: Queue depth ক্রমশ বাড়ছে, কেউ জানে না → সিস্টেম ক্র্যাশ
✅ সঠিক: Queue depth, consumer lag, error rate — সব মনিটর করো
   Alert: queue_depth > 10000 → PagerDuty notification
```

### ৬. Retry-তে Exponential Backoff না ব্যবহার
```
❌ ভুল: তাৎক্ষণিক retry → ব্যর্থ সার্ভিসকে আরও চাপ দেওয়া
✅ সঠিক: 1s → 2s → 4s → 8s → 16s (exponential backoff)
```

### ৭. Correlation ID না রাখা
```
❌ ভুল: বিতরিত সিস্টেমে কোন request কোথায় আটকে আছে বুঝতে পারছি না
✅ সঠিক: প্রতিটি মেসেজে unique correlation_id → পুরো flow trace করা যায়
   headers: { 'x-correlation-id': 'req_abc123_20240115' }
```

---

## 📋 সারসংক্ষেপ

```
মেসেজ কিউ — মূল শিক্ষা:
═══════════════════════════════════════════════════════════════

১. মেসেজ কিউ = Async + Decoupling + Resilience
   → সার্ভিসগুলো স্বাধীনভাবে কাজ করতে পারে

২. সঠিক প্রযুক্তি নির্বাচন:
   • সাধারণ task queue → RabbitMQ / Laravel Queue
   • Event streaming/replay → Kafka
   • Serverless/AWS → SQS + SNS
   • সহজ ও দ্রুত → Redis Streams / BullMQ

৩. Delivery guarantee বুঝে নির্বাচন করো:
   • আর্থিক লেনদেন → Exactly-once (Kafka transactions)
   • সাধারণ কাজ → At-least-once + idempotent consumer

৪. সবসময় মনে রাখো:
   • Manual ACK ব্যবহার করো
   • Idempotent consumer লেখো
   • DLQ সেটআপ করো
   • Monitoring ও alerting অবশ্যই রাখো
   • Correlation ID দিয়ে tracing করো
   • বড় payload queue-তে রেখো না — reference পাঠাও

৫. বাংলাদেশ প্রেক্ষাপট:
   • bKash/Nagad → Kafka (ট্রানজ্যাকশন log + exactly-once)
   • ই-কমার্স অর্ডার → RabbitMQ/Laravel Queue (task processing)
   • নোটিফিকেশন (SMS/Push) → SQS অথবা BullMQ (rate limited)
   • রিয়েল-টাইম ড্যাশবোর্ড → Redis Streams (দ্রুত, কম জটিল)

═══════════════════════════════════════════════════════════════
```
