# ⏳ Async Request-Response Pattern

## 📋 সুচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [Polling Pattern](#polling-pattern)
- [Callback/Webhook Pattern](#callbackwebhook-pattern)
- [Request-Acknowledge-Poll Pattern](#request-acknowledge-poll-pattern)
- [WebSocket ও SSE](#websocket-ও-sse)
- [Approach তুলনা](#approach-তুলনা)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Async Request-Response Pattern** হলো এমন একটি API design pattern যেখানে client একটি request পাঠায়, server সাথে সাথে acknowledge করে (202 Accepted), এবং পরে result তৈরি হলে client কে জানায় বা client নিজে poll করে result নিয়ে আসে।

### কেন দরকার?

কিছু operation এত সময় নেয় যে synchronous response দেওয়া সম্ভব না:
- 📦 Daraz: ১০,০০০ product bulk import (৫-১০ মিনিট)
- 📊 Prothom Alo: মাসিক analytics report generate (২-৩ মিনিট)
- 💰 bKash: ১ লক্ষ merchant settlement process (১৫-২০ মিনিট)
- 📱 Grameenphone: SIM activation batch processing (১০+ মিনিট)
- 🎥 Video platform: Video encoding/transcoding (৩০+ মিনিট)

### মূল ধারণা:

```
Synchronous (সমস্যা):
─────────────────────
Client ──── Request ────▶ Server
  ⏳                       (কাজ করছে...)
  ⏳                       (এখনো করছে...)
  ⏳                       (৫ মিনিট পার...)
  ⏳                       (timeout! ❌)
Client ◀─── 504 Error ─── Server


Asynchronous (সমাধান):
─────────────────────────
Client ──── Request ────▶ Server
Client ◀─── 202 + Job ID ── Server (তাৎক্ষণিক!)
  │
  │... (অন্য কাজ করছে)
  │
Client ──── Status Check ──▶ Server
Client ◀─── "Processing 60%" ── Server
  │
  │... (আরো অপেক্ষা)
  │
Client ──── Status Check ──▶ Server
Client ◀─── "Done! Here's result" ── Server ✅
```

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 🛍️ Daraz Bulk Product Import

Daraz seller portal এ একজন seller ১০,০০০ product এর CSV file upload করে। এটা synchronous ভাবে process করা অসম্ভব:

```
┌──────────────────────────────────────────────────────────────┐
│              Daraz Bulk Import Workflow                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: Upload                                              │
│  ┌─────────┐     POST /api/imports          ┌─────────┐     │
│  │  Seller │────── (CSV file) ──────────────▶│  Daraz  │     │
│  │  App    │◀──── 202 Accepted ─────────────│  API    │     │
│  └─────────┘      job_id: "JOB-9876"        └─────────┘     │
│       │                                          │           │
│       │                                          ▼           │
│  Step 2: Processing                         ┌─────────┐     │
│       │                                     │  Queue  │     │
│       │                                     │ Worker  │     │
│       │                                     └─────────┘     │
│       │                                          │           │
│  Step 3: Poll Status                             │           │
│       │    GET /api/imports/JOB-9876/status       │           │
│       │──────────────────────────────────────▶   │           │
│       │◀──── { progress: "45%", processed: 4500 }│           │
│       │                                          │           │
│  Step 4: Get Result                              │           │
│       │    GET /api/imports/JOB-9876/result       │           │
│       │──────────────────────────────────────▶   │           │
│       │◀──── { success: 9800, failed: 200 }      │           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 💳 bKash Bulk Settlement

প্রতিদিন রাতে bKash সব merchant এর settlement process করে:

```
Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
12:00 AM  →  Settlement job trigger
12:00 AM  →  202 Accepted, Job ID: SETTLE-2024-01-15
12:05 AM  →  Poll: "Processing... 10% (10,000/100,000 merchants)"
12:15 AM  →  Poll: "Processing... 45% (45,000/100,000 merchants)"
12:30 AM  →  Poll: "Processing... 80% (80,000/100,000 merchants)"
12:45 AM  →  Poll: "Completed ✅ (100,000/100,000 merchants)"
12:45 AM  →  GET result: { total: ৳5,00,00,000, merchants: 100,000 }
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🔄 Polling Pattern

### কিভাবে কাজ করে:

Client নিয়মিত interval এ server কে জিজ্ঞেস করে "কাজ শেষ হয়েছে?"

```
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Server  │
└────┬─────┘                              └────┬─────┘
     │                                         │
     │──── POST /jobs (start job) ────────────▶│
     │◀─── 202 { job_id: "J1" } ──────────────│
     │                                         │
     │  (wait 2 seconds)                       │
     │                                         │
     │──── GET /jobs/J1/status ───────────────▶│
     │◀─── 200 { status: "processing", 20% } ─│
     │                                         │
     │  (wait 2 seconds)                       │
     │                                         │
     │──── GET /jobs/J1/status ───────────────▶│
     │◀─── 200 { status: "processing", 60% } ─│
     │                                         │
     │  (wait 2 seconds)                       │
     │                                         │
     │──── GET /jobs/J1/status ───────────────▶│
     │◀─── 200 { status: "completed" } ───────│
     │                                         │
     │──── GET /jobs/J1/result ───────────────▶│
     │◀─── 200 { data: [...] } ───────────────│
     │                                         │
```

### Polling Strategies:

```
┌─────────────────────────────────────────────────────────────┐
│              Polling Strategies তুলনা                        │
├──────────────┬──────────────────────────────────────────────┤
│              │                                              │
│  Fixed       │  ■ ■ ■ ■ ■ ■ ■ ■ ■ ■                      │
│  Interval    │  (প্রতি 2s এ poll)                           │
│              │  সমস্যা: unnecessary requests                │
│              │                                              │
│  Exponential │  ■  ■   ■     ■         ■                   │
│  Backoff     │  2s 4s  8s   16s       32s                  │
│              │  ভালো: server load কম                        │
│              │                                              │
│  Adaptive    │  ■ ■ ■   ■     ■  ■ ■ ■                    │
│              │  (progress এর উপর ভিত্তি করে)               │
│              │  সবচেয়ে ভালো: smart polling                  │
│              │                                              │
└──────────────┴──────────────────────────────────────────────┘
```

---

## 📞 Callback/Webhook Pattern

### কিভাবে কাজ করে:

Client একটি callback URL দেয়। কাজ শেষ হলে server সেই URL এ POST করে।

```
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Server  │
└────┬─────┘                              └────┬─────┘
     │                                         │
     │── POST /jobs ──────────────────────────▶│
     │   { callback_url: "https://my.app/hook"}│
     │◀── 202 { job_id: "J1" } ───────────────│
     │                                         │
     │   (client অন্য কাজ করছে...)              │──── Processing...
     │                                         │
     │                                         │──── Done! ✅
     │                                         │
     │◀── POST https://my.app/hook ────────────│
     │    { job_id: "J1", status: "done",      │
     │      result: {...} }                    │
     │                                         │
     │── 200 OK (acknowledge) ────────────────▶│
     │                                         │
```

### Webhook vs Polling তুলনা:

```
┌───────────────────────────────────────────────────────────┐
│         Webhook (Push)          │    Polling (Pull)        │
├─────────────────────────────────┼─────────────────────────┤
│ ✅ Real-time notification       │ ❌ Delay (interval)      │
│ ✅ কম network traffic          │ ❌ বেশি request          │
│ ❌ Client এ endpoint লাগে       │ ✅ Client simple         │
│ ❌ Security complex (signing)   │ ✅ Security simple       │
│ ❌ Retry logic দরকার            │ ✅ Retry built-in        │
│ ❌ Firewall issue হতে পারে      │ ✅ Firewall friendly     │
│                                 │                         │
│ Use: bKash payment notification │ Use: Report generation  │
│ Use: Pathao ride updates        │ Use: Bulk imports       │
└─────────────────────────────────┴─────────────────────────┘
```

---

## 🔁 Request-Acknowledge-Poll Pattern

### সম্পূর্ণ Pattern:

```
┌───────────────────────────────────────────────────────────────────┐
│           Request → Acknowledge → Poll → Result                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────┐         ┌─────────┐        ┌────────┐               │
│  │ Client │         │   API   │        │ Worker │               │
│  └───┬────┘         └────┬────┘        └───┬────┘               │
│      │                   │                  │                     │
│      │─── POST /import ─▶│                  │                     │
│      │                   │── Queue Job ────▶│                     │
│      │◀── 202 Accepted ──│                  │                     │
│      │    Location:       │                  │                     │
│      │    /jobs/J1/status │                  │── Processing...    │
│      │                   │                  │                     │
│      │─── GET /jobs/J1 ─▶│                  │                     │
│      │◀── 200 {50%} ────│                  │                     │
│      │                   │                  │                     │
│      │─── GET /jobs/J1 ─▶│                  │── Done! ✅          │
│      │◀── 303 See Other ─│                  │                     │
│      │    Location:       │                  │                     │
│      │    /imports/I1     │                  │                     │
│      │                   │                  │                     │
│      │─── GET /imports/I1▶│                  │                     │
│      │◀── 200 {result}  ─│                  │                     │
│      │                   │                  │                     │
│  ┌───┴────┐         ┌────┴────┐        ┌───┴────┐               │
│  │ Client │         │   API   │        │ Worker │               │
│  └────────┘         └─────────┘        └────────┘               │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### HTTP Status Codes:

```
┌──────────┬────────────────────────────────────────────┐
│  Status  │  অর্থ                                      │
├──────────┼────────────────────────────────────────────┤
│  202     │  Accepted — কাজ গ্রহণ করা হয়েছে            │
│  200     │  OK — status/progress তথ্য                  │
│  303     │  See Other — কাজ শেষ, result অন্যত্র       │
│  410     │  Gone — Job expired, result আর নেই          │
│  409     │  Conflict — Job cancel হয়েছে               │
└──────────┴────────────────────────────────────────────┘
```

---

## 🌐 WebSocket ও SSE

### WebSocket — দ্বিমুখী Real-time Communication:

```
┌──────────┐                              ┌──────────┐
│  Client  │◀═══════ WebSocket ═══════════▶│  Server  │
└────┬─────┘    (persistent connection)    └────┬─────┘
     │                                         │
     │══ { type: "subscribe", job: "J1" } ════▶│
     │                                         │
     │◀═ { type: "progress", value: 20 } ═════│
     │◀═ { type: "progress", value: 50 } ═════│
     │◀═ { type: "progress", value: 80 } ═════│
     │◀═ { type: "complete", result: {...} } ══│
     │                                         │
     │══ { type: "cancel", job: "J2" } ═══════▶│  (client থেকেও পাঠাতে পারে)
     │◀═ { type: "cancelled", job: "J2" } ════│
     │                                         │
```

### Server-Sent Events (SSE) — একমুখী Streaming:

```
┌──────────┐                              ┌──────────┐
│  Client  │────── GET /jobs/J1/stream ──▶│  Server  │
│          │◀═══════════════════════════════│          │
└────┬─────┘   (one-way event stream)      └────┬─────┘
     │                                         │
     │◀═ event: progress                       │
     │   data: {"percent": 20}                 │
     │                                         │
     │◀═ event: progress                       │
     │   data: {"percent": 50}                 │
     │                                         │
     │◀═ event: progress                       │
     │   data: {"percent": 100}                │
     │                                         │
     │◀═ event: complete                       │
     │   data: {"result_url": "/imports/I1"}   │
     │                                         │
```

### তুলনা: WebSocket vs SSE vs Polling

```
┌─────────────────────────────────────────────────────────────────┐
│              Real-time Approaches তুলনা                          │
├──────────────┬─────────────────┬────────────────┬───────────────┤
│   Feature    │    Polling      │     SSE        │  WebSocket    │
├──────────────┼─────────────────┼────────────────┼───────────────┤
│ Direction    │ Client → Server │ Server → Client│ Bidirectional │
│ Connection   │ Multiple HTTP   │ Single HTTP    │ Persistent    │
│ Real-time    │ ❌ Delay        │ ✅ Yes         │ ✅ Yes        │
│ Complexity   │ ⭐ Low          │ ⭐⭐ Medium     │ ⭐⭐⭐ High     │
│ Browser      │ ✅ All          │ ✅ Modern      │ ✅ Modern     │
│ Scalability  │ ⭐ (high load)  │ ⭐⭐⭐ Good     │ ⭐⭐ Medium    │
│ Use Case     │ Simple status   │ Progress/Feed  │ Chat/Gaming   │
│              │ Batch jobs      │ Live updates   │ Real-time app │
├──────────────┼─────────────────┼────────────────┼───────────────┤
│ বাংলাদেশ    │ Daraz import    │ Prothom Alo    │ Pathao ride   │
│ Example      │ status check    │ live news      │ tracking      │
└──────────────┴─────────────────┴────────────────┴───────────────┘
```

---

## 💻 PHP কোড উদাহরণ

### Complete Async Job System (Daraz Bulk Import):

```php
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Str;

/**
 * Daraz Bulk Product Import - Async Request-Response Pattern
 * 
 * Flow: Upload → 202 Accepted → Poll Status → Get Result
 */
class BulkImportController extends Controller
{
    /**
     * Step 1: Submit Import Job
     * POST /api/imports
     * 
     * Response: 202 Accepted + Location header
     */
    public function submitImport(Request $request): JsonResponse
    {
        $request->validate([
            'file' => 'required|file|mimes:csv,xlsx|max:51200', // 50MB max
            'callback_url' => 'nullable|url',
            'options' => 'nullable|array',
        ]);

        // Job তৈরি করা
        $jobId = 'JOB-' . Str::upper(Str::random(12));
        $file = $request->file('file');
        $filePath = $file->store("imports/{$jobId}");

        // Job queue তে পাঠানো
        $job = ImportJob::create([
            'id' => $jobId,
            'seller_id' => $request->user()->id,
            'file_path' => $filePath,
            'callback_url' => $request->input('callback_url'),
            'status' => 'queued',
            'total_items' => $this->countCsvRows($filePath),
            'processed_items' => 0,
            'failed_items' => 0,
            'options' => $request->input('options', []),
            'created_at' => now(),
            'expires_at' => now()->addHours(24),
        ]);

        // Background worker এ dispatch
        ProcessBulkImport::dispatch($job);

        // 202 Accepted — কাজ গ্রহণ করা হয়েছে
        return response()->json([
            'job_id' => $jobId,
            'status' => 'queued',
            'message' => 'আপনার import job গ্রহণ করা হয়েছে। Status check করতে নিচের URL ব্যবহার করুন।',
            'estimated_time' => $this->estimateTime($job->total_items),
            'links' => [
                'status' => "/api/imports/{$jobId}/status",
                'cancel' => "/api/imports/{$jobId}/cancel",
                'result' => "/api/imports/{$jobId}/result",
            ],
        ], 202, [
            'Location' => "/api/imports/{$jobId}/status",
            'Retry-After' => '5', // 5 সেকেন্ড পর poll করুন
        ]);
    }

    /**
     * Step 2: Check Job Status (Polling Endpoint)
     * GET /api/imports/{jobId}/status
     */
    public function checkStatus(string $jobId): JsonResponse
    {
        $job = ImportJob::findOrFail($jobId);

        // Job expire হয়ে গেলে
        if ($job->expires_at < now() && $job->status !== 'completed') {
            return response()->json([
                'job_id' => $jobId,
                'status' => 'expired',
                'message' => 'Job expired. অনুগ্রহ করে আবার submit করুন।',
            ], 410); // 410 Gone
        }

        // Job cancel হয়ে গেলে
        if ($job->status === 'cancelled') {
            return response()->json([
                'job_id' => $jobId,
                'status' => 'cancelled',
                'cancelled_at' => $job->cancelled_at,
                'message' => 'Job বাতিল করা হয়েছে।',
            ], 409); // 409 Conflict
        }

        // Job শেষ হলে — redirect to result
        if ($job->status === 'completed') {
            return response()->json([
                'job_id' => $jobId,
                'status' => 'completed',
                'completed_at' => $job->completed_at,
                'links' => [
                    'result' => "/api/imports/{$jobId}/result",
                ],
            ], 303, [
                'Location' => "/api/imports/{$jobId}/result",
            ]);
        }

        // এখনো processing চলছে
        $progress = $job->total_items > 0 
            ? round(($job->processed_items / $job->total_items) * 100, 1)
            : 0;

        // Adaptive Retry-After: progress বেশি হলে তাড়াতাড়ি poll
        $retryAfter = $progress > 80 ? 2 : ($progress > 50 ? 5 : 10);

        return response()->json([
            'job_id' => $jobId,
            'status' => $job->status, // 'queued', 'processing'
            'progress' => [
                'percent' => $progress,
                'total' => $job->total_items,
                'processed' => $job->processed_items,
                'failed' => $job->failed_items,
                'current_item' => $job->current_item_name,
            ],
            'estimated_remaining' => $this->estimateRemaining($job),
            'started_at' => $job->started_at,
        ], 200, [
            'Retry-After' => (string) $retryAfter,
        ]);
    }

    /**
     * Step 3: Get Job Result
     * GET /api/imports/{jobId}/result
     */
    public function getResult(string $jobId): JsonResponse
    {
        $job = ImportJob::findOrFail($jobId);

        if ($job->status !== 'completed') {
            return response()->json([
                'error' => 'JOB_NOT_COMPLETE',
                'message' => 'Job এখনো শেষ হয়নি। Status endpoint poll করুন।',
                'current_status' => $job->status,
                'links' => ['status' => "/api/imports/{$jobId}/status"],
            ], 409);
        }

        return response()->json([
            'job_id' => $jobId,
            'summary' => [
                'total' => $job->total_items,
                'success' => $job->processed_items - $job->failed_items,
                'failed' => $job->failed_items,
                'duration_seconds' => $job->completed_at->diffInSeconds($job->started_at),
            ],
            'failed_items' => $job->getFailedItemsDetails(), // কোন items fail হয়েছে
            'download_links' => [
                'success_report' => "/api/imports/{$jobId}/report/success",
                'error_report' => "/api/imports/{$jobId}/report/errors",
            ],
            'completed_at' => $job->completed_at,
        ]);
    }

    /**
     * Job বাতিল করা
     * DELETE /api/imports/{jobId}/cancel
     */
    public function cancelJob(string $jobId): JsonResponse
    {
        $job = ImportJob::findOrFail($jobId);

        if (in_array($job->status, ['completed', 'cancelled'])) {
            return response()->json([
                'error' => 'CANNOT_CANCEL',
                'message' => "Job already {$job->status}. বাতিল করা সম্ভব না।",
            ], 409);
        }

        $job->update([
            'status' => 'cancelled',
            'cancelled_at' => now(),
        ]);

        // Worker কে stop signal পাঠানো
        CancelImportJob::dispatch($jobId);

        return response()->json([
            'job_id' => $jobId,
            'status' => 'cancelled',
            'message' => 'Job সফলভাবে বাতিল করা হয়েছে।',
            'processed_before_cancel' => $job->processed_items,
        ]);
    }

    /**
     * SSE Endpoint - Real-time Progress Stream
     * GET /api/imports/{jobId}/stream
     */
    public function streamProgress(string $jobId)
    {
        $job = ImportJob::findOrFail($jobId);

        return response()->stream(function () use ($jobId) {
            $lastProgress = -1;

            while (true) {
                $job = ImportJob::find($jobId);

                if (!$job) {
                    echo "event: error\n";
                    echo "data: {\"message\": \"Job not found\"}\n\n";
                    break;
                }

                $progress = $job->total_items > 0
                    ? round(($job->processed_items / $job->total_items) * 100)
                    : 0;

                // শুধু progress পরিবর্তন হলেই পাঠাও
                if ($progress !== $lastProgress) {
                    echo "event: progress\n";
                    echo "data: " . json_encode([
                        'percent' => $progress,
                        'processed' => $job->processed_items,
                        'total' => $job->total_items,
                    ]) . "\n\n";
                    $lastProgress = $progress;
                }

                if ($job->status === 'completed') {
                    echo "event: complete\n";
                    echo "data: " . json_encode([
                        'result_url' => "/api/imports/{$jobId}/result",
                    ]) . "\n\n";
                    break;
                }

                if ($job->status === 'failed') {
                    echo "event: error\n";
                    echo "data: " . json_encode([
                        'message' => $job->error_message,
                    ]) . "\n\n";
                    break;
                }

                ob_flush();
                flush();
                sleep(1);
            }
        }, 200, [
            'Content-Type' => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'Connection' => 'keep-alive',
            'X-Accel-Buffering' => 'no',
        ]);
    }

    private function countCsvRows(string $path): int
    {
        $file = new \SplFileObject(storage_path("app/{$path}"));
        $file->seek(PHP_INT_MAX);
        return $file->key(); // header বাদে
    }

    private function estimateTime(int $totalItems): string
    {
        $seconds = ceil($totalItems / 100) * 2; // ~100 items/2sec
        if ($seconds < 60) return "{$seconds} সেকেন্ড";
        $minutes = ceil($seconds / 60);
        return "{$minutes} মিনিট";
    }

    private function estimateRemaining($job): string
    {
        if ($job->processed_items === 0) return 'গণনা করা হচ্ছে...';

        $elapsed = now()->diffInSeconds($job->started_at);
        $rate = $job->processed_items / max($elapsed, 1);
        $remaining = ($job->total_items - $job->processed_items) / max($rate, 0.01);

        $minutes = ceil($remaining / 60);
        return "আনুমানিক {$minutes} মিনিট বাকি";
    }
}

/**
 * Background Worker - Actual Import Processing
 */
class ProcessBulkImport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $timeout = 3600; // 1 hour max
    public int $tries = 3;

    public function __construct(private ImportJob $job) {}

    public function handle(): void
    {
        $this->job->update(['status' => 'processing', 'started_at' => now()]);

        $csvData = $this->parseCsv($this->job->file_path);

        foreach ($csvData as $index => $row) {
            // Cancel check — প্রতি 100 row এ check
            if ($index % 100 === 0 && $this->isCancelled()) {
                return;
            }

            try {
                $this->processRow($row);
                $this->job->increment('processed_items');
            } catch (\Exception $e) {
                $this->job->increment('failed_items');
                $this->logFailedItem($index, $row, $e->getMessage());
            }

            // Progress update — প্রতি 50 row এ
            if ($index % 50 === 0) {
                $this->job->update(['current_item_name' => $row['name'] ?? "Row {$index}"]);
            }
        }

        $this->job->update([
            'status' => 'completed',
            'completed_at' => now(),
        ]);

        // Callback URL থাকলে notify করা
        if ($this->job->callback_url) {
            $this->sendWebhookNotification();
        }
    }

    private function isCancelled(): bool
    {
        return ImportJob::find($this->job->id)?->status === 'cancelled';
    }

    private function sendWebhookNotification(): void
    {
        Http::post($this->job->callback_url, [
            'event' => 'import.completed',
            'job_id' => $this->job->id,
            'summary' => [
                'total' => $this->job->total_items,
                'success' => $this->job->processed_items - $this->job->failed_items,
                'failed' => $this->job->failed_items,
            ],
            'result_url' => "/api/imports/{$this->job->id}/result",
            'timestamp' => now()->toISOString(),
        ]);
    }

    /**
     * Timeout Handling
     */
    public function failed(\Throwable $exception): void
    {
        $this->job->update([
            'status' => 'failed',
            'error_message' => $exception->getMessage(),
            'failed_at' => now(),
        ]);

        // Alert engineering team
        logger()->critical('Bulk import failed', [
            'job_id' => $this->job->id,
            'error' => $exception->getMessage(),
            'processed' => $this->job->processed_items,
            'total' => $this->job->total_items,
        ]);
    }

    private function parseCsv(string $path): array { return []; }
    private function processRow(array $row): void {}
    private function logFailedItem(int $index, array $row, string $error): void {}
}
```

---

## 🟨 JavaScript কোড উদাহরণ

### Async Job Client (Consumer Side):

```javascript
/**
 * Daraz Bulk Import Client
 * Async Request-Response Pattern Implementation
 */
class AsyncJobClient {
    constructor(baseUrl, apiKey) {
        this.baseUrl = baseUrl;
        this.apiKey = apiKey;
        this.defaultPollInterval = 5000; // 5 seconds
        this.maxPollAttempts = 360;      // 30 minutes max (5s × 360)
    }

    /**
     * Step 1: Job Submit করা
     * File upload → 202 Accepted → Job ID পাওয়া
     */
    async submitImport(file, options = {}) {
        const formData = new FormData();
        formData.append('file', file);
        formData.append('callback_url', options.callbackUrl || '');
        formData.append('options', JSON.stringify(options));

        const response = await fetch(`${this.baseUrl}/api/imports`, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
            },
            body: formData,
        });

        if (response.status !== 202) {
            const error = await response.json();
            throw new Error(`Import submission failed: ${error.message}`);
        }

        const data = await response.json();
        console.log(`📦 Import job submitted: ${data.job_id}`);
        console.log(`⏱️ Estimated time: ${data.estimated_time}`);

        return data;
    }

    /**
     * Step 2: Polling দিয়ে Status Check
     * Exponential Backoff + Adaptive Retry
     */
    async pollUntilComplete(jobId, options = {}) {
        const {
            onProgress = null,
            maxAttempts = this.maxPollAttempts,
            initialInterval = this.defaultPollInterval,
            abortSignal = null,
        } = options;

        let attempts = 0;
        let interval = initialInterval;

        while (attempts < maxAttempts) {
            // Cancellation check
            if (abortSignal?.aborted) {
                await this.cancelJob(jobId);
                throw new Error('Import cancelled by user');
            }

            attempts++;
            const statusResponse = await this.checkStatus(jobId);

            // Progress callback
            if (onProgress && statusResponse.progress) {
                onProgress(statusResponse.progress);
            }

            // Job শেষ হয়েছে!
            if (statusResponse.status === 'completed') {
                console.log(`✅ Import completed! (${attempts} polls)`);
                return await this.getResult(jobId);
            }

            // Error states
            if (statusResponse.status === 'failed') {
                throw new Error(`Import failed: ${statusResponse.error}`);
            }

            if (statusResponse.status === 'expired') {
                throw new Error('Import job expired');
            }

            if (statusResponse.status === 'cancelled') {
                throw new Error('Import was cancelled');
            }

            // Adaptive interval — Retry-After header ব্যবহার
            const retryAfter = statusResponse.retryAfter;
            if (retryAfter) {
                interval = retryAfter * 1000;
            } else {
                // Exponential backoff (max 30 seconds)
                interval = Math.min(interval * 1.5, 30000);
            }

            // অপেক্ষা...
            await this.sleep(interval);
        }

        throw new Error(`Polling timeout after ${maxAttempts} attempts`);
    }

    /**
     * Status Check (Single Poll)
     */
    async checkStatus(jobId) {
        const response = await fetch(`${this.baseUrl}/api/imports/${jobId}/status`, {
            headers: { 'Authorization': `Bearer ${this.apiKey}` },
        });

        const data = await response.json();

        return {
            ...data,
            retryAfter: response.headers.get('Retry-After')
                ? parseInt(response.headers.get('Retry-After'))
                : null,
        };
    }

    /**
     * Result নিয়ে আসা
     */
    async getResult(jobId) {
        const response = await fetch(`${this.baseUrl}/api/imports/${jobId}/result`, {
            headers: { 'Authorization': `Bearer ${this.apiKey}` },
        });

        if (!response.ok) {
            throw new Error('Failed to fetch result');
        }

        return await response.json();
    }

    /**
     * Job Cancel করা
     */
    async cancelJob(jobId) {
        const response = await fetch(`${this.baseUrl}/api/imports/${jobId}/cancel`, {
            method: 'DELETE',
            headers: { 'Authorization': `Bearer ${this.apiKey}` },
        });

        return await response.json();
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

/**
 * SSE (Server-Sent Events) Client
 * Real-time progress updates via event stream
 */
class SSEProgressClient {
    constructor(baseUrl, apiKey) {
        this.baseUrl = baseUrl;
        this.apiKey = apiKey;
    }

    /**
     * SSE দিয়ে real-time progress দেখা
     * Prothom Alo live news feed style
     */
    streamProgress(jobId, callbacks = {}) {
        const {
            onProgress = () => {},
            onComplete = () => {},
            onError = () => {},
        } = callbacks;

        const eventSource = new EventSource(
            `${this.baseUrl}/api/imports/${jobId}/stream`
        );

        eventSource.addEventListener('progress', (event) => {
            const data = JSON.parse(event.data);
            onProgress(data);
            console.log(`📊 Progress: ${data.percent}% (${data.processed}/${data.total})`);
        });

        eventSource.addEventListener('complete', (event) => {
            const data = JSON.parse(event.data);
            onComplete(data);
            console.log('✅ Complete!', data);
            eventSource.close();
        });

        eventSource.addEventListener('error', (event) => {
            if (event.data) {
                const data = JSON.parse(event.data);
                onError(data);
            }
            console.error('❌ Error occurred');
            eventSource.close();
        });

        eventSource.onerror = () => {
            console.warn('⚠️ SSE connection lost. Reconnecting...');
            // EventSource auto-reconnects
        };

        // Cleanup function return
        return () => {
            eventSource.close();
            console.log('🔌 SSE connection closed');
        };
    }
}

/**
 * WebSocket Client for Bidirectional Communication
 * Pathao ride tracking style
 */
class WebSocketJobClient {
    constructor(wsUrl) {
        this.wsUrl = wsUrl;
        this.ws = null;
        this.callbacks = new Map();
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
    }

    connect() {
        return new Promise((resolve, reject) => {
            this.ws = new WebSocket(this.wsUrl);

            this.ws.onopen = () => {
                console.log('🔌 WebSocket connected');
                this.reconnectAttempts = 0;
                resolve();
            };

            this.ws.onmessage = (event) => {
                const message = JSON.parse(event.data);
                this.handleMessage(message);
            };

            this.ws.onclose = () => {
                console.log('🔌 WebSocket disconnected');
                this.attemptReconnect();
            };

            this.ws.onerror = (error) => {
                console.error('❌ WebSocket error:', error);
                reject(error);
            };
        });
    }

    /**
     * Job subscribe করা — progress updates পেতে
     */
    subscribeToJob(jobId, callbacks = {}) {
        this.callbacks.set(jobId, callbacks);

        this.ws.send(JSON.stringify({
            type: 'subscribe',
            job_id: jobId,
        }));

        console.log(`👀 Subscribed to job: ${jobId}`);
    }

    /**
     * Job cancel command পাঠানো
     */
    cancelJob(jobId) {
        this.ws.send(JSON.stringify({
            type: 'cancel',
            job_id: jobId,
        }));
    }

    handleMessage(message) {
        const { type, job_id, ...data } = message;
        const callbacks = this.callbacks.get(job_id);

        if (!callbacks) return;

        switch (type) {
            case 'progress':
                callbacks.onProgress?.(data);
                break;
            case 'complete':
                callbacks.onComplete?.(data);
                this.callbacks.delete(job_id);
                break;
            case 'error':
                callbacks.onError?.(data);
                this.callbacks.delete(job_id);
                break;
            case 'cancelled':
                callbacks.onCancelled?.(data);
                this.callbacks.delete(job_id);
                break;
        }
    }

    attemptReconnect() {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error('❌ Max reconnect attempts reached');
            return;
        }

        this.reconnectAttempts++;
        const delay = Math.pow(2, this.reconnectAttempts) * 1000;
        console.log(`🔄 Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);

        setTimeout(() => this.connect(), delay);
    }

    disconnect() {
        this.ws?.close();
    }
}

// ═══════════════════════════════════════════════════════
// ব্যবহার উদাহরণ: Daraz Seller Portal
// ═══════════════════════════════════════════════════════

async function darazBulkImportExample() {
    const client = new AsyncJobClient('https://api.daraz.com.bd', 'seller-api-key');

    // File upload
    const fileInput = document.getElementById('csv-file');
    const file = fileInput.files[0];

    try {
        // 1️⃣ Submit import
        console.log('📤 Uploading file...');
        const { job_id } = await client.submitImport(file, {
            callbackUrl: 'https://my-store.com/webhooks/import-done',
        });

        // 2️⃣ Poll with progress UI
        console.log('⏳ Processing...');
        const abortController = new AbortController();

        // Cancel button
        document.getElementById('cancel-btn').onclick = () => {
            abortController.abort();
        };

        const result = await client.pollUntilComplete(job_id, {
            abortSignal: abortController.signal,
            onProgress: (progress) => {
                // Progress bar update
                const bar = document.getElementById('progress-bar');
                bar.style.width = `${progress.percent}%`;
                bar.textContent = `${progress.percent}% (${progress.processed}/${progress.total})`;

                console.log(
                    `📊 ${progress.percent}% — ` +
                    `✅ ${progress.processed - (progress.failed || 0)} success, ` +
                    `❌ ${progress.failed || 0} failed`
                );
            },
        });

        // 3️⃣ Result দেখানো
        console.log('🎉 Import complete!');
        console.log(`   ✅ Success: ${result.summary.success}`);
        console.log(`   ❌ Failed: ${result.summary.failed}`);
        console.log(`   ⏱️ Duration: ${result.summary.duration_seconds}s`);

        if (result.summary.failed > 0) {
            console.log(`   📋 Error report: ${result.download_links.error_report}`);
        }

    } catch (error) {
        console.error('❌ Import failed:', error.message);
    }
}

// ═══════════════════════════════════════════════════════
// SSE ব্যবহার উদাহরণ: Real-time Progress
// ═══════════════════════════════════════════════════════

function sseProgressExample(jobId) {
    const sseClient = new SSEProgressClient('https://api.daraz.com.bd', 'key');

    const cleanup = sseClient.streamProgress(jobId, {
        onProgress: ({ percent, processed, total }) => {
            document.getElementById('progress').textContent = 
                `${percent}% (${processed}/${total} products)`;
        },
        onComplete: ({ result_url }) => {
            document.getElementById('status').textContent = '✅ সম্পন্ন!';
            window.location.href = result_url;
        },
        onError: ({ message }) => {
            document.getElementById('status').textContent = `❌ ত্রুটি: ${message}`;
        },
    });

    // Page leave এ cleanup
    window.addEventListener('beforeunload', cleanup);
}

// ═══════════════════════════════════════════════════════
// WebSocket ব্যবহার: Pathao Ride Tracking Style
// ═══════════════════════════════════════════════════════

async function websocketExample(jobId) {
    const wsClient = new WebSocketJobClient('wss://ws.daraz.com.bd/jobs');
    await wsClient.connect();

    wsClient.subscribeToJob(jobId, {
        onProgress: ({ percent, current_item }) => {
            console.log(`📊 ${percent}% — Processing: ${current_item}`);
        },
        onComplete: ({ result_url }) => {
            console.log(`✅ Done! Result: ${result_url}`);
            wsClient.disconnect();
        },
        onError: ({ message }) => {
            console.error(`❌ Error: ${message}`);
            wsClient.disconnect();
        },
        onCancelled: () => {
            console.log('🚫 Job cancelled');
            wsClient.disconnect();
        },
    });
}
```

---

## ✅ কখন ব্যবহার করবেন / করবেন না

### ✅ কখন Async Pattern ব্যবহার করবেন:

| পরিস্থিতি | উদাহরণ | Approach |
|-----------|--------|----------|
| Long-running task (>5s) | Daraz bulk import | Polling |
| File processing | Video encoding, PDF generation | Polling + SSE |
| Batch operations | bKash bulk settlement | Webhook |
| Report generation | Prothom Alo analytics | Polling |
| Third-party dependent | Payment gateway, SMS | Webhook |
| Real-time tracking | Pathao ride | WebSocket |
| Live feed | Prothom Alo breaking news | SSE |

### ❌ কখন Async ব্যবহার করবেন না:

| পরিস্থিতি | কারণ | বিকল্প |
|-----------|------|--------|
| Fast operations (<1s) | Overhead বেশি | Synchronous |
| Simple CRUD | Complex করার দরকার নেই | REST sync |
| User expects instant result | UX খারাপ হবে | Sync + cache |
| Low traffic internal API | Complexity বেশি | Simple sync |

### 🎯 কোন Approach কখন:

```
┌─────────────────────────────────────────────────────────────┐
│              Async Approach Selection Guide                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Decision Tree:                                             │
│  ──────────────                                             │
│                                                             │
│  Long-running operation?                                    │
│  ├── NO → Synchronous API ব্যবহার করুন                     │
│  └── YES                                                    │
│       ├── Client একটি server?                               │
│       │   ├── YES → Webhook/Callback ব্যবহার করুন          │
│       │   └── NO (browser/mobile)                           │
│       │        ├── Real-time updates দরকার?                 │
│       │        │   ├── YES                                  │
│       │        │   │   ├── Bidirectional? → WebSocket       │
│       │        │   │   └── One-way? → SSE                  │
│       │        │   └── NO → Polling                        │
│       │        └── Progress bar দরকার?                      │
│       │            ├── YES → SSE বা Polling (short)        │
│       │            └── NO → Simple Polling                 │
│       └── Cancel support দরকার?                             │
│           ├── YES → WebSocket বা Polling + DELETE           │
│           └── NO → যেকোনো approach                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### ⚖️ Best Practices:

```
┌─────────────────────────────────────────────────────────────┐
│              Async API Best Practices                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ DO:                                                     │
│  • 202 Accepted + Location header ব্যবহার করুন             │
│  • Retry-After header দিন                                   │
│  • Job expiration/TTL সেট করুন                              │
│  • Cancellation support রাখুন                               │
│  • Progress percentage দিন                                  │
│  • Idempotency key support করুন                             │
│  • Error details সংরক্ষণ করুন                               │
│  • Rate limit polling requests                              │
│  • Timeout clearly define করুন                              │
│                                                             │
│  ❌ DON'T:                                                  │
│  • Client কে indefinitely poll করতে দেবেন না               │
│  • Result চিরকাল store করবেন না (TTL দিন)                  │
│  • Progress ছাড়া long operation চালাবেন না                  │
│  • Single point of failure রাখবেন না                        │
│  • Cancel ছাড়া long job শুরু করবেন না                      │
│  • Webhook retry ছাড়া callback পাঠাবেন না                  │
│  • Error state handle না করে রাখবেন না                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 সংক্ষিপ্ত রেফারেন্স

| বিষয় | মনে রাখার কথা |
|-------|---------------|
| 202 Accepted | "কাজ নিয়েছি, পরে দেব" |
| Location Header | "এখানে status দেখো" |
| Retry-After | "এত সময় পর আবার এসো" |
| Polling | Client বারবার জিজ্ঞেস করে |
| Webhook | Server নিজে জানায় |
| SSE | Server একদিকে stream করে |
| WebSocket | দুইদিকেই কথা বলা যায় |
| Job Cancellation | DELETE endpoint রাখা দরকার |
| Idempotency | একই request দুইবার এলেও সমস্যা নেই |
| Timeout | সব job এর সময়সীমা থাকা দরকার |

---

*"Async pattern হলো বলা — 'আমি কাজটা নিলাম, শেষ হলে জানাবো!' — ঠিক যেমন bKash payment notification কাজ করে।"* 🚀
