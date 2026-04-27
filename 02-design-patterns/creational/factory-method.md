# 🏭 Factory Method প্যাটার্ন

## 📌 সংজ্ঞা ও মূল ধারণা

### GoF Definition

> **"Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."**
> — Gang of Four (Design Patterns: Elements of Reusable Object-Oriented Software)

Factory Method হলো একটি **Creational Design Pattern** যা object তৈরির জন্য একটি interface define করে, কিন্তু কোন concrete class instantiate হবে সেটা subclass-এর উপর ছেড়ে দেয়। এই pattern-এর মূল উদ্দেশ্য হলো **object creation logic কে usage logic থেকে আলাদা করা** — যাতে client code কখনোই সরাসরি `new` keyword দিয়ে concrete class তৈরি না করে।

এটি **Open/Closed Principle (OCP)** এর একটি চমৎকার বাস্তবায়ন — আপনি নতুন product type যোগ করতে পারবেন existing code modify না করেই।

### মূল চারটি অংশ (Participants)

| অংশ | ভূমিকা | উদাহরণ |
|------|--------|--------|
| **Product** (Interface/Abstract) | যে object তৈরি হবে তার contract define করে | `Transport` interface |
| **ConcreteProduct** | Product interface-এর নির্দিষ্ট implementation | `Truck`, `Ship`, `Plane` |
| **Creator** (Abstract) | Factory method declare করে, যা Product return করে | `Logistics` abstract class |
| **ConcreteCreator** | Factory method implement করে, নির্দিষ্ট product তৈরি করে | `RoadLogistics`, `SeaLogistics` |

### Intent বিশ্লেষণ

Factory Method pattern-এ তিনটি গুরুত্বপূর্ণ বিষয় লক্ষ্য করুন:

1. **Interface for creation**: Creator class-এ একটি method থাকে যা Product return করে
2. **Subclass decides**: কোন product তৈরি হবে সেটা ConcreteCreator ঠিক করে
3. **Defer instantiation**: Creator নিজে কখনো concrete product তৈরি করে না

এই pattern **Template Method pattern-এর specialized version** হিসেবেও কাজ করে — যেখানে template method হলো factory method নিজেই।

```
মনে রাখুন: Factory Method ≠ Simple Factory
Factory Method হলো pattern, Simple Factory হলো idiom।
Factory Method inheritance ব্যবহার করে, Simple Factory শুধু conditional logic।
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### Vehicle Factory — বাস্তব রূপক

ধরুন বাংলাদেশে একটি **যানবাহন উৎপাদন কোম্পানি** আছে। তাদের বিভিন্ন কারখানা আছে:

- **ঢাকা কারখানা** → Car তৈরি করে
- **চট্টগ্রাম কারখানা** → Bike তৈরি করে
- **গাজীপুর কারখানা** → Truck তৈরি করে

প্রতিটি কারখানা একই **"যানবাহন তৈরি করো"** নির্দেশনা অনুসরণ করে, কিন্তু প্রতিটি কারখানা **ভিন্ন ধরনের** যানবাহন তৈরি করে। এটাই Factory Method — **interface একটাই, কিন্তু output ভিন্ন**।

```
কোম্পানি বলে: "একটা যানবাহন তৈরি করো"
    ↓
ঢাকা কারখানা → 🚗 Car তৈরি করলো
চট্টগ্রাম কারখানা → 🏍️ Bike তৈরি করলো
গাজীপুর কারখানা → 🚛 Truck তৈরি করলো
```

এখানে গুরুত্বপূর্ণ বিষয় হলো — কোম্পানির main office-কে জানতে হয় না ঠিক কোন ধরনের যানবাহন তৈরি হচ্ছে। তারা শুধু জানে যে একটি `Vehicle` পাবে যেটায় `start()`, `stop()`, `getFuelType()` method আছে।

### বাংলাদেশের Logistics উদাহরণ

**পাঠাও, RedX, Paperfly** — এই কোম্পানিগুলো কিভাবে delivery manage করে চিন্তা করুন:

- **পাঠাও** → Bike rider দিয়ে delivery (শহরের ভিতরে)
- **RedX** → Van/Truck দিয়ে delivery (বড় পার্সেল)
- **Paperfly** → Courier service দিয়ে delivery (সারা দেশে)

প্রতিটি কোম্পানি `createTransport()` method আলাদাভাবে implement করে, কিন্তু delivery process (`planDelivery()`, `trackShipment()`) সবার জন্য একই থাকে।

---

## 📊 UML ডায়াগ্রাম

### Class Diagram (ASCII)

```
    ┌─────────────────────────────┐          ┌──────────────────────────┐
    │      <<interface>>          │          │     <<interface>>        │
    │         Creator             │          │        Product           │
    ├─────────────────────────────┤          ├──────────────────────────┤
    │                             │          │ + operation(): string    │
    ├─────────────────────────────┤          │ + getType(): string      │
    │ + factoryMethod(): Product  │─────────▶│                          │
    │ + someOperation(): string   │ creates  └──────────┬───────────────┘
    └──────────┬──────────────────┘                     │
               │                                        │
        ┌──────┴──────┐                        ┌───────┴────────┐
        │             │                        │                │
┌───────▼──────┐ ┌────▼──────────┐   ┌─────────▼────┐  ┌───────▼───────┐
│ Concrete     │ │ Concrete      │   │ Concrete     │  │ Concrete      │
│ Creator A    │ │ Creator B     │   │ Product A    │  │ Product B     │
├──────────────┤ ├───────────────┤   ├──────────────┤  ├───────────────┤
│ factory      │ │ factory       │   │ + operation()│  │ + operation() │
│ Method():    │ │ Method():     │   │ + getType()  │  │ + getType()   │
│ Product      │ │ Product       │   └──────────────┘  └───────────────┘
│ {return new  │ │ {return new   │
│  ProductA} │ │  ProductB}  │
└──────────────┘ └───────────────┘
```

### Sequence Diagram (ASCII)

```
Client          Creator           ConcreteCreator      ConcreteProduct
  │                │                     │                    │
  │  someOperation()                     │                    │
  │───────────────▶│                     │                    │
  │                │   factoryMethod()   │                    │
  │                │────────────────────▶│                    │
  │                │                     │   new Product()    │
  │                │                     │───────────────────▶│
  │                │                     │   <<creates>>      │
  │                │     product         │◀───────────────────│
  │                │◀────────────────────│                    │
  │                │                     │                    │
  │                │  product.operation()│                    │
  │                │─────────────────────────────────────────▶│
  │                │                     │       result       │
  │   result       │◀────────────────────────────────────────│
  │◀───────────────│                     │                    │
```

---

## 💻 ইমপ্লিমেন্টেশন

### Basic Factory Method

#### PHP 8.3

```php
<?php

declare(strict_types=1);

/**
 * Product Interface — সকল vehicle-এর জন্য common contract
 */
interface Vehicle
{
    public function start(): string;
    public function stop(): string;
    public function getFuelType(): FuelType;
    public function getMaxSpeed(): int;
}

/**
 * Fuel Type Enum — PHP 8.1+ Backed Enum
 */
enum FuelType: string
{
    case Petrol = 'petrol';
    case Diesel = 'diesel';
    case Electric = 'electric';
    case CNG = 'cng';

    public function label(): string
    {
        return match($this) {
            self::Petrol => 'পেট্রোল',
            self::Diesel => 'ডিজেল',
            self::Electric => 'ইলেকট্রিক',
            self::CNG => 'সিএনজি',
        };
    }
}

/**
 * ConcreteProduct — Car
 */
readonly class Car implements Vehicle
{
    public function __construct(
        private string $model = 'Toyota Corolla',
        private int $maxSpeed = 180,
    ) {}

    public function start(): string
    {
        return "🚗 {$this->model} গাড়ি চালু হয়েছে";
    }

    public function stop(): string
    {
        return "🚗 {$this->model} গাড়ি বন্ধ হয়েছে";
    }

    public function getFuelType(): FuelType
    {
        return FuelType::Petrol;
    }

    public function getMaxSpeed(): int
    {
        return $this->maxSpeed;
    }
}

/**
 * ConcreteProduct — Bike
 */
readonly class Bike implements Vehicle
{
    public function __construct(
        private string $model = 'Yamaha FZS',
        private int $maxSpeed = 130,
    ) {}

    public function start(): string
    {
        return "🏍️ {$this->model} বাইক চালু হয়েছে";
    }

    public function stop(): string
    {
        return "🏍️ {$this->model} বাইক বন্ধ হয়েছে";
    }

    public function getFuelType(): FuelType
    {
        return FuelType::Petrol;
    }

    public function getMaxSpeed(): int
    {
        return $this->maxSpeed;
    }
}

/**
 * ConcreteProduct — Truck
 */
readonly class Truck implements Vehicle
{
    public function __construct(
        private string $model = 'Tata LPT',
        private int $maxSpeed = 90,
    ) {}

    public function start(): string
    {
        return "🚛 {$this->model} ট্রাক চালু হয়েছে";
    }

    public function stop(): string
    {
        return "🚛 {$this->model} ট্রাক বন্ধ হয়েছে";
    }

    public function getFuelType(): FuelType
    {
        return FuelType::Diesel;
    }

    public function getMaxSpeed(): int
    {
        return $this->maxSpeed;
    }
}

/**
 * Creator — Abstract VehicleFactory
 * Factory method declare করা হয়েছে, implementation subclass-এ
 */
abstract class VehicleFactory
{
    // Factory Method — subclass এটা implement করবে
    abstract public function createVehicle(): Vehicle;

    /**
     * Creator-এর নিজস্ব business logic
     * Factory method ব্যবহার করে product তৈরি করে কাজ করে
     */
    public function deliverVehicle(): string
    {
        $vehicle = $this->createVehicle();

        $output = [];
        $output[] = $vehicle->start();
        $output[] = "জ্বালানি: {$vehicle->getFuelType()->label()}";
        $output[] = "সর্বোচ্চ গতি: {$vehicle->getMaxSpeed()} km/h";
        $output[] = $vehicle->stop();
        $output[] = "✅ যানবাহন সফলভাবে ডেলিভারি হয়েছে!";

        return implode("\n", $output);
    }
}

/**
 * ConcreteCreator — CarFactory
 */
class CarFactory extends VehicleFactory
{
    public function __construct(
        private readonly string $model = 'Toyota Corolla',
    ) {}

    public function createVehicle(): Vehicle
    {
        return new Car(model: $this->model);
    }
}

/**
 * ConcreteCreator — BikeFactory
 */
class BikeFactory extends VehicleFactory
{
    public function __construct(
        private readonly string $model = 'Yamaha FZS',
    ) {}

    public function createVehicle(): Vehicle
    {
        return new Bike(model: $this->model);
    }
}

/**
 * ConcreteCreator — TruckFactory
 */
class TruckFactory extends VehicleFactory
{
    public function __construct(
        private readonly string $model = 'Tata LPT',
    ) {}

    public function createVehicle(): Vehicle
    {
        return new Truck(model: $this->model);
    }
}

// Client Code — Factory Method ব্যবহার
function clientCode(VehicleFactory $factory): void
{
    echo $factory->deliverVehicle();
    echo "\n---\n";
}

clientCode(new CarFactory('Honda Civic'));
clientCode(new BikeFactory('Bajaj Pulsar'));
clientCode(new TruckFactory('Ashok Leyland'));
```

#### JavaScript ES2022+

```javascript
/**
 * FuelType — Object.freeze দিয়ে enum-like behavior
 */
const FuelType = Object.freeze({
  Petrol: { value: 'petrol', label: 'পেট্রোল' },
  Diesel: { value: 'diesel', label: 'ডিজেল' },
  Electric: { value: 'electric', label: 'ইলেকট্রিক' },
  CNG: { value: 'cng', label: 'সিএনজি' },
});

/**
 * ConcreteProduct — Car
 * ES2022 Private fields (#) ব্যবহার
 */
class Car {
  #model;
  #maxSpeed;

  constructor(model = 'Toyota Corolla', maxSpeed = 180) {
    this.#model = model;
    this.#maxSpeed = maxSpeed;
  }

  start() {
    return `🚗 ${this.#model} গাড়ি চালু হয়েছে`;
  }

  stop() {
    return `🚗 ${this.#model} গাড়ি বন্ধ হয়েছে`;
  }

  get fuelType() {
    return FuelType.Petrol;
  }

  get maxSpeed() {
    return this.#maxSpeed;
  }
}

/**
 * ConcreteProduct — Bike
 */
class Bike {
  #model;
  #maxSpeed;

  constructor(model = 'Yamaha FZS', maxSpeed = 130) {
    this.#model = model;
    this.#maxSpeed = maxSpeed;
  }

  start() {
    return `🏍️ ${this.#model} বাইক চালু হয়েছে`;
  }

  stop() {
    return `🏍️ ${this.#model} বাইক বন্ধ হয়েছে`;
  }

  get fuelType() {
    return FuelType.Petrol;
  }

  get maxSpeed() {
    return this.#maxSpeed;
  }
}

/**
 * ConcreteProduct — Truck
 */
class Truck {
  #model;
  #maxSpeed;

  constructor(model = 'Tata LPT', maxSpeed = 90) {
    this.#model = model;
    this.#maxSpeed = maxSpeed;
  }

  start() {
    return `🚛 ${this.#model} ট্রাক চালু হয়েছে`;
  }

  stop() {
    return `🚛 ${this.#model} ট্রাক বন্ধ হয়েছে`;
  }

  get fuelType() {
    return FuelType.Diesel;
  }

  get maxSpeed() {
    return this.#maxSpeed;
  }
}

/**
 * Creator — Abstract VehicleFactory
 */
class VehicleFactory {
  createVehicle() {
    throw new Error('Subclass অবশ্যই createVehicle() implement করবে');
  }

  deliverVehicle() {
    const vehicle = this.createVehicle();

    return [
      vehicle.start(),
      `জ্বালানি: ${vehicle.fuelType.label}`,
      `সর্বোচ্চ গতি: ${vehicle.maxSpeed} km/h`,
      vehicle.stop(),
      '✅ যানবাহন সফলভাবে ডেলিভারি হয়েছে!',
    ].join('\n');
  }
}

/**
 * ConcreteCreator — CarFactory
 */
class CarFactory extends VehicleFactory {
  #model;

  constructor(model = 'Toyota Corolla') {
    super();
    this.#model = model;
  }

  createVehicle() {
    return new Car(this.#model);
  }
}

class BikeFactory extends VehicleFactory {
  #model;

  constructor(model = 'Yamaha FZS') {
    super();
    this.#model = model;
  }

  createVehicle() {
    return new Bike(this.#model);
  }
}

class TruckFactory extends VehicleFactory {
  #model;

  constructor(model = 'Tata LPT') {
    super();
    this.#model = model;
  }

  createVehicle() {
    return new Truck(this.#model);
  }
}

// Client Code
function clientCode(factory) {
  console.log(factory.deliverVehicle());
  console.log('---');
}

clientCode(new CarFactory('Honda Civic'));
clientCode(new BikeFactory('Bajaj Pulsar'));
clientCode(new TruckFactory('Ashok Leyland'));
```

---

### Parameterized Factory Method

Parameterized Factory Method-এ factory method একটি **parameter গ্রহণ করে** এবং সেই parameter-এর উপর ভিত্তি করে বিভিন্ন ধরনের product তৈরি করে। এটি classic factory method-এর চেয়ে **বেশি flexible** কিন্তু কিছু OCP violation হতে পারে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

enum VehicleType: string
{
    case Car = 'car';
    case Bike = 'bike';
    case Truck = 'truck';
    case CNG = 'cng';
}

class ParameterizedVehicleFactory extends VehicleFactory
{
    public function __construct(
        private readonly VehicleType $type,
    ) {}

    public function createVehicle(): Vehicle
    {
        return match($this->type) {
            VehicleType::Car => new Car(),
            VehicleType::Bike => new Bike(),
            VehicleType::Truck => new Truck(),
            VehicleType::CNG => new Car(model: 'Bajaj RE CNG'),
        };
    }
}

// ব্যবহার
$factory = new ParameterizedVehicleFactory(VehicleType::Car);
echo $factory->deliverVehicle();
```

#### JavaScript ES2022+

```javascript
class ParameterizedVehicleFactory extends VehicleFactory {
  #type;

  static Types = Object.freeze({
    CAR: 'car',
    BIKE: 'bike',
    TRUCK: 'truck',
  });

  constructor(type) {
    super();
    this.#type = type;
  }

  createVehicle() {
    const creators = {
      [ParameterizedVehicleFactory.Types.CAR]: () => new Car(),
      [ParameterizedVehicleFactory.Types.BIKE]: () => new Bike(),
      [ParameterizedVehicleFactory.Types.TRUCK]: () => new Truck(),
    };

    const creator = creators[this.#type];
    if (!creator) {
      throw new Error(`অজানা vehicle type: ${this.#type}`);
    }

    return creator();
  }
}

// ব্যবহার
const factory = new ParameterizedVehicleFactory(
  ParameterizedVehicleFactory.Types.CAR
);
console.log(factory.deliverVehicle());
```

---

### Factory Method with Registration

Registration-based factory হলো সবচেয়ে **extensible** approach। নতুন product যোগ করতে existing factory code পরিবর্তন করতে হয় না — শুধু register করলেই হলো। এটি **Open/Closed Principle** পুরোপুরি মেনে চলে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

class VehicleRegistry
{
    /** @var array<string, callable(): Vehicle> */
    private static array $creators = [];

    public static function register(string $type, callable $creator): void
    {
        self::$creators[$type] = $creator;
    }

    public static function create(string $type): Vehicle
    {
        if (!isset(self::$creators[$type])) {
            throw new \InvalidArgumentException(
                "'{$type}' নামে কোনো vehicle registered নেই"
            );
        }

        return (self::$creators[$type])();
    }

    /** @return list<string> */
    public static function getRegisteredTypes(): array
    {
        return array_keys(self::$creators);
    }

    public static function has(string $type): bool
    {
        return isset(self::$creators[$type]);
    }
}

// Registration — bootstrap বা service provider-এ করতে পারেন
VehicleRegistry::register('car', fn() => new Car());
VehicleRegistry::register('bike', fn() => new Bike());
VehicleRegistry::register('truck', fn() => new Truck());
VehicleRegistry::register('electric-car', fn() => new Car(
    model: 'Tesla Model 3',
    maxSpeed: 225,
));

// ব্যবহার
$vehicle = VehicleRegistry::create('electric-car');
echo $vehicle->start(); // 🚗 Tesla Model 3 গাড়ি চালু হয়েছে

// Runtime-এ নতুন type register করা
VehicleRegistry::register('scooter', fn() => new Bike(
    model: 'Honda Scoopy',
    maxSpeed: 80,
));
```

#### JavaScript ES2022+

```javascript
class VehicleRegistry {
  static #creators = new Map();

  static register(type, creator) {
    if (typeof creator !== 'function') {
      throw new TypeError('Creator অবশ্যই একটি function হতে হবে');
    }
    VehicleRegistry.#creators.set(type, creator);
  }

  static create(type) {
    const creator = VehicleRegistry.#creators.get(type);
    if (!creator) {
      throw new Error(`'${type}' নামে কোনো vehicle registered নেই`);
    }
    return creator();
  }

  static get registeredTypes() {
    return [...VehicleRegistry.#creators.keys()];
  }

  static has(type) {
    return VehicleRegistry.#creators.has(type);
  }
}

// Registration
VehicleRegistry.register('car', () => new Car());
VehicleRegistry.register('bike', () => new Bike());
VehicleRegistry.register('truck', () => new Truck());
VehicleRegistry.register('electric-car', () => new Car('Tesla Model 3', 225));

// ব্যবহার
const vehicle = VehicleRegistry.create('electric-car');
console.log(vehicle.start()); // 🚗 Tesla Model 3 গাড়ি চালু হয়েছে

// Plugin system — third-party code-ও register করতে পারে
VehicleRegistry.register('scooter', () => new Bike('Honda Scoopy', 80));
```

---

## 🌍 Real-World Applicable Areas

### 1. 💳 Payment Gateway Factory (বাংলাদেশ কনটেক্সট)

বাংলাদেশের e-commerce-এ সবচেয়ে common use case। bKash, Nagad, SSLCommerz, Stripe — প্রতিটি gateway-এর payment process আলাদা, কিন্তু interface একটাই।

#### PHP 8.3 — Full Working Example

```php
<?php

declare(strict_types=1);

enum PaymentStatus: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Failed = 'failed';
    case Refunded = 'refunded';
}

readonly class PaymentResult
{
    public function __construct(
        public PaymentStatus $status,
        public string $transactionId,
        public float $amount,
        public string $gateway,
        public ?string $message = null,
    ) {}
}

/**
 * Product Interface — Payment Gateway Contract
 */
interface PaymentGateway
{
    public function charge(float $amount, string $currency = 'BDT'): PaymentResult;
    public function refund(string $transactionId, float $amount): PaymentResult;
    public function verify(string $transactionId): PaymentStatus;
    public function getGatewayName(): string;
    public function getSupportedCurrencies(): array;
}

/**
 * ConcreteProduct — bKash Payment
 */
readonly class BkashPayment implements PaymentGateway
{
    public function __construct(
        private string $appKey,
        private string $appSecret,
        private string $username,
        private string $password,
        private bool $sandbox = false,
    ) {}

    public function charge(float $amount, string $currency = 'BDT'): PaymentResult
    {
        // bKash Tokenized Payment API call
        $baseUrl = $this->sandbox
            ? 'https://tokenized.sandbox.bka.sh/v1.2.0-beta'
            : 'https://tokenized.pay.bka.sh/v1.2.0-beta';

        // Token generation → Create Payment → Execute Payment
        $transactionId = 'BK' . bin2hex(random_bytes(8));

        return new PaymentResult(
            status: PaymentStatus::Completed,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
            message: 'bKash পেমেন্ট সফল হয়েছে',
        );
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        return new PaymentResult(
            status: PaymentStatus::Refunded,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
            message: 'bKash রিফান্ড প্রক্রিয়াধীন',
        );
    }

    public function verify(string $transactionId): PaymentStatus
    {
        return PaymentStatus::Completed;
    }

    public function getGatewayName(): string
    {
        return 'bKash';
    }

    public function getSupportedCurrencies(): array
    {
        return ['BDT'];
    }
}

/**
 * ConcreteProduct — Nagad Payment
 */
readonly class NagadPayment implements PaymentGateway
{
    public function __construct(
        private string $merchantId,
        private string $publicKey,
        private string $privateKey,
        private bool $sandbox = false,
    ) {}

    public function charge(float $amount, string $currency = 'BDT'): PaymentResult
    {
        $transactionId = 'NGD' . bin2hex(random_bytes(8));

        return new PaymentResult(
            status: PaymentStatus::Completed,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
            message: 'নগদ পেমেন্ট সফল হয়েছে',
        );
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        return new PaymentResult(
            status: PaymentStatus::Refunded,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
        );
    }

    public function verify(string $transactionId): PaymentStatus
    {
        return PaymentStatus::Completed;
    }

    public function getGatewayName(): string
    {
        return 'Nagad';
    }

    public function getSupportedCurrencies(): array
    {
        return ['BDT'];
    }
}

/**
 * ConcreteProduct — SSLCommerz Payment
 */
readonly class SslCommerzPayment implements PaymentGateway
{
    public function __construct(
        private string $storeId,
        private string $storePassword,
        private bool $sandbox = false,
    ) {}

    public function charge(float $amount, string $currency = 'BDT'): PaymentResult
    {
        $transactionId = 'SSL' . bin2hex(random_bytes(8));

        return new PaymentResult(
            status: PaymentStatus::Processing,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
            message: 'SSLCommerz redirect URL তৈরি হয়েছে',
        );
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        return new PaymentResult(
            status: PaymentStatus::Refunded,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
        );
    }

    public function verify(string $transactionId): PaymentStatus
    {
        return PaymentStatus::Completed;
    }

    public function getGatewayName(): string
    {
        return 'SSLCommerz';
    }

    public function getSupportedCurrencies(): array
    {
        return ['BDT', 'USD', 'EUR'];
    }
}

/**
 * ConcreteProduct — Stripe Payment (International)
 */
readonly class StripePayment implements PaymentGateway
{
    public function __construct(
        private string $secretKey,
        private string $publishableKey,
    ) {}

    public function charge(float $amount, string $currency = 'USD'): PaymentResult
    {
        $transactionId = 'ch_' . bin2hex(random_bytes(12));

        return new PaymentResult(
            status: PaymentStatus::Completed,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
            message: 'Stripe payment successful',
        );
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        return new PaymentResult(
            status: PaymentStatus::Refunded,
            transactionId: $transactionId,
            amount: $amount,
            gateway: $this->getGatewayName(),
        );
    }

    public function verify(string $transactionId): PaymentStatus
    {
        return PaymentStatus::Completed;
    }

    public function getGatewayName(): string
    {
        return 'Stripe';
    }

    public function getSupportedCurrencies(): array
    {
        return ['USD', 'EUR', 'GBP', 'BDT'];
    }
}

/**
 * Creator — Abstract PaymentProcessor
 */
abstract class PaymentProcessor
{
    abstract protected function createGateway(): PaymentGateway;

    public function processPayment(float $amount, string $currency = 'BDT'): PaymentResult
    {
        $gateway = $this->createGateway();

        if (!in_array($currency, $gateway->getSupportedCurrencies())) {
            return new PaymentResult(
                status: PaymentStatus::Failed,
                transactionId: '',
                amount: $amount,
                gateway: $gateway->getGatewayName(),
                message: "{$currency} এই gateway-তে সাপোর্টেড নয়",
            );
        }

        return $gateway->charge($amount, $currency);
    }

    public function processRefund(string $transactionId, float $amount): PaymentResult
    {
        return $this->createGateway()->refund($transactionId, $amount);
    }
}

/**
 * ConcreteCreator — BkashProcessor
 */
class BkashProcessor extends PaymentProcessor
{
    public function __construct(
        private readonly array $config,
    ) {}

    protected function createGateway(): PaymentGateway
    {
        return new BkashPayment(
            appKey: $this->config['app_key'],
            appSecret: $this->config['app_secret'],
            username: $this->config['username'],
            password: $this->config['password'],
            sandbox: $this->config['sandbox'] ?? false,
        );
    }
}

/**
 * ConcreteCreator — NagadProcessor
 */
class NagadProcessor extends PaymentProcessor
{
    public function __construct(
        private readonly array $config,
    ) {}

    protected function createGateway(): PaymentGateway
    {
        return new NagadPayment(
            merchantId: $this->config['merchant_id'],
            publicKey: $this->config['public_key'],
            privateKey: $this->config['private_key'],
            sandbox: $this->config['sandbox'] ?? false,
        );
    }
}

/**
 * ConcreteCreator — SslCommerzProcessor
 */
class SslCommerzProcessor extends PaymentProcessor
{
    public function __construct(
        private readonly array $config,
    ) {}

    protected function createGateway(): PaymentGateway
    {
        return new SslCommerzPayment(
            storeId: $this->config['store_id'],
            storePassword: $this->config['store_password'],
            sandbox: $this->config['sandbox'] ?? false,
        );
    }
}

// ============================================
// Client Code — ব্যবহার
// ============================================

$processor = new BkashProcessor([
    'app_key' => 'your_bkash_app_key',
    'app_secret' => 'your_bkash_app_secret',
    'username' => 'your_username',
    'password' => 'your_password',
    'sandbox' => true,
]);

$result = $processor->processPayment(500.00, 'BDT');
echo "{$result->gateway}: {$result->message} (TxnID: {$result->transactionId})\n";
```

#### JavaScript ES2022+ — Full Working Example

```javascript
/**
 * Payment Gateway — Factory Method Implementation
 */

class PaymentResult {
  #status;
  #transactionId;
  #amount;
  #gateway;
  #message;

  constructor({ status, transactionId, amount, gateway, message = null }) {
    this.#status = status;
    this.#transactionId = transactionId;
    this.#amount = amount;
    this.#gateway = gateway;
    this.#message = message;
  }

  get status() { return this.#status; }
  get transactionId() { return this.#transactionId; }
  get amount() { return this.#amount; }
  get gateway() { return this.#gateway; }
  get message() { return this.#message; }

  toJSON() {
    return {
      status: this.#status,
      transactionId: this.#transactionId,
      amount: this.#amount,
      gateway: this.#gateway,
      message: this.#message,
    };
  }
}

class BkashPayment {
  #config;

  constructor(config) {
    this.#config = config;
  }

  async charge(amount, currency = 'BDT') {
    const transactionId = `BK${crypto.randomUUID().slice(0, 16)}`;
    return new PaymentResult({
      status: 'completed',
      transactionId,
      amount,
      gateway: this.gatewayName,
      message: 'bKash পেমেন্ট সফল হয়েছে',
    });
  }

  async refund(transactionId, amount) {
    return new PaymentResult({
      status: 'refunded',
      transactionId,
      amount,
      gateway: this.gatewayName,
    });
  }

  get gatewayName() { return 'bKash'; }
  get supportedCurrencies() { return ['BDT']; }
}

class NagadPayment {
  #config;

  constructor(config) {
    this.#config = config;
  }

  async charge(amount, currency = 'BDT') {
    const transactionId = `NGD${crypto.randomUUID().slice(0, 16)}`;
    return new PaymentResult({
      status: 'completed',
      transactionId,
      amount,
      gateway: this.gatewayName,
      message: 'নগদ পেমেন্ট সফল হয়েছে',
    });
  }

  async refund(transactionId, amount) {
    return new PaymentResult({
      status: 'refunded',
      transactionId,
      amount,
      gateway: this.gatewayName,
    });
  }

  get gatewayName() { return 'Nagad'; }
  get supportedCurrencies() { return ['BDT']; }
}

/**
 * Creator — Abstract PaymentProcessor
 */
class PaymentProcessor {
  createGateway() {
    throw new Error('Subclass অবশ্যই createGateway() implement করবে');
  }

  async processPayment(amount, currency = 'BDT') {
    const gateway = this.createGateway();

    if (!gateway.supportedCurrencies.includes(currency)) {
      return new PaymentResult({
        status: 'failed',
        transactionId: '',
        amount,
        gateway: gateway.gatewayName,
        message: `${currency} এই gateway-তে সাপোর্টেড নয়`,
      });
    }

    return gateway.charge(amount, currency);
  }
}

class BkashProcessor extends PaymentProcessor {
  #config;

  constructor(config) {
    super();
    this.#config = config;
  }

  createGateway() {
    return new BkashPayment(this.#config);
  }
}

class NagadProcessor extends PaymentProcessor {
  #config;

  constructor(config) {
    super();
    this.#config = config;
  }

  createGateway() {
    return new NagadPayment(this.#config);
  }
}

// ব্যবহার
const processor = new BkashProcessor({
  appKey: 'your_app_key',
  appSecret: 'your_app_secret',
  sandbox: true,
});

const result = await processor.processPayment(500.00, 'BDT');
console.log(result.toJSON());
```

### 2. 📧 Notification Factory

```
NotificationFactory
├── EmailNotification    → SMTP/Mailgun/SES
├── SmsNotification      → Twilio/BulkSMS BD
├── PushNotification     → Firebase/OneSignal
└── SlackNotification    → Webhook API
```

### 3. 📄 Document Export Factory

```
DocumentExporter
├── PdfExporter          → DOMPDF/wkhtmltopdf
├── ExcelExporter        → PhpSpreadsheet/ExcelJS
├── CsvExporter          → Native CSV writers
└── JsonExporter         → Structured JSON output
```

### 4. 📝 Logger Factory

```
LoggerFactory
├── FileLogger           → Log file-এ write
├── DatabaseLogger       → DB table-এ insert
├── CloudLogger          → CloudWatch/Stackdriver
└── SlackLogger          → Critical alert Slack-এ
```

### 5. Framework-এ Factory Method

#### Laravel

```php
// Laravel-এ Factory Method সর্বত্র ব্যবহৃত হয়:

// Mail Driver — config/mail.php-এর driver অনুযায়ী
Mail::to($user)->send(new OrderShipped($order));
// পর্দার আড়ালে: MailManager->createSmtpDriver(), createSesDriver()

// Queue Driver
Queue::push(new ProcessPayment($order));
// QueueManager->createRedisDriver(), createSqsDriver()

// Cache Driver
Cache::put('key', 'value', 3600);
// CacheManager->createRedisDriver(), createFileDriver()

// Filesystem Driver
Storage::disk('s3')->put('file.jpg', $content);
// FilesystemManager->createS3Driver(), createLocalDriver()
```

#### Node.js — Database Driver Selection

```javascript
// Knex.js — database driver factory
import knex from 'knex';

// Configuration অনুযায়ী different driver তৈরি হয়
const db = knex({
  client: 'mysql2',        // এটাই factory method-এর parameter
  connection: {
    host: '127.0.0.1',
    user: 'root',
    password: '',
    database: 'mydb',
  },
});

// পর্দার আড়ালে knex client parameter দেখে
// MySQL2, PostgreSQL, বা SQLite driver instantiate করে
```

---

## 🔥 Advanced Deep Dive

### Factory Method in Frameworks — Laravel's Manager Pattern

Laravel-এর Manager pattern হলো Factory Method pattern-এর একটি **production-grade implementation**। `Illuminate\Support\Manager` class দেখলে বুঝবেন কিভাবে Factory Method বড় পরিসরে কাজ করে:

```php
<?php

// Laravel-এর Manager pattern-এর সরলীকৃত রূপ
// আসল source: Illuminate\Support\Manager

abstract class Manager
{
    protected Application $app;
    protected array $customCreators = [];
    protected array $drivers = [];

    abstract public function getDefaultDriver(): string;

    public function driver(?string $driver = null): mixed
    {
        $driver ??= $this->getDefaultDriver();

        // Singleton pattern — একবার তৈরি হলে cache করে
        if (!isset($this->drivers[$driver])) {
            $this->drivers[$driver] = $this->createDriver($driver);
        }

        return $this->drivers[$driver];
    }

    protected function createDriver(string $driver): mixed
    {
        // Custom creator আগে check করে (Registration pattern)
        if (isset($this->customCreators[$driver])) {
            return $this->callCustomCreator($driver);
        }

        // Convention-based: create{Driver}Driver() method খোঁজে
        $method = 'create' . ucfirst($driver) . 'Driver';

        if (method_exists($this, $method)) {
            return $this->$method();
        }

        throw new \InvalidArgumentException(
            "Driver [{$driver}] সাপোর্টেড নয়।"
        );
    }

    public function extend(string $driver, \Closure $callback): static
    {
        $this->customCreators[$driver] = $callback;
        return $this;
    }
}

// CacheManager — Manager pattern-এর বাস্তব উদাহরণ
class CacheManager extends Manager
{
    public function getDefaultDriver(): string
    {
        return $this->app['config']['cache.default']; // 'redis'
    }

    protected function createFileDriver(): FileStore
    {
        return new FileStore(
            $this->app['files'],
            $this->app['config']['cache.stores.file.path'],
        );
    }

    protected function createRedisDriver(): RedisStore
    {
        return new RedisStore(
            $this->app['redis'],
            $this->app['config']['cache.stores.redis.connection'],
        );
    }

    protected function createMemcachedDriver(): MemcachedStore
    {
        return new MemcachedStore(
            $this->getMemcachedConnection(),
        );
    }
}

// Runtime-এ নতুন driver register করা
$cacheManager->extend('dynamodb', function (Application $app) {
    return new DynamoDbStore(
        $app->make(DynamoDbClient::class),
        $app['config']['cache.stores.dynamodb.table'],
    );
});
```

**Laravel Manager Pattern-এর বৈশিষ্ট্য:**

1. **Convention over Configuration**: `create{Name}Driver()` naming convention ব্যবহার করে
2. **Custom Creator Registration**: `extend()` method দিয়ে runtime-এ নতুন driver যোগ
3. **Driver Caching**: একবার তৈরি হওয়া driver পুনরায় তৈরি হয় না (Singleton)
4. **Default Driver**: config থেকে default driver determine করে

---

### Factory Method vs Simple Factory vs Abstract Factory

| বৈশিষ্ট্য | Simple Factory | Factory Method | Abstract Factory |
|-----------|----------------|----------------|------------------|
| **Pattern Type** | Idiom (pattern নয়) | Creational Pattern | Creational Pattern |
| **Object Creation** | Single method-এ conditional | Subclass-এ override | Factory-র factory |
| **Extensibility** | নতুন type = method modify | নতুন type = নতুন subclass | নতুন family = নতুন factory |
| **OCP Compliance** | ❌ ভঙ্গ করে | ✅ মেনে চলে | ✅ মেনে চলে |
| **Complexity** | ⭐ সহজ | ⭐⭐ মাঝারি | ⭐⭐⭐ জটিল |
| **Use Case** | ছোট project, limited types | একটি product family | একাধিক সম্পর্কিত product |
| **Inheritance** | ❌ ব্যবহার করে না | ✅ ব্যবহার করে | ✅ ব্যবহার করে |
| **Single Product?** | হ্যাঁ | হ্যাঁ | না, product family |

#### তুলনামূলক কোড (PHP 8.3)

```php
<?php

// ========================
// ❶ Simple Factory — শুধু conditional logic
// ========================
class SimpleNotificationFactory
{
    public static function create(string $type): Notification
    {
        return match($type) {
            'email' => new EmailNotification(),
            'sms' => new SmsNotification(),
            'push' => new PushNotification(),
            // ⚠️ নতুন type যোগ করতে হলে এই method modify করতে হবে
            default => throw new \InvalidArgumentException("অজানা type: {$type}"),
        };
    }
}

// ========================
// ❷ Factory Method — subclass decides
// ========================
abstract class NotificationSender
{
    abstract protected function createNotification(): Notification;

    public function send(string $message, string $recipient): bool
    {
        $notification = $this->createNotification();
        $notification->setMessage($message);
        $notification->setRecipient($recipient);
        return $notification->deliver();
    }
}

class EmailNotificationSender extends NotificationSender
{
    protected function createNotification(): Notification
    {
        return new EmailNotification();
    }
}

// ✅ নতুন type = নতুন class, existing code untouched
class SlackNotificationSender extends NotificationSender
{
    protected function createNotification(): Notification
    {
        return new SlackNotification();
    }
}

// ========================
// ❸ Abstract Factory — product family
// ========================
interface UIFactory
{
    public function createButton(): Button;
    public function createCheckbox(): Checkbox;
    public function createTextInput(): TextInput;
}

class MaterialUIFactory implements UIFactory
{
    public function createButton(): Button
    {
        return new MaterialButton();
    }

    public function createCheckbox(): Checkbox
    {
        return new MaterialCheckbox();
    }

    public function createTextInput(): TextInput
    {
        return new MaterialTextInput();
    }
}

class BootstrapUIFactory implements UIFactory
{
    public function createButton(): Button
    {
        return new BootstrapButton();
    }

    public function createCheckbox(): Checkbox
    {
        return new BootstrapCheckbox();
    }

    public function createTextInput(): TextInput
    {
        return new BootstrapTextInput();
    }
}
```

---

### Factory Method with Strategy Pattern

Factory Method প্রায়ই Strategy pattern-এর সাথে একত্রে ব্যবহৃত হয়। Factory Method **object তৈরি** করে, Strategy সেই object-এর **behavior** নির্ধারণ করে।

```php
<?php

declare(strict_types=1);

// Strategy Interface
interface PricingStrategy
{
    public function calculatePrice(float $basePrice, int $quantity): float;
    public function getName(): string;
}

readonly class RegularPricing implements PricingStrategy
{
    public function calculatePrice(float $basePrice, int $quantity): float
    {
        return $basePrice * $quantity;
    }

    public function getName(): string { return 'Regular'; }
}

readonly class PremiumPricing implements PricingStrategy
{
    public function __construct(
        private float $discountPercent = 15.0,
    ) {}

    public function calculatePrice(float $basePrice, int $quantity): float
    {
        $total = $basePrice * $quantity;
        return $total - ($total * $this->discountPercent / 100);
    }

    public function getName(): string { return 'Premium'; }
}

readonly class WholesalePricing implements PricingStrategy
{
    public function calculatePrice(float $basePrice, int $quantity): float
    {
        $discount = match(true) {
            $quantity >= 100 => 30.0,
            $quantity >= 50 => 20.0,
            $quantity >= 20 => 10.0,
            default => 0.0,
        };

        $total = $basePrice * $quantity;
        return $total - ($total * $discount / 100);
    }

    public function getName(): string { return 'Wholesale'; }
}

// Factory Method — Strategy তৈরির জন্য
abstract class PricingStrategyFactory
{
    abstract public function createStrategy(): PricingStrategy;

    public function calculateOrderTotal(float $basePrice, int $quantity): array
    {
        $strategy = $this->createStrategy();
        $total = $strategy->calculatePrice($basePrice, $quantity);

        return [
            'strategy' => $strategy->getName(),
            'base_price' => $basePrice,
            'quantity' => $quantity,
            'total' => $total,
        ];
    }
}

class RegularPricingFactory extends PricingStrategyFactory
{
    public function createStrategy(): PricingStrategy
    {
        return new RegularPricing();
    }
}

class PremiumPricingFactory extends PricingStrategyFactory
{
    public function __construct(
        private readonly float $discountPercent = 15.0,
    ) {}

    public function createStrategy(): PricingStrategy
    {
        return new PremiumPricing($this->discountPercent);
    }
}
```

---

### Factory with Dependency Injection

Production-grade application-এ Factory Method সবসময় **Dependency Injection Container**-এর সাথে কাজ করে। এটি testability এবং flexibility বাড়ায়।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface Logger
{
    public function log(string $level, string $message, array $context = []): void;
}

readonly class FileLogger implements Logger
{
    public function __construct(
        private string $logPath,
        private string $dateFormat = 'Y-m-d H:i:s',
    ) {}

    public function log(string $level, string $message, array $context = []): void
    {
        $timestamp = date($this->dateFormat);
        $contextStr = $context ? json_encode($context) : '';
        $line = "[{$timestamp}] {$level}: {$message} {$contextStr}\n";
        file_put_contents($this->logPath, $line, FILE_APPEND | LOCK_EX);
    }
}

readonly class DatabaseLogger implements Logger
{
    public function __construct(
        private \PDO $pdo,
        private string $tableName = 'logs',
    ) {}

    public function log(string $level, string $message, array $context = []): void
    {
        $stmt = $this->pdo->prepare(
            "INSERT INTO {$this->tableName} (level, message, context, created_at)
             VALUES (:level, :message, :context, NOW())"
        );

        $stmt->execute([
            'level' => $level,
            'message' => $message,
            'context' => json_encode($context),
        ]);
    }
}

// Factory with DI
abstract class LoggerFactory
{
    abstract public function createLogger(): Logger;
}

class FileLoggerFactory extends LoggerFactory
{
    public function __construct(
        private readonly string $logPath,
    ) {}

    public function createLogger(): Logger
    {
        return new FileLogger(logPath: $this->logPath);
    }
}

class DatabaseLoggerFactory extends LoggerFactory
{
    public function __construct(
        private readonly \PDO $pdo,
    ) {}

    public function createLogger(): Logger
    {
        return new DatabaseLogger(pdo: $this->pdo);
    }
}

// Laravel Service Provider-এ register করা
class LoggerServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(LoggerFactory::class, function (Application $app) {
            $driver = config('logging.default');

            return match($driver) {
                'file' => new FileLoggerFactory(
                    logPath: storage_path('logs/app.log'),
                ),
                'database' => new DatabaseLoggerFactory(
                    pdo: $app->make(\PDO::class),
                ),
                default => throw new \RuntimeException(
                    "অজানা log driver: {$driver}"
                ),
            };
        });
    }
}
```

#### JavaScript ES2022+

```javascript
class FileLogger {
  #logPath;

  constructor(logPath) {
    this.#logPath = logPath;
  }

  async log(level, message, context = {}) {
    const { appendFile } = await import('node:fs/promises');
    const timestamp = new Date().toISOString();
    const contextStr = Object.keys(context).length
      ? JSON.stringify(context)
      : '';
    const line = `[${timestamp}] ${level}: ${message} ${contextStr}\n`;
    await appendFile(this.#logPath, line);
  }
}

class ConsoleLogger {
  async log(level, message, context = {}) {
    const methods = { error: 'error', warn: 'warn', info: 'info', debug: 'debug' };
    const method = methods[level] ?? 'log';
    console[method](`[${level.toUpperCase()}] ${message}`, context);
  }
}

// Factory with Dependency Injection
class LoggerFactory {
  createLogger() {
    throw new Error('Subclass অবশ্যই createLogger() implement করবে');
  }
}

class FileLoggerFactory extends LoggerFactory {
  #logPath;

  constructor(logPath) {
    super();
    this.#logPath = logPath;
  }

  createLogger() {
    return new FileLogger(this.#logPath);
  }
}

class ConsoleLoggerFactory extends LoggerFactory {
  createLogger() {
    return new ConsoleLogger();
  }
}

// DI Container integration (e.g., with tsyringe or inversify)
function createLoggerFactory(config) {
  const factories = {
    file: () => new FileLoggerFactory(config.logPath),
    console: () => new ConsoleLoggerFactory(),
  };

  const factory = factories[config.driver];
  if (!factory) throw new Error(`অজানা log driver: ${config.driver}`);
  return factory();
}
```

---

### Dynamic Factory (Runtime Registration)

Dynamic Factory pattern-এ নতুন product type **runtime-এ** register করা যায়। এটি **plugin system** তৈরি করতে অত্যন্ত কার্যকর।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface ExportHandler
{
    public function export(array $data): string;
    public function getMimeType(): string;
    public function getFileExtension(): string;
}

readonly class CsvExportHandler implements ExportHandler
{
    public function export(array $data): string
    {
        if (empty($data)) return '';

        $output = fopen('php://memory', 'r+');
        fputcsv($output, array_keys($data[0]));

        foreach ($data as $row) {
            fputcsv($output, $row);
        }

        rewind($output);
        $csv = stream_get_contents($output);
        fclose($output);

        return $csv;
    }

    public function getMimeType(): string { return 'text/csv'; }
    public function getFileExtension(): string { return 'csv'; }
}

readonly class JsonExportHandler implements ExportHandler
{
    public function __construct(
        private bool $prettyPrint = true,
    ) {}

    public function export(array $data): string
    {
        $flags = JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE;
        if ($this->prettyPrint) $flags |= JSON_PRETTY_PRINT;

        return json_encode($data, $flags);
    }

    public function getMimeType(): string { return 'application/json'; }
    public function getFileExtension(): string { return 'json'; }
}

/**
 * Dynamic Factory — runtime-এ handler register/unregister
 */
final class ExportFactory
{
    /** @var array<string, callable(): ExportHandler> */
    private array $handlers = [];

    private static ?self $instance = null;

    private function __construct() {}

    public static function getInstance(): self
    {
        return self::$instance ??= new self();
    }

    public function register(string $format, callable $creator): self
    {
        $this->handlers[strtolower($format)] = $creator;
        return $this;
    }

    public function unregister(string $format): self
    {
        unset($this->handlers[strtolower($format)]);
        return $this;
    }

    public function create(string $format): ExportHandler
    {
        $format = strtolower($format);

        if (!isset($this->handlers[$format])) {
            $available = implode(', ', array_keys($this->handlers));
            throw new \InvalidArgumentException(
                "'{$format}' format সাপোর্টেড নয়। উপলব্ধ formats: {$available}"
            );
        }

        return ($this->handlers[$format])();
    }

    /** @return list<string> */
    public function getAvailableFormats(): array
    {
        return array_keys($this->handlers);
    }
}

// Bootstrap-এ register করুন
$factory = ExportFactory::getInstance();
$factory->register('csv', fn() => new CsvExportHandler());
$factory->register('json', fn() => new JsonExportHandler());

// Plugin থেকে নতুন format register
$factory->register('xml', fn() => new XmlExportHandler());

// ব্যবহার
$handler = $factory->create('csv');
$csvOutput = $handler->export([
    ['নাম' => 'রহিম', 'বয়স' => 25],
    ['নাম' => 'করিম', 'বয়স' => 30],
]);
```

---

## ✅ সুবিধা

| # | সুবিধা | বিস্তারিত |
|---|--------|----------|
| 1 | **Open/Closed Principle** | নতুন product type যোগ করতে existing code modify করতে হয় না |
| 2 | **Single Responsibility** | Product creation logic আলাদা class-এ থাকে |
| 3 | **Loose Coupling** | Client code concrete class সম্পর্কে জানে না, শুধু interface জানে |
| 4 | **Testability** | Mock factory inject করে সহজে unit test করা যায় |
| 5 | **Parallel Development** | বিভিন্ন developer আলাদা ConcreteCreator-এ কাজ করতে পারে |
| 6 | **Framework Integration** | Laravel, Symfony, NestJS সবাই এই pattern ব্যবহার করে |
| 7 | **Runtime Flexibility** | Configuration অনুযায়ী runtime-এ product type পরিবর্তন |

## ❌ অসুবিধা

| # | অসুবিধা | বিস্তারিত |
|---|---------|----------|
| 1 | **Class Explosion** | প্রতিটি product-এর জন্য আলাদা Creator class লাগে |
| 2 | **Complexity** | ছোট project-এ অতিরিক্ত abstraction unnecessary হতে পারে |
| 3 | **Indirection** | Code follow করা কঠিন হয়ে যায়, IDE-তে "Go to Definition" complexity বাড়ে |
| 4 | **Learning Curve** | Junior developer-দের জন্য বুঝতে সময় লাগে |
| 5 | **Over-engineering Risk** | শুধু ২-৩টি type থাকলে simple factory যথেষ্ট |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ❌ ভুল ১: Factory-তে Business Logic রাখা

```php
// ❌ WRONG — Factory-তে business logic
class BadPaymentFactory
{
    public function createAndProcess(string $type, float $amount): PaymentResult
    {
        $gateway = $this->create($type);
        // ❌ Factory-র কাজ শুধু create করা, process করা নয়
        $result = $gateway->charge($amount);
        $this->sendNotification($result);
        $this->updateDatabase($result);
        return $result;
    }
}

// ✅ CORRECT — Factory শুধু create করে
class GoodPaymentFactory
{
    public function create(string $type): PaymentGateway
    {
        return match($type) {
            'bkash' => new BkashPayment($this->config),
            'nagad' => new NagadPayment($this->config),
            default => throw new \InvalidArgumentException("অজানা type"),
        };
    }
}
```

### ❌ ভুল ২: God Factory — একটি Factory সব কিছু তৈরি করে

```php
// ❌ WRONG — God Factory
class UniversalFactory
{
    public function create(string $type): object
    {
        return match($type) {
            'user' => new User(),
            'payment' => new Payment(),
            'order' => new Order(),
            'notification' => new Notification(),
            // সব কিছু একটাই factory-তে
        };
    }
}

// ✅ CORRECT — প্রতিটি domain-এর আলাদা factory
class PaymentGatewayFactory { /* শুধু payment গুলো */ }
class NotificationFactory { /* শুধু notification গুলো */ }
```

### ❌ ভুল ৩: Factory Method-এ Concrete Class Return Type

```php
// ❌ WRONG — concrete return type
abstract class BadFactory
{
    abstract public function create(): BkashPayment; // ❌ concrete type
}

// ✅ CORRECT — interface return type
abstract class GoodFactory
{
    abstract public function create(): PaymentGateway; // ✅ interface
}
```

### ❌ ভুল ৪: অপ্রয়োজনীয় Factory Method ব্যবহার

```php
// ❌ WRONG — যদি শুধু একটাই concrete implementation থাকে
// এবং ভবিষ্যতেও বাড়ার সম্ভাবনা না থাকে, তাহলে Factory Method overkill

// ✅ যখন Factory Method ব্যবহার করবেন:
// - ২+ concrete implementation আছে বা হবে
// - Runtime-এ implementation পরিবর্তন দরকার
// - Testing-এ mock/stub দরকার
```

---

## 🧪 টেস্টিং

### PHPUnit Testing

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class VehicleFactoryTest extends TestCase
{
    #[Test]
    public function carFactoryCreatesCarInstance(): void
    {
        $factory = new CarFactory('Honda Civic');
        $vehicle = $factory->createVehicle();

        $this->assertInstanceOf(Car::class, $vehicle);
        $this->assertInstanceOf(Vehicle::class, $vehicle);
    }

    #[Test]
    public function bikeFactoryCreatesBikeInstance(): void
    {
        $factory = new BikeFactory('Yamaha R15');
        $vehicle = $factory->createVehicle();

        $this->assertInstanceOf(Bike::class, $vehicle);
        $this->assertEquals(FuelType::Petrol, $vehicle->getFuelType());
    }

    #[Test]
    public function deliverVehicleReturnsExpectedOutput(): void
    {
        $factory = new TruckFactory('Tata LPT');
        $output = $factory->deliverVehicle();

        $this->assertStringContainsString('ট্রাক চালু হয়েছে', $output);
        $this->assertStringContainsString('ডিজেল', $output);
        $this->assertStringContainsString('ডেলিভারি হয়েছে', $output);
    }

    #[Test]
    #[DataProvider('factoryProvider')]
    public function allFactoriesReturnVehicleInterface(
        VehicleFactory $factory,
        string $expectedClass,
    ): void {
        $vehicle = $factory->createVehicle();

        $this->assertInstanceOf(Vehicle::class, $vehicle);
        $this->assertInstanceOf($expectedClass, $vehicle);
    }

    public static function factoryProvider(): array
    {
        return [
            'car factory' => [new CarFactory(), Car::class],
            'bike factory' => [new BikeFactory(), Bike::class],
            'truck factory' => [new TruckFactory(), Truck::class],
        ];
    }

    #[Test]
    public function registryThrowsExceptionForUnknownType(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('registered নেই');

        VehicleRegistry::create('spaceship');
    }

    #[Test]
    public function registryCanRegisterAndCreateNewType(): void
    {
        VehicleRegistry::register('test-vehicle', fn() => new Car('Test Car'));

        $vehicle = VehicleRegistry::create('test-vehicle');

        $this->assertInstanceOf(Vehicle::class, $vehicle);
    }
}

// Payment Gateway Factory Test
class PaymentProcessorTest extends TestCase
{
    #[Test]
    public function bkashProcessorCreatesCorrectGateway(): void
    {
        $processor = new BkashProcessor([
            'app_key' => 'test_key',
            'app_secret' => 'test_secret',
            'username' => 'test_user',
            'password' => 'test_pass',
            'sandbox' => true,
        ]);

        $result = $processor->processPayment(100.00);

        $this->assertEquals('bKash', $result->gateway);
        $this->assertEquals(PaymentStatus::Completed, $result->status);
        $this->assertEquals(100.00, $result->amount);
    }

    #[Test]
    public function unsupportedCurrencyReturnsFailed(): void
    {
        $processor = new BkashProcessor([
            'app_key' => 'test_key',
            'app_secret' => 'test_secret',
            'username' => 'test_user',
            'password' => 'test_pass',
        ]);

        $result = $processor->processPayment(100.00, 'JPY');

        $this->assertEquals(PaymentStatus::Failed, $result->status);
        $this->assertStringContainsString('সাপোর্টেড নয়', $result->message);
    }

    #[Test]
    public function factoryMethodCanBeMocked(): void
    {
        $mockGateway = $this->createMock(PaymentGateway::class);
        $mockGateway->method('getSupportedCurrencies')->willReturn(['BDT']);
        $mockGateway->method('charge')->willReturn(new PaymentResult(
            status: PaymentStatus::Completed,
            transactionId: 'MOCK_123',
            amount: 500.00,
            gateway: 'MockGateway',
        ));

        // Anonymous class দিয়ে factory mock করা
        $processor = new class($mockGateway) extends PaymentProcessor {
            public function __construct(
                private readonly PaymentGateway $gateway,
            ) {}

            protected function createGateway(): PaymentGateway
            {
                return $this->gateway;
            }
        };

        $result = $processor->processPayment(500.00);
        $this->assertEquals('MOCK_123', $result->transactionId);
    }
}
```

### Jest Testing (JavaScript)

```javascript
import { describe, it, expect, vi } from 'vitest'; // or jest

describe('VehicleFactory', () => {
  it('CarFactory should create Car instance', () => {
    const factory = new CarFactory('Honda Civic');
    const vehicle = factory.createVehicle();

    expect(vehicle).toBeInstanceOf(Car);
    expect(vehicle.fuelType).toBe(FuelType.Petrol);
  });

  it('deliverVehicle should return formatted string', () => {
    const factory = new TruckFactory('Tata LPT');
    const output = factory.deliverVehicle();

    expect(output).toContain('ট্রাক চালু হয়েছে');
    expect(output).toContain('ডিজেল');
    expect(output).toContain('ডেলিভারি হয়েছে');
  });

  it('base VehicleFactory should throw on createVehicle', () => {
    const factory = new VehicleFactory();

    expect(() => factory.createVehicle()).toThrow(
      'Subclass অবশ্যই createVehicle() implement করবে'
    );
  });
});

describe('VehicleRegistry', () => {
  it('should register and create vehicles', () => {
    VehicleRegistry.register('test-car', () => new Car('Test Model'));

    const vehicle = VehicleRegistry.create('test-car');
    expect(vehicle).toBeInstanceOf(Car);
  });

  it('should throw for unregistered type', () => {
    expect(() => VehicleRegistry.create('spaceship')).toThrow(
      'registered নেই'
    );
  });

  it('should list all registered types', () => {
    VehicleRegistry.register('type-a', () => new Car());
    VehicleRegistry.register('type-b', () => new Bike());

    const types = VehicleRegistry.registeredTypes;
    expect(types).toContain('type-a');
    expect(types).toContain('type-b');
  });
});

describe('PaymentProcessor', () => {
  it('BkashProcessor should process payment correctly', async () => {
    const processor = new BkashProcessor({
      appKey: 'test',
      appSecret: 'test',
      sandbox: true,
    });

    const result = await processor.processPayment(500.0, 'BDT');

    expect(result.gateway).toBe('bKash');
    expect(result.status).toBe('completed');
    expect(result.amount).toBe(500.0);
  });

  it('should fail for unsupported currency', async () => {
    const processor = new BkashProcessor({
      appKey: 'test',
      appSecret: 'test',
    });

    const result = await processor.processPayment(100, 'JPY');

    expect(result.status).toBe('failed');
    expect(result.message).toContain('সাপোর্টেড নয়');
  });

  it('factory method can be mocked', async () => {
    const mockGateway = {
      gatewayName: 'MockGateway',
      supportedCurrencies: ['BDT'],
      charge: vi.fn().mockResolvedValue(
        new PaymentResult({
          status: 'completed',
          transactionId: 'MOCK_123',
          amount: 300,
          gateway: 'MockGateway',
        })
      ),
    };

    class MockProcessor extends PaymentProcessor {
      createGateway() {
        return mockGateway;
      }
    }

    const processor = new MockProcessor();
    const result = await processor.processPayment(300, 'BDT');

    expect(result.transactionId).toBe('MOCK_123');
    expect(mockGateway.charge).toHaveBeenCalledWith(300, 'BDT');
  });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Factory Method ↔ Abstract Factory

**Abstract Factory** প্রায়ই **Factory Method** দিয়ে implement করা হয়। Abstract Factory-র প্রতিটি creation method আসলে একটি Factory Method।

```
Abstract Factory = একাধিক Factory Method একসাথে
Factory Method = একটি product তৈরি
Abstract Factory = সম্পর্কিত product family তৈরি
```

### Factory Method ↔ Template Method

Factory Method হলো **Template Method-এর specialization**। Template Method-এ একটি algorithm-এর skeleton define করা হয় যেখানে কিছু step subclass-এ defer করা হয়। Factory Method-এ সেই deferred step হলো **object creation**।

```php
// Template Method
abstract class DataProcessor
{
    // Template Method
    public function process(): void
    {
        $data = $this->readData();        // step 1
        $parsed = $this->parseData($data); // step 2 (deferred)
        $this->saveData($parsed);          // step 3
    }

    abstract protected function parseData(string $data): array; // factory-like
}

// Factory Method — template method-এর specialized version
abstract class ReportGenerator
{
    abstract protected function createFormatter(): Formatter; // factory method

    public function generate(array $data): string
    {
        $formatter = $this->createFormatter(); // creation step deferred
        return $formatter->format($data);
    }
}
```

### Factory Method ↔ Prototype

**Prototype** pattern-এ নতুন object তৈরি হয় existing object clone করে। Factory Method-এর বিকল্প হিসেবে ব্যবহার করা যায় যখন object creation expensive হয়।

```php
// Factory Method approach
$vehicle = $factory->createVehicle(); // নতুন instance তৈরি

// Prototype approach
$vehicle = $prototypeVehicle->clone(); // existing instance clone
```

### Factory Method ↔ Strategy

Strategy pattern behavior encapsulate করে, Factory Method সেই strategy object তৈরি করে। দুটি একসাথে ব্যবহার করলে **runtime-এ behavior পরিবর্তন** সহজ হয়।

```
User Type → PricingStrategyFactory → PricingStrategy → calculatePrice()
  Regular → RegularPricingFactory → RegularPricing → basePrice × qty
  Premium → PremiumPricingFactory → PremiumPricing → (basePrice × qty) - discount
```

### সম্পর্ক সারণি

| প্যাটার্ন | সম্পর্ক | বিস্তারিত |
|-----------|---------|----------|
| **Abstract Factory** | বড় ভাই | Factory Method-কে internally ব্যবহার করে |
| **Template Method** | পিতৃ প্যাটার্ন | Factory Method এর specialization |
| **Prototype** | বিকল্প | Clone দিয়ে creation, subclass দরকার নেই |
| **Strategy** | সঙ্গী | Factory Method strategy object তৈরি করে |
| **Singleton** | সহকারী | Factory নিজে Singleton হতে পারে |
| **Builder** | ভিন্ন উদ্দেশ্য | Complex object step-by-step তৈরি |

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

1. **Compile time-এ exact class জানা নেই** — কোন concrete class তৈরি হবে সেটা runtime-এ ঠিক হবে
2. **Framework/Library তৈরি করছেন** — user-কে extensibility দিতে চান
3. **Multiple implementations আছে** — একই interface-এর ২+ concrete class আছে
4. **Configuration-based creation** — config file/env variable অনুযায়ী object তৈরি
5. **Testing-এ mock দরকার** — factory mock করে concrete dependency এড়ানো
6. **Third-party integration** — বিভিন্ন payment gateway, email provider ইত্যাদি
7. **Plugin architecture** — runtime-এ নতুন type register করার সুবিধা চান

### ❌ ব্যবহার করবেন না যখন:

1. **শুধু একটি implementation** — এবং ভবিষ্যতেও বাড়ার সম্ভাবনা নেই
2. **Simple object creation** — constructor-এ ২-৩টি parameter, কোনো complexity নেই
3. **Performance critical path** — প্রতিটি microsecond গুরুত্বপূর্ণ (allocation overhead)
4. **Small script/utility** — ১০০-২০০ লাইনের ছোট script-এ overkill
5. **No polymorphism needed** — সব জায়গায় একই concrete class ব্যবহার হয়

### সিদ্ধান্ত নেওয়ার Flowchart

```
শুরু
  ↓
একাধিক concrete implementation আছে?
  ├── না → Simple instantiation ব্যবহার করুন
  ├── হ্যাঁ ↓
  │
  Runtime-এ type পরিবর্তন হবে?
  ├── না → Static method বা Simple Factory যথেষ্ট
  ├── হ্যাঁ ↓
  │
  সম্পর্কিত product family আছে?
  ├── হ্যাঁ → Abstract Factory ব্যবহার করুন
  ├── না ↓
  │
  ✅ Factory Method ব্যবহার করুন!
```

---

## 📋 সারসংক্ষেপ

### মূল বিষয়গুলো মনে রাখুন

```
🏭 Factory Method = "Object creation-এর জন্য interface, subclass ঠিক করবে কোনটা তৈরি হবে"

📐 চারটি অংশ:
   Product ←→ ConcreteProduct
   Creator ←→ ConcreteCreator

🎯 মূল নীতি:
   ✅ OCP — নতুন type = নতুন class, existing code untouched
   ✅ SRP — Creation logic আলাদা class-এ
   ✅ DIP — Abstraction-এর উপর নির্ভরতা, concrete-এর উপর নয়

🔧 তিনটি variation:
   1. Classic Factory Method (subclass override)
   2. Parameterized Factory Method (parameter-based)
   3. Registration-based Factory (runtime extensible)

🇧🇩 বাংলাদেশ context:
   - bKash/Nagad/SSLCommerz payment integration
   - পাঠাও/RedX/Paperfly logistics
   - বিভিন্ন SMS gateway (BulkSMS BD, Twilio)
```

### Quick Reference

| প্রশ্ন | উত্তর |
|--------|-------|
| **কী সমস্যা সমাধান করে?** | Object creation logic কে usage থেকে আলাদা করা |
| **কোন SOLID principle?** | Open/Closed, Single Responsibility, Dependency Inversion |
| **GoF Category?** | Creational |
| **কখন ব্যবহার?** | Multiple implementations, runtime selection, framework building |
| **কখন এড়াবেন?** | Single implementation, simple creation, small scripts |
| **Laravel-এ কোথায়?** | Manager classes (Cache, Queue, Mail, Session, Filesystem) |
| **Node.js-এ কোথায়?** | Database drivers, HTTP clients, Logger factories |
| **সবচেয়ে ভালো সঙ্গী?** | Strategy, Template Method, Dependency Injection |

---

> **"Program to an interface, not an implementation."**
> — Gang of Four
>
> Factory Method এই নীতির সবচেয়ে সুন্দর বাস্তবায়ন। আপনার কোডকে concrete class থেকে মুক্ত করুন — flexibility এবং testability অনেক বাড়বে।

---

*এই ডকুমেন্টটি senior engineer-দের জন্য তৈরি। Factory Method pattern-এর মূল ধারণা, বাস্তব প্রয়োগ, framework integration, testing strategy এবং সম্পর্কিত pattern সবকিছু এখানে আলোচনা করা হয়েছে। বাংলাদেশের software industry-র context-এ payment gateway, logistics, এবং notification system-এর উদাহরণ দেওয়া হয়েছে।*
