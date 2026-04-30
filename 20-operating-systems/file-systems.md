# 📂 File Systems — Linux ফাইলসিস্টেম গভীরে

> ফাইলসিস্টেম শুধু "ফাইল রাখার জায়গা" নয় — এটি একটি **abstraction layer** যা ডিস্কের raw block-কে inode, dentry, file descriptor-এ রূপান্তর করে।
> Daraz CI build cache, bKash MySQL fsync, Pathao container layer — সবকিছুর পেছনে এই subsystem।

---

## 📖 সূচিপত্র

- [VFS — Virtual File System abstraction](#-vfs--virtual-file-system-abstraction)
- [Inode, Dentry, Superblock, File](#-inode-dentry-superblock-file)
- [ext4 — Journaling ও Extents](#-ext4--journaling-ও-extents)
- [XFS, Btrfs, ZFS তুলনা](#-xfs-btrfs-zfs-তুলনা)
- [tmpfs ও বিশেষ ফাইলসিস্টেম](#-tmpfs-ও-বিশেষ-ফাইলসিস্টেম)
- [OverlayFS — Docker layer-এর জাদু](#-overlayfs--docker-layer-এর-জাদু)
- [FAT/NTFS overview](#-fatntfs-overview)
- [Page Cache ও Writeback](#-page-cache-ও-writeback)
- [fsync, fdatasync, O_DIRECT](#-fsync-fdatasync-o_direct)
- [Hard link vs Symlink](#-hard-link-vs-symlink)
- [Permission, setuid, setgid, sticky, ACL](#-permission-setuid-setgid-sticky-acl)
- [Extended Attributes (xattr)](#-extended-attributes-xattr)
- [Inotify / fanotify](#-inotify--fanotify)
- [PHP/Node ফাইল I/O ও cache hit](#-phpnode-ফাইল-io-ও-cache-hit)
- [টুলস: df, du, lsof, iotop, blktrace](#-টুলস-df-du-lsof-iotop-blktrace)
- [প্রোডাকশন pitfall ও বাংলাদেশি case](#-প্রোডাকশন-pitfall-ও-বাংলাদেশি-case)

---

## 📌 VFS — Virtual File System abstraction

Linux এ বিভিন্ন ধরনের ফাইলসিস্টেম (ext4, XFS, NFS, FAT, tmpfs, overlay) আছে। কিন্তু `open()`, `read()`, `write()` সিস্কল প্রতিটির জন্য **একই**। এটাই **VFS** — kernel-এর ভেতরের একটি **abstract layer** যা সব ফাইলসিস্টেমকে একটি common interface দেয়।

```
┌────────────────────────────────────────────────────────────┐
│  User space:   PHP fopen() / Node fs.open() / cat / cp     │
└────────────────────────────────────────────────────────────┘
                       │ syscall: open/read/write/stat
                       ▼
┌────────────────────────────────────────────────────────────┐
│              VFS (Virtual File System)                     │
│   - super_block, inode, dentry, file structures            │
│   - common API: ->read_iter, ->write_iter, ->lookup        │
└────────────────────────────────────────────────────────────┘
       │           │           │           │           │
       ▼           ▼           ▼           ▼           ▼
   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────────┐
   │ ext4 │   │  XFS │   │ NFS  │   │tmpfs │   │ overlay  │
   └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘   └────┬─────┘
      ▼          ▼          ▼          ▼            ▼
   block I/O   block      network     RAM        upper+lower
              I/O         (RPC)                  ফাইলসিস্টেম
```

**মূল ধারণা:** প্রতিটি ফাইলসিস্টেম VFS-এ register করে কিছু **operation table** (struct file_operations, inode_operations, super_operations)। কার্নেল syscall handle করার সময় শুধু সেই function pointer call করে।

---

## 📌 Inode, Dentry, Superblock, File

Linux ফাইলসিস্টেমের চারটি core data structure বুঝলে বাকি সব সহজ:

| Structure        | কী রাখে                                                     | কোথায়    |
|------------------|-------------------------------------------------------------|-----------|
| **superblock**   | পুরো ফাইলসিস্টেমের metadata (size, block size, inode count) | ডিস্কের শুরুতে + RAM cache |
| **inode**        | একটি ফাইলের metadata (size, perms, owner, block pointer)    | inode table |
| **dentry**       | নাম → inode mapping (path lookup cache)                     | RAM only   |
| **file**         | একটি open file-এর state (offset, mode, fd)                  | per-process |

```
path: /var/www/laravel/.env

  /        ──► dentry "/"      ──► inode 2     (root)
   var/    ──► dentry "var"    ──► inode 1024
   www/    ──► dentry "www"    ──► inode 5012
   laravel/──► dentry "laravel"──► inode 8801
   .env    ──► dentry ".env"   ──► inode 10293 ──► [block 4500..4502]
```

**Inode-এ যা থাকে (ext4):**

```
struct ext4_inode {
    __le16  i_mode;        // rwxr-xr-x ইত্যাদি
    __le16  i_uid;         // owner UID
    __le32  i_size_lo;     // size in bytes
    __le32  i_atime;       // access time
    __le32  i_ctime;       // metadata change
    __le32  i_mtime;       // modification time
    __le32  i_dtime;       // deletion time
    __le16  i_gid;         // group GID
    __le16  i_links_count; // hard link count
    __le32  i_blocks_lo;   // disk blocks used (512B units)
    __le32  i_block[15];   // direct + indirect block pointers / extent tree
    __le32  i_generation;
    ...
};
```

> 💡 ফাইলের **নাম** কিন্তু inode-এ থাকে না — থাকে directory entry-তে। এজন্যই **hard link** সম্ভব।

```bash
# inode নম্বর দেখুন
$ ls -li /etc/passwd
1052673 -rw-r--r-- 1 root root 2891 Oct 12 14:33 /etc/passwd

$ stat /etc/passwd
  File: /etc/passwd
  Size: 2891      Blocks: 8          IO Block: 4096   regular file
Device: 252,1     Inode: 1052673     Links: 1
Access: (0644/-rw-r--r--)  Uid: (0/root)   Gid: (0/root)
Access: 2025-01-15 09:12:41
Modify: 2024-10-12 14:33:08
Change: 2024-10-12 14:33:08
```

---

## 📌 ext4 — Journaling ও Extents

**ext4** এখনও বাংলাদেশের অধিকাংশ DigitalOcean droplet, AWS Mumbai EC2-এর ডিফল্ট ফাইলসিস্টেম।

### Journaling — crash-safe metadata

ext4 metadata পরিবর্তন **journal**-এ আগে লেখে, তারপর actual block-এ। ফলে power-failure-এও ফাইলসিস্টেম corrupt হয় না।

```
Application: write("data")
        │
        ▼
   ┌─────────────────────────────────────┐
   │  Step 1: Journal-এ metadata লেখা    │  (write-ahead log)
   │   - "block 4500 update হবে"          │
   │   - commit record                    │
   └─────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────┐
   │  Step 2: actual block 4500 update    │
   └─────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────┐
   │  Step 3: journal entry mark "done"   │
   └─────────────────────────────────────┘
```

**তিনটি journal mode:**

| Mode       | কী journal হয়                         | speed | safety |
|------------|----------------------------------------|-------|--------|
| `journal`  | metadata + data দুটোই                  | 🐢    | 🛡️🛡️🛡️ |
| `ordered`  | শুধু metadata, data আগে commit (default) | 🚶   | 🛡️🛡️ |
| `writeback`| শুধু metadata, data যেকোনো সময়         | 🏃    | 🛡️    |

```bash
# mount option দেখুন
$ mount | grep ext4
/dev/vda1 on / type ext4 (rw,relatime,errors=remount-ro)

# bKash payment DB server-এ data=journal use করা যেতে পারে যখন
# durability গুরুত্বপূর্ণ এবং UPS আছে
mount -o remount,data=journal /var/lib/mysql
```

### Extents — large file efficient

পুরনো ext2/ext3 প্রতিটি block-এর জন্য আলাদা pointer রাখত (indirect block tree)। 10GB ভিডিও ফাইলে millions of pointer! ext4-এর **extent** একটি tuple রাখে: `(start_block, length)`।

```
পুরনো ext3 (block pointer):
  inode → [4500] [4501] [4502] [4503] ... [14499]   (10000 entries!)

ext4 extent:
  inode → extent_header
            └─► (start=4500, length=10000)   (একটি entry!)
```

ফলে large file (Daraz product image, video) হ্যান্ডলিং অনেক fast।

---

## 📌 XFS, Btrfs, ZFS তুলনা

| ফিচার                | ext4         | XFS           | Btrfs          | ZFS              |
|----------------------|--------------|---------------|----------------|------------------|
| Max file size        | 16 TiB       | 8 EiB         | 16 EiB         | 16 EiB           |
| Snapshot             | ❌           | ❌            | ✅             | ✅               |
| Subvolume            | ❌           | ❌            | ✅             | ✅               |
| Built-in checksum    | ❌ (metadata only) | metadata | data + metadata| data + metadata  |
| Built-in RAID        | ❌           | ❌            | ✅ (RAID 0/1/10)| ✅ (RAIDZ)      |
| Compression          | ❌           | ❌            | ✅ (zstd, lzo)| ✅ (lz4, zstd)  |
| Online resize        | grow only    | grow only     | grow + shrink  | grow only        |
| Best for             | general      | large files, parallel I/O | snapshot-heavy | enterprise storage |

```bash
# RHEL/CentOS-এ XFS ডিফল্ট, /var/lib/docker-এ XFS দেখা যায়
$ df -T /var/lib/docker
Filesystem     Type   1K-blocks    Used Available Use% Mounted on
/dev/nvme1n1   xfs    104857600 8923456  95934144   9% /var/lib/docker
```

> 💡 **কখন XFS:** Pathao map tile server-এর মতো বিশাল ফাইল, বহু parallel writer।  
> **কখন Btrfs:** developer laptop, snapshot-based backup।  
> **কখন ZFS:** on-prem storage server, data integrity critical।

---

## 📌 tmpfs ও বিশেষ ফাইলসিস্টেম

**tmpfs** RAM-ভিত্তিক — `/tmp`, `/dev/shm`, `/run` এ ব্যবহার হয়। reboot করলে সব মুছে যায়, কিন্তু গতি disk-এর চেয়ে ১০০x।

```bash
$ mount | grep tmpfs
tmpfs on /run type tmpfs (rw,nosuid,nodev,size=1612288k)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,size=2g)

# Laravel session-এ tmpfs ব্যবহার করলে disk hit কমে
mount -t tmpfs -o size=512M tmpfs /var/www/laravel/storage/framework/sessions
```

**অন্যান্য in-memory FS:**

| FS        | Mount point            | কাজ                                  |
|-----------|------------------------|---------------------------------------|
| `procfs`  | `/proc`                | প্রসেস ও কার্নেল info                |
| `sysfs`   | `/sys`                 | কার্নেল object hierarchy              |
| `cgroup2` | `/sys/fs/cgroup`       | resource control                      |
| `devtmpfs`| `/dev`                 | device node                           |
| `debugfs` | `/sys/kernel/debug`    | kernel debugging                      |
| `tracefs` | `/sys/kernel/tracing`  | ftrace, eBPF tracing                  |

---

## 📌 OverlayFS — Docker layer-এর জাদু

Docker image যে "layer"-এর কথা শোনেন — সেটি কার্নেল-এর **OverlayFS**। একটি **read-only lower** এবং একটি **read-write upper** কে একসাথে mount করে।

```
                ┌──────────────────────────┐
   কন্টেইনার দেখে │  merged view: /         │
                └──────────────────────────┘
                          ▲
            ┌─────────────┴──────────────┐
            │                            │
   ┌────────────────┐         ┌──────────────────────┐
   │  upperdir (RW) │         │  lowerdir (RO)       │
   │  /var/lib/...  │         │  - layer3 (app code)  │
   │  /diff         │         │  - layer2 (composer)  │
   │  নতুন/পরিবর্তিত │         │  - layer1 (php base)  │
   │  ফাইল          │         │  - layer0 (alpine)    │
   └────────────────┘         └──────────────────────┘
```

**Copy-up:** কন্টেইনার যখন lower-এর একটি ফাইল edit করে, OverlayFS সেটি upper-এ **copy করে** তারপর modify করে। মূল lower অপরিবর্তিত থাকে।

```bash
# Docker overlay দেখুন
$ docker run -d --name web nginx
$ mount | grep overlay
overlay on /var/lib/docker/overlay2/abc123.../merged type overlay \
  (rw,relatime,lowerdir=L1:L2:L3,upperdir=U/diff,workdir=U/work)

# manual mount
mkdir -p /tmp/lower /tmp/upper /tmp/work /tmp/merged
echo "base" > /tmp/lower/file.txt
mount -t overlay overlay \
    -o lowerdir=/tmp/lower,upperdir=/tmp/upper,workdir=/tmp/work \
    /tmp/merged

cat /tmp/merged/file.txt          # "base"
echo "modified" > /tmp/merged/file.txt
ls /tmp/upper/                    # file.txt (copy-up হয়েছে)
ls /tmp/lower/                    # file.txt (অপরিবর্তিত)
```

**Daraz CI-তে impact:** একই base image (`php:8.3-fpm-alpine`) ১০০টা microservice শেয়ার করে। প্রতিটির আলাদা upper layer। ফলে ১০০ × 200MB = 20GB এর জায়গায় মাত্র 200MB + 100 × small upper।

---

## 📌 FAT/NTFS overview

**FAT32 / exFAT** — pendrive, SD card, BIOS partition। কোনো journal নেই, no permission, max file 4GB (FAT32)।

**NTFS** — Windows-এর native। MFT (Master File Table), journaling (USN journal), ACL, compression। Linux-এ `ntfs-3g` (FUSE) বা kernel-এর নতুন `ntfs3` ড্রাইভার দিয়ে read-write।

```bash
# Linux server-এ Windows backup mount
mount -t ntfs3 /dev/sdb1 /mnt/winbackup
```

---

## 📌 Page Cache ও Writeback

ফাইল `read()` করলে kernel আগে **page cache** (RAM) চেক করে। না থাকলে disk থেকে এনে cache-এ রাখে। `write()` সাধারণত শুধু page cache-এ লেখে — disk-এ যায় পরে ("dirty page writeback")।

```
        read("/var/www/index.php")
                │
                ▼
        ┌─────────────────────┐
        │   Page Cache (RAM)  │
        │   ┌─────────────┐   │
        │   │ index.php   │   │  hit  ✅ (fast: ~100ns)
        │   │ 4KB pages   │   │
        │   └─────────────┘   │
        └─────────────────────┘
                │ miss
                ▼
        ┌─────────────────────┐
        │   Block I/O layer   │
        │   I/O scheduler     │
        └─────────────────────┘
                │
                ▼
        ┌─────────────────────┐
        │   Disk (SSD/HDD)    │  ~100µs (NVMe) – 10ms (HDD)
        └─────────────────────┘
```

**Dirty page writeback** controlled by:

```bash
# Dirty data কত % হলে background writeback শুরু
$ sysctl vm.dirty_background_ratio
vm.dirty_background_ratio = 10

# Dirty data কত % হলে process নিজেই block হয়ে writeback করবে
$ sysctl vm.dirty_ratio
vm.dirty_ratio = 20

# Dirty page কত সেকেন্ড পর force flush
$ sysctl vm.dirty_expire_centisecs
vm.dirty_expire_centisecs = 3000     # 30 sec

# Cache দেখুন
$ free -h
              total   used   free   shared  buff/cache  available
Mem:           16Gi   3Gi    1Gi    200Mi    12Gi        12Gi
                                              ↑↑↑
                                  page cache + buffer
```

> ⚠️ **bKash MySQL server tuning:** যদি `dirty_ratio=20` থাকে এবং RAM 32GB, তাহলে 6.4GB dirty জমলে process **stall** করে। database এর জন্য এটি বিপজ্জনক — আমরা `dirty_bytes=256M`, `dirty_background_bytes=128M` সেট করি।

```bash
# Production tuning
echo 268435456 > /proc/sys/vm/dirty_bytes
echo 134217728 > /proc/sys/vm/dirty_background_bytes
```

---

## 📌 fsync, fdatasync, O_DIRECT

| Call          | কী করে                                                             | কখন                     |
|---------------|--------------------------------------------------------------------|--------------------------|
| `write()`     | শুধু page cache-এ লেখে                                              | সাধারণ                   |
| `fsync(fd)`   | data + metadata ডিস্কে force flush                                  | DB commit, critical save |
| `fdatasync(fd)`| data flush, metadata শুধু সাইজ পরিবর্তন হলে                        | DB log (faster fsync)    |
| `O_DIRECT`    | page cache bypass করে সরাসরি ডিস্কে                                  | DB own caching আছে        |
| `O_SYNC`      | প্রতিটি write fsync-এর মতো                                          | বিরল                     |

```php
<?php
// PHP — Laravel-এ critical write (e.g. payment log)
$fp = fopen('/var/log/payments.log', 'a');
fwrite($fp, json_encode($txn) . "\n");
// শুধু fwrite যথেষ্ট নয় — power loss হলে data হারাতে পারে
fflush($fp);                  // userspace buffer → kernel
// পরে syscall করে kernel buffer → disk
$meta = stream_get_meta_data($fp);
// PHP-তে fsync নেই, but PHP 8.1+ এ আছে:
fsync($fp);                   // data + metadata
// অথবা
fdatasync($fp);               // শুধু data
fclose($fp);
```

```javascript
// Node.js — Pathao ride save
const fs = require('fs');
const fd = fs.openSync('/var/log/rides.log', 'a');
fs.writeSync(fd, JSON.stringify(ride) + '\n');
fs.fsyncSync(fd);            // disk-এ guarantee
// performance critical হলে fdatasync:
fs.fdatasyncSync(fd);
fs.closeSync(fd);
```

> 💡 **PostgreSQL** ডিফল্টে প্রতিটি commit-এ fsync করে। SSD-তে fsync ~100µs, HDD-তে ~10ms — তাই DB-server-এ NVMe SSD বাধ্যতামূলক।

---

## 📌 Hard link vs Symlink

```
hard link:                       symbolic link:
                                 
  dentry "a.txt" ─┐               dentry "link" ─► inode (type=symlink)
                  ├─► inode 100                       │
  dentry "b.txt" ─┘                                   ▼
                                                "a.txt" (path string)
                                                      │
                                                      ▼
                                              dentry "a.txt" ─► inode 100
```

| বৈশিষ্ট্য          | Hard link              | Symbolic link (symlink) |
|-------------------|-----------------------|--------------------------|
| Inode             | একই inode             | আলাদা inode              |
| Different FS      | ❌                    | ✅                       |
| Directory         | ❌ (root ছাড়া)        | ✅                       |
| Original delete   | ফাইল বাঁচে            | dangling link            |
| Size              | শূন্য overhead         | path string size         |

```bash
# create
ln    /etc/passwd /root/passwd-hard      # hard
ln -s /etc/passwd /root/passwd-sym       # symbolic

ls -li /etc/passwd /root/passwd-hard /root/passwd-sym
# 1052673 -rw-r--r-- 2 root root 2891 ... /etc/passwd
# 1052673 -rw-r--r-- 2 root root 2891 ... /root/passwd-hard   (একই inode!)
# 5012345 lrwxrwxrwx 1 root root   11 ... /root/passwd-sym -> /etc/passwd

# i_links_count = 2 — hard link delete করলে file বাঁচবে
```

---

## 📌 Permission, setuid, setgid, sticky, ACL

```
  - rwx rwx rwx
  │ │   │   │
  │ │   │   └── others
  │ │   └────── group
  │ └────────── owner (user)
  └──────────── file type (-, d, l, c, b, s, p)
```

**সংখ্যাগত mode:**

```
  4 = read    2 = write    1 = execute
  
  chmod 754 file.sh   ⇒  rwx r-x r--
                          7   5   4
```

**Special bits:**

| Bit        | Octal | Symbol | কাজ                                                               |
|------------|-------|--------|--------------------------------------------------------------------|
| setuid     | 4000  | `s` (user x) | program owner-এর UID দিয়ে চলে (e.g. `passwd`)              |
| setgid     | 2000  | `s` (group x)| ফাইল: group-এর mode; ডিরেক্টরি: নতুন ফাইল parent-এর group পায় |
| sticky     | 1000  | `t` (other x)| ডিরেক্টরিতে: নিজের ফাইল ছাড়া কেউ delete করতে পারবে না (`/tmp`) |

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 ... /usr/bin/passwd
   ↑
   setuid — যেকোনো user চালালেও root হিসেবে রান হবে (যাতে /etc/shadow লিখতে পারে)

$ ls -ld /tmp
drwxrwxrwt 23 root root 4096 ... /tmp
        ↑
        sticky — সবাই লিখতে পারে, কিন্তু নিজের ফাইল ছাড়া delete নয়
```

> ⚠️ **Security:** কন্টেইনারে `setuid` binary-গুলোই **privilege escalation**-এর প্রধান route। তাই Pathao production image-এ আমরা `--cap-drop=ALL`, `no-new-privileges:true`, এবং Dockerfile-এ:
> ```dockerfile
> RUN find / -perm /6000 -type f -exec chmod a-s {} \; 2>/dev/null || true
> ```

### POSIX ACL — fine-grained permission

ক্লাসিক `rwxrwxrwx` শুধু একটি owner+group কাভার করে। ACL দিয়ে multiple user/group-এর জন্য আলাদা permission।

```bash
# install
apt install acl

# একটি specific user-কে read access
setfacl -m u:rakib:r-- /var/log/nginx/access.log
getfacl /var/log/nginx/access.log

# default ACL (ডিরেক্টরিতে নতুন ফাইল-এ inherit)
setfacl -d -m g:devs:rwx /var/www/laravel/storage/logs

# `+` চিহ্ন দেখায় ACL set আছে
$ ls -l /var/log/nginx/access.log
-rw-r--r--+ 1 www-data adm 1.2M ... access.log
```

---

## 📌 Extended Attributes (xattr)

xattr — inode-এ extra key/value metadata। SELinux label, file capability, integrity hash এখানে রাখা হয়।

```bash
# capability binary-তে rather than setuid
setcap cap_net_bind_service+ep /usr/local/bin/node
getcap /usr/local/bin/node
# /usr/local/bin/node = cap_net_bind_service+ep

# user xattr
setfattr -n user.checksum -v "sha256:abcd..." file.bin
getfattr -d file.bin
# user.checksum="sha256:abcd..."

# namespaces: security.*, system.*, trusted.*, user.*
```

> 💡 Node.js-কে port 80-তে bind করতে root ছাড়াই — `cap_net_bind_service` capability দিন। DigitalOcean droplet-এ সাধারণ pattern।

---

## 📌 Inotify / fanotify

ফাইল/ডিরেক্টরি change watch করার kernel API।

```javascript
// Node.js — fs.watch ব্যবহার করে inotify
const fs = require('fs');
fs.watch('/var/www/laravel/.env', (eventType, filename) => {
    console.log(`event: ${eventType}, file: ${filename}`);
    // app config reload
});
```

```bash
# CLI tool
apt install inotify-tools

inotifywait -m -r /var/www/uploads -e create -e modify -e delete
# /var/www/uploads/ CREATE photo.jpg
# /var/www/uploads/ MODIFY photo.jpg

# Daraz CI: dockerfile change হলে rebuild trigger
inotifywait -e modify /Dockerfile && docker build -t app .
```

**fanotify** — inotify-এর big brother, পুরো mount বা ফাইলসিস্টেম-wide monitor করতে পারে। antivirus, audit-এ ব্যবহার হয়।

**ulimit:**

```bash
$ sysctl fs.inotify.max_user_watches
fs.inotify.max_user_watches = 8192    # ⚠️ webpack/vite-এ কম পড়ে

# বাড়ান (Pathao dev workstation-এ)
echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf
sysctl -p
```

---

## 📌 PHP/Node ফাইল I/O ও cache hit

### PHP — opcache + realpath cache

PHP প্রতিটি request-এ ফাইল include/require করে। কার্নেল page cache সাহায্য করে, কিন্তু PHP-এর নিজের two-level cache আছে:

```ini
; php.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0    ; production-এ recheck বন্ধ
opcache.revalidate_freq=0

; Symlink/path cache
realpath_cache_size=4096K
realpath_cache_ttl=600
```

**কেন গুরুত্বপূর্ণ:** Daraz Laravel app-এ ~3000 PHP file। প্রতিটি request `realpath()` লাগে — kernel-এ syscall + dentry lookup। realpath cache না থাকলে CPU ৩০% বেশি।

### Node.js — fs sync vs async

```javascript
// ❌ খারাপ: event loop block করে (Pathao API-তে latency spike)
const data = fs.readFileSync('/etc/config.json');

// ✅ ভালো: kernel uses page cache, async
const data = await fs.promises.readFile('/etc/config.json');

// ✅ আরও ভালো: bulk read হলে stream
fs.createReadStream('/var/log/big.log')
  .on('data', chunk => process(chunk));
```

> **Cache hit pattern:** একটি 1GB log ফাইল `cat` করলে — প্রথমবার disk hit (slow), দ্বিতীয়বার পুরোটা page cache থেকে (RAM থাকলে)। `vmstat 1`-এ `bi` (block in) দেখুন।

---

## 📌 টুলস: df, du, lsof, iotop, blktrace

```bash
# ফাইলসিস্টেম usage
$ df -hT
Filesystem     Type   Size  Used Avail Use% Mounted on
/dev/vda1      ext4    50G   23G   25G  48% /
tmpfs          tmpfs  1.6G  4.0K  1.6G   1% /dev/shm
overlay        overlay 50G   23G   25G  48% /var/lib/docker/overlay2/.../merged

# inode usage (অনেক ছোট ফাইল হলে inode আগে শেষ হয়!)
$ df -hi
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/vda1        3.2M  450K  2.7M   14% /

# ডিরেক্টরি size
$ du -sh /var/log/*  | sort -h
$ du -h --max-depth=1 /var/lib/mysql

# কোন process কোন ফাইল open করেছে
$ lsof /var/log/nginx/access.log
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1234     root    7w  REG    8,1  102345  ... /var/log/nginx/access.log

# specific port-এর process
$ lsof -i :3306

# deleted but still open ফাইল (disk full debug)
$ lsof | grep deleted

# I/O top — কোন process disk hammering
$ iotop -oP

# কোন process কত I/O করেছে cumulative
$ pidstat -d 1

# block-level trace (advanced)
$ blktrace -d /dev/vda -o trace
$ blkparse trace.blktrace.0

# ফাইলসিস্টেম activity real-time (eBPF)
$ biolatency-bpfcc 5            # I/O latency histogram
$ biosnoop-bpfcc                  # প্রতিটি I/O log
$ filetop-bpfcc                   # most active files
```

---

## 📌 প্রোডাকশন pitfall ও বাংলাদেশি case

### ❌ Pitfall 1: "df -h বলছে full কিন্তু কিছুই নেই"

```bash
$ df -h /
/dev/vda1   50G   50G   0   100%  /

$ du -sh /*   # মোট ২০GB — তাহলে বাকি ৩০GB কোথায়?
```

**কারণ:** কোনো process একটা বড় ফাইল delete করেছে কিন্তু এখনও **fd open** রেখেছে। Inode reference count > 0, তাই block release হয়নি।

```bash
$ lsof | grep deleted | head
nginx 1234 root 12w REG 8,1 30G 0 /var/log/nginx/error.log (deleted)

# fix: process restart বা truncate
> /proc/1234/fd/12        # zero out করে release
systemctl restart nginx
```

bKash incident: একটা log rotate misconfig-এ /var/log full ছিল disk usage 100%, কিন্তু `du` show করছিল 5GB — কারণ rsyslog-এর deleted fd।

### ❌ Pitfall 2: inode exhaustion

ছোট ছোট লক্ষ লক্ষ ফাইল (PHP session, Laravel cache) → `df -h` বলছে 30% full কিন্তু `df -i` 100%।

```bash
$ df -i /var/lib/php/sessions
Filesystem  Inodes  IUsed  IFree IUse% Mounted on
/dev/vda1    3.2M   3.2M     0   100% /var/lib/php/sessions

# fix
find /var/lib/php/sessions -mtime +1 -delete

# অথবা ফাইলসিস্টেম-এ Redis-based session use করুন
```

### ❌ Pitfall 3: বড় ফাইল copy-up overhead

Docker overlay-এ একটি 5GB ML model ফাইল edit করলে পুরো 5GB lower → upper-এ copy হয়। Daraz recommendation engine container-এ এজন্য model ফাইল **volume mount**-এ রাখা হয়, image-এ না।

```yaml
# docker-compose.yml — model volume mount, copy-up এড়ানো
services:
  recommender:
    image: daraz/recommender:1.0
    volumes:
      - /data/models:/app/models:ro
```

### ❌ Pitfall 4: noatime ছাড়া unnecessary write

প্রতিটি `read()` inode-এর `atime` update করে — যা একটি **disk write**। read-heavy server-এ বিশাল overhead।

```bash
# /etc/fstab
/dev/vda1 / ext4 defaults,noatime,nodiratime 0 1
```

### ✅ Real-world pattern: Pathao log pipeline

```
Driver app  ──► Node.js API
                   │
                   │ fs.appendFile  (page cache)
                   ▼
              /var/log/api.log
                   │
                   │ inotify event
                   ▼
              Filebeat (read + ship)
                   ▼
              Kafka → Elasticsearch
```

`fdatasync` per-line use করলে latency 10x বাড়বে। তাই async append + Filebeat fanotify watch + 5s buffer।

---

## 🎯 সংক্ষেপে

- **VFS** Linux-এর সব ফাইলসিস্টেমকে একই API-তে আনে।
- **inode** = metadata, **dentry** = name→inode, **superblock** = FS-wide info।
- **ext4** journaling ও extent দিয়ে crash-safe + large file efficient।
- **OverlayFS** Docker layer-এর ভিত্তি — copy-up জানতে হবে।
- **Page cache** RAM, writeback async — DB tuning-এ `dirty_bytes` critical।
- **fsync/fdatasync** durability নিশ্চিত করে — production write-এ অবশ্যই use করুন।
- **lsof, iotop, blktrace, eBPF** দিয়ে I/O bottleneck খুঁজুন।
- inode exhaustion, deleted-but-open, atime overhead — common production bug।

> 🇧🇩 **Real-world:** bKash MySQL = ext4 + `noatime` + `dirty_bytes` tuning + NVMe।  
> Daraz CI = OverlayFS layer-এর সঠিক ordering = build cache hit ৭০% → ৯৫%।  
> Pathao API server = realpath cache + opcache = CPU ৩০% reduction।
