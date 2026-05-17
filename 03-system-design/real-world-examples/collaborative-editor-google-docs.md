# 📝 কোলাবোরেটিভ ডকুমেন্ট এডিটর সিস্টেম ডিজাইন (Google Docs)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Google Docs, Notion, Microsoft 365, Figma

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#️-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| 1 | Real-time Collaborative Editing | একাধিক ব্যবহারকারী একই ডকুমেন্ট একই সময়ে edit করতে পারবে |
| 2 | Cursor Presence | অন্য editor-দের cursor position রিয়েল-টাইমে দেখা যাবে |
| 3 | Comments & Suggestions | ডকুমেন্টের নির্দিষ্ট অংশে comment ও suggestion দেওয়া যাবে |
| 4 | Version History | পুরনো version-এ ফিরে যাওয়া যাবে, কে কখন কী পরিবর্তন করেছে দেখা যাবে |
| 5 | Sharing & Permissions | View, Comment, Edit — বিভিন্ন level-এর access control |
| 6 | Offline Editing | ইন্টারনেট ছাড়াও edit করা যাবে, পরে sync হবে |
| 7 | Rich Text Formatting | Bold, italic, heading, table, image — সব ধরনের formatting |

### Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Operation Propagation Latency | < 50ms (same region) |
| Conflict Resolution | Conflict-free merging (কোনো data loss নেই) |
| Concurrent Editors | 100+ একই ডকুমেন্টে |
| Document Size | 1000+ page support |
| Availability | 99.9% uptime |
| Offline Duration | ২৪ ঘণ্টা পর্যন্ত offline edit, তারপর auto-merge |

### 🇧🇩 Bangladesh Context

```
বাস্তব ব্যবহারের ক্ষেত্র:
─────────────────────────────
১. বিশ্ববিদ্যালয় গ্রুপ প্রজেক্ট:
   - BUET/DU/KUET-এর ছাত্ররা thesis paper একসাথে লিখছে
   - ৫-১০ জন একই document-এ কাজ করছে
   - রাত ২টায় deadline, সবাই ঢাকা-চট্টগ্রাম-রাজশাহী থেকে edit করছে

২. রিমোট টিম কোলাবোরেশন:
   - বাংলাদেশী software company-র distributed team
   - PRD (Product Requirement Document) লেখা
   - Sprint planning document তৈরি করা

৩. সরকারি ডকুমেন্ট ম্যানেজমেন্ট:
   - মন্ত্রণালয়ের নীতিমালা draft করা
   - একাধিক বিভাগের কর্মকর্তারা review করছেন
   - Digital Bangladesh উদ্যোগের অংশ হিসেবে paperless office

৪. মিডিয়া ও সাংবাদিকতা:
   - Prothom Alo/Daily Star-এর reporter ও editor একসাথে কাজ করছে
   - Breaking news article দ্রুত collaborate করে লেখা
```

---

## 📊 Back-of-envelope Estimation

### ট্রাফিক ও স্টোরেজ হিসাব

```
ধরি, Bangladesh-focused platform:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total Users:              ৫০ লক্ষ (5M)
Daily Active Users (DAU): ১০ লক্ষ (1M)
Concurrent Users (peak):  ২ লক্ষ (200K)
Documents Created/Day:    ৫ লক্ষ (500K)

প্রতি Document:
─────────────────
গড় আকার:               50 KB (text + formatting metadata)
গড় Operations/min:      30 keystrokes × active users
গড় Active Editors:      3 জন/document

Operations Per Second (Peak):
─────────────────────────────
Active Documents (concurrent): 50,000
Ops per doc per second:        3 users × 5 ops/sec = 15 ops/sec
Total OPS:                     50,000 × 15 = 750,000 ops/sec

Storage (Daily):
────────────────
New Documents:    500K × 50KB = 25 GB/day
Operation Log:   750K ops/sec × 86400 sec × 100 bytes = ~6.5 TB/day (raw)
                 Compacted (snapshot every 5 min): ~200 GB/day
Version History: 500K docs × 10 versions × 5KB diff = 25 GB/day

Network Bandwidth:
──────────────────
WebSocket Messages: 750K/sec × 200 bytes = 150 MB/sec = 1.2 Gbps
Peak (2x):          2.4 Gbps

Bangladesh Network Reality:
───────────────────────────
Average broadband speed: 40 Mbps
Mobile (4G):            10-15 Mbps
Latency (Dhaka DC):     5-20ms
Latency (intl, SG):     50-80ms
→ Edge server ঢাকায় রাখা essential
```

---

## 🏗️ High-Level Design

### সিস্টেম আর্কিটেকচার ওভারভিউ

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Browser  │  │ Browser  │  │ Mobile   │  │ Desktop  │          │
│  │ Editor A │  │ Editor B │  │ App      │  │ App      │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │              │              │              │                 │
│       └──────────────┴──────────────┴──────────────┘                │
│                          │ WebSocket + HTTP                          │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────────┐
│                    GATEWAY LAYER                                      │
├──────────────────────────┼──────────────────────────────────────────┤
│                          ▼                                           │
│  ┌───────────────────────────────────────┐                          │
│  │        Load Balancer (Nginx)          │                          │
│  │   (Sticky Sessions by doc_id)         │                          │
│  └───────────────┬───────────────────────┘                          │
│                  │                                                    │
└──────────────────┼──────────────────────────────────────────────────┘
                   │
┌──────────────────┼──────────────────────────────────────────────────┐
│            APPLICATION LAYER                                         │
├──────────────────┼──────────────────────────────────────────────────┤
│                  ▼                                                    │
│  ┌────────────────────┐    ┌─────────────────────────┐              │
│  │  PHP (Laravel)     │    │  Node.js (Express)       │              │
│  │  ─────────────     │    │  ───────────────────     │              │
│  │  • Document CRUD   │    │  • WebSocket Server      │              │
│  │  • Auth & Perms    │    │  • OT/CRDT Engine        │              │
│  │  • Sharing API     │    │  • Cursor Presence       │              │
│  │  • Version History │    │  • Real-time Sync        │              │
│  │  • Export/Import   │    │  • Conflict Resolution   │              │
│  └────────┬───────────┘    └──────────┬──────────────┘              │
│           │                            │                             │
└───────────┼────────────────────────────┼─────────────────────────────┘
            │                            │
┌───────────┼────────────────────────────┼─────────────────────────────┐
│           ▼          DATA LAYER        ▼                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  PostgreSQL  │  │    Redis     │  │  MongoDB     │              │
│  │  ──────────  │  │  ─────────  │  │  ─────────   │              │
│  │  Users       │  │  Sessions   │  │  Document    │              │
│  │  Permissions │  │  Cursors    │  │  Content     │              │
│  │  Metadata    │  │  Pub/Sub    │  │  Op Log      │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐                                 │
│  │     S3       │  │ Elasticsearch│                                 │
│  │  ─────────   │  │  ──────────  │                                 │
│  │  Attachments │  │  Full-text   │                                 │
│  │  Exports     │  │  Search      │                                 │
│  └──────────────┘  └──────────────┘                                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-time Data Flow

```
User A types "Hello"         User B sees "Hello"
      │                              ▲
      ▼                              │
┌──────────┐                  ┌──────────┐
│ Client A │                  │ Client B │
│ Local Op │                  │ Apply Op │
└────┬─────┘                  └────▲─────┘
     │                              │
     │ ① Send Operation             │ ⑤ Broadcast
     ▼                              │
┌────────────────────────────────────────┐
│         WebSocket Server (Node.js)      │
│                                         │
│  ② Receive Op                           │
│  ③ Transform against concurrent ops     │
│  ④ Apply to server document state       │
│  ⑤ Broadcast transformed op to others   │
└────────────────────────────────────────┘
     │
     │ ④ Persist
     ▼
┌──────────────┐
│  Operation   │
│    Log       │
│  (MongoDB)   │
└──────────────┘
```

---

## 💻 Detailed Design

### ১. Operational Transformation (OT) vs CRDT

```
┌─────────────────────────────────────────────────────────────────┐
│              OT (Operational Transformation)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  মূলনীতি: Operations কে transform করে conflict resolve করা       │
│                                                                  │
│  Original: "ABCD"                                                │
│                                                                  │
│  User A: Insert('X', pos=1)  →  "AXBCD"                        │
│  User B: Delete(pos=2)       →  "ABD"                           │
│                                                                  │
│  Server receives A first, then B:                                │
│  After A: "AXBCD"                                                │
│  B needs transform: Delete(pos=2) → Delete(pos=3)               │
│  Result: "AXBD"  ✓ (consistent for all)                         │
│                                                                  │
│  Pros: ✓ কম memory ব্যবহার                                      │
│        ✓ Google Docs এটি ব্যবহার করে                             │
│        ✓ Central server-এ simple implementation                   │
│  Cons: ✗ Central server required                                 │
│        ✗ Complex transform functions                             │
│        ✗ O(n²) worst case                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              CRDT (Conflict-free Replicated Data Type)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  মূলনীতি: Data structure নিজেই conflict-free                     │
│                                                                  │
│  প্রতিটি character-এর unique ID আছে:                            │
│  "A"     "B"     "C"     "D"                                    │
│  (1,A)   (2,A)   (3,A)   (4,A)                                 │
│                                                                  │
│  User A inserts 'X' between (1,A) and (2,A):                    │
│  New: (1.5, B) = 'X'                                            │
│                                                                  │
│  User B deletes (3,A):                                           │
│  Tombstone: (3,A) marked as deleted                              │
│                                                                  │
│  Result: কোনো transform দরকার নেই! Merge automatically!          │
│                                                                  │
│  Pros: ✓ No central server needed                                │
│        ✓ Offline-friendly                                        │
│        ✓ Mathematically proven convergence                       │
│  Cons: ✗ বেশি memory (tombstones)                                │
│        ✗ Complex implementation                                  │
│        ✗ Garbage collection needed                               │
└─────────────────────────────────────────────────────────────────┘
```

### ২. Database Schema Design

```sql
-- PostgreSQL: Users & Permissions
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    avatar_url TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL DEFAULT 'Untitled Document',
    owner_id UUID REFERENCES users(id),
    is_public BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE document_permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    permission_level VARCHAR(20) NOT NULL, -- 'view', 'comment', 'edit', 'admin'
    granted_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(document_id, user_id)
);

CREATE TABLE document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    snapshot JSONB NOT NULL, -- full document state at this point
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- MongoDB: Operation Log (high write throughput)
// Collection: operations
{
    _id: ObjectId,
    document_id: "uuid-string",
    user_id: "uuid-string",
    version: 1542,          // server version number
    operation: {
        type: "insert",     // insert, delete, retain, format
        position: 45,
        content: "Hello",
        attributes: { bold: true }
    },
    timestamp: ISODate("2024-01-15T10:30:00Z")
}

// Collection: document_content
{
    _id: ObjectId,
    document_id: "uuid-string",
    content: [...],         // CRDT/OT document state
    version: 1542,
    last_snapshot_version: 1500,
    active_cursors: [
        { user_id: "uuid", position: 45, color: "#FF5733", name: "রাহুল" }
    ]
}
```

### ৩. PHP (Laravel) — Document API & Permission Management

```php
<?php
// app/Http/Controllers/DocumentController.php

namespace App\Http\Controllers;

use App\Models\Document;
use App\Models\DocumentPermission;
use App\Services\DocumentVersionService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class DocumentController extends Controller
{
    private DocumentVersionService $versionService;

    public function __construct(DocumentVersionService $versionService)
    {
        $this->versionService = $versionService;
    }

    /**
     * নতুন ডকুমেন্ট তৈরি করা
     * Example: BUET CSE-18 batch-এর thesis document
     */
    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:500',
            'template' => 'nullable|string|in:blank,report,thesis,meeting_notes',
        ]);

        $document = Document::create([
            'title' => $validated['title'],
            'owner_id' => auth()->id(),
        ]);

        // Owner gets admin permission
        DocumentPermission::create([
            'document_id' => $document->id,
            'user_id' => auth()->id(),
            'permission_level' => 'admin',
            'granted_by' => auth()->id(),
        ]);

        // Initialize document content in MongoDB
        $this->initializeContent($document, $validated['template'] ?? 'blank');

        // Notify real-time server about new document
        $this->notifyRealtimeServer('document.created', [
            'document_id' => $document->id,
            'owner' => auth()->user()->name,
        ]);

        return response()->json([
            'document' => $document,
            'edit_url' => route('documents.edit', $document->id),
            'share_link' => route('documents.share', $document->id),
        ], 201);
    }

    /**
     * ডকুমেন্ট শেয়ার করা — permission management
     * বাংলাদেশ context: সরকারি office-এ document sharing hierarchy
     */
    public function share(Request $request, Document $document)
    {
        Gate::authorize('admin', $document);

        $validated = $request->validate([
            'emails' => 'required|array|min:1',
            'emails.*' => 'email',
            'permission' => 'required|in:view,comment,edit,admin',
            'message' => 'nullable|string|max:500',
            'expires_at' => 'nullable|date|after:now',
        ]);

        $sharedWith = [];

        foreach ($validated['emails'] as $email) {
            $user = \App\Models\User::where('email', $email)->first();

            if (!$user) {
                // Invite user — বাংলাদেশে email invitation
                $user = $this->inviteUser($email);
            }

            $permission = DocumentPermission::updateOrCreate(
                ['document_id' => $document->id, 'user_id' => $user->id],
                [
                    'permission_level' => $validated['permission'],
                    'granted_by' => auth()->id(),
                    'expires_at' => $validated['expires_at'] ?? null,
                ]
            );

            $sharedWith[] = [
                'user' => $user->email,
                'permission' => $validated['permission'],
            ];
        }

        // Send notification emails
        \App\Jobs\SendShareNotification::dispatch(
            $document, $sharedWith, $validated['message'] ?? null
        );

        return response()->json([
            'message' => 'Document shared successfully',
            'shared_with' => $sharedWith,
        ]);
    }

    /**
     * Version History — কোন version-এ restore করতে চান?
     */
    public function versions(Document $document)
    {
        Gate::authorize('view', $document);

        $versions = $this->versionService->getVersionHistory($document->id);

        return response()->json([
            'document_id' => $document->id,
            'versions' => $versions->map(fn($v) => [
                'version_number' => $v->version_number,
                'created_by' => $v->creator->name,
                'created_at' => $v->created_at->diffForHumans(),
                'changes_summary' => $v->changes_summary,
            ]),
        ]);
    }

    /**
     * Restore a specific version
     */
    public function restoreVersion(Request $request, Document $document, int $version)
    {
        Gate::authorize('edit', $document);

        $restored = $this->versionService->restoreToVersion($document->id, $version);

        // Notify all active editors about version restore
        $this->notifyRealtimeServer('document.version_restored', [
            'document_id' => $document->id,
            'version' => $version,
            'restored_by' => auth()->user()->name,
        ]);

        return response()->json([
            'message' => "Version {$version} restored successfully",
            'document' => $restored,
        ]);
    }

    private function notifyRealtimeServer(string $event, array $data): void
    {
        // Redis pub/sub to Node.js server
        \Illuminate\Support\Facades\Redis::publish('doc-events', json_encode([
            'event' => $event,
            'data' => $data,
            'timestamp' => now()->toISOString(),
        ]));
    }
}
```

```php
<?php
// app/Services/DocumentVersionService.php

namespace App\Services;

use App\Models\DocumentVersion;
use MongoDB\Client as MongoClient;

class DocumentVersionService
{
    private MongoClient $mongo;

    public function __construct()
    {
        $this->mongo = new MongoClient(config('database.connections.mongodb.dsn'));
    }

    /**
     * প্রতি ৫ মিনিটে বা ১০০ operations পর snapshot নেওয়া
     * এটি version history ও performance-এর জন্য গুরুত্বপূর্ণ
     */
    public function createSnapshot(string $documentId): void
    {
        $collection = $this->mongo->collab_editor->document_content;
        $currentDoc = $collection->findOne(['document_id' => $documentId]);

        if (!$currentDoc) return;

        $lastVersion = DocumentVersion::where('document_id', $documentId)
            ->max('version_number') ?? 0;

        DocumentVersion::create([
            'document_id' => $documentId,
            'version_number' => $lastVersion + 1,
            'snapshot' => $currentDoc['content'],
            'created_by' => $currentDoc['last_edited_by'] ?? null,
        ]);

        // Update last snapshot version in MongoDB
        $collection->updateOne(
            ['document_id' => $documentId],
            ['$set' => ['last_snapshot_version' => $currentDoc['version']]]
        );
    }

    /**
     * Version restore — Operation log replay করে নির্দিষ্ট version-এ ফেরত যাওয়া
     */
    public function restoreToVersion(string $documentId, int $targetVersion): array
    {
        $snapshot = DocumentVersion::where('document_id', $documentId)
            ->where('version_number', $targetVersion)
            ->firstOrFail();

        $collection = $this->mongo->collab_editor->document_content;
        $collection->updateOne(
            ['document_id' => $documentId],
            ['$set' => [
                'content' => $snapshot->snapshot,
                'version' => $snapshot->version_number,
                'restored_at' => new \MongoDB\BSON\UTCDateTime(),
            ]]
        );

        return ['version' => $targetVersion, 'content' => $snapshot->snapshot];
    }
}
```

### ৪. Node.js (Express) — WebSocket Server with OT Engine

```javascript
// server.js - Real-time Collaboration Server
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const Redis = require('ioredis');
const { MongoClient } = require('mongodb');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
    cors: { origin: '*' },
    transports: ['websocket', 'polling'], // Bangladesh-এ polling fallback গুরুত্বপূর্ণ
});

const redis = new Redis(process.env.REDIS_URL);
const redisSub = new Redis(process.env.REDIS_URL);
const mongoClient = new MongoClient(process.env.MONGODB_URI);

// Document sessions in memory
const documentSessions = new Map();

class DocumentSession {
    constructor(docId) {
        this.docId = docId;
        this.version = 0;
        this.operations = []; // buffered ops since last snapshot
        this.activeUsers = new Map(); // userId -> { socketId, cursor, name, color }
        this.pendingOps = []; // ops waiting to be processed
    }
}

/**
 * OT Transform Function
 * দুটি concurrent operation-কে transform করে যেন দুই পক্ষেই
 * consistent result আসে
 */
function transformOperation(opA, opB) {
    // opA came first (server order), opB needs transformation
    if (opA.type === 'insert' && opB.type === 'insert') {
        if (opA.position <= opB.position) {
            return { ...opB, position: opB.position + opA.content.length };
        }
        return opB;
    }

    if (opA.type === 'insert' && opB.type === 'delete') {
        if (opA.position <= opB.position) {
            return { ...opB, position: opB.position + opA.content.length };
        }
        return opB;
    }

    if (opA.type === 'delete' && opB.type === 'insert') {
        if (opA.position < opB.position) {
            return { ...opB, position: opB.position - opA.length };
        }
        return opB;
    }

    if (opA.type === 'delete' && opB.type === 'delete') {
        if (opA.position <= opB.position) {
            return { ...opB, position: opB.position - opA.length };
        }
        if (opA.position >= opB.position + opB.length) {
            return opB;
        }
        // Overlapping deletes — complex case
        const overlapStart = Math.max(opA.position, opB.position);
        const overlapEnd = Math.min(opA.position + opA.length, opB.position + opB.length);
        const overlapLength = overlapEnd - overlapStart;
        return {
            ...opB,
            position: Math.min(opA.position, opB.position),
            length: opB.length - overlapLength,
        };
    }

    return opB;
}

/**
 * Transform operation against all concurrent operations
 * Server-এ যে operations আগে apply হয়েছে সেগুলোর বিপরীতে transform
 */
function transformAgainstHistory(operation, concurrentOps) {
    let transformed = { ...operation };
    for (const prevOp of concurrentOps) {
        transformed = transformOperation(prevOp, transformed);
    }
    return transformed;
}

// WebSocket Connection Handler
io.on('connection', (socket) => {
    console.log(`Client connected: ${socket.id}`);

    /**
     * ডকুমেন্টে join করা
     * Bangladesh scenario: ৫০ জন ছাত্র exam preparation document-এ join করছে
     */
    socket.on('join-document', async (data) => {
        const { documentId, userId, userName, userColor } = data;

        // Verify permission (call Laravel API or check Redis cache)
        const hasAccess = await verifyPermission(documentId, userId);
        if (!hasAccess) {
            socket.emit('error', { message: 'Access denied' });
            return;
        }

        socket.join(documentId);

        // Get or create document session
        if (!documentSessions.has(documentId)) {
            const session = new DocumentSession(documentId);
            await loadDocumentState(session);
            documentSessions.set(documentId, session);
        }

        const session = documentSessions.get(documentId);
        session.activeUsers.set(userId, {
            socketId: socket.id,
            cursor: { position: 0, selection: null },
            name: userName,
            color: userColor || generateColor(userId),
        });

        // Send current document state to new joiner
        socket.emit('document-state', {
            content: session.content,
            version: session.version,
            activeUsers: Array.from(session.activeUsers.entries()).map(([id, info]) => ({
                userId: id,
                name: info.name,
                color: info.color,
                cursor: info.cursor,
            })),
        });

        // Notify others about new user
        socket.to(documentId).emit('user-joined', {
            userId,
            name: userName,
            color: userColor,
        });

        // Store socket metadata
        socket.data = { documentId, userId, userName };
    });

    /**
     * Operation receive করা ও process করা
     * এটি real-time collaboration-এর মূল অংশ
     */
    socket.on('operation', async (data) => {
        const { documentId, userId } = socket.data;
        const { operation, clientVersion } = data;

        const session = documentSessions.get(documentId);
        if (!session) return;

        // Get operations that happened since client's version
        const concurrentOps = session.operations.slice(clientVersion);

        // Transform the incoming operation
        const transformedOp = transformAgainstHistory(operation, concurrentOps);

        // Apply to server state
        applyOperation(session, transformedOp);
        session.version++;
        session.operations.push(transformedOp);

        // Persist to MongoDB operation log
        await persistOperation(documentId, userId, transformedOp, session.version);

        // Acknowledge to sender (with server version)
        socket.emit('ack', { version: session.version });

        // Broadcast transformed operation to all other clients
        socket.to(documentId).emit('remote-operation', {
            operation: transformedOp,
            version: session.version,
            userId,
            userName: socket.data.userName,
        });

        // Snapshot every 100 operations
        if (session.version % 100 === 0) {
            await createSnapshot(session);
        }
    });

    /**
     * Cursor position update
     * অন্যদের দেখানো — "রাকিব এখন ৩য় paragraph-এ আছে"
     */
    socket.on('cursor-update', (data) => {
        const { documentId, userId } = socket.data;
        const session = documentSessions.get(documentId);
        if (!session) return;

        const userInfo = session.activeUsers.get(userId);
        if (userInfo) {
            userInfo.cursor = data.cursor;
        }

        socket.to(documentId).emit('cursor-moved', {
            userId,
            name: socket.data.userName,
            color: userInfo?.color,
            cursor: data.cursor,
        });
    });

    /**
     * Disconnect handling
     * Bangladesh-এ power cut বা internet disconnect হলে graceful handle করা
     */
    socket.on('disconnect', () => {
        const { documentId, userId, userName } = socket.data || {};
        if (!documentId) return;

        const session = documentSessions.get(documentId);
        if (session) {
            session.activeUsers.delete(userId);

            // Notify others
            socket.to(documentId).emit('user-left', { userId, name: userName });

            // Clean up empty sessions after 5 minutes
            if (session.activeUsers.size === 0) {
                setTimeout(() => {
                    if (session.activeUsers.size === 0) {
                        documentSessions.delete(documentId);
                    }
                }, 5 * 60 * 1000);
            }
        }
    });
});

// Listen to Laravel events via Redis
redisSub.subscribe('doc-events');
redisSub.on('message', (channel, message) => {
    const { event, data } = JSON.parse(message);

    switch (event) {
        case 'document.version_restored':
            io.to(data.document_id).emit('version-restored', data);
            break;
        case 'document.permission_changed':
            io.to(data.document_id).emit('permission-changed', data);
            break;
    }
});

// Helper functions
async function loadDocumentState(session) {
    const db = mongoClient.db('collab_editor');
    const doc = await db.collection('document_content').findOne({
        document_id: session.docId,
    });
    if (doc) {
        session.content = doc.content;
        session.version = doc.version;
    }
}

async function persistOperation(docId, userId, operation, version) {
    const db = mongoClient.db('collab_editor');
    await db.collection('operations').insertOne({
        document_id: docId,
        user_id: userId,
        version,
        operation,
        timestamp: new Date(),
    });
}

function generateColor(userId) {
    const colors = ['#FF5733', '#33FF57', '#3357FF', '#FF33F5', '#F5FF33',
                    '#33FFF5', '#FF8C33', '#8C33FF', '#33FF8C', '#FF3333'];
    const hash = userId.split('').reduce((acc, c) => acc + c.charCodeAt(0), 0);
    return colors[hash % colors.length];
}

httpServer.listen(3001, () => {
    console.log('Real-time collaboration server running on port 3001');
});
```

### ৫. Offline Editing & Sync

```javascript
// client-side: OfflineManager.js
class OfflineManager {
    constructor(documentId) {
        this.documentId = documentId;
        this.pendingOps = []; // locally applied but not sent
        this.isOnline = navigator.onLine;
        this.db = null; // IndexedDB reference

        this.initIndexedDB();
        this.setupNetworkListeners();
    }

    /**
     * Bangladesh-এ frequently internet যায়-আসে
     * Load shedding, mobile network fluctuation — এসব handle করতে হবে
     */
    setupNetworkListeners() {
        window.addEventListener('online', () => {
            this.isOnline = true;
            this.syncPendingOperations();
        });

        window.addEventListener('offline', () => {
            this.isOnline = false;
            console.log('Offline mode activated — operations will be queued');
        });
    }

    /**
     * Offline-এ operation store করা (IndexedDB-তে)
     */
    async queueOperation(operation) {
        this.pendingOps.push({
            ...operation,
            timestamp: Date.now(),
            clientId: this.getClientId(),
        });

        // Persist to IndexedDB
        const tx = this.db.transaction('pending_ops', 'readwrite');
        await tx.objectStore('pending_ops').add({
            documentId: this.documentId,
            operation,
            timestamp: Date.now(),
        });
    }

    /**
     * Online হলে pending operations server-এ পাঠানো
     * Server OT engine transform করে conflict resolve করবে
     */
    async syncPendingOperations() {
        if (this.pendingOps.length === 0) return;

        console.log(`Syncing ${this.pendingOps.length} offline operations...`);

        // Send all pending ops as a batch
        const response = await fetch(`/api/documents/${this.documentId}/sync`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                operations: this.pendingOps,
                lastKnownVersion: this.lastServerVersion,
            }),
        });

        const result = await response.json();

        if (result.conflicts) {
            // Server transformed our ops — apply the transformed versions
            this.applyTransformedOps(result.transformedOps);
        }

        // Clear pending ops
        this.pendingOps = [];
        await this.clearIndexedDB();
    }
}
```

### ৬. Cursor & Presence Management

```
┌─────────────────────────────────────────────────────────┐
│                    Document Editor                        │
│                                                          │
│  The quick brown fox jumps │ over the lazy dog.         │
│                            ▲                             │
│                   রাহুল (🟢 green cursor)                │
│                                                          │
│  Lorem ipsum dolor ▌sit amet, consectetur               │
│                    ▲                                     │
│           সাবরিনা (🔵 blue cursor)                       │
│                                                          │
│  ████████████████████  ← তানভীর selecting (🟡 yellow)   │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ 👥 Active: রাহুল 🟢 | সাবরিনা 🔵 | তানভীর 🟡  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

Presence Data Structure (Redis):
─────────────────────────────────
Key: presence:{document_id}
Value: Hash
  {user_id_1}: { position: 45, selection: null, name: "রাহুল", color: "#33FF57", lastSeen: timestamp }
  {user_id_2}: { position: 102, selection: [102, 130], name: "সাবরিনা", color: "#3357FF", lastSeen: timestamp }
  {user_id_3}: { position: 78, selection: [78, 120], name: "তানভীর", color: "#F5FF33", lastSeen: timestamp }

TTL: 30 seconds (heartbeat every 10s)
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### OT vs CRDT — কোনটি বেছে নেবেন?

```
┌────────────────────┬──────────────────────────┬─────────────────────────────┐
│     বৈশিষ্ট্য      │         OT               │           CRDT              │
├────────────────────┼──────────────────────────┼─────────────────────────────┤
│ Architecture       │ Central Server Required  │ Decentralized/P2P possible  │
│ Complexity         │ Transform functions কঠিন │ Data structure কঠিন         │
│ Memory Usage       │ কম (only ops)            │ বেশি (tombstones, IDs)      │
│ Offline Support    │ Server reconnect দরকার   │ Excellent — merge anytime   │
│ Consistency        │ Server decides order     │ Eventually consistent       │
│ Latency           │ Server roundtrip needed   │ Local-first, instant        │
│ Real-world Use    │ Google Docs              │ Figma, Notion, Yjs          │
│ BD Context        │ Office-style collab      │ Mobile-first, offline       │
├────────────────────┼──────────────────────────┼─────────────────────────────┤
│ 🇧🇩 সুপারিশ       │ Stable internet-এ ভালো   │ বাংলাদেশের জন্য বেশি        │
│                    │ (office environment)      │ উপযুক্ত (offline support)    │
└────────────────────┴──────────────────────────┴─────────────────────────────┘
```

### Centralized vs Decentralized Architecture

```
Centralized (OT):                    Decentralized (CRDT):
─────────────────                    ─────────────────────

  Client A ──┐                        Client A ◄──► Client B
             │                              ▲           ▲
  Client B ──┼──► Server                    │           │
             │                              └─────┬─────┘
  Client C ──┘                                    │
                                           Client C

✓ Simple consistency              ✓ No single point of failure
✓ Easy permission control         ✓ Works without internet
✓ Simpler debugging               ✓ Lower latency (peer-to-peer)
✗ Server is bottleneck            ✗ Permission management কঠিন
✗ Server down = no editing        ✗ Debugging distributed state কঠিন
✗ Latency depends on server       ✗ বেশি bandwidth (gossip protocol)

🇧🇩 Bangladesh সিদ্ধান্ত:
═══════════════════════
Hybrid approach: CRDT locally + Central server for persistence
→ Internet থাকলে server sync, না থাকলে local CRDT
→ বাংলাদেশের unstable internet-এ best of both worlds
```

### Event Sourcing vs State-based Persistence

```
Event Sourcing:                     State-based:
───────────────                     ────────────
Store: [op1, op2, op3, ...]        Store: { final document state }

✓ Full audit trail                 ✓ Simple queries
✓ Time travel (any version)        ✓ কম storage
✓ Debugging সহজ                    ✓ Fast read
✗ Storage grows unbounded          ✗ No history
✗ Replay can be slow               ✗ Audit trail নেই
✗ Compaction needed                ✗ Version control কঠিন

আমাদের সিদ্ধান্ত: Event Sourcing + Periodic Snapshots
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
→ Operations log রাখি (audit + version history)
→ প্রতি ১০০ ops পর snapshot (fast loading)
→ ৩০ দিনের পুরনো ops archive করি (storage optimization)
```

### WebSocket vs WebRTC

```
┌────────────────┬────────────────────────┬─────────────────────────┐
│                │      WebSocket          │        WebRTC           │
├────────────────┼────────────────────────┼─────────────────────────┤
│ Connection     │ Client ↔ Server        │ Client ↔ Client (P2P)  │
│ Reliability    │ TCP-based, reliable    │ UDP-based, fast but     │
│                │                         │ unreliable              │
│ NAT Traversal  │ Proxy-friendly         │ STUN/TURN needed        │
│ Scalability    │ Server handles all     │ Mesh = O(n²) connections│
│ BD ISP Support │ সব ISP support করে     │ কিছু ISP block করে     │
│ Firewall       │ Port 80/443 দিয়ে যায়  │ Blocked in many BD      │
│                │                         │ corporate networks      │
├────────────────┼────────────────────────┼─────────────────────────┤
│ 🇧🇩 সিদ্ধান্ত  │ ✅ Primary choice      │ ⚠️ Optional for < 5     │
│                │ (reliable, ISP-safe)    │ users (low latency)     │
└────────────────┴────────────────────────┴─────────────────────────┘
```

---

## 📈 কেস স্টাডি

### কেস ১: ৫০ জন ছাত্র একই ডকুমেন্ট Edit করছে

```
Scenario: BUET CSE-18 batch-এর ৫০ জন ছাত্র একটি thesis compilation
          document-এ একই সময়ে edit করছে (submission deadline আজ রাত ১২টা)

Challenge:
──────────
• ৫০ concurrent editors = 50 × 5 ops/sec = 250 ops/sec per document
• প্রতিটি op সবাইকে broadcast = 250 × 49 = 12,250 messages/sec
• Network congestion: ঢাকায় রাত ১১টায় internet slow

Solution Architecture:
─────────────────────

  ┌─────────────────────────────────────────────────┐
  │            Operation Batching                     │
  │                                                   │
  │  Instead of: 1 op → broadcast to 49 users        │
  │  Do:         batch 10 ops → 1 message to 49      │
  │                                                   │
  │  Result: 12,250 msg/sec → 1,225 msg/sec (90%↓)  │
  └─────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────┐
  │            Document Sectioning                    │
  │                                                   │
  │  Document divided into sections:                  │
  │  Section 1: Introduction (editors: 5)             │
  │  Section 2: Literature Review (editors: 10)       │
  │  Section 3: Methodology (editors: 8)              │
  │  ...                                              │
  │                                                   │
  │  Only broadcast to editors in same section!        │
  │  Reduces cross-talk dramatically                  │
  └─────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────┐
  │            Cursor Throttling                      │
  │                                                   │
  │  Cursor updates: every 100ms instead of every     │
  │  keystroke. 50 users × 10 updates/sec = 500/sec  │
  │  (instead of 5000/sec)                            │
  └─────────────────────────────────────────────────┘

Performance Metrics:
────────────────────
Before optimization:  ~12,000 WebSocket messages/sec
After optimization:   ~1,700 WebSocket messages/sec
Latency (Dhaka):      15-25ms operation propagation
Memory (server):      ~200MB for 50-user document session
```

### কেস ২: Network Partition — Offline → Online Merge

```
Scenario: সরকারি অফিসে load shedding হলো, ২ জন officer offline-এ edit করলেন

Timeline:
─────────

  t=0     ┌─────────────────────────────────────┐
          │ Document: "বাংলাদেশ সরকারের নীতিমালা" │
          │ Version: 100                          │
          └─────────────────────────────────────┘
              │                    │
  t=1   Officer A (online)    Officer B (offline!)
         "সংশোধিত নীতি"       "নতুন ধারা যোগ"
              │                    │
  t=2   Version: 105           Local version: 103
              │                    │ (stored in IndexedDB)
  t=3        │              Internet ফিরে এলো!
              │                    │
              ▼                    ▼
          ┌─────────────────────────────────────┐
          │         MERGE PROCESS                │
          │                                      │
          │  1. B sends: "I have ops 101-103     │
          │     based on version 100"            │
          │                                      │
          │  2. Server: "Version 100 → 105      │
          │     has ops [a1, a2, a3, a4, a5]"   │
          │                                      │
          │  3. Transform B's ops against        │
          │     [a1, a2, a3, a4, a5]            │
          │                                      │
          │  4. Apply transformed ops            │
          │     New version: 108                 │
          │                                      │
          │  5. Send transformed ops to B        │
          │     (B updates local state)          │
          │                                      │
          │  Result: Both see consistent doc! ✓  │
          └─────────────────────────────────────┘

Conflict Example:
─────────────────
A edited paragraph 3: "পুরনো text" → "A-এর নতুন text"
B edited paragraph 3: "পুরনো text" → "B-এর নতুন text"

Resolution (OT):
  → A came first (server order) → A's text stays
  → B's op transformed: Insert "B-এর নতুন text" AFTER A's text
  → Final: "A-এর নতুন text\nB-এর নতুন text"
  → UI shows conflict marker → user manually resolves
```

### কেস ৩: Large Document Performance (1000+ Pages)

```
Scenario: একটি আইন firm-এর ১০০০+ পৃষ্ঠার legal document

Problem:
────────
• Full document load = 5MB+ (too slow on BD internet)
• OT on full document = O(n) per operation
• Memory: Full doc in RAM = expensive

Solution: Paginated/Chunked Loading
─────────────────────────────────────

  ┌──────────────────────────────────────────────┐
  │          Virtual Document Architecture        │
  │                                               │
  │  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
  │  │ Chunk 1 │ │ Chunk 2 │ │ Chunk 3 │ ...   │
  │  │ (pg 1-5)│ │(pg 6-10)│ │(pg11-15)│       │
  │  │  50KB   │ │  50KB   │ │  50KB   │       │
  │  └─────────┘ └─────────┘ └─────────┘       │
  │       ▲                                      │
  │       │ Only load visible chunks             │
  │       │ + 1 chunk before/after (buffer)      │
  │                                               │
  │  Viewport: [Chunk 12] [Chunk 13] [Chunk 14]  │
  │  Loaded:   [Chunk 11] ... [Chunk 15] (5 max) │
  │  Rest:     Metadata only (title, page count)  │
  └──────────────────────────────────────────────┘

  OT Optimization:
  ─────────────────
  • Operations only processed for loaded chunks
  • Background ops queued, applied when chunk loads
  • Chunk-level version tracking

  Performance Results:
  ────────────────────
  Initial Load:     200KB (viewport only) vs 5MB (full doc)
  Load Time (4G):   2 sec vs 30+ sec
  Memory Usage:     50MB vs 500MB
  OT Processing:    O(chunk_size) vs O(doc_size)
```

---

## 🔧 Advanced Topics

### ১. Rich Text CRDT (Peritext Algorithm)

```
সমস্যা: Plain text CRDT সহজ, কিন্তু formatting (bold, italic) কীভাবে?

Traditional approach:
  "Hello World" + [bold: 0-5, italic: 6-11]
  ↑ Position-based formatting breaks when text changes!

Peritext approach:
  প্রতিটি format mark anchor to specific characters:

  H  e  l  l  o     W  o  r  l  d
  ▲              ▲   ▲           ▲
  │  bold start  │   │ italic    │
  └──────────────┘   └───────────┘

  Character-level formatting anchors:
  { char: 'H', marks: [{ type: 'bold', action: 'start' }] }
  { char: 'o', marks: [{ type: 'bold', action: 'end' }] }
  { char: 'W', marks: [{ type: 'italic', action: 'start' }] }
  { char: 'd', marks: [{ type: 'italic', action: 'end' }] }

  ✓ Insert between bold chars → automatically bold!
  ✓ Concurrent bold + italic → both apply correctly!
  ✓ Delete formatted char → marks adjust automatically!
```

### ২. Commenting & Suggestion Mode

```php
<?php
// app/Http/Controllers/CommentController.php

namespace App\Http\Controllers;

use App\Models\DocumentComment;
use Illuminate\Http\Request;

class CommentController extends Controller
{
    /**
     * ডকুমেন্টে comment করা
     * Example: শিক্ষক ছাত্রের thesis-এ feedback দিচ্ছেন
     */
    public function store(Request $request, string $documentId)
    {
        $validated = $request->validate([
            'anchor_start' => 'required|integer|min:0',
            'anchor_end' => 'required|integer|min:0',
            'content' => 'required|string|max:2000',
            'type' => 'required|in:comment,suggestion',
            'suggested_text' => 'nullable|required_if:type,suggestion|string',
        ]);

        $comment = DocumentComment::create([
            'document_id' => $documentId,
            'user_id' => auth()->id(),
            'anchor_start' => $validated['anchor_start'],
            'anchor_end' => $validated['anchor_end'],
            'content' => $validated['content'],
            'type' => $validated['type'],
            'suggested_text' => $validated['suggested_text'] ?? null,
            'status' => 'open',
        ]);

        // Real-time broadcast
        \Illuminate\Support\Facades\Redis::publish('doc-events', json_encode([
            'event' => 'comment.created',
            'data' => [
                'document_id' => $documentId,
                'comment' => $comment->load('user'),
            ],
        ]));

        return response()->json($comment->load('user'), 201);
    }

    /**
     * Suggestion accept করা — suggested text apply হবে document-এ
     */
    public function acceptSuggestion(Request $request, string $commentId)
    {
        $comment = DocumentComment::where('type', 'suggestion')
            ->findOrFail($commentId);

        // Apply the suggestion as an operation
        $operation = [
            'type' => 'replace',
            'position' => $comment->anchor_start,
            'length' => $comment->anchor_end - $comment->anchor_start,
            'content' => $comment->suggested_text,
        ];

        // Send to real-time server for OT processing
        \Illuminate\Support\Facades\Redis::publish('doc-events', json_encode([
            'event' => 'suggestion.accepted',
            'data' => [
                'document_id' => $comment->document_id,
                'operation' => $operation,
                'accepted_by' => auth()->id(),
            ],
        ]));

        $comment->update(['status' => 'accepted']);

        return response()->json(['message' => 'Suggestion accepted']);
    }
}
```

### ৩. Export System (PDF, DOCX)

```
Export Architecture:
═══════════════════

  ┌──────────┐     ┌───────────────┐     ┌──────────────┐
  │  Client  │────►│ Laravel API   │────►│ Export Queue  │
  │  Request │     │ /export/pdf   │     │  (Redis)      │
  └──────────┘     └───────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
                                         ┌──────────────┐
                                         │ Export Worker │
                                         │  (Node.js)   │
                                         │              │
                                         │ • Puppeteer  │
                                         │   (PDF)      │
                                         │ • docx lib   │
                                         │   (DOCX)     │
                                         └──────┬───────┘
                                                │
                                                ▼
                                         ┌──────────────┐
                                         │     S3       │
                                         │ (exported    │
                                         │  files)      │
                                         └──────┬───────┘
                                                │
                                                ▼
                                         Webhook/Email
                                         to user with
                                         download link

Bangladesh consideration:
─────────────────────────
• Export হতে সময় লাগলে SMS notification (বাংলালিংক/গ্রামীণফোন)
• Low bandwidth-এ progressive download
• বাংলা font embedding (SolaimanLipi, Kalpurush)
```

### ৪. Access Analytics

```
┌─────────────────────────────────────────────────────────┐
│              Document Analytics Dashboard                 │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  📊 "প্রজেক্ট রিপোর্ট Q4 2024"                         │
│                                                          │
│  Total Views: 245    Unique Editors: 12    Comments: 34  │
│                                                          │
│  Editing Activity (Last 7 days):                         │
│  ┌────────────────────────────────────────┐             │
│  │    ██                                  │             │
│  │    ██  ██                              │             │
│  │    ██  ██      ██                      │             │
│  │ ██ ██  ██  ██  ██  ██                  │             │
│  │ ██ ██  ██  ██  ██  ██  ██             │             │
│  │ শনি রবি সোম মঙ্গল বুধ বৃহ শুক্র      │             │
│  └────────────────────────────────────────┘             │
│                                                          │
│  Top Contributors:                                       │
│  1. রাহুল আহমেদ     — 456 edits (38%)                  │
│  2. সাবরিনা আক্তার  — 312 edits (26%)                  │
│  3. তানভীর হাসান    — 198 edits (17%)                   │
│                                                          │
│  Section Heatmap:                                        │
│  Introduction  ████████░░ (80% complete)                 │
│  Methodology   ██████████ (100% complete)                │
│  Results       ████░░░░░░ (40% complete)                 │
│  Conclusion    ██░░░░░░░░ (20% complete)                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

### Key Design Decisions

```
┌─────────────────────────────────────────────────────────────────┐
│                    Architecture Summary                           │
├────────────────────┬────────────────────────────────────────────┤
│ Sync Protocol      │ CRDT (Yjs) + Central Server for auth       │
│ Transport          │ WebSocket (primary) + HTTP polling (fallback)│
│ Document Store     │ MongoDB (content) + PostgreSQL (metadata)   │
│ Operation Log      │ MongoDB (event sourcing + snapshots)        │
│ Cache/Presence     │ Redis (pub/sub + cursor state)              │
│ Offline Strategy   │ IndexedDB + CRDT merge on reconnect        │
│ Backend API        │ PHP Laravel (CRUD, auth, permissions)       │
│ Real-time Server   │ Node.js Express + Socket.IO                 │
│ Export             │ Queue-based async (Puppeteer/docx)          │
│ Search             │ Elasticsearch (full-text document search)   │
└────────────────────┴────────────────────────────────────────────┘
```

### বাংলাদেশ Context-এ মূল বিবেচনা

```
১. Network Resilience:
   → Offline-first architecture (CRDT)
   → Progressive loading (chunks)
   → Reconnection with automatic merge
   → Low-bandwidth optimization (operation batching, compression)

২. Infrastructure:
   → Dhaka-based edge server (low latency)
   → Singapore fallback (AWS ap-southeast-1)
   → CDN for static assets (CloudFront)

৩. Localization:
   → বাংলা Unicode full support (Avro/Probhat keyboard)
   → Bangla font rendering optimization
   → RTL text mixing support (Arabic + Bangla + English)

৪. Scalability Path:
   → Start: Single Node.js server (10K concurrent users)
   → Scale: Horizontally with Redis pub/sub (100K users)
   → Future: Kubernetes + CRDT P2P (1M+ users)
```

### Production Deployment Checklist

```
□ WebSocket server health check endpoint
□ Auto-scaling based on active connections
□ Rate limiting (max 100 ops/sec per user)
□ Document size limit (50MB)
□ Concurrent editor limit (200 per document)
□ Operation log compaction (daily cron)
□ Version snapshot cleanup (keep last 100)
□ Monitoring: Latency p95, p99 per region
□ Alert: > 100ms operation propagation
□ Backup: Document snapshots to S3 (hourly)
□ DR: Multi-region failover (Dhaka → Singapore)
□ Security: End-to-end encryption for sensitive docs
□ Compliance: Bangladesh Data Protection Act adherence
```

---

> **পরবর্তী পড়ুন**: [Real-time Chat System Design](./real-time-chat-system.md) | [Distributed File Storage](./distributed-file-storage.md)
