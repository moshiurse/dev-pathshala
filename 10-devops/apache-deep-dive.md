# 🪶 Apache HTTP Server (httpd) — Deep Dive

> **"The number-two web server still runs half the internet — especially Bangladesh-এর shared hosting জগৎ।"**
>
> Nginx, Caddy, Envoy-এর যুগেও Apache কেন এখনও প্রাসঙ্গিক, কীভাবে কাজ করে, কীভাবে production-grade tune করবেন — সব কিছু এই ডকুমেন্টে।

---

## 📖 সূচিপত্র

- [Apache কী এবং কেন এখনও প্রাসঙ্গিক](#-apache-কী-এবং-কেন-এখনও-প্রাসঙ্গিক)
- [আর্কিটেকচার — MPM Deep Dive](#-আর্কিটেকচার--mpm-deep-dive)
- [Request Lifecycle ও Hooks](#-request-lifecycle-ও-hooks)
- [Module System (Static vs DSO)](#-module-system-static-vs-dso)
- [Core Configuration](#-core-configuration)
- [.htaccess Deep Dive](#-htaccess-deep-dive)
- [Reverse Proxy ও Load Balancer](#-reverse-proxy-ও-load-balancer)
- [PHP Integration](#-php-integration-mod_php-vs-php-fpm)
- [Virtual Hosts](#-virtual-hosts)
- [TLS/SSL with mod_ssl](#-tlsssl-with-mod_ssl)
- [Caching, Compression](#-caching-ও-compression)
- [Security Hardening](#-security-hardening)
- [Performance Tuning](#-performance-tuning)
- [Observability](#-observability)
- [Operations](#-operations--apachectl-graceful-restart)
- [Real-world Examples](#-real-world-examples-bd-context)
- [Apache vs Nginx](#-apache-vs-nginx-comprehensive-comparison)
- [Anti-patterns ও Pitfalls](#-anti-patterns-ও-pitfalls)
- [Migration Guide: Apache → Nginx](#-migration-guide-apache--nginx)
- [Production Checklist](#-production-checklist)

---

## 📌 Apache কী এবং কেন এখনও প্রাসঙ্গিক

**Apache HTTP Server** (প্রক্রিয়ার নাম `httpd`) হলো Apache Software Foundation-এর open-source web server, যেটি ১৯৯৫ সাল থেকে ইন্টারনেটের মেরুদণ্ড। নাম এসেছে "**a patchy** server" থেকে — NCSA HTTPd-এর উপর প্যাচ দিয়ে শুরু।

### কেন এখনও মুখ্য (২০২৪+)?

```
   Apache কেন এখনও বাঁচে?
   ========================

   ┌─────────────────────────────────────────────┐
   │  ১. বিশাল ecosystem — ২০+ বছরের module     │
   │  ২. .htaccess — shared hosting এর প্রাণ    │
   │  ৩. cPanel/WHM/Plesk default web server    │
   │  ৪. mod_php এর legacy স্ট্যাক              │
   │  ৫. WordPress documentation-এ default     │
   │  ৬. Per-directory config flexibility       │
   │  ৭. Enterprise (RHEL/CentOS) default       │
   └─────────────────────────────────────────────┘
```

### কখন Apache বেছে নেবেন

| পরিস্থিতি | কারণ |
|----------|------|
| **Shared hosting (cPanel/WHM)** | প্রতি client `.htaccess` দিয়ে নিজের rule define করতে পারে — central reload লাগে না |
| **Multi-tenant `.htaccess` driven** | per-directory override absolute necessity |
| **Legacy PHP app (WordPress/Joomla/Drupal)** | mod_php বা mod_proxy_fcgi → existing `.htaccess` rules অপরিবর্তিত থাকে |
| **Complex per-directory rewriting** | RewriteRule-এর directory-context flexibility nginx-এ পাওয়া কষ্টকর |
| **Enterprise Java integration** | mod_proxy_ajp দিয়ে Tomcat backend |
| **Module-rich requirements** | mod_security (WAF), mod_rewrite, mod_auth_*, mod_ldap, mod_dav একসাথে |

### কখন Apache **বেছে নেবেন না**

| পরিস্থিতি | কেন না | বিকল্প |
|----------|--------|--------|
| High-concurrency edge proxy (১০K+ idle keep-alive) | event MPM ভালো হলেও nginx এর per-connection memory অনেক কম | **Nginx / OpenResty** |
| Modern microservice gateway | dynamic upstream discovery, rate-limit-by-key, JWT auth — এগুলো built-in দরকার | **Envoy / Traefik / Kong** |
| Static-only CDN edge | nginx-এর `sendfile` + tiny memory footprint জিতবে | **Nginx / Caddy** |
| TLS termination at scale (millions of conns) | event MPM এর per-thread overhead | **HAProxy / Nginx** |
| HTTP/3 (QUIC) production | Apache-এ এখনও experimental | **Nginx 1.25+ / Caddy / LiteSpeed** |

### Bangladesh-এ Apache বাস্তবতা

- **ExonHost, Eicra, AlphaNet, BDIX hosts** — সব cPanel/WHM চালায় → backend Apache (EasyApache 4)
- **WordPress sites (.com.bd domains-এর ৭০%+)** Apache + mod_php বা LiteSpeed (Apache-compatible)
- **University, govt portals** — পুরনো Joomla/Drupal Apache prefork-এ
- **Daraz, Pathao, bKash** — production edge nginx, কিন্তু internal admin tool কখনও কখনও Apache

---

## 🏗️ আর্কিটেকচার — MPM Deep Dive

Apache-এর process model নির্ধারিত হয় **MPM (Multi-Processing Module)** দিয়ে। compile time-এ একটাই MPM active।

```
   Apache MPM তিনটি স্বাদ
   =======================

   prefork              worker              event
   (1.x default)        (2.0+ hybrid)       (2.4+ default)

   ┌──────┐             ┌──────┐             ┌──────┐
   │parent│             │parent│             │parent│
   └──┬───┘             └──┬───┘             └──┬───┘
      │                    │                    │
   ┌──┴──┬──┬──┐        ┌──┴──┐              ┌──┴──┐
   │c1  │c2│c3│        │ c1  │              │ c1  │  ← child process
   └─┬──┴──┴──┘        │T1T2 │              │L T1 │  L = listener thread
     │ 1 req            │T3T4 │              │  T2 │  T = worker
     │ /proc            └─────┘              │  T3 │
     ▼                  ১ req/thread         └─────┘
   1 process            (thread-safe         keep-alive offload
   = 1 connection        modules দরকার)     listener-এ
```

### prefork MPM — process-per-request

**মডেল:** parent fork করে অনেকগুলো child process; প্রতিটি child একসাথে **একটি** connection serve করে।

```
prefork সিকোয়েন্স:
   parent (root) ──fork──► child#1 (apache user) ──serves──► request#1
                  ──fork──► child#2 (apache user) ──serves──► request#2
                  ...
                  ──fork──► child#N
```

**কনফিগ:**
```apache
<IfModule mpm_prefork_module>
    StartServers             5
    MinSpareServers          5
    MaxSpareServers         10
    ServerLimit            256
    MaxRequestWorkers      256
    MaxConnectionsPerChild   0
</IfModule>
```

**ভালো:**
- **thread-safe না হলেও চলে** → mod_php নিরাপদ (PHP extensions অনেক সময় thread-unsafe)
- buggy module crash হলে শুধু একটা child মরে — পুরো server না
- debug সহজ (১ process = ১ request)

**মন্দ:**
- **Memory ভয়াবহ:** ২৫০টা child × ৬০MB = **১৫GB RAM** শুধু idle অবস্থায়
- ১০K concurrent connection হ্যান্ডল করতে গেলে ১০K process — Linux fork bomb
- Context switching cost বেশি

### worker MPM — thread + process hybrid

**মডেল:** parent কয়েকটা child fork করে; প্রতিটি child-এ অনেক thread; প্রতিটি thread একটা connection serve করে।

```apache
<IfModule mpm_worker_module>
    StartServers             3
    MinSpareThreads         25
    MaxSpareThreads         75
    ThreadLimit             64
    ThreadsPerChild         25
    MaxRequestWorkers      400   # = ServerLimit × ThreadsPerChild
    ServerLimit             16
    MaxConnectionsPerChild   0
</IfModule>
```

**ভালো:** memory অনেক কম (একটা process অনেক thread share করে), per-connection RAM ~1MB এর মতো।

**মন্দ:**
- সব loaded module থ্রেড-নিরাপদ (thread-safe) হতে হবে → **mod_php না**
- thread crash → পুরো child process ডুবে → অন্য সব thread ও মরে

### event MPM — modern default

**মডেল:** worker-এর মতো, কিন্তু আলাদা একটা **listener thread** kqueue/epoll দিয়ে keep-alive connection-গুলো manage করে। Worker thread শুধু **active** request-এর সময় busy থাকে।

```
   event MPM ফ্লো
   ===============

   Client A (keep-alive idle) ──┐
   Client B (keep-alive idle) ──┼──► Listener thread (epoll) ◄── 1 thread
   Client C (keep-alive idle) ──┘                    │
                                                      │ data ready
                                                      ▼
                                              Worker thread pool
                                              (T1 T2 T3 ... TN)
                                              ↑          ↓
                                            request   response
```

```apache
<IfModule mpm_event_module>
    StartServers              3
    MinSpareThreads          25
    MaxSpareThreads          75
    ThreadLimit              64
    ThreadsPerChild          25
    MaxRequestWorkers       400
    ServerLimit              16
    AsyncRequestWorkerFactor  2
    MaxConnectionsPerChild   0
</IfModule>
```

**ভালো:**
- Keep-alive connection আর worker thread block করে না → **C10K** এর কাছাকাছি যেতে পারে
- modern Linux এ সবচেয়ে memory-efficient MPM
- TLS handshake এর slow client-ও listener-এ wait করে

**মন্দ:**
- active request এর জন্য তবু per-thread model → nginx-এর pure async এর সমান না
- module thread-safe লাগবে (mod_php বাদ)

### MPM তুলনা টেবিল

| বৈশিষ্ট্য | prefork | worker | event |
|----------|---------|--------|-------|
| Concurrency unit | process | thread | thread + async listener |
| Memory/conn (typical) | 30-100 MB | 1-3 MB | 0.5-2 MB |
| mod_php compatible | ✅ হ্যাঁ | ❌ না | ❌ না |
| Thread-safe modules দরকার | না | হ্যাঁ | হ্যাঁ |
| Keep-alive efficient | ❌ blocks worker | ❌ blocks thread | ✅ offload to listener |
| Slow client (slowloris) sensitivity | high | medium | low |
| C10K capable | ❌ না | আংশিক | ✅ হ্যাঁ |
| Best for | mod_php legacy | non-PHP backends | modern PHP-FPM, reverse proxy |
| Default since | Apache 1.x | Apache 2.0 | Apache 2.4 |

> **নিয়ম:** আজকে নতুন setup-এ **event + PHP-FPM** ব্যবহার করুন। prefork শুধু old mod_php migration window-এর জন্য।

---

## 🔄 Request Lifecycle ও Hooks

Apache প্রতিটা request-কে এক ধাপে শেষ করে না — অনেক hook phase আছে যেখানে module register হতে পারে। বুঝলে module dev এবং debug দুটোই সহজ।

```
   Apache Request Phases (module hooks)
   ====================================

   Client request আসছে
        │
        ▼
   ┌─────────────────────┐
   │ post_read_request   │ ◄── headers parse done; mod_unique_id, mod_setenvif
   ├─────────────────────┤
   │ translate_name      │ ◄── URL → filesystem; mod_alias, mod_rewrite
   ├─────────────────────┤
   │ map_to_storage      │ ◄── per-dir config merge; .htaccess walk
   ├─────────────────────┤
   │ header_parser       │
   ├─────────────────────┤
   │ access_checker      │ ◄── Require ip / Allow from; mod_authz_host
   ├─────────────────────┤
   │ check_user_id       │ ◄── mod_auth_basic, mod_auth_digest
   ├─────────────────────┤
   │ check_auth          │ ◄── authorization
   ├─────────────────────┤
   │ type_checker        │ ◄── MIME type detection
   ├─────────────────────┤
   │ fixups              │ ◄── last chance to mutate; mod_headers, mod_env
   ├─────────────────────┤
   │ HANDLER             │ ◄── content generator: mod_php, mod_proxy, default-handler
   ├─────────────────────┤
   │ logger              │ ◄── mod_log_config, mod_log_forensic
   ├─────────────────────┤
   │ cleanup             │ ◄── pool cleanup, free resources
   └─────────────────────┘
```

**দেখার মতো:** `.htaccess` scan হয় `map_to_storage` phase-এ; rewrite rule define করলে translate_name + fixup-এ কাজ করে — তাই নিচের থেকে উপরে directory walk হয় (পরে details আসছে)।

---

## 🧩 Module System (Static vs DSO)

Apache modules দুই ভাবে আসে:

1. **Static** — compile-time এ httpd binary-তে link হয়; reload লাগে না কিন্তু flexibility কম। `httpd -l` দেখায়।
2. **DSO (Dynamic Shared Object)** — `LoadModule` দিয়ে runtime-এ লোড। আজকের সব distro-তে default।

```apache
# /etc/httpd/conf.modules.d/00-base.conf (RHEL family)
LoadModule rewrite_module      modules/mod_rewrite.so
LoadModule headers_module      modules/mod_headers.so
LoadModule proxy_module        modules/mod_proxy.so
LoadModule proxy_http_module   modules/mod_proxy_http.so
LoadModule proxy_fcgi_module   modules/mod_proxy_fcgi.so
LoadModule ssl_module          modules/mod_ssl.so
LoadModule http2_module        modules/mod_http2.so
LoadModule mpm_event_module    modules/mod_mpm_event.so
```

```bash
# loaded modules দেখা
httpd -M

# compile-in (static) modules
httpd -l

# নতুন module compile (apxs টুল)
apxs -i -a -c mod_my_custom.c
# -c compile, -i install, -a auto-add LoadModule line
```

> **Debian/Ubuntu** এ `a2enmod rewrite` / `a2dismod` symlink toggle করে।
> **RHEL/CentOS/Rocky** এ `/etc/httpd/conf.modules.d/*.conf` সরাসরি edit করতে হয়।

---

## ⚙️ Core Configuration

### Directive Contexts ও তাদের precedence

```
   Config merge order (low → high priority)
   ========================================

   server config (httpd.conf)
        │
        ▼
   <VirtualHost>            ← ServerName-ভিত্তিক match
        │
        ▼
   <Directory /var/www>     ← filesystem path
        │
        ▼
   .htaccess walk           ← /var/www → /var/www/site → /var/www/site/wp-admin
        │
        ▼
   <Files "*.php">          ← filename pattern
        │
        ▼
   <Location "/admin">      ← URL path (after rewrite)
```

**মনে রাখার নিয়ম:**

- `<Directory>` → `<DirectoryMatch>` → `.htaccess` → `<Files>` → `<FilesMatch>` → `<Location>` → `<LocationMatch>` — এই ক্রমে merge হয়
- `<Location>` URL-based (rewrite-এর পর), `<Directory>` filesystem-based
- duplicate directive থাকলে **later একটাই জয়ী** — তাই vhost level config server-level কে override করে

### সর্বনিম্ন কাজ-চলার মতো httpd.conf

```apache
ServerRoot   "/etc/httpd"
PidFile      /var/run/httpd/httpd.pid
Listen       80
Listen       443

User  apache
Group apache

ServerName   www.example.com.bd:80
ServerAdmin  webmaster@example.com.bd

DocumentRoot "/var/www/html"

<Directory />
    AllowOverride none
    Require all denied
</Directory>

<Directory "/var/www/html">
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

# global default
<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>

ErrorLog  "logs/error_log"
LogLevel  warn
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %D" combined
CustomLog "logs/access_log" combined

# include vhosts
IncludeOptional conf.d/*.conf
IncludeOptional vhosts/*.conf
```

### গুরুত্বপূর্ণ directive-গুলো

| Directive | কাজ |
|-----------|------|
| `ServerRoot` | apache install root (config + modules) |
| `DocumentRoot` | site root on filesystem |
| `ServerName` | vhost canonical name |
| `ServerAlias` | extra hostnames (`www.x.com x.com *.x.com`) |
| `Listen 0.0.0.0:80` | bind address+port |
| `User`/`Group` | child process privilege drop |
| `AllowOverride` | কোন `.htaccess` directive অনুমোদিত |
| `Options` | `Indexes`, `FollowSymLinks`, `MultiViews`, `ExecCGI`, `SymLinksIfOwnerMatch` |
| `Require` | 2.4+ access control (replacing Order/Allow/Deny) |

### AllowOverride — performance vs flexibility

```apache
# কঠোরভাবে production
<Directory "/var/www/html">
    AllowOverride None         # .htaccess পড়বেই না — fastest
</Directory>

# shared hosting flexible
<Directory "/home/*/public_html">
    AllowOverride All          # সব override allowed — slowest
</Directory>

# ইন-বিটউইন
<Directory "/var/www/wordpress">
    AllowOverride FileInfo Options=Indexes Limit AuthConfig
</Directory>
```

**প্রভাব:** `AllowOverride None` থাকলে Apache ফাইলসিস্টেম walk করে `.htaccess` পড়ে না — এক request-এ ৫-১০টা stat() syscall save হয়। উচ্চ-RPS site এ এটা ১০-২০% throughput improvement দেয়।

### Order/Allow/Deny (legacy 2.2) → Require (2.4+)

```apache
# পুরনো 2.2 ভঙ্গি (এখনও mod_access_compat থাকলে কাজ করে, কিন্তু deprecated)
<Directory "/var/www/admin">
    Order allow,deny
    Allow from 103.230.106.0/24
    Deny from all
</Directory>

# আধুনিক 2.4+ ভঙ্গি
<Directory "/var/www/admin">
    Require ip 103.230.106.0/24       # BDIX office IP
    # Require all granted
    # Require valid-user
    # Require ldap-group cn=admins,...
</Directory>
```

### Alias, ScriptAlias, RedirectMatch

```apache
Alias       "/static"  "/var/data/static"
ScriptAlias "/cgi-bin" "/var/www/cgi-bin"

RedirectMatch 301 ^/old-products/(.*)$ /products/$1
Redirect      permanent /privacy /privacy-policy
```

---

## 🔧 .htaccess Deep Dive

`.htaccess` হলো per-directory config — Apache ফাইলসিস্টেম walk করতে করতে প্রতিটা directory-তে `.htaccess` পেলে merge করে। **shared hosting-এর প্রাণ**।

### কীভাবে `.htaccess` discover হয়

```
   Request: /home/user1/public_html/wp-admin/admin.php

   Apache walks (top → bottom) এবং প্রতিটা স্তরে .htaccess পেলে merge:

   /                      → no .htaccess
   /home                  → no
   /home/user1            → maybe
   /home/user1/public_html → ✅ .htaccess (WordPress permalinks)
   /home/user1/public_html/wp-admin → ✅ .htaccess (admin restriction)

   প্রতি request-এ ৫টা stat() — AllowOverride All এ
```

### AllowOverride scope

| Override class | কী allow করে |
|----------------|-------------|
| `AuthConfig` | AuthType, AuthName, AuthUserFile, Require |
| `FileInfo` | AddType, AddHandler, ErrorDocument, Header, RewriteRule |
| `Indexes` | DirectoryIndex, IndexOptions |
| `Limit` | Allow/Deny/Require for HTTP methods |
| `Options=...` | নির্দিষ্ট Options পরিবর্তন |
| `All` | সব override |
| `None` | কিছুই পড়বে না (fastest) |

### mod_rewrite — flag reference

```apache
RewriteEngine On
RewriteBase   /

RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

| Flag | মানে |
|------|------|
| `L` | last — আর কোনো rule check করো না |
| `R=301` | external redirect (default 302) |
| `QSA` | query string append (existing query preserve) |
| `NC` | case-insensitive match |
| `P` | proxy (mod_proxy দিয়ে internal forward) |
| `E=VAR:val` | environment variable set |
| `F` | forbidden (403) |
| `G` | gone (410) |
| `S=N` | next N rules skip |
| `END` | rewrite engine থামো (.htaccess এর পরবর্তী রাউন্ডেও না) |

### সাধারণ patterns (BD context)

#### WordPress permalinks (every cPanel site-এ থাকে)

```apache
# /home/user/public_html/.htaccess
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
```

#### Laravel `public/` routing

```apache
# /home/user/public_html/.htaccess (project root)
RewriteEngine On
RewriteRule ^(.*)$ public/$1 [L]
```

```apache
# /home/user/public_html/public/.htaccess (Laravel default)
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)/$ /$1 [L,R=301]

    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

#### Force HTTPS + non-www → www (Daraz-style canonical)

```apache
RewriteEngine On

# www → non-www (apex)
RewriteCond %{HTTP_HOST} ^www\.example\.com\.bd$ [NC]
RewriteRule ^(.*)$ https://example.com.bd/$1 [L,R=301]

# http → https
RewriteCond %{HTTPS} !=on
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

#### Hotlink protection (BDIX bandwidth save)

```apache
RewriteEngine On
RewriteCond %{HTTP_REFERER} !^$
RewriteCond %{HTTP_REFERER} !^https?://(www\.)?example\.com\.bd [NC]
RewriteRule \.(jpg|jpeg|png|gif|webp)$ - [F,NC]
```

#### IP block (abuse mitigate)

```apache
<RequireAll>
    Require all granted
    Require not ip 45.155.205.0/24
    Require not ip 185.220.101.0/24
</RequireAll>
```

### .htaccess performance impact

```
   AllowOverride All এ প্রতি request-এ:

   /home/u/public_html/wp-content/uploads/2024/img.jpg
   ↓
   stat /home/.htaccess
   stat /home/u/.htaccess
   stat /home/u/public_html/.htaccess              ← parsed
   stat /home/u/public_html/wp-content/.htaccess
   stat /home/u/public_html/wp-content/uploads/.htaccess
   stat .../2024/.htaccess

   = 6 syscall + parse cost প্রতি request-এ
```

**Mitigation:**
- shared hosting বাদে production single-tenant সাইটে `AllowOverride None` দিয়ে rules vhost-এ move করুন
- যদি `.htaccess` লাগে, তাহলে directory tree **shallow** রাখুন
- নির্দিষ্ট override class দিন (`FileInfo` only) — সব না

---

## 🔁 Reverse Proxy ও Load Balancer

Apache শুধু origin web server না — modern stack-এ এটা সাধারণত PHP-FPM/Tomcat/Node-এর সামনে reverse proxy হয়।

### ন্যূনতম reverse proxy

```apache
LoadModule proxy_module       modules/mod_proxy.so
LoadModule proxy_http_module  modules/mod_proxy_http.so

<VirtualHost *:80>
    ServerName api.example.com.bd

    ProxyPreserveHost   On
    ProxyRequests       Off                 # forward proxy বন্ধ — security critical

    ProxyPass        "/"  "http://127.0.0.1:3000/"
    ProxyPassReverse "/"  "http://127.0.0.1:3000/"

    # X-Forwarded-* হেডার backend-কে দিতে
    RequestHeader set X-Forwarded-Proto "http"
    RequestHeader set X-Forwarded-Port  "80"
    ProxyAddHeaders On

    ProxyTimeout       60
    ProxyIOBufferSize  16384
</VirtualHost>
```

> ⚠️ `ProxyRequests On` কখনই production-এ দেবেন না — সেটা Apache-কে open forward proxy বানিয়ে দেয় (abuse + spam)।

### Load balancer cluster

```apache
LoadModule proxy_balancer_module    modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module       modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule lbmethod_bybusyness_module modules/mod_lbmethod_bybusyness.so
LoadModule lbmethod_bytraffic_module  modules/mod_lbmethod_bytraffic.so

<Proxy "balancer://api-cluster">
    BalancerMember "http://10.0.0.11:3000" route=node1 loadfactor=1 retry=10 timeout=5 ping=2
    BalancerMember "http://10.0.0.12:3000" route=node2 loadfactor=1 retry=10 timeout=5 ping=2
    BalancerMember "http://10.0.0.13:3000" route=node3 loadfactor=2 retry=10 timeout=5 ping=2  # 2x capacity
    ProxySet lbmethod=bybusyness stickysession=ROUTEID
</Proxy>

<VirtualHost *:443>
    ServerName api.example.com.bd

    Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" \
        env=BALANCER_ROUTE_CHANGED

    ProxyPass        "/" "balancer://api-cluster/" stickysession=ROUTEID
    ProxyPassReverse "/" "balancer://api-cluster/"

    # /balancer-manager UI (admin দেখার জন্য)
    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 103.230.106.0/24       # office IP only
    </Location>
</VirtualHost>
```

### lbmethod তুলনা

| Method | Algorithm | কখন |
|--------|-----------|-----|
| `byrequests` | round-robin (weighted) | uniform backend |
| `bytraffic` | bytes sent ভিত্তিক | varied response size |
| `bybusyness` | open connection কমের দিকে | varied response time, slow endpoints |
| `heartbeat` | mod_heartbeat health-aware | dynamic cluster |

### BalancerMember parameters

| Param | মানে |
|-------|------|
| `loadfactor=N` | আনুপাতিক weight (১-১০০) |
| `retry=N` | dead member কে কত sec পরে retry |
| `timeout=N` | connect timeout |
| `ping=N` | HTTP/1.1 100-Continue ping |
| `route=name` | sticky session token |
| `status=+H` | hot standby (অন্যরা মরলেই active) |
| `status=+D` | disabled (drain) |

### WebSocket proxy

```apache
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so

<VirtualHost *:443>
    ServerName chat.example.com.bd

    # WebSocket upgrade — /ws path
    ProxyPass        "/ws" "ws://127.0.0.1:6001/ws"
    ProxyPassReverse "/ws" "ws://127.0.0.1:6001/ws"

    # বাকি HTTP traffic
    ProxyPass        "/" "http://127.0.0.1:6001/"
    ProxyPassReverse "/" "http://127.0.0.1:6001/"
</VirtualHost>
```

> Apache 2.4.47+ এ `mod_proxy_http` নিজেই WebSocket upgrade handle করে; আগের version এ `mod_proxy_wstunnel` লাগে।

### PHP-FPM (mod_proxy_fcgi) — production recommended

```apache
LoadModule proxy_fcgi_module  modules/mod_proxy_fcgi.so

<VirtualHost *:443>
    ServerName  example.com.bd
    DocumentRoot /var/www/example.com.bd/public

    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # OR TCP variant:
    # <FilesMatch "\.php$">
    #     SetHandler "proxy:fcgi://127.0.0.1:9000"
    # </FilesMatch>

    <Directory "/var/www/example.com.bd/public">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### AJP to Tomcat (legacy enterprise)

```apache
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so

<VirtualHost *:80>
    ServerName erp.legacy.gov.bd

    ProxyPass        "/"  "ajp://127.0.0.1:8009/"
    ProxyPassReverse "/"  "ajp://127.0.0.1:8009/"
</VirtualHost>
```

> ⚠️ **Ghostcat (CVE-2020-1938)** — Tomcat AJP-তে port `8009` কখনই public-এ expose করবেন না, এবং `secret` শেয়ার করুন।

### HTTP/2 to backend

```apache
LoadModule proxy_http2_module modules/mod_proxy_http2.so

ProxyPass "/api/" "h2c://backend.internal:8080/"     # plain h2c
ProxyPass "/api/" "h2://backend.internal:8443/"      # TLS h2
```

### mTLS to upstream (zero-trust internal)

```apache
SSLProxyEngine                 On
SSLProxyVerify                 require
SSLProxyCheckPeerName          on
SSLProxyMachineCertificateFile /etc/pki/tls/private/apache-client.pem
SSLProxyCACertificateFile      /etc/pki/tls/certs/internal-ca.pem

ProxyPass        "/secure/" "https://internal-api.bank.bd/"
ProxyPassReverse "/secure/" "https://internal-api.bank.bd/"
```

### Timeouts ও buffer

```apache
ProxyTimeout         60          # backend response timeout
ProxyIOBufferSize    16384       # 16KB streaming buffer
ProxyReceiveBufferSize 0         # OS default
ProxyVia             Off         # via header hide
```

---

## 🐘 PHP Integration (mod_php vs PHP-FPM)

### mod_php — পুরনো default

```apache
LoadModule php_module modules/libphp.so   # prefork only

<FilesMatch "\.php$">
    SetHandler application/x-httpd-php
</FilesMatch>

PHPIniDir /etc/php.d/
```

**Pros:** install করেই ভুলে যান, সহজ।
**Cons:**
- শুধু prefork-এ চলে (thread-unsafe)
- প্রতি Apache child-এ PHP interpreter loaded → ২০০ child × ৮০MB = ১৬GB RAM
- WordPress + MySQL + xdebug = ১২০MB+ per child
- ২০২৪-এ এটা **anti-pattern** high-traffic site-এ

### PHP-FPM via mod_proxy_fcgi — modern

```
   Apache event MPM                PHP-FPM pool (separate user)
   ┌──────────────┐                ┌─────────────────────┐
   │ static .css  │ ─────serves────► browser
   │ static .jpg  │
   │              │
   │ *.php        │ ──fcgi unix────► php-fpm child #1
   │              │                  php-fpm child #2 ... #N
   └──────────────┘                └─────────────────────┘
   লাইট, thread-pooled              শুধু PHP execute
```

**সুবিধা:**
- Apache event MPM + PHP-FPM combined = nginx + PHP-FPM-এর ৯০% efficiency
- PHP-FPM pool আলাদা user-এ চালানো যায় (multi-tenant security)
- `pm.max_children` PHP-তে আলাদা tune
- OPcache শুধু FPM master process-এ — Apache child-এ না

### mod_fcgid (alternative)

`mod_proxy_fcgi`-এর আগে standard ছিল। FastCGI process নিজেই spawn করে ও lifecycle manage করে। আজকে `mod_proxy_fcgi` + external PHP-FPM **preferred**।

### Shared hosting — user separation

| Approach | কীভাবে |
|----------|--------|
| **mpm-itk** | প্রতিটা vhost আলাদা UID/GID-এ child fork (per-request setuid) — cPanel-এ default |
| **mod_ruid2** | অনুরূপ; per-request privilege drop |
| **PHP-FPM pool-per-user** | প্রতি user-এর জন্য আলাদা pool, আলাদা socket, আলাদা UID |
| **mod_suphp** (legacy) | প্রতি PHP execute-এ user-এর UID-এ চলে — slow |

```apache
# mpm-itk উদাহরণ (cPanel-style)
<VirtualHost *:80>
    ServerName user1.example.com.bd
    DocumentRoot /home/user1/public_html
    <IfModule mpm_itk_module>
        AssignUserID user1 user1
    </IfModule>
</VirtualHost>
```

---

## 🏠 Virtual Hosts

### Name-based vs IP-based

```
   Name-based:  ১টা IP, অনেক hostname (Host: header দিয়ে differentiate)
                ── এটা ৯৯% ক্ষেত্রে যা use হয়
   IP-based:    আলাদা IP প্রতি site (legacy, IPv4 scarcity যুগে costly)
```

### Multi-vhost setup (cPanel pattern)

```apache
# /etc/httpd/vhosts/userdata.conf — cPanel ১০০০+ vhost generate করে এমন
Listen 80
Listen 443

<VirtualHost *:80>
    ServerName  example.com.bd
    ServerAlias www.example.com.bd
    DocumentRoot /home/exampleuser/public_html
    CustomLog "logs/example.com.bd-access_log" combined
    ErrorLog  "logs/example.com.bd-error_log"
</VirtualHost>

<VirtualHost *:80>
    ServerName  shop.com.bd
    ServerAlias www.shop.com.bd
    DocumentRoot /home/shopuser/public_html
    ...
</VirtualHost>
```

### mod_macro দিয়ে DRY

```apache
LoadModule macro_module modules/mod_macro.so

<Macro VHost $name $user>
    <VirtualHost *:443>
        ServerName  $name
        ServerAlias www.$name
        DocumentRoot /home/$user/public_html
        CustomLog "logs/$name-access_log" combined
        ErrorLog  "logs/$name-error_log"

        SSLEngine on
        SSLCertificateFile    /etc/letsencrypt/live/$name/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/$name/privkey.pem
    </VirtualHost>
</Macro>

Use VHost example.com.bd  exampleuser
Use VHost shop.com.bd     shopuser
Use VHost blog.com.bd     bloguser

UndefMacro VHost
```

### vhost selection নিয়ম

1. সব `<VirtualHost>` যার IP+port match করে — সেই pool থেকে বাছাই
2. `Host:` header `ServerName`/`ServerAlias`-এর সাথে exact বা wildcard match
3. কোনো match না হলে — **first defined vhost** default (অপ্রত্যাশিত behavior!)

> **ভালো অভ্যাস:** ১ম vhost একটা "catch-all default" বানান যেটা 444/444 দেয় — unknown host hit-এ leak ঠেকাতে।

```apache
# default catch-all (১ম vhost)
<VirtualHost *:443>
    ServerName  _default_
    SSLEngine on
    SSLCertificateFile    /etc/pki/tls/certs/snakeoil.pem
    SSLCertificateKeyFile /etc/pki/tls/private/snakeoil.key
    <Location "/">
        Require all denied
    </Location>
    ErrorDocument 403 "Unknown host"
</VirtualHost>
```

---

## 🔐 TLS/SSL with mod_ssl

### Production-grade SSL vhost

```apache
LoadModule ssl_module    modules/mod_ssl.so
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so

Listen 443

# global SSL
SSLPassPhraseDialog     builtin
SSLSessionCache         "shmcb:/run/httpd/sslcache(512000)"
SSLSessionCacheTimeout  300
SSLRandomSeed startup   file:/dev/urandom 256
SSLRandomSeed connect   builtin
SSLCryptoDevice         builtin

# OCSP stapling (browser এ revocation check fast করে)
SSLUseStapling          On
SSLStaplingCache        "shmcb:/run/httpd/stapling-cache(150000)"
SSLStaplingResponderTimeout 5
SSLStaplingReturnResponderErrors off

<VirtualHost *:443>
    ServerName www.example.com.bd

    SSLEngine On
    SSLCertificateFile      /etc/letsencrypt/live/example.com.bd/fullchain.pem
    SSLCertificateKeyFile   /etc/letsencrypt/live/example.com.bd/privkey.pem
    # SSLCertificateChainFile (Apache <2.4.8 — newer-এ fullchain.pem এ চলে)

    # modern profile (Mozilla intermediate)
    SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    SSLHonorCipherOrder     on
    SSLSessionTickets       off

    # HTTP/2
    Protocols h2 http/1.1

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"

    DocumentRoot "/var/www/example.com.bd/public"
</VirtualHost>
```

### mod_md — Let's Encrypt automatic certs (Apache 2.4.30+)

```apache
LoadModule md_module modules/mod_md.so

MDomain example.com.bd www.example.com.bd
MDCertificateAuthority https://acme-v02.api.letsencrypt.org/directory
MDCertificateAgreement accepted
MDContactEmail webmaster@example.com.bd
MDPrivateKeys RSA 2048
MDStoreDir /var/www/md
MDRenewMode auto

<VirtualHost *:443>
    ServerName example.com.bd
    SSLEngine on
    # mod_md নিজেই cert path inject করে
    Protocols h2 http/1.1
</VirtualHost>
```

> mod_md না থাকলে **certbot Apache plugin**: `certbot --apache -d example.com.bd -d www.example.com.bd`

### mTLS — client certificate

```apache
<VirtualHost *:443>
    ServerName api.internal.bank.bd

    SSLEngine               on
    SSLCertificateFile      /etc/pki/tls/certs/server.crt
    SSLCertificateKeyFile   /etc/pki/tls/private/server.key

    SSLVerifyClient         require
    SSLVerifyDepth          2
    SSLCACertificateFile    /etc/pki/tls/certs/internal-ca.pem

    <Location "/">
        SSLRequireSSL
        SSLOptions +StdEnvVars +ExportCertData
        # client cert info backend-কে header হিসেবে
        RequestHeader set X-Client-DN "%{SSL_CLIENT_S_DN}s"
        RequestHeader set X-Client-Cert "%{SSL_CLIENT_CERT}s"
    </Location>
</VirtualHost>
```

> বিস্তারিত TLS তত্ত্বের জন্য দেখুন [`14-networking/tls-ssl.md`](../14-networking/tls-ssl.md)।

---

## 📦 Caching ও Compression

### mod_cache (server-side reverse-proxy cache)

```apache
LoadModule cache_module        modules/mod_cache.so
LoadModule cache_disk_module   modules/mod_cache_disk.so
LoadModule cache_socache_module modules/mod_cache_socache.so

<IfModule mod_cache_disk.c>
    CacheEnable    disk "/"
    CacheRoot      "/var/cache/httpd/proxy"
    CacheDirLevels 2
    CacheDirLength 1
    CacheMaxFileSize 1048576
    CacheLock      on
    CacheLockPath  "/tmp/mod_cache-lock"
    CacheLockMaxAge 5
    CacheIgnoreHeaders Set-Cookie
    CacheIgnoreCacheControl Off
</IfModule>
```

### Client-side caching

```apache
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresDefault                       "access plus 1 month"
    ExpiresByType image/jpeg             "access plus 1 year"
    ExpiresByType image/webp             "access plus 1 year"
    ExpiresByType text/css               "access plus 1 month"
    ExpiresByType application/javascript "access plus 1 month"
    ExpiresByType text/html              "access plus 5 minutes"
</IfModule>

<IfModule mod_headers.c>
    <FilesMatch "\.(js|css|jpg|png|webp|woff2)$">
        Header set Cache-Control "public, max-age=31536000, immutable"
    </FilesMatch>
</IfModule>

FileETag MTime Size      # inode বাদ — multi-server e-tag mismatch ঠেকায়
```

### Compression — mod_deflate ও mod_brotli

```apache
LoadModule deflate_module modules/mod_deflate.so
LoadModule brotli_module  modules/mod_brotli.so   # 2.4.26+

<IfModule mod_brotli.c>
    BrotliCompressionQuality 5
    AddOutputFilterByType BROTLI_COMPRESS \
        text/html text/plain text/css text/xml \
        application/javascript application/json application/xml \
        image/svg+xml font/woff2
</IfModule>

<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE \
        text/html text/plain text/css text/xml \
        application/javascript application/json application/xml \
        image/svg+xml
    DeflateCompressionLevel 5
    SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png|webp|woff2)$ no-gzip
</IfModule>
```

---

## 🛡️ Security Hardening

### Information leakage বন্ধ

```apache
ServerTokens     Prod          # "Apache" বলবে, version না
ServerSignature  Off           # error page-এ version footer না
TraceEnable      Off           # XST attack ঠেকায়
FileETag         None          # inode leak ঠেকায়

Header unset    X-Powered-By
Header always unset Server     # full strip চাইলে mod_security দিয়ে
```

### Slowloris ঠেকাতে mod_reqtimeout

```apache
LoadModule reqtimeout_module modules/mod_reqtimeout.so

<IfModule mod_reqtimeout.c>
    RequestReadTimeout header=20-40,MinRate=500 \
                       body=20,MinRate=500
</IfModule>
```

### mod_security (WAF) + OWASP CRS

```apache
LoadModule security2_module modules/mod_security2.so

<IfModule security2_module>
    Include /etc/httpd/modsecurity.d/modsecurity.conf
    Include /etc/httpd/modsecurity.d/crs/crs-setup.conf
    Include /etc/httpd/modsecurity.d/crs/rules/*.conf

    SecRuleEngine On
    SecRequestBodyAccess On
    SecResponseBodyAccess Off
    SecAuditEngine RelevantOnly
    SecAuditLog /var/log/httpd/modsec_audit.log
</IfModule>
```

> **OWASP CRS** SQL injection, XSS, RFI/LFI, scanner detection — কয়েকশ rule। CRS-এর paranoia level (1-4) দিয়ে strictness control করুন।

### mod_evasive — DoS mitigation

```apache
LoadModule evasive20_module modules/mod_evasive20.so

<IfModule mod_evasive20.c>
    DOSHashTableSize    3097
    DOSPageCount        5      # 5 hits/sec same page
    DOSSiteCount        50     # 50 hits/sec same IP
    DOSPageInterval     1
    DOSSiteInterval     1
    DOSBlockingPeriod   60
    DOSEmailNotify      ops@example.com.bd
    DOSLogDir           /var/log/httpd/evasive
</IfModule>
```

### Security headers (link to dedicated chapter)

```apache
<IfModule mod_headers.c>
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
    Header always set Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; script-src 'self' 'unsafe-inline'"
</IfModule>
```

> বিস্তারিত: [`11-security/security-headers.md`](../11-security/security-headers.md)।

### CVE patching

- RHEL/Rocky/AlmaLinux: `dnf update httpd mod_ssl mod_http2`
- Ubuntu/Debian: `apt update && apt upgrade apache2 libapache2-mod-*`
- **monitor:** [httpd.apache.org/security_report.html](https://httpd.apache.org/security_report.html), CISA KEV
- প্রতি ৩-৬ মাসে minor version upgrade — major-এর জন্য staging test

---

## ⚡ Performance Tuning

### MPM (event) sizing

```apache
<IfModule mpm_event_module>
    StartServers              3
    MinSpareThreads          25
    MaxSpareThreads          75
    ThreadsPerChild          25
    ThreadLimit              64
    ServerLimit              16
    MaxRequestWorkers       400      # = ServerLimit × ThreadsPerChild
    MaxConnectionsPerChild  10000    # 0 = unlimited; non-zero memory leak ঠেকায়
    AsyncRequestWorkerFactor  2
</IfModule>
```

**MaxRequestWorkers calc:**

```
RAM available for Apache = total - (OS + DB + PHP-FPM + cache)
PHP-FPM in front?  → Apache child light (~5MB/thread)
   MaxRequestWorkers = (RAM_for_apache_MB) / 5

mod_php prefork?   → child heavy (~80MB)
   MaxRequestWorkers = (RAM_for_apache_MB) / 80
```

### KeepAlive

```apache
KeepAlive               On
KeepAliveTimeout        5        # 60-এ default থেকে কমান
MaxKeepAliveRequests    100
```

> event MPM-এ keep-alive সস্তা; prefork-এ keep-alive worker block করে — সেখানে timeout আরও কম (2s) রাখা ভালো।

### Generic timeouts

```apache
Timeout                 60
LimitRequestBody        20971520    # 20 MB max upload
```

### sendfile, mmap

```apache
EnableSendfile     On    # zero-copy static files
EnableMMAP         On    # NFS-এ Off করুন (cache coherency issue)
```

### AcceptFilter (Linux/BSD kernel optimization)

```apache
AcceptFilter http  data           # Linux: TCP_DEFER_ACCEPT
AcceptFilter https data
```

### Reverse DNS off

```apache
HostnameLookups Off       # default; কখনই On করবেন না — প্রতি request-এ DNS!
```

### mod_status / mod_info ON হলে restrict

```apache
ExtendedStatus On

<Location "/server-status">
    SetHandler server-status
    Require ip 127.0.0.1
    Require ip 103.230.106.0/24    # office BDIX
</Location>

<Location "/server-info">
    SetHandler server-info
    Require ip 127.0.0.1
</Location>
```

---

## 📊 Observability

### LogFormat ও CustomLog

```apache
# %D = response time (microseconds), %{X-Forwarded-For}i = real client IP behind LB
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %D %{X-Forwarded-For}i" combined
LogFormat "%v:%p %h %l %u %t \"%r\" %>s %b" vhost_combined

CustomLog "logs/access_log"      combined
CustomLog "logs/vhost_access"    vhost_combined
ErrorLog  "logs/error_log"
LogLevel  warn rewrite:trace2    # selective debug
```

### mod_log_forensic — full request capture

```apache
LoadModule log_forensic_module modules/mod_log_forensic.so
ForensicLog logs/forensic_log
```

### /server-status → Prometheus

```bash
# apache_exporter + Prometheus + Grafana
docker run -d -p 9117:9117 \
  bitnami/apache-exporter:latest \
  --scrape_uri=http://apache:80/server-status?auto

# Prometheus scrape config
# - job_name: apache
#   static_configs:
#     - targets: ['apache-exporter:9117']
```

### সাধারণত যেসব metric দরকার

| Metric | কোথায় |
|--------|------|
| `apache_busy_workers` / `apache_idle_workers` | mod_status |
| `apache_scoreboard_*` (R/W/K/D states) | mod_status |
| `apache_accesses_total` | mod_status |
| RPS, p95 latency | access log + Prometheus + log_exporter |
| 5xx rate | access log তে `%>s` filter |

### Log rotation

```bash
# rotatelogs (piped — recommended)
CustomLog "|/usr/bin/rotatelogs -l /var/log/httpd/access_log.%Y-%m-%d 86400" combined

# OR logrotate
# /etc/logrotate.d/httpd
/var/log/httpd/*log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
    endscript
}
```

---

## 🛠️ Operations — apachectl, graceful restart

```bash
# config validate (deploy-এর আগে অবশ্যই)
apachectl configtest
# OR: httpd -t -D DUMP_VHOSTS

# graceful — current connection শেষ হবার আগে হাত দেবে না
apachectl graceful           # SIGUSR1
apachectl graceful-stop      # finish then stop
apachectl restart            # SIGHUP — abrupt
apachectl stop               # SIGTERM
apachectl start

# show loaded modules
httpd -M

# show vhost order resolution
httpd -S
```

| Signal | কাজ |
|--------|------|
| `USR1` | graceful — config reload, connection drop না |
| `HUP`  | restart — সব worker kill, abrupt |
| `TERM` | shutdown |
| `WINCH` | graceful-stop (USR1 + stop) |

### Docker run

```bash
docker run -d --name apache \
  -p 80:80 -p 443:443 \
  -v /srv/site:/usr/local/apache2/htdocs/:ro \
  -v /etc/httpd/conf:/usr/local/apache2/conf/:ro \
  httpd:2.4
```

```dockerfile
FROM httpd:2.4
COPY ./conf/httpd.conf  /usr/local/apache2/conf/httpd.conf
COPY ./vhosts/          /usr/local/apache2/conf/vhosts/
COPY ./public/          /usr/local/apache2/htdocs/
RUN sed -i \
    -e 's/^#\(LoadModule .*mod_rewrite.so\)/\1/' \
    -e 's/^#\(LoadModule .*mod_proxy.so\)/\1/' \
    -e 's/^#\(LoadModule .*mod_proxy_fcgi.so\)/\1/' \
    /usr/local/apache2/conf/httpd.conf
```

### Kubernetes

- Deployment-এ `httpd` image, `readinessProbe` → `/server-status?auto`
- ConfigMap-এ vhost mount, secret-এ TLS cert
- HPA `apache_busy_workers` metric দিয়ে — তবে Apache-কে front-tier হিসেবে k8s-এ ব্যবহার দুর্লভ; সাধারণত nginx-ingress বা Envoy

---

## 🌏 Real-world Examples (BD Context)

### ১) WordPress on cPanel shared host (Apache + PHP-FPM + Let's Encrypt)

```apache
# /etc/apache2/conf.d/userdata/std/2_4/exampleuser/example.com.bd/php-fpm.conf
<IfModule mod_proxy_fcgi.c>
    <FilesMatch \.(php|phtml)$>
        SetHandler "proxy:unix:/var/run/php-fpm/exampleuser.sock|fcgi://localhost"
    </FilesMatch>
</IfModule>

# /usr/local/apache/conf/userdata/ssl/2_4/exampleuser/example.com.bd/...
# AutoSSL/Let's Encrypt cert path cPanel manage করে

# /home/exampleuser/public_html/.htaccess (WP standard)
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>

# Force HTTPS + non-www
<IfModule mod_rewrite.c>
RewriteCond %{HTTPS} off [OR]
RewriteCond %{HTTP_HOST} ^www\. [NC]
RewriteRule ^(.*)$ https://example.com.bd/$1 [L,R=301]
</IfModule>
```

### ২) Laravel production vhost

```apache
<VirtualHost *:443>
    ServerName  api.startup.com.bd
    DocumentRoot /var/www/api/current/public

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/api.startup.com.bd/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/api.startup.com.bd/privkey.pem

    Protocols h2 http/1.1

    <Directory "/var/www/api/current/public">
        Options -Indexes +FollowSymLinks
        AllowOverride None              # rules সরাসরি vhost-এ — fastest
        Require all granted

        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^ index.php [L]
    </Directory>

    <FilesMatch "\.php$">
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost"
    </FilesMatch>

    Header always set Strict-Transport-Security "max-age=63072000"

    ErrorLog  /var/log/httpd/api-error.log
    CustomLog /var/log/httpd/api-access.log combined
</VirtualHost>
```

### ৩) Apache-as-LB সামনে ৩টা Node.js (sticky session)

```apache
<Proxy "balancer://nodejs-cluster">
    BalancerMember "http://10.0.10.11:3000" route=n1
    BalancerMember "http://10.0.10.12:3000" route=n2
    BalancerMember "http://10.0.10.13:3000" route=n3
    ProxySet lbmethod=bybusyness stickysession=NODESID
    ProxySet timeout=10
</Proxy>

<VirtualHost *:443>
    ServerName app.example.com.bd
    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/app.example.com.bd/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app.example.com.bd/privkey.pem
    Protocols h2 http/1.1

    Header add Set-Cookie "NODESID=.%{BALANCER_WORKER_ROUTE}e; path=/; HttpOnly; Secure" \
        env=BALANCER_ROUTE_CHANGED

    ProxyPreserveHost  On
    ProxyRequests      Off
    RequestHeader set  X-Forwarded-Proto "https"

    # WebSocket
    ProxyPass        "/ws"  "balancer://nodejs-cluster/ws"  upgrade=websocket
    ProxyPassReverse "/ws"  "balancer://nodejs-cluster/ws"

    # HTTP
    ProxyPass        "/" "balancer://nodejs-cluster/" stickysession=NODESID
    ProxyPassReverse "/" "balancer://nodejs-cluster/"

    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 103.230.106.0/24
        AuthType Basic
        AuthName "ops only"
        AuthUserFile /etc/httpd/.htpasswd
        Require valid-user
    </Location>
</VirtualHost>
```

### ৪) Apache + Tomcat (legacy banking ERP)

```apache
<VirtualHost *:80>
    ServerName erp.bank.com.bd

    ProxyPass        "/erp/" "ajp://127.0.0.1:8009/erp/"
    ProxyPassReverse "/erp/" "ajp://127.0.0.1:8009/erp/"

    # static-গুলো Apache সরাসরি serve
    Alias "/erp/static" "/var/www/erp/static"
    <Directory "/var/www/erp/static">
        Require all granted
    </Directory>
</VirtualHost>
```

### ৫) BD ISP customer portal — prefork+mod_php → event+PHP-FPM migration

**আগের অবস্থা (problem):**
```
   prefork MPM
   MaxRequestWorkers 256
   প্রতি child ৯০MB (mod_php + xdebug + customer module)
   = 256 × 90 = 23 GB RAM
   peak এ swap → response time 8s+, 5xx storm
```

**Migration পরিকল্পনা:**

1. PHP-FPM install: `dnf install php-fpm`; pool config `/etc/php-fpm.d/www.conf`
   ```
   pm = dynamic
   pm.max_children = 80
   pm.start_servers = 10
   pm.min_spare_servers = 5
   pm.max_spare_servers = 20
   ```
2. mod_php disable: `mv /etc/httpd/conf.modules.d/10-php.conf{,.disabled}`
3. MPM swap: `/etc/httpd/conf.modules.d/00-mpm.conf` — prefork comment, event uncomment
4. vhost-এ:
   ```apache
   <FilesMatch \.php$>
     SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
   </FilesMatch>
   ```
5. `apachectl configtest && systemctl restart httpd php-fpm`
6. canary — staging-এ load test (`ab`, `wrk`), production-এ ১০% traffic shift

**Result (ছয় মাসের অভিজ্ঞতা):**
| Metric | আগে | পরে |
|--------|------|------|
| RAM idle | 23 GB | 3 GB |
| p95 latency | 4.2s | 0.6s |
| Concurrent users | ~700 | ~5000 |
| 5xx rate | 0.8% | 0.05% |
| OPcache hit | child-local | shared FPM master — high hit ratio |

---

## ⚖️ Apache vs Nginx Comprehensive Comparison

```
   Apache (event MPM)              Nginx (event-driven)
   ─────────────────                ────────────────────
   Process + Thread pool            Master + Worker (1 per CPU)
   Per-active-request thread        Per-worker async event loop
   Module: DSO + .htaccess          Module: compile-time + dynamic
   per-dir override (.htaccess)     central nginx.conf only
```

| Dimension | Apache | Nginx |
|-----------|--------|-------|
| Concurrency model | Process/thread (event MPM async listener) | Pure async event loop |
| Memory/connection | 0.5-2 MB (event) | 5-50 KB |
| Static file throughput | Good | **Better** (sendfile + tiny memory) |
| `.htaccess` (per-dir override) | ✅ Yes | ❌ No (central reload required) |
| Dynamic module load | ✅ DSO at runtime | ✅ since 1.9.11 |
| Reverse proxy | mod_proxy mature | upstream + load_module mature |
| HTTP/2 | mod_http2 | built-in |
| HTTP/3 (QUIC) | experimental | mainline (1.25+) |
| TLS termination | mod_ssl OK | OpenSSL/BoringSSL faster |
| WAF integration | mod_security2 native | mod_security3 / Coraza connector |
| Shared hosting/cPanel | ✅ industry default | rare |
| Legacy `mod_php` | ✅ (prefork only) | ❌ |
| WebSocket proxy | mod_proxy_wstunnel | upstream `Upgrade` header — simpler |
| Config syntax | XML-ish blocks | terse, C-like |
| Hot reload | `graceful` (USR1) | `nginx -s reload` |
| Memory at 10K idle conn | ~1-2 GB | ~50-200 MB |
| Throughput (static, 4 cores) | ~30K rps | ~80K rps |
| Throughput (PHP-FPM) | comparable | comparable |
| Documentation depth | enormous, ২০ বছর | excellent |
| Debugging ease | LogLevel module:trace8 | error_log debug |

### Recommendation matrix

```
   ┌──────────────────────────────┬──────────┐
   │ Use case                     │ Pick     │
   ├──────────────────────────────┼──────────┤
   │ cPanel shared hosting        │ Apache   │
   │ WordPress on VPS             │ Apache   │
   │ Laravel/Symfony API          │ Either   │
   │ High-RPS edge proxy          │ Nginx    │
   │ Static CDN edge              │ Nginx    │
   │ K8s ingress                  │ Nginx    │
   │ Tomcat/Java legacy           │ Apache   │
   │ HTTP/3 production            │ Nginx    │
   │ Multi-tenant per-dir rewrite │ Apache   │
   │ Microservice gateway         │ Envoy    │
   └──────────────────────────────┴──────────┘
```

---

## 🚫 Anti-patterns ও Pitfalls

### ১) `AllowOverride All` deep tree-এ

```apache
# WRONG (production high-traffic)
<Directory "/var/www">
    AllowOverride All
</Directory>
```
প্রতি request-এ ১০-১৫ stat() syscall। Solution: vhost-এ rules move করুন; `.htaccess` শুধু shared hosting-এর জন্য।

### ২) High-traffic-এ mod_php

mod_php prefork-এ আটকে রাখে; ১GB+ RAM waste; OPcache child-local। **Always PHP-FPM via mod_proxy_fcgi**।

### ৩) ProxyPreserveHost ভুলে যাওয়া

```apache
# WRONG — backend "Host: 127.0.0.1" দেখে, vhost match fail
ProxyPass "/" "http://127.0.0.1:3000/"

# RIGHT
ProxyPreserveHost On
ProxyPass "/" "http://127.0.0.1:3000/"
```

### ৪) X-Forwarded-* not set

CloudFlare/CDN পেছনে থাকলে real client IP harvest করতে চাইবেন। নিশ্চিত করুন:
```apache
RemoteIPHeader CF-Connecting-IP    # mod_remoteip
RemoteIPTrustedProxy 173.245.48.0/20
LogFormat "%a %l %u %t ..." combined   # %a = real IP after mod_remoteip
```

### ৫) `RequestReadTimeout` ছাড়া slowloris-এ ভোগা

```
Slowloris: ১টা attacker, ১০০০ TCP, প্রতিটায় ১ byte/30sec
→ MaxRequestWorkers ভরে যায়
→ legit user 503
```
**Fix:** mod_reqtimeout বাধ্যতামূলক।

### ৬) `/server-status` public

attacker-এর জন্য reconnaissance gold mine। অবশ্যই IP/auth restrict।

### ৭) `ProxyRequests On`

forward proxy enable হয় → spam relay, lateral movement vector।

### ৮) `Options +Indexes` accidental directory listing

deployment script ভুলে index.html delete করল → পুরো filesystem listing publicly visible। default-এ disable রাখুন।

### ৯) `FollowSymLinks` chroot এ symlink escape

```apache
Options +SymLinksIfOwnerMatch -FollowSymLinks
```

### ১০) `MaxRequestWorkers` calc ভুলে গিয়ে OOM

memory math করে set করুন। Linux OOM killer Apache মারলে graceful restart-ও পাবেন না।

---

## 🔄 Migration Guide: Apache → Nginx

| Apache directive / `.htaccess` rule | Nginx equivalent |
|-------------------------------------|------------------|
| `DocumentRoot /var/www/html` | `root /var/www/html;` |
| `ServerName x.com; ServerAlias y.com` | `server_name x.com y.com;` |
| `<Directory>` | `location ^~ /path/ { }` (filesystem) |
| `<Files \.php$>` | `location ~ \.php$ { }` |
| `Require all granted` | `allow all;` |
| `Require ip 1.2.3.0/24` | `allow 1.2.3.0/24; deny all;` |
| `Redirect 301 /old /new` | `return 301 /new;` (in location) |
| `RewriteRule ^/foo /bar [R=301,L]` | `rewrite ^/foo /bar permanent;` |
| `RewriteCond %{HTTPS} off / RewriteRule .*` | `if ($scheme = http) { return 301 https://$host$request_uri; }` |
| WP `RewriteRule . /index.php [L]` | `try_files $uri $uri/ /index.php?$query_string;` |
| Laravel `RewriteRule ^ index.php [L]` | `try_files $uri $uri/ /index.php?$query_string;` |
| `mod_proxy_fcgi`/`SetHandler proxy:unix:...` | `fastcgi_pass unix:/run/php/php8.2-fpm.sock;` |
| `ProxyPass / http://upstream/` | `proxy_pass http://upstream;` (in location) |
| `Header set X-Foo "bar"` | `add_header X-Foo bar always;` |
| `mod_deflate AddOutputFilter` | `gzip on; gzip_types ...;` |
| `BalancerMember`/`<Proxy balancer://>` | `upstream { server ...; }` |
| `mod_ssl + SSLCertificateFile` | `ssl_certificate /path/fullchain.pem;` |

### হারাবেন

- `.htaccess` per-directory dynamic config — nginx-এ central reload লাগবে
- 20 বছরের ঐ specific module যেটা nginx-এ port হয়নি (e.g., mod_speling, mod_dav-style WebDAV-এর কিছু feature)

### পাবেন

- ৪০-৬০% কম memory same load-এ
- Pure async → idle keep-alive cheap
- Simpler config syntax
- HTTP/3 ready
- nginx-এর native rate limit, JWT auth (njs/lua), GeoIP

### Migration sequence

```
1. Inventory: কতগুলো .htaccess, কোন rule production-critical
2. Each .htaccess → vhost block (Apache) → nginx server/location
3. Local Docker test (same backend)
4. Staging deploy, replay traffic mirror (e.g., goreplay)
5. Canary: 10% traffic via DNS weight বা LB
6. Full cutover, Apache 30 days warm standby
```

---

## ✅ Production Checklist

```
[Architecture]
□ event MPM (not prefork unless mod_php legacy)
□ MaxRequestWorkers = RAM-based calc, not default
□ MaxConnectionsPerChild > 0 (memory leak guard)
□ KeepAliveTimeout 5s, MaxKeepAliveRequests 100

[Security]
□ ServerTokens Prod, ServerSignature Off
□ TraceEnable Off
□ FileETag None (or MTime Size only)
□ mod_reqtimeout enabled (slowloris)
□ mod_security + OWASP CRS (paranoia ≥ 1)
□ /server-status & /balancer-manager IP-restricted + auth
□ ProxyRequests Off explicit
□ HSTS, X-Content-Type-Options, X-Frame-Options, CSP set
□ TLS: TLS 1.2+ only, Mozilla intermediate cipher list
□ OCSP stapling on
□ HTTP/2 enabled (Protocols h2 http/1.1)
□ mod_md or certbot auto-renew tested
□ Latest CVE patches applied; subscribe to dist security list

[Performance]
□ AllowOverride None (or minimal scope)
□ EnableSendfile On (off only for NFS)
□ HostnameLookups Off
□ mod_deflate + mod_brotli for compressible types
□ Static asset Cache-Control max-age=1y, immutable
□ mod_cache for proxy cacheable upstream

[Backend integration]
□ PHP via PHP-FPM (mod_proxy_fcgi), NOT mod_php
□ ProxyPreserveHost On for upstream host matching
□ X-Forwarded-Proto/For set correctly
□ mod_remoteip if behind CDN/CF — log real IP
□ Backend timeouts (ProxyTimeout) tuned

[Observability]
□ access_log %D (microsec) + X-Forwarded-For
□ error_log LogLevel warn (info for new deploy)
□ ExtendedStatus On (for /server-status metrics)
□ apache_exporter → Prometheus → Grafana
□ Log rotation (rotatelogs piped or logrotate + USR1)
□ mod_log_forensic for incident forensics (optional)

[Operations]
□ apachectl configtest in CI before deploy
□ Use `apachectl graceful` (USR1), never abrupt restart
□ Config in version control (Git)
□ Vhost order documented (httpd -S in CI)
□ Backup of /etc/httpd, /etc/letsencrypt
□ Disaster recovery: docker image or Ansible playbook
□ Runbook: 5xx spike, slowloris, cert renewal failure
```

---

## 🎓 সারসংক্ষেপ

- **Apache এখনও বাঁচে** — shared hosting, cPanel, WordPress, .htaccess ecosystem তাকে আজও অপরিহার্য করে
- **MPM = process model**: prefork (mod_php-only legacy), worker (thread+process), **event** (modern default — keep-alive offload)
- **`.htaccess` দ্বিধার তলোয়ার**: shared hosting-এ অপরিহার্য, single-tenant production-এ performance hit; AllowOverride scope কম রাখুন
- **আজকের stack**: event MPM + PHP-FPM via `mod_proxy_fcgi` + Let's Encrypt via `mod_md` + HTTP/2 + mod_security
- **Reverse proxy/LB**: `mod_proxy_balancer` দিয়ে multi-backend, sticky session, WebSocket, mTLS — সব সম্ভব
- **Nginx vs Apache**: high-RPS edge → nginx; multi-tenant `.htaccess` → apache; PHP backend → either
- **BD reality**: ExonHost/Eicra/AlphaNet সবাই apache-চালিত cPanel; modern startups এ nginx; legacy banking এ Apache+Tomcat AJP
- Production-এ migrate: `prefork+mod_php` → `event+PHP-FPM` — RAM ১০× কম, latency ৭× ভালো

> **শেষ কথা:** Apache শেখা মানে শুধু একটা web server শেখা না — Bangladesh-এর hosting industry-এর ভাষা শেখা।

---

## 🔗 Related Reading

- [Load Balancing](../03-system-design/load-balancing.md)
- [HTTP Protocol](../14-networking/http-protocol.md)
- [TLS/SSL](../14-networking/tls-ssl.md)
- [Security Headers](../11-security/security-headers.md)
- [Processes & Threads](../20-operating-systems/processes-threads.md)
- [I/O Models](../20-operating-systems/io-models.md)
