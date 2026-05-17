# 🎬 টিকেটিং সিস্টেম ডিজাইন (BookMyShow)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: BookMyShow, Ticketmaster, Shohoz.com
> **বাংলাদেশ কনটেক্সট**: Star Cineplex, Blockbuster Cinemas, Shyamoli Paribahan, bKash Payment

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#️-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ-tradeoffs)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

```
✅ ইউজার হিসেবে আমি চাই:
├── সিনেমা/ইভেন্ট ব্রাউজ করতে পারি (Star Cineplex, Blockbuster)
├── শো-এর সময়সূচী দেখতে পারি
├── সিট সিলেক্ট করতে পারি (seat map দেখে)
├── সিট লক করতে পারি (অন্য কেউ যেন একই সিট না পায়)
├── পেমেন্ট করতে পারি (bKash, Nagad, Card)
├── বুকিং কনফার্মেশন পাই (SMS + Email)
├── টিকেট ক্যান্সেল/রিফান্ড করতে পারি
└── বাস টিকেট বুক করতে পারি (Shyamoli, Greenline - Shohoz style)

✅ অ্যাডমিন হিসেবে আমি চাই:
├── নতুন শো/ইভেন্ট যোগ করতে পারি
├── সিট ম্যাপ কনফিগার করতে পারি
├── প্রাইসিং সেট করতে পারি (Dynamic pricing)
└── রিপোর্ট ও অ্যানালিটিক্স দেখতে পারি
```

### Non-Functional Requirements

| Requirement | বিবরণ | Target |
|-------------|--------|--------|
| **Consistency** | ডাবল বুকিং কখনোই হতে পারবে না! | Strong consistency |
| **Availability** | সিস্টেম সবসময় চালু থাকবে | 99.99% uptime |
| **Low Latency** | সিট সিলেকশন রিয়েল-টাইম | < 200ms response |
| **Scalability** | ঈদের দিন ১ লক্ষ concurrent user | Horizontal scaling |
| **Durability** | পেমেন্ট ডাটা কখনো হারাবে না | Multi-region backup |

### 🇧🇩 বাংলাদেশ কনটেক্সট

```
রিয়েল-ওয়ার্ল্ড সিনারিও:
├── Star Cineplex (Bashundhara, Shimanto) - ঈদের দিন "শকুন" রিলিজ
├── Blockbuster Cinemas - প্রিমিয়ার শো
├── BPL Cricket Final - Mirpur Stadium ৩০,০০০ সিট
├── Shyamoli Paribahan - ঢাকা-চট্টগ্রাম বাস টিকেট (ঈদে পাগলামি!)
├── Shohoz.com - অনলাইন বাস/ট্রেন টিকেট
├── bKash/Nagad Payment - মোবাইল ফিনান্সিয়াল সার্ভিস
└── Bangladesh Railway - অনলাইন ট্রেন টিকেট (esheba.cbtservice.gov.bd)
```

---

## 📊 Back-of-envelope Estimation

### Traffic Estimation (বাংলাদেশ কনটেক্সট)

```
ধরি Shohoz-এর মতো একটি সিস্টেম:

Daily Active Users (DAU):
├── Normal day: ৫০,০০০ users
├── Weekend: ১,০০,০০০ users
├── Eid/BPL Final: ৫,০০,০০০+ users (10x spike!)
└── Peak concurrent: ১,০০,০০০ users

Bookings per day:
├── Normal: ২০,০০০ bookings/day
├── Peak: ২,০০,০০০ bookings/day
└── Per second (peak): ~2,300 bookings/sec

Read:Write ratio = 100:1 (বেশিরভাগ মানুষ শুধু ব্রাউজ করে)
├── Reads: ১০,০০০ requests/sec (peak)
└── Writes: ১০০ requests/sec (peak, normal day)
```

### Storage Estimation

```
Per Booking Record: ~2KB
├── User info: 200 bytes
├── Show/Event info: 300 bytes
├── Seat info: 200 bytes
├── Payment info: 500 bytes
├── Metadata: 300 bytes
└── QR/Barcode: 500 bytes

Daily Storage:
├── ২০,০০০ bookings × 2KB = 40MB/day
├── Monthly: ~1.2GB
├── Yearly: ~15GB (শুধু bookings)
└── Total (with media, logs): ~500GB/year

Database Size (5 years):
├── Bookings: 75GB
├── Users: 20GB
├── Events/Shows: 10GB
├── Payments: 50GB
└── Total: ~150GB (manageable with single powerful DB)
```

### Bandwidth Estimation

```
Incoming (Peak):
├── ১০,০০০ req/sec × 1KB avg = 10MB/sec
└── Seat map images: additional 5MB/sec

Outgoing (Peak):
├── Seat availability responses: 20MB/sec
├── Booking confirmations: 2MB/sec
└── WebSocket updates: 5MB/sec

Total bandwidth needed: ~50MB/sec (peak)
```

---

## 🏗️ High-Level Design

### System Architecture (ASCII Diagram)

```
                         ┌─────────────────────────────────────────────────┐
                         │              CLIENT LAYER                        │
                         │  ┌──────┐  ┌──────────┐  ┌───────────────────┐ │
                         │  │Mobile│  │  Web App  │  │ Shohoz/Star App   │ │
                         │  │ App  │  │ (React)   │  │  (Partner APIs)   │ │
                         │  └──┬───┘  └────┬──────┘  └────────┬──────────┘ │
                         └─────┼───────────┼──────────────────┼────────────┘
                               │           │                  │
                         ┌─────▼───────────▼──────────────────▼────────────┐
                         │            LOAD BALANCER (Nginx/HAProxy)         │
                         └─────────────────────┬───────────────────────────┘
                                               │
                         ┌─────────────────────▼───────────────────────────┐
                         │              API GATEWAY (Kong/Laravel)          │
                         │  ┌─────────────────────────────────────────┐    │
                         │  │ Rate Limiting │ Auth │ Routing │ Logging │    │
                         │  └─────────────────────────────────────────┘    │
                         └──┬──────────┬──────────┬──────────┬─────────────┘
                            │          │          │          │
              ┌─────────────▼──┐ ┌─────▼────┐ ┌──▼───────┐ ┌▼──────────────┐
              │  SEARCH/BROWSE │ │ BOOKING  │ │ PAYMENT  │ │ NOTIFICATION  │
              │   SERVICE      │ │ SERVICE  │ │ SERVICE  │ │   SERVICE     │
              │  (Laravel)     │ │ (Laravel)│ │ (Node.js)│ │  (Node.js)    │
              └───────┬────────┘ └────┬─────┘ └────┬─────┘ └───────┬───────┘
                      │               │             │               │
                      │         ┌─────▼─────┐      │               │
                      │         │ SEAT LOCK  │      │               │
                      │         │  SERVICE   │      │               │
                      │         │  (Redis)   │      │               │
                      │         └─────┬──────┘      │               │
                      │               │             │               │
              ┌───────▼───────────────▼─────────────▼───────────────▼──────┐
              │                    DATA LAYER                                │
              │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌──────────────┐  │
              │  │PostgreSQL│  │  Redis    │  │  Kafka │  │ Elasticsearch│  │
              │  │(Bookings)│  │ (Locks/  │  │(Events)│  │  (Search)    │  │
              │  │          │  │  Cache)   │  │        │  │              │  │
              │  └──────────┘  └──────────┘  └────────┘  └──────────────┘  │
              └─────────────────────────────────────────────────────────────┘
```

### Read/Write Path Separation (CQRS Pattern)

```
READ PATH (সিট দেখা, সার্চ):
┌──────┐     ┌───────────┐     ┌──────────────┐     ┌───────────┐
│Client│────▶│API Gateway│────▶│ Read Replica │────▶│   Cache   │
│      │◀────│           │◀────│ (PostgreSQL) │◀────│  (Redis)  │
└──────┘     └───────────┘     └──────────────┘     └───────────┘

WRITE PATH (বুকিং, পেমেন্ট):
┌──────┐     ┌───────────┐     ┌──────────────┐     ┌───────────┐
│Client│────▶│API Gateway│────▶│Booking Service│────▶│  Primary  │
│      │◀────│           │◀────│+ Seat Lock    │◀────│    DB     │
└──────┘     └───────────┘     └──────────────┘     └───────────┘
                                       │
                                       ▼
                               ┌──────────────┐
                               │ Event Queue  │
                               │   (Kafka)    │
                               └──────┬───────┘
                                      │
                          ┌───────────┼───────────┐
                          ▼           ▼           ▼
                   ┌──────────┐ ┌──────────┐ ┌─────────┐
                   │Notification│ │Analytics│ │  Cache  │
                   │  Service  │ │ Service │ │Invalidate│
                   └──────────┘ └──────────┘ └─────────┘
```

### Booking Flow (Step by Step)

```
User Journey (Star Cineplex তে সিনেমা দেখতে যাওয়া):

┌─────┐  ১. শো সিলেক্ট  ┌──────────┐  ২. সিট দেখাও  ┌──────────┐
│User │─────────────────▶│  Search  │───────────────▶│  Seat    │
│     │                  │ Service  │                │  Service │
└─────┘                  └──────────┘                └────┬─────┘
                                                          │
   ┌──────────────────────────────────────────────────────┘
   │ ৩. সিট সিলেক্ট + লক (5 min TTL)
   ▼
┌──────────┐  ৪. পেমেন্ট  ┌──────────┐  ৫. কনফার্ম   ┌──────────┐
│Seat Lock │─────────────▶│ Payment  │──────────────▶│ Booking  │
│ (Redis)  │              │ (bKash)  │               │ Confirm  │
└──────────┘              └──────────┘               └────┬─────┘
                                                          │
                                                          ▼
                                                   ┌──────────┐
                                                   │SMS + QR  │
                                                   │ টিকেট!   │
                                                   └──────────┘
```

---

## 💻 Detailed Design

### ১. Database Schema

```sql
-- থিয়েটার/ভেন্যু (Star Cineplex, Mirpur Stadium)
CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,        -- 'Star Cineplex Bashundhara'
    city VARCHAR(100) NOT NULL,        -- 'Dhaka'
    address TEXT,
    total_screens INTEGER DEFAULT 1,   -- সিনেমার জন্য
    total_capacity INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- স্ক্রিন/হল
CREATE TABLE screens (
    id BIGSERIAL PRIMARY KEY,
    venue_id BIGINT REFERENCES venues(id),
    name VARCHAR(100),                 -- 'Hall 1', 'IMAX'
    total_seats INTEGER NOT NULL,
    screen_type VARCHAR(50),           -- '2D', '3D', 'IMAX'
    created_at TIMESTAMP DEFAULT NOW()
);

-- সিট ম্যাপ
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    screen_id BIGINT REFERENCES screens(id),
    row_label CHAR(1) NOT NULL,        -- 'A', 'B', 'C'
    seat_number INTEGER NOT NULL,      -- 1, 2, 3...
    seat_type VARCHAR(50) NOT NULL,    -- 'regular', 'premium', 'couple'
    price_multiplier DECIMAL(3,2) DEFAULT 1.00,
    is_active BOOLEAN DEFAULT TRUE,
    UNIQUE(screen_id, row_label, seat_number)
);

-- ইভেন্ট/শো (সিনেমা, ক্রিকেট, কনসার্ট)
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,       -- 'শকুন - Eid Special'
    type VARCHAR(50) NOT NULL,         -- 'movie', 'cricket', 'concert', 'bus'
    description TEXT,
    duration_minutes INTEGER,
    language VARCHAR(50),              -- 'Bangla', 'Hindi', 'English'
    created_at TIMESTAMP DEFAULT NOW()
);

-- শো টাইম
CREATE TABLE shows (
    id BIGSERIAL PRIMARY KEY,
    event_id BIGINT REFERENCES events(id),
    screen_id BIGINT REFERENCES screens(id),
    show_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    base_price DECIMAL(10,2) NOT NULL, -- ৳350, ৳500
    status VARCHAR(20) DEFAULT 'active', -- 'active', 'cancelled', 'completed'
    created_at TIMESTAMP DEFAULT NOW()
);

-- বুকিং
CREATE TABLE bookings (
    id BIGSERIAL PRIMARY KEY,
    booking_code VARCHAR(20) UNIQUE NOT NULL, -- 'BMS-2024-ABC123'
    user_id BIGINT NOT NULL,
    show_id BIGINT REFERENCES shows(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    -- 'pending', 'confirmed', 'cancelled', 'expired'
    payment_method VARCHAR(50),        -- 'bkash', 'nagad', 'card', 'cash'
    booked_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,              -- Lock expiry
    confirmed_at TIMESTAMP,
    cancelled_at TIMESTAMP
);

-- বুক করা সিট (Many-to-Many)
CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT REFERENCES bookings(id),
    seat_id BIGINT REFERENCES seats(id),
    show_id BIGINT REFERENCES shows(id),
    price DECIMAL(10,2) NOT NULL,
    UNIQUE(seat_id, show_id)           -- এক সিট এক শো-তে একবারই!
);

-- পেমেন্ট
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT REFERENCES bookings(id),
    amount DECIMAL(10,2) NOT NULL,
    method VARCHAR(50) NOT NULL,       -- 'bkash', 'nagad', 'visa', 'mastercard'
    transaction_id VARCHAR(255),       -- bKash TrxID
    status VARCHAR(20) DEFAULT 'pending',
    -- 'pending', 'completed', 'failed', 'refunded'
    gateway_response JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

-- Critical Index for preventing double booking
CREATE UNIQUE INDEX idx_no_double_booking
ON booking_seats(seat_id, show_id)
WHERE booking_id IN (
    SELECT id FROM bookings WHERE status IN ('pending', 'confirmed')
);
```

### ২. Seat Locking Mechanism - PHP (Laravel)

```php
<?php
// app/Services/SeatLockService.php

namespace App\Services;

use Illuminate\Support\Facades\Redis;
use App\Exceptions\SeatAlreadyLockedException;
use App\Models\Booking;
use App\Models\Seat;

class SeatLockService
{
    // সিট লক TTL - ৫ মিনিট (পেমেন্ট করার জন্য সময়)
    private const LOCK_TTL = 300; // seconds

    /**
     * সিট লক করা - Pessimistic Locking with Redis
     * 
     * যখন কেউ Star Cineplex এ সিট সিলেক্ট করে,
     * আমরা সেই সিট ৫ মিনিটের জন্য লক করি
     */
    public function lockSeats(int $showId, array $seatIds, int $userId): string
    {
        $lockKeys = [];
        $bookingCode = $this->generateBookingCode();

        // Redis MULTI/EXEC দিয়ে atomic operation
        $pipe = Redis::pipeline();

        foreach ($seatIds as $seatId) {
            $lockKey = "seat_lock:{$showId}:{$seatId}";
            $lockKeys[] = $lockKey;

            // চেক করি সিট already লক আছে কিনা
            if (Redis::exists($lockKey)) {
                $lockedBy = Redis::hget($lockKey, 'user_id');
                if ($lockedBy != $userId) {
                    throw new SeatAlreadyLockedException(
                        "সিট #{$seatId} ইতিমধ্যে অন্য কেউ সিলেক্ট করেছে! অন্য সিট বেছে নিন।"
                    );
                }
            }
        }

        // সব সিট একসাথে লক করি (atomic)
        foreach ($seatIds as $seatId) {
            $lockKey = "seat_lock:{$showId}:{$seatId}";

            // Lua script দিয়ে atomic SET-IF-NOT-EXISTS
            $luaScript = <<<'LUA'
                if redis.call('exists', KEYS[1]) == 0 then
                    redis.call('hset', KEYS[1], 'user_id', ARGV[1])
                    redis.call('hset', KEYS[1], 'booking_code', ARGV[2])
                    redis.call('hset', KEYS[1], 'locked_at', ARGV[3])
                    redis.call('expire', KEYS[1], ARGV[4])
                    return 1
                else
                    return 0
                end
            LUA;

            $result = Redis::eval(
                $luaScript,
                1,
                $lockKey,
                $userId,
                $bookingCode,
                now()->timestamp,
                self::LOCK_TTL
            );

            if ($result === 0) {
                // Rollback - আগের লকগুলো রিলিজ করি
                $this->releaseSeats($showId, $seatIds, $userId);
                throw new SeatAlreadyLockedException(
                    "দুঃখিত! কেউ আপনার আগেই সিট নিয়ে নিয়েছে।"
                );
            }
        }

        // Database এও pending booking create করি
        $booking = Booking::create([
            'booking_code' => $bookingCode,
            'user_id' => $userId,
            'show_id' => $showId,
            'total_amount' => $this->calculateTotal($showId, $seatIds),
            'status' => 'pending',
            'expires_at' => now()->addSeconds(self::LOCK_TTL),
        ]);

        // Booking seats সেভ করি
        foreach ($seatIds as $seatId) {
            $booking->seats()->attach($seatId, [
                'show_id' => $showId,
                'price' => $this->getSeatPrice($showId, $seatId),
            ]);
        }

        return $bookingCode;
    }

    /**
     * সিট রিলিজ করা (timeout বা ক্যান্সেল হলে)
     */
    public function releaseSeats(int $showId, array $seatIds, int $userId): void
    {
        foreach ($seatIds as $seatId) {
            $lockKey = "seat_lock:{$showId}:{$seatId}";

            // শুধু নিজের লক রিলিজ করতে পারবে
            $luaScript = <<<'LUA'
                if redis.call('hget', KEYS[1], 'user_id') == ARGV[1] then
                    redis.call('del', KEYS[1])
                    return 1
                end
                return 0
            LUA;

            Redis::eval($luaScript, 1, $lockKey, $userId);
        }
    }

    /**
     * Booking Code জেনারেট (BMS-2024-A1B2C3)
     */
    private function generateBookingCode(): string
    {
        return 'BMS-' . date('Y') . '-' . strtoupper(substr(md5(uniqid()), 0, 6));
    }

    private function calculateTotal(int $showId, array $seatIds): float
    {
        // সিটের টাইপ অনুযায়ী প্রাইস ক্যালকুলেট
        return Seat::whereIn('id', $seatIds)
            ->get()
            ->sum(fn($seat) => $this->getSeatPrice($showId, $seat->id));
    }

    private function getSeatPrice(int $showId, int $seatId): float
    {
        $show = \App\Models\Show::find($showId);
        $seat = Seat::find($seatId);
        return $show->base_price * $seat->price_multiplier;
    }
}
```

### ৩. Booking Controller - PHP (Laravel)

```php
<?php
// app/Http/Controllers/BookingController.php

namespace App\Http\Controllers;

use App\Services\SeatLockService;
use App\Services\PaymentService;
use App\Models\Booking;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class BookingController extends Controller
{
    public function __construct(
        private SeatLockService $seatLockService,
        private PaymentService $paymentService,
    ) {}

    /**
     * Step 1: সিট সিলেক্ট ও লক করা
     */
    public function selectSeats(Request $request)
    {
        $validated = $request->validate([
            'show_id' => 'required|exists:shows,id',
            'seat_ids' => 'required|array|min:1|max:10',
            'seat_ids.*' => 'exists:seats,id',
        ]);

        try {
            $bookingCode = $this->seatLockService->lockSeats(
                $validated['show_id'],
                $validated['seat_ids'],
                auth()->id()
            );

            return response()->json([
                'success' => true,
                'booking_code' => $bookingCode,
                'message' => 'সিট সিলেক্ট হয়েছে! ৫ মিনিটের মধ্যে পেমেন্ট করুন।',
                'expires_in' => 300,
                'total_amount' => Booking::where('booking_code', $bookingCode)->value('total_amount'),
            ]);
        } catch (\App\Exceptions\SeatAlreadyLockedException $e) {
            return response()->json([
                'success' => false,
                'message' => $e->getMessage(),
            ], 409); // Conflict
        }
    }

    /**
     * Step 2: পেমেন্ট প্রসেস (bKash/Nagad/Card)
     */
    public function processPayment(Request $request)
    {
        $validated = $request->validate([
            'booking_code' => 'required|exists:bookings,booking_code',
            'payment_method' => 'required|in:bkash,nagad,visa,mastercard,cash',
        ]);

        $booking = Booking::where('booking_code', $validated['booking_code'])
            ->where('user_id', auth()->id())
            ->where('status', 'pending')
            ->firstOrFail();

        // Expired কিনা চেক
        if ($booking->expires_at < now()) {
            $booking->update(['status' => 'expired']);
            return response()->json([
                'success' => false,
                'message' => 'সময় শেষ! আবার সিট সিলেক্ট করুন।',
            ], 410); // Gone
        }

        // Database Transaction দিয়ে payment + booking confirm
        return DB::transaction(function () use ($booking, $validated) {
            // bKash/Nagad Payment Gateway কল
            $paymentResult = $this->paymentService->charge(
                amount: $booking->total_amount,
                method: $validated['payment_method'],
                reference: $booking->booking_code
            );

            if ($paymentResult->success) {
                $booking->update([
                    'status' => 'confirmed',
                    'payment_method' => $validated['payment_method'],
                    'confirmed_at' => now(),
                ]);

                // Notification পাঠাও (async via queue)
                dispatch(new \App\Jobs\SendBookingConfirmation($booking));

                return response()->json([
                    'success' => true,
                    'message' => '🎉 বুকিং কনফার্ম! আপনার টিকেট SMS এ পাঠানো হয়েছে।',
                    'booking' => $booking->fresh()->load('seats'),
                    'transaction_id' => $paymentResult->transactionId,
                ]);
            }

            return response()->json([
                'success' => false,
                'message' => 'পেমেন্ট ব্যর্থ হয়েছে। আবার চেষ্টা করুন।',
            ], 402);
        });
    }
}
```

### ৪. Real-time Seat Availability - Node.js (Express + WebSocket)

```javascript
// server/services/seatAvailability.js

const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const Redis = require('ioredis');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
    cors: { origin: '*' }
});

// Redis subscriber - সিট স্ট্যাটাস আপডেট শুনবে
const redisSub = new Redis(process.env.REDIS_URL);
const redisClient = new Redis(process.env.REDIS_URL);

/**
 * Real-time সিট ম্যাপ আপডেট
 * 
 * যখন কেউ Star Cineplex Hall-1 এ সিট লক করে,
 * সব connected users তাৎক্ষণিক দেখবে সিট "taken" হয়ে গেছে
 */
io.on('connection', (socket) => {
    console.log(`User connected: ${socket.id}`);

    // ইউজার যখন একটি শো-এর seat map দেখতে চায়
    socket.on('join_show', async (showId) => {
        // Room join করাও (show-specific)
        socket.join(`show:${showId}`);

        // বর্তমান সিট অবস্থা পাঠাও
        const seatStatus = await getSeatStatusForShow(showId);
        socket.emit('seat_map_update', {
            showId,
            seats: seatStatus,
            timestamp: Date.now()
        });
    });

    // ইউজার যখন seat map থেকে বের হয়
    socket.on('leave_show', (showId) => {
        socket.leave(`show:${showId}`);
    });

    socket.on('disconnect', () => {
        console.log(`User disconnected: ${socket.id}`);
    });
});

/**
 * Redis Keyspace Notifications শুনি
 * যখনই কোনো seat lock/unlock হয়, সবাইকে জানাই
 */
redisSub.psubscribe('__keyevent@0__:*');
redisSub.on('pmessage', async (pattern, channel, key) => {
    // seat_lock:{showId}:{seatId} pattern match
    const match = key.match(/^seat_lock:(\d+):(\d+)$/);
    if (!match) return;

    const [, showId, seatId] = match;
    const event = channel.includes('set') ? 'locked' : 
                  channel.includes('expired') ? 'available' :
                  channel.includes('del') ? 'available' : null;

    if (event) {
        // ওই শো দেখছে এমন সবাইকে আপডেট পাঠাও
        io.to(`show:${showId}`).emit('seat_status_change', {
            seatId: parseInt(seatId),
            status: event,
            timestamp: Date.now()
        });
    }
});

/**
 * একটি শো-এর সব সিটের বর্তমান অবস্থা
 */
async function getSeatStatusForShow(showId) {
    // সব সিট লক keys খুঁজি
    const lockKeys = await redisClient.keys(`seat_lock:${showId}:*`);
    const lockedSeatIds = lockKeys.map(key => {
        const parts = key.split(':');
        return parseInt(parts[2]);
    });

    // DB থেকে confirmed bookings এর সিটও আনি
    const confirmedSeats = await getConfirmedSeatsFromDB(showId);

    return {
        locked: lockedSeatIds,       // হলুদ দেখাবে (temporarily held)
        booked: confirmedSeats,      // লাল দেখাবে (confirmed)
        // বাকি সব available - সবুজ দেখাবে
    };
}

/**
 * REST API - সিট availability চেক
 * (WebSocket না থাকলে fallback)
 */
app.get('/api/shows/:showId/seats', async (req, res) => {
    try {
        const { showId } = req.params;
        const seatStatus = await getSeatStatusForShow(showId);

        res.json({
            success: true,
            show_id: showId,
            seats: seatStatus,
            cache_ttl: 5 // ৫ সেকেন্ড ক্যাশ
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            message: 'সিট তথ্য লোড করতে সমস্যা হচ্ছে'
        });
    }
});

/**
 * Waiting Room / Virtual Queue
 * ঈদের দিন যখন সবাই একসাথে ঢুকতে চায়
 */
const waitingRoom = new Map(); // showId -> Queue

app.post('/api/shows/:showId/join-queue', async (req, res) => {
    const { showId } = req.params;
    const userId = req.user.id;

    // Queue position দাও
    const position = await redisClient.rpush(`waiting_room:${showId}`, userId);
    const estimatedWait = position * 3; // প্রতি ৩ সেকেন্ডে একজন

    res.json({
        success: true,
        position,
        estimated_wait_seconds: estimatedWait,
        message: `আপনার অবস্থান: #${position}। আনুমানিক অপেক্ষা: ${estimatedWait} সেকেন্ড`
    });
});

// DB helper function
async function getConfirmedSeatsFromDB(showId) {
    // PostgreSQL query via connection pool
    const { Pool } = require('pg');
    const pool = new Pool({ connectionString: process.env.DATABASE_URL });

    const result = await pool.query(`
        SELECT bs.seat_id 
        FROM booking_seats bs 
        JOIN bookings b ON bs.booking_id = b.id 
        WHERE bs.show_id = $1 AND b.status = 'confirmed'
    `, [showId]);

    return result.rows.map(r => r.seat_id);
}

// সার্ভার স্টার্ট
const PORT = process.env.PORT || 3001;
httpServer.listen(PORT, () => {
    console.log(`🎬 Seat Availability Service running on port ${PORT}`);
});
```

### ৫. Payment Integration (bKash) - PHP (Laravel)

```php
<?php
// app/Services/PaymentService.php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use App\Models\Payment;

class PaymentService
{
    private string $bkashBaseUrl;
    private string $bkashAppKey;
    private string $bkashAppSecret;

    public function __construct()
    {
        $this->bkashBaseUrl = config('services.bkash.base_url');
        $this->bkashAppKey = config('services.bkash.app_key');
        $this->bkashAppSecret = config('services.bkash.app_secret');
    }

    /**
     * bKash Payment Create
     * ইউজার bKash দিয়ে টিকেটের পেমেন্ট করবে
     */
    public function charge(float $amount, string $method, string $reference): object
    {
        return match ($method) {
            'bkash' => $this->chargeBkash($amount, $reference),
            'nagad' => $this->chargeNagad($amount, $reference),
            default => $this->chargeCard($amount, $method, $reference),
        };
    }

    private function chargeBkash(float $amount, string $reference): object
    {
        // Step 1: bKash Token নাও
        $token = $this->getBkashToken();

        // Step 2: Payment Create
        $response = Http::withHeaders([
            'Authorization' => $token,
            'X-APP-Key' => $this->bkashAppKey,
        ])->post("{$this->bkashBaseUrl}/checkout/payment/create", [
            'mode' => '0011',
            'payerReference' => $reference,
            'callbackURL' => route('payment.bkash.callback'),
            'amount' => $amount,
            'currency' => 'BDT',
            'intent' => 'sale',
            'merchantInvoiceNumber' => $reference,
        ]);

        if ($response->successful() && $response['statusCode'] === '0000') {
            // Payment record সেভ
            Payment::create([
                'booking_id' => $this->getBookingByCode($reference)->id,
                'amount' => $amount,
                'method' => 'bkash',
                'transaction_id' => $response['paymentID'],
                'status' => 'pending',
                'gateway_response' => $response->json(),
            ]);

            return (object) [
                'success' => true,
                'paymentId' => $response['paymentID'],
                'redirectUrl' => $response['bkashURL'],
                'transactionId' => $response['paymentID'],
            ];
        }

        return (object) [
            'success' => false,
            'message' => 'bKash payment তৈরি করতে সমস্যা হয়েছে',
        ];
    }

    private function getBkashToken(): string
    {
        $response = Http::post("{$this->bkashBaseUrl}/checkout/token/grant", [
            'app_key' => $this->bkashAppKey,
            'app_secret' => $this->bkashAppSecret,
        ]);

        return $response['id_token'];
    }
}
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ (Tradeoffs)

### ১. Optimistic vs Pessimistic Locking

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOCKING STRATEGY COMPARISON                            │
├─────────────────────┬───────────────────────┬───────────────────────────┤
│                     │  Optimistic Locking   │  Pessimistic Locking      │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ কীভাবে কাজ করে     │ Version check at      │ Lock at selection time    │
│                     │ commit time           │ (Redis TTL)               │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ Conflict handling   │ Retry on version      │ Reject immediately        │
│                     │ mismatch              │                           │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ UX                  │ "দুঃখিত, সিট নেই"    │ "সিট ৫ মিনিট আপনার"      │
│                     │ (payment এর সময়!)     │ (select করার সাথে সাথে)   │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ Throughput          │ বেশি (no lock hold)   │ কম (lock hold করে রাখে)   │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ Best for            │ Low contention        │ High contention           │
│                     │ (সাধারণ দিন)           │ (ঈদের দিন, BPL Final!)   │
├─────────────────────┼───────────────────────┼───────────────────────────┤
│ আমাদের চয়েস       │         ❌             │         ✅                │
│                     │ (UX খারাপ হয়)         │ (bKash pay করার সময়      │
│                     │                       │  সিট চলে যাবে না!)        │
└─────────────────────┴───────────────────────┴───────────────────────────┘

আমাদের সিদ্ধান্ত: Pessimistic Locking (Redis TTL)
কারণ: ইউজার সিট সিলেক্ট করে bKash app এ গেলো, ফিরে এসে দেখলো সিট
      অন্য কেউ নিয়ে গেছে - এটা ভয়ংকর UX! তাই আমরা ৫ মিনিট লক করি।
```

### ২. SQL vs NoSQL for Bookings

```
┌──────────────────────────────────────────────────────────────────┐
│                     SQL vs NoSQL DECISION                          │
├──────────────────┬─────────────────────┬─────────────────────────┤
│                  │ SQL (PostgreSQL)    │ NoSQL (MongoDB/DynamoDB) │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ ACID Compliance  │ ✅ Full ACID        │ ❌ Eventually consistent │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ Double Booking   │ ✅ UNIQUE constraint│ ⚠️ Need app-level check  │
│ Prevention       │ guarantees it       │                         │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ JOIN Performance │ ✅ Excellent        │ ❌ No native JOINs       │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ Horizontal Scale │ ⚠️ Harder (Citus)   │ ✅ Easy sharding         │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ Schema Flex      │ ❌ Rigid schema     │ ✅ Flexible              │
├──────────────────┼─────────────────────┼─────────────────────────┤
│ আমাদের চয়েস    │ ✅ BOOKINGS এর জন্য │ ✅ SEARCH/CATALOG এর জন্য│
└──────────────────┴─────────────────────┴─────────────────────────┘

সিদ্ধান্ত: Polyglot Persistence
├── PostgreSQL: Bookings, Payments (ACID দরকার!)
├── Redis: Seat locks, Session, Cache
├── Elasticsearch: Event search, Full-text
└── Kafka: Event streaming, Async processing
```

### ৩. Queue-based vs Direct Booking

```
DIRECT BOOKING:                    QUEUE-BASED BOOKING:
                                   
User ──▶ Book ──▶ DB              User ──▶ Queue ──▶ Worker ──▶ DB
         │                                   │
         ▼                                   ▼
      Response                          "আপনার অবস্থান #45"
      (slow if DB busy)                 (instant response!)

┌─────────────────┬──────────────────────┬─────────────────────────┐
│                 │   Direct Booking     │   Queue-based Booking    │
├─────────────────┼──────────────────────┼─────────────────────────┤
│ Response Time   │ Depends on DB load   │ Instant (queued)         │
├─────────────────┼──────────────────────┼─────────────────────────┤
│ Peak Handling   │ ❌ DB overwhelmed     │ ✅ Smooth, controlled    │
├─────────────────┼──────────────────────┼─────────────────────────┤
│ Complexity      │ ✅ Simple             │ ⚠️ More moving parts     │
├─────────────────┼──────────────────────┼─────────────────────────┤
│ User Experience │ Timeout errors       │ "আপনি queue তে আছেন"   │
├─────────────────┼──────────────────────┼─────────────────────────┤
│ Use Case        │ Normal days          │ Eid release, BPL final   │
└─────────────────┴──────────────────────┴─────────────────────────┘

আমাদের সিদ্ধান্ত: Hybrid
├── Normal load: Direct booking (simple, fast)
└── Spike detected: Virtual queue activate (threshold: 1000 concurrent/show)
```

### ৪. Consistency Model Comparison

```
Strong Consistency:
├── সিট বুক হলে সাথে সাথে সবাই দেখবে ✅
├── Latency বেশি (every read goes to primary) ⚠️
└── Use: Seat booking, Payment

Eventual Consistency:
├── কিছু সময় পুরনো ডাটা দেখতে পারে ⚠️
├── Latency কম (read from replica) ✅
└── Use: Show listing, Reviews, Recommendations

আমাদের Decision:
┌────────────────────┬──────────────────────┐
│ Feature            │ Consistency Model     │
├────────────────────┼──────────────────────┤
│ Seat Availability  │ Strong (Real-time)   │
│ Booking Confirm    │ Strong (ACID TX)     │
│ Payment            │ Strong (No loss!)    │
│ Show Listing       │ Eventual (OK if stale)│
│ Reviews/Ratings    │ Eventual             │
│ Notifications      │ Eventual (async)     │
└────────────────────┴──────────────────────┘
```

---

## 📈 কেস স্টাডি

### কেস ১: ঈদের দিন Blockbuster Release (১ লক্ষ Concurrent Users)

```
সিনারিও: "শকুন ২" ঈদের দিন Star Cineplex এ রিলিজ
─────────────────────────────────────────────────────

সমস্যা:
├── ঈদের ১ ঘণ্টা আগে থেকে ১,০০,০০০ ইউজার সাইটে
├── Star Cineplex Bashundhara তে ৫টি হলে মোট ১,২০০ সিট
├── সবাই একই শো (Evening 7:00 PM) চায়
└── bKash সার্ভারেও load বেশি (ঈদের দিন!)

Timeline:
─────────
17:00 - সাইট ওপেন, ট্রাফিক বাড়তে থাকে
17:30 - 50K concurrent, system normal ✅
17:45 - 80K concurrent, auto-scaling triggers ⚠️
18:00 - 100K concurrent, virtual queue activated! 🔴
18:01 - "আপনার অবস্থান #45,623..." 
18:05 - প্রথম batch (500 জন) seat map দেখতে পাচ্ছে
18:10 - Evening 7PM শো SOLD OUT! 🎉
18:11 - সবাইকে alternative shows suggest করা হচ্ছে

Solution Architecture:
─────────────────────
┌─────────────────────────────────────────────────────────────────┐
│                    SPIKE HANDLING SYSTEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────────┐       │
│  │ CDN/Edge │───▶│  Virtual    │───▶│  Rate-Limited    │       │
│  │  Cache   │    │   Queue     │    │  Booking Service │       │
│  └──────────┘    └──────┬──────┘    └────────┬─────────┘       │
│                         │                     │                  │
│                         ▼                     ▼                  │
│                  ┌─────────────┐      ┌──────────────┐          │
│                  │  WebSocket  │      │  DB with     │          │
│                  │ "Position   │      │  Connection  │          │
│                  │  Updates"   │      │  Pooling     │          │
│                  └─────────────┘      └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Key Decisions:
├── ১. Static content (poster, description) → CDN থেকে serve
├── ২. Seat map → Redis cache (1 sec TTL)
├── ৩. Virtual queue → Redis Sorted Set (score = timestamp)
├── ৪. DB connection pool → PgBouncer (max 200 connections)
├── ৫. Horizontal scaling → 10 instances of booking service
└── ৬. bKash timeout → Retry with exponential backoff
```

### কেস ২: BPL Cricket Final - Mirpur Stadium (৩০,০০০ সিট)

```
সিনারিও: Comilla Victorians vs Dhaka Dominators - BPL Final
──────────────────────────────────────────────────────────────

Challenge:
├── ৩০,০০০ সিট - একসাথে বিক্রি শুরু (10:00 AM)
├── ৫,০০,০০০+ মানুষ চেষ্টা করছে
├── বিভিন্ন ক্যাটেগরি: VIP (৳5000), Premium (৳2000), General (৳500)
├── এক ইউজার max ৪টি টিকেট কিনতে পারবে
└── Scalpers/Bots ঠেকাতে হবে

Anti-Bot Strategy:
─────────────────
┌─────────────────────────────────────────────┐
│           FAIR QUEUE SYSTEM                  │
├─────────────────────────────────────────────┤
│                                             │
│  ① CAPTCHA verification before queue entry  │
│  ② Random queue position (not FIFO!)        │
│  ③ Phone number OTP verification            │
│  ④ Max 4 tickets per verified account       │
│  ⑤ Purchase rate limiting per IP            │
│  ⑥ Browser fingerprinting                   │
│                                             │
└─────────────────────────────────────────────┘

Queue Implementation:
```

```javascript
// BPL Final - Virtual Queue with Random Position
const crypto = require('crypto');

class FairQueue {
    constructor(redisClient, eventId) {
        this.redis = redisClient;
        this.eventId = eventId;
        this.queueKey = `fair_queue:${eventId}`;
    }

    /**
     * Random position queue - bot দের advantage দেবে না
     * FIFO হলে bot আগে ঢুকে সব টিকেট নিয়ে নেবে!
     */
    async joinQueue(userId) {
        // Random score assign (timestamp নয়!)
        const randomScore = crypto.randomInt(1, 1000000);

        await this.redis.zadd(this.queueKey, randomScore, userId);
        const position = await this.redis.zrank(this.queueKey, userId);

        return {
            position: position + 1,
            total_in_queue: await this.redis.zcard(this.queueKey),
            estimated_wait: Math.ceil((position + 1) * 2), // ~2sec per person
        };
    }

    /**
     * Batch release - ৫০ জন করে ছাড়ি
     * (DB overwhelm হবে না)
     */
    async releaseBatch(batchSize = 50) {
        const batch = await this.redis.zpopmin(this.queueKey, batchSize);
        const userIds = [];

        for (let i = 0; i < batch.length; i += 2) {
            userIds.push(batch[i]); // member (userId)
        }

        // এদের token দাও - ১০ মিনিট validity
        for (const userId of userIds) {
            const token = crypto.randomUUID();
            await this.redis.setex(
                `booking_token:${userId}`,
                600, // 10 min
                token
            );
        }

        return userIds;
    }
}
```

### কেস ৩: Payment Timeout Handling

```
সমস্যা: ইউজার bKash এ গেলো, কিন্তু bKash server slow (ঈদের দিন!)
──────────────────────────────────────────────────────────────────────

Timeline:
  00:00 - User সিট সিলেক্ট করলো (Lock acquired, TTL: 5 min)
  00:30 - bKash payment initiate করলো
  02:00 - bKash OTP pending... (bKash server slow)
  04:00 - Still waiting... ⚠️ Lock almost expired!
  05:00 - Lock EXPIRED! সিট অন্য কারো জন্য available! 🔴
  05:30 - bKash payment SUCCESS! কিন্তু সিট তো গেছে... 😱

Solution: Payment State Machine
────────────────────────────────

┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌───────────┐
│ PENDING  │───▶│ PAYMENT  │───▶│  CONFIRMING  │───▶│ CONFIRMED │
│          │    │INITIATED │    │              │    │           │
└──────────┘    └────┬─────┘    └──────┬───────┘    └───────────┘
                     │                 │
                     ▼                 ▼
              ┌──────────┐     ┌──────────────┐
              │  EXPIRED  │     │AUTO-REFUND   │
              │ (TTL out) │     │(seat gone,   │
              └──────────┘     │money back)   │
                               └──────────────┘
```

```php
<?php
// app/Jobs/HandlePaymentTimeout.php

namespace App\Jobs;

use App\Models\Booking;
use App\Models\Payment;
use App\Services\PaymentService;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Contracts\Queue\ShouldQueue;

class HandlePaymentTimeout implements ShouldQueue
{
    use Queueable, SerializesModels;

    public function __construct(private Booking $booking) {}

    public function handle(PaymentService $paymentService): void
    {
        $this->booking->refresh();

        // Case 1: Lock expired কিন্তু payment চলছে
        if ($this->booking->status === 'pending' && $this->booking->expires_at < now()) {

            // Payment gateway চেক করি - হয়তো মাঝখানে success হয়ে গেছে
            $payment = Payment::where('booking_id', $this->booking->id)
                ->latest()
                ->first();

            if ($payment && $payment->status === 'completed') {
                // Payment হয়ে গেছে! সিট still available কিনা চেক
                if ($this->seatsStillAvailable()) {
                    // Lock re-acquire করি এবং confirm করি
                    $this->reacquireAndConfirm();
                } else {
                    // সিট গেছে 😢 - Auto refund!
                    $paymentService->refund($payment);
                    $this->booking->update(['status' => 'refunded']);

                    // User কে notify করি
                    dispatch(new SendRefundNotification(
                        $this->booking,
                        'দুঃখিত! আপনার সিট lock expire হয়ে গিয়েছিল। ৳' .
                        $payment->amount . ' refund করা হয়েছে।'
                    ));
                }
            } else {
                // Payment ও হয়নি - simply expire করি
                $this->booking->update(['status' => 'expired']);
            }
        }
    }

    private function seatsStillAvailable(): bool
    {
        // অন্য কেউ ইতিমধ্যে এই সিটগুলো বুক করেছে কিনা চেক
        $bookedSeats = $this->booking->seats()
            ->whereHas('bookings', fn($q) => $q
                ->where('id', '!=', $this->booking->id)
                ->where('status', 'confirmed')
            )
            ->count();

        return $bookedSeats === 0;
    }

    private function reacquireAndConfirm(): void
    {
        \DB::transaction(function () {
            $this->booking->update([
                'status' => 'confirmed',
                'confirmed_at' => now(),
            ]);
        });
    }
}
```

---

## 🔧 Advanced Topics

### ১. Seat Map Real-time Sync Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                  REAL-TIME SEAT MAP SYNC                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Browser A           Server              Browser B                   │
│  (ইউজার ক)          (Node.js)           (ইউজার খ)                  │
│     │                   │                    │                       │
│     │──── WS Connect ──▶│◀── WS Connect ────│                       │
│     │                   │                    │                       │
│     │── Select Seat ───▶│                    │                       │
│     │   A-5             │                    │                       │
│     │                   │── Lock A-5 ──▶Redis│                       │
│     │                   │                    │                       │
│     │◀─ "A-5 locked" ──│── "A-5 taken" ───▶│                       │
│     │   (তোমার!)        │   (অন্যে নিয়েছে!) │                       │
│     │                   │                    │                       │
│     │                   │    ৫ min later     │                       │
│     │                   │◀── TTL Expired ────│(Redis)                │
│     │                   │                    │                       │
│     │◀─ "A-5 available"│── "A-5 free!" ───▶│                       │
│     │                   │                    │                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ২. Dynamic Pricing Algorithm

```php
<?php
// app/Services/DynamicPricingService.php

namespace App\Services;

class DynamicPricingService
{
    /**
     * ডায়নামিক প্রাইসিং - ডিমান্ড অনুযায়ী দাম
     * 
     * উদাহরণ: Star Cineplex - ঈদের দিন Premium হলের দাম বেশি
     * সাধারণ দিন ৳350 → ঈদের দিন ৳500 → Last few seats ৳700
     */
    public function calculatePrice(int $showId, int $seatId): float
    {
        $show = \App\Models\Show::with('screen.seats')->find($showId);
        $seat = \App\Models\Seat::find($seatId);
        $basePrice = $show->base_price;

        // Factor 1: সিটের টাইপ
        $seatMultiplier = $seat->price_multiplier; // regular:1.0, premium:1.5, couple:2.0

        // Factor 2: কত % সিট বিক্রি হয়েছে (Demand)
        $totalSeats = $show->screen->total_seats;
        $bookedSeats = $show->bookings()->where('status', 'confirmed')->count();
        $occupancyRate = $bookedSeats / $totalSeats;

        $demandMultiplier = match (true) {
            $occupancyRate > 0.9 => 1.5,  // 90%+ filled: 50% বেশি!
            $occupancyRate > 0.7 => 1.3,  // 70%+ filled: 30% বেশি
            $occupancyRate > 0.5 => 1.1,  // 50%+ filled: 10% বেশি
            $occupancyRate < 0.2 => 0.8,  // 20% এর কম: 20% ডিসকাউন্ট!
            default => 1.0,
        };

        // Factor 3: সময় ভিত্তিক (শো-এর কত আগে কিনছে)
        $hoursUntilShow = now()->diffInHours($show->show_time);
        $timeMultiplier = match (true) {
            $hoursUntilShow < 2 => 1.2,    // Last minute: বেশি দাম
            $hoursUntilShow < 24 => 1.0,   // Same day: normal
            $hoursUntilShow > 168 => 0.85, // ১ সপ্তাহ+ আগে: ডিসকাউন্ট
            default => 1.0,
        };

        // Factor 4: বিশেষ দিন (ঈদ, পহেলা বৈশাখ, ভ্যালেন্টাইন)
        $specialDayMultiplier = $this->getSpecialDayMultiplier($show->show_time);

        $finalPrice = $basePrice
            * $seatMultiplier
            * $demandMultiplier
            * $timeMultiplier
            * $specialDayMultiplier;

        // সর্বোচ্চ ৩x base price (ইউজার যেন ঠকে না মনে করে)
        return min($finalPrice, $basePrice * 3);
    }

    private function getSpecialDayMultiplier(\Carbon\Carbon $showTime): float
    {
        // বাংলাদেশের বিশেষ দিনগুলো
        $specialDays = [
            // ঈদুল ফিতর, ঈদুল আযহা (approximate, varies yearly)
            'eid' => 1.4,
            // পহেলা বৈশাখ (April 14)
            '04-14' => 1.3,
            // ভ্যালেন্টাইনস ডে
            '02-14' => 1.2,
            // বিজয় দিবস
            '12-16' => 1.1,
        ];

        $monthDay = $showTime->format('m-d');
        return $specialDays[$monthDay] ?? 1.0;
    }
}
```

### ৩. Fraud Detection System

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FRAUD DETECTION RULES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Rule 1: একই IP থেকে ১ ঘণ্টায় ১০+ বুকিং attempt → BLOCK 🚫       │
│  Rule 2: একই ফোন নম্বরে ১ দিনে ২০+ টিকেট → FLAG 🚩                │
│  Rule 3: Booking করে ক্যান্সেল → ৩ বারের বেশি → SUSPICIOUS 🔍      │
│  Rule 4: Payment fail ratio > 50% → RATE LIMIT ⚠️                    │
│  Rule 5: Headless browser detected → CAPTCHA enforce 🤖              │
│  Rule 6: ১ একাউন্টে multiple payment methods → VERIFY 📞            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```javascript
// server/middleware/fraudDetection.js

const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class FraudDetector {
    /**
     * প্রতিটি booking attempt এ চেক করি
     */
    async checkForFraud(userId, ip, userAgent) {
        const checks = await Promise.all([
            this.checkIPRate(ip),
            this.checkUserRate(userId),
            this.checkCancelPattern(userId),
            this.checkBotSignals(userAgent),
        ]);

        const riskScore = checks.reduce((sum, check) => sum + check.score, 0);

        return {
            allowed: riskScore < 70,      // 70+ = blocked
            riskScore,
            action: riskScore >= 70 ? 'block' :
                    riskScore >= 40 ? 'captcha' :
                    riskScore >= 20 ? 'monitor' : 'allow',
            reasons: checks.filter(c => c.score > 0).map(c => c.reason),
        };
    }

    async checkIPRate(ip) {
        const key = `fraud:ip:${ip}`;
        const count = await redis.incr(key);
        await redis.expire(key, 3600); // 1 hour window

        if (count > 10) {
            return { score: 40, reason: `IP ${ip} থেকে ${count} attempts/hour` };
        }
        return { score: 0, reason: null };
    }

    async checkUserRate(userId) {
        const key = `fraud:user:${userId}:daily`;
        const count = await redis.incr(key);
        await redis.expire(key, 86400); // 24 hour window

        if (count > 20) {
            return { score: 50, reason: `User ${userId} - ${count} tickets/day` };
        }
        return { score: 0, reason: null };
    }

    async checkCancelPattern(userId) {
        const key = `fraud:cancel:${userId}`;
        const cancelCount = parseInt(await redis.get(key) || '0');

        if (cancelCount > 3) {
            return { score: 30, reason: `${cancelCount} cancellations recently` };
        }
        return { score: 0, reason: null };
    }

    async checkBotSignals(userAgent) {
        const botPatterns = [
            /headless/i, /phantom/i, /selenium/i,
            /puppeteer/i, /playwright/i
        ];

        const isBot = botPatterns.some(p => p.test(userAgent));
        if (isBot) {
            return { score: 80, reason: 'Bot detected via User-Agent' };
        }
        return { score: 0, reason: null };
    }
}

module.exports = new FraudDetector();
```

### ৪. Waiting Room / Virtual Queue (Node.js)

```javascript
// server/services/waitingRoom.js

const EventEmitter = require('events');

class WaitingRoom extends EventEmitter {
    constructor(redis, io) {
        super();
        this.redis = redis;
        this.io = io;
        this.batchSize = 50;        // একবারে ৫০ জন ছাড়ি
        this.batchInterval = 3000;   // প্রতি ৩ সেকেন্ডে
        this.activeTimers = new Map();
    }

    /**
     * Waiting room activate করো (threshold cross হলে)
     * 
     * উদাহরণ: ঈদের দিন "শকুন ২" এর Evening show তে
     * ১০০০+ concurrent users → waiting room ON!
     */
    async activate(showId) {
        const queueKey = `waiting_room:${showId}`;
        console.log(`🚦 Waiting room ACTIVATED for show ${showId}`);

        // প্রতি ৩ সেকেন্ডে ৫০ জন করে ছাড়ি
        const timer = setInterval(async () => {
            const batch = await this.redis.zpopmin(queueKey, this.batchSize);

            if (batch.length === 0) {
                clearInterval(timer);
                this.activeTimers.delete(showId);
                console.log(`✅ Waiting room cleared for show ${showId}`);
                return;
            }

            // প্রতি ২ জন করে userId বের করি (ZPOPMIN returns [member, score, ...])
            for (let i = 0; i < batch.length; i += 2) {
                const userId = batch[i];

                // Access token দাও - ১০ মিনিট validity
                const accessToken = require('crypto').randomUUID();
                await this.redis.setex(
                    `access_token:${showId}:${userId}`,
                    600,
                    accessToken
                );

                // WebSocket দিয়ে জানাও "আপনার পালা!"
                this.io.to(`user:${userId}`).emit('queue_your_turn', {
                    showId,
                    accessToken,
                    message: '🎉 আপনার পালা এসেছে! এখন সিট সিলেক্ট করুন।',
                    expires_in: 600,
                });
            }

            // Remaining queue size broadcast
            const remaining = await this.redis.zcard(queueKey);
            this.io.to(`show:${showId}:queue`).emit('queue_update', {
                remaining,
                estimated_wait: Math.ceil(remaining / this.batchSize) * 3,
            });

        }, this.batchInterval);

        this.activeTimers.set(showId, timer);
    }

    /**
     * কতজন show দেখছে সেটার উপর ভিত্তি করে auto-activate
     */
    async shouldActivate(showId) {
        const viewerCount = await this.redis.scard(`show_viewers:${showId}`);
        return viewerCount > 1000; // ১০০০+ viewers = activate!
    }
}

module.exports = WaitingRoom;
```

---

## 🎯 সারসংক্ষেপ

### Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────┐
│              টিকেটিং সিস্টেম ডিজাইন - মূল শিক্ষা                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ১. 🔒 Seat Locking: Redis TTL দিয়ে Pessimistic Lock               │
│     → ডাবল বুকিং ঠেকানোর সবচেয়ে reliable উপায়                     │
│                                                                      │
│  ২. 💳 Payment: Idempotent design + State Machine                   │
│     → bKash timeout হলেও ডাটা inconsistent হবে না                   │
│                                                                      │
│  ৩. 📊 CQRS: Read/Write path আলাদা                                 │
│     → Read heavy system (100:1 ratio) এ performance বাড়ে            │
│                                                                      │
│  ৪. 🚦 Virtual Queue: Spike এ graceful degradation                  │
│     → ঈদের দিন সিস্টেম crash না করে queue দিয়ে handle              │
│                                                                      │
│  ⑤. 🔄 Event-Driven: Kafka দিয়ে async processing                   │
│     → Notification, Analytics, Cache invalidation - সব async         │
│                                                                      │
│  ৬. 🛡️ Fraud Detection: Multi-layer bot protection                  │
│     → Scalpers থেকে genuine users কে protect করা                    │
│                                                                      │
│  ৭. 💰 Dynamic Pricing: Demand-based pricing                        │
│     → Revenue optimize করা, কম demand এ discount দেওয়া              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Technology Stack Summary

```
┌────────────────────┬──────────────────────────────────────────┐
│ Layer              │ Technology                                │
├────────────────────┼──────────────────────────────────────────┤
│ Frontend           │ React/Next.js + Socket.io Client         │
│ API Gateway        │ Kong / Laravel (PHP)                     │
│ Booking Service    │ Laravel (PHP) - ACID transactions        │
│ Real-time Service  │ Node.js (Express) + Socket.io            │
│ Payment Service    │ Node.js - bKash/Nagad integration        │
│ Primary DB         │ PostgreSQL (Bookings, Payments)          │
│ Cache/Locks        │ Redis (Seat locks, Session, Queue)       │
│ Search             │ Elasticsearch (Event discovery)          │
│ Message Queue      │ Apache Kafka (Event-driven async)        │
│ CDN                │ CloudFront / Cloudflare                  │
│ Monitoring         │ Prometheus + Grafana                     │
│ Load Balancer      │ Nginx / HAProxy                          │
└────────────────────┴──────────────────────────────────────────┘
```

### Interview Tips 🎤

```
System Design Interview এ এই প্রশ্নগুলো আসতে পারে:

Q1: "কীভাবে double booking prevent করবেন?"
→ Redis distributed lock + DB UNIQUE constraint (defense in depth)

Q2: "১ লক্ষ concurrent user handle করবেন কীভাবে?"
→ Virtual queue + Horizontal scaling + CQRS + CDN

Q3: "Payment timeout হলে কী হবে?"
→ State machine + Idempotent retry + Auto-refund flow

Q4: "Bot/Scalper ঠেকাবেন কীভাবে?"
→ Rate limiting + CAPTCHA + Random queue + Browser fingerprinting

Q5: "Seat map real-time কীভাবে?"
→ WebSocket + Redis Pub/Sub + Keyspace notifications
```

---

> 📝 **নোট**: এই ডিজাইনটি BookMyShow, Shohoz.com, এবং Star Cineplex এর মতো
> real-world সিস্টেমের architectural decisions এর উপর ভিত্তি করে তৈরি। প্রতিটি
> decision এর পেছনে specific tradeoff আছে যা আপনার scale এবং budget অনুযায়ী পরিবর্তন হতে পারে।
