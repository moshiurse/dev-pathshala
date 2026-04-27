# 📘 CI/CD পাইপলাইন

> **Continuous Integration ও Continuous Delivery/Deployment — সফটওয়্যার ডেলিভারি অটোমেশনের মূল ভিত্তি।**

---

## 📖 সংজ্ঞা

### Continuous Integration (CI)

CI হলো এমন একটি অনুশীলন যেখানে ডেভেলপাররা **দিনে একাধিকবার** তাদের কোড মূল ব্রাঞ্চে merge করে এবং **প্রতিটি merge এ অটোমেটেড বিল্ড ও টেস্ট** চলে।

### Continuous Delivery (CD)

CD হলো CI এর পরবর্তী ধাপ যেখানে কোড **সবসময় ডেপ্লয়-রেডি** অবস্থায় থাকে। প্রোডাকশনে ডেপ্লয়ের জন্য শুধু একটি **ম্যানুয়াল অনুমোদন** দরকার।

### Continuous Deployment

Continuous Delivery এর আরেক ধাপ এগিয়ে — কোনো ম্যানুয়াল অনুমোদন ছাড়াই **অটোমেটিক্যালি প্রোডাকশনে ডেপ্লয়** হয়ে যায়।

```
CI/CD পাইপলাইন ফ্লো:

কোড Push ──► Build ──► Unit Test ──► Integration Test ──► Staging Deploy
                                                              │
                                                              ▼
                                               Acceptance Test / QA
                                                              │
                                                              ▼
                                               Production Deploy (CD)
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### কারখানার Assembly Line

একটি গাড়ির কারখানা চিন্তা করুন:
- প্রতিটি ধাপে (ওয়েল্ডিং, পেইন্টিং, ইঞ্জিন) **অটোমেটিক মান নিয়ন্ত্রণ** হয়
- কোনো ধাপে ত্রুটি পাওয়া গেলে **লাইন বন্ধ** হয়ে যায়
- ত্রুটি ছাড়া সব ধাপ পাস করলে গাড়ি **ডেলিভারি রেডি**

CI/CD ও ঠিক এরকম — কোডের প্রতিটি পরিবর্তনে অটোমেটেড চেক চলে!

---

## 🔧 GitHub Actions — সম্পূর্ণ গাইড

### বেসিক ধারণা

```yaml
# .github/workflows/ci.yml

# Workflow — সম্পূর্ণ পাইপলাইন
name: CI/CD Pipeline

# Trigger — কখন চলবে
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Environment Variables — গ্লোবাল
env:
  NODE_VERSION: '20'
  PHP_VERSION: '8.3'

# Jobs — সমান্তরাল কাজ
jobs:
  # জব ১: কোড লিন্টিং
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: সেটআপ Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  # জব ২: ইউনিট টেস্ট
  test:
    runs-on: ubuntu-latest
    needs: lint  # lint পাস করার পরে চলবে
    strategy:
      matrix:
        node-version: [18, 20, 22]  # একাধিক ভার্সনে টেস্ট
    steps:
      - uses: actions/checkout@v4
      - name: সেটআপ Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - name: কভারেজ আপলোড
        uses: codecov/codecov-action@v3

  # জব ৩: বিল্ড
  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Build Artifact সেভ
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  # জব ৪: ডেপ্লয়
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'  # শুধু main ব্রাঞ্চে
    environment: production
    steps:
      - name: Build Artifact ডাউনলোড
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - name: প্রোডাকশনে ডেপ্লয়
        run: |
          echo "প্রোডাকশনে ডেপ্লয় হচ্ছে..."
          # AWS S3, Vercel, বা কাস্টম সার্ভারে ডেপ্লয়
```

### PHP Laravel প্রজেক্টের জন্য CI/CD

```yaml
# .github/workflows/laravel-ci.yml
name: Laravel CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  laravel-tests:
    runs-on: ubuntu-latest

    services:
      # MySQL সার্ভিস কন্টেইনার
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

      # Redis সার্ভিস কন্টেইনার
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - name: PHP সেটআপ
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo, pdo_mysql, redis
          coverage: xdebug

      - name: Composer Cache
        uses: actions/cache@v4
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}
          restore-keys: composer-

      - name: Dependencies ইনস্টল
        run: composer install --no-interaction --prefer-dist

      - name: Environment তৈরি
        run: |
          cp .env.testing .env
          php artisan key:generate

      - name: মাইগ্রেশন চালানো
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
        run: php artisan migrate --force

      - name: PHPUnit টেস্ট
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
          REDIS_HOST: 127.0.0.1
        run: php artisan test --coverage-clover coverage.xml

      - name: PHP CS Fixer (কোড স্টাইল)
        run: vendor/bin/php-cs-fixer fix --dry-run --diff

      - name: PHPStan (Static Analysis)
        run: vendor/bin/phpstan analyse

  deploy-staging:
    needs: laravel-tests
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Staging এ ডেপ্লয়
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/staging
            git pull origin develop
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan queue:restart
            echo "Staging ডেপ্লয় সম্পন্ন!"

  deploy-production:
    needs: laravel-tests
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # ম্যানুয়াল অনুমোদন দরকার
    steps:
      - uses: actions/checkout@v4
      - name: প্রোডাকশনে ডেপ্লয়
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /var/www/production
            php artisan down --retry=60
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan optimize
            php artisan up
            echo "প্রোডাকশন ডেপ্লয় সম্পন্ন!"
```

### Node.js/Express প্রজেক্টের CI/CD

```yaml
# .github/workflows/node-ci.yml
name: Node.js CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:7
        ports:
          - 27017:27017
      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check  # TypeScript চেক

      - name: ইউনিট টেস্ট
        run: npm test -- --coverage --forceExit
        env:
          MONGODB_URI: mongodb://localhost:27017/test
          REDIS_URL: redis://localhost:6379

      - name: E2E টেস্ট
        run: npm run test:e2e
        env:
          MONGODB_URI: mongodb://localhost:27017/test_e2e

  docker-build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Docker Buildx সেটআপ
        uses: docker/setup-buildx-action@v3

      - name: Docker Hub লগইন
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Docker Image বিল্ড ও পুশ
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myapp/api:latest
            myapp/api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: docker-build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: কুবারনেটিসে ডেপ্লয়
        run: |
          kubectl set image deployment/api-deployment \
            api=myapp/api:${{ github.sha }}
          kubectl rollout status deployment/api-deployment
```

---

## 🔬 গভীর বিশ্লেষণ (Deep Dive)

### Jenkins Pipeline (Jenkinsfile)

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        APP_NAME = 'my-app'
    }

    stages {
        stage('চেকআউট') {
            steps {
                checkout scm
            }
        }

        stage('ডিপেন্ডেন্সি ইনস্টল') {
            parallel {
                stage('PHP Dependencies') {
                    steps {
                        sh 'composer install --no-interaction'
                    }
                }
                stage('Node Dependencies') {
                    steps {
                        sh 'npm ci'
                    }
                }
            }
        }

        stage('কোড কোয়ালিটি') {
            parallel {
                stage('PHP Lint') {
                    steps {
                        sh 'vendor/bin/php-cs-fixer fix --dry-run'
                    }
                }
                stage('JS Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Static Analysis') {
                    steps {
                        sh 'vendor/bin/phpstan analyse'
                    }
                }
            }
        }

        stage('টেস্ট') {
            steps {
                sh 'php artisan test --coverage-clover coverage.xml'
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    junit '**/test-results/*.xml'
                    publishHTML(target: [
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Docker Build') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .
                    docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                """
            }
        }

        stage('ডেপ্লয়') {
            when {
                branch 'main'
            }
            input {
                message "প্রোডাকশনে ডেপ্লয় করবেন?"
                ok "হ্যাঁ, ডেপ্লয় করুন"
            }
            steps {
                sh "kubectl set image deployment/${APP_NAME} app=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "✅ বিল্ড #${BUILD_NUMBER} সফল!")
        }
        failure {
            slackSend(color: 'danger', message: "❌ বিল্ড #${BUILD_NUMBER} ব্যর্থ!")
        }
    }
}
```

### ডেপ্লয়মেন্ট স্ট্র্যাটেজি

```
১. Rolling Deployment (ধীরে ধীরে আপডেট):
┌───────────────────────────────────┐
│ v1 │ v1 │ v1 │ v1 │   ← শুরু    │
│ v2 │ v1 │ v1 │ v1 │   ← ধাপ ১   │
│ v2 │ v2 │ v1 │ v1 │   ← ধাপ ২   │
│ v2 │ v2 │ v2 │ v1 │   ← ধাপ ৩   │
│ v2 │ v2 │ v2 │ v2 │   ← সম্পন্ন  │
└───────────────────────────────────┘
সুবিধা: জিরো ডাউনটাইম, ধীরে ধীরে রোলআউট
অসুবিধা: মিশ্র ভার্সন চলে কিছুক্ষণ

২. Blue-Green Deployment:
┌─────────┐     ┌─────────┐
│  Blue   │ ←── │  Load   │
│  (v1)   │     │Balancer │
└─────────┘     └────┬────┘
┌─────────┐          │
│  Green  │          │ সুইচ!
│  (v2)   │ ◄────────┘
└─────────┘
সুবিধা: তাৎক্ষণিক রোলব্যাক, কোনো মিশ্র ভার্সন নেই
অসুবিধা: দ্বিগুণ রিসোর্স দরকার

৩. Canary Deployment:
┌─────────────────────────────────┐
│ v1(90%) ──────────── │ v2(10%) │  ← ১০% ট্রাফিক নতুন ভার্সনে
│ v1(70%) ──────────── │ v2(30%) │  ← সফল? আরো বাড়াও
│ v1(50%) ──────────── │ v2(50%) │  ← আরো বাড়াও
│ v2(100%) ─────────────────────│  ← সম্পূর্ণ রোলআউট
└─────────────────────────────────┘
সুবিধা: ঝুঁকি কম, রিয়েল ইউজার দিয়ে টেস্ট
অসুবিধা: জটিল সেটআপ, মনিটরিং দরকার
```

### Git Branching Strategy ও CI/CD

```
Git Flow + CI/CD:

main ──────●──────●──────●──────●──── (প্রোডাকশন)
           │      ▲      │      ▲
           │      │      │      │
develop ───●──●───●──●───●──●───●──── (স্টেজিং → অটো ডেপ্লয়)
              │      │      │
feature/* ────●      │      │         (CI: লিন্ট + টেস্ট)
                     │      │
feature/* ───────────●      │         (CI: লিন্ট + টেস্ট)
                            │
hotfix/* ───────────────────●         (CI + প্রোডাকশন হটফিক্স)
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **দ্রুত ফিডব্যাক** | কোড push এর মিনিটের মধ্যে বাগ ধরা পড়ে |
| ২ | **কম মানবিক ত্রুটি** | অটোমেশন ম্যানুয়াল ভুল কমায় |
| ৩ | **ঘন ঘন রিলিজ** | ছোট ছোট পরিবর্তন দ্রুত ডেলিভার |
| ৪ | **আত্মবিশ্বাস** | অটোমেটেড টেস্ট কোড কোয়ালিটি নিশ্চিত করে |
| ৫ | **রোলব্যাক সহজ** | সমস্যা হলে আগের ভার্সনে ফেরা সহজ |
| ৬ | **টিম প্রোডাক্টিভিটি** | "ডেপ্লয় ডে" এর ভয় নেই |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **প্রাথমিক সেটআপ খরচ** | পাইপলাইন তৈরি করতে সময় ও দক্ষতা দরকার |
| ২ | **মেইনটেন্যান্স** | পাইপলাইন নিজেও মেইনটেইন করতে হয় |
| ৩ | **টেস্ট লিখতে হবে** | অটোমেটেড টেস্ট ছাড়া CI অর্থহীন |
| ৪ | **ইনফ্রাস্ট্রাকচার খরচ** | CI রানার, সার্ভার, স্টোরেজের খরচ |
| ৫ | **ফলস সিকিউরিটি** | পাইপলাইন পাস মানেই সব ঠিক নয় — ভালো টেস্ট দরকার |

---

## 📏 বেস্ট প্র্যাকটিস

1. **প্রতিটি কমিটে CI চালান** — শুধু main/develop নয়, feature ব্রাঞ্চেও
2. **ফাস্ট ফেইল** — সবচেয়ে দ্রুত চেক (lint) আগে চালান
3. **ক্যাশিং ব্যবহার করুন** — `npm ci`, `composer install` ক্যাশ করে দ্রুত করুন
4. **সিক্রেটস নিরাপদ রাখুন** — API key, password কখনো কোডে নয়
5. **পাইপলাইন-অ্যাজ-কোড** — YAML/Jenkinsfile ভার্সন কন্ট্রোলে রাখুন
6. **নোটিফিকেশন সেটআপ** — Slack/Teams এ বিল্ড স্ট্যাটাস পাঠান
7. **Artifact সংরক্ষণ** — বিল্ড আউটপুট, টেস্ট রিপোর্ট সেভ রাখুন

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **CI** | প্রতি push এ অটো বিল্ড ও টেস্ট |
| **CD (Delivery)** | সবসময় ডেপ্লয়-রেডি, ম্যানুয়াল অনুমোদন |
| **CD (Deployment)** | অটোমেটিক প্রোডাকশন ডেপ্লয় |
| **টুলস** | GitHub Actions, Jenkins, GitLab CI, CircleCI |
| **মূলনীতি** | ফাস্ট ফেইল, ক্যাশিং, সিক্রেটস ম্যানেজমেন্ট |
