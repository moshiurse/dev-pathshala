# 🔔 GraphQL Subscriptions — Real-time Server-Pushed Updates

> **Query/Mutation যথেষ্ট নয় যখন UI-কে real-time live থাকতে হয় — order status, chat, dashboard, collaborative editing।**
> এই গাইডে আমরা GraphQL Subscription deep dive করব: transport (graphql-ws, SSE), PubSub backend (Redis, Kafka, RabbitMQ), Apollo + Lighthouse PHP server example, scaling, এবং Daraz/Foodpanda/bKash এর real-world case study।

---

## 📖 সূচিপত্র

- [Subscription কেন? Query/Mutation এর বাইরে](#-subscription-কেন)
- [Transport Options (graphql-ws, SSE)](#-transport-options)
- [Schema Design](#-schema-design)
- [Server Implementations](#-server-implementations)
- [PubSub Backend Trade-offs](#-pubsub-backend)
- [Code: Apollo + Redis (Pathao Foods)](#-code-apollo-server--redis-pubsub)
- [Code: Lighthouse PHP (Daraz Seller)](#-code-laravel-lighthouse-php)
- [Authorization in Subscriptions](#-authorization)
- [Scaling Subscriptions](#-scaling-subscriptions)
- [Subscriptions vs SSE vs WebSocket vs Polling](#-comparison--decision-matrix)
- [Live Queries (alternative)](#-live-queries-alternative)
- [Operational Concerns](#-operational-concerns)
- [Anti-patterns ও BD Case Studies](#-anti-patterns)
- [Checklist](#-checklist)

---

## 🎯 Subscription কেন?

GraphQL-এ তিনটি root operation আছে:
- **Query**: পড়ো (request-response)
- **Mutation**: লেখো (request-response)
- **Subscription**: শোনো (server-pushed stream)

```
                Query / Mutation              Subscription
              ┌───────────────────┐         ┌─────────────────┐
              │  Client → Server  │         │  Client → Server│
              │       request     │         │      subscribe  │
              │  Server → Client  │         │  Server ──┐     │
              │       response    │         │           │     │
              │      (one-shot)   │         │  Server ◄─┘     │
              └───────────────────┘         │      event      │
                                            │  Server ──┐     │
                                            │  Server ◄─┘     │
                                            │      event...   │
                                            └─────────────────┘
```

### Real-time Use Case

- **Live order tracking**: Pathao Foods order status (placed → cooking → out_for_delivery → delivered)
- **Chat / Comments**: Daraz seller chat, Foodpanda customer support
- **Dashboards**: bKash agent live transaction feed, admin metrics
- **Collaborative editing**: shared form, document
- **Notifications**: new message, friend request
- **IoT telemetry**: sensor data dashboard
- **Live auctions/bidding**: real-time price update

---

## 🚇 Transport Options

GraphQL spec subscription transport specify করে না — community standard আছে।

### ১. `graphql-ws` (recommended, modern)

- Protocol: `graphql-transport-ws` (২০২০+)
- Transport: WebSocket
- Library: `graphql-ws` (server + client)
- Features: connection lifecycle (`connection_init`, `connection_ack`, ping/pong), per-operation subscribe/complete, error frames

```
Client                          Server (graphql-ws)
  │── WS upgrade with        ─►│
  │   subprotocol             │
  │   "graphql-transport-ws"  │
  │◄────── 101 Switching ─────│
  │                           │
  │── connection_init ───────►│  (auth payload)
  │◄────── connection_ack ────│
  │                           │
  │── subscribe (id=1, query)►│
  │◄────── next (id=1, data) ─│  ← event 1
  │◄────── next (id=1, data) ─│  ← event 2
  │── complete (id=1) ───────►│  (cancel)
  │                           │
  │── ping ──────────────────►│
  │◄───── pong ───────────────│
```

### ২. `subscriptions-transport-ws` (legacy, **deprecated**)

- পুরোনো Apollo protocol (`graphql-ws` subprotocol name conflict — confusing!)
- নতুন project-এ ব্যবহার করবেন না
- পুরোনো client সাথে compat-এর জন্য Apollo Server এখনো support করতে পারে

### ৩. SSE (`graphql-sse`)

- Server-Sent Events-এর উপর — see `06-api-design/server-sent-events.md`
- Pros: HTTP/1.1+HTTP/2 friendly, simpler proxy, auto-reconnect built-in
- Cons: server→client only (mutation আলাদা HTTP POST)
- লাইব্রেরি: `graphql-sse`
- ভালো ফিট: read-heavy notification, restrictive corporate firewall

```
Client                         Server (graphql-sse)
  │── GET /graphql/stream ────►│  (SSE)
  │   query=subscription...    │
  │◄── 200 text/event-stream ──│
  │                            │
  │◄── event: next             │
  │    data: {...}             │
  │◄── event: next             │
  │    data: {...}             │
  │◄── event: complete         │
```

### ৪. HTTP Multipart (`@defer`/`@stream` — different concept)

- Subscription নয়! এগুলো single query-কে chunked response।
- Initial render fast → slow field পরে আসে।
- transport: `multipart/mixed`
- Apollo, Yoga, Relay support করে।

```graphql
query {
  feed { id, title }
  ...slowSection @defer
}

fragment slowSection on Feed {
  comments { ... }
  recommendations { ... }
}
```

---

## 🧱 Schema Design

```graphql
type Subscription {
  # বিশেষ orderId-র status update — filter argument
  orderStatusUpdated(orderId: ID!): OrderStatusEvent!

  # সব new order — admin
  orderCreated: Order!

  # chat room
  messageAdded(roomId: ID!): Message!

  # generic typing indicator
  typing(roomId: ID!): TypingEvent!
}

type OrderStatusEvent {
  orderId: ID!
  status: OrderStatus!
  occurredAt: DateTime!
  rider: Rider          # nullable
  estimatedDelivery: DateTime
}

enum OrderStatus {
  PLACED
  ACCEPTED
  COOKING
  READY_FOR_PICKUP
  OUT_FOR_DELIVERY
  DELIVERED
  CANCELLED
}
```

### Payload Design — Full State vs Delta

| | Full State | Delta |
|-|-----------|-------|
| Bandwidth | বেশি | কম |
| Client logic | simple | merge/patch logic |
| Out-of-order? | safe | সমস্যা হতে পারে |
| Initial sync? | implicit | আলাদা query লাগে |
| Best for | small payload, low frequency | high-frequency, large doc |

**Recommended pattern**: subscription-এর payload-এ "what changed + version/timestamp" পাঠান, client cached state update করুক। Initial state alongside Query দিয়ে নিন।

```graphql
query {
  order(id: "ord-123") { ... }    # initial snapshot
}
subscription {
  orderStatusUpdated(orderId: "ord-123") { ... }   # updates
}
```

### Schema Evolution

- Subscription field add করা — safe।
- Field remove — breaking; subscriber ভাঙে।
- Argument add (optional) — safe।
- Filter argument shape change — versioned name use করুন (`messageAddedV2`)।
- Payload type-এ field add — safe; remove — breaking।

---

## 🛠️ Server Implementations

### Apollo Server (Node, TS) + graphql-ws

Most popular Node stack। `@apollo/server` v4 + `graphql-ws` + `useServer` wiring।

### GraphQL Yoga (Node) + graphql-sse

Lightweight, Envelop plugin system, SSE built-in।

### Mercurius (Fastify, Node)

High perf, gRPC-like federation, built-in subscription।

### PHP Stack — Lighthouse (Laravel)

Daraz, Pathao backend (PHP-heavy)। Lighthouse-এর built-in subscription Pusher / Redis backend দিয়ে।

### .NET — Hot Chocolate

Strongly-typed, source-generator-based subscription।

### Hasura, PostGraphile

Postgres-উপর stand করে। **Hasura live queries**: automatic — DB row change → push। Powerful for prototyping।

### Federation

Apollo Federation v2 subscriptions support — single subgraph subscription field, gateway forward করে। Limited compared to Query: cross-subgraph join সাবধানে।

```
Client ──► Gateway ──► OrderSubgraph (subscription field)
              │
              ├──► UserSubgraph (entity reference)
              └──► ProductSubgraph (entity reference)
```

---

## 📨 PubSub Backend

Subscription-এর underlying message bus — কিভাবে server pod গুলো event share করে।

### ১. In-memory (`PubSub` from `graphql-subscriptions`)

```javascript
import { PubSub } from 'graphql-subscriptions';
const pubsub = new PubSub();
```

- ✅ Dev/test
- ❌ Production: single-pod only (not horizontal-scalable)
- ❌ Restart-এ subscription drop

### ২. Redis Pub/Sub (`graphql-redis-subscriptions`)

```javascript
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

const opts = { host: 'redis.local', port: 6379 };
const pubsub = new RedisPubSub({
  publisher: new Redis(opts),
  subscriber: new Redis(opts)
});
```

- ✅ Multi-pod, low latency
- ❌ **Lossy**: subscriber down থাকলে message miss
- ❌ No replay
- বেশিরভাগ web app-এর জন্য enough (delivery-guarantee বড় ব্যাপার না হলে)

### ৩. Redis Streams

- ✅ Durable, consumer group, replay
- ✅ Bounded retention (`MAXLEN`)
- ❌ Library support কম mature (custom adapter)
- Use case: order tracking যেখানে miss করা যাবে না

### ৪. Apache Kafka

- ✅ Durable, partitioned, infinite replay
- ✅ Massive throughput
- ❌ Operational complexity
- ✅ Best for: large enterprise, audit-trail (bKash transactions)
- See `18-kafka/` for deep dive

### ৫. RabbitMQ / AMQP

- ✅ Durable, ack semantics, fanout/topic exchange
- ✅ Per-subscriber queue
- See `rabbitmq-amqp.md`
- Use case: medium-scale real-time

### Trade-off Table

| Backend | Throughput | Latency | Durable | Replay | Ops cost |
|---------|-----------|---------|---------|--------|----------|
| In-memory | very high | μs | ❌ | ❌ | শূন্য |
| Redis Pub/Sub | high | <1ms | ❌ | ❌ | কম |
| Redis Streams | high | <2ms | ✅ | ✅ (bounded) | কম |
| Kafka | massive | 2-10ms | ✅ | ✅ (huge) | বেশি |
| RabbitMQ | medium-high | <5ms | ✅ | limited | মাঝারি |

---

## 💻 Code: Apollo Server + Redis PubSub

**Scenario**: Pathao Foods customer-এর live order tracking।

```typescript
// schema.ts
import gql from 'graphql-tag';

export const typeDefs = gql`
  type Query {
    order(id: ID!): Order
    me: User
  }
  type Mutation {
    updateOrderStatus(orderId: ID!, status: OrderStatus!): Order!
  }
  type Subscription {
    orderStatusUpdated(orderId: ID!): OrderStatusEvent!
  }

  type Order {
    id: ID!
    customerId: ID!
    status: OrderStatus!
    items: [OrderItem!]!
    rider: Rider
    estimatedDelivery: String
  }
  type OrderItem { id: ID!, name: String!, qty: Int! }
  type Rider { id: ID!, name: String!, phone: String! }
  type User { id: ID!, name: String! }
  type OrderStatusEvent {
    orderId: ID!
    status: OrderStatus!
    occurredAt: String!
    rider: Rider
    estimatedDelivery: String
  }
  enum OrderStatus {
    PLACED ACCEPTED COOKING READY_FOR_PICKUP
    OUT_FOR_DELIVERY DELIVERED CANCELLED
  }
`;
```

```typescript
// pubsub.ts
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

const opts = {
  host: process.env.REDIS_HOST!,
  port: Number(process.env.REDIS_PORT ?? 6379),
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (t: number) => Math.min(t * 50, 2000)
};

export const pubsub = new RedisPubSub({
  publisher: new Redis(opts),
  subscriber: new Redis(opts)
});

export const TOPICS = {
  ORDER_STATUS: (id: string) => `ORDER_STATUS:${id}`
};
```

```typescript
// resolvers.ts
import { pubsub, TOPICS } from './pubsub';
import { withFilter } from 'graphql-subscriptions';
import { GraphQLError } from 'graphql';
import { OrderModel } from './models';

export const resolvers = {
  Query: {
    order: async (_: any, { id }: any, ctx: any) => {
      const order = await OrderModel.findById(id);
      if (!order) return null;
      if (order.customerId !== ctx.user.id && !ctx.user.isAdmin) {
        throw new GraphQLError('Forbidden', { extensions: { code: 'FORBIDDEN' } });
      }
      return order;
    }
  },

  Mutation: {
    updateOrderStatus: async (_: any, { orderId, status }: any, ctx: any) => {
      if (!ctx.user.isAdmin && !ctx.user.isRider) {
        throw new GraphQLError('Forbidden');
      }
      const order = await OrderModel.update(orderId, { status });

      // Publish event to all subscribers of this order
      await pubsub.publish(TOPICS.ORDER_STATUS(orderId), {
        orderStatusUpdated: {
          orderId,
          status,
          occurredAt: new Date().toISOString(),
          rider: order.rider,
          estimatedDelivery: order.estimatedDelivery
        }
      });

      return order;
    }
  },

  Subscription: {
    orderStatusUpdated: {
      subscribe: withFilter(
        (_, { orderId }) =>
          pubsub.asyncIterator([TOPICS.ORDER_STATUS(orderId)]),
        // per-event auth filter (extra safety)
        async (payload, vars, ctx) => {
          const order = await OrderModel.findById(vars.orderId);
          if (!order) return false;
          return (
            order.customerId === ctx.user.id ||
            ctx.user.isAdmin ||
            (ctx.user.isRider && order.riderId === ctx.user.id)
          );
        }
      )
    }
  }
};
```

```typescript
// server.ts
import express from 'express';
import { createServer } from 'http';
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import bodyParser from 'body-parser';
import cors from 'cors';
import { typeDefs } from './schema';
import { resolvers } from './resolvers';
import { verifyJwt } from './auth';

const app = express();
const httpServer = createServer(app);
const schema = makeExecutableSchema({ typeDefs, resolvers });

// WebSocket server for subscriptions
const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql'
});

const serverCleanup = useServer(
  {
    schema,
    // connection-level auth (token in connectionParams)
    context: async (ctx, msg, args) => {
      const token =
        (ctx.connectionParams?.authorization as string)?.replace('Bearer ', '');
      const user = token ? await verifyJwt(token) : null;
      if (!user) throw new Error('Unauthorized');
      return { user };
    },
    onConnect: async (ctx) => {
      console.log('WS connected', ctx.connectionParams);
    },
    onDisconnect: (ctx, code, reason) => {
      console.log('WS disconnected', code, reason);
    },
    keepAlive: 12_000   // ping every 12s
  },
  wsServer
);

const apollo = new ApolloServer({
  schema,
  plugins: [
    ApolloServerPluginDrainHttpServer({ httpServer }),
    {
      async serverWillStart() {
        return { async drainServer() { await serverCleanup.dispose(); } };
      }
    }
  ]
});

await apollo.start();

app.use('/graphql', cors(), bodyParser.json(),
  expressMiddleware(apollo, {
    context: async ({ req }) => {
      const token = req.headers.authorization?.replace('Bearer ', '');
      const user = token ? await verifyJwt(token) : null;
      return { user };
    }
  })
);

httpServer.listen(4000, () =>
  console.log('Pathao Foods GraphQL :4000 (HTTP + WS)')
);
```

### Client (React + Apollo Client + graphql-ws)

```typescript
// apollo-client.ts
import { ApolloClient, InMemoryCache, HttpLink, split } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';

const httpLink = new HttpLink({
  uri: 'https://api.pathao-foods.com/graphql',
  headers: { authorization: `Bearer ${localStorage.token}` }
});

const wsLink = new GraphQLWsLink(createClient({
  url: 'wss://api.pathao-foods.com/graphql',
  connectionParams: () => ({
    authorization: `Bearer ${localStorage.token}`
  }),
  retryAttempts: 10,
  shouldRetry: () => true,
  keepAlive: 10_000
}));

const splitLink = split(
  ({ query }) => {
    const def = getMainDefinition(query);
    return def.kind === 'OperationDefinition' && def.operation === 'subscription';
  },
  wsLink,
  httpLink
);

export const apolloClient = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache()
});
```

```tsx
// OrderTracker.tsx
import { gql, useQuery, useSubscription } from '@apollo/client';

const GET_ORDER = gql`
  query GetOrder($id: ID!) {
    order(id: $id) {
      id status
      items { id name qty }
      rider { name phone }
      estimatedDelivery
    }
  }
`;

const ORDER_UPDATES = gql`
  subscription OnOrder($id: ID!) {
    orderStatusUpdated(orderId: $id) {
      orderId status occurredAt
      rider { name phone }
      estimatedDelivery
    }
  }
`;

export function OrderTracker({ orderId }: { orderId: string }) {
  const { data, loading } = useQuery(GET_ORDER, { variables: { id: orderId } });

  useSubscription(ORDER_UPDATES, {
    variables: { id: orderId },
    onData: ({ client, data: { data: sub } }) => {
      const ev = sub.orderStatusUpdated;
      // Update Apollo cache directly
      client.cache.modify({
        id: client.cache.identify({ __typename: 'Order', id: ev.orderId }),
        fields: {
          status: () => ev.status,
          rider: () => ev.rider,
          estimatedDelivery: () => ev.estimatedDelivery
        }
      });
    }
  });

  if (loading) return <p>লোড হচ্ছে...</p>;
  return (
    <div>
      <h2>Order #{data.order.id}</h2>
      <p>Status: <strong>{data.order.status}</strong></p>
      {data.order.rider && (
        <p>Rider: {data.order.rider.name} ({data.order.rider.phone})</p>
      )}
      <p>ETA: {data.order.estimatedDelivery}</p>
    </div>
  );
}
```

---

## 🐘 Code: Laravel Lighthouse (PHP)

**Scenario**: Daraz seller dashboard — new order, low-stock alert real-time।

### `composer.json`

```json
{
  "require": {
    "php": "^8.2",
    "laravel/framework": "^11.0",
    "nuwave/lighthouse": "^6.0",
    "pusher/pusher-php-server": "^7.0"
  }
}
```

### `config/lighthouse.php`

```php
'subscriptions' => [
    'queue_broadcasts' => env('LIGHTHOUSE_QUEUE_BROADCASTS', true),
    'broadcaster' => env('LIGHTHOUSE_BROADCASTER', 'pusher'),
    'broadcasters' => [
        'pusher' => [
            'driver' => 'pusher',
            'connection' => 'pusher',
        ],
        'echo' => [
            'driver' => 'echo',
            'connection' => 'redis',  // Laravel Echo Server / Soketi
        ],
    ],
    'storage' => env('LIGHTHOUSE_SUBSCRIPTION_STORAGE', 'redis'),
    'storage_ttl' => 60 * 60 * 24,   // 1 day
],
```

### `graphql/schema.graphql`

```graphql
type Order {
  id: ID!
  customer: User! @belongsTo
  total: Float!
  status: OrderStatus!
  createdAt: DateTime!
}

type Query {
  myOrders: [Order!]! @auth @paginate(model: "App\\Models\\Order")
}

type Mutation {
  updateOrderStatus(id: ID!, status: OrderStatus!): Order!
    @field(resolver: "App\\GraphQL\\Mutations\\UpdateOrderStatus")
    @guard
}

type Subscription {
  newOrder(sellerId: ID!): Order!
    @subscription(class: "App\\GraphQL\\Subscriptions\\NewOrder")

  lowStock(sellerId: ID!): Product!
    @subscription(class: "App\\GraphQL\\Subscriptions\\LowStock")
}

enum OrderStatus { PLACED ACCEPTED SHIPPED DELIVERED CANCELLED }
```

### `app/GraphQL/Subscriptions/NewOrder.php`

```php
<?php
namespace App\GraphQL\Subscriptions;

use Illuminate\Http\Request;
use Nuwave\Lighthouse\Schema\Types\GraphQLSubscription;
use Nuwave\Lighthouse\Subscriptions\Subscriber;
use GraphQL\Type\Definition\ResolveInfo;
use App\Models\User;
use App\Models\Order;

class NewOrder extends GraphQLSubscription
{
    /**
     * subscribe হওয়ার সময় auth check
     */
    public function authorize(Subscriber $subscriber, Request $request): bool
    {
        $user = $subscriber->context->user();
        $sellerId = (int) $subscriber->args['sellerId'];

        return $user && $user->id === $sellerId && $user->isSeller();
    }

    /**
     * প্রতিটি event-এর জন্য filter
     */
    public function filter(Subscriber $subscriber, $root): bool
    {
        $sellerId = (int) $subscriber->args['sellerId'];
        return $root instanceof Order && $root->seller_id === $sellerId;
    }

    /**
     * subscriber কে যে topic-এ map করবে
     */
    public function decodeTopic(string $fieldName, $root): string
    {
        return "NEW_ORDER_SELLER_{$root->seller_id}";
    }

    public function encodeTopic(Subscriber $subscriber, string $fieldName): string
    {
        return "NEW_ORDER_SELLER_{$subscriber->args['sellerId']}";
    }

    public function resolve($root, array $args, $context, ResolveInfo $info): Order
    {
        return $root;
    }
}
```

### `app/GraphQL/Mutations/UpdateOrderStatus.php`

```php
<?php
namespace App\GraphQL\Mutations;

use App\Models\Order;
use Nuwave\Lighthouse\Subscriptions\Subscriber;
use Nuwave\Lighthouse\Execution\Utils\Subscription;

class UpdateOrderStatus
{
    public function __invoke($_, array $args): Order
    {
        $order = Order::findOrFail($args['id']);
        $order->status = $args['status'];
        $order->save();

        // Broadcast to seller subscribers
        Subscription::broadcast('newOrder', $order);

        return $order;
    }
}
```

### Order creation hook (Eloquent observer)

```php
<?php
namespace App\Observers;

use App\Models\Order;
use Nuwave\Lighthouse\Execution\Utils\Subscription;

class OrderObserver
{
    public function created(Order $order): void
    {
        Subscription::broadcast('newOrder', $order);

        if ($order->items()->lowStockItems()->exists()) {
            foreach ($order->items()->lowStockItems()->get() as $item) {
                Subscription::broadcast('lowStock', $item->product);
            }
        }
    }
}
```

### Queue worker (Redis-backed broadcast)

```bash
php artisan queue:work redis --queue=lighthouse-subscriptions
```

### Client side

JavaScript Pusher / Laravel Echo +  Apollo:

```javascript
import Pusher from 'pusher-js';
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client';
import PusherLink from '@nuwave/pusher-link';

const pusher = new Pusher('PUSHER_KEY', { cluster: 'ap2' });
const pusherLink = new PusherLink({ pusher });
// ... wire similar to Apollo + WS link split
```

---

## 🔐 Authorization

Subscription-এ auth দু'লেয়ারে চেক করুন:

### ১. Connection-level (auth at WS handshake)

```typescript
useServer({
  schema,
  context: async (ctx) => {
    const token = ctx.connectionParams?.authorization;
    const user = await verifyJwt(token);
    if (!user) throw new Error('Unauthorized');     // close WS
    return { user };
  }
});
```

### ২. Per-event filter (defense in depth)

ACL change হলে existing subscription-ও bounce করতে হবে। প্রতিটি event-এ filter check করুন:

```typescript
subscribe: withFilter(
  () => pubsub.asyncIterator([TOPIC]),
  async (payload, vars, ctx) => {
    return await canAccess(ctx.user, payload.orderId);
  }
)
```

### ৩. Topic naming with auth scope

```
ORDER_STATUS:order-123          ← public-ish topic
ORDER_STATUS:user-42:order-123  ← user-scoped topic
```

দ্বিতীয় style-এ accidental cross-user leak কম।

### ৪. Token expiry

Long-lived WS connection-এ token expire হলে কী? Options:
- Disconnect সব → client reconnect with refreshed token (simple, recommended)
- Server-side periodic re-validation (`setInterval`)
- Refresh token via signaling message

---

## 📈 Scaling Subscriptions

### ১. Sticky Sessions vs Stateless via Shared PubSub

| | Sticky Session | Shared PubSub |
|-|----------------|---------------|
| Load balancer | needs sticky | round-robin OK |
| Pod restart | drop user | drop user (reconnect) |
| Cross-pod publish | নেই (single pod) | needed |
| Scale | limited | বিস্তৃত |

**Recommended**: Shared PubSub (Redis/Kafka) + stateless pod + LB reconnect-friendly।

### ২. Connection Limits

```bash
# OS file descriptor limit
ulimit -n 65535

# Node দিয়ে 50K-100K concurrent WS feasible per pod
# তার বেশি হলে → multiple pod
```

`/etc/security/limits.conf`:

```
node soft nofile 65535
node hard nofile 1048576
```

### ৩. Horizontal Scaling Architecture

```
                  ┌───────── HAProxy / Nginx (WSS) ──────────┐
                  │   round-robin or least-conn              │
                  └──┬──────────┬──────────┬─────────────────┘
                     ▼          ▼          ▼
                ┌───────┐  ┌───────┐  ┌───────┐
                │ Pod 1 │  │ Pod 2 │  │ Pod 3 │   (graphql-ws server)
                └──┬────┘  └──┬────┘  └──┬────┘
                   │          │          │
                   └──────────┼──────────┘
                              ▼
                       ┌────────────┐
                       │ Redis /    │
                       │ Kafka      │  (shared PubSub)
                       └────────────┘
```

### ৪. Backpressure

Subscriber slow হলে server queue ফুলে যায়। Strategies:
- Drop oldest events (`MAXLEN` Redis Stream)
- Disconnect slow consumer (graphql-ws `connectionInitWaitTimeout`, custom)
- Coalesce updates (debounce 100ms)

```typescript
// graphql-ws server option
useServer({
  schema,
  connectionInitWaitTimeout: 5_000  // 5s timeout for connection_init
}, wsServer);
```

বিস্তারিত দেখুন: `12-concurrency/backpressure.md`।

### ৫. Monitoring Slow Consumer

```javascript
// per-subscription duration metric (Prometheus)
const subscribeCounter = new prom.Counter({
  name: 'gql_subscribe_total',
  labelNames: ['operation']
});
const slowConsumer = new prom.Counter({
  name: 'gql_slow_consumer_total'
});

ws.on('drain', () => slowConsumer.inc());
```

---

## 🧮 Comparison & Decision Matrix

| | Polling | Long Polling | SSE | WebSocket | GraphQL Subscription (graphql-ws) |
|-|---------|--------------|-----|-----------|------------------------------------|
| Direction | both | server→client | server→client | bi | both (over WS) |
| Latency | ~interval | ~real-time | <100ms | <50ms | <50ms |
| Auto-reconnect | manual | manual | ✅ | manual | ✅ (library) |
| Proxy/firewall | ✅ | ✅ | ✅ | sometimes | sometimes |
| Multiplexed | per-resource | per-resource | one stream | one connection, multi-frame | multiple subscriptions over one WS ✅ |
| Schema-typed | ❌ | ❌ | ❌ | ❌ | ✅ |
| Best for | low-frequency | mobile fallback | one-way streams, notifications | chat, games | typed real-time over GraphQL |

→ Decision matrix: `06-api-design/polling-patterns.md` দেখুন।

---

## ⚡ Live Queries (alternative)

**Live Query**: subscription দিয়ে delta না — পুরো query result auto re-execute হয় যখন underlying data change।

```graphql
query @live {
  order(id: "ord-123") {
    status
    rider { name }
  }
}
```

- **Hasura**: Postgres triggers + multiplex; auto live across schema।
- **Relay**: client-side cache invalidation hint।

Pros: developer simpler — subscription বানাতে হয় না।
Cons: server expensive (re-run query); fine-grained invalidation tricky; not standardized।

---

## 🔭 Operational Concerns

### Idle Connection Limits (Nginx, CloudFront)

Default `proxy_read_timeout 60s` হলে idle WS drop হয়।

```nginx
location /graphql {
    proxy_pass http://gql_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_read_timeout 1d;       # WS-এর জন্য long
    proxy_send_timeout 1d;
}
```

দেখুন: `nginx-deep-dive.md`। CloudFront-এ WSS support আছে কিন্তু idle timeout 10 min — application layer ping send করুন।

### Reconnection / Resume Strategy

graphql-ws default reconnect attempt করে কিন্তু "missed events" automatically replay করে না। Strategy:

```typescript
// client side
const client = createClient({
  url,
  retryAttempts: Infinity,
  shouldRetry: (err) => true,
  on: {
    connected: () => {
      // after reconnect, re-fetch baseline (lastEventId-style)
      apolloClient.refetchQueries({ include: ['GetOrder'] });
    }
  }
});
```

Server-side: persist last N events keyed by connection (Redis Stream)।

### Mobile Network (BD 3G/4G)

- 4G→3G handover: IP change → WS drop → reconnect
- Background tab: browser ping reduce
- App background: iOS terminate WS in 10s; foreground-এ reconnect
- Plan: connection drop → reconnect with exponential backoff + jitter

### Observability

- Per-subscription metrics (count, duration, throughput)
- Slow consumer detection (high queue depth)
- WS connection count per pod
- Reconnect rate alert (anomaly = network/PubSub problem)
- Distributed tracing: subscribe span → publish span → receive span

```typescript
// example Prometheus metrics
const wsConn = new prom.Gauge({ name: 'gql_ws_connections' });
useServer({
  schema,
  onConnect: () => wsConn.inc(),
  onDisconnect: () => wsConn.dec()
}, wsServer);
```

---

## ⚠️ Anti-patterns

❌ **Full DB row payload প্রতিটি update-এ** — bandwidth waste; delta পাঠান।
❌ **Filter ছাড়া fire-hose subscription** (`allMessages`) — সব user সব ইভেন্ট পাবে → ACL leak।
❌ **`onConnect` auth skip** → unauthenticated user subscribe।
❌ **Per-event auth check skip** — token expire/revoke পরের event-এ leak।
❌ **One-shot async kাজে subscription** (e.g., file upload progress) — mutation + polling বা SSE ভালো।
❌ **In-memory PubSub production-এ** → multi-pod fail।
❌ **Subscription-এ heavy DB query প্রতিটি event-এ** → DB melt।
❌ **WebSocket keepalive না** → idle proxy drop।
❌ **Reconnect-এ baseline না-fetch** → stale UI।
❌ **No backpressure** — subscriber slow → server OOM।
❌ **Topic name guessable** (`order:123`) → IDOR via topic enumeration; use random/hash।
❌ **Subscription-এ N+1 query** — DataLoader ব্যবহার করুন।
❌ **Mutation-এ broadcast skip** — subscriber কিছু পাবে না।

---

## 🇧🇩 BD Case Studies

### Daraz — Seller Order Notification

- 50K+ active seller, peak 5K concurrent WS
- Lighthouse + Redis broadcaster
- Sticky session via AWS ALB (or shared Redis route)
- Retention: in-memory 5 min, missed events Redis Stream replay
- Mobile app fallback: FCM push notification

### Foodpanda — Customer Order Tracking

- ~৫ লাখ daily order, peak hour ১০-২০ হাজার active subscription
- Apollo + Kafka backbone (durable, replay)
- Customer app: WebSocket subscription
- Rider app: also WS, different filter
- ETA recompute mutation → broadcast every 30s

### Chaldal — Grocery Delivery ETA

- Fewer concurrent (10-20K) but high-value orders
- Subscription on order; ETA changes when rider GPS update
- Backend: Node + Mercurius + Redis streams
- Bangla SMS fallback if user disconnected (BD 3G reality)

### bKash — Agent Live Transaction Feed

- Compliance-heavy — every event auditable
- Kafka backbone (durable, immutable log)
- Subscription per agent_id with strict ACL
- Per-event re-auth (token expiry strict)
- Insertable encryption layer (sensitive)
- mTLS between gateway and PubSub

---

## ✅ Senior Engineer Subscription Checklist

**Schema:**
- [ ] Subscription field-গুলো filter argument receive করে (orderId, roomId)
- [ ] Payload delta-style, ছোট
- [ ] Initial state Query-এর সাথে দেওয়া আছে
- [ ] Schema versioning policy আছে

**Transport:**
- [ ] `graphql-ws` (NOT legacy `subscriptions-transport-ws`)
- [ ] WSS (TLS) only in production
- [ ] keepAlive ping configured (10-30s)
- [ ] connectionInitWaitTimeout সেট

**PubSub:**
- [ ] In-memory নয় production-এ
- [ ] Redis Pub/Sub (lossy OK), Streams (durable), Kafka (replay), RabbitMQ (ack)
- [ ] Topic naming convention (scope by user/tenant)

**Auth:**
- [ ] Token in `connectionParams`
- [ ] Per-event filter / ACL check
- [ ] Token expiry → reconnect
- [ ] Topic name not guessable

**Scale:**
- [ ] Sticky session NOT required (shared PubSub)
- [ ] FD limits raised (`ulimit -n`)
- [ ] Horizontal scale tested
- [ ] Backpressure strategy আছে

**Ops:**
- [ ] Nginx/CloudFront `proxy_read_timeout` long
- [ ] Reconnect + baseline-refetch strategy
- [ ] Per-subscription metric (Prometheus)
- [ ] Slow consumer alert
- [ ] Distributed tracing wired

**Mobile/BD:**
- [ ] 3G/4G handover reconnect tested
- [ ] App background → foreground reconnect
- [ ] FCM/push fallback for critical event
- [ ] Battery: keepAlive interval sane

---

> **মনে রাখুন**: Subscription একটি **stateful** primitive যা stateless GraphQL-এর উপরে operational complexity যোগ করে। যদি simple notification যথেষ্ট হয় → SSE বেছে নিন (`graphql-sse`)। যদি real-time bi-directional + typed schema দরকার → graphql-ws + Redis Streams / Kafka। বাংলাদেশের context-এ flaky network reality, mobile-first user, এবং cost-conscious infrastructure — এই তিনটে subscription architecture choice-এ বড় factor।
