# 🔗 ডাটাবেস রিলেশনশিপ — Advanced Deep Dive

## 📌 সংজ্ঞা ও মূল ধারণা

### Entity-Relationship Model

Entity-Relationship (ER) Model হলো ডাটাবেস ডিজাইনের ভিত্তি। এটি real-world entities এবং তাদের মধ্যকার সম্পর্ক (relationships) কে structured format-এ প্রকাশ করে। একটি বাংলাদেশি e-commerce platform (যেমন Daraz, Chaldal) ডিজাইন করতে গেলে আমাদের বুঝতে হবে — `User`, `Order`, `Product`, `Category`, `Review` এই entity গুলো কীভাবে পরস্পর সম্পর্কিত।

**তিনটি মূল উপাদান:**
- **Entity**: একটি real-world object (User, Product, Order)
- **Attribute**: Entity-র property (name, email, price)
- **Relationship**: Entity-দের মধ্যকার সংযোগ (User "places" Order)

### Key Types

```
┌─────────────────────────────────────────────────────────────────────┐
│  Key Type          │ বর্ণনা                                         │
├─────────────────────────────────────────────────────────────────────┤
│  Primary Key (PK)  │ প্রতিটি row-কে uniquely identify করে           │
│  Foreign Key (FK)  │ অন্য table-এর PK-কে reference করে              │
│  Candidate Key     │ PK হওয়ার যোগ্যতা রাখে এমন সব key              │
│  Composite Key     │ দুই বা ততোধিক column মিলে PK তৈরি করে          │
│  Surrogate Key     │ System-generated key (AUTO_INCREMENT, UUID)     │
│  Natural Key       │ Business meaning আছে এমন key (email, NID)      │
└─────────────────────────────────────────────────────────────────────┘
```

```sql
-- Primary Key: surrogate vs natural
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- surrogate key
    email VARCHAR(255) UNIQUE NOT NULL,              -- candidate key (natural)
    phone VARCHAR(20) UNIQUE,                        -- candidate key
    nid VARCHAR(17) UNIQUE                           -- candidate key (Bangladesh NID)
);

-- Composite Key: pivot table-এ সবচেয়ে বেশি ব্যবহৃত হয়
CREATE TABLE order_items (
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);
```

### Referential Integrity

Referential Integrity নিশ্চিত করে যে foreign key সবসময় একটি valid primary key-কে reference করছে। এটি ডাটার consistency রক্ষার মূল ভিত্তি। কোনো orphan record থাকবে না — অর্থাৎ, যদি একটি `order`-এর `user_id = 5` থাকে, তাহলে `users` table-এ অবশ্যই `id = 5` এমন একটি row থাকতে হবে।

```sql
-- Referential integrity enforcement
SET FOREIGN_KEY_CHECKS = 1;  -- MySQL: enforce করা

-- এই INSERT ব্যর্থ হবে কারণ user_id = 999 users table-এ নেই
INSERT INTO orders (user_id, total) VALUES (999, 5000.00);
-- ERROR 1452: Cannot add or update a child row: a foreign key constraint fails
```

---

## 📊 ER ডায়াগ্রাম — Bangladesh E-Commerce

```
┌──────────────┐       ┌───────────────────┐       ┌──────────────────┐
│   categories │       │     products      │       │     reviews      │
├──────────────┤       ├───────────────────┤       ├──────────────────┤
│ PK id        │──┐    │ PK id             │──┐    │ PK id            │
│ name         │  │    │ FK category_id    │  │    │ FK user_id       │
│ FK parent_id │──┘1:N │ name              │  │1:N │ FK product_id    │
│ slug         │───────│ price             │  └────│ rating (1-5)     │
│ is_active    │       │ stock             │       │ comment          │
└──────────────┘       │ description       │       │ created_at       │
      │                └───────────────────┘       └──────────────────┘
      │self-ref              │                            │
      └──────┘               │                            │
                             │ M:N                        │
                    ┌────────┴────────┐                   │
                    │  order_items    │                   │
                    ├─────────────────┤                   │
                    │ FK order_id     │                   │
                    │ FK product_id   │                   │
                    │ quantity        │                   │
                    │ unit_price      │                   │
                    └────────┬────────┘                   │
                             │ N:1                        │
┌──────────────┐    ┌────────┴────────┐    ┌─────────────┴──┐
│   profiles   │    │    orders       │    │     users      │
├──────────────┤    ├─────────────────┤    ├────────────────┤
│ PK id        │    │ PK id           │    │ PK id          │
│ FK user_id   │1:1 │ FK user_id      │N:1 │ name           │
│ avatar       │────│ FK coupon_id    │────│ email          │
│ bio          │    │ status          │    │ phone          │
│ address      │    │ total           │    │ password       │
│ division     │    │ shipping_address│    │ created_at     │
└──────────────┘    │ payment_method  │    └────────────────┘
                    └─────────────────┘           │
                             │ 1:1                │ M:N
                    ┌────────┴────────┐    ┌──────┴─────────┐
                    │   invoices      │    │  role_user     │
                    ├─────────────────┤    ├────────────────┤
                    │ PK id           │    │ FK user_id     │
                    │ FK order_id     │    │ FK role_id     │
                    │ invoice_number  │    │ assigned_at    │
                    │ amount          │    └────────┬───────┘
                    │ tax             │             │ N:1
                    │ issued_at       │    ┌────────┴───────┐
                    └─────────────────┘    │    roles       │
                                           ├────────────────┤
                    ┌─────────────────┐    │ PK id          │
                    │    coupons      │    │ name           │
                    ├─────────────────┤    │ guard_name     │
                    │ PK id           │    └────────────────┘
                    │ code            │
                    │ discount_type   │    ┌─────────────────────┐
                    │ discount_value  │    │ images (polymorphic)│
                    │ min_order       │    ├─────────────────────┤
                    │ expires_at      │    │ PK id               │
                    └─────────────────┘    │ imageable_type      │
                                           │ imageable_id        │
                                           │ path                │
                                           │ sort_order          │
                                           └─────────────────────┘
```

---

## 💻 Relationship Types Deep Dive

### ১. One-to-One (1:1)

একটি entity-র সাথে ঠিক একটি অন্য entity সম্পর্কিত। উদাহরণ: প্রতিটি `User`-এর একটি মাত্র `Profile`, প্রতিটি `Order`-এর একটি মাত্র `Invoice`।

**কখন ব্যবহার করবেন:**
- Table-টি অনেক বড় হলে — কম ব্যবহৃত column আলাদা table-এ রাখা (vertical partitioning)
- Security — sensitive data আলাদা table-এ রাখা (password, NID)
- Optional data — সব user-এর profile নাও থাকতে পারে

**কখন একটি table-এ merge করবেন:**
- দুটি table সবসময় একসাথে query হলে
- Data সবসময় 1:1 থাকবে এমন নিশ্চিত হলে
- JOIN overhead কমাতে চাইলে

#### SQL Schema

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE profiles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED UNIQUE NOT NULL,  -- UNIQUE ensures 1:1
    avatar VARCHAR(500),
    bio TEXT,
    date_of_birth DATE,
    division ENUM('Dhaka','Chittagong','Rajshahi','Khulna','Barisal','Sylhet','Rangpur','Mymensingh'),
    address TEXT,
    postal_code VARCHAR(10),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Query: user সহ profile আনা
SELECT u.*, p.avatar, p.bio, p.division
FROM users u
LEFT JOIN profiles p ON u.id = p.user_id
WHERE u.id = 1;
```

#### Laravel Eloquent

```php
// app/Models/User.php
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);
        // SQL: SELECT * FROM profiles WHERE user_id = ?
    }
}

// app/Models/Profile.php
class Profile extends Model
{
    protected $fillable = ['user_id', 'avatar', 'bio', 'division', 'address'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
        // SQL: SELECT * FROM users WHERE id = ?
    }
}

// ব্যবহার
$user = User::find(1);
$profile = $user->profile;           // lazy load
$user = User::with('profile')->find(1); // eager load — recommended

// Create through relationship
$user->profile()->create([
    'bio' => 'Senior Developer from Dhaka',
    'division' => 'Dhaka',
]);
```

#### Sequelize

```javascript
// models/User.js
const User = sequelize.define('User', {
    name: { type: DataTypes.STRING(100), allowNull: false },
    email: { type: DataTypes.STRING(255), allowNull: false, unique: true },
    phone: { type: DataTypes.STRING(20), unique: true },
});

// models/Profile.js
const Profile = sequelize.define('Profile', {
    userId: {
        type: DataTypes.BIGINT.UNSIGNED,
        allowNull: false,
        unique: true, // 1:1 enforce
        references: { model: 'users', key: 'id' },
    },
    avatar: DataTypes.STRING(500),
    bio: DataTypes.TEXT,
    division: DataTypes.ENUM('Dhaka','Chittagong','Rajshahi','Khulna','Barisal','Sylhet','Rangpur','Mymensingh'),
});

// Association setup
User.hasOne(Profile, { foreignKey: 'userId', onDelete: 'CASCADE' });
Profile.belongsTo(User, { foreignKey: 'userId' });

// Query
const user = await User.findByPk(1, {
    include: [{ model: Profile }],
});
console.log(user.Profile.division); // 'Dhaka'
```

#### Polymorphic One-to-One

একটি `Image` model যেকোনো entity-র সাথে 1:1 সম্পর্ক রাখতে পারে:

```php
// Laravel Polymorphic One-to-One
class Image extends Model
{
    public function imageable(): MorphTo
    {
        return $this->morphTo();
    }
}

class User extends Model
{
    public function avatar(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

class Product extends Model
{
    public function featuredImage(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

// images table: id, imageable_type, imageable_id, path
// imageable_type = 'App\Models\User' or 'App\Models\Product'
```

---

### ২. One-to-Many (1:N)

সবচেয়ে বেশি ব্যবহৃত relationship। একটি parent entity-র অনেক child entity থাকতে পারে, কিন্তু প্রতিটি child-এর একটি মাত্র parent। উদাহরণ: একজন `User`-এর অনেক `Order`, একটি `Category`-তে অনেক `Product`।

#### SQL Schema

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    coupon_id BIGINT UNSIGNED NULL,
    status ENUM('pending','confirmed','processing','shipped','delivered','cancelled','refunded')
        DEFAULT 'pending',
    total DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    shipping_address TEXT NOT NULL,
    payment_method ENUM('bkash','nagad','rocket','cod','card') NOT NULL,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (coupon_id) REFERENCES coupons(id) ON DELETE SET NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- একজন user-এর সব order আনা
SELECT o.*, u.name AS customer_name
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.user_id = 42
ORDER BY o.created_at DESC;

-- প্রতিটি user-এর order count সহ
SELECT u.id, u.name, COUNT(o.id) AS order_count, SUM(o.total) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
HAVING order_count > 0
ORDER BY total_spent DESC
LIMIT 20;
```

#### Cascading Actions

```sql
-- ON DELETE CASCADE: parent মুছলে child-ও মুছে যাবে
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE

-- ON DELETE SET NULL: parent মুছলে FK NULL হয়ে যাবে
FOREIGN KEY (coupon_id) REFERENCES coupons(id) ON DELETE SET NULL

-- ON DELETE RESTRICT: child থাকলে parent মোছা যাবে না (default)
FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT

-- ON UPDATE CASCADE: parent-এর PK change হলে child-এর FK update হবে
FOREIGN KEY (category_id) REFERENCES categories(id) ON UPDATE CASCADE
```

**বাস্তব সিদ্ধান্ত নির্দেশিকা:**

| Scenario | Action | কারণ |
|----------|--------|-------|
| User → Orders | CASCADE | User account delete হলে তার order-ও delete |
| Order → Coupon | SET NULL | Coupon মুছলেও order থাকবে, শুধু coupon ref হারাবে |
| OrderItem → Product | RESTRICT | Product মোছা যাবে না যতক্ষণ order-এ আছে |

#### Laravel Eloquent

```php
// app/Models/User.php
class User extends Model
{
    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }

    // Scoped relationship
    public function pendingOrders(): HasMany
    {
        return $this->hasMany(Order::class)->where('status', 'pending');
    }

    public function recentOrders(): HasMany
    {
        return $this->hasMany(Order::class)
            ->where('created_at', '>=', now()->subDays(30))
            ->orderByDesc('created_at');
    }
}

// app/Models/Order.php
class Order extends Model
{
    protected $fillable = ['user_id', 'status', 'total', 'shipping_address', 'payment_method'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }
}

// ব্যবহার
$user = User::with(['orders' => function ($query) {
    $query->where('status', 'delivered')->latest()->limit(10);
}])->find(42);

// Aggregate
$orderCount = $user->orders()->count();
$totalSpent = $user->orders()->where('status', 'delivered')->sum('total');

// withCount — N+1 ছাড়া count আনা
$users = User::withCount('orders')
    ->having('orders_count', '>', 5)
    ->get();
```

#### Sequelize

```javascript
// Associations
User.hasMany(Order, { foreignKey: 'userId', as: 'orders' });
Order.belongsTo(User, { foreignKey: 'userId', as: 'user' });
Order.hasMany(OrderItem, { foreignKey: 'orderId', as: 'items' });

// Query সহ eager loading ও filter
const user = await User.findByPk(42, {
    include: [{
        model: Order,
        as: 'orders',
        where: { status: 'delivered' },
        required: false, // LEFT JOIN (order না থাকলেও user আসবে)
        limit: 10,
        order: [['createdAt', 'DESC']],
        include: [{
            model: OrderItem,
            as: 'items',
            include: [{ model: Product }],
        }],
    }],
});

// Aggregate query
const totalSpent = await Order.sum('total', {
    where: { userId: 42, status: 'delivered' },
});
```

#### Self-Referencing One-to-Many

Category → SubCategory, Comment → Reply — parent এবং child একই table-এ:

```sql
CREATE TABLE categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id BIGINT UNSIGNED NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(120) UNIQUE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE CASCADE,
    INDEX idx_parent_id (parent_id)
);

-- Recursive CTE দিয়ে পুরো tree আনা (MySQL 8.0+)
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth, CAST(name AS CHAR(500)) AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.depth + 1,
           CONCAT(ct.path, ' > ', c.name)
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

```php
// Laravel Self-Referencing
class Category extends Model
{
    public function parent(): BelongsTo
    {
        return $this->belongsTo(Category::class, 'parent_id');
    }

    public function children(): HasMany
    {
        return $this->hasMany(Category::class, 'parent_id');
    }

    // Recursive children (সব level-এর)
    public function allChildren(): HasMany
    {
        return $this->children()->with('allChildren');
    }

    // Root categories scope
    public function scopeRoot($query)
    {
        return $query->whereNull('parent_id');
    }
}

// Recursive tree load
$tree = Category::root()->with('allChildren')->get();
```

---

### ৩. Many-to-Many (M:N)

দুটি entity পরস্পরের সাথে multiple সম্পর্ক রাখতে পারে। উদাহরণ: একজন `User`-এর অনেক `Role`, একটি `Role` অনেক `User`-এর। এটি implement করতে একটি **pivot/junction table** প্রয়োজন।

#### SQL Schema — Pivot Table Design

```sql
CREATE TABLE roles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    guard_name VARCHAR(50) DEFAULT 'web',
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Pivot table: extra columns সহ
CREATE TABLE role_user (
    user_id BIGINT UNSIGNED NOT NULL,
    role_id BIGINT UNSIGNED NOT NULL,
    assigned_by BIGINT UNSIGNED NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE,

    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (assigned_by) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_role_id (role_id)
);

-- Complex M:N: Product ↔ Tag
CREATE TABLE tags (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(60) UNIQUE NOT NULL
);

CREATE TABLE taggables (
    tag_id BIGINT UNSIGNED NOT NULL,
    taggable_id BIGINT UNSIGNED NOT NULL,
    taggable_type VARCHAR(100) NOT NULL,
    PRIMARY KEY (tag_id, taggable_id, taggable_type),
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

-- User এর সব role সহ query
SELECT u.name, r.name AS role_name, ru.assigned_at, ru.expires_at
FROM users u
INNER JOIN role_user ru ON u.id = ru.user_id
INNER JOIN roles r ON ru.role_id = r.id
WHERE u.id = 1 AND ru.is_active = TRUE
  AND (ru.expires_at IS NULL OR ru.expires_at > NOW());

-- নির্দিষ্ট role-এর সব active user
SELECT u.id, u.name, u.email
FROM users u
INNER JOIN role_user ru ON u.id = ru.user_id
INNER JOIN roles r ON ru.role_id = r.id
WHERE r.name = 'vendor' AND ru.is_active = TRUE;
```

#### Order ↔ Product (Pivot with Quantity)

```sql
CREATE TABLE order_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL DEFAULT 1,
    unit_price DECIMAL(10,2) NOT NULL,
    discount DECIMAL(10,2) DEFAULT 0.00,
    subtotal DECIMAL(12,2) GENERATED ALWAYS AS ((unit_price - discount) * quantity) STORED,

    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT,
    UNIQUE KEY uk_order_product (order_id, product_id)
);
```

#### Laravel Eloquent

```php
// app/Models/User.php
class User extends Model
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)
            ->withPivot(['assigned_by', 'assigned_at', 'expires_at', 'is_active'])
            ->withTimestamps()
            ->wherePivot('is_active', true);
    }

    public function activeRoles(): BelongsToMany
    {
        return $this->roles()
            ->wherePivot('is_active', true)
            ->where(function ($q) {
                $q->whereNull('role_user.expires_at')
                  ->orWhere('role_user.expires_at', '>', now());
            });
    }

    public function hasRole(string $roleName): bool
    {
        return $this->activeRoles()->where('name', $roleName)->exists();
    }

    public function assignRole(string $roleName, ?int $assignedBy = null, ?Carbon $expiresAt = null): void
    {
        $role = Role::where('name', $roleName)->firstOrFail();
        $this->roles()->syncWithoutDetaching([
            $role->id => [
                'assigned_by' => $assignedBy,
                'assigned_at' => now(),
                'expires_at' => $expiresAt,
                'is_active' => true,
            ],
        ]);
    }
}

// app/Models/Order.php
class Order extends Model
{
    public function products(): BelongsToMany
    {
        return $this->belongsToMany(Product::class, 'order_items')
            ->withPivot(['quantity', 'unit_price', 'discount'])
            ->withTimestamps();
    }
}

// ব্যবহার
$user->roles()->attach($roleId, ['assigned_by' => auth()->id()]);
$user->roles()->detach($roleId);
$user->roles()->sync([1, 2, 3]); // শুধু এই role গুলো রাখো, বাকি সব detach

// Pivot access
foreach ($order->products as $product) {
    echo $product->pivot->quantity;
    echo $product->pivot->unit_price;
}
```

#### Sequelize

```javascript
// M:N Association
User.belongsToMany(Role, {
    through: 'role_user',
    foreignKey: 'userId',
    otherKey: 'roleId',
    as: 'roles',
});
Role.belongsToMany(User, {
    through: 'role_user',
    foreignKey: 'roleId',
    otherKey: 'userId',
    as: 'users',
});

// Pivot model সহ (extra columns access করতে)
const RoleUser = sequelize.define('RoleUser', {
    assignedBy: DataTypes.BIGINT.UNSIGNED,
    assignedAt: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },
    expiresAt: DataTypes.DATE,
    isActive: { type: DataTypes.BOOLEAN, defaultValue: true },
}, { tableName: 'role_user' });

User.belongsToMany(Role, { through: RoleUser, foreignKey: 'userId' });
Role.belongsToMany(User, { through: RoleUser, foreignKey: 'roleId' });

// Query with pivot data
const user = await User.findByPk(1, {
    include: [{
        model: Role,
        as: 'roles',
        through: {
            where: { isActive: true },
            attributes: ['assignedAt', 'expiresAt'],
        },
    }],
});

// Attach/Detach
await user.addRole(role, { through: { assignedBy: adminId } });
await user.removeRole(role);
await user.setRoles([role1, role2]); // sync equivalent
```

---

### ৪. Polymorphic Relationships

একটি model একাধিক ভিন্ন ধরনের model-এর সাথে সম্পর্ক রাখতে পারে — একই table structure ব্যবহার করে। `type` column entity-র ধরন এবং `id` column সেই entity-র primary key সংরক্ষণ করে।

**বাস্তব উদাহরণ:** Comments, Images, Likes, Tags — এগুলো Products, Users, Posts যেকোনো model-এ প্রযোজ্য।

#### SQL Schema

```sql
-- Polymorphic comments: Product, Order, BlogPost — যেকোনো কিছুতে comment
CREATE TABLE comments (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    commentable_type VARCHAR(100) NOT NULL,  -- 'Product', 'Order', 'BlogPost'
    commentable_id BIGINT UNSIGNED NOT NULL,
    body TEXT NOT NULL,
    parent_id BIGINT UNSIGNED NULL,          -- reply threading
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id) REFERENCES comments(id) ON DELETE CASCADE,
    INDEX idx_commentable (commentable_type, commentable_id),
    INDEX idx_parent_id (parent_id)
);

-- Polymorphic images
CREATE TABLE images (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    imageable_type VARCHAR(100) NOT NULL,
    imageable_id BIGINT UNSIGNED NOT NULL,
    path VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_imageable (imageable_type, imageable_id)
);

-- Product-এর সব image আনা
SELECT * FROM images
WHERE imageable_type = 'Product' AND imageable_id = 15
ORDER BY sort_order;

-- Polymorphic likes (M:N polymorphic)
CREATE TABLE likes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    likeable_type VARCHAR(100) NOT NULL,
    likeable_id BIGINT UNSIGNED NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uk_user_likeable (user_id, likeable_type, likeable_id),
    INDEX idx_likeable (likeable_type, likeable_id)
);
```

#### Laravel Eloquent

```php
// Polymorphic One-to-Many: comments
class Comment extends Model
{
    protected $fillable = ['user_id', 'body', 'parent_id'];

    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function replies(): HasMany
    {
        return $this->hasMany(Comment::class, 'parent_id');
    }
}

class Product extends Model
{
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }

    public function images(): MorphMany
    {
        return $this->morphMany(Image::class, 'imageable')->orderBy('sort_order');
    }

    // Polymorphic M:N: tags
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }

    // Polymorphic M:N: likes
    public function likes(): MorphMany
    {
        return $this->morphMany(Like::class, 'likeable');
    }

    public function isLikedBy(User $user): bool
    {
        return $this->likes()->where('user_id', $user->id)->exists();
    }
}

class Tag extends Model
{
    // Inverse polymorphic M:N
    public function products(): MorphedByMany
    {
        return $this->morphedByMany(Product::class, 'taggable');
    }

    public function posts(): MorphedByMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }
}

// ব্যবহার
$product->comments()->create(['user_id' => 1, 'body' => 'চমৎকার পণ্য!']);
$product->tags()->sync([1, 5, 12]);

// Morph map (performance + readability)
// AppServiceProvider::boot()
Relation::enforceMorphMap([
    'product' => Product::class,
    'order' => Order::class,
    'post' => Post::class,
]);
```

#### Sequelize Polymorphic Workaround

Sequelize-এ native polymorphic support নেই। Manual implementation:

```javascript
// Polymorphic Comment model
const Comment = sequelize.define('Comment', {
    userId: { type: DataTypes.BIGINT.UNSIGNED, allowNull: false },
    commentableType: { type: DataTypes.STRING(100), allowNull: false },
    commentableId: { type: DataTypes.BIGINT.UNSIGNED, allowNull: false },
    body: { type: DataTypes.TEXT, allowNull: false },
    parentId: { type: DataTypes.BIGINT.UNSIGNED, allowNull: true },
});

// Helper function: polymorphic association setup
function addPolymorphicComments(Model, modelName) {
    Model.hasMany(Comment, {
        foreignKey: 'commentableId',
        constraints: false,
        scope: { commentableType: modelName },
        as: 'comments',
    });

    // Hook: auto-set commentableType
    Comment.addHook('beforeFind', (options) => {
        if (options.include) {
            options.include.forEach((inc) => {
                if (inc.model === Comment && inc.where) {
                    inc.where.commentableType = inc.where.commentableType || modelName;
                }
            });
        }
    });
}

addPolymorphicComments(Product, 'Product');
addPolymorphicComments(Order, 'Order');

// Query
const product = await Product.findByPk(15, {
    include: [{
        model: Comment,
        as: 'comments',
        where: { commentableType: 'Product' },
        include: [{ model: User, attributes: ['name'] }],
    }],
});
```

#### Polymorphic — সুবিধা ও অসুবিধা

| দিক | Polymorphic | Dedicated FK |
|------|------------|--------------|
| Flexibility | ✅ যেকোনো model-এ apply | ❌ প্রতিটি model-এর জন্য আলাদা table |
| FK Constraint | ❌ DB-level enforce অসম্ভব | ✅ পূর্ণ referential integrity |
| Query Performance | ⚠️ type+id composite index দরকার | ✅ simple FK join |
| Maintenance | ✅ একটি table/model | ❌ প্রতিটি relation-এর জন্য table |

---

### ৫. Self-Referencing Relationships

একটি table নিজের সাথেই relationship রাখে। Hierarchical data store করার জন্য বিভিন্ন strategy আছে:

#### Strategy ১: Adjacency List (সবচেয়ে সহজ)

```sql
-- প্রতিটি node তার parent-কে reference করে
CREATE TABLE employees (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id BIGINT UNSIGNED NULL,
    designation VARCHAR(100),
    department VARCHAR(50),
    FOREIGN KEY (manager_id) REFERENCES employees(id) ON DELETE SET NULL,
    INDEX idx_manager (manager_id)
);

-- Direct reports of a manager
SELECT * FROM employees WHERE manager_id = 5;

-- Full hierarchy (MySQL 8.0+ Recursive CTE)
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, designation, 0 AS level
    FROM employees WHERE manager_id IS NULL  -- CEO

    UNION ALL

    SELECT e.id, e.name, e.manager_id, e.designation, oc.level + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT CONCAT(REPEAT('  ', level), name) AS hierarchy, designation
FROM org_chart
ORDER BY level, name;
```

```php
// Laravel Adjacency List
class Employee extends Model
{
    public function manager(): BelongsTo
    {
        return $this->belongsTo(Employee::class, 'manager_id');
    }

    public function directReports(): HasMany
    {
        return $this->hasMany(Employee::class, 'manager_id');
    }

    public function allSubordinates(): HasMany
    {
        return $this->directReports()->with('allSubordinates');
    }
}
```

#### Strategy ২: Materialized Path

```sql
-- path column-এ পূর্বপুরুষদের chain সংরক্ষিত থাকে
CREATE TABLE categories_mp (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    path VARCHAR(500) NOT NULL DEFAULT '/',  -- '/1/5/12/'
    depth INT UNSIGNED DEFAULT 0,
    INDEX idx_path (path)
);

-- "Electronics > Mobile > Samsung" hierarchy:
-- id=1, path='/', depth=0             (Electronics)
-- id=5, path='/1/', depth=1            (Mobile)
-- id=12, path='/1/5/', depth=2         (Samsung)

-- সব descendant আনা (LIKE query — খুবই দ্রুত)
SELECT * FROM categories_mp WHERE path LIKE '/1/5/%';

-- সব ancestor আনা
SELECT * FROM categories_mp
WHERE '/1/5/12/' LIKE CONCAT(path, id, '/%')
ORDER BY depth;
```

#### Strategy ৩: Nested Set

```sql
CREATE TABLE categories_ns (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lft INT UNSIGNED NOT NULL,
    rgt INT UNSIGNED NOT NULL,
    depth INT UNSIGNED DEFAULT 0,
    INDEX idx_lft_rgt (lft, rgt)
);

--         Electronics (1, 18)
--        /                    \
--   Mobile (2, 9)         Laptop (10, 17)
--   /          \           /          \
-- Samsung(3,6) iPhone(7,8) Dell(11,14) HP(15,16)
-- /    \                    /    \
-- S24(4,5)               Inspiron(12,13)

-- সব descendant (অত্যন্ত দ্রুত read)
SELECT * FROM categories_ns
WHERE lft BETWEEN 2 AND 9  -- Mobile-এর সব child
ORDER BY lft;

-- Node-এর depth ও ancestor count
SELECT child.name, COUNT(parent.id) - 1 AS depth
FROM categories_ns AS child, categories_ns AS parent
WHERE child.lft BETWEEN parent.lft AND parent.rgt
GROUP BY child.id, child.name
ORDER BY child.lft;
```

#### Strategy ৪: Closure Table

```sql
-- মূল table
CREATE TABLE categories_ct (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Closure table: প্রতিটি ancestor-descendant pair সংরক্ষিত
CREATE TABLE category_closure (
    ancestor_id BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth INT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES categories_ct(id) ON DELETE CASCADE,
    FOREIGN KEY (descendant_id) REFERENCES categories_ct(id) ON DELETE CASCADE,
    INDEX idx_descendant (descendant_id)
);

-- Node insert করলে closure entry যোগ করতে হয়:
-- Self-reference (depth=0) + সব ancestor-এর সাথে link
INSERT INTO category_closure (ancestor_id, descendant_id, depth)
    SELECT ancestor_id, NEW_NODE_ID, depth + 1
    FROM category_closure
    WHERE descendant_id = PARENT_ID
    UNION ALL
    SELECT NEW_NODE_ID, NEW_NODE_ID, 0;

-- সব descendant আনা
SELECT c.* FROM categories_ct c
INNER JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 1 AND cc.depth > 0
ORDER BY cc.depth;

-- সব ancestor আনা
SELECT c.* FROM categories_ct c
INNER JOIN category_closure cc ON c.id = cc.ancestor_id
WHERE cc.descendant_id = 12 AND cc.depth > 0
ORDER BY cc.depth DESC;
```

#### Tree Strategy তুলনা

```
┌────────────────────┬─────────────┬─────────────┬────────────┬──────────────┐
│ Feature            │ Adjacency   │ Materialized│ Nested Set │ Closure      │
│                    │ List        │ Path        │            │ Table        │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ Read Subtree       │ Recursive   │ LIKE query  │ BETWEEN    │ JOIN         │
│                    │ CTE (slow)  │ (fast)      │ (fastest)  │ (fast)       │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ Insert/Move Node   │ ✅ সহজ      │ ⚠️ path     │ ❌ অনেক     │ ⚠️ closure    │
│                    │             │ update      │ update     │ update       │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ Delete Node        │ ✅ সহজ      │ ⚠️ মাঝারি   │ ❌ কঠিন     │ ✅ CASCADE    │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ Referential Int.   │ ✅ FK       │ ❌ path str  │ ❌ lft/rgt  │ ✅ FK        │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ Storage            │ ✅ O(n)     │ ✅ O(n)     │ ✅ O(n)    │ ⚠️ O(n²)     │
├────────────────────┼─────────────┼─────────────┼────────────┼──────────────┤
│ সেরা ব্যবহার       │ অগভীর tree, │ URL path,   │ Read-heavy │ সব ধরনের     │
│                    │ ঘন write    │ breadcrumb  │ tree       │ balanced use │
└────────────────────┴─────────────┴─────────────┴────────────┴──────────────┘
```

#### Sequelize — Self-Referencing

```javascript
// Adjacency List
const Employee = sequelize.define('Employee', {
    name: DataTypes.STRING(100),
    managerId: {
        type: DataTypes.BIGINT.UNSIGNED,
        references: { model: 'employees', key: 'id' },
    },
    designation: DataTypes.STRING(100),
});

Employee.hasMany(Employee, { as: 'directReports', foreignKey: 'managerId' });
Employee.belongsTo(Employee, { as: 'manager', foreignKey: 'managerId' });

// Recursive load (নির্দিষ্ট depth পর্যন্ত)
async function getOrgChart(managerId, maxDepth = 5) {
    function buildInclude(depth) {
        if (depth === 0) return [];
        return [{
            model: Employee,
            as: 'directReports',
            include: buildInclude(depth - 1),
        }];
    }
    return Employee.findByPk(managerId, {
        include: buildInclude(maxDepth),
    });
}
```

---

### ৬. Through / HasManyThrough Relationships

দুটি table-এর মধ্যে তৃতীয় table-এর মাধ্যমে সম্পর্ক — intermediate table skip করে সরাসরি query।

#### SQL

```sql
-- Country → Users → Orders (Country-র সব order সরাসরি)
-- Country → Users → Posts  (Country-র সব post সরাসরি)

CREATE TABLE countries (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code CHAR(2) UNIQUE NOT NULL  -- 'BD', 'IN', 'US'
);

-- users table-এ country_id FK আছে
ALTER TABLE users ADD COLUMN country_id BIGINT UNSIGNED,
    ADD FOREIGN KEY (country_id) REFERENCES countries(id);

-- Bangladesh-এর সব order সরাসরি query
SELECT o.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN countries c ON u.country_id = c.id
WHERE c.code = 'BD'
ORDER BY o.created_at DESC;

-- Country-wise order statistics
SELECT c.name AS country, COUNT(o.id) AS total_orders, SUM(o.total) AS revenue
FROM countries c
LEFT JOIN users u ON c.id = u.country_id
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY revenue DESC;
```

#### Laravel

```php
class Country extends Model
{
    public function users(): HasMany
    {
        return $this->hasMany(User::class);
    }

    // Through relationship: Country → Users → Orders
    public function orders(): HasManyThrough
    {
        return $this->hasManyThrough(
            Order::class,   // Final model
            User::class,    // Intermediate model
            'country_id',   // FK on users table
            'user_id',      // FK on orders table
            'id',           // Local key on countries
            'id'            // Local key on users
        );
    }

    // Country → Users → Reviews
    public function reviews(): HasManyThrough
    {
        return $this->hasManyThrough(Review::class, User::class);
    }

    // hasOneThrough: Country → User → Latest Profile
    public function latestUserProfile(): HasOneThrough
    {
        return $this->hasOneThrough(
            Profile::class,
            User::class
        )->latestOfMany();
    }
}

// ব্যবহার
$bangladesh = Country::where('code', 'BD')->first();
$orders = $bangladesh->orders()->where('status', 'delivered')->paginate(20);
$orderCount = $bangladesh->orders()->count();

// Eager load
$countries = Country::withCount('orders')
    ->with(['orders' => fn($q) => $q->latest()->limit(5)])
    ->get();
```

---

## 🔥 Advanced Scenarios

### ১. Eager Loading ও N+1 Problem

**N+1 Problem** হলো সবচেয়ে সাধারণ performance bottleneck। একটি parent query + প্রতিটি child-এর জন্য আলাদা query = N+1 queries।

```php
// ❌ N+1 Problem: 1 query for users + N queries for orders
$users = User::all(); // SELECT * FROM users (1 query)
foreach ($users as $user) {
    echo $user->orders->count(); // SELECT * FROM orders WHERE user_id = ? (N queries)
}

// ✅ Eager Loading: মাত্র 2 queries
$users = User::with('orders')->get();
// SELECT * FROM users
// SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...)

// ✅ Nested Eager Loading
$users = User::with([
    'orders.items.product.category',
    'orders.items.product.images',
    'profile',
])->get();

// ✅ Conditional Eager Loading
$users = User::with(['orders' => function ($query) {
    $query->where('status', 'delivered')
          ->where('created_at', '>=', now()->subYear())
          ->withCount('items')
          ->latest();
}])->get();

// ✅ Lazy Eager Loading (পরে load করা)
$users = User::all();
if ($needOrders) {
    $users->load('orders');
}

// ✅ withCount — relation count without loading
$products = Product::withCount(['reviews', 'comments', 'likes'])
    ->withAvg('reviews', 'rating')
    ->having('reviews_count', '>=', 5)
    ->orderByDesc('reviews_avg_rating')
    ->paginate(20);
```

#### Sequelize-এ Eager Loading

```javascript
// ❌ N+1
const users = await User.findAll();
for (const user of users) {
    const orders = await user.getOrders(); // N+1!
}

// ✅ Eager Loading
const users = await User.findAll({
    include: [{
        model: Order,
        as: 'orders',
        where: { status: 'delivered' },
        required: false,
        include: [{
            model: OrderItem,
            as: 'items',
            include: [{
                model: Product,
                attributes: ['id', 'name', 'price'],
                include: [{ model: Category, attributes: ['name'] }],
            }],
        }],
    }, {
        model: Profile,
        attributes: ['avatar', 'division'],
    }],
    order: [[{ model: Order, as: 'orders' }, 'createdAt', 'DESC']],
});

// Knex.js — manual eager loading alternative
const users = await knex('users').select('*');
const userIds = users.map(u => u.id);
const orders = await knex('orders')
    .whereIn('user_id', userIds)
    .orderBy('created_at', 'desc');

// Group orders by user
const ordersByUser = orders.reduce((acc, order) => {
    (acc[order.user_id] = acc[order.user_id] || []).push(order);
    return acc;
}, {});

users.forEach(user => {
    user.orders = ordersByUser[user.id] || [];
});
```

#### N+1 Detection

```php
// Laravel: barryvdh/laravel-debugbar বা beyondcode/laravel-query-detector
// .env
QUERY_DETECTOR_ENABLED=true

// Laravel Telescope: query tab-এ duplicate queries দেখুন

// Manual detection: DB::listen()
DB::listen(function ($query) {
    if (str_contains($query->sql, 'orders') && $query->time > 100) {
        Log::warning('Slow order query', [
            'sql' => $query->sql,
            'time' => $query->time,
            'bindings' => $query->bindings,
        ]);
    }
});
```

---

### ২. Database Constraints

```sql
-- Comprehensive constraint examples
CREATE TABLE products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    category_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(220) UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    compare_price DECIMAL(10,2) NULL,
    stock INT UNSIGNED NOT NULL DEFAULT 0,
    sku VARCHAR(50) UNIQUE NOT NULL,
    status ENUM('active','inactive','draft') DEFAULT 'draft',
    weight_grams INT UNSIGNED,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Foreign Key
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT ON UPDATE CASCADE,

    -- Check Constraints (MySQL 8.0.16+)
    CONSTRAINT chk_price_positive CHECK (price > 0),
    CONSTRAINT chk_compare_price CHECK (compare_price IS NULL OR compare_price >= price),
    CONSTRAINT chk_stock_non_negative CHECK (stock >= 0),
    CONSTRAINT chk_weight CHECK (weight_grams IS NULL OR weight_grams > 0),

    -- Composite Unique (একই category-তে duplicate name নয়)
    UNIQUE KEY uk_category_name (category_id, name),

    -- Indexes
    INDEX idx_status (status),
    INDEX idx_price (price),
    FULLTEXT INDEX ft_name (name)
);

-- Partial unique index (PostgreSQL)
-- শুধু active product-এ unique slug enforce করা
CREATE UNIQUE INDEX idx_active_slug ON products (slug)
WHERE status = 'active';
```

---

### ৩. Soft Deletes ও Relationships

Soft delete মানে record আসলে মুছে না, `deleted_at` column-এ timestamp set হয়। এটি relationships-এ complexity যোগ করে।

```sql
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMP NULL DEFAULT NULL;
CREATE INDEX idx_deleted_at ON products (deleted_at);

-- Soft deleted রেকর্ড বাদ দিয়ে query (default behavior)
SELECT * FROM products WHERE deleted_at IS NULL;

-- Soft deleted সহ সব রেকর্ড
SELECT * FROM products;

-- শুধু soft deleted রেকর্ড
SELECT * FROM products WHERE deleted_at IS NOT NULL;
```

```php
// Laravel Soft Deletes
class Product extends Model
{
    use SoftDeletes;

    protected $dates = ['deleted_at'];

    public function reviews(): HasMany
    {
        return $this->hasMany(Review::class);
    }

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class)->withTrashed();
        // Category soft-deleted হলেও এই relationship কাজ করবে
    }
}

// Cascading Soft Deletes
class Order extends Model
{
    use SoftDeletes;

    protected static function booted(): void
    {
        static::deleting(function (Order $order) {
            if ($order->isForceDeleting()) {
                $order->items()->forceDelete();
            } else {
                $order->items()->each->delete(); // cascade soft delete
            }
        });

        static::restoring(function (Order $order) {
            $order->items()->withTrashed()->restore();
        });
    }
}

// Query variations
$activeProducts = Product::all();                          // soft deleted বাদ
$allProducts = Product::withTrashed()->get();              // সব (soft deleted সহ)
$trashedOnly = Product::onlyTrashed()->get();              // শুধু soft deleted
$product->trashed();                                       // soft deleted কিনা check
$product->restore();                                       // restore করা
$product->forceDelete();                                   // permanently delete
```

---

### ৪. JSON Relationships (Semi-structured)

কখনো কখনো relational model-এর পরিবর্তে JSON column-এ related data store করা হয়। এটি schema flexibility দেয় কিন্তু referential integrity হারায়।

```sql
-- Product-এ specifications JSON হিসেবে (MySQL 5.7+)
CREATE TABLE products_v2 (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    specifications JSON,
    tag_ids JSON,    -- [1, 5, 12] — related tag IDs
    attributes JSON  -- {"color": "red", "size": "XL", "material": "cotton"}
);

-- JSON query examples
SELECT * FROM products_v2
WHERE JSON_CONTAINS(tag_ids, '5');

SELECT *, JSON_EXTRACT(specifications, '$.ram') AS ram
FROM products_v2
WHERE JSON_EXTRACT(specifications, '$.ram') >= 8
  AND JSON_EXTRACT(attributes, '$.color') = '"red"';

-- JSON_TABLE: JSON array কে rows-এ convert (MySQL 8.0+)
SELECT p.name, t.tag_name
FROM products_v2 p,
JSON_TABLE(p.tag_ids, '$[*]' COLUMNS (tag_id BIGINT PATH '$')) AS jt
INNER JOIN tags t ON jt.tag_id = t.id;
```

```php
// Laravel JSON casting
class Product extends Model
{
    protected $casts = [
        'specifications' => 'array',
        'tag_ids' => 'array',
        'attributes' => AsCollection::class,
    ];

    public function getTagsAttribute()
    {
        return Tag::whereIn('id', $this->tag_ids ?? [])->get();
    }
}

// Query JSON columns
Product::whereJsonContains('tag_ids', 5)->get();
Product::where('specifications->ram', '>=', 8)->get();
Product::where('attributes->color', 'red')->get();
```

**কখন JSON relationship ব্যবহার করবেন:**
- ✅ Semi-structured, schema-less data (product specifications)
- ✅ Read-heavy, কদাচিৎ join দরকার
- ❌ Referential integrity দরকার হলে ব্যবহার করবেন না
- ❌ Frequently updated relational data-তে ব্যবহার করবেন না
- ❌ Complex query ও JOIN দরকার হলে ব্যবহার করবেন না

---

### ৫. Cross-Database ও Microservice Relationships

বড় system-এ data বিভিন্ন database বা service-এ ছড়িয়ে থাকে। তখন traditional FK কাজ করে না।

```php
// Laravel: Cross-database relationship
class Order extends Model
{
    protected $connection = 'mysql_orders'; // orders database

    public function user(): BelongsTo
    {
        return $this->setConnection('mysql_users') // users database
            ->belongsTo(User::class);
    }
}

// Microservice: API-based relationship
class OrderService
{
    public function getOrderWithUser(int $orderId): array
    {
        $order = Order::findOrFail($orderId);

        // User service API call
        $user = Http::get("http://user-service/api/users/{$order->user_id}")
            ->throw()
            ->json();

        return array_merge($order->toArray(), ['user' => $user]);
    }
}
```

```javascript
// Knex.js: Cross-database query
const orders = await knex('orders_db.orders as o')
    .join('users_db.users as u', 'o.user_id', 'u.id')
    .select('o.*', 'u.name as customer_name')
    .where('o.status', 'delivered');

// API-based relationship (microservice)
async function getOrderWithUser(orderId) {
    const order = await knex('orders').where('id', orderId).first();
    const userResponse = await fetch(
        `${process.env.USER_SERVICE_URL}/api/users/${order.userId}`
    );
    const user = await userResponse.json();
    return { ...order, user };
}
```

**Eventual Consistency:** Microservice-এ data সবসময় immediately consistent থাকে না। Event-driven approach ব্যবহার করুন:

```
Order Service → publishes "OrderCreated" event
   ↓
Message Queue (RabbitMQ / Kafka)
   ↓
Inventory Service → stock decrease
Notification Service → email/SMS পাঠানো
Analytics Service → report update
```

---

## 🆚 Relationship Pattern Comparison Table

```
┌──────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ বৈশিষ্ট্য        │ One-to-One   │ One-to-Many  │ Many-to-Many │ Polymorphic  │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ FK অবস্থান       │ Child table  │ Child table  │ Pivot table  │ Morph table  │
│                  │ (UNIQUE FK)  │              │              │ (type + id)  │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ FK Constraint    │ ✅ হ্যাঁ      │ ✅ হ্যাঁ      │ ✅ হ্যাঁ      │ ❌ না        │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Join Count       │ 1            │ 1            │ 2            │ 1 + filter   │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Query Complexity │ Low          │ Low          │ Medium       │ Medium-High  │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Flexibility      │ Low          │ Medium       │ High         │ Very High    │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Use Case         │ Profile,     │ Orders,      │ Roles, Tags, │ Comments,    │
│                  │ Invoice      │ Comments     │ Categories   │ Images, Likes│
└──────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## ✅ সুবিধা ও ❌ অসুবিধা

### One-to-One
- ✅ Data isolation ও security (sensitive data আলাদা table-এ)
- ✅ Table-এর width কমায় (vertical partitioning)
- ✅ Optional data efficiently store করা যায়
- ❌ Extra JOIN overhead
- ❌ অতিরিক্ত ব্যবহারে unnecessary complexity

### One-to-Many
- ✅ সবচেয়ে intuitive ও সাধারণ pattern
- ✅ Strong referential integrity
- ✅ Efficient indexing ও query
- ❌ Deep nesting-এ multiple JOIN দরকার
- ❌ Cascade delete সতর্কতার সাথে ব্যবহার করতে হয়

### Many-to-Many
- ✅ Complex real-world relationship model করতে পারে
- ✅ Pivot table-এ extra metadata রাখা যায়
- ❌ Pivot table বড় হতে পারে (performance impact)
- ❌ Three-way JOIN — query complexity বাড়ে
- ❌ Data integrity maintain করা জটিল

### Polymorphic
- ✅ Maximum flexibility — একটি model অনেক model-এর সাথে relate
- ✅ কম table, কম migration
- ✅ DRY principle — duplicate table এড়ানো
- ❌ Database-level FK constraint অসম্ভব
- ❌ Query debugging কঠিন
- ❌ String-based type matching — typo ঝুঁকি

### Self-Referencing
- ✅ Hierarchical data একটি table-এ store
- ✅ Recursive CTE দিয়ে powerful query সম্ভব
- ❌ Tree manipulation complex (বিশেষত Nested Set)
- ❌ Infinite loop ঝুঁকি (circular reference)

---

## ⚠️ সাধারণ ভুল ও সমাধান

### ১. Foreign Key না দেওয়া

```sql
-- ❌ ভুল: FK constraint ছাড়া — orphan record তৈরি হবে
CREATE TABLE orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,  -- কোনো FK নেই!
    total DECIMAL(10,2)
);

-- ✅ সঠিক: FK constraint সহ
CREATE TABLE orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    total DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### ২. Cascade সঠিকভাবে সেট না করা

```sql
-- ❌ ভুল: RESTRICT default — user মোছা যাবে না যতক্ষণ order আছে
-- কখনো কখনো এটাই চাওয়া হয়, তবে সচেতনভাবে সিদ্ধান্ত নিতে হবে

-- ⚠️ বিপজ্জনক: সব জায়গায় CASCADE — user delete করলে সব data হারাবে
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
-- user মুছলে → সব order মুছবে → সব order_items মুছবে → সব reviews মুছবে

-- ✅ ভালো approach: soft delete + selective cascade
-- Critical data-তে RESTRICT, non-critical-এ CASCADE
```

### ৩. Circular Reference

```sql
-- ❌ Circular: A → B → A
-- employees.department_id → departments.id
-- departments.head_id → employees.id
-- এটি migration ও seeding-এ সমস্যা তৈরি করে

-- ✅ সমাধান: একটি FK nullable করুন
ALTER TABLE departments ADD COLUMN head_id BIGINT UNSIGNED NULL;
-- প্রথমে department create, তারপর head assign
```

### ৪. N+1 Query উপেক্ষা করা

```php
// ❌ ভুল — blade template-এ N+1
@foreach($products as $product)
    <p>Category: {{ $product->category->name }}</p>  {{-- প্রতিটি loop-এ query --}}
@endforeach

// ✅ সঠিক — Controller-এ eager load
$products = Product::with('category')->paginate(20);
```

### ৫. Index না দেওয়া

```sql
-- ❌ FK column-এ index না থাকলে JOIN অত্যন্ত ধীর হবে
-- MySQL InnoDB FK-তে automatic index দেয়, কিন্তু composite query-র জন্য:

-- ✅ Composite index for common queries
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

-- polymorphic relationship-এ composite index অপরিহার্য
CREATE INDEX idx_commentable ON comments (commentable_type, commentable_id);
```

### ৬. Pivot Table-এ ID না রাখা

```sql
-- ⚠️ Composite PK যথেষ্ট — কিন্তু extra columns ও soft delete থাকলে:
-- ✅ Surrogate PK রাখা ভালো
CREATE TABLE order_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- আলাদা PK
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL,
    UNIQUE KEY (order_id, product_id),  -- uniqueness enforce
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
);
```

---

## 🧪 টেস্টিং Relationships

### Laravel Factory ও Seeder

```php
// database/factories/UserFactory.php
class UserFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'phone' => fake()->unique()->numerify('01#########'), // BD phone
            'password' => Hash::make('password'),
        ];
    }

    // State: user সহ profile
    public function withProfile(): static
    {
        return $this->afterCreating(function (User $user) {
            Profile::factory()->for($user)->create();
        });
    }
}

// database/factories/OrderFactory.php
class OrderFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'status' => fake()->randomElement(['pending', 'confirmed', 'delivered']),
            'total' => fake()->randomFloat(2, 100, 50000),
            'shipping_address' => fake()->address(),
            'payment_method' => fake()->randomElement(['bkash', 'nagad', 'cod']),
        ];
    }

    // State: order সহ items
    public function withItems(int $count = 3): static
    {
        return $this->afterCreating(function (Order $order) use ($count) {
            OrderItem::factory()
                ->count($count)
                ->for($order)
                ->create();
        });
    }
}

// database/seeders/EcommerceSeeder.php
class EcommerceSeeder extends Seeder
{
    public function run(): void
    {
        // Categories with hierarchy
        $electronics = Category::create(['name' => 'Electronics', 'slug' => 'electronics']);
        $mobile = Category::create(['name' => 'Mobile', 'slug' => 'mobile', 'parent_id' => $electronics->id]);

        // Products
        $products = Product::factory()->count(50)
            ->recycle(Category::all())  // existing category ব্যবহার করো
            ->create();

        // Users with orders
        User::factory()
            ->count(100)
            ->withProfile()
            ->has(
                Order::factory()
                    ->count(rand(1, 10))
                    ->withItems(rand(1, 5))
            )
            ->create();

        // Roles ও assignment
        $roles = collect(['admin', 'vendor', 'customer'])->map(
            fn($name) => Role::create(['name' => $name])
        );
        User::all()->each(fn($user) => $user->roles()->attach(
            $roles->random()->id, ['assigned_at' => now()]
        ));
    }
}
```

### PHPUnit Relationship Tests

```php
// tests/Feature/RelationshipTest.php
class RelationshipTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_has_one_profile(): void
    {
        $user = User::factory()->withProfile()->create();

        $this->assertInstanceOf(Profile::class, $user->profile);
        $this->assertEquals($user->id, $user->profile->user_id);
    }

    public function test_user_has_many_orders(): void
    {
        $user = User::factory()->create();
        Order::factory()->count(5)->for($user)->create();

        $this->assertCount(5, $user->orders);
        $this->assertInstanceOf(Order::class, $user->orders->first());
    }

    public function test_many_to_many_roles(): void
    {
        $user = User::factory()->create();
        $role = Role::factory()->create(['name' => 'vendor']);

        $user->roles()->attach($role->id, ['assigned_at' => now()]);

        $this->assertTrue($user->roles->contains($role));
        $this->assertNotNull($user->roles->first()->pivot->assigned_at);
    }

    public function test_cascade_delete_removes_orders(): void
    {
        $user = User::factory()->create();
        Order::factory()->count(3)->for($user)->create();

        $this->assertDatabaseCount('orders', 3);

        $user->delete();

        $this->assertDatabaseCount('orders', 0); // CASCADE
    }

    public function test_polymorphic_comments(): void
    {
        $product = Product::factory()->create();
        $user = User::factory()->create();

        $comment = $product->comments()->create([
            'user_id' => $user->id,
            'body' => 'চমৎকার পণ্য!',
        ]);

        $this->assertInstanceOf(Product::class, $comment->commentable);
        $this->assertEquals('Product', $comment->commentable_type);
    }

    public function test_self_referencing_categories(): void
    {
        $parent = Category::factory()->create(['name' => 'Electronics']);
        $child = Category::factory()->create([
            'name' => 'Mobile',
            'parent_id' => $parent->id,
        ]);

        $this->assertTrue($parent->children->contains($child));
        $this->assertEquals($parent->id, $child->parent->id);
    }

    public function test_eager_loading_prevents_n_plus_one(): void
    {
        User::factory()->count(10)->has(Order::factory()->count(5))->create();

        DB::enableQueryLog();

        $users = User::with('orders')->get();
        $users->each(fn($u) => $u->orders->count());

        $this->assertCount(2, DB::getQueryLog()); // শুধু 2টি query
    }

    public function test_has_many_through(): void
    {
        $country = Country::factory()->create(['code' => 'BD']);
        $user = User::factory()->create(['country_id' => $country->id]);
        Order::factory()->count(3)->for($user)->create();

        $this->assertCount(3, $country->orders);
    }
}
```

### Sequelize Test (Jest)

```javascript
describe('Relationship Tests', () => {
    beforeAll(async () => {
        await sequelize.sync({ force: true });
    });

    test('User hasOne Profile', async () => {
        const user = await User.create({ name: 'Rahim', email: 'rahim@test.com' });
        const profile = await Profile.create({ userId: user.id, division: 'Dhaka' });

        const found = await User.findByPk(user.id, { include: [Profile] });
        expect(found.Profile).toBeDefined();
        expect(found.Profile.division).toBe('Dhaka');
    });

    test('User hasMany Orders', async () => {
        const user = await User.create({ name: 'Karim', email: 'karim@test.com' });
        await Order.bulkCreate([
            { userId: user.id, total: 1500, status: 'pending' },
            { userId: user.id, total: 3200, status: 'delivered' },
        ]);

        const found = await User.findByPk(user.id, {
            include: [{ model: Order, as: 'orders' }],
        });
        expect(found.orders).toHaveLength(2);
    });

    test('Many-to-Many User <-> Role', async () => {
        const user = await User.create({ name: 'Admin', email: 'admin@test.com' });
        const role = await Role.create({ name: 'admin' });

        await user.addRole(role, { through: { assignedAt: new Date() } });

        const found = await User.findByPk(user.id, {
            include: [{ model: Role, as: 'roles' }],
        });
        expect(found.roles.map(r => r.name)).toContain('admin');
    });

    test('Cascade delete removes child records', async () => {
        const user = await User.create({ name: 'Test', email: 'cascade@test.com' });
        await Order.create({ userId: user.id, total: 500, status: 'pending' });

        await user.destroy();

        const orders = await Order.findAll({ where: { userId: user.id } });
        expect(orders).toHaveLength(0);
    });
});
```

---

## 📋 সারসংক্ষেপ

### মূল নীতিমালা

১. **সঠিক relationship বেছে নিন** — business logic অনুযায়ী 1:1, 1:N, M:N বা polymorphic নির্বাচন করুন।

২. **সবসময় FK constraint দিন** — referential integrity ছাড়া ডাটাবেস অচিরেই inconsistent হয়ে যায়।

৩. **Cascade সচেতনভাবে সেট করুন** — প্রতিটি relationship-এ DELETE ও UPDATE action সুচিন্তিতভাবে নির্ধারণ করুন।

৪. **Eager loading ব্যবহার করুন** — N+1 problem সবচেয়ে সাধারণ performance killer। `with()` ও `include` ব্যবহার করুন।

৫. **Index দিন** — FK column, frequently filtered column, ও composite query column-এ index অপরিহার্য।

৬. **Polymorphic সতর্কতার সাথে** — flexibility অনেক বেশি, কিন্তু DB-level integrity হারায়। Critical data-তে dedicated FK ব্যবহার করুন।

৭. **Self-referencing strategy** — shallow tree-তে Adjacency List, deep read-heavy tree-তে Closure Table বা Nested Set বিবেচনা করুন।

৮. **Test করুন** — relationship, cascade, constraint — সব কিছু automated test-এ cover রাখুন।

### Quick Decision Guide

```
নতুন relationship ডিজাইন করতে হবে?
│
├─ প্রতিটি A-র ঠিক একটি B? ──────────────────→ One-to-One
├─ প্রতিটি A-র অনেক B, কিন্তু B-র একটি A? ──→ One-to-Many
├─ A-র অনেক B, B-রও অনেক A? ─────────────────→ Many-to-Many (pivot table)
├─ B একাধিক ভিন্ন ধরনের model-এর সাথে relate? → Polymorphic
├─ A নিজের সাথেই relate (hierarchy)? ─────────→ Self-Referencing
└─ A → C, কিন্তু B-এর মধ্য দিয়ে? ────────────→ HasManyThrough
```

> **মনে রাখুন:** ডাটাবেস relationship ডিজাইন একটি trade-off game। Performance, flexibility, integrity, ও complexity — এদের মধ্যে সঠিক balance খুঁজে বের করাই একজন senior engineer-এর কাজ। Bangladesh-এর e-commerce context-এ — bKash/Nagad payment, বিভাগভিত্তিক shipping, multi-vendor marketplace — এই সব বিবেচনায় রেখে schema design করুন।
