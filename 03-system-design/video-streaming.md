# 🎬 ভিডিও স্ট্রিমিং সিস্টেম (Video Streaming System Design)

## 📌 সংজ্ঞা ও মৌলিক ধারণা

**ভিডিও স্ট্রিমিং সিস্টেম** হলো এমন একটি প্ল্যাটফর্ম যেখানে ইউজার ভিডিও আপলোড, ট্রান্সকোড, স্টোর এবং স্ট্রিম করতে পারে। YouTube, Netflix, Facebook Video — সবই ভিডিও স্ট্রিমিং সিস্টেমের উদাহরণ।

### মূল চ্যালেঞ্জ

```
YouTube-এর স্কেল:
  → প্রতি মিনিটে ৫০০+ ঘণ্টার ভিডিও আপলোড হয়
  → প্রতিদিন ১ বিলিয়ন+ ঘণ্টা ভিডিও দেখা হয়
  → ২৪০p থেকে 4K পর্যন্ত রেজোলিউশন সাপোর্ট
  → সারা পৃথিবীতে কম বাফারিং

  এটি ডিজাইন করা সহজ নয়! 😱
```

---

## 🎯 বাস্তব উদাহরণ — বিটিভি vs নেটফ্লিক্স

```
বিটিভি (Traditional Broadcasting):
  → একটি চ্যানেল, একটি রেজোলিউশন
  → সবাই একই সময়ে একই জিনিস দেখে
  → ইন্টারনেট স্পিড বিবেচনা হয় না

নেটফ্লিক্স (Adaptive Streaming):
  → হাজার হাজার কন্টেন্ট, যেকোনো সময়
  → ইন্টারনেট ধীর → 360p, দ্রুত → 1080p (অটো অ্যাডজাস্ট)
  → ঢাকায় বসে আমেরিকান সার্ভার থেকে না এনে
    সিঙ্গাপুরের CDN Edge থেকে আনে → কম বাফারিং ✅
```

---

## 📊 সিস্টেম আর্কিটেকচার

```
          ভিডিও স্ট্রিমিং সিস্টেম আর্কিটেকচার
          ======================================

  ┌──────────┐                    ┌──────────────────┐
  │ ইউজার    │──── আপলোড ────→   │ Upload Service   │
  │ (Creator)│                    └────────┬─────────┘
  └──────────┘                             │
                                           ▼
                                  ┌──────────────────┐
                                  │ Message Queue    │
                                  │ (Kafka/SQS)     │
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │ Transcoding      │
                                  │ Service          │
                                  │ (FFmpeg Workers) │
                                  └────────┬─────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          ▼                ▼                ▼
                    ┌──────────┐    ┌──────────┐    ┌──────────┐
                    │ 240p     │    │ 720p     │    │ 1080p    │
                    │ Segment  │    │ Segment  │    │ Segment  │
                    └────┬─────┘    └────┬─────┘    └────┬─────┘
                         └───────────────┼───────────────┘
                                         ▼
                                  ┌──────────────────┐
                                  │ Object Storage   │
                                  │ (S3/GCS)         │
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
  ┌──────────┐                    │ CDN              │
  │ ইউজার    │◄── স্ট্রিম ──────  │ (CloudFront/     │
  │ (Viewer) │                    │  Akamai)         │
  └──────────┘                    └──────────────────┘
```

---

## 🎛️ ট্রান্সকোডিং (Transcoding)

```
ট্রান্সকোডিং = মূল ভিডিওকে একাধিক ফরম্যাট ও রেজোলিউশনে রূপান্তর

মূল ভিডিও (4K, 5GB):
  ├── 240p  (H.264, 50MB)    ← ২G নেটওয়ার্ক / বাংলাদেশের গ্রামে
  ├── 360p  (H.264, 100MB)   ← ধীর 3G
  ├── 480p  (H.264, 200MB)   ← 3G / Wi-Fi (ধীর)
  ├── 720p  (H.264, 500MB)   ← 4G / Wi-Fi
  ├── 1080p (H.264, 1GB)     ← ব্রডব্যান্ড
  └── 4K    (H.265, 2GB)     ← ফাইবার / 5G

প্রতিটি রেজোলিউশন আবার ছোট ছোট সেগমেন্টে (২-১০ সেকেন্ড) ভাগ করা হয়
→ এটি Adaptive Bitrate Streaming-এর ভিত্তি
```

### সেগমেন্ট কেন?

```
পুরো ভিডিও ডাউনলোড (প্রগ্রেসিভ):
  → ১ ঘণ্টার ভিডিও = ১GB ডাউনলোড শুরু → ইউজার ৫ মিনিট দেখে চলে গেল
  → ৯৫% ব্যান্ডউইথ অপচয়! 😰

সেগমেন্টেড স্ট্রিমিং:
  → ২-১০ সেকেন্ডের সেগমেন্ট একটি একটি করে আনো
  → ইউজার চলে গেলে পরের সেগমেন্ট আনা বন্ধ → কোনো অপচয় নেই ✅
```

---

## 📶 অ্যাডাপ্টিভ বিটরেট স্ট্রিমিং (ABR)

```
ইউজারের ইন্টারনেট স্পিড অনুযায়ী ভিডিও কোয়ালিটি অটো অ্যাডজাস্ট হয়

প্রোটোকল:
  HLS (HTTP Live Streaming) — Apple, সবচেয়ে জনপ্রিয়
  DASH (Dynamic Adaptive Streaming over HTTP) — ওপেন স্ট্যান্ডার্ড

কিভাবে কাজ করে:
  ┌────────────────────────────────────────────────────┐
  │ সময়  │ স্পিড      │ রেজোলিউশন │ কী হচ্ছে          │
  ├────────────────────────────────────────────────────┤
  │ 0:00  │ 5 Mbps    │ 1080p    │ শুরু              │
  │ 0:30  │ 2 Mbps 📉 │ 480p     │ স্পিড কমেছে        │
  │ 1:00  │ 0.5 Mbps 📉│ 240p    │ আরও কমেছে         │
  │ 1:30  │ 8 Mbps 📈 │ 1080p    │ Wi-Fi পেয়েছে!     │
  └────────────────────────────────────────────────────┘

  ইউজারের কোনো বাফারিং নেই — কোয়ালিটি ওঠানামা করে ✅
```

### HLS Manifest ফাইল (m3u8)

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1920x1080
1080p/playlist.m3u8

→ প্লেয়ার ব্যান্ডউইথ অনুযায়ী সঠিক playlist বেছে নেয়
```

---

## 🌐 CDN ব্যবহার ভিডিওতে

```
ভিডিও স্ট্রিমিংয়ে CDN সবচেয়ে গুরুত্বপূর্ণ:

  সমস্যা:
    ঢাকার ইউজার → আমেরিকা সার্ভার → ৩০০ms latency + বাফারিং 😰

  CDN সমাধান:
    ঢাকার ইউজার → সিঙ্গাপুর Edge → ২০ms latency → স্মুদ প্লেব্যাক ✅

  Netflix CDN কৌশল:
    → জনপ্রিয় কন্টেন্ট আগেভাগে Edge-এ রাখো (Pre-positioning)
    → রাতে কম ট্রাফিকে Edge সার্ভার রিফ্রেশ করো
    → ISP-এর ভেতরেই CDN বক্স বসাও (Open Connect)
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

class VideoUploadService
{
    private string $storagePath;

    public function __construct(string $storagePath)
    {
        $this->storagePath = $storagePath;
    }

    public function upload(array $file, int $userId): array
    {
        $videoId = uniqid('vid_', true);
        $extension = pathinfo($file['name'], PATHINFO_EXTENSION);
        $originalPath = "{$this->storagePath}/originals/{$videoId}.{$extension}";

        move_uploaded_file($file['tmp_name'], $originalPath);

        // ট্রান্সকোডিং জব Queue-তে পাঠাও
        $this->enqueueTranscoding($videoId, $originalPath);

        return ['video_id' => $videoId, 'status' => 'processing'];
    }

    private function enqueueTranscoding(string $videoId, string $path): void
    {
        $job = json_encode([
            'video_id' => $videoId,
            'source' => $path,
            'resolutions' => ['240p', '360p', '480p', '720p', '1080p'],
        ]);
        // Redis Queue-তে পাঠাও
        $redis = new Redis();
        $redis->connect('127.0.0.1');
        $redis->rPush('transcoding_queue', $job);
    }
}

// ট্রান্সকোডিং Worker
class TranscodingWorker
{
    private array $profiles = [
        '240p'  => ['-vf' => 'scale=426:240',  '-b:v' => '400k'],
        '360p'  => ['-vf' => 'scale=640:360',  '-b:v' => '800k'],
        '480p'  => ['-vf' => 'scale=854:480',  '-b:v' => '1400k'],
        '720p'  => ['-vf' => 'scale=1280:720', '-b:v' => '2800k'],
        '1080p' => ['-vf' => 'scale=1920:1080','-b:v' => '5000k'],
    ];

    public function process(array $job): void
    {
        foreach ($job['resolutions'] as $res) {
            $profile = $this->profiles[$res];
            $outputDir = "storage/videos/{$job['video_id']}/{$res}";
            mkdir($outputDir, 0755, true);

            // FFmpeg দিয়ে HLS সেগমেন্ট তৈরি
            $cmd = sprintf(
                'ffmpeg -i %s %s %s -hls_time 10 -hls_list_size 0 %s/playlist.m3u8',
                escapeshellarg($job['source']),
                $profile['-vf'],
                "-b:v {$profile['-b:v']}",
                $outputDir
            );
            exec($cmd);
        }

        // মাস্টার manifest তৈরি
        $this->generateMasterManifest($job['video_id']);
    }

    private function generateMasterManifest(string $videoId): void
    {
        $manifest = "#EXTM3U\n";
        foreach ($this->profiles as $res => $profile) {
            $bandwidth = (int) str_replace('k', '000', $profile['-b:v']);
            $manifest .= "#EXT-X-STREAM-INF:BANDWIDTH={$bandwidth}\n";
            $manifest .= "{$res}/playlist.m3u8\n";
        }
        file_put_contents("storage/videos/{$videoId}/master.m3u8", $manifest);
    }
}

// স্ট্রিমিং API
function streamVideo(string $videoId, string $quality): void
{
    $manifestPath = "storage/videos/{$videoId}/{$quality}/playlist.m3u8";
    if (!file_exists($manifestPath)) {
        http_response_code(404);
        return;
    }
    header('Content-Type: application/vnd.apple.mpegurl');
    header('Cache-Control: public, max-age=31536000');
    readfile($manifestPath);
}
```

---

## 💻 JavaScript কোড উদাহরণ

```javascript
// HLS.js দিয়ে অ্যাডাপ্টিভ ভিডিও প্লেয়ার
class VideoPlayer {
  constructor(videoElement) {
    this.video = videoElement;
    this.hls = null;
    this.stats = { currentQuality: '', bufferLength: 0, bandwidth: 0 };
  }

  loadVideo(videoId) {
    const manifestUrl = `/api/videos/${videoId}/master.m3u8`;

    if (Hls.isSupported()) {
      this.hls = new Hls({
        maxBufferLength: 30,        // ৩০ সেকেন্ড বাফার
        startLevel: -1,             // অটো কোয়ালিটি সিলেকশন
        capLevelToPlayerSize: true,  // প্লেয়ার সাইজ অনুযায়ী কোয়ালিটি
      });

      this.hls.loadSource(manifestUrl);
      this.hls.attachMedia(this.video);

      // কোয়ালিটি পরিবর্তন ইভেন্ট
      this.hls.on(Hls.Events.LEVEL_SWITCHED, (_, data) => {
        const level = this.hls.levels[data.level];
        this.stats.currentQuality = `${level.height}p`;
        console.log(`কোয়ালিটি পরিবর্তন: ${this.stats.currentQuality}`);
      });

      // বাফারিং ইভেন্ট
      this.hls.on(Hls.Events.FRAG_BUFFERED, (_, data) => {
        this.stats.bandwidth = Math.round(data.stats.bwEstimate / 1000);
        this.stats.bufferLength = this.hls.media.buffered.length > 0
          ? this.hls.media.buffered.end(0) - this.hls.media.currentTime
          : 0;
      });

      this.hls.on(Hls.Events.ERROR, (_, data) => {
        if (data.fatal) {
          console.error('HLS Error:', data.type);
          this.hls.recoverMediaError();
        }
      });
    } else if (this.video.canPlayType('application/vnd.apple.mpegurl')) {
      // Safari নেটিভ HLS সাপোর্ট
      this.video.src = manifestUrl;
    }
  }

  // ম্যানুয়াল কোয়ালিটি পরিবর্তন
  setQuality(levelIndex) {
    if (this.hls) {
      this.hls.currentLevel = levelIndex; // -১ = অটো
    }
  }

  getAvailableQualities() {
    if (!this.hls) return [];
    return this.hls.levels.map((level, i) => ({
      index: i,
      resolution: `${level.height}p`,
      bitrate: `${Math.round(level.bitrate / 1000)} kbps`,
    }));
  }
}

// সার্ভার সাইড (Node.js) — ভিডিও আপলোড ও ট্রান্সকোডিং
const express = require('express');
const { exec } = require('child_process');
const path = require('path');
const app = express();

app.post('/api/videos/upload', async (req, res) => {
  const videoId = `vid_${Date.now()}`;
  const inputPath = req.file.path;
  const outputBase = path.join('storage/videos', videoId);

  const resolutions = [
    { name: '360p', scale: '640:360', bitrate: '800k' },
    { name: '720p', scale: '1280:720', bitrate: '2800k' },
    { name: '1080p', scale: '1920:1080', bitrate: '5000k' },
  ];

  // প্রতিটি রেজোলিউশনে ট্রান্সকোড
  for (const r of resolutions) {
    const outputDir = path.join(outputBase, r.name);
    await fs.promises.mkdir(outputDir, { recursive: true });

    const cmd = `ffmpeg -i ${inputPath} -vf scale=${r.scale} -b:v ${r.bitrate} ` +
                `-hls_time 10 -hls_list_size 0 ${outputDir}/playlist.m3u8`;

    await new Promise((resolve, reject) => {
      exec(cmd, (err) => err ? reject(err) : resolve());
    });
  }

  res.json({ videoId, status: 'processing complete' });
});

// ব্যবহার (ক্লায়েন্ট)
const player = new VideoPlayer(document.getElementById('video'));
player.loadVideo('vid_12345');
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কেন |
|---|---|
| ভিডিও শেয়ারিং প্ল্যাটফর্ম | আপলোড, ট্রান্সকোড, স্ট্রিম পুরো পাইপলাইন |
| লাইভ স্ট্রিমিং | HLS/DASH + Low-Latency মোড |
| ই-লার্নিং | ভিডিও লেকচার, অ্যাডাপ্টিভ কোয়ালিটি |
| OTT প্ল্যাটফর্ম (Netflix-like) | DRM + CDN + ABR |
| সোশ্যাল মিডিয়া ভিডিও | শর্ট ভিডিও, দ্রুত এনকোডিং |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কেন |
|---|---|
| ছোট ভিডিও (<৫ সেকেন্ড) | সরাসরি MP4 সার্ভ করো, HLS অপ্রয়োজনীয় |
| অফলাইন-অনলি অ্যাপ | প্রগ্রেসিভ ডাউনলোড যথেষ্ট |
| রিয়েল-টাইম ভিডিও কলিং | WebRTC ব্যবহার করো, HLS/DASH নয় |
| লো-বাজেট MVP | YouTube/Vimeo embed করো, নিজে বানিও না |

---

## 🔑 মনে রাখার পয়েন্ট

1. **ট্রান্সকোডিং** — একাধিক রেজোলিউশনে রূপান্তর (FFmpeg, AWS Elastic Transcoder)
2. **ABR (Adaptive Bitrate)** — ইউজারের ইন্টারনেট অনুযায়ী কোয়ালিটি পরিবর্তন
3. **HLS/DASH** — সেগমেন্টেড স্ট্রিমিং প্রোটোকল, ইন্ডাস্ট্রি স্ট্যান্ডার্ড
4. **CDN** — ভিডিওর জন্য অপরিহার্য, Edge থেকে সার্ভ করো
5. ইন্টারভিউতে — Upload Pipeline, Transcoding DAG, CDN Pre-warming আলোচনা করুন
