# 🔌 সিস্টেম কল (System Calls)

> **আপনার PHP `file_get_contents()` বা Node.js `fs.readFile()` যত abstraction-ই হোক — শেষে সেটা একটা `read()` syscall। User space ও kernel space-এর মাঝে এই syscall-ই একমাত্র সেতু।**

---

## 📖 সূচিপত্র

- [User space vs Kernel space](#-user-space-vs-kernel-space)
- [Syscall mechanism — int 0x80 → syscall → vDSO](#-syscall-mechanism--int-0x80--syscall--vdso)
- [Syscall ABI (x86_64 Linux)](#-syscall-abi-x86_64-linux)
- [সাধারণ syscall তালিকা](#-সাধারণ-syscall-তালিকা)
- [PHP/JS উদাহরণ → কোন syscall](#-phpjs-উদাহরণ--কোন-syscall)
- [Syscall এর খরচ](#-syscall-এর-খরচ)
- [Syscall batching / vectored I/O](#-syscall-batching--vectored-io)
- [strace, ltrace, perf দিয়ে trace](#-strace-ltrace-perf-দিয়ে-trace)
- [seccomp — syscall filtering](#-seccomp--syscall-filtering)
- [Pitfalls](#-pitfalls)

---

## 📌 User space vs Kernel space

CPU-তে protection ring আছে। x86-এ ring 0 (kernel, full privilege) থেকে ring 3 (user, restricted)।

```
                  ┌──────────────────────────────────┐
                  │         RING 3 — User Mode        │
                  │   PHP, Node, Nginx, MySQL, ...    │
                  │                                   │
                  │   ✗ direct hardware access নিষেধ  │
                  │   ✗ অন্য প্রসেসের memory পড়া নিষেধ │
                  │   ✗ অর্ধেক CPU instruction নিষেধ   │
                  └──────────────┬───────────────────┘
                                 │  syscall (gate)
                                 ▼
                  ┌──────────────────────────────────┐
                  │         RING 0 — Kernel Mode      │
                  │   Linux kernel, drivers           │
                  │                                   │
                  │   ✓ সব hardware access            │
                  │   ✓ সব প্রসেসের page table        │
                  │   ✓ সব CPU instruction            │
                  └──────────────────────────────────┘
```

User mode থেকে kernel mode-এ যাওয়ার মাত্র তিনটি উপায়:
1. **Syscall** (intentional) — `read()`, `write()`, ...
2. **Exception** (CPU trap) — divide-by-zero, page fault
3. **Hardware interrupt** — NIC packet, disk done, timer tick

Syscall ছাড়া user code কখনো hardware ছোঁয় না — সব ফাইল, সব সকেট, সব মেমোরি allocation কার্নেলের অনুমোদনে।

---

## 📌 Syscall mechanism — int 0x80 → syscall → vDSO

### ইতিহাস (x86 evolution)

```
1. পুরোনো (i386):     int 0x80
   ─────────────────
   • software interrupt 0x80 trigger
   • IDT (Interrupt Descriptor Table) দেখে kernel handler
   • slow — ~1000+ cycles

2. P6+ (Pentium Pro):  sysenter / sysexit
   ────────────────────
   • Intel-only fast path
   • dedicated MSR register
   • ~ 200-400 cycles

3. AMD64 / আধুনিক:    syscall / sysret instruction
   ──────────────────
   • single instruction
   • kernel entry point MSR-এ stored
   • ~100-150 cycles (overhead only)

4. vDSO (virtual Dynamic Shared Object):
   ─────────────────────────────────────
   • কিছু "syscall" আসলে syscall না!
   • gettimeofday(), clock_gettime() — kernel মেমরি page
     user space এ map করে — pure userspace read
   • 0 ring transition, ~10ns
```

### vDSO inspect

```bash
$ cat /proc/self/maps | grep vdso
7ffd1bbe3000-7ffd1bbe5000 r-xp 00000000 00:00 0   [vdso]

$ ldd /bin/ls
        linux-vdso.so.1 (0x00007ffe...)   ← এটি actual file না, kernel-mapped
        libc.so.6 ...
```

vDSO না থাকলে Node.js-এর `Date.now()` প্রতিবার একটা syscall — অনেক logging-heavy app-এ ১০-১৫% overhead হতো।

---

## 📌 Syscall ABI (x86_64 Linux)

```
syscall instruction এর আগে:
  rax = syscall number             (e.g., 0 = read, 1 = write)
  rdi = arg1                       (e.g., fd)
  rsi = arg2                       (e.g., buf pointer)
  rdx = arg3                       (e.g., count)
  r10 = arg4
  r8  = arg5
  r9  = arg6

  syscall   ← এই instruction CPU mode flip করে kernel-এ যায়
  
syscall এর পরে:
  rax = return value (negative = -errno on error)
```

### একটি minimal syscall (assembly)

```asm
; write(1, "hello\n", 6)
mov rax, 1          ; sys_write
mov rdi, 1          ; stdout
lea rsi, [rel msg]
mov rdx, 6
syscall
```

C code এ এটা সাধারণত glibc wrapper দিয়ে:

```c
#include <unistd.h>
write(1, "hello\n", 6);          // glibc wraps the syscall
```

বা সরাসরি:

```c
#include <sys/syscall.h>
syscall(SYS_write, 1, "hello\n", 6);
```

---

## 📌 সাধারণ syscall তালিকা

Linux x86_64 এ 400+ syscall আছে (`/usr/include/asm/unistd_64.h`)। গুরুত্বপূর্ণগুলো:

### File / IO

| syscall | কাজ | C func |
|---------|-----|--------|
| `read(fd, buf, n)` | fd থেকে n বাইট পড়া | `read()` |
| `write(fd, buf, n)` | fd-এ লেখা | `write()` |
| `open(path, flags)` | ফাইল ওপেন → fd | `open()` |
| `openat(dirfd, path, flags)` | relative open | `openat()` |
| `close(fd)` | fd বন্ধ | `close()` |
| `lseek(fd, offset, whence)` | পয়েন্টার সরানো | `lseek()` |
| `stat(path, &buf)` | ফাইল info | `stat()` |
| `fsync(fd)` | dirty page disk-এ flush | `fsync()` |
| `mmap(...)` | file/anon memory map | `mmap()` |
| `munmap(addr, len)` | unmap | `munmap()` |
| `pipe(fds)` | pipe তৈরি | `pipe()` |
| `dup2(old, new)` | fd duplicate | redirection |
| `readv/writev` | vectored I/O | scatter/gather |
| `sendfile(out, in, ...)` | zero-copy file → socket | `sendfile()` |

### Process

| syscall | কাজ |
|---------|-----|
| `fork()` (clone) | নতুন প্রসেস |
| `execve(path, argv, envp)` | নতুন প্রোগ্রাম replace |
| `wait4(pid, ...)` | child reap |
| `exit_group(status)` | প্রসেস শেষ |
| `kill(pid, sig)` | signal পাঠানো |
| `getpid()`, `getuid()` | self id |
| `setrlimit()` | ulimit |

### Memory

| syscall | কাজ |
|---------|-----|
| `brk(addr)` | heap এর top বাড়ানো |
| `mmap(NULL, ...)` | anonymous mapping (বড় malloc) |
| `mprotect(addr, len, prot)` | permission বদলানো |
| `madvise(addr, len, hint)` | kernel-কে hint |

### Network

| syscall | কাজ |
|---------|-----|
| `socket(domain, type, proto)` | নতুন socket fd |
| `bind`, `listen`, `accept` | server side |
| `connect` | client side |
| `send`, `recv`, `sendmsg`, `recvmsg` | data transfer |
| `setsockopt` | TCP_NODELAY, SO_REUSEPORT etc |

### Sync / Concurrency

| syscall | কাজ |
|---------|-----|
| `futex(addr, op, val, ...)` | fast userspace mutex (এর উপর pthread_mutex) |
| `epoll_create1`, `epoll_ctl`, `epoll_wait` | event multiplexing |
| `eventfd`, `signalfd`, `timerfd_create` | fd-ifying events |
| `nanosleep`, `clock_nanosleep` | sleep |

### Modern

| syscall | কাজ |
|---------|-----|
| `io_uring_setup/_enter/_register` | async I/O ring (Linux 5.1+) |
| `clone3` | extended clone |
| `pidfd_open`, `pidfd_send_signal` | race-free PID handling |

---

## 📌 PHP/JS উদাহরণ → কোন syscall

### PHP

```php
<?php
$content = file_get_contents('/etc/passwd');
//                ↓ ঘটে যা
// open("/etc/passwd", O_RDONLY) → fd
// fstat(fd, &st)                ← সাইজ জানতে
// mmap(NULL, sz, PROT_READ, MAP_SHARED, fd, 0)   বা read() loop
// close(fd)

$file = fopen('/var/log/app.log', 'a');
fwrite($file, "log entry\n");
//   ↓
// open(...O_WRONLY|O_CREAT|O_APPEND...) 
// write(fd, "log entry\n", 10)
fclose($file);
//   ↓ close(fd)
```

### Node.js

```javascript
const fs = require('fs');

// async — libuv thread pool থেকে syscall হয়
fs.readFile('/etc/hostname', (err, data) => {
  // open, fstat, read, close — thread pool worker-এ
});

// sync — main thread block
const data = fs.readFileSync('/etc/hostname');

// HTTP server
const http = require('http');
http.createServer((req, res) => {
  res.end('hello');
}).listen(3000);
//  ↓ ঘটে
// socket(AF_INET, SOCK_STREAM, ...) → server fd
// bind, listen
// epoll_create1
// epoll_ctl(ADD, server_fd)
// epoll_wait(...)         ← request আসা পর্যন্ত block
// accept4 → client_fd
// epoll_ctl(ADD, client_fd)
// read(client_fd, ...)    ← request data
// write(client_fd, ...)   ← response
// close(client_fd)
```

---

## 📌 Syscall এর খরচ

```
ফাংশন কল (regular):     ~ 1 ns
syscall (modern):       ~ 100-300 ns (no work)
syscall + small read:   ~ 500 ns - 1 μs
syscall blocking:       সম্পূর্ণ context switch

Comparison:
  L1 cache access:      0.5 ns
  Memory access:        100 ns
  syscall overhead:     200 ns       ← এক syscall = প্রায় ১টা memory access
  Disk read (NVMe):     20 μs
  Disk read (HDD):      10 ms
  Network RTT (LAN):    1 ms
```

### Spectre/Meltdown mitigation এর ধাক্কা

KPTI (Kernel Page Table Isolation), retpoline ইত্যাদির পর syscall খরচ ~2-3x বেড়েছে কিছু পুরোনো CPU-তে। ফলে syscall-heavy app (Redis, MySQL) ১০-৩০% slower হয়েছিল।

```bash
# CPU-তে কী mitigation আছে
cat /sys/devices/system/cpu/vulnerabilities/*
```

### Syscall storm — pitfall

```php
// ❌ N syscall — প্রতি ১ বাইট-এ
$f = fopen('large.txt', 'r');
while (!feof($f)) {
    echo fgetc($f);    // প্রতি বার read(fd, buf, 1)
}

// ✅ 1 syscall — buffered
echo file_get_contents('large.txt');
```

```javascript
// ❌ tight loop syscall
for (let i = 0; i < 1e6; i++) {
    fs.appendFileSync('log', `${i}\n`);   // 1M open+write+close!
}

// ✅
const stream = fs.createWriteStream('log');
for (let i = 0; i < 1e6; i++) stream.write(`${i}\n`);
stream.end();
```

---

## 📌 Syscall batching / vectored I/O

কার্নেল boundary crossing কমাতে কয়েকটা trick:

### writev / readv (scatter-gather)

একাধিক buffer একটি syscall-এ:

```c
struct iovec iov[3];
iov[0].iov_base = header;  iov[0].iov_len = hdr_len;
iov[1].iov_base = body;    iov[1].iov_len = body_len;
iov[2].iov_base = footer;  iov[2].iov_len = ftr_len;
writev(fd, iov, 3);        // ১টা syscall, ৩টা buffer
```

HTTP response (header + body) Nginx writev দিয়ে পাঠায়।

### sendfile — zero-copy

```c
sendfile(out_fd, in_fd, NULL, count);
```

ফাইল → socket — userspace বাফারে কপি না করেই। Nginx-এ static file serve হয় এভাবে।

```
Traditional read+write:
  disk → kernel buf → user buf → kernel sock buf → NIC      (4 copies)

sendfile:
  disk → kernel buf → kernel sock buf → NIC                 (2 copies)

splice/tee + io_uring:
  disk → kernel buf → NIC (DMA)                             (1 copy)
```

### sendmsg, recvmsg

socket-এ multiple buffers + ancillary data (FD passing, timestamps)।

### io_uring

আধুনিকতম — একদম syscall-less submission/completion ring। বিস্তারিত [io-models.md](./io-models.md)।

---

## 📌 strace, ltrace, perf দিয়ে trace

### strace — syscall trace

```bash
# একটা কমান্ডের সব syscall
strace -f ls /tmp                     # -f = follow forks

# specific syscalls
strace -e trace=open,read,write -p <pid>
strace -e trace=network -p <pid>
strace -e trace=%file ./app           # সব file syscall

# timing summary
strace -c -p <pid>
% time     seconds  usecs/call  calls    errors  syscall
------- ----------- ----------- -------- -------- ----------
 65.20    0.012345         12     1023            read
 18.40    0.003456          4      856            write
 10.10    0.001890         15      125     12     openat
...

# attach to running process
strace -p <pid> -o trace.log

# string output truncate কম
strace -s 1000 -p <pid>

# timestamp
strace -tt -T -p <pid>          # -tt = wall time, -T = duration
```

> 💡 **Production ব্যবহার সাবধান**: strace প্রতিটা syscall-এ ptrace overhead — slow target ১০০x+। প্রোডাকশনে eBPF-based `bpftrace`/`bcc` ব্যবহার করুন।

### eBPF (modern)

```bash
# একটা syscall-এর latency histogram
funclatency-bpfcc -d 5 'vfs_read'

# একটা প্রসেসের সব syscall count
syscount-bpfcc -p <pid>

# bpftrace one-liner
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
```

### ltrace — library call trace

```bash
ltrace -p <pid>      # libc, libssl ইত্যাদির function call
```

### perf — profile

```bash
perf stat -p <pid> sleep 5            # কতবার syscall, cs, page-fault
perf trace -p <pid>                   # like strace, faster
```

### একটা real bug — strace দিয়ে ধরা

```bash
# php-fpm worker freeze, কেন?
$ ps aux | grep php-fpm
www  4521  0.0  ...  S+   php-fpm: pool www

$ strace -p 4521
read(7, <unfinished ...>          ← ৩০ সেকেন্ড আটকে
                                    fd 7 কী?

$ ls -l /proc/4521/fd/7
lrwx... -> 'socket:[1234567]'

$ ss -tnp | grep 1234567
ESTAB ... 10.0.0.5:3306 ... users:(("php-fpm",pid=4521,fd=7))
                  ↑
            MySQL slow query!
```

---

## 📌 seccomp — syscall filtering

প্রসেসকে শুধু কিছু syscall-এ আবদ্ধ করুন → security sandbox। Docker, Chrome, Firefox ব্যবহার করে।

### Modes

- `SECCOMP_MODE_STRICT` — শুধু read/write/exit/sigreturn
- `SECCOMP_MODE_FILTER` — BPF filter দিয়ে allow/deny per syscall

### উদাহরণ — Docker default profile

Docker ডিফল্টে ~৪৪টা syscall block করে: `reboot`, `kexec_load`, `mount`, `umount`, `swapon`, `unshare` ইত্যাদি।

```bash
docker run --security-opt seccomp=/path/to/profile.json myapp
docker run --security-opt seccomp=unconfined myapp     # ⚠️ disable
```

### PHP-এ seccomp (libc-only sandbox)

PHP-এর native binding নেই, কিন্তু `prctl(PR_SET_SECCOMP, ...)` থেকে বানানো extension আছে। সাধারণ ক্ষেত্রে কন্টেইনার level-এ যথেষ্ট।

```c
// সরাসরি C
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
seccomp_init(SCMP_ACT_KILL);
seccomp_rule_add(SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_load();
```

---

## ⚠️ Pitfalls

### ১. `EINTR` handling

Signal এলে কিছু syscall interrupted হয়, `errno = EINTR`। retry করতে হয়:

```c
ssize_t n;
do { n = read(fd, buf, sz); } while (n < 0 && errno == EINTR);
```

আধুনিক glibc-এ `SA_RESTART` দিয়ে auto-retry।

### ২. Partial read/write

```c
// ❌ ভুল ধারণা: write() সব বাইট লেখে
write(fd, buf, 1000);    // হয়তো শুধু 600 লেখা হলো (TCP, pipe)

// ✅ loop
ssize_t total = 0;
while (total < count) {
    ssize_t n = write(fd, buf + total, count - total);
    if (n <= 0) break;
    total += n;
}
```

### ৩. fsync ছাড়া durability নাই

```php
file_put_contents('important.txt', $data);
// shutdown হলে ডেটা page cache-এ — disk-এ পৌঁছায়নি!

$fp = fopen('important.txt', 'w');
fwrite($fp, $data);
fflush($fp);    // userspace buffer → kernel
// না — fsync লাগবে kernel → disk
$meta = stream_get_meta_data($fp);
// PHP-এ direct fsync নেই; eio extension অথবা DB ব্যবহার করুন
fclose($fp);
```

```javascript
// Node.js
const fd = fs.openSync('important.txt', 'w');
fs.writeSync(fd, data);
fs.fsyncSync(fd);    // ✅ disk-এ guarantee
fs.closeSync(fd);
```

### ৪. ulimit -n syscall fail

প্রতিটি `open()`, `socket()` একটি fd নেয়। default 1024 — Node.js high-conn server-এ "EMFILE: too many open files"।

```bash
ulimit -n 65536
# /etc/security/limits.conf
*  soft  nofile  65536
*  hard  nofile  65536
```

### ৫. PHP-এ stream_select syscall লিমিট

`select()` syscall FD_SETSIZE (default 1024) এর বেশি fd নিতে পারে না। বেশি concurrent connection হলে `epoll`-based extension (ev, swoole) দরকার।

### ৬. Container-এ syscall block-এ fail

কিছু runtime (Java newer JDK, Go ১.১৯+) `clone3` ব্যবহার করে — পুরোনো Docker/seccomp profile এটা block করে → "Operation not permitted"। Docker 20.10+ এ fix।

### ৭. strace overhead

প্রোডাকশনে strace করতে গিয়ে latency 10x → outage। eBPF/perf trace ব্যবহার করুন।

---

## 📊 সারমর্ম

```
┌─────────────────────────────────────────────────────────────────┐
│ ১. user space → kernel space-এর একমাত্র legal গেট: syscall      │
│ ২. x86_64 এ `syscall` instruction; পুরোনোটায় int 0x80          │
│ ৩. vDSO = syscall-less syscall (gettimeofday)                   │
│ ৪. PHP/JS-এর সব file/socket/timer call শেষমেশ syscall          │
│ ৫. Syscall ~150-300ns; storm avoid — buffer/batch করুন           │
│ ৬. writev, sendfile, io_uring = batching tricks                 │
│ ৭. strace = dev/staging; প্রোডাকশনে eBPF/bpftrace                │
│ ৮. seccomp = container syscall sandbox                           │
│ ৯. EINTR retry, partial write loop, fsync — must-handle         │
│ ১০. ulimit -n বাড়ান high-fd app-এ                                │
└─────────────────────────────────────────────────────────────────┘
```

> ➡️ পরবর্তী: [io-models.md](./io-models.md) — হাজার connection epoll দিয়ে।
