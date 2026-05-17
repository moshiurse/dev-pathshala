# 🚕 ট্যাক্সি হেইলিং অ্যাপ সিস্টেম ডিজাইন (Uber)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Uber, Lyft, Pathao, Grab, Obhai, Shohoz Rides

---

## 📑 সূচিপত্র

1. [Requirements](#-requirements)
2. [Back-of-envelope Estimation](#-back-of-envelope-estimation)
3. [High-Level Design](#️-high-level-design)
4. [Detailed Design](#-detailed-design)
5. [ট্রেড-অফ বিশ্লেষণ](#️-ট্রেড-অফ-বিশ্লেষণ)
6. [কেস স্টাডি](#-কেস-স্টাডি)
7. [Advanced Topics](#-advanced-topics)
8. [সারসংক্ষেপ](#-সারসংক্ষেপ)

---

## 📌 Requirements

### Functional Requirements

| # | Feature | বিবরণ |
|---|---------|--------|
| 1 | Ride Request | রাইডার পিকআপ ও ড্রপ লোকেশন দিয়ে রাইড রিকোয়েস্ট করবে |
| 2 | Driver Matching | নিকটতম available ড্রাইভারকে ম্যাচ করা |
| 3 | Real-time Tracking | রাইডার ও ড্রাইভার উভয়ের লাইভ লোকেশন ট্র্যাকিং |
| 4 | ETA Calculation | আনুমানিক সময় (ঢাকার ট্রাফিক কনডিশন অনুযায়ী) |
| 5 | Fare Calculation | দূরত্ব, সময়, surge multiplier অনুযায়ী ভাড়া |
| 6 | Payment | bKash, Nagad, Card, Cash payment support |
| 7 | Rating & Review | রাইড শেষে উভয়পক্ষ রেটিং দেবে |
| 8 | Ride History | আগের সব রাইডের ইতিহাস |
| 9 | Surge Pricing | চাহিদা বেশি হলে dynamic pricing |

### Non-Functional Requirements

| Requirement | Target | ব্যাখ্যা |
|-------------|--------|----------|
| Matching Latency | < 1 min | রিকোয়েস্ট থেকে ড্রাইভার ম্যাচ ১ মিনিটের মধ্যে |
| Location Update | প্রতি 3-5 সেকেন্ড | রিয়েল-টাইম GPS ডেটা |
| Availability | 99.99% uptime | ঈদ/পূজার সময়ও সার্ভিস চালু থাকবে |
| Scalability | 1M+ concurrent riders | পিক আওয়ারে (অফিস টাইম ঢাকা) handle করতে হবে |
| Consistency | Trip state eventual consistency < 2s | পেমেন্টে strong consistency |

### 🇧🇩 বাংলাদেশ Context

```
ঢাকার বিশেষ চ্যালেঞ্জ:
├── ট্রাফিক জ্যাম: ৩ কিমি যেতে ৪৫ মিনিট লাগতে পারে
├── সরু গলি: GPS accuracy কম (Purana Dhaka)
├── ফ্লাড সিজন: রাস্তা ডুবে গেলে route পরিবর্তন
├── ঈদ রাশ: স্বাভাবিকের ১০ গুণ demand
├── Payment: ৭০%+ cash payment preference
└── Network: 2G/3G এলাকায় কাজ করতে হবে
```

**Pathao-র শিক্ষা**: বাংলাদেশে bike ride সবচেয়ে জনপ্রিয় — গলিতে গাড়ি ঢুকতে পারে না কিন্তু বাইক পারে।

---

## 📊 Back-of-envelope Estimation

### ট্রাফিক ক্যালকুলেশন (Dhaka-centric)

```
Active Riders (Daily):         5,00,000 (Dhaka metro)
Active Drivers:                 1,00,000
Peak Hour Rides/min:            5,000 requests/min
Average Ride Duration:          25 min (Dhaka traffic!)

Location Updates:
├── Drivers online:             50,000 (peak)
├── Update frequency:           every 4 seconds
├── Location updates/sec:       50,000 / 4 = 12,500 updates/sec
└── Daily location points:      12,500 × 3600 × 16hrs = ~720M points/day

Storage:
├── Location point size:        ~50 bytes (lat, lng, timestamp, driver_id)
├── Daily location storage:     720M × 50B = ~36 GB/day
├── Trip record:                ~2 KB (metadata + route)
├── Daily trips:                3,00,000
└── Daily trip storage:         300K × 2KB = ~600 MB/day

Bandwidth:
├── Location update payload:    ~100 bytes
├── Inbound (driver→server):    12,500 × 100B = 1.25 MB/s
└── Outbound (server→rider):    ~5,000 × 200B = 1 MB/s (active tracking)
```

### সার্ভার এস্টিমেশন

```
API Servers:        20-30 instances (auto-scaling)
WebSocket Servers:  15-20 instances (location streaming)
Matching Engine:    5-10 instances (compute intensive)
Database:           
├── PostgreSQL:     3-node cluster (trips, users)
├── Redis Cluster:  6 nodes (location, caching)
└── MongoDB:        3-node replica (ride history, analytics)
```

---

## 🏗️ High-Level Design

### সিস্টেম আর্কিটেকচার

```
┌─────────────┐         ┌─────────────┐
│  Rider App  │         │ Driver App  │
│  (Android/  │         │  (Android/  │
│    iOS)     │         │    iOS)     │
└──────┬──────┘         └──────┬──────┘
       │                       │
       │ HTTPS/WSS             │ HTTPS/WSS
       │                       │
       ▼                       ▼
┌──────────────────────────────────────────┐
│           API Gateway / Load Balancer     │
│        (Kong / Nginx / AWS ALB)          │
└──────┬───────────┬───────────┬───────────┘
       │           │           │
       ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│   Trip   │ │ Location │ │  WebSocket   │
│ Service  │ │ Service  │ │   Gateway    │
│ (Laravel)│ │(Node.js) │ │  (Node.js)   │
└────┬─────┘ └────┬─────┘ └──────┬───────┘
     │             │              │
     ▼             ▼              │
┌──────────┐ ┌──────────────┐    │
│ Matching │ │  Geospatial  │    │
│  Engine  │ │    Index     │    │
│(Node.js) │ │(Redis + H3)  │◄───┘
└────┬─────┘ └──────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│              Data Layer                    │
├──────────┬───────────┬───────────────────┤
│PostgreSQL│   Redis   │     Kafka         │
│ (Trips,  │(Location, │  (Events,         │
│  Users)  │  Cache)   │   Analytics)      │
└──────────┴───────────┴───────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│          Supporting Services              │
├──────────┬──────────┬────────────────────┤
│ Payment  │  Notif.  │   Pricing/Surge    │
│(bKash/   │(FCM/SMS) │    Engine          │
│ Nagad)   │          │                    │
└──────────┴──────────┴────────────────────┘
```

### ডেটা ফ্লো - রাইড রিকোয়েস্ট

```
Rider                    Server                      Driver
  │                        │                           │
  │──── Request Ride ─────▶│                           │
  │                        │── Find Nearby Drivers ──▶ │
  │                        │◀─ Available Drivers ─────│
  │                        │                           │
  │                        │── Select Best Match ──┐   │
  │                        │◀──────────────────────┘   │
  │                        │                           │
  │                        │──── Send Ride Offer ─────▶│
  │                        │◀─── Accept/Reject ────────│
  │                        │                           │
  │◀── Driver Assigned ────│                           │
  │                        │                           │
  │◀── Live Location ──────│◀── GPS Updates ──────────│
  │                        │                           │
  │   (Driver arrives)     │                           │
  │◀── Trip Started ───────│                           │
  │                        │                           │
  │◀── Live Tracking ──────│◀── Route Updates ────────│
  │                        │                           │
  │   (Destination)        │                           │
  │◀── Trip Ended ─────────│──── Fare Calculated ────▶│
  │                        │                           │
  │──── Pay & Rate ───────▶│──── Earnings Update ────▶│
  │                        │                           │
```

---

## 💻 Detailed Design

### 1. Location Tracking & Storage

#### Geospatial Indexing Strategies

```
┌─────────────────────────────────────────────────────────┐
│                    Geohash Grid                          │
│                                                         │
│  ┌─────┬─────┬─────┬─────┐                            │
│  │wh4x │wh4z │wh50 │wh51 │  Precision 5: ~5km × 5km  │
│  ├─────┼─────┼─────┼─────┤                            │
│  │wh4w │wh4y │wh52 │wh53 │  Precision 6: ~1.2km       │
│  ├─────┼─────┼─────┼─────┤                            │
│  │wh4t │wh4v │wh54 │wh55 │  Precision 7: ~150m        │
│  ├─────┼─────┼─────┼─────┤                            │
│  │wh4s │wh4u │wh56 │wh57 │  Precision 8: ~40m         │
│  └─────┴─────┴─────┴─────┘                            │
│                                                         │
│  Dhaka Example:                                         │
│  Gulshan-2: geohash "wh4xq3" (precision 6)            │
│  Dhanmondi: geohash "wh4wm8" (precision 6)            │
└─────────────────────────────────────────────────────────┘
```

#### Redis Geospatial Commands

```
# ড্রাইভার লোকেশন আপডেট
GEOADD drivers:available 90.4125 23.8103 "driver_1234"

# ৩ কিমি রেডিয়াসে ড্রাইভার খোঁজা (Gulshan-2 থেকে)
GEORADIUS drivers:available 90.4125 23.8103 3 km ASC COUNT 10

# দুই পয়েন্টের দূরত্ব
GEODIST drivers:available "driver_1234" "driver_5678" km
```

### 2. Driver Matching Algorithm

```
┌───────────────────────────────────────────────────┐
│           Matching Engine Flow                     │
│                                                   │
│  Ride Request                                     │
│       │                                           │
│       ▼                                           │
│  ┌─────────────────┐                             │
│  │ Query Geospatial│                             │
│  │ Index (3km)     │                             │
│  └────────┬────────┘                             │
│           │                                       │
│           ▼                                       │
│  ┌─────────────────┐                             │
│  │ Filter:         │                             │
│  │ - Vehicle type  │ (Bike/Car/CNG)             │
│  │ - Driver status │ (available/busy)            │
│  │ - Rating > 4.0  │                             │
│  └────────┬────────┘                             │
│           │                                       │
│           ▼                                       │
│  ┌─────────────────┐                             │
│  │ Rank by Score:  │                             │
│  │ - ETA (40%)     │                             │
│  │ - Rating (30%)  │                             │
│  │ - Accept Rate   │                             │
│  │   (20%)         │                             │
│  │ - Earnings      │                             │
│  │   Balance (10%) │                             │
│  └────────┬────────┘                             │
│           │                                       │
│           ▼                                       │
│  ┌─────────────────┐                             │
│  │ Send offer to   │  Timeout: 15 sec            │
│  │ top driver      │  If rejected → next driver  │
│  └─────────────────┘                             │
└───────────────────────────────────────────────────┘
```

### 3. Trip Lifecycle State Machine

```
                    ┌───────────┐
                    │ REQUESTED │
                    └─────┬─────┘
                          │ Driver matched
                          ▼
                    ┌───────────┐
              ┌─────│  MATCHED  │
              │     └─────┬─────┘
   Driver     │           │ Driver accepted
   cancelled  │           ▼
              │     ┌───────────┐
              │     │ ACCEPTED  │
              │     └─────┬─────┘
              │           │ Driver arrived at pickup
              ▼           ▼
        ┌───────────┐ ┌───────────┐
        │ CANCELLED │ │  ARRIVED  │
        └───────────┘ └─────┬─────┘
                            │ Trip started
                            ▼
                      ┌───────────┐
                      │  ONGOING  │
                      └─────┬─────┘
                            │ Reached destination
                            ▼
                      ┌───────────┐
                      │ COMPLETED │
                      └─────┬─────┘
                            │ Payment processed
                            ▼
                      ┌───────────┐
                      │   PAID    │
                      └───────────┘
```

### 4. Surge/Dynamic Pricing

```
┌──────────────────────────────────────────────┐
│         Surge Pricing Formula                 │
│                                              │
│  surge_multiplier = f(demand, supply, area)  │
│                                              │
│  demand_factor = requests_per_min / baseline │
│  supply_factor = available_drivers / normal  │
│                                              │
│  multiplier = demand_factor / supply_factor  │
│                                              │
│  Example (Gulshan, Eid Eve 10pm):            │
│  ├── Requests: 500/min (baseline: 50/min)    │
│  ├── Drivers: 20 (normal: 200)               │
│  ├── demand_factor: 500/50 = 10              │
│  ├── supply_factor: 20/200 = 0.1            │
│  ├── raw_multiplier: 10/0.1 = 100x          │
│  └── capped_multiplier: 5x (max cap)        │
│                                              │
│  Final Fare = Base Fare × 5.0               │
│  ঢাকা: ৫০ টাকা × ৫ = ২৫০ টাকা (minimum)    │
└──────────────────────────────────────────────┘
```

### 5. PHP (Laravel) Code: Trip Service

```php
<?php

namespace App\Services;

use App\Models\Trip;
use App\Models\Driver;
use App\Events\TripRequested;
use App\Events\TripStatusChanged;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\DB;

class TripService
{
    private MatchingService $matchingService;
    private PricingService $pricingService;
    private GeospatialService $geoService;

    public function __construct(
        MatchingService $matchingService,
        PricingService $pricingService,
        GeospatialService $geoService
    ) {
        $this->matchingService = $matchingService;
        $this->pricingService = $pricingService;
        $this->geoService = $geoService;
    }

    /**
     * রাইড রিকোয়েস্ট তৈরি করা
     * Pathao/Uber BD স্টাইলে - vehicle type সহ
     */
    public function requestRide(array $data): Trip
    {
        // ভাড়া estimate করা (surge সহ)
        $fareEstimate = $this->pricingService->calculateEstimate(
            pickup: $data['pickup'],
            dropoff: $data['dropoff'],
            vehicleType: $data['vehicle_type'], // bike, car, cng
            area: $this->geoService->getArea($data['pickup'])
        );

        $trip = DB::transaction(function () use ($data, $fareEstimate) {
            return Trip::create([
                'rider_id' => $data['rider_id'],
                'pickup_lat' => $data['pickup']['lat'],
                'pickup_lng' => $data['pickup']['lng'],
                'dropoff_lat' => $data['dropoff']['lat'],
                'dropoff_lng' => $data['dropoff']['lng'],
                'vehicle_type' => $data['vehicle_type'],
                'status' => Trip::STATUS_REQUESTED,
                'estimated_fare' => $fareEstimate['total'],
                'surge_multiplier' => $fareEstimate['surge'],
                'payment_method' => $data['payment_method'], // bkash, nagad, cash
            ]);
        });

        // Matching Engine-এ পাঠানো
        event(new TripRequested($trip));

        return $trip;
    }

    /**
     * ভাড়া ক্যালকুলেশন - রাইড শেষে
     * ঢাকার ট্রাফিকে সময় ভিত্তিক চার্জ গুরুত্বপূর্ণ
     */
    public function calculateFinalFare(Trip $trip): array
    {
        $distanceKm = $this->geoService->calculateRouteDistance(
            $trip->route_polyline
        );
        $durationMin = $trip->ended_at->diffInMinutes($trip->started_at);

        // বাংলাদেশ প্রাইসিং (Pathao-র মতো structure)
        $rates = config("pricing.{$trip->vehicle_type}");

        $baseFare = $rates['base_fare'];           // বাইক: ২৫৳, কার: ৫০৳
        $perKmRate = $rates['per_km'];             // বাইক: ১২৳/km, কার: ২০৳/km
        $perMinRate = $rates['per_min'];           // বাইক: ২৳/min, কার: ৪৳/min
        $minimumFare = $rates['minimum_fare'];     // বাইক: ৪০৳, কার: ৮০৳

        $distanceCharge = $distanceKm * $perKmRate;
        $timeCharge = $durationMin * $perMinRate;
        $subtotal = $baseFare + $distanceCharge + $timeCharge;

        // Surge apply
        $surgedTotal = $subtotal * $trip->surge_multiplier;

        // Minimum fare check
        $finalFare = max($surgedTotal, $minimumFare);

        // Platform commission (Pathao: ~20%, Uber: ~25%)
        $commission = $finalFare * 0.20;
        $driverEarnings = $finalFare - $commission;

        $trip->update([
            'actual_fare' => $finalFare,
            'distance_km' => $distanceKm,
            'duration_min' => $durationMin,
            'driver_earnings' => $driverEarnings,
            'platform_commission' => $commission,
            'status' => Trip::STATUS_COMPLETED,
        ]);

        return [
            'fare' => $finalFare,
            'breakdown' => [
                'base_fare' => $baseFare,
                'distance_charge' => $distanceCharge,
                'time_charge' => $timeCharge,
                'surge_multiplier' => $trip->surge_multiplier,
                'total' => $finalFare,
            ],
            'driver_earnings' => $driverEarnings,
        ];
    }
}

/**
 * Surge Pricing Service
 */
class PricingService
{
    public function calculateEstimate(array $pickup, array $dropoff, string $vehicleType, string $area): array
    {
        $distance = $this->getEstimatedDistance($pickup, $dropoff);
        $duration = $this->getEstimatedDuration($pickup, $dropoff);
        $surge = $this->getSurgeMultiplier($area, $vehicleType);

        $rates = config("pricing.{$vehicleType}");
        $baseFare = $rates['base_fare'] + ($distance * $rates['per_km']) + ($duration * $rates['per_min']);

        return [
            'total' => round($baseFare * $surge, 2),
            'surge' => $surge,
            'distance_km' => $distance,
            'eta_minutes' => $duration,
        ];
    }

    /**
     * এলাকা ভিত্তিক surge multiplier
     * ঢাকায় Gulshan/Banani তে সন্ধ্যায় বেশি demand
     */
    private function getSurgeMultiplier(string $area, string $vehicleType): float
    {
        $key = "surge:{$area}:{$vehicleType}";
        $multiplier = Redis::get($key);

        if (!$multiplier) {
            // ডিমান্ড-সাপ্লাই ক্যালকুলেশন
            $requests = Redis::get("requests:count:{$area}") ?? 0;
            $drivers = Redis::scard("drivers:available:{$area}") ?? 1;
            $baseline = Redis::get("baseline:requests:{$area}") ?? 10;

            $demandFactor = $requests / max($baseline, 1);
            $supplyFactor = $drivers / max(config("pricing.normal_drivers.{$area}"), 1);

            $rawMultiplier = $demandFactor / max($supplyFactor, 0.1);
            $multiplier = min($rawMultiplier, 5.0); // Max 5x cap
            $multiplier = max($multiplier, 1.0);    // Min 1x

            Redis::setex($key, 300, $multiplier); // 5 min cache
        }

        return (float) $multiplier;
    }
}
```

### 6. Node.js (Express) Code: Real-time Location & WebSocket

```javascript
// location-service/server.js
const express = require('express');
const { Server } = require('socket.io');
const Redis = require('ioredis');
const http = require('http');

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: '*' },
  transports: ['websocket', 'polling'], // ঢাকায় 2G তে polling fallback
});

const redis = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
  { host: 'redis-node-3', port: 6379 },
]);

const pubsub = new Redis();

// ড্রাইভারের লোকেশন আপডেট হ্যান্ডলার
io.of('/driver').on('connection', (socket) => {
  const driverId = socket.handshake.auth.driverId;
  console.log(`Driver connected: ${driverId}`);

  // প্রতি ৪ সেকেন্ডে ড্রাইভার লোকেশন পাঠাবে
  socket.on('location:update', async (data) => {
    const { lat, lng, heading, speed } = data;

    try {
      // Redis Geospatial Index-এ আপডেট
      await redis.geoadd('drivers:available', lng, lat, driverId);

      // Driver-এর বিস্তারিত তথ্য hash-এ রাখা
      await redis.hset(`driver:${driverId}:location`, {
        lat: lat.toString(),
        lng: lng.toString(),
        heading: heading.toString(),
        speed: speed.toString(),
        updated_at: Date.now().toString(),
      });

      // Geohash বের করা (area tracking এর জন্য)
      const geohash = computeGeohash(lat, lng, 6); // precision 6 ~ 1.2km
      await redis.sadd(`drivers:area:${geohash}`, driverId);

      // যদি এই ড্রাইভার কোনো active trip-এ থাকে, rider-কে notify করো
      const activeTrip = await redis.get(`driver:${driverId}:active_trip`);
      if (activeTrip) {
        pubsub.publish(`trip:${activeTrip}:location`, JSON.stringify({
          driverId,
          lat,
          lng,
          heading,
          speed,
          timestamp: Date.now(),
        }));
      }
    } catch (err) {
      console.error(`Location update failed for ${driverId}:`, err);
    }
  });

  // ড্রাইভার অফলাইন হলে
  socket.on('disconnect', async () => {
    await redis.zrem('drivers:available', driverId);
    await redis.del(`driver:${driverId}:location`);
    console.log(`Driver disconnected: ${driverId}`);
  });
});

// রাইডারের লাইভ ট্র্যাকিং
io.of('/rider').on('connection', (socket) => {
  const riderId = socket.handshake.auth.riderId;

  socket.on('track:trip', async (tripId) => {
    // Trip-এর location channel subscribe করো
    const subscriber = new Redis();
    subscriber.subscribe(`trip:${tripId}:location`);

    subscriber.on('message', (channel, message) => {
      const locationData = JSON.parse(message);
      socket.emit('driver:location', locationData);
    });

    socket.on('disconnect', () => {
      subscriber.unsubscribe(`trip:${tripId}:location`);
      subscriber.quit();
    });
  });
});

// Matching Engine - নিকটতম ড্রাইভার খোঁজা
app.post('/api/match/find-drivers', async (req, res) => {
  const { lat, lng, radius = 3, vehicleType, count = 10 } = req.body;

  try {
    // Redis GEORADIUS দিয়ে nearby drivers খোঁজা
    const nearbyDrivers = await redis.georadius(
      'drivers:available',
      lng, lat,
      radius, 'km',
      'WITHCOORD', 'WITHDIST', 'ASC', 'COUNT', count
    );

    // ফিল্টার ও স্কোরিং
    const scoredDrivers = await Promise.all(
      nearbyDrivers.map(async ([driverId, distance, [dLng, dLat]]) => {
        const driverInfo = await redis.hgetall(`driver:${driverId}:info`);

        // Vehicle type ফিল্টার
        if (driverInfo.vehicle_type !== vehicleType) return null;

        // ETA calculate (ঢাকার speed factor ~ 15km/h peak time)
        const avgSpeedKmh = getAreaSpeedFactor(lat, lng);
        const etaMinutes = (parseFloat(distance) / avgSpeedKmh) * 60;

        // Matching Score
        const score = calculateMatchScore({
          eta: etaMinutes,
          rating: parseFloat(driverInfo.rating || 4.5),
          acceptRate: parseFloat(driverInfo.accept_rate || 0.8),
          distance: parseFloat(distance),
        });

        return {
          driver_id: driverId,
          lat: parseFloat(dLat),
          lng: parseFloat(dLng),
          distance_km: parseFloat(distance),
          eta_minutes: Math.round(etaMinutes),
          rating: parseFloat(driverInfo.rating),
          score,
        };
      })
    );

    const validDrivers = scoredDrivers
      .filter(Boolean)
      .sort((a, b) => b.score - a.score);

    res.json({ drivers: validDrivers });
  } catch (err) {
    console.error('Matching error:', err);
    res.status(500).json({ error: 'Matching failed' });
  }
});

/**
 * ম্যাচিং স্কোর ক্যালকুলেশন
 * ঢাকায় ETA-র ওজন বেশি কারণ ট্রাফিক unpredictable
 */
function calculateMatchScore({ eta, rating, acceptRate, distance }) {
  const etaScore = Math.max(0, 1 - (eta / 15)) * 40;      // 40% weight
  const ratingScore = (rating / 5) * 30;                    // 30% weight
  const acceptScore = acceptRate * 20;                      // 20% weight
  const distScore = Math.max(0, 1 - (distance / 5)) * 10;  // 10% weight

  return etaScore + ratingScore + acceptScore + distScore;
}

/**
 * এলাকা ভিত্তিক গড় গতি (km/h)
 * ঢাকার বিভিন্ন এলাকায় ভিন্ন ভিন্ন
 */
function getAreaSpeedFactor(lat, lng) {
  // Simplified - actual implementation uses ML model
  const hour = new Date().getHours();
  const isPeakHour = (hour >= 8 && hour <= 10) || (hour >= 17 && hour <= 20);

  if (isPeakHour) return 8;   // পিক আওয়ার: ৮ km/h (Dhaka reality!)
  return 20;                    // Off-peak: ২০ km/h
}

function computeGeohash(lat, lng, precision) {
  // Geohash encoding implementation
  const BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz';
  let minLat = -90, maxLat = 90, minLng = -180, maxLng = 180;
  let hash = '';
  let isLng = true;
  let bit = 0, ch = 0;

  while (hash.length < precision) {
    if (isLng) {
      const mid = (minLng + maxLng) / 2;
      if (lng >= mid) { ch |= (1 << (4 - bit)); minLng = mid; }
      else { maxLng = mid; }
    } else {
      const mid = (minLat + maxLat) / 2;
      if (lat >= mid) { ch |= (1 << (4 - bit)); minLat = mid; }
      else { maxLat = mid; }
    }
    isLng = !isLng;
    if (bit < 4) { bit++; }
    else { hash += BASE32[ch]; bit = 0; ch = 0; }
  }
  return hash;
}

server.listen(3000, () => {
  console.log('Location service running on port 3000');
});
```

### 7. Database Schema

```
┌─────────────────────────────────────────────────────┐
│                 PostgreSQL Schema                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  users                    trips                      │
│  ├── id (UUID)           ├── id (UUID)              │
│  ├── name                ├── rider_id (FK)          │
│  ├── phone               ├── driver_id (FK)         │
│  ├── email               ├── pickup_lat/lng         │
│  ├── type (rider/driver) ├── dropoff_lat/lng        │
│  ├── rating              ├── status (enum)          │
│  └── created_at          ├── vehicle_type           │
│                          ├── estimated_fare          │
│  drivers                 ├── actual_fare             │
│  ├── user_id (FK)       ├── surge_multiplier        │
│  ├── vehicle_type        ├── distance_km            │
│  ├── vehicle_number      ├── duration_min           │
│  ├── license_no          ├── payment_method         │
│  ├── is_online           ├── payment_status         │
│  ├── current_lat/lng     ├── started_at             │
│  ├── accept_rate         ├── ended_at               │
│  └── total_rides         └── created_at             │
│                                                      │
│  payments                 ratings                    │
│  ├── id                  ├── id                     │
│  ├── trip_id (FK)        ├── trip_id (FK)           │
│  ├── amount              ├── from_user_id           │
│  ├── method              ├── to_user_id             │
│  ├── provider (bkash/    ├── score (1-5)            │
│  │   nagad/card/cash)    ├── comment                │
│  ├── transaction_id      └── created_at             │
│  ├── status                                         │
│  └── processed_at                                   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              Redis Data Structures                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Geo Set:    drivers:available                       │
│              (lng, lat, driver_id)                   │
│                                                      │
│  Hash:       driver:{id}:location                   │
│              {lat, lng, heading, speed, updated_at}  │
│                                                      │
│  Hash:       driver:{id}:info                       │
│              {vehicle_type, rating, accept_rate}     │
│                                                      │
│  String:     driver:{id}:active_trip                │
│              trip_id                                 │
│                                                      │
│  String:     surge:{area}:{vehicle_type}            │
│              multiplier value                        │
│                                                      │
│  Set:        drivers:area:{geohash}                 │
│              {driver_id_1, driver_id_2, ...}        │
│                                                      │
│  Counter:    requests:count:{area}                  │
│              atomic increment per request            │
└─────────────────────────────────────────────────────┘
```

### 8. ETA Calculation

```
┌─────────────────────────────────────────────────────┐
│           ETA Calculation Pipeline                    │
│                                                      │
│  Input: pickup (lat,lng) → dropoff (lat,lng)        │
│                                                      │
│  Step 1: Route Finding                              │
│  ├── OSRM / Google Directions API                   │
│  ├── Consider: one-way streets, road closures       │
│  └── Output: route polyline + base distance         │
│                                                      │
│  Step 2: Traffic Adjustment                         │
│  ├── Real-time traffic data (Google Traffic)        │
│  ├── Historical patterns (ML model)                 │
│  │   ├── Day of week                               │
│  │   ├── Time of day                               │
│  │   ├── Weather (বর্ষায় ঢাকা = nightmare)        │
│  │   └── Special events (ঈদ, হরতাল)               │
│  └── Output: adjusted travel time                   │
│                                                      │
│  Step 3: Pickup ETA                                 │
│  ├── Driver current location → Pickup point         │
│  ├── Factor: ঢাকার signal wait time (~3min/signal) │
│  └── Output: driver arrival time                    │
│                                                      │
│  ঢাকা-specific adjustments:                        │
│  ├── Farmgate: +5 min (always jammed)              │
│  ├── Mirpur-10: +8 min (roundabout chaos)          │
│  ├── Airport Road: -3 min (relatively clear)       │
│  └── Rain: multiply ETA × 2.5                      │
└─────────────────────────────────────────────────────┘
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Geohash vs QuadTree vs H3

```
┌────────────┬────────────────┬────────────────┬────────────────┐
│  Feature   │    Geohash     │   QuadTree     │      H3        │
├────────────┼────────────────┼────────────────┼────────────────┤
│ Complexity │ সহজ            │ মাঝারি          │ মাঝারি          │
│ Precision  │ Fixed levels   │ Adaptive       │ Fixed (16 res) │
│ Neighbor   │ Edge cases আছে │ সহজ            │ সেরা            │
│ query      │                │                │                │
│ Redis      │ ✅ Built-in     │ ❌ Custom       │ ⚠️ Library     │
│ support    │                │ impl needed    │   needed       │
│ Density    │ ❌ Fixed grid   │ ✅ Adaptive     │ ✅ Hexagonal   │
│ handling   │                │                │   uniform      │
│ Use case   │ Simple MVP     │ Variable       │ Production     │
│            │ (Obhai-তে OK)  │ density areas  │ (Uber uses)    │
└────────────┴────────────────┴────────────────┴────────────────┘

সিদ্ধান্ত: বাংলাদেশে Redis Geohash দিয়ে শুরু করুন (সহজ + Redis built-in)।
Scale বাড়লে H3-তে migrate করুন (Uber এটাই করেছে)।
```

### 2. WebSocket vs MQTT vs Server-Sent Events

```
┌────────────┬─────────────────┬─────────────────┬──────────────────┐
│  Feature   │   WebSocket     │     MQTT        │      SSE         │
├────────────┼─────────────────┼─────────────────┼──────────────────┤
│ Direction  │ Bidirectional   │ Pub/Sub         │ Server → Client  │
│ Overhead   │ Low after       │ Very Low        │ Medium           │
│            │ handshake       │ (2 byte header) │                  │
│ Battery    │ মাঝারি           │ ✅ সেরা          │ মাঝারি            │
│ 2G/3G     │ ⚠️ Reconnect    │ ✅ QoS levels   │ ❌ Poor          │
│ support    │ issues          │                 │                  │
│ Complexity │ মাঝারি           │ বেশি (broker)    │ সহজ              │
│ Scaling    │ Sticky sessions │ Broker cluster  │ সহজ              │
│ Best for   │ Live tracking   │ IoT/Low battery │ One-way updates  │
└────────────┴─────────────────┴─────────────────┴──────────────────┘

সিদ্ধান্ত: 
- Driver → Server: MQTT (battery efficient, 2G friendly)
- Server → Rider: WebSocket (bidirectional needed for interaction)
- ঢাকায় 2G/3G network অনেক জায়গায় — MQTT-র QoS Level 1 দিয়ে
  guaranteed delivery পাওয়া যায়
```

### 3. Matching Strategy

```
┌────────────────────┬─────────────────────────────────────────┐
│ Strategy           │ বিবরণ                                    │
├────────────────────┼─────────────────────────────────────────┤
│ Nearest First      │ সবচেয়ে কাছের ড্রাইভারকে পাঠাও          │
│  ✅ Simple         │ কিন্তু ঢাকায় "কাছে" মানে "তাড়াতাড়ি    │
│  ❌ Not optimal    │ পৌঁছাবে" না (traffic dependent)          │
├────────────────────┼─────────────────────────────────────────┤
│ ETA-Based          │ Road network + traffic দিয়ে actual ETA  │
│  ✅ Better UX      │ হিসেব করে সবচেয়ে কম ETA-র ড্রাইভার     │
│  ❌ Compute heavy  │ Pathao এটা ব্যবহার করে                    │
├────────────────────┼─────────────────────────────────────────┤
│ Batch Matching     │ ১০-১৫ সেকেন্ড requests জমিয়ে globally  │
│  ✅ Global optimum │ optimal matching করা (Hungarian algo)   │
│  ❌ Added latency  │ Uber এটা ব্যবহার করে peak hours-এ       │
└────────────────────┴─────────────────────────────────────────┘

Recommendation: ETA-based for normal time, Batch during peak
(ঈদের সময় বা ভারি বৃষ্টিতে batch matching ON করো)
```

### 4. Consistency vs Availability (Trip State)

```
┌──────────────────────────────────────────────────────────┐
│                    CAP Theorem Trade-off                   │
│                                                          │
│  Trip State Machine:                                     │
│  ├── AP (Available + Partition tolerant)                 │
│  │   ├── Location updates (eventual consistency OK)     │
│  │   ├── ETA calculation (stale data acceptable)        │
│  │   └── Driver availability (2-3 sec lag OK)           │
│  │                                                       │
│  └── CP (Consistent + Partition tolerant)                │
│      ├── Trip creation (double booking prevention)      │
│      ├── Payment processing (must be consistent)        │
│      ├── Driver assignment (one trip at a time)         │
│      └── Fare calculation (accurate amount)             │
│                                                          │
│  Implementation:                                         │
│  ├── Redis (AP): Location, caching, counters            │
│  ├── PostgreSQL (CP): Trips, payments, users            │
│  └── Kafka (AP): Event streaming, analytics             │
└──────────────────────────────────────────────────────────┘
```

---

## 📈 কেস স্টাডি

### কেস ১: ঈদের রাতে Dhaka-তে 10x Demand

```
সমস্যা:
ঈদুল ফিতরের আগের রাত, ঢাকা থেকে সবাই গ্রামে যাচ্ছে।
স্বাভাবিক demand: 500 requests/min
ঈদ রাতে: 5,000 requests/min (10x!)
Available drivers: স্বাভাবিকের ২০% (তারাও বাড়ি যাচ্ছে)

Impact:
├── Matching latency: 1 min → 8-10 min
├── Server CPU: 95%+ (overloaded)
├── Redis memory: spike from failed retries
└── User experience: অধিকাংশ ride cancel হচ্ছে

সমাধান কৌশল:
┌────────────────────────────────────────────────┐
│ 1. Pre-scaling (ঈদের ১ সপ্তাহ আগে)            │
│    ├── Auto-scaling rules: 3x normal capacity  │
│    ├── CDN warm-up                             │
│    └── Database read replicas বাড়ানো            │
│                                                │
│ 2. Surge Pricing Activation                    │
│    ├── Progressive surge: 1.5x → 2x → 3x     │
│    ├── Cap at 5x (social responsibility)      │
│    ├── Advance booking discount                │
│    └── "ভাড়া বেশি হতে পারে" advance warning   │
│                                                │
│ 3. Supply-side Incentives                      │
│    ├── ঈদ বোনাস: প্রতি রাইডে +৫০ টাকা         │
│    ├── Streak bonus: ৫টা পর পর রাইডে ৫০০ টাকা  │
│    ├── Guaranteed hourly earning               │
│    └── "আজ রাতে ড্রাইভ করুন" push notification │
│                                                │
│ 4. Demand Management                           │
│    ├── Ride pooling encourage (30% discount)   │
│    ├── Scheduled rides ২৪ ঘণ্টা আগে enable     │
│    ├── Alternative transport suggest            │
│    │   (Shohoz bus, train tickets)             │
│    └── Queue system with transparent wait time │
│                                                │
│ 5. Technical Optimizations                     │
│    ├── Batch matching ON (15 sec window)       │
│    ├── Reduce location update to every 10 sec  │
│    ├── Circuit breaker for non-critical APIs   │
│    └── Cache aggressive (ETA, fare estimates)  │
└────────────────────────────────────────────────┘

Result (Pathao ঈদ ২০২৩):
├── ৮০% রাইড successful (target ছিল ৭০%)
├── Average wait time: ১২ মিনিট
├── Driver earnings: স্বাভাবিকের ৩.৫ গুণ
└── Zero downtime achieved
```

### কেস ২: ভারি বৃষ্টিতে Surge Pricing Backlash

```
সমস্যা:
ঢাকায় হঠাৎ ভারি বৃষ্টি, রাস্তা ডুবে গেছে।
Surge: 4.5x → ৫০ টাকার রাইড ২২৫ টাকা!
Social media-তে #BoycottPathao trending।

Root Cause Analysis:
├── Algorithm শুধু demand/supply দেখছে
├── Weather emergency consider করছে না
├── কোনো cap ছিল না
└── Communication failure (কেন বেশি charge তা explain হয়নি)

সমাধান:
┌────────────────────────────────────────────────┐
│ 1. Smart Surge (Weather-aware)                 │
│    ├── Weather API integrate                   │
│    ├── Emergency situations: max 2x cap        │
│    ├── Gradual increase (jump নয়)             │
│    └── "বৃষ্টির কারণে ভাড়া বেশি" message show  │
│                                                │
│ 2. Communication Strategy                      │
│    ├── Surge কেন হচ্ছে explain (in-app)        │
│    ├── "X মিনিট পর কমতে পারে" prediction       │
│    ├── Alternative suggest: "কাছের shelter      │
│    │   তে অপেক্ষা করুন"                        │
│    └── Post-ride surge explanation              │
│                                                │
│ 3. Social Safety Net                           │
│    ├── Hospital trips: no surge ever           │
│    ├── Night emergency: capped at 1.5x         │
│    └── Loyalty riders: surge discount          │
└────────────────────────────────────────────────┘
```

### কেস ৩: Dense Dhaka Traffic-এ Driver-Rider Matching

```
সমস্যা:
Gulshan-2 চৌরাস্তায় rider আছে। ২০০ মিটারে ৫ জন driver আছে
কিন্তু U-turn restriction এর কারণে actual ETA ১৫-২০ মিনিট!
নিকটতম (straight-line) driver আসলে সবচেয়ে দূরে।

            ┌─── One Way ───▶
            │
   Driver A │  ×  (U-turn নিষেধ)
   (200m)   │
            │         ┌── Rider ──┐
            │         └───────────┘
            │
   Driver B │  (1.2km কিন্তু সঠিক দিকে)
            │
            └─── One Way ───▶

সমাধান:
├── Straight-line distance নয়, road-network ETA ব্যবহার
├── OSRM (Open Source Routing Machine) integration
├── One-way street data incorporate
├── Historical trip data দিয়ে ML model train:
│   "এই road segment-এ এই সময়ে average speed X km/h"
└── Result: Driver B select হবে (actual ETA কম)
```

---

## 🔧 Advanced Topics

### 1. Ride Pooling (Shared Rides)

```
┌──────────────────────────────────────────────────────────┐
│              Ride Pooling Algorithm                        │
│                                                          │
│  Rider A: Uttara → Gulshan (08:30 AM)                   │
│  Rider B: Uttara → Banani (08:32 AM)                    │
│                                                          │
│  Route without pooling:                                  │
│    Trip 1: Uttara ──────────────────▶ Gulshan            │
│    Trip 2: Uttara ──────────────────▶ Banani             │
│    Total distance: 24 km (2 separate trips)             │
│                                                          │
│  Route with pooling:                                     │
│    Shared: Uttara ──▶ Banani (drop B) ──▶ Gulshan       │
│    Total distance: 14 km (shared trip)                  │
│    Savings: 40% distance, 30% fare each rider           │
│                                                          │
│  Matching Criteria:                                      │
│  ├── Pickup locations within 500m                       │
│  ├── Drop-off in same general direction                 │
│  ├── Detour < 25% of original route                     │
│  ├── Time window: 5 min flexibility                     │
│  └── Both riders opted-in for pooling                   │
│                                                          │
│  ঢাকায় Pool Ride-র সুবিধা:                              │
│  ├── ভাড়া ৩০-৪০% কম                                     │
│  ├── ঢাকার রাস্তায় গাড়ি কম (traffic কমায়)              │
│  └── ড্রাইভার earning/km বেশি                            │
└──────────────────────────────────────────────────────────┘
```

### 2. Fraud Detection System

```
┌──────────────────────────────────────────────────────────┐
│                Fraud Detection Pipeline                    │
│                                                          │
│  Real-time Checks:                                       │
│  ├── GPS Spoofing: location jumps > possible speed      │
│  │   (ঢাকায় ড্রাইভার GPS fake করে fare বাড়ায়)          │
│  ├── Trip Manipulation: unusual route (longer path)     │
│  ├── Fake Ratings: same IP/device multiple accounts     │
│  └── Payment Fraud: stolen cards, bKash account takeover│
│                                                          │
│  ML-based Detection:                                     │
│  ├── Feature: trip_distance vs straight_line_distance   │
│  │   (ratio > 2.5 → suspicious)                        │
│  ├── Feature: driver_speed anomalies                    │
│  ├── Feature: frequent short trips (same pickup/drop)  │
│  └── Feature: rating patterns (all 5s or all 1s)       │
│                                                          │
│  Bangladesh-specific Fraud:                              │
│  ├── ড্রাইভার রাইড শুরু করে দাঁড়িয়ে থাকে (time charge) │
│  ├── GPS off করে ঘুরপথে নিয়ে যাওয়া (নতুন rider)       │
│  ├── Fake driver accounts (verification bypass)        │
│  └── Cash payment নিয়ে "app-এ paid" mark করা            │
│                                                          │
│  Prevention:                                             │
│  ├── Route deviation alert (rider notification)         │
│  ├── Photo verification (driver NID match)             │
│  ├── Trip recording (audio/video opt-in)               │
│  └── bKash verification (linked account only)          │
└──────────────────────────────────────────────────────────┘
```

### 3. Driver Incentive System

```
┌──────────────────────────────────────────────────────────┐
│             Driver Incentive Engine                        │
│                                                          │
│  Goal: Supply বাড়ানো এবং driver retain করা               │
│                                                          │
│  Incentive Types:                                        │
│  ├── Streak Bonus: পর পর ৫ রাইড → ৩০০ টাকা extra        │
│  ├── Peak Hour Bonus: সকাল ৮-১০, সন্ধ্যা ৬-৮ → +৩০%    │
│  ├── Area Bonus: driver কম আছে এমন এলাকায় → +২০%       │
│  ├── Quest: "আজ ১৫টা রাইড complete → ১০০০ টাকা"         │
│  └── Loyalty Tier: Bronze → Silver → Gold → Platinum     │
│                                                          │
│  Algorithm (simplified):                                 │
│  incentive_budget = f(area_demand, current_supply,       │
│                       time_of_day, driver_tier)          │
│                                                          │
│  Tier Benefits (Pathao-style):                           │
│  ├── Bronze:  Standard commission (20%)                 │
│  ├── Silver:  18% commission + priority matching         │
│  ├── Gold:    15% commission + fuel allowance            │
│  └── Platinum: 12% commission + insurance + training     │
└──────────────────────────────────────────────────────────┘
```

### 4. Multi-modal Transport Integration

```
┌──────────────────────────────────────────────────────────┐
│         Multi-modal Transport (Shohoz + Pathao)           │
│                                                          │
│  Uttara → Motijheel (Office commute)                    │
│                                                          │
│  Option A: Pure ride = ১ ঘণ্টা ৪৫ মিনিট, ৪৫০ টাকা      │
│                                                          │
│  Option B: Multi-modal                                   │
│  ┌─────┐    ┌──────────┐    ┌─────────┐    ┌─────┐     │
│  │Bike │───▶│  Metro   │───▶│  Metro  │───▶│Bike │     │
│  │Ride │    │  Uttara  │    │Motijheel│    │Ride │     │
│  │৫ min│    │  ২০ min  │    │  Station│    │৫ min│     │
│  │৪০৳  │    │  ৫০৳     │    │         │    │৪০৳  │     │
│  └─────┘    └──────────┘    └─────────┘    └─────┘     │
│                                                          │
│  Total: ৩৫ মিনিট, ১৩০ টাকা (৭০% cheaper, ৬৫% faster)  │
│                                                          │
│  Integration Points:                                     │
│  ├── Shohoz bus/train schedule API                      │
│  ├── Dhaka Metro Rail real-time data                    │
│  ├── First-mile / Last-mile bike ride                   │
│  └── Single payment across all modes                   │
└──────────────────────────────────────────────────────────┘
```

### 5. Route Optimization

```php
<?php
// Route Optimization - ঢাকার রাস্তার জন্য customized

class RouteOptimizer
{
    /**
     * ঢাকার ট্রাফিক-aware routing
     * Standard routing ঢাকায় কাজ করে না কারণ:
     * - Signal এ ১০-১৫ মিনিট আটকে যায়
     * - U-turn পয়েন্ট limited
     * - বৃষ্টি হলে waterlogging
     */
    public function findOptimalRoute(array $origin, array $destination): array
    {
        // OSRM থেকে alternative routes নাও
        $routes = $this->getAlternativeRoutes($origin, $destination, count: 3);

        $scoredRoutes = array_map(function ($route) {
            $trafficScore = $this->getTrafficScore($route['segments']);
            $waterlogScore = $this->getWaterlogRisk($route['segments']);
            $signalCount = $this->countTrafficSignals($route['segments']);

            // ঢাকায় signal count অনেক important factor
            $signalDelay = $signalCount * 3; // average 3 min per signal

            $adjustedEta = $route['base_eta'] + $signalDelay + 
                          ($trafficScore * 0.5) + ($waterlogScore * 2);

            return [
                ...$route,
                'adjusted_eta' => $adjustedEta,
                'signal_count' => $signalCount,
                'waterlog_risk' => $waterlogScore > 0.7 ? 'HIGH' : 'LOW',
            ];
        }, $routes);

        usort($scoredRoutes, fn($a, $b) => $a['adjusted_eta'] <=> $b['adjusted_eta']);

        return $scoredRoutes[0]; // সবচেয়ে কম ETA-র route
    }
}
```

---

## 🎯 সারসংক্ষেপ

### মূল শিক্ষা (Key Takeaways)

```
┌──────────────────────────────────────────────────────────┐
│              System Design Summary                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Location is King                                     │
│     └── প্রতি সেকেন্ডে হাজার হাজার location update      │
│         handle করতে Redis Geospatial + WebSocket         │
│                                                          │
│  2. Matching ≠ Nearest                                   │
│     └── ঢাকায় "কাছে" ≠ "তাড়াতাড়ি পৌঁছাবে"            │
│         ETA-based matching is essential                   │
│                                                          │
│  3. Surge Pricing is Art + Science                       │
│     └── Algorithm ঠিক কিন্তু communication ভুল হলে       │
│         user trust হারাবেন                                │
│                                                          │
│  4. Local Context Matters                                │
│     └── Uber-র global design সরাসরি ঢাকায় কাজ করবে না   │
│         (traffic, payment, network সব আলাদা)            │
│                                                          │
│  5. Payment Diversity                                    │
│     └── বাংলাদেশে cash ৭০%+, bKash/Nagad ২৫%            │
│         card মাত্র ৫% — সব support করতে হবে              │
│                                                          │
│  6. Reliability > Features                               │
│     └── ঈদে/বৃষ্টিতে app crash করলে user আর ফিরবে না    │
│         graceful degradation plan থাকতে হবে              │
└──────────────────────────────────────────────────────────┘
```

### Technology Stack Summary

```
┌─────────────────┬─────────────────────────────────────┐
│ Component       │ Technology                          │
├─────────────────┼─────────────────────────────────────┤
│ API Backend     │ PHP (Laravel) - Trip, User, Payment │
│ Real-time       │ Node.js (Express) + Socket.IO       │
│ Location Store  │ Redis (Geospatial + Pub/Sub)        │
│ Primary DB      │ PostgreSQL (Trips, Users, Payments) │
│ Cache           │ Redis Cluster                       │
│ Message Queue   │ Apache Kafka                        │
│ Search          │ Elasticsearch (Ride history)        │
│ Monitoring      │ Prometheus + Grafana                │
│ Maps/Routing    │ OSRM + Google Maps API              │
│ Push Notif.     │ FCM (Firebase Cloud Messaging)      │
│ Payment Gateway │ bKash, Nagad, SSLCommerz            │
│ Deployment      │ Kubernetes (EKS) + Docker           │
└─────────────────┴─────────────────────────────────────┘
```

### ইন্টারভিউ টিপস

```
এই system design discuss করার সময়:

✅ DO:
├── প্রথমে requirements clearly define করুন
├── Back-of-envelope calculation দেখান
├── Trade-off explain করুন (কেন X, Y না)
├── Local context mention করুন (interviewer impressed হবে)
└── Scaling strategy step-by-step বলুন

❌ DON'T:
├── সব feature একসাথে design করতে যাবেন না
├── Single point of failure রাখবেন না
├── "Uber exactly এভাবে করে" বলবেন না (আপনার চিন্তা দেখান)
└── Payment/Security skip করবেন না
```

---

> 💡 **পরবর্তী পড়ুন**: Chat Messaging System Design, Food Delivery System Design
>
> 📚 **Reference**: Uber Engineering Blog, Pathao Tech Blog, System Design Interview by Alex Xu
