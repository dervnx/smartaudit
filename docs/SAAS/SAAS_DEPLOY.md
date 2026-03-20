# SaaS 部署方案

## 1. 部署架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户访问                                  │
│                    (HTTPS, 443 端口)                             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx (SLB)                                 │
│                 (负载均衡 + SSL 终止)                            │
│                   ┌────────────────┐                             │
│                   │  Nginx 集群     │                           │
│                   └────────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   SaaS API      │  │   SaaS API      │  │   SaaS API      │
│   Instance 1    │  │   Instance 2    │  │   Instance N    │
│   (容器)         │  │   (容器)         │  │   (容器)         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
          │                    │                    │
          └────────────────────┼────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         数据层                                    │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   PostgreSQL    │     Redis       │         S3 / OSS            │
│   主从集群       │     集群         │         (文件存储)          │
│                 │  (会话/缓存/队列) │                              │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

## 2. Docker 部署

### 2.1 Dockerfile

```dockerfile
# SaaS Backend
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 创建非 root 用户
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2.2 Docker Compose 配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # SaaS API
  saas-api:
    build:
      context: ./saas-backend
      dockerfile: Dockerfile
    container_name: smartaudit-saas-api
    environment:
      - DATABASE_URL=postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@postgres:5432/smartaudit_saas
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
      - PLATFORM_NAME=${PLATFORM_NAME}
    ports:
      - "8001:8000"
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

  # SaaS Frontend
  saas-frontend:
    build:
      context: ./saas-frontend
      dockerfile: Dockerfile
    container_name: smartaudit-saas-frontend
    ports:
      - "3001:80"
    depends_on:
      - saas-api
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: smartaudit-postgres
    environment:
      - POSTGRES_DB=smartaudit_saas
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

  # Nginx (反向代理)
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
      - saas-api
      - saas-frontend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 2.3 Nginx 配置

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;

    # 上传文件大小限制
    client_max_body_size 10M;

    # SaaS API upstream
    upstream saas_backend {
        server saas-api:8000;
        keepalive 32;
    }

    server {
        listen 80;
        server_name _;

        # 重定向到 HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name _;

        # SSL 配置
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # SaaS 管理后台
        location / {
            proxy_pass http://saas-frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # SaaS API
        location /saas/api/ {
            proxy_pass http://saas_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";

            # 超时设置
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # 健康检查
        location /health {
            proxy_pass http://saas_backend;
            access_log off;
        }
    }
}
```

## 3. 环境变量

```bash
# .env.saas

# 数据库
DB_USER=smartaudit
DB_PASSWORD=your_secure_password

# Redis
REDIS_PASSWORD=your_redis_password

# JWT
SECRET_KEY=your_very_long_secret_key_at_least_32_chars

# 平台配置
PLATFORM_NAME=SmartAudit
```

## 4. Kubernetes 部署

### 4.1 Deployment 配置

```yaml
# saas-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smartaudit-saas-api
  namespace: smartaudit
spec:
  replicas: 3
  selector:
    matchLabels:
      app: smartaudit-saas-api
  template:
    metadata:
      labels:
        app: smartaudit-saas-api
    spec:
      containers:
        - name: saas-api
          image: smartaudit/saas-api:latest
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
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
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
```

### 4.2 Service 配置

```yaml
# saas-api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: smartaudit-saas-api
  namespace: smartaudit
spec:
  selector:
    app: smartaudit-saas-api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### 4.3 Ingress 配置

```yaml
# saas-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: smartaudit-saas
  namespace: smartaudit
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - hosts:
        - saas.example.com
      secretName: smartaudit-tls
  rules:
    - host: saas.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: smartaudit-saas-frontend
                port:
                  number: 80
          - path: /saas/api/
            pathType: Prefix
            backend:
              service:
                name: smartaudit-saas-api
                port:
                  number: 80
```

## 5. 数据库初始化

```bash
# 执行数据库迁移
alembic upgrade head

# 或执行初始化脚本
psql -U smartaudit -d smartaudit_saas -f migrations/init_saas.sql
```

## 6. 监控配置

### 6.1 Prometheus 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'smartaudit-saas'
    static_configs:
      - targets: ['saas-api:8000']
    metrics_path: '/metrics'
```

### 6.2 Grafana 看板

关键指标：
- API 请求率
- API 响应时间 (P50/P95/P99)
- 错误率
- 数据库连接数
- Redis 内存使用
- CPU/内存使用率

## 7. 备份策略

| 类型 | 频率 | 保留时间 |
|------|------|----------|
| 全量备份 | 每天凌晨 2 点 | 30 天 |
| 增量备份 | 每小时 | 7 天 |
| 日志备份 | 实时 | 180 天 |

## 8. 灾难恢复

### 8.1 RTO (恢复时间目标)
- 数据库: < 1 小时
- 应用: < 30 分钟

### 8.2 RPO (恢复点目标)
- 数据库: < 1 小时
- 应用: < 5 分钟 (文件系统)

### 8.3 恢复流程

1. 恢复数据库从最新备份
2. 恢复配置文件
3. 启动应用服务
4. 验证服务可用性
