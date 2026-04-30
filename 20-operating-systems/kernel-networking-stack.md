# 🌐 Linux Kernel Networking Stack

> `14-networking/`-এ আমরা TCP/UDP/HTTP দেখেছি — application-এর দৃষ্টিকোণ থেকে।
> এখানে আমরা **kernel-level**: NIC থেকে আসা একটি packet user-space socket-এ পৌঁছানো পর্যন্ত পুরো journey, এবং প্রতিটি ধাপ যেখানে আপনি tune/observe/inject করতে পারেন।

---

## 📖 সূচিপত্র

- [Packet path: NIC থেকে user space](#-packet-path-nic-থেকে-user-space)
- [NAPI ও softirq](#-napi-ও-softirq)
- [Netfilter/iptables/nftables hooks](#-netfilteriptablesnftables-hooks)
- [conntrack — connection tracking](#-conntrack--connection-tracking)
- [Linux Bridge, veth, network namespace](#-linux-bridge-veth-network-namespace)
- [Container networking (CNI overview)](#-container-networking-cni-overview)
- [eBPF / XDP — fast path](#-ebpf--xdp--fast-path)
- [TCP tuning sysctls](#-tcp-tuning-sysctls)
- [DPDK ও kernel bypass](#-dpdk-ও-kernel-bypass)
- [Tools: tcpdump, ss, ip, ethtool, conntrack](#-tools-tcpdump-ss-ip-ethtool-conntrack)
- [Real-world: 100k concurrent at Daraz CDN edge](#-real-world-100k-concurrent-at-daraz-cdn-edge)

---

## 📌 Packet path: NIC থেকে user space

### Receive (rx) path

```
   Network cable
        │
        ▼
┌────────────────────┐
│  NIC (hardware)    │   1. packet match RX descriptor ring
│                    │   2. DMA to RAM (zero CPU)
│                    │   3. raise hardware IRQ
└────────────────────┘
        │
        ▼
┌────────────────────┐
│  IRQ handler        │   4. hardirq — minimal work,
│  (top half)         │      schedule softirq (NET_RX_SOFTIRQ)
└────────────────────┘
        │
        ▼
┌────────────────────┐
│  ksoftirqd          │   5. NAPI poll — batch packet pull
│  (softirq context)  │      from ring → sk_buff
└────────────────────┘
        │
        ▼
┌────────────────────┐
│  Network stack      │   6. eth → IP → TCP/UDP
│                     │      netfilter hooks → conntrack
└────────────────────┘
        │
        ▼
┌────────────────────┐
│  socket buffer      │   7. matching socket-এ enqueue
│  (sk_receive_queue) │
└────────────────────┘
        │
        ▼
┌────────────────────┐
│  user space         │   8. recv()/epoll wakeup → copy
│  (PHP/Node/Nginx)   │      to userspace buffer
└────────────────────┘
```

### Transmit (tx) path

```
   user write()
        │
        ▼
   socket send buffer
        │
        ▼
   TCP segmentation, IP routing
        │
        ▼
   netfilter (OUTPUT, POSTROUTING)
        │
        ▼
   qdisc (queueing discipline) — fq_codel, pfifo_fast, cake
        │
        ▼
   driver tx ring → DMA → NIC → wire
```

> 💡 প্রতিটি ধাপে **per-packet cost** আছে। 10 Gbps line rate = 14.88 Mpps (small packet)। প্রতি packet-এ ~67ns budget — সাধারণ kernel stack তা handle করতে পারে না, তাই XDP/DPDK।

---

## 📌 NAPI ও softirq

পুরনো Linux প্রতিটি packet-এর জন্য একটি IRQ raise করত — IRQ storm-এ CPU 100% interrupt context-এ থাকত, real work হত না।

**NAPI (New API):** packet আসলে interrupt → schedule poll → কিছুক্ষণ pure poll mode-এ batch করে process। হাই rate-এ interrupt-free।

```bash
# NAPI usage / softirq stat
$ cat /proc/softirqs | head
                CPU0       CPU1       CPU2       CPU3
      HI:           0          0          0          0
   TIMER:    1234567    1023456     934512    1145678
  NET_TX:       12345      11234      10456      13567
  NET_RX:    9876543    9234567    8765432    9123456     ← per CPU rx
   BLOCK:       45678      43210      41234      44567
  ...

# কোন CPU IRQ handle করছে
$ cat /proc/interrupts | grep eth0
 24:  9876543  ...   IR-PCI-MSI 524288-edge  eth0-rx-0
 25:        0  9234567 ...                    eth0-rx-1
 ...
```

**RSS (Receive Side Scaling)** — NIC মাল্টি-queue, প্রতিটি queue আলাদা CPU IRQ-এ। 5-tuple hash দিয়ে packet split।

```bash
# Queue count
$ ethtool -l eth0
Pre-set maximums:
RX:        16
Combined:  16
Current hardware settings:
Combined:  8

# বাড়ান
$ ethtool -L eth0 combined 16

# IRQ affinity (CPU pinning)
$ cat /proc/irq/24/smp_affinity_list
0-1
$ echo 2-3 > /proc/irq/24/smp_affinity_list

# RPS (software RSS — যখন NIC single queue)
$ echo ffff > /sys/class/net/eth0/queues/rx-0/rps_cpus
```

> 🇧🇩 Daraz CDN edge node = 32-core, NIC 100Gbps, 32 RX queue + IRQ affinity per-NUMA = packet drop শূন্য।

---

## 📌 Netfilter/iptables/nftables hooks

Netfilter kernel-এ **hook point**, iptables/nftables হলো user-space tool যা সেই hook-এ rule সেট করে।

```
                     incoming packet
                            │
                            ▼
                    ┌───────────────────┐
                    │   PREROUTING      │  ← DNAT (port forward)
                    │   (raw, mangle,   │
                    │    nat, conntrack)│
                    └─────────┬─────────┘
                              │
                  routing decision (local? forward?)
                              │
                ┌─────────────┴─────────────┐
                │                           │
        local destination         non-local (forward)
                │                           │
                ▼                           ▼
       ┌──────────────┐           ┌──────────────┐
       │   INPUT      │           │   FORWARD    │
       │ (filter)     │           │ (filter)     │
       └──────┬───────┘           └──────┬───────┘
              │                          │
              ▼                          │
       local process                     │
              │                          │
              ▼                          ▼
       ┌──────────────┐           ┌──────────────┐
       │   OUTPUT     │           │  POSTROUTING │
       │ (filter,nat) │           │ (nat, mangle)│
       └──────┬───────┘           └──────┬───────┘
              │                          │
              ▼                          ▼  ← SNAT (masquerade)
       POSTROUTING               outgoing packet
              │
              ▼
       outgoing packet
```

### iptables — legacy (still common)

```bash
# Default policy
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

# SSH, HTTP, HTTPS
iptables -A INPUT -p tcp --dport 22 -m limit --limit 4/min -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# DNAT — port forward (Pathao internal LB-এ port 80 → 8080)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# SNAT / Masquerade — NAT gateway (cloud egress)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Save / restore
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

### nftables — modern replacement

```bash
nft list ruleset

nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule  inet filter input ct state established,related accept
nft add rule  inet filter input iif lo accept
nft add rule  inet filter input tcp dport {22, 80, 443} accept
```

| বৈশিষ্ট্য    | iptables             | nftables                       |
|--------------|----------------------|---------------------------------|
| Tables       | filter, nat, mangle, raw — আলাদা | unified                |
| Performance  | rule by rule         | bytecode VM                     |
| IPv4/IPv6    | `iptables` + `ip6tables` | একই tool                  |
| Set          | ipset external        | built-in                        |

### Common hooks for containers

```bash
# Docker creates these
$ iptables -t nat -L DOCKER -n
Chain DOCKER (2 references)
target     prot opt source destination
RETURN     all  --  0.0.0.0/0 0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0 0.0.0.0/0 tcp dpt:80 to:172.17.0.2:80
```

> ⚠️ K8s `kube-proxy` iptables/IPVS দিয়ে service IP → pod IP DNAT করে। 5000 service-এ iptables rule 50000+ — performance issue। তাই Pathao `ipvs` বা **Cilium eBPF** mode-এ K8s চালায়।

---

## 📌 conntrack — connection tracking

netfilter একটি **stateful** firewall রাখে — প্রতিটি active flow track।

```bash
$ conntrack -L | head
tcp 6 431999 ESTABLISHED src=10.0.0.5 dst=8.8.4.4 sport=54321 dport=443 \
    src=8.8.4.4 dst=10.0.0.5 sport=443 dport=54321 ASSURED mark=0 use=1

$ cat /proc/sys/net/netfilter/nf_conntrack_count
12345
$ cat /proc/sys/net/netfilter/nf_conntrack_max
65536          ← ⚠️ NAT gateway, busy server-এ আগে শেষ হয়
```

### Conntrack table full → connection drop!

```bash
$ dmesg | grep conntrack
nf_conntrack: table full, dropping packet

# fix — বাড়ান
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_buckets=262144

# Persistent
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.d/99-prod.conf
```

> 🇧🇩 Daraz NAT gateway একবার Black Friday-তে conntrack overflow — সব new connection drop। `nf_conntrack_max` 256k → 2M করার পর fix।

---

## 📌 Linux Bridge, veth, network namespace

Container networking-এর primitive:

### veth (virtual ethernet pair)

```
   namespace A             namespace B
   ┌──────────┐             ┌──────────┐
   │  vethA   │ ◄────────►  │  vethB   │
   │  IP:1.1  │  pipe       │  IP:1.2  │
   └──────────┘             └──────────┘
```

```bash
# pair create
ip link add veth0 type veth peer name veth1

# একটিকে namespace-এ move
ip link set veth1 netns ns-pathao
ip netns exec ns-pathao ip addr add 10.0.0.2/24 dev veth1
ip netns exec ns-pathao ip link set veth1 up
```

### Linux Bridge

L2 software switch — multiple veth-কে একই subnet-এ যোগ করে।

```bash
ip link add cni0 type bridge
ip link set cni0 up
ip addr add 10.244.0.1/16 dev cni0

# veth0 (host side) bridge-এ
ip link set veth0 master cni0
ip link set veth0 up
```

### Pod networking (K8s flannel/bridge)

```
   Pod A (ns)          Pod B (ns)          Pod C (ns)
   eth0:10.244.0.5    eth0:10.244.0.6     eth0:10.244.0.7
       │                  │                  │
       │ veth pair         │ veth pair        │ veth pair
       ▼                  ▼                  ▼
   ┌──────────────── cni0 bridge ─────────────────┐
   │                10.244.0.1                    │
   └──────────────────────────────────────────────┘
                          │
                          ▼  routing
                   eth0 (host NIC)
                          │
                          ▼  VXLAN encapsulation (cross-node)
                   underlay network
```

```bash
# Pod-এর net ns inspect
PID=$(crictl inspect <id> | jq .info.pid)
nsenter -t $PID -n ip addr
nsenter -t $PID -n ip route
nsenter -t $PID -n ss -tln
```

---

## 📌 Container networking (CNI overview)

Container Network Interface — K8s-এর pluggable network spec।

| Plugin     | Backend                                | Notes                          |
|------------|----------------------------------------|---------------------------------|
| flannel    | VXLAN / host-gw                        | simple                         |
| Calico     | BGP / VXLAN / eBPF                     | network policy strong          |
| Cilium     | eBPF (no iptables!)                   | observability, performance     |
| Weave      | encrypted mesh                         | older                          |
| AWS VPC CNI| ENI per pod                            | AWS native                     |

> 🇧🇩 Pathao K8s = Cilium (eBPF) — kube-proxy replace, network policy direct cgroup attach। iptables overhead নেই।

---

## 📌 eBPF / XDP — fast path

### XDP (eXpress Data Path)

eBPF program **NIC driver-এ আগে** run হয় — kernel stack হিট করার আগেই packet drop/redirect/forward। Fastest possible packet processing in Linux।

```
   NIC → driver → ┌─────────────────────────┐
                  │  XDP eBPF program        │  ← এখানে decision
                  └─────────────────────────┘
                         │
            ┌────────────┼────────────┬──────────┐
            ▼            ▼            ▼          ▼
        XDP_PASS    XDP_DROP    XDP_TX     XDP_REDIRECT
        (kernel    (instant     (echo back)(other NIC/CPU)
         stack)     drop)
```

**Use cases:**

- DDoS mitigation — SYN flood drop in driver (Cloudflare)
- L4 load balancer — Katran (Facebook), Cilium
- Packet filter — like iptables but 10x faster

```c
// xdp_drop_icmp.c — সরল XDP example
SEC("xdp")
int drop_icmp(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *end  = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;
    if ((void*)(eth + 1) > end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;
    struct iphdr *ip = (void*)(eth + 1);
    if ((void*)(ip + 1) > end) return XDP_PASS;
    if (ip->protocol == IPPROTO_ICMP) return XDP_DROP;
    return XDP_PASS;
}
```

```bash
# Load
$ ip link set dev eth0 xdpgeneric obj xdp_drop_icmp.o sec xdp
$ ping <ip>     # blocked

# Unload
$ ip link set dev eth0 xdpgeneric off
```

### tc (traffic control) eBPF

XDP-এর পরে, কিন্তু network stack-এর আগে — tc ingress/egress hook-এ eBPF।

```bash
# tc filter দিয়ে eBPF
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj filter.o sec ingress
```

### Cilium / Katran প্রসঙ্গে

- **Cilium** K8s CNI — pod-to-pod কোনো iptables/conntrack নেই, eBPF lookup direct। 10x service routing performance।
- **Katran** Facebook L4 LB — XDP-based, single 100Gbps server millions pps handle।

---

## 📌 TCP tuning sysctls

প্রোডাকশনে এই sysctls প্রায়ই tune করতে হয়:

```bash
# /etc/sysctl.d/99-network-prod.conf

# ── Connection backlog ────────────────────────────
net.core.somaxconn = 65535                     # listen() backlog cap
net.ipv4.tcp_max_syn_backlog = 8192            # half-open SYN queue

# ── Buffers ───────────────────────────────────────
net.core.rmem_max = 16777216                   # max socket recv buf
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216        # min default max
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.netdev_max_backlog = 30000            # NIC → softirq queue

# ── Congestion control ────────────────────────────
net.ipv4.tcp_congestion_control = bbr          # বা cubic (default)
net.core.default_qdisc = fq                     # BBR-এর জন্য fq

# ── Fast Open ─────────────────────────────────────
net.ipv4.tcp_fastopen = 3                       # client+server enable

# ── TIME-WAIT / connection reuse ──────────────────
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.ip_local_port_range = 1024 65535      # ephemeral port range
net.ipv4.tcp_max_tw_buckets = 262144

# ── SYN cookies (DDoS protection) ─────────────────
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 3

# ── Keepalive (idle conn detection) ───────────────
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 3

# ── conntrack (NAT gateway) ───────────────────────
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 600

# ── Reverse path filter ───────────────────────────
net.ipv4.conf.all.rp_filter = 1

# ── File descriptors ──────────────────────────────
fs.file-max = 2097152

# Apply
$ sudo sysctl --system
```

### somaxconn সমস্যা

Nginx `listen 80 backlog=65535;` — কিন্তু kernel `somaxconn=128` (পুরনো default) মানলে আসলে 128 cap। তাই sysctl-এ মিলাতে হবে।

```bash
# verify
$ ss -lnt | grep ':80 '
LISTEN 0 65535 0.0.0.0:80 ...
                ↑ Recv-Q max
```

### BBR — modern congestion control

পুরনো `cubic` packet loss-কে congestion signal ধরে। BBR (Google) RTT + bandwidth probe — high-bandwidth + lossy লিঙ্কে অনেক ভালো।

```bash
$ sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic bbr

$ sysctl -w net.ipv4.tcp_congestion_control=bbr
$ sysctl -w net.core.default_qdisc=fq

# verify
$ ss -tin | grep bbr
bbr wscale:7,7 rto:204 rtt:3.5/1.2 mss:1448 ...
```

> 🇧🇩 BD ↔ Mumbai/Singapore latency 30–80ms, occasional packet loss — BBR enable করে bKash API egress throughput ৩০-৪০% বেড়েছে।

### TCP Fast Open

3-way handshake-এর সাথে data piggyback — RTT save।

```bash
sysctl -w net.ipv4.tcp_fastopen=3
```

---

## 📌 DPDK ও kernel bypass

কখনো kernel stack-ই too slow। **DPDK** (Data Plane Development Kit) — userspace driver, kernel bypass, poll mode।

```
   Standard:                     DPDK:
   NIC → kernel → user           NIC → user (direct)
   (syscall overhead)            (poll loop, no IRQ)
```

- 10–100 Mpps possible single core
- Use case: HFT, telco, software router (VPP)
- Cost: dedicated CPU core, no kernel features (firewall, conntrack)

**Comparison:**

| Tech     | Overhead    | Features          | Use case          |
|----------|-------------|-------------------|--------------------|
| Standard | high (~5 Mpps/core) | full kernel  | normal apps        |
| AF_XDP   | medium       | partial bypass + zero-copy | balance |
| XDP      | low          | early hook        | DDoS, LB          |
| DPDK     | very low    | almost none       | HFT, telco        |

> 💡 99% production apps DPDK দরকার নেই। eBPF/XDP সাধারণত যথেষ্ট।

---

## 📌 Tools: tcpdump, ss, ip, ethtool, conntrack

```bash
# ── ip — modern replacement of ifconfig/route ──────
ip a                            # interface + IP
ip link                         # L2 info
ip -s link show eth0           # stats
ip route                        # routing table
ip rule                         # policy routing
ip neigh                        # ARP table
ip netns                        # namespaces

# ── ss — modern replacement of netstat ─────────────
ss -tlnp                        # TCP listening + process
ss -tnp state established       # established conns
ss -s                           # summary
ss -i                           # tcp_info (rtt, cwnd, bbr state)
ss -tn dst :443 | head          # outbound connections to 443

# ── tcpdump — packet capture ────────────────────────
tcpdump -i eth0 -nn 'tcp port 80'
tcpdump -i any -w out.pcap 'host 10.0.0.5 and port 443'
tcpdump -nni eth0 'icmp' -c 10
tcpdump -i lo -A 'tcp port 8080'           # ASCII payload

# ── ethtool — NIC info & tune ───────────────────────
ethtool eth0                              # speed, duplex
ethtool -S eth0 | grep -i 'drop\|err'     # stats — drops!
ethtool -g eth0                            # ring buffer size
ethtool -G eth0 rx 4096 tx 4096           # increase ring
ethtool -k eth0                            # offload features
ethtool -K eth0 gro on lro off             # toggle

# ── conntrack ───────────────────────────────────────
conntrack -L
conntrack -S                               # stats
conntrack -E                               # event stream

# ── nstat (counter) ─────────────────────────────────
nstat -a | grep -i retrans
nstat -a TcpExtTCPLossProbes

# ── Traffic stats ──────────────────────────────────
sar -n DEV 1                               # bytes/pkt per interface
iftop -i eth0                              # top bandwidth talkers
nload eth0

# ── eBPF tools ─────────────────────────────────────
tcptrace-bpfcc                             # connection summary
tcpconnect-bpfcc                           # new connect
tcpaccept-bpfcc
tcpretrans-bpfcc                           # retransmits — packet loss
tcplife-bpfcc                              # connection lifecycle
sockstat-bpfcc

# ── Diagnostic full session ────────────────────────
mtr -r -c 100 cdn.daraz.com.bd            # path latency + loss
```

---

## 📌 Real-world: 100k concurrent at Daraz CDN edge

Scenario: Daraz Black Friday — single edge node 100k concurrent HTTPS connections, 10 Gbps line।

### Step 1: NIC ও IRQ tuning

```bash
# Multi-queue
ethtool -L eth0 combined 16

# IRQ affinity per NUMA node
for i in $(seq 24 39); do
    echo "$((i-24))" > /proc/irq/$i/smp_affinity_list
done

# Larger ring
ethtool -G eth0 rx 4096 tx 4096

# Offload
ethtool -K eth0 gro on tso on gso on
```

### Step 2: Sysctl

```bash
cat > /etc/sysctl.d/99-edge.conf <<EOF
fs.file-max = 4194304

net.core.somaxconn = 65535
net.core.netdev_max_backlog = 100000
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.core.default_qdisc = fq

net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
net.ipv4.tcp_notsent_lowat = 131072

net.netfilter.nf_conntrack_max = 4194304
net.netfilter.nf_conntrack_buckets = 1048576
EOF
sysctl --system
```

### Step 3: Process limits

```bash
# /etc/security/limits.d/nginx.conf
nginx soft nofile 1048576
nginx hard nofile 1048576

# systemd unit
[Service]
LimitNOFILE=1048576
```

### Step 4: Nginx tuning

```nginx
worker_processes auto;
worker_rlimit_nofile 1048576;

events {
    worker_connections 65536;
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 65;
    keepalive_requests 1000;

    open_file_cache max=200000 inactive=20s;

    # SSL session cache (handshake-এর CPU cost কমায়)
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
}
```

### Step 5: XDP DDoS protect

```bash
# একটি simple XDP rate limit / SYN flood drop
xdp-loader load eth0 ddos-shield.o
```

### Step 6: Observability

```bash
# Real-time
watch -n1 "ss -s; ethtool -S eth0 | grep -i drop"

# eBPF
tcpretrans-bpfcc &     # retransmit ratio
tcplife-bpfcc &        # connection duration distribution
profile-bpfcc -F 99 -p $(pgrep nginx) 30   # CPU profile
```

### Result

- 110k concurrent stable
- p99 latency 15ms (LAN client)
- packet drop < 0.001%
- CPU 60% (16 core), headroom থাকে

---

## 🎯 সংক্ষেপে

- Packet path: NIC → IRQ → softirq/NAPI → IP/TCP → socket → user
- NAPI + RSS + IRQ affinity = high pps without IRQ storm
- Netfilter hooks (PRE/IN/FWD/OUT/POST) — iptables/nftables দিয়ে rule
- conntrack table fill = NAT gateway-এ classic outage; `nf_conntrack_max` বাড়ান
- veth + bridge + netns = container networking-এর primitive; CNI orchestrate করে
- eBPF/XDP fast path — Cilium, Katran, DDoS shield
- TCP sysctl: `somaxconn`, `tcp_max_syn_backlog`, `rmem/wmem`, `bbr`, `fastopen`
- Tool: `ip`, `ss`, `tcpdump`, `ethtool`, `conntrack`, `mtr`, `tcpretrans-bpfcc`
- DPDK = ultimate kernel bypass, কিন্তু production app-এ rarely দরকার

> 🇧🇩 **Daraz CDN edge:** RSS 16 queue + BBR + TCP Fast Open + conntrack 4M + Nginx 1M fd = single node 100k concurrent।  
> bKash API gateway: nftables + SYN cookies + rate-limit per-IP (XDP) = DDoS resilient।  
> Pathao K8s: Cilium eBPF (no iptables) = kube-proxy overhead nil at 5000+ services।
