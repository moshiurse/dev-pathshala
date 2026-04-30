# 🛡️ Production Server Deployment & Hardening Guide

> **একটি ফাঁকা Linux VPS থেকে production-ready, secured, monitored server পর্যন্ত — step-by-step।**
> AWS EC2, DigitalOcean Droplet, Vultr, Linode, Eicra/ExonHost — যেকোনো VPS এ এই guide কাজ করবে।
> Daraz প্রোডাক্ট page এর latency, bKash এর uptime, Pathao এর scalability — সব শুরু হয় একটা সঠিকভাবে hardened server দিয়ে।

---

## 📖 সূচিপত্র

1. [Initial setup — distro choice, SSH](#1-️-initial-setup--distro-ssh)
2. [User & permissions](#2--user--permissions)
3. [Firewall — ufw/iptables/fail2ban](#3--firewall--ufw-iptables-fail2ban)
4. [System tuning — sysctl, ulimit, swap](#4--system-tuning--sysctl-ulimit-swap)
5. [Web server — Nginx/Apache + TLS](#5--web-server--nginxapache--tls)
6. [Application runtime](#6--application-runtime--php-fpm-nodejs-python)
7. [Database setup](#7-️-database-setup)
8. [CI/CD deployment](#8--cicd-deployment)
9. [Containerized deployment](#9--containerized-deployment)
10. [Monitoring & logging](#10--monitoring--logging)
11. [Backup & DR](#11--backup--dr)
12. [Security hardening (advanced)](#12--security-hardening-advanced)
13. [Zero-downtime deploy](#13--zero-downtime-deploy)
14. [CDN & WAF](#14--cdn--waf--cloudflare)
15. [Cost optimization](#15--cost-optimization)
16. [BD-specific tips](#16-️-bd-specific-tips)
17. [Production deployment checklist (50+)](#17--production-deployment-checklist-50)

---

## 1. ⚙️ Initial setup — distro, SSH

### Distro নির্বাচন

| Distro | কখন? | BD context |
|--------|------|-----------|
| **Ubuntu 22.04 LTS / 24.04 LTS** | ৯০% case এ default পছন্দ — community বড়, AWS/DO image ready | ✅ বেশিরভাগ BD dev এ familiar |
| **Debian 12 (Bookworm)** | Minimal, stable, fewer security CVE | server-only veteran এর জন্য |
| **AlmaLinux / Rocky 9** | RHEL-compatible — enterprise/SELinux | bKash, Bank — yum/dnf ecosystem |
| Amazon Linux 2023 | AWS-only, kernel optimized for EC2 | AWS Mumbai এ default |
| Arch / Gentoo | ❌ production এ avoid | unpredictable |

> **নিয়ম:** যা team জানে, তাই বেছে নিন। **LTS** (Long Term Support) version ছাড়া production এ যাবেন না — security update ৫ বছর পাবেন।

### প্রথম login এর পরে

```bash
# update everything (security CVE patches)
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove -y

# locale, timezone (BD এর জন্য Asia/Dhaka)
sudo timedatectl set-timezone Asia/Dhaka
sudo dpkg-reconfigure locales  # en_US.UTF-8 generate

# hostname meaningful রাখুন
sudo hostnamectl set-hostname app-prod-01.daraz.com.bd

# /etc/hosts এ FQDN
echo "127.0.1.1 app-prod-01.daraz.com.bd app-prod-01" | sudo tee -a /etc/hosts

# essentials install
sudo apt install -y curl wget git vim htop iotop net-tools \
    ca-certificates gnupg lsb-release software-properties-common \
    unattended-upgrades fail2ban ufw
```

### SSH hardening — সবচেয়ে গুরুত্বপূর্ণ ধাপ

**Key-based auth setup (local machine থেকে):**

```bash
# local এ key generate (যদি না থাকে)
ssh-keygen -t ed25519 -C "yourname@laptop" -f ~/.ssh/id_ed25519

# server এ copy
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<server-ip>
# (or)
cat ~/.ssh/id_ed25519.pub | ssh root@<ip> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**`/etc/ssh/sshd_config` এ:**

```
# Port — 22 থেকে অন্য কিছু (debate: security through obscurity vs fail2ban যথেষ্ট)
Port 2222

# Root login disable
PermitRootLogin no

# Password auth disable, key only
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes

# Empty password নিষেধ
PermitEmptyPasswords no

# Protocol 2 only
Protocol 2

# Idle timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# কে SSH করতে পারবে
AllowUsers deploy admin
# AllowGroups ssh-users  # (group-based)

# X11 forwarding দরকার নেই server এ
X11Forwarding no

# Banner — legal warning
Banner /etc/issue.net

# Strict modes
StrictModes yes
MaxAuthTries 3
LoginGraceTime 30s
```

```bash
# /etc/issue.net
echo "*** Authorized Access Only ***
Disconnect IMMEDIATELY if you are not an authorized user.
All actions logged and monitored." | sudo tee /etc/issue.net

# config validate before reload
sudo sshd -t

# reload (নতুন session keep থাকবে)
sudo systemctl reload ssh
```

> ⚠️ **নতুন port এ test করুন আগে current session বন্ধ করার আগে।** নতুন terminal খুলে `ssh -p 2222 deploy@server` confirm করুন।

### Port-change debate

```
✅ Port-change করার পক্ষে:
- Bot scan attempt log noise কমে (90% bot port 22 try করে)
- fail2ban এর load কমে

❌ বিপক্ষে:
- Real security নয় — nmap সব port scan করে
- Audit/compliance team confused
- Default port-blocking firewall এ ঝামেলা

বাস্তব pragmatic: Port 22 রাখুন, কিন্তু fail2ban + key-only + ufw কঠোর।
Daraz/Pathao এর scale এ সাধারণত port 22 ই থাকে, behind a bastion host।
```

---

## 2. 👤 User & permissions

```bash
# deploy user create (CI/CD এর জন্য)
sudo adduser deploy --disabled-password
sudo usermod -aG sudo deploy

# SSH key copy
sudo mkdir -p /home/deploy/.ssh
sudo cp ~/.ssh/authorized_keys /home/deploy/.ssh/
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys

# admin user (interactive sudo)
sudo adduser admin
sudo usermod -aG sudo admin
```

### Passwordless sudo for CI (limited)

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

```
# শুধু নির্দিষ্ট command এ passwordless — full NOPASSWD avoid করুন
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app, \
                           /usr/bin/systemctl reload nginx, \
                           /usr/bin/docker, \
                           /usr/bin/docker-compose
```

### File permissions hygiene

```
ক্লাসিক rule:
  Directory: 755 (drwxr-xr-x)
  File:      644 (-rw-r--r--)
  Secret:    600 (-rw-------)
  Executable:755

malicious upload-able location (uploads/) → never executable
```

```bash
# web root permission
sudo chown -R www-data:www-data /var/www/app
sudo find /var/www/app -type d -exec chmod 755 {} \;
sudo find /var/www/app -type f -exec chmod 644 {} \;

# Laravel storage/bootstrap-cache writable
sudo chmod -R 775 /var/www/app/storage /var/www/app/bootstrap/cache
sudo chown -R deploy:www-data /var/www/app/storage /var/www/app/bootstrap/cache

# .env protect
sudo chmod 600 /var/www/app/.env
sudo chown deploy:www-data /var/www/app/.env
```

---

## 3. 🔥 Firewall — ufw, iptables, fail2ban

### ufw (Ubuntu/Debian — সহজ)

```bash
# default deny incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# essentials allow
sudo ufw allow 2222/tcp comment 'SSH (custom port)'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# rate limit SSH (brute force suppress)
sudo ufw limit 2222/tcp

# specific IP only — admin panel
sudo ufw allow from 203.0.113.45 to any port 9090 comment 'Prometheus from office'

# enable
sudo ufw enable
sudo ufw status numbered
```

### firewalld (RHEL/Alma/Rocky)

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --remove-service=ssh   # default 22
sudo firewall-cmd --reload
```

### iptables raw (advanced — DDoS rule)

```bash
# SYN flood protection
sudo iptables -A INPUT -p tcp --syn -m limit --limit 25/s --limit-burst 50 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# ICMP flood
sudo iptables -A INPUT -p icmp -m limit --limit 1/s -j ACCEPT
sudo iptables -A INPUT -p icmp -j DROP

# persist
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### fail2ban — brute force auto-ban

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

`/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd
ignoreip = 127.0.0.1/8 ::1 203.0.113.45  # office IP

[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s

[nginx-http-auth]
enabled = true
port    = http,https
logpath = /var/log/nginx/error.log

[nginx-botsearch]
enabled = true
port    = http,https
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-limit-req]
enabled = true
port    = http,https
logpath = /var/log/nginx/error.log
maxretry = 10
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
sudo fail2ban-client unban <IP>  # mistake হলে
```

### CrowdSec — fail2ban এর modern replacement

```bash
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec
sudo cscli collections install crowdsecurity/nginx
sudo cscli decisions list
```

> CrowdSec community blocklist share করে — অন্য server যে IP attack করেছে আপনি auto-block পাবেন।

---

## 4. 🎚️ System tuning — sysctl, ulimit, swap

### Swap — কখন ও কত

```
নিয়ম:
- < 2GB RAM   → 2× RAM swap
- 2-8GB RAM   → equal swap
- > 8GB RAM   → 4-8GB swap (just safety net)
- DB server   → swap minimum (vm.swappiness=1), DB-এর জন্য RAM dedicated
- Dedicated cache (Redis) → no swap, OOM-killer better
```

```bash
# swap file create
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# swappiness (10 = swap কম, 60 = default)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-tune.conf
sudo sysctl -p /etc/sysctl.d/99-tune.conf
```

### `/etc/sysctl.d/99-server-tune.conf`

```conf
# ── Network ─────────────────────────────────────
# একসাথে কত pending connection accept করবে (Nginx high-traffic এর জন্য)
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535

# ephemeral port range (outgoing connection limit বাড়ায়)
net.ipv4.ip_local_port_range = 1024 65535

# TIME_WAIT socket reuse
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15

# SYN flood প্রতিরোধ
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_syncookies = 1

# Buffer tune (high-bandwidth)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Keepalive (idle connection cleanup)
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 5

# ── Memory / VM ─────────────────────────────────
vm.swappiness = 10
vm.vfs_cache_pressure = 50
vm.overcommit_memory = 1   # Redis/Postgres এর জন্য recommended

# ── File system ────────────────────────────────
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288  # node.js dev tools

# ── Security ────────────────────────────────────
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.log_martians = 1
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
```

```bash
sudo sysctl -p /etc/sysctl.d/99-server-tune.conf
```

> বিস্তারিত — `20-operating-systems/` (kernel/scheduling docs) দেখুন।

### ulimit / systemd LimitNOFILE

```bash
# /etc/security/limits.conf
sudo tee -a /etc/security/limits.conf <<EOF
*        soft    nofile   65535
*        hard    nofile   1048576
*        soft    nproc    65535
*        hard    nproc    65535
www-data soft    nofile   65535
www-data hard    nofile   1048576
EOF

# /etc/pam.d/common-session এ (Ubuntu)
echo 'session required pam_limits.so' | sudo tee -a /etc/pam.d/common-session
```

systemd service file এ:

```ini
[Service]
LimitNOFILE=1048576
LimitNPROC=65535
```

```bash
# verify
ulimit -n   # current shell
cat /proc/$(pidof nginx | awk '{print $1}')/limits | grep "Max open files"
```

---

## 5. 🌐 Web server — Nginx/Apache + TLS

### Nginx install

```bash
# official mainline (apt default পুরনো)
curl https://nginx.org/keys/nginx_signing.key | sudo gpg --dearmor -o /usr/share/keyrings/nginx-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-keyring.gpg] http://nginx.org/packages/mainline/ubuntu $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
sudo apt update && sudo apt install nginx
sudo systemctl enable --now nginx
```

### vhost config — `/etc/nginx/conf.d/app.conf`

```nginx
upstream php_app {
    server unix:/run/php/php8.3-fpm.sock;
    keepalive 32;
}

server {
    listen 80;
    server_name app.daraz.com.bd;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name app.daraz.com.bd;
    root /var/www/app/public;
    index index.php;

    # TLS
    ssl_certificate     /etc/letsencrypt/live/app.daraz.com.bd/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.daraz.com.bd/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.google-analytics.com" always;

    # gzip
    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

    # rate limit (zone defined in nginx.conf)
    limit_req zone=app_zone burst=50 nodelay;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass php_app;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 60s;
        fastcgi_buffers 16 16k;
    }

    location ~ /\.(ht|env|git) { deny all; }
    location = /favicon.ico   { access_log off; log_not_found off; }
}
```

`/etc/nginx/nginx.conf` এ:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=app_zone:10m rate=10r/s;
    client_max_body_size 50M;
    keepalive_timeout 65;
    server_tokens off;
    ...
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

> বিস্তারিত — `nginx-deep-dive.md`। TLS provisioning — `ssl-certbot.md`।

### Apache (যদি .htaccess legacy app থাকে)

```bash
sudo apt install apache2 libapache2-mod-php8.3
sudo a2enmod rewrite headers ssl http2
sudo a2dissite 000-default
```

> বিস্তারিত — `apache-deep-dive.md`।

---

## 6. 🚀 Application runtime — PHP-FPM, Node.js, Python

### PHP-FPM tuning

`/etc/php/8.3/fpm/pool.d/www.conf`:

```ini
user = www-data
group = www-data
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; ─── pm mode ──────────────────────────────
; static     — সব worker সবসময় চালু (max RPS, max RAM)
; dynamic    — min থেকে max এ scale (default, balance)
; ondemand   — request এলে worker spawn (সবচেয়ে কম RAM, latency দাম)
pm = dynamic

; ─── max_children নির্ধারণ ────────────────
; available RAM / per-worker memory
; e.g., 4GB available, প্রতি worker ~50MB → 4096/50 = 80
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
pm.max_requests = 500   ; memory leak guard

; ─── timeouts ────────────────────────────
request_terminate_timeout = 60s
request_slowlog_timeout = 10s
slowlog = /var/log/php-fpm-slow.log

; status page (Prometheus exporter এর জন্য)
pm.status_path = /fpm-status
ping.path = /fpm-ping
```

**max_children calculation:**

```
Step 1: per-worker memory measure
  ps -ylC php-fpm8.3 --sort:rss | awk '{sum+=$8; n++} END {print sum/n " KB avg"}'
  ধরা যাক 60MB avg

Step 2: available RAM (OS + Nginx + Redis বাদ দিয়ে)
  4GB - (OS 500MB + Nginx 100MB + Redis 200MB + buffer 500MB) = 2.7GB

Step 3: max_children = 2700 / 60 = 45

Margin রেখে ৪০ set করুন।
```

### Node.js with PM2

```bash
sudo npm install -g pm2

# ecosystem.config.js
cat > /var/www/app/ecosystem.config.js <<'EOF'
module.exports = {
  apps: [{
    name: 'api',
    script: './server.js',
    instances: 'max',          // CPU core count
    exec_mode: 'cluster',      // load-balanced
    max_memory_restart: '512M',
    env: { NODE_ENV: 'production', PORT: 3000 },
    error_file: '/var/log/pm2/api-err.log',
    out_file: '/var/log/pm2/api-out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
  }]
};
EOF

pm2 start ecosystem.config.js
pm2 save
pm2 startup systemd -u deploy --hp /home/deploy
```

### Node.js with systemd (no PM2)

`/etc/systemd/system/api.service`:

```ini
[Unit]
Description=API Node.js Service
After=network.target

[Service]
Type=simple
User=deploy
Group=www-data
WorkingDirectory=/var/www/app
Environment=NODE_ENV=production
Environment=PORT=3000
EnvironmentFile=/var/www/app/.env
ExecStart=/usr/bin/node /var/www/app/server.js
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=api
LimitNOFILE=65535

# security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/www/app/storage /var/log/app
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now api
sudo journalctl -u api -f
```

### Python + Gunicorn + systemd

```bash
sudo apt install python3.12 python3.12-venv
sudo -u deploy bash <<'EOF'
cd /var/www/app
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt gunicorn
EOF
```

`/etc/systemd/system/django.service`:

```ini
[Unit]
Description=Django via gunicorn
After=network.target

[Service]
User=deploy
Group=www-data
WorkingDirectory=/var/www/app
Environment="PATH=/var/www/app/.venv/bin"
EnvironmentFile=/var/www/app/.env
ExecStart=/var/www/app/.venv/bin/gunicorn \
    --workers 4 \
    --threads 2 \
    --timeout 60 \
    --access-logfile - \
    --bind unix:/run/django.sock \
    config.wsgi:application
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Worker count: `2 × CPU + 1` (Gunicorn doc)। CPU-bound হলে threads বাড়ান, IO-bound হলে workers।

---

## 7. 🗄️ Database setup

### Local install vs DBaaS

| | Local install | DBaaS (RDS/Neon) |
|--|---------------|------------------|
| Cost | $15-30/mo (VPS) | $50-200+/mo |
| Backup | Manual (cron) | Auto, point-in-time |
| HA/Failover | Manual setup | Auto multi-AZ |
| Patch | Your job | Provider |
| Latency | Same VM = <1ms | Cross-VPC = 1-5ms |
| BD-fit | ✅ MVP/early | scaling phase |

### MySQL 8 install + secure

```bash
sudo apt install mysql-server-8.0
sudo mysql_secure_installation   # root password, anon user remove, test DB drop

# bind only localhost (or VPC)
sudo sed -i 's/^bind-address.*/bind-address = 127.0.0.1/' /etc/mysql/mysql.conf.d/mysqld.cnf

# dedicated app user
sudo mysql <<EOF
CREATE DATABASE app_prod CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'app'@'localhost' IDENTIFIED BY 'STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON app_prod.* TO 'app'@'localhost';
FLUSH PRIVILEGES;
EOF

sudo systemctl enable --now mysql
```

### PostgreSQL 16

```bash
sudo apt install postgresql-16 postgresql-contrib

# user + db
sudo -u postgres psql <<EOF
CREATE USER app WITH ENCRYPTED PASSWORD 'STRONG_PW';
CREATE DATABASE app_prod OWNER app;
EOF

# /etc/postgresql/16/main/pg_hba.conf — local md5 only
# /etc/postgresql/16/main/postgresql.conf:
#   listen_addresses = 'localhost'
#   shared_buffers = 25% RAM
#   effective_cache_size = 50-75% RAM
#   work_mem = 16MB
#   maintenance_work_mem = 256MB
#   max_connections = 200 (PgBouncer pool করুন above)
sudo systemctl restart postgresql
```

### Backup — logical + physical

```bash
# logical (mysqldump)
mysqldump --single-transaction --quick --routines --triggers app_prod | \
  gzip > /backup/mysql/app_prod-$(date +%F).sql.gz

# physical (Percona XtraBackup) — large DB এর জন্য faster
sudo apt install percona-xtrabackup-80
xtrabackup --backup --target-dir=/backup/xb-$(date +%F) --user=root --password=...

# Postgres
pg_dump -Fc app_prod > /backup/pg/app_prod-$(date +%F).dump

# binlog enable (point-in-time recovery)
# /etc/mysql/my.cnf:
#   log_bin = /var/log/mysql/mysql-bin.log
#   binlog_format = ROW
#   expire_logs_days = 7
```

### Replication hint (master-replica)

```sql
-- master
SHOW MASTER STATUS;  -- file, position note

-- replica (slave)
CHANGE MASTER TO
  MASTER_HOST='10.0.0.5',
  MASTER_USER='replica',
  MASTER_PASSWORD='...',
  MASTER_LOG_FILE='mysql-bin.000123',
  MASTER_LOG_POS=4567;
START SLAVE;
SHOW SLAVE STATUS\G  -- Seconds_Behind_Master = 0 ভালো
```

---

## 8. 🔄 CI/CD deployment

### GitHub Actions — SSH + rsync deploy

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with: { php-version: '8.3' }

      - name: Composer install (no dev)
        run: composer install --no-dev --optimize-autoloader --prefer-dist

      - name: Build assets
        run: |
          npm ci
          npm run build

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -p 2222 ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

      - name: Rsync to server
        run: |
          rsync -avz --delete \
            --exclude='.git' --exclude='.env' --exclude='storage/logs' \
            -e "ssh -p 2222 -i ~/.ssh/id_ed25519" \
            ./ deploy@${{ secrets.SERVER_HOST }}:/var/www/app/

      - name: Post-deploy
        run: |
          ssh -p 2222 deploy@${{ secrets.SERVER_HOST }} bash -s <<'EOF'
            cd /var/www/app
            php artisan migrate --force
            php artisan config:cache route:cache view:cache
            sudo systemctl reload php8.3-fpm
            sudo systemctl reload nginx
          EOF
```

### Docker pull deploy (image-based)

```yaml
- name: Build & push image
  run: |
    docker build -t registry.gitlab.com/myorg/app:${{ github.sha }} .
    echo ${{ secrets.REGISTRY_TOKEN }} | docker login registry.gitlab.com -u CI --password-stdin
    docker push registry.gitlab.com/myorg/app:${{ github.sha }}

- name: Deploy on server
  run: |
    ssh deploy@host "
      docker pull registry.gitlab.com/myorg/app:${{ github.sha }}
      docker compose -f /opt/app/docker-compose.yml up -d
    "
```

### Blue-green via two systemd units + Nginx upstream

```ini
# /etc/systemd/system/api-blue.service  (port 3001)
# /etc/systemd/system/api-green.service (port 3002)
```

```nginx
# /etc/nginx/conf.d/upstream-active.conf
upstream api_active { server 127.0.0.1:3001; }   # currently blue
```

Switch script:

```bash
#!/bin/bash
# /usr/local/bin/blue-green-switch.sh
set -e
CURRENT=$(grep -oP '300[12]' /etc/nginx/conf.d/upstream-active.conf)
NEW=$([ "$CURRENT" = "3001" ] && echo 3002 || echo 3001)
NEW_COLOR=$([ "$NEW" = "3001" ] && echo blue || echo green)

systemctl restart api-${NEW_COLOR}
sleep 5

# health check
for i in {1..10}; do
  curl -sf http://127.0.0.1:${NEW}/health && break
  sleep 2
done

sed -i "s/300[12]/${NEW}/" /etc/nginx/conf.d/upstream-active.conf
nginx -t && systemctl reload nginx
echo "Switched to ${NEW_COLOR} (port ${NEW})"
```

> বিস্তারিত blue-green / canary — `deployment-strategies.md`। CI/CD pipeline pattern — `ci-cd.md`।

---

## 9. 🐳 Containerized deployment

### Production docker-compose pattern

`/opt/app/docker-compose.prod.yml`:

```yaml
services:
  app:
    image: registry.gitlab.com/org/app:v1.2.3
    restart: unless-stopped
    env_file: /opt/app/.env
    ports: ["127.0.0.1:8080:8080"]   # bind localhost only, Nginx আগে
    volumes:
      - ./storage:/app/storage
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits: { memory: 512M, cpus: '1.0' }
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets: [db_password]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

secrets:
  db_password:
    file: /opt/app/secrets/db_password.txt

volumes:
  pgdata:
```

### Watchtower — auto-update?

```yaml
# avoid in production unless tested!
watchtower:
  image: containrrr/watchtower
  command: --interval 3600 --cleanup
```

> ❌ **production এ watchtower নয়** — uncontrolled image update = surprise outage। Tag-pinned, manual rollout করুন।

---

## 10. 📊 Monitoring & logging

### Prometheus + node_exporter + Grafana

```bash
# node_exporter (host metrics)
useradd -rs /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/.../node_exporter-1.8.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
mv node_exporter-*/node_exporter /usr/local/bin/

cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Node Exporter
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now node_exporter

# ufw — only Prometheus server থেকে allow
ufw allow from 10.0.0.5 to any port 9100
```

`/etc/prometheus/prometheus.yml` (central server এ):

```yaml
scrape_configs:
  - job_name: 'nodes'
    static_configs:
      - targets:
          - 'app-prod-01.internal:9100'
          - 'db-prod-01.internal:9100'
  - job_name: 'nginx'
    static_configs: [{ targets: ['app-prod-01:9113'] }]
  - job_name: 'php-fpm'
    static_configs: [{ targets: ['app-prod-01:9253'] }]
```

### Grafana dashboard imports

- 1860 — Node Exporter Full
- 12708 — Nginx Prometheus Exporter
- 4701 — PHP-FPM
- 763 — Redis

### Loki — log aggregation

```yaml
# docker-compose loki.yml
loki:
  image: grafana/loki:2.9.0
  ports: ["3100:3100"]

promtail:
  image: grafana/promtail:2.9.0
  volumes:
    - /var/log:/var/log
    - ./promtail.yml:/etc/promtail/config.yml
```

### Alertmanager — alert routing

```yaml
route:
  receiver: discord-bd-team
receivers:
  - name: discord-bd-team
    discord_configs:
      - webhook_url: 'https://discord.com/api/webhooks/...'
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: ...
```

### Lightweight alternatives (BD startup-friendly)

| Tool | কী | $? |
|------|-----|------|
| **Netdata** | Real-time, zero-config, beautiful UI | Free / $$ Cloud |
| **Uptime Kuma** | Self-hosted UptimeRobot | Free, ১ Droplet |
| **Sentry** | Error tracking | Free 5K events/mo |
| **Better Stack** | Logs + uptime + heartbeat | Free 10GB logs |

> বিস্তারিত — `monitoring.md`।

---

## 11. 💾 Backup & DR

### Strategy — 3-2-1 rule

```
3 copies of data, on 2 different media, 1 offsite
```

### Snapshot — DigitalOcean

```bash
# DO API দিয়ে scheduled snapshot
doctl compute droplet-action snapshot <droplet-id> --snapshot-name "app-$(date +%F)"
# automated weekly snapshot $0.05/GB-mo
```

### AWS EBS

```bash
aws ec2 create-snapshot --volume-id vol-xxx --description "daily-$(date +%F)"
# Lifecycle Manager (DLM) — automate retention
```

### restic — encrypted offsite to S3/B2

```bash
sudo apt install restic

# initialize
export RESTIC_REPOSITORY=s3:s3.amazonaws.com/bucket/server01
export RESTIC_PASSWORD_FILE=/root/.restic-pw
restic init

# backup
restic backup /var/www /etc /home --exclude='*.log' --exclude='cache'

# retention
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune

# restore
restic restore latest --target /tmp/restore --include /etc/nginx
```

cron:

```cron
0 3 * * * /usr/local/bin/restic-backup.sh >> /var/log/restic.log 2>&1
0 4 * * 0 restic check  # weekly integrity check
```

### MySQL binlog continuous backup

```bash
# binlog file ship to S3 every minute
* * * * * /usr/local/bin/binlog-ship.sh

# DR scenario: full backup + binlog replay
mysql < full-backup.sql
mysqlbinlog mysql-bin.000123 | mysql
```

### Backup monitoring — must do!

```bash
# healthchecks.io ping after success
restic backup ... && curl https://hc-ping.com/UUID
# fail হলে — alert
```

> ⚠️ "Untested backup is no backup" — তিন মাসে অন্তত একবার full restore test করুন।

---

## 12. 🔐 Security hardening (advanced)

### Disable unused services

```bash
systemctl list-unit-files --state=enabled
# unused candidates: cups, avahi-daemon, snapd (যদি না লাগে)
sudo systemctl disable --now cups avahi-daemon
sudo apt purge cups* avahi-daemon
```

### Automatic security updates — unattended-upgrades

```bash
sudo apt install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

`/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
Unattended-Upgrade::Mail "ops@daraz.com.bd";
```

### File Integrity Monitoring — AIDE

```bash
sudo apt install aide
sudo aideinit
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# daily check
echo "0 5 * * * root /usr/bin/aide --check | mail -s 'AIDE report' ops@..." > /etc/cron.d/aide
```

### Audit logging — auditd

```bash
sudo apt install auditd
sudo systemctl enable --now auditd

# /etc/audit/rules.d/audit.rules
echo "
-w /etc/passwd -p wa -k user_change
-w /etc/shadow -p wa -k user_change
-w /etc/sudoers -p wa -k sudoers
-w /var/log/sudo.log -p wa
-a always,exit -F arch=b64 -S execve -k exec_log
" | sudo tee -a /etc/audit/rules.d/audit.rules
sudo augenrules --load
```

### AppArmor (Ubuntu/Debian)

```bash
sudo aa-status
# nginx, mysql সবসময় enforce mode এ থাকা উচিত
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

### SELinux (RHEL/Alma) — enforcing রাখুন

```bash
sestatus
# /etc/selinux/config: SELINUX=enforcing
setenforce 1

# nginx custom port (2222) এ চলতে দিন
semanage port -a -t ssh_port_t -p tcp 2222
```

### Secrets — SOPS + age

```bash
# encrypt .env-style file with age key
age-keygen -o ~/.config/sops/age/keys.txt
sops --age <pubkey> -e secrets.yml > secrets.enc.yml
# git এ commit safe; deploy এ decrypt
sops -d secrets.enc.yml > /opt/app/.env
```

> বিস্তারিত — `secrets-management.md`।

### CrowdSec community blocklist — already covered in §3

---

## 13. 🪵 Logging & rotation

### journald

```ini
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=2G
MaxRetentionSec=30day
ForwardToSyslog=no
Compress=yes
```

### logrotate

`/etc/logrotate.d/app`:

```
/var/log/app/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 deploy www-data
    sharedscripts
    postrotate
        systemctl reload api > /dev/null 2>&1 || true
    endscript
}
```

### Central syslog/Loki

```bash
# rsyslog forward to loki
echo '*.* @@loki.internal:514' > /etc/rsyslog.d/30-loki.conf
systemctl restart rsyslog
```

> ELK/EFK central logging — `16-observability/elk-efk-stack.md` (যখন আসবে)।

---

## 14. 🌍 CDN & WAF — Cloudflare

### Setup steps

1. Domain → Cloudflare add → Nameserver point
2. SSL/TLS mode → **Full (Strict)** — origin এ valid cert (Let's Encrypt) থাকা মাস্ট
3. **Always Use HTTPS** on
4. Min TLS version: 1.2
5. **Bot Fight Mode** on (free)
6. **WAF Managed Rules** — OWASP set enable
7. **Page Rules** — `/admin/*` → Security level: High, Cache: Bypass
8. **Rate Limiting** — `/login` 5 req/min per IP

### Origin protection

```nginx
# Nginx এ — শুধু Cloudflare IP allow (origin pull)
include /etc/nginx/cloudflare-real-ip.conf;
real_ip_header CF-Connecting-IP;

# UFW দিয়ে Cloudflare IP only
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  ufw allow from $ip to any port 443 proto tcp
done
ufw deny 443
```

> bKash, Daraz — Cloudflare এর Dhaka POP থেকে static asset serve করে, latency <10ms।

---

## 15. 💰 Cost optimization

```
Rightsizing (every 3 months):
  - CPU < 30% avg → downsize
  - Memory < 50% → downsize
  - Disk < 60% → don't oversize

Reserved Instance:
  - 1-year RI = ৪০% off
  - 3-year RI = ৬০% off
  - স্থিতিশীল workload এ মাস্ট

Spot/Preemptible (batch):
  - CI runner, ML training, video transcode
  - ৭০-৯০% সস্তা
  - 2-min warning এ shutdown handle

Cleanup:
  - Detached EBS volume = $$$
  - Old snapshot prune
  - Unused Elastic IP ($3.6/mo idle)
  - Empty load balancer ($20/mo idle)

CDN:
  - Cloudflare free → ৯০% bandwidth সাশ্রয়
  - S3 → CloudFront origin → no egress charge from S3
```

---

## 16. 🇧🇩 BD-specific tips

### BDIX peering

BDIX (Bangladesh Internet Exchange) — local ISP দের traffic দেশের ভিতরে রাখে, latency কম। যদি আপনার ইউজার ৯০% BD হয়:

✅ **Eicra, ExonHost, ICC, Mazeda** — BDIX-peered, GP/Robi/BL user দের জন্য <10ms
✅ Static asset — **Cloudflare (Dhaka POP)** আছে, BDIX-routed
❌ AWS Mumbai — fastlink (international) থেকে আসে, ~40-60ms

### Region choice — Mumbai vs Singapore

| | AWS Mumbai (ap-south-1) | AWS Singapore (ap-southeast-1) | DO Singapore |
|--|------------------------|-------------------------------|--------------|
| BD latency | ৪০-৬০ms | ৭০-৯০ms | ৭০-৯০ms |
| Compliance | Closer to BD law | Closer to ASEAN | — |
| Service availability | Most | All | Limited |
| Cost | similar | similar | flat-cheap |

> **নিয়ম:** B2C BD app → Mumbai। B2B/SEA expansion → Singapore।

### Mobile data spec (GP/Robi/BL)

- 4G average DL 15-25 Mbps, peak 50
- Latency 30-80ms (4G), 100-200ms (3G fallback)
- Image-heavy app — WebP/AVIF, lazy load critical
- Daraz প্রোডাক্ট image — 50KB target

### Cloudflare Dhaka POP

✅ static cache <5ms
✅ argo smart routing — origin (Mumbai) এও latency কমে
✅ Workers-at-edge — Bangladesh user থেকে closest worker run

---

## 17. ✅ Production deployment checklist (50+)

### Pre-launch

- [ ] OS LTS, fully patched
- [ ] Timezone Asia/Dhaka, NTP active
- [ ] Hostname FQDN set
- [ ] SSH: key-only, root disabled, port custom
- [ ] Non-root deploy user with limited sudo
- [ ] UFW/firewalld: deny incoming default, only 22(custom)/80/443 open
- [ ] fail2ban / CrowdSec active
- [ ] Swap sized appropriately
- [ ] sysctl tuning applied (somaxconn, file-max)
- [ ] ulimit / LimitNOFILE = 65535+
- [ ] unattended-upgrades enabled (security only)
- [ ] /etc/issue.net legal banner

### Web server

- [ ] Nginx mainline (or Apache 2.4+)
- [ ] HTTP → HTTPS 301
- [ ] TLS 1.2/1.3 only, modern ciphers
- [ ] Let's Encrypt + auto-renewal
- [ ] OCSP stapling
- [ ] HSTS preload header
- [ ] CSP, X-Frame-Options, X-Content-Type-Options
- [ ] Rate limit configured
- [ ] gzip/brotli
- [ ] server_tokens off
- [ ] Hidden file deny (.git, .env)

### Application

- [ ] PHP-FPM/Node/Gunicorn under systemd
- [ ] max_children/workers calculated, not default
- [ ] OPcache (PHP) / preloading enabled
- [ ] Worker memory limit + restart policy
- [ ] .env file 600 perm, owned by deploy
- [ ] Symlink-based release (current → releases/timestamp)
- [ ] Database migrations idempotent
- [ ] Cache warming post-deploy

### Database

- [ ] Bind localhost only (or VPC private)
- [ ] Strong root password, app user least-privilege
- [ ] Backup automated (logical + physical)
- [ ] Backup tested (restore drill within 90 days)
- [ ] Binlog/WAL enabled (PITR)
- [ ] Connection pooler (PgBouncer/ProxySQL)
- [ ] Slow query log on
- [ ] Replica/HA strategy

### Monitoring

- [ ] node_exporter + Prometheus scrape
- [ ] Grafana dashboard imported
- [ ] Alertmanager → Discord/PagerDuty
- [ ] Uptime check (Uptime Kuma / Better Stack)
- [ ] Sentry/error tracker integrated
- [ ] Log shipping (Loki/ELK) or at minimum journald + logrotate

### Backup & DR

- [ ] Daily snapshot (DO/EBS)
- [ ] Offsite encrypted backup (restic → S3/B2)
- [ ] Backup retention policy (7d/4w/6m)
- [ ] Restore tested
- [ ] DR runbook documented

### Security

- [ ] Secrets in SOPS/Vault, not git
- [ ] AppArmor/SELinux enforcing
- [ ] AIDE/auditd installed
- [ ] CrowdSec / fail2ban active
- [ ] Cloudflare WAF / Bot Fight on
- [ ] CDN origin-IP hidden, only CF allow
- [ ] No unused service running
- [ ] Periodic CVE scan (trivy, grype)

### Deployment

- [ ] CI/CD pipeline green
- [ ] Zero-downtime deploy verified (graceful reload)
- [ ] Rollback procedure tested
- [ ] Health endpoint `/health` exists
- [ ] DB migration `--force` only in CI

### Cost

- [ ] Billing alert set
- [ ] Right-sized instance
- [ ] Reserved/Savings plan (if predictable)
- [ ] Unused resource cleanup cron

### BD-specific

- [ ] Region: Mumbai (B2C BD) or local DC (regulated)
- [ ] Cloudflare Dhaka POP CDN
- [ ] BDIX peering (if local DC)
- [ ] Mobile-optimized images (WebP)
- [ ] payment gateway (SSLCommerz/bKash/ShurjoPay) tested

---

## 📚 আরো পড়ুন

- `nginx-deep-dive.md`, `apache-deep-dive.md` — web server tuning
- `ssl-certbot.md` — TLS automation
- `docker.md`, `kubernetes.md` — container orchestration
- `deployment-strategies.md` — blue-green, canary
- `ci-cd.md` — pipeline patterns
- `monitoring.md` — observability stack
- `secrets-management.md` — Vault/SOPS
- `saas-paas-iaas.md` — cloud service model choice
- `14-networking/tls-ssl.md` — TLS theory
- `20-operating-systems/` — kernel/sysctl reference

---

> **শেষ কথা:** Production server hardening একদিনের কাজ নয় — এটি একটি **ongoing discipline**। প্রতি deploy এর পর checklist verify, প্রতি মাসে cost-review, প্রতি quarter এ DR drill। Daraz এর Black Friday বা bKash এর Eid bonus surge — এই সময় যেগুলো সচল রাখে, সেগুলো এই checklist থেকেই আসে।
