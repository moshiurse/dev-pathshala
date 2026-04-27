# 🎭 মকিং ও স্টাবিং (Mocking & Stubbing) — গভীর বিশ্লেষণ

> **"ইউনিট টেস্ট মানে একটি ইউনিটকে তার সকল dependency থেকে বিচ্ছিন্ন করে পরীক্ষা করা। Test Doubles হলো সেই বিচ্ছিন্নতা অর্জনের হাতিয়ার।"**

---

## 📌 সংজ্ঞা — Test Doubles Taxonomy

Gerard Meszaros তাঁর *xUnit Test Patterns* বইতে **Test Double** শব্দটি প্রথম ব্যবহার করেন। সিনেমার "stunt double" থেকে ধারণাটি নেওয়া — যেমন একজন stunt double আসল অভিনেতার বদলে বিপজ্জনক দৃশ্যে কাজ করে, তেমনি Test Double আসল dependency-র বদলে টেস্টে কাজ করে।

Test Double-এর **পাঁচটি** মূল প্রকারভেদ:

| প্রকার | উদ্দেশ্য | আচরণ পরীক্ষা করে? | State পরীক্ষা করে? |
|--------|----------|-------------------|-------------------|
| **Dummy** | শুধু parameter পূরণ করতে | ❌ | ❌ |
| **Stub** | পূর্বনির্ধারিত মান ফেরত দেয় | ❌ | ✅ |
| **Spy** | interaction রেকর্ড করে | ✅ (পরে যাচাই) | ✅ |
| **Mock** | প্রত্যাশা পূর্বনির্ধারিত থাকে | ✅ (আগে থেকে সেট) | ❌ |
| **Fake** | সরলীকৃত কিন্তু কার্যকরী বাস্তবায়ন | ❌ | ✅ |

### মূল পার্থক্য: State Verification vs Behavior Verification

Martin Fowler এই দুটি পদ্ধতির মধ্যে গুরুত্বপূর্ণ পার্থক্য টানেন:

- **State Verification (Stub/Fake):** টেস্ট শেষে সিস্টেমের **অবস্থা** (state) পরীক্ষা করা হয়। "ফলাফল কী হলো?"
- **Behavior Verification (Mock/Spy):** সিস্টেম নির্দিষ্ট **পদ্ধতিতে** কাজ করেছে কিনা যাচাই করা হয়। "কীভাবে কাজটি হলো?"

---

## 📊 Test Double Relationship — সম্পর্ক চিত্র

```
                        ┌─────────────────────────┐
                        │      Test Double         │
                        │  (Generic Substitute)    │
                        └────────────┬────────────┘
                                     │
            ┌────────────┬───────────┼───────────┬────────────┐
            ▼            ▼           ▼           ▼            ▼
       ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
       │  Dummy  │ │  Stub   │ │   Spy   │ │  Mock   │ │  Fake   │
       │         │ │         │ │         │ │         │ │         │
       │ কোনো    │ │ নির্দিষ্ট│ │ কল      │ │পূর্ব-    │ │ সরলীকৃত │
       │ আচরণ   │ │ মান     │ │ রেকর্ড  │ │নির্ধারিত│ │ কার্যকরী │
       │ নেই    │ │ ফেরত   │ │ করে    │ │প্রত্যাশা│ │ বাস্তবায়ন│
       └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
            │            │           │           │            │
            ▼            ▼           ▼           ▼            ▼
       Parameter    Indirect      Indirect   Indirect    Lightweight
       Filling      Input         Output     Output      Implementation
                    Control       Verif.     Verif.

       ◄──── জটিলতা বৃদ্ধি (Increasing Complexity) ────────────────►
       ◄──── কম আচরণ ──────────────────────── বেশি আচরণ ──────────►
```

```
  ┌──────────────────── Verification Style ────────────────────┐
  │                                                             │
  │   State Verification          Behavior Verification         │
  │   ┌───────────────┐          ┌───────────────────┐         │
  │   │ Stub          │          │ Mock               │         │
  │   │ Fake          │          │ Spy                │         │
  │   └───────────────┘          └───────────────────┘         │
  │                                                             │
  │   "ফলাফল কী হলো?"           "কীভাবে কাজ করলো?"            │
  └─────────────────────────────────────────────────────────────┘
```

---

## 💻 প্রতিটি Test Double — গভীর বিশ্লেষণ

---

### ১. Dummy — শুধু জায়গা পূরণকারী

Dummy object শুধুমাত্র method signature বা constructor-এর parameter পূরণ করতে ব্যবহৃত হয়। এটি কখনোই আসলে ব্যবহৃত হয় না — যদি ব্যবহৃত হয়, তাহলে exception ছুড়বে।

**বাস্তব উদাহরণ:** ধরুন আপনি বিকাশ পেমেন্ট সিস্টেমের `TransactionLogger` টেস্ট করছেন। Logger-এর constructor-এ একটি `Formatter` দরকার, কিন্তু আপনি যে method টেস্ট করছেন সেটি Formatter ব্যবহার করে না।

#### PHP উদাহরণ

```php
<?php

interface NotificationChannel
{
    public function send(string $to, string $message): void;
}

class TransactionService
{
    public function __construct(
        private PaymentGateway $gateway,
        private NotificationChannel $notifier // এই parameter কিছু ক্ষেত্রে ব্যবহৃত হয় না
    ) {}

    public function calculateFee(float $amount): float
    {
        // notifier এখানে ব্যবহৃত হচ্ছে না
        return $amount * 0.015; // ১.৫% ফি
    }
}

class TransactionServiceTest extends \PHPUnit\Framework\TestCase
{
    public function test_fee_calculation_for_bkash_transaction(): void
    {
        // Dummy — শুধু constructor পূরণ করতে
        $dummyNotifier = $this->createMock(NotificationChannel::class);
        $dummyGateway = $this->createMock(PaymentGateway::class);

        $service = new TransactionService($dummyGateway, $dummyNotifier);

        // Dummy কখনো ব্যবহৃত হচ্ছে না — শুধু parameter পূরণ করেছে
        $this->assertEquals(15.0, $service->calculateFee(1000));
    }
}
```

#### JavaScript (Jest) উদাহরণ

```javascript
// dummy হিসেবে একটি empty object বা null-like object পাস করা
class TransactionService {
  constructor(gateway, notifier) {
    this.gateway = gateway;
    this.notifier = notifier;
  }

  calculateFee(amount) {
    return amount * 0.015;
  }
}

describe('TransactionService', () => {
  test('বিকাশ ট্রানজ্যাকশনের ফি সঠিকভাবে গণনা করে', () => {
    const dummyGateway = {}; // কখনো ব্যবহৃত হবে না
    const dummyNotifier = {}; // কখনো ব্যবহৃত হবে না

    const service = new TransactionService(dummyGateway, dummyNotifier);

    expect(service.calculateFee(1000)).toBe(15);
  });
});
```

---

### ২. Stub — পূর্বনির্ধারিত মান ফেরত দেয়

Stub এমন একটি Test Double যেটি **সবসময় পূর্বনির্ধারিত মান ফেরত দেয়**। SUT (System Under Test) কে নির্দিষ্ট indirect input দিতে Stub ব্যবহৃত হয়। এটি verification করে না — শুধু টেস্টকে একটি নির্দিষ্ট পথে পরিচালিত করে।

#### PHP (Mockery) উদাহরণ

```php
<?php

use Mockery;

interface BkashApiClient
{
    public function getBalance(string $accountNumber): float;
    public function getTransactionStatus(string $trxId): string;
}

class BkashPaymentService
{
    public function __construct(private BkashApiClient $client) {}

    public function hasInsufficientBalance(string $account, float $requiredAmount): bool
    {
        $balance = $this->client->getBalance($account);
        return $balance < $requiredAmount;
    }

    public function isTransactionSuccessful(string $trxId): bool
    {
        return $this->client->getTransactionStatus($trxId) === 'COMPLETED';
    }
}

class BkashPaymentServiceTest extends \PHPUnit\Framework\TestCase
{
    use \Mockery\Adapter\Phpunit\MockeryPHPUnitIntegration;

    public function test_detects_insufficient_balance(): void
    {
        // Stub — নির্দিষ্ট মান ফেরত দেবে, কিন্তু কতবার কল হলো যাচাই করবে না
        $stub = Mockery::mock(BkashApiClient::class);
        $stub->shouldReceive('getBalance')
             ->with('01712345678')
             ->andReturn(500.00); // সবসময় ৫০০ টাকা ফেরত দেবে

        $service = new BkashPaymentService($stub);

        $this->assertTrue($service->hasInsufficientBalance('01712345678', 1000.00));
        $this->assertFalse($service->hasInsufficientBalance('01712345678', 200.00));
    }

    public function test_transaction_status_check(): void
    {
        $stub = Mockery::mock(BkashApiClient::class);
        $stub->shouldReceive('getTransactionStatus')
             ->andReturn('COMPLETED');

        $service = new BkashPaymentService($stub);

        $this->assertTrue($service->isTransactionSuccessful('TRX123456'));
    }

    protected function tearDown(): void
    {
        Mockery::close();
    }
}
```

#### JavaScript (Jest) উদাহরণ

```javascript
// bkashApi.js
class BkashApiClient {
  async getBalance(accountNumber) { /* আসল API কল */ }
  async getTransactionStatus(trxId) { /* আসল API কল */ }
}

// bkashPaymentService.js
class BkashPaymentService {
  constructor(client) {
    this.client = client;
  }

  async hasInsufficientBalance(account, requiredAmount) {
    const balance = await this.client.getBalance(account);
    return balance < requiredAmount;
  }
}

// bkashPaymentService.test.js
describe('BkashPaymentService', () => {
  test('ব্যালেন্স কম থাকলে সনাক্ত করতে পারে', async () => {
    // jest.fn() দিয়ে Stub তৈরি
    const stubClient = {
      getBalance: jest.fn().mockResolvedValue(500.00),
      getTransactionStatus: jest.fn().mockResolvedValue('COMPLETED'),
    };

    const service = new BkashPaymentService(stubClient);

    await expect(service.hasInsufficientBalance('01712345678', 1000)).resolves.toBe(true);
    await expect(service.hasInsufficientBalance('01712345678', 200)).resolves.toBe(false);
  });

  test('stub বিভিন্ন কলে ভিন্ন ভিন্ন মান ফেরত দিতে পারে', async () => {
    const stubClient = {
      getBalance: jest.fn()
        .mockResolvedValueOnce(500)   // প্রথম কলে ৫০০
        .mockResolvedValueOnce(1500)  // দ্বিতীয় কলে ১৫০০
        .mockResolvedValue(0),        // পরবর্তী সব কলে ০
    };

    const service = new BkashPaymentService(stubClient);

    await expect(service.hasInsufficientBalance('017xxx', 1000)).resolves.toBe(true);  // 500 < 1000
    await expect(service.hasInsufficientBalance('017xxx', 1000)).resolves.toBe(false); // 1500 > 1000
    await expect(service.hasInsufficientBalance('017xxx', 1)).resolves.toBe(true);     // 0 < 1
  });
});
```

---

### ৩. Spy — interaction রেকর্ড করে, পরে যাচাই করা যায়

Spy মূলত Stub-এর মতোই কাজ করে, কিন্তু অতিরিক্তভাবে **কোন method কতবার, কোন argument দিয়ে কল হলো** তা রেকর্ড করে রাখে। পরে assertion-এ সেই তথ্য যাচাই করা যায়।

**Mock vs Spy-র মূল পার্থক্য:**
- **Mock:** প্রত্যাশা **আগে** সেট করা হয় → টেস্ট ব্যর্থ হলে Mock নিজেই ত্রুটি ছুড়ে দেয়
- **Spy:** আগে কিছু সেট করা হয় না → টেস্ট শেষে **আপনি নিজে** assertion লিখে যাচাই করেন

#### PHP (Mockery Spy) উদাহরণ

```php
<?php

interface SmsGateway
{
    public function send(string $to, string $message): bool;
}

class OtpService
{
    public function __construct(
        private SmsGateway $smsGateway,
        private OtpRepository $repository
    ) {}

    public function sendOtp(string $phoneNumber): string
    {
        $otp = str_pad((string) random_int(0, 999999), 6, '0', STR_PAD_LEFT);
        $this->repository->store($phoneNumber, $otp);
        $this->smsGateway->send($phoneNumber, "আপনার OTP: {$otp}");
        return $otp;
    }
}

class OtpServiceTest extends \PHPUnit\Framework\TestCase
{
    use \Mockery\Adapter\Phpunit\MockeryPHPUnitIntegration;

    public function test_otp_sends_sms_to_correct_number(): void
    {
        // Spy — রেকর্ড করবে, পরে আমরা যাচাই করবো
        $smsSpy = Mockery::spy(SmsGateway::class);
        $repoStub = Mockery::mock(OtpRepository::class);
        $repoStub->shouldReceive('store')->andReturn(true);

        $service = new OtpService($smsSpy, $repoStub);
        $otp = $service->sendOtp('01712345678');

        // পরে যাচাই — spy কী রেকর্ড করেছে?
        $smsSpy->shouldHaveReceived('send')
               ->once()
               ->with('01712345678', Mockery::pattern('/আপনার OTP: \d{6}/'));
    }

    public function test_otp_not_sent_if_repository_fails(): void
    {
        $smsSpy = Mockery::spy(SmsGateway::class);
        $repoStub = Mockery::mock(OtpRepository::class);
        $repoStub->shouldReceive('store')->andThrow(new \RuntimeException('DB error'));

        $service = new OtpService($smsSpy, $repoStub);

        try {
            $service->sendOtp('01712345678');
        } catch (\RuntimeException $e) {
            // SMS পাঠানো হয়নি — spy যাচাই
            $smsSpy->shouldNotHaveReceived('send');
        }
    }

    protected function tearDown(): void
    {
        Mockery::close();
    }
}
```

#### JavaScript (Jest spyOn) উদাহরণ

```javascript
// smsGateway.js
class SmsGateway {
  async send(to, message) {
    // আসল SMS API কল (যেমন: BulkSMSBD, Infobip)
  }
}

// otpService.js
class OtpService {
  constructor(smsGateway, repository) {
    this.smsGateway = smsGateway;
    this.repository = repository;
  }

  async sendOtp(phoneNumber) {
    const otp = String(Math.floor(Math.random() * 1000000)).padStart(6, '0');
    await this.repository.store(phoneNumber, otp);
    await this.smsGateway.send(phoneNumber, `আপনার OTP: ${otp}`);
    return otp;
  }
}

// otpService.test.js
describe('OtpService', () => {
  test('সঠিক নম্বরে SMS পাঠানো হয়', async () => {
    const gateway = new SmsGateway();
    // spyOn — আসল method-এ spy বসানো
    const sendSpy = jest.spyOn(gateway, 'send').mockResolvedValue(true);
    const repository = { store: jest.fn().mockResolvedValue(true) };

    const service = new OtpService(gateway, repository);
    const otp = await service.sendOtp('01712345678');

    // পরে যাচাই
    expect(sendSpy).toHaveBeenCalledTimes(1);
    expect(sendSpy).toHaveBeenCalledWith('01712345678', expect.stringContaining('আপনার OTP:'));

    sendSpy.mockRestore(); // spy পরিষ্কার করা
  });

  test('spy দিয়ে কল ক্রম (call order) যাচাই', async () => {
    const callOrder = [];
    const gateway = {
      send: jest.fn().mockImplementation(() => {
        callOrder.push('sms');
        return Promise.resolve(true);
      }),
    };
    const repository = {
      store: jest.fn().mockImplementation(() => {
        callOrder.push('store');
        return Promise.resolve(true);
      }),
    };

    const service = new OtpService(gateway, repository);
    await service.sendOtp('01712345678');

    // store আগে হওয়া উচিত, তারপর sms
    expect(callOrder).toEqual(['store', 'sms']);
  });
});
```

---

### ৪. Mock — পূর্বনির্ধারিত প্রত্যাশা সহ

Mock হলো সবচেয়ে "strict" Test Double। আপনি **টেস্ট চালানোর আগেই** বলে দেন কোন method, কতবার, কোন argument দিয়ে কল হতে হবে। প্রত্যাশা পূরণ না হলে Mock নিজেই টেস্ট ফেল করে।

#### PHP (Mockery) উদাহরণ

```php
<?php

interface PaymentProcessor
{
    public function charge(string $account, float $amount, string $currency): string;
    public function refund(string $transactionId): bool;
}

class OrderService
{
    public function __construct(
        private PaymentProcessor $processor,
        private OrderRepository $orders
    ) {}

    public function placeOrder(string $account, array $items): string
    {
        $total = array_sum(array_column($items, 'price'));
        $trxId = $this->processor->charge($account, $total, 'BDT');
        $this->orders->save(['account' => $account, 'trx_id' => $trxId, 'total' => $total]);
        return $trxId;
    }
}

class OrderServiceTest extends \PHPUnit\Framework\TestCase
{
    use \Mockery\Adapter\Phpunit\MockeryPHPUnitIntegration;

    public function test_order_charges_correct_amount_in_bdt(): void
    {
        // Mock — প্রত্যাশা আগেই সেট
        $mockProcessor = Mockery::mock(PaymentProcessor::class);
        $mockProcessor->shouldReceive('charge')
                      ->once()                              // ঠিক একবার কল হবে
                      ->with('01712345678', 350.00, 'BDT')  // এই arguments দিয়ে
                      ->andReturn('TRX_ABC123');             // এই মান ফেরত দেবে

        $mockOrders = Mockery::mock(OrderRepository::class);
        $mockOrders->shouldReceive('save')->once();

        $service = new OrderService($mockProcessor, $mockOrders);
        $items = [
            ['name' => 'চা', 'price' => 50.00],
            ['name' => 'সিঙ্গারা', 'price' => 100.00],
            ['name' => 'বিরিয়ানি', 'price' => 200.00],
        ];

        $trxId = $service->placeOrder('01712345678', $items);

        $this->assertEquals('TRX_ABC123', $trxId);
        // Mockery tearDown-এ স্বয়ংক্রিয়ভাবে প্রত্যাশা যাচাই করবে
    }

    protected function tearDown(): void
    {
        Mockery::close();
    }
}
```

#### JavaScript (Jest Mock) উদাহরণ

```javascript
// jest.mock() দিয়ে সম্পূর্ণ module mock করা
// paymentProcessor.js
export class PaymentProcessor {
  async charge(account, amount, currency) { /* API কল */ }
}

// orderService.test.js
jest.mock('./paymentProcessor');

describe('OrderService', () => {
  test('সঠিক পরিমাণ BDT-তে চার্জ করে', async () => {
    const mockCharge = jest.fn().mockResolvedValue('TRX_ABC123');
    const mockProcessor = { charge: mockCharge };
    const mockOrders = { save: jest.fn().mockResolvedValue(true) };

    const service = new OrderService(mockProcessor, mockOrders);
    const items = [
      { name: 'চা', price: 50 },
      { name: 'সিঙ্গারা', price: 100 },
      { name: 'বিরিয়ানি', price: 200 },
    ];

    const trxId = await service.placeOrder('01712345678', items);

    expect(trxId).toBe('TRX_ABC123');
    expect(mockCharge).toHaveBeenCalledWith('01712345678', 350, 'BDT');
    expect(mockCharge).toHaveBeenCalledTimes(1);
    expect(mockOrders.save).toHaveBeenCalledWith(
      expect.objectContaining({ trx_id: 'TRX_ABC123', total: 350 })
    );
  });
});
```

---

### ৫. Fake — সরলীকৃত কিন্তু কার্যকরী বাস্তবায়ন

Fake হলো production dependency-র একটি **সরলীকৃত কিন্তু কার্যকরী** বাস্তবায়ন। এটি mock/stub-এর মতো hardcoded মান ফেরত দেয় না — বরং সত্যিকারের logic আছে, শুধু সেটি production-ready নয় (যেমন: ডাটাবেসের বদলে in-memory array)।

#### PHP উদাহরণ — InMemoryRepository ও FakeMailer

```php
<?php

interface UserRepository
{
    public function save(User $user): void;
    public function findByPhone(string $phone): ?User;
    public function findAll(): array;
    public function delete(string $id): void;
}

// Fake — পুরোপুরি কার্যকরী, কিন্তু in-memory
class InMemoryUserRepository implements UserRepository
{
    private array $users = [];

    public function save(User $user): void
    {
        $this->users[$user->getId()] = $user;
    }

    public function findByPhone(string $phone): ?User
    {
        foreach ($this->users as $user) {
            if ($user->getPhone() === $phone) {
                return $user;
            }
        }
        return null;
    }

    public function findAll(): array
    {
        return array_values($this->users);
    }

    public function delete(string $id): void
    {
        unset($this->users[$id]);
    }
}

// Fake Mailer — আসলে ইমেইল পাঠায় না, কিন্তু রেকর্ড করে
class FakeMailer implements MailerInterface
{
    private array $sentMails = [];

    public function send(string $to, string $subject, string $body): void
    {
        $this->sentMails[] = compact('to', 'subject', 'body');
    }

    // টেস্ট হেল্পার methods
    public function getSentMails(): array
    {
        return $this->sentMails;
    }

    public function assertSentTo(string $email, string $subject): void
    {
        $found = array_filter($this->sentMails, fn($mail) =>
            $mail['to'] === $email && $mail['subject'] === $subject
        );

        if (empty($found)) {
            throw new \PHPUnit\Framework\AssertionFailedError(
                "Expected mail to {$email} with subject '{$subject}' was not sent."
            );
        }
    }

    public function assertNothingSent(): void
    {
        if (!empty($this->sentMails)) {
            throw new \PHPUnit\Framework\AssertionFailedError(
                count($this->sentMails) . ' mail(s) sent unexpectedly.'
            );
        }
    }
}

// ব্যবহার
class UserRegistrationTest extends \PHPUnit\Framework\TestCase
{
    public function test_registration_stores_user_and_sends_welcome_email(): void
    {
        $repo = new InMemoryUserRepository();
        $mailer = new FakeMailer();
        $service = new UserRegistrationService($repo, $mailer);

        $service->register('রহিম', '01712345678', 'rahim@example.com');

        // State verification — Fake-এর আসল state পরীক্ষা
        $user = $repo->findByPhone('01712345678');
        $this->assertNotNull($user);
        $this->assertEquals('রহিম', $user->getName());

        $mailer->assertSentTo('rahim@example.com', 'স্বাগতম!');
    }
}
```

#### JavaScript উদাহরণ — InMemoryDatabase

```javascript
// Fake InMemory Database
class InMemoryUserRepository {
  constructor() {
    this.users = new Map();
    this.nextId = 1;
  }

  async save(userData) {
    const id = String(this.nextId++);
    const user = { id, ...userData, createdAt: new Date() };
    this.users.set(id, user);
    return user;
  }

  async findByPhone(phone) {
    return [...this.users.values()].find(u => u.phone === phone) || null;
  }

  async findAll() {
    return [...this.users.values()];
  }

  async delete(id) {
    return this.users.delete(id);
  }

  // টেস্ট হেল্পার
  count() {
    return this.users.size;
  }

  clear() {
    this.users.clear();
    this.nextId = 1;
  }
}

// Fake bKash API — নিজস্ব logic আছে
class FakeBkashApi {
  constructor() {
    this.accounts = new Map();
    this.transactions = [];
  }

  setBalance(account, balance) {
    this.accounts.set(account, balance);
  }

  async charge(account, amount) {
    const balance = this.accounts.get(account) || 0;
    if (balance < amount) {
      throw new Error('Insufficient balance');
    }
    this.accounts.set(account, balance - amount);
    const trxId = `TRX_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
    this.transactions.push({ trxId, account, amount, type: 'charge' });
    return trxId;
  }

  getTransactions() {
    return [...this.transactions];
  }
}

describe('Fake bKash API', () => {
  test('ব্যালেন্স কম থাকলে charge ব্যর্থ হয়', async () => {
    const fakeBkash = new FakeBkashApi();
    fakeBkash.setBalance('01712345678', 500);

    await expect(fakeBkash.charge('01712345678', 1000))
      .rejects.toThrow('Insufficient balance');
  });

  test('সফল charge-এ ব্যালেন্স কমে যায়', async () => {
    const fakeBkash = new FakeBkashApi();
    fakeBkash.setBalance('01712345678', 1000);

    const trxId = await fakeBkash.charge('01712345678', 300);

    expect(trxId).toMatch(/^TRX_/);
    expect(fakeBkash.getTransactions()).toHaveLength(1);
  });
});
```

---

## 🔧 টুলস — বিস্তারিত ব্যবহার

---

### PHPUnit Built-in Mocking

PHPUnit-এর নিজস্ব mocking ফ্রেমওয়ার্ক বেশ শক্তিশালী। `createMock()` দিয়ে কোনো interface বা class-এর mock তৈরি করা যায়।

```php
<?php

class PHPUnitMockingExamplesTest extends \PHPUnit\Framework\TestCase
{
    // ১. createMock() — সবচেয়ে সাধারণ ব্যবহার
    public function test_basic_mock(): void
    {
        $mock = $this->createMock(BkashApiClient::class);

        $mock->method('getBalance')
             ->willReturn(5000.00);

        $this->assertEquals(5000.00, $mock->getBalance('01712345678'));
    }

    // ২. expects() দিয়ে কতবার কল হবে তা নির্ধারণ
    public function test_expects_invocation_count(): void
    {
        $mock = $this->createMock(SmsGateway::class);

        $mock->expects($this->once())      // ঠিক একবার
             ->method('send')
             ->with('01712345678', $this->stringContains('OTP'))
             ->willReturn(true);

        // $this->exactly(3)   — ঠিক ৩ বার
        // $this->atLeast(2)   — কমপক্ষে ২ বার
        // $this->atMost(5)    — সর্বোচ্চ ৫ বার
        // $this->never()      — কখনোই না
        // $this->any()        — যেকোনো সংখ্যকবার

        $mock->send('01712345678', 'আপনার OTP: 123456');
    }

    // ৩. with() — argument matching
    public function test_argument_constraints(): void
    {
        $mock = $this->createMock(PaymentProcessor::class);

        $mock->expects($this->once())
             ->method('charge')
             ->with(
                 $this->equalTo('01712345678'),                    // সঠিক মান
                 $this->greaterThan(0),                            // ০-এর বেশি
                 $this->logicalOr($this->equalTo('BDT'), $this->equalTo('USD'))
             )
             ->willReturn('TRX_123');

        $result = $mock->charge('01712345678', 500, 'BDT');
        $this->assertEquals('TRX_123', $result);
    }

    // ৪. willReturnCallback() — ডায়নামিক রিটার্ন
    public function test_dynamic_return(): void
    {
        $mock = $this->createMock(BkashApiClient::class);

        $mock->method('getBalance')
             ->willReturnCallback(function (string $account) {
                 return match ($account) {
                     '01712345678' => 5000.00,
                     '01812345678' => 12000.00,
                     default => 0.00,
                 };
             });

        $this->assertEquals(5000.00, $mock->getBalance('01712345678'));
        $this->assertEquals(12000.00, $mock->getBalance('01812345678'));
        $this->assertEquals(0.00, $mock->getBalance('01900000000'));
    }

    // ৫. willThrowException() — exception সিমুলেশন
    public function test_exception_simulation(): void
    {
        $mock = $this->createMock(BkashApiClient::class);

        $mock->method('getBalance')
             ->willThrowException(new \RuntimeException('API timeout'));

        $this->expectException(\RuntimeException::class);
        $mock->getBalance('01712345678');
    }

    // ৬. consecutive calls — পর পর ভিন্ন মান
    public function test_consecutive_returns(): void
    {
        $mock = $this->createMock(BkashApiClient::class);

        $mock->method('getTransactionStatus')
             ->willReturnOnConsecutiveCalls('PENDING', 'PENDING', 'COMPLETED');

        $this->assertEquals('PENDING', $mock->getTransactionStatus('TRX1'));
        $this->assertEquals('PENDING', $mock->getTransactionStatus('TRX1'));
        $this->assertEquals('COMPLETED', $mock->getTransactionStatus('TRX1'));
    }
}
```

---

### Mockery (PHP) — বিস্তারিত

Mockery, PHPUnit-এর মকিং-এর চেয়ে আরো fluent এবং expressive API দেয়।

```php
<?php

use Mockery;
use Mockery\Adapter\Phpunit\MockeryPHPUnitIntegration;

class MockeryAdvancedTest extends \PHPUnit\Framework\TestCase
{
    use MockeryPHPUnitIntegration;

    // ১. shouldReceive + andReturn
    public function test_basic_mockery(): void
    {
        $mock = Mockery::mock(BkashApiClient::class);
        $mock->shouldReceive('getBalance')->andReturn(5000.00);

        $this->assertEquals(5000.00, $mock->getBalance('any'));
    }

    // ২. Ordered expectations — নির্দিষ্ট ক্রমে কল হতে হবে
    public function test_ordered_calls(): void
    {
        $mock = Mockery::mock(PaymentProcessor::class);
        $mock->shouldReceive('charge')->once()->ordered();
        $mock->shouldReceive('refund')->never()->ordered();

        $mock->charge('01712345678', 500, 'BDT');
    }

    // ৩. Argument matching — Mockery matchers
    public function test_argument_matchers(): void
    {
        $mock = Mockery::mock(SmsGateway::class);
        $mock->shouldReceive('send')
             ->with(
                 Mockery::pattern('/^017\d{8}$/'),  // regex প্যাটার্ন
                 Mockery::type('string')             // যেকোনো string
             )
             ->andReturn(true);

        $this->assertTrue($mock->send('01712345678', 'Hello'));
    }

    // ৪. Spy — Mockery spy() ব্যবহার
    public function test_mockery_spy(): void
    {
        $spy = Mockery::spy(SmsGateway::class);

        // কোনো expectation সেট না করে আগে কল করি
        $spy->send('01712345678', 'Test');
        $spy->send('01812345678', 'Test 2');

        // পরে যাচাই
        $spy->shouldHaveReceived('send')->twice();
        $spy->shouldHaveReceived('send')->with('01712345678', 'Test');
    }

    // ৫. andReturnUsing — ডায়নামিক closure
    public function test_dynamic_return_with_closure(): void
    {
        $mock = Mockery::mock(BkashApiClient::class);
        $mock->shouldReceive('getBalance')
             ->andReturnUsing(fn(string $acc) => strlen($acc) === 11 ? 5000.0 : 0.0);

        $this->assertEquals(5000.0, $mock->getBalance('01712345678'));
        $this->assertEquals(0.0, $mock->getBalance('invalid'));
    }

    protected function tearDown(): void
    {
        Mockery::close();
    }
}
```

---

### Jest Mocking — বিস্তারিত

Jest-এর mocking ক্ষমতা অত্যন্ত শক্তিশালী এবং বিভিন্ন স্তরে কাজ করে।

```javascript
// ═══════════════════════════════════════════════
// ১. jest.fn() — standalone mock function
// ═══════════════════════════════════════════════

test('jest.fn() দিয়ে mock function তৈরি', () => {
  const mockFn = jest.fn();

  mockFn('arg1', 'arg2');
  mockFn('arg3');

  expect(mockFn).toHaveBeenCalledTimes(2);
  expect(mockFn).toHaveBeenNthCalledWith(1, 'arg1', 'arg2');
  expect(mockFn.mock.calls).toEqual([['arg1', 'arg2'], ['arg3']]);
});

test('mockImplementation দিয়ে custom logic', () => {
  const calculateFee = jest.fn().mockImplementation((amount) => {
    if (amount > 25000) return amount * 0.02;  // ২% (বড় amount)
    return amount * 0.015;                       // ১.৫% (ছোট amount)
  });

  expect(calculateFee(1000)).toBe(15);
  expect(calculateFee(50000)).toBe(1000);
});

// ═══════════════════════════════════════════════
// ২. jest.mock() — সম্পূর্ণ module mock
// ═══════════════════════════════════════════════

// __mocks__/bkashApi.js (manual mock)
const bkashApi = {
  getBalance: jest.fn().mockResolvedValue(5000),
  charge: jest.fn().mockResolvedValue('TRX_MOCK_123'),
};
module.exports = bkashApi;

// bkashService.test.js
jest.mock('./bkashApi'); // __mocks__/bkashApi.js ব্যবহার করবে
const bkashApi = require('./bkashApi');

test('module mock ব্যবহার', async () => {
  const balance = await bkashApi.getBalance('01712345678');
  expect(balance).toBe(5000);
});

// ═══════════════════════════════════════════════
// ৩. jest.spyOn() — existing method-এ spy বসানো
// ═══════════════════════════════════════════════

test('spyOn দিয়ে Date.now mock করা', () => {
  const fixedTime = new Date('2024-01-15T10:00:00Z').getTime();
  const spy = jest.spyOn(Date, 'now').mockReturnValue(fixedTime);

  expect(Date.now()).toBe(fixedTime);

  spy.mockRestore(); // আসল method ফিরিয়ে আনা
});

// ═══════════════════════════════════════════════
// ৪. mockResolvedValue / mockRejectedValue — async mock
// ═══════════════════════════════════════════════

test('async mock — সফল ও ব্যর্থ উভয় ক্ষেত্র', async () => {
  const fetchUser = jest.fn()
    .mockResolvedValueOnce({ id: 1, name: 'রহিম' })  // প্রথম কল: সফল
    .mockRejectedValueOnce(new Error('Not found'));    // দ্বিতীয় কল: ব্যর্থ

  const user = await fetchUser('01712345678');
  expect(user.name).toBe('রহিম');

  await expect(fetchUser('invalid')).rejects.toThrow('Not found');
});

// ═══════════════════════════════════════════════
// ৫. mock.calls ও mock.results — বিস্তারিত তথ্য
// ═══════════════════════════════════════════════

test('mock-এর বিস্তারিত তথ্য পরীক্ষা', () => {
  const mockFn = jest.fn((x) => x * 2);

  mockFn(5);
  mockFn(10);

  // কতবার কোন arguments দিয়ে কল হয়েছে
  expect(mockFn.mock.calls).toEqual([[5], [10]]);

  // প্রতিটি কলের return value
  expect(mockFn.mock.results).toEqual([
    { type: 'return', value: 10 },
    { type: 'return', value: 20 },
  ]);
});
```

---

### Laravel Test Helpers — Fake Facades

Laravel-এর Facade system মকিংকে অত্যন্ত সহজ করে। `::fake()` কল করলে আসল implementation-এর বদলে একটি fake ব্যবহৃত হয়।

```php
<?php

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Notification;

class LaravelFakeHelpersTest extends \Tests\TestCase
{
    // ═══════════════════════════════════════════════
    // Http::fake() — বিকাশ API কল সিমুলেশন
    // ═══════════════════════════════════════════════
    public function test_bkash_api_integration(): void
    {
        Http::fake([
            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/token/grant' => Http::response([
                'id_token' => 'fake_token_123',
                'token_type' => 'Bearer',
            ], 200),

            'tokenized.sandbox.bka.sh/v1.2.0-beta/tokenized/checkout/payment/create' => Http::response([
                'paymentID' => 'PAY_123',
                'bkashURL' => 'https://sandbox.payment.bkash.com?paymentId=PAY_123',
            ], 200),

            '*' => Http::response('Not Found', 404),
        ]);

        $service = app(BkashPaymentService::class);
        $result = $service->createPayment(500.00, '01712345678');

        $this->assertEquals('PAY_123', $result['paymentID']);

        Http::assertSent(function ($request) {
            return str_contains($request->url(), 'payment/create')
                && $request['amount'] === '500.00';
        });

        Http::assertSentCount(2);
    }

    // ═══════════════════════════════════════════════
    // Queue::fake() — job dispatch যাচাই
    // ═══════════════════════════════════════════════
    public function test_order_dispatches_payment_processing_job(): void
    {
        Queue::fake();

        $orderService = app(OrderService::class);
        $orderService->placeOrder([
            'user_id' => 1,
            'amount' => 500,
        ]);

        Queue::assertPushed(ProcessPaymentJob::class, function ($job) {
            return $job->amount === 500;
        });

        Queue::assertPushedOn('payments', ProcessPaymentJob::class);
        Queue::assertNotPushed(SendReceiptJob::class);
    }

    // ═══════════════════════════════════════════════
    // Mail::fake() — ইমেইল পাঠানো যাচাই
    // ═══════════════════════════════════════════════
    public function test_registration_sends_welcome_email(): void
    {
        Mail::fake();

        $service = app(UserRegistrationService::class);
        $service->register([
            'name' => 'করিম',
            'email' => 'karim@example.com',
        ]);

        Mail::assertSent(WelcomeMail::class, function ($mail) {
            return $mail->hasTo('karim@example.com');
        });
        Mail::assertSent(WelcomeMail::class, 1); // ঠিক ১টি
    }

    // ═══════════════════════════════════════════════
    // Event::fake() — event dispatch যাচাই
    // ═══════════════════════════════════════════════
    public function test_order_fires_events(): void
    {
        Event::fake([OrderPlaced::class, PaymentProcessed::class]);

        $service = app(OrderService::class);
        $service->placeOrder(/* ... */);

        Event::assertDispatched(OrderPlaced::class);
        Event::assertNotDispatched(PaymentProcessed::class);
    }

    // ═══════════════════════════════════════════════
    // Storage::fake() — ফাইল সিস্টেম সিমুলেশন
    // ═══════════════════════════════════════════════
    public function test_user_avatar_upload(): void
    {
        Storage::fake('public');

        $file = \Illuminate\Http\UploadedFile::fake()->image('avatar.jpg', 200, 200);

        $response = $this->postJson('/api/users/avatar', [
            'avatar' => $file,
        ]);

        $response->assertStatus(200);
        Storage::disk('public')->assertExists('avatars/' . $file->hashName());
    }

    // ═══════════════════════════════════════════════
    // Notification::fake() — notification যাচাই
    // ═══════════════════════════════════════════════
    public function test_otp_notification_sent(): void
    {
        Notification::fake();

        $user = User::factory()->create(['phone' => '01712345678']);
        $service = app(OtpService::class);
        $service->sendOtp($user);

        Notification::assertSentTo($user, OtpNotification::class, function ($notification) {
            return strlen($notification->otp) === 6;
        });
    }
}
```

---

## 🔥 Advanced টপিকস

---

### ১. HTTP কল মকিং — বিকাশ API ও Node.js nock

বাস্তব প্রজেক্টে সবচেয়ে বেশি mock করা হয় external HTTP API কল। বাংলাদেশের প্রেক্ষাপটে বিকাশ, নগদ, SSL Commerz-এর মতো payment gateway-র API call mock করা অত্যন্ত গুরুত্বপূর্ণ।

#### PHP — Laravel Http::fake() sequence ব্যবহার

```php
<?php

class BkashTokenFlowTest extends \Tests\TestCase
{
    public function test_token_refresh_on_expiry(): void
    {
        Http::fake([
            '*/token/grant' => Http::sequence()
                ->push(['id_token' => 'token_1', 'expires_in' => 3600], 200)
                ->push(['id_token' => 'token_2', 'expires_in' => 3600], 200),

            '*/payment/create' => Http::sequence()
                ->push(['error' => 'Token expired'], 401)  // প্রথমবার: expired
                ->push(['paymentID' => 'PAY_456'], 200),   // retry-তে সফল
        ]);

        $service = app(BkashPaymentService::class);
        $result = $service->createPaymentWithRetry(1000.00, '01712345678');

        $this->assertEquals('PAY_456', $result['paymentID']);

        // মোট ৪টি HTTP কল হওয়া উচিত:
        // token_grant → payment_create (401) → token_grant → payment_create (200)
        Http::assertSentCount(4);
    }

    public function test_handles_bkash_api_timeout(): void
    {
        Http::fake([
            '*/payment/create' => function () {
                throw new \Illuminate\Http\Client\ConnectionException('Connection timed out');
            },
        ]);

        $service = app(BkashPaymentService::class);

        $this->expectException(PaymentGatewayException::class);
        $this->expectExceptionMessage('বিকাশ সার্ভার অনুপলব্ধ');
        $service->createPayment(500.00, '01712345678');
    }
}
```

#### JavaScript — nock দিয়ে HTTP মকিং

```javascript
const nock = require('nock');
const { BkashClient } = require('./bkashClient');

describe('bKash API Client', () => {
  afterEach(() => {
    nock.cleanAll();
  });

  test('সফল পেমেন্ট তৈরি', async () => {
    // bKash sandbox API mock
    const scope = nock('https://tokenized.sandbox.bka.sh')
      .post('/v1.2.0-beta/tokenized/checkout/token/grant')
      .reply(200, {
        id_token: 'fake_token',
        token_type: 'Bearer',
      })
      .post('/v1.2.0-beta/tokenized/checkout/payment/create')
      .reply(200, {
        paymentID: 'PAY_789',
        bkashURL: 'https://sandbox.payment.bkash.com?paymentId=PAY_789',
      });

    const client = new BkashClient({
      appKey: 'test_key',
      appSecret: 'test_secret',
    });

    const result = await client.createPayment(500, '01712345678');

    expect(result.paymentID).toBe('PAY_789');
    expect(scope.isDone()).toBe(true); // সব mock endpoint কল হয়েছে
  });

  test('নেটওয়ার্ক ত্রুটি হ্যান্ডলিং', async () => {
    nock('https://tokenized.sandbox.bka.sh')
      .post('/v1.2.0-beta/tokenized/checkout/token/grant')
      .replyWithError('ETIMEDOUT');

    const client = new BkashClient({ appKey: 'test', appSecret: 'test' });

    await expect(client.createPayment(500, '01712345678'))
      .rejects.toThrow('ETIMEDOUT');
  });

  test('retry logic — ৩ বার চেষ্টা করে', async () => {
    const scope = nock('https://tokenized.sandbox.bka.sh')
      .post('/v1.2.0-beta/tokenized/checkout/token/grant')
      .times(3) // ৩ বার ব্যর্থ
      .reply(503, { message: 'Service Unavailable' });

    const client = new BkashClient({ appKey: 'test', appSecret: 'test', maxRetries: 3 });

    await expect(client.createPayment(500, '01712345678'))
      .rejects.toThrow();

    expect(scope.isDone()).toBe(true);
  });
});
```

---

### ২. Database Mocking — Repository Pattern

ডাটাবেস mock করার সবচেয়ে কার্যকর উপায় হলো Repository Pattern ব্যবহার করা। Interface-এর বিপরীতে প্রোগ্রাম করলে সহজেই InMemory বাস্তবায়ন দিয়ে আসল database প্রতিস্থাপন করা যায়।

```php
<?php

interface ProductRepository
{
    public function findById(string $id): ?Product;
    public function findByCategory(string $category): array;
    public function save(Product $product): void;
    public function search(string $query): array;
}

// Production — Eloquent ব্যবহার করে
class EloquentProductRepository implements ProductRepository
{
    public function findById(string $id): ?Product
    {
        return ProductModel::find($id)?->toDomainEntity();
    }
    // ...
}

// Test — In-Memory বাস্তবায়ন
class InMemoryProductRepository implements ProductRepository
{
    private array $products = [];

    public function findById(string $id): ?Product
    {
        return $this->products[$id] ?? null;
    }

    public function findByCategory(string $category): array
    {
        return array_filter($this->products, fn(Product $p) => $p->getCategory() === $category);
    }

    public function save(Product $product): void
    {
        $this->products[$product->getId()] = $product;
    }

    public function search(string $query): array
    {
        return array_filter($this->products, fn(Product $p) =>
            str_contains(strtolower($p->getName()), strtolower($query))
        );
    }

    // টেস্ট হেল্পার — একাধিক product একসাথে যোগ
    public function seedWith(array $products): void
    {
        foreach ($products as $product) {
            $this->save($product);
        }
    }
}

class ProductSearchTest extends \PHPUnit\Framework\TestCase
{
    private InMemoryProductRepository $repo;
    private ProductSearchService $service;

    protected function setUp(): void
    {
        $this->repo = new InMemoryProductRepository();
        $this->service = new ProductSearchService($this->repo);

        $this->repo->seedWith([
            new Product('1', 'বিরিয়ানি মসলা', 'মসলা', 120),
            new Product('2', 'চিংড়ি মাছ', 'মাছ', 800),
            new Product('3', 'বাসমতি চাল', 'চাল', 350),
            new Product('4', 'মসলা চা', 'পানীয়', 200),
        ]);
    }

    public function test_search_finds_matching_products(): void
    {
        $results = $this->service->search('মসলা');

        $this->assertCount(2, $results); // 'বিরিয়ানি মসলা' ও 'মসলা চা'
    }

    public function test_search_returns_empty_for_no_match(): void
    {
        $results = $this->service->search('ল্যাপটপ');

        $this->assertEmpty($results);
    }
}
```

---

### ৩. সময় (Time) মকিং

সময়-নির্ভর কোড টেস্ট করা কঠিন — কারণ `now()` প্রতিবার ভিন্ন মান দেয়। সময় freeze করে এই সমস্যা সমাধান করা যায়।

#### PHP — Carbon::setTestNow()

```php
<?php

use Carbon\Carbon;

class SubscriptionService
{
    public function isExpired(Subscription $subscription): bool
    {
        return Carbon::now()->isAfter($subscription->getExpiresAt());
    }

    public function getDaysRemaining(Subscription $subscription): int
    {
        return max(0, Carbon::now()->diffInDays($subscription->getExpiresAt(), false));
    }
}

class SubscriptionServiceTest extends \PHPUnit\Framework\TestCase
{
    protected function tearDown(): void
    {
        Carbon::setTestNow(); // সময় reset
    }

    public function test_subscription_not_expired_before_date(): void
    {
        Carbon::setTestNow('2024-06-15 10:00:00'); // সময় freeze

        $subscription = new Subscription(expiresAt: Carbon::parse('2024-07-15'));
        $service = new SubscriptionService();

        $this->assertFalse($service->isExpired($subscription));
        $this->assertEquals(30, $service->getDaysRemaining($subscription));
    }

    public function test_subscription_expired_after_date(): void
    {
        Carbon::setTestNow('2024-08-01 00:00:00');

        $subscription = new Subscription(expiresAt: Carbon::parse('2024-07-15'));
        $service = new SubscriptionService();

        $this->assertTrue($service->isExpired($subscription));
        $this->assertEquals(0, $service->getDaysRemaining($subscription));
    }

    public function test_time_dependent_discount(): void
    {
        // রমজান মাসে বিশেষ ছাড়
        Carbon::setTestNow('2024-03-15'); // রমজান ২০২৪

        $service = new DiscountService();

        $this->assertTrue($service->isRamadanDiscount());
    }
}
```

#### JavaScript — jest.useFakeTimers()

```javascript
class TokenManager {
  constructor() {
    this.tokens = new Map();
  }

  createToken(userId, ttlMs = 3600000) { // ১ ঘণ্টা
    const token = `tok_${Math.random().toString(36).slice(2)}`;
    this.tokens.set(token, {
      userId,
      expiresAt: Date.now() + ttlMs,
    });
    return token;
  }

  isValid(token) {
    const data = this.tokens.get(token);
    if (!data) return false;
    return Date.now() < data.expiresAt;
  }
}

describe('TokenManager — সময় মকিং', () => {
  beforeEach(() => {
    jest.useFakeTimers();
    jest.setSystemTime(new Date('2024-06-15T10:00:00Z'));
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  test('নতুন token valid থাকে', () => {
    const manager = new TokenManager();
    const token = manager.createToken('user_1');

    expect(manager.isValid(token)).toBe(true);
  });

  test('মেয়াদ উত্তীর্ণ হলে token invalid হয়', () => {
    const manager = new TokenManager();
    const token = manager.createToken('user_1', 60000); // ১ মিনিট

    // ২ মিনিট পরের সময়ে যাই
    jest.advanceTimersByTime(120000);

    expect(manager.isValid(token)).toBe(false);
  });

  test('setTimeout/setInterval mock', () => {
    const callback = jest.fn();

    setTimeout(callback, 5000); // ৫ সেকেন্ড পরে

    expect(callback).not.toHaveBeenCalled();

    jest.advanceTimersByTime(5000);

    expect(callback).toHaveBeenCalledTimes(1);
  });
});
```

---

### ৪. Partial Mocks — শুধু নির্দিষ্ট methods mock করা

কখনো কখনো পুরো class mock করা দরকার হয় না — শুধু একটি বা দুটি method mock করলেই চলে। বাকি methods আসল implementation ব্যবহার করে।

```php
<?php

class ReportService
{
    public function generateMonthlyReport(int $month, int $year): array
    {
        $data = $this->fetchDataFromDatabase($month, $year);
        return $this->formatReport($data);
    }

    protected function fetchDataFromDatabase(int $month, int $year): array
    {
        // ভারী ডাটাবেস query — এটা mock করতে চাই
        return DB::table('transactions')
            ->whereMonth('created_at', $month)
            ->whereYear('created_at', $year)
            ->get()
            ->toArray();
    }

    public function formatReport(array $data): array
    {
        // এটা আসল logic — mock করব না
        $total = array_sum(array_column($data, 'amount'));
        return [
            'total' => $total,
            'count' => count($data),
            'average' => count($data) > 0 ? $total / count($data) : 0,
        ];
    }
}

class ReportServiceTest extends \PHPUnit\Framework\TestCase
{
    public function test_monthly_report_calculation(): void
    {
        // Partial Mock — শুধু fetchDataFromDatabase mock করা
        $service = Mockery::mock(ReportService::class)->makePartial();
        $service->shouldReceive('fetchDataFromDatabase')
                ->with(6, 2024)
                ->andReturn([
                    ['amount' => 5000],
                    ['amount' => 3000],
                    ['amount' => 7000],
                ]);

        $report = $service->generateMonthlyReport(6, 2024);

        $this->assertEquals(15000, $report['total']);
        $this->assertEquals(3, $report['count']);
        $this->assertEquals(5000, $report['average']);
    }
}
```

```javascript
// JavaScript — Partial Mock
class NotificationService {
  async sendAll(userId, message) {
    const [smsResult, emailResult] = await Promise.all([
      this.sendSms(userId, message),
      this.sendEmail(userId, message),
    ]);
    return { sms: smsResult, email: emailResult };
  }

  async sendSms(userId, message) {
    // আসল SMS API কল
  }

  async sendEmail(userId, message) {
    // আসল Email API কল
  }
}

describe('Partial Mock', () => {
  test('শুধু SMS mock করে email আসল রাখা', async () => {
    const service = new NotificationService();

    // শুধু sendSms mock — sendEmail আসল থাকবে
    jest.spyOn(service, 'sendSms').mockResolvedValue({ sent: true });
    jest.spyOn(service, 'sendEmail').mockResolvedValue({ sent: true });

    const result = await service.sendAll('user_1', 'Hello');

    expect(service.sendSms).toHaveBeenCalledWith('user_1', 'Hello');
    expect(result.sms).toEqual({ sent: true });
  });
});
```

---

### ৫. Static Methods / Facades মকিং

Static method mock করা কঠিন কারণ এগুলো class-level-এ কাজ করে, instance-level-এ নয়। Laravel Facade এই সমস্যার চমৎকার সমাধান দেয়।

```php
<?php

// Laravel Facade মকিং
class PaymentControllerTest extends \Tests\TestCase
{
    public function test_payment_gateway_facade(): void
    {
        // Facade::shouldReceive() — static call mock করে
        PaymentGateway::shouldReceive('charge')
            ->once()
            ->with('01712345678', 500.00, 'BDT')
            ->andReturn(['trx_id' => 'TRX_123', 'status' => 'success']);

        $response = $this->postJson('/api/payments', [
            'account' => '01712345678',
            'amount' => 500.00,
            'currency' => 'BDT',
        ]);

        $response->assertStatus(200)
                 ->assertJson(['trx_id' => 'TRX_123']);
    }

    // Cache Facade mock
    public function test_cached_exchange_rate(): void
    {
        Cache::shouldReceive('remember')
             ->once()
             ->andReturn(110.50); // ১ USD = ১১০.৫০ BDT

        $service = app(ExchangeRateService::class);
        $rate = $service->getUsdToBdtRate();

        $this->assertEquals(110.50, $rate);
    }
}
```

---

### ৬. File System মকিং

ফাইল সিস্টেম mock করা গুরুত্বপূর্ণ — কারণ আসল ফাইল তৈরি/মুছে ফেলা টেস্টকে flaky করে তোলে।

```php
<?php

// Laravel Storage::fake()
class CsvExportTest extends \Tests\TestCase
{
    public function test_transaction_csv_export(): void
    {
        Storage::fake('local');

        $service = app(TransactionExportService::class);
        $filePath = $service->exportToCsv([
            ['trx_id' => 'TRX_1', 'amount' => 500, 'date' => '2024-06-15'],
            ['trx_id' => 'TRX_2', 'amount' => 1000, 'date' => '2024-06-16'],
        ]);

        Storage::disk('local')->assertExists($filePath);
        $content = Storage::disk('local')->get($filePath);
        $this->assertStringContainsString('TRX_1', $content);
        $this->assertStringContainsString('TRX_2', $content);
    }
}
```

```javascript
// Jest — fs module mock
jest.mock('fs/promises');
const fs = require('fs/promises');

describe('ConfigLoader', () => {
  test('config ফাইল সঠিকভাবে পড়ে', async () => {
    const fakeConfig = JSON.stringify({
      bkash: { appKey: 'test_key', appSecret: 'test_secret' },
      database: { host: 'localhost' },
    });

    fs.readFile.mockResolvedValue(fakeConfig);

    const loader = new ConfigLoader();
    const config = await loader.load('/etc/app/config.json');

    expect(config.bkash.appKey).toBe('test_key');
    expect(fs.readFile).toHaveBeenCalledWith('/etc/app/config.json', 'utf-8');
  });

  test('ফাইল না পাওয়া গেলে default config ব্যবহার', async () => {
    fs.readFile.mockRejectedValue(new Error('ENOENT: no such file'));

    const loader = new ConfigLoader();
    const config = await loader.load('/missing/config.json');

    expect(config).toEqual(ConfigLoader.DEFAULT_CONFIG);
  });
});
```

---

### ৭. কখন Mock করবেন না — Over-Mocking Anti-pattern

> **"সবকিছু mock করলে আপনি আসলে mock-ই টেস্ট করছেন, production কোড নয়।"**

#### যেখানে Mock করা উচিত নয়:

```
  ┌─────────────────────────────────────────────────────────┐
  │           ❌ Mock করবেন না এগুলো:                       │
  │                                                         │
  │  ● Value Objects (Money, DateRange, Address)            │
  │  ● Pure functions (কোনো side effect নেই)               │
  │  ● Data structures (Array, List, Map)                   │
  │  ● আপনার নিজের domain logic                            │
  │  ● Entity/Model-এর business rules                      │
  │  ● Simple DTOs/POJOs                                   │
  │                                                         │
  ├─────────────────────────────────────────────────────────┤
  │           ✅ Mock করুন এগুলো:                           │
  │                                                         │
  │  ● External API কল (বিকাশ, নগদ, SMS)                  │
  │  ● Database query (Repository interface দিয়ে)          │
  │  ● File system I/O                                      │
  │  ● Network calls                                       │
  │  ● System clock/time                                    │
  │  ● Random number generation                            │
  │  ● Email/SMS পাঠানো                                     │
  │  ● Third-party library-র side effects                   │
  └─────────────────────────────────────────────────────────┘
```

#### Over-Mocking উদাহরণ — ভুল পদ্ধতি

```php
<?php

// ❌ ভুল — Value Object mock করা অর্থহীন
public function test_wrong_way(): void
{
    $mockMoney = Mockery::mock(Money::class);
    $mockMoney->shouldReceive('getAmount')->andReturn(500);
    $mockMoney->shouldReceive('getCurrency')->andReturn('BDT');
    $mockMoney->shouldReceive('add')->andReturn($mockMoney);

    // এখানে আপনি Money class-ই টেস্ট করছেন না, mock টেস্ট করছেন!
}

// ✅ সঠিক — আসল Value Object ব্যবহার করুন
public function test_right_way(): void
{
    $money = new Money(500, 'BDT');
    $total = $money->add(new Money(200, 'BDT'));

    $this->assertEquals(700, $total->getAmount());
    $this->assertEquals('BDT', $total->getCurrency());
}
```

---

### ৮. Mock vs Real — Test Confidence Spectrum

```
  Low Confidence                                    High Confidence
  (দ্রুত, সস্তা)                                  (ধীর, ব্যয়বহুল)
  ◄──────────────────────────────────────────────────────────────►

  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ সব Mock  │  │ বেশিরভাগ │  │ মিশ্র    │  │ বেশিরভাগ │  │ সব Real  │
  │          │  │ Mock     │  │ (Balanced)│  │ Real     │  │          │
  │ Unit     │  │          │  │          │  │          │  │ E2E      │
  │ Tests    │  │          │  │Integration│  │          │  │ Tests    │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
       │                           │                           │
       ▼                           ▼                           ▼
  দ্রুত ফিডব্যাক              সামঞ্জস্যপূর্ণ            আসল পরিবেশে আত্মবিশ্বাস
  কিন্তু ভুয়া আত্মবিশ্বাস    গতি ও আত্মবিশ্বাস         কিন্তু ধীর ও ভঙ্গুর
```

**আদর্শ Test Pyramid:**

```
            ╱╲
           ╱  ╲         E2E (কম সংখ্যক, সব Real)
          ╱    ╲        আসল বিকাশ sandbox, আসল DB
         ╱──────╲
        ╱        ╲      Integration (মাঝারি সংখ্যক)
       ╱          ╲     কিছু Real, কিছু Fake
      ╱────────────╲
     ╱              ╲   Unit (বেশি সংখ্যক, Mock/Stub)
    ╱                ╲  দ্রুত, বিচ্ছিন্ন
   ╱──────────────────╲
```

---

### ৯. Contract Tests — Mock বাস্তবের সাথে মিলছে কিনা যাচাই

**সবচেয়ে বড় ঝুঁকি:** আপনার mock আসল API-র behavior ঠিকভাবে অনুকরণ করছে না। API বদলে গেলে mock আপডেট না হলে টেস্ট পাস করবে কিন্তু production-এ ভাঙবে।

**Contract Test** নিশ্চিত করে যে আপনার mock এবং আসল implementation **একই interface contract** মেনে চলে।

```php
<?php

// Contract — উভয় implementation-ই এই টেস্টগুলো পাস করবে
abstract class UserRepositoryContractTest extends \PHPUnit\Framework\TestCase
{
    abstract protected function createRepository(): UserRepository;

    public function test_can_save_and_retrieve_user(): void
    {
        $repo = $this->createRepository();
        $user = new User('1', 'রহিম', '01712345678');

        $repo->save($user);

        $found = $repo->findByPhone('01712345678');
        $this->assertNotNull($found);
        $this->assertEquals('রহিম', $found->getName());
    }

    public function test_returns_null_for_missing_user(): void
    {
        $repo = $this->createRepository();

        $found = $repo->findByPhone('01700000000');
        $this->assertNull($found);
    }

    public function test_delete_removes_user(): void
    {
        $repo = $this->createRepository();
        $user = new User('1', 'রহিম', '01712345678');
        $repo->save($user);

        $repo->delete('1');

        $this->assertNull($repo->findByPhone('01712345678'));
    }
}

// InMemory Fake — এই contract পাস করে
class InMemoryUserRepositoryTest extends UserRepositoryContractTest
{
    protected function createRepository(): UserRepository
    {
        return new InMemoryUserRepository();
    }
}

// Eloquent — এই contract-ও পাস করে (integration test হিসেবে)
class EloquentUserRepositoryTest extends UserRepositoryContractTest
{
    use \Illuminate\Foundation\Testing\RefreshDatabase;

    protected function createRepository(): UserRepository
    {
        return new EloquentUserRepository();
    }
}
```

```javascript
// JavaScript — Contract test pattern
function runRepositoryContract(createRepo) {
  describe('Repository Contract', () => {
    let repo;

    beforeEach(async () => {
      repo = await createRepo();
    });

    test('save ও retrieve কাজ করে', async () => {
      await repo.save({ id: '1', name: 'রহিম', phone: '01712345678' });
      const found = await repo.findByPhone('01712345678');
      expect(found.name).toBe('রহিম');
    });

    test('নেই এমন user-এ null ফেরত দেয়', async () => {
      const found = await repo.findByPhone('01700000000');
      expect(found).toBeNull();
    });
  });
}

// দুটি ভিন্ন implementation, একই contract
describe('InMemoryUserRepository', () => {
  runRepositoryContract(() => new InMemoryUserRepository());
});

describe('MongoUserRepository', () => {
  runRepositoryContract(async () => {
    const repo = new MongoUserRepository(testDbConnection);
    await repo.clear();
    return repo;
  });
});
```

---

## ✅ Best Practices — সর্বোত্তম অনুশীলন

### ১. Arrange-Act-Assert (AAA) প্যাটার্ন অনুসরণ করুন

```php
public function test_payment_processing(): void
{
    // Arrange — প্রস্তুতি
    $mock = Mockery::mock(PaymentGateway::class);
    $mock->shouldReceive('charge')->andReturn('TRX_123');
    $service = new PaymentService($mock);

    // Act — কাজ সম্পাদন
    $result = $service->processPayment('01712345678', 500);

    // Assert — যাচাই
    $this->assertEquals('TRX_123', $result->getTransactionId());
}
```

### ২. Interface-এর বিপরীতে Mock করুন, Concrete Class-এর বিপরীতে নয়

```php
// ✅ সঠিক
$mock = Mockery::mock(PaymentGatewayInterface::class);

// ❌ ভুল — concrete class mock করলে internal implementation-এ coupling হয়
$mock = Mockery::mock(BkashPaymentGateway::class);
```

### ৩. প্রতিটি টেস্টে একটি মাত্র বিষয় যাচাই করুন

```php
// ✅ সঠিক — একটি দিক যাচাই
public function test_insufficient_balance_returns_error(): void { /* ... */ }
public function test_successful_payment_returns_trx_id(): void { /* ... */ }

// ❌ ভুল — অনেক কিছু একসাথে যাচাই
public function test_payment(): void
{
    // balance check, charge, notification, logging... সব একসাথে
}
```

### ৪. Test Setup-এ Helper Method ব্যবহার করুন

```php
class PaymentServiceTest extends TestCase
{
    private function createServiceWithMockedGateway(
        float $balance = 5000.00,
        string $trxId = 'TRX_DEFAULT'
    ): PaymentService {
        $gateway = Mockery::mock(PaymentGatewayInterface::class);
        $gateway->shouldReceive('getBalance')->andReturn($balance);
        $gateway->shouldReceive('charge')->andReturn($trxId);

        return new PaymentService($gateway);
    }

    public function test_example(): void
    {
        $service = $this->createServiceWithMockedGateway(balance: 500.00);
        // ...
    }
}
```

### ৫. Mock পরিষ্কার করুন

```php
// PHP Mockery
protected function tearDown(): void
{
    Mockery::close();
}
```

```javascript
// Jest
afterEach(() => {
  jest.restoreAllMocks();
  nock.cleanAll();
});
```

### ৬. Behavior টেস্ট করুন, Implementation নয়

```php
// ✅ সঠিক — behavior যাচাই
$this->assertTrue($service->hasInsufficientBalance('017xxx', 1000));

// ❌ ভুল — implementation detail যাচাই (এটা ভেঙে যাবে যদি internal logic বদলায়)
$mock->shouldHaveReceived('getBalance')->once();
$mock->shouldHaveReceived('calculateFee')->once();
$mock->shouldHaveReceived('checkMinimumBalance')->once();
```

---

## ⚠️ Anti-Patterns — যা করবেন না

### ১. সবকিছু Mock করা (Over-Mocking)

```php
// ❌ অতিরিক্ত Mock — আপনি কোনো আসল কোডই টেস্ট করছেন না
public function test_over_mocked(): void
{
    $mockUser = Mockery::mock(User::class);
    $mockUser->shouldReceive('getName')->andReturn('রহিম');

    $mockRepo = Mockery::mock(UserRepository::class);
    $mockRepo->shouldReceive('find')->andReturn($mockUser);

    $mockFormatter = Mockery::mock(NameFormatter::class);
    $mockFormatter->shouldReceive('format')->andReturn('জনাব রহিম');

    // আসলে কিছুই টেস্ট হচ্ছে না — শুধু mock-দের wire করেছেন
}
```

### ২. Implementation Details টেস্ট করা

```php
// ❌ ভুল — internal method কল টেস্ট করা
$service->shouldHaveReceived('validateInput')->once();
$service->shouldHaveReceived('sanitizeData')->once();
$service->shouldHaveReceived('transformResult')->once();

// ✅ সঠিক — observable behavior টেস্ট করা
$this->assertEquals($expected, $service->processOrder($input));
```

### ৩. Mock-এ Fragile Matching

```php
// ❌ ভুল — খুব নির্দিষ্ট argument matching
$mock->shouldReceive('send')
     ->with('01712345678', 'আপনার OTP: 123456, মেয়াদ: ৫ মিনিট, তারিখ: ২০২৪-০৬-১৫');

// ✅ সঠিক — গুরুত্বপূর্ণ অংশ যাচাই
$mock->shouldReceive('send')
     ->with('01712345678', Mockery::pattern('/OTP: \d{6}/'));
```

### ৪. God Mock — একটি Mock অনেক কাজ করে

```php
// ❌ ভুল — একটি mock-এ ১০টি method setup
$mock = Mockery::mock(EverythingService::class);
$mock->shouldReceive('getUser')->andReturn(/*...*/);
$mock->shouldReceive('getBalance')->andReturn(/*...*/);
$mock->shouldReceive('getTransactions')->andReturn(/*...*/);
// ... আরো ১০টি method

// ✅ সঠিক — Interface Segregation + প্রতিটি dependency আলাদা mock
$userRepo = Mockery::mock(UserRepository::class);
$balanceService = Mockery::mock(BalanceService::class);
```

### ৫. টেস্টে Production Logic রাখা

```javascript
// ❌ ভুল — টেস্টে production logic পুনরায় লেখা
test('ফি সঠিকভাবে গণনা হয়', () => {
  const amount = 1000;
  const fee = amount * 0.015; // production logic এখানে repeat হচ্ছে!

  expect(service.calculateFee(amount)).toBe(fee);
});

// ✅ সঠিক — expected value সরাসরি লেখা
test('ফি সঠিকভাবে গণনা হয়', () => {
  expect(service.calculateFee(1000)).toBe(15);
});
```

---

## 📋 Test Double তুলনা সারণী

| বৈশিষ্ট্য | Dummy | Stub | Spy | Mock | Fake |
|-----------|-------|------|-----|------|------|
| **উদ্দেশ্য** | Parameter পূরণ | নির্দিষ্ট মান ফেরত | কল রেকর্ড | প্রত্যাশা যাচাই | সরলীকৃত বাস্তবায়ন |
| **আচরণ আছে?** | ❌ | সীমিত | সীমিত | সীমিত | ✅ সম্পূর্ণ |
| **Verification** | কোনোটা না | State | Behavior (পরে) | Behavior (আগে) | State |
| **জটিলতা** | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ব্যবহার ক্ষেত্র** | Constructor | Indirect input | Side effect যাচাই | Interaction যাচাই | Complex dependency |
| **PHP টুল** | createMock() | willReturn() | Mockery::spy() | shouldReceive() | নিজে লিখুন |
| **Jest টুল** | `{}` / `null` | jest.fn().mockReturnValue() | jest.spyOn() | jest.fn() + expect | নিজে লিখুন |
| **পুনঃব্যবহারযোগ্য?** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **রক্ষণাবেক্ষণ** | কম | কম | মাঝারি | বেশি | বেশি (কিন্তু মূল্যবান) |

---

## 📝 সারসংক্ষেপ

### মূল শিক্ষা

১. **Test Double বুঝুন** — Dummy, Stub, Spy, Mock, Fake প্রতিটির নির্দিষ্ট ব্যবহার আছে। সব জায়গায় Mock ব্যবহার করবেন না।

২. **Boundary-তে Mock করুন** — external system (বিকাশ API, SMS gateway, ডাটাবেস) mock করুন, আপনার নিজের domain logic mock করবেন না।

৩. **Behavior টেস্ট করুন, Implementation নয়** — method কতবার কল হলো তার চেয়ে গুরুত্বপূর্ণ হলো সঠিক ফলাফল পাওয়া।

৪. **Contract Test লিখুন** — আপনার Fake/Mock আসল implementation-এর সাথে সামঞ্জস্যপূর্ণ কিনা নিশ্চিত করুন।

৫. **Test Confidence Spectrum** — Unit test দিয়ে দ্রুত ফিডব্যাক পান, Integration/E2E দিয়ে আত্মবিশ্বাস বাড়ান। সঠিক ভারসাম্য বজায় রাখুন।

৬. **Over-Mocking এড়িয়ে চলুন** — যদি আপনার টেস্ট শুধু mock setup আর verification-ই হয়, তাহলে আপনি সম্ভবত ভুল জিনিস টেস্ট করছেন।

### দ্রুত সিদ্ধান্ত গাইড

```
dependency কি external system (API, DB, File)?
  ├─ হ্যাঁ → Mock/Fake ব্যবহার করুন
  └─ না → dependency কি side-effect free?
       ├─ হ্যাঁ → আসল object ব্যবহার করুন
       └─ না → dependency কি ধীর বা flaky?
            ├─ হ্যাঁ → Fake তৈরি করুন (InMemory)
            └─ না → আসল object ব্যবহার করুন
```

> **"Write tests that give you confidence, not tests that give you coverage."**
> — কোড কাভারেজ নয়, আত্মবিশ্বাস অর্জন করুন আপনার টেস্ট থেকে। ✨
