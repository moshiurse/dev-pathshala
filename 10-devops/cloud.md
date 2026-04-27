# 📘 ক্লাউড সার্ভিস (AWS, GCP, Azure)

> **মডার্ন সফটওয়্যার ইনফ্রাস্ট্রাকচারের ভিত্তি — ক্লাউড কম্পিউটিং এর মূল সার্ভিস ও ধারণা।**

---

## 📖 সংজ্ঞা

ক্লাউড কম্পিউটিং হলো **ইন্টারনেটের মাধ্যমে** কম্পিউটিং রিসোর্স (সার্ভার, স্টোরেজ, ডাটাবেস, নেটওয়ার্কিং) **অন-ডিমান্ড** ব্যবহার করা — নিজের হার্ডওয়্যার না কিনে।

### ক্লাউড সার্ভিস মডেল

```
┌─────────────────────────────────────────────────────────┐
│                  আপনি ম্যানেজ করেন ↑                    │
│                                                         │
│  On-Premises │   IaaS    │   PaaS    │   SaaS          │
│  ──────────── │ ────────── │ ────────── │ ──────────     │
│  Application │ Application│ Application│ ────────        │
│  Data        │ Data       │ Data       │                 │
│  Runtime     │ Runtime    │ ──────────  │                 │
│  Middleware  │ Middleware │            │                  │
│  OS          │ OS         │            │                  │
│  Virtualizn  │ ──────────  │            │                  │
│  Servers     │            │            │                  │
│  Storage     │            │            │                  │
│  Networking  │            │            │                  │
│  ──────────── │            │            │                  │
│            ক্লাউড প্রোভাইডার ম্যানেজ করে ↓               │
└─────────────────────────────────────────────────────────┘

IaaS: EC2, GCE, Azure VM — আপনি OS থেকে সব ম্যানেজ করেন
PaaS: Elastic Beanstalk, App Engine, Azure App Service — শুধু কোড দিন
SaaS: Gmail, Slack, Salesforce — ব্যবহার করুন, কিছুই ম্যানেজ করতে হয় না
```

---

## ☁️ AWS মূল সার্ভিস (সবচেয়ে বড় প্রোভাইডার)

### কম্পিউট সার্ভিস

```
┌──────────────────────────────────────────────────┐
│                 AWS Compute                       │
│                                                  │
│  EC2 ──────── ভার্চুয়াল সার্ভার (IaaS)          │
│  ECS ──────── কন্টেইনার সার্ভিস (Docker)         │
│  EKS ──────── ম্যানেজড Kubernetes               │
│  Lambda ───── সার্ভারলেস ফাংশন (FaaS)           │
│  Fargate ──── সার্ভারলেস কন্টেইনার              │
│  Lightsail ── সিম্পল VPS                         │
└──────────────────────────────────────────────────┘
```

### সম্পূর্ণ AWS আর্কিটেকচার উদাহরণ

```
ইউজার → Route 53 (DNS) → CloudFront (CDN)
                              │
                              ▼
                     ALB (Load Balancer)
                       │         │
                       ▼         ▼
              ┌─────────┐ ┌─────────┐
              │ ECS/EKS │ │ ECS/EKS │  ← অটো-স্কেলিং
              │ (App)   │ │ (App)   │
              └────┬────┘ └────┬────┘
                   │           │
          ┌────────┼───────────┼────────┐
          ▼        ▼           ▼        ▼
      ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
      │ RDS  │ │Redis │ │  S3  │ │ SQS  │
      │(DB)  │ │Cache │ │(File)│ │(Queue)│
      └──────┘ └──────┘ └──────┘ └──────┘
```

### AWS সার্ভিস তুলনা ও ব্যবহার

| ক্যাটাগরি | সার্ভিস | কাজ | ব্যবহার |
|----------|---------|-----|--------|
| **কম্পিউট** | EC2 | ভার্চুয়াল সার্ভার | ফুল কন্ট্রোল দরকার হলে |
| | Lambda | সার্ভারলেস ফাংশন | ইভেন্ট-ড্রিভেন, ছোট কাজ |
| | ECS/EKS | কন্টেইনার | ডকার/K8s অ্যাপ |
| **ডাটাবেস** | RDS | ম্যানেজড SQL | MySQL, PostgreSQL |
| | DynamoDB | NoSQL | হাই-থ্রুপুট key-value |
| | ElastiCache | ইন-মেমোরি ক্যাশ | Redis, Memcached |
| **স্টোরেজ** | S3 | অবজেক্ট স্টোরেজ | ফাইল, ব্যাকআপ, স্ট্যাটিক |
| | EBS | ব্লক স্টোরেজ | EC2 এর ডিস্ক |
| | EFS | ফাইল সিস্টেম | শেয়ার্ড স্টোরেজ |
| **নেটওয়ার্কিং** | VPC | প্রাইভেট নেটওয়ার্ক | আইসোলেটেড নেটওয়ার্ক |
| | ALB/NLB | লোড ব্যালেন্সার | ট্রাফিক ডিস্ট্রিবিউশন |
| | CloudFront | CDN | স্ট্যাটিক কন্টেন্ট ডেলিভারি |
| | Route 53 | DNS | ডোমেইন ম্যানেজমেন্ট |
| **মেসেজিং** | SQS | মেসেজ কিউ | অ্যাসিঙ্ক্রোনাস প্রসেসিং |
| | SNS | নোটিফিকেশন | পুশ নোটিফিকেশন, ইমেইল |
| | EventBridge | ইভেন্ট বাস | ইভেন্ট-ড্রিভেন আর্কিটেকচার |
| **সিকিউরিটি** | IAM | অ্যাক্সেস কন্ট্রোল | ইউজার/রোল পারমিশন |
| | Secrets Manager | সিক্রেটস | API key, পাসওয়ার্ড সংরক্ষণ |
| | WAF | ওয়েব ফায়ারওয়াল | DDoS, SQL injection প্রতিরোধ |

---

## 🔧 বাস্তব উদাহরণ

### AWS Lambda — সার্ভারলেস ফাংশন (Node.js)

```javascript
// handler.js — AWS Lambda ফাংশন
exports.handler = async (event) => {
    console.log('ইভেন্ট:', JSON.stringify(event));

    // API Gateway থেকে আসা রিকোয়েস্ট
    const httpMethod = event.httpMethod;
    const path = event.path;
    const body = event.body ? JSON.parse(event.body) : null;

    try {
        switch (`${httpMethod} ${path}`) {
            case 'GET /users':
                return await getUsers();

            case 'POST /users':
                return await createUser(body);

            default:
                return response(404, { message: 'রাউট পাওয়া যায়নি' });
        }
    } catch (error) {
        console.error('ত্রুটি:', error);
        return response(500, { message: 'সার্ভার ত্রুটি' });
    }
};

async function getUsers() {
    // DynamoDB থেকে ডাটা পড়া
    const { DynamoDBClient, ScanCommand } = require('@aws-sdk/client-dynamodb');
    const client = new DynamoDBClient({});

    const result = await client.send(new ScanCommand({
        TableName: process.env.USERS_TABLE
    }));

    return response(200, { users: result.Items });
}

async function createUser(body) {
    const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');
    const client = new DynamoDBClient({});

    await client.send(new PutItemCommand({
        TableName: process.env.USERS_TABLE,
        Item: {
            id: { S: Date.now().toString() },
            name: { S: body.name },
            email: { S: body.email }
        }
    }));

    // SQS তে মেসেজ পাঠানো (ওয়েলকাম ইমেইলের জন্য)
    const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');
    const sqs = new SQSClient({});

    await sqs.send(new SendMessageCommand({
        QueueUrl: process.env.EMAIL_QUEUE_URL,
        MessageBody: JSON.stringify({
            type: 'welcome_email',
            email: body.email,
            name: body.name
        })
    }));

    return response(201, { message: 'ইউজার তৈরি হয়েছে' });
}

function response(statusCode, body) {
    return {
        statusCode,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body)
    };
}
```

### S3 — ফাইল আপলোড (PHP Laravel)

```php
<?php

// config/filesystems.php
'disks' => [
    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'ap-southeast-1'),
        'bucket' => env('AWS_BUCKET'),
    ],
],

// app/Services/FileUploadService.php
namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

class FileUploadService
{
    public function upload(UploadedFile $file, string $directory = 'uploads'): string
    {
        $path = Storage::disk('s3')->putFile($directory, $file, 'public');
        return Storage::disk('s3')->url($path);
    }

    public function delete(string $path): bool
    {
        return Storage::disk('s3')->delete($path);
    }

    public function getTemporaryUrl(string $path, int $minutes = 30): string
    {
        return Storage::disk('s3')->temporaryUrl(
            $path,
            now()->addMinutes($minutes)
        );
    }
}
```

### SQS — মেসেজ কিউ (PHP Laravel)

```php
<?php

// config/queue.php
'connections' => [
    'sqs' => [
        'driver' => 'sqs',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'prefix' => env('SQS_PREFIX', 'https://sqs.ap-southeast-1.amazonaws.com/your-account-id'),
        'queue' => env('SQS_QUEUE', 'default'),
        'region' => env('AWS_DEFAULT_REGION', 'ap-southeast-1'),
    ],
],

// app/Jobs/SendWelcomeEmail.php
namespace App\Jobs;

use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public int $backoff = 60;

    public function __construct(
        private readonly User $user
    ) {}

    public function handle(): void
    {
        Mail::to($this->user->email)
            ->send(new WelcomeEmail($this->user));
    }

    public function failed(\Throwable $exception): void
    {
        // ব্যর্থ হলে Slack এ নোটিফাই
        logger()->error("ওয়েলকাম ইমেইল ব্যর্থ: {$this->user->email}", [
            'error' => $exception->getMessage()
        ]);
    }
}

// ব্যবহার
SendWelcomeEmail::dispatch($user)->onQueue('emails');
```

---

## 🔬 তিন প্রোভাইডার তুলনা

| বৈশিষ্ট্য | AWS | GCP | Azure |
|----------|-----|-----|-------|
| **কম্পিউট** | EC2, Lambda | Compute Engine, Cloud Functions | VM, Azure Functions |
| **কন্টেইনার** | ECS, EKS | GKE, Cloud Run | AKS, Container Apps |
| **SQL DB** | RDS, Aurora | Cloud SQL, AlloyDB | Azure SQL, Cosmos DB |
| **NoSQL** | DynamoDB | Firestore, Bigtable | Cosmos DB |
| **স্টোরেজ** | S3 | Cloud Storage | Blob Storage |
| **CDN** | CloudFront | Cloud CDN | Azure CDN |
| **DNS** | Route 53 | Cloud DNS | Azure DNS |
| **কিউ** | SQS | Pub/Sub | Service Bus |
| **সার্ভারলেস** | Lambda | Cloud Functions | Azure Functions |
| **AI/ML** | SageMaker | Vertex AI | Azure ML |
| **মার্কেট শেয়ার** | ~৩২% | ~১০% | ~২৩% |
| **শক্তি** | সবচেয়ে বেশি সার্ভিস | ML/Data, K8s | Enterprise, Microsoft ইন্টিগ্রেশন |

---

## 💰 খরচ অপটিমাইজেশন

### কৌশলসমূহ

```
১. Right-sizing:
   - বেশি বড় ইনস্ট্যান্স ব্যবহার করবেন না
   - CloudWatch/Monitoring দেখে সঠিক সাইজ নির্ধারণ করুন

২. Reserved Instances / Savings Plans:
   - ১-৩ বছরের কমিটমেন্টে ৩০-৭০% সাশ্রয়
   - স্থিতিশীল ওয়ার্কলোডের জন্য

৩. Spot Instances:
   - ৬০-৯০% সাশ্রয়!
   - কিন্তু যেকোনো সময় বন্ধ হয়ে যেতে পারে
   - ব্যাচ প্রসেসিং, CI/CD রানারের জন্য উপযুক্ত

৪. Auto-scaling:
   - লোড অনুযায়ী রিসোর্স বাড়ানো/কমানো
   - রাতে/উইকেন্ডে কম রিসোর্স

৫. সার্ভারলেস:
   - ব্যবহার অনুযায়ী পে (per-request billing)
   - কম ট্রাফিকে সবচেয়ে সস্তা
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **স্কেলেবিলিটি** | সেকেন্ডে রিসোর্স বাড়ানো/কমানো |
| ২ | **কোনো আপফ্রন্ট খরচ নেই** | ব্যবহার অনুযায়ী পে |
| ৩ | **গ্লোবাল রিচ** | বিশ্বের যেকোনো region এ ডেপ্লয় |
| ৪ | **ম্যানেজড সার্ভিস** | DB, Cache, Queue — সব ম্যানেজড |
| ৫ | **হাই অ্যাভেইলেবিলিটি** | মাল্টি-AZ, মাল্টি-region সাপোর্ট |
| ৬ | **সিকিউরিটি** | এনক্রিপশন, IAM, ফায়ারওয়াল বিল্ট-ইন |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **ভেন্ডর লক-ইন** | একটি ক্লাউড থেকে অন্যটিতে মাইগ্রেশন কঠিন |
| ২ | **জটিল প্রাইসিং** | খরচ বুঝতে ও অপটিমাইজ করতে কঠিন |
| ৩ | **লার্নিং কার্ভ** | শত শত সার্ভিস — শিখতে অনেক সময় |
| ৪ | **ডাটা ট্রান্সফার খরচ** | ক্লাউড থেকে ডাটা বের করতে টাকা লাগে (egress) |
| ৫ | **আউটেজ ঝুঁকি** | প্রোভাইডার ডাউন হলে আপনিও ডাউন |

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **IaaS** | VM/সার্ভার — পুরো কন্ট্রোল, বেশি ম্যানেজমেন্ট |
| **PaaS** | শুধু কোড দিন — প্ল্যাটফর্ম বাকিটা করবে |
| **SaaS** | রেডিমেড সফটওয়্যার — শুধু ব্যবহার করুন |
| **ট্রেন্ড** | সার্ভারলেস, কন্টেইনার, এজ কম্পিউটিং |
| **সুপারিশ** | ছোট শুরু করুন, প্রয়োজনে স্কেল করুন |
