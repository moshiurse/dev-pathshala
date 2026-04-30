# 🛠️ LLM Ops — Production-এ LLM চালানোর শিল্প

## 📖 সংজ্ঞা

**LLM Ops (LLMOps)** হলো MLOps-এর LLM-specific extension — prompt versioning, evaluation, cost tracking, observability, caching, fallback, governance, safety filtering — যা production-এ LLM সিস্টেম reliable, cost-efficient, এবং auditable রাখে।

```
┌──────────────────────────────────────────────────────────────────┐
│                      LLM OPS LIFECYCLE                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Develop ──► Test ──► Deploy ──► Monitor ──► Improve ──► Develop  │
│     │         │        │           │           │                  │
│     ▼         ▼        ▼           ▼           ▼                  │
│  Prompt    Eval    Canary     Trace,        Prompt                │
│  registry  suite   rollout    cost,         A/B,                  │
│            (RAGAS) prompt     latency,      finetune              │
│                    version    drift                               │
└──────────────────────────────────────────────────────────────────┘
```

**MLOps-এর সাথে পার্থক্য:**

| MLOps | LLMOps |
|-------|--------|
| Train custom model | Mostly use foundation model |
| Feature pipeline focus | Prompt + RAG pipeline focus |
| Model artifact versioning | Prompt + system message versioning |
| Test set numerical | Eval often LLM-as-judge, qualitative |
| GPU training infra | Inference infra (KV cache, vLLM, fallback chains) |

---

## 🎯 কেন এটা ক্রিটিক্যাল?

```
"আমি GPT-4 API call করেছি, কাজ করছে — done!"
                       │
                       ▼
       ┌───────────────────────────────┐
       │ Reality (production):         │
       │ - Cost spike (1000x in a day) │
       │ - Random hallucination        │
       │ - Vendor outage = total down  │
       │ - PII leaked in logs          │
       │ - Prompt change broke 30% of  │
       │   queries silently            │
       │ - User complaint, no traces   │
       │ - Bangla query 5x more cost   │
       └───────────────────────────────┘
```

LLM Ops না থাকলে — startup pivot to "AI-powered" → ৬ মাসে cash burn → shutdown। বাংলাদেশী startup-এ এই pattern ভয়াবহ common।

---

## 📦 Core Components Overview

```
                     ┌──────────────────────────┐
                     │       USER REQUEST        │
                     └────────────┬──────────────┘
                                  ▼
        ┌────────────────────────────────────────────────────┐
        │               INPUT GUARDRAILS                      │
        │   - PII redaction                                   │
        │   - Prompt injection detection                      │
        │   - Moderation API                                  │
        │   - Rate limit (per user/tenant)                    │
        └────────────────────────┬───────────────────────────┘
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │                CACHE LAYER                          │
        │   - Exact match (Redis)                             │
        │   - Semantic cache (vector similarity)              │
        └────────────────────────┬───────────────────────────┘
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │            PROMPT REGISTRY / VERSIONING             │
        │   - Langfuse / PromptLayer / git                    │
        │   - Active version per environment                  │
        └────────────────────────┬───────────────────────────┘
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │              MODEL ROUTER / FALLBACK                │
        │   GPT-4o ──► Claude ──► Llama 3 (self-host) ──► cached
        │   - Retry with backoff                              │
        │   - Circuit breaker                                 │
        │   - Timeout                                         │
        └────────────────────────┬───────────────────────────┘
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │              OUTPUT GUARDRAILS                      │
        │   - Schema validation                               │
        │   - PII scrub on output                             │
        │   - Hallucination check (groundedness)              │
        │   - Toxicity / brand-safety                         │
        └────────────────────────┬───────────────────────────┘
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │               OBSERVABILITY                          │
        │   - Langfuse / Helicone / LangSmith                 │
        │   - Trace, cost, latency, eval scores               │
        │   - Audit log (compliance)                          │
        └────────────────────────────────────────────────────┘
```

---

## 1️⃣ Prompt Versioning & Registry

### Problem

```
v1 prompt → 90% accuracy → deployed
↓
"একটু improve করি" → v2 prompt → silent commit
↓
3 দিন পর: support ticket spike → কেন? কেউ জানে না
↓
Rollback করতে চাই — কোনটা v1?
```

### Solution: Prompt Registry

**Tool options:**

| Tool | Strength | Open source? |
|------|----------|--------------|
| **Langfuse** | Trace + prompt registry + eval, all-in-one | Yes (self-host) |
| **Helicone** | API gateway + log + cost; minimal setup | Yes |
| **PromptLayer** | Mature prompt registry, lineage | No (SaaS) |
| **LangSmith** | LangChain native, deep tracing | No (SaaS) |
| **Promptfoo** | Eval-focused, CI/CD integration | Yes |
| **Git + custom** | Full control, simple | Yes |

### Langfuse — open-source, self-host friendly

```javascript
// install: pnpm add langfuse
import { Langfuse } from "langfuse";

const lf = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
  baseUrl: "https://langfuse.daraz-internal.com",
});

// Fetch active version of a prompt by label
const prompt = await lf.getPrompt("daraz-support-system", undefined, {
  label: "production",   // or "staging", "v3"
});

// Compile with variables
const compiled = prompt.compile({
  user_name: "রাহিম",
  language: "bn",
});

console.log(compiled);
// "You are Daraz support agent... User name: রাহিম, language: bn"
```

### Versioning workflow

```
1. Author edits prompt in Langfuse UI / commits to git
2. Tag as "v4-staging"
3. Run eval suite (next section) — gate on metrics
4. Promote: label "production" → "v4"
5. Old "v3" still labeled "fallback"
6. If issues → relabel "v3" as "production" (instant rollback)
7. Audit log: who promoted, when, eval results
```

### Git-only minimal approach

```
prompts/
  daraz-support/
    v1.md
    v2.md
    v3.md       ← current
    metadata.yaml  (active: v3, eval: scores.json)
  bkash-bot/
    ...
```

```javascript
import fs from "node:fs/promises";
import yaml from "js-yaml";

async function loadPrompt(name) {
  const meta = yaml.load(await fs.readFile(`prompts/${name}/metadata.yaml`, "utf-8"));
  return await fs.readFile(`prompts/${name}/${meta.active}.md`, "utf-8");
}
```

---

## 2️⃣ Caching — Cost Saver #1

### Exact-match cache (Redis)

```javascript
import crypto from "node:crypto";
import { createClient } from "redis";

const redis = createClient();
await redis.connect();

function cacheKey(model, messages, params) {
  const data = JSON.stringify({ model, messages, params });
  return `llm:${crypto.createHash("sha256").update(data).digest("hex")}`;
}

async function callWithCache(model, messages, params = {}) {
  const key = cacheKey(model, messages, params);
  const cached = await redis.get(key);
  if (cached) {
    metrics.increment("llm.cache.hit");
    return JSON.parse(cached);
  }

  const r = await openai.chat.completions.create({ model, messages, ...params });
  await redis.set(key, JSON.stringify(r), { EX: 3600 * 24 });
  metrics.increment("llm.cache.miss");
  return r;
}
```

**বাস্তব hit rate:**
- Customer support FAQ: 30-50%
- Generic Q&A: 5-15%
- Personalized chat: < 5%

### Semantic cache (vector similarity)

Different wording, same meaning → cache hit।

```
"How do I cancel my order?"        ─┐
"আমার অর্ডার ক্যান্সেল কীভাবে করব?"   ─┼─► same answer cached
"order cancel korbo kemne?"         ─┘
```

```javascript
// Pseudo — use redisearch/pgvector/qdrant
async function semanticCache(query, threshold = 0.93) {
  const qVec = await embed(query);
  const hits = await vectorStore.search(qVec, k = 1);
  if (hits.length && hits[0].similarity >= threshold) {
    metrics.increment("llm.semantic_cache.hit");
    return hits[0].cached_response;
  }
  return null;
}

async function answer(query) {
  let cached = await semanticCache(query);
  if (cached) return cached;

  const response = await callLLM(query);

  await vectorStore.upsert({
    embedding: await embed(query),
    cached_response: response,
    ttl: 3600 * 24 * 7,
  });
  return response;
}
```

**সাবধান:**
- Threshold বেশি কম → wrong answer serve করবে।
- Personalized query (order ID আছে) cache করা যাবে না — privacy + correctness issue।
- Cache key-এ tenant_id, language, version include করুন।

### GPTCache (open-source library)

Pre-built semantic cache library, multi-backend support।

```python
# Python
from gptcache import cache
from gptcache.adapter import openai
from gptcache.embedding import OpenAI

cache.init(embedding_func=OpenAI().to_embeddings)
cache.set_openai_key()

# Drop-in replacement
response = openai.ChatCompletion.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "..."}],
)
# Auto-cached based on semantic similarity
```

### Provider-level prompt caching (২০২৪+)

- **Anthropic prompt caching** — same prefix → 90% discount, 5-min TTL
- **Gemini context caching** — explicit cache, hours-long TTL
- **OpenAI prompt caching** — auto, 50% discount on cached input

```javascript
// Anthropic — mark large system prompt as cacheable
const message = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  system: [
    {
      type: "text",
      text: HUGE_SYSTEM_PROMPT,  // 5K+ tokens
      cache_control: { type: "ephemeral" },  // ← cache!
    },
  ],
  messages: [...],
});
```

---

## 3️⃣ Rate Limit Handling, Retry, Timeout

### Exponential backoff

```javascript
async function callLLMWithRetry(fn, opts = {}) {
  const maxAttempts = opts.maxAttempts ?? 5;
  const baseDelay   = opts.baseDelay   ?? 500;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      const retryable =
        err.status === 429 ||                  // rate limit
        err.status === 500 ||                  // server error
        err.status === 503 ||                  // overloaded
        err.code === "ECONNRESET" ||
        err.code === "ETIMEDOUT";

      if (!retryable || attempt === maxAttempts) throw err;

      // Honor Retry-After header if present
      const retryAfter = err.headers?.["retry-after"];
      const delay = retryAfter
        ? Number(retryAfter) * 1000
        : baseDelay * 2 ** (attempt - 1) + Math.random() * 200;  // jitter

      console.warn(`Attempt ${attempt} failed (${err.status}); retrying in ${delay}ms`);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### Timeout

```javascript
function withTimeout(promise, ms = 30000) {
  return Promise.race([
    promise,
    new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), ms)),
  ]);
}

const r = await withTimeout(
  openai.chat.completions.create({...}),
  20_000
);
```

> Streaming endpoint-এ chunk-level timeout (10s no chunk = abort) ব্যবহার করুন; full timeout-এ already half-streamed user-কে hang।

### Circuit Breaker

(see `12-concurrency/` for full pattern; once `circuit-breaker.md` exists, link there.)

```javascript
class CircuitBreaker {
  constructor({ threshold = 5, resetTimeoutMs = 30_000 }) {
    this.failures = 0;
    this.threshold = threshold;
    this.resetTimeoutMs = resetTimeoutMs;
    this.state = "CLOSED";   // CLOSED, OPEN, HALF_OPEN
    this.openedAt = 0;
  }

  async run(fn) {
    if (this.state === "OPEN") {
      if (Date.now() - this.openedAt > this.resetTimeoutMs) {
        this.state = "HALF_OPEN";
      } else {
        throw new Error("Circuit OPEN; skipping call");
      }
    }

    try {
      const result = await fn();
      if (this.state === "HALF_OPEN") this.reset();
      return result;
    } catch (err) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.state = "OPEN";
        this.openedAt = Date.now();
      }
      throw err;
    }
  }

  reset() { this.failures = 0; this.state = "CLOSED"; }
}

const breaker = new CircuitBreaker({ threshold: 5, resetTimeoutMs: 30_000 });
const r = await breaker.run(() => openai.chat.completions.create({...}));
```

> 📚 আরো details: পরবর্তী সংযোজন `12-concurrency/circuit-breaker.md`-এ pattern, fallback, bulkhead।

---

## 4️⃣ Fallback Model Chains

```
Primary failed (rate limit / outage / cost spike)?
   ↓
Try secondary (different provider, similar quality)
   ↓
Try cheap model (degraded but answer)
   ↓
Try self-hosted (if available)
   ↓
Cached "polite apology" response
```

### Implementation

```javascript
const MODEL_CHAIN = [
  { name: "gpt-4o",              client: openaiClient,   maxRetries: 2 },
  { name: "claude-3-5-sonnet",   client: anthropicClient,maxRetries: 2 },
  { name: "gpt-4o-mini",         client: openaiClient,   maxRetries: 1 },
  { name: "llama-3-70b-self",    client: vllmClient,     maxRetries: 1 },
];

async function chatWithFallback(messages, params) {
  let lastError;
  for (const m of MODEL_CHAIN) {
    try {
      return await callLLMWithRetry(
        () => m.client.chat.completions.create({ model: m.name, messages, ...params }),
        { maxAttempts: m.maxRetries }
      );
    } catch (err) {
      lastError = err;
      console.warn(`Model ${m.name} failed: ${err.message}; falling back`);
      metrics.increment("llm.fallback", { from: m.name });
    }
  }
  // last-resort cached apology
  return cachedApologyResponse(messages);
}
```

> ⚠️ **Quality drift:** fallback model output schema/style ভিন্ন হতে পারে। Eval suite আলাদাভাবে cover করুন।

---

## 5️⃣ Streaming Proxy

Frontend থেকে directly OpenAI key expose করা impossible (security)। তাই backend proxy → SSE pass-through।

```javascript
// Express SSE proxy
import express from "express";
const app = express();

app.post("/api/chat", express.json(), async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.setHeader("X-Accel-Buffering", "no");

  // Auth
  const userId = await authenticate(req);
  if (!userId) return res.status(401).end();

  // Rate limit
  if (!await rateLimitCheck(userId)) {
    res.write(`event: error\ndata: rate-limited\n\n`);
    return res.end();
  }

  // PII redaction
  const cleaned = redactPII(req.body.message);

  // Stream
  try {
    const stream = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: cleaned }],
      stream: true,
      stream_options: { include_usage: true },
    });

    let usage = null;
    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta?.content || "";
      if (delta) res.write(`data: ${JSON.stringify({ token: delta })}\n\n`);
      if (chunk.usage) usage = chunk.usage;
    }

    // Bill user
    await trackCost(userId, usage);
    res.write(`event: done\ndata: ${JSON.stringify({ usage })}\n\n`);
  } catch (err) {
    res.write(`event: error\ndata: ${JSON.stringify({ msg: err.message })}\n\n`);
  } finally {
    res.end();
  }
});
```

---

## 6️⃣ Observability — Tracing & Metrics

### Why traces?

LLM call একটা multi-step pipeline:
```
Embed query → Vector search → Rerank → LLM call (with tool calls) → Output validate
```

কোন step slow/buggy? Trace ছাড়া black box।

### Langfuse trace example

```javascript
import { Langfuse } from "langfuse";
const lf = new Langfuse();

const trace = lf.trace({
  name: "daraz-support-chat",
  userId: "user-123",
  sessionId: "sess-456",
  metadata: { tenant: "daraz-bd" },
});

const retrieval = trace.span({ name: "vector-retrieval" });
const docs = await vectorStore.search(query);
retrieval.end({ output: { count: docs.length } });

const llmCall = trace.generation({
  name: "answer-generation",
  model: "gpt-4o",
  modelParameters: { temperature: 0.3 },
  input: messages,
});
const r = await openai.chat.completions.create({ ... });
llmCall.end({
  output: r.choices[0].message,
  usage: r.usage,
});

trace.update({ output: r.choices[0].message.content });
await lf.flushAsync();
```

### LangSmith (LangChain ecosystem)

```javascript
import { Client } from "langsmith";
import { traceable } from "langsmith/traceable";

const tracedAnswer = traceable(
  async function answer(query) { /* ... */ },
  { name: "support-answer", project_name: "daraz-prod" }
);
```

### Helicone — proxy approach (zero code change)

```javascript
const openai = new OpenAI({
  baseURL: "https://oai.hconeai.com/v1",
  defaultHeaders: { "Helicone-Auth": `Bearer ${HELICONE_KEY}` },
});
// Now every call logged in Helicone with cost, latency, user
```

### What to track

```
Per request:
  - trace_id, user_id, tenant_id, session_id
  - prompt_version, model, parameters
  - input/output tokens, cost
  - TTFT, total latency
  - cache_hit (exact/semantic/none)
  - tool calls + results
  - eval scores (if computed)
  - errors

Aggregates:
  - p50/p95/p99 latency per model
  - $/day per tenant
  - Token volume per endpoint
  - Cache hit rate
  - Fallback rate
  - Error rate breakdown
```

---

## 7️⃣ Drift & Hallucination Detection

### Drift = output distribution change over time

কারণ:
- Provider silently upgraded model
- User input pattern changed (new feature, viral query)
- RAG corpus drift

**Detection:**
- Daily: random sample of N production queries → re-run with golden prompt
- Compare: avg output length, vocabulary distribution, eval score
- Alert if drift > threshold

### Hallucination Detection (groundedness)

RAG-এ retrieved context ছিল কি? Output সেটা থেকে এসেছে কি?

```javascript
async function checkGroundedness(answer, context) {
  const r = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "You are a strict fact-checker. Determine if every claim in ANSWER is supported by CONTEXT. Reply JSON: {grounded: bool, unsupported_claims: string[]}" },
      { role: "user", content: `CONTEXT:\n${context}\n\nANSWER:\n${answer}` },
    ],
    response_format: { type: "json_object" },
    temperature: 0,
  });
  return JSON.parse(r.choices[0].message.content);
}

const result = await checkGroundedness(answer, retrievedDocs);
if (!result.grounded) {
  console.warn("Hallucination!", result.unsupported_claims);
  // Fallback: "জানা নেই" reply, OR escalate
}
```

Production-grade: **RAGAS** library (Python) — automated metrics। Covered in `evaluations-and-safety.md`।

---

## 8️⃣ PII Redaction

User input/output-এ PII (phone, NID, account) থাকতে পারে। Provider log-এ leak হলে compliance violation।

### Bangladesh-specific PII

```
- Phone:        +88017XXXXXXXX, 017XXXXXXXX, ০১৭...
- NID:          10-digit (old) / 13-digit / 17-digit
- bKash/Nagad:  same as phone
- TIN:          12-digit
- Card number:  16-digit (Luhn)
- Email:        standard
- Address:      house/road/sector — harder to detect
```

### Regex-based fast path

```javascript
const BD_PATTERNS = {
  phone: /(\+?88)?0?1[3-9]\d{8}/g,
  nid_old: /\b\d{10}\b/g,
  nid_new: /\b\d{13}\b|\b\d{17}\b/g,
  email: /[\w.-]+@[\w.-]+\.\w+/g,
  card: /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g,
};

function redactPII(text) {
  let cleaned = text;
  for (const [type, re] of Object.entries(BD_PATTERNS)) {
    cleaned = cleaned.replace(re, `[${type.toUpperCase()}_REDACTED]`);
  }
  return cleaned;
}
```

### Microsoft Presidio (full-featured)

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

results = analyzer.analyze(text="My phone is +8801712345678", language="en")
anonymized = anonymizer.anonymize(text="...", analyzer_results=results)
```

### LLM-based redactor (slow but accurate)

For unstructured PII (addresses, names) — use small LLM as classifier; cache results।

---

## 9️⃣ Content Moderation

### OpenAI Moderation (free)

```javascript
const mod = await openai.moderations.create({
  input: userMessage,
  model: "omni-moderation-latest",
});
const flagged = mod.results[0].flagged;
const categories = mod.results[0].categories;
// { hate: false, harassment: false, sexual: false, violence: false, ... }
```

### Llama Guard (open-source, self-host)

Meta-এর fine-tuned Llama for safety classification — input + output দুটোই check করতে পারে।

```python
# vLLM-এ Llama Guard 3 host করে ব্যবহার
result = llm.complete(f"<|user|>{user_msg}<|assistant|>{model_output}")
# Output: "safe" or "unsafe\nS1, S5"  (categories)
```

### NeMo Guardrails (NVIDIA)

YAML-driven rule + LLM hybrid; jailbreak detection, topic restriction।

### Custom moderator chain

```
1. Input regex (fast, profanity, sensitive keywords BD context)
2. OpenAI moderation (medium speed, broad)
3. Llama Guard (more nuanced, self-host)
4. Output: schema validation + groundedness + brand-safety LLM
```

---

## 🔟 A/B Testing Prompts

### Why?

প্রতিটা prompt change ১০-৩০% cost/quality vary করে। Hunch দিয়ে decide করা যাবে না।

### Architecture

```
                    ┌────────────────┐
   request ──►      │ Experiment     │
                    │ assigner       │
                    │ (sticky hash on│
                    │  user_id)      │
                    └────┬───────────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       prompt v3 (95%)       prompt v4 (5%) ← canary
              │                     │
              └──────────┬──────────┘
                         ▼
                    track metrics:
                    - eval score
                    - user thumbs up/down
                    - conversation length
                    - escalation rate
                    - cost per session
```

### Implementation sketch

```javascript
function pickPromptVersion(userId) {
  const hash = crypto.createHash("md5").update(userId).digest()[0];   // 0-255
  const pct = (hash / 256) * 100;
  if (pct < 5) return "v4";    // 5% canary
  return "v3";                  // 95% control
}

const version = pickPromptVersion(userId);
const prompt = await lf.getPrompt("daraz-support", version);
const r = await callLLM(prompt.compile(vars));

await metrics.record({
  experiment: "daraz-support-v4-test",
  variant: version,
  user_id: userId,
  // ... outcome metrics
});
```

### Canary rollout for prompt changes

```
Day 1:   1% traffic → new prompt (alarm if eval drops)
Day 2:   5% traffic
Day 3:   25%
Day 5:   50%
Day 7:   100%  (with old as fallback)
```

---

## 1️⃣1️⃣ Governance & Audit Logs

Compliance (BD Bank-এর জন্য, Bangladesh Bank guideline, GDPR if EU users):

- প্রতিটা LLM call audit log: who, when, what input, what output, what action taken
- Retention 90 days minimum, 7 years for financial
- Right to deletion (user request → all their LLM logs delete)
- Tenant data isolation

### Audit log schema

```sql
CREATE TABLE llm_audit (
  id          UUID PRIMARY KEY,
  trace_id    UUID,
  tenant_id   TEXT NOT NULL,
  user_id     TEXT,
  endpoint    TEXT,
  model       TEXT,
  prompt_id   TEXT,
  prompt_ver  TEXT,
  input_redacted TEXT,
  output      TEXT,
  tools_called JSONB,
  cost_usd    DECIMAL(10,6),
  status      TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_audit_tenant_created ON llm_audit (tenant_id, created_at DESC);
```

---

## 1️⃣2️⃣ Multi-Tenant Architecture for SaaS LLM Apps

Imagine you're building "AI Customer Support SaaS" for Bangladesh — tenants: Daraz, Pathao, Foodpanda।

### Concerns

```
1. Tenant data isolation (vector DB, prompts, history)
2. Per-tenant cost cap
3. Per-tenant rate limit
4. Per-tenant model preference (GPT-4 vs Claude)
5. Per-tenant prompt customization
6. Per-tenant audit log
7. Noisy neighbor protection
```

### Tenant context propagation

```javascript
// Middleware
async function tenantMiddleware(req, res, next) {
  const apiKey = req.headers["x-api-key"];
  const tenant = await tenantStore.findByApiKey(apiKey);
  if (!tenant) return res.status(401).end();

  req.tenant = tenant;
  // every downstream call inherits
  als.run({ tenantId: tenant.id }, () => next());
}

// AsyncLocalStorage
import { AsyncLocalStorage } from "node:async_hooks";
const als = new AsyncLocalStorage();

function currentTenant() {
  return als.getStore()?.tenantId;
}

async function callLLM(messages, opts) {
  const tenantId = currentTenant();

  // Per-tenant model preference
  const config = await tenantStore.getConfig(tenantId);
  const model = config.preferredModel || "gpt-4o-mini";

  // Per-tenant rate limit
  if (!await tenantRateLimit(tenantId)) {
    throw new Error("Tenant rate limit");
  }

  // Per-tenant cost cap
  const monthlyCost = await tenantCostStore.getMonthly(tenantId);
  if (monthlyCost > config.monthlyCostCap) {
    throw new Error("Tenant monthly budget exceeded");
  }

  const r = await openai.chat.completions.create({ model, messages });

  await tenantCostStore.add(tenantId, computeCost(r.usage, model));
  await audit.log({ tenantId, ...r });

  return r;
}
```

### Per-tenant vector store namespace

```javascript
// Pinecone — namespace
await index.namespace(tenant.id).upsert({...});
const results = await index.namespace(tenant.id).query({...});

// pgvector — separate schema or row-level filter
SELECT ... FROM products WHERE tenant_id = $1 ORDER BY embedding <=> $2 LIMIT 10
```

---

## 1️⃣3️⃣ Cost Capping at Daraz Scale (PHP API Gateway)

**Scenario:** Daraz has 50+ internal teams using OpenAI. একজন junior dev একটা bug লিখলো — infinite loop calling GPT-4। ৬ ঘন্টায় $50,000 burn।

**Solution:** internal API gateway, every team uses gateway URL — gateway enforce cost cap, audit, model routing।

```php
<?php
// llm-gateway.php (Symfony / Laravel / vanilla)

require 'vendor/autoload.php';
use Predis\Client as Redis;

$redis = new Redis(['host' => 'redis.internal']);
$pdo   = new PDO('pgsql:host=audit-db;dbname=llm_gateway', '...', '...');

function authenticate(): array {
    $token = $_SERVER['HTTP_X_INTERNAL_TOKEN'] ?? '';
    $teamRow = (new PDO(...))->query("SELECT * FROM teams WHERE token='$token'")->fetch();
    if (!$teamRow) { http_response_code(401); exit; }
    return $teamRow;
}

function checkBudget(Redis $redis, string $teamId, float $monthlyCap): void {
    $key = "team:$teamId:cost:" . date('Ym');
    $current = (float)($redis->get($key) ?? 0);
    if ($current >= $monthlyCap) {
        http_response_code(429);
        echo json_encode(['error' => 'Monthly budget exceeded for team', 'limit'=>$monthlyCap]);
        exit;
    }
}

function checkRateLimit(Redis $redis, string $teamId): void {
    $key = "team:$teamId:rl:" . floor(time()/60);
    $count = $redis->incr($key);
    $redis->expire($key, 70);
    if ($count > 100) {  // 100 RPM per team
        http_response_code(429);
        exit;
    }
}

function callOpenAI(array $body): array {
    $ch = curl_init('https://api.openai.com/v1/chat/completions');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($body),
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . getenv('OPENAI_KEY'),
            'Content-Type: application/json',
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 60,
    ]);
    $resp = curl_exec($ch);
    return json_decode($resp, true);
}

function computeCost(array $usage, string $model): float {
    $pricing = [
        'gpt-4o'      => ['in' => 2.50, 'out' => 10.00],
        'gpt-4o-mini' => ['in' => 0.15, 'out' => 0.60],
    ];
    $p = $pricing[$model] ?? ['in' => 0, 'out' => 0];
    return ($usage['prompt_tokens']     / 1e6) * $p['in']
         + ($usage['completion_tokens'] / 1e6) * $p['out'];
}

// Main
$team = authenticate();
checkRateLimit($redis, $team['id']);
checkBudget($redis, $team['id'], (float)$team['monthly_cap_usd']);

$body = json_decode(file_get_contents('php://input'), true);

// Force allowed models per team
$allowed = json_decode($team['allowed_models'], true);
if (!in_array($body['model'], $allowed)) {
    http_response_code(403);
    echo json_encode(['error' => 'Model not allowed for team', 'allowed' => $allowed]);
    exit;
}

// Call
$start = microtime(true);
$response = callOpenAI($body);
$latency = (microtime(true) - $start) * 1000;

// Track cost
if (isset($response['usage'])) {
    $cost = computeCost($response['usage'], $body['model']);
    $redis->incrbyfloat("team:{$team['id']}:cost:" . date('Ym'), $cost);

    // Audit
    $stmt = $pdo->prepare("INSERT INTO llm_audit
        (team_id, model, input_tokens, output_tokens, cost_usd, latency_ms, created_at)
        VALUES (?, ?, ?, ?, ?, ?, NOW())");
    $stmt->execute([
        $team['id'], $body['model'],
        $response['usage']['prompt_tokens'],
        $response['usage']['completion_tokens'],
        $cost, $latency,
    ]);
}

header('Content-Type: application/json');
echo json_encode($response);
```

**Internal teams use:** `https://llm-gateway.daraz.internal/v1/chat/completions` (drop-in OpenAI-compatible)।

---

## 🔁 Putting It All Together — Production Pipeline

```javascript
// production-llm.js — full pipeline
import { Langfuse } from "langfuse";
import { CircuitBreaker } from "./circuit-breaker.js";

const lf = new Langfuse();
const breaker = new CircuitBreaker({ threshold: 5, resetTimeoutMs: 30000 });

export async function answerSupportQuery({
  tenantId, userId, sessionId, query, language = "bn",
}) {
  const trace = lf.trace({ name: "support-answer", userId, sessionId, metadata: { tenantId } });

  // 1. Input guardrail
  const cleaned = redactPII(query);
  const moderation = await openai.moderations.create({ input: cleaned });
  if (moderation.results[0].flagged) {
    trace.update({ output: "MODERATION_BLOCK" });
    return { response: "এই বিষয়ে সাহায্য করতে পারছি না।", blocked: true };
  }

  // 2. Cache (exact)
  const cacheKey = `${tenantId}:${language}:${hash(cleaned)}`;
  const cached = await redis.get(cacheKey);
  if (cached) { trace.update({ output: "CACHE_HIT" }); return JSON.parse(cached); }

  // 3. Semantic cache
  const semCached = await semanticCache(cleaned, tenantId);
  if (semCached) { trace.update({ output: "SEM_CACHE_HIT" }); return semCached; }

  // 4. RAG retrieval
  const retr = trace.span({ name: "retrieval" });
  const docs = await vectorSearch(cleaned, { tenantId, k: 5 });
  retr.end({ output: { count: docs.length } });

  // 5. Prompt registry
  const promptObj = await lf.getPrompt(`${tenantId}-support`, undefined, { label: "production" });
  const messages = promptObj.compile({ docs, query: cleaned, language });

  // 6. LLM call with fallback + circuit breaker
  const gen = trace.generation({ name: "llm", model: "gpt-4o", input: messages });
  let response;
  try {
    response = await breaker.run(() =>
      callLLMWithRetry(() => chatWithFallback(messages, { temperature: 0.3, max_tokens: 600 }))
    );
  } catch (err) {
    gen.end({ output: { error: err.message }, level: "ERROR" });
    return { response: "আমি এখন সাহায্য করতে পারছি না, পরে চেষ্টা করুন।", error: true };
  }
  gen.end({ output: response.choices[0].message, usage: response.usage });

  // 7. Output guardrail
  const groundedness = await checkGroundedness(response.choices[0].message.content, docs);
  if (!groundedness.grounded) {
    trace.update({ metadata: { hallucination: true, claims: groundedness.unsupported_claims } });
    // fallback to "I don't know" or escalate
  }

  // 8. Cost tracking
  const cost = computeCost(response.usage, "gpt-4o");
  await tenantCostStore.add(tenantId, cost);

  // 9. Cache result
  await redis.set(cacheKey, JSON.stringify(response), { EX: 3600 });

  // 10. Audit
  await audit.log({ tenantId, userId, sessionId, query: cleaned, response: response.choices[0].message.content, cost, model: "gpt-4o" });

  trace.update({ output: response.choices[0].message.content });
  await lf.flushAsync();

  return { response: response.choices[0].message.content, traceId: trace.id };
}
```

---

## 🚫 Common Pitfalls

1. **Logging full prompt + response without redaction** — PII leak in logs!
2. **Cache key user-specific data ছাড়া (tenant_id, language)** — cross-tenant leak।
3. **Circuit breaker без semaphore** — concurrent failure spike still passes through।
4. **Streaming endpoint cost track না করা** — usage object না আসা পর্যন্ত wait।
5. **Fallback chain-এ একই provider বার বার** — provider outage হলে full chain fail।
6. **Prompt hot-reload ছাড়া** — production-এ prompt fix করতে full deploy লাগে।
7. **Eval CI-তে integrate না করা** — bad prompt merge হয়ে যায়।
8. **Token cost provider-পক্ষে calculate** — provider price পরিবর্তন করলে gateway আপডেট হয় না।
9. **Per-tenant noisy neighbor** — একজন tenant-এর spike সবাইকে slow করে; bulkhead pattern দরকার।
10. **Rate limit-এ Retry-After honor না করা** — provider আপনাকে blacklist করে।

---

## 🧰 Senior Engineer Checklist

- [ ] Every LLM call traced (Langfuse/LangSmith/Helicone)
- [ ] Prompt versioned, label-driven rollout
- [ ] Eval suite gated in CI before prompt merge
- [ ] Exact + semantic cache, hit rate dashboard
- [ ] Provider-side prompt caching (Anthropic/OpenAI/Gemini) used where applicable
- [ ] Retry with jitter + circuit breaker
- [ ] Multi-provider fallback chain configured
- [ ] PII redaction pre-call, output scrub post-call
- [ ] Moderation API on input AND output
- [ ] Per-tenant cost cap (daily + monthly)
- [ ] Per-tenant rate limit (RPM + RPD)
- [ ] Audit log immutable, retention policy aligned with compliance
- [ ] Drift monitor (daily golden-set re-run)
- [ ] Hallucination check on RAG outputs
- [ ] Canary rollout for prompt + model changes
- [ ] Documented runbook (model outage → fallback steps)

---

## 📚 আরো পড়ুন

- "The Rise of LLMOps" — DeepLearning.AI short course
- Langfuse docs: langfuse.com/docs
- Helicone blog: helicone.ai/blog
- "LLMs in Production" book — Manning
- OpenAI prompt caching docs
- Anthropic prompt caching guide
- Microsoft Presidio (PII): microsoft.github.io/presidio/

→ পরবর্তী: [Fine-tuning vs RAG](./fine-tuning-vs-rag.md) — knowledge inject করার সিদ্ধান্ত।
