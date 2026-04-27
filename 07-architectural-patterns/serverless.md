# ☁️ সার্ভারলেস আর্কিটেকচার (Serverless Architecture)

## 📌 সংজ্ঞা ও মূল ধারণা

### Serverless কী?

সার্ভারলেস আর্কিটেকচার মানে সার্ভার নেই — এমন না। সার্ভারলেস মানে **আপনাকে সার্ভার ম্যানেজ করতে হবে না**। ক্লাউড প্রোভাইডার সার্ভার provisioning, scaling, patching সব নিজে হ্যান্ডেল করে। আপনি শুধু কোড লিখবেন এবং deploy করবেন।

সার্ভারলেস দুটি প্রধান ক্যাটেগরিতে ভাগ করা যায়:

**১. FaaS (Function as a Service):**
- AWS Lambda, Azure Functions, Google Cloud Functions
- আপনি একটি function লেখেন, ক্লাউড সেটি event-এ response-এ execute করে
- প্রতিটি invocation স্বাধীন, stateless, ephemeral

**২. BaaS (Backend as a Service):**
- Firebase, AWS Amplify, Supabase
- পুরো backend service ready-made পাওয়া যায়
- Authentication, database, storage — সব managed

### Cold Start vs Warm Start

```
Cold Start (প্রথমবার invoke):
┌──────────────────────────────────────────────────────────┐
│ Download Code → Start Runtime → Init Dependencies → Run  │
│     ~100ms        ~200ms          ~300ms+         ~50ms  │
│                                                          │
│ Total: ~650ms+ (PHP Bref: ~800ms, Node.js: ~200-400ms)  │
└──────────────────────────────────────────────────────────┘

Warm Start (container পুনরায় ব্যবহার):
┌──────────────────────────────────────────────────────────┐
│ Run Function                                             │
│   ~50ms                                                  │
│                                                          │
│ Total: ~50ms (কোনো initialization নেই)                   │
└──────────────────────────────────────────────────────────┘
```

Cold start হয় যখন Lambda container প্রথমবার তৈরি হয় বা দীর্ঘ সময় idle থাকার পর destroy হয়ে যায়। Warm start হয় যখন আগের container পুনরায় ব্যবহার হয়। PHP-তে cold start তুলনামূলক বেশি কারণ runtime নিজেই ভারী।

### Execution Model

সার্ভারলেস function-এর execution model তিনটি মূল বৈশিষ্ট্যের উপর নির্ভর করে:

- **Event-Triggered:** প্রতিটি function একটি event-এ respond করে (HTTP request, S3 upload, SQS message, schedule)
- **Stateless:** Function-এর মধ্যে কোনো state persist করে না। প্রতিটি invocation স্বাধীন। State রাখতে হলে DynamoDB, Redis, বা S3 ব্যবহার করতে হবে
- **Ephemeral:** Function execute হয়ে শেষ হলে container destroy হতে পারে। Maximum execution time সীমিত (Lambda: 15 মিনিট)

### ক্লাউড প্রোভাইডার তুলনা

| Feature              | AWS Lambda         | Azure Functions       | Google Cloud Functions |
|---------------------|--------------------|-----------------------|------------------------|
| Max Timeout         | 15 min             | 230 sec (Consumption) | 9 min (HTTP), 60 min (Event) |
| Max Memory          | 10,240 MB          | 1,536 MB              | 32,768 MB              |
| PHP Support         | Bref (custom)      | Custom handler        | Custom runtime         |
| Node.js Support     | Native (v20)       | Native (v20)          | Native (v20)           |
| Cold Start          | ~200ms (Node.js)   | ~500ms (Node.js)      | ~400ms (Node.js)       |
| Pricing (per 1M)    | $0.20              | $0.20                 | $0.40                  |
| Concurrent Limit    | 1000 (default)     | 200 (Consumption)     | 1000                   |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### Taxi vs নিজের গাড়ি

সার্ভারলেস বোঝার সবচেয়ে সহজ উপমা হলো **ট্যাক্সি বনাম নিজের গাড়ি**:

**নিজের গাড়ি (Traditional Server):**
- গাড়ি কিনতে হয় (সার্ভার কেনা/ভাড়া)
- পার্কিং লাগে (hosting cost সবসময় চলে)
- মেইনটেন্যান্স করতে হয় (OS update, security patch)
- ব্যবহার না করলেও খরচ চলতে থাকে
- বেশি মানুষ হলে বড় গাড়ি কিনতে হয় (scaling)

**Uber/Pathao (Serverless):**
- শুধু যখন দরকার তখনই ডাকো (event-driven invocation)
- ভাড়া শুধু ব্যবহারের জন্য (pay-per-execution)
- গাড়ি maintain করা তোমার দায়িত্ব না (managed infrastructure)
- ৫ জনের জন্য ৫টা গাড়ি ডাকো (automatic scaling)
- কখনো কখনো গাড়ি আসতে সময় লাগে (cold start!)

**বাংলাদেশ কনটেক্সটে উদাহরণ:**
- bKash webhook processing — payment notification আসলেই function trigger হয়, না আসলে কোনো cost নেই
- Daraz-এর image resizing — product image upload হলেই thumbnail, medium, large তৈরি হয়
- OTP verification — SMS পাঠানোর সময়ই function চলে, বাকি সময় idle

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### সার্ভারলেস অ্যাপ্লিকেশন আর্কিটেকচার

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Serverless Application Architecture              │
│                                                                     │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌─────────────┐  │
│  │  Client   │───▶│    API    │───▶│  Lambda  │───▶│  DynamoDB   │  │
│  │ (Mobile/  │    │  Gateway  │    │ Function │    │  (Storage)  │  │
│  │   Web)    │◀───│           │◀───│          │◀───│             │  │
│  └──────────┘    └───────────┘    └──────────┘    └─────────────┘  │
│                                         │                           │
│                                         ├───▶ S3 (Files)           │
│                                         ├───▶ SES (Email)          │
│                                         ├───▶ SNS (Notification)   │
│                                         └───▶ SQS (Queue)         │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Sources → Lambda → Services Flow

```
Event Sources                 Lambda Functions              Downstream Services
┌─────────────┐              ┌──────────────┐              ┌──────────────┐
│ API Gateway  │─────────────▶│  API Handler │─────────────▶│  DynamoDB    │
│ (REST/HTTP)  │              │              │              │              │
└─────────────┘              └──────────────┘              └──────────────┘

┌─────────────┐              ┌──────────────┐              ┌──────────────┐
│ S3 Upload    │─────────────▶│ Image Resize │─────────────▶│  S3 Output   │
│ (PutObject)  │              │              │       ┌─────▶│  + CDN       │
└─────────────┘              └──────────────┘       │      └──────────────┘
                                                     │
┌─────────────┐              ┌──────────────┐       │      ┌──────────────┐
│ SQS Queue    │─────────────▶│ Order Process│───────┼─────▶│  RDS/Aurora  │
│              │              │              │       │      │              │
└─────────────┘              └──────────────┘       │      └──────────────┘
                                                     │
┌─────────────┐              ┌──────────────┐       │      ┌──────────────┐
│ CloudWatch   │─────────────▶│ Cleanup Job  │───────┘─────▶│  SNS/SES     │
│ (Schedule)   │              │              │              │ (Notify)     │
└─────────────┘              └──────────────┘              └──────────────┘

┌─────────────┐              ┌──────────────┐              ┌──────────────┐
│ DynamoDB     │─────────────▶│ Stream       │─────────────▶│  Elasticsearch│
│ Streams      │              │ Processor    │              │  (Search)    │
└─────────────┘              └──────────────┘              └──────────────┘
```

### API Gateway + Lambda + DynamoDB (bKash Webhook)

```
  bKash Payment                API Gateway           Lambda              DynamoDB
  Notification                 (HTTPS)               Function            Table
      │                           │                     │                   │
      │  POST /webhook/bkash      │                     │                   │
      │──────────────────────────▶│                     │                   │
      │                           │  Validate + Route   │                   │
      │                           │────────────────────▶│                   │
      │                           │                     │  Verify Signature │
      │                           │                     │──────┐            │
      │                           │                     │◀─────┘            │
      │                           │                     │                   │
      │                           │                     │  Save Transaction │
      │                           │                     │──────────────────▶│
      │                           │                     │                   │
      │                           │                     │  Send SNS (SMS)   │
      │                           │                     │──────▶ SNS        │
      │                           │                     │                   │
      │                           │   200 OK            │                   │
      │◀──────────────────────────│◀────────────────────│                   │
```

---

## 💻 Serverless Fundamentals

### AWS Lambda — Function Structure

#### PHP (Bref Framework)

Bref হলো PHP-তে AWS Lambda চালানোর জন্য সবচেয়ে জনপ্রিয় framework। যেহেতু AWS Lambda natively PHP support করে না, Bref custom runtime layer ব্যবহার করে:

```php
<?php
// handler.php — PHP Lambda Handler (Bref)
declare(strict_types=1);

use Bref\Context\Context;
use Bref\Event\Http\HttpHandler;
use Bref\Event\Http\HttpRequestEvent;
use Bref\Event\Http\HttpResponse;

require __DIR__ . '/vendor/autoload.php';

class BkashWebhookHandler extends HttpHandler
{
    public function __construct(
        private readonly DynamoDbClient $dynamoDb,
        private readonly SnsClient $sns,
        private readonly string $tableName = 'bkash_transactions',
    ) {}

    public function handleRequest(
        HttpRequestEvent $event,
        Context $context
    ): HttpResponse {
        $requestId = $context->getAwsRequestId();
        $body = json_decode($event->getBody(), true, flags: JSON_THROW_ON_ERROR);

        // bKash webhook signature verify
        $signature = $event->getHeaders()['x-bkash-signature'] ?? '';
        if (!$this->verifySignature($event->getBody(), $signature)) {
            return new HttpResponse(
                json_encode(['error' => 'Invalid signature']),
                ['Content-Type' => 'application/json'],
                401
            );
        }

        // DynamoDB-তে transaction সংরক্ষণ
        $this->dynamoDb->putItem([
            'TableName' => $this->tableName,
            'Item' => [
                'transactionId' => ['S' => $body['trxID']],
                'amount' => ['N' => (string) $body['amount']],
                'sender' => ['S' => $body['customerMsisdn']],
                'status' => ['S' => 'COMPLETED'],
                'timestamp' => ['N' => (string) time()],
                'requestId' => ['S' => $requestId],
            ],
        ]);

        // SNS notification পাঠানো
        $this->sns->publish([
            'TopicArn' => $_ENV['NOTIFICATION_TOPIC_ARN'],
            'Message' => json_encode([
                'type' => 'PAYMENT_RECEIVED',
                'trxID' => $body['trxID'],
                'amount' => $body['amount'],
            ]),
        ]);

        return new HttpResponse(
            json_encode(['status' => 'ok', 'requestId' => $requestId]),
            ['Content-Type' => 'application/json'],
            200
        );
    }

    private function verifySignature(string $payload, string $signature): bool
    {
        $secret = $_ENV['BKASH_WEBHOOK_SECRET'];
        $computed = hash_hmac('sha256', $payload, $secret);
        return hash_equals($computed, $signature);
    }
}

// Lambda handler হিসেবে return
return new BkashWebhookHandler(
    dynamoDb: new DynamoDbClient(['region' => $_ENV['AWS_REGION']]),
    sns: new SnsClient(['region' => $_ENV['AWS_REGION']]),
);
```

#### Node.js (Native Lambda)

```javascript
// handler.mjs — Node.js Lambda Handler (ES Module)
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';
import { createHmac, timingSafeEqual } from 'node:crypto';

const dynamoDb = new DynamoDBClient({ region: process.env.AWS_REGION });
const sns = new SNSClient({ region: process.env.AWS_REGION });

// SDK client initialization — function-এর বাইরে রাখলে warm start-এ reuse হয়
const TABLE_NAME = process.env.TABLE_NAME ?? 'bkash_transactions';
const TOPIC_ARN = process.env.NOTIFICATION_TOPIC_ARN;

export const handler = async (event, context) => {
    const { requestId } = context.awsRequestId;
    const body = JSON.parse(event.body);
    const signature = event.headers?.['x-bkash-signature'] ?? '';

    // Signature verification
    if (!verifySignature(event.body, signature)) {
        return {
            statusCode: 401,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ error: 'Invalid signature' }),
        };
    }

    // DynamoDB-তে সংরক্ষণ
    await dynamoDb.send(new PutItemCommand({
        TableName: TABLE_NAME,
        Item: {
            transactionId: { S: body.trxID },
            amount: { N: String(body.amount) },
            sender: { S: body.customerMsisdn },
            status: { S: 'COMPLETED' },
            timestamp: { N: String(Date.now()) },
            requestId: { S: context.awsRequestId },
        },
    }));

    // SNS notification
    await sns.send(new PublishCommand({
        TopicArn: TOPIC_ARN,
        Message: JSON.stringify({
            type: 'PAYMENT_RECEIVED',
            trxID: body.trxID,
            amount: body.amount,
        }),
    }));

    return {
        statusCode: 200,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ status: 'ok', requestId: context.awsRequestId }),
    };
};

function verifySignature(payload, signature) {
    const secret = process.env.BKASH_WEBHOOK_SECRET;
    const computed = createHmac('sha256', secret).update(payload).digest('hex');
    return timingSafeEqual(Buffer.from(computed), Buffer.from(signature));
}
```

### Serverless Framework Configuration

```yaml
# serverless.yml
service: bkash-webhook-service
frameworkVersion: '3'

provider:
  name: aws
  region: ap-southeast-1  # Singapore (বাংলাদেশের সবচেয়ে কাছের region)
  runtime: nodejs20.x
  memorySize: 256
  timeout: 30
  environment:
    TABLE_NAME: ${self:service}-transactions-${sls:stage}
    NOTIFICATION_TOPIC_ARN: !Ref PaymentNotificationTopic
    BKASH_WEBHOOK_SECRET: ${ssm:/bkash/webhook-secret}
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:PutItem
            - dynamodb:GetItem
            - dynamodb:Query
          Resource: !GetAtt TransactionsTable.Arn
        - Effect: Allow
          Action: sns:Publish
          Resource: !Ref PaymentNotificationTopic

functions:
  bkashWebhook:
    handler: handler.handler
    events:
      - httpApi:
          path: /webhook/bkash
          method: POST
    reservedConcurrency: 100

  # PHP function (Bref ব্যবহার করে)
  phpBkashWebhook:
    handler: handler.php
    runtime: provided.al2023
    layers:
      - ${bref:layer.php-83-fpm}
    events:
      - httpApi:
          path: /webhook/bkash-php
          method: POST

plugins:
  - serverless-bref

resources:
  Resources:
    TransactionsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-transactions-${sls:stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: transactionId
            AttributeType: S
        KeySchema:
          - AttributeName: transactionId
            KeyType: HASH

    PaymentNotificationTopic:
      Type: AWS::SNS::Topic
      Properties:
        TopicName: ${self:service}-payment-notifications-${sls:stage}
```

---

## 🔥 Complex Scenarios

### ১. REST API with Serverless — Full CRUD

#### Node.js CRUD Handler

```javascript
// api/products.mjs — E-commerce Product CRUD (Daraz-style)
import { DynamoDBDocumentClient, PutCommand, GetCommand, QueryCommand,
         UpdateCommand, DeleteCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const TABLE = process.env.PRODUCTS_TABLE;

// Router pattern — একটি Lambda-তে CRUD handle করা
export const handler = async (event) => {
    const { httpMethod, pathParameters, body } = event;
    const productId = pathParameters?.id;

    try {
        switch (`${httpMethod} ${event.resource}`) {
            case 'POST /products':
                return await createProduct(JSON.parse(body));
            case 'GET /products/{id}':
                return await getProduct(productId);
            case 'GET /products':
                return await listProducts(event.queryStringParameters);
            case 'PUT /products/{id}':
                return await updateProduct(productId, JSON.parse(body));
            case 'DELETE /products/{id}':
                return await deleteProduct(productId);
            default:
                return response(404, { error: 'Route not found' });
        }
    } catch (err) {
        console.error('Handler error:', err);
        return response(500, { error: 'Internal server error' });
    }
};

async function createProduct(data) {
    const product = {
        id: crypto.randomUUID(),
        ...data,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
    };

    await client.send(new PutCommand({ TableName: TABLE, Item: product }));
    return response(201, product);
}

async function getProduct(id) {
    const { Item } = await client.send(
        new GetCommand({ TableName: TABLE, Key: { id } })
    );
    if (!Item) return response(404, { error: 'Product not found' });
    return response(200, Item);
}

async function listProducts(params) {
    const limit = parseInt(params?.limit ?? '20', 10);
    const category = params?.category;

    const command = category
        ? new QueryCommand({
            TableName: TABLE,
            IndexName: 'category-index',
            KeyConditionExpression: 'category = :cat',
            ExpressionAttributeValues: { ':cat': category },
            Limit: limit,
        })
        : new QueryCommand({
            TableName: TABLE,
            Limit: limit,
        });

    const { Items } = await client.send(command);
    return response(200, { products: Items, count: Items.length });
}

async function updateProduct(id, data) {
    const updateExpr = Object.keys(data)
        .map((key, i) => `#k${i} = :v${i}`)
        .join(', ');

    const exprNames = Object.fromEntries(
        Object.keys(data).map((key, i) => [`#k${i}`, key])
    );
    const exprValues = Object.fromEntries(
        Object.values(data).map((val, i) => [`:v${i}`, val])
    );

    const { Attributes } = await client.send(new UpdateCommand({
        TableName: TABLE,
        Key: { id },
        UpdateExpression: `SET ${updateExpr}, #upd = :updAt`,
        ExpressionAttributeNames: { ...exprNames, '#upd': 'updatedAt' },
        ExpressionAttributeValues: { ...exprValues, ':updAt': new Date().toISOString() },
        ReturnValues: 'ALL_NEW',
    }));

    return response(200, Attributes);
}

async function deleteProduct(id) {
    await client.send(new DeleteCommand({ TableName: TABLE, Key: { id } }));
    return response(204, null);
}

const response = (statusCode, body) => ({
    statusCode,
    headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
    },
    body: body ? JSON.stringify(body) : '',
});
```

#### PHP Bref CRUD Handler

```php
<?php
// api/ProductHandler.php — PHP Bref CRUD
declare(strict_types=1);

use Aws\DynamoDb\DynamoDbClient;
use Aws\DynamoDb\Marshaler;
use Bref\Context\Context;
use Bref\Event\Http\HttpHandler;
use Bref\Event\Http\HttpRequestEvent;
use Bref\Event\Http\HttpResponse;

final class ProductHandler extends HttpHandler
{
    private readonly Marshaler $marshaler;

    public function __construct(
        private readonly DynamoDbClient $db,
        private readonly string $table,
    ) {
        $this->marshaler = new Marshaler();
    }

    public function handleRequest(HttpRequestEvent $event, Context $context): HttpResponse
    {
        $method = $event->getMethod();
        $path = $event->getPath();
        $pathParts = explode('/', trim($path, '/'));

        return match (true) {
            $method === 'POST' && $path === '/products'
                => $this->create($event),
            $method === 'GET' && count($pathParts) === 2
                => $this->get($pathParts[1]),
            $method === 'GET' && $path === '/products'
                => $this->list($event),
            $method === 'PUT' && count($pathParts) === 2
                => $this->update($pathParts[1], $event),
            $method === 'DELETE' && count($pathParts) === 2
                => $this->delete($pathParts[1]),
            default => $this->json(['error' => 'Not found'], 404),
        };
    }

    private function create(HttpRequestEvent $event): HttpResponse
    {
        $data = json_decode($event->getBody(), true, flags: JSON_THROW_ON_ERROR);
        $product = [
            'id' => bin2hex(random_bytes(16)),
            ...$data,
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
        ];

        $this->db->putItem([
            'TableName' => $this->table,
            'Item' => $this->marshaler->marshalItem($product),
        ]);

        return $this->json($product, 201);
    }

    private function get(string $id): HttpResponse
    {
        $result = $this->db->getItem([
            'TableName' => $this->table,
            'Key' => $this->marshaler->marshalItem(['id' => $id]),
        ]);

        if (!$result['Item']) {
            return $this->json(['error' => 'Product not found'], 404);
        }

        return $this->json($this->marshaler->unmarshalItem($result['Item']));
    }

    private function list(HttpRequestEvent $event): HttpResponse
    {
        $params = $event->getQueryParameters();
        $limit = (int) ($params['limit'] ?? 20);

        $result = $this->db->scan([
            'TableName' => $this->table,
            'Limit' => $limit,
        ]);

        $items = array_map(
            fn(array $item) => $this->marshaler->unmarshalItem($item),
            $result['Items']
        );

        return $this->json(['products' => $items, 'count' => count($items)]);
    }

    private function update(string $id, HttpRequestEvent $event): HttpResponse
    {
        $data = json_decode($event->getBody(), true, flags: JSON_THROW_ON_ERROR);
        $data['updatedAt'] = date('c');

        $expressions = [];
        $names = [];
        $values = [];

        foreach ($data as $key => $value) {
            $alias = '#' . $key;
            $placeholder = ':' . $key;
            $expressions[] = "{$alias} = {$placeholder}";
            $names[$alias] = $key;
            $values[$placeholder] = $this->marshaler->marshalValue($value);
        }

        $result = $this->db->updateItem([
            'TableName' => $this->table,
            'Key' => $this->marshaler->marshalItem(['id' => $id]),
            'UpdateExpression' => 'SET ' . implode(', ', $expressions),
            'ExpressionAttributeNames' => $names,
            'ExpressionAttributeValues' => $values,
            'ReturnValues' => 'ALL_NEW',
        ]);

        return $this->json($this->marshaler->unmarshalItem($result['Attributes']));
    }

    private function delete(string $id): HttpResponse
    {
        $this->db->deleteItem([
            'TableName' => $this->table,
            'Key' => $this->marshaler->marshalItem(['id' => $id]),
        ]);

        return $this->json(null, 204);
    }

    private function json(mixed $data, int $status = 200): HttpResponse
    {
        return new HttpResponse(
            $data !== null ? json_encode($data, JSON_THROW_ON_ERROR) : '',
            ['Content-Type' => 'application/json', 'Access-Control-Allow-Origin' => '*'],
            $status,
        );
    }
}
```

---

### ২. Event Processing Pipeline

S3-তে image upload হলে → Lambda trigger → resize → DynamoDB-তে metadata সংরক্ষণ → SNS notification। বাংলাদেশের e-commerce (যেমন Daraz, Chaldal) এ এই pattern খুবই common:

#### Node.js — Image Processing Pipeline

```javascript
// pipelines/imageProcessor.mjs
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';
import sharp from 'sharp';

const s3 = new S3Client({});
const dynamoDb = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sns = new SNSClient({});

const SIZES = {
    thumbnail: { width: 150, height: 150, fit: 'cover' },
    medium: { width: 600, height: 600, fit: 'inside' },
    large: { width: 1200, height: 1200, fit: 'inside' },
};

export const handler = async (event) => {
    // S3 event থেকে file info বের করা
    const results = await Promise.allSettled(
        event.Records.map(record => processImage(record))
    );

    const failures = results.filter(r => r.status === 'rejected');
    if (failures.length > 0) {
        console.error('Failed images:', failures.map(f => f.reason.message));
        // Partial failure — SQS-এ retry হবে, failed গুলো DLQ-তে যাবে
        throw new Error(`${failures.length}/${results.length} images failed`);
    }

    return { processed: results.length };
};

async function processImage(record) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    const outputBucket = process.env.OUTPUT_BUCKET;

    // Original image download
    const { Body, ContentType } = await s3.send(
        new GetObjectCommand({ Bucket: bucket, Key: key })
    );
    const imageBuffer = Buffer.from(await Body.transformToByteArray());

    // সব size-এ parallel resize
    const resizeResults = await Promise.all(
        Object.entries(SIZES).map(async ([sizeName, dimensions]) => {
            const resized = await sharp(imageBuffer)
                .resize(dimensions)
                .webp({ quality: 85 })
                .toBuffer();

            const outputKey = `processed/${sizeName}/${key.replace(/\.[^.]+$/, '.webp')}`;

            await s3.send(new PutObjectCommand({
                Bucket: outputBucket,
                Key: outputKey,
                Body: resized,
                ContentType: 'image/webp',
                CacheControl: 'public, max-age=31536000',
            }));

            return { sizeName, key: outputKey, size: resized.length };
        })
    );

    // DynamoDB-তে metadata সংরক্ষণ
    const metadata = {
        imageId: key,
        originalSize: record.s3.object.size,
        variants: Object.fromEntries(
            resizeResults.map(r => [r.sizeName, { key: r.key, size: r.size }])
        ),
        processedAt: new Date().toISOString(),
    };

    await dynamoDb.send(new PutCommand({
        TableName: process.env.METADATA_TABLE,
        Item: metadata,
    }));

    // SNS notification
    await sns.send(new PublishCommand({
        TopicArn: process.env.NOTIFICATION_TOPIC,
        Message: JSON.stringify({
            type: 'IMAGE_PROCESSED',
            imageId: key,
            variants: resizeResults.map(r => r.sizeName),
        }),
    }));

    return metadata;
}
```

#### PHP — SQS Queue Processor

```php
<?php
// pipelines/SqsProcessor.php — Order Processing Queue
declare(strict_types=1);

use Bref\Context\Context;
use Bref\Event\Sqs\SqsEvent;
use Bref\Event\Sqs\SqsHandler;

final class OrderQueueProcessor extends SqsHandler
{
    public function __construct(
        private readonly DynamoDbClient $db,
        private readonly SnsClient $sns,
        private readonly SesClient $ses,
    ) {}

    public function handleSqs(SqsEvent $event, Context $context): void
    {
        foreach ($event->getRecords() as $record) {
            $order = json_decode(
                $record->getBody(),
                true,
                flags: JSON_THROW_ON_ERROR
            );

            try {
                $this->processOrder($order);
            } catch (\Throwable $e) {
                // এই message retry হবে, max retry-র পর DLQ-তে যাবে
                error_log("Order {$order['orderId']} failed: {$e->getMessage()}");
                throw $e;
            }
        }
    }

    private function processOrder(array $order): void
    {
        // ১. Order status update
        $this->db->updateItem([
            'TableName' => $_ENV['ORDERS_TABLE'],
            'Key' => ['orderId' => ['S' => $order['orderId']]],
            'UpdateExpression' => 'SET #status = :status, #processedAt = :time',
            'ExpressionAttributeNames' => [
                '#status' => 'status',
                '#processedAt' => 'processedAt',
            ],
            'ExpressionAttributeValues' => [
                ':status' => ['S' => 'PROCESSING'],
                ':time' => ['S' => date('c')],
            ],
        ]);

        // ২. Inventory check ও update
        foreach ($order['items'] as $item) {
            $this->updateInventory($item['productId'], $item['quantity']);
        }

        // ৩. Customer notification (SMS via SNS)
        $this->sns->publish([
            'TopicArn' => $_ENV['ORDER_NOTIFICATION_TOPIC'],
            'Message' => json_encode([
                'orderId' => $order['orderId'],
                'status' => 'PROCESSING',
                'customerPhone' => $order['customerPhone'],
            ]),
            'MessageAttributes' => [
                'type' => [
                    'DataType' => 'String',
                    'StringValue' => 'ORDER_UPDATE',
                ],
            ],
        ]);

        // ৪. Confirmation email
        $this->ses->sendEmail([
            'Destination' => ['ToAddresses' => [$order['customerEmail']]],
            'Source' => $_ENV['FROM_EMAIL'],
            'Message' => [
                'Subject' => ['Data' => "Order #{$order['orderId']} Confirmed"],
                'Body' => ['Html' => ['Data' => $this->buildEmailHtml($order)]],
            ],
        ]);
    }

    private function updateInventory(string $productId, int $quantity): void
    {
        $this->db->updateItem([
            'TableName' => $_ENV['INVENTORY_TABLE'],
            'Key' => ['productId' => ['S' => $productId]],
            'UpdateExpression' => 'SET stock = stock - :qty',
            'ConditionExpression' => 'stock >= :qty',
            'ExpressionAttributeValues' => [':qty' => ['N' => (string) $quantity]],
        ]);
    }

    private function buildEmailHtml(array $order): string
    {
        $items = implode('', array_map(
            fn(array $item) => "<li>{$item['name']} x {$item['quantity']} — ৳{$item['price']}</li>",
            $order['items']
        ));
        return "<h2>ধন্যবাদ! আপনার অর্ডার নিশ্চিত হয়েছে</h2><ul>{$items}</ul>";
    }
}
```

---

### ৩. Serverless Microservices — Step Functions Orchestration

Step Functions ব্যবহার করে complex workflow orchestrate করা যায়। নিচে একটি e-commerce order processing workflow:

```json
{
  "Comment": "E-Commerce Order Processing — Orchestration",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-southeast-1:ACCOUNT:function:validateOrder",
      "Next": "CheckInventory",
      "Catch": [{ "ErrorEquals": ["ValidationError"], "Next": "OrderFailed" }]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-southeast-1:ACCOUNT:function:checkInventory",
      "Next": "ProcessPayment",
      "Catch": [{ "ErrorEquals": ["OutOfStockError"], "Next": "NotifyOutOfStock" }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-southeast-1:ACCOUNT:function:processPayment",
      "Next": "ParallelProcessing",
      "Retry": [{ "ErrorEquals": ["PaymentGatewayTimeout"], "MaxAttempts": 3, "BackoffRate": 2 }],
      "Catch": [{ "ErrorEquals": ["PaymentFailed"], "Next": "OrderFailed" }]
    },
    "ParallelProcessing": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "UpdateInventory",
          "States": {
            "UpdateInventory": { "Type": "Task", "Resource": "arn:aws:lambda:...:updateInventory", "End": true }
          }
        },
        {
          "StartAt": "SendConfirmation",
          "States": {
            "SendConfirmation": { "Type": "Task", "Resource": "arn:aws:lambda:...:sendConfirmation", "End": true }
          }
        },
        {
          "StartAt": "NotifyWarehouse",
          "States": {
            "NotifyWarehouse": { "Type": "Task", "Resource": "arn:aws:lambda:...:notifyWarehouse", "End": true }
          }
        }
      ],
      "Next": "OrderCompleted"
    },
    "OrderCompleted": { "Type": "Succeed" },
    "OrderFailed": { "Type": "Fail", "Cause": "Order processing failed" },
    "NotifyOutOfStock": { "Type": "Task", "Resource": "arn:aws:lambda:...:notifyOutOfStock", "Next": "OrderFailed" }
  }
}
```

### ৪. Serverless WebSocket — Real-time Chat

```javascript
// websocket/handler.mjs — API Gateway WebSocket Handler
import { DynamoDBDocumentClient, PutCommand, DeleteCommand,
         ScanCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { ApiGatewayManagementApiClient,
         PostToConnectionCommand } from '@aws-sdk/client-apigatewaymanagementapi';

const dynamoDb = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const CONNECTIONS_TABLE = process.env.CONNECTIONS_TABLE;

// $connect route — নতুন connection সংরক্ষণ
export const connect = async (event) => {
    const connectionId = event.requestContext.connectionId;
    const userId = event.queryStringParameters?.userId ?? 'anonymous';

    await dynamoDb.send(new PutCommand({
        TableName: CONNECTIONS_TABLE,
        Item: {
            connectionId,
            userId,
            connectedAt: new Date().toISOString(),
        },
    }));

    return { statusCode: 200 };
};

// $disconnect route — connection মুছে ফেলা
export const disconnect = async (event) => {
    await dynamoDb.send(new DeleteCommand({
        TableName: CONNECTIONS_TABLE,
        Key: { connectionId: event.requestContext.connectionId },
    }));
    return { statusCode: 200 };
};

// sendMessage route — সবাইকে message broadcast
export const sendMessage = async (event) => {
    const { domainName, stage } = event.requestContext;
    const apiGw = new ApiGatewayManagementApiClient({
        endpoint: `https://${domainName}/${stage}`,
    });

    const body = JSON.parse(event.body);
    const message = JSON.stringify({
        action: 'message',
        data: body.data,
        sender: body.userId,
        timestamp: new Date().toISOString(),
    });

    // সব active connection fetch
    const { Items: connections } = await dynamoDb.send(
        new ScanCommand({ TableName: CONNECTIONS_TABLE })
    );

    // সবাইকে broadcast (stale connection গুলো cleanup)
    const sendResults = await Promise.allSettled(
        connections.map(async ({ connectionId }) => {
            try {
                await apiGw.send(new PostToConnectionCommand({
                    ConnectionId: connectionId,
                    Data: message,
                }));
            } catch (err) {
                if (err.statusCode === 410) {
                    // Stale connection — cleanup
                    await dynamoDb.send(new DeleteCommand({
                        TableName: CONNECTIONS_TABLE,
                        Key: { connectionId },
                    }));
                }
                throw err;
            }
        })
    );

    return { statusCode: 200 };
};
```

### ৫. File Processing Pipeline — CSV Import

```php
<?php
// pipelines/CsvImporter.php — বড় CSV ফাইল process করা
declare(strict_types=1);

use Bref\Context\Context;
use Bref\Event\S3\S3Event;
use Bref\Event\S3\S3Handler;

final class CsvImporter extends S3Handler
{
    private const BATCH_SIZE = 25; // DynamoDB batch limit

    public function __construct(
        private readonly S3Client $s3,
        private readonly DynamoDbClient $db,
        private readonly SqsClient $sqs,
    ) {}

    public function handleS3(S3Event $event, Context $context): void
    {
        foreach ($event->getRecords() as $record) {
            $bucket = $record->getBucket()->getName();
            $key = urldecode($record->getObject()->getKey());

            $this->processCsv($bucket, $key, $context);
        }
    }

    private function processCsv(string $bucket, string $key, Context $ctx): void
    {
        // S3 থেকে stream হিসেবে পড়া (memory efficient)
        $result = $this->s3->getObject(['Bucket' => $bucket, 'Key' => $key]);
        $stream = $result['Body'];

        $headers = null;
        $batch = [];
        $totalProcessed = 0;

        while ($line = fgets($stream)) {
            // Lambda timeout চেক — ৩০ সেকেন্ড বাকি থাকলে বন্ধ করো
            if ($ctx->getRemainingTimeInMillis() < 30_000) {
                // বাকি অংশ SQS-এ পাঠিয়ে দাও পরবর্তী Lambda-র জন্য
                $this->sqs->sendMessage([
                    'QueueUrl' => $_ENV['RESUME_QUEUE_URL'],
                    'MessageBody' => json_encode([
                        'bucket' => $bucket,
                        'key' => $key,
                        'startLine' => $totalProcessed,
                    ]),
                ]);
                break;
            }

            $row = str_getcsv(trim($line));

            if ($headers === null) {
                $headers = $row;
                continue;
            }

            $item = array_combine($headers, $row);
            $batch[] = $this->formatForDynamoDb($item);
            $totalProcessed++;

            if (count($batch) >= self::BATCH_SIZE) {
                $this->writeBatch($batch);
                $batch = [];
            }
        }

        // বাকি items write
        if (!empty($batch)) {
            $this->writeBatch($batch);
        }
    }

    private function writeBatch(array $items): void
    {
        $requests = array_map(
            fn(array $item) => ['PutRequest' => ['Item' => $item]],
            $items
        );

        $this->db->batchWriteItem([
            'RequestItems' => [$_ENV['IMPORT_TABLE'] => $requests],
        ]);
    }

    private function formatForDynamoDb(array $item): array
    {
        $marshaler = new \Aws\DynamoDb\Marshaler();
        $item['importedAt'] = date('c');
        $item['id'] = bin2hex(random_bytes(16));
        return $marshaler->marshalItem($item);
    }
}
```

### ৬. Scheduled Jobs (Cron)

```javascript
// cron/cleanup.mjs — Database Cleanup ও Report Generation
import { DynamoDBDocumentClient, ScanCommand,
         BatchWriteCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';

const dynamoDb = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const ses = new SESClient({});

// প্রতিদিন রাত ২টায় পুরানো session মুছে ফেলা
export const cleanupSessions = async () => {
    const thirtyDaysAgo = Date.now() - 30 * 24 * 60 * 60 * 1000;
    let totalDeleted = 0;
    let lastEvaluatedKey;

    do {
        const { Items, LastEvaluatedKey } = await dynamoDb.send(new ScanCommand({
            TableName: process.env.SESSIONS_TABLE,
            FilterExpression: 'lastActive < :cutoff',
            ExpressionAttributeValues: { ':cutoff': thirtyDaysAgo },
            ExclusiveStartKey: lastEvaluatedKey,
        }));

        if (Items.length > 0) {
            // ২৫টা করে batch delete
            const batches = chunkArray(Items, 25);
            for (const batch of batches) {
                await dynamoDb.send(new BatchWriteCommand({
                    RequestItems: {
                        [process.env.SESSIONS_TABLE]: batch.map(item => ({
                            DeleteRequest: { Key: { sessionId: item.sessionId } },
                        })),
                    },
                }));
                totalDeleted += batch.length;
            }
        }

        lastEvaluatedKey = LastEvaluatedKey;
    } while (lastEvaluatedKey);

    console.log(`Cleaned up ${totalDeleted} expired sessions`);
    return { deletedCount: totalDeleted };
};

// প্রতি সোমবার সকাল ৯টায় weekly report generate
export const generateWeeklyReport = async () => {
    const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();

    const { Items: orders } = await dynamoDb.send(new ScanCommand({
        TableName: process.env.ORDERS_TABLE,
        FilterExpression: 'createdAt >= :weekAgo',
        ExpressionAttributeValues: { ':weekAgo': oneWeekAgo },
    }));

    const report = {
        totalOrders: orders.length,
        totalRevenue: orders.reduce((sum, o) => sum + (o.amount ?? 0), 0),
        avgOrderValue: orders.length > 0
            ? orders.reduce((sum, o) => sum + (o.amount ?? 0), 0) / orders.length
            : 0,
        topProducts: getTopProducts(orders),
    };

    await ses.send(new SendEmailCommand({
        Destination: { ToAddresses: [process.env.ADMIN_EMAIL] },
        Source: process.env.FROM_EMAIL,
        Message: {
            Subject: { Data: `Weekly Report — ${new Date().toLocaleDateString('bn-BD')}` },
            Body: {
                Html: { Data: buildReportHtml(report) },
            },
        },
    }));

    return report;
};

function chunkArray(arr, size) {
    return Array.from({ length: Math.ceil(arr.length / size) },
        (_, i) => arr.slice(i * size, (i + 1) * size));
}

function getTopProducts(orders) {
    const counts = {};
    for (const order of orders) {
        for (const item of order.items ?? []) {
            counts[item.name] = (counts[item.name] ?? 0) + item.quantity;
        }
    }
    return Object.entries(counts)
        .sort(([, a], [, b]) => b - a)
        .slice(0, 10)
        .map(([name, count]) => ({ name, count }));
}

function buildReportHtml(report) {
    return `<h2>সাপ্তাহিক রিপোর্ট</h2>
        <p>মোট অর্ডার: ${report.totalOrders}</p>
        <p>মোট আয়: ৳${report.totalRevenue.toLocaleString('bn-BD')}</p>
        <p>গড় অর্ডার মূল্য: ৳${report.avgOrderValue.toFixed(2)}</p>`;
}
```

```yaml
# serverless.yml — Cron events
functions:
  cleanupSessions:
    handler: cron/cleanup.cleanupSessions
    events:
      - schedule: cron(0 20 * * ? *)  # প্রতিদিন রাত ২টা (UTC+6 → 20:00 UTC)
    timeout: 900  # ১৫ মিনিট (max)

  weeklyReport:
    handler: cron/cleanup.generateWeeklyReport
    events:
      - schedule: cron(0 3 ? * MON *)  # সোমবার সকাল ৯টা (UTC+6)
    timeout: 300
```

### ৭. Serverless Authentication — Custom Authorizer

```javascript
// auth/authorizer.mjs — JWT Custom Authorizer
import { createPublicKey, createVerify } from 'node:crypto';

// JWKS cache — warm start-এ reuse হবে
let cachedKeys = null;

export const handler = async (event) => {
    try {
        const token = extractToken(event.authorizationToken);
        const decoded = await verifyToken(token);

        return generatePolicy(decoded.sub, 'Allow', event.methodArn, {
            userId: decoded.sub,
            email: decoded.email,
            role: decoded.role ?? 'user',
        });
    } catch (err) {
        console.error('Auth failed:', err.message);
        return generatePolicy('anonymous', 'Deny', event.methodArn);
    }
};

function extractToken(authHeader) {
    if (!authHeader?.startsWith('Bearer ')) {
        throw new Error('Missing or invalid Authorization header');
    }
    return authHeader.slice(7);
}

async function verifyToken(token) {
    const [headerB64, payloadB64, signatureB64] = token.split('.');
    const header = JSON.parse(Buffer.from(headerB64, 'base64url').toString());
    const payload = JSON.parse(Buffer.from(payloadB64, 'base64url').toString());

    // Token expiry check
    if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
        throw new Error('Token expired');
    }

    // Issuer validation
    const expectedIssuer = process.env.JWT_ISSUER;
    if (payload.iss !== expectedIssuer) {
        throw new Error('Invalid issuer');
    }

    // JWKS থেকে public key fetch ও signature verify
    const publicKey = await getPublicKey(header.kid);
    const data = `${headerB64}.${payloadB64}`;
    const signature = Buffer.from(signatureB64, 'base64url');

    const verify = createVerify('RSA-SHA256');
    verify.update(data);

    if (!verify.verify(publicKey, signature)) {
        throw new Error('Invalid signature');
    }

    return payload;
}

async function getPublicKey(kid) {
    if (!cachedKeys) {
        const jwksUrl = `${process.env.JWT_ISSUER}/.well-known/jwks.json`;
        const response = await fetch(jwksUrl);
        const { keys } = await response.json();
        cachedKeys = new Map(keys.map(k => [k.kid, k]));
    }

    const jwk = cachedKeys.get(kid);
    if (!jwk) throw new Error(`Key ${kid} not found`);
    return createPublicKey({ key: jwk, format: 'jwk' });
}

function generatePolicy(principalId, effect, resource, context = {}) {
    return {
        principalId,
        policyDocument: {
            Version: '2012-10-17',
            Statement: [{
                Action: 'execute-api:Invoke',
                Effect: effect,
                Resource: resource.replace(/\/[^/]+\/[^/]+$/, '/*'),
            }],
        },
        context,
    };
}
```

```php
<?php
// auth/Authorizer.php — PHP JWT Authorizer
declare(strict_types=1);

final class JwtAuthorizer
{
    private static ?array $cachedKeys = null;

    public function __invoke(array $event): array
    {
        try {
            $token = $this->extractToken($event['authorizationToken'] ?? '');
            $decoded = $this->verifyToken($token);

            return $this->generatePolicy(
                $decoded['sub'],
                'Allow',
                $event['methodArn'],
                [
                    'userId' => $decoded['sub'],
                    'email' => $decoded['email'] ?? '',
                    'role' => $decoded['role'] ?? 'user',
                ]
            );
        } catch (\Throwable $e) {
            error_log("Auth failed: {$e->getMessage()}");
            return $this->generatePolicy('anonymous', 'Deny', $event['methodArn']);
        }
    }

    private function extractToken(string $header): string
    {
        if (!str_starts_with($header, 'Bearer ')) {
            throw new \RuntimeException('Missing Bearer token');
        }
        return substr($header, 7);
    }

    private function verifyToken(string $token): array
    {
        [$headerB64, $payloadB64, $signatureB64] = explode('.', $token);

        $payload = json_decode(
            base64_decode(strtr($payloadB64, '-_', '+/')),
            true,
            flags: JSON_THROW_ON_ERROR
        );

        if (($payload['exp'] ?? 0) < time()) {
            throw new \RuntimeException('Token expired');
        }

        if (($payload['iss'] ?? '') !== $_ENV['JWT_ISSUER']) {
            throw new \RuntimeException('Invalid issuer');
        }

        $header = json_decode(
            base64_decode(strtr($headerB64, '-_', '+/')),
            true,
            flags: JSON_THROW_ON_ERROR
        );

        $publicKey = $this->getPublicKey($header['kid']);
        $data = "{$headerB64}.{$payloadB64}";
        $signature = base64_decode(strtr($signatureB64, '-_', '+/'));

        $valid = openssl_verify($data, $signature, $publicKey, OPENSSL_ALGO_SHA256);
        if ($valid !== 1) {
            throw new \RuntimeException('Invalid signature');
        }

        return $payload;
    }

    private function getPublicKey(string $kid): \OpenSSLAsymmetricKey
    {
        if (self::$cachedKeys === null) {
            $jwksUrl = $_ENV['JWT_ISSUER'] . '/.well-known/jwks.json';
            $jwks = json_decode(file_get_contents($jwksUrl), true);
            self::$cachedKeys = array_column($jwks['keys'], null, 'kid');
        }

        $jwk = self::$cachedKeys[$kid]
            ?? throw new \RuntimeException("Key {$kid} not found");

        $pem = "-----BEGIN PUBLIC KEY-----\n"
            . chunk_split(base64_encode(/* JWK to DER conversion */
                $this->jwkToDer($jwk)), 64, "\n")
            . "-----END PUBLIC KEY-----";

        return openssl_pkey_get_public($pem);
    }

    private function generatePolicy(
        string $principalId,
        string $effect,
        string $resource,
        array $context = []
    ): array {
        return [
            'principalId' => $principalId,
            'policyDocument' => [
                'Version' => '2012-10-17',
                'Statement' => [[
                    'Action' => 'execute-api:Invoke',
                    'Effect' => $effect,
                    'Resource' => preg_replace('#/[^/]+/[^/]+$#', '/*', $resource),
                ]],
            ],
            'context' => $context,
        ];
    }
}
```

### ৮. Cold Start Optimization

Cold start সার্ভারলেসের সবচেয়ে বড় সমস্যা। এটি কমানোর কিছু প্রমাণিত কৌশল:

#### Provisioned Concurrency (serverless.yml)

```yaml
functions:
  criticalApi:
    handler: handler.handler
    provisionedConcurrency: 5  # সবসময় ৫টা warm container ready
    # খরচ: ~$3.50/day per provisioned instance
```

#### Warm-up Strategy (Node.js)

```javascript
// warmup/plugin.mjs — Lambda Warmup (CloudWatch schedule দিয়ে)
export const handler = async (event) => {
    // Warmup event detect করে early return
    if (event.source === 'serverless-warmup') {
        console.log('Warmup invocation — keeping container alive');
        return { statusCode: 200, body: 'Warmed up' };
    }

    // actual business logic
    return await processRequest(event);
};
```

#### Connection Pooling — RDS Proxy ব্যবহার

```javascript
// db/connection.mjs — Lambda-তে DB connection reuse
import mysql from 'mysql2/promise';

// Connection module scope-এ রাখলে warm start-এ reuse হবে
let connection = null;

async function getConnection() {
    if (connection?.connection?._closing === false) {
        return connection; // Warm start — আগের connection পুনরায় ব্যবহার
    }

    // RDS Proxy ব্যবহার করলে connection pooling handle হয়
    connection = await mysql.createConnection({
        host: process.env.RDS_PROXY_ENDPOINT, // RDS Proxy endpoint
        user: process.env.DB_USER,
        database: process.env.DB_NAME,
        ssl: { rejectUnauthorized: true },
        // IAM auth (password-less)
        authPlugins: { mysql_clear_password: () => () => generateRdsAuthToken() },
    });

    return connection;
}

export { getConnection };
```

#### Bundle Size Optimization

```javascript
// esbuild.config.mjs — Node.js bundle ছোট রাখা
import { build } from 'esbuild';

await build({
    entryPoints: ['src/handler.mjs'],
    bundle: true,
    minify: true,
    platform: 'node',
    target: 'node20',
    outfile: 'dist/handler.mjs',
    format: 'esm',
    external: ['@aws-sdk/*'], // Lambda runtime-এ already আছে
    treeShaking: true,
});
// Result: 50KB bundle vs 15MB node_modules
```

**PHP Bref Optimization:**

```yaml
# serverless.yml — PHP-তে cold start কমানো
functions:
  api:
    handler: public/index.php
    runtime: provided.al2023
    layers:
      - ${bref:layer.php-83-fpm}
    environment:
      BREF_AUTOLOAD_PATH: /var/task/vendor/autoload.php
      # OPcache warm রাখা
      PHP_OPCACHE_ENABLE: 1
      PHP_OPCACHE_MAX_ACCELERATED_FILES: 10000
      PHP_OPCACHE_VALIDATE_TIMESTAMPS: 0
```

---

## 💰 Cost Analysis

### Pricing Model

সার্ভারলেসের খরচ তিনটি বিষয়ে নির্ভর করে:

```
মোট খরচ = (Request Count × $0.20/1M) + (GB-seconds × $0.0000166667) + Data Transfer
```

| Component     | AWS Lambda Pricing                        |
|---------------|-------------------------------------------|
| Requests      | $0.20 per 1 million requests              |
| Duration      | $0.0000166667 per GB-second               |
| Free Tier     | 1M requests + 400,000 GB-sec/month        |
| Data Transfer | $0.09/GB (out to internet)                |

### Cost Calculation — বাস্তব উদাহরণ

**bKash Webhook Processing (মাঝারি ই-কমার্স সাইট):**

```
প্রতিদিন ১০,০০০ payment notification
= মাসে ৩,০০,০০০ requests

প্রতিটি execution:
  - Memory: 256 MB
  - Duration: 200ms average

মাসিক খরচ:
  Requests: 300,000 × $0.20/1M = $0.06
  Duration: 300,000 × 0.2s × 0.25GB = 15,000 GB-sec × $0.0000166667 = $0.25

  মোট: ~$0.31/মাস ≈ ৳37/মাস !! 😲

তুলনা — EC2 t3.micro:
  $8.47/মাস = ৳1,016/মাস (24/7 চলবে)
```

**High Traffic সিনারিও (বড় ই-কমার্স):**

```
প্রতিদিন ১০,০০,০০০ API calls
= মাসে ৩ কোটি requests

প্রতিটি execution:
  - Memory: 512 MB
  - Duration: 500ms average

মাসিক খরচ:
  Requests: 30,000,000 × $0.20/1M = $6.00
  Duration: 30,000,000 × 0.5s × 0.5GB = 7,500,000 GB-sec × $0.0000166667 = $125.00

  মোট: ~$131/মাস ≈ ৳15,720/মাস

তুলনা — EC2 (auto-scaling 3-10 instances):
  ~$100-250/মাস (সার্ভার management সহ)
```

### কখন সার্ভারলেস ব্যয়বহুল হয়ে যায়?

- **অনেক বেশি execution time** — যদি function ৫+ সেকেন্ড চলে
- **খুব বেশি memory** — 1GB+ memory-তে cost দ্রুত বাড়ে
- **ক্রমাগত high traffic** — 24/7 constant load-এ EC2 সস্তা
- **অনেক data transfer** — বড় response body-তে data transfer cost বাড়ে

### Cost Optimization Strategy

- **Right-size memory:** Lambda CPU memory-এর সাথে scale করে। কখনো কখনো বেশি memory = কম duration = কম খরচ
- **Avoid unnecessary calls:** caching layer (CloudFront, API Gateway cache) ব্যবহার করুন
- **Arm64 architecture:** Graviton2 ব্যবহারে ২০% সস্তা
- **Reserved Concurrency:** অপ্রয়োজনীয় scaling রোধ করে

---

## 🔒 Security

### IAM — Least Privilege Policy

```yaml
# serverless.yml — শুধু প্রয়োজনীয় permission দেওয়া
provider:
  iam:
    role:
      statements:
        # শুধু নির্দিষ্ট table-এ read/write
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
          Resource:
            - !GetAtt OrdersTable.Arn
        # শুধু নির্দিষ্ট S3 bucket-এ read
        - Effect: Allow
          Action: s3:GetObject
          Resource: arn:aws:s3:::${self:service}-uploads/*
        # কখনোই এটি করবেন না!
        # - Effect: Allow
        #   Action: '*'
        #   Resource: '*'
```

### Environment Variable Encryption

```javascript
// Secrets Manager থেকে runtime-এ secret fetch
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsClient = new SecretsManagerClient({});
let cachedSecret = null; // warm start-এ reuse

async function getSecret(secretId) {
    if (cachedSecret) return cachedSecret;

    const { SecretString } = await secretsClient.send(
        new GetSecretValueCommand({ SecretId: secretId })
    );

    cachedSecret = JSON.parse(SecretString);
    return cachedSecret;
}

export const handler = async (event) => {
    const secrets = await getSecret('prod/bkash/api-keys');
    // secrets.apiKey, secrets.apiSecret ব্যবহার করুন
    // কখনোই environment variable-এ plain text secret রাখবেন না!
};
```

### API Gateway Throttling

```yaml
# serverless.yml — Rate limiting
functions:
  api:
    handler: handler.handler
    events:
      - httpApi:
          path: /api/{proxy+}
          method: ANY
          throttle:
            maxRequestsPerSecond: 100   # burst limit
            maxConcurrentRequests: 50    # steady state

# WAF দিয়ে advanced protection
resources:
  Resources:
    ApiWafAcl:
      Type: AWS::WAFv2::WebACL
      Properties:
        Rules:
          - Name: RateLimitRule
            Priority: 1
            Action: { Block: {} }
            Statement:
              RateBasedStatement:
                Limit: 2000         # ৫ মিনিটে ২০০০ request per IP
                AggregateKeyType: IP
```

---

## ✅ সুবিধা (Pros)

| সুবিধা | বিস্তারিত |
|---------|------------|
| **Zero Server Management** | OS update, security patch, capacity planning কিছুই করতে হয় না |
| **Auto Scaling** | ০ থেকে হাজার request স্বয়ংক্রিয়ভাবে handle করে |
| **Pay-per-Use** | ব্যবহার না করলে কোনো খরচ নেই — startup-দের জন্য আদর্শ |
| **High Availability** | AWS multi-AZ automatically manage করে |
| **Faster Time-to-Market** | Infrastructure নিয়ে চিন্তা না করে সরাসরি business logic-এ focus |
| **Built-in Integration** | AWS services-এর সাথে native integration (S3, SQS, DynamoDB) |

## ❌ অসুবিধা (Cons)

| অসুবিধা | বিস্তারিত |
|----------|------------|
| **Cold Start** | প্রথম request-এ latency বেশি, বিশেষত PHP/Java-তে |
| **Execution Limit** | Maximum ১৫ মিনিট — দীর্ঘ process-এর জন্য উপযুক্ত না |
| **Vendor Lock-in** | AWS Lambda-র কোড Azure-এ সরাসরি চলবে না |
| **Debugging কঠিন** | Distributed system debug করা জটিল, local testing সীমিত |
| **Stateless** | State management নিজে handle করতে হয় |
| **Cost Unpredictability** | Traffic spike-এ হঠাৎ বিল বেড়ে যেতে পারে |
| **Limited Runtime** | সব language/version সমর্থিত না (PHP custom runtime লাগে) |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Serverless Monolith (এক বিশাল Function)

```javascript
// ❌ ভুল — একটি Lambda-তে সবকিছু
export const handler = async (event) => {
    if (event.path === '/users') { /* user logic */ }
    if (event.path === '/orders') { /* order logic */ }
    if (event.path === '/products') { /* product logic */ }
    if (event.path === '/reports') { /* report logic */ }
    // ৫০০০ লাইনের function — cold start বিশাল, deploy ধীর
};

// ✅ সঠিক — আলাদা function
// users/handler.mjs, orders/handler.mjs, products/handler.mjs
```

**সমস্যা:** Bundle size বিশাল হয়ে যায়, cold start বাড়ে, একটি bug পুরো API ভেঙে দেয়। প্রতিটি domain-এর জন্য আলাদা function রাখুন।

### ২. Lambda-to-Lambda Synchronous Calls

```javascript
// ❌ ভুল — Lambda থেকে Lambda সরাসরি invoke
export const handler = async (event) => {
    const lambda = new LambdaClient({});
    const result = await lambda.send(new InvokeCommand({
        FunctionName: 'processOrder',
        Payload: JSON.stringify(event),
    }));
    // ডবল cost! ডবল latency! timeout chain!
};

// ✅ সঠিক — SQS/SNS/EventBridge দিয়ে decouple
export const handler = async (event) => {
    await sqs.send(new SendMessageCommand({
        QueueUrl: process.env.ORDER_QUEUE,
        MessageBody: JSON.stringify(event),
    }));
    return { statusCode: 202, body: 'Accepted' };
};
```

**সমস্যা:** দুইটি Lambda-র execution time আপনার বিলে যোগ হয়। একটি timeout হলে অন্যটিও fail করে। Queue ব্যবহার করে asynchronous করুন।

### ৩. Error Handling ও Dead Letter Queue ছাড়া চালানো

```javascript
// ❌ ভুল — কোনো error handling নেই
export const handler = async (event) => {
    const data = JSON.parse(event.body); // এটি throw করতে পারে!
    await db.putItem(data); // এটিও fail হতে পারে!
};

// ✅ সঠিক — proper error handling + DLQ
export const handler = async (event) => {
    try {
        const data = JSON.parse(event.body);
        if (!data.id) throw new ValidationError('Missing id');
        await db.putItem(data);
        return { statusCode: 200, body: JSON.stringify({ success: true }) };
    } catch (err) {
        if (err instanceof ValidationError) {
            return { statusCode: 400, body: JSON.stringify({ error: err.message }) };
        }
        console.error('Unhandled error:', err);
        throw err; // Lambda retry করবে, তারপর DLQ-তে যাবে
    }
};
```

```yaml
# DLQ configuration
functions:
  processOrder:
    handler: handler.handler
    onError: !GetAtt DeadLetterQueue.Arn
    maximumRetryAttempts: 2
```

### ৪. Monitoring ও Tracing না করা

```yaml
# ✅ X-Ray tracing enable করুন
provider:
  tracing:
    lambda: true
    apiGateway: true

# CloudWatch Alarms
resources:
  Resources:
    LambdaErrorAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmName: ${self:service}-errors
        MetricName: Errors
        Namespace: AWS/Lambda
        Statistic: Sum
        Period: 300
        EvaluationPeriods: 1
        Threshold: 5
        ComparisonOperator: GreaterThanThreshold
        AlarmActions:
          - !Ref AlertTopic
```

### ৫. Vendor Lock-in উপেক্ষা করা

**সমস্যা:** সব কোড AWS-specific SDK-তে লিখলে পরে Azure/GCP-তে migrate করা অত্যন্ত কঠিন।

**সমাধান:** Business logic আলাদা রাখুন, infrastructure code আলাদা। Hexagonal architecture ব্যবহার করুন:

```javascript
// ✅ Port/Adapter pattern — business logic infrastructure-agnostic
// domain/OrderService.mjs
export class OrderService {
    constructor(orderRepo, paymentGateway, notifier) {
        this.orderRepo = orderRepo;        // interface
        this.paymentGateway = paymentGateway; // interface
        this.notifier = notifier;            // interface
    }

    async processOrder(orderData) {
        // Pure business logic — কোনো AWS dependency নেই
        const order = Order.create(orderData);
        await this.paymentGateway.charge(order.total);
        await this.orderRepo.save(order);
        await this.notifier.notify(order);
        return order;
    }
}

// infrastructure/aws/DynamoOrderRepo.mjs — AWS implementation
// infrastructure/azure/CosmosOrderRepo.mjs — Azure implementation
```

---

## 🧪 টেস্টিং সার্ভারলেস

### Unit Testing — Lambda Handler (Jest)

```javascript
// __tests__/handler.test.mjs
import { jest } from '@jest/globals';
import { handler } from '../handler.mjs';

// AWS SDK mock
jest.unstable_mockModule('@aws-sdk/client-dynamodb', () => ({
    DynamoDBClient: jest.fn().mockImplementation(() => ({})),
}));

jest.unstable_mockModule('@aws-sdk/lib-dynamodb', () => ({
    DynamoDBDocumentClient: { from: jest.fn(() => ({ send: mockSend })) },
    PutCommand: jest.fn(),
    GetCommand: jest.fn(),
}));

const mockSend = jest.fn();

describe('bKash Webhook Handler', () => {
    beforeEach(() => mockSend.mockReset());

    test('valid webhook — saves to DynamoDB', async () => {
        mockSend.mockResolvedValue({});

        const event = {
            body: JSON.stringify({
                trxID: 'TRX123',
                amount: 500,
                customerMsisdn: '01712345678',
            }),
            headers: { 'x-bkash-signature': 'valid-signature' },
        };

        const context = { awsRequestId: 'req-123' };
        const result = await handler(event, context);

        expect(result.statusCode).toBe(200);
        expect(mockSend).toHaveBeenCalledTimes(2); // DynamoDB + SNS
    });

    test('invalid signature — returns 401', async () => {
        const event = {
            body: '{}',
            headers: { 'x-bkash-signature': 'invalid' },
        };

        const result = await handler(event, { awsRequestId: 'req-456' });
        expect(result.statusCode).toBe(401);
    });
});
```

### Unit Testing — PHP (PHPUnit)

```php
<?php
// tests/BkashWebhookTest.php
declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;

final class BkashWebhookHandlerTest extends TestCase
{
    private BkashWebhookHandler $handler;
    private MockObject $dynamoDb;
    private MockObject $sns;

    protected function setUp(): void
    {
        $this->dynamoDb = $this->createMock(DynamoDbClient::class);
        $this->sns = $this->createMock(SnsClient::class);

        $this->handler = new BkashWebhookHandler(
            dynamoDb: $this->dynamoDb,
            sns: $this->sns,
        );
    }

    #[Test]
    public function valid_webhook_saves_transaction(): void
    {
        $this->dynamoDb->expects($this->once())
            ->method('putItem')
            ->with($this->callback(function (array $params): bool {
                return $params['Item']['transactionId']['S'] === 'TRX123'
                    && $params['Item']['amount']['N'] === '500';
            }));

        $this->sns->expects($this->once())->method('publish');

        $event = new HttpRequestEvent([
            'body' => json_encode([
                'trxID' => 'TRX123',
                'amount' => 500,
                'customerMsisdn' => '01712345678',
            ]),
            'headers' => ['x-bkash-signature' => $this->generateValidSignature()],
        ]);

        $response = $this->handler->handleRequest($event, new Context());

        $this->assertSame(200, $response->getStatusCode());
    }

    #[Test]
    public function invalid_signature_returns_401(): void
    {
        $event = new HttpRequestEvent([
            'body' => '{}',
            'headers' => ['x-bkash-signature' => 'invalid'],
        ]);

        $response = $this->handler->handleRequest($event, new Context());

        $this->assertSame(401, $response->getStatusCode());
    }
}
```

### Integration Testing ও Local Development

```bash
# SAM CLI দিয়ে locally Lambda invoke
sam local invoke BkashWebhookFunction -e events/bkash-webhook.json

# serverless-offline দিয়ে full local API
npx serverless offline start --stage dev

# Load testing (artillery)
npx artillery run loadtest.yml
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

| Scenario | কারণ |
|----------|-------|
| Webhook processing (bKash, SSL Commerz) | Event-driven, sporadic traffic |
| Image/file processing | CPU-intensive কিন্তু short-lived |
| REST API (low-medium traffic) | Auto-scaling, pay-per-use |
| Cron jobs / scheduled tasks | সবসময় চালু রাখার দরকার নেই |
| IoT data ingestion | Millions of events, unpredictable volume |
| MVP / Startup prototype | Zero upfront cost, দ্রুত launch |
| OTP / SMS verification | Burst traffic, তারপর idle |

### ❌ ব্যবহার করবেন না যখন:

| Scenario | কারণ |
|----------|-------|
| Long-running processes (>15 min) | Lambda timeout limit |
| WebSocket-heavy (gaming) | Persistent connection ভালো মানানসই না |
| ML model training | GPU দরকার, দীর্ঘ execution |
| Legacy monolith migration | শুধু "lift and shift" কাজ করে না |
| Consistent high traffic (24/7) | EC2/ECS তুলনামূলক সস্তা হবে |
| Low-latency requirements (<10ms) | Cold start acceptable না |
| Stateful applications | Session management complex হয়ে যায় |

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| Pattern | সম্পর্ক |
|---------|---------|
| **Microservices** | প্রতিটি Lambda function একটি microservice হতে পারে। Serverless microservices-এর সবচেয়ে lightweight form |
| **Event-Driven Architecture** | সার্ভারলেস স্বাভাবিকভাবেই event-driven। SNS, SQS, EventBridge দিয়ে events route করা হয় |
| **CQRS** | Command (write Lambda) ও Query (read Lambda) আলাদা করা যায়। DynamoDB Streams দিয়ে read model sync |
| **API Gateway Pattern** | API Gateway সার্ভারলেসের entry point হিসেবে কাজ করে — routing, auth, rate limiting |
| **Saga Pattern** | Step Functions দিয়ে distributed transaction manage করা যায়। প্রতিটি step একটি Lambda |
| **Circuit Breaker** | Lambda retry policy + DLQ দিয়ে circuit breaker implement করা যায় |
| **Strangler Fig** | Monolith-এর route গুলো ধীরে ধীরে Lambda-তে migrate করা — progressive modernization |
| **Hexagonal Architecture** | Business logic infrastructure-agnostic রাখা, Lambda handler শুধু adapter |

---

## 📋 সারসংক্ষেপ

```
সার্ভারলেস আর্কিটেকচার — মূল বিষয়গুলো:

┌─────────────────────────────────────────────────────────┐
│  ১. সার্ভার নেই না, সার্ভার ম্যানেজ করতে হয় না        │
│  ২. Event-driven: কিছু ঘটলেই function চলে              │
│  ৩. Pay-per-use: ব্যবহার না করলে কোনো খরচ নেই          │
│  ৪. Auto-scaling: ০ থেকে হাজারে স্বয়ংক্রিয়ভাবে        │
│  ৫. Stateless: প্রতিটি invocation স্বাধীন               │
│  ৬. Cold start সমস্যা: Provisioned Concurrency দিয়ে    │
│     বা warm-up strategy দিয়ে সমাধান করুন               │
│  ৭. ১৫ মিনিটের execution limit মনে রাখুন               │
│  ৮. DLQ (Dead Letter Queue) সবসময় configure করুন      │
│  ৯. Business logic ও infrastructure code আলাদা রাখুন   │
│ ১০. Monitoring (X-Ray, CloudWatch) অবশ্যই ব্যবহার করুন │
└─────────────────────────────────────────────────────────┘

PHP (Bref) vs Node.js:
┌──────────────┬──────────────────┬──────────────────┐
│ বিষয়         │ PHP (Bref)       │ Node.js          │
├──────────────┼──────────────────┼──────────────────┤
│ Cold Start   │ ~800ms           │ ~200-400ms       │
│ Runtime      │ Custom (Bref)    │ Native           │
│ Ecosystem    │ Laravel/Symfony  │ Rich npm          │
│ Best For     │ Existing PHP app │ New serverless    │
│ Concurrency  │ Single-threaded  │ Event loop        │
└──────────────┴──────────────────┴──────────────────┘

মনে রাখুন: সার্ভারলেস সব সমস্যার সমাধান নয়।
সঠিক use case-এ ব্যবহার করলে এটি অত্যন্ত শক্তিশালী,
ভুল জায়গায় ব্যবহার করলে এটি জটিলতা ও খরচ বাড়ায়।
```
