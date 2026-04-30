# 🤖 AI ইঞ্জিনিয়ারিং (AI Engineering — LLM, RAG, Vector DB, MCP)

> ২০২৩-এর পর সফটওয়্যার ইঞ্জিনিয়ারিং দুনিয়ার সবচেয়ে দ্রুত পরিবর্তিত শাখা।
> এই সেকশনে LLM-এর internals থেকে শুরু করে production-grade RAG, agent, MCP, evaluation — সবকিছু বাংলায়, PHP/JavaScript/Python (যেখানে দরকার) উদাহরণসহ।
> বাংলাদেশী context (Daraz, bKash, Pathao, Foodpanda, Chaldal) এবং বাংলা NLP-এর বাস্তব সমস্যা মাথায় রেখে লেখা।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|------|------|--------------|
| ১ | LLM Fundamentals | [llm-fundamentals.md](./llm-fundamentals.md) | Transformer, attention, tokenizer (BPE), pretraining → RLHF, context window, KV cache, GPT-4o/Claude/Gemini/Llama 3/Qwen তুলনা, inference parameters, vLLM/Ollama |
| ২ | Prompt Engineering | [prompt-engineering.md](./prompt-engineering.md) | Zero/few-shot, CoT, ReAct, Tree-of-Thought, structured output, function calling, prompt injection defense, Bangla prompting |
| ৩ | Embeddings | [embeddings.md](./embeddings.md) | Vector space, cosine/dot/euclidean, OpenAI/Cohere/BGE/multilingual-e5, Matryoshka, CLIP, chunking, pgvector |
| ৪ | LLM Ops | [llm-ops.md](./llm-ops.md) | Prompt versioning, semantic cache, fallback chains, Langfuse/Helicone, PII redaction, moderation, multi-tenant SaaS, cost capping |
| ৫ | Fine-tuning vs RAG | [fine-tuning-vs-rag.md](./fine-tuning-vs-rag.md) | কখন কোনটা; LoRA/QLoRA, instruction tuning, DPO, decision matrix |
| ৬ | Vector Databases | [vector-databases.md](./vector-databases.md) | pgvector, Pinecone, Weaviate, Qdrant, Milvus, HNSW, IVF, hybrid search |
| ৭ | RAG Systems | [rag-systems.md](./rag-systems.md) | Naive → Advanced → Modular RAG, query rewriting, reranking, GraphRAG, agentic RAG |
| ৮ | LLM Agents & Tools | [llm-agents-tools.md](./llm-agents-tools.md) | ReAct, planner/executor, multi-agent, LangGraph, AutoGen, CrewAI |
| ৯ | MCP Protocol | [mcp-protocol.md](./mcp-protocol.md) | Model Context Protocol — server/client, transport, tool/resource/prompt capability |
| ১০ | Evaluations & Safety | [evaluations-and-safety.md](./evaluations-and-safety.md) | RAGAS, LLM-as-judge, golden sets, red teaming, jailbreak, hallucination, bias |

> 🛠️ **নোট:** এই সেকশনের প্রথম ৫টি (Part A: Core) এই agent লিখেছে। বাকি ৫টি (Part B: Applied — fine-tuning, vector DB, RAG, agents, MCP, eval) আরেকটি agent লিখছে। দু'টি ভাগ একসাথে পড়লে production-grade AI system বানানোর পূর্ণাঙ্গ ছবি পাবেন।

---

## 🗺️ টপিক সম্পর্ক ডায়াগ্রাম

```
                    ┌─────────────────────────────────────────┐
                    │       AI ইঞ্জিনিয়ারিং স্ট্যাক           │
                    └────────────────────┬────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              ▼                          ▼                          ▼
       ┌─────────────┐            ┌─────────────┐            ┌─────────────┐
       │ FOUNDATION  │            │  RETRIEVAL  │            │   ACTIONS   │
       └──────┬──────┘            └──────┬──────┘            └──────┬──────┘
              │                          │                          │
   ┌──────────┼──────────┐         ┌─────┴─────┐               ┌────┴────┐
   ▼          ▼          ▼         ▼           ▼               ▼         ▼
 LLM      Prompt    Embeddings  Vector DB    RAG          Agents+    MCP
 Funda-   Engg      (vec gen)   (pgvector,   (naive→       Tools      (model ↔
 mentals  (CoT,                  Pinecone,    advanced→    (ReAct,     external
 (atten., ReAct,                 Qdrant,      modular,     planner,    services
 BPE,     JSON                   HNSW)        rerank,      Lang-       standard)
 KV cache)mode)                              GraphRAG)    Graph)
   │         │          │          │           │             │         │
   └─────────┴──────────┴──────────┼───────────┴─────────────┴─────────┘
                                   │
                    ┌──────────────┴───────────────┐
                    ▼                              ▼
             ┌──────────────┐              ┌──────────────┐
             │   LLM Ops    │              │ Eval & Safety│
             │ (versioning, │              │ (RAGAS, red  │
             │  caching,    │              │  teaming,    │
             │  observ.,    │              │  hallucin.,  │
             │  cost cap)   │              │  guardrails) │
             └──────┬───────┘              └──────┬───────┘
                    │                              │
                    └──────────────┬───────────────┘
                                   ▼
                       ┌──────────────────────┐
                       │ FINE-TUNING vs RAG   │
                       │   (knowledge gap     │
                       │   কীভাবে পূরণ করবো?)  │
                       └──────────────────────┘
```

---

## 🎯 শেখার পথ (Learning Path)

```
Step 1: LLM Fundamentals → মডেল ভিতরে কী হচ্ছে বুঝুন
         │
Step 2: Prompt Engineering → মডেলের সাথে কথা বলার শিল্প
         │
Step 3: Embeddings → semantic similarity ও vector space
         │
Step 4: Fine-tuning vs RAG → knowledge inject করার সিদ্ধান্ত
         │
Step 5: Vector Databases → embedding store ও ANN search
         │
Step 6: RAG Systems → প্রোডাকশন RAG pipeline
         │
Step 7: LLM Agents → tool use, planning, multi-step
         │
Step 8: MCP Protocol → tool integration standardization
         │
Step 9: LLM Ops → reliability, cost, observability
         │
Step 10: Evaluations & Safety → কোয়ালিটি, security, governance
```

---

## 🇧🇩 কেন বাংলাদেশী ইঞ্জিনিয়ারের জন্য এই সেকশন গুরুত্বপূর্ণ?

| সমস্যা | AI সমাধান | প্রাসঙ্গিক টপিক |
|--------|-----------|----------------|
| **Daraz** product search বাংলা+ইংরেজি mixed query handle করতে পারছে না | Multilingual embeddings + hybrid search (BM25 + vector) | Embeddings, Vector DB, RAG |
| **Pathao** support 24/7 বাংলায় চালাতে গিয়ে agent cost বেশি | RAG-based chatbot with knowledge base of FAQ + policy | RAG, Prompt Engg, LLM Ops |
| **bKash** fraud detection-এ rule-based system false positive বেশি | Embedding-based anomaly detection + LLM reasoning | Embeddings, Agents |
| **Foodpanda** menu-based Q&A ("ভেজিটেরিয়ান বার্গার আছে?") | RAG over menu items, Bangla NLP | RAG, Embeddings |
| **Chaldal** customer service ticket triage | LLM classification + summarization | LLM Fundamentals, Prompts |
| Government সরকারি ফর্ম পূরণে সহায়তা | Multi-step agent + form filling tools | Agents, MCP |
| Bangla code-mixing ("amar order ta cancel koren please") | Multilingual model evaluation + prompt tuning | Eval, Prompt Engg |
| GPU খরচ ম্যানেজ করা (BD startup-এ tight budget) | Quantization, caching, smaller models, fallback chains | LLM Ops, Fine-tuning |

---

## ⚠️ বাংলা NLP-এর বাস্তব চ্যালেঞ্জ

```
┌──────────────────────────────────────────────────────────────────┐
│ চ্যালেঞ্জ                  │ প্রভাব                              │
├──────────────────────────────────────────────────────────────────┤
│ Bangla script (ৎ, ঁ, যুক্তাক্ষর) │ Tokenizer ভেঙে ফেলে — বেশি token │
│ → কম script-aware tokenizer │   → বেশি cost, কম context           │
├──────────────────────────────────────────────────────────────────┤
│ Code-mixing (Banglish, Romanized)│ "amar order ta" — model এর      │
│ "ami bhalo achi"            │   training data-তে কম, accuracy কম   │
├──────────────────────────────────────────────────────────────────┤
│ Dialect variation           │ চট্টগ্রাম ↔ ঢাকা ↔ সিলেট আলাদা       │
│ (চাটগাঁইয়া, সিলেটি)         │   → embedding similarity drop         │
├──────────────────────────────────────────────────────────────────┤
│ Limited high-quality        │ Bangla Wikipedia ছোট, formal Bangla   │
│ training corpora            │   text কম → reasoning quality         │
├──────────────────────────────────────────────────────────────────┤
│ Right-to-left? — না, কিন্তু │ Some old fonts/encoding issues        │
│ Unicode normalization দরকার │   (Bijoy ↔ Unicode)                   │
└──────────────────────────────────────────────────────────────────┘
```

**প্র্যাকটিক্যাল গাইড:** Bangla content-এ best results-এর জন্য (২০২৬ সালে):
- **Closed:** GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro — তিনটাই ভালো; Claude সবচেয়ে natural Bangla লেখে।
- **Open:** Llama 3.1 70B, Qwen 2.5 72B, Aya 23 (Cohere multilingual) — Aya specifically multilingual।
- **Embedding:** `multilingual-e5-large`, `BGE-M3`, `Cohere embed-multilingual-v3` — Bangla support করে।

---

## 🏗️ Production AI Stack — Reference Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          USER (Web/Mobile)                          │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ HTTPS
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  API GATEWAY (Kong/Nginx) — auth, rate limit, PII redaction         │
│  → see 06-api-design/api-gateway.md                                 │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  AI ORCHESTRATION LAYER (Node.js/PHP/Python)                        │
│  - Prompt versioning (Langfuse)                                     │
│  - Semantic + exact cache (Redis)                                   │
│  - Fallback chain: GPT-4o → Claude → Llama (self-hosted)            │
│  - Streaming via SSE/WebSocket                                      │
└──────────────┬──────────────────────┬──────────────────────────────┘
               │                      │
               ▼                      ▼
       ┌───────────────┐      ┌───────────────┐
       │  RETRIEVAL    │      │   LLM CALL    │
       │  - pgvector   │      │  - OpenAI     │
       │  - BM25       │      │  - Anthropic  │
       │  - Reranker   │      │  - vLLM (own) │
       └───────┬───────┘      └───────┬───────┘
               │                      │
               └──────────┬───────────┘
                          ▼
              ┌────────────────────────┐
              │ OBSERVABILITY & EVAL   │
              │ - Langfuse traces      │
              │ - RAGAS scoring        │
              │ - Hallucination check  │
              │ - Cost tracking/$user  │
              └────────────────────────┘
```

---

## 🚀 কেন একজন সিনিয়র ইঞ্জিনিয়ারের জন্য এটা mandatory (২০২৬)?

1. **প্রতিটা SaaS product-এ AI feature** — chatbot, summarization, search, recommendation। AI না জানলে product roadmap discussion-এ contribute করা যাবে না।
2. **Cost engineering** — naive prompt vs optimized prompt-এ ১০x cost difference হতে পারে। Senior দায়িত্ব হলো budget control।
3. **Reliability** — LLM non-deterministic; retry, fallback, eval ছাড়া production-এ নামালে disaster।
4. **Security & compliance** — PII leak, prompt injection, data poisoning — নতুন attack surface। Senior কে guardrail design করতে হবে।
5. **Architecture decisions** — RAG vs fine-tune vs prompt-only? Vector DB কোনটা? Self-host vs managed? — এই decisions-এ company-এর millions of taka নির্ভর করে।
6. **Team enablement** — junior দের শেখানো, code review-এ AI-specific bug ধরা (token leak, infinite loop in agents, etc.)।
7. **MCP (২০২৪-২০২৫ standard)** — Anthropic-এর MCP fast track-এ industry standard হচ্ছে। Tool integration এখন protocol-driven।

---

## 🔗 অন্য সেকশনের সাথে সংযোগ

| এই সেকশন | সম্পর্কিত সেকশন | কেন |
|----------|----------------|------|
| RAG, embeddings | [05-database/full-text-search.md](../05-database/full-text-search.md) | BM25 + vector hybrid search |
| LLM Ops, fallback | [12-concurrency/](../12-concurrency/README.md) | Circuit breaker, retry, timeout |
| Streaming, gateway | [14-networking/](../14-networking/README.md) | SSE, WebSocket, HTTP/2 |
| API design | [06-api-design/api-gateway.md](../06-api-design/api-gateway.md) | LLM API gateway pattern |
| Observability | [16-observability/](../16-observability/README.md) | Trace LLM calls, distributed tracing |
| Frontend streaming | [19-frontend-engineering/](../19-frontend-engineering/README.md) | UI for streaming chat (Vercel AI SDK) |
| Performance | [17-performance/](../17-performance/README.md) | Latency, TTFT, TPS optimization |
| Security | [11-security/](../11-security/README.md) | Prompt injection, PII, jailbreak |

---

## 📚 প্রস্তাবিত পঠন ক্রম

**নতুন (LLM-এ একদম নতুন):**
1. `llm-fundamentals.md` → 2. `prompt-engineering.md` → 3. `embeddings.md` → 4. `vector-databases.md` → 5. `rag-systems.md`

**Mid-level (LLM ব্যবহার করছেন কিন্তু production-এ নামাননি):**
1. `llm-ops.md` → 2. `evaluations-and-safety.md` → 3. `rag-systems.md` (advanced) → 4. `fine-tuning-vs-rag.md`

**Senior (architecture decision নিতে হবে):**
1. `fine-tuning-vs-rag.md` → 2. `llm-ops.md` → 3. `mcp-protocol.md` → 4. `llm-agents-tools.md` → 5. `evaluations-and-safety.md`

---

> 💡 **টিপ:** AI field-এ knowledge ৬ মাসে obsolete হয়ে যায়। এই সেকশন **principles**-এ focus করেছে — model name বদলালেও patterns same থাকবে। নতুন paper/release follow করতে: [Hugging Face Daily Papers](https://huggingface.co/papers), [Latent Space podcast](https://www.latent.space/), [Sebastian Raschka's blog](https://sebastianraschka.com/)।

> ⚠️ **সতর্কতা:** Code example-এ API key hardcode করা আছে demonstration-এর জন্য। Production-এ env var, secret manager (Vault/AWS Secrets Manager) ব্যবহার করুন। `11-security/` দেখুন।
