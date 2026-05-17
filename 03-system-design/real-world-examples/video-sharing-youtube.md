# 🎥 ভিডিও শেয়ারিং প্ল্যাটফর্ম সিস্টেম ডিজাইন (YouTube)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: YouTube, TikTok, Vimeo, Dailymotion, Chorki, Hoichoi

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
| 1 | Video Upload | যেকোনো format-এ video upload (MP4, AVI, MOV, MKV) |
| 2 | Transcoding | Multiple resolution-এ convert (144p → 4K) |
| 3 | Streaming | HLS/DASH adaptive bitrate streaming |
| 4 | Search | Title, description, tags দিয়ে video search |
| 5 | Recommendation | Personalized video suggestions |
| 6 | Like/Comment | Engagement system (like, dislike, comment, reply) |
| 7 | Channel/Subscribe | Creator channels এবং subscription system |
| 8 | Watch History | User-এর দেখা video-র history track করা |
| 9 | Playlist | Custom playlist create ও manage করা |
| 10 | Notifications | New upload, live stream alerts |

### Non-Functional Requirements

- **Scalability**: প্রতি মিনিটে ৫০০+ ঘন্টার video upload handle করা
- **Availability**: 99.99% uptime (yearly ~52 min downtime)
- **Low Latency**: Video playback start < 2 seconds
- **Adaptive Bitrate**: Network condition অনুযায়ী quality adjust
- **Global CDN**: বিশ্বব্যাপী low-latency delivery
- **Consistency**: Eventually consistent (comments, likes)
- **Durability**: কোনো video কখনো lost হবে না (11 nines durability)

### 🇧🇩 Bangladesh Context

```
বাংলাদেশ-নির্দিষ্ট চ্যালেঞ্জ:
├── 📱 Mobile-first users (৮০%+ mobile traffic)
├── 🌐 Low bandwidth areas (গ্রামে 2G/3G connection)
├── 💰 Data cost sensitivity (১ GB = ৫০-১০০ টাকা)
├── 🏏 Cricket streaming peaks (BPL, World Cup)
├── 📺 Bangla content creators (Salman Muqtadir, Tawhid Afridi)
├── 🎬 Local OTT competition (Chorki, Hoichoi, Bongo)
└── ⚡ Load shedding (offline download গুরুত্বপূর্ণ)
```

**Bangladesh-specific optimizations:**
- 144p/240p default quality for 2G/3G users
- Data saver mode (compressed thumbnails, lower bitrate)
- Offline download feature (load shedding-এর সময় দেখার জন্য)
- Bangla language UI ও voice search
- Local CDN edge servers (Dhaka, Chittagong)
- BKash/Nagad payment integration for premium content

---

## 📊 Back-of-envelope Estimation

### Traffic Estimation

```
Daily Active Users (DAU):        ২ billion (globally), ৫০ million (Bangladesh)
Videos uploaded per minute:      ৫০০ hours
Average video duration:          ৭ minutes
Average video size (raw):        ৫০০ MB (1080p, 7 min)
Average video size (compressed): ১৫০ MB (multiple resolutions combined)

Read:Write ratio:                ১০০:১ (heavily read-heavy)
```

### Storage Estimation

```
প্রতিদিন upload:
= ৫০০ hours/min × ৬০ min × ২৪ hours
= ৭২০,০০০ hours/day

Storage per hour (all resolutions): ~২ GB
Daily new storage = ৭২০,০০০ × ২ GB = ১.৪৪ PB/day

৫ বছরে total storage:
= ১.৪৪ PB × ৩৬৫ × ৫ = ~২.৬ EB (Exabytes)
```

### Bandwidth Estimation

```
Peak concurrent viewers:          ১০০ million
Average bitrate (adaptive):       ৫ Mbps
Peak bandwidth = ১০০M × ৫ Mbps = ৫০০ Tbps

Bangladesh peak (cricket match):
= ১০ million × ২ Mbps = ২০ Tbps
```

### Server Estimation

```
Upload servers:     ৫,০০০+ (globally distributed)
Transcoding:        ৫০,০০০+ GPU instances
CDN edge servers:   ২০০,০০০+ (১৫০+ countries)
API servers:        ১০,০০০+
Database servers:   ৫,০০০+ (sharded)
```

---

## 🏗️ High-Level Design

### System Architecture Overview

```
+------------------------------------------------------------------+
|                        CLIENT LAYER                                |
|  [Mobile App]  [Web Browser]  [Smart TV]  [Gaming Console]       |
+------------------------------------------------------------------+
         |                    |                    |
         v                    v                    v
+------------------------------------------------------------------+
|                      LOAD BALANCER (L7)                           |
|              [AWS ALB / Nginx / HAProxy]                          |
+------------------------------------------------------------------+
         |                              |
         v                              v
+------------------+          +--------------------+
|   UPLOAD PATH    |          |   STREAMING PATH   |
+------------------+          +--------------------+
         |                              |
         v                              v
+------------------+          +--------------------+
| Upload Service   |          |  CDN Edge Server   |
| (Pre-signed URL) |          |  (Akamai/CF)      |
+------------------+          +--------------------+
         |                              |
         v                              v
+------------------+          +--------------------+
| Object Storage   |<-------->| Origin Server      |
| (S3/GCS)        |          | (Video Segments)   |
+------------------+          +--------------------+
         |
         v
+------------------+
| Message Queue    |
| (Kafka/SQS)     |
+------------------+
         |
         v
+------------------+
| Transcoding      |
| Pipeline (FFmpeg)|
+------------------+
         |
         v
+------------------+
| Post-Processing  |
| (Thumbnail, AI)  |
+------------------+
```

### Upload vs Streaming Path (Separated)

```
                    UPLOAD PATH (Write-Heavy)
                    ========================

User ──► API Gateway ──► Upload Service ──► Temp Storage (S3)
                                                    |
                                                    v
                                            Message Queue (Kafka)
                                                    |
                              +─────────────────────+──────────────────+
                              |                     |                  |
                              v                     v                  v
                        Transcoder 1          Transcoder 2       Transcoder N
                        (1080p→240p)          (1080p→480p)       (1080p→4K)
                              |                     |                  |
                              +─────────────────────+──────────────────+
                                                    |
                                                    v
                                          Final Storage (S3/GCS)
                                                    |
                                                    v
                                          CDN Invalidation + DB Update


                    STREAMING PATH (Read-Heavy)
                    ===========================

User ──► DNS (GeoDNS) ──► Nearest CDN Edge
                                |
                          [Cache HIT?]
                          /          \
                        YES           NO
                        |              |
                        v              v
                   Serve from      Origin Server
                   Edge Cache      (Pull content)
                        |              |
                        v              v
                   User Player ◄───────+
                   (HLS/DASH adaptive bitrate)
```

### Data Flow Architecture

```
+───────────────────────────────────────────────────────────+
|                    DATA STORES                              |
+───────────────────────────────────────────────────────────+
|                                                            |
|  [PostgreSQL]     [Cassandra]      [Elasticsearch]        |
|   - Users          - Watch History   - Video Search       |
|   - Channels       - Likes/Views     - Autocomplete      |
|   - Videos meta    - Comments                             |
|                                                            |
|  [Redis]           [S3/GCS]         [Kafka]               |
|   - Session         - Video files    - Events             |
|   - Cache           - Thumbnails     - Analytics          |
|   - Rate Limit      - Subtitles      - Notifications     |
|                                                            |
+───────────────────────────────────────────────────────────+
```

---

## 💻 Detailed Design

### 1. Video Upload & Processing Pipeline

```
UPLOAD PIPELINE বিস্তারিত:
============================

Step 1: Pre-signed URL Generation
    Client ──► API Server ──► Generate S3 Pre-signed URL
                                    |
                              Return URL to Client

Step 2: Direct Upload to Storage
    Client ──► S3 (Direct upload, bypassing API server)
                    |
              Upload Complete Webhook

Step 3: Transcoding Queue
    S3 Event ──► Lambda/Worker ──► Kafka Topic "video.uploaded"

Step 4: Transcoding (Parallel)
    +──────────────────────────────────────────────+
    | FFmpeg Workers (GPU Instances)                |
    |                                              |
    | Input: original.mp4 (1080p)                  |
    |                                              |
    | Output:                                      |
    |   ├── 2160p (4K)  - 15 Mbps  - HEVC        |
    |   ├── 1080p (FHD) - 8 Mbps   - H.264       |
    |   ├── 720p  (HD)  - 5 Mbps   - H.264       |
    |   ├── 480p  (SD)  - 2.5 Mbps - H.264       |
    |   ├── 360p        - 1 Mbps   - H.264       |
    |   ├── 240p        - 0.5 Mbps - H.264       |
    |   └── 144p        - 0.2 Mbps - H.264       |
    |                                              |
    | + HLS segments (10 sec chunks)               |
    | + DASH manifests                             |
    | + Thumbnails (every 5 sec)                   |
    | + Sprite sheet (for seek preview)            |
    +──────────────────────────────────────────────+

Step 5: Post-Processing
    ├── AI Content Moderation (nudity, violence)
    ├── Copyright Detection (Content ID fingerprint)
    ├── Auto-captioning (Speech-to-Text)
    ├── Metadata extraction (duration, codec info)
    └── Thumbnail generation (AI-selected best frame)

Step 6: Publish
    ├── Update DB status: "processing" → "published"
    ├── Push to CDN (warm cache for subscribers)
    ├── Send notifications to subscribers
    └── Index in Elasticsearch
```

### 2. Adaptive Bitrate Streaming (HLS/DASH)

```
HLS (HTTP Live Streaming) Structure:
======================================

master.m3u8 (Master Playlist)
├── stream_4k.m3u8      (2160p, 15 Mbps)
├── stream_1080p.m3u8   (1080p, 8 Mbps)
├── stream_720p.m3u8    (720p, 5 Mbps)
├── stream_480p.m3u8    (480p, 2.5 Mbps)
├── stream_360p.m3u8    (360p, 1 Mbps)
├── stream_240p.m3u8    (240p, 0.5 Mbps)  ◄── Bangladesh 3G default
└── stream_144p.m3u8    (144p, 0.2 Mbps)  ◄── Bangladesh 2G default

প্রতিটি stream playlist:
stream_720p.m3u8
├── segment_001.ts  (10 sec chunk)
├── segment_002.ts
├── segment_003.ts
└── ...

Adaptive Bitrate Logic:
========================
Player measures: Download speed, Buffer level, CPU usage
                        |
                        v
              +─────────────────+
              | ABR Algorithm   |
              | (Buffer-based)  |
              +─────────────────+
                        |
            +-----------+-----------+
            |           |           |
            v           v           v
      Switch UP    Stay Same   Switch DOWN
   (good network)  (stable)  (poor network)
```

### 3. Video Storage Architecture (Hot/Warm/Cold)

```
TIERED STORAGE STRATEGY:
=========================

+──────────────────────────────────────────────────────+
|  HOT TIER (SSD/NVMe) - ৭ দিনের মধ্যে uploaded      |
|  ├── Viral videos (trending)                         |
|  ├── New uploads (< 7 days)                          |
|  ├── Popular creator content                         |
|  └── Storage: S3 Standard / GCS Standard             |
|  Cost: $$$  |  Latency: < 10ms                       |
+──────────────────────────────────────────────────────+
                        |
                        v (৩০ দিন পর)
+──────────────────────────────────────────────────────+
|  WARM TIER - ৩০ দিন - ১ বছর পুরোনো                 |
|  ├── Moderate traffic videos                         |
|  ├── Educational content (evergreen)                 |
|  └── Storage: S3 Infrequent Access                   |
|  Cost: $$   |  Latency: < 50ms                       |
+──────────────────────────────────────────────────────+
                        |
                        v (১ বছর পর)
+──────────────────────────────────────────────────────+
|  COLD TIER - ১ বছর+ পুরোনো, low views              |
|  ├── Rarely accessed videos                          |
|  ├── Archived content                                |
|  └── Storage: S3 Glacier / GCS Coldline              |
|  Cost: $    |  Latency: < 12 hours (retrieval)       |
+──────────────────────────────────────────────────────+
```

### 4. Recommendation Engine Overview

```
RECOMMENDATION SYSTEM:
=======================

Input Signals:
├── Watch History (কী দেখেছে)
├── Search History (কী খুঁজেছে)
├── Like/Dislike (কী পছন্দ করেছে)
├── Watch Duration (কতক্ষণ দেখেছে)
├── Channel Subscriptions
├── Demographics (age, location: Bangladesh)
└── Device & Network (mobile, low bandwidth)

                    +───────────────────+
                    | Candidate         |
                    | Generation        |
                    | (Millions → 1000) |
                    +───────────────────+
                              |
                              v
                    +───────────────────+
                    | Ranking Model     |
                    | (1000 → 50)       |
                    | (Deep Neural Net) |
                    +───────────────────+
                              |
                              v
                    +───────────────────+
                    | Re-ranking        |
                    | (Diversity,       |
                    |  Freshness,       |
                    |  Ad insertion)    |
                    +───────────────────+
                              |
                              v
                    [Final 20 recommendations shown]

Bangladesh-specific ranking factors:
- Bangla language content boost
- Local trending (BPL cricket, Eid content)
- Low-bandwidth friendly (shorter videos prioritized on 2G)
- Data-saver thumbnails
```

### 5. Database Schema

```sql
-- Videos Table (PostgreSQL - Primary metadata)
CREATE TABLE videos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    duration_sec    INTEGER NOT NULL,
    status          VARCHAR(20) DEFAULT 'processing',
    -- status: uploading, processing, published, blocked, deleted
    visibility      VARCHAR(20) DEFAULT 'public',
    -- visibility: public, unlisted, private
    storage_path    VARCHAR(1000) NOT NULL,
    thumbnail_url   VARCHAR(1000),
    original_format VARCHAR(20),
    file_size_bytes BIGINT,
    view_count      BIGINT DEFAULT 0,
    like_count      BIGINT DEFAULT 0,
    dislike_count   BIGINT DEFAULT 0,
    language        VARCHAR(10) DEFAULT 'bn', -- Bengali default
    category_id     INTEGER,
    tags            TEXT[],
    created_at      TIMESTAMP DEFAULT NOW(),
    published_at    TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- Channels Table
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(200) NOT NULL,
    handle          VARCHAR(100) UNIQUE NOT NULL, -- @handle
    description     TEXT,
    avatar_url      VARCHAR(500),
    banner_url      VARCHAR(500),
    subscriber_count BIGINT DEFAULT 0,
    video_count     INTEGER DEFAULT 0,
    country         VARCHAR(2) DEFAULT 'BD',
    verified        BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Subscriptions Table
CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    notify          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, channel_id)
);

-- Watch History (Cassandra - High write throughput)
-- PRIMARY KEY: (user_id, watched_at) for time-series queries
CREATE TABLE watch_history (
    user_id         UUID,
    video_id        UUID,
    watched_at      TIMESTAMP,
    watch_duration  INTEGER,  -- seconds watched
    completed       BOOLEAN,
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);

-- Comments Table (PostgreSQL with partitioning)
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id), -- replies
    content         TEXT NOT NULL,
    like_count      INTEGER DEFAULT 0,
    is_pinned       BOOLEAN DEFAULT FALSE,
    is_hearted      BOOLEAN DEFAULT FALSE, -- creator heart
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
) PARTITION BY HASH(video_id);

-- Video Resolutions (transcoding output tracking)
CREATE TABLE video_resolutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id),
    resolution      VARCHAR(10) NOT NULL, -- '1080p', '720p', etc.
    bitrate_kbps    INTEGER NOT NULL,
    codec           VARCHAR(20) NOT NULL, -- 'h264', 'hevc', 'vp9'
    storage_path    VARCHAR(1000) NOT NULL,
    manifest_url    VARCHAR(1000),
    file_size_bytes BIGINT,
    status          VARCHAR(20) DEFAULT 'pending'
);
```

### 6. PHP (Laravel) Code: Upload API with Pre-signed URL

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Jobs\ProcessVideoUpload;
use App\Models\Video;
use Aws\S3\S3Client;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class VideoUploadController extends Controller
{
    private S3Client $s3Client;

    public function __construct(S3Client $s3Client)
    {
        $this->s3Client = $s3Client;
    }

    /**
     * Step 1: Generate pre-signed URL for direct upload to S3
     * Client uploads directly to S3, bypassing our server
     */
    public function initializeUpload(Request $request): JsonResponse
    {
        $request->validate([
            'filename' => 'required|string|max:255',
            'file_size' => 'required|integer|max:137438953472', // 128GB max
            'content_type' => 'required|in:video/mp4,video/avi,video/mov,video/mkv,video/webm',
            'title' => 'required|string|max:500',
            'description' => 'nullable|string|max:5000',
            'language' => 'nullable|string|max:10',
        ]);

        $user = $request->user();
        $channel = $user->channel;

        if (!$channel) {
            return response()->json(['error' => 'Channel not found'], 404);
        }

        $videoId = Str::uuid()->toString();
        $extension = pathinfo($request->filename, PATHINFO_EXTENSION);
        $storagePath = "uploads/{$channel->id}/{$videoId}/original.{$extension}";

        // Create video record with 'uploading' status
        $video = Video::create([
            'id' => $videoId,
            'channel_id' => $channel->id,
            'title' => $request->title,
            'description' => $request->description,
            'status' => 'uploading',
            'visibility' => 'private', // Private until processing complete
            'storage_path' => $storagePath,
            'original_format' => $extension,
            'file_size_bytes' => $request->file_size,
            'language' => $request->language ?? 'bn',
        ]);

        // Generate pre-signed URL (valid for 6 hours)
        $command = $this->s3Client->getCommand('PutObject', [
            'Bucket' => config('services.s3.video_bucket'),
            'Key' => $storagePath,
            'ContentType' => $request->content_type,
            'Metadata' => [
                'video-id' => $videoId,
                'channel-id' => $channel->id,
                'uploader-id' => $user->id,
            ],
        ]);

        $presignedUrl = $this->s3Client->createPresignedRequest(
            $command,
            '+6 hours'
        )->getUri()->__toString();

        return response()->json([
            'video_id' => $videoId,
            'upload_url' => $presignedUrl,
            'expires_in' => 21600, // 6 hours in seconds
            'storage_path' => $storagePath,
        ], 201);
    }

    /**
     * Step 2: Client confirms upload is complete
     * Triggers transcoding pipeline
     */
    public function completeUpload(Request $request, string $videoId): JsonResponse
    {
        $video = Video::where('id', $videoId)
            ->where('status', 'uploading')
            ->firstOrFail();

        // Verify the file exists in S3
        $exists = $this->s3Client->doesObjectExist(
            config('services.s3.video_bucket'),
            $video->storage_path
        );

        if (!$exists) {
            return response()->json(['error' => 'Upload not found in storage'], 404);
        }

        // Update status to processing
        $video->update(['status' => 'processing']);

        // Dispatch transcoding job to queue
        ProcessVideoUpload::dispatch($video)->onQueue('video-transcoding');

        return response()->json([
            'video_id' => $video->id,
            'status' => 'processing',
            'message' => 'Video is being processed. You will be notified when ready.',
            'estimated_time' => $this->estimateProcessingTime($video->file_size_bytes),
        ]);
    }

    /**
     * Bangladesh context: estimate based on file size
     * Average processing: 1x-3x real-time
     */
    private function estimateProcessingTime(int $fileSizeBytes): string
    {
        $sizeGB = $fileSizeBytes / (1024 * 1024 * 1024);

        if ($sizeGB < 1) return '5-10 minutes';
        if ($sizeGB < 5) return '15-30 minutes';
        if ($sizeGB < 20) return '1-2 hours';
        return '2-6 hours';
    }
}
```

```php
<?php

namespace App\Jobs;

use App\Models\Video;
use App\Models\VideoResolution;
use App\Services\TranscodingService;
use App\Services\ThumbnailService;
use App\Services\ContentModerationService;
use App\Events\VideoPublished;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class ProcessVideoUpload implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $timeout = 7200; // 2 hours max
    public int $tries = 3;

    public function __construct(private Video $video) {}

    public function handle(
        TranscodingService $transcoder,
        ThumbnailService $thumbnails,
        ContentModerationService $moderation
    ): void {
        Log::info("Processing video: {$this->video->id}");

        try {
            // Step 1: Content moderation check
            $moderationResult = $moderation->analyze($this->video->storage_path);
            if ($moderationResult->isBlocked()) {
                $this->video->update([
                    'status' => 'blocked',
                    'block_reason' => $moderationResult->reason,
                ]);
                return;
            }

            // Step 2: Transcode to multiple resolutions
            $resolutions = ['2160p', '1080p', '720p', '480p', '360p', '240p', '144p'];

            foreach ($resolutions as $resolution) {
                $result = $transcoder->transcode(
                    inputPath: $this->video->storage_path,
                    outputResolution: $resolution,
                    videoId: $this->video->id
                );

                VideoResolution::create([
                    'video_id' => $this->video->id,
                    'resolution' => $resolution,
                    'bitrate_kbps' => $result->bitrate,
                    'codec' => $result->codec,
                    'storage_path' => $result->outputPath,
                    'manifest_url' => $result->manifestUrl,
                    'file_size_bytes' => $result->fileSize,
                    'status' => 'completed',
                ]);
            }

            // Step 3: Generate thumbnails
            $thumbnailUrl = $thumbnails->generateBestFrame($this->video->storage_path);
            $thumbnails->generateSpriteSheet($this->video->storage_path, $this->video->id);

            // Step 4: Extract metadata (duration, etc.)
            $metadata = $transcoder->extractMetadata($this->video->storage_path);

            // Step 5: Publish
            $this->video->update([
                'status' => 'published',
                'visibility' => 'public',
                'thumbnail_url' => $thumbnailUrl,
                'duration_sec' => $metadata->durationSeconds,
                'published_at' => now(),
            ]);

            // Notify subscribers
            event(new VideoPublished($this->video));

        } catch (\Exception $e) {
            Log::error("Video processing failed: {$this->video->id}", [
                'error' => $e->getMessage(),
            ]);
            $this->video->update(['status' => 'failed']);
            throw $e;
        }
    }
}
```

### 7. Node.js (Express) Code: Streaming & Metadata API

```javascript
const express = require('express');
const router = express.Router();
const { S3Client, GetObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const Redis = require('ioredis');
const { Pool } = require('pg');
const kafka = require('./kafka');

const redis = new Redis(process.env.REDIS_URL);
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const s3 = new S3Client({ region: process.env.AWS_REGION });

/**
 * GET /api/videos/:id/stream
 * Returns HLS master playlist URL with adaptive bitrate
 * Bangladesh optimization: default to lower quality on slow connections
 */
router.get('/videos/:id/stream', async (req, res) => {
    try {
        const { id } = req.params;
        const userAgent = req.headers['user-agent'];
        const clientHints = req.headers['downlink']; // Network Information API

        // Check cache first
        const cached = await redis.get(`stream:${id}`);
        if (cached) {
            const streamInfo = JSON.parse(cached);
            return res.json(streamInfo);
        }

        // Fetch video and its resolutions
        const videoQuery = await db.query(
            `SELECT v.id, v.title, v.duration_sec, v.status,
                    vr.resolution, vr.manifest_url, vr.bitrate_kbps
             FROM videos v
             JOIN video_resolutions vr ON v.id = vr.video_id
             WHERE v.id = $1 AND v.status = 'published' AND vr.status = 'completed'
             ORDER BY vr.bitrate_kbps DESC`,
            [id]
        );

        if (videoQuery.rows.length === 0) {
            return res.status(404).json({ error: 'Video not found' });
        }

        const video = videoQuery.rows[0];
        const resolutions = videoQuery.rows.map(row => ({
            resolution: row.resolution,
            manifest_url: row.manifest_url,
            bitrate_kbps: row.bitrate_kbps,
        }));

        // Bangladesh bandwidth detection
        const networkQuality = detectNetworkQuality(clientHints, req.ip);
        const defaultResolution = getDefaultResolution(networkQuality);

        const streamInfo = {
            video_id: id,
            title: video.title,
            duration: video.duration_sec,
            master_playlist: `${process.env.CDN_BASE_URL}/hls/${id}/master.m3u8`,
            resolutions,
            default_resolution: defaultResolution,
            cdn_url: process.env.CDN_BASE_URL,
            subtitles: await getSubtitles(id),
        };

        // Cache for 1 hour
        await redis.setex(`stream:${id}`, 3600, JSON.stringify(streamInfo));

        // Track view event asynchronously
        trackViewEvent(id, req.user?.id, req.ip);

        res.json(streamInfo);
    } catch (error) {
        console.error('Stream error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

/**
 * GET /api/videos/:id
 * Video metadata endpoint with view counting
 */
router.get('/videos/:id', async (req, res) => {
    try {
        const { id } = req.params;

        // Try cache
        const cached = await redis.get(`video:meta:${id}`);
        if (cached) {
            return res.json(JSON.parse(cached));
        }

        const result = await db.query(
            `SELECT v.*, c.name as channel_name, c.handle as channel_handle,
                    c.avatar_url as channel_avatar, c.subscriber_count,
                    c.verified as channel_verified
             FROM videos v
             JOIN channels c ON v.channel_id = c.id
             WHERE v.id = $1 AND v.status = 'published'`,
            [id]
        );

        if (result.rows.length === 0) {
            return res.status(404).json({ error: 'Video not found' });
        }

        const video = result.rows[0];

        const metadata = {
            id: video.id,
            title: video.title,
            description: video.description,
            duration: video.duration_sec,
            views: video.view_count,
            likes: video.like_count,
            dislikes: video.dislike_count,
            published_at: video.published_at,
            language: video.language,
            tags: video.tags,
            thumbnail: video.thumbnail_url,
            channel: {
                id: video.channel_id,
                name: video.channel_name,
                handle: video.channel_handle,
                avatar: video.channel_avatar,
                subscribers: video.subscriber_count,
                verified: video.channel_verified,
            },
        };

        // Cache for 5 minutes (views change frequently)
        await redis.setex(`video:meta:${id}`, 300, JSON.stringify(metadata));

        res.json(metadata);
    } catch (error) {
        console.error('Metadata error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

/**
 * POST /api/videos/:id/view
 * Deduplicated view counting using Redis HyperLogLog
 */
router.post('/videos/:id/view', async (req, res) => {
    const { id } = req.params;
    const viewerId = req.user?.id || req.ip;

    // HyperLogLog for approximate unique view counting
    const key = `views:${id}:${getTodayDate()}`;
    const isNew = await redis.pfadd(key, viewerId);
    await redis.expire(key, 86400 * 2); // Expire after 2 days

    if (isNew) {
        // Increment view count (batched, not real-time to DB)
        await redis.hincrby('view_counts_buffer', id, 1);
    }

    // Publish to Kafka for analytics
    await kafka.produce('video.viewed', {
        video_id: id,
        viewer_id: viewerId,
        timestamp: Date.now(),
        country: req.headers['cf-ipcountry'] || 'BD',
        device: parseUserAgent(req.headers['user-agent']),
    });

    res.status(204).send();
});

/**
 * Bangladesh-specific network quality detection
 */
function detectNetworkQuality(downlinkHeader, ip) {
    if (downlinkHeader) {
        const mbps = parseFloat(downlinkHeader);
        if (mbps < 0.5) return 'very_slow';   // 2G
        if (mbps < 2) return 'slow';           // 3G
        if (mbps < 10) return 'moderate';      // 4G
        return 'fast';                          // WiFi/Broadband
    }
    // Default for Bangladesh IPs
    return 'slow';
}

function getDefaultResolution(networkQuality) {
    const mapping = {
        'very_slow': '144p',
        'slow': '240p',
        'moderate': '480p',
        'fast': '1080p',
    };
    return mapping[networkQuality] || '360p';
}

async function getSubtitles(videoId) {
    const result = await db.query(
        `SELECT language, url FROM subtitles WHERE video_id = $1`,
        [videoId]
    );
    return result.rows;
}

function trackViewEvent(videoId, userId, ip) {
    // Fire-and-forget analytics event
    kafka.produce('analytics.video_view', {
        video_id: videoId,
        user_id: userId,
        ip,
        timestamp: Date.now(),
    }).catch(err => console.error('Analytics error:', err));
}

function getTodayDate() {
    return new Date().toISOString().split('T')[0];
}

function parseUserAgent(ua) {
    if (/mobile/i.test(ua)) return 'mobile';
    if (/tablet/i.test(ua)) return 'tablet';
    return 'desktop';
}

module.exports = router;
```

### 8. Comment & Engagement System

```javascript
/**
 * POST /api/videos/:id/comments
 * Nested comment system with real-time updates
 */
router.post('/videos/:id/comments', authenticate, async (req, res) => {
    const { id } = req.params;
    const { content, parent_id } = req.body;

    if (!content || content.trim().length === 0) {
        return res.status(400).json({ error: 'Comment cannot be empty' });
    }

    // Rate limiting: max 10 comments per minute
    const rateKey = `comment_rate:${req.user.id}`;
    const count = await redis.incr(rateKey);
    if (count === 1) await redis.expire(rateKey, 60);
    if (count > 10) {
        return res.status(429).json({ error: 'Too many comments. Please wait.' });
    }

    const result = await db.query(
        `INSERT INTO comments (video_id, user_id, parent_id, content)
         VALUES ($1, $2, $3, $4)
         RETURNING id, content, created_at`,
        [id, req.user.id, parent_id || null, content.trim()]
    );

    const comment = result.rows[0];

    // Publish real-time update via WebSocket
    await redis.publish(`comments:${id}`, JSON.stringify({
        type: 'new_comment',
        comment: { ...comment, user: req.user },
    }));

    res.status(201).json(comment);
});
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. HLS vs DASH

```
+─────────────────+──────────────────────+──────────────────────+
|   বৈশিষ্ট্য     |        HLS           |        DASH          |
+─────────────────+──────────────────────+──────────────────────+
| Developer       | Apple                | MPEG consortium      |
| Container       | MPEG-TS / fMP4       | fMP4                 |
| DRM Support     | FairPlay             | Widevine, PlayReady  |
| iOS Support     | ✅ Native            | ❌ Needs player      |
| Android         | ✅ ExoPlayer         | ✅ Native            |
| Segment Size    | 6-10 sec default     | 2-6 sec default      |
| Latency         | Higher (30s+)        | Lower (10-20s)       |
| Adoption        | Wider (Apple ecosystem)| Open standard       |
| BD Context      | ✅ Better for mobile | ⚠️ Complex setup    |
+─────────────────+──────────────────────+──────────────────────+

সিদ্ধান্ত: HLS primary (iOS + broader support) + DASH fallback
YouTube আসলে দুটোই support করে, কিন্তু DASH prefer করে (VP9/AV1 codec)
বাংলাদেশে mobile users বেশি হওয়ায় HLS priority দেওয়া উচিত
```

### 2. Pre-transcoding vs On-demand Transcoding

```
PRE-TRANSCODING (YouTube approach):
====================================
장점:
  ✅ Instant playback (no wait time)
  ✅ Better quality (more time for encoding)
  ✅ Simpler streaming infrastructure
  ✅ CDN caching efficient

단점:
  ❌ High storage cost (7+ versions per video)
  ❌ Processing delay before publish (minutes to hours)
  ❌ Wasted storage for unpopular resolutions

ON-DEMAND TRANSCODING:
=======================
장점:
  ✅ Lower storage (transcode only what's requested)
  ✅ Faster publish time
  ✅ Cost-effective for low-view videos

단점:
  ❌ First viewer faces delay
  ❌ Complex real-time transcoding infrastructure
  ❌ Higher compute cost during peaks

HYBRID APPROACH (recommended for BD market):
=============================================
- Popular resolutions pre-transcoded: 360p, 480p, 720p
- 4K/1080p on-demand for premium content
- 144p/240p always pre-transcoded (BD low-bandwidth users)
```

### 3. Pull CDN vs Push CDN

```
+─────────────────+──────────────────────+──────────────────────+
|                  |     PULL CDN         |     PUSH CDN         |
+─────────────────+──────────────────────+──────────────────────+
| কীভাবে কাজ করে  | Edge request করলে    | Origin থেকে edge-এ   |
|                  | origin থেকে আনে     | আগেই পাঠিয়ে দেয়    |
| Best for         | Long-tail content    | Viral/popular videos |
| Cache Miss       | First request slow   | No cache miss        |
| Storage at Edge  | On-demand            | Pre-populated        |
| Cost             | Pay per request      | Higher storage cost  |
| BD Context       | গ্রামের edge server  | Dhaka CDN pre-warm   |
+─────────────────+──────────────────────+──────────────────────+

সিদ্ধান্ত: Hybrid
- Push: Top 1% viral videos → Dhaka, Chittagong edges
- Pull: Remaining 99% content → fetch on demand
- Pre-warm: Cricket match streams → push to BD edges before match
```

### 4. SQL vs NoSQL for Video Metadata

```
+─────────────────+──────────────────────+──────────────────────+
|                  |   SQL (PostgreSQL)   | NoSQL (Cassandra)    |
+─────────────────+──────────────────────+──────────────────────+
| Video Metadata   | ✅ ACID, relations   | ⚠️ Denormalized     |
| View Counts      | ❌ Write bottleneck  | ✅ High write        |
| Comments         | ✅ Nested queries    | ⚠️ Complex          |
| Watch History    | ❌ Scale issues      | ✅ Time-series       |
| Search           | ⚠️ LIKE is slow     | ❌ No full-text      |
| Recommendations  | ❌ Complex joins     | ✅ Wide rows         |
+─────────────────+──────────────────────+──────────────────────+

সিদ্ধান্ত: Polyglot Persistence
- PostgreSQL: Videos, Channels, Users (structured, relational)
- Cassandra: Watch History, View Counts (high write throughput)
- Elasticsearch: Search, Autocomplete
- Redis: Caching, Rate Limiting, Real-time counters
- Neo4j (optional): Recommendation graph
```

---

## 📈 কেস স্টাডি

### Case Study 1: Viral Video (100M Views in 24 Hours)

```
পরিস্থিতি: একজন Bangladeshi YouTuber-এর video viral হয়েছে
(যেমন: "বাংলাদেশ vs India Cricket Highlights")

Timeline:
=========
T+0h:   Video published, 1K views
T+1h:   Shared on Facebook groups, 100K views
T+3h:   Trending #1 in Bangladesh, 5M views
T+6h:   International spread, 20M views
T+12h:  Peak: 50M views, 2M concurrent
T+24h:  Total: 100M views

System Response:
================

1. CDN Auto-scaling:
   ├── T+1h: BD edge servers handle load
   ├── T+3h: Additional edge nodes activated in India, ME
   ├── T+6h: Global CDN warm-up triggered
   └── T+12h: Dedicated CDN capacity allocated

2. Origin Shield:
   ├── Multi-tier caching prevents origin overload
   ├── 99.5% requests served from edge
   └── Only 0.5% reaches origin

3. View Counter:
   ├── Redis HyperLogLog for deduplication
   ├── Batch writes to DB every 30 seconds
   └── Approximate count shown (exact later)

4. Comment System:
   ├── Rate limiting activated (spam prevention)
   ├── Comments queued during peak
   └── Real-time WebSocket scaled to 500K connections

5. Cost Impact:
   ├── CDN bandwidth: $50,000-100,000 for the day
   ├── Compute: $5,000-10,000
   └── Revenue (ads): $100,000-500,000
```

### Case Study 2: Live Streaming Cricket Match (BPL Final)

```
পরিস্থিতি: BPL Final Live Stream - ১০ million concurrent viewers in Bangladesh

Architecture for Live:
======================

                    Encoder (Stadium)
                          |
                    RTMP Ingest Server
                          |
                    Live Transcoder
                    (Real-time, <1s)
                          |
              +-----------+-----------+
              |           |           |
           1080p        720p        480p
              |           |           |
         HLS Packager (2-sec segments)
                          |
                    Origin Server
                          |
              +-----------+-----------+
              |           |           |
         BD Edge 1   BD Edge 2   BD Edge 3
         (Dhaka)    (CTG)       (Sylhet)
              |           |           |
         5M users    3M users    2M users

Key Challenges:
===============
1. Ultra-low latency: 5-10 sec delay acceptable
2. Massive concurrent: 10M viewers, 2 Tbps bandwidth
3. Chat system: 100K messages/second
4. Score overlay: Real-time graphics injection
5. BD network: ISPs bottleneck at peak

Solutions:
==========
1. P2P CDN (WebRTC mesh) to reduce server load by 40%
2. ISP peering agreements (Grameenphone, Robi, Banglalink)
3. Edge computing at ISP level (content closer to users)
4. Adaptive quality: Force 360p during peak congestion
5. Chat sharding: Regional chat rooms (Dhaka, CTG, etc.)
```

### Case Study 3: Copyright Detection (Content ID)

```
Content ID System:
==================

Upload Time:
    New Video ──► Audio Fingerprint (Chromaprint)
                      |
                  ──► Video Fingerprint (perceptual hash)
                      |
                  ──► Match against Reference DB
                      |
              +───────+───────+
              |       |       |
           No Match  Match   Partial
              |       |       |
           Publish  Block/   Monetize
                   Claim     for Owner

Bangladesh Context:
===================
- Bangla music copyright (Coke Studio Bangla)
- Cricket footage (ICC, BCB rights)
- Bollywood/Tollywood content in Bangla dub
- Waz mahfil recordings (religious content - fair use?)
- News clips (Somoy TV, NTV, etc.)

Challenges:
===========
- ভুল detection (false positive) - legitimate commentary blocked
- Fair use in Bangladesh copyright law is vague
- Small creators affected by large media companies' claims
- Bangla audio fingerprinting less accurate than English
```

---

## 🔧 Advanced Topics

### 1. Live Streaming Architecture

```
LIVE STREAMING PIPELINE:
=========================

Broadcaster                         Viewers
(OBS Studio)                       (Mobile/Web)
     |                                  ^
     | RTMP/SRT                         | HLS/DASH
     v                                  |
+──────────+    +──────────+    +──────────+
| Ingest   |───►| Live     |───►| Edge     |
| Server   |    | Encoder  |    | CDN      |
+──────────+    +──────────+    +──────────+
     |               |               |
     v               v               v
+──────────+    +──────────+    +──────────+
| Recording|    | DVR      |    | Chat     |
| (Archive)|    | (Rewind) |    | Server   |
+──────────+    +──────────+    +──────────+

Low-Latency Options:
├── Standard HLS: 20-30 sec delay
├── Low-Latency HLS (LL-HLS): 2-5 sec delay
├── WebRTC: < 1 sec delay (for small audiences)
└── SRT Protocol: 1-3 sec (broadcast quality)

Bangladesh-specific:
- Low-latency mode disabled on 2G/3G (buffer issues)
- DVR rewind essential (missed moments during buffering)
- Audio-only mode for extreme low bandwidth
```

### 2. Content Moderation (AI)

```
AI MODERATION PIPELINE:
========================

Video Upload ──► Frame Extraction (1 frame/sec)
                        |
              +---------+---------+
              |         |         |
         NSFW Model  Violence  Text/Logo
         (nudity)    Detection  Detection
              |         |         |
              +---------+---------+
                        |
                  Confidence Score
                        |
              +---------+---------+
              |         |         |
         Score > 0.9  0.5-0.9  Score < 0.5
         AUTO BLOCK   HUMAN     AUTO APPROVE
                      REVIEW
                        |
                  BD Moderator Team
                  (Bangla content specialists)

Bangladesh-specific moderation:
- Political content sensitivity
- Religious content guidelines
- Bangla hate speech detection (NLP models)
- Underage protection (BD child protection laws)
```

### 3. Monetization System

```
MONETIZATION ARCHITECTURE:
===========================

Ad Serving Flow:
Video Request ──► Ad Decision Engine
                        |
              +---------+---------+
              |         |         |
         Pre-roll    Mid-roll   Post-roll
         (before)    (during)   (after)
              |         |         |
         Ad Auction (Real-time Bidding)
              |
         Winner Ad ──► Player

Revenue Split:
├── Platform: 45%
├── Creator: 55%
└── Bangladesh payment: bKash/Nagad monthly payout

Creator Monetization Options:
├── Ad Revenue (CPM: $0.50-2.00 for BD traffic)
├── Channel Membership (monthly subscription)
├── Super Chat (live stream donations)
├── Merchandise Shelf
└── Brand Partnerships (sponsored content)

Bangladesh Context:
- Lower CPM than US/EU ($0.50 vs $7-15)
- bKash payout threshold: ৫,০০০ টাকা minimum
- Tax deduction: ১০% TDS on creator earnings
- Rising BD creator economy (2024: $50M+ market)
```

### 4. Video Search & Discovery

```
SEARCH ARCHITECTURE:
=====================

User Query: "বাংলাদেশ ক্রিকেট হাইলাইটস"
     |
     v
+──────────────────────────────+
| Query Processing              |
| ├── Bangla tokenization       |
| ├── Spell correction          |
| ├── Query expansion           |
| └── Intent detection          |
+──────────────────────────────+
     |
     v
+──────────────────────────────+
| Elasticsearch Cluster         |
| ├── Title index (boost: 3x)  |
| ├── Description index         |
| ├── Tags index (boost: 2x)   |
| ├── Captions/Subtitles index  |
| └── Channel name index        |
+──────────────────────────────+
     |
     v
+──────────────────────────────+
| Ranking Signals               |
| ├── Text relevance (BM25)    |
| ├── View count                |
| ├── Freshness (recent boost) |
| ├── Engagement rate           |
| ├── Watch duration            |
| └── Language match (Bangla)   |
+──────────────────────────────+
     |
     v
[Top 20 results displayed]

Autocomplete:
- Trie-based suggestion (Redis Sorted Sets)
- Bangla autocomplete with phonetic matching
- "বাংল" → "বাংলাদেশ ক্রিকেট", "বাংলা নাটক", "বাংলা গান"
```

### 5. Offline Download

```
OFFLINE DOWNLOAD SYSTEM:
=========================

Download Request ──► DRM License Server
                          |
                    License Token (time-limited)
                          |
                    Encrypted Download
                    (AES-128 encryption)
                          |
                    Local Storage (Device)
                          |
                    Offline Player
                    (Decrypt + Play)

Constraints:
├── Download expires: 48 hours (configurable)
├── Max downloads: 25 videos at a time
├── Quality: User selects (144p-1080p)
├── DRM: Widevine L1 (hardware) / L3 (software)
└── Storage estimate shown before download

Bangladesh Use Case:
====================
- Load shedding: ২-৪ ঘন্টা power cut → offline watch
- Train/Bus travel: Dhaka-Chittagong (no network zones)
- Data saving: WiFi-তে download, পরে offline-এ দেখা
- Rural areas: Download at market (WiFi zone) → watch at home

Storage Optimization:
├── 144p (7 min video): ~15 MB  ◄── Best for BD
├── 360p (7 min video): ~50 MB
├── 720p (7 min video): ~150 MB
└── 1080p (7 min video): ~350 MB
```

---

## 🎯 সারসংক্ষেপ

### Key Architecture Decisions

```
+──────────────────────────────────────────────────────────────+
|              VIDEO PLATFORM ARCHITECTURE SUMMARY               |
+──────────────────────────────────────────────────────────────+
|                                                               |
|  1. SEPARATION OF CONCERNS                                    |
|     ├── Upload path ≠ Streaming path                         |
|     ├── Metadata API ≠ Video serving                         |
|     └── Real-time (comments) ≠ Batch (transcoding)           |
|                                                               |
|  2. STORAGE STRATEGY                                          |
|     ├── Object Storage (S3/GCS) for videos                   |
|     ├── Tiered storage (hot/warm/cold)                       |
|     └── CDN for global delivery                              |
|                                                               |
|  3. PROCESSING PIPELINE                                       |
|     ├── Async transcoding (queue-based)                      |
|     ├── Multiple resolutions pre-generated                   |
|     └── AI moderation before publish                         |
|                                                               |
|  4. SCALABILITY PATTERNS                                      |
|     ├── Horizontal scaling (stateless services)              |
|     ├── Database sharding (by video_id, user_id)            |
|     ├── Read replicas for metadata                           |
|     └── CDN absorbs 99%+ read traffic                        |
|                                                               |
|  5. BANGLADESH OPTIMIZATIONS                                  |
|     ├── Low-bandwidth adaptive defaults (240p/360p)          |
|     ├── Offline download (load shedding friendly)            |
|     ├── Local CDN edges (Dhaka, CTG, Sylhet)                |
|     ├── Mobile-first UI design                               |
|     └── bKash/Nagad payment integration                      |
|                                                               |
+──────────────────────────────────────────────────────────────+
```

### Interview Tips

```
ভিডিও প্ল্যাটফর্ম system design interview-এ মনে রাখবেন:

1. "Upload এবং Streaming path আলাদা করুন" - এটা প্রথমেই বলুন
2. "CDN ছাড়া video platform অসম্ভব" - bandwidth cost explain করুন
3. "Transcoding is the bottleneck" - async processing essential
4. "View counting is harder than you think" - deduplication, bot filtering
5. "Storage grows forever" - tiered storage strategy দরকার
6. "Recommendation drives engagement" - not just search

বাংলাদেশ context-এ extra points:
- Low bandwidth handling (ABR + data saver)
- Offline support (load shedding reality)
- Mobile-first (৮০%+ mobile users)
- Local CDN peering (ISP partnerships)
- Payment localization (bKash/Nagad)
```

### Technology Stack Summary

| Layer | Technology | কেন |
|-------|-----------|------|
| API | Laravel (PHP) + Express (Node.js) | Laravel: CRUD, Auth; Node: Real-time, Streaming |
| Database | PostgreSQL + Cassandra | Relational + High-write time-series |
| Cache | Redis Cluster | Session, counters, rate limiting |
| Search | Elasticsearch | Full-text search + Bangla tokenizer |
| Queue | Apache Kafka | Event streaming, transcoding jobs |
| Storage | AWS S3 / Google Cloud Storage | Durable object storage |
| CDN | CloudFront / Akamai | Global video delivery |
| Transcoding | FFmpeg + GPU instances | Video encoding pipeline |
| Monitoring | Prometheus + Grafana | Metrics & alerting |
| CI/CD | GitHub Actions + ArgoCD | Deployment automation |

---

> 📝 **নোট**: এই design YouTube-এর মতো large-scale platform-এর জন্য। ছোট scale-এ (যেমন Chorki/Hoichoi) অনেক component simpler হবে, কিন্তু core concepts একই থাকবে।

> 🇧🇩 **বাংলাদেশ প্রসঙ্গ**: আমাদের দেশে internet infrastructure দ্রুত উন্নত হচ্ছে (4G/5G rollout), তবে এখনও গ্রামাঞ্চলে bandwidth limitation একটি বড় challenge। তাই adaptive bitrate এবং offline capability আমাদের জন্য extra গুরুত্বপূর্ণ।
