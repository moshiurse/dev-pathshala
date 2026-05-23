# 📡 RPC Patterns — Remote Procedure Call-এর ধারণা, ফ্রেমওয়ার্ক, ডিজাইন প্যাটার্ন ও বাস্তব ব্যবহার

## 📖 Definition (সংজ্ঞা, ইতিহাস, তুলনা, ফ্রেমওয়ার্ক, প্যাটার্ন)

### ১. RPC কী?

**RPC (Remote Procedure Call)** হলো এমন একটি programming model যেখানে একটি application অন্য একটি process, server, বা service-এর function/method-কে এমনভাবে call করে যেন সেটা local function।

অর্থাৎ developer-এর mental model হয়:

- `getUser(123)`
- `chargeWallet("017xxxxxxxx", 500)`
- `reserveInventory("SKU-123", 2)`

কিন্তু বাস্তবে এই call local memory-এর মধ্যে execute হচ্ছে না।

বাস্তবে যা হয়:

- function call serialize হয়
- network-এর মাধ্যমে remote machine-এ যায়
- remote side method execute করে
- result serialize হয়ে ফিরে আসে

এই abstraction-টাই RPC-এর মূল জাদু।

### Local call বনাম Remote call

লোকাল call-এ:

- function call সস্তা
- latency microsecond/nanosecond level
- network failure নেই
- serialization লাগে না

Remote call-এ:

- network latency থাকে
- connection fail করতে পারে
- timeout হতে পারে
- payload serialize/deserialize করতে হয়
- authentication/authorization দরকার হয়

তাই RPC local function-এর মতো **দেখালেও**, semantics local function-এর মতো **না**।

### কেন RPC দরকার হলো?

Distributed system বড় হওয়ার সাথে সাথে এক service-এর data বা logic অন্য service-এর দরকার হয়।

উদাহরণ:

- Daraz order-service জানতে চায় inventory আছে কিনা
- bKash payment-service fraud-service থেকে risk score নেয়
- Pathao trip-service pricing-service থেকে fare estimate চায়
- Chaldal checkout-service promotion-service থেকে discount rule fetch করে

এই সব ক্ষেত্রে developer যদি raw socket protocol manually বানাতো, তাহলে কাজ কঠিন হয়ে যেত। RPC framework এই complexity কমিয়ে দেয়।

---

### ২. RPC-এর ইতিহাস

RPC নতুন কোনো ধারণা নয়। Distributed computing-এর শুরু থেকেই এর evolution হয়েছে।

| সময়কাল | প্রযুক্তি | কী গুরুত্বপূর্ণ ছিল |
|--------|-----------|----------------------|
| 1970s | Early distributed systems | local procedure call-এর model network-এ নেওয়ার ধারণা |
| 1980s | **Sun RPC / ONC RPC** | Unix/NFS ecosystem-এ widely used classic RPC |
| 1990s | **DCE RPC** | enterprise distributed systems-এ stronger tooling |
| 1990s | **CORBA** | object-oriented distributed invocation, IDL, ORB |
| 1990s | **DCOM** | Microsoft ecosystem-এ distributed component model |
| late 1990s | **XML-RPC** | সহজ XML-based RPC over HTTP |
| 2000s | **SOAP/WSDL** | enterprise contract-heavy web service era |
| 2007+ | **Apache Thrift** | Facebook-এর multi-language RPC + serialization |
| 2009+ | **Apache Avro RPC** | schema evolution friendly RPC in data ecosystem |
| 2015+ | **gRPC** | HTTP/2 + protobuf + streaming |
| 2020s | **tRPC / Connect RPC** | modern developer-friendly RPC, browser/full-stack focus |

### Sun RPC / ONC RPC

Sun Microsystems NFS-এর মতো system-এ remote procedure call সহজ করার জন্য ONC RPC তৈরি করে।

এখানে মূল focus ছিল:

- simple procedure interface
- XDR serialization
- Unix ecosystem compatibility

### DCE RPC

Distributed Computing Environment (DCE) enterprise environment-এ RPC-কে formalize করে।

এতে ছিল:

- stronger security model
- naming service integration
- enterprise-grade distributed computing support

### CORBA

CORBA (Common Object Request Broker Architecture) object-oriented distributed invocation model দেয়।

কেন্দ্রীয় ধারণা:

- IDL দিয়ে interface define
- ORB (Object Request Broker) call route করে
- বহু language support

কিন্তু complexity খুব বেশি ছিল। অনেক architecture diagram-এ ভালো লাগলেও বাস্তবে development এবং debugging ব্যয়বহুল হতো।

### DCOM

Microsoft ecosystem-এ COM object-কে network-এর উপর extend করার পথ ছিল DCOM।

এটি Windows-heavy enterprise setup-এ জনপ্রিয় ছিল, কিন্তু cross-platform simplicity দিতে পারেনি।

### XML-RPC ও SOAP era

Web stack mature হওয়ার পর XML-based RPC popular হয়। XML-RPC সহজ ছিল, SOAP ছিল contract-heavy, verbose, enterprise-oriented।

### Modern era

আজকের modern RPC landscape broadly তিন ভাগে ভাগ করা যায়:

- lightweight text-based: JSON-RPC
- binary schema-based: Thrift, Avro RPC, gRPC
- developer-experience focused: tRPC, Connect RPC

---

### ৩. RPC-এর Core Execution Model

একটি classic RPC call-এর flow সাধারণত এমন:

1. Client application method call করে
2. Client stub সেই method call-কে network message-এ রূপান্তর করে
3. Transport layer request server-এ পাঠায়
4. Server skeleton request parse করে
5. Actual service method execute হয়
6. Result serialize হয়
7. Response client-এ ফিরে আসে
8. Client stub result-কে local return value-এর মতো expose করে

### গুরুত্বপূর্ণ component

| Component | কাজ |
|----------|-----|
| Client | application code যা method call করে |
| Client Stub | local-looking API, কিন্তু behind the scenes network call করে |
| Serializer | params/result-কে binary বা text payload-এ encode করে |
| Transport | HTTP, TCP, WebSocket, Unix socket ইত্যাদি |
| Server Skeleton | request unpack করে appropriate handler call করে |
| Service Implementation | business logic execute করে |
| Response Mapper | success/error কে client-friendly object-এ রূপান্তর করে |

### Stub ও Skeleton ধারণা

- **Stub**: client-side proxy
- **Skeleton**: server-side dispatcher

এরা developer-কে raw bytes, sockets, framing, correlation id, error mapping এগুলোর ঝামেলা থেকে বাঁচায়।

---

### ৪. IDL (Interface Definition Language) কী?

**IDL** হলো interface contract লেখার ভাষা।

এর সাহায্যে আপনি declare করেন:

- কোন service আছে
- কোন method আছে
- request message-এর field কী
- response message-এর field কী
- optional/required semantics
- কখনও versioning বা namespace

উদাহরণস্বরূপ:

```idl
service WalletService {
  GetBalance(GetBalanceRequest) returns (GetBalanceResponse)
  TransferMoney(TransferRequest) returns (TransferResponse)
}
```

IDL কেন দরকার?

- client ও server একই contract follow করে
- বহু language-এ code generate করা যায়
- compatibility manage করা সহজ হয়
- documentation source-of-truth হয়

### সব RPC framework-এ কি IDL লাগে?

না।

| Framework | IDL লাগে? | মন্তব্য |
|----------|-----------|---------|
| JSON-RPC | সাধারণত না | method name + params convention-based |
| XML-RPC | সাধারণত না | XML structure fixed, contract docs আলাদা হতে পারে |
| Thrift | হ্যাঁ | `.thrift` file থেকে code generation |
| Avro RPC | হ্যাঁ | Avro schema/protocol লাগে |
| gRPC | হ্যাঁ | `.proto` লাগে |
| tRPC | না | TypeScript type inference-ভিত্তিক |
| Connect RPC | সাধারণত protobuf contract | তবে transport/browser DX উন্নত |

### IDL-এর trade-off

সুবিধা:

- explicit contract
- code generation
- polyglot support
- safer refactoring

অসুবিধা:

- extra compilation step
- schema evolution discipline লাগে
- build pipeline জটিল হতে পারে

---

### ৫. RPC বনাম REST বনাম GraphQL বনাম Message Queue

এগুলো একে অপরের full replacement নয়। এগুলো ভিন্ন problem space address করে।

| দিক | RPC | REST | GraphQL | Message Queue |
|-----|-----|------|----------|---------------|
| mental model | function/method call | resource manipulation | graph query/mutation | async message exchange |
| API semantics | verb-oriented | resource-oriented | client-shaped data | event/command-oriented |
| coupling | তুলনামূলক বেশি | মাঝারি | schema coupling | producer/consumer loose coupling |
| response pattern | usually immediate | immediate | immediate | often delayed/async |
| transport | HTTP/TCP/WebSocket/Unix socket | mostly HTTP | HTTP/WebSocket | broker-specific |
| contract style | IDL বা method contract | OpenAPI/implicit resources | schema-first | message schema |
| browser friendliness | varies | খুব ভালো | খুব ভালো | কম |
| streaming | framework dependent | limited | subscriptions | natural async stream |
| latency overhead | কম হতে পারে | medium | parsing cost বেশি হতে পারে | immediate নয় |
| caching | framework-specific | HTTP-native | complex | queue semantics |
| public API suitability | medium | high | high for flexible clients | low |
| internal microservice suitability | খুব ভালো | ভালো | selective | async workloads-এর জন্য খুব ভালো |

### RPC কখন REST-এর চেয়ে ভালো?

RPC ভালো যখন:

- operation resource-এর চেয়ে action-centric
- low latency দরকার
- internal microservice communication হচ্ছে
- strongly-typed contract দরকার
- one service অন্য service-এর method-style API consume করছে
- streaming দরকার
- client ও server controlled environment-এ আছে

উদাহরণ:

- bKash ledger-service → fraud-service: `CheckRisk(transaction)`
- Pathao dispatch-service → pricing-service: `EstimateFare(route)`
- Chaldal checkout-service → inventory-service: `ReserveItems(items)`

### REST কখন RPC-এর চেয়ে ভালো?

REST ভালো যখন:

- public API expose করতে হবে
- standard HTTP semantics কাজে লাগাতে হবে
- caching/CDN দরকার
- resources naturally model করা যায়
- third-party integrator অনেক
- browser/devtools friendliness দরকার

উদাহরণ:

- Daraz public seller API
- Prothom Alo public content API
- GP self-care mobile app public backend endpoints

### GraphQL কখন better fit?

GraphQL ভালো যখন:

- client ভিন্ন ভিন্ন shape-এর data চায়
- frontend team over-fetching/under-fetching কমাতে চায়
- multiple resources এক query-তে দরকার
- mobile client bandwidth sensitive

তবে GraphQL সব operation-এর জন্য perfect না। Heavy command-style operations-এ RPC বা REST বেশি পরিষ্কার হতে পারে।

### Message Queue কখন better fit?

Message queue ভালো যখন:

- immediate response দরকার নেই
- loose coupling দরকার
- retry/durable delivery দরকার
- background processing হবে
- event-driven architecture বানাতে হবে

উদাহরণ:

- `SendWelcomeSms`
- `GenerateInvoicePdf`
- `SyncAnalyticsEvent`
- `PublishNewsToSearchIndex`

### Semantic পার্থক্য: Verb বনাম Resource

RPC style:

- `CreateTrip`
- `CancelOrder`
- `VerifyNid`
- `CheckEligibility`

REST style:

- `POST /trips`
- `POST /orders/{id}/cancel`
- `POST /users/{id}/verifications/nid`
- `GET /loan-applications/{id}/eligibility`

RPC সাধারণত business action-centric।

REST সাধারণত resource-centric।

### Performance comparison (general tendency)

| বিষয় | RPC | REST | GraphQL |
|------|-----|------|----------|
| payload size | ছোট হতে পারে | medium | variable |
| parsing cost | binary হলে কম | JSON parse cost medium | query parse + resolve cost বেশি |
| multiplexing | framework-specific | HTTP/1.1-এ limited, HTTP/2-এ ভালো | HTTP/2 possible |
| caching | built-in নাও থাকতে পারে | HTTP-native strong | custom caching লাগে |
| N+1 risk | low at transport level | low | resolver design খারাপ হলে high |

**Note:** performance শুধু protocol-এর উপর নির্ভর করে না।

আরও গুরুত্বপূর্ণ বিষয়:

- payload shape
- network RTT
- server implementation
- retry behavior
- compression
- connection reuse
- observability overhead

---

### ৬. RPC Frameworks / Protocols

#### ৬.১ JSON-RPC

**JSON-RPC** হলো lightweight, text-based, simple RPC protocol যেখানে request/response JSON object আকারে যায়।

Transport হতে পারে:

- HTTP
- WebSocket
- raw TCP

সবচেয়ে common fields:

- `jsonrpc`
- `id`
- `method`
- `params`
- `result`
- `error`

##### Typical request

```json
{
  "jsonrpc": "2.0",
  "id": 101,
  "method": "wallet.getBalance",
  "params": {
    "account": "01712345678"
  }
}
```

##### Success response

```json
{
  "jsonrpc": "2.0",
  "id": 101,
  "result": {
    "balance": 4200.75,
    "currency": "BDT"
  }
}
```

##### Error response

```json
{
  "jsonrpc": "2.0",
  "id": 101,
  "error": {
    "code": -32001,
    "message": "Account not found",
    "data": {
      "account": "01712345678"
    }
  }
}
```

##### Batch request

একই HTTP request-এ multiple RPC call পাঠানো যায়:

```json
[
  {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "inventory.check",
    "params": {"sku": "RICE-25KG"}
  },
  {
    "jsonrpc": "2.0",
    "id": 2,
    "method": "pricing.get",
    "params": {"sku": "RICE-25KG"}
  }
]
```

##### Notification

যদি `id` না থাকে, সেটাকে notification ধরা হয়।

মানে:

- client response expect করছে না
- server best-effort process করবে

```json
{
  "jsonrpc": "2.0",
  "method": "analytics.trackEvent",
  "params": {
    "event": "checkout_started",
    "userId": 55
  }
}
```

##### JSON-RPC কোথায় ভালো?

- simple internal tools
- admin panel backend
- blockchain APIs (Ethereum ecosystem)
- lightweight microservices
- browser-friendly custom RPC

##### JSON-RPC-এর limitation

- contract enforcement weak
- type safety কম
- error code standardization loosely followed
- large systems-এ governance challenge হতে পারে

---

#### ৬.২ XML-RPC

**XML-RPC** হলো XML-based remote procedure call protocol।

এটি JSON-RPC-এর আগের যুগের সহজ HTTP-based RPC style।

বৈশিষ্ট্য:

- XML payload
- HTTP transport
- simple method call structure
- SOAP-এর চেয়ে হালকা

উদাহরণ payload:

```xml
<methodCall>
  <methodName>wallet.getBalance</methodName>
  <params>
    <param>
      <value><string>01712345678</string></value>
    </param>
  </params>
</methodCall>
```

আজকের modern system-এ XML-RPC কম ব্যবহৃত, কারণ:

- XML verbose
- parsing heavy
- tooling modern JSON ecosystem-এর তুলনায় দুর্বল

তবে legacy integration-এ এটা এখনও দেখা যেতে পারে।

---

#### ৬.৩ Apache Thrift

**Apache Thrift** মূলত Facebook তৈরি করেছিল cross-language service communication-এর জন্য।

এর দুটি বড় শক্তি:

- IDL-based schema
- code generation

Thrift support করে:

- বহু language
- বহু transport
- বহু protocol

##### Thrift IDL example

```thrift
service PricingService {
  double estimateFare(1:string zone, 2:double distanceKm, 3:string vehicleType)
}
```

##### Transport options

- TSocket
- TFramedTransport
- THttpClient

##### Protocol options

- TBinaryProtocol
- TCompactProtocol
- TJSONProtocol

##### Thrift কখন useful?

- polyglot organization
- mature IDL/codegen workflow already আছে
- HTTP/2 dependency avoid করতে চান
- simple request/response with efficient binary protocol দরকার

##### Thrift বনাম gRPC

| বিষয় | Thrift | gRPC |
|------|--------|------|
| transport flexibility | বেশি | mostly HTTP/2 |
| schema | Thrift IDL | protobuf |
| browser integration | কম | gRPC-Web/Connect via adapters |
| ecosystem momentum | কমেছে | অনেক বেশি |
| streaming model | limited compared to gRPC | strong |
| existing legacy deployment | অনেক জায়গায় আছে | modern default in many orgs |

**When to use over gRPC:**

- legacy Thrift ecosystem already আছে
- HTTP/2 operational complexity avoid করতে চান
- multiple transport/protocol flexibility জরুরি

---

#### ৬.৪ Apache Avro RPC

**Apache Avro RPC** Avro schema ও protocol-এর উপর ভিত্তি করে তৈরি RPC approach।

এটি বিশেষ করে data ecosystem-এ গুরুত্বপূর্ণ।

Strong points:

- schema-based serialization
- compact binary encoding
- schema evolution friendly
- Hadoop ecosystem alignment

##### Avro RPC কোথায় ভালো?

- data pipeline platform
- batch + service hybrid environment
- analytics-heavy backend
- schema evolution একটি বড় concern হলে

##### Schema evolution কেন গুরুত্বপূর্ণ?

ধরুন Prothom Alo content-distribution system-এ `Article` message ছিল:

- `id`
- `title`
- `publishedAt`

পরে আপনি `authorName` যোগ করলেন।

যদি consumer পুরনো version-এ থাকে, Avro schema resolution rules backward/forward compatibility manage করতে সাহায্য করে।

এটা long-lived data contract-এ খুব মূল্যবান।

---

#### ৬.৫ gRPC (সংক্ষেপে)

**gRPC** modern binary RPC framework যেখানে:

- HTTP/2 transport
- protobuf serialization
- unary + streaming support
- strong code generation
- deadline, metadata, status model বেশ mature

এই repository-তে gRPC-এর উপর আলাদা deep dive আছে:

- **দেখুন: `06-api-design/grpc.md`**

এই file-এ gRPC-এর deep protocol details repeat করা হলো না, কারণ সেটা আলাদা file-এ বিস্তৃতভাবে আলোচনা করা হয়েছে।

##### gRPC-এর key differentiators

- HTTP/2 multiplexing
- protobuf efficiency
- built-in streaming
- mature cross-language ecosystem
- service mesh ecosystem-এর সাথে strong fit

---

#### ৬.৬ tRPC

**tRPC** হলো TypeScript-first RPC framework যেখানে end-to-end type safety primary goal।

বিশেষত্ব:

- TypeScript types directly share করা যায়
- আলাদা IDL/code generation লাগে না
- full-stack TS app-এ developer experience অসাধারণ
- React Query-এর সাথে clean integration

##### tRPC কোথায় shine করে?

- Next.js full-stack app
- internal dashboard
- admin portal
- startup product যেখানে backend + frontend দুটোই TS

উদাহরণ:

Daraz-এর internal seller dashboard যদি পুরোপুরি TypeScript stack-এ হয়, তাহলে tRPC দিয়ে:

- frontend autocompletion পাবে
- backend input/output types reuse হবে
- contract drift কমবে

##### Limitation

- non-TypeScript consumer-এর জন্য ideal না
- public API standard হিসেবে REST/gRPC-এর মতো universal না
- polyglot microservices-এ weak fit

---

#### ৬.৭ Connect RPC

**Connect RPC** modern RPC approach যা gRPC ecosystem-এর উপর build করলেও browser-native developer experience-এ গুরুত্ব দেয়।

এটি useful যখন:

- browser থেকে clean RPC call দরকার
- gRPC-Web-এর complexity কমাতে চান
- protobuf contract রাখতে চান
- multiple protocol fallback দরকার

Connect RPC-এর advantage:

- browser-friendly
- HTTP semantics-এর সাথে ভালো compatibility
- gRPC-এর কিছু শক্তি retain করে
- simpler tooling in some frontend scenarios

---

### ৭. RPC Design Patterns

#### ৭.১ Request-Response Pattern

সবচেয়ে common RPC pattern।

Flow:

- client request পাঠায়
- server process করে
- single response দেয়

Use cases:

- `GetUser`
- `GetOrderStatus`
- `CheckInventory`
- `EstimateFare`

এটা synchronous mental model দেয়।

#### ৭.২ Streaming Pattern

সব RPC এক request → এক response না। অনেক সময় stream দরকার হয়।

প্রধান ধরন:

- server streaming
- client streaming
- bidirectional streaming

##### Server streaming

একটি request-এর বিপরীতে server অনেকগুলো message ফেরত দেয়।

উদাহরণ:

- live GPS updates
- order status changes
- stock ticker

##### Client streaming

client ধাপে ধাপে data upload করে, শেষে server summary response দেয়।

উদাহরণ:

- bulk transaction upload
- log ingestion
- ride telemetry upload

##### Bidirectional streaming

দুইদিকেই stream খোলা থাকে।

উদাহরণ:

- live chat
- real-time dispatch matching
- collaborative session

#### ৭.৩ Fire-and-Forget / One-way RPC

এখানে caller response wait করে না।

Use cases:

- analytics event
- audit trail write request
- background notification trigger

সতর্কতা:

- delivery guarantee unclear হতে পারে
- caller success/failure জানে না
- idempotency ও durability আলাদা ভাবে design করতে হয়

#### ৭.৪ Callback Pattern

এক service request নেয়, পরে অন্য endpoint/method-এ callback দেয়।

উদাহরণ:

- payment processing started
- later callback with success/failure

বাংলাদেশ context:

একটি merchant platform bKash/Nagad/SSLCOMMERZ style payment processing-এ callback URL receive করতে পারে।

Trade-off:

- client publicly reachable endpoint লাগতে পারে
- correlation id manage করতে হয়
- retry ও signature verification জরুরি

#### ৭.৫ Circuit Breaker with RPC

Remote service slow বা unhealthy হলে caller বারবার একই failing call না করে circuit trip করতে পারে।

States:

- Closed
- Open
- Half-Open

উপকারিতা:

- cascading failure কমায়
- thread/connection resource বাঁচায়
- দ্রুত fallback enable করে

উদাহরণ:

Foodpanda-like delivery platform-এ `promo-service` down হলে checkout পুরো system hang না করে fallback discount=0 return করতে পারে।

#### ৭.৬ Retry with Idempotency Key

Network failure হলে retry useful। কিন্তু retry blindly করলে duplicate side effect হতে পারে।

সমাধান:

- idempotency key
- request fingerprint
- dedupe store

বিশেষ করে money movement RPC-এ এটা critical।

উদাহরণ:

`ChargeWallet(account=..., amount=500, idempotencyKey=abc-123)`

প্রথম call timeout হলেও second retry same effect produce করবে, double charge করবে না।

#### ৭.৭ Timeout এবং Deadline Propagation

Caller যদি 500ms budget নিয়ে call শুরু করে, তাহলে downstream service-এ full 500ms দিয়ে নতুন chain বানানো উচিত না।

বরং:

- incoming deadline read করতে হবে
- remaining time downstream-এ propagate করতে হবে

না হলে timeout cascading হয়।

উদাহরণ:

- API Gateway timeout = 1000ms
- order-service inventory-service-কে 900ms দেয়
- inventory-service warehouse-service-কে আবার 900ms দিলে total budget ভেঙে যায়

সঠিক পদ্ধতি:

- remaining budget propagate করো

#### ৭.৮ Load Balancing Strategies

##### Client-side load balancing

client নিজেই service registry থেকে instances discover করে।

সুবিধা:

- proxy hop কম
- smarter retry possible

অসুবিধা:

- client library complex হয়
- service discovery logic প্রতিটি client-এ ছড়ায়

##### Proxy-side / Server-side load balancing

client একটি load balancer বা proxy-কে call করে, proxy backend instance বেছে নেয়।

সুবিধা:

- simpler client
- centralized policy
- observability সহজ

অসুবিধা:

- extra hop
- proxy bottleneck হতে পারে

##### Common balancing algorithms

- round robin
- least connections
- EWMA latency aware
- consistent hashing
- zone aware routing

#### ৭.৯ Service Mesh Integration

Modern RPC অনেক সময় service mesh-এর সাথে কাজ করে।

Mesh কী দেয়:

- mTLS
- retries
- timeout policy
- circuit breaking
- observability
- traffic shifting

এর ফলে application code thinner হতে পারে, কিন্তু transport behavior mesh policy-র উপর নির্ভরশীল হয়ে যায়।

---

### ৮. RPC Communication Patterns

#### ৮.১ Synchronous RPC

Caller wait করে যতক্ষণ না response আসে।

ভালো যখন:

- immediate answer দরকার
- workflow sequential
- UX response blocking

খারাপ যখন:

- downstream slow
- fan-out বেশি
- user-facing latency tight

#### ৮.২ Asynchronous RPC

Caller request পাঠিয়ে পরে result poll/callback/stream-এর মাধ্যমে পায়।

ভালো যখন:

- long-running job
- report generation
- ML inference job
- reconciliation process

Pattern হতে পারে:

- submit job
- return job id
- later `GetJobStatus(jobId)`

#### ৮.৩ Multiplexed RPC

একই connection-এর উপর multiple concurrent streams/calls চলতে পারে।

এটা useful:

- mobile network efficiency-তে
- high-QPS microservice communication-এ
- connection setup overhead কমাতে

HTTP/2-based systems (যেমন gRPC, Connect-এর কিছু mode) এখানে advantage পায়।

#### ৮.৪ Different Transport-এর উপর RPC

##### TCP

- low-level control বেশি
- binary protocol efficient হতে পারে
- custom framing লাগে

##### HTTP

- firewall friendly
- proxy/CDN/tooling compatibility ভালো
- debugging সহজ

##### WebSocket

- long-lived bidirectional communication
- browser support ভালো
- event + RPC hybrid useful

##### Unix Socket

- একই machine-এর process-এর মধ্যে দ্রুত communication
- local-only internal daemon communication-এ ভালো

উদাহরণ:

এক server-এ Nginx sidecar এবং PHP worker Unix socket দিয়ে কথা বলতে পারে।

---

### ৯. Serialization Formats

RPC-এর performance, compatibility, debuggability অনেকটাই serialization format-এর উপর নির্ভর করে।

#### ৯.১ Protocol Buffers

- binary
- schema-based
- compact
- code generation friendly
- backward compatibility rules আছে

ভালো যখন:

- cross-language RPC
- performance-sensitive microservice
- strict contract দরকার

#### ৯.২ MessagePack

- binary
- JSON-like structure
- schema-less হতে পারে
- compact

ভালো যখন:

- JSON-এর চেয়ে ছোট payload চান
- schema strict না হলেও চলবে
- dynamic language-heavy system

#### ৯.৩ FlatBuffers

- zero-copy read philosophy
- high-performance scenarios
- memory-sensitive system

ভালো যখন:

- game/edge/high-performance local IPC
- parse overhead minimize করতে চান

#### ৯.৪ JSON

- text-based
- human-readable
- ubiquitous tooling
- debugging easy

ভালো যখন:

- simplicity priority
- public-facing integration
- developer onboarding দ্রুত দরকার

#### ৯.৫ CBOR

- binary JSON-like
- compact
- machine-friendly
- IoT/embedded contexts-এ useful

#### Comparison Table

| Format | Type | Human-readable | Schema | Size | Speed | Best Fit |
|--------|------|----------------|--------|------|-------|----------|
| Protocol Buffers | binary | না | strong | ছোট | দ্রুত | microservices, gRPC |
| MessagePack | binary | না | optional | ছোট | দ্রুত | lightweight RPC |
| FlatBuffers | binary | না | strong | ছোট | খুব দ্রুত read | high-performance systems |
| JSON | text | হ্যাঁ | weak/optional | বড় | মাঝারি | simple APIs, JSON-RPC |
| CBOR | binary | না | optional | ছোট | দ্রুত | constrained environments |

### Serialization format বাছাইয়ের practical rule

- মানুষে-debug করবে? → JSON
- cross-language strict contract? → Protobuf/Thrift/Avro
- TS-first internal app? → JSON/tRPC enough হতে পারে
- ultra-low latency local system? → FlatBuffers consider করা যায়

---

### ১০. Service Discovery for RPC

RPC scale করলে service location hardcoded রাখা যায় না।

কারণ:

- instances dynamic
- autoscaling হয়
- container/pod restart হয়
- IP change হয়

#### ১০.১ Client-side discovery

Flow:

- client registry query করে
- healthy instance list পায়
- নিজেই one instance choose করে

Useful in:

- smart clients
- internal platform libraries
- service mesh ছাড়া environments

#### ১০.২ Server-side discovery

Flow:

- client শুধু load balancer address জানে
- load balancer registry থেকে backend resolve করে
- request forward করে

Useful in:

- simpler clients
- central traffic policy
- browser/mobile/public edge systems

#### ১০.৩ Service Registry

Common registry tools:

- Consul
- etcd
- ZooKeeper
- Eureka (ecosystem-specific)
- Kubernetes service registry / DNS

Registry সাধারণত store করে:

- service name
- instance address
- port
- metadata
- health state
- zone/region

#### Registration pattern

একটি new instance উঠলে:

1. registry-তে register করে
2. health check pass করে
3. traffic receive শুরু করে
4. shutdown-এর আগে deregister করে

---

### ১১. Error Handling in RPC

Remote call-এ error handling অত্যন্ত গুরুত্বপূর্ণ, কারণ failure modes local function call-এর তুলনায় অনেক বেশি।

#### Common error categories

- network unreachable
- timeout
- authentication failed
- authorization denied
- validation failed
- rate limited
- dependency unavailable
- internal server error
- version mismatch
- serialization error

#### Status code design

সব framework একই code set use করে না, কিন্তু conceptual mapping useful:

| Category | Meaning | Retry? |
|----------|---------|--------|
| INVALID_ARGUMENT | request malformed | না |
| UNAUTHENTICATED | auth missing/invalid | না, unless token refresh |
| PERMISSION_DENIED | caller allowed না | না |
| NOT_FOUND | target absent | depends |
| ALREADY_EXISTS | duplicate create | না |
| FAILED_PRECONDITION | business rule unmet | না |
| RESOURCE_EXHAUSTED | quota/rate limit | পরে |
| UNAVAILABLE | service temporarily down | হ্যাঁ |
| DEADLINE_EXCEEDED | timeout | হ্যাঁ, carefully |
| INTERNAL | server bug | maybe |

#### Rich error details

শুধু `message="failed"` দিলে চলবে না।

ভালো error response-এ থাকে:

- machine-readable code
- human-readable message
- field-level details
- retry hint
- correlation/request id
- optional docs link

উদাহরণ:

```json
{
  "code": "INVALID_ARGUMENT",
  "message": "amount must be greater than 0",
  "details": {
    "field": "amount",
    "min": 1
  },
  "requestId": "req-8844"
}
```

#### Retry semantics

সব error retry করা যায় না।

Retry-worthy:

- UNAVAILABLE
- transient network reset
- timeout on idempotent read
- 429/RESOURCE_EXHAUSTED with backoff

Retry-not-worthy:

- invalid input
- auth failure
- permission denied
- business rule violation

#### Timeout cascading

এক service slow হলে upstream-গুলো wait করতে করতে exhausted হয়ে যায়।

Symptoms:

- thread pool saturation
- connection pool exhaustion
- end-user request timeout
- retry storm

Countermeasure:

- strict deadlines
- bounded retries
- circuit breaker
- bulkhead isolation
- graceful fallback

---

### ১২. Security in RPC

RPC internal হলেই secure ধরে নেওয়া ভুল। East-west traffic-ও attacker target করতে পারে।

#### ১২.১ mTLS

**mTLS (mutual TLS)**-এ client এবং server দুজনেই certificate দিয়ে একে অপরকে authenticate করে।

উপকারিতা:

- service identity strong হয়
- internal traffic encrypt হয়
- zero-trust model enable হয়

বাংলাদেশ context:

bKash-এর মতো financial platform-এ internal payment authorization call plain HTTP-তে পাঠানো ভয়ংকর ভুল হবে। mTLS mandatory হওয়া উচিত।

#### ১২.২ Token-based auth

সব RPC call-এ certificate-based identity যথেষ্ট নাও হতে পারে। Caller user context বা scope জানতে হতে পারে।

তখন:

- JWT
- opaque token + introspection
- HMAC signature
- API key (less ideal for internal service-to-service)

ব্যবহার হতে পারে।

#### ১২.৩ Per-method authorization

সব caller সব method call করতে পারবে না।

উদাহরণ:

| Method | কে call করতে পারবে |
|--------|--------------------|
| `wallet.GetBalance` | customer-service, app-gateway |
| `wallet.AdjustBalance` | ledger-service only |
| `fraud.OverrideScore` | admin-backoffice only |
| `content.PublishBreakingNews` | editor-service only |

#### ১২.৪ Input validation

RPC internal হলেই validation বাদ দেওয়া যাবে না।

কারণ:

- buggy caller থাকতে পারে
- compromised service থাকতে পারে
- schema mismatch হতে পারে
- integer overflow বা string bomb হতে পারে

#### ১২.৫ Auditability

Sensitive RPC-এ log করা উচিত:

- caller identity
- method name
- subject id
- request id
- decision (allow/deny)
- latency

তবে PII বা secrets raw log করা যাবে না।

---

## 🇧🇩 Real-world Examples (বাংলাদেশি বাস্তব প্রেক্ষাপট)

### ১. bKash-style internal service communication

একটি mobile wallet platform-এ সাধারণ flow হতে পারে:

- app-gateway request নেয়
- auth-service token verify করে
- wallet-service balance fetch করে
- fraud-service risk score দেয়
- ledger-service posting করে
- notification-service SMS push করে

এখানে public edge-এ REST থাকতে পারে, কিন্তু internal east-west communication-এ RPC বেশি মানানসই।

কারণ:

- method-driven domain actions বেশি
- latency critical
- internal teams contract-controlled
- streaming/metadata useful

### ২. Daraz-style order orchestration

Daraz checkout flow-এ সম্ভাব্য RPC methods:

- `cart.GetActiveCart(userId)`
- `pricing.Calculate(cart, userTier, campaign)`
- `inventory.Reserve(items)`
- `payment.Authorize(method, amount)`
- `shipping.GetOptions(address, items)`

এই workflow resource-centric public REST endpoint দিয়েও expose করা যায়, কিন্তু internal orchestration action-heavy হওয়ায় RPC natural fit।

### ৩. Pathao-style ride dispatch

Ride matching highly interactive।

এখানে useful patterns:

- driver location stream
- fare estimate RPC
- bidirectional stream for dispatch events
- timeout-based retry

Possible methods:

- `dispatch.MatchDriver(request)`
- `pricing.EstimateFare(route)`
- `driverStream.Subscribe(driverId)`

### ৪. Prothom Alo-style content pipeline

Newsroom system-এ services থাকতে পারে:

- article-service
- moderation-service
- recommendation-service
- push-notification-service
- search-indexer

সব communication sync RPC হবে না।

ভালো mix হতে পারে:

- sync RPC for preview, moderation check
- async queue for indexing/push

### ৫. Chaldal-style grocery inventory and delivery

Checkout-এর সময় multiple fast RPC দরকার হতে পারে:

- stock check
- slot availability
- promo validation
- substitution recommendation

এখানে JSON-RPC যথেষ্ট হতে পারে যদি system তুলনামূলক simple হয়।

Scale বাড়লে gRPC/Thrift/Connect বেশি useful হতে পারে।

### ৬. Foodpanda-style delivery operations

Delivery system-এ fire-and-forget, callback, request-response — সব pattern একসাথে থাকে।

উদাহরণ:

- request-response: `GetDeliveryFee`
- fire-and-forget: `TrackAnalyticsEvent`
- callback: payment/partner order confirmation
- streaming: rider live location

### ৭. Grameenphone / GP digital platform

Telecom self-care app-এ operations থাকতে পারে:

- `CheckPackageEligibility`
- `ActivateBundle`
- `GetUsageSummary`
- `ReserveRecharge`

এখানে public app API REST হতে পারে, কিন্তু core BSS/OSS integration RPC-like contract-heavy হতে পারে।

### ৮. Merchant ecosystem

ধরুন বাংলাদেশি একটি marketplace partner onboarding platform আছে।

Possible split:

- public onboarding API → REST
- internal risk, KYC, document parsing → RPC
- final notification + search sync → queue/event

### Practical takeaway from BD context

বাংলাদেশি startup বা enterprise system-এ common pattern হয়:

- public/mobile/web edge → REST/GraphQL
- internal service-to-service → RPC
- background async workload → queue/event bus

এই hybrid approach-টাই সবচেয়ে বাস্তবসম্মত।

---

## 🗺️ ASCII Diagrams

### Diagram ১: Classic RPC call flow

```text
+-------------+         +----------------+        +----------------+        +------------------+
| Client App  | ----->  | Client Stub    | -----> |   Network      | -----> | Server Skeleton  |
| getBalance  |         | marshal params |        | HTTP/TCP/WS    |        | unmarshal + route|
+-------------+         +----------------+        +----------------+        +------------------+
                                                                                      |
                                                                                      v
                                                                            +------------------+
                                                                            | Service Method   |
                                                                            | execute business |
                                                                            +------------------+
                                                                                      |
                                                                                      v
+-------------+ <-----  +----------------+ <----- +----------------+ <----- +------------------+
| Return Data |         | Client Stub    |        |   Network      |        | Serialize Result |
| balance=4200|         | map response   |        |                |        | or map error     |
+-------------+         +----------------+        +----------------+        +------------------+
```

### Diagram ২: Local call illusion বনাম remote reality

```text
Developer thinks:

app.js
  |
  |----> pricingClient.estimateFare(route)
  |----> returns 185.00

Actually happens:

app.js
  |
  v
serialize(route)
  |
  v
TCP connect / reuse
  |
  v
send bytes over network
  |
  v
server decode
  |
  v
auth + routing + validation
  |
  v
execute estimateFare()
  |
  v
encode response
  |
  v
network back to caller
  |
  v
decode to JS/PHP object
```

### Diagram ৩: RPC vs REST mental model

```text
RPC:

Client -------------------------------> Service
        ChargeWallet(account, 500)

REST:

Client -------------------------------> /wallet-transactions
        POST {
          "account": "017...",
          "amount": 500,
          "type": "debit"
        }
```

### Diagram ৪: JSON-RPC request/response envelope

```text
HTTP POST /rpc

{
  "jsonrpc": "2.0",
  "id": 55,
  "method": "order.cancel",
  "params": {
    "orderId": "DZD-9001",
    "reason": "customer_request"
  }
}

                |
                v

{
  "jsonrpc": "2.0",
  "id": 55,
  "result": {
    "cancelled": true,
    "refundStatus": "pending"
  }
}
```

### Diagram ৫: Streaming patterns

```text
1) Server Streaming

Client ---- request ----> Server
Client <--- item 1 ------ Server
Client <--- item 2 ------ Server
Client <--- item 3 ------ Server
Client <--- complete ---- Server

2) Client Streaming

Client ---- chunk 1 ----> Server
Client ---- chunk 2 ----> Server
Client ---- chunk 3 ----> Server
Client <--- summary ----- Server

3) Bidirectional Streaming

Client <=== stream events ===> Server
```

### Diagram ৬: Client-side vs Proxy-side load balancing

```text
Client-side LB:

             +--> inventory-1
Client Lib --+--> inventory-2
             +--> inventory-3
                (registry aware)

Proxy-side LB:

Client ---> Envoy / Nginx / LB ---> inventory-1
                               \--> inventory-2
                               \--> inventory-3
```

### Diagram ৭: Service discovery with registry

```text
             +----------------------+
             |  Service Registry    |
             | Consul / etcd / DNS  |
             +----------+-----------+
                        ^
                        |
         register       | lookup
                        |
+----------------+      |       +----------------+
| pricing-svc-1  |------+       | order-service  |
| 10.0.0.11:9000 |              | client library |
+----------------+              +----------------+

+----------------+
| pricing-svc-2  |
| 10.0.0.12:9000 |
+----------------+
```

### Diagram ৮: Deadline propagation

```text
User Request total budget: 1000ms

Gateway (1000ms)
   |
   +--> order-service starts after 50ms, remaining 950ms
             |
             +--> inventory-service starts after 200ms, remaining 750ms
                         |
                         +--> warehouse-service gets only 300ms, not fresh 1000ms
```

### Diagram ৯: Circuit breaker around RPC

```text
           request
Client --------------> Circuit Breaker --------------> promo-service
                           |
                           |
                           +--> if failures > threshold
                           |
                           +--> OPEN state
                           |
                           +--> return fallback immediately
```

### Diagram ১০: Retry with idempotency key

```text
Client
  |
  |-- ChargeWallet(key=abc-123, amount=500) ---------------------->
  |<------------------------ timeout / unknown --------------------
  |
  |-- retry ChargeWallet(key=abc-123, amount=500) ---------------->
  |<------------------- server sees duplicate key -----------------
  |<------------------- returns original result -------------------
```

### Diagram ১১: Service mesh + RPC

```text
+-----------+       +-------------+       +-------------+
| order app  |<---->| order sidecar|<---->| mesh policy |
+-----------+       +-------------+       +-------------+
      |                     |
      | mTLS, retry, trace  |
      v                     v
+-----------+       +-------------+
| pay app   |<---->| pay sidecar  |
+-----------+       +-------------+
```

### Diagram ১২: Sync vs Async RPC

```text
Synchronous:

Client ---- request ----> Service
Client <--- response ---- Service
(Caller blocks)

Asynchronous:

Client ---- submit job --> Service
Client <--- job id ------ Service
... later ...
Client ---- get status --> Service
Client <--- done/result -- Service
```

### Diagram ১৩: RPC over different transports

```text
           +-------------------+
           | RPC Method Call   |
           +---------+---------+
                     |
     +---------------+-------------------------------+
     |               |               |               |
     v               v               v               v
   HTTP            TCP           WebSocket       Unix Socket
   easy debug      custom        bidi stream     same host fast
   proxy friendly  low-level     browser good    local IPC
```

### Diagram ১৪: Public API vs internal RPC hybrid architecture

```text
Mobile App / Web / Partner
          |
          v
   +--------------+
   | REST Gateway |
   +------+-------+
          |
          +--------------------+
          |                    |
          v                    v
   +-------------+      +-------------+
   | order-svc   |<---->| pricing-svc |
   |   RPC       |      |    RPC      |
   +-------------+      +-------------+
          |
          v
   +-------------+
   | event/queue |
   +-------------+
```

---

## 💻 PHP Code

### Example ১: Simple JSON-RPC server in PHP

```php
<?php
// server.php

declare(strict_types=1);

header('Content-Type: application/json; charset=utf-8');

$raw = file_get_contents('php://input');
$request = json_decode($raw, true);

if (!is_array($request)) {
    echo json_encode([
        'jsonrpc' => '2.0',
        'id' => null,
        'error' => [
            'code' => -32700,
            'message' => 'Parse error'
        ]
    ], JSON_UNESCAPED_UNICODE);
    exit;
}

$methods = [
    'wallet.getBalance' => function (array $params): array {
        $account = $params['account'] ?? null;

        $fakeStore = [
            '01712345678' => 4200.75,
            '01812345678' => 900.00,
        ];

        if (!$account || !isset($fakeStore[$account])) {
            throw new RuntimeException('Account not found');
        }

        return [
            'account' => $account,
            'balance' => $fakeStore[$account],
            'currency' => 'BDT'
        ];
    },

    'wallet.transferPreview' => function (array $params): array {
        $amount = (float) ($params['amount'] ?? 0);

        if ($amount <= 0) {
            throw new InvalidArgumentException('Amount must be greater than 0');
        }

        $fee = $amount >= 1000 ? 10 : 5;

        return [
            'amount' => $amount,
            'fee' => $fee,
            'totalDebit' => $amount + $fee,
        ];
    },
];

function handleRpc(array $request, array $methods): ?array
{
    $id = $request['id'] ?? null;
    $method = $request['method'] ?? null;
    $params = $request['params'] ?? [];

    if (($request['jsonrpc'] ?? null) !== '2.0') {
        return [
            'jsonrpc' => '2.0',
            'id' => $id,
            'error' => [
                'code' => -32600,
                'message' => 'Invalid Request'
            ]
        ];
    }

    if (!is_string($method) || !isset($methods[$method])) {
        return [
            'jsonrpc' => '2.0',
            'id' => $id,
            'error' => [
                'code' => -32601,
                'message' => 'Method not found'
            ]
        ];
    }

    try {
        $result = $methods[$method]((array) $params);

        // Notification হলে response পাঠাব না
        if (!array_key_exists('id', $request)) {
            return null;
        }

        return [
            'jsonrpc' => '2.0',
            'id' => $id,
            'result' => $result
        ];
    } catch (InvalidArgumentException $e) {
        return [
            'jsonrpc' => '2.0',
            'id' => $id,
            'error' => [
                'code' => -32602,
                'message' => $e->getMessage()
            ]
        ];
    } catch (Throwable $e) {
        return [
            'jsonrpc' => '2.0',
            'id' => $id,
            'error' => [
                'code' => -32000,
                'message' => $e->getMessage()
            ]
        ];
    }
}

$isBatch = array_is_list($request);

if ($isBatch) {
    $responses = [];
    foreach ($request as $item) {
        if (!is_array($item)) {
            $responses[] = [
                'jsonrpc' => '2.0',
                'id' => null,
                'error' => [
                    'code' => -32600,
                    'message' => 'Invalid batch item'
                ]
            ];
            continue;
        }

        $response = handleRpc($item, $methods);
        if ($response !== null) {
            $responses[] = $response;
        }
    }

    echo json_encode($responses, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
    exit;
}

$response = handleRpc($request, $methods);
if ($response !== null) {
    echo json_encode($response, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
}
```

### Example ২: PHP JSON-RPC client using cURL

```php
<?php
// client.php

declare(strict_types=1);

function rpcCall(string $url, string $method, array $params = [], int|string|null $id = 1): array
{
    $payload = [
        'jsonrpc' => '2.0',
        'id' => $id,
        'method' => $method,
        'params' => $params,
    ];

    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        CURLOPT_POSTFIELDS => json_encode($payload, JSON_UNESCAPED_UNICODE),
        CURLOPT_TIMEOUT => 5,
    ]);

    $responseBody = curl_exec($ch);

    if ($responseBody === false) {
        throw new RuntimeException('Transport error: ' . curl_error($ch));
    }

    $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($statusCode >= 400) {
        throw new RuntimeException('HTTP error: ' . $statusCode);
    }

    $decoded = json_decode($responseBody, true);

    if (!is_array($decoded)) {
        throw new RuntimeException('Invalid JSON-RPC response');
    }

    if (isset($decoded['error'])) {
        throw new RuntimeException('RPC error: ' . $decoded['error']['message']);
    }

    return $decoded['result'];
}

try {
    $result = rpcCall(
        'http://localhost:8080/server.php',
        'wallet.getBalance',
        ['account' => '01712345678']
    );

    print_r($result);
} catch (Throwable $e) {
    echo 'Failed: ' . $e->getMessage() . PHP_EOL;
}
```

### Example ৩: PHP-তে mini RPC framework

```php
<?php
// MiniRpcServer.php

declare(strict_types=1);

final class MiniRpcServer
{
    private array $handlers = [];

    public function register(string $method, callable $handler): void
    {
        $this->handlers[$method] = $handler;
    }

    public function handle(array $request): array
    {
        $id = $request['id'] ?? null;
        $method = $request['method'] ?? '';
        $params = (array) ($request['params'] ?? []);

        if (!isset($this->handlers[$method])) {
            return $this->error($id, 'METHOD_NOT_FOUND', 'Method not found');
        }

        try {
            $result = ($this->handlers[$method])($params, [
                'requestId' => $request['meta']['requestId'] ?? null,
                'caller' => $request['meta']['caller'] ?? 'unknown',
            ]);

            return [
                'id' => $id,
                'ok' => true,
                'result' => $result,
            ];
        } catch (DomainException $e) {
            return $this->error($id, 'FAILED_PRECONDITION', $e->getMessage());
        } catch (Throwable $e) {
            return $this->error($id, 'INTERNAL', $e->getMessage());
        }
    }

    private function error(int|string|null $id, string $code, string $message): array
    {
        return [
            'id' => $id,
            'ok' => false,
            'error' => [
                'code' => $code,
                'message' => $message,
            ],
        ];
    }
}
```

```php
<?php
// bootstrap.php

declare(strict_types=1);

require_once __DIR__ . '/MiniRpcServer.php';

$server = new MiniRpcServer();

$server->register('inventory.reserve', function (array $params, array $context): array {
    $items = $params['items'] ?? [];

    if ($items === []) {
        throw new DomainException('At least one item is required');
    }

    return [
        'reservationId' => 'RSV-' . random_int(1000, 9999),
        'reservedBy' => $context['caller'],
        'itemCount' => count($items),
        'expiresInSeconds' => 300,
    ];
});

$request = [
    'id' => 88,
    'method' => 'inventory.reserve',
    'params' => [
        'items' => [
            ['sku' => 'RICE-25KG', 'qty' => 1],
            ['sku' => 'OIL-5L', 'qty' => 2],
        ],
    ],
    'meta' => [
        'requestId' => 'req-bd-1001',
        'caller' => 'checkout-service',
    ],
];

print_r($server->handle($request));
```

### Example ৪: Service registration pattern in PHP

```php
<?php
// registry_demo.php

declare(strict_types=1);

final class ServiceRegistry
{
    private array $services = [];

    public function register(string $serviceName, string $address, array $meta = []): void
    {
        $this->services[$serviceName][] = [
            'address' => $address,
            'meta' => $meta,
            'healthy' => true,
        ];
    }

    public function deregister(string $serviceName, string $address): void
    {
        $instances = $this->services[$serviceName] ?? [];

        $this->services[$serviceName] = array_values(array_filter(
            $instances,
            fn (array $instance) => $instance['address'] !== $address
        ));
    }

    public function discover(string $serviceName): array
    {
        return array_values(array_filter(
            $this->services[$serviceName] ?? [],
            fn (array $instance) => $instance['healthy'] === true
        ));
    }
}

$registry = new ServiceRegistry();
$registry->register('pricing-service', '10.10.0.11:9000', ['zone' => 'dhaka-a']);
$registry->register('pricing-service', '10.10.0.12:9000', ['zone' => 'dhaka-b']);
$registry->register('inventory-service', '10.10.0.21:9100', ['zone' => 'dhaka-a']);

$pricingInstances = $registry->discover('pricing-service');
print_r($pricingInstances);
```

### PHP example design notes

উপরের code-গুলোর মধ্যে গুরুত্বপূর্ণ design idea হলো:

- request envelope আলাদা করা
- handler registry ব্যবহার করা
- transport concern আর business logic আলাদা রাখা
- error mapping standard করা
- request id/context propagate করা

এভাবে PHP/Laravel ecosystem-এ internal RPC framework বানানো সম্ভব, যদিও production scale-এ established framework/library ব্যবহার করা ভালো।

---

## 🟨 JavaScript / Node.js Code

### Example ১: JSON-RPC with `jayson`

```js
// server.js
const jayson = require('jayson');

const server = new jayson.Server({
  'wallet.getBalance': function (args, callback) {
    const balances = {
      '01712345678': 4200.75,
      '01812345678': 900.0,
    };

    const account = args.account;

    if (!account || !(account in balances)) {
      return callback({ code: -32001, message: 'Account not found' });
    }

    callback(null, {
      account,
      balance: balances[account],
      currency: 'BDT',
    });
  },

  'pricing.estimateFare': function (args, callback) {
    const distanceKm = Number(args.distanceKm || 0);
    const vehicleType = args.vehicleType || 'bike';

    if (distanceKm <= 0) {
      return callback({ code: -32602, message: 'distanceKm must be > 0' });
    }

    const base = vehicleType === 'car' ? 80 : 40;
    const perKm = vehicleType === 'car' ? 22 : 12;

    callback(null, {
      estimatedFare: base + distanceKm * perKm,
      currency: 'BDT',
    });
  },
});

server.http().listen(3000, () => {
  console.log('JSON-RPC server listening on http://localhost:3000');
});
```

```js
// client.js
const jayson = require('jayson/promise');

async function main() {
  const client = await jayson.Client.http('http://localhost:3000');

  const balance = await client.request('wallet.getBalance', {
    account: '01712345678',
  });

  console.log('Balance result:', balance.result);

  const fare = await client.request('pricing.estimateFare', {
    distanceKm: 12,
    vehicleType: 'bike',
  });

  console.log('Fare result:', fare.result);
}

main().catch((error) => {
  console.error(error);
});
```

### Example ২: tRPC complete example

```ts
// server/trpc.ts
import { initTRPC } from '@trpc/server';
import { z } from 'zod';

const t = initTRPC.create();

const products = [
  { id: 'p1', name: 'Teer Oil 5L', price: 850, stock: 30 },
  { id: 'p2', name: 'Aarong Milk Powder', price: 420, stock: 12 },
];

export const appRouter = t.router({
  getProduct: t.procedure
    .input(z.object({ id: z.string() }))
    .query(({ input }) => {
      const product = products.find((p) => p.id === input.id);
      if (!product) {
        throw new Error('Product not found');
      }
      return product;
    }),

  listProducts: t.procedure.query(() => {
    return products;
  }),

  reserveStock: t.procedure
    .input(
      z.object({
        id: z.string(),
        qty: z.number().int().positive(),
      })
    )
    .mutation(({ input }) => {
      const product = products.find((p) => p.id === input.id);

      if (!product) {
        throw new Error('Product not found');
      }

      if (product.stock < input.qty) {
        throw new Error('Insufficient stock');
      }

      product.stock -= input.qty;

      return {
        reservationId: `RSV-${Date.now()}`,
        remainingStock: product.stock,
      };
    }),
});

export type AppRouter = typeof appRouter;
```

```ts
// server/index.ts
import express from 'express';
import * as trpcExpress from '@trpc/server/adapters/express';
import { appRouter } from './trpc';

const app = express();

app.use(
  '/trpc',
  trpcExpress.createExpressMiddleware({
    router: appRouter,
  })
);

app.listen(4000, () => {
  console.log('tRPC server running on http://localhost:4000/trpc');
});
```

```ts
// client/trpcClient.ts
import { createTRPCProxyClient, httpBatchLink } from '@trpc/client';
import type { AppRouter } from '../server/trpc';

const client = createTRPCProxyClient<AppRouter>({
  links: [
    httpBatchLink({
      url: 'http://localhost:4000/trpc',
    }),
  ],
});

async function run() {
  const product = await client.getProduct.query({ id: 'p1' });
  console.log('Product:', product);

  const reservation = await client.reserveStock.mutate({ id: 'p1', qty: 2 });
  console.log('Reservation:', reservation);
}

run().catch(console.error);
```

```tsx
// React Query integration sketch
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from './server/trpc';
import { httpBatchLink } from '@trpc/client';

const trpc = createTRPCReact<AppRouter>();
const queryClient = new QueryClient();
const trpcClient = trpc.createClient({
  links: [httpBatchLink({ url: 'http://localhost:4000/trpc' })],
});

function ProductList() {
  const products = trpc.listProducts.useQuery();

  if (products.isLoading) return <div>Loading...</div>;

  return (
    <ul>
      {products.data?.map((product) => (
        <li key={product.id}>
          {product.name} - {product.price} BDT
        </li>
      ))}
    </ul>
  );
}

export default function App() {
  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        <ProductList />
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

### Example ৩: Custom RPC over WebSocket

```js
// ws-server.js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 5000 });

const methods = {
  'dispatch.estimateArrival': async ({ distanceKm, trafficLevel }) => {
    const trafficMultiplier = trafficLevel === 'high' ? 1.6 : 1.2;
    const etaMinutes = Math.ceil(distanceKm * trafficMultiplier * 4);

    return {
      etaMinutes,
      city: 'Dhaka',
    };
  },

  'news.subscribeBreaking': async () => {
    return {
      status: 'subscribed',
    };
  },
};

wss.on('connection', (socket) => {
  socket.on('message', async (raw) => {
    let request;

    try {
      request = JSON.parse(raw.toString());
    } catch (error) {
      socket.send(
        JSON.stringify({
          id: null,
          error: { code: 'BAD_JSON', message: 'Invalid JSON' },
        })
      );
      return;
    }

    const { id, method, params } = request;

    if (!methods[method]) {
      socket.send(
        JSON.stringify({
          id,
          error: { code: 'METHOD_NOT_FOUND', message: 'Unknown method' },
        })
      );
      return;
    }

    try {
      const result = await methods[method](params || {});
      socket.send(JSON.stringify({ id, result }));
    } catch (error) {
      socket.send(
        JSON.stringify({
          id,
          error: { code: 'INTERNAL', message: error.message },
        })
      );
    }
  });

  // server push example
  const interval = setInterval(() => {
    socket.send(
      JSON.stringify({
        method: 'driver.locationUpdate',
        params: {
          lat: 23.8103 + Math.random() / 100,
          lng: 90.4125 + Math.random() / 100,
          driverId: 'DRV-100',
        },
      })
    );
  }, 3000);

  socket.on('close', () => clearInterval(interval));
});

console.log('Custom RPC WebSocket server on ws://localhost:5000');
```

```js
// ws-client.js
const WebSocket = require('ws');

const socket = new WebSocket('ws://localhost:5000');

socket.on('open', () => {
  socket.send(
    JSON.stringify({
      id: 1,
      method: 'dispatch.estimateArrival',
      params: {
        distanceKm: 6.5,
        trafficLevel: 'high',
      },
    })
  );
});

socket.on('message', (raw) => {
  const message = JSON.parse(raw.toString());

  if (message.id === 1) {
    console.log('RPC response:', message.result || message.error);
    return;
  }

  if (message.method === 'driver.locationUpdate') {
    console.log('Stream event:', message.params);
  }
});
```

### Node.js example design notes

Node.js ecosystem-এ RPC implement করার সময় useful practices:

- request schema validate করো (`zod`, `joi`)
- method naming convention রাখো (`service.action`)
- correlation id propagate করো
- timeout wrapper দাও
- retry client-side carefully করো
- WebSocket stream-এ heartbeat/ping-pong রাখো

---

## 🎯 When to Use (কখন কোন RPC framework বা approach ব্যবহার করবেন)

### Decision Table: কোন পরিস্থিতিতে কোনটি?

| Scenario | Recommended Choice | কারণ |
|----------|--------------------|------|
| public CRUD API | REST | browser-friendly, HTTP semantics, docs tooling |
| public flexible mobile/web data fetching | GraphQL | client-shaped data |
| internal low-latency polyglot microservices | gRPC / Thrift | strong contract + performance |
| TypeScript full-stack product | tRPC | end-to-end type safety |
| browser-native protobuf RPC | Connect RPC | modern frontend-friendly RPC |
| simple internal admin service | JSON-RPC | very low complexity |
| legacy XML integration | XML-RPC | legacy compatibility |
| data ecosystem / schema evolution heavy | Avro RPC | schema resolution strong |
| one-way async background task | Message Queue | decoupled processing |
| same-host daemon communication | Unix socket RPC | low overhead local IPC |

### JSON-RPC কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- দ্রুত internal tool বানাতে হবে
- debugging সহজ রাখতে চান
- human-readable payload ভালো লাগে
- strict type system দরকার নেই

এড়িয়ে চলুন যখন:

- large polyglot microservice platform
- strong contract governance দরকার
- performance খুব critical

### Thrift কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- legacy Thrift estate already আছে
- multi-language codegen দরকার
- transport flexibility valuable

### Avro RPC কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- data platform-centric architecture
- schema evolution long-term concern
- analytics ecosystem already Avro-based

### gRPC কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- high-performance internal APIs
- streaming দরকার
- strict protobuf contract দরকার
- mesh-friendly service communication চান

আরও গভীরতার জন্য দেখুন:

- **`06-api-design/grpc.md`**

### tRPC কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- full-stack TypeScript app
- frontend/backend একই repo/team
- code generation ছাড়া typed RPC চান

এড়িয়ে চলুন যখন:

- PHP, Go, Java consumer থাকবে
- public external API standard করতে হবে

### Connect RPC কবে ব্যবহার করবেন?

ব্যবহার করুন যখন:

- browser-first protobuf RPC দরকার
- gRPC-Web friction কমাতে চান
- modern frontend platform বানাচ্ছেন

### REST বনাম RPC — দ্রুত thumb rules

RPC বেছে নিন যদি:

- action-oriented domain
- internal services
- low latency
- contract-first integration
- streaming useful

REST বেছে নিন যদি:

- external/public API
- resource modeling natural
- caching/CDN important
- third-party onboarding priority

GraphQL বেছে নিন যদি:

- frontend-specific data shaping দরকার
- multiple resources এক call-এ প্রয়োজন

Message Queue বেছে নিন যদি:

- immediate response না লাগলেও চলে
- retry + durability + decoupling দরকার

### Bangladesh practical recommendation matrix

| Organization type | Suggested mix |
|-------------------|---------------|
| early-stage startup | public REST + internal JSON-RPC |
| TS-heavy startup | public REST + internal tRPC |
| fintech scale-up | public REST + internal gRPC + async queue |
| marketplace with many teams | edge REST/GraphQL + internal gRPC/Connect + events |
| content platform | public REST/GraphQL + internal RPC + search queue |
| enterprise with legacy systems | REST edge + Thrift/XML-RPC/Avro where needed |

### Anti-patterns

নিচের ভুলগুলো এড়িয়ে চলা উচিত:

- remote call-কে local call ধরে latency ignore করা
- every tiny function as separate RPC করা
- no timeout set করা
- blind retry on non-idempotent operation
- error code standard না রাখা
- service discovery hardcode করা
- internal traffic unencrypted রাখা
- no tracing/correlation id রাখা

### Practical architecture recommendation

সাধারণত সবচেয়ে balanced architecture হয়:

1. Public edge → REST বা GraphQL
2. Internal synchronous domain calls → RPC
3. Background workflows → queue/event bus
4. Service-to-service security → mTLS + token/scope
5. Reliability → timeout + retry + circuit breaker + observability

---

## 🧠 Key Takeaways

- RPC-এর মূল idea: remote method-কে local call-এর মতো feel করানো, কিন্তু local call-এর মতো treat করা যাবে না।
- History জানলে বোঝা যায় RPC নতুন নয়; ONC RPC, DCE RPC, CORBA, DCOM—সবই একই problem space-এর evolution।
- Client stub, server skeleton, serialization, transport, service implementation—এই chain বুঝলেই RPC architecture পরিষ্কার হয়।
- IDL strong contract দেয়; কিন্তু সব framework-এ IDL লাগে না। tRPC type inference দিয়ে অন্য পথ নেয়।
- JSON-RPC simple ও practical; gRPC/Thrift/Avro stronger contract-oriented; Connect ও tRPC modern DX-oriented।
- RPC verb-oriented; REST resource-oriented; GraphQL data-shaping-oriented; Message Queue async decoupling-oriented।
- Request-response, streaming, fire-and-forget, callback—সবগুলো RPC pattern এক architecture-এ coexist করতে পারে।
- Retry কেবল retry-worthy error-এ করুন; money movement-এর মতো operation-এ idempotency key critical।
- Timeout এবং deadline propagation না করলে cascading failure খুব দ্রুত হয়।
- Load balancing client-side বা proxy-side—দুটোরই জায়গা আছে; observability ও operational simplicity বিবেচনা করুন।
- Service discovery hardcoded address-এর বদলে registry/DNS/service mesh দিয়ে করা উচিত।
- Error response machine-readable হতে হবে; শুধু human message যথেষ্ট নয়।
- Security-তে mTLS, token, per-method authorization, input validation—সব দরকার।
- Bangladesh context-এ public API-এর জন্য REST, internal service-to-service-এর জন্য RPC, async work-এর জন্য queue—এই hybrid model সাধারণত সবচেয়ে practical।
- gRPC specifics আলাদা ভাবে জানতে চাইলে অবশ্যই `grpc.md` পড়ুন; এই file general RPC landscape বোঝার জন্য।
- ভাল RPC design মানে শুধু fast protocol না—বরং clear contract, safe retries, observable behavior, secure transport, এবং realistic failure handling।
