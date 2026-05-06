# 🔄 Change Data Capture (CDC) Deep Dive

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [CDC এর প্রকারভেদ](#cdc-এর-প্রকারভেদ)
- [Debezium Architecture](#debezium-architecture)
- [Kafka Connect Integration](#kafka-connect-integration)
- [CDC vs Polling](#cdc-vs-polling)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Database Connectors](#database-connectors)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Schema Evolution ও Advanced Topics](#schema-evolution-ও-advanced-topics)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Change Data Capture (CDC)** হলো একটি pattern যা database এ হওয়া প্রতিটি change (INSERT, UPDATE, DELETE) ক্যাপচার করে এবং অন্য সিস্টেমে real-time এ propagate করে।

### 🔑 মূল ধারণা:

```
┌────────────────────────────────────────────────────────────────┐
│                    CDC - মূল ধারণা                              │
│                                                                │
│  "Database এর প্রতিটি পরিবর্তন একটি EVENT"                     │
│                                                                │
│  INSERT → Created Event                                        │
│  UPDATE → Updated Event (before + after)                       │
│  DELETE → Deleted Event                                        │
│                                                                │
│  এই events অন্য সিস্টেমে stream করা হয়                        │
│  → Real-time sync                                              │
│  → Event-driven architecture                                   │
│  → Zero-code event publishing                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### CDC কেন দরকার:

| সমস্যা | CDC সমাধান |
|--------|-----------|
| Dual writes problem | DB change থেকে auto event |
| Search index sync | DB → Elasticsearch real-time |
| Cache invalidation | DB change → Redis invalidate |
| Data warehouse sync | OLTP → OLAP real-time |
| Microservice events | DB change → Kafka events |
| Audit trail | সব changes capture করে log |

---

## 📊 CDC এর প্রকারভেদ

### ১. Log-based CDC (সবচেয়ে জনপ্রিয়) ⭐

```
┌────────────────────────────────────────────────────────┐
│              LOG-BASED CDC                              │
│                                                        │
│  ┌──────────┐      ┌─────────────────┐                │
│  │ Database │ ───► │ Transaction Log │                │
│  │          │      │ (binlog/WAL)    │                │
│  └──────────┘      └────────┬────────┘                │
│                              │                         │
│                              ▼                         │
│                     ┌──────────────────┐               │
│                     │  CDC Connector   │               │
│                     │  (Debezium)      │               │
│                     └────────┬─────────┘               │
│                              │                         │
│                              ▼                         │
│                     ┌──────────────────┐               │
│                     │  Message Broker  │               │
│                     │  (Kafka)         │               │
│                     └──────────────────┘               │
│                                                        │
│  সুবিধা:                                               │
│  ✅ সবচেয়ে accurate (transaction log পড়ে)              │
│  ✅ Low latency (milliseconds)                         │
│  ✅ No performance impact on DB                        │
│  ✅ DELETE ও capture হয়                               │
│                                                        │
│  অসুবিধা:                                              │
│  ❌ DB-specific implementation                         │
│  ❌ Log retention configure করতে হয়                    │
│  ❌ Complex setup                                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### ২. Trigger-based CDC

```
┌────────────────────────────────────────────────────────┐
│            TRIGGER-BASED CDC                           │
│                                                        │
│  ┌──────────┐                                          │
│  │ Database │                                          │
│  │          │   TRIGGER fires on INSERT/UPDATE/DELETE   │
│  │ ┌──────┐ │                                          │
│  │ │Table │─┼───► TRIGGER ───► ┌───────────────┐      │
│  │ └──────┘ │                  │ Change Table  │      │
│  │          │                  │ (shadow table)│      │
│  └──────────┘                  └───────┬───────┘      │
│                                        │               │
│                                        ▼               │
│                                ┌──────────────┐        │
│                                │  Poller/     │        │
│                                │  Publisher   │        │
│                                └──────────────┘        │
│                                                        │
│  সুবিধা: ✅ সব DB তে কাজ করে, ✅ Custom logic          │
│  অসুবিধা: ❌ Performance impact, ❌ Maintenance         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### ৩. Timestamp-based CDC

```
┌────────────────────────────────────────────────────────┐
│          TIMESTAMP-BASED CDC                           │
│                                                        │
│  SELECT * FROM products                                │
│  WHERE updated_at > '2024-01-15 10:30:00'             │
│                                                        │
│  ┌──────────┐    Poll every N seconds                  │
│  │ Database │ ◄──────────────── ┌──────────┐          │
│  │          │ ──── results ───► │  Poller  │          │
│  └──────────┘                   └──────────┘          │
│                                                        │
│  সুবিধা: ✅ Simple, ✅ No special DB features           │
│  অসুবিধা: ❌ DELETE ধরা যায় না                          │
│           ❌ Polling delay                              │
│           ❌ updated_at কলাম দরকার                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### ৪. Snapshot-based CDC

```
┌────────────────────────────────────────────────────────┐
│          SNAPSHOT-BASED CDC                            │
│                                                        │
│  পুরো table compare করে changes বের করো:               │
│                                                        │
│  Snapshot T1:  {A, B, C, D}                            │
│  Snapshot T2:  {A, B', C, E}                           │
│                                                        │
│  Diff:                                                 │
│    B → B' (Updated)                                    │
│    D removed (Deleted)                                 │
│    E added (Inserted)                                  │
│                                                        │
│  সুবিধা: ✅ DELETE ধরা যায়, ✅ No schema change         │
│  অসুবিধা: ❌ Resource intensive                         │
│           ❌ Not real-time                              │
│           ❌ Large tables এ impractical                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### তুলনা সারণি:

```
┌──────────────┬──────────┬─────────┬─────────┬──────────┐
│   বৈশিষ্ট্য  │ Log-based│ Trigger │Timestamp│ Snapshot │
├──────────────┼──────────┼─────────┼─────────┼──────────┤
│ Latency      │  <1s     │  <1s    │  N sec  │ Minutes  │
│ DELETE capture│  ✅      │   ✅    │   ❌    │   ✅     │
│ DB Impact    │  None    │  High   │  Medium │   High   │
│ Accuracy     │  100%    │  100%   │   ~95%  │   100%   │
│ Complexity   │  High    │  Medium │   Low   │   Low    │
│ Scalability  │  High    │  Low    │  Medium │   Low    │
└──────────────┴──────────┴─────────┴─────────┴──────────┘
```

---

## 🏗️ Debezium Architecture

**Debezium** হলো সবচেয়ে জনপ্রিয় open-source CDC platform যা Kafka Connect এর উপর চলে।

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEBEZIUM ARCHITECTURE                         │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                 Source Databases                       │       │
│  │  ┌─────────┐  ┌─────────────┐  ┌──────────────┐     │       │
│  │  │  MySQL  │  │ PostgreSQL  │  │   MongoDB    │     │       │
│  │  │ binlog  │  │    WAL      │  │   oplog      │     │       │
│  │  └────┬────┘  └──────┬──────┘  └──────┬───────┘     │       │
│  └───────┼───────────────┼────────────────┼─────────────┘       │
│          │               │                │                      │
│          ▼               ▼                ▼                      │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              Kafka Connect Cluster                     │       │
│  │  ┌──────────────────────────────────────────────┐    │       │
│  │  │         Debezium Source Connectors            │    │       │
│  │  │                                              │    │       │
│  │  │  ┌─────────┐ ┌──────────┐ ┌──────────────┐  │    │       │
│  │  │  │  MySQL  │ │PostgreSQL│ │   MongoDB    │  │    │       │
│  │  │  │Connector│ │Connector │ │  Connector   │  │    │       │
│  │  │  └─────────┘ └──────────┘ └──────────────┘  │    │       │
│  │  └──────────────────────────────────────────────┘    │       │
│  └───────────────────────┬──────────────────────────────┘       │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                  Apache Kafka                         │       │
│  │                                                      │       │
│  │  Topic: dbserver1.inventory.products                 │       │
│  │  Topic: dbserver1.inventory.orders                   │       │
│  │  Topic: dbserver1.inventory.customers                │       │
│  │                                                      │       │
│  └───────────────────────┬──────────────────────────────┘       │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────────┐       │
│  │               Sink Connectors / Consumers             │       │
│  │                                                      │       │
│  │  ┌───────────┐  ┌────────┐  ┌──────────────────┐    │       │
│  │  │Elasticsearch│ │ Redis  │  │ Data Warehouse   │    │       │
│  │  │  Sink      │ │  Sink  │  │    Sink          │    │       │
│  │  └───────────┘  └────────┘  └──────────────────┘    │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Debezium Event Format:

```json
{
  "schema": { ... },
  "payload": {
    "before": {
      "id": 1001,
      "name": "Samsung Galaxy S24",
      "price": 89999,
      "stock": 50
    },
    "after": {
      "id": 1001,
      "name": "Samsung Galaxy S24",
      "price": 84999,
      "stock": 45
    },
    "source": {
      "version": "2.4.0",
      "connector": "mysql",
      "name": "daraz-catalog",
      "ts_ms": 1705312456000,
      "db": "products_db",
      "table": "products"
    },
    "op": "u",
    "ts_ms": 1705312456789
  }
}
```

**op field**: `c` = create, `u` = update, `d` = delete, `r` = read (snapshot)

---

## 🔌 Kafka Connect Integration

### Debezium MySQL Connector Configuration:

```json
{
  "name": "daraz-mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-primary.daraz.internal",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "${secrets:mysql-password}",
    "database.server.id": "184054",
    "topic.prefix": "daraz-catalog",
    "database.include.list": "products_db",
    "table.include.list": "products_db.products,products_db.categories",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.daraz-catalog",
    "include.schema.changes": "true",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "daraz.cdc.$3"
  }
}
```

---

## ⚡ CDC vs Polling

```
┌─────────────────────────────────────────────────────────────┐
│                   CDC vs POLLING                             │
│                                                             │
│  POLLING:                                                   │
│  ┌──────┐  SELECT * WHERE updated > ?  ┌──────────┐        │
│  │Poller│ ◄───────────────────────────► │ Database │        │
│  └──────┘   প্রতি 5 সেকেন্ডে query      └──────────┘        │
│                                                             │
│  সমস্যা:                                                    │
│  • DB তে load বাড়ে                                         │
│  • Delay থাকে (polling interval)                           │
│  • DELETE ধরা যায় না                                       │
│  • Missed updates (between polls)                          │
│  • updated_at column দরকার                                 │
│                                                             │
│  ─────────────────────────────────────────────────────      │
│                                                             │
│  CDC:                                                       │
│  ┌──────────┐    Transaction Log     ┌───────────┐         │
│  │ Database │ ──────────────────────►│ Debezium  │         │
│  └──────────┘    (stream)            └───────────┘         │
│                                                             │
│  সুবিধা:                                                    │
│  • DB তে zero load (log আগেই লেখা হয়)                     │
│  • Real-time (<1 second latency)                           │
│  • সব operation ধরা যায় (INSERT/UPDATE/DELETE)              │
│  • কোনো schema change দরকার নেই                            │
│  • Exactly-once possible (with offset tracking)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Performance তুলনা:

```
Latency (সময়ের পার্থক্য):

Polling:  |████████████████████████████████| 5-30 seconds
CDC:      |██|                               <1 second

DB Load (ডাটাবেস চাপ):

Polling:  |████████████████████| High (repeated queries)
CDC:      |██|                   Minimal (reads log)

Completeness (সম্পূর্ণতা):

Polling:  |████████████████| ~90% (misses deletes, rapid changes)
CDC:      |████████████████████████████████| 100%
```

---

## 🇧🇩 বাস্তব জীবনের উদাহরণ

### উদাহরণ: Daraz Product Catalog → Elasticsearch Sync

```
┌─────────────────────────────────────────────────────────────────┐
│          Daraz: Product Search Sync with CDC                     │
│                                                                 │
│  সমস্যা:                                                        │
│  - MySQL এ ৫০ লক্ষ+ products আছে                                │
│  - Elasticsearch এ search index maintain করতে হয়                │
│  - Product update হলে search এ real-time reflect করতে হয়        │
│  - Polling করলে DB overloaded হয়ে যায়                           │
│                                                                 │
│  সমাধান: CDC (Debezium + Kafka)                                 │
│                                                                 │
│  ┌──────────┐    binlog    ┌──────────┐    ┌────────────────┐   │
│  │  MySQL   │────────────►│ Debezium │───►│     Kafka      │   │
│  │(Products)│             └──────────┘    └────────┬───────┘   │
│  └──────────┘                                      │           │
│       │                                            │           │
│       │ Admin Panel                                ▼           │
│       │ থেকে product                    ┌──────────────────┐   │
│       │ update করে                      │  ES Consumer     │   │
│       │                                 │  Service          │   │
│       │                                 └────────┬─────────┘   │
│       │                                          │             │
│       │                                          ▼             │
│       │                                 ┌──────────────────┐   │
│       │                                 │  Elasticsearch   │   │
│       │                                 │  (Search Index)  │   │
│       │                                 └──────────────────┘   │
│       │                                          │             │
│       │                                          ▼             │
│       │                                 ┌──────────────────┐   │
│       │                                 │   Daraz App      │   │
│       │                                 │   Search Page    │   │
│       ▼                                 └──────────────────┘   │
│                                                                 │
│  ফলাফল:                                                        │
│  ✅ Product update হলে <2 sec এ search এ দেখা যায়               │
│  ✅ MySQL এ extra load নেই                                      │
│  ✅ Price change real-time reflect হয়                           │
│  ✅ Stock update instant (sold out সাথে সাথে দেখায়)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### আরেকটি উদাহরণ: Prothom Alo News Sync

```
┌────────────────────────────────────────────────┐
│     Prothom Alo: CMS → Search/Cache Sync       │
│                                                │
│  PostgreSQL (CMS)                              │
│       │                                        │
│       │ WAL (Write-Ahead Log)                  │
│       ▼                                        │
│  Debezium                                      │
│       │                                        │
│       ├──► Elasticsearch (খবর search)          │
│       │                                        │
│       ├──► Redis (homepage cache invalidate)   │
│       │                                        │
│       └──► Analytics DB (reader metrics)       │
│                                                │
│  সুবিধা: সাংবাদিক খবর publish করলে             │
│  সাথে সাথে search ও homepage আপডেট হয়!        │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 🔗 Database Connectors

### MySQL Binlog:

```
┌────────────────────────────────────────────────────────┐
│              MySQL Binary Log (binlog)                  │
│                                                        │
│  MySQL সব write operations binlog এ record করে:        │
│                                                        │
│  binlog-000001:                                        │
│  ├── Position 4:    FORMAT_DESCRIPTION                 │
│  ├── Position 124:  INSERT INTO products (...)         │
│  ├── Position 456:  UPDATE products SET price=...      │
│  ├── Position 789:  DELETE FROM products WHERE...      │
│  └── Position 1024: INSERT INTO orders (...)           │
│                                                        │
│  Debezium MySQL Connector:                             │
│  - binlog position track করে                          │
│  - GTID (Global Transaction ID) support               │
│  - ROW format binlog পড়ে                              │
│  - Table schema track করে                              │
│                                                        │
│  MySQL Configuration:                                   │
│  server-id         = 1                                 │
│  log_bin           = mysql-bin                         │
│  binlog_format     = ROW                               │
│  binlog_row_image  = FULL                              │
│  expire_logs_days  = 3                                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### PostgreSQL WAL:

```
┌────────────────────────────────────────────────────────┐
│        PostgreSQL Write-Ahead Log (WAL)                 │
│                                                        │
│  PostgreSQL সব changes WAL এ লেখে (crash recovery):    │
│                                                        │
│  WAL Segment: 000000010000000000000001                  │
│  ├── LSN 0/1500000: INSERT (products)                  │
│  ├── LSN 0/1500120: UPDATE (products)                  │
│  └── LSN 0/1500240: DELETE (products)                  │
│                                                        │
│  Logical Replication Slot:                             │
│  - Debezium একটি replication slot তৈরি করে            │
│  - WAL data decode করে (pgoutput plugin)              │
│  - Position (LSN) track করে                           │
│                                                        │
│  PostgreSQL Configuration:                              │
│  wal_level = logical                                   │
│  max_replication_slots = 4                             │
│  max_wal_senders = 4                                   │
│                                                        │
│  Slot তৈরি:                                            │
│  SELECT * FROM pg_create_logical_replication_slot(      │
│    'debezium_slot', 'pgoutput'                         │
│  );                                                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### MongoDB Oplog:

```
┌────────────────────────────────────────────────────────┐
│              MongoDB Operation Log (oplog)              │
│                                                        │
│  MongoDB replica set এ oplog.rs collection:            │
│                                                        │
│  {                                                     │
│    "ts": Timestamp(1705312456, 1),                     │
│    "op": "u",         // i=insert, u=update, d=delete  │
│    "ns": "shop.products",                              │
│    "o": { "$set": { "price": 84999 } },               │
│    "o2": { "_id": ObjectId("...") }                    │
│  }                                                     │
│                                                        │
│  Debezium MongoDB Connector:                           │
│  - oplog / Change Streams ব্যবহার করে                  │
│  - Resume token track করে                             │
│  - Document-level change capture                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### CDC Consumer - Elasticsearch Sync:

```php
<?php

namespace App\Services\CDC;

use Elasticsearch\Client as ElasticsearchClient;
use Illuminate\Support\Facades\Log;

class ProductSearchSyncService
{
    private ElasticsearchClient $elasticsearch;
    private const INDEX_NAME = 'daraz_products';

    public function __construct(ElasticsearchClient $elasticsearch)
    {
        $this->elasticsearch = $elasticsearch;
    }

    /**
     * Debezium CDC event প্রসেস করো
     * Kafka consumer থেকে call হবে
     */
    public function handleCDCEvent(array $cdcEvent): void
    {
        $operation = $cdcEvent['payload']['op'];
        $before = $cdcEvent['payload']['before'] ?? null;
        $after = $cdcEvent['payload']['after'] ?? null;
        $source = $cdcEvent['payload']['source'];

        Log::info("CDC Event received", [
            'operation' => $operation,
            'table'     => $source['table'],
            'id'        => $after['id'] ?? $before['id'] ?? null,
        ]);

        match ($operation) {
            'c' => $this->handleCreate($after),      // INSERT
            'u' => $this->handleUpdate($after),      // UPDATE
            'd' => $this->handleDelete($before),     // DELETE
            'r' => $this->handleSnapshot($after),    // SNAPSHOT READ
            default => Log::warning("Unknown CDC operation: {$operation}"),
        };
    }

    /**
     * নতুন product তৈরি হলে Elasticsearch এ index করো
     */
    private function handleCreate(array $product): void
    {
        $document = $this->transformToSearchDocument($product);

        $this->elasticsearch->index([
            'index' => self::INDEX_NAME,
            'id'    => $product['id'],
            'body'  => $document,
        ]);

        Log::info("Product indexed in ES", ['id' => $product['id']]);
    }

    /**
     * Product update হলে ES document update করো
     */
    private function handleUpdate(array $product): void
    {
        $document = $this->transformToSearchDocument($product);

        $this->elasticsearch->update([
            'index' => self::INDEX_NAME,
            'id'    => $product['id'],
            'body'  => ['doc' => $document],
        ]);

        Log::info("Product updated in ES", ['id' => $product['id']]);
    }

    /**
     * Product delete হলে ES থেকেও মুছো
     */
    private function handleDelete(array $product): void
    {
        $this->elasticsearch->delete([
            'index' => self::INDEX_NAME,
            'id'    => $product['id'],
        ]);

        Log::info("Product removed from ES", ['id' => $product['id']]);
    }

    private function handleSnapshot(array $product): void
    {
        // Initial snapshot — create এর মতোই handle করো
        $this->handleCreate($product);
    }

    /**
     * MySQL row → Elasticsearch document transform
     */
    private function transformToSearchDocument(array $product): array
    {
        return [
            'name'          => $product['name'],
            'name_bn'       => $product['name_bn'] ?? null,
            'description'   => $product['description'],
            'category_id'   => $product['category_id'],
            'brand'         => $product['brand'],
            'price'         => (float) $product['price'],
            'discount_price'=> (float) ($product['discount_price'] ?? $product['price']),
            'stock'         => (int) $product['stock'],
            'is_active'     => (bool) $product['is_active'],
            'rating'        => (float) ($product['avg_rating'] ?? 0),
            'sold_count'    => (int) ($product['sold_count'] ?? 0),
            'tags'          => json_decode($product['tags'] ?? '[]', true),
            'updated_at'    => $product['updated_at'],
        ];
    }
}
```

### CDC Event Processor with Offset Management:

```php
<?php

namespace App\Services\CDC;

use App\Models\CDCOffset;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class CDCEventProcessor
{
    private array $handlers = [];

    public function registerHandler(string $table, callable $handler): void
    {
        $this->handlers[$table] = $handler;
    }

    /**
     * Kafka message প্রসেস করো with exactly-once guarantee
     */
    public function processMessage(string $topic, int $partition, int $offset, array $cdcEvent): void
    {
        $offsetKey = "{$topic}:{$partition}";

        // Offset চেক করো (exactly-once)
        $lastOffset = $this->getLastProcessedOffset($offsetKey);
        if ($offset <= $lastOffset) {
            Log::debug("Skipping already processed offset", [
                'topic' => $topic, 'offset' => $offset
            ]);
            return;
        }

        $table = $cdcEvent['payload']['source']['table'] ?? null;
        $handler = $this->handlers[$table] ?? null;

        if (!$handler) {
            Log::warning("No handler for table: {$table}");
            $this->updateOffset($offsetKey, $offset);
            return;
        }

        // Transaction এ process + offset update
        DB::transaction(function () use ($handler, $cdcEvent, $offsetKey, $offset) {
            // Business logic
            $handler($cdcEvent);

            // Offset save করো (same transaction)
            $this->updateOffset($offsetKey, $offset);
        });
    }

    private function getLastProcessedOffset(string $key): int
    {
        return (int) CDCOffset::where('offset_key', $key)->value('offset_value') ?? -1;
    }

    private function updateOffset(string $key, int $offset): void
    {
        CDCOffset::updateOrCreate(
            ['offset_key' => $key],
            ['offset_value' => $offset, 'updated_at' => now()]
        );
    }
}
```

### Cache Invalidation via CDC:

```php
<?php

namespace App\Services\CDC;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Log;

class CacheInvalidationService
{
    /**
     * CDC event আসলে সংশ্লিষ্ট cache invalidate করো
     * Daraz: product update → product cache + category cache + homepage cache clear
     */
    public function handleProductChange(array $cdcEvent): void
    {
        $operation = $cdcEvent['payload']['op'];
        $after = $cdcEvent['payload']['after'] ?? null;
        $before = $cdcEvent['payload']['before'] ?? null;

        $product = $after ?? $before;
        $productId = $product['id'];
        $categoryId = $product['category_id'];

        // Product cache invalidate
        Redis::del("product:{$productId}");
        Redis::del("product:{$productId}:details");

        // Category listing cache invalidate
        Redis::del("category:{$categoryId}:products");

        // Price change হলে deal cache ও clear করো
        if ($operation === 'u' && $before && $after) {
            if ($before['price'] !== $after['price']) {
                Redis::del("deals:today");
                Redis::del("category:{$categoryId}:deals");
                Log::info("Price changed, deal caches cleared", [
                    'product_id' => $productId,
                    'old_price'  => $before['price'],
                    'new_price'  => $after['price'],
                ]);
            }
        }

        // Stock 0 হলে homepage "popular items" cache clear
        if ($after && (int) $after['stock'] === 0) {
            Redis::del("homepage:popular");
            Redis::del("category:{$categoryId}:available");
        }

        Log::info("Cache invalidated for product", ['id' => $productId]);
    }
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### CDC Consumer Service (Node.js):

```javascript
// services/CDCConsumerService.js
const { Kafka } = require('kafkajs');
const { Client: ESClient } = require('@elastic/elasticsearch');
const Redis = require('ioredis');

class CDCConsumerService {
    constructor() {
        this.kafka = new Kafka({
            clientId: 'daraz-cdc-consumer',
            brokers: ['kafka-1:9092', 'kafka-2:9092'],
        });

        this.elasticsearch = new ESClient({
            node: 'http://elasticsearch:9200',
        });

        this.redis = new Redis({
            host: 'redis-cluster.daraz.internal',
            port: 6379,
        });

        this.handlers = new Map();
    }

    registerHandler(table, handler) {
        this.handlers.set(table, handler);
    }

    async start() {
        const consumer = this.kafka.consumer({
            groupId: 'daraz-search-sync',
        });

        await consumer.connect();

        // Debezium CDC topics subscribe করো
        await consumer.subscribe({
            topics: [
                'daraz-catalog.products_db.products',
                'daraz-catalog.products_db.categories',
                'daraz-catalog.products_db.brands',
            ],
            fromBeginning: false,
        });

        await consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                try {
                    const cdcEvent = JSON.parse(message.value.toString());
                    await this.processCDCEvent(cdcEvent, topic);
                } catch (error) {
                    console.error('❌ CDC processing error:', error.message);
                    // Dead Letter Queue তে পাঠাও
                    await this.sendToDLQ(topic, message, error);
                }
            },
        });

        console.log('🚀 CDC Consumer started');
    }

    async processCDCEvent(cdcEvent, topic) {
        const { op, before, after, source } = cdcEvent.payload;
        const table = source.table;

        console.log(`📥 CDC Event: ${op} on ${table}`);

        const handler = this.handlers.get(table);
        if (handler) {
            await handler(op, before, after, source);
        }
    }

    async sendToDLQ(topic, message, error) {
        const dlqProducer = this.kafka.producer();
        await dlqProducer.connect();
        await dlqProducer.send({
            topic: `${topic}.dlq`,
            messages: [{
                key: message.key,
                value: message.value,
                headers: {
                    error: error.message,
                    original_topic: topic,
                    failed_at: new Date().toISOString(),
                },
            }],
        });
        await dlqProducer.disconnect();
    }
}

module.exports = CDCConsumerService;
```

### Elasticsearch Sync Handler:

```javascript
// handlers/productSearchHandler.js

class ProductSearchHandler {
    constructor(esClient) {
        this.esClient = esClient;
        this.indexName = 'daraz_products';
    }

    /**
     * Product CDC event handle করো
     * op: c=create, u=update, d=delete, r=read(snapshot)
     */
    async handle(op, before, after, source) {
        switch (op) {
            case 'c': // CREATE
            case 'r': // SNAPSHOT
                await this.indexProduct(after);
                break;

            case 'u': // UPDATE
                await this.updateProduct(after);
                break;

            case 'd': // DELETE
                await this.deleteProduct(before);
                break;

            default:
                console.warn(`Unknown op: ${op}`);
        }
    }

    async indexProduct(product) {
        const document = this.transformToSearchDoc(product);

        await this.esClient.index({
            index: this.indexName,
            id: product.id.toString(),
            body: document,
        });

        console.log(`✅ Indexed product: ${product.id} - ${product.name}`);
    }

    async updateProduct(product) {
        const document = this.transformToSearchDoc(product);

        await this.esClient.update({
            index: this.indexName,
            id: product.id.toString(),
            body: { doc: document },
        });

        console.log(`🔄 Updated product: ${product.id}`);
    }

    async deleteProduct(product) {
        await this.esClient.delete({
            index: this.indexName,
            id: product.id.toString(),
        });

        console.log(`🗑️ Deleted product: ${product.id}`);
    }

    transformToSearchDoc(product) {
        return {
            name: product.name,
            name_bn: product.name_bn || null,
            description: product.description,
            category_id: product.category_id,
            brand: product.brand,
            price: parseFloat(product.price),
            discount_price: parseFloat(product.discount_price || product.price),
            stock: parseInt(product.stock),
            is_active: Boolean(product.is_active),
            rating: parseFloat(product.avg_rating || 0),
            sold_count: parseInt(product.sold_count || 0),
            tags: JSON.parse(product.tags || '[]'),
            suggest: {
                input: [product.name, product.brand].filter(Boolean),
                weight: parseInt(product.sold_count || 0),
            },
            updated_at: product.updated_at,
        };
    }
}

module.exports = ProductSearchHandler;
```

### Real-time Analytics with CDC:

```javascript
// handlers/analyticsHandler.js
const { Pool } = require('pg');

class AnalyticsCDCHandler {
    constructor() {
        // TimescaleDB (PostgreSQL extension) for time-series
        this.timescalePool = new Pool({
            host: 'timescaledb.daraz.internal',
            database: 'analytics',
        });
    }

    /**
     * Order table CDC events → Analytics DB তে real-time sync
     * Daraz: প্রতিটি order change track করো analytics এর জন্য
     */
    async handleOrderChange(op, before, after, source) {
        const order = after || before;
        const timestamp = new Date(source.ts_ms);

        switch (op) {
            case 'c': // নতুন order
                await this.recordNewOrder(order, timestamp);
                break;

            case 'u': // Order status change
                if (before && after && before.status !== after.status) {
                    await this.recordStatusChange(before, after, timestamp);
                }
                break;

            case 'd': // Order cancelled/deleted
                await this.recordCancellation(before, timestamp);
                break;
        }
    }

    async recordNewOrder(order, timestamp) {
        await this.timescalePool.query(
            `INSERT INTO order_metrics (time, order_id, customer_id, amount, category, region)
             VALUES ($1, $2, $3, $4, $5, $6)`,
            [timestamp, order.id, order.customer_id, order.total,
             order.category, order.delivery_region]
        );

        // Real-time dashboard update
        await this.timescalePool.query(
            `INSERT INTO hourly_sales (time, region, total_amount, order_count)
             VALUES (time_bucket('1 hour', $1::timestamptz), $2, $3, 1)
             ON CONFLICT (time, region)
             DO UPDATE SET
                total_amount = hourly_sales.total_amount + EXCLUDED.total_amount,
                order_count = hourly_sales.order_count + 1`,
            [timestamp, order.delivery_region, order.total]
        );
    }

    async recordStatusChange(before, after, timestamp) {
        await this.timescalePool.query(
            `INSERT INTO order_status_changes
             (time, order_id, from_status, to_status, duration_seconds)
             VALUES ($1, $2, $3, $4, $5)`,
            [
                timestamp,
                after.id,
                before.status,
                after.status,
                Math.floor((timestamp - new Date(before.updated_at)) / 1000),
            ]
        );
    }

    async recordCancellation(order, timestamp) {
        await this.timescalePool.query(
            `INSERT INTO cancellations (time, order_id, amount, reason, region)
             VALUES ($1, $2, $3, $4, $5)`,
            [timestamp, order.id, order.total, order.cancel_reason,
             order.delivery_region]
        );
    }
}

module.exports = AnalyticsCDCHandler;
```

---

## 🔧 Schema Evolution ও Advanced Topics

### Schema Evolution Handling:

```
┌────────────────────────────────────────────────────────────┐
│              SCHEMA EVOLUTION with CDC                      │
│                                                            │
│  সমস্যা: DB schema change হলে CDC event format ও পালটায়   │
│                                                            │
│  v1: products (id, name, price)                            │
│  v2: products (id, name, price, discount_price)  ← নতুন!   │
│                                                            │
│  সমাধান:                                                    │
│  ┌────────────────────────────────────────────┐            │
│  │  Schema Registry (Confluent/Apicurio)      │            │
│  │                                            │            │
│  │  Schema v1 ──► Compatible? ──► Schema v2   │            │
│  │                                            │            │
│  │  Rules:                                    │            │
│  │  ✅ Add optional field (backward compat)   │            │
│  │  ✅ Add default value                      │            │
│  │  ❌ Remove required field                  │            │
│  │  ❌ Change field type                      │            │
│  └────────────────────────────────────────────┘            │
│                                                            │
│  Debezium auto-detects schema changes                      │
│  এবং schema history topic এ record করে                    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Exactly-Once Delivery:

```javascript
// Kafka Transactions দিয়ে exactly-once CDC processing
const producer = kafka.producer({
    idempotent: true,
    transactionalId: 'cdc-processor-1',
});

await producer.connect();

// Transaction এ consume + produce + offset commit
const transaction = await producer.transaction();
try {
    // CDC event process করে output topic এ লিখো
    await transaction.send({
        topic: 'processed-products',
        messages: [{ key: productId, value: JSON.stringify(processed) }],
    });

    // Consumer offset commit (same transaction)
    await transaction.sendOffsets({
        consumerGroupId: 'cdc-processor-group',
        topics: [{
            topic: 'daraz-catalog.products_db.products',
            partitions: [{ partition: 0, offset: (currentOffset + 1).toString() }],
        }],
    });

    await transaction.commit();
} catch (error) {
    await transaction.abort();
    throw error;
}
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন CDC ব্যবহার করবেন:

| পরিস্থিতি | উদাহরণ |
|-----------|---------|
| Search index sync | MySQL → Elasticsearch (Daraz products) |
| Cache invalidation | DB change → Redis clear (Prothom Alo) |
| Real-time analytics | OLTP → TimescaleDB/ClickHouse |
| Microservice events | Monolith→microservice migration |
| Data warehouse sync | Production → BigQuery/Redshift |
| Audit logging | সব DB changes track করা |
| Cross-DC replication | Dhaka DC → Singapore DC sync |

### ❌ কখন CDC ব্যবহার করবেন না:

| পরিস্থিতি | কারণ |
|-----------|-------|
| Simple CRUD notification | Overkill — application event যথেষ্ট |
| Low-volume data | Polling সহজ ও যথেষ্ট |
| Schema-less needs | CDC schema-aware, flexible নয় |
| No Kafka infrastructure | Setup cost অনেক বেশি |
| Aggregated events দরকার | CDC row-level, aggregate নয় |

### 💡 Best Practices:

1. **Log retention configure করুন** — binlog/WAL যেন Debezium পড়ার আগে delete না হয়
2. **Schema Registry ব্যবহার করুন** — schema evolution safe রাখতে
3. **Monitoring** — connector lag, replication slot size monitor করুন
4. **Snapshot strategy** — initial load এর জন্য snapshot mode ঠিক করুন
5. **Topic naming convention** — `{prefix}.{database}.{table}` follow করুন
6. **Consumer idempotency** — CDC events duplicate আসতে পারে, inbox pattern ব্যবহার করুন

---

## 🎓 সারসংক্ষেপ

```
┌────────────────────────────────────────────────────┐
│              CDC - মূল কথা                          │
├────────────────────────────────────────────────────┤
│                                                    │
│  কি: Database changes কে events এ পরিণত করা       │
│  কেন: Real-time sync, no dual writes              │
│  কিভাবে: Transaction log (binlog/WAL) পড়ে         │
│  Tool: Debezium + Kafka Connect                    │
│  সেরা ব্যবহার: Search sync, cache invalidation    │
│                                                    │
│  CDC = "Database কে event source বানানো"           │
│                                                    │
└────────────────────────────────────────────────────┘
```
