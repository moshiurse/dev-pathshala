# 📚 RAG Systems (Retrieval-Augmented Generation)

> **RAG** = LLM-কে real-time external knowledge দিয়ে answer তৈরি করতে বলা।
> LLM যা জানে না (আপনার private data, latest info), সেটা retrieve করে context-এ দিন → হ্যালুসিনেশন কম, citation দেওয়া যায়, সস্তা।
> Pathao support agent driver guideline-এ Bangla-তে answer দেয়, bKash চ্যাটবট policy doc থেকে quote করে — সব RAG।

---

## 📖 সূচিপত্র

- [RAG কী এবং কেন](#-rag-কী-এবং-কেন)
- [Pipeline Overview](#-pipeline-overview)
- [Document Loaders](#-document-loaders)
- [Chunking Strategies](#-chunking-strategies)
- [Indexing Pipeline](#-indexing-pipeline)
- [Hybrid Search & RRF](#-hybrid-search--rrf)
- [Re-ranking](#-re-ranking)
- [Query Transformations](#-query-transformations)
- [Context Construction](#-context-construction--prompt-assembly)
- [Citation Generation](#-citation-generation)
- [Evaluation (RAGAS)](#-evaluation-ragas)
- [Caching Layer](#-caching-layer)
- [RAG vs Long-context vs Fine-tuning](#-rag-vs-long-context-vs-fine-tuning)
- [Advanced Patterns](#-advanced-patterns)
- [Multimodal RAG](#-multimodal-rag)
- [কোড: LangChain.js Full Pipeline](#-কোড-langchainjs-full-pipeline)
- [PHP Integration](#-php-integration)
- [BD Real-world: Pathao Driver Guidelines](#-bd-real-world-pathao-driver-guidelines)
- [Pitfalls](#-common-pitfalls)
- [Checklist](#-production-checklist)

---

## 🎯 RAG কী এবং কেন

```
LLM-only (no RAG):
─────────────────
User: "Pathao bike ride এ helmet না থাকলে কি হবে?"
GPT-4: "সাধারণত..." (generic, hallucinated, no source)
❌ Pathao policy জানে না
❌ Bangla nuance মিস
❌ Citation দিতে পারে না

RAG:
────
1. User query → embed → vector DB search Pathao guideline doc
2. Top-3 relevant chunks retrieve
3. Prompt: "এই context-এর ভিত্তিতে answer দাও:\n[chunks]\n\nQ: ..."
4. LLM grounded answer + page citation

✅ Updated knowledge (just reindex docs)
✅ Citation: "Driver Guideline v3.2, Section 4.1"
✅ Hallucination minimized
✅ Cheap (no fine-tuning)
✅ Multi-tenant friendly (different docs per tenant)
```

### কখন RAG vs অন্য কিছু

```
                                        ┌─────────────┐
              Knowledge dynamic? ───YES─→│    RAG      │
                       │                 └─────────────┘
                       │NO
                       ▼
              < 100K tokens?  ──────YES─→ Long-context (Gemini 2M)
                       │
                       │NO
                       ▼
              Style/format change? ──YES→ Fine-tuning
                       │
                       │NO
                       ▼
                  Prompt engineering
```

---

## 🏗️ Pipeline Overview

```
┌─────────────────────── INDEXING (offline) ───────────────────────┐
│                                                                  │
│  Documents (PDF, HTML, DOCX, MD, DB rows)                        │
│        │                                                         │
│        ▼                                                         │
│   Loader → raw text + metadata                                   │
│        │                                                         │
│        ▼                                                         │
│   Cleaner (remove headers/footers, normalize)                    │
│        │                                                         │
│        ▼                                                         │
│   Chunker (recursive / semantic / parent-doc)                    │
│        │                                                         │
│        ▼                                                         │
│   Embedder (bge-m3 / OpenAI 3-small)                             │
│        │                                                         │
│        ▼                                                         │
│   Vector DB (Qdrant/pgvector) + BM25 index (ES/Tantivy)          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────── QUERY (online) ──────────────────────────┐
│                                                                 │
│  User Query                                                     │
│        │                                                        │
│        ▼                                                        │
│   Query Transform (HyDE, multi-query, decomposition)            │
│        │                                                        │
│        ▼                                                        │
│   Retrieve: vector + BM25 → RRF fusion → top-50                 │
│        │                                                        │
│        ▼                                                        │
│   Rerank: cross-encoder → top-5                                 │
│        │                                                        │
│        ▼                                                        │
│   Context Construction (window mgmt, dedup)                     │
│        │                                                        │
│        ▼                                                        │
│   LLM (with citation prompt)                                    │
│        │                                                        │
│        ▼                                                        │
│   Post-process (citation extraction, safety)                    │
│        │                                                        │
│        ▼                                                        │
│   Response → user                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📂 Document Loaders

### Common Sources & Tools

| Source       | JS Tool                              | PHP Tool                         | Notes                              |
| ------------ | ------------------------------------ | -------------------------------- | ---------------------------------- |
| PDF (text)   | `pdf-parse`, `unpdf`                 | `smalot/pdfparser`               | Quick extraction                   |
| PDF (scan)   | `tesseract.js` + OCR                 | `tesseract` CLI via `exec`       | Bangla-tesseract-ben.traineddata   |
| DOCX         | `mammoth`                            | `phpoffice/phpword`              |                                    |
| HTML         | `cheerio` + `@mozilla/readability`   | `symfony/dom-crawler`            | Strip nav/footer                   |
| Markdown     | `unified` + remark                   | `league/commonmark`              | Preserve headings → hierarchy      |
| Confluence   | REST API                             | REST API                         | Auth needed                        |
| Notion       | `@notionhq/client`                   | REST API                         |                                    |
| Google Docs  | Drive API                            | Drive API                        |                                    |
| DB rows      | direct SQL                           | PDO                              | "row → text" template              |

### LangChain.js Loaders

```javascript
import { PDFLoader } from '@langchain/community/document_loaders/fs/pdf';
import { CheerioWebBaseLoader } from '@langchain/community/document_loaders/web/cheerio';
import { DocxLoader } from '@langchain/community/document_loaders/fs/docx';
import { DirectoryLoader } from 'langchain/document_loaders/fs/directory';

const loader = new DirectoryLoader('./pathao-docs', {
  '.pdf':  (p) => new PDFLoader(p, { splitPages: true }),
  '.docx': (p) => new DocxLoader(p),
});
const docs = await loader.load();
// docs[i] = { pageContent, metadata: { source, page } }
```

### Bangla PDF Gotcha

```
সমস্যা: অনেক বাংলা PDF font-embedding-এ scrambled glyph
Test: extracted text-এ "ঢ" বদলে "Aw" দেখা যাচ্ছে?
সমাধান:
1. pdf2text-এর বদলে OCR (tesseract --lang ben)
2. বা Apache Tika (Java) — best Bangla support
3. বা document AI (Google) — paid কিন্তু accurate
```

---

## ✂️ Chunking Strategies

> **নিয়ম:** chunk size = embedding model context window-এর 25-50% (e.g., 512 tokens for OpenAI 8k)

### 1) Fixed-size

```
chunkSize = 500 tokens, overlap = 50

"ড্রাইভার গাইডলাইন... helmet বাধ্যতামূলক... [500] ... জরিমানা ৫০০..."
                                          ^^^^^^^^^^ overlap
                                          "...জরিমানা ৫০০ টাকা... [500] ..."

✅ সহজ, predictable
❌ Sentence/paragraph cut করে
❌ Context lose
```

### 2) Recursive Character Splitter (LangChain default)

```
Separators (priority order): ["\n\n", "\n", "। ", ". ", " ", ""]

Try splitting by \n\n (paragraph) first.
If chunk still > maxSize, split by \n.
... down the chain.

✅ Respects structure
✅ Bangla-aware ("। " for sentence end)
```

### 3) Semantic Chunking

```
1. Split into sentences
2. Embed each sentence
3. Compute cosine distance between consecutive sentences
4. High distance → topic shift → chunk boundary

✅ Topic-coherent chunks
❌ Slow (embed every sentence)
❌ Cost
```

### 4) Parent-Document / Small-to-Big

```
Index small chunks (200 tokens) for precision
Store reference to parent chunk (2000 tokens) for context

Query:
  vector search on small chunks → top-5
  retrieve their parent chunks → pass to LLM

Result: precise retrieval + rich context
```

### 5) Sliding Window

```
window = 500, stride = 250 → 50% overlap

Use when: dialogues, where context spans
Don't use when: structured docs (header table)
```

### 6) Document-Structure-Aware

```
For Markdown/HTML:
  - Split by H1, H2, H3 hierarchy
  - Each section = chunk
  - Carry breadcrumb in metadata: "Pathao Guide > Chapter 4 > 4.1 Helmet"

For code:
  - Split by function/class boundary (tree-sitter)

For tables:
  - Each row → chunk OR full table → chunk if small
```

### Chunking Decision Tree

```
Doc type?
├── Conversational/Narrative   → Recursive 500/50
├── Technical doc with headers → Document-structure (preserve hierarchy)
├── Long policy/legal          → Parent-document (precise retrieve, big context)
├── Code                       → AST-based (tree-sitter)
├── Tables                     → Row-level OR HTML→Markdown
└── Mixed                      → Recursive + metadata enrichment
```

### Chunk Metadata (Critical!)

```javascript
{
  pageContent: "হেলমেট না পরলে...",
  metadata: {
    source: "pathao-driver-guide-v3.2.pdf",
    page: 12,
    chapter: "Chapter 4: Safety",
    section: "4.1 Helmet Policy",
    language: "bn",
    last_updated: "2024-08-15",
    doc_type: "policy",
    tenant_id: "pathao",
    chunk_id: "guide-4.1-c3",
    // for parent-doc retrieval:
    parent_id: "guide-4.1",
  }
}
```

---

## 🏭 Indexing Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                  Indexing Pipeline                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Source Watcher (S3, Notion webhook, DB CDC)                 │
│        │                                                     │
│        ▼                                                     │
│  Job Queue (BullMQ, Redis)                                   │
│        │                                                     │
│        ▼                                                     │
│  Worker:                                                     │
│   1. Fetch document                                          │
│   2. Hash content → skip if unchanged                        │
│   3. Extract text                                            │
│   4. Clean (remove boilerplate)                              │
│   5. Chunk                                                   │
│   6. Enrich metadata (LLM-extracted summary, entities)       │
│   7. Embed (batch 100 at a time)                             │
│   8. Upsert to vector DB (idempotent by chunk_id)            │
│   9. Index BM25 (Elasticsearch)                              │
│  10. Update version table                                    │
│                                                              │
│  Errors → DLQ + alert                                        │
│  Metrics → indexing rate, embedding cost, doc count          │
└──────────────────────────────────────────────────────────────┘
```

### Idempotency

```javascript
// Stable chunk_id = hash(source + section + chunk_index)
// Reindex same doc → same IDs → upsert overwrites
const chunkId = createHash('sha256')
  .update(`${source}:${section}:${chunkIndex}`)
  .digest('hex')
  .slice(0, 16);
```

### Versioned Index

```
On embedding model upgrade:
  1. Create products_v2 collection
  2. Reembed all → write to v2
  3. Test queries against both
  4. Atomic alias swap: products → v2
  5. Drop v1 after grace period

Never overwrite live index in-place!
```

---

## 🔀 Hybrid Search & RRF

### কেন hybrid?

```
"OTP code ভুল আসছে bKash"
                ↓
Vector only:    OTP, code, error → semantic match
                But may miss "OTP" exact token in unrelated docs

BM25 only:      Exact "OTP", "bKash" tokens
                But miss "verification code" synonym

Hybrid (RRF):   Best of both
```

### Reciprocal Rank Fusion (RRF)

```
RRF score(d) = Σ 1 / (k + rank_i(d))    (k=60 typical)

Example:
  Doc-A:  rank in vector=3,  rank in bm25=1   → 1/63 + 1/61 = 0.0323
  Doc-B:  rank in vector=1,  rank in bm25=10  → 1/61 + 1/70 = 0.0307
  Doc-C:  rank in vector=20, rank in bm25=2   → 1/80 + 1/62 = 0.0286

→ Sorted: A, B, C (A wins because both rankers ranked it high)

✅ No score normalization needed
✅ Robust to ranker scale differences
```

### Implementation (LangChain.js)

```javascript
import { EnsembleRetriever } from 'langchain/retrievers/ensemble';

const ensemble = new EnsembleRetriever({
  retrievers: [vectorRetriever, bm25Retriever],
  weights:    [0.5, 0.5],
});
const docs = await ensemble.invoke('OTP কোড আসছে না bKash');
```

---

## 🎯 Re-ranking

```
Why? ANN top-10 ≠ semantically best top-10
- Embedding compresses too much
- Cross-encoder reads (query, doc) pair → much higher precision

Pipeline:
  Retrieve top-50 (cheap, ANN+BM25)
       │
       ▼
  Cross-encoder rerank → top-5 (expensive, ~50 inferences/query)
       │
       ▼
  LLM context
```

### Models

| Model                                | Type       | Multilingual | Speed (CPU) | Notes                |
| ------------------------------------ | ---------- | ------------ | ----------- | -------------------- |
| **Cohere Rerank 3 (multilingual)**   | API        | ✅ 100+ lang | ~150ms/50   | Best for Bangla, paid|
| **Jina Reranker v2 (multilingual)**  | API/OSS    | ✅           | ~200ms      | Free tier, OSS       |
| **`BAAI/bge-reranker-v2-m3`**        | Self-host  | ✅✅ Bangla  | ~500ms/50   | Open source 👑       |
| `cross-encoder/ms-marco-MiniLM-L-12` | Self-host  | English-only | ~50ms       | Don't use for Bangla |

### Code (Cohere Rerank)

```javascript
import { CohereClient } from 'cohere-ai';
const cohere = new CohereClient({ token: process.env.COHERE_API_KEY });

const reranked = await cohere.rerank({
  model: 'rerank-multilingual-v3.0',
  query: 'হেলমেট না পরলে জরিমানা কত?',
  documents: top50.map(d => d.pageContent),
  topN: 5,
});
const top5 = reranked.results.map(r => top50[r.index]);
```

---

## 🔁 Query Transformations

### 1) HyDE (Hypothetical Document Embeddings)

```
Idea: short query embedding ≠ long doc embedding (asymmetric)
Solution: LLM-এ first hypothetical answer generate → embed that

Query: "helmet না পরলে?"
       ↓ LLM
HyDE: "Pathao policy অনুযায়ী helmet ছাড়া ride করলে driver-কে
       ৫০০ টাকা জরিমানা এবং তিনবার violation-এ account suspend
       করা হবে। Safety reasons-এ helmet বাধ্যতামূলক।"
       ↓ embed
       ↓ vector search → much better recall
```

### 2) Multi-Query

```
LLM generates N reformulations:
  "helmet jorimana"
  "Pathao helmet penalty fine"
  "হেলমেট ছাড়া পেনাল্টি কত"
  "rider helmet rule violation"

Embed all → search each → merge unique results

✅ Recall boost
❌ N× embedding + retrieval cost
```

### 3) Step-Back

```
Original: "শুক্রবার রাতে Pathao bike Banani থেকে Uttara surge price?"
Step-back: "Pathao bike surge pricing কীভাবে কাজ করে?"

→ retrieve general policy doc
→ then answer specific
```

### 4) Query Decomposition

```
"Pathao এবং Uber-এ helmet policy পার্থক্য কি?"
      ↓ LLM decompose
   - "Pathao helmet policy কী?"
   - "Uber Bangladesh helmet policy কী?"
      ↓ retrieve each independently
      ↓ combine in final prompt
```

### 5) Query Routing

```
LLM classifies → route to specialized retriever:
  Policy question → policy_index
  Order status   → orders DB
  Payment issue  → bKash policy_index + transaction DB
```

---

## 📦 Context Construction & Prompt Assembly

### Window Management

```
Total context budget: 8K tokens (GPT-4) or 32K, etc.

Allocation example (8K window):
  System prompt:        300 tokens
  Retrieved chunks:    4500 tokens (top-5 × ~900)
  Conversation hist:   1500 tokens
  Query:                100 tokens
  ────────────────────────────────
  Input:               6400 tokens
  Output budget:       1600 tokens
```

### Deduplication

```
Two retrieved chunks may be near-duplicates (same paragraph from
two pages, or version-1 and version-2 of same doc).
→ Cluster by embedding similarity > 0.95 → keep one
→ Diversify with MMR (Maximal Marginal Relevance):

MMR(d) = λ × sim(q, d) - (1-λ) × max sim(d, selected)
```

### Prompt Template

```
System: তুমি Pathao এর support assistant। শুধু নিচের context থেকে
        উত্তর দাও। যদি context-এ উত্তর না থাকে, "এই বিষয়ে আমার
        কাছে তথ্য নেই" বলো।

Context:
[1] (driver-guide-v3.2.pdf, page 12, Section 4.1)
হেলমেট না পরে ride করলে...

[2] (driver-guide-v3.2.pdf, page 13, Section 4.2)
জরিমানা ৫০০ টাকা প্রথমবার...

[3] (faq.html, "Safety FAQs")
তৃতীয়বার violation-এ ৭ দিন suspend...

Question: হেলমেট না পরলে কতবার পর account suspend হবে?

Answer in Bangla. Always cite sources as [1], [2], etc.
```

---

## 📑 Citation Generation

### Approach 1: Numbered References (most reliable)

Inject `[1]`, `[2]` markers in context, instruct model to use them.
Post-process: replace `[1]` with link/footnote.

### Approach 2: JSON Output Mode

```javascript
const schema = {
  answer: "string",
  citations: [
    { source: "...", page: 0, quote: "..." }
  ],
};
// Use OpenAI structured outputs or zod-ai
```

### Approach 3: Self-RAG (verify after)

```
1. LLM generates answer
2. Second LLM call: "Does this claim match this chunk? yes/no"
3. Drop unsupported claims, add [Need verification] tags
```

### Anti-Hallucination Prompt

```
- "শুধু নিচের context থেকে উত্তর দাও"
- "যা নেই তা guess কোরো না"
- "প্রতিটি claim-এর পাশে [chunk_id] উল্লেখ করো"
- "context insufficient হলে স্পষ্টভাবে বলো"
```

---

## 📊 Evaluation (RAGAS)

### Metrics

```
1. Faithfulness:
   "Generated answer-এর কতটুকু claim retrieved context-এ আছে?"
   answer ── decompose to claims ── verify each in context

2. Answer Relevance:
   "Answer কি query-এর সাথে relevant?"
   answer ── reverse engineer probable questions ── compare with original

3. Context Precision:
   "Top-K context-এ relevant chunks কোথায় (early ranks?)"
   ranking quality of retriever

4. Context Recall:
   "Ground truth answer-এর সব claim retrieved context-এ আছে কি?"

5. Context Relevance:
   "Retrieved chunks query-এর সাথে কতটা সম্পর্কিত?"
```

### RAGAS in Code (Python via JS sub-process or direct)

```javascript
// Approach: log to dataset, run RAGAS in CI
import fs from 'fs';

const evalRow = {
  question: query,
  answer: response,
  contexts: top5Chunks.map(c => c.pageContent),
  ground_truth: expected, // from labeled dataset
};
fs.appendFileSync('eval.jsonl', JSON.stringify(evalRow) + '\n');

// In CI: python -m ragas evaluate eval.jsonl
```

### LangSmith / Langfuse (recommended)

```javascript
import { Langfuse } from 'langfuse';
const lf = new Langfuse();

const trace = lf.trace({ name: 'rag', input: query });
const retrieval = trace.span({ name: 'retrieve', input: query });
// ... do retrieval ...
retrieval.end({ output: chunks });

const generation = trace.generation({
  name: 'llm', model: 'gpt-4o-mini', input: prompt,
});
// ... LLM call ...
generation.end({ output: response });

trace.update({ output: response });
```

---

## 💾 Caching Layer

```
┌──────────────────────────────────────────────────────┐
│  Three cache levels                                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  L1: Exact-match (Redis):                            │
│      key = hash(query + tenant)                      │
│      TTL = 5 min for hot queries                     │
│      Hit rate: 5-15% in support bots                 │
│                                                      │
│  L2: Semantic cache:                                 │
│      Embed query → ANN against past queries (>0.95)  │
│      Return cached answer if very similar            │
│      Hit rate: 20-40%                                │
│      Risk: subtle paraphrase, different intent       │
│                                                      │
│  L3: Embedding cache:                                │
│      key = hash(text)                                │
│      Avoid reembedding same chunk on reindex         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Code (Redis exact + semantic)

```javascript
import { createClient } from 'redis';
const redis = createClient(); await redis.connect();

async function cachedAnswer(query, tenantId) {
  // L1
  const k1 = `rag:exact:${tenantId}:${hash(query)}`;
  const exact = await redis.get(k1);
  if (exact) return JSON.parse(exact);

  // L2 semantic
  const qVec = await embed(query);
  const semHits = await qdrant.search('query_cache', {
    vector: qVec, limit: 1, score_threshold: 0.97,
    filter: { must: [{ key: 'tenant', match: { value: tenantId } }] },
  });
  if (semHits.length) return semHits[0].payload.answer;

  // Miss → full RAG
  const ans = await runRAG(query);
  await redis.setEx(k1, 300, JSON.stringify(ans));
  await qdrant.upsert('query_cache', { points: [{
    id: nanoid(), vector: qVec,
    payload: { tenant: tenantId, query, answer: ans, ts: Date.now() }
  }] });
  return ans;
}
```

---

## ⚖️ RAG vs Long-context vs Fine-tuning

| Approach          | Knowledge update | Cost (1M queries) | Latency | Citation | Best for                   |
| ----------------- | ---------------- | ----------------- | ------- | -------- | -------------------------- |
| **RAG**           | Re-index (mins)  | $20-200           | 200-800ms | ✅       | Dynamic, large corpus       |
| **Long-context**  | Re-prompt        | $5000+ (huge tok) | 2-10s   | ⚠️       | Small-medium static doc     |
| **Fine-tuning**   | Re-train ($$$$)  | $5-50 per query   | 100-500ms | ❌      | Style, format, domain language |
| **Prompt-only**   | Edit prompt      | $5-20             | 100ms   | ❌       | Simple, generic tasks       |

> Detailed: see [fine-tuning-vs-rag.md](./fine-tuning-vs-rag.md).

---

## 🚀 Advanced Patterns

### 1) Agentic RAG

```
Agent loop:
  LLM decides:
    - retrieve more?
    - rewrite query?
    - call tool (DB, calc, API)?
    - finalize answer?

Use case: complex multi-hop question
"Pathao-তে গত মাসে আমার কতবার ride হয়েছে এবং fine থাকলে কত?"
  → Tool 1: DB query rides count
  → Tool 2: Retrieve fine policy
  → Combine
```

See [llm-agents-tools.md](./llm-agents-tools.md).

### 2) GraphRAG (Microsoft)

```
Idea: build knowledge graph from corpus, traverse for context

Pipeline:
  1. LLM extracts (entity, relation, entity) triples per chunk
  2. Build Neo4j graph
  3. Community detection (Leiden) → community summaries
  4. Query: traverse relevant communities + retrieve their chunks

When better than RAG:
  - Multi-hop questions
  - "Compare A and B" — needs cross-doc reasoning
  - Network-heavy data (org hierarchy, supply chain)

Cost: 10-50x indexing (LLM-extracted triples)
```

### 3) CRAG (Corrective RAG)

```
1. Retrieve as usual
2. Lightweight evaluator: are chunks relevant? (T5-based)
3. If LOW confidence → web search fallback
4. If MID → mix retrieved + web
5. If HIGH → use as-is

✅ Robust to corpus gaps
❌ Latency
```

### 4) Self-RAG

```
LLM trained with reflection tokens:
  [Retrieve], [Relevant], [Supported], [Useful]

Generates: "জরিমানা ৫০০ [Retrieve] [Relevant: doc1] [Supported]"

Each token decides: retrieve more? Is it relevant? Is claim supported?
```

### 5) RAG-Fusion

Multi-Query + RRF: generate 4 queries, retrieve each, RRF merge.

### 6) Adaptive Retrieval

```
LLM first decides: do I need retrieval at all?
"2+2=?" → no retrieval
"bKash send money fee?" → retrieve
```

---

## 🖼️ Multimodal RAG

### Architectures

```
1) CLIP-based (text+image shared embedding):
   - Embed product photos + Bangla text in same space
   - Daraz: query "নীল শাড়ি" → returns matching image SKUs

2) Caption-then-embed:
   - Image → BLIP/LLaVA → text caption (Bangla)
   - Embed caption like normal RAG

3) Multimodal LLM at inference:
   - Retrieve image chunks
   - Pass to GPT-4o / Gemini Vision in context

Example query:
   User uploads photo of saree they like
   → CLIP embed → ANN search Daraz catalog
   → Top-10 similar products + price + reviews
```

### Tools

- **CLIP** (OpenAI): English-centric, weak on Bangla queries
- **Multilingual CLIP** (`sentence-transformers/clip-ViT-B-32-multilingual-v1`): supports Bangla queries
- **SigLIP** (Google): better than CLIP at retrieval
- **Jina CLIP v1**: open source, 89 languages including Bangla

---

## 💻 কোড: LangChain.js Full Pipeline

### Setup

```bash
npm i @langchain/openai @langchain/community @langchain/core \
      @langchain/textsplitters pgvector pg \
      cohere-ai langfuse zod
```

### Indexing Worker

```javascript
// indexer.js
import { OpenAIEmbeddings } from '@langchain/openai';
import { RecursiveCharacterTextSplitter } from '@langchain/textsplitters';
import { PDFLoader } from '@langchain/community/document_loaders/fs/pdf';
import { PGVectorStore } from '@langchain/community/vectorstores/pgvector';
import { createHash } from 'crypto';

const embeddings = new OpenAIEmbeddings({ model: 'text-embedding-3-small' });

const store = await PGVectorStore.initialize(embeddings, {
  postgresConnectionOptions: { connectionString: process.env.DATABASE_URL },
  tableName: 'pathao_kb',
  columns: { idColumnName: 'id', vectorColumnName: 'embedding',
             contentColumnName: 'content', metadataColumnName: 'metadata' },
  distanceStrategy: 'cosine',
});

async function indexPDF(path, docMetadata) {
  const loader = new PDFLoader(path, { splitPages: true });
  const pages = await loader.load();

  const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 800,
    chunkOverlap: 120,
    separators: ['\n\n', '\n', '। ', '. ', ' ', ''],
  });

  const chunks = await splitter.splitDocuments(pages);

  const enriched = chunks.map((c, i) => {
    const id = createHash('sha256')
      .update(`${docMetadata.source}:${c.metadata.loc?.pageNumber}:${i}`)
      .digest('hex').slice(0, 16);
    return {
      pageContent: c.pageContent,
      metadata: {
        ...docMetadata,
        ...c.metadata,
        chunk_id: id,
        language: 'bn',
      },
    };
  });

  await store.addDocuments(enriched, { ids: enriched.map(e => e.metadata.chunk_id) });
  console.log(`Indexed ${enriched.length} chunks from ${path}`);
}

await indexPDF('./docs/pathao-driver-guide-v3.2.pdf', {
  source: 'pathao-driver-guide-v3.2',
  doc_type: 'policy',
  tenant_id: 'pathao',
});
```

### Query Pipeline

```javascript
// rag.js
import { ChatOpenAI } from '@langchain/openai';
import { OpenAIEmbeddings } from '@langchain/openai';
import { PGVectorStore } from '@langchain/community/vectorstores/pgvector';
import { ChatPromptTemplate } from '@langchain/core/prompts';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { CohereClient } from 'cohere-ai';
import { Langfuse } from 'langfuse';

const llm = new ChatOpenAI({ model: 'gpt-4o-mini', temperature: 0.2 });
const embeddings = new OpenAIEmbeddings({ model: 'text-embedding-3-small' });
const cohere = new CohereClient({ token: process.env.COHERE_API_KEY });
const lf = new Langfuse();

const store = await PGVectorStore.initialize(embeddings, /* ... */);

// 1) Multi-query expansion
async function expandQuery(query) {
  const prompt = `Generate 3 alternative phrasings (English/Bangla mix ok)
of the user's question. Output JSON array of strings only.

Question: ${query}`;
  const out = await llm.invoke(prompt);
  try { return [query, ...JSON.parse(out.content)]; }
  catch { return [query]; }
}

// 2) Retrieve top-50 (vector + filter)
async function retrieve(queries, tenantId, k = 20) {
  const allHits = [];
  for (const q of queries) {
    const hits = await store.similaritySearchWithScore(q, k, {
      tenant_id: tenantId,
    });
    allHits.push(...hits.map(([doc, score]) => ({ doc, score })));
  }
  // Dedup by chunk_id, keep best score
  const map = new Map();
  for (const h of allHits) {
    const id = h.doc.metadata.chunk_id;
    if (!map.has(id) || map.get(id).score < h.score) map.set(id, h);
  }
  return [...map.values()].sort((a, b) => b.score - a.score).slice(0, 50);
}

// 3) Cohere multilingual rerank → top-5
async function rerank(query, candidates, topN = 5) {
  const res = await cohere.rerank({
    model: 'rerank-multilingual-v3.0',
    query,
    documents: candidates.map(c => c.doc.pageContent),
    topN,
  });
  return res.results.map(r => candidates[r.index].doc);
}

// 4) Build prompt with citations
function buildPrompt(query, docs) {
  const ctx = docs.map((d, i) =>
    `[${i + 1}] (source: ${d.metadata.source}, page: ${d.metadata.loc?.pageNumber ?? '?'})\n${d.pageContent}`
  ).join('\n\n');

  return ChatPromptTemplate.fromMessages([
    ['system', `তুমি Pathao এর support assistant। শুধু context থেকে
উত্তর দাও। প্রতিটি দাবির পরে [N] citation marker দাও। যদি context-এ উত্তর
না থাকে, "এই বিষয়ে আমার কাছে যথেষ্ট তথ্য নেই" বলো।

Context:
${ctx}`],
    ['human', '{q}'],
  ]);
}

// 5) Main RAG fn
export async function answer(query, tenantId = 'pathao') {
  const trace = lf.trace({ name: 'rag', input: query, userId: tenantId });

  const queries = await expandQuery(query);
  trace.span({ name: 'expand', output: queries }).end();

  const top50 = await retrieve(queries, tenantId);
  trace.span({ name: 'retrieve', output: { count: top50.length } }).end();

  const top5 = await rerank(query, top50);
  trace.span({ name: 'rerank', output: top5.map(d => d.metadata.chunk_id) }).end();

  const prompt = buildPrompt(query, top5);
  const chain = prompt.pipe(llm).pipe(new StringOutputParser());

  const gen = trace.generation({ name: 'gen', model: 'gpt-4o-mini' });
  const text = await chain.invoke({ q: query });
  gen.end({ output: text });

  // Extract citations [N] → real sources
  const citations = top5.map((d, i) => ({
    n: i + 1, source: d.metadata.source, page: d.metadata.loc?.pageNumber,
  }));

  trace.update({ output: { text, citations } });
  await lf.flushAsync();

  return { answer: text, citations };
}
```

### Usage

```javascript
const out = await answer('হেলমেট না পরলে কতবার violation-এর পর suspend?');
console.log(out.answer);
// "Pathao policy অনুযায়ী হেলমেট না পরলে প্রথমবার ৫০০ টাকা [1],
//  দ্বিতীয়বার ১০০০ টাকা [1], তৃতীয়বার ৭ দিন account suspend হবে [2]।"
console.log(out.citations);
// [{n:1, source:'pathao-driver-guide-v3.2', page:12}, ...]
```

---

## 🐘 PHP Integration

PHP-তে full RAG pipeline না বানিয়ে, Node.js RAG service-কে micro-service হিসেবে call করাই production pattern।

```php
<?php
// app/Services/RagClient.php (Laravel-style)

namespace App\Services;

use GuzzleHttp\Client;

final class RagClient
{
    public function __construct(
        private Client $http,
        private string $baseUrl,
        private string $apiKey,
    ) {}

    public function ask(string $query, string $tenantId, ?string $userId = null): array
    {
        $res = $this->http->post("{$this->baseUrl}/v1/answer", [
            'json' => [
                'query'     => $query,
                'tenant_id' => $tenantId,
                'user_id'   => $userId,
            ],
            'headers' => [
                'Authorization' => "Bearer {$this->apiKey}",
                'X-Request-Id'  => bin2hex(random_bytes(8)),
            ],
            'timeout' => 30,
        ]);
        return json_decode((string) $res->getBody(), true);
    }
}

// Controller
namespace App\Http\Controllers;

use App\Services\RagClient;
use Illuminate\Http\Request;

class SupportController extends \Illuminate\Routing\Controller
{
    public function ask(Request $req, RagClient $rag)
    {
        $req->validate([
            'q' => 'required|string|min:3|max:1000',
        ]);

        $userId = $req->user()->id;
        $cacheKey = 'rag:'.md5($req->input('q')).":{$userId}";
        $cached = cache()->get($cacheKey);
        if ($cached) return response()->json($cached);

        $out = $rag->ask($req->input('q'), 'pathao', (string) $userId);
        cache()->put($cacheKey, $out, now()->addMinutes(5));

        return response()->json($out);
    }
}
```

PHP-only RAG (small scale):

```php
// Pure PHP RAG with pgvector + OpenAI for SMB use case
$qVec = $oai->embeddings()->create([
    'model' => 'text-embedding-3-small',
    'input' => $query,
])->embeddings[0]->embedding;

$stmt = $pdo->prepare(
    "SELECT content, metadata, 1 - (embedding <=> :v::vector) AS sim
     FROM pathao_kb
     WHERE metadata->>'tenant_id' = :t
     ORDER BY embedding <=> :v::vector
     LIMIT 5"
);
$stmt->execute([
    ':v' => '['.implode(',', $qVec).']',
    ':t' => 'pathao',
]);
$chunks = $stmt->fetchAll(\PDO::FETCH_ASSOC);

$context = '';
foreach ($chunks as $i => $c) {
    $meta = json_decode($c['metadata'], true);
    $context .= "[".($i+1)."] (source: {$meta['source']})\n{$c['content']}\n\n";
}

$resp = $oai->chat()->create([
    'model' => 'gpt-4o-mini',
    'messages' => [
        ['role' => 'system', 'content' => "শুধু context থেকে উত্তর দাও।\n\n{$context}"],
        ['role' => 'user',   'content' => $query],
    ],
]);
```

---

## 🇧🇩 BD Real-world: Pathao Driver Guidelines

### Stack

```
Docs:
  - Driver Guideline PDF (Bangla, 84 pages, v3.x)
  - FAQ HTML (200+ items)
  - Policy update emails (CMS)
  - Past support tickets (anonymized)

Indexing:
  - Daily cron: pull Confluence → reindex deltas
  - Embedding: bge-m3 (self-hosted on g4dn.xlarge)
  - Vector DB: Qdrant cluster (3 nodes, AWS Singapore)
  - BM25: Elasticsearch (multilingual analyzer with Bangla normalizer)

Query:
  - Frontend: support panel (Bangla input)
  - Multi-query expansion (3 variants)
  - Hybrid: vector + BM25 + RRF
  - Rerank: Cohere multilingual
  - LLM: GPT-4o-mini (Bangla output, cheap)
  - Citations: page+section

Eval:
  - 500 labeled Q&A pairs
  - Daily RAGAS run in CI
  - Faithfulness target: > 0.85
  - Context recall:    > 0.80

Cost:
  - Embedding (one-time): ~$5 for whole corpus
  - Per query: ~$0.002 (embedding $0.0001 + LLM $0.002)
  - 50K queries/day → $100/day
```

### Lessons Learned

1. **Bangla normalization** essential: NFC, ZWJ/ZWNJ cleanup, digit conversion
2. **Mixed Bangla-English** queries (~50%): multilingual model required
3. **Citation rendered as clickable PDF page** dramatically increased trust
4. **5% query cap on LLM** — rest answered by FAQ exact match (cheaper)
5. **Fallback to human agent** when faithfulness < 0.5 or no context found
6. **PII redaction** before logging (driver phone, license number)

---

## ⚠️ Common Pitfalls

```
1. ❌ Chunk size too large
   - 2000-token chunks → ANN finds vague match → wasted context

2. ❌ Chunk size too small
   - 100-token chunks → no context, snippet feels broken

3. ❌ No metadata filtering
   - Multi-tenant data mixed → leak between Pathao and Foodpanda 😱

4. ❌ Query embed ≠ doc embed model
   - Indexed with v1, queried with v2 → garbage

5. ❌ No reranking
   - Top-K from ANN often ranks 4 above 1 → reranker fixes

6. ❌ No deduplication
   - Same paragraph in multiple PDFs → 5 redundant chunks in context

7. ❌ Missing citation
   - Users distrust ungrounded answers → always cite

8. ❌ Stale cache on policy update
   - Cache-bust on indexing job

9. ❌ Embedding all docs at once → blow API rate limit
   - Batch + backoff

10. ❌ Treating RAG as static
    - Continuous evaluation, golden dataset, regression tests
```

---

## 📋 Production Checklist

```
Indexing:
  ☐ Idempotent chunk IDs
  ☐ Versioned embedding model
  ☐ Metadata: source, page, language, tenant, doc_type
  ☐ Reindex pipeline (cron / webhook)
  ☐ Cost tracking (embedding tokens)

Retrieval:
  ☐ Hybrid search (vector + BM25 + RRF)
  ☐ Reranker (Cohere/Jina/bge-reranker)
  ☐ Multi-query for recall-critical
  ☐ Filter pushdown (tenant, language, date)
  ☐ Top-K tuning via evaluation

Generation:
  ☐ System prompt forces grounding + citation
  ☐ Citation format consistent (numbered)
  ☐ Refusal when context insufficient
  ☐ Output schema (JSON) if downstream parsing
  ☐ Streaming for long answers

Eval & Ops:
  ☐ Golden dataset (100+ Q&A)
  ☐ RAGAS in CI
  ☐ Langfuse/LangSmith tracing
  ☐ Latency P50/P95 SLO
  ☐ Cost per query dashboard
  ☐ Hallucination alarm (faithfulness < threshold)
  ☐ Human handoff path

Safety:
  ☐ PII redaction in logs
  ☐ Tenant isolation tested
  ☐ Prompt injection defense (system prompt protection)
  ☐ Rate limit per user
  ☐ Content safety filter on output
```

---

## 🎯 সারমর্ম

```
RAG Success Formula:
  Good chunking + Right embedding + Hybrid search + Rerank
  + Tight prompt + Citations + Continuous eval = Trust

BD Reality:
  - Bangla + English code-mix dominant
  - Multilingual embedding non-negotiable
  - Self-host (Qdrant + bge-m3) cheap & compliant
  - Latency budget tight (Singapore AWS or local DC)
  - Citations build trust faster than fluency
```

---

## 🔗 আরও পড়ুন

- [Vector Databases](./vector-databases.md)
- [Embeddings](./embeddings.md)
- [LLM Agents & Tools](./llm-agents-tools.md) — Agentic RAG
- [Fine-tuning vs RAG](./fine-tuning-vs-rag.md)
- [LLM Ops](./llm-ops.md) — production observability
