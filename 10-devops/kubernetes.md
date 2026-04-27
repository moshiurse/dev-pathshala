# 📘 কুবারনেটিস (Kubernetes / K8s)

> **কন্টেইনার অর্কেস্ট্রেশনের রাজা — প্রোডাকশনে কন্টেইনার ম্যানেজমেন্টের স্ট্যান্ডার্ড।**

---

## 📖 সংজ্ঞা

Kubernetes (K8s) হলো একটি **ওপেন-সোর্স কন্টেইনার অর্কেস্ট্রেশন প্ল্যাটফর্ম** যা অটোমেটিক্যালি কন্টেইনারাইজড অ্যাপ্লিকেশন **ডেপ্লয়, স্কেল ও ম্যানেজ** করে।

### কেন Docker Compose যথেষ্ট নয়?

| বৈশিষ্ট্য | Docker Compose | Kubernetes |
|-----------|---------------|------------|
| **স্কেলিং** | ম্যানুয়াল | অটো-স্কেলিং |
| **সেলফ-হিলিং** | নেই | কন্টেইনার ক্র্যাশ করলে অটো রিস্টার্ট |
| **লোড ব্যালেন্সিং** | নেই (আলাদা সেটআপ) | বিল্ট-ইন |
| **রোলিং আপডেট** | সীমিত | বিল্ট-ইন, রোলব্যাক সহ |
| **মাল্টি-হোস্ট** | সিঙ্গল হোস্ট | একাধিক সার্ভারে ডিস্ট্রিবিউটেড |
| **সিক্রেটস** | .env ফাইল | এনক্রিপ্টেড সিক্রেটস |

---

## 🏗️ Kubernetes আর্কিটেকচার

```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │  API     │ │  etcd    │ │Scheduler │ │Controller │  │
│  │  Server  │ │(Key-Value│ │          │ │ Manager   │  │
│  │          │ │  Store)  │ │          │ │           │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Node 1     │ │   Node 2     │ │   Node 3     │
│ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
│ │  kubelet │ │ │ │  kubelet │ │ │ │  kubelet │ │
│ │kube-proxy│ │ │ │kube-proxy│ │ │ │kube-proxy│ │
│ │          │ │ │ │          │ │ │ │          │ │
│ │ ┌──┐┌──┐│ │ │ ┌──┐┌──┐│ │ │ ┌──┐┌──┐│ │
│ │ │P1││P2││ │ │ │P3││P4││ │ │ │P5││P6││ │
│ │ └──┘└──┘│ │ │ └──┘└──┘│ │ │ └──┘└──┘│ │
│ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
└──────────────┘ └──────────────┘ └──────────────┘
   P = Pod (কন্টেইনারের সবচেয়ে ছোট ইউনিট)
```

### মূল কম্পোনেন্ট:

| কম্পোনেন্ট | কাজ |
|------------|-----|
| **API Server** | সমস্ত যোগাযোগের কেন্দ্রবিন্দু — kubectl এটার সাথে কথা বলে |
| **etcd** | ক্লাস্টারের সব ডাটা সংরক্ষণ (distributed key-value store) |
| **Scheduler** | কোন Pod কোন Node এ চলবে সেটা ঠিক করে |
| **Controller Manager** | ক্লাস্টারের স্টেট desired state এ রাখে |
| **kubelet** | প্রতি Node এ চলে — Pod চালু/বন্ধ করে |
| **kube-proxy** | নেটওয়ার্কিং ও লোড ব্যালেন্সিং |

---

## 📋 Kubernetes Resources

### ১. Pod — সবচেয়ে ছোট ইউনিট

```yaml
# pod.yaml — সাধারণত সরাসরি Pod তৈরি করা হয় না
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    app: my-api
    tier: backend
spec:
  containers:
    - name: api
      image: myapp/api:1.0.0
      ports:
        - containerPort: 3000
      env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
      resources:
        requests:           # ন্যূনতম রিসোর্স
          cpu: "100m"       # 0.1 CPU core
          memory: "128Mi"   # 128 MB RAM
        limits:             # সর্বোচ্চ রিসোর্স
          cpu: "500m"
          memory: "512Mi"
      livenessProbe:        # কন্টেইনার জীবিত কিনা চেক
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 15
        periodSeconds: 10
      readinessProbe:       # ট্রাফিক নেওয়ার জন্য প্রস্তুত কিনা
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
```

### ২. Deployment — সবচেয়ে বেশি ব্যবহৃত

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: my-api
spec:
  replicas: 3                    # ৩টি Pod চলবে সবসময়
  selector:
    matchLabels:
      app: my-api
  strategy:
    type: RollingUpdate          # রোলিং আপডেট (জিরো ডাউনটাইম)
    rollingUpdate:
      maxSurge: 1                # আপডেটের সময় সর্বোচ্চ ১টি অতিরিক্ত Pod
      maxUnavailable: 0          # কোনো Pod অনুপলব্ধ থাকবে না
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: api
          image: myapp/api:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: mongodb-uri
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
```

### ৩. Service — নেটওয়ার্কিং ও লোড ব্যালেন্সিং

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP               # ক্লাস্টারের ভিতরে অ্যাক্সেস
  selector:
    app: my-api                  # deployment এর label ম্যাচ করবে
  ports:
    - protocol: TCP
      port: 80                   # সার্ভিস পোর্ট
      targetPort: 3000           # কন্টেইনার পোর্ট

---
# LoadBalancer — বাইরে থেকে অ্যাক্সেস (ক্লাউডে)
apiVersion: v1
kind: Service
metadata:
  name: api-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - port: 80
      targetPort: 3000

---
# NodePort — ডেভেলপমেন্টে ব্যবহার
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport
spec:
  type: NodePort
  selector:
    app: my-api
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080            # 30000-32767 রেঞ্জে
```

### ৪. Ingress — HTTP রাউটিং

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
        - admin.example.com
      secretName: tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

### ৫. ConfigMap ও Secret

```yaml
# configmap.yaml — সংবেদনশীল নয় এমন কনফিগ
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://api-service;
      }
    }

---
# secret.yaml — সংবেদনশীল ডাটা (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  mongodb-uri: bW9uZ29kYjovL3VzZXI6cGFzc0Bob3N0OjI3MDE3L2Ri  # base64
  jwt-secret: c3VwZXJfc2VjcmV0X2tleQ==                          # base64
  redis-password: cmVkaXNfcGFzcw==
```

### ৬. HorizontalPodAutoscaler — অটো স্কেলিং

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2                 # ন্যূনতম ২টি Pod
  maxReplicas: 20                # সর্বোচ্চ ২০টি Pod
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # CPU ৭০% এর উপরে গেলে স্কেল আপ
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80  # মেমোরি ৮০% এর উপরে গেলে স্কেল আপ
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # ৬০ সেকেন্ড অপেক্ষা
      policies:
        - type: Pods
          value: 2                      # একবারে সর্বোচ্চ ২টি Pod যোগ
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # ৫ মিনিট অপেক্ষা (দ্রুত কমাবে না)
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

---

## 🔬 সম্পূর্ণ প্রোডাকশন সেটআপ উদাহরণ

### PHP Laravel on Kubernetes

```yaml
# laravel-k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: laravel-app

---
# laravel-k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  namespace: laravel-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel
      component: web
  template:
    metadata:
      labels:
        app: laravel
        component: web
    spec:
      initContainers:
        # মাইগ্রেশন চালানো (একবারই)
        - name: migrate
          image: myapp/laravel:latest
          command: ["php", "artisan", "migrate", "--force"]
          envFrom:
            - configMapRef:
                name: laravel-config
            - secretRef:
                name: laravel-secrets
      containers:
        - name: php-fpm
          image: myapp/laravel:latest
          ports:
            - containerPort: 9000
          envFrom:
            - configMapRef:
                name: laravel-config
            - secretRef:
                name: laravel-secrets
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          volumeMounts:
            - name: storage
              mountPath: /var/www/html/storage/app/public
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: laravel-storage
        - name: nginx-config
          configMap:
            name: nginx-config

---
# Queue Worker Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-queue
  namespace: laravel-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: laravel
      component: queue
  template:
    metadata:
      labels:
        app: laravel
        component: queue
    spec:
      containers:
        - name: queue-worker
          image: myapp/laravel:latest
          command: ["php", "artisan", "queue:work", "--sleep=3", "--tries=3"]
          envFrom:
            - configMapRef:
                name: laravel-config
            - secretRef:
                name: laravel-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

---
# Cron Job — Laravel Scheduler
apiVersion: batch/v1
kind: CronJob
metadata:
  name: laravel-scheduler
  namespace: laravel-app
spec:
  schedule: "* * * * *"          # প্রতি মিনিটে
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: scheduler
              image: myapp/laravel:latest
              command: ["php", "artisan", "schedule:run"]
              envFrom:
                - configMapRef:
                    name: laravel-config
                - secretRef:
                    name: laravel-secrets
          restartPolicy: OnFailure
```

---

## 🛠️ প্রয়োজনীয় kubectl কমান্ড

```bash
# ক্লাস্টার তথ্য
kubectl cluster-info
kubectl get nodes

# Deployments
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl rollout status deployment/api-deployment
kubectl rollout undo deployment/api-deployment        # রোলব্যাক!
kubectl scale deployment api-deployment --replicas=5  # স্কেল

# Pods
kubectl get pods
kubectl get pods -o wide                 # বিস্তারিত
kubectl describe pod <pod-name>          # সম্পূর্ণ বিবরণ
kubectl logs <pod-name> -f               # লাইভ লগ
kubectl exec -it <pod-name> -- sh        # শেল অ্যাক্সেস

# Services
kubectl get services
kubectl get ingress

# ডিবাগিং
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods                         # রিসোর্স ব্যবহার
kubectl top nodes
```

---

## ✅ সুবিধা (Pros)

| # | সুবিধা | ব্যাখ্যা |
|---|--------|---------|
| ১ | **অটো-স্কেলিং** | লোড অনুযায়ী Pod বাড়ায়/কমায় |
| ২ | **সেলফ-হিলিং** | Pod ক্র্যাশ করলে অটো রিস্টার্ট |
| ৩ | **জিরো-ডাউনটাইম ডেপ্লয়** | রোলিং আপডেট বিল্ট-ইন |
| ৪ | **সার্ভিস ডিসকভারি** | DNS ভিত্তিক অটো ডিসকভারি |
| ৫ | **মাল্টি-ক্লাউড** | AWS, GCP, Azure সব জায়গায় চলে |
| ৬ | **ডিক্লারেটিভ** | YAML এ desired state লিখুন, K8s বাকিটা করবে |

## ❌ অসুবিধা (Cons)

| # | অসুবিধা | ব্যাখ্যা |
|---|---------|---------|
| ১ | **জটিলতা** | শেখা ও পরিচালনা কঠিন — অনেক কম্পোনেন্ট |
| ২ | **রিসোর্স ওভারহেড** | Control Plane নিজেই রিসোর্স খরচ করে |
| ৩ | **ছোট টিমের জন্য অতিরিক্ত** | ৩-৫ সার্ভিসের জন্য K8s overkill |
| ৪ | **ডিবাগিং কঠিন** | নেটওয়ার্ক, DNS, Volume সমস্যা ট্র্যাক করা কঠিন |
| ৫ | **খরচ** | ম্যানেজড K8s (EKS, GKE) খরচসাপেক্ষ |

---

## 📝 সারসংক্ষেপ

| বিষয় | বিবরণ |
|-------|--------|
| **K8s কী** | কন্টেইনার অর্কেস্ট্রেশন প্ল্যাটফর্ম |
| **মূল ইউনিট** | Pod → Deployment → Service → Ingress |
| **শক্তি** | অটো-স্কেলিং, সেলফ-হিলিং, রোলিং আপডেট |
| **ম্যানেজড** | AWS EKS, Google GKE, Azure AKS |
| **কখন ব্যবহার** | মাইক্রোসার্ভিস, হাই ট্রাফিক, মাল্টি-টিম |
| **বিকল্প** | Docker Compose (ছোট), Docker Swarm (মাঝারি) |
