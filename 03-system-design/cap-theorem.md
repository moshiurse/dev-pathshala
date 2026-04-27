# 🔺 CAP থিওরেম — ডিস্ট্রিবিউটেড সিস্টেমের মৌলিক সীমাবদ্ধতা

> **"একটি ডিস্ট্রিবিউটেড সিস্টেম একই সাথে তিনটির মধ্যে সর্বোচ্চ দুইটি গ্যারান্টি দিতে পারে: Consistency, Availability, এবং Partition Tolerance।"**
> — Eric Brewer, 2000 (UC Berkeley)

---

## 📌 সংজ্ঞা — Eric Brewer's Theorem

২০০০ সালে UC Berkeley-র কম্পিউটার সায়েন্টিস্ট **Eric Brewer** তাঁর কীনোটে এই conjecture উপস্থাপন করেন। পরবর্তীতে ২০০২ সালে **Seth Gilbert** এবং **Nancy Lynch** (MIT) এটি আনুষ্ঠানিকভাবে প্রমাণ করেন।

### ফর্মাল ডেফিনিশন

একটি ডিস্ট্রিবিউটেড ডেটা স্টোর নিম্নোক্ত তিনটি প্রপার্টির মধ্যে **একই সাথে সর্বোচ্চ দুটি** গ্যারান্টি দিতে পারে:

| প্রপার্টি | সংক্ষেপ | অর্থ |
|-----------|---------|------|
| **Consistency** | C | প্রতিটি read অপারেশন সবচেয়ে সাম্প্রতিক write-এর ডেটা ফেরত দেবে, অথবা error দেবে |
| **Availability** | A | প্রতিটি request একটি (non-error) response পাবে — গ্যারান্টি নেই যে সেটি সবচেয়ে সাম্প্রতিক write ধারণ করবে |
| **Partition Tolerance** | P | নেটওয়ার্ক পার্টিশন (নোডগুলোর মধ্যে যোগাযোগ বিচ্ছিন্ন) হলেও সিস্টেম কাজ চালিয়ে যাবে |

### কেন এটি গুরুত্বপূর্ণ?

বাস্তব ডিস্ট্রিবিউটেড সিস্টেমে নেটওয়ার্ক পার্টিশন **অবশ্যম্ভাবী** — তাই P বাদ দেওয়া সম্ভব নয়। ফলে আমাদের আসল সিদ্ধান্ত হলো: **C বনাম A** — পার্টিশন ঘটলে কোনটি ত্যাগ করবেন?

---

## 📊 CAP Triangle — ASCII ডায়াগ্রাম

```
                         C (Consistency)
                          /\
                         /  \
                        /    \
                       / CP   \
                      / Systems \
                     /  ________\
                    / /MongoDB  \ \
                   / / HBase     \ \
                  / / Zookeeper   \ \
                 / / Redis Cluster \ \
                /  /                \  \
               /  /    CA Systems    \  \
              /  /   (Single-node     \  \
             /  /   PostgreSQL,MySQL)   \  \
            /  /   ⚠️ Not possible      \  \
           /  /    in distributed         \  \
          /  /__________ _________________ \  \
         /  AP Systems                  CP  \
        / Cassandra    DynamoDB    CouchDB   \
       /    DNS       Riak       Couchbase    \
      /________________________________________________\
    A (Availability)                    P (Partition Tolerance)


    ┌─────────────────────────────────────────────────────┐
    │         CAP Trade-off Decision Space                │
    │                                                     │
    │  ┌──────────┐   ┌──────────┐   ┌──────────┐       │
    │  │    CP    │   │    AP    │   │    CA    │       │
    │  │Consistent│   │Available │   │Both C+A  │       │
    │  │Partition │   │Partition │   │No Partition│      │
    │  │Tolerant  │   │Tolerant  │   │Tolerance  │      │
    │  └──────────┘   └──────────┘   └──────────┘       │
    │  MongoDB        Cassandra      Single-node         │
    │  HBase          DynamoDB       PostgreSQL           │
    │  Zookeeper      CouchDB        (not distributed)   │
    │  Redis Cluster  DNS                                 │
    │  Etcd           Riak                                │
    └─────────────────────────────────────────────────────┘
```

---

## 💻 Deep Dive — C, A, P প্রতিটি প্রপার্টির বিস্তারিত বিশ্লেষণ

---

### 🔵 Consistency (সামঞ্জস্যতা)

Consistency মানে সব নোড **একই সময়ে একই ডেটা** দেখে। কিন্তু consistency-র বিভিন্ন স্তর আছে — প্রতিটি ভিন্ন গ্যারান্টি দেয়।

#### ১. Linearizability (Strong Consistency)

**সবচেয়ে শক্তিশালী** consistency মডেল। প্রতিটি অপারেশন তাৎক্ষণিকভাবে সব নোডে প্রতিফলিত হয়। মনে হয় যেন একটি মাত্র কপি আছে।

```
   Client A: write(x=5) ──────┐
                               ▼
   ┌─────────────────────────────────────┐
   │  Linearizable System               │
   │  সব নোড atomic-ভাবে আপডেট হয়     │
   │  write(x=5) → সাথে সাথে সব জায়গায় │
   └─────────────────────────────────────┘
                               │
   Client B: read(x) ─────────┘ → অবশ্যই 5 পাবে
```

```php
<?php
/**
 * Linearizable Read — Strong Consistency Example
 * এখানে read করার আগে leader-এর সাথে confirm করা হচ্ছে
 */
class LinearizableStore
{
    private string $leaderUrl;

    public function __construct(string $leaderUrl)
    {
        $this->leaderUrl = $leaderUrl;
    }

    /**
     * read করার আগে leader-এর কাছ থেকে সর্বশেষ committed value নিশ্চিত করে
     * Quorum read বা leader read — দুটোই linearizable হতে পারে
     */
    public function linearizableRead(string $key): mixed
    {
        // leader-কে জিজ্ঞেস করো — সবচেয়ে সাম্প্রতিক committed value কী?
        $ch = curl_init("{$this->leaderUrl}/read?key={$key}&consistency=linearizable");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 5);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            // consistency গ্যারান্টি দিতে না পারলে error — availability sacrifice
            throw new \RuntimeException(
                "Linearizable read ব্যর্থ — leader unreachable। CP trade-off!"
            );
        }

        return json_decode($response, true)['value'];
    }

    /**
     * Write — সব replica-তে confirm হওয়ার পরেই success
     */
    public function linearizableWrite(string $key, mixed $value): bool
    {
        $payload = json_encode(['key' => $key, 'value' => $value]);

        $ch = curl_init("{$this->leaderUrl}/write");
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $payload,
            CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 10,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode !== 200) {
            throw new \RuntimeException("Write failed — all replicas must acknowledge");
        }

        $result = json_decode($response, true);
        return $result['replicated_to_all'] === true;
    }
}
```

#### ২. Sequential Consistency

সব অপারেশন একটি **গ্লোবাল অর্ডারে** সম্পন্ন হয়, তবে real-time ordering গ্যারান্টি নেই। অর্থাৎ ক্লায়েন্ট A-র write সম্পন্ন হওয়ার পর ক্লায়েন্ট B সেটি **এখনও** পুরোনো মান দেখতে পারে, কিন্তু B-র নিজের অপারেশনগুলো সঠিক ক্রমে থাকবে।

```
  Timeline:
  Client A: W(x=1) ────── W(x=2) ──────────>
  Client B: ──────── R(x=1) ──── R(x=2) ──>   ✅ valid
  Client B: ──────── R(x=2) ──── R(x=1) ──>   ❌ invalid (backward jump)
```

#### ৩. Causal Consistency

**কারণ-ও-ফলাফল (cause-effect)** সম্পর্কিত অপারেশনগুলো সঠিক ক্রমে দেখা যায়। causal সম্পর্ক নেই এমন অপারেশনগুলো যেকোনো ক্রমে আসতে পারে।

```
  Client A: W(x=1) ─────────────────────>
                 │ causally depends
                 ▼
  Client A: W(y=2) ─────────────────────>

  Client B দেখবে: x=1 আগে, y=2 পরে (causal order মানতে হবে)
  কিন্তু unrelated z=99 যেকোনো সময় দেখা যেতে পারে
```

```javascript
/**
 * Causal Consistency — Vector Clock ভিত্তিক implementation
 * প্রতিটি write-এ causal dependency ট্র্যাক হয়
 */
class CausalStore {
    constructor(nodeId, peers = []) {
        this.nodeId = nodeId;
        this.peers = peers;
        this.data = new Map();
        // Vector clock — প্রতিটি নোডের জন্য একটি counter
        this.vectorClock = {};
        this.pendingWrites = [];
    }

    /**
     * write করলে vector clock বাড়াও এবং causal dependency রেকর্ড করো
     */
    write(key, value) {
        // নিজের clock বাড়াও
        this.vectorClock[this.nodeId] = (this.vectorClock[this.nodeId] || 0) + 1;

        const entry = {
            key,
            value,
            vectorClock: { ...this.vectorClock }, // snapshot
            origin: this.nodeId,
            timestamp: Date.now(),
        };

        this.data.set(key, entry);
        this._replicateToPeers(entry);

        return entry;
    }

    /**
     * read — শুধু সেই write দেখাবে যার causal dependencies পূরণ হয়েছে
     */
    read(key) {
        const entry = this.data.get(key);
        if (!entry) return null;

        // causal dependency check — এই entry-র vector clock
        // আমাদের local clock-এর সমান বা ছোট হতে হবে
        if (this._isCausallyReady(entry.vectorClock)) {
            return entry.value;
        }

        // dependency এখনও আসেনি — stale value return করো
        return this._getLastCausallyConsistentValue(key);
    }

    /**
     * remote write apply করার আগে causal order check
     */
    applyRemoteWrite(entry) {
        if (this._isCausallyReady(entry.vectorClock)) {
            this.data.set(entry.key, entry);
            this._mergeVectorClock(entry.vectorClock);
            this._applyPendingWrites(); // queue-এ থাকা writes চেক করো
        } else {
            // causal dependency মেটেনি — queue-এ রাখো
            this.pendingWrites.push(entry);
        }
    }

    _isCausallyReady(remoteClock) {
        for (const [nodeId, count] of Object.entries(remoteClock)) {
            if (nodeId === this.nodeId) continue;
            const localCount = this.vectorClock[nodeId] || 0;
            if (count > localCount + 1) return false;
        }
        return true;
    }

    _mergeVectorClock(remoteClock) {
        for (const [nodeId, count] of Object.entries(remoteClock)) {
            this.vectorClock[nodeId] = Math.max(
                this.vectorClock[nodeId] || 0,
                count
            );
        }
    }

    _applyPendingWrites() {
        let applied = true;
        while (applied) {
            applied = false;
            this.pendingWrites = this.pendingWrites.filter(entry => {
                if (this._isCausallyReady(entry.vectorClock)) {
                    this.data.set(entry.key, entry);
                    this._mergeVectorClock(entry.vectorClock);
                    applied = true;
                    return false; // remove from pending
                }
                return true; // keep in pending
            });
        }
    }

    _replicateToPeers(entry) {
        this.peers.forEach(peer => peer.applyRemoteWrite({ ...entry }));
    }

    _getLastCausallyConsistentValue(key) {
        return null; // simplified — আসলে last known consistent value রাখা হয়
    }
}
```

#### ৪. Eventual Consistency

**সবচেয়ে দুর্বল** গ্যারান্টি — নতুন কোনো write না হলে, **শেষ পর্যন্ত** সব replica একই মানে converge করবে। কিন্তু কখন? কোনো time bound নেই।

```
  t=0   Node A: x=5    Node B: x=3    Node C: x=3   (inconsistent)
  t=1   Node A: x=5    Node B: x=5    Node C: x=3   (still inconsistent)
  t=2   Node A: x=5    Node B: x=5    Node C: x=5   (converged ✅)

  ⚠️ convergence time: milliseconds থেকে seconds — কোনো গ্যারান্টি নেই
```

---

### 🟢 Availability (প্রাপ্যতা)

Availability মানে **প্রতিটি request** যেকোনো non-failing নোডে পৌঁছালে একটি response পাবে। এটি **গ্যারান্টি দেয় না** যে response-এ সর্বশেষ write-এর ডেটা থাকবে।

```
  ┌─────────┐     request      ┌──────────┐
  │ Client  │ ───────────────> │  Node A  │ ─── response (might be stale)
  └─────────┘                  └──────────┘
                                    ✅ সবসময় response দেবে

  ┌─────────┐     request      ┌──────────┐
  │ Client  │ ───────────────> │  Node B  │ ─── response (might be stale)
  └─────────┘                  └──────────┘
                                    ✅ সবসময় response দেবে

  এমনকি Node A ও Node B-র মধ্যে network বিচ্ছিন্ন হলেও
  দুটোই response দিতে থাকবে (AP system)
```

```php
<?php
/**
 * High Availability Pattern — যেকোনো replica থেকে read/write
 * bKash-এর মতো সিস্টেমে: availability = revenue
 */
class HighAvailabilityStore
{
    private array $replicas;
    private int $timeoutMs;

    public function __construct(array $replicaUrls, int $timeoutMs = 2000)
    {
        $this->replicas  = $replicaUrls;
        $this->timeoutMs = $timeoutMs;
    }

    /**
     * যেকোনো একটি replica থেকে read — প্রথম response-ই গ্রহণযোগ্য
     * Eventual consistency — stale data আসতে পারে
     */
    public function availableRead(string $key): mixed
    {
        // সব replica-তে parallel request পাঠাও
        $multiHandle = curl_multi_init();
        $handles = [];

        foreach ($this->replicas as $i => $url) {
            $ch = curl_init("{$url}/read?key={$key}");
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
            curl_setopt($ch, CURLOPT_TIMEOUT_MS, $this->timeoutMs);
            curl_multi_add_handle($multiHandle, $ch);
            $handles[$i] = $ch;
        }

        // প্রথম successful response পেলেই return করো
        do {
            $status = curl_multi_exec($multiHandle, $running);

            foreach ($handles as $i => $ch) {
                $info = curl_multi_info_read($multiHandle);
                if ($info && $info['handle'] === $ch) {
                    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
                    if ($httpCode === 200) {
                        $response = curl_multi_getcontent($ch);
                        // cleanup
                        foreach ($handles as $h) {
                            curl_multi_remove_handle($multiHandle, $h);
                            curl_close($h);
                        }
                        curl_multi_close($multiHandle);
                        return json_decode($response, true)['value'];
                    }
                }
            }
        } while ($running > 0);

        // cleanup
        foreach ($handles as $h) {
            curl_multi_remove_handle($multiHandle, $h);
            curl_close($h);
        }
        curl_multi_close($multiHandle);

        throw new \RuntimeException("সব replica unavailable — বিরল ঘটনা!");
    }

    /**
     * Hinted Handoff — unavailable node-এর জন্য write অন্য node-এ রাখো
     * পরে সেই node ফিরে এলে forward করো
     */
    public function writeWithHintedHandoff(string $key, mixed $value): array
    {
        $results = ['success' => [], 'hinted' => [], 'failed' => []];

        foreach ($this->replicas as $url) {
            $response = $this->sendWrite($url, $key, $value);

            if ($response['status'] === 'ok') {
                $results['success'][] = $url;
            } elseif ($response['status'] === 'timeout') {
                // hint store করো — পরে forward হবে
                $this->storeHint($url, $key, $value);
                $results['hinted'][] = $url;
            } else {
                $results['failed'][] = $url;
            }
        }

        // কমপক্ষে একটি success হলেই available বলবো
        if (count($results['success']) > 0) {
            return ['status' => 'available', 'details' => $results];
        }

        throw new \RuntimeException("কোনো replica write গ্রহণ করেনি");
    }

    private function sendWrite(string $url, string $key, mixed $value): array
    {
        $ch = curl_init("{$url}/write");
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => json_encode(['key' => $key, 'value' => $value]),
            CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT_MS     => $this->timeoutMs,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode === 200) return ['status' => 'ok'];
        if ($httpCode === 0)   return ['status' => 'timeout'];
        return ['status' => 'error', 'code' => $httpCode];
    }

    private function storeHint(string $targetUrl, string $key, mixed $value): void
    {
        // Hinted handoff queue-এ সংরক্ষণ — পরে target node-এ পাঠানো হবে
        file_put_contents(
            '/var/lib/hints/' . md5($targetUrl . $key) . '.json',
            json_encode([
                'target'    => $targetUrl,
                'key'       => $key,
                'value'     => $value,
                'timestamp' => microtime(true),
            ])
        );
    }
}
```

---

### 🔴 Partition Tolerance (পার্টিশন সহনশীলতা)

নেটওয়ার্ক পার্টিশন মানে দুটি নোডের মধ্যে **message হারিয়ে যাওয়া বা বিলম্বিত হওয়া**। ডিস্ট্রিবিউটেড সিস্টেমে এটি **অনিবার্য** — তাই আমরা P বাদ দিতে পারি না।

```
  ┌───────────────────────────────────────────────┐
  │              নেটওয়ার্ক পার্টিশন               │
  │                                               │
  │   Partition 1         ║        Partition 2     │
  │  ┌─────────┐         ║       ┌─────────┐      │
  │  │ Node A  │   ══X══ ║ ══X══ │ Node C  │      │
  │  │ (x=5)   │         ║       │ (x=3)   │      │
  │  └─────────┘         ║       └─────────┘      │
  │  ┌─────────┐         ║       ┌─────────┐      │
  │  │ Node B  │         ║       │ Node D  │      │
  │  │ (x=5)   │         ║       │ (x=3)   │      │
  │  └─────────┘         ║       └─────────┘      │
  │                      ║                        │
  │  A-B communicate ✅  ║  C-D communicate ✅    │
  │  A-C communicate ❌  ║  B-D communicate ❌    │
  └───────────────────────────────────────────────┘

  CP সিদ্ধান্ত: Partition 2 request reject করবে (unavailable হবে)
  AP সিদ্ধান্ত: দুই partition আলাদাভাবে কাজ করবে (inconsistent হবে)
```

---

## 🔥 CAP Trade-offs — বিস্তারিত বিশ্লেষণ

---

### CP Systems (Consistency + Partition Tolerance)

**পার্টিশন ঘটলে:** availability ত্যাগ করে consistency বজায় রাখে। minority partition-এর নোডগুলো request reject করে।

#### উদাহরণ ডেটাবেস:

| ডেটাবেস | CP কেন? |
|---------|---------|
| **MongoDB** (single-master) | Primary fail হলে election না হওয়া পর্যন্ত write unavailable |
| **HBase** | ZooKeeper-নির্ভর — master unavailable হলে region unavailable |
| **Redis Cluster** | Partition-এ minority shard request reject করে |
| **ZooKeeper** | Quorum না পেলে read/write refuse করে |
| **Etcd** | Raft consensus — majority ছাড়া কাজ করে না |
| **CockroachDB** | Serializable isolation — partition-এ কিছু range unavailable |

```
  CP System Behavior During Partition:
  ┌────────────────────────────────────────────────┐
  │                                                │
  │   Majority Partition       Minority Partition  │
  │   (3 out of 5 nodes)      (2 out of 5 nodes)  │
  │                                                │
  │   ┌──────────┐            ┌──────────┐        │
  │   │ Node A ✅│            │ Node D ❌│        │
  │   │ Leader   │     X      │ Follower │        │
  │   │ write ok │            │ read-only│        │
  │   └──────────┘            └──────────┘        │
  │   ┌──────────┐            ┌──────────┐        │
  │   │ Node B ✅│            │ Node E ❌│        │
  │   │ Follower │     X      │ Follower │        │
  │   │ read ok  │            │ REJECT!  │        │
  │   └──────────┘            └──────────┘        │
  │   ┌──────────┐                                │
  │   │ Node C ✅│   Majority পক্ষে সব কাজ হয়   │
  │   │ Follower │   Minority পক্ষ unavailable     │
  │   └──────────┘                                │
  └────────────────────────────────────────────────┘
```

---

### AP Systems (Availability + Partition Tolerance)

**পার্টিশন ঘটলে:** consistency ত্যাগ করে — সব নোড request serve করতে থাকে, কিন্তু ভিন্ন ভিন্ন ডেটা দেখাতে পারে। পরে reconciliation হয়।

#### উদাহরণ ডেটাবেস:

| ডেটাবেস | AP কেন? |
|---------|---------|
| **Cassandra** | Tunable consistency, কিন্তু default AP — যেকোনো নোডে write সম্ভব |
| **DynamoDB** | Eventually consistent read default — সবসময় available |
| **CouchDB** | Multi-master replication — conflict পরে resolve হয় |
| **DNS** | Stale records দেয় কিন্তু সবসময় respond করে |
| **Riak** | Masterless — যেকোনো নোড read/write নেয় |
| **Amazon S3** | Eventually consistent (GET after PUT কখনো stale) |

---

### CA Systems — কেন ডিস্ট্রিবিউটেড সিস্টেমে CA অসম্ভব?

**CA = Consistency + Availability, কিন্তু Partition Tolerance নেই**

একটি **single-node** RDBMS (PostgreSQL, MySQL একটি মাত্র সার্ভারে) কার্যত CA — কোনো partition হওয়ার সুযোগ নেই কারণ একটি মাত্র নোড। কিন্তু:

```
  ┌─────────────────────────────────────────────────────┐
  │  CA সিস্টেম কেন ডিস্ট্রিবিউটেডে অসম্ভব?          │
  │                                                     │
  │  ১. ডিস্ট্রিবিউটেড = একাধিক নোড                   │
  │  ২. একাধিক নোড = নেটওয়ার্ক দিয়ে সংযুক্ত          │
  │  ৩. নেটওয়ার্ক = পার্টিশন সম্ভব (অবশ্যম্ভাবী)     │
  │  ৪. পার্টিশন হলে সিদ্ধান্ত নিতে হবে: C না A?      │
  │                                                     │
  │  তাই ডিস্ট্রিবিউটেড সিস্টেমে P অবশ্যই থাকবে      │
  │  এবং সিদ্ধান্ত সবসময়: CP অথবা AP                   │
  └─────────────────────────────────────────────────────┘

  Single-node PostgreSQL:    CA ✅ (but not distributed!)
  PostgreSQL + Streaming     CP ✅ (sync replica = consistent,
  Replication:                      partition = primary only)
```

---

## 🧠 Advanced Topics

---

### ১. PACELC Theorem — CAP-এর সম্প্রসারণ

CAP শুধু পার্টিশনের সময়ের আচরণ বলে। কিন্তু **পার্টিশন না হলে?** তখনও latency বনাম consistency trade-off থাকে। এটাই PACELC।

```
  PACELC = Partitioned → A vs C, Else → Latency vs Consistency

  ┌──────────────────────────────────────────────────────┐
  │  If Partitioned:                                     │
  │    ├── Choose Availability  (PA)                     │
  │    └── Choose Consistency   (PC)                     │
  │                                                      │
  │  Else (normal operation):                            │
  │    ├── Choose Low Latency   (EL)                     │
  │    └── Choose Consistency   (EC)                     │
  └──────────────────────────────────────────────────────┘

  ┌──────────────┬──────────────┬──────────────────────┐
  │ ডেটাবেস      │ PACELC      │ ব্যাখ্যা             │
  ├──────────────┼──────────────┼──────────────────────┤
  │ DynamoDB     │ PA / EL     │ সবসময় available +    │
  │              │             │ low latency           │
  ├──────────────┼──────────────┼──────────────────────┤
  │ Cassandra    │ PA / EL     │ AP + normal-এ fast    │
  │              │             │ local read            │
  ├──────────────┼──────────────┼──────────────────────┤
  │ MongoDB      │ PC / EC     │ সবসময় consistent     │
  │              │             │ primary read          │
  ├──────────────┼──────────────┼──────────────────────┤
  │ CockroachDB  │ PC / EC     │ serializable + sync   │
  ├──────────────┼──────────────┼──────────────────────┤
  │ Cosmos DB    │ PA/PC / EL/EC│ tunable — 5 level    │
  ├──────────────┼──────────────┼──────────────────────┤
  │ YugabyteDB   │ PC / EC     │ Raft-based strong     │
  └──────────────┴──────────────┴──────────────────────┘
```

---

### ২. Consistency Models in Practice — কোড সহ

#### Strong Consistency (Read-after-Write গ্যারান্টি)

```php
<?php
/**
 * Strong Consistency — Synchronous Replication
 * bKash ব্যালেন্স: একজন পাঠালে সাথে সাথে সবখানে দেখা যেতে হবে
 */
class StrongConsistentBalance
{
    private array $replicas;

    public function __construct(array $replicas)
    {
        $this->replicas = $replicas;
    }

    public function transfer(string $from, string $to, float $amount): bool
    {
        // Phase 1: সব replica-তে prepare
        $prepared = [];
        foreach ($this->replicas as $replica) {
            $result = $replica->prepare($from, $to, $amount);
            if (!$result) {
                // একটিও fail হলে সব rollback — consistency!
                foreach ($prepared as $p) {
                    $p->rollback($from, $to, $amount);
                }
                return false; // unavailable হবে, কিন্তু inconsistent হবে না
            }
            $prepared[] = $replica;
        }

        // Phase 2: সব prepare সফল → commit
        foreach ($this->replicas as $replica) {
            $replica->commit($from, $to, $amount);
        }

        return true;
    }
}
```

#### Eventual Consistency (Read-Your-Writes সহ)

```javascript
/**
 * Read-Your-Writes Consistency
 * Pathao-তে: রাইডার trip শেষ করলে নিজে সাথে সাথে updated balance দেখবে
 * কিন্তু অন্য সার্ভার থেকে পড়লে কিছুক্ষণ পুরোনো balance দেখাতে পারে
 */
class ReadYourWritesStore {
    constructor() {
        this.localWriteTimestamps = new Map(); // key → last write timestamp
        this.replicas = [];
    }

    async write(key, value) {
        const timestamp = Date.now();
        
        // primary-তে write
        await this.replicas[0].write(key, value, timestamp);
        
        // local timestamp মনে রাখো — পরে read করার সময় ব্যবহার হবে
        this.localWriteTimestamps.set(key, timestamp);
        
        // async replication — background-এ হবে
        this._asyncReplicate(key, value, timestamp);
        
        return { success: true, timestamp };
    }

    async read(key) {
        const myLastWrite = this.localWriteTimestamps.get(key) || 0;

        // প্রতিটি replica try করো
        for (const replica of this.replicas) {
            const result = await replica.read(key);
            
            // আমার নিজের write-এর পর হতে হবে
            if (result && result.timestamp >= myLastWrite) {
                return result.value;
            }
        }

        // fallback: primary থেকে পড়ো (guaranteed fresh)
        const primaryResult = await this.replicas[0].read(key);
        return primaryResult ? primaryResult.value : null;
    }

    async _asyncReplicate(key, value, timestamp) {
        // fire-and-forget — background-এ replicate হবে
        const promises = this.replicas.slice(1).map(replica =>
            replica.write(key, value, timestamp).catch(err => {
                console.warn(`Replication to ${replica.id} failed, will retry: ${err}`);
                this._scheduleRetry(replica, key, value, timestamp);
            })
        );
        // await করবো না — availability বজায় থাকবে
        Promise.allSettled(promises);
    }

    _scheduleRetry(replica, key, value, timestamp) {
        setTimeout(() => {
            replica.write(key, value, timestamp).catch(() => {
                this._scheduleRetry(replica, key, value, timestamp);
            });
        }, 5000); // 5 সেকেন্ড পর retry
    }
}
```

---

### ৩. Conflict Resolution — যখন দুই পক্ষ ভিন্ন ডেটা লেখে

AP সিস্টেমে পার্টিশনের সময় দুই পক্ষে ভিন্ন write হতে পারে। পার্টিশন heal হলে **conflict resolve** করতে হবে।

#### কৌশল ১: Last-Write-Wins (LWW)

```
  Node A: write(x="Dhaka", t=100)     ──┐
                                         ├── Partition heals → x="Chittagong" (t=105 > t=100)
  Node B: write(x="Chittagong", t=105) ──┘

  ⚠️ সমস্যা: Node A-র write চিরতরে হারিয়ে গেল — data loss!
```

#### কৌশল ২: Vector Clocks

```javascript
/**
 * Vector Clock — concurrent write detect করে
 * conflict থাকলে application-কে জানায়
 */
class VectorClock {
    constructor() {
        this.clock = {};
    }

    increment(nodeId) {
        this.clock[nodeId] = (this.clock[nodeId] || 0) + 1;
        return this;
    }

    merge(otherClock) {
        const merged = new VectorClock();
        const allNodes = new Set([
            ...Object.keys(this.clock),
            ...Object.keys(otherClock.clock)
        ]);
        for (const node of allNodes) {
            merged.clock[node] = Math.max(
                this.clock[node] || 0,
                otherClock.clock[node] || 0
            );
        }
        return merged;
    }

    /**
     * compare: "before", "after", "concurrent", "equal"
     */
    compare(other) {
        let dominated = false;
        let dominates = false;
        const allNodes = new Set([
            ...Object.keys(this.clock),
            ...Object.keys(other.clock)
        ]);

        for (const node of allNodes) {
            const a = this.clock[node] || 0;
            const b = other.clock[node] || 0;
            if (a < b) dominated = true;
            if (a > b) dominates = true;
        }

        if (!dominated && !dominates) return 'equal';
        if (dominated && !dominates) return 'before';     // this < other
        if (!dominated && dominates) return 'after';      // this > other
        return 'concurrent';  // ⚠️ CONFLICT — উভয় পক্ষে ভিন্ন write!
    }
}

// ব্যবহার:
const clockA = new VectorClock();
clockA.increment('nodeA'); // {nodeA: 1}

const clockB = new VectorClock();
clockB.increment('nodeB'); // {nodeB: 1}

console.log(clockA.compare(clockB)); // "concurrent" — CONFLICT!
```

#### কৌশল ৩: CRDTs (Conflict-free Replicated Data Types)

```javascript
/**
 * G-Counter CRDT — শুধু বাড়ে, কখনো কমে না
 * উদাহরণ: Pathao-তে total ride count
 * সব নোড স্বাধীনভাবে বাড়াতে পারে, merge করলে correct total পাওয়া যায়
 */
class GCounter {
    constructor(nodeId) {
        this.nodeId = nodeId;
        this.counts = {}; // {nodeId → count}
    }

    increment(amount = 1) {
        this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + amount;
    }

    value() {
        return Object.values(this.counts).reduce((sum, c) => sum + c, 0);
    }

    /**
     * merge — কোনো conflict নেই!
     * প্রতিটি নোডের সর্বোচ্চ count নাও — তাতেই correct total
     */
    merge(other) {
        const merged = new GCounter(this.nodeId);
        const allNodes = new Set([
            ...Object.keys(this.counts),
            ...Object.keys(other.counts)
        ]);
        for (const node of allNodes) {
            merged.counts[node] = Math.max(
                this.counts[node] || 0,
                other.counts[node] || 0
            );
        }
        return merged;
    }
}

// Partition-এর দুই পাশে স্বাধীনভাবে increment:
const counterA = new GCounter('dhaka-dc');
counterA.increment(10); // 10 rides in Dhaka DC

const counterB = new GCounter('ctg-dc');
counterB.increment(7);  // 7 rides in Chittagong DC

// Partition heal → merge → correct total!
const merged = counterA.merge(counterB);
console.log(merged.value()); // 17 — কোনো conflict নেই!
```

#### কৌশল ৪: Manual/Application-level Merge

```php
<?php
/**
 * Shopping Cart Merge — Amazon/Daraz স্টাইলে
 * দুটি version-ই রাখো, user-কে দেখাও
 */
class CartConflictResolver
{
    /**
     * দুটি conflicting cart merge করো — union strategy
     * কোনো item হারাবে না
     */
    public function mergeCartConflict(array $cartA, array $cartB): array
    {
        $merged = [];

        $allItems = array_unique(
            array_merge(array_keys($cartA), array_keys($cartB))
        );

        foreach ($allItems as $itemId) {
            $qtyA = $cartA[$itemId] ?? 0;
            $qtyB = $cartB[$itemId] ?? 0;
            // বেশি quantity রাখো — কোনো item মুছবে না
            $merged[$itemId] = max($qtyA, $qtyB);
        }

        return $merged;
    }
}

// উদাহরণ:
$resolver = new CartConflictResolver();
$merged = $resolver->mergeCartConflict(
    ['item-1' => 2, 'item-2' => 1],           // Partition A-তে user যোগ করেছে
    ['item-1' => 1, 'item-3' => 3]            // Partition B-তে যোগ করেছে
);
// result: ['item-1' => 2, 'item-2' => 1, 'item-3' => 3]
```

---

### ৪. Quorum Consensus — W + R > N

Quorum হলো **"কতজনের সাথে কথা বললে সঠিক উত্তর পাবো?"** এর গাণিতিক সমাধান।

```
  N = মোট replica সংখ্যা
  W = write quorum (কতগুলো replica-তে write confirm হতে হবে)
  R = read quorum (কতগুলো replica থেকে read করতে হবে)

  Strong Consistency: W + R > N

  ┌────────────────────────────────────────────────┐
  │ উদাহরণ: N=3, W=2, R=2                         │
  │                                                │
  │ Write(x=5):                                    │
  │   Node A: ✅ (write done)                      │
  │   Node B: ✅ (write done)     W=2 achieved!    │
  │   Node C: ❌ (not yet)                         │
  │                                                │
  │ Read(x):                                       │
  │   Node A: x=5  ✅                              │
  │   Node C: x=3  (stale)        R=2 achieved!   │
  │                                                │
  │ W(2) + R(2) = 4 > N(3)                        │
  │ → কমপক্ষে একটি নোড write ও read দুটোতেই আছে  │
  │ → সর্বশেষ value অবশ্যই পাওয়া যাবে (Node A)   │
  └────────────────────────────────────────────────┘

  Trade-off Matrix:
  ┌────────┬────────┬──────────┬──────────────────────┐
  │ Config │ W + R  │ Guarantee│ Use Case             │
  ├────────┼────────┼──────────┼──────────────────────┤
  │ W=3,R=1│ 4 > 3  │ Strong   │ Fast read, slow write│
  │ W=1,R=3│ 4 > 3  │ Strong   │ Fast write, slow read│
  │ W=2,R=2│ 4 > 3  │ Strong   │ Balanced             │
  │ W=1,R=1│ 2 < 3  │ Eventual │ Max performance      │
  └────────┴────────┴──────────┴──────────────────────┘
```

```php
<?php
/**
 * Quorum-based Read/Write
 * N=5 replica cluster-এ configurable consistency
 */
class QuorumStore
{
    private array  $replicas;
    private int    $n;
    private int    $writeQuorum;
    private int    $readQuorum;

    public function __construct(array $replicas, int $w, int $r)
    {
        $this->replicas    = $replicas;
        $this->n           = count($replicas);
        $this->writeQuorum = $w;
        $this->readQuorum  = $r;
    }

    public function isStronglyConsistent(): bool
    {
        return ($this->writeQuorum + $this->readQuorum) > $this->n;
    }

    /**
     * Quorum Write — W replica-তে confirm হলেই success
     */
    public function quorumWrite(string $key, mixed $value): array
    {
        $version   = microtime(true);
        $acks      = 0;
        $responses = [];

        foreach ($this->replicas as $replica) {
            try {
                $result = $replica->write($key, $value, $version);
                if ($result) {
                    $acks++;
                    $responses[] = ['replica' => $replica->getId(), 'status' => 'ack'];
                }
            } catch (\Exception $e) {
                $responses[] = ['replica' => $replica->getId(), 'status' => 'failed'];
            }

            // W পেয়ে গেলে early return — availability বাড়ে
            if ($acks >= $this->writeQuorum) {
                return [
                    'success'     => true,
                    'acks'        => $acks,
                    'quorum_met'  => true,
                    'version'     => $version,
                    'responses'   => $responses,
                ];
            }
        }

        // quorum meet হয়নি
        return [
            'success'    => false,
            'acks'       => $acks,
            'quorum_met' => false,
            'error'      => "Write quorum না পাওয়ায় ব্যর্থ: {$acks}/{$this->writeQuorum}",
        ];
    }

    /**
     * Quorum Read — R replica থেকে পড়ো, সবচেয়ে নতুন version নাও
     */
    public function quorumRead(string $key): array
    {
        $responses = [];
        $reads     = 0;

        foreach ($this->replicas as $replica) {
            try {
                $result = $replica->read($key);
                if ($result !== null) {
                    $reads++;
                    $responses[] = $result;
                }
            } catch (\Exception $e) {
                // replica unavailable — skip
            }

            if ($reads >= $this->readQuorum) {
                break;
            }
        }

        if ($reads < $this->readQuorum) {
            throw new \RuntimeException(
                "Read quorum পূরণ হয়নি: {$reads}/{$this->readQuorum}"
            );
        }

        // সবচেয়ে নতুন version নাও
        usort($responses, fn($a, $b) => $b['version'] <=> $a['version']);
        $latest = $responses[0];

        // Read repair: পুরোনো replica-গুলোতে latest value পাঠাও
        $this->readRepair($key, $latest, $responses);

        return $latest;
    }

    /**
     * Read Repair — পুরোনো replica-গুলোকে আপডেট করো
     */
    private function readRepair(string $key, array $latest, array $responses): void
    {
        foreach ($responses as $resp) {
            if ($resp['version'] < $latest['version']) {
                // background-এ আপডেট করো
                $replica = $this->findReplica($resp['replica_id']);
                $replica?->write($key, $latest['value'], $latest['version']);
            }
        }
    }

    private function findReplica(string $id): ?object
    {
        foreach ($this->replicas as $r) {
            if ($r->getId() === $id) return $r;
        }
        return null;
    }
}
```

```javascript
/**
 * Quorum Configuration Helper
 * বিভিন্ন use case-এ কোন quorum সেটিং সবচেয়ে ভালো?
 */
class QuorumCalculator {
    constructor(n) {
        this.n = n;
    }

    /**
     * Strong consistency-র জন্য minimum quorum বের করো
     */
    getStrongConsistencyOptions() {
        const options = [];

        for (let w = 1; w <= this.n; w++) {
            for (let r = 1; r <= this.n; r++) {
                if (w + r > this.n) {
                    options.push({
                        W: w,
                        R: r,
                        writeFaultTolerance: this.n - w,  // কতটি node fail সহ্য করবে
                        readFaultTolerance: this.n - r,
                        writeLatency: w > this.n / 2 ? 'high' : 'low',
                        readLatency: r > this.n / 2 ? 'high' : 'low',
                    });
                }
            }
        }

        return options;
    }

    /**
     * bKash-এর মতো financial system-এর জন্য recommended config
     */
    getFinancialSystemConfig() {
        const w = Math.floor(this.n / 2) + 1; // majority
        const r = Math.floor(this.n / 2) + 1; // majority

        return {
            N: this.n,
            W: w,
            R: r,
            stronglyConsistent: true,
            faultTolerance: Math.floor((this.n - 1) / 2),
            note: `bKash ব্যালেন্স: ${w} replica confirm না হলে transaction fail হবে`
        };
    }

    /**
     * Pathao-র মতো ride-sharing-এর জন্য — availability প্রাধান্য
     */
    getRideShareConfig() {
        return {
            N: this.n,
            W: 1,  // একটিতে write হলেই enough
            R: 1,  // একটি থেকে পড়লেই enough
            stronglyConsistent: false,
            note: 'Pathao driver location: eventual consistency OK — 2-3 সেকেন্ড পুরোনো location গ্রহণযোগ্য'
        };
    }
}

const calc = new QuorumCalculator(5);
console.log(calc.getFinancialSystemConfig());
// { N: 5, W: 3, R: 3, stronglyConsistent: true, faultTolerance: 2, ... }
console.log(calc.getRideShareConfig());
// { N: 5, W: 1, R: 1, stronglyConsistent: false, ... }
```

---

### ৫. Split-Brain Problem এবং সমাধান

Split-brain ঘটে যখন নেটওয়ার্ক পার্টিশনের কারণে **দুটি partition-ই নিজেকে primary** মনে করে এবং স্বাধীনভাবে write গ্রহণ করে।

```
  Normal Operation:
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │ Node A  │ ──> │ Node B  │ ──> │ Node C  │
  │ PRIMARY │     │ REPLICA │     │ REPLICA │
  └─────────┘     └─────────┘     └─────────┘

  Split-Brain! (Network Partition):
  ┌─────────────────┐  ║  ┌─────────────────┐
  │ Node A          │  ║  │ Node B          │
  │ (thinks PRIMARY)│  ║  │ (elects itself  │
  │ write: x=100    │  ║  │  as PRIMARY!)   │
  │                 │  X  │ write: x=200    │
  └─────────────────┘  ║  │                 │
                       ║  │ Node C          │
                       ║  │ (follows B)     │
                       ║  └─────────────────┘

  ⚠️ দুটি primary! দুটি ভিন্ন x value! DATA INCONSISTENCY!
```

#### সমাধান ১: Fencing Token

```php
<?php
/**
 * Fencing Token — পুরোনো leader-কে ব্লক করো
 * প্রতিটি leader election-এ একটি monotonically increasing token তৈরি হয়
 */
class FencingTokenManager
{
    private int    $currentToken;
    private string $lockStore; // distributed lock store URL

    public function __construct(string $lockStoreUrl)
    {
        $this->lockStore    = $lockStoreUrl;
        $this->currentToken = 0;
    }

    /**
     * নতুন leader election-এ নতুন token নাও
     */
    public function acquireLeadership(): int
    {
        $this->currentToken = $this->getNextToken();
        return $this->currentToken;
    }

    /**
     * write করার সময় fencing token পাঠাও
     * storage layer token verify করবে
     */
    public function writeWithFencing(string $key, mixed $value): bool
    {
        $payload = [
            'key'           => $key,
            'value'         => $value,
            'fencing_token' => $this->currentToken,
        ];

        // Storage layer check করবে:
        // if (request.token < storage.lastSeenToken) → REJECT!
        // পুরোনো leader-র write আর গ্রহণ হবে না
        $response = $this->sendToStorage($payload);

        return $response['accepted'] ?? false;
    }

    private function getNextToken(): int
    {
        // Atomic increment — ZooKeeper বা etcd-র মাধ্যমে
        return time() * 1000 + random_int(0, 999);
    }

    private function sendToStorage(array $payload): array
    {
        // Storage-এ fencing token সহ write পাঠানো
        return ['accepted' => true];
    }
}
```

#### সমাধান ২: STONITH (Shoot The Other Node In The Head)

```
  Split-brain detect হলে:
  ┌──────────────────────────────────────────────┐
  │  ১. Quorum check — majority আমার পক্ষে?     │
  │  ২. না হলে → নিজেকে shutdown করো            │
  │  ৩. হ্যাঁ হলে → অন্য partition-কে fence করো │
  │     (power off, IPMI reset, disk I/O block)  │
  └──────────────────────────────────────────────┘

  Node A (minority):  "আমি majority নই → আমি step down করি"
  Node B (majority):  "আমি majority → আমি নতুন primary"
```

---

### ৬. Real-world CAP Decisions — বাংলাদেশ প্রসঙ্গ

#### 🟢 bKash — ব্যালেন্স Consistency অবশ্যই (CP)

```
  ┌─────────────────────────────────────────────────────┐
  │  bKash Balance Transfer                             │
  │                                                     │
  │  ❌ ভুল (AP approach):                             │
  │  রহিম: ৫০০০ টাকা পাঠালো → Node A: balance=৫০০০    │
  │  করিম: ৫০০০ টাকা পেলো  → Node B: balance=১৫০০০   │
  │  কিন্তু Node A আর B sync হয়নি!                   │
  │  রহিম আবার ৫০০০ পাঠাতে পারলো!                     │
  │  ⚠️ ডাবল স্পেন্ডিং — টাকা তৈরি হয়ে গেল!        │
  │                                                     │
  │  ✅ সঠিক (CP approach):                            │
  │  রহিম: ৫০০০ পাঠালো → LOCK → সব replica sync →     │
  │  confirm → UNLOCK                                   │
  │  Partition হলে? → "সাময়িক অসুবিধার জন্য দুঃখিত"   │
  │  Unavailable, কিন্তু কখনো inconsistent নয়          │
  └─────────────────────────────────────────────────────┘
```

#### 🟡 Pathao — Driver Location Availability প্রাধান্য (AP)

```
  ┌─────────────────────────────────────────────────────┐
  │  Pathao Driver Location                             │
  │                                                     │
  │  Driver location ২-৩ সেকেন্ড পুরোনো?              │
  │  → গ্রহণযোগ্য! GPS নিজেই ~3m error দেয়           │
  │                                                     │
  │  কিন্তু location service unavailable?              │
  │  → ❌ কেউ রাইড বুক করতে পারছে না!                  │
  │  → Revenue loss!                                    │
  │                                                     │
  │  তাই: Eventual consistency + high availability     │
  │  সব সার্ভার independently location দেখায়            │
  │  কিছুক্ষণ inconsistent হলেও always available        │
  └─────────────────────────────────────────────────────┘
```

#### 🔴 Shohoz Bus Ticket — Mixed Approach

```
  ┌─────────────────────────────────────────────────────┐
  │  Shohoz Ticket Booking                              │
  │                                                     │
  │  সিট সার্চ: AP (eventual consistency OK)           │
  │  → কখনো কখনো "available" দেখায় কিন্তু আসলে sold  │
  │  → গ্রহণযোগ্য — booking-এর সময় check হবে          │
  │                                                     │
  │  পেমেন্ট ও সিট লক: CP (strong consistency)        │
  │  → double booking হলে মারাত্মক!                     │
  │  → partition-এ unavailable হবে, কিন্তু              │
  │    দুজনকে একই সিট দেবে না                          │
  └─────────────────────────────────────────────────────┘
```

---

## 🆚 Database CAP Classification Table

| # | ডেটাবেস | CAP | PACELC | Default Consistency | Tunable? | নোট |
|---|---------|-----|--------|-------------------|----------|------|
| 1 | **PostgreSQL** (single) | CA | N/A | Strong | N/A | Single-node — partition N/A |
| 2 | **MySQL** (single) | CA | N/A | Strong | N/A | Single-node — partition N/A |
| 3 | **MongoDB** | CP | PC/EC | Strong (primary read) | হ্যাঁ | readPreference দিয়ে tunable |
| 4 | **Cassandra** | AP | PA/EL | Eventual | হ্যাঁ | ConsistencyLevel.QUORUM → CP-তে যায় |
| 5 | **DynamoDB** | AP | PA/EL | Eventual | হ্যাঁ | ConsistentRead=true → strong |
| 6 | **CouchDB** | AP | PA/EL | Eventual | না | Multi-master, conflict resolution |
| 7 | **Redis Cluster** | CP | PC/EL | Strong | আংশিক | Async replication → data loss possible |
| 8 | **HBase** | CP | PC/EC | Strong | না | ZooKeeper-নির্ভর |
| 9 | **ZooKeeper** | CP | PC/EC | Linearizable | না | সবসময় consistent |
| 10 | **Etcd** | CP | PC/EC | Linearizable | না | Raft consensus |
| 11 | **CockroachDB** | CP | PC/EC | Serializable | না | Geo-partitioned survival |
| 12 | **Riak** | AP | PA/EL | Eventual | হ্যাঁ | Dynamo-inspired |
| 13 | **Cosmos DB** | Tunable | Tunable | 5 levels | হ্যাঁ | Strong → Eventual সবই available |
| 14 | **YugabyteDB** | CP | PC/EC | Strong | হ্যাঁ | Raft-based + tunable |
| 15 | **ScyllaDB** | AP | PA/EL | Eventual | হ্যাঁ | Cassandra-compatible |
| 16 | **TiDB** | CP | PC/EC | Snapshot | না | Raft + Percolator |
| 17 | **Neo4j Cluster** | CP | PC/EC | Causal | আংশিক | Core-Read Replica architecture |

---

## ✅ Pros / Cons — প্রতিটি Trade-off-এর সুবিধা ও অসুবিধা

### CP Systems — সুবিধা ও অসুবিধা

| ✅ সুবিধা | ❌ অসুবিধা |
|-----------|-----------|
| ডেটা সবসময় সঠিক ও আপ-টু-ডেট | পার্টিশনে সিস্টেম unavailable হয় |
| Financial transaction-এ আবশ্যক | Latency বেশি (সব replica sync করতে হয়) |
| Race condition / double-spending নেই | Write throughput কম |
| সহজ application logic | Single point of failure (leader) |
| Audit trail নির্ভরযোগ্য | Recovery time বেশি (leader election) |

### AP Systems — সুবিধা ও অসুবিধা

| ✅ সুবিধা | ❌ অসুবিধা |
|-----------|-----------|
| সবসময় available — downtime নেই | Stale data পড়ার সম্ভাবনা |
| Low latency — local node থেকে serve | Conflict resolution জটিল |
| High write throughput | Application logic জটিল হয় |
| Geo-distributed-এ ভালো কাজ করে | Data loss possible (LWW) |
| Horizontal scaling সহজ | Debugging কঠিন |

### CA Systems — সুবিধা ও অসুবিধা

| ✅ সুবিধা | ❌ অসুবিধা |
|-----------|-----------|
| Strong consistency + always available | ডিস্ট্রিবিউটেড নয় — scale হয় না |
| সহজ reasoning | Single point of failure |
| ACID transactions সরাসরি | Vertical scaling-এর limit আছে |
| Mature tooling (PostgreSQL, MySQL) | Network partition = total failure |

---

## ⚠️ Common Misconceptions — সাধারণ ভুল ধারণা

### ভুল ধারণা ১: "CAP-এ তিনটির মধ্যে দুটি বেছে নাও"

**সত্য:** P বাদ দেওয়া সম্ভব নয় (ডিস্ট্রিবিউটেড সিস্টেমে)। আসল সিদ্ধান্ত: **পার্টিশন ঘটলে** C না A? স্বাভাবিক অবস্থায় তিনটিই থাকতে পারে।

### ভুল ধারণা ২: "CP মানে কখনো available নয়"

**সত্য:** CP সিস্টেম স্বাভাবিক অবস্থায় (কোনো পার্টিশন নেই) পুরোপুরি available। শুধু পার্টিশনের সময় minority partition unavailable হয়।

### ভুল ধারণা ৩: "AP মানে কোনো consistency নেই"

**সত্য:** AP সিস্টেমে eventual consistency আছে — একটু দেরিতে হলেও সব replica converge করে। শুধু **পার্টিশনের মুহূর্তে** inconsistency হয়।

### ভুল ধারণা ৪: "Cassandra AP, তাই consistent হতে পারে না"

**সত্য:** Cassandra-র consistency **tunable**। `QUORUM` বা `ALL` consistency level ব্যবহার করলে strong consistency পাওয়া যায়। Default AP, কিন্তু per-query CP সম্ভব।

### ভুল ধারণা ৫: "CAP binary — হয় আছে, নয় নেই"

**সত্য:** CAP একটি **spectrum**। বেশিরভাগ modern সিস্টেম tunable consistency অফার করে। একই ডেটাবেসে বিভিন্ন operation-এ বিভিন্ন consistency level ব্যবহার সম্ভব।

```
  Consistency Spectrum:
  ◄──────────────────────────────────────────────────►
  Eventual    Causal    Read-Your-   Sequential  Strong
  Consistency Consistency  Writes   Consistency  (Linear)

  Cassandra ◄──────────────────────────────────► MongoDB
  (default)  CouchDB   DynamoDB    CockroachDB  ZooKeeper
              Riak     Cosmos DB    YugabyteDB    Etcd
```

### ভুল ধারণা ৬: "Single-node database-এ CAP প্রযোজ্য"

**সত্য:** CAP শুধু **ডিস্ট্রিবিউটেড** সিস্টেমের জন্য। Single-node PostgreSQL-এ কোনো partition হওয়ার সুযোগ নেই, তাই CAP-এর কোনো মানে নেই। সেটি CA, কিন্তু শুধু "ডিস্ট্রিবিউটেড নয়" বলে।

---

## 📋 সারসংক্ষেপ — মূল বিষয়গুলো এক নজরে

```
  ┌──────────────────────────────────────────────────────────┐
  │                    CAP THEOREM SUMMARY                   │
  │                                                          │
  │  ১. ডিস্ট্রিবিউটেড সিস্টেমে P অনিবার্য               │
  │  ২. আসল প্রশ্ন: পার্টিশনে C না A?                     │
  │  ৩. Financial (bKash) → CP (consistency আবশ্যক)        │
  │  ৪. Location/Social (Pathao) → AP (availability আবশ্যক)│
  │  ৫. Modern সিস্টেম tunable — per-operation বেছে নেওয়া │
  │     সম্ভব                                               │
  │  ৬. PACELC — পার্টিশন ছাড়াও Latency vs Consistency    │
  │  ৭. Quorum (W+R>N) — consistency ও availability balance │
  │  ৮. CRDTs / Vector Clocks — AP সিস্টেমে conflict       │
  │     resolution                                          │
  │  ৯. Split-brain → Fencing tokens / STONITH              │
  │ ১০. CAP binary নয় — spectrum                           │
  └──────────────────────────────────────────────────────────┘
```

### সিদ্ধান্ত গাছ — কোন সিস্টেম কখন?

```
  আপনার সিস্টেম কি distributed?
  │
  ├── না → Single-node RDBMS (PostgreSQL/MySQL) ← CA
  │
  └── হ্যাঁ → Network partition ঘটলে কী হবে?
       │
       ├── ভুল ডেটা মারাত্মক? (financial, inventory)
       │   └── হ্যাঁ → CP (MongoDB, CockroachDB, ZooKeeper)
       │       └── Downtime সহ্য করা যায়, ভুল ডেটা সহ্য করা যায় না
       │
       ├── Downtime মারাত্মক? (e-commerce, social, location)
       │   └── হ্যাঁ → AP (Cassandra, DynamoDB, CouchDB)
       │       └── Stale ডেটা সহ্য করা যায়, downtime সহ্য করা যায় না
       │
       └── দুটোই দরকার? → Tunable consistency (Cosmos DB, Cassandra QUORUM)
           └── Operation অনুযায়ী CP/AP switch করো
```

---

> **মনে রাখবেন:** CAP একটি theoretical framework — বাস্তবে প্রতিটি সিস্টেম ডিজাইনের সিদ্ধান্ত নির্ভর করে **আপনার business requirement** এর উপর। bKash-এ ১ টাকাও ভুল হলে চলবে না, কিন্তু Pathao-তে driver-এর location ২ সেকেন্ড পুরোনো হলে কোনো সমস্যা নেই। **"সবকিছু নির্ভর করে" — এটাই distributed systems-এর সত্য।**
