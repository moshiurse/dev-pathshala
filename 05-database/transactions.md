# 🔒 ট্রানজ্যাকশন ও ACID

## 📌 সংজ্ঞা

### Transaction কী?

Transaction হলো ডেটাবেসে এক বা একাধিক operation-এর একটি **logical unit of work** যেটি সম্পূর্ণভাবে execute হবে অথবা একটিও execute হবে না। এটি "all or nothing" principle অনুসরণ করে।

বাস্তব উদাহরণ ধরুন — bKash থেকে আপনি ১০০০ টাকা পাঠাচ্ছেন আপনার বন্ধুকে। এই operation-এ দুটি কাজ হয়:

1. আপনার account থেকে ১০০০ টাকা কমবে
2. আপনার বন্ধুর account-এ ১০০০ টাকা যোগ হবে

যদি প্রথম কাজটি হয় কিন্তু দ্বিতীয়টি না হয়, তাহলে ১০০০ টাকা হারিয়ে যাবে। Transaction guarantee দেয় যে হয় দুটোই হবে, নাহলে কোনোটাই হবে না।

```sql
-- bKash money transfer transaction
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE phone = '01712345678';
    UPDATE accounts SET balance = balance + 1000 WHERE phone = '01898765432';
    INSERT INTO transactions (sender, receiver, amount, type) 
        VALUES ('01712345678', '01898765432', 1000, 'TRANSFER');
COMMIT;
```

---

### ACID Properties — গভীর বিশ্লেষণ

ACID হলো চারটি fundamental property যা প্রতিটি reliable transaction system-এ থাকতে হয়।

#### ১. Atomicity (অবিভাজ্যতা)

**সংজ্ঞা:** Transaction-এর সব operation হয় সম্পূর্ণ succeed করবে, নাহলে সম্পূর্ণ fail করবে। আংশিক execution কোনোভাবেই সম্ভব নয়।

**বাস্তব উদাহরণ:** ধরুন আপনি Daraz থেকে একটি ফোন কিনছেন। Order place করার সময় তিনটি কাজ হয় — inventory কমানো, payment process করা, order record তৈরি করা। যদি payment fail করে, তাহলে inventory-ও আগের অবস্থায় ফিরে যাবে।

**কীভাবে কাজ করে:**
- **Undo Log (InnoDB):** প্রতিটি modification-এর আগে পুরনো value undo log-এ সংরক্ষণ করে। Rollback হলে undo log থেকে restore করে।
- **Write-Ahead Logging (PostgreSQL):** পরিবর্তন disk-এ লেখার আগে WAL-এ log করে। Crash হলে WAL থেকে recover করে।

```sql
-- Atomicity উদাহরণ: e-commerce order
BEGIN;
    -- inventory কমাও
    UPDATE products SET stock = stock - 1 WHERE id = 101 AND stock > 0;
    
    -- payment record তৈরি করো
    INSERT INTO payments (order_id, amount, status) VALUES (5001, 45000, 'COMPLETED');
    
    -- order তৈরি করো
    INSERT INTO orders (id, user_id, product_id, amount) VALUES (5001, 42, 101, 45000);
    
    -- যদি কোনো একটি fail করে, সব rollback হবে
COMMIT;
```

#### ২. Consistency (সামঞ্জস্যতা)

**সংজ্ঞা:** Transaction শুরু এবং শেষের সময় database অবশ্যই একটি valid state-এ থাকবে। কোনো constraint, rule, বা trigger violate হবে না।

**বাস্তব উদাহরণ:** ব্যাংক account-এ balance কখনো negative হতে পারবে না। যদি আপনার account-এ ৫০০ টাকা থাকে আর ১০০০ টাকা transfer করতে চান, consistency এটি আটকে দেবে।

**Consistency নিশ্চিত করার উপায়:**
- CHECK constraints
- FOREIGN KEY constraints
- UNIQUE constraints
- Triggers
- Application-level validation

```sql
-- Consistency: balance কখনো negative হবে না
ALTER TABLE accounts ADD CONSTRAINT positive_balance CHECK (balance >= 0);

-- এই transaction fail করবে যদি balance যথেষ্ট না থাকে
BEGIN;
    UPDATE accounts SET balance = balance - 10000 
    WHERE phone = '01712345678';
    -- ERROR: new row violates check constraint "positive_balance"
ROLLBACK;
```

#### ৩. Isolation (বিচ্ছিন্নতা)

**সংজ্ঞা:** একাধিক concurrent transaction একে অপরকে প্রভাবিত করবে না। প্রতিটি transaction এমনভাবে execute হবে যেন সে একাই চলছে।

**বাস্তব উদাহরণ:** দুজন মানুষ একই সময়ে একটি concert-এর শেষ ticket কিনতে চাইছে। Isolation নিশ্চিত করে যে একজনই ticket পাবে, দুজন একই ticket পাবে না।

```sql
-- Isolation: concurrent booking
-- Transaction A                          -- Transaction B
BEGIN;                                    BEGIN;
SELECT seats FROM events                  SELECT seats FROM events
WHERE id = 1; -- seats = 1               WHERE id = 1; -- seats = 1
UPDATE events SET seats = seats - 1       -- Transaction B অপেক্ষা করবে
WHERE id = 1;                             -- A commit করার পর B দেখবে seats = 0
COMMIT;                                   UPDATE events SET seats = seats - 1
                                          WHERE id = 1; -- fail বা 0 > check
                                          ROLLBACK;
```

#### ৪. Durability (স্থায়িত্ব)

**সংজ্ঞা:** Transaction একবার commit হলে সেই পরিবর্তন স্থায়ী — system crash, power failure, বা যেকোনো বিপর্যয়েও data হারাবে না।

**বাস্তব উদাহরণ:** আপনি bKash-এ টাকা পাঠালেন এবং confirmation পেলেন। এরপর bKash-এর server crash করলেও আপনার transaction record থাকবে।

**কীভাবে কাজ করে:**
- **WAL (Write-Ahead Log):** Commit-এর আগে log disk-এ flush করে
- **Checkpointing:** পর্যায়ক্রমে dirty page disk-এ লেখে
- **Replication:** Multiple server-এ data copy রাখে

```sql
-- Durability নিশ্চিত করতে MySQL/InnoDB setting
SET GLOBAL innodb_flush_log_at_trx_commit = 1; -- প্রতি commit-এ log flush
SET GLOBAL sync_binlog = 1;                     -- binary log sync
```

---

## 📊 Transaction Lifecycle ডায়াগ্রাম

### মূল Transaction Flow

```
    ┌─────────────────────────────────────────────────────────┐
    │                  TRANSACTION LIFECYCLE                    │
    └─────────────────────────────────────────────────────────┘

    [Client Request]
          │
          ▼
    ┌──────────┐
    │  BEGIN    │◄─── Transaction শুরু, নতুন TX ID বরাদ্দ
    │TRANSACTION│
    └────┬─────┘
         │
         ▼
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ INSERT/  │────▶│ UPDATE/  │────▶│ DELETE/  │
    │ SELECT   │     │ SELECT   │     │ SELECT   │
    └────┬─────┘     └────┬─────┘     └────┬─────┘
         │                │                │
         │    Undo Log-এ পুরনো value সংরক্ষণ
         │                │                │
         ▼                ▼                ▼
    ┌─────────────────────────────────────────┐
    │         সব কিছু কি সফল হয়েছে?           │
    └────────────┬────────────────┬────────────┘
                 │                │
            হ্যাঁ ✅            না ❌
                 │                │
                 ▼                ▼
          ┌──────────┐    ┌──────────────┐
          │  COMMIT  │    │   ROLLBACK   │
          │          │    │              │
          │ WAL flush│    │ Undo Log থেকে│
          │ to disk  │    │ restore করো  │
          └──────────┘    └──────────────┘
                 │                │
                 ▼                ▼
          ┌──────────┐    ┌──────────────┐
          │ Data     │    │ Data আগের   │
          │ স্থায়ী    │    │ অবস্থায় ফিরে │
          │ সংরক্ষিত  │    │ গেছে         │
          └──────────┘    └──────────────┘
```

### Savepoint Flow

```
    BEGIN TRANSACTION
          │
          ▼
    ┌─────────────┐
    │ Operation A  │  ◄── INSERT INTO orders...
    └──────┬──────┘
           │
           ▼
    ╔══════════════╗
    ║ SAVEPOINT    ║  ◄── SAVEPOINT sp1;
    ║    sp1       ║
    ╚══════┬═══════╝
           │
           ▼
    ┌─────────────┐
    │ Operation B  │  ◄── UPDATE inventory...
    └──────┬──────┘
           │
           ▼
    ╔══════════════╗
    ║ SAVEPOINT    ║  ◄── SAVEPOINT sp2;
    ║    sp2       ║
    ╚══════┬═══════╝
           │
           ▼
    ┌─────────────┐
    │ Operation C  │  ◄── INSERT INTO payments... (FAIL!)
    └──────┬──────┘
           │
           ▼
    ╔══════════════════════╗
    ║ ROLLBACK TO sp2     ║  ◄── C বাতিল, A ও B অক্ষত
    ╚══════┬═══════════════╝
           │
           ▼
    ┌─────────────┐
    │ Operation D  │  ◄── বিকল্প payment method
    └──────┬──────┘
           │
           ▼
       ┌────────┐
       │ COMMIT │  ◄── A, B, D commit — C বাদ
       └────────┘
```

---

## 💻 Basic Transactions

### Raw SQL

```sql
-- ========================================
-- MySQL / PostgreSQL Basic Transaction
-- ========================================

-- ১. সহজ transaction
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (42, 5000);
    UPDATE wallets SET balance = balance - 5000 WHERE user_id = 42;
COMMIT;

-- ২. Error handling সহ (PostgreSQL PL/pgSQL)
DO $$
DECLARE
    v_order_id INTEGER;
BEGIN
    INSERT INTO orders (user_id, total, status) 
        VALUES (42, 5000, 'PENDING')
        RETURNING id INTO v_order_id;
    
    UPDATE wallets 
        SET balance = balance - 5000 
        WHERE user_id = 42 AND balance >= 5000;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'অপর্যাপ্ত ব্যালেন্স';
    END IF;
    
    UPDATE orders SET status = 'CONFIRMED' WHERE id = v_order_id;
    
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Transaction failed: %', SQLERRM;
        -- PL/pgSQL block-এ automatic rollback হয়
END $$;

-- ৩. MySQL Stored Procedure সহ
DELIMITER //
CREATE PROCEDURE transfer_money(
    IN sender_phone VARCHAR(15),
    IN receiver_phone VARCHAR(15),
    IN amount DECIMAL(12,2)
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Transfer failed';
    END;
    
    START TRANSACTION;
        UPDATE accounts SET balance = balance - amount 
            WHERE phone = sender_phone AND balance >= amount;
        
        IF ROW_COUNT() = 0 THEN
            SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient balance';
        END IF;
        
        UPDATE accounts SET balance = balance + amount 
            WHERE phone = receiver_phone;
        
        INSERT INTO transfer_log (sender, receiver, amount, created_at)
            VALUES (sender_phone, receiver_phone, amount, NOW());
    COMMIT;
END //
DELIMITER ;

CALL transfer_money('01712345678', '01898765432', 1000.00);
```

### PHP Laravel

```php
<?php

use Illuminate\Support\Facades\DB;
use App\Models\Order;
use App\Models\Wallet;
use App\Models\Payment;
use App\Exceptions\InsufficientBalanceException;

// ========================================
// ১. Closure-based Transaction (সবচেয়ে সহজ)
// ========================================
// Laravel স্বয়ংক্রিয়ভাবে commit/rollback handle করে
// Exception throw হলে rollback, না হলে commit

$order = DB::transaction(function () {
    $order = Order::create([
        'user_id' => auth()->id(),
        'total'   => 5000,
        'status'  => 'pending',
    ]);

    Wallet::where('user_id', auth()->id())
        ->where('balance', '>=', 5000)
        ->decrement('balance', 5000);

    Payment::create([
        'order_id' => $order->id,
        'amount'   => 5000,
        'method'   => 'bkash',
        'status'   => 'completed',
    ]);

    return $order;
});

// ========================================
// ২. Manual Transaction (বেশি control)
// ========================================

DB::beginTransaction();

try {
    $wallet = Wallet::where('user_id', $userId)->lockForUpdate()->first();
    
    if ($wallet->balance < $amount) {
        throw new InsufficientBalanceException('ব্যালেন্স অপর্যাপ্ত');
    }

    $wallet->decrement('balance', $amount);

    $order = Order::create([
        'user_id' => $userId,
        'total'   => $amount,
        'status'  => 'confirmed',
    ]);

    event(new OrderCreated($order));

    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    Log::error('Transaction failed', [
        'user_id' => $userId,
        'error'   => $e->getMessage(),
    ]);
    throw $e;
}

// ========================================
// ৩. Transaction with retry (deadlock handling)
// ========================================
// দ্বিতীয় parameter হলো retry attempts — deadlock হলে পুনরায় চেষ্টা করবে

$result = DB::transaction(function () use ($senderId, $receiverId, $amount) {
    $sender = Wallet::where('user_id', $senderId)->lockForUpdate()->first();
    $receiver = Wallet::where('user_id', $receiverId)->lockForUpdate()->first();

    if ($sender->balance < $amount) {
        throw new InsufficientBalanceException();
    }

    $sender->decrement('balance', $amount);
    $receiver->increment('balance', $amount);

    return TransferLog::create([
        'sender_id'   => $senderId,
        'receiver_id' => $receiverId,
        'amount'      => $amount,
    ]);
}, 5); // ৫ বার retry করবে deadlock-এ
```

### Node.js — Sequelize ও Knex

```javascript
// ========================================
// Sequelize Transactions
// ========================================
const { Sequelize, Transaction } = require('sequelize');
const sequelize = new Sequelize('mysql://root:pass@localhost/bkash_db');

// ১. Managed Transaction (auto commit/rollback)
async function transferMoney(senderId, receiverId, amount) {
    const result = await sequelize.transaction(async (t) => {
        const sender = await Wallet.findOne({
            where: { userId: senderId },
            lock: t.LOCK.UPDATE, // SELECT FOR UPDATE
            transaction: t,
        });

        if (sender.balance < amount) {
            throw new Error('অপর্যাপ্ত ব্যালেন্স');
        }

        await sender.decrement('balance', { by: amount, transaction: t });

        const receiver = await Wallet.findOne({
            where: { userId: receiverId },
            lock: t.LOCK.UPDATE,
            transaction: t,
        });

        await receiver.increment('balance', { by: amount, transaction: t });

        const transfer = await TransferLog.create({
            senderId,
            receiverId,
            amount,
            status: 'COMPLETED',
        }, { transaction: t });

        return transfer;
    });

    return result;
}

// ২. Unmanaged Transaction (manual control)
async function createOrderWithPayment(userId, cartItems) {
    const t = await sequelize.transaction();

    try {
        const order = await Order.create({
            userId,
            status: 'PENDING',
            total: cartItems.reduce((sum, item) => sum + item.price, 0),
        }, { transaction: t });

        for (const item of cartItems) {
            const [affectedRows] = await Product.update(
                { stock: sequelize.literal('stock - 1') },
                {
                    where: {
                        id: item.productId,
                        stock: { [Op.gt]: 0 },
                    },
                    transaction: t,
                }
            );

            if (affectedRows === 0) {
                throw new Error(`Product ${item.productId} out of stock`);
            }

            await OrderItem.create({
                orderId: order.id,
                productId: item.productId,
                price: item.price,
            }, { transaction: t });
        }

        await t.commit();
        return order;
    } catch (error) {
        await t.rollback();
        throw error;
    }
}

// ========================================
// Knex Transactions
// ========================================
const knex = require('knex')({
    client: 'pg',
    connection: 'postgres://user:pass@localhost/bank_db',
});

// ১. Knex transaction সহ bKash transfer
async function bkashTransfer(senderPhone, receiverPhone, amount) {
    return knex.transaction(async (trx) => {
        const sender = await trx('accounts')
            .where('phone', senderPhone)
            .forUpdate() // row lock
            .first();

        if (!sender || sender.balance < amount) {
            throw new Error('ব্যালেন্স অপর্যাপ্ত');
        }

        await trx('accounts')
            .where('phone', senderPhone)
            .decrement('balance', amount);

        await trx('accounts')
            .where('phone', receiverPhone)
            .increment('balance', amount);

        const [logEntry] = await trx('transfer_log')
            .insert({
                sender_phone: senderPhone,
                receiver_phone: receiverPhone,
                amount,
                status: 'SUCCESS',
                created_at: new Date(),
            })
            .returning('*');

        return logEntry;
    });
}
```

---

## 🔥 Advanced Deep Dive

### ১. Isolation Levels

Isolation level নির্ধারণ করে যে concurrent transaction গুলো একে অপরের uncommitted data কতটুকু দেখতে পারবে। যত বেশি isolation, তত বেশি safety কিন্তু তত কম performance।

```
    Isolation Level Spectrum
    
    কম Isolation ◄─────────────────────────────► বেশি Isolation
    বেশি Performance                               কম Performance
    
    ┌──────────────┬──────────────┬──────────────┬──────────────┐
    │    READ      │    READ      │  REPEATABLE  │              │
    │ UNCOMMITTED  │  COMMITTED   │    READ      │ SERIALIZABLE │
    │              │              │              │              │
    │ Dirty Read ✓ │ Dirty Read ✗ │ Dirty Read ✗ │ Dirty Read ✗ │
    │ Non-Rep  ✓   │ Non-Rep  ✓   │ Non-Rep  ✗   │ Non-Rep  ✗   │
    │ Phantom  ✓   │ Phantom  ✓   │ Phantom  ✓*  │ Phantom  ✗   │
    └──────────────┴──────────────┴──────────────┴──────────────┘
    
    * MySQL InnoDB-তে REPEATABLE READ-এও gap lock দিয়ে phantom আটকায়
```

#### READ UNCOMMITTED (সবচেয়ে কম isolation)

**ব্যাখ্যা:** এক transaction অন্য transaction-এর uncommitted data পড়তে পারে। এটি সবচেয়ে দ্রুত কিন্তু সবচেয়ে বিপজ্জনক।

**যে সমস্যা সমাধান করে:** কোনো সমস্যা সমাধান করে না, শুধু maximum performance দেয়।

**যে সমস্যা তৈরি করে:** Dirty Read, Non-Repeatable Read, Phantom Read — সবই সম্ভব।

**কখন ব্যবহার করবেন:** শুধুমাত্র analytics/reporting query-তে যেখানে approximate data গ্রহণযোগ্য। কখনোই financial transaction-এ ব্যবহার করবেন না।

```sql
-- READ UNCOMMITTED সেট করা
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- MySQL এ session level এ সেট
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- অথবা hint ব্যবহার করে (MySQL)
SELECT /*+ SET_VAR(transaction_isolation='READ-UNCOMMITTED') */
    COUNT(*) AS approx_total_orders
FROM orders
WHERE created_at > '2024-01-01';
```

**Performance Impact:** ⚡ সবচেয়ে দ্রুত — কোনো lock acquire করতে হয় না।

#### READ COMMITTED (PostgreSQL Default)

**ব্যাখ্যা:** শুধুমাত্র committed data পড়তে পারে। প্রতিটি SELECT statement execution-এর সময় সর্বশেষ committed snapshot দেখে।

**যে সমস্যা সমাধান করে:** Dirty Read আটকায়।

**যে সমস্যা তৈরি করে:** Non-Repeatable Read — একই transaction-এ একই query দুবার চালালে ভিন্ন ফলাফল আসতে পারে কারণ মাঝে অন্য transaction commit করেছে।

```sql
-- READ COMMITTED (PostgreSQL default)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- উদাহরণ: Non-Repeatable Read সমস্যা
-- Transaction A
BEGIN;
SELECT balance FROM accounts WHERE phone = '01712345678';
-- balance = 5000

-- এদিকে Transaction B commit করে balance কমিয়ে দিলো

SELECT balance FROM accounts WHERE phone = '01712345678';
-- balance = 3000 (ভিন্ন ফলাফল!)
COMMIT;
```

**Performance Impact:** ⚡⚡ ভালো performance — short-lived read lock।

#### REPEATABLE READ (MySQL InnoDB Default)

**ব্যাখ্যা:** Transaction শুরুর সময়কার snapshot ব্যবহার করে। পুরো transaction জুড়ে একই data দেখে। নতুন committed data দেখতে পায় না।

**যে সমস্যা সমাধান করে:** Dirty Read + Non-Repeatable Read।

**যে সমস্যা তৈরি করে:** Phantom Read — নতুন row insert হলে সেটি দেখতে পারে (PostgreSQL-এ, MySQL InnoDB gap lock দিয়ে আটকায়)।

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
-- এই snapshot transaction শেষ পর্যন্ত consistent থাকবে
SELECT * FROM products WHERE category = 'electronics' AND price < 50000;
-- 10 rows returned

-- অন্য transaction নতুন product insert + commit করলেও
-- এই transaction-এ আবার query করলে একই 10 rows পাবে

SELECT * FROM products WHERE category = 'electronics' AND price < 50000;
-- 10 rows (MySQL InnoDB), PostgreSQL-এ phantom row দেখতে পারে
COMMIT;
```

**Performance Impact:** ⚡⚡⚡ মাঝারি — snapshot maintenance-এর overhead আছে।

#### SERIALIZABLE (সবচেয়ে বেশি isolation)

**ব্যাখ্যা:** Transaction গুলো এমনভাবে execute হয় যেন সেগুলো একটির পর একটি serial ভাবে চলছে। সম্পূর্ণ isolation নিশ্চিত করে।

**যে সমস্যা সমাধান করে:** সব concurrency সমস্যা — Dirty Read, Non-Repeatable Read, Phantom Read, Write Skew।

**যে সমস্যা তৈরি করে:** Performance bottleneck, deadlock-এর সম্ভাবনা বৃদ্ধি, serialization failure-এ retry প্রয়োজন।

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- PostgreSQL SSI (Serializable Snapshot Isolation) ব্যবহার করে
-- conflict detect করলে transaction abort করে
BEGIN;
SELECT COUNT(*) FROM bookings WHERE seat_id = 42 AND show_date = '2024-12-25';
-- count = 0

INSERT INTO bookings (user_id, seat_id, show_date) VALUES (7, 42, '2024-12-25');
COMMIT;
-- যদি অন্য concurrent transaction একই কাজ করে, একটি abort হবে
-- ERROR: could not serialize access due to read/write dependencies
```

**Performance Impact:** ⚡⚡⚡⚡ সবচেয়ে ধীর — lock contention বেশি, retry logic দরকার।

#### MySQL vs PostgreSQL Default Comparison

| বৈশিষ্ট্য | MySQL (InnoDB) | PostgreSQL |
|---|---|---|
| Default Level | REPEATABLE READ | READ COMMITTED |
| MVCC | Undo Log ভিত্তিক | Tuple versioning |
| Phantom Prevention | Gap Lock (RR-তেও) | শুধু SERIALIZABLE-এ |
| SERIALIZABLE | Lock-based | SSI (optimistic) |

#### Isolation Level সেট করার উপায়

```php
// Laravel — Isolation Level সেট করা
// ১. Raw query দিয়ে
DB::statement('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
DB::transaction(function () {
    // serializable isolation-এ execute হবে
});

// ২. Config-এ (config/database.php)
'mysql' => [
    'isolation_level' => 'REPEATABLE READ', // global default
],
```

```javascript
// Sequelize — Isolation Level
const { Transaction } = require('sequelize');

await sequelize.transaction({
    isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE,
}, async (t) => {
    // serializable isolation-এ execute হবে
    const booking = await Booking.findOne({
        where: { seatId: 42 },
        transaction: t,
    });
});

// Knex — Isolation Level
await knex.transaction(async (trx) => {
    await trx.raw('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    // ... operations
});
```

---

### ২. Concurrency Problems

#### Dirty Read (অপরিচ্ছন্ন পঠন)

অন্য transaction-এর uncommitted data পড়া — যেটি পরে rollback হতে পারে।

```
    সময়রেখা ─────────────────────────────────────────────►

    Transaction A (bKash):
    ├── BEGIN
    ├── UPDATE balance = 4000 (ছিলো 5000, 1000 কমালো)
    │                                              ├── ROLLBACK ❌
    │                                              │   (balance আবার 5000)
    │
    Transaction B (Balance Check):
    │          ├── BEGIN
    │          ├── SELECT balance → 4000 ⚠️ ভুল!
    │          │   (uncommitted data পড়লো)
    │          ├── COMMIT
    │
    ফলাফল: B ভুল balance দেখলো (4000), আসলে balance 5000
```

#### Non-Repeatable Read (অ-পুনরাবৃত্তিযোগ্য পঠন)

একই transaction-এ একই row দুবার পড়লে ভিন্ন value পাওয়া।

```
    সময়রেখা ─────────────────────────────────────────────►

    Transaction A (Report Generation):
    ├── BEGIN
    ├── SELECT balance WHERE id=1 → 5000
    │                                    ├── SELECT balance → 3000 ⚠️
    │                                    │   (ভিন্ন value!)
    │                                    ├── COMMIT
    │
    Transaction B (Transfer):
    │       ├── BEGIN
    │       ├── UPDATE balance = 3000 WHERE id=1
    │       ├── COMMIT ✅
    │
    ফলাফল: A একই row-তে 5000 ও 3000 দেখলো — report inconsistent
```

#### Phantom Read (অলীক পঠন)

একই transaction-এ একই range query-তে নতুন/মুছে যাওয়া row দেখা।

```
    সময়রেখা ─────────────────────────────────────────────►

    Transaction A (Inventory Report):
    ├── BEGIN
    ├── SELECT COUNT(*) WHERE category='phone' → 10
    │                                           ├── SELECT COUNT(*) → 11 ⚠️
    │                                           │   (নতুন phantom row!)
    │                                           ├── COMMIT
    │
    Transaction B (New Product):
    │         ├── BEGIN
    │         ├── INSERT INTO products (category='phone', ...)
    │         ├── COMMIT ✅
    │
    ফলাফল: A একই query-তে 10 ও 11 পেলো
```

#### Lost Update (হারানো আপডেট)

দুটি transaction একই data পড়ে আপডেট করলে একটির আপডেট হারিয়ে যাওয়া।

```
    সময়রেখা ─────────────────────────────────────────────►

    Transaction A (Admin):
    ├── BEGIN
    ├── SELECT stock WHERE id=1 → 100
    │                              ├── UPDATE stock = 100 - 5 = 95
    │                              ├── COMMIT
    │
    Transaction B (Customer):
    │    ├── BEGIN
    │    ├── SELECT stock WHERE id=1 → 100 (একই পুরনো value)
    │    │                                    ├── UPDATE stock = 100 - 3 = 97
    │    │                                    ├── COMMIT
    │
    ফলাফল: stock 97 হলো (A-র 5 কমানো হারিয়ে গেলো)
    সঠিক হওয়া উচিত ছিলো: 100 - 5 - 3 = 92
```

#### Write Skew

দুটি transaction ভিন্ন row পড়ে ভিন্ন row আপডেট করে কিন্তু একটি invariant ভেঙে যায়।

```
    সময়রেখা ─────────────────────────────────────────────►
    
    নিয়ম: হাসপাতালে সবসময় কমপক্ষে ১ জন ডাক্তার on-call থাকতে হবে
    বর্তমানে: Dr. Rahim ও Dr. Karim দুজনই on-call

    Transaction A (Dr. Rahim):
    ├── BEGIN
    ├── SELECT COUNT(*) on_call = 2 (≥1, তাই ছুটি নিতে পারি)
    │                                    ├── UPDATE SET on_call=false
    │                                    │   WHERE doctor='Rahim'
    │                                    ├── COMMIT
    │
    Transaction B (Dr. Karim):
    │    ├── BEGIN
    │    ├── SELECT COUNT(*) on_call = 2 (≥1, তাই ছুটি নিতে পারি)
    │    │                                    ├── UPDATE SET on_call=false
    │    │                                    │   WHERE doctor='Karim'
    │    │                                    ├── COMMIT
    │
    ফলাফল: দুজনই ছুটি নিলো! on-call ডাক্তার = 0 ⚠️
    সমাধান: SERIALIZABLE isolation অথবা explicit lock
```

---

### ৩. Locking Mechanisms

#### Lock Types

```
    ┌─────────────────────────────────────────────┐
    │              LOCK HIERARCHY                   │
    ├─────────────────────────────────────────────┤
    │                                              │
    │  Table Level:  IS ──── IX ──── S ──── X      │
    │                │       │       │       │     │
    │  Row Level:    ·       ·       S ──── X      │
    │                                              │
    │  Compatibility Matrix:                       │
    │  ┌────┬────┬────┬────┬────┐                  │
    │  │    │ IS │ IX │ S  │ X  │                  │
    │  ├────┼────┼────┼────┼────┤                  │
    │  │ IS │ ✅ │ ✅ │ ✅ │ ❌ │                  │
    │  │ IX │ ✅ │ ✅ │ ❌ │ ❌ │                  │
    │  │ S  │ ✅ │ ❌ │ ✅ │ ❌ │                  │
    │  │ X  │ ❌ │ ❌ │ ❌ │ ❌ │                  │
    │  └────┴────┴────┴────┴────┘                  │
    │  IS = Intention Shared, IX = Intention Excl.│
    │  S = Shared, X = Exclusive                   │
    └─────────────────────────────────────────────┘
```

**Shared Lock (S):** একাধিক transaction একসাথে পড়তে পারে কিন্তু কেউ লিখতে পারে না।

**Exclusive Lock (X):** শুধুমাত্র একটি transaction পড়তে ও লিখতে পারে, বাকিরা অপেক্ষা করবে।

**Intention Lock:** Table-level এ hint দেয় যে row-level lock নেওয়া হবে। Full table scan এড়াতে সাহায্য করে।

#### Gap Lock ও Next-Key Lock (InnoDB)

```sql
-- Gap Lock: দুটি index value-র মধ্যবর্তী gap lock করে
-- যেমন: id 10 এবং 20 এর মধ্যে gap lock করলে 11-19 insert হবে না
SELECT * FROM products WHERE price BETWEEN 1000 AND 5000 FOR UPDATE;
-- এটি 1000-5000 range-এর gap-এও lock দেয়

-- Next-Key Lock = Record Lock + Gap Lock
-- InnoDB REPEATABLE READ-এ phantom read আটকাতে ব্যবহার করে
-- record এবং তার আগের gap দুটোই lock করে
```

#### Row-Level vs Table-Level Locking

```sql
-- Row-Level Lock (InnoDB default, সুনির্দিষ্ট)
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- শুধু id=42 row lock হবে

-- Table-Level Lock (MyISAM, বা explicit)
LOCK TABLES orders WRITE;
-- পুরো table lock — অন্য সব query block হবে
-- ব্যবহার: bulk import, schema change
UNLOCK TABLES;
```

#### Deadlock

```
    ┌──────────────────────────────────────────────┐
    │              DEADLOCK SCENARIO                 │
    ├──────────────────────────────────────────────┤
    │                                               │
    │  TX-A: Lock Row 1 ─────────► Wait Row 2      │
    │           ▲                      │            │
    │           │      DEADLOCK!       │            │
    │           │         💀           ▼            │
    │  TX-B: Wait Row 1 ◄──────── Lock Row 2      │
    │                                               │
    └──────────────────────────────────────────────┘
```

```sql
-- Deadlock উদাহরণ:
-- Transaction A                    -- Transaction B
BEGIN;                              BEGIN;
UPDATE accounts SET balance=4000    UPDATE accounts SET balance=9000
WHERE id = 1; -- Lock row 1        WHERE id = 2; -- Lock row 2

UPDATE accounts SET balance=6000    UPDATE accounts SET balance=1000
WHERE id = 2; -- Wait for B...     WHERE id = 1; -- Wait for A... 💀

-- InnoDB deadlock detect করে একটি transaction rollback করে

-- Deadlock prevention: সবসময় একই ক্রমে lock নিন
-- সমাধান: id ascending order-এ lock করুন
BEGIN;
UPDATE accounts SET balance=4000 WHERE id = 1; -- আগে id=1
UPDATE accounts SET balance=6000 WHERE id = 2; -- তারপর id=2
COMMIT;
```

```sql
-- Lock wait timeout
SET innodb_lock_wait_timeout = 10; -- 10 সেকেন্ড পর timeout

-- Deadlock তথ্য দেখুন
SHOW ENGINE INNODB STATUS; -- Last detected deadlock section দেখুন
```

#### Laravel ও Sequelize-এ Locking

```php
// Laravel Locking
// Shared Lock — অন্যরা পড়তে পারবে কিন্তু লিখতে পারবে না
$user = User::where('id', 1)->sharedLock()->first();

// Exclusive Lock — SELECT FOR UPDATE
$wallet = Wallet::where('user_id', $userId)->lockForUpdate()->first();

// Deadlock-safe transfer: consistent lock ordering
DB::transaction(function () use ($id1, $id2, $amount) {
    // সবসময় ছোট id আগে lock করুন
    $ids = collect([$id1, $id2])->sort()->values();
    
    $wallets = Wallet::whereIn('user_id', $ids)
        ->orderBy('user_id')
        ->lockForUpdate()
        ->get()
        ->keyBy('user_id');
    
    $wallets[$id1]->decrement('balance', $amount);
    $wallets[$id2]->increment('balance', $amount);
}, 3); // 3 retry for deadlock
```

```javascript
// Sequelize Locking
const { Transaction } = require('sequelize');

// Shared Lock
const user = await User.findOne({
    where: { id: 1 },
    lock: Transaction.LOCK.SHARE,
    transaction: t,
});

// Exclusive Lock (FOR UPDATE)
const wallet = await Wallet.findOne({
    where: { userId },
    lock: Transaction.LOCK.UPDATE,
    transaction: t,
});

// Knex — FOR UPDATE with SKIP LOCKED (queue processing pattern)
const [nextJob] = await trx('job_queue')
    .where('status', 'pending')
    .orderBy('created_at')
    .limit(1)
    .forUpdate()
    .skipLocked() // locked row skip করে পরেরটি নেয়
    .select('*');
```

---

### ৪. MVCC (Multi-Version Concurrency Control)

MVCC হলো আধুনিক database-এর concurrency control mechanism যেখানে reader কখনো writer-কে block করে না এবং writer কখনো reader-কে block করে না। প্রতিটি transaction data-র নিজস্ব version/snapshot দেখে।

#### PostgreSQL MVCC

```
    ┌───────────────────────────────────────────────────────┐
    │           PostgreSQL Tuple Versioning                  │
    ├───────────────────────────────────────────────────────┤
    │                                                        │
    │  Row "accounts" id=1:                                  │
    │                                                        │
    │  ┌─────────────────────────────────────┐               │
    │  │ xmin=100  xmax=102  balance=5000   │ ◄── Dead tuple │
    │  └─────────────────────────────────────┘               │
    │         │                                              │
    │         │ UPDATE (TX 102)                               │
    │         ▼                                              │
    │  ┌─────────────────────────────────────┐               │
    │  │ xmin=102  xmax=105  balance=3000   │ ◄── Dead tuple │
    │  └─────────────────────────────────────┘               │
    │         │                                              │
    │         │ UPDATE (TX 105)                               │
    │         ▼                                              │
    │  ┌─────────────────────────────────────┐               │
    │  │ xmin=105  xmax=∞    balance=4500   │ ◄── Live tuple │
    │  └─────────────────────────────────────┘               │
    │                                                        │
    │  xmin = কোন TX তৈরি করেছে                               │
    │  xmax = কোন TX মুছেছে/আপডেট করেছে (∞ = এখনো live)     │
    │                                                        │
    │  Visibility Rule:                                      │
    │  TX 103 দেখবে balance=3000 (xmin=102 committed,       │
    │          xmax=105 not yet committed at TX 103 time)    │
    └───────────────────────────────────────────────────────┘
```

**PostgreSQL Visibility Rules:**
- `xmin` committed এবং `xmax` না থাকলে বা uncommitted হলে → tuple visible
- `xmax` committed হলে → tuple invisible (deleted/updated)
- Transaction-এর snapshot অনুযায়ী কোন TX committed কিনা যাচাই করে

**VACUUM — Dead Tuple পরিষ্কার:**

```sql
-- Dead tuple (আগের version) disk space দখল করে রাখে
-- VACUUM এগুলো পরিষ্কার করে

-- Manual vacuum
VACUUM VERBOSE accounts;

-- Aggressive vacuum (frozen XIDs reclaim)
VACUUM FULL accounts; -- table rewrite করে, exclusive lock নেয়

-- Autovacuum settings
ALTER TABLE accounts SET (
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_scale_factor = 0.1
);

-- Dead tuple পরিসংখ্যান দেখুন
SELECT relname, n_dead_tup, n_live_tup, 
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'accounts';
```

#### MySQL InnoDB MVCC

```
    ┌───────────────────────────────────────────────────────┐
    │              InnoDB Undo Log Versioning                │
    ├───────────────────────────────────────────────────────┤
    │                                                        │
    │  Clustered Index (Current Data):                       │
    │  ┌────────────────────────────────┐                    │
    │  │ id=1  balance=4500  TX_ID=105  │                    │
    │  │         roll_ptr ─────────┐    │                    │
    │  └────────────────────────────┘   │                    │
    │                                   │                    │
    │  Undo Log (Version Chain):        │                    │
    │                                   ▼                    │
    │  ┌────────────────────────┐  ┌────────────────────┐   │
    │  │ balance=5000 TX_ID=100 │◄─│ balance=3000       │   │
    │  │ (সবচেয়ে পুরনো)        │  │ TX_ID=102          │   │
    │  └────────────────────────┘  └────────────────────┘   │
    │                                                        │
    │  Read View (TX 103 শুরু হলে):                          │
    │  - Active TXs at start: [104, 105]                    │
    │  - TX 103 দেখবে balance=3000 (TX 102 committed)       │
    │  - TX 105 এখনো active তাই 4500 দেখবে না              │
    └───────────────────────────────────────────────────────┘
```

**InnoDB vs PostgreSQL MVCC পার্থক্য:**

| বৈশিষ্ট্য | PostgreSQL | MySQL InnoDB |
|---|---|---|
| Version Storage | Heap-এ নতুন tuple | Undo log-এ পুরনো version |
| Cleanup | VACUUM দরকার | Purge thread automatically |
| UPDATE Cost | নতুন tuple + index update | In-place update + undo log |
| Bloat সমস্যা | VACUUM না হলে bloat | কম (auto purge) |
| Read Performance | Dead tuple skip করতে হয় | Undo log traverse করতে হয় |

---

### ৫. Distributed Transactions

Single database-এর বাইরে যখন একাধিক service/database জুড়ে transaction দরকার হয়, তখন distributed transaction ব্যবহার করা হয়। এটি অনেক জটিল কারণ network failure, partial failure handle করতে হয়।

#### Two-Phase Commit (2PC)

```
    ┌─────────────────────────────────────────────────────────┐
    │              TWO-PHASE COMMIT (2PC)                      │
    ├─────────────────────────────────────────────────────────┤
    │                                                          │
    │              ┌──────────────┐                             │
    │              │ Coordinator  │                             │
    │              │  (TX Manager)│                             │
    │              └──────┬───────┘                             │
    │                     │                                    │
    │    Phase 1: PREPARE (ভোট সংগ্রহ)                         │
    │         ┌───────────┼───────────┐                        │
    │         ▼           ▼           ▼                        │
    │   ┌──────────┐ ┌──────────┐ ┌──────────┐               │
    │   │ DB-A     │ │ DB-B     │ │ DB-C     │               │
    │   │ bKash    │ │ Bank     │ │ Merchant │               │
    │   │          │ │          │ │          │               │
    │   │ PREPARED │ │ PREPARED │ │ PREPARED │               │
    │   │   ✅     │ │   ✅     │ │   ✅     │               │
    │   └────┬─────┘ └────┬─────┘ └────┬─────┘               │
    │        │            │            │                       │
    │    Phase 2: COMMIT (চূড়ান্ত সিদ্ধান্ত)                   │
    │        │            │            │                       │
    │        ▼            ▼            ▼                       │
    │   ┌──────────┐ ┌──────────┐ ┌──────────┐               │
    │   │COMMITTED │ │COMMITTED │ │COMMITTED │               │
    │   │   ✅     │ │   ✅     │ │   ✅     │               │
    │   └──────────┘ └──────────┘ └──────────┘               │
    │                                                          │
    │   ⚠️ সমস্যা: Coordinator crash হলে participants          │
    │      অনির্দিষ্টকাল PREPARED state-এ থাকতে পারে           │
    └─────────────────────────────────────────────────────────┘
```

```sql
-- XA Transaction (MySQL)
-- distributed transaction-এর SQL interface

-- Coordinator side:
XA START 'transfer-001';
UPDATE bkash_accounts SET balance = balance - 1000 WHERE phone = '01712345678';
XA END 'transfer-001';
XA PREPARE 'transfer-001';

-- সব participant PREPARED হলে:
XA COMMIT 'transfer-001';

-- কোনো participant fail করলে:
XA ROLLBACK 'transfer-001';
```

#### Three-Phase Commit (3PC)

2PC-এর blocking সমস্যা সমাধানে 3PC একটি অতিরিক্ত pre-commit phase যোগ করে। কিন্তু বাস্তবে network partition handle করতে পারে না বলে কম ব্যবহৃত হয়।

```
    Phase 1: CAN COMMIT? → Yes/No ভোট
    Phase 2: PRE-COMMIT  → সবাই ready, কিন্তু এখনো commit নয়
    Phase 3: DO COMMIT   → চূড়ান্ত commit

    সুবিধা: Coordinator crash হলে participants timeout-এ abort করতে পারে
    অসুবিধা: Network partition-এ inconsistency হতে পারে
```

#### Saga Pattern (Distributed Transaction Alternative)

দীর্ঘমেয়াদী distributed transaction-এর জন্য Saga pattern সবচেয়ে জনপ্রিয়। প্রতিটি step-এর জন্য compensating action থাকে।

```
    ┌──────────────────────────────────────────────────────────┐
    │              SAGA: E-Commerce Order Flow                  │
    │         (Choreography-based / Event-driven)              │
    ├──────────────────────────────────────────────────────────┤
    │                                                           │
    │  সফল Flow:                                               │
    │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────┐ │
    │  │ Create   │──▶│ Reserve  │──▶│ Process  │──▶│ Ship  │ │
    │  │ Order    │   │ Stock    │   │ Payment  │   │ Order │ │
    │  └──────────┘   └──────────┘   └──────────┘   └───────┘ │
    │                                                           │
    │  ব্যর্থ Flow (Compensating Actions):                      │
    │  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
    │  │ Cancel   │◀──│ Release  │◀──│ Payment  │ ✗ FAIL      │
    │  │ Order    │   │ Stock    │   │ Failed   │             │
    │  └──────────┘   └──────────┘   └──────────┘             │
    │                                                           │
    └──────────────────────────────────────────────────────────┘
```

```javascript
// Saga Pattern Implementation — Node.js
class OrderSaga {
    constructor() {
        this.steps = [];
    }

    addStep(execute, compensate) {
        this.steps.push({ execute, compensate });
        return this;
    }

    async run(context) {
        const completedSteps = [];

        try {
            for (const step of this.steps) {
                await step.execute(context);
                completedSteps.push(step);
            }
        } catch (error) {
            // Compensating actions — উল্টো ক্রমে
            for (const step of completedSteps.reverse()) {
                try {
                    await step.compensate(context);
                } catch (compError) {
                    console.error('Compensation failed:', compError);
                    // manual intervention দরকার — dead letter queue-তে পাঠান
                }
            }
            throw error;
        }
    }
}

// ব্যবহার: Daraz-style e-commerce order
const orderSaga = new OrderSaga();

orderSaga
    .addStep(
        async (ctx) => { ctx.order = await orderService.create(ctx.cart); },
        async (ctx) => { await orderService.cancel(ctx.order.id); }
    )
    .addStep(
        async (ctx) => { await inventoryService.reserve(ctx.order.items); },
        async (ctx) => { await inventoryService.release(ctx.order.items); }
    )
    .addStep(
        async (ctx) => { ctx.payment = await paymentService.charge(ctx.order); },
        async (ctx) => { await paymentService.refund(ctx.payment.id); }
    )
    .addStep(
        async (ctx) => { await shippingService.initiate(ctx.order); },
        async (ctx) => { await shippingService.cancel(ctx.order.id); }
    );

await orderSaga.run({ cart: userCart });
```

#### Distributed Transaction সমস্যা এবং বিকল্প

**সমস্যাগুলো:**
- **Blocking:** 2PC-তে coordinator crash হলে সব participant block হয়
- **Performance:** Network latency-র কারণে অনেক ধীর
- **Availability:** CAP theorem অনুযায়ী partition tolerance-এ consistency ত্যাগ করতে হতে পারে

**Eventually Consistent বিকল্প:**
- **Saga Pattern:** Compensating transaction দিয়ে eventual consistency
- **Outbox Pattern:** Event ও DB write একই local transaction-এ
- **CDC (Change Data Capture):** Database log থেকে event stream

---

### ৬. Optimistic vs Pessimistic Locking

#### Pessimistic Locking (হতাশাবাদী)

ধরে নেয় conflict হবেই, তাই আগেই lock করে রাখে। High contention scenario-তে ভালো কাজ করে।

```sql
-- SELECT FOR UPDATE — exclusive row lock
BEGIN;
SELECT * FROM products WHERE id = 101 FOR UPDATE;
-- এই row-তে অন্য কেউ write করতে পারবে না যতক্ষণ না commit হয়

UPDATE products SET stock = stock - 1 WHERE id = 101;
COMMIT;

-- FOR UPDATE NOWAIT — lock না পেলে সাথে সাথে error
SELECT * FROM products WHERE id = 101 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row

-- FOR UPDATE SKIP LOCKED — locked row skip করে (queue pattern)
SELECT * FROM job_queue 
WHERE status = 'pending' 
ORDER BY created_at 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
```

#### Optimistic Locking (আশাবাদী)

ধরে নেয় conflict হবে না, update-এর সময় version check করে। Low contention scenario-তে ভালো।

```sql
-- Version column দিয়ে optimistic locking
-- Step 1: Read with version
SELECT id, name, stock, version FROM products WHERE id = 101;
-- stock=50, version=3

-- Step 2: Update with version check (CAS — Compare And Swap)
UPDATE products 
SET stock = 49, version = version + 1 
WHERE id = 101 AND version = 3;

-- যদি affected rows = 0, তার মানে অন্য কেউ আগেই update করেছে
-- retry করতে হবে
```

```php
// Laravel — Pessimistic Locking
DB::transaction(function () {
    // SELECT FOR UPDATE
    $product = Product::where('id', 101)->lockForUpdate()->first();
    
    if ($product->stock < 1) {
        throw new OutOfStockException();
    }
    
    $product->decrement('stock');
    Order::create(['product_id' => 101, 'user_id' => auth()->id()]);
});

// Laravel — Optimistic Locking (manual implementation)
// Model-এ version column যোগ করুন
$product = Product::find(101);
$originalVersion = $product->version;

// business logic ...

$affected = Product::where('id', 101)
    ->where('version', $originalVersion)
    ->update([
        'stock' => $product->stock - 1,
        'version' => $originalVersion + 1,
    ]);

if ($affected === 0) {
    throw new StaleDataException('Data has been modified by another user');
}
```

```javascript
// Sequelize — Optimistic Locking (built-in support)
// Model definition — version column সক্রিয়
const Product = sequelize.define('Product', {
    name: DataTypes.STRING,
    stock: DataTypes.INTEGER,
}, {
    version: true, // version column auto-manage করবে
});

async function purchaseProduct(productId) {
    const product = await Product.findByPk(productId);
    product.stock -= 1;
    
    try {
        await product.save();
        // Sequelize স্বয়ংক্রিয়ভাবে WHERE version = X চেক করে
    } catch (error) {
        if (error instanceof Sequelize.OptimisticLockError) {
            // retry logic
            return purchaseProduct(productId);
        }
        throw error;
    }
}

// Sequelize — Pessimistic Locking
await sequelize.transaction(async (t) => {
    const product = await Product.findOne({
        where: { id: productId },
        lock: t.LOCK.UPDATE,
        transaction: t,
    });
    
    product.stock -= 1;
    await product.save({ transaction: t });
});
```

#### কোনটি কখন ব্যবহার করবেন?

```
    ┌─────────────────────────────────────────────────────┐
    │           DECISION GUIDE                             │
    ├─────────────────────────────────────────────────────┤
    │                                                      │
    │  Conflict Rate কি বেশি?                              │
    │     │                                                │
    │     ├── হ্যাঁ ──► PESSIMISTIC (lockForUpdate)         │
    │     │          উদাহরণ: bKash transfer,               │
    │     │          concert ticket booking                │
    │     │                                                │
    │     └── না ───► OPTIMISTIC (version column)          │
    │                উদাহরণ: blog post edit,               │
    │                profile update, CMS content           │
    │                                                      │
    │  Long-running operation?                             │
    │     ├── হ্যাঁ ──► OPTIMISTIC (lock ধরে রাখা ব্যয়বহুল) │
    │     └── না ───► যেকোনোটি                              │
    │                                                      │
    │  Retry logic implement করা সহজ?                      │
    │     ├── হ্যাঁ ──► OPTIMISTIC                           │
    │     └── না ───► PESSIMISTIC                           │
    └─────────────────────────────────────────────────────┘
```

---

### ৭. Savepoints

Savepoint হলো transaction-এর মধ্যে একটি চিহ্নিত বিন্দু যেখানে আংশিক rollback করা যায়। পুরো transaction rollback না করে শুধু নির্দিষ্ট অংশ বাতিল করতে পারেন।

```sql
-- Savepoint ব্যবহার
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (42, 15000);
    -- order তৈরি হলো
    
    SAVEPOINT payment_attempt;
    
    -- bKash payment try
    INSERT INTO payments (order_id, method, status) VALUES (1001, 'bkash', 'PROCESSING');
    -- bKash API call fail হলো...
    
    ROLLBACK TO SAVEPOINT payment_attempt;
    -- payment বাতিল, কিন্তু order এখনো আছে
    
    SAVEPOINT card_payment;
    
    -- Card payment try
    INSERT INTO payments (order_id, method, status) VALUES (1001, 'card', 'COMPLETED');
    
    -- সফল!
    RELEASE SAVEPOINT card_payment;
    
COMMIT;
-- order + card payment দুটোই commit হলো
```

```php
// Laravel — Nested Transactions (internally savepoints ব্যবহার করে)
DB::transaction(function () {
    Order::create(['user_id' => 42, 'total' => 15000]);
    
    try {
        // এটি SAVEPOINT তৈরি করবে (nested transaction)
        DB::transaction(function () {
            Payment::create([
                'order_id' => 1001,
                'method'   => 'bkash',
                'status'   => 'PROCESSING',
            ]);
            
            $response = BkashGateway::charge(1001, 15000);
            if (!$response->success) {
                throw new PaymentFailedException();
            }
        });
    } catch (PaymentFailedException $e) {
        // ROLLBACK TO SAVEPOINT — order অক্ষত থাকবে
        
        // বিকল্প payment try করুন
        DB::transaction(function () {
            Payment::create([
                'order_id' => 1001,
                'method'   => 'card',
                'status'   => 'COMPLETED',
            ]);
        });
    }
});

// Laravel savepoint counter দেখতে:
// DB::transactionLevel() — বর্তমান nesting depth
```

```javascript
// Sequelize — Savepoints
await sequelize.transaction(async (t) => {
    await Order.create({ userId: 42, total: 15000 }, { transaction: t });

    try {
        // nested transaction = savepoint
        await sequelize.transaction({ transaction: t }, async (sp) => {
            await Payment.create({
                orderId: 1001,
                method: 'bkash',
            }, { transaction: sp });

            const result = await bkashCharge(1001, 15000);
            if (!result.success) throw new Error('bKash failed');
        });
    } catch (e) {
        // savepoint rollback হয়েছে, বিকল্প try
        await Payment.create({
            orderId: 1001,
            method: 'card',
            status: 'COMPLETED',
        }, { transaction: t });
    }
});
```

---

### ৮. Transaction Patterns

#### Unit of Work Pattern

একটি business operation-এর সব database change একসাথে track করে এবং একটি single transaction-এ commit করে।

```php
// Laravel-এ Eloquent ORM নিজেই Unit of Work pattern follow করে
// কিন্তু explicit implementation:

class UnitOfWork
{
    private array $newEntities = [];
    private array $dirtyEntities = [];
    private array $deletedEntities = [];

    public function registerNew(Model $entity): void
    {
        $this->newEntities[] = $entity;
    }

    public function registerDirty(Model $entity): void
    {
        $this->dirtyEntities[] = $entity;
    }

    public function registerDeleted(Model $entity): void
    {
        $this->deletedEntities[] = $entity;
    }

    public function commit(): void
    {
        DB::transaction(function () {
            foreach ($this->newEntities as $entity) {
                $entity->save();
            }
            foreach ($this->dirtyEntities as $entity) {
                $entity->save();
            }
            foreach ($this->deletedEntities as $entity) {
                $entity->delete();
            }
        });

        $this->clear();
    }

    private function clear(): void
    {
        $this->newEntities = [];
        $this->dirtyEntities = [];
        $this->deletedEntities = [];
    }
}
```

#### Outbox Pattern (Event + DB Consistency)

Database write এবং event publish-কে consistent রাখতে outbox table ব্যবহার করা হয়। "Dual write" সমস্যা সমাধান করে।

```
    ┌──────────────────────────────────────────────────┐
    │              OUTBOX PATTERN                        │
    ├──────────────────────────────────────────────────┤
    │                                                   │
    │  ┌──────────────────────────────────────┐         │
    │  │  Single Database Transaction          │        │
    │  │                                       │        │
    │  │  1. INSERT INTO orders (...);         │        │
    │  │  2. INSERT INTO outbox_events (       │        │
    │  │       event_type, payload, status     │        │
    │  │     );                                │        │
    │  │  3. COMMIT;                           │        │
    │  └──────────────┬───────────────────────┘        │
    │                 │                                 │
    │                 ▼                                 │
    │  ┌──────────────────────────┐                    │
    │  │  Background Worker /     │                    │
    │  │  CDC (Debezium)          │                    │
    │  │                          │                    │
    │  │  outbox_events poll করে  │                    │
    │  │  → Kafka/RabbitMQ-তে    │                    │
    │  │    publish করে           │                    │
    │  │  → status = 'PUBLISHED' │                    │
    │  └──────────────────────────┘                    │
    └──────────────────────────────────────────────────┘
```

```php
// Outbox Pattern — Laravel Implementation
DB::transaction(function () use ($orderData) {
    $order = Order::create($orderData);

    // একই transaction-এ outbox event save
    OutboxEvent::create([
        'event_type'   => 'ORDER_CREATED',
        'aggregate_id' => $order->id,
        'payload'      => json_encode([
            'order_id' => $order->id,
            'user_id'  => $order->user_id,
            'total'    => $order->total,
        ]),
        'status'       => 'PENDING',
    ]);
});

// Background job যেটি outbox poll করে
// app/Console/Commands/ProcessOutbox.php
class ProcessOutbox extends Command
{
    public function handle()
    {
        $events = OutboxEvent::where('status', 'PENDING')
            ->orderBy('created_at')
            ->limit(100)
            ->get();

        foreach ($events as $event) {
            try {
                // Kafka/RabbitMQ-তে publish
                MessageBroker::publish($event->event_type, $event->payload);
                $event->update(['status' => 'PUBLISHED']);
            } catch (\Exception $e) {
                $event->update(['status' => 'FAILED', 'error' => $e->getMessage()]);
            }
        }
    }
}
```

#### Idempotent Transactions

একই operation একাধিকবার execute করলেও result একই থাকে। Retry, message redelivery-তে অত্যন্ত গুরুত্বপূর্ণ।

```php
// Idempotent Transaction — idempotency key ব্যবহার
public function processPayment(string $idempotencyKey, array $paymentData): Payment
{
    return DB::transaction(function () use ($idempotencyKey, $paymentData) {
        // আগে কি এই key দিয়ে process হয়েছে?
        $existing = Payment::where('idempotency_key', $idempotencyKey)->first();
        
        if ($existing) {
            return $existing; // আগের result return, আবার process না করে
        }

        $payment = Payment::create([
            'idempotency_key' => $idempotencyKey,
            'amount'          => $paymentData['amount'],
            'user_id'         => $paymentData['user_id'],
            'status'          => 'COMPLETED',
        ]);

        Wallet::where('user_id', $paymentData['user_id'])
            ->decrement('balance', $paymentData['amount']);

        return $payment;
    });
}
```

```javascript
// Idempotent Transaction — Node.js
async function processPayment(idempotencyKey, paymentData) {
    return sequelize.transaction(async (t) => {
        const existing = await Payment.findOne({
            where: { idempotencyKey },
            transaction: t,
        });

        if (existing) return existing;

        const payment = await Payment.create({
            idempotencyKey,
            ...paymentData,
            status: 'COMPLETED',
        }, { transaction: t });

        await Wallet.decrement('balance', {
            by: paymentData.amount,
            where: { userId: paymentData.userId },
            transaction: t,
        });

        return payment;
    });
}
```

---

## ✅ সুবিধা ও ❌ অসুবিধা (Isolation Level অনুসারে)

| Isolation Level | ✅ সুবিধা | ❌ অসুবিধা |
|---|---|---|
| **READ UNCOMMITTED** | সবচেয়ে দ্রুত, কোনো lock নেই | Dirty read, data corruption risk, আর্থিক লেনদেনে অনুপযোগী |
| **READ COMMITTED** | Dirty read নেই, ভালো performance, বেশিরভাগ application-এর জন্য যথেষ্ট | Non-repeatable read, same query-তে ভিন্ন result, report-এ inconsistency |
| **REPEATABLE READ** | Consistent reads, snapshot isolation, MySQL default | Phantom read (PostgreSQL-এ), বেশি memory (snapshot রাখতে), long TX-এ undo log বড় হয় |
| **SERIALIZABLE** | সর্বোচ্চ safety, কোনো concurrency anomaly নেই | সবচেয়ে ধীর, deadlock/serialization failure বেশি, retry logic দরকার, scalability কম |

---

## ⚠️ সাধারণ ভুল ও সমাধান

### ভুল ১: দীর্ঘ Transaction (Long-Running Transactions)

```php
// ❌ ভুল: Transaction-এর মধ্যে API call, email পাঠানো
DB::transaction(function () use ($order) {
    $order->update(['status' => 'confirmed']);
    
    // 😱 external API call — 5-30 সেকেন্ড লাগতে পারে!
    $response = Http::post('https://bkash-api.com/charge', [...]);
    
    // 😱 email পাঠানো — আরো দেরি!
    Mail::send(new OrderConfirmation($order));
    
    Payment::create([...]);
});
// এই সময়ে row lock ধরে রাখছে — অন্য সব request block!

// ✅ সঠিক: Transaction ছোট রাখুন, external call বাইরে
DB::transaction(function () use ($order) {
    $order->update(['status' => 'confirmed']);
    Payment::create([...]);
});

// Transaction-এর বাইরে external calls
$response = Http::post('https://bkash-api.com/charge', [...]);
Mail::send(new OrderConfirmation($order));
// অথবা queue-তে পাঠান: dispatch(new SendOrderEmail($order));
```

### ভুল ২: Rollback ভুলে যাওয়া

```javascript
// ❌ ভুল: rollback ভুলে গেছে
const t = await sequelize.transaction();
try {
    await Order.create({ ... }, { transaction: t });
    await Payment.create({ ... }, { transaction: t });
    await t.commit();
} catch (error) {
    // rollback নেই! 😱
    // connection pool-এ open transaction থেকে যাবে
    // memory leak, lock না ছাড়া — eventual crash
    throw error;
}

// ✅ সঠিক: finally-তে rollback
const t = await sequelize.transaction();
try {
    await Order.create({ ... }, { transaction: t });
    await Payment.create({ ... }, { transaction: t });
    await t.commit();
} catch (error) {
    await t.rollback();
    throw error;
}

// ✅ আরো ভালো: managed transaction ব্যবহার করুন
await sequelize.transaction(async (t) => {
    await Order.create({ ... }, { transaction: t });
    await Payment.create({ ... }, { transaction: t });
    // auto commit/rollback
});
```

### ভুল ৩: Deadlock-প্রবণ Lock Ordering

```php
// ❌ ভুল: inconsistent lock order → deadlock
// Request A: lock user 1, then user 2
// Request B: lock user 2, then user 1
function transfer($fromId, $toId, $amount) {
    DB::transaction(function () use ($fromId, $toId, $amount) {
        $from = Wallet::where('user_id', $fromId)->lockForUpdate()->first();
        $to   = Wallet::where('user_id', $toId)->lockForUpdate()->first();
        // Deadlock! 💀
    });
}

// ✅ সঠিক: সবসময় consistent order-এ lock নিন
function transfer($fromId, $toId, $amount) {
    DB::transaction(function () use ($fromId, $toId, $amount) {
        $ids = collect([$fromId, $toId])->sort()->values();
        
        $wallets = Wallet::whereIn('user_id', $ids)
            ->orderBy('user_id') // consistent order
            ->lockForUpdate()
            ->get()
            ->keyBy('user_id');
        
        if ($wallets[$fromId]->balance < $amount) {
            throw new InsufficientBalanceException();
        }
        
        $wallets[$fromId]->decrement('balance', $amount);
        $wallets[$toId]->increment('balance', $amount);
    }, 3);
}
```

### ভুল ৪: N+1 Query Transaction-এ

```php
// ❌ ভুল: transaction-এ N+1 query — lock বেশিক্ষণ ধরে রাখে
DB::transaction(function () use ($orderIds) {
    foreach ($orderIds as $id) {
        $order = Order::find($id);           // N queries!
        $order->update(['status' => 'shipped']);
    }
});

// ✅ সঠিক: batch update
DB::transaction(function () use ($orderIds) {
    Order::whereIn('id', $orderIds)->update(['status' => 'shipped']);
});
```

### ভুল ৫: Auto-commit ভুল বোঝা

```sql
-- MySQL-এ প্রতিটি statement স্বয়ংক্রিয়ভাবে auto-commit হয়
-- যদি explicit transaction না লেখেন:

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- এটি auto-commit হয়ে গেছে!
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
-- এটিও auto-commit!

-- প্রথমটি সফল আর দ্বিতীয়টি fail হলে → inconsistent state!

-- ✅ সঠিক: explicit transaction ব্যবহার করুন
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

---

## 🧪 টেস্টিং Transactions

### PHPUnit — Laravel Transaction Testing

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\Wallet;
use App\Models\TransferLog;
use App\Services\TransferService;
use Illuminate\Foundation\Testing\RefreshDatabase;

class TransferServiceTest extends TestCase
{
    use RefreshDatabase; // প্রতি test-এ DB migration fresh করে

    public function test_successful_transfer(): void
    {
        // Arrange
        $sender = Wallet::factory()->create(['balance' => 5000]);
        $receiver = Wallet::factory()->create(['balance' => 1000]);
        $service = new TransferService();

        // Act
        $service->transfer($sender->user_id, $receiver->user_id, 2000);

        // Assert
        $this->assertDatabaseHas('wallets', [
            'user_id' => $sender->user_id,
            'balance' => 3000,
        ]);
        $this->assertDatabaseHas('wallets', [
            'user_id' => $receiver->user_id,
            'balance' => 3000,
        ]);
        $this->assertDatabaseCount('transfer_logs', 1);
    }

    public function test_insufficient_balance_rolls_back(): void
    {
        $sender = Wallet::factory()->create(['balance' => 500]);
        $receiver = Wallet::factory()->create(['balance' => 1000]);
        $service = new TransferService();

        $this->expectException(\App\Exceptions\InsufficientBalanceException::class);
        $service->transfer($sender->user_id, $receiver->user_id, 2000);

        // Rollback verify — balance অপরিবর্তিত
        $this->assertDatabaseHas('wallets', [
            'user_id' => $sender->user_id,
            'balance' => 500,
        ]);
        $this->assertDatabaseHas('wallets', [
            'user_id' => $receiver->user_id,
            'balance' => 1000,
        ]);
        $this->assertDatabaseCount('transfer_logs', 0);
    }

    public function test_concurrent_transfers_dont_overdraft(): void
    {
        $sender = Wallet::factory()->create(['balance' => 1000]);
        $receiver1 = Wallet::factory()->create(['balance' => 0]);
        $receiver2 = Wallet::factory()->create(['balance' => 0]);
        $service = new TransferService();

        // সমান্তরালে দুটি transfer — মোট 1200, balance 1000
        $promises = [];
        $exception_count = 0;

        try {
            $service->transfer($sender->user_id, $receiver1->user_id, 700);
        } catch (\Exception $e) {
            $exception_count++;
        }

        try {
            $service->transfer($sender->user_id, $receiver2->user_id, 500);
        } catch (\Exception $e) {
            $exception_count++;
        }

        $sender->refresh();
        $this->assertGreaterThanOrEqual(0, $sender->balance);
        $this->assertGreaterThanOrEqual(1, $exception_count);
    }
}
```

### Jest — Node.js Transaction Testing

```javascript
// __tests__/transfer.test.js
const { sequelize, Wallet, TransferLog } = require('../models');
const TransferService = require('../services/TransferService');

describe('TransferService', () => {
    // প্রতিটি test-এর পরে transaction rollback করে DB clean রাখা
    let transaction;

    beforeEach(async () => {
        transaction = await sequelize.transaction();
    });

    afterEach(async () => {
        await transaction.rollback();
    });

    it('সফল transfer-এ balance সঠিকভাবে আপডেট হয়', async () => {
        const sender = await Wallet.create(
            { userId: 1, balance: 5000 },
            { transaction }
        );
        const receiver = await Wallet.create(
            { userId: 2, balance: 1000 },
            { transaction }
        );

        await TransferService.transfer(1, 2, 2000, transaction);

        await sender.reload({ transaction });
        await receiver.reload({ transaction });

        expect(sender.balance).toBe(3000);
        expect(receiver.balance).toBe(3000);

        const logs = await TransferLog.findAll({
            where: { senderId: 1 },
            transaction,
        });
        expect(logs).toHaveLength(1);
        expect(logs[0].amount).toBe(2000);
    });

    it('অপর্যাপ্ত balance-এ transfer rollback হয়', async () => {
        await Wallet.create({ userId: 1, balance: 500 }, { transaction });
        await Wallet.create({ userId: 2, balance: 1000 }, { transaction });

        await expect(
            TransferService.transfer(1, 2, 2000, transaction)
        ).rejects.toThrow('অপর্যাপ্ত ব্যালেন্স');

        const sender = await Wallet.findOne({
            where: { userId: 1 },
            transaction,
        });
        expect(sender.balance).toBe(500); // rollback — অপরিবর্তিত
    });

    it('idempotent key দিয়ে duplicate charge হয় না', async () => {
        await Wallet.create({ userId: 1, balance: 5000 }, { transaction });
        const key = 'pay-abc-123';

        const result1 = await TransferService.idempotentCharge(
            key, { userId: 1, amount: 1000 }, transaction
        );
        const result2 = await TransferService.idempotentCharge(
            key, { userId: 1, amount: 1000 }, transaction
        );

        // একই result, দুবার charge হয়নি
        expect(result1.id).toBe(result2.id);

        const wallet = await Wallet.findOne({
            where: { userId: 1 },
            transaction,
        });
        expect(wallet.balance).toBe(4000); // 1000 একবারই কমেছে
    });
});
```

---

## 📋 সারসংক্ষেপ

### Isolation Level তুলনা ম্যাট্রিক্স

```
┌──────────────────┬────────────┬──────────────┬────────────────┬──────────────┐
│                  │   READ     │    READ      │   REPEATABLE   │              │
│  সমস্যা          │UNCOMMITTED │  COMMITTED   │     READ       │ SERIALIZABLE │
├──────────────────┼────────────┼──────────────┼────────────────┼──────────────┤
│ Dirty Read       │    ⚠️      │      ✅      │      ✅        │      ✅      │
│ Non-Repeatable   │    ⚠️      │      ⚠️      │      ✅        │      ✅      │
│ Phantom Read     │    ⚠️      │      ⚠️      │    ⚠️/✅*      │      ✅      │
│ Lost Update      │    ⚠️      │      ⚠️      │      ✅        │      ✅      │
│ Write Skew       │    ⚠️      │      ⚠️      │      ⚠️        │      ✅      │
├──────────────────┼────────────┼──────────────┼────────────────┼──────────────┤
│ Performance      │   ⚡⚡⚡⚡   │    ⚡⚡⚡     │     ⚡⚡        │      ⚡      │
│ Lock Contention  │    কম      │     কম       │     মাঝারি     │     বেশি     │
│ Default In       │    —       │  PostgreSQL  │  MySQL InnoDB  │      —       │
├──────────────────┼────────────┼──────────────┼────────────────┼──────────────┤
│ ব্যবহার          │ Analytics  │ General OLTP │ Financial apps │ Critical     │
│                  │ Reporting  │ Web apps     │ Inventory      │ banking,     │
│                  │ (approx.)  │ (most cases) │ management     │ booking      │
└──────────────────┴────────────┴──────────────┴────────────────┴──────────────┘

✅ = সমস্যা প্রতিরোধ করে    ⚠️ = সমস্যা হতে পারে
* MySQL InnoDB-তে REPEATABLE READ-এ gap lock দিয়ে phantom আটকায়
```

### সিদ্ধান্ত গাইড — বাংলাদেশ Context

| ব্যবসায়িক দৃশ্যপট | Isolation Level | Locking Strategy | Pattern |
|---|---|---|---|
| **bKash/Nagad Transfer** | REPEATABLE READ + lockForUpdate | Pessimistic | Unit of Work |
| **ব্যাংক লেনদেন** | SERIALIZABLE | Pessimistic | 2PC / Saga |
| **E-commerce Order** (Daraz) | READ COMMITTED | Optimistic (stock) | Saga + Outbox |
| **Ticket Booking** (Shohoz) | REPEATABLE READ | Pessimistic (FOR UPDATE SKIP LOCKED) | Queue-based |
| **Blog/CMS Update** | READ COMMITTED | Optimistic (version) | Simple TX |
| **Analytics/Reporting** | READ UNCOMMITTED / READ COMMITTED | None | Read-only TX |
| **Inventory Management** | REPEATABLE READ | Pessimistic | Unit of Work |
| **Multi-service Order** | READ COMMITTED (per service) | Mixed | Saga + Outbox |

### মূল নীতিমালা

1. **Transaction যতটা সম্ভব ছোট রাখুন** — lock ধরে রাখার সময় কমান
2. **External call transaction-এর বাইরে রাখুন** — API, email, file upload
3. **Consistent lock ordering** মেনে চলুন — deadlock এড়াতে
4. **Managed transaction ব্যবহার করুন** — auto commit/rollback
5. **Retry logic রাখুন** — deadlock ও serialization failure-র জন্য
6. **Idempotency key ব্যবহার করুন** — duplicate operation এড়াতে
7. **Monitor করুন** — long-running transaction, lock wait, deadlock frequency
8. **Test করুন** — concurrent scenario, rollback behaviour, edge cases

---

> **💡 মনে রাখবেন:** Transaction হলো data integrity-র ভিত্তি। সঠিক isolation level, locking strategy, এবং error handling ছাড়া production-এ data corruption অনিবার্য। বাংলাদেশের fintech ecosystem (bKash, Nagad, bank integration) এ transaction management-এ কোনো আপোষ চলে না — কারণ এখানে প্রতিটি ভুলের মূল্য সরাসরি মানুষের টাকা।
