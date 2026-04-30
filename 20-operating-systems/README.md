# 🐧 অপারেটিং সিস্টেম ও কার্নেল ডিপ ডাইভ (Operating Systems & Kernel)

> অ্যাপ্লিকেশন কোডের নিচে যা ঘটছে — প্রসেস, মেমোরি, সিস্টেম কল, I/O মডেল, কন্টেইনার আইসোলেশন।
> প্রতিটি টপিকে **Linux kernel** পার্স্পেক্টিভ, PHP/Node.js উদাহরণ, ASCII ডায়াগ্রাম এবং বাংলাদেশী প্রোডাকশন সার্ভার (bKash, Daraz, Pathao, DigitalOcean BD VPS) থেকে রিয়েল-ওয়ার্ল্ড উদাহরণ।

---

## 📂 টপিক সূচি

| # | বিষয় | ফাইল | মূল বিষয়বস্তু |
|---|-------|------|--------------|
| ১ | প্রসেস ও থ্রেড | [processes-threads.md](./processes-threads.md) | PCB, fork/exec, NPTL, clone(), task_struct, zombie/orphan |
| ২ | মেমোরি ম্যানেজমেন্ট (কার্নেল) | [memory-management.md](./memory-management.md) | Virtual memory, paging, MMU, TLB, OOM killer, CoW |
| ৩ | CPU শিডিউলিং | [cpu-scheduling.md](./cpu-scheduling.md) | CFS, vruntime, nice, SCHED_FIFO, NUMA, taskset |
| ৪ | সিস্টেম কল | [system-calls.md](./system-calls.md) | User/Kernel space, syscall ABI, vDSO, strace, seccomp |
| ৫ | ফাইল সিস্টেম | [file-systems.md](./file-systems.md) | VFS, ext4/XFS, inode, page cache, journaling, fsync |
| ৬ | I/O মডেল | [io-models.md](./io-models.md) | select/poll/epoll, kqueue, io_uring, C10K → C10M |
| ৭ | IPC | [ipc.md](./ipc.md) | pipe, FIFO, shared memory, signals, futex, UNIX socket |
| ৮ | Namespaces ও cgroups | [namespaces-cgroups.md](./namespaces-cgroups.md) | Container internals, PID/NET/MNT NS, cgroup v2 |
| ৯ | Linux Kernel আর্কিটেকচার | [linux-kernel-architecture.md](./linux-kernel-architecture.md) | Monolithic vs microkernel, modules, ring 0/3, /proc, /sys |
| ১০ | Boot ও Init | [boot-init.md](./boot-init.md) | BIOS/UEFI → GRUB → kernel → initramfs → systemd |
| ১১ | কার্নেল নেটওয়ার্কিং স্ট্যাক | [kernel-networking-stack.md](./kernel-networking-stack.md) | NIC → softirq → netfilter → socket buffer → eBPF/XDP |

---

## 🗺️ টপিক সম্পর্ক ডায়াগ্রাম

```
                ┌────────────────────────────────────────────┐
                │   ইউজার অ্যাপ্লিকেশন (PHP-FPM, Node.js)      │
                └──────────────────────┬─────────────────────┘
                                       │ glibc / libc
                                       ▼
        ┌──────────────────────── SYSTEM CALLS (4) ─────────────────────────┐
        │      read, write, open, mmap, fork, execve, futex, epoll_wait     │
        └─────────────────────────────────┬─────────────────────────────────┘
                                          │  (user → kernel transition)
                                          ▼
   ╔══════════════════════════════ LINUX KERNEL (9) ══════════════════════════╗
   ║                                                                          ║
   ║   ┌──────────────┐   ┌─────────────┐   ┌───────────────┐   ┌──────────┐ ║
   ║   │ PROCESS      │   │ MEMORY      │   │ FILE SYSTEM    │   │  IPC     │ ║
   ║   │ MGMT (1)     │◀─▶│ MGMT (2)    │◀─▶│ VFS (5)        │◀─▶│  (7)     │ ║
   ║   │ task_struct  │   │ page table  │   │ inode/dentry   │   │ pipe/shm │ ║
   ║   └──────┬───────┘   └──────┬──────┘   └────────┬───────┘   └─────┬────┘ ║
   ║          │                  │                   │                 │      ║
   ║          ▼                  ▼                   ▼                 ▼      ║
   ║   ┌──────────────┐   ┌─────────────┐   ┌───────────────┐   ┌──────────┐ ║
   ║   │ SCHEDULER (3)│   │ I/O LAYER(6)│   │ NET STACK (11) │   │ NS+CG(8) │ ║
   ║   │ CFS, RT      │   │ epoll, AIO  │   │ TCP/IP, eBPF  │   │ container│ ║
   ║   └──────────────┘   └─────────────┘   └───────────────┘   └──────────┘ ║
   ║                                                                          ║
   ╚══════════════════════════════════╤═══════════════════════════════════════╝
                                      │  (driver layer)
                                      ▼
                ┌──────────────────────────────────────────┐
                │    HARDWARE: CPU, RAM, Disk, NIC          │
                │    BOOT (10): UEFI → GRUB → kernel        │
                └──────────────────────────────────────────┘
```

---

## 🎯 কেন সিনিয়র ইঞ্জিনিয়ারের OS জানা জরুরি

একজন সিনিয়র ইঞ্জিনিয়ার শুধু ফ্রেমওয়ার্ক জানলে চলে না — প্রোডাকশনে যখন `502 Bad Gateway` হয়, যখন কন্টেইনার OOMKilled হয়, যখন CPU 100% কিন্তু থ্রুপুট কম — তখন OS লেভেলে নামতে হয়।

| পরিস্থিতি | OS জ্ঞান যা দরকার |
|-----------|------------------|
| PHP-FPM worker hang হয়ে গেছে | প্রসেস স্টেট (D = uninterruptible sleep), `strace` দিয়ে syscall ট্রেস |
| Node.js heap বাড়তে বাড়তে OOMKilled | cgroup memory limit, RSS vs virtual, `/proc/<pid>/status` |
| API latency p99 হঠাৎ বেড়ে গেছে | Context switch cost, CPU throttling, page faults |
| `connection refused` যদিও সার্ভার আপ | TCP backlog (`somaxconn`), file descriptor limit, ephemeral port exhaustion |
| Docker কন্টেইনার slow | cgroup CPU quota, namespace overhead, OverlayFS layers |
| Kafka consumer lag বাড়ছে | Disk I/O — page cache miss, fsync cost, dirty_ratio |
| bKash payment API timeout spike | TIME_WAIT exhaustion, syscall storm, NUMA imbalance |
| Daraz checkout 503 | epoll bottleneck, accept queue full, ulimit -n hit |

---

## 🏠 বাংলাদেশী প্রোডাকশন প্রসঙ্গ

| কোম্পানি | OS-লেভেল চ্যালেঞ্জ | সংশ্লিষ্ট টপিক |
|---------|-------------------|---------------|
| **bKash** (পেমেন্ট সার্ভার) | লক্ষাধিক concurrent transaction, low p99 latency দরকার | epoll, CPU affinity, NUMA |
| **Daraz** (ই-কমার্স) | Flash sale-এ ১০ লক্ষ req/min, কন্টেইনার অটো-স্কেল | cgroups, PID namespace, page cache |
| **Pathao** (রাইড-শেয়ারিং) | WebSocket — হাজার হাজার persistent connection | file descriptors, TCP keepalive, epoll ET |
| **Nagad** (পেমেন্ট) | mTLS — প্রচুর crypto syscall, kernel entropy | syscall cost, getrandom, vDSO |
| **Robi/GP** (Telco) | Real-time billing, hard latency budget | SCHED_FIFO, CPU isolation, IRQ affinity |
| **Prothom Alo** (নিউজ) | High traffic spike, CDN miss-এ ফলব্যাক | Nginx worker tuning, page cache, sendfile |
| **Shared Hosting (BD)** | একই VPS-এ ১০০ সাইট, isolation দরকার | cgroups, namespaces, memory_limit |

### সাধারণ BD VPS / সার্ভার স্পেক

```
┌────────────────────────────────────────────────────────────────┐
│ DigitalOcean Singapore / Bangalore (BD ডেভেলপার সবচেয়ে বেশি) │
├────────────────────────────────────────────────────────────────┤
│ Basic Droplet:  1 vCPU, 1GB RAM, 25GB SSD     — $6/mo          │
│ Standard:       2 vCPU, 4GB RAM, 80GB SSD     — $24/mo         │
│ AWS Mumbai:     t3.medium (2 vCPU, 4GB)       — ~$30/mo        │
│ Local DC (BDIX): Aamra/Link3 colocation                         │
└────────────────────────────────────────────────────────────────┘

এই কম-স্পেকের সার্ভারে OS টিউনিং না করলে:
  • ulimit -n (default 1024) দ্রুত শেষ হয় — Node.js/Nginx ফেইল
  • vm.swappiness=60 — মেমোরি প্রেশারে disk swap → latency স্পাইক
  • net.core.somaxconn=128 — high concurrency-তে SYN drop
```

---

## 📊 Process vs Thread vs Coroutine তুলনা সারসংক্ষেপ

```
┌────────────────┬──────────────┬──────────────┬───────────────┐
│  বৈশিষ্ট্য      │   Process    │    Thread    │   Coroutine    │
├────────────────┼──────────────┼──────────────┼───────────────┤
│ Address space  │  আলাদা        │  শেয়ার্ড      │  শেয়ার্ড        │
│ Switch cost    │  ~5-10 μs    │  ~1-2 μs    │  ~50-200 ns    │
│ Memory/each    │  ~2-10 MB    │  ~1-2 MB    │  ~2-8 KB       │
│ Crash isolat.  │   ✅ হ্যাঁ    │   ❌ না       │   ❌ না         │
│ IPC দরকার      │   হ্যাঁ       │   না          │   না            │
│ উদাহরণ         │ PHP-FPM      │ Java Tomcat  │ Node.js, Go    │
└────────────────┴──────────────┴──────────────┴───────────────┘
```

---

## 🛠️ অবজার্ভেবিলিটি কমান্ড চিট-শিট

```bash
# প্রসেস ও থ্রেড
ps -eLf                           # সব thread সহ
top -H -p <pid>                   # প্রতি থ্রেডের CPU
pstree -p                         # process tree

# মেমোরি
free -h                           # সিস্টেম মেমোরি ওভারভিউ
vmstat 1                          # প্রতি সেকেন্ডে paging/IO
cat /proc/<pid>/status            # VmRSS, VmSize, Threads
pmap -x <pid>                     # মেমোরি ম্যাপ

# CPU
mpstat -P ALL 1                   # প্রতি কোরের usage
pidstat -t 1                      # প্রতি থ্রেডের stats
perf top                          # লাইভ CPU profiler

# I/O
iotop                             # প্রতি প্রসেসের disk IO
iostat -x 1                       # ডিস্ক utilization
lsof -p <pid>                     # ওপেন ফাইল/সকেট

# Syscall
strace -c -p <pid>                # সারমর্ম
strace -e trace=network -p <pid>  # শুধু network syscall

# কার্নেল ইভেন্ট
dmesg -T | tail                   # kernel ring buffer
journalctl -k -f                  # systemd kernel log
```

---

## 🎯 শেখার পথ (Learning Path)

```
Step 1: Processes & Threads → প্রোগ্রাম কীভাবে চলে
         │
Step 2: Memory Management → প্রোগ্রাম কীভাবে RAM ব্যবহার করে
         │
Step 3: CPU Scheduling → কে কখন CPU পায়
         │
Step 4: System Calls → অ্যাপ কীভাবে কার্নেলকে কথা বলে
         │
Step 5: I/O Models → হাজার connection কীভাবে handle হয়
         │
Step 6: File Systems → ডেটা কীভাবে ডিস্কে থাকে
         │
Step 7: IPC → প্রসেস-প্রসেস যোগাযোগ
         │
Step 8: Namespaces & cgroups → Docker/K8s কীভাবে কাজ করে
         │
Step 9: Kernel Architecture → পুরো ছবিটা একসাথে
         │
Step 10-11: Boot, Network Stack → বুটআপ থেকে প্যাকেট পর্যন্ত
```

---

> 💡 **টিপ**: প্রতিটি ফাইলে strace/perf/proc কমান্ড দেওয়া আছে — শুধু পড়বেন না, একটা Linux VM-এ চালিয়ে দেখুন। OS শেখার একমাত্র উপায় হাতে-কলমে।
> নতুন হলে [processes-threads.md](./processes-threads.md) দিয়ে শুরু করুন।
