# ⚡ gRPC — সম্পূর্ণ গভীর বিশ্লেষণ (PHP + Node.js)

## 📌 সংজ্ঞা

**gRPC** (gRPC Remote Procedure Call) হলো Google-এর তৈরি একটি ওপেন-সোর্স, উচ্চ-কার্যক্ষম RPC (Remote Procedure Call) ফ্রেমওয়ার্ক। এটি **HTTP/2** প্রোটোকলের উপর ভিত্তি করে তৈরি এবং ডেটা সিরিয়ালাইজেশনের জন্য **Protocol Buffers (protobuf)** ব্যবহার করে।

### মূল ধারণাসমূহ:

| বিষয় | বিবরণ |
|-------|--------|
| **Google RPC** | Google অভ্যন্তরীণভাবে "Stubby" নামে RPC সিস্টেম ব্যবহার করতো। gRPC হলো এর ওপেন-সোর্স সংস্করণ |
| **HTTP/2** | মাল্টিপ্লেক্সিং, হেডার কম্প্রেশন, বাইনারি ফ্রেমিং — একই TCP কানেকশনে একাধিক স্ট্রিম |
| **Protocol Buffers** | ভাষা-নিরপেক্ষ, প্ল্যাটফর্ম-নিরপেক্ষ বাইনারি সিরিয়ালাইজেশন ফরম্যাট। JSON-এর তুলনায় ৩-১০ গুণ দ্রুত |
| **IDL** | Interface Definition Language — `.proto` ফাইলে সার্ভিস ও মেসেজ ডিফাইন করা হয়, তারপর যেকোনো ভাষায় কোড জেনারেট হয় |

### কেন gRPC?

বাংলাদেশের ফিনটেক কোম্পানিগুলোতে (যেমন bKash, Nagad-এর মতো সিস্টেমে) যেখানে প্রতি সেকেন্ডে হাজার হাজার inter-service কল হয়, সেখানে REST API-এর JSON সিরিয়ালাইজেশন/ডিসিরিয়ালাইজেশন overhead অনেক বেশি। gRPC binary protobuf ব্যবহার করে latency কমায় এবং throughput বাড়ায়।

```
REST/JSON payload:  {"user_id": 12345, "name": "রহিম", "balance": 5000.50}  → ~60 bytes
Protobuf payload:   [binary encoded]                                         → ~18 bytes
                                                            সাশ্রয়: ~70%
```

---

## 📊 gRPC Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        gRPC ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐         HTTP/2 (Binary)        ┌──────────────┐  │
│  │  gRPC Client │ ◄════════════════════════════► │  gRPC Server │  │
│  │              │    Multiplexed Streams          │              │  │
│  │  ┌────────┐  │    Header Compression           │  ┌────────┐  │  │
│  │  │  Stub  │  │    Flow Control                 │  │Service │  │  │
│  │  │(Auto-  │  │    Bidirectional                │  │ Impl   │  │  │
│  │  │ Gen)   │  │                                 │  │        │  │  │
│  │  └───┬────┘  │                                 │  └───┬────┘  │  │
│  │      │       │                                 │      │       │  │
│  │  ┌───▼────┐  │                                 │  ┌───▼────┐  │  │
│  │  │Channel │  │                                 │  │Server  │  │  │
│  │  │Manager │  │                                 │  │Runtime │  │  │
│  │  └───┬────┘  │                                 │  └───┬────┘  │  │
│  │      │       │                                 │      │       │  │
│  │  ┌───▼────┐  │                                 │  ┌───▼────┐  │  │
│  │  │Protobuf│  │      ┌──────────────┐           │  │Protobuf│  │  │
│  │  │Encode  │──┼─────►│   Network    │──────────►│  │Decode  │  │  │
│  │  └────────┘  │      │  (TCP/TLS)   │           │  └────────┘  │  │
│  └──────────────┘      └──────────────┘           └──────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    .proto ফাইল (IDL)                        │    │
│  │  ┌─────────┐    ┌──────────┐    ┌──────────┐               │    │
│  │  │ Message │    │ Service  │    │   RPC    │               │    │
│  │  │  Types  │    │  Defn    │    │ Methods  │               │    │
│  │  └─────────┘    └──────────┘    └──────────┘               │    │
│  │         │              │              │                     │    │
│  │         ▼              ▼              ▼                     │    │
│  │  ┌─────────────────────────────────────┐                   │    │
│  │  │        protoc (Code Generator)       │                   │    │
│  │  │   PHP │ JS │ Go │ Java │ Python │..  │                   │    │
│  │  └─────────────────────────────────────┘                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘

═══ কল ফ্লো ═══

Client App                                              Server App
    │                                                        │
    │  1. stub.GetUser(request)                              │
    │  ──────────────────►                                   │
    │  2. Protobuf Serialize                                 │
    │  ──────────────────►                                   │
    │  3. HTTP/2 POST /package.Service/Method                │
    │  ════════════════════════════════════►                  │
    │                        4. Protobuf Deserialize ◄───────│
    │                        5. Service Method Execute        │
    │                        6. Protobuf Serialize Response   │
    │  ◄════════════════════════════════════                  │
    │  7. Protobuf Deserialize                               │
    │  8. Return Response Object                             │
    │  ◄──────────────────                                   │
```

---

## 💻 Core Concepts

### ১. Protocol Buffers

Protocol Buffers (protobuf) হলো gRPC-র মেরুদণ্ড। `.proto` ফাইলে আপনি মেসেজ স্ট্রাকচার এবং সার্ভিস ডিফাইন করেন, তারপর `protoc` কম্পাইলার দিয়ে যেকোনো ভাষায় কোড জেনারেট করেন।

#### .proto ফাইল — মৌলিক কাঠামো

```protobuf
// user.proto
syntax = "proto3";

package fintech;

// ══════ বার্তা (Message) টাইপসমূহ ══════

// স্কেলার টাইপ: int32, int64, float, double, bool, string, bytes
// কম্পোজিট: message, enum, oneof, map, repeated

message User {
    int64 id = 1;                          // ফিল্ড নম্বর ১
    string name = 2;                       // ফিল্ড নম্বর ২
    string email = 3;
    string phone = 4;                      // বাংলাদেশি ফোন: "01712345678"
    UserRole role = 5;
    repeated Address addresses = 6;        // repeated = অ্যারে/লিস্ট
    map<string, string> metadata = 7;      // key-value map
    google.protobuf.Timestamp created_at = 8;

    // oneof — একটিমাত্র ফিল্ড সেট হবে
    oneof verification {
        NIDVerification nid = 9;
        PassportVerification passport = 10;
    }
}

enum UserRole {
    ROLE_UNSPECIFIED = 0;                  // proto3 তে প্রথম enum value অবশ্যই 0
    ROLE_CUSTOMER = 1;
    ROLE_MERCHANT = 2;
    ROLE_AGENT = 3;
    ROLE_ADMIN = 4;
}

message Address {
    string division = 1;                   // ঢাকা, চট্টগ্রাম, রাজশাহী...
    string district = 2;
    string upazila = 3;
    string details = 4;
}

message NIDVerification {
    string nid_number = 1;
    bytes nid_front_image = 2;
    bytes nid_back_image = 3;
}

message PassportVerification {
    string passport_number = 1;
    string country_code = 2;
}

// ══════ সার্ভিস ডেফিনিশন ══════

service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
    rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

    // Server Streaming
    rpc WatchUserActivity(WatchActivityRequest) returns (stream ActivityEvent);

    // Client Streaming
    rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);

    // Bidirectional Streaming
    rpc UserChat(stream ChatMessage) returns (stream ChatMessage);
}

// ══════ রিকোয়েস্ট/রেসপন্স মেসেজ ══════

message GetUserRequest {
    int64 user_id = 1;
}

message GetUserResponse {
    User user = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    string phone = 3;
    UserRole role = 4;
}

message CreateUserResponse {
    User user = 1;
    string message = 2;
}

message UpdateUserRequest {
    int64 user_id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.FieldMask update_mask = 4;  // partial update
}

message UpdateUserResponse {
    User user = 1;
}

message DeleteUserRequest {
    int64 user_id = 1;
}

message DeleteUserResponse {
    bool success = 1;
    string message = 2;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
    string filter = 3;                     // উদাহরণ: "role=MERCHANT AND division=Dhaka"
}

message ListUsersResponse {
    repeated User users = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message WatchActivityRequest {
    int64 user_id = 1;
}

message ActivityEvent {
    string event_type = 1;
    string description = 2;
    google.protobuf.Timestamp timestamp = 3;
}

message BulkCreateResponse {
    int32 created_count = 1;
    repeated string failed_ids = 2;
}

message ChatMessage {
    int64 sender_id = 1;
    int64 receiver_id = 2;
    string content = 3;
    google.protobuf.Timestamp sent_at = 4;
}
```

#### কোড জেনারেশন

```bash
# PHP কোড জেনারেট
protoc --php_out=./generated \
       --grpc_out=./generated \
       --plugin=protoc-gen-grpc=$(which grpc_php_plugin) \
       user.proto

# Node.js — দুটি পদ্ধতি:
# পদ্ধতি ১: Dynamic loading (runtime-এ .proto পড়ে) — সবচেয়ে জনপ্রিয়
# পদ্ধতি ২: Static generation
protoc --js_out=import_style=commonjs,binary:./generated \
       --grpc_out=grpc_js:./generated \
       --plugin=protoc-gen-grpc=$(which grpc_tools_node_protoc_plugin) \
       user.proto
```

---

### ২. Service Types — চার ধরনের RPC

gRPC চারটি মৌলিক RPC প্যাটার্ন সমর্থন করে:

```
┌───────────────────────────────────────────────────────────────┐
│                    gRPC Service Types                         │
├───────────┬──────────────────┬──────────────────────────────┤
│   Type    │    Request       │      Response                │
├───────────┼──────────────────┼──────────────────────────────┤
│  Unary    │  Single Message  │  Single Message              │
│  Server   │  Single Message  │  Stream of Messages          │
│  Client   │  Stream of Msgs  │  Single Message              │
│  BiDi     │  Stream of Msgs  │  Stream of Messages          │
└───────────┴──────────────────┴──────────────────────────────┘
```

#### ক) Unary RPC — একটি রিকোয়েস্ট, একটি রেসপন্স

সবচেয়ে সাধারণ প্যাটার্ন। REST API-র GET/POST-এর মতো।

```protobuf
// user.proto — Unary
service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

**PHP Implementation (Server):**

```php
<?php
// সার্ভার — UserServiceImpl.php
require_once __DIR__ . '/vendor/autoload.php';
require_once __DIR__ . '/generated/GPBMetadata/User.php';
require_once __DIR__ . '/generated/Fintech/UserServiceStub.php';

use Fintech\GetUserRequest;
use Fintech\GetUserResponse;
use Fintech\User;
use Fintech\UserRole;

class UserServiceImpl extends \Fintech\UserServiceStub
{
    private array $users = [];

    public function __construct()
    {
        // ডেমো ডেটা — বাস্তবে ডেটাবেইজ থেকে আসবে
        $this->users[1] = [
            'id' => 1,
            'name' => 'আব্দুর রহিম',
            'email' => 'rahim@example.com',
            'phone' => '01712345678',
            'role' => UserRole::ROLE_CUSTOMER,
        ];
    }

    /**
     * Unary RPC — GetUser
     */
    public function GetUser(
        \Fintech\GetUserRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\GetUserResponse {
        $userId = $request->getUserId();

        if (!isset($this->users[$userId])) {
            // gRPC Status Code NOT_FOUND
            $context->setStatus(
                \Grpc\Status::status(
                    \Grpc\STATUS_NOT_FOUND,
                    "ইউজার পাওয়া যায়নি: ID {$userId}"
                )
            );
            return null;
        }

        $userData = $this->users[$userId];

        $user = new User();
        $user->setId($userData['id']);
        $user->setName($userData['name']);
        $user->setEmail($userData['email']);
        $user->setPhone($userData['phone']);
        $user->setRole($userData['role']);

        $response = new GetUserResponse();
        $response->setUser($user);

        return $response;
    }
}

// সার্ভার চালু করা
$server = new \Grpc\RpcServer();
$server->addHttp2Port('0.0.0.0:50051');
$server->handle(new UserServiceImpl());
echo "gRPC সার্ভার চালু হয়েছে পোর্ট 50051-এ...\n";
$server->run();
```

**PHP Client:**

```php
<?php
// ক্লায়েন্ট — grpc_client.php
require_once __DIR__ . '/vendor/autoload.php';

use Fintech\UserServiceClient;
use Fintech\GetUserRequest;

// Channel তৈরি (insecure — ডেভেলপমেন্টের জন্য)
$client = new UserServiceClient('localhost:50051', [
    'credentials' => \Grpc\ChannelCredentials::createInsecure(),
]);

// Unary কল
$request = new GetUserRequest();
$request->setUserId(1);

// Metadata (অপশনাল হেডার)
$metadata = [
    'authorization' => ['Bearer eyJhbGciOiJSUzI1NiIs...'],
    'x-request-id'  => [uniqid('req_')],
];

// Deadline সেট (৫ সেকেন্ড টাইমআউট)
$options = [
    'timeout' => 5 * 1000000, // মাইক্রোসেকেন্ডে
];

/** @var \Fintech\GetUserResponse $response */
[$response, $status] = $client->GetUser($request, $metadata, $options)->wait();

if ($status->code === \Grpc\STATUS_OK) {
    $user = $response->getUser();
    echo "ইউজার: {$user->getName()}\n";
    echo "ইমেইল: {$user->getEmail()}\n";
    echo "ফোন: {$user->getPhone()}\n";
} else {
    echo "ত্রুটি [{$status->code}]: {$status->details}\n";
}

$client->close();
```

**Node.js Implementation (Server):**

```javascript
// server.js — Unary RPC
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// .proto ফাইল ডাইনামিক লোড
const PROTO_PATH = path.join(__dirname, 'protos', 'user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,       // int64 → string (JS-এ বড় সংখ্যা হারায় না)
    enums: String,
    defaults: true,
    oneofs: true,
});
const fintechProto = grpc.loadPackageDefinition(packageDefinition).fintech;

// ইন-মেমোরি স্টোর (বাস্তবে DB ব্যবহার করবেন)
const users = new Map([
    [1, { id: 1, name: 'আব্দুর রহিম', email: 'rahim@example.com', phone: '01712345678', role: 'ROLE_CUSTOMER' }],
    [2, { id: 2, name: 'করিম উদ্দিন', email: 'karim@example.com', phone: '01812345678', role: 'ROLE_MERCHANT' }],
]);

let nextId = 3;

// ═══════ Unary RPC Handler ═══════
function getUser(call, callback) {
    const userId = parseInt(call.request.user_id);
    const user = users.get(userId);

    if (!user) {
        return callback({
            code: grpc.status.NOT_FOUND,
            message: `ইউজার পাওয়া যায়নি: ID ${userId}`,
        });
    }

    callback(null, { user });
}

function createUser(call, callback) {
    const { name, email, phone, role } = call.request;

    // ভ্যালিডেশন
    if (!name || !phone) {
        return callback({
            code: grpc.status.INVALID_ARGUMENT,
            message: 'নাম এবং ফোন নম্বর আবশ্যক',
        });
    }

    // বাংলাদেশি ফোন নম্বর ভ্যালিডেশন
    if (!/^01[3-9]\d{8}$/.test(phone)) {
        return callback({
            code: grpc.status.INVALID_ARGUMENT,
            message: 'সঠিক বাংলাদেশি ফোন নম্বর দিন (01XXXXXXXXX)',
        });
    }

    const user = { id: nextId++, name, email, phone, role: role || 'ROLE_CUSTOMER' };
    users.set(user.id, user);

    callback(null, { user, message: 'ইউজার সফলভাবে তৈরি হয়েছে' });
}

// সার্ভার চালু
function startServer() {
    const server = new grpc.Server();
    server.addService(fintechProto.UserService.service, {
        GetUser: getUser,
        CreateUser: createUser,
        // অন্যান্য মেথড নিচে যোগ হবে
    });

    const credentials = grpc.ServerCredentials.createInsecure();
    server.bindAsync('0.0.0.0:50051', credentials, (err, port) => {
        if (err) {
            console.error('সার্ভার শুরু করতে ব্যর্থ:', err);
            return;
        }
        console.log(`gRPC সার্ভার চালু হয়েছে পোর্ট ${port}-এ`);
    });
}

startServer();
```

**Node.js Client:**

```javascript
// client.js — Unary RPC
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, 'protos', 'user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
});
const fintechProto = grpc.loadPackageDefinition(packageDefinition).fintech;

// Stub তৈরি
const client = new fintechProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure()
);

// Deadline সেট (৫ সেকেন্ড পরে টাইমআউট)
const deadline = new Date();
deadline.setSeconds(deadline.getSeconds() + 5);

// Metadata
const metadata = new grpc.Metadata();
metadata.add('authorization', 'Bearer eyJhbGciOiJSUzI1NiIs...');
metadata.add('x-request-id', `req_${Date.now()}`);

// Unary কল — Promise wrapper
function getUser(userId) {
    return new Promise((resolve, reject) => {
        client.GetUser(
            { user_id: userId },
            metadata,
            { deadline },
            (err, response) => {
                if (err) return reject(err);
                resolve(response);
            }
        );
    });
}

// ব্যবহার
(async () => {
    try {
        const result = await getUser(1);
        console.log('ইউজার পাওয়া গেছে:', result.user);
    } catch (err) {
        console.error(`ত্রুটি [${err.code}]: ${err.message}`);
    }
})();
```

---

#### খ) Server Streaming RPC — একটি রিকোয়েস্ট, একাধিক রেসপন্স

ক্লায়েন্ট একটি রিকোয়েস্ট পাঠায়, সার্ভার ক্রমাগত ডেটা স্ট্রিম করে। **ব্যবহার:** রিয়েল-টাইম ট্রানজেকশন ফিড, লাইভ ড্যাশবোর্ড।

```protobuf
service UserService {
    rpc WatchUserActivity(WatchActivityRequest) returns (stream ActivityEvent);
}
```

**PHP Server — Server Streaming:**

```php
<?php
// Server Streaming — ইউজার অ্যাক্টিভিটি ওয়াচ
public function WatchUserActivity(
    \Fintech\WatchActivityRequest $request,
    \Grpc\ServerCallWriter $writer,
    \Grpc\ServerContext $context
): void {
    $userId = $request->getUserId();

    if (!isset($this->users[$userId])) {
        $context->setStatus(\Grpc\Status::status(
            \Grpc\STATUS_NOT_FOUND,
            "ইউজার পাওয়া যায়নি"
        ));
        $writer->finish();
        return;
    }

    // সিমুলেটেড ইভেন্ট স্ট্রিম
    $events = [
        ['type' => 'LOGIN', 'desc' => 'অ্যাপে লগইন করেছেন'],
        ['type' => 'BALANCE_CHECK', 'desc' => 'ব্যালেন্স চেক করেছেন'],
        ['type' => 'SEND_MONEY', 'desc' => '৫০০ টাকা পাঠিয়েছেন'],
        ['type' => 'CASHOUT', 'desc' => '১০০০ টাকা ক্যাশ আউট'],
    ];

    foreach ($events as $eventData) {
        // ক্লায়েন্ট ক্যান্সেল করেছে কিনা চেক
        if ($context->isCancelled()) {
            break;
        }

        $event = new \Fintech\ActivityEvent();
        $event->setEventType($eventData['type']);
        $event->setDescription($eventData['desc']);

        $writer->write($event);
        usleep(500000); // ০.৫ সেকেন্ড বিরতি
    }

    $writer->finish();
}
```

**Node.js Server — Server Streaming:**

```javascript
// Server Streaming Handler
function watchUserActivity(call) {
    const userId = parseInt(call.request.user_id);

    if (!users.has(userId)) {
        call.destroy({
            code: grpc.status.NOT_FOUND,
            message: `ইউজার পাওয়া যায়নি: ID ${userId}`,
        });
        return;
    }

    const events = [
        { event_type: 'LOGIN', description: 'অ্যাপে লগইন করেছেন' },
        { event_type: 'BALANCE_CHECK', description: 'ব্যালেন্স চেক করেছেন' },
        { event_type: 'SEND_MONEY', description: '৫০০ টাকা পাঠিয়েছেন' },
        { event_type: 'CASHOUT', description: '১০০০ টাকা ক্যাশ আউট' },
    ];

    let index = 0;
    const interval = setInterval(() => {
        if (index >= events.length || call.cancelled) {
            clearInterval(interval);
            call.end();
            return;
        }

        call.write({
            ...events[index],
            timestamp: { seconds: Math.floor(Date.now() / 1000) },
        });
        index++;
    }, 500);

    // ক্লায়েন্ট ক্যান্সেল করলে ক্লিনআপ
    call.on('cancelled', () => {
        clearInterval(interval);
        console.log('ক্লায়েন্ট স্ট্রিম ক্যান্সেল করেছে');
    });
}
```

**Node.js Client — Server Streaming:**

```javascript
// ক্লায়েন্ট সাইড — Server Streaming গ্রহণ
function watchActivity(userId) {
    const call = client.WatchUserActivity({ user_id: userId }, metadata);

    call.on('data', (event) => {
        console.log(`[${event.event_type}] ${event.description}`);
    });

    call.on('end', () => {
        console.log('স্ট্রিম শেষ হয়েছে');
    });

    call.on('error', (err) => {
        console.error(`স্ট্রিমিং ত্রুটি [${err.code}]: ${err.message}`);
    });

    call.on('status', (status) => {
        console.log('স্ট্যাটাস:', status);
    });

    // ৩০ সেকেন্ড পর ক্যান্সেল
    setTimeout(() => call.cancel(), 30000);
}
```

---

#### গ) Client Streaming RPC — একাধিক রিকোয়েস্ট, একটি রেসপন্স

ক্লায়েন্ট একাধিক মেসেজ স্ট্রিম করে, সার্ভার শেষে একটি রেসপন্স দেয়। **ব্যবহার:** বাল্ক ডেটা আপলোড, ব্যাচ অপারেশন।

```protobuf
service UserService {
    rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);
}
```

**Node.js Server — Client Streaming:**

```javascript
function bulkCreateUsers(call, callback) {
    let createdCount = 0;
    const failedIds = [];

    call.on('data', (request) => {
        const { name, email, phone, role } = request;

        try {
            if (!name || !phone) throw new Error('অসম্পূর্ণ ডেটা');

            const user = { id: nextId++, name, email, phone, role: role || 'ROLE_CUSTOMER' };
            users.set(user.id, user);
            createdCount++;
        } catch (err) {
            failedIds.push(email || 'unknown');
        }
    });

    call.on('end', () => {
        callback(null, {
            created_count: createdCount,
            failed_ids: failedIds,
        });
    });

    call.on('error', (err) => {
        console.error('Client streaming ত্রুটি:', err.message);
    });
}
```

**Node.js Client — Client Streaming:**

```javascript
function bulkCreate(usersList) {
    return new Promise((resolve, reject) => {
        const call = client.BulkCreateUsers(metadata, (err, response) => {
            if (err) return reject(err);
            resolve(response);
        });

        // ক্রমান্বয়ে ইউজার পাঠানো
        for (const user of usersList) {
            call.write({
                name: user.name,
                email: user.email,
                phone: user.phone,
                role: user.role,
            });
        }

        call.end(); // স্ট্রিম শেষ
    });
}

// ব্যবহার
(async () => {
    const result = await bulkCreate([
        { name: 'ফাতেমা খাতুন', email: 'fatema@example.com', phone: '01912345678', role: 'ROLE_CUSTOMER' },
        { name: 'জাহিদ হাসান', email: 'zahid@example.com', phone: '01612345678', role: 'ROLE_MERCHANT' },
        { name: 'নাসরিন আক্তার', email: 'nasrin@example.com', phone: '01512345678', role: 'ROLE_CUSTOMER' },
    ]);
    console.log(`তৈরি হয়েছে: ${result.created_count}, ব্যর্থ: ${result.failed_ids.length}`);
})();
```

---

#### ঘ) Bidirectional Streaming — দুই দিকেই স্ট্রিম

ক্লায়েন্ট ও সার্ভার উভয়ই একসাথে মেসেজ পাঠাতে ও গ্রহণ করতে পারে। **ব্যবহার:** রিয়েল-টাইম চ্যাট, লাইভ ট্রেডিং ফিড।

```protobuf
service UserService {
    rpc UserChat(stream ChatMessage) returns (stream ChatMessage);
}
```

**Node.js Server — BiDi Streaming:**

```javascript
function userChat(call) {
    console.log('BiDi চ্যাট স্ট্রিম শুরু হয়েছে');

    call.on('data', (message) => {
        console.log(`[${message.sender_id} → ${message.receiver_id}]: ${message.content}`);

        // ইকো ব্যাক + অটো রিপ্লাই
        call.write({
            sender_id: message.receiver_id,
            receiver_id: message.sender_id,
            content: `বার্তা পেয়েছি: "${message.content}"`,
            sent_at: { seconds: Math.floor(Date.now() / 1000) },
        });
    });

    call.on('end', () => {
        console.log('চ্যাট স্ট্রিম শেষ');
        call.end();
    });

    call.on('error', (err) => {
        console.error('BiDi ত্রুটি:', err.message);
    });
}
```

**Node.js Client — BiDi Streaming:**

```javascript
function startChat(myId, targetId) {
    const call = client.UserChat(metadata);

    call.on('data', (message) => {
        console.log(`[প্রাপ্ত] ${message.sender_id}: ${message.content}`);
    });

    call.on('end', () => console.log('চ্যাট শেষ'));
    call.on('error', (err) => console.error('চ্যাট ত্রুটি:', err));

    // মেসেজ পাঠানো
    const messages = ['আসসালামু আলাইকুম', 'কেমন আছেন?', 'বিদায়'];
    let i = 0;
    const interval = setInterval(() => {
        if (i >= messages.length) {
            clearInterval(interval);
            call.end();
            return;
        }
        call.write({ sender_id: myId, receiver_id: targetId, content: messages[i++] });
    }, 1000);
}
```

---

### ৩. Channel ও Stub

**Channel** হলো একটি gRPC সার্ভারের সাথে ভার্চুয়াল কানেকশন। অভ্যন্তরীণভাবে এটি HTTP/2 TCP কানেকশন ম্যানেজ করে।

```javascript
// Node.js — Channel অপশন কনফিগারেশন
const channelOptions = {
    // কানেকশন পুলিং ও কিপ-অ্যালাইভ
    'grpc.keepalive_time_ms': 30000,           // প্রতি ৩০ সেকেন্ডে পিং
    'grpc.keepalive_timeout_ms': 5000,          // পিং টাইমআউট ৫ সেকেন্ড
    'grpc.keepalive_permit_without_calls': 1,   // কল ছাড়াও পিং
    'grpc.http2.min_time_between_pings_ms': 10000,

    // মেসেজ সাইজ লিমিট
    'grpc.max_send_message_length': 50 * 1024 * 1024,   // ৫০ MB
    'grpc.max_receive_message_length': 50 * 1024 * 1024,

    // রিট্রাই কনফিগারেশন
    'grpc.service_config': JSON.stringify({
        methodConfig: [{
            name: [{ service: 'fintech.UserService' }],
            retryPolicy: {
                maxAttempts: 3,
                initialBackoff: '0.1s',
                maxBackoff: '1s',
                backoffMultiplier: 2,
                retryableStatusCodes: ['UNAVAILABLE', 'DEADLINE_EXCEEDED'],
            },
        }],
    }),
};

const client = new fintechProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure(),
    channelOptions
);

// Channel state পর্যবেক্ষণ
const channel = client.getChannel();
console.log('Channel state:', channel.getConnectivityState(true));
// IDLE → CONNECTING → READY → TRANSIENT_FAILURE → SHUTDOWN
```

```php
<?php
// PHP — Channel কনফিগারেশন
$client = new \Fintech\UserServiceClient('localhost:50051', [
    'credentials' => \Grpc\ChannelCredentials::createInsecure(),
    'grpc.keepalive_time_ms' => 30000,
    'grpc.keepalive_timeout_ms' => 5000,
    'grpc.max_send_message_length' => 50 * 1024 * 1024,
    'grpc.max_receive_message_length' => 50 * 1024 * 1024,
]);

// Channel কানেক্টিভিটি চেক
$state = $client->getConnectivityState();
echo "Channel State: {$state}\n";
```

---

### ৪. Metadata — হেডার ও ট্রেইলার

gRPC-তে metadata হলো HTTP/2 headers/trailers। অথেনটিকেশন টোকেন, ট্রেসিং আইডি ইত্যাদি পাঠাতে ব্যবহার হয়।

```javascript
// Node.js — Metadata পাঠানো ও গ্রহণ

// ক্লায়েন্ট সাইড
const metadata = new grpc.Metadata();
metadata.add('authorization', 'Bearer <token>');
metadata.add('x-request-id', `req_${Date.now()}`);
metadata.add('x-client-version', '2.1.0');
metadata.add('accept-language', 'bn-BD');  // বাংলা

// Binary metadata (key অবশ্যই '-bin' দিয়ে শেষ হবে)
const binaryData = Buffer.from('binary-payload');
metadata.add('custom-data-bin', binaryData);

// সার্ভার সাইড — metadata পড়া ও initial metadata পাঠানো
function getUser(call, callback) {
    const clientMeta = call.metadata;

    // হেডার থেকে টোকেন পড়া
    const authHeader = clientMeta.get('authorization')[0];
    const requestId = clientMeta.get('x-request-id')[0];

    console.log(`Request ID: ${requestId}, Auth: ${authHeader}`);

    // Initial metadata পাঠানো (হেডার)
    const responseMeta = new grpc.Metadata();
    responseMeta.add('x-response-id', `res_${Date.now()}`);
    responseMeta.add('x-server-region', 'ap-south-1');
    call.sendMetadata(responseMeta);

    // রেসপন্স...
    callback(null, { user: users.get(parseInt(call.request.user_id)) });
}

// ক্লায়েন্ট — response metadata পড়া
const call = client.GetUser({ user_id: 1 }, metadata, (err, response) => {
    if (!err) console.log('User:', response.user);
});
call.on('metadata', (meta) => {
    console.log('Server headers:', meta.getMap());
});
call.on('status', (status) => {
    console.log('Server trailers:', status.metadata.getMap());
});
```

```php
<?php
// PHP — Metadata
$metadata = [
    'authorization' => ['Bearer <token>'],
    'x-request-id'  => [uniqid('req_')],
];

[$response, $status] = $client->GetUser($request, $metadata)->wait();

// Trailing metadata (trailers)
$trailers = $status->metadata;
```

---

### ৫. Error Handling — gRPC স্ট্যাটাস কোড

gRPC নিজস্ব status code সিস্টেম ব্যবহার করে (HTTP status code নয়):

```
┌──────┬──────────────────────┬──────────────────────────────────────┐
│ Code │ নাম                  │ ব্যবহার                              │
├──────┼──────────────────────┼──────────────────────────────────────┤
│  0   │ OK                   │ সফল                                  │
│  1   │ CANCELLED            │ ক্লায়েন্ট ক্যান্সেল করেছে           │
│  2   │ UNKNOWN              │ অজানা ত্রুটি                         │
│  3   │ INVALID_ARGUMENT     │ ভুল ইনপুট (400)                      │
│  4   │ DEADLINE_EXCEEDED    │ টাইমআউট (408)                        │
│  5   │ NOT_FOUND            │ রিসোর্স পাওয়া যায়নি (404)           │
│  6   │ ALREADY_EXISTS       │ ডুপ্লিকেট (409)                      │
│  7   │ PERMISSION_DENIED    │ অনুমতি নেই (403)                     │
│  8   │ RESOURCE_EXHAUSTED   │ রিসোর্স ফুরিয়ে গেছে (429)           │
│  9   │ FAILED_PRECONDITION  │ পূর্বশর্ত পূরণ হয়নি (412)           │
│ 10   │ ABORTED              │ লেনদেন বাতিল (409)                   │
│ 11   │ OUT_OF_RANGE         │ সীমার বাইরে                          │
│ 12   │ UNIMPLEMENTED        │ মেথড বাস্তবায়িত হয়নি (501)         │
│ 13   │ INTERNAL             │ সার্ভার অভ্যন্তরীণ ত্রুটি (500)     │
│ 14   │ UNAVAILABLE          │ সার্ভিস অনুপলব্ধ (503)               │
│ 15   │ DATA_LOSS            │ ডেটা হারানো                          │
│ 16   │ UNAUTHENTICATED      │ অথেনটিকেট হয়নি (401)               │
└──────┴──────────────────────┴──────────────────────────────────────┘
```

```javascript
// Node.js — Rich Error Details (google.rpc.Status)
const { Status } = require('@grpc/grpc-js');

function createUser(call, callback) {
    const { phone, email } = call.request;

    // একাধিক ভ্যালিডেশন ত্রুটি
    const violations = [];
    if (!/^01[3-9]\d{8}$/.test(phone)) {
        violations.push({ field: 'phone', description: 'সঠিক বাংলাদেশি ফোন নম্বর নয়' });
    }
    if (email && !/\S+@\S+\.\S+/.test(email)) {
        violations.push({ field: 'email', description: 'সঠিক ইমেইল নয়' });
    }

    if (violations.length > 0) {
        const meta = new grpc.Metadata();
        meta.add('error-details-bin', Buffer.from(JSON.stringify(violations)));

        return callback({
            code: grpc.status.INVALID_ARGUMENT,
            message: 'ইনপুট ভ্যালিডেশন ব্যর্থ',
            metadata: meta,
        });
    }
    // ... সফল তৈরি
}

// ক্লায়েন্ট সাইড — ত্রুটি হ্যান্ডলিং
client.CreateUser(request, (err, response) => {
    if (err) {
        console.error(`gRPC ত্রুটি [${err.code}]: ${err.message}`);

        switch (err.code) {
            case grpc.status.INVALID_ARGUMENT:
                const details = err.metadata?.get('error-details-bin');
                if (details) {
                    const violations = JSON.parse(details[0]);
                    violations.forEach(v => console.error(`  ✗ ${v.field}: ${v.description}`));
                }
                break;
            case grpc.status.UNAVAILABLE:
                console.log('সার্ভিস অনুপলব্ধ, পুনরায় চেষ্টা করুন');
                break;
            case grpc.status.DEADLINE_EXCEEDED:
                console.log('টাইমআউট — deadline বাড়ান');
                break;
        }
    }
});
```

---

## 🔧 Full Implementation — User Service CRUD

### PHP: সম্পূর্ণ সার্ভার ও ক্লায়েন্ট

```bash
# PHP প্রজেক্ট সেটআপ
composer require grpc/grpc google/protobuf

# gRPC PHP extension ইনস্টল
pecl install grpc
echo "extension=grpc.so" >> php.ini
```

```php
<?php
// php_server.php — সম্পূর্ণ User Service সার্ভার

require_once __DIR__ . '/vendor/autoload.php';

class UserServiceServer extends \Fintech\UserServiceStub
{
    private \PDO $db;

    public function __construct()
    {
        $this->db = new \PDO(
            'mysql:host=localhost;dbname=fintech_grpc;charset=utf8mb4',
            'root', 'secret'
        );
        $this->db->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
    }

    // ═══ GetUser (Unary) ═══
    public function GetUser(
        \Fintech\GetUserRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\GetUserResponse {
        $stmt = $this->db->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$request->getUserId()]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);

        if (!$row) {
            $context->setStatus(\Grpc\Status::status(
                \Grpc\STATUS_NOT_FOUND,
                "ইউজার আইডি {$request->getUserId()} পাওয়া যায়নি"
            ));
            return null;
        }

        return (new \Fintech\GetUserResponse())
            ->setUser($this->rowToUser($row));
    }

    // ═══ CreateUser (Unary) ═══
    public function CreateUser(
        \Fintech\CreateUserRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\CreateUserResponse {
        // ভ্যালিডেশন
        if (empty($request->getName()) || empty($request->getPhone())) {
            $context->setStatus(\Grpc\Status::status(
                \Grpc\STATUS_INVALID_ARGUMENT,
                'নাম এবং ফোন নম্বর আবশ্যক'
            ));
            return null;
        }

        try {
            $stmt = $this->db->prepare(
                'INSERT INTO users (name, email, phone, role) VALUES (?, ?, ?, ?)'
            );
            $stmt->execute([
                $request->getName(),
                $request->getEmail(),
                $request->getPhone(),
                $request->getRole(),
            ]);

            $userId = (int) $this->db->lastInsertId();

            $user = new \Fintech\User();
            $user->setId($userId);
            $user->setName($request->getName());
            $user->setEmail($request->getEmail());
            $user->setPhone($request->getPhone());
            $user->setRole($request->getRole());

            return (new \Fintech\CreateUserResponse())
                ->setUser($user)
                ->setMessage('ইউজার সফলভাবে তৈরি হয়েছে');

        } catch (\PDOException $e) {
            if ($e->getCode() === '23000') {
                $context->setStatus(\Grpc\Status::status(
                    \Grpc\STATUS_ALREADY_EXISTS,
                    'এই ফোন নম্বর বা ইমেইল ইতিমধ্যে নিবন্ধিত'
                ));
                return null;
            }
            throw $e;
        }
    }

    // ═══ UpdateUser (Unary) ═══
    public function UpdateUser(
        \Fintech\UpdateUserRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\UpdateUserResponse {
        $userId = $request->getUserId();
        $updates = [];
        $params = [];

        if (!empty($request->getName())) {
            $updates[] = 'name = ?';
            $params[] = $request->getName();
        }
        if (!empty($request->getEmail())) {
            $updates[] = 'email = ?';
            $params[] = $request->getEmail();
        }

        if (empty($updates)) {
            $context->setStatus(\Grpc\Status::status(
                \Grpc\STATUS_INVALID_ARGUMENT,
                'কমপক্ষে একটি ফিল্ড আপডেট করুন'
            ));
            return null;
        }

        $params[] = $userId;
        $sql = 'UPDATE users SET ' . implode(', ', $updates) . ' WHERE id = ?';
        $stmt = $this->db->prepare($sql);
        $stmt->execute($params);

        if ($stmt->rowCount() === 0) {
            $context->setStatus(\Grpc\Status::status(
                \Grpc\STATUS_NOT_FOUND,
                "ইউজার পাওয়া যায়নি"
            ));
            return null;
        }

        // আপডেটেড ইউজার ফেচ
        $stmt = $this->db->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$userId]);

        return (new \Fintech\UpdateUserResponse())
            ->setUser($this->rowToUser($stmt->fetch(\PDO::FETCH_ASSOC)));
    }

    // ═══ DeleteUser (Unary) ═══
    public function DeleteUser(
        \Fintech\DeleteUserRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\DeleteUserResponse {
        $stmt = $this->db->prepare('DELETE FROM users WHERE id = ?');
        $stmt->execute([$request->getUserId()]);

        $success = $stmt->rowCount() > 0;

        return (new \Fintech\DeleteUserResponse())
            ->setSuccess($success)
            ->setMessage($success ? 'ইউজার মুছে ফেলা হয়েছে' : 'ইউজার পাওয়া যায়নি');
    }

    // ═══ ListUsers (Unary with Pagination) ═══
    public function ListUsers(
        \Fintech\ListUsersRequest $request,
        \Grpc\ServerContext $context
    ): ?\Fintech\ListUsersResponse {
        $pageSize = $request->getPageSize() ?: 20;
        $offset = 0;

        if ($request->getPageToken()) {
            $offset = (int) base64_decode($request->getPageToken());
        }

        $countStmt = $this->db->query('SELECT COUNT(*) FROM users');
        $total = (int) $countStmt->fetchColumn();

        $stmt = $this->db->prepare('SELECT * FROM users LIMIT ? OFFSET ?');
        $stmt->execute([$pageSize, $offset]);
        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);

        $response = new \Fintech\ListUsersResponse();
        $response->setTotalCount($total);

        foreach ($rows as $row) {
            $response->getUsers()[] = $this->rowToUser($row);
        }

        $nextOffset = $offset + $pageSize;
        if ($nextOffset < $total) {
            $response->setNextPageToken(base64_encode((string) $nextOffset));
        }

        return $response;
    }

    private function rowToUser(array $row): \Fintech\User
    {
        $user = new \Fintech\User();
        $user->setId((int) $row['id']);
        $user->setName($row['name']);
        $user->setEmail($row['email'] ?? '');
        $user->setPhone($row['phone']);
        $user->setRole((int) $row['role']);
        return $user;
    }
}

// ═══ সার্ভার বুটস্ট্র্যাপ ═══
$server = new \Grpc\RpcServer();
$server->addHttp2Port('0.0.0.0:50051');
$server->handle(new UserServiceServer());
echo "✅ gRPC PHP সার্ভার চালু: 0.0.0.0:50051\n";
$server->run();
```

### Node.js: সম্পূর্ণ সার্ভার ও ক্লায়েন্ট

```javascript
// node_server.js — সম্পূর্ণ User Service
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, 'protos', 'user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
});
const fintechProto = grpc.loadPackageDefinition(packageDefinition).fintech;

// ═══ ইন-মেমোরি ডেটাবেইজ (বাস্তবে PostgreSQL/MongoDB) ═══
const db = {
    users: new Map([
        [1, { id: 1, name: 'আব্দুর রহিম', email: 'rahim@bkash.com', phone: '01712345678', role: 'ROLE_CUSTOMER' }],
        [2, { id: 2, name: 'করিম উদ্দিন', email: 'karim@nagad.com', phone: '01812345678', role: 'ROLE_MERCHANT' }],
    ]),
    nextId: 3,
};

// ═══ সার্ভিস হ্যান্ডলারসমূহ ═══
const handlers = {
    GetUser(call, callback) {
        const userId = parseInt(call.request.user_id);
        const user = db.users.get(userId);

        if (!user) {
            return callback({
                code: grpc.status.NOT_FOUND,
                message: `ইউজার পাওয়া যায়নি: ID ${userId}`,
            });
        }
        callback(null, { user });
    },

    CreateUser(call, callback) {
        const { name, email, phone, role } = call.request;

        if (!name || !phone) {
            return callback({
                code: grpc.status.INVALID_ARGUMENT,
                message: 'নাম এবং ফোন নম্বর আবশ্যক',
            });
        }

        // ডুপ্লিকেট ফোন চেক
        for (const [, u] of db.users) {
            if (u.phone === phone) {
                return callback({
                    code: grpc.status.ALREADY_EXISTS,
                    message: `ফোন নম্বর ${phone} ইতিমধ্যে নিবন্ধিত`,
                });
            }
        }

        const user = { id: db.nextId++, name, email, phone, role: role || 'ROLE_CUSTOMER' };
        db.users.set(user.id, user);
        callback(null, { user, message: 'ইউজার সফলভাবে তৈরি হয়েছে' });
    },

    UpdateUser(call, callback) {
        const userId = parseInt(call.request.user_id);
        const existing = db.users.get(userId);

        if (!existing) {
            return callback({ code: grpc.status.NOT_FOUND, message: 'ইউজার পাওয়া যায়নি' });
        }

        const { name, email } = call.request;
        if (name) existing.name = name;
        if (email) existing.email = email;

        db.users.set(userId, existing);
        callback(null, { user: existing });
    },

    DeleteUser(call, callback) {
        const userId = parseInt(call.request.user_id);
        const existed = db.users.delete(userId);

        callback(null, {
            success: existed,
            message: existed ? 'ইউজার মুছে ফেলা হয়েছে' : 'ইউজার পাওয়া যায়নি',
        });
    },

    ListUsers(call, callback) {
        const pageSize = call.request.page_size || 20;
        let offset = 0;

        if (call.request.page_token) {
            offset = parseInt(Buffer.from(call.request.page_token, 'base64').toString());
        }

        const allUsers = [...db.users.values()];
        const pageUsers = allUsers.slice(offset, offset + pageSize);
        const nextOffset = offset + pageSize;

        callback(null, {
            users: pageUsers,
            total_count: allUsers.length,
            next_page_token: nextOffset < allUsers.length
                ? Buffer.from(String(nextOffset)).toString('base64')
                : '',
        });
    },

    // Server Streaming
    WatchUserActivity(call) {
        const userId = parseInt(call.request.user_id);
        if (!db.users.has(userId)) {
            call.destroy({ code: grpc.status.NOT_FOUND, message: 'ইউজার পাওয়া যায়নি' });
            return;
        }

        const events = [
            { event_type: 'LOGIN', description: 'অ্যাপে লগইন' },
            { event_type: 'SEND_MONEY', description: '৫০০ টাকা পাঠিয়েছেন' },
            { event_type: 'PAYMENT', description: 'বিদ্যুৎ বিল পরিশোধ' },
        ];

        let i = 0;
        const interval = setInterval(() => {
            if (i >= events.length || call.cancelled) {
                clearInterval(interval);
                call.end();
                return;
            }
            call.write({ ...events[i++], timestamp: { seconds: Math.floor(Date.now() / 1000) } });
        }, 1000);

        call.on('cancelled', () => clearInterval(interval));
    },

    // Client Streaming
    BulkCreateUsers(call, callback) {
        let created = 0;
        const failed = [];

        call.on('data', (req) => {
            try {
                if (!req.name || !req.phone) throw new Error('অসম্পূর্ণ');
                const user = { id: db.nextId++, ...req, role: req.role || 'ROLE_CUSTOMER' };
                db.users.set(user.id, user);
                created++;
            } catch {
                failed.push(req.email || 'unknown');
            }
        });

        call.on('end', () => callback(null, { created_count: created, failed_ids: failed }));
    },

    // BiDi Streaming
    UserChat(call) {
        call.on('data', (msg) => {
            call.write({
                sender_id: msg.receiver_id,
                receiver_id: msg.sender_id,
                content: `প্রাপ্ত: "${msg.content}"`,
                sent_at: { seconds: Math.floor(Date.now() / 1000) },
            });
        });
        call.on('end', () => call.end());
    },
};

// ═══ সার্ভার শুরু ═══
function main() {
    const server = new grpc.Server({
        'grpc.max_receive_message_length': 50 * 1024 * 1024,
        'grpc.max_send_message_length': 50 * 1024 * 1024,
    });

    server.addService(fintechProto.UserService.service, handlers);

    server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), (err, port) => {
        if (err) {
            console.error('সার্ভার ত্রুটি:', err);
            process.exit(1);
        }
        console.log(`✅ gRPC Node.js সার্ভার চালু: 0.0.0.0:${port}`);
    });
}

main();
```

```javascript
// node_client.js — সম্পূর্ণ ক্লায়েন্ট
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, 'protos', 'user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true, longs: String, enums: String, defaults: true, oneofs: true,
});
const fintechProto = grpc.loadPackageDefinition(packageDefinition).fintech;

const client = new fintechProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure(),
    { 'grpc.keepalive_time_ms': 30000 }
);

// ═══ Promise-ভিত্তিক Wrapper ═══
function promisify(method) {
    return (request, metadata = new grpc.Metadata()) => {
        const deadline = new Date();
        deadline.setSeconds(deadline.getSeconds() + 10);

        return new Promise((resolve, reject) => {
            method.call(client, request, metadata, { deadline }, (err, res) => {
                if (err) reject(err);
                else resolve(res);
            });
        });
    };
}

const rpc = {
    getUser: promisify(client.GetUser),
    createUser: promisify(client.CreateUser),
    updateUser: promisify(client.UpdateUser),
    deleteUser: promisify(client.DeleteUser),
    listUsers: promisify(client.ListUsers),
};

// ═══ ব্যবহার ═══
(async () => {
    try {
        // তৈরি
        const created = await rpc.createUser({
            name: 'সালমা বেগম', email: 'salma@example.com',
            phone: '01912345678', role: 'ROLE_CUSTOMER',
        });
        console.log('তৈরি:', created.user);

        // পড়া
        const fetched = await rpc.getUser({ user_id: created.user.id });
        console.log('পড়া:', fetched.user);

        // আপডেট
        const updated = await rpc.updateUser({ user_id: created.user.id, name: 'সালমা আক্তার' });
        console.log('আপডেট:', updated.user);

        // তালিকা
        const list = await rpc.listUsers({ page_size: 10 });
        console.log(`মোট ইউজার: ${list.total_count}`);
        list.users.forEach(u => console.log(`  - ${u.name} (${u.phone})`));

        // মুছে ফেলা
        const deleted = await rpc.deleteUser({ user_id: created.user.id });
        console.log('মুছে ফেলা:', deleted.message);

    } catch (err) {
        console.error(`ত্রুটি [${err.code}]: ${err.message}`);
    }
})();
```

---

## 🔥 Advanced Topics

### ১. gRPC vs REST vs GraphQL

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ বৈশিষ্ট্য        │ gRPC             │ REST             │ GraphQL          │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ প্রোটোকল         │ HTTP/2           │ HTTP/1.1 বা 2    │ HTTP/1.1 বা 2    │
│ ডেটা ফরম্যাট     │ Protobuf(Binary) │ JSON/XML(Text)   │ JSON(Text)       │
│ কন্ট্র্যাক্ট     │ .proto (strict)  │ OpenAPI(loose)   │ Schema(strict)   │
│ কোড জেনারেশন     │ ✅ বিল্ট-ইন       │ ❌ আলাদা টুল     │ ⚠️ আলাদা টুল    │
│ স্ট্রিমিং        │ ✅ ৪ ধরনের        │ ❌ SSE/WebSocket │ ⚠️ Subscription  │
│ ব্রাউজার সাপোর্ট │ ⚠️ gRPC-Web      │ ✅ নেটিভ         │ ✅ নেটিভ         │
│ পারফরম্যান্স      │ ⭐⭐⭐⭐⭐        │ ⭐⭐⭐           │ ⭐⭐⭐⭐         │
│ Latency          │ খুবই কম          │ মাঝারি            │ মাঝারি            │
│ পেলোড সাইজ       │ ছোট (binary)     │ বড় (text)        │ মাঝারি            │
│ শেখার বক্ররেখা    │ 🔺 খাড়া          │ 🔻 সমতল          │ 🔺 মাঝারি        │
│ ডিবাগিং          │ কঠিন(binary)     │ সহজ(readable)    │ মাঝারি            │
│ মাইক্রোসার্ভিস   │ ⭐⭐⭐⭐⭐        │ ⭐⭐⭐           │ ⭐⭐⭐⭐         │
│ মোবাইল/IoT       │ ⭐⭐⭐⭐⭐        │ ⭐⭐⭐           │ ⭐⭐⭐⭐         │
│ ব্যবহার ক্ষেত্র  │ Internal service │ Public API       │ Flexible query   │
│                  │ communication    │ Web/Mobile       │ Frontend-driven  │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘

📌 বাংলাদেশ প্রেক্ষাপট:
  ► ফিনটেক (bKash/Nagad টাইপ): Internal → gRPC, External → REST
  ► ই-কমার্স (Daraz টাইপ): BFF → GraphQL, Backend → gRPC
  ► স্টার্টআপ: REST দিয়ে শুরু, স্কেল হলে gRPC
```

---

### ২. gRPC-Web — ব্রাউজার সাপোর্ট

ব্রাউজার সরাসরি HTTP/2 framing অ্যাক্সেস করতে পারে না। gRPC-Web একটি অ্যাডাপ্টেশন লেয়ার যা Envoy proxy ব্যবহার করে gRPC কলকে ব্রাউজার-সামঞ্জস্যপূর্ণ করে।

```
ব্রাউজার ──(gRPC-Web/HTTP1.1)──► Envoy Proxy ──(gRPC/HTTP2)──► gRPC Server
```

```yaml
# envoy.yaml — gRPC-Web proxy কনফিগারেশন
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: auto
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route:
                            cluster: grpc_service
                            timeout: 30s
                            max_stream_duration:
                              grpc_timeout_header_max: 30s
                      cors:
                        allow_origin_string_match:
                          - prefix: "*"
                        allow_methods: GET, PUT, DELETE, POST, OPTIONS
                        allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,x-custom-header,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout,authorization
                        expose_headers: grpc-status,grpc-message
                http_filters:
                  - name: envoy.filters.http.grpc_web
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                  - name: envoy.filters.http.cors
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
    - name: grpc_service
      connect_timeout: 0.25s
      type: logical_dns
      http2_protocol_options: {}
      lb_policy: round_robin
      load_assignment:
        cluster_name: cluster_0
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: grpc-server, port_value: 50051 }
```

---

### ৩. Load Balancing gRPC

gRPC HTTP/2 ব্যবহার করে বলে সাধারণ L4 (TCP) load balancer সমস্যা সৃষ্টি করে — একটি persistent connection-এ সব রিকোয়েস্ট একই সার্ভারে যায়। **L7 load balancing** বা **client-side load balancing** দরকার।

```
═══ L4 vs L7 Load Balancing ═══

L4 (TCP) — সমস্যা:
  Client ─── TCP Connection ──► LB ──── TCP ──── Server A (সব রিকোয়েস্ট এখানে!)
                                                  Server B (কিছু পায় না)

L7 (HTTP/2) — সমাধান:
  Client ─── HTTP/2 Stream ──► LB ──── Stream 1 ──► Server A
                                  ├─── Stream 2 ──► Server B
                                  └─── Stream 3 ──► Server A

Client-side LB:
  Client ──┬── Connection ──► Server A
            ├── Connection ──► Server B
            └── Connection ──► Server C
            (round-robin / weighted / pick-first)
```

```javascript
// Node.js — Client-side Load Balancing
const client = new fintechProto.UserService(
    // DNS resolver ব্যবহার করে একাধিক সার্ভার আবিষ্কার
    'dns:///grpc-service.fintech.local:50051',
    grpc.credentials.createInsecure(),
    {
        // Round-robin load balancing
        'grpc.service_config': JSON.stringify({
            loadBalancingConfig: [{ round_robin: {} }],
        }),
        // DNS resolution interval
        'grpc.dns_min_time_between_resolutions_ms': 15000,
    }
);
```

---

### ৪. Authentication — TLS ও টোকেন-ভিত্তিক

```javascript
// ═══ TLS সার্টিফিকেট দিয়ে সিকিউর সার্ভার ═══
const fs = require('fs');

// সার্ভার সাইড — mTLS (Mutual TLS)
const serverCredentials = grpc.ServerCredentials.createSsl(
    fs.readFileSync('certs/ca.crt'),                // CA সার্টিফিকেট
    [{
        cert_chain: fs.readFileSync('certs/server.crt'),  // সার্ভার সার্টিফিকেট
        private_key: fs.readFileSync('certs/server.key'), // সার্ভার প্রাইভেট কী
    }],
    true  // ক্লায়েন্ট সার্টিফিকেট চেক (mTLS)
);

server.bindAsync('0.0.0.0:50051', serverCredentials, (err, port) => {
    console.log(`Secure gRPC সার্ভার চালু: ${port}`);
});

// ক্লায়েন্ট সাইড — TLS
const channelCredentials = grpc.credentials.createSsl(
    fs.readFileSync('certs/ca.crt'),
    fs.readFileSync('certs/client.key'),
    fs.readFileSync('certs/client.crt')
);

const client = new fintechProto.UserService('grpc.fintech.com:443', channelCredentials);

// ═══ টোকেন-ভিত্তিক Auth (JWT) ═══
// কল-লেভেল credentials
const callCredentials = grpc.credentials.createFromMetadataGenerator(
    (params, callback) => {
        const metadata = new grpc.Metadata();
        metadata.add('authorization', `Bearer ${getJwtToken()}`);
        callback(null, metadata);
    }
);

// Channel + Call credentials একত্রিত
const combinedCredentials = grpc.credentials.combineChannelCredentials(
    channelCredentials,
    callCredentials
);

const secureClient = new fintechProto.UserService('grpc.fintech.com:443', combinedCredentials);
```

---

### ৫. Interceptors/Middleware

Interceptor হলো gRPC-র middleware। প্রতিটি RPC কলের আগে ও পরে কোড চালাতে পারেন — লগিং, auth চেক, retry, metrics।

```javascript
// ═══ Node.js Server Interceptors ═══

// লগিং Interceptor
function loggingInterceptor(methodDescriptor, call) {
    const startTime = Date.now();
    const method = methodDescriptor.path;
    const requestId = call.metadata.get('x-request-id')[0] || 'unknown';

    console.log(`[→ REQUEST] ${method} | ID: ${requestId}`);

    // Original handler-এ ডেলিগেট
    const listener = new grpc.ServerListenerBuilder()
        .withOnReceiveMessage((message, next) => {
            console.log(`[📦 MESSAGE] ${method} | Payload size: ${JSON.stringify(message).length}`);
            next(message);
        })
        .build();

    const responder = new grpc.ResponderBuilder()
        .withStart((metadata, next) => {
            next(metadata);
        })
        .withSendMessage((message, next) => {
            const duration = Date.now() - startTime;
            console.log(`[← RESPONSE] ${method} | ${duration}ms`);
            next(message);
        })
        .withSendStatus((status, next) => {
            const duration = Date.now() - startTime;
            console.log(`[✓ STATUS] ${method} | Code: ${status.code} | ${duration}ms`);
            next(status);
        })
        .build();

    return { listener, responder };
}

// Auth Interceptor
function authInterceptor(methodDescriptor, call) {
    const publicMethods = ['/fintech.UserService/HealthCheck'];

    if (publicMethods.includes(methodDescriptor.path)) {
        return call; // পাবলিক মেথড — auth skip
    }

    const token = call.metadata.get('authorization')[0];
    if (!token || !token.startsWith('Bearer ')) {
        const status = {
            code: grpc.status.UNAUTHENTICATED,
            details: 'অথেনটিকেশন টোকেন প্রয়োজন',
        };
        call.sendStatus(status);
        return null;
    }

    try {
        const decoded = verifyJwt(token.replace('Bearer ', ''));
        // ইউজার ইনফো metadata-তে যোগ
        call.metadata.add('x-user-id', String(decoded.userId));
        call.metadata.add('x-user-role', decoded.role);
    } catch {
        call.sendStatus({
            code: grpc.status.UNAUTHENTICATED,
            details: 'অবৈধ বা মেয়াদোত্তীর্ণ টোকেন',
        });
        return null;
    }

    return call;
}

// সার্ভারে Interceptor যোগ
const server = new grpc.Server({
    interceptors: [loggingInterceptor, authInterceptor],
});

// ═══ Client Interceptor (Retry + Timeout) ═══
function retryInterceptor(options, nextCall) {
    let savedMessage, savedMetadata;
    const maxRetries = 3;
    let retryCount = 0;

    const requester = new grpc.RequesterBuilder()
        .withStart((metadata, listener, next) => {
            savedMetadata = metadata;
            const newListener = new grpc.ListenerBuilder()
                .withOnReceiveStatus((status, next) => {
                    if (
                        retryCount < maxRetries &&
                        [grpc.status.UNAVAILABLE, grpc.status.DEADLINE_EXCEEDED].includes(status.code)
                    ) {
                        retryCount++;
                        const delay = Math.pow(2, retryCount) * 100; // Exponential backoff
                        console.log(`🔄 পুনরায় চেষ্টা ${retryCount}/${maxRetries} (${delay}ms পর)`);
                        setTimeout(() => {
                            const newCall = nextCall(options);
                            newCall.start(savedMetadata, listener);
                            newCall.sendMessage(savedMessage);
                            newCall.halfClose();
                        }, delay);
                    } else {
                        next(status);
                    }
                })
                .build();
            next(metadata, newListener);
        })
        .withSendMessage((message, next) => {
            savedMessage = message;
            next(message);
        })
        .build();

    return new grpc.InterceptingCall(nextCall(options), requester);
}

// ক্লায়েন্টে Interceptor ব্যবহার
const client = new fintechProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure(),
    { interceptors: [retryInterceptor] }
);
```

```php
<?php
// PHP — Interceptor (Client Middleware)
// PHP gRPC-তে interceptor ডিরেক্টলি সমর্থিত নয়, wrapper pattern ব্যবহার করুন

class GrpcClientWrapper
{
    private \Fintech\UserServiceClient $client;
    private string $authToken;

    public function __construct(string $host, string $token)
    {
        $this->client = new \Fintech\UserServiceClient($host, [
            'credentials' => \Grpc\ChannelCredentials::createInsecure(),
        ]);
        $this->authToken = $token;
    }

    private function getMetadata(): array
    {
        return [
            'authorization' => ["Bearer {$this->authToken}"],
            'x-request-id'  => [uniqid('req_')],
            'x-timestamp'   => [(string) microtime(true)],
        ];
    }

    private function getOptions(int $timeoutSeconds = 5): array
    {
        return ['timeout' => $timeoutSeconds * 1000000];
    }

    public function getUser(int $userId): ?\Fintech\User
    {
        $request = new \Fintech\GetUserRequest();
        $request->setUserId($userId);

        $startTime = microtime(true);

        [$response, $status] = $this->client->GetUser(
            $request, $this->getMetadata(), $this->getOptions()
        )->wait();

        $duration = round((microtime(true) - $startTime) * 1000, 2);
        $this->log("GetUser", $status->code, $duration);

        if ($status->code !== \Grpc\STATUS_OK) {
            throw new \RuntimeException("[{$status->code}] {$status->details}");
        }

        return $response->getUser();
    }

    private function log(string $method, int $code, float $durationMs): void
    {
        error_log("[gRPC] {$method} | code={$code} | {$durationMs}ms");
    }
}
```

---

### ৬. Health Checking Protocol

gRPC-র একটি স্ট্যান্ডার্ড health checking প্রোটোকল আছে যা Kubernetes, Consul-এর সাথে কাজ করে:

```protobuf
// health.proto (gRPC standard)
syntax = "proto3";
package grpc.health.v1;

message HealthCheckRequest {
    string service = 1;
}

message HealthCheckResponse {
    enum ServingStatus {
        UNKNOWN = 0;
        SERVING = 1;
        NOT_SERVING = 2;
        SERVICE_UNKNOWN = 3;
    }
    ServingStatus status = 1;
}

service Health {
    rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

```javascript
// Node.js — Health Check Implementation
const grpcHealth = require('grpc-health-check');

const statusMap = {
    'fintech.UserService': grpcHealth.servingStatus.SERVING,
    '': grpcHealth.servingStatus.SERVING, // overall server status
};

const healthImpl = new grpcHealth.Implementation(statusMap);

server.addService(grpcHealth.service, healthImpl);

// ডাইনামিক স্ট্যাটাস পরিবর্তন
// ডেটাবেইজ ডাউন হলে:
healthImpl.setStatus('fintech.UserService', grpcHealth.servingStatus.NOT_SERVING);
// পুনরুদ্ধার হলে:
healthImpl.setStatus('fintech.UserService', grpcHealth.servingStatus.SERVING);
```

```bash
# grpcurl দিয়ে health check
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check

# Kubernetes liveness/readiness probe
# k8s deployment yaml:
# livenessProbe:
#   grpc:
#     port: 50051
#   initialDelaySeconds: 10
#   periodSeconds: 10
```

---

### ৭. gRPC in Microservices Communication

বাংলাদেশের ফিনটেক সিস্টেমে মাইক্রোসার্ভিসের মধ্যে gRPC ব্যবহারের একটি বাস্তবসম্মত আর্কিটেকচার:

```
┌──────────────────────────────────────────────────────────────────┐
│                  ফিনটেক মাইক্রোসার্ভিস আর্কিটেকচার              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Mobile App ──(REST/GraphQL)──► API Gateway (Kong/Envoy)         │
│  Web App   ──(gRPC-Web)──────►      │                            │
│                                      │                            │
│                              ┌───────┴───────┐                   │
│                              │  gRPC (HTTP/2) │                   │
│                              └───────┬───────┘                   │
│                                      │                            │
│         ┌────────────────────────────┼──────────────────┐        │
│         │                            │                  │        │
│   ┌─────▼─────┐    ┌───────▼──────┐  ┌────▼─────┐      │        │
│   │ User Svc  │◄──►│ Wallet Svc   │  │ Notif Svc│      │        │
│   │ (Node.js) │    │ (Go)         │  │ (Node.js)│      │        │
│   │ :50051    │    │ :50052       │  │ :50053   │      │        │
│   └─────┬─────┘    └──────┬───────┘  └──────────┘      │        │
│         │                  │                             │        │
│   ┌─────▼─────┐    ┌──────▼───────┐  ┌──────────┐      │        │
│   │ PostgreSQL│    │    Redis     │  │ Kafka    │      │        │
│   │ (Users)   │    │  (Balance)   │  │ (Events) │      │        │
│   └───────────┘    └──────────────┘  └──────────┘      │        │
│                                                         │        │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────┐      │        │
│   │ KYC Svc    │  │ Txn Svc     │  │ Report   │      │        │
│   │ (PHP)      │  │ (Go)        │  │ Svc(PHP) │      │        │
│   │ :50054     │  │ :50055      │  │ :50056   │      │        │
│   └─────────────┘  └──────────────┘  └──────────┘      │        │
│                                                         │        │
│   Service Discovery: Consul / Kubernetes DNS            │        │
│   Observability: Jaeger(Tracing) + Prometheus(Metrics)  │        │
└──────────────────────────────────────────────────────────────────┘
```

```javascript
// সার্ভিস-থেকে-সার্ভিস কল (User Svc → Wallet Svc)
// wallet.proto
// service WalletService {
//     rpc GetBalance(GetBalanceRequest) returns (GetBalanceResponse);
//     rpc Transfer(TransferRequest) returns (TransferResponse);
// }

class UserServiceWithWallet {
    constructor() {
        // অন্য সার্ভিসের ক্লায়েন্ট
        this.walletClient = new walletProto.WalletService(
            'wallet-svc.fintech.local:50052',
            grpc.credentials.createInsecure(),
            {
                'grpc.service_config': JSON.stringify({
                    loadBalancingConfig: [{ round_robin: {} }],
                    methodConfig: [{
                        name: [{ service: 'fintech.WalletService' }],
                        timeout: '5s',
                        retryPolicy: {
                            maxAttempts: 3,
                            initialBackoff: '0.1s',
                            maxBackoff: '1s',
                            backoffMultiplier: 2,
                            retryableStatusCodes: ['UNAVAILABLE'],
                        },
                    }],
                }),
            }
        );
    }

    async getUserWithBalance(call, callback) {
        const userId = parseInt(call.request.user_id);
        const user = db.users.get(userId);
        if (!user) return callback({ code: grpc.status.NOT_FOUND, message: 'ইউজার নেই' });

        // Wallet সার্ভিসে কল — deadline propagation
        const deadline = call.getDeadline();
        this.walletClient.GetBalance(
            { user_id: userId },
            new grpc.Metadata(),
            { deadline },
            (err, balanceRes) => {
                if (err) {
                    console.error('Wallet সার্ভিস ত্রুটি:', err.message);
                    user.balance = 'unavailable';
                } else {
                    user.balance = balanceRes.balance;
                }
                callback(null, { user });
            }
        );
    }
}
```

---

### ৮. Deadline/Timeout Handling

Deadline propagation মাইক্রোসার্ভিসে অত্যন্ত গুরুত্বপূর্ণ — চেইনের প্রতিটি সার্ভিস যেন জানে কতক্ষণ বাকি আছে।

```javascript
// ═══ Deadline Propagation ═══

// ক্লায়েন্ট — ৫ সেকেন্ড deadline সেট
const deadline = new Date();
deadline.setSeconds(deadline.getSeconds() + 5);

client.GetUser({ user_id: 1 }, metadata, { deadline }, (err, res) => {
    if (err && err.code === grpc.status.DEADLINE_EXCEEDED) {
        console.error('⏰ টাইমআউট! সার্ভার ৫ সেকেন্ডে রেসপন্স দিতে পারেনি');
    }
});

// সার্ভার — remaining deadline চেক ও propagation
function getUser(call, callback) {
    const deadline = call.getDeadline();
    const remainingMs = deadline ? deadline.getTime() - Date.now() : Infinity;

    if (remainingMs <= 0) {
        return callback({
            code: grpc.status.DEADLINE_EXCEEDED,
            message: 'Deadline ইতিমধ্যে পেরিয়ে গেছে',
        });
    }

    console.log(`বাকি সময়: ${remainingMs}ms`);

    // downstream সার্ভিসে কল করলে deadline propagate করুন
    const downstreamDeadline = new Date(Date.now() + Math.min(remainingMs - 100, 3000));
    // 100ms বাফার রাখা হয়েছে নিজের কাজের জন্য

    walletClient.GetBalance(
        { user_id: call.request.user_id },
        new grpc.Metadata(),
        { deadline: downstreamDeadline },
        (err, balance) => {
            // ...
        }
    );
}
```

```php
<?php
// PHP — Deadline
$deadline = microtime(true) + 5.0; // ৫ সেকেন্ড
$options = ['timeout' => 5 * 1000000]; // মাইক্রোসেকেন্ডে

[$response, $status] = $client->GetUser($request, $metadata, $options)->wait();

if ($status->code === \Grpc\STATUS_DEADLINE_EXCEEDED) {
    echo "⏰ টাইমআউট!\n";
}
```

---

### ৯. Reflection ও Server Listing

gRPC reflection সার্ভিস আবিষ্কার ও ডিবাগিং-এর জন্য অত্যন্ত উপযোগী:

```javascript
// Node.js — Reflection সক্রিয় করা
const { ReflectionService } = require('@grpc/reflection');

const reflection = new ReflectionService(packageDefinition);
reflection.addToServer(server);

// এখন grpcurl, grpc_cli, Postman ইত্যাদি দিয়ে সার্ভিস আবিষ্কার করা যাবে
```

```bash
# grpcurl দিয়ে সার্ভিস তালিকা
grpcurl -plaintext localhost:50051 list
# Output:
# fintech.UserService
# grpc.health.v1.Health
# grpc.reflection.v1alpha.ServerReflection

# মেথড তালিকা
grpcurl -plaintext localhost:50051 list fintech.UserService
# Output:
# fintech.UserService.GetUser
# fintech.UserService.CreateUser
# fintech.UserService.UpdateUser
# ...

# মেথড describe
grpcurl -plaintext localhost:50051 describe fintech.UserService.GetUser

# সরাসরি কল
grpcurl -plaintext -d '{"user_id": 1}' localhost:50051 fintech.UserService/GetUser
```

---

### ১০. Testing gRPC

```javascript
// ═══ Node.js — ইউনিট ও ইন্টিগ্রেশন টেস্ট ═══
const { describe, it, before, after } = require('mocha');
const { expect } = require('chai');
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

describe('UserService gRPC Tests', function () {
    let server, client;
    const TEST_PORT = 50099;

    before((done) => {
        // টেস্ট সার্ভার শুরু
        server = new grpc.Server();
        server.addService(fintechProto.UserService.service, handlers);
        server.bindAsync(
            `0.0.0.0:${TEST_PORT}`,
            grpc.ServerCredentials.createInsecure(),
            (err) => {
                if (err) return done(err);

                client = new fintechProto.UserService(
                    `localhost:${TEST_PORT}`,
                    grpc.credentials.createInsecure()
                );
                done();
            }
        );
    });

    after((done) => {
        server.tryShutdown(done);
    });

    describe('Unary: GetUser', () => {
        it('বিদ্যমান ইউজার ফেরত দেবে', (done) => {
            client.GetUser({ user_id: 1 }, (err, response) => {
                expect(err).to.be.null;
                expect(response.user).to.exist;
                expect(response.user.name).to.equal('আব্দুর রহিম');
                expect(response.user.phone).to.match(/^01\d{9}$/);
                done();
            });
        });

        it('অবিদ্যমান ইউজারে NOT_FOUND দেবে', (done) => {
            client.GetUser({ user_id: 99999 }, (err) => {
                expect(err).to.exist;
                expect(err.code).to.equal(grpc.status.NOT_FOUND);
                done();
            });
        });
    });

    describe('Unary: CreateUser', () => {
        it('সফলভাবে ইউজার তৈরি করবে', (done) => {
            const request = {
                name: 'টেস্ট ইউজার',
                email: 'test@example.com',
                phone: '01312345678',
                role: 'ROLE_CUSTOMER',
            };

            client.CreateUser(request, (err, response) => {
                expect(err).to.be.null;
                expect(response.user.name).to.equal('টেস্ট ইউজার');
                expect(response.user.id).to.be.above(0);
                done();
            });
        });

        it('অসম্পূর্ণ ডেটায় INVALID_ARGUMENT দেবে', (done) => {
            client.CreateUser({ name: '', phone: '' }, (err) => {
                expect(err.code).to.equal(grpc.status.INVALID_ARGUMENT);
                done();
            });
        });
    });

    describe('Server Streaming: WatchUserActivity', () => {
        it('একাধিক ইভেন্ট স্ট্রিম করবে', (done) => {
            const events = [];
            const call = client.WatchUserActivity({ user_id: 1 });

            call.on('data', (event) => events.push(event));
            call.on('end', () => {
                expect(events.length).to.be.greaterThan(0);
                expect(events[0].event_type).to.be.a('string');
                done();
            });
            call.on('error', done);
        });
    });

    describe('Deadline Handling', () => {
        it('অতি-ক্ষুদ্র deadline-এ DEADLINE_EXCEEDED দেবে', (done) => {
            const deadline = new Date();
            deadline.setMilliseconds(deadline.getMilliseconds() + 1); // ১ms!

            client.GetUser({ user_id: 1 }, new grpc.Metadata(), { deadline }, (err) => {
                expect(err).to.exist;
                expect(err.code).to.equal(grpc.status.DEADLINE_EXCEEDED);
                done();
            });
        });
    });
});
```

```php
<?php
// ═══ PHP — PHPUnit টেস্ট ═══
use PHPUnit\Framework\TestCase;

class UserServiceGrpcTest extends TestCase
{
    private static \Fintech\UserServiceClient $client;

    public static function setUpBeforeClass(): void
    {
        self::$client = new \Fintech\UserServiceClient('localhost:50051', [
            'credentials' => \Grpc\ChannelCredentials::createInsecure(),
        ]);
    }

    public function testGetUser_ExistingUser_ReturnsUser(): void
    {
        $request = new \Fintech\GetUserRequest();
        $request->setUserId(1);

        [$response, $status] = self::$client->GetUser($request)->wait();

        $this->assertEquals(\Grpc\STATUS_OK, $status->code);
        $this->assertNotNull($response->getUser());
        $this->assertEquals('আব্দুর রহিম', $response->getUser()->getName());
    }

    public function testGetUser_NonExistent_ReturnsNotFound(): void
    {
        $request = new \Fintech\GetUserRequest();
        $request->setUserId(99999);

        [$response, $status] = self::$client->GetUser($request)->wait();

        $this->assertEquals(\Grpc\STATUS_NOT_FOUND, $status->code);
        $this->assertNull($response);
    }

    public function testCreateUser_ValidData_CreatesSuccessfully(): void
    {
        $request = new \Fintech\CreateUserRequest();
        $request->setName('পিএইচপি টেস্ট');
        $request->setEmail('phptest@example.com');
        $request->setPhone('01412345678');
        $request->setRole(\Fintech\UserRole::ROLE_CUSTOMER);

        [$response, $status] = self::$client->CreateUser($request)->wait();

        $this->assertEquals(\Grpc\STATUS_OK, $status->code);
        $this->assertGreaterThan(0, $response->getUser()->getId());
    }

    public function testCreateUser_InvalidPhone_ReturnsError(): void
    {
        $request = new \Fintech\CreateUserRequest();
        $request->setName('ভুল ফোন');
        $request->setPhone('12345'); // অবৈধ

        [$response, $status] = self::$client->CreateUser($request)->wait();

        $this->assertEquals(\Grpc\STATUS_INVALID_ARGUMENT, $status->code);
    }

    public function testTimeout_VeryShortDeadline_Fails(): void
    {
        $request = new \Fintech\GetUserRequest();
        $request->setUserId(1);

        [$response, $status] = self::$client->GetUser(
            $request,
            [],
            ['timeout' => 1] // ১ মাইক্রোসেকেন্ড
        )->wait();

        $this->assertEquals(\Grpc\STATUS_DEADLINE_EXCEEDED, $status->code);
    }
}
```

---

## ✅ সুবিধা ও অসুবিধা

### সুবিধা

| # | সুবিধা | বিবরণ |
|---|--------|--------|
| ১ | **উচ্চ কার্যক্ষমতা** | Binary protobuf + HTTP/2 = JSON/REST-এর তুলনায় ৭-১০ গুণ দ্রুত সিরিয়ালাইজেশন |
| ২ | **কঠোর কন্ট্র্যাক্ট** | `.proto` ফাইল থেকে টাইপ-সেইফ কোড জেনারেশন; ক্লায়েন্ট-সার্ভার অসামঞ্জস্যতা কম্পাইল-টাইমে ধরা পড়ে |
| ৩ | **স্ট্রিমিং** | ৪ ধরনের RPC — রিয়েল-টাইম ডেটা ফিডের জন্য আদর্শ |
| ৪ | **বহুভাষা সমর্থন** | ১১+ ভাষায় অফিসিয়াল সাপোর্ট — polyglot microservices-এ অপরিহার্য |
| ৫ | **HTTP/2 সুবিধা** | মাল্টিপ্লেক্সিং, হেডার কম্প্রেশন, সার্ভার পুশ |
| ৬ | **বিল্ট-ইন ফিচার** | Deadline propagation, cancellation, health check, reflection, load balancing |
| ৭ | **ছোট পেলোড** | মোবাইল নেটওয়ার্কে (বাংলাদেশের 3G/4G) ব্যান্ডউইথ সাশ্রয় |

### অসুবিধা

| # | অসুবিধা | বিবরণ |
|---|---------|--------|
| ১ | **ব্রাউজার সমস্যা** | সরাসরি ব্রাউজার থেকে কল করা যায় না; gRPC-Web + proxy দরকার |
| ২ | **ডিবাগিং কঠিন** | Binary payload; curl/Postman দিয়ে সহজে টেস্ট করা যায় না (grpcurl দরকার) |
| ৩ | **শেখার বক্ররেখা** | Protobuf, HTTP/2, code generation — নতুনদের জন্য জটিল |
| ৪ | **ইকোসিস্টেম ছোট** | REST-এর তুলনায় টুলিং ও ডকুমেন্টেশন কম |
| ৫ | **ব্রেকিং চেঞ্জ** | Proto ফিল্ড নম্বর পরিবর্তন করলে backward compatibility ভাঙে |
| ৬ | **PHP সাপোর্ট** | PHP-তে gRPC সার্ভার সমর্থন সীমিত; সাধারণত Go/Node.js সার্ভার + PHP ক্লায়েন্ট |
| ৭ | **ফায়ারওয়াল** | কিছু কর্পোরেট ফায়ারওয়াল HTTP/2 ব্লক করতে পারে |

---

## ⚠️ সাধারণ ভুলসমূহ

### ১. ফিল্ড নম্বর পরিবর্তন করা
```protobuf
// ❌ ভুল — ফিল্ড নম্বর কখনো পরিবর্তন করবেন না!
message User {
    string name = 1;       // ছিলো string email = 1; → ক্লায়েন্ট ভাঙবে!
}

// ✅ সঠিক — নতুন ফিল্ড যোগ করুন, পুরানো reserved রাখুন
message User {
    reserved 3;            // পুরানো ফিল্ড ৩ রিজার্ভ
    reserved "old_field";
    string name = 1;
    string email = 2;
    string phone = 4;      // নতুন ফিল্ড নম্বর
}
```

### ২. Deadline সেট না করা
```javascript
// ❌ ভুল — deadline ছাড়া কল; ইনফিনিট ব্লকিং হতে পারে
client.GetUser({ user_id: 1 }, (err, res) => {});

// ✅ সঠিক — সবসময় deadline সেট করুন
const deadline = new Date(Date.now() + 5000);
client.GetUser({ user_id: 1 }, metadata, { deadline }, (err, res) => {});
```

### ৩. Error Details না পাঠানো
```javascript
// ❌ ভুল — শুধু generic error
callback({ code: grpc.status.INTERNAL, message: 'error' });

// ✅ সঠিক — descriptive error + metadata
const meta = new grpc.Metadata();
meta.add('error-details-bin', Buffer.from(JSON.stringify({
    violations: [{ field: 'phone', rule: 'bd_phone_format' }],
})));
callback({ code: grpc.status.INVALID_ARGUMENT, message: 'ইনপুট ত্রুটি', metadata: meta });
```

### ৪. স্ট্রিম ক্লোজ না করা
```javascript
// ❌ ভুল — call.end() ভুলে গেলে রিসোর্স লিক
const call = client.BulkCreateUsers(metadata, callback);
for (const u of users) call.write(u);
// call.end() নেই! সার্ভার চিরকাল অপেক্ষা করবে

// ✅ সঠিক
for (const u of users) call.write(u);
call.end(); // অবশ্যই!
```

### ৫. বড় পেলোড পাঠানো (মেসেজ সাইজ লিমিট)
```javascript
// ❌ ভুল — ডিফল্ট ৪MB লিমিট; বড় ফাইল পাঠালে ক্র্যাশ
// RESOURCE_EXHAUSTED: Received message larger than max

// ✅ সঠিক — Channel option-এ সাইজ বাড়ান অথবা streaming ব্যবহার করুন
const client = new Service('host:port', creds, {
    'grpc.max_receive_message_length': 50 * 1024 * 1024, // 50MB
});
// অথবা বড় ডেটার জন্য Client Streaming ব্যবহার করুন
```

### ৬. L4 Load Balancer ব্যবহার করা
```
❌ ভুল: AWS NLB / HAProxy (TCP mode) → একই সার্ভারে সব রিকোয়েস্ট

✅ সঠিক:
  - Envoy, Nginx (HTTP/2), AWS ALB (HTTP/2) → L7 load balancing
  - অথবা client-side load balancing (round_robin / xds)
```

### ৭. Protobuf-এ default value ভুলে যাওয়া
```javascript
// ⚠️ Proto3-তে সব ফিল্ডের default value আছে
// int32 → 0, string → "", bool → false
// তাই "ফিল্ড সেট করা হয়নি" আর "ফিল্ড 0/empty" আলাদা করা কঠিন

// ✅ সঠিক — wrapper types বা optional ব্যবহার করুন (proto3 syntax)
// optional int32 age = 1;  → has_age() দিয়ে চেক করা যাবে
```

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                     gRPC সারসংক্ষেপ                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📌 কী: HTTP/2 + Protobuf ভিত্তিক উচ্চ-কার্যক্ষম RPC           │
│                                                                 │
│  🎯 কখন ব্যবহার করবেন:                                         │
│     ✓ মাইক্রোসার্ভিস inter-service communication                │
│     ✓ উচ্চ throughput, কম latency দরকার                         │
│     ✓ রিয়েল-টাইম স্ট্রিমিং (চ্যাট, ফিড, নোটিফিকেশন)         │
│     ✓ মোবাইল ক্লায়েন্ট (ব্যান্ডউইথ সাশ্রয়)                    │
│     ✓ Polyglot মাইক্রোসার্ভিস (PHP+Go+Node.js+Java)           │
│                                                                 │
│  ⛔ কখন ব্যবহার করবেন না:                                      │
│     ✗ পাবলিক-ফেসিং REST API (ব্রাউজার সমস্যা)                 │
│     ✗ সিম্পল CRUD (REST যথেষ্ট)                                │
│     ✗ ছোট টিম যারা protobuf শিখতে চায় না                     │
│                                                                 │
│  🏗️ বাংলাদেশ প্রেক্ষাপট:                                      │
│     ► ফিনটেক: Wallet↔Transaction↔KYC সার্ভিসের মধ্যে gRPC      │
│     ► ই-কমার্স: Inventory↔Order↔Payment internal gRPC            │
│     ► রাইডশেয়ার: Location streaming, real-time matching         │
│     ► External API: REST/GraphQL (ব্রাউজার/3rd party)           │
│                                                                 │
│  🔑 মনে রাখবেন:                                                │
│     1. সবসময় deadline সেট করুন                                 │
│     2. ফিল্ড নম্বর কখনো পরিবর্তন করবেন না                     │
│     3. L7 load balancing ব্যবহার করুন                           │
│     4. Health check ইমপ্লিমেন্ট করুন                            │
│     5. Interceptor দিয়ে cross-cutting concerns হ্যান্ডেল করুন   │
│     6. স্ট্রিম অবশ্যই end() করুন                               │
│     7. Error-এ proper status code ও details দিন               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

> **📖 আরও শিখুন:**
> - [gRPC Official Docs](https://grpc.io/docs/)
> - [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
> - [grpcurl — CLI Tool](https://github.com/fullstorydev/grpcurl)
> - [Awesome gRPC](https://github.com/grpc-ecosystem/awesome-grpc)
