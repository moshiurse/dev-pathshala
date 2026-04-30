# 🧬 ORM — Object-Relational Mapping গভীর বিশ্লেষণ

> "ORM is the Vietnam of computer science." — Ted Neward (২০০৬)
>
> তবুও আজও ৯০%+ web application ORM ব্যবহার করে। কেন? Productivity। কখন এড়াবেন? Performance critical path-এ।

---

## 📖 সূচিপত্র

- [ORM কী এবং কেন?](#-orm-কী-এবং-কেন)
- [ORM-এর Trade-offs](#-orm-এর-trade-offs)
- [Active Record vs Data Mapper](#-active-record-vs-data-mapper-pattern)
- [ORM vs Query Builder vs Raw SQL](#-orm-vs-query-builder-vs-raw-sql)
- [মূল ম্যাপিং কনসেপ্ট](#-মূল-ম্যাপিং-কনসেপ্ট)
- [Identity Map ও Unit of Work](#-identity-map-ও-unit-of-work)
- [Lazy vs Eager Loading ও N+1 সমস্যা](#-lazy-vs-eager-loading-ও-n1-সমস্যা)
- [Relationships](#-relationships)
- [Migrations ও Schema Management](#-migrations-ও-schema-management)
- [Soft Delete, Timestamps, Observers](#-soft-delete-timestamps-observers)
- [Transactions ও Locking](#-transactions-ও-locking)
- [Performance Tuning](#-performance-tuning)
- [Caching Layers](#-caching-layers)
- [Multi-DB / Read-Replica / Multi-Tenancy](#-multi-db--read-replica--multi-tenancy)
- [ORM Comparison Matrix](#-orm-comparison-matrix)
- [কখন ORM ব্যবহার করবেন না](#-কখন-orm-ব্যবহার-করবেন-না)
- [Anti-patterns](#-anti-patterns)
- [BD Real-world কেস স্টাডি](#-bd-real-world-কেস-স্টাডি)
- [Full Code: PHP (Eloquent + Doctrine)](#-full-code-php-eloquent--doctrine)
- [Full Code: Node.js (Prisma + TypeORM)](#-full-code-nodejs-prisma--typeorm)
- [Checklist](#-checklist)

---

## 📌 ORM কী এবং কেন?

**Object-Relational Mapping (ORM)** হলো এমন একটি লেয়ার যা **relational database-এর row** এবং **OOP language-এর object**-এর মধ্যে স্বয়ংক্রিয় mapping করে দেয়। আপনি SQL লিখার বদলে method call (`User::find(1)`, `prisma.user.findUnique(...)`) করেন; ORM এটিকে SQL-এ রূপান্তরিত করে database-এ পাঠায় এবং ফেরত আসা rows-কে object-এ map করে।

```
┌─────────────────────────────────────────────────────────────────┐
│         Application Code (PHP / TS / Java / Python)             │
│                                                                  │
│   $user = User::find(42);                                       │
│   $user->name = "Rakib";                                        │
│   $user->save();                                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Object ⇄ SQL translation
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ORM Engine                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐    │
│  │ Identity Map│  │ Unit of Work │  │ Query Builder / DSL │    │
│  └─────────────┘  └──────────────┘  └─────────────────────┘    │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐    │
│  │ Hydration   │  │ Lazy Proxies │  │ Migration Runner    │    │
│  └─────────────┘  └──────────────┘  └─────────────────────┘    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ SQL (PDO / pg / mysql2)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         RDBMS (MySQL / PostgreSQL / SQL Server / Oracle)        │
└─────────────────────────────────────────────────────────────────┘
```

### কেন ORM দরকার?

1. **Productivity** — boilerplate SQL/PDO কোড লেখার বদলে business logic-এ ফোকাস।
2. **Type safety** (TS, Java) — compile-time-এ schema mismatch ধরা পড়ে।
3. **Database portability** — MySQL → PostgreSQL switch করতে dialect সমস্যায় পড়তে হয় না।
4. **Security** — parameter binding default থাকায় SQL injection বহুলাংশে বন্ধ।
5. **Maintainability** — domain model এক জায়গায়; refactor সহজ।
6. **Migration tooling** — schema versioning built-in।
7. **Testing** — in-memory SQLite দিয়ে fast unit test।

### কেন কখনো কখনো বিরক্তিকর?

- জেনারেট হওয়া SQL কখনো suboptimal (extra JOIN, SELECT *)।
- "Magic" আচরণে hidden N+1 query।
- Complex analytical query (window function, CTE, recursive) প্রকাশ করা কঠিন।
- Object-graph হঠাৎ অনেক বড় হয়ে memory blow up করে।

---

## ⚖️ ORM-এর Trade-offs

```
                    Productivity
                         ▲
                         │
              Eloquent ●─┤   ● Prisma
                         │
              Sequelize ●┤   ● TypeORM (AR mode)
                         │
                         │   ● Doctrine
                         │   ● TypeORM (DM mode)
                         │
                         │           ● Knex / Drizzle (Query Builder)
                         │
                         │                       ● Raw SQL / PDO
                         │
                         └─────────────────────────────►
                                                    Performance / Control
```

| বিষয় | ORM-এর সুবিধা | ORM-এর অসুবিধা |
|------|---------------|------------------|
| Code volume | ৭০% কম কোড | "Magic" debug কঠিন |
| SQL knowledge | নতুন dev দ্রুত productive | Senior dev SQL ভুলে যায় |
| Performance | 90% query যথেষ্ট ভাল | Hot path-এ ১০-১০০x ধীর হতে পারে |
| Schema change | Migration system সাহায্য করে | ORM model ও DB drift হয় |
| Complex query | Simple CRUD সহজ | OLAP/CTE/recursive কষ্টকর |

---

## 🏛️ Active Record vs Data Mapper Pattern

ORM মূলত দুইটি pattern অনুসরণ করে। Martin Fowler-এর *PoEAA* বইতে দুটোরই বিস্তারিত আছে।

### Active Record (AR)

Object **নিজেই জানে কীভাবে database-এ save হবে**। অর্থাৎ data এবং persistence logic একই class-এ।

```php
// Eloquent (Laravel) - Active Record
$user = new User();
$user->name = "Karim";
$user->phone = "01711-111111";
$user->save();           // object নিজেই DB call করছে

User::where('city', 'Dhaka')->get();   // static query
```

**সুবিধা:** খুবই সহজ, fast prototyping, scaffolding-friendly।
**অসুবিধা:** SRP (Single Responsibility) ভঙ্গ — domain object-এ persistence concern; complex domain-এ unmaintainable।

**উদাহরণ:** Laravel Eloquent, Ruby on Rails AR, Sequelize (default), TypeORM (default), CakePHP ORM।

### Data Mapper (DM)

Domain object **persistence সম্পর্কে কিছু জানে না**। আলাদা **Mapper / EntityManager / Repository** class object ↔ DB রূপান্তর করে।

```php
// Doctrine - Data Mapper
$user = new User();
$user->setName("Karim");
$user->setPhone("01711-111111");

$entityManager->persist($user);    // EntityManager এর হাতে দিলেন
$entityManager->flush();            // এবার DB-তে যাবে
```

**সুবিধা:** POPO/POJO entity, testable, DDD-friendly, separation of concerns।
**অসুবিধা:** বেশি boilerplate, learning curve বেশি।

**উদাহরণ:** Doctrine (PHP), Hibernate (Java/JPA), MikroORM (TS), Prisma (hybrid — closer to DM), TypeORM (`EntityManager` mode)।

### তুলনা

```
┌────────────────────────┬──────────────────────────┬────────────────────────────┐
│       বৈশিষ্ট্য         │     Active Record        │       Data Mapper          │
├────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Entity ও DB সম্পর্ক    │ Tightly coupled          │ Decoupled                  │
│ Save করার দায়িত্ব      │ Object নিজে              │ EntityManager / Repository │
│ Boilerplate            │ কম                       │ বেশি                       │
│ DDD ফিট                │ মাঝারি                   │ চমৎকার                     │
│ Learning curve         │ সহজ                      │ steep                      │
│ Test করা               │ static call mock কঠিন    │ EntityManager mockable     │
│ Bulk update            │ সহজ                      │ DBAL / DQL দরকার           │
│ আদর্শ ব্যবহার          │ CRUD-heavy MVC app       │ Complex domain, microservice│
│ উদাহরণ                 │ Eloquent, Rails AR       │ Doctrine, Hibernate        │
└────────────────────────┴──────────────────────────┴────────────────────────────┘
```

---

## ⚙️ ORM vs Query Builder vs Raw SQL

প্রতিটি স্তরের নিজস্ব ভূমিকা আছে। একটি সিরিয়াস সিস্টেমে তিনটিই থাকতে পারে।

```
┌───────────┐   উচ্চ abstraction
│   ORM     │   - Object hydration
│ (Eloquent)│   - Relationships, lazy loading
└─────┬─────┘   - Migrations, observers
      │
┌─────▼──────┐
│ Query      │  - Fluent SQL DSL
│ Builder    │  - কোনো object mapping নেই
│ (Knex)     │  - Raw SQL এর প্রায় সমান power
└─────┬──────┘
      │
┌─────▼──────┐  নিম্ন abstraction
│ Raw SQL /  │  - সর্বোচ্চ control
│   PDO      │  - hand-tuned performance
└────────────┘  - কোনো magic নেই
```

### তুলনা টেবিল

| বিষয় | ORM | Query Builder | Raw SQL |
|------|-----|---------------|---------|
| উদাহরণ | Eloquent, Doctrine, Prisma | Knex, Laravel QB, Drizzle, jOOQ | PDO, mysql2, pg |
| Object mapping | ✅ আছে | ❌ array/plain object | ❌ array |
| SQL injection safe | ✅ default | ✅ binding | ⚠️ manual |
| Type safety (TS) | ✅ Prisma/Drizzle ভালো | ✅ Drizzle ভালো | ❌ |
| Performance overhead | মাঝারি (10–30%) | ছোট (5–10%) | শূন্য |
| Complex CTE/Window | কঠিন | সহজ | সবচেয়ে সহজ |
| Bulk insert (1M row) | ❌ ধীর | ⚠️ মাঝারি | ✅ COPY/LOAD সবচেয়ে ভাল |
| Reporting/OLAP | ❌ | ⚠️ | ✅ |
| Schema migration | ✅ built-in | ✅ Knex-migrate | ❌ manual |
| Learning curve | মাঝারি/উচ্চ | কম | কম (SQL জানলে) |
| Vendor lock-in | উচ্চ | মাঝারি | নিম্ন |

### সঠিক স্তর বেছে নিন

```
Domain CRUD              → ORM
Reporting / Analytics    → Raw SQL
Bulk import (10M row)    → Raw SQL (LOAD DATA / COPY)
Search filter UI         → Query Builder (dynamic where)
Audit/admin tooling      → Mixed (ORM + raw)
Migrations               → ORM tooling
```

---

## 🧩 মূল ম্যাপিং কনসেপ্ট

### Entity

DB row-এর application-level প্রতিরূপ। PK identity বহন করে।

```php
class Product {
    public int $id;
    public string $title;
    public int $price;
    public ?Seller $seller;       // FK → Seller entity
}
```

### Repository

নির্দিষ্ট entity-এর জন্য collection-like abstraction (`findById`, `findActive`, `save`)। Data Mapper ORM-এ আলাদা class; AR ORM-এ static method-এ থাকে।

### Identity Map

একই session/request-এ একই PK-র object **একবারই** memory-তে থাকে।

```
$user1 = $em->find(User::class, 5);
$user2 = $em->find(User::class, 5);
$user1 === $user2;   // true — দুইবার DB hit হয়নি
```

**লাভ:** Stale reference, double-update বাগ, অতিরিক্ত DB call থেকে রক্ষা।

### Unit of Work

Session-এর মধ্যে যা যা পরিবর্তন হলো সব track করে এবং `flush()` কলে এক সাথে DB-তে পাঠায়। ফলে ১০ entity পরিবর্তন → ১ transaction → কম round-trip।

```
┌──────────────────────────────────────────┐
│            Unit of Work                   │
├──────────────────────────────────────────┤
│ NEW       : [Product#new1, Product#new2] │
│ DIRTY     : [User#5 (name changed)]      │
│ REMOVED   : [Order#42]                   │
│ CLEAN     : [Category#1, Category#3]     │
└──────────────────────────────────────────┘
       flush() → BEGIN; INSERT; UPDATE; DELETE; COMMIT;
```

### Hydration

DB থেকে আসা raw row → entity object তৈরি করার process। Doctrine-এ:
- `HYDRATE_OBJECT` (পূর্ণ entity)
- `HYDRATE_ARRAY` (associative array — দ্রুত)
- `HYDRATE_SCALAR` (flat — সবচেয়ে দ্রুত)

### Proxy

Lazy relation-এর placeholder। Real query তখনই হয় যখন property access করা হয়। Doctrine-এর proxies `var/cache/proxies/__CG__User.php` জাতীয় auto-generated class।

---

## 🔁 Identity Map ও Unit of Work

### Identity Map ছাড়া কী হয়?

```php
// Identity Map নাই (Eloquent-এ default নেই!)
$u1 = User::find(5);   // SELECT ... WHERE id=5
$u1->name = 'Rakib';

$u2 = User::find(5);   // আবার SELECT — পুরনো name নিয়ে আসে!
$u2->save();           // আপনার পরিবর্তন overwrite হয়ে যাবে!
```

Doctrine/Hibernate-এ এটি ঘটে না কারণ একই EntityManager-এ একই PK-র জন্য একই object ফিরবে।

> ⚠️ **মনে রাখবেন:** Eloquent-এ Identity Map নেই (laravel/eloquent-cache প্যাকেজ আছে third-party)। এটি বড় request-এ একই data বারবার fetch করে। সচেতন থাকুন।

---

## 🐌 Lazy vs Eager Loading ও N+1 সমস্যা

### N+1 Problem — Daraz catalog real example

ধরুন Daraz home page-এ 50টি product দেখাচ্ছেন; প্রতিটির seller এবং category দরকার।

```php
// 🚫 N+1 trap
$products = Product::all();             // 1 query → SELECT * FROM products LIMIT 50

foreach ($products as $p) {
    echo $p->seller->name;              // 50 queries → SELECT * FROM sellers WHERE id=?
    echo $p->category->name;            // 50 queries → SELECT * FROM categories WHERE id=?
}
// মোট: 1 + 50 + 50 = 101 queries 😱
```

API latency: প্রতিটি query 2ms হলে ২০২ms শুধু DB-তে। 1000 RPS-এ DB হাঁপিয়ে যাবে।

### সমাধান — Eager Loading

```php
// ✅ Eloquent: with()
$products = Product::with(['seller', 'category'])->get();
// 3 queries: products, sellers IN (...), categories IN (...)

foreach ($products as $p) {
    echo $p->seller->name;             // already loaded — কোনো query নাই
    echo $p->category->name;
}
// মোট: ৩টি query, ১০০x দ্রুত
```

```sql
-- Generated SQL (3টি)
SELECT * FROM products LIMIT 50;
SELECT * FROM sellers   WHERE id IN (12, 47, 88, 91, ...);
SELECT * FROM categories WHERE id IN (3, 5, 8, ...);
```

### Detect N+1

| Tool | কোথায় |
|------|--------|
| Laravel Telescope | dev environment, query log |
| Laravel Debugbar | request-এ query count |
| `barryvdh/laravel-debugbar` | UI overlay |
| `beyondcode/laravel-query-detector` | N+1 হলে exception |
| Doctrine | `EntityManager` query logger |
| Prisma | `log: ['query']` |
| Datadog APM | production monitoring |
| pg_stat_statements | DB level top queries |

### Eager loading-এর variants

| Strategy | কী করে | কখন |
|----------|--------|------|
| `with('seller')` | দ্বিতীয় query IN clause | বেশিরভাগ ক্ষেত্রে best |
| `withCount('reviews')` | subquery যোগ | শুধু count দরকার |
| Joined fetch (Doctrine) | একটি JOIN query | ছোট one-to-one |
| Lazy eager (`load()`) | পরে demand-এ load | শর্তসাপেক্ষ |
| `loadMissing()` | যেগুলো নেই শুধু সেগুলো | mixed cache scenario |

### Eager-এর pitfall — Cartesian explosion

```sql
-- ভুল: JOIN দিয়ে multiple has-many eager করলে
SELECT * FROM products p
JOIN reviews  r ON r.product_id = p.id    -- 1000 review
JOIN images   i ON i.product_id = p.id;   -- 100 image
-- result: 1000 * 100 = 100,000 row! মেমরি explode।
```

ভাল ORM (Eloquent, Doctrine) এই scenario-তে আলাদা query করে IN clause-এ; raw JOIN লিখলে বিপদ।

---

## 🔗 Relationships

```
One-to-One       :  user ↔ profile
One-to-Many      :  seller → products
Many-to-Many     :  product ↔ tag (pivot table)
Polymorphic      :  comment → (post | video | product)
Self-referential :  category → parent_category
```

### Many-to-Many — Daraz product ↔ tag

```sql
products(id, title)
tags(id, name)
product_tag(product_id, tag_id)   -- pivot
```

```php
// Eloquent
class Product extends Model {
    public function tags() {
        return $this->belongsToMany(Tag::class)
                    ->withTimestamps()
                    ->withPivot('priority');
    }
}

// attach / detach / sync
$product->tags()->attach($tagId, ['priority' => 1]);
$product->tags()->sync([1, 2, 3]);   // exact set
```

### Polymorphic — Pathao support ticket comments

```sql
comments(
  id, body,
  commentable_type,    -- 'App\Order' / 'App\Ride' / 'App\Parcel'
  commentable_id
)
```

```php
class Comment extends Model {
    public function commentable() {
        return $this->morphTo();
    }
}
class Order extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

> ⚠️ Polymorphic relation-এ FK constraint দিতে পারবেন না — referential integrity নিজেকে সামলাতে হবে।

---

## 📜 Migrations ও Schema Management

### Eloquent Migration

```php
// database/migrations/2024_07_01_000000_create_orders_table.php
public function up(): void {
    Schema::create('orders', function (Blueprint $t) {
        $t->bigIncrements('id');
        $t->string('order_no', 32)->unique();
        $t->foreignId('user_id')->constrained()->cascadeOnDelete();
        $t->unsignedBigInteger('amount');
        $t->enum('status', ['pending','paid','shipped','delivered'])
          ->default('pending');
        $t->timestamps();
        $t->index(['user_id', 'status']);
    });
}
public function down(): void {
    Schema::dropIfExists('orders');
}
```

```bash
php artisan migrate
php artisan migrate:rollback
php artisan migrate:status
```

### Doctrine Migration

```php
// migrations/Version20240701123000.php
public function up(Schema $schema): void {
    $this->addSql("CREATE TABLE orders (
        id BIGSERIAL PRIMARY KEY,
        order_no VARCHAR(32) UNIQUE NOT NULL,
        user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
        amount BIGINT NOT NULL,
        status VARCHAR(20) DEFAULT 'pending',
        created_at TIMESTAMPTZ DEFAULT NOW(),
        updated_at TIMESTAMPTZ DEFAULT NOW()
    )");
    $this->addSql("CREATE INDEX idx_orders_user_status ON orders(user_id, status)");
}
public function down(Schema $schema): void {
    $this->addSql("DROP TABLE orders");
}
```

```bash
bin/console doctrine:migrations:diff      # entity ↔ DB diff থেকে auto generate
bin/console doctrine:migrations:migrate
```

### Prisma Migrate

```prisma
model Order {
  id        BigInt   @id @default(autoincrement())
  orderNo   String   @unique @map("order_no") @db.VarChar(32)
  userId    BigInt   @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  amount    BigInt
  status    OrderStatus @default(pending)
  createdAt DateTime @default(now()) @map("created_at")

  @@index([userId, status])
  @@map("orders")
}
```

```bash
npx prisma migrate dev --name create_orders     # development
npx prisma migrate deploy                        # production
```

### TypeORM Migration

```ts
export class CreateOrders1719810000000 implements MigrationInterface {
  public async up(q: QueryRunner): Promise<void> {
    await q.query(`CREATE TABLE orders (...)`);
  }
  public async down(q: QueryRunner): Promise<void> {
    await q.query(`DROP TABLE orders`);
  }
}
```

> বিস্তারিত migration strategy (zero-downtime, expand-contract, blue-green) এর জন্য `migration.md` দেখুন।

---

## 🗑️ Soft Delete, Timestamps, Observers

### Soft Delete

Row physically delete না করে `deleted_at` set করা।

```php
class Order extends Model {
    use SoftDeletes;
}

$order->delete();              // UPDATE orders SET deleted_at = NOW()
Order::withTrashed()->get();   // সব সহ
Order::onlyTrashed()->get();   // শুধু deleted
$order->restore();             // deleted_at = NULL
```

**সতর্কতা:**
- Unique constraint সমস্যা: একই email-এ user delete করার পর re-register করতে পারবেন না (deleted row-এও email থাকে)। সমাধান: partial unique index (`UNIQUE WHERE deleted_at IS NULL`) PostgreSQL-এ।
- Storage বাড়ে; archival job দরকার।

### Timestamps

`created_at`, `updated_at` automatic; বেশিরভাগ ORM-এ ON/OFF করা যায়।

### Observers / Lifecycle Hooks

```php
class OrderObserver {
    public function creating(Order $o): void {
        $o->order_no = 'BD-'.strtoupper(Str::random(10));
    }
    public function created(Order $o): void {
        OrderPlaced::dispatch($o);
    }
    public function updated(Order $o): void {
        if ($o->isDirty('status') && $o->status === 'paid') {
            SendReceiptSms::dispatch($o);
        }
    }
}

Order::observe(OrderObserver::class);
```

Doctrine-এ এই কাজটি `#[ORM\HasLifecycleCallbacks]` অথবা EventSubscriber দিয়ে।

> ⚠️ Observers থেকে ভারী/external call করবেন না — queue-এ পাঠান। নাহলে save() slow হবে এবং transaction বড় হবে।

---

## 🔒 Transactions ও Locking

### ORM-এ Transaction

```php
// Eloquent
DB::transaction(function () use ($from, $to, $amount) {
    $from->decrement('balance', $amount);
    $to->increment('balance', $amount);
    Transfer::create([...]);
}, 3);   // deadlock হলে ৩ বার retry
```

```php
// Doctrine
$em->wrapInTransaction(function ($em) use ($order) {
    $em->persist($order);
    foreach ($order->getItems() as $item) $em->persist($item);
});
```

```ts
// Prisma
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where:{id:from}, data:{ balance:{ decrement:amount }}});
  await tx.account.update({ where:{id:to},   data:{ balance:{ increment:amount }}});
  await tx.transfer.create({ data:{...} });
}, { isolationLevel: 'Serializable' });
```

### Isolation Level সেট করা

| ORM | API |
|-----|-----|
| Eloquent | `DB::statement('SET TRANSACTION ISOLATION LEVEL REPEATABLE READ')` then `transaction()` |
| Doctrine | `$em->getConnection()->setTransactionIsolation(TransactionIsolationLevel::SERIALIZABLE)` |
| Prisma | `$transaction(fn, { isolationLevel: 'Serializable' })` |
| TypeORM | `dataSource.transaction('SERIALIZABLE', cb)` |

### Optimistic Locking (version column)

```php
// Eloquent — manual
$post = Post::find(1);   // version = 5
$post->title = 'New';
$updated = Post::where('id', 1)
               ->where('version', $post->version)
               ->update(['title' => 'New', 'version' => $post->version + 1]);
if (! $updated) throw new ConflictException();
```

```php
// Doctrine — built-in
#[ORM\Version]
#[ORM\Column(type:'integer')]
private int $version;
// Doctrine নিজেই version check করে, mismatch হলে OptimisticLockException
```

### Pessimistic Locking

```php
$account = Account::where('id', $id)->lockForUpdate()->first();   // SELECT ... FOR UPDATE
```

```php
// Doctrine
$em->find(Account::class, $id, LockMode::PESSIMISTIC_WRITE);
```

> বিস্তারিত isolation, deadlock handling, MVCC-এর জন্য `transactions.md` পড়ুন।

---

## ⚡ Performance Tuning

### ১. শুধু দরকারি column SELECT

```php
Product::select(['id', 'title', 'price'])->get();   // SELECT id,title,price
```

বড় table-এ `image_blob`, `description_html` বাদ দিলে ১০x দ্রুত হতে পারে।

### ২. Chunking

```php
// 5M order export
Order::where('status','delivered')
     ->chunkById(1000, function ($chunk) {
         foreach ($chunk as $o) writeToCsv($o);
     });
// memory নিয়ন্ত্রণে থাকে; cursor-based pagination
```

### ৩. Cursor / Lazy collection

```php
foreach (Order::cursor() as $o) { ... }   // PDO unbuffered query
```

### ৪. Bulk insert

```php
// 🚫 ধীর — N round trip
foreach ($rows as $r) Product::create($r);

// ✅ দ্রুত
Product::insert($rows);   // 1 SQL
DB::table('products')->upsert($rows, ['sku'], ['price','stock']);
```

৫ লাখ row Eloquent `create()` লুপে ~৪০ মিনিট, `insert()` chunked-এ ~৩০ সেকেন্ড।

### ৫. Raw SQL fallback

```php
$revenue = DB::select("
    SELECT seller_id, SUM(amount) total
    FROM orders
    WHERE created_at >= ? AND status='delivered'
    GROUP BY seller_id
    ORDER BY total DESC LIMIT 100
", [now()->subDays(30)]);
```

### ৬. Query log + EXPLAIN

```php
DB::enableQueryLog();
// ... code ...
dd(DB::getQueryLog());
```

```sql
EXPLAIN ANALYZE SELECT ...   -- production-এ আগেই check
```

> বিস্তারিত execution plan reading এর জন্য `query-execution-plan.md`, indexing-এর জন্য `indexing.md` দেখুন।

### ৭. Common ORM perf pitfalls

```
❌ accessor-এ DB call          → প্রতিটি serialization N query
❌ collection-এ count()         → পুরো hydrate; বদলে query()->count()
❌ ->get()->count()             → একই কারণ
❌ where('a', $b)->orWhere(...) chain — parenthesization ভুলে wrong result
❌ pluck after get              → আগেই ->pluck() করুন (memory ও CPU)
❌ relationship in loop without with()  → N+1
❌ eager তে JOIN চাপিয়ে দেওয়া   → cartesian explosion
❌ ->all()->paginate()           → first all() pulls everything; ->paginate() directly
```

---

## 🗃️ Caching Layers

```
┌────────────────────────────────────────────────┐
│ L1 : Identity Map (per-request, in-memory)     │  Doctrine, Hibernate
├────────────────────────────────────────────────┤
│ L2 : Cross-request entity cache                │  Doctrine 2nd-level cache,
│      (Redis/Memcached)                         │  Hibernate L2
├────────────────────────────────────────────────┤
│ L3 : Query result cache                        │  Eloquent + spatie/laravel-responsecache,
│      (Redis JSON blob)                         │  rememberable trait
├────────────────────────────────────────────────┤
│ L4 : HTTP/CDN cache                            │  Nginx fastcgi_cache, Cloudflare
└────────────────────────────────────────────────┘
```

### Eloquent + Cache

```php
$products = Cache::remember("products:home", 300, function () {
    return Product::with('seller','category')
                  ->where('featured', true)
                  ->take(50)->get();
});
```

### Doctrine 2nd-level cache

```yaml
doctrine:
  orm:
    second_level_cache:
      enabled: true
      regions:
        product_region:
          cache_driver: { type: pool, pool: cache.app }
          lifetime: 300
```

```php
#[ORM\Cache(usage: 'READ_ONLY', region: 'product_region')]
class Product { ... }
```

### Prisma Accelerate

Edge-distributed query cache + connection pooling SaaS:

```ts
const products = await prisma.product.findMany({
  where: { featured: true },
  cacheStrategy: { ttl: 60, swr: 600 }
});
```

> Cache invalidation hard problem — model save event-এ relevant key clear করুন; বিস্তারিত `06-caching/` দেখুন।

---

## 🌐 Multi-DB / Read-Replica / Multi-Tenancy

### Read replica (Eloquent)

```php
// config/database.php
'mysql' => [
    'read' => [
        'host' => ['10.0.0.21','10.0.0.22','10.0.0.23'],
    ],
    'write' => [ 'host' => '10.0.0.10' ],
    'sticky' => true,    // request-এ write হলে read-ও master থেকে
    ...
],
```

`SELECT` queries replica-তে যাবে; `INSERT/UPDATE/DELETE` master-এ।

### Multi-tenancy strategies

```
┌─────────────────────┬─────────────────────────────────────────────┐
│ Strategy            │ ORM-এ কী করবেন                              │
├─────────────────────┼─────────────────────────────────────────────┤
│ Shared DB, tenant_id│ Global scope: ->where('tenant_id', $tid)    │
│ DB-per-tenant       │ Connection switch per request               │
│ Schema-per-tenant   │ PG schema search_path; Doctrine multi-EM    │
└─────────────────────┴─────────────────────────────────────────────┘
```

### Sharding-friendly ORM patterns

- Surrogate PK হিসেবে UUID/ULID/snowflake ব্যবহার (auto-increment cross-shard collision হয়)।
- কোনো cross-shard JOIN নয়; aggregator সার্ভিসে roll up।
- ORM relation-এ FK constraint বাদ দিতে হতে পারে যদি FK অন্য shard-এ থাকে।

---

## 📊 ORM Comparison Matrix

| Feature | Eloquent (Laravel) | Doctrine (Symfony) | Prisma (Node) | TypeORM | Sequelize | MikroORM | Drizzle | Objection.js |
|---------|--------------------|--------------------|---------------|---------|-----------|----------|---------|--------------|
| Pattern | Active Record | Data Mapper | Hybrid (DM-ish) | AR + DM | AR | DM | Query Builder + Schema | AR over Knex |
| Language | PHP | PHP | TS/JS | TS/JS | JS/TS | TS | TS | TS/JS |
| Type safety | ❌ (PHPStan সহায়ক) | ⚠️ via attributes | ✅ best-in-class | ✅ভাল | ⚠️ | ✅ভাল | ✅ভাল | ⚠️ |
| Identity Map | ❌ | ✅ | ❌ | ❌ (DM mode-এ ✅) | ❌ | ✅ | ❌ | ❌ |
| Unit of Work | ❌ | ✅ | ❌ | ⚠️ | ❌ | ✅ | ❌ | ❌ |
| Migration | ✅ artisan | ✅ doctrine-migrations | ✅ prisma migrate | ✅ | ✅ umzug | ✅ | ✅ drizzle-kit | ✅ knex |
| Raw SQL escape hatch | ✅ DB::raw | ✅ DBAL/native | ✅ $queryRaw | ✅ | ✅ | ✅ | ✅ | ✅ |
| Eager loading | with() | fetch=EAGER / DQL | include | relations: | include | populate | with | withGraphFetched |
| Soft delete | ✅ trait | ✅ filter | ⚠️ manual middleware | ✅ | ✅ paranoid | ✅ | ❌ manual | ❌ manual |
| Bulk insert speed | মাঝারি | ভাল (DBAL) | ভাল | মাঝারি | মাঝারি | ভাল | চমৎকার | ভাল |
| Maturity | উচ্চ (২০১১+) | উচ্চ (২০০৬+) | উচ্চ (২০২১) | উচ্চ | উচ্চ | মাঝারি | নতুন (২০২২) | মাঝারি |
| Community | বিশাল (Laravel) | বড় | বড় ও fast-growing | বড় | বড় | ছোট কিন্তু সক্রিয় | দ্রুত বাড়ছে | ছোট |
| Learning curve | সহজ | steep | মাঝারি | মাঝারি | সহজ | মাঝারি | সহজ (SQL জানলে) | সহজ |
| Schema source | DB → migration | Entity attrs / XML | `schema.prisma` | Entity decorators | model() call | Entity decorators | TS schema | Model class |
| Best for | Laravel monolith, MVC | Symfony, DDD heavy | TS-first apps, modern | Mixed AR/DM TS apps | লেগেসি Node | DDD TS app | edge / serverless TS | Knex love-করা team |

---

## 🚫 কখন ORM ব্যবহার করবেন না

```
❌ OLAP / reporting queries (window, CTE, recursive, pivot)
❌ Bulk import — 10M row LOAD DATA / COPY-এই ভাল
❌ ETL pipeline — Spark / Airflow / dbt
❌ অতি hot path (HFT, ad bidding, payment authorize) — raw prepared statement
❌ Stored procedure-heavy enterprise system
❌ Complex full-text search → Elasticsearch / Solr
❌ Graph traversal → Neo4j Cypher
```

ORM থেকে raw-এ পরিবর্তন এক ক্লাসে করতে পারলে সবচেয়ে ভাল — repository pattern সাহায্য করে। `repository-pattern.md` দেখুন।

---

## 🚨 Anti-patterns

### ১. "Fat model, dumb everywhere else"

10,000-লাইনের `User` class — auth, billing, notification — সব এখানে। Solution: Service / Action class-এ logic; model শুধু persistence।

### ২. Loop-এ save()

```php
foreach ($items as $i) $i->save();   // N transaction
// ✅ DB::transaction(fn() => ...) অথবা bulk update
```

### ৩. Magic accessor-এ external API

```php
public function getProfileImageAttribute() {
    return Storage::disk('s3')->url($this->image_path);   // S3 sign প্রতিবার!
}
```

### ৪. Auto-load everything

```php
class User {
    protected $with = ['orders','addresses','payments','reviews']; // সর্বদা load!
}
// User::find(1) → ৫টি query সবসময়; দরকার না হলেও
```

### ৫. ORM দিয়ে aggregation

```php
$total = Order::all()->sum('amount');   // সব row PHP-তে এনে যোগ!
// ✅ Order::sum('amount')   — DB-তে SUM()
```

### ৬. `findOrFail` দিয়ে exception-driven flow

প্রতিবার 404 ছোড়া + stack trace = expensive। যেখানে miss স্বাভাবিক, সেখানে `find()` দিয়ে null check।

### ৭. Schema "magic" auto-sync production-এ

`synchronize: true` (TypeORM) production-এ data হারাতে পারে। সবসময় migration ব্যবহার।

---

## 🇧🇩 BD Real-world কেস স্টাডি

### bKash — Raw PDO থেকে Doctrine-এ মাইগ্রেশন

পুরনো settlement service raw PDO + manual SQL ছিল। সমস্যা:

- ১২০+ SQL file scattered, কোথায় কোন query চলছে অজানা
- New developer onboard করতে ৩ সপ্তাহ
- Test coverage ১২%, deadlock অস্পষ্ট

**Action:** Doctrine adopt; Settlement, Wallet, Transaction-কে aggregate root হিসেবে modeled; `#[ORM\Version]` দিয়ে optimistic locking; second-level cache 5s TTL।

**Result:** Code 35% কম, dev onboarding 5 দিন, test coverage 78%; latency 8% বাড়ল কিন্তু p99 stable। Hot reconciliation report এখনো raw SQL।

> Lesson: Critical money path = raw SQL + audit; everything else = Doctrine।

### Pathao — TypeScript microservice-এ Prisma

Driver matching service-এ Prisma adopted। `prisma.driver.findMany({ where: { active: true, location: { near: ... } } })` clean। Transaction guarantee দিয়ে ride-create + payment-authorize একসাথে। Prisma Accelerate আনার পর Singapore region থেকে latency 110ms → 22ms।

**Pitfall শিখলো:** Prisma `findMany` ১০০K+ row-এ memory blow up — pagination + `take/skip` cursor mandatory।

### Daraz — Eloquent N+1 audit গল্প

Black Friday-র আগে home page p95 ৩.২ সেকেন্ড। Telescope log:

```
GET /api/home → 412 queries
```

কারণ: প্রতিটি product card-এ accessor `formatted_seller_rating` যা `$this->seller->reviews->avg('star')` করছিল — পুরো review collection load। Fix:

1. `Product::with(['seller:id,name,rating_cached'])` — `rating_cached` denormalized column
2. Query Detector CI-তে চালু — N+1 হলে build fail
3. Hot product list → Redis cache 30s

p95: 3200ms → 180ms।

---

## 💻 Full Code: PHP (Eloquent + Doctrine)

Domain: Daraz simplified — `Seller`, `Product`, `Category`, `Tag`, `Order`, `OrderItem`।

### A. Eloquent (Laravel)

```php
// database/migrations/...
Schema::create('sellers', function (Blueprint $t) {
    $t->id();
    $t->string('name');
    $t->string('phone', 20)->unique();
    $t->decimal('rating_cached', 3, 2)->default(0);
    $t->timestamps();
});

Schema::create('categories', function (Blueprint $t) {
    $t->id();
    $t->string('name');
    $t->foreignId('parent_id')->nullable()->constrained('categories');
    $t->timestamps();
});

Schema::create('products', function (Blueprint $t) {
    $t->id();
    $t->string('sku', 32)->unique();
    $t->string('title');
    $t->text('description')->nullable();
    $t->unsignedBigInteger('price_paisa');
    $t->unsignedInteger('stock')->default(0);
    $t->foreignId('seller_id')->constrained();
    $t->foreignId('category_id')->constrained();
    $t->boolean('featured')->default(false);
    $t->softDeletes();
    $t->timestamps();
    $t->index(['seller_id', 'featured']);
    $t->index(['category_id', 'price_paisa']);
});

Schema::create('tags', function (Blueprint $t) {
    $t->id(); $t->string('name')->unique(); $t->timestamps();
});
Schema::create('product_tag', function (Blueprint $t) {
    $t->foreignId('product_id')->constrained()->cascadeOnDelete();
    $t->foreignId('tag_id')->constrained()->cascadeOnDelete();
    $t->primary(['product_id','tag_id']);
});

Schema::create('orders', function (Blueprint $t) {
    $t->id();
    $t->string('order_no', 40)->unique();
    $t->foreignId('user_id')->constrained();
    $t->unsignedBigInteger('total_paisa');
    $t->enum('status',['pending','paid','shipped','delivered','cancelled'])
      ->default('pending');
    $t->unsignedInteger('version')->default(0);
    $t->timestamps();
    $t->index(['user_id','status']);
});

Schema::create('order_items', function (Blueprint $t) {
    $t->id();
    $t->foreignId('order_id')->constrained()->cascadeOnDelete();
    $t->foreignId('product_id')->constrained();
    $t->unsignedInteger('qty');
    $t->unsignedBigInteger('price_paisa');
});
```

```php
// app/Models/Seller.php
class Seller extends Model {
    protected $fillable = ['name','phone'];
    public function products() { return $this->hasMany(Product::class); }
}

// app/Models/Category.php
class Category extends Model {
    protected $fillable = ['name','parent_id'];
    public function parent()   { return $this->belongsTo(self::class,'parent_id'); }
    public function children() { return $this->hasMany(self::class,'parent_id'); }
    public function products() { return $this->hasMany(Product::class); }
}

// app/Models/Product.php
class Product extends Model {
    use SoftDeletes;
    protected $fillable = ['sku','title','description','price_paisa','stock',
                           'seller_id','category_id','featured'];
    protected $casts = ['featured'=>'boolean','price_paisa'=>'integer'];

    public function seller()   { return $this->belongsTo(Seller::class); }
    public function category() { return $this->belongsTo(Category::class); }
    public function tags()     { return $this->belongsToMany(Tag::class); }

    public function scopeAvailable($q) {
        return $q->where('stock','>',0)->whereNull('deleted_at');
    }
    public function scopeForCategory($q, int $catId) {
        return $q->where('category_id', $catId);
    }
    public function getPriceTakaAttribute(): string {
        return '৳ '.number_format($this->price_paisa / 100, 2);
    }
}

// app/Models/Order.php
class Order extends Model {
    protected $fillable = ['order_no','user_id','total_paisa','status'];
    public function items() { return $this->hasMany(OrderItem::class); }
    public function user()  { return $this->belongsTo(User::class); }
}

class OrderItem extends Model {
    public $timestamps = false;
    protected $fillable = ['order_id','product_id','qty','price_paisa'];
    public function product() { return $this->belongsTo(Product::class); }
}
```

```php
// app/Services/CheckoutService.php
class CheckoutService {
    public function place(int $userId, array $cart): Order {
        return DB::transaction(function () use ($userId, $cart) {
            $productIds = array_column($cart, 'product_id');

            // pessimistic lock — race condition prevent
            $products = Product::whereIn('id', $productIds)
                ->lockForUpdate()->get()->keyBy('id');

            $order = Order::create([
                'order_no'    => 'BD'.now()->format('YmdHis').Str::random(6),
                'user_id'     => $userId,
                'total_paisa' => 0,
                'status'      => 'pending',
            ]);

            $total = 0;
            foreach ($cart as $line) {
                $p = $products[$line['product_id']];
                if ($p->stock < $line['qty']) {
                    throw new InsufficientStockException($p->id);
                }
                $p->decrement('stock', $line['qty']);
                OrderItem::create([
                    'order_id'    => $order->id,
                    'product_id'  => $p->id,
                    'qty'         => $line['qty'],
                    'price_paisa' => $p->price_paisa,
                ]);
                $total += $p->price_paisa * $line['qty'];
            }

            $order->update(['total_paisa' => $total]);
            return $order->fresh('items.product');
        }, attempts: 3);
    }
}

// Home page — N+1 free
class HomeController {
    public function index() {
        $featured = Cache::remember('home:featured', 60, function () {
            return Product::query()
                ->available()
                ->where('featured', true)
                ->with([
                    'seller:id,name,rating_cached',
                    'category:id,name',
                    'tags:id,name',
                ])
                ->select(['id','sku','title','price_paisa','seller_id','category_id'])
                ->limit(50)
                ->get();
        });
        return view('home', compact('featured'));
    }
}
```

### B. Doctrine (Symfony)

```php
// src/Entity/Seller.php
#[ORM\Entity(repositoryClass: SellerRepository::class)]
#[ORM\Table(name: 'sellers')]
class Seller {
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type:'bigint')]
    private ?int $id = null;
    #[ORM\Column(length: 120)]
    private string $name;
    #[ORM\Column(length: 20, unique: true)]
    private string $phone;
    #[ORM\Column(type:'decimal', precision:3, scale:2)]
    private string $ratingCached = '0.00';

    #[ORM\OneToMany(mappedBy:'seller', targetEntity: Product::class)]
    private Collection $products;

    public function __construct() { $this->products = new ArrayCollection(); }
    // getters/setters omitted for brevity
}

#[ORM\Entity]
#[ORM\Table(name:'products')]
#[ORM\Index(columns:['seller_id','featured'])]
#[ORM\Index(columns:['category_id','price_paisa'])]
#[Gedmo\SoftDeleteable(fieldName: 'deletedAt')]
class Product {
    #[ORM\Id, ORM\GeneratedValue, ORM\Column(type:'bigint')]
    private ?int $id = null;
    #[ORM\Column(length:32, unique:true)]
    private string $sku;
    #[ORM\Column(length:255)]
    private string $title;
    #[ORM\Column(type:'text', nullable:true)]
    private ?string $description = null;
    #[ORM\Column(type:'bigint')]
    private int $pricePaisa;
    #[ORM\Column(type:'integer')]
    private int $stock = 0;

    #[ORM\ManyToOne(targetEntity: Seller::class, inversedBy:'products')]
    #[ORM\JoinColumn(nullable:false)]
    private Seller $seller;

    #[ORM\ManyToOne(targetEntity: Category::class)]
    #[ORM\JoinColumn(nullable:false)]
    private Category $category;

    #[ORM\ManyToMany(targetEntity: Tag::class)]
    #[ORM\JoinTable(name:'product_tag')]
    private Collection $tags;

    #[ORM\Column(type:'boolean')]
    private bool $featured = false;

    #[ORM\Column(type:'datetime', nullable:true)]
    private ?\DateTimeInterface $deletedAt = null;

    #[ORM\Version, ORM\Column(type:'integer')]
    private int $version = 1;
    // ...
}
```

```php
// src/Repository/ProductRepository.php
class ProductRepository extends ServiceEntityRepository {
    public function findFeaturedHome(): array {
        return $this->createQueryBuilder('p')
            ->select('p','s','c','t')
            ->leftJoin('p.seller','s')
            ->leftJoin('p.category','c')
            ->leftJoin('p.tags','t')
            ->where('p.featured = :f')->setParameter('f', true)
            ->andWhere('p.stock > 0')
            ->andWhere('p.deletedAt IS NULL')
            ->setMaxResults(50)
            ->getQuery()
            ->enableResultCache(60, 'home_featured')
            ->getResult();
    }
}

// src/Service/CheckoutService.php
class CheckoutService {
    public function __construct(
        private EntityManagerInterface $em,
        private ProductRepository $products,
    ) {}

    public function place(User $user, array $cart): Order {
        return $this->em->wrapInTransaction(function () use ($user,$cart) {
            $order = (new Order())->setUser($user)
                ->setOrderNo('BD'.bin2hex(random_bytes(8)));

            foreach ($cart as $line) {
                $p = $this->em->find(
                    Product::class, $line['product_id'],
                    LockMode::PESSIMISTIC_WRITE
                );
                if ($p->getStock() < $line['qty']) {
                    throw new InsufficientStockException($p->getId());
                }
                $p->decrementStock($line['qty']);

                $item = (new OrderItem())
                    ->setProduct($p)->setQty($line['qty'])
                    ->setPricePaisa($p->getPricePaisa());
                $order->addItem($item);
            }
            $this->em->persist($order);
            return $order;
        });
    }
}
```

---

## 💻 Full Code: Node.js (Prisma + TypeORM)

### A. Prisma

```prisma
// prisma/schema.prisma
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }
generator client { provider = "prisma-client-js" }

model Seller {
  id            BigInt    @id @default(autoincrement())
  name          String
  phone         String    @unique @db.VarChar(20)
  ratingCached  Decimal   @default(0) @map("rating_cached") @db.Decimal(3,2)
  products      Product[]
  createdAt     DateTime  @default(now()) @map("created_at")
  @@map("sellers")
}

model Category {
  id        BigInt     @id @default(autoincrement())
  name      String
  parentId  BigInt?    @map("parent_id")
  parent    Category?  @relation("Children", fields: [parentId], references: [id])
  children  Category[] @relation("Children")
  products  Product[]
  @@map("categories")
}

model Product {
  id          BigInt   @id @default(autoincrement())
  sku         String   @unique @db.VarChar(32)
  title       String
  description String?
  pricePaisa  BigInt   @map("price_paisa")
  stock       Int      @default(0)
  sellerId    BigInt   @map("seller_id")
  seller      Seller   @relation(fields: [sellerId], references: [id])
  categoryId  BigInt   @map("category_id")
  category    Category @relation(fields: [categoryId], references: [id])
  tags        Tag[]    @relation("ProductTags")
  featured    Boolean  @default(false)
  deletedAt   DateTime? @map("deleted_at")
  createdAt   DateTime @default(now()) @map("created_at")
  orderItems  OrderItem[]

  @@index([sellerId, featured])
  @@index([categoryId, pricePaisa])
  @@map("products")
}

model Tag {
  id       BigInt    @id @default(autoincrement())
  name     String    @unique
  products Product[] @relation("ProductTags")
  @@map("tags")
}

enum OrderStatus { pending paid shipped delivered cancelled }

model Order {
  id          BigInt      @id @default(autoincrement())
  orderNo     String      @unique @map("order_no") @db.VarChar(40)
  userId      BigInt      @map("user_id")
  totalPaisa  BigInt      @map("total_paisa")
  status      OrderStatus @default(pending)
  version     Int         @default(0)
  items       OrderItem[]
  createdAt   DateTime    @default(now()) @map("created_at")
  @@index([userId, status])
  @@map("orders")
}

model OrderItem {
  id          BigInt  @id @default(autoincrement())
  orderId     BigInt  @map("order_id")
  order       Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId   BigInt  @map("product_id")
  product     Product @relation(fields: [productId], references: [id])
  qty         Int
  pricePaisa  BigInt  @map("price_paisa")
  @@map("order_items")
}
```

```ts
// src/services/checkout.ts
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient({ log: ['warn','error'] });

export async function placeOrder(userId: bigint, cart: {productId: bigint; qty: number}[]) {
  return prisma.$transaction(async (tx) => {
    const order = await tx.order.create({
      data: {
        orderNo: 'BD' + Date.now() + Math.random().toString(36).slice(2,8).toUpperCase(),
        userId, totalPaisa: 0n,
      },
    });

    let total = 0n;
    for (const line of cart) {
      // pessimistic lock — Prisma raw because no built-in FOR UPDATE
      const [p] = await tx.$queryRaw<{id:bigint;stock:number;price_paisa:bigint}[]>`
        SELECT id, stock, price_paisa
        FROM products WHERE id = ${line.productId}
        FOR UPDATE`;
      if (!p || p.stock < line.qty) throw new Error(`out_of_stock:${line.productId}`);

      await tx.product.update({
        where: { id: line.productId },
        data:  { stock: { decrement: line.qty } },
      });
      await tx.orderItem.create({
        data: { orderId: order.id, productId: line.productId, qty: line.qty, pricePaisa: p.price_paisa },
      });
      total += p.price_paisa * BigInt(line.qty);
    }

    return tx.order.update({
      where:{ id: order.id },
      data: { totalPaisa: total },
      include:{ items: { include: { product: true } } },
    });
  }, { isolationLevel: Prisma.TransactionIsolationLevel.ReadCommitted, timeout: 10_000 });
}

// Home — eager
export const featuredHome = () => prisma.product.findMany({
  where: { featured: true, stock: { gt: 0 }, deletedAt: null },
  select: {
    id:true, sku:true, title:true, pricePaisa:true,
    seller:   { select: { id:true, name:true, ratingCached:true } },
    category: { select: { id:true, name:true } },
    tags:     { select: { id:true, name:true } },
  },
  take: 50,
});
```

### B. TypeORM (Data Mapper mode)

```ts
// src/entity/Product.ts
@Entity('products')
@Index(['seller','featured'])
@Index(['category','pricePaisa'])
export class Product {
  @PrimaryGeneratedColumn('increment') id!: string;
  @Column({ length: 32, unique: true }) sku!: string;
  @Column() title!: string;
  @Column({ type:'text', nullable:true }) description?: string;
  @Column({ type:'bigint', name:'price_paisa' }) pricePaisa!: string;
  @Column({ default: 0 }) stock!: number;

  @ManyToOne(() => Seller, s => s.products, { eager: false })
  @JoinColumn({ name: 'seller_id' })
  seller!: Seller;

  @ManyToOne(() => Category)
  @JoinColumn({ name: 'category_id' })
  category!: Category;

  @ManyToMany(() => Tag)
  @JoinTable({ name: 'product_tag' })
  tags!: Tag[];

  @Column({ default: false }) featured!: boolean;
  @DeleteDateColumn({ name: 'deleted_at' }) deletedAt?: Date;
  @VersionColumn() version!: number;
}

// src/services/checkout.ts
export async function placeOrder(ds: DataSource, userId: string, cart: any[]) {
  return ds.transaction('SERIALIZABLE', async (em) => {
    const productRepo = em.getRepository(Product);
    const orderRepo   = em.getRepository(Order);

    const products = await productRepo.find({
      where: { id: In(cart.map(c => c.productId)) },
      lock:  { mode: 'pessimistic_write' },
    });
    const byId = new Map(products.map(p => [p.id, p]));

    const order = orderRepo.create({
      orderNo: 'BD'+Date.now(),
      userId, status: 'pending', totalPaisa: '0',
      items: [],
    });

    let total = 0n;
    for (const line of cart) {
      const p = byId.get(line.productId);
      if (!p || p.stock < line.qty) throw new Error('out_of_stock');
      p.stock -= line.qty;
      const item = em.create(OrderItem, {
        product: p, qty: line.qty, pricePaisa: p.pricePaisa,
      });
      order.items.push(item);
      total += BigInt(p.pricePaisa) * BigInt(line.qty);
      await em.save(p);
    }
    order.totalPaisa = total.toString();
    return em.save(order);
  });
}
```

---

## ✅ Checklist

### Design
- [ ] AR vs DM সচেতনভাবে বেছেছেন (CRUD-heavy → AR; DDD-heavy → DM)
- [ ] Critical money/transactional path raw SQL/repository-এ isolated
- [ ] Migration file PR review পদ্ধতিতে আছে
- [ ] Schema source-of-truth (Prisma schema / Doctrine entity / migration) defined

### Performance
- [ ] N+1 detector dev এবং CI-তে enable
- [ ] All public list endpoint-এ explicit `with()`/`include`
- [ ] Heavy column (blob, html) `select()`-এ exclude
- [ ] Bulk operation-এ chunk + bulk insert
- [ ] Hot read endpoint-এ cache (TTL + invalidation strategy)
- [ ] EXPLAIN-এ unintended seq scan নাই
- [ ] `pg_stat_statements` / slow query log monitor হচ্ছে

### Correctness
- [ ] Money column = integer (paisa/cents), float নয়
- [ ] FK constraints DB-তে present (ORM-only constraint যথেষ্ট নয়)
- [ ] Transaction boundary স্পষ্ট, observers-এ heavy কাজ নেই
- [ ] Optimistic lock (`@Version`) editable resource-এ
- [ ] Soft-delete unique index partial (`WHERE deleted_at IS NULL`)

### Operations
- [ ] Production-এ schema auto-sync **OFF**
- [ ] Read replica routing config + `sticky` after write
- [ ] Migration zero-downtime (expand-contract) plan
- [ ] Connection pool sizing tested under load
- [ ] APM (Telescope, Datadog, New Relic) ORM query trace

### Anti-pattern audit
- [ ] কোনো `Model::all()` লুপে save() নাই
- [ ] Accessor-এ DB/HTTP call নাই
- [ ] Default `$with` overuse নেই
- [ ] Aggregation DB-তে (`sum/count/avg`), application-এ নয়
- [ ] `findOrFail` শুধু genuine 404 path-এ

---

> **চূড়ান্ত কথা:** ORM হলো লিভারেজ — productivity ১০x, ভুল হলে performance ১০x ধীর। জানুন কখন trust করবেন (CRUD), কখন bypass করবেন (analytics, bulk, hot money path)। SQL ভালো করে শিখুন, ORM তখন আপনার সহকর্মী, প্রভু নয়।
