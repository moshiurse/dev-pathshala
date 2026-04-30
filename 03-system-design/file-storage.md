# 📁 ফাইল স্টোরেজ সিস্টেম ডিজাইন (File Storage System Design)

> বড় স্কেলে ফাইল আপলোড, স্টোর, প্রসেস ও ডেলিভারি — Daraz প্রোডাক্ট ইমেজ থেকে bKash KYC ডকুমেন্ট পর্যন্ত।

---

## 📌 ফাইল স্টোরেজ সিস্টেম ওভারভিউ

Daraz বাংলাদেশে লক্ষ লক্ষ প্রোডাক্ট ইমেজ স্টোর করে, Pathao ড্রাইভারদের ছবি রাখে, bKash KYC ডকুমেন্ট সংরক্ষণ করে।
এই সবকিছুর জন্য একটি স্কেলেবল ফাইল স্টোরেজ সিস্টেম দরকার।

```
  ┌──────────┐      ┌──────────┐      ┌──────────────────┐
  │  Client   │─────▶│  API GW  │─────▶│  Storage Service │
  │ (App/Web) │      │ (Nginx)  │      │  (PHP/Node.js)   │
  └──────────┘      └──────────┘      └────────┬─────────┘
                                                │
                       ┌────────────────────────┼──────────────────┐
                       ▼                        ▼                  ▼
               ┌──────────────┐      ┌────────────────┐    ┌───────────┐
               │ Object Store │      │  Metadata DB   │    │    CDN    │
               │  (S3/MinIO)  │      │ (MySQL/Postgres)│    │(CloudFront)│
               │ ┌───┐ ┌───┐ │      │ name, size     │    │ Edge Node │
               │ │img│ │pdf│ │      │ hash, owner    │    │ ঢাকা, চট্ট │
               │ └───┘ └───┘ │      └────────────────┘    └───────────┘
               └──────────────┘
```

---

## 📊 অবজেক্ট স্টোরেজ vs ব্লক স্টোরেজ vs ফাইল স্টোরেজ

```
┌────────────────┬──────────────────┬────────────────┬──────────────┐
│   বৈশিষ্ট্য    │ Object Storage   │ Block Storage  │ File Storage │
├────────────────┼──────────────────┼────────────────┼──────────────┤
│ ডাটা ইউনিট     │ Object (key-val) │ Fixed block    │ File/Dir     │
│ প্রোটোকল       │ HTTP REST API    │ iSCSI          │ NFS, SMB     │
│ স্কেলেবিলিটি   │ ✅ অসীম          │ ⚠️ সীমিত       │ ⚠️ মাঝারি    │
│ পারফরম্যান্স   │ মাঝারি latency   │ ✅ খুব দ্রুত    │ মাঝারি       │
│ খরচ            │ ✅ সস্তা         │ ❌ ব্যয়বহুল    │ মাঝারি       │
│ উদাহরণ         │ S3, MinIO        │ EBS, SAN       │ EFS, NFS     │
│ ব্যবহার        │ ইমেজ, ভিডিও     │ DB ডিস্ক       │ শেয়ার্ড ফাইল │
└────────────────┴──────────────────┴────────────────┴──────────────┘
```

| পরিস্থিতি | স্টোরেজ টাইপ |
|---|---|
| Daraz প্রোডাক্ট ইমেজ | ✅ Object Storage (S3) |
| bKash KYC ডকুমেন্ট | ✅ Object Storage (S3) |
| MySQL ডাটাবেস ভলিউম | ✅ Block Storage (EBS) |
| Grameenphone কল রেকর্ড আর্কাইভ | ✅ Object Storage (Glacier) |

---

## 🏗️ S3-কম্প্যাটিবল স্টোরেজ আর্কিটেকচার

```
  Bucket: "daraz-product-images"
  ├── electronics/
  │   ├── phone-001.jpg       ◀── Object (Key = "electronics/phone-001.jpg")
  │   └── laptop-045.webp
  ├── fashion/
  │   └── shirt-100.jpg
  └── thumbnails/
      └── phone-001_thumb.jpg
```

### অভ্যন্তরীণ আর্কিটেকচার

```
                    ┌─────────────────────┐
                    │    S3 API Gateway    │
                    │  (REST: PUT/GET/DEL) │
                    └──────────┬──────────┘
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
          ┌──────────┐  ┌──────────┐  ┌──────────┐
          │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
          │ Replica×3│  │ Replica×3│  │ Replica×3│
          └──────────┘  └──────────┘  └──────────┘

  ✅ প্রতিটি object ৩টি AZ তে রেপ্লিকেট — 99.999999999% durability
```

**লোকাল অল্টারনেটিভ:** MinIO (ওপেন-সোর্স, ডেভ/টেস্ট), DigitalOcean Spaces (ছোট প্রজেক্ট), Backblaze B2 (আর্কাইভ)

---

## 📤 আপলোড ফ্লো

### ১. ডাইরেক্ট আপলোড (❌ স্কেল হয় না)

```
  Client ──[file]──▶ App Server ──[file]──▶ S3      ← সার্ভার বোটলনেক!
```

### ২. Pre-signed URL আপলোড (✅ রেকমেন্ডেড)

```
  Client ──[request]──▶ Server ──[generate URL]──▶ S3
    │                      │
    │  ◀──[signed URL]─────┘
    └──[upload directly]─────────────────────────▶ S3
```

#### PHP কোড — Pre-signed URL জেনারেশন

```php
<?php
use Aws\S3\S3Client;

public function getUploadUrl(Request $request)
{
    $request->validate([
        'file_name' => 'required|string',
        'file_type' => 'required|in:image/jpeg,image/png,image/webp',
    ]);

    $s3 = new S3Client(['region' => 'ap-southeast-1', 'version' => 'latest']);
    $key = 'uploads/' . auth()->id() . '/' . uniqid() . '_' . $request->file_name;

    $cmd = $s3->getCommand('PutObject', [
        'Bucket' => 'daraz-product-images', 'Key' => $key,
        'ContentType' => $request->file_type,
    ]);

    $url = $s3->createPresignedRequest($cmd, '+15 minutes');
    return response()->json(['upload_url' => (string) $url->getUri(), 'key' => $key]);
}
```

#### JavaScript কোড — ক্লায়েন্ট সাইড আপলোড

```javascript
async function uploadFile(file) {
    // ১. সার্ভার থেকে signed URL নাও
    const { data } = await axios.post('/api/upload-url', {
        file_name: file.name, file_type: file.type,
    });
    // ২. সরাসরি S3 তে আপলোড
    await axios.put(data.upload_url, file, {
        headers: { 'Content-Type': file.type },
        onUploadProgress: (e) => console.log(`${Math.round((e.loaded/e.total)*100)}%`),
    });
    // ৩. সার্ভারকে জানাও
    await axios.post('/api/upload-complete', { file_key: data.key });
}
```

### ৩. মাল্টিপার্ট আপলোড (বড় ফাইলের জন্য)

```
  500MB ভিডিও ──▶ ১০টি 50MB অংশ ──▶ সমান্তরালে S3 তে আপলোড ──▶ জোড়া দাও
  ✅ ব্যর্থ হলে শুধু ঐ অংশ পুনরায় আপলোড  ✅ সমান্তরাল — দ্রুততর
```

---

## 🌐 CDN ইন্টিগ্রেশন

CDN ফাইলকে ব্যবহারকারীর কাছাকাছি edge server থেকে সার্ভ করে। ঢাকার ইউজারের জন্য ঢাকার edge node।

```
  Cache Miss:   ইউজার ──▶ CDN Edge (ঢাকা) ──▶ Origin (S3) ──▶ ক্যাশে রাখো ──▶ রেসপন্স
  Cache Hit:    ইউজার ──▶ CDN Edge (ঢাকা) ──▶ সরাসরি রেসপন্স (অতি দ্রুত!)
```

```
  Origin URL:  https://daraz-images.s3.amazonaws.com/products/phone-001.jpg
  CDN URL:     https://img.daraz.com.bd/products/phone-001.jpg?w=400&format=webp
```

**Origin Pull** (CDN নিজে টানে, সহজ, ✅ বেশিরভাগ ক্ষেত্রে) vs **Push CDN** (আমরা পুশ করি, ভিডিওর জন্য)

---

## 🖼️ ইমেজ প্রসেসিং পাইপলাইন

```
  আপলোড ──▶ S3 (Original) ──▶ Queue ──▶ Worker
                                          │
                               ┌──────────┼──────────┐
                               ▼          ▼          ▼
                          Thumb 150px  Med 400px  Large 800px
                               │          │          │
                               └──────────┼──────────┘
                                          ▼
                                  WebP কনভার্ট (80%)
                                          ▼
                                    S3 + CDN invalidate
```

### PHP কোড (Intervention Image)

```php
<?php
use Intervention\Image\ImageManager;
use Intervention\Image\Drivers\Gd\Driver;

$manager = new ImageManager(new Driver());
$sizes = ['thumb' => [150,150], 'medium' => [400,400], 'large' => [800,800]];

foreach ($sizes as $name => [$w, $h]) {
    $manager->read($sourcePath)->cover($w, $h)->toWebp(quality: 80)
        ->save("{$outputDir}/{$name}.webp");
}
```

### JavaScript কোড (Sharp)

```javascript
const sharp = require('sharp');
const sizes = { thumb: [150,150], medium: [400,400], large: [800,800] };

for (const [name, [w, h]] of Object.entries(sizes)) {
    await sharp(inputBuffer).resize(w, h, { fit: 'cover' })
        .webp({ quality: 80 }).toFile(`${outputDir}/${name}.webp`);
}
```

---

## 🗃️ মেটাডাটা ম্যানেজমেন্ট

S3 তে ফাইল থাকে, ডাটাবেসে মেটাডাটা — সার্চ, ফিল্টার, অ্যাক্সেস কন্ট্রোলের জন্য।

```sql
CREATE TABLE files (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid         CHAR(36) UNIQUE NOT NULL,
    owner_id     BIGINT NOT NULL,
    file_key     VARCHAR(500) NOT NULL,       -- S3 object key
    original_name VARCHAR(255) NOT NULL,
    mime_type    VARCHAR(100) NOT NULL,
    file_size    BIGINT NOT NULL,
    content_hash VARCHAR(64) NOT NULL,        -- SHA-256 (ডিডুপ্লিকেশন)
    is_public    BOOLEAN DEFAULT FALSE,
    variants     JSON,                        -- {"thumb":"...","medium":"..."}
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_owner (owner_id),
    INDEX idx_hash (content_hash)
);
```

### ডিডুপ্লিকেশন (Content Hashing)

```
  আপলোড ──▶ SHA-256 হ্যাশ ──▶ DB তে চেক
                                 ├── আছে ──▶ রেফারেন্স দাও (নতুন কপি নয়)
                                 └── নেই ──▶ S3 তে আপলোড + DB এন্ট্রি

  Pathao তে ১০,০০০ ড্রাইভার একই ডিফল্ট avatar ──▶ শুধু ১ কপি!
```

---

## 🔐 অ্যাক্সেস কন্ট্রোল

```
  পাবলিক ফাইল                     প্রাইভেট ফাইল
  ─────────────                   ──────────────
  Daraz প্রোডাক্ট ইমেজ             bKash NID ডকুমেন্ট
  ওয়েবসাইট লোগো                   Pathao ড্রাইভিং লাইসেন্স
  ✅ CDN দিয়ে সার্ভ                ✅ Signed URL দিয়ে সার্ভ (সময়সীমিত)
```

### PHP — Signed URL দিয়ে প্রাইভেট ফাইল অ্যাক্সেস

```php
<?php
public function getDownloadUrl(string $fileUuid)
{
    $file = File::where('uuid', $fileUuid)->firstOrFail();
    if ($file->owner_id !== auth()->id() && !auth()->user()->isAdmin()) {
        abort(403, 'অ্যাক্সেস নেই');
    }

    $s3 = new S3Client(['region' => 'ap-southeast-1', 'version' => 'latest']);
    $cmd = $s3->getCommand('GetObject', [
        'Bucket' => $file->bucket, 'Key' => $file->file_key,
    ]);
    $url = $s3->createPresignedRequest($cmd, '+15 minutes');

    return response()->json(['download_url' => (string) $url->getUri()]);
}
```

### JavaScript — Signed URL (Node.js)

```javascript
const { S3Client, GetObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');

const s3 = new S3Client({ region: 'ap-southeast-1' });

async function getDownloadUrl(bucket, key) {
    const cmd = new GetObjectCommand({ Bucket: bucket, Key: key });
    return await getSignedUrl(s3, cmd, { expiresIn: 900 }); // ১৫ মিনিট
}
```

---

## 💻 PHP ও JS সম্পূর্ণ কোড উদাহরণ

### Laravel S3 ফাইল সার্ভিস

```php
<?php
use Illuminate\Support\Facades\Storage;

class FileService
{
    public function upload(UploadedFile $file, int $ownerId): array
    {
        $allowed = ['image/jpeg','image/png','image/webp','application/pdf'];
        if (!in_array($file->getMimeType(), $allowed)) {
            throw new \InvalidArgumentException('অনুমোদিত ফাইল টাইপ নয়');
        }

        $hash = hash_file('sha256', $file->getRealPath());
        $existing = File::where('content_hash', $hash)->first();
        if ($existing) return ['file' => $existing, 'deduplicated' => true];

        $key = "uploads/{$ownerId}/" . uniqid() . '_' . $file->getClientOriginalName();
        Storage::disk('s3')->put($key, file_get_contents($file), 'private');

        $record = File::create([
            'uuid' => \Str::uuid(), 'owner_id' => $ownerId,
            'file_key' => $key, 'original_name' => $file->getClientOriginalName(),
            'mime_type' => $file->getMimeType(), 'file_size' => $file->getSize(),
            'content_hash' => $hash,
        ]);

        if (str_starts_with($file->getMimeType(), 'image/')) {
            ProcessImageJob::dispatch($record->id);
        }
        return ['file' => $record, 'deduplicated' => false];
    }
}
```

### Node.js Express আপলোড API

```javascript
const multer = require('multer');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const crypto = require('crypto');

const s3 = new S3Client({ region: 'ap-southeast-1' });
const upload = multer({ limits: { fileSize: 10 * 1024 * 1024 } });

app.post('/api/files', upload.single('file'), async (req, res) => {
    const { file } = req;
    const allowed = ['image/jpeg','image/png','image/webp','application/pdf'];
    if (!allowed.includes(file.mimetype)) return res.status(400).json({ error: 'অনুমোদিত নয়' });

    const hash = crypto.createHash('sha256').update(file.buffer).digest('hex');
    const key = `uploads/${req.user.id}/${crypto.randomUUID()}_${file.originalname}`;

    await s3.send(new PutObjectCommand({
        Bucket: 'daraz-product-images', Key: key,
        Body: file.buffer, ContentType: file.mimetype,
    }));

    const record = await db.file.create({ data: {
        uuid: crypto.randomUUID(), ownerId: req.user.id, fileKey: key,
        originalName: file.originalname, mimeType: file.mimetype,
        fileSize: file.size, contentHash: hash,
    }});
    res.json({ file: record });
});
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### ✅ S3/Object Storage ব্যবহার করবেন

- লক্ষ লক্ষ ফাইল (Daraz প্রোডাক্ট ইমেজ)
- CDN দিয়ে দ্রুত ডেলিভারি দরকার
- durability গুরুত্বপূর্ণ (bKash KYC)
- pay-per-use খরচ মডেল চান

### ❌ S3 দরকার নেই

- ছোট প্রজেক্ট, <1,000 ফাইল — লোকাল ডিস্ক যথেষ্ট
- শুধু ডেভ/টেস্ট — MinIO ব্যবহার করুন
- ব্লক-লেভেল অ্যাক্সেস দরকার — EBS ব্যবহার করুন

```
  ফাইল সংখ্যা?
  ├── <1K     ──▶ লোকাল ডিস্ক
  ├── 1K-100K ──▶ MinIO / DO Spaces
  └── >100K   ──▶ AWS S3 + CloudFront
```

---

## ❌ সাধারণ ভুল ও খারাপ প্যাটার্ন

### ভুল ১: ডাটাবেসে BLOB হিসেবে ফাইল রাখা

```php
// ❌ খারাপ — DB ফুলে যাবে, CDN ব্যবহার অসম্ভব
DB::table('products')->insert(['image' => file_get_contents($file)]);

// ✅ ভালো — S3 তে ফাইল, DB তে শুধু key
Storage::disk('s3')->put($key, file_get_contents($file));
DB::table('products')->insert(['image_key' => $key]);
```

### ভুল ২: ফাইল ভ্যালিডেশন না করা

```php
// ❌ খারাপ — কোনো চেক নেই, ম্যালওয়্যার আপলোড সম্ভব
Storage::put('uploads/' . $file->getClientOriginalName(), $file);

// ✅ ভালো — টাইপ, সাইজ, নাম সব চেক
$request->validate(['file' => 'required|file|max:10240|mimes:jpg,png,webp,pdf']);
$safeName = uniqid() . '.' . $file->extension();
```

### ভুল ৩: অ্যাপ সার্ভার দিয়ে ফাইল সার্ভ করা

```
  ❌ Client ──▶ PHP Server ──▶ S3 ──▶ PHP ──▶ Client    (সার্ভার বোটলনেক)
  ✅ Client ──▶ CDN Edge ──▶ Client                      (দ্রুত, সার্ভার ফ্রি)
```

### ভুল ৪: সাইজ লিমিট ও ফাইলনেম স্যানিটাইজ না করা

```javascript
// ❌ কোনো লিমিট নেই — 5GB আপলোড সম্ভব!
const upload = multer();

// ✅ লিমিট + ফিল্টার
const upload = multer({
    limits: { fileSize: 10 * 1024 * 1024, files: 5 },
    fileFilter: (req, file, cb) => {
        cb(null, ['image/jpeg','image/png','image/webp'].includes(file.mimetype));
    },
});
// ✅ সিস্টেম জেনারেটেড নাম ব্যবহার করুন: "a3f8b2c1-uuid.jpg"
//    কখনো ইউজারের দেওয়া নাম সরাসরি ব্যবহার করবেন না
```

---

## 💡 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────────────┐
│                  ফাইল স্টোরেজ চেকলিস্ট                       │
├───────────────────────────────────────────────────────────────┤
│ ✅ Object Storage (S3/MinIO) ব্যবহার করুন, DB BLOB নয়        │
│ ✅ Pre-signed URL দিয়ে আপলোড — সার্ভার লোড কমান              │
│ ✅ CDN দিয়ে ফাইল সার্ভ — দ্রুত ডেলিভারি                      │
│ ✅ ইমেজ প্রসেসিং — রিসাইজ, WebP, কম্প্রেশন                   │
│ ✅ মেটাডাটা DB তে — সার্চ ও ট্র্যাকিং                        │
│ ✅ Content hashing — ডিডুপ্লিকেশন                             │
│ ✅ Signed URL — প্রাইভেট ফাইলের নিরাপদ অ্যাক্সেস              │
│ ✅ ভ্যালিডেশন — টাইপ, সাইজ, ফাইলনেম                          │
└───────────────────────────────────────────────────────────────┘
```

---

## 🔗 সম্পর্কিত টপিক

- [CDN](./cdn.md) — CDN কনফিগারেশন ও ক্যাশিং বিস্তারিত
- [Caching](./caching.md) — ক্যাশিং স্ট্র্যাটেজি
- [Database Scaling](./database-scaling.md) — মেটাডাটা DB স্কেলিং
- [Video Streaming](./video-streaming.md) — ভিডিও ফাইলের বিশেষ হ্যান্ডলিং

> **"ফাইল অ্যাপ সার্ভারে রাখবেন না, S3 তে রাখুন। সার্ভার stateless রাখুন।"** 🎯
