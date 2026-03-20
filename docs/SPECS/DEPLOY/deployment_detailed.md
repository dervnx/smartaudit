# 部署详细操作手册

## 1. 环境准备

### 1.1 服务器要求

| 配置项 | 最低配置 | 推荐配置 |
|--------|----------|----------|
| CPU | 4 核 | 8 核 |
| 内存 | 8 GB | 16 GB |
| 系统盘 | 100 GB SSD | 200 GB SSD |
| 数据盘 | 500 GB | 1 TB SSD |
| 操作系统 | Ubuntu 20.04+ / Debian 11+ | Ubuntu 22.04 |

### 1.2 安装基础软件

#### Ubuntu/Debian 系统初始化

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

#### CentOS/RHEL 系统初始化

```bash
# 更新系统包
sudo yum update -y

# 安装基础软件
sudo yum install -y curl wget git vim htop net-tools

# 安装 Docker
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 2. 数据库安装与配置

### 2.1 PostgreSQL 安装（Ubuntu/Debian）

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

### 2.3 MySQL 安装（可选）

```bash
# 安装 MySQL
sudo apt install -y mysql-server

# 启动 MySQL
sudo systemctl start mysql
sudo systemctl enable mysql

# 安全初始化
sudo mysql_secure_installation

# 登录 MySQL
sudo mysql

# 创建数据库和用户
CREATE DATABASE smartaudit CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'smartaudit_user'@'%' IDENTIFIED BY 'YourStrongPassword123!';
GRANT ALL PRIVILEGES ON smartaudit.* TO 'smartaudit_user'@'%';
FLUSH PRIVILEGES;

\q
```

## 3. Redis 安装

### 3.1 Redis 安装（Ubuntu/Debian）

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
