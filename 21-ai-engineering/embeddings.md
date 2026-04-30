# 🔢 Embeddings — Text কে Vector-এ রূপান্তর

## 📖 সংজ্ঞা

**Embedding** হলো একটা piece of data (text/image/audio) কে একটা **fixed-dimensional dense vector**-এ map করা, যেখানে similar meaning-এর data কাছাকাছি vector পায়। এটা semantic search, RAG, recommendation, classification — সবকিছুর foundation।

```
"Daraz-এ ভালো শার্ট"     ──► [0.23, -0.11, 0.87, ..., 0.04]   (1536 dims)
"Daraz online shirt"     ──► [0.21, -0.13, 0.85, ..., 0.06]   ← কাছাকাছি
"bKash পেমেন্ট"           ──► [-0.45, 0.78, -0.12, ..., 0.31]  ← দূরে
"আকাশ কেমন আজ?"          ──► [0.91, 0.05, -0.34, ..., 0.42]   ← দূরে
```

**মূল ধারণা:** vectors-এর মধ্যে **distance/similarity** → meaning-এর closeness reflect করে। এই মাত্রটাই magic।

---

## 🌌 Vector Space — মানসিক মডেল

কল্পনা করুন একটা ৩-D স্থান (আসলে ৭৬৮ বা ১৫৩৬ বা ৩০৭২ মাত্রা — visualize অসম্ভব, কিন্তু math same)।

```
                  ↑ semantic axis 2 (formality?)
                  │
    "ফরমাল চিঠি"  ●
                  │
                  │       "হাই কেমন আছ"
   ───────────────●───────────────●──────► axis 1 (greeting?)
                  │
                  │  ●  "বিদায়"
                  │
                  ↓
```

**যা আমরা চাই (semantic property):**
- "রাজা" - "পুরুষ" + "নারী" ≈ "রাণী"  (analogical reasoning, word2vec era)
- "বার্গার" + "মাংস" close in vector space
- "Daraz" এবং "অনলাইন কেনাকাটা" close
- Bangla "ধন্যবাদ" এবং English "thank you" multilingual model-এ close

---

## 📏 Similarity Metrics

```
Three primary distance/similarity measures:

1. Cosine similarity   = (A · B) / (||A|| · ||B||)
                       Range: [-1, 1] (1 = identical direction)
                       Magnitude ignore — direction matter
                       USE: text embeddings (most common)

2. Dot product         = A · B
                       Range: (-∞, ∞)
                       Magnitude matter
                       USE: when vectors normalized OR magnitude meaningful

3. Euclidean (L2)      = √Σ(Aᵢ - Bᵢ)²
                       Range: [0, ∞) (0 = identical)
                       Sensitive to magnitude
                       USE: clustering, image embeddings sometimes
```

### কোনটা কখন?

```
┌─────────────────┬────────────────────────────────────────┐
│ Embedding model │ Recommended metric                     │
├─────────────────┼────────────────────────────────────────┤
│ OpenAI ada/3    │ cosine (already L2-normalized → also OK│
│                 │   dot product, equivalent)             │
│ Cohere v3       │ cosine                                 │
│ BGE             │ cosine                                 │
│ E5/multilingual │ cosine                                 │
│ Sentence-BERT   │ cosine                                 │
│ CLIP (image)    │ cosine                                 │
│ Voyage          │ cosine OR dot                          │
└─────────────────┴────────────────────────────────────────┘
```

> **Pro tip:** OpenAI embedding-গুলো already L2-normalized (||v|| = 1), তাই dot product এবং cosine similarity equivalent। `pgvector`-এ `<#>` (negative dot) অপারেটর index-friendly এবং fast।

### Code: Cosine in Python/Node/PHP

```python
# Python
import numpy as np
def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

```javascript
// JavaScript
function cosine(a, b) {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    na  += a[i] * a[i];
    nb  += b[i] * b[i];
  }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
}
```

```php
<?php
function cosine(array $a, array $b): float {
    $dot = 0; $na = 0; $nb = 0;
    foreach ($a as $i => $av) {
        $bv = $b[$i];
        $dot += $av * $bv;
        $na  += $av * $av;
        $nb  += $bv * $bv;
    }
    return $dot / (sqrt($na) * sqrt($nb));
}
```

---

## 🏪 Embedding Model Families (২০২৬ landscape)

### Closed / Hosted

| Model | Provider | Dim | Max input | $/1M tokens | Bangla | Notes |
|-------|----------|-----|-----------|-------------|--------|-------|
| **text-embedding-3-small** | OpenAI | 1536 (Matryoshka 256-1536) | 8191 | $0.02 | ⭐⭐⭐ | Default cheap |
| **text-embedding-3-large** | OpenAI | 3072 (Matryoshka 256-3072) | 8191 | $0.13 | ⭐⭐⭐⭐ | High quality |
| **embed-english-v3** | Cohere | 1024 | 512 | $0.10 | English only | |
| **embed-multilingual-v3** | Cohere | 1024 | 512 | $0.10 | ⭐⭐⭐⭐ | 100+ lang including Bangla |
| **voyage-3-large** | Voyage AI | 1024 | 32K | $0.18 | ⭐⭐⭐ | Best for code/legal/finance |
| **voyage-3-lite** | Voyage AI | 512 | 32K | $0.02 | ⭐⭐ | Cheap |
| **jina-embeddings-v3** | Jina AI | 1024 | 8K | $0.02 | ⭐⭐⭐ | Multilingual, MRL |
| **gemini-embedding** | Google | 768/1536/3072 | 2K | varies | ⭐⭐⭐ | New, MRL support |

### Open-source / Self-host

| Model | Dim | Bangla | Strengths |
|-------|-----|--------|-----------|
| **BGE-M3** (BAAI) | 1024 | ⭐⭐⭐⭐ | Multilingual + dense+sparse+colbert একসাথে |
| **multilingual-e5-large** | 1024 | ⭐⭐⭐⭐ | Strong multilingual, license permissive |
| **multilingual-e5-base** | 768 | ⭐⭐⭐⭐ | Smaller, fast |
| **all-MiniLM-L6-v2** (Sentence-Transformers) | 384 | ⭐ | English-centric, super fast |
| **nomic-embed-text-v1.5** | 768 (MRL 64-768) | ⭐⭐ | Fully open data + weights |
| **GTE-large** (Alibaba) | 1024 | ⭐⭐⭐ | Strong English |
| **Aya 23 embedding heads** | varies | ⭐⭐⭐⭐ | Cohere Aya — multilingual focus |

### বাংলার জন্য recommendation

```
Production Bangla embedding:
1. Cohere embed-multilingual-v3   ← best balance (hosted)
2. multilingual-e5-large          ← best self-hosted
3. BGE-M3                          ← if hybrid (dense+sparse) দরকার
4. text-embedding-3-large          ← good fallback (large dim)
5. jina-embeddings-v3              ← cheap + multilingual
```

---

## 📐 Dimensionality

```
Higher dim → more capacity → potentially better quality
            → বেশি storage + memory + slower search
```

**Trade-off table:**

```
Dim     Storage/M vec (fp32)   Storage (fp16)   ANN search speed
─────────────────────────────────────────────────────────────────
384     1.5 GB                  768 MB           Fast
768     3.0 GB                  1.5 GB           Medium
1024    4.0 GB                  2.0 GB           Medium
1536    6.0 GB                  3.0 GB           Slower
3072    12 GB                   6.0 GB           Slowest
```

> **Real cost example:** Daraz-এ ১০M product embedding @ 3072 dim fp32 = 120 GB just for vectors। Index overhead + replicas = 300 GB+।

### Matryoshka Embeddings (MRL) — গেম চেঞ্জার

২০২২-এ Aditya Kusupati-এর paper। **Single model** train করা হয় এমনভাবে যেন vector-এর প্রথম 256, 512, 768, 1024 dims **alone-ই useful** হয়।

```
Full vector (1536 dims):  [v₁, v₂, ..., v₁₅₃₆]
                          └─ first 256 ─┘ also good!
                          └────── first 768 ────── ┘ better!
                          └────────── full 1536 ──────────┘ best
```

**ব্যবহার:**
- Coarse-to-fine search: first stage use 256-dim (fast, candidate set), then re-rank with 1536-dim (precise)
- Storage tier: hot (1536), warm (768), cold (256) — নির্ভর করে query frequency-র উপর
- Edge devices: mobile-এ 256-dim use করুন

```javascript
// OpenAI text-embedding-3-small with custom dim
const r = await openai.embeddings.create({
  model: "text-embedding-3-small",
  input: text,
  dimensions: 512,   // truncated, retrains MRL property
});
```

---

## ✂️ Chunking — Embedding-এর হিরো (বা ভিলেন)

বড় document একসাথে embed করা ভুল। কেন?

1. **Context dilution** — ১০ পাতার doc এক vector-এ → specific query match করবে না।
2. **Context window limit** — Cohere v3: 512 token; OpenAI: 8191 token।
3. **Retrieval granularity** — chunk-level retrieve করলে exact relevant para আনা যায়।

### Chunking Strategies

```
┌────────────────────┬────────────────────────────────────────┐
│ Strategy           │ When to use                            │
├────────────────────┼────────────────────────────────────────┤
│ Fixed token        │ Quick, uniform doc                     │
│   (e.g., 512 tok)  │ Default LangChain RecursiveText splitter│
├────────────────────┼────────────────────────────────────────┤
│ Sentence-based     │ Conversational, FAQ                    │
│ (use NLTK/spaCy)   │ Bangla — সাবধান sentence boundary      │
├────────────────────┼────────────────────────────────────────┤
│ Recursive char     │ General purpose; LangChain default     │
│ (try \n\n, \n,     │                                        │
│  ., space)         │                                        │
├────────────────────┼────────────────────────────────────────┤
│ Semantic chunking  │ Long-form, blog, paper                 │
│ (embed sentences,  │ Higher quality, expensive              │
│  group similar)    │                                        │
├────────────────────┼────────────────────────────────────────┤
│ Document-aware     │ Markdown (split by heading), HTML, code│
│                    │ (split by function/class)              │
├────────────────────┼────────────────────────────────────────┤
│ Late chunking      │ NEW (2024) — embed full doc once,      │
│ (Jina paper)       │ then split embeddings; preserves       │
│                    │ cross-chunk context                    │
├────────────────────┼────────────────────────────────────────┤
│ Parent-child       │ Embed small chunk for retrieval,       │
│ (small-to-big)     │ return larger parent for context       │
└────────────────────┴────────────────────────────────────────┘
```

### Recommended defaults (২০২৬)

```
Chunk size:  300-800 tokens    (sweet spot for most RAG)
Overlap:     10-20% of size    (preserve cross-chunk meaning)
Separator priority:  ["\n\n", "\n", "।", ".", " "]   ← Bangla "।"
```

### Chunking code — LangChain.js

```javascript
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 600,
  chunkOverlap: 100,
  separators: ["\n\n", "\n", "।", "।\n", ".", "?", "!", " ", ""],  // Bangla danda
});

const text = await fs.readFile("daraz-faq.txt", "utf-8");
const chunks = await splitter.splitText(text);
console.log(`Total chunks: ${chunks.length}`);
```

### Bangla-aware sentence splitting

বাংলা sentence boundary "।" (danda) — most English-trained splitter ভুল করে।

```javascript
function bengaliSentenceSplit(text) {
  // Bangla danda + English period/?/!
  return text.split(/(?<=[।.!?])\s+/).filter(s => s.trim().length > 0);
}

const sentences = bengaliSentenceSplit(
  "ঢাকা বাংলাদেশের রাজধানী। জনসংখ্যা প্রায় ২ কোটি। শহরটি ব্যস্ত।"
);
// ["ঢাকা বাংলাদেশের রাজধানী।", "জনসংখ্যা প্রায় ২ কোটি।", "শহরটি ব্যস্ত।"]
```

### Semantic chunking (advanced)

```python
# Python — LlamaIndex SemanticSplitter
from llama_index.core.node_parser import SemanticSplitterNodeParser
from llama_index.embeddings.openai import OpenAIEmbedding

splitter = SemanticSplitterNodeParser(
    buffer_size=1,
    breakpoint_percentile_threshold=95,
    embed_model=OpenAIEmbedding(),
)

nodes = splitter.get_nodes_from_documents(docs)
# adjacent sentences-এর embedding distance বড় হলে split
```

---

## 🖼️ Image Embeddings (CLIP)

OpenAI CLIP — text এবং image কে **same vector space**-এ এম্বেড করে। তাই: text query → image search; image query → similar image; cross-modal।

```
"Daraz-এ লাল শার্ট"  ─embed─►  vector  ─┐
                                          │ cosine
[image of red shirt] ─embed─►  vector  ─┘
                                          ▼
                                       similar!
```

**Models:**
- `openai/clip-vit-base-patch32` — 512 dim
- `openai/clip-vit-large-patch14` — 768 dim
- `laion/CLIP-ViT-bigG-14` — 1280 dim (LAION-2B trained)
- `jina-clip-v2` — multilingual + image (Bangla support!)

**BD use case:** Daraz-এ user "লাল ফুলহাতা টিশার্ট" লিখলে product image catalog থেকে CLIP দিয়ে retrieve।

```python
# Python with sentence-transformers
from sentence_transformers import SentenceTransformer
from PIL import Image

model = SentenceTransformer("clip-ViT-B-32")

text_emb = model.encode("লাল ফুলহাতা টিশার্ট")
img_emb  = model.encode(Image.open("product.jpg"))

similarity = cosine(text_emb, img_emb)
```

---

## 🎙️ Audio Embeddings

- **Whisper** (OpenAI) — primarily transcription, but encoder-output use করা যায়।
- **Wav2Vec2** (Meta) — self-supervised audio embedding।
- **CLAP** (Microsoft) — text-audio joint embedding (CLIP-like)।

**BD use case:** voice query support (call center) → transcribe + embed → match with FAQ embedding।

---

## 💸 Cost & Latency Optimization

### Cost calculation

```
Daraz product catalog: 5M products
Each product description: ~150 tokens

Total tokens to embed = 5M × 150 = 750M tokens

Cost (one-time):
- OpenAI text-embedding-3-small: 750 × $0.02 = $15
- OpenAI text-embedding-3-large: 750 × $0.13 = $97.50
- Cohere multilingual v3:        750 × $0.10 = $75
- Self-host BGE-M3 on 1× A10:    GPU $0.5/hr × 12 hr = $6
```

### Optimization tactics

```
1. Cache embeddings
   - Hash(text) → vector in Redis/disk
   - Re-embed only when text changes

2. Batch embedding
   - OpenAI: up to 2048 input strings per request
   - 10x throughput vs single-item

3. Truncate input
   - Most retrieval works with first 200-400 tokens
   - Long-doc tail rarely retrieved

4. Use smaller dim (Matryoshka)
   - text-embedding-3-large @ 512 dim = 1/6 storage,
     ~95% retrieval quality

5. Quantize stored vectors
   - fp32 → fp16: half storage, ~99% quality
   - fp32 → int8: quarter storage, ~95% quality
   - Binary embeddings (1-bit per dim): 1/32 storage,
     90% quality, 32x faster search

6. Async + parallel
   - Don't embed sequentially; use Promise.all in JS

7. Self-host for high volume
   - >10M docs/month → self-host BGE-M3 ROI positive
```

### Caching pattern

```javascript
// Embedding cache wrapper
import crypto from "node:crypto";
import { createClient } from "redis";

const redis = createClient();
await redis.connect();

async function cachedEmbed(text, model = "text-embedding-3-small") {
  const key = `emb:${model}:${crypto.createHash("sha256").update(text).digest("hex")}`;

  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const r = await openai.embeddings.create({ model, input: text });
  const vec = r.data[0].embedding;

  // 30-day TTL; embeddings rarely change
  await redis.set(key, JSON.stringify(vec), { EX: 60 * 60 * 24 * 30 });
  return vec;
}
```

### Batch embedding

```javascript
async function batchEmbed(texts, batchSize = 256) {
  const results = [];
  for (let i = 0; i < texts.length; i += batchSize) {
    const batch = texts.slice(i, i + batchSize);
    const r = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: batch,                     // array, not string
    });
    results.push(...r.data.map(d => d.embedding));
    console.log(`Embedded ${i + batch.length}/${texts.length}`);
  }
  return results;
}
```

---

## 🧪 Code: End-to-End Embedding Pipeline

### JavaScript — OpenAI + LangChain.js + pgvector

```javascript
// embed-and-store.js
import OpenAI from "openai";
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";
import pg from "pg";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const db = new pg.Pool({ connectionString: process.env.DATABASE_URL });

// 1. Schema (run once)
await db.query(`
  CREATE EXTENSION IF NOT EXISTS vector;

  CREATE TABLE IF NOT EXISTS daraz_products (
    id          SERIAL PRIMARY KEY,
    product_id  TEXT UNIQUE NOT NULL,
    title_bn    TEXT,
    title_en    TEXT,
    description TEXT,
    embedding   vector(1536)
  );

  CREATE INDEX IF NOT EXISTS idx_daraz_emb
    ON daraz_products
    USING hnsw (embedding vector_cosine_ops);
`);

// 2. Chunk + embed
async function ingestProduct(p) {
  const text = `${p.title_bn} ${p.title_en} ${p.description}`.slice(0, 4000);

  const r = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
    dimensions: 1536,
  });

  const vec = `[${r.data[0].embedding.join(",")}]`;

  await db.query(
    `INSERT INTO daraz_products (product_id, title_bn, title_en, description, embedding)
     VALUES ($1, $2, $3, $4, $5::vector)
     ON CONFLICT (product_id) DO UPDATE SET embedding = EXCLUDED.embedding`,
    [p.product_id, p.title_bn, p.title_en, p.description, vec]
  );
}

// 3. Search
async function searchProducts(query, k = 5) {
  const qr = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
    dimensions: 1536,
  });
  const qvec = `[${qr.data[0].embedding.join(",")}]`;

  const { rows } = await db.query(
    `SELECT product_id, title_bn, title_en,
            1 - (embedding <=> $1::vector) AS similarity
     FROM daraz_products
     ORDER BY embedding <=> $1::vector
     LIMIT $2`,
    [qvec, k]
  );

  return rows;
}

// Demo
await ingestProduct({
  product_id: "DZ-001",
  title_bn: "লাল ফুলহাতা টি-শার্ট",
  title_en: "Red Full-Sleeve T-Shirt",
  description: "Cotton, slim fit, sizes M/L/XL, made in BD",
});
await ingestProduct({
  product_id: "DZ-002",
  title_bn: "নীল হাফ-হাতা শার্ট",
  title_en: "Blue Half-Sleeve Shirt",
  description: "Polyester blend, formal, sizes M-XXL",
});

const results = await searchProducts("কটনের লাল গেঞ্জি");
console.log(results);
// → DZ-001 highest similarity
```

### LangChain.js — higher-level abstraction

```javascript
import { OpenAIEmbeddings } from "@langchain/openai";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";
import { Document } from "langchain/document";

const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
  dimensions: 1536,
});

const store = await PGVectorStore.initialize(embeddings, {
  postgresConnectionOptions: { connectionString: process.env.DATABASE_URL },
  tableName: "daraz_products_lc",
  collectionTableName: "daraz_collections",
});

await store.addDocuments([
  new Document({ pageContent: "লাল ফুলহাতা টি-শার্ট", metadata: { id: "DZ-001" } }),
  new Document({ pageContent: "নীল হাফ-হাতা শার্ট",  metadata: { id: "DZ-002" } }),
]);

const hits = await store.similaritySearch("কটনের লাল গেঞ্জি", 3);
console.log(hits);
```

### LlamaIndex.ts equivalent

```typescript
import { Document, VectorStoreIndex, OpenAIEmbedding } from "llamaindex";

const docs = [
  new Document({ text: "লাল ফুলহাতা টি-শার্ট" }),
  new Document({ text: "নীল হাফ-হাতা শার্ট" }),
];

const index = await VectorStoreIndex.fromDocuments(docs, {
  embedModel: new OpenAIEmbedding({ model: "text-embedding-3-small" }),
});

const queryEngine = index.asQueryEngine();
const response = await queryEngine.query({ query: "কটনের লাল গেঞ্জি" });
```

---

## 🐘 PHP Example — Foodpanda Menu Embedding

```php
<?php
// foodpanda-menu-embed.php
require 'vendor/autoload.php';

$openai = OpenAI::client(getenv('OPENAI_API_KEY'));

// 1. Postgres connection
$pdo = new PDO('pgsql:host=localhost;dbname=foodpanda',
               getenv('PG_USER'), getenv('PG_PASS'));

// 2. Schema (run once)
$pdo->exec("CREATE EXTENSION IF NOT EXISTS vector");
$pdo->exec("
    CREATE TABLE IF NOT EXISTS menu_items (
        id SERIAL PRIMARY KEY,
        restaurant_id INT NOT NULL,
        item_name TEXT NOT NULL,
        description TEXT,
        cuisine TEXT,
        is_vegetarian BOOLEAN,
        embedding vector(1536)
    );
    CREATE INDEX IF NOT EXISTS idx_menu_emb
        ON menu_items USING hnsw (embedding vector_cosine_ops);
");

function embedAndStore(PDO $pdo, $openai, array $item): void {
    $text = "{$item['item_name']}. {$item['description']}. Cuisine: {$item['cuisine']}.";

    $r = $openai->embeddings()->create([
        'model' => 'text-embedding-3-small',
        'input' => $text,
        'dimensions' => 1536,
    ]);
    $vec = '[' . implode(',', $r->embeddings[0]->embedding) . ']';

    $stmt = $pdo->prepare("
        INSERT INTO menu_items (restaurant_id, item_name, description, cuisine, is_vegetarian, embedding)
        VALUES (?, ?, ?, ?, ?, ?::vector)
    ");
    $stmt->execute([
        $item['restaurant_id'], $item['item_name'], $item['description'],
        $item['cuisine'], $item['is_vegetarian'], $vec,
    ]);
}

function search(PDO $pdo, $openai, string $query, int $k = 5): array {
    $r = $openai->embeddings()->create([
        'model' => 'text-embedding-3-small',
        'input' => $query,
        'dimensions' => 1536,
    ]);
    $vec = '[' . implode(',', $r->embeddings[0]->embedding) . ']';

    $stmt = $pdo->prepare("
        SELECT item_name, description, is_vegetarian,
               1 - (embedding <=> ?::vector) AS similarity
        FROM menu_items
        ORDER BY embedding <=> ?::vector
        LIMIT ?
    ");
    $stmt->execute([$vec, $vec, $k]);
    return $stmt->fetchAll(PDO::FETCH_ASSOC);
}

// Ingest sample menu
$items = [
    ['restaurant_id'=>1, 'item_name'=>'চিকেন বিরিয়ানি',
     'description'=>'খাঁটি কাচ্চি স্টাইলে রান্না করা চিকেন বিরিয়ানি',
     'cuisine'=>'Bangladeshi', 'is_vegetarian'=>false],
    ['restaurant_id'=>1, 'item_name'=>'ভেজিটেবল বার্গার',
     'description'=>'Fresh veg patty with cheese, no meat',
     'cuisine'=>'Continental', 'is_vegetarian'=>true],
    ['restaurant_id'=>2, 'item_name'=>'মাংস ছাড়া পিৎজা',
     'description'=>'Margherita with extra olives, fully vegetarian',
     'cuisine'=>'Italian', 'is_vegetarian'=>true],
];
foreach ($items as $i) embedAndStore($pdo, $openai, $i);

// User query: "ভেজিটেরিয়ান বার্গার আছে?"
$results = search($pdo, $openai, "ভেজিটেরিয়ান বার্গার আছে?");
echo json_encode($results, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
```

---

## 🇧🇩 BD Real-World: Daraz Mixed Bangla+English Search

**সমস্যা:** Daraz-এ একই product-এর title কখনো বাংলা, কখনো English, কখনো mixed:
- "Men's Cotton Shirt"
- "পুরুষদের কটন শার্ট"
- "Apnar size er shirt — formal"

**Naive approach (BM25 only):** miss করে cross-language match।

**Embedding approach:** multilingual model দিয়ে embed → একই vector space → cross-language match।

```javascript
// daraz-mixed-search.js
import { CohereClient } from "cohere-ai";

const cohere = new CohereClient({ token: process.env.COHERE_KEY });

async function multilingualEmbed(texts) {
  const r = await cohere.embed({
    texts,
    model: "embed-multilingual-v3.0",
    inputType: "search_document",  // for indexing
  });
  return r.embeddings;
}

async function multilingualQuery(query) {
  const r = await cohere.embed({
    texts: [query],
    model: "embed-multilingual-v3.0",
    inputType: "search_query",     // for query (different prompt!)
  });
  return r.embeddings[0];
}

// Test cross-language similarity
const docs = [
  "Men's Cotton Shirt — Premium quality",
  "পুরুষদের কটন শার্ট — উন্নত মানের",
  "Women's silk saree",
  "মহিলাদের সিল্ক শাড়ি",
];

const docEmbs  = await multilingualEmbed(docs);
const queryEmb = await multilingualQuery("পুরুষের জামা");

docs.forEach((d, i) => {
  console.log(d, "→ sim:", cosine(queryEmb, docEmbs[i]).toFixed(3));
});
// "Men's Cotton Shirt" → 0.71  ← cross-language match!
// "পুরুষদের কটন শার্ট" → 0.85
// "Women's silk saree" → 0.32
// "মহিলাদের সিল্ক শাড়ি" → 0.41
```

> **Cohere bonus:** `inputType` parameter আলাদা করেছে document vs query — asymmetric embedding, retrieval quality boost।

---

## 🚫 Common Pitfalls

1. **Different model on index vs query** — vectors incompatible space-এ; nonsense results।
2. **Forget normalization** — model normalize করে না কিন্তু আপনি cosine assume করছেন।
3. **Truncation silent** — input > model max → silently truncate; document tail lost।
4. **Embedding-এর age** — model upgrade হলে পুরো index re-embed করতে হবে; budget plan।
5. **PII embed করে cloud-এ পাঠানো** — bKash transaction text embed করার আগে redact।
6. **Bangla text-এ wrong encoding** — Bijoy → Unicode normalize আগে। `unorm.nfc()`।
7. **One-shot embed huge file** — chunk, batch — না হলে token limit error।
8. **Cosine similarity cutoff hardcoded as 0.7** — model-specific; calibrate per model।
9. **Symmetric embedding-এ asymmetric task** — query (short) ও document (long) আলাদা style; voyage/cohere asymmetric models use করুন।
10. **Embedding-কে relevance-এর single source ভাবা** — BM25 + reranker hybrid almost সবসময় better (covered in `rag-systems.md`)।

---

## 🧰 Senior Engineer Checklist

- [ ] Model + version pinned (e.g., `text-embedding-3-small@v1`)
- [ ] Re-embed migration plan documented (model upgrade strategy)
- [ ] Embedding cost tracked per tenant
- [ ] Embedding cache hit rate monitored
- [ ] Bangla normalization (NFC) আগে embed
- [ ] Chunk size + overlap A/B tested with eval set
- [ ] PII filter pre-embed
- [ ] Asymmetric embedding (query vs doc) considered if model supports
- [ ] Vector index type chosen consciously (HNSW vs IVF — see `vector-databases.md`)
- [ ] Backup strategy for embedding store

---

## 📚 আরো পড়ুন

- "Sentence-BERT" — Reimers & Gurevych, 2019
- "Matryoshka Representation Learning" — Kusupati et al, 2022
- "BGE-M3: Multi-Linguality, Multi-Functionality, Multi-Granularity" — BAAI, 2024
- "Late Chunking" — Jina AI, 2024
- "Multilingual E5" — Microsoft, 2024
- MTEB Leaderboard — huggingface.co/spaces/mteb/leaderboard

→ পরবর্তী: [LLM Ops](./llm-ops.md) — production-এ LLM চালাতে যা যা লাগে।
