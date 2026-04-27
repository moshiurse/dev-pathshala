# 🔀 CQRS (Command Query Responsibility Segregation)

## 📌 সংজ্ঞা ও মূল ধারণা

**CQRS** হলো একটি architectural pattern যেখানে একটি application-এর **data write (Command)** এবং **data read (Query)** operations সম্পূর্ণ আলাদা model ব্যবহার করে handle করা হয়। এই pattern-টি সর্বপ্রথম **Greg Young** ২০১০ সালে formalize করেন, যদিও এর মূল ভিত্তি **Bertrand Meyer**-এর **CQS (Command Query Separation)** principle থেকে এসেছে।

### CQS vs CQRS — পার্থক্য বোঝা

| দিক | CQS (Command Query Separation) | CQRS |
|------|------|------|
| স্কোপ | Method level — একটি method হয় state change করবে, নয়তো data return করবে | Architecture level — সম্পূর্ণ আলাদা model ও infrastructure |
| প্রবর্তক | Bertrand Meyer (Eiffel language) | Greg Young |
| জটিলতা | সহজ, যেকোনো codebase-এ প্রয়োগ সম্ভব | জটিল, distributed system-এ ব্যবহার |
| Database | একই database, একই model | আলাদা database, আলাদা model হতে পারে |

### কেন Read ও Write আলাদা করবো?

বাংলাদেশের একটি বড় e-commerce platform যেমন **Daraz** বা **Chaldal**-এর কথা ভাবুন। এখানে প্রতি সেকেন্ডে হাজার হাজার user product search করছে (Read), কিন্তু order place হচ্ছে তুলনামূলক কম সংখ্যক (Write)। Read এবং Write-এর requirement সম্পূর্ণ আলাদা:

- **Write (Command):** Data integrity, validation, business rules enforce করা, transaction management
- **Read (Query):** Speed, denormalized data, caching, search optimization

একই model দিয়ে দুটো কাজ করলে কোনোটাই optimize করা যায় না — এটাই CQRS-এর মূল সমস্যা সমাধানের জায়গা।

### Command vs Query — মৌলিক পার্থক্য

```
Command (আদেশ):
  ├── System-এর state পরিবর্তন করে
  ├── কোনো data return করে না (শুধু success/failure)
  ├── Imperative mood: PlaceOrder, CancelOrder, UpdateProfile
  └── Side effects আছে

Query (প্রশ্ন):
  ├── System-এর state পরিবর্তন করে না
  ├── Data return করে
  ├── Interrogative: GetOrderDetails, SearchProducts
  └── Side-effect free, idempotent
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 📚 লাইব্রেরি অ্যানালজি

একটি বিশ্ববিদ্যালয়ের লাইব্রেরি কল্পনা করুন:

**Librarian (Command Side):**
- নতুন বই যোগ করেন (AddBook)
- বই ইস্যু করেন (IssueBook)
- ফেরত নেন (ReturnBook)
- ক্ষতিগ্রস্ত বই সরিয়ে ফেলেন (RemoveBook)
- প্রতিটি কাজের জন্য register-এ entry দেন (Event Log)

**Catalog System (Query Side):**
- শিক্ষার্থীরা computer-এ বই search করে (SearchBooks)
- কোন বই available আছে দেখে (CheckAvailability)
- জনপ্রিয় বইয়ের তালিকা দেখে (GetPopularBooks)
- Catalog-টি Librarian-এর entry থেকে আপডেট হয়, কিন্তু সরাসরি বই manage করে না

এখানে Librarian-এর register হলো **Write Model**, আর Catalog System হলো **Read Model** — দুটো আলাদাভাবে optimized।

### 🏦 বাংলাদেশী ব্যাংকিং উদাহরণ

**bKash/Nagad** এর মতো mobile banking system-এ:
- **Command Side:** SendMoney, CashOut, PayBill — প্রতিটি transaction atomically process হয়, fraud detection চলে, balance update হয়
- **Query Side:** CheckBalance, TransactionHistory, MiniStatement — দ্রুত response দিতে হয়, cached/denormalized data থেকে serve হয়

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### ডায়াগ্রাম ১: Simple CQRS (Single Database)

```
                        ┌─────────────────┐
                        │     Client      │
                        │  (React/Mobile) │
                        └────────┬────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              ┌─────▼─────┐            ┌──────▼─────┐
              │  Command   │            │   Query    │
              │   API      │            │    API     │
              └─────┬──────┘            └──────┬─────┘
                    │                          │
              ┌─────▼──────┐            ┌──────▼─────┐
              │  Command   │            │   Query    │
              │  Handler   │            │  Handler   │
              │ (Write     │            │ (Read      │
              │  Model)    │            │  Model)    │
              └─────┬──────┘            └──────┬─────┘
                    │                          │
                    └──────────┬───────────────┘
                               │
                        ┌──────▼──────┐
                        │  Single DB  │
                        │ (PostgreSQL)│
                        └─────────────┘
```

### ডায়াগ্রাম ২: Full CQRS (Separate Databases)

```
                        ┌─────────────────┐
                        │     Client      │
                        └────────┬────────┘
                    ┌────────────┴────────────┐
                    │                         │
              ┌─────▼──────┐           ┌──────▼──────┐
              │  Command   │           │   Query     │
              │  Service   │           │  Service    │
              └─────┬──────┘           └──────▲──────┘
                    │                         │
              ┌─────▼──────┐           ┌──────┴──────┐
              │  Write DB  │──Events──▶│  Read DB    │
              │ (PostgreSQL│           │ (Elastic/   │
              │  normalized)│          │  MongoDB)   │
              └────────────┘           └─────────────┘
                    │
              ┌─────▼──────┐
              │  Event Bus │
              │ (RabbitMQ/ │
              │  Kafka)    │
              └────────────┘
```

### ডায়াগ্রাম ৩: CQRS + Event Sourcing

```
     ┌──────────┐      ┌──────────────┐      ┌──────────────┐
     │  Client  │─CMD─▶│  Command     │─────▶│  Aggregate   │
     └──────────┘      │  Handler     │      │  (Domain)    │
           │           └──────────────┘      └──────┬───────┘
           │                                        │ Events
           │           ┌──────────────┐      ┌──────▼───────┐
           │           │  Event       │◀─────│  Event       │
           │           │  Handlers/   │      │  Store       │
           │           │  Projectors  │      │ (Append-only)│
           │           └──────┬───────┘      └──────────────┘
           │                  │
           │    ┌─────────────┼─────────────┐
           │    │             │             │
           │  ┌─▼──────┐ ┌───▼────┐ ┌──────▼──┐
           │  │Elastic │ │ Redis  │ │Postgres │
           │  │Search  │ │ Cache  │ │Reporting│
           │  │(Search)│ │(Fast)  │ │(Complex)│
           │  └───▲────┘ └───▲────┘ └────▲────┘
           │      │          │           │
           └──QUERY──────────┴───────────┘
```

### ডায়াগ্রাম ৪: Data Flow

```
Command Flow:
  Client ──▶ API Gateway ──▶ Command Bus ──▶ Validation Middleware
    ──▶ Command Handler ──▶ Domain Logic ──▶ Write DB ──▶ Publish Event

Query Flow:
  Client ──▶ API Gateway ──▶ Query Bus ──▶ Cache Middleware
    ──▶ Query Handler ──▶ Read DB/Cache ──▶ DTO ──▶ Response

Sync Flow:
  Event Published ──▶ Event Bus ──▶ Projection Handler
    ──▶ Update Read Model ──▶ Invalidate Cache
```

---

## 💻 CQRS Implementation Levels

### Level 1: Same Database, Separate Models

সবচেয়ে সহজ CQRS — একই database, কিন্তু write এবং read-এর জন্য আলাদা model/class ব্যবহার।

#### PHP (Laravel) Implementation

```php
<?php

declare(strict_types=1);

// ========== Command Side ==========

// Command DTO
final readonly class CreateProductCommand
{
    public function __construct(
        public string $name,
        public string $description,
        public float $price,
        public int $stockQuantity,
        public string $categoryId,
        public string $sellerId,
    ) {}
}

// Write Model — Eloquent Model with business logic
final class Product extends Model
{
    protected $fillable = ['name', 'description', 'price', 'stock_quantity', 'category_id', 'seller_id', 'status'];

    public function reduceStock(int $quantity): void
    {
        if ($this->stock_quantity < $quantity) {
            throw new InsufficientStockException(
                "Product {$this->id} has only {$this->stock_quantity} items"
            );
        }
        $this->stock_quantity -= $quantity;
        $this->save();
    }

    public function activate(): void
    {
        if (empty($this->name) || $this->price <= 0) {
            throw new InvalidProductException('Product must have name and valid price');
        }
        $this->status = 'active';
        $this->save();
    }
}

// Command Handler
final readonly class CreateProductHandler
{
    public function __construct(
        private Product $product,
        private EventDispatcher $events,
    ) {}

    public function handle(CreateProductCommand $command): string
    {
        $product = $this->product->create([
            'name' => $command->name,
            'description' => $command->description,
            'price' => $command->price,
            'stock_quantity' => $command->stockQuantity,
            'category_id' => $command->categoryId,
            'seller_id' => $command->sellerId,
            'status' => 'draft',
        ]);

        $this->events->dispatch(new ProductCreated($product->id, $command->name, $command->price));

        return $product->id;
    }
}

// ========== Query Side ==========

// Read-only DTO
final readonly class ProductListItem
{
    public function __construct(
        public string $id,
        public string $name,
        public float $price,
        public string $categoryName,
        public int $reviewCount,
        public float $averageRating,
        public bool $inStock,
    ) {}
}

// Query
final readonly class SearchProductsQuery
{
    public function __construct(
        public ?string $keyword = null,
        public ?string $categoryId = null,
        public ?float $minPrice = null,
        public ?float $maxPrice = null,
        public string $sortBy = 'relevance',
        public int $page = 1,
        public int $perPage = 20,
    ) {}
}

// Query Handler — optimized reads with joins and denormalization
final readonly class SearchProductsHandler
{
    public function __construct(private DB $db) {}

    /** @return ProductListItem[] */
    public function handle(SearchProductsQuery $query): array
    {
        $builder = $this->db->table('products')
            ->join('categories', 'products.category_id', '=', 'categories.id')
            ->leftJoin('reviews', 'products.id', '=', 'reviews.product_id')
            ->select([
                'products.id',
                'products.name',
                'products.price',
                'categories.name as category_name',
                DB::raw('COUNT(reviews.id) as review_count'),
                DB::raw('COALESCE(AVG(reviews.rating), 0) as average_rating'),
                DB::raw('products.stock_quantity > 0 as in_stock'),
            ])
            ->where('products.status', 'active')
            ->groupBy('products.id', 'products.name', 'products.price',
                       'categories.name', 'products.stock_quantity');

        if ($query->keyword) {
            $builder->where('products.name', 'ILIKE', "%{$query->keyword}%");
        }

        if ($query->categoryId) {
            $builder->where('products.category_id', $query->categoryId);
        }

        if ($query->minPrice !== null) {
            $builder->where('products.price', '>=', $query->minPrice);
        }

        if ($query->maxPrice !== null) {
            $builder->where('products.price', '<=', $query->maxPrice);
        }

        return $builder
            ->offset(($query->page - 1) * $query->perPage)
            ->limit($query->perPage)
            ->get()
            ->map(fn ($row) => new ProductListItem(
                id: $row->id,
                name: $row->name,
                price: (float) $row->price,
                categoryName: $row->category_name,
                reviewCount: (int) $row->review_count,
                averageRating: round((float) $row->average_rating, 1),
                inStock: (bool) $row->in_stock,
            ))
            ->all();
    }
}
```

#### Node.js Implementation

```javascript
// ========== Command Side ==========

// Command DTO
class CreateProductCommand {
  /** @readonly */ name;
  /** @readonly */ description;
  /** @readonly */ price;
  /** @readonly */ stockQuantity;
  /** @readonly */ categoryId;
  /** @readonly */ sellerId;

  constructor({ name, description, price, stockQuantity, categoryId, sellerId }) {
    Object.freeze(Object.assign(this, { name, description, price, stockQuantity, categoryId, sellerId }));
  }
}

// Write Model — domain logic সহ
class ProductWriteModel {
  #pool; // PostgreSQL pool

  constructor(pool) {
    this.#pool = pool;
  }

  async create(command) {
    const { rows: [product] } = await this.#pool.query(
      `INSERT INTO products (name, description, price, stock_quantity, category_id, seller_id, status)
       VALUES ($1, $2, $3, $4, $5, $6, 'draft') RETURNING id`,
      [command.name, command.description, command.price, command.stockQuantity, command.categoryId, command.sellerId]
    );
    return product.id;
  }

  async reduceStock(productId, quantity) {
    const { rows: [product] } = await this.#pool.query(
      'SELECT stock_quantity FROM products WHERE id = $1 FOR UPDATE',
      [productId]
    );

    if (!product || product.stock_quantity < quantity) {
      throw new Error(`Insufficient stock for product ${productId}`);
    }

    await this.#pool.query(
      'UPDATE products SET stock_quantity = stock_quantity - $1 WHERE id = $2',
      [quantity, productId]
    );
  }
}

// Command Handler
class CreateProductHandler {
  #writeModel;
  #eventBus;

  constructor(writeModel, eventBus) {
    this.#writeModel = writeModel;
    this.#eventBus = eventBus;
  }

  async handle(command) {
    const productId = await this.#writeModel.create(command);

    await this.#eventBus.publish('ProductCreated', {
      productId,
      name: command.name,
      price: command.price,
      occurredAt: new Date().toISOString(),
    });

    return productId;
  }
}

// ========== Query Side ==========

// Query Handler — optimized read
class SearchProductsHandler {
  #pool;

  constructor(pool) {
    this.#pool = pool;
  }

  async handle({ keyword, categoryId, minPrice, maxPrice, sortBy = 'relevance', page = 1, perPage = 20 }) {
    const conditions = ["p.status = 'active'"];
    const params = [];
    let paramIndex = 1;

    if (keyword) {
      conditions.push(`p.name ILIKE $${paramIndex++}`);
      params.push(`%${keyword}%`);
    }
    if (categoryId) {
      conditions.push(`p.category_id = $${paramIndex++}`);
      params.push(categoryId);
    }
    if (minPrice != null) {
      conditions.push(`p.price >= $${paramIndex++}`);
      params.push(minPrice);
    }
    if (maxPrice != null) {
      conditions.push(`p.price <= $${paramIndex++}`);
      params.push(maxPrice);
    }

    const offset = (page - 1) * perPage;
    params.push(perPage, offset);

    const { rows } = await this.#pool.query(`
      SELECT p.id, p.name, p.price, c.name AS category_name,
             COUNT(r.id) AS review_count,
             COALESCE(AVG(r.rating), 0) AS average_rating,
             p.stock_quantity > 0 AS in_stock
      FROM products p
      JOIN categories c ON p.category_id = c.id
      LEFT JOIN reviews r ON p.id = r.product_id
      WHERE ${conditions.join(' AND ')}
      GROUP BY p.id, p.name, p.price, c.name, p.stock_quantity
      LIMIT $${paramIndex++} OFFSET $${paramIndex}
    `, params);

    return rows.map(row => ({
      id: row.id,
      name: row.name,
      price: parseFloat(row.price),
      categoryName: row.category_name,
      reviewCount: parseInt(row.review_count),
      averageRating: Math.round(parseFloat(row.average_rating) * 10) / 10,
      inStock: row.in_stock,
    }));
  }
}
```

---

### Level 2: Separate Read/Write Databases

এই level-এ Write DB (PostgreSQL) এবং Read DB (Elasticsearch/MongoDB) সম্পূর্ণ আলাদা। Event-based synchronization ব্যবহার করা হয়।

#### PHP (Laravel) Implementation

```php
<?php

declare(strict_types=1);

// Write Side — PostgreSQL
final readonly class PlaceOrderHandler
{
    public function __construct(
        private OrderRepository $orderRepo,
        private EventBus $eventBus,
        private Connection $db,
    ) {}

    public function handle(PlaceOrderCommand $command): string
    {
        return $this->db->transaction(function () use ($command) {
            $order = Order::create(
                customerId: $command->customerId,
                items: $command->items,
                shippingAddress: $command->shippingAddress,
            );

            $this->orderRepo->save($order);

            // Domain Events publish — read model async update হবে
            foreach ($order->pullDomainEvents() as $event) {
                $this->eventBus->publish($event);
            }

            return $order->id;
        });
    }
}

// Read Side Projector — Elasticsearch এ sync করে
final readonly class OrderProjector
{
    public function __construct(
        private Client $elasticsearch,
    ) {}

    public function onOrderPlaced(OrderPlaced $event): void
    {
        $this->elasticsearch->index([
            'index' => 'orders',
            'id'    => $event->orderId,
            'body'  => [
                'order_id'         => $event->orderId,
                'customer_id'      => $event->customerId,
                'customer_name'    => $event->customerName,
                'items'            => $event->items,
                'total_amount'     => $event->totalAmount,
                'status'           => 'placed',
                'shipping_address' => $event->shippingAddress,
                'placed_at'        => $event->occurredAt->format('c'),
            ],
        ]);
    }

    public function onOrderStatusChanged(OrderStatusChanged $event): void
    {
        $this->elasticsearch->update([
            'index' => 'orders',
            'id'    => $event->orderId,
            'body'  => [
                'doc' => [
                    'status'     => $event->newStatus->value,
                    'updated_at' => $event->occurredAt->format('c'),
                ],
            ],
        ]);
    }
}

// Query Handler — Elasticsearch থেকে read
final readonly class SearchOrdersHandler
{
    public function __construct(private Client $elasticsearch) {}

    public function handle(SearchOrdersQuery $query): OrderSearchResult
    {
        $must = [['term' => ['customer_id' => $query->customerId]]];

        if ($query->status) {
            $must[] = ['term' => ['status' => $query->status->value]];
        }

        if ($query->dateFrom) {
            $must[] = ['range' => ['placed_at' => ['gte' => $query->dateFrom->format('c')]]];
        }

        $response = $this->elasticsearch->search([
            'index' => 'orders',
            'body'  => [
                'query' => ['bool' => ['must' => $must]],
                'sort'  => [['placed_at' => 'desc']],
                'from'  => ($query->page - 1) * $query->perPage,
                'size'  => $query->perPage,
            ],
        ]);

        $items = array_map(
            fn (array $hit) => new OrderSummaryDTO(...$hit['_source']),
            $response['hits']['hits']
        );

        return new OrderSearchResult(
            items: $items,
            total: $response['hits']['total']['value'],
            page: $query->page,
        );
    }
}
```

#### Node.js Implementation

```javascript
import { Client as ESClient } from '@elastic/elasticsearch';
import pg from 'pg';

// Write Side — PostgreSQL
class PlaceOrderHandler {
  #pool;
  #eventBus;

  constructor(pool, eventBus) {
    this.#pool = pool;
    this.#eventBus = eventBus;
  }

  async handle(command) {
    const client = await this.#pool.connect();
    try {
      await client.query('BEGIN');

      const { rows: [order] } = await client.query(
        `INSERT INTO orders (customer_id, shipping_address, status, total_amount)
         VALUES ($1, $2, 'placed', $3) RETURNING id`,
        [command.customerId, JSON.stringify(command.shippingAddress), command.totalAmount]
      );

      for (const item of command.items) {
        await client.query(
          `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
           VALUES ($1, $2, $3, $4)`,
          [order.id, item.productId, item.quantity, item.unitPrice]
        );
      }

      await client.query('COMMIT');

      await this.#eventBus.publish('OrderPlaced', {
        orderId: order.id,
        customerId: command.customerId,
        items: command.items,
        totalAmount: command.totalAmount,
        shippingAddress: command.shippingAddress,
        occurredAt: new Date().toISOString(),
      });

      return order.id;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}

// Read Side Projector — Elasticsearch sync
class OrderProjector {
  #esClient;

  constructor(esClient) {
    this.#esClient = esClient;
  }

  async onOrderPlaced(event) {
    await this.#esClient.index({
      index: 'orders',
      id: event.orderId,
      document: {
        orderId: event.orderId,
        customerId: event.customerId,
        items: event.items,
        totalAmount: event.totalAmount,
        status: 'placed',
        shippingAddress: event.shippingAddress,
        placedAt: event.occurredAt,
      },
    });
  }

  async onOrderStatusChanged(event) {
    await this.#esClient.update({
      index: 'orders',
      id: event.orderId,
      doc: { status: event.newStatus, updatedAt: event.occurredAt },
    });
  }
}

// Query Handler — Elasticsearch read
class SearchOrdersHandler {
  #esClient;

  constructor(esClient) {
    this.#esClient = esClient;
  }

  async handle({ customerId, status, dateFrom, page = 1, perPage = 20 }) {
    const must = [{ term: { customerId } }];
    if (status) must.push({ term: { status } });
    if (dateFrom) must.push({ range: { placedAt: { gte: dateFrom } } });

    const result = await this.#esClient.search({
      index: 'orders',
      query: { bool: { must } },
      sort: [{ placedAt: 'desc' }],
      from: (page - 1) * perPage,
      size: perPage,
    });

    return {
      items: result.hits.hits.map(hit => hit._source),
      total: result.hits.total.value,
      page,
    };
  }
}
```

---

### Level 3: CQRS + Event Sourcing

সবচেয়ে advanced level — Event Store-ই হলো single source of truth। Aggregate-এর বর্তমান state events replay করে তৈরি হয়। Read models হলো events থেকে তৈরি projections।

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// ========== Domain Events ==========
enum OrderEventType: string
{
    case Placed = 'order.placed';
    case ItemAdded = 'order.item_added';
    case Confirmed = 'order.confirmed';
    case Shipped = 'order.shipped';
    case Cancelled = 'order.cancelled';
}

final readonly class DomainEvent
{
    public function __construct(
        public string $eventId,
        public string $aggregateId,
        public OrderEventType $type,
        public array $payload,
        public int $version,
        public DateTimeImmutable $occurredAt,
    ) {}
}

// ========== Aggregate ==========
final class OrderAggregate
{
    private string $id;
    private string $customerId;
    private array $items = [];
    private string $status = 'draft';
    private float $totalAmount = 0;
    private int $version = 0;

    /** @var DomainEvent[] */
    private array $uncommittedEvents = [];

    public static function place(string $orderId, string $customerId, array $items): self
    {
        $order = new self();

        $totalAmount = array_reduce(
            $items,
            fn (float $sum, array $item) => $sum + ($item['unitPrice'] * $item['quantity']),
            0.0
        );

        $order->apply(new DomainEvent(
            eventId: Uuid::uuid4()->toString(),
            aggregateId: $orderId,
            type: OrderEventType::Placed,
            payload: [
                'customer_id' => $customerId,
                'items' => $items,
                'total_amount' => $totalAmount,
            ],
            version: 1,
            occurredAt: new DateTimeImmutable(),
        ));

        return $order;
    }

    public function confirm(): void
    {
        if ($this->status !== 'placed') {
            throw new InvalidOrderStateException("Cannot confirm order in '{$this->status}' status");
        }

        $this->apply(new DomainEvent(
            eventId: Uuid::uuid4()->toString(),
            aggregateId: $this->id,
            type: OrderEventType::Confirmed,
            payload: ['confirmed_at' => (new DateTimeImmutable())->format('c')],
            version: $this->version + 1,
            occurredAt: new DateTimeImmutable(),
        ));
    }

    public function cancel(string $reason): void
    {
        if (in_array($this->status, ['shipped', 'cancelled'], true)) {
            throw new InvalidOrderStateException("Cannot cancel order in '{$this->status}' status");
        }

        $this->apply(new DomainEvent(
            eventId: Uuid::uuid4()->toString(),
            aggregateId: $this->id,
            type: OrderEventType::Cancelled,
            payload: ['reason' => $reason],
            version: $this->version + 1,
            occurredAt: new DateTimeImmutable(),
        ));
    }

    private function apply(DomainEvent $event): void
    {
        $this->applyEvent($event);
        $this->uncommittedEvents[] = $event;
    }

    private function applyEvent(DomainEvent $event): void
    {
        match ($event->type) {
            OrderEventType::Placed => $this->applyPlaced($event),
            OrderEventType::Confirmed => $this->status = 'confirmed',
            OrderEventType::Shipped => $this->status = 'shipped',
            OrderEventType::Cancelled => $this->status = 'cancelled',
            default => null,
        };
        $this->version = $event->version;
    }

    private function applyPlaced(DomainEvent $event): void
    {
        $this->id = $event->aggregateId;
        $this->customerId = $event->payload['customer_id'];
        $this->items = $event->payload['items'];
        $this->totalAmount = $event->payload['total_amount'];
        $this->status = 'placed';
    }

    public static function fromEvents(array $events): self
    {
        $order = new self();
        foreach ($events as $event) {
            $order->applyEvent($event);
        }
        return $order;
    }

    /** @return DomainEvent[] */
    public function pullUncommittedEvents(): array
    {
        $events = $this->uncommittedEvents;
        $this->uncommittedEvents = [];
        return $events;
    }
}

// ========== Event Store ==========
final readonly class PostgresEventStore
{
    public function __construct(private Connection $db) {}

    public function append(string $aggregateId, array $events, int $expectedVersion): void
    {
        $this->db->transaction(function () use ($aggregateId, $events, $expectedVersion) {
            // Optimistic concurrency check
            $currentVersion = $this->db->table('event_store')
                ->where('aggregate_id', $aggregateId)
                ->max('version') ?? 0;

            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException(
                    "Expected version {$expectedVersion}, got {$currentVersion}"
                );
            }

            foreach ($events as $event) {
                $this->db->table('event_store')->insert([
                    'event_id'     => $event->eventId,
                    'aggregate_id' => $event->aggregateId,
                    'event_type'   => $event->type->value,
                    'payload'      => json_encode($event->payload),
                    'version'      => $event->version,
                    'occurred_at'  => $event->occurredAt,
                ]);
            }
        });
    }

    /** @return DomainEvent[] */
    public function getEvents(string $aggregateId): array
    {
        return $this->db->table('event_store')
            ->where('aggregate_id', $aggregateId)
            ->orderBy('version')
            ->get()
            ->map(fn ($row) => new DomainEvent(
                eventId: $row->event_id,
                aggregateId: $row->aggregate_id,
                type: OrderEventType::from($row->event_type),
                payload: json_decode($row->payload, true),
                version: $row->version,
                occurredAt: new DateTimeImmutable($row->occurred_at),
            ))
            ->all();
    }
}
```

#### Node.js Implementation

```javascript
// ========== Aggregate ==========
class OrderAggregate {
  #id;
  #customerId;
  #items = [];
  #status = 'draft';
  #totalAmount = 0;
  #version = 0;
  #uncommittedEvents = [];

  static place(orderId, customerId, items) {
    const order = new OrderAggregate();
    const totalAmount = items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);

    order.#applyAndRecord({
      eventId: crypto.randomUUID(),
      aggregateId: orderId,
      type: 'order.placed',
      payload: { customerId, items, totalAmount },
      version: 1,
      occurredAt: new Date().toISOString(),
    });

    return order;
  }

  confirm() {
    if (this.#status !== 'placed') {
      throw new Error(`Cannot confirm order in '${this.#status}' status`);
    }
    this.#applyAndRecord({
      eventId: crypto.randomUUID(),
      aggregateId: this.#id,
      type: 'order.confirmed',
      payload: { confirmedAt: new Date().toISOString() },
      version: this.#version + 1,
      occurredAt: new Date().toISOString(),
    });
  }

  cancel(reason) {
    if (['shipped', 'cancelled'].includes(this.#status)) {
      throw new Error(`Cannot cancel order in '${this.#status}' status`);
    }
    this.#applyAndRecord({
      eventId: crypto.randomUUID(),
      aggregateId: this.#id,
      type: 'order.cancelled',
      payload: { reason },
      version: this.#version + 1,
      occurredAt: new Date().toISOString(),
    });
  }

  #applyAndRecord(event) {
    this.#applyEvent(event);
    this.#uncommittedEvents.push(event);
  }

  #applyEvent(event) {
    switch (event.type) {
      case 'order.placed':
        this.#id = event.aggregateId;
        this.#customerId = event.payload.customerId;
        this.#items = event.payload.items;
        this.#totalAmount = event.payload.totalAmount;
        this.#status = 'placed';
        break;
      case 'order.confirmed': this.#status = 'confirmed'; break;
      case 'order.shipped': this.#status = 'shipped'; break;
      case 'order.cancelled': this.#status = 'cancelled'; break;
    }
    this.#version = event.version;
  }

  static fromEvents(events) {
    const order = new OrderAggregate();
    for (const event of events) order.#applyEvent(event);
    return order;
  }

  pullUncommittedEvents() {
    const events = [...this.#uncommittedEvents];
    this.#uncommittedEvents = [];
    return events;
  }

  get version() { return this.#version; }
}

// ========== Event Store ==========
class PostgresEventStore {
  #pool;

  constructor(pool) {
    this.#pool = pool;
  }

  async append(aggregateId, events, expectedVersion) {
    const client = await this.#pool.connect();
    try {
      await client.query('BEGIN');

      const { rows: [{ max: currentVersion }] } = await client.query(
        'SELECT COALESCE(MAX(version), 0) as max FROM event_store WHERE aggregate_id = $1',
        [aggregateId]
      );

      if (parseInt(currentVersion) !== expectedVersion) {
        throw new Error(`Concurrency conflict: expected ${expectedVersion}, got ${currentVersion}`);
      }

      for (const event of events) {
        await client.query(
          `INSERT INTO event_store (event_id, aggregate_id, event_type, payload, version, occurred_at)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [event.eventId, event.aggregateId, event.type, JSON.stringify(event.payload), event.version, event.occurredAt]
        );
      }

      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  async getEvents(aggregateId) {
    const { rows } = await this.#pool.query(
      'SELECT * FROM event_store WHERE aggregate_id = $1 ORDER BY version',
      [aggregateId]
    );
    return rows.map(row => ({
      eventId: row.event_id,
      aggregateId: row.aggregate_id,
      type: row.event_type,
      payload: row.payload,
      version: row.version,
      occurredAt: row.occurred_at,
    }));
  }
}
```

---

## 🔥 Complex Scenarios

### ১. Command Bus Implementation

Command Bus হলো একটি middleware pipeline যা command কে appropriate handler-এ route করে। প্রতিটি command এর জন্য exactly একটি handler থাকে।

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// Command Bus Interface
interface CommandBus
{
    public function dispatch(object $command): mixed;
}

// Middleware Interface
interface CommandMiddleware
{
    public function handle(object $command, Closure $next): mixed;
}

// Validation Middleware
final readonly class ValidationMiddleware implements CommandMiddleware
{
    public function __construct(private ValidatorFactory $validator) {}

    public function handle(object $command, Closure $next): mixed
    {
        if (method_exists($command, 'rules')) {
            $validator = $this->validator->make(
                (array) $command,
                $command->rules()
            );

            if ($validator->fails()) {
                throw new CommandValidationException($validator->errors()->all());
            }
        }

        return $next($command);
    }
}

// Transaction Middleware
final readonly class TransactionMiddleware implements CommandMiddleware
{
    public function __construct(private Connection $db) {}

    public function handle(object $command, Closure $next): mixed
    {
        return $this->db->transaction(fn () => $next($command));
    }
}

// Logging Middleware
final readonly class LoggingMiddleware implements CommandMiddleware
{
    public function __construct(private LoggerInterface $logger) {}

    public function handle(object $command, Closure $next): mixed
    {
        $commandName = $command::class;
        $this->logger->info("Dispatching command: {$commandName}", [
            'payload' => (array) $command,
        ]);

        $startTime = microtime(true);

        try {
            $result = $next($command);
            $duration = round((microtime(true) - $startTime) * 1000, 2);
            $this->logger->info("Command {$commandName} completed in {$duration}ms");
            return $result;
        } catch (Throwable $e) {
            $this->logger->error("Command {$commandName} failed: {$e->getMessage()}");
            throw $e;
        }
    }
}

// Command Bus Implementation
final class MiddlewareCommandBus implements CommandBus
{
    /** @var array<class-string, callable> */
    private array $handlers = [];

    /** @var CommandMiddleware[] */
    private array $middlewares = [];

    public function register(string $commandClass, callable $handler): void
    {
        $this->handlers[$commandClass] = $handler;
    }

    public function addMiddleware(CommandMiddleware $middleware): void
    {
        $this->middlewares[] = $middleware;
    }

    public function dispatch(object $command): mixed
    {
        $handler = $this->handlers[$command::class]
            ?? throw new HandlerNotFoundException("No handler for " . $command::class);

        $pipeline = array_reduce(
            array_reverse($this->middlewares),
            fn (Closure $next, CommandMiddleware $middleware) =>
                fn (object $cmd) => $middleware->handle($cmd, $next),
            fn (object $cmd) => $handler($cmd),
        );

        return $pipeline($command);
    }
}

// ব্যবহার
$bus = new MiddlewareCommandBus();
$bus->addMiddleware(new LoggingMiddleware($logger));
$bus->addMiddleware(new ValidationMiddleware($validatorFactory));
$bus->addMiddleware(new TransactionMiddleware($db));

$bus->register(PlaceOrderCommand::class, fn ($cmd) => $placeOrderHandler->handle($cmd));
$bus->register(CancelOrderCommand::class, fn ($cmd) => $cancelOrderHandler->handle($cmd));

$orderId = $bus->dispatch(new PlaceOrderCommand(
    customerId: 'cust-123',
    items: [['productId' => 'prod-1', 'quantity' => 2, 'unitPrice' => 500.00]],
    shippingAddress: ['city' => 'Dhaka', 'area' => 'Mirpur'],
));
```

#### Node.js Implementation

```javascript
// Middleware-based Command Bus
class CommandBus {
  #handlers = new Map();
  #middlewares = [];

  register(commandName, handler) {
    this.#handlers.set(commandName, handler);
  }

  use(middleware) {
    this.#middlewares.push(middleware);
  }

  async dispatch(command) {
    const commandName = command.constructor.name;
    const handler = this.#handlers.get(commandName);
    if (!handler) throw new Error(`No handler registered for ${commandName}`);

    const pipeline = this.#middlewares.reduceRight(
      (next, middleware) => (cmd) => middleware(cmd, next),
      (cmd) => handler(cmd),
    );

    return pipeline(command);
  }
}

// Validation Middleware
const validationMiddleware = async (command, next) => {
  if (typeof command.validate === 'function') {
    const errors = command.validate();
    if (errors.length > 0) {
      throw new ValidationError('Command validation failed', errors);
    }
  }
  return next(command);
};

// Transaction Middleware
const createTransactionMiddleware = (pool) => async (command, next) => {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    command._dbClient = client;
    const result = await next(command);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
};

// Logging Middleware
const loggingMiddleware = (logger) => async (command, next) => {
  const name = command.constructor.name;
  const start = performance.now();
  logger.info(`Dispatching: ${name}`, { payload: command });

  try {
    const result = await next(command);
    logger.info(`${name} completed in ${(performance.now() - start).toFixed(2)}ms`);
    return result;
  } catch (err) {
    logger.error(`${name} failed: ${err.message}`);
    throw err;
  }
};

// ব্যবহার
const commandBus = new CommandBus();
commandBus.use(loggingMiddleware(logger));
commandBus.use(validationMiddleware);
commandBus.use(createTransactionMiddleware(pool));

commandBus.register('PlaceOrderCommand', (cmd) => placeOrderHandler.handle(cmd));

const orderId = await commandBus.dispatch(new PlaceOrderCommand({
  customerId: 'cust-123',
  items: [{ productId: 'prod-1', quantity: 2, unitPrice: 500 }],
  shippingAddress: { city: 'Dhaka', area: 'Mirpur' },
}));
```

---

### ২. Query Bus Implementation

Query Bus command bus-এর মতোই, তবে এখানে caching middleware বিশেষভাবে গুরুত্বপূর্ণ।

#### PHP Implementation

```php
<?php

declare(strict_types=1);

interface QueryBus
{
    public function ask(object $query): mixed;
}

// Cache Middleware — Query-এর জন্য সবচেয়ে critical middleware
final readonly class QueryCacheMiddleware
{
    public function __construct(
        private CacheInterface $cache,
        private int $defaultTtl = 300,
    ) {}

    public function handle(object $query, Closure $next): mixed
    {
        if (!$query instanceof CacheableQuery) {
            return $next($query);
        }

        $cacheKey = $query->cacheKey();
        $ttl = $query->cacheTtl() ?? $this->defaultTtl;

        return $this->cache->remember($cacheKey, $ttl, fn () => $next($query));
    }
}

// Cacheable Query Interface
interface CacheableQuery
{
    public function cacheKey(): string;
    public function cacheTtl(): ?int;
}

// Concrete Query
final readonly class GetProductDetailsQuery implements CacheableQuery
{
    public function __construct(public string $productId) {}

    public function cacheKey(): string
    {
        return "product:details:{$this->productId}";
    }

    public function cacheTtl(): ?int
    {
        return 600; // 10 minutes
    }
}

// Query Bus Implementation
final class MiddlewareQueryBus implements QueryBus
{
    private array $handlers = [];
    private array $middlewares = [];

    public function register(string $queryClass, callable $handler): void
    {
        $this->handlers[$queryClass] = $handler;
    }

    public function addMiddleware(callable $middleware): void
    {
        $this->middlewares[] = $middleware;
    }

    public function ask(object $query): mixed
    {
        $handler = $this->handlers[$query::class]
            ?? throw new \RuntimeException("No handler for " . $query::class);

        $pipeline = array_reduce(
            array_reverse($this->middlewares),
            fn (Closure $next, callable $middleware) =>
                fn (object $q) => $middleware->handle($q, $next),
            fn (object $q) => $handler($q),
        );

        return $pipeline($query);
    }
}
```

#### Node.js Implementation

```javascript
class QueryBus {
  #handlers = new Map();
  #middlewares = [];

  register(queryName, handler) {
    this.#handlers.set(queryName, handler);
  }

  use(middleware) {
    this.#middlewares.push(middleware);
  }

  async ask(query) {
    const name = query.constructor.name;
    const handler = this.#handlers.get(name);
    if (!handler) throw new Error(`No handler for query: ${name}`);

    const pipeline = this.#middlewares.reduceRight(
      (next, mw) => (q) => mw(q, next),
      (q) => handler(q),
    );

    return pipeline(query);
  }
}

// Cache Middleware
const createCacheMiddleware = (redis) => async (query, next) => {
  if (typeof query.cacheKey !== 'function') return next(query);

  const key = query.cacheKey();
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const result = await next(query);
  const ttl = query.cacheTtl?.() ?? 300;
  await redis.setex(key, ttl, JSON.stringify(result));

  return result;
};

// Cacheable Query
class GetProductDetailsQuery {
  constructor(productId) {
    this.productId = productId;
  }
  cacheKey() { return `product:details:${this.productId}`; }
  cacheTtl() { return 600; }
}

// ব্যবহার
const queryBus = new QueryBus();
queryBus.use(createCacheMiddleware(redisClient));
queryBus.register('GetProductDetailsQuery', (q) => productQueryHandler.handle(q));

const product = await queryBus.ask(new GetProductDetailsQuery('prod-456'));
```

---

### ৩. Event-based Synchronization

Domain events ব্যবহার করে read model synchronize করা এবং projection rebuild করার mechanism।

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// Projection Manager — Read model rebuild করতে পারে
final class ProjectionManager
{
    /** @var array<string, ProjectionHandler[]> */
    private array $projectors = [];

    public function register(string $eventType, ProjectionHandler $handler): void
    {
        $this->projectors[$eventType][] = $handler;
    }

    public function handle(DomainEvent $event): void
    {
        $handlers = $this->projectors[$event->type->value] ?? [];
        foreach ($handlers as $handler) {
            try {
                $handler->project($event);
            } catch (Throwable $e) {
                // Dead letter queue-তে পাঠাও, বাকি projections চালিয়ে যাও
                $this->sendToDeadLetterQueue($event, $handler::class, $e);
            }
        }
    }

    // সম্পূর্ণ projection rebuild — Event Store থেকে সব event replay করে
    public function rebuild(string $projectorClass, EventStore $eventStore): void
    {
        $projector = $this->findProjector($projectorClass);
        $projector->reset(); // বিদ্যমান read model মুছে ফেলো

        $lastPosition = 0;
        $batchSize = 1000;

        while (true) {
            $events = $eventStore->getEventsFrom($lastPosition, $batchSize);

            if (empty($events)) break;

            foreach ($events as $event) {
                $projector->project($event);
                $lastPosition = $event->position;
            }
        }
    }
}

// Order Summary Projection
final class OrderSummaryProjection implements ProjectionHandler
{
    public function __construct(
        private Connection $readDb,
        private CacheInterface $cache,
    ) {}

    public function project(DomainEvent $event): void
    {
        match ($event->type) {
            OrderEventType::Placed => $this->onOrderPlaced($event),
            OrderEventType::Confirmed => $this->updateStatus($event->aggregateId, 'confirmed'),
            OrderEventType::Shipped => $this->updateStatus($event->aggregateId, 'shipped'),
            OrderEventType::Cancelled => $this->onOrderCancelled($event),
            default => null,
        };
    }

    private function onOrderPlaced(DomainEvent $event): void
    {
        $this->readDb->table('order_summaries')->insert([
            'order_id'     => $event->aggregateId,
            'customer_id'  => $event->payload['customer_id'],
            'total_amount' => $event->payload['total_amount'],
            'item_count'   => count($event->payload['items']),
            'status'       => 'placed',
            'placed_at'    => $event->occurredAt,
        ]);

        $this->cache->forget("customer:orders:{$event->payload['customer_id']}");
    }

    private function updateStatus(string $orderId, string $status): void
    {
        $this->readDb->table('order_summaries')
            ->where('order_id', $orderId)
            ->update(['status' => $status, 'updated_at' => now()]);
    }

    private function onOrderCancelled(DomainEvent $event): void
    {
        $this->updateStatus($event->aggregateId, 'cancelled');
        // Inventory restore event trigger করো
    }

    public function reset(): void
    {
        $this->readDb->table('order_summaries')->truncate();
    }
}
```

#### Node.js Implementation

```javascript
class ProjectionManager {
  #projectors = new Map();
  #deadLetterQueue;

  constructor(deadLetterQueue) {
    this.#deadLetterQueue = deadLetterQueue;
  }

  register(eventType, handler) {
    if (!this.#projectors.has(eventType)) {
      this.#projectors.set(eventType, []);
    }
    this.#projectors.get(eventType).push(handler);
  }

  async handle(event) {
    const handlers = this.#projectors.get(event.type) ?? [];
    await Promise.allSettled(
      handlers.map(async (handler) => {
        try {
          await handler.project(event);
        } catch (err) {
          await this.#deadLetterQueue.push({ event, handler: handler.constructor.name, error: err.message });
        }
      })
    );
  }

  async rebuild(projector, eventStore) {
    await projector.reset();
    let lastPosition = 0;
    const batchSize = 1000;

    while (true) {
      const events = await eventStore.getEventsFrom(lastPosition, batchSize);
      if (events.length === 0) break;

      for (const event of events) {
        await projector.project(event);
        lastPosition = event.position;
      }
    }
  }
}

// Order Summary Projection
class OrderSummaryProjection {
  #readDb;
  #redis;

  constructor(readDb, redis) {
    this.#readDb = readDb;
    this.#redis = redis;
  }

  async project(event) {
    const handlers = {
      'order.placed': (e) => this.#onOrderPlaced(e),
      'order.confirmed': (e) => this.#updateStatus(e.aggregateId, 'confirmed'),
      'order.shipped': (e) => this.#updateStatus(e.aggregateId, 'shipped'),
      'order.cancelled': (e) => this.#onOrderCancelled(e),
    };

    const handler = handlers[event.type];
    if (handler) await handler(event);
  }

  async #onOrderPlaced(event) {
    await this.#readDb.query(
      `INSERT INTO order_summaries (order_id, customer_id, total_amount, item_count, status, placed_at)
       VALUES ($1, $2, $3, $4, 'placed', $5)`,
      [event.aggregateId, event.payload.customerId, event.payload.totalAmount,
       event.payload.items.length, event.occurredAt]
    );
    await this.#redis.del(`customer:orders:${event.payload.customerId}`);
  }

  async #updateStatus(orderId, status) {
    await this.#readDb.query(
      'UPDATE order_summaries SET status = $1, updated_at = NOW() WHERE order_id = $2',
      [status, orderId]
    );
  }

  async #onOrderCancelled(event) {
    await this.#updateStatus(event.aggregateId, 'cancelled');
  }

  async reset() {
    await this.#readDb.query('TRUNCATE order_summaries');
  }
}
```

---

### ৪. Materialized Views / Read Models

একই event stream থেকে multiple read models তৈরি করা যায়, প্রতিটি আলাদা use case-এর জন্য optimized।

```
Events Stream
    │
    ├──▶ Elasticsearch Index (Product Search — full-text, filters)
    │
    ├──▶ Redis Hash (Product Recommendations — fast key-value)
    │
    ├──▶ PostgreSQL Table (Reporting — complex aggregations)
    │
    └──▶ MongoDB Collection (Product Catalog — nested documents)
```

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// একই ProductUpdated event থেকে তিনটি আলাদা read model update হয়

final readonly class SearchIndexProjection
{
    public function __construct(private Client $elasticsearch) {}

    public function onProductUpdated(ProductUpdated $event): void
    {
        $this->elasticsearch->index([
            'index' => 'products',
            'id'    => $event->productId,
            'body'  => [
                'name'        => $event->name,
                'description' => $event->description,
                'price'       => $event->price,
                'category'    => $event->categoryName,
                'tags'        => $event->tags,
                'in_stock'    => $event->stockQuantity > 0,
                'updated_at'  => $event->occurredAt->format('c'),
            ],
        ]);
    }
}

final readonly class RecommendationProjection
{
    public function __construct(private Redis $redis) {}

    public function onProductUpdated(ProductUpdated $event): void
    {
        // Category-wise trending products
        $this->redis->zadd(
            "category:{$event->categoryId}:products",
            $event->salesCount, // score হিসেবে sales count
            $event->productId,
        );

        // Product details cache
        $this->redis->hset("product:{$event->productId}", [
            'name'  => $event->name,
            'price' => $event->price,
            'image' => $event->imageUrl,
        ]);
    }
}

final readonly class ReportingProjection
{
    public function __construct(private Connection $reportDb) {}

    public function onProductUpdated(ProductUpdated $event): void
    {
        $this->reportDb->table('product_reports')->updateOrInsert(
            ['product_id' => $event->productId],
            [
                'name'             => $event->name,
                'price'            => $event->price,
                'category_id'      => $event->categoryId,
                'stock_quantity'   => $event->stockQuantity,
                'total_sales'      => $event->salesCount,
                'revenue'          => $event->totalRevenue,
                'last_updated'     => $event->occurredAt,
            ],
        );
    }
}
```

---

### ৫. Handling Eventual Consistency

CQRS-এ read model update হতে সময় লাগে — এটি **eventual consistency**। UI-তে এটি handle করা critical।

#### Optimistic Update Pattern

```javascript
// Frontend — React example
class OrderService {
  async placeOrder(orderData) {
    // ১. Command পাঠাও
    const { orderId } = await api.post('/commands/place-order', orderData);

    // ২. Optimistic update — UI immediately update করো
    queryCache.setQueryData(['orders', 'list'], (old) => [
      {
        id: orderId,
        ...orderData,
        status: 'placed',
        _optimistic: true, // flag: এটি এখনো confirmed না
      },
      ...old,
    ]);

    // ৩. Poll/WebSocket দিয়ে confirm হওয়া পর্যন্ত অপেক্ষা
    await this.#waitForConsistency(orderId);

    // ৪. Confirmed data দিয়ে replace করো
    queryCache.invalidateQueries(['orders', 'list']);
  }

  async #waitForConsistency(orderId, maxRetries = 10) {
    for (let i = 0; i < maxRetries; i++) {
      await new Promise((r) => setTimeout(r, 500 * (i + 1))); // exponential backoff
      try {
        const order = await api.get(`/queries/orders/${orderId}`);
        if (order) return order;
      } catch { /* read model এখনো ready না */ }
    }
    // Timeout — user-কে জানাও যে order process হচ্ছে
    showNotification('আপনার অর্ডার প্রসেস হচ্ছে, কিছুক্ষণ পর দেখুন');
  }
}
```

#### WebSocket-based Confirmation (Node.js Backend)

```javascript
import { WebSocketServer } from 'ws';

class ConsistencyNotifier {
  #wss;
  #connections = new Map(); // userId -> WebSocket

  constructor(server) {
    this.#wss = new WebSocketServer({ server });
    this.#wss.on('connection', (ws, req) => {
      const userId = this.#extractUserId(req);
      this.#connections.set(userId, ws);
      ws.on('close', () => this.#connections.delete(userId));
    });
  }

  // Projection update হলে client-কে notify করো
  async onProjectionUpdated(event) {
    const ws = this.#connections.get(event.userId);
    if (ws?.readyState === 1) {
      ws.send(JSON.stringify({
        type: 'READ_MODEL_UPDATED',
        entity: event.entityType,
        entityId: event.entityId,
        timestamp: event.occurredAt,
      }));
    }
  }

  #extractUserId(req) {
    const url = new URL(req.url, 'http://localhost');
    return url.searchParams.get('userId');
  }
}
```

#### Stale Data Handling Strategies

```
Strategy ১: Version-based
  - Write response-এ version number return করো
  - Read request-এ minimum version চাও
  - Read model-এর version কম হলে সরাসরি write DB থেকে পড়ো (fallback)

Strategy ২: Timestamp-based
  - Command এর timestamp ট্র্যাক করো
  - Query response-এ "data as of" timestamp দেখাও
  - UI-তে "Last updated: 2 seconds ago" দেখাও

Strategy ৩: Causal Consistency Token
  - Command response-এ consistency token দাও
  - Query-তে token পাঠাও
  - Read model ঐ token পর্যন্ত updated না হলে wait/fallback করো
```

---

### ৬. Multi-tenant CQRS

বাংলাদেশের SaaS platform-এ যেখানে একাধিক ব্যবসায়ী একই platform ব্যবহার করে (যেমন **Shurjopay**, **SSLCommerz** merchant portal):

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// Tenant-aware Command
final readonly class CreateInvoiceCommand
{
    public function __construct(
        public string $tenantId,
        public string $customerId,
        public array $lineItems,
        public string $currency = 'BDT',
    ) {}
}

// Tenant Isolation Middleware
final readonly class TenantIsolationMiddleware implements CommandMiddleware
{
    public function __construct(private TenantContext $tenantContext) {}

    public function handle(object $command, Closure $next): mixed
    {
        if (!property_exists($command, 'tenantId')) {
            throw new MissingTenantException('Command must include tenantId');
        }

        // Tenant context set করো — সব DB query-তে automatic filter হবে
        $this->tenantContext->setCurrentTenant($command->tenantId);

        try {
            return $next($command);
        } finally {
            $this->tenantContext->clear();
        }
    }
}

// Tenant-scoped Read Model Projection
final readonly class TenantInvoiceProjection implements ProjectionHandler
{
    public function __construct(private Connection $readDb) {}

    public function project(DomainEvent $event): void
    {
        if ($event->type !== 'invoice.created') return;

        // প্রতিটি tenant-এর জন্য আলাদা partition বা schema
        $this->readDb->table("tenant_{$event->payload['tenant_id']}.invoice_summaries")
            ->insert([
                'invoice_id'  => $event->aggregateId,
                'customer_id' => $event->payload['customer_id'],
                'total'       => $event->payload['total'],
                'currency'    => $event->payload['currency'],
                'created_at'  => $event->occurredAt,
            ]);
    }
}
```

---

### ৭. CQRS in E-commerce — সম্পূর্ণ উদাহরণ

বাংলাদেশী e-commerce platform (Daraz/Chaldal-এর মতো) এর জন্য complete CQRS implementation:

#### Commands, Events, Queries সংজ্ঞা

```
Commands:                    Events:                      Queries:
├── PlaceOrder              ├── OrderPlaced               ├── GetOrderDetails
├── CancelOrder             ├── OrderCancelled            ├── SearchProducts
├── UpdateInventory         ├── InventoryUpdated          ├── GetUserDashboard
├── ProcessPayment          ├── PaymentProcessed          ├── GetOrderHistory
├── ApplyDiscount           ├── PaymentFailed             ├── GetInventoryReport
└── ShipOrder               ├── OrderShipped              └── GetSalesAnalytics
                            └── DiscountApplied
```

#### PHP Implementation

```php
<?php

declare(strict_types=1);

// ========== Commands ==========
final readonly class PlaceOrderCommand
{
    public function __construct(
        public string $customerId,
        public array $items,       // [['productId' => ..., 'qty' => ..., 'price' => ...]]
        public array $shippingAddress,
        public string $paymentMethod,
        public ?string $couponCode = null,
    ) {}
}

final readonly class ProcessPaymentCommand
{
    public function __construct(
        public string $orderId,
        public float $amount,
        public string $paymentMethod,   // 'bkash', 'nagad', 'card', 'cod'
        public string $currency = 'BDT',
    ) {}
}

// ========== Command Handlers ==========
final readonly class PlaceOrderHandler
{
    public function __construct(
        private EventStore $eventStore,
        private InventoryService $inventory,
        private CouponService $coupons,
    ) {}

    public function handle(PlaceOrderCommand $cmd): string
    {
        // Stock check
        foreach ($cmd->items as $item) {
            if (!$this->inventory->hasStock($item['productId'], $item['qty'])) {
                throw new InsufficientStockException("Product {$item['productId']} out of stock");
            }
        }

        // Coupon validation
        $discount = 0;
        if ($cmd->couponCode) {
            $discount = $this->coupons->validate($cmd->couponCode, $cmd->items);
        }

        $orderId = Uuid::uuid4()->toString();
        $order = OrderAggregate::place($orderId, $cmd->customerId, $cmd->items);

        if ($discount > 0) {
            $order->applyDiscount($cmd->couponCode, $discount);
        }

        $events = $order->pullUncommittedEvents();
        $this->eventStore->append($orderId, $events, 0);

        return $orderId;
    }
}

final readonly class ProcessPaymentHandler
{
    public function __construct(
        private EventStore $eventStore,
        private PaymentGateway $gateway,
    ) {}

    public function handle(ProcessPaymentCommand $cmd): void
    {
        $events = $this->eventStore->getEvents($cmd->orderId);
        $order = OrderAggregate::fromEvents($events);

        try {
            $transactionId = $this->gateway->charge(
                amount: $cmd->amount,
                method: $cmd->paymentMethod,
                currency: $cmd->currency,
            );
            $order->markPaymentProcessed($transactionId);
        } catch (PaymentFailedException $e) {
            $order->markPaymentFailed($e->getMessage());
        }

        $newEvents = $order->pullUncommittedEvents();
        $this->eventStore->append($cmd->orderId, $newEvents, $order->version);
    }
}

// ========== Projections ==========
final class OrderSummaryProjection
{
    public function __construct(
        private Connection $readDb,
        private Client $elasticsearch,
    ) {}

    public function onOrderPlaced(DomainEvent $event): void
    {
        $data = [
            'order_id'      => $event->aggregateId,
            'customer_id'   => $event->payload['customer_id'],
            'items'         => json_encode($event->payload['items']),
            'total_amount'  => $event->payload['total_amount'],
            'status'        => 'placed',
            'placed_at'     => $event->occurredAt->format('Y-m-d H:i:s'),
        ];

        $this->readDb->table('order_summaries')->insert($data);
        $this->elasticsearch->index(['index' => 'orders', 'id' => $event->aggregateId, 'body' => $data]);
    }

    public function onPaymentProcessed(DomainEvent $event): void
    {
        $this->readDb->table('order_summaries')
            ->where('order_id', $event->aggregateId)
            ->update([
                'status'         => 'paid',
                'transaction_id' => $event->payload['transaction_id'],
                'paid_at'        => $event->occurredAt->format('Y-m-d H:i:s'),
            ]);
    }
}

final class InventoryProjection
{
    public function __construct(private Connection $readDb) {}

    public function onOrderPlaced(DomainEvent $event): void
    {
        foreach ($event->payload['items'] as $item) {
            $this->readDb->table('inventory_view')
                ->where('product_id', $item['productId'])
                ->decrement('available_qty', $item['qty']);

            $this->readDb->table('inventory_view')
                ->where('product_id', $item['productId'])
                ->increment('reserved_qty', $item['qty']);
        }
    }

    public function onOrderCancelled(DomainEvent $event): void
    {
        foreach ($event->payload['items'] as $item) {
            $this->readDb->table('inventory_view')
                ->where('product_id', $item['productId'])
                ->increment('available_qty', $item['qty']);

            $this->readDb->table('inventory_view')
                ->where('product_id', $item['productId'])
                ->decrement('reserved_qty', $item['qty']);
        }
    }
}

// ========== Query Handlers ==========
final readonly class GetUserDashboardHandler
{
    public function __construct(
        private Connection $readDb,
        private Redis $redis,
    ) {}

    public function handle(GetUserDashboardQuery $query): UserDashboardDTO
    {
        $cacheKey = "dashboard:{$query->userId}";
        $cached = $this->redis->get($cacheKey);
        if ($cached) return UserDashboardDTO::fromJson($cached);

        $recentOrders = $this->readDb->table('order_summaries')
            ->where('customer_id', $query->userId)
            ->orderByDesc('placed_at')
            ->limit(5)
            ->get();

        $stats = $this->readDb->table('order_summaries')
            ->where('customer_id', $query->userId)
            ->selectRaw('COUNT(*) as total_orders, SUM(total_amount) as total_spent')
            ->first();

        $dto = new UserDashboardDTO(
            userId: $query->userId,
            recentOrders: $recentOrders->toArray(),
            totalOrders: (int) $stats->total_orders,
            totalSpent: (float) $stats->total_spent,
        );

        $this->redis->setex($cacheKey, 300, $dto->toJson());

        return $dto;
    }
}
```

#### Node.js Implementation

```javascript
// ========== Commands ==========
class PlaceOrderCommand {
  constructor({ customerId, items, shippingAddress, paymentMethod, couponCode = null }) {
    Object.freeze(Object.assign(this, { customerId, items, shippingAddress, paymentMethod, couponCode }));
  }
}

// ========== Command Handler ==========
class PlaceOrderHandler {
  #eventStore;
  #inventoryService;

  constructor(eventStore, inventoryService) {
    this.#eventStore = eventStore;
    this.#inventoryService = inventoryService;
  }

  async handle(cmd) {
    for (const item of cmd.items) {
      const hasStock = await this.#inventoryService.check(item.productId, item.qty);
      if (!hasStock) throw new Error(`Product ${item.productId} out of stock`);
    }

    const orderId = crypto.randomUUID();
    const order = OrderAggregate.place(orderId, cmd.customerId, cmd.items);

    const events = order.pullUncommittedEvents();
    await this.#eventStore.append(orderId, events, 0);

    return orderId;
  }
}

// ========== Projections ==========
class OrderSummaryProjection {
  #readDb;
  #esClient;

  constructor(readDb, esClient) {
    this.#readDb = readDb;
    this.#esClient = esClient;
  }

  async project(event) {
    const handlers = {
      'order.placed': (e) => this.#onOrderPlaced(e),
      'payment.processed': (e) => this.#onPaymentProcessed(e),
      'order.cancelled': (e) => this.#onOrderCancelled(e),
      'order.shipped': (e) => this.#onOrderShipped(e),
    };
    await handlers[event.type]?.(event);
  }

  async #onOrderPlaced(event) {
    const data = {
      orderId: event.aggregateId,
      customerId: event.payload.customerId,
      items: event.payload.items,
      totalAmount: event.payload.totalAmount,
      status: 'placed',
      placedAt: event.occurredAt,
    };

    await this.#readDb.query(
      `INSERT INTO order_summaries (order_id, customer_id, items, total_amount, status, placed_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [data.orderId, data.customerId, JSON.stringify(data.items), data.totalAmount, data.status, data.placedAt]
    );

    await this.#esClient.index({ index: 'orders', id: data.orderId, document: data });
  }

  async #onPaymentProcessed(event) {
    await this.#readDb.query(
      `UPDATE order_summaries SET status = 'paid', transaction_id = $1, paid_at = $2 WHERE order_id = $3`,
      [event.payload.transactionId, event.occurredAt, event.aggregateId]
    );
  }

  async #onOrderCancelled(event) {
    await this.#readDb.query(
      `UPDATE order_summaries SET status = 'cancelled' WHERE order_id = $1`,
      [event.aggregateId]
    );
  }

  async #onOrderShipped(event) {
    await this.#readDb.query(
      `UPDATE order_summaries SET status = 'shipped', tracking_id = $1 WHERE order_id = $2`,
      [event.payload.trackingId, event.aggregateId]
    );
  }
}

class InventoryProjection {
  #readDb;

  constructor(readDb) {
    this.#readDb = readDb;
  }

  async project(event) {
    if (event.type === 'order.placed') {
      for (const item of event.payload.items) {
        await this.#readDb.query(
          `UPDATE inventory_view
           SET available_qty = available_qty - $1, reserved_qty = reserved_qty + $1
           WHERE product_id = $2`,
          [item.qty, item.productId]
        );
      }
    }

    if (event.type === 'order.cancelled') {
      for (const item of event.payload.items) {
        await this.#readDb.query(
          `UPDATE inventory_view
           SET available_qty = available_qty + $1, reserved_qty = reserved_qty - $1
           WHERE product_id = $2`,
          [item.qty, item.productId]
        );
      }
    }
  }
}

// ========== Query Handler ==========
class GetUserDashboardHandler {
  #readDb;
  #redis;

  constructor(readDb, redis) {
    this.#readDb = readDb;
    this.#redis = redis;
  }

  async handle({ userId }) {
    const cached = await this.#redis.get(`dashboard:${userId}`);
    if (cached) return JSON.parse(cached);

    const [ordersResult, statsResult] = await Promise.all([
      this.#readDb.query(
        `SELECT * FROM order_summaries WHERE customer_id = $1 ORDER BY placed_at DESC LIMIT 5`,
        [userId]
      ),
      this.#readDb.query(
        `SELECT COUNT(*) as total_orders, COALESCE(SUM(total_amount), 0) as total_spent
         FROM order_summaries WHERE customer_id = $1`,
        [userId]
      ),
    ]);

    const dashboard = {
      userId,
      recentOrders: ordersResult.rows,
      totalOrders: parseInt(statsResult.rows[0].total_orders),
      totalSpent: parseFloat(statsResult.rows[0].total_spent),
    };

    await this.#redis.setex(`dashboard:${userId}`, 300, JSON.stringify(dashboard));
    return dashboard;
  }
}
```

---

## 🗄️ Database Strategies

### Write-optimized vs Read-optimized

| বৈশিষ্ট্য | Write DB | Read DB |
|-----------|----------|---------|
| **Structure** | Normalized (3NF) | Denormalized |
| **Optimization** | ACID transactions, constraints | Fast reads, indexing |
| **প্রযুক্তি** | PostgreSQL, MySQL | Elasticsearch, Redis, MongoDB |
| **Scaling** | Vertical scaling সাধারণত | Horizontal scaling সহজ |
| **Schema** | Strict, validated | Flexible, query-driven |

### Database নির্বাচন গাইড

| Use Case | Write DB | Read DB | কারণ |
|----------|----------|---------|------|
| Product Search | PostgreSQL | Elasticsearch | Full-text search, faceting |
| User Session/Cart | PostgreSQL | Redis | Ultra-low latency |
| Order History | PostgreSQL | MongoDB | Nested documents, flexible schema |
| Analytics Dashboard | PostgreSQL | ClickHouse/TimescaleDB | Time-series aggregation |
| Recommendations | PostgreSQL | Redis (Sorted Sets) | Fast scoring ও ranking |

### Change Data Capture (CDC) with Debezium

```
PostgreSQL (Write DB)
       │
       │  WAL (Write-Ahead Log)
       ▼
   Debezium Connector
       │
       │  Change Events
       ▼
   Apache Kafka
       │
   ┌───┼───────────────┐
   ▼   ▼               ▼
 Elastic  Redis     MongoDB
 (Search) (Cache)   (Catalog)
```

CDC ব্যবহার করলে application code-এ event publish করতে হয় না — database-এর WAL থেকে সরাসরি changes capture হয়। এটি dual-write problem (write DB তে save হলো কিন্তু event publish fail হলো) সমাধান করে।

---

## ✅ সুবিধা (Pros) ও ❌ অসুবিধা (Cons)

| ✅ সুবিধা | ❌ অসুবিধা |
|----------|-----------|
| Read ও Write আলাদাভাবে scale করা যায় | System complexity উল্লেখযোগ্যভাবে বাড়ে |
| প্রতিটি side আলাদাভাবে optimize করা যায় | Eventual consistency handle করতে হয় |
| Read model বিভিন্ন format/DB-তে রাখা যায় | বেশি infrastructure দরকার (multiple DBs, message broker) |
| Complex domain-এর business logic পরিষ্কার থাকে | Team-এর learning curve বেশি |
| Event Sourcing-এর সাথে মিলিয়ে পূর্ণ audit trail পাওয়া যায় | Data synchronization failure handle করতে হয় |
| Separate team গুলো আলাদাভাবে কাজ করতে পারে | Debugging কঠিন — data multiple জায়গায় থাকে |
| Read-heavy workload-এ dramatically better performance | Simple CRUD application-এ over-engineering |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Simple CRUD-এ CQRS ব্যবহার

❌ **ভুল:** প্রতিটি application-এ CQRS ব্যবহার করা।

```
// এই ধরনের simple entity-তে CQRS লাগবে না:
User Profile → CRUD is enough
Blog Post → simple read/write, CQRS overkill
Settings Page → single table, no complex reads
```

✅ **সমাধান:** শুধুমাত্র complex domain (high read/write ratio difference, complex business rules, different scaling needs) এ CQRS ব্যবহার করুন।

### ২. Eventual Consistency Ignore করা

❌ **ভুল:** User order place করার সাথে সাথে order list-এ দেখতে চাচ্ছে, কিন্তু read model এখনো update হয়নি।

✅ **সমাধান:** Optimistic UI updates, polling, বা WebSocket-based notification ব্যবহার করুন (উপরের section ৫ দেখুন)।

### ৩. Read-Write Model Share করা

❌ **ভুল:** একই Entity class read ও write দুটোতেই ব্যবহার করা — এতে CQRS-এর মূল উদ্দেশ্যই নষ্ট হয়।

✅ **সমাধান:** Write-এ rich domain model, Read-এ lightweight DTO — দুটো সম্পূর্ণ আলাদা class।

### ৪. Command Handler-এ Idempotency না রাখা

❌ **ভুল:** Network retry-তে একই command দুবার execute হয়ে duplicate order create হচ্ছে।

✅ **সমাধান:**

```php
final readonly class IdempotentCommandMiddleware implements CommandMiddleware
{
    public function __construct(private Connection $db) {}

    public function handle(object $command, Closure $next): mixed
    {
        if (!property_exists($command, 'idempotencyKey')) {
            return $next($command);
        }

        // আগে execute হয়েছে কিনা check
        $existing = $this->db->table('processed_commands')
            ->where('idempotency_key', $command->idempotencyKey)
            ->first();

        if ($existing) {
            return json_decode($existing->result, true);
        }

        $result = $next($command);

        $this->db->table('processed_commands')->insert([
            'idempotency_key' => $command->idempotencyKey,
            'command_type'    => $command::class,
            'result'          => json_encode($result),
            'processed_at'    => now(),
        ]);

        return $result;
    }
}
```

### ৫. Projection Failure Handle না করা

❌ **ভুল:** Event projection fail হলে read model inconsistent থেকে যাচ্ছে, কোনো alerting নেই।

✅ **সমাধান:** Dead letter queue, retry mechanism, ও projection health monitoring ব্যবহার করুন। Projection rebuild capability সবসময় রাখুন।

---

## 🧪 টেস্টিং CQRS

### Command Handler Unit Test (PHP - PHPUnit)

```php
<?php

final class PlaceOrderHandlerTest extends TestCase
{
    public function test_places_order_successfully(): void
    {
        $eventStore = $this->createMock(EventStore::class);
        $inventoryService = $this->createMock(InventoryService::class);

        $inventoryService->method('hasStock')->willReturn(true);

        $eventStore->expects($this->once())
            ->method('append')
            ->with(
                $this->isType('string'),
                $this->callback(fn (array $events) =>
                    $events[0]->type === OrderEventType::Placed
                    && $events[0]->payload['customer_id'] === 'cust-1'
                ),
                $this->equalTo(0),
            );

        $handler = new PlaceOrderHandler($eventStore, $inventoryService);

        $orderId = $handler->handle(new PlaceOrderCommand(
            customerId: 'cust-1',
            items: [['productId' => 'p-1', 'qty' => 2, 'price' => 500]],
            shippingAddress: ['city' => 'Dhaka'],
            paymentMethod: 'bkash',
        ));

        $this->assertNotEmpty($orderId);
    }

    public function test_throws_when_insufficient_stock(): void
    {
        $inventoryService = $this->createMock(InventoryService::class);
        $inventoryService->method('hasStock')->willReturn(false);

        $handler = new PlaceOrderHandler(
            $this->createMock(EventStore::class),
            $inventoryService,
        );

        $this->expectException(InsufficientStockException::class);

        $handler->handle(new PlaceOrderCommand(
            customerId: 'cust-1',
            items: [['productId' => 'p-1', 'qty' => 100, 'price' => 500]],
            shippingAddress: ['city' => 'Dhaka'],
            paymentMethod: 'bkash',
        ));
    }
}
```

### Command Handler Unit Test (JS - Jest)

```javascript
describe('PlaceOrderHandler', () => {
  let eventStore;
  let inventoryService;
  let handler;

  beforeEach(() => {
    eventStore = { append: jest.fn(), getEvents: jest.fn() };
    inventoryService = { check: jest.fn() };
    handler = new PlaceOrderHandler(eventStore, inventoryService);
  });

  it('should place order when stock available', async () => {
    inventoryService.check.mockResolvedValue(true);

    const orderId = await handler.handle(new PlaceOrderCommand({
      customerId: 'cust-1',
      items: [{ productId: 'p-1', qty: 2, unitPrice: 500 }],
      shippingAddress: { city: 'Dhaka' },
      paymentMethod: 'bkash',
    }));

    expect(orderId).toBeDefined();
    expect(eventStore.append).toHaveBeenCalledWith(
      expect.any(String),
      expect.arrayContaining([
        expect.objectContaining({ type: 'order.placed' }),
      ]),
      0,
    );
  });

  it('should throw when stock insufficient', async () => {
    inventoryService.check.mockResolvedValue(false);

    await expect(handler.handle(new PlaceOrderCommand({
      customerId: 'cust-1',
      items: [{ productId: 'p-1', qty: 999, unitPrice: 500 }],
      shippingAddress: { city: 'Dhaka' },
      paymentMethod: 'bkash',
    }))).rejects.toThrow('out of stock');
  });
});
```

### Projection Test

```javascript
describe('OrderSummaryProjection', () => {
  let projection;
  let readDb;

  beforeEach(() => {
    readDb = { query: jest.fn().mockResolvedValue({ rows: [] }) };
    projection = new OrderSummaryProjection(readDb, { index: jest.fn() });
  });

  it('should insert order summary on OrderPlaced', async () => {
    await projection.project({
      type: 'order.placed',
      aggregateId: 'order-1',
      payload: {
        customerId: 'cust-1',
        items: [{ productId: 'p-1', qty: 2, unitPrice: 500 }],
        totalAmount: 1000,
      },
      occurredAt: '2024-01-15T10:00:00Z',
    });

    expect(readDb.query).toHaveBeenCalledWith(
      expect.stringContaining('INSERT INTO order_summaries'),
      expect.arrayContaining(['order-1', 'cust-1']),
    );
  });

  it('should update status on PaymentProcessed', async () => {
    await projection.project({
      type: 'payment.processed',
      aggregateId: 'order-1',
      payload: { transactionId: 'txn-123' },
      occurredAt: '2024-01-15T10:05:00Z',
    });

    expect(readDb.query).toHaveBeenCalledWith(
      expect.stringContaining("status = 'paid'"),
      expect.arrayContaining(['txn-123']),
    );
  });
});
```

### Eventual Consistency Integration Test

```javascript
describe('Eventual Consistency', () => {
  it('should sync read model within acceptable delay', async () => {
    // Command execute
    const orderId = await commandBus.dispatch(new PlaceOrderCommand({
      customerId: 'cust-1',
      items: [{ productId: 'p-1', qty: 1, unitPrice: 500 }],
      shippingAddress: { city: 'Dhaka' },
      paymentMethod: 'cod',
    }));

    // Read model-এ এখনই নাও থাকতে পারে
    // Polling with timeout
    const maxWait = 5000;
    const startTime = Date.now();
    let order = null;

    while (Date.now() - startTime < maxWait) {
      try {
        order = await queryBus.ask(new GetOrderDetailsQuery(orderId));
        if (order) break;
      } catch { /* not yet available */ }
      await new Promise((r) => setTimeout(r, 200));
    }

    expect(order).not.toBeNull();
    expect(order.orderId).toBe(orderId);
    expect(order.status).toBe('placed');
  });
});
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করুন যখন:

1. **Read/Write ratio অনেক আলাদা** — যেমন e-commerce-এ product search (read) অনেক বেশি, order (write) কম
2. **Read ও Write-এর optimization requirement আলাদা** — write-এ consistency, read-এ speed
3. **Complex domain logic** — rich business rules যা একটি simple CRUD model-এ ধরানো কঠিন
4. **আলাদাভাবে scale করতে হবে** — read replica বা read-optimized store দরকার
5. **Multiple read representations দরকার** — একই data search, dashboard, reporting-এ আলাদা format-এ দরকার
6. **Event Sourcing ব্যবহার করতে চান** — CQRS ছাড়া Event Sourcing প্রায় অসম্ভব
7. **Microservices architecture** — service গুলোর নিজস্ব read model দরকার

### ❌ ব্যবহার করবেন না যখন:

1. **Simple CRUD application** — blog, user profile, settings page
2. **Team experience কম** — CQRS এর learning curve ও debugging complexity high
3. **Strong consistency আবশ্যক** — যেখানে eventual consistency গ্রহণযোগ্য নয়
4. **Low traffic application** — CQRS-এর infrastructure cost justify হয় না
5. **Tight deadline** — CQRS implement করতে সময় বেশি লাগে

### Team Readiness Checklist

```
☐ Team distributed systems concepts বোঝে
☐ Eventual consistency নিয়ে experience আছে
☐ Message broker (Kafka/RabbitMQ) operate করতে পারে
☐ Multiple database manage করার capability আছে
☐ Domain-Driven Design (DDD) practice করে
☐ Proper monitoring ও observability setup আছে
☐ CI/CD pipeline mature
☐ Business stakeholders eventual consistency accept করে
```

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| Pattern | সম্পর্ক |
|---------|---------|
| **Event Sourcing** | CQRS-এর সবচেয়ে natural companion — Event Store write model, Projections read model |
| **Domain-Driven Design (DDD)** | CQRS DDD-এর Bounded Context concept-এর সাথে চমৎকারভাবে কাজ করে |
| **Mediator Pattern** | Command Bus ও Query Bus হলো Mediator pattern-এর implementation |
| **Repository Pattern** | Write side-এ Aggregate Repository, Read side-এ Query-specific Repository |
| **Saga / Process Manager** | Long-running business processes যা multiple commands coordinate করে |
| **Event-Driven Architecture** | CQRS events publish করে, অন্যান্য service subscribe করে |
| **API Gateway** | Command ও Query আলাদা endpoint-এ route করতে Gateway ব্যবহার |
| **Strangler Fig Pattern** | Legacy monolith থেকে ধীরে ধীরে CQRS-এ migrate করতে ব্যবহার |

---

## 📋 সারসংক্ষেপ

```
CQRS মনে রাখার সূত্র:

C - Command: State পরিবর্তন করে, data return করে না
Q - Query:   State পড়ে, কোনো পরিবর্তন করে না
R - Responsibility: প্রতিটির দায়িত্ব আলাদা
S - Segregation: সম্পূর্ণ পৃথক model, পৃথক pathway

তিনটি Level:
  Level 1: একই DB, আলাদা model (সবচেয়ে সহজ, কম risk)
  Level 2: আলাদা DB, event-based sync (মাঝারি জটিলতা)
  Level 3: Event Sourcing + CQRS (সবচেয়ে শক্তিশালী, সবচেয়ে জটিল)

মূল প্রশ্ন করুন নিজেকে:
  "আমার read ও write requirement কি সত্যিই এতটা আলাদা যে
   আলাদা model/infrastructure justify করে?"

উত্তর "হ্যাঁ" হলে → CQRS বিবেচনা করুন
উত্তর "না" বা "জানি না" হলে → Simple architecture দিয়ে শুরু করুন
```

> **মনে রাখবেন:** CQRS একটি powerful pattern, কিন্তু এটি **accidental complexity** যোগ করে। শুধুমাত্র তখনই ব্যবহার করুন যখন **essential complexity** এটি demand করে। বাংলাদেশের context-এ, bKash, Pathao, বা Chaldal-এর মতো high-traffic, complex-domain platform-এ CQRS চমৎকার কাজ করে — কিন্তু একটি ছোট e-commerce site-এ এটি over-engineering হবে।
