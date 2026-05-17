# ☁️ ক্লাউড স্টোরেজ সিস্টেম ডিজাইন (Google Drive/Dropbox)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Google Drive, Dropbox, OneDrive, iCloud
> **বাংলাদেশ কনটেক্সট**: সরকারি ডিজিটাল ফাইল স্টোরেজ, বিশ্ববিদ্যালয় ডকুমেন্ট শেয়ারিং

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| 1 | File Upload/Download | যেকোনো সাইজের ফাইল আপলোড ও ডাউনলোড |
| 2 | Folder Structure | হায়ারার্কিক্যাল ফোল্ডার ব্যবস্থাপনা |
| 3 | Sharing & Permissions | ফাইল/ফোল্ডার শেয়ার করা (view, edit, comment) |
| 4 | Cross-device Sync | সব ডিভাইসে real-time সিঙ্ক |
| 5 | File Versioning | পূর্ববর্তী ভার্সনে ফিরে যাওয়া |
| 6 | Search | ফাইলের নাম ও কন্টেন্ট দিয়ে সার্চ |
| 7 | Offline Support | ইন্টারনেট ছাড়াই ফাইল এক্সেস |
| 8 | Trash/Recovery | ডিলিট করা ফাইল রিকভার করা |

### Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────┐
│  Non-Functional Requirements                            │
├─────────────────────────────────────────────────────────┤
│  • Durability: 99.999999999% (11 nines)                 │
│  • Availability: 99.99% uptime                          │
│  • Resumable Upload: নেটওয়ার্ক ড্রপে আবার শুরু         │
│  • Low Latency Sync: < 5s change propagation            │
│  • Scalability: 500M+ users support                     │
│  • Security: End-to-end encryption                      │
│  • Bandwidth Optimization: Delta sync, compression      │
└─────────────────────────────────────────────────────────┘
```

### 🇧🇩 বাংলাদেশ কনটেক্সট

**বাস্তব সমস্যাগুলো:**

1. **ধীর ইন্টারনেট**: গ্রামীণ এলাকায় 2G/3G — বড় ফাইল আপলোডে সমস্যা
2. **সরকারি ডিজিটাল বাংলাদেশ**: a2i প্রকল্পে লাখ লাখ সরকারি ডকুমেন্ট ডিজিটাইজেশন
3. **বিশ্ববিদ্যালয় শেয়ারিং**: ঢাবি, বুয়েট-এর শিক্ষার্থীদের অ্যাসাইনমেন্ট সাবমিশন
4. **ব্যাংকিং ডকুমেন্ট**: Bangladesh Bank এর regulatory documents
5. **Load Shedding**: পাওয়ার কাটে upload resume করতে হবে
6. **Data Residency**: বাংলাদেশের ডেটা বাংলাদেশেই রাখার নিয়ম

---

## 📊 Back-of-envelope Estimation

### ইউজার ও স্টোরেজ ক্যালকুলেশন

```
মোট ইউজার:              500 Million (Global scale)
DAU (Daily Active Users): 100 Million
প্রতি ইউজার স্টোরেজ:    15 GB (free tier)
মোট স্টোরেজ:            500M × 15 GB = 7.5 Exabytes

বাংলাদেশ স্কেল (ধরা যাক):
- ইউজার: 5 Million
- DAU: 1 Million
- প্রতি ইউজার: 5 GB avg
- মোট: 25 PB
```

### ট্রাফিক Estimation

```
Average file size:           1 MB
Uploads per user per day:    2 files
Total uploads/day:           100M × 2 = 200M files
Upload throughput:           200M × 1MB / 86400s ≈ 2.3 GB/s

Read:Write ratio:            3:1
Downloads/day:               600M files
Download throughput:         ≈ 7 GB/s

Peak traffic (3x average):  Upload: ~7 GB/s, Download: ~21 GB/s
```

### স্টোরেজ ব্রেকডাউন

```
┌────────────────────────────────────────────┐
│  Storage Type      │ Amount    │ Cost/mo   │
├────────────────────┼───────────┼───────────┤
│  Object Storage    │ 7.5 EB   │ ~$150M    │
│  Metadata DB       │ 500 TB   │ ~$2M      │
│  Cache (Redis)     │ 50 TB    │ ~$5M      │
│  CDN Bandwidth     │ 21 GB/s  │ ~$30M     │
└────────────────────────────────────────────┘
```

---

## 🏗️ High-Level Design

### সিস্টেম আর্কিটেকচার ডায়াগ্রাম

```
                        ┌─────────────────┐
                        │   Load Balancer  │
                        │   (Nginx/HAProxy)│
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
    ┌─────────▼──────┐  ┌───────▼───────┐  ┌──────▼──────────┐
    │  API Gateway   │  │  Sync Service │  │  Notification   │
    │  (Laravel)     │  │  (Node.js)    │  │  Service        │
    └───────┬────────┘  └───────┬───────┘  └──────┬──────────┘
            │                   │                  │
    ┌───────▼────────┐  ┌──────▼────────┐         │
    │  Block Server  │  │  Change       │         │
    │  (Chunking +   │  │  Detection    │         │
    │   Dedup)       │  │  Engine       │         │
    └───────┬────────┘  └──────┬────────┘         │
            │                  │                   │
    ┌───────▼──────────────────▼───────────────────▼────┐
    │              Message Queue (RabbitMQ/Kafka)         │
    └───────┬──────────────────┬───────────────────┬────┘
            │                  │                   │
    ┌───────▼──────┐  ┌───────▼───────┐  ┌───────▼───────┐
    │  Object      │  │  Metadata DB  │  │  Search       │
    │  Storage     │  │  (PostgreSQL) │  │  (Elastic-    │
    │  (S3/MinIO)  │  │               │  │   search)     │
    └──────────────┘  └───────────────┘  └───────────────┘
```

### ক্লায়েন্ট আর্কিটেকচার

```
┌─────────────────────────────────────────────────────┐
│                  Client Application                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐ │
│  │ Watcher  │  │ Chunker   │  │ Local Database   │ │
│  │ (File    │→→│ (Split +  │→→│ (SQLite -        │ │
│  │  System) │  │  Hash)    │  │  file metadata)  │ │
│  └──────────┘  └───────────┘  └──────────────────┘ │
│       │              │                    │          │
│       ▼              ▼                    ▼          │
│  ┌──────────────────────────────────────────────┐   │
│  │         Sync Engine (Conflict Resolver)       │   │
│  └──────────────────────┬───────────────────────┘   │
│                         │                            │
└─────────────────────────┼────────────────────────────┘
                          │ HTTPS/WebSocket
                          ▼
                    ┌─────────────┐
                    │   Server    │
                    └─────────────┘
```

### মূল কম্পোনেন্ট ব্যাখ্যা

| কম্পোনেন্ট | দায়িত্ব |
|------------|---------|
| API Gateway | Authentication, rate limiting, routing |
| Block Server | ফাইলকে chunks এ ভাগ করা, deduplication |
| Sync Service | ক্লায়েন্টদের মধ্যে changes সিঙ্ক করা |
| Notification Service | Real-time change alerts (WebSocket) |
| Object Storage | Actual file blocks সংরক্ষণ |
| Metadata DB | ফাইল/ফোল্ডার তথ্য, permissions |
| Message Queue | Async processing, event distribution |

---

## 💻 Detailed Design

### 1️⃣ Block-Level Sync (চাঙ্কিং ও ডিডুপ্লিকেশন)

**কেন Block-level?**
- পুরো ফাইল না পাঠিয়ে শুধু পরিবর্তিত অংশ পাঠানো
- বাংলাদেশের slow internet-এ bandwidth সাশ্রয়
- Deduplication — একই content দুবার store না করা

```
┌─────────────────────────────────────────────────────────┐
│  Original File (10 MB)                                   │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐   │
│  │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │ B8 │ B9 │B10│   │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘   │
│       Each block = 1 MB, SHA-256 hash computed           │
│                                                          │
│  Modified File (block 3 & 7 changed):                    │
│  ┌────┬────┬════┬────┬────┬────┬════┬────┬────┬────┐   │
│  │ B1 │ B2 │ B3'│ B4 │ B5 │ B6 │ B7'│ B8 │ B9 │B10│   │
│  └────┴────┴════┴────┴────┴────┴════┴────┴────┴────┘   │
│       Only B3' and B7' uploaded (2 MB instead of 10 MB)  │
└─────────────────────────────────────────────────────────┘
```

**Rolling Hash (Rabin fingerprint) ব্যবহার:**

```
Content-Defined Chunking:
─────────────────────────
ফাইলের উপর sliding window চালানো হয়।
যখন hash % chunk_size == magic_number,
তখন chunk boundary তৈরি হয়।

সুবিধা: ফাইলের মাঝে ডেটা insert করলেও
        বাকি chunks unchanged থাকে।
```

### 2️⃣ File Versioning & Conflict Resolution

```
Version Tree:
─────────────
                    v1 (original)
                     │
                    v2 (edit by user A)
                   / \
                  /   \
          v3 (user A)  v3' (user B) ← CONFLICT!
                  \   /
                   \ /
                   v4 (merged / user decides)
```

**Conflict Resolution Strategy:**

```
┌────────────────────────────────────────────────────┐
│  Last-Writer-Wins (LWW):                           │
│    - সবচেয়ে সাম্প্রতিক edit রাখা হয়               │
│    - অন্য version "conflicted copy" হিসেবে save    │
│                                                    │
│  Dropbox approach:                                 │
│    - "Abir's conflicted copy (2024-01-15)"         │
│    - User manually resolve করে                    │
│                                                    │
│  Google Docs approach:                             │
│    - Operational Transformation (OT)               │
│    - Real-time collaborative editing               │
│    - No conflicts — concurrent edits merged        │
└────────────────────────────────────────────────────┘
```

### 3️⃣ Sharing & Permission Model (ACL)

```
Permission Hierarchy:
─────────────────────
Owner > Editor > Commenter > Viewer

┌─────────────────────────────────────────────────┐
│  File: "thesis_final.docx"                      │
│  Owner: rahim@buet.ac.bd                        │
│                                                  │
│  Shared with:                                    │
│  ├── karim@buet.ac.bd    → Editor               │
│  ├── dept_cs@buet.ac.bd  → Viewer (group)       │
│  └── Anyone with link    → Viewer               │
│                                                  │
│  Inheritance:                                    │
│  /Research/                                      │
│    ├── /Papers/         ← inherits from parent   │
│    │   └── thesis.docx  ← own + inherited ACL   │
│    └── /Data/           ← inherits from parent   │
└─────────────────────────────────────────────────┘
```

### 4️⃣ Chunked Upload System (PHP Laravel)

```php
<?php
// app/Http/Controllers/FileUploadController.php

namespace App\Http\Controllers;

use App\Models\FileUpload;
use App\Models\FileBlock;
use App\Services\BlockStorageService;
use App\Services\DeduplicationService;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class FileUploadController extends Controller
{
    private BlockStorageService $blockStorage;
    private DeduplicationService $dedup;

    public function __construct(
        BlockStorageService $blockStorage,
        DeduplicationService $dedup
    ) {
        $this->blockStorage = $blockStorage;
        $this->dedup = $dedup;
    }

    /**
     * Upload শুরু করা — pre-signed URL generate করে
     * বাংলাদেশের slow internet-এ resumable upload essential
     */
    public function initiateUpload(Request $request)
    {
        $request->validate([
            'filename'    => 'required|string|max:255',
            'file_size'   => 'required|integer|min:1',
            'mime_type'   => 'required|string',
            'block_hashes'=> 'required|array',  // Client-side computed SHA-256
            'folder_id'   => 'nullable|uuid',
        ]);

        $userId = auth()->id();
        $blockSize = $this->calculateBlockSize($request->file_size);
        $totalBlocks = ceil($request->file_size / $blockSize);

        // Deduplication check — কোন blocks আগে থেকে আছে?
        $existingBlocks = $this->dedup->findExistingBlocks(
            $request->block_hashes
        );

        // Upload session তৈরি করা
        $upload = FileUpload::create([
            'id'            => Str::uuid(),
            'user_id'       => $userId,
            'filename'      => $request->filename,
            'file_size'     => $request->file_size,
            'mime_type'     => $request->mime_type,
            'block_size'    => $blockSize,
            'total_blocks'  => $totalBlocks,
            'uploaded_blocks'=> count($existingBlocks),
            'status'        => 'in_progress',
            'folder_id'     => $request->folder_id,
            'expires_at'    => now()->addHours(24),
        ]);

        // যে blocks upload করতে হবে তাদের pre-signed URL
        $blocksToUpload = [];
        foreach ($request->block_hashes as $index => $hash) {
            if (in_array($hash, $existingBlocks)) {
                // এই block আগে থেকেই আছে — skip!
                // বাংলাদেশে bandwidth বাঁচাবে
                FileBlock::create([
                    'upload_id'   => $upload->id,
                    'block_index' => $index,
                    'block_hash'  => $hash,
                    'status'      => 'deduplicated',
                ]);
                continue;
            }

            $presignedUrl = $this->blockStorage->generatePresignedUrl(
                bucket: 'file-blocks',
                key: "blocks/{$hash}",
                expiry: 3600
            );

            $blocksToUpload[] = [
                'block_index'   => $index,
                'block_hash'    => $hash,
                'upload_url'    => $presignedUrl,
                'expected_size' => min(
                    $blockSize,
                    $request->file_size - ($index * $blockSize)
                ),
            ];

            FileBlock::create([
                'upload_id'   => $upload->id,
                'block_index' => $index,
                'block_hash'  => $hash,
                'status'      => 'pending',
            ]);
        }

        return response()->json([
            'upload_id'       => $upload->id,
            'block_size'      => $blockSize,
            'blocks_to_upload'=> $blocksToUpload,
            'blocks_skipped'  => count($existingBlocks),
            'bandwidth_saved' => count($existingBlocks) * $blockSize,
            'message'         => count($existingBlocks) > 0
                ? "Dedup: {$existingBlocks} blocks already exist!"
                : "All blocks need upload",
        ], 201);
    }

    /**
     * Individual block upload complete callback
     */
    public function completeBlock(Request $request, string $uploadId)
    {
        $request->validate([
            'block_index' => 'required|integer',
            'block_hash'  => 'required|string|size:64',
            'checksum'    => 'required|string', // MD5 verification
        ]);

        $block = FileBlock::where('upload_id', $uploadId)
            ->where('block_index', $request->block_index)
            ->firstOrFail();

        // Integrity verification
        $storedChecksum = $this->blockStorage->getObjectChecksum(
            "blocks/{$request->block_hash}"
        );

        if ($storedChecksum !== $request->checksum) {
            $block->update(['status' => 'failed']);
            return response()->json([
                'error' => 'Checksum mismatch — re-upload needed',
                'retry_url' => $this->blockStorage->generatePresignedUrl(
                    'file-blocks', "blocks/{$request->block_hash}", 3600
                ),
            ], 409);
        }

        $block->update(['status' => 'completed']);

        // সব blocks complete হলে file assemble করা
        $upload = FileUpload::find($uploadId);
        $completedCount = $upload->blocks()
            ->whereIn('status', ['completed', 'deduplicated'])
            ->count();

        if ($completedCount >= $upload->total_blocks) {
            $this->assembleFile($upload);
        }

        return response()->json([
            'status'    => 'block_received',
            'progress'  => round(($completedCount / $upload->total_blocks) * 100, 1),
            'remaining' => $upload->total_blocks - $completedCount,
        ]);
    }

    /**
     * Block size নির্ধারণ — নেটওয়ার্ক condition অনুযায়ী
     * বাংলাদেশের 2G/3G তে ছোট blocks ব্যবহার করা হয়
     */
    private function calculateBlockSize(int $fileSize): int
    {
        if ($fileSize < 1024 * 1024) {          // < 1 MB
            return $fileSize;                    // Single block
        } elseif ($fileSize < 100 * 1024 * 1024) { // < 100 MB
            return 4 * 1024 * 1024;              // 4 MB blocks
        } else {                                  // >= 100 MB
            return 8 * 1024 * 1024;              // 8 MB blocks
        }
    }

    private function assembleFile(FileUpload $upload): void
    {
        // File metadata entry তৈরি করা
        $upload->update(['status' => 'completed']);

        // Event fire — sync service কে notify করা
        event(new \App\Events\FileUploaded($upload));
    }
}
```

### 5️⃣ Sync Service (Node.js Express)

```javascript
// src/services/syncService.js

const express = require('express');
const WebSocket = require('ws');
const { createClient } = require('redis');
const { Pool } = require('pg');
const crypto = require('crypto');

const app = express();
const wss = new WebSocket.Server({ noServer: true });

// PostgreSQL connection
const db = new Pool({
  host: process.env.DB_HOST,
  database: 'cloud_storage',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

// Redis for pub/sub (change notifications)
const redisSubscriber = createClient({ url: process.env.REDIS_URL });
const redisPublisher = createClient({ url: process.env.REDIS_URL });

/**
 * Change Detection Engine
 * ক্লায়েন্ট থেকে local changes receive করে
 * এবং server-side changes broadcast করে
 */
class SyncEngine {
  constructor() {
    this.connectedClients = new Map(); // userId -> Set<WebSocket>
  }

  /**
   * ক্লায়েন্ট থেকে sync request handle করা
   * Client পাঠায়: তার local state এর cursor/timestamp
   * Server পাঠায়: সেই timestamp এর পরের সব changes
   */
  async getChangesSince(userId, cursor) {
    const query = `
      SELECT
        c.id,
        c.file_id,
        c.change_type,
        c.block_hashes,
        c.metadata,
        c.timestamp,
        f.path,
        f.filename
      FROM file_changes c
      JOIN files f ON c.file_id = f.id
      WHERE f.user_id = $1
        AND c.timestamp > $2
      ORDER BY c.timestamp ASC
      LIMIT 1000
    `;

    const result = await db.query(query, [userId, cursor]);

    return {
      changes: result.rows.map(row => ({
        id: row.id,
        fileId: row.file_id,
        path: row.path,
        filename: row.filename,
        type: row.change_type, // 'create', 'modify', 'delete', 'move'
        blockHashes: row.block_hashes,
        metadata: row.metadata,
        timestamp: row.timestamp,
      })),
      newCursor: result.rows.length > 0
        ? result.rows[result.rows.length - 1].timestamp
        : cursor,
      hasMore: result.rows.length === 1000,
    };
  }

  /**
   * ক্লায়েন্ট থেকে change push করা
   * Conflict detection সহ
   */
  async pushChanges(userId, changes) {
    const results = [];

    for (const change of changes) {
      try {
        // Conflict check: server-এ এই file কি ইতিমধ্যে modify হয়েছে?
        const conflict = await this.checkConflict(
          change.fileId,
          change.baseVersion
        );

        if (conflict) {
          results.push({
            fileId: change.fileId,
            status: 'conflict',
            serverVersion: conflict.version,
            resolution: 'create_conflicted_copy',
            conflictedPath: this.generateConflictName(
              change.path, userId
            ),
          });
          continue;
        }

        // Change apply করা
        const applied = await this.applyChange(userId, change);
        results.push({
          fileId: change.fileId,
          status: 'applied',
          newVersion: applied.version,
          timestamp: applied.timestamp,
        });

        // অন্য connected clients কে notify করা
        await this.notifyOtherDevices(userId, change);

        // Shared users কে notify করা
        await this.notifySharedUsers(change.fileId, userId);

      } catch (error) {
        results.push({
          fileId: change.fileId,
          status: 'error',
          message: error.message,
          retryable: true,
        });
      }
    }

    return results;
  }

  /**
   * Conflict detection
   * Optimistic locking ব্যবহার করা হয়
   */
  async checkConflict(fileId, baseVersion) {
    const result = await db.query(
      'SELECT version, updated_at FROM files WHERE id = $1',
      [fileId]
    );

    if (result.rows.length === 0) return null;

    const serverVersion = result.rows[0].version;
    if (serverVersion > baseVersion) {
      return { version: serverVersion };
    }
    return null;
  }

  /**
   * বাংলাদেশী style conflict naming
   * "report.pdf" → "report (Rahim-এর conflicted copy 2024-01-15).pdf"
   */
  generateConflictName(originalPath, userId) {
    const ext = originalPath.split('.').pop();
    const base = originalPath.replace(`.${ext}`, '');
    const date = new Date().toISOString().split('T')[0];
    return `${base} (conflicted copy ${date}).${ext}`;
  }

  /**
   * WebSocket দিয়ে real-time notification
   * বাংলাদেশের unstable connection-এ reconnection logic
   */
  async notifyOtherDevices(userId, change) {
    const clients = this.connectedClients.get(userId) || new Set();

    const notification = JSON.stringify({
      type: 'file_change',
      data: {
        fileId: change.fileId,
        path: change.path,
        changeType: change.type,
        timestamp: Date.now(),
      },
    });

    for (const ws of clients) {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(notification);
      }
    }

    // Redis pub/sub — অন্য server instances কে জানানো
    await redisPublisher.publish(
      `sync:user:${userId}`,
      notification
    );
  }

  async notifySharedUsers(fileId, excludeUserId) {
    const sharedUsers = await db.query(`
      SELECT DISTINCT user_id FROM file_permissions
      WHERE file_id = $1 AND user_id != $2
    `, [fileId, excludeUserId]);

    for (const row of sharedUsers.rows) {
      await redisPublisher.publish(
        `sync:user:${row.user_id}`,
        JSON.stringify({
          type: 'shared_file_change',
          data: { fileId, changedBy: excludeUserId },
        })
      );
    }
  }
}

// WebSocket connection handling
const syncEngine = new SyncEngine();

wss.on('connection', (ws, req) => {
  const userId = req.userId; // Authenticated via middleware

  // Client register করা
  if (!syncEngine.connectedClients.has(userId)) {
    syncEngine.connectedClients.set(userId, new Set());
  }
  syncEngine.connectedClients.get(userId).add(ws);

  // Heartbeat — বাংলাদেশের unstable connection detect করতে
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });

  ws.on('message', async (message) => {
    const data = JSON.parse(message);

    switch (data.type) {
      case 'sync_request':
        const changes = await syncEngine.getChangesSince(
          userId, data.cursor
        );
        ws.send(JSON.stringify({ type: 'sync_response', ...changes }));
        break;

      case 'push_changes':
        const results = await syncEngine.pushChanges(
          userId, data.changes
        );
        ws.send(JSON.stringify({ type: 'push_response', results }));
        break;

      case 'ping':
        ws.send(JSON.stringify({ type: 'pong' }));
        break;
    }
  });

  ws.on('close', () => {
    syncEngine.connectedClients.get(userId)?.delete(ws);
  });
});

// Heartbeat interval — dead connections cleanup
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

// REST API endpoints
app.post('/api/sync/changes', async (req, res) => {
  const { cursor } = req.body;
  const userId = req.user.id;

  const changes = await syncEngine.getChangesSince(userId, cursor);
  res.json(changes);
});

app.post('/api/sync/push', async (req, res) => {
  const { changes } = req.body;
  const userId = req.user.id;

  const results = await syncEngine.pushChanges(userId, changes);
  res.json({ results });
});

module.exports = { app, wss, syncEngine };
```

### 6️⃣ Database Schema

```sql
-- File Metadata (PostgreSQL)
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    parent_folder_id UUID REFERENCES files(id),
    filename VARCHAR(255) NOT NULL,
    path TEXT NOT NULL,              -- /documents/thesis/final.pdf
    mime_type VARCHAR(100),
    file_size BIGINT NOT NULL DEFAULT 0,
    is_folder BOOLEAN DEFAULT FALSE,
    version INTEGER DEFAULT 1,
    block_list JSONB,               -- ordered list of block hashes
    checksum VARCHAR(64),           -- SHA-256 of complete file
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    -- Index for fast path lookups
    CONSTRAINT unique_path_per_user UNIQUE (user_id, path)
);

CREATE INDEX idx_files_parent ON files(parent_folder_id);
CREATE INDEX idx_files_user_folder ON files(user_id, parent_folder_id);
CREATE INDEX idx_files_updated ON files(updated_at);

-- File versions history
CREATE TABLE file_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID NOT NULL REFERENCES files(id),
    version INTEGER NOT NULL,
    block_list JSONB NOT NULL,
    file_size BIGINT,
    modified_by UUID REFERENCES users(id),
    change_description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),

    UNIQUE(file_id, version)
);

-- Block references (content-addressable)
CREATE TABLE blocks (
    hash VARCHAR(64) PRIMARY KEY,   -- SHA-256 hash = address
    size INTEGER NOT NULL,
    reference_count INTEGER DEFAULT 1,
    storage_path TEXT NOT NULL,      -- S3 path
    created_at TIMESTAMP DEFAULT NOW()
);

-- Sharing & Permissions
CREATE TABLE file_permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID NOT NULL REFERENCES files(id),
    user_id UUID REFERENCES users(id),
    email VARCHAR(255),              -- invite before signup
    permission_level VARCHAR(20) NOT NULL, -- owner/editor/commenter/viewer
    shared_by UUID REFERENCES users(id),
    link_sharing BOOLEAN DEFAULT FALSE,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Change log for sync
CREATE TABLE file_changes (
    id BIGSERIAL PRIMARY KEY,
    file_id UUID NOT NULL REFERENCES files(id),
    user_id UUID NOT NULL,
    change_type VARCHAR(20) NOT NULL, -- create/modify/delete/move/rename
    block_hashes JSONB,
    metadata JSONB,
    previous_version INTEGER,
    new_version INTEGER,
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_changes_user_time ON file_changes(user_id, timestamp);
```

### 7️⃣ Upload Flow — Resumable Upload

```
Resumable Upload Flow (বাংলাদেশের slow internet-এ জরুরি):
═══════════════════════════════════════════════════════════

Client                     Server                    S3
  │                          │                        │
  │─── 1. Initiate Upload ──→│                        │
  │    (filename, size,      │                        │
  │     block hashes)        │                        │
  │                          │                        │
  │←── 2. Upload Plan ───────│                        │
  │    (which blocks needed, │                        │
  │     pre-signed URLs,     │                        │
  │     deduplicated blocks) │                        │
  │                          │                        │
  │─── 3. Upload Block 1 ────────────────────────────→│
  │←── 4. ACK Block 1 ──────────────────────────────── │
  │                          │                        │
  │─── 5. Upload Block 2 ────────────────────────────→│
  │    ╳╳╳ NETWORK DROP ╳╳╳  │                        │
  │                          │                        │
  │ (reconnect after 30s)    │                        │
  │                          │                        │
  │─── 6. Resume Upload ────→│                        │
  │    (upload_id)           │                        │
  │                          │                        │
  │←── 7. Resume Info ───────│                        │
  │    (block 2 incomplete,  │                        │
  │     blocks 3-10 pending) │                        │
  │                          │                        │
  │─── 8. Re-upload Block 2 ─────────────────────────→│
  │←── 9. ACK ───────────────────────────────────────── │
  │         ...              │                        │
  │─── 10. All blocks done ──→│                        │
  │                          │── Assemble file ──→    │
  │←── 11. Upload Complete ──│                        │
```

### 8️⃣ Download & CDN Strategy

```
Download Flow:
══════════════

Small Files (< 10 MB):
  Client → CDN Edge (Dhaka POP) → Origin (Singapore)
  - Single GET request
  - Cache-Control: public, max-age=3600

Large Files (> 10 MB):
  Client → CDN → Parallel Range Requests
  - GET /file?range=0-4194304        (Block 1: 0-4MB)
  - GET /file?range=4194305-8388608  (Block 2: 4-8MB)
  - Client reassembles blocks locally

বাংলাদেশ CDN Strategy:
┌─────────────────────────────────────────────┐
│  Edge Locations:                            │
│  ├── Dhaka (Primary — lowest latency)       │
│  ├── Singapore (Regional origin)            │
│  └── Mumbai (Fallback)                      │
│                                             │
│  Strategy:                                  │
│  - Hot files: Dhaka edge cache              │
│  - Warm files: Singapore                    │
│  - Cold files: S3 directly (higher latency) │
└─────────────────────────────────────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Block-level vs File-level Sync

```
┌────────────────────┬──────────────────┬──────────────────┐
│                    │  Block-level     │  File-level      │
├────────────────────┼──────────────────┼──────────────────┤
│ Bandwidth          │ ✅ শুধু changed   │ ❌ পুরো file      │
│                    │    blocks         │    re-upload     │
├────────────────────┼──────────────────┼──────────────────┤
│ Complexity         │ ❌ জটিল (chunking│ ✅ সহজ           │
│                    │    dedup, merge)  │    implementation│
├────────────────────┼──────────────────┼──────────────────┤
│ Deduplication      │ ✅ Cross-file     │ ❌ File-level     │
│                    │    dedup possible │    only          │
├────────────────────┼──────────────────┼──────────────────┤
│ BD Internet        │ ✅ Slow connection│ ❌ বড় file       │
│                    │    এ অনেক ভালো   │    timeout হবে   │
├────────────────────┼──────────────────┼──────────────────┤
│ Decision           │ ✅ CHOSEN         │                  │
│                    │ (Dropbox style)  │                  │
└────────────────────┴──────────────────┴──────────────────┘
```

### 2. Strong vs Eventual Consistency (Metadata)

```
┌────────────────────┬──────────────────┬──────────────────┐
│                    │  Strong          │  Eventual        │
├────────────────────┼──────────────────┼──────────────────┤
│ Correctness        │ ✅ সবসময় latest  │ ❌ Stale data     │
│                    │    data দেখায়    │    দেখতে পারে    │
├────────────────────┼──────────────────┼──────────────────┤
│ Latency            │ ❌ Higher         │ ✅ Lower          │
│                    │    (quorum read) │    (any replica) │
├────────────────────┼──────────────────┼──────────────────┤
│ Availability       │ ❌ Partition এ    │ ✅ Always         │
│                    │    unavailable   │    available     │
├────────────────────┼──────────────────┼──────────────────┤
│ Use Case           │ Permission       │ File listing,    │
│                    │ checks, billing  │ search index     │
├────────────────────┼──────────────────┼──────────────────┤
│ Decision           │ Metadata writes  │ Read replicas,   │
│                    │ → Strong         │ search → Eventual│
└────────────────────┴──────────────────┴──────────────────┘
```

### 3. Push vs Pull for Sync Notification

```
┌────────────────────┬──────────────────┬──────────────────┐
│                    │  Push (WebSocket)│  Pull (Polling)  │
├────────────────────┼──────────────────┼──────────────────┤
│ Latency            │ ✅ Instant        │ ❌ Polling        │
│                    │    notification  │    interval delay│
├────────────────────┼──────────────────┼──────────────────┤
│ Server Load        │ ❌ Connection     │ ✅ Stateless      │
│                    │    state maintain│                  │
├────────────────────┼──────────────────┼──────────────────┤
│ BD Mobile          │ ❌ Connection     │ ✅ Works with     │
│                    │    drops বেশি    │    intermittent  │
├────────────────────┼──────────────────┼──────────────────┤
│ Decision           │ Desktop/WiFi     │ Mobile/2G        │
│ (Hybrid)           │ → WebSocket      │ → Long polling   │
└────────────────────┴──────────────────┴──────────────────┘
```

### 4. S3 vs Custom Object Store

```
┌────────────────────┬──────────────────┬──────────────────┐
│                    │  AWS S3          │  Custom (MinIO)  │
├────────────────────┼──────────────────┼──────────────────┤
│ Durability         │ ✅ 11 nines       │ ⚠️ Self-managed  │
│ Cost (1 PB)        │ ❌ ~$23K/month   │ ✅ ~$8K/month    │
│ BD Data Residency  │ ❌ Nearest: Mumbai│ ✅ Local DC       │
│ Ops Complexity     │ ✅ Managed        │ ❌ Team needed    │
├────────────────────┼──────────────────┼──────────────────┤
│ Decision           │ Global users     │ BD compliance    │
│                    │ → S3             │ → MinIO in BD DC │
└────────────────────┴──────────────────┴──────────────────┘
```

---

## 📈 কেস স্টাডি

### কেস ১: বাংলাদেশের Slow Internet-এ Large File Upload

**সমস্যা:** রহিম সাহেব, সিলেটের একটি কলেজের শিক্ষক। তিনি ৫০০ MB-এর একটি ভিডিও লেকচার আপলোড করতে চান। তার ইন্টারনেট speed 512 Kbps।

```
Traditional approach:
  500 MB / 512 Kbps = ~2.2 hours continuous upload
  Network drop → Start from 0 → FRUSTRATED USER

Our approach (Block-level + Resumable):
  ┌──────────────────────────────────────────────────┐
  │  Step 1: File → 125 blocks × 4 MB              │
  │  Step 2: Upload block by block                   │
  │                                                  │
  │  Block 1 ████████████ ✓ (65s)                   │
  │  Block 2 ████████████ ✓ (65s)                   │
  │  Block 3 ████████╳ NETWORK DROP!                │
  │                                                  │
  │  ... (30 seconds later, reconnect) ...           │
  │                                                  │
  │  Block 3 ████████████ ✓ (resume from byte 3MB)  │
  │  Block 4 ████████████ ✓                         │
  │  ...                                             │
  │  Block 125 ████████████ ✓                       │
  │                                                  │
  │  Result: Total time ~2.5 hours (with drops)     │
  │  But: NO data loss, NO restart needed!           │
  └──────────────────────────────────────────────────┘
```

**Adaptive Block Size:**
```
Connection Speed → Block Size
< 256 Kbps      → 512 KB (2G তে ব্যবহৃত)
256-1024 Kbps   → 2 MB
1-5 Mbps        → 4 MB
> 5 Mbps        → 8 MB

বাংলাদেশের average mobile: 1-3 Mbps → 4 MB blocks
```

### কেস ২: বুয়েট CSE Department — Team Collaboration

**সমস্যা:** ৫ জন ছাত্র একই LaTeX thesis file-এ কাজ করছে। দুজন simultaneously edit করে।

```
Timeline:
─────────
09:00 - Karim opens thesis.tex (v5)
09:05 - Rahim opens thesis.tex (v5)
09:10 - Karim saves changes (v5 → v6) ✓
09:12 - Rahim saves changes (based on v5) → CONFLICT!

Resolution:
───────────
Server detects: Rahim's base version (5) < current (6)

Option A (Dropbox style):
  - Rahim's version saved as:
    "thesis (Rahim's conflicted copy 2024-01-15).tex"
  - Rahim notified to manually merge

Option B (Google Docs style):
  - Operational Transformation applied
  - Both edits merged automatically
  - Both users see real-time changes

আমাদের system:
  - Binary files → Option A (conflicted copy)
  - Text files → Option B (OT-based merge)
  - User gets notification: "Conflict detected! Review changes."
```

### কেস ৩: Deduplication Savings — Bangladesh Government

**সিনারিও:** Digital Bangladesh project — ১ লাখ সরকারি অফিসের ডকুমেন্ট ক্লাউডে

```
Without Deduplication:
  - NID form template: 500 KB × 100,000 offices = 50 GB
  - Same circular to all offices: 2 MB × 100,000 = 200 GB
  - Common letterhead images: 1 MB × 100,000 = 100 GB
  Total wasted: ~350 GB for IDENTICAL content

With Block-level Deduplication:
  - NID form template: stored ONCE = 500 KB
  - Circular: stored ONCE = 2 MB
  - Letterhead: stored ONCE = 1 MB
  Total: ~3.5 MB (saved 99.999%!)

Real-world dedup ratio (typical cloud storage):
  ├── Enterprise: 60-70% dedup ratio
  ├── Government: 70-80% (lots of templates)
  └── University: 50-60% (assignments, textbooks)

Annual savings for 25 PB storage:
  Without dedup: 25 PB × $23/TB/month = $575K/month
  With 60% dedup: 10 PB × $23/TB/month = $230K/month
  Savings: $345K/month = $4.1M/year ≈ ৪৫ কোটি টাকা/বছর!
```

---

## 🔧 Advanced Topics

### 1. Content-Addressable Storage (CAS)

```
Traditional:                   Content-Addressable:
─────────────                  ─────────────────────
Path-based addressing          Hash-based addressing
/users/1/docs/file.pdf         SHA-256(content) = abc123...

Store: PUT /blocks/abc123...   (hash IS the address)
Read:  GET /blocks/abc123...

সুবিধা:
├── Automatic dedup (same hash = same content)
├── Integrity verification built-in
├── Immutable (content never changes at an address)
└── Perfect for versioning (each version = new hash tree)

Block Storage Layout:
┌─────────────────────────────────────────────┐
│  /blocks/                                    │
│    ├── ab/                                   │
│    │   ├── abc123def456...  (4 MB block)    │
│    │   └── ab98765432...    (4 MB block)    │
│    ├── cd/                                   │
│    │   └── cd112233...      (4 MB block)    │
│    └── ...                                   │
│                                              │
│  2-character prefix = 256 subdirectories     │
│  Prevents too many files in one directory    │
└─────────────────────────────────────────────┘
```

### 2. Encryption Strategy

```
Encryption Layers:
══════════════════

┌───────────────────────────────────────────────┐
│  Layer 1: In Transit (TLS 1.3)               │
│  Client ←──── HTTPS ────→ Server             │
│                                               │
│  Layer 2: At Rest (AES-256-GCM)             │
│  Server ──→ Encrypt ──→ S3                   │
│                                               │
│  Layer 3: Client-side (Optional, Premium)    │
│  Client ──→ Encrypt ──→ Server (zero-knowledge)│
└───────────────────────────────────────────────┘

Key Management:
  ┌──────────────┐     ┌───────────────┐
  │ Master Key   │────→│ KMS (AWS/HSM) │
  │ (per tenant) │     └───────────────┘
  └──────┬───────┘
         │ derives
  ┌──────▼───────┐
  │ File Key     │ (unique per file, encrypted by master)
  │ (DEK)        │
  └──────┬───────┘
         │ encrypts
  ┌──────▼───────┐
  │ File Blocks  │
  └──────────────┘

বাংলাদেশ Compliance (BCC/BTRC):
- Data at rest: AES-256 mandatory
- Key storage: Hardware Security Module (HSM)
- Audit log: সব access log 5 বছর রাখতে হবে
```

### 3. Smart Sync (On-Demand Files)

```
Smart Sync Concept:
═══════════════════

Desktop এ সব file download না করে placeholder রাখা।
User click করলে তখন download হবে।

File States:
┌─────────┬──────────────────────────────────────┐
│  Icon   │  State                               │
├─────────┼──────────────────────────────────────┤
│  ☁️     │  Cloud-only (placeholder, 0 bytes)   │
│  ✓      │  Downloaded (locally available)       │
│  📌     │  Pinned (always keep offline)         │
└─────────┴──────────────────────────────────────┘

বাংলাদেশে বিশেষভাবে useful:
- Laptop-এ 128 GB SSD common
- Cloud-এ 1 TB data থাকতে পারে
- Smart sync ছাড়া impossible!

Implementation (FUSE/Virtual filesystem):
  User sees: /CloudDrive/videos/lecture.mp4 (500 MB)
  Actually:  Placeholder file (few KB metadata)
  On access: Download triggered → file materialized
```

### 4. Ransomware Detection

```
Ransomware Detection Signals:
═════════════════════════════

┌─────────────────────────────────────────────────┐
│  Signal                  │ Threshold            │
├──────────────────────────┼──────────────────────┤
│  Mass file rename        │ > 50 files/minute    │
│  Extension change        │ .docx → .encrypted   │
│  Entropy spike           │ > 7.5 bits/byte      │
│  Rapid file modification │ > 100 files/5min     │
│  All blocks changing     │ > 90% blocks new     │
└──────────────────────────┴──────────────────────┘

Detection Flow:
  Normal activity → Monitor → OK
  Suspicious activity → ALERT → Pause sync → Notify admin
  Confirmed ransomware → Rollback to last safe version

Protection:
  ├── Immutable snapshots (every 4 hours)
  ├── Anomaly detection ML model
  ├── User notification (SMS/email)
  └── Auto-rollback option
```

### 5. Data Residency & Compliance

```
Bangladesh Data Residency:
══════════════════════════

┌─────────────────────────────────────────────────┐
│  বাংলাদেশ সরকারের নিয়ম (ICT Division):        │
│                                                  │
│  ১. সরকারি ডেটা বাংলাদেশে রাখতে হবে            │
│  ২. Cross-border transfer-এ অনুমতি লাগবে       │
│  ৩. Encryption mandatory                        │
│  ৪. Annual security audit                       │
│                                                  │
│  Solution Architecture:                         │
│                                                  │
│  Primary: Dhaka DC (GP/Banglalink DC)           │
│  DR: Chittagong DC                              │
│  Global CDN: Only cached, not stored            │
│                                                  │
│  Metadata: Always in BD                         │
│  Blocks: BD primary, Singapore DR (encrypted)   │
└─────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

### মূল শিক্ষা (Key Takeaways)

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Block-level Sync = Bandwidth সাশ্রয়ের মূল চাবিকাঠি     │
│     → বাংলাদেশের slow internet-এ essential                  │
│                                                              │
│  2. Content-Addressable Storage = Dedup + Integrity          │
│     → ৬০-৮০% storage savings possible                       │
│                                                              │
│  3. Resumable Upload = User experience এর জন্য mandatory    │
│     → Load shedding / network drop handle করতে পারে        │
│                                                              │
│  4. Hybrid Sync (WebSocket + Long Polling)                   │
│     → Connection quality অনুযায়ী adapt করা                 │
│                                                              │
│  5. Conflict Resolution = UX decision                        │
│     → Technical solution আছে, কিন্তু user experience        │
│       design আরো গুরুত্বপূর্ণ                                │
│                                                              │
│  6. Security = Multi-layer encryption + Compliance           │
│     → Bangladesh specific rules মানতে হবে                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Interview Tips

```
System Design Interview-এ এই topic আসলে:

1. Start with Requirements (5 min)
   - Scale clarify করুন
   - "Google Drive for Bangladesh" বললে constraints ভিন্ন

2. High-level Design (10 min)
   - Client → Block Server → Storage
   - Sync Service + Notification

3. Deep Dive (15 min)
   - Block-level sync explain করুন
   - Deduplication এর savings calculate করুন
   - Conflict resolution strategy বলুন

4. Scale & Trade-offs (10 min)
   - Consistency vs Availability
   - Push vs Pull notification
   - CDN strategy for specific region

সাধারণ ভুল:
  ❌ File-level sync বলা (interviewer disappointed হবে)
  ❌ Conflict resolution skip করা
  ❌ Security/encryption ভুলে যাওয়া
  ❌ Resumable upload mention না করা
```

### Quick Reference — Technology Stack

```
┌──────────────────────────────────────────────────┐
│  Component          │  Technology                │
├─────────────────────┼────────────────────────────┤
│  API Server         │  PHP Laravel               │
│  Sync Service       │  Node.js (Express + WS)    │
│  Object Storage     │  AWS S3 / MinIO            │
│  Metadata DB        │  PostgreSQL (+ read replicas)│
│  Cache              │  Redis Cluster             │
│  Message Queue      │  RabbitMQ / Kafka          │
│  Search             │  Elasticsearch             │
│  CDN                │  CloudFront / Cloudflare   │
│  Monitoring         │  Prometheus + Grafana      │
│  Client Sync DB     │  SQLite                    │
│  Block Hashing      │  SHA-256 + Rabin fingerprint│
│  Encryption         │  AES-256-GCM + TLS 1.3    │
└──────────────────────┴────────────────────────────┘
```

---

> **📝 নোট**: এই ডিজাইন production-ready নয় — এটি একটি শিক্ষামূলক case study। বাস্তবে আরো অনেক edge case handle করতে হয় (quota management, abuse detection, GDPR compliance, multi-region replication, etc.)।

> **🔗 আরো পড়ুন**: Dropbox Tech Blog, Google Drive Architecture Paper, Building a Distributed File Sync System (MIT)
