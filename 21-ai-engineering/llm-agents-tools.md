# 🤖 LLM Agents & Tools

> **Agent** = LLM যেটা শুধু text generate করে না, বরং **নিজেই decide করে কোন tool call করবে, ফলাফল দেখে পরের step ঠিক করবে**, এভাবে একটা goal complete করে।
> Foodpanda menu chatbot যে restaurant search + order status + payment retry সব করে, Pathao Foods order-status agent যে DB থেকে latest status আনে — এসব agent।

---

## 📖 সূচিপত্র

- [Agent কী এবং কেন](#-agent-কী-এবং-কেন)
- [ReAct Loop](#-react-loop)
- [Tool/Function Calling Spec](#-toolfunction-calling-spec)
- [Tool Registry & Schema Validation](#-tool-registry--schema-validation)
- [Multi-step & Planning Agents](#-multi-step--planning-agents)
- [Multi-agent Systems](#-multi-agent-systems)
- [Memory: Short, Long, Episodic](#-memory-short-long-episodic)
- [LangGraph Deep Dive](#-langgraph-deep-dive)
- [Computer/Browser Use](#-computer--browser-use-agents)
- [Code Interpreter / Sandbox](#-code-interpreter--sandbox)
- [Safety & Sandboxing](#-safety--sandboxing)
- [Cost & Loop Runaway](#-cost--loop-runaway-protection)
- [কোড: LangGraph Foodpanda Agent](#-কোড-langgraph-foodpanda-agent)
- [PHP Tool Gateway](#-php-tool-gateway)
- [BD: Pathao Foods Order-Status Agent](#-bd-pathao-foods-order-status-agent)
- [Pitfalls](#-common-pitfalls)
- [Checklist](#-production-checklist)

---

## 🎯 Agent কী এবং কেন

```
Plain LLM:
─────────
User: "আমার শেষ Foodpanda order এর status কি?"
LLM: "আমি real-time data দেখতে পারি না..."  ❌

Agent (with tools):
──────────────────
User: "আমার শেষ Foodpanda order এর status কি?"
LLM thinks: "তথ্য দরকার DB থেকে"
LLM calls:  get_user_orders(user_id=123, limit=1)
Tool returns: {order_id: "FP-789", status: "delivered", item: "Chicken biryani"}
LLM responds: "আপনার সর্বশেষ order FP-789 (Chicken biryani) delivered হয়েছে।"  ✅

Plain LLM = brain in a jar
Agent = brain + hands + eyes (tools)
```

### Capabilities Matrix

| Capability       | Prompt-only | RAG  | Agent      |
| ---------------- | ----------- | ---- | ---------- |
| Static knowledge | ✅          | ✅✅ | ✅✅       |
| Real-time data   | ❌          | ⚠️    | ✅✅       |
| Calculations     | ⚠️          | ⚠️    | ✅ (tool)  |
| Multi-step task  | ⚠️          | ⚠️    | ✅✅       |
| Side effects     | ❌          | ❌    | ✅✅ (write) |
| Cost             | $          | $$   | $$$        |
| Latency          | low         | mid  | high (loop) |

---

## 🔄 ReAct Loop

**ReAct** = **Re**asoning + **Act**ion. Yao et al. 2022.

```
   ┌────────────────────────────────────────┐
   │              ReAct Loop                │
   ├────────────────────────────────────────┤
   │                                        │
   │  Thought: "User wants order status,    │
   │            I should call get_orders."  │
   │            │                           │
   │            ▼                           │
   │  Action:   get_orders(user_id=123)     │
   │            │                           │
   │            ▼                           │
   │  Observation: {order: FP-789, status:  │
   │                "in_kitchen"}           │
   │            │                           │
   │            ▼                           │
   │  Thought: "Still cooking, I should     │
   │            tell user with ETA."        │
   │            │                           │
   │            ▼                           │
   │  Action:   final_answer("আপনার order   │
   │            রান্না হচ্ছে, আনুমানিক ১৫ মিনিট") │
   │                                        │
   └────────────────────────────────────────┘

Loop until: final answer | max steps reached | error
```

### Modern: Native Function Calling

OpenAI/Anthropic-এর native function calling আজকাল ReAct prompt-এর চেয়ে stable—LLM JSON-formatted tool call return করে, parse করা সহজ।

---

## 🛠️ Tool/Function Calling Spec

### OpenAI Format

```json
{
  "type": "function",
  "function": {
    "name": "get_orders",
    "description": "Fetch latest orders for a user",
    "parameters": {
      "type": "object",
      "properties": {
        "user_id": { "type": "integer" },
        "limit":   { "type": "integer", "default": 5 }
      },
      "required": ["user_id"]
    }
  }
}
```

### Anthropic Format

```json
{
  "name": "get_orders",
  "description": "...",
  "input_schema": {
    "type": "object",
    "properties": { "user_id": { "type": "integer" } },
    "required": ["user_id"]
  }
}
```

### Gemini Format

```json
{
  "function_declarations": [{
    "name": "get_orders",
    "description": "...",
    "parameters": { "type": "OBJECT", "properties": { /* ... */ } }
  }]
}
```

### Tool Choice Modes

```
auto:           model decides
required/any:   force at least one tool call
none:           prevent tool calls (force text)
specific:       force a particular tool
```

### Multi-tool Parallel Calls

```
GPT-4o, Claude 3.5+ can return multiple tool calls in one turn:

[
  { id: "c1", name: "get_user_address", args: {} },
  { id: "c2", name: "list_nearby_restaurants", args: {area: "?"} } // pending c1
]

Strategy: execute non-dependent calls in parallel.
Dependent ones (c2 depends on c1) → still sequential.
```

---

## 📜 Tool Registry & Schema Validation

```typescript
// tool-registry.ts (TypeScript + Zod)
import { z } from 'zod';

interface Tool<T extends z.ZodSchema> {
  name: string;
  description: string;
  schema: T;
  handler: (args: z.infer<T>, ctx: ToolContext) => Promise<unknown>;
  rateLimit?: { rps: number };
  permissions?: string[];   // e.g., ["user:read"]
  costEstimate?: number;    // for budget tracking
  timeout?: number;         // ms
}

export class ToolRegistry {
  private tools = new Map<string, Tool<any>>();

  register<T extends z.ZodSchema>(tool: Tool<T>) {
    this.tools.set(tool.name, tool);
  }

  toOpenAI() {
    return [...this.tools.values()].map(t => ({
      type: 'function' as const,
      function: {
        name: t.name,
        description: t.description,
        parameters: zodToJsonSchema(t.schema),
      },
    }));
  }

  async execute(name: string, rawArgs: unknown, ctx: ToolContext) {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Unknown tool: ${name}`);

    // 1. Permission check
    if (tool.permissions) ctx.requirePermissions(tool.permissions);

    // 2. Rate limit (Redis token bucket)
    if (tool.rateLimit) await ctx.checkRateLimit(name, tool.rateLimit);

    // 3. Validate args (security boundary!)
    const args = tool.schema.parse(rawArgs);

    // 4. Execute with timeout
    return await withTimeout(
      tool.handler(args, ctx),
      tool.timeout ?? 10_000,
      `Tool ${name} timeout`
    );
  }
}
```

### Why Validation is Critical

```
LLM might output:
  args: { user_id: "'; DROP TABLE users; --" }   😱

Without validation:
  → SQL injection if you just .execute(rawArgs.query)

With Zod:
  z.object({ user_id: z.number().int().positive() }).parse(rawArgs)
  → throws → tool returns error → LLM corrects
```

---

## 🧮 Multi-step & Planning Agents

### Plan-and-Execute

```
Step 1: PLANNER LLM
  Input: "User wants refund for last week's biryani order"
  Output: [
    "1. Find user's orders from last week",
    "2. Identify biryani order",
    "3. Check eligibility (delivered, < 14 days)",
    "4. Initiate refund via payment service",
    "5. Notify user"
  ]

Step 2: EXECUTOR LLM (per step)
  - calls tools as needed for each step
  - reports progress

Step 3: REPLANNER (on failure)
  - if step fails, replan remaining

✅ Better for complex tasks (less drift)
✅ Easier to monitor progress
❌ Plans can be wrong (LLM-generated plan)
❌ More LLM calls
```

### BabyAGI-style (Task list driven)

```
Loop:
  1. task_list.pop() → execute via Executor agent
  2. Result → Task Generator → new tasks added
  3. Prioritizer reorders queue
  4. Repeat until objective achieved or budget exhausted
```

### When to use which

```
Single tool call enough?           → Plain function calling
Few steps, dependent?              → ReAct loop
Many steps, predictable workflow?  → Plan-and-Execute
Open-ended exploration?            → BabyAGI-style
Multiple specialized roles needed? → Multi-agent (next section)
```

---

## 👥 Multi-agent Systems

### Frameworks

| Framework   | Lang      | Style                   | When                     |
| ----------- | --------- | ----------------------- | ------------------------ |
| **CrewAI**  | Python    | Role-based crew         | Sequential delegation    |
| **AutoGen** | Python/JS | Conversational agents   | Free-form discussion     |
| **LangGraph** | TS/Py    | State machine / graph   | 👑 production-grade      |
| **OpenAI Swarm** | Python | Lightweight handoffs   | Simple agent routing     |

### CrewAI-style Pattern (in TS)

```
Roles (Foodpanda support):
  - Triage agent: classifies query
  - Order agent: handles order issues
  - Refund agent: handles money issues
  - Menu agent: recommends food

Triage decides → hand off → specialist resolves → reply

Pros:
  ✅ Each agent focused, smaller prompt
  ✅ Easier eval per role
Cons:
  ❌ More LLM calls
  ❌ Handoff loss of context if not careful
```

### Communication Patterns

```
Hierarchical (orchestrator):
  Orchestrator → A → B → A → Orchestrator → user

Peer-to-peer (group chat):
  All agents see all messages, "speaker selection" picks next

Pipeline:
  agent1 → agent2 → agent3 → output (no return)

Blackboard:
  Shared scratchpad, agents read/write asynchronously
```

---

## 🧠 Memory: Short, Long, Episodic

```
┌─────────────────────────────────────────────────────────┐
│                    Memory Types                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. WORKING (short-term):                               │
│     - Last N messages in context window                 │
│     - "ConversationBufferWindow" (last 10)              │
│     - Cheap, lossy                                      │
│                                                         │
│  2. SUMMARY:                                            │
│     - LLM summarizes old turns periodically             │
│     - Summary + recent N turns in context               │
│     - "ConversationSummaryBuffer"                       │
│                                                         │
│  3. LONG-TERM (vector store):                           │
│     - All conversations embedded → vector DB            │
│     - Retrieve relevant past on each turn               │
│     - User preferences, past issues                     │
│                                                         │
│  4. EPISODIC (event log):                               │
│     - Structured events: "user ordered biryani"         │
│     - Queryable by time, type                           │
│     - DB: Postgres event_log table                      │
│                                                         │
│  5. SEMANTIC (knowledge):                               │
│     - Facts about user: "lives in Banani, vegetarian"   │
│     - Updated by LLM extraction                         │
│     - DB: user_facts JSON column                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Practical Stack

```javascript
// Memory layers
const memory = {
  working: lastMessages.slice(-10),
  summary: await summarizer.summarize(allMessages.slice(0, -10)),
  longTerm: await vectorMemory.search(userQuery, { userId, k: 3 }),
  facts: await getUserFacts(userId),  // {area: "Banani", veg: true, ...}
};

const systemPrompt = `
User profile: ${JSON.stringify(memory.facts)}
Past conversation summary: ${memory.summary}
Relevant past chats: ${memory.longTerm.map(m => m.text).join('\n')}

Recent messages:
${memory.working.map(m => `${m.role}: ${m.content}`).join('\n')}
`;
```

---

## 📈 LangGraph Deep Dive

LangGraph = state machine for agents. Each node is a function (or LLM call), edges define transitions.

### Why graph over loop?

```
Plain ReAct loop:
  while (!done) { call_llm(); execute_tool(); }
  - Hard to inspect mid-flow
  - Hard to checkpoint
  - Hard to add human-in-the-loop

LangGraph:
  State = TypedDict
  Nodes = pure functions on state
  Edges = conditional / static
  Built-in: persistence, time-travel, streaming, HITL
```

### Anatomy

```javascript
import { StateGraph, START, END } from '@langchain/langgraph';
import { z } from 'zod';

// 1. Define state schema
const State = z.object({
  messages: z.array(z.any()),
  user_id: z.number().optional(),
  intent: z.string().optional(),
  retries: z.number().default(0),
});

// 2. Define nodes
async function classifyIntent(state) {
  const intent = await llm.classify(state.messages);
  return { intent };
}

async function callTools(state) {
  // execute tools based on LLM decision
  return { messages: [...state.messages, toolResult] };
}

// 3. Build graph
const graph = new StateGraph(State)
  .addNode('classify', classifyIntent)
  .addNode('tools', callTools)
  .addNode('respond', generateResponse)
  .addEdge(START, 'classify')
  .addConditionalEdges('classify', (s) => {
    if (s.intent === 'order_status') return 'tools';
    return 'respond';
  })
  .addEdge('tools', 'respond')
  .addEdge('respond', END);

const app = graph.compile({ checkpointer: postgresSaver });
```

### Features

- **Checkpointing**: state persisted in Postgres/SQLite, resume mid-execution
- **Time travel**: rewind to any past state, branch
- **Human-in-the-loop**: pause node, wait for human approval
- **Streaming**: token-stream + state-stream
- **Parallel branches**: fan-out/fan-in

---

## 💻 Computer / Browser Use Agents

```
Anthropic Claude Computer Use, OpenAI Operator, browser-use:

Tools:
  - take_screenshot()  → image to LLM (vision)
  - click(x, y)
  - type(text)
  - scroll(direction)
  - wait(ms)
  - read_page() → DOM/text

Pattern:
  1. Screenshot
  2. LLM decides action
  3. Execute via Playwright/Puppeteer
  4. Loop

Use cases (BD context):
  - Price scraping competitor sites
  - Form-filling automation
  - QA testing internal apps
  - "Scroll Daraz, screenshot top 10 phones, summarize"

Risks:
  - Misclicks → wrong purchase 😱
  - Captchas
  - Rate limits / IP bans
  - Cost (vision tokens expensive)

Always:
  - Sandbox browser (separate user, no SSO cookies)
  - Confirm-before-purchase / destructive actions
  - Hard step limit
  - Read-only mode for prod data
```

---

## 🐍 Code Interpreter / Sandbox

```
LLM writes code → runs in sandbox → reads stdout/files

Use cases:
  - Data analysis on CSV upload
  - Charts generation
  - Math problems too complex for LLM math

Sandbox options (BD-friendly):
  - E2B (cloud, easy API)
  - Pyodide (browser-side WASM)
  - Docker per-request (DIY, secure)
  - Jupyter Kernel Gateway
  - Modal Labs (serverless)
```

### Mini-example

```javascript
import { Sandbox } from 'e2b';

const sbx = await Sandbox.create({ template: 'python' });
const code = `
import pandas as pd
df = pd.read_csv('orders.csv')
print(df.groupby('city')['total'].sum().sort_values(ascending=False).head())
`;
const exec = await sbx.runCode(code);
console.log(exec.logs.stdout);
await sbx.close();
```

---

## 🛡️ Safety & Sandboxing

### Layers

```
┌──────────────────────────────────────────────────┐
│            Defense in Depth                      │
├──────────────────────────────────────────────────┤
│                                                  │
│  L1 LLM Layer:                                   │
│    - System prompt: "never run rm -rf"           │
│    - Tool descriptions discourage misuse         │
│    - JSON schema strict                          │
│                                                  │
│  L2 Application Layer:                           │
│    - Schema validation (Zod)                     │
│    - Permission check per tool                   │
│    - Rate limit per user/tool                    │
│    - Allow/deny lists for tool args              │
│                                                  │
│  L3 Network Layer:                               │
│    - egress firewall (no S3, no internet)        │
│    - SSRF protection                             │
│                                                  │
│  L4 OS Layer:                                    │
│    - Linux namespaces, cgroups (see              │
│      ../20-operating-systems/namespaces-cgroups.md) │
│    - seccomp                                     │
│                                                  │
│  L5 VM/Hypervisor Layer:                         │
│    - Firecracker microVMs (AWS Lambda)           │
│    - gVisor (Google sandboxing)                  │
│    - Docker rootless                             │
│                                                  │
└──────────────────────────────────────────────────┘

Risk vs Layer:
  Just a calculator tool?       L1 + L2 enough
  Code execution?               L1-L4 mandatory
  Untrusted code with internet? L1-L5 mandatory
```

### Sandbox Options Comparison

| Sandbox       | Isolation | Speed     | Cost    | BD-deployable |
| ------------- | --------- | --------- | ------- | ------------- |
| Docker        | medium    | fast      | low     | ✅            |
| gVisor        | high      | medium    | low     | ✅ (k8s addon) |
| Firecracker   | very high | very fast | low     | ✅ (advanced)  |
| nsjail        | medium    | very fast | low     | ✅            |
| pyodide WASM  | very high | medium    | none    | ✅ (browser)   |
| E2B (managed) | high      | fast      | $$      | API only      |

---

## 💸 Cost & Loop Runaway Protection

```
Real horror story:
  Agent stuck in loop calling tool 1000× → $200 in 10 min 😱

Defenses:
  1. Hard max_steps (e.g., 15)
  2. Per-conversation token budget
  3. Tool call dedup: same args within N steps → block
  4. Loop detection: same Thought repeated → escalate
  5. Cost dashboard per user/agent
  6. Alarm: > $X/hr per agent
```

```javascript
class AgentBudget {
  constructor(maxSteps, maxTokens, maxCostUsd) {
    this.steps = 0;
    this.tokens = 0;
    this.cost = 0;
    this.limits = { maxSteps, maxTokens, maxCostUsd };
    this.calledTools = new Map(); // for loop detection
  }

  recordStep() {
    if (++this.steps > this.limits.maxSteps)
      throw new BudgetExceededError('max_steps');
  }

  recordLLM(usage, model) {
    this.tokens += usage.total_tokens;
    this.cost += pricePerToken[model] * usage.total_tokens;
    if (this.tokens > this.limits.maxTokens)
      throw new BudgetExceededError('tokens');
    if (this.cost > this.limits.maxCostUsd)
      throw new BudgetExceededError('cost');
  }

  recordToolCall(name, args) {
    const key = `${name}:${JSON.stringify(args)}`;
    const n = (this.calledTools.get(key) ?? 0) + 1;
    this.calledTools.set(key, n);
    if (n > 3) throw new BudgetExceededError(`tool_loop:${name}`);
  }
}
```

---

## 💻 কোড: LangGraph Foodpanda Agent

```bash
npm i @langchain/langgraph @langchain/openai @langchain/core zod \
      pg redis ioredis
```

```typescript
// foodpanda-agent.ts
import { StateGraph, START, END, MessagesAnnotation } from '@langchain/langgraph';
import { ToolNode } from '@langchain/langgraph/prebuilt';
import { ChatOpenAI } from '@langchain/openai';
import { tool } from '@langchain/core/tools';
import { z } from 'zod';
import { Pool } from 'pg';
import Redis from 'ioredis';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL!);

// ─────────────────── Tool 1: search_menu (vector + filter)
const searchMenu = tool(
  async ({ query, area, max_price, veg }) => {
    const qVec = await embed(query);
    const sql = `
      SELECT m.id, m.name_bn, m.name_en, m.price, r.name AS restaurant, r.area,
             1 - (m.embedding <=> $1::vector) AS sim
      FROM menu_items m JOIN restaurants r ON r.id = m.restaurant_id
      WHERE r.is_open = true
        ${area ? "AND r.area = $2" : ""}
        ${max_price ? "AND m.price <= $3" : ""}
        ${veg ? "AND m.is_veg = true" : ""}
      ORDER BY m.embedding <=> $1::vector
      LIMIT 10
    `;
    const params = [`[${qVec.join(',')}]`];
    if (area) params.push(area);
    if (max_price) params.push(String(max_price));
    const { rows } = await pool.query(sql, params);
    return JSON.stringify(rows);
  },
  {
    name: 'search_menu',
    description: 'Search Foodpanda menu items by query + filters',
    schema: z.object({
      query:     z.string().describe('Search text in Bangla/English'),
      area:      z.enum(['Banani','Gulshan','Dhanmondi','Uttara']).optional(),
      max_price: z.number().positive().optional(),
      veg:       z.boolean().optional(),
    }),
  }
);

// ─────────────────── Tool 2: get_order_status
const getOrderStatus = tool(
  async ({ user_id, order_id }, { configurable }) => {
    if (configurable.user_id !== user_id) {
      return JSON.stringify({ error: 'forbidden' });
    }
    const { rows } = await pool.query(
      `SELECT id, status, eta_minutes, total, items
       FROM orders
       WHERE user_id = $1 ${order_id ? 'AND id = $2' : ''}
       ORDER BY created_at DESC LIMIT 1`,
      order_id ? [user_id, order_id] : [user_id]
    );
    return JSON.stringify(rows[0] ?? null);
  },
  {
    name: 'get_order_status',
    description: 'Get latest order status for the authenticated user',
    schema: z.object({
      user_id:  z.number().int(),
      order_id: z.string().optional(),
    }),
  }
);

// ─────────────────── Tool 3: calculate_total
const calculateTotal = tool(
  async ({ items }) => {
    const total = items.reduce(
      (s, it) => s + it.price * it.quantity, 0
    );
    const delivery = total >= 500 ? 0 : 49;
    return JSON.stringify({ subtotal: total, delivery, total: total + delivery });
  },
  {
    name: 'calculate_total',
    description: 'Calculate cart total with delivery fee',
    schema: z.object({
      items: z.array(z.object({
        price: z.number(),
        quantity: z.number().int().positive(),
      })),
    }),
  }
);

// ─────────────────── Rate-limit & budget wrapper
function withGuards(t) {
  return tool(async (args, ctx) => {
    const userId = ctx.configurable.user_id;
    const key = `tool:${t.name}:${userId}`;
    const cnt = await redis.incr(key);
    if (cnt === 1) await redis.expire(key, 60);
    if (cnt > 30) return JSON.stringify({ error: 'rate_limited' });
    return t.invoke(args, ctx);
  }, t);
}

const tools = [searchMenu, getOrderStatus, calculateTotal].map(withGuards);

// ─────────────────── LLM with tools bound
const llm = new ChatOpenAI({
  model: 'gpt-4o-mini',
  temperature: 0.3,
}).bindTools(tools);

// ─────────────────── Nodes
async function callModel(state) {
  if (state.messages.length > 30) {
    return { messages: [{ role: 'system', content: 'Conversation too long.' }] };
  }
  const response = await llm.invoke(state.messages);
  return { messages: [response] };
}

const toolNode = new ToolNode(tools);

function shouldContinue(state) {
  const last = state.messages[state.messages.length - 1];
  if (last.tool_calls?.length) return 'tools';
  return END;
}

// ─────────────────── Graph
const workflow = new StateGraph(MessagesAnnotation)
  .addNode('agent', callModel)
  .addNode('tools', toolNode)
  .addEdge(START, 'agent')
  .addConditionalEdges('agent', shouldContinue)
  .addEdge('tools', 'agent');

const app = workflow.compile({
  // Persist state in Postgres for resume / HITL
  // checkpointer: new PostgresSaver(pool),
});

// ─────────────────── Run
export async function runAgent(userId: number, userMessage: string) {
  const sysPrompt = `তুমি Foodpanda Bangladesh-এর ভার্চুয়াল assistant।
- শুধু tool call করে real data আনবে; guess করবে না।
- User-এর user_id সবসময় ${userId}, অন্য কারো order দেখাবে না।
- দাম BDT (৳) তে দেখাও।
- সংক্ষেপে Bangla-তে answer দাও।`;

  const out = await app.invoke({
    messages: [
      { role: 'system', content: sysPrompt },
      { role: 'user',   content: userMessage },
    ],
  }, {
    configurable: { user_id: userId, thread_id: `user-${userId}` },
    recursionLimit: 12,
  });

  return out.messages[out.messages.length - 1].content;
}

// Usage
const reply = await runAgent(123,
  'Banani এলাকায় ৩০০ টাকার নিচে veg খাবার কী আছে?'
);
console.log(reply);
```

### Streaming Token-by-Token

```typescript
const stream = await app.stream(
  { messages: [...] },
  { configurable: { user_id: 123 }, streamMode: 'messages' }
);
for await (const chunk of stream) {
  process.stdout.write(chunk[0].content ?? '');
}
```

### Human-in-the-Loop (Approval before destructive tool)

```typescript
const app = workflow.compile({
  checkpointer,
  interruptBefore: ['tools'],   // pause before any tool
});

// Caller resumes after human approves
await app.invoke(null, { configurable: { thread_id: '...' } });
```

---

## 🐘 PHP Tool Gateway

PHP backend can expose tools as REST endpoints; Node agent calls them. This keeps domain logic in PHP (existing Laravel/Symfony app).

```php
<?php
// routes/api.php (Laravel)
Route::middleware(['auth:agent-token'])->prefix('tools')->group(function () {
    Route::post('search-menu',       [ToolController::class, 'searchMenu']);
    Route::post('get-order-status',  [ToolController::class, 'getOrderStatus']);
    Route::post('calculate-total',   [ToolController::class, 'calculateTotal']);
});

// app/Http/Controllers/ToolController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Services\MenuSearch;

class ToolController extends Controller
{
    public function searchMenu(Request $req, MenuSearch $svc)
    {
        $data = $req->validate([
            'query'     => 'required|string|min:1|max:200',
            'area'      => 'nullable|in:Banani,Gulshan,Dhanmondi,Uttara',
            'max_price' => 'nullable|numeric|min:0',
            'veg'       => 'nullable|boolean',
        ]);

        $userId = $req->attributes->get('agent_user_id');

        // Rate limit per user+tool
        $key = "tool:search_menu:{$userId}";
        if (cache()->increment($key) > 60) {
            return response()->json(['error' => 'rate_limited'], 429);
        }
        cache()->put($key, cache()->get($key, 1), now()->addMinute());

        $results = $svc->search($data);
        return response()->json($results);
    }

    public function getOrderStatus(Request $req)
    {
        $data = $req->validate([
            'order_id' => 'nullable|string',
        ]);
        $userId = $req->attributes->get('agent_user_id');

        $order = \DB::table('orders')
            ->where('user_id', $userId)
            ->when($data['order_id'] ?? null,
                fn($q, $id) => $q->where('id', $id))
            ->latest()->first();

        return response()->json($order);
    }
}
```

### Auto-generate JSON Schema for Tool Manifest

```php
// app/Console/Commands/ExportToolManifest.php
public function handle(): int
{
    $manifest = [
        [
            'type' => 'function',
            'function' => [
                'name' => 'search_menu',
                'description' => 'Search Foodpanda BD menu',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'query'     => ['type' => 'string'],
                        'area'      => ['type' => 'string', 'enum' => ['Banani','Gulshan','Dhanmondi','Uttara']],
                        'max_price' => ['type' => 'number'],
                        'veg'       => ['type' => 'boolean'],
                    ],
                    'required' => ['query'],
                ],
            ],
        ],
        // ... more
    ];
    file_put_contents(base_path('tool-manifest.json'),
        json_encode($manifest, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE));
    return 0;
}
```

Node agent fetches `tool-manifest.json` and calls `/api/tools/<name>` for execution.

---

## 🇧🇩 BD: Pathao Foods Order-Status Agent

### Use Case

```
User (Bangla): "আমার order এখনো আসেনি, কী অবস্থা?"

Agent steps:
  1. Tool: get_user_orders(user_id, status='active')
     → {order_id: 'PF-9981', status: 'rider_assigned', eta: 12 min,
        rider_phone: '01700-XXX'}
  2. Tool: get_rider_location(order_id)
     → {distance_km: 1.2, eta_min: 8}
  3. Compose Bangla reply with rider info + action buttons
     ("Call rider", "Cancel order")
```

### Architecture

```
┌────────────────────────────────────────────────────────┐
│  Pathao Foods Agent                                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Mobile app → API gateway → /v1/agent/chat (Node)      │
│       │                                                │
│       ▼                                                │
│  LangGraph agent (gpt-4o-mini)                         │
│       │                                                │
│       ▼                                                │
│  Tools (REST → PHP backend):                           │
│     get_user_orders, get_rider_location,               │
│     cancel_order (HITL approval), search_menu,         │
│     apply_voucher, raise_complaint                     │
│       │                                                │
│       ▼                                                │
│  Memory: Redis (last 10 turns) + Postgres (long-term)  │
│       │                                                │
│       ▼                                                │
│  Observability: Langfuse + Prometheus                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Numbers

```
Daily conversations:    50K
Avg turns/conversation: 3
Avg tools/turn:         1.4
LLM cost:               $200/day (gpt-4o-mini)
Tool call latency P95:  120ms
End-to-end P95:         2.8s
Containment rate:       72% (no human escalation)
CSAT:                   4.4 / 5
```

### Lessons

1. **Strict tool schema** prevents hallucinated SQL
2. **HITL on cancel/refund** avoided fraud
3. **Rate limit per user/tool** caught buggy retries early
4. **Bangla system prompt** crucial—English prompt → English bleed-through
5. **Trace every step** in Langfuse — debug agent behavior weekly
6. **Allow-list of areas/restaurants** prevents nonsense filters

---

## ⚠️ Common Pitfalls

```
1. ❌ Too many tools (>20) → LLM confused
   Fix: route via classifier first, sub-agents

2. ❌ Vague tool descriptions
   "get_data" — what data? When?
   Fix: include examples in description

3. ❌ Tool name collisions
   "search" twice for menu vs orders → LLM picks wrong
   Fix: namespacing: "menu_search", "orders_search"

4. ❌ No timeout on tool
   Slow API hangs whole agent

5. ❌ Returning huge tool output
   1MB JSON → blows context, latency
   Fix: paginate, summarize before return

6. ❌ State not persisted
   Crash mid-conversation = lost progress
   Fix: LangGraph checkpointer

7. ❌ Loop without budget
   Infinite tool calls = bankruptcy
   Fix: max_steps + token budget + dedup

8. ❌ Trust LLM with destructive ops
   "delete_account" tool → eventually hallucinated
   Fix: HITL approval, irreversible actions on read-only LLM

9. ❌ Cross-user data leak
   tool reads global, doesn't check user_id
   Fix: inject user_id from auth, never accept from LLM

10. ❌ No fallback on LLM error
    LLM 500 → conversation dies
    Fix: retry, fallback model, graceful "try later"
```

---

## 📋 Production Checklist

```
Architecture:
  ☐ Tool registry with schema validation
  ☐ State persistence (LangGraph checkpointer)
  ☐ Memory layers (working + summary + vector)
  ☐ Multi-agent only if needed (avoid premature complexity)

Tools:
  ☐ Each tool: clear name, description, examples
  ☐ Zod / JSON Schema strict validation
  ☐ Per-tool rate limit
  ☐ Per-tool timeout
  ☐ Per-tool permission (RBAC)
  ☐ user_id injected by host, NOT trusted from LLM
  ☐ Idempotency keys for write tools

Safety:
  ☐ HITL for destructive actions
  ☐ Sandboxed code execution (Docker/Firecracker/E2B)
  ☐ Network egress controlled
  ☐ Prompt injection defense
  ☐ Output content filter

Cost & Limits:
  ☐ max_steps per run
  ☐ Token budget per session
  ☐ Tool-call dedup
  ☐ Loop detection alarm
  ☐ Per-user cost dashboard

Observability:
  ☐ Trace each step (Langfuse / LangSmith)
  ☐ Latency P50/P95/P99
  ☐ Tool success rate per tool
  ☐ Containment rate (no escalation)
  ☐ User satisfaction (thumbs up/down)

Reliability:
  ☐ Retry with backoff on LLM errors
  ☐ Fallback model
  ☐ Graceful degradation (no tools → plain reply)
  ☐ Idempotent tool execution (retry-safe)
```

---

## 🎯 সারমর্ম

```
Agent = LLM + Tools + Loop + State

When to use Agent:
  ✅ Real-time data needed
  ✅ Multi-step task
  ✅ Side effects (write to DB / API)
  ❌ Simple Q&A → use RAG
  ❌ Static knowledge → use prompt-only

BD reality:
  - LangGraph + Node.js: best DX
  - PHP exposes tools as REST (existing app)
  - gpt-4o-mini cheapest sweet spot for Bangla
  - Always sandbox, always observe, always budget
```

---

## 🔗 আরও পড়ুন

- [MCP Protocol](./mcp-protocol.md) — standard tool interface
- [RAG Systems](./rag-systems.md) — Agentic RAG
- [LLM Ops](./llm-ops.md) — production agent monitoring
- [Namespaces & cgroups](../20-operating-systems/namespaces-cgroups.md) — sandboxing
- [API Gateway](../06-api-design/api-gateway.md) — tool gateway pattern
- [Authentication](../06-api-design/authentication.md) — agent token / RBAC
