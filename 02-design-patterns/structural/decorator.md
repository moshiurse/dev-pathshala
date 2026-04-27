# 🎀 Decorator প্যাটার্ন — Complete Deep Dive

## 📌 সংজ্ঞা ও পরিচিতি

> **GoF Definition:** *"Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality."*

Decorator প্যাটার্ন হলো একটি **Structural Design Pattern** যেটি আপনাকে একটি অবজেক্টের উপর নতুন behavior wrap করে যুক্ত করতে দেয় — **runtime-এ, dynamically** — মূল অবজেক্টের ক্লাস পরিবর্তন না করেই। এটি inheritance-এর একটি শক্তিশালী বিকল্প।

### মূল ধারণা

ধরুন আপনার একটি `Notifier` ক্লাস আছে যেটি শুধু SMS পাঠায়। এখন আপনি চান Email-ও পাঠাবে, Slack-এও পাঠাবে। Inheritance দিয়ে করলে class explosion হবে — `SMSAndEmailNotifier`, `SMSAndSlackNotifier`, `SMSAndEmailAndSlackNotifier`... এভাবে combination বাড়তেই থাকবে।

Decorator প্যাটার্নে আপনি প্রতিটি অতিরিক্ত responsibility কে আলাদা **wrapper class** হিসেবে তৈরি করেন। তারপর সেগুলো একটার উপর আরেকটা **stack** করে wrap করেন — ঠিক পেঁয়াজের খোসার মতো।

### মূল নীতি (Core Principles)

1. **Open/Closed Principle (OCP):** বিদ্যমান কোড পরিবর্তন না করে নতুন functionality যোগ করা
2. **Single Responsibility Principle (SRP):** প্রতিটি decorator একটিমাত্র responsibility handle করে
3. **Composition over Inheritance:** inheritance এর বদলে composition ব্যবহার
4. **Interface Segregation:** সব decorator একই interface implement করে — client কোড জানে না কতগুলো layer wrap করা আছে

---

## 🏠 বাস্তব জীবনের উদাহরণ

### ☕ কফি শপ অ্যানালজি

একটি কফি শপে ধরুন:
- **Base Coffee** = ৳১৫০ (Plain Black Coffee)
- **+ Milk** = ৳৩০ অতিরিক্ত
- **+ Sugar** = ৳১০ অতিরিক্ত
- **+ Whipped Cream** = ৳৫০ অতিরিক্ত

আপনি একটি plain coffee নেন, তারপর তার উপর milk decorator যোগ করেন, তার উপর sugar decorator — প্রতিটি layer আগের layer কে wrap করে এবং নিজের দাম ও description যোগ করে।

```
[Whipped Cream Decorator]
  └─ [Sugar Decorator]
       └─ [Milk Decorator]
            └─ [Base Coffee]

মোট = ১৫০ + ৩০ + ১০ + ৫০ = ৳২৪০
```

### 🇧🇩 বাংলাদেশ কনটেক্সট — ই-কমার্স অর্ডার

Daraz বা Chaldal-এ অর্ডার করার সময়:
- **Base Product** = ৳৫০০ (একটি শার্ট)
- **+ Gift Wrapping** = ৳৫০
- **+ Express Delivery** = ৳১০০
- **+ Insurance** = ৳৩০

প্রতিটি add-on একটি decorator — মূল প্রোডাক্টের ক্লাস পরিবর্তন না করেই নতুন feature যোগ হচ্ছে।

### 🇧🇩 bKash API Middleware

bKash পেমেন্ট API-তে একটি request process করার সময়:
```
[Rate Limiter] → [Auth Token Validator] → [Request Logger] → [Actual bKash API Call]
```
প্রতিটি middleware আসলে একটি decorator — মূল API handler কে wrap করে অতিরিক্ত responsibility যোগ করছে।

---

## 📊 UML ডায়াগ্রাম

```
  ┌─────────────────────────┐
  │    <<interface>>         │
  │      Component          │
  ├─────────────────────────┤
  │ + operation(): string   │
  │ + cost(): float         │
  └────────┬────────────────┘
           │ implements
     ┌─────┴──────────────────────────┐
     │                                │
┌────┴──────────────┐    ┌───────────┴──────────────┐
│ ConcreteComponent  │    │   <<abstract>>            │
│ (BaseCoffee)       │    │   Decorator               │
├────────────────────┤    ├──────────────────────────┤
│ + operation()      │    │ - wrapped: Component      │
│ + cost()           │    ├──────────────────────────┤
└────────────────────┘    │ + __construct(Component)  │
                          │ + operation()             │
                          │ + cost()                  │
                          └───────────┬──────────────┘
                                      │ extends
                    ┌─────────────────┼──────────────────┐
                    │                 │                   │
           ┌────────┴───┐   ┌───────┴──────┐   ┌───────┴──────────┐
           │ MilkDeco    │   │ SugarDeco    │   │ WhipCreamDeco    │
           ├─────────────┤   ├──────────────┤   ├──────────────────┤
           │ +operation()│   │ +operation() │   │ +operation()     │
           │ +cost()     │   │ +cost()      │   │ +cost()          │
           └─────────────┘   └──────────────┘   └──────────────────┘

  Decorator চেইন কিভাবে কাজ করে:
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Client       │───▶│ WhipCream    │───▶│ Sugar        │───▶│ BaseCoffee   │
  │ calls cost() │    │ 50 + next    │    │ 10 + next    │    │ returns 150  │
  └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                       cost() = 210        cost() = 160        cost() = 150
```

---

## 💻 ইমপ্লিমেন্টেশন

### 1️⃣ Basic Decorator — কফি শপ সিস্টেম

#### PHP 8.3

```php
<?php

declare(strict_types=1);

// Component Interface
interface Coffee
{
    public function description(): string;
    public function cost(): float;
}

// Concrete Component
class BasicCoffee implements Coffee
{
    public function description(): string
    {
        return 'বেসিক কফি';
    }

    public function cost(): float
    {
        return 150.0;
    }
}

// Abstract Decorator — এটি মূল chaining mechanism
abstract class CoffeeDecorator implements Coffee
{
    public function __construct(
        protected readonly Coffee $wrapped
    ) {}

    public function description(): string
    {
        return $this->wrapped->description();
    }

    public function cost(): float
    {
        return $this->wrapped->cost();
    }
}

// Concrete Decorators
class MilkDecorator extends CoffeeDecorator
{
    public function description(): string
    {
        return $this->wrapped->description() . ' + দুধ';
    }

    public function cost(): float
    {
        return $this->wrapped->cost() + 30.0;
    }
}

class SugarDecorator extends CoffeeDecorator
{
    public function description(): string
    {
        return $this->wrapped->description() . ' + চিনি';
    }

    public function cost(): float
    {
        return $this->wrapped->cost() + 10.0;
    }
}

class WhippedCreamDecorator extends CoffeeDecorator
{
    public function description(): string
    {
        return $this->wrapped->description() . ' + হুইপড ক্রিম';
    }

    public function cost(): float
    {
        return $this->wrapped->cost() + 50.0;
    }
}

// ব্যবহার — decorator stacking
$coffee = new BasicCoffee();
$coffee = new MilkDecorator($coffee);
$coffee = new SugarDecorator($coffee);
$coffee = new WhippedCreamDecorator($coffee);

echo $coffee->description(); // বেসিক কফি + দুধ + চিনি + হুইপড ক্রিম
echo "৳" . $coffee->cost();  // ৳240
```

#### JavaScript (ES2022+)

```javascript
// Component Interface (duck typing + JSDoc)

/** @typedef {{ description(): string, cost(): number }} Coffee */

// Concrete Component
class BasicCoffee {
    description() {
        return 'বেসিক কফি';
    }

    cost() {
        return 150;
    }
}

// Abstract Decorator — ES2022 private fields ব্যবহার
class CoffeeDecorator {
    #wrapped;

    constructor(coffee) {
        if (new.target === CoffeeDecorator) {
            throw new Error('CoffeeDecorator সরাসরি instantiate করা যাবে না');
        }
        this.#wrapped = coffee;
    }

    get wrapped() {
        return this.#wrapped;
    }

    description() {
        return this.#wrapped.description();
    }

    cost() {
        return this.#wrapped.cost();
    }
}

// Concrete Decorators
class MilkDecorator extends CoffeeDecorator {
    description() {
        return `${this.wrapped.description()} + দুধ`;
    }

    cost() {
        return this.wrapped.cost() + 30;
    }
}

class SugarDecorator extends CoffeeDecorator {
    description() {
        return `${this.wrapped.description()} + চিনি`;
    }

    cost() {
        return this.wrapped.cost() + 10;
    }
}

class WhippedCreamDecorator extends CoffeeDecorator {
    description() {
        return `${this.wrapped.description()} + হুইপড ক্রিম`;
    }

    cost() {
        return this.wrapped.cost() + 50;
    }
}

// ব্যবহার
let coffee = new BasicCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
coffee = new WhippedCreamDecorator(coffee);

console.log(coffee.description()); // বেসিক কফি + দুধ + চিনি + হুইপড ক্রিম
console.log(`৳${coffee.cost()}`);  // ৳240
```

---

### 2️⃣ পিৎজা/ই-কমার্স অর্ডার সিস্টেম (Price Calculation)

#### PHP 8.3 — বাংলাদেশ ই-কমার্স কনটেক্সট

```php
<?php

declare(strict_types=1);

interface OrderItem
{
    public function name(): string;
    public function price(): float;
    public function breakdown(): array;
}

class BaseProduct implements OrderItem
{
    public function __construct(
        private readonly string $productName,
        private readonly float $basePrice
    ) {}

    public function name(): string
    {
        return $this->productName;
    }

    public function price(): float
    {
        return $this->basePrice;
    }

    public function breakdown(): array
    {
        return [['item' => $this->productName, 'amount' => $this->basePrice]];
    }
}

abstract class OrderDecorator implements OrderItem
{
    public function __construct(
        protected readonly OrderItem $order
    ) {}

    public function name(): string
    {
        return $this->order->name();
    }

    public function price(): float
    {
        return $this->order->price();
    }

    public function breakdown(): array
    {
        return $this->order->breakdown();
    }
}

class GiftWrapping extends OrderDecorator
{
    public function name(): string
    {
        return $this->order->name() . ' 🎁';
    }

    public function price(): float
    {
        return $this->order->price() + 50.0;
    }

    public function breakdown(): array
    {
        return [
            ...$this->order->breakdown(),
            ['item' => 'গিফট র‍্যাপিং', 'amount' => 50.0],
        ];
    }
}

class ExpressDelivery extends OrderDecorator
{
    public function price(): float
    {
        return $this->order->price() + 100.0;
    }

    public function breakdown(): array
    {
        return [
            ...$this->order->breakdown(),
            ['item' => 'এক্সপ্রেস ডেলিভারি (ঢাকার মধ্যে)', 'amount' => 100.0],
        ];
    }
}

class InsuranceAddon extends OrderDecorator
{
    // বীমা মূল্য পণ্যের দামের ২% — dynamic calculation
    public function price(): float
    {
        $insuranceCost = $this->order->price() * 0.02;
        return $this->order->price() + $insuranceCost;
    }

    public function breakdown(): array
    {
        $insuranceCost = $this->order->price() * 0.02;
        return [
            ...$this->order->breakdown(),
            ['item' => 'প্রোডাক্ট ইন্সুরেন্স (২%)', 'amount' => $insuranceCost],
        ];
    }
}

class VATDecorator extends OrderDecorator
{
    public function __construct(
        OrderItem $order,
        private readonly float $vatRate = 0.075 // বাংলাদেশে ৭.৫% VAT
    ) {
        parent::__construct($order);
    }

    public function price(): float
    {
        return $this->order->price() * (1 + $this->vatRate);
    }

    public function breakdown(): array
    {
        $vatAmount = $this->order->price() * $this->vatRate;
        return [
            ...$this->order->breakdown(),
            ['item' => "ভ্যাট ({$this->vatRate * 100}%)", 'amount' => $vatAmount],
        ];
    }
}

// ব্যবহার
$order = new BaseProduct('স্যামসাং গ্যালাক্সি A54', 35000.0);
$order = new GiftWrapping($order);
$order = new ExpressDelivery($order);
$order = new InsuranceAddon($order);
$order = new VATDecorator($order);

echo "অর্ডার: {$order->name()}\n";
echo "সর্বমোট: ৳" . number_format($order->price(), 2) . "\n\n";

echo "--- বিস্তারিত ---\n";
foreach ($order->breakdown() as $line) {
    printf("  %-35s ৳%s\n", $line['item'], number_format($line['amount'], 2));
}
```

#### JavaScript (ES2022+)

```javascript
class BaseProduct {
    #name;
    #price;

    constructor(name, price) {
        this.#name = name;
        this.#price = price;
    }

    name()      { return this.#name; }
    price()     { return this.#price; }
    breakdown() { return [{ item: this.#name, amount: this.#price }]; }
}

class OrderDecorator {
    #order;
    constructor(order) { this.#order = order; }
    get order()   { return this.#order; }
    name()        { return this.#order.name(); }
    price()       { return this.#order.price(); }
    breakdown()   { return this.#order.breakdown(); }
}

class GiftWrapping extends OrderDecorator {
    name()  { return `${this.order.name()} 🎁`; }
    price() { return this.order.price() + 50; }
    breakdown() {
        return [...this.order.breakdown(), { item: 'গিফট র‍্যাপিং', amount: 50 }];
    }
}

class ExpressDelivery extends OrderDecorator {
    price() { return this.order.price() + 100; }
    breakdown() {
        return [...this.order.breakdown(), { item: 'এক্সপ্রেস ডেলিভারি', amount: 100 }];
    }
}

class VATDecorator extends OrderDecorator {
    #rate;
    constructor(order, rate = 0.075) {
        super(order);
        this.#rate = rate;
    }
    price() { return this.order.price() * (1 + this.#rate); }
    breakdown() {
        const vatAmt = this.order.price() * this.#rate;
        return [...this.order.breakdown(), { item: `ভ্যাট (${this.#rate * 100}%)`, amount: vatAmt }];
    }
}

// ব্যবহার
let order = new BaseProduct('স্যামসাং গ্যালাক্সি A54', 35000);
order = new GiftWrapping(order);
order = new ExpressDelivery(order);
order = new VATDecorator(order);

console.log(`অর্ডার: ${order.name()}`);
console.log(`সর্বমোট: ৳${order.price().toFixed(2)}`);
order.breakdown().forEach(({ item, amount }) => {
    console.log(`  ${item.padEnd(30)} ৳${amount.toFixed(2)}`);
});
```

---

### 3️⃣ Stream Decorators — Encryption → Compression → Buffering

এটি Decorator pattern-এর সবচেয়ে classic real-world use case। PHP-র নিজের `php://filter` stream wrappers এই pattern-ই ব্যবহার করে।

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface DataStream
{
    public function write(string $data): string;
    public function read(string $data): string;
}

class FileStream implements DataStream
{
    public function write(string $data): string
    {
        // সরাসরি raw data write
        return $data;
    }

    public function read(string $data): string
    {
        return $data;
    }
}

abstract class StreamDecorator implements DataStream
{
    public function __construct(
        protected readonly DataStream $stream
    ) {}
}

class EncryptionDecorator extends StreamDecorator
{
    public function __construct(
        DataStream $stream,
        private readonly string $key = 'secret-key-256'
    ) {
        parent::__construct($stream);
    }

    public function write(string $data): string
    {
        $encrypted = $this->encrypt($data);
        return $this->stream->write($encrypted);
    }

    public function read(string $data): string
    {
        $raw = $this->stream->read($data);
        return $this->decrypt($raw);
    }

    private function encrypt(string $data): string
    {
        // সরলীকৃত — প্রোডাকশনে openssl_encrypt ব্যবহার করুন
        return base64_encode(openssl_encrypt(
            $data, 'AES-256-CBC', $this->key,
            0, str_pad('', 16, "\0")
        ));
    }

    private function decrypt(string $data): string
    {
        return openssl_decrypt(
            base64_decode($data), 'AES-256-CBC', $this->key,
            0, str_pad('', 16, "\0")
        );
    }
}

class CompressionDecorator extends StreamDecorator
{
    public function write(string $data): string
    {
        $compressed = gzcompress($data, 9);
        return $this->stream->write($compressed);
    }

    public function read(string $data): string
    {
        $raw = $this->stream->read($data);
        return gzuncompress($raw);
    }
}

class BufferingDecorator extends StreamDecorator
{
    private array $buffer = [];

    public function __construct(
        DataStream $stream,
        private readonly int $bufferSize = 1024
    ) {
        parent::__construct($stream);
    }

    public function write(string $data): string
    {
        $this->buffer[] = $data;
        $combined = implode('', $this->buffer);

        if (strlen($combined) >= $this->bufferSize) {
            $this->buffer = [];
            return $this->stream->write($combined);
        }

        return ''; // buffer-এ রেখে দিলাম, flush হয়নি
    }

    public function flush(): string
    {
        $combined = implode('', $this->buffer);
        $this->buffer = [];
        return $this->stream->write($combined);
    }

    public function read(string $data): string
    {
        return $this->stream->read($data);
    }
}

// Decorator stacking — ক্রম গুরুত্বপূর্ণ!
// ভেতর থেকে বাইরে: File ← Encryption ← Compression ← Buffering
$stream = new FileStream();
$stream = new EncryptionDecorator($stream);
$stream = new CompressionDecorator($stream);
$stream = new BufferingDecorator($stream, 512);

$result = $stream->write('বাংলাদেশের গোপন তথ্য...');
```

#### JavaScript (ES2022+)

```javascript
class FileStream {
    write(data) { return data; }
    read(data)  { return data; }
}

class EncryptionDecorator {
    #stream;
    #key;

    constructor(stream, key = 'secret') {
        this.#stream = stream;
        this.#key = key;
    }

    write(data) {
        const encrypted = this.#simpleEncrypt(data);
        return this.#stream.write(encrypted);
    }

    read(data) {
        const raw = this.#stream.read(data);
        return this.#simpleDecrypt(raw);
    }

    #simpleEncrypt(data) {
        // সরলীকৃত XOR — প্রোডাকশনে Web Crypto API ব্যবহার করুন
        return btoa(
            [...data].map((c, i) =>
                String.fromCharCode(c.charCodeAt(0) ^ this.#key.charCodeAt(i % this.#key.length))
            ).join('')
        );
    }

    #simpleDecrypt(data) {
        const decoded = atob(data);
        return [...decoded].map((c, i) =>
            String.fromCharCode(c.charCodeAt(0) ^ this.#key.charCodeAt(i % this.#key.length))
        ).join('');
    }
}

class CompressionDecorator {
    #stream;

    constructor(stream) {
        this.#stream = stream;
    }

    async write(data) {
        const blob = new Blob([data]);
        const cs = new CompressionStream('gzip');
        const compressed = blob.stream().pipeThrough(cs);
        const buffer = await new Response(compressed).arrayBuffer();
        return this.#stream.write(new Uint8Array(buffer));
    }

    read(data) {
        return this.#stream.read(data);
    }
}

class BufferingDecorator {
    #stream;
    #buffer = [];
    #bufferSize;

    constructor(stream, bufferSize = 1024) {
        this.#stream = stream;
        this.#bufferSize = bufferSize;
    }

    write(data) {
        this.#buffer.push(data);
        const combined = this.#buffer.join('');

        if (combined.length >= this.#bufferSize) {
            this.#buffer = [];
            return this.#stream.write(combined);
        }
        return null; // এখনো flush হয়নি
    }

    flush() {
        const combined = this.#buffer.join('');
        this.#buffer = [];
        return this.#stream.write(combined);
    }

    read(data) { return this.#stream.read(data); }
}

// ব্যবহার
let stream = new FileStream();
stream = new EncryptionDecorator(stream);
stream = new BufferingDecorator(stream, 512);

stream.write('বাংলাদেশের গোপন তথ্য...');
```

---

## 🌍 Real-World Applicable Areas

### 1. HTTP Middleware — Laravel Middleware IS Decorator Pattern!

Laravel-এর middleware পুরোপুরি Decorator pattern। প্রতিটি middleware request অবজেক্ট কে wrap করে, কিছু কাজ করে, তারপর পরবর্তী layer-এ পাঠায়।

#### PHP 8.3 — Laravel-স্টাইল Middleware Pipeline

```php
<?php

declare(strict_types=1);

// সরলীকৃত Request/Response
class Request
{
    public function __construct(
        public readonly string $method,
        public readonly string $path,
        public array $headers = [],
        public array $attributes = [],
    ) {}
}

class Response
{
    public function __construct(
        public readonly int $status,
        public readonly string $body,
        public array $headers = [],
    ) {}
}

// Middleware Interface
interface Middleware
{
    public function handle(Request $request, Closure $next): Response;
}

// bKash API Authentication Middleware
class BkashAuthMiddleware implements Middleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $token = $request->headers['Authorization'] ?? null;

        if (!$token || !str_starts_with($token, 'Bearer ')) {
            return new Response(401, 'bKash টোকেন প্রয়োজন');
        }

        // টোকেন verify করে user info যোগ করি
        $request->attributes['bkash_merchant'] = 'merchant_12345';

        return $next($request);
    }
}

// Rate Limiting Middleware
class RateLimitMiddleware implements Middleware
{
    private static array $requests = [];

    public function __construct(
        private readonly int $maxRequests = 60,
        private readonly int $windowSeconds = 60
    ) {}

    public function handle(Request $request, Closure $next): Response
    {
        $ip = $request->headers['X-Forwarded-For'] ?? '127.0.0.1';
        $now = time();
        $key = $ip;

        self::$requests[$key] = array_filter(
            self::$requests[$key] ?? [],
            fn(int $time) => ($now - $time) < $this->windowSeconds
        );

        if (count(self::$requests[$key]) >= $this->maxRequests) {
            return new Response(429, 'অনুগ্রহ করে কিছুক্ষণ পর চেষ্টা করুন');
        }

        self::$requests[$key][] = $now;

        $response = $next($request);

        $remaining = $this->maxRequests - count(self::$requests[$key]);
        $response->headers['X-RateLimit-Remaining'] = (string) $remaining;

        return $response;
    }
}

// Logging Middleware
class RequestLogMiddleware implements Middleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $start = microtime(true);

        $response = $next($request);

        $duration = round((microtime(true) - $start) * 1000, 2);
        error_log("[{$request->method}] {$request->path} → {$response->status} ({$duration}ms)");

        return $response;
    }
}

// Pipeline — Laravel's Illuminate\Pipeline এর সরলীকৃত রূপ
class Pipeline
{
    private array $middlewares = [];

    public function through(Middleware ...$middlewares): self
    {
        $this->middlewares = $middlewares;
        return $this;
    }

    public function process(Request $request, Closure $destination): Response
    {
        $pipeline = array_reduce(
            array_reverse($this->middlewares),
            fn(Closure $next, Middleware $middleware) =>
                fn(Request $req) => $middleware->handle($req, $next),
            $destination
        );

        return $pipeline($request);
    }
}

// ব্যবহার — bKash Payment API
$pipeline = new Pipeline();

$response = $pipeline
    ->through(
        new RequestLogMiddleware(),
        new RateLimitMiddleware(maxRequests: 100),
        new BkashAuthMiddleware(),
    )
    ->process(
        new Request('POST', '/api/bkash/payment', [
            'Authorization' => 'Bearer bkash_token_xyz',
            'X-Forwarded-For' => '103.108.8.1',
        ]),
        fn(Request $req) => new Response(200, json_encode([
            'status'  => 'success',
            'trxID'   => 'TRX' . random_int(100000, 999999),
            'merchant' => $req->attributes['bkash_merchant'],
        ]))
    );

echo $response->body;
```

### 2. Logging Decorator — যেকোনো Service-এ Logging যোগ করুন

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface PaymentGateway
{
    public function charge(float $amount, string $currency): array;
}

class BkashGateway implements PaymentGateway
{
    public function charge(float $amount, string $currency): array
    {
        // প্রকৃত bKash API call
        return ['trxID' => 'BK' . random_int(100000, 999999), 'status' => 'success'];
    }
}

// এই decorator যেকোনো PaymentGateway-তে logging যোগ করবে
class LoggingPaymentDecorator implements PaymentGateway
{
    public function __construct(
        private readonly PaymentGateway $gateway,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {}

    public function charge(float $amount, string $currency): array
    {
        $this->logger->info('পেমেন্ট শুরু', compact('amount', 'currency'));

        try {
            $result = $this->gateway->charge($amount, $currency);
            $this->logger->info('পেমেন্ট সফল', $result);
            return $result;
        } catch (\Throwable $e) {
            $this->logger->error('পেমেন্ট ব্যর্থ', [
                'error' => $e->getMessage(),
                'amount' => $amount,
            ]);
            throw $e;
        }
    }
}
```

### 3. Caching Decorator — Repository Pattern-এ Cache যোগ

#### PHP 8.3

```php
<?php

declare(strict_types=1);

interface ProductRepository
{
    public function find(int $id): ?array;
    public function findAll(): array;
}

class DatabaseProductRepository implements ProductRepository
{
    public function find(int $id): ?array
    {
        // ডাটাবেস query
        return ['id' => $id, 'name' => 'প্রোডাক্ট #' . $id, 'price' => 500.0];
    }

    public function findAll(): array
    {
        return []; // DB থেকে সব প্রোডাক্ট
    }
}

class CachingProductRepository implements ProductRepository
{
    private array $cache = [];

    public function __construct(
        private readonly ProductRepository $repository,
        private readonly int $ttlSeconds = 3600
    ) {}

    public function find(int $id): ?array
    {
        $key = "product_{$id}";

        if ($this->hasValidCache($key)) {
            return $this->cache[$key]['data'];
        }

        $product = $this->repository->find($id);

        if ($product !== null) {
            $this->cache[$key] = [
                'data' => $product,
                'expires_at' => time() + $this->ttlSeconds,
            ];
        }

        return $product;
    }

    public function findAll(): array
    {
        $key = 'all_products';

        if ($this->hasValidCache($key)) {
            return $this->cache[$key]['data'];
        }

        $products = $this->repository->findAll();
        $this->cache[$key] = [
            'data' => $products,
            'expires_at' => time() + $this->ttlSeconds,
        ];

        return $products;
    }

    private function hasValidCache(string $key): bool
    {
        return isset($this->cache[$key])
            && $this->cache[$key]['expires_at'] > time();
    }
}

// একসাথে Logging + Caching decorator stack করা
class LoggingProductRepository implements ProductRepository
{
    public function __construct(
        private readonly ProductRepository $repository
    ) {}

    public function find(int $id): ?array
    {
        error_log("ProductRepository::find({$id}) called");
        $start = microtime(true);
        $result = $this->repository->find($id);
        $ms = round((microtime(true) - $start) * 1000, 2);
        error_log("ProductRepository::find({$id}) took {$ms}ms");
        return $result;
    }

    public function findAll(): array
    {
        error_log("ProductRepository::findAll() called");
        return $this->repository->findAll();
    }
}

// Decorator stacking — ক্রম গুরুত্বপূর্ণ!
// Logging → Caching → Database
// (Logging দেখবে cache hit হলে দ্রুত, miss হলে ধীর)
$repo = new DatabaseProductRepository();
$repo = new CachingProductRepository($repo, ttlSeconds: 300);
$repo = new LoggingProductRepository($repo);

$product = $repo->find(42);  // DB hit → cache save
$product = $repo->find(42);  // cache hit — অনেক দ্রুত!
```

#### JavaScript (ES2022+) — Caching Decorator

```javascript
class ProductRepository {
    async find(id) {
        // API call
        const res = await fetch(`/api/products/${id}`);
        return res.json();
    }
}

class CachingDecorator {
    #repo;
    #cache = new Map();
    #ttl;

    constructor(repo, ttlMs = 60_000) {
        this.#repo = repo;
        this.#ttl = ttlMs;
    }

    async find(id) {
        const key = `product_${id}`;
        const cached = this.#cache.get(key);

        if (cached && Date.now() < cached.expiresAt) {
            return cached.data;
        }

        const data = await this.#repo.find(id);
        this.#cache.set(key, { data, expiresAt: Date.now() + this.#ttl });
        return data;
    }
}

class LoggingDecorator {
    #repo;

    constructor(repo) {
        this.#repo = repo;
    }

    async find(id) {
        console.time(`find(${id})`);
        const result = await this.#repo.find(id);
        console.timeEnd(`find(${id})`);
        return result;
    }
}

// Stacking
let repo = new ProductRepository();
repo = new CachingDecorator(repo, 30_000);
repo = new LoggingDecorator(repo);

await repo.find(42); // API call
await repo.find(42); // cache hit
```

### 4. Express.js Middleware as Decorators

```javascript
// Express middleware আসলে decorator pattern-ই
// প্রতিটি middleware req/res কে "decorate" করে

import express from 'express';
const app = express();

// Authentication Decorator
const authMiddleware = (req, res, next) => {
    const token = req.headers.authorization;
    if (!token) return res.status(401).json({ error: 'টোকেন নেই' });
    req.user = { id: 1, name: 'রহিম' }; // token verify করে user বসানো
    next();
};

// Rate Limiting Decorator
const rateLimiter = (() => {
    const hits = new Map();
    return (req, res, next) => {
        const ip = req.ip;
        const count = (hits.get(ip) ?? 0) + 1;
        hits.set(ip, count);
        if (count > 100) return res.status(429).json({ error: 'সীমা অতিক্রম' });
        setTimeout(() => hits.set(ip, (hits.get(ip) ?? 1) - 1), 60_000);
        next();
    };
})();

// Decorator stacking — ক্রম গুরুত্বপূর্ণ
app.use(rateLimiter);       // বাইরের layer — প্রথমে rate check
app.use(authMiddleware);    // তারপর auth check

app.post('/api/bkash/pay', (req, res) => {
    res.json({ trxID: 'TRX' + Date.now(), user: req.user });
});
```

### 5. TypeScript/JS Decorator Syntax (Stage 3 Proposal)

```javascript
// TC39 Stage 3 Decorators — 2024 syntax
// এটি method-level decoration

function logged(originalMethod, context) {
    const methodName = context.name;

    return function (...args) {
        console.log(`→ ${methodName}(${args.join(', ')})`);
        const result = originalMethod.call(this, ...args);
        console.log(`← ${methodName} returned:`, result);
        return result;
    };
}

function cached(originalMethod, context) {
    const cache = new Map();

    return function (...args) {
        const key = JSON.stringify(args);
        if (cache.has(key)) return cache.get(key);
        const result = originalMethod.call(this, ...args);
        cache.set(key, result);
        return result;
    };
}

function timeit(originalMethod, context) {
    return function (...args) {
        const start = performance.now();
        const result = originalMethod.call(this, ...args);
        console.log(`${context.name}: ${(performance.now() - start).toFixed(2)}ms`);
        return result;
    };
}

class ProductService {
    @logged
    @cached
    @timeit
    findProduct(id) {
        // ভারী operation
        return { id, name: `প্রোডাক্ট ${id}` };
    }
}

const svc = new ProductService();
svc.findProduct(1); // logged → cached (miss) → timeit → actual call
svc.findProduct(1); // logged → cached (hit!) → return immediately
```

### 6. PHP 8 Attributes as Metadata Decorators

```php
<?php

declare(strict_types=1);

// PHP 8 Attributes — metadata-based decoration
#[\Attribute(\Attribute::TARGET_METHOD)]
class Cacheable
{
    public function __construct(
        public readonly int $ttl = 3600,
        public readonly string $prefix = ''
    ) {}
}

#[\Attribute(\Attribute::TARGET_METHOD)]
class RateLimit
{
    public function __construct(
        public readonly int $maxAttempts = 60,
        public readonly int $windowSeconds = 60
    ) {}
}

#[\Attribute(\Attribute::TARGET_METHOD)]
class LogExecution {}

class BkashPaymentService
{
    #[LogExecution]
    #[RateLimit(maxAttempts: 10, windowSeconds: 60)]
    #[Cacheable(ttl: 300, prefix: 'bkash')]
    public function getBalance(string $merchantId): array
    {
        return ['balance' => 50000.00, 'currency' => 'BDT'];
    }
}

// Attribute Reader — runtime-এ attribute পড়ে decorator-এর মতো আচরণ করে
class AttributeDecoratorProxy
{
    public function __construct(
        private readonly object $target
    ) {}

    public function __call(string $method, array $args): mixed
    {
        $reflection = new \ReflectionMethod($this->target, $method);
        $attributes = $reflection->getAttributes();

        foreach ($attributes as $attr) {
            $instance = $attr->newInstance();

            match (true) {
                $instance instanceof LogExecution => error_log("Executing: {$method}"),
                $instance instanceof RateLimit    => $this->checkRateLimit($instance),
                $instance instanceof Cacheable    => null, // cache logic
                default => null,
            };
        }

        return $reflection->invoke($this->target, ...$args);
    }

    private function checkRateLimit(RateLimit $limit): void
    {
        // Rate limiting logic
    }
}

$service = new AttributeDecoratorProxy(new BkashPaymentService());
$service->getBalance('MERCHANT_001');
```

---

## 🔥 Advanced Deep Dive

### Decorator vs Inheritance — কখন কোনটি?

```
┌──────────────────────────┬─────────────────────────────────────────┐
│      Inheritance          │         Decorator                       │
├──────────────────────────┼─────────────────────────────────────────┤
│ Compile-time extension    │ Runtime extension                       │
│ Static, fixed hierarchy   │ Dynamic, flexible composition           │
│ "is-a" relationship       │ "has-a" / "wraps-a" relationship        │
│ Class explosion সমস্যা    │ N decorators = N classes, ∞ combos      │
│ Subclass সব method পায়   │ শুধু interface-এর method delegate হয়    │
│ Tight coupling            │ Loose coupling                          │
│ তুলনামূলক সহজ            │ শুরুতে জটিল, পরে রক্ষণাবেক্ষণ সহজ      │
└──────────────────────────┴─────────────────────────────────────────┘
```

**Inheritance ব্যবহার করুন যখন:**
- সত্যিকারের "is-a" সম্পর্ক আছে (Dog is-a Animal)
- Behavior combinations কম এবং fixed
- Performance critical — delegate call-এর overhead নেই

**Decorator ব্যবহার করুন যখন:**
- Runtime-এ behavior যোগ/বাদ দিতে হবে
- Combinations অনেক (2টি option-এ 4টি combo, 3টিতে 8টি!)
- Single Responsibility মেনে চলতে হবে
- Existing class modify করা সম্ভব না (third-party library)

### Decorator vs Proxy vs Adapter

```
┌────────────┬──────────────────────────┬────────────────────────────┐
│  Pattern   │  উদ্দেশ্য                │  Interface পরিবর্তন?       │
├────────────┼──────────────────────────┼────────────────────────────┤
│ Decorator  │ নতুন behavior যোগ করা    │ না — একই interface          │
│ Proxy      │ Access control করা       │ না — একই interface          │
│ Adapter    │ Incompatible interface    │ হ্যাঁ — নতুন interface      │
│            │  compatible করা          │                             │
└────────────┴──────────────────────────┴────────────────────────────┘
```

**মূল পার্থক্য:**
- **Decorator** = "আমি তোমার কাজ করবো + আরো কিছু extra করবো"
- **Proxy** = "আমি তোমার হয়ে কাজ করবো, কিন্তু access control করবো"
- **Adapter** = "আমি তোমার ভাষা বুঝি না, আমি translate করে দিচ্ছি"

### Stacking Multiple Decorators — ক্রম গুরুত্বপূর্ণ!

Decorator stacking-এ ক্রম (order) অত্যন্ত গুরুত্বপূর্ণ:

```php
<?php
// ক্রম ১: Logging → Caching → Database
// Log দেখবে: cache hit হলে 0.1ms, miss হলে 50ms
$repo = new DatabaseProductRepo();
$repo = new CachingDecorator($repo);
$repo = new LoggingDecorator($repo);  // ✅ সঠিক

// ক্রম ২: Caching → Logging → Database
// Log দেখবে: সবসময় DB call-এর সময় (cache-র ভেতরে)
// কিন্তু cache HIT হলে log-ই হবে না!
$repo = new DatabaseProductRepo();
$repo = new LoggingDecorator($repo);
$repo = new CachingDecorator($repo);  // ❌ ভুল ক্রম (usually)
```

```
সঠিক ক্রম (বাইরে → ভেতরে):
[Logging] → [Rate Limit] → [Auth] → [Caching] → [Actual Service]

কারণ:
- Logging সবার বাইরে → সব request log হয়
- Rate limit → auth-এর আগে (unwanted request আটকায়)
- Auth → cache-এর আগে (unauthorized user cache পায় না)
- Cache → service-এর আগে (DB call বাঁচায়)
```

### Decorator with Dependency Injection (Laravel Container)

```php
<?php

declare(strict_types=1);

// Laravel Service Provider-এ decorator bind করা
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // মূল repository bind
        $this->app->bind(ProductRepository::class, function ($app) {
            // ভেতর থেকে বাইরে decorator stack
            $repo = new DatabaseProductRepository(
                $app->make(\Illuminate\Database\Connection::class)
            );

            // Cache decorator wrap
            $repo = new CachingProductRepository(
                $repo,
                $app->make(\Illuminate\Contracts\Cache\Repository::class)
            );

            // Logging decorator wrap
            if (config('app.debug')) {
                $repo = new LoggingProductRepository(
                    $repo,
                    $app->make(\Psr\Log\LoggerInterface::class)
                );
            }

            return $repo;
        });
    }
}
```

### Laravel Pipeline Class Internals

Laravel-এর `Illuminate\Pipeline\Pipeline` ক্লাস `array_reduce` দিয়ে middleware chain তৈরি করে — এটি decorator pattern-এর একটি elegant implementation:

```php
<?php
// Laravel Pipeline-এর সরলীকৃত রূপ
class Pipeline
{
    private array $pipes = [];
    private mixed $passable;

    public function send(mixed $passable): self
    {
        $this->passable = $passable;
        return $this;
    }

    public function through(array $pipes): self
    {
        $this->pipes = $pipes;
        return $this;
    }

    public function then(Closure $destination): mixed
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes),
            $this->carry(),
            fn($passable) => $destination($passable)
        );

        return $pipeline($this->passable);
    }

    // এখানেই Decorator pattern-এর magic ঘটে
    private function carry(): Closure
    {
        return function (Closure $stack, mixed $pipe): Closure {
            return function (mixed $passable) use ($stack, $pipe): mixed {
                if (is_callable($pipe)) {
                    return $pipe($passable, $stack);
                }

                // String হলে class resolve করো (Laravel container)
                $instance = new $pipe();
                return $instance->handle($passable, $stack);
            };
        };
    }
}

// ব্যবহার
$result = (new Pipeline())
    ->send($request)
    ->through([
        TrimStrings::class,
        ValidatePostSize::class,
        CheckForMaintenanceMode::class,
    ])
    ->then(fn($request) => $router->dispatch($request));
```

---

## ✅ Pros (সুবিধাসমূহ)

1. **Open/Closed Principle** — বিদ্যমান কোড পরিবর্তন না করে নতুন behavior যোগ
2. **Single Responsibility** — প্রতিটি decorator একটিমাত্র কাজ করে
3. **Runtime Flexibility** — runtime-এ decorator যোগ/বাদ দেওয়া যায়
4. **Composable** — একাধিক decorator stack করে complex behavior তৈরি
5. **Testability** — প্রতিটি decorator আলাদাভাবে test করা যায়
6. **No Class Explosion** — N feature-এর জন্য N class, 2^N নয়
7. **Transparent to Client** — client কোড জানে না কতগুলো layer আছে

## ❌ Cons (অসুবিধাসমূহ)

1. **Debugging Complexity** — অনেক layer হলে debug করা কঠিন (কোন layer-এ সমস্যা?)
2. **Order Dependency** — decorator-এর ক্রম ভুল হলে ভুল behavior
3. **Unwrapping Difficulty** — stack থেকে নির্দিষ্ট decorator বাদ দেওয়া কঠিন
4. **Performance Overhead** — প্রতিটি layer-এ একটি extra method call
5. **Identity Problem** — `$decorated instanceof ConcreteComponent` false হবে
6. **Initial Complexity** — ছোট project-এ over-engineering মনে হতে পারে

---

## ⚠️ Common Mistakes (সাধারণ ভুলসমূহ)

### ❌ ভুল ১: Decorator-এ নতুন public method যোগ করা

```php
// ❌ ভুল — নতুন method যোগ করলে polymorphism ভাঙে
class SpecialCoffee extends CoffeeDecorator
{
    public function description(): string { /* ... */ }
    public function cost(): float { /* ... */ }
    public function getCalories(): int { return 200; } // ❌ interface-এ নেই!
}

// Client কোড এটি access করতে পারবে না কারণ variable type হলো Coffee interface
$coffee = new SpecialCoffee(new BasicCoffee());
$coffee->getCalories(); // 💥 কাজ করলেও design ভুল

// ✅ সঠিক — interface-এ method যোগ করুন অথবা আলাদা concern হিসেবে handle করুন
```

### ❌ ভুল ২: Decorator-এ state maintain করা (সাবধানে!)

```php
// ⚠️ সাবধান — mutable state decorator-এ tricky
class CountingDecorator extends CoffeeDecorator
{
    private int $accessCount = 0; // ⚠️ shared state হতে পারে

    public function cost(): float
    {
        $this->accessCount++;
        return $this->wrapped->cost();
    }
}
```

### ❌ ভুল ৩: অতিরিক্ত Decorator Stacking

```php
// ❌ ১০+ layer decorator stack — debugging nightmare!
$service = new LogDecorator(
    new CacheDecorator(
        new RetryDecorator(
            new AuthDecorator(
                new RateLimitDecorator(
                    new ValidationDecorator(
                        new MetricsDecorator(
                            new ActualService()
                        )))))));

// ✅ ৩-৫ layer-এর বেশি হলে Pipeline/Chain of Responsibility বিবেচনা করুন
```

### ❌ ভুল ৪: Base decorator-এ delegation ভুলে যাওয়া

```php
// ❌ ভুল — সব method delegate করতে ভুলে গেছে
class MyDecorator implements Coffee
{
    public function __construct(private Coffee $wrapped) {}

    public function description(): string
    {
        return 'My Special ' . $this->wrapped->description();
    }

    // ❌ cost() delegate করতে ভুলে গেছে!
    // PHP Fatal Error হবে কারণ interface implement করা হয়নি
}
```

---

## 🧪 টেস্টিং

### PHPUnit — PHP 8.3

```php
<?php

declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;

class CoffeeDecoratorTest extends TestCase
{
    #[Test]
    public function basic_coffee_has_correct_price_and_description(): void
    {
        $coffee = new BasicCoffee();

        $this->assertSame(150.0, $coffee->cost());
        $this->assertSame('বেসিক কফি', $coffee->description());
    }

    #[Test]
    public function single_decorator_adds_cost(): void
    {
        $coffee = new MilkDecorator(new BasicCoffee());

        $this->assertSame(180.0, $coffee->cost());
        $this->assertStringContainsString('দুধ', $coffee->description());
    }

    #[Test]
    public function stacked_decorators_accumulate_cost(): void
    {
        $coffee = new BasicCoffee();
        $coffee = new MilkDecorator($coffee);
        $coffee = new SugarDecorator($coffee);
        $coffee = new WhippedCreamDecorator($coffee);

        $this->assertSame(240.0, $coffee->cost());
        $this->assertSame(
            'বেসিক কফি + দুধ + চিনি + হুইপড ক্রিম',
            $coffee->description()
        );
    }

    #[Test]
    public function decorator_order_does_not_affect_final_price(): void
    {
        // ক্রম ১: Milk → Sugar
        $order1 = new SugarDecorator(new MilkDecorator(new BasicCoffee()));

        // ক্রম ২: Sugar → Milk
        $order2 = new MilkDecorator(new SugarDecorator(new BasicCoffee()));

        $this->assertSame($order1->cost(), $order2->cost());
    }

    #[Test]
    public function same_decorator_can_be_applied_multiple_times(): void
    {
        $coffee = new BasicCoffee();
        $coffee = new SugarDecorator($coffee);
        $coffee = new SugarDecorator($coffee); // ডাবল চিনি!

        $this->assertSame(170.0, $coffee->cost()); // 150 + 10 + 10
    }

    #[Test]
    public function decorated_coffee_implements_same_interface(): void
    {
        $plain     = new BasicCoffee();
        $decorated = new WhippedCreamDecorator(
            new MilkDecorator($plain)
        );

        $this->assertInstanceOf(Coffee::class, $plain);
        $this->assertInstanceOf(Coffee::class, $decorated);
    }
}

// Caching Decorator Test
class CachingProductRepositoryTest extends TestCase
{
    #[Test]
    public function caches_result_on_first_call(): void
    {
        $mockRepo = $this->createMock(ProductRepository::class);
        $mockRepo->expects($this->once()) // শুধু একবার call হবে!
            ->method('find')
            ->with(42)
            ->willReturn(['id' => 42, 'name' => 'টেস্ট']);

        $cachedRepo = new CachingProductRepository($mockRepo);

        $first  = $cachedRepo->find(42);  // DB hit
        $second = $cachedRepo->find(42);  // cache hit

        $this->assertSame($first, $second);
    }

    #[Test]
    public function returns_null_for_missing_product(): void
    {
        $mockRepo = $this->createMock(ProductRepository::class);
        $mockRepo->method('find')->willReturn(null);

        $cachedRepo = new CachingProductRepository($mockRepo);

        $this->assertNull($cachedRepo->find(999));
    }
}

// Middleware Test
class MiddlewareTest extends TestCase
{
    #[Test]
    public function rate_limit_blocks_excess_requests(): void
    {
        $middleware = new RateLimitMiddleware(maxRequests: 2, windowSeconds: 60);
        $handler = fn(Request $r) => new Response(200, 'OK');

        $req = new Request('GET', '/api/test', ['X-Forwarded-For' => '10.0.0.1']);

        $r1 = $middleware->handle($req, $handler);
        $r2 = $middleware->handle($req, $handler);
        $r3 = $middleware->handle($req, $handler); // তৃতীয় request blocked

        $this->assertSame(200, $r1->status);
        $this->assertSame(200, $r2->status);
        $this->assertSame(429, $r3->status);
    }

    #[Test]
    public function auth_middleware_rejects_without_token(): void
    {
        $middleware = new BkashAuthMiddleware();
        $handler = fn(Request $r) => new Response(200, 'OK');

        $request = new Request('POST', '/api/pay'); // কোনো token নেই

        $response = $middleware->handle($request, $handler);
        $this->assertSame(401, $response->status);
    }
}
```

### Jest — JavaScript (ES2022+)

```javascript
import { describe, it, expect, jest } from '@jest/globals';

describe('Coffee Decorator', () => {
    it('basic coffee returns correct price', () => {
        const coffee = new BasicCoffee();
        expect(coffee.cost()).toBe(150);
        expect(coffee.description()).toBe('বেসিক কফি');
    });

    it('stacked decorators accumulate price', () => {
        let coffee = new BasicCoffee();
        coffee = new MilkDecorator(coffee);
        coffee = new SugarDecorator(coffee);
        coffee = new WhippedCreamDecorator(coffee);

        expect(coffee.cost()).toBe(240);
        expect(coffee.description()).toBe('বেসিক কফি + দুধ + চিনি + হুইপড ক্রিম');
    });

    it('same decorator can be applied multiple times', () => {
        let coffee = new BasicCoffee();
        coffee = new SugarDecorator(coffee);
        coffee = new SugarDecorator(coffee);

        expect(coffee.cost()).toBe(170); // 150 + 10 + 10
    });

    it('decorator order does not affect price', () => {
        const order1 = new SugarDecorator(new MilkDecorator(new BasicCoffee()));
        const order2 = new MilkDecorator(new SugarDecorator(new BasicCoffee()));

        expect(order1.cost()).toBe(order2.cost());
    });
});

describe('Caching Decorator', () => {
    it('caches result after first call', async () => {
        const mockRepo = {
            find: jest.fn().mockResolvedValue({ id: 42, name: 'টেস্ট' }),
        };

        const cached = new CachingDecorator(mockRepo);

        await cached.find(42);
        await cached.find(42);

        expect(mockRepo.find).toHaveBeenCalledTimes(1); // শুধু একবার!
    });

    it('re-fetches after TTL expires', async () => {
        jest.useFakeTimers();

        const mockRepo = {
            find: jest.fn().mockResolvedValue({ id: 1 }),
        };

        const cached = new CachingDecorator(mockRepo, 1000); // 1 সেকেন্ড TTL

        await cached.find(1); // fetch
        jest.advanceTimersByTime(1500); // TTL expired
        await cached.find(1); // re-fetch

        expect(mockRepo.find).toHaveBeenCalledTimes(2);

        jest.useRealTimers();
    });
});

describe('Order System', () => {
    it('calculates total with multiple add-ons', () => {
        let order = new BaseProduct('শার্ট', 500);
        order = new GiftWrapping(order);
        order = new ExpressDelivery(order);
        order = new VATDecorator(order, 0.075);

        // (500 + 50 + 100) * 1.075 = 698.75
        expect(order.price()).toBeCloseTo(698.75);
    });

    it('breakdown shows all line items', () => {
        let order = new BaseProduct('শার্ট', 500);
        order = new GiftWrapping(order);

        const items = order.breakdown();
        expect(items).toHaveLength(2);
        expect(items[0].item).toBe('শার্ট');
        expect(items[1].item).toBe('গিফট র‍্যাপিং');
    });
});
```

---

## 🔗 সম্পর্কিত প্যাটার্ন

### Decorator ↔ Adapter
| বিষয় | Decorator | Adapter |
|-------|-----------|---------|
| উদ্দেশ্য | নতুন behavior যোগ | Interface translate |
| Interface | একই রাখে | পরিবর্তন করে |
| Wrapping | বারবার stack হয় | সাধারণত একবার |
| উদাহরণ | Logger wrap | XML→JSON convert |

### Decorator ↔ Composite
- **Composite** = গাছের মতো structure (tree), সব node একই interface
- **Decorator** = single object wrap করে behavior যোগ করে
- দুটোই **recursive composition** ব্যবহার করে, কিন্তু উদ্দেশ্য ভিন্ন
- Decorator-কে "এক child-বিশিষ্ট Composite" ভাবতে পারেন

### Decorator ↔ Proxy
- **Proxy** = access control (lazy loading, security, remote proxy)
- **Decorator** = নতুন functionality যোগ
- দুটোই same interface implement করে এবং wrap করে
- Proxy সাধারণত object-এর **lifecycle** নিয়ন্ত্রণ করে, Decorator করে না

### Decorator ↔ Strategy
- **Strategy** = object-এর অভ্যন্তরীণ algorithm পরিবর্তন (internal)
- **Decorator** = object-এর বাইরের দিক থেকে behavior যোগ (external)
- Strategy হলো "ভেতর থেকে বদলানো", Decorator হলো "বাইরে থেকে মোড়ানো"

### Decorator ↔ Chain of Responsibility
- **Chain of Responsibility** = request একটি handler থেকে পরেরটিতে যায়, যেকোনো একজন handle করতে পারে
- **Decorator** = সব layer-ই request process করে এবং পরবর্তীতে pass করে
- Laravel Middleware = দুটোর মিশ্রণ (Decorator + CoR)

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন

1. **Runtime-এ behavior পরিবর্তন প্রয়োজন** — যেমন feature flags দিয়ে logging on/off
2. **Cross-cutting concerns** — logging, caching, auth, validation — এগুলো decorator হিসেবে চমৎকার কাজ করে
3. **Class modification সম্ভব নয়** — third-party library/sealed class extend করতে পারছেন না
4. **Combination explosion** — inheritance-এ অনেক subclass তৈরি হচ্ছে
5. **Single Responsibility মানতে হবে** — একটি class-এ অনেক responsibility জমা হচ্ছে
6. **Middleware pipeline** — HTTP middleware, message queue handlers
7. **Transparent wrapping** — client-এর জানার দরকার নেই কী wrap হচ্ছে

### ❌ ব্যবহার করবেন না

1. **সরল inheritance যথেষ্ট** — ২-৩টি fixed variant থাকলে decorator overkill
2. **Performance critical path** — অনেক layer-এর overhead acceptable না হলে
3. **Object identity গুরুত্বপূর্ণ** — `instanceof` check ভাঙবে
4. **State-heavy objects** — decorator-এ complex mutable state maintain করা কঠিন
5. **খুব ছোট project** — ২-৩ class-এর project-এ pattern ব্যবহার over-engineering
6. **Constructor arguments ভিন্ন** — যদি concrete component এবং decorator-এর constructor খুব আলাদা হয়

---

## 📋 সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────────────┐
│                  🎀 Decorator Pattern Summary                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ধরন:        Structural Pattern                                  │
│  উদ্দেশ্য:    Runtime-এ object-এ নতুন behavior যোগ করা            │
│  মূলনীতি:    Composition over Inheritance                        │
│                                                                  │
│  মূল উপাদান:                                                     │
│  ┌─────────────────┐  ┌──────────────────┐                       │
│  │ Component (I)    │  │ ConcreteComponent │                      │
│  │ (Coffee)         │  │ (BasicCoffee)     │                      │
│  └────────┬────────┘  └──────────────────┘                       │
│           │                                                      │
│  ┌────────┴────────┐                                             │
│  │ Decorator (A)    │  ← same interface + wrapped reference      │
│  │ (CoffeeDecorator)│                                            │
│  └────────┬────────┘                                             │
│           │                                                      │
│  ┌────────┴────────┐                                             │
│  │ ConcreteDecorator│  ← extra behavior যোগ করে                  │
│  │ (MilkDecorator)  │                                            │
│  └─────────────────┘                                             │
│                                                                  │
│  বাস্তব ব্যবহার:                                                 │
│  • Laravel/Express Middleware                                    │
│  • Repository Caching Layer                                      │
│  • Logging/Monitoring Wrapper                                    │
│  • Stream Processing Pipeline                                    │
│  • bKash API Middleware Chain                                    │
│  • E-commerce Order Add-ons                                      │
│                                                                  │
│  মনে রাখুন:                                                      │
│  ১. Decorator ক্রম গুরুত্বপূর্ণ (Logging → Cache → Service)      │
│  ২. Same interface — client জানে না কতগুলো layer আছে            │
│  ৩. ৫টির বেশি layer হলে Pipeline/CoR বিবেচনা করুন               │
│  ৪. প্রতিটি decorator আলাদাভাবে unit test করুন                  │
│  ৫. DI container দিয়ে decorator wiring করলে clean থাকে          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

> **"Favor object composition over class inheritance."** — Gang of Four

Decorator pattern সঠিকভাবে ব্যবহার করলে আপনার codebase হবে flexible, maintainable, এবং SOLID principles অনুসরণকারী। Laravel-এর middleware system এই pattern-এর সবচেয়ে সুন্দর real-world implementation — প্রতিদিন ব্যবহার করছেন, হয়তো জানতেনই না যে এটি Decorator pattern! 🎀
