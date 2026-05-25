# ⚡ DynamoDB Architecture — AWS-এর Serverless NoSQL

> **"DynamoDB হলো AWS-এর fully managed, serverless, key-value ও document database — যেকোনো scale-এ single-digit millisecond latency guarantee করে।"**
>
> Amazon.com-এর shopping cart, Prime Video, Alexa, AWS IAM — সব DynamoDB-তে চলে।

---

## 📌 সংজ্ঞা ও ইতিহাস

### DynamoDB কী?

DynamoDB হলো AWS-এর **fully managed NoSQL database service** — provisioning, patching, replication, failover সব AWS handle করে। এটি Amazon-এর ২০০৭ সালের Dynamo paper ("Dynamo: Amazon's Highly Available Key-value Store") থেকে অনুপ্রাণিত, তবে অনেক evolved।

```
┌──────────────────────────────────────────────────────────────┐
│  DynamoDB Core Promises:                                      │
│                                                               │
│  ✅ Single-digit ms latency (any scale)                      │
│  ✅ Fully managed (no servers to manage)                     │
│  ✅ Auto-scaling (0 to millions of requests/sec)             │
│  ✅ Built-in security, backup, recovery                      │
│  ✅ Multi-region (Global Tables)                             │
│  ✅ Event-driven (DynamoDB Streams)                          │
│  ✅ ACID transactions (since 2018)                           │
│                                                               │
│  Trade-off:                                                   │
│  ❌ No ad-hoc queries (no SQL joins, limited filtering)      │
│  ❌ Requires careful access pattern design upfront           │
│  ❌ Vendor lock-in (AWS-specific)                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 🏗️ DynamoDB Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB Internal Architecture                    │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │           Request Router                            │      │
│  │   - Authentication / Authorization (IAM)           │      │
│  │   - Request validation                             │      │
│  │   - Route to correct partition                     │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │           Partition Metadata                        │      │
│  │   - Table → Partition mapping                      │      │
│  │   - Partition key → Storage node mapping           │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │                                    │
│  ┌───────────────────────▼────────────────────────────┐      │
│  │              Storage Nodes                          │      │
│  │                                                     │      │
│  │   ┌──────────────┐  ┌──────────────┐               │      │
│  │   │ Partition 1  │  │ Partition 2  │ ...           │      │
│  │   │              │  │              │               │      │
│  │   │ Leader node  │  │ Leader node  │               │      │
│  │   │ + 2 replicas │  │ + 2 replicas │               │      │
│  │   │ (3 AZs)     │  │ (3 AZs)     │               │      │
│  │   └──────────────┘  └──────────────┘               │      │
│  │                                                     │      │
│  │   B-tree storage engine                            │      │
│  │   Write: leader → 2/3 quorum ack → client ack     │      │
│  │   Read (strong): leader node                       │      │
│  │   Read (eventual): any replica (cheaper)           │      │
│  └─────────────────────────────────────────────────────┘      │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐      │
│  │              DynamoDB Streams                        │      │
│  │   - Change data capture (CDC)                       │      │
│  │   - Ordered by partition key                        │      │
│  │   - 24-hour retention                               │      │
│  │   - Triggers Lambda functions                       │      │
│  └─────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Partition Strategy

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB Partitioning                             │
│                                                               │
│  Table: "Orders"                                             │
│  Partition Key: user_id                                      │
│                                                               │
│  hash(user_id) → Partition assignment                        │
│                                                               │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│  │Partition 1 │   │Partition 2 │   │Partition 3 │           │
│  │hash: 0-33% │   │hash: 34-66%│   │hash: 67-100%│          │
│  │            │   │            │   │            │           │
│  │user_1 orders│  │user_5 orders│  │user_9 orders│           │
│  │user_2 orders│  │user_6 orders│  │user_10 orders│          │
│  │user_3 orders│  │user_7 orders│  │            │           │
│  └────────────┘   └────────────┘   └────────────┘           │
│                                                               │
│  প্রতিটি Partition capacity:                                  │
│    - 10GB data                                               │
│    - 3000 RCU (Read Capacity Units)                          │
│    - 1000 WCU (Write Capacity Units)                         │
│                                                               │
│  Table বড় হলে: auto-split into more partitions              │
│                                                               │
│  ⚠️ Hot partition problem:                                   │
│    একটি partition key-তে অতিরিক্ত traffic → throttle!       │
│    Solution: composite key, write sharding                   │
└──────────────────────────────────────────────────────────────┘
```

### Replication Model

```
┌──────────────────────────────────────────────────────────────┐
│          DynamoDB Replication (Single Region)                  │
│                                                               │
│                    Availability Zone A                        │
│                    ┌──────────────┐                           │
│                    │  Replica 1   │ ← Leader                 │
│                    │  (writes)    │                           │
│                    └──────┬───────┘                           │
│                           │                                   │
│           ┌───────────────┼───────────────┐                   │
│           ▼                               ▼                   │
│  AZ B ┌──────────────┐        AZ C ┌──────────────┐         │
│       │  Replica 2   │             │  Replica 3   │         │
│       │  (reads)     │             │  (reads)     │         │
│       └──────────────┘             └──────────────┘         │
│                                                               │
│  Write: Client → Leader → 2/3 replicas ack → Client ack    │
│  Strong Read: Goes to Leader                                 │
│  Eventually Consistent Read: Any replica (2x cheaper)       │
│                                                               │
│  Global Tables (Multi-Region):                               │
│  ┌────────────┐       ┌────────────┐       ┌────────────┐   │
│  │ ap-south-1 │◄─────►│ us-east-1  │◄─────►│ eu-west-1  │   │
│  │  (Mumbai)  │       │ (Virginia) │       │ (Ireland)  │   │
│  └────────────┘       └────────────┘       └────────────┘   │
│                                                               │
│  Multi-master: write to any region                           │
│  Conflict resolution: Last Writer Wins (timestamp)          │
│  Replication lag: typically < 1 second                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Data Model

### Primary Key Types

```
┌──────────────────────────────────────────────────────────────┐
│  1. Simple Primary Key (Partition Key only):                  │
│                                                               │
│     Table: Users                                             │
│     PK: user_id                                              │
│                                                               │
│     user_id │ name    │ email          │ city                │
│     ────────┼─────────┼────────────────┼────────────────     │
│     u001    │ রহিম    │ rahim@mail.com │ ঢাকা               │
│     u002    │ করিম    │ karim@mail.com │ চট্টগ্রাম          │
│                                                               │
│  2. Composite Primary Key (Partition Key + Sort Key):         │
│                                                               │
│     Table: Orders                                            │
│     PK: user_id (partition) + order_date (sort)              │
│                                                               │
│     user_id │ order_date  │ order_id │ total │ status        │
│     ────────┼─────────────┼──────────┼───────┼───────        │
│     u001    │ 2024-01-01  │ ord-101  │ 500   │ delivered     │
│     u001    │ 2024-01-15  │ ord-102  │ 1200  │ shipped       │
│     u001    │ 2024-02-01  │ ord-103  │ 800   │ pending       │
│     u002    │ 2024-01-05  │ ord-104  │ 300   │ delivered     │
│                                                               │
│     Query: user_id = "u001" AND order_date BETWEEN ...       │
│     Same partition → efficient range query!                  │
└──────────────────────────────────────────────────────────────┘
```

### Single Table Design Pattern

```
┌──────────────────────────────────────────────────────────────┐
│  DynamoDB Best Practice: Single Table Design                  │
│                                                               │
│  Multiple entities in one table (overloaded keys):           │
│                                                               │
│  PK           │ SK              │ data                        │
│  ─────────────┼─────────────────┼──────────────────────      │
│  USER#u001    │ PROFILE         │ {name, email, city}        │
│  USER#u001    │ ORDER#2024-01   │ {orderId, total, items}    │
│  USER#u001    │ ORDER#2024-02   │ {orderId, total, items}    │
│  USER#u001    │ ADDRESS#home    │ {street, city, zip}        │
│  PRODUCT#p001 │ DETAILS         │ {name, price, category}    │
│  PRODUCT#p001 │ REVIEW#u001     │ {rating, comment}          │
│                                                               │
│  Access patterns:                                            │
│    GET user profile: PK=USER#u001, SK=PROFILE                │
│    GET user orders:  PK=USER#u001, SK begins_with ORDER#     │
│    GET product reviews: PK=PRODUCT#p001, SK begins_with REVIEW# │
│                                                               │
│  একটি table-ই সব entity handle করে — কম latency, কম cost!   │
└──────────────────────────────────────────────────────────────┘
```

### Secondary Indexes

```
┌──────────────────────────────────────────────────────────────┐
│  GSI (Global Secondary Index):                                │
│    - নতুন partition key + sort key                            │
│    - Separate throughput capacity                             │
│    - Eventually consistent only                              │
│    - Cross-partition query enable করে                         │
│                                                               │
│  Example: "email দিয়ে user খুঁজতে চাই"                       │
│    GSI: PK = email, SK = user_id                             │
│                                                               │
│  LSI (Local Secondary Index):                                │
│    - Same partition key, different sort key                   │
│    - Table-এর সাথে same partition                             │
│    - Strong consistency support                              │
│    - Table create-এর সময়ই define করতে হবে                    │
│                                                               │
│  Example: "user-এর orders total amount দিয়ে sort করতে চাই"  │
│    LSI: PK = user_id (same), SK = total_amount (different)   │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP Code Example (AWS SDK)

```php
<?php

use Aws\DynamoDb\DynamoDbClient;
use Aws\DynamoDb\Marshaler;

class DynamoDBRepository
{
    private DynamoDbClient $client;
    private Marshaler $marshaler;
    private string $tableName;

    public function __construct(string $tableName)
    {
        $this->client = new DynamoDbClient([
            'region' => 'ap-southeast-1',
            'version' => 'latest',
        ]);
        $this->marshaler = new Marshaler();
        $this->tableName = $tableName;
    }

    /**
     * Single item put
     */
    public function putItem(array $item): void
    {
        $this->client->putItem([
            'TableName' => $this->tableName,
            'Item' => $this->marshaler->marshalItem($item),
            'ConditionExpression' => 'attribute_not_exists(PK)', // prevent overwrite
        ]);
    }

    /**
     * Get item by primary key
     */
    public function getItem(string $pk, string $sk): ?array
    {
        $result = $this->client->getItem([
            'TableName' => $this->tableName,
            'Key' => $this->marshaler->marshalItem(['PK' => $pk, 'SK' => $sk]),
            'ConsistentRead' => true, // strong consistency
        ]);

        $item = $result['Item'] ?? null;
        return $item ? $this->marshaler->unmarshalItem($item) : null;
    }

    /**
     * Query items by partition key with sort key condition
     */
    public function queryByPartition(string $pk, string $skPrefix, int $limit = 25): array
    {
        $result = $this->client->query([
            'TableName' => $this->tableName,
            'KeyConditionExpression' => 'PK = :pk AND begins_with(SK, :sk)',
            'ExpressionAttributeValues' => $this->marshaler->marshalItem([
                ':pk' => $pk,
                ':sk' => $skPrefix,
            ]),
            'Limit' => $limit,
            'ScanIndexForward' => false, // descending (latest first)
        ]);

        return array_map(
            fn($item) => $this->marshaler->unmarshalItem($item),
            $result['Items']
        );
    }

    /**
     * Transaction write (ACID across multiple items)
     */
    public function transferFunds(string $fromUser, string $toUser, float $amount): void
    {
        $this->client->transactWriteItems([
            'TransactItems' => [
                [
                    'Update' => [
                        'TableName' => $this->tableName,
                        'Key' => $this->marshaler->marshalItem([
                            'PK' => "USER#{$fromUser}", 'SK' => 'BALANCE',
                        ]),
                        'UpdateExpression' => 'SET balance = balance - :amount',
                        'ConditionExpression' => 'balance >= :amount',
                        'ExpressionAttributeValues' => $this->marshaler->marshalItem([':amount' => $amount]),
                    ],
                ],
                [
                    'Update' => [
                        'TableName' => $this->tableName,
                        'Key' => $this->marshaler->marshalItem([
                            'PK' => "USER#{$toUser}", 'SK' => 'BALANCE',
                        ]),
                        'UpdateExpression' => 'SET balance = balance + :amount',
                        'ExpressionAttributeValues' => $this->marshaler->marshalItem([':amount' => $amount]),
                    ],
                ],
                [
                    'Put' => [
                        'TableName' => $this->tableName,
                        'Item' => $this->marshaler->marshalItem([
                            'PK' => "TXN#{$fromUser}",
                            'SK' => 'TXN#' . date('c'),
                            'type' => 'transfer',
                            'from' => $fromUser,
                            'to' => $toUser,
                            'amount' => $amount,
                        ]),
                    ],
                ],
            ],
        ]);
    }

    /**
     * Batch write (up to 25 items)
     */
    public function batchWrite(array $items): void
    {
        $requests = array_map(fn($item) => [
            'PutRequest' => ['Item' => $this->marshaler->marshalItem($item)],
        ], $items);

        $this->client->batchWriteItem([
            'RequestItems' => [$this->tableName => $requests],
        ]);
    }
}

// Usage — E-commerce (Daraz-style)
$repo = new DynamoDBRepository('EcommerceTable');

// Create user
$repo->putItem([
    'PK' => 'USER#u001',
    'SK' => 'PROFILE',
    'name' => 'রহিম',
    'email' => 'rahim@example.com',
    'city' => 'ঢাকা',
    'created_at' => date('c'),
]);

// Place order
$repo->putItem([
    'PK' => 'USER#u001',
    'SK' => 'ORDER#' . date('Y-m-d') . '#ord-101',
    'order_id' => 'ord-101',
    'total' => 1500,
    'items' => [['name' => 'T-Shirt', 'qty' => 2, 'price' => 750]],
    'status' => 'pending',
]);

// Get user's recent orders
$orders = $repo->queryByPartition('USER#u001', 'ORDER#', 10);
```

---

## 💻 JavaScript Code Example

```javascript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient, PutCommand, GetCommand,
  QueryCommand, TransactWriteCommand, BatchWriteCommand,
} from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({ region: 'ap-southeast-1' });
const docClient = DynamoDBDocumentClient.from(client);
const TABLE_NAME = 'EcommerceTable';

class DynamoRepository {
  /**
   * Put single item
   */
  async put(item) {
    await docClient.send(new PutCommand({
      TableName: TABLE_NAME,
      Item: item,
      ConditionExpression: 'attribute_not_exists(PK)',
    }));
  }

  /**
   * Get item by PK + SK
   */
  async get(pk, sk) {
    const { Item } = await docClient.send(new GetCommand({
      TableName: TABLE_NAME,
      Key: { PK: pk, SK: sk },
      ConsistentRead: true,
    }));
    return Item || null;
  }

  /**
   * Query by partition key with optional SK prefix
   */
  async query(pk, skPrefix, { limit = 25, ascending = false } = {}) {
    const { Items } = await docClient.send(new QueryCommand({
      TableName: TABLE_NAME,
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
      ExpressionAttributeValues: { ':pk': pk, ':sk': skPrefix },
      Limit: limit,
      ScanIndexForward: ascending,
    }));
    return Items || [];
  }

  /**
   * ACID transaction — multi-item write
   */
  async transferFunds(fromUser, toUser, amount) {
    await docClient.send(new TransactWriteCommand({
      TransactItems: [
        {
          Update: {
            TableName: TABLE_NAME,
            Key: { PK: `USER#${fromUser}`, SK: 'BALANCE' },
            UpdateExpression: 'SET balance = balance - :amount',
            ConditionExpression: 'balance >= :amount',
            ExpressionAttributeValues: { ':amount': amount },
          },
        },
        {
          Update: {
            TableName: TABLE_NAME,
            Key: { PK: `USER#${toUser}`, SK: 'BALANCE' },
            UpdateExpression: 'SET balance = balance + :amount',
            ExpressionAttributeValues: { ':amount': amount },
          },
        },
        {
          Put: {
            TableName: TABLE_NAME,
            Item: {
              PK: `TXN#${fromUser}`,
              SK: `TXN#${new Date().toISOString()}`,
              type: 'transfer',
              from: fromUser,
              to: toUser,
              amount,
            },
          },
        },
      ],
    }));
  }

  /**
   * DynamoDB Streams handler (Lambda trigger)
   */
  async handleStreamEvent(event) {
    for (const record of event.Records) {
      if (record.eventName === 'INSERT' && record.dynamodb.NewImage.PK.S.startsWith('ORDER#')) {
        // New order placed — trigger downstream
        const order = record.dynamodb.NewImage;
        console.log('New order:', order);
        // Send notification, update analytics, etc.
      }
    }
  }
}

// Usage
const repo = new DynamoRepository();

await repo.put({
  PK: 'USER#u001', SK: 'PROFILE',
  name: 'রহিম', email: 'rahim@example.com',
  city: 'ঢাকা', createdAt: new Date().toISOString(),
});

await repo.put({
  PK: 'USER#u001', SK: `ORDER#${new Date().toISOString()}#ord-101`,
  orderId: 'ord-101', total: 1500,
  items: [{ name: 'T-Shirt', qty: 2, price: 750 }],
  status: 'pending',
});

const orders = await repo.query('USER#u001', 'ORDER#', { limit: 10 });
console.log('Recent orders:', orders);
```

---

## 📊 DynamoDB vs অন্যান্য NoSQL

| বৈশিষ্ট্য | DynamoDB | MongoDB | Cassandra | HBase |
|-----------|----------|---------|-----------|-------|
| **Management** | Fully managed | Self/Atlas | Self-managed | Self-managed |
| **Consistency** | Strong + eventual | Strong (single doc) | Tunable | Strong (single row) |
| **Scaling** | Auto (serverless) | Manual sharding | Linear scale | Manual regions |
| **Query** | PK/SK + GSI | Flexible (MQL) | CQL (limited) | Scan/Get |
| **Transactions** | ACID (25 items) | ACID (multi-doc) | Lightweight txn | Single row |
| **Latency** | <10ms guaranteed | Depends on setup | Low | Low |
| **Cost model** | Pay-per-request/provisioned | Infrastructure | Infrastructure | Infrastructure |
| **Vendor lock** | AWS only | Open-source | Open-source | Open-source |

---

## ✅ DynamoDB Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│  Design Principles:                                           │
│                                                               │
│  1. Access patterns FIRST, schema SECOND                     │
│     → List all queries before designing table                │
│                                                               │
│  2. Single table design                                      │
│     → One table serves multiple entities                     │
│     → Reduces latency (fewer round trips)                    │
│                                                               │
│  3. Partition key = even distribution                        │
│     → Avoid hot partitions (e.g., date as PK = bad)         │
│     → High cardinality keys (user_id, order_id)             │
│                                                               │
│  4. Sort key = query flexibility                             │
│     → begins_with, between, >, < operations                 │
│     → Hierarchical: STATUS#DATE, TYPE#TIMESTAMP              │
│                                                               │
│  5. GSI for alternative access patterns                      │
│     → "GSI overloading" — multiple entities share GSI        │
│     → Eventually consistent — acceptable?                    │
│                                                               │
│  6. On-demand for unpredictable, provisioned for steady     │
│     → On-demand: pay per request, no capacity planning       │
│     → Provisioned + auto-scaling: 70% cost savings          │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔑 মনে রাখার বিষয়

1. **Access pattern first** — DynamoDB-তে schema design করার আগে সব query pattern list করুন
2. **Single table** — Multiple entities একটি table-এ রাখুন (joins নেই, তাই denormalize)
3. **Partition key = distribution** — High cardinality key choose করুন (hot partition avoid)
4. **Sort key = range query** — Composite sort key দিয়ে flexible query
5. **GSI is projection** — Eventual consistency, extra cost, কিন্তু alternative access pattern
6. **Transactions available** — 25 items পর্যন্ত ACID transaction সম্ভব
7. **Streams = CDC** — Real-time event processing, Lambda trigger, cross-region replication
8. **Cost model matters** — On-demand (unpredictable) vs Provisioned (steady, 70% cheaper)
