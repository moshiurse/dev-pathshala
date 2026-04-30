# 🚪 API Gateway — মাইক্রোসার্ভিসের Edge Layer

## 📖 সংজ্ঞা ও মূল ধারণা

**API Gateway** হলো একটি বিশেষ ধরনের server যা client (Web/Mobile/Partner) এবং একগুচ্ছ backend microservices-এর মাঝখানে **single entry point** হিসেবে কাজ করে। এটি OSI মডেলের L7 (application layer)-এ অবস্থান নিয়ে routing, authentication, rate limiting, transformation, aggregation, observability — এসব cross-cutting concern এক জায়গায় handle করে।

**Chris Richardson-এর সংজ্ঞা (Microservices Patterns):**
> "An API Gateway is a service that's the entry point into the application from the outside world. It's responsible for request routing, API composition, and other edge functions such as authentication."

```
ছাড়া API Gateway:
    Mobile App ──► Auth Service
              ──► Product Service        ৭টা আলাদা TCP connection
              ──► Cart Service           ৭ বার auth check
              ──► Order Service          client এ অনেক logic
              ──► Payment Service        CORS/TLS ৭ জায়গায়
              ──► Notification Svc
              ──► Search Service

API Gateway সহ:
                         ┌────────────────────────┐
                         │     API GATEWAY        │
    Mobile App ────►     │  - TLS termination     │
    Web App    ────►     │  - JWT verify          │  ┌► Product Svc
    Partner    ────►     │  - Rate limit          │──┼► Cart Svc
    IoT Device ────►     │  - Routing             │  ├► Order Svc
                         │  - Aggregation         │  ├► Payment Svc
                         │  - Caching             │  └► Search Svc
                         │  - Tracing/Logging     │
                         └────────────────────────┘
```

---

## 🎯 কেন API Gateway? — সমস্যা যা সমাধান করে

### সমস্যা ১: Client-side complexity
১০টা microservice থাকলে mobile app-কে ১০টা hostname/port জানতে হবে, প্রতিটার জন্য আলাদা auth, retry, circuit breaker logic বসাতে হবে। App update ছাড়া service URL পরিবর্তন করা যাবে না।

### সমস্যা ২: Cross-cutting concerns duplication
JWT validation, rate limiting, CORS, logging — প্রতিটা service-এ আলাদা ভাবে implement করলে inconsistent হয় এবং DRY ভঙ্গ হয়।

### সমস্যা ৩: Protocol heterogeneity
Internal-এ gRPC ব্যবহার করলে দ্রুত হয়, কিন্তু browser থেকে gRPC সরাসরি call করা কঠিন। Gateway REST↔gRPC translation করে।

### সমস্যা ৪: Chatty client
Daraz home page লোড করতে দরকার: banner, recommended products, cart count, notifications, user profile — ৫টা আলাদা call। 4G নেটওয়ার্কে এটা slow। Gateway একটাই `/home` endpoint-এ aggregate করে দেয়।

### সমস্যা ৫: Security perimeter
Internal services public internet-এ expose করা risky। Gateway-ই একমাত্র public endpoint, বাকি সব private VPC subnet-এ লুকানো থাকে।

### সমস্যা ৬: Operational visibility
সব traffic একটা layer দিয়ে গেলে centralized metrics, tracing, audit log পাওয়া সহজ।

---

## 🧭 API Gateway vs BFF — পার্থক্য

`bff-pattern.md`-এ আমরা BFF নিয়ে আলোচনা করেছি। দুটো প্যাটার্ন related কিন্তু different:

| বৈশিষ্ট্য | API Gateway | BFF |
|----------|-------------|-----|
| উদ্দেশ্য | Generic edge concerns (auth, rate limit, routing) | Client-specific data shaping |
| সংখ্যা | সাধারণত ১টা cluster | প্রতি client type এ ১টা (Web/Mobile/TV) |
| Owner | Platform/Infra team | Frontend/Product team |
| Logic | Thin, configuration-driven | Business orchestration, transformation |
| Customization | Plugin/policy | Full custom code |
| Sits where | Edge (public-facing) | পিছনে — Gateway এর behind অথবা Gateway-ই hold করে |
| Example | Kong, AWS API Gateway | Node.js BFF for Daraz Mobile |

**সাধারণ pipeline:**
```
Client ──► API Gateway (auth, rate limit) ──► BFF (aggregation) ──► Microservices
```

কিছু সংস্থা দুটো একসাথে merge করে — Gateway-ই BFF-এর কাজও করে। ছোট স্কেলে এটা ঠিক আছে, কিন্তু বড় হলে আলাদা করা ভালো।

---

## 🔧 Core Responsibilities (গভীর ভাবে)

### ১. Routing (Path/Host/Header-based)
- `/api/products/*` → product-service:8080
- `Host: partner.daraz.com` → partner-service
- `X-API-Version: v2` → v2 cluster (canary)

### ২. Authentication & Authorization
- **JWT validation** (signature + exp + iss + aud check) using JWKS cache
- **OAuth2 token introspection** (RFC 7662) — opaque token কে authorization server-এ যাচাই
- **API key** (header/query) — partner integration
- **mTLS** — high-trust partner (bKash merchant API)
- **Authorization** — RBAC/ABAC, scope check, route-level policy

### ৩. Rate Limiting / Throttling / Quota
- Per-IP, per-user, per-API-key
- Algorithms: token bucket, sliding window (দেখুন `rate-limiting.md`)
- Distributed counter: Redis/Memcached
- Quota: প্রতি partner-এর monthly limit

### ৪. Request/Response Transformation
- Header rewrite (`X-Forwarded-User: <jwt.sub>`)
- Body transformation (snake_case → camelCase)
- Strip internal headers from response
- Field projection (mobile: কম field পাঠাও)

### ৫. Protocol Translation
- REST → gRPC (Envoy gRPC-JSON transcoder)
- REST → GraphQL (rare, usually opposite)
- HTTP/1.1 → HTTP/2 (upstream)
- WebSocket/SSE proxying with sticky session

### ৬. Aggregation / Composition
একটি request → multiple downstream → একটি response। Promise.allSettled দিয়ে partial failure tolerate করা যায়।

### ৭. Caching
- Response cache (GET, public, max-age) — Daraz product detail ৬০ সেকেন্ড cache
- Negative cache (404 cached short-time)
- ETag/If-None-Match support

### ৮. Observability
- Access log (JSON, structured)
- Prometheus metrics (`http_requests_total`, latency histogram)
- OpenTelemetry trace propagation (`traceparent`, `tracestate`)
- Audit log for sensitive endpoints

### ৯. Resilience
- **Timeouts** (connect, read, idle, total)
- **Retries** with exponential backoff + jitter (idempotent only)
- **Circuit breaker** (5xx threshold → trip → fallback)
- **Request hedging** — slow upstream → second parallel call
- **Bulkhead** — connection pool per upstream

### ১০. Edge Concerns
- TLS termination (and re-encryption / mTLS to upstream)
- HTTP/2 / HTTP/3 (QUIC)
- Body size limit (DoS protection)
- CORS handling (`Access-Control-*`)
- WAF integration (ModSecurity, AWS WAF)
- Schema validation (OpenAPI request/response check)

### ১১. Traffic Management
- **Canary** — 5% traffic → v2
- **Blue-Green** — switch label
- **A/B testing** — header/cookie based routing
- **Shadow traffic** — copy production → staging silently

---

## 🏗️ Architectural Patterns

### Edge Gateway (North-South traffic)
Public internet ↔ datacenter সীমানায় বসে। Daraz-এর `api.daraz.com.bd` যেই Kong cluster-এ resolve হয়, সেটাই edge gateway।

### Internal Gateway (East-West)
Service mesh-এর alternative। Internal microservices-এর মাঝে routing/auth হলে। তবে `service-mesh.md`-এ দেখানো mesh এই কাজটা ভালো করে — Gateway সাধারণত edge-এ থাকে।

### Gateway-per-Client (BFF overlap)
আলাদা gateway: `mobile-gw.daraz.com.bd`, `web-gw.daraz.com.bd`, `partner-gw.daraz.com.bd`। প্রতিটায় আলাদা rate limit, auth policy।

### Aggregator Gateway
শুধু aggregation/orchestration-এ focused। GraphQL gateway একটা উদাহরণ।

### Embedded/Sidecar Gateway
প্রতিটা service-এর পাশে ছোট gateway (Envoy sidecar)। Service mesh data plane এই pattern follow করে।

```
              ┌─────────── Edge Gateway ────────────┐
              │  TLS, WAF, global rate limit, JWT   │
              └──────────────┬──────────────────────┘
                             │
              ┌──────────────▼──────────────────────┐
              │   Internal Mesh (Istio/Linkerd)     │
              │  mTLS, service-to-service routing   │
              └──────────────────────────────────────┘
```

---

## 🆚 API Gateway vs Service Mesh vs Load Balancer vs Reverse Proxy

| বৈশিষ্ট্য | API Gateway | Service Mesh | Load Balancer (L4) | Reverse Proxy (L7) |
|----------|-------------|--------------|---------------------|--------------------|
| OSI Layer | L7 | L7 (sidecar) | L4 (TCP/UDP) | L7 |
| Traffic | North-South | East-West | উভয় | উভয় |
| Auth/Rate limit | ✅ rich | ✅ basic | ❌ | কিছুটা |
| Service discovery | ✅ | ✅ | নির্ভর করে | manual |
| mTLS | ✅ (edge to upstream) | ✅ (auto, internal) | ❌ | manual |
| Aggregation | ✅ | ❌ | ❌ | ❌ |
| Plugin ecosystem | rich | medium | minimal | medium |
| Examples | Kong, AWS APIGW | Istio, Linkerd | AWS NLB, HAProxy L4 | Nginx, Apache, Traefik |
| Tradeoff | feature-rich, heavier | per-pod overhead | দ্রুত, dumb | middle ground |

**মনে রাখুন:** এদের overlap আছে। Production-এ সাধারণত: NLB (L4) → API Gateway → Service Mesh → Pod।

---

## 🛠️ Popular Implementations Comparison

| Tool | Type | Open Source | Cloud-Managed | Plugins | Performance | Config | Learning Curve |
|------|------|-------------|---------------|---------|-------------|--------|----------------|
| **Kong** | Nginx+Lua | ✅ | Konnect | ✅ rich (Lua) | উচ্চ | declarative YAML/DB | মাঝারি |
| **Tyk** | Go | ✅ | Tyk Cloud | ✅ JS/Go | উচ্চ | YAML/Dashboard | সহজ |
| **KrakenD** | Go | ✅ | ❌ | মাঝারি (Go plugins) | অতি দ্রুত | JSON, declarative | সহজ |
| **AWS API Gateway** | Managed | ❌ | ✅ AWS | Lambda authorizers | উচ্চ | Console/IaC | সহজ |
| **Azure API Management** | Managed | ❌ | ✅ Azure | Policy XML | উচ্চ | Portal/Bicep | মাঝারি |
| **Apigee** | Managed | ❌ | ✅ GCP | rich (JS, Java) | উচ্চ | XML proxy | কঠিন (enterprise) |
| **Ambassador (Emissary)** | Envoy | ✅ | ❌ | Envoy filters | অতি দ্রুত | K8s CRD | মাঝারি |
| **Gloo Edge** | Envoy | ✅ | Solo.io Cloud | rich | অতি দ্রুত | K8s CRD | মাঝারি |
| **Traefik** | Go | ✅ | Traefik Hub | মাঝারি | উচ্চ | auto-discovery | সহজ |
| **HAProxy** | C | ✅ | HAProxy ALOHA | Lua | অতি দ্রুত | haproxy.cfg | মাঝারি |
| **Nginx (as gateway)** | C+Lua/njs | ✅ | Nginx Plus | Lua/njs | অতি দ্রুত | nginx.conf | মাঝারি |
| **Istio Ingress Gateway** | Envoy | ✅ | বিভিন্ন | Envoy filters | অতি দ্রুত | K8s CRD | কঠিন |
| **Spring Cloud Gateway** | JVM | ✅ | ❌ | Java filters | মাঝারি | YAML/Java | মাঝারি (Java team) |
| **Express Gateway** | Node.js | ✅ | ❌ | JS | মাঝারি | YAML | সহজ |
| **Zuul (Netflix)** | JVM | ✅ (legacy) | ❌ | Groovy filters | মাঝারি | code | মাঝারি |
| **Ocelot** | .NET | ✅ | ❌ | C# middleware | মাঝারি | JSON | সহজ (C# team) |
| **Caddy** | Go | ✅ | ❌ | Go modules | উচ্চ | Caddyfile | সহজ |

**বাংলাদেশের context-এ পছন্দ:**
- ছোট startup, AWS Mumbai-এ deploy → AWS API Gateway (managed, কম DevOps)
- মাঝারি স্কেল (Daraz/Pathao) → Kong on EKS (control আছে, plugin-rich)
- বড় K8s heavy (Grameenphone digital platform) → Istio Ingress + Envoy

---

## 🦍 Deep Dive: Kong

**Kong** = Nginx + OpenResty (Lua) + DB (Postgres/Cassandra) বা DB-less mode। Plugin architecture-এ সব কাজ হয়।

### Modes
1. **Traditional (DB-backed):** Postgres/Cassandra-এ config; Admin API দিয়ে CRUD।
2. **DB-less (declarative):** YAML file + `KONG_DECLARATIVE_CONFIG`। GitOps-friendly।
3. **Hybrid:** Control plane (DB) + Data plane (config push via mTLS)। Multi-region scaling।

### Important Plugins
- `jwt`, `key-auth`, `oauth2`, `basic-auth`, `ldap-auth`
- `rate-limiting`, `request-size-limiting`, `ip-restriction`
- `cors`, `request-transformer`, `response-transformer`
- `prometheus`, `zipkin`, `opentelemetry`, `file-log`, `http-log`
- `proxy-cache`, `request-termination`
- কাস্টম Lua plugin লেখা যায়।

### Daraz-like Setup — declarative `kong.yaml`
```yaml
_format_version: "3.0"
_transform: true

services:
  - name: product-service
    url: http://product-svc.internal:8080
    connect_timeout: 2000
    read_timeout: 5000
    retries: 2
    routes:
      - name: products-route
        paths: ["/api/v1/products"]
        strip_path: false
        methods: [GET, POST]
    plugins:
      - name: rate-limiting
        config:
          minute: 600
          policy: redis
          redis_host: redis.internal
          redis_port: 6379
      - name: proxy-cache
        config:
          response_code: [200]
          request_method: [GET]
          content_type: ["application/json"]
          cache_ttl: 60
          strategy: memory

  - name: order-service
    url: http://order-svc.internal:8080
    routes:
      - name: orders-route
        paths: ["/api/v1/orders"]
    plugins:
      - name: jwt
        config:
          key_claim_name: kid
          claims_to_verify: [exp]
      - name: rate-limiting
        config:
          minute: 120
          policy: redis

  - name: payment-service
    url: https://payment-svc.internal:8443
    tls_verify: true
    client_certificate:
      id: payment-mtls-cert
    routes:
      - name: payments-route
        paths: ["/api/v1/payments"]
    plugins:
      - name: jwt
      - name: request-size-limiting
        config:
          allowed_payload_size: 1   # MB
      - name: opentelemetry
        config:
          endpoint: http://otel-collector:4318/v1/traces

consumers:
  - username: daraz-mobile-app
    jwt_secrets:
      - key: "mobile-key-id-2024"
        algorithm: RS256
        rsa_public_key: |
          -----BEGIN PUBLIC KEY-----
          ...
          -----END PUBLIC KEY-----

plugins:
  - name: prometheus
  - name: cors
    config:
      origins: ["https://www.daraz.com.bd", "https://m.daraz.com.bd"]
      methods: [GET, POST, PUT, DELETE, OPTIONS]
      headers: [Authorization, Content-Type, X-Request-Id]
      credentials: true
      max_age: 3600
```

`kong start -c kong.conf --vv` দিয়ে চালাতে পারবেন (DB-less)।

### Custom Lua Plugin (skeleton)
```lua
-- kong/plugins/bd-mobile-validator/handler.lua
local plugin = {
  PRIORITY = 1000,
  VERSION = "1.0.0",
}

function plugin:access(conf)
  local mobile = kong.request.get_header("X-User-Mobile")
  if mobile and not string.match(mobile, "^01[3-9]%d%d%d%d%d%d%d%d$") {
    return kong.response.exit(400, { error = "Invalid BD mobile number" })
  end
end

return plugin
```

---

## 🚀 Deep Dive: Envoy / Ambassador

**Envoy** = high-performance L7 proxy (Lyft); Istio, Ambassador, Gloo, Consul Connect — সব Envoy-ই use করে।

### Core Concepts
- **Listeners** — যেই port-এ shun (e.g., 0.0.0.0:443)
- **Filter chains** — listener-এর request-এ apply হয় (HTTP connection manager, JWT auth, rate limit)
- **Routes** — path/host match → cluster
- **Clusters** — upstream service group (load balancing policy, health check)
- **Endpoints** — actual pod IP

### xDS API
Envoy dynamically config নেয় control plane থেকে (LDS, RDS, CDS, EDS)। Istio Pilot এই control plane।

### Sample EnvoyFilter (Istio) — JWT validation + rate limit
```yaml
apiVersion: networking.istio.io/v1beta1
kind: EnvoyFilter
metadata:
  name: jwt-and-ratelimit
  namespace: istio-system
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.jwt_authn
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
          providers:
            daraz_auth:
              issuer: "https://auth.daraz.com.bd"
              audiences: ["daraz-api"]
              remote_jwks:
                http_uri:
                  uri: "https://auth.daraz.com.bd/.well-known/jwks.json"
                  cluster: jwks_cluster
                  timeout: 5s
                cache_duration: { seconds: 300 }
          rules:
          - match: { prefix: "/api/v1/orders" }
            requires: { provider_name: "daraz_auth" }
```

### Upstream mTLS
```yaml
clusters:
- name: payment_cluster
  type: STRICT_DNS
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      common_tls_context:
        tls_certificates:
        - certificate_chain: { filename: "/etc/certs/client.crt" }
          private_key:       { filename: "/etc/certs/client.key" }
        validation_context:
          trusted_ca: { filename: "/etc/certs/ca.crt" }
```

---

## ☁️ Deep Dive: AWS API Gateway

| Type | Use Case | Latency | Pricing | Notes |
|------|----------|---------|---------|-------|
| **REST API** | Full feature, request validation, usage plan | বেশি (~30ms) | বেশি | API key, throttling, caching, WAF |
| **HTTP API** | Modern, কম feature, কম দাম | কম (~10ms) | ~৭০% সস্তা | JWT authorizer built-in |
| **WebSocket API** | Realtime (chat, live tracking) | কম | per-message | Pathao live ride tracking |

### Authorizers
- **Lambda Authorizer (Token/Request)** — custom logic; bKash signature verify।
- **Cognito Authorizer** — built-in JWT verify।
- **JWT Authorizer (HTTP API only)** — JWKS-based, কোন code নয়।
- **IAM Authorizer** — internal AWS account থেকে SigV4 call।

### Usage Plans & API Keys
Partner integration এর জন্য:
- API key per partner
- Usage plan: 10000 req/day, 100 req/sec burst
- Different stage (dev/prod) এ আলাদা limit

### VPC Link
Private ALB/NLB-এ থাকা microservice-এ access — Gateway public, backend private।

### বাংলাদেশের startup-এ cost consideration
- HTTP API: $1/million req (Mumbai region) — চমৎকার
- REST API: $3.5/million + cache cost
- ১ million DAU-এর Pathao-mini app, ২০ req/user/day = ৬০০ million/month → HTTP API ~$৬০০, REST ~$২,১০০
- নিজেই Kong on EKS চালালে EC2 + EKS cost (~$৩০০-৫০০/month) অনেক কম, কিন্তু DevOps overhead বেশি।

---

## 💻 Implementation: Minimal API Gateway in Node.js (Fastify)

### `package.json`
```json
{
  "name": "daraz-edge-gateway",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "fastify": "^4.27.0",
    "@fastify/http-proxy": "^9.5.0",
    "ioredis": "^5.4.1",
    "jose": "^5.6.3",
    "opossum": "^8.1.4",
    "pino": "^9.2.0",
    "undici": "^6.19.2"
  }
}
```

### `gateway.js`
```javascript
import Fastify from 'fastify';
import { createRemoteJWKSet, jwtVerify } from 'jose';
import Redis from 'ioredis';
import CircuitBreaker from 'opossum';
import { request as undiciRequest } from 'undici';
import { randomUUID } from 'crypto';

const app = Fastify({
  logger: { level: 'info' },
  trustProxy: true,
  bodyLimit: 1 * 1024 * 1024,            // 1 MB
  disableRequestLogging: false,
});

const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost', port: 6379 });
const JWKS = createRemoteJWKSet(new URL('https://auth.daraz.com.bd/.well-known/jwks.json'), {
  cooldownDuration: 30_000,
  cacheMaxAge: 600_000,
});

const SERVICES = {
  '/api/v1/products': 'http://product-svc.internal:8080',
  '/api/v1/orders':   'http://order-svc.internal:8080',
  '/api/v1/payments': 'http://payment-svc.internal:8080',
  '/api/v1/cart':     'http://cart-svc.internal:8080',
  '/api/v1/users':    'http://user-svc.internal:8080',
};

// 1) Trace propagation hook
app.addHook('onRequest', async (req, reply) => {
  req.id = req.headers['x-request-id'] || randomUUID();
  if (!req.headers['traceparent']) {
    const traceId = randomUUID().replace(/-/g, '');
    const spanId  = randomUUID().replace(/-/g, '').slice(0, 16);
    req.headers['traceparent'] = `00-${traceId}-${spanId}-01`;
  }
  reply.header('X-Request-Id', req.id);
});

// 2) JWT validation (skip /health, /home public path)
const PUBLIC = new Set(['/health', '/home', '/api/v1/products']);  // products is GET-public
app.addHook('preHandler', async (req, reply) => {
  if (PUBLIC.has(req.routerPath) || req.method === 'OPTIONS') return;
  const auth = req.headers.authorization;
  if (!auth?.startsWith('Bearer ')) {
    return reply.code(401).send({ error: 'missing_token' });
  }
  try {
    const { payload } = await jwtVerify(auth.slice(7), JWKS, {
      issuer: 'https://auth.daraz.com.bd',
      audience: 'daraz-api',
    });
    req.user = payload;
    req.headers['x-user-id'] = payload.sub;
    req.headers['x-user-roles'] = (payload.roles || []).join(',');
  } catch (e) {
    return reply.code(401).send({ error: 'invalid_token', detail: e.code });
  }
});

// 3) Rate limit — token bucket per API key/user (Redis Lua)
const TOKEN_BUCKET_LUA = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local data = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(data[1]) or capacity
local ts = tonumber(data[2]) or now
local delta = math.max(0, now - ts)
tokens = math.min(capacity, tokens + delta * refill_rate)
local allowed = 0
if tokens >= 1 then tokens = tokens - 1; allowed = 1 end
redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', key, 3600)
return { allowed, tokens }
`;
const sha = await redis.script('LOAD', TOKEN_BUCKET_LUA);

app.addHook('preHandler', async (req, reply) => {
  const id = req.user?.sub || req.headers['x-api-key'] || req.ip;
  const cap = req.user ? 600 : 60;       // logged in: 600/min, anon: 60/min
  const rate = cap / 60;                 // per second
  const [allowed, remaining] = await redis.evalsha(
    sha, 1, `rl:${id}`, cap, rate, Math.floor(Date.now() / 1000)
  );
  reply.header('X-RateLimit-Limit', cap);
  reply.header('X-RateLimit-Remaining', Math.floor(remaining));
  if (!allowed) return reply.code(429).send({ error: 'rate_limited' });
});

// 4) Circuit breaker wrapper
function makeBreaker(name, target) {
  const fn = async (path, opts) => {
    const res = await undiciRequest(`${target}${path}`, {
      method: opts.method,
      headers: opts.headers,
      body: opts.body,
      headersTimeout: 3000,
      bodyTimeout: 5000,
    });
    if (res.statusCode >= 500) throw new Error(`upstream_5xx_${res.statusCode}`);
    return res;
  };
  return new CircuitBreaker(fn, {
    timeout: 5000,
    errorThresholdPercentage: 50,
    resetTimeout: 15000,
    rollingCountTimeout: 10000,
    name,
  });
}
const breakers = Object.fromEntries(
  Object.entries(SERVICES).map(([prefix, url]) => [prefix, makeBreaker(prefix, url)])
);

// 5) Generic proxy route
app.all('/api/v1/*', async (req, reply) => {
  const prefix = Object.keys(SERVICES).find(p => req.url.startsWith(p));
  if (!prefix) return reply.code(404).send({ error: 'no_route' });
  const breaker = breakers[prefix];
  try {
    const res = await breaker.fire(req.url, {
      method: req.method,
      headers: { ...req.headers, host: undefined },
      body: req.method === 'GET' ? undefined : JSON.stringify(req.body),
    });
    reply.code(res.statusCode);
    for (const [k, v] of Object.entries(res.headers)) {
      if (!['transfer-encoding', 'connection'].includes(k)) reply.header(k, v);
    }
    return reply.send(await res.body.text());
  } catch (err) {
    req.log.error({ err, prefix }, 'upstream_failed');
    return reply.code(503).send({ error: 'service_unavailable', service: prefix });
  }
});

// 6) Aggregation: GET /home
app.get('/home', async (req, reply) => {
  const userId = req.user?.sub;
  const headers = { 'traceparent': req.headers.traceparent, 'x-request-id': req.id };

  const calls = [
    undiciRequest(`${SERVICES['/api/v1/products']}/featured`, { headers }),
    undiciRequest(`${SERVICES['/api/v1/products']}/banners`,  { headers }),
    userId
      ? undiciRequest(`${SERVICES['/api/v1/cart']}/${userId}/count`, { headers })
      : Promise.resolve({ statusCode: 200, body: { json: async () => ({ count: 0 }) } }),
    userId
      ? undiciRequest(`${SERVICES['/api/v1/users']}/${userId}/notifications/unread`, { headers })
      : Promise.resolve({ statusCode: 200, body: { json: async () => ({ unread: 0 }) } }),
  ];

  const results = await Promise.allSettled(calls);
  const safeJson = async (r) =>
    r.status === 'fulfilled' && r.value.statusCode < 500
      ? await r.value.body.json().catch(() => null)
      : null;

  reply.send({
    featured:      await safeJson(results[0]),
    banners:       await safeJson(results[1]),
    cart:          await safeJson(results[2]),
    notifications: await safeJson(results[3]),
    degraded:      results.some(r => r.status === 'rejected'),
  });
});

// 7) Health & graceful shutdown
app.get('/health', async () => ({ ok: true }));

const shutdown = async () => {
  app.log.info('shutting_down');
  await app.close();
  await redis.quit();
  process.exit(0);
};
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

app.listen({ host: '0.0.0.0', port: 8000 });
```

**বৈশিষ্ট্য:** JWKS-cached JWT verify, Redis token bucket, opossum circuit breaker, Promise.allSettled aggregation, traceparent propagation, graceful shutdown। Production-এ এর সাথে যোগ করুন: Prometheus metrics endpoint, OpenTelemetry SDK, structured access log, helmet headers।

---

## 🐘 Implementation: Laravel-based Gateway (PHP)

ছোট স্কেলে / বা Laravel ecosystem-এ থাকলে middleware দিয়ে gateway বানানো যায়।

### `routes/api.php`
```php
Route::middleware(['gateway.jwt', 'gateway.ratelimit'])
    ->prefix('api/v1')->group(function () {
        Route::any('products/{path?}', [GatewayController::class, 'proxy'])
            ->where('path', '.*')
            ->defaults('service', 'product');
        Route::any('orders/{path?}',   [GatewayController::class, 'proxy'])
            ->where('path', '.*')
            ->defaults('service', 'order');
        Route::any('payments/{path?}', [GatewayController::class, 'proxy'])
            ->where('path', '.*')
            ->defaults('service', 'payment');
    });
```

### `app/Http/Controllers/GatewayController.php`
```php
namespace App\Http\Controllers;

use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Exception\GuzzleException;
use Illuminate\Http\Request;
use Psr\Http\Message\RequestInterface;

class GatewayController extends Controller
{
    private array $services = [
        'product' => 'http://product-svc.internal:8080',
        'order'   => 'http://order-svc.internal:8080',
        'payment' => 'https://payment-svc.internal:8443',
    ];

    public function proxy(Request $request, string $path = '')
    {
        $service = $request->route('service');
        $base    = $this->services[$service] ?? abort(404);

        $stack = HandlerStack::create();
        // Retry with exponential backoff (idempotent only)
        $stack->push(Middleware::retry(
            function ($retries, RequestInterface $req, $resp = null, $err = null) {
                if ($retries >= 2) return false;
                if (!in_array($req->getMethod(), ['GET', 'HEAD', 'PUT', 'DELETE'])) return false;
                return $err || ($resp && $resp->getStatusCode() >= 500);
            },
            fn($retries) => 200 * (2 ** $retries) + random_int(0, 100)
        ));
        // Inject internal headers
        $stack->push(Middleware::mapRequest(function (RequestInterface $req) use ($request) {
            return $req
                ->withHeader('X-User-Id', (string)($request->attributes->get('user_id') ?? ''))
                ->withHeader('X-Request-Id', (string)$request->header('X-Request-Id', \Str::uuid()))
                ->withHeader('traceparent', (string)$request->header('traceparent', ''));
        }));

        $client = new Client([
            'base_uri'        => $base,
            'handler'         => $stack,
            'connect_timeout' => 2,
            'timeout'         => 8,
            'http_errors'     => false,
            'verify'          => $service === 'payment' ? storage_path('certs/ca.pem') : true,
            'cert'            => $service === 'payment'
                ? [storage_path('certs/client.pem'), config('gateway.client_key_pass')]
                : null,
        ]);

        $upstreamPath = "/api/v1/{$service}/{$path}";

        try {
            $res = $client->request($request->method(), $upstreamPath, [
                'query'   => $request->query(),
                'headers' => collect($request->headers->all())
                    ->except(['host', 'authorization', 'cookie'])
                    ->map(fn($v) => is_array($v) ? $v[0] : $v)->toArray(),
                'body'    => $request->getContent() ?: null,
            ]);
        } catch (GuzzleException $e) {
            report($e);
            return response()->json(['error' => 'service_unavailable'], 503);
        }

        return response($res->getBody()->getContents(), $res->getStatusCode())
            ->withHeaders(array_diff_key(
                $res->getHeaders(),
                array_flip(['Transfer-Encoding', 'Connection'])
            ));
    }
}
```

### `app/Http/Middleware/GatewayJwt.php`
```php
namespace App\Http\Middleware;

use Closure;
use Firebase\JWT\JWK;
use Firebase\JWT\JWT;
use Illuminate\Support\Facades\Cache;

class GatewayJwt
{
    public function handle($request, Closure $next)
    {
        $token = $request->bearerToken();
        if (!$token) return response()->json(['error' => 'missing_token'], 401);

        $jwks = Cache::remember('jwks', 600, fn() =>
            json_decode(file_get_contents('https://auth.daraz.com.bd/.well-known/jwks.json'), true)
        );
        try {
            $payload = (array) JWT::decode($token, JWK::parseKeySet($jwks));
            if (($payload['aud'] ?? null) !== 'daraz-api') throw new \Exception('aud_mismatch');
            $request->attributes->set('user_id', $payload['sub']);
            $request->attributes->set('roles', $payload['roles'] ?? []);
        } catch (\Throwable $e) {
            return response()->json(['error' => 'invalid_token'], 401);
        }

        return $next($request);
    }
}
```

`GatewayRateLimit` middleware-এ Laravel-এর built-in `RateLimiter::attempt()` Redis driver দিয়ে ব্যবহার করুন (দেখুন `rate-limiting.md`)।

---

## 🏭 Production Concerns

### High Availability
- Multi-AZ deployment (AWS Mumbai: ap-south-1a, 1b, 1c)
- ALB/NLB-এর behind ৩+ gateway instance
- Health check endpoint (`/health`, `/ready`)
- Graceful shutdown — SIGTERM → drain connection → exit (Kubernetes preStop hook)

### Auto-scaling
- HPA on CPU + RPS + p99 latency custom metric
- Pre-scale before flash sale (Daraz 11.11 — 2 ঘণ্টা আগে scale up)
- Connection pooling — node-undici/HTTP keep-alive (avoid TCP handshake storm)

### Observability
| Aspect | Tool | Metric |
|--------|------|--------|
| Metrics | Prometheus + Grafana | RPS, latency p50/p95/p99, 4xx/5xx rate, breaker open count |
| Tracing | OpenTelemetry → Jaeger/Tempo | end-to-end span (gateway → svc → db) |
| Logging | Loki/ELK | JSON access log: req_id, user_id, route, status, upstream_ms |
| Alert | Alertmanager/PagerDuty | p99 > 1s for 5 min, 5xx > 1% |

### Secrets & Key Rotation
- Vault/AWS Secrets Manager-এ upstream credential, mTLS cert
- JWKS auto-refresh (cooldown 30s, cache 10 min)
- Rotate signing key without downtime — `kid` header + dual-key window

### Schema Validation
OpenAPI spec → request validation (Kong's `openapi-validator` plugin, Spectral CI)। Schema mismatch → 400 before reaching backend।

### Capacity Planning
- প্রতি instance ~5000 RPS (Kong/Envoy on c6i.xlarge)
- Redis: 100k ops/sec single node, cluster mode 1M+
- TLS: ECDSA cert (P-256) — ~3× faster handshake than RSA-2048

---

## 🚫 Anti-Patterns

| Anti-Pattern | কেন খারাপ | বিকল্প |
|--------------|-----------|--------|
| **Gateway-Monolith** | সব business logic gateway-এ → SPOF, slow deploy | Logic ফিরিয়ে দিন service-এ; gateway থিন রাখুন |
| **Business logic in plugin** | Kong custom Lua-এ order calculation | দরকার হলে BFF বা service-এ |
| **N+1 from gateway** | একটি request → ১০০টা downstream call | DataLoader/batching, GraphQL Federation |
| **Tight coupling to backend** | Gateway-এ DB schema জানা | Stable API contract, versioning |
| **Synchronous chaining** | A→B→C→D sequential | Parallel where possible, async event |
| **No timeout/retry** | Cascading failure | Bulkhead, circuit breaker, deadline |
| **Single gateway, all clients** | Mobile-Web-Partner একই config | Gateway-per-client + BFF |
| **Hard-coded routes in code** | প্রতি service add-এ deploy | Declarative config, GitOps |
| **Caching authenticated response globally** | Data leak between users | Vary by user/scope key |

---

## 🇧🇩 বাংলাদেশের Real-World Scenarios

### Daraz Bangladesh
- **Stack:** Kong cluster (hybrid mode) on AWS EKS Mumbai, 6 control + 12 data plane nodes
- Mobile, Web, Seller Center — তিনটি আলাদা route group
- Flash sale (11.11): peak ~50k RPS; pre-scaled to 30 instances
- Plugins: jwt, rate-limiting (Redis cluster), proxy-cache (banner/category 60s), prometheus, opentelemetry
- Backend: product-svc, cart-svc, order-svc, payment-svc, search-svc (Elasticsearch behind), recommendation-svc (Python ML)
- BFF layer: Daraz-mobile-bff (Node.js) Kong-এর behind, যেটা 7-8 service কে aggregate করে

### Pathao (Rides + Foods + Parcel)
- **Multiple gateways per business unit:**
  - `rider-gw.pathao.com` — driver app (location streaming, ride accept)
  - `customer-gw.pathao.com` — passenger app
  - `merchant-gw.pathao.com` — restaurant/parcel partner portal
- WebSocket gateway আলাদা — live ride tracking (sticky session, longer idle timeout)
- Surge pricing service-কে gateway-এ caching off (real-time)
- bKash payment redirect — gateway-এ idempotency-key mandatory

### bKash Merchant API Gateway
- **mTLS-only** — partner-কে CA-signed cert provide করা হয়
- Apigee/Kong with strict OAuth2 + HMAC signature verification
- Per-partner usage plan: tokenized checkout 1000 req/min, refund 100/min
- All requests audit-logged for Bangladesh Bank compliance

### Foodpanda BD
- AWS API Gateway (regional) + Lambda authorizer
- Driver location update WebSocket via API Gateway WebSocket API
- Rate limit per restaurant POS partner (API key based)

### Chaldal
- Nginx + OpenResty (Lua) custom gateway → cost-effective
- Inventory service heavy-cached (5s TTL) — avoid stampede with single-flight

### SSLCommerz (Payment Aggregator)
- Edge gateway: WAF + DDoS protection + IP allowlist for merchant callback
- Bank-side mTLS (Sonali, EBL, Brac, DBBL each different cert)
- Idempotency on transaction-id mandatory

### Grameenphone Digital Services (MyGP, Bioscope)
- Apigee (Google Cloud) — enterprise grade with API monetization
- Per-MSISDN rate limit, OAuth2 + scope-based access for partner apps

---

## 🪜 Migration Path: Monolith → Gateway → Microservices

**Strangler Fig Pattern**:

```
Phase 0: Pure Monolith
   Client ──► PHP Monolith (everything)

Phase 1: Introduce Gateway (no behavior change)
   Client ──► Gateway ──► PHP Monolith
              (routing, auth, logging only)

Phase 2: Carve out first microservice (e.g., Product)
   Client ──► Gateway ──┬─► Product Service (new)
                       └─► PHP Monolith (rest)
   Gateway routes /api/v1/products/* → Product Svc

Phase 3: Iterate per bounded context
   Gateway ──┬─► Product Svc
            ├─► Order Svc
            ├─► Payment Svc
            └─► Monolith (shrinking)

Phase 4: Monolith retired
   Gateway ──► Pure microservices + BFF layer
```

**Bangladesh Tips:**
- প্রথমে read-heavy endpoint বের করুন (product catalog)
- Database-সহ একসাথে split করবেন না — first dual-write/CDC, then split
- Gateway-এ feature flag — gradual cutover (5% → 50% → 100%)

---

## 🤔 কখন API Gateway ব্যবহার করবেন (এবং কখন নয়)

### ব্যবহার করুন যখন:
- ৫+ microservices public-এ expose করছেন
- Multi-client (Web/Mobile/Partner) এবং প্রতিটার আলাদা auth/rate
- Cross-cutting concerns (auth, rate, logging) consistent চান
- Backend internal protocol (gRPC) browser থেকে translate করতে হবে
- Partner integration — API key, usage plan, monetization দরকার

### এড়িয়ে চলুন যখন:
- ১-২টা service, ছোট internal app — Nginx reverse proxy যথেষ্ট
- Pure internal east-west traffic — service mesh বেশি appropriate
- Ultra-low latency (HFT, real-time gaming) — extra hop allowed না
- Team-এ DevOps capacity নেই — managed service (AWS API Gateway) নিন

---

## ⚠️ Common Pitfalls

1. **Single point of failure** — gateway down = all down. Multi-AZ + multiple instance must।
2. **Latency tax** — অতিরিক্ত hop। প্রতিটা plugin profile করুন (Kong-এ `kong-plugin-prometheus` দিয়ে)।
3. **JWKS fetch storm** — gateway restart হলে সব instance একসাথে JWKS fetch করে → auth server-এ DDoS। Cache + jitter।
4. **Cookie/Session forwarding** — internal service stateless রাখুন, cookie strip করুন।
5. **Body buffering** — large upload (e.g., merchant CSV) gateway-এ buffer হলে memory blow up। Streaming proxy use করুন।
6. **CORS misconfig** — `Access-Control-Allow-Origin: *` + `credentials: true` invalid; specific origin দিন।
7. **Retry storm** — gateway retry + service retry + client retry = exponential overload। Token-based deadline (e.g., `X-Request-Deadline`) propagate করুন।
8. **Logging PII** — Authorization header, body-এ password log করবেন না; redaction filter আবশ্যক।
9. **Plugin order matter** — auth → rate limit → transformation → proxy। ভুল order-এ unauth request quota খেয়ে ফেলবে।
10. **Cache poisoning** — `Vary` header ভুল হলে User A-এর private data User B পাবে।

---

## 📋 Production Readiness Checklist

### Architecture
- [ ] Edge vs internal gateway responsibility clearly separated
- [ ] BFF/aggregation logic gateway-এ নাকি আলাদা layer-এ — সিদ্ধান্ত নেয়া
- [ ] Multi-AZ deployment, ৩+ instance
- [ ] Service mesh-এর সাথে role overlap নেই

### Security
- [ ] TLS 1.2+ only, strong cipher suite, HSTS header
- [ ] mTLS to sensitive upstream (payment, bank)
- [ ] JWT validation with JWKS rotation, `kid` aware
- [ ] Body size limit, request timeout enforced
- [ ] WAF integrated (OWASP top 10)
- [ ] CORS restricted to known origin
- [ ] Secret-এ Vault/Secrets Manager, code/config-এ নয়

### Reliability
- [ ] Connect/read/total timeout per upstream
- [ ] Circuit breaker with fallback response
- [ ] Retry only on idempotent + with backoff + jitter
- [ ] Bulkhead / connection pool per upstream
- [ ] Graceful shutdown with connection draining
- [ ] Deadline propagation (`X-Request-Deadline` or gRPC deadline)

### Performance
- [ ] HTTP/2 to upstream, keep-alive enabled
- [ ] Response cache for hot read endpoints
- [ ] Single-flight / coalescing for cache stampede
- [ ] Load test: target RPS × 3 capacity
- [ ] p99 latency budget set (e.g., gateway adds < 30ms)

### Observability
- [ ] Prometheus metrics: RPS, latency histogram, error rate, breaker state
- [ ] OpenTelemetry trace + `traceparent` propagation
- [ ] Structured JSON access log with req_id, user_id, route, upstream
- [ ] PII redacted in log
- [ ] Alert: p99 > X, 5xx > 1%, breaker open, JWKS fetch fail

### Operations
- [ ] GitOps for declarative config (Kong YAML / Envoy CRD in Git)
- [ ] Canary deploy with traffic split (5% → 50% → 100%)
- [ ] Rate limit per consumer documented and tested
- [ ] Runbook: rollback, JWKS rotation, breaker reset, partner key revoke
- [ ] DR test: simulate AZ failure quarterly

### Compliance (Bangladesh)
- [ ] Audit log retained per Bangladesh Bank/BTRC requirement (payment, telco)
- [ ] Data residency — sensitive data Mumbai region OK?
- [ ] PCI-DSS scope minimization — payment card data never logged

---

## 📚 সারসংক্ষেপ

**API Gateway** হলো microservices-এর edge — যা client-কে দেয় single, secure, observable entry point এবং backend service-গুলোকে cross-cutting concern থেকে মুক্ত রাখে। সঠিক ব্যবহারে এটা scalability, security, এবং developer productivity—তিনটাই বাড়ায়; ভুল ব্যবহারে এটা single point of failure এবং hidden monolith হয়ে যায়।

**মূল মন্ত্র:**
1. Gateway-কে **thin** রাখুন — business logic-কে BFF/service-এ পাঠান।
2. **Configuration > code** — declarative YAML/CRD, GitOps।
3. **Observability first** — metrics, trace, log day-1 থেকে।
4. **Resilience by default** — timeout, retry, circuit breaker, bulkhead।
5. **Security at edge** — TLS, JWT, rate limit, WAF।

বাংলাদেশের context-এ ছোট startup-এর জন্য **AWS API Gateway** (HTTP API) cost-effective; Daraz/Pathao স্কেলে **Kong on EKS** বা **Envoy/Istio** flexibility ও control দেয়; bKash/SSLCommerz-এর মতো payment platform-এ **Apigee/Kong + mTLS + audit log** প্রয়োজন।

> **Senior tip:** Gateway-এ যা যোগ করছেন, প্রতিটা feature-এর জন্য জিজ্ঞেস করুন — "এটা কি cross-cutting? নাকি specific business logic?" Cross-cutting হলে gateway, না হলে service বা BFF। এই একটা প্রশ্ন আপনাকে gateway-monolith anti-pattern থেকে বাঁচাবে।
