# 📐 স্কেলেবিলিটি নীতিমালা (Scalability Principles)

> **"Scalability শুধু আরও সার্ভার যোগ করা নয় — এটি একটি আর্কিটেকচারাল মানসিকতা।"**

---

## 📌 সংজ্ঞা

**Scalability** হলো একটি সিস্টেমের ক্রমবর্ধমান workload সামলানোর ক্ষমতা — হোক সেটা বেশি user, বেশি data, বেশি request, বা বেশি complexity — সিস্টেমের performance ও reliability না কমিয়ে।

### স্কেলেবিলিটি vs পারফরম্যান্স

| ধারণা | ব্যাখ্যা |
|-------|----------|
| **Performance** | নির্দিষ্ট load-এ system কতটা দ্রুত respond করে |
| **Scalability** | load বাড়ালে system কতটা ভালোভাবে মানিয়ে নেয় |

একটি system fast হতে পারে কিন্তু scalable না (single server-এ কোনো cache ছাড়া super-optimized code)।

---

## 🏗️ মূল স্কেলেবিলিটি নীতিমালা

### ১. Statelessness (স্টেটলেসনেস)

সার্ভার কোনো request-specific state ধরে রাখবে না। প্রতিটি request self-contained।

```
❌ Stateful Server:
  Server A ─── session data memory-তে
  User আবার এলে Server A তেই যেতে হবে (sticky session)
  Server A মারা গেলে session হারিয়ে যাবে

✅ Stateless Server:
  Server A, B, C ─── কোনো local state নেই
  Session data → Redis / DB
  যেকোনো server handle করতে পারে
  Server মরলেও অন্যরা চালিয়ে যেতে পারে
```

**কেন গুরুত্বপূর্ণ:**
- Horizontal scaling সহজ হয় (যেকোনো সময় server add/remove)
- Load balancing সহজ হয় (round-robin ব্যবহার করা যায়)
- Fault tolerance বাড়ে

---

### ২. Modularity (মডুলারিটি)

সিস্টেমকে স্বতন্ত্র, loosely-coupled module-এ ভাগ করা যাতে প্রতিটি আলাদাভাবে scale করা যায়।

```
┌─────────────────────────────────────────────────────────────┐
│                   Monolithic Architecture                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Auth + Products + Orders + Payments + Notifications  │    │
│  │              সব একসাথে scale হবে 💀                   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  পুরো app scale → ১০× cost, শুধু Orders busy ছিল            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Modular Architecture                        │
│                                                              │
│  ┌──────┐  ┌──────────┐  ┌────────┐  ┌─────────┐  ┌─────┐ │
│  │ Auth │  │ Products │  │ Orders │  │Payments │  │Notif│  │
│  │ ×2   │  │   ×3     │  │  ×10   │  │  ×5    │  │ ×2  │  │
│  └──────┘  └──────────┘  └────────┘  └─────────┘  └─────┘ │
│                                                              │
│  শুধু Orders ×10 scale → ৩× cost, targeted scaling ✅        │
└─────────────────────────────────────────────────────────────┘
```

#### মডুলারিটির নিয়ম:

| নিয়ম | ব্যাখ্যা |
|-------|----------|
| **Single Responsibility** | প্রতিটি module একটি নির্দিষ্ট কাজ করবে |
| **Loose Coupling** | Module-এর মধ্যে ন্যূনতম নির্ভরতা |
| **High Cohesion** | সম্পর্কিত logic একসাথে থাকবে |
| **Well-defined Interface** | Module-এ ঢোকার clear API থাকবে |
| **Independent Deployability** | আলাদাভাবে deploy ও scale করা যাবে |

---

### ৩. Horizontal Scaling (Scale Out)

```
                    Load Balancer
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Server 1 │   │ Server 2 │   │ Server 3 │
    │ (same    │   │ (same    │   │ (same    │
    │  code)   │   │  code)   │   │  code)   │
    └──────────┘   └──────────┘   └──────────┘
```

**Horizontal vs Vertical Scaling:**

| বৈশিষ্ট্য | Vertical (Scale Up) | Horizontal (Scale Out) |
|-----------|--------------------|-----------------------|
| কৌশল | বড় server | বেশি server |
| সীমা | Hardware limit | প্রায় unlimited |
| Cost curve | Exponential | Linear |
| Downtime | Server বদলাতে downtime | Zero downtime add/remove |
| Complexity | কম | বেশি (distributed systems) |

---

### ৪. Caching (ক্যাশিং)

বারবার ব্যবহৃত data দ্রুত-অ্যাক্সেসযোগ্য layer-এ রাখা।

```
Request Flow (without cache):
  User → App → Database (50ms) → Response

Request Flow (with cache):
  User → App → Cache (0.5ms) → Response (hit)
  User → App → Cache (miss) → DB (50ms) → Cache-এ store → Response
```

**Caching Layers:**

```
┌─────────────────────────────────────────────────────────────┐
│                     Caching Hierarchy                         │
│                                                              │
│  Browser Cache (client)                          ~0ms        │
│       ▼                                                      │
│  CDN Cache (edge)                                ~5ms        │
│       ▼                                                      │
│  Application Cache (Redis/Memcached)             ~1ms        │
│       ▼                                                      │
│  Database Query Cache                            ~5ms        │
│       ▼                                                      │
│  Database (disk)                                 ~50ms       │
└─────────────────────────────────────────────────────────────┘
```

---

### ৫. Asynchronous Processing

যা তাৎক্ষণিক response-এ দরকার নেই, সেটা background-এ করা।

```
❌ Synchronous (সব একসাথে):
  User → [Order] → [Payment 500ms] → [SMS 200ms] → [Email 300ms] → Response
  মোট: 1000ms, যেকোনোটা fail = সব fail

✅ Asynchronous (queue দিয়ে):
  User → [Order] → [Payment 500ms] → Response (500ms)
                         │
                    Message Queue
                    ┌────┴────┐
                    ▼         ▼
              [SMS worker] [Email worker]
              (পরে হবে)    (পরে হবে)
```

---

### ৬. Database Scaling Strategies

```
┌─────────────────────────────────────────────────────────┐
│              Database Scaling Progression                 │
│                                                          │
│  Stage 1: Read Replicas                                  │
│    Master (Write) → Replica1 (Read) → Replica2 (Read)   │
│                                                          │
│  Stage 2: Vertical Partitioning (by feature)            │
│    Users DB │ Orders DB │ Products DB │ Analytics DB     │
│                                                          │
│  Stage 3: Horizontal Sharding (by data)                 │
│    Users A-M → Shard1 │ Users N-Z → Shard2             │
│                                                          │
│  Stage 4: CQRS + Event Sourcing                         │
│    Write Model (normalized) ≠ Read Model (denormalized)  │
└─────────────────────────────────────────────────────────┘
```

---

### ৭. Load Balancing

Traffic-কে একাধিক server-এ সমানভাবে ভাগ করা।

```
                Internet Traffic
                      │
              ┌───────▼───────┐
              │  DNS (GSLB)   │ ← Geographic routing
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │    L4 / L7    │ ← Load Balancer
              │  Balancer     │
              └───────┬───────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Server 1 │  │Server 2 │  │Server 3 │
    └─────────┘  └─────────┘  └─────────┘
```

---

### ৮. Partitioning & Sharding

বড় dataset-কে ছোট ছোট টুকরোতে ভাগ করা।

| কৌশল | ব্যাখ্যা | উদাহরণ |
|------|----------|---------|
| **Range** | key range অনুযায়ী ভাগ | A-M → Shard1, N-Z → Shard2 |
| **Hash** | key-এর hash থেকে shard নির্ধারণ | hash(user_id) % 4 |
| **Geographic** | location অনুযায়ী ভাগ | BD → Shard1, IN → Shard2 |
| **Functional** | feature অনুযায়ী ভাগ | Users → DB1, Orders → DB2 |

---

### ৯. Replication

Data-র copy একাধিক node-এ রাখা — availability ও read performance বাড়ায়।

```
                ┌──────────────┐
                │    Master    │ ← All Writes
                └──────┬───────┘
                       │ replication
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Replica1 │ │ Replica2 │ │ Replica3 │  ← Reads distributed
    └──────────┘ └──────────┘ └──────────┘
```

---

### ১০. Content Delivery Network (CDN)

Static content edge server-এ cache করে user-এর কাছাকাছি serve করা।

---

## 📊 Amdahl's Law (অ্যামডালের সূত্র)

### সংজ্ঞা

Amdahl's Law বলে — একটি program-এর সর্বোচ্চ speedup সীমিত, যদি program-এর কিছু অংশ serial (parallelizable নয়) হয়।

### সূত্র

```
                         1
  Speedup(N) = ─────────────────────
                 S + (1 - S) / N

  যেখানে:
    S = program-এর serial (non-parallelizable) fraction
    N = processor/thread/instance সংখ্যা
    (1 - S) = parallelizable fraction
```

### বাস্তব উদাহরণ

ধরুন একটি web request processing pipeline:

```
Total processing time: 100ms
  - Authentication (serial):     10ms  (S = 0.1)
  - Business logic (parallel):   60ms
  - DB write (serial):           30ms  (S = 0.3)

মোট serial fraction, S = 0.4 (40%)
Parallelizable fraction = 0.6 (60%)
```

| Processors (N) | Speedup | Effective Time |
|----------------|---------|----------------|
| 1 | 1.00× | 100ms |
| 2 | 1.43× | 70ms |
| 4 | 1.67× | 60ms |
| 8 | 1.82× | 55ms |
| 16 | 1.90× | 52.6ms |
| ∞ | 2.50× | 40ms (max!) |

```
Speedup
  ▲
  │                          Theoretical max = 1/S = 2.5×
  │                    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
2.5│                  ╱
  │               ╱
2.0│            ╱
  │         ╱
1.5│      ╱
  │   ╱
1.0│──╱
  │
  └──────────────────────────────────────────────→ Processors (N)
     1    2    4    8    16   32   64   128

  S = 0.4 হলে, ∞ processor দিলেও max 2.5× speedup!
```

### Amdahl's Law-এর শিক্ষা

| শিক্ষা | ব্যাখ্যা |
|--------|----------|
| Serial bottleneck চিহ্নিত করুন | কোন অংশ parallelize করা যায় না? |
| শুধু server বাড়ালে হবে না | Serial part-ই ultimate limit |
| Serial fraction কমানো = সবচেয়ে effective optimization | |
| Diminishing returns | N বাড়ালে marginal improvement কমতে থাকে |

### সিস্টেম ডিজাইনে প্রয়োগ

```
বাস্তব serial bottleneck উদাহরণ:

1. Database lock contention:
   - Shared row-level lock = serial section
   - Optimistic locking / CQRS = serial fraction কমায়

2. Global state synchronization:
   - Distributed lock = serial
   - Event sourcing / eventual consistency = parallelism বাড়ায়

3. Sequential processing:
   - Synchronous pipeline = serial
   - Async message queue = parallel workers

4. Leader election / single writer:
   - Single master DB write = serial
   - Multi-master / partitioning = parallel writes
```

---

## 📊 Gunther's Universal Scalability Law (USL)

### সংজ্ঞা

Amdahl's Law শুধু **contention** ধরে। কিন্তু বাস্তবে distributed system-এ আরেকটি factor আছে — **coherence** (data consistency overhead)। Gunther's USL উভয়ই model করে।

### সূত্র

```
                           N
  Throughput(N) = ────────────────────────────────
                   1 + σ(N-1) + κ·N·(N-1)

  যেখানে:
    N = node/processor সংখ্যা
    σ (sigma) = contention coefficient (serial fraction, Amdahl's S)
    κ (kappa) = coherence coefficient (coordination/consistency cost)
```

### তিনটি মডেলের তুলনা

```
Throughput
  ▲
  │         Linear (ideal)
  │        ╱
  │       ╱ ╲ Amdahl (σ > 0, κ = 0)
  │      ╱    ╲──────────────── asymptote
  │     ╱       ╲
  │    ╱     USL  ╲  (σ > 0, κ > 0)
  │   ╱              ╲
  │  ╱                  ╲  ← throughput কমতে থাকে!
  │ ╱                      ╲
  │╱
  └──────────────────────────────────────→ Nodes (N)
     1    2    4    8    16   32   64

  Linear: throughput = N (perfect scaling)
  Amdahl: throughput reaches asymptote (never decreases)
  USL:    throughput can DECREASE after a peak! (retrograde)
```

### কেন Throughput কমে?

**Coherence penalty (κ):** যখন nodes বাড়ে, তাদের মধ্যে data সামঞ্জস্য রাখার cost quadratically বাড়ে।

```
N = 2 nodes: 2×1 = 2 communication paths
N = 4 nodes: 4×3 = 12 communication paths
N = 8 nodes: 8×7 = 56 communication paths
N = 16 nodes: 16×15 = 240 communication paths!

Communication overhead = O(N²) — এটাই κ term!
```

### বাস্তব উদাহরণ

**Database cluster with strong consistency:**

```
┌─────────────────────────────────────────────────────────────┐
│  Scenario: 5-node PostgreSQL cluster (synchronous replication)│
│                                                              │
│  Write operation:                                            │
│    1. Leader receives write                                  │
│    2. Leader sends to 4 followers                           │
│    3. Wait for majority (3) ack                             │
│    4. Acknowledge client                                     │
│                                                              │
│  N বাড়ালে:                                                  │
│    - σ effect: lock contention বাড়ে                         │
│    - κ effect: replication ack wait বাড়ে (N² messaging)     │
│                                                              │
│  Result: N=5 থেকে N=10 করলে write throughput কমতে পারে!    │
└─────────────────────────────────────────────────────────────┘
```

### USL-এর শিক্ষা

| শিক্ষা | Action |
|--------|--------|
| শুধু contention (σ) নয়, coherence (κ) ও বিবেচনা করুন | Distributed lock, consensus protocol-এর cost measure করুন |
| Node বাড়ালে throughput কমতে পারে | Benchmark করে optimal N খুঁজুন |
| κ কমাতে: eventual consistency, partitioning, CRDT ব্যবহার | Strong consistency সবার জন্য না |
| σ কমাতে: serial section কমানো, lock-free algorithms | Amdahl's Law-এর advice follow করুন |
| Optimal N = √((1-σ) / κ) | এর বেশি node আসলে ক্ষতিকর |

### USL Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│           Scale করার আগে নিজেকে জিজ্ঞাসা করুন              │
│                                                              │
│  1. কোন অংশ serial (σ)?                                     │
│     → Lock, single writer, sequential processing            │
│     → কমাতে: optimistic locking, partitioning, async        │
│                                                              │
│  2. কোথায় coordination cost (κ)?                            │
│     → Distributed transaction, consensus, cache invalidation │
│     → কমাতে: eventual consistency, local cache, CQRS        │
│                                                              │
│  3. Optimal point কোথায়?                                    │
│     → Load test করে throughput curve plot করুন               │
│     → Peak-এর আগে stop করুন                                 │
│                                                              │
│  4. Scale up বা scale out?                                   │
│     → κ বেশি হলে scale up ভালো (কম coordination)            │
│     → σ বেশি হলে bottleneck fix করুন                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 PHP Code Example — Measuring Scalability

```php
<?php

class ScalabilityCalculator
{
    /**
     * Amdahl's Law: Maximum speedup with N processors
     */
    public static function amdahlSpeedup(float $serialFraction, int $processors): float
    {
        if ($serialFraction < 0 || $serialFraction > 1) {
            throw new \InvalidArgumentException('Serial fraction must be between 0 and 1');
        }
        return 1 / ($serialFraction + (1 - $serialFraction) / $processors);
    }

    /**
     * Gunther's USL: Throughput with N nodes
     */
    public static function uslThroughput(int $nodes, float $sigma, float $kappa): float
    {
        return $nodes / (1 + $sigma * ($nodes - 1) + $kappa * $nodes * ($nodes - 1));
    }

    /**
     * USL optimal number of nodes
     */
    public static function uslOptimalNodes(float $sigma, float $kappa): int
    {
        if ($kappa <= 0) return PHP_INT_MAX; // no coherence cost
        return (int) ceil(sqrt((1 - $sigma) / $kappa));
    }

    /**
     * Find peak throughput with USL parameters
     */
    public static function findPeakThroughput(float $sigma, float $kappa, int $maxNodes = 100): array
    {
        $peak = ['nodes' => 1, 'throughput' => 1.0];

        for ($n = 1; $n <= $maxNodes; $n++) {
            $t = self::uslThroughput($n, $sigma, $kappa);
            if ($t > $peak['throughput']) {
                $peak = ['nodes' => $n, 'throughput' => $t];
            }
        }
        return $peak;
    }
}

// Example: bKash payment processing
$sigma = 0.05;  // 5% serial (DB leader write)
$kappa = 0.002; // coherence cost (replication ack)

echo "Optimal nodes: " . ScalabilityCalculator::uslOptimalNodes($sigma, $kappa) . "\n";

$peak = ScalabilityCalculator::findPeakThroughput($sigma, $kappa);
echo "Peak throughput at {$peak['nodes']} nodes: {$peak['throughput']}x\n";

// Amdahl's comparison
for ($n = 1; $n <= 32; $n *= 2) {
    $amdahl = ScalabilityCalculator::amdahlSpeedup($sigma, $n);
    $usl = ScalabilityCalculator::uslThroughput($n, $sigma, $kappa);
    echo "N=$n: Amdahl={$amdahl}x, USL={$usl}x\n";
}
```

---

## 💻 JavaScript Code Example

```javascript
class ScalabilityModel {
  /**
   * Amdahl's Law speedup
   */
  static amdahlSpeedup(serialFraction, processors) {
    return 1 / (serialFraction + (1 - serialFraction) / processors);
  }

  /**
   * Gunther's USL throughput
   */
  static uslThroughput(nodes, sigma, kappa) {
    return nodes / (1 + sigma * (nodes - 1) + kappa * nodes * (nodes - 1));
  }

  /**
   * Optimal number of nodes (USL)
   */
  static optimalNodes(sigma, kappa) {
    if (kappa <= 0) return Infinity;
    return Math.ceil(Math.sqrt((1 - sigma) / kappa));
  }

  /**
   * Generate scaling curve data
   */
  static scalingCurve(sigma, kappa, maxNodes = 64) {
    const data = [];
    for (let n = 1; n <= maxNodes; n++) {
      data.push({
        nodes: n,
        linear: n,
        amdahl: this.amdahlSpeedup(sigma, n),
        usl: this.uslThroughput(n, sigma, kappa),
      });
    }
    return data;
  }
}

// Example: Daraz product search cluster
const sigma = 0.03;  // 3% serial fraction
const kappa = 0.001; // coherence overhead

console.log(`Optimal nodes: ${ScalabilityModel.optimalNodes(sigma, kappa)}`);
console.log(`\nScaling curve:`);

const curve = ScalabilityModel.scalingCurve(sigma, kappa, 32);
curve.filter(d => [1, 2, 4, 8, 16, 32].includes(d.nodes)).forEach(d => {
  console.log(
    `N=${String(d.nodes).padStart(2)}: ` +
    `Linear=${d.linear.toFixed(1)}x, ` +
    `Amdahl=${d.amdahl.toFixed(2)}x, ` +
    `USL=${d.usl.toFixed(2)}x`
  );
});
```

---

## 🎯 Modularity for Scalability — গভীর আলোচনা

### কেন Modularity Scalability-র জন্য অপরিহার্য?

Monolithic system-এ সব component একসাথে scale করতে হয়। Modular system-এ:
- প্রতিটি module আলাদাভাবে scale করা যায়
- Bottleneck module চিহ্নিত ও ঠিক করা সহজ
- Team autonomy — আলাদা team আলাদা module নিয়ে কাজ করতে পারে
- Technology heterogeneity — ভিন্ন module-এ ভিন্ন tech stack

### Decomposition Strategies

```
┌─────────────────────────────────────────────────────────────┐
│              Decomposition Strategy Selection                 │
│                                                              │
│  ┌─────────────────┐                                         │
│  │ By Business     │  Users, Orders, Payments, Inventory     │
│  │ Capability      │  (DDD Bounded Context)                  │
│  └─────────────────┘                                         │
│                                                              │
│  ┌─────────────────┐                                         │
│  │ By Scalability  │  Read-heavy → separate read service     │
│  │ Requirement     │  Write-heavy → separate write service   │
│  └─────────────────┘                                         │
│                                                              │
│  ┌─────────────────┐                                         │
│  │ By Change       │  Frequently changing → separate module  │
│  │ Frequency       │  Stable → core module                   │
│  └─────────────────┘                                         │
│                                                              │
│  ┌─────────────────┐                                         │
│  │ By Data         │  Hot data → high-perf service + cache   │
│  │ Temperature     │  Cold data → cost-optimized storage     │
│  └─────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

### Module Communication Patterns

| Pattern | Latency | Coupling | Use Case |
|---------|---------|----------|----------|
| Synchronous REST/gRPC | Low | High | Real-time, simple request-response |
| Async Message Queue | Medium | Low | Background tasks, event notification |
| Event Streaming (Kafka) | Low | Very Low | Event-driven, analytics, replication |
| Shared Database | Lowest | Highest | ❌ Anti-pattern for modularity |

### Practical Scalability Pattern — CQRS with Separate Read/Write

```
┌──────────────────────────────────────────────────┐
│              CQRS for Scalability                  │
│                                                    │
│  Write Side (×2 instances):                        │
│  ┌────────────┐     ┌──────────────┐              │
│  │ Command    │────►│  Write DB    │              │
│  │ Handler    │     │ (PostgreSQL) │              │
│  └────────────┘     └──────┬───────┘              │
│                            │ CDC / Events          │
│                            ▼                       │
│                     ┌──────────────┐              │
│                     │ Event Stream │              │
│                     │   (Kafka)    │              │
│                     └──────┬───────┘              │
│                            │                       │
│  Read Side (×20 instances):                        │
│  ┌────────────┐     ┌──────────────┐              │
│  │  Query     │◄────│  Read Store  │              │
│  │  Handler   │     │(Elasticsearch│              │
│  └────────────┘     │  / Redis)    │              │
│                     └──────────────┘              │
│                                                    │
│  Read:Write ratio 100:1 → Read side ×20 scale     │
└──────────────────────────────────────────────────┘
```

---

## ✅ Scalability Checklist

```
□ Stateless application servers
□ Session data externalized (Redis/DB)
□ Horizontal scaling ready (no sticky sessions)
□ Caching at appropriate layers
□ Async processing for non-critical paths
□ Database read replicas configured
□ Sharding strategy planned
□ Load balancer in place
□ CDN for static assets
□ Monitoring & auto-scaling rules
□ Amdahl's Law: serial bottlenecks identified
□ USL: coordination costs measured
□ Modular decomposition (scale hotspots independently)
□ Connection pooling enabled
□ Rate limiting & backpressure
```

---

## 🔑 মনে রাখার বিষয়

1. **Scalability ≠ Performance** — Fast code scalable নাও হতে পারে
2. **Amdahl's Law** — Serial fraction-ই ultimate limit; শুধু server বাড়িয়ে কাজ হয় না
3. **Gunther's USL** — Node বাড়ালে coordination cost-ও বাড়ে; অতিরিক্ত node ক্ষতিকর হতে পারে
4. **Modularity** — Independent scaling-এর পূর্বশর্ত
5. **Statelessness** — Horizontal scaling-এর পূর্বশর্ত
6. **Measure first** — Premature optimization না করে bottleneck খুঁজুন, তারপর targeted scaling করুন
