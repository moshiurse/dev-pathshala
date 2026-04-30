# 🔐 SSL/Certbot — TLS Automation Operations Guide

> **TLS theory `14-networking/tls-ssl.md` এ আছে — এই ফাইল কেবল operational/automation দিকের।**
> Daraz এর ১০০+ subdomain, bKash এর mTLS partner gateway, Pathao API endpoint — সবগুলোর cert manual issue/renew অসম্ভব। **ACME + Certbot + cert-manager** ই production scale এর উত্তর।

---

## 📖 সূচিপত্র

- [ACME Protocol Overview](#-acme-protocol-overview-rfc-8555)
- [Challenge Types — HTTP-01, DNS-01, TLS-ALPN-01](#-challenge-types)
- [Certbot — Install & First Cert](#-certbot--install--first-cert)
- [Wildcard Cert (DNS-01)](#-wildcard-certificate-dns-01)
- [Auto-renewal](#-auto-renewal)
- [Alternatives — acme.sh, lego, Caddy, Traefik](#-alternatives)
- [Multi-server Architecture](#-multi-server-architecture)
- [Kubernetes — cert-manager](#-kubernetes--cert-manager)
- [Docker — Nginx + certbot Sidecar](#-docker--nginx--certbot-sidecar)
- [Free CAs Comparison](#-free-cas-comparison)
- [mTLS with Internal CA](#-mtls-with-internal-ca)
- [Operational Concerns](#-operational-concerns)
- [Common Errors](#-common-errors--troubleshooting)
- [BD Context](#-bd-context)
- [Full Production Example](#-full-nginx--certbot-production-example)
- [Checklist](#-checklist)

---

## 📜 ACME Protocol Overview (RFC 8555)

ACME (Automatic Certificate Management Environment) — Let's Encrypt আবিষ্কৃত একটি প্রটোকল যা CA-client দের মাঝে certificate issuance automate করে।

```
┌─────────────────────────────────────────────────────────────────┐
│                  ACME flow (issuance)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client (certbot)                       ACME Server (LE)       │
│        │                                       │                │
│        │ 1. POST /new-account (with key)       │                │
│        │──────────────────────────────────────►│                │
│        │ ◄────────────── 201 Account ──────────│                │
│        │                                       │                │
│        │ 2. POST /new-order                    │                │
│        │   identifiers: [daraz.com.bd]         │                │
│        │──────────────────────────────────────►│                │
│        │ ◄─── Order: pending, authorizations ──│                │
│        │                                       │                │
│        │ 3. GET /authz/:id (challenge options) │                │
│        │──────────────────────────────────────►│                │
│        │ ◄── http-01, dns-01, tls-alpn-01 ─────│                │
│        │                                       │                │
│        │ 4. Prepare proof (file/DNS record)    │                │
│        │ 5. POST /challenge (ready to verify)  │                │
│        │──────────────────────────────────────►│                │
│        │                                       │                │
│        │       6. Server validates ◄──── HTTP/DNS check ──┐    │
│        │                                       │           │    │
│        │ ◄────── status: valid ────────────────│           ▼    │
│        │                              আপনার server / DNS         │
│        │ 7. Finalize (CSR submit)              │                │
│        │──────────────────────────────────────►│                │
│        │                                       │                │
│        │ 8. GET /cert (download)               │                │
│        │──────────────────────────────────────►│                │
│        │ ◄────── cert.pem + chain ─────────────│                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

State machine:
  new-order  →  pending  →  ready  →  processing  →  valid
                    ↑                                  │
                    └─── (challenge fail/expire) ──────┘ → invalid
```

---

## 🎯 Challenge Types

### HTTP-01

```
CA:    "your-domain.com এর `/.well-known/acme-challenge/<token>` এ
        এই string রাখো: `<token>.<thumbprint>`"

Client: file create on web server

CA:    HTTP GET http://your-domain.com/.well-known/acme-challenge/<token>
       expected response পেলে → valid
```

✅ **কখন:** standard public web server, port 80 খোলা
❌ **কাজ করে না:** wildcard cert (`*.daraz.com.bd`), private/internal endpoint, port 80 firewall-blocked

### DNS-01

```
CA:     "_acme-challenge.your-domain.com এ এই TXT record রাখো"
Client: DNS API দিয়ে TXT record create
CA:     DNS query করে TXT verify
```

✅ **কখন:** wildcard (`*.example.com`), private endpoint, port 80 closed
✅ একসাথে multiple SAN (e.g., `*.daraz.com.bd, daraz.com.bd, api.daraz.com.bd`)
❌ **কষ্ট:** DNS provider এ API লাগে; manual DNS এ automation হয় না

### TLS-ALPN-01

```
CA:     "port 443 এ ALPN protocol acme-tls/1 দিয়ে handshake করব,
         self-signed cert এ token-based extension থাকতে হবে"
Client: short-lived cert serve on 443 with ALPN
CA:     TLS handshake করে verify
```

✅ **কখন:** port 80 closed কিন্তু 443 খোলা
✅ Sidecar/inline proxy এ practical
❌ Apache/Nginx কে momentarily reconfigure করতে হয় বা dedicated solver port

### তুলনা টেবিল

| | HTTP-01 | DNS-01 | TLS-ALPN-01 |
|--|---------|--------|-------------|
| Wildcard cert | ❌ | ✅ | ❌ |
| Port required | 80 | none | 443 |
| Public server needed | ✅ | ❌ | ✅ |
| Internal/Private hostname | ❌ | ✅ | ❌ |
| Automation difficulty | easy | DNS provider API লাগে | medium |
| Use case | classic public web | wildcards, private | port-80-blocked |

---

## 🛠️ Certbot — Install & First Cert

### Install (Ubuntu/Debian)

```bash
# snap (recommended by EFF — auto-update)
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# OR apt (older but stable)
sudo apt install certbot python3-certbot-nginx python3-certbot-apache
```

### Plugin choice

| Plugin | কীভাবে কাজ করে | কখন? |
|--------|----------------|------|
| `--nginx` | Nginx config auto-edit + reload | already-running Nginx |
| `--apache` | Apache vhost auto-edit | already-running Apache |
| `--webroot -w /var/www` | static file drop in webroot | web server already serving, custom config |
| `--standalone` | port 80 nije bind করে | web server না থাকলে / first-time bootstrap |
| `--manual` | আপনি নিজে file/DNS রাখবেন | scripting |
| `--dns-cloudflare` etc. | DNS provider API দিয়ে TXT | wildcard, private |

### First cert — Nginx plugin (সহজ)

```bash
sudo certbot --nginx -d daraz.com.bd -d www.daraz.com.bd \
  --email ops@daraz.com.bd \
  --agree-tos --no-eff-email \
  --redirect
# Nginx config edit, cert install, HTTPS redirect বসিয়ে দেয়
```

### Webroot mode (Nginx config edit চান না)

```bash
sudo certbot certonly --webroot -w /var/www/app/public \
  -d daraz.com.bd -d www.daraz.com.bd \
  --email ops@daraz.com.bd --agree-tos --no-eff-email
```

Nginx এ আগে এটা থাকতে হবে:

```nginx
server {
    listen 80;
    server_name daraz.com.bd www.daraz.com.bd;

    location /.well-known/acme-challenge/ {
        root /var/www/app/public;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

### Standalone mode (bootstrap)

```bash
# Nginx বন্ধ করতে হবে port 80 ছাড়ার জন্য
sudo systemctl stop nginx
sudo certbot certonly --standalone -d new-service.com.bd
sudo systemctl start nginx

# OR HAProxy/Nginx পেছনে থাকলে certbot আলাদা port এ
sudo certbot certonly --standalone --http-01-port 8080 -d ...
# (HAProxy/Nginx এ /.well-known proxy_pass to :8080)
```

---

## 🌟 Wildcard Certificate (DNS-01)

### Cloudflare DNS plugin

```bash
sudo snap install certbot-dns-cloudflare
sudo snap set certbot trust-plugin-with-root=ok
sudo snap connect certbot:plugin certbot-dns-cloudflare
```

API token (Cloudflare → My Profile → API Tokens → Create with `Zone.DNS:Edit`):

```bash
sudo mkdir -p /etc/letsencrypt/secrets
sudo tee /etc/letsencrypt/secrets/cloudflare.ini <<'EOF'
dns_cloudflare_api_token = abcdef1234567890_TOKEN
EOF
sudo chmod 600 /etc/letsencrypt/secrets/cloudflare.ini
```

Issue wildcard:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d 'daraz.com.bd' -d '*.daraz.com.bd' \
  --email ops@daraz.com.bd --agree-tos --no-eff-email
```

ফলাফল: একটি cert যা `daraz.com.bd` + সব subdomain (`api`, `seller`, `cdn`, ...) cover করে।

### Other DNS providers

| DNS Provider | Plugin | Auth |
|--------------|--------|------|
| AWS Route 53 | `certbot-dns-route53` | IAM key (recommended IAM role on EC2) |
| DigitalOcean | `certbot-dns-digitalocean` | API token |
| GoDaddy | `certbot-dns-godaddy` (3rd party) | key + secret |
| Namecheap | `acme.sh` better support | API key |
| **`.com.bd` (BD provider)** | manual | `--manual` + manual TXT record |

### `.com.bd` — manual DNS-01

বাংলাদেশি BTCL `.com.bd`, `.bd` domain — সাধারণত API নেই, manual TXT add করতে হয়:

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d 'shop.com.bd' -d '*.shop.com.bd' \
  --email ops@... --agree-tos
# certbot prompt দেবে — TXT record DNS provider panel এ add করে continue press
```

> ⚠️ Manual mode এ auto-renewal কাজ করে না। **DNS provider API এ migrate করুন** অথবা **acme.sh + alias mode** ব্যবহার করুন (CNAME `_acme-challenge.shop.com.bd` → `_acme-challenge.shop.cf-managed.net` to a Cloudflare-managed zone where automation কাজ করে)।

### CNAME Alias trick (DNS-01 automation যেখানে provider API নেই)

```
shop.com.bd এ এক-বার (manual):
  _acme-challenge.shop.com.bd  CNAME  _acme-challenge.shop.cf.tld

cf.tld → Cloudflare API দিয়ে managed
certbot --dns-cloudflare --domain shop.com.bd
  → ACME server _acme-challenge.shop.com.bd query
  → CNAME follow → cf.tld TXT lookup → ✅
```

---

## ♻️ Auto-renewal

Certbot install এ `/etc/cron.d/certbot` বা `certbot.timer` (snap) auto-create হয় — দিনে ২ বার renew check।

### systemd timer (preferred)

```bash
systemctl status certbot.timer
systemctl list-timers | grep certbot

# manual run (dry-run safe — actual renew হবে না)
sudo certbot renew --dry-run
```

### Cron (fallback)

```cron
# /etc/cron.d/certbot
0 */12 * * * root certbot -q renew
```

### Renewal hooks

`/etc/letsencrypt/renewal-hooks/`:

```
deploy/    — চালু হয় শুধু renewal সফল হলে (নির্দিষ্ট cert এর জন্য)
post/      — সব renew attempt এর পর (সফল/ব্যর্থ)
pre/       — renew এর আগে
```

Example — Nginx reload:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/nginx-reload.sh <<'EOF'
#!/bin/bash
systemctl reload nginx
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/nginx-reload.sh
```

Multi-server cert distribution hook:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/distribute.sh <<'EOF'
#!/bin/bash
# only act for specific cert
if [[ "$RENEWED_LINEAGE" != "/etc/letsencrypt/live/daraz.com.bd" ]]; then
  exit 0
fi
for host in app2 app3 lb1; do
  rsync -az "$RENEWED_LINEAGE/" "deploy@$host:/etc/letsencrypt/live/daraz.com.bd/"
  ssh "deploy@$host" "sudo systemctl reload nginx"
done
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/distribute.sh
```

> **Variable available in hook:** `$RENEWED_LINEAGE` (cert path), `$RENEWED_DOMAINS` (space-separated)

---

## 🔄 Alternatives

### acme.sh — bash, lightweight, DNS API rich

```bash
curl https://get.acme.sh | sh -s email=ops@daraz.com.bd
```

- ১৫০+ DNS provider supported (Cloudflare, Namecheap, GoDaddy, DigitalOcean, BD-specific... অনেক certbot plugin এর চেয়ে বেশি)
- Pure bash — Python dep নেই, container-friendly
- ZeroSSL default, Let's Encrypt swap সহজ

```bash
acme.sh --issue --dns dns_cf -d shop.com.bd -d '*.shop.com.bd' \
  --server letsencrypt
acme.sh --install-cert -d shop.com.bd \
  --key-file /etc/nginx/ssl/key.pem \
  --fullchain-file /etc/nginx/ssl/fullchain.pem \
  --reloadcmd "systemctl reload nginx"
```

### lego — Go binary, single file

```bash
# single static binary, ideal for containers/k8s
lego --email ops@... --domains "*.daraz.com.bd" \
  --dns cloudflare --accept-tos run
```

### Caddy — built-in auto HTTPS

```caddyfile
daraz.com.bd, www.daraz.com.bd {
    reverse_proxy localhost:3000
}
```

✅ Caddyfile এ domain লিখলেই — Caddy নিজে cert issue + renew + OCSP staple। **শূন্য config**।
✅ Wildcard, DNS-01 — DNS plugin compile করতে হয় (xcaddy)।

> বাংলাদেশের ছোট startup, single-VM deploy — Caddy সবচেয়ে ঝামেলামুক্ত।

### Traefik — reverse proxy + ACME

```yaml
certificatesResolvers:
  le:
    acme:
      email: ops@daraz.com.bd
      storage: /acme.json
      httpChallenge:
        entryPoint: web
      # OR for wildcard:
      dnsChallenge:
        provider: cloudflare
```

### mod_md (Apache built-in)

Apache 2.4.30+ এ:

```apache
MDomain daraz.com.bd www.daraz.com.bd
MDCertificateAgreement accepted
MDContactEmail ops@daraz.com.bd

<VirtualHost *:443>
    ServerName daraz.com.bd
    SSLEngine on
    # mod_md auto-provides cert
</VirtualHost>
```

> বিস্তারিত — `apache-deep-dive.md`।

---

## 🌐 Multi-server Architecture

### Pattern 1: Shared cert via NFS / sync

```
       Let's Encrypt
            ▲
            │
     ┌──────┴──────┐
     │   cert-host │  certbot এই machine এ renew, NFS export
     └──────┬──────┘
            │ NFS / rsync
   ┌────────┼────────┐
   ▼        ▼        ▼
  app1    app2     app3   (Nginx mount, periodic reload)
```

### Pattern 2: Central cert manager + push

```bash
# central host এ certbot
# renewal hook → rsync to all + reload (already shown)
```

### Pattern 3: Load Balancer terminates TLS

```
                Internet
                   │
              ┌────▼────┐
              │   ALB   │  (cert in ACM/Cloudflare)
              └────┬────┘
                   │ HTTP (private VPC)
       ┌───────────┼───────────┐
       ▼           ▼           ▼
      app1        app2        app3   (no cert needed)
```

✅ AWS ACM (free, auto-renew)
✅ Cloudflare proxy (Origin cert OR Full Strict)

### Pattern 4: HAProxy SSL termination

```
frontend https
    bind *:443 ssl crt /etc/haproxy/certs/  alpn h2,http/1.1
    default_backend app
```

`/etc/haproxy/certs/` এ all-in-one PEM (cert+key concat) — certbot deploy hook থেকে generate:

```bash
cat /etc/letsencrypt/live/$d/fullchain.pem /etc/letsencrypt/live/$d/privkey.pem \
  > /etc/haproxy/certs/$d.pem
systemctl reload haproxy
```

---

## ☸️ Kubernetes — cert-manager

cert-manager = K8s-native ACME controller।

### Install

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```

### ClusterIssuer (Let's Encrypt prod)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@daraz.com.bd
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones: ["daraz.com.bd"]
```

### Ingress with auto-cert

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: daraz-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - daraz.com.bd
        - "*.daraz.com.bd"   # DNS-01 solver pick হবে
      secretName: daraz-tls
  rules:
    - host: daraz.com.bd
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: daraz-app
                port: { number: 80 }
```

cert-manager পেছনে: Order → Challenge resource তৈরি → solver pod spawn → DNS/HTTP challenge solve → CertificateRequest → Secret update → Ingress তে inject।

### Status check

```bash
kubectl get certificate
kubectl describe certificate daraz-tls
kubectl get challenge   # if pending
kubectl describe order
```

---

## 🐳 Docker — Nginx + certbot Sidecar

`docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:1.27
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    command: >
      sh -c "while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g 'daemon off;'"

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: >
      sh -c "trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done"
```

**First-time bootstrap:**

```bash
mkdir -p ./certbot/www ./certbot/conf

# stub config (nginx এ /.well-known route only)
cat > nginx/conf.d/00-bootstrap.conf <<EOF
server {
    listen 80;
    server_name daraz.com.bd;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://\$host\$request_uri; }
}
EOF

docker compose up -d nginx

# issue cert (one-shot)
docker compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d daraz.com.bd -d www.daraz.com.bd \
  --email ops@... --agree-tos --no-eff-email

# now enable HTTPS server block, reload
```

---

## 💰 Free CAs Comparison

| CA | Rate limits | Wildcard | Validity | Note |
|----|-------------|----------|----------|------|
| **Let's Encrypt** | 50 certs/week per registered domain; 5 duplicates/week; 300 new orders/3hr | ✅ | 90 days | most trusted, default everywhere |
| **ZeroSSL** | similar but slightly looser; account-required | ✅ | 90 days | acme.sh default; nicer UI |
| **Buypass Go** | ~20 certs/week | ✅ | 180 days ⚡ | longer validity = renewal pressure কম |
| **Google Trust Services** | (production-only API)  | ✅ | 90 days | enterprise/ZT scenarios |
| **SSL.com (free 90d trial)** | per account | ✅ | 90 days | EV options paid |

> Let's Encrypt rate limit hit হলে — staging endpoint test, then ZeroSSL fallback রাখুন।

### Staging endpoint (testing)

```bash
certbot certonly --staging --dry-run -d test.daraz.com.bd ...
```

Cert untrusted হবে browser এ — শুধু integration test এর জন্য।

---

## 🔑 mTLS with Internal CA

External CA mTLS এ certificate-per-client করার চেষ্টা = chaos। Internal CA চাই।

### cfssl (CloudFlare)

```bash
# CA bootstrap
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# client cert
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=client \
  client-csr.json | cfssljson -bare bkash-partner
```

### smallstep CA (modern, ACME-compatible internal CA)

```bash
step ca init --name "Daraz Internal CA" \
  --dns ca.internal.daraz.com.bd \
  --address :443 --provisioner admin

# নিজস্ব ACME server চালু করুন!
step ca provisioner add acme --type ACME

# এখন certbot/acme.sh দিয়ে internal cert পাওয়া যাবে
acme.sh --issue --server https://ca.internal.daraz.com.bd/acme/acme/directory \
  -d service.internal.daraz.com.bd --standalone
```

### HashiCorp Vault PKI

```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal \
  common_name="Daraz Root CA" ttl=87600h

vault write pki/roles/internal-services \
  allowed_domains="internal.daraz.com.bd" \
  allow_subdomains=true max_ttl=720h

# issue
vault write pki/issue/internal-services \
  common_name="api.internal.daraz.com.bd" ttl=24h
```

> bKash এর partner gateway এ — **mTLS** mandatory। Vault PKI দিয়ে partner-specific client cert issue, short TTL, auto-rotate।

---

## ⚠️ Operational Concerns

### Rate limits (Let's Encrypt) — ৯৫% incident এর কারণ

```
50 certs / week / registered domain
5 duplicate certs / week (same FQDN list)
300 new orders / 3 hours / account
100 names per cert (SAN)
50 pending authorization / account

→ অতিক্রম করলে ১ সপ্তাহ ban
```

**Defense:**
- Always test on `--staging` first
- CI/CD এ "force renew" flag careful
- Wildcard ব্যবহার করে multiple subdomain এক cert এ আনুন
- cert-manager এ accidental loop detect (Renewal storm) → set `dnsNames` carefully

### OCSP Stapling

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/$d/chain.pem;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;
```

⚠️ Let's Encrypt OCSP responder occasional outage → stapling cache stale → SSL Labs F grade। Backup resolver রাখুন।

### Certificate Transparency monitoring

```bash
# crt.sh দিয়ে নিজের domain এর সব cert audit
curl -s "https://crt.sh/?q=daraz.com.bd&output=json" | jq -r '.[].name_value' | sort -u
```

✅ unauthorized cert issuance detect করুন (rogue CA / supply chain)
✅ Cert Spotter, Facebook CT Monitor — email alert

### Expiry monitoring/alerting

```bash
# cron — 14 দিন আগে warn
domains=("daraz.com.bd" "api.daraz.com.bd")
for d in "${domains[@]}"; do
  exp=$(echo | openssl s_client -servername "$d" -connect "$d:443" 2>/dev/null \
        | openssl x509 -noout -enddate | cut -d= -f2)
  exp_ts=$(date -d "$exp" +%s)
  now=$(date +%s)
  days=$(( (exp_ts - now) / 86400 ))
  if (( days < 14 )); then
    echo "WARN: $d expires in $days days" | mail -s "[ALERT] Cert expiry" ops@...
  fi
done
```

বা **Better Stack / Uptime Kuma / Grafana** — TLS expiry check built-in।

---

## 🐛 Common Errors & Troubleshooting

### `Connection refused` / HTTP-01 firewall

```
Detail: Fetching http://daraz.com.bd/.well-known/acme-challenge/abc:
        Connection refused
```

✅ Check `ufw status` — port 80 open?
✅ Cloud security group — port 80 ingress?
✅ Cloudflare proxy on? **HTTP-01 fail** because CF terminate। DNS-01 use করুন বা CF gray cloud temporarily।

### `Mixed content` after HTTPS

```
Browser console: Loaded http://cdn.../style.css from secure page
```

✅ All `<img>`, `<script>`, `<link>` URL relative করুন (`//cdn...` or `/path`)
✅ `Content-Security-Policy: upgrade-insecure-requests` header

### HSTS preload trap

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
```

⚠️ Once submitted to **hstspreload.org**, browsers will refuse HTTP forever for that domain — even subdomain। কোনো subdomain TLS ছাড়া রাখলে অগম্য।

> Recovery: ১ বছর+ লাগে preload removal। **প্রথমে ছোট max-age (300) দিয়ে test, তারপর বাড়ান**।

### `unauthorized` — TXT record not propagated

```bash
# DNS check (multiple resolvers)
dig +short TXT _acme-challenge.daraz.com.bd @1.1.1.1
dig +short TXT _acme-challenge.daraz.com.bd @8.8.8.8
```

⏱️ TTL/propagation — `--dns-cloudflare-propagation-seconds 60` বাড়ান।

### "Too many failed authorizations"

⚠️ ১ ঘণ্টায় বেশি wrong attempt → soft-block। ১ ঘণ্টা wait, fix root cause first।

### Wildcard cert HTTP-01 দিয়ে চাইছেন

```
Error: HTTP-01 challenge cannot validate wildcard
```

→ DNS-01 obligatory।

---

## 🇧🇩 BD Context

### `daraz.com.bd` wildcard cert

Daraz এর শত শত subdomain — `seller.daraz.com.bd`, `api.daraz.com.bd`, `cdn.daraz.com.bd`, `m.daraz.com.bd`, `aff.daraz.com.bd`...

**একটা wildcard `*.daraz.com.bd`** — DNS-01 (Cloudflare API) দিয়ে। প্রতি ৯০ দিনে auto-renew।

### Cloudflare Origin Certs vs Let's Encrypt

| | Cloudflare Origin Cert | Let's Encrypt |
|--|------------------------|---------------|
| Trust | Cloudflare-issued, **only valid via CF proxy** | Public CA — সবার কাছে valid |
| Validity | 15 years | 90 days |
| Issuance | One-click in CF dashboard | ACME |
| Use case | Origin behind CF proxy (Full Strict) | Standalone, direct browser access |

✅ **Best practice:** CF proxy on → Origin cert (15 yr, no renewal headache) origin server এ।
+ Public Let's Encrypt cert একটাও রাখুন (যদি কেউ direct origin IP hit করে, বা CF বাইপাস দরকার হয়)।

### bKash mTLS partner gateway

- Merchant দের নিজস্ব **client certificate** ইস্যু করে bKash internal CA
- API call এ TLS handshake এ client cert verify
- Cert revoke (CRL/OCSP) — compromised partner block
- Vault PKI দিয়ে issue, short-TTL (24-72hr) auto-rotate, partner দের API দিয়ে fetch

### `.com.bd` BTCL domain

- Manual DNS এ stuck → CNAME alias trick (covered earlier) — `_acme-challenge` কে Cloudflare-managed zone এ point করুন
- অথবা Cloudflare এ domain transfer (BTCL change support করছে ২০২৩+)

### SSLCommerz, AamarPay, ShurjoPay TLS

local payment gateway integration এ — তাদের CA certificate সঠিক chain pin করুন। Let's Encrypt CA root rotation (Sept 2024 ISRG Root X2) এ কিছু legacy gateway broken — chain pin update করতে হবে।

---

## 📋 Full Nginx + Certbot Production Example

### Setup (combined)

```bash
# 1. install
sudo apt install nginx certbot python3-certbot-nginx

# 2. base config
sudo tee /etc/nginx/conf.d/daraz.conf <<'EOF'
server {
    listen 80;
    server_name daraz.com.bd www.daraz.com.bd;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
EOF

sudo mkdir -p /var/www/certbot
sudo nginx -t && sudo systemctl reload nginx

# 3. issue cert (webroot)
sudo certbot certonly --webroot -w /var/www/certbot \
  -d daraz.com.bd -d www.daraz.com.bd \
  --email ops@daraz.com.bd --agree-tos --no-eff-email

# 4. TLS server block
sudo tee /etc/nginx/conf.d/daraz-ssl.conf <<'EOF'
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name daraz.com.bd www.daraz.com.bd;

    ssl_certificate     /etc/letsencrypt/live/daraz.com.bd/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/daraz.com.bd/privkey.pem;

    # Mozilla intermediate (2024)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/daraz.com.bd/chain.pem;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # HSTS — start small (300s), confirm, then bump to 2yr+preload
    add_header Strict-Transport-Security "max-age=300" always;

    root /var/www/daraz/public;
    index index.php;

    location / { try_files $uri $uri/ =404; }
}
EOF

sudo nginx -t && sudo systemctl reload nginx

# 5. renew hook
sudo tee /etc/letsencrypt/renewal-hooks/deploy/nginx.sh <<'EOF'
#!/bin/bash
systemctl reload nginx
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/nginx.sh

# 6. verify auto-renew
sudo certbot renew --dry-run

# 7. SSL Labs test
# https://www.ssllabs.com/ssltest/analyze.html?d=daraz.com.bd
# A+ grade target
```

---

## ✅ Checklist

### Setup
- [ ] DNS A/AAAA correctly pointing to server (verify with `dig`)
- [ ] Port 80 + 443 open in firewall + cloud SG
- [ ] `/etc/letsencrypt/` permissions correct (root:root 700)
- [ ] Email registered for expiry warnings
- [ ] First cert issued via `--staging` → success → real

### Renewal
- [ ] `certbot.timer` or cron active
- [ ] `certbot renew --dry-run` passes
- [ ] Deploy hook reloads webserver/proxy
- [ ] Multi-server distribution hook (if applicable)
- [ ] Slack/email alert on renewal failure

### Security
- [ ] TLS 1.2 + 1.3 only (no 1.0/1.1)
- [ ] Modern cipher suite (Mozilla intermediate)
- [ ] OCSP stapling on
- [ ] HSTS header (escalate gradually)
- [ ] CSP, X-Frame-Options
- [ ] CAA DNS record (lock to Let's Encrypt only)
   ```
   daraz.com.bd. IN CAA 0 issue "letsencrypt.org"
   daraz.com.bd. IN CAA 0 issuewild "letsencrypt.org"
   daraz.com.bd. IN CAA 0 iodef "mailto:ops@..."
   ```

### Monitoring
- [ ] Expiry alert 30/14/7 day
- [ ] CT log monitoring (crt.sh / Cert Spotter)
- [ ] OCSP responder reachable
- [ ] SSL Labs A/A+ grade
- [ ] Mixed content check post-deploy

### Wildcard / Multi-domain
- [ ] DNS-01 challenge automated (provider API token)
- [ ] All SAN included in single cert (where appropriate)
- [ ] Don't accidentally hit rate limit (test on staging)

### Kubernetes
- [ ] cert-manager installed, healthy
- [ ] ClusterIssuer (prod + staging)
- [ ] Ingress annotation correct
- [ ] CertificateRequest → Order → Challenge → ✅ traced

### mTLS
- [ ] Internal CA (smallstep/Vault) established
- [ ] Short TTL + auto-rotate
- [ ] CRL/OCSP for revocation
- [ ] Partner onboarding/offboarding documented

### BD-specific
- [ ] `.com.bd` automation (CNAME alias if no API)
- [ ] Cloudflare Origin cert + LE both configured (if behind CF)
- [ ] Local payment gateway TLS chain pinned correctly

---

## 📚 আরো পড়ুন

- `14-networking/tls-ssl.md` — TLS handshake, cert chain theory
- `nginx-deep-dive.md` — Nginx TLS tuning
- `apache-deep-dive.md` — Apache TLS, mod_md
- `kubernetes.md` — Ingress, cert-manager K8s
- `secrets-management.md` — Vault PKI, SOPS
- `server-deployment-guide.md` — full stack deployment context
- `saas-paas-iaas.md` — SSL responsibility per cloud model

---

> **শেষ কথা:** TLS এখন আর "ইচ্ছা থাকলে" নয় — Chrome HTTP-only site কে warn দেখায়, SEO penalty দেয়। ACME + certbot/cert-manager দিয়ে এটা **set-and-forget** করুন। কিন্তু ভুলে গেলে ৯০ দিন পর Daraz/bKash এর payment page এ "Not Secure" — যা কোম্পানির trust পুরো ধ্বংস করে। **monitoring + alerting মাস্ট, automation alone যথেষ্ট নয়।**
