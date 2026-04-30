# 🌐 GraphQL Federation — Distributed GraphQL at Scale

## 📖 সংজ্ঞা ও মূল ধারণা

**GraphQL Federation** হলো একটি architecture pattern যেখানে একটি unified GraphQL API multiple independent **subgraphs** (microservices) থেকে compose করা হয়। প্রতিটি subgraph তার নিজের domain এর schema define করে; একটি **Gateway** (router) এদেরকে combine করে একটি **supergraph** তৈরি করে যেটা client-এর কাছে single endpoint হিসেবে exposed হয়। Apollo GraphOS এর Apollo Federation v2 (২০২২) বর্তমানে industry standard।

```
            ┌─────────────────────────┐
            │   Apollo Router/Gateway │
            │   (Supergraph schema)   │
            └──────────┬──────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌──────────┐
   │ Product │   │ Seller  │   │  Order  │   │ Reviews  │
   │subgraph │   │subgraph │   │subgraph │   │ subgraph │
   │ (Node)  │   │ (PHP)   │   │  (Go)   │   │ (Python) │
   └─────────┘   └─────────┘   └─────────┘   └──────────┘
```

**Key idea:** Schema ownership decentralized — প্রতিটা team তাদের domain-এর schema নিজে maintain করে; Gateway runtime-এ সব combine করে।

---

## 🆚 Federation v1 vs v2 vs Schema Stitching

### Schema Stitching (legacy)
পুরাতন approach — Gateway code-এ manually merge করা হতো; type conflict resolve করতে heavy custom code। Apollo deprecated।

### Federation v1 (২০১৯)
- `extend type` syntax
- Gateway query plan তৈরি করে subgraphs-এ delegate
- কিন্তু value types (e.g., currency object) duplicate define করতে হতো — error-prone

### Federation v2 (বর্তমানে standard)
- `@shareable`, `@inaccessible`, `@override`, `@interfaceObject`
- `extend` লাগে না — same type কয়েক subgraph-এ define করা যায়
- Composition rules অনেক flexible
- Better error messages
- Subscription support stable

| Feature | Stitching | Fed v1 | Fed v2 |
|---------|-----------|--------|--------|
| Schema composition | Manual code | `extend type` | Native, declarative |
| Type duplication | OK | Error | OK with `@shareable` |
| Override fields | Hard | Hard | `@override` |
| Interface objects | Manual | Limited | `@interfaceObject` ✅ |
| Tooling | Custom | Apollo | Apollo + Cosmo + GraphQL Hive |
| Industry adoption | Legacy | Sunsetting | ✅ Standard |

---

## 🔑 Core Directives

### `@key` — Entity identification
"এই type-এ এই field দিয়ে uniquely identify করা যায়।" Gateway entity resolution-এর জন্য এটা ব্যবহার করে।

```graphql
# Product subgraph
type Product @key(fields: "id") {
  id: ID!
  title: String!
  price: Float!
}
```

### `@key` (composite) — Multi-field key
```graphql
type Inventory @key(fields: "productId warehouseId") {
  productId: ID!
  warehouseId: ID!
  quantity: Int!
}
```

### `@shareable` (v2) — Multiple subgraphs same field define করতে পারে
```graphql
# Product subgraph
type Product @key(fields: "id") {
  id: ID!
  title: String! @shareable
}

# Search subgraph (denormalized for fast search)
type Product @key(fields: "id") {
  id: ID!
  title: String! @shareable   # same field, OK now
}
```

### `@external` — এই subgraph-এ field-টা actually নেই, অন্য subgraph-এ আছে
```graphql
# Reviews subgraph extends Product
type Product @key(fields: "id") {
  id: ID! @external
  reviews: [Review!]!
  averageRating: Float!
}
```

### `@requires` — এই field resolve করতে অন্য subgraph-এর field দরকার
```graphql
# Shipping subgraph
type Product @key(fields: "id") {
  id: ID! @external
  weight: Float! @external      # comes from Product subgraph
  shippingCost: Float! @requires(fields: "weight")
}
```

### `@provides` — এই subgraph external field-এর partial value দিতে পারবে (avoid extra hop)
```graphql
type Order @key(fields: "id") {
  id: ID!
  product: Product @provides(fields: "title")  # avoid Product subgraph hop for title
}

type Product @key(fields: "id") {
  id: ID! @external
  title: String! @external
}
```

### `@override` — Field ownership অন্য subgraph থেকে নিয়ে নাও (migration use case)
```graphql
# new Pricing subgraph takes over Product.price
type Product @key(fields: "id") {
  id: ID!
  price: Float! @override(from: "ProductSubgraph")
}
```

### `@inaccessible` — Public supergraph-এ hide কিন্তু internally available
```graphql
type Product @key(fields: "id") {
  id: ID!
  internalCost: Float! @inaccessible
}
```

### `@interfaceObject` (v2) — Interface implementation distribute
```graphql
# Reviews subgraph adds reviews to ALL types implementing Media
type Media @interfaceObject @key(fields: "id") {
  id: ID!
  reviews: [Review!]!
}
```

---

## 🔄 Entity Resolution Flow

```
Client Query:
  query {
    order(id: "O-100") {
      id
      total
      product { title, averageRating, shippingCost }
    }
  }

Gateway Query Plan:
  Step 1 → Order subgraph: get order(id: "O-100") → { id, total, productId }
  Step 2 → Product subgraph: _entities([{__typename:"Product", id: pid}]) → { title, weight }
           (parallel) Reviews subgraph: _entities → { averageRating }
  Step 3 → Shipping subgraph: _entities([{__typename:"Product", id, weight}]) → { shippingCost }
            (because @requires weight)
  Step 4 → Merge & return
```

**`_entities` query:** Federation auto-generates এই magical query যা `representations` (entity reference + key fields) accept করে।

```graphql
query {
  _entities(representations: [
    { __typename: "Product", id: "P-1" }
    { __typename: "Product", id: "P-2" }
  ]) {
    ... on Product { title price }
  }
}
```

---

## 🏗️ Composition & Supergraph Schema

প্রতিটা subgraph-এর schema থেকে **supergraph SDL** generate হয় build time-এ:

```bash
# Apollo Rover CLI
rover supergraph compose --config supergraph-config.yaml > supergraph.graphql
```

`supergraph-config.yaml`:
```yaml
federation_version: 2
subgraphs:
  product:
    routing_url: http://product-svc:4001/graphql
    schema:
      subgraph_url: http://product-svc:4001/graphql
  seller:
    routing_url: http://seller-svc:4002/graphql
    schema:
      subgraph_url: http://seller-svc:4002/graphql
  reviews:
    routing_url: http://reviews-svc:4003/graphql
    schema:
      subgraph_url: http://reviews-svc:4003/graphql
```

**Hot reload:** Apollo Router schema URL poll করে; subgraph deploy হলে নতুন supergraph তৈরি, router runtime-এ reload (zero downtime)।

---

## 💻 Node.js Subgraph Example — Daraz Product Subgraph

```javascript
// product-subgraph/src/server.js
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import { buildSubgraphSchema } from '@apollo/subgraph';
import gql from 'graphql-tag';

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.7",
          import: ["@key", "@shareable", "@inaccessible"])

  type Product @key(fields: "id") {
    id: ID!
    title: String! @shareable
    description: String
    price: Float!
    weight: Float!                   # used by Shipping via @requires
    sellerId: ID!
    categoryId: ID!
    images: [Image!]!
    internalCost: Float! @inaccessible  # admin-only
  }

  type Image {
    url: String!
    thumbnail: String!
    altText: String
  }

  type Query {
    product(id: ID!): Product
    productsByCategory(categoryId: ID!, limit: Int = 20): [Product!]!
    searchProducts(query: String!): [Product!]!
  }
`;

const resolvers = {
  Query: {
    product: async (_, { id }, { dataSources }) => dataSources.productAPI.byId(id),
    productsByCategory: async (_, args, ctx) => ctx.dataSources.productAPI.byCategory(args),
    searchProducts: async (_, { query }, ctx) => ctx.dataSources.searchAPI.find(query),
  },
  Product: {
    // CRITICAL: __resolveReference for entity resolution
    __resolveReference: async (ref, { dataSources }) => {
      // Gateway calls this when another subgraph references Product
      return dataSources.productAPI.byId(ref.id);
    },
  },
};

const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
});

const { url } = await startStandaloneServer(server, {
  listen: { port: 4001 },
  context: async () => ({
    dataSources: { /* product DB, search ES */ },
  }),
});
console.log(`Product subgraph at ${url}`);
```

### Reviews Subgraph (extends Product)

```javascript
// reviews-subgraph/src/server.js
import { ApolloServer } from '@apollo/server';
import { buildSubgraphSchema } from '@apollo/subgraph';
import gql from 'graphql-tag';
import DataLoader from 'dataloader';

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.7",
          import: ["@key", "@external"])

  type Review {
    id: ID!
    rating: Int!
    body: String
    authorName: String
    createdAt: String!
  }

  type Product @key(fields: "id") {
    id: ID! @external
    reviews(limit: Int = 10): [Review!]!
    averageRating: Float!
    reviewCount: Int!
  }
`;

const resolvers = {
  Product: {
    __resolveReference: (ref) => ({ id: ref.id }), // just need id
    reviews: async (product, { limit }, { loaders }) =>
      loaders.reviewsByProduct.load(`${product.id}:${limit}`),
    averageRating: async (product, _, { loaders }) =>
      loaders.avgRating.load(product.id),
    reviewCount: async (product, _, { loaders }) =>
      loaders.reviewCount.load(product.id),
  },
};

// DataLoader prevents N+1 in batched _entities calls
function buildLoaders(db) {
  return {
    avgRating: new DataLoader(async (productIds) => {
      const rows = await db.query(
        'SELECT product_id, AVG(rating) as avg FROM reviews WHERE product_id = ANY($1) GROUP BY product_id',
        [productIds]
      );
      const map = new Map(rows.map(r => [r.product_id, r.avg]));
      return productIds.map(id => map.get(id) ?? 0);
    }),
    reviewCount: new DataLoader(/* ... */),
    reviewsByProduct: new DataLoader(/* ... */),
  };
}
```

### Shipping Subgraph (uses @requires)

```javascript
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.7",
          import: ["@key", "@external", "@requires"])

  type Product @key(fields: "id") {
    id: ID! @external
    weight: Float! @external
    shippingCost(district: String!): Float! @requires(fields: "weight")
  }
`;

const resolvers = {
  Product: {
    __resolveReference: (ref) => ref, // weight will be passed automatically
    shippingCost: ({ weight }, { district }) => {
      // Bangladesh district-based shipping
      const baseRate = district === 'dhaka' ? 60 : 120;
      return baseRate + Math.ceil(weight / 0.5) * 10;
    },
  },
};
```

### Apollo Router (Gateway)

```yaml
# router.yaml
supergraph:
  listen: 0.0.0.0:4000
  introspection: false   # production-এ off
  
cors:
  origins:
    - https://daraz.com.bd
    - https://m.daraz.com.bd

traffic_shaping:
  all:
    timeout: 30s
  subgraphs:
    reviews:
      timeout: 5s        # non-critical
      experimental_retry:
        min_per_sec: 10

telemetry:
  metrics:
    prometheus:
      enabled: true
```

```bash
# Run router
./router --config router.yaml --supergraph supergraph.graphql
```

---

## 🐘 PHP Federation Subgraph (Lighthouse / GraphQLite)

PHP-এ Apollo official subgraph library নেই, কিন্তু `nuwave/lighthouse` (Laravel) ও `wedevs/wp-graphql-federation` Federation v2 spec implement করে।

```php
// app/GraphQL/schema.graphql (Lighthouse + Federation)
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0",
        import: ["@key", "@shareable"])

type Seller @key(fields: "id") {
  id: ID!
  businessName: String! @shareable
  rating: Float!
  joinedAt: DateTime!
  productsCount: Int!
}

type Query {
  seller(id: ID!): Seller @find
}
```

```php
// app/GraphQL/EntityResolvers/SellerResolver.php
namespace App\GraphQL\EntityResolvers;

use Nuwave\Lighthouse\Federation\Resolvers\EntityResolverProvider;

class SellerResolver
{
    /**
     * Federation _entities resolver
     */
    public static function __resolveReference(array $representation): ?\App\Models\Seller
    {
        return \App\Models\Seller::find($representation['id']);
    }
}
```

```php
// config/lighthouse.php
'federation' => [
    'entities_resolver_namespace' => 'App\\GraphQL\\EntityResolvers',
],
```

---

## ⚖️ Federation vs Monolithic GraphQL

| মানদণ্ড | Monolithic GraphQL | Federation |
|---------|---------------------|------------|
| Team autonomy | Low — single repo, deploy bottleneck | High — per-team subgraph |
| Schema ownership | Shared (conflict-prone) | Distributed |
| Deployment | একসাথে সব | Independent per subgraph |
| Performance | Single hop | Multi-hop (gateway → subgraph) |
| Complexity | Low | High (composition, query planning) |
| Schema validation | Trivial | Composition checks needed in CI |
| Best for | Small/medium teams, single domain | Large org, multiple domains |
| Tooling overhead | Minimal | Apollo Studio/GraphOS, Rover, schema registry |

> **Rule of thumb:** ৩+ team, ৫+ domain না হলে Federation এ যাবেন না — single GraphQL server (with modules) যথেষ্ট।

---

## 🛠️ Tooling Ecosystem

- **Apollo GraphOS / Apollo Studio** — managed schema registry, schema check, metrics।
- **WunderGraph Cosmo** — open source Apollo alternative।
- **GraphQL Hive** (The Guild) — open source schema registry, federation support।
- **Rover CLI** — `rover subgraph publish`, `rover supergraph compose`, schema check।
- **Apollo Router (Rust)** — production gateway, faster than legacy Apollo Gateway (Node)।
- **GraphQL Mesh** — non-Apollo alternative, also stitches REST/gRPC into federation।

---

## 🚦 CI/CD Schema Workflow

```
Developer pushes subgraph schema change
              │
              ▼
   ┌──────────────────────┐
   │ rover subgraph check │   ← Static composition check
   │ (against current     │     + breaking change detection
   │  supergraph)          │
   └──────────┬───────────┘
              │ pass
              ▼
   ┌──────────────────────┐
   │ Operation check      │   ← Real client queries replay
   │ (registered queries) │
   └──────────┬───────────┘
              │ pass
              ▼
   ┌──────────────────────┐
   │ Deploy subgraph      │
   │ rover subgraph publish│
   └──────────┬───────────┘
              │
              ▼
   Apollo Router auto-pulls new supergraph (uplink)
```

---

## ✅ কখন Federation ব্যবহার করবেন

- ৩+ team, প্রতিটার নিজস্ব domain (product, payment, order, user...)
- Microservices already exist, GraphQL দিয়ে unify করতে চান
- Independent deploy cadence ও schema ownership দরকার
- Single endpoint, polyglot backend (Node + Go + PHP) চান
- Schema discovery দরকার — frontend dev কোথায় কোন field find করবে clear

## ❌ কখন এড়াবেন

- ছোট team (১-২), single repo enough → monolithic GraphQL ভালো
- Microservices আগে নেই → REST/gRPC থেকেই start
- Latency-critical path (Federation gateway 5-20ms overhead যোগ করে)
- Mutation-heavy app — federation mutations cross-subgraph transactions complex
- Tooling/operational maturity নেই (schema registry, observability)

---

## ⚠️ Pitfalls

1. **N+1 in `__resolveReference`** — Gateway একসাথে অনেক entity-এর reference পাঠায়; প্রতিটার জন্য আলাদা DB query করলে disaster। **DataLoader** mandatory।
2. **Circular @requires** — Subgraph A requires field from B, B requires from A → query plan fail।
3. **Shared types without `@shareable`** — Composition error। প্রতিটা type ownership clear করুন।
4. **Auth context propagation** — Gateway থেকে subgraph-এ JWT/user context forward করতে হবে (header pass-through configure)।
5. **Schema drift** — Subgraph local-এ যা আছে, registry-এ ভিন্ন → composition fail। CI check mandatory।
6. **Mutation atomicity** — Cross-subgraph mutation atomic না (no distributed transaction)। Saga pattern ব্যবহার।
7. **Subscription complexity** — Federation v2-এ subscription আছে কিন্তু single subgraph-এ rooted; cross-subgraph subscription difficult।
8. **Over-federation** — সব service-কে subgraph করতে গেলে gateway query plan complex। Bounded context-এ subgraph রাখুন।
9. **Caching difficulty** — Per-entity cache (e.g., Apollo Server entity cache) configure না করলে DB hammer।
10. **Versioning** — Federation-এ version নেই; backward-compatible evolution করতে হয় (deprecate, then remove)।

---

## 🇧🇩 Real-world Example — Daraz BD Federation Architecture

```
                  ┌──────────────────┐
                  │ Daraz Web/Mobile │
                  │   (clients)      │
                  └────────┬─────────┘
                           │ single endpoint
                           ▼
                ┌─────────────────────┐
                │   Apollo Router     │
                │ (graphql.daraz.bd)  │
                └────────┬────────────┘
                         │
   ┌───────┬─────────┬───┴────┬─────────┬──────────┐
   ▼       ▼         ▼        ▼         ▼          ▼
┌──────┐┌──────┐┌────────┐┌────────┐┌────────┐┌────────┐
│Catalog││Seller││Inventory││Order  ││Reviews ││Shipping│
│ team  ││ team ││  team  ││ team  ││ team   ││ team   │
│(Node) ││(PHP) ││ (Go)   ││(Java) ││(Python)││ (Go)   │
└──────┘└──────┘└────────┘└────────┘└────────┘└────────┘
```

**Daraz এ benefit:**
- Catalog team নতুন field add করলে Seller team-এর deploy বন্ধ করতে হয় না
- Mobile app ১টা endpoint থেকে product + reviews + shipping cost সব পায়
- Polyglot — legacy PHP services-ও subgraph হিসেবে join করতে পারে

**অন্যান্য BD use cases:**
- **bKash** — Payment, Customer, Merchant, Settlement, Compliance subgraphs
- **Pathao** — Ride, Food, Parcel, Pay subgraphs একই supergraph-এ (যদিও Pathao actually internally REST-heavy)
- **Shohoz** — Bus, Launch, Train, Hotel domains; federation আদর্শ
- **Foodpanda** — Restaurant, Menu, Cart, Rider, Promo subgraphs

---

## 📊 Performance Considerations

| Aspect | Tip |
|--------|-----|
| Gateway latency | Apollo Router (Rust) > Apollo Gateway (Node) — 5x faster |
| Query planning | Cache plans (router does this automatically) |
| Subgraph fan-out | Use `@provides` to skip extra hops |
| Entity resolution | DataLoader + batch `_entities` query |
| Caching | Apollo Cache Control headers + CDN, per-field TTL |
| Persisted queries | Reduce payload, prevent injection — APQ enabled |
| Authorization | Use `@requiresScopes` (Fed v2.5+) directive |

---

## 📝 সারসংক্ষেপ

- **Federation = distributed GraphQL।** একাধিক subgraph থেকে gateway runtime-এ একটি unified supergraph compose করে।
- **Apollo Federation v2** বর্তমানে standard; Schema Stitching ও v1 deprecated।
- **Core directives:** `@key`, `@shareable`, `@external`, `@requires`, `@provides`, `@override`, `@inaccessible`, `@interfaceObject`।
- Gateway entity resolution-এর জন্য `__resolveReference` আর `_entities` query magical — DataLoader দিয়ে N+1 ঠেকান।
- **Trade-off:** Team autonomy ↑ vs Operational complexity ↑। ৩+ team না হলে monolithic GraphQL যথেষ্ট।
- **CI/CD:** `rover subgraph check` দিয়ে breaking change ও composition validate; schema registry mandatory।
- Apollo Router (Rust), GraphQL Hive, WunderGraph Cosmo — production-grade tooling।
- **BD-তে Daraz, bKash, Shohoz** — multi-domain platforms-এ federation perfectly fit।
- **Pitfalls:** N+1 in entity resolution, auth propagation, cross-subgraph transactions, schema drift।
