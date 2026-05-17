# 🏠 অনলাইন রেন্টাল প্ল্যাটফর্ম সিস্টেম ডিজাইন (Airbnb)

> **লেভেল**: অ্যাডভান্সড | **টেক স্ট্যাক**: PHP (Laravel) + Node.js (Express)
> **রিয়েল-ওয়ার্ল্ড উদাহরণ**: Airbnb, Booking.com, VRBO, BD Context: ShareTrip, GoZayaan

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
| 1 | Property Listing | Host property তৈরি করতে পারবে (photos, description, amenities, rules) |
| 2 | Search with Filters | Location, date, price range, amenities দিয়ে search করা যাবে |
| 3 | Booking Management | Guest booking request পাঠাবে, Host accept/reject করবে |
| 4 | Calendar Management | Host availability calendar manage করবে, blocked dates সেট করবে |
| 5 | Reviews & Ratings | Guest ও Host একে অপরকে review দিতে পারবে |
| 6 | Messaging | Host-Guest real-time messaging |
| 7 | Payment Processing | Secure payment, host payout, refund management |
| 8 | Wishlist | Guest পছন্দের property save করতে পারবে |

### Non-Functional Requirements

```
+------------------------------------------------------------------+
|  Non-Functional Requirements                                      |
+------------------------------------------------------------------+
|  🔍 Search Latency        : < 200ms (p99)                        |
|  🔒 Double Booking        : Zero tolerance (strong consistency)   |
|  🌐 Availability          : 99.99% uptime                        |
|  📍 Geo-Search            : Radius-based search within 50ms      |
|  📸 Image Loading         : < 1s for listing thumbnails          |
|  💳 Payment Processing    : PCI DSS compliant                    |
|  📱 Multi-Platform        : Web, iOS, Android                    |
|  🌏 Multi-Region          : Bangladesh, SE Asia support           |
+------------------------------------------------------------------+
```

### 🇧🇩 Bangladesh Context

- **Cox's Bazar**: Peak season (Eid, Winter) এ hotel/resort booking rush
- **Sajek Valley / Bandarban**: Hill tract resorts, limited availability
- **Dhaka**: Short-term apartment rental, corporate stays
- **Payment**: bKash, Nagad, DBBL Nexus, Visa/Mastercard support
- **Language**: Bengali + English bilingual support
- **Challenges**: Intermittent internet, mobile-first users, trust issues

---

## 📊 Back-of-envelope Estimation

### ট্র্যাফিক অনুমান (Bangladesh Scale)

```
Total Users           : 5M registered users
Daily Active Users    : 500K DAU
Properties Listed     : 200K properties
Daily Searches        : 2M search queries
Daily Bookings        : 10K bookings
Peak Load (Eid)       : 5x normal traffic

Search QPS:
  - Average: 2M / 86400 ≈ 23 QPS
  - Peak (Eid season): 23 × 5 = 115 QPS

Booking QPS:
  - Average: 10K / 86400 ≈ 0.12 QPS
  - Peak: 0.6 QPS (booking is write-heavy, low QPS but critical)
```

### স্টোরেজ অনুমান

```
Property Data:
  - 200K properties × 5KB metadata = 1GB
  - 200K properties × 20 photos × 2MB = 8TB (images)
  - CDN cached images: ~2TB active cache

Database:
  - User data: 5M × 2KB = 10GB
  - Booking records: 3.6M/year × 1KB = 3.6GB/year
  - Messages: 50M messages/year × 0.5KB = 25GB/year
  - Reviews: 2M reviews × 1KB = 2GB

Search Index (Elasticsearch):
  - 200K properties × 10KB enriched doc = 2GB
```

### Bandwidth অনুমান

```
Incoming (Write):
  - Image uploads: 500 listings/day × 20 photos × 2MB = 20GB/day
  - API requests: 5M requests/day × 1KB avg = 5GB/day

Outgoing (Read):
  - Search results: 2M searches × 50KB (thumbnails) = 100GB/day
  - Property pages: 500K views × 5MB (full images) = 2.5TB/day
  - CDN offloads 90% → Origin serves ~250GB/day
```

---

## 🏗️ High-Level Design

### System Architecture (ASCII Diagram)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                               │
│  │ Web App  │  │ iOS App  │  │ Android  │                               │
│  │ (React)  │  │ (Swift)  │  │ (Kotlin) │                               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                               │
└───────┼──────────────┼──────────────┼───────────────────────────────────┘
        │              │              │
        ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        CDN (CloudFront/Cloudflare)                        │
│              Static assets, images, cached search results                 │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     API GATEWAY (Kong / Nginx)                            │
│         Rate limiting, Auth, Request routing, Load balancing              │
└──────┬──────────┬──────────┬──────────┬──────────┬─────────────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
┌──────────┐┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│  Search  ││ Booking  ││ Payment  ││ Messaging││  User    │
│ Service  ││ Service  ││ Service  ││ Service  ││ Service  │
│(Node.js) ││ (Laravel)││ (Laravel)││(Node.js) ││(Laravel) │
└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘
     │           │           │           │           │
     ▼           ▼           ▼           ▼           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                        │
│                                                                           │
│  ┌─────────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐        │
│  │ PostgreSQL  │  │  Redis   │  │Elasticsearch│ │ Object Store │        │
│  │ (Primary DB)│  │ (Cache)  │  │(Search Idx)│  │   (S3/R2)    │        │
│  └─────────────┘  └──────────┘  └───────────┘  └──────────────┘        │
│                                                                           │
│  ┌─────────────┐  ┌──────────┐  ┌───────────┐                           │
│  │  MongoDB    │  │  Kafka   │  │TimescaleDB│                           │
│  │ (Messages) │  │ (Events) │  │(Analytics) │                           │
│  └─────────────┘  └──────────┘  └───────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Service Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│ Service           │ Responsibility            │ Tech Stack   │
├───────────────────┼───────────────────────────┼──────────────┤
│ Search Service    │ Geo-search, filters,      │ Node.js +    │
│                   │ ranking, autocomplete     │ Elasticsearch│
├───────────────────┼───────────────────────────┼──────────────┤
│ Booking Service   │ Reservation, calendar,    │ Laravel +    │
│                   │ availability check        │ PostgreSQL   │
├───────────────────┼───────────────────────────┼──────────────┤
│ Payment Service   │ Charge, payout, refund,   │ Laravel +    │
│                   │ bKash/Nagad/Card          │ Stripe/SSLCommerz│
├───────────────────┼───────────────────────────┼──────────────┤
│ Messaging Service │ Real-time chat,           │ Node.js +    │
│                   │ notifications             │ Socket.io    │
├───────────────────┼───────────────────────────┼──────────────┤
│ User Service      │ Auth, profile,            │ Laravel +    │
│                   │ verification              │ PostgreSQL   │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 Detailed Design

### 1. 📍 Geo-Spatial Search

Airbnb-style platform এ সবচেয়ে critical feature হলো location-based search। 
Cox's Bazar এ "সমুদ্রের কাছে" resort search করতে গেলে geo-spatial query দরকার।

#### Geohash Approach

```
Geohash: পৃথিবীকে grid cells এ ভাগ করা

Precision Level:
  ┌───────────────────────────────────┐
  │ Geohash Length │ Cell Size        │
  ├────────────────┼──────────────────┤
  │      1         │ 5,009km × 4,992km│
  │      3         │ 156km × 156km   │
  │      5         │ 4.9km × 4.9km   │  ← City level
  │      6         │ 1.2km × 609m    │  ← Neighborhood
  │      7         │ 153m × 153m     │  ← Street level
  └───────────────────────────────────┘

Cox's Bazar Beach Area: geohash "wh3x" (precision 4)
Sajek Valley:           geohash "wh7p" (precision 4)

Nearby Search Strategy:
  - User location → Calculate geohash
  - Query geohash + 8 neighboring cells
  - Filter by exact distance

  ┌─────┬─────┬─────┐
  │ NW  │  N  │ NE  │
  ├─────┼─────┼─────┤
  │  W  │ CTR │  E  │  ← Center + 8 neighbors
  ├─────┼─────┼─────┤
  │ SW  │  S  │ SE  │
  └─────┴─────┴─────┘
```

#### Elasticsearch Geo-Search Implementation (Node.js)

```javascript
// services/search/src/controllers/searchController.js
const { Client } = require('@elastic/elasticsearch');

const esClient = new Client({ node: process.env.ELASTICSEARCH_URL });

// Property index mapping
const createPropertyIndex = async () => {
  await esClient.indices.create({
    index: 'properties',
    body: {
      mappings: {
        properties: {
          title: { type: 'text', analyzer: 'bengali_analyzer' },
          description: { type: 'text', analyzer: 'bengali_analyzer' },
          location: { type: 'geo_point' },
          geohash: { type: 'keyword' },
          price_per_night: { type: 'float' },
          property_type: { type: 'keyword' },
          amenities: { type: 'keyword' },
          rating: { type: 'float' },
          total_reviews: { type: 'integer' },
          is_available: { type: 'boolean' },
          city: { type: 'keyword' },
          district: { type: 'keyword' },
          max_guests: { type: 'integer' },
          created_at: { type: 'date' }
        }
      },
      settings: {
        analysis: {
          analyzer: {
            bengali_analyzer: {
              type: 'custom',
              tokenizer: 'standard',
              filter: ['lowercase', 'bengali_stemmer']
            }
          },
          filter: {
            bengali_stemmer: { type: 'stemmer', language: 'bengali' }
          }
        }
      }
    }
  });
};

// Geo-search API endpoint
const searchProperties = async (req, res) => {
  const {
    lat, lng, radius = '10km',
    check_in, check_out,
    min_price, max_price,
    guests, amenities,
    property_type, sort_by = 'relevance',
    page = 1, limit = 20
  } = req.query;

  // Build Elasticsearch query
  const must = [];
  const filter = [];

  // Geo-distance filter
  if (lat && lng) {
    filter.push({
      geo_distance: {
        distance: radius,
        location: { lat: parseFloat(lat), lon: parseFloat(lng) }
      }
    });
  }

  // Price range filter
  if (min_price || max_price) {
    const range = {};
    if (min_price) range.gte = parseFloat(min_price);
    if (max_price) range.lte = parseFloat(max_price);
    filter.push({ range: { price_per_night: range } });
  }

  // Guest capacity filter
  if (guests) {
    filter.push({ range: { max_guests: { gte: parseInt(guests) } } });
  }

  // Amenities filter
  if (amenities) {
    const amenityList = amenities.split(',');
    amenityList.forEach(amenity => {
      filter.push({ term: { amenities: amenity.trim() } });
    });
  }

  // Property type filter
  if (property_type) {
    filter.push({ term: { property_type } });
  }

  // Availability filter (check against booking service)
  filter.push({ term: { is_available: true } });

  // Sort configuration
  let sort = [];
  switch (sort_by) {
    case 'price_low':
      sort = [{ price_per_night: 'asc' }];
      break;
    case 'price_high':
      sort = [{ price_per_night: 'desc' }];
      break;
    case 'rating':
      sort = [{ rating: 'desc' }];
      break;
    case 'distance':
      sort = [{
        _geo_distance: {
          location: { lat: parseFloat(lat), lon: parseFloat(lng) },
          order: 'asc', unit: 'km'
        }
      }];
      break;
    default:
      // Relevance-based scoring with boost factors
      must.push({
        function_score: {
          functions: [
            { field_value_factor: { field: 'rating', factor: 1.5, modifier: 'sqrt' } },
            { field_value_factor: { field: 'total_reviews', factor: 1.2, modifier: 'log1p' } },
            {
              gauss: {
                location: {
                  origin: { lat: parseFloat(lat), lon: parseFloat(lng) },
                  scale: '5km', decay: 0.5
                }
              }
            }
          ],
          score_mode: 'multiply'
        }
      });
  }

  const body = {
    from: (page - 1) * limit,
    size: limit,
    query: { bool: { must, filter } },
    sort,
    highlight: {
      fields: { title: {}, description: {} }
    }
  };

  try {
    const result = await esClient.search({ index: 'properties', body });

    const properties = result.body.hits.hits.map(hit => ({
      id: hit._id,
      ...hit._source,
      score: hit._score,
      distance: hit.sort ? hit.sort[0] : null,
      highlights: hit.highlight
    }));

    res.json({
      success: true,
      data: properties,
      meta: {
        total: result.body.hits.total.value,
        page: parseInt(page),
        limit: parseInt(limit),
        took_ms: result.body.took
      }
    });
  } catch (error) {
    console.error('Search error:', error);
    res.status(500).json({ success: false, message: 'Search failed' });
  }
};

module.exports = { searchProperties, createPropertyIndex };
```

### 2. 📅 Booking & Availability Calendar

#### Double Booking Prevention Strategy

```
Double Booking Problem:
  Guest A এবং Guest B একই সময়ে Cox's Bazar এর
  একটি resort room বুক করতে চাইছে।

  Timeline:
  ─────────────────────────────────────────────────
  T1: Guest A checks availability → Room available ✓
  T2: Guest B checks availability → Room available ✓
  T3: Guest A confirms booking → Booked ✓
  T4: Guest B confirms booking → ??? CONFLICT!
  ─────────────────────────────────────────────────

Solution: Pessimistic Locking + Atomic Transaction

  ┌─────────────────────────────────────────────────┐
  │         Booking Flow (Race Condition Safe)       │
  │                                                  │
  │  1. BEGIN TRANSACTION                            │
  │  2. SELECT ... FOR UPDATE (lock the dates)       │
  │  3. Check availability                           │
  │  4. INSERT booking record                        │
  │  5. UPDATE calendar (mark dates blocked)         │
  │  6. COMMIT TRANSACTION                           │
  │                                                  │
  │  If conflict → ROLLBACK → Return "unavailable"   │
  └─────────────────────────────────────────────────┘
```

#### PHP (Laravel) - Booking Service with Availability Check

```php
<?php
// app/Services/BookingService.php

namespace App\Services;

use App\Models\Booking;
use App\Models\PropertyCalendar;
use App\Models\Property;
use App\Events\BookingCreated;
use App\Exceptions\DoubleBookingException;
use App\Exceptions\PropertyUnavailableException;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;
use Carbon\Carbon;
use Carbon\CarbonPeriod;

class BookingService
{
    private PaymentService $paymentService;
    private NotificationService $notificationService;

    public function __construct(
        PaymentService $paymentService,
        NotificationService $notificationService
    ) {
        $this->paymentService = $paymentService;
        $this->notificationService = $notificationService;
    }

    /**
     * Property booking তৈরি করা - Double booking prevention সহ
     * Pessimistic locking ব্যবহার করে race condition handle করা হচ্ছে
     */
    public function createBooking(array $data): Booking
    {
        $property = Property::findOrFail($data['property_id']);
        $checkIn = Carbon::parse($data['check_in']);
        $checkOut = Carbon::parse($data['check_out']);
        $guests = $data['guests'];

        // Basic validation
        $this->validateBookingDates($checkIn, $checkOut);
        $this->validateGuestCapacity($property, $guests);

        // Calculate pricing
        $pricing = $this->calculatePricing($property, $checkIn, $checkOut, $guests);

        return DB::transaction(function () use ($property, $data, $checkIn, $checkOut, $pricing) {
            // Pessimistic lock - SELECT FOR UPDATE
            // এটি অন্য concurrent transactions কে wait করতে বাধ্য করবে
            $lockedDates = PropertyCalendar::where('property_id', $property->id)
                ->whereBetween('date', [$checkIn->toDateString(), $checkOut->subDay()->toDateString()])
                ->lockForUpdate()
                ->get();

            // Check if any date is already booked
            $unavailableDates = $lockedDates->where('status', '!=', 'available');
            if ($unavailableDates->isNotEmpty()) {
                throw new DoubleBookingException(
                    'নির্বাচিত তারিখগুলো ইতিমধ্যে বুক করা হয়েছে: ' .
                    $unavailableDates->pluck('date')->implode(', ')
                );
            }

            // Create booking record
            $booking = Booking::create([
                'property_id' => $property->id,
                'guest_id' => $data['user_id'],
                'host_id' => $property->host_id,
                'check_in' => $checkIn,
                'check_out' => $data['check_out'],
                'guests' => $data['guests'],
                'total_price' => $pricing['total'],
                'service_fee' => $pricing['service_fee'],
                'cleaning_fee' => $pricing['cleaning_fee'],
                'taxes' => $pricing['taxes'],
                'currency' => $pricing['currency'],
                'status' => 'pending_payment',
                'booking_code' => $this->generateBookingCode(),
            ]);

            // Mark calendar dates as booked
            PropertyCalendar::where('property_id', $property->id)
                ->whereBetween('date', [$checkIn->toDateString(), Carbon::parse($data['check_out'])->subDay()->toDateString()])
                ->update([
                    'status' => 'booked',
                    'booking_id' => $booking->id,
                ]);

            // Invalidate search cache for this property
            Cache::tags(['property_' . $property->id, 'search'])->flush();

            // Process payment hold
            $paymentIntent = $this->paymentService->createHold(
                $booking,
                $pricing['total'],
                $data['payment_method']
            );

            $booking->update([
                'payment_intent_id' => $paymentIntent->id,
                'status' => 'confirmed',
            ]);

            // Dispatch events
            event(new BookingCreated($booking));

            // Notify host
            $this->notificationService->notifyHost($booking, 'new_booking');

            Log::info('Booking created successfully', [
                'booking_id' => $booking->id,
                'property_id' => $property->id,
                'dates' => $checkIn->toDateString() . ' to ' . $data['check_out'],
            ]);

            return $booking;
        }, 5); // 5 retries on deadlock
    }

    /**
     * Dynamic pricing calculation
     * Eid, peak season, weekend এ বেশি দাম
     */
    private function calculatePricing(Property $property, Carbon $checkIn, Carbon $checkOut, int $guests): array
    {
        $nights = $checkIn->diffInDays($checkOut);
        $totalBasePrice = 0;

        // Date-wise pricing (seasonal/dynamic)
        $period = CarbonPeriod::create($checkIn, $checkOut->copy()->subDay());
        foreach ($period as $date) {
            $dailyRate = $this->getDailyRate($property, $date);
            $totalBasePrice += $dailyRate;
        }

        // Extra guest fee
        $extraGuestFee = 0;
        if ($guests > $property->base_guests) {
            $extraGuests = $guests - $property->base_guests;
            $extraGuestFee = $extraGuests * $property->extra_guest_fee * $nights;
        }

        $subtotal = $totalBasePrice + $extraGuestFee;
        $cleaningFee = $property->cleaning_fee;
        $serviceFee = $subtotal * 0.12; // 12% service fee
        $taxes = ($subtotal + $serviceFee) * 0.15; // 15% VAT (Bangladesh)

        return [
            'base_price' => $totalBasePrice,
            'extra_guest_fee' => $extraGuestFee,
            'cleaning_fee' => $cleaningFee,
            'service_fee' => round($serviceFee, 2),
            'taxes' => round($taxes, 2),
            'total' => round($subtotal + $cleaningFee + $serviceFee + $taxes, 2),
            'currency' => 'BDT',
            'nights' => $nights,
        ];
    }

    /**
     * Daily rate with seasonal pricing
     * Cox's Bazar peak: Eid, December-February
     */
    private function getDailyRate(Property $property, Carbon $date): float
    {
        // Check for custom pricing (host-set special dates)
        $customRate = PropertyCalendar::where('property_id', $property->id)
            ->where('date', $date->toDateString())
            ->value('custom_price');

        if ($customRate) return $customRate;

        // Seasonal multiplier
        $multiplier = 1.0;

        // Eid season detection (approximate)
        if ($this->isEidSeason($date)) {
            $multiplier = 2.0; // Eid এ দ্বিগুণ দাম
        }
        // Winter peak (Cox's Bazar)
        elseif ($date->month >= 11 || $date->month <= 2) {
            $multiplier = 1.5;
        }
        // Weekend premium
        elseif ($date->isWeekend()) {
            $multiplier = 1.2;
        }

        return $property->base_price * $multiplier;
    }

    private function isEidSeason(Carbon $date): bool
    {
        // Simplified - real implementation would use Hijri calendar
        $eidDates = Cache::remember('eid_dates_' . $date->year, 86400, function () use ($date) {
            return config('seasons.eid_dates.' . $date->year, []);
        });

        foreach ($eidDates as $eidRange) {
            if ($date->between(Carbon::parse($eidRange['start']), Carbon::parse($eidRange['end']))) {
                return true;
            }
        }
        return false;
    }

    private function validateBookingDates(Carbon $checkIn, Carbon $checkOut): void
    {
        if ($checkIn->isPast()) {
            throw new \InvalidArgumentException('Check-in date অতীতের হতে পারবে না');
        }
        if ($checkOut->lte($checkIn)) {
            throw new \InvalidArgumentException('Check-out date অবশ্যই check-in এর পরে হতে হবে');
        }
        if ($checkIn->diffInDays($checkOut) > 365) {
            throw new \InvalidArgumentException('Maximum 365 দিনের booking করা যাবে');
        }
    }

    private function validateGuestCapacity(Property $property, int $guests): void
    {
        if ($guests > $property->max_guests) {
            throw new \InvalidArgumentException(
                "এই property এ সর্বোচ্চ {$property->max_guests} জন guest থাকতে পারবে"
            );
        }
    }

    private function generateBookingCode(): string
    {
        return 'BK-' . strtoupper(substr(md5(uniqid()), 0, 8));
    }
}
```

### 3. 🔍 Search Ranking Algorithm

```
Search Ranking Score = weighted combination of factors

┌──────────────────────────────────────────────────────────┐
│           SEARCH RANKING FORMULA                          │
│                                                           │
│  Score = w1 × Relevance                                   │
│        + w2 × Distance_Decay                              │
│        + w3 × Quality_Score                               │
│        + w4 × Price_Competitiveness                       │
│        + w5 × Recency_Boost                               │
│        + w6 × Conversion_History                          │
│                                                           │
│  Weights (example):                                       │
│    w1 = 0.25 (text match relevance)                       │
│    w2 = 0.20 (proximity to search center)                 │
│    w3 = 0.25 (rating × review_count)                      │
│    w4 = 0.10 (price vs market average)                    │
│    w5 = 0.10 (recently updated listings)                  │
│    w6 = 0.10 (past booking conversion rate)               │
└──────────────────────────────────────────────────────────┘

Quality Score Breakdown:
  ┌────────────────────┐
  │ Rating (4.8/5)  40%│
  │ Response Rate   20%│
  │ Review Count    20%│
  │ Photo Quality   10%│
  │ Profile Complete10%│
  └────────────────────┘
```

### 4. 🗄️ Database Schema

```sql
-- Properties Table
CREATE TABLE properties (
    id BIGSERIAL PRIMARY KEY,
    host_id BIGINT REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    title_bn VARCHAR(200),  -- Bengali title
    description TEXT,
    description_bn TEXT,    -- Bengali description
    property_type VARCHAR(50),  -- apartment, house, resort, houseboat
    location GEOGRAPHY(POINT, 4326),  -- PostGIS
    address JSONB,  -- {street, city, district, division, postal_code}
    geohash VARCHAR(12),
    price_per_night DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'BDT',
    cleaning_fee DECIMAL(10,2) DEFAULT 0,
    max_guests INTEGER DEFAULT 2,
    bedrooms INTEGER DEFAULT 1,
    bathrooms INTEGER DEFAULT 1,
    amenities TEXT[],  -- {wifi, ac, pool, parking, kitchen}
    house_rules JSONB,
    check_in_time TIME DEFAULT '14:00',
    check_out_time TIME DEFAULT '11:00',
    rating DECIMAL(3,2) DEFAULT 0,
    total_reviews INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    instant_book BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_properties_location ON properties USING GIST(location);
CREATE INDEX idx_properties_geohash ON properties(geohash);
CREATE INDEX idx_properties_price ON properties(price_per_night);
CREATE INDEX idx_properties_host ON properties(host_id);

-- Availability Calendar
CREATE TABLE property_calendar (
    id BIGSERIAL PRIMARY KEY,
    property_id BIGINT REFERENCES properties(id),
    date DATE NOT NULL,
    status VARCHAR(20) DEFAULT 'available',  -- available, booked, blocked
    custom_price DECIMAL(10,2),
    booking_id BIGINT REFERENCES bookings(id),
    min_nights INTEGER DEFAULT 1,
    UNIQUE(property_id, date)
);

CREATE INDEX idx_calendar_property_date ON property_calendar(property_id, date);
CREATE INDEX idx_calendar_status ON property_calendar(status);

-- Bookings Table
CREATE TABLE bookings (
    id BIGSERIAL PRIMARY KEY,
    booking_code VARCHAR(20) UNIQUE,
    property_id BIGINT REFERENCES properties(id),
    guest_id BIGINT REFERENCES users(id),
    host_id BIGINT REFERENCES users(id),
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    guests INTEGER NOT NULL,
    total_price DECIMAL(12,2),
    service_fee DECIMAL(10,2),
    cleaning_fee DECIMAL(10,2),
    taxes DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'BDT',
    status VARCHAR(20) DEFAULT 'pending',
    -- pending, confirmed, checked_in, completed, cancelled, refunded
    payment_intent_id VARCHAR(100),
    payment_method VARCHAR(50),  -- bkash, nagad, card, bank_transfer
    cancellation_reason TEXT,
    special_requests TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_bookings_guest ON bookings(guest_id);
CREATE INDEX idx_bookings_host ON bookings(host_id);
CREATE INDEX idx_bookings_property_dates ON bookings(property_id, check_in, check_out);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Reviews Table
CREATE TABLE reviews (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT REFERENCES bookings(id) UNIQUE,
    reviewer_id BIGINT REFERENCES users(id),
    reviewee_id BIGINT REFERENCES users(id),
    property_id BIGINT REFERENCES properties(id),
    type VARCHAR(20),  -- guest_to_host, host_to_guest
    overall_rating DECIMAL(2,1),  -- 1.0 to 5.0
    cleanliness DECIMAL(2,1),
    accuracy DECIMAL(2,1),
    communication DECIMAL(2,1),
    location_rating DECIMAL(2,1),
    value DECIMAL(2,1),
    comment TEXT,
    comment_bn TEXT,
    host_response TEXT,
    is_public BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Messages Table (MongoDB schema)
-- messages collection:
-- {
--   _id: ObjectId,
--   conversation_id: "conv_123",
--   sender_id: 456,
--   receiver_id: 789,
--   booking_id: 101,
--   message: "কখন check-in করতে পারবো?",
--   type: "text|image|booking_request",
--   read_at: ISODate,
--   created_at: ISODate
-- }
```

### 5. 💬 Real-time Messaging (Node.js + Socket.io)

```javascript
// services/messaging/src/server.js
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const mongoose = require('mongoose');
const redis = require('redis');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: process.env.ALLOWED_ORIGINS?.split(',') }
});

const redisClient = redis.createClient({ url: process.env.REDIS_URL });

// Message Schema (MongoDB)
const messageSchema = new mongoose.Schema({
  conversationId: { type: String, index: true },
  senderId: { type: Number, required: true },
  receiverId: { type: Number, required: true },
  bookingId: { type: Number },
  message: { type: String, required: true },
  type: { type: String, enum: ['text', 'image', 'booking_update'], default: 'text' },
  readAt: { type: Date, default: null },
  createdAt: { type: Date, default: Date.now }
});

const Message = mongoose.model('Message', messageSchema);

// Socket.io authentication middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const user = await verifyJWT(token);
    socket.userId = user.id;
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.userId} connected`);

  // Track online status in Redis
  redisClient.set(`user:online:${socket.userId}`, 'true', 'EX', 300);

  // Join user's personal room
  socket.join(`user:${socket.userId}`);

  // Send message
  socket.on('send_message', async (data) => {
    const { receiverId, message, conversationId, bookingId, type } = data;

    const newMessage = await Message.create({
      conversationId,
      senderId: socket.userId,
      receiverId,
      bookingId,
      message,
      type: type || 'text'
    });

    // Emit to receiver if online
    io.to(`user:${receiverId}`).emit('new_message', {
      id: newMessage._id,
      conversationId,
      senderId: socket.userId,
      message,
      type: newMessage.type,
      createdAt: newMessage.createdAt
    });

    // Push notification if receiver offline
    const isOnline = await redisClient.get(`user:online:${receiverId}`);
    if (!isOnline) {
      await sendPushNotification(receiverId, {
        title: 'নতুন মেসেজ',
        body: message.substring(0, 100),
        data: { conversationId, bookingId }
      });
    }
  });

  // Mark messages as read
  socket.on('mark_read', async ({ conversationId }) => {
    await Message.updateMany(
      { conversationId, receiverId: socket.userId, readAt: null },
      { readAt: new Date() }
    );
  });

  socket.on('disconnect', () => {
    redisClient.del(`user:online:${socket.userId}`);
  });
});
```

### 6. 💳 Payment Flow

```
Payment Flow (Bangladesh Context):

┌──────┐    ┌───────────┐    ┌──────────────┐    ┌──────────┐
│Guest │    │  Booking  │    │   Payment    │    │  Payment │
│      │───▶│  Service  │───▶│   Service    │───▶│ Gateway  │
│      │    │ (Laravel) │    │  (Laravel)   │    │(SSLCommerz│
└──────┘    └───────────┘    └──────────────┘    │/ bKash)  │
                                                  └────┬─────┘
                                                       │
                              ┌─────────────────────────┘
                              ▼
                 ┌────────────────────────┐
                 │   Payment States       │
                 │                        │
                 │  ┌─────────┐           │
                 │  │ Pending │           │
                 │  └────┬────┘           │
                 │       │                │
                 │  ┌────▼────┐           │
                 │  │  Hold   │ (amount   │
                 │  │ Created │  reserved)│
                 │  └────┬────┘           │
                 │       │                │
                 │  ┌────▼────┐           │
                 │  │Captured │ (after    │
                 │  │         │  check-in)│
                 │  └────┬────┘           │
                 │       │                │
                 │  ┌────▼────┐           │
                 │  │  Host   │ (24hrs    │
                 │  │ Payout  │  after    │
                 │  │         │  check-in)│
                 │  └─────────┘           │
                 └────────────────────────┘

Host Payout Distribution:
  Total Payment: ৳10,000
  ├── Platform Fee (12%): ৳1,200
  ├── Payment Gateway (2.5%): ৳250
  ├── VAT (15% on fees): ৳217
  └── Host Receives: ৳8,333
```

---

## ⚖️ ট্রেড-অফ বিশ্লেষণ

### 1. Elasticsearch vs PostGIS for Geo-Search

```
┌──────────────────┬─────────────────────┬─────────────────────┐
│   Criteria       │   Elasticsearch     │     PostGIS          │
├──────────────────┼─────────────────────┼─────────────────────┤
│ Query Speed      │ ⚡ Very fast (ms)   │ 🔄 Fast (10-50ms)   │
│ Full-text Search │ ✅ Excellent        │ ❌ Limited           │
│ Faceted Search   │ ✅ Built-in aggs    │ ❌ Manual            │
│ Geo Precision    │ 🔄 Good             │ ✅ Excellent         │
│ Complex Shapes   │ 🔄 Basic            │ ✅ Full topology     │
│ Scaling          │ ✅ Horizontal       │ 🔄 Vertical mainly   │
│ Consistency      │ ❌ Eventually       │ ✅ Strong (ACID)     │
│ Operational Cost │ ❌ High (cluster)   │ ✅ Lower (extension) │
│ Bangladesh Fit   │ ✅ Better for       │ ✅ Better for        │
│                  │   search platform   │   precise mapping   │
└──────────────────┴─────────────────────┴─────────────────────┘

আমাদের সিদ্ধান্ত: Elasticsearch (primary search) + PostGIS (booking validation)
কারণ: Search latency < 200ms requirement + full-text Bengali search needed
```

### 2. Geohash vs Quadtree vs S2 Geometry

```
┌──────────────────┬───────────┬───────────┬──────────────┐
│   Feature        │  Geohash  │ Quadtree  │ S2 Geometry  │
├──────────────────┼───────────┼───────────┼──────────────┤
│ Implementation   │  Simple   │  Medium   │   Complex    │
│ Storage          │  String   │  Tree     │   Cell IDs   │
│ Prefix Query     │  ✅ Yes   │  ❌ No    │   ✅ Yes     │
│ Edge Handling    │  ❌ Poor  │  ✅ Good  │   ✅ Best    │
│ Non-rectangular  │  ❌ No    │  ✅ Yes   │   ✅ Yes     │
│ DB Support       │  ✅ Wide  │  🔄 Custom│   🔄 Limited │
│ Cox's Bazar fit  │  ✅ Good  │  ✅ Good  │   ✅ Best    │
│ Used by          │ Elasticsearch│ Uber  │   Google     │
└──────────────────┴───────────┴───────────┴──────────────┘

আমাদের সিদ্ধান্ত: Geohash (Elasticsearch native support) + S2 for precise areas
```

### 3. Strong vs Eventual Consistency for Bookings

```
┌──────────────────────────────────────────────────────────┐
│                 Consistency Decision Matrix                │
├──────────────────┬──────────────────┬────────────────────┤
│   Operation      │   Consistency    │   Reason           │
├──────────────────┼──────────────────┼────────────────────┤
│ Booking Create   │ STRONG           │ Double booking=$$  │
│ Calendar Update  │ STRONG           │ Tied to booking    │
│ Payment Process  │ STRONG           │ Money involved     │
│ Search Results   │ EVENTUAL (30s)   │ Stale OK briefly   │
│ Review Display   │ EVENTUAL (5min)  │ Not time-critical  │
│ Message Delivery │ EVENTUAL (1s)    │ Best-effort quick  │
│ Analytics        │ EVENTUAL (hours) │ Batch processing OK│
│ Image CDN        │ EVENTUAL (1hr)   │ Cache invalidation │
└──────────────────┴──────────────────┴────────────────────┘

Pattern: Strong consistency শুধু money + availability operations এ।
বাকি সব eventual consistency (better performance, availability)।
```

### 4. Monolith vs Microservices

```
বাংলাদেশের startup context এ:

Phase 1 (MVP, 0-10K users): Modular Monolith (Laravel)
  ┌────────────────────────────────────┐
  │         Laravel Monolith            │
  │  ┌────────┐ ┌────────┐ ┌────────┐ │
  │  │ Search │ │Booking │ │Payment │ │
  │  │ Module │ │ Module │ │ Module │ │
  │  └────────┘ └────────┘ └────────┘ │
  │          Single Database            │
  └────────────────────────────────────┘
  ✅ Fast development, simple deployment
  ✅ Single Dhaka server sufficient
  ❌ Limited scaling

Phase 2 (Growth, 10K-100K users): Extract critical services
  - Search → Node.js + Elasticsearch (separate)
  - Messaging → Node.js + Socket.io (separate)
  - Rest stays Laravel monolith

Phase 3 (Scale, 100K+ users): Full microservices
  - All services independent
  - Kubernetes deployment
  - Multi-region (Dhaka + Singapore)
```

---

## 📈 কেস স্টাডি

### 🏖️ Case 1: Cox's Bazar Eid Vacation Rush

```
Scenario: ঈদুল ফিতর এর ১ সপ্তাহ আগে Cox's Bazar booking spike

Timeline:
───────────────────────────────────────────────────────────
  ঈদের ৩০ দিন আগে     : Normal traffic (100 bookings/day)
  ঈদের ১৪ দিন আগে     : 3x spike (300 bookings/day)
  ঈদের ৭ দিন আগে      : 10x spike (1000 bookings/day)
  ঈদের দিন              : Last-minute cancellations + rebooking
───────────────────────────────────────────────────────────

Problems Encountered:
  1. Database connection pool exhaustion (max 100 connections)
  2. Double booking incidents (3 cases in first hour)
  3. Payment gateway timeout (bKash server overloaded)
  4. Search latency spike: 200ms → 2000ms
  5. Image CDN bandwidth exceeded

Solutions Implemented:
  ┌─────────────────────────────────────────────────────────┐
  │ Problem              │ Solution                          │
  ├──────────────────────┼──────────────────────────────────┤
  │ DB connections       │ PgBouncer connection pooling      │
  │                      │ Read replicas for search queries  │
  ├──────────────────────┼──────────────────────────────────┤
  │ Double booking       │ Redis distributed lock +          │
  │                      │ SELECT FOR UPDATE + retry logic   │
  ├──────────────────────┼──────────────────────────────────┤
  │ Payment timeout      │ Queue-based processing,           │
  │                      │ fallback to Nagad/Card            │
  ├──────────────────────┼──────────────────────────────────┤
  │ Search latency       │ Pre-computed results cache,       │
  │                      │ Elasticsearch replica shards      │
  ├──────────────────────┼──────────────────────────────────┤
  │ CDN bandwidth        │ Image optimization pipeline,      │
  │                      │ WebP format, lazy loading         │
  └─────────────────────────────────────────────────────────┘

Auto-scaling Config:
  - Normal: 2 app servers, 1 DB primary + 1 replica
  - Eid mode: 8 app servers, 1 DB primary + 3 replicas
  - Trigger: CPU > 70% OR response time > 500ms
```

### 📅 Case 2: Calendar Sync Across Platforms

```
Problem: 
  Host রহিম তার Cox's Bazar resort তিনটি platform এ list করেছেন:
  - আমাদের platform
  - Booking.com
  - Agoda

  Guest A আমাদের platform এ book করলো কিন্তু Booking.com 
  এ সেই dates available দেখাচ্ছে → Double booking!

Solution: iCal Sync + Webhook Integration

  ┌──────────────┐     iCal Export     ┌──────────────┐
  │ Our Platform │ ──────────────────▶  │  Booking.com │
  │              │ ◀──────────────────  │              │
  └──────────────┘     iCal Import     └──────────────┘
         │                                      │
         │          ┌──────────────┐            │
         └─────────▶│  Calendar    │◀───────────┘
                    │  Sync Worker │
                    │  (every 15m) │
                    └──────────────┘

  Sync Strategy:
  1. Real-time: Webhook on booking → instant block on other platforms
  2. Periodic: iCal sync every 15 minutes as fallback
  3. Conflict resolution: First-write-wins (our platform priority)
  
  Edge Case:
  - Network failure during sync → Mark dates as "tentative"
  - Show warning to guest: "এই dates অন্য platform এও available"
```

### 💰 Case 3: Host Payout & Dispute Resolution

```
Scenario: 
  Guest Karim ১ রাত থেকে complain করলো AC কাজ করছে না।
  ৳5,000/night এর booking এ refund চাইছে।

Dispute Resolution Flow:
  ┌────────┐         ┌──────────┐         ┌──────────┐
  │ Guest  │──রিপোর্ট──▶│Resolution│──নোটিফাই──▶│  Host   │
  │ Karim  │         │  Center  │         │  Rahim  │
  └────────┘         └────┬─────┘         └────┬─────┘
                          │                     │
                     ┌────▼─────┐          ┌───▼────┐
                     │ Evidence │          │Response│
                     │ Review   │◀─────────│(24 hr) │
                     └────┬─────┘          └────────┘
                          │
                ┌─────────┼─────────┐
                ▼         ▼         ▼
          ┌──────────┐┌───────┐┌──────────┐
          │Full Refund││Partial││No Refund │
          │  ৳5,000  ││৳2,500 ││  ৳0      │
          └──────────┘└───────┘└──────────┘

  Payout Hold Timeline:
  - Booking payment: Hold immediately
  - Check-in confirmed: Still on hold
  - 24 hours after check-in: Release to host (if no dispute)
  - Dispute filed: Extend hold, investigate
  - Resolution: Partial/full refund or release to host
```

---

## 🔧 Advanced Topics

### 1. 🖼️ Image Processing Pipeline

```
Image Upload → Processing Pipeline:

┌────────┐    ┌──────────┐    ┌───────────┐    ┌────────┐
│ Upload │───▶│  Queue   │───▶│ Processor │───▶│  CDN   │
│  API   │    │ (Redis)  │    │ (Worker)  │    │ (S3+CF)│
└────────┘    └──────────┘    └───────────┘    └────────┘

Processing Steps:
  1. Virus scan (ClamAV)
  2. EXIF data extraction (location, camera info)
  3. NSFW content detection (ML model)
  4. Resize to multiple sizes:
     - Thumbnail: 300×200 (listing card)
     - Medium: 800×600 (listing page)
     - Large: 1920×1080 (gallery view)
     - Original: stored for host
  5. Convert to WebP + fallback JPEG
  6. Watermark for unbooked views (optional)
  7. Upload to S3 with CDN distribution

Storage Strategy:
  s3://rentals-bd/properties/{property_id}/
    ├── original/  (host access only)
    ├── large/     (gallery)
    ├── medium/    (listing page)
    └── thumb/     (search results, cards)

Bangladesh Optimization:
  - WebP format (30% smaller than JPEG)
  - Progressive loading for slow connections
  - Regional CDN edge (Singapore/Mumbai)
  - Lazy loading below-the-fold images
```

### 2. 🤖 Smart Pricing ML Model

```
Dynamic Pricing Factors:

┌──────────────────────────────────────────────────────────┐
│                PRICING MODEL INPUTS                        │
├──────────────────────────────────────────────────────────┤
│ Market Demand Signals:                                    │
│   • Search volume for area (last 7 days)                 │
│   • Booking rate for similar properties                   │
│   • Competitor pricing (Booking.com, Agoda)              │
│   • Time until check-in (last-minute vs advance)         │
│                                                           │
│ Property Signals:                                         │
│   • Historical occupancy rate                             │
│   • Rating & review velocity                              │
│   • Amenity uniqueness score                              │
│   • Photo quality score (ML-assessed)                    │
│                                                           │
│ External Signals:                                         │
│   • Season (Eid, winter, rainy)                          │
│   • Local events (cricket match, fair)                   │
│   • Weather forecast                                      │
│   • Day of week                                           │
│   • Government holidays                                   │
└──────────────────────────────────────────────────────────┘

Output: Suggested price range (min, recommended, max)

Example for Cox's Bazar Resort:
  - Base price: ৳5,000/night
  - Eid week: ৳5,000 × 2.0 = ৳10,000 (suggested)
  - Rainy season: ৳5,000 × 0.6 = ৳3,000 (suggested)
  - Weekend: ৳5,000 × 1.3 = ৳6,500 (suggested)
  - Last 3 days empty: ৳5,000 × 0.8 = ৳4,000 (fill pricing)
```

### 3. 🛡️ Fraud Detection

```
Fraud Types in Rental Platform:

┌──────────────────────────────────────────────────┐
│ Type              │ Detection Strategy            │
├───────────────────┼──────────────────────────────┤
│ Fake Listings     │ • Image reverse-search       │
│                   │ • NID verification           │
│                   │ • Physical visit verification│
│                   │ • Review pattern analysis    │
├───────────────────┼──────────────────────────────┤
│ Payment Fraud     │ • Velocity checks            │
│                   │ • Device fingerprinting      │
│                   │ • Unusual booking patterns   │
│                   │ • bKash/Nagad verification   │
├───────────────────┼──────────────────────────────┤
│ Review Fraud      │ • NLP sentiment analysis     │
│                   │ • Reviewer behavior patterns │
│                   │ • Time-between-reviews       │
│                   │ • Cross-reference bookings   │
├───────────────────┼──────────────────────────────┤
│ Off-platform      │ • Message scanning (phone    │
│ Booking           │   number, email detection)   │
│                   │ • Link detection in chat     │
└───────────────────┴──────────────────────────────┘

Bangladesh-specific Trust:
  - NID (National ID) verification via EC API
  - Mobile number verification (Grameenphone, Robi, etc.)
  - bKash account verification
  - Physical address verification (local partner)
```

### 4. 💱 Multi-Currency Support

```
Currency Flow for International Guests:

  Guest (USD) → Platform (BDT conversion) → Host (BDT payout)

  ┌──────────┐    ┌─────────────┐    ┌──────────────┐
  │  Guest   │    │  Currency   │    │    Host      │
  │ Pays $50 │───▶│  Service    │───▶│ Gets ৳5,500  │
  │          │    │ Rate: 1:110 │    │ (minus fees) │
  └──────────┘    └─────────────┘    └──────────────┘

  Exchange Rate Strategy:
  - Real-time rates from Bangladesh Bank API
  - 1% markup for currency risk
  - Rate locked at booking time (not payout time)
  - Display in guest's preferred currency
  - Settlement always in BDT for BD hosts

  Supported Currencies:
  ┌─────────────────────────────────────┐
  │ BDT (৳)  - Primary                  │
  │ USD ($)  - International guests     │
  │ GBP (£)  - UK diaspora             │
  │ EUR (€)  - European tourists        │
  │ INR (₹)  - Indian tourists          │
  │ SAR (﷼)  - Gulf BD workers          │
  └─────────────────────────────────────┘
```

### 5. 🔐 Trust & Safety System

```
Trust Score System:

  ┌────────────────────────────────────────────┐
  │         USER TRUST SCORE (0-100)            │
  │                                             │
  │  Identity Verification:     +20 points      │
  │    ✓ Email verified                         │
  │    ✓ Phone verified                         │
  │    ✓ NID uploaded                           │
  │    ✓ Selfie match                           │
  │                                             │
  │  Platform History:          +40 points      │
  │    ✓ Completed bookings (×2 each, max 20)  │
  │    ✓ Positive reviews (×3 each, max 15)    │
  │    ✓ Response rate > 90%     (+5)           │
  │                                             │
  │  Behavior Signals:          +40 points      │
  │    ✓ No cancellations        (+10)          │
  │    ✓ No disputes             (+10)          │
  │    ✓ Payment on time         (+10)          │
  │    ✓ Profile completeness    (+10)          │
  │                                             │
  │  Risk Penalties:                            │
  │    ✗ Cancellation            -10            │
  │    ✗ Negative review         -5             │
  │    ✗ Reported by other user  -15            │
  │    ✗ Suspicious activity     -20            │
  └────────────────────────────────────────────┘

  Trust Tier:
    90-100: Superhost / Trusted Guest (instant book)
    70-89:  Verified user (normal flow)
    50-69:  New/moderate user (extra verification)
    0-49:   Restricted (manual review required)
```

---

## 🎯 সারসংক্ষেপ

### Key Design Decisions Summary

```
┌─────────────────────────────────────────────────────────────┐
│              ARCHITECTURAL DECISIONS SUMMARY                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Search: Elasticsearch + Geohash                          │
│     → Fast geo-search, Bengali full-text, faceted results    │
│                                                              │
│  2. Booking: PostgreSQL + Pessimistic Locking                │
│     → Zero double-booking guarantee, ACID compliance         │
│                                                              │
│  3. Messaging: MongoDB + Socket.io                           │
│     → Flexible schema, real-time delivery                    │
│                                                              │
│  4. Caching: Redis (multi-purpose)                           │
│     → Session, search cache, rate limiting, pub/sub          │
│                                                              │
│  5. Payment: SSLCommerz + bKash API                          │
│     → Local payment support, escrow-style hold               │
│                                                              │
│  6. Images: S3 + CloudFront + WebP                           │
│     → Fast loading on mobile networks                        │
│                                                              │
│  7. Architecture: Start modular monolith → extract services  │
│     → Practical for BD startup budget and team size          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 🇧🇩 বাংলাদেশ Context এ মূল চ্যালেঞ্জ ও সমাধান

| চ্যালেঞ্জ | সমাধান |
|-----------|--------|
| Slow mobile internet (3G/4G) | WebP images, lazy loading, pagination, aggressive caching |
| Payment ecosystem fragmented | Multi-gateway: bKash + Nagad + Card + Bank |
| Trust deficit (new platform) | NID verification, review system, escrow payment |
| Seasonal demand (Eid spikes) | Auto-scaling, pre-warming cache, queue-based booking |
| Limited tech talent locally | Laravel (large BD community) + gradual Node.js adoption |
| Power/Internet outages | Offline-first PWA, SMS notifications, queued operations |

### Interview Tips

```
সিস্টেম ডিজাইন ইন্টারভিউতে এই বিষয়গুলো highlight করুন:

✅ Double booking prevention → Show you understand concurrency
✅ Geo-search tradeoffs → Show breadth of knowledge
✅ Consistency choices → Show pragmatic thinking
✅ Scaling strategy → Show evolutionary architecture
✅ Payment security → Show awareness of critical paths
✅ Bangladesh context → Show local market understanding

Common Follow-up Questions:
  Q: "How would you handle 10x traffic on Eid?"
  A: Pre-scale, cache popular searches, queue bookings, circuit breaker

  Q: "What if Elasticsearch goes down?"
  A: Fallback to PostgreSQL with PostGIS, degraded but functional

  Q: "How to prevent host from cancelling confirmed bookings?"
  A: Penalty system (super-host status loss), partial refund to guest automatic
```

---

> 📚 **আরও পড়ুন**: [Airbnb Engineering Blog](https://medium.com/airbnb-engineering) | [Booking.com Tech Blog](https://blog.booking.com)
> 
> 💡 **প্র্যাকটিস**: এই design কে নিজে whiteboard এ আঁকার চেষ্টা করুন - প্রতিটি component এর responsibility মনে রাখুন।
