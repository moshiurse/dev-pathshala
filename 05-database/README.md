# 📘 ডাটাবেস ডিজাইন

> ডাটাবেস মডেলিং, অপটিমাইজেশন ও সেরা অনুশীলনসমূহ।
> প্রতিটি টপিকে Raw SQL, PHP (Laravel Eloquent), JavaScript (Sequelize/Knex) উদাহরণ এবং গভীর বিশ্লেষণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | নরমালাইজেশন | [normalization.md](./normalization.md) | UNF→1NF→2NF→3NF→BCNF→4NF→5NF, Denormalization, NoSQL Embedding |
| ২ | ইনডেক্সিং | [indexing.md](./indexing.md) | B-Tree/Hash/Composite/Covering/Full-Text, EXPLAIN Analysis, JSON Indexing |
| ৩ | SQL অপটিমাইজেশন | [sql-optimization.md](./sql-optimization.md) | N+1 সমস্যা, Cursor Pagination, Locking, Partitioning, Query Caching |
| ৪ | ট্রানজ্যাকশন ও ACID | [transactions.md](./transactions.md) | Isolation Levels, MVCC, Distributed TX, 2PC, Saga, Optimistic/Pessimistic Lock |
| ৫ | NoSQL ডাটাবেস | [nosql.md](./nosql.md) | MongoDB, Redis, Cassandra, Neo4j, Elasticsearch, CAP Theorem, Polyglot Persistence |
| ৬ | ডাটাবেস রিলেশনশিপ | [relationships.md](./relationships.md) | 1:1, 1:N, M:N, Polymorphic, Self-Referencing (4 Tree Strategies), Soft Deletes |
| ৭ | মাইগ্রেশন স্ট্র্যাটেজি | [migration.md](./migration.md) | Zero-Downtime Migration, Large Table, Multi-tenant, Rollback, Cross-DB |
| ৮ | রেপ্লিকেশন | [replication.md](./replication.md) | Master-Slave, Master-Master, Sync vs Async, Failover |
| ৯ | শার্ডিং | [sharding.md](./sharding.md) | Horizontal vs Vertical, Shard Key, Consistent Hashing |
| ১০ | কানেকশন পুলিং | [connection-pooling.md](./connection-pooling.md) | Pool Configuration, PDO Persistent, pg-pool |
| ১১ | কোয়েরি এক্সিকিউশন প্ল্যান | [query-execution-plan.md](./query-execution-plan.md) | EXPLAIN/ANALYZE, Index Usage, Optimization |
| ১২ | টাইম-সিরিজ ডাটাবেস | [time-series-db.md](./time-series-db.md) | InfluxDB, TimescaleDB, Retention Policy, Downsampling |
| ১৩ | গ্রাফ ডাটাবেস | [graph-database.md](./graph-database.md) | Neo4j, Cypher Query, Graph Traversal, Social Network |
| ১৪ | ফুল-টেক্সট সার্চ | [full-text-search.md](./full-text-search.md) | Elasticsearch, Inverted Index, Analyzer, Relevance Scoring |
| ১৫ | ORM | [orm.md](./orm.md) | Active Record vs Data Mapper, Eloquent, Doctrine, Prisma, TypeORM, N+1 Problem, Identity Map |
| ১৬ | Elasticsearch | [elasticsearch.md](./elasticsearch.md) | Cluster/Shards, Mapping, BM25, Bangla Analyzer, kNN/Hybrid Search, ILM, Daraz Case Study |
| ১৭ | Solr | [solr.md](./solr.md) | SolrCloud, ZooKeeper, edismax, JSON Facet API, Solr vs Elasticsearch, Migration Patterns |
| ১৮ | Redis ডিপ ডাইভ | [redis-deep-dive.md](./redis-deep-dive.md) | Data Structures, Pub/Sub, Streams, Lua, Sentinel, Cluster, Cache Patterns, Distributed Lock |

---

## 🗺️ টপিক সম্পর্ক ডায়াগ্রাম

```
    ┌──────────────────┐
    │  Normalization    │ ◄── ডিজাইনের ভিত্তি
    │  (স্কিমা ডিজাইন)  │
    └────────┬─────────┘
             │
    ┌────────▼─────────┐     ┌──────────────────┐
    │  Relationships    │────►│  Indexing          │
    │  (টেবিল সম্পর্ক)  │     │  (পারফরম্যান্স)    │
    └────────┬─────────┘     └────────┬─────────┘
             │                        │
             ▼                        ▼
    ┌──────────────────┐     ┌──────────────────┐
    │  Transactions     │     │  SQL Optimization  │
    │  (ডাটা সুরক্ষা)   │     │  (কোয়েরি টিউনিং)  │
    └────────┬─────────┘     └──────────────────┘
             │
    ┌────────▼─────────┐     ┌──────────────────┐
    │  NoSQL            │     │  Migration         │
    │  (বিকল্প ডাটাবেস) │     │  (স্কিমা বিবর্তন)  │
    └──────────────────┘     └──────────────────┘
```

---

## 📖 প্রতিটি টপিকে যা পাবেন

- ✅ বাংলায় বিস্তারিত ব্যাখ্যা ও বাস্তব জীবনের analogy
- ✅ ASCII ডায়াগ্রাম (ER, B-Tree, MVCC, CAP Triangle)
- ✅ Raw SQL (MySQL + PostgreSQL)
- ✅ PHP Laravel (Eloquent, Migration, Query Builder)
- ✅ JavaScript (Sequelize, Knex, Mongoose)
- ✅ Advanced scenarios (Zero-downtime, Distributed TX, Polyglot Persistence)
- ✅ সুবিধা (Pros) ও অসুবিধা (Cons) টেবিল
- ✅ সাধারণ ভুল ও Anti-Patterns
- ✅ টেস্টিং (PHPUnit + Jest)
- ✅ Decision Guide ও Comparison Tables
- ✅ বাংলাদেশ context (bKash, Daraz, Pathao উদাহরণ)
