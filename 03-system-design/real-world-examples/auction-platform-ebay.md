# 🏷️ অকশন প্ল্যাটফর্ম সিস্টেম ডিজাইন (eBay)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: eBay, Sotheby's, Bikroy.com, Bangladesh Government e-GP

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বর্ণনা |
|---|---------|--------|
| 1 | Item Listing | সেলার প্রোডাক্ট লিস্ট করতে পারবে (ছবি, বর্ণনা, starting price) |
| 2 | Place Bid | বায়ার রিয়েল-টাইমে বিড করতে পারবে |
| 3 | Proxy Bidding | অটোমেটিক বিডিং — ম্যাক্সিমাম প্রাইস সেট করলে সিস্টেম নিজে বিড করবে |
| 4 | Auction Timer | কাউন্টডাউন টাইমার + snipe protection (শেষ মুহূর্তে বিড আসলে সময় বাড়বে) |
| 5 | Buy Now | ফিক্সড প্রাইসে তাৎক্ষণিক কেনার অপশন |
| 6 | Seller Dashboard | বিক্রেতার সব অকশনের স্ট্যাটাস, earning analytics |
| 7 | Payment Escrow | পেমেন্ট এসক্রো — ডেলিভারি কনফার্ম না হওয়া পর্যন্ত টাকা হোল্ড |
| 8 | Notification | বিড আউটবিড, অকশন শেষ, পেমেন্ট কনফার্মেশন নোটিফিকেশন |

### Non-Functional Requirements

| # | Requirement | টার্গেট |
|---|-------------|---------|
| 1 | Real-time Updates | < 100ms latency তে বিড আপডেট সবাই দেখবে |
| 2 | Consistency | কোনো invalid bid accept হবে না (race condition proof) |
| 3 | High Availability | 99.99% uptime — অকশন চলাকালীন ডাউন হলে বিশাল ক্ষতি |
| 4 | Fraud Prevention | Shill bidding, bid manipulation ডিটেক্ট করবে |
| 5 | Scalability | 1M+ concurrent bidders handle করতে পারবে |
| 6 | Durability | কোনো বিড হারিয়ে যাবে না — guaranteed persistence |

### 🇧🇩 বাংলাদেশ কনটেক্সট

```
বাংলাদেশে অকশন/বিডিং সিস্টেমের উদাহরণ:
┌─────────────────────────────────────────────────────────┐
│ ১. Bikroy.com         → C2C মার্কেটপ্লেস (নেগোশিয়েশন) │
│ ২. e-GP (সরকারি)      → Government tender bidding       │
│ ৩. জমি নিলাম          → Court-ordered land auction      │
│ ৪. Facebook Marketplace BD → ইনফর্মাল বিডিং            │
│ ৫. চা নিলাম (চট্টগ্রাম)  → Tea auction (Chittagong)     │
│ ৬. গার্মেন্টস surplus  → Bulk lot auction               │
└─────────────────────────────────────────────────────────┘

চ্যালেঞ্জ:
- bKash/Nagad integration (escrow)
- ঢাকার বাইরে slow internet → offline-first bidding
- NID verification for trust
- বাংলা + English bilingual UI
```

---

## 📊 Back-of-envelope Estimation

### ট্র্যাফিক এস্টিমেশন (eBay স্কেল)

```
Daily Active Users (DAU):     50M
Active Auctions:              200M items
Bids per day:                 500M
Peak bids/second:             50,000

── গণনা ──────────────────────────────────
Average bid size:             ~200 bytes
Daily bid data:               500M × 200B = 100GB/day
Bid write QPS (average):      500M / 86400 ≈ 5,800 QPS
Peak QPS:                     50,000 QPS (10x average)

WebSocket connections (peak): 5M concurrent
Notification messages/day:    2B (push + email + SMS)
```

### স্টোরেজ এস্টিমেশন

```
┌──────────────────────────────────────────────────┐
│ Data Type          │ Size/record │ Records/year  │
├──────────────────────────────────────────────────┤
│ Bid records        │ 200B        │ 180B → 36TB  │
│ Auction metadata   │ 2KB         │ 500M → 1TB   │
│ Item images        │ 5MB avg     │ 500M → 2.5PB │
│ User profiles      │ 5KB         │ 300M → 1.5TB │
│ Transaction logs   │ 500B        │ 2B → 1TB     │
└──────────────────────────────────────────────────┘

Total Storage (yearly): ~2.5PB (mostly images)
```

### 🇧🇩 বাংলাদেশ স্কেল (Bikroy.com ধরনের)

```
DAU:                  500K
Active listings:      2M
Bids/day:            100K
Peak bids/sec:       200
WebSocket:           50K concurrent
Storage:             500GB/year
```

---

## 🏗️ High-Level Design

### System Architecture (ASCII Diagram)

```
                        ┌──────────────────────────────────────────────┐
                        │              CLIENT LAYER                     │
                        │  ┌─────────┐ ┌─────────┐ ┌───────────────┐  │
                        │  │ Web App │ │ Mobile  │ │ Seller Portal │  │
                        │  │ (React) │ │  App    │ │   (Laravel)   │  │
                        │  └────┬────┘ └────┬────┘ └───────┬───────┘  │
                        └───────┼───────────┼──────────────┼───────────┘
                                │           │              │
                        ════════╪═══════════╪══════════════╪═══════════
                                ▼           ▼              ▼
                        ┌──────────────────────────────────────────────┐
                        │           API GATEWAY (Kong/Nginx)            │
                        │  ┌────────────────────────────────────────┐  │
                        │  │ Rate Limiting │ Auth │ Load Balancing  │  │
                        │  └────────────────────────────────────────┘  │
                        └───────┬───────────┬──────────────┬───────────┘
                                │           │              │
              ┌─────────────────┼───────────┼──────────────┼──────────┐
              │                 ▼           ▼              ▼          │
              │  ┌──────────────────┐ ┌──────────┐ ┌────────────┐   │
              │  │   BID SERVICE    │ │ AUCTION  │ │  LISTING   │   │
              │  │   (Node.js)      │ │ ENGINE   │ │  SERVICE   │   │
              │  │                  │ │ (Node.js)│ │  (Laravel) │   │
              │  │ • Validate bid   │ │          │ │            │   │
              │  │ • Race condition │ │ • Timer  │ │ • CRUD     │   │
              │  │ • Proxy bidding  │ │ • Rules  │ │ • Search   │   │
              │  └────────┬─────────┘ └─────┬────┘ └─────┬──────┘   │
              │           │                 │            │           │
              │           ▼                 ▼            ▼           │
              │  ┌──────────────────────────────────────────────┐    │
              │  │           MESSAGE QUEUE (Redis/Kafka)         │    │
              │  └──────┬──────────────┬────────────────┬───────┘    │
              │         │              │                │            │
              │         ▼              ▼                ▼            │
              │  ┌────────────┐ ┌────────────┐ ┌──────────────┐     │
              │  │NOTIFICATION│ │  PAYMENT   │ │   FRAUD      │     │
              │  │  SERVICE   │ │  SERVICE   │ │  DETECTION   │     │
              │  │            │ │            │ │              │     │
              │  │ • Push     │ │ • Escrow   │ │ • ML Model   │     │
              │  │ • Email    │ │ • bKash    │ │ • Shill det. │     │
              │  │ • SMS      │ │ • Stripe   │ │ • Pattern    │     │
              │  └────────────┘ └────────────┘ └──────────────┘     │
              │                                                      │
              │              MICROSERVICES LAYER                      │
              └──────────────────────────────────────────────────────┘
                                        │
              ┌─────────────────────────────────────────────────────┐
              │                   DATA LAYER                         │
              │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────┐ │
              │  │PostgreSQL│ │  Redis   │ │  Elastic │ │  S3   │ │
              │  │(Bids,    │ │(Cache,   │ │ Search   │ │(Images│ │
              │  │ Auctions)│ │ Realtime)│ │(Listings)│ │ Files)│ │
              │  └──────────┘ └──────────┘ └──────────┘ └───────┘ │
              └─────────────────────────────────────────────────────┘
```

### Real-time Bid Update Flow

```
  Bidder A                 Server                    All Viewers
     │                       │                           │
     │── HTTP POST /bid ────▶│                           │
     │                       │── Validate ──┐            │
     │                       │◀─────────────┘            │
     │                       │── Update DB ──┐           │
     │                       │◀──────────────┘           │
     │                       │                           │
     │◀── 200 OK ───────────│                           │
     │                       │── WebSocket broadcast ───▶│
     │                       │   (new highest bid)       │
     │                       │                           │
     │                       │── Check Proxy Bids ──┐    │
     │                       │◀─────────────────────┘    │
     │                       │                           │
     │                       │── Auto-bid placed ───────▶│
     │◀── Outbid notify ────│                           │
     │                       │                           │
```

---

## 💻 Detailed Design

### 1. Database Schema

```sql
-- Core Tables
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    phone           VARCHAR(20),          -- bKash/Nagad linked
    nid_number      VARCHAR(17),          -- Bangladesh NID
    trust_score     DECIMAL(3,2) DEFAULT 0.00,
    is_verified     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE items (
    id              BIGSERIAL PRIMARY KEY,
    seller_id       BIGINT REFERENCES users(id),
    title           VARCHAR(255) NOT NULL,
    title_bn        VARCHAR(255),         -- বাংলা টাইটেল
    description     TEXT,
    category_id     INT REFERENCES categories(id),
    images          JSONB,                -- S3 URLs
    condition       VARCHAR(20),          -- new, used, refurbished
    starting_price  DECIMAL(12,2) NOT NULL,
    reserve_price   DECIMAL(12,2),        -- মিনিমাম সেল প্রাইস
    buy_now_price   DECIMAL(12,2),
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE auctions (
    id              BIGSERIAL PRIMARY KEY,
    item_id         BIGINT REFERENCES items(id),
    status          VARCHAR(20) DEFAULT 'scheduled',
                    -- scheduled, active, ended, cancelled
    start_time      TIMESTAMP NOT NULL,
    end_time        TIMESTAMP NOT NULL,
    current_price   DECIMAL(12,2),
    bid_count       INT DEFAULT 0,
    winner_id       BIGINT REFERENCES users(id),
    snipe_extension BOOLEAN DEFAULT TRUE, -- anti-snipe
    version         INT DEFAULT 0,        -- optimistic locking
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE bids (
    id              BIGSERIAL PRIMARY KEY,
    auction_id      BIGINT REFERENCES auctions(id),
    bidder_id       BIGINT REFERENCES users(id),
    amount          DECIMAL(12,2) NOT NULL,
    max_amount      DECIMAL(12,2),        -- proxy bid max
    is_proxy        BOOLEAN DEFAULT FALSE,
    bid_time        TIMESTAMP DEFAULT NOW(),
    ip_address      INET,
    user_agent      TEXT
);

-- Indexes for performance
CREATE INDEX idx_bids_auction_amount ON bids(auction_id, amount DESC);
CREATE INDEX idx_auctions_status_end ON auctions(status, end_time);
CREATE INDEX idx_auctions_version ON auctions(id, version);

CREATE TABLE payments (
    id              BIGSERIAL PRIMARY KEY,
    auction_id      BIGINT REFERENCES auctions(id),
    buyer_id        BIGINT REFERENCES users(id),
    seller_id       BIGINT REFERENCES users(id),
    amount          DECIMAL(12,2) NOT NULL,
    escrow_status   VARCHAR(20) DEFAULT 'held',
                    -- held, released, refunded, disputed
    payment_method  VARCHAR(30),          -- bkash, nagad, card, bank
    transaction_id  VARCHAR(100),
    released_at     TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

### 2. Bid Placement — PHP (Laravel) with Optimistic Locking

```php
<?php

namespace App\Services;

use App\Models\Auction;
use App\Models\Bid;
use App\Events\BidPlaced;
use App\Events\OutbidNotification;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;

class BidService
{
    /**
     * Place a bid with optimistic locking to prevent race conditions.
     * 
     * বিড প্লেসমেন্ট — Race condition হ্যান্ডলিং সহ
     * Optimistic locking: version number চেক করে update করা হয়
     */
    public function placeBid(int $auctionId, int $bidderId, float $amount): array
    {
        $maxRetries = 3;
        $attempt = 0;

        while ($attempt < $maxRetries) {
            try {
                return DB::transaction(function () use ($auctionId, $bidderId, $amount) {
                    
                    // Step 1: Auction ফেচ করো (current version সহ)
                    $auction = Auction::where('id', $auctionId)
                        ->where('status', 'active')
                        ->where('end_time', '>', now())
                        ->first();

                    if (!$auction) {
                        throw new \Exception('অকশন অ্যাক্টিভ নেই বা শেষ হয়ে গেছে');
                    }

                    // Step 2: Bid validation
                    $this->validateBid($auction, $bidderId, $amount);

                    // Step 3: Optimistic lock — version চেক করে update
                    $updated = Auction::where('id', $auctionId)
                        ->where('version', $auction->version)
                        ->update([
                            'current_price' => $amount,
                            'bid_count'     => DB::raw('bid_count + 1'),
                            'version'       => DB::raw('version + 1'),
                        ]);

                    // যদি version mismatch হয়, retry হবে
                    if ($updated === 0) {
                        throw new \Exception('OPTIMISTIC_LOCK_CONFLICT');
                    }

                    // Step 4: Bid record save
                    $bid = Bid::create([
                        'auction_id' => $auctionId,
                        'bidder_id'  => $bidderId,
                        'amount'     => $amount,
                        'bid_time'   => now(),
                        'ip_address' => request()->ip(),
                    ]);

                    // Step 5: Anti-snipe — শেষ ৫ মিনিটে বিড আসলে সময় বাড়াও
                    $this->handleSnipeProtection($auction);

                    // Step 6: Proxy bids চেক করো
                    $this->processProxyBids($auction, $bid);

                    // Step 7: Real-time broadcast
                    event(new BidPlaced($auction, $bid));

                    // Step 8: পুরানো highest bidder কে notify করো
                    $this->notifyOutbidUser($auction, $bidderId);

                    return [
                        'success'       => true,
                        'bid_id'        => $bid->id,
                        'current_price' => $amount,
                        'bid_count'     => $auction->bid_count + 1,
                    ];
                });

            } catch (\Exception $e) {
                if ($e->getMessage() === 'OPTIMISTIC_LOCK_CONFLICT') {
                    $attempt++;
                    usleep(50000 * $attempt); // exponential backoff
                    continue;
                }
                throw $e;
            }
        }

        throw new \Exception('বিড প্লেস করা যায়নি — অনেক বেশি concurrent bid');
    }

    private function validateBid(Auction $auction, int $bidderId, float $amount): void
    {
        // সেলার নিজের আইটেমে বিড করতে পারবে না
        if ($auction->item->seller_id === $bidderId) {
            throw new \Exception('নিজের আইটেমে বিড করা যাবে না');
        }

        // মিনিমাম increment চেক
        $minIncrement = $this->getMinIncrement($auction->current_price);
        if ($amount < $auction->current_price + $minIncrement) {
            throw new \Exception("মিনিমাম বিড: ৳" . ($auction->current_price + $minIncrement));
        }

        // ফ্রড চেক — একই IP থেকে অনেক বিড
        $recentBids = Bid::where('auction_id', $auction->id)
            ->where('ip_address', request()->ip())
            ->where('bid_time', '>', now()->subMinutes(1))
            ->count();

        if ($recentBids > 5) {
            throw new \Exception('Too many bids — suspicious activity detected');
        }
    }

    /**
     * Minimum bid increment — প্রাইস অনুযায়ী increment বাড়ে
     */
    private function getMinIncrement(float $currentPrice): float
    {
        return match(true) {
            $currentPrice < 500    => 10,    // ৳১০ increment
            $currentPrice < 5000   => 50,    // ৳৫০
            $currentPrice < 50000  => 100,   // ৳১০০
            $currentPrice < 500000 => 500,   // ৳৫০০
            default                => 1000,  // ৳১০০০
        };
    }

    /**
     * Anti-snipe: শেষ ৫ মিনিটে বিড আসলে ৫ মিনিট extend
     */
    private function handleSnipeProtection(Auction $auction): void
    {
        $timeLeft = $auction->end_time->diffInMinutes(now());
        
        if ($timeLeft <= 5 && $auction->snipe_extension) {
            $auction->update([
                'end_time' => $auction->end_time->addMinutes(5)
            ]);

            // ব্রডকাস্ট — সময় বেড়েছে
            Redis::publish("auction:{$auction->id}", json_encode([
                'type'     => 'time_extended',
                'end_time' => $auction->end_time->toISOString(),
            ]));
        }
    }

    /**
     * Proxy Bidding — অটোমেটিক বিডিং সিস্টেম
     */
    private function processProxyBids(Auction $auction, Bid $newBid): void
    {
        $proxyBids = Bid::where('auction_id', $auction->id)
            ->where('bidder_id', '!=', $newBid->bidder_id)
            ->whereNotNull('max_amount')
            ->where('max_amount', '>', $newBid->amount)
            ->orderBy('max_amount', 'desc')
            ->first();

        if ($proxyBids) {
            $increment = $this->getMinIncrement($newBid->amount);
            $autoBidAmount = min(
                $newBid->amount + $increment,
                $proxyBids->max_amount
            );

            // Auto-bid place করো
            Bid::create([
                'auction_id' => $auction->id,
                'bidder_id'  => $proxyBids->bidder_id,
                'amount'     => $autoBidAmount,
                'is_proxy'   => true,
                'bid_time'   => now(),
            ]);

            Auction::where('id', $auction->id)->update([
                'current_price' => $autoBidAmount,
                'bid_count'     => DB::raw('bid_count + 1'),
            ]);
        }
    }
}
```

### 3. Real-time Bid Streaming — Node.js (WebSocket)

```javascript
// bid-realtime-server.js
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const Redis = require('ioredis');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
    cors: { origin: '*' },
    transports: ['websocket', 'polling'],
});

const redis = new Redis(process.env.REDIS_URL);
const redisSub = new Redis(process.env.REDIS_URL);

// Auction room tracking
const auctionViewers = new Map(); // auctionId -> Set of socketIds

/**
 * WebSocket connection handler
 * প্রতিটি ক্লায়েন্ট একটি নির্দিষ্ট অকশন room-এ join করে
 */
io.on('connection', (socket) => {
    console.log(`Client connected: ${socket.id}`);

    // অকশন room join
    socket.on('join_auction', async (auctionId) => {
        socket.join(`auction:${auctionId}`);
        
        // Viewer count track
        if (!auctionViewers.has(auctionId)) {
            auctionViewers.set(auctionId, new Set());
        }
        auctionViewers.get(auctionId).add(socket.id);

        // বর্তমান স্টেট পাঠাও
        const auctionState = await getAuctionState(auctionId);
        socket.emit('auction_state', auctionState);

        // Viewer count broadcast
        io.to(`auction:${auctionId}`).emit('viewer_count', {
            count: auctionViewers.get(auctionId).size,
        });
    });

    // বিড প্লেসমেন্ট (HTTP API থেকেও আসতে পারে)
    socket.on('place_bid', async (data) => {
        try {
            const { auctionId, amount, token } = data;
            
            // JWT verify
            const user = verifyToken(token);
            if (!user) {
                socket.emit('bid_error', { message: 'Unauthorized' });
                return;
            }

            // Redis-based rate limiting
            const rateKey = `bid_rate:${user.id}:${auctionId}`;
            const bidCount = await redis.incr(rateKey);
            if (bidCount === 1) await redis.expire(rateKey, 60);
            
            if (bidCount > 10) {
                socket.emit('bid_error', { 
                    message: 'খুব বেশি বিড — ১ মিনিট অপেক্ষা করুন' 
                });
                return;
            }

            // Bid queue-তে পাঠাও (ordered processing)
            await redis.rpush('bid_queue', JSON.stringify({
                auctionId,
                bidderId: user.id,
                amount,
                timestamp: Date.now(),
                socketId: socket.id,
            }));

        } catch (error) {
            socket.emit('bid_error', { message: error.message });
        }
    });

    socket.on('disconnect', () => {
        // Cleanup viewer tracking
        for (const [auctionId, viewers] of auctionViewers) {
            if (viewers.has(socket.id)) {
                viewers.delete(socket.id);
                io.to(`auction:${auctionId}`).emit('viewer_count', {
                    count: viewers.size,
                });
            }
        }
    });
});

/**
 * Redis Pub/Sub listener — Laravel থেকে bid events আসবে
 */
redisSub.subscribe('bid_events', 'auction_events');

redisSub.on('message', (channel, message) => {
    const data = JSON.parse(message);

    if (channel === 'bid_events') {
        // নতুন বিড — সবাইকে জানাও
        io.to(`auction:${data.auctionId}`).emit('new_bid', {
            bidId: data.bidId,
            amount: data.amount,
            bidderName: data.bidderName,
            bidCount: data.bidCount,
            timestamp: data.timestamp,
        });
    }

    if (channel === 'auction_events') {
        switch (data.type) {
            case 'time_extended':
                io.to(`auction:${data.auctionId}`).emit('time_extended', {
                    newEndTime: data.endTime,
                    reason: 'Anti-snipe: শেষ মুহূর্তে বিড এসেছে',
                });
                break;

            case 'auction_ended':
                io.to(`auction:${data.auctionId}`).emit('auction_ended', {
                    winnerId: data.winnerId,
                    winnerName: data.winnerName,
                    finalPrice: data.finalPrice,
                    totalBids: data.totalBids,
                });
                break;

            case 'buy_now':
                io.to(`auction:${data.auctionId}`).emit('auction_ended', {
                    type: 'buy_now',
                    buyerName: data.buyerName,
                    price: data.price,
                });
                break;
        }
    }
});

/**
 * Bid Queue Processor — FIFO order-এ বিড প্রসেস
 * Race condition এড়াতে sequential processing
 */
async function processBidQueue() {
    while (true) {
        const bidData = await redis.blpop('bid_queue', 1);
        if (!bidData) continue;

        const bid = JSON.parse(bidData[1]);
        
        try {
            // Lua script দিয়ে atomic bid validation + placement
            const result = await redis.eval(
                BID_PLACEMENT_LUA_SCRIPT,
                1,
                `auction:${bid.auctionId}`,
                bid.amount,
                bid.bidderId,
                Date.now()
            );

            if (result[0] === 1) {
                // Success — broadcast
                redis.publish('bid_events', JSON.stringify({
                    auctionId: bid.auctionId,
                    bidId: result[1],
                    amount: bid.amount,
                    bidderId: bid.bidderId,
                    timestamp: Date.now(),
                }));
            } else {
                // Failure — bidder কে জানাও
                io.to(bid.socketId).emit('bid_error', {
                    message: result[1],
                });
            }
        } catch (error) {
            console.error('Bid processing error:', error);
        }
    }
}

/**
 * Auction Timer Service — অকশন শেষ হলে winner ঘোষণা
 */
async function auctionTimerService() {
    setInterval(async () => {
        const endingAuctions = await redis.zrangebyscore(
            'auction_timers',
            0,
            Date.now()
        );

        for (const auctionId of endingAuctions) {
            await endAuction(auctionId);
            await redis.zrem('auction_timers', auctionId);
        }
    }, 1000); // প্রতি সেকেন্ডে চেক
}

async function getAuctionState(auctionId) {
    const cached = await redis.get(`auction_state:${auctionId}`);
    if (cached) return JSON.parse(cached);
    // DB fallback...
}

// Server start
const PORT = process.env.PORT || 3001;
httpServer.listen(PORT, () => {
    console.log(`🏷️ Auction Realtime Server running on port ${PORT}`);
    processBidQueue();
    auctionTimerService();
});
```

### 4. Auction Timer & Snipe Protection Flow

```
    Auction Timeline
    ═══════════════════════════════════════════════════════════════

    Start                                              Original End
    │                                                       │
    ├───────────────────────────────────────────────────────┤
    │                                                       │
    │   Normal bidding period                               │
    │                                                       │
    │                                          ┌─ 5min ─┐   │
    │                                          │ DANGER  │   │
    │                                          │  ZONE   │   │
    │                                          └────┬────┘   │
    │                                               │        │
    │                                         Snipe bid!     │
    │                                               │        │
    │                                               ▼        │
    │                                          ┌─────────────┤──── +5min ────┐
    │                                          │ EXTENDED!   │               │
    │                                          └─────────────┘   New End     │
    │                                                            Time        │
    ═══════════════════════════════════════════════════════════════════════════

    Anti-Snipe Rule:
    - শেষ ৫ মিনিটে বিড আসলে → ৫ মিনিট বাড়ে
    - Maximum ৩ বার extend হবে (total +15min)
    - এতে last-second sniping আর কাজ করে না
```

### 5. Payment Escrow Flow

```
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │  BUYER   │        │  ESCROW  │        │  SELLER  │
    │          │        │  SYSTEM  │        │          │
    └────┬─────┘        └────┬─────┘        └────┬─────┘
         │                   │                   │
         │─── Pay (bKash) ──▶│                   │
         │                   │── Hold funds ──┐  │
         │                   │◀───────────────┘  │
         │◀── Payment held ──│                   │
         │                   │── Notify seller ─▶│
         │                   │                   │
         │                   │         Ship item │
         │◀──────────────────│───────────────────│
         │                   │                   │
         │── Confirm recv ──▶│                   │
         │                   │── Release fund ──▶│
         │                   │                   │
         │                   │   ┌─────────┐     │
         │                   │   │ DISPUTE │     │
         │── Open dispute ──▶│   │  PATH   │     │
         │                   │   └────┬────┘     │
         │                   │        │          │
         │                   │── Investigate ───▶│
         │                   │        │          │
         │◀── Refund/Release─│◀───────┘          │
         │                   │                   │
    
    Timeline:
    ┌─────────────────────────────────────────────────────┐
    │ Day 0: Payment held                                 │
    │ Day 1-3: Seller ships                               │
    │ Day 3-7: Delivery                                   │
    │ Day 7: Auto-confirm (buyer didn't dispute)          │
    │ Day 7: Funds released to seller                     │
    │                                                     │
    │ Dispute window: 7 days from delivery                │
    │ Auto-release: 14 days if no action                  │
    └─────────────────────────────────────────────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Optimistic vs Pessimistic Locking

```
┌────────────────────┬────────────────────────────┬────────────────────────────┐
│                    │ OPTIMISTIC LOCKING         │ PESSIMISTIC LOCKING        │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ কীভাবে কাজ করে    │ Version number চেক করে     │ Row-level lock ধরে রাখে    │
│                    │ conflict হলে retry         │ transaction শেষ না হওয়া    │
│                    │                            │ পর্যন্ত অন্যরা wait করে   │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ Throughput         │ ✅ High (no waiting)        │ ❌ Low (blocking)           │
│ Consistency        │ ✅ Eventual (retry needed)  │ ✅ Strong (guaranteed)      │
│ Contention (high)  │ ❌ Many retries             │ ✅ Ordered, no retry        │
│ Contention (low)   │ ✅ Best performance         │ ❌ Unnecessary overhead     │
│ Deadlock risk      │ ✅ None                     │ ❌ Possible                 │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ Best for           │ বেশিরভাগ auctions          │ Celebrity auction          │
│                    │ (low-medium contention)    │ (extreme contention)       │
├────────────────────┼────────────────────────────┼────────────────────────────┤
│ 🏆 আমাদের পছন্দ   │ ✅ Default choice           │ Hot auction fallback       │
└────────────────────┴────────────────────────────┴────────────────────────────┘
```

### 2. SQL vs NoSQL for Bid History

```
┌────────────────────┬────────────────────────┬────────────────────────────┐
│                    │ PostgreSQL (SQL)        │ Cassandra/DynamoDB (NoSQL) │
├────────────────────┼────────────────────────┼────────────────────────────┤
│ ACID Compliance    │ ✅ Full ACID             │ ❌ Eventual consistency     │
│ Write Throughput   │ ⚠️ Limited (single node) │ ✅ Linear scalability       │
│ Complex Queries    │ ✅ JOINs, aggregations   │ ❌ Limited query patterns   │
│ Bid Analytics      │ ✅ Easy SQL analytics    │ ❌ Needs separate pipeline  │
│ Horizontal Scale   │ ⚠️ Read replicas only    │ ✅ Natural sharding         │
├────────────────────┼────────────────────────┼────────────────────────────┤
│ 🏆 আমাদের পছন্দ   │ Active bids (hot data)  │ Historical bids (cold)     │
└────────────────────┴────────────────────────┴────────────────────────────┘

Strategy: Hybrid approach
- PostgreSQL: Active auctions + recent bids (strong consistency)
- Cassandra: Archived bid history (write-heavy, analytics)
- Redis: Current auction state cache (ultra-fast reads)
```

### 3. WebSocket vs SSE vs Long Polling

```
┌────────────────────┬─────────────────┬─────────────────┬─────────────────┐
│                    │ WebSocket       │ SSE             │ Long Polling    │
├────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Direction          │ Bidirectional   │ Server → Client │ Client → Server │
│ Latency            │ ~10ms           │ ~50ms           │ ~500ms          │
│ Connection cost    │ High (stateful) │ Medium          │ Low             │
│ বাংলাদেশ 3G/4G    │ ⚠️ Reconnection  │ ✅ Auto-reconnect│ ✅ Works well    │
│ Scalability        │ ⚠️ Sticky session│ ✅ Stateless     │ ✅ Stateless     │
│ Bid placement      │ ✅ Yes           │ ❌ No            │ ✅ Via POST      │
├────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 🏆 আমাদের পছন্দ   │ ✅ Primary       │ Fallback        │ Legacy/3G       │
└────────────────────┴─────────────────┴─────────────────┴─────────────────┘

Decision:
- WebSocket: Default (modern browsers, stable connection)
- SSE: Fallback (WebSocket fail হলে)
- Long Polling: Bangladesh rural areas (unstable 3G)
```

### 4. Synchronous vs Async Bid Processing

```
┌────────────────────┬──────────────────────────┬──────────────────────────┐
│                    │ SYNCHRONOUS              │ ASYNC (Queue-based)      │
├────────────────────┼──────────────────────────┼──────────────────────────┤
│ Response Time      │ ✅ Immediate feedback     │ ⚠️ Delayed confirmation   │
│ Throughput         │ ❌ Limited by DB          │ ✅ Queue absorbs spikes   │
│ Order Guarantee    │ ⚠️ Race conditions        │ ✅ FIFO processing        │
│ Complexity         │ ✅ Simple                  │ ⚠️ Queue + worker setup  │
│ Failure Handling   │ ❌ Lost on crash          │ ✅ Retry from queue       │
├────────────────────┼──────────────────────────┼──────────────────────────┤
│ 🏆 আমাদের পছন্দ   │ Normal auctions          │ High-traffic auctions    │
└────────────────────┴──────────────────────────┴──────────────────────────┘

Hybrid Strategy:
- < 100 bids/sec: Synchronous (simple, fast feedback)
- > 100 bids/sec: Auto-switch to queue (Redis RPUSH → worker)
- Client দেখে "Bid received, confirming..." → তারপর confirm/reject
```

---

## 📈 কেস স্টাডি

### কেস ১: Last-Minute Bidding War (Auction Sniping)

```
সিনারিও: একটি rare iPhone 16 Pro Max — শেষ ৩০ সেকেন্ডে ২০টি বিড

Timeline:
════════════════════════════════════════════════════════════════
T-30s: ৳85,000 (Bidder A)
T-25s: ৳86,000 (Bidder B)
T-20s: ৳87,000 (Bidder C) ← proxy bid active (max ৳95,000)
T-15s: ৳88,000 (Bidder A)
T-12s: ৳89,000 (AUTO - Bidder C proxy)
T-10s: ৳90,000 (Bidder D - নতুন bidder!)
T-8s:  ৳91,000 (AUTO - Bidder C proxy)
T-5s:  ৳95,000 (Bidder A - বড় jump!)   ← SNIPE ZONE!
                                          ┌──────────────────┐
T-5s:  ANTI-SNIPE ACTIVATED!             │ +5 min extended! │
                                          └──────────────────┘
T+1m:  ৳96,000 (Bidder C - proxy exhausted, manual bid)
T+3m:  ৳97,000 (Bidder A)
T+5m:  AUCTION ENDS — Winner: Bidder A @ ৳97,000

সমস্যা: Anti-snipe ছাড়া Bidder A ৳85,001 দিয়ে জিততে পারত!
সমাধান: Snipe protection fair bidding নিশ্চিত করে
```

### কেস ২: Celebrity Auction — 1M Concurrent Viewers

```
সিনারিও: সাকিব আল হাসানের match-worn jersey auction
- 1M concurrent WebSocket connections
- Peak: 10,000 bids/second
- Duration: 3 days

Architecture Response:
┌─────────────────────────────────────────────────────────────┐
│                    SCALING STRATEGY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐     ┌─────────────────────┐                   │
│  │   CDN   │────▶│ WebSocket Cluster    │                   │
│  │(Static) │     │ (20 nodes, sticky)   │                   │
│  └─────────┘     └──────────┬──────────┘                   │
│                             │                               │
│                    ┌────────▼────────┐                      │
│                    │  Redis Pub/Sub  │                      │
│                    │  (Cluster mode) │                      │
│                    └────────┬────────┘                      │
│                             │                               │
│              ┌──────────────┼──────────────┐                │
│              ▼              ▼              ▼                │
│     ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│     │ Bid      │   │ Bid      │   │ Bid      │            │
│     │ Worker 1 │   │ Worker 2 │   │ Worker N │            │
│     └──────────┘   └──────────┘   └──────────┘            │
│                             │                               │
│                    ┌────────▼────────┐                      │
│                    │  PostgreSQL     │                      │
│                    │  (Write master  │                      │
│                    │   + 5 replicas) │                      │
│                    └─────────────────┘                      │
│                                                             │
│  Optimizations:                                             │
│  • Bid batching: 100ms window-তে bids একসাথে process       │
│  • Read replicas: Viewer-রা replica থেকে পড়ে               │
│  • Circuit breaker: DB overwhelm হলে queue-তে buffer       │
│  • Graceful degradation: viewer count approx দেখায়         │
└─────────────────────────────────────────────────────────────┘
```

### কেস ৩: Shill Bidding Fraud Detection

```
সিনারিও: সেলার নিজেই fake account দিয়ে bid করছে (দাম বাড়াতে)

Detection Signals:
┌──────────────────────────────────────────────────────────────┐
│  Signal                          │ Weight │ Threshold        │
├──────────────────────────────────┼────────┼──────────────────┤
│  Same IP as seller               │  0.9   │ Auto-block       │
│  New account (< 7 days)          │  0.3   │ Flag for review  │
│  Always bids on same seller      │  0.7   │ Pattern alert    │
│  Never wins (inflates price)     │  0.6   │ Statistical      │
│  Bid just above previous         │  0.4   │ Minimum pattern  │
│  Same device fingerprint         │  0.8   │ Auto-block       │
│  Unusual bid timing pattern      │  0.5   │ ML detection     │
└──────────────────────────────────┴────────┴──────────────────┘

Prevention System:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Real-time scoring: প্রতিটি বিডে fraud score calculate
2. Threshold: score > 0.7 → bid hold + manual review  
3. Historical: ML model — past fraud patterns learn
4. Penalty: Account ban + seller rating damage
5. NID verification: বাংলাদেশে NID mandatory করা (unique identity)
```

---

## 🔧 Advanced Topics

### 1. Reserve Price & Dutch Auction

```
Standard Auction (English):
  ৳1,000 → ৳2,000 → ৳3,000 → ... → ৳10,000 (Winner!)
  দাম বাড়তে থাকে, সবচেয়ে বেশি দাম দেয় সে জেতে

Dutch Auction (Reverse):
  ৳10,000 → ৳9,000 → ৳8,000 → ... → ৳5,000 (First claim!)
  দাম কমতে থাকে, প্রথম যে claim করে সে জেতে
  Use case: পচনশীল পণ্য (ফুল, মাছ), government surplus

Reserve Price:
  ┌─────────────────────────────────────────────────┐
  │  Starting: ৳1,000                               │
  │  Reserve:  ৳5,000 (hidden from bidders)         │
  │                                                 │
  │  Scenario 1: Final bid ৳7,000 > Reserve        │
  │  Result: ✅ SOLD at ৳7,000                      │
  │                                                 │
  │  Scenario 2: Final bid ৳4,000 < Reserve        │
  │  Result: ❌ NOT SOLD — "Reserve not met"        │
  │  Seller can: Accept anyway / Relist / Counter   │
  └─────────────────────────────────────────────────┘
```

### 2. Recommendation Engine

```
┌─────────────────────────────────────────────────────────┐
│              RECOMMENDATION PIPELINE                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User Behavior         →  Feature Extraction            │
│  ┌──────────────┐         ┌──────────────────┐         │
│  │ Bid history  │───────▶│ Category affinity │         │
│  │ Watch list   │───────▶│ Price range      │         │
│  │ Search query │───────▶│ Brand preference │         │
│  │ Browse time  │───────▶│ Time patterns    │         │
│  └──────────────┘         └────────┬─────────┘         │
│                                    │                    │
│                           ┌────────▼─────────┐         │
│                           │  ML Model        │         │
│                           │  (Collaborative  │         │
│                           │   Filtering +    │         │
│                           │   Content-based) │         │
│                           └────────┬─────────┘         │
│                                    │                    │
│                           ┌────────▼─────────┐         │
│                           │  Ranked Results  │         │
│                           │  "আপনার জন্য"    │         │
│                           └──────────────────┘         │
│                                                         │
│  Bangladesh Context:                                    │
│  - Ramadan-এ ইসলামিক আইটেম boost                       │
│  - পহেলা বৈশাখে traditional items                      │
│  - Cricket season-এ sports memorabilia                  │
│  - Location-based: ঢাকা vs চট্টগ্রাম preference        │
└─────────────────────────────────────────────────────────┘
```

### 3. Trust & Reputation System

```php
<?php
// Trust Score Calculation — Laravel Service

class TrustScoreService
{
    /**
     * User trust score (0.0 - 5.0)
     * 
     * বিশ্বাসযোগ্যতা স্কোর গণনা:
     * - Transaction history
     * - Dispute rate
     * - Response time
     * - NID verification
     * - Account age
     */
    public function calculateTrustScore(User $user): float
    {
        $weights = [
            'transaction_success' => 0.30,  // সফল লেনদেন
            'dispute_rate'        => 0.25,  // বিতর্কের হার (inverse)
            'response_time'       => 0.15,  // রেসপন্স সময়
            'verification'        => 0.15,  // NID/Phone verified
            'account_age'         => 0.10,  // অ্যাকাউন্টের বয়স
            'community_reports'   => 0.05,  // রিপোর্ট সংখ্যা (inverse)
        ];

        $scores = [
            'transaction_success' => $this->getTransactionScore($user),
            'dispute_rate'        => $this->getDisputeScore($user),
            'response_time'       => $this->getResponseScore($user),
            'verification'        => $this->getVerificationScore($user),
            'account_age'         => $this->getAgeScore($user),
            'community_reports'   => $this->getReportScore($user),
        ];

        $totalScore = 0;
        foreach ($weights as $factor => $weight) {
            $totalScore += $scores[$factor] * $weight;
        }

        return round($totalScore * 5, 2); // 0.0 - 5.0 scale
    }

    private function getVerificationScore(User $user): float
    {
        $score = 0;
        if ($user->email_verified) $score += 0.2;
        if ($user->phone_verified) $score += 0.3;   // bKash linked
        if ($user->nid_verified)   $score += 0.5;   // NID verified
        return $score;
    }
}
```

### 4. Cross-border Auction (বাংলাদেশ → Global)

```
┌─────────────────────────────────────────────────────────────┐
│            CROSS-BORDER AUCTION CHALLENGES                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Currency Conversion:                                       │
│  ┌─────────────┐      ┌────────────┐      ┌────────────┐  │
│  │ BDT (৳)     │─────▶│ Real-time  │─────▶│ USD ($)    │  │
│  │ Seller sees │      │ FX Rate    │      │ Buyer sees │  │
│  └─────────────┘      └────────────┘      └────────────┘  │
│                                                             │
│  Shipping Complexity:                                       │
│  - Bangladesh → International: 7-21 days                    │
│  - Customs duty calculation (auto)                          │
│  - Prohibited items (leather restrictions in some countries)│
│                                                             │
│  Payment:                                                   │
│  - International: Stripe, PayPal                           │
│  - Bangladesh domestic: bKash, Nagad, DBBL Nexus           │
│  - Escrow period: Extended for international (21 days)      │
│                                                             │
│  Legal Compliance:                                          │
│  - Bangladesh Bank regulations (foreign currency)           │
│  - Export license for certain goods                          │
│  - VAT/Tax calculation by destination                       │
└─────────────────────────────────────────────────────────────┘
```

### 5. Search & Discovery Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                SEARCH ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Query: "iPhone 15 Pro Max ঢাকা"                       │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────┐                                       │
│  │ Query Parser     │ → Language detect (Bangla + English)  │
│  │                  │ → Tokenize + Normalize                │
│  │                  │ → Spelling correction                 │
│  └────────┬─────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌──────────────────┐                                       │
│  │ Elasticsearch    │                                       │
│  │                  │                                       │
│  │ • Full-text search (Bangla analyzer)                    │
│  │ • Faceted filters (category, price, location)           │
│  │ • Geo-distance (ঢাকার কাছে)                            │
│  │ • Auction status filter (active only)                   │
│  │ • Relevance scoring                                     │
│  └────────┬─────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌──────────────────┐                                       │
│  │ Re-ranking       │                                       │
│  │                  │                                       │
│  │ • Ending soon boost (urgency)                           │
│  │ • Trust score of seller                                  │
│  │ • Bid activity (popular items)                          │
│  │ • Personalization (user history)                        │
│  └──────────────────┘                                       │
│                                                             │
│  Bangla Search Challenges:                                  │
│  - "আইফোন" = "iPhone" = "iphone" → synonym mapping        │
│  - "মোবাইল" = "phone" = "ফোন" → alias                     │
│  - বাংলা stemming (limited NLP tools)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 সারসংক্ষেপ

### Key Decisions Summary

```
┌─────────────────────────────────────────────────────────────┐
│              ARCHITECTURE DECISION RECORD                    │
├─────────────────────────┬───────────────────────────────────┤
│ Decision                │ Choice & Reason                   │
├─────────────────────────┼───────────────────────────────────┤
│ Bid consistency         │ Optimistic locking + retry        │
│ Real-time transport     │ WebSocket (SSE fallback)          │
│ Bid processing          │ Hybrid sync/async (load-based)    │
│ Primary DB              │ PostgreSQL (ACID for bids)        │
│ Cache layer             │ Redis (auction state + pub/sub)   │
│ Search                  │ Elasticsearch (Bangla support)    │
│ Payment                 │ Escrow model (bKash + Stripe)     │
│ Fraud detection         │ ML scoring + rule engine          │
│ Timer management        │ Redis sorted sets + workers       │
│ Image storage           │ S3 + CloudFront CDN              │
│ Notification            │ Firebase + SMS (Bangladesh)       │
│ Anti-snipe              │ 5-min extension (max 3x)          │
└─────────────────────────┴───────────────────────────────────┘
```

### বাংলাদেশ-Specific Considerations

```
┌─────────────────────────────────────────────────────────────┐
│  🇧🇩 বাংলাদেশ স্পেশাল                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Payment Integration:                                    │
│     bKash, Nagad, Rocket → escrow via API                  │
│                                                             │
│  2. Identity Verification:                                  │
│     NID + Porichoy API → real identity confirm              │
│                                                             │
│  3. Network Resilience:                                     │
│     Offline bid queue → sync when online                    │
│     Progressive image loading (slow 3G)                     │
│                                                             │
│  4. Language:                                               │
│     Bilingual UI (বাংলা/English toggle)                     │
│     Bangla search with transliteration                      │
│                                                             │
│  5. Legal:                                                  │
│     Digital Commerce Act 2023 compliance                    │
│     Consumer protection (7-day return for online)           │
│     VAT on platform commission (15%)                        │
│                                                             │
│  6. Delivery:                                               │
│     Pathao/RedX/Steadfast courier integration               │
│     Cash on Delivery option (with partial escrow)           │
│                                                             │
│  7. Trust Building:                                         │
│     Video verification for high-value items                 │
│     Local pickup option (ঢাকার মধ্যে)                      │
│     Community-based reputation (Facebook link)              │
└─────────────────────────────────────────────────────────────┘
```

### Interview Tips

```
অকশন সিস্টেম ডিজাইন ইন্টারভিউতে যা মনে রাখবেন:

✅ DO:
  • Race condition সমাধান দেখান (optimistic/pessimistic locking)
  • Real-time architecture explain করুন (WebSocket + Redis pub/sub)
  • Anti-snipe mechanism — fairness নিশ্চিত করে
  • Fraud detection — shill bidding ধরার পদ্ধতি
  • Payment escrow — trust তৈরি করে
  • Scale calculation — back-of-envelope estimation

❌ DON'T:
  • Race condition ignore করবেন না — এটাই মূল চ্যালেঞ্জ
  • শুধু CRUD API দেখাবেন না — real-time + consistency দেখান
  • Single point of failure রাখবেন না
  • Timer management সহজ মনে করবেন না — distributed timer কঠিন

🎯 Key Insight:
  "অকশন সিস্টেমের সবচেয়ে কঠিন সমস্যা হলো consistency —
   দুইজন একই সময়ে বিড করলে কে জিতবে? এটা solve করতে পারলে
   বাকিটা সহজ।"
```

---

> **📚 আরও পড়ুন**: [eBay Architecture](https://www.ebayinc.com/stories/blogs/tech/), [Auction Theory (Wikipedia)](https://en.wikipedia.org/wiki/Auction_theory)
> **🔗 Related**: Payment System Design, Real-time Notification System, Fraud Detection ML Pipeline
