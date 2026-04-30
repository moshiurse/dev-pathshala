# 📦 Namespaces ও cgroups — কন্টেইনারের ভিত্তি

> "Container" আসলে কী? কোনো magic না — শুধু Linux kernel-এর **namespace** + **cgroup** + **capability** + **seccomp** + **pivot_root**।
> Docker/runc/containerd এসবই orchestrate করে। Daraz, Pathao প্রোডাকশনে যা চলে — সেটি বুঝতে এই দুটি concept master করতে হবে।

---

## 📖 সূচিপত্র

- [Namespace ও cgroup-এর ভূমিকা](#-namespace-ও-cgroup-এর-ভূমিকা)
- [৮টি namespace বিস্তারিত](#-৮টি-namespace-বিস্তারিত)
- [unshare, nsenter, lsns ব্যবহার](#-unshare-nsenter-lsns-ব্যবহার)
- [cgroup v1 vs v2](#-cgroup-v1-vs-v2)
- [cgroup controllers (cpu/memory/io/pids/devices)](#-cgroup-controllers-cpumemoryiopidsdevices)
- [Docker/runc কন্টেইনার তৈরি কীভাবে](#-dockerrunc-কন্টেইনার-তৈরি-কীভাবে)
- [Capabilities](#-capabilities)
- [seccomp ও LSM (AppArmor/SELinux)](#-seccomp-ও-lsm-apparmorselinux)
- [Rootless container](#-rootless-container)
- [K8s resource limits → cgroup](#-k8s-resource-limits--cgroup)
- [Daraz/Pathao production tuning](#-darazpathao-production-tuning)

---

## 📌 Namespace ও cgroup-এর ভূমিকা

| Concept     | প্রশ্নের উত্তর                                           |
|-------------|----------------------------------------------------------|
| **Namespace** | "এই process কী **দেখতে পাবে**?" (isolation)            |
| **cgroup**    | "এই process কতটুকু **ব্যবহার করতে পারবে**?" (limit)    |

```
┌──────────────────────────────────────────────────────────────┐
│                     Linux kernel (single)                    │
├──────────────────────────────────────────────────────────────┤
│  Container A          Container B          Container C       │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐     │
│  │ ns set:  │         │ ns set:  │         │ ns set:  │     │
│  │  PID #1  │         │  PID #2  │         │  PID #3  │     │
│  │  NET     │         │  NET     │         │  NET     │     │
│  │  MNT     │         │  MNT     │         │  MNT     │     │
│  │  USER    │         │  USER    │         │  USER    │     │
│  └──────────┘         └──────────┘         └──────────┘     │
│                                                              │
│  cgroup limits: cpu=2, mem=2G    cpu=4, mem=8G    cpu=1,...│
└──────────────────────────────────────────────────────────────┘
```

> 💡 কন্টেইনার **VM নয়** — কোনো guest kernel নেই। সব host kernel-ই। শুধু "view" আলাদা।

---

## 📌 ৮টি namespace বিস্তারিত

| Namespace    | আলাদা করে                                       | syscall flag         | Kernel | Year |
|--------------|--------------------------------------------------|----------------------|--------|------|
| **MNT**      | mount points, ফাইলসিস্টেম view                  | `CLONE_NEWNS`        | 2.4.19 | 2002 |
| **UTS**      | hostname, domainname                            | `CLONE_NEWUTS`       | 2.6.19 | 2006 |
| **IPC**      | System V IPC, POSIX mqueue                      | `CLONE_NEWIPC`       | 2.6.19 | 2006 |
| **PID**      | process ID number space                         | `CLONE_NEWPID`       | 2.6.24 | 2008 |
| **NET**      | network device, IP, port, route, firewall      | `CLONE_NEWNET`       | 2.6.29 | 2009 |
| **USER**     | UID/GID mapping                                 | `CLONE_NEWUSER`      | 3.8    | 2013 |
| **CGROUP**   | cgroup root view                                | `CLONE_NEWCGROUP`    | 4.6    | 2016 |
| **TIME**     | `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME` offset      | `CLONE_NEWTIME`      | 5.6    | 2020 |

```bash
# Process-এর namespace দেখুন
$ ls -la /proc/self/ns/
lrwxrwxrwx 1 root root cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root ipc    -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root mnt    -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root net    -> 'net:[4026531992]'
lrwxrwxrwx 1 root root pid    -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root time   -> 'time:[4026531834]'
lrwxrwxrwx 1 root root user   -> 'user:[4026531837]'
lrwxrwxrwx 1 root root uts    -> 'uts:[4026531838]'

# একই inode = একই namespace
```

### MNT — mount isolation

কন্টেইনার দেখে শুধু **নিজের rootfs**। `pivot_root` দিয়ে real / replace করা হয়।

```bash
$ unshare -m bash
# inside: এই shell-এর mount table আলাদা
$ mount -t tmpfs none /mnt
$ ls /mnt    # শুধু এই shell-এ দেখা যাবে
```

### UTS — hostname

```bash
$ unshare -u bash
# hostname secret-app
$ hostname
secret-app
$ exit
$ hostname    # আগের ফিরে এলো
```

### PID — process tree

কন্টেইনার-এর `ps -ef`-এ host process দেখা যায় না। ভেতরের প্রথম process **PID 1**।

```
   host-side view:               container view:
   PID 5012 (containerd-shim)    
   PID 5040 (PID 1 in ns) ─►     PID 1 (php-fpm)
   PID 5041                      PID 12 (php-fpm child)
```

```bash
$ unshare --fork --pid --mount-proc bash
# echo $$
1
# ps aux
PID  USER  COMMAND
1    root  bash
8    root  ps aux
```

> ⚠️ PID 1-এর special semantics: zombie reaper, signal default mask — **PID 1 problem** এজন্যই।

### NET — full network stack

প্রতিটি net ns-এর নিজস্ব loopback, route table, iptables rule, port space।

```bash
$ ip netns add ns-pathao
$ ip netns exec ns-pathao ip link
1: lo: <LOOPBACK> ...   # শুধু lo, down state
$ ip netns exec ns-pathao ip addr add 127.0.0.1/8 dev lo
$ ip netns exec ns-pathao ip link set lo up
$ ip netns exec ns-pathao ss -tln    # নিজস্ব socket table
```

### USER — UID translation

কন্টেইনার-এ "root" (UID 0) host-এ unprivileged user (e.g. UID 100000) হতে পারে।

```
   container view:        host view:
   UID 0 (root) ──────►   UID 100000  (mapping)
   UID 1 (daemon)─────►   UID 100001
```

```bash
$ unshare --user --map-root-user bash
# whoami
root
# id
uid=0(root) gid=0(root)
# (host-এ আমি unprivileged user)
```

> 💡 USER ns + rootless docker = container "root" host-এ হার্মলেস — ব্রেকআউট হলেও nothing।

### CGROUP — cgroup root view

পুরনো versions-এ কন্টেইনার host-এর cgroup tree দেখতে পেত। Cgroup ns দিয়ে শুধু নিজের subtree show।

### TIME — monotonic clock offset

কন্টেইনার `CLOCK_BOOTTIME` reset করতে পারে — checkpoint/restore (CRIU)-এ দরকার।

---

## 📌 unshare, nsenter, lsns ব্যবহার

```bash
# ১. একটি namespace-এ shell run
$ unshare --fork --pid --mount-proc --uts --ipc --net bash
# inside isolated environment

# ২. existing process-এর namespace-এ ঢুকুন
$ pgrep -f nginx
1234
$ nsenter -t 1234 --net --pid --mount bash
# nginx-এর container-এ debugging shell

# ৩. সব namespace list
$ lsns
        NS TYPE   NPROCS   PID USER   COMMAND
4026531834 time      450     1 root   /sbin/init
4026531835 cgroup    450     1 root   /sbin/init
4026531836 pid       450     1 root   /sbin/init
4026532183 mnt         3  9082 root   nginx: master
4026532184 net         3  9082 root   nginx: master
...

# ৪. কন্টেইনার-এ কী চলছে দেখুন
$ docker ps
CONTAINER ID   COMMAND
abc123def      "/docker-entrypoint…"

$ docker top abc123def
$ docker inspect -f '{{.State.Pid}}' abc123def
9082

$ nsenter -t 9082 --all
# কন্টেইনার-এর exact view
```

---

## 📌 cgroup v1 vs v2

cgroup = "control group" — process-গুলোকে hierarchically গ্রুপ করে resource limit/account।

```
v1 (legacy):                          v2 (unified):
                                      
 /sys/fs/cgroup/                       /sys/fs/cgroup/
 ├── cpu/                              ├── docker-abc.scope/
 │   └── docker-abc/                   │   ├── cpu.max
 │       └── cpu.cfs_quota_us          │   ├── memory.max
 ├── memory/                           │   ├── io.max
 │   └── docker-abc/                   │   └── pids.max
 │       └── memory.limit_in_bytes     ├── docker-xyz.scope/
 ├── blkio/                            └── ...
 │   └── docker-abc/
 ...                                   single hierarchy, all controllers
 separate hierarchy per controller
```

### তুলনা

| বৈশিষ্ট্য             | cgroup v1                           | cgroup v2                           |
|----------------------|-------------------------------------|--------------------------------------|
| Hierarchy            | প্রতিটি controller আলাদা            | unified single tree                  |
| Controller name      | `cpu`, `memory`, `blkio`, ...      | `cpu`, `memory`, `io`, `pids`, ...  |
| Memory + swap        | `memory.memsw.limit_in_bytes`      | `memory.swap.max`                    |
| I/O                  | `blkio` (legacy), buggy             | `io` (proper write-back accounting) |
| PSI (Pressure Stall) | ❌                                  | ✅ `cpu.pressure`, `memory.pressure`|
| Default in           | RHEL 7, Ubuntu 20.04                | RHEL 9, Ubuntu 22.04+, Fedora 31+   |

```bash
# কোনটি active?
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs              # v2

$ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (...)

# Docker container-এর cgroup
$ cat /proc/$(docker inspect -f '{{.State.Pid}}' web)/cgroup
0::/system.slice/docker-abc123.scope
```

---

## 📌 cgroup controllers (cpu/memory/io/pids/devices)

### CPU controller

```bash
# cgroup v2 — একটি cgroup তৈরি করে CPU limit
mkdir /sys/fs/cgroup/myapp
echo "+cpu" > /sys/fs/cgroup/cgroup.subtree_control

# 200ms প্রতি 1000ms = 0.2 CPU
echo "200000 1000000" > /sys/fs/cgroup/myapp/cpu.max

# কোন process এই গ্রুপে?
echo $$ > /sys/fs/cgroup/myapp/cgroup.procs
# এখন এই shell + child সব 0.2 CPU bound

# CPU stat
$ cat /sys/fs/cgroup/myapp/cpu.stat
usage_usec 1234567
user_usec 800000
system_usec 434567
nr_periods 50
nr_throttled 5
throttled_usec 12345    # ⚠️ throttled হলে latency spike
```

### Memory controller

```bash
echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control
echo $((512*1024*1024)) > /sys/fs/cgroup/myapp/memory.max     # 512 MB
echo $((100*1024*1024)) > /sys/fs/cgroup/myapp/memory.high    # soft

# limit ছাড়ালে → OOM kill (memory.max), or throttle (memory.high)
$ cat /sys/fs/cgroup/myapp/memory.events
low 0
high 12         # 12 বার throttle হয়েছে
max 0
oom 0
oom_kill 0
```

### I/O controller

```bash
# 100 MB/s read, 50 MB/s write on /dev/vda (8:0)
echo "8:0 rbps=104857600 wbps=52428800" > /sys/fs/cgroup/myapp/io.max
```

### pids controller — fork bomb defense

```bash
echo 100 > /sys/fs/cgroup/myapp/pids.max
# এই গ্রুপে max 100 thread/process
```

### devices

```bash
# ডিফল্ট deny, শুধু /dev/null, /dev/zero allow
# (cgroup v2-এ eBPF program দিয়ে handle হয়)
```

---

## 📌 Docker/runc কন্টেইনার তৈরি কীভাবে

`docker run` এর পেছনে কী ঘটে:

```
docker CLI ──► dockerd (daemon) ──► containerd ──► containerd-shim ──► runc
                                                                          │
                                                                          ▼
                                            ┌─────────────────────────────────┐
                                            │ ১. namespace তৈরি (clone flags)  │
                                            │ ২. cgroup তৈরি ও limit বসানো    │
                                            │ ৩. mount rootfs (overlay)        │
                                            │ ৪. pivot_root করে / সরানো        │
                                            │ ৫. capability drop              │
                                            │ ৬. seccomp profile load         │
                                            │ ৭. AppArmor profile attach      │
                                            │ ৮. UID/GID map                  │
                                            │ ৯. exec কন্টেইনার entrypoint     │
                                            └─────────────────────────────────┘
```

### সরাসরি unshare দিয়ে "DIY container"

```bash
# একটি rootfs prepare
mkdir -p /tmp/rootfs
debootstrap --variant=minbase bookworm /tmp/rootfs

# কন্টেইনার চালু
unshare --pid --mount --net --uts --ipc --user --map-root-user --fork \
        chroot /tmp/rootfs /bin/bash

# inside:
mount -t proc none /proc
hostname my-container
ps aux       # শুধু নিজের process দেখা যাবে
```

এটি runc-এর সরলরূপ। runc OCI spec (`config.json`) পড়ে — সব flags, cgroup limit, capability, seccomp profile একসাথে apply।

```bash
# OCI bundle তৈরি
mkdir -p /tmp/bundle/rootfs
docker export $(docker create alpine) | tar -x -C /tmp/bundle/rootfs
cd /tmp/bundle && runc spec
runc run mycontainer
```

---

## 📌 Capabilities

ক্লাসিক UNIX-এ root (UID 0) সব করতে পারে। Linux **capabilities** root-এর ক্ষমতাকে ৪০+ ভাগে বিভক্ত করেছে। কন্টেইনার-এ unnecessary capability drop = security hardening।

| Capability             | কী করে                                      |
|------------------------|---------------------------------------------|
| `CAP_NET_ADMIN`        | network config (iptables, route)            |
| `CAP_NET_BIND_SERVICE` | <1024 port bind                             |
| `CAP_SYS_ADMIN`        | mount, namespace ইত্যাদি — "new root"      |
| `CAP_SYS_PTRACE`       | strace, gdb attach                          |
| `CAP_CHOWN`            | অন্যের ফাইলের owner change                  |
| `CAP_DAC_OVERRIDE`     | file permission ignore                      |
| `CAP_KILL`             | অন্য user-এর process kill                   |
| `CAP_SETUID/SETGID`    | UID/GID change                              |
| `CAP_SYS_MODULE`       | kernel module load                          |
| `CAP_BPF`              | BPF program load                            |

```bash
# একটি process-এর capabilities দেখুন
$ getpcaps 1234
1234: cap_net_bind_service+ep

# binary-তে file capability set
$ setcap cap_net_bind_service+ep /usr/local/bin/node

# Docker container default 14টি capability পায়, drop:
$ docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

> 🇧🇩 **Pathao security baseline:** সব production container `--cap-drop=ALL` দিয়ে চালু হয়। শুধু যা একদম দরকার সেগুলো `--cap-add` করা হয়।

---

## 📌 seccomp ও LSM (AppArmor/SELinux)

### seccomp — syscall filter (BPF)

কন্টেইনার-এ ৩০০+ syscall-এর মধ্যে অনেকগুলো অপ্রয়োজনীয়। seccomp filter দিয়ে block করুন।

```bash
# Docker default profile blocks ~44 syscalls (e.g., mount, kexec_load, reboot)
$ docker run --security-opt seccomp=/path/to/profile.json alpine

# strict no-new-privs
$ docker run --security-opt no-new-privileges:true alpine
```

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {"names": ["read","write","open","close","stat","mmap","exit_group"],
     "action": "SCMP_ACT_ALLOW"}
  ]
}
```

### AppArmor / SELinux — Mandatory Access Control

LSM (Linux Security Module) — kernel hook-এ extra check। AppArmor (Ubuntu, Debian) path-based; SELinux (RHEL, Fedora) label-based।

```bash
# AppArmor — Docker container-এর profile
$ aa-status | grep docker
$ docker run --security-opt apparmor=docker-default nginx

# SELinux — RHEL container
$ docker run --security-opt label=type:container_t nginx
```

---

## 📌 Rootless container

Docker daemon root হিসেবে চলে — দীর্ঘকালীন security হেডেক। **rootless** mode-এ daemon ও container সব unprivileged user হিসেবে চলে (USER ns-এর সাহায্যে)।

```bash
# Docker rootless
dockerd-rootless-setuptool.sh install
docker context use rootless

# Podman — daemon-less, rootless ডিফল্ট
podman run -d --name web nginx

$ ps aux | grep nginx
rakib   12345  ...   # not root!
```

> 💡 **DigitalOcean BD droplet** যেখানে multiple developer একই VM ভাগ করে — rootless Podman ideal।

---

## 📌 K8s resource limits → cgroup

```yaml
# pathao-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api
        image: pathao/api:1.2
        resources:
          requests:
            cpu: 500m         # 0.5 CPU
            memory: 512Mi
          limits:
            cpu: 1000m        # 1 CPU
            memory: 1Gi
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]
          seccompProfile:
            type: RuntimeDefault
```

**যা আসলে হয়:**

```
requests.cpu=500m  ──► cgroup cpu.weight (relative share)
limits.cpu=1000m   ──► cgroup cpu.max = "100000 100000"   (1 CPU)
limits.memory=1Gi  ──► cgroup memory.max = 1073741824
```

```bash
# pod-এর cgroup দেখুন
$ kubectl exec -it api-pod -- cat /sys/fs/cgroup/cpu.max
100000 100000

$ kubectl exec -it api-pod -- cat /sys/fs/cgroup/memory.max
1073741824
```

### CPU throttling — silent killer

```bash
$ cat /sys/fs/cgroup/cpu.stat
nr_throttled 1234
throttled_usec 5678901

# nr_throttled > 0 মানে container CPU limit-এ hit করেছে — latency spike
```

> ⚠️ Pathao একটি incident: API p99 latency 500ms → root cause CPU limit (`200m`) — Node.js GC-এর সময় throttle হয়ে গিয়েছিল। limit `1000m` করার পর fix।

---

## 📌 Daraz/Pathao production tuning

### 1. Resource right-sizing — VPA + PSI

cgroup v2-এর **PSI** (Pressure Stall Information) ব্যবহার করে resource starvation detect:

```bash
$ cat /sys/fs/cgroup/system.slice/docker-abc.scope/cpu.pressure
some avg10=12.3 avg60=8.1 avg300=4.2 total=12345678
full avg10=2.1  avg60=1.0 avg300=0.5 total=234567

# avg10 > 10% means container CPU starved last 10s
```

### 2. Memory: `memory.high` vs `memory.max`

- `memory.max` hard limit — ছাড়ালে **OOM kill**
- `memory.high` soft limit — throttle, but no kill

K8s Quality of Service (QoS):

| QoS class    | `requests` | `limits`    | OOM order        |
|--------------|------------|-------------|------------------|
| Guaranteed   | =limits    | set         | last to die      |
| Burstable    | <limits    | set         | medium           |
| BestEffort   | not set    | not set     | first to die     |

> 💡 bKash payment gateway — Guaranteed QoS (request=limit), যাতে kernel pressure-এ কখনো OOM kill না হয়।

### 3. Network namespace + veth — Pod networking

```
Pod (namespace ns-pod-1)             Host (root ns)
  ┌──────────────────┐                ┌──────────────────┐
  │  eth0            │ ◄── veth ───►  │  vethXXXX        │
  │  10.244.1.5      │   pair          │                  │
  └──────────────────┘                 │  cni0 bridge      │
                                       │  10.244.1.1       │
                                       └──────────────────┘
```

```bash
# veth pair দেখুন
$ ip link show type veth
$ ip netns list

# pod-এর net ns-এ ঢুকুন
PID=$(crictl inspect <id> | jq .info.pid)
nsenter -t $PID -n ip addr
nsenter -t $PID -n ss -tln
```

### 4. cgroup tuning দিয়ে noisy-neighbor fix

```bash
# bKash node-এ একটা batch job database neighbour-কে কষ্ট দিচ্ছিল
# fix: I/O weight কম করা
echo "8:0 wbps=52428800" > /sys/fs/cgroup/batch.scope/io.max
```

### 5. Pid limit — fork bomb / leaked threads

```yaml
# K8s
apiVersion: v1
kind: LimitRange
spec:
  limits:
  - type: Container
    default:
      pids: 4096
```

Daraz Java microservice একবার thread leak করেছিল — 30k thread, host-এ fork() failure। `pids.max=4096` set করার পর সমস্যা contained।

---

## 🎯 সংক্ষেপে

- **Container = namespace + cgroup + capability + seccomp + AppArmor/SELinux + pivot_root** — কোনো magic নয়।
- ৮টি namespace: PID, MNT, NET, UTS, IPC, USER, CGROUP, TIME — প্রতিটি **isolation**।
- cgroup v2 unified hierarchy — controllers: cpu, memory, io, pids, devices, hugetlb।
- `unshare`/`nsenter`/`lsns`/`crictl` দিয়ে কন্টেইনার debug করুন।
- Capability drop ALL, add শুধু যা লাগে — security baseline।
- K8s resource limit → cgroup limit এর direct mapping। CPU throttling latency-এর প্রধান source।
- Rootless container (Podman) shared infra-তে best practice।

> 🇧🇩 **Pathao stack:** cgroup v2 + PSI monitoring + Guaranteed QoS payment service + rootless dev VM = প্রোডাকশন-গ্রেড isolation।  
> Daraz CI = pids.max + memory.high tuned, যাতে runaway build host-কে কাবু না করে।  
> bKash core API = `--cap-drop=ALL --read-only --no-new-privileges` baseline।
