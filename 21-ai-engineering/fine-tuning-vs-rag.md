# 🎓 Fine-tuning vs RAG (এবং সব Adaptation Techniques)

> "আমার domain-এ LLM ভালো করছে না — fine-tune করব?"
> ৮০% ক্ষেত্রে উত্তর: **না, RAG বা prompt engineering করো।**
> বাকি ২০%—যেমন Bangla tone, structured output format—এ fine-tuning জিতে।
> এই অধ্যায়: কখন কোনটা, কীভাবে, কত খরচ।

---

## 📖 সূচিপত্র

- [Decision Framework](#-decision-framework)
- [Adaptation Spectrum](#-adaptation-spectrum)
- [Fine-tuning Methods](#-fine-tuning-methods)
- [Datasets](#-datasets-instruction--preference)
- [Tooling](#-tooling)
- [GPU Memory Math](#-gpu-memory-math)
- [Evaluation Pre/Post](#-evaluation-prepost-fine-tune)
- [Catastrophic Forgetting](#-catastrophic-forgetting)
- [Combining: Fine-tune + RAG](#-combining-fine-tune--rag)
- [Distillation](#-distillation)
- [Quantization](#-quantization)
- [Bangla-specific Considerations](#-bangla-specific-considerations)
- [Decision Matrix](#-decision-matrix)
- [JS/PHP Integration Post Fine-tune](#-jsphp-integration-post-fine-tune)
- [BD Example: bKash Chatbot](#-bd-example-bkash-chatbot)
- [Pitfalls](#-pitfalls)
- [Checklist](#-checklist)

---

## 🎯 Decision Framework

```
Start here:
  ↓
1. Did prompt engineering try?
   ├─ NO → try first (cheapest, fastest)
   └─ YES, still failing
      ↓
2. Is the issue knowledge (facts)?
   ├─ YES → RAG ⭐
   └─ NO
      ↓
3. Is the issue style/format/language tone?
   ├─ YES → Fine-tuning candidate
   └─ NO
      ↓
4. Is the issue reasoning/logic?
   ├─ YES → Better model (GPT-4 → o1) or CoT prompting
   └─ NO
      ↓
5. Is the issue cost/latency?
   ├─ YES → Distill to smaller model, then fine-tune
   └─ NO → Re-examine; LLM may be wrong choice
```

### Quick Heuristic

```
"আমার model X জানে না"           → RAG (knowledge gap)
"আমার model ভুল format দেয়"      → Fine-tune
"আমার model Bangla তে formal না" → Fine-tune
"আমার model latest data নেয় না"  → RAG
"আমার model বড়, ধীর, ব্যয়বহুল"   → Distill + Fine-tune
"আমার model কখনো-কখনো ভুল করে"   → Eval first, then RAG/FT
```

---

## 🌈 Adaptation Spectrum

```
Cost ↑                                     Capability ↑
                                                    │
  ┌────────────────┐                                │
  │ Prompt eng.    │ Free, instant, low control     │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Few-shot       │ +context, better quality       │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ RAG            │ +knowledge, citations, dynamic │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Tool/Agent     │ +actions, side effects         │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Fine-tune (PEFT)│ +style, format, lang          │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Full fine-tune │ +deep behavior change          │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Continued      │ New domain knowledge baked     │
  │ pre-training   │ (very expensive)               │
  └────────────────┘                                │
  ┌────────────────┐                                │
  │ Train from     │ Maximum control, max cost      │
  │ scratch        │ (only big labs)                ▼
  └────────────────┘
```

---

## 🛠️ Fine-tuning Methods

### Full Fine-tuning

```
সব weight update করা হয়।
Memory: 70B model FP16 = 140 GB just for weights,
        + gradients (140 GB) + optimizer state (560 GB) = 840 GB!
GPU:    8× A100 80GB minimum
Cost:   $5K-$50K per run
Use:    rare. Big behavior change, foundation model creation.
```

### LoRA (Low-Rank Adaptation) ⭐ most popular

```
Idea: weight update ΔW = A × B, where rank(A,B) << rank(W)

  W (frozen)          A · B (trainable, tiny!)
  [d × d]             [d × r] · [r × d],  r=8-64

Train only A, B. Original W frozen.
Result: 0.1-1% of full params trained.

Memory savings:
  70B FP16: 140 GB → trainable params ~1 GB!
  Single A100 80GB can fit (with QLoRA below).

Inference:
  Either: keep base + LoRA (swap LoRAs at runtime!)
  Or:     merge ΔW into W (no overhead, but lose modularity)
```

### QLoRA (Quantized LoRA)

```
1. Base model quantized to NF4 (4-bit)
2. LoRA adapter trained on top in BF16
3. 70B fits on single 48-GB GPU!

Quality: very close to full fine-tune, 1% loss typical
Cost:    $20-200 instead of $5K
Tooling: Unsloth (best speed), HF PEFT
```

### DoRA (Weight-Decomposed LoRA)

```
ΔW decomposed into magnitude + direction
Slightly better quality than LoRA at same param budget
Newer, less battle-tested
```

### Prefix Tuning / P-Tuning

```
Train soft prompts (continuous embeddings) prepended to input
Tiny params, but harder to optimize, less popular than LoRA
```

### Prompt Tuning

```
Same idea, even fewer params (just embedding for "prompt tokens")
Diminished returns vs LoRA
```

### RLHF / DPO (Preference-based)

```
RLHF (PPO):
  1. SFT (supervised fine-tune) on instructions
  2. Reward model trained on human preferences (A vs B)
  3. RL loop: model generates → reward score → PPO update
  Complex, unstable, expensive

DPO (Direct Preference Optimization):
  - Skip reward model, optimize directly on preferences
  - Stable, simple, cheaper
  - Default for "make my model align with my brand voice"

KTO, IPO, SimPO: newer variants, marginal gains

ORPO: combines SFT + DPO in one stage
```

---

## 📚 Datasets: Instruction & Preference

### Instruction Format (Alpaca / ChatML)

```jsonl
{"messages":[
  {"role":"system","content":"তুমি bKash এর support assistant।"},
  {"role":"user","content":"OTP আসছে না কেন?"},
  {"role":"assistant","content":"OTP না আসার কয়েকটি কারণ থাকতে পারে: ..."}
]}
{"messages":[ ... ]}
```

### Preference Format (DPO)

```jsonl
{
  "prompt": "OTP আসছে না কেন?",
  "chosen": "OTP না আসার কারণ: ১) নেটওয়ার্ক সমস্যা ২) ...",
  "rejected": "I don't know, please contact support."
}
```

### Quantity Heuristics

```
Style/format change:        50-500 examples
Domain language (Bangla):   1K-10K examples
New task (classification):  500-5K examples
New knowledge:              ❌ don't fine-tune for this — use RAG
RLHF/DPO alignment:         5K-50K preference pairs
```

### Dataset Quality > Quantity

```
✅ 200 hand-curated, perfect examples
   beats
❌ 50K noisy, AI-generated low-effort examples

Quality checks:
- Diverse intents
- Edge cases included
- Output format consistent
- No hallucination in training data!
- Bangla examples: native speaker reviewed
```

### Synthesis Tools

```
- Distilabel:    LLM-driven dataset gen
- Argilla:       human-in-loop labeling
- Self-Instruct: bootstrap from seed examples
- Use GPT-4 to generate, then filter/dedupe
- License-aware: don't train on OpenAI outputs to compete with OpenAI
  (Terms forbid; legally messy)
```

---

## 🧰 Tooling

| Tool                       | Lang   | Method          | Strength                          |
| -------------------------- | ------ | --------------- | --------------------------------- |
| **HF Transformers + PEFT** | Python | LoRA, QLoRA, FT | Standard, max flexibility         |
| **Unsloth**                | Python | QLoRA mainly    | 2-5× faster, single GPU friendly ⭐|
| **Axolotl**                | Python | All             | Config-driven, prod-ready         |
| **TRL** (HF)               | Python | SFT, DPO, RLHF  | Preference training               |
| **OpenAI Fine-tuning**     | API    | Proprietary     | Easiest, GPT-3.5/4o-mini          |
| **Together AI**            | API    | LoRA, full      | Wide model selection              |
| **Modal / Replicate**      | API    | Any             | Pay-per-second GPU                |
| **MosaicML / Databricks**  | API    | Full FT         | Enterprise scale                  |
| **AWS SageMaker**          | API    | All             | AWS shop                          |
| **Llama Factory**          | Python | All             | GUI option                        |

### When to use API vs DIY

```
OpenAI/Together fine-tuning:
  ✅ Easy, no infra
  ✅ Hosted inference included
  ❌ Vendor lock-in, model choice limited, no quantization

Self-hosted (Unsloth/Axolotl):
  ✅ Any open model (Llama, Mistral, Qwen, Gemma)
  ✅ Quantization, custom LoRA merge
  ✅ Run on-prem (BD compliance)
  ❌ DevOps, GPU procurement
```

---

## 💾 GPU Memory Math

### Cheat Sheet

```
Model size in bytes (FP16):  N_params × 2
Weights:                      = M
Gradients (full FT):          = M
Optimizer (AdamW):            = 2-4× M (momentum + variance, often FP32)
Activations:                  varies, ~ batch × seq × hidden

Total full FT:    ≈ 4-8 × M   (e.g., 7B → ~30-50 GB)
LoRA (BF16):      ≈ M (frozen) + tiny adapter, but bf16 fits
QLoRA (4-bit):    ≈ M / 4 + small adapter
Inference (FP16): ≈ M
Inference (4-bit):≈ M / 4
```

### Examples

```
Llama-3-70B (70B params):
  FP32: 280 GB
  FP16: 140 GB
  INT8:  70 GB
  INT4:  35 GB    ← single H100 80GB inference

Full FT 70B:       ~600 GB → 8× A100 80GB ($$$$)
LoRA 70B (BF16):   ~140 GB → 2× A100 80GB
QLoRA 70B (4-bit): ~48 GB  → single A100 80GB ⭐ or 2× 4090

Llama-3-8B:
  Full FT:          ~50 GB  → 1× A100 80GB
  QLoRA:            ~10 GB  → single 3090/4090 ($1500 GPU)

Mistral-7B:
  QLoRA: fits on 16GB GPU (RTX 4060 Ti 16GB)
  → can fine-tune on a desk in Dhaka 😊
```

### Bangladesh Context

```
GPU options:
- Cloud: Vast.ai, RunPod, Lambda — pay per hour
  RTX 4090: $0.40/hr, A100 80GB: $1.50/hr
- Local: a 4090 from Star Tech ৳3.5L (~$3K)
- Ata Tech / Smart Technologies sometimes have A100 boxes
- Big banks/telcos invest in on-prem; SMB → cloud preferred
```

---

## 📊 Evaluation Pre/Post Fine-tune

### Required Datasets

```
1. Train set (used for FT)
2. Validation set (used during FT for early stopping)
3. Held-out test set (NEVER seen during FT, used for final eval)
4. Regression set: previous tasks the model should still do
```

### Metrics

```
Task-specific (classification, NER):
  - Accuracy, F1, precision/recall
  - Confusion matrix per class

Generation:
  - BLEU, ROUGE (limited utility)
  - BERTScore
  - LLM-as-judge (GPT-4 scores 1-5)
  - Human eval (gold standard)

Calibration:
  - ECE (expected calibration error)
  - Refusal rate (correct rejection of unanswerable)

Safety:
  - Toxic output rate
  - Hallucination rate
  - Jailbreak resistance
```

### Eval Harness

```
- lm-eval-harness (EleutherAI): standard benchmarks
- HELM (Stanford): holistic
- MT-Bench: multi-turn
- Custom: your domain-specific suite ⭐ most important

Bangla benchmarks:
- IndicBenchmarks (incl. Bangla)
- BanglaBERT eval suite
- Custom: 200 Bangla support tickets, scored
```

### A/B Test in Production

```
- Shadow traffic: 1% to FT model, compare quality
- Canary: 5% → 25% → 100%
- Track: CSAT, containment, escalation rate, latency, cost
- Roll back if quality regresses
```

---

## 🧠 Catastrophic Forgetting

```
Problem:
  Fine-tune Llama-3 on Bangla support tickets
  → forgets it can do English coding 😱

Causes:
  - Too many epochs
  - Narrow dataset
  - Full FT (LoRA mitigates by freezing base)

Mitigations:
  1. Use LoRA/QLoRA (preserves base behaviour)
  2. Mix in general data (10-20% of original distribution)
  3. Lower learning rate (e-5 not e-4)
  4. Fewer epochs (1-3, not 10)
  5. Regularization: KL divergence to base model
  6. Continual learning techniques (EWC, replay buffer)

Test: include regression set in eval (general benchmarks).
```

---

## 🤝 Combining: Fine-tune + RAG

```
The pro pattern (used by every serious BD AI team):

  Fine-tune for:           RAG for:
  - Tone (Bangla formal)   - Up-to-date facts
  - Format (JSON output)   - Long-tail knowledge
  - Domain vocabulary      - Citations
  - Refusal behaviour      - Multi-tenant docs
  - Bangla nuance          - Compliance updates

Architecture:
  User → query → retrieve docs (RAG)
              → prompt with retrieved context
              → fine-tuned LLM generates answer in your voice
              → response with citations

bKash example:
  - Fine-tune: Bangla customer service tone, refusal patterns,
              "Sir/Madam সম্বোধন," structured output
  - RAG: latest fee tables, policy updates, FAQ
```

### Order of operations

```
1. Start with prompt engineering + RAG (no FT)
2. Measure quality on golden set
3. If consistent style/format issues → FT
4. After FT, re-test RAG behavior (formats may change)
5. Continuous eval, both layers
```

---

## 🎓 Distillation

```
Idea: train small student model on big teacher model's outputs

Steps:
1. Big LLM (GPT-4) labels 100K of your inputs with answers
2. Small model (Mistral-7B) fine-tuned on (input, GPT-4 output) pairs
3. Small model approximates GPT-4 on your domain at 1/100 cost

Variants:
- Black-box: only teacher API access (output supervised)
- White-box: access teacher logits/internals (more efficient)
- Self-distillation: bigger ckpt → smaller ckpt of same model

When to use:
  ✅ Production needs latency < 200ms but quality > GPT-3.5
  ✅ Cost reduction at scale
  ✅ Privacy: keep model on-prem
  ❌ Teacher's TOS forbids training competitors (OpenAI! be careful)
```

### License Watch

```
OpenAI ToS: "use Output to develop models that compete with OpenAI" → forbidden
Workarounds:
- Use Llama / Mixtral / Qwen as teacher (Apache/license-friendly)
- Generate synthetic data from your data, not theirs
- Read latest ToS — changes
```

---

## 🗜️ Quantization

```
Reduce precision to fit smaller GPUs:

FP32 (32-bit) → FP16 (16-bit) → BF16 → INT8 → INT4 → INT3 → INT2

Quality loss usually negligible until INT4, more visible at INT3.
Llama 3 70B at INT4: ~99% of FP16 quality on benchmarks.
```

### Methods

| Method         | Speed | Quality | Notes                            |
| -------------- | ----- | ------- | -------------------------------- |
| **GPTQ**       | medium | great  | Calibration-based, post-train    |
| **AWQ**        | fast   | great  | Activation-aware, mobile-friendly|
| **GGUF**       | fast   | good   | llama.cpp, CPU-friendly ⭐ for BD |
| **FP8**        | fast   | great  | H100 native, training too        |
| **bitsandbytes** | medium | ok    | Easy in-process for QLoRA        |
| **EXL2**       | fast   | great  | ExLlamaV2, GPU-only              |

### GGUF + llama.cpp

```bash
# Convert
python convert-hf-to-gguf.py llama-3-8b/ --outfile llama-3-8b.gguf
# Quantize
./llama-quantize llama-3-8b.gguf llama-3-8b-q4_k_m.gguf q4_k_m
# Run on CPU
./llama-cli -m llama-3-8b-q4_k_m.gguf -p "হ্যালো" -n 100
```

CPU inference for Mistral-7B Q4: ~10 tok/s on a decent laptop. Useful for **on-prem BD deployments without GPU**.

---

## 🇧🇩 Bangla-specific Considerations

### Challenges

```
1. Limited high-quality datasets:
   - English: trillions of tokens
   - Bangla:  much less, much noisier
   - Most public Bangla data is news / Wikipedia / OSCAR — stilted

2. Code-mixed text (Banglish):
   "আমার OTP আসছে না, please help"
   Tokenizer may split poorly; model may slip into English

3. Tokenizer issues:
   - Llama-3 tokenizer: Bangla token efficiency okay
   - GPT-4o tokenizer: better
   - Older models (Llama-2): 1 Bangla word = 5-8 tokens! (2-3× cost)

4. Diacritics and Unicode:
   - Same word can be encoded differently (ZWJ, NFC vs NFD)
   - Always normalize before tokenize

5. Honorifics, formality:
   - "আপনি" vs "তুমি" vs "তুই"
   - bKash chatbot must use "আপনি"
   - Hard to express in prompt alone → fine-tune wins

6. Number formats:
   - Bangla numerals "১২৩" vs Arabic "123"
   - Both used; normalize for parsing
```

### Strategies

```
Tokenizer-aware models:
- Use Llama-3+, Qwen2, Aya (Cohere multilingual)
- BUET csebuetnlp/banglat5 for Bangla-only

Pretraining augmentation:
- Continued pre-train Llama on 5-50B Bangla tokens
- Big lift, big cost ($10K+) — only for large players

PEFT/LoRA on existing multilingual:
- Most cost-effective ⭐
- Aya-23-8B + LoRA on 5K Bangla support pairs

Synthetic data:
- GPT-4o generates Bangla examples (high quality)
- Native speaker reviews 10% → flags errors
- Iterative

Evaluation must be Bangla:
- English benchmarks meaningless for tone
- Build 200-500 example Bangla golden set
- Native reviewer scores
```

### Dataset sources

```
- bn_wiki dump
- OSCAR (Common Crawl filtered)
- BUET csebuetnlp curated corpora
- Prothom Alo / DBL News scrapes (license check!)
- Your own product data (most valuable)
- Synthetic via GPT-4o (filter heavily)
```

---

## 📊 Decision Matrix

| Need                              | Best fit                       | Why                       |
| --------------------------------- | ------------------------------ | ------------------------- |
| Latest news/policy updates        | RAG                            | Knowledge changes daily   |
| "Use formal Bangla, sir/madam"    | Fine-tune (LoRA)               | Style baked in            |
| Strict JSON output schema         | Fine-tune OR JSON mode + prompt| Format consistency        |
| New language (Bangla) capability  | Continued pre-train + FT       | Vocabulary acquisition    |
| Cheaper inference of GPT-4 quality| Distill to 7B, FT              | Cost optimization         |
| Domain jargon (medical/legal)     | RAG + FT                       | Terms + facts both        |
| Reasoning improvement             | Better model (o1) or CoT       | RL-trained reasoners      |
| Multi-tenant per-customer voice   | LoRA per tenant + base         | Adapter swap at runtime   |
| Privacy/on-prem                   | OS model + QLoRA + GGUF        | No external API           |
| <100 examples available           | Few-shot in prompt             | Too few for FT            |
| 1K-10K examples                   | LoRA / QLoRA                   | Sweet spot                |
| 100K+ examples                    | Full FT or large LoRA          | Justify the cost          |

---

## 💻 JS/PHP Integration Post Fine-tune

### Hosted on Together AI

```javascript
import Together from 'together-ai';
const together = new Together();

const r = await together.chat.completions.create({
  model: 'your-org/bkash-bangla-ft',
  messages: [
    { role: 'system', content: 'তুমি bKash সাপোর্ট।' },
    { role: 'user',   content: 'OTP আসছে না।' },
  ],
});
console.log(r.choices[0].message.content);
```

### Self-hosted with vLLM (LoRA hot-swap)

```bash
# Serve base + multiple LoRAs
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --enable-lora \
  --lora-modules bkash=./loras/bkash pathao=./loras/pathao \
  --port 8000
```

```javascript
// OpenAI-compatible client → just change baseURL
import OpenAI from 'openai';
const client = new OpenAI({
  baseURL: 'http://vllm.internal:8000/v1',
  apiKey: 'sk-local',
});

const r = await client.chat.completions.create({
  model: 'bkash',  // selects the bkash LoRA
  messages: [...],
});
```

### Self-hosted with TGI (Text Generation Inference)

```bash
docker run --gpus all -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-generation-inference:latest \
  --model-id ./bkash-ft-merged --quantize awq
```

### Self-hosted CPU (llama.cpp + GGUF)

```bash
./llama-server -m bkash-ft-q4_k_m.gguf -c 4096 --port 8080
```

```javascript
// llama.cpp server is OpenAI-compatible
const r = await fetch('http://localhost:8080/v1/chat/completions', {
  method: 'POST',
  body: JSON.stringify({ messages: [...] }),
});
```

### PHP Integration

```php
<?php
// Same: use any OpenAI-compatible client
$client = OpenAI::factory()
    ->withBaseUri('http://vllm.internal:8000/v1')
    ->withApiKey('sk-local')
    ->make();

$response = $client->chat()->create([
    'model'    => 'bkash',
    'messages' => [
        ['role' => 'system', 'content' => 'তুমি bKash এর সাপোর্ট।'],
        ['role' => 'user',   'content' => $req->input('q')],
    ],
    'temperature' => 0.3,
    'max_tokens'  => 400,
]);

return response()->json([
    'answer' => $response->choices[0]->message->content,
]);
```

### Switching LoRA per Tenant

```php
$tenantToLora = [
    'bkash'   => 'bkash',
    'pathao'  => 'pathao',
    'daraz'   => 'daraz',
];
$model = $tenantToLora[$tenantId] ?? 'base';
$response = $client->chat()->create(['model' => $model, /* ... */]);
```

---

## 🇧🇩 BD Example: bKash Chatbot

### Goal

Bangla-first customer support chatbot:
- Formal tone ("আপনি", "জনাব")
- Knows latest fee structure (changes monthly)
- Citations to policy doc
- Refuses non-bKash topics
- < 1.5s P95 latency
- Cost-conscious

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│  bKash Chatbot Stack                                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  RAG Layer:                                              │
│   - Policy docs, FAQ → Qdrant + bge-m3 embeddings        │
│   - Hybrid (vector + BM25) + Cohere multilingual rerank  │
│                                                          │
│  Fine-tuned Model:                                       │
│   - Base: Aya-23-8B (multilingual, Bangla-strong)        │
│   - LoRA on 4K curated Bangla support Q&A pairs          │
│   - DPO on 1K (preferred vs rejected) for tone           │
│   - Hosted on vLLM (2× A100 40GB)                        │
│                                                          │
│  Orchestration:                                          │
│   - Node.js + LangGraph                                  │
│   - Retrieves → assembles prompt → FT model              │
│                                                          │
│  Tools (when needed):                                    │
│   - lookup_transaction (DB)                              │
│   - get_balance (API)                                    │
│   - escalate_to_human                                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Training Pipeline

```
1. Data collection:
   - 4 months of anonymized support transcripts (~10K)
   - Filter: clear question, helpful answer
   - Clean: remove PII (regex + LLM check)
   - Dedupe near-duplicates (embedding cluster)

2. Curate to ~4K high-quality examples:
   - Native speaker review of 10% sample
   - Reject inconsistent tone, factual errors
   - Mix: account, payment, OTP, fees, fraud, refund

3. SFT on Aya-23-8B with QLoRA:
   - r=32, alpha=64, dropout=0.05
   - 3 epochs, lr=2e-4 (cosine)
   - Batch 4 × grad-accum 8
   - Time: ~6 hours on 1× A100 80GB
   - Cost: ~$10

4. Build preference dataset:
   - 1K examples with 2 candidate answers each
   - Native reviewer picks better one
   - Often: chosen has formal "আপনি", rejected uses "তুমি" or English fallback

5. DPO from SFT checkpoint:
   - 2 epochs, lr=5e-7 (very low for stability)
   - Time: ~3 hours
   - Cost: ~$5

6. Evaluation:
   - 200 held-out Bangla queries
   - LLM-as-judge (GPT-4o) + 50 human-scored
   - Metrics: tone score, factuality (with RAG), refusal correctness

7. Deploy:
   - Merge LoRA OR keep adapter for hot-swap
   - vLLM serve, OpenAI-compatible
   - Canary 5% → ramp
```

### Numbers

```
Training cost:        ~$30 (QLoRA + DPO)
Eval cost:            ~$50 (GPT-4o-as-judge)
Inference cost:       $0.0003 per query (self-hosted vLLM)
Latency P95:          1.2s end-to-end (RAG + LLM)
Tone score (1-5):     4.6 vs base 3.2
Factuality with RAG:  92% vs base+RAG 88%
Refusal correctness:  94% vs base 67% (FT taught it boundaries)
Containment:          74%, ↑18% over base+RAG
```

### Lessons

1. **DPO catches subtle tone** SFT alone misses
2. **PII redaction in training data** is non-negotiable (Bangladesh DPA + brand)
3. **Mixed Bangla-English pairs** (~30%) prevented English regression
4. **Citation prompt** preserved during FT (don't FT-out the citation behavior)
5. **A/B test mandatory** — first FT actually regressed factuality, fixed via more diverse data
6. **Periodic re-train** (quarterly) as policy changes
7. **Base model selection mattered**: Aya-23 > Llama-3 for Bangla out-of-box

---

## ⚠️ Pitfalls

```
1. ❌ Fine-tuning to "add knowledge"
   Model regurgitates training facts but can't reason about them
   Fix: Use RAG. FT does NOT reliably add facts.

2. ❌ Too few examples
   <100 → noise, overfit
   Fix: more data or use few-shot prompting

3. ❌ Too many epochs → overfit + forgetting
   Fix: 1-3 epochs, watch eval loss, early stop

4. ❌ Training data with hallucinations
   Model learns to hallucinate confidently 😱
   Fix: rigorous review of training data

5. ❌ No held-out eval
   "Train loss is great!" → useless without test set

6. ❌ Forgetting base capabilities
   FT model can't do English or coding anymore
   Fix: regression test, mix in general data

7. ❌ Fine-tuning when prompt would work
   Wasted $$$ + maintenance
   Fix: prompt eng. baseline first

8. ❌ Picking wrong base model
   Base bad at Bangla → no amount of FT helps much
   Fix: try Aya, Qwen, Gemma, Llama, pick winner pre-FT

9. ❌ Not versioning models
   "What's in production?" — nobody knows
   Fix: model registry, hash, eval report linked

10. ❌ Ignoring license
    Llama 3 has commercial restrictions ($X revenue)
    Mistral, Qwen, Gemma have varying terms
    Fix: legal review

11. ❌ Quantizing then fine-tuning naively
    QLoRA carefully designed; arbitrary stack-up breaks
    Fix: use Unsloth / Axolotl recipes

12. ❌ Forgetting tokenizer match
    Train with tokenizer A, deploy with B → garbage
    Fix: always pair tokenizer with weights
```

---

## 📋 Checklist

```
Decision:
  ☐ Tried prompt eng. + RAG first
  ☐ Documented why FT is needed (style/format/tone)
  ☐ Defined success metrics + golden eval set
  ☐ Budget + GPU access secured
  ☐ License of base model checked

Data:
  ☐ ≥ 500 quality examples (or rationale for less)
  ☐ PII redacted
  ☐ Native speaker review (Bangla case)
  ☐ Train/val/test split (80/10/10)
  ☐ Regression set defined

Training:
  ☐ Method picked (LoRA / QLoRA / DPO / full FT)
  ☐ Hyper-params: lr, epochs, r, alpha tuned via val
  ☐ Reproducible: seed, config in repo
  ☐ Logging: W&B / MLflow

Evaluation:
  ☐ Task metrics
  ☐ Regression metrics (general benchmarks)
  ☐ Tone / format human review
  ☐ Safety: jailbreak, toxicity
  ☐ Latency, throughput

Deployment:
  ☐ Versioned (semantic + git hash)
  ☐ Canary → ramp
  ☐ Monitoring: quality (LLM judge), CSAT, cost
  ☐ Rollback plan
  ☐ Periodic retrain schedule

Combine:
  ☐ FT-after-RAG verified (RAG quality unchanged)
  ☐ FT model still respects citation/refusal rules
```

---

## 🎯 সারমর্ম

```
Order of attempts:
  1. Prompt engineering
  2. Few-shot examples
  3. RAG
  4. Tools / agents
  5. Fine-tuning (LoRA/QLoRA)
  6. RLHF / DPO (alignment)
  7. Continued pre-training (rare)
  8. From scratch (don't)

Combine: Fine-tune for STYLE, RAG for FACTS.

Bangladesh:
  - QLoRA on Aya-23 / Qwen2 / Llama-3 = $20-100 affordable
  - Self-host on RunPod / Vast / on-prem 4090
  - GGUF for CPU inference (no GPU on-prem)
  - Always evaluate in Bangla — English benchmarks lie
  - Periodic retrain as policies and language evolve
```

---

## 🔗 আরও পড়ুন

- [LLM Fundamentals](./llm-fundamentals.md)
- [Prompt Engineering](./prompt-engineering.md)
- [RAG Systems](./rag-systems.md) — your first stop before FT
- [Embeddings](./embeddings.md)
- [LLM Ops](./llm-ops.md) — deploying & monitoring FT models
- [LLM Agents & Tools](./llm-agents-tools.md)
