# 🔄 মাইগ্রেশন স্ট্র্যাটেজি — Database Migration Strategies

## 📌 সংজ্ঞা ও মূল ধারণা

### Database Migration কী?

Database migration হলো database schema বা data-কে একটি known state থেকে আরেকটি known state-এ নিয়ে যাওয়ার controlled, versioned process। এটিকে আপনি database-এর "Git" হিসেবে ভাবতে পারেন — প্রতিটি migration file একটি commit-এর মতো, যেটি schema-র একটি নির্দিষ্ট পরিবর্তন রেকর্ড করে।

Production-এ bKash-এর মতো সিস্টেমে প্রতিদিন কোটি কোটি transaction হয়। এখানে একটি ভুল migration পুরো payment system বন্ধ করে দিতে পারে। তাই migration strategy বোঝা senior engineer-দের জন্য অত্যন্ত গুরুত্বপূর্ণ।

### Schema Migration vs Data Migration

```
┌──────────────────────────────────────────────────────────────────┐
│              Migration Types                                      │
├────────────────────────┬─────────────────────────────────────────┤
│   Schema Migration     │   Data Migration                        │
├────────────────────────┼─────────────────────────────────────────┤
│ - Column add/remove    │ - Backfilling existing rows             │
│ - Index creation       │ - Data format transformation            │
│ - FK constraints       │ - Merging/splitting tables              │
│ - Table rename         │ - ETL processes                         │
│ - Type change          │ - Cross-database data transfer          │
│                        │                                         │
│ DDL statements         │ DML statements (INSERT/UPDATE/DELETE)   │
│ সাধারণত দ্রুত          │ বড় table-এ অনেক সময় লাগে              │
│ সাধারণত reversible     │ প্রায়ই irreversible                     │
└────────────────────────┴─────────────────────────────────────────┘
```

### Version Control for Database (Migration Files)

প্রতিটি migration file-এ একটি unique identifier (timestamp বা version number) থাকে। Database-এ একটি tracking table থাকে যেটি রেকর্ড রাখে কোন migration run হয়েছে।

```sql
-- Migration tracking table (অভ্যন্তরীণভাবে framework-গুলো এটি maintain করে)
CREATE TABLE schema_migrations (
    id SERIAL PRIMARY KEY,
    migration VARCHAR(255) NOT NULL UNIQUE,
    batch INT NOT NULL,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- উদাহরণ ডেটা:
-- | migration                              | batch | executed_at         |
-- | 2024_01_15_000001_create_users_table    | 1     | 2024-01-15 10:30:00 |
-- | 2024_01_15_000002_create_orders_table   | 1     | 2024-01-15 10:30:01 |
-- | 2024_01_20_000001_add_phone_to_users    | 2     | 2024-01-20 14:00:00 |
```

### Idempotent Migrations

Idempotent migration মানে — একই migration বারবার চালালেও result একই থাকবে, error হবে না। Production-এ এটি অত্যন্ত গুরুত্বপূর্ণ, কারণ network failure বা partial execution-এর পর migration পুনরায় চালাতে হতে পারে।

```sql
-- ❌ Non-idempotent (দ্বিতীয়বার চালালে error হবে)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- ✅ Idempotent (নিরাপদে বারবার চালানো যায়)
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

-- PostgreSQL-এ index-এর জন্য:
CREATE INDEX IF NOT EXISTS idx_users_phone ON users(phone);

-- MySQL-এ conditional check:
SET @exist := (SELECT COUNT(*) FROM information_schema.columns 
               WHERE table_name = 'users' AND column_name = 'phone');
SET @sqlstmt := IF(@exist > 0, 'SELECT "Column already exists"',
                'ALTER TABLE users ADD COLUMN phone VARCHAR(20)');
PREPARE stmt FROM @sqlstmt;
EXECUTE stmt;
```

---

## 📊 Migration Workflow ডায়াগ্রাম

### Development → Staging → Production Flow

```
                    Migration Workflow (CI/CD Pipeline)
═══════════════════════════════════════════════════════════════════

  Developer          Git Repository        CI/CD Pipeline
  ─────────          ──────────────        ──────────────
      │                    │                      │
      │  php artisan       │                      │
      │  make:migration    │                      │
      ├──────────────►     │                      │
      │                    │                      │
      │  git push          │                      │
      ├───────────────────►│                      │
      │                    │   webhook trigger    │
      │                    ├─────────────────────►│
      │                    │                      │
      │                    │              ┌───────┴───────┐
      │                    │              │  STAGE 1:     │
      │                    │              │  Build & Test │
      │                    │              │               │
      │                    │              │  - Unit tests │
      │                    │              │  - migrate    │
      │                    │              │    :fresh     │
      │                    │              │  - Rollback   │
      │                    │              │    test       │
      │                    │              └───────┬───────┘
      │                    │                      │ ✅ Pass
      │                    │              ┌───────┴───────┐
      │                    │              │  STAGE 2:     │
      │                    │              │  Staging DB   │
      │                    │              │               │
      │                    │              │  - Run        │
      │                    │              │    migration  │
      │                    │              │  - Smoke test │
      │                    │              │  - Check lock │
      │                    │              │    duration   │
      │                    │              └───────┬───────┘
      │                    │                      │ ✅ Pass
      │                    │              ┌───────┴───────┐
      │                    │              │  STAGE 3:     │
      │                    │              │  Production   │
      │                    │              │               │
      │                    │              │  - Backup DB  │
      │                    │              │  - Run        │
      │                    │              │    migration  │
      │                    │              │  - Verify     │
      │                    │              │  - Monitor    │
      │                    │              └───────────────┘
```

### Migration State Machine

```
    ┌─────────┐    migrate     ┌──────────┐    migrate     ┌──────────┐
    │ State 0 │───────────────►│ State 1  │───────────────►│ State 2  │
    │ (v0)    │                │ (v1)     │                │ (v2)     │
    │         │◄───────────────│          │◄───────────────│          │
    └─────────┘    rollback    └──────────┘    rollback    └──────────┘

    প্রতিটি state = database-এর একটি known, valid configuration
    প্রতিটি transition = একটি migration file (up/down)
```

---

## 💻 Migration Fundamentals

### Laravel Migrations

#### Migration তৈরি

```bash
# Basic migration
php artisan make:migration create_products_table

# Table specify করে
php artisan make:migration create_products_table --create=products

# Existing table modify করতে
php artisan make:migration add_discount_to_products_table --table=products
```

#### Schema Builder — সম্পূর্ণ উদাহরণ

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

// Laravel 9+ Anonymous Migration (class name collision এড়ানো যায়)
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            // Primary Key
            $table->id(); // BIGINT UNSIGNED AUTO_INCREMENT
            $table->uuid('uuid')->unique();

            // String Types
            $table->string('name', 255);
            $table->string('slug', 255)->unique();
            $table->string('sku', 100)->unique();
            $table->text('description')->nullable();
            $table->longText('specifications')->nullable();

            // Numeric Types
            $table->decimal('price', 12, 2);           // বাংলাদেশী মুদ্রায় বড় অঙ্ক
            $table->decimal('discount_price', 12, 2)->nullable();
            $table->unsignedInteger('stock_quantity')->default(0);
            $table->unsignedSmallInteger('min_order_qty')->default(1);
            $table->float('weight_kg', 8, 2)->nullable();

            // Boolean & Enum
            $table->boolean('is_active')->default(true);
            $table->boolean('is_featured')->default(false);
            $table->enum('status', ['draft', 'published', 'archived'])->default('draft');
            $table->enum('condition', ['new', 'refurbished', 'used'])->default('new');

            // Date & Time
            $table->date('launch_date')->nullable();
            $table->dateTime('sale_starts_at')->nullable();
            $table->dateTime('sale_ends_at')->nullable();
            $table->timestamps();    // created_at, updated_at
            $table->softDeletes();   // deleted_at

            // JSON (MySQL 5.7+, PostgreSQL)
            $table->json('attributes')->nullable();  // {"color": "red", "size": "XL"}
            $table->json('meta_data')->nullable();

            // Foreign Keys
            $table->foreignId('category_id')
                  ->constrained('categories')
                  ->onUpdate('cascade')
                  ->onDelete('restrict');

            $table->foreignId('brand_id')
                  ->nullable()
                  ->constrained()         // brands table infer হবে
                  ->nullOnDelete();       // brand মুছলে NULL হবে

            $table->foreignId('created_by')
                  ->constrained('users')
                  ->onDelete('cascade');

            // Indexes
            $table->index('price');
            $table->index('status');
            $table->index(['category_id', 'status']);           // Composite index
            $table->fullText(['name', 'description']);          // Full-text search (MySQL)

            // Table-level options
            $table->comment('E-commerce product catalog - Daraz/Evaly style');
        });

        // Separate index migration (ভারী table-এ আলাদা migration-এ করা ভালো)
        Schema::table('products', function (Blueprint $table) {
            $table->index('launch_date');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

#### Migration চালানো ও Rollback

```bash
# সব pending migration চালান
php artisan migrate

# নির্দিষ্ট environment-এ
php artisan migrate --env=staging

# Rollback (শেষ batch)
php artisan migrate:rollback

# নির্দিষ্ট সংখ্যক step rollback
php artisan migrate:rollback --step=3

# সব rollback করে আবার migrate
php artisan migrate:refresh

# সব table drop করে আবার migrate (⚠️ শুধু development-এ!)
php artisan migrate:fresh

# Migration status দেখুন
php artisan migrate:status

# Migration squash (অনেক migration file একটিতে merge)
php artisan schema:dump
php artisan schema:dump --prune  # পুরানো file মুছে দেয়
```

#### Migration Squashing

যখন project-এ ২০০+ migration file জমা হয়, তখন squash করা প্রয়োজন। Laravel একটি SQL dump তৈরি করে `database/schema/` ডিরেক্টরিতে রাখে। পরবর্তী `migrate` command প্রথমে সেই dump load করে, তারপর নতুন migration-গুলো চালায়।

---

### Sequelize / Knex Migrations

#### Sequelize Migration

```javascript
// migrations/20240115100000-create-products.js
'use strict';

/** @type {import('sequelize-cli').Migration} */
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('products', {
      id: {
        type: Sequelize.BIGINT.UNSIGNED,
        autoIncrement: true,
        primaryKey: true,
      },
      uuid: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.UUIDV4,
        unique: true,
        allowNull: false,
      },
      name: {
        type: Sequelize.STRING(255),
        allowNull: false,
      },
      slug: {
        type: Sequelize.STRING(255),
        allowNull: false,
        unique: true,
      },
      price: {
        type: Sequelize.DECIMAL(12, 2),
        allowNull: false,
      },
      discount_price: {
        type: Sequelize.DECIMAL(12, 2),
        allowNull: true,
      },
      stock_quantity: {
        type: Sequelize.INTEGER.UNSIGNED,
        defaultValue: 0,
      },
      status: {
        type: Sequelize.ENUM('draft', 'published', 'archived'),
        defaultValue: 'draft',
      },
      is_active: {
        type: Sequelize.BOOLEAN,
        defaultValue: true,
      },
      attributes: {
        type: Sequelize.JSON,
        allowNull: true,
      },
      category_id: {
        type: Sequelize.BIGINT.UNSIGNED,
        allowNull: false,
        references: {
          model: 'categories',
          key: 'id',
        },
        onUpdate: 'CASCADE',
        onDelete: 'RESTRICT',
      },
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
      },
      deleted_at: {
        type: Sequelize.DATE,
        allowNull: true,
      },
    });

    // Indexes আলাদাভাবে যোগ করুন
    await queryInterface.addIndex('products', ['category_id', 'status'], {
      name: 'idx_products_category_status',
    });
    await queryInterface.addIndex('products', ['price'], {
      name: 'idx_products_price',
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable('products');
  },
};
```

#### Knex Migration

```javascript
// migrations/20240115100000_create_products.js

/**
 * @param {import('knex').Knex} knex
 */
exports.up = async function (knex) {
  await knex.schema.createTable('products', (table) => {
    table.bigIncrements('id');
    table.uuid('uuid').notNullable().unique();
    table.string('name', 255).notNullable();
    table.string('slug', 255).notNullable().unique();
    table.decimal('price', 12, 2).notNullable();
    table.decimal('discount_price', 12, 2).nullable();
    table.integer('stock_quantity').unsigned().defaultTo(0);
    table.enu('status', ['draft', 'published', 'archived']).defaultTo('draft');
    table.boolean('is_active').defaultTo(true);
    table.json('attributes').nullable();

    table.bigInteger('category_id').unsigned().notNullable()
      .references('id').inTable('categories')
      .onUpdate('CASCADE').onDelete('RESTRICT');

    table.timestamps(true, true); // created_at, updated_at
    table.timestamp('deleted_at').nullable();

    table.index(['category_id', 'status']);
    table.index('price');
  });
};

/**
 * @param {import('knex').Knex} knex
 */
exports.down = async function (knex) {
  await knex.schema.dropTableIfExists('products');
};
```

#### Programmatic Migration Execution

```javascript
// Sequelize: প্রোগ্রামেটিকভাবে migration চালান
const { Umzug, SequelizeStorage } = require('umzug');

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.js' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});

// সব pending migration চালান
await umzug.up();

// নির্দিষ্ট migration পর্যন্ত rollback
await umzug.down({ to: '20240115100000-create-products' });

// Knex: programmatic execution
await knex.migrate.latest();           // সব pending migrate
await knex.migrate.rollback();         // শেষ batch rollback
await knex.migrate.rollback(true);     // সব rollback
await knex.migrate.status();           // pending migration দেখুন
```

---

### Raw SQL Migrations

#### Numbered SQL Files Approach (Flyway-style)

```
migrations/
├── V001__create_users_table.sql
├── V002__create_products_table.sql
├── V003__add_phone_to_users.sql
├── V004__create_orders_table.sql
└── V005__add_index_on_orders.sql
```

```sql
-- V001__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
```

```sql
-- Migration runner script (custom implementation)
CREATE TABLE IF NOT EXISTS _migrations (
    version VARCHAR(50) PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    checksum VARCHAR(64) NOT NULL,       -- file-এর SHA256 hash
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    execution_time_ms INT
);

-- Migration চালানোর আগে check করুন:
-- 1. version ইতিমধ্যে _migrations table-এ আছে কিনা
-- 2. checksum match করছে কিনা (tamper detection)
-- 3. version order সঠিক কিনা
```

---

## 🔥 Advanced Scenarios

### ১. Zero-Downtime Migrations

#### কেন ALTER TABLE সমস্যা তৈরি করে?

MySQL-এর InnoDB engine-এ অনেক ALTER TABLE operation table-level lock নেয়। bKash-এর মতো সিস্টেমে যেখানে `transactions` table-এ প্রতি সেকেন্ডে হাজার হাজার row insert হচ্ছে, সেখানে ১ মিনিটের lock-ও বিপর্যয় ঘটাতে পারে।

```
⚠️ Lock-creating operations (MySQL):
─────────────────────────────────────
ALTER TABLE transactions ADD COLUMN tax DECIMAL(10,2);  -- 50M rows → minutes of lock
ALTER TABLE users MODIFY COLUMN name VARCHAR(500);       -- full table copy
ALTER TABLE orders ADD INDEX idx_status(status);         -- table lock

✅ Online DDL (MySQL 8.0+, কিছু operation-এ):
───────────────────────────────────────────────
ALTER TABLE users ADD COLUMN phone VARCHAR(20), ALGORITHM=INPLACE, LOCK=NONE;
```

#### Expand-Contract Pattern

এটি zero-downtime migration-এর সবচেয়ে নির্ভরযোগ্য pattern। ধাপে ধাপে পরিবর্তন করা হয়, প্রতিটি ধাপ backward-compatible।

**উদাহরণ: Column rename (`name` → `full_name`)**

```php
// ধাপ ১: EXPAND — নতুন column যোগ করুন (Migration #1)
return new class extends Migration {
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('full_name', 255)->nullable()->after('name');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('full_name');
        });
    }
};

// ধাপ ২: DUAL-WRITE — Application code update
// Model-এ দুই column-ই লেখুন
class User extends Model
{
    protected static function booted(): void
    {
        static::saving(function (User $user) {
            // দুই column-এ একই value রাখুন
            if ($user->isDirty('name')) {
                $user->full_name = $user->name;
            }
            if ($user->isDirty('full_name')) {
                $user->name = $user->full_name;
            }
        });
    }
}

// ধাপ ৩: BACKFILL — পুরানো data কপি করুন (Migration #2)
return new class extends Migration {
    public function up(): void
    {
        // Batched update — একবারে সব update করলে lock হবে
        DB::table('users')
            ->whereNull('full_name')
            ->orderBy('id')
            ->chunk(5000, function ($users) {
                foreach ($users as $user) {
                    DB::table('users')
                        ->where('id', $user->id)
                        ->update(['full_name' => $user->name]);
                }
            });

        // এবার NOT NULL constraint যোগ করুন
        Schema::table('users', function (Blueprint $table) {
            $table->string('full_name', 255)->nullable(false)->change();
        });
    }
};

// ধাপ ৪: SWITCH — Application code-কে নতুন column ব্যবহার করতে দিন
// সব read/write full_name থেকে হবে

// ধাপ ৫: CONTRACT — পুরানো column মুছুন (Migration #3)
return new class extends Migration {
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('name');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('name', 255)->after('id');
        });
        DB::table('users')->update(['name' => DB::raw('full_name')]);
    }
};
```

#### Ghost Tables (gh-ost)

`gh-ost` হলো GitHub-এর তৈরি tool যেটি binary log parsing ব্যবহার করে trigger-less online schema migration করে।

```bash
# gh-ost দিয়ে zero-downtime ALTER
gh-ost \
  --host=db-primary.bkash.internal \
  --database=payments \
  --table=transactions \
  --alter="ADD COLUMN tax_amount DECIMAL(12,2) DEFAULT 0.00" \
  --allow-on-master \
  --chunk-size=1000 \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --initially-drop-ghost-table \
  --execute

# কী করে gh-ost:
# 1. _transactions_gho (ghost table) তৈরি করে নতুন schema সহ
# 2. Binary log থেকে real-time changes ট্র্যাক করে
# 3. পুরানো data chunk-এ chunk-এ copy করে
# 4. সব sync হলে atomic rename করে
# 5. পুরানো table drop করে
```

#### PostgreSQL Concurrent Index Creation

```sql
-- ❌ সাধারণ index — table lock হবে
CREATE INDEX idx_orders_user ON orders(user_id);

-- ✅ Concurrent — lock ছাড়াই index তৈরি হবে
-- (সময় বেশি লাগবে, কিন্তু traffic বাধাগ্রস্ত হবে না)
CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);

-- ⚠️ সতর্কতা: CONCURRENTLY ব্যবহার করলে transaction block-এ চালানো যাবে না
-- Laravel-এ:
-- DB::unprepared('CREATE INDEX CONCURRENTLY ...');
-- এটি migration-এ $withinTransaction = false; সেট করতে হবে
```

```php
// Laravel-এ concurrent index
return new class extends Migration {
    // Transaction বন্ধ করুন — CONCURRENTLY transaction-এ কাজ করে না
    public $withinTransaction = false;

    public function up(): void
    {
        DB::unprepared(
            'CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user_id ON orders(user_id)'
        );
    }

    public function down(): void
    {
        DB::unprepared('DROP INDEX CONCURRENTLY IF EXISTS idx_orders_user_id');
    }
};
```

#### Zero-Downtime Column Type Change

```sql
-- PostgreSQL: INT → BIGINT (বড় table-এ সরাসরি ALTER ঝুঁকিপূর্ণ)

-- ধাপ ১: নতুন column যোগ
ALTER TABLE orders ADD COLUMN id_new BIGINT;

-- ধাপ ২: Trigger দিয়ে নতুন row-তে dual-write
CREATE OR REPLACE FUNCTION sync_order_id_new()
RETURNS TRIGGER AS $$
BEGIN
    NEW.id_new := NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_id
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_order_id_new();

-- ধাপ ৩: পুরানো data backfill (batched)
DO $$
DECLARE
    batch_size INT := 10000;
    max_id BIGINT;
    current_id BIGINT := 0;
BEGIN
    SELECT MAX(id) INTO max_id FROM orders;
    WHILE current_id < max_id LOOP
        UPDATE orders
        SET id_new = id
        WHERE id > current_id AND id <= current_id + batch_size
          AND id_new IS NULL;
        current_id := current_id + batch_size;
        COMMIT;
        PERFORM pg_sleep(0.1);  -- replication lag কমাতে
    END LOOP;
END $$;

-- ধাপ ৪: Swap columns (maintenance window-এ)
ALTER TABLE orders RENAME COLUMN id TO id_old;
ALTER TABLE orders RENAME COLUMN id_new TO id;
-- constraint ও index আপডেট করুন

-- ধাপ ৫: পুরানো column ও trigger মুছুন
DROP TRIGGER trg_sync_order_id ON orders;
DROP FUNCTION sync_order_id_new();
ALTER TABLE orders DROP COLUMN id_old;
```

---

### ২. Large Table Migrations

#### Batched Data Migration

বড় table-এ (৫০ মিলিয়ন+ rows) একবারে সব row update করা সম্ভব নয়। Batch processing অপরিহার্য।

```php
// Laravel: Chunk processing
use Illuminate\Support\Facades\DB;

return new class extends Migration {
    public function up(): void
    {
        Schema::table('transactions', function (Blueprint $table) {
            $table->string('formatted_amount', 50)->nullable();
        });

        // ৫০ মিলিয়ন row-কে ৫০০০-এর batch-এ process করুন
        DB::table('transactions')
            ->orderBy('id')
            ->chunk(5000, function ($transactions) {
                $updates = [];
                foreach ($transactions as $txn) {
                    $updates[] = [
                        'id' => $txn->id,
                        'formatted_amount' => number_format($txn->amount, 2) . ' BDT',
                    ];
                }

                // Bulk upsert
                foreach (array_chunk($updates, 1000) as $batch) {
                    $cases = '';
                    $ids = [];
                    foreach ($batch as $row) {
                        $cases .= "WHEN {$row['id']} THEN '{$row['formatted_amount']}' ";
                        $ids[] = $row['id'];
                    }
                    $idList = implode(',', $ids);
                    DB::statement("
                        UPDATE transactions 
                        SET formatted_amount = CASE id {$cases} END
                        WHERE id IN ({$idList})
                    ");
                }
            });
    }
};
```

```php
// Laravel: LazyCollection (memory-efficient — একটি row-ও RAM-এ জমা হয় না)
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $lastId = 0;
    while (true) {
        $rows = DB::table('transactions')
            ->where('id', '>', $lastId)
            ->orderBy('id')
            ->limit(5000)
            ->get();

        if ($rows->isEmpty()) break;

        foreach ($rows as $row) {
            yield $row;
        }

        $lastId = $rows->last()->id;
    }
})->each(function ($txn) {
    DB::table('transactions')
        ->where('id', $txn->id)
        ->update(['formatted_amount' => number_format($txn->amount, 2) . ' BDT']);
});
```

```javascript
// Sequelize: Streaming large table migration
const { QueryTypes } = require('sequelize');

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.addColumn('transactions', 'formatted_amount', {
      type: Sequelize.STRING(50),
      allowNull: true,
    });

    const BATCH_SIZE = 5000;
    let lastId = 0;
    let hasMore = true;

    while (hasMore) {
      const rows = await queryInterface.sequelize.query(
        `SELECT id, amount FROM transactions WHERE id > :lastId ORDER BY id LIMIT :limit`,
        {
          replacements: { lastId, limit: BATCH_SIZE },
          type: QueryTypes.SELECT,
        }
      );

      if (rows.length === 0) {
        hasMore = false;
        break;
      }

      // Batch update
      const cases = rows
        .map((r) => `WHEN ${r.id} THEN '${Number(r.amount).toFixed(2)} BDT'`)
        .join(' ');
      const ids = rows.map((r) => r.id).join(',');

      await queryInterface.sequelize.query(`
        UPDATE transactions
        SET formatted_amount = CASE id ${cases} END
        WHERE id IN (${ids})
      `);

      lastId = rows[rows.length - 1].id;

      // Replication lag কমাতে sleep
      await new Promise((resolve) => setTimeout(resolve, 100));
    }
  },

  async down(queryInterface) {
    await queryInterface.removeColumn('transactions', 'formatted_amount');
  },
};
```

#### pt-online-schema-change Workflow

```bash
# Percona Toolkit - MySQL-এ large table ALTER-এর জন্য industry standard
pt-online-schema-change \
  --alter="ADD COLUMN tax_amount DECIMAL(12,2) DEFAULT 0.00 AFTER amount" \
  --host=db-primary.internal \
  --user=migration_user \
  --ask-pass \
  --chunk-size=2000 \
  --max-lag=1s \
  --check-interval=1 \
  --recurse=1 \
  --no-drop-old-table \
  D=ecommerce,t=orders \
  --execute

# Workflow:
# 1. _orders_new table তৈরি হয় নতুন schema সহ
# 2. Triggers তৈরি হয় (INSERT, UPDATE, DELETE)
# 3. পুরানো data chunk-এ chunk-এ copy হয়
# 4. Replication lag monitor হয় (--max-lag)
# 5. Atomic RENAME TABLE swap হয়
```

#### pg_repack for PostgreSQL

```bash
# pg_repack: PostgreSQL-এ VACUUM FULL-এর বিকল্প (lock ছাড়াই)
# Table bloat কমায় ও index rebuild করে

pg_repack --table=orders --no-superuser-check -d ecommerce_db

# নির্দিষ্ট index repack:
pg_repack --index=idx_orders_created_at -d ecommerce_db
```

---

### ৩. Data Migrations

#### Backfilling Data

```php
// Laravel: নতুন calculated column backfill
return new class extends Migration {
    public function up(): void
    {
        Schema::table('orders', function (Blueprint $table) {
            $table->decimal('total_with_vat', 14, 2)->nullable();
        });

        // বাংলাদেশে VAT ১৫% (context-specific)
        DB::statement('
            UPDATE orders
            SET total_with_vat = total_amount * 1.15
            WHERE total_with_vat IS NULL
        ');

        Schema::table('orders', function (Blueprint $table) {
            $table->decimal('total_with_vat', 14, 2)->nullable(false)->default(0)->change();
        });
    }

    public function down(): void
    {
        Schema::table('orders', function (Blueprint $table) {
            $table->dropColumn('total_with_vat');
        });
    }
};
```

#### Data Transformation During Migration

```javascript
// Knex: Complex data transformation
// Address string-কে structured JSON-এ রূপান্তর
exports.up = async function (knex) {
  await knex.schema.table('customers', (table) => {
    table.json('address_structured').nullable();
  });

  const BATCH = 2000;
  let offset = 0;

  while (true) {
    const rows = await knex('customers')
      .whereNull('address_structured')
      .whereNotNull('address')
      .orderBy('id')
      .limit(BATCH)
      .offset(offset);

    if (rows.length === 0) break;

    for (const row of rows) {
      // "Mirpur 10, Dhaka 1216" → structured JSON
      const parts = row.address.split(',').map((s) => s.trim());
      const structured = {
        street: parts[0] || '',
        city: parts[1] || '',
        postal_code: parts[2] || '',
        country: 'Bangladesh',
      };

      await knex('customers')
        .where('id', row.id)
        .update({ address_structured: JSON.stringify(structured) });
    }

    offset += BATCH;
  }
};

exports.down = async function (knex) {
  await knex.schema.table('customers', (table) => {
    table.dropColumn('address_structured');
  });
};
```

#### Seed Data vs Migration Data

```
┌───────────────────────────────────┬───────────────────────────────────┐
│        Seed Data                  │        Migration Data             │
├───────────────────────────────────┼───────────────────────────────────┤
│ Test/development data             │ Production-essential data         │
│ Faker-generated                   │ Real business data                │
│ Disposable                        │ Permanent                         │
│ php artisan db:seed               │ Migration-এর মধ্যেই থাকে         │
│ পুনরায় চালানো যায়               │ একবারই চলে                       │
│                                   │                                   │
│ উদাহরণ: ১০০০ fake user           │ উদাহরণ: Division/District list   │
│         Test product catalog      │         Default roles/permissions │
│         Dummy transactions        │         System configuration      │
└───────────────────────────────────┴───────────────────────────────────┘
```

```php
// Migration-এ production-essential data insert
return new class extends Migration {
    public function up(): void
    {
        Schema::create('divisions', function (Blueprint $table) {
            $table->id();
            $table->string('name_en', 100);
            $table->string('name_bn', 100);
            $table->string('code', 10)->unique();
            $table->timestamps();
        });

        // বাংলাদেশের ৮টি বিভাগ — এটি seed নয়, migration data
        DB::table('divisions')->insert([
            ['name_en' => 'Dhaka',       'name_bn' => 'ঢাকা',      'code' => 'DHK', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Chittagong',  'name_bn' => 'চট্টগ্রাম',  'code' => 'CTG', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Rajshahi',    'name_bn' => 'রাজশাহী',    'code' => 'RAJ', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Khulna',      'name_bn' => 'খুলনা',      'code' => 'KHL', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Barisal',     'name_bn' => 'বরিশাল',     'code' => 'BAR', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Sylhet',      'name_bn' => 'সিলেট',      'code' => 'SYL', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Rangpur',     'name_bn' => 'রংপুর',      'code' => 'RNG', 'created_at' => now(), 'updated_at' => now()],
            ['name_en' => 'Mymensingh',  'name_bn' => 'ময়মনসিংহ',  'code' => 'MYM', 'created_at' => now(), 'updated_at' => now()],
        ]);
    }

    public function down(): void
    {
        Schema::dropIfExists('divisions');
    }
};
```

---

### ৪. Multi-tenant Database Migrations

#### Shared Database (tenant_id column) Approach

```php
// সব table-এ tenant_id column — সবচেয়ে সহজ approach
return new class extends Migration {
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->foreignId('tenant_id')->constrained()->onDelete('cascade');
            $table->string('name');
            $table->decimal('price', 12, 2);
            $table->timestamps();

            // tenant_id সব query-তে ব্যবহৃত হবে, তাই composite index জরুরি
            $table->index(['tenant_id', 'created_at']);
            $table->unique(['tenant_id', 'slug']);
        });
    }
};

// Global scope দিয়ে automatic tenant filtering
class Product extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope('tenant', function ($query) {
            $query->where('tenant_id', auth()->user()->tenant_id);
        });
    }
}
```

#### Separate Databases Per Tenant

```php
// প্রতিটি tenant-এর জন্য আলাদা database-তে migration চালান
class TenantMigrationCommand extends Command
{
    protected $signature = 'tenants:migrate {--tenant= : Specific tenant ID}';

    public function handle(): void
    {
        $tenants = $this->option('tenant')
            ? Tenant::where('id', $this->option('tenant'))->get()
            : Tenant::all();

        foreach ($tenants as $tenant) {
            $this->info("Migrating tenant: {$tenant->name} ({$tenant->database})");

            // Dynamic database connection configure করুন
            config([
                'database.connections.tenant' => [
                    'driver'   => 'mysql',
                    'host'     => $tenant->db_host,
                    'database' => $tenant->database,
                    'username' => $tenant->db_user,
                    'password' => decrypt($tenant->db_password),
                ],
            ]);

            DB::purge('tenant');

            Artisan::call('migrate', [
                '--database' => 'tenant',
                '--path'     => 'database/migrations/tenant',
                '--force'    => true,
            ]);

            $this->info("✅ Migration complete for {$tenant->name}");
        }
    }
}
```

#### Schema-Based Multitenancy (PostgreSQL)

```sql
-- PostgreSQL schema-based isolation — প্রতিটি tenant আলাদা schema-তে
CREATE SCHEMA tenant_bkash;
CREATE SCHEMA tenant_nagad;
CREATE SCHEMA tenant_rocket;

-- প্রতিটি schema-তে একই structure
SET search_path TO tenant_bkash;
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    amount DECIMAL(14, 2) NOT NULL,
    type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tenant switch করতে শুধু search_path বদলান
SET search_path TO tenant_nagad;
-- এখন সব query tenant_nagad schema-র table-এ যাবে
```

```php
// Laravel-এ schema-based multitenancy
return new class extends Migration {
    public function up(): void
    {
        $tenants = DB::table('tenants')->pluck('schema_name');

        foreach ($tenants as $schema) {
            DB::statement("SET search_path TO {$schema}");

            Schema::create('invoices', function (Blueprint $table) {
                $table->id();
                $table->foreignId('order_id')->constrained();
                $table->decimal('amount', 14, 2);
                $table->enum('status', ['pending', 'paid', 'cancelled']);
                $table->timestamps();
            });
        }

        DB::statement('SET search_path TO public');
    }
};
```

---

### ৫. Rollback Strategies

#### Reversible vs Irreversible Migrations

```
Reversible (নিরাপদে rollback করা যায়):
───────────────────────────────────────
✅ ADD COLUMN              → DROP COLUMN
✅ CREATE TABLE            → DROP TABLE
✅ ADD INDEX               → DROP INDEX
✅ ADD CONSTRAINT          → DROP CONSTRAINT

Irreversible (rollback-এ data হারাবে):
───────────────────────────────────────
❌ DROP COLUMN             → Column data চিরতরে হারিয়ে যায়
❌ DROP TABLE              → সব row হারিয়ে যায়
❌ TRUNCATE TABLE          → data ফেরত আনা যায় না
❌ Change column type      → precision loss সম্ভব
   (DECIMAL → INT)
```

#### Data-Safe Rollback Pattern

```php
// Production-safe: পুরানো data preserve করে rollback-এর path রাখুন
return new class extends Migration {
    public function up(): void
    {
        // column drop করার আগে data backup রাখুন
        Schema::create('_backup_users_legacy_fields', function (Blueprint $table) {
            $table->unsignedBigInteger('user_id')->primary();
            $table->string('old_username', 100)->nullable();
            $table->string('old_display_name', 100)->nullable();
            $table->timestamp('backed_up_at')->useCurrent();
        });

        DB::statement('
            INSERT INTO _backup_users_legacy_fields (user_id, old_username, old_display_name)
            SELECT id, username, display_name FROM users
            WHERE username IS NOT NULL OR display_name IS NOT NULL
        ');

        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['username', 'display_name']);
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('username', 100)->nullable();
            $table->string('display_name', 100)->nullable();
        });

        // Backup থেকে data restore করুন
        DB::statement('
            UPDATE users u
            JOIN _backup_users_legacy_fields b ON u.id = b.user_id
            SET u.username = b.old_username,
                u.display_name = b.old_display_name
        ');

        Schema::dropIfExists('_backup_users_legacy_fields');
    }
};
```

#### Feature Flags + Migration Rollback

```php
// Feature flag ব্যবহার করে migration-এর impact নিয়ন্ত্রণ করুন
class OrderService
{
    public function calculateTotal(Order $order): float
    {
        // Feature flag check — migration rollback হলে পুরানো logic-এ ফিরে যান
        if (Feature::active('new-vat-calculation')) {
            // নতুন column ব্যবহার করে (migration #42)
            return $order->subtotal + $order->vat_amount + $order->delivery_charge;
        }

        // পুরানো calculation (migration rollback-safe)
        return $order->subtotal * 1.15 + $order->delivery_charge;
    }
}
```

---

### ৬. Database Seeding

#### Laravel Factory Pattern

```php
// database/factories/ProductFactory.php
class ProductFactory extends Factory
{
    protected $model = Product::class;

    public function definition(): array
    {
        return [
            'name'           => $this->faker->words(3, true),
            'slug'           => $this->faker->unique()->slug(),
            'sku'            => 'SKU-' . $this->faker->unique()->numerify('######'),
            'price'          => $this->faker->randomFloat(2, 50, 50000),
            'stock_quantity' => $this->faker->numberBetween(0, 1000),
            'status'         => $this->faker->randomElement(['draft', 'published', 'archived']),
            'is_active'      => $this->faker->boolean(80),
            'description'    => $this->faker->paragraph(),
            'attributes'     => [
                'color' => $this->faker->safeColorName(),
                'size'  => $this->faker->randomElement(['S', 'M', 'L', 'XL']),
            ],
        ];
    }

    // State methods — নির্দিষ্ট অবস্থায় product তৈরি
    public function published(): static
    {
        return $this->state(['status' => 'published', 'is_active' => true]);
    }

    public function outOfStock(): static
    {
        return $this->state(['stock_quantity' => 0]);
    }

    public function expensive(): static
    {
        return $this->state(['price' => $this->faker->randomFloat(2, 10000, 100000)]);
    }
}

// Seeder-এ ব্যবহার
class ProductSeeder extends Seeder
{
    public function run(): void
    {
        // ১০০০টি সাধারণ product
        Product::factory()->count(1000)->create();

        // ২০০টি published, expensive product
        Product::factory()->count(200)->published()->expensive()->create();

        // Category সহ product
        Category::factory()
            ->count(20)
            ->has(Product::factory()->count(50))
            ->create();
    }
}
```

#### Sequelize Seeders

```javascript
// seeders/20240115-demo-products.js
'use strict';
const { faker } = require('@faker-js/faker');

module.exports = {
  async up(queryInterface) {
    const products = Array.from({ length: 500 }, (_, i) => ({
      uuid: faker.string.uuid(),
      name: faker.commerce.productName(),
      slug: faker.helpers.slugify(faker.commerce.productName()) + '-' + i,
      price: faker.commerce.price({ min: 100, max: 50000, dec: 2 }),
      stock_quantity: faker.number.int({ min: 0, max: 1000 }),
      status: faker.helpers.arrayElement(['draft', 'published', 'archived']),
      is_active: faker.datatype.boolean(0.8),
      attributes: JSON.stringify({
        color: faker.color.human(),
        weight: faker.number.float({ min: 0.1, max: 50, fractionDigits: 2 }),
      }),
      category_id: faker.number.int({ min: 1, max: 20 }),
      created_at: new Date(),
      updated_at: new Date(),
    }));

    // Bulk insert (chunk করে)
    const chunkSize = 100;
    for (let i = 0; i < products.length; i += chunkSize) {
      await queryInterface.bulkInsert(
        'products',
        products.slice(i, i + chunkSize)
      );
    }
  },

  async down(queryInterface) {
    await queryInterface.bulkDelete('products', null, {});
  },
};
```

#### Idempotent Seeding

```php
// Production seed — বারবার চালালেও duplicate হবে না
class RolePermissionSeeder extends Seeder
{
    public function run(): void
    {
        $roles = [
            ['name' => 'admin',    'guard_name' => 'web'],
            ['name' => 'merchant', 'guard_name' => 'web'],
            ['name' => 'customer', 'guard_name' => 'web'],
            ['name' => 'support',  'guard_name' => 'web'],
        ];

        foreach ($roles as $role) {
            // updateOrCreate — থাকলে update, না থাকলে create
            Role::updateOrCreate(
                ['name' => $role['name']],
                $role
            );
        }

        $permissions = [
            'manage-products', 'view-orders', 'manage-orders',
            'manage-users', 'view-reports', 'manage-settings',
        ];

        foreach ($permissions as $permission) {
            Permission::firstOrCreate(['name' => $permission, 'guard_name' => 'web']);
        }

        // Role-Permission mapping
        $rolePermissions = [
            'admin'    => $permissions,
            'merchant' => ['manage-products', 'view-orders', 'manage-orders'],
            'customer' => ['view-orders'],
            'support'  => ['view-orders', 'view-reports'],
        ];

        foreach ($rolePermissions as $roleName => $perms) {
            Role::findByName($roleName)->syncPermissions($perms);
        }
    }
}
```

---

### ৭. Cross-Database Migration

#### Dual-Write Pattern

MySQL থেকে PostgreSQL-এ migrate করার সময় dual-write pattern ব্যবহার করলে zero-downtime transition সম্ভব।

```
Phase 1: MySQL only (current state)
─────────────────────────────────────
  App ──write──► MySQL ──read──► App

Phase 2: Dual-write, read from MySQL
─────────────────────────────────────
  App ──write──► MySQL ──read──► App
    └──write──► PostgreSQL

Phase 3: Dual-write, read from PostgreSQL
─────────────────────────────────────────
  App ──write──► MySQL
    └──write──► PostgreSQL ──read──► App

Phase 4: PostgreSQL only (target state)
───────────────────────────────────────
  App ──write──► PostgreSQL ──read──► App
```

```javascript
// Dual-write service implementation
class DualWriteRepository {
  constructor(mysqlPool, pgPool, readFrom = 'mysql') {
    this.mysql = mysqlPool;
    this.pg = pgPool;
    this.readFrom = readFrom;
  }

  async createOrder(order) {
    // দুই database-এই লিখুন
    const [mysqlResult, pgResult] = await Promise.allSettled([
      this.mysql.query(
        'INSERT INTO orders (user_id, total, status) VALUES (?, ?, ?)',
        [order.userId, order.total, order.status]
      ),
      this.pg.query(
        'INSERT INTO orders (user_id, total, status) VALUES ($1, $2, $3) RETURNING id',
        [order.userId, order.total, order.status]
      ),
    ]);

    // Primary write fail হলে error throw করুন
    if (this.readFrom === 'mysql' && mysqlResult.status === 'rejected') {
      throw mysqlResult.reason;
    }
    if (this.readFrom === 'pg' && pgResult.status === 'rejected') {
      throw pgResult.reason;
    }

    // Secondary write fail হলে retry queue-তে পাঠান
    if (mysqlResult.status === 'rejected' || pgResult.status === 'rejected') {
      await this.queueForRetry('createOrder', order);
    }

    return this.readFrom === 'mysql'
      ? mysqlResult.value
      : pgResult.value;
  }

  async findOrder(id) {
    const pool = this.readFrom === 'mysql' ? this.mysql : this.pg;
    const query = this.readFrom === 'mysql'
      ? 'SELECT * FROM orders WHERE id = ?'
      : 'SELECT * FROM orders WHERE id = $1';
    return pool.query(query, [id]);
  }
}
```

#### Data Validation After Migration

```sql
-- Migration-পরবর্তী data validation queries

-- ১. Row count match
SELECT 'MySQL' as source, COUNT(*) as total FROM mysql_db.orders
UNION ALL
SELECT 'PostgreSQL', COUNT(*) FROM pg_db.orders;

-- ২. Checksum verification (PostgreSQL)
SELECT MD5(STRING_AGG(
    COALESCE(id::text, '') || '|' ||
    COALESCE(user_id::text, '') || '|' ||
    COALESCE(total::text, '') || '|' ||
    COALESCE(status, ''),
    ',' ORDER BY id
)) AS checksum
FROM orders;

-- ৩. Sample-based comparison
-- ১০০০টি random row-এর data মিলিয়ে দেখুন
SELECT id, user_id, total, status, created_at
FROM orders
ORDER BY RANDOM()
LIMIT 1000;
```

---

### ৮. Migration in Microservices

#### প্রতিটি Service নিজের Migration নিজে পরিচালনা করে

```
┌─────────────────────────────────────────────────────────┐
│                  Microservice Architecture               │
├──────────────┬──────────────┬──────────────┬────────────┤
│ User Service │ Order Service│ Payment Svc  │ Product Svc│
│              │              │              │            │
│ users DB     │ orders DB    │ payments DB  │ products DB│
│ ├─ migration │ ├─ migration │ ├─ migration │├─ migration│
│ │  v001      │ │  v001      │ │  v001      ││  v001     │
│ │  v002      │ │  v002      │ │  v002      ││  v002     │
│ │  v003      │ │  v003      │              ││  v003     │
│ └────────────│ └────────────│ └────────────│└───────────│
└──────────────┴──────────────┴──────────────┴────────────┘

নিয়ম: কোনো service অন্য service-এর database সরাসরি access করবে না
যোগাযোগ হবে API বা Event-এর মাধ্যমে
```

#### Event-Driven Schema Evolution

```javascript
// Order Service — schema change হলে event publish করুন

// Migration: orders table-এ delivery_zone column যোগ
exports.up = async function (knex) {
  await knex.schema.table('orders', (table) => {
    table.string('delivery_zone', 50).nullable();
  });
};

// Application: নতুন field-সহ event publish
class OrderService {
  async createOrder(data) {
    const order = await this.orderRepo.create(data);

    // Event-এ schema version include করুন
    await this.eventBus.publish('order.created', {
      schemaVersion: 3, // ← consumer-কে জানান কোন version
      payload: {
        orderId: order.id,
        userId: order.userId,
        total: order.total,
        deliveryZone: order.deliveryZone, // নতুন field
      },
    });
  }
}

// Payment Service — backward-compatible consumer
class PaymentEventHandler {
  async handleOrderCreated(event) {
    // Schema version check — নতুন field না থাকলেও কাজ করবে
    const { payload, schemaVersion } = event;

    const paymentData = {
      orderId: payload.orderId,
      amount: payload.total,
    };

    // নতুন field optional হিসেবে handle করুন
    if (schemaVersion >= 3 && payload.deliveryZone) {
      paymentData.deliveryZone = payload.deliveryZone;
    }

    await this.processPayment(paymentData);
  }
}
```

---

## ✅ Migration Best Practices Checklist

```
Production Migration চালানোর আগে এই checklist অনুসরণ করুন:

□  Migration staging-এ সফলভাবে চলেছে
□  Rollback (down method) পরীক্ষা করা হয়েছে
□  Large table-এ lock duration পরিমাপ করা হয়েছে
□  Database backup নেওয়া হয়েছে
□  Maintenance window schedule করা হয়েছে (প্রয়োজনে)
□  Team-কে জানানো হয়েছে
□  Monitoring/alerting সেটআপ করা হয়েছে
□  Rollback plan documented আছে
□  Migration idempotent কিনা verify করা হয়েছে
□  Foreign key ও index impact বিশ্লেষণ করা হয়েছে
□  Disk space পর্যাপ্ত আছে (ALTER TABLE অতিরিক্ত space লাগে)
□  Replication lag monitoring চালু আছে
□  Application code backward-compatible (expand-contract)
□  Feature flags যথাস্থানে আছে
□  Post-migration verification query প্রস্তুত আছে
```

---

## ⚠️ সাধারণ ভুল

### ১. Destructive Migration in Production

```php
// ❌ মারাত্মক ভুল — production-এ data হারাবে
public function up(): void
{
    Schema::dropIfExists('user_sessions');  // ❌ সব session data হারাবে!

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('phone');        // ❌ phone number হারাবে!
    });
}

// ✅ সঠিক approach — আগে deprecate, পরে remove
// Migration #1: Column-কে nullable করুন, application code থেকে usage সরান
// Migration #2: ২ সপ্তাহ পর column drop করুন (backup নিয়ে)
```

### ২. Rollback Method না লেখা

```php
// ❌ down() ফাঁকা — rollback করতে পারবেন না
public function down(): void
{
    // Nothing here...
}

// ✅ সবসময় meaningful rollback লিখুন
public function down(): void
{
    Schema::table('orders', function (Blueprint $table) {
        $table->dropColumn('delivery_zone');
    });
}
```

### ৩. Untested Migration

```php
// ❌ সরাসরি production-এ migration push করা
// ✅ CI/CD pipeline-এ এই test-গুলো mandatory করুন:
// - migrate:fresh (শুরু থেকে সব migration চলে কিনা)
// - migrate:rollback (rollback কাজ করে কিনা)
// - Data integrity check (FK constraints সঠিক কিনা)
```

### ৪. Migration-এ Business Logic রাখা

```php
// ❌ Migration file-এ complex business logic
public function up(): void
{
    // Eloquent model ব্যবহার — model class পরে change হলে migration break হবে
    User::where('type', 'premium')->each(function ($user) {
        $user->subscription()->create([...]);
    });
}

// ✅ Raw query ব্যবহার করুন — model dependency নেই
public function up(): void
{
    DB::statement("
        INSERT INTO subscriptions (user_id, plan, created_at)
        SELECT id, 'premium', NOW() FROM users WHERE type = 'premium'
    ");
}
```

### ৫. একটি বিশাল Migration File

```php
// ❌ একটি migration-এ ২০টি table তৈরি ও ডেটা migration
// সমস্যা: rollback-এ কোনটা fail হলো বোঝা যায় না

// ✅ ছোট, focused migration file তৈরি করুন:
// 2024_01_15_000001_create_categories_table.php
// 2024_01_15_000002_create_products_table.php
// 2024_01_15_000003_add_fk_products_categories.php
// 2024_01_15_000004_seed_default_categories.php
```

---

## 🧪 টেস্টিং Migrations

### CI/CD Pipeline-এ Migration Test

```yaml
# .github/workflows/migration-test.yml
name: Migration Test

on: [push, pull_request]

jobs:
  migration-test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: test_db
          MYSQL_ROOT_PASSWORD: secret
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Fresh Migration Test
        run: |
          php artisan migrate:fresh --env=testing
          echo "✅ All migrations ran successfully from scratch"

      - name: Rollback Test
        run: |
          php artisan migrate:rollback --step=999 --env=testing
          echo "✅ All rollbacks completed successfully"

      - name: Re-migrate Test
        run: |
          php artisan migrate --env=testing
          echo "✅ Re-migration successful (idempotency verified)"

      - name: Seed Test
        run: |
          php artisan db:seed --env=testing
          echo "✅ Seeding completed successfully"
```

### PHPUnit RefreshDatabase Trait

```php
// tests/Feature/ProductMigrationTest.php
namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ProductMigrationTest extends TestCase
{
    // প্রতিটি test-এর আগে migrate:fresh চলবে, transaction-এ wrap হবে
    use RefreshDatabase;

    public function test_products_table_has_required_columns(): void
    {
        $this->assertTrue(Schema::hasTable('products'));
        $this->assertTrue(Schema::hasColumns('products', [
            'id', 'uuid', 'name', 'slug', 'price',
            'stock_quantity', 'status', 'category_id',
        ]));
    }

    public function test_products_table_has_correct_indexes(): void
    {
        // Index existence check via raw query
        $indexes = DB::select("SHOW INDEX FROM products WHERE Key_name = 'idx_products_category_status'");
        $this->assertNotEmpty($indexes);
    }

    public function test_foreign_key_constraint_works(): void
    {
        $this->expectException(\Illuminate\Database\QueryException::class);

        // Nonexistent category_id — FK violation হবে
        DB::table('products')->insert([
            'uuid'        => \Str::uuid(),
            'name'        => 'Test Product',
            'slug'        => 'test-product',
            'price'       => 100.00,
            'category_id' => 99999,
            'created_by'  => 1,
        ]);
    }
}
```

### Jest with Database Setup/Teardown

```javascript
// tests/migrations.test.js
const { Sequelize } = require('sequelize');
const { Umzug, SequelizeStorage } = require('umzug');

describe('Database Migrations', () => {
  let sequelize;
  let umzug;

  beforeAll(async () => {
    sequelize = new Sequelize('sqlite::memory:', { logging: false });
    umzug = new Umzug({
      migrations: { glob: 'migrations/*.js' },
      context: sequelize.getQueryInterface(),
      storage: new SequelizeStorage({ sequelize }),
      logger: undefined,
    });
  });

  afterAll(async () => {
    await sequelize.close();
  });

  test('all migrations run successfully', async () => {
    const migrations = await umzug.up();
    expect(migrations.length).toBeGreaterThan(0);
    migrations.forEach((m) => {
      expect(m.name).toBeDefined();
    });
  });

  test('all migrations rollback successfully', async () => {
    await umzug.up();
    const rolledBack = await umzug.down({ to: 0 });
    expect(rolledBack.length).toBeGreaterThan(0);
  });

  test('migrations are idempotent (re-run safe)', async () => {
    await umzug.up();
    await umzug.down({ to: 0 });
    // দ্বিতীয়বার চালান — error হওয়া উচিত না
    const migrations = await umzug.up();
    expect(migrations.length).toBeGreaterThan(0);
  });

  test('products table has correct structure', async () => {
    await umzug.up();
    const [results] = await sequelize.query(
      "PRAGMA table_info('products')"
    );
    const columnNames = results.map((c) => c.name);
    expect(columnNames).toContain('id');
    expect(columnNames).toContain('name');
    expect(columnNames).toContain('price');
    expect(columnNames).toContain('category_id');
  });
});
```

---

## 📏 Migration Decision Guide

```
আপনার কোন Migration Strategy দরকার?
═══════════════════════════════════════

Table-এ কত row আছে?
│
├─ < 100K rows
│   └─ সরাসরি ALTER TABLE চালান (সাধারণত ১ সেকেন্ডের কম)
│
├─ 100K - 1M rows
│   ├─ MySQL 8.0+ ALGORITHM=INPLACE সমর্থন করলে → Online DDL ব্যবহার করুন
│   └─ না হলে → off-peak hour-এ চালান
│
├─ 1M - 10M rows
│   ├─ Index add → PostgreSQL: CONCURRENTLY; MySQL: pt-online-schema-change
│   ├─ Column add → gh-ost বা pt-online-schema-change
│   └─ Data backfill → Batched migration (chunk 5000-10000)
│
└─ 10M+ rows (bKash-scale)
    ├─ Schema change → gh-ost / pt-online-schema-change (mandatory)
    ├─ Data migration → Background job (queue worker)
    ├─ Column rename → Expand-Contract pattern (multi-deploy)
    └─ Type change → Ghost column + trigger + batched copy

Downtime কি গ্রহণযোগ্য?
│
├─ হ্যাঁ (maintenance window আছে)
│   └─ সরাসরি ALTER চালান, কিন্তু backup ও rollback plan রাখুন
│
└─ না (24/7 system, e.g., bKash, Daraz)
    ├─ Expand-Contract pattern
    ├─ Ghost table tools (gh-ost, pt-osc)
    ├─ Blue-Green deployment
    └─ Feature flag সহ gradual rollout

Multi-tenant?
│
├─ Shared DB → tenant_id column + composite index
├─ Schema per tenant → PostgreSQL schema-based isolation
└─ DB per tenant → Tenant migration command (loop through all DBs)
```

---

## 📋 সারসংক্ষেপ

### মূল বিষয়সমূহ

| বিষয় | মূল কথা |
|--------|---------|
| **Migration কী** | Database-র version control — প্রতিটি পরিবর্তন tracked ও reversible |
| **Schema vs Data** | Schema = structure (DDL), Data = content (DML) — আলাদা strategy দরকার |
| **Idempotent** | `IF NOT EXISTS` / `IF EXISTS` ব্যবহার করুন — বারবার চালানো নিরাপদ করুন |
| **Zero-Downtime** | Expand-Contract + gh-ost/pt-osc — large table-এ mandatory |
| **Rollback** | সবসময় `down()` লিখুন, destructive operation-এ data backup রাখুন |
| **Multi-tenant** | Shared DB সহজতম, Schema-based PostgreSQL-এ ভালো isolation |
| **Microservices** | প্রতিটি service নিজের DB migrate করে, event-driven schema evolution |
| **Testing** | CI/CD-তে fresh migrate → rollback → re-migrate test mandatory |

### bKash/Daraz-Scale System-এ Migration Checklist

```
Production-Ready Migration:
──────────────────────────
1. ✅ Staging-এ test হয়েছে (production-size data সহ)
2. ✅ Rollback plan আছে ও tested
3. ✅ Zero-downtime tool ব্যবহৃত (gh-ost/pt-osc)
4. ✅ Feature flag দিয়ে application code guard করা হয়েছে
5. ✅ Monitoring dashboard-এ migration metric যোগ করা হয়েছে
6. ✅ Replication lag alerting সেট করা হয়েছে
7. ✅ Post-migration data validation query প্রস্তুত
8. ✅ Team communication plan আছে (Slack channel/war room)
9. ✅ Database backup verified ও restore-tested
10. ✅ Gradual rollout plan (canary → percentage → full)
```

> **মনে রাখুন:** Database migration শুধু technical কাজ না — এটি একটি coordinated process যেখানে engineering, ops, এবং business team-কে একসাথে কাজ করতে হয়। বাংলাদেশের fintech ও e-commerce scale-এ একটি ভুল migration কোটি টাকার লেনদেন বন্ধ করে দিতে পারে। তাই plan, test, validate, এবং monitor — প্রতিটি ধাপ গুরুত্বপূর্ণ।
