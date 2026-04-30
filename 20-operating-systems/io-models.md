# 🌊 I/O মডেল (I/O Models — blocking থেকে io_uring)

> **C10K সমস্যা থেকে শুরু — এক সার্ভারে ১০,০০০ concurrent connection হ্যান্ডেল করা। আজ আমরা C10M (১ কোটি)-এর কথা বলি। এই অগ্রগতি I/O model evolution-এর ফল — blocking → epoll → io_uring।**

---

## 📖 সূচিপত্র

- [I/O মডেলের পাঁচ ধরন](#-io-মডেলের-পাঁচ-ধরন)
- [Blocking I/O](#-blocking-io)
- [Non-blocking I/O](#-non-blocking-io)
- [I/O Multiplexing — select / poll / epoll / kqueue](#-io-multiplexing--select--poll--epoll--kqueue)
- [Signal-driven I/O](#-signal-driven-io)
- [Asynchronous I/O — POSIX AIO, io_uring](#-asynchronous-io--posix-aio-io_uring)
- [epoll: edge-triggered vs level-triggered](#-epoll-edge-triggered-vs-level-triggered)
- [io_uring deep dive](#-io_uring-deep-dive)
- [Node.js libuv ও Nginx-এর ভিতরে](#-nodejs-libuv-ও-nginx-এর-ভিতরে)
- [PHP stream_select ও Swoole](#-php-stream_select-ও-swoole)
- [bKash gateway — হাজার merchant connection](#-bkash-gateway--হাজার-merchant-connection)
- [তুলনা ছক ও সারমর্ম](#-তুলনা-ছক-ও-সারমর্ম)

---

## 📌 I/O মডেলের পাঁচ ধরন

প্রতিটি I/O অপারেশন (socket read উদাহরণ) দুই phase:

```
Phase 1: Wait for data       (NIC → kernel buffer)
Phase 2: Copy data           (kernel buffer → user buffer)
```

মডেলগুলোর পার্থক্য — এই দুই phase-এ user thread কী করছে।

```
┌──────────────────────┬─────────────────┬─────────────────┐
│  Model               │  Phase 1 wait   │  Phase 2 copy   │
├──────────────────────┼─────────────────┼─────────────────┤
│ Blocking             │  block          │  block          │
│ Non-blocking (poll)  │  busy-poll      │  block          │
│ I/O multiplexing     │  block on epoll │  block          │
│ Signal-driven        │  ✓ (signal)     │  block          │
│ Asynchronous (AIO)   │  ✓              │  ✓ (callback)   │
└──────────────────────┴─────────────────┴─────────────────┘
```

---

## 📌 Blocking I/O

```
User Process              Kernel
   │                        │
   ├── recvfrom() ─────────▶│
   │  (block)               │  no data yet
   │  (block)               │  ⏳⏳⏳
   │  (block)               │  data এলো!
   │  (block)               │  buffer copy চলছে
   │ ◀─── return data ──────┤
   │                        │
```

সবচেয়ে সহজ মডেল — কিন্তু ১ thread = ১ connection। ১০,০০০ connection = ১০,০০০ thread = RAM/scheduler crash।

```c
// classic Apache prefork
int s = socket(...);
bind(s, ...); listen(s);
while (1) {
    int c = accept(s, ...);     // block here
    if (fork() == 0) {
        char buf[4096];
        read(c, buf, sizeof(buf));   // block
        write(c, response, len);
        close(c);
    }
}
```

PHP-FPM worker এই blocking model-এ চলে — প্রতি worker একটি request পুরো শেষ করে।

---

## 📌 Non-blocking I/O

```c
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

while (1) {
    int n = read(fd, buf, sz);
    if (n < 0 && errno == EAGAIN) {
        // ডেটা নেই — অন্য কাজ করো
        continue;
    }
    break;
}
```

```
User Process              Kernel
   │                        │
   ├── recv() ─────────────▶│
   │ ◀─── EAGAIN ────────── │  (no data)
   │ (অন্য কাজ)              │
   ├── recv() ─────────────▶│
   │ ◀─── EAGAIN ────────── │
   │ (অন্য কাজ)              │
   ├── recv() ─────────────▶│
   │ ◀─── data ──────────── │
```

**সমস্যা**: busy-poll = CPU 100%। শুধু non-blocking দিয়ে আসল প্রোডাকশন সার্ভার বানানো যায় না — multiplexing দরকার।

---

## 📌 I/O Multiplexing — select / poll / epoll / kqueue

এক thread একসাথে অনেক fd-কে monitor করে। কার্নেল বলে দেয় কোনগুলো ready।

```
              ┌─── fd 5  (client A)
              ├─── fd 6  (client B)
   epoll_wait ├─── fd 7  (client C)            ──▶ "fd 5, fd 7 ready!"
              ├─── fd 8  (client D)
              └─── fd 9  (database)

   একটি thread → 10,000 fd সম্ভব
```

### select() — POSIX, পুরোনো

```c
fd_set rfds;
FD_ZERO(&rfds);
FD_SET(sock, &rfds);
struct timeval tv = {5, 0};
int n = select(sock+1, &rfds, NULL, NULL, &tv);
if (FD_ISSET(sock, &rfds)) { /* ready */ }
```

**সীমাবদ্ধতা**:
- `FD_SETSIZE` (default 1024) maximum
- প্রতি call-এ সম্পূর্ণ fd_set kernel-এ copy → O(N)
- linear scan — যদি ১০,০০০ fd, ১টা ready, scan ১০,০০০

### poll()

```c
struct pollfd fds[N];
fds[0].fd = sock; fds[0].events = POLLIN;
poll(fds, N, timeout_ms);
```

`FD_SETSIZE` লিমিট নেই, কিন্তু এখনও O(N)।

### epoll (Linux 2.6+)

```c
int ep = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = sock;
epoll_ctl(ep, EPOLL_CTL_ADD, sock, &ev);    // একবার add

struct epoll_event events[1024];
int n = epoll_wait(ep, events, 1024, -1);   // block, ready গুলো ফেরত
for (int i = 0; i < n; i++) {
    handle(events[i].data.fd);
}
```

**কেন তেজ**:
- কার্নেলে fd list রাখা হয় (red-black tree) — প্রতি call-এ copy লাগে না
- ready fd-এর list directly returned — O(ready), not O(total)
- `epoll_wait` blocking, semi-blocking, non-blocking তিনটাই করা যায়

### kqueue — BSD/macOS-এর equivalent

```c
int kq = kqueue();
struct kevent kev;
EV_SET(&kev, sock, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &kev, 1, NULL, 0, NULL);
```

epoll-এর চেয়ে general — file, signal, timer, process exit সবই কভার করে।

### IOCP — Windows

Windows-এর সমতুল্য। সম্পূর্ণ async (Phase 2-ও কার্নেল করে)।

---

## 📌 Signal-driven I/O

```c
fcntl(fd, F_SETOWN, getpid());
fcntl(fd, F_SETFL, O_ASYNC | O_NONBLOCK);
signal(SIGIO, handler);
// SIGIO async handler-এ
```

প্রায় কেউ ব্যবহার করে না — signal handler complexity, race-prone।

---

## 📌 Asynchronous I/O — POSIX AIO, io_uring

### POSIX AIO (`aio_read`)

স্ট্যান্ডার্ড আছে, কিন্তু Linux implementation দুর্বল — glibc-এ আসলে thread pool থেকে blocking call করে। ফলে ব্যাপক ব্যবহার হয়নি।

### Linux AIO (`io_submit`)

কেবল direct I/O (`O_DIRECT`)-এর জন্য কাজ করে। buffered I/O-তে সম্ভব fall-back to blocking।

### io_uring (Linux 5.1, ২০১৯)

আসল game-changer। ডিজাইনার Jens Axboe (block layer maintainer)। নিচে বিস্তারিত।

---

## 📌 epoll: edge-triggered vs level-triggered

### Level-Triggered (LT) — default

> "যতবার read করবো, ততবার ready বলবে যদি data থাকে।"

```
fd-এ 1000 byte এলো:
  epoll_wait → "fd ready"
  read 200 byte
  epoll_wait → "fd ready" (still 800 left)
  read 800
  epoll_wait → block (no data)
```

সহজ, forgiving। blocking-এর মতো coding pattern।

### Edge-Triggered (ET) — `EPOLLET`

> "শুধু একবার বলবো যখন state change হয়। তুমি পুরোটা drain করো।"

```
fd-এ 1000 byte এলো:
  epoll_wait → "fd ready" (একবার!)
  read 200 byte
  epoll_wait → block (state change হয়নি, কিন্তু 800 byte পড়া হয়নি!)  ⚠️
```

**সঠিক pattern**: ET-এ সবসময় loop করে EAGAIN পর্যন্ত পড়তে হবে।

```c
while (1) {
    int n = read(fd, buf, sz);
    if (n < 0 && errno == EAGAIN) break;
    if (n <= 0) break;
    process(buf, n);
}
```

| ফিচার | LT | ET |
|------|----|----|
| Default mode | ✅ | manual |
| পুরো drain লাগে? | না | হ্যাঁ |
| Spurious wakeup ঝুঁকি | বেশি | কম |
| Performance | কম | বেশি (কম syscall) |
| ব্যবহার | সাধারণ | high-perf (Nginx ET ব্যবহার করে) |

### oneshot (`EPOLLONESHOT`)

একবার fire করার পর fd auto-disable। multi-thread server-এ একই fd একাধিক worker-এ যাওয়া রোধ করে।

---

## 📌 io_uring deep dive

io_uring-এর মূল ধারণা: **kernel-user-এ shared ring buffer** যাতে syscall ছাড়াই request submit ও completion read করা যায়।

```
┌───────────────── User Space ───────────────────┐
│                                                 │
│   Submission Queue (SQ)         Completion Queue (CQ)
│   ┌──────────────────┐         ┌──────────────────┐
│   │ SQE: read fd=5    │         │ CQE: fd=5, n=200│
│   │ SQE: write fd=6   │         │ CQE: fd=6, n=10 │
│   │ SQE: accept ...   │   ◀──   │ CQE: ...        │
│   └──────────────────┘         └──────────────────┘
│         ▲                              ▲
└─────────┼──────────────────────────────┼─────────┘
          │  shared mmap'd ring          │
┌─────────┼──────────────────────────────┼─────────┐
│         ▼                              │         │
│   ┌──────────┐                  ┌──────────┐    │
│   │ Worker   │  pulls SQE       │ Notifier │    │
│   │ Threads  │  ────▶  do I/O   │  fills   │    │
│   │ (kernel) │  ──── ▶ NIC/disk │   CQE    │    │
│   └──────────┘                  └──────────┘    │
└─────────────────── Kernel ──────────────────────┘
```

### মূল API তিনটা মাত্র syscall

```c
io_uring_setup(entries, &params)    // ring তৈরি
io_uring_enter(fd, to_submit, ...)  // optional — submit/wait
io_uring_register(...)              // FD/buffer pre-register
```

আসলে most operations syscall ছাড়াই হয় — user code SQE memory-তে লেখে, kernel poll mode-এ থাকলে দেখে এবং execute করে।

### সাধারণ usage (liburing wrapper)

```c
#include <liburing.h>
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// একাধিক request batched submit
struct io_uring_sqe *sqe;
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sz, offset);

sqe = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe, fd2, buf2, sz2, offset2);

io_uring_submit(&ring);

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
// process result
io_uring_cqe_seen(&ring, cqe);
```

### Advanced features

- **SQPOLL** — kernel side poll thread, user write SQE → kernel auto-pickup → **0 syscalls**!
- **Registered FDs / buffers** — kernel pre-pin করে রাখে, প্রতি op-এ lookup এড়ানো
- **Linked SQEs** — chain operation: read → write atomically
- **Multishot accept** — একটা SQE → একাধিক CQE (অনেক client accept)
- **Network ops support** — accept, send, recv, sendmsg
- **eBPF integration** (Linux 6.x+)

### Benchmark (Jens Axboe-এর paper)

```
Workload: random 4K reads, NVMe, 8 cores
─────────────────────────────────────────────
sync read:           ~ 200K IOPS
libaio:              ~ 450K IOPS
io_uring (basic):    ~ 1.6M IOPS
io_uring (SQPOLL):   ~ 2.5M IOPS    ← syscall-free
```

### কে ব্যবহার করছে

- **scyllaDB**, **RocksDB**, **MongoDB** — disk path
- **Node.js** (libuv) — file I/O experimental
- **Nginx** — অপশনাল (since 1.25)
- **Postgres** — async I/O effort চলছে
- **Tokio (Rust)** — `tokio-uring`

> ⚠️ **Old kernel/cgroup compat**: io_uring-এ security hole পাওয়া গেছে (2023)। Container security-conscious env-এ disable হতে পারে। Linux 6.1+ এ অনেক আগের জিনিস mature।

---

## 📌 Node.js libuv ও Nginx-এর ভিতরে

### Node.js — libuv

```
        Node.js JS code
              │
              ▼
        ┌──────────────┐
        │  V8 (JS VM)   │
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │   libuv      │
        ├──────────────┤
        │ Event Loop   │ ──── epoll_wait (Linux)
        │              │ ──── kqueue (macOS)
        │              │ ──── IOCP (Windows)
        ├──────────────┤
        │ Thread Pool  │ ──── 4 default thread (UV_THREADPOOL_SIZE)
        │              │     • file system ops (fs.readFile)
        │              │     • DNS (getaddrinfo)
        │              │     • crypto (pbkdf2)
        └──────────────┘
```

**গুরুত্বপূর্ণ ব্যাপার**:
- **Network I/O** → পুরোপুরি event loop + epoll (single thread async)
- **File I/O** → libuv thread pool (epoll-এ regular file fully async না Linux-এ)
- CPU-bound JS → event loop block — `worker_threads` ব্যবহার করুন

### Node TCP server flow

```javascript
const net = require('net');
const server = net.createServer((socket) => {
    socket.on('data', (chunk) => socket.write(chunk));    // echo
});
server.listen(3000);
```

```
internally:
  socket(...) → server fd
  bind, listen
  epoll_ctl(ADD, fd, EPOLLIN | EPOLLET)
  
event loop iteration:
  epoll_wait(...) → ready fd list
    server fd ready → accept4 → new client fd
                       epoll_ctl(ADD, client_fd, EPOLLIN)
    client fd ready → read → 'data' event → JS callback
                       write → if EAGAIN, mark for EPOLLOUT
```

### Nginx

```nginx
events {
    worker_connections 4096;     # প্রতি worker
    use epoll;                   # Linux ডিফল্ট
    multi_accept on;             # accept loop drain
}
worker_processes auto;           # = CPU count
```

প্রতিটি worker একটি thread — own epoll instance। Master accepts SIGHUP → graceful reload।

```
master
  ├── worker 0 (epoll, ~10000 conn)
  ├── worker 1 (epoll, ~10000 conn)
  ├── worker 2 (epoll)
  └── worker 3 (epoll)
```

---

## 📌 PHP stream_select ও Swoole

### Vanilla PHP

PHP পাল্টা সিনক্রোনাস — কিন্তু `stream_select()` দিয়ে `select`/`poll`-এর কাছাকাছি কিছু সম্ভব।

```php
<?php
$server = stream_socket_server('tcp://0.0.0.0:8080', $errno, $errstr);
stream_set_blocking($server, false);
$clients = [];

while (true) {
    $read = array_merge([$server], $clients);
    $write = $except = null;
    
    if (stream_select($read, $write, $except, 5) > 0) {
        foreach ($read as $sock) {
            if ($sock === $server) {
                // নতুন connection
                $client = stream_socket_accept($server, 0);
                stream_set_blocking($client, false);
                $clients[] = $client;
            } else {
                $data = fread($sock, 8192);
                if ($data === '' || $data === false) {
                    fclose($sock);
                    $clients = array_filter($clients, fn($c) => $c !== $sock);
                } else {
                    fwrite($sock, "Echo: $data");
                }
            }
        }
    }
}
```

⚠️ `stream_select` (PHP) এর তলে `select()` syscall — FD_SETSIZE 1024 limit। ১০০০+ connection হলে fail।

### Swoole — PHP-এর জন্য event loop extension

```php
<?php
// install: pecl install swoole
$server = new Swoole\Server('0.0.0.0', 9501);
$server->set(['worker_num' => 4, 'max_conn' => 100000]);

$server->on('Connect', fn($s, $fd) => echo "$fd connected\n");
$server->on('Receive', function ($s, $fd, $rid, $data) {
    $s->send($fd, "Echo: $data");
});
$server->on('Close', fn($s, $fd) => echo "$fd closed\n");

$server->start();
```

Swoole epoll ব্যবহার করে — ১০০K+ concurrent connection PHP-তেই সম্ভব। Laravel Octane Swoole-এর উপর বানানো।

### ReactPHP — pure-PHP event loop

```php
<?php
$loop = React\EventLoop\Loop::get();
$server = new React\Socket\SocketServer('0.0.0.0:8080', [], $loop);

$server->on('connection', function ($conn) {
    $conn->on('data', fn($d) => $conn->write("Echo: $d"));
});
```

ReactPHP `stream_select` (default) বা `ev`/`event`/`uv` extension থাকলে সেগুলো ব্যবহার করে।

---

## 📌 bKash gateway — হাজার merchant connection

### সমস্যা

bKash payment gateway-এ একই সাথে হাজার merchant server (Daraz, Pathao, Foodpanda, Robi-Cash, ...) persistent HTTPS connection রাখে webhook দেওয়ার জন্য। Traditional Apache prefork-এ:

```
10,000 merchants × 2MB (thread stack) = 20 GB শুধু stack
+ context switch storm = CPU 90% sys
+ TLS handshake CPU = বাকিটাও শেষ
```

### সমাধান (epoll-based)

```
┌─────────────────────────────────────────────┐
│   bKash Gateway (Nginx + Node.js cluster)    │
├─────────────────────────────────────────────┤
│                                              │
│   Nginx (TLS termination)                    │
│     ├── 8 worker × 10,000 conn = 80,000 conn│
│     └── epoll ET, sendfile, SO_REUSEPORT    │
│                                              │
│         ↓ keepalive HTTP/2                   │
│                                              │
│   Node.js cluster (8 worker)                 │
│     ├── libuv epoll                          │
│     ├── Redis pub/sub for cross-worker fanout│
│     └── per worker ~ 8000 conn easily        │
│                                              │
│   Backend (Postgres, Kafka, etc.)            │
└─────────────────────────────────────────────┘
```

### Tuning checklist

```bash
# OS limits
ulimit -n 1000000
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
sysctl -w net.ipv4.ip_local_port_range="10000 65000"
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=15
sysctl -w fs.file-max=2000000

# Nginx
worker_rlimit_nofile 200000;
worker_connections 65535;
events { use epoll; multi_accept on; }

# Kernel feature
echo 1 > /proc/sys/net/ipv4/tcp_fastopen
```

### Result

```
Apache prefork:        ~ 2,000 conn/server, 90% CPU
Nginx + Node + epoll:  ~ 80,000 conn/server, 30% CPU
Node + io_uring (PoC): ~ 200,000 conn/server, 25% CPU
```

---

## 📊 তুলনা ছক ও সারমর্ম

```
┌──────────────────┬──────────┬──────────┬──────────┬──────────┬───────────┐
│  Model           │ select   │  poll    │  epoll   │ io_uring │  IOCP     │
├──────────────────┼──────────┼──────────┼──────────┼──────────┼───────────┤
│ OS               │ POSIX    │ POSIX    │ Linux    │ Linux    │ Windows   │
│ FD limit         │ 1024     │ none     │ none     │ none     │ none      │
│ Time complexity  │  O(N)    │  O(N)    │ O(ready) │ O(ready) │ O(ready)  │
│ Phase 2 async    │  ❌      │  ❌      │  ❌      │  ✅      │  ✅       │
│ Syscall per op   │   1+     │   1+     │   1      │  ~0 (SQ) │   1       │
│ Thread-safe add  │  ❌      │  ❌      │  ✅      │  ✅      │  ✅       │
│ Edge-trigger     │  ❌      │  ❌      │  ✅      │  N/A     │  N/A      │
│ Used by          │  Old     │  legacy  │ Nginx,   │ ScyllaDB │ IIS,      │
│                  │  apps    │ legacy   │ Node     │ RocksDB  │ SQL Server│
└──────────────────┴──────────┴──────────┴──────────┴──────────┴───────────┘
```

```
┌────────────────────────────────────────────────────────────────────┐
│ ১. Blocking → 1 conn = 1 thread; OK পর্যন্ত ~1000 conn               │
│ ২. select/poll = O(N), legacy                                       │
│ ৩. epoll = Linux-এর ক্ল্যাসিক C10K solution; ET দিয়ে সর্বোচ্চ perf   │
│ ৪. io_uring = ভবিষ্যৎ; syscall-less, full async, file+net           │
│ ৫. Node.js network = epoll, file = thread pool                      │
│ ৬. Nginx = epoll ET + SO_REUSEPORT + sendfile                       │
│ ৭. PHP scaling: vanilla = stream_select (limited), Swoole = epoll  │
│ ৮. ulimit -n, somaxconn, tcp_max_syn_backlog — must-tune            │
│ ৯. ET mode-এ EAGAIN পর্যন্ত drain — কোডে ভুলবেন না                  │
│ ১০. Phase 1 vs Phase 2 — true async শুধু io_uring/IOCP-এ            │
└────────────────────────────────────────────────────────────────────┘
```

> 💡 **বাংলাদেশী context**: Daraz checkout, bKash gateway, Pathao real-time tracking — সব production system epoll-based stack-এ চলে। Apache এখনো অনেক BD shared hosting-এ আছে কিন্তু high-traffic site-এ Nginx/Node + epoll standard।

> ➡️ **পরবর্তী section**: file-systems.md, ipc.md, namespaces-cgroups.md, linux-kernel-architecture.md, boot-init.md, kernel-networking-stack.md (Part B তে আসবে)।
