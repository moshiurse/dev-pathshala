# 🔌 Boot Sequence ও Init System

> পাওয়ার বাটন চাপার পর shell prompt আসা পর্যন্ত কী ঘটে?
> BIOS/UEFI → bootloader → kernel → initramfs → init (PID 1) → services। প্রতিটি ধাপ deeply জানলে boot failure debug সহজ।

---

## 📖 সূচিপত্র

- [Boot sequence ধাপে ধাপে](#-boot-sequence-ধাপে-ধাপে)
- [BIOS vs UEFI](#-bios-vs-uefi)
- [Bootloader (GRUB)](#-bootloader-grub)
- [Kernel + initramfs](#-kernel--initramfs)
- [Init system: ভূমিকা](#-init-system-ভূমিকা)
- [systemd বিস্তারিত](#-systemd-বিস্তারিত)
- [Unit types ও dependency](#-unit-types-ও-dependency)
- [systemctl ও journalctl](#-systemctl-ও-journalctl)
- [Cron vs systemd timers](#-cron-vs-systemd-timers)
- [Service hardening directives](#-service-hardening-directives)
- [SysVinit, Upstart, OpenRC, runit, s6](#-sysvinit-upstart-openrc-runit-s6)
- [Container init (tini, dumb-init)](#-container-init-tini-dumb-init)
- [Real-world: Node/PHP app deploy on DigitalOcean](#-real-world-nodephp-app-deploy-on-digitalocean)

---

## 📌 Boot sequence ধাপে ধাপে

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Power on → CPU reset vector → firmware (BIOS/UEFI)            │
│    POST (Power On Self Test)                                     │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. UEFI reads /EFI/BOOT/BOOTX64.EFI বা boot order entry          │
│    (BIOS legacy: MBR-এর প্রথম 446 byte)                          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. Bootloader (GRUB2) — kernel + initramfs RAM-এ load             │
│    GRUB menu, kernel cmdline (root=, ro/rw, console=)             │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. Kernel decompress, init hardware, mount initramfs              │
│    (initramfs = early userspace, only essential drivers)          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. initramfs /init script: load real root drivers (LVM, RAID,     │
│    LUKS), mount real root, switch_root → exec /sbin/init          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. /sbin/init → systemd (PID 1)                                   │
│    Reads default.target → graphical/multi-user → service start    │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. login prompt / SSH ready                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📌 BIOS vs UEFI

| বৈশিষ্ট্য         | BIOS (legacy)              | UEFI                          |
|------------------|----------------------------|--------------------------------|
| Boot mode        | MBR (max 2TB disk)         | GPT (8 ZB)                    |
| Bootloader area  | first 512 byte (MBR)       | EFI System Partition (FAT32)  |
| Secure Boot      | ❌                         | ✅                            |
| Fast Boot        | slow                       | fast (parallel init)          |
| Network boot     | PXE                        | HTTP boot, iPXE               |
| 16-bit real mode | ✅                         | 32/64-bit protected mode       |
| Era              | <2010                      | 2010+, প্রায় সব সার্ভার       |

```bash
# কোন mode-এ booted?
$ ls /sys/firmware/efi
config_table  efivars  fw_platform_size  systab    # → UEFI

# UEFI variables
$ efibootmgr -v
BootCurrent: 0001
BootOrder: 0001,0000,2001
Boot0001* Ubuntu  HD(1,GPT,...)/File(\EFI\ubuntu\shimx64.efi)
```

> 💡 DigitalOcean droplet UEFI ব্যবহার করে। AWS EC2 instance type-ভেদে দুটোই পাওয়া যায় (m5 = BIOS, m6i = UEFI)।

---

## 📌 Bootloader (GRUB)

GRUB2 এখন de-facto standard।

```bash
# GRUB config
$ ls /boot/grub/
grub.cfg  i386-pc/  fonts/  themes/  ...

$ ls /boot/efi/EFI/ubuntu/
grubx64.efi  shimx64.efi  mmx64.efi  grub.cfg

# Kernel cmdline
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-6.5 root=UUID=abc... ro console=tty1 quiet splash

# Kernel cmdline change করা
$ vim /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mitigations=auto transparent_hugepage=madvise"

$ sudo update-grub        # Debian/Ubuntu
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg     # RHEL
```

**GRUB rescue** — boot failure-এ menu-এ `e` press করে edit, single-user mode-এ বুট করুন:

```
linux /vmlinuz... root=... ro single
                                ↑
                                init=/bin/bash দিয়ে recovery shell
```

---

## 📌 Kernel + initramfs

### Kernel image

`/boot/vmlinuz-6.5.0-21-generic` — compressed kernel। GRUB-এর কাছে decompressor আছে (gzip/zstd)।

### initramfs / initrd

`/boot/initrd.img-6.5.0-21-generic` — একটি ছোট ramdisk যাতে real root mount-এর জন্য দরকারি driver, tool আছে।

```bash
# Inspect initramfs content
$ lsinitramfs /boot/initrd.img-$(uname -r) | head
.
bin
bin/cat
bin/sh
sbin
sbin/blkid
lib/modules/6.5.0-21-generic/kernel/drivers/scsi/...
lib/modules/.../crypto/aes_x86_64.ko    # LUKS-এর জন্য
init                                    # entry script
```

কেন দরকার? কারণ kernel-এ সব driver compiled-in নেই (size সমস্যা)। বুটে real root যদি LVM/RAID/LUKS-এ থাকে, সেগুলো module load করতে initramfs-এর tool লাগে।

```bash
# initramfs রিজেনারেট
$ sudo update-initramfs -u -k all       # Debian
$ sudo dracut --regenerate-all --force  # RHEL
```

---

## 📌 Init system: ভূমিকা

PID 1 — kernel-এর প্রথম userspace process। তার দায়িত্ব:

1. system services start করা proper order-এ
2. zombie process reap (orphan-এর parent)
3. signal handle (e.g., shutdown)
4. service supervise (crash → restart)

PID 1 die করলে kernel **panic** করে।

---

## 📌 systemd বিস্তারিত

systemd এখন সবচেয়ে dominant init। Ubuntu, Debian, RHEL, Fedora, Arch — সব defaults to systemd।

```
                       systemd (PID 1)
                            │
            ┌───────────────┼───────────────────┐
            │               │                   │
       systemd-journald  systemd-udevd     systemd-logind
       (logs)            (device events)   (user sessions)
            │
       systemd-networkd, systemd-resolved, systemd-timesyncd ...
            │
       user services: nginx, mysql, php-fpm, node-app
```

### বৈশিষ্ট্য

- **Parallel start** — dependency graph ধরে যেগুলো independent সব একসাথে
- **Socket activation** — service শুধু request আসলেই start (inetd-এর modern version)
- **cgroup-based service tracking** — fork করে পালালেও systemd track করে
- **journald** — structured binary log
- **timer** — cron-এর replacement
- **resource limit / sandbox** — built-in security

---

## 📌 Unit types ও dependency

| Unit type    | Suffix       | কী represent করে                       |
|--------------|--------------|------------------------------------------|
| Service      | `.service`   | একটি daemon (nginx, php-fpm)             |
| Socket       | `.socket`    | listening socket (UDS/TCP)               |
| Timer        | `.timer`     | scheduled trigger                        |
| Mount        | `.mount`     | filesystem mount                         |
| Automount    | `.automount` | on-demand mount                          |
| Target       | `.target`    | grouping (multi-user.target)            |
| Device       | `.device`    | udev event                               |
| Path         | `.path`      | file change watch                        |
| Slice        | `.slice`     | cgroup group                             |
| Scope        | `.scope`     | externally created cgroup (e.g. login)  |

```bash
# একটি .service file
$ cat /etc/systemd/system/pathao-api.service
[Unit]
Description=Pathao API (Node.js)
After=network-online.target redis.service
Wants=network-online.target
Requires=redis.service

[Service]
Type=simple
User=pathao
Group=pathao
WorkingDirectory=/opt/pathao-api
ExecStart=/usr/bin/node /opt/pathao-api/server.js
ExecReload=/bin/kill -USR2 $MAINPID
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
KillSignal=SIGTERM

# Logs
StandardOutput=journal
StandardError=journal

# Environment
Environment="NODE_ENV=production"
EnvironmentFile=/etc/pathao-api/env

[Install]
WantedBy=multi-user.target
```

### Service Type-গুলো

| Type       | কখন                                                |
|------------|----------------------------------------------------|
| `simple`   | foreground process (default)                       |
| `exec`     | simple এর মত, কিন্তু execve সফল হওয়ার জন্য wait    |
| `forking`  | classic daemon (nginx, sshd) যে fork করে background |
| `oneshot`  | run once, exit (firewall rules apply)              |
| `notify`   | service systemd-কে READY signal পাঠায়              |
| `dbus`     | D-Bus service                                      |

### Dependencies

| Directive   | Semantic                                           |
|-------------|----------------------------------------------------|
| `Requires=` | hard dependency, fail = এটিও fail                  |
| `Wants=`    | soft dependency, ignored if fail                   |
| `After=`    | order — এই unit-এর পরে start                       |
| `Before=`   | order — এই unit-এর আগে                             |
| `Conflicts=`| একসাথে চলবে না                                     |
| `BindsTo=`  | "আঠাল" — অন্যটি stop হলে এটাও                      |

---

## 📌 systemctl ও journalctl

```bash
# Service control
$ sudo systemctl start  pathao-api
$ sudo systemctl stop   pathao-api
$ sudo systemctl restart pathao-api
$ sudo systemctl reload pathao-api          # ExecReload
$ sudo systemctl enable --now pathao-api    # boot-এ + এখন

# Status
$ systemctl status pathao-api
● pathao-api.service - Pathao API (Node.js)
     Loaded: loaded (/etc/systemd/system/pathao-api.service; enabled)
     Active: active (running) since Mon 2025-01-15 09:00:00 +06; 5h ago
   Main PID: 12345 (node)
      Tasks: 11 (limit: 4915)
     Memory: 245.3M
        CPU: 1min 23.456s
     CGroup: /system.slice/pathao-api.service
             └─12345 /usr/bin/node /opt/pathao-api/server.js

# Logs (journalctl)
$ journalctl -u pathao-api               # সব
$ journalctl -u pathao-api -f            # follow (tail -f)
$ journalctl -u pathao-api --since "1 hour ago"
$ journalctl -u pathao-api -p err        # priority error+
$ journalctl -u pathao-api -o json        # structured
$ journalctl _PID=12345                   # specific process
$ journalctl --boot                       # current boot
$ journalctl --list-boots
$ journalctl --disk-usage

# Failed services
$ systemctl --failed

# Boot analysis
$ systemd-analyze
Startup finished in 1.234s (kernel) + 5.678s (userspace) = 6.912s
multi-user.target reached after 5.430s in userspace.

$ systemd-analyze blame | head
3.456s NetworkManager-wait-online.service
1.234s docker.service
0.890s mysql.service
...

$ systemd-analyze critical-chain
$ systemd-analyze plot > boot.svg

# Dependency tree
$ systemctl list-dependencies pathao-api
$ systemctl list-dependencies --reverse mysql.service
```

### Drop-in override

main unit file edit না করে override:

```bash
$ sudo systemctl edit pathao-api
# এডিটর খুলবে /etc/systemd/system/pathao-api.service.d/override.conf
[Service]
LimitNOFILE=65535
Environment="LOG_LEVEL=debug"
```

---

## 📌 Cron vs systemd timers

### Classic cron

```bash
$ crontab -e
# m h dom mon dow command
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
*/5 * * * * /usr/local/bin/heartbeat.sh
@reboot /opt/app/init.sh
```

### systemd timer (modern)

```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=Daily DB Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=postgres
```

```ini
# /etc/systemd/system/db-backup.timer
[Unit]
Description=Daily DB Backup Timer

[Timer]
OnCalendar=daily
Persistent=true        # missed run (server off) catch up
RandomizedDelaySec=10m

[Install]
WantedBy=timers.target
```

```bash
$ systemctl enable --now db-backup.timer
$ systemctl list-timers
NEXT                         LEFT          UNIT
Tue 2025-01-16 00:00:00 +06  9h left       db-backup.timer
Mon 2025-01-15 15:00:00 +06  44min left    apt-daily.timer
```

| বৈশিষ্ট্য         | cron          | systemd timer           |
|------------------|---------------|--------------------------|
| Logging          | redirect by hand | journalctl auto       |
| Missed run catchup | ❌          | `Persistent=true` ✅     |
| Resource control | ❌            | cgroup limits ✅         |
| Random jitter    | ❌            | `RandomizedDelaySec` ✅  |
| Calendar syntax  | classic       | flexible (`OnCalendar=Mon..Fri 09:00`) |
| Dependency       | ❌            | unit dependency ✅       |

---

## 📌 Service hardening directives

systemd একটি service-কে **almost-container** বানাতে পারে — কিছু directive দিয়ে।

```ini
[Service]
# Filesystem
ProtectSystem=strict             # /usr, /boot, /etc read-only
ProtectHome=true                 # /home, /root inaccessible
ReadWritePaths=/var/log/api /var/lib/api
PrivateTmp=true                  # নিজস্ব /tmp
ProtectKernelTunables=true       # /proc/sys, /sys read-only
ProtectKernelModules=true        # module load disable
ProtectControlGroups=true        # /sys/fs/cgroup read-only
ProtectKernelLogs=true
ProtectClock=true
ProtectHostname=true
ProtectProc=invisible

# Privilege
NoNewPrivileges=true             # setuid effects neutralized
User=pathao
Group=pathao
DynamicUser=true                 # auto-allocated UID, ephemeral
SupplementaryGroups=

# Capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Network
PrivateNetwork=false              # true হলে network namespace isolated
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
IPAddressDeny=any
IPAddressAllow=10.0.0.0/8 127.0.0.1

# Syscall filter (seccomp)
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallArchitectures=native

# Resource limits (cgroup)
LimitNOFILE=65535
LimitNPROC=4096
MemoryMax=1G
CPUQuota=200%                    # 2 CPU cores
TasksMax=512
IOWeight=100

# Restart policy
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=60
StartLimitBurst=5
```

**Audit:**

```bash
$ systemd-analyze security pathao-api
  → score 1.5 SAFE (0=secure, 10=dangerous)
  → কোন directive missing list দেখাবে
```

> 💡 DigitalOcean BD droplet-এ Node app — এই directive-গুলোর সাহায্যে effectively containerless container পাবেন।

---

## 📌 SysVinit, Upstart, OpenRC, runit, s6

| Init system   | Era         | Style                  | বৈশিষ্ট্য                          |
|---------------|-------------|-------------------------|-------------------------------------|
| **SysVinit**  | 1983–       | sequential shell scripts (`/etc/init.d/`) | runlevels (0-6), simple |
| **Upstart**   | 2006–2014   | event-driven           | Ubuntu (replaced by systemd)        |
| **OpenRC**    | 2007–       | scripts + supervisor    | Alpine, Gentoo                      |
| **systemd**   | 2010–       | unit files, parallel    | Linux mainstream                    |
| **runit**     | —           | minimal supervise tree  | Void Linux, simple                  |
| **s6**        | —           | very minimal, robust    | container-friendly                  |

### SysVinit example

```bash
$ ls /etc/init.d/
nginx  mysql  cron  ...

$ /etc/init.d/nginx start
$ chkconfig nginx on        # RHEL
$ update-rc.d nginx defaults # Debian

$ runlevel
N 5

$ telinit 1                 # single user mode
```

### Alpine OpenRC (container base এ এখনও)

```bash
$ rc-status
$ rc-service nginx start
$ rc-update add nginx default
```

> 💡 Alpine Linux container image OpenRC ব্যবহার করে — small footprint। কিন্তু কন্টেইনারে সাধারণত init system লাগে না, app-ই PID 1।

---

## 📌 Container init (tini, dumb-init)

কন্টেইনার-এ application নিজেই PID 1। কিন্তু সব app PID 1-এর দায়িত্ব ঠিকঠাক পালন করে না:

1. **Zombie reaping** — child exit হলে SIGCHLD পেয়ে wait() করতে হবে
2. **Signal forwarding** — SIGTERM child-এ forward
3. **PID 1 default signal mask** — ignore many signals!

### Why this matters — "PID 1 problem"

```dockerfile
# ❌ shell wrapper
CMD bash -c "node server.js"
# bash PID 1 → SIGTERM forward করে না → কন্টেইনার stop slow
```

### tini

```dockerfile
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

- ছোট (24KB) — শুধু signal proxy + zombie reap
- Docker `--init` flag-এ built-in

```bash
$ docker run --init -d --name web myapp:1.0
# implicitly tini wraps entrypoint
```

### dumb-init (Yelp)

similar; common in Python/Java images।

```dockerfile
RUN apt-get install -y dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "-m", "myapp"]
```

### s6-overlay — multi-process container

কখনো কখনো একটি কন্টেইনারে cron + nginx + php-fpm দরকার (anti-pattern, কিন্তু legacy migration)। `s6-overlay` proper supervisor + init প্রদান করে।

```dockerfile
FROM alpine
RUN apk add --no-cache s6-overlay
COPY services.d/ /etc/services.d/
ENTRYPOINT ["/init"]
```

> ⚠️ তবে recommendation: একটি কন্টেইনার = একটি প্রসেস। sidecar-এর জন্য আলাদা container।

---

## 📌 Real-world: Node/PHP app deploy on DigitalOcean

### Step 1: User & directory

```bash
sudo useradd -r -s /usr/sbin/nologin pathao
sudo mkdir -p /opt/pathao-api /var/log/pathao-api
sudo chown -R pathao:pathao /opt/pathao-api /var/log/pathao-api
```

### Step 2: Hardened Node service

```bash
sudo tee /etc/systemd/system/pathao-api.service > /dev/null <<'EOF'
[Unit]
Description=Pathao API
After=network-online.target redis.service mysql.service
Wants=network-online.target

[Service]
Type=simple
User=pathao
Group=pathao
WorkingDirectory=/opt/pathao-api
EnvironmentFile=/etc/pathao-api/env
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
KillSignal=SIGTERM

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/log/pathao-api
ProtectHome=true
PrivateTmp=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
LockPersonality=true
MemoryMax=512M
CPUQuota=100%
LimitNOFILE=65535

# Logs
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now pathao-api
sudo systemctl status pathao-api
```

### Step 3: Hardened PHP-FPM (Laravel)

PHP-FPM-এর own pool config-এর পাশাপাশি systemd:

```bash
# /etc/systemd/system/php8.3-fpm.service.d/override.conf
[Service]
LimitNOFILE=65535
ProtectSystem=full
ReadWritePaths=/var/log/php /var/lib/php /var/www/laravel/storage
NoNewPrivileges=true
PrivateTmp=true
MemoryMax=2G
```

### Step 4: Socket activation (advanced)

systemd-এ socket activation দিয়ে — service শুধু request আসলে start হবে:

```ini
# pathao-api.socket
[Unit]
Description=Pathao API socket

[Socket]
ListenStream=/run/pathao-api.sock
SocketUser=www-data
SocketMode=0660

[Install]
WantedBy=sockets.target
```

```ini
# pathao-api.service
[Service]
Type=notify
StandardInput=socket
ExecStart=/usr/bin/node server.js
```

Node app তখন stdin-এ inherited fd পাবে, listen করবে — boot time-এ memory save।

### Step 5: Backup timer

```ini
# /etc/systemd/system/mysql-backup.service
[Service]
Type=oneshot
User=mysql
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/mysql-backup.timer
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now mysql-backup.timer
```

### Step 6: Boot performance

```bash
$ systemd-analyze blame | head -5
2.345s NetworkManager-wait-online.service
1.234s snapd.seeded.service
0.789s docker.service
0.456s mysql.service

# slow ones disable / mask
$ sudo systemctl mask snapd.seeded.service       # ক্লাউড server-এ snap unused
```

---

## 🎯 সংক্ষেপে

- Boot: firmware → bootloader → kernel + initramfs → init (PID 1) → services
- UEFI আধুনিক standard, GPT/Secure Boot সাপোর্ট
- GRUB2 dominant bootloader; kernel cmdline-এ অনেক tuning সম্ভব
- initramfs early userspace যাতে real root mount করা যায়
- **systemd** dominant init — unit, dependency, parallel, cgroup, journald
- timer = modern cron (`Persistent`, jitter, journal)
- Service hardening directives দিয়ে almost-container security
- Container PID 1 problem → tini/dumb-init দিয়ে সমাধান
- `systemd-analyze`, `journalctl`, `systemctl` — daily SRE tool

> 🇧🇩 **DigitalOcean BD droplet pattern:** Node + PHP-FPM + MySQL + Redis সবই hardened systemd service হিসেবে। `systemd-analyze security` score 1.5–2.0 maintain করুন।  
> Pathao multi-instance VPS: `socket activation` দিয়ে cold service auto-start।  
> bKash batch jobs = systemd timer (cron নয়) — `Persistent=true` reboot recovery।
