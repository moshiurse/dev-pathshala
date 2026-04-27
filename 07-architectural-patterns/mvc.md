# 🏗️ MVC (Model-View-Controller) প্যাটার্ন

## 📌 সংজ্ঞা ও মূল ধারণা

**MVC (Model-View-Controller)** হলো একটি architectural design pattern যা একটি application-কে তিনটি interconnected component-এ ভাগ করে — **Model**, **View**, এবং **Controller**। এই separation of concerns নীতি অনুসরণ করে application-এর business logic, user interface, এবং input handling আলাদা আলাদা ভাবে manage করা হয়।

### 🕰️ ইতিহাস

১৯৭৯ সালে নরওয়েজিয়ান কম্পিউটার বিজ্ঞানী **Trygve Reenskaug** Xerox PARC-এ কাজ করার সময় MVC pattern-টি প্রথম প্রস্তাব করেন। তিনি Smalltalk-76 programming language-এ GUI application তৈরির সময় এই ধারণাটি উদ্ভাবন করেন। মূল লক্ষ্য ছিল user-এর mental model এবং digital model-এর মধ্যে একটি সেতুবন্ধন তৈরি করা। পরবর্তীতে এটি web development-এর সবচেয়ে জনপ্রিয় architectural pattern হয়ে ওঠে — Ruby on Rails, Laravel, Django, ASP.NET MVC, Spring MVC সবই এই pattern অনুসরণ করে।

### তিনটি কম্পোনেন্ট বিস্তারিত

#### 🟦 Model (মডেল)
Model হলো application-এর **data layer** এবং **business logic** এর ধারক। এটি:
- Database-এর সাথে সরাসরি যোগাযোগ করে (CRUD operations)
- Data validation নিশ্চিত করে
- Business rules enforce করে
- Data relationships define করে (One-to-Many, Many-to-Many ইত্যাদি)
- অন্য component-কে data change সম্পর্কে notify করতে পারে (Observer pattern)

#### 🟩 View (ভিউ)
View হলো **presentation layer** — user যা দেখে এবং যার সাথে interact করে:
- Model থেকে data নিয়ে visual representation তৈরি করে
- HTML, CSS, JS (web), বা XML/JSON (API) format-এ output দেয়
- কোনো business logic থাকা উচিত না — শুধু display logic
- একটি Model-এর জন্য multiple View থাকতে পারে (mobile, desktop, API)

#### 🟧 Controller (কন্ট্রোলার)
Controller হলো **intermediary** — Model ও View-এর মধ্যে সমন্বয়কারী:
- User-এর input/request receive করে
- Input validate এবং sanitize করে
- প্রয়োজনীয় Model method call করে
- সঠিক View select করে এবং data pass করে
- HTTP request/response cycle manage করে

---

## 🏠 বাস্তব জীবনের উদাহরণ (Analogy)

### 🍽️ রেস্তোরাঁ Analogy

কল্পনা করুন আপনি ঢাকার একটি রেস্তোরাঁয় গেছেন:

```
👤 Customer (User/Browser)
    │
    ▼
🧑‍🍳 Waiter (Controller)
    │   - আপনার order নেয় (receives request)
    │   - order কিচেনে পাঠায় (calls Model)
    │   - খাবার সাজিয়ে আপনার কাছে আনে (returns View)
    │
    ▼
🍳 Kitchen (Model)
    │   - রেসিপি জানে (business logic)
    │   - ingredients manage করে (database)
    │   - খাবার তৈরি করে (data processing)
    │
    ▼
🍽️ Plate/Menu (View)
        - খাবার সুন্দরভাবে সাজানো (presentation)
        - Menu card-এ তথ্য দেখানো (data display)
        - বিভিন্ন plate style (multiple views)
```

**বাংলাদেশী উদাহরণ — bKash Transaction:**

```
📱 bKash App (User Interface)
    │
    ▼
🔄 Transaction Controller
    │   - User PIN verify করে
    │   - Transaction request process করে
    │   - Response display করে
    │
    ▼
💰 Transaction Model
    │   - Balance check করে
    │   - Transaction limit validate করে
    │   - Money transfer execute করে
    │   - Transaction log রাখে
    │
    ▼
📊 Confirmation View
        - Transaction success/failure দেখায়
        - Transaction ID, amount, reference দেখায়
```

---

## 📊 আর্কিটেকচার ডায়াগ্রাম (ASCII)

### Request Flow Diagram

```
                        ┌─────────────────────────────────────────┐
                        │              MVC Architecture            │
                        └─────────────────────────────────────────┘

    ┌──────────┐     ┌──────────────┐     ┌───────────┐     ┌──────────┐
    │          │────▶│              │────▶│           │────▶│          │
    │  Client  │     │  Controller  │     │   Model   │     │ Database │
    │ (Browser)│◀────│              │◀────│           │◀────│          │
    │          │     │              │     │           │     │          │
    └──────────┘     └──────┬───────┘     └───────────┘     └──────────┘
         ▲                  │
         │                  ▼
         │           ┌──────────────┐
         │           │              │
         └───────────│    View      │
                     │  (Template)  │
                     │              │
                     └──────────────┘
```

### Component Interaction Diagram (বিস্তারিত)

```
    ┌─────────────────── HTTP Request ──────────────────────┐
    │                                                        │
    ▼                                                        │
┌────────┐    ┌────────────┐    ┌────────────┐    ┌────────┐│
│ Router │───▶│ Middleware  │───▶│ Controller │    │ Client ││
│        │    │ (Auth,CORS) │    │            │    │        ││
└────────┘    └────────────┘    └─────┬──────┘    └────────┘│
                                      │                      │
                          ┌───────────┼───────────┐          │
                          ▼           ▼           ▼          │
                    ┌──────────┐ ┌─────────┐ ┌────────┐     │
                    │ Service  │ │ Request │ │ Events │     │
                    │  Layer   │ │Validate │ │Observer│     │
                    └────┬─────┘ └─────────┘ └────────┘     │
                         │                                    │
                         ▼                                    │
                    ┌──────────┐    ┌────────────┐           │
                    │  Model/  │───▶│  Database   │           │
                    │Repository│◀───│  (MySQL/    │           │
                    │          │    │   MongoDB)  │           │
                    └────┬─────┘    └────────────┘           │
                         │                                    │
                         ▼                                    │
                    ┌──────────┐    ┌────────────┐           │
                    │  View/   │───▶│  Response   │───────────┘
                    │ Resource │    │ (HTML/JSON) │
                    └──────────┘    └────────────┘
```

---

## 💻 বেসিক MVC ইমপ্লিমেন্টেশন

### 🐘 PHP (Laravel) উদাহরণ

#### Model — `Product.php`

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletes;

// Product Model — bKash-এর মতো e-commerce পণ্য পরিচালনার জন্য
class Product extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'slug',
        'price',
        'discount_price',
        'category_id',
        'stock_quantity',
        'status',
    ];

    protected $casts = [
        'price'          => 'decimal:2',
        'discount_price' => 'decimal:2',
        'stock_quantity'  => 'integer',
        'status'         => ProductStatus::class, // PHP 8.1+ Enum casting
    ];

    // ──────────────── Relationships ────────────────

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function orders(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function reviews(): HasMany
    {
        return $this->hasMany(Review::class);
    }

    // ──────────────── Accessors (Laravel 10+ syntax) ────────────────

    protected function formattedPrice(): Attribute
    {
        return Attribute::make(
            get: fn () => '৳' . number_format((float) $this->price, 2),
        );
    }

    protected function discountPercentage(): Attribute
    {
        return Attribute::make(
            get: function () {
                if (!$this->discount_price) {
                    return 0;
                }
                return round(
                    (($this->price - $this->discount_price) / $this->price) * 100
                );
            },
        );
    }

    // ──────────────── Mutators ────────────────

    protected function name(): Attribute
    {
        return Attribute::make(
            set: fn (string $value) => mb_strtolower($value, 'UTF-8'),
        );
    }

    // ──────────────── Query Scopes ────────────────

    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', ProductStatus::Active);
    }

    public function scopeInStock(Builder $query): Builder
    {
        return $query->where('stock_quantity', '>', 0);
    }

    public function scopePriceRange(Builder $query, float $min, float $max): Builder
    {
        return $query->whereBetween('price', [$min, $max]);
    }

    public function scopeByCategory(Builder $query, int $categoryId): Builder
    {
        return $query->where('category_id', $categoryId);
    }

    // ──────────────── Business Methods ────────────────

    public function isAvailable(): bool
    {
        return $this->status === ProductStatus::Active && $this->stock_quantity > 0;
    }

    public function reduceStock(int $quantity): void
    {
        if ($quantity > $this->stock_quantity) {
            throw new \DomainException('পর্যাপ্ত স্টক নেই।');
        }
        $this->decrement('stock_quantity', $quantity);
    }
}

// PHP 8.1+ Enum — Product Status
enum ProductStatus: string
{
    case Active       = 'active';
    case Inactive     = 'inactive';
    case OutOfStock   = 'out_of_stock';
    case Discontinued = 'discontinued';

    public function label(): string
    {
        return match ($this) {
            self::Active       => 'সক্রিয়',
            self::Inactive     => 'নিষ্ক্রিয়',
            self::OutOfStock   => 'স্টক শেষ',
            self::Discontinued => 'বন্ধ',
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::Active       => 'green',
            self::Inactive     => 'gray',
            self::OutOfStock   => 'red',
            self::Discontinued => 'black',
        };
    }
}
```

#### Controller — `ProductController.php`

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\StoreProductRequest;
use App\Http\Requests\UpdateProductRequest;
use App\Http\Resources\ProductResource;
use App\Http\Resources\ProductCollection;
use App\Models\Product;
use App\Services\ProductService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class ProductController extends Controller
{
    // Constructor Promotion (PHP 8.0+)
    public function __construct(
        private readonly ProductService $productService,
    ) {}

    // ── Web Routes (View return) ──

    public function index(Request $request): View
    {
        $products = Product::query()
            ->active()
            ->inStock()
            ->when(
                $request->filled('category'),
                fn ($q) => $q->byCategory((int) $request->category)
            )
            ->when(
                $request->filled('min_price') && $request->filled('max_price'),
                fn ($q) => $q->priceRange(
                    (float) $request->min_price,
                    (float) $request->max_price,
                )
            )
            ->with(['category', 'reviews'])
            ->latest()
            ->paginate(perPage: 15);

        return view('products.index', compact('products'));
    }

    public function show(Product $product): View
    {
        $product->load(['category', 'reviews.user', 'orders']);
        $relatedProducts = $this->productService->getRelated($product, limit: 4);

        return view('products.show', compact('product', 'relatedProducts'));
    }

    public function store(StoreProductRequest $request): RedirectResponse
    {
        $product = $this->productService->create($request->validated());

        return redirect()
            ->route('products.show', $product)
            ->with('success', 'পণ্য সফলভাবে তৈরি হয়েছে।');
    }

    public function update(UpdateProductRequest $request, Product $product): RedirectResponse
    {
        $this->productService->update($product, $request->validated());

        return redirect()
            ->route('products.show', $product)
            ->with('success', 'পণ্য সফলভাবে আপডেট হয়েছে।');
    }

    public function destroy(Product $product): RedirectResponse
    {
        $this->productService->delete($product);

        return redirect()
            ->route('products.index')
            ->with('success', 'পণ্য সফলভাবে মুছে ফেলা হয়েছে।');
    }

    // ── API Routes (JSON return) ──

    public function apiIndex(Request $request): ProductCollection
    {
        $products = Product::query()
            ->active()
            ->inStock()
            ->paginate(perPage: $request->integer('per_page', 15));

        return new ProductCollection($products);
    }

    public function apiShow(Product $product): ProductResource
    {
        return new ProductResource($product->load('category', 'reviews'));
    }

    public function apiStore(StoreProductRequest $request): JsonResponse
    {
        $product = $this->productService->create($request->validated());

        return (new ProductResource($product))
            ->response()
            ->setStatusCode(201);
    }
}
```

#### Form Request — `StoreProductRequest.php`

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Models\ProductStatus;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Enum;

class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create-product');
    }

    /** @return array<string, mixed> */
    public function rules(): array
    {
        return [
            'name'           => ['required', 'string', 'max:255'],
            'slug'           => ['required', 'string', 'unique:products,slug'],
            'price'          => ['required', 'numeric', 'min:0'],
            'discount_price' => ['nullable', 'numeric', 'lt:price'],
            'category_id'    => ['required', 'exists:categories,id'],
            'stock_quantity'  => ['required', 'integer', 'min:0'],
            'status'         => ['required', new Enum(ProductStatus::class)],
        ];
    }

    /** @return array<string, string> */
    public function messages(): array
    {
        return [
            'name.required'       => 'পণ্যের নাম আবশ্যক।',
            'price.required'      => 'মূল্য আবশ্যক।',
            'price.min'           => 'মূল্য ০ এর কম হতে পারে না।',
            'discount_price.lt'   => 'ডিসকাউন্ট মূল্য মূল মূল্যের চেয়ে কম হতে হবে।',
            'stock_quantity.min'   => 'স্টক পরিমাণ ঋণাত্মক হতে পারে না।',
        ];
    }
}
```

#### View — `products/index.blade.php`

```blade
{{-- resources/views/products/index.blade.php --}}
@extends('layouts.app')

@section('title', 'পণ্য তালিকা')

@section('content')
<div class="container mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold mb-6">🛍️ পণ্য তালিকা</h1>

    {{-- ফিল্টার কম্পোনেন্ট --}}
    <x-product-filter :categories="$categories" />

    {{-- প্রোডাক্ট গ্রিড --}}
    <div class="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-6">
        @forelse($products as $product)
            <x-product-card :product="$product" />
        @empty
            <div class="col-span-full text-center py-12">
                <p class="text-gray-500 text-lg">কোনো পণ্য পাওয়া যায়নি।</p>
            </div>
        @endforelse
    </div>

    {{-- পেজিনেশন --}}
    <div class="mt-8">
        {{ $products->withQueryString()->links() }}
    </div>
</div>
@endsection
```

#### Blade Component — `product-card.blade.php`

```blade
{{-- resources/views/components/product-card.blade.php --}}
@props(['product'])

<div class="bg-white rounded-lg shadow-md overflow-hidden hover:shadow-lg transition">
    <img src="{{ $product->image_url }}" alt="{{ $product->name }}" class="w-full h-48 object-cover">

    <div class="p-4">
        <span class="inline-block px-2 py-1 text-xs rounded"
              style="background-color: {{ $product->status->color() }}20;
                     color: {{ $product->status->color() }}">
            {{ $product->status->label() }}
        </span>

        <h3 class="mt-2 font-semibold text-lg">{{ $product->name }}</h3>
        <p class="text-sm text-gray-600">{{ $product->category->name }}</p>

        <div class="mt-3 flex items-center justify-between">
            @if($product->discount_price)
                <div>
                    <span class="text-red-500 font-bold">৳{{ number_format($product->discount_price, 2) }}</span>
                    <span class="text-gray-400 line-through text-sm ml-1">{{ $product->formatted_price }}</span>
                    <span class="text-green-600 text-xs ml-1">{{ $product->discount_percentage }}% ছাড়</span>
                </div>
            @else
                <span class="text-gray-800 font-bold">{{ $product->formatted_price }}</span>
            @endif
        </div>

        <a href="{{ route('products.show', $product) }}"
           class="mt-3 block text-center bg-blue-600 text-white py-2 rounded hover:bg-blue-700">
            বিস্তারিত দেখুন
        </a>
    </div>
</div>
```

#### Routes — `web.php` ও `api.php`

```php
<?php
// routes/web.php
use App\Http\Controllers\ProductController;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::resource('products', ProductController::class);
});

// routes/api.php
Route::prefix('v1')->middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::get('/products', [ProductController::class, 'apiIndex']);
    Route::get('/products/{product}', [ProductController::class, 'apiShow']);
    Route::post('/products', [ProductController::class, 'apiStore']);
});
```

---

### 🟡 JavaScript (Express + Mongoose) উদাহরণ

#### Model — `models/Product.js`

```javascript
// models/Product.js
import mongoose from 'mongoose';

// Product Status Enum (JS-তে enum নেই, object freeze ব্যবহার)
export const ProductStatus = Object.freeze({
    Active: 'active',
    Inactive: 'inactive',
    OutOfStock: 'out_of_stock',
    Discontinued: 'discontinued',

    label(status) {
        const labels = {
            active: 'সক্রিয়',
            inactive: 'নিষ্ক্রিয়',
            out_of_stock: 'স্টক শেষ',
            discontinued: 'বন্ধ',
        };
        return labels[status] ?? 'অজানা';
    },
});

const productSchema = new mongoose.Schema(
    {
        name: {
            type: String,
            required: [true, 'পণ্যের নাম আবশ্যক'],
            trim: true,
            maxlength: 255,
            set: (v) => v.toLowerCase(),
        },
        slug: {
            type: String,
            required: true,
            unique: true,
            lowercase: true,
        },
        price: {
            type: Number,
            required: [true, 'মূল্য আবশ্যক'],
            min: [0, 'মূল্য ০ এর কম হতে পারে না'],
        },
        discountPrice: {
            type: Number,
            default: null,
            validate: {
                validator(value) {
                    return value === null || value < this.price;
                },
                message: 'ডিসকাউন্ট মূল্য মূল মূল্যের চেয়ে কম হতে হবে',
            },
        },
        category: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'Category',
            required: true,
        },
        stockQuantity: {
            type: Number,
            required: true,
            min: 0,
            default: 0,
        },
        status: {
            type: String,
            enum: Object.values(ProductStatus).filter(v => typeof v === 'string'),
            default: ProductStatus.Active,
        },
    },
    {
        timestamps: true,
        toJSON: { virtuals: true },
        toObject: { virtuals: true },
    }
);

// ──────────────── Virtuals ────────────────

productSchema.virtual('formattedPrice').get(function () {
    return `৳${this.price.toFixed(2)}`;
});

productSchema.virtual('discountPercentage').get(function () {
    if (!this.discountPrice) return 0;
    return Math.round(((this.price - this.discountPrice) / this.price) * 100);
});

productSchema.virtual('reviews', {
    ref: 'Review',
    localField: '_id',
    foreignField: 'product',
});

// ──────────────── Static Methods (Query Scopes) ────────────────

productSchema.statics.findActive = function () {
    return this.find({ status: ProductStatus.Active });
};

productSchema.statics.findInStock = function () {
    return this.find({ status: ProductStatus.Active, stockQuantity: { $gt: 0 } });
};

productSchema.statics.findByPriceRange = function (min, max) {
    return this.find({ price: { $gte: min, $lte: max } });
};

// ──────────────── Instance Methods ────────────────

productSchema.methods.isAvailable = function () {
    return this.status === ProductStatus.Active && this.stockQuantity > 0;
};

productSchema.methods.reduceStock = async function (quantity) {
    if (quantity > this.stockQuantity) {
        throw new Error('পর্যাপ্ত স্টক নেই।');
    }
    this.stockQuantity -= quantity;
    await this.save();
};

// ──────────────── Indexes ────────────────

productSchema.index({ slug: 1 });
productSchema.index({ status: 1, stockQuantity: 1 });
productSchema.index({ price: 1 });
productSchema.index({ category: 1 });

const Product = mongoose.model('Product', productSchema);
export default Product;
```

#### Controller — `controllers/productController.js`

```javascript
// controllers/productController.js
import Product, { ProductStatus } from '../models/Product.js';
import ProductService from '../services/ProductService.js';
import { AppError } from '../utils/errors.js';

class ProductController {
    #productService;

    constructor() {
        this.#productService = new ProductService();
    }

    // ── Web Routes (HTML render) ──

    index = async (req, res, next) => {
        try {
            const { category, min_price, max_price, page = 1 } = req.query;
            const limit = 15;
            const skip = (page - 1) * limit;

            let query = Product.findInStock();

            if (category) {
                query = query.where('category').equals(category);
            }
            if (min_price && max_price) {
                query = query.where('price').gte(+min_price).lte(+max_price);
            }

            const [products, total] = await Promise.all([
                query.populate('category').sort('-createdAt').skip(skip).limit(limit),
                Product.countDocuments(query.getFilter()),
            ]);

            res.render('products/index', {
                products,
                pagination: {
                    page: +page,
                    totalPages: Math.ceil(total / limit),
                    total,
                },
            });
        } catch (error) {
            next(error);
        }
    };

    show = async (req, res, next) => {
        try {
            const product = await Product.findById(req.params.id)
                .populate('category')
                .populate('reviews');

            if (!product) {
                throw new AppError('পণ্য পাওয়া যায়নি', 404);
            }

            const relatedProducts = await this.#productService.getRelated(product, 4);

            res.render('products/show', { product, relatedProducts });
        } catch (error) {
            next(error);
        }
    };

    store = async (req, res, next) => {
        try {
            const product = await this.#productService.create(req.body);
            res.redirect(`/products/${product._id}`);
        } catch (error) {
            next(error);
        }
    };

    // ── API Routes (JSON) ──

    apiIndex = async (req, res, next) => {
        try {
            const { page = 1, per_page = 15 } = req.query;
            const skip = (page - 1) * per_page;

            const [products, total] = await Promise.all([
                Product.findInStock()
                    .populate('category')
                    .skip(skip)
                    .limit(+per_page),
                Product.countDocuments({ status: ProductStatus.Active }),
            ]);

            res.json({
                data: products,
                meta: {
                    page: +page,
                    per_page: +per_page,
                    total,
                    total_pages: Math.ceil(total / per_page),
                },
            });
        } catch (error) {
            next(error);
        }
    };

    apiStore = async (req, res, next) => {
        try {
            const product = await this.#productService.create(req.body);
            res.status(201).json({ data: product });
        } catch (error) {
            next(error);
        }
    };
}

export default new ProductController();
```

#### View — `views/products/index.ejs`

```ejs
<!-- views/products/index.ejs -->
<!DOCTYPE html>
<html lang="bn">
<head>
    <meta charset="UTF-8">
    <title>পণ্য তালিকা</title>
</head>
<body>
    <div class="container">
        <h1>🛍️ পণ্য তালিকা</h1>

        <div class="product-grid">
            <% if (products.length === 0) { %>
                <p class="empty">কোনো পণ্য পাওয়া যায়নি।</p>
            <% } else { %>
                <% products.forEach(product => { %>
                    <div class="product-card">
                        <h3><%= product.name %></h3>
                        <p class="category"><%= product.category?.name %></p>
                        <div class="price">
                            <% if (product.discountPrice) { %>
                                <span class="discount">৳<%= product.discountPrice.toFixed(2) %></span>
                                <span class="original"><%= product.formattedPrice %></span>
                                <span class="badge"><%= product.discountPercentage %>% ছাড়</span>
                            <% } else { %>
                                <span><%= product.formattedPrice %></span>
                            <% } %>
                        </div>
                        <a href="/products/<%= product._id %>">বিস্তারিত দেখুন</a>
                    </div>
                <% }); %>
            <% } %>
        </div>

        <%- include('../partials/pagination', { pagination }) %>
    </div>
</body>
</html>
```

#### Routes — `routes/productRoutes.js`

```javascript
// routes/productRoutes.js
import { Router } from 'express';
import productController from '../controllers/productController.js';
import { authenticate } from '../middleware/auth.js';
import { validate } from '../middleware/validate.js';
import { productSchema } from '../validators/productValidator.js';

const router = Router();

// Web Routes
router.get('/products', productController.index);
router.get('/products/:id', productController.show);
router.post('/products', authenticate, validate(productSchema), productController.store);

// API Routes
const apiRouter = Router();
apiRouter.get('/products', productController.apiIndex);
apiRouter.post('/products', authenticate, validate(productSchema), productController.apiStore);

export { router as webRoutes, apiRouter as apiRoutes };
```

#### App Entry — `app.js`

```javascript
// app.js
import express from 'express';
import { webRoutes, apiRoutes } from './routes/productRoutes.js';
import { errorHandler } from './middleware/errorHandler.js';

const app = express();

app.set('view engine', 'ejs');
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use('/', webRoutes);
app.use('/api/v1', apiRoutes);

app.use(errorHandler);

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 🔥 Advanced Complex Scenarios

### ১. Service Layer Pattern সাথে MVC

#### সমস্যা: Fat Controller / Fat Model

যখন সব business logic Controller-এ রাখা হয় তখন তাকে **Fat Controller** বলে। আবার সব Model-এ রাখলে হয় **Fat Model**। উভয়ই maintenance nightmare তৈরি করে। সমাধান হলো **Service Layer** — business logic কে আলাদা class-এ রাখা।

```
Controller (thin) → Service (business logic) → Model/Repository (data access)
```

#### PHP — `ProductService.php`

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Product;
use App\Models\ProductStatus;
use App\Events\ProductCreated;
use App\Events\ProductStockLow;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Str;

final class ProductService
{
    private const LOW_STOCK_THRESHOLD = 5;

    public function __construct(
        private readonly Product $product,
    ) {}

    /** @param array<string, mixed> $data */
    public function create(array $data): Product
    {
        return DB::transaction(function () use ($data) {
            $data['slug'] = $data['slug'] ?? Str::slug($data['name']);

            $product = $this->product->create($data);

            // Event fire — Observer প্যাটার্ন
            event(new ProductCreated($product));

            // Cache invalidate
            Cache::tags(['products'])->flush();

            return $product->load('category');
        });
    }

    /** @param array<string, mixed> $data */
    public function update(Product $product, array $data): Product
    {
        return DB::transaction(function () use ($product, $data) {
            $product->update($data);

            if ($product->stock_quantity <= self::LOW_STOCK_THRESHOLD) {
                event(new ProductStockLow($product));
            }

            Cache::tags(['products'])->flush();

            return $product->fresh(['category']);
        });
    }

    public function delete(Product $product): void
    {
        DB::transaction(function () use ($product) {
            $product->delete(); // Soft delete
            Cache::tags(['products'])->flush();
        });
    }

    /** @return \Illuminate\Database\Eloquent\Collection<int, Product> */
    public function getRelated(Product $product, int $limit = 4)
    {
        return Product::query()
            ->active()
            ->inStock()
            ->where('category_id', $product->category_id)
            ->where('id', '!=', $product->id)
            ->limit($limit)
            ->get();
    }

    public function processOrder(Product $product, int $quantity, string $userId): array
    {
        return DB::transaction(function () use ($product, $quantity, $userId) {
            // Stock lock — race condition এড়ানোর জন্য pessimistic locking
            $product = Product::query()->lockForUpdate()->find($product->id);

            if (!$product->isAvailable()) {
                throw new \DomainException('পণ্যটি বর্তমানে উপলব্ধ নয়।');
            }

            $product->reduceStock($quantity);

            $totalPrice = ($product->discount_price ?? $product->price) * $quantity;

            if ($product->stock_quantity <= self::LOW_STOCK_THRESHOLD) {
                event(new ProductStockLow($product));
            }

            return [
                'product_id'  => $product->id,
                'quantity'    => $quantity,
                'unit_price'  => $product->discount_price ?? $product->price,
                'total_price' => $totalPrice,
                'user_id'     => $userId,
            ];
        });
    }
}
```

#### JavaScript — `services/ProductService.js`

```javascript
// services/ProductService.js
import Product, { ProductStatus } from '../models/Product.js';
import { EventBus } from '../events/EventBus.js';
import { CacheManager } from '../utils/cache.js';
import mongoose from 'mongoose';

class ProductService {
    #cache;
    #events;

    static LOW_STOCK_THRESHOLD = 5;

    constructor() {
        this.#cache = new CacheManager('products');
        this.#events = EventBus.getInstance();
    }

    async create(data) {
        const session = await mongoose.startSession();
        session.startTransaction();

        try {
            const product = await Product.create([data], { session });

            this.#events.emit('product:created', product[0]);
            await this.#cache.invalidateTag('products');

            await session.commitTransaction();
            return product[0].populate('category');
        } catch (error) {
            await session.abortTransaction();
            throw error;
        } finally {
            session.endSession();
        }
    }

    async update(productId, data) {
        const session = await mongoose.startSession();
        session.startTransaction();

        try {
            const product = await Product.findByIdAndUpdate(productId, data, {
                new: true,
                runValidators: true,
                session,
            });

            if (!product) throw new Error('পণ্য পাওয়া যায়নি');

            if (product.stockQuantity <= ProductService.LOW_STOCK_THRESHOLD) {
                this.#events.emit('product:low-stock', product);
            }

            await this.#cache.invalidateTag('products');
            await session.commitTransaction();

            return product;
        } catch (error) {
            await session.abortTransaction();
            throw error;
        } finally {
            session.endSession();
        }
    }

    async getRelated(product, limit = 4) {
        const cacheKey = `related:${product._id}:${limit}`;
        const cached = await this.#cache.get(cacheKey);
        if (cached) return cached;

        const related = await Product.findInStock()
            .where('category').equals(product.category)
            .where('_id').ne(product._id)
            .limit(limit);

        await this.#cache.set(cacheKey, related, 3600);
        return related;
    }

    async processOrder(productId, quantity, userId) {
        const session = await mongoose.startSession();
        session.startTransaction();

        try {
            // Atomic update — race condition এড়ানো
            const product = await Product.findOneAndUpdate(
                {
                    _id: productId,
                    status: ProductStatus.Active,
                    stockQuantity: { $gte: quantity },
                },
                { $inc: { stockQuantity: -quantity } },
                { new: true, session }
            );

            if (!product) {
                throw new Error('পণ্যটি উপলব্ধ নেই অথবা পর্যাপ্ত স্টক নেই।');
            }

            const unitPrice = product.discountPrice ?? product.price;
            const result = {
                productId: product._id,
                quantity,
                unitPrice,
                totalPrice: unitPrice * quantity,
                userId,
            };

            if (product.stockQuantity <= ProductService.LOW_STOCK_THRESHOLD) {
                this.#events.emit('product:low-stock', product);
            }

            await session.commitTransaction();
            return result;
        } catch (error) {
            await session.abortTransaction();
            throw error;
        } finally {
            session.endSession();
        }
    }
}

export default ProductService;
```

---

### ২. Repository Pattern সাথে MVC

Repository Pattern ব্যবহার করলে Controller সরাসরি Model access করে না। এতে data source পরিবর্তন করা সহজ হয় (যেমন MySQL থেকে MongoDB, বা API থেকে cache)।

```
Controller → Service → Repository → Model → Database
```

#### PHP — `ProductRepository.php`

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Models\Product;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;

interface ProductRepositoryInterface
{
    public function findById(int $id): ?Product;
    public function paginate(array $filters, int $perPage): LengthAwarePaginator;
    public function create(array $data): Product;
    public function update(Product $product, array $data): Product;
    public function delete(Product $product): void;
}

final class ProductRepository implements ProductRepositoryInterface
{
    public function __construct(
        private readonly Product $model,
    ) {}

    public function findById(int $id): ?Product
    {
        return $this->model->with(['category', 'reviews'])->find($id);
    }

    /** @param array{category_id?: int, min_price?: float, max_price?: float, status?: string} $filters */
    public function paginate(array $filters = [], int $perPage = 15): LengthAwarePaginator
    {
        return $this->model->query()
            ->active()
            ->inStock()
            ->when(
                isset($filters['category_id']),
                fn ($q) => $q->byCategory($filters['category_id'])
            )
            ->when(
                isset($filters['min_price'], $filters['max_price']),
                fn ($q) => $q->priceRange($filters['min_price'], $filters['max_price'])
            )
            ->with('category')
            ->latest()
            ->paginate($perPage);
    }

    public function create(array $data): Product
    {
        return $this->model->create($data);
    }

    public function update(Product $product, array $data): Product
    {
        $product->update($data);
        return $product->fresh();
    }

    public function delete(Product $product): void
    {
        $product->delete();
    }
}
```

#### JavaScript — `repositories/ProductRepository.js`

```javascript
// repositories/ProductRepository.js
import Product, { ProductStatus } from '../models/Product.js';

class ProductRepository {
    async findById(id) {
        return Product.findById(id).populate('category').populate('reviews');
    }

    async paginate(filters = {}, page = 1, perPage = 15) {
        const query = Product.findInStock();

        if (filters.category) {
            query.where('category').equals(filters.category);
        }
        if (filters.minPrice !== undefined && filters.maxPrice !== undefined) {
            query.where('price').gte(filters.minPrice).lte(filters.maxPrice);
        }

        const skip = (page - 1) * perPage;
        const [data, total] = await Promise.all([
            query.populate('category').sort('-createdAt').skip(skip).limit(perPage),
            Product.countDocuments(query.getFilter()),
        ]);

        return { data, total, page, perPage, totalPages: Math.ceil(total / perPage) };
    }

    async create(data) {
        return Product.create(data);
    }

    async update(id, data) {
        return Product.findByIdAndUpdate(id, data, { new: true, runValidators: true });
    }

    async delete(id) {
        return Product.findByIdAndDelete(id);
    }
}

export default ProductRepository;
```

---

### ৩. MVC তে Event Handling

Event-driven architecture MVC-কে আরও decoupled করে। Model-এ কিছু ঘটলে (create, update, delete) event fire হয় এবং listener সেটি handle করে।

#### PHP — Laravel Observer

```php
<?php

declare(strict_types=1);

namespace App\Observers;

use App\Models\Product;
use App\Notifications\LowStockNotification;
use App\Notifications\NewProductNotification;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ProductObserver
{
    public function created(Product $product): void
    {
        Log::info("নতুন পণ্য তৈরি হয়েছে: {$product->name}");

        // Admin-কে notify করুন
        $product->category->admin->notify(
            new NewProductNotification($product)
        );

        Cache::tags(['products'])->flush();
    }

    public function updated(Product $product): void
    {
        if ($product->wasChanged('stock_quantity') && $product->stock_quantity <= 5) {
            Log::warning("পণ্যের স্টক কম: {$product->name} ({$product->stock_quantity})");

            $product->category->admin->notify(
                new LowStockNotification($product)
            );
        }

        Cache::tags(['products'])->flush();
    }

    public function deleted(Product $product): void
    {
        Log::info("পণ্য মুছে ফেলা হয়েছে: {$product->name}");
        Cache::tags(['products'])->flush();
    }
}

// AppServiceProvider-এ register করতে হবে:
// Product::observe(ProductObserver::class);
```

#### JavaScript — EventEmitter Pattern

```javascript
// events/EventBus.js
import { EventEmitter } from 'events';

// Singleton EventBus — পুরো application-এ একটাই instance
class EventBus extends EventEmitter {
    static #instance = null;

    static getInstance() {
        if (!EventBus.#instance) {
            EventBus.#instance = new EventBus();
        }
        return EventBus.#instance;
    }
}

export { EventBus };

// events/productListeners.js
import { EventBus } from './EventBus.js';
import { sendNotification } from '../utils/notifications.js';
import { logger } from '../utils/logger.js';
import { CacheManager } from '../utils/cache.js';

const events = EventBus.getInstance();
const cache = new CacheManager('products');

events.on('product:created', async (product) => {
    logger.info(`নতুন পণ্য তৈরি হয়েছে: ${product.name}`);
    await sendNotification('admin', `নতুন পণ্য: ${product.name}`);
    await cache.invalidateTag('products');
});

events.on('product:low-stock', async (product) => {
    logger.warn(`পণ্যের স্টক কম: ${product.name} (${product.stockQuantity})`);
    await sendNotification('inventory-team', `স্টক কম: ${product.name}`);
});

events.on('product:deleted', async (product) => {
    logger.info(`পণ্য মুছে ফেলা হয়েছে: ${product.name}`);
    await cache.invalidateTag('products');
});
```

---

### ৪. MVC তে Caching Strategy

MVC-তে caching তিন স্তরে কাজ করে: **Query Cache**, **View/Response Cache**, এবং **Application Cache**।

#### PHP — Laravel Caching

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

final class CachedProductService
{
    private const CACHE_TTL = 3600; // 1 ঘণ্টা

    public function __construct(
        private readonly ProductService $productService,
    ) {}

    // Query Cache — ডাটাবেস query-এর ফলাফল cache
    public function getPopularProducts(int $limit = 10): mixed
    {
        return Cache::tags(['products', 'popular'])->remember(
            key: "popular_products:{$limit}",
            ttl: self::CACHE_TTL,
            callback: fn () => Product::query()
                ->active()
                ->withCount('orders')
                ->orderByDesc('orders_count')
                ->limit($limit)
                ->get(),
        );
    }

    // Page/View Cache — পুরো rendered HTML cache
    public function getCachedProductPage(int $productId): string
    {
        return Cache::remember(
            key: "product_page:{$productId}",
            ttl: self::CACHE_TTL,
            callback: fn () => view('products.show', [
                'product' => Product::with(['category', 'reviews'])->findOrFail($productId),
            ])->render(),
        );
    }
}

// Controller-এ ব্যবহার
// Response Cache — HTTP response level cache
class ProductController extends Controller
{
    public function show(Product $product)
    {
        // ETag-based response caching
        $etag = md5($product->updated_at->toIso8601String());

        return response()
            ->view('products.show', compact('product'))
            ->header('ETag', $etag)
            ->header('Cache-Control', 'public, max-age=3600');
    }
}
```

#### JavaScript — Express Caching

```javascript
// middleware/cache.js
import { CacheManager } from '../utils/cache.js';

const cache = new CacheManager('response');

export function cacheResponse(ttl = 3600) {
    return async (req, res, next) => {
        const key = `response:${req.originalUrl}`;
        const cached = await cache.get(key);

        if (cached) {
            return res.json(JSON.parse(cached));
        }

        // response-এর json method override করে cache-এ save
        const originalJson = res.json.bind(res);
        res.json = async (data) => {
            await cache.set(key, JSON.stringify(data), ttl);
            return originalJson(data);
        };

        next();
    };
}

// Route-এ ব্যবহার
// routes/productRoutes.js
import { cacheResponse } from '../middleware/cache.js';

apiRouter.get('/products', cacheResponse(1800), productController.apiIndex);
apiRouter.get('/products/:id', cacheResponse(3600), productController.apiShow);
```

---

### ৫. MVC তে API Resource/Transformer

API response-কে consistent ও structured রাখতে Resource/Transformer pattern ব্যবহার করা হয়। এটি Model-এর internal structure থেকে API response-কে decouple করে।

#### PHP — Laravel API Resource

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    /** @return array<string, mixed> */
    public function toArray(Request $request): array
    {
        return [
            'id'                  => $this->id,
            'name'                => $this->name,
            'slug'                => $this->slug,
            'price'               => $this->price,
            'formatted_price'     => $this->formatted_price,
            'discount_price'      => $this->discount_price,
            'discount_percentage' => $this->discount_percentage,
            'is_available'        => $this->isAvailable(),
            'stock_quantity'      => $this->when(
                $request->user()?->isAdmin(),
                $this->stock_quantity,
            ),
            'status' => [
                'value' => $this->status->value,
                'label' => $this->status->label(),
            ],
            'category'   => new CategoryResource($this->whenLoaded('category')),
            'reviews'    => ReviewResource::collection($this->whenLoaded('reviews')),
            'created_at' => $this->created_at->toIso8601String(),
            'links'      => [
                'self' => route('api.products.show', $this->id),
            ],
        ];
    }
}

// Collection Resource — pagination meta সহ
class ProductCollection extends \Illuminate\Http\Resources\Json\ResourceCollection
{
    public $collects = ProductResource::class;

    /** @return array<string, mixed> */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total'        => $this->total(),
                'per_page'     => $this->perPage(),
                'current_page' => $this->currentPage(),
                'last_page'    => $this->lastPage(),
            ],
        ];
    }
}
```

#### JavaScript — Express Transformer

```javascript
// transformers/productTransformer.js

class ProductTransformer {
    static transform(product, user = null) {
        const data = {
            id: product._id,
            name: product.name,
            slug: product.slug,
            price: product.price,
            formattedPrice: product.formattedPrice,
            discountPrice: product.discountPrice,
            discountPercentage: product.discountPercentage,
            isAvailable: product.isAvailable(),
            status: {
                value: product.status,
                label: ProductStatus.label(product.status),
            },
            category: product.category
                ? CategoryTransformer.transform(product.category)
                : null,
            reviews: product.reviews?.map(ReviewTransformer.transform) ?? [],
            createdAt: product.createdAt?.toISOString(),
            links: {
                self: `/api/v1/products/${product._id}`,
            },
        };

        // Admin-only fields
        if (user?.role === 'admin') {
            data.stockQuantity = product.stockQuantity;
        }

        return data;
    }

    static transformCollection(products, user = null) {
        return products.map((p) => ProductTransformer.transform(p, user));
    }

    static paginatedResponse(products, pagination, user = null) {
        return {
            data: this.transformCollection(products, user),
            meta: {
                total: pagination.total,
                perPage: pagination.perPage,
                currentPage: pagination.page,
                totalPages: pagination.totalPages,
            },
        };
    }
}

export default ProductTransformer;
```

---

## 🔄 MVC Variants

### MVP (Model-View-Presenter)

MVP-তে View এবং Model সরাসরি communicate করে না। Presenter (Controller-এর মতো) সব manage করে এবং View-কে explicitly update করে। View সম্পূর্ণ passive — Presenter-এর নির্দেশ ছাড়া কিছু করে না।

```
┌──────────┐     ┌────────────┐     ┌──────────┐
│   View   │◀───▶│  Presenter │────▶│  Model   │
│ (Passive)│     │(All logic) │◀────│          │
└──────────┘     └────────────┘     └──────────┘
```

```php
// PHP MVP উদাহরণ
interface ProductViewInterface
{
    public function showProducts(array $products): void;
    public function showError(string $message): void;
}

final class ProductPresenter
{
    public function __construct(
        private readonly ProductViewInterface $view,
        private readonly ProductRepositoryInterface $repository,
    ) {}

    public function loadProducts(): void
    {
        try {
            $products = $this->repository->paginate();
            $this->view->showProducts($products->items());
        } catch (\Throwable $e) {
            $this->view->showError('পণ্য লোড করতে সমস্যা হয়েছে।');
        }
    }
}
```

### MVVM (Model-View-ViewModel)

MVVM-তে ViewModel View এবং Model-এর মধ্যে বসে। ViewModel data binding-এর মাধ্যমে View-কে automatically update করে। Vue.js, Angular, Knockout.js এই pattern ব্যবহার করে।

```
┌──────────┐  data-binding  ┌─────────────┐     ┌──────────┐
│   View   │◀══════════════▶│  ViewModel  │────▶│  Model   │
│ (Template)│               │(Reactive)   │◀────│          │
└──────────┘               └─────────────┘     └──────────┘
```

```javascript
// Vue.js MVVM উদাহরণ — ViewModel হিসেবে
import { ref, computed, onMounted } from 'vue';

// ViewModel (Composition API)
export function useProductList() {
    const products = ref([]);
    const loading = ref(false);
    const error = ref(null);
    const searchQuery = ref('');

    // Computed — View-এ automatically update হবে
    const filteredProducts = computed(() =>
        products.value.filter((p) =>
            p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
        )
    );

    const totalPrice = computed(() =>
        filteredProducts.value.reduce((sum, p) => sum + p.price, 0)
    );

    async function fetchProducts() {
        loading.value = true;
        try {
            const res = await fetch('/api/v1/products');
            const data = await res.json();
            products.value = data.data;
        } catch (e) {
            error.value = 'পণ্য লোড করতে সমস্যা হয়েছে।';
        } finally {
            loading.value = false;
        }
    }

    onMounted(fetchProducts);

    return { products, filteredProducts, totalPrice, loading, error, searchQuery };
}
```

### MVA (Model-View-Adapter)

MVA-তে Adapter (Mediating Controller) View এবং Model-কে সম্পূর্ণ decouple করে। View Model সম্পর্কে কিছু জানে না এবং Model-ও View সম্পর্কে কিছু জানে না।

### 📊 MVC Variants তুলনা

| বৈশিষ্ট্য | MVC | MVP | MVVM | MVA |
|---|---|---|---|---|
| **View-Model সম্পর্ক** | View Model observe করতে পারে | সরাসরি নেই | Two-way data binding | সম্পূর্ণ decoupled |
| **Controller/Presenter** | Thin orchestrator | All presentation logic | ViewModel (reactive state) | Adapter (mediator) |
| **View-এর ভূমিকা** | Semi-active | Passive (Presenter update করে) | Template (binding-driven) | Fully passive |
| **Testing** | Controller testable | Presenter সহজে testable | ViewModel unit testable | Adapter testable |
| **Complexity** | কম | মাঝারি | মাঝারি-বেশি | বেশি |
| **Best For** | Web apps (server-side) | Android, WinForms | SPA (Vue, Angular) | Complex desktop apps |
| **বাংলাদেশে ব্যবহার** | Laravel, Express projects | Android apps | Vue/Angular frontends | বিরল |

---

## ✅ সুবিধা (Pros) ও ❌ অসুবিধা (Cons)

| ✅ সুবিধা | ❌ অসুবিধা |
|---|---|
| **Separation of Concerns** — প্রতিটি component এর নিজস্ব দায়িত্ব | **Overhead** — ছোট project-এ অতিরিক্ত complexity |
| **Parallel Development** — ফ্রন্টএন্ড ও ব্যাকএন্ড আলাদা কাজ | **Tight Coupling** — Controller প্রায়ই View ও Model উভয়ের উপর নির্ভরশীল |
| **Testability** — প্রতিটি layer আলাদাভাবে test করা যায় | **Fat Controller/Model** — সঠিকভাবে না করলে anti-pattern তৈরি হয় |
| **Reusability** — একই Model বিভিন্ন View-এ ব্যবহার যোগ্য | **Multiple Files** — একটি feature-এ ৩-৪টি file তৈরি করতে হয় |
| **Maintainability** — বড় project দীর্ঘমেয়াদে maintain করা সহজ | **Learning Curve** — নতুন developer-দের জন্য concept বোঝা কঠিন |
| **Scalability** — দলের আকার বাড়লেও কাজ ভাগ করা যায় | **Boilerplate** — প্রচুর repetitive কোড লিখতে হয় |
| **Community Support** — বিশাল ecosystem ও documentation | **Not for real-time** — WebSocket/real-time app-এ জটিলতা বাড়ে |

---

## ⚠️ সাধারণ ভুল ও Anti-Patterns

### ১. Fat Controller (কন্ট্রোলারে অতিরিক্ত লজিক)

```php
// ❌ খারাপ — Controller-এ সব business logic
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // Validation, business logic, DB query, notification — সব এখানে
        $validated = $request->validate([
            'product_id' => 'required|exists:products,id',
            'quantity' => 'required|integer|min:1',
        ]);

        $product = Product::find($validated['product_id']);

        if ($product->stock_quantity < $validated['quantity']) {
            return back()->with('error', 'পর্যাপ্ত স্টক নেই');
        }

        $product->decrement('stock_quantity', $validated['quantity']);
        $total = $product->price * $validated['quantity'];
        $tax = $total * 0.15;

        $order = Order::create([
            'user_id' => auth()->id(),
            'total' => $total + $tax,
            'tax' => $tax,
        ]);

        OrderItem::create([
            'order_id' => $order->id,
            'product_id' => $product->id,
            'quantity' => $validated['quantity'],
            'price' => $product->price,
        ]);

        Mail::to(auth()->user())->send(new OrderConfirmation($order));
        Notification::send($product->vendor, new NewOrderNotification($order));

        return redirect()->route('orders.show', $order);
    }
}
```

```php
// ✅ ভালো — Thin Controller, Service-এ business logic
class OrderController extends Controller
{
    public function __construct(
        private readonly OrderService $orderService,
    ) {}

    public function store(StoreOrderRequest $request): RedirectResponse
    {
        $order = $this->orderService->placeOrder(
            userId: auth()->id(),
            productId: $request->validated('product_id'),
            quantity: $request->validated('quantity'),
        );

        return redirect()
            ->route('orders.show', $order)
            ->with('success', 'অর্ডার সফলভাবে সম্পন্ন হয়েছে।');
    }
}
```

### ২. View-তে Business Logic

```javascript
// ❌ খারাপ — View/Template-এ business logic
// EJS Template
<% products.forEach(product => { %>
    <div class="product">
        <% 
        // ⛔ ডিসকাউন্ট ক্যালকুলেশন View-তে করা হচ্ছে
        let finalPrice = product.price;
        if (product.discountPrice && new Date() < new Date(product.discountExpiry)) {
            finalPrice = product.discountPrice;
            if (user.isPremium) {
                finalPrice *= 0.95; // Premium user-দের ৫% extra ছাড়
            }
        }
        if (product.stock < 10) {
            finalPrice *= 1.1; // স্টক কম থাকলে ১০% বেশি
        }
        %>
        <span>৳<%= finalPrice.toFixed(2) %></span>
    </div>
<% }); %>
```

```javascript
// ✅ ভালো — Model/Service-এ logic, View শুধু display করে
// Model-এ computed property
productSchema.methods.getFinalPrice = function (user) {
    return this.pricingStrategy.calculate(this, user);
};

// View শুধু display করে
// <span>৳<%= product.finalPrice %></span>
```

### ৩. God Model (সব একটি Model-এ)

```php
// ❌ খারাপ — একটি Model-এ সব responsibility
class User extends Model
{
    public function calculateTax() { /* ... */ }
    public function sendEmail() { /* ... */ }
    public function generatePDF() { /* ... */ }
    public function processPayment() { /* ... */ }
    public function exportToCSV() { /* ... */ }
    public function syncWithBkash() { /* ... */ }
}
```

```php
// ✅ ভালো — Single Responsibility Principle
class User extends Model
{
    // শুধু User-related data ও relationships
}

class TaxCalculator { /* কর হিসাব */ }
class EmailService { /* ইমেইল পাঠানো */ }
class PaymentGateway { /* পেমেন্ট প্রক্রিয়া */ }
class BkashIntegration { /* bKash sync */ }
```

### ৪. Controller-এ সরাসরি DB Query

```javascript
// ❌ খারাপ — Controller-এ raw DB query
app.get('/products', async (req, res) => {
    const products = await db.query(
        `SELECT p.*, c.name as category_name 
         FROM products p 
         JOIN categories c ON p.category_id = c.id 
         WHERE p.status = 'active' 
         AND p.stock > 0 
         ORDER BY p.created_at DESC 
         LIMIT 15 OFFSET ${(req.query.page - 1) * 15}`
    );
    res.render('products', { products });
});
```

```javascript
// ✅ ভালো — Model/Repository ব্যবহার করুন
app.get('/products', async (req, res) => {
    const products = await productRepository.paginate(
        { status: 'active' },
        req.query.page,
        15,
    );
    res.render('products', { products });
});
```

---

## 🧪 টেস্টিং MVC

### Model Testing

#### PHP — PHPUnit

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\Models;

use App\Models\Product;
use App\Models\ProductStatus;
use App\Models\Category;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ProductTest extends TestCase
{
    use RefreshDatabase;

    public function test_product_is_available_when_active_and_in_stock(): void
    {
        $product = Product::factory()->create([
            'status'         => ProductStatus::Active,
            'stock_quantity'  => 10,
        ]);

        $this->assertTrue($product->isAvailable());
    }

    public function test_product_is_not_available_when_out_of_stock(): void
    {
        $product = Product::factory()->create([
            'status'         => ProductStatus::Active,
            'stock_quantity'  => 0,
        ]);

        $this->assertFalse($product->isAvailable());
    }

    public function test_reduce_stock_decrements_quantity(): void
    {
        $product = Product::factory()->create(['stock_quantity' => 10]);

        $product->reduceStock(3);

        $this->assertEquals(7, $product->fresh()->stock_quantity);
    }

    public function test_reduce_stock_throws_when_insufficient(): void
    {
        $product = Product::factory()->create(['stock_quantity' => 2]);

        $this->expectException(\DomainException::class);
        $this->expectExceptionMessage('পর্যাপ্ত স্টক নেই।');

        $product->reduceStock(5);
    }

    public function test_formatted_price_accessor(): void
    {
        $product = Product::factory()->create(['price' => 1500.50]);

        $this->assertEquals('৳1,500.50', $product->formatted_price);
    }

    public function test_discount_percentage_calculation(): void
    {
        $product = Product::factory()->create([
            'price'          => 1000,
            'discount_price' => 750,
        ]);

        $this->assertEquals(25, $product->discount_percentage);
    }

    public function test_active_scope_filters_correctly(): void
    {
        Product::factory()->count(3)->create(['status' => ProductStatus::Active]);
        Product::factory()->count(2)->create(['status' => ProductStatus::Inactive]);

        $this->assertCount(3, Product::active()->get());
    }
}
```

#### JavaScript — Jest

```javascript
// tests/models/Product.test.js
import mongoose from 'mongoose';
import Product, { ProductStatus } from '../../models/Product.js';
import { connectTestDB, clearTestDB, closeTestDB } from '../helpers/db.js';

beforeAll(() => connectTestDB());
afterEach(() => clearTestDB());
afterAll(() => closeTestDB());

describe('Product Model', () => {
    const validProduct = {
        name: 'Test Product',
        slug: 'test-product',
        price: 1500.50,
        category: new mongoose.Types.ObjectId(),
        stockQuantity: 10,
        status: ProductStatus.Active,
    };

    test('সক্রিয় ও স্টকে থাকলে isAvailable() true', async () => {
        const product = await Product.create(validProduct);
        expect(product.isAvailable()).toBe(true);
    });

    test('স্টক ০ হলে isAvailable() false', async () => {
        const product = await Product.create({ ...validProduct, stockQuantity: 0 });
        expect(product.isAvailable()).toBe(false);
    });

    test('reduceStock() সঠিকভাবে কমায়', async () => {
        const product = await Product.create(validProduct);
        await product.reduceStock(3);
        expect(product.stockQuantity).toBe(7);
    });

    test('অপর্যাপ্ত স্টকে reduceStock() error দেয়', async () => {
        const product = await Product.create({ ...validProduct, stockQuantity: 2 });
        await expect(product.reduceStock(5)).rejects.toThrow('পর্যাপ্ত স্টক নেই।');
    });

    test('formattedPrice সঠিকভাবে format করে', async () => {
        const product = await Product.create(validProduct);
        expect(product.formattedPrice).toBe('৳1500.50');
    });

    test('discountPercentage সঠিকভাবে হিসাব করে', async () => {
        const product = await Product.create({
            ...validProduct,
            price: 1000,
            discountPrice: 750,
        });
        expect(product.discountPercentage).toBe(25);
    });

    test('findInStock() শুধু active ও stock>0 return করে', async () => {
        await Product.create(validProduct);
        await Product.create({ ...validProduct, slug: 'inactive', status: 'inactive' });
        await Product.create({ ...validProduct, slug: 'empty', stockQuantity: 0 });

        const results = await Product.findInStock();
        expect(results).toHaveLength(1);
    });
});
```

### Controller (Feature) Testing

#### PHP — Laravel Feature Test

```php
<?php

namespace Tests\Feature;

use App\Models\Product;
use App\Models\ProductStatus;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ProductControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_index_displays_active_products(): void
    {
        Product::factory()->count(3)->create(['status' => ProductStatus::Active]);
        Product::factory()->create(['status' => ProductStatus::Inactive]);

        $response = $this->get('/products');

        $response->assertOk();
        $response->assertViewHas('products');
        $this->assertCount(3, $response->viewData('products'));
    }

    public function test_api_store_creates_product(): void
    {
        $user = User::factory()->admin()->create();

        $payload = [
            'name'           => 'নতুন পণ্য',
            'slug'           => 'notun-ponno',
            'price'          => 500.00,
            'category_id'    => 1,
            'stock_quantity'  => 50,
            'status'         => 'active',
        ];

        $response = $this->actingAs($user, 'sanctum')
            ->postJson('/api/v1/products', $payload);

        $response->assertCreated()
            ->assertJsonPath('data.name', 'নতুন পণ্য')
            ->assertJsonPath('data.price', 500.00);

        $this->assertDatabaseHas('products', ['slug' => 'notun-ponno']);
    }

    public function test_store_validation_fails_without_name(): void
    {
        $user = User::factory()->admin()->create();

        $response = $this->actingAs($user, 'sanctum')
            ->postJson('/api/v1/products', ['price' => 500]);

        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['name', 'slug', 'category_id']);
    }
}
```

#### JavaScript — Supertest

```javascript
// tests/controllers/product.test.js
import request from 'supertest';
import app from '../../app.js';
import Product from '../../models/Product.js';
import { connectTestDB, clearTestDB, closeTestDB } from '../helpers/db.js';
import { generateToken } from '../helpers/auth.js';

beforeAll(() => connectTestDB());
afterEach(() => clearTestDB());
afterAll(() => closeTestDB());

describe('Product Controller', () => {
    describe('GET /api/v1/products', () => {
        test('সক্রিয় পণ্য list return করে', async () => {
            await Product.create([
                { name: 'P1', slug: 'p1', price: 100, stockQuantity: 10, status: 'active', category: someId },
                { name: 'P2', slug: 'p2', price: 200, stockQuantity: 0, status: 'active', category: someId },
            ]);

            const res = await request(app).get('/api/v1/products');

            expect(res.status).toBe(200);
            expect(res.body.data).toHaveLength(1); // stock > 0 শুধু
        });
    });

    describe('POST /api/v1/products', () => {
        test('নতুন পণ্য তৈরি করে', async () => {
            const token = generateToken({ role: 'admin' });

            const res = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${token}`)
                .send({
                    name: 'নতুন পণ্য',
                    slug: 'notun-ponno',
                    price: 500,
                    category: someCategoryId,
                    stockQuantity: 50,
                });

            expect(res.status).toBe(201);
            expect(res.body.data.name).toBe('নতুন পণ্য');
        });

        test('authentication ছাড়া 401 return করে', async () => {
            const res = await request(app)
                .post('/api/v1/products')
                .send({ name: 'Test' });

            expect(res.status).toBe(401);
        });
    });
});
```

---

## 📏 কখন ব্যবহার করবেন / করবেন না

### ✅ কখন MVC ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|---|---|
| **Web Application** (e-commerce, blog, CMS) | MVC-এর natural fit — request-response cycle |
| **Team-based Development** | ফ্রন্টএন্ড ও ব্যাকএন্ড ডেভেলপাররা আলাদাভাবে কাজ করতে পারে |
| **Multiple UI Platforms** | একই Model, ভিন্ন View (web, mobile API, admin panel) |
| **CRUD-heavy Applications** | Resource-based routing MVC-তে সহজ |
| **বাংলাদেশী E-commerce** (Daraz, Chaldal, PriyoShop) | Product, Order, User management |
| **bKash/Nagad-এর মতো fintech** | Transaction, Account, Payment management |
| **Government Portal** | Form submission, approval workflows |

### ❌ কখন MVC ব্যবহার করবেন না

| পরিস্থিতি | কারণ | বিকল্প |
|---|---|---|
| **Simple Script/CLI Tool** | অতিরিক্ত overhead | Procedural বা Simple OOP |
| **Real-time Application** (chat, gaming) | Request-response model মানে না | Event-driven, CQRS |
| **Microservices** | একটি service-এ MVC জটিলতা বাড়ায় | Hexagonal / Clean Architecture |
| **Machine Learning Pipeline** | Data flow ভিন্ন প্যাটার্ন অনুসরণ করে | Pipeline Pattern |
| **Static Website** | No dynamic content | Static Site Generator |
| **Serverless Functions** | একটি function-এ MVC অপ্রয়োজনীয় | Function-based architecture |

---

## 🔗 অন্যান্য প্যাটার্নের সাথে সম্পর্ক

| প্যাটার্ন | MVC-এর সাথে সম্পর্ক |
|---|---|
| **Observer Pattern** | Model data পরিবর্তনে View update হয় — Observer pattern-এর implementation |
| **Strategy Pattern** | Controller বিভিন্ন strategy অনুযায়ী Model ও View select করতে পারে |
| **Factory Pattern** | Controller-এ response তৈরিতে Factory ব্যবহার হতে পারে |
| **Repository Pattern** | Model-এর data access logic আলাদা করে — MVC-এর সাথে ব্যবহৃত |
| **Service Layer** | Controller ও Model-এর মধ্যে business logic layer — MVC-তে best practice |
| **CQRS** | MVC-এর read/write operation আলাদা করা — advanced scaling-এ ব্যবহৃত |
| **Microservices** | প্রতিটি microservice অভ্যন্তরীণভাবে MVC হতে পারে |
| **Clean Architecture** | MVC-এর উন্নত সংস্করণ — dependency inversion সহ |
| **Hexagonal Architecture** | MVC-কে ports & adapters-এ বিভক্ত করে — external dependency isolate করে |
| **Event Sourcing** | MVC Model-এর data persistence strategy হিসেবে ব্যবহৃত |

---

## 📋 সারসংক্ষেপ

| বিষয় | বিবরণ |
|---|---|
| **প্যাটার্ন টাইপ** | Architectural Pattern |
| **উদ্ভাবক** | Trygve Reenskaug (১৯৭৯, Xerox PARC) |
| **মূল নীতি** | Separation of Concerns — UI, Business Logic, Data আলাদা |
| **তিন কম্পোনেন্ট** | Model (data + logic), View (presentation), Controller (coordinator) |
| **Best Practice** | Thin Controller + Service Layer + Repository |
| **PHP Framework** | Laravel, Symfony, CodeIgniter |
| **JS Framework** | Express, NestJS, Sails.js |
| **Variants** | MVP, MVVM, MVA, HMVC |
| **Testing** | প্রতিটি layer আলাদাভাবে testable |
| **সবচেয়ে বড় সুবিধা** | Maintainability ও Team Collaboration |
| **সবচেয়ে বড় সমস্যা** | Fat Controller / God Model anti-pattern |
| **বাংলাদেশে ব্যবহার** | Laravel (৮০%+ BD web projects), Express.js |

> **মনে রাখুন:** MVC শুধু একটি file structure নয় — এটি একটি **চিন্তার পদ্ধতি**। সঠিকভাবে implement করলে এটি আপনার codebase-কে maintainable, testable, এবং scalable করে তোলে। তবে blindly MVC follow না করে project-এর প্রয়োজন অনুযায়ী adapt করুন। ছোট project-এ simple structure যথেষ্ট, বড় project-এ MVC + Service + Repository দিয়ে শুরু করুন।

---

*📚 আরও পড়ুন: [SOLID Principles](../01-solid/), [Design Patterns](../02-design-patterns/), [Clean Code](../08-clean-code/)*
