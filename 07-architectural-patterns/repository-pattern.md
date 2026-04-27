# 🗃️ রিপোজিটরি প্যাটার্ন (Repository Pattern)

## 📌 সংজ্ঞা ও মূল ধারণা

**Martin Fowler** তাঁর বিখ্যাত বই *"Patterns of Enterprise Application Architecture"* (PoEAA)-তে Repository Pattern-কে এভাবে সংজ্ঞায়িত করেছেন:

> "A Repository mediates between the domain and data mapping layers, acting like an in-memory domain object collection."

সোজা কথায়, Repository হলো আপনার **domain layer** আর **data access layer**-এর মধ্যে একটি **মধ্যস্থতাকারী** (mediator)। এটি domain objects-কে একটি **collection-like interface** দিয়ে expose করে, যেন মনে হয় আপনি একটি in-memory collection থেকে data নিচ্ছেন — আসলে পেছনে database query চলছে।

### মূল নীতিগুলো:

- **Abstraction**: Data access-এর বিস্তারিত বিষয় (SQL, ORM queries) domain layer থেকে লুকিয়ে রাখে
- **Collection Semantics**: `add()`, `remove()`, `findById()` — যেন একটি collection ব্যবহার করছেন
- **Domain-Centric**: Repository **domain entity** কেন্দ্রিক, database table কেন্দ্রিক না
- **Persistence Ignorance**: Domain layer জানে না data কোথায় সংরক্ষিত — MySQL, MongoDB, নাকি API

### Repository vs Active Record vs Data Mapper vs DAO

| বৈশিষ্ট্য | Repository | Active Record | Data Mapper | DAO |
|---|---|---|---|---|
| **দায়িত্ব** | Domain object collection | Object-এ CRUD built-in | Object-DB mapping | Raw data access |
| **Domain Coupling** | Loose | Tight | Medium | None |
| **উদাহরণ** | Custom Repository class | Laravel Eloquent Model | Doctrine ORM | JDBC DAO |
| **Complexity** | Medium-High | Low | High | Low-Medium |
| **Testing** | সহজ (mockable) | কঠিন | মাঝামাঝি | মাঝামাঝি |
| **Best For** | Complex domain logic | Simple CRUD apps | Complex mapping | Legacy systems |

**Active Record**-এ model নিজেই database-এর সাথে কথা বলে (`$user->save()`)। **Data Mapper**-এ একটি আলাদা mapper class entity আর database-এর মধ্যে mapping করে। **DAO** (Data Access Object) সরাসরি data access-এর abstraction দেয়, কিন্তু domain-aware না। **Repository** সবচেয়ে domain-centric — এটি domain-এর ভাষায় কথা বলে।

---

## 🏠 বাস্তব জীবনের উদাহরণ

কল্পনা করুন আপনি **ঢাকা বিশ্ববিদ্যালয়ের কেন্দ্রীয় লাইব্রেরিতে** গেছেন একটি বই খুঁজতে।

**Repository ছাড়া (সরাসরি Data Access):**
আপনি নিজে প্রতিটি shelf-এ গিয়ে, Dewey Decimal System বুঝে, ক্যাটালগ কার্ড দেখে বই খুঁজছেন। আপনাকে লাইব্রেরির **internal organization** জানতে হচ্ছে।

**Repository দিয়ে:**
আপনি librarian-কে বললেন: "আমাকে Design Patterns বইটা দিন।" Librarian জানে বইটা কোন shelf-এ, কোন section-এ। আপনার কোনো internal detail জানার দরকার নেই।

এখানে:
- **আপনি** = Service / Controller (domain logic)
- **Librarian** = Repository
- **Shelf/Catalog System** = Database / ORM
- **বই** = Domain Entity

Librarian বদলালে (নতুন librarian এলে) আপনার বই চাওয়ার পদ্ধতি বদলায় না। ঠিক তেমনি, database বদলালে (MySQL → MongoDB) Repository-র interface একই থাকে।

---

## 📊 আর্কিটেকচার ডায়াগ্রাম

### মূল Flow:

```
┌─────────────┐    ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐
│  Controller  │───▶│   Service   │───▶│   Repository     │───▶│   Database   │
│  (HTTP)      │    │  (Business  │    │   (Interface)    │    │  (MySQL/     │
│              │◀───│   Logic)    │◀───│                  │◀───│   MongoDB)   │
└─────────────┘    └─────────────┘    └──────────────────┘    └──────────────┘
                                              │
                                     ┌────────┴────────┐
                                     │  Abstraction     │
                                     │  Layer           │
                                     └────────┬────────┘
                          ┌──────────────┬─────┴──────┬───────────────┐
                          ▼              ▼            ▼               ▼
                   ┌───────────┐  ┌──────────┐ ┌──────────┐  ┌────────────┐
                   │ Eloquent  │  │ Doctrine │ │ MongoDB  │  │ In-Memory  │
                   │ Repo      │  │ Repo     │ │ Repo     │  │ Repo       │
                   └───────────┘  └──────────┘ └──────────┘  └────────────┘
```

### Repository Abstraction Layer:

```
   Domain Layer                Repository Layer              Infrastructure
  ┌─────────────┐         ┌─────────────────────┐        ┌─────────────────┐
  │             │         │  <<interface>>       │        │                 │
  │  Service    │────────▶│  UserRepository      │◁───────│ EloquentUser    │
  │  Layer      │         │                     │        │ Repository      │
  │             │         │  + findById(id)     │        │                 │
  │  (Depends   │         │  + findByEmail(e)   │        ├─────────────────┤
  │   on        │         │  + save(user)       │        │ MongoUser       │
  │   Interface)│         │  + delete(user)     │        │ Repository      │
  │             │         │  + findActive()     │        │                 │
  └─────────────┘         └─────────────────────┘        ├─────────────────┤
                                                         │ InMemoryUser    │
                                                         │ Repository      │
                                                         │ (Testing)       │
                                                         └─────────────────┘
```

---

## 💻 Basic Repository Implementation

### PHP (Laravel 11 + PHP 8.3)

#### Interface সংজ্ঞা:

```php
<?php
// app/Repositories/Contracts/UserRepositoryInterface.php

namespace App\Repositories\Contracts;

use App\Models\User;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

interface UserRepositoryInterface
{
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function findActive(): Collection;
    public function save(User $user): User;
    public function delete(User $user): bool;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
    public function findByPhoneNumber(string $phone): ?User;
}
```

#### Eloquent Implementation:

```php
<?php
// app/Repositories/Eloquent/EloquentUserRepository.php

namespace App\Repositories\Eloquent;

use App\Models\User;
use App\Repositories\Contracts\UserRepositoryInterface;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

final readonly class EloquentUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private User $model,
    ) {}

    public function findById(int $id): ?User
    {
        return $this->model->find($id);
    }

    public function findByEmail(string $email): ?User
    {
        return $this->model
            ->where('email', $email)
            ->first();
    }

    public function findActive(): Collection
    {
        return $this->model
            ->where('is_active', true)
            ->orderBy('created_at', 'desc')
            ->get();
    }

    public function save(User $user): User
    {
        $user->save();
        return $user->fresh();
    }

    public function delete(User $user): bool
    {
        return $user->delete();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->model
            ->latest()
            ->paginate($perPage);
    }

    public function findByPhoneNumber(string $phone): ?User
    {
        return $this->model
            ->where('phone', $phone)
            ->first();
    }
}
```

#### Service Provider Binding:

```php
<?php
// app/Providers/RepositoryServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Repositories\Contracts\UserRepositoryInterface;
use App\Repositories\Eloquent\EloquentUserRepository;

final class RepositoryServiceProvider extends ServiceProvider
{
    public array $bindings = [
        UserRepositoryInterface::class => EloquentUserRepository::class,
    ];
}
```

#### Controller ব্যবহার:

```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use App\Repositories\Contracts\UserRepositoryInterface;
use Illuminate\Http\JsonResponse;

final readonly class UserController extends Controller
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {}

    public function show(int $id): JsonResponse
    {
        $user = $this->userRepository->findById($id);

        if (!$user) {
            return response()->json(['message' => 'User not found'], 404);
        }

        return response()->json($user);
    }

    public function index(): JsonResponse
    {
        $users = $this->userRepository->paginate(20);
        return response()->json($users);
    }
}
```

### JavaScript (Node.js + ES2022+)

#### Interface / Abstract Class:

```javascript
// src/repositories/contracts/UserRepository.js

export class UserRepository {
    async findById(id) {
        throw new Error('Method findById() must be implemented');
    }

    async findByEmail(email) {
        throw new Error('Method findByEmail() must be implemented');
    }

    async findActive() {
        throw new Error('Method findActive() must be implemented');
    }

    async save(userData) {
        throw new Error('Method save() must be implemented');
    }

    async delete(id) {
        throw new Error('Method delete() must be implemented');
    }

    async paginate(page = 1, perPage = 15) {
        throw new Error('Method paginate() must be implemented');
    }

    async findByPhoneNumber(phone) {
        throw new Error('Method findByPhoneNumber() must be implemented');
    }
}
```

#### Mongoose Implementation:

```javascript
// src/repositories/mongoose/MongooseUserRepository.js

import { UserRepository } from '../contracts/UserRepository.js';
import { UserModel } from '../../models/User.js';

export class MongooseUserRepository extends UserRepository {
    #model;

    constructor(model = UserModel) {
        super();
        this.#model = model;
    }

    async findById(id) {
        return this.#model.findById(id).lean();
    }

    async findByEmail(email) {
        return this.#model.findOne({ email }).lean();
    }

    async findActive() {
        return this.#model
            .find({ isActive: true })
            .sort({ createdAt: -1 })
            .lean();
    }

    async save(userData) {
        if (userData._id) {
            return this.#model
                .findByIdAndUpdate(userData._id, userData, { new: true })
                .lean();
        }
        const user = new this.#model(userData);
        const saved = await user.save();
        return saved.toObject();
    }

    async delete(id) {
        const result = await this.#model.findByIdAndDelete(id);
        return !!result;
    }

    async paginate(page = 1, perPage = 15) {
        const skip = (page - 1) * perPage;
        const [data, total] = await Promise.all([
            this.#model.find().sort({ createdAt: -1 }).skip(skip).limit(perPage).lean(),
            this.#model.countDocuments(),
        ]);

        return {
            data,
            total,
            page,
            perPage,
            totalPages: Math.ceil(total / perPage),
        };
    }

    async findByPhoneNumber(phone) {
        return this.#model.findOne({ phone }).lean();
    }
}
```

#### Sequelize Implementation:

```javascript
// src/repositories/sequelize/SequelizeUserRepository.js

import { UserRepository } from '../contracts/UserRepository.js';
import { User } from '../../models/sequelize/User.js';

export class SequelizeUserRepository extends UserRepository {
    #model;

    constructor(model = User) {
        super();
        this.#model = model;
    }

    async findById(id) {
        return this.#model.findByPk(id, { raw: true });
    }

    async findByEmail(email) {
        return this.#model.findOne({ where: { email }, raw: true });
    }

    async findActive() {
        return this.#model.findAll({
            where: { isActive: true },
            order: [['createdAt', 'DESC']],
            raw: true,
        });
    }

    async save(userData) {
        if (userData.id) {
            await this.#model.update(userData, { where: { id: userData.id } });
            return this.findById(userData.id);
        }
        const user = await this.#model.create(userData);
        return user.get({ plain: true });
    }

    async delete(id) {
        const deleted = await this.#model.destroy({ where: { id } });
        return deleted > 0;
    }

    async paginate(page = 1, perPage = 15) {
        const { rows: data, count: total } = await this.#model.findAndCountAll({
            offset: (page - 1) * perPage,
            limit: perPage,
            order: [['createdAt', 'DESC']],
            raw: true,
        });

        return { data, total, page, perPage, totalPages: Math.ceil(total / perPage) };
    }

    async findByPhoneNumber(phone) {
        return this.#model.findOne({ where: { phone }, raw: true });
    }
}
```

#### Dependency Injection Setup (Node.js):

```javascript
// src/container.js — Simple DI Container

class Container {
    #bindings = new Map();
    #singletons = new Map();

    bind(abstract, concrete) {
        this.#bindings.set(abstract, { concrete, shared: false });
    }

    singleton(abstract, concrete) {
        this.#bindings.set(abstract, { concrete, shared: true });
    }

    resolve(abstract) {
        const binding = this.#bindings.get(abstract);
        if (!binding) throw new Error(`No binding for: ${abstract}`);

        if (binding.shared && this.#singletons.has(abstract)) {
            return this.#singletons.get(abstract);
        }

        const instance = typeof binding.concrete === 'function'
            ? new binding.concrete()
            : binding.concrete;

        if (binding.shared) this.#singletons.set(abstract, instance);
        return instance;
    }
}

export const container = new Container();

// Registration
import { MongooseUserRepository } from './repositories/mongoose/MongooseUserRepository.js';

container.singleton('UserRepository', MongooseUserRepository);

// Usage in controller
// const userRepo = container.resolve('UserRepository');
```

---

## 🔥 Complex Scenarios

### ১. Generic / Base Repository

যখন প্রতিটি entity-র জন্য একই CRUD code বারবার লিখতে হয়, তখন **Generic Base Repository** তৈরি করা বুদ্ধিমানের কাজ। এটি **DRY principle** মেনে চলে।

#### PHP:

```php
<?php
// app/Repositories/Contracts/BaseRepositoryInterface.php

namespace App\Repositories\Contracts;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

/**
 * @template T of Model
 */
interface BaseRepositoryInterface
{
    public function findById(int $id): ?Model;
    public function findAll(): Collection;
    public function create(array $attributes): Model;
    public function update(int $id, array $attributes): ?Model;
    public function delete(int $id): bool;
    public function paginate(int $perPage = 15): LengthAwarePaginator;
    public function findWhere(array $conditions): Collection;
}
```

```php
<?php
// app/Repositories/Eloquent/BaseRepository.php

namespace App\Repositories\Eloquent;

use App\Repositories\Contracts\BaseRepositoryInterface;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

/**
 * @template T of Model
 * @implements BaseRepositoryInterface<T>
 */
abstract class BaseRepository implements BaseRepositoryInterface
{
    public function __construct(
        protected readonly Model $model,
    ) {}

    public function findById(int $id): ?Model
    {
        return $this->model->find($id);
    }

    public function findAll(): Collection
    {
        return $this->model->all();
    }

    public function create(array $attributes): Model
    {
        return $this->model->create($attributes);
    }

    public function update(int $id, array $attributes): ?Model
    {
        $entity = $this->findById($id);
        if (!$entity) return null;

        $entity->update($attributes);
        return $entity->fresh();
    }

    public function delete(int $id): bool
    {
        $entity = $this->findById($id);
        return $entity?->delete() ?? false;
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->model->latest()->paginate($perPage);
    }

    public function findWhere(array $conditions): Collection
    {
        return $this->model->where($conditions)->get();
    }
}
```

```php
<?php
// app/Repositories/Eloquent/EloquentProductRepository.php
// bKash-like fintech app-এ product/offer management

namespace App\Repositories\Eloquent;

use App\Models\Product;
use App\Repositories\Contracts\ProductRepositoryInterface;

final class EloquentProductRepository extends BaseRepository implements ProductRepositoryInterface
{
    public function __construct(Product $model)
    {
        parent::__construct($model);
    }

    public function findByCategory(string $category): \Illuminate\Support\Collection
    {
        return $this->model
            ->where('category', $category)
            ->where('is_active', true)
            ->orderBy('sort_order')
            ->get();
    }

    public function findInPriceRange(float $min, float $max): \Illuminate\Support\Collection
    {
        return $this->model
            ->whereBetween('price', [$min, $max])
            ->where('is_active', true)
            ->get();
    }
}
```

#### JavaScript:

```javascript
// src/repositories/BaseRepository.js

export class BaseRepository {
    _model;

    constructor(model) {
        if (new.target === BaseRepository) {
            throw new Error('BaseRepository is abstract');
        }
        this._model = model;
    }

    async findById(id) {
        return this._model.findById(id).lean();
    }

    async findAll() {
        return this._model.find().lean();
    }

    async create(attributes) {
        const entity = new this._model(attributes);
        const saved = await entity.save();
        return saved.toObject();
    }

    async update(id, attributes) {
        return this._model
            .findByIdAndUpdate(id, { $set: attributes }, { new: true })
            .lean();
    }

    async delete(id) {
        const result = await this._model.findByIdAndDelete(id);
        return !!result;
    }

    async paginate(page = 1, perPage = 15) {
        const skip = (page - 1) * perPage;
        const [data, total] = await Promise.all([
            this._model.find().sort({ createdAt: -1 }).skip(skip).limit(perPage).lean(),
            this._model.countDocuments(),
        ]);
        return { data, total, page, perPage, totalPages: Math.ceil(total / perPage) };
    }

    async findWhere(conditions) {
        return this._model.find(conditions).lean();
    }
}
```

```javascript
// src/repositories/mongoose/ProductRepository.js

import { BaseRepository } from '../BaseRepository.js';
import { ProductModel } from '../../models/Product.js';

export class MongooseProductRepository extends BaseRepository {
    constructor() {
        super(ProductModel);
    }

    async findByCategory(category) {
        return this._model
            .find({ category, isActive: true })
            .sort({ sortOrder: 1 })
            .lean();
    }

    async findInPriceRange(min, max) {
        return this._model
            .find({ price: { $gte: min, $lte: max }, isActive: true })
            .lean();
    }
}
```

---

### ২. Specification Pattern with Repository

**Specification Pattern** repository-র সাথে মিলিয়ে ব্যবহার করলে **জটিল query logic**-কে ছোট ছোট, reusable, composable specification-এ ভাঙা যায়। ধরুন, একটি e-commerce platform-এ product search — দাম, category, availability, rating সব মিলিয়ে filter করতে হবে।

#### PHP:

```php
<?php
// app/Specifications/Specification.php

namespace App\Specifications;

use Illuminate\Database\Eloquent\Builder;

interface Specification
{
    public function apply(Builder $query): Builder;
}
```

```php
<?php
// app/Specifications/AndSpecification.php

namespace App\Specifications;

use Illuminate\Database\Eloquent\Builder;

final readonly class AndSpecification implements Specification
{
    /** @param Specification[] $specifications */
    public function __construct(
        private array $specifications,
    ) {}

    public function apply(Builder $query): Builder
    {
        foreach ($this->specifications as $spec) {
            $query = $spec->apply($query);
        }
        return $query;
    }
}
```

```php
<?php
// app/Specifications/Product/PriceRangeSpecification.php

namespace App\Specifications\Product;

use App\Specifications\Specification;
use Illuminate\Database\Eloquent\Builder;

final readonly class PriceRangeSpecification implements Specification
{
    public function __construct(
        private float $minPrice,
        private float $maxPrice,
    ) {}

    public function apply(Builder $query): Builder
    {
        return $query->whereBetween('price', [$this->minPrice, $this->maxPrice]);
    }
}
```

```php
<?php
// app/Specifications/Product/CategorySpecification.php

namespace App\Specifications\Product;

use App\Specifications\Specification;
use Illuminate\Database\Eloquent\Builder;

final readonly class CategorySpecification implements Specification
{
    public function __construct(
        private string $category,
    ) {}

    public function apply(Builder $query): Builder
    {
        return $query->where('category', $this->category);
    }
}
```

```php
<?php
// app/Specifications/Product/InStockSpecification.php

namespace App\Specifications\Product;

use App\Specifications\Specification;
use Illuminate\Database\Eloquent\Builder;

final readonly class InStockSpecification implements Specification
{
    public function apply(Builder $query): Builder
    {
        return $query->where('stock_quantity', '>', 0);
    }
}
```

```php
<?php
// Repository-তে ব্যবহার

// app/Repositories/Contracts/ProductRepositoryInterface.php
namespace App\Repositories\Contracts;

use App\Specifications\Specification;
use Illuminate\Support\Collection;

interface ProductRepositoryInterface extends BaseRepositoryInterface
{
    public function match(Specification $specification): Collection;
}

// EloquentProductRepository-তে
public function match(Specification $specification): Collection
{
    $query = $this->model->newQuery();
    return $specification->apply($query)->get();
}

// Controller-এ ব্যবহার:
$spec = new AndSpecification([
    new CategorySpecification('electronics'),
    new PriceRangeSpecification(500, 50000),
    new InStockSpecification(),
]);

$products = $this->productRepository->match($spec);
```

#### JavaScript:

```javascript
// src/specifications/Specification.js

export class Specification {
    apply(query) {
        throw new Error('Must implement apply()');
    }

    and(...specs) {
        return new AndSpecification([this, ...specs]);
    }

    or(...specs) {
        return new OrSpecification([this, ...specs]);
    }

    not() {
        return new NotSpecification(this);
    }
}

export class AndSpecification extends Specification {
    #specs;
    constructor(specs) { super(); this.#specs = specs; }
    apply(query) {
        return this.#specs.reduce((q, spec) => spec.apply(q), query);
    }
}

export class OrSpecification extends Specification {
    #specs;
    constructor(specs) { super(); this.#specs = specs; }
    apply(query) {
        const conditions = this.#specs.map(spec => spec.toCondition());
        query.$or = conditions;
        return query;
    }
}

// Product-specific specifications
export class PriceRangeSpec extends Specification {
    #min; #max;
    constructor(min, max) { super(); this.#min = min; this.#max = max; }
    apply(query) {
        return { ...query, price: { $gte: this.#min, $lte: this.#max } };
    }
    toCondition() { return { price: { $gte: this.#min, $lte: this.#max } }; }
}

export class CategorySpec extends Specification {
    #category;
    constructor(category) { super(); this.#category = category; }
    apply(query) {
        return { ...query, category: this.#category };
    }
    toCondition() { return { category: this.#category }; }
}

export class InStockSpec extends Specification {
    apply(query) {
        return { ...query, stockQuantity: { $gt: 0 } };
    }
    toCondition() { return { stockQuantity: { $gt: 0 } }; }
}

// ব্যবহার:
const spec = new CategorySpec('electronics')
    .and(new PriceRangeSpec(500, 50000), new InStockSpec());

const products = await productRepo.match(spec);
```

---

### ৩. Repository with Caching (Decorator Pattern)

বাংলাদেশের একটি জনপ্রিয় e-commerce সাইটে লক্ষ লক্ষ ব্যবহারকারী একই product catalog দেখে। প্রতিবার database hit করা performance-এর জন্য ক্ষতিকর। **Cache Decorator** pattern দিয়ে repository-কে wrap করে caching যোগ করা যায় — মূল repository code পরিবর্তন না করেই।

#### PHP (Laravel Cache):

```php
<?php
// app/Repositories/Cache/CachedProductRepository.php

namespace App\Repositories\Cache;

use App\Repositories\Contracts\ProductRepositoryInterface;
use App\Specifications\Specification;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Cache;

final readonly class CachedProductRepository implements ProductRepositoryInterface
{
    private const int TTL_SECONDS = 3600;
    private const string CACHE_PREFIX = 'products';

    public function __construct(
        private ProductRepositoryInterface $innerRepository,
    ) {}

    public function findById(int $id): ?Model
    {
        return Cache::tags([self::CACHE_PREFIX])
            ->remember(
                self::CACHE_PREFIX . ":id:{$id}",
                self::TTL_SECONDS,
                fn () => $this->innerRepository->findById($id),
            );
    }

    public function findAll(): Collection
    {
        return Cache::tags([self::CACHE_PREFIX])
            ->remember(
                self::CACHE_PREFIX . ':all',
                self::TTL_SECONDS,
                fn () => $this->innerRepository->findAll(),
            );
    }

    public function create(array $attributes): Model
    {
        $result = $this->innerRepository->create($attributes);
        Cache::tags([self::CACHE_PREFIX])->flush();
        return $result;
    }

    public function update(int $id, array $attributes): ?Model
    {
        $result = $this->innerRepository->update($id, $attributes);
        Cache::tags([self::CACHE_PREFIX])->flush();
        return $result;
    }

    public function delete(int $id): bool
    {
        $result = $this->innerRepository->delete($id);
        Cache::tags([self::CACHE_PREFIX])->flush();
        return $result;
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->innerRepository->paginate($perPage);
    }

    public function findWhere(array $conditions): Collection
    {
        $key = self::CACHE_PREFIX . ':where:' . md5(serialize($conditions));
        return Cache::tags([self::CACHE_PREFIX])
            ->remember($key, self::TTL_SECONDS, fn () => $this->innerRepository->findWhere($conditions));
    }

    public function match(Specification $specification): Collection
    {
        $key = self::CACHE_PREFIX . ':spec:' . md5(serialize($specification));
        return Cache::tags([self::CACHE_PREFIX])
            ->remember($key, self::TTL_SECONDS, fn () => $this->innerRepository->match($specification));
    }

    // ... delegate other methods
}
```

```php
// Service Provider-এ binding:
$this->app->bind(ProductRepositoryInterface::class, function ($app) {
    return new CachedProductRepository(
        new EloquentProductRepository(new Product()),
    );
});
```

#### JavaScript (Redis Cache):

```javascript
// src/repositories/cache/CachedProductRepository.js

import Redis from 'ioredis';

export class CachedProductRepository {
    #innerRepo;
    #redis;
    #ttl;
    #prefix = 'products';

    constructor(innerRepo, redisClient = new Redis(), ttl = 3600) {
        this.#innerRepo = innerRepo;
        this.#redis = redisClient;
        this.#ttl = ttl;
    }

    async #getFromCache(key) {
        const cached = await this.#redis.get(`${this.#prefix}:${key}`);
        return cached ? JSON.parse(cached) : null;
    }

    async #setCache(key, data) {
        await this.#redis.setex(`${this.#prefix}:${key}`, this.#ttl, JSON.stringify(data));
    }

    async #invalidateAll() {
        const keys = await this.#redis.keys(`${this.#prefix}:*`);
        if (keys.length > 0) {
            await this.#redis.del(...keys);
        }
    }

    async findById(id) {
        const cacheKey = `id:${id}`;
        const cached = await this.#getFromCache(cacheKey);
        if (cached) return cached;

        const result = await this.#innerRepo.findById(id);
        if (result) await this.#setCache(cacheKey, result);
        return result;
    }

    async create(attributes) {
        const result = await this.#innerRepo.create(attributes);
        await this.#invalidateAll();
        return result;
    }

    async update(id, attributes) {
        const result = await this.#innerRepo.update(id, attributes);
        await this.#invalidateAll();
        return result;
    }

    async delete(id) {
        const result = await this.#innerRepo.delete(id);
        await this.#invalidateAll();
        return result;
    }

    async findAll() {
        const cacheKey = 'all';
        const cached = await this.#getFromCache(cacheKey);
        if (cached) return cached;

        const result = await this.#innerRepo.findAll();
        await this.#setCache(cacheKey, result);
        return result;
    }

    async paginate(page = 1, perPage = 15) {
        return this.#innerRepo.paginate(page, perPage);
    }
}
```

---

### ৪. Repository with Unit of Work

**Unit of Work** pattern একাধিক repository-র changes গুলোকে একটি **single transaction**-এ গ্রুপ করে। যদি কোনো একটি operation fail করে, সব কিছু **rollback** হবে। bKash-এর মতো fintech app-এ fund transfer-এ এটি অত্যন্ত গুরুত্বপূর্ণ।

#### PHP:

```php
<?php
// app/UnitOfWork/UnitOfWork.php

namespace App\UnitOfWork;

use App\Repositories\Contracts\UserRepositoryInterface;
use App\Repositories\Contracts\TransactionRepositoryInterface;
use App\Repositories\Contracts\WalletRepositoryInterface;
use Illuminate\Support\Facades\DB;

final class UnitOfWork
{
    private bool $committed = false;

    public function __construct(
        private readonly UserRepositoryInterface $userRepository,
        private readonly TransactionRepositoryInterface $transactionRepository,
        private readonly WalletRepositoryInterface $walletRepository,
    ) {}

    public function users(): UserRepositoryInterface
    {
        return $this->userRepository;
    }

    public function transactions(): TransactionRepositoryInterface
    {
        return $this->transactionRepository;
    }

    public function wallets(): WalletRepositoryInterface
    {
        return $this->walletRepository;
    }

    /**
     * @template T
     * @param callable(): T $operation
     * @return T
     */
    public function commit(callable $operation): mixed
    {
        return DB::transaction(function () use ($operation) {
            $result = $operation();
            $this->committed = true;
            return $result;
        });
    }
}
```

```php
<?php
// ব্যবহার - Fund Transfer Service

final readonly class FundTransferService
{
    public function __construct(
        private UnitOfWork $unitOfWork,
    ) {}

    public function transfer(int $senderId, int $receiverId, float $amount): void
    {
        $this->unitOfWork->commit(function () use ($senderId, $receiverId, $amount) {
            $senderWallet = $this->unitOfWork->wallets()->findByUserId($senderId);
            $receiverWallet = $this->unitOfWork->wallets()->findByUserId($receiverId);

            if ($senderWallet->balance < $amount) {
                throw new \DomainException('Insufficient balance');
            }

            $this->unitOfWork->wallets()->debit($senderId, $amount);
            $this->unitOfWork->wallets()->credit($receiverId, $amount);

            $this->unitOfWork->transactions()->create([
                'sender_id' => $senderId,
                'receiver_id' => $receiverId,
                'amount' => $amount,
                'type' => 'transfer',
                'status' => 'completed',
            ]);
        });
    }
}
```

#### JavaScript:

```javascript
// src/unitOfWork/UnitOfWork.js

import mongoose from 'mongoose';

export class UnitOfWork {
    #session = null;
    #userRepo;
    #transactionRepo;
    #walletRepo;

    constructor(userRepo, transactionRepo, walletRepo) {
        this.#userRepo = userRepo;
        this.#transactionRepo = transactionRepo;
        this.#walletRepo = walletRepo;
    }

    get users() { return this.#userRepo; }
    get transactions() { return this.#transactionRepo; }
    get wallets() { return this.#walletRepo; }

    async commit(operation) {
        this.#session = await mongoose.startSession();
        this.#session.startTransaction();

        try {
            const result = await operation(this.#session);
            await this.#session.commitTransaction();
            return result;
        } catch (error) {
            await this.#session.abortTransaction();
            throw error;
        } finally {
            this.#session.endSession();
        }
    }
}

// ব্যবহার:
class FundTransferService {
    #unitOfWork;
    constructor(unitOfWork) { this.#unitOfWork = unitOfWork; }

    async transfer(senderId, receiverId, amount) {
        return this.#unitOfWork.commit(async (session) => {
            const senderWallet = await this.#unitOfWork.wallets.findByUserId(senderId);

            if (senderWallet.balance < amount) {
                throw new Error('Insufficient balance');
            }

            await this.#unitOfWork.wallets.debit(senderId, amount, session);
            await this.#unitOfWork.wallets.credit(receiverId, amount, session);

            await this.#unitOfWork.transactions.create({
                senderId, receiverId, amount,
                type: 'transfer',
                status: 'completed',
            });
        });
    }
}
```

---

### ৫. Repository with Event Sourcing

**Event Sourcing**-এ entity-র বর্তমান state সরাসরি সংরক্ষণ না করে, entity-তে ঘটে যাওয়া **সকল event** সংরক্ষণ করা হয়। Entity-র state পুনর্গঠন করতে হলে সব event replay করতে হয়। Fintech app-এ audit trail-এর জন্য এটি অত্যন্ত কার্যকর।

#### PHP:

```php
<?php
// app/Events/DomainEvent.php

namespace App\Events;

abstract readonly class DomainEvent
{
    public function __construct(
        public string $aggregateId,
        public string $eventType,
        public array $payload,
        public \DateTimeImmutable $occurredAt = new \DateTimeImmutable(),
    ) {}
}

// app/Events/Wallet/MoneyDeposited.php
final readonly class MoneyDeposited extends DomainEvent
{
    public function __construct(string $walletId, float $amount)
    {
        parent::__construct($walletId, 'money_deposited', ['amount' => $amount]);
    }
}

// app/Events/Wallet/MoneyWithdrawn.php
final readonly class MoneyWithdrawn extends DomainEvent
{
    public function __construct(string $walletId, float $amount)
    {
        parent::__construct($walletId, 'money_withdrawn', ['amount' => $amount]);
    }
}
```

```php
<?php
// app/Repositories/EventSourced/EventSourcedWalletRepository.php

namespace App\Repositories\EventSourced;

use App\Domain\Wallet;
use App\Events\DomainEvent;
use Illuminate\Support\Facades\DB;

final class EventSourcedWalletRepository
{
    public function save(Wallet $wallet): void
    {
        $events = $wallet->releaseEvents();

        DB::table('wallet_events')->insert(
            array_map(fn (DomainEvent $event) => [
                'aggregate_id' => $event->aggregateId,
                'event_type' => $event->eventType,
                'payload' => json_encode($event->payload),
                'occurred_at' => $event->occurredAt->format('Y-m-d H:i:s'),
            ], $events)
        );
    }

    public function findById(string $walletId): Wallet
    {
        $events = DB::table('wallet_events')
            ->where('aggregate_id', $walletId)
            ->orderBy('occurred_at')
            ->get();

        $wallet = new Wallet($walletId);

        foreach ($events as $event) {
            $wallet->applyEvent(
                $event->event_type,
                json_decode($event->payload, true),
            );
        }

        return $wallet;
    }
}
```

#### JavaScript:

```javascript
// src/repositories/eventSourced/EventSourcedWalletRepository.js

export class EventSourcedWalletRepository {
    #eventStore;

    constructor(eventStore) {
        this.#eventStore = eventStore;
    }

    async save(wallet) {
        const events = wallet.releaseEvents();
        for (const event of events) {
            await this.#eventStore.append({
                aggregateId: event.aggregateId,
                eventType: event.eventType,
                payload: event.payload,
                occurredAt: event.occurredAt,
            });
        }
    }

    async findById(walletId) {
        const events = await this.#eventStore.getEvents(walletId);
        const wallet = new Wallet(walletId);

        for (const event of events) {
            wallet.applyEvent(event.eventType, event.payload);
        }

        return wallet;
    }
}

// Wallet Aggregate
class Wallet {
    #id;
    #balance = 0;
    #events = [];

    constructor(id) { this.#id = id; }

    get balance() { return this.#balance; }

    deposit(amount) {
        this.#events.push({
            aggregateId: this.#id,
            eventType: 'money_deposited',
            payload: { amount },
            occurredAt: new Date(),
        });
        this.#balance += amount;
    }

    withdraw(amount) {
        if (this.#balance < amount) throw new Error('Insufficient balance');
        this.#events.push({
            aggregateId: this.#id,
            eventType: 'money_withdrawn',
            payload: { amount },
            occurredAt: new Date(),
        });
        this.#balance -= amount;
    }

    applyEvent(type, payload) {
        switch (type) {
            case 'money_deposited': this.#balance += payload.amount; break;
            case 'money_withdrawn': this.#balance -= payload.amount; break;
        }
    }

    releaseEvents() {
        const events = [...this.#events];
        this.#events = [];
        return events;
    }
}
```

---

### ৬. Multi-Database Repository

একটি বড় মাপের system-এ বিভিন্ন ধরনের data বিভিন্ন database-এ থাকতে পারে — transactional data MySQL-এ, search data Elasticsearch-এ, session data Redis-তে। **একই interface, ভিন্ন storage backend** — এটাই Multi-Database Repository-র শক্তি।

#### PHP:

```php
<?php
// app/Repositories/Contracts/ProductSearchRepositoryInterface.php

namespace App\Repositories\Contracts;

interface ProductSearchRepositoryInterface
{
    public function search(string $query, array $filters = []): array;
    public function index(array $product): void;
    public function removeFromIndex(int $productId): void;
}

// MySQL Implementation
final readonly class MySQLProductSearchRepository implements ProductSearchRepositoryInterface
{
    public function search(string $query, array $filters = []): array
    {
        return Product::where('name', 'LIKE', "%{$query}%")
            ->when(isset($filters['category']), fn ($q) => $q->where('category', $filters['category']))
            ->get()
            ->toArray();
    }

    public function index(array $product): void { /* MySQL-তে auto-indexed */ }
    public function removeFromIndex(int $productId): void { /* No-op */ }
}

// Elasticsearch Implementation
final readonly class ElasticsearchProductSearchRepository implements ProductSearchRepositoryInterface
{
    public function __construct(
        private \Elasticsearch\Client $client,
    ) {}

    public function search(string $query, array $filters = []): array
    {
        $body = [
            'query' => [
                'bool' => [
                    'must' => [
                        ['multi_match' => ['query' => $query, 'fields' => ['name^3', 'description']]],
                    ],
                    'filter' => array_map(fn ($key, $val) => ['term' => [$key => $val]], 
                        array_keys($filters), array_values($filters)),
                ],
            ],
        ];

        $response = $this->client->search(['index' => 'products', 'body' => $body]);
        return array_column($response['hits']['hits'], '_source');
    }

    public function index(array $product): void
    {
        $this->client->index([
            'index' => 'products',
            'id' => $product['id'],
            'body' => $product,
        ]);
    }

    public function removeFromIndex(int $productId): void
    {
        $this->client->delete(['index' => 'products', 'id' => $productId]);
    }
}

// Repository Factory
final readonly class ProductSearchRepositoryFactory
{
    public static function create(string $driver = 'elasticsearch'): ProductSearchRepositoryInterface
    {
        return match ($driver) {
            'mysql' => app(MySQLProductSearchRepository::class),
            'elasticsearch' => app(ElasticsearchProductSearchRepository::class),
            default => throw new \InvalidArgumentException("Unknown driver: {$driver}"),
        };
    }
}
```

#### JavaScript:

```javascript
// src/factories/ProductSearchRepositoryFactory.js

import { MySQLProductSearchRepo } from '../repositories/mysql/ProductSearchRepo.js';
import { ElasticsearchProductSearchRepo } from '../repositories/elasticsearch/ProductSearchRepo.js';
import { RedisProductSearchRepo } from '../repositories/redis/ProductSearchRepo.js';

export class ProductSearchRepositoryFactory {
    static create(driver = 'elasticsearch') {
        const repos = {
            mysql: () => new MySQLProductSearchRepo(),
            elasticsearch: () => new ElasticsearchProductSearchRepo(),
            redis: () => new RedisProductSearchRepo(),
        };

        const factory = repos[driver];
        if (!factory) throw new Error(`Unknown driver: ${driver}`);
        return factory();
    }
}

// Elasticsearch implementation
export class ElasticsearchProductSearchRepo {
    #client;

    constructor(client) {
        this.#client = client;
    }

    async search(query, filters = {}) {
        const { hits } = await this.#client.search({
            index: 'products',
            body: {
                query: {
                    bool: {
                        must: [{ multi_match: { query, fields: ['name^3', 'description'] } }],
                        filter: Object.entries(filters).map(([k, v]) => ({ term: { [k]: v } })),
                    },
                },
            },
        });
        return hits.hits.map(h => h._source);
    }

    async index(product) {
        await this.#client.index({ index: 'products', id: product.id, body: product });
    }

    async removeFromIndex(productId) {
        await this.#client.delete({ index: 'products', id: productId });
    }
}
```

---

### ৭. Repository in Hexagonal Architecture

**Hexagonal Architecture** (Ports & Adapters)-এ Repository একটি **driven port** (secondary port) হিসেবে কাজ করে। Domain layer একটি port (interface) define করে, আর infrastructure layer সেই port-এর **adapter** (implementation) তৈরি করে।

```
                    ┌──────────────────────────────────────┐
                    │          Application Core            │
                    │                                      │
   Driving         │   ┌──────────┐    ┌──────────────┐   │        Driven
   Adapters        │   │          │    │              │   │        Adapters
                   │   │ Service  │───▶│  Repository  │   │
  ┌──────────┐     │   │ Layer    │    │  Port        │   │     ┌──────────────┐
  │ HTTP API │────▶│   │          │    │ (Interface)  │───│────▶│ PostgreSQL   │
  └──────────┘     │   └──────────┘    └──────────────┘   │     │ Adapter      │
  ┌──────────┐     │                                      │     └──────────────┘
  │   CLI    │────▶│                                      │     ┌──────────────┐
  └──────────┘     │                                      │────▶│ In-Memory    │
  ┌──────────┐     │                                      │     │ Adapter      │
  │  gRPC    │────▶│                                      │     └──────────────┘
  └──────────┘     └──────────────────────────────────────┘
```

```php
<?php
// Domain Layer - Port
namespace App\Domain\Ports;

use App\Domain\Entities\Account;

interface AccountRepositoryPort
{
    public function findById(string $id): ?Account;
    public function save(Account $account): void;
    public function findByPhoneNumber(string $phone): ?Account;
}

// Infrastructure Layer - PostgreSQL Adapter
namespace App\Infrastructure\Adapters;

final readonly class PostgresAccountRepository implements AccountRepositoryPort
{
    public function __construct(private \PDO $pdo) {}

    public function findById(string $id): ?Account
    {
        $stmt = $this->pdo->prepare('SELECT * FROM accounts WHERE id = :id');
        $stmt->execute(['id' => $id]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row ? Account::fromArray($row) : null;
    }

    public function save(Account $account): void
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO accounts (id, phone, balance) VALUES (:id, :phone, :balance)
             ON CONFLICT (id) DO UPDATE SET phone = :phone, balance = :balance'
        );
        $stmt->execute($account->toArray());
    }

    public function findByPhoneNumber(string $phone): ?Account
    {
        $stmt = $this->pdo->prepare('SELECT * FROM accounts WHERE phone = :phone');
        $stmt->execute(['phone' => $phone]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row ? Account::fromArray($row) : null;
    }
}

// Testing - In-Memory Adapter
final class InMemoryAccountRepository implements AccountRepositoryPort
{
    /** @var array<string, Account> */
    private array $accounts = [];

    public function findById(string $id): ?Account
    {
        return $this->accounts[$id] ?? null;
    }

    public function save(Account $account): void
    {
        $this->accounts[$account->id] = $account;
    }

    public function findByPhoneNumber(string $phone): ?Account
    {
        foreach ($this->accounts as $account) {
            if ($account->phone === $phone) return $account;
        }
        return null;
    }
}
```

---

### ৮. Criteria / Query Builder in Repository

যখন repository-তে query flexibility দরকার কিন্তু raw ORM queries expose করতে চান না, তখন **Criteria object** ব্যবহার করুন।

#### PHP:

```php
<?php
// app/Repositories/Criteria/Criteria.php

namespace App\Repositories\Criteria;

final class Criteria
{
    private array $filters = [];
    private array $orderBy = [];
    private ?int $limit = null;
    private ?int $offset = null;
    private array $includes = [];

    public static function create(): self
    {
        return new self();
    }

    public function where(string $field, mixed $operator, mixed $value = null): self
    {
        if ($value === null) {
            $value = $operator;
            $operator = '=';
        }
        $this->filters[] = compact('field', 'operator', 'value');
        return $this;
    }

    public function orderBy(string $field, string $direction = 'asc'): self
    {
        $this->orderBy[] = compact('field', 'direction');
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function offset(int $offset): self
    {
        $this->offset = $offset;
        return $this;
    }

    public function with(string ...$relations): self
    {
        $this->includes = array_merge($this->includes, $relations);
        return $this;
    }

    public function getFilters(): array { return $this->filters; }
    public function getOrderBy(): array { return $this->orderBy; }
    public function getLimit(): ?int { return $this->limit; }
    public function getOffset(): ?int { return $this->offset; }
    public function getIncludes(): array { return $this->includes; }
}
```

```php
<?php
// Repository-তে Criteria apply করা

public function matchCriteria(Criteria $criteria): Collection
{
    $query = $this->model->newQuery();

    foreach ($criteria->getFilters() as $filter) {
        $query->where($filter['field'], $filter['operator'], $filter['value']);
    }

    foreach ($criteria->getOrderBy() as $order) {
        $query->orderBy($order['field'], $order['direction']);
    }

    if ($criteria->getIncludes()) {
        $query->with($criteria->getIncludes());
    }

    if ($criteria->getLimit()) {
        $query->limit($criteria->getLimit());
    }

    if ($criteria->getOffset()) {
        $query->offset($criteria->getOffset());
    }

    return $query->get();
}

// ব্যবহার:
$criteria = Criteria::create()
    ->where('category', 'electronics')
    ->where('price', '<=', 50000)
    ->where('is_active', true)
    ->orderBy('price', 'asc')
    ->with('brand', 'reviews')
    ->limit(20);

$products = $this->productRepository->matchCriteria($criteria);
```

#### JavaScript:

```javascript
// src/repositories/criteria/Criteria.js

export class Criteria {
    #filters = [];
    #orderBy = [];
    #limit = null;
    #offset = null;
    #includes = [];

    static create() {
        return new Criteria();
    }

    where(field, operator, value) {
        if (value === undefined) { value = operator; operator = '='; }
        this.#filters.push({ field, operator, value });
        return this;
    }

    orderBy(field, direction = 'asc') {
        this.#orderBy.push({ field, direction });
        return this;
    }

    limit(limit) { this.#limit = limit; return this; }
    offset(offset) { this.#offset = offset; return this; }
    with(...relations) { this.#includes.push(...relations); return this; }

    toMongoQuery() {
        const operatorMap = { '=': '$eq', '!=': '$ne', '>': '$gt', '>=': '$gte', '<': '$lt', '<=': '$lte' };
        const filter = {};

        for (const { field, operator, value } of this.#filters) {
            filter[field] = operator === '=' ? value : { [operatorMap[operator]]: value };
        }

        const sort = {};
        for (const { field, direction } of this.#orderBy) {
            sort[field] = direction === 'asc' ? 1 : -1;
        }

        return { filter, sort, limit: this.#limit, skip: this.#offset, populate: this.#includes };
    }
}

// Repository-তে:
async matchCriteria(criteria) {
    const { filter, sort, limit, skip, populate } = criteria.toMongoQuery();
    let query = this._model.find(filter).sort(sort);
    if (populate.length) query = query.populate(populate.join(' '));
    if (limit) query = query.limit(limit);
    if (skip) query = query.skip(skip);
    return query.lean();
}

// ব্যবহার:
const criteria = Criteria.create()
    .where('category', 'electronics')
    .where('price', '<=', 50000)
    .orderBy('price', 'asc')
    .with('brand', 'reviews')
    .limit(20);

const products = await productRepo.matchCriteria(criteria);
```

---

## 🆚 Repository vs Other Data Access Patterns

### বিস্তারিত তুলনা:

| দিক | Repository | Active Record | Data Mapper | DAO | Query Object |
|---|---|---|---|---|---|
| **Abstraction Level** | High (Domain) | Low (Table) | Medium (Mapping) | Low (SQL) | Medium (Query) |
| **Domain Focus** | ✅ Entity-centric | ❌ Table-centric | ✅ Entity-centric | ❌ Data-centric | ❌ Query-centric |
| **Testability** | ✅ সহজে mock | ❌ DB dependency | ⚠️ মাঝামাঝি | ⚠️ মাঝামাঝি | ⚠️ মাঝামাঝি |
| **Complexity** | মাঝারি-উচ্চ | নিম্ন | উচ্চ | নিম্ন | মাঝারি |
| **ORM Independence** | ✅ হ্যাঁ | ❌ না | ⚠️ আংশিক | ✅ হ্যাঁ | ❌ না |
| **Learning Curve** | মাঝারি | সহজ | কঠিন | সহজ | মাঝারি |
| **Use Case** | Complex domain | Simple CRUD | Complex mapping | Legacy/raw SQL | Dynamic queries |

### Repository vs Active Record (Laravel Eloquent সরাসরি):

```php
// ❌ Active Record — Controller সরাসরি Model ব্যবহার করছে
class UserController extends Controller
{
    public function show(int $id)
    {
        // Controller জানে Eloquent ব্যবহার হচ্ছে — tight coupling
        $user = User::where('id', $id)->where('is_active', true)->first();
        return response()->json($user);
    }
}

// ✅ Repository — Controller শুধু interface জানে
class UserController extends Controller
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
    ) {}

    public function show(int $id)
    {
        // Controller-এর কোনো ধারণা নেই পেছনে কী ORM চলছে
        $user = $this->users->findById($id);
        return response()->json($user);
    }
}
```

---

## ✅ সুবিধা (Pros)

1. **Testability**: Interface-এর বিপরীতে mock বা in-memory implementation দিয়ে unit test করা যায়
2. **Loose Coupling**: Domain layer database technology থেকে মুক্ত
3. **Single Responsibility**: Data access logic একটি নির্দিষ্ট জায়গায় থাকে
4. **Swappable Storage**: MySQL থেকে MongoDB-তে switch করতে শুধু নতুন implementation লাগবে
5. **Domain Language**: Repository method-এর নাম domain-এর ভাষায় হয় (`findActiveUsers()` vs raw SQL)
6. **Caching/Logging**: Decorator pattern দিয়ে cross-cutting concerns যোগ করা সহজ
7. **Consistency**: পুরো application জুড়ে data access-এর consistent pattern

## ❌ অসুবিধা (Cons)

1. **Over-engineering**: ছোট CRUD app-এর জন্য অতিরিক্ত complexity
2. **Extra Abstraction Layer**: Code navigate করতে আরেকটু বেশি সময় লাগে
3. **Potential for Leaky Abstraction**: ORM-specific behavior leak হতে পারে
4. **Method Explosion**: Entity-র বিভিন্ন query-র জন্য method বাড়তেই থাকে
5. **Performance**: অতিরিক্ত layer-এর কারণে সামান্য performance overhead (negligible)
6. **Boilerplate**: Interface + Implementation + Binding — বেশি code লিখতে হয়

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. প্রতিটি Database Table-এর জন্য Repository (ভুল!)

Repository হওয়া উচিত **Aggregate Root** কেন্দ্রিক, table কেন্দ্রিক না।

```php
// ❌ ভুল — প্রতিটি table-এ আলাদা repository
class OrderRepository {}
class OrderItemRepository {}      // এটা আলাদা repository হওয়া উচিত না!
class OrderStatusRepository {}    // এটাও না!

// ✅ সঠিক — Order হলো Aggregate Root
class OrderRepository
{
    public function findWithItems(int $orderId): ?Order
    {
        // Order সাথে তার items, status সব একসাথে load হবে
        return Order::with(['items', 'status'])->find($orderId);
    }

    public function addItem(Order $order, OrderItem $item): void
    {
        $order->items()->save($item);
    }
}
```

### ২. ORM Queries Leak করা

```php
// ❌ ভুল — Repository থেকে Query Builder return করছে
interface UserRepositoryInterface
{
    public function getActiveUsersQuery(): Builder;  // Eloquent Builder leak হচ্ছে!
}

// ✅ সঠিক — Repository সম্পূর্ণ result return করবে
interface UserRepositoryInterface
{
    public function findActive(): Collection;
}
```

### ৩. Fat Repository (অতিরিক্ত Methods)

```php
// ❌ ভুল — Repository-তে অসংখ্য specific method
interface UserRepositoryInterface
{
    public function findByName(string $name): Collection;
    public function findByAge(int $age): Collection;
    public function findByNameAndAge(string $name, int $age): Collection;
    public function findByNameOrEmail(string $name, string $email): Collection;
    public function findByNameAndAgeAndCity(string $name, int $age, string $city): Collection;
    // ... ৫০টা method!
}

// ✅ সঠিক — Specification বা Criteria Pattern ব্যবহার করুন
interface UserRepositoryInterface
{
    public function findById(int $id): ?User;
    public function save(User $user): User;
    public function delete(User $user): bool;
    public function match(Specification $spec): Collection;
    public function matchCriteria(Criteria $criteria): Collection;
}
```

### ৪. Interface ছাড়া Repository

```javascript
// ❌ ভুল — সরাসরি concrete class ব্যবহার
class UserService {
    constructor() {
        this.repo = new MongooseUserRepository();  // Tight coupling!
    }
}

// ✅ সঠিক — Dependency Injection
class UserService {
    #repo;
    constructor(userRepository) {
        this.#repo = userRepository;  // Interface-এর উপর নির্ভরশীল
    }
}
```

---

## 🧪 টেস্টিং রিপোজিটরি

### In-Memory Repository (Unit Testing):

#### PHP (PHPUnit):

```php
<?php
// tests/Unit/InMemoryUserRepository.php

namespace Tests\Unit;

use App\Models\User;
use App\Repositories\Contracts\UserRepositoryInterface;
use Illuminate\Support\Collection;
use Illuminate\Pagination\LengthAwarePaginator;

final class InMemoryUserRepository implements UserRepositoryInterface
{
    private array $users = [];

    public function findById(int $id): ?User
    {
        return $this->users[$id] ?? null;
    }

    public function findByEmail(string $email): ?User
    {
        return collect($this->users)->firstWhere('email', $email);
    }

    public function findActive(): Collection
    {
        return collect($this->users)->filter(fn (User $u) => $u->is_active);
    }

    public function save(User $user): User
    {
        if (!$user->id) $user->id = count($this->users) + 1;
        $this->users[$user->id] = $user;
        return $user;
    }

    public function delete(User $user): bool
    {
        unset($this->users[$user->id]);
        return true;
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        $items = collect($this->users);
        return new LengthAwarePaginator($items->take($perPage), $items->count(), $perPage);
    }

    public function findByPhoneNumber(string $phone): ?User
    {
        return collect($this->users)->firstWhere('phone', $phone);
    }
}
```

```php
<?php
// tests/Unit/UserServiceTest.php

namespace Tests\Unit;

use App\Services\UserService;
use App\Models\User;
use PHPUnit\Framework\TestCase;

final class UserServiceTest extends TestCase
{
    private InMemoryUserRepository $repository;
    private UserService $service;

    protected function setUp(): void
    {
        $this->repository = new InMemoryUserRepository();
        $this->service = new UserService($this->repository);
    }

    public function test_can_register_user(): void
    {
        $user = $this->service->register('রহিম', 'rahim@example.com', '01711111111');

        $this->assertNotNull($user->id);
        $this->assertEquals('rahim@example.com', $user->email);
    }

    public function test_cannot_register_duplicate_email(): void
    {
        $this->service->register('রহিম', 'rahim@example.com', '01711111111');

        $this->expectException(\DomainException::class);
        $this->service->register('করিম', 'rahim@example.com', '01722222222');
    }
}
```

#### JavaScript (Jest):

```javascript
// tests/unit/InMemoryUserRepository.js

export class InMemoryUserRepository {
    #users = new Map();
    #nextId = 1;

    async findById(id) {
        return this.#users.get(id) ?? null;
    }

    async findByEmail(email) {
        return [...this.#users.values()].find(u => u.email === email) ?? null;
    }

    async findActive() {
        return [...this.#users.values()].filter(u => u.isActive);
    }

    async save(userData) {
        if (!userData.id) userData.id = String(this.#nextId++);
        this.#users.set(userData.id, { ...userData });
        return { ...userData };
    }

    async delete(id) {
        return this.#users.delete(id);
    }

    async paginate(page = 1, perPage = 15) {
        const all = [...this.#users.values()];
        const start = (page - 1) * perPage;
        return {
            data: all.slice(start, start + perPage),
            total: all.length,
            page,
            perPage,
            totalPages: Math.ceil(all.length / perPage),
        };
    }

    async findByPhoneNumber(phone) {
        return [...this.#users.values()].find(u => u.phone === phone) ?? null;
    }
}
```

```javascript
// tests/unit/userService.test.js

import { describe, it, expect, beforeEach } from '@jest/globals';
import { UserService } from '../../src/services/UserService.js';
import { InMemoryUserRepository } from './InMemoryUserRepository.js';

describe('UserService', () => {
    let service;
    let repository;

    beforeEach(() => {
        repository = new InMemoryUserRepository();
        service = new UserService(repository);
    });

    it('should register a new user', async () => {
        const user = await service.register('রহিম', 'rahim@example.com', '01711111111');

        expect(user.id).toBeDefined();
        expect(user.email).toBe('rahim@example.com');
    });

    it('should reject duplicate email', async () => {
        await service.register('রহিম', 'rahim@example.com', '01711111111');

        await expect(
            service.register('করিম', 'rahim@example.com', '01722222222'),
        ).rejects.toThrow('Email already exists');
    });

    it('should find user by phone number', async () => {
        await service.register('রহিম', 'rahim@example.com', '01711111111');
        const found = await repository.findByPhoneNumber('01711111111');

        expect(found).not.toBeNull();
        expect(found.email).toBe('rahim@example.com');
    });
});
```

---

## 📂 প্রজেক্ট স্ট্রাকচার

### PHP (Laravel):

```
app/
├── Domain/
│   ├── Entities/
│   │   ├── User.php
│   │   ├── Product.php
│   │   └── Wallet.php
│   └── Events/
│       ├── DomainEvent.php
│       └── Wallet/
│           ├── MoneyDeposited.php
│           └── MoneyWithdrawn.php
├── Http/
│   └── Controllers/
│       ├── UserController.php
│       └── ProductController.php
├── Models/
│   ├── User.php
│   └── Product.php
├── Providers/
│   └── RepositoryServiceProvider.php
├── Repositories/
│   ├── Contracts/
│   │   ├── BaseRepositoryInterface.php
│   │   ├── UserRepositoryInterface.php
│   │   ├── ProductRepositoryInterface.php
│   │   ├── WalletRepositoryInterface.php
│   │   └── TransactionRepositoryInterface.php
│   ├── Eloquent/
│   │   ├── BaseRepository.php
│   │   ├── EloquentUserRepository.php
│   │   ├── EloquentProductRepository.php
│   │   └── EloquentWalletRepository.php
│   ├── Cache/
│   │   └── CachedProductRepository.php
│   ├── Criteria/
│   │   └── Criteria.php
│   └── EventSourced/
│       └── EventSourcedWalletRepository.php
├── Services/
│   ├── UserService.php
│   ├── ProductService.php
│   └── FundTransferService.php
├── Specifications/
│   ├── Specification.php
│   ├── AndSpecification.php
│   ├── OrSpecification.php
│   └── Product/
│       ├── PriceRangeSpecification.php
│       ├── CategorySpecification.php
│       └── InStockSpecification.php
└── UnitOfWork/
    └── UnitOfWork.php
tests/
├── Unit/
│   ├── InMemoryUserRepository.php
│   ├── UserServiceTest.php
│   └── ProductServiceTest.php
└── Integration/
    └── EloquentUserRepositoryTest.php
```

### Node.js:

```
src/
├── config/
│   └── database.js
├── container.js
├── models/
│   ├── User.js
│   └── Product.js
├── repositories/
│   ├── contracts/
│   │   └── UserRepository.js
│   ├── BaseRepository.js
│   ├── mongoose/
│   │   ├── MongooseUserRepository.js
│   │   └── MongooseProductRepository.js
│   ├── sequelize/
│   │   └── SequelizeUserRepository.js
│   ├── cache/
│   │   └── CachedProductRepository.js
│   ├── criteria/
│   │   └── Criteria.js
│   ├── eventSourced/
│   │   └── EventSourcedWalletRepository.js
│   └── elasticsearch/
│       └── ProductSearchRepo.js
├── services/
│   ├── UserService.js
│   ├── ProductService.js
│   └── FundTransferService.js
├── specifications/
│   ├── Specification.js
│   └── product/
│       ├── PriceRangeSpec.js
│       ├── CategorySpec.js
│       └── InStockSpec.js
├── unitOfWork/
│   └── UnitOfWork.js
├── factories/
│   └── ProductSearchRepositoryFactory.js
├── controllers/
│   ├── UserController.js
│   └── ProductController.js
└── routes/
    └── index.js
tests/
├── unit/
│   ├── InMemoryUserRepository.js
│   ├── userService.test.js
│   └── productService.test.js
└── integration/
    └── mongooseUserRepository.test.js
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ ব্যবহার করবেন যখন:

- **Complex Domain Logic** আছে — bKash-এর মতো fintech app যেখানে fund transfer, wallet management, transaction history সব আছে
- **একাধিক Data Source** — MySQL + Redis + Elasticsearch একসাথে ব্যবহার করছেন
- **Testability** গুরুত্বপূর্ণ — CI/CD pipeline-এ unit test দ্রুত চলাতে in-memory repository দরকার
- **টিম বড়** — ১০+ developer-এর team-এ consistent data access pattern দরকার
- **Database Migration** সম্ভাবনা আছে — MySQL থেকে PostgreSQL-এ যেতে পারেন
- **Domain-Driven Design (DDD)** follow করছেন

### ❌ ব্যবহার করবেন না যখন:

- **সাধারণ CRUD Application** — একটি blog বা ছোট portfolio site
- **Prototype / MVP** — দ্রুত market-এ যেতে হবে, Active Record যথেষ্ট
- **Solo Developer** — একা কাজ করলে over-engineering হতে পারে
- **Database কখনো বদলাবে না** — যদি নিশ্চিত থাকেন সবসময় MySQL-ই থাকবে
- **Simple Microservice** — যেটি শুধু একটি resource CRUD করে

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | সম্পর্ক |
|---|---|
| **Unit of Work** | Repository-র সাথে ব্যবহার করে transaction management করা হয় |
| **Specification** | Complex query logic-কে reusable objects-এ encapsulate করে |
| **Decorator** | Caching, logging, profiling — repository-তে layer যোগ করতে |
| **Factory** | Multi-database scenario-তে সঠিক repository instance তৈরি করতে |
| **Strategy** | Runtime-এ ভিন্ন repository implementation বেছে নিতে |
| **Domain-Driven Design** | Repository হলো DDD-র একটি core tactical pattern |
| **CQRS** | Read repository আর Write repository আলাদা রাখতে |
| **Event Sourcing** | Event-based repository implementation |
| **Hexagonal Architecture** | Repository হলো secondary/driven port |
| **Dependency Injection** | Interface-based repository binding-এর ভিত্তি |

---

## 📋 সারসংক্ষেপ

Repository Pattern হলো enterprise application development-এর একটি **মৌলিক architectural pattern** যা domain layer আর data persistence layer-এর মধ্যে **clean boundary** তৈরি করে।

### মূল শিক্ষা:

1. **সবসময় Interface দিয়ে শুরু করুন** — concrete implementation পরে আসবে
2. **Aggregate Root কেন্দ্রিক Repository তৈরি করুন** — প্রতিটি table-এর জন্য না
3. **Specification/Criteria Pattern ব্যবহার করুন** — method explosion ঠেকাতে
4. **Decorator Pattern দিয়ে caching যোগ করুন** — মূল code পরিবর্তন ছাড়াই
5. **Unit of Work ব্যবহার করুন** — transaction management-এর জন্য
6. **In-Memory Repository দিয়ে test করুন** — দ্রুত, নির্ভরযোগ্য unit test
7. **প্রয়োজনে ব্যবহার করুন** — সাধারণ CRUD app-এ over-engineering করবেন না

> **"Repository Pattern ব্যবহার করা মানে আপনার application-এর data access layer-কে একটি সুশৃঙ্খল library-র মতো সাজানো — যেখানে librarian (Repository) জানে সব কিছু কোথায় আছে, আর আপনাকে (domain logic) শুধু চাইতে হয়।"**
