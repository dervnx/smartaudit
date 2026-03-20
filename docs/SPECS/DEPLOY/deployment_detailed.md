# 部署详细操作手册

## 0. MVP 最小化部署（核心审核功能）

> 本章节为 MVP（最小可行产品）快速启动指南，仅包含运行核心审核功能所需的最低配置。

### 0.1 MVP 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      MVP 最小架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────┐        ┌─────────────┐                    │
│   │   Nginx     │───────▶│  FastAPI    │                    │
│   │  (反向代理)  │        │  (应用服务)  │                    │
│   └─────────────┘        └──────┬──────┘                    │
│                                  │                           │
│                    ┌─────────────┼─────────────┐            │
│                    ▼             ▼             ▼            │
│              ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│              │PostgreSQL│  │  Redis  │  │ Celery  │         │
│              │ (主库)   │  │ (缓存)   │  │ (异步)  │         │
│              └─────────┘  └─────────┘  └─────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 0.2 快速启动（Docker Compose）

```bash
# 1. 创建工作目录
mkdir -p /opt/smartaudit && cd /opt/smartaudit

# 2. 创建 docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api

  api:
    image: smartaudit/api:latest
    environment:
      - DATABASE_URL=postgresql://smartaudit:password@db:5432/smartaudit
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/1
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=smartaudit
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=smartaudit
    volumes:
      - pg_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  celery:
    image: smartaudit/api:latest
    command: celery -A app.tasks worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://smartaudit:password@db:5432/smartaudit
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/1

volumes:
  pg_data:
EOF

# 3. 创建 Nginx 配置
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:8000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /api/ {
            proxy_pass http://api/api/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF

# 4. 启动服务
docker-compose up -d

# 5. 验证服务
curl http://localhost/api/v1/health
```

### 0.3 初始化数据库

```bash
# 进入 API 容器
docker-compose exec api bash

# 初始化数据库（创建表结构）
alembic upgrade head

# 创建初始管理员账户
python -m app.scripts.create_admin \
  --username admin \
  --password Admin123! \
  --name "系统管理员"

# 退出容器
exit
```

### 0.4 核心功能验证

```bash
# 1. 登录获取 Token
curl -X POST http://localhost/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Admin123!"}'

# 2. 创建项目
curl -X POST http://localhost/api/v1/projects \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "测试项目",
    "code": "test001",
    "audit_config": {
      "mode": "auto",
      "need_manual_review": false
    }
  }'

# 3. 提交审核数据
curl -X POST http://localhost/api/v1/audit/submit \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "<PROJECT_ID>",
    "input_data": {"name": "张三", "id_card": "110101199001011234"}
  }'

# 4. 查询审核结果
curl -X GET http://localhost/api/v1/audit/pending \
  -H "Authorization: Bearer <TOKEN>"
```

### 0.5 MVP 最小配置要求

| 组件 | 最低配置 | 说明 |
|------|----------|------|
| 服务器 | 2核4G | 仅支持少量并发 |
| PostgreSQL | 15+ | 主数据库 |
| Redis | 7+ | 缓存 + 消息队列 |
| Nginx | latest | 反向代理 |

### 0.6 停止服务

```bash
# 停止服务（保留数据）
docker-compose stop

# 完全清除（删除数据）
docker-compose down -v
```

---

## 1. 环境准备

### 1.1 服务器要求

| 配置项 | 最低配置 | 推荐配置 |
|--------|----------|----------|
| CPU | 4 核 | 8 核 |
| 内存 | 8 GB | 16 GB |
| 系统盘 | 100 GB SSD | 200 GB SSD |
| 数据盘 | 500 GB | 1 TB SSD |
| 操作系统 | Debian 11+ | Debian 12 |

### 1.2 安装基础软件

#### Debian 系统初始化

```bash
# 更新系统包
sudo apt update && sudo apt upgrade -y

# 安装基础软件
sudo apt install -y curl wget git vim htop net-tools supervisor

# 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 启动 Docker 并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 添加当前用户到 docker 组（避免每次用 sudo）
sudo usermod -aG docker $USER
newgrp docker

# 验证 Docker 安装
docker --version
# 输出：Docker version 24.0.x, build xxxxx

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 验证 Docker Compose 安装
docker-compose --version
# 输出：Docker Compose version v2.20.0
```

## 2. 数据库安装与配置

### 2.1 PostgreSQL 安装（Debian）

```bash
# 安装 PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# 启动 PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# 检查 PostgreSQL 状态
sudo systemctl status postgresql

# 切换到 postgres 用户
sudo -u postgres psql

# 在 psql 中执行以下命令：
# 创建数据库
CREATE DATABASE smartaudit;

# 创建用户
CREATE USER smartaudit_user WITH PASSWORD 'YourStrongPassword123!';

# 授权
GRANT ALL PRIVILEGES ON DATABASE smartaudit TO smartaudit_user;

# 退出 psql
\q
```

### 2.2 PostgreSQL 配置

```bash
# 编辑 PostgreSQL 配置
sudo vim /etc/postgresql/14/main/postgresql.conf

# 修改以下配置：
# listen_addresses = '*'
# max_connections = 200
# shared_buffers = 256MB
# effective_cache_size = 1GB
# maintenance_work_mem = 64MB
# checkpoint_completion_target = 0.9
# wal_buffers = 16MB
# default_statistics_target = 100
# random_page_cost = 1.1
# effective_io_concurrency = 200
# work_mem = 6553kB
# min_wal_size = 1GB
# max_wal_size = 4GB

# 编辑访问控制配置
sudo vim /etc/postgresql/14/main/pg_hba.conf

# 添加以下行允许远程连接：
# host    all    all    0.0.0.0/0    md5

# 重启 PostgreSQL
sudo systemctl restart postgresql

# 测试连接
psql -h localhost -U smartaudit_user -d smartaudit
# 输入密码后执行：
SELECT version();
\q
```

## 3. Redis 安装

### 3.1 Redis 安装（Debian）

```bash
# 安装 Redis
sudo apt install -y redis-server

# 启动 Redis
sudo systemctl start redis-server
sudo systemctl enable redis-server

# 配置 Redis 远程访问
sudo vim /etc/redis/redis.conf

# 修改以下配置：
# bind 0.0.0.0
# protected-mode no
# requirepass YourRedisPassword123

# 重启 Redis
sudo systemctl restart redis-server

# 测试 Redis 连接
redis-cli
AUTH YourRedisPassword123
PING
# 输出：PONG
\q
```

## 4. 应用部署

### 4.1 创建项目目录

```bash
# 创建项目目录
sudo mkdir -p /opt/smartaudit
sudo chown -R $USER:$USER /opt/smartaudit

# 创建数据目录
sudo mkdir -p /opt/smartaudit/data/{uploads,backups,logs}
sudo chown -R $USER:$USER /opt/smartaudit/data
```

### 4.2 Docker Compose 部署

#### 创建 docker-compose.yml

```bash
cd /opt/smartaudit
vim docker-compose.yml
```

```yaml
version: '3.8'

services:
  # 后端 API 服务
  api:
    image: smartaudit/api:latest
    container_name: smartaudit-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://smartaudit_user:YourStrongPassword123!@db:5432/smartaudit
      - REDIS_URL=redis://:YourRedisPassword123@redis:6379/0
      - SECRET_KEY=your-secret-key-change-in-production
      - DEBUG=false
    volumes:
      - ./data/uploads:/app/uploads
      - ./data/logs:/app/logs
    depends_on:
      - db
      - redis
    networks:
      - smartaudit-network

  # Celery Worker
  celery:
    image: smartaudit/api:latest
    container_name: smartaudit-celery
    restart: unless-stopped
    command: celery -app app.celery_app worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://smartaudit_user:YourStrongPassword123!@db:5432/smartaudit
      - REDIS_URL=redis://:YourRedisPassword123@redis:6379/0
    volumes:
      - ./data:/app/data
    depends_on:
      - db
      - redis
    networks:
      - smartaudit-network

  # Celery Beat (定时任务)
  celery-beat:
    image: smartaudit/api:latest
    container_name: smartaudit-celery-beat
    restart: unless-stopped
    command: celery -app app.celery_app beat --loglevel=info
    environment:
      - DATABASE_URL=postgresql+asyncpg://smartaudit_user:YourStrongPassword123!@db:5432/smartaudit
      - REDIS_URL=redis://:YourRedisPassword123@redis:6379/0
    depends_on:
      - db
      - redis
    networks:
      - smartaudit-network

  # 前端 Web
  web:
    image: smartaudit/web:latest
    container_name: smartaudit-web
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - smartaudit-network

  # PostgreSQL 数据库
  db:
    image: postgres:15-alpine
    container_name: smartaudit-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=smartaudit
      - POSTGRES_USER=smartaudit_user
      - POSTGRES_PASSWORD=YourStrongPassword123!
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - smartaudit-network

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: smartaudit-redis
    restart: unless-stopped
    command: redis-server --requirepass YourRedisPassword123
    volumes:
      - redisdata:/data
    ports:
      - "6379:6379"
    networks:
      - smartaudit-network

networks:
  smartaudit-network:
    driver: bridge

volumes:
  pgdata:
  redisdata:
```

#### 创建 Nginx 配置

```bash
vim nginx.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    upstream api_backend {
        server api:8000;
    }

    server {
        listen 80;
        server_name _;

        # 重定向到 HTTPS（生产环境启用）
        # return 301 https://$host$request_uri;

        client_max_body_size 100M;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # WebSocket 支持（如需要）
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        location /uploads/ {
            alias /app/uploads/;
            expires 7d;
            add_header Cache-Control "public, immutable";
        }
    }

    # HTTPS 配置（生产环境启用）
    # server {
    #     listen 443 ssl http2;
    #     server_name your-domain.com;
    #
    #     ssl_certificate /etc/nginx/ssl/cert.pem;
    #     ssl_certificate_key /etc/nginx/ssl/key.pem;
    #     ssl_protocols TLSv1.2 TLSv1.3;
    #     ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    #
    #     # 其他配置同上
    # }
}
```

#### 启动服务

```bash
# 拉取镜像（如果有镜像仓库）
# docker-compose pull

# 后台启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f api
docker-compose logs -f web

# 查看所有服务日志
docker-compose logs -f
```

### 4.3 初始化数据库

```bash
# 进入 API 容器
docker-compose exec api bash

# 执行数据库迁移
alembic upgrade head

# 创建初始数据（如需要）
python -m app.scripts.init_data

# 退出容器
exit
```

### 4.4 验证部署

```bash
# 检查 API 是否正常
curl http://localhost:8000/health

# 检查 Web 是否正常
curl http://localhost/

# 查看容器状态
docker-compose ps
```

## 5. 独立部署版本生成（SaaS 平台操作）

### 5.1 生成独立部署包

```bash
# 登录 SaaS 平台管理后台

# 1. 选择目标租户
# 进入 租户管理 -> 选择租户 -> 点击"生成独立部署版本"

# 2. 选择基础版本
# 选择要部署的版本（如 v1.2.0）

# 3. 配置租户信息
# - 租户 Logo
# - 租户名称
# - 自定义域名（可选）

# 4. 点击"生成部署包"
# 系统会自动打包：
# - Docker 镜像
# - docker-compose.yml
# - 环境变量配置
# - 数据库初始化脚本
# - 部署说明文档

# 5. 下载部署包
# 生成的部署包会保存在：
# /opt/smartaudit-deployments/{tenant_id}/
```

### 5.2 租户独立部署包结构

```
smartaudit-deployment-{tenant_code}-{version}/
├── docker-compose.yml      # Docker Compose 配置
├── .env                     # 环境变量配置
├── init-db.sh              # 数据库初始化脚本
├── deploy.sh               # 一键部署脚本
├── nginx/
│   └── nginx.conf          # Nginx 配置
├── ssl/                    # SSL 证书目录
├── backup/
│   └── config.yaml         # 备份配置
├── upgrade/
│   └── upgrade.sh          # 升级脚本
└── README.md               # 部署说明
```

### 5.3 客户服务器部署独立版本

```bash
# 1. 将部署包上传到客户服务器
scp smartaudit-deployment-tenant_a-v1.2.0.tar.gz user@customer-server:/opt/

# 2. SSH 登录客户服务器
ssh user@customer-server

# 3. 解压部署包
cd /opt
tar -xzf smartaudit-deployment-tenant_a-v1.2.0.tar.gz
cd smartaudit-deployment-tenant_a-v1.2.0

# 4. 配置环境变量
vim .env
# 修改以下配置：
# DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/smartaudit
# REDIS_URL=redis://:password@localhost:6379/0
# SECRET_KEY=生成新的密钥

# 5. 执行部署脚本
chmod +x deploy.sh
./deploy.sh

# 6. 验证部署
curl http://localhost:8000/health
curl http://localhost/

# 7. 配置开机自启
sudo systemctl enable smartaudit
```

## 6. 升级操作

### 6.1 SaaS 平台升级

```bash
# 1. 在 SaaS 平台创建新版本
# 进入 版本管理 -> 发布新版本
# 上传新版本镜像
# 填写版本信息

# 2. 选择要升级的租户
# 进入 租户管理 -> 选择租户 -> 版本管理
# 点击"检查更新"
# 显示有新版本后点击"升级"

# 3. 租户确认升级
# 租户管理员会收到升级通知
# 确认后系统自动执行升级
```

### 6.2 独立部署版本升级

```bash
# 1. 下载新版本部署包（从 SaaS 平台）

# 2. 上传到客户服务器
scp smartaudit-deployment-tenant_a-v1.3.0.tar.gz user@customer-server:/opt/

# 3. 登录客户服务器
ssh user@customer-server

# 4. 停止当前服务
cd /opt/smartaudit
docker-compose down

# 5. 解压新版本
cd /opt
tar -xzf smartaudit-deployment-tenant_a-v1.3.0.tar.gz
cd smartaudit-deployment-tenant_a-v1.3.0

# 6. 备份当前版本（重要！）
cd /opt
mv smartaudit smartaudit_backup_$(date +%Y%m%d)

# 7. 执行升级
chmod +x upgrade.sh
./upgrade.sh

# 8. 验证升级
curl http://localhost:8000/health
docker-compose ps
```

### 6.3 升级脚本示例

```bash
#!/bin/bash
# upgrade.sh

set -e

echo "开始升级 SmartAudit..."

# 加载环境变量
export $(cat .env | grep -v '^#' | xargs)

# 创建备份
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# 备份数据库
echo "备份数据库..."
docker-compose exec -T db pg_dump -U smartaudit_user smartaudit > $BACKUP_DIR/db_backup.sql

# 备份配置文件
cp .env $BACKUP_DIR/
cp -r nginx $BACKUP_DIR/

# 拉取新镜像
echo "拉取新镜像..."
docker-compose pull api
docker-compose pull celery
docker-compose pull web

# 更新服务
echo "更新服务..."
docker-compose up -d

# 等待服务启动
echo "等待服务启动..."
sleep 30

# 执行数据库迁移
echo "执行数据库迁移..."
docker-compose exec -T api alembic upgrade head

# 清理旧镜像
echo "清理旧镜像..."
docker image prune -f

echo "升级完成！"
echo "请访问 http://localhost/ 验证"
```

## 7. 备份与恢复

### 7.1 数据库备份

```bash
# 创建备份脚本
vim /opt/smartaudit/backup-db.sh
```

```bash
#!/bin/bash
# backup-db.sh

BACKUP_DIR="/opt/smartaudit/data/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/smartaudit_backup_$DATE.sql"

mkdir -p $BACKUP_DIR

# 执行备份
docker-compose exec -T db pg_dump -U smartaudit_user smartaudit > $BACKUP_FILE

# 压缩备份
gzip $BACKUP_FILE

# 计算校验码
sha256sum ${BACKUP_FILE}.gz > ${BACKUP_FILE}.gz.sha256

# 删除 30 天前的备份
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.sha256" -mtime +30 -delete

echo "备份完成: ${BACKUP_FILE}.gz"
```

```bash
# 添加执行权限
chmod +x /opt/smartaudit/backup-db.sh

# 测试备份
/opt/smartaudit/backup-db.sh

# 配置定时任务
crontab -e
# 添加以下行：
# 每天凌晨 2 点执行备份
# 0 2 * * * /opt/smartaudit/backup-db.sh >> /opt/smartaudit/data/logs/backup.log 2>&1
```

### 7.2 数据恢复

```bash
# 停止服务
cd /opt/smartaudit
docker-compose down

# 选择要恢复的备份文件
BACKUP_FILE="/opt/smartaudit/data/backups/smartaudit_backup_20240101_020000.sql.gz"

# 解压备份
gunzip -k $BACKUP_FILE
UNCOMPRESSED_FILE="${BACKUP_FILE%.gz}"

# 恢复数据库
docker-compose exec -T db psql -U smartaudit_user -d smartaudit < $UNCOMPRESSED_FILE

# 删除解压的文件
rm $UNCOMPRESSED_FILE

# 启动服务
docker-compose up -d

echo "数据恢复完成"
```

## 8. 监控与日志

### 8.1 日志配置

```bash
# 创建日志目录
mkdir -p /opt/smartaudit/data/logs/{api,web,nginx,celery}

# 配置日志轮转
vim /etc/logrotate.d/smartaudit
```

```
/opt/smartaudit/data/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 $USER $USER
    sharedscripts
    postrotate
        docker-compose -f /opt/smartaudit/docker-compose.yml restart api web celery
    endscript
}
```

### 8.2 查看日志

```bash
# 查看 API 日志
docker-compose logs -f api

# 查看所有服务日志
docker-compose logs -f

# 查看特定容器的错误日志
docker-compose logs --tail=100 --since="1h" api | grep ERROR

# 导出日志
docker-compose logs api > /tmp/api_logs_$(date +%Y%m%d).log
```

### 8.3 监控脚本

```bash
# 创建监控脚本
vim /opt/smartaudit/monitor.sh
```

```bash
#!/bin/bash
# monitor.sh

# 检查 API 服务
API_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health)
if [ "$API_STATUS" != "200" ]; then
    echo "[ALERT] API 服务异常，HTTP 状态码: $API_STATUS"
    # 发送告警通知（可接入钉钉/企业微信/邮件等）
fi

# 检查数据库连接
DB_STATUS=$(docker-compose exec -T db pg_isready -U smartaudit_user)
if [ "$DB_STATUS" != "" ]; then
    echo "[ALERT] 数据库连接异常"
fi

# 检查磁盘空间
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 85 ]; then
    echo "[ALERT] 磁盘空间使用率: ${DISK_USAGE}%"
fi

# 检查内存使用
MEM_USAGE=$(free | awk 'NR==2 {printf "%.0f", $3/$2 * 100}')
if [ "$MEM_USAGE" -gt 85 ]; then
    echo "[ALERT] 内存使用率: ${MEM_USAGE}%"
fi

echo "监控检查完成: $(date)"
```

```bash
# 添加执行权限
chmod +x /opt/smartaudit/monitor.sh

# 配置定时监控（每 5 分钟执行一次）
crontab -e
# */5 * * * * /opt/smartaudit/monitor.sh >> /opt/smartaudit/data/logs/monitor.log 2>&1
```

## 9. 故障排除

### 9.1 常见问题

#### 服务无法启动

```bash
# 1. 检查端口占用
netstat -tlnp | grep 8000

# 2. 查看详细日志
docker-compose logs api

# 3. 检查环境变量
docker-compose exec api env | grep -E "DATABASE|REDIS"

# 4. 检查数据库连接
docker-compose exec api python -c "from app.db.database import engine; print('DB OK')"
```

#### 前端无法访问

```bash
# 1. 检查 Nginx 日志
docker-compose logs web

# 2. 检查 Nginx 配置
docker-compose exec web nginx -t

# 3. 检查静态文件
ls -la /opt/smartaudit/data/uploads/
```

#### 数据库连接失败

```bash
# 1. 检查 PostgreSQL 状态
docker-compose ps db
docker-compose logs db

# 2. 测试数据库连接
docker-compose exec db psql -U smartaudit_user -d smartaudit

# 3. 检查连接字符串
echo $DATABASE_URL
```

### 9.2 重置服务

```bash
# 完全重置（谨慎使用，会删除所有数据）
cd /opt/smartaudit
docker-compose down -v  # -v 会删除数据卷
docker-compose up -d
docker-compose exec api alembic upgrade head
```

## 10. 安全配置

### 10.1 防火墙配置

```bash
# 只开放必要端口
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS
sudo ufw enable
sudo ufw status
```

### 10.2 SSL 证书配置

```bash
# 使用 Let's Encrypt 免费证书
sudo apt install -y certbot python3-certbot-nginx

# 获取证书
sudo certbot --nginx -d your-domain.com

# 自动续期
sudo certbot renew --dry-run

# 配置自动续期 Cron
crontab -e
# 0 0 * * * certbot renew --quiet
```

### 10.3 修改默认密码

```bash
# 修改数据库密码
docker-compose exec db psql -U smartaudit_user -d smartaudit
ALTER USER smartaudit_user WITH PASSWORD 'NewStrongPassword123!';
\q

# 更新环境变量
vim /opt/smartaudit/.env
# 修改 POSTGRES_PASSWORD 和 DATABASE_URL

# 修改 Redis 密码
# 编辑 docker-compose.yml 中的 Redis 配置

# 重启服务
docker-compose restart
```

---

## 11. 多地多中心部署

### 11.1 地域规划

| 地域 | 机房 | 角色 | 网络延迟 |
|------|------|------|----------|
| 华北 | 北京 | 主中心 | < 5ms |
| 华东 | 上海 | 同城备中心 | < 30ms |
| 华南 | 广州 | 异地灾备 | < 50ms |

### 11.2 主中心部署

```bash
# 1. 在主中心服务器初始化
ssh root@bj-master-01
cd /opt/smartaudit

# 2. 克隆部署仓库
git clone https://github.com/your-org/smartaudit-deploy.git
cd smartaudit-deploy/multi-region/primary

# 3. 配置地域标识
cat > .env << EOF
REGION_ID=cn-north
REGION_NAME=华北
REGION_PRIORITY=1
ROLE=primary
EOF

# 4. 配置数据库主从复制
cat >> docker-compose.yml << EOF
db-primary:
  image: postgres:15
  environment:
    POSTGRES_REPLICATION: primary
    POSTGRES_REPLICATION_USER: replicator
    POSTGRES_REPLICATION_PASSWORD: replication_password
  command: postgres -c wal_level=logical -c max_wal_senders=4

db-replica:
  image: postgres:15
  environment:
    POSTGRES_REPLICATION: replica
    POSTGRES_REPLICATION_USER: replicator
    POSTGRES_REPLICATION_PASSWORD: replication_password
EOF

# 5. 配置 Redis 主从
cat >> docker-compose.yml << EOF
redis-master:
  image: redis:7
  command: redis-server --appendonly yes --replica-read-only yes

redis-slave:
  image: redis:7
  command: redis-server --replicaof redis-master 6379
EOF

# 6. 启动主中心
docker-compose up -d
```

### 11.3 同城备中心部署

```bash
# 1. 在备中心服务器初始化
ssh root@sh-standby-01
cd /opt/smartaudit

# 2. 克隆部署仓库
git clone https://github.com/your-org/smartaudit-deploy.git
cd smartaudit-deploy/multi-region/standby

# 3. 配置地域标识
cat > .env << EOF
REGION_ID=cn-east
REGION_NAME=华东
REGION_PRIORITY=2
ROLE=standby
PRIMARY_REGION=cn-north
EOF

# 4. 启动备中心（会自动同步主中心数据）
docker-compose up -d
```

### 11.4 配置全局负载均衡

```nginx
# /etc/nginx/conf.d/gslb.conf

upstream smartaudit_backend {
    # 主中心（华北）
    server bj-master-01.smartaudit.com:443 weight=100;

    # 同城备中心（华东）
    server sh-standby-01.smartaudit.com:443 weight=50 backup;

    # 异地灾备（华南，冷备）
    server gz-dr-01.smartaudit.com:443 weight=10 backup;
}

server {
    listen 443 ssl;
    server_name api.smartaudit.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    location / {
        proxy_pass https://smartaudit_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Region-ID $http_x_region_id;

        # 健康检查配置
        proxy_connect_timeout 5s;
        proxy_next_upstream error timeout http_502;
    }
}

# GeoIP 地域路由
geo $region {
    default cn-north;
    31.192.0.0/16 cn-east;    # 上海 IP 段
    58.192.0.0/16 cn-south;   # 广州 IP 段
}

server {
    listen 443 ssl;
    server_name ~^(?<region>[a-z]+)-api\.smartaudit\.com$;

    # 根据地域路由到对应中心
    if ($region = cn-east) {
        proxy_pass https://sh-standby-01.smartaudit.com;
    }
    if ($region = cn-south) {
        proxy_pass https://gz-dr-01.smartaudit.com;
    }
}
```

### 11.5 配置数据库流复制

```bash
# 主中心创建复制用户
docker exec -it smartaudit-db-primary psql -U postgres -c \
  "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replication_password';"

# 主中心配置 pg_hba.conf 允许复制
echo "host replication replicator 10.0.1.0/24 md5" >> /var/lib/postgresql/data/pg_hba.conf

# 主中心配置 postgresql.conf
echo "wal_level = logical" >> /var/lib/postgresql/data/postgresql.conf
echo "max_wal_senders = 4" >> /var/lib/postgresql/data/postgresql.conf
echo "wal_keep_size = 1GB" >> /var/lib/postgresql/data/postgresql.conf

# 重启主中心数据库
docker exec -it smartaudit-db-primary pg_ctl restart

# 备中心创建复制槽（备中心执行）
docker exec -it smartaudit-db-replica psql -U postgres -c \
  "SELECT * FROM pg_create_physical_replication_slot('standby_slot');"

# 备中心启动复制
docker exec -it smartaudit-db-replica psql -U postgres -c \
  "SELECT pg_start_replication(slot_name => 'standby_slot', logical => false);"
```

### 11.6 配置 Redis 哨兵集群

```python
# redis-sentinel.conf
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 30000
sentinel parallel-syncs mymaster 1

sentinel announce-ip "10.0.2.10"
sentinel announce-port 26379
```

```bash
# 启动哨兵节点
docker run -d --name redis-sentinel \
  -v /opt/redis/sentinel.conf:/etc/redis/sentinel.conf \
  redis:7 redis-sentinel /etc/redis/sentinel.conf
```

### 11.7 跨地域数据同步脚本

```bash
#!/bin/bash
# sync_data.sh - 跨地域数据同步脚本

set -e

PRIMARY_REGION="cn-north"
BACKUP_REGION="cn-east"
CURRENT_REGION=${REGION_ID:-cn-north}

if [ "$CURRENT_REGION" = "$PRIMARY_REGION" ]; then
    echo "主中心：执行数据同步任务"

    # 导出关键数据
    docker exec smartaudit-db-primary pg_dump -U postgres smartaudit \
      --format=custom \
      --file=/tmp/sync_data.dump

    # 上传到对象存储
    ossutil cp /tmp/sync_data.dump oss://smartaudit-backup/sync/

    echo "同步任务完成，等待备中心拉取"
else
    echo "备中心：从对象存储拉取数据"

    # 从对象存储下载
    ossutil cp oss://smartaudit-backup/sync/sync_data.dump /tmp/

    # 应用增量数据
    docker exec -i smartaudit-db-replica pg_restore -U postgres -d smartaudit \
      --data-only \
      /tmp/sync_data.dump

    echo "数据同步完成"
fi
```

### 11.8 健康检查与故障切换

```bash
#!/bin/bash
# health_check.sh - 健康检查脚本

set -e

CHECK_INTERVAL=5
FAIL_THRESHOLD=3
PRIMARY_API="http://bj-master-01:8000/health"
STANDBY_API="http://sh-standby-01:8000/health"

check_service() {
    local url=$1
    curl -sf "$url" > /dev/null 2>&1
    return $?
}

# 检查主中心
fail_count=0
while ! check_service "$PRIMARY_API"; do
    ((fail_count++))
    echo "主中心不可用 (${fail_count}/${FAIL_THRESHOLD})"

    if [ $fail_count -ge $FAIL_THRESHOLD ]; then
        echo "触发故障切换！"
        # 通知 DNS 切换
        curl -X POST "http://dns-manager/api/failover" \
          -d '{"from": "cn-north", "to": "cn-east"}'
        exit 1
    fi
    sleep $CHECK_INTERVAL
done

echo "主中心正常"
```

### 11.9 切换演练流程

```bash
# 1. 演练前检查
echo "=== 切换演练前检查 ==="
# - 确认数据同步延迟 < 1 分钟
# - 确认备中心服务正常
# - 确认 DNS 切换脚本可用

# 2. 执行切换
echo "=== 执行切换演练 ==="
# 停止主中心服务
ssh root@bj-master-01 "docker-compose down"

# 等待检测（5秒 x 3次）
sleep 15

# 验证备中心接管
curl https://sh-standby-01.smartaudit.com/api/v1/health

# 3. 回切
echo "=== 执行回切 ==="
# 恢复主中心
ssh root@bj-master-01 "docker-compose up -d"

# 等待数据追平
sleep 60

# 验证数据一致性
ssh root@bj-master-01 "/opt/smartaudit/scripts/verify_data.sh"

# 切回主中心
curl -X POST "http://dns-manager/api/failover" \
  -d '{"from": "cn-east", "to": "cn-north"}'

# 4. 演练后验证
echo "=== 演练后验证 ==="
# - 验证所有数据完整性
# - 验证所有服务状态
# - 生成演练报告
```

### 11.10 多中心部署检查清单

| 检查项 | 主中心 | 同城备 | 异地灾备 |
|--------|--------|--------|----------|
| 服务启动 | □ | □ | □ |
| 数据库复制 | □ | □ | □ |
| Redis 复制 | □ | □ | □ |
| Nginx 健康检查 | □ | □ | □ |
| DNS 路由 | □ | □ | □ |
| 告警配置 | □ | □ | □ |
| 备份策略 | □ | □ | □ |
| 切换演练 | □ | □ | □ |

