# 📖 API ডকুমেন্টেশন — সম্পূর্ণ গাইড

## 📌 সংজ্ঞা ও গুরুত্ব

API ডকুমেন্টেশন হলো একটি **টেকনিক্যাল রেফারেন্স ম্যানুয়াল** যা ডেভেলপারদের বলে দেয় কীভাবে একটি API ব্যবহার করতে হবে — কোন endpoint-এ কী পাঠাতে হবে, কী ফেরত আসবে, কোন authentication লাগবে, এবং error হলে কী হবে।

### কেন API ডকুমেন্টেশন গুরুত্বপূর্ণ?

```
১. Developer Experience (DX): ভালো ডক = দ্রুত integration = সুখী ডেভেলপার
২. Onboarding Time কমায়: নতুন ডেভেলপার ৩ দিনের বদলে ৩ ঘণ্টায় শুরু করতে পারে
৩. Support Ticket কমায়: ৮০% প্রশ্নের উত্তর ডকুমেন্টেশনেই থাকে
৪. API Adoption বাড়ায়: bKash, SSLCommerz-এর মতো পেমেন্ট গেটওয়ে ভালো ডক দিলে বেশি merchant ব্যবহার করে
৫. Contract হিসেবে কাজ করে: Frontend ও Backend টিম একসাথে parallel কাজ করতে পারে
```

### Documentation-First vs Code-First

**Documentation-First (Design-First):** আগে OpenAPI spec লিখি, তারপর কোড করি। এটি একটি **contract** তৈরি করে যা উভয় পক্ষ (consumer + provider) মেনে চলে।

**Code-First:** আগে কোড লিখি, তারপর annotation/decorator থেকে ডক generate করি। দ্রুত prototyping-এর জন্য সুবিধাজনক, তবে ডক outdated হওয়ার ঝুঁকি বেশি।

```
Documentation-First:
  ✅ Frontend-Backend parallel development সম্ভব
  ✅ API design review সহজ
  ✅ Mock server তৈরি করা যায় spec থেকে
  ❌ Initial overhead বেশি
  ❌ Spec আর code sync রাখতে discipline লাগে

Code-First:
  ✅ দ্রুত development শুরু করা যায়
  ✅ Code আর doc সবসময় sync থাকে (যদি annotation ঠিক থাকে)
  ❌ API design review করা কঠিন (কোড না দেখে বোঝা যায় না)
  ❌ Annotation ভুলে গেলে doc incomplete হয়
```

---

## 📊 API Documentation Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│                   API DOCUMENTATION WORKFLOW                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐     ┌──────────────────┐     ┌──────────────┐  │
│  │  Requirements    │────▶│  API Design      │────▶│  OpenAPI     │  │
│  │  Analysis        │     │  (endpoints,     │     │  Spec লেখা   │  │
│  │  (কী কী API     │     │   schemas,       │     │  (YAML/JSON) │  │
│  │   লাগবে?)       │     │   auth)          │     │              │  │
│  └─────────────────┘     └──────────────────┘     └──────┬───────┘  │
│                                                          │          │
│                          ┌───────────────────────────────┤          │
│                          │                               │          │
│                          ▼                               ▼          │
│               ┌──────────────────┐           ┌────────────────────┐ │
│               │  Mock Server     │           │  Code Generation   │ │
│               │  তৈরি করা        │           │  (Server stubs,    │ │
│               │  (Prism, Stoplight│           │   Client SDKs)     │ │
│               └──────────────────┘           └─────────┬──────────┘ │
│                          │                             │            │
│                          ▼                             ▼            │
│               ┌──────────────────┐           ┌────────────────────┐ │
│               │  Frontend Dev    │           │  Backend Dev       │ │
│               │  (Mock দিয়ে কাজ) │           │  (Spec অনুযায়ী)    │ │
│               └────────┬─────────┘           └─────────┬──────────┘ │
│                        │                               │            │
│                        └───────────┬───────────────────┘            │
│                                    ▼                                │
│                         ┌────────────────────┐                      │
│                         │  Integration Test  │                      │
│                         │  + Doc Validation  │                      │
│                         │  (Spec vs Actual)  │                      │
│                         └────────┬───────────┘                      │
│                                  ▼                                  │
│                         ┌────────────────────┐                      │
│                         │  Publish Docs      │                      │
│                         │  (Swagger UI /     │                      │
│                         │   Redoc / Portal)  │                      │
│                         └────────────────────┘                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

```
┌───────────────────────────────────────────────────────────┐
│           DOCUMENTATION TOOLS ECOSYSTEM                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│   Spec Format          UI Rendering       Code Tools      │
│   ──────────          ────────────       ──────────       │
│   OpenAPI 3.x   ───▶  Swagger UI   ◀──  L5-Swagger      │
│   AsyncAPI      ───▶  Redoc        ◀──  Scribe           │
│   GraphQL SDL   ───▶  Stoplight    ◀──  swagger-jsdoc    │
│   Protobuf      ───▶  GraphiQL     ◀──  tsoa             │
│                  ───▶  Apollo Studio◀──  Scramble         │
│                                                           │
│   Testing              Sharing           SDK Gen          │
│   ───────              ───────           ───────          │
│   Dredd          ───▶  Postman     ◀──  openapi-gen      │
│   Schemathesis   ───▶  Insomnia    ◀──  swagger-codegen  │
│   spectral       ───▶  ReadMe.com                        │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## 💻 OpenAPI/Swagger Deep Dive

### ১. OpenAPI 3.0 Specification — মূল কাঠামো

OpenAPI Specification (OAS) হলো REST API-র জন্য **standard description format**। এটি machine-readable (YAML/JSON) এবং এই spec থেকে documentation UI, client SDK, server stub, test — সব generate করা যায়।

**মূল অংশগুলো:**

```yaml
openapi: 3.0.3                    # ← OpenAPI version

info:                              # ← API-র মেটাডেটা
  title: "API শিরোনাম"
  description: "বিস্তারিত বর্ণনা"
  version: "1.0.0"
  contact:
    name: "API Support"
    email: "api@example.com"
  license:
    name: "MIT"

servers:                           # ← কোন কোন server-এ API available
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:                             # ← সব endpoint এখানে define হয়
  /products:
    get: ...
    post: ...
  /products/{id}:
    get: ...
    put: ...
    delete: ...

components:                        # ← reusable schemas, parameters, responses
  schemas: ...
  parameters: ...
  responses: ...
  securitySchemes: ...

security:                          # ← global security requirement
  - bearerAuth: []

tags:                              # ← endpoint grouping
  - name: Products
    description: "পণ্য সম্পর্কিত API"
  - name: Auth
    description: "Authentication সম্পর্কিত API"

externalDocs:                      # ← বাইরের ডকুমেন্টেশনের লিঙ্ক
  description: "বিস্তারিত গাইড"
  url: https://docs.example.com
```

### ২. সম্পূর্ণ OpenAPI YAML উদাহরণ — E-commerce Product API

এটি একটি বাস্তবসম্মত e-commerce API spec যেখানে CRUD, authentication, pagination, error response সব আছে:

```yaml
openapi: 3.0.3
info:
  title: বাংলাশপ E-Commerce API
  description: |
    বাংলাদেশী e-commerce প্ল্যাটফর্মের জন্য সম্পূর্ণ RESTful API।
    এই API দিয়ে পণ্য ম্যানেজমেন্ট, অর্ডার প্রসেসিং এবং পেমেন্ট ইন্টিগ্রেশন করা যায়।

    ## Authentication
    Bearer token ব্যবহার করুন। `/auth/login` endpoint থেকে token নিন।

    ## Rate Limiting
    - প্রতি মিনিটে ৬০টি request (authenticated)
    - প্রতি মিনিটে ২০টি request (unauthenticated)

    ## Pagination
    সব list endpoint `page` ও `per_page` query parameter সাপোর্ট করে।
  version: 2.1.0
  contact:
    name: বাংলাশপ ডেভেলপার সাপোর্ট
    email: devs@banglashop.com.bd
    url: https://developers.banglashop.com.bd
  license:
    name: MIT

servers:
  - url: https://api.banglashop.com.bd/v2
    description: Production Server
  - url: https://sandbox.banglashop.com.bd/v2
    description: Sandbox (টেস্টিং)

tags:
  - name: Authentication
    description: লগইন, রেজিস্ট্রেশন, টোকেন রিফ্রেশ
  - name: Products
    description: পণ্য তৈরি, পড়া, আপডেট, মুছে ফেলা
  - name: Orders
    description: অর্ডার ম্যানেজমেন্ট
  - name: Webhooks
    description: ইভেন্ট নোটিফিকেশন

paths:
  /auth/login:
    post:
      tags: [Authentication]
      summary: ইউজার লগইন
      description: Email ও password দিয়ে লগইন করে JWT token নিন।
      operationId: loginUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email:
                  type: string
                  format: email
                  example: "merchant@banglashop.com.bd"
                password:
                  type: string
                  format: password
                  minLength: 8
                  example: "SecureP@ss123"
            examples:
              merchant_login:
                summary: Merchant লগইন
                value:
                  email: "merchant@banglashop.com.bd"
                  password: "SecureP@ss123"
      responses:
        '200':
          description: সফল লগইন
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '422':
          $ref: '#/components/responses/ValidationError'
        '429':
          $ref: '#/components/responses/TooManyRequests'

  /products:
    get:
      tags: [Products]
      summary: পণ্যের তালিকা (Paginated)
      description: |
        সব পণ্যের paginated তালিকা। ফিল্টার ও সর্ট সাপোর্ট করে।
        Cache-Control header দেখুন — response ৫ মিনিট cache হয়।
      operationId: listProducts
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/PerPageParam'
        - name: category
          in: query
          description: ক্যাটাগরি slug দিয়ে ফিল্টার
          schema:
            type: string
            example: "electronics"
        - name: min_price
          in: query
          schema:
            type: number
            format: float
            minimum: 0
            example: 100.00
        - name: max_price
          in: query
          schema:
            type: number
            format: float
            example: 50000.00
        - name: sort
          in: query
          schema:
            type: string
            enum: [price_asc, price_desc, newest, popular]
            default: newest
        - name: search
          in: query
          description: পণ্যের নাম বা বর্ণনায় সার্চ
          schema:
            type: string
            example: "স্মার্টফোন"
      responses:
        '200':
          description: পণ্যের তালিকা
          headers:
            X-Total-Count:
              schema:
                type: integer
              description: মোট পণ্য সংখ্যা
            X-Rate-Limit-Remaining:
              schema:
                type: integer
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedProducts'
              example:
                data:
                  - id: "prod_abc123"
                    name: "স্যামসাং গ্যালাক্সি A54"
                    slug: "samsung-galaxy-a54"
                    price: 32999.00
                    currency: "BDT"
                    stock: 45
                    category:
                      id: "cat_01"
                      name: "মোবাইল ফোন"
                    images:
                      - url: "https://cdn.banglashop.com.bd/products/a54-1.jpg"
                        alt: "Samsung Galaxy A54 front view"
                    created_at: "2024-01-15T10:30:00Z"
                meta:
                  current_page: 1
                  per_page: 20
                  total: 156
                  last_page: 8
                links:
                  self: "/v2/products?page=1"
                  next: "/v2/products?page=2"
                  prev: null

    post:
      tags: [Products]
      summary: নতুন পণ্য তৈরি
      operationId: createProduct
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProductRequest'
          multipart/form-data:
            schema:
              $ref: '#/components/schemas/CreateProductMultipart'
      responses:
        '201':
          description: পণ্য সফলভাবে তৈরি হয়েছে
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '422':
          $ref: '#/components/responses/ValidationError'

  /products/{productId}:
    parameters:
      - $ref: '#/components/parameters/ProductIdParam'
    get:
      tags: [Products]
      summary: একটি পণ্যের বিস্তারিত
      operationId: getProduct
      responses:
        '200':
          description: পণ্যের বিস্তারিত তথ্য
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '404':
          $ref: '#/components/responses/NotFound'
    put:
      tags: [Products]
      summary: পণ্য আপডেট
      operationId: updateProduct
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateProductRequest'
      responses:
        '200':
          description: আপডেট সফল
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/ValidationError'
    delete:
      tags: [Products]
      summary: পণ্য মুছে ফেলা
      operationId: deleteProduct
      security:
        - bearerAuth: []
      responses:
        '204':
          description: পণ্য সফলভাবে মুছে ফেলা হয়েছে
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    Product:
      type: object
      properties:
        id:
          type: string
          readOnly: true
          example: "prod_abc123"
        name:
          type: string
          maxLength: 255
          example: "স্যামসাং গ্যালাক্সি A54"
        slug:
          type: string
          readOnly: true
        description:
          type: string
          example: "6.4 ইঞ্চি Super AMOLED ডিসপ্লে, 128GB স্টোরেজ"
        price:
          type: number
          format: float
          minimum: 0
          example: 32999.00
        currency:
          type: string
          enum: [BDT, USD]
          default: BDT
        stock:
          type: integer
          minimum: 0
          example: 45
        sku:
          type: string
          example: "SAM-A54-128-BLK"
        category:
          $ref: '#/components/schemas/Category'
        images:
          type: array
          items:
            $ref: '#/components/schemas/ProductImage'
        status:
          type: string
          enum: [active, draft, archived]
          default: active
        created_at:
          type: string
          format: date-time
          readOnly: true
        updated_at:
          type: string
          format: date-time
          readOnly: true

    Category:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        slug:
          type: string

    ProductImage:
      type: object
      properties:
        url:
          type: string
          format: uri
        alt:
          type: string
        is_primary:
          type: boolean
          default: false

    CreateProductRequest:
      type: object
      required: [name, price, category_id]
      properties:
        name:
          type: string
          minLength: 3
          maxLength: 255
        description:
          type: string
        price:
          type: number
          format: float
          minimum: 0.01
        category_id:
          type: string
        stock:
          type: integer
          minimum: 0
          default: 0
        sku:
          type: string
        status:
          type: string
          enum: [active, draft]
          default: draft

    CreateProductMultipart:
      type: object
      required: [name, price, category_id]
      properties:
        name:
          type: string
        price:
          type: number
        category_id:
          type: string
        images:
          type: array
          items:
            type: string
            format: binary

    UpdateProductRequest:
      type: object
      properties:
        name:
          type: string
        description:
          type: string
        price:
          type: number
          format: float
        stock:
          type: integer
        status:
          type: string
          enum: [active, draft, archived]

    ProductResponse:
      type: object
      properties:
        data:
          $ref: '#/components/schemas/Product'

    PaginatedProducts:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Product'
        meta:
          $ref: '#/components/schemas/PaginationMeta'
        links:
          $ref: '#/components/schemas/PaginationLinks'

    PaginationMeta:
      type: object
      properties:
        current_page:
          type: integer
        per_page:
          type: integer
        total:
          type: integer
        last_page:
          type: integer

    PaginationLinks:
      type: object
      properties:
        self:
          type: string
        next:
          type: string
          nullable: true
        prev:
          type: string
          nullable: true

    AuthResponse:
      type: object
      properties:
        access_token:
          type: string
        token_type:
          type: string
          example: "Bearer"
        expires_in:
          type: integer
          example: 3600
        refresh_token:
          type: string

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              example: "VALIDATION_ERROR"
            message:
              type: string
              example: "অনুরোধটি বৈধ নয়"
            details:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                  message:
                    type: string

  parameters:
    ProductIdParam:
      name: productId
      in: path
      required: true
      description: পণ্যের unique ID
      schema:
        type: string
        pattern: '^prod_[a-zA-Z0-9]+$'
        example: "prod_abc123"

    PageParam:
      name: page
      in: query
      description: পৃষ্ঠা নম্বর
      schema:
        type: integer
        minimum: 1
        default: 1

    PerPageParam:
      name: per_page
      in: query
      description: প্রতি পৃষ্ঠায় আইটেম সংখ্যা
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  responses:
    Unauthorized:
      description: Authentication ব্যর্থ বা token অনুপস্থিত
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            error:
              code: "UNAUTHORIZED"
              message: "বৈধ authentication token প্রদান করুন"

    NotFound:
      description: রিসোর্স খুঁজে পাওয়া যায়নি
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            error:
              code: "NOT_FOUND"
              message: "অনুরোধকৃত রিসোর্স খুঁজে পাওয়া যায়নি"

    ValidationError:
      description: ইনপুট validation ব্যর্থ
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            error:
              code: "VALIDATION_ERROR"
              message: "ইনপুট ডেটায় সমস্যা আছে"
              details:
                - field: "price"
                  message: "মূল্য অবশ্যই ধনাত্মক সংখ্যা হতে হবে"
                - field: "name"
                  message: "পণ্যের নাম আবশ্যক"

    TooManyRequests:
      description: Rate limit অতিক্রম করেছে
      headers:
        Retry-After:
          schema:
            type: integer
          description: কত সেকেন্ড পর আবার চেষ্টা করতে হবে
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: "JWT token — `/auth/login` থেকে প্রাপ্ত access_token ব্যবহার করুন"

    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: "API Key — ড্যাশবোর্ড থেকে generate করুন"

    oauth2Auth:
      type: oauth2
      description: OAuth 2.0 Authorization Code Flow
      flows:
        authorizationCode:
          authorizationUrl: https://auth.banglashop.com.bd/authorize
          tokenUrl: https://auth.banglashop.com.bd/token
          refreshUrl: https://auth.banglashop.com.bd/token/refresh
          scopes:
            products:read: "পণ্য পড়তে পারবে"
            products:write: "পণ্য তৈরি/আপডেট/মুছতে পারবে"
            orders:read: "অর্ডার পড়তে পারবে"
            orders:write: "অর্ডার ম্যানেজ করতে পারবে"
```

### ৩. Data Types, Schemas, $ref References

OpenAPI-তে **schema** হলো data structure-এর সংজ্ঞা। `$ref` দিয়ে **reusable schema** reference করা যায় যাতে একই জিনিস বারবার লিখতে না হয়।

```yaml
# $ref কীভাবে কাজ করে:

# একবার define করুন components-এ:
components:
  schemas:
    Money:
      type: object
      properties:
        amount:
          type: number
          format: float
          example: 1500.50
        currency:
          type: string
          enum: [BDT, USD, EUR]
          default: BDT

    Address:
      type: object
      required: [street, city, postal_code]
      properties:
        street:
          type: string
        city:
          type: string
        district:
          type: string
          description: "জেলা (বাংলাদেশ-specific)"
        postal_code:
          type: string
          pattern: '^\d{4}$'

    # Schema Composition — allOf, oneOf, anyOf
    OrderItem:
      allOf:                          # ← inheritance/merge
        - $ref: '#/components/schemas/Product'
        - type: object
          properties:
            quantity:
              type: integer
              minimum: 1
            subtotal:
              $ref: '#/components/schemas/Money'

    PaymentMethod:
      oneOf:                          # ← exactly one match
        - $ref: '#/components/schemas/BkashPayment'
        - $ref: '#/components/schemas/CardPayment'
        - $ref: '#/components/schemas/CodPayment'
      discriminator:
        propertyName: method
        mapping:
          bkash: '#/components/schemas/BkashPayment'
          card: '#/components/schemas/CardPayment'
          cod: '#/components/schemas/CodPayment'

    BkashPayment:
      type: object
      required: [method, bkash_number]
      properties:
        method:
          type: string
          enum: [bkash]
        bkash_number:
          type: string
          pattern: '^01[3-9]\d{8}$'
          description: "বৈধ বাংলাদেশী মোবাইল নম্বর"

    CardPayment:
      type: object
      required: [method, card_token]
      properties:
        method:
          type: string
          enum: [card]
        card_token:
          type: string
          description: "Tokenized card (PCI compliance)"

    CodPayment:
      type: object
      required: [method]
      properties:
        method:
          type: string
          enum: [cod]
        note:
          type: string

# যেকোনো জায়গায় reference করুন:
paths:
  /orders:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                shipping_address:
                  $ref: '#/components/schemas/Address'     # ← reuse!
                payment:
                  $ref: '#/components/schemas/PaymentMethod'
                total:
                  $ref: '#/components/schemas/Money'       # ← reuse!
```

### ৪. Request Body, Parameters (path, query, header)

```yaml
# Parameter-এর ৪ ধরনের location:

parameters:
  # ১. Path Parameter — URL-এর মধ্যে (required)
  - name: orderId
    in: path
    required: true
    schema:
      type: string
      pattern: '^ord_[a-zA-Z0-9]{10}$'

  # ২. Query Parameter — URL-এর পরে ?key=value
  - name: status
    in: query
    required: false
    schema:
      type: string
      enum: [pending, processing, shipped, delivered, cancelled]
    description: "অর্ডার স্ট্যাটাস দিয়ে ফিল্টার"

  # ৩. Header Parameter — HTTP header-এ
  - name: X-Request-ID
    in: header
    required: false
    schema:
      type: string
      format: uuid
    description: "Idempotency key — duplicate request আটকায়"

  # ৪. Cookie Parameter
  - name: session_id
    in: cookie
    schema:
      type: string

# Request Body — POST/PUT/PATCH-এ data পাঠানো
requestBody:
  required: true
  description: "পণ্য তৈরির জন্য প্রয়োজনীয় তথ্য"
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateProductRequest'
      examples:
        smartphone:
          summary: "স্মার্টফোন তৈরি"
          value:
            name: "Xiaomi Redmi Note 13"
            price: 22999.00
            category_id: "cat_phones"
            stock: 100
            sku: "XIA-RN13-256"
        clothing:
          summary: "পোশাক তৈরি"
          value:
            name: "পাঞ্জাবি — নেভি ব্লু"
            price: 1500.00
            category_id: "cat_clothing"
            stock: 50

    # Multiple content types সাপোর্ট:
    multipart/form-data:
      schema:
        type: object
        properties:
          name:
            type: string
          price:
            type: number
          images:
            type: array
            items:
              type: string
              format: binary
            maxItems: 5
        encoding:
          images:
            contentType: image/jpeg, image/png, image/webp
```

### ৫. Response Definitions with Examples

```yaml
responses:
  '200':
    description: সফল response
    headers:
      X-RateLimit-Limit:
        schema:
          type: integer
          example: 60
      X-RateLimit-Remaining:
        schema:
          type: integer
          example: 45
      Cache-Control:
        schema:
          type: string
          example: "public, max-age=300"
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/PaginatedProducts'
        examples:
          with_results:
            summary: "পণ্য পাওয়া গেছে"
            value:
              data:
                - id: "prod_001"
                  name: "ওয়ালটন প্রিমো H10"
                  price: 8999.00
                  currency: "BDT"
              meta:
                current_page: 1
                total: 42
          empty:
            summary: "কোনো পণ্য পাওয়া যায়নি"
            value:
              data: []
              meta:
                current_page: 1
                total: 0

  # Error response pattern:
  '400':
    description: Bad Request
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
        examples:
          invalid_json:
            value:
              error:
                code: "BAD_REQUEST"
                message: "JSON ফরম্যাট সঠিক নয়"
          missing_field:
            value:
              error:
                code: "VALIDATION_ERROR"
                message: "আবশ্যক ফিল্ড অনুপস্থিত"
                details:
                  - field: "name"
                    message: "পণ্যের নাম দিতে হবে"
```

### ৬. Security Definitions

ওপরের spec-এ ৩ ধরনের security scheme দেখানো হয়েছে — **Bearer JWT**, **API Key**, এবং **OAuth2**। বাস্তবে বাংলাদেশী API-গুলোতে (bKash, Nagad, SSLCommerz) সাধারণত API Key + Secret-based authentication বেশি দেখা যায়। OAuth2 ব্যবহার হয় third-party integration-এ।

---

## 🔧 Tools & Implementation

### Swagger UI — Setup ও Customization

Swagger UI হলো OpenAPI spec থেকে interactive documentation generate করার সবচেয়ে জনপ্রিয় tool। "Try it out" বোতাম দিয়ে সরাসরি API call করা যায়।

### Laravel: L5-Swagger (Annotations)

L5-Swagger PHP annotation ব্যবহার করে controller/model থেকে OpenAPI spec generate করে:

```php
<?php
// app/Http/Controllers/Api/V2/ProductController.php

namespace App\Http\Controllers\Api\V2;

use App\Http\Controllers\Controller;
use App\Http\Requests\StoreProductRequest;
use App\Http\Resources\ProductResource;
use App\Http\Resources\ProductCollection;
use App\Models\Product;
use Illuminate\Http\Request;
use OpenApi\Attributes as OA;

#[OA\Info(
    version: "2.1.0",
    title: "বাংলাশপ API",
    description: "E-commerce API ডকুমেন্টেশন",
    contact: new OA\Contact(name: "Dev Team", email: "dev@banglashop.com.bd")
)]
#[OA\Server(url: "https://api.banglashop.com.bd/v2", description: "Production")]
#[OA\Server(url: "https://sandbox.banglashop.com.bd/v2", description: "Sandbox")]
#[OA\SecurityScheme(
    securityScheme: "bearerAuth",
    type: "http",
    scheme: "bearer",
    bearerFormat: "JWT"
)]
class ProductController extends Controller
{
    #[OA\Get(
        path: "/products",
        summary: "পণ্যের তালিকা",
        tags: ["Products"],
        parameters: [
            new OA\Parameter(
                name: "page",
                in: "query",
                required: false,
                schema: new OA\Schema(type: "integer", default: 1)
            ),
            new OA\Parameter(
                name: "per_page",
                in: "query",
                required: false,
                schema: new OA\Schema(type: "integer", default: 20, maximum: 100)
            ),
            new OA\Parameter(
                name: "category",
                in: "query",
                required: false,
                schema: new OA\Schema(type: "string"),
                description: "ক্যাটাগরি slug দিয়ে ফিল্টার"
            ),
            new OA\Parameter(
                name: "search",
                in: "query",
                required: false,
                schema: new OA\Schema(type: "string"),
                description: "পণ্যের নামে সার্চ"
            ),
            new OA\Parameter(
                name: "sort",
                in: "query",
                schema: new OA\Schema(
                    type: "string",
                    enum: ["price_asc", "price_desc", "newest", "popular"]
                )
            ),
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: "পণ্যের তালিকা",
                content: new OA\JsonContent(
                    properties: [
                        new OA\Property(property: "data", type: "array",
                            items: new OA\Items(ref: "#/components/schemas/Product")
                        ),
                        new OA\Property(property: "meta", ref: "#/components/schemas/PaginationMeta"),
                    ]
                )
            ),
        ]
    )]
    public function index(Request $request): ProductCollection
    {
        $products = Product::query()
            ->when($request->category, fn($q, $cat) => $q->whereCategory($cat))
            ->when($request->search, fn($q, $s) => $q->search($s))
            ->when($request->sort, fn($q, $sort) => match($sort) {
                'price_asc'  => $q->orderBy('price'),
                'price_desc' => $q->orderByDesc('price'),
                'popular'    => $q->orderByDesc('views'),
                default      => $q->latest(),
            })
            ->paginate($request->per_page ?? 20);

        return new ProductCollection($products);
    }

    #[OA\Post(
        path: "/products",
        summary: "নতুন পণ্য তৈরি",
        security: [["bearerAuth" => []]],
        tags: ["Products"],
        requestBody: new OA\RequestBody(
            required: true,
            content: new OA\JsonContent(ref: "#/components/schemas/CreateProductRequest")
        ),
        responses: [
            new OA\Response(
                response: 201,
                description: "পণ্য তৈরি সফল",
                content: new OA\JsonContent(
                    properties: [
                        new OA\Property(property: "data", ref: "#/components/schemas/Product"),
                    ]
                )
            ),
            new OA\Response(response: 401, ref: "#/components/responses/Unauthorized"),
            new OA\Response(response: 422, ref: "#/components/responses/ValidationError"),
        ]
    )]
    public function store(StoreProductRequest $request): ProductResource
    {
        $product = Product::create($request->validated());
        return new ProductResource($product);
    }

    #[OA\Get(
        path: "/products/{productId}",
        summary: "পণ্যের বিস্তারিত",
        tags: ["Products"],
        parameters: [
            new OA\Parameter(
                name: "productId",
                in: "path",
                required: true,
                schema: new OA\Schema(type: "string")
            ),
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: "পণ্যের তথ্য",
                content: new OA\JsonContent(
                    properties: [
                        new OA\Property(property: "data", ref: "#/components/schemas/Product"),
                    ]
                )
            ),
            new OA\Response(response: 404, ref: "#/components/responses/NotFound"),
        ]
    )]
    public function show(Product $product): ProductResource
    {
        return new ProductResource($product->load(['category', 'images']));
    }
}
```

### Laravel: Scribe — Code থেকে Auto-generation

Scribe route definitions, FormRequest, এবং API Resource থেকে **স্বয়ংক্রিয়ভাবে** ডকুমেন্টেশন তৈরি করে:

```php
// config/scribe.php — মূল কনফিগারেশন
return [
    'title' => 'বাংলাশপ API ডকুমেন্টেশন',
    'description' => 'E-commerce platform API',
    'base_url' => env('APP_URL') . '/api/v2',
    'type' => 'laravel',

    'routes' => [
        [
            'match' => [
                'prefixes' => ['api/v2/*'],
                'domains' => ['*'],
            ],
            'include' => [],
            'exclude' => [],
            'apply' => [
                'headers' => [
                    'Content-Type' => 'application/json',
                    'Accept' => 'application/json',
                ],
                'response_calls' => [
                    'methods' => ['GET'],  // শুধু GET endpoint-এ actual call করবে
                    'config' => [
                        'app.env' => 'documentation',
                    ],
                ],
            ],
        ],
    ],

    'auth' => [
        'enabled' => true,
        'default' => false,
        'in' => 'bearer',
        'name' => 'Authorization',
        'use_value' => 'Bearer {YOUR_AUTH_TOKEN}',
        'placeholder' => '{YOUR_AUTH_TOKEN}',
    ],

    'try_it_out' => [
        'enabled' => true,
        'base_url' => env('SCRIBE_TRY_IT_URL', 'https://sandbox.banglashop.com.bd/api/v2'),
    ],
];

// Controller-এ Scribe-specific annotation:
/**
 * @group Products
 *
 * পণ্য সম্পর্কিত API endpoints
 */
class ProductController extends Controller
{
    /**
     * পণ্যের তালিকা
     *
     * Paginated product listing with filters.
     *
     * @queryParam page integer পৃষ্ঠা নম্বর। Example: 1
     * @queryParam per_page integer প্রতি পৃষ্ঠায় আইটেম (max 100)। Example: 20
     * @queryParam category string ক্যাটাগরি slug। Example: electronics
     * @queryParam search string সার্চ টার্ম। Example: samsung
     *
     * @responseField data object[] পণ্যের তালিকা
     * @responseField data[].id string পণ্যের ID
     * @responseField data[].name string পণ্যের নাম
     * @responseField data[].price number মূল্য (BDT)
     * @responseField meta object Pagination তথ্য
     */
    public function index(Request $request): ProductCollection { /* ... */ }
}
```

```bash
# Scribe দিয়ে ডকুমেন্টেশন generate:
php artisan scribe:generate

# Output: public/docs/index.html
# Postman collection-ও generate হয়: public/docs/collection.json
```

### Laravel: Scramble — Zero-config API Docs

```php
// Scramble কোনো annotation ছাড়াই Laravel route, FormRequest, Resource থেকে
// স্বয়ংক্রিয়ভাবে OpenAPI spec generate করে।

// composer require dedoc/scramble
// config/scramble.php
return [
    'api_path' => 'api',
    'api_domain' => null,
    'info' => [
        'version' => '2.1.0',
        'description' => 'বাংলাশপ API',
    ],
    // Scramble Laravel-এর type system (return types, FormRequest rules) থেকে
    // schema infer করে — কোনো extra annotation লাগে না!
];

// /docs/api এ গেলেই Swagger UI দেখাবে
```

### Express: swagger-jsdoc + swagger-ui-express

```javascript
// src/config/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const options = {
  definition: {
    openapi: '3.0.3',
    info: {
      title: 'বাংলাশপ API',
      version: '2.1.0',
      description: 'E-commerce REST API — Node.js Express',
      contact: {
        name: 'Developer Support',
        email: 'devs@banglashop.com.bd',
      },
    },
    servers: [
      { url: 'https://api.banglashop.com.bd/v2', description: 'Production' },
      { url: 'http://localhost:3000/v2', description: 'Local Development' },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
    },
  },
  apis: ['./src/routes/*.js', './src/models/*.js'],
};

const swaggerSpec = swaggerJsdoc(options);

function setupSwagger(app) {
  // Swagger UI serve
  app.use(
    '/api-docs',
    swaggerUi.serve,
    swaggerUi.setup(swaggerSpec, {
      customCss: '.swagger-ui .topbar { display: none }',
      customSiteTitle: 'বাংলাশপ API Docs',
      swaggerOptions: {
        persistAuthorization: true,
        displayRequestDuration: true,
        filter: true,
        tryItOutEnabled: true,
      },
    })
  );

  // Raw JSON spec endpoint (SDK generation, CI validation ইত্যাদির জন্য)
  app.get('/api-docs.json', (req, res) => {
    res.setHeader('Content-Type', 'application/json');
    res.send(swaggerSpec);
  });
}

module.exports = { setupSwagger, swaggerSpec };

// src/routes/products.js
const express = require('express');
const router = express.Router();
const { authenticate, authorize } = require('../middleware/auth');
const ProductController = require('../controllers/ProductController');
const { validate } = require('../middleware/validate');
const { createProductSchema, updateProductSchema } = require('../validators/product');

/**
 * @openapi
 * components:
 *   schemas:
 *     Product:
 *       type: object
 *       properties:
 *         id:
 *           type: string
 *           example: "prod_abc123"
 *         name:
 *           type: string
 *           example: "স্যামসাং গ্যালাক্সি A54"
 *         price:
 *           type: number
 *           format: float
 *           example: 32999.00
 *         currency:
 *           type: string
 *           enum: [BDT, USD]
 *         stock:
 *           type: integer
 *           example: 45
 *         category:
 *           $ref: '#/components/schemas/Category'
 *         status:
 *           type: string
 *           enum: [active, draft, archived]
 *         created_at:
 *           type: string
 *           format: date-time
 *
 *     CreateProductRequest:
 *       type: object
 *       required:
 *         - name
 *         - price
 *         - category_id
 *       properties:
 *         name:
 *           type: string
 *           minLength: 3
 *           maxLength: 255
 *         price:
 *           type: number
 *           minimum: 0.01
 *         category_id:
 *           type: string
 *         stock:
 *           type: integer
 *           default: 0
 *         sku:
 *           type: string
 *
 *     PaginatedProducts:
 *       type: object
 *       properties:
 *         data:
 *           type: array
 *           items:
 *             $ref: '#/components/schemas/Product'
 *         meta:
 *           type: object
 *           properties:
 *             current_page:
 *               type: integer
 *             total:
 *               type: integer
 *             per_page:
 *               type: integer
 */

/**
 * @openapi
 * /products:
 *   get:
 *     tags: [Products]
 *     summary: পণ্যের তালিকা (Paginated)
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: per_page
 *         schema:
 *           type: integer
 *           default: 20
 *       - in: query
 *         name: category
 *         schema:
 *           type: string
 *         description: ক্যাটাগরি slug
 *       - in: query
 *         name: search
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: সফল
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/PaginatedProducts'
 */
router.get('/', ProductController.index);

/**
 * @openapi
 * /products:
 *   post:
 *     tags: [Products]
 *     summary: নতুন পণ্য তৈরি
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreateProductRequest'
 *     responses:
 *       201:
 *         description: পণ্য তৈরি সফল
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   $ref: '#/components/schemas/Product'
 *       401:
 *         description: Unauthorized
 *       422:
 *         description: Validation Error
 */
router.post('/', authenticate, authorize('admin', 'merchant'), validate(createProductSchema), ProductController.store);

/**
 * @openapi
 * /products/{productId}:
 *   get:
 *     tags: [Products]
 *     summary: পণ্যের বিস্তারিত
 *     parameters:
 *       - in: path
 *         name: productId
 *         required: true
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: পণ্যের তথ্য
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   $ref: '#/components/schemas/Product'
 *       404:
 *         description: পণ্য পাওয়া যায়নি
 */
router.get('/:productId', ProductController.show);

module.exports = router;

// src/app.js — Swagger setup
const express = require('express');
const { setupSwagger } = require('./config/swagger');

const app = express();
app.use(express.json());

setupSwagger(app);  // /api-docs এ Swagger UI পাবেন

app.use('/v2/products', require('./routes/products'));
app.use('/v2/auth', require('./routes/auth'));
```

### Express: tsoa — TypeScript-based Auto-generation

```typescript
// tsoa TypeScript decorator থেকে OpenAPI spec ও route generate করে

// src/controllers/ProductController.ts
import {
  Controller, Get, Post, Put, Delete,
  Route, Tags, Security, Body, Path, Query,
  SuccessResponse, Response, Example
} from 'tsoa';

interface Product {
  id: string;
  name: string;
  price: number;
  currency: 'BDT' | 'USD';
  stock: number;
  status: 'active' | 'draft' | 'archived';
}

interface CreateProductInput {
  /** @minLength 3 @maxLength 255 */
  name: string;
  /** @minimum 0.01 */
  price: number;
  category_id: string;
  stock?: number;
}

interface PaginatedResponse<T> {
  data: T[];
  meta: { current_page: number; total: number; per_page: number; };
}

@Route('v2/products')
@Tags('Products')
export class ProductController extends Controller {

  /** পণ্যের তালিকা (Paginated) */
  @Get('/')
  public async listProducts(
    @Query() page: number = 1,
    @Query() per_page: number = 20,
    @Query() category?: string,
    @Query() search?: string,
  ): Promise<PaginatedResponse<Product>> {
    // implementation
  }

  /** নতুন পণ্য তৈরি */
  @Post('/')
  @Security('bearerAuth')
  @SuccessResponse(201, 'Created')
  @Response(422, 'Validation Error')
  public async createProduct(
    @Body() body: CreateProductInput
  ): Promise<{ data: Product }> {
    this.setStatus(201);
    // implementation
  }

  @Get('{productId}')
  @Response(404, 'Not Found')
  public async getProduct(
    @Path() productId: string
  ): Promise<{ data: Product }> {
    // implementation
  }
}
```

### Postman Collection — তৈরি, শেয়ার, টেস্ট ও Mock Server

```javascript
// Postman Collection v2.1 format — export/import করা যায়
// OpenAPI spec থেকে auto-import করা সবচেয়ে efficient

// Postman Environment Variables:
// {{base_url}} = https://sandbox.banglashop.com.bd/v2
// {{auth_token}} = (login করলে auto-set হবে)

// Pre-request Script (Collection level — auto-login):
// pm.sendRequest({
//     url: pm.environment.get('base_url') + '/auth/login',
//     method: 'POST',
//     header: { 'Content-Type': 'application/json' },
//     body: { mode: 'raw', raw: JSON.stringify({
//         email: pm.environment.get('test_email'),
//         password: pm.environment.get('test_password')
//     })}
// }, function(err, res) {
//     pm.environment.set('auth_token', res.json().access_token);
// });

// Test Script (Product creation test):
const testProductCreation = `
pm.test("Status code 201", function () {
    pm.response.to.have.status(201);
});

pm.test("Response has product data", function () {
    const json = pm.response.json();
    pm.expect(json.data).to.have.property('id');
    pm.expect(json.data).to.have.property('name');
    pm.expect(json.data.price).to.be.a('number');
    pm.expect(json.data.currency).to.equal('BDT');
});

pm.test("Response time < 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

// পরবর্তী test-এ ব্যবহারের জন্য product ID সংরক্ষণ:
if (pm.response.code === 201) {
    pm.environment.set('created_product_id', pm.response.json().data.id);
}
`;

// Mock Server:
// Postman-এ Collection থেকে Mock Server তৈরি করলে
// একটি URL পাবেন (যেমন: https://abc123.mock.pstmn.io)
// যেটায় request পাঠালে Collection-এর example response ফেরত আসবে।
// Frontend team এটা দিয়ে parallel development করতে পারে।
```

### Redoc — বিকল্প UI

```javascript
// Express-এ Redoc setup:
const express = require('express');
const { swaggerSpec } = require('./config/swagger');

const app = express();

// Redoc — সুন্দর, clean, তিন-কলাম layout
app.get('/docs', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <title>বাংলাশপ API Docs</title>
        <meta charset="utf-8"/>
        <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700" rel="stylesheet">
      </head>
      <body>
        <redoc spec-url='/api-docs.json'
          hide-download-button="false"
          theme='{
            "colors": { "primary": { "main": "#1a73e8" } },
            "typography": { "fontSize": "15px" },
            "sidebar": { "width": "280px" }
          }'
        ></redoc>
        <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
      </body>
    </html>
  `);
});

app.get('/api-docs.json', (req, res) => res.json(swaggerSpec));
```

```php
// Laravel-এ Redoc:
// routes/web.php
Route::get('/docs', function () {
    return view('redoc');
});

// resources/views/redoc.blade.php
// <redoc spec-url="{{ url('/api-docs.json') }}"></redoc>
// <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
```

---

## 🔥 Advanced Topics

### ১. Code-First vs Design-First — বিস্তারিত তুলনা

```
┌──────────────────┬────────────────────────┬────────────────────────┐
│     বৈশিষ্ট্য     │    Design-First        │    Code-First          │
├──────────────────┼────────────────────────┼────────────────────────┤
│ প্রক্রিয়া          │ Spec → Code            │ Code → Spec            │
│ প্রধান টুল        │ Stoplight, SwaggerHub  │ Annotations, Scribe    │
│ দলীয় সহযোগিতা    │ ✅ সহজ (non-dev ও      │ ❌ কঠিন (কোড পড়তে     │
│                  │    review করতে পারে)    │    হয়)                 │
│ Parallel Dev     │ ✅ Mock server দিয়ে     │ ❌ Backend শেষ হওয়া    │
│                  │    Frontend শুরু করা যায়│    পর্যন্ত অপেক্ষা     │
│ Single Source    │ Spec file              │ Code                   │
│ of Truth         │                        │                        │
│ Drift ঝুঁকি       │ ⚠️ Spec আর code আলাদা  │ ✅ কম (code-ই source)  │
│                  │    হয়ে যেতে পারে        │                        │
│ বড় দলে           │ ✅ সেরা                 │ ⚠️ মাঝামাঝি            │
│ ছোট দলে/MVP     │ ⚠️ Overhead বেশি       │ ✅ দ্রুত               │
│ বাংলাদেশ context │ SSLCommerz, bKash      │ অধিকাংশ startup       │
│                  │ (ভালো ডক দরকার)        │ (দ্রুত ship করতে হয়)    │
└──────────────────┴────────────────────────┴────────────────────────┘
```

**সুপারিশ:** বড় public API → Design-First। Internal microservice → Code-First।

### ২. Auto-generating Docs from Code

```php
// PHP: OpenAPI Attributes (PHP 8.1+) দিয়ে auto-generate

// Model schema annotation:
#[OA\Schema(
    schema: "Product",
    title: "Product",
    description: "পণ্য মডেল",
    required: ["name", "price"]
)]
class Product extends Model
{
    #[OA\Property(type: "string", example: "prod_abc123", readOnly: true)]
    public string $id;

    #[OA\Property(type: "string", minLength: 3, maxLength: 255)]
    public string $name;

    #[OA\Property(type: "number", format: "float", minimum: 0)]
    public float $price;
}

// CLI command দিয়ে spec generate:
// php artisan l5-swagger:generate
// Output: storage/api-docs/api-docs.json
```

```javascript
// JSDoc comment থেকে swagger-jsdoc auto-generate করে
// উপরের Express section-এ দেখানো @openapi comment গুলোই এই কাজ করে।

// models/Product.js — Model-level schema docs:
/**
 * @openapi
 * components:
 *   schemas:
 *     Product:
 *       type: object
 *       description: পণ্য মডেল
 *       required:
 *         - name
 *         - price
 *       properties:
 *         id:
 *           type: string
 *           readOnly: true
 *         name:
 *           type: string
 *           minLength: 3
 *         price:
 *           type: number
 *           minimum: 0.01
 */
```

### ৩. API Playground / Try-it-out

Swagger UI-তে built-in "Try it out" আছে। Custom playground বানাতে চাইলে:

```javascript
// Express — custom API playground endpoint
const express = require('express');
const router = express.Router();

router.post('/playground/execute', async (req, res) => {
  const { method, path, headers, body, params } = req.body;

  // Sandbox environment-এ forward করা
  // Production-এ এটা enable করবেন না!
  const sandboxUrl = `${process.env.SANDBOX_API_URL}${path}`;

  try {
    const response = await fetch(sandboxUrl, {
      method,
      headers: {
        'Content-Type': 'application/json',
        ...headers,
      },
      body: ['POST', 'PUT', 'PATCH'].includes(method) ? JSON.stringify(body) : undefined,
    });

    const data = await response.json();
    res.json({
      status: response.status,
      headers: Object.fromEntries(response.headers),
      body: data,
      timing: response.headers.get('X-Response-Time'),
    });
  } catch (error) {
    res.status(502).json({ error: 'Sandbox API unavailable' });
  }
});
```

### ৪. Versioned Documentation

```php
// Laravel — version-wise আলাদা documentation

// config/l5-swagger.php
return [
    'documentations' => [
        'v1' => [
            'api' => [
                'title' => 'বাংলাশপ API v1 (Legacy)',
            ],
            'routes' => [
                'api' => 'api/v1/documentation',
                'docs' => 'docs/v1',
            ],
            'paths' => [
                'docs_json' => 'api-docs-v1.json',
                'annotations' => [base_path('app/Http/Controllers/Api/V1')],
            ],
        ],
        'v2' => [
            'api' => [
                'title' => 'বাংলাশপ API v2 (Current)',
            ],
            'routes' => [
                'api' => 'api/v2/documentation',
                'docs' => 'docs/v2',
            ],
            'paths' => [
                'docs_json' => 'api-docs-v2.json',
                'annotations' => [base_path('app/Http/Controllers/Api/V2')],
            ],
        ],
    ],
];
```

```javascript
// Express — version-wise Swagger
const swaggerV1 = swaggerJsdoc({
  definition: { openapi: '3.0.3', info: { title: 'API v1', version: '1.0.0' } },
  apis: ['./src/routes/v1/*.js'],
});
const swaggerV2 = swaggerJsdoc({
  definition: { openapi: '3.0.3', info: { title: 'API v2', version: '2.0.0' } },
  apis: ['./src/routes/v2/*.js'],
});

app.use('/docs/v1', swaggerUi.serveFiles(swaggerV1), swaggerUi.setup(swaggerV1));
app.use('/docs/v2', swaggerUi.serveFiles(swaggerV2), swaggerUi.setup(swaggerV2));
```

### ৫. Webhook Documentation

```yaml
# OpenAPI 3.1.0 — webhooks support
webhooks:
  orderStatusChanged:
    post:
      summary: অর্ডার স্ট্যাটাস পরিবর্তন
      description: |
        অর্ডারের স্ট্যাটাস পরিবর্তন হলে আপনার registered URL-এ POST request পাঠানো হবে।
        আপনাকে ২০০ status code ফেরত দিতে হবে ৫ সেকেন্ডের মধ্যে।
        ব্যর্থ হলে ৩ বার retry হবে (exponential backoff)।
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                event:
                  type: string
                  enum: [order.created, order.paid, order.shipped, order.delivered, order.cancelled]
                timestamp:
                  type: string
                  format: date-time
                data:
                  type: object
                  properties:
                    order_id:
                      type: string
                    status:
                      type: string
                    total:
                      type: number
                signature:
                  type: string
                  description: "HMAC-SHA256 signature — webhook secret দিয়ে verify করুন"
      responses:
        '200':
          description: Webhook সফলভাবে গ্রহণ করা হয়েছে
```

```php
// Laravel — Webhook verification middleware
class VerifyWebhookSignature
{
    public function handle($request, Closure $next)
    {
        $signature = $request->header('X-Webhook-Signature');
        $payload = $request->getContent();
        $secret = config('services.banglashop.webhook_secret');

        $expected = hash_hmac('sha256', $payload, $secret);

        if (!hash_equals($expected, $signature)) {
            return response()->json(['error' => 'Invalid signature'], 403);
        }

        return $next($request);
    }
}
```

### ৬. SDK Generation from OpenAPI

```bash
# openapi-generator দিয়ে client SDK generate করা:

# JavaScript/TypeScript SDK:
npx openapi-generator-cli generate \
  -i https://api.banglashop.com.bd/v2/openapi.json \
  -g typescript-axios \
  -o ./sdk/typescript \
  --additional-properties=npmName=banglashop-sdk,npmVersion=2.1.0

# PHP SDK:
npx openapi-generator-cli generate \
  -i https://api.banglashop.com.bd/v2/openapi.json \
  -g php \
  -o ./sdk/php \
  --additional-properties=packageName=BanglaShop

# Python SDK:
npx openapi-generator-cli generate \
  -i https://api.banglashop.com.bd/v2/openapi.json \
  -g python \
  -o ./sdk/python
```

```javascript
// Generated SDK ব্যবহারের উদাহরণ:
const { ProductsApi, Configuration } = require('banglashop-sdk');

const config = new Configuration({
  basePath: 'https://api.banglashop.com.bd/v2',
  accessToken: 'your-jwt-token',
});

const productsApi = new ProductsApi(config);

// Type-safe API call:
const products = await productsApi.listProducts({
  page: 1,
  perPage: 20,
  category: 'electronics',
});

console.log(products.data); // Product[] — auto-typed!
```

### ৭. API Changelog Documentation

```markdown
# API Changelog কীভাবে maintain করবেন:

## Format:
## [version] - YYYY-MM-DD

### Added (নতুন যোগ হয়েছে)
### Changed (পরিবর্তন হয়েছে)
### Deprecated (ভবিষ্যতে সরানো হবে)
### Removed (সরিয়ে দেওয়া হয়েছে)
### Fixed (ত্রুটি সংশোধন)
### Security (নিরাপত্তা সংক্রান্ত)
```

```php
// Laravel — API changelog endpoint
Route::get('/api/changelog', function () {
    return response()->json([
        'versions' => [
            [
                'version' => '2.1.0',
                'date' => '2024-03-15',
                'changes' => [
                    ['type' => 'added', 'description' => 'Product search endpoint যোগ হয়েছে'],
                    ['type' => 'changed', 'description' => 'Pagination response format আপডেট'],
                    ['type' => 'deprecated', 'description' => '/products/list endpoint — /products ব্যবহার করুন'],
                ],
            ],
            [
                'version' => '2.0.0',
                'date' => '2024-01-01',
                'changes' => [
                    ['type' => 'removed', 'description' => 'v1 API endpoints সরিয়ে দেওয়া হয়েছে'],
                    ['type' => 'added', 'description' => 'OAuth 2.0 সাপোর্ট'],
                    ['type' => 'security', 'description' => 'Rate limiting যোগ হয়েছে'],
                ],
            ],
        ],
    ]);
});
```

### ৮. GraphQL Documentation — GraphiQL ও Apollo Studio

GraphQL API-র জন্য OpenAPI প্রযোজ্য নয়। GraphQL **introspection** দিয়ে নিজেই self-documenting:

```javascript
// Express + Apollo Server — GraphQL docs built-in
const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');

const typeDefs = `#graphql
  """পণ্য মডেল — e-commerce product"""
  type Product {
    "Unique product identifier"
    id: ID!
    "পণ্যের নাম (বাংলা বা ইংরেজি)"
    name: String!
    "মূল্য (BDT)"
    price: Float!
    "মজুদ পরিমাণ"
    stock: Int!
    "ক্যাটাগরি"
    category: Category!
    "পণ্যের ছবি"
    images: [ProductImage!]!
    status: ProductStatus!
    createdAt: DateTime!
  }

  enum ProductStatus {
    ACTIVE
    DRAFT
    ARCHIVED
  }

  """Pagination-সহ পণ্যের তালিকা"""
  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  input ProductFilter {
    category: String
    minPrice: Float
    maxPrice: Float
    status: ProductStatus
    search: String
  }

  type Query {
    """একটি পণ্যের বিস্তারিত"""
    product(id: ID!): Product

    """পণ্যের তালিকা (cursor-based pagination)"""
    products(
      first: Int = 20
      after: String
      filter: ProductFilter
    ): ProductConnection!
  }

  type Mutation {
    """নতুন পণ্য তৈরি"""
    createProduct(input: CreateProductInput!): Product!
  }
`;

const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: true,  // Production-এ disable করার কথা ভাবুন
  plugins: [
    // Apollo Studio-তে auto-upload schema
    // ApolloServerPluginSchemaReporting(),
  ],
});

// /graphql endpoint-এ Apollo Sandbox / GraphiQL পাবেন
// এটি automatically schema থেকে docs তৈরি করে
```

### ৯. gRPC Documentation — protoc-gen-doc

```protobuf
// proto/product.proto
syntax = "proto3";
package banglashop.v2;

// পণ্য সার্ভিস — CRUD operations
service ProductService {
  // পণ্যের তালিকা
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
  // পণ্যের বিস্তারিত
  rpc GetProduct(GetProductRequest) returns (Product);
  // নতুন পণ্য তৈরি
  rpc CreateProduct(CreateProductRequest) returns (Product);
  // পণ্য আপডেট
  rpc UpdateProduct(UpdateProductRequest) returns (Product);
  // পণ্য মুছে ফেলা
  rpc DeleteProduct(DeleteProductRequest) returns (google.protobuf.Empty);
}

// পণ্যের মূল মডেল
message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  double price = 4;          // BDT-তে মূল্য
  int32 stock = 5;
  string category_id = 6;
  ProductStatus status = 7;
  google.protobuf.Timestamp created_at = 8;
}

enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;
  PRODUCT_STATUS_ACTIVE = 1;
  PRODUCT_STATUS_DRAFT = 2;
  PRODUCT_STATUS_ARCHIVED = 3;
}

message ListProductsRequest {
  int32 page_size = 1;       // প্রতি পৃষ্ঠায় আইটেম (max 100)
  string page_token = 2;     // পরবর্তী পৃষ্ঠার token
  string filter = 3;         // ফিল্টার expression
}
```

```bash
# protoc-gen-doc দিয়ে HTML/Markdown documentation generate:
protoc --doc_out=./docs --doc_opt=html,index.html proto/*.proto
protoc --doc_out=./docs --doc_opt=markdown,api.md proto/*.proto
```

### ১০. Documentation Testing — Docs আর Actual API মিলছে কিনা যাচাই

```javascript
// Dredd — API description আর actual API implementation-এর মধ্যে test:
// dredd openapi.yaml http://localhost:3000/v2

// dredd.yml configuration:
// language: nodejs
// server: node src/app.js
// blueprint: openapi.yaml
// endpoint: http://localhost:3000/v2

// Schemathesis — property-based API testing from OpenAPI spec:
// schemathesis run https://localhost:3000/api-docs.json --checks all

// Jest দিয়ে custom doc validation:
const SwaggerParser = require('@apidevtools/swagger-parser');
const axios = require('axios');

describe('API Documentation Validation', () => {
  let spec;

  beforeAll(async () => {
    spec = await SwaggerParser.validate('./openapi.yaml');
  });

  test('OpenAPI spec বৈধ কিনা', () => {
    expect(spec.openapi).toMatch(/^3\./);
    expect(spec.info.title).toBeDefined();
  });

  test('সব documented endpoint accessible', async () => {
    const paths = Object.keys(spec.paths);

    for (const path of paths) {
      const methods = Object.keys(spec.paths[path]).filter(m =>
        ['get', 'post', 'put', 'delete', 'patch'].includes(m)
      );

      for (const method of methods) {
        const url = `http://localhost:3000/v2${path.replace(/{[^}]+}/g, 'test-id')}`;

        // শুধু check করছি endpoint exist করে কিনা (404 না 405 না)
        try {
          const res = await axios({ method, url, validateStatus: () => true });
          expect(res.status).not.toBe(404);
          expect(res.status).not.toBe(405); // Method Not Allowed
        } catch (e) {
          // Connection error মানেই endpoint নেই
          fail(`${method.toUpperCase()} ${path} endpoint accessible নয়`);
        }
      }
    }
  });

  test('Response schema spec-এর সাথে মিলছে কিনা', async () => {
    const Ajv = require('ajv');
    const ajv = new Ajv({ allErrors: true });

    const productSchema = spec.components.schemas.Product;
    const validate = ajv.compile(productSchema);

    const res = await axios.get('http://localhost:3000/v2/products');
    const firstProduct = res.data.data[0];

    const valid = validate(firstProduct);
    expect(valid).toBe(true);
  });
});
```

```php
// Laravel — PHPUnit দিয়ে doc validation
class ApiDocumentationTest extends TestCase
{
    private array $spec;

    protected function setUp(): void
    {
        parent::setUp();
        $specPath = storage_path('api-docs/api-docs.json');
        $this->spec = json_decode(file_get_contents($specPath), true);
    }

    public function test_all_documented_routes_exist(): void
    {
        $routes = collect(\Route::getRoutes())->map(fn($r) => [
            'uri' => $r->uri(),
            'methods' => $r->methods(),
        ]);

        foreach ($this->spec['paths'] as $path => $methods) {
            $laravelPath = 'api/v2' . str_replace(['{', '}'], ['{', '}'], $path);

            foreach (array_keys($methods) as $method) {
                if (in_array($method, ['get', 'post', 'put', 'delete', 'patch'])) {
                    $exists = $routes->contains(function ($route) use ($laravelPath, $method) {
                        return str_contains($route['uri'], $laravelPath)
                            && in_array(strtoupper($method), $route['methods']);
                    });

                    $this->assertTrue($exists, "Route {$method} {$path} documented but not implemented");
                }
            }
        }
    }

    public function test_product_response_matches_schema(): void
    {
        $product = Product::factory()->create();
        $response = $this->getJson("/api/v2/products/{$product->id}");

        $response->assertJsonStructure([
            'data' => ['id', 'name', 'price', 'currency', 'stock', 'status'],
        ]);
    }
}
```

### ১১. Developer Portal Design

একটি ভালো Developer Portal-এ ৩টি মূল অংশ থাকে:

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVELOPER PORTAL                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ GETTING      │  │ TUTORIALS    │  │ API REFERENCE    │   │
│  │ STARTED      │  │ (গাইড)       │  │ (রেফারেন্স)       │   │
│  │ (শুরু করুন)   │  │              │  │                  │   │
│  ├─────────────┤  ├──────────────┤  ├──────────────────┤   │
│  │ • সাইন আপ    │  │ • পণ্য তৈরি   │  │ • সব Endpoint   │   │
│  │ • API Key    │  │   শুরু থেকে    │  │ • Schema        │   │
│  │   পাওয়া      │  │   শেষ পর্যন্ত   │  │ • Error Codes   │   │
│  │ • প্রথম API  │  │ • Payment     │  │ • Rate Limits   │   │
│  │   Call       │  │   Integration │  │ • Auth Guide    │   │
│  │ • SDK        │  │ • Webhook     │  │ • Webhook       │   │
│  │   Install    │  │   Setup       │  │   Events        │   │
│  │ • Sandbox    │  │ • bKash       │  │ • Changelog     │   │
│  │   পরিচিতি    │  │   Integration │  │ • SDKs          │   │
│  └─────────────┘  └──────────────┘  └──────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ SUPPORT & COMMUNITY                                  │   │
│  │ • FAQ    • Status Page    • GitHub Issues            │   │
│  │ • Forum  • Discord/Slack  • support@api.com.bd      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**বাংলাদেশী উদাহরণ:**
- **bKash Developer Portal** — Merchant integration guide, sandbox environment, webhook documentation
- **SSLCommerz** — Payment API docs, IPN (Instant Payment Notification) guide
- **Pathao Courier API** — Delivery tracking, pricing calculation API

---

## ✅ Best Practices

```
✅ ১. উদাহরণ দিন — প্রতিটি endpoint-এ request ও response-এর বাস্তব উদাহরণ রাখুন
     (শুধু schema যথেষ্ট নয়, ডেভেলপাররা example দেখে বোঝে দ্রুত)

✅ ২. Error response document করুন — শুধু 200 নয়, 400/401/403/404/422/429/500 সব
     প্রতিটি error code-এর অর্থ ও সমাধান লিখুন

✅ ৩. Authentication guide সবার আগে — "কীভাবে শুরু করব?" প্রশ্নের উত্তর ৫ মিনিটে পাওয়া উচিত

✅ ৪. "Try it out" রাখুন — Sandbox environment দিন যেখানে real data affect হবে না
     bKash-এর sandbox environment এর ভালো উদাহরণ

✅ ৫. Versioning স্পষ্ট করুন — কোন version-এ কী পরিবর্তন হয়েছে, কোনটা deprecated

✅ ৬. Rate limiting document করুন — কত request/minute, limit অতিক্রম করলে কী হবে,
     Retry-After header কীভাবে ব্যবহার করবেন

✅ ৭. Pagination pattern consistent রাখুন — সব list endpoint-এ একই format

✅ ৮. SDK দিন — ডেভেলপারকে raw HTTP call লিখতে বাধ্য করবেন না
     OpenAPI spec থেকে auto-generate করুন

✅ ৯. Changelog maintain করুন — প্রতিটি release-এ কী বদলেছে

✅ ১০. CI/CD-তে doc validation যোগ করুন — spec আর actual API মিলছে কিনা auto-check

✅ ১১. Webhook documentation-এ retry policy, signature verification, ও example payload দিন

✅ ১২. বাংলায় লেখার কথা ভাবুন — বাংলাদেশী ডেভেলপারদের জন্য বাংলা ডক অনেক বেশি accessible
```

---

## ⚠️ Common Mistakes

```
❌ ১. Outdated Docs — কোড আপডেট হয়েছে কিন্তু ডক আপডেট হয়নি
     সমাধান: Auto-generation ব্যবহার করুন (Scribe, Scramble, swagger-jsdoc)
     CI-তে doc validation test যোগ করুন

❌ ২. Example নেই — শুধু schema দেওয়া, কোনো real example নেই
     সমাধান: প্রতিটি endpoint-এ কমপক্ষে ১টি success ও ১টি error example

❌ ৩. Error response undocumented — শুধু happy path document করা
     সমাধান: সব possible error code ও তাদের কারণ ও সমাধান লিখুন

❌ ৪. Authentication অস্পষ্ট — "API key লাগবে" লিখে কিন্তু কোথায় পাবে, কীভাবে পাঠাবে তা নেই
     সমাধান: Step-by-step auth guide — signup → key generate → first call

❌ ৫. Copy-paste error — একটি endpoint-এর ডক অন্য endpoint-এ paste করে name পরিবর্তন করা
     সমাধান: Code review-তে doc-ও check করুন

❌ ৬. Interactive testing নেই — ডক পড়ে বুঝতে পারছে কিন্তু try করতে পারছে না
     সমাধান: Swagger UI "Try it out" বা sandbox environment

❌ ৭. Pagination undocumented — list API আছে কিন্তু কীভাবে paginate করবে বলা নেই
     সমাধান: page/per_page parameter, response-এ meta object, next/prev link

❌ ৮. Breaking changes announce না করা — API-এ backward-incompatible change করে notice না দেওয়া
     সমাধান: Changelog + deprecation notice + migration guide

❌ ৯. একটিমাত্র format — শুধু PDF বা শুধু Wiki page
     সমাধান: Interactive UI (Swagger/Redoc) + machine-readable spec (JSON/YAML)

❌ ১০. Search নেই — ১০০+ endpoint-এর ডকে কোনো search functionality নেই
      সমাধান: Swagger UI/Redoc-এ built-in search আছে; portal-এ Algolia/Meilisearch
```

---

## 📋 Tools Comparison Table

```
┌──────────────────┬──────────────┬─────────────┬──────────┬─────────────┬──────────────┐
│ Tool             │ ধরন           │ ভাষা/ফ্রেমওয়ার্ক│ Approach │ Try-it-out  │ মূল্য         │
├──────────────────┼──────────────┼─────────────┼──────────┼─────────────┼──────────────┤
│ Swagger UI       │ UI Renderer  │ যেকোনো      │ উভয়      │ ✅          │ Free (OSS)   │
│ Redoc            │ UI Renderer  │ যেকোনো      │ উভয়      │ ❌ (Pro-তে) │ Free + Pro   │
│ L5-Swagger       │ Laravel Pkg  │ PHP/Laravel │ Code-1st │ ✅          │ Free (OSS)   │
│ Scribe           │ Laravel Pkg  │ PHP/Laravel │ Code-1st │ ✅          │ Free (OSS)   │
│ Scramble         │ Laravel Pkg  │ PHP/Laravel │ Code-1st │ ✅          │ Free (OSS)   │
│ swagger-jsdoc    │ Node Pkg     │ JS/Express  │ Code-1st │ ✅          │ Free (OSS)   │
│ tsoa             │ TS Framework │ TS/Express  │ Code-1st │ ✅          │ Free (OSS)   │
│ Postman          │ API Platform │ যেকোনো      │ উভয়      │ ✅          │ Free + Paid  │
│ Stoplight        │ Design Tool  │ যেকোনো      │ Design-1st│ ✅          │ Free + Paid  │
│ SwaggerHub       │ Platform     │ যেকোনো      │ Design-1st│ ✅          │ Free + Paid  │
│ ReadMe.com       │ Portal       │ যেকোনো      │ উভয়      │ ✅          │ Paid         │
│ GraphiQL         │ GraphQL IDE  │ GraphQL     │ Schema   │ ✅          │ Free (OSS)   │
│ Apollo Studio    │ GraphQL Plt  │ GraphQL     │ Schema   │ ✅          │ Free + Paid  │
│ protoc-gen-doc   │ gRPC Docs    │ Protobuf    │ Code-1st │ ❌          │ Free (OSS)   │
│ openapi-gen      │ SDK Gen      │ যেকোনো      │ Spec     │ N/A         │ Free (OSS)   │
│ Dredd            │ Doc Testing  │ যেকোনো      │ Spec     │ N/A         │ Free (OSS)   │
│ Schemathesis     │ Doc Testing  │ যেকোনো      │ Spec     │ N/A         │ Free (OSS)   │
│ Spectral         │ Linter       │ যেকোনো      │ Spec     │ N/A         │ Free (OSS)   │
└──────────────────┴──────────────┴─────────────┴──────────┴─────────────┴──────────────┘
```

---

## সারসংক্ষেপ

```
API ডকুমেন্টেশন = ডেভেলপার অভিজ্ঞতার (DX) ভিত্তি

মূল শিক্ষা:
━━━━━━━━━━━
১. OpenAPI Spec শিখুন — এটি industry standard, এখান থেকে সব generate হয়
২. Auto-generation ব্যবহার করুন — Scribe/Scramble (Laravel), swagger-jsdoc (Express)
৩. "Try it out" দিন — Swagger UI বা Sandbox দিয়ে interactive testing
৪. CI-তে validation যোগ করুন — ডক আর কোড sync আছে কিনা auto-check
৫. Error documentation অবহেলা করবেন না — happy path + error path দুটোই document করুন
৬. SDK generate করুন — openapi-generator দিয়ে client library বানান
৭. Changelog রাখুন — প্রতিটি API পরিবর্তন documented থাকুক

বাংলাদেশী প্রসঙ্গ:
━━━━━━━━━━━━━━━━━━
• bKash Developer Portal — ভালো sandbox + webhook doc-এর উদাহরণ
• SSLCommerz — IPN documentation, multi-currency support
• বাংলাদেশী startup-গুলো প্রায়ই ডক neglect করে — এটি একটি competitive advantage হতে পারে
• বাংলায় API ডকুমেন্টেশন লেখা নতুন ডেভেলপারদের জন্য barrier কমায়

প্রযুক্তি পছন্দ:
━━━━━━━━━━━━━━━
• Laravel → Scramble (zero-config) বা Scribe (বেশি control)
• Express → swagger-jsdoc + swagger-ui-express
• TypeScript → tsoa (type-safe route + doc generation)
• GraphQL → Apollo Studio বা GraphiQL
• gRPC → protoc-gen-doc
• Design-First → Stoplight Studio বা SwaggerHub
```

---

> **"ভালো API তৈরি করা কঠিন, কিন্তু ভালো API ডকুমেন্টেশন তৈরি করা আরও কঠিন — এবং আরও গুরুত্বপূর্ণ।"**
> 
> একটি API যতই শক্তিশালী হোক, ডকুমেন্টেশন ছাড়া সেটি ব্যবহারযোগ্য নয়। আপনার ডকুমেন্টেশনকে **প্রথম product** হিসেবে ভাবুন — কারণ আপনার API-র **প্রথম ব্যবহারকারী** একজন ডেভেলপার যে আপনার ডক পড়ছে।
