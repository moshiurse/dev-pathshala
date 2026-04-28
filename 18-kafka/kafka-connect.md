# 📘 Kafka Connect — ডাটা ইন্টিগ্রেশন ফ্রেমওয়ার্ক

> Kafka Connect হলো Kafka ইকোসিস্টেমের সবচেয়ে শক্তিশালী ডাটা ইন্টিগ্রেশন টুল —
> কোড না লিখেই বিভিন্ন সিস্টেমের মধ্যে ডাটা মুভ করা যায়।

---

## 📖 Kafka Connect কী?

Kafka Connect হলো একটি **ডাটা ইন্টিগ্রেশন ফ্রেমওয়ার্ক** যা বিভিন্ন ডাটা সোর্স এবং
ডেস্টিনেশনের মধ্যে **কোনো কোড না লিখেই** ডাটা স্ট্রিম করতে পারে।

### 🏠 বাংলাদেশি উদাহরণ দিয়ে বুঝি

ধরুন, **Daraz** এর সিস্টেমে:
- MySQL ডাটাবেসে প্রোডাক্ট ডাটা আছে
- Elasticsearch এ সার্চ ইনডেক্স দরকার
- S3 তে ডাটা লেক বানাতে হবে
- MongoDB তে ক্যাটালগ ডাটা আছে

এখন এই সবগুলোকে ম্যানুয়ালি কানেক্ট করতে গেলে প্রতিটার জন্য আলাদা কোড লিখতে হতো।
**Kafka Connect** এটাকে **কনফিগারেশন-ভিত্তিক** করে দিয়েছে — শুধু JSON কনফিগ দিন, ব্যস!

### 🔌 বাংলা অ্যানালজি

Kafka Connect হলো **ইউনিভার্সাল অ্যাডাপ্টার/কনভার্টার** এর মতো।

যেমন আপনার ফোনের চার্জার — একই অ্যাডাপ্টার দিয়ে ভারতের সকেট, UK সকেট, US সকেট
সবখানে চার্জ করতে পারেন। Kafka Connect ও তেমনই — MySQL, MongoDB, Elasticsearch,
S3 যেকোনো সিস্টেমের সাথে **একই ফ্রেমওয়ার্ক** দিয়ে কানেক্ট করতে পারেন।

### 📊 Kafka ইকোসিস্টেমে Connect এর ভূমিকা

```
                         Kafka ইকোসিস্টেম
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  ┌──────────────┐                  ┌──────────────┐     │
  │  │   Source      │    ┌────────┐   │    Sink      │     │
  │  │  Connectors   │───→│ Kafka  │──→│  Connectors  │     │
  │  │  (ডাটা আনে)  │    │ Broker │   │ (ডাটা পাঠায়)│     │
  │  └──────────────┘    └────────┘   └──────────────┘     │
  │         ↑                │               │              │
  │    External              │          External            │
  │    Systems          Kafka Streams    Systems            │
  │  (MySQL, Mongo)     (প্রসেসিং)   (ES, S3, HDFS)       │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

### 🎯 Kafka Connect এর মূল সুবিধাসমূহ

| সুবিধা | বিবরণ |
|---------|--------|
| **No Code** | কোড না লিখেই ডাটা পাইপলাইন তৈরি |
| **Scalable** | ডিস্ট্রিবিউটেড মোডে অনেক ওয়ার্কার চালানো যায় |
| **Fault Tolerant** | ওয়ার্কার ফেইল করলে অটো রিব্যালেন্স হয় |
| **Schema Support** | Schema Registry দিয়ে ডাটা ভ্যালিডেশন |
| **Pluggable** | শত শত কানেক্টর প্লাগইন পাওয়া যায় |

---

## 📊 Source vs Sink কানেক্টর

Kafka Connect এ দুই ধরনের কানেক্টর আছে:

### 📥 Source Connector — বাইরে থেকে Kafka তে ডাটা আনে

```
External System ──→ Source Connector ──→ Kafka Topic
```

উদাহরণ: **bKash** এর MySQL ডাটাবেস থেকে ট্রানজেকশন ডাটা Kafka টপিকে পাঠানো।

### 📤 Sink Connector — Kafka থেকে বাইরে ডাটা পাঠায়

```
Kafka Topic ──→ Sink Connector ──→ External System
```

উদাহরণ: Kafka টপিক থেকে **Daraz** এর Elasticsearch সার্চ ইনডেক্সে ডাটা পাঠানো।

### 📊 সম্পূর্ণ ডাটা ফ্লো ডায়াগ্রাম

```
┌──────────────┐    Source       ┌─────────────┐      Sink       ┌──────────────┐
│              │   Connector     │             │    Connector     │              │
│  MySQL DB    │──────────────→ │   Kafka     │ ──────────────→ │ Elastic      │
│  (bKash)     │──────────────→ │   Topics    │ ──────────────→ │ Search       │
│              │                 │             │                  │              │
│  MongoDB     │──────────────→ │  ┌───────┐  │ ──────────────→ │ Amazon S3    │
│  (Pathao)    │──────────────→ │  │ topic1│  │ ──────────────→ │ (Data Lake)  │
│              │                 │  │ topic2│  │                  │              │
│  CSV Files   │──────────────→ │  │ topic3│  │ ──────────────→ │ PostgreSQL   │
│  (Reports)   │──────────────→ │  └───────┘  │ ──────────────→ │ (Analytics)  │
│              │                 │             │                  │              │
└──────────────┘                 └─────────────┘                 └──────────────┘
```

### 🏠 বাস্তব উদাহরণ — Daraz ইকমার্স

```
                    Daraz ডাটা পাইপলাইন

  ┌─────────┐  Source   ┌───────┐   Sink   ┌───────────────┐
  │ Products│────────→ │       │ ────────→│ Elasticsearch │  ← সার্চ
  │  MySQL  │          │       │          └───────────────┘
  ├─────────┤          │ Kafka │   Sink   ┌───────────────┐
  │ Orders  │────────→ │ Broker│ ────────→│   Amazon S3   │  ← ডাটা লেক
  │  MySQL  │          │       │          └───────────────┘
  ├─────────┤          │       │   Sink   ┌───────────────┐
  │ Reviews │────────→ │       │ ────────→│   MongoDB     │  ← অ্যানালিটিক্স
  │  MySQL  │          │       │          └───────────────┘
  └─────────┘          └───────┘
```

---

## 🔌 জনপ্রিয় কানেক্টর সমূহ

### 📋 কানেক্টর ক্যাটালগ

| কানেক্টর | টাইপ | ব্যবহার | বাংলাদেশি উদাহরণ |
|-----------|-------|---------|-------------------|
| **JDBC Source** | Source | রিলেশনাল DB থেকে ডাটা আনা | bKash MySQL → Kafka |
| **JDBC Sink** | Sink | Kafka থেকে রিলেশনাল DB তে লেখা | Kafka → Nagad PostgreSQL |
| **Elasticsearch Sink** | Sink | সার্চ ইনডেক্স তৈরি | Kafka → Daraz সার্চ |
| **S3 Sink** | Sink | ডাটা লেক তৈরি | Kafka → GP ডাটা লেক |
| **MongoDB Source** | Source | MongoDB থেকে CDC | Pathao MongoDB → Kafka |
| **MongoDB Sink** | Sink | Kafka থেকে MongoDB তে লেখা | Kafka → Pathao Analytics |
| **Debezium MySQL** | Source | MySQL binlog CDC | bKash ট্রানজেকশন CDC |
| **Debezium PostgreSQL** | Source | PostgreSQL WAL CDC | Nagad ট্রানজেকশন CDC |
| **Debezium MongoDB** | Source | MongoDB oplog CDC | Pathao রাইড CDC |
| **FileStream Source** | Source | ফাইল থেকে ডাটা (টেস্ট) | লোকাল টেস্টিং |
| **FileStream Sink** | Sink | ফাইলে ডাটা লেখা (টেস্ট) | লোকাল টেস্টিং |
| **HTTP Source** | Source | REST API থেকে ডাটা | এক্সটার্নাল API পোলিং |
| **HDFS Sink** | Sink | Hadoop ফাইল সিস্টেমে লেখা | বিগ ডাটা পাইপলাইন |

### 🔧 কানেক্টর ইনস্টল করা

```bash
# Confluent Hub থেকে কানেক্টর ইনস্টল
confluent-hub install debezium/debezium-connector-mysql:latest
confluent-hub install confluentinc/kafka-connect-elasticsearch:latest
confluent-hub install confluentinc/kafka-connect-s3:latest
confluent-hub install mongodb/kafka-connect-mongodb:latest

# অথবা ম্যানুয়ালি plugin.path এ JAR রাখুন
# connect-distributed.properties:
plugin.path=/usr/share/java,/usr/share/confluent-hub-components
```

---

## 📋 Schema Registry ও Avro

### 🤔 Schema Registry কী?

Schema Registry হলো একটি **সেন্ট্রালাইজড সার্ভিস** যা Kafka মেসেজের স্কিমা
(ডাটা স্ট্রাকচার) সংরক্ষণ ও ম্যানেজ করে।

### 🏠 বাংলাদেশি উদাহরণ

ধরুন **bKash** এর ট্রানজেকশন ডাটা Kafka তে যাচ্ছে। প্রডিউসার ও কনজিউমার দুজনকেই
জানতে হবে ডাটা কেমন দেখতে (ফিল্ড নাম, টাইপ ইত্যাদি)। Schema Registry এই
"চুক্তিপত্র" সংরক্ষণ করে।

### 📊 Schema Registry ইন্টারেকশন

```
                    Schema Registry ফ্লো

  ┌──────────┐                  ┌────────────────┐
  │ Producer │──── ① Schema ──→│                │
  │ (Source  │     Register     │    Schema      │
  │ Connect) │←── ② Schema ───│   Registry     │
  │          │     ID           │  (স্কিমা সংরক্ষণ) │
  └──────────┘                  └────────────────┘
       │                               ↑
       │ ③ Data + Schema ID            │ ④ Get Schema
       ↓                               │    by ID
  ┌──────────┐                  ┌──────────────┐
  │  Kafka   │ ─── ⑤ Data ──→ │  Consumer    │
  │  Broker  │     + Schema ID │  (Sink       │
  │          │                  │   Connect)   │
  └──────────┘                  └──────────────┘
```

### 📝 Avro স্কিমা উদাহরণ — bKash ট্রানজেকশন

```json
{
  "type": "record",
  "name": "BkashTransaction",
  "namespace": "com.bkash.events",
  "fields": [
    {"name": "transaction_id", "type": "string"},
    {"name": "sender_wallet",  "type": "string"},
    {"name": "receiver_wallet","type": "string"},
    {"name": "amount",         "type": "double"},
    {"name": "currency",       "type": {"type": "string", "default": "BDT"}},
    {"name": "type",           "type": {"type": "enum", "name": "TxnType",
                                "symbols": ["SEND_MONEY", "CASH_OUT", "PAYMENT", "ADD_MONEY"]}},
    {"name": "timestamp",      "type": "long"}
  ]
}
```

### 🔄 Schema Evolution (স্কিমা বিবর্তন)

সময়ের সাথে সাথে ডাটা স্ট্রাকচার পরিবর্তন হয়। Schema Registry তিন ধরনের
কম্প্যাটিবিলিটি সাপোর্ট করে:

| কম্প্যাটিবিলিটি | অর্থ | উদাহরণ |
|-------------------|-------|--------|
| **BACKWARD** | নতুন স্কিমা পুরোনো ডাটা পড়তে পারবে | নতুন ফিল্ড যোগ (ডিফল্ট সহ) |
| **FORWARD** | পুরোনো স্কিমা নতুন ডাটা পড়তে পারবে | ফিল্ড রিমুভ করা |
| **FULL** | উভয় দিকেই কম্প্যাটিবল | সবচেয়ে নিরাপদ |
| **NONE** | কোনো চেক নেই | ⚠️ প্রোডাকশনে এড়িয়ে চলুন |

```bash
# Schema Registry কনফিগারেশন
# connect-distributed.properties এ যোগ করুন:
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081

value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://schema-registry:8081
```

---

## 🔄 Single Message Transforms (SMTs)

### 🤔 SMT কী?

SMT হলো **হালকা ডাটা ট্রান্সফর্মেশন** যা কানেক্টরের ভেতরেই চালানো যায়।
প্রতিটি মেসেজে আলাদা আলাদা করে ট্রান্সফর্মেশন প্রয়োগ হয়।

> ⚠️ জটিল ট্রান্সফর্মেশনের জন্য Kafka Streams ব্যবহার করুন, SMT নয়।

### 📋 জনপ্রিয় SMT সমূহ

| SMT | কাজ | উদাহরণ |
|-----|------|--------|
| **InsertField** | নতুন ফিল্ড যোগ | টাইমস্ট্যাম্প বা টপিক নাম যোগ |
| **ReplaceField** | ফিল্ড রিনেম/ড্রপ | সেনসিটিভ ফিল্ড বাদ দেওয়া |
| **MaskField** | ফিল্ড মাস্ক করা | ক্রেডিট কার্ড নম্বর মাস্ক |
| **TimestampRouter** | টপিক নামে তারিখ যোগ | `orders` → `orders-2024-01-15` |
| **RegexRouter** | টপিক নাম পরিবর্তন | রিজেক্স দিয়ে টপিক রাউটিং |
| **ValueToKey** | ভ্যালু থেকে কী সেট | ডাটাবেস ID কে মেসেজ কী বানানো |
| **ExtractField** | নেস্টেড ফিল্ড বের করা | স্ট্রাক্ট থেকে নির্দিষ্ট ফিল্ড |
| **Flatten** | নেস্টেড স্ট্রাক্ট ফ্ল্যাট করা | `address.city` → `address_city` |

### 📝 SMT কনফিগারেশন উদাহরণ

```json
{
  "name": "bkash-transactions-source",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",

    "transforms": "addTimestamp,maskWallet,routeByDate",

    "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addTimestamp.timestamp.field": "processed_at",

    "transforms.maskWallet.type": "org.apache.kafka.connect.transforms.MaskField$Value",
    "transforms.maskWallet.fields": "sender_wallet,receiver_wallet",
    "transforms.maskWallet.replacement": "****",

    "transforms.routeByDate.type": "org.apache.kafka.connect.transforms.TimestampRouter",
    "transforms.routeByDate.topic.format": "${topic}-${timestamp}",
    "transforms.routeByDate.timestamp.format": "yyyy-MM-dd"
  }
}
```

### 🔗 SMT চেইনিং

একাধিক SMT একসাথে চেইন করা যায়। এগুলো **যে ক্রমে লেখা হয় সেই ক্রমে** এক্সিকিউট হয়:

```
মেসেজ → Transform1 → Transform2 → Transform3 → Kafka/External System
```

---

## 🏗️ Distributed vs Standalone মোড

Kafka Connect দুটি মোডে চালানো যায়:

### 📌 Standalone মোড

- **একটি মাত্র প্রসেস** চলে
- **টেস্টিং ও ডেভেলপমেন্ট** এর জন্য উপযুক্ত
- অফসেট লোকাল ফাইলে সংরক্ষণ হয়

```properties
# connect-standalone.properties
bootstrap.servers=localhost:9092
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
offset.storage.file.filename=/var/kafka-connect/offsets.dat
```

```bash
# Standalone মোডে চালানো
connect-standalone.sh connect-standalone.properties connector1.properties
```

### 📌 Distributed মোড

- **একাধিক ওয়ার্কার** একসাথে চলে
- **প্রোডাকশন** এর জন্য উপযুক্ত
- অফসেট Kafka টপিকে সংরক্ষণ হয়
- ওয়ার্কার ফেইল করলে **অটো রিব্যালেন্স** হয়

```properties
# connect-distributed.properties
bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
group.id=connect-cluster

key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://schema-registry:8081

# ইন্টারনাল টপিক কনফিগারেশন
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status

config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

rest.port=8083
plugin.path=/usr/share/java,/usr/share/confluent-hub-components
```

### 📊 Distributed মোড আর্কিটেকচার

```
                  Distributed Kafka Connect ক্লাস্টার

  ┌──────────────────────────────────────────────────────────┐
  │                 Connect ক্লাস্টার (group.id)             │
  │                                                          │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
  │  │  Worker 1    │  │  Worker 2    │  │  Worker 3    │   │
  │  │              │  │              │  │              │   │
  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │   │
  │  │ │ Task A-1 │ │  │ │ Task A-2 │ │  │ │ Task B-1 │ │   │
  │  │ │ Task B-2 │ │  │ │ Task C-1 │ │  │ │ Task C-2 │ │   │
  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │   │
  │  │  REST :8083  │  │  REST :8083  │  │  REST :8083  │   │
  │  └──────────────┘  └──────────────┘  └──────────────┘   │
  │         ↕                  ↕                  ↕          │
  │  ┌───────────────────────────────────────────────────┐   │
  │  │            Kafka Broker ক্লাস্টার                  │   │
  │  │  connect-configs  connect-offsets  connect-status  │   │
  │  └───────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

### ⚖️ Standalone vs Distributed তুলনা

| বৈশিষ্ট্য | Standalone | Distributed |
|------------|-----------|-------------|
| **ওয়ার্কার সংখ্যা** | ১টি | একাধিক |
| **Fault Tolerance** | ❌ নেই | ✅ অটো রিব্যালেন্স |
| **অফসেট সংরক্ষণ** | লোকাল ফাইল | Kafka টপিক |
| **স্কেলিং** | ❌ সম্ভব নয় | ✅ ওয়ার্কার যোগ করুন |
| **কানেক্টর ম্যানেজমেন্ট** | কমান্ড লাইন | REST API |
| **ব্যবহার** | ডেভ/টেস্ট | প্রোডাকশন |

---

## 📝 কনফিগারেশন উদাহরণ

### 1️⃣ MySQL Source Connector — Daraz প্রোডাক্ট টেবিল

```json
{
  "name": "daraz-products-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://mysql-primary:3306/daraz_catalog",
    "connection.user": "kafka_connect",
    "connection.password": "${file:/secrets/mysql-password.txt:password}",

    "table.whitelist": "products,categories,brands",
    "mode": "timestamp+incrementing",
    "timestamp.column.name": "updated_at",
    "incrementing.column.name": "id",

    "topic.prefix": "daraz.catalog.",
    "poll.interval.ms": 5000,

    "transforms": "createKey,addSource",
    "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.createKey.fields": "id",
    "transforms.addSource.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addSource.static.field": "source_system",
    "transforms.addSource.static.value": "daraz-catalog-db",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "tasks.max": 3
  }
}
```

### 2️⃣ Elasticsearch Sink Connector — সার্চ ইনডেক্স

```json
{
  "name": "daraz-search-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "connection.url": "http://elasticsearch:9200",

    "topics": "daraz.catalog.products",
    "type.name": "_doc",
    "key.ignore": false,

    "schema.ignore": false,
    "compact.map.entries": true,

    "write.method": "upsert",
    "max.buffered.records": 5000,
    "batch.size": 2000,
    "flush.timeout.ms": 120000,
    "linger.ms": 1000,

    "behavior.on.null.values": "delete",
    "behavior.on.malformed.documents": "warn",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "tasks.max": 2
  }
}
```

### 3️⃣ S3 Sink Connector — ডাটা লেক

```json
{
  "name": "gp-datalake-s3-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "s3.region": "ap-southeast-1",
    "s3.bucket.name": "grameenphone-datalake",

    "topics": "gp.cdr.voice,gp.cdr.data,gp.cdr.sms",

    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "parquet.codec": "snappy",

    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
    "locale": "en-US",
    "timezone": "Asia/Dhaka",
    "partition.duration.ms": 3600000,

    "flush.size": 100000,
    "rotate.interval.ms": 600000,
    "rotate.schedule.interval.ms": 3600000,

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "tasks.max": 6
  }
}
```

### 4️⃣ MongoDB Source Connector — Pathao রাইড ডাটা

```json
{
  "name": "pathao-rides-source",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
    "connection.uri": "mongodb://kafka_user:password@mongo1:27017,mongo2:27017,mongo3:27017",

    "database": "pathao_rides",
    "collection": "ride_events",

    "pipeline": "[{\"$match\": {\"operationType\": {\"$in\": [\"insert\", \"update\"]}}}]",

    "publish.full.document.only": true,
    "copy.existing": true,

    "topic.prefix": "pathao.rides",
    "topic.suffix": "",

    "output.format.value": "schema",
    "output.schema.infer.value": true,

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "tasks.max": 2
  }
}
```

---

## 🔍 Debezium CDC Deep Dive

### 🤔 Change Data Capture (CDC) কী?

CDC হলো ডাটাবেসের **প্রতিটি পরিবর্তন (INSERT, UPDATE, DELETE)** ক্যাপচার করে
ইভেন্ট হিসেবে স্ট্রিম করা। এটা পোলিং এর চেয়ে অনেক বেশি কার্যকর।

### 🏠 বাংলাদেশি উদাহরণ

**bKash** এর ক্ষেত্রে — প্রতি সেকেন্ডে হাজার হাজার ট্রানজেকশন হচ্ছে। প্রতিটি
ট্রানজেকশনের রিয়েল-টাইম নোটিফিকেশন, ফ্রড ডিটেকশন, অ্যানালিটিক্স দরকার।
CDC ব্যবহার করলে ডাটাবেসে যেই মুহূর্তে INSERT হবে, সেই মুহূর্তেই Kafka তে ইভেন্ট চলে যাবে।

### ⚖️ পোলিং vs CDC তুলনা

| বৈশিষ্ট্য | JDBC পোলিং | Debezium CDC |
|------------|------------|--------------|
| **লেটেন্সি** | পোল ইন্টারভালের উপর নির্ভর | মিলিসেকেন্ড |
| **DB লোড** | ক্যোয়ারি চালায় বারবার | শুধু binlog পড়ে |
| **DELETE ক্যাপচার** | ❌ পারে না | ✅ পারে |
| **স্কিমা পরিবর্তন** | সীমিত | সম্পূর্ণ ক্যাপচার |
| **Historical ডাটা** | শুধু বর্তমান অবস্থা | আগে/পরে দুটোই |

### 📊 Debezium আর্কিটেকচার

```
┌──────────────────┐              ┌────────────────┐              ┌─────────────┐
│                  │              │                │              │             │
│    MySQL DB      │── binlog ──→│   Debezium     │── events ──→│   Kafka     │
│   (bKash txn)   │              │   Connector    │              │   Topics    │
│                  │              │                │              │             │
│  ┌────────────┐  │              │  ┌──────────┐  │              │  ┌───────┐  │
│  │ binlog     │──┼──────────── │  │ Snapshot │  │              │  │txn    │  │
│  │ (WAL)      │  │              │  │ Reader   │  │              │  │events │  │
│  └────────────┘  │              │  └──────────┘  │              │  └───────┘  │
│                  │              │  ┌──────────┐  │              │  ┌───────┐  │
│  ┌────────────┐  │              │  │ Binlog   │  │              │  │schema │  │
│  │ Table Data │  │              │  │ Reader   │  │              │  │changes│  │
│  └────────────┘  │              │  └──────────┘  │              │  └───────┘  │
│                  │              │                │              │             │
└──────────────────┘              └────────────────┘              └─────────────┘
```

### 📝 Debezium ইভেন্ট স্ট্রাকচার

প্রতিটি Debezium ইভেন্টে **before** (আগের অবস্থা) এবং **after** (পরের অবস্থা) থাকে:

```json
{
  "schema": { "..." : "..." },
  "payload": {
    "before": {
      "id": 1001,
      "wallet_id": "01712345678",
      "balance": 5000.00
    },
    "after": {
      "id": 1001,
      "wallet_id": "01712345678",
      "balance": 4500.00
    },
    "source": {
      "version": "2.4.0",
      "connector": "mysql",
      "name": "bkash-prod",
      "ts_ms": 1705305600000,
      "db": "bkash_wallet",
      "table": "transactions",
      "server_id": 1,
      "file": "mysql-bin.000003",
      "pos": 12345
    },
    "op": "u",
    "ts_ms": 1705305600123,
    "transaction": null
  }
}
```

> `op` ফিল্ড: `c` = CREATE, `u` = UPDATE, `d` = DELETE, `r` = READ (স্ন্যাপশট)

### 📝 MySQL CDC কনফিগারেশন — bKash ট্রানজেকশন

```json
{
  "name": "bkash-transactions-cdc",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-primary",
    "database.port": 3306,
    "database.user": "debezium",
    "database.password": "${file:/secrets/debezium-mysql.txt:password}",

    "database.server.id": 184054,
    "topic.prefix": "bkash",

    "database.include.list": "bkash_wallet,bkash_payments",
    "table.include.list": "bkash_wallet.transactions,bkash_payments.payment_requests",

    "include.schema.changes": true,
    "schema.history.internal.kafka.bootstrap.servers": "kafka1:9092,kafka2:9092",
    "schema.history.internal.kafka.topic": "bkash.schema-changes",

    "snapshot.mode": "initial",
    "snapshot.locking.mode": "minimal",

    "decimal.handling.mode": "double",
    "time.precision.mode": "connect",

    "tombstones.on.delete": true,
    "provide.transaction.metadata": true,

    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "tasks.max": 1
  }
}
```

### 📸 Snapshot মোড

| মোড | বিবরণ |
|------|--------|
| **initial** | প্রথমবার পুরো টেবিল কপি, তারপর binlog ফলো |
| **schema_only** | শুধু স্কিমা নেয়, ডাটা নেয় না |
| **never** | স্ন্যাপশট নেয় না, সরাসরি binlog থেকে শুরু |
| **when_needed** | অফসেট না পেলে স্ন্যাপশট নেয় |
| **initial_only** | শুধু স্ন্যাপশট নেয়, পরে binlog ফলো করে না |

### 🔧 স্কিমা পরিবর্তন হ্যান্ডলিং

Debezium ডাটাবেসের DDL পরিবর্তন (ALTER TABLE, CREATE TABLE ইত্যাদি) ট্র্যাক করে।
`include.schema.changes: true` সেট করলে এই পরিবর্তনগুলো আলাদা টপিকে রেকর্ড হয়।

```
# স্কিমা পরিবর্তন টপিক
bkash.schema-changes

# ডাটা টপিক (অটোমেটিক আপডেটেড স্কিমা)
bkash.bkash_wallet.transactions
```

---

## ⚠️ এরর হ্যান্ডলিং

### 📌 Error Tolerance

Kafka Connect কানেক্টরে দুই ধরনের এরর টলারেন্স সেট করা যায়:

```json
{
  "errors.tolerance": "none",
  "errors.tolerance": "all"
}
```

| মোড | আচরণ |
|------|--------|
| **none** (ডিফল্ট) | একটি এরর হলেই কানেক্টর থেমে যায় |
| **all** | এরর স্কিপ করে পরবর্তী রেকর্ডে চলে যায় |

### 💀 Dead Letter Queue (DLQ)

এরর হওয়া মেসেজগুলোকে আলাদা টপিকে পাঠানো যায় যেন পরে অ্যানালাইজ করা যায়:

```json
{
  "name": "daraz-orders-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "topics": "daraz.orders",

    "errors.tolerance": "all",

    "errors.deadletterqueue.topic.name": "daraz.orders.dlq",
    "errors.deadletterqueue.topic.replication.factor": 3,

    "errors.deadletterqueue.context.headers.enable": true,

    "errors.log.enable": true,
    "errors.log.include.messages": true
  }
}
```

### 📊 DLQ ফ্লো

```
                  Error Handling ফ্লো

  ┌──────────┐      ┌──────────┐      ┌───────────┐
  │  Kafka   │─────→│  Sink    │─────→│ External  │   ← সফল মেসেজ
  │  Topic   │      │Connector │      │  System   │
  └──────────┘      └──────────┘      └───────────┘
                         │
                    ❌ এরর হলে
                         │
                         ↓
                    ┌──────────┐      ┌───────────┐
                    │   DLQ    │─────→│ মনিটরিং  │   ← ফেইলড মেসেজ
                    │  Topic   │      │ ও অ্যালার্ট│     পরে রিপ্রসেস
                    └──────────┘      └───────────┘
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | উদাহরণ |
|------------|--------|
| ✅ স্ট্যান্ডার্ড ডাটা ইন্টিগ্রেশন | MySQL → Kafka → Elasticsearch |
| ✅ CDC (Change Data Capture) | bKash রিয়েল-টাইম ট্রানজেকশন ট্র্যাকিং |
| ✅ ডাটা লেক তৈরি | Grameenphone CDR → S3 Parquet |
| ✅ সার্চ ইনডেক্স সিঙ্ক | Daraz প্রোডাক্ট → Elasticsearch |
| ✅ ডাটাবেস মাইগ্রেশন/রেপ্লিকেশন | MySQL → PostgreSQL |
| ✅ লগ অ্যাগ্রিগেশন | অ্যাপ্লিকেশন লগ → Kafka → ELK |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন নয় | বিকল্প |
|------------|---------|--------|
| ❌ জটিল ট্রান্সফর্মেশন | SMT সীমিত | Kafka Streams / Flink |
| ❌ কাস্টম বিজনেস লজিক | Connect ফ্রেমওয়ার্ক সিম্পল | কাস্টম কনজিউমার/প্রডিউসার |
| ❌ রিয়েল-টাইম অ্যাগ্রিগেশন | Connect শুধু মুভ করে | Kafka Streams / ksqlDB |
| ❌ মাল্টি-টপিক জয়েন | সাপোর্ট নেই | Kafka Streams |
| ❌ কন্ডিশনাল রাউটিং | সীমিত সাপোর্ট | কাস্টম কনজিউমার |

---

## 🎯 বেস্ট প্র্যাকটিস

### 1️⃣ প্রোডাকশন কনফিগারেশন

```properties
# রেপ্লিকেশন ফ্যাক্টর কমপক্ষে ৩
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

# কনভার্টার সঠিকভাবে সেট করুন
key.converter=io.confluent.connect.avro.AvroConverter
value.converter=io.confluent.connect.avro.AvroConverter
```

### 2️⃣ মনিটরিং

```bash
# কানেক্টর স্ট্যাটাস চেক
curl -s http://localhost:8083/connectors/daraz-products-source/status | jq .

# সব কানেক্টর লিস্ট
curl -s http://localhost:8083/connectors | jq .

# কানেক্টর রিস্টার্ট
curl -X POST http://localhost:8083/connectors/daraz-products-source/restart

# নির্দিষ্ট টাস্ক রিস্টার্ট
curl -X POST http://localhost:8083/connectors/daraz-products-source/tasks/0/restart
```

### 3️⃣ সিকিউরিটি

- পাসওয়ার্ড কখনো সরাসরি কনফিগে দেবেন না
- `ConfigProvider` ব্যবহার করুন:

```properties
# FileConfigProvider ব্যবহার
config.providers=file
config.providers.file.class=org.apache.kafka.common.config.provider.FileConfigProvider

# কানেক্টর কনফিগে রেফারেন্স
"database.password": "${file:/secrets/db-credentials.properties:password}"
```

### 4️⃣ পারফরম্যান্স টিউনিং

| সেটিং | সুপারিশ | কারণ |
|--------|---------|-------|
| `tasks.max` | সোর্স টেবিল/পার্টিশন সংখ্যা | প্যারালেলিজম বাড়ায় |
| `batch.size` | ২০০০-৫০০০ | থ্রুপুট বাড়ায় |
| `flush.size` (S3) | ৫০,০০০-১,০০,০০০ | বড় ফাইল তৈরি করে |
| `poll.interval.ms` | ৫০০০-১০,০০০ | JDBC পোলিং ইন্টারভাল |
| `consumer.max.poll.records` | ৫০০ | ব্যাচ সাইজ কনজিউমারে |

### 5️⃣ টেস্টিং কৌশল

```bash
# ১. প্রথমে Standalone মোডে টেস্ট করুন
connect-standalone.sh connect-standalone.properties my-connector.properties

# ২. ড্রাই রান — শুধু কনফিগ ভ্যালিডেশন
curl -X PUT http://localhost:8083/connector-plugins/JdbcSourceConnector/config/validate \
  -H "Content-Type: application/json" \
  -d @my-connector-config.json

# ৩. কানেক্টর তৈরি (Distributed মোডে)
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @my-connector-config.json

# ৪. স্ট্যাটাস মনিটর
watch -n 5 'curl -s http://localhost:8083/connectors/my-connector/status | jq .'
```

### 6️⃣ সাধারণ সমস্যা ও সমাধান

| সমস্যা | কারণ | সমাধান |
|---------|-------|--------|
| কানেক্টর FAILED | কনফিগ ভুল বা কানেকশন সমস্যা | লগ চেক, কনফিগ ঠিক করুন |
| ডাটা মিসিং | অফসেট সমস্যা | অফসেট রিসেট করুন |
| স্লো পারফরম্যান্স | টাস্ক কম | `tasks.max` বাড়ান |
| Schema এরর | স্কিমা মিসম্যাচ | Schema Registry চেক করুন |
| OOM (আউট অফ মেমোরি) | ব্যাচ সাইজ বেশি | `batch.size` কমান |

---

## 🔗 পরবর্তী টপিক

➡️ [Kafka Security — সিকিউরিটি ও অথেনটিকেশন](./kafka-security.md)

---

## 📚 সারসংক্ষেপ

```
Kafka Connect মূল ধারণা:

  ১. কোনো কোড ছাড়াই ডাটা পাইপলাইন      → JSON কনফিগারেশন দিয়ে
  ২. Source Connector                      → বাইরে থেকে Kafka তে ডাটা আনে
  ৩. Sink Connector                        → Kafka থেকে বাইরে ডাটা পাঠায়
  ৪. Schema Registry                       → ডাটা স্ট্রাকচার ম্যানেজমেন্ট
  ৫. SMT (Single Message Transforms)       → হালকা ট্রান্সফর্মেশন
  ৬. Debezium CDC                          → রিয়েল-টাইম ডাটাবেস পরিবর্তন ক্যাপচার
  ৭. Distributed মোড                       → প্রোডাকশনে ব্যবহার করুন
  ৮. Dead Letter Queue                     → এরর মেসেজ সংরক্ষণ
```

> **মনে রাখুন:** Kafka Connect ডাটা **মুভ** করে, **প্রসেস** করে না।
> জটিল প্রসেসিং এর জন্য Kafka Streams বা Flink ব্যবহার করুন।
