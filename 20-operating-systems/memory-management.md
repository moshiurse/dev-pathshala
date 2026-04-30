# 🧠 মেমোরি ম্যানেজমেন্ট — কার্নেল পার্স্পেক্টিভ

> **`17-performance/memory-management.md` যেখানে শেষ — অ্যাপ্লিকেশন লেভেল GC, leak, profiling — সেখান থেকে শুরু।
> এই ফাইলে আমরা নামি কার্নেল লেভেলে: virtual memory, page table, MMU, TLB, page fault, OOM killer, cgroup memory।**

---

## 📖 সূচিপত্র

- [Virtual Memory কেন দরকার](#-virtual-memory-কেন-দরকার)
- [প্রসেস অ্যাড্রেস স্পেস লেআউট](#-প্রসেস-অ্যাড্রেস-স্পেস-লেআউট)
- [Paging, Page Table, MMU, TLB](#-paging-page-table-mmu-tlb)
- [Demand Paging ও Page Fault](#-demand-paging-ও-page-fault)
- [Copy-on-Write (CoW)](#-copy-on-write-cow)
- [Linux Memory Zones](#-linux-memory-zones)
- [Swap ও swappiness](#-swap-ও-swappiness)
- [OOM Killer](#-oom-killer)
- [Container memory ও cgroup](#-container-memory-ও-cgroup)
- [PHP / Node মেমোরি লিমিট কীভাবে কার্নেলে ম্যাপ হয়](#-php--node-মেমোরি-লিমিট-কীভাবে-কার্নেলে-ম্যাপ-হয়)
- [অবজার্ভেবিলিটি](#-অবজার্ভেবিলিটি)
- [Pitfalls](#-pitfalls)

---

## 📌 Virtual Memory কেন দরকার

প্রতিটি প্রসেস মনে করে সে পুরো ৬৪-বিট অ্যাড্রেস স্পেস (২^৪৮ ≈ ২৫৬ TB usable on x86_64) একা পেয়েছে। কিন্তু আসল RAM ৪/৮/১৬ GB। কার্নেল mediator হিসেবে কাজ করে।

### সমস্যা যা virtual memory সমাধান করে

```
১. Isolation:    Process A কখনোই Process B-এর মেমোরি পড়তে পারবে না
২. Relocation:   প্রোগ্রাম যেকোনো RAM address-এ load হতে পারে
৩. Over-commit:  RAM-এর চেয়ে বেশি allocate করা যায় (lazy, swap)
৪. Sharing:      libc, .so file কে অনেক প্রসেস একই physical RAM-এ পড়ে
৫. Protection:   read-only page (text, .so) modify করতে গেলে SIGSEGV
```

```
       VIRTUAL ADDRESS SPACE                  PHYSICAL RAM
       (প্রতি প্রসেসের নিজের)                   (সব প্রসেস শেয়ার করে)
   ┌──────────────────────┐                ┌────────────────────┐
   │ Process A: 0x..7fff │                │                    │
   │  page X ────────────┼──── MMU ──────▶│  physical frame 42 │
   │                      │   page table   │                    │
   │                      │                │                    │
   ├──────────────────────┤                │  physical frame 17 │ ◀── shared
   │ Process B: 0x..7fff │                │  (libc.so)         │     (CoW/RO)
   │  page X ────────────┼──── MMU ──────▶│                    │
   │                      │                │                    │
   └──────────────────────┘                └────────────────────┘
```

---

## 📌 প্রসেস অ্যাড্রেস স্পেস লেআউট

x86_64 Linux-এ একটি প্রসেসের ভার্চুয়াল মেমোরি কেমন দেখায়:

```
0xFFFFFFFFFFFFFFFF  ┌──────────────────────────────┐ ┐
                    │     KERNEL SPACE              │ │ user-space
                    │   (top 128TB on x86_64)       │ │ থেকে দেখা
                    │   user mode-এ access নিষেধ     │ │ যায় না
0xFFFF800000000000  ├──────────────────────────────┤ ┘
                    │     (non-canonical hole)      │
0x00007FFFFFFFFFFF  ├──────────────────────────────┤ ┐
                    │   STACK (main thread)         │ │
                    │   ↓ grows down               │ │
                    │                              │ │
                    │   ─── (vDSO, vsyscall) ───   │ │
                    │                              │ │
                    │   MMAP REGION                 │ │  USER
                    │   • shared libraries (libc)   │ │  SPACE
                    │   • mmap'd files              │ │  (128 TB)
                    │   • thread stacks             │ │
                    │   • anonymous mmap (malloc>128KB)│
                    │                              │ │
                    │   ↑ ↓ heap ও mmap বাড়ে এদিকে │ │
                    │                              │ │
                    │   HEAP                        │ │
                    │   ↑ grows up via brk()        │ │
                    ├──────────────────────────────┤ │
                    │   BSS  (uninitialized data)   │ │
                    │   DATA (initialized globals)  │ │
                    │   TEXT (code, read-only)      │ │
0x0000000000400000  ├──────────────────────────────┤ │
                    │   (NULL pointer trap, unmapped│ │
0x0000000000000000  └──────────────────────────────┘ ┘
```

### বাস্তবে দেখুন

```bash
$ cat /proc/self/maps
00400000-0040b000 r-xp 00000000 fd:00 1234  /bin/cat        ← TEXT
0060a000-0060b000 r--p 0000a000 fd:00 1234  /bin/cat        ← read-only data
0060b000-0060c000 rw-p 0000b000 fd:00 1234  /bin/cat        ← DATA
01a8d000-01aae000 rw-p 00000000 00:00 0     [heap]          ← HEAP (brk)
7f3c... r-xp ...       /lib/x86_64-linux-gnu/libc-2.31.so  ← shared lib
7ffe... rw-p ...       [stack]                              ← STACK
7ffe... r-xp ...       [vdso]                               ← virtual syscall
```

### সেগমেন্ট ব্যাখ্যা

| সেগমেন্ট | কী থাকে | permission |
|----------|---------|-----------|
| TEXT | machine code (program) | `r-x` |
| DATA | initialized globals (`int x = 5;`) | `rw-` |
| BSS | zero-init globals (`int y;`) | `rw-` |
| HEAP | `malloc`, PHP variable, JS object | `rw-`, `brk()` দিয়ে বাড়ে |
| MMAP | shared libs, large allocations, file maps | বিভিন্ন |
| STACK | function call frame, local var | `rw-`, auto-grow |

---

## 📌 Paging, Page Table, MMU, TLB

মেমোরি **page** নামক ফিক্সড সাইজের ব্লকে ভাগ — Linux x86_64-এ ডিফল্ট **4 KB** (huge page 2 MB / 1 GB-ও আছে)।

### ৪-লেভেল পেজ টেবিল (x86_64)

```
Virtual Address (48 bits used):
┌────────┬────────┬────────┬────────┬──────────────┐
│ PML4   │ PDPT   │ PD     │ PT     │   offset     │
│ 9 bits │ 9 bits │ 9 bits │ 9 bits │   12 bits    │
└────┬───┴────┬───┴────┬───┴────┬───┴──────┬───────┘
     │        │        │        │          │
     │        │        │        │          │  → 4KB page-এর ভিতরে byte
     │        │        │        │          ▼
     │        │        │        │     [Physical Frame Number]
     │        │        │        │          +
     │        │        │        ▼          |
     │        │        ▼     Page Table  → physical address
     │        ▼      Page Dir
     ▼     PDPT
   PML4 (CR3 register এ pointer)
```

প্রতিটি memory access-এ ৪টা lookup হলে CPU ১/৫ গুণ slow হবে — তাই **TLB (Translation Lookaside Buffer)**: একটি ছোট cache (১২৮-১৫০০ entry) যা সদ্য translated virtual→physical mapping রাখে।

```
CPU                                      RAM
 │                                        │
 ├── load virtual addr 0x7fff1234 ───┐    │
 │                                  │    │
 ▼                                  ▼    │
TLB hit?  ─── YES ──▶ direct physical lookup (1-2 cycles)
   │
   NO (TLB miss)
   ▼
 Page Walk (4 lookups in RAM, ~100 cycles)
   ▼
 Page found?  ─── YES ──▶ update TLB → access (slow first time)
   │
   NO (Page Fault!)
   ▼
 Kernel handler called (trap)
```

### Huge Pages

বড় DB (MySQL, PostgreSQL, Redis) যখন GB-scale heap ব্যবহার করে, 4KB page-এর জন্য TLB miss বেড়ে যায়। সমাধান: **2MB হিউজ পেজ**।

```bash
# huge page enable
echo 1024 > /proc/sys/vm/nr_hugepages   # 1024 × 2MB = 2GB

# Transparent Huge Pages (THP)
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# MySQL/Redis-এর জন্য সাধারণত madvise বা never (latency ↓)
```

> 💡 **Daraz MySQL tuning**: প্রোডাকশন DB সার্ভারে THP `always` থেকে `madvise` করায় p99 latency ১৫% কমে গেছিল — THP defrag-এ random latency spike হতো।

---

## 📌 Demand Paging ও Page Fault

কার্নেল lazy — `malloc(1GB)` করলে সাথে সাথে 1GB RAM allocate হয় না। শুধু virtual address range রিজার্ভ হয়। যখন প্রথমবার write হয়, তখনই physical page allocate হয়।

### Page Fault এর ধরন

```
                 page fault!
                       │
        ┌──────────────┴──────────────┐
        │                             │
   MINOR fault                  MAJOR fault
   (soft)                       (hard)
        │                             │
        │  page RAM-এ আছে কিন্তু       │  page disk-এ (swap বা file),
        │  page table-এ যোগ হয়নি      │  RAM-এ আনতে হবে
        │  (~ < 1 μs)                  │  (~ 1-10 ms!)
        │                             │
   ─────┴────────────────             ┴──────────────
   • প্রথমবার heap touch              • swapped out page পড়া
   • CoW trigger                     • mmap'd file প্রথম access
   • shared library load             • RAM full, eviction হয়েছে

                 অথবা
                       │
              ┌────────▼────────┐
              │  INVALID page   │  → SIGSEGV (segfault)
              │  (NULL deref,   │
              │   stack overflow)│
              └─────────────────┘
```

### বাস্তবে দেখুন

```bash
# একটি প্রসেসের page fault stat
$ ps -o min_flt,maj_flt,cmd -p <pid>
MINFL  MAJFL  CMD
12453  3      php-fpm: pool www
                ↑
        শুধু 3টা major fault — RAM-এ ভালোমতো fit হচ্ছে

# Major fault অনেক বেশি = swap বা I/O pressure
$ vmstat 1
 procs  ─────memory─────  ───swap──  ───io───  ─system─  ─cpu─
  r  b   swpd   free  ...  si   so   bi   bo   in   cs  us sy id wa
  2  0      0  4.5G        0    0    5    8   200  500  10 5 80 5
                          ↑swap in/out — যদি বাড়তে থাকে, RAM short
```

---

## 📌 Copy-on-Write (CoW)

`fork()` যদি প্যারেন্টের সব মেমোরি কপি করত, ১GB Node.js master fork করলে সাথে সাথে ১GB RAM খরচ হতো। **CoW** এই খরচ এড়ায়।

```
fork() এর আগে:                         fork() এর ঠিক পরে (CoW):
                                       ────────────────────────
Parent's heap:                         Parent + Child দুজনেই
 ┌──────────────┐                       একই physical frame দেখে,
 │  page A (RW) ├──▶ frame 42           page table-এ permission
 │  page B (RW) ├──▶ frame 43           force-করা READ-ONLY!
 │  page C (RW) ├──▶ frame 44
 └──────────────┘
                                       Child writes page B:
                                       1. CPU trap (write protection)
                                       2. Kernel: page B এর copy বানাও
                                          frame 99 = copy of frame 43
                                       3. Child's page B → frame 99 (RW)
                                       4. write resume
```

ফল: PHP-FPM `pm = static` দিয়ে ৫০টা worker spawn হলে actual extra RAM শুধু সেটুকু যেটা প্রতি worker আলাদা করে modify করেছে।

### Redis BGSAVE — CoW-এর ক্ল্যাসিক সমস্যা

Redis ডিস্কে snapshot নেওয়ার জন্য `fork()` করে। যদি Redis `4GB` ডেটা ধরে রাখে, child process snapshot লেখার সময় parent যদি অনেক key update করে — প্রতিটা ১KB পেজ touch-এ একটা CoW copy হবে। প্রোডাকশনে dataset-এর সমান RAM extra লাগে worst-case।

```bash
# Redis instance-এ লক্ষ্য রাখা
redis-cli info memory | grep -i copy_on_write
# used_memory_peak_during_fork:...
```

---

## 📌 Linux Memory Zones

হার্ডওয়্যার constraint-এর কারণে কার্নেল RAM-কে zone-এ ভাগ করে।

```
┌──────────────────────────────────────────────────────────────┐
│ ZONE_DMA      │ 0–16 MB    │ পুরোনো ISA device-এর জন্য         │
├──────────────────────────────────────────────────────────────┤
│ ZONE_DMA32    │ 16 MB–4 GB │ ৩২-বিট DMA capable device         │
├──────────────────────────────────────────────────────────────┤
│ ZONE_NORMAL   │ 4 GB+      │ সাধারণ কাজে — kernel direct map   │
├──────────────────────────────────────────────────────────────┤
│ ZONE_HIGHMEM  │ (32-bit)   │ kernel direct-map এর বাইরে        │
│               │            │ (64-bit এ অপ্রয়োজনীয়)             │
└──────────────────────────────────────────────────────────────┘

দেখুন: cat /proc/zoneinfo
```

### NUMA (Non-Uniform Memory Access)

মাল্টি-সকেট সার্ভারে (যেমন bKash data center-এর dual-socket Xeon) প্রতি CPU socket-এর নিজস্ব "local" RAM আছে। অন্য socket-এর RAM access ২-৩x ধীর।

```bash
$ numactl --hardware
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32000 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32000 MB
node distances:
node   0   1
  0:  10  21          ← 21/10 = 2.1x slow remote access
  1:  21  10
```

---

## 📌 Swap ও swappiness

RAM full হলে কার্নেল least-recently-used page disk-এ লিখে দেয় (swap)। 

```bash
$ free -h
              total   used   free  shared  buff/cache  available
Mem:           7.8G   3.2G   1.1G    340M        3.5G       4.0G
Swap:          2.0G   512M   1.5G

# swappiness: 0–100, higher = swap বেশি ব্যবহার করো
$ cat /proc/sys/vm/swappiness
60                              # default — desktop-এর জন্য ঠিক
```

### Production tuning

```bash
# DB/Redis সার্ভার — swap যত পারো এড়াও
sysctl -w vm.swappiness=1

# /etc/sysctl.conf
vm.swappiness = 1
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
```

> 💡 **bKash Redis**: swap-এ ৫০MB-ও গেলে p99 latency ১০ms → ১০০ms হতে পারে (major fault)। প্রোডাকশনে `swappiness=1`, কিছু dev `swapoff -a` করেও দেয়।

---

## 📌 OOM Killer

কার্নেল যখন allocate করার মতো RAM বা swap কোনোটাই খুঁজে পায় না, **OOM Killer** চালু হয় — কোনো প্রসেস মেরে memory ফিরে নেয়।

### স্কোরিং অ্যালগরিদম

```
oom_score = f(RSS, runtime, nice, oom_score_adj)

higher score → kill করার candidate
```

```bash
# কোন প্রসেস কত vulnerable
$ cat /proc/<pid>/oom_score
567

# ম্যানুয়ালি adjust (-1000 = অমর, 1000 = সবার আগে kill)
$ echo -1000 > /proc/<pid>/oom_score_adj
```

### OOM event দেখুন

```bash
$ dmesg -T | grep -i 'killed process'
[Mon Oct  6 14:32:01] Out of memory: Killed process 12345 (php-fpm)
                       total-vm:512000kB, anon-rss:480000kB,
                       file-rss:0kB, shmem-rss:0kB, oom_score_adj:0
```

### Containerর OOM (cgroup OOM)

System-wide OOM-এর আগে cgroup-নির্ভর OOM কিল হয়। Kubernetes-এ pod যদি memory limit ছাড়ায়:

```bash
# pod log
kubectl describe pod foo
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```

Exit code 137 = 128 + 9 (SIGKILL)।

---

## 📌 Container memory ও cgroup

Docker/Kubernetes-এ `--memory=512m` দেওয়ার মানে কার্নেল cgroup-এর memory controller সেট করে।

### cgroup v2 (আধুনিক)

```bash
# কন্টেইনারের ভিতরে
$ cat /sys/fs/cgroup/memory.max
536870912                              # 512MB hard limit
$ cat /sys/fs/cgroup/memory.current
312345678                              # বর্তমান ব্যবহার
$ cat /sys/fs/cgroup/memory.swap.max
0                                      # swap বন্ধ
$ cat /sys/fs/cgroup/memory.events
oom 0
oom_kill 0                             # OOM-kill কতবার হয়েছে
```

### Container-aware language runtime

```bash
# পুরোনো Java JVM container-এর memory limit বুঝত না — host RAM দেখে heap বানাত
# JDK 10+ এ -XX:+UseContainerSupport ডিফল্ট

# Node.js — manually set
node --max-old-space-size=400 server.js     # container limit 512MB হলে

# PHP-FPM — pm.max_children × memory_limit ≤ container memory
```

---

## 📌 PHP / Node মেমোরি লিমিট কীভাবে কার্নেলে ম্যাপ হয়

### PHP `memory_limit`

```ini
; php.ini
memory_limit = 256M
```

PHP নিজেই Zend MM ব্যবহার করে — `mmap`/`brk` দিয়ে কার্নেল থেকে RAM নেয়, internally pool করে। `memory_limit` ছাড়ালে PHP **Fatal Error: Allowed memory size of N bytes exhausted** — এটা PHP-এর নিজস্ব check, OS পর্যন্ত যায় না।

```php
// বর্তমান usage
echo memory_get_usage(true) . "\n";   // emalloc allocated
echo memory_get_peak_usage(true);     // peak

// system level — /proc থেকে
$status = file_get_contents('/proc/self/status');
preg_match('/VmRSS:\s+(\d+)/', $status, $m);
echo "RSS: " . $m[1] . " KB\n";
```

### Node.js `--max-old-space-size`

V8-এর old generation-এর সর্বোচ্চ size (MB)। এর বাইরে allocate করতে গেলে **JavaScript heap out of memory** crash।

```bash
node --max-old-space-size=4096 server.js  # 4GB
```

| লিমিট | কে enforce করে | crash type |
|------|---------------|-----------|
| `memory_limit` (PHP) | PHP runtime | Fatal Error (catchable) |
| `--max-old-space-size` (Node) | V8 | FATAL ERROR (process abort) |
| `ulimit -v` | kernel rlimit | malloc fail / SIGSEGV |
| cgroup `memory.max` | kernel cgroup | OOMKilled (SIGKILL) |
| host RAM exhaust | kernel OOM | global OOM kill |

---

## 📌 অবজার্ভেবিলিটি

### সিস্টেম-ওয়াইড

```bash
free -h                          # সারমর্ম
free -m -s 2                     # প্রতি ২ সেকেন্ড refresh
vmstat 1                         # paging, swap, IO, CPU
sar -r 1                         # memory utilization (sysstat)
cat /proc/meminfo                # সবচেয়ে detailed
slabtop                          # kernel slab cache
```

### `/proc/meminfo` মূল ফিল্ড

```
MemTotal:        8053236 kB
MemFree:         1023456 kB     ← পুরোপুরি unused
MemAvailable:    4523456 kB     ← user app আসলে যতটা পাবে
Buffers:           87432 kB     ← block device buffer
Cached:          3056234 kB     ← page cache (file)
SwapTotal:       2097148 kB
SwapFree:        2097148 kB
Dirty:              4321 kB     ← write-pending (yet to fsync)
Slab:             523456 kB     ← kernel object cache
```

> 💡 **MemFree কম, Cached বেশি — চিন্তার কিছু নেই**। Linux unused RAM-কে file cache হিসেবে ব্যবহার করে; পুরো cache instantly evict করা যায়।

### প্রতি প্রসেস

```bash
# RSS, VSZ ইত্যাদি
ps aux | head
ps -o pid,vsz,rss,pmem,cmd -p <pid>

# detailed map
pmap -x <pid>

# /proc/<pid>/status
cat /proc/<pid>/status | grep -E 'Vm|Rss'
# VmPeak:    peak virtual size
# VmSize:    current virtual size
# VmRSS:     resident set (actual RAM)
# VmData:    data + heap
# VmStk:     stack
# VmSwap:    swapped out

# detailed segment-by-segment
cat /proc/<pid>/smaps             # প্রতিটা mapping-এ pss, rss, swap
```

### PSS vs RSS

- **RSS (Resident Set Size)**: এই প্রসেস যত page RAM-এ আছে — শেয়ার্ড সহ। Sum of all process RSS > total RAM হতে পারে!
- **PSS (Proportional Set Size)**: শেয়ার্ড page-কে ভাগ করে নেয়। Sum of all PSS ≈ actual RAM use।

```bash
# একটি প্রসেসের actual memory footprint
$ cat /proc/<pid>/smaps_rollup | grep Pss
Pss:       45230 kB
```

### Memory leak খোঁজা (kernel-level tool)

```bash
# valgrind — userspace
valgrind --leak-check=full ./myapp

# বড় allocation track
perf record -e syscalls:sys_enter_brk,syscalls:sys_enter_mmap -p <pid>
perf report

# eBPF (BCC)
memleak-bpfcc -p <pid>
```

---

## ⚠️ Pitfalls

### ১. "VSZ অনেক বেশি, RSS কম" — চিন্তা নেই

```
VSZ = 1.2 GB, RSS = 120 MB
```
অর্থাৎ প্রসেস virtually 1.2GB রিজার্ভ করেছে কিন্তু actually 120MB use করছে — Java/Node-এ স্বাভাবিক।

### ২. Container-এ memory.usage_in_bytes মিথ্যা বলে

`memory.usage_in_bytes` (cgroup v1) এ page cache অন্তর্ভুক্ত। App actually 200MB use করছে কিন্তু metric 480MB দেখাচ্ছে — page cache। সঠিক metric: `memory.stat` থেকে `rss` ফিল্ড।

### ৩. Overcommit accounting

```bash
$ cat /proc/sys/vm/overcommit_memory
0    # default — heuristic; allocation কখনো reject হতে পারে যদি obvious-ly বেশি

# 1 = always allow (Redis recommend করে)
# 2 = strict — RAM + swap × ratio এর বেশি না
```

Redis-এর জন্য `1` সেট করতে বলা হয় কারণ fork() time-এ kernel ভাবে চাইল্ড পুরো parent-এর RAM allocate করবে — যদিও CoW-এর কারণে actually করবে না।

### ৪. Stack overflow vs heap exhaustion

```c
void recurse() { char buf[1024]; recurse(); }   // SIGSEGV (stack)

while (1) malloc(1024 * 1024);                  // OOM (heap)
```

### ৫. Memory fragmentation

দীর্ঘসময় চলা প্রসেসে heap fragmented হতে পারে — `RSS` বাড়ে কিন্তু useful data কম। PHP-FPM `pm.max_requests` দিয়ে periodic worker recycle এর কারণ এটা।

### ৬. Shared hosting "Allowed memory size exhausted"

BD shared hosting (Hostever, Exonhost) এ `memory_limit = 64M` হলে Laravel-এর কিছু query সহজে ছাড়িয়ে যায়। সমাধান: `ini_set('memory_limit', '256M')` (যদি allow থাকে) অথবা VPS-এ যান।

---

## 📊 সারমর্ম

```
┌──────────────────────────────────────────────────────────────────┐
│  ১. প্রতি প্রসেসের নিজের virtual address space; MMU mapping করে   │
│  ২. Page = 4KB; page table 4-level; TLB cache করে                │
│  ৩. malloc lazy — actual RAM page fault-এ allocate হয়             │
│  ৪. fork() = CoW; পেজ write-এ আলাদা copy                          │
│  ৫. Major page fault = disk hit = milliseconds                    │
│  ৬. Linux unused RAM cache হিসেবে ব্যবহার করে — MemFree ≠ available │
│  ৭. OOM killer স্কোর দিয়ে candidate বেছে SIGKILL পাঠায়             │
│  ৮. cgroup memory limit container-এ enforce করে                  │
│  ৯. swappiness, dirty_ratio, overcommit — production tuning      │
│ ১০. /proc/<pid>/{status,maps,smaps,smaps_rollup} = সোনার খনি     │
└──────────────────────────────────────────────────────────────────┘
```

> ➡️ পরবর্তী: [cpu-scheduling.md](./cpu-scheduling.md) — কে কখন CPU পায়।
