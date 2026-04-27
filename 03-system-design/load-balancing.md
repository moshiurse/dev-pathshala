# ⚖️ লোড ব্যালেন্সিং (Load Balancing) — বিস্তারিত গভীর আলোচনা

## 📌 সংজ্ঞা ও মূল ধারণা

**লোড ব্যালেন্সিং** হলো একটি কৌশল যেখানে ইনকামিং নেটওয়ার্ক ট্রাফিক বা রিকোয়েস্টকে একাধিক সার্ভারের মধ্যে সুষমভাবে বিতরণ করা হয়, যাতে কোনো একটি সার্ভারে অতিরিক্ত চাপ না পড়ে। এটি **high availability**, **fault tolerance**, এবং **horizontal scalability** নিশ্চিত করার মূল স্তম্ভ।

### মূল উদ্দেশ্য

- **ট্রাফিক বিতরণ**: রিকোয়েস্টগুলো সমানভাবে সার্ভারে ভাগ করা
- **High Availability**: একটি সার্ভার মারা গেলেও সার্ভিস চালু থাকবে
- **Horizontal Scaling**: শুধু নতুন সার্ভার যোগ করে capacity বাড়ানো
- **Health Monitoring**: অসুস্থ সার্ভার সনাক্ত করে ট্রাফিক সরানো

### বাস্তব উদাহরণ — bKash

ধরুন, bKash-এ ঈদের দিন প্রতি সেকেন্ডে ৫০,০০০+ ট্রানজেকশন রিকোয়েস্ট আসছে। একটি সার্ভার সর্বোচ্চ ৫,০০০ রিকোয়েস্ট/সেকেন্ড হ্যান্ডেল করতে পারে। লোড ব্যালেন্সার এই ৫০,০০০ রিকোয়েস্টকে ১০টি সার্ভারে ভাগ করে দেয়:

```
                        ঈদের দিন bKash ট্রাফিক
                        ========================

    ৫০,০০০ req/s
         │
         ▼
  ┌──────────────┐
  │  Load        │
  │  Balancer    │
  │  (F5/Nginx)  │
  └──────┬───────┘
         │
    ┌────┼────┬────┬────┬────┬────┬────┬────┬────┐
    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼
  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
  │S1 ││S2 ││S3 ││S4 ││S5 ││S6 ││S7 ││S8 ││S9 ││S10│
  │5K ││5K ││5K ││5K ││5K ││5K ││5K ││5K ││5K ││5K │
  └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘
       প্রতিটি সার্ভার ≈ ৫,০০০ req/s হ্যান্ডেল করে
```

---

## 📊 Load Balancer Architecture

### সামগ্রিক আর্কিটেকচার

```
                    Load Balancer সম্পূর্ণ আর্কিটেকচার
                    =====================================

  ক্লায়েন্ট (ঢাকা/চট্টগ্রাম/সিলেট)
       │
       ▼
  ┌─────────────┐
  │  DNS/GSLB   │ ◄── GeoDNS: নিকটতম ডেটা সেন্টারে রাউট
  └──────┬──────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌────────┐
│ঢাকা DC│ │চট্টগ্রাম│
│        │ │  DC    │
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌────────┐ ┌────────┐
│  L4 LB │ │  L4 LB │  ◄── TCP/UDP লেভেল (NLB)
│ (NLB)  │ │ (NLB)  │
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌────────┐ ┌────────┐
│  L7 LB │ │  L7 LB │  ◄── HTTP/HTTPS লেভেল (ALB/Nginx)
│(Nginx) │ │(Nginx) │
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌────────┐ ┌────────┐
│ App    │ │ App    │  ◄── PHP-FPM / Node.js সার্ভার
│Servers │ │Servers │
└────────┘ └────────┘
```

### L4 (Transport Layer) vs L7 (Application Layer) — মৌলিক পার্থক্য

```
  L4 Load Balancer (TCP/UDP)          L7 Load Balancer (HTTP/HTTPS)
  ============================        ==============================

  ┌──────────────────────┐            ┌──────────────────────┐
  │   TCP SYN প্যাকেট     │            │   HTTP GET /api/pay  │
  │   src: 103.4.x.x    │            │   Host: bkash.com    │
  │   dst: LB_IP:443    │            │   Cookie: sess=abc   │
  │                      │            │   Content-Type: json │
  └──────────┬───────────┘            └──────────┬───────────┘
             │                                   │
             ▼                                   ▼
  ┌──────────────────────┐            ┌──────────────────────┐
  │  শুধু IP + Port দেখে  │            │  URL, Header, Cookie │
  │  সিদ্ধান্ত নেয়        │            │  সব কিছু পার্স করে   │
  │                      │            │  সিদ্ধান্ত নেয়        │
  │  ⚡ অত্যন্ত দ্রুত     │            │  🧠 স্মার্ট রাউটিং    │
  │  কম latency          │            │  বেশি latency        │
  └──────────┬───────────┘            └──────────┬───────────┘
             │                                   │
             ▼                                   ▼
     সার্ভার নির্বাচন               URL ভিত্তিক রাউটিং:
     (IP hash / RR)                /api/* → API সার্ভার
                                   /static/* → CDN/Static
                                   /ws/* → WebSocket সার্ভার
```

---

## 💻 Algorithms Deep Dive

### ১. Round Robin (রাউন্ড রবিন)

সবচেয়ে সহজ অ্যালগরিদম — পালা করে প্রতিটি সার্ভারে রিকোয়েস্ট পাঠায়।

```
  Round Robin ফ্লো
  ================

  Request 1 ──→ Server A
  Request 2 ──→ Server B
  Request 3 ──→ Server C
  Request 4 ──→ Server A  ◄── আবার শুরু
  Request 5 ──→ Server B
  Request 6 ──→ Server C
       ...         ...

  ┌───┐   ┌───┐   ┌───┐
  │ A │   │ B │   │ C │
  │ 2 │   │ 2 │   │ 2 │  ◄── সমান বিতরণ
  └───┘   └───┘   └───┘
```

**PHP Implementation:**

```php
<?php

class RoundRobinBalancer
{
    private array $servers;
    private int $currentIndex = 0;

    public function __construct(array $servers)
    {
        $this->servers = $servers;
    }

    public function getNextServer(): string
    {
        $server = $this->servers[$this->currentIndex];
        $this->currentIndex = ($this->currentIndex + 1) % count($this->servers);
        return $server;
    }

    public function distribute(int $requestCount): array
    {
        $distribution = array_fill_keys($this->servers, 0);
        for ($i = 0; $i < $requestCount; $i++) {
            $server = $this->getNextServer();
            $distribution[$server]++;
        }
        return $distribution;
    }
}

// bKash পেমেন্ট সার্ভার
$balancer = new RoundRobinBalancer([
    'dhaka-pay-01.bkash.internal',
    'dhaka-pay-02.bkash.internal',
    'dhaka-pay-03.bkash.internal',
]);

// ১০,০০০ ট্রানজেকশন বিতরণ
$result = $balancer->distribute(10000);
print_r($result);
// প্রতিটি সার্ভারে ≈ ৩,৩৩৩ রিকোয়েস্ট
```

**JavaScript Implementation:**

```javascript
class RoundRobinBalancer {
  #servers;
  #currentIndex = 0;

  constructor(servers) {
    this.#servers = [...servers];
  }

  getNextServer() {
    const server = this.#servers[this.#currentIndex];
    this.#currentIndex = (this.#currentIndex + 1) % this.#servers.length;
    return server;
  }
}

const balancer = new RoundRobinBalancer([
  'dhaka-pay-01.bkash.internal',
  'dhaka-pay-02.bkash.internal',
  'dhaka-pay-03.bkash.internal',
]);

for (let i = 0; i < 9; i++) {
  console.log(`Request ${i + 1} → ${balancer.getNextServer()}`);
}
```

**ব্যবহার ক্ষেত্র**: সব সার্ভার সমান ক্ষমতার হলে। Stateless API সার্ভারে সবচেয়ে কার্যকর।

---

### ২. Weighted Round Robin (ওয়েটেড রাউন্ড রবিন)

সার্ভারের ক্ষমতা অনুসারে ওজন (weight) বরাদ্দ করা হয়। বেশি ক্ষমতার সার্ভারে বেশি রিকোয়েস্ট যায়।

```
  Weighted Round Robin
  ====================

  Server A (weight=5) ████████████████████  ←── ১৬ CPU core, ৬৪GB RAM
  Server B (weight=3) ████████████          ←── ৮ CPU core, ৩২GB RAM
  Server C (weight=2) ████████              ←── ৪ CPU core, ১৬GB RAM

  ১০ রিকোয়েস্টের মধ্যে:
    A পাবে ৫টি
    B পাবে ৩টি
    C পাবে ২টি
```

```php
<?php

class WeightedRoundRobinBalancer
{
    private array $servers;      // ['server' => ..., 'weight' => ...]
    private int $currentIndex = -1;
    private int $currentWeight = 0;

    public function __construct(array $servers)
    {
        $this->servers = $servers;
    }

    private function maxWeight(): int
    {
        return max(array_column($this->servers, 'weight'));
    }

    private function gcdWeights(): int
    {
        $weights = array_column($this->servers, 'weight');
        $result = $weights[0];
        for ($i = 1; $i < count($weights); $i++) {
            $result = $this->gcd($result, $weights[$i]);
        }
        return $result;
    }

    private function gcd(int $a, int $b): int
    {
        return $b === 0 ? $a : $this->gcd($b, $a % $b);
    }

    public function getNextServer(): string
    {
        $n = count($this->servers);
        while (true) {
            $this->currentIndex = ($this->currentIndex + 1) % $n;
            if ($this->currentIndex === 0) {
                $this->currentWeight -= $this->gcdWeights();
                if ($this->currentWeight <= 0) {
                    $this->currentWeight = $this->maxWeight();
                }
            }
            if ($this->servers[$this->currentIndex]['weight'] >= $this->currentWeight) {
                return $this->servers[$this->currentIndex]['server'];
            }
        }
    }
}

$balancer = new WeightedRoundRobinBalancer([
    ['server' => 'dhaka-premium-01', 'weight' => 5],   // ১৬ core
    ['server' => 'dhaka-standard-02', 'weight' => 3],   // ৮ core
    ['server' => 'ctg-standard-03', 'weight' => 2],     // ৪ core (চট্টগ্রাম)
]);
```

---

### ৩. Least Connections (লিস্ট কানেকশনস)

যে সার্ভারে বর্তমানে সবচেয়ে কম সংযোগ আছে, সেখানে পরবর্তী রিকোয়েস্ট পাঠায়। দীর্ঘ সময়ের রিকোয়েস্ট (যেমন WebSocket, ফাইল আপলোড) এর জন্য আদর্শ।

```
  Least Connections ফ্লো
  =======================

  বর্তমান অবস্থা:
  ┌──────────┬──────────────┐
  │ Server   │ Active Conn  │
  ├──────────┼──────────────┤
  │ A        │ ████ (12)    │
  │ B        │ ██ (6)       │  ◄── সবচেয়ে কম
  │ C        │ ███ (9)      │
  └──────────┴──────────────┘

  নতুন রিকোয়েস্ট → Server B তে যাবে (6 active conn)
```

```javascript
class LeastConnectionsBalancer {
  #servers; // Map<string, { connections: number, maxConn: number }>

  constructor(servers) {
    this.#servers = new Map();
    servers.forEach(s => {
      this.#servers.set(s.address, { connections: 0, maxConn: s.maxConn || 1000 });
    });
  }

  getNextServer() {
    let minConn = Infinity;
    let selected = null;

    for (const [address, info] of this.#servers) {
      if (info.connections < minConn && info.connections < info.maxConn) {
        minConn = info.connections;
        selected = address;
      }
    }

    if (!selected) throw new Error('সব সার্ভার পূর্ণ!');

    this.#servers.get(selected).connections++;
    return selected;
  }

  releaseConnection(address) {
    const info = this.#servers.get(address);
    if (info && info.connections > 0) {
      info.connections--;
    }
  }

  getStats() {
    const stats = {};
    for (const [addr, info] of this.#servers) {
      stats[addr] = info.connections;
    }
    return stats;
  }
}

// bKash API Gateway
const lb = new LeastConnectionsBalancer([
  { address: 'api-01.bkash.internal', maxConn: 5000 },
  { address: 'api-02.bkash.internal', maxConn: 5000 },
  { address: 'api-03.bkash.internal', maxConn: 3000 },
]);

// সিমুলেশন: রিকোয়েস্ট আসা ও শেষ হওয়া
const server1 = lb.getNextServer();
console.log(`Routing to: ${server1}`);
// ... কাজ শেষে
lb.releaseConnection(server1);
```

---

### ৪. Weighted Least Connections

Least Connections + Weight মিলিয়ে — সূত্র: `connections / weight` অনুপাত যার সবচেয়ে কম, সে রিকোয়েস্ট পায়।

```
  Weighted Least Connections
  ==========================

  ┌──────────┬────────┬──────┬───────────────┐
  │ Server   │ Conn   │ Wt   │ Conn/Wt       │
  ├──────────┼────────┼──────┼───────────────┤
  │ A        │ 10     │ 5    │ 2.0           │
  │ B        │ 8      │ 3    │ 2.67          │
  │ C        │ 3      │ 2    │ 1.5 ◄── জিতবে │
  └──────────┴────────┴──────┴───────────────┘

  নতুন রিকোয়েস্ট → Server C (সবচেয়ে কম অনুপাত)
```

```php
<?php

class WeightedLeastConnBalancer
{
    private array $servers;

    public function __construct(array $servers)
    {
        $this->servers = [];
        foreach ($servers as $s) {
            $this->servers[$s['address']] = [
                'weight' => $s['weight'],
                'connections' => 0,
            ];
        }
    }

    public function getNextServer(): string
    {
        $minRatio = PHP_FLOAT_MAX;
        $selected = null;

        foreach ($this->servers as $addr => $info) {
            $ratio = $info['connections'] / $info['weight'];
            if ($ratio < $minRatio) {
                $minRatio = $ratio;
                $selected = $addr;
            }
        }

        $this->servers[$selected]['connections']++;
        return $selected;
    }

    public function release(string $address): void
    {
        if (isset($this->servers[$address]) && $this->servers[$address]['connections'] > 0) {
            $this->servers[$address]['connections']--;
        }
    }
}

$lb = new WeightedLeastConnBalancer([
    ['address' => 'dhaka-big-01', 'weight' => 5],
    ['address' => 'dhaka-med-02', 'weight' => 3],
    ['address' => 'ctg-small-03', 'weight' => 2],
]);
```

---

### ৫. IP Hash (আইপি হ্যাশ)

ক্লায়েন্টের IP অ্যাড্রেস হ্যাশ করে একটি নির্দিষ্ট সার্ভারে ম্যাপ করে। একই ক্লায়েন্ট সবসময় একই সার্ভারে যায় — **session affinity** নিশ্চিত করে।

```
  IP Hash ফ্লো
  =============

  Client 103.4.146.52  ──hash──→ hash % 3 = 0 ──→ Server A
  Client 27.147.198.3  ──hash──→ hash % 3 = 2 ──→ Server C
  Client 103.4.146.52  ──hash──→ hash % 3 = 0 ──→ Server A (একই!)
  Client 58.97.201.88  ──hash──→ hash % 3 = 1 ──→ Server B

  ⚠️ সমস্যা: সার্ভার যোগ/বাদ দিলে সব ম্যাপিং বদলে যায়!
```

```php
<?php

class IpHashBalancer
{
    private array $servers;

    public function __construct(array $servers)
    {
        $this->servers = array_values($servers);
    }

    public function getServer(string $clientIp): string
    {
        $hash = crc32($clientIp);
        $index = abs($hash) % count($this->servers);
        return $this->servers[$index];
    }
}

$lb = new IpHashBalancer([
    'dhaka-01.bkash.internal',
    'dhaka-02.bkash.internal',
    'ctg-01.bkash.internal',
]);

// বাংলাদেশের বিভিন্ন ISP থেকে আসা রিকোয়েস্ট
echo $lb->getServer('103.4.146.52') . "\n";   // Grameenphone
echo $lb->getServer('27.147.198.3') . "\n";   // Link3
echo $lb->getServer('103.4.146.52') . "\n";   // একই IP → একই সার্ভার
```

---

### ৬. Consistent Hashing (কনসিস্টেন্ট হ্যাশিং)

IP Hash-এর সমস্যা সমাধান করে। সার্ভার যোগ/বাদ হলে শুধু কিছু key পুনঃবিতরণ হয়, সব নয়। **ক্যাশ সার্ভার** (Redis/Memcached) এর জন্য অপরিহার্য।

```
  Consistent Hashing Ring
  ========================

              0°
              │
        S_A ──┤── 45°
              │
     330°─────┤─── 90° ── S_B
              │
              │
    270°──────┤───── 135°
       │      │
     S_D      │
    240°──────┤───── 180°
              │         │
              │       S_C
    210°──────┘

  Key "user:1001" → hash = 120° → ঘড়ির কাঁটায় পরের সার্ভার S_C (180°)
  Key "txn:5502"  → hash = 300° → ঘড়ির কাঁটায় পরের সার্ভার S_A (330°→45°)

  ✅ S_B মারা গেলে শুধু S_B এর key গুলো S_C তে যাবে
     বাকি সব অপরিবর্তিত!
```

```javascript
const crypto = require('crypto');

class ConsistentHashRing {
  #ring = new Map();     // hash → server
  #sortedKeys = [];
  #replicas;

  constructor(servers, replicas = 150) {
    this.#replicas = replicas;
    servers.forEach(server => this.addServer(server));
  }

  #hash(key) {
    return parseInt(
      crypto.createHash('md5').update(key).digest('hex').substring(0, 8),
      16
    );
  }

  addServer(server) {
    for (let i = 0; i < this.#replicas; i++) {
      const hash = this.#hash(`${server}:${i}`);
      this.#ring.set(hash, server);
      this.#sortedKeys.push(hash);
    }
    this.#sortedKeys.sort((a, b) => a - b);
  }

  removeServer(server) {
    for (let i = 0; i < this.#replicas; i++) {
      const hash = this.#hash(`${server}:${i}`);
      this.#ring.delete(hash);
      this.#sortedKeys = this.#sortedKeys.filter(k => k !== hash);
    }
  }

  getServer(key) {
    if (this.#ring.size === 0) throw new Error('কোনো সার্ভার নেই!');
    const hash = this.#hash(key);
    for (const ringKey of this.#sortedKeys) {
      if (hash <= ringKey) return this.#ring.get(ringKey);
    }
    return this.#ring.get(this.#sortedKeys[0]); // wrap around
  }
}

// bKash ক্যাশ ক্লাস্টার
const ring = new ConsistentHashRing([
  'redis-dhaka-01', 'redis-dhaka-02',
  'redis-ctg-01', 'redis-ctg-02',
]);

console.log(ring.getServer('user:01912345678'));  // ফোন নম্বর দিয়ে ক্যাশ
console.log(ring.getServer('txn:TXN20240101001'));

// নতুন সার্ভার যোগ — মাত্র ~1/N key পুনঃবিতরণ হবে
ring.addServer('redis-dhaka-03');
```

---

### ৭. Random (র‍্যান্ডম)

সম্পূর্ণ এলোমেলোভাবে সার্ভার নির্বাচন। সহজ কিন্তু প্রোডাকশনে কম ব্যবহৃত। **Power of Two Choices** ভ্যারিয়েন্ট বেশি কার্যকর — দুটি এলোমেলো সার্ভার থেকে কম লোডের সার্ভারটি নির্বাচন।

```
  Random vs Power of Two Choices
  ==============================

  Random:                           Power of Two:
  ──────                            ─────────────
  নতুন req → random(servers)        নতুন req → pick 2 random
       → সেই সার্ভারে পাঠাও             → কম conn ওয়ালাটা বেছে নাও

  সমস্যা: দুর্ভাগ্যক্রমে               সুবিধা: প্রায় optimal
  একই সার্ভার বারবার পেতে পারে         বিতরণ, O(1) complexity
```

```php
<?php

class PowerOfTwoChoicesBalancer
{
    private array $servers;

    public function __construct(array $servers)
    {
        $this->servers = [];
        foreach ($servers as $addr) {
            $this->servers[$addr] = 0; // active connections
        }
    }

    public function getNextServer(): string
    {
        $addresses = array_keys($this->servers);

        // দুটি এলোমেলো সার্ভার বাছাই
        $idx1 = random_int(0, count($addresses) - 1);
        do {
            $idx2 = random_int(0, count($addresses) - 1);
        } while ($idx2 === $idx1 && count($addresses) > 1);

        $s1 = $addresses[$idx1];
        $s2 = $addresses[$idx2];

        // কম কানেকশনের সার্ভার নির্বাচন
        $selected = $this->servers[$s1] <= $this->servers[$s2] ? $s1 : $s2;
        $this->servers[$selected]++;
        return $selected;
    }

    public function release(string $addr): void
    {
        if (isset($this->servers[$addr])) {
            $this->servers[$addr] = max(0, $this->servers[$addr] - 1);
        }
    }
}
```

---

### ৮. Resource Based (রিসোর্স ভিত্তিক)

সার্ভারের বর্তমান CPU, মেমোরি, ডিস্ক ব্যবহার দেখে সিদ্ধান্ত নেয়। সবচেয়ে বুদ্ধিমান পদ্ধতি কিন্তু বাস্তবায়ন জটিল। সার্ভারে একটি এজেন্ট চলে যা health তথ্য পাঠায়।

```
  Resource Based ফ্লো
  ====================

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Server A │     │ Server B │     │ Server C │
  │ CPU: 85% │     │ CPU: 30% │     │ CPU: 60% │
  │ RAM: 70% │     │ RAM: 45% │     │ RAM: 55% │
  │ Disk I/O │     │ Disk I/O │     │ Disk I/O │
  │  : HIGH  │     │  : LOW   │     │  : MED   │
  └────┬─────┘     └────┬─────┘     └────┬─────┘
       │                │                │
       └───────┬────────┴────────┬───────┘
               │  Health Report  │
               ▼                 ▼
        ┌──────────────────────────┐
        │       Load Balancer      │
        │  Score A = 0.85*0.4 +    │
        │          0.70*0.3 +      │
        │          0.90*0.3 = 0.82 │
        │  Score B = 0.38 ◄── জিতবে│
        │  Score C = 0.58          │
        └──────────────────────────┘
```

```javascript
class ResourceBasedBalancer {
  #servers = new Map();
  #weights = { cpu: 0.4, memory: 0.3, diskIO: 0.3 };

  addServer(address, metrics) {
    this.#servers.set(address, metrics);
  }

  updateMetrics(address, metrics) {
    this.#servers.set(address, { ...this.#servers.get(address), ...metrics });
  }

  #calculateScore(metrics) {
    return (
      (metrics.cpuUsage / 100) * this.#weights.cpu +
      (metrics.memoryUsage / 100) * this.#weights.memory +
      (metrics.diskIO / 100) * this.#weights.diskIO
    );
  }

  getNextServer() {
    let minScore = Infinity;
    let selected = null;

    for (const [addr, metrics] of this.#servers) {
      if (!metrics.healthy) continue;
      const score = this.#calculateScore(metrics);
      if (score < minScore) {
        minScore = score;
        selected = addr;
      }
    }

    if (!selected) throw new Error('কোনো সুস্থ সার্ভার নেই!');
    return selected;
  }
}

const lb = new ResourceBasedBalancer();
lb.addServer('dhaka-01', { cpuUsage: 85, memoryUsage: 70, diskIO: 90, healthy: true });
lb.addServer('dhaka-02', { cpuUsage: 30, memoryUsage: 45, diskIO: 20, healthy: true });
lb.addServer('ctg-01',   { cpuUsage: 60, memoryUsage: 55, diskIO: 50, healthy: true });

console.log(lb.getNextServer()); // dhaka-02 (সবচেয়ে কম স্কোর)
```

---

## 🔥 Advanced Topics

### ১. L4 vs L7 Load Balancing — বিস্তারিত তুলনা

```
  ┌────────────────────┬──────────────────────┬──────────────────────┐
  │ বৈশিষ্ট্য           │ L4 (Transport)       │ L7 (Application)     │
  ├────────────────────┼──────────────────────┼──────────────────────┤
  │ OSI Layer          │ Layer 4 (TCP/UDP)    │ Layer 7 (HTTP/S)     │
  │ পার্স করে           │ IP, Port             │ URL, Header, Cookie  │
  │ গতি                │ ⚡ অত্যন্ত দ্রুত       │ 🐢 তুলনামূলক ধীর     │
  │ SSL Termination    │ ❌ না                 │ ✅ হ্যাঁ               │
  │ Content Routing    │ ❌ না                 │ ✅ হ্যাঁ               │
  │ WebSocket          │ ✅ Pass-through       │ ✅ Aware              │
  │ DDoS Protection    │ ✅ ভালো               │ ⚠️ শুধু App-level     │
  │ Connection Reuse   │ ❌ না                 │ ✅ Multiplexing       │
  │ Caching            │ ❌ না                 │ ✅ Response cache     │
  │ AWS সমতুল্য         │ NLB                  │ ALB                  │
  │ ব্যবহার ক্ষেত্র      │ TCP proxy, Gaming,   │ REST API, gRPC,      │
  │                    │ Database, IoT        │ Microservices        │
  │ বাংলাদেশ উদাহরণ     │ bKash USSD gateway   │ bKash REST API       │
  └────────────────────┴──────────────────────┴──────────────────────┘
```

```
  প্যাকেট প্রসেসিং তুলনা
  ========================

  L4:
  ┌─────────────────┐     ┌─────────────────┐
  │ TCP SYN from    │     │ Forward entire  │
  │ 103.4.x.x:4521 │ ──→ │ TCP stream to   │
  │ to LB:443      │     │ backend:8080    │
  └─────────────────┘     └─────────────────┘
  প্যাকেটের ভেতরে কী আছে তা দেখে না, শুধু ফরোয়ার্ড করে

  L7:
  ┌─────────────────┐     ┌──────────────────┐     ┌──────────────┐
  │ HTTP Request    │     │ Parse:           │     │ Route to:    │
  │ GET /api/pay    │ ──→ │ URL: /api/pay    │ ──→ │ pay-service  │
  │ Host: bkash.com │     │ Header analysis  │     │ :8081        │
  │ Auth: Bearer... │     │ Auth validation  │     │              │
  └─────────────────┘     └──────────────────┘     └──────────────┘
  সম্পূর্ণ HTTP রিকোয়েস্ট পার্স করে, বুঝে, তারপর সিদ্ধান্ত নেয়
```

---

### ২. Nginx Configuration — PHP-FPM + Node.js

```nginx
# /etc/nginx/nginx.conf — bKash-style প্রোডাকশন কনফিগারেশন

worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 16384;
    multi_accept on;
    use epoll;
}

http {
    # === বেসিক অপটিমাইজেশন ===
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;

    # === আপস্ট্রিম: PHP-FPM সার্ভার (পেমেন্ট প্রসেসিং) ===
    upstream php_backend {
        least_conn;

        server dhaka-php-01:9000 weight=5 max_fails=3 fail_timeout=30s;
        server dhaka-php-02:9000 weight=5 max_fails=3 fail_timeout=30s;
        server dhaka-php-03:9000 weight=3 max_fails=3 fail_timeout=30s;
        server ctg-php-01:9000   weight=2 max_fails=3 fail_timeout=30s backup;

        # কানেকশন পুলিং
        keepalive 32;
    }

    # === আপস্ট্রিম: Node.js সার্ভার (রিয়েলটাইম নোটিফিকেশন) ===
    upstream node_backend {
        ip_hash;  # WebSocket-এর জন্য session stickiness

        server dhaka-node-01:3000 max_fails=2 fail_timeout=10s;
        server dhaka-node-02:3000 max_fails=2 fail_timeout=10s;
        server dhaka-node-03:3000 max_fails=2 fail_timeout=10s;

        keepalive 64;
    }

    # === আপস্ট্রিম: Static Asset সার্ভার ===
    upstream static_backend {
        server dhaka-static-01:8080;
        server dhaka-static-02:8080;
    }

    # === Rate Limiting ===
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    limit_req_zone $binary_remote_addr zone=payment_limit:10m rate=10r/s;

    # === SSL/TLS সেটিংস ===
    ssl_certificate     /etc/ssl/certs/bkash.com.pem;
    ssl_certificate_key /etc/ssl/private/bkash.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:20m;
    ssl_session_timeout 4h;

    # === মূল সার্ভার ব্লক ===
    server {
        listen 443 ssl http2;
        server_name api.bkash.com;

        # Health check endpoint
        location /health {
            access_log off;
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }

        # PHP API রাউট (পেমেন্ট, ব্যালেন্স, ট্রানজেকশন)
        location ~ ^/api/(payment|balance|transaction) {
            limit_req zone=payment_limit burst=20 nodelay;

            fastcgi_pass php_backend;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME /var/www/api/public/index.php;

            fastcgi_connect_timeout 5s;
            fastcgi_send_timeout 30s;
            fastcgi_read_timeout 30s;
            fastcgi_buffer_size 16k;
            fastcgi_buffers 16 16k;
        }

        # Node.js রিয়েলটাইম (নোটিফিকেশন, WebSocket)
        location /ws/ {
            proxy_pass http://node_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 86400s;  # WebSocket দীর্ঘক্ষণ চালু
        }

        location /api/notifications {
            limit_req zone=api_limit burst=50 nodelay;

            proxy_pass http://node_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Static ফাইল
        location /static/ {
            proxy_pass http://static_backend;
            proxy_cache_valid 200 1d;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }

    # HTTP → HTTPS রিডাইরেক্ট
    server {
        listen 80;
        server_name api.bkash.com;
        return 301 https://$host$request_uri;
    }
}
```

---

### ৩. HAProxy Configuration

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log /dev/log local0
    stats socket /var/run/haproxy.sock mode 660 level admin
    ssl-default-bind-ciphers HIGH:!aNULL:!MD5
    ssl-default-bind-options ssl-min-ver TLSv1.2
    tune.ssl.default-dh-param 2048

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    timeout http-request 10s
    timeout http-keep-alive 15s
    retries 3

# === স্ট্যাটিস্টিক্স ড্যাশবোর্ড ===
listen stats
    bind *:8404
    stats enable
    stats uri /haproxy-stats
    stats refresh 10s
    stats admin if TRUE
    stats auth admin:SecurePass123

# === ফ্রন্টএন্ড: HTTPS টার্মিনেশন ===
frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/bkash.pem
    bind *:80
    redirect scheme https if !{ ssl_fc }

    # ACL ভিত্তিক রাউটিং
    acl is_payment path_beg /api/payment
    acl is_websocket hdr(Upgrade) -i websocket
    acl is_static path_beg /static

    # ব্যাকএন্ড নির্বাচন
    use_backend payment_servers if is_payment
    use_backend websocket_servers if is_websocket
    use_backend static_servers if is_static
    default_backend api_servers

# === ব্যাকএন্ড: পেমেন্ট সার্ভার (PHP) ===
backend payment_servers
    balance leastconn
    option httpchk GET /health
    http-check expect status 200

    # Sticky session (cookie ভিত্তিক)
    cookie SERVERID insert indirect nocache

    server dhaka-pay-01 10.0.1.10:8080 weight 5 check inter 3s fall 3 rise 2 cookie s1
    server dhaka-pay-02 10.0.1.11:8080 weight 5 check inter 3s fall 3 rise 2 cookie s2
    server dhaka-pay-03 10.0.1.12:8080 weight 3 check inter 3s fall 3 rise 2 cookie s3
    server ctg-pay-01   10.0.2.10:8080 weight 2 check inter 5s fall 3 rise 2 cookie s4 backup

# === ব্যাকএন্ড: API সার্ভার (Node.js) ===
backend api_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server dhaka-api-01 10.0.1.20:3000 check inter 3s fall 3 rise 2
    server dhaka-api-02 10.0.1.21:3000 check inter 3s fall 3 rise 2
    server dhaka-api-03 10.0.1.22:3000 check inter 3s fall 3 rise 2

# === ব্যাকএন্ড: WebSocket সার্ভার ===
backend websocket_servers
    balance source
    option httpchk GET /health
    timeout server 86400s
    timeout tunnel 86400s

    server dhaka-ws-01 10.0.1.30:3000 check inter 5s
    server dhaka-ws-02 10.0.1.31:3000 check inter 5s
```

---

### ৪. AWS ALB / NLB / CLB তুলনা

```
  AWS Load Balancer তুলনা
  ========================

  ┌────────────────┬──────────────┬──────────────┬──────────────┐
  │ বৈশিষ্ট্য       │ ALB          │ NLB          │ CLB (Legacy) │
  ├────────────────┼──────────────┼──────────────┼──────────────┤
  │ Layer          │ L7 (HTTP)    │ L4 (TCP/UDP) │ L4 + L7      │
  │ প্রোটোকল       │ HTTP, HTTPS, │ TCP, UDP,    │ HTTP, HTTPS, │
  │                │ gRPC, WS     │ TLS          │ TCP, SSL     │
  │ Performance    │ ভালো          │ অসাধারণ       │ মাঝারি        │
  │ Static IP      │ ❌            │ ✅            │ ❌            │
  │ Path routing   │ ✅            │ ❌            │ ❌            │
  │ Host routing   │ ✅            │ ❌            │ ❌            │
  │ WAF সাপোর্ট     │ ✅            │ ❌            │ ❌            │
  │ Target types   │ IP, Instance,│ IP, Instance,│ Instance     │
  │                │ Lambda       │ ALB          │              │
  │ মূল্য (ঢাকা)    │ $0.0225/hr   │ $0.0225/hr   │ $0.025/hr    │
  │ খরচ মডেল       │ LCU          │ NLCU         │ Fixed        │
  └────────────────┴──────────────┴──────────────┴──────────────┘

  কখন কোনটি ব্যবহার করবেন:
  ─────────────────────────

  ALB → REST API, gRPC, মাইক্রোসার্ভিস, URL-based রাউটিং
        উদাহরণ: bKash app এর REST API

  NLB → Ultra-low latency, TCP/UDP, Static IP দরকার, gaming
        উদাহরণ: bKash-এর USSD gateway, real-time পেমেন্ট processing

  CLB → ❌ নতুন প্রজেক্টে ব্যবহার করবেন না (deprecated)
```

```
  AWS-এ bKash-style আর্কিটেকচার
  ================================

                  Route 53 (GeoDNS)
                       │
              ┌────────┴────────┐
              ▼                 ▼
        ┌──────────┐     ┌──────────┐
        │ ap-south │     │ ap-south │
        │ east-1a  │     │ east-1b  │
        │ (ঢাকা)   │     │ (চট্টগ্রাম) │
        └────┬─────┘     └────┬─────┘
             │                │
             ▼                ▼
        ┌─────────┐     ┌─────────┐
        │   NLB   │     │   NLB   │   ◄── TCP/TLS termination
        └────┬────┘     └────┬────┘
             │                │
             ▼                ▼
        ┌─────────┐     ┌─────────┐
        │   ALB   │     │   ALB   │   ◄── Path-based routing
        └────┬────┘     └────┬────┘
             │                │
     ┌───────┼───────┐       │
     ▼       ▼       ▼       ▼
  ┌──────┐┌──────┐┌──────┐┌──────┐
  │ ECS  ││ ECS  ││ ECS  ││ ECS  │
  │ Pay  ││ Notif││ Auth ││ Pay  │
  └──────┘└──────┘└──────┘└──────┘
```

---

### ৫. Health Checks — Active ও Passive

**Active Health Check**: লোড ব্যালেন্সার নিজে নির্দিষ্ট interval-এ সার্ভারকে চেক করে।
**Passive Health Check**: সার্ভারের আসল রেসপন্স দেখে সিদ্ধান্ত নেয় (যেমন বারবার 5xx ত্রুটি)।

```
  Active Health Check ফ্লো
  =========================

  Load Balancer                          Backend Servers
       │                                      │
       │──── GET /health ────────────────────→ │ Server A
       │◄─── 200 OK {"status":"healthy"} ──── │ ✅
       │                                      │
       │──── GET /health ────────────────────→ │ Server B
       │◄─── 503 Service Unavailable ──────── │ ❌ (fail_count++)
       │                                      │
       │──── GET /health ────────────────────→ │ Server C
       │◄─── Connection Timeout ───────────── │ ❌❌ (fail_count++)
       │                                      │
       │  3 consecutive fails → Server removed│
       │  2 consecutive pass  → Server added  │


  Passive Health Check ফ্লো
  ==========================

  Client → LB → Server B → 502 Bad Gateway
  Client → LB → Server B → 502 Bad Gateway
  Client → LB → Server B → 502 Bad Gateway
                    │
                    ▼
          ৩টি 5xx পরপর → Server B কে
          pool থেকে সরিয়ে দাও
          30s পরে আবার চেষ্টা করো
```

**PHP Health Check Endpoint:**

```php
<?php
// /health.php — প্রোডাকশন-গ্রেড health check

header('Content-Type: application/json');

$checks = [];
$healthy = true;

// ১. ডাটাবেস চেক
try {
    $pdo = new PDO(
        'mysql:host=db-master.bkash.internal;dbname=payments',
        'healthcheck',
        'readonly_pass',
        [PDO::ATTR_TIMEOUT => 2]
    );
    $pdo->query('SELECT 1');
    $checks['database'] = ['status' => 'ok', 'latency_ms' => 0];
} catch (PDOException $e) {
    $checks['database'] = ['status' => 'error', 'message' => 'DB unreachable'];
    $healthy = false;
}

// ২. Redis চেক
try {
    $redis = new Redis();
    $redis->connect('redis-master.bkash.internal', 6379, 1);
    $redis->ping();
    $checks['redis'] = ['status' => 'ok'];
} catch (Exception $e) {
    $checks['redis'] = ['status' => 'error'];
    $healthy = false;
}

// ৩. ডিস্ক স্পেস চেক
$freeSpace = disk_free_space('/');
$totalSpace = disk_total_space('/');
$usedPercent = (1 - $freeSpace / $totalSpace) * 100;
$checks['disk'] = [
    'status' => $usedPercent < 90 ? 'ok' : 'warning',
    'used_percent' => round($usedPercent, 1),
];
if ($usedPercent > 95) $healthy = false;

// ৪. মেমোরি চেক
$memUsage = memory_get_usage(true);
$memLimit = ini_get('memory_limit');
$checks['memory'] = [
    'status' => 'ok',
    'usage_mb' => round($memUsage / 1024 / 1024, 1),
];

http_response_code($healthy ? 200 : 503);
echo json_encode([
    'healthy' => $healthy,
    'timestamp' => date('c'),
    'server' => gethostname(),
    'checks' => $checks,
], JSON_PRETTY_PRINT);
```

**Node.js Health Check:**

```javascript
// healthCheck.js
const http = require('http');
const { createClient } = require('redis');
const mysql = require('mysql2/promise');

async function checkHealth() {
  const checks = {};
  let healthy = true;

  // ডাটাবেস চেক
  try {
    const conn = await mysql.createConnection({
      host: 'db-master.bkash.internal',
      user: 'healthcheck',
      password: 'readonly_pass',
      connectTimeout: 2000,
    });
    await conn.query('SELECT 1');
    await conn.end();
    checks.database = { status: 'ok' };
  } catch (err) {
    checks.database = { status: 'error', message: err.message };
    healthy = false;
  }

  // Redis চেক
  try {
    const redis = createClient({ url: 'redis://redis-master.bkash.internal:6379' });
    await redis.connect();
    await redis.ping();
    await redis.disconnect();
    checks.redis = { status: 'ok' };
  } catch (err) {
    checks.redis = { status: 'error', message: err.message };
    healthy = false;
  }

  return { healthy, checks, timestamp: new Date().toISOString() };
}

// Express/Fastify route হিসেবে ব্যবহার
module.exports = async (req, res) => {
  const result = await checkHealth();
  res.status(result.healthy ? 200 : 503).json(result);
};
```

---

### ৬. Session Persistence (Sticky Sessions)

**সমস্যা**: HTTP stateless — ইউজারের session কোন সার্ভারে সেটা LB জানে না। প্রতিবার ভিন্ন সার্ভারে গেলে session হারিয়ে যাবে।

```
  Sticky Session সমস্যা ও সমাধান
  ================================

  ❌ সমস্যা (sticky session ছাড়া):
  ──────────────────────────────

  User Login → Server A (session তৈরি)
  Next Request → Server B (session নেই! → 401 Unauthorized)


  ⚠️ আংশিক সমাধান (sticky session):
  ──────────────────────────────────

  User Login → Server A (cookie: SERVERID=s1)
  Next Request → LB cookie দেখে → Server A তে পাঠায়
  ✅ কাজ করে কিন্তু...

  সমস্যা:
  • Server A মারা গেলে সব session হারাবে
  • Uneven distribution — জনপ্রিয় সার্ভারে বেশি load
  • Auto scaling-এ নতুন সার্ভারে কেউ যায় না


  ✅ সর্বোত্তম সমাধান (Centralized Session — Redis):
  ──────────────────────────────────────────────────

  ┌────────┐     ┌─────────┐     ┌──────────┐
  │ Client │ ──→ │   LB    │ ──→ │ Server A │──┐
  └────────┘     │(কোনো    │     │          │  │
                 │server-এ │     └──────────┘  │
                 │পাঠাতে   │     ┌──────────┐  │   ┌─────────┐
                 │পারে)    │ ──→ │ Server B │──┼──→│  Redis  │
                 │         │     │          │  │   │ Session │
                 │         │     └──────────┘  │   │  Store  │
                 │         │     ┌──────────┐  │   └─────────┘
                 │         │ ──→ │ Server C │──┘
                 └─────────┘     └──────────┘

  যেকোনো সার্ভারেই যাক, Redis থেকে session পড়বে ✅
```

**PHP — Redis Session Store:**

```php
<?php
// php.ini অথবা runtime সেটিং
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis-session.bkash.internal:6379?auth=secretpass&database=1');
ini_set('session.gc_maxlifetime', 1800); // ৩০ মিনিট

session_start();

// এখন session স্বয়ংক্রিয়ভাবে Redis-এ সংরক্ষিত হবে
$_SESSION['user_id'] = 'USR_01712345678';
$_SESSION['last_transaction'] = 'TXN_20240101_001';

// যেকোনো সার্ভার থেকে এই session পড়া যাবে
echo "User: " . $_SESSION['user_id'];
```

```javascript
// Node.js — Redis session (express-session)
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');
const express = require('express');

const app = express();

const redisClient = createClient({
  url: 'redis://redis-session.bkash.internal:6379',
  password: 'secretpass',
  database: 1,
});
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient, prefix: 'bkash:sess:' }),
  secret: 'super-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,
    httpOnly: true,
    maxAge: 30 * 60 * 1000, // ৩০ মিনিট
    sameSite: 'strict',
  },
}));

app.post('/api/login', (req, res) => {
  req.session.userId = 'USR_01712345678';
  req.session.loginTime = Date.now();
  res.json({ message: 'লগইন সফল' });
});
```

---

### ৭. SSL/TLS Termination at Load Balancer

SSL/TLS termination মানে LB-তে encryption/decryption হয়, ব্যাকএন্ড সার্ভারে plain HTTP যায়। এতে সার্ভারের CPU বোঝা কমে।

```
  SSL Termination ফ্লো
  ======================

  ┌────────┐  HTTPS (encrypted)  ┌─────────────┐  HTTP (plain)  ┌──────────┐
  │ Client │ ══════════════════→ │ Load        │ ─────────────→ │ Backend  │
  │        │ ◄══════════════════ │ Balancer    │ ◄───────────── │ Server   │
  └────────┘  HTTPS (encrypted)  │             │  HTTP (plain)  └──────────┘
                                 │ SSL cert    │
                                 │ installed   │
                                 │ here        │
                                 └─────────────┘

  সুবিধা:
  ✅ ব্যাকএন্ড সার্ভারে SSL overhead নেই
  ✅ সার্টিফিকেট এক জায়গায় ম্যানেজ
  ✅ SSL offloading → CPU সাশ্রয়

  সতর্কতা:
  ⚠️ LB ↔ Backend HTTP — তাই internal network নিরাপদ রাখতে হবে
  ⚠️ PCI DSS compliance-এ end-to-end encryption লাগতে পারে
     → সেক্ষেত্রে SSL re-encryption (LB → Backend-ও HTTPS)
```

---

### ৮. Global Server Load Balancing (GSLB / GeoDNS)

বিভিন্ন ভৌগোলিক অবস্থানের ডেটা সেন্টারে ট্রাফিক বিতরণ। ইউজারকে নিকটতম ডেটা সেন্টারে পাঠায়।

```
  GSLB — বাংলাদেশ ডিপ্লয়মেন্ট
  ==============================

                    DNS Query: api.bkash.com
                         │
                         ▼
                  ┌──────────────┐
                  │   GSLB/      │
                  │   Route 53   │
                  │   GeoDNS     │
                  └──────┬───────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
            ▼            ▼            ▼
   ┌──────────────┐ ┌──────────┐ ┌──────────────┐
   │ ঢাকা DC      │ │চট্টগ্রাম DC│ │ সিঙ্গাপুর DC  │
   │ (Primary)    │ │(Secondary)│ │ (DR site)    │
   │ 103.x.x.1   │ │ 103.x.x.2│ │ 52.x.x.3    │
   └──────────────┘ └──────────┘ └──────────────┘

  ঢাকা ইউজার → ঢাকা DC       (latency ~5ms)
  চট্টগ্রাম ইউজার → চট্টগ্রাম DC  (latency ~8ms)
  প্রবাসী ইউজার → সিঙ্গাপুর DC   (latency ~50ms)

  Failover: ঢাকা DC ডাউন → সব ট্রাফিক চট্টগ্রাম + সিঙ্গাপুরে
```

---

### ৯. Auto Scaling with Load Balancer

লোড বাড়লে স্বয়ংক্রিয়ভাবে সার্ভার যোগ হয়, কমলে সরিয়ে নেয়। LB নতুন সার্ভারকে স্বয়ংক্রিয়ভাবে pool-এ যোগ করে।

```
  Auto Scaling ফ্লো
  ==================

  স্বাভাবিক সময় (রাত ২টা):
  ──────────────────────────
  LB → [Server 1] [Server 2]           ◄── ২টি সার্ভার যথেষ্ট

  পিক সময় (বিকাল ৫টা, bKash বেতন):
  ────────────────────────────────────
  CloudWatch Alarm: CPU > 70% for 3min
       │
       ▼
  Auto Scaling Group: scale out +3
       │
       ▼
  LB → [S1] [S2] [S3] [S4] [S5]       ◄── ৫টি সার্ভার

  ঈদের দিন (চরম পিক):
  ────────────────────
  CloudWatch Alarm: CPU > 70%, Request count > 10K/min
       │
       ▼
  Auto Scaling Group: scale out +8
       │
       ▼
  LB → [S1] [S2] [S3] ... [S10]        ◄── ১০টি সার্ভার

  ঈদের পরের দিন (ট্রাফিক কমে):
  ─────────────────────────────
  CloudWatch: CPU < 30% for 10min
       │
       ▼
  Auto Scaling: scale in -5 (cooldown: 300s)
       │
       ▼
  LB → [S1] [S2] [S3] [S4] [S5]        ◄── ধীরে ধীরে কমায়
```

```javascript
// AWS SDK — Auto Scaling Policy উদাহরণ
const {
  AutoScalingClient,
  PutScalingPolicyCommand,
} = require('@aws-sdk/client-auto-scaling');

const client = new AutoScalingClient({ region: 'ap-south-1' });

// Target Tracking Policy — CPU 60% এ রাখো
async function setupScaling() {
  const command = new PutScalingPolicyCommand({
    AutoScalingGroupName: 'bkash-payment-asg',
    PolicyName: 'cpu-target-tracking',
    PolicyType: 'TargetTrackingScaling',
    TargetTrackingConfiguration: {
      PredefinedMetricSpecification: {
        PredefinedMetricType: 'ASGAverageCPUUtilization',
      },
      TargetValue: 60.0,
      ScaleInCooldown: 300,
      ScaleOutCooldown: 60,
    },
  });

  const result = await client.send(command);
  console.log('Scaling policy তৈরি হয়েছে:', result.PolicyARN);
}

setupScaling();
```

---

### ১০. Load Balancing in Microservices

মাইক্রোসার্ভিসে দুই ধরনের লোড ব্যালেন্সিং হয়: **Server-side** (traditional) এবং **Client-side** (সার্ভিস নিজে সিদ্ধান্ত নেয়)।

```
  Server-side vs Client-side LB
  ==============================

  Server-side (Traditional):
  ──────────────────────────

  ┌──────────┐     ┌────────┐     ┌─────────────┐
  │ Payment  │ ──→ │  LB    │ ──→ │ User Svc 1  │
  │ Service  │     │(Nginx/ │ ──→ │ User Svc 2  │
  └──────────┘     │ ALB)   │ ──→ │ User Svc 3  │
                   └────────┘     └─────────────┘

  ✅ সার্ভিস জানে না কয়টা instance আছে
  ❌ Extra hop, single point of failure


  Client-side (Modern — Service Mesh):
  ─────────────────────────────────────

  ┌──────────────────────┐     ┌─────────────┐
  │ Payment Service      │ ──→ │ User Svc 1  │
  │ ┌──────────────────┐ │ ──→ │ User Svc 2  │
  │ │ Built-in LB      │ │ ──→ │ User Svc 3  │
  │ │ + Service Registry│ │     └─────────────┘
  │ └──────────────────┘ │
  └──────────────────────┘
       │
       ▼
  ┌────────────────┐
  │ Service        │
  │ Registry       │  ← সব instance এর তালিকা
  │ (Consul/Eureka)│
  └────────────────┘

  ✅ কোনো extra hop নেই
  ✅ LB failure = কোনো single point নেই
  ❌ প্রতিটি সার্ভিসে LB logic রাখতে হয়
```

```javascript
// Client-side Load Balancing — Node.js উদাহরণ
const Consul = require('consul');

class ServiceDiscoveryLB {
  #consul;
  #cache = new Map(); // serviceName → [instances]
  #balancers = new Map();

  constructor(consulHost = 'consul.bkash.internal') {
    this.#consul = new Consul({ host: consulHost, port: 8500 });
  }

  async discover(serviceName) {
    const services = await this.#consul.health.service({
      service: serviceName,
      passing: true, // শুধু সুস্থ সার্ভিস
    });

    const instances = services.map(s => ({
      address: s.Service.Address,
      port: s.Service.Port,
      weight: parseInt(s.Service.Meta?.weight || '1'),
    }));

    this.#cache.set(serviceName, instances);
    return instances;
  }

  async getEndpoint(serviceName) {
    let instances = this.#cache.get(serviceName);
    if (!instances || instances.length === 0) {
      instances = await this.discover(serviceName);
    }

    // Weighted Round Robin
    if (!this.#balancers.has(serviceName)) {
      this.#balancers.set(serviceName, { index: 0 });
    }
    const state = this.#balancers.get(serviceName);
    const instance = instances[state.index % instances.length];
    state.index++;

    return `http://${instance.address}:${instance.port}`;
  }
}

// ব্যবহার
const lb = new ServiceDiscoveryLB();

async function makePayment(userId, amount) {
  const userServiceUrl = await lb.getEndpoint('user-service');
  const paymentServiceUrl = await lb.getEndpoint('payment-service');

  // ইউজার verify
  const userResp = await fetch(`${userServiceUrl}/api/users/${userId}`);
  // পেমেন্ট process
  const payResp = await fetch(`${paymentServiceUrl}/api/pay`, {
    method: 'POST',
    body: JSON.stringify({ userId, amount }),
    headers: { 'Content-Type': 'application/json' },
  });

  return payResp.json();
}
```

---

## ✅ Pros/Cons — অ্যালগরিদম তুলনা

```
  ┌─────────────────────┬────────────────────────┬─────────────────────────┬──────────────────────┐
  │ Algorithm           │ সুবিধা                   │ অসুবিধা                   │ সেরা ব্যবহার ক্ষেত্র    │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Round Robin         │ সহজ, কম overhead        │ সার্ভার ক্ষমতা ভিন্ন হলে  │ সমান ক্ষমতার          │
  │                     │ সমান বিতরণ              │ অকার্যকর                 │ stateless সার্ভার     │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Weighted RR         │ ক্ষমতা অনুযায়ী বিতরণ    │ রিয়েলটাইম লোড দেখে না   │ ভিন্ন ক্ষমতার সার্ভার   │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Least Connections   │ ভারী রিকোয়েস্টে ভালো    │ সমান রিকোয়েস্টে RR-এর   │ দীর্ঘ রিকোয়েস্ট       │
  │                     │ adaptive                │ চেয়ে খারাপ হতে পারে     │ (upload, WebSocket)  │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ IP Hash             │ Session affinity,       │ সার্ভার পরিবর্তনে সব     │ ক্যাশিং,              │
  │                     │ সহজ                    │ ম্যাপিং বদলায়            │ session persistence  │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Consistent Hashing  │ সার্ভার যোগে মাত্র 1/N   │ implementation জটিল     │ ডিস্ট্রিবিউটেড ক্যাশ   │
  │                     │ key বদলায়              │                         │ (Redis cluster)      │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Random              │ সহজ, কোনো state নেই    │ ভারসাম্যহীন হতে পারে     │ খুব বেশি সার্ভার      │
  │ (Power of Two)      │ (P2C প্রায় optimal)     │                         │ থাকলে P2C ভালো      │
  ├─────────────────────┼────────────────────────┼─────────────────────────┼──────────────────────┤
  │ Resource Based      │ সবচেয়ে স্মার্ট          │ agent দরকার,            │ heterogeneous        │
  │                     │ সিদ্ধান্ত                │ latency বেশি            │ infrastructure       │
  └─────────────────────┴────────────────────────┴─────────────────────────┴──────────────────────┘
```

---

## ⚠️ Common Mistakes — সাধারণ ভুলসমূহ

### ১. Single Point of Failure (SPOF) হিসেবে LB

```
  ❌ ভুল:                              ✅ সঠিক:
  ──────                                ──────

  ┌────────┐                            ┌────────────────────┐
  │  LB    │ ◄── মারা গেলে             │   Virtual IP       │
  │(একটি)  │     সব শেষ!               │   (VRRP/Keepalived)│
  └────────┘                            └────────┬───────────┘
                                           ┌─────┴─────┐
                                           ▼           ▼
                                        ┌─────┐    ┌─────┐
                                        │ LB  │    │ LB  │
                                        │ Act │    │Stby │
                                        └─────┘    └─────┘
                                        Active-Passive HA
```

### ২. Health Check না রাখা

**ভুল**: `server backend1:8080;` — কোনো health check নেই। সার্ভার মারা গেলেও ট্রাফিক পাঠাবে।

**সঠিক**: সবসময় health check কনফিগার করুন:
```nginx
server backend1:8080 max_fails=3 fail_timeout=30s;
```

### ৩. Connection Draining ভুলে যাওয়া

সার্ভার বাদ দেওয়ার আগে চলমান রিকোয়েস্ট শেষ হতে দিন। হঠাৎ সরিয়ে দিলে ইউজার ত্রুটি পাবে:

```
  ❌ Hard Remove:                    ✅ Graceful Drain:
  Request 1 → Processing... 💥      Request 1 → Processing... ✅
  Request 2 → Processing... 💥      Request 2 → Processing... ✅
  (সার্ভার হঠাৎ বন্ধ → ত্রুটি)        (নতুন req আসবে না, পুরানো শেষ হবে)
                                    30s পরে সার্ভার নিরাপদে বন্ধ
```

### ৪. Sticky Session-এ অতিরিক্ত নির্ভরতা

Sticky session ব্যবহার করলে auto-scaling ও failover জটিল হয়। **সর্বদা centralized session store (Redis) ব্যবহার করুন।**

### ৫. SSL Everywhere ভুলে যাওয়া

LB-তে SSL terminate করলেও internal network-এ mTLS বা VPC isolation নিশ্চিত করুন। বিশেষত PCI DSS compliance-এর ক্ষেত্রে।

### ৬. Thundering Herd — Health Check Storm

সব ক্লায়েন্ট একসাথে reconnect করলে নতুন সার্ভারে একসাথে ঝড় বয়ে যায়:

```
  সার্ভার পুনরায় চালু
       │
       ▼
  ১০,০০০ ক্লায়েন্ট একসাথে reconnect → 💥 সার্ভার আবার ক্র্যাশ

  সমাধান: Exponential backoff + jitter
  প্রতিটি ক্লায়েন্ট random delay-এ reconnect
```

### ৭. অপর্যাপ্ত Timeout সেটিং

```
  ❌ ভুল timeout:                     ✅ সঠিক:
  connect_timeout: 60s               connect_timeout: 3-5s
  (সার্ভার ডাউন হলে ৬০সে অপেক্ষা!)    read_timeout: 30s (API অনুযায়ী)
                                     write_timeout: 30s
```

---

## 🧪 Testing — লোড টেস্টিং

### k6 দিয়ে লোড টেস্ট

```javascript
// load-test.js — k6 স্ক্রিপ্ট
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// কাস্টম মেট্রিক্স
const errorRate = new Rate('errors');
const paymentDuration = new Trend('payment_duration');

// ঈদের দিনের ট্রাফিক সিমুলেশন
export const options = {
  scenarios: {
    // ধীরে ধীরে বাড়ে — সকালের ট্রাফিক
    morning_ramp: {
      executor: 'ramping-vus',
      startVUs: 10,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 500 },
        { duration: '5m', target: 1000 },
        { duration: '3m', target: 500 },
        { duration: '2m', target: 0 },
      ],
      startTime: '0s',
    },
    // ঈদের সালামী — হঠাৎ spike
    eid_spike: {
      executor: 'ramping-arrival-rate',
      startRate: 50,
      timeUnit: '1s',
      preAllocatedVUs: 2000,
      stages: [
        { duration: '30s', target: 200 },
        { duration: '1m', target: 2000 },  // 💥 spike
        { duration: '2m', target: 2000 },
        { duration: '1m', target: 100 },
      ],
      startTime: '17m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.01'],         // ত্রুটি ১% এর নিচে
    payment_duration: ['p(95)<800'],
  },
};

const BASE_URL = 'https://api.bkash.com';

export default function () {
  // ১. ব্যালেন্স চেক
  const balanceRes = http.get(`${BASE_URL}/api/balance`, {
    headers: {
      Authorization: `Bearer ${__ENV.AUTH_TOKEN}`,
      'Content-Type': 'application/json',
    },
  });
  check(balanceRes, {
    'balance status 200': (r) => r.status === 200,
    'balance latency < 200ms': (r) => r.timings.duration < 200,
  });

  // ২. পেমেন্ট রিকোয়েস্ট
  const paymentStart = Date.now();
  const paymentRes = http.post(
    `${BASE_URL}/api/payment/send`,
    JSON.stringify({
      receiver: '01712345678',
      amount: Math.floor(Math.random() * 5000) + 100,
      pin: '12345',
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );

  paymentDuration.add(Date.now() - paymentStart);
  errorRate.add(paymentRes.status !== 200);

  check(paymentRes, {
    'payment success': (r) => r.status === 200,
    'payment < 500ms': (r) => r.timings.duration < 500,
    'has transaction id': (r) => JSON.parse(r.body).txnId !== undefined,
  });

  sleep(Math.random() * 2 + 0.5); // ইউজার think time
}
```

```bash
# k6 চালানো
k6 run --env AUTH_TOKEN=test_token_123 load-test.js

# InfluxDB + Grafana-তে রিয়েলটাইম মনিটরিং
k6 run --out influxdb=http://grafana.bkash.internal:8086/k6 load-test.js
```

### Artillery দিয়ে টেস্ট

```yaml
# artillery-config.yml
config:
  target: "https://api.bkash.com"
  phases:
    - duration: 120
      arrivalRate: 50
      name: "ওয়ার্মআপ"
    - duration: 300
      arrivalRate: 200
      rampTo: 1000
      name: "ঈদ পিক সিমুলেশন"
    - duration: 120
      arrivalRate: 1000
      name: "sustained load"
    - duration: 60
      arrivalRate: 50
      name: "কুলডাউন"
  defaults:
    headers:
      Content-Type: "application/json"
  plugins:
    expect: {}

scenarios:
  - name: "bKash পেমেন্ট ফ্লো"
    weight: 70
    flow:
      - post:
          url: "/api/auth/login"
          json:
            phone: "01712345678"
            pin: "12345"
          capture:
            - json: "$.token"
              as: "authToken"
          expect:
            - statusCode: 200
      - get:
          url: "/api/balance"
          headers:
            Authorization: "Bearer {{ authToken }}"
          expect:
            - statusCode: 200
      - post:
          url: "/api/payment/send"
          headers:
            Authorization: "Bearer {{ authToken }}"
          json:
            receiver: "01898765432"
            amount: 500
          expect:
            - statusCode: 200
            - hasProperty: "txnId"

  - name: "শুধু ব্যালেন্স চেক"
    weight: 30
    flow:
      - get:
          url: "/api/balance"
          expect:
            - statusCode: 200
```

```bash
# Artillery চালানো
artillery run artillery-config.yml

# HTML রিপোর্ট তৈরি
artillery run --output report.json artillery-config.yml
artillery report report.json --output report.html
```

---

## 📋 সারসংক্ষেপ

### মূল শিক্ষা

```
  লোড ব্যালেন্সিং সিদ্ধান্ত গাছ (Decision Tree)
  ================================================

  তোমার কি URL/Header ভিত্তিক রাউটিং লাগে?
       │
       ├── হ্যাঁ → L7 Load Balancer (ALB / Nginx / HAProxy HTTP mode)
       │           │
       │           ├── সব সার্ভার সমান? → Round Robin
       │           ├── ভিন্ন ক্ষমতা? → Weighted Round Robin
       │           ├── দীর্ঘ রিকোয়েস্ট? → Least Connections
       │           └── ক্যাশ সার্ভার? → Consistent Hashing
       │
       └── না → L4 Load Balancer (NLB / HAProxy TCP mode)
                │
                ├── UDP/Gaming? → Round Robin
                ├── Database proxy? → Least Connections
                └── Fixed mapping? → IP Hash
```

### প্রোডাকশন চেকলিস্ট

```
  ✅ LB High Availability (Active-Passive / Active-Active)
  ✅ Health checks (active + passive) কনফিগার করা
  ✅ SSL/TLS termination সেট করা
  ✅ Connection draining সক্রিয় করা
  ✅ Centralized session store (Redis) ব্যবহার
  ✅ Rate limiting কনফিগার করা
  ✅ Monitoring + alerting সেটআপ (Grafana/CloudWatch)
  ✅ লোড টেস্ট (k6/Artillery) দিয়ে capacity planning
  ✅ Auto Scaling policy সেট করা
  ✅ GSLB/GeoDNS — multi-region failover
  ✅ Connection timeout সঠিকভাবে সেট করা
  ✅ Access logs সক্রিয় করা
  ✅ DDoS protection (WAF + NLB)
```

### বাংলাদেশ প্রসঙ্গে বিশেষ বিবেচনা

- **ঢাকা-চট্টগ্রাম redundancy**: দুই শহরে ডেটা সেন্টার রাখুন, GSLB দিয়ে failover
- **bKash/Nagad-এর মতো ফিনটেক**: PCI DSS compliance-এ end-to-end encryption বাধ্যতামূলক
- **ঈদ/রমজান spike**: Predictive auto-scaling — আগে থেকে সার্ভার বাড়িয়ে রাখুন
- **ISP variability**: Grameenphone, Robi, Banglalink — বিভিন্ন ISP-এর latency ভিন্ন, GeoDNS কাজে আসে
- **USSD gateway**: L4 LB ব্যবহার করুন (TCP proxy) — USSD HTTP নয়
- **বাংলাদেশ ব্যাংক compliance**: ট্রানজেকশন লগ সংরক্ষণ, audit trail বাধ্যতামূলক

---

> **💡 মনে রাখুন**: লোড ব্যালেন্সিং শুধু ট্রাফিক ভাগ করা নয় — এটি **reliability**, **scalability**, এবং **performance**-এর ভিত্তি। সঠিক অ্যালগরিদম, সঠিক health check, এবং সঠিক আর্কিটেকচার — এই তিনটি মিলে একটি শক্তিশালী সিস্টেম তৈরি হয়।
