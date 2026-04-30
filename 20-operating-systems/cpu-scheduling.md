# ⏱️ CPU শিডিউলিং (CPU Scheduling)

> **হাজার প্রসেসের জন্য মাত্র ৪/৮/১৬টা CPU। কে কখন CPU পাবে — সিদ্ধান্ত নেয় কার্নেল scheduler। এই সিদ্ধান্তেই নির্ভর করে আপনার API-এর latency।**

---

## 📖 সূচিপত্র

- [কেন scheduling দরকার](#-কেন-scheduling-দরকার)
- [Scheduling Class ও Policy](#-scheduling-class-ও-policy)
- [CFS (Completely Fair Scheduler)](#-cfs-completely-fair-scheduler)
- [Niceness ও Priority](#-niceness-ও-priority)
- [Real-time scheduling](#-real-time-scheduling)
- [Context switch — খরচ ও পরিমাপ](#-context-switch--খরচ-ও-পরিমাপ)
- [Multi-core, CPU affinity, NUMA](#-multi-core-cpu-affinity-numa)
- [Load average ব্যাখ্যা](#-load-average-ব্যাখ্যা)
- [Container CPU (shares, quota)](#-container-cpu-shares-quota)
- [Node.js কেন epoll-এ থ্রেড-পার-conn কে হারায়](#-nodejs-কেন-epoll-এ-থ্রেড-পার-conn-কে-হারায়)
- [অবজার্ভেবিলিটি](#-অবজার্ভেবিলিটি)
- [Pitfalls](#-pitfalls)

---

## 📌 কেন scheduling দরকার

```
        CPU 0        CPU 1        CPU 2        CPU 3
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ ?       │  │ ?       │  │ ?       │  │ ?       │
    └─────────┘  └─────────┘  └─────────┘  └─────────┘
        ▲                                        ▲
        │                                        │
   ┌────┴────────────────── runqueue ────────────┴────┐
   │  nginx  php-fpm  mysql  cron  bash  node  ...   │ ← runnable tasks
   └──────────────────────────────────────────────────┘

Scheduler-এর কাজ: প্রতি tick-এ (~1ms) ঠিক করো —
   • কোন task এখন CPU পাবে
   • কতক্ষণ চলবে (time slice)
   • কখন preempt হবে
```

**লক্ষ্য (যা পরস্পরবিরোধী!)**:
1. **Fairness** — সবাই কিছু না কিছু CPU পাক
2. **Throughput** — যত বেশি কাজ শেষ হোক
3. **Latency** — interactive task দ্রুত response পাক
4. **Power** — অপ্রয়োজনে CPU চালু না থাকুক

---

## 📌 Scheduling Class ও Policy

Linux scheduler-এ multiple "class" আছে। প্রতিটিতে নিজস্ব algorithm। উচ্চ-priority class আগে চলে।

```
                    Linux Scheduler Classes (priority order)
   ┌──────────────────────────────────────────────────────────┐
   │ 1. stop_sched_class    │  হার্ডওয়্যার stop, hot-unplug    │
   ├──────────────────────────────────────────────────────────┤
   │ 2. dl_sched_class      │  SCHED_DEADLINE (EDF)            │
   │                        │  rt-budget বাঁধা                  │
   ├──────────────────────────────────────────────────────────┤
   │ 3. rt_sched_class      │  SCHED_FIFO, SCHED_RR            │
   │                        │  hard real-time                   │
   ├──────────────────────────────────────────────────────────┤
   │ 4. fair_sched_class    │  SCHED_NORMAL (was OTHER),       │ ← 99%
   │    (CFS)               │  SCHED_BATCH, SCHED_IDLE         │   এখানে
   ├──────────────────────────────────────────────────────────┤
   │ 5. idle_sched_class    │  নাথিং করার থাকলে                  │
   └──────────────────────────────────────────────────────────┘
```

| Policy | Class | Use case |
|--------|-------|----------|
| `SCHED_NORMAL` (`SCHED_OTHER`) | CFS | 99% সব সাধারণ প্রসেস |
| `SCHED_BATCH` | CFS | Background CPU-bound (compile, video encode) — wakeup latency কম দরকার |
| `SCHED_IDLE` | CFS | শুধু CPU idle হলে চলবে (very low priority) |
| `SCHED_FIFO` | RT | Real-time, no time slicing — যতক্ষণ চাও চলো |
| `SCHED_RR` | RT | Real-time + time slice (round-robin) |
| `SCHED_DEADLINE` | DL | Hard deadline (runtime, period, deadline) |

```bash
$ chrt -p $$       # current shell-এর policy দেখা
pid 12345's current scheduling policy: SCHED_OTHER
pid 12345's current scheduling priority: 0

$ chrt -f 50 ./myapp     # SCHED_FIFO priority 50 দিয়ে চালাও
```

---

## 📌 CFS (Completely Fair Scheduler)

Linux 2.6.23 (২০০৭) থেকে ডিফল্ট। আগেই O(1) scheduler ছিল। Ingo Molnar ডিজাইন করেন।

### মূল ধারণা — vruntime

প্রতিটি task একটি **virtual runtime** (`vruntime`) ধরে রাখে — এটি বুঝায় task কত CPU time পেয়েছে (nice-weighted)। CFS সবসময় **সবচেয়ে কম vruntime-ওয়ালা task-কে next চালায়** → fairness।

```
Idealized "fair" world:
  ১০টা task, ১টা CPU →  প্রতি task পায় ১০% CPU।
  
CFS এই ideal-এর কাছাকাছি থাকার চেষ্টা করে — vruntime দিয়ে।
```

### Red-Black Tree

CFS runqueue একটি balanced BST (red-black tree), key = `vruntime`। leftmost node = next-to-run, O(log N)।

```
            [nginx vrun=120]
           /                \
   [php vrun=80]      [mysql vrun=180]
       /    \              /
[cron 50] [bash 100]   [node 150]
   ↑
leftmost — পরের task to run
```

### টাইম স্লাইস

CFS-এ classic time slice নেই, বদলে আছে:

- **`sched_min_granularity_ns`** (default 0.75 ms): নূন্যতম কতক্ষণ task চলবে preempt হওয়ার আগে
- **`sched_latency_ns`** (default 6 ms): এই window-তে সব runnable task একবার করে চলার চেষ্টা — কিন্তু min_granularity-এর কম না

```
যদি 4 task থাকে: প্রতি task ≈ 6/4 = 1.5 ms
যদি 100 task থাকে: 6/100 = 0.06 ms < min — তাই প্রতি task 0.75 ms
```

```bash
sysctl kernel.sched_min_granularity_ns
sysctl kernel.sched_latency_ns
sysctl kernel.sched_wakeup_granularity_ns
```

---

## 📌 Niceness ও Priority

### Nice value

Range `-20` (highest priority) থেকে `+19` (lowest)। Default = 0।

```
nice  -20  -10   0   +10  +19
       ↑                    ↑
   বেশি CPU              কম CPU
```

CFS-এ nice value `vruntime`-এ multiplier হিসেবে কাজ করে: nice -5 হলে vruntime ধীরে বাড়ে → বেশি CPU পায়।

```bash
# নতুন কমান্ড nice দিয়ে
nice -n 10 ./background-job.sh

# চলমান প্রসেস renice
renice -n 5 -p <pid>

# bKash batch reconciliation job
nice -n 19 ionice -c 3 ./reconciliation.php
```

### Priority numbers

```
Static priority:       0–139
                       0–99    → real-time tasks (rtprio)
                      100–139  → SCHED_NORMAL (mapped from nice -20..+19)

ps -eo pid,ni,pri,rtprio,policy,cmd
```

---

## 📌 Real-time scheduling

Hard real-time guarantee দরকার হলে — telecom signaling, audio mixing, robotics, billing system।

### SCHED_FIFO

Priority 1-99, যতক্ষণ ইচ্ছা চলবে — কেউ preempt করতে পারবে না (যদি না higher priority RT task আসে)। ⚠️ infinite loop = system hang।

```c
struct sched_param sp = { .sched_priority = 50 };
sched_setscheduler(0, SCHED_FIFO, &sp);
```

### SCHED_RR

SCHED_FIFO + time slice। একই priority-র মধ্যে round-robin।

### Safety net

```bash
# ডিফল্টে RT task সর্বোচ্চ 95% CPU পায়, 5% বাকিদের জন্য
$ cat /proc/sys/kernel/sched_rt_runtime_us
950000
$ cat /proc/sys/kernel/sched_rt_period_us
1000000
```

> 💡 **GP/Robi telco billing**: কিছু কম্পোনেন্ট SCHED_FIFO-তে চালানো হয় — packet processing-এ < 1ms jitter দরকার। Kernel `PREEMPT_RT` patch ব্যবহার করে।

---

## 📌 Context switch — খরচ ও পরিমাপ

প্রতিবার scheduler অন্য task-এ switch করলে:

```
১. বর্তমান task এর register state save (rax, rbx, ..., rip, rsp)
২. memory mapping switch (CR3 reload → TLB flush!)
৩. kernel stack switch
৪. সব কিছু restore — পরের task-এর জন্য
৫. cache cold — L1/L2 miss প্রথম কয়েক হাজার access
```

### খরচ

| Switch type | আনুমানিক time |
|-------------|--------------|
| Same thread (timer interrupt, no real switch) | ~10 ns |
| Thread → thread (same process) | ~1-2 μs |
| Process → process | ~3-5 μs (TLB flush বেশি) |
| With cache pollution | ~10-50 μs effective |

প্রতি সেকেন্ডে ১০,০০০ context switch = প্রায় 1-5% CPU শুধু switching-এ।

### পরিমাপ

```bash
# system-wide
$ vmstat 1
 procs  ───memory───  ──swap─  ──io─  ─system─  ─cpu─
  r  b   swpd   free   ...     cs  in
  3  0      0  4G            8500 5000   ← cs = context switches/sec

# per-process
$ pidstat -w 1 -p <pid>
PID    cswch/s    nvcswch/s    Command
1234   1200       50           php-fpm
       ↑          ↑
   voluntary    involuntary (preempt)

# /proc থেকে cumulative
cat /proc/<pid>/status | grep ctxt
voluntary_ctxt_switches:        12345
nonvoluntary_ctxt_switches:     678
```

| ধরন | অর্থ |
|-----|------|
| Voluntary (`cswch/s`) | নিজে syscall block-এ গেছে (read, sleep, mutex wait) |
| Involuntary (`nvcswch/s`) | scheduler জোর করে preempt — time slice শেষ |

> ⚠️ **High involuntary switch + high run queue** = CPU contention। প্রসেস scale করা দরকার।

---

## 📌 Multi-core, CPU affinity, NUMA

### CPU affinity (taskset)

একটি প্রসেসকে নির্দিষ্ট CPU-তে pin করা যায় — cache locality + jitter কমে।

```bash
# CPU 2,3 এ চালাও
taskset -c 2,3 ./myapp

# চলমান প্রসেসের affinity দেখা/সেট করা
taskset -cp <pid>
taskset -cp 0-3 <pid>

# /proc থেকে
cat /proc/<pid>/status | grep Cpus_allowed_list
```

### Programmatic (PHP, Node)

```c
// C: sched_setaffinity()
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(2, &set);
sched_setaffinity(0, sizeof(set), &set);
```

```php
<?php
// PHP — extension proctitle/pcntl-এ সরাসরি নেই, exec দিয়ে
exec("taskset -cp 2 " . getmypid());
```

```javascript
// Node.js — direct API নেই, taskset দিয়ে launch করুন
// অথবা os.setPriority() (nice change)
const os = require('os');
os.setPriority(0, -10);   // nice = -10 (root দরকার হতে পারে)
```

### NUMA awareness

```bash
$ numactl --hardware            # node layout
$ numactl --cpunodebind=0 --membind=0 ./mysql    # CPU0 + RAM0 শুধু

$ numastat -p <pid>             # প্রতি node-এ memory usage
```

> 💡 **bKash dual-socket Xeon**: MySQL server-এ NUMA imbalance-এ লুকানো 30% latency। `mysqld --innodb-numa-interleave=1` বা `numactl --interleave=all` দিয়ে সমাধান।

### IRQ affinity

NIC interrupt একটি কোরে আটকে থাকলে সেই কোর saturate হয়ে যায়। `irqbalance` daemon বা manual:

```bash
$ cat /proc/interrupts                          # কোন CPU কত IRQ
$ echo 4 > /proc/irq/24/smp_affinity            # IRQ 24 → CPU 2 (bitmask 0b0100)
```

---

## 📌 Load average ব্যাখ্যা

`uptime` এর তিনটি সংখ্যা — last 1, 5, 15 minute-এর exponentially-weighted load average।

```bash
$ uptime
 15:42:01 up 23 days,  4:12,  2 users,  load average: 2.34, 1.87, 1.45
```

### Load = কী আসলে?

Linux-এ load = **runnable + uninterruptible (D) tasks-এর গড়**।

```
load = avg over time of (R + D state tasks)
```

⚠️ অন্যান্য UNIX-এ শুধু R counts; Linux-এ D-ও — ফলে disk-bound system-এও load high।

### ব্যাখ্যা (4-core সিস্টেমে)

| Load | অর্থ |
|------|------|
| 0.5 | অর্ধেক CPU ব্যবহার, কিছু idle |
| 4.0 | পুরো full — সব 4 কোর busy, no queue |
| 8.0 | 2x oversubscribed — half tasks waiting |
| 16.0 | severe contention |

```
1-min > 5-min > 15-min   → load বাড়ছে (problem coming)
1-min < 5-min < 15-min   → load কমছে (recovering)
```

### Daraz flash sale-এ load চিত্র

```
সময়     load(1m, 5m, 15m)    ব্যাখ্যা
-----    ------------------    --------
22:00    0.5  0.4  0.4        স্বাভাবিক
22:30    1.2  0.8  0.6        গরম হচ্ছে
22:55    8.0  3.0  1.5        sale shortly — auto-scale ট্রিগার
23:00    25   12   4           ⚠️ overloaded, 5xx বাড়ছে
23:15    6.0  18   8           স্কেল-আউট কাজ করছে
23:45    3.0  10   12          recovery চলছে
```

---

## 📌 Container CPU (shares, quota)

Docker `--cpus`, K8s `cpu: 500m` — দুটোই cgroup-এর CPU controller-এ map হয়।

### cgroup v2 — CPU controller

```bash
# cgroup ফাইল (container এর ভিতরে দেখুন)
$ cat /sys/fs/cgroup/cpu.max
50000 100000        # 50ms quota প্রতি 100ms period = 0.5 CPU

$ cat /sys/fs/cgroup/cpu.weight
100                 # default; relative share (1-10000)
```

| Setting | অর্থ | scheduler effect |
|---------|------|------------------|
| `cpu.weight` (= cpu.shares × ratio) | আপেক্ষিক ভাগ | CFS-এ vruntime weight |
| `cpu.max quota period` | hard cap | quota শেষ হলে throttle |

### CPU throttling

```bash
$ cat /sys/fs/cgroup/cpu.stat
nr_periods 12345
nr_throttled 234           ← কতবার throttled হয়েছে
throttled_usec 1234567     ← মোট throttled সময়

# K8s pod-এ
$ kubectl exec pod -- cat /sys/fs/cgroup/cpu.stat
```

> ⚠️ **K8s CPU limit pitfall**: একটা K8s pod যদি `cpu: limit 1` দিয়ে থাকে আর হঠাৎ Java/Node multi-thread spike করে — pod throttled হয়, p99 latency 10x বাড়ে। অনেক টিম `cpu limit` সরিয়ে শুধু `request` রাখে (controversial)।

---

## 📌 Node.js কেন epoll-এ থ্রেড-পার-conn কে হারায়

```
─────────────────────────────────────────────────────────────────
Apache prefork (thread-per-connection)            10,000 client
─────────────────────────────────────────────────────────────────
   10,000 OS thread                              
   each ~2 MB stack → 20 GB virtual               ❌ RAM ফেইল
   each blocked on socket read()                   
   context switch storm                            ❌ CPU 90% sys
   
─────────────────────────────────────────────────────────────────
Node.js (single thread + epoll)                   10,000 client
─────────────────────────────────────────────────────────────────
   1 OS thread (event loop) + 4 worker (libuv)
   total RAM ~ 100-200 MB                         ✅
   no per-conn thread; epoll_wait returns N
   ready FDs, callback হয়                          ✅ minimal switch
```

**মূল কারণ**: scheduler-এর কাজ context switch করা — কিন্তু থ্রেড যত বেশি, switching overhead তত বেশি, useful work কম। epoll মডেলে scheduler একটাই thread দেখছে → কম switching, কাছাকাছি 100% useful CPU।

বিস্তারিত [io-models.md](./io-models.md) এ।

---

## 📌 অবজার্ভেবিলিটি

```bash
# সিস্টেম
top                                    # সাধারণ
htop                                   # color, tree, multi-cpu bar
mpstat -P ALL 1                        # প্রতি কোরের usage
vmstat 1                               # cs, in, r/b column

# প্রতি প্রসেস/থ্রেড
pidstat -t 1 -p <pid>                  # প্রতি থ্রেডের cpu, switch
ps -eo pid,ni,pri,rtprio,policy,psr,cmd     # psr = কোন CPU তে last ran

# Run queue length
sar -q 1                               # runq-sz, plist-sz, ldavg

# সদ্য schedule হওয়া event
perf sched record -- sleep 5
perf sched latency
perf sched timehist                    # প্রতি task কতক্ষণ wait/run

# eBPF (modern)
runqlat-bpfcc                          # run queue latency histogram
runqlen-bpfcc                          # queue length sampling
cpudist-bpfcc                          # প্রতি CPU on-cpu time histogram
```

### `top` লাইন পড়া

```
%Cpu(s):  45.2 us,  12.1 sy,  0.0 ni, 38.5 id,  3.0 wa,  0.0 hi,  1.2 si
           ↑       ↑       ↑      ↑      ↑       ↑       ↑
         user    kernel  nice   idle   iowait   hard    soft
                                              IRQ      IRQ
```

| ফিল্ড | অর্থ |
|------|------|
| `us` | user-space code চলছে |
| `sy` | kernel-space (syscall-এ অনেক সময়) |
| `wa` | iowait — disk I/O-এর জন্য idle CPU |
| `si` | softirq — network packet processing |
| `st` | steal — VM hypervisor অন্য VM-কে দিয়েছে |

> **High `sy` = syscall storm** (উদাহরণ: tight read/write loop) → batching বা epoll
> **High `wa` = disk bottleneck** → SSD upgrade, page cache বাড়াও
> **High `si` = network heavy** → IRQ affinity, RPS/RFS
> **High `st` (cloud VM)** = noisy neighbor → instance change

---

## ⚠️ Pitfalls

### ১. `nice` ভুলবেন না

ব্যাকগ্রাউন্ড job (cron, batch report) `nice -n 19` ছাড়া চালালে peak hour-এ p99 latency ভাঙে।

```bash
# bKash daily reconciliation
nice -n 19 ionice -c 3 php artisan reconciliation:run
```

### ২. Real-time priority infinite loop = system hang

`SCHED_FIFO` priority 99 এ একটা bug লুপ → SSH-ও পাবেন না।

### ৩. K8s CPU limit + GC

JVM/Node app-এ GC pause-এর সময় short CPU spike। `cpu: limit 1` দিয়ে container throttle হয়ে GC ১০x বেশি সময় নেয় → tail latency disaster।

### ৪. Affinity ভুলে long-running container-এ

`taskset -c 0` দিয়ে সব worker একই কোরে — অন্য কোর idle, throughput 1/N। প্রোডাকশনে affinity carefully।

### ৫. CFS bandwidth bug (older kernel < 5.4)

K8s + CFS quota = "200ms throttle" bug — fixed in Linux 5.4+। পুরোনো EKS/GKE node-এ আপগ্রেড করুন।

### ৬. Load average ≠ CPU utilization

Disk-bound process হলে load 20 হতে পারে CPU 20% utilization-এও। `vmstat 1`-এর `wa` দেখুন।

### ৭. Node.js cluster বনাম worker_threads

CPU-bound = `worker_threads` (V8 isolated, same process)
I/O-bound = `cluster` (separate process, কম complexity)
Mixed = দুটোই

---

## 📊 সারমর্ম

```
┌────────────────────────────────────────────────────────────────┐
│ ১. Linux scheduler-এ class: stop > dl > rt > fair > idle       │
│ ২. সাধারণ সব app CFS (fair) class-এ — vruntime-based            │
│ ৩. nice -20 to +19; 0 default; lower = higher priority         │
│ ৪. SCHED_FIFO/RR = hard real-time, যত্ন নিয়ে use করুন           │
│ ৫. context switch ~1-5 μs; cs/sec দিয়ে contention বুঝুন         │
│ ৬. taskset দিয়ে CPU pin; numactl দিয়ে NUMA-aware                │
│ ৭. Load = R + D tasks; 1m vs 15m trend বলে                      │
│ ৮. cgroup cpu.max = throttle, cpu.weight = relative share      │
│ ৯. K8s CPU limit + GC = tail latency bomb                       │
│ ১০. eBPF (runqlat, cpudist) = modern observability             │
└────────────────────────────────────────────────────────────────┘
```

> ➡️ পরবর্তী: [system-calls.md](./system-calls.md) — অ্যাপ কীভাবে কার্নেলকে কথা বলে।
