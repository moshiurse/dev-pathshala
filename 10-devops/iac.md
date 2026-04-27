# 📘 ইনফ্রাস্ট্রাকচার অ্যাজ কোড (IaC)

> **"ইনফ্রাস্ট্রাকচার ম্যানুয়ালি তৈরি করবেন না — কোড দিয়ে করুন।"**

---

## 📖 সংজ্ঞা

Infrastructure as Code (IaC) হলো **কোড/কনফিগারেশন ফাইল ব্যবহার করে ইনফ্রাস্ট্রাকচার প্রোভিশনিং ও ম্যানেজমেন্ট** করা — ম্যানুয়ালি ক্লাউড কনসোলে ক্লিক করার বদলে।

### IaC এর দুটি পদ্ধতি

```
ডিক্লারেটিভ (Declarative):
"আমি চাই ৩টি সার্ভার, ১টি ডাটাবেস, ১টি লোড ব্যালেন্সার"
→ Terraform, CloudFormation, Pulumi
→ কী চাই বলুন — টুল বাকিটা করবে

ইমপারেটিভ (Imperative):
"প্রথমে VPC তৈরি করো, তারপর সাবনেট, তারপর EC2 লঞ্চ করো"
→ Ansible, Shell Scripts, SDK
→ কিভাবে করতে হবে ধাপে ধাপে বলুন
```

---

## 🏠 বাস্তব জীবনের উদাহরণ

### বাড়ি তৈরির উদাহরণ

| ম্যানুয়াল | IaC |
|-----------|-----|
| মুখে বলে বাড়ি বানানো | ব্লুপ্রিন্ট/নকশা অনুযায়ী বানানো |
| প্রতিবার ভিন্ন ফলাফল | প্রতিবার একই ফলাফল |
| কে কী করেছে ট্র্যাক নেই | Git এ সব পরিবর্তন ট্র্যাক |
| ভুল হলে শুরু থেকে | ভুল হলে রোলব্যাক |

---

## 🔧 Terraform — সবচেয়ে জনপ্রিয় IaC টুল

### Terraform বেসিক ধারণা

```
Terraform Workflow:

  Write → Plan → Apply → (Destroy)

  terraform init    ← প্রোভাইডার ডাউনলোড
  terraform plan    ← কী পরিবর্তন হবে দেখুন (dry run)
  terraform apply   ← পরিবর্তন প্রয়োগ করুন
  terraform destroy ← সব মুছে ফেলুন
```

### AWS — সম্পূর্ণ ওয়েব অ্যাপ ইনফ্রাস্ট্রাকচার

```hcl
# main.tf — Terraform কনফিগারেশন

# ============================================
# প্রোভাইডার কনফিগারেশন
# ============================================
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # স্টেট ফাইল S3 তে রাখা (টিম কোলাবোরেশনের জন্য)
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# ============================================
# ভ্যারিয়েবল
# ============================================
variable "project_name" {
  description = "প্রজেক্টের নাম"
  type        = string
  default     = "my-app"
}

variable "environment" {
  description = "পরিবেশ (production, staging, development)"
  type        = string
  default     = "production"
}

variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-southeast-1"
}

variable "db_password" {
  description = "ডাটাবেস পাসওয়ার্ড"
  type        = string
  sensitive   = true  # লগে দেখাবে না
}

# ============================================
# VPC — প্রাইভেট নেটওয়ার্ক
# ============================================
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# পাবলিক সাবনেট (লোড ব্যালেন্সার)
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
  }
}

# প্রাইভেট সাবনেট (অ্যাপ ও ডাটাবেস)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# ইন্টারনেট গেটওয়ে
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# ============================================
# সিকিউরিটি গ্রুপ
# ============================================
# ALB সিকিউরিটি গ্রুপ
resource "aws_security_group" "alb" {
  name   = "${var.project_name}-alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# অ্যাপ সিকিউরিটি গ্রুপ
resource "aws_security_group" "app" {
  name   = "${var.project_name}-app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "শুধু ALB থেকে অ্যাক্সেস"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ডাটাবেস সিকিউরিটি গ্রুপ
resource "aws_security_group" "db" {
  name   = "${var.project_name}-db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "শুধু অ্যাপ থেকে MySQL অ্যাক্সেস"
  }
}

# ============================================
# RDS — ম্যানেজড MySQL ডাটাবেস
# ============================================
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-db-subnet"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_db_instance" "main" {
  identifier     = "${var.project_name}-db"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.medium"

  allocated_storage     = 50
  max_allocated_storage = 200       # অটো স্কেলিং
  storage_encrypted     = true

  db_name  = "app_database"
  username = "admin"
  password = var.db_password

  multi_az               = true     # হাই অ্যাভেইলেবিলিটি
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  backup_retention_period = 7       # ৭ দিনের ব্যাকআপ
  backup_window           = "03:00-04:00"
  maintenance_window      = "Sun:04:00-Sun:05:00"

  skip_final_snapshot = false
  final_snapshot_identifier = "${var.project_name}-final-snapshot"

  tags = {
    Name = "${var.project_name}-mysql"
  }
}

# ============================================
# ElastiCache — Redis
# ============================================
resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.project_name}-redis-subnet"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.project_name}-redis"
  description          = "Redis cache cluster"
  node_type            = "cache.t3.medium"
  num_cache_clusters   = 2           # ১ primary + ১ replica

  engine               = "redis"
  engine_version       = "7.0"
  port                 = 6379

  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.app.id]

  automatic_failover_enabled = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}

# ============================================
# ECS — কন্টেইনার সার্ভিস
# ============================================
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project_name}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app.repository_url}:latest"
      portMappings = [{
        containerPort = 3000
        protocol      = "tcp"
      }]
      environment = [
        { name = "NODE_ENV", value = "production" },
        { name = "DB_HOST", value = aws_db_instance.main.address },
        { name = "REDIS_HOST", value = aws_elasticache_replication_group.main.primary_endpoint_address },
      ]
      secrets = [
        { name = "DB_PASSWORD", valueFrom = aws_secretsmanager_secret.db_password.arn },
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.project_name}"
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "app"
        }
      }
      healthCheck = {
        command     = ["CMD-SHELL", "wget --spider http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "${var.project_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.app.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 3000
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true           # ব্যর্থ ডেপ্লয়ে অটো রোলব্যাক
  }
}

# ============================================
# ALB — লোড ব্যালেন্সার
# ============================================
resource "aws_lb" "main" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_target_group" "app" {
  name        = "${var.project_name}-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 5
    interval            = 30
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# ============================================
# আউটপুট
# ============================================
output "alb_dns" {
  description = "ALB DNS নাম"
  value       = aws_lb.main.dns_name
}

output "db_endpoint" {
  description = "ডাটাবেস এন্ডপয়েন্ট"
  value       = aws_db_instance.main.endpoint
}

output "redis_endpoint" {
  description = "Redis এন্ডপয়েন্ট"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
}
```

### Terraform Variables ফাইল

```hcl
# terraform.tfvars (প্রোডাকশন)
project_name = "my-app"
environment  = "production"
aws_region   = "ap-southeast-1"

# terraform.staging.tfvars (স্টেজিং)
project_name = "my-app"
environment  = "staging"
aws_region   = "ap-southeast-1"
```

---

## 🔧 Ansible — কনফিগারেশন ম্যানেজমেন্ট

### Ansible দিয়ে সার্ভার সেটআপ

```yaml
# playbooks/setup-server.yml
---
- name: ওয়েব সার্ভার সেটআপ
  hosts: webservers
  become: yes

  vars:
    app_user: deploy
    app_dir: /var/www/myapp
    php_version: "8.3"
    node_version: "20"

  tasks:
    # সিস্টেম আপডেট
    - name: সিস্টেম প্যাকেজ আপডেট
      apt:
        update_cache: yes
        upgrade: dist
        cache_valid_time: 3600

    # প্রয়োজনীয় প্যাকেজ
    - name: প্রয়োজনীয় প্যাকেজ ইনস্টল
      apt:
        name:
          - nginx
          - supervisor
          - git
          - curl
          - unzip
          - htop
          - ufw
        state: present

    # PHP ইনস্টল
    - name: PHP {{ php_version }} ইনস্টল
      apt:
        name:
          - "php{{ php_version }}-fpm"
          - "php{{ php_version }}-mysql"
          - "php{{ php_version }}-mbstring"
          - "php{{ php_version }}-xml"
          - "php{{ php_version }}-curl"
          - "php{{ php_version }}-zip"
          - "php{{ php_version }}-gd"
          - "php{{ php_version }}-redis"
          - "php{{ php_version }}-bcmath"
        state: present

    # Node.js ইনস্টল
    - name: Node.js {{ node_version }} ইনস্টল
      shell: |
        curl -fsSL https://deb.nodesource.com/setup_{{ node_version }}.x | bash -
        apt-get install -y nodejs
      args:
        creates: /usr/bin/node

    # Composer ইনস্টল
    - name: Composer ইনস্টল
      shell: |
        curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
      args:
        creates: /usr/local/bin/composer

    # অ্যাপ ইউজার তৈরি
    - name: ডেপ্লয় ইউজার তৈরি
      user:
        name: "{{ app_user }}"
        groups: www-data
        shell: /bin/bash

    # অ্যাপ ডিরেক্টরি
    - name: অ্যাপ ডিরেক্টরি তৈরি
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: www-data
        mode: "0775"

    # Nginx কনফিগারেশন
    - name: Nginx সাইট কনফিগারেশন
      template:
        src: templates/nginx-site.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Nginx রিস্টার্ট

    - name: Nginx সাইট এনাবল
      file:
        src: /etc/nginx/sites-available/myapp
        dest: /etc/nginx/sites-enabled/myapp
        state: link
      notify: Nginx রিস্টার্ট

    # ফায়ারওয়াল
    - name: UFW ফায়ারওয়াল সেটআপ
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:
        - '22'
        - '80'
        - '443'

    - name: UFW এনাবল
      ufw:
        state: enabled
        default: deny

    # Supervisor (Queue Worker)
    - name: Supervisor কনফিগারেশন
      template:
        src: templates/supervisor-worker.conf.j2
        dest: /etc/supervisor/conf.d/laravel-worker.conf
      notify: Supervisor আপডেট

  handlers:
    - name: Nginx রিস্টার্ট
      service:
        name: nginx
        state: restarted

    - name: Supervisor আপডেট
      shell: supervisorctl reread && supervisorctl update

---
# playbooks/deploy.yml — ডেপ্লয়মেন্ট প্লেবুক
- name: অ্যাপ্লিকেশন ডেপ্লয়
  hosts: webservers
  become: yes
  become_user: deploy

  vars:
    app_dir: /var/www/myapp
    repo_url: "git@github.com:user/myapp.git"
    branch: main

  tasks:
    - name: লেটেস্ট কোড টানুন
      git:
        repo: "{{ repo_url }}"
        dest: "{{ app_dir }}"
        version: "{{ branch }}"
        force: yes

    - name: Composer dependencies ইনস্টল
      composer:
        command: install
        working_dir: "{{ app_dir }}"
        no_dev: yes
        optimize_autoloader: yes

    - name: NPM dependencies ইনস্টল ও বিল্ড
      shell: npm ci && npm run build
      args:
        chdir: "{{ app_dir }}"

    - name: মাইগ্রেশন চালানো
      shell: php artisan migrate --force
      args:
        chdir: "{{ app_dir }}"

    - name: ক্যাশ ক্লিয়ার ও রিবিল্ড
      shell: |
        php artisan config:cache
        php artisan route:cache
        php artisan view:cache
        php artisan optimize
      args:
        chdir: "{{ app_dir }}"

    - name: Queue Worker রিস্টার্ট
      shell: php artisan queue:restart
      args:
        chdir: "{{ app_dir }}"

    - name: PHP-FPM রিলোড
      become: yes
      become_user: root
      service:
        name: "php8.3-fpm"
        state: reloaded
```

### Ansible Inventory

```ini
# inventory/production
[webservers]
web1.example.com ansible_user=ubuntu
web2.example.com ansible_user=ubuntu

[dbservers]
db1.example.com ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

---

## 🔬 Terraform vs Ansible তুলনা

| বৈশিষ্ট্য | Terraform | Ansible |
|----------|-----------|---------|
| **ধরন** | ডিক্লারেটিভ | ইমপারেটিভ + ডিক্লারেটিভ |
| **উদ্দেশ্য** | ইনফ্রাস্ট্রাকচার প্রোভিশনিং | কনফিগারেশন ম্যানেজমেন্ট |
| **ভাষা** | HCL | YAML |
| **স্টেট** | স্টেট ফাইল রাখে | স্টেটলেস |
| **এজেন্ট** | এজেন্টলেস | এজেন্টলেস (SSH) |
| **শক্তি** | ক্লাউড রিসোর্স তৈরি | সার্ভার কনফিগারেশন |
| **ব্যবহার** | VPC, EC2, RDS তৈরি | Nginx, PHP, Node ইনস্টল |

### একসাথে ব্যবহার (সেরা পদ্ধতি):

```
Terraform → ইনফ্রাস্ট্রাকচার তৈরি (VPC, EC2, RDS)
    ↓
Ansible → সার্ভার কনফিগারেশন (Nginx, PHP, Deploy)
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **পুনরুৎপাদনযোগ্য** | একই কোড = একই ইনফ্রাস্ট্রাকচার, প্রতিবার |
| ২ | **ভার্সন কন্ট্রোল** | Git এ ইনফ্রাস্ট্রাকচার পরিবর্তন ট্র্যাক |
| ৩ | **কোড রিভিউ** | ইনফ্রাস্ট্রাকচার পরিবর্তন PR দিয়ে রিভিউ |
| ৪ | **দ্রুত প্রোভিশনিং** | মিনিটে নতুন environment তৈরি |
| ৫ | **ডকুমেন্টেশন** | কোড নিজেই ডকুমেন্টেশন |
| ৬ | **ডিজাস্টার রিকভারি** | সব হারিয়ে গেলেও কোড থেকে পুনর্নির্মাণ |
| ৭ | **কনসিস্টেন্সি** | Dev/Staging/Production একই কোড থেকে |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **লার্নিং কার্ভ** | HCL, Ansible YAML শিখতে সময় লাগে |
| ২ | **স্টেট ম্যানেজমেন্ট** | Terraform state ফাইল সাবধানে ম্যানেজ করতে হয় |
| ৩ | **ড্রিফট** | কেউ ম্যানুয়ালি পরিবর্তন করলে কোডের সাথে মেলে না |
| ৪ | **জটিলতা** | বড় ইনফ্রাস্ট্রাকচারে মডিউল ম্যানেজমেন্ট কঠিন |
| ৫ | **প্রাথমিক বিনিয়োগ** | শুরুতে সেটআপে সময় লাগে |

---

## 📏 বেস্ট প্র্যাকটিস

1. **মডিউলার স্ট্রাকচার** — পুনর্ব্যবহারযোগ্য মডিউল তৈরি করুন
2. **রিমোট স্টেট** — Terraform state S3/GCS এ রাখুন, লোকালে নয়
3. **স্টেট লকিং** — DynamoDB দিয়ে একই সময়ে দুজন পরিবর্তন রোধ
4. **ভ্যারিয়েবল ব্যবহার** — হার্ডকোড করবেন না
5. **সিক্রেটস আলাদা** — পাসওয়ার্ড কোডে রাখবেন না
6. **plan রিভিউ** — `terraform plan` সবসময় apply এর আগে দেখুন
7. **ছোট পরিবর্তন** — একবারে অনেক পরিবর্তন করবেন না

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **IaC কী** | কোড দিয়ে ইনফ্রাস্ট্রাকচার ম্যানেজমেন্ট |
| **Terraform** | ক্লাউড রিসোর্স প্রোভিশনিং (ডিক্লারেটিভ) |
| **Ansible** | সার্ভার কনফিগারেশন ও ডেপ্লয়মেন্ট |
| **মূলনীতি** | ভার্সন কন্ট্রোল, মডিউলার, রিমোট স্টেট |
| **সেরা পদ্ধতি** | Terraform (ইনফ্রা) + Ansible (কনফিগ) একসাথে |
