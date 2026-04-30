# 🐧 Linux Kernel Architecture

> Kernel হলো সিস্টেমের "OS" — বাকি সব (shell, app, container) তার client।
> Linux **monolithic** kernel — সব subsystem একই address space-এ। কিন্তু module/eBPF দিয়ে runtime extensibility পায়।

---

## 📖 সূচিপত্র

- [Monolithic vs Microkernel](#-monolithic-vs-microkernel)
- [Linux subsystems map](#-linux-subsystems-map)
- [User space vs Kernel space, Ring 0](#-user-space-vs-kernel-space-ring-0)
- [Syscall path](#-syscall-path)
- [Kernel modules](#-kernel-modules)
- [/proc ও /sys](#-proc-ও-sys)
- [dmesg, kernel logs, taint](#-dmesg-kernel-logs-taint)
- [Kernel build/config basics](#-kernel-buildconfig-basics)
- [eBPF — programmable kernel](#-ebpf--programmable-kernel)
- [bpftrace ও BCC tools](#-bpftrace-ও-bcc-tools)
- [Real-world: bKash debugging](#-real-world-bkash-debugging)

---

## 📌 Monolithic vs Microkernel

```
Monolithic (Linux):                    Microkernel (Mach, L4, Minix):

┌────────────────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
│  app  │  app  │  app   │              │  app │ │  fs  │ │ net  │  user
└───┬───┴───┬───┴────┬───┘              │      │ │ srv  │ │ srv  │  servers
    │ syscall        │                  └───┬──┘ └──┬───┘ └──┬───┘
    ▼                ▼                      │ IPC   │        │
┌─────────────────────────┐                 ▼       ▼        ▼
│  Kernel (one binary)    │              ┌──────────────────────┐
│  fs, net, sched, mm,    │              │   microkernel         │
│  drivers — all together │              │   (sched + IPC only)  │
└─────────────────────────┘              └──────────────────────┘
        ring 0                                 ring 0
fast (no IPC), bigger TCB                small TCB, IPC overhead
```

| বৈশিষ্ট্য         | Monolithic (Linux)                | Microkernel (L4, QNX, seL4)        |
|------------------|------------------------------------|-------------------------------------|
| Code size in ring 0 | বড় (millions of LoC)             | ছোট (10-100k LoC)                   |
| Driver crash     | পুরো kernel down                   | শুধু সেই driver server crash         |
| Performance      | দ্রুত (function call)              | slow (IPC overhead)                 |
| Verifiability    | কঠিন                                | seL4 formally verified               |
| ব্যবহার         | Linux, BSD, Windows (hybrid)       | QNX (অটোমোটিভ), seL4 (defense)      |

**Linux pragmatic:** monolithic but **modular** — runtime কিছু code load/unload করা যায় (kernel module)। eBPF আরও safer — sandboxed code kernel-এ run।

---

## 📌 Linux subsystems map

```
┌─────────────────────────────────────────────────────────────────┐
│                       User space                                 │
│   bash, php-fpm, node, nginx, systemd, ...                       │
└─────────────────────────────────────────────────────────────────┘
                            │ syscall (int 0x80 / syscall / sysenter)
═══════════════════════════════════════════════════════════════════
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  System Call Interface  (open, read, write, fork, mmap, ...)    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Process  │ │  Memory  │ │  Virtual │ │  Network │           │
│  │   Mgmt   │ │   Mgmt   │ │  FileSys │ │   Stack  │           │
│  │ scheduler│ │  (mm)    │ │  (VFS)   │ │  (TCP/IP)│           │
│  │ signals  │ │  paging  │ │ ext4/xfs │ │ netfilter│           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │   IPC    │ │ Drivers  │ │ Security │ │   eBPF   │           │
│  │ pipe/shm │ │ char/blk │ │ LSM,     │ │ verifier │           │
│  │ futex    │ │ network  │ │ seccomp  │ │  + JIT   │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                                                                 │
│  Architecture-specific (x86_64, arm64, riscv)                   │
└─────────────────────────────────────────────────────────────────┘
                            │
═══════════════════════════════════════════════════════════════════
                            ▼
                       Hardware
              CPU, RAM, NIC, NVMe, GPU, ...
```

**Subsystem-wise উদাহরণ:**

| Subsystem    | source dir            | কী করে                                    |
|--------------|-----------------------|--------------------------------------------|
| Process mgmt | `kernel/sched/`       | task_struct, CFS, EEVDF (6.6+) scheduler  |
| Memory mgmt  | `mm/`                 | page allocator, slab, mmap, swap          |
| VFS          | `fs/`                 | open/read/write generic layer             |
| Filesystems  | `fs/ext4/`, `fs/xfs/` | concrete FS implementations                |
| Net stack    | `net/`                | sockets, IP, TCP, netfilter               |
| Drivers      | `drivers/`            | devicedrivers (NIC, block, USB, GPU)      |
| eBPF         | `kernel/bpf/`         | verifier, JIT, maps                       |
| Security     | `security/`           | LSM (SELinux, AppArmor, BPF-LSM)          |

---

## 📌 User space vs Kernel space, Ring 0

x86 CPU-এ ৪টি **privilege level** (ring 0–3)। Linux মাত্র দুটি ব্যবহার করে: ring 0 (kernel) ও ring 3 (user)।

```
                Ring 3 (user)
                  ↕  syscall / interrupt
                Ring 0 (kernel)
                  ↕  hardware

  user code:   `open("/etc/passwd", O_RDONLY)`
                       │
                       ▼  CPU instruction: syscall
                  trap into ring 0
                       ▼
  kernel:      sys_open() — VFS lookup → ext4 → block I/O
                       ▼  iret
                  return to ring 3 with fd
```

Memory split (x86_64):

```
0xFFFFFFFF_FFFFFFFF  ┌──────────────────┐  
                     │   kernel space    │  upper canonical half
0xFFFF800000000000  ├──────────────────┤
                     │     unused        │
0x00007FFFFFFFFFFF  ├──────────────────┤
                     │    user space     │  per-process
0x0000000000000000  └──────────────────┘
```

প্রতিটি process-এর user-space আলাদা; kernel-space সবার কাছে **একই** (page table-এর shared half)।

---

## 📌 Syscall path

```
         user PHP/Node
              │
              │  glibc wrapper:  read(fd, buf, n)
              ▼
     ┌──────────────────────┐
     │  syscall instruction  │   (x86_64: syscall, arm64: svc #0)
     │  rax = sys_read num   │
     │  rdi/rsi/rdx = args   │
     └──────────────────────┘
              │ trap to ring 0
              ▼
     ┌──────────────────────────┐
     │  entry_SYSCALL_64 (asm)   │
     │  → sys_read() → vfs_read │
     │  → ext4_file_read_iter   │
     │  → block I/O / page cache│
     └──────────────────────────┘
              │ sysret
              ▼
       back to user, rax=bytes
```

```bash
# Trace syscalls
$ strace -e trace=openat,read,write -p 1234

# কোন syscall কতবার (production load profile)
$ strace -c -p 1234
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.00    0.234567        1234       190           epoll_wait
 22.00    0.110000          50      2200           read
 ...
```

> 💡 **PHP-FPM cold request**: ~50 syscall। নিরীহ মনে হয়, কিন্তু 10k req/s × 50 = 500k syscall/s — strace overhead enormous, প্রোডাকশনে careful।

---

## 📌 Kernel modules

`.ko` ফাইল — runtime kernel-এ load করা যায়। driver, filesystem, netfilter ইত্যাদি।

```bash
# Load করা modules
$ lsmod | head
Module                  Size  Used by
overlay               155648  3
br_netfilter           32768  0
xt_conntrack           16384  4
nf_conntrack          172032  4 xt_conntrack,nf_nat,...

# Kernel-এ একটি module-এর info
$ modinfo overlay
filename:       /lib/modules/6.5.0-21-generic/kernel/fs/overlayfs/overlay.ko
license:        GPL
description:    Overlay filesystem
author:         Miklos Szeredi <miklos@szeredi.hu>
parm:           ovl_check_copy_up:bool
...

# Load
$ sudo modprobe overlay
$ sudo insmod ./mymodule.ko key=value

# Unload
$ sudo rmmod overlay      # only if no user

# Auto-load config
$ cat /etc/modules-load.d/k8s.conf
br_netfilter
overlay
```

> 💡 K8s host bootstrap-এ `br_netfilter` ও `overlay` module load লাগে — kubeadm preflight check করে।

### সাধারণ production module checklist

```bash
# Networking-এ দরকারি modules
$ lsmod | grep -E 'nf_|xt_|ip_|br_|vxlan'

# eBPF support
$ cat /boot/config-$(uname -r) | grep -E 'CONFIG_BPF|CONFIG_DEBUG_INFO_BTF'
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_DEBUG_INFO_BTF=y       # CO-RE eBPF এর জন্য lagmust
```

---

## 📌 /proc ও /sys

দুটি **virtual** ফাইলসিস্টেম (no disk, kernel data exposed as files)।

### /proc — process + system info

```bash
$ ls /proc
1  10  100  ...  cpuinfo  meminfo  uptime  loadavg  net  sys

# CPU info
$ cat /proc/cpuinfo | grep "model name" | head -1
model name : Intel(R) Xeon(R) Gold 6248R CPU @ 3.00GHz

# Memory
$ cat /proc/meminfo | head
MemTotal:       16382544 kB
MemFree:         1234560 kB
MemAvailable:   12345678 kB
Buffers:          234560 kB
Cached:         8901234 kB
SwapTotal:       4194304 kB

# একটা process-এর info
$ ls /proc/1234/
cgroup  cmdline  cwd  environ  exe  fd/  limits  maps  mountinfo
ns/  oom_score  smaps  stat  status  syscall  task/

$ cat /proc/1234/status | grep -E 'Vm|Threads|State'
State:    R (running)
VmPeak:   2345678 kB
VmRSS:    789012 kB
Threads:  16

$ cat /proc/1234/limits
Limit                     Soft Limit    Hard Limit    Units
Max open files            1024          1048576       files
Max memory size           unlimited     unlimited     bytes

# net stat
$ cat /proc/net/tcp | head      # TCP connection table
$ cat /proc/net/dev             # interface stats
```

### /sys — sysfs, kernel object hierarchy

```bash
$ ls /sys
block  bus  class  dev  devices  firmware  fs  kernel  module  power

# একটি disk-এর info
$ cat /sys/block/nvme0n1/queue/scheduler
none [mq-deadline] kyber bfq

# scheduler change (production-এ tuning)
$ echo 'kyber' > /sys/block/nvme0n1/queue/scheduler

# একটি module-এর parameter
$ cat /sys/module/nf_conntrack/parameters/hashsize
65536

# CPU online/offline
$ echo 0 > /sys/devices/system/cpu/cpu7/online    # CPU 7 disable

# Network interface stats
$ cat /sys/class/net/eth0/statistics/rx_bytes
$ cat /sys/class/net/eth0/mtu
1500
```

### sysctl — runtime kernel tuning

```bash
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1

# Persistent
$ cat /etc/sysctl.d/99-prod.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
vm.swappiness = 10
fs.file-max = 2097152
$ sysctl --system
```

---

## 📌 dmesg, kernel logs, taint

```bash
# Kernel ring buffer
$ dmesg | tail -20
[12345.678] eth0: link is up
[12346.123] systemd[1]: Started Daily apt upgrade.
[12350.001] Out of memory: Killed process 9081 (java) total-vm:4G,...

# Persistent (journald)
$ journalctl -k                    # kernel only
$ journalctl --since "1 hour ago"

# Real-time
$ dmesg -wH

# Taint flags — module/feature যেগুলো support invalidate করতে পারে
$ cat /proc/sys/kernel/tainted
0       # 0 = clean
# 4096 = unsigned module loaded
# 1    = proprietary module
```

> ⚠️ Daraz incident: কন্টেইনার host OOM kill হচ্ছিল। `dmesg | grep -i oom`-এ `Out of memory: Killed process` log। investigation: cgroup memory limit = host limit, একটা batch job memory leak করছিল।

---

## 📌 Kernel build/config basics

```bash
# কোন kernel চলছে?
$ uname -r
6.5.0-21-generic

# build time config
$ cat /boot/config-$(uname -r) | grep -E 'CONFIG_(SMP|PREEMPT|HZ|NUMA)'
CONFIG_SMP=y
CONFIG_PREEMPT_VOLUNTARY=y
CONFIG_HZ_250=y
CONFIG_NUMA=y

# Kernel parameters (boot time)
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-6.5 root=UUID=... ro console=tty1 transparent_hugepage=madvise
```

**Source build outline (rare in prod, common in dev):**

```bash
git clone --depth 1 https://github.com/torvalds/linux
cd linux
make defconfig
make menuconfig                # ncurses-based config UI
make -j$(nproc)
sudo make modules_install install
update-grub
```

> 💡 প্রোডাকশনে Ubuntu/Debian/RHEL stock kernel-ই use করা হয়। Custom build দরকার পড়ে শুধু eBPF/XDP feature যেগুলো old kernel-এ নেই।

---

## 📌 eBPF — programmable kernel

eBPF (extended Berkeley Packet Filter) Linux-এর **most important feature of the decade**। User-space থেকে ছোট sandboxed program kernel-এ load করা যায় — observability, networking, security সব revolutionize হয়েছে।

```
   user-space loader (libbpf, bpftrace, Cilium)
        │  bpf() syscall
        ▼
   ┌──────────────────────┐
   │  Verifier             │  সব path proven safe (no infinite loop, no
   │  (kernel)             │   uninitialized memory, bounded execution)
   └──────────────────────┘
        │ pass
        ▼
   ┌──────────────────────┐
   │  JIT compile to native │
   └──────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────┐
   │  Hook attach:                            │
   │   - kprobe / kretprobe (kernel function) │
   │   - uprobe (user function)               │
   │   - tracepoint (stable kernel events)    │
   │   - XDP (NIC driver, fastest)            │
   │   - tc (traffic control)                 │
   │   - cgroup (network, syscall, sock)      │
   │   - LSM (security hook)                  │
   │   - perf event (CPU sample)              │
   └─────────────────────────────────────────┘
```

### eBPF maps — kernel↔user shared data

| Map type             | use                              |
|----------------------|----------------------------------|
| `BPF_MAP_TYPE_HASH`  | key/value — counters, lookup     |
| `BPF_MAP_TYPE_ARRAY` | indexed                          |
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | high-rate events to user |
| `BPF_MAP_TYPE_RINGBUF`| modern alternative              |
| `BPF_MAP_TYPE_LRU_HASH`| auto-evict                     |

### Use cases

- **Observability** — bcc tools, bpftrace, Pixie
- **Networking** — Cilium (K8s CNI), Katran (Facebook L4 LB), Calico
- **Security** — Falco, Tetragon (runtime threat detection)
- **Performance** — Profiling, scheduler tuning

---

## 📌 bpftrace ও BCC tools

`bpftrace` — eBPF-এর "awk for tracing"। one-liner দিয়ে kernel-level visibility।

```bash
# Install
$ apt install bpftrace bpfcc-tools

# 1. কোন process file open করছে?
$ bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%s -> %s\n", comm, str(args->filename));
}'
nginx -> /etc/nginx/nginx.conf
php-fpm -> /var/www/index.php
node -> /etc/resolv.conf

# 2. TCP connect latency (per process)
$ bpftrace -e '
kprobe:tcp_v4_connect { @start[tid] = nsecs; }
kretprobe:tcp_v4_connect /@start[tid]/ {
    @ns[comm] = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'
@ns[curl]:
[1K, 2K)    1 |@                              |
[2K, 4K)    8 |@@@@@@@@@@                     |
[4K, 8K)   45 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
...

# 3. কোন process কত memory allocate করছে (uprobe libc malloc)
$ bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc {
    @bytes[comm] = sum(arg0);
}'

# 4. Slow disk I/O culprit
$ biolatency-bpfcc 5
$ biotop-bpfcc

# 5. Run queue latency (scheduler delay) — bKash p99 spike investigation
$ runqlat-bpfcc 5

# 6. Kill uncovered syscall
$ killsnoop-bpfcc

# 7. TCP retransmits — packet loss debug
$ tcpretrans-bpfcc

# 8. file open frequency
$ opensnoop-bpfcc -p $(pgrep -f php-fpm)
```

### BCC pre-built tools (in `bpfcc-tools`)

```
execsnoop      নতুন process trace
opensnoop      file open trace
biotop         I/O top per process
biosnoop       per I/O details
biolatency     I/O latency histogram
ext4slower     slow ext4 ops
tcpconnect     new TCP connect
tcpaccept      accepted connection
tcpretrans     retransmits
runqlat        runqueue latency
profile        CPU profile
oomkill        OOM kill log
killsnoop      kill() syscall
funccount      function call count
stackcount     stack samples
```

> 💡 **bKash on-call playbook:** API latency spike হলে `runqlat`, `tcpretrans`, `biolatency` — three commands cover ৮০% root cause।

---

## 📌 Real-world: bKash debugging

### Case 1: "API p99 spike but CPU low"

```bash
# CPU low মানে কী wait করছে? scheduler delay?
$ runqlat-bpfcc 10
        usecs   : count   distribution
        0->1    : 1240    |@@@@@@@@                    |
        2->3    : 4567    |@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
        4->7    : 890     |@@@@@                       |
       16->31   : 234     |@                           |
      512->1023 : 45      |                            |
     1024->2047 : 89      |                            |   ← outlier!

# 89 events 1ms+ scheduler delay — কোন process?
$ runqlen-bpfcc      # which CPU has long queue
```

Root cause: cgroup CPU `limit=200m` → CFS throttling। ফিক্স: limit `1000m`।

### Case 2: "Container restarts intermittently"

```bash
# OOM kill check
$ dmesg -T | grep -i "oom\|killed" | tail
[Mon Jan 15 14:32:01] Memory cgroup out of memory: Killed process 9081 (php-fpm) ...

# eBPF
$ oomkill-bpfcc
14:32:01 Triggered by PID 9081 ("php-fpm"), OOM kill of PID 9081 ("php-fpm"),
         pages 524288, loadavg: 5.21 4.89 3.12

# Container limit
$ kubectl describe pod payment-7b
  Limits:    memory: 512Mi
```

Solution: PHP `memory_limit=128M` per worker, `pm.max_children` কমানো, K8s memory limit বাড়ানো 1Gi।

### Case 3: "Mysterious slow file write"

```bash
$ ext4slower-bpfcc 1
TIME     COMM           PID    T BYTES   OFF_KB   LAT(ms) FILENAME
14:32:11 mysqld         1234   S 4096    523      245.32  ibdata1
14:32:11 nginx          5678   W 8192    1234       3.21  access.log
14:32:12 mysqld         1234   F 0       0        198.45  ib_logfile0    ← fsync slow!
```

→ disk I/O contention। `iotop` confirm — backup process হাতে নিয়েছে। `ionice` দিয়ে নিচু priority।

---

## 🎯 সংক্ষেপে

- Linux **monolithic** কিন্তু modular (kernel module + eBPF)।
- 8 major subsystems: process, memory, VFS, net, IPC, drivers, security, eBPF।
- User ⇄ kernel boundary = **syscall**; trace করুন `strace`/`bpftrace` দিয়ে।
- `/proc`, `/sys`, `sysctl` — kernel state read/tune করার মূল interface।
- Kernel module runtime extensibility, কিন্তু eBPF **safer + observable**।
- eBPF + bpftrace/BCC = প্রোডাকশন debugging-এর golden tool। strace/perf-এর চেয়ে অনেক কম overhead।
- `runqlat`, `biolatency`, `tcpretrans`, `oomkill`, `opensnoop` — every SRE-র toolbox।

> 🇧🇩 **bKash SRE playbook:** alert fire → SSH → `dmesg`, `runqlat`, `biolatency`, `tcpretrans` — 5 মিনিটে root cause। Custom kernel build কখনো প্রয়োজন হয়নি, BTF-enabled stock Ubuntu kernel + bpftrace যথেষ্ট।  
> Daraz observability = Pixie (eBPF-based APM) + Falco (runtime security)।  
> Pathao Cilium (CNI based on eBPF) দিয়ে network policy enforcement।
