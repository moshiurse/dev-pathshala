# 🧬 প্রসেস ও থ্রেড (Processes & Threads)

> **প্রতিটি রানিং প্রোগ্রাম একটি প্রসেস। প্রতিটি প্রসেসের ভিতরে এক বা একাধিক থ্রেড। কার্নেল উভয়কে `task_struct` হিসেবে দেখে।**

---

## 📖 সূচিপত্র

- [প্রসেস কী, থ্রেড কী](#-প্রসেস-কী-থ্রেড-কী)
- [Process Control Block (PCB) ও task_struct](#-process-control-block-pcb-ও-task_struct)
- [প্রসেস স্টেট মেশিন](#-প্রসেস-স্টেট-মেশিন)
- [fork(), exec(), wait(), exit()](#-fork-exec-wait-exit)
- [Linux clone() ও থ্রেডিং মডেল](#-linux-clone-ও-থ্রেডিং-মডেল)
- [PID, TID, PGID, SID](#-pid-tid-pgid-sid)
- [Zombie ও Orphan প্রসেস](#-zombie-ও-orphan-প্রসেস)
- [প্রসেস ট্রি ও PID 1](#-প্রসেস-ট্রি-ও-pid-1)
- [PHP-FPM, Node.js, Nginx — বাস্তবে](#-php-fpm-nodejs-nginx--বাস্তবে)
- [অবজার্ভেবিলিটি কমান্ড](#-অবজার্ভেবিলিটি-কমান্ড)
- [pitfalls ও real-world bug](#-pitfalls-ও-real-world-bug)

---

## 📌 প্রসেস কী, থ্রেড কী

### সংজ্ঞা

- **প্রসেস (Process)**: একটি স্বাধীন এক্সিকিউশন ইউনিট, যার নিজস্ব ভার্চুয়াল অ্যাড্রেস স্পেস (text/data/heap/stack), ফাইল ডিস্ক্রিপ্টর টেবিল, signal handler, এবং নিরাপত্তা context (UID/GID) আছে।
- **থ্রেড (Thread)**: একটি প্রসেসের ভিতরে একটি লাইটওয়েট এক্সিকিউশন context — নিজস্ব stack ও CPU register set আছে, কিন্তু heap, file descriptors, code segment **শেয়ার করে** একই প্রসেসের অন্য থ্রেডের সাথে।

```
┌──────────────────────────── PROCESS A ────────────────────────────┐
│                                                                    │
│   Code (text)   │  Heap (shared)   │  Globals/BSS (shared)          │
│                 │                  │                                │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐       │
│   │   Thread 1     │  │   Thread 2     │  │   Thread 3     │       │
│   │   ┌─────────┐  │  │   ┌─────────┐  │  │   ┌─────────┐  │       │
│   │   │ Stack   │  │  │   │ Stack   │  │  │   │ Stack   │  │       │
│   │   │ Regs    │  │  │   │ Regs    │  │  │   │ Regs    │  │       │
│   │   │ TLS     │  │  │   │ TLS     │  │  │   │ TLS     │  │       │
│   │   └─────────┘  │  │   └─────────┘  │  │   └─────────┘  │       │
│   └────────────────┘  └────────────────┘  └────────────────┘       │
│                                                                    │
│   File descriptors (shared):  0=stdin  1=stdout  3=socket  4=db   │
│   Signal handlers (shared)                                         │
│   PID = 1234                                                       │
└────────────────────────────────────────────────────────────────────┘
```

### Linux-এর বিশেষত্ব

Linux-এ প্রসেস ও থ্রেডের মধ্যে ভিতরে কোনো বড় পার্থক্য নেই — কার্নেল উভয়কেই **task** বলে। `clone()` syscall-এর কোন ফ্ল্যাগ পাস হয়েছে তার উপর নির্ভর করে এটি প্রসেস হবে নাকি থ্রেড।

```
fork()    →  clone(SIGCHLD)                    → নতুন প্রসেস
pthread   →  clone(CLONE_VM | CLONE_FS |        → নতুন থ্রেড
              CLONE_FILES | CLONE_SIGHAND |       (একই process এর ভিতরে)
              CLONE_THREAD | CLONE_SYSVSEM)
```

---

## 📌 Process Control Block (PCB) ও task_struct

কার্নেল প্রতিটি প্রসেসের জন্য একটি struct রাখে — Linux-এ এর নাম `task_struct` (define করা আছে `include/linux/sched.h`-এ, ১০০০+ লাইন!)। এটিই OS textbook-এর "PCB"।

```
struct task_struct {
    pid_t                pid;             // 1234
    pid_t                tgid;            // Thread Group ID = process এর "PID"
    volatile long        state;           // R, S, D, Z, T
    void                *stack;           // kernel stack
    struct mm_struct    *mm;              // virtual memory descriptor (থ্রেডে শেয়ার্ড)
    struct files_struct *files;           // open file descriptors
    struct fs_struct    *fs;              // cwd, root, umask
    struct signal_struct*signal;          // signal handlers
    struct cred         *cred;            // UID, GID, capabilities
    struct task_struct  *parent;          // parent process
    struct list_head     children;        // child list
    cpumask_t            cpus_allowed;    // CPU affinity mask
    int                  prio, static_prio, normal_prio;
    struct sched_entity  se;              // CFS scheduler entity (vruntime ইত্যাদি)
    u64                  utime, stime;    // user/kernel time
    /* ... আরও শতাধিক ফিল্ড */
};
```

**প্রতিটি `task_struct` আনুমানিক ৭-৮ KB**। Linux-এ একটি প্রসেসের জন্য সর্বনিম্ন **kernel memory cost** এটাই, plus kernel stack (16KB)।

### বাস্তব পর্যবেক্ষণ

```bash
# /proc/<pid>/status থেকে task_struct এর মূল ফিল্ড দেখা যায়
$ cat /proc/$$/status | head -20
Name:   bash
State:  S (sleeping)
Tgid:   12345              ← Thread Group ID
Pid:    12345              ← Task ID
PPid:   12340              ← Parent PID
Uid:    1000  1000  1000  1000
Threads: 1
SigQ:   0/63421
```

---

## 📌 প্রসেস স্টেট মেশিন

Linux-এ প্রসেস কয়েকটি অবস্থায় থাকতে পারে। `ps`-এর `STAT` কলামে এই অক্ষরগুলো দেখা যায়।

```
                         ┌───────────────┐
                         │   NEW         │
                         │ (fork কল হলো) │
                         └───────┬───────┘
                                 │ scheduler queue-এ ঢোকে
                                 ▼
              ┌───────────────────────────────────┐
              │   R   RUNNING / RUNNABLE          │
              │  (CPU পাচ্ছে অথবা CPU পাওয়ার       │
              │   জন্য queue-এ আছে)                │
              └──┬───────────────────┬────────────┘
                 │                   │
   I/O wait      │                   │ time slice শেষ /
   syscall block │                   │ preempt
                 ▼                   │
   ┌────────────────────┐           │
   │  S  Interruptible  │           │
   │     SLEEP          │           │
   │  (signal-এ wake)   │           │
   └─────────┬──────────┘           │
             │  data এলো             │
             └──────────────┬────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │  D  Uninterruptible     │  ⚠️ kill -9 ও কাজ করে না
              │     SLEEP               │     (NFS hang, broken disk)
              │  (disk I/O চলছে)         │
              └─────────────────────────┘

              ┌─────────────────────────┐
              │  T  STOPPED             │  ← Ctrl+Z, SIGSTOP
              └─────────────────────────┘

              ┌─────────────────────────┐
              │  Z  ZOMBIE              │  ← exit() হয়েছে কিন্তু parent
              │  (defunct)              │     এখনও wait() করেনি
              └─────────────────────────┘

              ┌─────────────────────────┐
              │  X  DEAD                │  ← parent wait করল, cleanup শেষ
              └─────────────────────────┘
```

### স্টেট অক্ষর সারসংক্ষেপ

| অক্ষর | অর্থ | উদাহরণ |
|------|-----|--------|
| `R` | Running/Runnable | লুপ চলছে এমন কোড |
| `S` | Interruptible sleep | `read()` syscall-এ wait |
| `D` | Uninterruptible sleep | NFS read hang, disk I/O |
| `T` | Stopped | Ctrl+Z, gdb attach |
| `Z` | Zombie | exit হয়েছে, parent wait করেনি |
| `X` | Dead | reaping সম্পন্ন (সাধারণত দেখা যায় না) |
| `+` | Foreground group | terminal-এ চলছে |
| `s` | Session leader | login shell |
| `<` | High priority | nice < 0 |
| `N` | Low priority | nice > 0 |
| `l` | Multi-threaded | `ps -L`-এ দেখুন |

### প্রোডাকশন বাগ

> **bKash incident pattern**: কিছু worker `D` state-এ আটকে আছে, `kill -9` কাজ করছে না। সাধারণত এর মানে — NFS mount unresponsive অথবা SAN storage hang। সমাধান: storage layer ঠিক করা, প্রসেস নিজে থেকে state ছাড়বে না।

---

## 📌 fork(), exec(), wait(), exit()

UNIX-এর মূল প্রসেস তৈরির API চারটি। প্রতিটি Bangladeshi ডেভেলপারের প্রিয় Linux command (যেমন `ls | grep foo`) এই চারটি syscall ব্যবহার করে।

```
┌─────────────────────────────────────────────────────────────┐
│ Parent Process (PID 100)                                     │
│   fd: stdin, stdout, vars: x=10                              │
└────────────────────┬────────────────────────────────────────┘
                     │ fork()
                     ▼
        ┌────────────┴────────────┐
        ▼                         ▼
┌──────────────┐           ┌──────────────┐
│ Parent (100) │           │ Child  (101) │  ← exact copy (CoW)
│ fork() == 101│           │ fork() == 0  │
│ x=10         │           │ x=10         │
└──────┬───────┘           └──────┬───────┘
       │ wait(101)               │ execve("/bin/ls",...)
       │ (block)                 │  ← অ্যাড্রেস স্পেস সম্পূর্ণ বদলে গেল
       │                         ▼
       │                   ┌──────────────┐
       │                   │  /bin/ls চলে  │
       │                   │  (PID একই 101)│
       │                   └──────┬───────┘
       │                          │ exit(0)
       │                          ▼
       │                       ZOMBIE
       │ ◀────────────────────────┘ (exit code নিল)
       ▼
   continue
```

### PHP উদাহরণ — pcntl_fork

```php
<?php
// shared hosting-এ pcntl_* এক্সটেনশন প্রায়ই ডিজেবল করা থাকে — VPS-এ চালান
// install: apt install php-pcntl

$pid = pcntl_fork();

if ($pid === -1) {
    die("fork ব্যর্থ\n");
} elseif ($pid === 0) {
    // চাইল্ড প্রসেস
    echo "চাইল্ড: PID = " . getmypid() . ", PPID = " . posix_getppid() . "\n";
    sleep(2);
    exit(42);  // exit code 42
} else {
    // প্যারেন্ট প্রসেস
    echo "প্যারেন্ট: চাইল্ড PID = $pid\n";
    pcntl_waitpid($pid, $status);
    if (pcntl_wifexited($status)) {
        echo "চাইল্ড exit করেছে code: " . pcntl_wexitstatus($status) . "\n";
    }
}
```

### PHP — pre-fork worker pool (PHP-FPM-এর মতো)

```php
<?php
// একটি সাধারণ pre-fork model — nginx/php-fpm এই প্যাটার্নই ব্যবহার করে
$workers = 4;
$children = [];

for ($i = 0; $i < $workers; $i++) {
    $pid = pcntl_fork();
    if ($pid === 0) {
        // worker
        while (true) {
            // সাধারণত accept() করে কাজ নিত — এখানে শুধু sleep
            echo "Worker " . getmypid() . " কাজ করছে\n";
            sleep(rand(1, 3));
        }
    } else {
        $children[] = $pid;
    }
}

// SIGTERM হলে চাইল্ডদের kill করো
pcntl_signal(SIGTERM, function () use (&$children) {
    foreach ($children as $pid) posix_kill($pid, SIGTERM);
    exit;
});

// reap zombies
while (count($children)) {
    $pid = pcntl_wait($status);
    $children = array_filter($children, fn($p) => $p !== $pid);
}
```

### Node.js উদাহরণ — child_process

```javascript
const { fork, spawn, exec } = require('child_process');

// fork() — শুধু Node.js child এর জন্য (IPC channel দিয়ে)
const child = fork('./worker.js');
child.send({ task: 'process_payment', amount: 1500 });
child.on('message', (msg) => {
    console.log('worker থেকে এলো:', msg);
});

// spawn() — যেকোনো প্রোগ্রাম, stream API
const ps = spawn('ps', ['-ef']);
ps.stdout.on('data', d => console.log(d.toString()));

// exec() — shell command, output buffer-এ আসে (≤ 200KB)
exec('ls -la /var/log', (err, stdout) => console.log(stdout));
```

### Node.js — cluster (built-in pre-fork)

```javascript
// bKash/Daraz API তে multi-core ব্যবহারের সবচেয়ে সাধারণ প্যাটার্ন
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
    const cpus = os.cpus().length;
    console.log(`Master ${process.pid} starting, forking ${cpus} workers`);

    for (let i = 0; i < cpus; i++) cluster.fork();

    cluster.on('exit', (worker, code) => {
        console.log(`Worker ${worker.process.pid} died (${code}), restarting...`);
        cluster.fork();  // auto-restart
    });
} else {
    require('./server');  // worker handles requests
    console.log(`Worker ${process.pid} started`);
}
```

> 💡 **Internals**: `cluster.fork()` আসলে `child_process.fork()` কল করে, যা একই Node.js binary-কে আবার চালায় (`execve` সহ)। সব worker একই TCP socket শেয়ার করে — কার্নেল `SO_REUSEPORT` দিয়ে accept লোড ভাগ করে।

---

## 📌 Linux clone() ও থ্রেডিং মডেল

`fork()` আসলে glibc-এ `clone()` syscall-এর একটি wrapper।

```c
// fork() এর equivalent
pid_t pid = clone(child_func, stack_top,
                  SIGCHLD,  // parent কে notify করো child মারা গেলে
                  args);

// pthread_create() এর equivalent
clone(thread_func, stack_top,
      CLONE_VM        |  // address space share
      CLONE_FS        |  // cwd, root, umask share
      CLONE_FILES     |  // file descriptor table share
      CLONE_SIGHAND   |  // signal handlers share
      CLONE_THREAD    |  // same thread group (TGID একই)
      CLONE_SYSVSEM   |  // semaphore undo share
      CLONE_SETTLS    |  // TLS area
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID,
      args);
```

### থ্রেডিং মডেল

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│  মডেল              │  N user → M kernel    │  উদাহরণ              │
├────────────────────┼──────────────────────┼──────────────────────┤
│  1:1 (kernel)      │  প্রতি user thread    │  Linux NPTL          │
│                    │  = একটি kernel thread │  Java HotSpot, C++   │
│                    │  সরাসরি              │  pthread             │
├────────────────────┼──────────────────────┼──────────────────────┤
│  N:1 (user)        │  সব thread একটি      │  পুরনো GreenThread,   │
│                    │  kernel task-এ        │  GIL-এর আগের Python   │
├────────────────────┼──────────────────────┼──────────────────────┤
│  M:N (hybrid)      │  M user thread →     │  Go goroutine,       │
│                    │  N OS thread (M>>N)  │  Erlang BEAM,        │
│                    │  user-space scheduler │  Java Project Loom   │
└────────────────────┴──────────────────────┴──────────────────────┘
```

**Linux-এ NPTL (Native POSIX Thread Library, glibc 2.3.2+)** প্রতিটি pthread-কে একটি kernel task হিসেবে তৈরি করে — এটাই সবচেয়ে predictable, scheduler-friendly মডেল। আপনার Java Tomcat, MySQL, Nginx-worker_threads সবাই এই মডেলে চলে।

---

## 📌 PID, TID, PGID, SID

```
Session                 (লগইন সেশন, controlling terminal)
└── Process Group       (job; Ctrl+C যেটাকে signal পাঠায়)
    └── Process (PID)   (একটি address space)
        └── Thread (TID) (একটি execution context)
```

| ID | অর্থ | system call |
|----|-----|-------------|
| `PID` | Process ID (আসলে main thread-এর TID) | `getpid()` |
| `TID` | Thread ID (kernel-level প্রতিটি task) | `gettid()` |
| `TGID` | Thread Group ID = PID-এর সমান | task_struct→tgid |
| `PPID` | Parent PID | `getppid()` |
| `PGID` | Process Group ID | `getpgid()` |
| `SID` | Session ID | `getsid()` |

```bash
# মাল্টি-থ্রেডেড প্রসেসের TID গুলো দেখা
$ ps -eLf | grep nginx | head
USER  PID    PPID  LWP   ...   CMD
root  1024   1     1024        nginx: master process
www   1025   1024  1025        nginx: worker process
www   1025   1024  1026        nginx: worker process  ← একই PID, ভিন্ন LWP=TID
```

---

## 📌 Zombie ও Orphan প্রসেস

### Zombie (Z, defunct)

চাইল্ড `exit()` করেছে, কিন্তু প্যারেন্ট এখনও `wait()`/`waitpid()` কল করেনি। কার্নেল exit code রাখার জন্য `task_struct` ধরে রাখে — কিন্তু address space, files সব ফ্রি করে দেয়।

```bash
$ ps aux | awk '$8=="Z"'
user  4521  0.0  0.0   0    0  ?  Z  10:00  0:00  [worker] <defunct>
```

**সমাধান**:
- প্যারেন্টে `wait()` কল করো অথবা
- `signal(SIGCHLD, SIG_IGN)` সেট করো — কার্নেল auto-reap করবে
- প্যারেন্ট মারা গেলে init/systemd auto-reap করে

### Orphan

প্যারেন্ট আগে মারা গেছে, চাইল্ড এখনও চলছে। এদের নতুন প্যারেন্ট হয় **PID 1 (init/systemd)**, যে নিয়মিত `wait()` করে।

```
আগে:                            পরে:
PID 1 (init)                    PID 1 (init)
 └── PID 100 (parent)            └── PID 200 (orphan, এখন init এর child)
      └── PID 200 (child)
                                (PID 100 মারা গেছে)
```

> ⚠️ **Docker pitfall**: কন্টেইনারে PID 1 হিসেবে যদি `node app.js` চালান, এটি signal forward এবং zombie reap কোনোটাই করে না। ফলে multi-process Node app-এ zombie জমে। সমাধান: `tini` বা `dumb-init` ব্যবহার করুন — `docker run --init`।

---

## 📌 প্রসেস ট্রি ও PID 1

```bash
$ pstree -p 1
systemd(1)─┬─NetworkManager(823)
           ├─cron(901)
           ├─nginx(1024)─┬─nginx(1025)    ← worker
           │             ├─nginx(1026)
           │             └─nginx(1027)
           ├─php-fpm(1100)─┬─php-fpm(1101)
           │               ├─php-fpm(1102)
           │               └─php-fpm(1103)
           ├─mysqld(1200)───{mysqld}(1201..1250)  ← 50 threads
           └─sshd(1500)───sshd(2100)───bash(2101)───vim(2150)
```

### init system

| init | বৈশিষ্ট্য | কোথায় |
|------|----------|--------|
| `SysV init` | পুরোনো, sequential script | পুরোনো CentOS 6 |
| `Upstart` | event-driven | Ubuntu 9.10–14.10 |
| `systemd` | parallel, dependency-graph, cgroup | প্রায় সব আধুনিক distro |
| `tini` / `dumb-init` | minimal, reaper | Docker container PID 1 |

---

## 📌 PHP-FPM, Node.js, Nginx — বাস্তবে

### Nginx (multi-process, event-driven)

```
master (root, 1 process)
  ├── worker 1 (www-data) ── epoll loop ── হাজার connection
  ├── worker 2 (www-data) ── epoll loop
  ├── worker 3 (www-data) ── epoll loop
  └── worker 4 (www-data) ── epoll loop
                            (worker_processes auto = CPU count)
```

Master কাজ: config reload (`SIGHUP`), worker spawn/respawn। workers কাজ: actual request handling।

### PHP-FPM (multi-process, blocking)

```
master
  └── pool "www"
       ├── worker 1 (sleeping, accept-এ block)
       ├── worker 2 (running PHP code)
       ├── worker 3 (waiting MySQL response — D state হতে পারে)
       └── ... (pm.max_children পর্যন্ত)
```

প্রতিটি worker একটি request পুরো শেষ করে — synchronously। PHP-FPM-এর সিদ্ধান্ত: **প্রতি request-কে আলাদা প্রসেস দাও** → memory leak ভয় নেই, crash isolation আছে, কিন্তু প্রতি worker ~30-50MB।

```ini
; php-fpm pool config (production tuning, BD VPS 4GB RAM)
pm = dynamic
pm.max_children = 50           ; 50 × 40MB = 2GB
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
pm.max_requests = 500          ; 500 request পর worker recycle (memory leak ঠেকাতে)
```

### Node.js (single-thread + cluster)

```
cluster master
  ├── worker 1 (Node V8 + libuv event loop)
  ├── worker 2
  ├── worker 3
  └── worker 4
```

প্রতি worker **একটি event loop** চালায় — হাজার connection হ্যান্ডেল করে। CPU-bound কাজের জন্য `worker_threads`।

---

## 📌 অবজার্ভেবিলিটি কমান্ড

```bash
# সব প্রসেস তালিকা (long form)
ps -ef
ps aux
ps -ejH                # forest view

# একটি প্রসেস ও তার সব থ্রেড
ps -L -p <pid>
ps -eLf | grep nginx

# top — interactive
top
top -H -p <pid>        # প্রতি থ্রেড আলাদা সারিতে
htop                   # color, mouse support, tree view

# প্রসেস ট্রি
pstree -p
pstree -p <pid>

# /proc — সবচেয়ে শক্তিশালী
ls /proc/<pid>/
cat /proc/<pid>/status     # name, state, threads, memory
cat /proc/<pid>/cmdline    # \0 separated argv
cat /proc/<pid>/environ    # env vars
ls -l /proc/<pid>/fd/      # ওপেন file descriptor → কোন ফাইল
cat /proc/<pid>/cgroup     # কোন cgroup এ
cat /proc/<pid>/limits     # ulimit values
ls /proc/<pid>/task/       # সব thread (TID directories)

# parent-child relationship
ps -o pid,ppid,pgid,sid,comm -p <pid>

# সব child দেখা
pgrep -P <ppid>
```

### Real-time process monitoring

```bash
# context switch count
pidstat -w 1

# state distribution sampling
while true; do
  ps -eo state,comm | awk '{print $1}' | sort | uniq -c
  sleep 1
done

# কোন syscall-এ আটকে আছে
cat /proc/<pid>/wchan      # kernel function নাম
cat /proc/<pid>/stack      # kernel stack trace
```

---

## ⚠️ Pitfalls ও Real-world Bug

### ১. Fork bomb (incident in BD shared hosting)

```bash
:(){ :|:& };:    # ☠️ চালাবেন না — সিস্টেম freeze
```

প্রতিরোধ:
```bash
# /etc/security/limits.conf
*  soft  nproc  4096
*  hard  nproc  8192
```

### ২. PHP-FPM "504 Gateway Timeout"

লক্ষণ: Nginx বলে 504, কিন্তু সার্ভার লোড কম। কারণ: সব PHP-FPM worker `D` state-এ MySQL slow query-তে আটকে।

```bash
# কারণ খুঁজুন
ps -eo pid,state,wchan,cmd | grep php-fpm | grep ' D '
strace -p <stuck_pid>      # syscall দেখুন
```

### ৩. Daraz checkout flash sale outage

লক্ষণ: Node.js cluster master CPU 100%, কিন্তু worker-রা idle। কারণ: master `cluster.fork()` loop-এ আটকে — IPC overflow।
সমাধান: `--max-old-space-size`, `numWorkers ≤ os.cpus().length`।

### ৪. Zombie জমে docker container

```bash
docker exec <container> ps aux | grep '<defunct>'
```
সমাধান: `docker run --init ...` অথবা Dockerfile-এ:
```dockerfile
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

### ৫. Node.js child_process zombie leak

```javascript
// ❌ exit handler না দিলে zombie
const ps = spawn('long-running-cmd');

// ✅ ঠিক
ps.on('exit', () => console.log('reaped'));
ps.on('error', e => console.error(e));
```

### ৬. TID confusion in logs

`ps` দেখাচ্ছে nginx worker PID 1025, কিন্তু `top -H` দেখাচ্ছে অনেকগুলো — সব আসলে একই PID-এর thread (TID)। `ps -L` দিয়ে confirm করুন।

---

## 📊 সারমর্ম — মনে রাখার মূল পয়েন্ট

```
┌─────────────────────────────────────────────────────────────────┐
│ ১. Linux-এ process ও thread একই struct (task_struct)            │
│ ২. fork() = clone(SIGCHLD), pthread = clone(CLONE_VM|...)       │
│ ৩. State: R, S, D, Z, T — D state-এ kill -9 কাজ করে না         │
│ ৪. Zombie = exit হয়েছে, parent reap করেনি                       │
│ ৫. Orphan = parent মৃত, init adopt করেছে                         │
│ ৬. PID 1 (init/systemd) সব orphan reap করে                     │
│ ৭. PHP-FPM = pre-fork; Nginx = master+worker; Node = cluster   │
│ ৮. Docker container-এ tini/dumb-init দিয়ে PID 1 ঠিক করুন        │
│ ৯. /proc/<pid>/* হলো প্রসেস inspect করার সোনার খনি              │
│ ১০. ps, top, htop, pstree, pidstat — হাত গরম রাখুন              │
└─────────────────────────────────────────────────────────────────┘
```

> ➡️ **পরবর্তী পড়ুন**: [memory-management.md](./memory-management.md) — প্রতিটি প্রসেস কীভাবে RAM ব্যবহার করে।
