# 📘 ডকার ও কন্টেইনার

> **কন্টেইনারাইজেশন — "আমার মেশিনে তো চলে" সমস্যার চূড়ান্ত সমাধান।**

---

## 📖 সংজ্ঞা

### কন্টেইনার কী?

কন্টেইনার হলো একটি **হালকা, স্বয়ংসম্পূর্ণ, এক্সিকিউটেবল প্যাকেজ** যাতে অ্যাপ্লিকেশন চালানোর জন্য প্রয়োজনীয় সবকিছু থাকে — কোড, রানটাইম, সিস্টেম টুলস, লাইব্রেরি ও সেটিংস।

### Docker কী?

Docker হলো সবচেয়ে জনপ্রিয় **কন্টেইনার প্ল্যাটফর্ম** যা অ্যাপ্লিকেশন তৈরি, শিপ ও রান করতে সাহায্য করে।

```
ভার্চুয়াল মেশিন vs কন্টেইনার:

┌──────────────────────┐  ┌──────────────────────┐
│    Virtual Machine    │  │     Container         │
├──────────────────────┤  ├──────────────────────┤
│   App A  │   App B   │  │   App A  │   App B   │
│  Bins/Libs│ Bins/Libs │  │  Bins/Libs│ Bins/Libs │
│ Guest OS │ Guest OS  │  │   Docker Engine       │
│      Hypervisor       │  │      Host OS          │
│      Host OS          │  │      Hardware         │
│      Hardware         │  └──────────────────────┘
└──────────────────────┘
  ভারী (GB), ধীর বুট        হালকা (MB), দ্রুত বুট
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### শিপিং কন্টেইনারের উদাহরণ

আন্তর্জাতিক শিপিং বিপ্লব হয়েছিল **স্ট্যান্ডার্ড কন্টেইনার** আবিষ্কারে:
- আগে: প্রতিটি পণ্য আলাদাভাবে লোড/আনলোড — ধীর, ক্ষতির ঝুঁকি
- পরে: সব পণ্য **একটি স্ট্যান্ডার্ড বাক্সে** — যেকোনো জাহাজ, ট্রাক, ট্রেনে চলে

Docker কন্টেইনারও তেমন — **যেকোনো সার্ভারে** একইভাবে চলে!

---

## 📦 Dockerfile — ইমেজ তৈরি

### PHP Laravel Dockerfile

```dockerfile
# ============================================
# ধাপ ১: Dependencies ইনস্টল (Builder Stage)
# ============================================
FROM composer:2 AS composer-builder

WORKDIR /app

# শুধু dependency ফাইল কপি — ক্যাশ অপটিমাইজেশন
COPY composer.json composer.lock ./
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-autoloader \
    --prefer-dist

# সম্পূর্ণ কোড কপি ও অটোলোডার জেনারেট
COPY . .
RUN composer dump-autoload --optimize --no-dev

# ============================================
# ধাপ ২: Frontend Build
# ============================================
FROM node:20-alpine AS frontend-builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ============================================
# ধাপ ৩: প্রোডাকশন ইমেজ
# ============================================
FROM php:8.3-fpm-alpine AS production

# প্রয়োজনীয় PHP এক্সটেনশন
RUN apk add --no-cache \
        libpng-dev \
        libjpeg-turbo-dev \
        freetype-dev \
        oniguruma-dev \
        libzip-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
        pdo_mysql \
        mbstring \
        gd \
        zip \
        opcache \
        pcntl \
        bcmath

# Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# PHP কনফিগারেশন অপটিমাইজ
COPY docker/php/php.ini /usr/local/etc/php/conf.d/app.ini
COPY docker/php/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

WORKDIR /var/www/html

# Composer dependencies কপি (Builder Stage থেকে)
COPY --from=composer-builder /app/vendor ./vendor

# Frontend assets কপি (Frontend Builder থেকে)
COPY --from=frontend-builder /app/public/build ./public/build

# অ্যাপ্লিকেশন কোড কপি
COPY . .

# পারমিশন সেট
RUN chown -R www-data:www-data storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

# Laravel অপটিমাইজেশন
RUN php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache

# হেলথচেক
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD php artisan --version || exit 1

EXPOSE 9000

USER www-data

CMD ["php-fpm"]
```

### Node.js/Express Dockerfile

```dockerfile
# ============================================
# ধাপ ১: Builder Stage
# ============================================
FROM node:20-alpine AS builder

WORKDIR /app

# Dependencies ইনস্টল (ক্যাশ অপটিমাইজ)
COPY package.json package-lock.json ./
RUN npm ci

# TypeScript কম্পাইল
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# শুধু প্রোডাকশন dependencies
RUN npm ci --production && npm cache clean --force

# ============================================
# ধাপ ২: প্রোডাকশন ইমেজ
# ============================================
FROM node:20-alpine AS production

# সিকিউরিটি: non-root ইউজার
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# শুধু প্রয়োজনীয় ফাইল কপি
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup package.json ./

# Environment
ENV NODE_ENV=production
ENV PORT=3000

# হেলথচেক
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

EXPOSE 3000

USER appuser

CMD ["node", "dist/server.js"]
```

---

## 🐙 Docker Compose — মাল্টি-কন্টেইনার

### সম্পূর্ণ Laravel স্ট্যাক

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # PHP-FPM অ্যাপ্লিকেশন
  # ============================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: laravel-app
    restart: unless-stopped
    volumes:
      - ./storage:/var/www/html/storage  # পারসিস্ট্যান্ট স্টোরেজ
    environment:
      - DB_HOST=mysql
      - DB_DATABASE=${DB_DATABASE:-laravel}
      - DB_USERNAME=${DB_USERNAME:-root}
      - DB_PASSWORD=${DB_PASSWORD:-secret}
      - REDIS_HOST=redis
      - CACHE_DRIVER=redis
      - SESSION_DRIVER=redis
      - QUEUE_CONNECTION=redis
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network

  # ============================================
  # Nginx ওয়েব সার্ভার
  # ============================================
  nginx:
    image: nginx:alpine
    container_name: laravel-nginx
    restart: unless-stopped
    ports:
      - "${APP_PORT:-80}:80"
      - "${APP_SSL_PORT:-443}:443"
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./public:/var/www/html/public:ro
      - ./docker/nginx/ssl:/etc/nginx/ssl:ro  # SSL সার্টিফিকেট
    depends_on:
      - app
    networks:
      - app-network

  # ============================================
  # MySQL ডাটাবেস
  # ============================================
  mysql:
    image: mysql:8.0
    container_name: laravel-mysql
    restart: unless-stopped
    ports:
      - "${DB_PORT:-3306}:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD:-secret}
      MYSQL_DATABASE: ${DB_DATABASE:-laravel}
    volumes:
      - mysql-data:/var/lib/mysql           # ডাটা পারসিস্ট
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
      - ./docker/mysql/init:/docker-entrypoint-initdb.d  # ইনিশিয়াল SQL
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # ============================================
  # Redis ক্যাশ ও কিউ
  # ============================================
  redis:
    image: redis:7-alpine
    container_name: laravel-redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    networks:
      - app-network

  # ============================================
  # Laravel Queue Worker
  # ============================================
  queue-worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: laravel-queue
    restart: unless-stopped
    command: php artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - app
      - redis
    networks:
      - app-network

  # ============================================
  # Laravel Task Scheduler (Cron)
  # ============================================
  scheduler:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: laravel-scheduler
    restart: unless-stopped
    command: >
      sh -c "while true; do
        php artisan schedule:run --verbose --no-interaction;
        sleep 60;
      done"
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - app
    networks:
      - app-network

# ============================================
# Volumes — পারসিস্ট্যান্ট ডাটা
# ============================================
volumes:
  mysql-data:
    driver: local
  redis-data:
    driver: local

# ============================================
# Networks
# ============================================
networks:
  app-network:
    driver: bridge
```

### Node.js + MongoDB স্ট্যাক

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      target: production
    container_name: node-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGODB_URI=mongodb://mongo:27017/myapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  mongo:
    image: mongo:7
    container_name: node-mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: node-redis
    restart: unless-stopped
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongo-data:
  redis-data:
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### Docker Networking

```
Bridge Network (ডিফল্ট):
┌─────────────────────────────────────────┐
│            docker0 (bridge)              │
│                                          │
│  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │ app  │──│ mysql│──│ redis│          │
│  │:9000 │  │:3306 │  │:6379 │          │
│  └──────┘  └──────┘  └──────┘          │
│                                          │
│  কন্টেইনাররা নাম দিয়ে যোগাযোগ করে:    │
│  app → mysql:3306 (DNS resolution)      │
│  app → redis:6379                        │
└─────────────────────────────────────────┘

Host Network (কন্টেইনার সরাসরি হোস্টের নেটওয়ার্ক ব্যবহার করে):
- পারফরম্যান্স ভালো
- পোর্ট ম্যাপিং দরকার নেই
- আইসোলেশন কম

Overlay Network (মাল্টি-হোস্ট):
- Docker Swarm / Kubernetes এ ব্যবহৃত
- বিভিন্ন সার্ভারের কন্টেইনার একই নেটওয়ার্কে
```

### Docker Image অপটিমাইজেশন

```dockerfile
# ❌ খারাপ — বড় ইমেজ, ধীর বিল্ড
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm python3 git curl
COPY . /app
WORKDIR /app
RUN npm install
CMD ["node", "server.js"]
# ইমেজ সাইজ: ~800MB 😱

# ✅ ভালো — ছোট ইমেজ, দ্রুত বিল্ড
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production
COPY dist/ ./dist/
USER node
CMD ["node", "dist/server.js"]
# ইমেজ সাইজ: ~120MB ✅

# ✅ সেরা — Multi-stage build
# (উপরের Dockerfile উদাহরণগুলো দেখুন)
# ইমেজ সাইজ: ~80MB 🎉
```

### Docker Layer Caching

```dockerfile
# প্রতিটি instruction একটি layer তৈরি করে
# পরিবর্তন না হলে ক্যাশ থেকে ব্যবহার করে

# Layer ১: বেস ইমেজ (ক্যাশ)
FROM node:20-alpine

WORKDIR /app

# Layer ২: Dependencies (ক্যাশ — package.json না বদলালে)
COPY package.json package-lock.json ./
RUN npm ci

# Layer ৩: সোর্স কোড (প্রতি বিল্ডে পরিবর্তন হয়)
COPY . .
RUN npm run build

# ✅ টিপ: কম পরিবর্তন হওয়া ফাইল আগে COPY করুন
# বেশি পরিবর্তন হওয়া ফাইল পরে COPY করুন
```

### সিকিউরিটি বেস্ট প্র্যাকটিস

```dockerfile
# ১. Non-root ইউজার ব্যবহার করুন
USER node  # বা www-data

# ২. .dockerignore ফাইল ব্যবহার করুন
# .dockerignore:
# node_modules
# .git
# .env
# *.md
# tests/
# docker-compose*.yml

# ৩. সিক্রেটস ইমেজে রাখবেন না
# ❌ ENV DB_PASSWORD=secret123
# ✅ রানটাইমে environment variable দিন

# ৪. ইমেজ স্ক্যান করুন
# docker scout cves myimage:latest

# ৫. রিড-অনলি ফাইলসিস্টেম
# docker run --read-only myimage
```

---

## 🛠️ প্রয়োজনীয় Docker কমান্ড

```bash
# ইমেজ বিল্ড
docker build -t myapp:1.0 .
docker build -t myapp:1.0 --no-cache .          # ক্যাশ ছাড়া

# কন্টেইনার চালানো
docker run -d -p 3000:3000 --name api myapp:1.0
docker run -it --rm myapp:1.0 sh                  # ইন্টার‍্যাক্টিভ শেল

# কন্টেইনার ম্যানেজমেন্ট
docker ps                        # চলমান কন্টেইনার
docker ps -a                     # সব কন্টেইনার
docker logs -f api               # লাইভ লগ
docker exec -it api sh           # চলমান কন্টেইনারে শেল
docker stop api                  # থামানো
docker rm api                    # মুছে ফেলা

# Docker Compose
docker compose up -d             # সব সার্ভিস চালু
docker compose down              # সব বন্ধ
docker compose logs -f app       # নির্দিষ্ট সার্ভিসের লগ
docker compose exec app sh       # সার্ভিসে শেল
docker compose build --no-cache  # রিবিল্ড

# ক্লিনআপ
docker system prune -a           # অব্যবহৃত সব মুছুন
docker volume prune              # অব্যবহৃত volume মুছুন
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **পরিবেশ সামঞ্জস্য** | Dev, Staging, Production সব জায়গায় একই পরিবেশ |
| ২ | **আইসোলেশন** | প্রতিটি সার্ভিস আলাদা কন্টেইনারে — কনফ্লিক্ট নেই |
| ৩ | **দ্রুত স্টার্টআপ** | সেকেন্ডে কন্টেইনার চালু — VM এর মতো মিনিট লাগে না |
| ৪ | **রিসোর্স দক্ষ** | VM এর চেয়ে অনেক কম রিসোর্স ব্যবহার |
| ৫ | **সহজ স্কেলিং** | `docker compose scale app=5` — ৫টি ইনস্ট্যান্স! |
| ৬ | **ভার্সন কন্ট্রোল** | Dockerfile কোডের মতো ভার্সন কন্ট্রোল হয় |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **লার্নিং কার্ভ** | Dockerfile, Compose, Networking শিখতে সময় লাগে |
| ২ | **পারফরম্যান্স ওভারহেড** | সামান্য ওভারহেড আছে (তবে VM এর চেয়ে অনেক কম) |
| ৩ | **ডাটা পারসিস্টেন্স** | কন্টেইনার মুছে গেলে ডাটা যায় — Volume ম্যানেজ করতে হয় |
| ৪ | **সিকিউরিটি** | ভুল কনফিগারেশনে সিকিউরিটি ঝুঁকি |
| ৫ | **macOS/Windows এ ধীর** | Linux ছাড়া অন্য OS এ VM এর উপর চলে — ধীর |

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **Docker** | কন্টেইনার প্ল্যাটফর্ম — অ্যাপ প্যাকেজ ও রান করে |
| **Dockerfile** | ইমেজ তৈরির ব্লুপ্রিন্ট |
| **Docker Compose** | মাল্টি-কন্টেইনার অর্কেস্ট্রেশন |
| **মূলনীতি** | Multi-stage build, non-root, layer caching |
| **ব্যবহার** | Dev environment, CI/CD, Production deployment |
