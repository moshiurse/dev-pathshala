# 🔮 GraphQL — সম্পূর্ণ গভীর বিশ্লেষণ (Deep Dive)

## 📌 সংজ্ঞা (Definition)

**GraphQL** হলো API-এর জন্য একটি **query language** এবং সেই query গুলো execute করার জন্য একটি **runtime**। Facebook ২০১২ সালে অভ্যন্তরীণভাবে এটি তৈরি করে এবং ২০১৫ সালে open-source করে। GraphQL-এর মূল শক্তি হলো এর **strongly-typed schema system** — যেখানে client ঠিক যতটুকু data দরকার ঠিক ততটুকুই request করতে পারে, বেশিও না কমও না।

### কেন GraphQL তৈরি হলো?

Facebook-এর mobile app-এ REST API ব্যবহার করার সময় তিনটি বড় সমস্যা দেখা দিয়েছিল:

1. **Over-fetching**: REST endpoint থেকে অনেক বেশি data আসতো যেটা client-এর দরকার নেই
2. **Under-fetching**: একটি page render করতে multiple REST endpoint-এ call করতে হতো
3. **Versioning nightmare**: API পরিবর্তন করলে client ভেঙে যেতো, v1/v2/v3 maintain করা কষ্টকর

### মূল বৈশিষ্ট্যসমূহ

```
┌─────────────────────────────────────────────────────────┐
│                    GraphQL মূলনীতি                        │
├─────────────────────────────────────────────────────────┤
│  ১. Declarative Data Fetching                           │
│     → Client বলে কী data চাই, Server দেয়                │
│                                                         │
│  ২. Single Endpoint                                     │
│     → সব request একটি মাত্র /graphql endpoint-এ যায়     │
│                                                         │
│  ৩. Strongly Typed Schema                               │
│     → Schema Definition Language (SDL) দিয়ে API define  │
│                                                         │
│  ৪. Introspection                                       │
│     → API নিজেই বলতে পারে সে কী কী support করে          │
│                                                         │
│  ৫. Real-time with Subscriptions                        │
│     → WebSocket দিয়ে real-time data push                 │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 GraphQL vs REST — তুলনামূলক চিত্র

```
╔══════════════════════════════════════════════════════════════════╗
║              REST API Request Flow (বাংলাদেশি ই-কমার্স)          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Product Page লোড করতে:                                          ║
║                                                                  ║
║  Client ──→ GET /api/products/101           → Product info        ║
║  Client ──→ GET /api/products/101/variants  → Variants list       ║
║  Client ──→ GET /api/categories/5           → Category info       ║
║  Client ──→ GET /api/products/101/reviews   → Reviews             ║
║  Client ──→ GET /api/users/42               → Seller info         ║
║                                                                  ║
║  ✗ ৫টি HTTP request!                                             ║
║  ✗ প্রতিটি response-এ অপ্রয়োজনীয় field আসে                     ║
║  ✗ Total payload: ~15KB                                          ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║              GraphQL Request Flow                                ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Client ──→ POST /graphql  (single query)                        ║
║                                                                  ║
║  {                                                               ║
║    product(id: 101) {                                            ║
║      name, price, description                                    ║
║      variants { size, color, stock }                             ║
║      category { name, parentCategory { name } }                  ║
║      reviews(first: 5) { rating, comment }                       ║
║      seller { name, rating }                                     ║
║    }                                                             ║
║  }                                                               ║
║                                                                  ║
║  ✓ মাত্র ১টি HTTP request!                                       ║
║  ✓ শুধু যা দরকার তাই আসে                                        ║
║  ✓ Total payload: ~3KB                                           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 💻 Core Concepts (মূল ধারণাসমূহ)

### ১. Schema & Type System (স্কিমা ও টাইপ সিস্টেম)

GraphQL-এর হৃদয় হলো এর **Type System**। Schema Definition Language (SDL) দিয়ে আমরা পুরো API-এর structure define করি। এটা একটা **contract** — client এবং server উভয়েই জানে data কেমন হবে।

#### Scalar Types (আদিম টাইপ)

```graphql
# GraphQL-এর built-in scalar types
# Int     → পূর্ণসংখ্যা (32-bit)
# Float   → দশমিক সংখ্যা
# String  → UTF-8 text
# Boolean → true/false
# ID      → unique identifier (String হিসেবে serialize হয়)

# Custom Scalar — বাংলাদেশি টাকার জন্য
scalar BDT
scalar DateTime
scalar JSON
```

#### Object Types (অবজেক্ট টাইপ)

```graphql
"""
বাংলাদেশি ই-কমার্স পণ্য — এখানে প্রতিটি field-এর type define করা আছে।
! চিহ্ন মানে non-nullable (এই field অবশ্যই value থাকবে)।
"""
type Product {
  id: ID!
  name: String!
  nameBn: String                    # বাংলা নাম (nullable)
  slug: String!
  description: String
  price: Float!
  discountPrice: Float
  sku: String!
  stock: Int!
  isActive: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!

  # Relationships (সম্পর্ক)
  category: Category!
  variants: [ProductVariant!]!      # non-nullable list of non-nullable items
  reviews: [Review!]!
  seller: Seller!
  images: [ProductImage!]!
  tags: [Tag!]
}

type ProductVariant {
  id: ID!
  product: Product!
  size: String
  color: String
  weight: String
  additionalPrice: Float!
  stock: Int!
  sku: String!
}

type Category {
  id: ID!
  name: String!
  nameBn: String
  slug: String!
  parentCategory: Category          # self-referencing — nested categories
  childCategories: [Category!]!
  products: [Product!]!
  depth: Int!
}
```

#### Enum Types (গণনা টাইপ)

```graphql
enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  OUT_FOR_DELIVERY
  DELIVERED
  CANCELLED
  RETURNED
  REFUNDED
}

enum PaymentMethod {
  BKASH
  NAGAD
  ROCKET
  CARD
  CASH_ON_DELIVERY
}

enum SortOrder {
  ASC
  DESC
}

enum ProductSortField {
  PRICE
  CREATED_AT
  POPULARITY
  RATING
}
```

#### Interface (ইন্টারফেস)

```graphql
"""
Interface হলো একটি abstract type — যেকোনো type এটা implement করলে
এই field গুলো অবশ্যই থাকতে হবে।
"""
interface Node {
  id: ID!
}

interface Timestampable {
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Product implements Node & Timestampable {
  id: ID!
  name: String!
  price: Float!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Category implements Node & Timestampable {
  id: ID!
  name: String!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

#### Union Types (ইউনিয়ন টাইপ)

```graphql
"""
Union হলো OR relationship — SearchResult হতে পারে Product, Category,
অথবা Seller যেকোনো একটি।
"""
union SearchResult = Product | Category | Seller

type Query {
  search(query: String!): [SearchResult!]!
}

# Client-side query with inline fragments
# query {
#   search(query: "শাড়ি") {
#     ... on Product { name, price }
#     ... on Category { name, depth }
#     ... on Seller { name, rating }
#   }
# }
```

#### Input Types (ইনপুট টাইপ)

```graphql
"""
Input types শুধুমাত্র argument হিসেবে ব্যবহার হয়।
Regular type আর input type আলাদা রাখতে হয়।
"""
input CreateProductInput {
  name: String!
  nameBn: String
  description: String
  price: Float!
  discountPrice: Float
  categoryId: ID!
  sku: String!
  stock: Int!
  variants: [CreateVariantInput!]
  tagIds: [ID!]
}

input CreateVariantInput {
  size: String
  color: String
  weight: String
  additionalPrice: Float! = 0.0     # default value
  stock: Int!
  sku: String!
}

input ProductFilterInput {
  categoryId: ID
  minPrice: Float
  maxPrice: Float
  inStock: Boolean
  sellerId: ID
  search: String
  sortBy: ProductSortField = CREATED_AT
  sortOrder: SortOrder = DESC
}
```

---

### ২. Queries (কোয়েরি)

Query হলো GraphQL-এ data **পড়ার** উপায়। REST-এর GET request-এর সমতুল্য।

#### Basic Query

```graphql
# সবচেয়ে সাধারণ query — একটি product আনতে
query GetProduct {
  product(id: "101") {
    id
    name
    price
    stock
  }
}

# Response:
# {
#   "data": {
#     "product": {
#       "id": "101",
#       "name": "জামদানি শাড়ি",
#       "price": 5500.00,
#       "stock": 23
#     }
#   }
# }
```

#### Nested Query (নেস্টেড কোয়েরি)

```graphql
# Nested relationships — একবারেই product, category, variants সব আনা যায়
query GetProductWithDetails {
  product(id: "101") {
    name
    price
    category {
      name
      parentCategory {
        name
        parentCategory {
          name      # তিন স্তরের category hierarchy
        }
      }
    }
    variants {
      size
      color
      stock
      additionalPrice
    }
    reviews(first: 3, orderBy: { field: CREATED_AT, direction: DESC }) {
      rating
      comment
      user {
        name
      }
    }
  }
}
```

#### Arguments, Aliases & Fragments

```graphql
# Aliases — একই field-কে ভিন্ন নামে আনা
# Fragments — পুনরাবৃত্তি কমাতে shared field set
query CompareProducts {
  cheapProduct: product(id: "101") {
    ...ProductFields
  }
  expensiveProduct: product(id: "205") {
    ...ProductFields
  }
}

fragment ProductFields on Product {
  id
  name
  price
  discountPrice
  stock
  category {
    name
  }
  variants {
    size
    color
    additionalPrice
  }
}
```

#### Variables (ভেরিয়েবল)

```graphql
# Variables দিয়ে dynamic query — hardcoded value এড়ানো
query GetProducts(
  $filter: ProductFilterInput
  $first: Int = 10
  $after: String
) {
  products(filter: $filter, first: $first, after: $after) {
    edges {
      node {
        id
        name
        price
        discountPrice
        category { name }
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
      totalCount
    }
  }
}

# Variables JSON:
# {
#   "filter": {
#     "categoryId": "5",
#     "minPrice": 500,
#     "maxPrice": 5000,
#     "inStock": true,
#     "sortBy": "PRICE",
#     "sortOrder": "ASC"
#   },
#   "first": 20,
#   "after": "cursor_abc123"
# }
```

---

### ৩. Mutations (মিউটেশন)

Mutation দিয়ে data **তৈরি, পরিবর্তন বা মুছে ফেলা** হয়। REST-এর POST/PUT/PATCH/DELETE-এর সমতুল্য।

```graphql
# ── CREATE ──
mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    id
    name
    price
    sku
    variants {
      id
      size
      color
    }
  }
}

# Variables:
# {
#   "input": {
#     "name": "নকশিকাঁথা কুশন কভার",
#     "nameBn": "নকশিকাঁথা কুশন কভার",
#     "price": 850.00,
#     "categoryId": "12",
#     "sku": "NK-CUSH-001",
#     "stock": 50,
#     "variants": [
#       { "size": "16x16", "color": "লাল", "stock": 25, "sku": "NK-CUSH-001-R" },
#       { "size": "18x18", "color": "নীল", "stock": 25, "sku": "NK-CUSH-001-B" }
#     ]
#   }
# }

# ── UPDATE ──
mutation UpdateProduct($id: ID!, $input: UpdateProductInput!) {
  updateProduct(id: $id, input: $input) {
    id
    name
    price
    updatedAt
  }
}

# ── DELETE ──
mutation DeleteProduct($id: ID!) {
  deleteProduct(id: $id) {
    success
    message
  }
}

# ── COMPLEX: অর্ডার তৈরি ──
mutation PlaceOrder($input: PlaceOrderInput!) {
  placeOrder(input: $input) {
    id
    orderNumber
    status
    totalAmount
    paymentMethod
    items {
      product { name }
      variant { size, color }
      quantity
      unitPrice
    }
    shippingAddress {
      division
      district
      upazila
      address
    }
  }
}
```

---

### ৪. Subscriptions (সাবস্ক্রিপশন — রিয়েল-টাইম)

Subscription হলো WebSocket-ভিত্তিক real-time data push। কোনো event ঘটলে server স্বয়ংক্রিয়ভাবে client-কে জানায়।

```graphql
# অর্ডার স্ট্যাটাস পরিবর্তনের real-time update
subscription OnOrderStatusChanged($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    orderNumber
    status
    updatedAt
    estimatedDelivery
  }
}

# নতুন product যোগ হলে notification
subscription OnNewProduct($categoryId: ID) {
  productAdded(categoryId: $categoryId) {
    id
    name
    price
    category { name }
  }
}

# Inventory alert — stock কমে গেলে
subscription OnLowStock($threshold: Int! = 5) {
  lowStockAlert(threshold: $threshold) {
    product { id, name, sku }
    currentStock
    variant { size, color }
  }
}
```

---

### ৫. Resolvers (রিজলভার)

Resolver হলো সেই function যেটা GraphQL query-র প্রতিটি field-এর জন্য actual data fetch করে। প্রতিটি field-এর একটি resolver থাকতে পারে।

```
Resolver function signature:
resolver(parent, args, context, info)

parent  → parent object-এর resolved value
args    → query-তে পাঠানো arguments
context → সব resolver-এ shared data (auth, db, dataloaders)
info    → query execution metadata (AST, field name, path)
```

```
┌──────────────────────────────────────────────────────────┐
│              Resolver Execution Chain                     │
│                                                          │
│  Query: product(id: "101") { name, category { name } }  │
│                                                          │
│  ① Query.product(_, {id:"101"}, ctx)                     │
│     └→ DB থেকে product row fetch                         │
│     └→ return { id:101, name:"শাড়ি", category_id: 5 }   │
│                                                          │
│  ② Product.name(parent, _, ctx)                          │
│     └→ parent.name return (trivial resolver)             │
│     └→ return "শাড়ি"                                     │
│                                                          │
│  ③ Product.category(parent, _, ctx)                      │
│     └→ parent.category_id দিয়ে Category fetch            │
│     └→ return { id:5, name:"পোশাক" }                     │
│                                                          │
│  ④ Category.name(parent, _, ctx)                         │
│     └→ return "পোশাক"                                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 🔧 Implementation (বাস্তবায়ন)

### PHP: Laravel + Lighthouse

[Lighthouse](https://lighthouse-php.com/) হলো Laravel-এর জন্য সবচেয়ে জনপ্রিয় GraphQL framework। এটা schema-first approach follow করে — আপনি `.graphql` ফাইলে schema লেখেন, Lighthouse সেটা থেকে পুরো API তৈরি করে দেয়।

#### Installation ও Setup

```bash
# Laravel project-এ Lighthouse install
composer require nuwave/lighthouse

# Lighthouse-এর config publish
php artisan vendor:publish --tag=lighthouse-config

# Default schema file তৈরি
php artisan lighthouse:ide-helper

# GraphQL Playground install (development tool)
composer require mll-lab/laravel-graphql-playground
```

#### Schema (schema.graphql)

```graphql
# schema.graphql — Lighthouse-এর entry point

"Root query type — সব read operation এখানে"
type Query {
  "একটি নির্দিষ্ট product আনতে"
  product(id: ID! @eq): Product @find

  "Product list — filter ও pagination সহ"
  products(
    filter: ProductFilterInput @spread
    orderBy: _ @orderBy(columns: ["price", "created_at", "name"])
  ): [Product!]! @paginate(defaultCount: 15)

  "Category tree — nested categories"
  categories(
    parentId: ID @eq(key: "parent_id")
  ): [Category!]! @all

  "Global search — product, category, seller সব খুঁজবে"
  search(query: String! @trim): [SearchResult!]!
    @field(resolver: "App\\GraphQL\\Queries\\SearchQuery")

  "Authenticated user-এর তথ্য"
  me: User @auth @guard(with: ["sanctum"])
}

type Mutation {
  "নতুন product তৈরি — শুধু seller/admin পারবে"
  createProduct(input: CreateProductInput! @spread): Product
    @guard(with: ["sanctum"])
    @can(ability: "create", model: "App\\Models\\Product")
    @field(resolver: "App\\GraphQL\\Mutations\\CreateProductMutation")

  "Product update"
  updateProduct(id: ID!, input: UpdateProductInput! @spread): Product
    @guard(with: ["sanctum"])
    @can(ability: "update", find: "id", model: "App\\Models\\Product")
    @field(resolver: "App\\GraphQL\\Mutations\\UpdateProductMutation")

  "Product delete (soft delete)"
  deleteProduct(id: ID!): DeleteResponse!
    @guard(with: ["sanctum"])
    @can(ability: "delete", find: "id", model: "App\\Models\\Product")
    @field(resolver: "App\\GraphQL\\Mutations\\DeleteProductMutation")

  "অর্ডার দেওয়া"
  placeOrder(input: PlaceOrderInput!): Order!
    @guard(with: ["sanctum"])
    @field(resolver: "App\\GraphQL\\Mutations\\PlaceOrderMutation")

  "User registration"
  register(input: RegisterInput! @spread): AuthPayload!
    @field(resolver: "App\\GraphQL\\Mutations\\RegisterMutation")

  "User login"
  login(email: String!, password: String!): AuthPayload!
    @field(resolver: "App\\GraphQL\\Mutations\\LoginMutation")
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order
    @subscription(class: "App\\GraphQL\\Subscriptions\\OrderStatusSubscription")
}

# ── Types ──

type Product {
  id: ID!
  name: String!
  name_bn: String
  slug: String!
  description: String
  price: Float!
  discount_price: Float
  sku: String!
  stock: Int!
  is_active: Boolean!
  created_at: DateTime!
  updated_at: DateTime!

  category: Category! @belongsTo
  variants: [ProductVariant!]! @hasMany
  reviews: [Review!]! @hasMany
  seller: Seller! @belongsTo
  images: [ProductImage!]! @hasMany
  tags: [Tag!]! @belongsToMany

  # Computed fields
  averageRating: Float
    @field(resolver: "App\\GraphQL\\Fields\\AverageRatingField")
  reviewCount: Int! @count(relation: "reviews")
  isInStock: Boolean!
    @method(name: "isInStock")
}

type ProductVariant {
  id: ID!
  size: String
  color: String
  weight: String
  additional_price: Float!
  stock: Int!
  sku: String!
  product: Product! @belongsTo
}

type Category {
  id: ID!
  name: String!
  name_bn: String
  slug: String!
  depth: Int!
  parent: Category @belongsTo(relation: "parentCategory")
  children: [Category!]! @hasMany(relation: "childCategories")
  products: [Product!]! @hasMany
  productCount: Int! @count(relation: "products")
}

type AuthPayload {
  token: String!
  user: User!
}

type DeleteResponse {
  success: Boolean!
  message: String!
}
```

#### Eloquent Models

```php
<?php
// app/Models/Product.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Product extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name', 'name_bn', 'slug', 'description', 'price',
        'discount_price', 'sku', 'stock', 'is_active',
        'category_id', 'seller_id',
    ];

    protected $casts = [
        'price' => 'float',
        'discount_price' => 'float',
        'stock' => 'integer',
        'is_active' => 'boolean',
    ];

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function variants(): HasMany
    {
        return $this->hasMany(ProductVariant::class);
    }

    public function reviews(): HasMany
    {
        return $this->hasMany(Review::class);
    }

    public function seller(): BelongsTo
    {
        return $this->belongsTo(Seller::class);
    }

    public function images(): HasMany
    {
        return $this->hasMany(ProductImage::class);
    }

    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }

    public function isInStock(): bool
    {
        return $this->stock > 0;
    }
}
```

#### Custom Resolver (Mutation)

```php
<?php
// app/GraphQL/Mutations/CreateProductMutation.php

namespace App\GraphQL\Mutations;

use App\Models\Product;
use App\Models\ProductVariant;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

final class CreateProductMutation
{
    public function __invoke(mixed $root, array $args): Product
    {
        return DB::transaction(function () use ($args) {
            $input = $args['input'];

            $product = Product::create([
                'name'           => $input['name'],
                'name_bn'        => $input['name_bn'] ?? null,
                'slug'           => Str::slug($input['name']),
                'description'    => $input['description'] ?? null,
                'price'          => $input['price'],
                'discount_price' => $input['discount_price'] ?? null,
                'sku'            => $input['sku'],
                'stock'          => $input['stock'],
                'category_id'    => $input['category_id'],
                'seller_id'      => auth()->id(),
                'is_active'      => true,
            ]);

            // Variants তৈরি
            if (!empty($input['variants'])) {
                foreach ($input['variants'] as $variantData) {
                    $product->variants()->create($variantData);
                }

                // মোট stock হিসাব (variants থেকে)
                $totalStock = $product->variants()->sum('stock');
                $product->update(['stock' => $totalStock]);
            }

            // Tags সংযুক্ত
            if (!empty($input['tag_ids'])) {
                $product->tags()->attach($input['tag_ids']);
            }

            return $product->fresh(['category', 'variants', 'tags']);
        });
    }
}
```

#### Custom Query Resolver

```php
<?php
// app/GraphQL/Queries/SearchQuery.php

namespace App\GraphQL\Queries;

use App\Models\Product;
use App\Models\Category;
use App\Models\Seller;
use Illuminate\Support\Collection;

final class SearchQuery
{
    public function __invoke(mixed $root, array $args): Collection
    {
        $query = $args['query'];

        $products = Product::where('name', 'LIKE', "%{$query}%")
            ->orWhere('name_bn', 'LIKE', "%{$query}%")
            ->where('is_active', true)
            ->limit(10)
            ->get();

        $categories = Category::where('name', 'LIKE', "%{$query}%")
            ->orWhere('name_bn', 'LIKE', "%{$query}%")
            ->limit(5)
            ->get();

        $sellers = Seller::where('name', 'LIKE', "%{$query}%")
            ->limit(5)
            ->get();

        return collect()
            ->merge($products)
            ->merge($categories)
            ->merge($sellers);
    }
}
```

#### Authentication Middleware

```php
<?php
// app/GraphQL/Mutations/LoginMutation.php

namespace App\GraphQL\Mutations;

use App\Models\User;
use Illuminate\Support\Facades\Hash;
use GraphQL\Error\Error;

final class LoginMutation
{
    public function __invoke(mixed $root, array $args): array
    {
        $user = User::where('email', $args['email'])->first();

        if (!$user || !Hash::check($args['password'], $user->password)) {
            throw new Error(
                'ইমেইল বা পাসওয়ার্ড ভুল',
                extensions: ['code' => 'INVALID_CREDENTIALS']
            );
        }

        $token = $user->createToken('auth-token')->plainTextToken;

        return [
            'token' => $token,
            'user'  => $user,
        ];
    }
}
```

---

### JS: Apollo Server + Express

Apollo Server হলো JavaScript/TypeScript ecosystem-এর সবচেয়ে জনপ্রিয় GraphQL server। এটা Node.js-এ চলে এবং Express, Fastify সহ বিভিন্ন framework-এর সাথে কাজ করে।

#### Setup

```bash
npm init -y
npm install @apollo/server graphql express cors
npm install @apollo/server-express4 @graphql-tools/merge
npm install dataloader
npm install -D nodemon
```

#### Type Definitions

```javascript
// src/schema/typeDefs.js

const { gql } = require('graphql-tag');

const typeDefs = gql`
  scalar DateTime

  type Query {
    product(id: ID!): Product
    products(filter: ProductFilterInput, pagination: PaginationInput): ProductConnection!
    categories(parentId: ID): [Category!]!
    search(query: String!): [SearchResult!]!
    me: User
  }

  type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    deleteProduct(id: ID!): DeleteResponse!
    placeOrder(input: PlaceOrderInput!): Order!
    register(input: RegisterInput!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!
  }

  type Subscription {
    orderStatusChanged(orderId: ID!): Order!
    productAdded(categoryId: ID): Product!
  }

  type Product {
    id: ID!
    name: String!
    nameBn: String
    slug: String!
    description: String
    price: Float!
    discountPrice: Float
    sku: String!
    stock: Int!
    isActive: Boolean!
    createdAt: DateTime!
    updatedAt: DateTime!
    category: Category!
    variants: [ProductVariant!]!
    reviews(first: Int, after: String): ReviewConnection!
    seller: Seller!
    images: [ProductImage!]!
    tags: [Tag!]!
    averageRating: Float
    reviewCount: Int!
  }

  type ProductVariant {
    id: ID!
    size: String
    color: String
    weight: String
    additionalPrice: Float!
    stock: Int!
    sku: String!
  }

  type Category {
    id: ID!
    name: String!
    nameBn: String
    slug: String!
    depth: Int!
    parent: Category
    children: [Category!]!
    products: [Product!]!
    productCount: Int!
  }

  union SearchResult = Product | Category | Seller

  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  input ProductFilterInput {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
    search: String
  }

  input PaginationInput {
    first: Int = 10
    after: String
  }

  input CreateProductInput {
    name: String!
    nameBn: String
    description: String
    price: Float!
    discountPrice: Float
    categoryId: ID!
    sku: String!
    stock: Int!
    variants: [CreateVariantInput!]
    tagIds: [ID!]
  }

  input CreateVariantInput {
    size: String
    color: String
    weight: String
    additionalPrice: Float! = 0.0
    stock: Int!
    sku: String!
  }

  type AuthPayload {
    token: String!
    user: User!
  }

  type DeleteResponse {
    success: Boolean!
    message: String!
  }
`;

module.exports = typeDefs;
```

#### Resolvers

```javascript
// src/resolvers/index.js

const { PubSub } = require('graphql-subscriptions');
const { GraphQLError } = require('graphql');

const pubsub = new PubSub();
const ORDER_STATUS_CHANGED = 'ORDER_STATUS_CHANGED';
const PRODUCT_ADDED = 'PRODUCT_ADDED';

const resolvers = {
  Query: {
    product: async (_, { id }, { dataSources }) => {
      return dataSources.productAPI.getById(id);
    },

    products: async (_, { filter, pagination }, { dataSources }) => {
      const { first = 10, after } = pagination || {};
      return dataSources.productAPI.getProducts(filter, first, after);
    },

    categories: async (_, { parentId }, { dataSources }) => {
      return dataSources.categoryAPI.getByParent(parentId);
    },

    search: async (_, { query }, { dataSources }) => {
      const [products, categories, sellers] = await Promise.all([
        dataSources.productAPI.search(query),
        dataSources.categoryAPI.search(query),
        dataSources.sellerAPI.search(query),
      ]);
      return [...products, ...categories, ...sellers];
    },

    me: async (_, __, { user }) => {
      if (!user) {
        throw new GraphQLError('অনুমোদন প্রয়োজন', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }
      return user;
    },
  },

  Mutation: {
    createProduct: async (_, { input }, { user, dataSources }) => {
      if (!user) {
        throw new GraphQLError('লগইন করুন', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }

      const product = await dataSources.productAPI.create({
        ...input,
        sellerId: user.id,
      });

      // Real-time notification
      pubsub.publish(PRODUCT_ADDED, { productAdded: product });

      return product;
    },

    updateProduct: async (_, { id, input }, { user, dataSources }) => {
      if (!user) {
        throw new GraphQLError('লগইন করুন', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }

      const existing = await dataSources.productAPI.getById(id);
      if (existing.sellerId !== user.id && user.role !== 'ADMIN') {
        throw new GraphQLError('আপনার এই product পরিবর্তনের অনুমতি নেই', {
          extensions: { code: 'FORBIDDEN' },
        });
      }

      return dataSources.productAPI.update(id, input);
    },

    deleteProduct: async (_, { id }, { user, dataSources }) => {
      if (!user || user.role !== 'ADMIN') {
        throw new GraphQLError('শুধু Admin মুছতে পারবে', {
          extensions: { code: 'FORBIDDEN' },
        });
      }

      await dataSources.productAPI.delete(id);
      return { success: true, message: 'Product সফলভাবে মুছে ফেলা হয়েছে' };
    },

    login: async (_, { email, password }, { dataSources }) => {
      const result = await dataSources.userAPI.authenticate(email, password);
      if (!result) {
        throw new GraphQLError('ইমেইল বা পাসওয়ার্ড ভুল', {
          extensions: { code: 'INVALID_CREDENTIALS' },
        });
      }
      return result;
    },
  },

  Subscription: {
    orderStatusChanged: {
      subscribe: (_, { orderId }) =>
        pubsub.asyncIterableIterator([ORDER_STATUS_CHANGED]),
      resolve: (payload) => payload.orderStatusChanged,
    },
    productAdded: {
      subscribe: (_, { categoryId }) =>
        pubsub.asyncIterableIterator([PRODUCT_ADDED]),
    },
  },

  // Field-level resolvers
  Product: {
    category: async (parent, _, { loaders }) => {
      return loaders.categoryLoader.load(parent.categoryId);
    },
    variants: async (parent, _, { dataSources }) => {
      return dataSources.variantAPI.getByProductId(parent.id);
    },
    averageRating: async (parent, _, { dataSources }) => {
      return dataSources.reviewAPI.getAverageRating(parent.id);
    },
    reviewCount: async (parent, _, { dataSources }) => {
      return dataSources.reviewAPI.getCount(parent.id);
    },
  },

  Category: {
    parent: async (cat, _, { loaders }) => {
      return cat.parentId ? loaders.categoryLoader.load(cat.parentId) : null;
    },
    children: async (cat, _, { dataSources }) => {
      return dataSources.categoryAPI.getByParent(cat.id);
    },
  },

  // Union resolver
  SearchResult: {
    __resolveType(obj) {
      if (obj.sku) return 'Product';
      if (obj.depth !== undefined) return 'Category';
      if (obj.rating !== undefined) return 'Seller';
      return null;
    },
  },
};

module.exports = resolvers;
```

#### Server Setup

```javascript
// src/index.js

const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');
const express = require('express');
const cors = require('cors');
const http = require('http');
const { WebSocketServer } = require('ws');
const { useServer } = require('graphql-ws/use/ws');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const typeDefs = require('./schema/typeDefs');
const resolvers = require('./resolvers');
const { createLoaders } = require('./loaders');
const { getUserFromToken } = require('./auth');

async function startServer() {
  const app = express();
  const httpServer = http.createServer(app);
  const schema = makeExecutableSchema({ typeDefs, resolvers });

  // WebSocket server — Subscriptions-এর জন্য
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql',
  });
  useServer({ schema }, wsServer);

  const server = new ApolloServer({
    schema,
    formatError: (formattedError, error) => {
      // Production-এ internal error hide করা
      if (process.env.NODE_ENV === 'production') {
        if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
          return { message: 'সার্ভারে সমস্যা হয়েছে', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
        }
      }
      return formattedError;
    },
  });

  await server.start();

  app.use(
    '/graphql',
    cors(),
    express.json(),
    expressMiddleware(server, {
      context: async ({ req }) => {
        const token = req.headers.authorization?.replace('Bearer ', '');
        const user = token ? await getUserFromToken(token) : null;
        return {
          user,
          loaders: createLoaders(),
          dataSources: {
            // data source instances
          },
        };
      },
    })
  );

  httpServer.listen(4000, () => {
    console.log('🚀 Server: http://localhost:4000/graphql');
    console.log('🔌 WebSocket: ws://localhost:4000/graphql');
  });
}

startServer();
```

---

## 🔥 Advanced Topics (উন্নত বিষয়সমূহ)

### ১. N+1 Problem ও DataLoader

GraphQL-এর সবচেয়ে বড় performance সমস্যা হলো **N+1 problem**। ধরুন ২০টি product আনলেন, প্রতিটির category আলাদা query-তে আনলে ১ + ২০ = ২১টি SQL query হবে!

```
সমস্যা (N+1):
  SELECT * FROM products LIMIT 20;         -- 1 query
  SELECT * FROM categories WHERE id = 1;   -- +1
  SELECT * FROM categories WHERE id = 2;   -- +1
  SELECT * FROM categories WHERE id = 1;   -- +1 (duplicate!)
  ... (মোট ২০টি আলাদা query)

সমাধান (DataLoader দিয়ে batch + cache):
  SELECT * FROM products LIMIT 20;                         -- 1 query
  SELECT * FROM categories WHERE id IN (1, 2, 3, 5, 8);   -- 1 query (batched!)
  মোট: মাত্র ২টি query!
```

#### JavaScript — DataLoader

```javascript
// src/loaders/index.js

const DataLoader = require('dataloader');
const db = require('../database');

function createLoaders() {
  return {
    categoryLoader: new DataLoader(async (categoryIds) => {
      // একবারে সব category fetch — IN clause দিয়ে
      const categories = await db('categories')
        .whereIn('id', categoryIds);

      // DataLoader-এর requirement: input order মেনে return করতে হবে
      const categoryMap = new Map(categories.map(c => [c.id, c]));
      return categoryIds.map(id => categoryMap.get(id) || null);
    }),

    productsBySellerLoader: new DataLoader(async (sellerIds) => {
      const products = await db('products')
        .whereIn('seller_id', sellerIds)
        .where('is_active', true);

      const grouped = new Map();
      for (const product of products) {
        if (!grouped.has(product.seller_id)) {
          grouped.set(product.seller_id, []);
        }
        grouped.get(product.seller_id).push(product);
      }
      return sellerIds.map(id => grouped.get(id) || []);
    }),

    reviewStatsLoader: new DataLoader(async (productIds) => {
      const stats = await db('reviews')
        .whereIn('product_id', productIds)
        .groupBy('product_id')
        .select(
          'product_id',
          db.raw('AVG(rating) as avg_rating'),
          db.raw('COUNT(*) as review_count')
        );

      const statsMap = new Map(stats.map(s => [s.product_id, s]));
      return productIds.map(id =>
        statsMap.get(id) || { avg_rating: 0, review_count: 0 }
      );
    }),
  };
}

module.exports = { createLoaders };
```

#### PHP — Lighthouse BatchLoader

```php
<?php
// app/GraphQL/Fields/AverageRatingField.php

namespace App\GraphQL\Fields;

use App\Models\Review;
use Nuwave\Lighthouse\Execution\BatchLoader\BatchLoaderRegistry;
use Illuminate\Support\Collection;

final class AverageRatingField
{
    public function __invoke($product): float
    {
        // Lighthouse-এর built-in batch loading
        // @belongsTo, @hasMany directive ব্যবহার করলে Lighthouse নিজেই batch করে
        // Custom field-এর জন্য BatchLoader ব্যবহার করা যায়

        $loader = BatchLoaderRegistry::instance(
            Review::class,
            function (Collection $productIds): array {
                return Review::whereIn('product_id', $productIds)
                    ->groupBy('product_id')
                    ->selectRaw('product_id, AVG(rating) as avg_rating')
                    ->pluck('avg_rating', 'product_id')
                    ->toArray();
            }
        );

        return $loader->load($product->id) ?? 0.0;
    }
}
```

---

### ২. Pagination — Relay Cursor-Based Connection

Relay-style cursor pagination হলো GraphQL-এর standard pagination pattern। Offset-based pagination-এর চেয়ে এটা অনেক reliable — বিশেষ করে real-time data-তে যেখানে item যোগ/মুছে যায়।

```graphql
# Connection spec অনুযায়ী schema
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ProductEdge {
  node: Product!       # actual data
  cursor: String!      # opaque cursor (base64 encoded)
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Query
query GetProducts {
  products(first: 10, after: "Y3Vyc29yXzEw") {
    edges {
      node {
        id
        name
        price
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

```javascript
// Cursor-based pagination implementation — JS

async function getProducts(filter, first, after) {
  let query = db('products').where('is_active', true);

  if (filter?.categoryId) query = query.where('category_id', filter.categoryId);
  if (filter?.minPrice) query = query.where('price', '>=', filter.minPrice);
  if (filter?.maxPrice) query = query.where('price', '<=', filter.maxPrice);
  if (filter?.inStock) query = query.where('stock', '>', 0);

  if (after) {
    const decodedCursor = Buffer.from(after, 'base64').toString('utf8');
    const [, cursorId] = decodedCursor.split('_');
    query = query.where('id', '>', parseInt(cursorId));
  }

  const totalCount = await query.clone().count('* as count').first();
  const items = await query.orderBy('id').limit(first + 1);
  const hasNextPage = items.length > first;
  const edges = items.slice(0, first).map(item => ({
    node: item,
    cursor: Buffer.from(`cursor_${item.id}`).toString('base64'),
  }));

  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!after,
      startCursor: edges[0]?.cursor || null,
      endCursor: edges[edges.length - 1]?.cursor || null,
    },
    totalCount: totalCount.count,
  };
}
```

---

### ৩. Authentication & Authorization

```graphql
# Lighthouse (PHP) — Custom Directive দিয়ে authorization

# schema.graphql-এ directive define
directive @hasRole(role: String!) on FIELD_DEFINITION

type Mutation {
  deleteProduct(id: ID!): DeleteResponse!
    @guard(with: ["sanctum"])
    @hasRole(role: "admin")
}
```

```php
<?php
// app/GraphQL/Directives/HasRoleDirective.php

namespace App\GraphQL\Directives;

use Closure;
use GraphQL\Error\Error;
use GraphQL\Type\Definition\ResolveInfo;
use Nuwave\Lighthouse\Schema\Directives\BaseDirective;
use Nuwave\Lighthouse\Schema\Values\FieldValue;
use Nuwave\Lighthouse\Support\Contracts\FieldMiddleware;
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

final class HasRoleDirective extends BaseDirective implements FieldMiddleware
{
    public static function definition(): string
    {
        return /** @lang GraphQL */ <<<GRAPHQL
            directive @hasRole(role: String!) on FIELD_DEFINITION
        GRAPHQL;
    }

    public function handleField(FieldValue $fieldValue): void
    {
        $fieldValue->wrapResolver(fn (callable $resolver) =>
            function (mixed $root, array $args, GraphQLContext $context, ResolveInfo $info) use ($resolver) {
                $requiredRole = $this->directiveArgValue('role');
                $user = $context->user();

                if (!$user || $user->role !== $requiredRole) {
                    throw new Error(
                        "আপনার '{$requiredRole}' role প্রয়োজন",
                        extensions: ['code' => 'FORBIDDEN']
                    );
                }

                return $resolver($root, $args, $context, $info);
            }
        );
    }
}
```

---

### ৪. File Upload

GraphQL-এ file upload করতে **multipart request spec** ব্যবহার হয়। এটা `graphql-upload` package দিয়ে handle করা হয়।

```graphql
# Schema
scalar Upload

type Mutation {
  uploadProductImage(
    productId: ID!
    file: Upload!
    isPrimary: Boolean! = false
  ): ProductImage!
}
```

```javascript
// JS — Apollo Server-এ file upload

const { GraphQLUpload } = require('graphql-upload-minimal');
const { createWriteStream, mkdir } = require('fs');
const path = require('path');

const resolvers = {
  Upload: GraphQLUpload,

  Mutation: {
    uploadProductImage: async (_, { productId, file, isPrimary }, { user }) => {
      const { createReadStream, filename, mimetype } = await file;

      // Validation
      const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
      if (!allowedTypes.includes(mimetype)) {
        throw new GraphQLError('শুধু JPG, PNG, WebP ফাইল আপলোড করা যাবে');
      }

      const uniqueName = `${Date.now()}-${filename}`;
      const uploadPath = path.join(__dirname, '../../uploads', uniqueName);

      await new Promise((resolve, reject) =>
        createReadStream()
          .pipe(createWriteStream(uploadPath))
          .on('finish', resolve)
          .on('error', reject)
      );

      return db('product_images').insert({
        product_id: productId,
        path: `/uploads/${uniqueName}`,
        is_primary: isPrimary,
      }).returning('*');
    },
  },
};
```

---

### ৫. Error Handling (ত্রুটি ব্যবস্থাপনা)

GraphQL-এ error handling REST থেকে ভিন্ন — HTTP status code সবসময় 200 থাকে, error গুলো response body-র `errors` array-তে আসে।

```javascript
// Custom error classes তৈরি

class ValidationError extends GraphQLError {
  constructor(message, field) {
    super(message, {
      extensions: {
        code: 'VALIDATION_ERROR',
        field,
        timestamp: new Date().toISOString(),
      },
    });
  }
}

class NotFoundError extends GraphQLError {
  constructor(resource, id) {
    super(`${resource} (ID: ${id}) পাওয়া যায়নি`, {
      extensions: {
        code: 'NOT_FOUND',
        resource,
        resourceId: id,
      },
    });
  }
}

class BusinessLogicError extends GraphQLError {
  constructor(message, details = {}) {
    super(message, {
      extensions: {
        code: 'BUSINESS_LOGIC_ERROR',
        ...details,
      },
    });
  }
}

// ব্যবহার
const resolvers = {
  Mutation: {
    placeOrder: async (_, { input }, { user, dataSources }) => {
      // Stock check
      for (const item of input.items) {
        const product = await dataSources.productAPI.getById(item.productId);
        if (!product) throw new NotFoundError('Product', item.productId);
        if (product.stock < item.quantity) {
          throw new BusinessLogicError(
            `"${product.name}" এর পর্যাপ্ত stock নেই`,
            { availableStock: product.stock, requested: item.quantity }
          );
        }
      }
      // ... order creation
    },
  },
};
```

```json
// Error response উদাহরণ
{
  "data": { "placeOrder": null },
  "errors": [
    {
      "message": "\"জামদানি শাড়ি\" এর পর্যাপ্ত stock নেই",
      "path": ["placeOrder"],
      "extensions": {
        "code": "BUSINESS_LOGIC_ERROR",
        "availableStock": 2,
        "requested": 5
      }
    }
  ]
}
```

---

### ৬. Schema Stitching & Federation (Microservices)

বড় application-এ একটি monolithic GraphQL schema maintain করা কঠিন হয়ে পড়ে। **Apollo Federation** দিয়ে schema-কে ছোট ছোট **subgraph**-এ ভাগ করা যায়, প্রতিটি আলাদা service হিসেবে চলে।

```
┌──────────────────────────────────────────────┐
│              Apollo Gateway / Router          │
│          (Single entry point: /graphql)       │
└──────────┬──────────┬──────────┬─────────────┘
           │          │          │
     ┌─────▼─────┐ ┌──▼────┐ ┌──▼──────────┐
     │ Products   │ │ Users │ │ Orders      │
     │ Subgraph   │ │ Sub.  │ │ Subgraph    │
     │ (Laravel)  │ │ (Node)│ │ (Node)      │
     └───────────┘ └───────┘ └─────────────┘
```

```graphql
# Products Subgraph — @key directive দিয়ে entity define
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
  category: Category!
}

# Orders Subgraph — Product-কে reference করে
type Order @key(fields: "id") {
  id: ID!
  items: [OrderItem!]!
  totalAmount: Float!
}

type OrderItem {
  product: Product!   # Products subgraph থেকে resolve হবে
  quantity: Int!
  unitPrice: Float!
}

# extend করে অন্য subgraph-এর type-এ field যোগ
extend type Product @key(fields: "id") {
  id: ID! @external
  orders: [Order!]!   # Orders subgraph এই field resolve করবে
}
```

---

### ৭. Persisted Queries

Client প্রতিবার পুরো query string পাঠানোর বদলে একটি **hash** পাঠায়। Server-এ আগে থেকে query registered থাকে। এতে bandwidth কমে এবং **arbitrary query attack** রোধ হয়।

```javascript
// Automatic Persisted Queries (APQ) — Apollo Server-এ built-in

const { ApolloServer } = require('@apollo/server');
const { ApolloServerPluginCacheControl } = require('@apollo/server/plugin/cacheControl');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  persistedQueries: {
    ttl: 900, // 15 minutes cache
  },
});

// Client-side (Apollo Client):
// ১ম request: hash পাঠায় → server-এ না থাকলে error
// ২য় request: hash + full query পাঠায় → server cache করে
// ৩য়+ request: শুধু hash পাঠায় → server cache থেকে serve করে
```

```
APQ Flow:
Client ──[hash only]──→ Server
       ←──[PersistedQueryNotFound]──
Client ──[hash + full query]──→ Server (stores in cache)
       ←──[response]──
Client ──[hash only]──→ Server (cache hit!)
       ←──[response]──
```

---

### ৮. Rate Limiting / Query Complexity

GraphQL-এ একটি বড় ঝুঁকি হলো কেউ অত্যন্ত **deeply nested** বা **expensive** query পাঠাতে পারে। এটা রোধ করতে **query complexity analysis** ব্যবহার হয়।

```javascript
// Query depth limiting ও cost analysis

const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(7),  // সর্বোচ্চ ৭ স্তর নেস্টিং
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
      onCost: (cost) => {
        console.log(`Query complexity: ${cost}`);
      },
    }),
  ],
});

// খারাপ query (blocked):
// {
//   categories {            depth 1, cost = 2
//     children {            depth 2, cost = 2 * 10
//       children {          depth 3, cost = 2 * 10 * 10
//         children {        depth 4, cost = 2 * 10 * 10 * 10
//           products {      depth 5, cost = 2 * 10^4
//             variants {    depth 6, cost = 2 * 10^5 ← খুব বেশি!
//               ...
//             }
//           }
//         }
//       }
//     }
//   }
// }
```

```php
<?php
// Lighthouse (PHP) — config/lighthouse.php-এ complexity setting

return [
    'security' => [
        'max_query_complexity' => 500,
        'max_query_depth' => 7,
    ],

    // Schema-তে @complexity directive ব্যবহার
    // type Query {
    //   products(first: Int!): [Product!]!
    //     @paginate
    //     @complexity(resolver: "App\\GraphQL\\Complexity\\ProductComplexity")
    // }
];
```

---

### ৯. Caching GraphQL

GraphQL caching REST-এর চেয়ে জটিল কারণ সব request একই endpoint-এ POST হিসেবে যায়, তাই HTTP cache সরাসরি কাজ করে না।

```
┌──────────────────────────────────────────────────────────────┐
│                GraphQL Caching Strategy                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ১. Client-side (Apollo Client)                              │
│     └→ InMemoryCache — normalized cache by __typename + id   │
│     └→ Query result স্বয়ংক্রিয়ভাবে cache হয়                 │
│                                                              │
│  ২. CDN / HTTP Cache (GET request + APQ ব্যবহার করলে)        │
│     └→ @cacheControl directive দিয়ে max-age set              │
│     └→ CDN query hash দিয়ে cache key তৈরি করে                │
│                                                              │
│  ৩. Server-side (Response Cache)                             │
│     └→ Redis / Memcached-এ পুরো response cache               │
│     └→ Query + variables = cache key                         │
│                                                              │
│  ৪. Data-source level                                        │
│     └→ DataLoader per-request caching (automatic)            │
│     └→ Redis cache layer (manual)                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```graphql
# Cache control directives (Apollo Server)
type Product @cacheControl(maxAge: 300) {
  id: ID!
  name: String! @cacheControl(maxAge: 3600)
  price: Float! @cacheControl(maxAge: 60)
  stock: Int! @cacheControl(maxAge: 10)
  reviews: [Review!]! @cacheControl(maxAge: 120)
}
```

```javascript
// Server-side response caching plugin

const responseCachePlugin =
  require('@apollo/server-plugin-response-cache').default;
const { KeyvAdapter } = require('@apollo/utils.keyvadapter');
const Keyv = require('keyv');
const KeyvRedis = require('@keyv/redis');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      cache: new KeyvAdapter(new Keyv({ store: new KeyvRedis('redis://localhost:6379') })),
      sessionId: (ctx) => ctx.user?.id || null, // user-specific cache
    }),
  ],
});
```

---

### ১০. Testing GraphQL

#### PHP — PHPUnit দিয়ে Lighthouse Testing

```php
<?php
// tests/Feature/GraphQL/ProductTest.php

namespace Tests\Feature\GraphQL;

use App\Models\Product;
use App\Models\Category;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Nuwave\Lighthouse\Testing\MakesGraphQLRequests;
use Tests\TestCase;

class ProductTest extends TestCase
{
    use RefreshDatabase, MakesGraphQLRequests;

    public function test_can_query_single_product(): void
    {
        $category = Category::factory()->create(['name' => 'পোশাক']);
        $product = Product::factory()->create([
            'name' => 'জামদানি শাড়ি',
            'price' => 5500.00,
            'category_id' => $category->id,
        ]);

        $response = $this->graphQL(/** @lang GraphQL */ '
            query GetProduct($id: ID!) {
                product(id: $id) {
                    id
                    name
                    price
                    category {
                        name
                    }
                }
            }
        ', ['id' => $product->id]);

        $response->assertJson([
            'data' => [
                'product' => [
                    'name' => 'জামদানি শাড়ি',
                    'price' => 5500.00,
                    'category' => ['name' => 'পোশাক'],
                ],
            ],
        ]);
    }

    public function test_can_create_product_as_authenticated_seller(): void
    {
        $user = User::factory()->create(['role' => 'seller']);
        $category = Category::factory()->create();

        $response = $this->actingAs($user)->graphQL(/** @lang GraphQL */ '
            mutation CreateProduct($input: CreateProductInput!) {
                createProduct(input: $input) {
                    id
                    name
                    price
                    sku
                }
            }
        ', [
            'input' => [
                'name' => 'টেস্ট প্রোডাক্ট',
                'price' => 999.00,
                'sku' => 'TEST-001',
                'stock' => 10,
                'category_id' => $category->id,
            ],
        ]);

        $response->assertJson([
            'data' => [
                'createProduct' => [
                    'name' => 'টেস্ট প্রোডাক্ট',
                    'price' => 999.00,
                ],
            ],
        ]);

        $this->assertDatabaseHas('products', ['sku' => 'TEST-001']);
    }

    public function test_unauthenticated_user_cannot_create_product(): void
    {
        $response = $this->graphQL(/** @lang GraphQL */ '
            mutation {
                createProduct(input: {
                    name: "Hack",
                    price: 0,
                    sku: "HACK",
                    stock: 0,
                    categoryId: "1"
                }) {
                    id
                }
            }
        ');

        $response->assertGraphQLErrorMessage('Unauthenticated.');
    }

    public function test_product_pagination(): void
    {
        $category = Category::factory()->create();
        Product::factory()->count(25)->create(['category_id' => $category->id]);

        $response = $this->graphQL(/** @lang GraphQL */ '
            query {
                products(first: 10) {
                    data { id, name }
                    paginatorInfo {
                        currentPage
                        lastPage
                        total
                        hasMorePages
                    }
                }
            }
        ');

        $response->assertJson([
            'data' => [
                'products' => [
                    'paginatorInfo' => [
                        'total' => 25,
                        'hasMorePages' => true,
                    ],
                ],
            ],
        ]);
    }
}
```

#### JavaScript — Jest দিয়ে Apollo Server Testing

```javascript
// tests/graphql/product.test.js

const { ApolloServer } = require('@apollo/server');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const typeDefs = require('../../src/schema/typeDefs');
const resolvers = require('../../src/resolvers');
const { createLoaders } = require('../../src/loaders');
const assert = require('assert');

describe('Product GraphQL API', () => {
  let server;

  beforeAll(() => {
    server = new ApolloServer({
      schema: makeExecutableSchema({ typeDefs, resolvers }),
    });
  });

  it('should fetch a product by ID', async () => {
    const result = await server.executeOperation(
      {
        query: `
          query GetProduct($id: ID!) {
            product(id: $id) {
              id
              name
              price
              category { name }
            }
          }
        `,
        variables: { id: '101' },
      },
      {
        contextValue: {
          user: null,
          loaders: createLoaders(),
          dataSources: createMockDataSources(),
        },
      }
    );

    assert.strictEqual(result.body.kind, 'single');
    expect(result.body.singleResult.errors).toBeUndefined();
    expect(result.body.singleResult.data.product).toMatchObject({
      id: '101',
      name: expect.any(String),
      price: expect.any(Number),
    });
  });

  it('should create a product when authenticated', async () => {
    const mockUser = { id: '1', role: 'seller' };

    const result = await server.executeOperation(
      {
        query: `
          mutation CreateProduct($input: CreateProductInput!) {
            createProduct(input: $input) {
              id
              name
              price
            }
          }
        `,
        variables: {
          input: {
            name: 'টেস্ট পণ্য',
            price: 500,
            sku: 'TEST-001',
            stock: 10,
            categoryId: '1',
          },
        },
      },
      { contextValue: { user: mockUser, loaders: createLoaders(), dataSources: createMockDataSources() } }
    );

    assert.strictEqual(result.body.kind, 'single');
    expect(result.body.singleResult.errors).toBeUndefined();
    expect(result.body.singleResult.data.createProduct.name).toBe('টেস্ট পণ্য');
  });

  it('should reject unauthenticated mutation', async () => {
    const result = await server.executeOperation(
      {
        query: `
          mutation { deleteProduct(id: "1") { success } }
        `,
      },
      { contextValue: { user: null, loaders: createLoaders(), dataSources: createMockDataSources() } }
    );

    assert.strictEqual(result.body.kind, 'single');
    expect(result.body.singleResult.errors).toBeDefined();
    expect(result.body.singleResult.errors[0].extensions.code).toBe('UNAUTHENTICATED');
  });
});

function createMockDataSources() {
  return {
    productAPI: {
      getById: jest.fn().mockResolvedValue({
        id: '101', name: 'Mock Product', price: 500,
        categoryId: '1', sellerId: '1',
      }),
      create: jest.fn().mockImplementation(input => ({
        id: '999', ...input,
      })),
      delete: jest.fn().mockResolvedValue(true),
    },
    categoryAPI: { search: jest.fn().mockResolvedValue([]) },
    sellerAPI: { search: jest.fn().mockResolvedValue([]) },
  };
}
```

---

## 🆚 GraphQL vs REST — বিস্তারিত তুলনা

| মানদণ্ড | REST | GraphQL |
|---|---|---|
| **Endpoint** | Resource প্রতি আলাদা (`/products`, `/users`) | একটি মাত্র `/graphql` |
| **Data Fetching** | Server নির্ধারণ করে কী আসবে | Client নির্ধারণ করে কী চাই |
| **Over-fetching** | ✗ সাধারণ সমস্যা | ✓ শুধু requested fields আসে |
| **Under-fetching** | ✗ Multiple round-trip দরকার | ✓ একটি request-এ সব আসে |
| **Type System** | Swagger/OpenAPI (optional) | Built-in strongly typed SDL |
| **Real-time** | WebSocket/SSE আলাদাভাবে | Subscription built-in |
| **Caching** | ✓ HTTP caching সহজ (GET + URL) | ✗ জটিল (POST + same URL) |
| **File Upload** | ✓ নেটিভ multipart support | ✗ Spec বাইরে, আলাদা library লাগে |
| **Error Handling** | HTTP status codes (404, 500) | সবসময় 200, errors array-তে |
| **Versioning** | URL versioning (v1, v2) | Schema evolution, deprecated fields |
| **Learning Curve** | ✓ সহজ, পরিচিত | ✗ নতুন concepts শিখতে হয় |
| **Tooling** | ✓ Postman, curl, browser | ✓ GraphiQL, Playground, Apollo Studio |
| **Performance** | ✓ সাধারণত দ্রুত (simple queries) | ✗ N+1 problem, complexity issues |
| **API Discovery** | Swagger docs দরকার | ✓ Introspection built-in |
| **Mobile Friendly** | ✗ Bandwidth waste | ✓ কম data, দ্রুত load |

---

## ✅ সুবিধা ও অসুবিধা

### সুবিধা (Pros)

```
✓ Precise Data Fetching — শুধু যা দরকার তাই আনা যায়, bandwidth বাঁচে
✓ Single Request — জটিল nested data একবারেই পাওয়া যায়
✓ Strong Typing — Schema-ই documentation, compile-time error ধরা পড়ে
✓ Introspection — API নিজেই বলে কী কী পারে, auto-complete কাজ করে
✓ Evolution without Versioning — নতুন field যোগ, পুরোনো @deprecated করলেই হয়
✓ Frontend Freedom — Backend পরিবর্তন ছাড়াই frontend নতুন data আনতে পারে
✓ Real-time — Subscription দিয়ে WebSocket built-in
✓ Ecosystem — Apollo, Relay, Lighthouse — সমৃদ্ধ tool ecosystem
```

### অসুবিধা (Cons)

```
✗ Complexity — ছোট API-র জন্য overkill
✗ N+1 Problem — DataLoader ছাড়া performance বিপর্যয়
✗ Caching কঠিন — HTTP caching সরাসরি কাজ করে না
✗ File Upload — নেটিভ support নেই, আলাদা spec follow করতে হয়
✗ Rate Limiting জটিল — endpoint-based limiting করা যায় না
✗ Security Risk — malicious deep/complex queries দিয়ে DDoS সম্ভব
✗ Learning Curve — Team-এর সবাইকে নতুন paradigm শিখতে হয়
✗ Error Handling — সবসময় HTTP 200, monitoring tool বিভ্রান্ত হতে পারে
```

---

## ⚠️ সাধারণ ভুল (Common Mistakes)

### ১. DataLoader ব্যবহার না করা
```
ভুল: প্রতিটি product-এর category আলাদা SQL query-তে আনা
    → 100 product = 101 queries!
✓ সঠিক: DataLoader দিয়ে batch করা → মাত্র 2 queries
```

### ২. Schema-তে Business Logic রাখা
```
ভুল: Schema directive দিয়ে complex validation
✓ সঠিক: Resolver/Service layer-এ business logic রাখা
```

### ৩. Query Depth/Complexity সীমা না দেওয়া
```
ভুল: যেকোনো depth-এর query allow করা
    → { a { b { c { d { e { f { g { ... } } } } } } } }
✓ সঠিক: depthLimit(7) + complexityLimit(1000) set করা
```

### ৪. সব কিছু একটি Query-তে আনার চেষ্টা
```
ভুল: একটি mega query-তে homepage-এর সব data
✓ সঠিক: Component অনুযায়ী ছোট focused query (colocation)
```

### ৫. Error Handling ignore করা
```
ভুল: Generic error message পাঠানো
✓ সঠিক: Error extensions-এ code, field, details পাঠানো
```

### ৬. Subscription অপব্যবহার
```
ভুল: সব data change subscription দিয়ে আনা
✓ সঠিক: শুধু real-time জরুরি feature-এ subscription ব্যবহার
   (যেমন: order tracking, chat, live notification)
```

### ৭. REST মানসিকতায় GraphQL করা
```
ভুল: getProductById, getAllProducts — REST-style naming
✓ সঠিক: product(id), products(filter) — GraphQL-idiomatic naming
```

### ৮. Production-এ Introspection চালু রাখা
```
ভুল: Production server-এ introspection enable
    → Attacker পুরো schema দেখতে পারে
✓ সঠিক: Production-এ introspection: false set করা
```

---

## 📋 সারসংক্ষেপ (Summary)

```
┌─────────────────────────────────────────────────────────────────┐
│                    GraphQL সারসংক্ষেপ                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ◆ কী: API-এর জন্য query language + runtime (Facebook, 2015)   │
│                                                                 │
│  ◆ কেন: Over/under-fetching সমাধান, single endpoint,           │
│         strongly typed, real-time support                       │
│                                                                 │
│  ◆ Core: Schema (SDL), Query, Mutation, Subscription, Resolver │
│                                                                 │
│  ◆ PHP Stack: Laravel + Lighthouse (schema-first, directives)  │
│  ◆ JS Stack: Apollo Server + Express (code-first/schema-first) │
│                                                                 │
│  ◆ Must-do:                                                     │
│    • DataLoader দিয়ে N+1 সমাধান                                 │
│    • Query depth + complexity limit                             │
│    • Cursor-based pagination (Relay spec)                       │
│    • Proper error handling with extensions                      │
│    • Production-এ introspection বন্ধ                             │
│    • Persisted queries (security + performance)                 │
│                                                                 │
│  ◆ কখন ব্যবহার করবেন:                                           │
│    ✓ Complex, nested data (e-commerce product catalog)          │
│    ✓ Multiple client (web, mobile, 3rd party)                   │
│    ✓ Rapidly evolving frontend                                  │
│    ✓ Real-time features দরকার                                   │
│                                                                 │
│  ◆ কখন ব্যবহার করবেন না:                                        │
│    ✗ Simple CRUD API                                            │
│    ✗ File-heavy application                                     │
│    ✗ ছোট team, tight deadline                                   │
│    ✗ শুধু server-to-server communication                        │
│                                                                 │
│  ◆ বাংলাদেশ Context:                                            │
│    • ই-কমার্স: Product → Variants → Categories (nested)         │
│    • Payment: bKash/Nagad/Rocket integration                    │
│    • Mobile-first: কম bandwidth, precise data fetching জরুরি    │
│    • Multi-vendor marketplace: Federation দিয়ে microservices    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

> **"GraphQL হলো client-driven API design — server বলে কী আছে, client বলে কী চাই।"**
> এটা REST-কে replace করে না, বরং সঠিক use case-এ সঠিক tool ব্যবহার করাই প্রকৃত প্রজ্ঞা।
