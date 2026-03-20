# 租户部署方案

## 1. 部署模式

### 1.1 SaaS 多租户模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      平台方服务器                                  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   SaaS 服务集群                          │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │    │
│  │  │  Tenant A   │  │  Tenant B   │  │  Tenant C   │     │    │
│  │  │  (数据隔离) │  │  (数据隔离) │  │  (数据隔离) │     │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ PostgreSQL  │  │    Redis    │  │     S3      │             │
│  │  (共享)      │  │   (共享)    │  │   (共享)    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 独立部署模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      租户自有服务器                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   租户服务集群                            │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │    │
│  │  │  API 1      │  │  API 2      │  │  Celery     │     │    │
│  │  │  (容器)     │  │  (容器)     │  │  Workers    │     │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ PostgreSQL  │  │    Redis     │  │    OSS/S3   │             │
│  │  (独立)     │  │  (独立)      │  │  (备份存储) │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Docker 部署

### 2.1 Docker Compose 配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Tenant API
  tenant-api:
    build:
      context: ./tenant-backend
      dockerfile: Dockerfile
    container_name: smartaudit-tenant-api
    environment:
      - DATABASE_URL=postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@postgres:5432/smartaudit
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
      - SAAS_API_URL=${SAAS_API_URL}
      - TENANT_ID=${TENANT_ID}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Celery Worker
  celery-worker:
    build:
      context: ./tenant-backend
      dockerfile: Dockerfile
    container_name: smartaudit-celery-worker
    command: celery -A app.tasks worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@postgres:5432/smartaudit
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # Celery Beat (定时任务)
  celery-beat:
    build:
      context: ./tenant-backend
      dockerfile: Dockerfile
    container_name: smartaudit-celery-beat
    command: celery -A app.tasks beat --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@postgres:5432/smartaudit
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: smartaudit-postgres
    environment:
      - POSTGRES_DB=smartaudit
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    container_name: smartaudit-redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Nginx
  nginx:
    image: nginx:alpine
    container_name: smartaudit-nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - tenant-api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 2.2 Nginx 配置

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;

    client_max_body_size 10M;

    upstream backend {
        server tenant-api:8000;
        keepalive 32;
    }

    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name _;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;

        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";
        }

        location /health {
            proxy_pass http://backend;
            access_log off;
        }
    }
}
```

## 3. 环境变量

```bash
# .env

# 数据库
DB_USER=smartaudit
DB_PASSWORD=your_secure_password

# Redis
REDIS_PASSWORD=your_redis_password

# JWT
SECRET_KEY=your_very_long_secret_key_at_least_32_chars

# SaaS 平台
SAAS_API_URL=https://saas-api.example.com

# 租户标识
TENANT_ID=your_tenant_id
```

## 4. Kubernetes 部署

### 4.1 Deployment 配置

```yaml
# tenant-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartaudit-tenant-api
  namespace: smartaudit-tenant
spec:
  replicas: 3
  selector:
    matchLabels:
      app: smartaudit-tenant-api
  template:
    metadata:
      labels:
        app: smartaudit-tenant-api
    spec:
      containers:
        - name: tenant-api
          image: smartaudit/tenant-api:latest
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: smartaudit-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: smartaudit-secrets
                  key: redis-url
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: smartaudit-secrets
                  key: secret-key
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartaudit-celery-worker
  namespace: smartaudit-tenant
spec:
  replicas: 2
  selector:
    matchLabels:
      app: celery-worker
  template:
    metadata:
      labels:
        app: celery-worker
    spec:
      containers:
        - name: worker
          image: smartaudit/tenant-api:latest
          command: ["celery", "-A", "app.tasks", "worker", "--loglevel=info"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: smartaudit-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: smartaudit-secrets
                  key: redis-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### 4.2 Service 配置

```yaml
# tenant-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: smartaudit-tenant-api
  namespace: smartaudit-tenant
spec:
  selector:
    app: smartaudit-tenant-api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### 4.3 Ingress 配置

```yaml
# tenant-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: smartaudit-tenant
  namespace: smartaudit-tenant
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - hosts:
        - tenant.example.com
      secretName: smartaudit-tls
  rules:
    - host: tenant.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: smartaudit-tenant-api
                port:
                  number: 80
```

### 4.4 PVC 配置

```yaml
# tenant-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: smartaudit-data
  namespace: smartaudit-tenant
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

## 5. 备份存储配置

### 5.1 S3 备份

```yaml
# backup-s3.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backup-credentials
  namespace: smartaudit-tenant
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: AKIA...
  AWS_SECRET_ACCESS_KEY: ...
  AWS_DEFAULT_REGION: us-east-1
```

### 5.2 CronJob 备份

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: smartaudit-backup
  namespace: smartaudit-tenant
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: smartaudit/tenant-api:latest
              command: ["python", "-m", "app.tasks.backup"]
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: smartaudit-secrets
                      key: database-url
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: backup-credentials
                      key: AWS_ACCESS_KEY_ID
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: backup-credentials
                      key: AWS_SECRET_ACCESS_KEY
              volumeMounts:
                - name: backup-storage
                  mountPath: /data/backups
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: smartaudit-backup
          restartPolicy: OnFailure
```

## 6. 监控配置

### 6.1 Prometheus 配置

```yaml
# prometheus.yaml
scrape_configs:
  - job_name: 'smartaudit-tenant'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 6.2 关键指标

| 指标 | 说明 |
|------|------|
| api_request_total | API 请求总数 |
| api_request_duration_seconds | API 响应时间 |
| celery_task_total | Celery 任务总数 |
| celery_task_duration_seconds | Celery 任务执行时间 |
| database_connections | 数据库连接数 |
| redis_memory_usage | Redis 内存使用 |

## 7. 灾难恢复

### 7.1 RTO/RPO 目标

| 类型 | 目标 |
|------|------|
| RTO (恢复时间) | < 30 分钟 |
| RPO (恢复点) | < 1 小时 |

### 7.2 恢复步骤

1. 启动新服务器
2. 部署应用镜像
3. 从 S3 下载最新备份
4. 执行数据库恢复
5. 验证服务可用性
6. 更新 DNS

## 8. 环境要求

### 8.1 最低配置

| 组件 | 最低配置 |
|------|----------|
| CPU | 2 核 |
| 内存 | 4 GB |
| 系统盘 | 50 GB SSD |
| 数据盘 | 100 GB SSD |
| 数据库 | PostgreSQL 15 |
| 缓存 | Redis 7 |

### 8.2 推荐配置

| 组件 | 推荐配置 |
|------|----------|
| CPU | 4 核 |
| 内存 | 8 GB |
| 系统盘 | 100 GB SSD |
| 数据盘 | 200 GB SSD |
| 数据库 | PostgreSQL 15 主从 |
| 缓存 | Redis 7 哨兵 |
| 备份 | S3/OSS |
