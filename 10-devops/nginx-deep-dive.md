# 🌀 Nginx Deep Dive — মাস্টার-ওয়ার্কার থেকে প্রোডাকশন টিউনিং পর্যন্ত

> **Nginx** শুধু একটা "ওয়েব সার্ভার" নয় — এটা Bangladesh-এর প্রায় প্রতিটা সিরিয়াস স্টার্টআপের (Daraz, Pathao, bKash gateway, Foodpanda, Chaldal) সামনের গেট, লোড ব্যালেন্সার, TLS টার্মিনেটর, ক্যাশ লেয়ার, রেট লিমিটার এবং WAF।
> এই ডকুমেন্টে আমরা Nginx-কে সিনিয়র SRE/DevOps ইঞ্জিনিয়ারের চোখ দিয়ে দেখব — আর্কিটেকচার, ইভেন্ট লুপ, ডিরেক্টিভ সিম্যান্টিক্স, প্রোডাকশন প্যাটার্ন ও সব কুখ্যাত pitfalls সহ।

---

## 📖 সূচিপত্র

- [Nginx কী এবং কেন](#-nginx-কী-এবং-কেন)
- [আর্কিটেকচার: Master + Worker মডেল](#-আর্কিটেকচার-master--worker-মডেল)
- [Event Loop ও epoll](#-event-loop-ও-epoll)
- [Request Processing Phases](#-request-processing-phases)
- [Module আর্কিটেকচার](#-module-আর্কিটেকচার)
- [কনফিগারেশন ভাষা ও ডিরেক্টিভ কনটেক্সট](#-কনফিগারেশন-ভাষা-ও-ডিরেক্টিভ-কনটেক্সট)
- [Location ম্যাচিং অ্যালগরিদম](#-location-ম্যাচিং-অ্যালগরিদম)
- [Reverse Proxy ও Service Proxying](#-reverse-proxy-ও-service-proxying)
- [Load Balancing](#-load-balancing)
- [TLS/SSL সেটআপ](#-tlsssl-সেটআপ)
- [Caching](#-caching)
- [Rate Limiting ও Connection Control](#-rate-limiting-ও-connection-control)
- [Security Hardening](#-security-hardening)
- [Performance Tuning](#-performance-tuning)
- [Observability](#-observability)
- [Module Ecosystem](#-module-ecosystem)
- [Operations ও Zero-Downtime Reload](#-operations-ও-zero-downtime-reload)
- [প্রোডাকশন প্যাটার্ন (Full Configs)](#-প্রোডাকশন-প্যাটার্ন-full-configs)
- [Anti-patterns ও Pitfalls](#-anti-patterns-ও-pitfalls)
- [Nginx vs Apache (সংক্ষিপ্ত)](#-nginx-vs-apache-সংক্ষিপ্ত)
- [প্রোডাকশন চেকলিস্ট](#-প্রোডাকশন-চেকলিস্ট)

---

## 📌 Nginx কী এবং কেন

**Nginx** (উচ্চারণ: "engine-X") হলো একটি **event-driven**, **non-blocking**, **asynchronous** ওয়েব সার্ভার ও রিভার্স প্রক্সি। ২০০৪ সালে Igor Sysoev লিখেছিলেন **C10K problem** — এক মেশিন কীভাবে ১০,০০০+ সমান্তরাল কানেকশন হ্যান্ডেল করবে — সমাধানের জন্য।

### Nginx যা যা করে

| ভূমিকা | উদাহরণ |
|--------|--------|
| **Web Server** | স্ট্যাটিক ফাইল (HTML, CSS, JS, image) সার্ভ |
| **Reverse Proxy** | PHP-FPM, Node.js, Python WSGI-এর সামনে |
| **Load Balancer** | একাধিক upstream-এর মধ্যে ট্রাফিক ভাগ |
| **TLS Terminator** | SSL/TLS handshake শেষ, backend-এ plain HTTP |
| **API Gateway** | rate limit, auth_request, JWT validation |
| **Cache Layer** | proxy_cache, FastCGI cache (Varnish-এর বিকল্প) |
| **WAF Front** | ModSecurity / Coraza সামনে |
| **Streaming** | TCP/UDP load balancing (`stream` module), RTMP |

### কেন Nginx — Bangladesh কনটেক্সট

Daraz Bangladesh-এ ১১.১১ ক্যাম্পেইনের সময় প্রতি সেকেন্ডে ~৩০,০০০+ HTTPS রিকোয়েস্ট আসে। Apache prefork দিয়ে এটা করতে হলে ৩০,০০০ প্রসেস/থ্রেড লাগবে — RAM ফেটে যাবে। Nginx ৪-৮টি worker প্রসেসে সব কানেকশন event loop-এ ঘোরায় — RAM ৩০০-৫০০MB-এর মধ্যে থাকে।

```
Apache prefork (১ কানেকশন = ১ প্রসেস)        Nginx (১ worker = ১০K+ কানেকশন)
═══════════════════════════════════════      ═══════════════════════════════════
   10K connections                              10K connections
        │                                            │
        ▼                                            ▼
  ┌────────────┐                              ┌────────────┐
  │ 10K procs  │ ≈ 30 GB RAM                  │ 4 workers  │ ≈ 200 MB RAM
  │ (each 3MB) │ context-switch storm         │ (epoll)    │ no thread overhead
  └────────────┘                              └────────────┘
```

---

## 🏗️ আর্কিটেকচার: Master + Worker মডেল

Nginx চলে **একটি master প্রসেস** এবং **একাধিক worker প্রসেস** নিয়ে।

```
                     Nginx Process Tree
                     ══════════════════

                  ┌──────────────────────┐
                  │   master (root)       │
                  │   - read config       │
                  │   - bind listening    │
                  │     sockets           │
                  │   - manage workers    │
                  │   - handle signals    │
                  └──────────┬───────────┘
                             │ fork()
        ┌────────────┬───────┼───────┬────────────┐
        ▼            ▼       ▼       ▼            ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
   │worker 0 │ │worker 1 │ │worker 2 │ │worker 3 │  (nginx user)
   │ epoll() │ │ epoll() │ │ epoll() │ │ epoll() │
   │ 10K conn│ │ 10K conn│ │ 10K conn│ │ 10K conn│
   └─────────┘ └─────────┘ └─────────┘ └─────────┘
        │           │           │           │
        └───────────┴───────────┴───────────┘
                    │
              shared listening
              socket (fd=6, 7)
              port 80, 443
```

### Master প্রসেসের দায়িত্ব

- কনফিগ পড়া, syntax-check (`nginx -t`)।
- privileged port (80/443) bind করা — root লাগে।
- worker fork করে privilege drop (setuid → `nginx` user)।
- signal handling (HUP=reload, USR1=reopen log, USR2=binary upgrade, QUIT=graceful shutdown)।
- worker crash হলে নতুন worker spawn।

### Worker প্রসেসের দায়িত্ব

- accept(), read(), write() — সব I/O।
- TLS handshake, gzip, proxy_pass, cache lookup — সব এখানে।
- কোনো রিকোয়েস্ট processing-এ master জড়িত নয় — ডেটা পাথ পুরোপুরি worker-এ।

### One Worker per CPU Core

```nginx
# nginx.conf — main context
worker_processes auto;          # = nproc (CPU cores)
worker_cpu_affinity auto;       # প্রতি worker একটা core-এ pin
worker_rlimit_nofile 65535;     # প্রতি worker fd limit
```

8-core মেশিনে `auto` মানে ৮টা worker। প্রতিটা worker আলাদা core-এ চলে — কোনো কন্টেক্সট সুইচ ওভারহেড নেই, CPU cache hot থাকে।

### Shared Listening Sockets — দুটো কৌশল

**সমস্যা:** ৪টা worker একসাথে `accept()` করতে গেলে "thundering herd" — সবাই জাগে কিন্তু একজন কানেকশন পায়।

#### ১. accept_mutex (পুরনো ডিফল্ট, এখন off)

```nginx
events {
    accept_mutex on;             # একসাথে শুধু একটা worker accept করবে
    accept_mutex_delay 500ms;
}
```
সমস্যা: high-concurrency-তে একজন worker সব কানেকশন নিয়ে নেয়, অন্যরা idle।

#### ২. SO_REUSEPORT (Linux 3.9+, recommended)

```nginx
http {
    server {
        listen 80 reuseport;     # প্রতিটা worker আলাদা socket queue পায়
        listen 443 ssl http2 reuseport;
        ...
    }
}
```
কার্নেল level-এ কানেকশন distribute করে, lock-free, CPU cache-friendly। Daraz API gateway-তে এই অপশন on করলে ~১৫% latency কমে।

### Memory Footprint

```bash
$ ps -eo pid,rss,cmd | grep nginx
   1234   2400 nginx: master process /usr/sbin/nginx
   1235  35840 nginx: worker process    # 35 MB
   1236  35712 nginx: worker process
   1237  36096 nginx: worker process
   1238  35968 nginx: worker process
```
মোট ~১৪০ MB — Apache prefork-এ এই কাজ করতে ৩-৪ GB লাগত।

---

## ⚡ Event Loop ও epoll

Nginx-এর core হলো একটা **event loop** যা Linux-এ `epoll`, FreeBSD-তে `kqueue`, Solaris-এ `event ports` ব্যবহার করে। এই বিষয়ে গভীর আলোচনা: [`20-operating-systems/io-models.md`](../20-operating-systems/io-models.md) ও [`20-operating-systems/kernel-networking-stack.md`](../20-operating-systems/kernel-networking-stack.md)।

```
        Nginx Worker Event Loop (simplified)
        ════════════════════════════════════

   ┌──────────────────────────────────────────┐
   │  while (1) {                              │
   │     timeout = ngx_event_find_timer();     │
   │     epoll_wait(epfd, events, n, timeout); │ ◄── kernel এ ঘুমায়
   │     for (i = 0; i < nready; i++) {        │
   │         ev = events[i].data.ptr;          │
   │         ev->handler(ev);                  │ ◄── read/write/accept
   │     }                                     │
   │     ngx_event_expire_timers();            │
   │     ngx_event_process_posted();           │
   │  }                                        │
   └──────────────────────────────────────────┘
```

### কেন epoll C10K-এ জেতে

| Model | 10K conn @ idle CPU | একটা ফাইল descriptor চেক |
|-------|---------------------|--------------------------|
| `select()` | ~৬০% CPU | O(n) — সব fd স্ক্যান |
| `poll()` | ~৪০% CPU | O(n) |
| `epoll` (edge-triggered) | <১% CPU | O(1) — শুধু ready fd |

```nginx
events {
    use epoll;                   # Linux-এ ডিফল্ট, explicit ভালো
    worker_connections 10240;    # প্রতি worker max conn
    multi_accept on;             # একবারে যত accept সম্ভব
}
```

**মোট কানেকশন ক্ষমতা:** `worker_processes × worker_connections`। কিন্তু একটা reverse proxy রিকোয়েস্ট ২ কানেকশন খায় (client + upstream), তাই কার্যকর সংখ্যা অর্ধেক।

### Non-blocking everywhere

Nginx-এ disk I/O পর্যন্ত block হতে পারে — sendfile, aio thread pool দিয়ে ঠিক করা হয়:

```nginx
http {
    sendfile on;                 # zero-copy: kernel → socket
    tcp_nopush on;               # MSS-পূর্ণ packet পাঠায়
    aio threads;                 # disk read async (thread pool)
    aio_write on;
    directio 4m;                 # বড় ফাইলে page-cache bypass
}
```

---

## 🔁 Request Processing Phases

প্রতিটা HTTP রিকোয়েস্ট Nginx-এ **১১টা ফেজ** দিয়ে যায়। মডিউলগুলো নির্দিষ্ট ফেজে handler register করে।

```
Phase                     Handler উদাহরণ                Directive
═════════════════════════════════════════════════════════════════════════
1. POST_READ              realip                       set_real_ip_from
2. SERVER_REWRITE         rewrite (server-level)       rewrite ... break
3. FIND_CONFIG            location ম্যাচিং (internal)  -
4. REWRITE                rewrite (location-level)     rewrite ^/old /new
5. POST_REWRITE           internal                     -
6. PREACCESS              limit_req, limit_conn        limit_req zone=...
7. ACCESS                 allow/deny, auth_basic,      auth_request /verify
                          auth_request, auth_jwt
8. POST_ACCESS            satisfy logic                satisfy any
9. PRECONTENT             try_files, mirror            try_files $uri =404
10. CONTENT               proxy_pass, fastcgi_pass,    proxy_pass ...
                          static file, return         return 200 "ok"
11. LOG                   access_log, lua log_by_lua   access_log ...
═════════════════════════════════════════════════════════════════════════
```

এটা বুঝলে ডিবাগ অনেক সহজ — যেমন `auth_request` সবসময় `proxy_pass`-এর আগে চলে, আর `limit_req` সবার আগে। Daraz-এ কেউ যদি বুঝতে না পারে কেন rate limit জারি হলেও log-এ access_log আসছে, কারণ rate limit ফেজ-৬, log ফেজ-১১ — log সব রিকোয়েস্টের জন্যই হয়।

---

## 🧩 Module আর্কিটেকচার

Nginx-এ কোড ভাগ করা চারটে মূল ক্যাটাগরিতে:

| ক্যাটাগরি | কাজ | উদাহরণ |
|----------|-----|--------|
| **Core** | event loop, memory pool, config parser | `ngx_core_module` |
| **Event** | epoll/kqueue wrapper | `ngx_epoll_module` |
| **HTTP** | HTTP প্রোটোকল, location, proxy | `ngx_http_proxy_module`, `ngx_http_ssl_module` |
| **Mail** | SMTP/POP3/IMAP proxy | `ngx_mail_module` |
| **Stream** | TCP/UDP load balance | `ngx_stream_module` |

### Static vs Dynamic Modules

```bash
# Build-time static (default): nginx বাইনারিতে সরাসরি
$ ./configure --with-http_ssl_module --with-http_v2_module
$ make && make install

# Dynamic (1.9.11+): .so ফাইল, runtime load
$ ./configure --add-dynamic-module=/path/to/ngx_brotli
# nginx.conf:
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;
```

### `nginx -V` দিয়ে কী মডিউল আছে দেখা

```bash
$ nginx -V 2>&1 | tr ' ' '\n' | grep -E '^--with|^--add'
--with-http_ssl_module
--with-http_v2_module
--with-http_realip_module
--with-stream
--with-stream_ssl_module
--add-module=/build/ngx_brotli
```

---

## 📝 কনফিগারেশন ভাষা ও ডিরেক্টিভ কনটেক্সট

Nginx config একটা hierarchical context tree:

```
main (global)
├── events { ... }                    ◄── worker_connections, use
├── http { ... }                      ◄── HTTP-related সব
│   ├── upstream backend { ... }
│   ├── map $var $newvar { ... }
│   ├── geo $remote_addr $country { ... }
│   ├── limit_req_zone ...
│   ├── server { ... }                ◄── virtual host
│   │   ├── listen 443 ssl;
│   │   ├── server_name api.daraz.com.bd;
│   │   ├── location / { ... }        ◄── URI matching
│   │   │   ├── if ($arg_debug) { ... }   ⚠️ "if is evil"
│   │   │   ├── limit_req zone=api;
│   │   │   └── proxy_pass http://backend;
│   │   └── location @fallback { ... } ◄── named location
│   └── server { ... }
├── stream { ... }                    ◄── TCP/UDP
│   └── server { ... }
└── mail { ... }
```

### ডিরেক্টিভ inheritance

ডিরেক্টিভ ভিতরের কনটেক্সটে **inherit** হয় — যদি না সেখানে override করা হয়। কিন্তু কিছু ডিরেক্টিভ (যেমন `add_header`) ভিতরে পুনরায় declare করলে বাইরের সব হারিয়ে যায় — এটা প্রচুর বাগ তৈরি করে। সমাধান: `more_set_headers` (headers-more module) ব্যবহার।

### গুরুত্বপূর্ণ Variables

| Variable | অর্থ |
|----------|------|
| `$remote_addr` | client IP (TCP-level) |
| `$http_x_forwarded_for` | header value (যদি LB চেইনে থাকে) |
| `$realip_remote_addr` | realip module setup-এর পর actual client |
| `$host` | request Host header (lowercase) |
| `$uri` | normalized URI (rewrite-এর পরে) |
| `$request_uri` | original raw URI |
| `$args` / `$query_string` | query string |
| `$arg_<name>` | নির্দিষ্ট query param (`$arg_id`) |
| `$cookie_<name>` | cookie value |
| `$upstream_addr` | কোন backend serve করল |
| `$upstream_response_time` | backend latency |
| `$request_time` | total Nginx-এ সময় |
| `$upstream_cache_status` | HIT/MISS/EXPIRED/STALE/UPDATING/REVALIDATED |

### `map` দিয়ে ব্রাঞ্চিং (if-এর বিকল্প)

```nginx
http {
    # User-Agent → device type
    map $http_user_agent $device_type {
        default        desktop;
        ~*mobile       mobile;
        ~*tablet       tablet;
        ~*bot|crawler  bot;
    }

    # Country code (geoip2) → cache key suffix
    map $geoip2_country_code $cache_region {
        default     global;
        BD          bd;
        IN          in;
    }

    server {
        location / {
            proxy_cache_key "$scheme$host$request_uri$cache_region";
            proxy_set_header X-Device $device_type;
        }
    }
}
```

`map` evaluate হয় **lazily** — ব্যবহার না হলে cost zero।

### `geo` দিয়ে IP-based logic

```nginx
geo $is_internal {
    default         0;
    10.0.0.0/8      1;       # office network
    103.108.140.0/24 1;      # Daraz BD office
}

server {
    location /admin {
        if ($is_internal = 0) { return 403; }
        proxy_pass http://admin_backend;
    }
}
```

### Include patterns — config organization

```
/etc/nginx/
├── nginx.conf              # main, events, http opening
├── conf.d/
│   ├── ssl.conf            # global TLS settings
│   ├── gzip.conf
│   ├── log_format.conf
│   ├── upstreams.conf
│   └── maps.conf
├── snippets/
│   ├── proxy_params.conf
│   ├── ssl_params.conf
│   └── security_headers.conf
└── sites-available/
    ├── daraz.com.bd.conf
    ├── api.daraz.com.bd.conf
    └── admin.daraz.com.bd.conf
        → symlink to sites-enabled/
```

```nginx
# nginx.conf
http {
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 🎯 Location ম্যাচিং অ্যালগরিদম

`location` অনেকের কাছে magic — আসলে ম্যাচিং algorithm deterministic এবং **এই ক্রমে** চলে:

```
1. Exact match:  location = /path           ◄── সবচেয়ে আগে, পেলে stop
2. Longest prefix with ^~:                  ◄── পেলে stop, regex skip
3. Regex (in config order):
      location ~ \.php$        (case-sensitive)
      location ~* \.(jpg|png)$ (case-insensitive)
4. Longest prefix without ^~:               ◄── fallback
```

### একটা example ভেঙে দেখি

```nginx
server {
    location = /favicon.ico   { access_log off; expires 7d; }     # 1
    location = /              { return 301 /home; }               # 1

    location ^~ /static/      { expires 1y; }                     # 2

    location ~* \.(jpg|jpeg|png|gif|webp|css|js)$ {              # 3
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    location ~ \.php$ {                                           # 3
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }

    location /api/ {                                              # 4 (prefix)
        proxy_pass http://api_backend;
    }
    location / {                                                  # 4 (prefix)
        try_files $uri $uri/ /index.php?$args;
    }
}
```

**Edge cases:**
- `/static/logo.png` → matches `^~ /static/` → stop, **regex skip** (image regex চলবে না)। এটা সাবধান — যদি static-এর ভিতরে .php থাকত, fastcgi চলত না।
- `/api/users.json` → `/api/` prefix, কিন্তু regex `\.json` নেই। যদি থাকত, regex জিতত।
- `/index.php` → exact নেই, prefix নেই except `/`, regex `\.php$` ম্যাচ → fastcgi।

### `try_files` — static fallback pattern

```nginx
location / {
    try_files $uri $uri/ /index.php?$args;
    # 1. /about.html থাকলে serve
    # 2. /about/ ডিরেক্টরি থাকলে index serve
    # 3. কোনোটাই না থাকলে /index.php?$args (Laravel/WordPress front controller)
}
```

### Named location (`@`) — internal redirect target

```nginx
location / {
    try_files $uri @app;
}
location @app {
    proxy_pass http://node_app;
}
```
Named location বাইরে থেকে accessible না — শুধু `try_files`, `error_page`, `post_action` থেকে।

---

## 🔁 Reverse Proxy ও Service Proxying

এটা Nginx-এর **সবচেয়ে গুরুত্বপূর্ণ** ব্যবহার Bangladesh-এ — Pathao API, Foodpanda checkout, Chaldal cart — সব Nginx-এর পিছনে Node.js / PHP-FPM / Java backend।

### `proxy_pass` ও URI rewriting — কুখ্যাত গোল্ডেন রুল

```nginx
# CASE A — proxy_pass-এ URI আছে (trailing path/slash):
location /api/ {
    proxy_pass http://backend/;       # ◄── slash আছে
    # /api/users → backend GET /users   (prefix /api/ "চাঁছা" হয়)
}

# CASE B — proxy_pass-এ শুধু host (URI নেই):
location /api/ {
    proxy_pass http://backend;        # ◄── slash নেই
    # /api/users → backend GET /api/users  (full URI পাঠানো হয়)
}
```

⚠️ **এই একটা slash-এর ভুলে Pathao-তে once production outage হয়েছিল** — `/api/v1/orders` backend-এ `/v1/orders` হিসেবে যাচ্ছিল, backend 404।

```nginx
# ⚠️ regex location-এ proxy_pass-এ URI দেওয়া ILLEGAL:
location ~ ^/api/(.*)$ {
    proxy_pass http://backend/$1;     # ❌ Nginx error
    proxy_pass http://backend;        # ✅ ঠিক
}
```

### `proxy_set_header` — যা backend দরকার

```nginx
# /etc/nginx/snippets/proxy_params.conf
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;
proxy_set_header X-Request-ID      $request_id;     # 1.11.0+

# WebSocket / HTTP/1.1 keepalive support
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        $connection_upgrade;
```

```nginx
# Connection header WebSocket-aware করার map
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }
}
```

### Buffering — streaming-এর শত্রু

```nginx
location /api {
    proxy_buffering on;              # default — Nginx full response RAM/disk-এ নেয়
    proxy_buffer_size       8k;      # response header buffer
    proxy_buffers           16 16k;  # body buffer (16 × 16k = 256k)
    proxy_busy_buffers_size 32k;
    proxy_max_temp_file_size 1024m;  # বড় হলে disk-এ
}

# Server-Sent Events / streaming JSON / WebSocket-এ buffering off:
location /sse {
    proxy_buffering    off;
    proxy_cache        off;
    proxy_read_timeout 24h;
    add_header X-Accel-Buffering no;   # downstream proxy-কেও বলো
    proxy_pass http://node_sse;
}
```

Foodpanda-র live order tracking SSE endpoint-এ প্রথমে buffering on ছিল — user-রা ৬০ সেকেন্ড পরে update দেখত। `proxy_buffering off` করার পর realtime।

### Timeouts

```nginx
proxy_connect_timeout 5s;       # backend TCP handshake
proxy_send_timeout    60s;      # request body পাঠানো
proxy_read_timeout    60s;      # response body পড়া (default 60)

# bKash payment webhook endpoint long-poll:
location /webhook {
    proxy_read_timeout 120s;
}
```

### Upstream keepalive (অনেকেই ভুলে যায়)

```nginx
upstream node_api {
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    keepalive 32;                # প্রতি worker × 32 idle conn cache
    keepalive_timeout 60s;
    keepalive_requests 1000;
}

server {
    location /api {
        proxy_http_version 1.1;          # ⚠️ বাধ্যতামূলক!
        proxy_set_header Connection "";  # ⚠️ "close" header strip করে
        proxy_pass http://node_api;
    }
}
```

⚠️ `proxy_http_version 1.1` ছাড়া keepalive কাজ করে না — প্রতিটা রিকোয়েস্টে নতুন TCP+TLS handshake। Pathao API-এর p99 latency 250ms থেকে 80ms-এ নেমেছিল শুধু এই একটা ঠিক করে।

### Retry — `proxy_next_upstream`

```nginx
location /api {
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries   3;
    proxy_next_upstream_timeout 10s;
    proxy_pass http://node_api;
}
```

⚠️ POST-এ retry ডিফল্টে off (idempotency)। জোর করতে: `proxy_next_upstream ... non_idempotent;` — কিন্তু payment-এ কখনো না!

### `proxy_intercept_errors` — custom error pages

```nginx
location /api {
    proxy_pass http://backend;
    proxy_intercept_errors on;
    error_page 502 503 504 /maintenance.html;
}
location = /maintenance.html {
    internal;
    root /var/www/static;
}
```

### WebSocket Proxy (Pathao real-time driver location)

```nginx
upstream socket_io {
    ip_hash;                     # sticky — same client → same node
    server 10.0.2.10:4000;
    server 10.0.2.11:4000;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name ws.pathao.com.bd;

    location /socket.io/ {
        proxy_pass http://socket_io;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host       $host;
        proxy_read_timeout 1h;       # idle WebSocket
        proxy_send_timeout 1h;
    }
}
```

### gRPC Proxying

```nginx
upstream grpc_backend {
    server 10.0.3.10:50051;
    server 10.0.3.11:50051;
}

server {
    listen 443 ssl http2;
    server_name grpc.bkash.internal;

    location / {
        grpc_pass grpc://grpc_backend;       # plain gRPC
        # grpcs_pass grpcs://...             # gRPC over TLS
        grpc_set_header X-Real-IP $remote_addr;
        error_page 502 = /error502grpc;
    }
    location = /error502grpc {
        internal;
        default_type application/grpc;
        add_header grpc-status 14;
        add_header grpc-message "unavailable";
        return 204;
    }
}
```

### FastCGI to PHP-FPM (Daraz/WordPress style)

```nginx
upstream php_fpm {
    server unix:/run/php/php8.2-fpm.sock;
    # বা TCP: server 10.0.4.10:9000;
    keepalive 16;
}

server {
    listen 443 ssl http2;
    server_name blog.daraz.com.bd;
    root /var/www/wordpress;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        # SECURITY: "/foo.php/bar" attack ঠেকাও
        try_files $uri =404;

        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass   php_fpm;
        fastcgi_index  index.php;
        include        fastcgi_params;

        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param  PATH_INFO       $fastcgi_path_info;
        fastcgi_param  HTTPS           on;

        # Buffer tuning (PHP response যদি বড় হয়)
        fastcgi_buffer_size       16k;
        fastcgi_buffers           32 16k;
        fastcgi_busy_buffers_size 32k;

        fastcgi_read_timeout 60s;

        # Keepalive PHP-FPM-এ
        fastcgi_keep_conn on;
    }

    # Sensitive files block
    location ~ /\.(ht|git|env) { deny all; }
    location ~ \.(yml|ini|log)$ { deny all; }
}
```

⚠️ `try_files $uri =404;` — এই লাইন না থাকলে attacker `/uploads/evil.jpg/x.php` দিয়ে মালিকহীন কোড execute করতে পারে। **ImageTragick/PHP-FPM history-র সবচেয়ে বড় বাগ।**

### `uwsgi_pass`, `scgi_pass`

Python (Django/Flask) উদাহরণ:
```nginx
upstream django {
    server unix:/run/uwsgi/django.sock;
}
location / {
    uwsgi_pass  django;
    include     uwsgi_params;
}
```
SCGI আজকাল প্রায় deprecated — Mailman/cgit-এ এখনো পাওয়া যায়।

---

## ⚖️ Load Balancing

`upstream` block। গভীর তত্ত্বের জন্য: [`03-system-design/load-balancing.md`](../03-system-design/load-balancing.md)।

### অ্যালগরিদম

```nginx
# 1. Round-robin (default) — সমান weight হলে rotation
upstream rr {
    server 10.0.1.10;
    server 10.0.1.11;
    server 10.0.1.12;
}

# 2. Weighted round-robin
upstream weighted {
    server 10.0.1.10 weight=3;       # ৩x বেশি load
    server 10.0.1.11 weight=1;
}

# 3. Least connections
upstream lc {
    least_conn;
    server 10.0.1.10;
    server 10.0.1.11;
}

# 4. IP hash — same client IP always same backend (session stick)
upstream session {
    ip_hash;
    server 10.0.1.10;
    server 10.0.1.11;
}

# 5. Generic hash + consistent (Ketama)
upstream cache {
    hash $request_uri consistent;    # consistent hashing — node add/remove-এ minimal disruption
    server cache1:11211;
    server cache2:11211;
    server cache3:11211;
}

# 6. Random with two_choices (P2C — power-of-two-choices)
upstream p2c {
    random two least_conn;
    server 10.0.1.10;
    server 10.0.1.11;
    server 10.0.1.12;
}

# 7. fair (3rd-party module — ngx_http_upstream_fair_module) — backend response time-ভিত্তিক
```

### Server পরামিটার

```nginx
upstream api {
    server 10.0.1.10 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.1.11 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.1.12 backup;            # primary সব ডাউন হলেই active
    server 10.0.1.13 down;              # maintenance
    server 10.0.1.14 weight=1 slow_start=30s;  # NGINX Plus only — gradual ramp-up

    keepalive 32;
}
```

### Health Check — Passive (ফ্রি)

`max_fails=3 fail_timeout=30s` মানে: কোনো server-এ ৩০ সেকেন্ডে ৩বার fail হলে next ৩০ সেকেন্ড ওই server-কে skip করো। এটা **passive** — শুধু real traffic দিয়ে হেলথ চেক হয়।

### Health Check — Active

- **NGINX Plus**: built-in `health_check uri=/health interval=5s fails=2 passes=2;`
- **OSS**: 3rd party `nginx_upstream_check_module` (Tengine থেকে)
- **Pragmatic alternative**: separate Consul/HAProxy front Nginx-এর

```nginx
# Plus example:
upstream api {
    zone api 64k;
    server 10.0.1.10;
    server 10.0.1.11;
}
server {
    location @hc {
        health_check uri=/health interval=5s match=api_ok;
        proxy_pass http://api;
    }
    match api_ok {
        status 200;
        body ~ '"status":"ok"';
    }
}
```

### Session Persistence

```nginx
# OSS: ip_hash (NAT-এর পিছনে অসম)
upstream s { ip_hash; server ...; }

# Plus: cookie-based sticky
upstream s {
    sticky cookie srv_id expires=1h domain=.daraz.com.bd path=/;
    server ...;
}
```

### Shared Memory Zone (Plus)

```nginx
upstream api {
    zone api_zone 64k;          # workers এর মধ্যে state share
    server 10.0.1.10;
    ...
}
```
OSS-এ প্রতিটা worker আলাদা upstream state রাখে — passive health check inconsistent হতে পারে।

---

## 🔐 TLS/SSL সেটআপ

বিস্তারিত: [`14-networking/tls-ssl.md`](../14-networking/tls-ssl.md)।

### Minimal modern TLS

```nginx
# /etc/nginx/snippets/ssl_params.conf

ssl_certificate     /etc/letsencrypt/live/daraz.com.bd/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/daraz.com.bd/privkey.pem;

# Mozilla "Intermediate" profile (2024)
ssl_protocols       TLSv1.2 TLSv1.3;
ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;       # TLS 1.3-এ irrelevant; 1.2-এ client-এ ছেড়ে দাও

ssl_session_cache    shared:SSL:50m;  # ৫০MB ≈ ২০০K sessions
ssl_session_timeout  1d;
ssl_session_tickets  off;             # forward secrecy (key rotation কঠিন হলে off)

# OCSP stapling — client-কে CA-তে যেতে দিতে হয় না
ssl_stapling         on;
ssl_stapling_verify  on;
ssl_trusted_certificate /etc/letsencrypt/live/daraz.com.bd/chain.pem;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# HSTS (১ বছর, subdomain সহ, preload)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# DH params (TLS 1.2 ECDHE-RSA-এর জন্য — TLS 1.3-এ লাগে না)
ssl_dhparam /etc/nginx/dhparam.pem;   # openssl dhparam -out ... 2048
```

### Server block

```nginx
server {
    listen 443 ssl http2 reuseport;
    listen [::]:443 ssl http2 reuseport;
    server_name daraz.com.bd www.daraz.com.bd;

    include snippets/ssl_params.conf;
    include snippets/security_headers.conf;
    ...
}

# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name daraz.com.bd www.daraz.com.bd;
    return 301 https://$host$request_uri;
}
```

### HTTP/3 / QUIC (Nginx 1.25+)

```nginx
server {
    listen 443 ssl reuseport;        # TLS over TCP (HTTP/1.1, /2)
    listen 443 quic reuseport;       # QUIC over UDP
    http2 on;
    http3 on;

    add_header Alt-Svc 'h3=":443"; ma=86400';   # browser-কে HTTP/3 জানাও

    ssl_certificate ...;
    ssl_certificate_key ...;
    ssl_early_data on;               # 0-RTT (replay risk মাথায় রাখুন)
}
```
UDP 443 firewall-এ open রাখতে হবে। DigitalOcean BD-তে cloud firewall-এ rule add করুন।

### mTLS (mutual TLS) — bKash internal services

```nginx
server {
    listen 443 ssl;
    server_name internal-api.bkash.local;

    ssl_certificate     /etc/ssl/server.crt;
    ssl_certificate_key /etc/ssl/server.key;

    ssl_client_certificate /etc/ssl/client-ca.crt;  # CA যেটা client cert sign করে
    ssl_verify_client      on;                      # বাধ্যতামূলক client cert
    ssl_verify_depth       2;

    location / {
        proxy_set_header X-Client-DN     $ssl_client_s_dn;
        proxy_set_header X-Client-Verify $ssl_client_verify;
        proxy_pass http://java_backend;
    }
}
```

### Let's Encrypt — certbot + auto-renewal

```bash
# Initial issue (webroot mode, Nginx already running)
$ sudo certbot certonly --webroot -w /var/www/html \
    -d daraz.com.bd -d www.daraz.com.bd \
    --email ops@daraz.com.bd --agree-tos --no-eff-email

# Renewal hook — renew হলে Nginx reload (zero-downtime)
$ cat /etc/letsencrypt/renewal-hooks/deploy/nginx-reload.sh
#!/bin/sh
nginx -t && nginx -s reload

$ sudo systemctl enable --now certbot.timer
```

### TLS hardening চেকলিস্ট

- ✅ TLS 1.0/1.1 disable (PCI-DSS requirement)
- ✅ Weak ciphers (RC4, 3DES, NULL) disable
- ✅ HSTS preload (chrome list-এ submit)
- ✅ OCSP stapling
- ✅ Test: `testssl.sh daraz.com.bd` ও SSL Labs A+ score
- ✅ certificate expiry alert (Prometheus blackbox exporter বা CloudFlare expiry monitor)

---

## 💾 Caching

Nginx-এর caching CDN-এর কাজ ৭০-৮০% কভার করে — বিশেষ করে **microcaching** Daraz product list page-এ।

### `proxy_cache_path` সেটআপ

```nginx
http {
    # /var/cache/nginx-এ ১০ GB cache, key zone 1GB ≈ 8M keys
    proxy_cache_path /var/cache/nginx/api
        levels=1:2
        keys_zone=api_cache:100m
        max_size=10g
        inactive=60m
        use_temp_path=off;

    proxy_cache_key "$scheme$request_method$host$request_uri";

    # cache HIT/MISS header (debug)
    add_header X-Cache-Status $upstream_cache_status always;
}

server {
    location /api/products {
        proxy_cache              api_cache;
        proxy_cache_valid        200 302  10m;
        proxy_cache_valid        404      1m;
        proxy_cache_lock         on;          # thundering herd ঠেকাও
        proxy_cache_lock_timeout 5s;
        proxy_cache_use_stale    error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_revalidate   on;          # If-Modified-Since/ETag

        # auth/cookie থাকলে bypass
        proxy_cache_bypass $cookie_session $http_authorization;
        proxy_no_cache     $cookie_session $http_authorization;

        proxy_pass http://api_backend;
    }
}
```

| Directive | কাজ |
|-----------|-----|
| `levels=1:2` | hash-এর ১ ও ২ char দিয়ে subdir (`/a/bc/abc...`) — directory-তে file বেশি হলে FS slow |
| `keys_zone` | shared memory: keys+metadata, **content নয়** |
| `max_size` | disk-এ max content size (LRU eviction) |
| `inactive` | এই সময়ে access না হলে delete |
| `proxy_cache_lock` | একই URL-এ multiple request → শুধু একটা backend-এ যায় |
| `use_stale` | backend down হলে old cache serve |
| `background_update` | stale serve, async revalidate |

### Microcaching — Daraz product page

```nginx
location /product/ {
    proxy_cache product_cache;
    proxy_cache_valid 200 1s;     # ১ সেকেন্ড!
    proxy_cache_lock on;
    proxy_pass http://app;
}
```
১১.১১ ক্যাম্পেইনে একই product URL-এ ১০,০০০ req/sec → backend-এ মাত্র ১ req/sec যায়। **৯৯.৯৯% cache hit ratio**।

### FastCGI Caching (PHP-FPM-এর সামনে)

```nginx
fastcgi_cache_path /var/cache/nginx/fcgi levels=1:2 keys_zone=wp:100m max_size=2g inactive=1h;
fastcgi_cache_key "$scheme$request_method$host$request_uri";

server {
    set $skip_cache 0;
    if ($request_method = POST) { set $skip_cache 1; }
    if ($query_string != "")    { set $skip_cache 1; }
    if ($http_cookie ~* "wordpress_logged_in|wp-postpass|comment_author") { set $skip_cache 1; }
    if ($request_uri ~* "/wp-admin/|/wp-login\.php|/cart/|/checkout/") { set $skip_cache 1; }

    location ~ \.php$ {
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache     $skip_cache;
        fastcgi_cache        wp;
        fastcgi_cache_valid  200 60m;
        fastcgi_cache_lock   on;
        ...
    }
}
```

### Cache Purging

- **OSS**: `proxy_cache_path` directory delete করে restart, বা [`ngx_cache_purge`](https://github.com/FRiCKLE/ngx_cache_purge) module
- **Plus**: `proxy_cache_purge` directive built-in
- **Pragmatic**: short TTL + `proxy_cache_revalidate` — purge আর দরকার হয় না

```nginx
# OSS purge endpoint (internal শুধু)
location ~ /purge(/.*) {
    allow 10.0.0.0/8;
    deny  all;
    proxy_cache_purge api_cache "$scheme$request_method$host$1";
}
```

---

## 🚦 Rate Limiting ও Connection Control

### `limit_req_zone` — token bucket algorithm

```nginx
http {
    # IP-based: প্রতি IP-তে 10 req/sec, zone size 10MB ≈ 160K IPs
    limit_req_zone $binary_remote_addr zone=api_rl:10m rate=10r/s;

    # User-based (JWT subject):
    limit_req_zone $jwt_claim_sub zone=user_rl:10m rate=100r/m;
}

server {
    location /api {
        limit_req zone=api_rl burst=20 nodelay;
        # rate=10r/s, burst=20:
        #   - bucket capacity 20
        #   - refill 10 token/sec
        #   - nodelay: burst-এর মধ্যে instant pass; burst exceed → 503
        # nodelay ছাড়া: burst queue-এ delay
        limit_req_status 429;   # default 503; 429 better
        proxy_pass http://api;
    }
}
```

### Token Bucket আন্ডার-দ্য-হুড

```
Bucket capacity = burst (20 tokens)
Refill rate     = rate  (10/sec → প্রতি 100ms ১টা token)

Request আসে → bucket-এ token আছে?
   ├── হ্যাঁ: token কমাও, pass
   └── না : nodelay হলে 429; নাহলে wait until refill

burst=20 nodelay → momentarily 20 req allow, পরে 10/s steady
```

### `limit_conn_zone` — concurrent connection cap

```nginx
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
limit_conn_zone $server_name        zone=conn_per_server:10m;

server {
    limit_conn conn_per_server 10000;     # সব IP মিলিয়ে এক server-এ ১০K
    location / {
        limit_conn conn_per_ip 20;        # প্রতি IP ২০ conn
    }
    location /download/ {
        limit_conn conn_per_ip 2;         # download slot সীমিত
        limit_rate 1m;                    # 1 MB/s per conn
        limit_rate_after 10m;             # প্রথম 10MB full speed (small file fast)
    }
}
```

### একাধিক zone একই location-এ

```nginx
location /api/login {
    limit_req zone=api_rl   burst=5  nodelay;
    limit_req zone=login_rl burst=3  nodelay;     # brute-force protection
    proxy_pass http://auth;
}
```

---

## 🛡️ Security Hardening

বিস্তারিত header guide: [`11-security/security-headers.md`](../11-security/security-headers.md)।

### Headers (snippet)

```nginx
# /etc/nginx/snippets/security_headers.conf
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options    "nosniff" always;
add_header X-Frame-Options           "SAMEORIGIN" always;
add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
add_header Permissions-Policy        "geolocation=(), microphone=(), camera=()" always;
add_header Content-Security-Policy   "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.google-analytics.com" always;
add_header Cross-Origin-Opener-Policy   "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
```

⚠️ `always` flag বাদ দিলে ৪xx/৫xx response-এ header missing হয়।

### Server tokens

```nginx
http {
    server_tokens off;             # "Server: nginx" থেকে version remove
    # Full hide (headers-more module):
    more_clear_headers 'Server';
    more_set_headers 'Server: BD-API';
}
```

### Slow loris ও body attacks

```nginx
client_body_timeout      10s;
client_header_timeout    10s;
send_timeout             10s;
client_max_body_size     20m;       # upload limit
client_body_buffer_size  128k;
large_client_header_buffers 4 8k;
```

### IP allow/deny + GeoIP2

```nginx
# Block known bad ASNs/countries
load_module modules/ngx_http_geoip2_module.so;

http {
    geoip2 /usr/share/geoip/GeoLite2-Country.mmdb {
        $geoip2_country_code source=$realip_remote_addr country iso_code;
    }

    map $geoip2_country_code $blocked_country {
        default 0;
        RU      1;          # uncomment as needed
        KP      1;
    }

    server {
        if ($blocked_country) { return 403; }

        location /admin/ {
            allow 103.108.140.0/24;     # office
            allow 10.0.0.0/8;
            deny  all;
        }
    }
}
```

### `auth_request` — external auth subrequest

```nginx
location /api/ {
    auth_request /_auth;
    auth_request_set $user $upstream_http_x_user;
    proxy_set_header X-User $user;
    proxy_pass http://api;
}
location = /_auth {
    internal;
    proxy_pass              http://auth_service/verify;
    proxy_pass_request_body off;
    proxy_set_header        Content-Length "";
    proxy_set_header        X-Original-URI $request_uri;
    proxy_set_header        Authorization $http_authorization;
}
```
auth service 200 দিলে main request continue, 401/403 দিলে stop। JWT validation, OAuth introspection — সব এই pattern-এ।

### ModSecurity / Coraza WAF

```nginx
# libmodsecurity + ngx_http_modsecurity
load_module modules/ngx_http_modsecurity_module.so;

http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    # OWASP CRS rules include
}
```
আধুনিক বিকল্প: **Coraza** (Go), **CrowdSec** (community-driven IP block list)।

### fail2ban integration

`/var/log/nginx/access.log`-এ 401/403/429 burst detect করে IP iptables-এ ban। `/etc/fail2ban/jail.d/nginx.conf`:

```ini
[nginx-limit-req]
enabled = true
filter  = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 10
findtime = 60
bantime  = 3600
```

---

## 🚀 Performance Tuning

### Worker tuning

```nginx
worker_processes      auto;
worker_cpu_affinity   auto;
worker_rlimit_nofile  65535;        # ulimit for nginx user

events {
    worker_connections 16384;       # = ulimit / 2 ≈
    use                epoll;
    multi_accept       on;
}
```

### File I/O

```nginx
http {
    sendfile      on;               # zero-copy
    tcp_nopush    on;               # full-MSS packet
    tcp_nodelay   on;               # disable Nagle (small packet)

    # Async file I/O (Linux native AIO)
    aio           threads;          # thread pool (default 32 threads)
    aio_write     on;
    directio      8m;               # large file: bypass page cache

    open_file_cache          max=10000 inactive=60s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;
}
```

`open_file_cache` ছাড়া প্রতিটা static request-এ `stat()` syscall — ১০K req/sec-এ measurable overhead।

### Compression

```nginx
# gzip
gzip              on;
gzip_vary         on;
gzip_proxied      any;
gzip_comp_level   5;            # 6+ CPU expensive, gain marginal
gzip_min_length   1024;
gzip_types        text/plain text/css application/json
                  application/javascript text/xml application/xml
                  application/xml+rss text/javascript image/svg+xml;

# Brotli (ngx_brotli module — better than gzip)
brotli            on;
brotli_comp_level 5;
brotli_types      text/plain text/css application/json
                  application/javascript image/svg+xml;

# Pre-compressed static files (build-time gzip/brotli)
gzip_static       on;
brotli_static     on;
```

webpack/Vite build-এ `.br`/`.gz` জেনারেট করো, Nginx serve করার সময় on-the-fly compression skip — CPU 30-40% কমে।

### Keepalive

```nginx
keepalive_timeout    65s;
keepalive_requests   1000;          # প্রতি conn-এ max requests

# upstream-এ:
upstream backend { keepalive 32; ... }
```

### 103 Early Hints (HTTP/2+, Nginx 1.25.4+)

```nginx
location / {
    early_hints 1;                  # 103 Early Hints with Link headers
    add_header Link "</css/app.css>; rel=preload; as=style";
    proxy_pass http://app;
}
```
Browser preconnect শুরু করে দেয় full response আসার আগেই।

### Linux sysctl (host tuning)

```bash
# /etc/sysctl.d/99-nginx.conf
net.core.somaxconn               = 65535       # listen backlog cap
net.core.netdev_max_backlog      = 16384
net.ipv4.tcp_max_syn_backlog     = 65535
net.ipv4.ip_local_port_range     = 1024 65535  # ephemeral ports
net.ipv4.tcp_tw_reuse            = 1           # TIME_WAIT reuse (client-side)
net.ipv4.tcp_fin_timeout         = 15
net.ipv4.tcp_keepalive_time      = 600
fs.file-max                      = 2097152

# Apply: sysctl -p /etc/sysctl.d/99-nginx.conf
```

```nginx
# matching listen backlog
listen 443 ssl http2 backlog=65535 reuseport;
```

---

## 📊 Observability

### Custom log format

```nginx
log_format main_ext '$remote_addr - $remote_user [$time_iso8601] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time '
                    'ua=$upstream_addr cs=$upstream_cache_status '
                    'xff="$http_x_forwarded_for" rid=$request_id';

log_format json_combined escape=json
'{'
   '"ts":"$time_iso8601",'
   '"remote":"$remote_addr",'
   '"xff":"$http_x_forwarded_for",'
   '"method":"$request_method",'
   '"uri":"$request_uri",'
   '"status":$status,'
   '"bytes":$body_bytes_sent,'
   '"rt":$request_time,'
   '"urt":"$upstream_response_time",'
   '"ua":"$upstream_addr",'
   '"cache":"$upstream_cache_status",'
   '"host":"$host",'
   '"rid":"$request_id"'
'}';

access_log /var/log/nginx/access.log json_combined buffer=64k flush=5s;
error_log  /var/log/nginx/error.log  warn;
```

`buffer=64k flush=5s` — disk I/O batch করে high-traffic-এ throughput বাড়ায়।

### `stub_status` + Prometheus

```nginx
server {
    listen 127.0.0.1:8080;
    location /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
$ curl localhost:8080/stub_status
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

→ `nginx-prometheus-exporter` দিয়ে scrape:
```bash
nginx-prometheus-exporter -nginx.scrape-uri=http://localhost:8080/stub_status
```

আরও বিস্তারিত metrics: **VTS module** (`nginx-module-vts`) — প্রতি upstream/server/cache zone-এ আলাদা stat, JSON/Prometheus endpoint।

### OpenTelemetry tracing

```nginx
# nginx-otel module (1.24+ official)
load_module modules/ngx_otel_module.so;

http {
    otel_exporter {
        endpoint otel-collector.observability:4317;
    }
    otel_service_name nginx-edge;
    otel_trace on;
    otel_trace_context propagate;     # W3C traceparent

    server {
        location /api {
            otel_span_name api_proxy;
            otel_span_attr deployment.env "production";
            proxy_pass http://api;
        }
    }
}
```

### Real-time tail patterns

```bash
# Slow requests (> 1s)
tail -f /var/log/nginx/access.log | awk '$NF > 1.0'

# Status code distribution (last 1000)
tail -n 1000 access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Top IPs hitting
tail -n 10000 access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head

# Real-time error_log filter
tail -f error.log | grep -v "client closed connection"
```

---

## 🧰 Module Ecosystem

### OpenResty / `ngx_http_lua_module`

OpenResty Bangladesh-এ অনেক fintech (bKash gateway-এর কাছাকাছি কোম্পানি) ব্যবহার করে, কারণ Lua দিয়ে complex auth/routing/transformation Nginx-এর ভিতরেই করা যায় — backend hop save।

```nginx
location /api {
    access_by_lua_block {
        local jwt = require "resty.jwt"
        local token = ngx.var.http_authorization:gsub("Bearer ", "")
        local ok = jwt:verify("secret", token)
        if not ok.verified then
            ngx.status = 401
            ngx.say('{"error":"invalid token"}')
            return ngx.exit(401)
        end
        ngx.req.set_header("X-User-Id", ok.payload.sub)
    }
    proxy_pass http://api;
}
```

Dynamic upstream Lua দিয়ে: Consul/etcd থেকে service discovery।

### njs (JavaScript)

```nginx
js_path "/etc/nginx/njs/";
js_import main from main.js;

server {
    location /transform {
        js_content main.transform;
    }
}
```

### অন্যান্য মডিউল

| মডিউল | কাজ |
|-------|-----|
| `ngx_http_geoip2_module` | MaxMind DB থেকে country/ASN |
| `ngx_brotli` | Brotli compression |
| `headers-more-nginx-module` | header add/clear/replace strict |
| `nginx-rtmp-module` | RTMP live streaming (BD bKash live training) |
| `ngx_http_image_filter_module` | resize/crop/rotate (build-in but `--with`) |
| `ngx_http_substitution_filter` | response body string replace |
| `nginx-vts` | virtual host traffic status |

### NGINX Unit

Nginx team-এর আধুনিক **app server** (Nginx না, ভাই-প্রজেক্ট)। PHP/Python/Node/Java/Go সব এক runtime-এ, REST API-দ্বারা dynamic config। প্রোডাকশন adoption এখনো কম, কিন্তু polyglot stack-এ চমৎকার।

---

## 🛠️ Operations ও Zero-Downtime Reload

### Daily commands

```bash
nginx -t                    # config syntax check (always run before reload!)
nginx -T                    # dump full effective config (include resolution সহ)
nginx -V                    # build flags ও version
nginx -s reload             # graceful reload (SIGHUP)
nginx -s reopen             # reopen log files (SIGUSR1) — log rotation-এর পর
nginx -s quit               # graceful shutdown (SIGQUIT)
nginx -s stop               # immediate stop (SIGTERM)
```

### `reload` কীভাবে কাজ করে

```
1. master নতুন config পড়ে → ভুল হলে old config চলতে থাকে
2. master নতুন worker spawn করে নতুন config নিয়ে
3. পুরনো worker-গুলোকে "shutting down" mark — নতুন কানেকশন accept করে না
4. পুরনো worker active কানেকশন শেষ হলে exit
5. কোনো কানেকশন drop হয় না — true zero-downtime
```

### Binary upgrade (nginx version বদল drop ছাড়া)

```bash
# 1. নতুন nginx বাইনারি install
$ apt install nginx=1.26.x

# 2. master-কে USR2 — নতুন master fork (পুরনো master-এর child)
$ kill -USR2 $(cat /var/run/nginx.pid)

# 3. পুরনো workers-কে WINCH — graceful shutdown (নতুন conn accept বন্ধ)
$ kill -WINCH $(cat /var/run/nginx.pid.oldbin)

# 4. test ঠিক থাকলে পুরনো master-কেও QUIT
$ kill -QUIT $(cat /var/run/nginx.pid.oldbin)

# কিছু ভুল হলে rollback:
$ kill -HUP  $(cat /var/run/nginx.pid.oldbin)   # পুরনো worker-দের আবার জাগাও
$ kill -QUIT $(cat /var/run/nginx.pid)          # নতুন master kill
```

### Log rotation

```
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 nginx adm
    sharedscripts
    postrotate
        [ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

### Docker / Kubernetes

```dockerfile
# Dockerfile — multi-stage with custom modules
FROM nginx:1.27-alpine AS base
COPY nginx.conf            /etc/nginx/nginx.conf
COPY conf.d/               /etc/nginx/conf.d/
COPY snippets/             /etc/nginx/snippets/

# Health probe-friendly:
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s \
  CMD wget -q --spider http://127.0.0.1/healthz || exit 1
```

```yaml
# k8s ingress-nginx — Daraz BD setup
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Edge: bd-dhaka-1";
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.daraz.com.bd]
      secretName: api-tls
  rules:
    - host: api.daraz.com.bd
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port: { number: 80 }
```

---

## 🏭 প্রোডাকশন প্যাটার্ন (Full Configs)

### ১. WordPress + PHP-FPM (Daraz Blog Bangladesh)

```nginx
upstream php_fpm {
    server unix:/run/php/php8.2-fpm.sock;
    keepalive 16;
}

fastcgi_cache_path /var/cache/nginx/wp levels=1:2 keys_zone=wp:100m max_size=2g inactive=60m use_temp_path=off;

server {
    listen 443 ssl http2 reuseport;
    server_name blog.daraz.com.bd;
    root /var/www/wordpress;
    index index.php;

    include snippets/ssl_params.conf;
    include snippets/security_headers.conf;

    set $skip_cache 0;
    if ($request_method = POST)                                            { set $skip_cache 1; }
    if ($query_string != "")                                               { set $skip_cache 1; }
    if ($http_cookie ~* "wordpress_logged_in|wp-postpass|comment_author")  { set $skip_cache 1; }
    if ($request_uri ~* "/wp-(admin|login)|/cart/|/checkout/|/my-account/"){ set $skip_cache 1; }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~* \.(jpg|jpeg|png|gif|webp|svg|ico|css|js|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass            php_fpm;
        fastcgi_index           index.php;
        include                 fastcgi_params;
        fastcgi_param           SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param           HTTPS on;
        fastcgi_keep_conn       on;

        fastcgi_cache_bypass    $skip_cache;
        fastcgi_no_cache        $skip_cache;
        fastcgi_cache           wp;
        fastcgi_cache_valid     200 60m;
        fastcgi_cache_lock      on;
        add_header X-FCGI-Cache $upstream_cache_status;
    }

    location ~ /\.(ht|git|env)  { deny all; }
    location ~ /xmlrpc\.php     { deny all; }   # brute force vector
    location ~ /wp-login\.php {
        limit_req zone=login_rl burst=3 nodelay;
        try_files $uri =404;
        fastcgi_pass php_fpm;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### ২. Node.js API + Health-check fallback (Pathao Rides API)

```nginx
upstream pathao_api {
    least_conn;
    server 10.0.1.10:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 max_fails=3 fail_timeout=30s backup;
    keepalive 64;
}

limit_req_zone $binary_remote_addr zone=api_rl:20m rate=50r/s;

server {
    listen 443 ssl http2 reuseport;
    server_name api.pathao.com.bd;

    include snippets/ssl_params.conf;
    include snippets/security_headers.conf;

    access_log /var/log/nginx/pathao-api.access.log json_combined buffer=64k flush=5s;
    error_log  /var/log/nginx/pathao-api.error.log  warn;

    client_max_body_size 5m;

    # Health endpoint (no auth, no log noise)
    location = /healthz {
        access_log off;
        return 200 "ok\n";
    }

    location / {
        limit_req         zone=api_rl burst=100 nodelay;
        limit_req_status  429;

        proxy_http_version 1.1;
        proxy_set_header   Connection "";
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Request-ID      $request_id;

        proxy_connect_timeout 5s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;

        proxy_next_upstream         error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries   2;
        proxy_next_upstream_timeout 10s;

        proxy_pass http://pathao_api;
    }
}
```

### ৩. Static SPA (React) — Chaldal frontend

```nginx
server {
    listen 443 ssl http2 reuseport;
    server_name chaldal.com www.chaldal.com;
    root /var/www/chaldal-spa;
    index index.html;

    include snippets/ssl_params.conf;
    include snippets/security_headers.conf;

    # Hashed assets — immutable, 1 year
    location /static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # index.html — never cache (deploy-এর সাথে সাথে নতুন UI)
    location = /index.html {
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API → backend (same-origin দিয়ে CORS এড়ানো)
    location /api/ {
        proxy_pass http://chaldal_api/;
        include snippets/proxy_params.conf;
    }
}
```

### ৪. API Gateway with JWT auth (Foodpanda microservices)

```nginx
upstream auth_svc      { server 10.0.10.10:8000; keepalive 16; }
upstream order_svc     { server 10.0.10.11:8001; server 10.0.10.12:8001; keepalive 32; }
upstream restaurant_svc{ server 10.0.10.13:8002; keepalive 32; }
upstream payment_svc   { server 10.0.10.14:8003; keepalive 16; }

limit_req_zone $binary_remote_addr        zone=ip_rl:10m   rate=30r/s;
limit_req_zone $http_x_api_key            zone=key_rl:10m  rate=200r/s;

server {
    listen 443 ssl http2 reuseport;
    server_name gateway.foodpanda.com.bd;
    include snippets/ssl_params.conf;
    include snippets/security_headers.conf;

    # Internal auth subrequest
    location = /_jwt_verify {
        internal;
        proxy_pass              http://auth_svc/verify;
        proxy_pass_request_body off;
        proxy_set_header        Content-Length "";
        proxy_set_header        X-Original-URI $request_uri;
        proxy_set_header        Authorization $http_authorization;
    }

    location /v1/orders/ {
        auth_request     /_jwt_verify;
        auth_request_set $user_id $upstream_http_x_user_id;
        proxy_set_header X-User-Id $user_id;

        limit_req zone=ip_rl  burst=60 nodelay;
        limit_req zone=key_rl burst=400 nodelay;

        proxy_pass http://order_svc/;
        include snippets/proxy_params.conf;
    }

    location /v1/restaurants/ {
        # Public endpoint — no auth, but cached
        proxy_cache       api_cache;
        proxy_cache_valid 200 30s;
        proxy_cache_lock  on;
        proxy_pass http://restaurant_svc/;
        include snippets/proxy_params.conf;
    }

    location /v1/payment/ {
        auth_request /_jwt_verify;
        # Payment-এ NEVER retry POST
        proxy_next_upstream off;
        proxy_pass http://payment_svc/;
        include snippets/proxy_params.conf;
    }
}
```

### ৫. Streaming large download (no buffering)

```nginx
location /downloads/ {
    proxy_buffering    off;
    proxy_request_buffering off;       # client → upstream streamed
    proxy_max_temp_file_size 0;
    proxy_read_timeout 1h;
    chunked_transfer_encoding on;

    limit_conn conn_per_ip 2;
    limit_rate_after 10m;
    limit_rate 5m;

    proxy_pass http://file_origin;
}
```

### ৬. Image proxy with resize

```nginx
location ~* ^/img/(\d+)x(\d+)/(.+)$ {
    image_filter resize $1 $2;
    image_filter_jpeg_quality 85;
    image_filter_buffer 10M;
    proxy_pass http://image_origin/$3;
    expires 30d;
}
```

### ৭. mTLS internal proxy (bKash-style)

```nginx
server {
    listen 8443 ssl;
    server_name internal-pay.bkash.local;

    ssl_certificate     /etc/ssl/edge/server.crt;
    ssl_certificate_key /etc/ssl/edge/server.key;

    ssl_client_certificate /etc/ssl/edge/client-ca.crt;
    ssl_verify_client      on;
    ssl_verify_depth       2;

    location / {
        if ($ssl_client_verify != SUCCESS) { return 403; }
        proxy_set_header X-Client-CN     $ssl_client_s_dn_cn;
        proxy_set_header X-Client-Serial $ssl_client_serial;
        proxy_pass http://java_payment_backend;
        include snippets/proxy_params.conf;
    }
}
```

---

## ❌ Anti-patterns ও Pitfalls

### ১. Upstream keepalive-এ `proxy_http_version 1.1` ভুলে যাওয়া

```nginx
# ❌ keepalive কাজ করে না
upstream api { server x; keepalive 32; }
location / { proxy_pass http://api; }   # default HTTP/1.0, Connection: close

# ✅
location / {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_pass http://api;
}
```

### ২. "if is evil"

`if` location context-এ অপ্রত্যাশিত আচরণ করে — phase ordering, content handler reset। যা জানা যায় বুঝে use করো:
- ✅ `if` server context-এ allowed: `if ($host = old.com) { return 301 ... }`
- ❌ `if` location-এ `proxy_pass`/`rewrite`-এর সাথে — bug হবে।

```nginx
# ❌ ভুল
location / {
    if ($http_user_agent ~* bot) {
        proxy_pass http://bot_backend;     # silently broken
    }
    proxy_pass http://normal_backend;
}

# ✅ map দিয়ে
map $http_user_agent $backend_pool {
    default     normal_backend;
    ~*bot       bot_backend;
}
location / { proxy_pass http://$backend_pool; }
```

### ৩. Location regex order

```nginx
# ❌ /static/file.css — regex ঢুকে যায় যদি ^~ না থাকে
location /static/         { ... }
location ~ \.css$         { proxy_pass http://css_processor; }

# ✅
location ^~ /static/      { expires 1y; }    # stop, regex skip
```

### ৪. `proxy_pass` trailing slash গোলমাল

আগে দেখিয়েছি — `proxy_pass http://b/;` vs `proxy_pass http://b;` — URL rewrite আলাদা। **প্রতিবার test করো:**

```bash
$ curl -H "X-Debug: 1" -v http://api.daraz.com.bd/api/foo
# backend log-এ /foo নাকি /api/foo দেখো
```

### ৫. Auth'd content accidentally cache

```nginx
# ❌ logged-in user-এর dashboard cache হয়ে গেল
location / {
    proxy_cache c;
    proxy_pass http://app;
}

# ✅ Cookie/Authorization দেখলে bypass
location / {
    proxy_cache        c;
    proxy_cache_bypass $cookie_session $http_authorization;
    proxy_no_cache     $cookie_session $http_authorization;
    proxy_pass http://app;
}
```

### ৬. `limit_req` burst ছাড়া

```nginx
# ❌ rate=10r/s মানে literal 100ms-এ ১টা — user double-click-এ 503
limit_req zone=rl;

# ✅ burst সঙ্গে nodelay
limit_req zone=rl burst=20 nodelay;
```

### ৭. `add_header` inheritance ভাঙে

```nginx
server {
    add_header X-Frame-Options DENY always;
    location /api {
        add_header X-API-Version 1 always;   # ❌ X-Frame-Options এখানে missing!
    }
}
# ✅ snippet include করো প্রতিটা location-এ
```

### ৮. SSI / large `client_max_body_size` সব location-এ

Default `1m`। File upload endpoint `20m` দিলেও বাকি location-এ small রাখো — DoS surface কমে।

### ৯. DNS resolution কাজ না করা

```nginx
# ❌ upstream-এ hostname রাখলে nginx start-এ একবার resolve, IP বদলালে আর update না
upstream api { server api.internal:8000; }

# ✅ resolver + variable
resolver 10.0.0.2 valid=10s;
location / {
    set $upstream api.internal:8000;
    proxy_pass http://$upstream;
}
```

### ১০. error_log debug level production-এ

```nginx
error_log /var/log/nginx/error.log debug;   # ❌ disk fill, latency
error_log /var/log/nginx/error.log warn;    # ✅
```

---

## ⚔️ Nginx vs Apache (সংক্ষিপ্ত)

| বৈশিষ্ট্য | Nginx | Apache (prefork) |
|----------|-------|------------------|
| Architecture | Event-driven, async | Process/Thread-per-conn |
| ১০K conn RAM | ~১৫০ MB | ~৩-৬ GB |
| Static file | অসামান্য (sendfile) | ভালো |
| Dynamic (PHP) | FastCGI to PHP-FPM | mod_php (in-process) |
| `.htaccess` | ❌ নেই | ✅ আছে |
| Module reload | reload-এ | restart often |
| Config reload | zero-downtime | zero-downtime (graceful) |
| HTTP/3 | 1.25+ | mod_http3 (experimental) |
| Best for | reverse proxy, edge, static | shared-hosting, mod-heavy app |

বিস্তারিত comparison: `apache-deep-dive.md` (যদি থাকে)।

---

## ✅ প্রোডাকশন চেকলিস্ট

### Architecture
- [ ] `worker_processes auto;` ও `worker_cpu_affinity auto;`
- [ ] `worker_rlimit_nofile` ≥ ৬৫,৫৩৫ এবং ulimit-এ সমান
- [ ] `listen ... reuseport` multi-core load distribution-এর জন্য
- [ ] `events { use epoll; multi_accept on; worker_connections 16384; }`

### TLS
- [ ] TLS 1.0/1.1 disabled, শুধু 1.2 ও 1.3
- [ ] Mozilla Intermediate cipher list
- [ ] HSTS preload (`max-age=31536000; includeSubDomains; preload`)
- [ ] OCSP stapling on, `ssl_trusted_certificate` set
- [ ] HTTP/2 enabled; HTTP/3 if Nginx ≥ 1.25
- [ ] Let's Encrypt auto-renewal cron + reload hook tested
- [ ] `testssl.sh` ও SSL Labs A+

### Reverse Proxy
- [ ] `proxy_http_version 1.1;` + `proxy_set_header Connection "";` everywhere
- [ ] upstream-এ `keepalive` set
- [ ] `Host`, `X-Real-IP`, `X-Forwarded-For/Proto/Host` সব forwarded
- [ ] timeouts (connect/send/read) tuned per route
- [ ] WebSocket route-এ Upgrade/Connection header map
- [ ] `try_files $uri =404;` PHP location-এ (path traversal block)
- [ ] `proxy_next_upstream` POST-এ idempotent only

### Caching (যদি use করো)
- [ ] `proxy_cache_lock on;` থিনিং herd ঠেকাতে
- [ ] `proxy_cache_use_stale` + `background_update`
- [ ] auth/cookie বাইপাস rule
- [ ] cache HIT ratio Prometheus-এ monitor

### Security
- [ ] `server_tokens off;`
- [ ] সব security headers (`HSTS`, `X-Content-Type-Options`, `CSP`, `Frame-Options`, `Referrer-Policy`)
- [ ] `client_max_body_size`, `client_body_timeout`, `client_header_timeout` set
- [ ] sensitive path block (`.git`, `.env`, `.htaccess`)
- [ ] admin endpoint IP-allowlisted
- [ ] WAF (ModSecurity/Coraza) attack-prone path-এ
- [ ] `limit_req` ও `limit_conn` API/login-এ
- [ ] fail2ban integration for repeat 401/429

### Performance
- [ ] `sendfile on; tcp_nopush on; tcp_nodelay on;`
- [ ] `aio threads;` বড় file serve হলে
- [ ] `open_file_cache` enabled
- [ ] gzip + brotli + `_static` precompressed serve
- [ ] sysctl: `somaxconn`, `tcp_max_syn_backlog`, `ip_local_port_range`
- [ ] `listen ... backlog=` matched

### Observability
- [ ] JSON access log + `$request_time`, `$upstream_response_time`, `$upstream_addr`, `$upstream_cache_status`
- [ ] log buffer + flush configured
- [ ] `stub_status` exposed → Prometheus exporter
- [ ] OpenTelemetry tracing (W3C traceparent propagated)
- [ ] error_log level `warn` (debug only when needed)
- [ ] logrotate + USR1 hook
- [ ] alerts: 5xx rate, p99 latency, cert expiry, upstream down

### Operations
- [ ] `nginx -t` mandatory in CI before reload
- [ ] reload via `systemctl reload nginx` (zero-downtime)
- [ ] config under git, immutable rollout (blue/green Nginx hosts)
- [ ] runbook for: pull bad backend out, drain conns, binary upgrade
- [ ] disaster: spare config snapshot

---

## 📚 সারসংক্ষেপ

1. **Master + Worker** আর্কিটেকচার + **epoll** event loop = কম RAM-এ লাখো কানেকশন।
2. **Phases** বুঝে directive বসাও — ভুল phase-এ বসালে কাজই করবে না।
3. **Location matching** order মুখস্থ: `=` → `^~` → regex → prefix। `^~` স্ট্র্যাটেজিকভাবে static path-এ ব্যবহার করো।
4. **Reverse proxy-তে `proxy_http_version 1.1` + `Connection ""`** keepalive-এর জন্য *বাধ্যতামূলক*।
5. `proxy_pass` trailing-slash সিম্যান্টিক্স ভুল করলে production outage — সবসময় integration test।
6. **Microcaching (1s)** Daraz-style traffic-এ backend save করে magic-এর মতো।
7. **`if`** location-এ avoid করো — `map`/`try_files`/`return` দিয়ে কাজ চালাও।
8. **TLS** Mozilla intermediate, OCSP stapling, HSTS preload, HTTP/2 + HTTP/3।
9. **Rate limit** burst+nodelay + `429` status — 503 noise-এ user হারায়।
10. **`nginx -t`** ছাড়া কখনো reload করবে না। binary upgrade USR2/WINCH/QUIT sequence learn করে রাখো।

> Nginx শেখার শেষ নেই — কিন্তু এই ডকুমেন্টের patterns পেলে তুমি Bangladesh-এর top tech কোম্পানির edge layer চালাতে পারবে। ভালো DevOps মানে boring config — কোনো surprise নেই, কোনো 3 AM call নেই। 🌙
