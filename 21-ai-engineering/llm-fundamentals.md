# 🧠 LLM Fundamentals — ভিতরে কী হচ্ছে?

## 📖 সংজ্ঞা ও মূল ধারণা

**Large Language Model (LLM)** হলো একটি neural network — সাধারণত **Transformer architecture**-ভিত্তিক — যা কয়েক বিলিয়ন (billion) থেকে কয়েক ট্রিলিয়ন (trillion) parameter নিয়ে বিশাল text corpus-এর উপর train করা। মূল task: একটা sequence-এর পরের token কী হবে সেটা probabilistically predict করা (autoregressive next-token prediction)।

```
Input:  "ঢাকার আবহাওয়া আজ"
        │
        ▼  (Tokenize)
Tokens: [ঢাকা, র, আব, হাওয়া, আজ]
        │
        ▼  (Embed → Transformer layers → Softmax)
Output distribution: { "গরম": 0.31, "ঠান্ডা": 0.18, "ভালো": 0.12, ... }
        │
        ▼  (Sample with temperature/top-p)
Next token: "গরম"
        │
        ▼  (Append, repeat)
"ঢাকার আবহাওয়া আজ গরম"
```

LLM-এর "intelligence" এই simple objective থেকে **emergent** — yet billions of parameters + trillion tokens-এ এটা reasoning, code, translation, summary সব কিছু shockingly ভালো করে।

---

## 🏛️ Transformer Architecture — Conceptual Tour

২০১৭ সালে Google-এর "Attention is All You Need" paper-এ Transformer প্রস্তাবিত হয়। এর আগে RNN/LSTM ব্যবহার হতো, কিন্তু sequential processing-এর কারণে slow এবং long-range dependency capture করতে পারতো না।

### ১. Self-Attention — মূল innovation

প্রতিটা token একই sequence-এর অন্য সব token-কে "দেখে" এবং ঠিক করে কে কাকে কতটা গুরুত্ব দেবে।

```
"বিড়ালটা ইঁদুর ধরেছে কারণ সে ক্ষুধার্ত ছিল"
                                  ▲
                                  │
                       "সে" attention দেয়:
                       - বিড়ালটা: 0.78  ← high (subject)
                       - ইঁদুর:    0.15
                       - কারণ:    0.04
                       - ক্ষুধার্ত: 0.03
```

Mathematically (conceptual):

```
Q = X·W_Q   (Query  — "আমি কী খুঁজছি?")
K = X·W_K   (Key    — "আমার কাছে কী আছে?")
V = X·W_V   (Value  — "আমি কী দিতে পারি?")

Attention(Q,K,V) = softmax(Q·Kᵀ / √d_k) · V
                              │
                              └── scale factor; gradient stable রাখে
```

প্রতিটা token-এর representation update হয় — অন্য সব token-এর Value-গুলোর weighted sum দিয়ে, যেখানে weight = Q·K dot product।

### ২. Multi-Head Attention

একটা attention মাত্র একটা "perspective" capture করে। Multi-head মানে parallel-এ ৮/১৬/৩২ টা আলাদা attention head — একেকটা একেক ধরনের relationship শেখে (syntactic, semantic, coreference, ইত্যাদি)।

```
Input ──┬─► Head 1 (syntactic relations) ──┐
        ├─► Head 2 (long-range references)─┤
        ├─► Head 3 (semantic similarity) ──┼── Concat ──► Linear ──► Output
        ├─► ...                            │
        └─► Head N (positional patterns) ──┘
```

### ৩. Positional Encoding

Attention "set"-এর মত — order জানে না। তাই token position inject করতে হয়। দুটো main approach:

| পদ্ধতি | কীভাবে | ব্যবহার করে |
|--------|--------|-------------|
| **Sinusoidal** (original) | sin/cos function position-এর | original Transformer |
| **Learned absolute** | প্রতিটা position-এর জন্য learnable vector | GPT-2, BERT |
| **RoPE (Rotary)** | rotation matrix Q,K-এ apply | Llama, Mistral, GPT-NeoX |
| **ALiBi** | attention score-এ linear bias | BLOOM, MPT |

RoPE বর্তমানে dominant — context window extension সহজ (NTK-aware scaling, YaRN দিয়ে 4K → 128K)।

### ৪. Feed-Forward Network (FFN) / MLP

প্রতিটা attention block-এর পরে একটা position-wise MLP:

```
FFN(x) = max(0, x·W₁ + b₁)·W₂ + b₂   (ReLU ভিত্তিক, original)
       = SwiGLU/GEGLU variants         (modern: Llama, GPT-4)
```

FFN parameter Transformer-এর প্রায় ⅔ ধারণ করে — এখানে "knowledge" stored বলে অনেক research বলে।

### ৫. Residual + LayerNorm

প্রতিটা sub-layer wrap করা হয়:
```
output = LayerNorm(x + Sublayer(x))   (post-norm — original)
output = x + Sublayer(LayerNorm(x))   (pre-norm — modern, training stable)
```

### ৬. Decoder-only Block (GPT-style)

```
        ┌─────────────────────────────┐
        │   Token + Positional Embed   │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │  ┌─────────────────────┐    │
        │  │ Masked Self-Attention│    │ ◄── causal mask: future hide
        │  └──────────┬──────────┘    │
        │             ▼                │
        │  ┌─────────────────────┐    │
        │  │     LayerNorm       │    │
        │  └──────────┬──────────┘    │
        │             ▼                │
        │  ┌─────────────────────┐    │
        │  │      FFN/MLP        │    │
        │  └──────────┬──────────┘    │
        │             ▼                │
        │       LayerNorm              │
        │             │                │
        │  (× N layers, e.g., 96 in    │
        │   GPT-3 175B)                │
        └──────────────┬──────────────┘
                       ▼
              Linear → Softmax → Token probs
```

---

## 🔤 Tokens, BPE, Vocabulary

LLM character বা word নিয়ে কাজ করে না — **subword token** নিয়ে কাজ করে।

### BPE (Byte-Pair Encoding) — কীভাবে কাজ করে

```
Initial vocab: প্রতিটা character আলাদা token
Iteratively merge সবচেয়ে frequent pair:

Step 1: ['ঢ', 'া', 'ক', 'া', 'র']
Step 2: pair ('া','ক') frequent → merge → 'াক'
Step 3: ['ঢ', 'াক', 'া', 'র']
... continue until vocab size limit (e.g., 50K, 100K, 200K)
```

**Modern variants:**
- **BPE** — GPT, Llama
- **SentencePiece** (Google) — language-agnostic, no whitespace pre-tokenization, T5/Llama
- **Tiktoken** — OpenAI-এর fast Rust BPE

### বাংলা টোকেনাইজেশনের সমস্যা

```
Text: "আমি বাংলাদেশে থাকি"  (English: "I live in Bangladesh")

GPT-4 (cl100k):    [Tokens: ~14, byte fallback heavy]
Llama 2 (32K):     [Tokens: ~22]
Llama 3 (128K):    [Tokens: ~10]  ← অনেক ভালো
GPT-4o (o200k):    [Tokens: ~9]   ← best for Bangla
```

**বাস্তব প্রভাব:** Bangla content-এ token bloat ২x-৪x English-এর তুলনায়। মানে:
- $/token cost double-triple
- Effective context window অর্ধেক
- Latency বেশি

**সমাধান:**
- বেশি Bangla support করা model (GPT-4o, Claude, Llama 3+) ব্যবহার
- Custom tokenizer train করা (নিজের domain-এ Bangla token efficient হবে)
- BPE expansion (vocabulary extension) — fine-tuning-এ cover করব

### Tokenization কোডে দেখা

```javascript
// JavaScript — tiktoken
import { encoding_for_model } from "tiktoken";

const enc = encoding_for_model("gpt-4o");
const text = "ঢাকার আবহাওয়া আজ গরম";
const tokens = enc.encode(text);

console.log(`Text: ${text}`);
console.log(`Tokens: ${tokens.length}`);
console.log(`Token IDs: ${[...tokens]}`);
// সাবধান: tiktoken Bangla-তে byte-fallback করে — token গুনতে কাজে লাগে
enc.free();
```

```php
<?php
// PHP — yethan/tiktoken বা rubix/tensor; অথবা OpenAI API থেকে usage object থেকে count নাও
require 'vendor/autoload.php';
use Yethee\Tiktoken\EncoderProvider;

$provider = new EncoderProvider();
$encoder = $provider->getForModel('gpt-4o');

$text = "ঢাকার আবহাওয়া আজ গরম";
$tokens = $encoder->encode($text);

echo "Token count: " . count($tokens) . "\n";
echo "Tokens: " . implode(',', $tokens) . "\n";
```

---

## 🧬 Decoder-only vs Encoder-only vs Encoder-Decoder

```
┌──────────────────┬─────────────────┬──────────────────┬──────────────────┐
│ Architecture     │ Encoder-only    │ Decoder-only     │ Encoder-Decoder  │
├──────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ Attention mask   │ Bidirectional   │ Causal (left)    │ Enc=bi, Dec=causal│
│ Best for         │ Understanding   │ Generation       │ Seq2Seq          │
│ Examples         │ BERT, RoBERTa,  │ GPT-3/4, Llama,  │ T5, BART,        │
│                  │ ELECTRA         │ Claude, Gemini   │ Flan-T5          │
│ Use case         │ Classification, │ Chatbot, code,   │ Translation,     │
│                  │ embedding,      │ summarize,       │ summarization    │
│                  │ NER             │ creative writing │ (older approach) │
│ Today (2026)     │ Niche (embed)   │ Dominant ✓       │ Niche (specific) │
└──────────────────┴─────────────────┴──────────────────┴──────────────────┘
```

আজকের সব major LLM (GPT-4o, Claude, Gemini, Llama, Mistral, Qwen, DeepSeek) **decoder-only**। কারণ:
- Single objective (next-token) → simple, scales well
- In-context learning emergent property
- Multi-task একই architecture-এ

---

## 🎓 Training Pipeline: Pretraining → SFT → RLHF/DPO

```
                  ┌─────────────────────────────────────────┐
                  │        STAGE 1: PRETRAINING              │
                  │  Data: ১০-১৫ ট্রিলিয়ন token (web, books, │
                  │        code, papers)                     │
                  │  Objective: next-token prediction        │
                  │  Compute: ১০² - ১০⁴ H100 GPU, weeks     │
                  │  Output: "base model" (raw, not chat)    │
                  └────────────────────┬────────────────────┘
                                       ▼
                  ┌─────────────────────────────────────────┐
                  │   STAGE 2: SFT (Supervised Fine-Tuning)  │
                  │   = Instruction Tuning                   │
                  │  Data: ~১০K-১M (instruction, response)   │
                  │  Objective: same next-token, but on      │
                  │             curated chat format          │
                  │  Output: "instruct model" — chat-ready   │
                  └────────────────────┬────────────────────┘
                                       ▼
                  ┌─────────────────────────────────────────┐
                  │   STAGE 3: ALIGNMENT (RLHF / DPO)        │
                  │  Data: human preference (A vs B)         │
                  │  Method:                                 │
                  │    • RLHF — train reward model + PPO     │
                  │    • DPO  — direct optimization,         │
                  │             reward model লাগে না — সহজ   │
                  │  Output: aligned model (helpful,         │
                  │          harmless, honest)               │
                  └─────────────────────────────────────────┘
```

### Why each stage?

- **Pretraining** language understanding, world knowledge দেয় — কিন্তু "প্রশ্ন উত্তর" জানে না; auto-complete করে।
- **SFT** chat format শেখায়; "user: ... assistant: ..."।
- **RLHF/DPO** preference align করে — উত্তর কেমন হলে human like করে সেটা শেখায়। Helpfulness + safety।

### DPO (Direct Preference Optimization) — RLHF-এর সহজ alternative

RLHF complex (separate reward model + PPO unstable)। ২০২৩-এ Stanford-এর paper DPO প্রস্তাব করে — directly preference data দিয়ে policy update। Loss:

```
L_DPO = -log σ( β · log(π_θ(y_w|x)/π_ref(y_w|x))
              - β · log(π_θ(y_l|x)/π_ref(y_l|x)) )

y_w = preferred (winner)
y_l = rejected (loser)
β   = strength of KL constraint
```

আজ Llama 3, Mistral, Qwen — সবাই DPO/IPO/KTO ব্যবহার করছে।

---

## 📏 Context Window & KV Cache

### Context Window

মডেল একটা single forward pass-এ যত token process করতে পারে।

| Model | Context window | Notes |
|-------|----------------|-------|
| GPT-3 | 2K-4K | পুরোনো |
| GPT-4 (orig) | 8K-32K | |
| GPT-4o | 128K | input, output 16K |
| Claude 3.5 Sonnet | 200K | |
| Claude 3 Opus | 200K | |
| Gemini 1.5 Pro | 2M | longest production |
| Llama 3.1 | 128K | |
| Mistral Large 2 | 128K | |
| Qwen 2.5 | 128K (7B), 1M (special) | |

**সতর্কতা:** "lost in the middle" effect — long context-এ middle portion-এর information ignore করতে পারে। প্রায় ৩২K-এর পর performance degrade শুরু হয় most models-এ। ALWAYS evaluate করুন, blindly trust করবেন না।

### KV Cache — Inference-এর secret weapon

Decoder-only autoregressive generation-এ প্রতিটা new token-এ আগের সব tokens-এর Key, Value আবার compute করতে হবে — quadratic cost!

**Solution:** আগের tokens-এর K, V cache করে রাখি; new token-এর জন্য শুধু one column compute করি।

```
Without KV cache: token N তৈরিতে O(N²) computation
With KV cache:    token N তৈরিতে O(N) computation
                  কিন্তু memory: O(N · num_layers · 2 · d_model)
```

**বাস্তব প্রভাব:**
- KV cache memory bottleneck → batch size limit
- Llama 3 70B-এ 1 user, 32K context → KV cache প্রায় 20GB
- vLLM-এর **PagedAttention** এই memory virtualize করে → 24x throughput

```
KV cache size = 2 × n_layers × n_heads × head_dim × seq_len × batch × dtype_bytes
              = 2 × 80 × 64 × 128 × 32000 × 1 × 2 (fp16)
              ≈ 41 GB  (Llama 3 70B, 32K context, batch 1)
```

---

## 🏆 Open vs Closed Models — Comparison (২০২৬)

| Model | Provider | Type | Context | Bangla | Strength | $/1M in/out |
|-------|----------|------|---------|--------|----------|-------------|
| **GPT-4o** | OpenAI | Closed | 128K | ⭐⭐⭐⭐ | Multimodal, code, function | $2.50 / $10 |
| **GPT-4o mini** | OpenAI | Closed | 128K | ⭐⭐⭐ | Cheap, fast | $0.15 / $0.60 |
| **o1 / o3** | OpenAI | Closed | 128K | ⭐⭐⭐⭐ | Reasoning (chain-of-thought built-in) | $15-60 |
| **Claude 3.5 Sonnet** | Anthropic | Closed | 200K | ⭐⭐⭐⭐⭐ | Writing, code, long context | $3 / $15 |
| **Claude 3 Haiku** | Anthropic | Closed | 200K | ⭐⭐⭐⭐ | Cheap, fast | $0.25 / $1.25 |
| **Gemini 1.5 Pro** | Google | Closed | 2M | ⭐⭐⭐⭐ | Longest context, multimodal | $1.25 / $5 |
| **Gemini 1.5 Flash** | Google | Closed | 1M | ⭐⭐⭐ | Cheapest at scale | $0.075 / $0.30 |
| **Llama 3.1 70B** | Meta | Open | 128K | ⭐⭐⭐ | Best open ~70B | self-host |
| **Llama 3.1 405B** | Meta | Open | 128K | ⭐⭐⭐⭐ | Frontier-level open | self-host |
| **Mistral Large 2** | Mistral | Open weights | 128K | ⭐⭐⭐ | Code, function calling | $2 / $6 (API) |
| **Qwen 2.5 72B** | Alibaba | Open | 128K | ⭐⭐⭐ | Strong multilingual, code | self-host |
| **DeepSeek V3 / R1** | DeepSeek | Open | 64-128K | ⭐⭐ | Code, reasoning, super cheap | $0.14 / $0.28 |
| **Aya 23 / Aya Expanse** | Cohere | Open | 8-128K | ⭐⭐⭐⭐ | Specifically multilingual (Bangla included) | self-host |

**Bangla-specific recommendation:**
- Best Bangla output quality: **Claude 3.5 Sonnet**
- Best Bangla open-source: **Aya Expanse 32B**, then **Qwen 2.5 72B**
- Cheapest production-grade Bangla: **GPT-4o mini** or **Claude Haiku**

---

## 🎛️ Inference Parameters — গভীর ভাবে

```
                        Logits (prob over vocab)
                                │
                    ┌───────────┼────────────┐
                    ▼           ▼            ▼
              Temperature   Top-K         Top-P
                                          (Nucleus)
                    │           │            │
                    └───────────┼────────────┘
                                ▼
                        Sampled token
```

### `temperature` (0.0 - 2.0)

Logits-কে temperature দিয়ে divide করে softmax → distribution sharpen বা flatten।

```
T = 0.0 → argmax (deterministic, greedy)
T = 0.7 → balanced (default)
T = 1.0 → original distribution
T = 2.0 → very random, creative
```

**গাইডলাইন:**
- Code generation: 0.0 - 0.3
- Q&A, RAG: 0.0 - 0.5
- Creative writing: 0.7 - 1.0
- Brainstorming: 1.0 - 1.5

### `top_p` (Nucleus sampling, 0.0 - 1.0)

Probability mass-এর top P% রাখে। `top_p=0.9` মানে যে token-গুলো মিলিয়ে ৯০% probability — শুধু সেগুলো থেকে sample।

### `top_k` (1 - vocab_size)

Top K most probable token-এ limit। `top_k=50` মানে শুধু top 50 candidate।

> **Best practice:** **temperature** OR **top_p** ব্যবহার করুন, দুটো একসাথে নয়। OpenAI doc-এ এই recommendation।

### `max_tokens`

Output length cap। সাবধান:
- Cost control করে
- কিন্তু mid-sentence cut হতে পারে → finish_reason check করুন

### `frequency_penalty` (-2.0 to 2.0) ও `presence_penalty`

- **frequency_penalty:** যত বার token appear, তত penalty (repetition reduce)
- **presence_penalty:** একবার appear করলেই penalty (topic diversity)

```
frequency_penalty = 0.5 → repetitive marketing copy ঠিক হয়
presence_penalty  = 0.6 → একই topic-এ আটকে না থেকে নতুন বিষয়ে যাবে
```

### `logit_bias`

নির্দিষ্ট token-এর probability manually adjust। Use cases:
- "Yes/No" classifier — শুধু `Yes` (token X) ও `No` (token Y) bias কর +100, বাকিগুলো -100
- Banned words — bias = -100
- Forced format

### `stop` sequences

যেই string-এ পৌঁছালে generation থামবে। RAG/agent-এ critical।

```javascript
stop: ["</answer>", "\n\nUser:", "```\n\n"]
```

### `seed` (newer)

Reproducibility-এর জন্য। কিন্তু not 100% — provider warns "best effort"।

---

## ⚡ Streaming Responses

LLM token by token generate করে। User-কে wait করানোর বদলে token-গুলো আসা মাত্রই stream করুন। UX-এ ৩-৫x perceived speed।

### Server-Sent Events (SSE) — standard transport

```
GET /chat HTTP/1.1
Accept: text/event-stream

──── Response ────
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"token": "ঢাকা"}

data: {"token": "র"}

data: {"token": " আবহাওয়া"}

data: [DONE]
```

দেখুন `14-networking/` SSE vs WebSocket comparison।

### JavaScript — Vercel AI SDK (recommended for Next.js)

```javascript
// app/api/chat/route.ts (Next.js Route Handler)
import { openai } from "@ai-sdk/openai";
import { streamText } from "ai";

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: openai("gpt-4o"),
    messages,
    temperature: 0.3,
    maxTokens: 1024,
  });

  return result.toDataStreamResponse();
}
```

```javascript
// Frontend — hook
"use client";
import { useChat } from "ai/react";

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({ api: "/api/chat" });

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id} className={m.role === "user" ? "user" : "ai"}>
          {m.content}
        </div>
      ))}
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="বাংলায় প্রশ্ন করুন..."
        />
        <button disabled={isLoading}>পাঠান</button>
      </form>
    </div>
  );
}
```

### JavaScript — vanilla OpenAI SDK with streaming

```javascript
import OpenAI from "openai";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function streamChat(userMessage) {
  const stream = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "system", content: "তুমি Daraz-এর support agent, সবসময় বাংলায় উত্তর দাও।" },
      { role: "user", content: userMessage },
    ],
    temperature: 0.3,
    stream: true,
    stream_options: { include_usage: true }, // token usage track করতে
  });

  let fullText = "";
  let usage = null;

  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta?.content || "";
    process.stdout.write(delta);
    fullText += delta;
    if (chunk.usage) usage = chunk.usage;
  }

  console.log(`\n\nTokens — input: ${usage.prompt_tokens}, output: ${usage.completion_tokens}`);
  return fullText;
}

streamChat("আমার order #12345 কোথায়?");
```

### PHP — openai-php/client

```php
<?php
// composer require openai-php/client
require 'vendor/autoload.php';

$client = OpenAI::client(getenv('OPENAI_API_KEY'));

$stream = $client->chat()->createStreamed([
    'model'       => 'gpt-4o',
    'messages'    => [
        ['role' => 'system', 'content' => 'তুমি Daraz-এর support agent।'],
        ['role' => 'user',   'content' => 'আমার অর্ডার কোথায়?'],
    ],
    'temperature' => 0.3,
    'max_tokens'  => 512,
]);

foreach ($stream as $response) {
    $delta = $response->choices[0]->delta->content ?? '';
    echo $delta;
    flush();              // SSE-এর জন্য buffer flush
    ob_flush();
}
```

```php
<?php
// Bonus: PHP থেকে SSE response যা frontend pour করবে
header('Content-Type: text/event-stream');
header('Cache-Control: no-cache');
header('X-Accel-Buffering: no');  // Nginx buffer disable

foreach ($stream as $response) {
    $delta = $response->choices[0]->delta->content ?? '';
    if ($delta !== '') {
        echo "data: " . json_encode(['token' => $delta]) . "\n\n";
        flush();
        ob_flush();
    }
}
echo "data: [DONE]\n\n";
```

---

## 💰 Cost Models — Tokens are money

```
Cost = (input_tokens × input_$/1M) + (output_tokens × output_$/1M)
```

**বাস্তব হিসাব — Daraz support chatbot:**

```
দিনে ১০,০০০ conversation × গড় ৫ turn × turn-এ:
   - input: 800 tokens (system + history + user)
   - output: 200 tokens

দৈনিক token:
   - input:  10K × 5 × 800 = 40M tokens
   - output: 10K × 5 × 200 = 10M tokens

GPT-4o cost:
   - input:  40M × $2.50/1M = $100
   - output: 10M × $10/1M   = $100
   - মোট:    $200/day = $6,000/month = ~৭ লক্ষ টাকা

GPT-4o mini cost:
   - input:  40M × $0.15/1M = $6
   - output: 10M × $0.60/1M = $6
   - মোট:    $12/day = $360/month = ~৪২,০০০ টাকা

Self-hosted Llama 3.1 70B (4x H100 @ $2/hr):
   - $8/hr × 24 × 30 = $5,760/month (full utilization দরকার)
```

**Cost optimization patterns:**
1. **Tier routing** — easy queries → mini model, hard → big model
2. **Caching** — semantic + exact (covered in `llm-ops.md`)
3. **Prompt compression** — system prompt summarize, history truncate
4. **Batch API** (OpenAI/Anthropic batch) — 50% discount, 24-hr SLA
5. **Prompt caching** (Anthropic, Gemini) — same prefix → 90% discount

---

## ⏱️ Latency Factors

```
┌──────────────────────────────────────────────────────────────┐
│  Total Latency = TTFT + (output_tokens / TPS)                │
├──────────────────────────────────────────────────────────────┤
│  TTFT = Time To First Token                                  │
│       = network + queue + prefill (input processing)         │
│  TPS  = Tokens Per Second (decode speed)                     │
└──────────────────────────────────────────────────────────────┘
```

**বাস্তব numbers (২০২৬, ballpark):**

| Model | TTFT (p50) | TPS (output) |
|-------|-----------|--------------|
| GPT-4o | 400-800ms | 80-120 |
| GPT-4o mini | 300-500ms | 100-150 |
| Claude 3.5 Sonnet | 600-1200ms | 60-90 |
| Claude Haiku | 300-500ms | 120-180 |
| Gemini Flash | 200-400ms | 150-250 |
| Groq Llama 3 70B | 200ms | 250-500 ⚡ |
| Self-host Llama 3 70B (vLLM, 8xA100) | 500-1500ms | 40-80 |

**Optimization:**
- **Streaming** — TTFT-ই matter করে
- **Speculative decoding** — small model draft → big model verify (2-3x speedup)
- **Quantization** — fp16 → int8 → int4 (vLLM, Ollama)
- **Continuous batching** (vLLM) — multiple requests একসাথে process
- **PagedAttention** — KV cache memory efficient
- **Prefix caching** — same system prompt reuse

---

## 🏠 Hosted vs Self-Hosted

```
┌────────────────────┬──────────────────┬─────────────────────┐
│                    │ Hosted (API)     │ Self-Hosted         │
├────────────────────┼──────────────────┼─────────────────────┤
│ Setup time         │ ৫ মিনিট          │ দিন/সপ্তাহ          │
│ Initial cost       │ $0               │ GPU $$$ ($30K+)     │
│ Per-token cost     │ Pay per use      │ Fixed (utilization) │
│ Privacy/Compliance │ Data goes out    │ Full control        │
│ Scaling            │ Provider         │ আপনি                │
│ Latency control    │ Limited          │ Full                │
│ Model choice       │ Provider's list  │ যেকোনো open model   │
│ Best for           │ Startup, MVP,    │ High volume, BD     │
│                    │ low volume       │ data sovereignty,   │
│                    │                  │ specialized fine-tune│
└────────────────────┴──────────────────┴─────────────────────┘
```

### Self-host stack overview

| Tool | Use case |
|------|----------|
| **vLLM** | Production-grade — PagedAttention, continuous batching, OpenAI-compatible API; recommend high-throughput |
| **TensorRT-LLM** | NVIDIA-optimized, fastest for Hopper/Ampere |
| **llama.cpp** | CPU/GGUF quantized inference; edge deployment |
| **Ollama** | Developer-friendly local; built on llama.cpp |
| **TGI (HuggingFace Text Generation Inference)** | HF-native, production |
| **MLC LLM** | Cross-platform (mobile/web) |
| **SGLang** | Newer; structured generation |

### vLLM example (Linux + GPU)

```bash
# Install
pip install vllm

# Launch OpenAI-compatible server
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-70B-Instruct \
  --tensor-parallel-size 4 \
  --max-model-len 32768 \
  --quantization awq \
  --port 8000

# Then call from any OpenAI SDK with base_url="http://localhost:8000/v1"
```

```javascript
// Same OpenAI SDK, just point base URL
const client = new OpenAI({
  baseURL: "http://gpu-server.daraz-internal:8000/v1",
  apiKey: "not-needed-for-self-host",
});
```

---

## 🇧🇩 Bangla/Bengali Support — Practical Comparison

আমি (যাই agent লিখছে) practical experiment-এর summary সাজিয়েছি; numbers approximate, but pattern consistent।

```
Test prompt: "Daraz-এ কেন মানুষ বেশি অর্ডার দেয়? ৩টা কারণ লেখো বাংলায়।"

Model              Bangla quality    Code-switching tolerance    Verdict
─────────────────────────────────────────────────────────────────────────
GPT-4o             ⭐⭐⭐⭐           ⭐⭐⭐⭐⭐                    Best balance
GPT-4o mini        ⭐⭐⭐             ⭐⭐⭐⭐                      Cost-effective
Claude 3.5 Sonnet  ⭐⭐⭐⭐⭐         ⭐⭐⭐⭐                      Most natural Bangla
Claude Haiku       ⭐⭐⭐⭐           ⭐⭐⭐⭐                      Cheap + good
Gemini 1.5 Pro     ⭐⭐⭐⭐           ⭐⭐⭐                        Strong, sometimes formal
Gemini Flash       ⭐⭐⭐             ⭐⭐⭐                        Decent
Llama 3.1 70B      ⭐⭐⭐             ⭐⭐⭐                        Open, sometimes errors
Llama 3.1 8B       ⭐⭐               ⭐⭐                          Avoid Bangla-only
Mistral Large 2    ⭐⭐⭐             ⭐⭐⭐                        Better with English mix
Qwen 2.5 72B       ⭐⭐⭐             ⭐⭐⭐                        OK; better at code
Aya 23/Expanse 32B ⭐⭐⭐⭐           ⭐⭐⭐⭐                      Best open multilingual
DeepSeek V3        ⭐⭐               ⭐⭐⭐                        English-Chinese centric
```

**Caveats:**
- Banglish ("amar order kobe pabo") handle করায় GPT-4o, Claude সবচেয়ে ভালো
- Pure Bangla long text-এ Claude সবচেয়ে natural ও grammatically correct
- Open-source-এ Aya specifically multilingual purpose-এ trained — best Bangla among open

---

## 🎯 BD Context: Daraz Support Agent — Mini Implementation

```javascript
// daraz-support.js
import OpenAI from "openai";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const SYSTEM_PROMPT = `
তুমি Daraz Bangladesh-এর গ্রাহক সেবা agent। তোমার নাম "দারাজ সহায়ক"।

নীতি:
- সবসময় বাংলায় উত্তর দাও যদি user বাংলায় বা banglish-এ লেখে।
- English-এ লিখলে English-এ উত্তর দাও।
- সংক্ষিপ্ত ও বন্ধুসুলভ ভাষায় কথা বলো।
- যদি user-এর order সংক্রান্ত প্রশ্ন হয় কিন্তু order ID নেই, polite-ভাবে চাও।
- কখনোই প্রতিশ্রুতি দিও না — "আমি যাচাই করছি" বলো।
- Refund/cancellation policy সরাসরি না বলে human agent-এ escalate করো যদি unsure থাকো।

format:
- প্রথমে empathetic acknowledgment
- তারপর actionable response
- শেষে: "আরো কিছু জানতে চান?"
`;

async function chat(userMessage, history = []) {
  const messages = [
    { role: "system", content: SYSTEM_PROMPT },
    ...history,
    { role: "user", content: userMessage },
  ];

  const stream = await client.chat.completions.create({
    model: "gpt-4o-mini",        // cost-effective choice for support
    messages,
    temperature: 0.3,            // factual, not creative
    max_tokens: 400,
    presence_penalty: 0.3,       // diverse phrasing
    stream: true,
    stream_options: { include_usage: true },
  });

  let response = "";
  let usage = null;
  for await (const chunk of stream) {
    response += chunk.choices[0]?.delta?.content || "";
    if (chunk.usage) usage = chunk.usage;
  }

  // Cost tracking (per-conversation)
  const cost = (usage.prompt_tokens / 1e6) * 0.15 +
               (usage.completion_tokens / 1e6) * 0.60;
  console.log(`Conversation cost: $${cost.toFixed(6)}`);

  return { response, usage };
}

// Usage
const { response } = await chat("amar order #DZ12345 ekhono ase nai keno?");
console.log(response);
```

---

## ✅ কখন কোন parameter ব্যবহার করবেন (cheat-sheet)

| Use case | model | temp | top_p | max_tok | Notes |
|----------|-------|------|-------|---------|-------|
| Code generation | GPT-4o / Claude | 0.0-0.2 | 1.0 | 2048 | Deterministic |
| RAG Q&A | mini/flash/haiku | 0.0-0.3 | 1.0 | 512 | Factual, low temp |
| Classification | mini | 0.0 | 1.0 | 50 | + logit_bias |
| Creative writing | sonnet/4o | 0.8-1.0 | 0.95 | 1500 | বাংলা creative |
| Brainstorm | 4o | 1.0-1.3 | 0.95 | 1024 | Diverse ideas |
| Summarization | flash/mini | 0.2-0.4 | 1.0 | 300 | Faithful |
| Bangla translation | sonnet/4o | 0.1-0.3 | 1.0 | 1024 | Quality matter |
| Function calling | 4o/sonnet | 0.0 | 1.0 | 256 | JSON nodes |

---

## 🚫 Common Pitfalls

1. **`max_tokens` খুব ছোট** → mid-sentence cut। `finish_reason="length"` check করুন।
2. **Streaming-এ error handling ভুলে যাওয়া** — connection drop, partial content, retry logic দরকার।
3. **Token count না জানা** — input_tokens estimate না করে blindly long history পাঠালে context overflow।
4. **Bangla-তে wrong tokenizer ব্যবহার** — Llama 2 দিয়ে Bangla চালালে token bloat ৩x।
5. **Temperature + top_p দুটো একসাথে চাপা** — undefined behavior, OpenAI doc-এ ban।
6. **`stop` token system prompt-এর মধ্যে accidentally রাখা** → empty response।
7. **Self-host করার আগে utilization হিসাব না করা** — GPU idle থাকলে hosted-এর চেয়ে দামি।
8. **System prompt-এ secret/PII রাখা** — provider log-এ যেতে পারে।
9. **`o1`/`o3` (reasoning model) gas করে temperature/top_p set করা** — ignored।
10. **Streaming response-এ JSON parse করা প্রতি chunk-এ** — incomplete JSON crash; full text accumulate then parse।

---

## 🧰 Senior Engineer Checklist

- [ ] Token cost estimation script আছে (input + output, per-tenant)
- [ ] Tokenizer aware logging (Bangla-এ token explode হলে alert)
- [ ] Streaming endpoint backed by SSE/WebSocket, not polling
- [ ] Model fallback chain configured (4o → 4o-mini → cached)
- [ ] `finish_reason` check করা হয় সব response-এ
- [ ] Bangla evaluation set আছে (unit test for translation/summarization)
- [ ] KV cache memory monitor (self-host-এ)
- [ ] Latency p50/p95/p99 tracked (TTFT, TPS আলাদাভাবে)
- [ ] Per-tenant rate limit + budget cap
- [ ] Document model version pinning (e.g., `gpt-4o-2024-11-20`) — silent upgrades issue করে

---

## 📚 আরো পড়ুন

- "Attention is All You Need" — Vaswani et al, 2017
- "The Illustrated Transformer" — Jay Alammar (visual gold)
- "GPT-2 paper / GPT-3 paper" — OpenAI scaling law
- "InstructGPT" — RLHF foundation
- "Direct Preference Optimization" — Stanford, 2023
- HuggingFace `transformers` source code পড়ুন
- vLLM paper — PagedAttention, 2023

→ পরবর্তী: [Prompt Engineering](./prompt-engineering.md) — model কে কী জিজ্ঞেস করব?
