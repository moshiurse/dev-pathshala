# 🏗️ মাল্টি-টিয়ার আর্কিটেকচার (Multi-Tier Architecture) — System Design & Infrastructure Deep Dive

> **গুরুত্বপূর্ণ পার্থক্য:** `07-architectural-patterns/layered-architecture.md` ফাইলটি মূলত **কোড-লেভেল logical layers** (Presentation, Business Logic, Data Access) নিয়ে।
> এই ফাইলটি পুরোপুরি **physical deployment tiers** নিয়ে — অর্থাৎ কোন কম্পোনেন্ট কোন সার্ভারে, কোন নেটওয়ার্ক জোনে, কোন লোড ব্যালেন্সারের পেছনে, কিভাবে আলাদা স্কেল হবে, কিভাবে firewall rule বসবে, এবং কিভাবে একটি production system বাস্তবে deploy হবে।

---

## 📌 Definition

### ১. Multi-Tier Architecture — সংজ্ঞা

**Multi-Tier Architecture** হলো এমন একটি infrastructure-centric deployment pattern যেখানে একটি system-কে একাধিক **physical tier** বা **network-isolated runtime zone**-এ ভাগ করা হয়।

প্রতিটি tier সাধারণত:

- আলাদা server group-এ deploy হয়
- আলাদা subnet/VLAN/VPC segment-এ থাকে
- আলাদা scaling policy পায়
- আলাদা security rule পায়
- আলাদা failure boundary তৈরি করে
- অনেক সময় আলাদা team ownership-ও পায়

সহজ ভাষায়:

- Browser/Mobile user-facing অংশ এক tier
- Reverse proxy / web delivery আরেক tier
- API / application logic আরেক tier
- Database / cache / message queue আলাদা tier

এখানে **tier** মানে শুধু code folder structure না।
এখানে **tier** মানে network hop, machine boundary, process isolation, deployment blast radius, এবং operational independence।

#### Logical Layer vs Physical Tier

অনেকেই Layered Architecture আর Multi-Tier Architecture-কে এক জিনিস মনে করে।
এটি বড় একটি ভুল।

```
+----------------------+---------------------------------------------+
| ধারণা                | কী বোঝায়                                   |
+----------------------+---------------------------------------------+
| Logical Layer        | কোডের ভেতরের responsibility separation      |
| Physical Tier        | deployment/runtime/network separation        |
+----------------------+---------------------------------------------+
| উদাহরণ               | Controller → Service → Repository           |
| উদাহরণ               | CDN → Nginx → App Server → DB Server        |
+----------------------+---------------------------------------------+
| focus                | maintainability, code organization          |
| focus                | scaling, security, isolation, operations    |
+----------------------+---------------------------------------------+
```

আরও পরিষ্কারভাবে:

```
Layered Architecture (একই সার্ভারের ভেতরও হতে পারে)
===================================================

┌──────────────────────────────────────────────┐
│                One App Process               │
│                                              │
│  Controller Layer                            │
│  Service Layer                               │
│  Repository Layer                            │
│  ORM / DB Access Layer                       │
└──────────────────────────────────────────────┘


Multi-Tier Architecture (ভিন্ন server/network boundary)
=======================================================

Client Device
    │
    ▼
Web Tier Server Group
    │
    ▼
Application Tier Server Group
    │
    ▼
Data Tier Server Group
```

#### Physical deployment tiers বলতে কী বোঝায়?

Physical tier সাধারণত এই জিনিসগুলোর এক বা একাধিক বোঝায়:

- আলাদা VM / EC2 / container group
- আলাদা subnet
- আলাদা security group
- আলাদা load balancer target group
- আলাদা autoscaling group
- আলাদা IAM/secret boundary
- আলাদা monitoring dashboard

#### কেন tier আলাদা করা হয়?

##### ১) Security isolation

যদি web tier compromise হয়, attacker যেন সরাসরি database tier-এ পৌঁছাতে না পারে।

##### ২) Independent scaling

Static content-এর জন্য web tier 20 instance লাগতে পারে,
কিন্তু database tier হয়তো 2 node-ই যথেষ্ট।

##### ৩) Fault isolation

এক tier-এ spike হলে পুরো system যেন একসাথে না ভেঙে পড়ে।

##### ৪) Team ownership

Platform team web tier manage করতে পারে,
application team app tier,
DBA/SRE data tier।

##### ৫) Technology heterogeneity

একই system-এ Nginx, PHP-FPM, Node.js, Redis, MySQL, Kafka — সব আলাদা tier-এ চলতে পারে।

##### ৬) Compliance and auditability

বিশেষ করে finance, telecom, healthcare domain-এ database tier direct internet access থেকে বিচ্ছিন্ন রাখতেই হয়।

#### Bangladesh context-এ এর মূল্য

বাংলাদেশের production systems-এ multi-tier খুব common কারণ:

- **bKash**-এ transaction path highly secured হতে হয়
- **Daraz**-এ campaign day-তে web traffic আর checkout traffic আলাদা pressure দেয়
- **Pathao**-তে real-time location API আর ride-history storage-এর load একরকম না
- **Prothom Alo**-তে read-heavy news traffic cache/CDN friendly
- **Chaldal**-এ inventory এবং order processing tier আলাদা করে scale করতে হয়
- **GP / Robi / Banglalink**-এর enterprise portal-এ DMZ + app zone + DB zone খুব common pattern

---

#### Evolution: Single-Tier → 2-Tier → 3-Tier → 4-Tier → N-Tier

### Single-Tier

সবকিছু একই machine/process-এ।

```
┌──────────────────────────────────────────────┐
│                SINGLE TIER                   │
├──────────────────────────────────────────────┤
│ UI + Business Logic + Data + Files সব একসাথে │
│ এক machine-এ                                  │
└──────────────────────────────────────────────┘
```

**কোথায় দেখা যায়:**

- local desktop app
- ছোট internal tool
- prototype
- developer machine demo

**সমস্যা:**

- scale করা কঠিন
- security segmentation নেই
- team separation নেই
- blast radius বিশাল

### 2-Tier

Client সরাসরি database/server-এর সাথে কথা বলে।

### 3-Tier

Client → App/Web → Database

এটাই classic web architecture।

### 4-Tier

Client → Web Tier → App Tier → Data Tier

Web delivery আর application logic আলাদা।

### N-Tier

Enterprise-scale decomposition:

- Client Tier
- CDN/Edge Tier
- WAF / API Gateway Tier
- Web Tier
- Application / Service Tier
- Cache Tier
- Queue Tier
- Background Worker Tier
- Data Tier
- Analytics Tier

#### Evolution ASCII timeline

```
1995-2005                      2005-2015                     2015-Now
==========                     ==========                    ==========

Single / 2-Tier         →      3-Tier Web            →      N-Tier Cloud Native

[Client+Logic+DB]              [Client]                     [Client]
     or                        [App/Web]                    [CDN/WAF]
[Client]                       [DB]                         [API Gateway]
[DB]                                                        [Services]
                                                            [Cache]
                                                            [Queue]
                                                            [Workers]
                                                            [DB/Analytics]
```

#### একটি গুরুত্বপূর্ণ observation

Layer count বাড়লেই architecture ভালো হয় না।
ভালো architecture হলো যেখানে tier separation **need-driven**।

যদি একটি Prothom Alo style content site হয়:

- CDN tier
- web tier
- CMS app tier
- DB tier

যথেষ্ট হতে পারে।

কিন্তু bKash-style regulated payments system-এ:

- API edge
- auth tier
- transaction tier
- ledger tier
- fraud/risk tier
- queue tier
- audit tier
- core banking integration tier
- DB tier

বেশি যুক্তিযুক্ত।

---

### ২. Common Tier Patterns

#### 2-Tier (Client-Server)

এখানে client তুলনামূলক “fat” হয়।
Business logic-এর বড় অংশ client side-এ থাকতে পারে,
আর client সরাসরি database/server-এর সাথে connect করে।

```
+-------------------+        SQL / Proprietary Protocol        +------------------+
| Fat Client        |  ------------------------------------>   | Database Server  |
| Desktop App       |                                           | Oracle / SQL     |
| Branch Software   |  <------------------------------------    |                  |
+-------------------+            Result Set / Rows              +------------------+
```

**Typical characteristics:**

- client machine-এ business rules থাকে
- DB credential client side-এ থাকতে পারে
- schema change হলে client update লাগে
- network trust model দুর্বল

**Bangladesh legacy example:**

পুরনো banking branch software বা ERP system যেখানে branch desktop app সরাসরি central Oracle database-এ query চালায়।

**Pros:**

- simple
- কম component
- low initial cost

**Cons:**

- database exposed
- client upgrade painful
- scaling poor
- security risky

#### 2-Tier Bangladesh example

```
Sonali Bank / Legacy Branch Scenario
====================================

Branch Operator PC
      │
      │ Oracle Client / ODBC
      ▼
Core DB Server

Problem:
- Branch PC compromise হলে DB credential leak হতে পারে
- WAN latency directly DB query-কে impact করে
- Offline tolerance কম
```

---

#### 3-Tier Architecture

এটি সবচেয়ে common web architecture।

তিনটি tier:

1. **Client Tier** — Browser / Mobile App
2. **Application Tier** — Web server + app logic
3. **Data Tier** — Database server

```
┌──────────────────┐      HTTPS      ┌──────────────────┐      SQL      ┌──────────────────┐
│ Client Tier      │  ------------>  │ Application Tier │  ---------->  │ Data Tier        │
│ Browser/Mobile   │                 │ PHP/Node/Java    │               │ MySQL/Postgres   │
└──────────────────┘  <------------  └──────────────────┘  <----------  └──────────────────┘
       HTML/JSON                             Business Logic                  Rows / Indexes
```

**Why it became dominant:**

- DB direct exposure বন্ধ করে
- business logic centralized হয়
- client thin হয়
- deployment manageable
- web/mobile support সহজ হয়

**Bangladesh example:**

একটি University admission portal:

- Student browser/mobile = client tier
- PHP/Laravel app server = application tier
- MySQL = data tier

#### 3-Tier request flow

```
User clicks "Pay Admission Fee"
             │
             ▼
Client Tier (Browser)
             │ HTTPS
             ▼
Application Tier
- authenticate user
- validate form
- create payment intent
- call gateway
- write transaction row
             │ SQL
             ▼
Data Tier
- students
- applications
- payments
- audit_logs
```

#### 3-Tier strong use cases

- ERP
- CMS
- e-commerce checkout
- admin portal
- business dashboard

---

#### 4-Tier Architecture

এখানে web delivery concern আর business logic concern আলাদা করা হয়।

চারটি tier:

1. Client Tier
2. Web Tier
3. Application Tier
4. Data Tier

```
┌──────────────┐   HTTPS   ┌──────────────┐   FastCGI/HTTP   ┌──────────────┐   SQL   ┌──────────────┐
│ Client Tier  │ ------->  │ Web Tier     │ -------------->  │ App Tier     │ ----->  │ Data Tier    │
│ Browser/App  │           │ CDN/Nginx    │                  │ PHP/Node API  │         │ DB + Cache   │
└──────────────┘ <-------   └──────────────┘ <--------------  └──────────────┘ <-----  └──────────────┘
      HTML                static files / TLS termination         JSON / render           rows / cache
```

**Web tier-এর কাজ:**

- TLS termination
- static file serving
- caching header
- request filtering
- WAF integration
- load balancing to app tier

**App tier-এর কাজ:**

- authentication
- business rules
- domain workflows
- API generation
- queue publish

**Data tier-এর কাজ:**

- primary DB
- replicas
- Redis
- search engine

**Bangladesh example:**

**Prothom Alo / The Daily Star** style media portal:

- Client tier: browser/app
- Web tier: CDN + Nginx
- App tier: CMS API + rendering services
- Data tier: MySQL + Redis

কারণ static asset, image, article page cache, API, admin CMS — সব একরকম workload না।

---

#### N-Tier (Enterprise)

N-Tier architecture-এ নির্দিষ্ট সংখ্যা fix না।
System-এর operational need অনুযায়ী tier বাড়তে থাকে।

Typical enterprise N-tier:

1. Client Tier
2. CDN / Edge Tier
3. WAF / API Gateway Tier
4. Web Tier
5. Application / Service Tier
6. Caching Tier
7. Message Queue Tier
8. Background Processing Tier
9. Data Tier
10. Analytics / Reporting Tier

```
Internet User
    │
    ▼
+------------------+
| CDN / Edge Tier  |
+------------------+
    │
    ▼
+------------------+
| WAF / API GW     |
+------------------+
    │
    ▼
+------------------+
| Web Tier         |
+------------------+
    │
    ▼
+------------------+
| Service Tier     |
+------------------+
    │      │      │
    │      │      └───────────────┐
    │      ▼                      │
    │  +-----------+              │
    │  | Cache     |              │
    │  +-----------+              │
    │                             │
    ▼                             ▼
+-----------+               +-------------+
| Queue     |               | Data Tier   |
+-----------+               +-------------+
    │
    ▼
+------------------+
| Worker Tier      |
+------------------+
```

**কখন লাগে:**

- large enterprise system
- regulated workloads
- mixed traffic pattern
- asynchronous processing-heavy system
- many teams independently deploy করে

**Bangladesh example candidates:**

- Daraz mega-sale platform
- bKash payment platform
- large telecom self-care platform
- Pathao ride + food + pay ecosystem

---

### ৩. Tier Boundaries & Communication

Multi-tier architecture-এর আসল শক্তি আসে **boundary design** থেকে।

Tier boundary মানে:

- network hop
- protocol change
- trust boundary
- rate limit boundary
- secret boundary
- scaling boundary
- operational ownership boundary

#### Network segmentation

সাধারণভাবে production environment-এ tier segmentation এভাবে হতে পারে:

- **Public/Internet Zone**
- **DMZ / Edge Zone**
- **Private Application Zone**
- **Restricted Data Zone**

```
                   INTERNET
                       │
                       ▼
          +---------------------------+
          | Public Entry              |
          | DNS / CDN / WAF / LB      |
          +---------------------------+
                       │
             ======== Firewall ========
                       │
                       ▼
          +---------------------------+
          | DMZ / Web Zone            |
          | Nginx / Reverse Proxy     |
          +---------------------------+
                       │
             ======== Firewall ========
                       │
                       ▼
          +---------------------------+
          | Private App Zone          |
          | API / Services / Workers  |
          +---------------------------+
                       │
             ======== Firewall ========
                       │
                       ▼
          +---------------------------+
          | Restricted Data Zone      |
          | DB / Redis / MQ           |
          +---------------------------+
```

#### DMZ কী?

DMZ (Demilitarized Zone) হলো এমন একটি network segment যা public traffic গ্রহণ করতে পারে কিন্তু deep internal systems-এ unrestricted access পায় না।

Typical DMZ components:

- reverse proxy
- WAF
- load balancer
- bastion-host-সদৃশ tightly controlled component

#### Private network ও database network

App tier সাধারণত private subnet-এ থাকে।
DB tier আরও restricted subnet-এ থাকে যেখানে:

- internet route নেই
- inbound শুধু app tier থেকে আসে
- admin access jump host/VPN ছাড়া বন্ধ থাকে

#### Communication patterns between tiers

##### Synchronous communication

- HTTP/HTTPS
- gRPC
- FastCGI
- SQL/TCP

**যখন দরকার:**

- immediate response
- read request
- checkout confirmation
- auth verification

##### Asynchronous communication

- Kafka
- RabbitMQ
- SQS
- Redis Streams
- background jobs

**যখন দরকার:**

- email sending
- report generation
- notification fan-out
- image processing
- fraud analysis

#### Sync vs Async diagram

```
Synchronous Path
================

Client -> API Gateway -> App Service -> DB
   ^                                     │
   └--------------- response ------------┘


Asynchronous Path
=================

Client -> API -> Queue -> Worker -> DB/Email/SMS
   ^
   └------ immediate ACK / accepted -----
```

#### Security zones and firewall rules

একটি ভালো multi-tier design-এ **"allow only what is needed"** rule চলে।

উদাহরণ:

- Internet → CDN/WAF: 80/443 allowed
- CDN/WAF → Web tier: 443 allowed
- Web tier → App tier: 8080 or 9000 internal only
- App tier → Redis: 6379 allowed only from app SG
- App tier → MySQL: 3306 allowed only from app SG
- Worker tier → Queue: 5672/9092 only internal
- Bastion/VPN → DB: limited admin port access

```
+-----------------+----------------------+-------------------------------+
| From            | To                   | Rule                          |
+-----------------+----------------------+-------------------------------+
| Internet        | CDN/WAF              | Allow 80/443                  |
| CDN/WAF         | Web Tier             | Allow 443                     |
| Web Tier        | App Tier             | Allow 8080/9000 internal only |
| App Tier        | Redis Tier           | Allow 6379                    |
| App Tier        | DB Tier              | Allow 3306/5432               |
| Worker Tier     | Queue Tier           | Allow 5672/9092               |
| Admin VPN       | Bastion              | Allow SSH                     |
| Bastion         | DB Tier              | Allow admin only              |
+-----------------+----------------------+-------------------------------+
```

#### Latency implications of crossing tier boundaries

প্রতিটি tier hop latency যোগ করে।

যেমন:

- client → CDN: 20ms
- CDN → ALB: 5ms
- ALB → web/app: 2ms
- app → Redis: 1ms
- app → DB: 3ms
- app → queue publish: 2ms

Total latency accumulate হয়।

#### Latency budget example

```
User request total budget = 200ms

DNS + TLS                    = 25ms
Edge / CDN / WAF             = 10ms
Load Balancer                =  3ms
App processing               = 40ms
Redis read                   =  2ms
DB query                     = 25ms
Serialization                =  5ms
Network return path          = 20ms
-------------------------------------
Total                        = 130ms
```

Tier বেশি মানেই flexibility বেশি,
কিন্তু hop বেশি মানেই latency ও failure probability-ও বাড়ে।

#### Boundary design principle

প্রতিটি নতুন tier যোগ করার আগে জিজ্ঞেস করুন:

- এটি কি আলাদা security boundary দেয়?
- এটি কি আলাদা scaling boundary দেয়?
- এটি কি failure blast radius কমায়?
- এটি কি operational clarity বাড়ায়?
- নাকি শুধু diagram-কে fancy বানায়?

---

### ৪. Scaling Each Tier Independently

Multi-tier architecture-এর সবচেয়ে বড় operational benefit হলো **independent scaling**।

সব tier একইভাবে scale হয় না।

#### Web tier scaling

Web tier সাধারণত stateless হলে horizontal scale খুব সহজ।

```
                 +------------------+
Traffic -------> | Load Balancer    |
                 +--------+---------+
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        +---------+  +---------+  +---------+
        | Nginx-1  |  | Nginx-2  |  | Nginx-3  |
        +---------+  +---------+  +---------+
```

**Use when:**

- TLS termination heavy
- static asset delivery heavy
- reverse proxy CPU saturation

#### Application tier scaling

API/app tier-ও horizontal scale-friendly হলে সহজে বাড়ানো যায়।

```
                 +------------------+
                 | Internal LB       |
                 +--------+---------+
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
   +-------------+  +-------------+  +-------------+
   | App Node 1  |  | App Node 2  |  | App Node 3  |
   +-------------+  +-------------+  +-------------+
```

**শর্ত:**

- session externalized হতে হবে
- uploads shared storage-এ যাবে
- local memory state-এর উপর নির্ভর করা যাবে না

#### Data tier scaling

Data tier সবচেয়ে কঠিন।

কারণ data consistency, storage, locks, replication lag — সব issue এখানে।

##### Vertical scaling

- বেশি CPU
- বেশি RAM
- faster SSD/NVMe
- bigger instance class

এটি DB tier-এ common কারণ:

- DB stateful
- scale up operationally simpler
- query planner ও buffer pool benefit পায়

##### Horizontal read scaling with replicas

```
                 +------------------+
Writes --------> | Primary DB       |
                 +--------+---------+
                          │ Replication
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     +---------+     +---------+     +---------+
     | Replica1|     | Replica2|     | Replica3|
     +---------+     +---------+     +---------+
          ▲               ▲               ▲
          └------- Read Queries ----------┘
```

**Use case:**

- Prothom Alo article reads
- Daraz product catalog reads
- Chaldal inventory view

##### Sharding (advanced)

শুধু তখন, যখন single primary আর যথেষ্ট না।

- user-based shard
- region-based shard
- tenant-based shard

#### Caching tier as separate scalable tier

Redis/Memcached tier database pressure কমাতে আলাদা রাখা হয়।

```
Client
  │
  ▼
App Tier
  │
  ├── GET cache:key ------> Cache Tier (Redis Cluster)
  │         │
  │         ├── hit  -> return fast
  │         └── miss -> query DB
  │
  └── SQL -------------> Data Tier
```

#### Auto-scaling strategies per tier

##### Web tier auto scaling metrics

- requests per second
- CPU
- active connections
- network throughput

##### App tier auto scaling metrics

- CPU
- memory
- p95 latency
- queue length
- concurrent requests

##### Worker tier auto scaling metrics

- queue depth
- job age
- jobs/sec

##### DB tier auto scaling metrics

DB fully auto-scale করা কঠিন।
সাধারণত:

- storage autoscale
- read replica autoscale
- manual/assisted vertical scaling

#### Example: Daraz 11.11 sale day scaling

```
Tier                    Normal Day        Campaign Peak
------------------------------------------------------
CDN Edge Nodes          managed           managed
Web Tier                4                 20
App Tier                8                 40
Redis Nodes             2                 6
DB Primary              1 large           1 xlarge
DB Replicas             2                 6
Worker Tier             3                 25
```

এখানে বুঝতে হবে,
সব tier একই multiplier-এ scale করে না।

---

### ৫. Load Balancing Between Tiers

Multi-tier architecture-এ শুধু client-facing edge-এ load balancer থাকে না,
অনেক সময় **tier-to-tier** communication-এই load balancing হয়।

#### কোথায় কোথায় load balancing থাকে?

- Internet → edge LB
- Edge → web tier
- Web tier → app tier
- App tier → service tier
- App tier → read replicas
- Queue consumers → worker partitioning

#### L4 vs L7 Load Balancer

##### L4 (Transport Layer)

IP + port level decision।

**উদাহরণ:**

- AWS NLB
- TCP proxy
- LVS

**ভালো যখন:**

- ultra low latency
- TLS passthrough
- raw TCP
- database proxying

##### L7 (Application Layer)

HTTP-aware routing।

**উদাহরণ:**

- Nginx
- HAProxy
- AWS ALB
- Envoy

**ভালো যখন:**

- path-based routing
- host-based routing
- header-based routing
- auth / WAF integration

#### L4 vs L7 quick comparison

```
+----------------------+-------------------------------+------------------------------+
| Aspect               | L4 Load Balancer              | L7 Load Balancer             |
+----------------------+-------------------------------+------------------------------+
| Decision basis       | IP, port, TCP/UDP             | URL, host, header, cookie    |
| Speed                | খুব দ্রুত                     | তুলনামূলক বেশি overhead      |
| Smart routing        | সীমিত                         | শক্তিশালী                    |
| TLS termination      | optional/pass-through         | common                       |
| Sticky session       | possible                      | richer options               |
| Best use             | raw traffic, DB, TCP          | web/API routing              |
+----------------------+-------------------------------+------------------------------+
```

#### Health checks

LB health check ছাড়া multi-tier setup fragile হয়ে যায়।

Health check types:

- TCP connect check
- HTTP 200 `/health`
- deep health `/ready`
- dependency-aware health

**Best practice:**

- `/live` = process alive?
- `/ready` = dependencies okay? serve traffic?
- `/health` = summary

#### Sticky session considerations

যদি session app node memory-তে রাখেন,
তাহলে LB sticky session লাগতে পারে।

কিন্তু modern multi-tier setup-এ preferred হলো:

- Redis session store
- JWT/stateless auth
- shared store

তাহলে sticky session avoid করা যায়।

#### Connection pooling between tiers

Tier boundary যত বাড়ে, connection overhead তত গুরুত্বপূর্ণ হয়।

**উদাহরণ:**

- Nginx keepalive to upstream
- App → DB connection pool
- App → Redis connection reuse
- ProxySQL / PgBouncer

```
Without Pooling
===============

Request 1 -> open DB conn -> query -> close
Request 2 -> open DB conn -> query -> close
Request 3 -> open DB conn -> query -> close

High overhead

With Pooling
============

Pool has 50 warm connections
Request uses pooled conn
query done
conn returned to pool
```

#### Cross-tier load balancing example

```
Internet
   │
   ▼
+----------------+
| Edge LB (L7)   |
+-------+--------+
        │
   ┌────┴────┐
   ▼         ▼
+------+   +------+
|Web-1 |   |Web-2 |
+--+---+   +---+--+
   │           │
   └────┬──────┘
        ▼
+----------------+
| Internal LB    |
+-------+--------+
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
+----+ +----+ +----+
|A-1| |A-2| |A-3|
+----+ +----+ +----+
```

#### Failure scenario

যদি health check weak হয়,
LB traffic unhealthy node-এ পাঠাতে থাকবে,
যার ফলে:

- timeout বাড়বে
- retry storm হবে
- upstream saturation হবে
- cascading failure হবে

---

### ৭. Multi-Tier in Cloud (AWS/GCP)

Cloud-এ multi-tier deployment আরও natural হয়ে গেছে কারণ managed networking primitives সহজলভ্য।

#### AWS reference pattern

```
User
 │
 ▼
CloudFront
 │
 ▼
AWS WAF
 │
 ▼
Application Load Balancer
 │
 ├───────────────► ECS/EKS Web/App Services
 │                     │
 │                     ├────────► ElastiCache Redis
 │                     ├────────► RDS MySQL / PostgreSQL
 │                     └────────► SQS / MSK / Worker
 │
 └───────────────► Static assets in S3
```

#### Common AWS mapping

- **Client/Edge Tier** → Route53 + CloudFront + WAF
- **Web/App Tier** → ALB + ECS/EKS/EC2 Auto Scaling
- **Cache Tier** → ElastiCache Redis
- **Data Tier** → RDS/Aurora
- **Queue Tier** → SQS / MSK / MQ
- **Worker Tier** → ECS services / Lambda / K8s workers

#### VPC isolation

একটি production AWS VPC-তে সাধারণত:

- public subnet for ALB/NAT
- private app subnet for services
- isolated data subnet for RDS/Redis

```
VPC 10.0.0.0/16
================

Public Subnet AZ-a      Public Subnet AZ-b
+------------------+    +------------------+
| ALB              |    | ALB              |
| NAT Gateway      |    | NAT Gateway      |
+------------------+    +------------------+

Private App Subnet AZ-a Private App Subnet AZ-b
+------------------+    +------------------+
| ECS/EKS Pods     |    | ECS/EKS Pods     |
| Internal APIs    |    | Internal APIs    |
+------------------+    +------------------+

Data Subnet AZ-a        Data Subnet AZ-b
+------------------+    +------------------+
| RDS Primary      |    | RDS Standby      |
| Redis Node       |    | Redis Replica    |
+------------------+    +------------------+
```

#### Security groups as tier contracts

AWS security group-গুলো tier boundary enforce করার excellent উপায়:

- ALB SG accepts 443 from internet
- App SG accepts 8080 from ALB SG only
- Redis SG accepts 6379 from App SG only
- DB SG accepts 3306 from App SG only

এটি effectively infrastructure-level contract।

#### Different availability zones for each tier

High availability-এর জন্য প্রতিটি critical tier ideally multi-AZ হবে।

```
AZ-a                         AZ-b
====                         ====

ALB node                     ALB node
Web/App pods                 Web/App pods
Redis primary/replica        Redis replica/primary
RDS primary                  RDS standby
Worker pods                  Worker pods
```

#### GCP equivalent

- Cloud CDN
- Cloud Armor
- Global HTTP(S) Load Balancer
- GKE / MIG
- Memorystore
- Cloud SQL / Spanner
- Pub/Sub

#### Cloud benefits

- provisioning fast
- autoscaling easy
- network policy strong
- managed observability integration
- managed DB/cache reduces ops load

#### But beware

Cloud managed service ব্যবহার করলেই architecture magically ভালো হয় না।
Tier contract, latency budget, failure policy — এগুলো আলাদাভাবে design করতে হবে।

---

### ৮. Advantages and Disadvantages

#### Advantages

##### ১) Security isolation

সবচেয়ে critical সুবিধা।
DB public internet থেকে completely hidden রাখা যায়।

##### ২) Independent scaling

CDN/cache-heavy workload আর transaction-heavy workload একসাথে scale করতে হয় না।

##### ৩) Fault containment

Web tier overload হলেও data tier সাথে সাথে down নাও হতে পারে যদি throttling, circuit breaker, cache layer থাকে।

##### ৪) Team independence

Infra team, platform team, backend team, data team আলাদা ownership নিতে পারে।

##### ৫) Technology heterogeneity

এক tier-এ Nginx,
এক tier-এ PHP,
এক tier-এ Node.js,
এক tier-এ Go worker,
এক tier-এ MySQL,
এক tier-এ Redis — perfectly valid।

##### ৬) Operational clarity

কোন bottleneck কোথায় — web, app, cache, DB — আলাদা করে দেখা যায়।

##### ৭) Compliance friendliness

Audit zone, PCI zone, restricted subnet, secret segmentation — সব সহজ হয়।

#### Disadvantages

##### ১) Network latency

প্রতিটি hop latency যোগ করে।

##### ২) Operational complexity

- more infra
- more config
- more dashboards
- more failure modes

##### ৩) Debugging difficulty

একটি request 7 tier পার হলে tracing ছাড়া root cause খুঁজে পাওয়া কঠিন।

##### ৪) Cost

- extra load balancer
- extra nodes
- extra NAT/data transfer
- extra observability bill

##### ৫) Deployment coordination

কখনো কখনো web tier, app tier, DB migration, worker tier — সব coordinated rollout দরকার হয়।

##### ৬) Misconfiguration risk

Firewall rule ভুল হলে production outage বা security hole — দুই-ই হতে পারে।

#### Trade-off summary

```
+------------------------+-----------------------------------------------+
| Benefit                | Cost / Complexity                             |
+------------------------+-----------------------------------------------+
| Better security        | More firewall / network config                |
| Better scaling         | More infra resources                          |
| Better isolation       | More latency hops                             |
| Better team ownership  | More coordination overhead                    |
| Better resilience      | More monitoring/tracing requirement           |
+------------------------+-----------------------------------------------+
```

---

### ৯. Multi-Tier vs Microservices

এ দুটি concept related, কিন্তু same জিনিস না।

#### Multi-Tier কী নিয়ে?

- physical deployment separation
- network boundaries
- infrastructure zones
- load balancers, subnets, firewalls

#### Microservices কী নিয়ে?

- domain/service boundary
- independently deployable services
- business capability separation
- API contracts

#### সম্পর্ক কী?

Microservices **multi-tier setup-এর ভিতরে** run করতে পারে।

উদাহরণ:

- CDN tier
- API Gateway tier
- Service tier-এর মধ্যে 20টি microservice
- Cache tier
- DB tier

অর্থাৎ:

- Multi-tier = infrastructure topology
- Microservices = service decomposition model

#### Comparison table

```
+----------------------+--------------------------------------+--------------------------------------+
| Aspect               | Multi-Tier                           | Microservices                        |
+----------------------+--------------------------------------+--------------------------------------+
| Primary focus        | deployment & network                 | domain & service boundaries          |
| Main question        | কোন tier কোথায় run করবে?            | কোন service কী responsibility নেবে?  |
| Isolation type       | infra/runtime isolation              | logical/business isolation           |
| Scaling              | tier-level                           | service-level                        |
| Example              | web tier, app tier, data tier        | user service, order service, payment |
+----------------------+--------------------------------------+--------------------------------------+
```

#### খুব common misunderstanding

একটি monolith-ও multi-tier হতে পারে।

উদাহরণ:

- Nginx web tier
- Laravel monolith app tier
- MySQL data tier
- Redis cache tier

এটি **multi-tier**, কিন্তু **microservices** নয়।

আবার microservices architecture-ও single-tier-ish হতে পারে development environment-এ,
যেখানে সব service একই VM-এ চলছে।

#### Bangladesh example

**Daraz**-এর order, catalog, payment, recommendation microservices থাকতে পারে,
কিন্তু তারা সবাই একটি broader multi-tier infrastructure-এর ভিতরে থাকে:

- Edge tier
- Gateway tier
- Service tier
- Cache tier
- Data tier

---

## 🌍 Real-world examples

### ৬. Real-World Bangladesh Examples

এই section-এ আমরা Bangladesh context-এ multi-tier architecture কেমন দেখায় তা দেখি।

---

### Daraz Architecture (CDN → API Gateway → Microservices → DB)

Daraz-এর মতো large e-commerce platform-এ traffic pattern mixed:

- browse traffic অনেক বেশি
- checkout traffic latency-sensitive
- seller/admin traffic আলাদা
- campaign traffic spike-driven

#### Possible tier layout

```
Customer Browser / Mobile App
           │
           ▼
+---------------------------+
| CDN / Edge Cache          |
| product image, JS, CSS    |
+---------------------------+
           │
           ▼
+---------------------------+
| API Gateway / WAF         |
| auth, rate limit, routing |
+---------------------------+
           │
           ▼
+---------------------------+
| Service Tier              |
| catalog | cart | order    |
| payment | search | user   |
+---------------------------+
      │           │
      │           ├──────────────► Redis Cache Tier
      │
      ├──────────────────────────► MySQL / Aurora Tier
      │
      └──────────────────────────► Queue / Worker Tier
```

#### কেন multi-tier এখানে fit করে?

- static content cacheable
- APIs independently scalable
- payment path stricter security চায়
- search workload catalog write workload-এর মতো নয়
- queue-driven post-order tasks async করা যায়

#### Daraz campaign day note

11.11, 12.12, Eid campaign-এ:

- CDN cache hit ratio enormous value দেয়
- app tier horizontal scaling critical
- DB read replicas catalog side-এ helpful
- queue tier order confirmation, notification, fraud evaluation handle করে

---

### bKash Architecture (Mobile → API → Transaction Service → Core Banking)

bKash-এর মতো mobile financial service platform-এ multi-tier architecture security ও audit কারণে অত্যন্ত গুরুত্বপূর্ণ।

#### Possible high-level flow

```
Mobile App / USSD / Partner App
             │
             ▼
+---------------------------+
| Edge/API Tier             |
| WAF, rate limit, auth     |
+---------------------------+
             │
             ▼
+---------------------------+
| Transaction Service Tier  |
| debit, credit, validation |
+---------------------------+
      │              │
      │              ├────────────► Fraud/Risk Tier
      │
      ├───────────────────────────► Ledger / DB Tier
      │
      └───────────────────────────► Core Banking / Switch Integration Tier
```

#### কেন strict tiering লাগে?

- PCI-like controls
- audit log immutability
- fraud detection isolation
- transaction and reporting workload আলাদা
- partner API traffic consumer app traffic থেকে আলাদা হতে পারে

#### Example security posture

- mobile traffic internet-facing edge-এ terminate হবে
- transaction service private subnet-এ
- ledger DB highly restricted subnet-এ
- admin/reporting আলাদা read path ব্যবহার করবে
- SMS notification async queue দিয়ে যাবে

---

### Pathao Architecture (App → Real-time API → Location Service → DB + Redis)

Pathao-এর মতো real-time mobility platform-এ request types heterogeneous:

- rider app requests
- captain/driver location updates
- matching logic
- ETA calculation
- trip history storage
- payment settlement

#### Possible tier model

```
Rider App / Captain App
          │
          ▼
+---------------------------+
| API Gateway Tier          |
+---------------------------+
          │
          ├──────────────► Real-time Location Tier
          │                 - Redis GEO / in-memory state
          │
          ├──────────────► Matching Service Tier
          │                 - nearest driver selection
          │
          ├──────────────► Trip Service Tier
          │                 - trip lifecycle
          │
          └──────────────► Payment/Settlement Tier

All services eventually use:
- Redis Cache Tier
- PostgreSQL/MySQL Data Tier
- Queue + Worker Tier
```

#### কেন multi-tier useful?

- real-time tier low latency চায়
- trip history writes durable storage চায়
- analytics/reporting online ride matching path থেকে আলাদা হওয়া উচিত
- push notifications queue-backed হতে পারে

---

### Prothom Alo / Media Portal Example

News/media portal-এর workload সাধারণত read-heavy।

#### Physical multi-tier view

```
Reader Browser
     │
     ▼
CDN Tier
     │
     ▼
Web Tier (Nginx cache)
     │
     ▼
CMS / Article API Tier
     │
     ├────────► Redis Cache
     └────────► MySQL
```

#### Benefit

- breaking news spike cache absorb করতে পারে
- admin CMS tier public reader tier থেকে আলাদা রাখা যায়
- image/CDN tier massively scalable

---

### Chaldal / Grocery Platform Example

Grocery system-এ inventory freshness গুরুত্বপূর্ণ।

Possible tier split:

- web/mobile tier
- catalog tier
- inventory tier
- order tier
- picking/warehouse tier
- payment tier
- DB/cache tier

এখানে warehouse operator tablet traffic এবং consumer browse traffic একরকম নয়।

---

### Foodpanda Bangladesh Example

Food delivery flow:

- customer app
- restaurant panel
- rider app
- dispatch engine
- ETA service
- notification service
- settlement service

Multi-tier সুবিধা:

- dispatch real-time traffic isolate করা যায়
- restaurant menu read path cache-friendly
- settlement and payout tier separated from hot order path

---

### GP / Telecom Self-Care Example

Telecom self-care portal বা recharge app-এ:

- internet-facing portal tier
- auth tier
- subscriber info service tier
- billing integration tier
- CRM/reporting tier
- DB tier

এখানে legacy BSS/OSS integration-এর কারণে extra integration tier খুব common।

---

## 🗺️ ASCII Diagrams

এই section-এ physical separation বোঝার জন্য diagram-heavy summary দেয়া হলো।

### Diagram 1: Single-tier vs 3-tier vs N-tier

```
Single-Tier
===========

┌────────────────────────────────────────────┐
│ Browser/UI + Logic + DB সব একসাথে           │
│ One machine / One deployment                │
└────────────────────────────────────────────┘


3-Tier
======

┌────────────┐    ┌────────────┐    ┌────────────┐
│ Client     │ -> │ App Server │ -> │ Database   │
└────────────┘    └────────────┘    └────────────┘


N-Tier
======

┌────────┐ -> ┌───────┐ -> ┌─────────┐ -> ┌──────────┐ -> ┌─────────┐
│ Client │    │ CDN   │    │ Gateway │    │ Services │    │ Cache   │
└────────┘    └───────┘    └─────────┘    └──────────┘    └─────────┘
                                                   │
                                                   ├────────> Queue
                                                   │
                                                   └────────> Database
```

### Diagram 2: 4-tier e-commerce path

```
Customer Browser
      │ HTTPS
      ▼
+-------------------------+
| Web Tier                |
| CDN + WAF + Nginx       |
+-----------+-------------+
            │ internal HTTP
            ▼
+-------------------------+
| Application Tier        |
| Product API             |
| Cart API                |
| Checkout API            |
+-----------+-------------+
            │
     ┌──────┴─────────────┐
     ▼                    ▼
+----------+         +-----------+
| Redis    |         | MySQL     |
| Cache    |         | Primary   |
+----------+         +-----------+
```

### Diagram 3: Security zone separation

```
                 [ Internet Zone ]
                        │
                        ▼
               +------------------+
               | CDN / WAF / LB   |
               +------------------+
                        │
              --- allow 443 only ---
                        │
                        ▼
                 [ DMZ / Web Zone ]
               +------------------+
               | Nginx / Reverse  |
               | Proxy            |
               +------------------+
                        │
             --- allow 8080 only ---
                        │
                        ▼
               [ Private App Zone ]
               +------------------+
               | API / Services   |
               +------------------+
                        │
          --- allow 3306,6379 internal only ---
                        │
                        ▼
                [ Restricted Data ]
          +----------------+   +----------------+
          | DB             |   | Redis / MQ     |
          +----------------+   +----------------+
```

### Diagram 4: Cross-tier latency accumulation

```
Client          30ms
  │
  ▼
CDN/WAF         10ms
  │
  ▼
ALB              3ms
  │
  ▼
App Tier        40ms
  │
  ├── Redis      2ms
  │
  └── MySQL     18ms

Total ≈ 103ms + internet variability
```

### Diagram 5: Independent scaling

```
                  Peak Traffic Event
                  ==================

Web Tier Scale Out             App Tier Scale Out          DB Tier Scale Up
------------------             ------------------          ----------------

 2 nodes -> 8 nodes             4 nodes -> 16 nodes        db.r6g.large
                                                           -> db.r6g.4xlarge

+---+ +---+                     +---+ +---+ +---+          +-------------+
|W1| |W2|      =>               |A1| |A2| |A3| ...         | Primary DB  |
+---+ +---+                     +---+ +---+ +---+          | Bigger RAM  |
                                                          +-------------+
```

### Diagram 6: Read replica fan-out

```
                    +----------------+
Writes -----------> | Primary DB     |
                    +-------+--------+
                            |
               replication  |
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
+-------------+      +-------------+      +-------------+
| Read Replica|      | Read Replica|      | Read Replica|
| Dhaka       |      | Singapore   |      | Analytics   |
+-------------+      +-------------+      +-------------+
```

### Diagram 7: Queue as separate tier

```
App Tier
   │
   ├── sync request ----------> DB
   │
   └── async event -----------> MQ Tier ---------> Worker Tier
                                              │
                                              ├── Email
                                              ├── SMS
                                              └── Push Notification
```

### Diagram 8: AWS multi-tier reference

```
Users
  │
  ▼
Route53
  │
  ▼
CloudFront
  │
  ▼
AWS WAF
  │
  ▼
ALB
  │
  ├────────► ECS Service: Web/App
  │              │
  │              ├────────► ElastiCache Redis
  │              ├────────► RDS Aurora
  │              └────────► SQS
  │                                 │
  │                                 ▼
  └────────────────────────────► Worker Service
```

### Diagram 9: Kubernetes namespace separation

```
+-------------------------------------------------------------+
| Kubernetes Cluster                                          |
|                                                             |
|  namespace: edge                                            |
|    - ingress-nginx                                          |
|    - cert-manager                                           |
|                                                             |
|  namespace: app                                             |
|    - api deployment                                         |
|    - worker deployment                                      |
|                                                             |
|  namespace: data                                            |
|    - redis (managed/external preferable)                    |
|    - db proxy                                               |
|                                                             |
|  namespace: observability                                   |
|    - prometheus                                             |
|    - grafana                                                |
|    - tempo / jaeger                                         |
+-------------------------------------------------------------+
```

### Diagram 10: bKash-style secure transaction path

```
Mobile App
   │
   ▼
+----------------------+
| API Edge             |
| auth + rate limit    |
+----------+-----------+
           │
           ▼
+----------------------+
| Transaction Tier     |
| validate + idempotent|
+-----+-----------+----+
      │           │
      │           ├────────────► Fraud Tier
      │
      ├────────────────────────► Ledger DB Tier
      │
      └────────────────────────► Core Banking Adapter Tier
```

### Diagram 11: Pathao real-time split

```
Rider App / Captain App
           │
           ▼
+----------------------+
| API Gateway          |
+--+---------+---------+
   │         │
   │         ├────────────► Location Tier (Redis GEO)
   │
   ├──────────────────────► Matching Tier
   │
   ├──────────────────────► Trip Tier (DB writes)
   │
   └──────────────────────► Notification Queue Tier
```

### Diagram 12: Failure isolation example

```
If Cache Tier Fails
===================

Client -> Web -> App -> Cache (fail)
                    │
                    └──────────────► DB fallback

Impact:
- latency increases
- DB pressure increases
- but whole system may remain available

If no tier isolation:
- one failure may crash the entire process
```

---

## 🐘 PHP Code

### ১০. PHP Code Example — Complete 3-Tier Example

এখানে আমরা একটি **Laravel/PHP application tier** দেখবো যেখানে:

- Web Tier = Nginx
- App Tier = Laravel + PHP-FPM
- Data Tier = MySQL
- Optional Cache Tier = Redis
- Health endpoint আছে
- DB connection pooling/proxy consideration আছে

---

### 10.1 Nginx config (Web Tier)

```nginx
upstream laravel_app {
    least_conn;
    server app-1.internal:9000 max_fails=3 fail_timeout=10s;
    server app-2.internal:9000 max_fails=3 fail_timeout=10s;

    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    root /var/www/public;
    index index.php;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log warn;

    # Static assets web tier-এ serve হবে
    location ~* \.(css|js|jpg|jpeg|png|gif|svg|ico|woff2?)$ {
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
        try_files $uri =404;
    }

    # Health check for load balancer
    location = /nginx-health {
        access_log off;
        return 200 "ok\n";
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass laravel_app;
        fastcgi_read_timeout 30s;
        fastcgi_connect_timeout 5s;
        fastcgi_send_timeout 30s;
        fastcgi_keep_conn on;
    }
}
```

#### Nginx config explanation

- `upstream laravel_app` app tier-এর multiple PHP-FPM node-এর দিকে route করে
- `least_conn` burst traffic-এ balanced distribution দেয়
- `keepalive 64` upstream connection reuse করে
- `nginx-health` endpoint L7 health check-এর জন্য useful
- static assets web tier-এ serve হওয়ায় app tier-এর pressure কমে

---

### 10.2 Laravel application tier structure

```text
app/
├── Http/
│   └── Controllers/
│       └── OrderController.php
├── Services/
│   └── OrderService.php
├── Repositories/
│   └── OrderRepository.php
└── Support/
    └── HealthCheck.php
```

এখানে code-level layering আছে,
কিন্তু deployment perspective-এ এগুলো সব **App Tier**-এর অংশ।

---

### 10.3 Laravel route and health endpoints

```php
<?php

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Route;

Route::get('/health/live', function () {
    return response()->json([
        'status' => 'alive',
        'tier' => 'app',
        'service' => 'checkout-api',
        'timestamp' => now()->toIso8601String(),
    ]);
});

Route::get('/health/ready', function () {
    try {
        DB::select('SELECT 1');
        Redis::connection()->ping();

        return response()->json([
            'status' => 'ready',
            'database' => 'ok',
            'redis' => 'ok',
            'timestamp' => now()->toIso8601String(),
        ]);
    } catch (Throwable $e) {
        return response()->json([
            'status' => 'not-ready',
            'error' => $e->getMessage(),
        ], 503);
    }
});
```

---

### 10.4 Laravel OrderController

```php
<?php

namespace App\Http\Controllers;

use App\Services\OrderService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function __construct(private OrderService $orderService)
    {
    }

    public function store(Request $request): JsonResponse
    {
        $payload = $request->validate([
            'user_id' => ['required', 'integer'],
            'product_id' => ['required', 'integer'],
            'quantity' => ['required', 'integer', 'min:1'],
            'payment_method' => ['required', 'string'],
        ]);

        $order = $this->orderService->placeOrder($payload);

        return response()->json([
            'message' => 'অর্ডার সফলভাবে তৈরি হয়েছে',
            'data' => $order,
        ], 201);
    }
}
```

---

### 10.5 Laravel OrderService

```php
<?php

namespace App\Services;

use App\Repositories\OrderRepository;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;
use RuntimeException;

class OrderService
{
    public function __construct(private OrderRepository $orders)
    {
    }

    public function placeOrder(array $payload): array
    {
        $cacheKey = 'product:stock:' . $payload['product_id'];
        $cachedStock = Redis::get($cacheKey);

        if ($cachedStock !== null && (int) $cachedStock < $payload['quantity']) {
            throw new RuntimeException('পর্যাপ্ত স্টক নেই');
        }

        return DB::transaction(function () use ($payload, $cacheKey) {
            $product = $this->orders->lockProduct($payload['product_id']);

            if ($product->stock < $payload['quantity']) {
                throw new RuntimeException('পর্যাপ্ত স্টক নেই');
            }

            $total = $product->price * $payload['quantity'];

            $order = $this->orders->createOrder([
                'user_id' => $payload['user_id'],
                'product_id' => $payload['product_id'],
                'quantity' => $payload['quantity'],
                'payment_method' => $payload['payment_method'],
                'total_amount' => $total,
                'status' => 'pending',
            ]);

            $this->orders->decrementStock($payload['product_id'], $payload['quantity']);

            Redis::del($cacheKey);

            return [
                'order_id' => $order->id,
                'status' => $order->status,
                'total_amount' => $total,
            ];
        });
    }
}
```

---

### 10.6 Laravel OrderRepository

```php
<?php

namespace App\Repositories;

use App\Models\Order;
use App\Models\Product;

class OrderRepository
{
    public function lockProduct(int $productId): Product
    {
        return Product::query()
            ->whereKey($productId)
            ->lockForUpdate()
            ->firstOrFail();
    }

    public function createOrder(array $data): Order
    {
        return Order::query()->create($data);
    }

    public function decrementStock(int $productId, int $quantity): int
    {
        return Product::query()
            ->whereKey($productId)
            ->decrement('stock', $quantity);
    }
}
```

---

### 10.7 Laravel database configuration

```php
<?php

return [
    'default' => env('DB_CONNECTION', 'mysql'),

    'connections' => [
        'mysql' => [
            'driver' => 'mysql',
            'host' => env('DB_HOST', 'db-primary.internal'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'daraz_clone'),
            'username' => env('DB_USERNAME', 'app_user'),
            'password' => env('DB_PASSWORD', ''),
            'unix_socket' => env('DB_SOCKET', ''),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'strict' => true,
            'engine' => null,
            'options' => extension_loaded('pdo_mysql') ? array_filter([
                PDO::ATTR_PERSISTENT => true,
                PDO::ATTR_TIMEOUT => 3,
            ]) : [],
        ],
    ],
];
```

#### Note on PHP connection pooling

PHP traditional request-per-process model-এ “true shared pool” app-runtime level-এ সীমিত।
Production-এ ভালো pattern হলো:

- **ProxySQL** MySQL-এর সামনে বসানো
- PHP-FPM process reuse
- persistent PDO connection
- connection limit carefully tune করা

#### ProxySQL example

```ini
mysql_servers=
(
  { address = "db-primary.internal", port = 3306, hostgroup = 10, max_connections = 200 },
  { address = "db-replica-1.internal", port = 3306, hostgroup = 20, max_connections = 300 },
  { address = "db-replica-2.internal", port = 3306, hostgroup = 20, max_connections = 300 }
)

mysql_users=
(
  { username = "app_user", password = "secret", default_hostgroup = 10, active = 1 }
)
```

এতে app tier সরাসরি DB-এর বদলে DB proxy tier-এর সাথে connect করতে পারে।

---

### 10.8 MySQL data tier example

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_products_stock (stock)
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_orders_user_id (user_id),
    INDEX idx_orders_status (status)
);
```

---

### 10.9 PHP-FPM pool config

```ini
[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 80
pm.start_servers = 10
pm.min_spare_servers = 10
pm.max_spare_servers = 20
pm.max_requests = 1000
request_terminate_timeout = 30s
```

এটি app tier capacity planning-এর অংশ।

---

### ১২. Deployment Patterns (Docker Compose, Kubernetes, Terraform)

#### 12.1 Docker Compose multi-tier setup

```yaml
version: '3.9'

services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./infra/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./src:/var/www:ro
    depends_on:
      - app
    networks:
      - edge
      - app_net

  app:
    build:
      context: ./src
      dockerfile: Dockerfile
    environment:
      APP_ENV: production
      DB_HOST: proxysql
      DB_PORT: 6033
      DB_DATABASE: appdb
      DB_USERNAME: app_user
      DB_PASSWORD: secret
      REDIS_HOST: redis
    depends_on:
      - proxysql
      - redis
    networks:
      - app_net
      - data_net

  proxysql:
    image: proxysql/proxysql:2.6.2
    networks:
      - data_net

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - data_net

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: appdb
      MYSQL_USER: app_user
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: rootsecret
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - data_net

networks:
  edge:
  app_net:
  data_net:

volumes:
  mysql_data:
```

#### Compose interpretation

- `edge` network = web-facing tier
- `app_net` = web ↔ app communication
- `data_net` = app ↔ db/cache communication

এটি local বা small deployment-এ multi-tier thinking শেখার জন্য ভালো।

---

#### 12.2 Kubernetes multi-tier with namespaces

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: edge
---
apiVersion: v1
kind: Namespace
metadata:
  name: app
---
apiVersion: v1
kind: Namespace
metadata:
  name: observability
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-edge
  namespace: edge
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-edge
  template:
    metadata:
      labels:
        app: nginx-edge
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  namespace: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel-app
  template:
    metadata:
      labels:
        app: laravel-app
    spec:
      containers:
        - name: php
          image: registry.example.com/laravel-app:latest
          ports:
            - containerPort: 9000
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

#### Optional network policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-edge-to-app
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: laravel-app
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
```

এখানে namespace এবং network policy tier boundary হিসেবে কাজ করছে।

---

#### 12.3 Terraform infrastructure as code example

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-southeast-1a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "app_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "ap-southeast-1a"
}

resource "aws_subnet" "data_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.21.0/24"
  availability_zone = "ap-southeast-1a"
}

resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
}

resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

এই IaC snippet দেখায় কীভাবে tier contract code দিয়ে enforce করা যায়।

---

## 🟨 JS Code

### ১১. Node.js Code Example — Express.js + Redis + DB Pool

এখানে একটি **application tier** example দেখানো হচ্ছে যেখানে:

- Express.js app tier
- Redis caching tier interaction
- MySQL connection pool
- health endpoints
- readiness vs liveness separation

---

### 11.1 Express bootstrap

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const Redis = require('ioredis');

const app = express();
app.use(express.json());

const dbPool = mysql.createPool({
  host: process.env.DB_HOST || 'db-primary.internal',
  port: Number(process.env.DB_PORT || 3306),
  user: process.env.DB_USER || 'app_user',
  password: process.env.DB_PASSWORD || 'secret',
  database: process.env.DB_NAME || 'ride_app',
  waitForConnections: true,
  connectionLimit: 30,
  queueLimit: 100,
  enableKeepAlive: true,
  keepAliveInitialDelay: 10000,
});

const redis = new Redis({
  host: process.env.REDIS_HOST || 'redis.internal',
  port: Number(process.env.REDIS_PORT || 6379),
  lazyConnect: true,
  maxRetriesPerRequest: 2,
});

(async () => {
  await redis.connect();
})();
```

---

### 11.2 Health endpoints

```javascript
app.get('/health/live', (_req, res) => {
  res.json({
    status: 'alive',
    service: 'pathao-like-trip-api',
    tier: 'application',
    time: new Date().toISOString(),
  });
});

app.get('/health/ready', async (_req, res) => {
  try {
    await dbPool.query('SELECT 1');
    await redis.ping();

    res.json({
      status: 'ready',
      db: 'ok',
      redis: 'ok',
      time: new Date().toISOString(),
    });
  } catch (error) {
    res.status(503).json({
      status: 'not-ready',
      error: error.message,
    });
  }
});
```

---

### 11.3 Product details with Redis cache tier

```javascript
app.get('/products/:id', async (req, res) => {
  const { id } = req.params;
  const cacheKey = `product:${id}`;

  try {
    const cached = await redis.get(cacheKey);

    if (cached) {
      return res.json({
        source: 'cache-tier',
        data: JSON.parse(cached),
      });
    }

    const [rows] = await dbPool.query(
      'SELECT id, name, price, stock FROM products WHERE id = ?',
      [id]
    );

    if (rows.length === 0) {
      return res.status(404).json({ message: 'পণ্য পাওয়া যায়নি' });
    }

    await redis.set(cacheKey, JSON.stringify(rows[0]), 'EX', 60);

    return res.json({
      source: 'data-tier',
      data: rows[0],
    });
  } catch (error) {
    return res.status(500).json({ message: error.message });
  }
});
```

---

### 11.4 Order placement with pooled DB connection

```javascript
app.post('/orders', async (req, res) => {
  const connection = await dbPool.getConnection();

  try {
    const { userId, productId, quantity } = req.body;

    await connection.beginTransaction();

    const [products] = await connection.query(
      'SELECT id, price, stock FROM products WHERE id = ? FOR UPDATE',
      [productId]
    );

    if (products.length === 0) {
      await connection.rollback();
      return res.status(404).json({ message: 'পণ্য পাওয়া যায়নি' });
    }

    const product = products[0];

    if (product.stock < quantity) {
      await connection.rollback();
      return res.status(400).json({ message: 'পর্যাপ্ত স্টক নেই' });
    }

    const totalAmount = Number(product.price) * Number(quantity);

    const [result] = await connection.query(
      `INSERT INTO orders (user_id, product_id, quantity, total_amount, status)
       VALUES (?, ?, ?, ?, ?)`,
      [userId, productId, quantity, totalAmount, 'pending']
    );

    await connection.query(
      'UPDATE products SET stock = stock - ? WHERE id = ?',
      [quantity, productId]
    );

    await connection.commit();
    await redis.del(`product:${productId}`);

    return res.status(201).json({
      orderId: result.insertId,
      totalAmount,
      status: 'pending',
    });
  } catch (error) {
    await connection.rollback();
    return res.status(500).json({ message: error.message });
  } finally {
    connection.release();
  }
});
```

---

### 11.5 App startup

```javascript
const port = Number(process.env.PORT || 8080);

app.listen(port, () => {
  console.log(`Application tier listening on ${port}`);
});
```

---

### 11.6 Nginx reverse proxy for Node.js app tier

```nginx
upstream node_app {
    server node-app-1.internal:8080;
    server node-app-2.internal:8080;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name api.pathao-clone.local;

    location = /edge-health {
        return 200 'ok';
    }

    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://node_app;
    }
}
```

এখানে web tier এবং app tier পৃথক।

---

### ১৩. Monitoring Multi-Tier Systems

Multi-tier system observable না হলে operationally dangerous।

#### 13.1 Per-tier metrics

##### Edge/Web tier metrics

- requests per second
- TLS handshake errors
- 4xx / 5xx rate
- active connections
- cache hit ratio
- WAF blocked requests

##### App tier metrics

- p50 / p95 / p99 latency
- error rate
- CPU / memory
- event loop lag (Node.js)
- PHP-FPM worker saturation
- DB pool exhaustion

##### Cache tier metrics

- hit ratio
- eviction rate
- memory usage
- connected clients
- command latency

##### Data tier metrics

- CPU
- disk IOPS
- slow queries
- replication lag
- lock wait time
- connection count

##### Queue/worker tier metrics

- queue depth
- oldest message age
- consumer lag
- job success/failure rate
- retry count

#### 13.2 Cross-tier tracing

একটি request যদি edge → app → cache → DB → queue → worker path traverse করে,
তাহলে distributed tracing অপরিহার্য।

Trace span example:

```
[trace_id=abc123]
  edge.request
    └── app.http_handler
          ├── redis.get product:101
          ├── mysql.select products
          ├── mysql.insert orders
          └── kafka.publish order.created
```

#### 13.3 Alerting examples

**Edge tier alerts**

- 5xx > 2% for 5 min
- active connections > threshold
- WAF anomalies surge

**App tier alerts**

- p95 latency > 500ms
- DB pool wait time > 100ms
- error rate > 1%

**Data tier alerts**

- replication lag > 10s
- free storage < 15%
- CPU > 80% sustained
- slow query spike

**Worker tier alerts**

- queue lag > SLA
- dead letter queue growing

#### 13.4 Node.js Prometheus metrics example

```javascript
const client = require('prom-client');

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.05, 0.1, 0.2, 0.5, 1, 2],
});

register.registerMetric(httpDuration);

app.use((req, res, next) => {
  const end = httpDuration.startTimer();

  res.on('finish', () => {
    end({
      method: req.method,
      route: req.route?.path || req.path,
      status_code: String(res.statusCode),
    });
  });

  next();
});

app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

#### 13.5 Laravel logging/tracing hint

```php
<?php

Log::withContext([
    'trace_id' => request()->header('X-Trace-Id'),
    'tier' => 'app',
    'service' => 'checkout-api',
]);
```

#### 13.6 Observability stack example

```
+------------------+      +------------------+      +------------------+
| Prometheus       | ---> | Grafana          | ---> | Alertmanager     |
+------------------+      +------------------+      +------------------+
         ^
         │ scrape
         │
  +------+------+------+-------------------------------+
  | edge       app      redis       mysql       workers |
  +-----------------------------------------------------+

+------------------+      +------------------+
| OpenTelemetry    | ---> | Jaeger / Tempo   |
+------------------+      +------------------+
```

#### 13.7 Dashboard design principle

একটি good dashboard-এ tier অনুযায়ী breakdown থাকা উচিত:

- edge dashboard
- app dashboard
- cache dashboard
- DB dashboard
- worker dashboard
- end-to-end user latency dashboard

#### 13.8 Bangladesh production intuition

- **bKash**-এ transaction success rate ও ledger lag critical metric
- **Pathao**-তে dispatch latency ও location freshness critical metric
- **Daraz**-এ cart-to-checkout conversion drop campaign-time alert-worthy
- **Prothom Alo**-তে CDN hit ratio sudden drop cost/latency spike তৈরি করতে পারে

---

## 🎯 When to use

### ১৪. When to use Multi-Tier

Multi-tier architecture সব project-এর জন্য দরকার নেই।
কিন্তু কিছু পরিস্থিতিতে এটি অত্যন্ত যৌক্তিক।

#### Strong signals that you need multi-tier

- internet-facing production system
- security-sensitive data আছে
- traffic profile mixed
- scaling per component আলাদা হওয়া দরকার
- multiple teams কাজ করছে
- compliance / audit boundary দরকার
- caching, queueing, DB isolation useful

#### Good candidates

- payment systems
- e-commerce platform
- ride sharing / food delivery
- media/news portal
- ERP / enterprise admin system
- telecom self-care portal
- healthcare or education portals with sensitive data

#### When not to over-engineer

ছোট CRUD app,
internal tool,
MVP,
low traffic product,
single team prototype — এদের জন্য full-blown N-tier overkill হতে পারে।

#### Decision matrix

```
+------------------------------------------+------------------------------+
| Scenario                                 | Recommended Approach         |
+------------------------------------------+------------------------------+
| Small internal admin tool                | 2-tier / simple 3-tier       |
| Standard business web app                | 3-tier                       |
| Content-heavy portal                     | 4-tier with CDN/cache        |
| Large e-commerce                         | N-tier                       |
| Regulated payment system                 | strict N-tier                |
| Real-time dispatch platform              | N-tier + queue + cache       |
| MVP startup                              | simple 3-tier first          |
+------------------------------------------+------------------------------+
```

#### Practical trade-off questions

নিচের প্রশ্নগুলো “হ্যাঁ” হলে multi-tier justified হতে পারে:

- web traffic এবং business transaction traffic কি একই না?
- database কি public exposure থেকে সম্পূর্ণ আড়ালে রাখা দরকার?
- caching কি separate capacity plan চায়?
- background job volume কি online API path-এর থেকে আলাদা?
- load balancer / WAF / rate limiting কি dedicated edge tier warrant করে?
- observability কি per-tier দরকার?

#### Suggested maturity path

```
Stage 1: Simple 3-tier
Client -> App -> DB

Stage 2: Add web/cache separation
Client -> CDN/Nginx -> App -> DB/Redis

Stage 3: Add queue/worker
Client -> Edge -> App -> DB + MQ -> Worker

Stage 4: Add gateway + service split
Client -> CDN -> API GW -> Services -> Cache/DB/MQ
```

#### Architecture selection advice

**If you are building:**

- **Prothom Alo-style content site** → CDN + web tier + app tier + DB/cache tier
- **Daraz-style commerce** → CDN + gateway + service tier + cache + DB + queue/worker
- **bKash-style financial app** → strict security zones + transaction tier + ledger tier + integration tier + audit tier
- **Pathao-style mobility app** → gateway + real-time tier + trip tier + cache + queue + DB

#### Common anti-patterns

##### Anti-pattern 1: DB directly accessible from internet

খুব ঝুঁকিপূর্ণ।

##### Anti-pattern 2: Session stored only in local memory

horizontal scaling কঠিন হয়।

##### Anti-pattern 3: One load balancer but no readiness health checks

bad deploy হলে unhealthy node traffic পেতে থাকে।

##### Anti-pattern 4: App tier directly doing heavy async work

user-facing latency বেড়ে যায়।

##### Anti-pattern 5: Too many tiers too early

operational complexity business value-কে ছাড়িয়ে যায়।

#### Short decision checklist

- [ ] public users আছে
- [ ] database private রাখতে হবে
- [ ] independent scaling দরকার
- [ ] queue/cache দরকার
- [ ] multiple runtime concerns আছে
- [ ] security zone দরকার
- [ ] monitoring maturity আছে

যদি ৪ বা তার বেশি box checked হয়,
multi-tier worth considering।

---

## ✅ Key Takeaways

### ১৫. Key Takeaways

- **Multi-Tier Architecture** মূলত **physical deployment and infrastructure separation** নিয়ে, code layering নিয়ে নয়।
- **Layered Architecture** এবং **Multi-Tier Architecture** related হলেও same concept নয়।
- **2-tier** legacy/simple systems-এ দেখা যায়, **3-tier** classic web standard, **4-tier** delivery vs logic separation দেয়, আর **N-tier** enterprise scale-এর জন্য।
- Tier separation-এর প্রধান কারণ: **security**, **independent scaling**, **fault isolation**, **team ownership**, **compliance**।
- ভালো multi-tier system-এ network segmentation, firewall rule, health check, connection pooling, caching, queueing — সব deliberate ভাবে design করা হয়।
- **Web tier** এবং **app tier** সহজে horizontal scale হয়; **data tier** সাধারণত careful vertical scaling + read replicas + cache চায়।
- **L4 vs L7 load balancer** বেছে নিতে হবে traffic type, smart routing need, এবং latency budget অনুযায়ী।
- **Sticky session** যতটা সম্ভব avoid করে stateless app + Redis/session store/JWT ব্যবহার করা ভালো।
- **Multi-tier** এবং **microservices** এক জিনিস নয়; microservices একটি multi-tier infrastructure-এর ভিতরে run করতে পারে।
- **AWS/GCP cloud** multi-tier design সহজ করেছে, কিন্তু subnet, SG, AZ, observability, failure mode design এখনও critical।
- **Bangladesh examples**: Daraz campaign scaling, bKash secure transaction path, Pathao real-time location tier, Prothom Alo read-heavy CDN path — সব multi-tier thinking-এর excellent উদাহরণ।
- Monitoring ছাড়া multi-tier system blind হয়ে যায়; per-tier metrics + tracing + alerting অপরিহার্য।
- শুরুতেই over-engineer করা উচিত না; **need-driven tiering** সবচেয়ে practical approach।

---

## শেষ কথা

যদি এক লাইনে বলি:

> **Layered Architecture বলে code কীভাবে organize হবে, আর Multi-Tier Architecture বলে system কোথায় deploy হবে, কোন boundary পার হবে, কিভাবে scale হবে, এবং কিভাবে survive করবে।**

Production system design শেখার সময় এই দুইটি concept আলাদা করে বুঝতে পারা খুব গুরুত্বপূর্ণ।

বিশেষ করে Bangladesh-এর high-growth digital products — **bKash, Daraz, Pathao, Prothom Alo, Chaldal, Foodpanda, GP** — এদের বাস্তব production infrastructure বুঝতে চাইলে multi-tier deployment perspective জানা বাধ্যতামূলক।
