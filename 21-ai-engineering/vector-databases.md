# 🧭 Vector Databases (ভেক্টর ডাটাবেস ও ANN)

> **Vector Database** হলো এমন ডাটাবেস যা high-dimensional embedding (যেমন 768/1536-D float vector) সংরক্ষণ করে এবং **Approximate Nearest Neighbor (ANN)** এর মাধ্যমে millisecond-এ "সবচেয়ে কাছের" vector গুলো বের করে।
> Daraz-এ "নীল শাড়ি ৫০০ টাকার নিচে" সার্চ, Pathao support agent-এ FAQ retrieval, bKash চ্যাটবটে policy lookup — সবকিছুর পেছনে vector DB।

---

## 📖 সূচিপত্র

- [Vector DB কী এবং কেন](#-vector-db-কী-এবং-কেন)
- [Distance Metrics](#-distance-metrics)
- [ANN Algorithms](#-ann-algorithms-deep-dive)
- [Indexing Trade-offs](#-indexing-trade-offs)
- [Filter Pushdown / Hybrid Search](#-filter-pushdown--hybrid-metadata--vector)
- [Vendor Comparison Matrix](#-vendor-comparison-matrix)
- [Self-hosted vs Managed](#-self-hosted-vs-managed)
- [Sharding ও Replication](#-sharding-ও-replication)
- [Bangla Embedding Model Selection](#-bangla-embedding-model-selection)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#-কখন-ব্যবহার-করবেন--করবেন-না)
- [Pitfalls](#-common-pitfalls)
- [Bangladesh Real-world Example](#-bangladesh-real-world-daraz-catalog-1m-products)
- [Production Checklist](#-production-checklist)

---

## 🎯 Vector DB কী এবং কেন

```
Traditional DB (Postgres):
─────────────────────────
SELECT * FROM products WHERE name LIKE '%শাড়ি%' AND price < 500;

❌ "জামদানি" সার্চ করলে "tangail saree" পাবে না
❌ Synonym, semantic similarity ধরতে পারে না
❌ "আমার বউয়ের জন্য সুন্দর শাড়ি" → কেমন ক্যোয়েরি?

Vector DB:
──────────
1. "জামদানি saree" → embedding model → [0.12, -0.45, 0.78, ...] (1536-D)
2. ANN index-এ search → top-K closest vectors
3. Result: jamdani, tangail, banarasi, katan — সব semantic ভাবে relevant!

✅ Multilingual (Bangla + English mix)
✅ Semantic understanding
✅ Sub-100ms latency on millions of vectors
```

### কখন দরকার?

| Use case                       | Traditional | Vector DB |
| ------------------------------ | ----------- | --------- |
| Exact ID lookup                | ✅          | ❌        |
| `WHERE price < 500`            | ✅          | ➕ filter  |
| "এই প্রোডাক্টের মতো আরো"      | ❌          | ✅        |
| Bangla→English semantic match  | ❌          | ✅        |
| RAG context retrieval          | ❌          | ✅        |
| Image similarity (CLIP)        | ❌          | ✅        |
| Anomaly detection              | ❌          | ✅        |

---

## 📐 Distance Metrics

```
ভেক্টর A = [a1, a2, ..., an]
ভেক্টর B = [b1, b2, ..., bn]

1. Cosine Similarity (সবচেয়ে common, text embedding-এ):
   cos(θ) = (A · B) / (|A| · |B|)
   Range: -1 (opposite) ... +1 (identical)
   👉 magnitude ignore করে, শুধু direction দেখে
   👉 OpenAI embeddings, sentence-transformers default

2. Dot Product:
   A · B = Σ aᵢ × bᵢ
   👉 যদি vector গুলো normalized হয়, cosine = dot product
   👉 GPU-তে দ্রুততম

3. Euclidean (L2):
   √(Σ (aᵢ - bᵢ)²)
   👉 image embedding, geographic data

4. Manhattan (L1):
   Σ |aᵢ - bᵢ|
   👉 sparse data
```

> **নিয়ম:** OpenAI `text-embedding-3-small/large`, Cohere multilingual, BGE — সবগুলোতে **cosine** বা **normalized dot product** ব্যবহার করুন। ভুল metric মানে ভুল ranking।

---

## 🔬 ANN Algorithms Deep Dive

### কেন Approximate? (Brute Force-এর সমস্যা)

```
N = 10 million vectors, d = 1536
Brute Force (Flat):
  প্রতি query-তে N × d = 15.36 billion multiplications
  ≈ 3-5 seconds on CPU
  Recall: 100% (perfect)

ANN লক্ষ্য: 0.95-0.99 recall, কিন্তু 1000x faster
```

### 1️⃣ Flat (Brute Force)

```
Index: কিছু না, শুধু সব vector store
Query: O(N × d)
Recall: 100%
Use: < 100K vectors, ground truth evaluation
Memory: N × d × 4 bytes (float32)
```

### 2️⃣ IVF (Inverted File Index)

```
Build:
1. K-means clustering → nlist centroids (e.g., 4096)
2. প্রতিটি vector কে nearest centroid এ assign
3. centroid → [vec1, vec2, ...] mapping (inverted file)

Query:
1. query vector কে nearest nprobe centroids খুঁজে (e.g., 16)
2. শুধু ঐ clusters-এ brute force search

Tradeoff:
  nprobe ↑ → recall ↑, latency ↑
  nlist ≈ √N (rule of thumb)
```

```
ASCII visualization:
┌─────────────────────────────────────────┐
│   .  .   ●─────●           .   .        │
│  .   ●  / cluster1 \   .  ●─●           │
│   ●──●          ●     . / c3 \          │
│   \ c2 /     ●──────●  ●──────●         │
│    ●──●     . cluster3      Q (query)   │
│   .                       ↗             │
│         . . .   ●─●    nprobe=2:        │
│              \ c4 /     search c3, c4   │
│               ●─●                        │
└─────────────────────────────────────────┘
```

### 3️⃣ IVF-PQ (Product Quantization)

```
সমস্যা: 1M × 1536 × 4 = 6 GB RAM 😱

PQ সমাধান:
1. Vector কে m সাবভেক্টরে ভাগ করো (e.g., 1536 → 8 × 192-D)
2. প্রতিটি subspace-এ K-means → 256 centroids → 1 byte index
3. Original vector → m bytes!
   1536 × 4 = 6144 bytes → 8 bytes (768x compression!)

Query: precomputed distance table → asymmetric distance computation (ADC)

Tradeoff:
  Compression ↑ → Recall ↓
  m=8, ksub=256 typical → ~85-92% recall
```

### 4️⃣ HNSW (Hierarchical Navigable Small World) — 👑 most popular

```
Idea: multi-layer skip list, কিন্তু graph-এ
        Layer 2:  ●───────●───────●        (hubs only)
                  │       │       │
        Layer 1:  ●───●───●───●───●        (more)
                  │   │   │   │   │
        Layer 0:  ●─●─●─●─●─●─●─●─●        (all)

Build:
- প্রতি node-কে random level assign (exponential decay)
- প্রতি level-এ M nearest neighbors connect
- ef_construction parameter: build quality

Query:
- Top layer-এ greedy search → entry point নামাও
- Layer 0-এ ef_search candidates → top-K return

Params:
  M:               16-64    (per-layer connections)
  ef_construction: 100-500  (build quality, slower)
  ef_search:       50-200   (query quality, latency)

Performance:
  Build: O(N × log N × ef_construction)
  Query: O(log N × ef_search)
  Memory: ~1.5x flat (graph overhead)
  Recall: 0.95-0.99 easily
```

> **কেন HNSW জিতে যায়:** No training (vs IVF k-means), incremental insert সাপোর্ট করে, latency very predictable। Pinecone, Weaviate, Qdrant, pgvector 0.5+, Elasticsearch — সবার default।

### 5️⃣ ScaNN (Google)

```
Idea: anisotropic vector quantization + asymmetric hashing
- IVF-PQ + learned quantization
- Distance approximation এ MSE নয়, "ranking loss" optimize
- TPU/GPU-তে blazing fast
- Open source, কিন্তু integration কঠিন
- Vertex AI Matching Engine internally ScaNN
```

### 6️⃣ DiskANN (Microsoft)

```
সমস্যা: HNSW সব RAM-এ। Billion-scale-এ অসম্ভব।

DiskANN:
- Vamana graph (HNSW-similar but disk-aware)
- Compressed PQ vectors RAM-এ (filtering)
- Full-precision vectors SSD-তে (reranking)
- Single SSD-তে 1B+ vectors, < 10ms latency
- Used by: Bing, Microsoft 365, Azure AI Search

Tradeoff: SSD I/O bound, but cost-effective
```

### Algorithm Comparison

| Algo      | Build      | Query      | Memory       | Recall  | Updates | Disk |
| --------- | ---------- | ---------- | ------------ | ------- | ------- | ---- |
| Flat      | O(1)       | O(N·d)     | N·d·4 B      | 100%    | ✅      | ➕    |
| IVF       | O(N·iter)  | O(N/nlist) | ~1.05x       | 90-95%  | ⚠️      | ➕    |
| IVF-PQ    | O(N·iter)  | medium     | ~0.05x       | 85-92%  | ⚠️      | ✅    |
| HNSW      | O(N log N) | O(log N)   | ~1.5x        | 95-99%  | ✅      | ❌    |
| ScaNN     | medium     | very fast  | ~0.1x        | 92-97%  | ⚠️      | ✅    |
| DiskANN   | slow       | fast       | tiny RAM     | 95-98%  | ⚠️      | ✅    |

---

## ⚖️ Indexing Trade-offs

```
চারটি বিরোধী চাপ — যেকোনো ৩টা পান, ৪র্থটার দাম দেন:

      Recall (accuracy)
            ▲
            │
            │
     ◄──────┼──────►
   Memory   │   Latency
            │
            ▼
       Build time

উদাহরণ:
- HNSW high M=64, ef=400: ★ Recall ★ Latency ✗ Memory ✗ Build
- IVF-PQ aggressive: ★ Memory ★ Latency ✗ Recall ✗ Build (k-means)
- Flat: ★ Recall ★ Build ✗ Latency ✗ Memory at scale
```

### Decision Framework

```
N (vector count)?
├── < 100K           → Flat (brute force in app memory)
├── 100K - 10M       → HNSW (Qdrant/pgvector/Weaviate)
├── 10M - 100M       → HNSW + sharding, or IVF-PQ
├── 100M - 1B        → DiskANN, IVF-PQ, or ScaNN
└── > 1B             → DiskANN + custom infra (Bing scale)

Latency requirement?
├── < 10ms           → HNSW in RAM, no rerank
├── 10-100ms         → HNSW + cross-encoder rerank top-50
└── > 100ms ok       → DiskANN, hybrid retrieval

Update pattern?
├── Read-heavy, batch update      → IVF-PQ (rebuild ok)
├── Real-time inserts             → HNSW (incremental)
└── Frequent deletes              → Be careful! HNSW deletes mark + tombstone
```

---

## 🔍 Filter Pushdown / Hybrid Metadata + Vector

বাস্তব query: "নীল শাড়ি, দাম ৳৫০০-৳১৫০০, রেটিং ≥ ৪.০, ঢাকায় ডেলিভারি"

### তিনটি Strategy

```
1) Pre-filter (filter then ANN):
   ─────────────────────────────
   WHERE price BETWEEN 500 AND 1500 AND rating >= 4.0
   → candidate set (e.g., 50K vectors)
   → brute force on subset
   ✅ Accurate
   ❌ Slow if filter result large

2) Post-filter (ANN then filter):
   ─────────────────────────────
   ANN top-1000 → filter → top-K
   ✅ Fast ANN
   ❌ যদি filter selective (1%), top-1000 থেকে কিছুই নাও পেতে পারে

3) Filter-aware ANN (pushdown):
   ─────────────────────────────
   HNSW traversal-এ প্রতি node check করে filter pass করে কিনা
   Skip non-matching nodes during graph walk
   ✅ Best of both worlds
   ✅ Qdrant, Weaviate, Milvus, pgvector এ implemented
```

### Selectivity Decision

```
Filter selectivity = matching_rows / total_rows

selectivity > 50%   → Post-filter ok
selectivity 5-50%   → Filter-aware ANN (best)
selectivity < 5%    → Pre-filter (small candidate set)
selectivity < 0.1%  → Skip ANN, just SQL!
```

### Hybrid Search (Vector + BM25 + Metadata)

```
Query: "ঢাকা area-তে cheap halal restaurant"

Pipeline:
  ┌─────────────────────────────────┐
  │  BM25 (lexical) → top-100       │  → score_bm25
  │  Vector (semantic) → top-100    │  → score_vec
  │  Metadata: area=Dhaka, halal=1  │  → mandatory filter
  └─────────────────────────────────┘
                  │
                  ▼
   Reciprocal Rank Fusion (RRF):
   score = Σ 1 / (k + rank_i),  k=60
                  │
                  ▼
              Top-K → rerank (cross-encoder)
```

---

## 🏆 Vendor Comparison Matrix

| Vendor                    | Type          | Algo(s)         | Filter | Hybrid | BD-friendly                 | License       | Cost (1M vec, 1536-D) |
| ------------------------- | ------------- | --------------- | ------ | ------ | --------------------------- | ------------- | --------------------- |
| **Pinecone**              | Managed SaaS  | Proprietary     | ✅     | ✅     | US/EU only, network latency | Closed        | ~$70/mo (Serverless)  |
| **Weaviate**              | OSS + Cloud   | HNSW            | ✅     | ✅ BM25 | self-host on AWS Singapore  | BSD-3         | $0 self / $25+ cloud  |
| **Qdrant**                | OSS + Cloud   | HNSW            | ✅✅   | ✅     | Rust, fast, self-host friendly | Apache-2   | $0 self / $40+ cloud  |
| **Milvus / Zilliz**       | OSS + Cloud   | HNSW, IVF, DiskANN | ✅  | ✅     | enterprise-grade            | Apache-2      | $0 self / $99+ cloud  |
| **Chroma**                | OSS embedded  | HNSW            | ✅     | ❌     | prototype-only, not prod    | Apache-2      | $0 self               |
| **Vespa**                 | OSS + Cloud   | HNSW            | ✅     | ✅✅   | Yahoo-grade, complex setup  | Apache-2      | self-host             |
| **pgvector** (Postgres)   | OSS extension | HNSW, IVF       | ✅✅   | ✅ tsvector | already-using-Postgres   | PostgreSQL    | $0 (existing PG)      |
| **Redis Stack**           | OSS + Cloud   | HNSW, FLAT      | ✅     | ✅     | ক্যাশের সাথে combo          | RSALv2/SSPL   | existing Redis        |
| **Elasticsearch**         | OSS + Cloud   | HNSW            | ✅✅   | ✅✅ BM25 | already running ES       | Elastic-2.0   | existing ES cluster   |
| **MongoDB Atlas Vector**  | Managed       | HNSW            | ✅     | ✅     | document DB combo           | SSPL/Closed   | included in Atlas     |
| **Typesense**             | OSS           | HNSW            | ✅     | ✅     | search-first                | GPL-3         | $0 self               |
| **OpenSearch**            | OSS           | HNSW, IVF       | ✅     | ✅     | AWS-native                  | Apache-2      | self-host             |

### Decision Cheat Sheet (Bangladesh context)

```
আপনার stack?
├── Already on Postgres (Laravel/Symfony) → pgvector ⭐
│       রিজন: কোনো নতুন infra না, ACID transaction, single DB backup
├── Already on Elasticsearch → ES dense_vector + BM25 hybrid ⭐
├── Need < 5ms latency, < 10M vectors → Qdrant self-hosted
├── Need managed, ok with $$$ → Pinecone or Weaviate Cloud
├── Billion-scale (rare in BD) → Milvus + DiskANN
└── Just prototyping → Chroma (in-memory)

BD-specific consideration:
- AWS Singapore latency from Dhaka: ~30-50ms
- Pinecone (US-East): ~250ms — 😱 RAG-এ ব্যবহার-অযোগ্য
- Self-host on local DC (Aamra, BDCOM) বা AWS Mumbai/Singapore
```

---

## ☁️ Self-hosted vs Managed

```
Managed (Pinecone, Weaviate Cloud, Zilliz):
  ✅ No ops burden
  ✅ Auto-scaling, backups
  ❌ Vendor lock-in
  ❌ Egress cost (RAG every query → embedding + DB call)
  ❌ Data residency (Bangladesh Bank guidelines: financial data)
  ❌ Cost balloons at scale ($1000+/mo @ 10M vec common)

Self-hosted (Qdrant, Milvus, pgvector):
  ✅ Cheaper at scale
  ✅ Data stays on-prem (bKash, Nagad regulatory)
  ✅ Customization (custom distance, plugins)
  ❌ DevOps: monitoring, backup, upgrade
  ❌ Capacity planning
  ❌ HA setup complex (Milvus etcd, Qdrant raft)

Cost Math (10M vec, 1536-D):
  Pinecone p2.x1: $720/mo, ~50 QPS
  Qdrant on c5.4xlarge (16 vCPU, 32 GB): $500/mo, 200+ QPS
  pgvector on existing RDS: $0 marginal (if RAM available)
```

---

## 🌐 Sharding ও Replication

### Sharding Strategies

```
1) Random Sharding (most common):
   shard_id = hash(doc_id) % N_shards
   - Even distribution
   - Query: scatter-gather to all shards, merge top-K
   - Each shard maintains own HNSW

2) Semantic Sharding:
   - K-means coarse clustering → shard per cluster
   - Query: route to nearest 1-2 shards (like IVF coarse)
   - Less network, but skewed shards possible

3) Tenant Sharding (multi-tenant SaaS):
   shard_id = hash(tenant_id) % N
   - Strong isolation
   - Daraz seller A's data ≠ seller B
```

### Replication

```
Primary-Replica:
  - Writes → primary, async replicate → replicas
  - Reads → replicas (load balance)
  - Qdrant supports via Raft consensus
  - Milvus uses etcd + message broker (Pulsar)

Multi-region:
  - Read replicas in Singapore + Mumbai
  - Bangladesh user → Singapore (closer)
  - Eventual consistency typical (vector DB-এ ok)
```

### Capacity Planning Formula

```
Memory per shard (HNSW):
  M_total = N × d × 4 × 1.5 + N × M × 2 × 4
           ↑     ↑    ↑     ↑     ↑
           vec count, dim, fp32, hnsw overhead, graph edges

উদাহরণ: 5M vec, 1536-D, M=32
  = 5M × 1536 × 4 × 1.5 + 5M × 32 × 8
  = 46 GB + 1.3 GB ≈ 48 GB

→ 64 GB RAM instance প্রয়োজন (with OS, queries)
→ যদি বেশি, shard করুন
```

---

## 🇧🇩 Bangla Embedding Model Selection

বাংলা ও বাংলিশ (code-mixed) এর জন্য ভুল model = ভুল retrieval।

| Model                                        | Dim   | Bangla? | Multilingual | License | Latency (CPU) |
| -------------------------------------------- | ----- | ------- | ------------ | ------- | ------------- |
| **OpenAI `text-embedding-3-small`**          | 1536  | ✅✅    | ✅ 100+ lang | Closed  | API ~50ms     |
| **OpenAI `text-embedding-3-large`**          | 3072  | ✅✅✅  | ✅           | Closed  | API ~80ms     |
| **Cohere `embed-multilingual-v3`**           | 1024  | ✅✅✅  | ✅ 100+ lang | Closed  | API ~40ms     |
| **`BAAI/bge-m3`** (HF)                       | 1024  | ✅✅    | ✅ 100+ lang | MIT     | ~100ms self   |
| **`intfloat/multilingual-e5-large`**         | 1024  | ✅✅    | ✅ 100 lang  | MIT     | ~100ms self   |
| **`sentence-transformers/paraphrase-multilingual-mpnet-base-v2`** | 768 | ✅ | ✅ 50 lang | Apache | ~50ms self  |
| **`sagorsarker/bangla-bert-base`** (BD)      | 768   | ✅✅✅  | ❌ Bangla only | MIT  | ~80ms self    |
| **`csebuetnlp/banglabert`** (BUET)           | 768   | ✅✅✅  | ❌ Bangla    | CC-BY-NC | ~80ms self  |

### Recommendation Matrix

```
Use case
├── Bangla-only domain (legal docs, news)     → banglabert / bangla-bert-base
├── Bangla + English mixed (e-commerce)       → bge-m3 ⭐ or Cohere v3
├── Bangla + English + chat (support)         → OpenAI 3-small (cheap, good)
├── Best quality, budget OK                   → OpenAI 3-large or Cohere v3
└── Self-hosted, GPU-less                     → multilingual-e5-large quantized
```

> **গুরুত্বপূর্ণ:** "Hello" এর English embedding ≠ "হ্যালো" এর Bangla embedding multi-lingual model ছাড়া। Cross-lingual retrieval চাইলে **bge-m3, Cohere multilingual v3, বা OpenAI** ছাড়া উপায় নাই।

### Test on Your Data

```
Always benchmark:
1. ১০০টা real queries (Bangla, English, mixed) লিখুন
2. Manual relevant docs label করুন (top-5)
3. Embedding model দিয়ে index → measure Recall@10, MRR
4. Compare 3-4 models, খরচ-quality balance
```

---

## 💻 কোড উদাহরণ

### 1) pgvector Setup + Insert + ANN Query (Node.js)

```bash
# Install
npm i pg openai
psql -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

```sql
-- schema.sql
CREATE TABLE products (
    id           BIGSERIAL PRIMARY KEY,
    sku          TEXT UNIQUE NOT NULL,
    name_bn      TEXT NOT NULL,
    name_en      TEXT,
    category     TEXT,
    price        NUMERIC(10,2),
    rating       REAL,
    in_stock     BOOLEAN DEFAULT true,
    embedding    vector(1536),
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- HNSW index (pgvector >= 0.5)
CREATE INDEX products_embedding_hnsw
  ON products USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200);

-- Filter columns indexed separately
CREATE INDEX products_category_idx ON products(category);
CREATE INDEX products_price_idx    ON products(price);
```

```javascript
// indexer.js — Node.js indexing pipeline
import { Pool } from 'pg';
import OpenAI from 'openai';
import pgvector from 'pgvector/pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const oai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function init() {
  const client = await pool.connect();
  await pgvector.registerType(client);
  client.release();
}

async function embed(texts) {
  // Batch embed for efficiency (up to 2048 inputs per call)
  const res = await oai.embeddings.create({
    model: 'text-embedding-3-small',
    input: texts,
  });
  return res.data.map(d => d.embedding);
}

async function indexProducts(products) {
  // products: [{sku, name_bn, name_en, category, price, rating}, ...]
  const texts = products.map(p =>
    `${p.name_bn} ${p.name_en ?? ''} ${p.category}`.trim()
  );

  // Batch embed in chunks of 100
  const embeddings = [];
  for (let i = 0; i < texts.length; i += 100) {
    const batch = texts.slice(i, i + 100);
    embeddings.push(...(await embed(batch)));
  }

  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    for (let i = 0; i < products.length; i++) {
      const p = products[i];
      await client.query(
        `INSERT INTO products (sku, name_bn, name_en, category, price, rating, embedding)
         VALUES ($1,$2,$3,$4,$5,$6,$7)
         ON CONFLICT (sku) DO UPDATE
           SET name_bn=$2, name_en=$3, category=$4, price=$5, rating=$6, embedding=$7`,
        [p.sku, p.name_bn, p.name_en, p.category, p.price, p.rating,
         pgvector.toSql(embeddings[i])]
      );
    }
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

// Search with filter pushdown
async function search(query, { category, maxPrice, minRating, k = 10 } = {}) {
  const [qVec] = await embed([query]);

  const filters = [];
  const params = [pgvector.toSql(qVec)];
  if (category) {
    filters.push(`category = $${params.length + 1}`);
    params.push(category);
  }
  if (maxPrice) {
    filters.push(`price <= $${params.length + 1}`);
    params.push(maxPrice);
  }
  if (minRating) {
    filters.push(`rating >= $${params.length + 1}`);
    params.push(minRating);
  }
  filters.push('in_stock = true');
  params.push(k);

  const sql = `
    SELECT sku, name_bn, price, rating,
           1 - (embedding <=> $1) AS similarity
    FROM products
    WHERE ${filters.join(' AND ')}
    ORDER BY embedding <=> $1
    LIMIT $${params.length}
  `;
  // <=>  cosine distance (lower = more similar)
  // <#>  negative inner product
  // <->  L2

  const { rows } = await pool.query(sql, params);
  return rows;
}

// Tune query-time recall
async function searchHighRecall(query, opts) {
  await pool.query('SET LOCAL hnsw.ef_search = 100'); // default 40
  return search(query, opts);
}

await init();
const results = await search('নীল রঙের সুতি শাড়ি', {
  category: 'saree',
  maxPrice: 1500,
  minRating: 4.0,
  k: 20,
});
console.log(results);
```

### 2) Qdrant (Node.js Client)

```bash
docker run -p 6333:6333 -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage qdrant/qdrant
```

```javascript
import { QdrantClient } from '@qdrant/js-client-rest';
import OpenAI from 'openai';

const qdrant = new QdrantClient({ url: 'http://localhost:6333' });
const oai = new OpenAI();

const COLLECTION = 'daraz_products';

// 1) Create collection (one-time)
await qdrant.createCollection(COLLECTION, {
  vectors: { size: 1536, distance: 'Cosine' },
  hnsw_config: { m: 32, ef_construct: 256 },
  optimizers_config: { default_segment_number: 4 },
  // Quantization for memory savings
  quantization_config: {
    scalar: { type: 'int8', quantile: 0.99, always_ram: true }
  },
});

// 2) Create payload indexes for filter pushdown
await qdrant.createPayloadIndex(COLLECTION, {
  field_name: 'category', field_schema: 'keyword',
});
await qdrant.createPayloadIndex(COLLECTION, {
  field_name: 'price', field_schema: 'float',
});

// 3) Upsert
async function upsertProducts(products) {
  const texts = products.map(p => `${p.name_bn} ${p.name_en} ${p.category}`);
  const { data } = await oai.embeddings.create({
    model: 'text-embedding-3-small', input: texts,
  });

  await qdrant.upsert(COLLECTION, {
    wait: true,
    points: products.map((p, i) => ({
      id: p.id,
      vector: data[i].embedding,
      payload: {
        sku: p.sku, name_bn: p.name_bn, name_en: p.name_en,
        category: p.category, price: p.price, rating: p.rating,
        seller_id: p.seller_id, in_stock: p.in_stock,
      },
    })),
  });
}

// 4) Hybrid search with filter pushdown
async function search(query, filters = {}, k = 10) {
  const { data } = await oai.embeddings.create({
    model: 'text-embedding-3-small', input: [query],
  });

  const must = [];
  if (filters.category)
    must.push({ key: 'category', match: { value: filters.category } });
  if (filters.maxPrice)
    must.push({ key: 'price', range: { lte: filters.maxPrice } });
  if (filters.minRating)
    must.push({ key: 'rating', range: { gte: filters.minRating } });
  must.push({ key: 'in_stock', match: { value: true } });

  return qdrant.search(COLLECTION, {
    vector: data[0].embedding,
    filter: { must },
    limit: k,
    with_payload: true,
    params: { hnsw_ef: 128, exact: false }, // tune recall
    score_threshold: 0.65, // drop noise
  });
}

const out = await search('জামদানি শাড়ি লাল', {
  category: 'saree', maxPrice: 5000, minRating: 4.0,
});
```

### 3) Pinecone (Node.js)

```javascript
import { Pinecone } from '@pinecone-database/pinecone';
import OpenAI from 'openai';

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
const oai = new OpenAI();

// One-time
await pc.createIndex({
  name: 'daraz-products',
  dimension: 1536,
  metric: 'cosine',
  spec: {
    serverless: { cloud: 'aws', region: 'ap-southeast-1' }, // Singapore
  },
});

const index = pc.index('daraz-products').namespace('bd-catalog');

// Upsert
async function upsert(products) {
  const texts = products.map(p => `${p.name_bn} ${p.name_en}`);
  const { data } = await oai.embeddings.create({
    model: 'text-embedding-3-small', input: texts,
  });
  await index.upsert(products.map((p, i) => ({
    id: String(p.id),
    values: data[i].embedding,
    metadata: {
      name_bn: p.name_bn, category: p.category,
      price: p.price, rating: p.rating,
    },
  })));
}

// Query
async function query(text, filters = {}, k = 10) {
  const { data } = await oai.embeddings.create({
    model: 'text-embedding-3-small', input: [text],
  });
  return index.query({
    vector: data[0].embedding,
    topK: k,
    includeMetadata: true,
    filter: {
      ...(filters.category && { category: { $eq: filters.category } }),
      ...(filters.maxPrice && { price: { $lte: filters.maxPrice } }),
      ...(filters.minRating && { rating: { $gte: filters.minRating } }),
    },
  });
}
```

### 4) PHP + pgvector (PDO)

```php
<?php
// composer.json: { "require": { "openai-php/client": "^0.10" } }

declare(strict_types=1);

final class VectorStore
{
    public function __construct(
        private \PDO $pdo,
        private \OpenAI\Client $oai,
        private string $model = 'text-embedding-3-small'
    ) {}

    /** @param string[] $texts */
    public function embed(array $texts): array
    {
        $res = $this->oai->embeddings()->create([
            'model' => $this->model,
            'input' => $texts,
        ]);
        return array_map(fn($d) => $d->embedding, $res->embeddings);
    }

    /** Convert PHP float[] to pgvector literal: '[0.1,0.2,...]' */
    private function toVectorLiteral(array $vec): string
    {
        return '[' . implode(',', $vec) . ']';
    }

    public function upsert(array $products): void
    {
        $texts = array_map(
            fn($p) => trim("{$p['name_bn']} {$p['name_en']} {$p['category']}"),
            $products
        );
        $embeddings = $this->embed($texts);

        $this->pdo->beginTransaction();
        try {
            $stmt = $this->pdo->prepare(<<<SQL
                INSERT INTO products
                  (sku, name_bn, name_en, category, price, rating, embedding)
                VALUES (?, ?, ?, ?, ?, ?, ?::vector)
                ON CONFLICT (sku) DO UPDATE
                  SET name_bn = EXCLUDED.name_bn,
                      embedding = EXCLUDED.embedding
            SQL);

            foreach ($products as $i => $p) {
                $stmt->execute([
                    $p['sku'], $p['name_bn'], $p['name_en'],
                    $p['category'], $p['price'], $p['rating'],
                    $this->toVectorLiteral($embeddings[$i]),
                ]);
            }
            $this->pdo->commit();
        } catch (\Throwable $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }

    public function search(
        string $query,
        ?string $category = null,
        ?float $maxPrice = null,
        ?float $minRating = null,
        int $k = 10
    ): array {
        [$qVec] = $this->embed([$query]);
        $vec = $this->toVectorLiteral($qVec);

        $where  = ['in_stock = true'];
        $params = [$vec];
        if ($category)  { $where[] = 'category = ?'; $params[] = $category; }
        if ($maxPrice)  { $where[] = 'price <= ?';   $params[] = $maxPrice; }
        if ($minRating) { $where[] = 'rating >= ?';  $params[] = $minRating; }
        $params[] = $k;

        $sql = sprintf(
            'SELECT sku, name_bn, price, rating,
                    1 - (embedding <=> ?::vector) AS similarity
             FROM products
             WHERE %s
             ORDER BY embedding <=> ?::vector
             LIMIT ?',
            implode(' AND ', $where)
        );

        // Note: pgvector position needs to be replicated for ORDER BY.
        // Simplification: use a subquery or repeat parameter.
        $stmt = $this->pdo->prepare(str_replace(
            'ORDER BY embedding <=> ?::vector',
            'ORDER BY embedding <=> \'' . $vec . '\'::vector',
            $sql
        ));
        $stmt->execute($params);
        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
}

// Usage
$pdo = new PDO('pgsql:host=localhost;dbname=daraz', 'user', 'pass', [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
]);
$pdo->exec("SET hnsw.ef_search = 100");

$oai = OpenAI::client(getenv('OPENAI_API_KEY'));
$store = new VectorStore($pdo, $oai);

$results = $store->search('নীল সুতি শাড়ি', 'saree', 1500.0, 4.0, 20);
foreach ($results as $r) {
    printf("%s (৳%.0f, ⭐%.1f) sim=%.3f\n",
        $r['name_bn'], $r['price'], $r['rating'], $r['similarity']);
}
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন

- Semantic search (Daraz "এই প্রোডাক্টের মতো")
- RAG (Pathao support FAQ retrieval)
- Recommendation ("যারা এটা কিনেছে তারা আরো কিনেছে" — collaborative + content)
- Duplicate detection (Daraz-এ একই পণ্য বিভিন্ন seller)
- Multi-modal (text + image search via CLIP)
- Anomaly detection (bKash fraud — atypical transaction patterns)
- Multilingual (Bangla query → English product match)

### ❌ এড়িয়ে চলুন

- Exact ID/SKU lookup → Postgres B-tree
- Range/aggregation heavy (`SUM(price) GROUP BY`) → analytical DB
- < 10K vectors → in-memory cosine in app
- Strong consistency required → vector DB eventual
- Frequently updated metadata-only filtering → traditional DB
- Compliance forbids embeddings of PII (GDPR/PDPA — Bangladesh DPA 2023)

---

## ⚠️ Common Pitfalls

```
1. ❌ Wrong distance metric
   - Trained with cosine, queried with L2 → garbage results
   - Always verify model card

2. ❌ Mixing embedding models
   - Index v1 of OpenAI, query v2 → silently bad
   - Re-embed everything when changing model

3. ❌ Ignoring normalization
   - Some models output non-normalized vectors
   - normalize() before indexing if using dot product

4. ❌ HNSW with frequent deletes
   - Deletes are tombstones — graph degrades
   - Periodic rebuild or use IVF

5. ❌ Filter selectivity ignored
   - 0.1% match rate + post-filter = 0 results
   - Use pre-filter or filter-aware ANN

6. ❌ Forgetting ef_search tuning
   - Default 40 → 70% recall
   - 100-200 → 95%+ recall (latency 2-3x)

7. ❌ One vector per document (long docs)
   - 10-page PDF in 1 vector → loses detail
   - Chunk → vector per chunk

8. ❌ No hybrid search for keyword-heavy domains
   - "iPhone 15 Pro Max 256GB" — exact tokens matter
   - Combine BM25 + vector via RRF

9. ❌ Embedding cost ignored
   - 10M docs × $0.02/1M tokens × 500 tokens = $100 one-time, ok
   - But per-query embedding: 1M queries × $0.00002 = $20/mo, scales

10. ❌ Cold start: empty vector DB at launch
    - Bootstrap with synthetic queries from product titles
```

---

## 🇧🇩 Bangladesh Real-world: Daraz Catalog 1M+ Products

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Daraz Vector Search                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   User Query: "ঢাকায় delivery cheap blue saree under 1500"  │
│                          │                                   │
│                          ▼                                   │
│   ┌──────────────────────────────────────────────┐          │
│   │  Query Understanding (LLM-based)             │          │
│   │  - Intent: search                            │          │
│   │  - Entities: {color: blue, max_price: 1500,  │          │
│   │              category: saree, area: dhaka}   │          │
│   │  - Translation: "blue cotton saree"          │          │
│   └──────────────────────────────────────────────┘          │
│                          │                                   │
│       ┌──────────────────┼──────────────────┐                │
│       ▼                                     ▼                │
│  ┌──────────┐                       ┌──────────────┐         │
│  │ Qdrant   │                       │ Elasticsearch│         │
│  │ HNSW     │                       │ BM25         │         │
│  │ embed    │                       │ name_bn,     │         │
│  │ payload: │                       │ name_en      │         │
│  │ category │                       └──────────────┘         │
│  │ price    │                              │                  │
│  │ seller   │                              │                  │
│  └──────────┘                              │                  │
│       │                                    │                  │
│       └──────────┬─────────────────────────┘                  │
│                  ▼                                            │
│         Reciprocal Rank Fusion                                │
│                  │                                            │
│                  ▼                                            │
│         Cross-encoder Rerank (top-50 → top-10)                │
│         Model: bge-reranker-v2-m3 (multilingual)              │
│                  │                                            │
│                  ▼                                            │
│         Personalization layer:                                │
│         - User's past clicks boost                            │
│         - Seller rating boost                                 │
│         - Promotion boost                                     │
│                  │                                            │
│                  ▼                                            │
│              Top 10 results                                   │
└──────────────────────────────────────────────────────────────┘
```

### Numbers

```
Catalog:             1.2M active SKUs
Embedding model:     bge-m3 (Bangla + English)
Embedding dim:       1024
Index:               Qdrant HNSW M=32, ef_construct=256
Memory:              ~10 GB (with int8 quantization)
Sharding:            4 shards by seller_id hash
Replication:         3 replicas
Latency target:      P95 < 80ms (vector + rerank + personalize)
QPS peak:            500 (Pohela Boishakh, Eid sale)
Reembed schedule:    weekly batch + real-time on new SKU
Cost:                3× c5.4xlarge AWS Singapore ≈ $1500/mo
```

### Key Lessons

1. **Bangla normalization** crucial: "শাড়ী" / "শাড়ি" / "শারি" — Unicode normalize NFC + custom rules
2. **Code-mixed queries** (~40% of Bangladeshi e-commerce): "blue শাড়ি 1000 taka" — multilingual model essential
3. **Synonyms via embedding** beat manual lists: জামদানি ≈ tangail ≈ traditional saree
4. **Filter pushdown** brought P95 from 250ms → 80ms (price, category, area)
5. **Reranking** improved CTR by 18% over plain ANN
6. **Cold-start sellers**: new sellers' products embedded from category templates initially

---

## 📋 Production Checklist

```
☐ Embedding model selected & benchmarked on Bangla+English
☐ Distance metric matches model card (cosine for OpenAI/BGE)
☐ ANN algorithm chosen by N, latency, memory budget
☐ HNSW M, ef_construction, ef_search tuned via Recall@10
☐ Filter pushdown enabled (payload indexes created)
☐ Hybrid search (BM25 + vector + RRF) for keyword-heavy domain
☐ Reranking layer (cross-encoder) for top results
☐ Embedding cost monitored (per query + reindex)
☐ Reindexing strategy on model upgrade (versioned indices)
☐ Backup & restore tested (snapshot + WAL)
☐ HA: replication + failover tested
☐ Sharding strategy chosen (random/tenant/semantic)
☐ Capacity plan: RAM = 1.5×N×d×4 + edges
☐ Monitoring: P50/P95/P99, recall@K, cache hit, cost/query
☐ Eviction/TTL for stale documents
☐ PII review: no embedded NID, phone number, account info
☐ Data residency: BD financial data on-prem if required
☐ Cold-start plan for empty-state launch
```

---

## 🎯 সারমর্ম

| বিষয়                  | সিদ্ধান্ত (Bangladesh context)                       |
| --------------------- | ----------------------------------------------------- |
| Default DB            | pgvector (যদি Postgres থাকে), নাহলে Qdrant            |
| Algorithm             | HNSW (95% cases), DiskANN (1B+)                       |
| Embedding             | bge-m3 (self) বা OpenAI 3-small (managed)             |
| Hosting               | AWS Singapore/Mumbai or local DC for compliance       |
| Hybrid                | BM25 + vector + RRF + cross-encoder rerank            |
| Filter strategy       | Filter-aware ANN (payload indexes)                    |
| Reembedding           | Versioned index, blue-green swap                      |

> Vector DB শুধু একটা component — পুরো RAG/search pipeline-এর success নির্ভর করে embedding model, chunking, filtering, এবং reranking এর উপর। পরের অধ্যায়ে [RAG Systems](./rag-systems.md) দেখুন।

---

## 🔗 আরও পড়ুন

- [RAG Systems](./rag-systems.md) — কীভাবে এই সব মিলিয়ে production pipeline বানাবেন
- [Embeddings](./embeddings.md) — embedding model গভীরে
- [Full-Text Search](../05-database/full-text-search.md) — BM25 fundamentals
- [LLM Agents & Tools](./llm-agents-tools.md) — agent থেকে vector DB call
