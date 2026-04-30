# 🆔 ডিস্ট্রিবিউটেড আইডি জেনারেশন (Distributed ID Generation)

> **একটি বিশাল সিস্টেমে প্রতিটি অর্ডার, রাইড বা ট্রানজেকশনের জন্য কিভাবে ইউনিক আইডি তৈরি করবেন?**

---

## 📌 কেন ডিস্ট্রিবিউটেড আইডি দরকার?

**Daraz Bangladesh**-এ প্রতিদিন লক্ষ লক্ষ অর্ডার আসে। একটি ডাটাবেস দিয়ে সামলানো অসম্ভব,
তাই একাধিক সার্ভারে ডাটাবেস ভাগ (sharding) করা হয়। এখন auto-increment ব্যবহার করলে:

```
❌ সমস্যা: Auto-Increment + Multiple Shards = আইডি সংঘর্ষ

  Shard 1 (ঢাকা)          Shard 2 (চট্টগ্রাম)        Shard 3 (সিলেট)
  ┌─────────────┐         ┌─────────────┐          ┌─────────────┐
  │ order_id: 1 │         │ order_id: 1 │          │ order_id: 1 │
  │ order_id: 2 │         │ order_id: 2 │          │ order_id: 2 │
  └─────────────┘         └─────────────┘          └─────────────┘
         │                       │                        │
         └───────────┬───────────┘────────────────────────┘
                     ▼
          💥 তিনটি shard-এই order_id: 1 আছে! কোনটি আসল?
```

### 🏠 বাস্তব বাংলাদেশ উদাহরণ

| সার্ভিস | আইডি উদাহরণ | প্রতিদিন আনুমানিক |
|---------|-------------|-------------------|
| **Daraz** অর্ডার | `DZ-80912345678` | ১০ লক্ষ+ অর্ডার |
| **Pathao** রাইড | `RIDE-2024-78A3F` | ৫ লক্ষ+ রাইড |
| **bKash** ট্রানজেকশন | `TXN9A3F7B2E` | ১ কোটি+ লেনদেন |
| **Nagad** পেমেন্ট | `NGD20240115...` | ৫০ লক্ষ+ |
| **Grameenphone** রিচার্জ | `GP-RCH-...` | ২০ লক্ষ+ |

---

## 🔑 UUID v4 (Universally Unique Identifier)

UUID v4 হলো **128-bit র‍্যান্ডম আইডি**। কোনো coordination ছাড়াই যেকোনো মেশিনে তৈরি করা যায়।

```
UUID v4 গঠন:  550e8400-e29b-41d4-a716-446655440000
               ├──────┤ ├──┤ ├──┤ ├──┤ ├──────────┤
               time-lo  mid  hi+v  clk    node
               মোট: 128 bits = 36 characters (with hyphens)
```

✅ **সুবিধা:** কোনো coordination লাগে না, সংঘর্ষের সম্ভাবনা প্রায় শূন্য
❌ **অসুবিধা:** সর্টেবল না, 36 chars (বড়), B-tree ইনডেক্সে ধীর, পড়া কঠিন

```php
<?php
// UUID v4 — PHP
function generateUUIDv4(): string {
    $data = random_bytes(16);
    $data[6] = chr(ord($data[6]) & 0x0f | 0x40); // version 4
    $data[8] = chr(ord($data[8]) & 0x3f | 0x80); // variant RFC 4122
    return vsprintf('%s%s-%s-%s-%s-%s%s%s', str_split(bin2hex($data), 4));
}

$order = ['id' => generateUUIDv4(), 'user' => 'rahim_uddin', 'amount' => 1250.00];
```

```javascript
// UUID v4 — JavaScript
const { randomUUID } = require('crypto');

const transaction = {
    id: randomUUID(), // "1b9d6bcd-bbfd-4b2d-9b5e-ab8dfbbd4bed"
    sender: '01712345678',
    amount: 500,
    type: 'send-money'
};
```

---

## ❄️ Twitter Snowflake ID

**Snowflake** হলো সবচেয়ে জনপ্রিয় ডিস্ট্রিবিউটেড আইডি অ্যালগরিদম — **64-bit integer**,
সময় অনুযায়ী সর্টেবল এবং প্রতিটি মেশিনে ইউনিক।

```
Snowflake ID বিট লেআউট (64-bit):
┌───┬──────────────────────────────┬───────────────┬────────────────┐
│ 0 │     41 bits: Timestamp (ms)  │ 10 bits: M.ID │ 12 bits: Seq   │
└───┴──────────────────────────────┴───────────────┴────────────────┘

┌───────┬──────────────┬──────────────────────────────────┐
│ Bits  │ ক্ষেত্র      │ বিবরণ                            │
├───────┼──────────────┼──────────────────────────────────┤
│  1    │ Sign         │ সবসময় 0 (positive integer)      │
│ 41    │ Timestamp    │ কাস্টম epoch থেকে (~69 বছর)      │
│ 10    │ Machine ID   │ 1024 টি মেশিন সাপোর্ট            │
│ 12    │ Sequence     │ প্রতি ms-এ 4096 আইডি             │
└───────┴──────────────┴──────────────────────────────────┘

ক্যাপাসিটি: 4,096 × 1,000 ms × 1,024 মেশিন = ~400 কোটি আইডি/সেকেন্ড 🚀
Epoch: 2024-01-01 → 2^41 ms ≈ 2093 সাল পর্যন্ত চলবে ✅
```

### কিভাবে ইউনিকনেস নিশ্চিত হয়?

```
  Server A (Machine ID: 5)              Server B (Machine ID: 12)
  ┌──────────────────────┐              ┌──────────────────────┐
  │ Timestamp: 17089...  │              │ Timestamp: 17089...  │
  │ Machine:   00101     │ ◄── আলাদা ──►│ Machine:   01100     │
  │ Sequence:  000000001 │              │ Sequence:  000000001 │
  └──────────────────────┘              └──────────────────────┘
        ▼                                      ▼
   7089370115080193                      7089370115153921
   (সম্পূর্ণ আলাদা আইডি ✅)             (সম্পূর্ণ আলাদা আইডি ✅)
```

---

## 🗄️ ডাটাবেস অটো-ইনক্রিমেন্টের সীমাবদ্ধতা

```
সমস্যা ১: Single Point of Failure        সমস্যা ২: Multi-Master Conflict
─────────────────────────────             ──────────────────────────────
        ┌──────────┐                       Master 1        Master 2
        │ Master DB │ ◄── মারা গেলে        ┌────────┐     ┌────────┐
        │ (auto_inc)│     সব বন্ধ! 💀      │ id:1001│     │ id:1001│ 💥
        └──────────┘                       │ id:1002│     │ id:1002│ 💥
        /    |    \                        └────────┘     └────────┘
   App1   App2   App3                      Odd/Even workaround ভঙ্গুর!
```

| সমস্যা | বিবরণ |
|--------|-------|
| **Scalability** | Horizontally scale করা যায় না |
| **ID অনুমানযোগ্য** | `/order/1001` → পরেরটি `/1002` — নিরাপত্তা ঝুঁকি |
| **Merge Conflict** | একাধিক DB merge করলে ID clash |
| **Bottleneck** | সব request একটি DB-তে — ধীর |

```
❌ Bad: GET /api/orders/10045 → পরেরটি /10046 (অনুমানযোগ্য!)
✅ Good: GET /api/orders/01HQJX7V6R... → অনুমান করা অসম্ভব
```

---

## 📊 ULID (Universally Unique Lexicographically Sortable Identifier)

UUID-এর **উন্নত বিকল্প** — সর্টেবল এবং URL-safe।

```
ULID গঠন (128 bits):
  01ARZ3NDEKTSV4RRFFQ69G5FAV
  ├────────┤├──────────────────┤
   Timestamp     Randomness        Encoding: Crockford's Base32
   (48 bits)     (80 bits)         অক্ষর: 0-9 A-Z (I,L,O,U বাদ)
   10 chars      16 chars

  UUID v4:  550e8400-e29b-41d4-a716-446655440000  (36 chars, unsorted)
  ULID:     01ARZ3NDEKTSV4RRFFQ69G5FAV            (26 chars, sorted ✅)

  ইনডেক্স: UUID ████░░░░░░ (slow) | ULID ██████████ (fast 🚀)
```

```php
<?php
// ULID জেনারেটর — PHP
function generateULID(): string {
    $encoding = '0123456789ABCDEFGHJKMNPQRSTVWXYZ';
    $timestamp = intval(microtime(true) * 1000);

    $timePart = '';
    for ($i = 9; $i >= 0; $i--) {
        $timePart = $encoding[$timestamp % 32] . $timePart;
        $timestamp = intdiv($timestamp, 32);
    }

    $randPart = '';
    $randomBytes = random_bytes(10);
    for ($i = 0; $i < 10; $i++) {
        $byte = ord($randomBytes[$i]);
        $randPart .= $encoding[$byte >> 3];
        if (strlen($randPart) < 16) {
            $randPart .= $encoding[$byte & 0x1F];
        }
    }
    return $timePart . substr($randPart, 0, 16);
}

echo "Ride: " . generateULID(); // "01HQJX7V6R8N3MTKFQZ5P0XWYA"
```

```javascript
// ULID জেনারেটর — JavaScript
function generateULID() {
    const ENCODING = '0123456789ABCDEFGHJKMNPQRSTVWXYZ';
    let timestamp = Date.now();

    let timePart = '';
    for (let i = 9; i >= 0; i--) {
        timePart = ENCODING[timestamp % 32] + timePart;
        timestamp = Math.floor(timestamp / 32);
    }

    const randBytes = require('crypto').randomBytes(10);
    let randPart = '';
    for (const byte of randBytes) randPart += ENCODING[byte % 32];

    return timePart + randPart.substring(0, 16);
}

console.log(`Payment: ${generateULID()}`); // "01HQJX9P2T5K7WDJFG3HN8RQVX"
```

---

## 📸 Instagram-style ID জেনারেশন

Instagram **PostgreSQL-এর ভিতরেই** আইডি তৈরি করে — কোনো বাইরের সার্ভিস লাগে না।

```
Instagram ID বিট লেআউট (64-bit):
┌───────────────────────────────────────────────────────────┐
│  41 bits: Timestamp (ms)  │ 13 bits: Shard │ 10 bits: Seq │
│  কাস্টম epoch থেকে       │  ID (0-8191)   │ (0-1023)     │
└───────────────────────────────────────────────────────────┘
  ✅ DB-র ভেতরেই হয়   ✅ Shard-aware   ✅ সর্টেবল   ✅ 64-bit
```

```sql
-- Instagram-style ID জেনারেটর (PL/pgSQL)
CREATE OR REPLACE FUNCTION generate_id(shard_id INT)
RETURNS BIGINT AS $$
DECLARE
    our_epoch  BIGINT := 1704067200000; -- 2024-01-01 UTC
    now_ms     BIGINT;
    seq_id     BIGINT;
    result     BIGINT;
BEGIN
    SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_ms;
    seq_id := nextval('table_id_seq') % 1024;

    result := (now_ms - our_epoch) << 23;  -- timestamp
    result := result | (shard_id << 10);    -- shard
    result := result | seq_id;              -- sequence
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- ব্যবহার: SELECT generate_id(42);
-- আইডি থেকে shard বের করা: shard = (id >> 10) & 0x1FFF → 42 🎯
```

---

## 📋 তুলনামূলক টেবিল (Comparison Table)

```
┌──────────────────┬──────────┬──────────┬───────────┬───────────┬──────────┐
│ বৈশিষ্ট্য        │ UUID v4  │ Snowflake│   ULID    │ Instagram │ Auto-Inc │
├──────────────────┼──────────┼──────────┼───────────┼───────────┼──────────┤
│ সাইজ (bits)      │   128    │    64    │    128    │     64    │  32/64   │
│ সাইজ (chars)     │    36    │   ~19    │    26     │    ~19    │  varies  │
│ সর্টেবল          │    ❌    │    ✅    │    ✅     │     ✅    │   ✅     │
│ ইউনিকনেস         │  খুব ভালো │  নিশ্চিত │  খুব ভালো │  নিশ্চিত  │ সীমিত   │
│ সমন্বয় দরকার?   │    না    │  মেশিন ID│   না      │  Shard ID │    হ্যাঁ │
│ পারফরম্যান্স     │  মাঝারি  │   উচ্চ   │    ভালো   │    উচ্চ   │  উচ্চ    │
│ ইনডেক্স দক্ষতা   │   খারাপ  │   ভালো   │   ভালো    │    ভালো   │ সবচেয়ে  │
│ URL-safe          │    না    │    হ্যাঁ │    হ্যাঁ  │    হ্যাঁ  │   হ্যাঁ  │
│ অনুমানযোগ্যতা    │    না    │  আংশিক  │    না     │  আংশিক   │   হ্যাঁ  │
│ Horizontal Scale │    ✅    │    ✅    │    ✅     │     ✅    │    ❌    │
└──────────────────┴──────────┴──────────┴───────────┴───────────┴──────────┘
```

---

## 💻 PHP এবং JS — Snowflake ID জেনারেটর (সম্পূর্ণ ক্লাস)

### PHP ইমপ্লিমেন্টেশন

```php
<?php
class SnowflakeIdGenerator
{
    private const EPOCH        = 1704067200000; // 2024-01-01 UTC (ms)
    private const MACHINE_BITS = 10;
    private const SEQ_BITS     = 12;
    private const MAX_MACHINE  = (1 << self::MACHINE_BITS) - 1; // 1023
    private const MAX_SEQUENCE = (1 << self::SEQ_BITS) - 1;     // 4095
    private const MACHINE_SHIFT = self::SEQ_BITS;
    private const TIME_SHIFT    = self::MACHINE_BITS + self::SEQ_BITS;

    private int $machineId;
    private int $sequence = 0;
    private int $lastTimestamp = -1;

    public function __construct(int $machineId)
    {
        if ($machineId < 0 || $machineId > self::MAX_MACHINE) {
            throw new InvalidArgumentException("Machine ID: 0-" . self::MAX_MACHINE);
        }
        $this->machineId = $machineId;
    }

    public function generate(): int
    {
        $timestamp = $this->currentTimeMs();

        if ($timestamp === $this->lastTimestamp) {
            $this->sequence = ($this->sequence + 1) & self::MAX_SEQUENCE;
            if ($this->sequence === 0) {
                $timestamp = $this->waitNextMs($this->lastTimestamp);
            }
        } else {
            $this->sequence = 0;
        }

        if ($timestamp < $this->lastTimestamp) {
            throw new RuntimeException("ঘড়ি পিছিয়ে গেছে!");
        }
        $this->lastTimestamp = $timestamp;

        return (($timestamp - self::EPOCH) << self::TIME_SHIFT)
             | ($this->machineId << self::MACHINE_SHIFT)
             | $this->sequence;
    }

    private function currentTimeMs(): int  { return intval(microtime(true) * 1000); }
    private function waitNextMs(int $last): int {
        $ts = $this->currentTimeMs();
        while ($ts <= $last) $ts = $this->currentTimeMs();
        return $ts;
    }
}

// bKash ট্রানজেকশন আইডি তৈরি
$gen = new SnowflakeIdGenerator(machineId: 7);
for ($i = 0; $i < 3; $i++) echo "TXN-{$gen->generate()}\n";
```

### JavaScript ইমপ্লিমেন্টেশন

```javascript
class SnowflakeIdGenerator {
    static EPOCH = 1704067200000n;
    static MACHINE_BITS = 10n;
    static SEQ_BITS = 12n;
    static MAX_MACHINE = (1n << this.MACHINE_BITS) - 1n;
    static MAX_SEQUENCE = (1n << this.SEQ_BITS) - 1n;
    static MACHINE_SHIFT = this.SEQ_BITS;
    static TIME_SHIFT = this.MACHINE_BITS + this.SEQ_BITS;

    #machineId; #sequence = 0n; #lastTimestamp = -1n;

    constructor(machineId) {
        const mid = BigInt(machineId);
        if (mid < 0n || mid > SnowflakeIdGenerator.MAX_MACHINE)
            throw new Error('Machine ID: 0-1023');
        this.#machineId = mid;
    }

    generate() {
        let ts = BigInt(Date.now());
        if (ts === this.#lastTimestamp) {
            this.#sequence = (this.#sequence + 1n) & SnowflakeIdGenerator.MAX_SEQUENCE;
            if (this.#sequence === 0n) ts = this.#waitNextMs(this.#lastTimestamp);
        } else {
            this.#sequence = 0n;
        }
        if (ts < this.#lastTimestamp) throw new Error('ঘড়ি পিছিয়ে গেছে!');
        this.#lastTimestamp = ts;

        return ((ts - SnowflakeIdGenerator.EPOCH) << SnowflakeIdGenerator.TIME_SHIFT)
             | (this.#machineId << SnowflakeIdGenerator.MACHINE_SHIFT)
             | this.#sequence;
    }

    #waitNextMs(last) {
        let ts = BigInt(Date.now());
        while (ts <= last) ts = BigInt(Date.now());
        return ts;
    }
}

// Pathao রাইড আইডি তৈরি
const gen = new SnowflakeIdGenerator(12);
for (let i = 0; i < 3; i++) console.log(`RIDE-${gen.generate()}`);
```

---

## 🎯 কখন কোনটি ব্যবহার করবেন

```
                আপনার সিস্টেমের প্রয়োজন কী?
                          │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       সিম্পল,        সর্টেবল,       DB-ভিত্তিক,
     দ্রুত লাগবে    হাই-পারফরম্যান্স  shard-aware
            │              │              │
            ▼              ▼              ▼
       UUID v4      ┌──────┴─────┐   Instagram
                    ▼            ▼     style
                Snowflake      ULID
```

| পরিস্থিতি | সুপারিশ | কারণ |
|-----------|---------|------|
| ছোট প্রজেক্ট, single DB | **Auto-Increment** | সবচেয়ে সিম্পল |
| মাইক্রোসার্ভিস, কোনো সমন্বয় নেই | **UUID v4** | কোনো setup লাগে না |
| হাই-ট্রাফিক, সর্টিং দরকার | **Snowflake** | দ্রুত, 64-bit, সর্টেবল |
| সর্টেবল + coordination-free | **ULID** | UUID-এর ভালো বিকল্প |
| PostgreSQL + Sharding | **Instagram-style** | DB-তেই সমাধান |

### 🏠 বাংলাদেশ কনটেক্সট

```
🛒 Daraz:   লক্ষ লক্ষ অর্ডার → Snowflake (দ্রুত, সর্টেবল, multi-DC)
🏍️ Pathao:  রাইড আইডি → ULID (সর্টেবল, ফোনেও তৈরি করা যায়)
💰 bKash:   ট্রানজেকশন → Snowflake (হাই-থ্রুপুট, প্রতি সেকেন্ডে হাজার+)
📱 GP MyGP: রিচার্জ রেফ → UUID v4 (সিম্পল, সর্টিং দরকার নেই)
🏥 ক্লিনিক: পেশেন্ট ID → Auto-Increment (একটি DB, যথেষ্ট!)
```

---

## ❌ সাধারণ ভুল (Common Mistakes)

### ভুল ১: র‍্যান্ডম স্ট্রিং দিয়ে আইডি

```php
// ❌ Bad: collision ঝুঁকি!
function badId(): string {
    return substr(str_shuffle('abcdefghijklmnop0123456789'), 0, 10);
}
// ✅ Good: প্রমাণিত অ্যালগরিদম
$id = generateULID();
```

### ভুল ২: শুধু Timestamp দিয়ে আইডি

```javascript
// ❌ Bad: একই ms-এ দুটি request = collision!
const badId = () => Date.now().toString();
// ✅ Good: Timestamp + Machine + Sequence
const goodId = new SnowflakeIdGenerator(1).generate();
```

### ভুল ৩: সিকিউরিটি-সেনসিটিভ ক্ষেত্রে অনুমানযোগ্য আইডি

```
❌ /reset-password?token=1001 → পরেরটি ?token=1002 (হ্যাকার অনুমান করবে!)
✅ /reset-password?token=a8f5f167f44f4964e6c998dee827110c
```

### ভুল ৪: বড় স্কেলে Auto-Increment

```
❌ ১০ কোটি ইউজারের সিস্টেমে single DB auto-increment → Bottleneck 💀
✅ শুরু থেকেই distributed ID strategy → ভবিষ্যতে scale সহজ
```

### ভুল ৫: Clock Skew উপেক্ষা করা

```
❌ সার্ভারের ঘড়ি sync না করে Snowflake ব্যবহার → সময়ের ক্রম ভেঙে যায়!
✅ NTP দিয়ে ঘড়ি sync রাখুন + clock backward detection (আমাদের কোডে আছে ✅)
```

---

## 💡 সারসংক্ষেপ

```
┌──────────────────────────────────────────────────────────┐
│              মূল শিক্ষা (Key Takeaways)                  │
│                                                          │
│  ১. Auto-Increment → single DB-তে ভালো, distributed-এ না│
│  ২. UUID v4 → সিম্পল কিন্তু সর্টেবল নয়, ইনডেক্স ধীর    │
│  ৩. Snowflake → industry standard (Twitter, Discord)     │
│  ৪. ULID → UUID-এর আধুনিক বিকল্প, সর্টেবল ও URL-safe  │
│  ৫. Instagram-style → PostgreSQL-এ shard-aware সমাধান    │
│                                                          │
│  ⚠️ মনে রাখুন:                                           │
│  • সিকিউরিটি আইডি সবসময় unpredictable হবে              │
│  • NTP দিয়ে ঘড়ি sync রাখুন                              │
│  • শুরুতেই সঠিক strategy নিন — পরে পাল্টানো কঠিন        │
└──────────────────────────────────────────────────────────┘
```

---

## 🔗 আরও পড়ুন

- [Twitter Snowflake (GitHub)](https://github.com/twitter-archive/snowflake)
- [ULID Specification](https://github.com/ulid/spec)
- [Instagram Engineering: Sharding & IDs](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- [UUID RFC 4122](https://tools.ietf.org/html/rfc4122)

---

> **📝 নোট:** এই study material **dev-pathshala** প্রজেক্টের অংশ। সিস্টেম ডিজাইন
> ইন্টারভিউতে Distributed ID Generation অত্যন্ত গুরুত্বপূর্ণ টপিক।
> Snowflake ও ULID ভালোভাবে বোঝা থাকলে ইন্টারভিউতে ভালো করা সম্ভব। 🎯
