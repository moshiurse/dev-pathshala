# 🔄 Stateful ও Stateless Replication in Web Applications

> **"Stateless replicate করা সহজ, stateful replicate করা কঠিন — কিন্তু উভয়ই জানা জরুরি।"**

---

## 📌 সংজ্ঞা

Web application replication হলো একটি application-এর একাধিক instance চালানো যাতে:
- **High Availability**: একটি instance মারা গেলেও সার্ভিস চালু থাকে
- **Scalability**: বেশি traffic handle করতে পারে
- **Fault Tolerance**: failure-তেও সিস্টেম কাজ করে

কিন্তু application-টি **stateful** নাকি **stateless** — এটাই নির্ধারণ করে replication কতটা সহজ বা জটিল হবে।

---

## 📊 Stateful vs Stateless — মূল পার্থক্য

```
┌──────────────────────────────────────────────────────────────┐
│                STATEFUL SERVER                                 │
│                                                               │
│  Server A ─── user session, cart, temp data memory-তে         │
│  Server B ─── অন্য user-এর data                              │
│                                                               │
│  সমস্যা:                                                      │
│    User যদি Server A তে login করে, পরের request               │
│    Server B তে গেলে session পাবে না!                          │
│    → Sticky session লাগবে (load balancer constraint)          │
│    → Server A মারা গেলে session হারাবে                        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                STATELESS SERVER                                │
│                                                               │
│  Server A ─── কোনো local state নেই                            │
│  Server B ─── কোনো local state নেই                            │
│  Server C ─── কোনো local state নেই                            │
│                                                               │
│  Session/State → External store (Redis, DB)                   │
│                                                               │
│  যেকোনো request, যেকোনো server handle করতে পারে ✅            │
└──────────────────────────────────────────────────────────────┘
```

| বৈশিষ্ট্য | Stateful | Stateless |
|-----------|----------|-----------|
| **State location** | Server memory | External store |
| **Scaling** | কঠিন (state migrate করতে হয়) | সহজ (add/remove server) |
| **Load balancing** | Sticky session লাগে | Round-robin চলে |
| **Failover** | State হারায় | State safe (external) |
| **Consistency** | Server-local (strong) | External store-এ depend করে |
| **Complexity** | Application সহজ, ops কঠিন | Application একটু complex, ops সহজ |

---

## 🏗️ Stateless Replication in Web Applications

### কনসেপ্ট

Stateless replication-এ application server কোনো request-specific state ধরে রাখে না। প্রতিটি request self-contained — all necessary information (token, session ID) request-এর সাথেই আসে।

```
┌────────────────────────────────────────────────────────────┐
│              Stateless Replication Architecture              │
│                                                             │
│                     ┌─────────────┐                         │
│   Client ──────────►│   Load      │                         │
│   (JWT token /      │  Balancer   │                         │
│    session cookie)  │ (Round Robin)│                         │
│                     └──────┬──────┘                         │
│                            │                                │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                   │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│        │ Server 1 │ │ Server 2 │ │ Server 3 │              │
│        │ (no state)│ │(no state)│ │(no state)│              │
│        └─────┬────┘ └─────┬────┘ └─────┬────┘              │
│              │             │             │                   │
│              └─────────────┼─────────────┘                  │
│                            ▼                                │
│              ┌──────────────────────────┐                   │
│              │   Shared State Store     │                   │
│              │  (Redis / Database)      │                   │
│              └──────────────────────────┘                   │
└────────────────────────────────────────────────────────────┘
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|--------|----------|
| **Easy horizontal scaling** | নতুন server add করলেই capacity বাড়ে |
| **Simple load balancing** | Round-robin, random — কোনো constraint নেই |
| **Zero-downtime deployment** | Rolling update — server এক এক করে replace |
| **Fault tolerant** | Server মরলে অন্যরা নির্বিঘ্নে চালিয়ে যায় |
| **Auto-scaling friendly** | Traffic spike-এ auto scale-up, কমলে scale-down |

### State Externalization কৌশল

#### 1. JWT (JSON Web Token) — Client-side state

```
┌──────────────────────────────────────────────────────────┐
│  Client                    Server                         │
│                                                           │
│  Login request ──────────────►                            │
│                              ◄── JWT token (signed)       │
│  Subsequent requests:                                     │
│  Authorization: Bearer <JWT> ──────────────►              │
│                              Server verifies signature    │
│                              No DB lookup needed!         │
└──────────────────────────────────────────────────────────┘
```

#### 2. Redis Session Store — Server-side state externalized

```
┌──────────────────────────────────────────────────────────┐
│  request + session_id cookie                              │
│       │                                                   │
│       ▼                                                   │
│  Any App Server → Redis GET session:{id} → user data     │
│                                                           │
│  Session create: Redis SET session:{id} {data} EX 3600   │
│  Session read:   Redis GET session:{id}                   │
│  Session destroy: Redis DEL session:{id}                  │
└──────────────────────────────────────────────────────────┘
```

### PHP Code Example — Stateless (Laravel)

```php
<?php
// config/session.php — Redis-backed session (externalized state)
return [
    'driver' => env('SESSION_DRIVER', 'redis'),
    'connection' => 'session',
    'lifetime' => 120,
];

// JWT-based API (fully stateless)
// app/Http/Middleware/JwtAuth.php
class JwtAuth
{
    public function handle(Request $request, Closure $next)
    {
        $token = $request->bearerToken();
        if (!$token) {
            return response()->json(['error' => 'Token missing'], 401);
        }

        try {
            // Token-এ সব user info আছে — DB lookup দরকার নেই
            $payload = JWT::decode($token, new Key(config('jwt.secret'), 'HS256'));
            $request->merge(['auth_user' => (array) $payload]);
        } catch (\Exception $e) {
            return response()->json(['error' => 'Invalid token'], 401);
        }

        return $next($request);
    }
}

// Controller — stateless, server কোনো state ধরে রাখে না
class OrderController extends Controller
{
    public function store(Request $request)
    {
        $userId = $request->input('auth_user.sub'); // JWT থেকে

        $order = Order::create([
            'user_id' => $userId,
            'items' => $request->input('items'),
            'total' => $this->calculateTotal($request->input('items')),
        ]);

        // Async processing — state queue-তে
        ProcessOrderJob::dispatch($order->id);

        return response()->json($order, 201);
    }
}
```

### JavaScript Code Example — Stateless (Express)

```javascript
import express from 'express';
import jwt from 'jsonwebtoken';
import Redis from 'ioredis';

const app = express();
const redis = new Redis(process.env.REDIS_URL);

// Stateless JWT middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// External cache — server instance-এ কোনো state নেই
app.get('/api/products/:id', authenticate, async (req, res) => {
  const cacheKey = `product:${req.params.id}`;
  const cached = await redis.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  const product = await db.products.findById(req.params.id);
  await redis.setex(cacheKey, 3600, JSON.stringify(product));
  res.json(product);
});

// যেকোনো server instance এই request handle করতে পারে
// Server 1 crash করলে Load Balancer পরের request Server 2/3 এ পাঠাবে
```

---

## 🏗️ Stateful Replication in Web Applications

### কনসেপ্ট

কিছু ক্ষেত্রে server-এ state রাখা অনিবার্য — real-time connections (WebSocket), in-memory game state, large computation cache, ML model inference। এসব ক্ষেত্রে stateful replication দরকার।

```
┌────────────────────────────────────────────────────────────┐
│              Stateful Replication Architecture               │
│                                                             │
│                     ┌─────────────┐                         │
│   Client ──────────►│   Load      │                         │
│   (session affinity │  Balancer   │                         │
│    / sticky session)│(IP/Cookie   │                         │
│                     │  hash)      │                         │
│                     └──────┬──────┘                         │
│                            │                                │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                   │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│        │ Server 1 │ │ Server 2 │ │ Server 3 │              │
│        │ State:   │ │ State:   │ │ State:   │              │
│        │ User A,B │ │ User C,D │ │ User E,F │              │
│        │ (local)  │ │ (local)  │ │ (local)  │              │
│        └──────────┘ └──────────┘ └──────────┘              │
│                                                             │
│   ⚠️ Server 2 crash → User C,D-এর state হারাবে!           │
│   Solution: state replication / checkpointing              │
└────────────────────────────────────────────────────────────┘
```

### কখন Stateful Replication দরকার?

| Use Case | কেন Stateful? |
|----------|---------------|
| **WebSocket connections** | Connection server-specific, client reconnect costly |
| **Real-time gaming** | Game state in-memory, latency critical |
| **Streaming sessions** | Video/audio stream state |
| **Long-running computations** | Intermediate results memory-তে |
| **In-memory cache (local)** | Hot data locality, cross-server overhead |
| **ML model serving** | Model memory-তে load — expensive to reload |

### Stateful Replication কৌশল

#### 1. Sticky Sessions (Session Affinity)

```
┌──────────────────────────────────────────────────────────┐
│  Load Balancer Configuration:                             │
│                                                           │
│  NGINX (ip_hash):                                        │
│    upstream backend {                                     │
│      ip_hash;                                            │
│      server app1:3000;                                   │
│      server app2:3000;                                   │
│      server app3:3000;                                   │
│    }                                                     │
│                                                           │
│  একই IP → সবসময় একই server                               │
│  সমস্যা: server down → session lost                       │
│  সমস্যা: uneven distribution                             │
└──────────────────────────────────────────────────────────┘
```

#### 2. State Replication (Active-Active)

```
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Server 1 ◄────── state sync ──────► Server 2            │
│  State: {A,B,C}                       State: {A,B,C}     │
│                                                           │
│  যেকোনো server handle করতে পারে                           │
│  কিন্তু: sync overhead, consistency issue                 │
│  N nodes = N×(N-1) sync messages!                        │
└──────────────────────────────────────────────────────────┘
```

#### 3. State Checkpointing

```
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Server 1 (active)                                       │
│    State: {WebSocket connections, game state}             │
│         │                                                 │
│         │ periodic checkpoint (every 5s)                  │
│         ▼                                                 │
│  ┌──────────────────┐                                    │
│  │  Redis / Storage  │  ← state snapshot                 │
│  └──────────────────┘                                    │
│         │                                                 │
│         │ on failure, new server loads checkpoint         │
│         ▼                                                 │
│  Server 4 (takes over)                                   │
│    State: restored from checkpoint (≤5s stale)           │
└──────────────────────────────────────────────────────────┘
```

#### 4. Consistent Hashing (Stateful Routing)

```
┌──────────────────────────────────────────────────────────┐
│  Hash Ring:                                               │
│                                                           │
│       Server A                                            │
│      ╱         ╲                                          │
│  User1,2        User5,6                                  │
│    │                │                                     │
│  Server C        Server B                                │
│      ╲         ╱                                          │
│       User3,4                                             │
│                                                           │
│  Server B removed → User5,6 goes to next (Server A)     │
│  Minimal disruption — শুধু affected users re-route হয়   │
└──────────────────────────────────────────────────────────┘
```

### PHP Code Example — Stateful WebSocket with Replication

```php
<?php
// Ratchet WebSocket server with Redis pub/sub for cross-instance messaging

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;
use Predis\Client as Redis;

class ChatServer implements MessageComponentInterface
{
    protected \SplObjectStorage $clients;
    protected array $rooms = [];       // Local state!
    protected array $userConnections = []; // Local state!
    protected Redis $redis;
    protected string $serverId;

    public function __construct()
    {
        $this->clients = new \SplObjectStorage();
        $this->redis = new Redis(getenv('REDIS_URL'));
        $this->serverId = gethostname();

        // Subscribe to cross-server messages
        $this->subscribeToRedisChannel();
    }

    public function onOpen(ConnectionInterface $conn)
    {
        $this->clients->attach($conn);
        $userId = $this->extractUserId($conn);

        // Local state — এই user এই server-এ connected
        $this->userConnections[$userId] = $conn;

        // Redis-এ register — কোন user কোন server-এ
        $this->redis->hset('user_locations', $userId, $this->serverId);

        // Restore user state from checkpoint
        $state = $this->redis->get("user_state:{$userId}");
        if ($state) {
            $conn->send(json_encode(['type' => 'state_restore', 'data' => json_decode($state)]));
        }
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        $data = json_decode($msg, true);
        $userId = $this->extractUserId($from);

        if ($data['type'] === 'chat') {
            $targetUser = $data['to'];
            $targetServer = $this->redis->hget('user_locations', $targetUser);

            if ($targetServer === $this->serverId) {
                // Same server — deliver locally
                $this->userConnections[$targetUser]?->send($msg);
            } else {
                // Different server — publish to Redis channel
                $this->redis->publish("server:{$targetServer}", $msg);
            }
        }

        // Periodic state checkpoint
        $this->checkpointUserState($userId);
    }

    public function onClose(ConnectionInterface $conn)
    {
        $userId = $this->extractUserId($conn);
        $this->checkpointUserState($userId); // Save final state
        unset($this->userConnections[$userId]);
        $this->redis->hdel('user_locations', $userId);
        $this->clients->detach($conn);
    }

    protected function checkpointUserState(string $userId): void
    {
        $state = [
            'rooms' => $this->getUserRooms($userId),
            'last_active' => time(),
            'unread' => $this->getUnreadCount($userId),
        ];
        // 1-hour TTL checkpoint
        $this->redis->setex("user_state:{$userId}", 3600, json_encode($state));
    }

    protected function subscribeToRedisChannel(): void
    {
        // Separate connection for pub/sub
        $sub = new Redis(getenv('REDIS_URL'));
        $sub->subscribe(["server:{$this->serverId}"], function ($message) {
            $data = json_decode($message, true);
            $targetUser = $data['to'];
            $this->userConnections[$targetUser]?->send($message);
        });
    }
}
```

### JavaScript Code Example — Stateful with Cluster Coordination

```javascript
import { Server } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { createServer } from 'http';

const httpServer = createServer();
const io = new Server(httpServer, { cors: { origin: '*' } });

// Redis adapter — cross-server event delivery
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Local state (per server instance)
const localGameState = new Map(); // userId -> game state
const SERVER_ID = process.env.HOSTNAME || `server-${process.pid}`;

io.on('connection', (socket) => {
  const userId = socket.handshake.auth.userId;

  // Register user location
  pubClient.hSet('user_servers', userId, SERVER_ID);

  // Restore state from checkpoint (if reconnection)
  restoreUserState(userId).then((state) => {
    if (state) {
      localGameState.set(userId, state);
      socket.emit('state:restore', state);
    }
  });

  socket.on('game:action', (action) => {
    // Update local state
    const state = localGameState.get(userId) || initGameState();
    applyAction(state, action);
    localGameState.set(userId, state);

    // Broadcast to room (works cross-server via Redis adapter)
    socket.to(state.roomId).emit('game:update', { userId, action, state });

    // Periodic checkpoint
    checkpointState(userId, state);
  });

  socket.on('disconnect', async () => {
    // Checkpoint on disconnect
    const state = localGameState.get(userId);
    if (state) {
      await pubClient.setEx(`user_state:${userId}`, 3600, JSON.stringify(state));
    }
    localGameState.delete(userId);
    await pubClient.hDel('user_servers', userId);
  });
});

async function restoreUserState(userId) {
  const raw = await pubClient.get(`user_state:${userId}`);
  return raw ? JSON.parse(raw) : null;
}

async function checkpointState(userId, state) {
  // Checkpoint every 10 seconds (debounced)
  await pubClient.setEx(`user_state:${userId}`, 3600, JSON.stringify(state));
}

httpServer.listen(3000, () => console.log(`${SERVER_ID} listening on :3000`));
```

---

## 🔄 Hybrid Approach — Best Practice

বেশিরভাগ production system **hybrid** approach ব্যবহার করে:

```
┌────────────────────────────────────────────────────────────┐
│              Hybrid: Stateless + Stateful                    │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │  Stateless Layer (REST API)      │ ← ×20 instances       │
│  │  - Product listing               │    Round-robin LB     │
│  │  - Order placement               │    Auto-scale         │
│  │  - User profile                  │                       │
│  └──────────────────────────────────┘                       │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │  Stateful Layer (WebSocket)      │ ← ×5 instances        │
│  │  - Real-time chat                │    Sticky session     │
│  │  - Live notifications            │    Redis pub/sub      │
│  │  - Presence/typing indicator     │    Checkpointing      │
│  └──────────────────────────────────┘                       │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │  Shared State (External)         │                       │
│  │  - Redis (sessions, cache)       │                       │
│  │  - PostgreSQL (persistent data)  │                       │
│  │  - Kafka (events)                │                       │
│  └──────────────────────────────────┘                       │
└────────────────────────────────────────────────────────────┘
```

---

## ✅ Decision Matrix

| Factor | Stateless ব্যবহার করুন | Stateful ব্যবহার করুন |
|--------|------------------------|----------------------|
| Request-response API | ✅ | ❌ |
| WebSocket/real-time | ❌ | ✅ |
| Simple CRUD | ✅ | ❌ |
| Gaming/collaboration | ❌ | ✅ |
| Auto-scaling priority | ✅ | ❌ |
| Low latency (local state) | ❌ | ✅ |
| Easy ops/deployment | ✅ | ❌ |
| ML model serving | ❌ | ✅ |

---

## 🔑 মনে রাখার বিষয়

1. **Default Stateless**: সম্ভব হলে সবসময় stateless architecture ব্যবহার করুন
2. **Externalize State**: Session, cache, state → Redis/DB-তে রাখুন
3. **Stateful ≠ Bad**: কিছু use case-এ stateful অনিবার্য (WebSocket, gaming)
4. **Checkpoint Always**: Stateful server-এ periodic state checkpointing করুন
5. **Hybrid is Real**: Production system প্রায়ই stateless API + stateful WebSocket মিশ্রণ
6. **Redis Pub/Sub**: Cross-server messaging-এর জন্য standard solution
7. **Consistent Hashing**: Stateful routing-এর জন্য — node change-এ minimal disruption
