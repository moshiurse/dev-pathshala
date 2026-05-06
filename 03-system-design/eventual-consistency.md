# 🔄 Eventual Consistency & Consistency Patterns

## 📋 সূচিপত্র
- [সংজ্ঞা ও ধারণা](#সংজ্ঞা-ও-ধারণা)
- [Consistency Models](#consistency-models)
- [বাস্তব জীবনের উদাহরণ](#বাস্তব-জীবনের-উদাহরণ)
- [ASCII Diagram](#ascii-diagram)
- [Conflict Resolution Strategies](#conflict-resolution-strategies)
- [PHP কোড উদাহরণ](#php-কোড-উদাহরণ)
- [JavaScript কোড উদাহরণ](#javascript-কোড-উদাহরণ)
- [Tunable Consistency](#tunable-consistency)
- [কখন ব্যবহার করবেন / করবেন না](#কখন-ব্যবহার-করবেন--করবেন-না)

---

## 🎯 সংজ্ঞা ও ধারণা

**Eventual Consistency** হলো একটি consistency model যেখানে সিস্টেমে কোনো নতুন update না আসলে, সব replica গুলো **শেষ পর্যন্ত** (eventually) একই state-এ পৌঁছাবে। এটি CAP theorem-এর availability এবং partition tolerance বেছে নেওয়ার ফলাফল।

### 🔑 মূল ধারণা:

| Consistency Model | বর্ণনা | Latency | Availability |
|---|---|---|---|
| **Strong (Linearizability)** | সব read সর্বশেষ write দেখবে | 🔴 High | 🔴 Low |
| **Sequential** | সব node একই order দেখবে | 🟡 Medium | 🟡 Medium |
| **Causal** | causally related operations ordered | 🟢 Low-Medium | 🟢 High |
| **Eventual** | শেষ পর্যন্ত সব node একই হবে | 🟢 Very Low | 🟢 Very High |

### 📚 Consistency Guarantees:

1. **Read-Your-Writes**: আপনি যা লিখেছেন তা অবশ্যই পড়তে পারবেন
2. **Monotonic Reads**: একবার নতুন value পড়লে, পুরনো value আর দেখবেন না
3. **Session Consistency**: একটি session-এর মধ্যে consistent view
4. **Consistent Prefix**: Write-এর order maintain হবে read-এ

---

## 🌍 বাস্তব জীবনের উদাহরণ

### 💰 bKash Balance Scenario:

কল্পনা করুন রহিম ভাই bKash থেকে ১০০০ টাকা পাঠালেন করিম ভাইকে:

```
সময় T0: রহিম ভাই-এর balance = ৫০০০ টাকা (সব server-এ)
সময় T1: রহিম ভাই ১০০০ টাকা পাঠালেন
সময় T2: Server-1 → balance = ৪০০০ টাকা ✅
          Server-2 → balance = ৫০০০ টাকা (এখনো update হয়নি) ❌
          Server-3 → balance = ৫০০০ টাকা (এখনো update হয়নি) ❌
সময় T3: Server-2 → balance = ৪০০০ টাকা ✅ (replicated)
সময় T4: Server-3 → balance = ৪০০০ টাকা ✅ (replicated)
```

**সমস্যা**: T2 তে রহিম ভাই যদি মোবাইল app refresh করেন এবং request Server-2 তে যায়, তাহলে তিনি ৫০০০ টাকা দেখবেন — যা ভুল!

### 🛵 Pathao Ride Example:

```
Driver location update:
- Driver ঢাকায় গুলশান-২ তে আছেন
- Phone-এ GPS update পাঠালো Server-A তে
- Rider-এর phone Server-B থেকে পড়ছে
- কিছুক্ষণ (200ms) পুরনো location দেখাবে
- Eventually সব server sync হবে
```

### 📰 Prothom Alo Article Like Count:

```
একটি breaking news article-এ like দিলেন ১০,০০০ জন:
- Dhaka server: 10,000 likes ✅
- Chittagong server: 9,847 likes (propagation delay)
- Sylhet server: 9,623 likes (propagation delay)
→ কিছুক্ষণ পর সব server-এ 10,000 দেখাবে
```

---

## 📊 ASCII Diagram

### Strong vs Eventual Consistency:

```
┌─────────────────────────────────────────────────────────────┐
│              STRONG CONSISTENCY                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client ──Write(X=5)──► Primary ──sync──► Replica-1         │
│                              │                               │
│                              ├──────sync──► Replica-2        │
│                              │                               │
│                              └──────sync──► Replica-3        │
│                                                              │
│  ⏱️ Response time: SLOW (all replicas must acknowledge)      │
│  ✅ Guarantee: Next read from ANY node returns X=5           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              EVENTUAL CONSISTENCY                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client ──Write(X=5)──► Node-1 ──async──► Node-2            │
│       │                    │                                 │
│       │                    └──────async──► Node-3            │
│       │                                                      │
│       └── Response: OK! (immediately)                        │
│                                                              │
│  ⏱️ Response time: FAST (write to one node only)             │
│  ⚠️ Guarantee: Eventually ALL nodes will have X=5            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Consistency Spectrum:

```
    Strong ◄────────────────────────────────► Eventual
    (Linearizable)                             (BASE)

    ┌──────┬──────────┬──────────┬───────────┐
    │Linear│Sequential│  Causal  │ Eventual  │
    │izable│Consistency│Consistency│Consistency│
    ├──────┼──────────┼──────────┼───────────┤
    │ACID  │          │          │  BASE     │
    │Bank  │  Social  │  Chat    │  DNS      │
    │Trans │  Media   │  Apps    │  CDN      │
    │action│  Feed    │  Email   │  Likes    │
    └──────┴──────────┴──────────┴───────────┘
         ▲                              ▲
    High Latency                   Low Latency
    Low Availability              High Availability
```

### Read-Your-Writes Consistency:

```
    ┌─────────────────────────────────────────────┐
    │         Read-Your-Writes Pattern             │
    ├─────────────────────────────────────────────┤
    │                                              │
    │  User ──Write──► Node-A                      │
    │    │                │                        │
    │    │                ▼ (async replication)     │
    │    │              Node-B                      │
    │    │              Node-C                      │
    │    │                                         │
    │    └──Read──► SAME Node-A (sticky session)   │
    │         ✅ Sees own write                     │
    │                                              │
    │  বিকল্প: Read from any node where            │
    │  timestamp >= last_write_timestamp           │
    │                                              │
    └─────────────────────────────────────────────┘
```

---

## 🔧 Conflict Resolution Strategies

### 1. Last-Write-Wins (LWW):

```
Node-A: X = "Dhaka"   (timestamp: 100)
Node-B: X = "Chittagong" (timestamp: 102)

Resolution: X = "Chittagong" (highest timestamp wins)
⚠️ সমস্যা: Data loss হতে পারে!
```

### 2. Vector Clocks:

```
Initial:  X = "" , VC = {A:0, B:0, C:0}

Node-A writes "Dhaka":    VC = {A:1, B:0, C:0}
Node-B writes "Comilla":  VC = {A:0, B:1, C:0}

Conflict detected! (neither dominates)
→ Application-level resolution needed
```

### 3. CRDTs (Conflict-free Replicated Data Types):

```
G-Counter (শুধু বাড়ে):
  Node-A: count = 5
  Node-B: count = 3
  Node-C: count = 7
  Total = max merge per node = 5 + 3 + 7 = 15

OR-Set (Add/Remove operations):
  Node-A: add("bKash"), add("Nagad")
  Node-B: remove("bKash"), add("Rocket")
  Merge: {"Nagad", "Rocket"} (add wins over remove in OR-Set)
```

---

## 💻 PHP কোড উদাহরণ

```php
<?php

/**
 * Eventual Consistency Pattern - bKash Balance Service
 * একটি distributed balance service যা eventual consistency follow করে
 */

class EventualConsistencyBalanceService
{
    private array $replicas = [];
    private array $vectorClocks = [];
    private string $nodeId;
    
    public function __construct(string $nodeId, array $replicaNodes)
    {
        $this->nodeId = $nodeId;
        $this->replicas = $replicaNodes;
        $this->vectorClocks[$nodeId] = 0;
    }
    
    /**
     * Write with vector clock tracking
     * bKash-এর মতো service-এ balance update
     */
    public function writeBalance(string $userId, float $amount): array
    {
        // Vector clock increment
        $this->vectorClocks[$this->nodeId]++;
        
        $record = [
            'user_id' => $userId,
            'balance' => $amount,
            'vector_clock' => $this->vectorClocks,
            'timestamp' => microtime(true),
            'node_id' => $this->nodeId,
        ];
        
        // Local write (fast response)
        $this->writeLocal($record);
        
        // Async replication to other nodes
        $this->replicateAsync($record);
        
        return [
            'status' => 'accepted',
            'consistency' => 'eventual',
            'written_to' => $this->nodeId,
        ];
    }
    
    /**
     * Read with consistency level
     */
    public function readBalance(string $userId, string $consistencyLevel = 'eventual'): array
    {
        switch ($consistencyLevel) {
            case 'strong':
                return $this->readQuorum($userId);
            case 'read_your_writes':
                return $this->readFromWriteNode($userId);
            case 'eventual':
            default:
                return $this->readLocal($userId);
        }
    }
    
    /**
     * Quorum Read - majority nodes থেকে পড়া
     * W + R > N (Write nodes + Read nodes > Total nodes)
     */
    private function readQuorum(string $userId): array
    {
        $responses = [];
        $totalNodes = count($this->replicas) + 1;
        $quorumSize = (int) floor($totalNodes / 2) + 1;
        
        // Read from local
        $responses[] = $this->readLocal($userId);
        
        // Read from remote replicas until quorum
        foreach ($this->replicas as $replica) {
            if (count($responses) >= $quorumSize) break;
            $responses[] = $this->readFromReplica($replica, $userId);
        }
        
        // Return the most recent value (highest vector clock)
        return $this->resolveConflicts($responses);
    }
    
    /**
     * Conflict Resolution using Vector Clocks
     */
    private function resolveConflicts(array $responses): array
    {
        $winner = $responses[0];
        
        foreach ($responses as $response) {
            if ($this->vectorClockDominates($response['vector_clock'], $winner['vector_clock'])) {
                $winner = $response;
            }
        }
        
        return $winner;
    }
    
    /**
     * Vector Clock comparison
     * VC-A dominates VC-B if all entries in A >= B and at least one is >
     */
    private function vectorClockDominates(array $vcA, array $vcB): bool
    {
        $dominated = false;
        foreach ($vcA as $node => $clock) {
            $otherClock = $vcB[$node] ?? 0;
            if ($clock < $otherClock) return false;
            if ($clock > $otherClock) $dominated = true;
        }
        return $dominated;
    }
    
    /**
     * Anti-Entropy Protocol - Background sync
     * পর্যায়ক্রমে nodes-এর মধ্যে data sync করা
     */
    public function antiEntropySync(): void
    {
        foreach ($this->replicas as $replica) {
            $remoteData = $this->getMerkleTreeDigest($replica);
            $localData = $this->getLocalMerkleTreeDigest();
            
            $differences = $this->findDifferences($localData, $remoteData);
            
            foreach ($differences as $key) {
                $this->syncKey($key, $replica);
            }
        }
    }
    
    /**
     * Read Repair - পড়ার সময় inconsistency ঠিক করা
     */
    private function readRepair(string $userId, array $responses): void
    {
        $latestValue = $this->resolveConflicts($responses);
        
        foreach ($responses as $response) {
            if ($response['balance'] !== $latestValue['balance']) {
                $this->repairNode($response['node_id'], $userId, $latestValue);
            }
        }
    }
    
    // Helper methods (simplified)
    private function writeLocal(array $record): void { /* Redis/DB write */ }
    private function replicateAsync(array $record): void { /* Queue-based async */ }
    private function readLocal(string $userId): array { return []; }
    private function readFromWriteNode(string $userId): array { return []; }
    private function readFromReplica(string $replica, string $userId): array { return []; }
    private function getMerkleTreeDigest(string $replica): array { return []; }
    private function getLocalMerkleTreeDigest(): array { return []; }
    private function findDifferences(array $local, array $remote): array { return []; }
    private function syncKey(string $key, string $replica): void { }
    private function repairNode(string $nodeId, string $userId, array $value): void { }
}

/**
 * Session Consistency Manager
 * User session-এর মধ্যে consistency guarantee
 */
class SessionConsistencyManager
{
    private string $sessionId;
    private array $lastWriteTimestamps = [];
    
    public function __construct(string $sessionId)
    {
        $this->sessionId = $sessionId;
    }
    
    /**
     * Read-Your-Writes guarantee implementation
     */
    public function readWithConsistency(string $key, callable $readFn): mixed
    {
        $lastWrite = $this->lastWriteTimestamps[$key] ?? 0;
        
        $result = $readFn($key);
        
        // যদি read-এর timestamp আমাদের last write-এর আগে হয়,
        // তাহলে primary থেকে পড়ো
        if ($result['timestamp'] < $lastWrite) {
            $result = $this->readFromPrimary($key);
        }
        
        return $result;
    }
    
    /**
     * Monotonic Reads guarantee
     * একবার নতুন value পড়লে পুরনো value আর দেখাবে না
     */
    public function monotonicRead(string $key, callable $readFn): mixed
    {
        $result = $readFn($key);
        $lastSeen = $this->lastWriteTimestamps[$key] ?? 0;
        
        if ($result['timestamp'] < $lastSeen) {
            // পুরনো data পেয়েছি, আবার চেষ্টা করি
            $result = $this->readFromPrimary($key);
        }
        
        $this->lastWriteTimestamps[$key] = $result['timestamp'];
        return $result;
    }
    
    public function recordWrite(string $key, float $timestamp): void
    {
        $this->lastWriteTimestamps[$key] = $timestamp;
    }
    
    private function readFromPrimary(string $key): array { return []; }
}

// ব্যবহার উদাহরণ:
$service = new EventualConsistencyBalanceService('node-dhaka-1', ['node-ctg-1', 'node-syl-1']);

// bKash-এ টাকা পাঠানো
$result = $service->writeBalance('user-rahim-01', 4000.00);
echo "Write Status: {$result['status']}\n"; // accepted (fast!)

// Balance পড়া - different consistency levels
$balance = $service->readBalance('user-rahim-01', 'eventual');      // Fastest, may be stale
$balance = $service->readBalance('user-rahim-01', 'read_your_writes'); // Sees own writes
$balance = $service->readBalance('user-rahim-01', 'strong');         // Slowest, always latest
```

---

## 🟨 JavaScript কোড উদাহরণ

```javascript
/**
 * Eventual Consistency Pattern - Distributed Counter (CRDT)
 * Prothom Alo article views/likes counting
 */

class GCounter {
    constructor(nodeId, nodes) {
        this.nodeId = nodeId;
        this.counts = {};
        nodes.forEach(node => this.counts[node] = 0);
    }

    // শুধুমাত্র local node-এ increment
    increment(amount = 1) {
        this.counts[this.nodeId] += amount;
    }

    // Total value (সব node-এর যোগফল)
    value() {
        return Object.values(this.counts).reduce((sum, c) => sum + c, 0);
    }

    // দুটি counter merge (conflict-free!)
    merge(otherCounter) {
        for (const [node, count] of Object.entries(otherCounter.counts)) {
            this.counts[node] = Math.max(this.counts[node] || 0, count);
        }
    }
}

/**
 * Vector Clock implementation
 * Distributed system-এ causality tracking
 */
class VectorClock {
    constructor(nodeId) {
        this.nodeId = nodeId;
        this.clock = {};
    }

    increment() {
        this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
        return { ...this.clock };
    }

    update(remoteClock) {
        for (const [node, time] of Object.entries(remoteClock)) {
            this.clock[node] = Math.max(this.clock[node] || 0, time);
        }
        this.increment();
    }

    // Compare two vector clocks
    static compare(vcA, vcB) {
        let aGreater = false;
        let bGreater = false;

        const allNodes = new Set([...Object.keys(vcA), ...Object.keys(vcB)]);

        for (const node of allNodes) {
            const a = vcA[node] || 0;
            const b = vcB[node] || 0;
            if (a > b) aGreater = true;
            if (b > a) bGreater = true;
        }

        if (aGreater && !bGreater) return 'A_DOMINATES';
        if (bGreater && !aGreater) return 'B_DOMINATES';
        if (!aGreater && !bGreater) return 'EQUAL';
        return 'CONCURRENT'; // Conflict!
    }
}

/**
 * Eventual Consistency Service
 * bKash-style distributed balance management
 */
class EventualConsistencyService {
    constructor(nodeId, replicaUrls) {
        this.nodeId = nodeId;
        this.replicas = replicaUrls;
        this.localStore = new Map();
        this.vectorClock = new VectorClock(nodeId);
        this.pendingReplications = [];
    }

    /**
     * Write with configurable consistency
     * @param {string} key 
     * @param {any} value 
     * @param {object} options - { consistency: 'one' | 'quorum' | 'all' }
     */
    async write(key, value, options = { consistency: 'one' }) {
        const vc = this.vectorClock.increment();
        const record = {
            key,
            value,
            vectorClock: vc,
            timestamp: Date.now(),
            nodeId: this.nodeId
        };

        // Local write সবসময়
        this.localStore.set(key, record);

        switch (options.consistency) {
            case 'all':
                // Strong consistency - সব node-এ write confirm
                await this.replicateToAll(record);
                break;
            case 'quorum':
                // Quorum write - majority confirm
                await this.replicateToQuorum(record);
                break;
            case 'one':
            default:
                // Eventual - async replication
                this.replicateAsync(record);
                break;
        }

        return { status: 'ok', consistency: options.consistency };
    }

    /**
     * Read with tunable consistency
     */
    async read(key, options = { consistency: 'one' }) {
        switch (options.consistency) {
            case 'all':
                return await this.readFromAllAndResolve(key);
            case 'quorum':
                return await this.readQuorum(key);
            case 'one':
            default:
                return this.localStore.get(key);
        }
    }

    /**
     * Quorum Read implementation
     * N=3, R=2 → 2 nodes-এর agreement দরকার
     */
    async readQuorum(key) {
        const responses = [this.localStore.get(key)];
        const quorumSize = Math.floor(this.replicas.length / 2) + 1;

        const replicaReads = this.replicas.map(url =>
            fetch(`${url}/read/${key}`).then(r => r.json()).catch(() => null)
        );

        const results = await Promise.allSettled(replicaReads);
        for (const result of results) {
            if (result.status === 'fulfilled' && result.value) {
                responses.push(result.value);
            }
            if (responses.length >= quorumSize) break;
        }

        return this.resolveConflicts(responses.filter(Boolean));
    }

    /**
     * Conflict resolution - Last Write Wins + Vector Clock
     */
    resolveConflicts(responses) {
        if (responses.length === 0) return null;

        let winner = responses[0];
        for (let i = 1; i < responses.length; i++) {
            const comparison = VectorClock.compare(
                responses[i].vectorClock,
                winner.vectorClock
            );

            if (comparison === 'A_DOMINATES') {
                winner = responses[i];
            } else if (comparison === 'CONCURRENT') {
                // Concurrent writes - LWW fallback
                if (responses[i].timestamp > winner.timestamp) {
                    winner = responses[i];
                }
            }
        }
        return winner;
    }

    /**
     * Anti-entropy background sync
     * Merkle tree based difference detection
     */
    async antiEntropySync() {
        for (const replicaUrl of this.replicas) {
            try {
                const remoteDigest = await fetch(`${replicaUrl}/digest`).then(r => r.json());
                const localDigest = this.computeDigest();

                const diff = this.findDifferences(localDigest, remoteDigest);
                for (const key of diff) {
                    await this.syncKey(key, replicaUrl);
                }
            } catch (error) {
                console.log(`Sync failed with ${replicaUrl}: ${error.message}`);
            }
        }
    }

    // Async replication (fire-and-forget)
    replicateAsync(record) {
        this.replicas.forEach(url => {
            fetch(`${url}/replicate`, {
                method: 'POST',
                body: JSON.stringify(record),
                headers: { 'Content-Type': 'application/json' }
            }).catch(err => {
                // Queue for retry
                this.pendingReplications.push({ url, record, retries: 0 });
            });
        });
    }

    async replicateToAll(record) { /* ... */ }
    async replicateToQuorum(record) { /* ... */ }
    async readFromAllAndResolve(key) { /* ... */ }
    computeDigest() { return {}; }
    findDifferences(local, remote) { return []; }
    async syncKey(key, url) { /* ... */ }
}

// ====== ব্যবহার উদাহরণ ======

// bKash Balance Service
const bkashService = new EventualConsistencyService('dhaka-node-1', [
    'http://ctg-node:3000',
    'http://sylhet-node:3000',
    'http://khulna-node:3000'
]);

// টাকা পাঠানো - Quorum consistency (safe for money)
await bkashService.write('balance:rahim', 4000, { consistency: 'quorum' });

// Balance check - Read your writes
const balance = await bkashService.read('balance:rahim', { consistency: 'quorum' });

// Prothom Alo Article Likes - Eventual consistency (fast, acceptable delay)
const likeCounter = new GCounter('dhaka-server', ['dhaka-server', 'ctg-server', 'sylhet-server']);
likeCounter.increment(); // Like from Dhaka server
console.log(`Total likes: ${likeCounter.value()}`);

// Two servers merge their counts
const ctgCounter = new GCounter('ctg-server', ['dhaka-server', 'ctg-server', 'sylhet-server']);
ctgCounter.increment();
ctgCounter.increment();

likeCounter.merge(ctgCounter); // Conflict-free merge!
console.log(`After merge: ${likeCounter.value()}`); // 3
```

---

## ⚙️ Tunable Consistency

### Cassandra-style Quorum Configuration:

```
┌─────────────────────────────────────────────────────────┐
│           TUNABLE CONSISTENCY (N=3)                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  N = Total replicas = 3                                  │
│  W = Write acknowledgments needed                        │
│  R = Read acknowledgments needed                         │
│                                                          │
│  Rule: W + R > N → Strong Consistency                    │
│                                                          │
│  ┌─────────────┬───┬───┬───────────────────────┐        │
│  │ Config      │ W │ R │ Guarantee             │        │
│  ├─────────────┼───┼───┼───────────────────────┤        │
│  │ Strong      │ 3 │ 1 │ Always latest         │        │
│  │ Quorum      │ 2 │ 2 │ Latest (overlap)      │        │
│  │ Fast Write  │ 1 │ 3 │ Latest (read heavy)   │        │
│  │ Eventual    │ 1 │ 1 │ May be stale          │        │
│  └─────────────┴───┴───┴───────────────────────┘        │
│                                                          │
│  bKash balance: W=2, R=2 (Quorum - safe for money)      │
│  Prothom Alo likes: W=1, R=1 (Eventual - fast)          │
│  Grameenphone bill: W=3, R=1 (Strong write)             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Consistency in Distributed Caches:

```
┌──────────────────────────────────────────────┐
│     CACHE CONSISTENCY CHALLENGE              │
├──────────────────────────────────────────────┤
│                                              │
│  App Server ──write──► Database              │
│      │                      │                │
│      │    Cache stale!      │ (replicated)   │
│      ▼                      ▼                │
│  Redis Cache ◄── invalidation delay ──┘      │
│                                              │
│  সমাধান:                                     │
│  1. Write-through cache                      │
│  2. Cache invalidation on write              │
│  3. TTL-based expiration                     │
│  4. Change Data Capture (CDC)                │
│                                              │
└──────────────────────────────────────────────┘
```

---

## 🎯 Designing for Eventual Consistency

### Best Practices:

```
┌──────────────────────────────────────────────────────┐
│  DESIGN PATTERNS FOR EVENTUAL CONSISTENCY            │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Idempotent Operations                            │
│     ✅ SET balance = 4000                            │
│     ❌ DECREMENT balance BY 1000                     │
│                                                      │
│  2. Compensating Transactions                        │
│     Transfer failed? → Reverse the debit             │
│                                                      │
│  3. Conflict-free Data Structures (CRDTs)            │
│     Counters, Sets, Registers that auto-merge        │
│                                                      │
│  4. Event Sourcing                                   │
│     Store events, not state                          │
│     Replay to get current state                      │
│                                                      │
│  5. Saga Pattern                                     │
│     Distributed transactions as event chains         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## ✅❌ কখন ব্যবহার করবেন / করবেন না

### ✅ Eventual Consistency ব্যবহার করবেন:

| Use Case | উদাহরণ | কারণ |
|---|---|---|
| Social media likes/views | Prothom Alo article likes | সামান্য delay গ্রহণযোগ্য |
| DNS propagation | Domain name changes | গ্লোবাল propagation time লাগে |
| Shopping cart | Daraz cart sync | Eventually sync হলেই চলে |
| User session data | Login status across services | Short delay acceptable |
| Analytics/Metrics | Grameenphone data usage | Real-time precision অপ্রয়োজনীয় |
| Content delivery | CDN cached content | Slight staleness OK |

### ❌ Eventual Consistency ব্যবহার করবেন না:

| Use Case | উদাহরণ | কারণ |
|---|---|---|
| Financial transactions | bKash মূল balance | টাকা হারাতে পারে |
| Inventory (critical) | Last item in stock | Overselling হতে পারে |
| Booking/Reservation | Pathao seat booking | Double booking হবে |
| Authentication | Login/password change | Security risk |
| Sequential operations | Order processing steps | Order ভুল হবে |

### 🎯 Decision Matrix:

```
                    High Availability Needed?
                    YES                NO
                ┌──────────────┬──────────────┐
 Data Loss      │              │              │
 Acceptable?    │  EVENTUAL    │   STRONG     │
 YES            │  (likes,     │   (with      │
                │   views)     │   timeout)   │
                ├──────────────┼──────────────┤
 Data Loss      │              │              │
 Acceptable?    │  QUORUM      │   STRONG     │
 NO             │  (balance    │   (bank      │
                │   check)     │   transfer)  │
                └──────────────┴──────────────┘
```

---

## 📝 সারসংক্ষেপ

> **মনে রাখবেন**: "Eventual consistency is not about being wrong — it's about being temporarily outdated."

- Strong consistency = সবাই সবসময় একই জিনিস দেখে (slow)
- Eventual consistency = সবাই **শেষ পর্যন্ত** একই জিনিস দেখবে (fast)
- আপনার use case অনুযায়ী সঠিক consistency level বেছে নিন
- Financial data → Strong/Quorum
- Social features → Eventual
- CRDTs ব্যবহার করলে conflict resolution automatic হয়
