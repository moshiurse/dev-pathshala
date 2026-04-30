# 🔌 MCP — Model Context Protocol

> **MCP (Model Context Protocol)** = AI tools/data-source-গুলোকে যেকোনো LLM host (Claude Desktop, Cursor, Cline, কাস্টম agent) এর সাথে standard interface-এ connect করার protocol।
> Anthropic 2024-এ open-spec হিসেবে release করেছে। **"AI-এর জন্য USB-C"**—একবার server বানালে যেকোনো client থেকে কাজে আসবে।

---

## 📖 সূচিপত্র

- [MCP কী এবং কেন](#-mcp-কী-এবং-কেন)
- [Architecture](#-architecture)
- [Transports](#-transports)
- [Capabilities](#-capabilities-resources-tools-prompts-sampling)
- [Discovery & Negotiation](#-discovery--negotiation)
- [Security Model](#-security-model)
- [MCP vs OpenAPI vs LangChain Tools](#-mcp-vs-openapi-vs-langchain-tools)
- [Build MCP Server (Node.js)](#-build-mcp-server-nodejs--bkash-transactions)
- [Build MCP Client](#-build-mcp-client)
- [Integrate with Claude Desktop / IDE](#-integrate-with-claude-desktop--ide)
- [PHP MCP Server Outline](#-php-mcp-server-outline)
- [Production Deployment](#-production-deployment)
- [Future Direction](#-future-direction)
- [Pitfalls](#-pitfalls)
- [Checklist](#-checklist)

---

## 🎯 MCP কী এবং কেন

### সমস্যা: M × N integration explosion

```
Without MCP:
─────────────
       Claude     GPT-4    Gemini   Local LLM
         │          │         │         │
   ┌─────┴────┬─────┴─────────┴─────────┘
   │          │
 bKash tool  GitHub tool  Notion tool  Postgres tool ...
   (custom)  (custom)     (custom)     (custom)

প্রতিটা LLM × প্রতিটা tool = M × N integrations 😱
নতুন LLM আসলে সব tools আবার লিখতে হয়
```

### সমাধান: MCP standard

```
With MCP:
──────────
   Claude     GPT-4    Gemini   Custom agent       ← Hosts (M)
     │          │         │          │
     └──────────┴─── MCP ─┴──────────┘             ← Standard wire format
                    │
       ┌────────────┼────────────┐
       │            │            │
   bKash MCP   GitHub MCP   Postgres MCP ...       ← Servers (N)

M + N integrations। নতুন LLM-এ কাজ already করে।
```

### Why now?

```
আগে:
- প্রত্যেক agent framework নিজস্ব tool format
- LangChain tool ≠ OpenAI function ≠ Anthropic tool
- Tool reuse impossible

MCP-এর সাথে:
- "bKash transactions" একবার server বানালে
  → Claude Desktop, Cursor, ChatGPT MCP, Cline, custom—সবাই use করে
- Local stdio বা remote HTTP—same protocol
- Resource (data) + Tool (action) + Prompt (template) — সব একসাথে
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────┐                                             │
│  │   HOST     │  Claude Desktop, Cursor, custom agent       │
│  │ (the LLM   │  - Manages connections                      │
│  │   app)     │  - Asks user for consent                    │
│  └─────┬──────┘                                             │
│        │ owns                                               │
│        ▼                                                    │
│  ┌────────────┐                                             │
│  │  CLIENT    │  Per-server stateful connection             │
│  │ (1 per     │  - Speaks JSON-RPC 2.0                      │
│  │  server)   │  - Negotiates capabilities                  │
│  └─────┬──────┘                                             │
│        │ MCP wire (stdio / HTTP+SSE / Streamable HTTP)      │
│        ▼                                                    │
│  ┌────────────┐                                             │
│  │  SERVER    │  Exposes resources, tools, prompts          │
│  │  (yours)   │  - Process or HTTP service                  │
│  │            │  - Talks to underlying APIs / DBs           │
│  └────────────┘                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Wire format: JSON-RPC 2.0

```json
// Request
{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "get_transactions", "arguments": { "limit": 10 } } }

// Response
{ "jsonrpc": "2.0", "id": 1,
  "result": { "content": [{ "type": "text", "text": "..." }] } }

// Notification (no response)
{ "jsonrpc": "2.0", "method": "notifications/resources/updated",
  "params": { "uri": "bkash://account/123" } }
```

---

## 🚇 Transports

### 1) stdio (local desktop apps)

```
Host spawns server as child process
stdin  ← JSON-RPC requests
stdout → JSON-RPC responses
stderr → logs

Pros:
  ✅ Zero network surface (most secure for local tools)
  ✅ Easy: "node my-server.js"
  ✅ Per-user isolation
Cons:
  ❌ Local only, no sharing
  ❌ Each host spawns its own instance
```

### 2) HTTP + SSE (legacy, deprecated for new servers)

```
POST /messages → server-side requests
GET  /sse      → server-sent events (server → client)
```

### 3) Streamable HTTP (current standard, post-2025)

```
Single endpoint POST /mcp:
  - Client sends JSON-RPC request
  - Server responds with either:
    - JSON (one-shot)
    - SSE stream (long-running with notifications)
  - Optional session via Mcp-Session-Id header
  - Resumable streams via Last-Event-ID

Pros:
  ✅ Works through CDN, load balancers
  ✅ Auth via standard HTTP (Bearer tokens, OAuth)
  ✅ Multi-user same server
  ✅ Cloud-deployable
```

---

## 🎁 Capabilities (Resources, Tools, Prompts, Sampling)

```
┌──────────────────────────────────────────────────────────────┐
│  MCP exposes 4 primitives                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. RESOURCES  (data — read-only)                            │
│     - URI-addressable: file://, https://, custom://          │
│     - User explicitly attaches to context                    │
│     - bKash example: bkash://transactions/last-30-days       │
│                                                              │
│  2. TOOLS  (actions — model-invoked)                         │
│     - LLM decides when to call                               │
│     - Side effects allowed                                   │
│     - bKash example: get_transactions, send_money            │
│                                                              │
│  3. PROMPTS  (slash commands — user-invoked)                 │
│     - User picks from menu: /summarize-transactions          │
│     - Server returns templated message(s)                    │
│                                                              │
│  4. SAMPLING  (server asks host for LLM completion)          │
│     - Reverse: server requests LLM call from host            │
│     - Use: agentic server logic, recursion                   │
│     - Requires user consent (security boundary)              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Comparison

| Primitive | Who initiates | Use case                     | Consent  |
| --------- | ------------- | ---------------------------- | -------- |
| Resource  | User attaches | Provide context (file, doc)  | Per-attach |
| Tool      | LLM           | Action / fetch (often)       | Per-server (or per-call) |
| Prompt    | User          | Reusable workflow            | Per-invoke |
| Sampling  | Server        | Server-side LLM reasoning    | Per-server (cautious) |

---

## 🔍 Discovery & Negotiation

```
Lifecycle:
1. initialize:
   client → server: { protocolVersion: "2024-11-05",
                      capabilities: {...}, clientInfo: {...} }
   server → client: { protocolVersion, capabilities: {
                        resources: {...}, tools: {...},
                        prompts: {...}, sampling: {...}
                      }, serverInfo: {...} }

2. initialized notification (client → server)

3. List capabilities:
   tools/list, resources/list, prompts/list

4. Use:
   tools/call, resources/read, prompts/get

5. Notifications:
   tools/list_changed, resources/updated, ...

6. Shutdown
```

---

## 🛡️ Security Model

```
Threats to defend against:
- Malicious server: steals data via tool call
- Malicious client: misuses server beyond user intent
- Malicious LLM (prompt injection): tries to call destructive tool
- Confused deputy: server calls another server with wrong auth

MCP defenses:
1. Host owns consent UX
   - User approves each server (Claude Desktop checkbox)
   - User approves each tool call (configurable strictness)
   - User attaches resources explicitly

2. Server announces dangerous tools
   - "destructive": true, "openWorld": true hints

3. Sampling requires explicit user opt-in
   - Server can't silently use LLM credits

4. Transport-layer auth
   - stdio: process boundary
   - HTTP: Bearer / OAuth / mTLS

5. Sandbox the server (you, the developer)
   - Run in container, dropped privileges
   - Egress firewall
```

### Permission Model Example

```
Claude Desktop config:
  "bkash-mcp": {
    "command": "node",
    "args": ["./bkash-mcp/dist/index.js"],
    "env": { "BKASH_API_KEY": "..." },
    "permissions": {
      "tools": {
        "get_transactions": "allow",
        "send_money":       "ask"   // confirm each call
      }
    }
  }
```

---

## ⚖️ MCP vs OpenAPI vs LangChain Tools

| Aspect              | MCP                          | OpenAPI Tool                  | LangChain Tool        |
| ------------------- | ---------------------------- | ----------------------------- | --------------------- |
| Standard owner      | Anthropic + community        | OpenAI Initiative             | LangChain (lib)       |
| Wire format         | JSON-RPC 2.0                 | HTTP + JSON (REST)            | In-process Python/JS  |
| Discovery           | tools/list, resources/list   | OpenAPI spec URL              | Code import           |
| Streaming           | ✅ SSE / Streamable HTTP     | ⚠️ depends                    | ⚠️ depends             |
| Resources (read-only)| ✅ first-class               | ❌                            | ⚠️ via tools           |
| Prompt templates    | ✅ first-class               | ❌                            | ✅ (separate)          |
| Local + remote      | ✅ stdio + HTTP              | HTTP only                     | In-process or API     |
| Cross-vendor        | ✅✅ Anthropic, OpenAI, Google integrations growing | Mostly OpenAI    | Framework-specific    |
| Sampling (reverse)  | ✅                           | ❌                            | ❌                    |

> **TL;DR:** OpenAPI = great for typed REST APIs called via tool calling. MCP = standard for **stateful AI tool plugins** with discovery, resources, prompts, and dual transports. LangChain tools = framework-internal. MCP supersedes most use cases.

---

## 💻 Build MCP Server (Node.js) — bKash Transactions

### Spec

We'll build a server that exposes:

- **Tools**: `get_transactions`, `get_balance`, `categorize_spending`
- **Resource**: `bkash://account/transactions/recent` (last 30 days, attachable)
- **Prompt**: `summarize_month` (templated user message for monthly summary)

### Setup

```bash
mkdir bkash-mcp && cd bkash-mcp
npm init -y
npm i @modelcontextprotocol/sdk zod
npm i -D typescript @types/node tsx
npx tsc --init --target es2022 --module nodenext --moduleResolution nodenext \
  --outDir dist --rootDir src --esModuleInterop --strict
mkdir src
```

### Server Code

```typescript
// src/index.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';

// ─────────────────── Mock bKash API client (replace with real)
class BkashAPI {
  constructor(private apiKey: string, private accountId: string) {}

  async listTransactions(opts: { from?: string; to?: string; limit?: number }) {
    // Real: fetch https://tokenized.pay.bka.sh/v1.2.0-beta/...
    return [
      { id: 'TX001', date: '2024-09-20', type: 'send_money',
        amount: 500, recipient: '01700000000', note: 'rent' },
      { id: 'TX002', date: '2024-09-21', type: 'merchant',
        amount: 250, recipient: 'Foodpanda', note: 'lunch' },
      // ...
    ].slice(0, opts.limit ?? 50);
  }

  async getBalance() { return { balance_bdt: 12_450 }; }
}

const api = new BkashAPI(
  process.env.BKASH_API_KEY ?? '',
  process.env.BKASH_ACCOUNT_ID ?? ''
);

// ─────────────────── Tool schemas
const GetTransactionsSchema = z.object({
  from:  z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
  to:    z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
  limit: z.number().int().min(1).max(200).default(50),
});

const CategorizeSchema = z.object({
  transactions: z.array(z.object({
    id: z.string(), amount: z.number(), recipient: z.string(),
  })),
});

// ─────────────────── Server
const server = new Server(
  { name: 'bkash-mcp', version: '0.1.0' },
  { capabilities: { tools: {}, resources: {}, prompts: {} } }
);

// ── tools/list
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'get_transactions',
      description: 'Fetch bKash transactions for a date range. ' +
                   'Returns array of {id, date, type, amount, recipient, note}.',
      inputSchema: {
        type: 'object',
        properties: {
          from:  { type: 'string', format: 'date',
                   description: 'YYYY-MM-DD (inclusive)' },
          to:    { type: 'string', format: 'date',
                   description: 'YYYY-MM-DD (inclusive)' },
          limit: { type: 'integer', minimum: 1, maximum: 200, default: 50 },
        },
      },
    },
    {
      name: 'get_balance',
      description: 'Get current bKash account balance in BDT',
      inputSchema: { type: 'object', properties: {} },
    },
    {
      name: 'categorize_spending',
      description: 'Group transactions into spend categories ' +
                   '(food, transport, rent, bills, other).',
      inputSchema: {
        type: 'object',
        properties: {
          transactions: { type: 'array', items: { type: 'object' } },
        },
        required: ['transactions'],
      },
    },
  ],
}));

// ── tools/call
server.setRequestHandler(CallToolRequestSchema, async (req) => {
  const { name, arguments: rawArgs } = req.params;

  try {
    switch (name) {
      case 'get_transactions': {
        const args = GetTransactionsSchema.parse(rawArgs);
        const txs = await api.listTransactions(args);
        return {
          content: [{ type: 'text', text: JSON.stringify(txs, null, 2) }],
        };
      }
      case 'get_balance': {
        const b = await api.getBalance();
        return {
          content: [{ type: 'text', text: `Balance: ৳${b.balance_bdt}` }],
        };
      }
      case 'categorize_spending': {
        const args = CategorizeSchema.parse(rawArgs);
        const cats = categorize(args.transactions);
        return {
          content: [{ type: 'text', text: JSON.stringify(cats, null, 2) }],
        };
      }
      default:
        return { content: [{ type: 'text', text: `Unknown tool: ${name}` }],
                 isError: true };
    }
  } catch (e: any) {
    return {
      content: [{ type: 'text',
                  text: `Error: ${e.message ?? 'unknown'}` }],
      isError: true,
    };
  }
});

function categorize(txs: { recipient: string; amount: number }[]) {
  const rules: Array<[RegExp, string]> = [
    [/foodpanda|chaldal|pathao foods/i, 'food'],
    [/uber|pathao|cng|bus/i,            'transport'],
    [/rent|bari|landlord/i,             'rent'],
    [/electric|gas|water|internet|wifi|robi|grameen|banglalink/i, 'bills'],
  ];
  const totals: Record<string, number> = {};
  for (const tx of txs) {
    let cat = 'other';
    for (const [re, c] of rules) if (re.test(tx.recipient)) { cat = c; break; }
    totals[cat] = (totals[cat] ?? 0) + tx.amount;
  }
  return totals;
}

// ── resources/list
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: 'bkash://account/transactions/recent',
      name: 'Recent Transactions (30 days)',
      description: 'JSON dump of last 30 days of bKash transactions',
      mimeType: 'application/json',
    },
  ],
}));

// ── resources/read
server.setRequestHandler(ReadResourceRequestSchema, async (req) => {
  if (req.params.uri === 'bkash://account/transactions/recent') {
    const today = new Date();
    const from = new Date(today.getTime() - 30 * 86400_000)
      .toISOString().slice(0, 10);
    const txs = await api.listTransactions({
      from, to: today.toISOString().slice(0, 10), limit: 200,
    });
    return {
      contents: [{
        uri: req.params.uri,
        mimeType: 'application/json',
        text: JSON.stringify(txs, null, 2),
      }],
    };
  }
  throw new Error(`Unknown resource: ${req.params.uri}`);
});

// ── prompts/list
server.setRequestHandler(ListPromptsRequestSchema, async () => ({
  prompts: [{
    name: 'summarize_month',
    description: 'Summarize last month\'s bKash spending in Bangla',
    arguments: [
      { name: 'month', description: 'YYYY-MM (default: last month)',
        required: false },
    ],
  }],
}));

// ── prompts/get
server.setRequestHandler(GetPromptRequestSchema, async (req) => {
  const month = req.params.arguments?.month ?? lastMonth();
  return {
    description: `bKash summary for ${month}`,
    messages: [{
      role: 'user',
      content: {
        type: 'text',
        text: `${month} মাসের bKash transaction গুলো বিশ্লেষণ করে ` +
              `Bangla-তে summarize করো। মোট ব্যয়, ক্যাটাগরি অনুযায়ী ` +
              `breakdown, এবং কোথায় বেশি খরচ হয়েছে সেই insight দাও।\n\n` +
              `প্রথমে get_transactions tool call করো from=${month}-01 to=${monthEnd(month)}, ` +
              `তারপর categorize_spending tool দিয়ে categories বের করো।`,
      },
    }],
  };
});

function lastMonth() {
  const d = new Date(); d.setMonth(d.getMonth() - 1);
  return d.toISOString().slice(0, 7);
}
function monthEnd(m: string) {
  const [y, mo] = m.split('-').map(Number);
  return new Date(y, mo, 0).toISOString().slice(0, 10);
}

// ─────────────────── Start
const transport = new StdioServerTransport();
await server.connect(transport);
console.error('bkash-mcp server running on stdio');
```

### Run & Test

```bash
npx tsx src/index.ts
# Or build & run:
npx tsc && node dist/index.js
```

Test with **MCP Inspector**:

```bash
npx @modelcontextprotocol/inspector node dist/index.js
# Opens browser UI to call tools / read resources / get prompts
```

---

## 🌐 Streamable HTTP Variant

```typescript
// src/http.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StreamableHTTPServerTransport } from
  '@modelcontextprotocol/sdk/server/streamableHttp.js';
import express from 'express';

const app = express();
app.use(express.json());

app.post('/mcp', async (req, res) => {
  // Auth
  const token = req.headers.authorization?.replace(/^Bearer /, '');
  if (!await verifyToken(token)) return res.status(401).end();

  const server = buildServer(/* per-user context */);
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: () => crypto.randomUUID(),
  });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3030);
```

---

## 🖥️ Build MCP Client

```typescript
// client.ts
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from
  '@modelcontextprotocol/sdk/client/stdio.js';

const transport = new StdioClientTransport({
  command: 'node',
  args: ['./dist/index.js'],
  env: { BKASH_API_KEY: process.env.BKASH_API_KEY ?? '' },
});

const client = new Client(
  { name: 'my-agent', version: '1.0.0' },
  { capabilities: {} }
);

await client.connect(transport);

// List tools
const { tools } = await client.listTools();
console.log(tools.map(t => t.name));

// Call tool
const out = await client.callTool({
  name: 'get_transactions',
  arguments: { from: '2024-09-01', to: '2024-09-30', limit: 100 },
});
console.log(out.content);

// Read resource
const res = await client.readResource({
  uri: 'bkash://account/transactions/recent',
});
console.log(res.contents);

// List & get prompt
const { prompts } = await client.listPrompts();
const p = await client.getPrompt({
  name: 'summarize_month', arguments: { month: '2024-09' },
});
console.log(p.messages);

await client.close();
```

### Bridge to LLM (Anthropic SDK example)

```typescript
import Anthropic from '@anthropic-ai/sdk';
const ai = new Anthropic();

async function chat(userMsg: string) {
  let messages = [{ role: 'user' as const, content: userMsg }];
  const tools = (await client.listTools()).tools.map(t => ({
    name: t.name, description: t.description, input_schema: t.inputSchema,
  }));

  while (true) {
    const r = await ai.messages.create({
      model: 'claude-3-5-sonnet-latest',
      max_tokens: 1024, tools, messages,
    });
    if (r.stop_reason === 'end_turn') return r.content;

    if (r.stop_reason === 'tool_use') {
      const toolUses = r.content.filter(c => c.type === 'tool_use');
      messages.push({ role: 'assistant', content: r.content });

      const results = await Promise.all(toolUses.map(async (tu: any) => {
        const out = await client.callTool({ name: tu.name, arguments: tu.input });
        return { type: 'tool_result' as const, tool_use_id: tu.id,
                 content: out.content };
      }));
      messages.push({ role: 'user', content: results });
    }
  }
}
```

---

## 🖱️ Integrate with Claude Desktop / IDE

### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (mac) /
`%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "bkash": {
      "command": "node",
      "args": ["/absolute/path/to/bkash-mcp/dist/index.js"],
      "env": {
        "BKASH_API_KEY":  "sk_live_...",
        "BKASH_ACCOUNT_ID": "01700000000"
      }
    }
  }
}
```

Restart Claude Desktop → ⚙ icon shows "bkash" connected → tools/resources available.

### Cursor / Cline / Continue / Zed

Same config pattern (JSON file with `mcpServers`). Each IDE documents its config path.

### Custom Host

Use `@modelcontextprotocol/sdk` Client API as shown above.

---

## 🐘 PHP MCP Server Outline

PHP-এ MCP server (community SDK: `php-mcp/server`):

```bash
composer require php-mcp/server
```

```php
<?php
// server.php
require __DIR__ . '/vendor/autoload.php';

use PhpMcp\Server\Server;
use PhpMcp\Server\Transports\StdioServerTransport;
use PhpMcp\Server\Attributes\McpTool;
use PhpMcp\Server\Attributes\McpResource;
use PhpMcp\Server\Attributes\McpPrompt;

final class BkashMcp
{
    public function __construct(private \PDO $pdo) {}

    #[McpTool(
        name: 'get_transactions',
        description: 'Fetch bKash transactions for date range'
    )]
    public function getTransactions(
        ?string $from = null,
        ?string $to = null,
        int $limit = 50
    ): array {
        $stmt = $this->pdo->prepare(
            "SELECT id, date, type, amount, recipient, note
             FROM transactions
             WHERE date BETWEEN COALESCE(:f, '1970-01-01')
                            AND COALESCE(:t, CURRENT_DATE)
             ORDER BY date DESC LIMIT :l"
        );
        $stmt->bindValue(':f', $from);
        $stmt->bindValue(':t', $to);
        $stmt->bindValue(':l', $limit, \PDO::PARAM_INT);
        $stmt->execute();
        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }

    #[McpTool(name: 'get_balance', description: 'Current account balance')]
    public function getBalance(): array
    {
        return ['balance_bdt' => 12450];
    }

    #[McpResource(
        uri: 'bkash://account/transactions/recent',
        name: 'Recent Transactions',
        mimeType: 'application/json'
    )]
    public function recentTransactions(): string
    {
        $tx = $this->getTransactions(date('Y-m-d', strtotime('-30 days')),
                                     date('Y-m-d'), 200);
        return json_encode($tx, JSON_UNESCAPED_UNICODE);
    }

    #[McpPrompt(
        name: 'summarize_month',
        description: "Bangla summary of last month's spending"
    )]
    public function summarizeMonth(?string $month = null): array
    {
        $month ??= date('Y-m', strtotime('first day of last month'));
        return [
            ['role' => 'user', 'content' =>
                "$month মাসের bKash transactions বিশ্লেষণ করে Bangla-তে " .
                "summary দাও। মোট ব্যয়, category breakdown, insight।"],
        ];
    }
}

$pdo = new \PDO('pgsql:host=...;dbname=bkash', 'u', 'p');
$server = Server::make()->withTool(new BkashMcp($pdo))->build();
$server->run(new StdioServerTransport());
```

For HTTP transport, deploy via PHP-FPM + Nginx with the SDK's `StreamableHttpServerTransport`.

---

## 🚀 Production Deployment

### Stdio (per-user desktop)

- Ship as installable binary or `npm install -g`
- Sign binaries (Apple notarization on mac)
- Auto-update mechanism
- Local secrets in OS keychain (not env file in repo)

### Streamable HTTP (multi-user SaaS)

```
┌───────────────────────────────────────────────────────┐
│  Production HTTP MCP Stack                            │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Cloudflare / CDN (TLS, DDoS, geo)                    │
│        │                                              │
│        ▼                                              │
│  API Gateway (auth, rate limit, logs)                 │
│        │                                              │
│        ▼                                              │
│  MCP Server pods (k8s, autoscale)                     │
│        │                                              │
│        ▼                                              │
│  Backend services (bKash API, DB, cache)              │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### Auth Patterns

```
1. Bearer token (per-user API key):
   - Issued from your app's settings page
   - Scoped: "read:transactions", "write:none"

2. OAuth 2.0:
   - User clicks "Connect bKash" in Claude Desktop
   - Redirect → consent → token

3. mTLS (B2B):
   - Mutual cert for enterprise clients

4. Workload identity (cloud):
   - GCP/Azure managed identity for service-to-service
```

### Rate Limiting

```
Per user:           60 req/min
Per tool:           varies (write tools stricter)
Per session:        5K req/hour
Enforcement:        Redis token bucket (see api-gateway.md)
```

### Observability

```
Metrics:
  - mcp.tool.calls{name, status}
  - mcp.tool.latency_ms
  - mcp.session.count
  - mcp.errors{code}

Tracing:
  - OpenTelemetry: client → server → backend
  - Trace ID in MCP request meta

Logging:
  - JSON structured
  - PII redaction (bKash account, phone)
  - Sample 10% of successful, 100% errors
```

### Compliance (Bangladesh financial)

```
- Bangladesh Bank guidelines for fintech APIs
- Personal Data Protection Act 2023 (PDPA-BD draft)
- No PII in LLM training (logs scrubbed)
- Data residency: BD/SG region only
- Audit log: every tool call signed, 7-year retention
- Consent: explicit user OAuth before any read
```

---

## 🔮 Future Direction

```
- Streamable HTTP becoming default
- OAuth 2.1 for MCP standardized
- Server registries (npm-like for discoverability)
- More native LLM support: OpenAI MCP support, Gemini
- Standardized "computer use" via MCP
- Sub-resources, resource templates with arguments
- Versioning & deprecation hints
- Privacy-preserving sampling (encrypted prompts)
```

---

## ⚠️ Pitfalls

```
1. ❌ Returning huge tool results
   200 KB JSON → blows context, latency
   Fix: paginate, summarize, compress

2. ❌ Synchronous slow tool blocks the LLM
   Fix: async with progress notifications, use SSE

3. ❌ No schema validation server-side
   Trusting LLM input → injection
   Fix: Zod / JSON Schema strict

4. ❌ Mixing per-user state in stdio process
   Stdio is per-user; HTTP needs explicit session mgmt
   Fix: HTTP server stateless or session-keyed

5. ❌ Putting secrets in tool args
   LLM logs them → leak
   Fix: server holds secrets via env, args are scoped IDs only

6. ❌ Forgetting prompt safety
   Prompts can be exploited if echo user input
   Fix: validate prompt args, escape

7. ❌ No version compat handling
   Server upgrade breaks old clients
   Fix: protocolVersion negotiation, additive changes

8. ❌ "destructive" tool exposed without HITL
   LLM hallucination → wipes account
   Fix: confirmation step, idempotency, dry-run

9. ❌ Local stdio server with internet access
   Security gap
   Fix: egress restriction, allowlist endpoints

10. ❌ No auth on HTTP MCP
    Public internet → anyone uses your bKash tools
    Fix: token / OAuth mandatory
```

---

## 📋 Checklist

```
Design:
  ☐ Tools: clear name, detailed description with examples
  ☐ Resources: stable URIs, versioned schema
  ☐ Prompts: idempotent templates
  ☐ Schemas: strict (Zod / JSON Schema)
  ☐ Pagination on list outputs
  ☐ Error model: { isError: true, content: [...] }

Security:
  ☐ Schema validation before any side effect
  ☐ Auth (Bearer/OAuth) on HTTP transport
  ☐ Rate limit per user/tool
  ☐ Idempotency keys for write tools
  ☐ HITL for destructive actions
  ☐ Secrets out-of-band (env, vault), never in args
  ☐ PII redaction in logs

Compatibility:
  ☐ protocolVersion handling
  ☐ Capability negotiation
  ☐ Backward-compat changes only
  ☐ Tested with multiple hosts (Claude Desktop, Cursor, custom)

Operations:
  ☐ Inspector smoke test in CI
  ☐ Tool call traces (OTel)
  ☐ Metrics dashboard
  ☐ Health endpoint (HTTP)
  ☐ Graceful shutdown (drain in-flight)
  ☐ Crash-safe (file/DB writes idempotent)

Distribution:
  ☐ npm package OR signed binary OR Docker image
  ☐ Install docs
  ☐ Example config snippet for Claude Desktop
  ☐ Versioned releases, changelog
```

---

## 🎯 সারমর্ম

```
MCP = AI-এর USB-C
  Hosts (LLM apps) ←→ Clients ←→ Servers (your tools)

Build once, work everywhere:
  - Claude Desktop, Cursor, Cline, custom agents

Capabilities: Tools, Resources, Prompts, Sampling
Transports:   stdio (local), Streamable HTTP (cloud)
Security:     User consent, server schemas, transport auth

BD use cases:
  - bKash personal finance MCP
  - Pathao driver guideline MCP
  - Daraz seller dashboard MCP
  - Government doc / NID lookup MCP (with strict consent)

Future: OAuth 2.1 MCP, server registries, full computer-use
```

---

## 🔗 আরও পড়ুন

- [Spec](https://modelcontextprotocol.io)
- [LLM Agents & Tools](./llm-agents-tools.md) — concepts behind MCP tools
- [API Gateway](../06-api-design/api-gateway.md) — HTTP MCP behind gateway
- [Authentication](../06-api-design/authentication.md) — OAuth / Bearer for MCP
- [Networking](../14-networking/README.md) — SSE, HTTP/2 fundamentals
