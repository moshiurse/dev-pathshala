# 🕸️ গ্রাফ ডাটাবেস (Graph Database)

> গ্রাফ ডাটাবেস ডেটার মধ্যে সম্পর্ক (relationships) কে প্রথম-শ্রেণির নাগরিক হিসেবে বিবেচনা করে।
> সোশ্যাল নেটওয়ার্ক, রেকমেন্ডেশন ইঞ্জিন, ফ্রড ডিটেকশন — যেখানে সম্পর্ক গুরুত্বপূর্ণ, সেখানে Graph DB অপরাজেয়।

---

## 📖 সূচিপত্র

- [গ্রাফ ডাটাবেস কী?](#-গ্রাফ-ডাটাবেস-কী)
- [RDBMS vs Graph DB](#-rdbms-vs-graph-db)
- [Nodes, Edges, Properties](#-nodes-edges-properties-মডেল)
- [প্রধান Graph DB সমূহ](#-প্রধান-graph-db-সমূহ)
- [Cypher Query Language](#-cypher-query-language)
- [Use Cases](#-use-cases)
- [কোড উদাহরণ](#-কোড-উদাহরণ)
- [কখন ব্যবহার করবেন](#-কখন-ব্যবহার-করবেন--করবেন-না)

---

## 📌 গ্রাফ ডাটাবেস কী?

গ্রাফ ডাটাবেস ডেটাকে **নোড** (সত্তা) এবং **এজ** (সম্পর্ক) হিসেবে সংরক্ষণ করে। এটি গাণিতিক গ্রাফ থিওরির উপর ভিত্তি করে তৈরি।

```
সোশ্যাল নেটওয়ার্ক (বাংলাদেশী ডেভেলপার):
────────────────────────────────────────

    ┌───────────┐  FOLLOWS   ┌───────────┐
    │  করিম     │──────────►│  রহিম     │
    │ (Dhaka)   │           │ (Sylhet)  │
    │ PHP Dev   │           │ Node Dev  │
    └─────┬─────┘           └─────┬─────┘
          │                       │
          │ WORKS_AT              │ WORKS_AT
          │                       │
    ┌─────▼─────┐           ┌─────▼─────┐
    │  bKash    │           │  Pathao   │
    │ (Company) │           │ (Company) │
    └─────┬─────┘           └───────────┘
          │
          │ USES
          │
    ┌─────▼─────┐
    │  Laravel  │
    │(Framework)│
    └───────────┘

  নোড: করিম, রহিম, bKash, Pathao, Laravel
  এজ: FOLLOWS, WORKS_AT, USES
  প্রোপার্টি: city, role, etc.
```

### কেন গ্রাফ ডাটাবেস?

```
প্রশ্ন: "করিম এর বন্ধুদের বন্ধুরা কোথায় কাজ করে?"

RDBMS তে (SQL):
─────────────────
SELECT DISTINCT c.name
FROM users a
JOIN friendships f1 ON a.id = f1.user_id
JOIN friendships f2 ON f1.friend_id = f2.user_id
JOIN employment e ON f2.friend_id = e.user_id
JOIN companies c ON e.company_id = c.id
WHERE a.name = 'করিম';

❌ ৩টি JOIN! ডেটা বাড়লে অত্যন্ত ধীর

Graph DB তে (Cypher):
──────────────────────
MATCH (a:User {name: 'করিম'})-[:FRIEND]->()-[:FRIEND]->()-[:WORKS_AT]->(c:Company)
RETURN DISTINCT c.name;

✅ সহজ, পড়তে সুবিধা, এবং বিশাল ডেটাসেটেও দ্রুত!
```

---

## 📊 RDBMS vs Graph DB

```
┌────────────────────┬──────────────────┬──────────────────┐
│     বৈশিষ্ট্য       │     RDBMS        │    Graph DB      │
├────────────────────┼──────────────────┼──────────────────┤
│ ডেটা মডেল          │ Tables, Rows     │ Nodes, Edges     │
├────────────────────┼──────────────────┼──────────────────┤
│ সম্পর্ক             │ Foreign Keys     │ First-class      │
│                    │ + JOINs          │ Edges             │
├────────────────────┼──────────────────┼──────────────────┤
│ JOIN Performance   │ ❌ ধীর (N+1,     │ ✅ দ্রুত          │
│                    │ বড় ডেটায়)       │ (Index-free adj.) │
├────────────────────┼──────────────────┼──────────────────┤
│ Traversal          │ ❌ কষ্টকর         │ ✅ সহজ           │
│ (Path finding)     │ (Recursive CTE)  │ (Built-in)       │
├────────────────────┼──────────────────┼──────────────────┤
│ Schema             │ Rigid (Fixed)    │ Flexible          │
├────────────────────┼──────────────────┼──────────────────┤
│ Aggregation        │ ✅ দ্রুত          │ ⚠️ মাঝারি         │
├────────────────────┼──────────────────┼──────────────────┤
│ ACID               │ ✅ Full          │ ✅ (Neo4j)       │
├────────────────────┼──────────────────┼──────────────────┤
│ Scale              │ Vertical         │ Horizontal       │
│                    │                  │ (Sharding)       │
├────────────────────┼──────────────────┼──────────────────┤
│ Best For           │ Structured,      │ Connected,       │
│                    │ Transactional    │ Relationship-    │
│                    │                  │ heavy data       │
└────────────────────┴──────────────────┴──────────────────┘

JOIN Depth vs Performance:
──────────────────────────
         Response Time
  10s   │
        │         RDBMS
   5s   │        ╱
        │       ╱
   1s   │      ╱
        │     ╱
 100ms  │    ╱     Graph DB
        │───╱─────────────────────
  10ms  │  ╱
        │ ╱
   1ms  │╱
        └───────────────────────── JOIN Depth
         1    2    3    4    5    6

  RDBMS: JOIN depth বাড়লে exponentially ধীর হয়
  Graph DB: প্রায় constant time! ✅
```

---

## 📖 Nodes, Edges, Properties মডেল

```
┌─────────────────────────────────────────────────────────┐
│              Property Graph Model                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Node (নোড/Vertex):                                     │
│  ┌─────────────────────────┐                            │
│  │ Label: User             │                            │
│  │ Properties:             │                            │
│  │   name: "করিম"           │                            │
│  │   phone: "01712345678"  │                            │
│  │   city: "ঢাকা"          │                            │
│  │   role: "developer"     │                            │
│  └─────────────────────────┘                            │
│                                                         │
│  Edge (এজ/Relationship):                                │
│  ┌─────────────────────────┐                            │
│  │ Type: PURCHASED         │                            │
│  │ Properties:             │                            │
│  │   date: "2024-01-15"    │                            │
│  │   amount: 15999         │                            │
│  │   payment: "bKash"      │                            │
│  │ Direction: →            │                            │
│  └─────────────────────────┘                            │
│                                                         │
│  সম্পূর্ণ গ্রাফ:                                          │
│                                                         │
│  (করিম)──[PURCHASED {amount:15999}]──►(iPhone Case)     │
│    │                                       │             │
│    │ REVIEWED                              │ CATEGORY    │
│    ▼                                       ▼             │
│  (iPhone 15)◄──[SIMILAR_TO]──(Samsung S24) │             │
│    │                                       │             │
│    │ SOLD_BY                   ┌───────────┘             │
│    ▼                           ▼                         │
│  (Daraz)                 (Accessories)                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Daraz ই-কমার্স গ্রাফ মডেল

```
┌──────────────────────────────────────────────────────────────┐
│                  Daraz Graph Model                            │
│                                                              │
│  ┌────────┐  PURCHASED   ┌──────────┐  BELONGS_TO  ┌──────┐│
│  │ User   │─────────────►│ Product  │─────────────►│ Cat. ││
│  │ করিম   │              │ iPhone 15│              │Phones││
│  └───┬────┘              └────┬─────┘              └──────┘│
│      │                        │                             │
│      │ VIEWED                 │ SOLD_BY                     │
│      ▼                        ▼                             │
│  ┌────────┐              ┌──────────┐                       │
│  │Product │              │  Seller  │                       │
│  │Samsung │              │ Official │                       │
│  │Galaxy  │              │  Store   │                       │
│  └────────┘              └──────────┘                       │
│      │                                                      │
│      │ ALSO_VIEWED_BY                                       │
│      ▼                                                      │
│  ┌────────┐  FRIEND_OF   ┌────────┐                        │
│  │ User   │◄────────────│ User   │                        │
│  │ রহিম   │             │ জামাল   │                        │
│  └────────┘             └────────┘                        │
│                                                              │
│  রেকমেন্ডেশন: "করিম iPhone 15 কিনেছে,                       │
│  রহিমও iPhone দেখেছে — রহিমকে iPhone 15 suggest করো!"       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 প্রধান Graph DB সমূহ

```
┌──────────────┬───────────────┬───────────────┬───────────────┐
│              │    Neo4j      │ Amazon Neptune│  ArangoDB     │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Model        │ Property Graph│ Property Graph│ Multi-model   │
│              │               │ + RDF         │ (Graph+Doc+KV)│
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Query Lang.  │ Cypher        │ Gremlin /     │ AQL           │
│              │               │ SPARQL        │               │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Hosting      │ Self/Cloud    │ AWS Managed   │ Self/Cloud    │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ ACID         │ ✅            │ ✅            │ ✅            │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Scale        │ ✅ Cluster    │ ✅ Auto       │ ✅ Cluster    │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ Best For     │ General graph │ AWS ecosystem │ Mixed         │
│              │ use cases     │               │ workloads     │
├──────────────┼───────────────┼───────────────┼───────────────┤
│ সুবিধা       │ সবচেয়ে জনপ্রিয়│ Fully managed│ মাল্টি-মডেল   │
│              │ বড় community │ serverless   │               │
└──────────────┴───────────────┴───────────────┴───────────────┘
```

---

## 📖 Cypher Query Language

Cypher হলো Neo4j এর ডিক্লেয়ারেটিভ কোয়েরি ল্যাঙ্গুয়েজ। এটি ASCII art pattern matching ব্যবহার করে।

### মৌলিক Syntax

```cypher
// =============================================
// Cypher Basics
// =============================================

// --- Node তৈরি ---
CREATE (u:User {name: 'করিম', phone: '01712345678', city: 'ঢাকা'})
CREATE (p:Product {name: 'iPhone 15', price: 159999, category: 'phones'})
CREATE (c:Company {name: 'bKash', industry: 'fintech'})

// --- Relationship তৈরি ---
CREATE (u)-[:PURCHASED {date: '2024-01-15', amount: 159999}]->(p)
CREATE (u)-[:WORKS_AT {since: '2022-03-01', role: 'Senior Dev'}]->(c)

// --- Node খুঁজে বের করা ---
MATCH (u:User {name: 'করিম'})
RETURN u

// --- সম্পর্ক সহ Query ---
// করিম কী কী কিনেছে?
MATCH (u:User {name: 'করিম'})-[:PURCHASED]->(p:Product)
RETURN p.name, p.price

// bKash এ কারা কাজ করে?
MATCH (u:User)-[:WORKS_AT]->(c:Company {name: 'bKash'})
RETURN u.name, u.role

// --- Path Finding ---
// করিম থেকে রহিম পর্যন্ত সংক্ষিপ্ততম পথ
MATCH path = shortestPath(
    (a:User {name: 'করিম'})-[*..6]-(b:User {name: 'রহিম'})
)
RETURN path
```

### অ্যাডভান্সড Queries

```cypher
// =============================================
// রেকমেন্ডেশন ইঞ্জিন (Daraz-style)
// =============================================

// ১. "যারা এটা কিনেছে, তারা এটাও কিনেছে" (Collaborative Filtering)
MATCH (u:User)-[:PURCHASED]->(p:Product {name: 'iPhone 15'}),
      (u)-[:PURCHASED]->(other:Product)
WHERE other.name <> 'iPhone 15'
RETURN other.name, COUNT(*) AS purchase_count
ORDER BY purchase_count DESC
LIMIT 5

// ২. "আপনার বন্ধুরা এটা পছন্দ করেছে"
MATCH (me:User {name: 'করিম'})-[:FRIEND]->(friend:User),
      (friend)-[:REVIEWED {rating: 5}]->(p:Product)
WHERE NOT (me)-[:PURCHASED]->(p)
RETURN p.name, COUNT(friend) AS friend_recommendations
ORDER BY friend_recommendations DESC
LIMIT 10

// ৩. ফ্রড ডিটেকশন (bKash-style)
// একই ফোন নম্বর থেকে ভিন্ন ভিন্ন অ্যাকাউন্টে লেনদেন
MATCH (a1:Account)-[:USES_PHONE]->(phone:Phone)<-[:USES_PHONE]-(a2:Account),
      (a1)-[:TRANSFERRED_TO]->(a3:Account),
      (a2)-[:TRANSFERRED_TO]->(a3)
WHERE a1 <> a2
RETURN a1, a2, a3, phone
// সন্দেহজনক! একই ফোন থেকে দুই অ্যাকাউন্ট একই জায়গায় টাকা পাঠাচ্ছে

// ৪. সোশ্যাল নেটওয়ার্ক — "People You May Know"
MATCH (me:User {name: 'করিম'})-[:FRIEND]->(:User)-[:FRIEND]->(suggestion:User)
WHERE NOT (me)-[:FRIEND]->(suggestion)
AND me <> suggestion
RETURN suggestion.name, COUNT(*) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10

// ৫. Impact Analysis — "এই সার্ভিস ডাউন হলে কী প্রভাবিত হবে?"
MATCH (s:Service {name: 'payment-service'})<-[:DEPENDS_ON*1..3]-(dependent:Service)
RETURN dependent.name, length(shortestPath((dependent)-[:DEPENDS_ON*]->(s))) AS depth
ORDER BY depth
```

---

## 🎯 Use Cases

```
┌──────────────────────┬──────────────────────────────────────┐
│     Use Case         │     বাংলাদেশী উদাহরণ                  │
├──────────────────────┼──────────────────────────────────────┤
│ সোশ্যাল নেটওয়ার্ক     │ "People You May Know" ফিচার          │
│                      │ বন্ধুদের বন্ধু, mutual connections     │
├──────────────────────┼──────────────────────────────────────┤
│ রেকমেন্ডেশন ইঞ্জিন    │ Daraz: "এটাও দেখুন", "যারা কিনেছে   │
│                      │ তারা এটাও কিনেছে"                    │
├──────────────────────┼──────────────────────────────────────┤
│ ফ্রড ডিটেকশন          │ bKash/Nagad: সন্দেহজনক লেনদেন       │
│                      │ pattern চিহ্নিত, money laundering    │
│                      │ ring সনাক্ত                          │
├──────────────────────┼──────────────────────────────────────┤
│ Knowledge Graph      │ Wikipedia বাংলা: entities ও তাদের    │
│                      │ সম্পর্ক                               │
├──────────────────────┼──────────────────────────────────────┤
│ Network Topology     │ Grameenphone: cell tower network,    │
│                      │ routing optimization                 │
├──────────────────────┼──────────────────────────────────────┤
│ Identity & Access    │ কে কোন resource এ access পাবে,       │
│ Management           │ role hierarchy, permission graph     │
├──────────────────────┼──────────────────────────────────────┤
│ Supply Chain         │ Daraz: সেলার → ওয়্যারহাউজ → ডেলিভারি  │
│                      │ হাব → কাস্টমার tracking               │
├──────────────────────┼──────────────────────────────────────┤
│ Dependency Mapping   │ মাইক্রোসার্ভিস dependency graph,       │
│                      │ impact analysis                      │
└──────────────────────┴──────────────────────────────────────┘
```

### ফ্রড ডিটেকশন ভিজুয়ালাইজেশন

```
সন্দেহজনক Money Laundering Ring:

    ┌──────────┐   ৫০,০০০৳    ┌──────────┐
    │ Account A│─────────────►│ Account D│
    │ (করিম)   │              │ (ভুয়া)   │
    └────┬─────┘              └────┬─────┘
         │                        │
    ২০,০০০৳                  ৪৫,০০০৳
         │                        │
    ┌────▼─────┐              ┌────▼─────┐
    │ Account B│   ৩০,০০০৳    │ Account E│
    │ (রহিম)   │─────────────►│ (ভুয়া)   │
    └────┬─────┘              └────┬─────┘
         │                        │
    ১৫,০০০৳                  ৭০,০০০৳ (Cash Out!)
         │                        │
    ┌────▼─────┐              ┌────▼─────┐
    │ Account C│──────────────│  Agent   │
    │ (জামাল)  │              │ (ভুয়া)   │
    └──────────┘              └──────────┘

    🔍 Graph DB এই circular pattern সহজে detect করতে পারে:
    MATCH (a)-[:TRANSFERRED*3..6]->(a) RETURN a
    (যে account থেকে টাকা ঘুরে ফিরে আসছে!)
```

---

## 💻 কোড উদাহরণ

### PHP (neo4j-php-client)

```php
<?php
// composer require laudis/neo4j-php-client

use Laudis\Neo4j\ClientBuilder;
use Laudis\Neo4j\Contracts\TransactionInterface;

// =============================================
// Neo4j PHP Client
// বাস্তব উদাহরণ: Daraz রেকমেন্ডেশন সিস্টেম
// =============================================

$client = ClientBuilder::create()
    ->withDriver('bolt', 'bolt://neo4j:password@localhost:7687')
    ->build();

// --- ডেটা তৈরি ---
function seedData(TransactionInterface $tsx): void
{
    // ইউজার তৈরি
    $tsx->run('
        CREATE (u1:User {id: "U001", name: "করিম", city: "ঢাকা", phone: "01712345678"})
        CREATE (u2:User {id: "U002", name: "রহিম", city: "চট্টগ্রাম", phone: "01898765432"})
        CREATE (u3:User {id: "U003", name: "জামাল", city: "ঢাকা", phone: "01612345678"})
    ');
    
    // প্রোডাক্ট তৈরি
    $tsx->run('
        CREATE (p1:Product {id: "P001", name: "iPhone 15", price: 159999, category: "phones"})
        CREATE (p2:Product {id: "P002", name: "AirPods Pro", price: 35999, category: "accessories"})
        CREATE (p3:Product {id: "P003", name: "iPhone Case", price: 999, category: "accessories"})
        CREATE (p4:Product {id: "P004", name: "Samsung S24", price: 139999, category: "phones"})
    ');
    
    // সম্পর্ক তৈরি
    $tsx->run('
        MATCH (u1:User {id: "U001"}), (u2:User {id: "U002"}), (u3:User {id: "U003"})
        MATCH (p1:Product {id: "P001"}), (p2:Product {id: "P002"}), (p3:Product {id: "P003"})
        
        CREATE (u1)-[:PURCHASED {date: "2024-01-15", amount: 159999}]->(p1)
        CREATE (u1)-[:PURCHASED {date: "2024-01-16", amount: 35999}]->(p2)
        CREATE (u2)-[:PURCHASED {date: "2024-01-17", amount: 159999}]->(p1)
        CREATE (u2)-[:PURCHASED {date: "2024-01-18", amount: 999}]->(p3)
        CREATE (u1)-[:FRIEND]->(u2)
        CREATE (u2)-[:FRIEND]->(u3)
        CREATE (u1)-[:VIEWED]->(p3)
    ');
}

$client->writeTransaction(function (TransactionInterface $tsx) {
    seedData($tsx);
    echo "✅ ডেটা তৈরি সফল!\n";
});

// --- রেকমেন্ডেশন Query ---

// ১. "যারা iPhone 15 কিনেছে তারা এটাও কিনেছে"
$result = $client->run('
    MATCH (u:User)-[:PURCHASED]->(p:Product {name: "iPhone 15"}),
          (u)-[:PURCHASED]->(recommended:Product)
    WHERE recommended.name <> "iPhone 15"
    RETURN recommended.name AS product, 
           recommended.price AS price,
           COUNT(*) AS buyers
    ORDER BY buyers DESC
    LIMIT 5
');

echo "\n📦 iPhone 15 কেনার সাথে জনপ্রিয়:\n";
foreach ($result as $record) {
    echo "  - {$record->get('product')}: ৳{$record->get('price')} ({$record->get('buyers')} জন)\n";
}

// ২. "করিম এর জন্য ব্যক্তিগত রেকমেন্ডেশন"
$personalRecs = $client->run('
    MATCH (me:User {name: "করিম"})-[:FRIEND]->(friend:User)-[:PURCHASED]->(p:Product)
    WHERE NOT (me)-[:PURCHASED]->(p)
    RETURN p.name AS product, p.price AS price, 
           COLLECT(friend.name) AS recommended_by
    LIMIT 5
');

echo "\n🎯 করিম এর জন্য সাজেশন:\n";
foreach ($personalRecs as $record) {
    $friends = implode(', ', $record->get('recommended_by'));
    echo "  - {$record->get('product')}: ৳{$record->get('price')} (সাজেশন: $friends)\n";
}

// ৩. Shortest Path
$path = $client->run('
    MATCH path = shortestPath(
        (a:User {name: "করিম"})-[*..6]-(b:User {name: "জামাল"})
    )
    RETURN [node IN nodes(path) | 
        CASE WHEN node:User THEN node.name ELSE node.name END
    ] AS path_nodes,
    length(path) AS hops
');

foreach ($path as $record) {
    $nodes = implode(' → ', $record->get('path_nodes'));
    echo "\n🔗 করিম → জামাল পথ: $nodes ({$record->get('hops')} hops)\n";
}

// ৪. ফ্রড ডিটেকশন
$fraud = $client->run('
    MATCH (a:Account)-[:TRANSFERRED_TO*3..6]->(a)
    RETURN a.id AS suspicious_account, a.owner AS owner
    LIMIT 10
');

echo "\n⚠️ সন্দেহজনক অ্যাকাউন্ট (circular transfers):\n";
foreach ($fraud as $record) {
    echo "  🚨 {$record->get('suspicious_account')}: {$record->get('owner')}\n";
}
```

### JavaScript (neo4j-driver)

```javascript
// npm install neo4j-driver

// =============================================
// Neo4j JavaScript Driver
// বাস্তব উদাহরণ: সোশ্যাল নেটওয়ার্ক + রেকমেন্ডেশন
// =============================================

const neo4j = require('neo4j-driver');

const driver = neo4j.driver(
    'bolt://localhost:7687',
    neo4j.auth.basic('neo4j', 'password')
);

// --- ডেটা সিড করা ---
async function seedSocialNetwork() {
    const session = driver.session();
    
    try {
        await session.executeWrite(async (tx) => {
            // ইউজার ও কোম্পানি তৈরি
            await tx.run(`
                CREATE (u1:User {id: 'U001', name: 'করিম', city: 'ঢাকা', skills: ['PHP', 'Laravel']})
                CREATE (u2:User {id: 'U002', name: 'রহিম', city: 'চট্টগ্রাম', skills: ['Node.js', 'React']})
                CREATE (u3:User {id: 'U003', name: 'জামাল', city: 'ঢাকা', skills: ['Python', 'Django']})
                CREATE (u4:User {id: 'U004', name: 'কামাল', city: 'সিলেট', skills: ['PHP', 'React']})
                
                CREATE (c1:Company {name: 'bKash', industry: 'FinTech', city: 'ঢাকা'})
                CREATE (c2:Company {name: 'Pathao', industry: 'Ride-sharing', city: 'ঢাকা'})
                CREATE (c3:Company {name: 'Daraz', industry: 'E-commerce', city: 'ঢাকা'})
                
                CREATE (u1)-[:WORKS_AT {role: 'Senior Dev', since: 2022}]->(c1)
                CREATE (u2)-[:WORKS_AT {role: 'Full Stack Dev', since: 2023}]->(c2)
                CREATE (u3)-[:WORKS_AT {role: 'Data Engineer', since: 2021}]->(c3)
                
                CREATE (u1)-[:FRIEND]->(u2)
                CREATE (u2)-[:FRIEND]->(u3)
                CREATE (u3)-[:FRIEND]->(u4)
                CREATE (u1)-[:FRIEND]->(u4)
            `);
        });
        
        console.log('✅ সোশ্যাল নেটওয়ার্ক ডেটা তৈরি সম্পন্ন');
    } finally {
        await session.close();
    }
}

// --- "People You May Know" ---
async function peopleYouMayKnow(userName) {
    const session = driver.session();
    
    try {
        const result = await session.executeRead(async (tx) => {
            return tx.run(`
                MATCH (me:User {name: $name})-[:FRIEND]->(friend)-[:FRIEND]->(suggestion:User)
                WHERE NOT (me)-[:FRIEND]->(suggestion)
                AND me <> suggestion
                RETURN suggestion.name AS name,
                       suggestion.city AS city,
                       COUNT(friend) AS mutual_friends,
                       COLLECT(friend.name) AS through
                ORDER BY mutual_friends DESC
                LIMIT 10
            `, { name: userName });
        });
        
        console.log(`\n👥 "${userName}" হয়তো চেনেন:`);
        result.records.forEach(record => {
            const name = record.get('name');
            const city = record.get('city');
            const mutuals = record.get('mutual_friends').toNumber();
            const through = record.get('through');
            console.log(`  ${name} (${city}) - ${mutuals} mutual friend: ${through.join(', ')}`);
        });
    } finally {
        await session.close();
    }
}

// --- Skill-based রেকমেন্ডেশন ---
async function recommendBySkills(userName) {
    const session = driver.session();
    
    try {
        const result = await session.executeRead(async (tx) => {
            return tx.run(`
                MATCH (me:User {name: $name})
                WITH me, me.skills AS mySkills
                MATCH (other:User)
                WHERE other <> me
                WITH other, mySkills,
                     [s IN other.skills WHERE s IN mySkills] AS commonSkills
                WHERE SIZE(commonSkills) > 0
                RETURN other.name AS name,
                       other.city AS city,
                       commonSkills,
                       SIZE(commonSkills) AS matchCount
                ORDER BY matchCount DESC
            `, { name: userName });
        });
        
        console.log(`\n🎯 "${userName}" এর skill match:`);
        result.records.forEach(record => {
            const name = record.get('name');
            const skills = record.get('commonSkills');
            console.log(`  ${name}: ${skills.join(', ')}`);
        });
    } finally {
        await session.close();
    }
}

// --- Graph Visualization ডেটা ---
async function getGraphData() {
    const session = driver.session();
    
    try {
        const result = await session.executeRead(async (tx) => {
            return tx.run(`
                MATCH (n)-[r]->(m)
                RETURN n, r, m
                LIMIT 100
            `);
        });
        
        const nodes = new Map();
        const edges = [];
        
        result.records.forEach(record => {
            const n = record.get('n');
            const m = record.get('m');
            const r = record.get('r');
            
            nodes.set(n.identity.toString(), {
                id: n.identity.toString(),
                label: n.labels[0],
                properties: n.properties
            });
            
            nodes.set(m.identity.toString(), {
                id: m.identity.toString(),
                label: m.labels[0],
                properties: m.properties
            });
            
            edges.push({
                source: n.identity.toString(),
                target: m.identity.toString(),
                type: r.type,
                properties: r.properties
            });
        });
        
        return {
            nodes: Array.from(nodes.values()),
            edges
        };
    } finally {
        await session.close();
    }
}

// চালানো
(async () => {
    await seedSocialNetwork();
    await peopleYouMayKnow('করিম');
    await recommendBySkills('করিম');
    
    const graphData = await getGraphData();
    console.log('\n📊 Graph Data:', JSON.stringify(graphData, null, 2));
    
    await driver.close();
})();
```

---

## ❌ Bad / ✅ Good Patterns

```
❌ ভুল: সব ডেটা গ্রাফে রাখা
──────────────────────────────
// ইউজার প্রোফাইল, অর্ডার হিস্ট্রি, ইনভেন্টরি
// সব Neo4j তে রাখা → RDBMS এর কাজ Graph DB তে!

✅ সঠিক: Polyglot Persistence
   - RDBMS: ইউজার, অর্ডার, ইনভেন্টরি (structured data)
   - Graph DB: সম্পর্ক, রেকমেন্ডেশন, ফ্রড ডিটেকশন
   - Redis: ক্যাশ, সেশন

──────────────────────────────────────────

❌ ভুল: Aggregation এর জন্য Graph DB ব্যবহার
   // "গত মাসে মোট কত অর্ডার হয়েছে?" → RDBMS/TSDB

✅ সঠিক: Graph DB শুধু relationship-heavy query তে
   // "করিমের বন্ধুরা কী কিনেছে?" → Graph DB ✅

──────────────────────────────────────────

❌ ভুল: Super Node তৈরি করা
   // একটি Node এ মিলিয়ন relationship
   // (Dhaka:City)-[:LIVES_IN]-(1M Users) → ধীর!

✅ সঠিক: Category/Bucket দিয়ে ভাগ করুন
   // (Dhaka:City)-[:HAS_AREA]->(Gulshan:Area)-[:LIVES_IN]-(Users)
```

---

## 🎯 কখন ব্যবহার করবেন / করবেন না

### Graph DB ব্যবহার করুন:

```
✅ সোশ্যাল নেটওয়ার্ক (বন্ধু, ফলোয়ার, গ্রুপ)
✅ রেকমেন্ডেশন ইঞ্জিন ("এটাও পছন্দ হতে পারে")
✅ ফ্রড ডিটেকশন (সন্দেহজনক লেনদেন pattern)
✅ Knowledge Graph (Wikipedia, Google Knowledge)
✅ Network/IT Infrastructure mapping
✅ Access Control (Role → Permission hierarchy)
✅ Supply Chain management
✅ Dependency analysis (মাইক্রোসার্ভিস dependencies)
```

### Graph DB ব্যবহার করবেন না:

```
❌ সাধারণ CRUD operations → RDBMS
❌ টাইম-সিরিজ ডেটা → TSDB
❌ ফুল-টেক্সট সার্চ → Elasticsearch
❌ ক্যাশিং → Redis
❌ ডকুমেন্ট স্টোরেজ → MongoDB
❌ সিম্পল key-value → Redis/DynamoDB
❌ Aggregation-heavy analytics → RDBMS/ClickHouse
```

---

## 📊 সারসংক্ষেপ

```
┌───────────────────────────────────────────────────────┐
│            গ্রাফ ডাটাবেস সারসংক্ষেপ                     │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Graph DB = সম্পর্ক-কেন্দ্রিক ডাটাবেস                   │
│  Model: Nodes (সত্তা) + Edges (সম্পর্ক) + Properties   │
│                                                       │
│  Neo4j: সবচেয়ে জনপ্রিয়, Cypher query language          │
│  Amazon Neptune: AWS managed, Gremlin/SPARQL          │
│                                                       │
│  RDBMS এর চেয়ে Graph DB ভালো:                         │
│  - Deep relationship traversal                       │
│  - Pattern matching                                   │
│  - Path finding                                       │
│                                                       │
│  মূল Use Cases:                                       │
│  - সোশ্যাল নেটওয়ার্ক, রেকমেন্ডেশন                     │
│  - ফ্রড ডিটেকশন, Knowledge Graph                      │
│  - Dependency mapping, Access control                │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

> 💡 **পরবর্তী**: [ফুল-টেক্সট সার্চ](./full-text-search.md) — Elasticsearch এবং সার্চ ইঞ্জিন
