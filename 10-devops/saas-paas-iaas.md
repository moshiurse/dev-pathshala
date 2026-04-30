# ☁️ SaaS, PaaS, IaaS — ক্লাউড সার্ভিস মডেল

> **"আপনি কতটুকু ম্যানেজ করবেন, আর কতটুকু ক্লাউড প্রোভাইডার?"** — এই প্রশ্নের উত্তরই হলো ক্লাউড সার্ভিস মডেল।
> Daraz, bKash, Pathao — সবাই কোনো না কোনো ক্লাউড মডেল ব্যবহার করছে। কোনটা কখন উপযুক্ত, সেই সিদ্ধান্তই startup এর CapEx/OpEx নির্ধারণ করে।

---

## 📖 সূচিপত্র

- [মূল ধারণা — Pizza-as-a-Service](#-মূল-ধারণা--pizza-as-a-service-অ্যানালজি)
- [শেয়ার্ড রেসপনসিবিলিটি মডেল](#-শেয়ারড-রেসপনসিবিলিটি-মডেল)
- [IaaS — Infrastructure as a Service](#-iaas--infrastructure-as-a-service)
- [PaaS — Platform as a Service](#-paas--platform-as-a-service)
- [SaaS — Software as a Service](#-saas--software-as-a-service)
- [FaaS / Serverless](#-faas--serverless)
- [CaaS — Container as a Service](#-caas--container-as-a-service)
- [BaaS — Backend as a Service](#-baas--backend-as-a-service)
- [DBaaS — Database as a Service](#-dbaas--database-as-a-service)
- [প্রাইসিং মডেল](#-প্রাইসিং-মডেল)
- [Vendor Lock-in vs Portability](#-vendor-lock-in-vs-portability)
- [Decision Framework](#-decision-framework)
- [BD Startup Cost Calculation](#-bd-startup-cost-calculation--১০ক-ইউজার)
- [Migration Path](#-migration-path)
- [Anti-patterns](#-anti-patterns)
- [Checklist](#-checklist)

---

## 🍕 মূল ধারণা — Pizza-as-a-Service অ্যানালজি

ক্লাউড মডেল বুঝতে সবচেয়ে সহজ উপমা — **পিজ্জা খাওয়ার ৪টি উপায়**।

```
┌──────────────────────────────────────────────────────────────────────┐
│                Pizza-as-a-Service অ্যানালজি                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ 1. On-Premises (নিজে বানাও)                                         │
│    আটা, পনির, সস, চুলা, গ্যাস — সব আপনার                              │
│    → নিজস্ব ডেটা সেন্টার (Mazeda, Aamra)                              │
│                                                                      │
│ 2. IaaS (Take & Bake — হাফ-রেডি)                                     │
│    পিজ্জা প্রি-মেড পান, কিন্তু বাড়ির চুলায় বেক করেন                    │
│    → AWS EC2, DigitalOcean Droplet (VM দেওয়া আছে, OS সেটআপ আপনার)   │
│                                                                      │
│ 3. PaaS (Pizza Delivery — পিজ্জা আসে গরম)                           │
│    শুধু খাওয়ার টেবিল, প্লেট, কোক আপনার                                │
│    → Heroku, Render, Vercel — কোড push করেন, বাকি সব platform এর    │
│                                                                      │
│ 4. SaaS (Restaurant — সব রেডি)                                      │
│    ঢুকুন, বসুন, খান                                                  │
│    → Gmail, Slack, Daraz Sellers Center — ব্যবহার করুন, ব্যাস       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Stack-level visualization

```
              ┌─────────────┬──────┬──────┬──────┬──────┐
              │ On-Premises │ IaaS │ PaaS │ FaaS │ SaaS │
┌─────────────┼─────────────┼──────┼──────┼──────┼──────┤
│ Application │     YOU     │  YOU │  YOU │  YOU │ ☁️   │
│ Data        │     YOU     │  YOU │  YOU │  YOU │ ☁️   │
│ Runtime     │     YOU     │  YOU │  ☁️  │  ☁️  │ ☁️   │
│ Middleware  │     YOU     │  YOU │  ☁️  │  ☁️  │ ☁️   │
│ OS          │     YOU     │  YOU │  ☁️  │  ☁️  │ ☁️   │
│ Virtualizn  │     YOU     │  ☁️  │  ☁️  │  ☁️  │ ☁️   │
│ Server      │     YOU     │  ☁️  │  ☁️  │  ☁️  │ ☁️   │
│ Storage     │     YOU     │  ☁️  │  ☁️  │  ☁️  │ ☁️   │
│ Networking  │     YOU     │  ☁️  │  ☁️  │  ☁️  │ ☁️   │
└─────────────┴─────────────┴──────┴──────┴──────┴──────┘
                  Control ⬆️         Convenience ⬆️
                  Cost (long-term) ⬇️ Cost (short-term) ⬇️
```

---

## 🛡️ শেয়ার্ড রেসপনসিবিলিটি মডেল

ক্লাউডে নিরাপত্তা **shared** — আপনি ও provider উভয়ের দায়িত্ব আছে।

### IaaS এ দায়িত্ব বণ্টন (AWS EC2)

| Layer | আপনার দায়িত্ব | AWS এর দায়িত্ব |
|-------|---------------|---------------|
| Physical security | ❌ | ✅ DC guard, biometric |
| Hypervisor | ❌ | ✅ Xen/Nitro |
| Network infra | ❌ | ✅ Backbone, DDoS scrubbing |
| OS patches | ✅ `apt update`, kernel CVE | ❌ |
| Firewall (Security Group) | ✅ port 22 কাকে দেবেন | ❌ |
| App code | ✅ SQLi, XSS prevent | ❌ |
| Data encryption | ✅ at-rest/in-transit | ⚠️ tools দেয় |
| IAM/credentials | ✅ key rotate | ⚠️ IAM service |
| Backup | ✅ snapshot schedule | ⚠️ tools দেয় |

### PaaS এ (Heroku, Render)

| Layer | আপনি | Provider |
|-------|------|----------|
| OS patch | ❌ | ✅ |
| Runtime (PHP/Node) | ❌ | ✅ stack auto-update |
| Web server config | ❌ | ✅ |
| App code | ✅ | ❌ |
| Env vars/secrets | ✅ | ⚠️ stored secure |
| Data | ✅ | ⚠️ storage layer |

### SaaS এ (Gmail, Slack)

| Layer | আপনি | Provider |
|-------|------|----------|
| Almost everything | ❌ | ✅ |
| User accounts/SSO | ✅ | — |
| Data classification | ✅ কী upload করছেন | — |
| Sharing permissions | ✅ | — |

> **নিয়ম:** যত উপরে যাবেন, provider তত বেশি দায়িত্ব নেবে — কিন্তু **আপনার data ও access control আপনারই দায়িত্ব**, সবসময়।

---

## 🖥️ IaaS — Infrastructure as a Service

### সংজ্ঞা

Virtualized hardware — VM, storage, network — API দিয়ে provision করা যায়। OS থেকে উপরে সব আপনার।

### কখন বেছে নেবেন?

- কাস্টম kernel টিউনিং দরকার (e.g., bKash-এর low-latency payment gateway)
- Legacy app যা নির্দিষ্ট OS/library চায়
- Compliance — data কোন physical region এ থাকবে নিজে control করতে চান
- Long-running, predictable workload — reserved instance এ সস্তা

### Provider তুলনা (BD-relevant)

| Provider | Region | $/mo (2vCPU/4GB) | BD latency | বিশেষত্ব |
|----------|--------|------------------|------------|----------|
| AWS EC2 t3.medium | Mumbai (ap-south-1) | ~$30 (on-demand) | ~40-60ms | সবচেয়ে বড় ecosystem |
| AWS EC2 (reserved 1yr) | Mumbai | ~$18 | — | ৪০% সস্তা |
| AWS Spot | Mumbai | ~$9 | — | batch/CI runner |
| GCP Compute Engine e2-medium | Mumbai/Singapore | ~$25 | ~50ms | sustained-use auto discount |
| Azure VM B2s | Pune/Singapore | ~$30 | ~50ms | Microsoft stack |
| **DigitalOcean Droplet** | Singapore (sgp1) | **$24** (2vCPU/4GB) | ~70-90ms | flat-rate, simple billing |
| Vultr | Singapore | $24 | ~70ms | DO-clone, hourly |
| Linode (Akamai) | Singapore/Mumbai | $24 | ~50-80ms | Akamai CDN integration |
| Hetzner Cloud | Helsinki/Falkenstein | €5 (CX21) | ~250ms ❌ | সবচেয়ে সস্তা, কিন্তু latency BD-এর জন্য খারাপ |
| **Eicra/ExonHost** | Dhaka (BDIX) | ৳1500-3000 | <10ms ✅ | local users দের জন্য fastest |
| **ICC/Mazeda** | Dhaka | ৳2000-5000 | <10ms | enterprise, BDIX peering |

### BD context — region নির্বাচন

```
                BD ইউজারের দিক থেকে latency
┌──────────────────────────────────────────────────────┐
│  BDIX-connected local DC ........... 2-10ms  ⚡⚡⚡  │
│  AWS Mumbai (ap-south-1) ........... 40-60ms  ⚡⚡   │
│  GCP Mumbai (asia-south1) .......... 45-65ms  ⚡⚡   │
│  AWS Singapore (ap-southeast-1) .... 70-90ms  ⚡    │
│  DigitalOcean Singapore (sgp1) ..... 70-90ms  ⚡    │
│  AWS Tokyo .......................... 100-130ms      │
│  AWS US-East ........................ 220-280ms ❌  │
│  Hetzner EU ......................... 200-280ms ❌  │
└──────────────────────────────────────────────────────┘
```

> **Daraz, Pathao, bKash সাধারণত AWS Mumbai + Cloudflare CDN ব্যবহার করে।** Cloudflare এর Dhaka POP থাকায় static asset ~5ms তে পৌঁছায়।

---

## 🚀 PaaS — Platform as a Service

### সংজ্ঞা

আপনি কোড push করেন, platform বাকি সব করে — build, deploy, scale, SSL, log।

### বিখ্যাত PaaS

| Provider | strong suit | starting price | BD-friendly? |
|----------|-------------|----------------|--------------|
| Heroku | original PaaS, Rails/Node | $5/mo (eco) → $25 (basic) | কার্ড লাগে, free tier নেই |
| **Render** | Heroku replacement | free tier আছে, $7/mo starter | ✅ |
| **Railway** | ডেভেলপার-প্রিয় UX | $5 credit/mo free | ✅ |
| **Fly.io** | global edge, BD distance OK | pay-as-you-go, ~$2/mo small | ✅ region: bom, sin |
| **Vercel** | Next.js/frontend | hobby free | ✅ Cloudflare-like edge |
| Netlify | JAMstack | free tier | ✅ |
| AWS Elastic Beanstalk | EC2 wrapper | EC2 cost | দরকার AWS account |
| GCP App Engine | Standard/Flex env | free quota | দরকার GCP billing |
| Azure App Service | .NET-friendly | ~$13/mo basic | — |
| **DigitalOcean App Platform** | DO simplicity | $5/mo starter | ✅ Singapore region |

### PaaS এর গুণ

✅ Zero-ops — git push, done
✅ Auto SSL (Let's Encrypt baked in)
✅ Built-in CI/CD
✅ Auto-scale (managed)
✅ Add-ons (Postgres, Redis) এক click

### PaaS এর ব্যথা

❌ Vendor lock-in (Heroku-specific buildpacks, Vercel-specific edge functions)
❌ Cost predictability খারাপ (traffic spike → bill spike)
❌ Custom infra impossible — চাইলে UDP/raw TCP server আনতে পারবেন না
❌ Egress bandwidth pricey (Vercel Pro: $40/100GB egress)
❌ Database vendor-locked (Heroku Postgres = AWS RDS, কিন্তু export কষ্ট)

---

## 🎁 SaaS — Software as a Service

### সংজ্ঞা

End-user-ready application, browser/app থেকে সরাসরি ব্যবহার। আপনি subscription দেন।

### বিখ্যাত SaaS

- **Productivity:** Gmail, Google Workspace, Microsoft 365, Notion, Slack
- **CRM/Sales:** Salesforce, HubSpot, Pipedrive
- **DevOps:** GitHub, GitLab Cloud, CircleCI, Datadog, Sentry
- **BD-specific:**
  - **bKash for Business** — merchant SaaS dashboard
  - **Pathao Tong** — restaurant POS SaaS
  - **Daraz Sellers Center** — multi-tenant seller portal
  - **Shoppingaa, Sheba.xyz** — local SaaS
  - **Tally on Cloud** (BD accountants এর জন্য)

### SaaS অ্যাপ্লিকেশন ডিজাইন — Multi-tenancy

আপনি যদি SaaS **বানাচ্ছেন** (just consume নয়), মূল প্রশ্ন — **tenant data কীভাবে আলাদা রাখবেন?**

```
┌─────────────────────────────────────────────────────────────┐
│             Multi-tenancy Models                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣ SILO (Database-per-tenant)                              │
│     ─────────────────────────                               │
│     Tenant A → DB_A                                         │
│     Tenant B → DB_B                                         │
│     Tenant C → DB_C                                         │
│                                                             │
│     ✅ Maximum isolation (compliance, GDPR)                  │
│     ✅ Per-tenant backup/restore সহজ                          │
│     ✅ Noisy neighbor নেই                                     │
│     ❌ Cost বেশি (১০০০ টেন্যান্ট = ১০০০ DB)                  │
│     ❌ Schema migration nightmare                             │
│     উদাহরণ: SAP, Salesforce enterprise tier                  │
│                                                             │
│  2️⃣ POOL (Shared DB, shared schema, tenant_id column)       │
│     ───────────────────────────────────────                 │
│     SELECT * FROM orders WHERE tenant_id = ? AND ...        │
│                                                             │
│     ✅ Cheapest, easiest to scale                            │
│     ✅ Single migration                                       │
│     ❌ One bug = সব tenant এর data leak                       │
│     ❌ Noisy neighbor risk                                    │
│     ❌ Per-tenant backup কঠিন                                  │
│     উদাহরণ: Slack, Notion (ছোট workspace)                    │
│                                                             │
│  3️⃣ BRIDGE (Shared DB, schema-per-tenant)                   │
│     ────────────────────────────────                        │
│     PostgreSQL: SET search_path = tenant_a                   │
│                                                             │
│     ✅ Middle ground — isolation + cost                      │
│     ✅ tenant drop = DROP SCHEMA                             │
│     ❌ PG: ~5000 schema এর পর slow                            │
│     উদাহরণ: Citus, mid-size B2B SaaS                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> বিস্তারিত sharding/isolation strategy — `05-database/sharding.md` দেখুন।

### Tenant isolation কোডে (Laravel example)

```php
// app/Models/Concerns/BelongsToTenant.php
trait BelongsToTenant
{
    protected static function bootBelongsToTenant(): void
    {
        // SELECT এ স্বয়ংক্রিয় filter
        static::addGlobalScope('tenant', function ($query) {
            if ($tenantId = app('currentTenant')?->id) {
                $query->where('tenant_id', $tenantId);
            }
        });

        // INSERT এ স্বয়ংক্রিয় tenant_id
        static::creating(function ($model) {
            if (!$model->tenant_id && $tenantId = app('currentTenant')?->id) {
                $model->tenant_id = $tenantId;
            }
        });
    }
}

// Order model
class Order extends Model
{
    use BelongsToTenant;
}

// Middleware — সাবডোমেইন থেকে tenant resolve
class IdentifyTenant
{
    public function handle($request, Closure $next)
    {
        $subdomain = explode('.', $request->getHost())[0];
        // shop1.mysaas.com.bd → tenant: shop1
        $tenant = Tenant::where('slug', $subdomain)->firstOrFail();
        app()->instance('currentTenant', $tenant);
        return $next($request);
    }
}
```

### Per-tenant rate-limiting (Node.js + Redis)

```javascript
// প্রতি tenant এর জন্য আলাদা rate limit (plan-based)
const Redis = require('ioredis');
const redis = new Redis();

async function tenantRateLimit(req, res, next) {
  const tenant = req.tenant; // {id, plan: 'free'|'pro'|'enterprise'}
  const limits = {
    free: { rpm: 60, burst: 10 },
    pro: { rpm: 1000, burst: 100 },
    enterprise: { rpm: 10000, burst: 1000 }
  };
  const limit = limits[tenant.plan];

  const key = `rl:${tenant.id}:${Math.floor(Date.now() / 60000)}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 60);

  if (count > limit.rpm) {
    res.set('Retry-After', '60');
    return res.status(429).json({
      error: 'Rate limit exceeded',
      plan: tenant.plan,
      upgrade_url: 'https://saas.com.bd/pricing'
    });
  }
  next();
}
```

### Billing — Stripe integration

```javascript
// নতুন subscription
const stripe = require('stripe')(process.env.STRIPE_SECRET);

async function subscribeTenant(tenantId, priceId) {
  const tenant = await Tenant.findById(tenantId);

  let customerId = tenant.stripe_customer_id;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: tenant.email,
      metadata: { tenant_id: tenantId }
    });
    customerId = customer.id;
    await tenant.update({ stripe_customer_id: customerId });
  }

  const subscription = await stripe.subscriptions.create({
    customer: customerId,
    items: [{ price: priceId }],
    trial_period_days: 14,
    metadata: { tenant_id: tenantId }
  });

  return subscription;
}

// Webhook — payment failed হলে suspend
app.post('/webhooks/stripe', async (req, res) => {
  const event = stripe.webhooks.constructEvent(
    req.body, req.headers['stripe-signature'], process.env.STRIPE_WEBHOOK_SECRET
  );

  if (event.type === 'invoice.payment_failed') {
    const tenantId = event.data.object.customer_metadata.tenant_id;
    await Tenant.update({ status: 'suspended' }, { where: { id: tenantId } });
  }
  res.json({ received: true });
});
```

> **BD note:** Stripe এখনো BD merchant account সরাসরি দেয় না (২০২৪ পর্যন্ত)। বিকল্প — **SSLCommerz, ShurjoPay, AamarPay** local gateway, অথবা US/SG entity দিয়ে Stripe।

### Feature flags — plan-based

```php
// LaunchDarkly / Unleash / নিজস্ব
class FeatureFlag {
  public static function enabled(string $flag, Tenant $t): bool {
    $features = [
      'free' => ['basic_reports'],
      'pro'  => ['basic_reports', 'advanced_reports', 'api_access'],
      'enterprise' => ['*'],
    ];
    $allowed = $features[$t->plan] ?? [];
    return in_array('*', $allowed) || in_array($flag, $allowed);
  }
}

// usage
if (FeatureFlag::enabled('api_access', $tenant)) {
  // expose API endpoints
}
```

---

## ⚡ FaaS / Serverless

### সংজ্ঞা

আপনি **শুধু function লেখেন**, ক্লাউড কোম্পানি event এর সময় run করে এবং execution time × memory অনুযায়ী charge করে।

> বিস্তারিত আর্কিটেকচার — `07-architectural-patterns/serverless.md` দেখুন। এখানে শুধু operational/pricing aspect।

### Provider তুলনা

| Provider | Free tier (per month) | Cold start | Max exec | Memory |
|----------|----------------------|------------|----------|--------|
| AWS Lambda | 1M req + 400K GB-s | 100-1000ms (Node ~200ms) | 15 min | 128MB - 10GB |
| Cloudflare Workers | 100K req/day free | <5ms (V8 isolates) | 30s CPU (paid) | 128MB |
| Vercel Functions | 100K invocations | 100-500ms | 60s (hobby) / 5min (pro) | up to 3GB |
| GCP Cloud Functions | 2M req | 200-2000ms | 9 min (gen 1), 60 min (gen 2) | up to 16GB |
| Azure Functions | 1M req | similar | 5/10 min (consumption) | up to 14GB |

### Cold start বাস্তবতা

```
┌────────────────────────────────────────────────────┐
│  Cold start latency (Node.js, average)             │
├────────────────────────────────────────────────────┤
│  Cloudflare Workers ........ <5ms      ✅          │
│  AWS Lambda (1024MB) ....... 100-300ms            │
│  AWS Lambda (128MB) ........ 500-1500ms ⚠️        │
│  AWS Lambda + VPC .......... 1-3s ❌ (avoid)      │
│  AWS Lambda + ARM (Graviton) 80-200ms (cheaper)   │
│  Vercel Edge Functions ..... <50ms                │
│  GCP gen-2 ................. 200-800ms            │
└────────────────────────────────────────────────────┘
```

### Pricing example — bKash transaction notification

ধরা যাক bKash এ প্রতি মাসে ১০ মিলিয়ন SMS notification trigger হয়, প্রতি function ~৫০০ms × 256MB।

```
AWS Lambda:
- Requests:  10M × $0.20/1M       = $2
- Compute:   10M × 0.5s × 0.25GB = 1.25M GB-s
             1.25M × $0.0000166  = $20.75
- Total                          ≈ $23/মাস (Mumbai)

Cloudflare Workers (paid plan):
- Base                            $5/mo
- 10M req × $0.30/M (>10M free)   $0
- Total                          = $5/মাস ✅ (অনেক সস্তা যদি use case fit হয়)

EC2 t3.small (1 vCPU/2GB):
- $15/mo (always on, 24/7)
- কিন্তু 10M event handle করতে capacity যথেষ্ট নয় → t3.medium ×2 = $60
```

### কখন FaaS **নয়**

❌ Long-running job (>15 min)
❌ Stateful, WebSocket connection (Lambda → API Gateway WS hack costly)
❌ Predictable, sustained 24/7 load (always-on EC2 cheaper)
❌ Heavy cold-start sensitive UX (login API)
❌ Database connection pool issues (Lambda × 1000 concurrent → Postgres পুড়ে যাবে; RDS Proxy/PgBouncer লাগবে)

---

## 🐳 CaaS — Container as a Service

মাঝামাঝি — IaaS এর control + PaaS এর simplicity, container-based।

| Provider | বৈশিষ্ট্য | দাম |
|----------|----------|-----|
| AWS ECS/Fargate | Serverless container, no VM management | ~$30/mo (0.25vCPU/0.5GB always on) |
| AWS EKS | Managed K8s | $73/mo (control plane) + nodes |
| GCP GKE Autopilot | Pay per pod, fully managed | similar |
| Azure AKS | Managed K8s, control plane free | nodes only |
| **DigitalOcean Kubernetes (DOKS)** | $12/mo control plane FREE, nodes $12+ | ✅ BD startup-friendly |
| Google Cloud Run | Container PaaS, scales to zero | pay per req, very BD-friendly |
| **Fly.io** | Container-based, global | ~$2-5/mo small |

> Kubernetes deep dive: `kubernetes.md`। Docker: `docker.md`।

---

## 🔌 BaaS — Backend as a Service

Frontend dev দের stack — auth, DB, file storage সব ready।

| Provider | বৈশিষ্ট্য | কখন? |
|----------|----------|------|
| **Firebase** | Realtime DB, Auth, FCM push, Hosting | mobile app, prototype, hackathon |
| **Supabase** | Open-source Firebase, Postgres-based, self-hostable | Firebase + SQL চাইলে |
| **AWS Amplify** | AWS ecosystem-tied | যদি AWS থাকেন |
| **Appwrite** | Self-hostable BaaS | সম্পূর্ণ control |
| **PocketBase** | Single-binary, SQLite | tiny project |

### BD context — Pathao early days

Pathao তাদের MVP **Firebase Realtime DB** দিয়ে শুরু করেছিল (rumour), কারণ:
- ride matching = realtime listener
- driver location push = `child_added` event
- কোনো backend dev না লাগিয়েই app চালু

কিন্তু scale (১০M+ user) এ Firebase pricing ও query limit এর জন্য তারা **PostgreSQL + Redis + Kafka** এ migrate করে।

---

## 🗄️ DBaaS — Database as a Service

| Provider | DB type | BD region | বিশেষত্ব |
|----------|---------|-----------|----------|
| **AWS RDS** (MySQL/Postgres/MariaDB) | SQL | Mumbai | classic, multi-AZ |
| **AWS Aurora** | MySQL/PG-compatible | Mumbai | 5x faster, auto-storage scale |
| **AWS Aurora Serverless v2** | same | Mumbai | scales to 0.5 ACU |
| **Google Cloud SQL** | SQL | Mumbai | similar to RDS |
| **PlanetScale** | MySQL (Vitess) | Mumbai/Singapore | branching, no FK ⚠️ |
| **Neon** | Postgres (serverless, branching) | Singapore/AWS | scales to zero |
| **Supabase** | Postgres + REST + Auth | Singapore | open-source friendly |
| **CockroachDB Cloud** | Distributed SQL | Mumbai | global consistency |
| **MongoDB Atlas** | Document | Mumbai | M0 free tier 512MB |
| **Redis Cloud** | KV cache | Mumbai | 30MB free |
| **Upstash Redis** | Serverless Redis | Singapore | per-request pricing, BD-cheap |

### কেন self-host না করে DBaaS?

✅ Backup, replication, point-in-time recovery built-in
✅ Patch, upgrade managed
✅ Multi-AZ failover automatic
✅ Encryption at-rest, TLS, IAM auth

❌ **৩x-৫x বেশি দাম** self-hosted vs DBaaS (BD startup এর জন্য বড় ব্যাপার)
❌ Egress charge (Mumbai RDS → Dhaka VPS = bandwidth bill)
❌ Some advanced extension (`pg_cron`, custom UDF) restricted

### DBaaS pricing comparison (16GB RAM Postgres)

| Option | $/মাস |
|--------|-------|
| Self-host on DO Droplet (4vCPU/8GB) | $48 |
| AWS RDS db.t3.large (8GB) | $130 |
| Aurora Serverless v2 (2-4 ACU) | $90-180 |
| Neon Pro (autosuspend) | $19 + usage |
| Supabase Pro | $25 + usage |

---

## 💵 প্রাইসিং মডেল

```
┌────────────────────────────────────────────────────────────┐
│                  প্রাইসিং মডেল                              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  💰 Pay-as-you-go                                          │
│  ────────────────                                          │
│  ব্যবহার যত, বিল তত। Default for cloud.                    │
│  Pros: কোনো commitment নেই                                  │
│  Cons: বিল predict কঠিন                                    │
│                                                            │
│  📅 Reserved (1yr/3yr commit)                              │
│  ────────────────────────                                  │
│  AWS RI, Azure Reserved Instance — ৪০-৭৫% সস্তা             │
│  Pros: predictable workload এ অনেক সাশ্রয়                  │
│  Cons: under-utilization এ টাকা waste                       │
│                                                            │
│  ⚡ Spot / Preemptible                                     │
│  ───────────────────                                       │
│  Unused capacity, ৭০-৯০% discount, কিন্তু ২ মিনিট notice এ │
│  reclaim হতে পারে। Batch job, CI runner, ML training এ ভালো│
│                                                            │
│  🆓 Free Tier                                              │
│  ──────────                                                │
│  AWS: 12 months t2.micro free                               │
│  GCP: $300 credit, 90 days                                 │
│  DO: $200 credit, 60 days (referral)                        │
│  Cloudflare: forever-free Workers/R2/KV                     │
│  Oracle Cloud: ARM 4 vCPU/24GB always-free ⚡ (BD-friendly)│
│                                                            │
│  📦 Savings Plans (AWS)                                    │
│  ─────────────────                                         │
│  $/hr commitment, RI এর চেয়ে flexible                       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Hidden costs — যেগুলো bill এ surprise দেয়

⚠️ **Egress / data transfer:** AWS Mumbai → internet = $0.09/GB। ১০০GB user download = $9/mo, কিন্তু S3 → CloudFront → user free (CloudFront cache hit)
⚠️ **NAT Gateway:** $0.045/hr × 24 × 30 = $32/mo per NAT (private subnet)
⚠️ **Load Balancer:** ALB = $20/mo idle
⚠️ **EBS snapshots:** $0.05/GB-mo, weekly = monthly bill
⚠️ **Cross-AZ traffic:** $0.01/GB even within VPC
⚠️ **Cloudwatch logs ingestion:** $0.50/GB ingest, $0.03/GB store

> **BD startup tip:** $৫০/মাস budget startup AWS এ গিয়ে প্রথম bill দেখে $২৫০ পেলে হার্ট অ্যাটাক হয়। **DigitalOcean / Vultr এ flat-rate pricing** beginner দের জন্য safer।

---

## 🔒 Vendor Lock-in vs Portability

```
┌──────────────────────────────────────────────────────────┐
│             Lock-in spectrum                             │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Low lock-in  ←─────────────────────→  High lock-in     │
│                                                          │
│  bare VM        Docker        Kubernetes    AWS Lambda   │
│  PostgreSQL     Postgres      Aurora        DynamoDB     │
│  Redis          Redis         ElastiCache   DAX          │
│  S3 (MinIO)     S3-compat     S3            ─────         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Lock-in কমানোর strategy

✅ **Open standard ব্যবহার করুন:** S3 API (MinIO compatible), PostgreSQL (RDS/Aurora/Self-host), Redis (ElastiCache/Upstash/self)
✅ **Containerize:** Docker ইমেজ যেকোনো cloud এ run করে
✅ **Terraform/Pulumi:** infra-as-code, multi-cloud module
✅ **Abstraction layer:** Laravel filesystem driver — S3/local/B2 swap
❌ যেগুলো avoid: AWS DynamoDB, Cosmos DB, Bigtable (proprietary API)

### Multi-cloud — কখন worth?

✅ Compliance (data sovereignty)
✅ Major-vendor outage protection (AWS US-East dies → Cloudflare/GCP fallback)
❌ ছোট startup এর জন্য over-engineering — single-cloud + good backup এ যথেষ্ট

---

## 🧭 Decision Framework

```
শুরু করছেন? ছোট team, MVP?
│
├─ Yes → PaaS (Render, Vercel, DO App Platform)  ✅
│        OR FaaS (Cloudflare Workers, Lambda)
│
├─ Predictable 24/7 load, custom infra needs?
│  └─ Yes → IaaS (EC2 reserved, DO Droplet)       ✅
│
├─ Container-native, ৫+ service?
│  └─ Yes → CaaS (DOKS, ECS Fargate, GKE Autopilot) ✅
│
├─ Mobile/web app, no backend dev?
│  └─ Yes → BaaS (Firebase, Supabase)              ✅
│
├─ Just need DB?
│  └─ Yes → DBaaS (RDS, Neon, PlanetScale)         ✅
│
└─ Use ready-made tool (CRM, email, etc)?
   └─ Yes → SaaS (HubSpot, SendGrid)                ✅
```

### বিস্তারিত matrix

| প্রশ্ন | IaaS | PaaS | FaaS | CaaS | SaaS |
|------|------|------|------|------|------|
| OS-level control দরকার? | ✅ | ❌ | ❌ | ⚠️ | ❌ |
| 24/7 sustained traffic? | ✅ | ✅ | ❌ | ✅ | — |
| Spiky/event-driven? | ❌ | ⚠️ | ✅ | ⚠️ | — |
| Sub-second cold start critical? | ✅ | ✅ | ⚠️ CF Workers | ✅ | ✅ |
| Team has DevOps skill? | required | optional | optional | required | none |
| Scale to zero? | ❌ | ⚠️ | ✅ | ⚠️ Cloud Run | ✅ |
| Vendor lock-in concern? | low | high | very high | medium | very high |
| Compliance/data residency? | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |

---

## 💸 BD Startup Cost Calculation — ১০K ইউজার

ধরা যাক একটি বাংলাদেশি e-commerce startup, ১০,০০০ MAU, ~১০০ orders/day, peak ৫০ concurrent users, ১GB DB, ১০GB images/month.

### Option A — Pure IaaS (DigitalOcean Singapore)

| Component | Spec | $/mo | ৳/mo (1$=110৳) |
|-----------|------|------|----------------|
| App Droplet | 2vCPU/4GB | $24 | ৳2,640 |
| DB Droplet (self-host PG) | 2vCPU/4GB | $24 | ৳2,640 |
| Spaces (S3-compat, 250GB) | + CDN | $5 | ৳550 |
| Load Balancer | DO LB | $12 | ৳1,320 |
| Snapshots/backup | weekly | $5 | ৳550 |
| Cloudflare Free | DNS + CDN | $0 | ৳0 |
| **Total** | | **$70** | **৳7,700** |

### Option B — PaaS heavy

| Component | $/mo |
|-----------|------|
| Render Web Service Standard | $25 |
| Render Postgres Standard | $20 |
| Render Redis | $10 |
| Cloudflare R2 (10GB) | $0.15 |
| Cloudflare CDN free | $0 |
| **Total** | **~$55** |

### Option C — Serverless

| Component | $/mo |
|-----------|------|
| Cloudflare Workers Paid | $5 |
| Neon Postgres Pro | $19 |
| Upstash Redis | $0 (free tier) |
| R2 storage 10GB | $0.15 |
| **Total** | **~$24** ⚡ |

### Option D — AWS over-engineered (typical mistake)

| Component | $/mo |
|-----------|------|
| ALB | $20 |
| 2× EC2 t3.medium | $60 |
| RDS db.t3.medium Multi-AZ | $130 |
| ElastiCache t3.micro | $13 |
| NAT Gateway | $32 |
| S3 + CloudFront | $5 |
| Route53 + CloudWatch | $10 |
| **Total** | **~$270** ❌ overkill for 10K MAU |

> **শিক্ষা:** ১০K MAU এ AWS production-grade setup টানার দরকার নেই। **DigitalOcean / Render / Cloudflare combo** সাশ্রয়ী এবং BDT 7,000-10,000/মাসে চলে — যা বাংলাদেশি startup এর budget এ মানানসই। Daraz/bKash scale (১M+ users) এ গিয়ে AWS Mumbai জাস্টিফাই হয়।

---

## 🚚 Migration Path

### ১. PaaS → IaaS (when bill grows)

Heroku $200/mo → DO Droplet $24 + কিছু DevOps effort
Step:
- Dockerize app
- DB export (`pg_dump`) → restore on managed DB or self-host
- DNS cut-over (CNAME swap)
- Worker/cron জাব্বে migrate

### ২. IaaS → CaaS

Single VM → Kubernetes
Step:
- Helm chart বানান
- ConfigMap/Secret এ env migrate
- Ingress (Nginx/Traefik) এ TLS
- Blue-green for zero downtime (`deployment-strategies.md`)

### ৩. Monolith on PaaS → Serverless

ধাপে ধাপে strangler pattern:
- Image processing → Lambda
- Email sender → SQS + Lambda
- Webhook handler → API Gateway + Lambda
- Core app monolith রাখুন

### ৪. SaaS dependence কমানো

vendor outage protection:
- SendGrid → fallback to Mailgun via abstraction layer
- Stripe → SSLCommerz fallback (BD)
- Firebase → migrate to Supabase (Postgres self-hostable)

---

## ⛔ Anti-patterns

❌ **"AWS Use Everything Just Because"** — ১০K user এ ECS+Fargate+RDS Multi-AZ+Aurora overkill। Bill ১০x হবে।

❌ **PaaS এ heavy compute task** — Heroku/Render এ video transcoding করলে dyno timeout, bill পুড়ে যাবে। FaaS বা dedicated worker VM ব্যবহার করুন।

❌ **Free-tier-only architecture** — production এ free tier rate limit hit করলে app down। Always have paid fallback।

❌ **Lambda + Postgres direct connection (no proxy)** — 1000 concurrent Lambda = 1000 PG connection = DB ক্র্যাশ। **RDS Proxy / PgBouncer / Supabase pooler** ব্যবহার করুন।

❌ **Vendor lock-in unaware** — DynamoDB তে ২ বছর data রাখার পর migrate করতে চাইলে rewrite হবে।

❌ **No cost alarm** — AWS Budgets / DO Billing alert সেট না করে credit card দিলে $৫,০০০ bill এক রাতে আসতে পারে (DDoS amplify scenario)।

❌ **DBaaS এ heavy egress** — AWS RDS Mumbai → DO Droplet Singapore = প্রতি GB $0.09 = মাসে $৫০-১০০ extra।

❌ **Single-cloud, single-region for critical service** — bKash যদি শুধু Mumbai এ থাকে এবং AWS Mumbai down হয়, পুরো দেশের payment বন্ধ। Multi-region/multi-AZ মাস্ট।

❌ **SaaS multi-tenancy in pool model without strong tenant_id checks** — One missed `WHERE tenant_id = ?` = সব tenant এর data leak। **RLS (Row Level Security)** ব্যবহার করুন PostgreSQL এ।

---

## 🎯 কখন কোনটা — চূড়ান্ত summary

| Scenario | Recommendation |
|----------|----------------|
| Solo dev, side project | **PaaS** (Render free) + **DBaaS** (Neon free) |
| BD startup MVP (<10K users) | **DO Droplet + Spaces + Cloudflare** ($30-70/mo) |
| Growing SaaS (10K-100K users) | **DOKS / ECS Fargate + RDS** ($200-500/mo) |
| Enterprise / banking (bKash) | **Multi-AZ AWS / on-prem hybrid + DR** |
| Spiky workload (event-driven) | **FaaS (Lambda/CF Workers)** |
| Internal tool, no traffic spike | **bare DO Droplet + nginx** |
| Mobile-first MVP | **Firebase / Supabase** |
| Compliance (BD Bank rule, DPP) | **Local DC (Mazeda/Aamra) IaaS** |

---

## ✅ Checklist

### Provider নির্বাচনের আগে
- [ ] Target user latency requirement (BD-only? global?) নির্ধারণ
- [ ] Region availability check (Mumbai / Singapore / local DC)
- [ ] Free tier limits পড়েছেন (যাতে bill surprise না হয়)
- [ ] Egress cost calculate করেছেন
- [ ] Compliance/data residency law check (BD Bank, NBR)
- [ ] Backup/DR strategy define
- [ ] Cost alarm/budget alert set

### SaaS বানানোর সময় (multi-tenancy)
- [ ] Silo/Pool/Bridge মডেল decide
- [ ] tenant_id সব table এ + index
- [ ] PostgreSQL RLS / app-level filter enforce
- [ ] Per-tenant rate-limit
- [ ] Per-tenant backup/export
- [ ] Subscription/billing (Stripe/SSLCommerz)
- [ ] Plan-based feature flag
- [ ] Tenant onboarding/offboarding flow
- [ ] Tenant data export (GDPR-compatible)

### FaaS deploy এর আগে
- [ ] Cold start test (p99 latency acceptable?)
- [ ] Database connection pooler (RDS Proxy / PgBouncer)
- [ ] Timeout < provider max (15 min Lambda)
- [ ] Memory size — cost vs speed tradeoff
- [ ] VPC attached only when absolutely needed
- [ ] Reserved concurrency (poison-pill protection)

### Cost Optimization
- [ ] Reserved Instance / Savings Plan analyze
- [ ] Spot/Preemptible for non-prod, batch
- [ ] Unused EBS/snapshot cleanup
- [ ] Right-size — CPU utilization < 30% হলে downsize
- [ ] CloudFront/Cloudflare cache hit ratio > 90%
- [ ] S3 lifecycle policy (IA, Glacier)
- [ ] Cross-region traffic minimize

---

## 📚 আরো পড়ুন

- `cloud.md` — AWS/GCP/Azure সার্ভিস বিস্তারিত
- `kubernetes.md` — CaaS deep dive
- `docker.md` — কন্টেইনার fundamentals
- `deployment-strategies.md` — blue-green, canary
- `07-architectural-patterns/serverless.md` — FaaS architecture patterns
- `05-database/sharding.md` — multi-tenancy DB partitioning
- `server-deployment-guide.md` — production VPS setup (this folder)

---

> **শেষ কথা:** "Cloud" কোনো জাদু নয় — অন্যের কম্পিউটার, অন্যের bill। সঠিক service model বেছে নিতে পারলে BD startup ৳1000/মাসে শুরু করে গ্লোবাল scale এ পৌঁছাতে পারে। ভুল model বেছে নিলে প্রথম মাসেই credit card decline।
