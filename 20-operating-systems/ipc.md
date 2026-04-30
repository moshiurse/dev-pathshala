# 🔗 IPC — Inter-Process Communication

> দুটি প্রসেস একে অপরের সাথে কথা বলবে কীভাবে? Pipe? Socket? Shared memory? Signal?
> Nginx ↔ PHP-FPM, Redis client ↔ Redis server, কন্টেইনার শাটডাউন — সব IPC-এর প্রয়োগ।

---

## 📖 সূচিপত্র

- [কেন IPC দরকার](#-কেন-ipc-দরকার)
- [IPC mechanism তুলনা](#-ipc-mechanism-তুলনা)
- [Pipe — anonymous ও named (FIFO)](#-pipe--anonymous-ও-named-fifo)
- [Unix Domain Socket](#-unix-domain-socket)
- [Signal](#-signal)
- [Shared Memory](#-shared-memory)
- [Message Queue](#-message-queue)
- [Semaphore ও futex](#-semaphore-ও-futex)
- [eventfd, signalfd, memfd](#-eventfd-signalfd-memfd)
- [D-Bus overview](#-d-bus-overview)
- [Real-world: Nginx + PHP-FPM, Redis UDS, Graceful shutdown](#-real-world-nginx--php-fpm-redis-uds-graceful-shutdown)
- [Pitfall ও তুলনা](#-pitfall-ও-তুলনা)

---

## 📌 কেন IPC দরকার

প্রসেসগুলো **আলাদা virtual address space**-এ চলে — একে অপরের memory দেখতে পায় না (security)। কিন্তু web server, DB, queue worker — সবাই একে অপরের সাথে data exchange করে। এজন্য **kernel-mediated** যোগাযোগ লাগে।

```
   ┌─────────────┐       ┌─────────────┐
   │  Process A  │       │  Process B  │
   │  (Nginx)    │       │  (PHP-FPM)  │
   │             │       │             │
   │  নিজস্ব VA   │       │  নিজস্ব VA  │
   └──────┬──────┘       └──────┬──────┘
          │                     │
          ▼  syscall             syscall ▼
   ┌──────────────────────────────────────┐
   │             Linux Kernel              │
   │  pipe / socket / shm / signal / ...   │
   └──────────────────────────────────────┘
```

---

## 📌 IPC mechanism তুলনা

| Mechanism             | Direction  | Kernel copy | Speed   | Cross-machine | Use case                      |
|-----------------------|------------|-------------|---------|----------------|-------------------------------|
| **Anonymous pipe**    | one-way    | ✅          | 🚶      | ❌             | parent-child (`grep | sort`) |
| **Named pipe (FIFO)** | one-way    | ✅          | 🚶      | ❌             | unrelated process             |
| **Unix Domain Socket**| two-way    | ✅          | 🚶🚶    | ❌             | Nginx ↔ PHP-FPM               |
| **TCP/UDP socket**    | two-way    | ✅          | 🚶      | ✅             | networked services            |
| **Signal**            | one-way    | minimal     | ⚡      | ❌             | event/kill notification       |
| **Shared memory**     | two-way    | ❌ (zero copy)| ⚡⚡    | ❌             | high-throughput, market data  |
| **Message queue**     | one-way    | ✅          | 🚶      | ❌             | async, prioritized message    |
| **Semaphore/futex**   | sync only  | minimal     | ⚡      | ❌             | mutual exclusion              |
| **eventfd/signalfd**  | notification| ✅         | ⚡      | ❌             | event loop integration        |
| **D-Bus**             | RPC bus    | ✅✅        | 🐢      | ❌             | systemd, desktop services     |

---

## 📌 Pipe — anonymous ও named (FIFO)

### Anonymous pipe

`pipe()` syscall দুটি fd ফেরত দেয় — `[0]=read end`, `[1]=write end`। সাধারণত `fork()` করার আগে create করা হয়, যাতে child inherit করে।

```
   parent process              child process
   ┌──────────┐                 ┌──────────┐
   │ fd[1] ─► │ ──── kernel ─── │ ◄─ fd[0] │
   │  write   │     pipe        │   read   │
   └──────────┘                 └──────────┘
```

```bash
# Shell-এ pipe প্রতিদিন
$ ps aux | grep nginx | wc -l
#   producer  filter   consumer
```

```php
<?php
// PHP — proc_open দিয়ে pipe
$descriptors = [
    0 => ['pipe', 'r'],   // child stdin
    1 => ['pipe', 'w'],   // child stdout
    2 => ['pipe', 'w'],   // child stderr
];
$proc = proc_open('grep error', $descriptors, $pipes);
fwrite($pipes[0], "info: ok\nerror: db failed\nwarn: slow\n");
fclose($pipes[0]);
echo stream_get_contents($pipes[1]);   // "error: db failed"
proc_close($proc);
```

```javascript
// Node.js
const { spawn } = require('child_process');
const grep = spawn('grep', ['error']);

grep.stdout.on('data', d => console.log('out:', d.toString()));
grep.stdin.write('info: ok\nerror: db failed\nwarn: slow\n');
grep.stdin.end();
```

### Named pipe (FIFO)

ফাইলসিস্টেমে একটি **পথ থাকে** যাতে unrelated process-ও open করতে পারে।

```bash
mkfifo /tmp/jobs.fifo

# Terminal 1 — consumer
cat /tmp/jobs.fifo | while read job; do
    echo "processing: $job"
done

# Terminal 2 — producer
echo "send-otp:01700000000" > /tmp/jobs.fifo
echo "send-otp:01711111111" > /tmp/jobs.fifo
```

> 💡 **পরিমিত:** FIFO buffer ডিফল্ট 64KB। বেশি লিখতে গেলে writer block। কোনো reader না থাকলে writer SIGPIPE পাবে।

---

## 📌 Unix Domain Socket

UDS local TCP-এর মতো API (socket/bind/listen/accept) কিন্তু **kernel-level loopback ছাড়াই** — শুধু VFS path।

```
┌─────────────────────────────────────────────────────┐
│  Nginx                          PHP-FPM             │
│   ┌──────┐    UDS path:          ┌──────┐          │
│   │ sock │◄── /run/php-fpm.sock ►│ sock │          │
│   └──────┘                       └──────┘          │
└─────────────────────────────────────────────────────┘
            no IP stack, no checksum, no port
            only kernel buffer copy
```

**TCP loopback vs UDS:**

| Metric                | TCP 127.0.0.1 | UDS              |
|-----------------------|----------------|-------------------|
| Throughput            | ~5 GB/s       | ~10 GB/s          |
| Latency (1KB)         | ~30 µs        | ~10 µs            |
| Header overhead       | TCP+IP        | নেই               |
| Permission control    | port-based    | filesystem perm   |

```nginx
# /etc/nginx/sites-available/laravel
location ~ \.php$ {
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;   # UDS
    # বদলে: fastcgi_pass 127.0.0.1:9000;          # TCP
    fastcgi_index index.php;
    include fastcgi_params;
}
```

```javascript
// Node.js — UDS server
const net = require('net');
const fs = require('fs');

const SOCK = '/tmp/api.sock';
try { fs.unlinkSync(SOCK); } catch {}

const server = net.createServer(conn => {
    conn.on('data', d => conn.write('ack: ' + d));
});
server.listen(SOCK, () => {
    fs.chmodSync(SOCK, 0o660);   // permission lock down
    console.log('listening on', SOCK);
});

// Client
const client = net.createConnection(SOCK);
client.write('hello');
```

```php
<?php
// PHP UDS client
$fp = stream_socket_client('unix:///tmp/api.sock', $errno, $err, 5);
fwrite($fp, "ping");
echo fread($fp, 1024);
fclose($fp);
```

> 🇧🇩 **Pathao API tier:** Nginx → PHP-FPM সব node-এ UDS ব্যবহার করে। ৩০% latency কমেছে এবং `ip_local_port_range` exhaustion (ephemeral port leak) সমস্যা নেই।

---

## 📌 Signal

Signal হলো **asynchronous notification** — kernel একটি প্রসেসকে "event" সম্পর্কে জানায়। ৩১টি standard + 32টি real-time signal।

```
  kernel detects event ─► সংশ্লিষ্ট process-এর pending mask-এ bit set
                                                │
                                                ▼
  process schedule হলে কার্নেল signal handler call করে
  (আগে current execution interrupt করে)
```

### গুরুত্বপূর্ণ signals

| Signal      | Number | Default action       | বর্ণনা                                |
|-------------|--------|----------------------|----------------------------------------|
| `SIGHUP`    | 1      | terminate            | terminal hangup; daemon-এ "reload"     |
| `SIGINT`    | 2      | terminate            | Ctrl+C                                 |
| `SIGQUIT`   | 3      | core dump            | Ctrl+\                                 |
| `SIGKILL`   | 9      | terminate (uncatchable!)| force kill                          |
| `SIGUSR1`   | 10     | terminate            | user-defined #1 (Nginx: log reopen)   |
| `SIGUSR2`   | 12     | terminate            | user-defined #2                        |
| `SIGPIPE`   | 13     | terminate            | broken pipe (write to closed)          |
| `SIGTERM`   | 15     | terminate            | polite stop                            |
| `SIGCHLD`   | 17     | ignore               | child exited                           |
| `SIGSTOP`   | 19     | stop (uncatchable)   | pause                                  |
| `SIGCONT`   | 18     | continue             | resume                                 |

```bash
$ kill -l            # সব signal
$ kill -TERM 1234    # polite
$ kill -KILL 1234    # nuclear
$ kill -HUP $(cat /run/nginx.pid)    # reload config
$ kill -USR1 $(cat /run/nginx.pid)   # reopen log files
```

### Signal handler — async-signal-safe

Signal handler **যেকোনো সময়** চলতে পারে — main code-এর মাঝে। তাই ভেতরে শুধু **async-signal-safe function** call করা যায় (`write`, `_exit`, না `printf`/`malloc`)।

```php
<?php
// PHP graceful shutdown — Pathao worker
declare(ticks=1);   // tick-based delivery (PHP 7+ এ pcntl_async_signals preferred)
pcntl_async_signals(true);

$running = true;

pcntl_signal(SIGTERM, function ($sig) use (&$running) {
    error_log("SIGTERM received, finishing current job...");
    $running = false;
});

pcntl_signal(SIGINT, function () { exit(0); });

while ($running) {
    $job = $queue->pop(timeout: 5);
    if ($job) processRide($job);
}
echo "clean exit\n";
```

```javascript
// Node.js — Express graceful shutdown (K8s rolling update)
const http = require('http');
const server = http.createServer(app);

server.listen(3000);

const shutdown = (signal) => {
    console.log(`${signal} received, draining...`);
    server.close(err => {
        // existing connection শেষ করে exit
        if (err) { console.error(err); process.exit(1); }
        // DB pool, redis ছেড়ে দিন
        Promise.all([db.end(), redis.quit()])
            .then(() => process.exit(0));
    });
    // safety net
    setTimeout(() => process.exit(1), 10_000).unref();
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

> ⚠️ **K8s pattern:** Pod terminate করলে **SIGTERM** যায়, ৩০s `terminationGracePeriodSeconds` পরে **SIGKILL**। হ্যান্ডলার না থাকলে in-flight request lost।

### `posix_kill` ও `setitimer`

```php
<?php
// নিজেকে SIGUSR1 পাঠান (test)
posix_kill(posix_getpid(), SIGUSR1);

// একটি timer signal
pcntl_alarm(5);    // 5 sec পরে SIGALRM
pcntl_signal(SIGALRM, fn() => print("timeout\n"));
sleep(10);
```

---

## 📌 Shared Memory

দুটি প্রসেস **একই physical RAM page** map করে — kernel copy ছাড়াই data exchange। সবচেয়ে দ্রুত IPC।

```
   Process A                    Process B
   ┌─────────────┐              ┌─────────────┐
   │ VA: 0x7f00..│              │ VA: 0x7e90..│
   │     │       │              │     │       │
   └─────┼───────┘              └─────┼───────┘
         └──────────┬───────────────┘
                    ▼
         ┌────────────────────┐
         │ shared physical    │
         │ memory page        │
         └────────────────────┘
```

### POSIX shared memory (`shm_open`)

```c
// C — POSIX shm
int fd = shm_open("/market_data", O_CREAT|O_RDWR, 0660);
ftruncate(fd, 4096);
void *p = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
strcpy(p, "BDT/USD 109.5");
```

`/dev/shm/market_data` ফাইলসিস্টেমে দেখা যাবে।

```bash
$ ls -l /dev/shm
-rw-rw---- 1 trader trader 4096 ... market_data

$ ipcs -m    # System V SHM listing (পুরনো API)
```

```php
<?php
// PHP-তে System V SHM
$key = ftok(__FILE__, 'a');
$shm = shmop_open($key, 'c', 0660, 1024);
shmop_write($shm, 'BDT/USD 109.5', 0);
echo shmop_read($shm, 0, 14);
shmop_close($shm);
```

> 💡 PHP-এর **APCu** opcache shared memory-এই রাখে। তাই FPM workers একই cache দেখে।

### System V (legacy) — `shmget`/`shmat`

```bash
$ ipcs -m
$ ipcrm -m <shmid>     # cleanup
```

> ⚠️ orphan shm leak common — process crash হলে kernel manually cleanup করতে হয়।

---

## 📌 Message Queue

structured message + priority। দুটি API:

- **POSIX mqueue** — `mq_open`, `mq_send`, `mq_receive`। `/dev/mqueue` mount-এ দেখা যায়।
- **System V** — `msgget`, `msgsnd`, `msgrcv`। `ipcs -q`-এ দেখা যায়।

```bash
# POSIX mqueue mount
mkdir /dev/mqueue
mount -t mqueue none /dev/mqueue
ls /dev/mqueue/
```

```c
// C — POSIX mqueue
mqd_t mq = mq_open("/orders", O_CREAT|O_WRONLY, 0660, NULL);
mq_send(mq, "order:42", 9, 5);   // priority 5
```

> 💡 প্রোডাকশনে আজকাল userspace queue (Redis, RabbitMQ, Kafka) বেশি জনপ্রিয় — kernel mqueue limit ছোট (default 10 msg)।

---

## 📌 Semaphore ও futex

**Semaphore** — counting lock। resource-এর সংখ্যা track করে।

```c
// POSIX semaphore (named)
sem_t *s = sem_open("/db_pool", O_CREAT, 0660, 10);   // 10 slot
sem_wait(s);    // -1 (block যদি 0)
// ... use connection ...
sem_post(s);    // +1
sem_close(s);
```

**futex (fast userspace mutex)** — userspace-এ atomic op, contention হলেই kernel-এ যায়। glibc `pthread_mutex` এর ভিত্তি।

```bash
# একটি locked process trace করুন
$ strace -e futex -p 1234
futex(0x7f8a..., FUTEX_WAIT_PRIVATE, 2, NULL) = ...
```

> 🇧🇩 Daraz inventory service-এ DB connection pool slot allocate করতে semaphore। contention 100k QPS-এ বাড়লে futex contention দেখা যায় `perf top`-এ।

---

## 📌 eventfd, signalfd, memfd

আধুনিক Linux IPC primitives — **fd-based**, যাতে event loop (`epoll`, `select`) integrate করা যায়।

### eventfd — userspace counter event

```c
int efd = eventfd(0, EFD_NONBLOCK);
// thread A
write(efd, &(uint64_t){1}, 8);   // ++
// thread B
uint64_t val;
read(efd, &val, 8);    // resets to 0
```

Node.js libuv internally signal pipe-এর বদলে eventfd ব্যবহার করে inter-thread wakeup-এ।

### signalfd — signal-কে fd হিসেবে read

ক্লাসিক signal handler async-signal-safe সমস্যা এড়ায়। main loop synchronously signal read করে।

```c
sigset_t mask; sigemptyset(&mask); sigaddset(&mask, SIGTERM);
sigprocmask(SIG_BLOCK, &mask, NULL);
int sfd = signalfd(-1, &mask, 0);
// epoll_wait এ sfd—event এলে synchronous read
```

### memfd — anonymous file in RAM

```c
int fd = memfd_create("scratch", 0);
ftruncate(fd, 1<<20);
// pass to child via fork+execve, FD inherit
```

container runtime (runc) seal করা memfd ব্যবহার করে যাতে কন্টেইনার নিজের binary tamper করতে না পারে।

---

## 📌 D-Bus overview

D-Bus হলো **message bus** — একটি broker daemon (`dbus-daemon`) যার মাধ্যমে desktop ও system services কথা বলে। systemd, NetworkManager, BlueZ, PulseAudio সবাই D-Bus user।

```
   App ──► dbus-daemon (broker) ──► Service
                  │
              session bus / system bus
```

```bash
$ busctl list                          # সব service
$ busctl tree org.freedesktop.systemd1
$ busctl call org.freedesktop.systemd1 \
  /org/freedesktop/systemd1 \
  org.freedesktop.systemd1.Manager \
  ListUnits
```

> 💡 server-side application development-এ D-Bus সাধারণত দরকার পড়ে না, কিন্তু systemd integration-এ লাগে (e.g. `systemctl restart` actually D-Bus call)।

---

## 📌 Real-world: Nginx + PHP-FPM, Redis UDS, Graceful shutdown

### 1. Nginx ↔ PHP-FPM via UDS — production tuning

```ini
; /etc/php/8.3/fpm/pool.d/www.conf
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
listen.backlog = 511

pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
```

```nginx
# /etc/nginx/nginx.conf
upstream php-fpm {
    server unix:/run/php/php8.3-fpm.sock;
    keepalive 32;     # keep UDS connections alive
}
```

### 2. Redis via UDS — bKash session cache

```ini
# /etc/redis/redis.conf
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
port 0     # TCP off! শুধু UDS
```

```php
<?php
$redis = new Redis();
$redis->connect('/var/run/redis/redis.sock');   // UDS path
$redis->set('session:user:42', json_encode($data));
```

Latency `127.0.0.1:6379`-এ ~80µs → UDS-এ ~30µs। 10k req/s-এ noticeable CPU saving।

### 3. Graceful shutdown ও PID 1 problem (Pathao K8s)

K8s pod delete:
```
kubectl delete pod web-7b   ──► kubelet sends SIGTERM
                                 │
                                 ▼ (preStop hook optional)
                                Container PID 1 — যদি `node`/`php` সরাসরি, signal পায়
                                 যদি `bash -c "node app.js"`, bash signal forward করে না!
                                 │
                                 ▼ 30s wait
                                kubelet sends SIGKILL
```

**সমাধান — `tini`/`dumb-init` PID 1 হিসেবে:**

```dockerfile
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

tini শুধু signal proxy ও zombie reap করে। Node ঠিকঠাক SIGTERM পায়, graceful shutdown হয়।

---

## 📌 Pitfall ও তুলনা

### ❌ Pitfall 1: SIGPIPE crash

Reader pipe বন্ধ করে দিলে writer-এর `write()` SIGPIPE পায় — default action **terminate**!

```php
<?php
// fix
pcntl_signal(SIGPIPE, SIG_IGN);   // ignore, write fails with EPIPE
```

```javascript
// Node-এ ডিফল্টে EPIPE error event আসে — handle করুন
stream.on('error', err => { if (err.code !== 'EPIPE') throw err; });
```

### ❌ Pitfall 2: Signal handler-এ unsafe code

```php
// ❌ খারাপ — handler-এ DB connection
pcntl_signal(SIGTERM, function () {
    DB::insert(...);    // re-entrancy risk!
});

// ✅ ভালো — flag set, main loop check
$shouldStop = false;
pcntl_signal(SIGTERM, fn() => $shouldStop = true);
while (!$shouldStop) { ... }
```

### ❌ Pitfall 3: SHM leak

System V `shmget` create করার পর process crash হলে segment **persistent** থাকে। reboot ছাড়া যাবে না।

```bash
ipcs -m
ipcrm -m <shmid>

# RemoveIPC=yes (systemd) — user logout-এ cleanup
```

### ❌ Pitfall 4: Bash wrapper signal লস

```dockerfile
# ❌ shell form — bash PID 1, child PID 2, SIGTERM bash পায়, forward করে না
CMD node server.js     

# ✅ exec form — node সরাসরি PID 1
CMD ["node", "server.js"]
```

### ❌ Pitfall 5: ulimit/মেসেজ-কিউ ছোট

```bash
# POSIX mqueue limits
$ cat /proc/sys/fs/mqueue/msg_max
10
$ cat /proc/sys/fs/mqueue/msgsize_max
8192

# বাড়ান
echo 100  > /proc/sys/fs/mqueue/msg_max
echo 65536 > /proc/sys/fs/mqueue/msgsize_max
```

---

## 🎯 সংক্ষেপে

- **Pipe/FIFO** simple unidirectional, parent-child বা CLI pipeline।
- **UDS** সবচেয়ে practical local IPC — Nginx↔PHP-FPM, Redis, MySQL সব সাপোর্ট করে।
- **Signal** event/control — `SIGTERM` graceful, `SIGKILL` instant। handler-এ async-safe code।
- **Shared memory** zero-copy — ultra-low latency (HFT, market data, APCu)।
- **Semaphore/futex** synchronization primitive।
- **eventfd/signalfd/memfd** — modern fd-based, epoll-friendly।
- **PID 1 problem** কন্টেইনার-এ critical — `tini` ব্যবহার করুন।

> 🇧🇩 **Real-world signature:** Pathao K8s — `tini` + Node `SIGTERM` handler + 30s grace = zero-downtime rolling deploy।  
> bKash Redis UDS — local cache hit latency ~30µs।  
> Daraz PHP-FPM UDS pool — per-request port ephemeral exhaustion দূর।
