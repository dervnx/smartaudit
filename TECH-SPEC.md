# 智核 SmartAudit 技术规范文档

---

## 目录

1. [技术架构](#1-技术架构)
2. [技术选型](#2-技术选型)
3. [数据库设计](#3-数据库设计)
4. [API 接口设计](#4-api-接口设计)
5. [前端项目规范](#5-前端项目规范)
6. [后端项目规范](#6-后端项目规范)
7. [安全规范](#7-安全规范)
8. [部署方案](#8-部署方案)
9. [消息队列设计](#9-消息队列设计)
10. [系统版本管理](#10-系统版本管理)
11. [备份管理](#11-备份管理)
12. [开发规范](#12-开发规范)
13. [MVP 版本规划](#13-mvp-版本规划)

---

## 1. 技术架构

### 1.1 系统架构图（多租户 SaaS 模式）

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                  │
├──────────┬──────────┬──────────┬──────────┬──────────────────────┤
│  PC Web  │  Mobile  │  Mini    │  Native  │                      │
│  React   │   H5     │ Program │   App    │                      │
│  +Vite   │  React   │  React   │ React    │                      │
│          │ +Vite    │ Native  │ Native   │                      │
└──────────┴──────────┴──────────┴──────────┴──────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx / API Gateway                          │
│                   (负载均衡、SSL、路由分发)                         │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   FastAPI       │  │   FastAPI       │  │   FastAPI       │
│   Instance 1    │  │   Instance 2    │  │   Instance N    │
│   (主服务)       │  │   (从服务)       │  │   (水平扩展)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
          │                    │                    │
          └────────────────────┼────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         数据层                                    │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   PostgreSQL    │     Redis       │        MinIO / S3           │
│   (主数据库)     │   (缓存/会话/MQ) │     (文件存储)             │
│   主从复制       │   哨兵/集群      │                            │
└─────────────────┴─────────────────┴─────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      第三方接口层                                  │
│     (身份证验证 / 学历验证 / 学位验证 / 银行四要素 / 运营商验证)     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 前后端分离架构

```
┌─────────────────────┐         ┌─────────────────────┐
│    React Frontend   │  HTTP   │   Python FastAPI    │
│                     │◄───────►│                     │
│  - Tailwind CSS     │  JSON   │  - Pydantic         │
│  - React Router     │  REST   │  - SQLAlchemy       │
│  - Zustand          │         │  - Alembic          │
│  - Axios/TanStack   │         │  - uvicorn/gunicorn │
└─────────────────────┘         └─────────────────────┘
                                              │
                                              ▼
                                    ┌─────────────────────┐
                                    │    PostgreSQL       │
                                    │    - 租户数据隔离    │
                                    │    - 行级安全策略    │
                                    └─────────────────────┘
```

### 1.3 独立部署模式架构

对于需要独立部署的租户，提供完整的私有化部署方案：

```
┌─────────────────────────────────────────────────────────────────┐
│                      租户独立部署环境                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────────────────────────────┐   │
│  │   负载均衡   │───►│         Nginx / API Gateway          │   │
│  └─────────────┘    └─────────────────────────────────────┘   │
│                                      │                           │
│              ┌───────────────────────┼───────────────────────┐   │
│              ▼                       ▼                       ▼   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   FastAPI       │  │   FastAPI       │  │   FastAPI       │ │
│  │   主服务实例     │  │   从服务实例     │  │   Worker 任务   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│              │                       │               │           │
│              └───────────────────────┼───────────────┘           │
│                                      ▼                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   PostgreSQL    │  │     Redis       │  │     MinIO       │ │
│  │   主从集群       │  │   (缓存/会话/MQ)│  │   (文件存储)     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    备份存储 (独立 S3/OSS)                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 部署模式对比

| 特性 | SaaS 多租户模式 | 独立部署模式 |
|------|----------------|-------------|
| 架构 | 共享资源 | 独立资源 |
| 部署位置 | 平台方服务器 | 租户自有服务器 |
| 数据隔离 | 行级隔离 | 独立数据库 |
| 系统升级 | 平台统一升级 | 租户自主升级 |
| 备份 | 平台统一管理 | 租户自主管理 |
| 运维成本 | 低（平台托管） | 高（自有运维） |
| 适用场景 | 中小客户 | 大型客户/政府/金融 |

---

## 2. 技术选型

### 2.1 技术栈总览

| 层级 | 技术选型 | 版本 | 说明 |
|------|----------|------|------|
| **前端框架** | React | 18.x | 组件化开发 |
| **前端构建** | Vite | 5.x | 快速构建工具 |
| **UI 框架** | Tailwind CSS | 3.x | 原子化 CSS |
| **状态管理** | Zustand | 4.x | 轻量状态管理 |
| **路由** | React Router | 6.x | SPA 路由 |
| **HTTP 客户端** | Axios / TanStack Query | - | 数据请求/缓存 |
| **表单处理** | React Hook Form + Zod | - | 表单验证 |
| **后端框架** | FastAPI | 0.109+ | 高性能 ASGI |
| **Python 版本** | Python | 3.11+ | - |
| **ORM** | SQLAlchemy | 2.x | 数据库 ORM |
| **数据验证** | Pydantic | 2.x | 数据校验 |
| **数据库** | PostgreSQL | 15+ | 主数据库 |
| **缓存/消息队列** | Redis | 7+ | 缓存/会话/RedisMQ |
| **消息队列** | Redis Queue / Celery | - | 异步任务队列 |
| **文件存储** | MinIO / S3 | - | 文件对象存储 |
| **定时任务** | APScheduler | - | 定时任务 |
| **API 文档** | OpenAPI/Swagger | 3.0 | 自动文档 |
| **ORM** | SQLAlchemy | 2.x | 数据库 ORM |
| **数据验证** | Pydantic | 2.x | 数据校验 |
| **数据库** | PostgreSQL | 15+ | 主数据库 |
| **缓存** | Redis | 7+ | 缓存/会话 |
| **文件存储** | MinIO / S3 | - | 文件对象存储 |
| **定时任务** | APScheduler | - | 定时任务 |
| **API 文档** | OpenAPI/Swagger | 3.0 | 自动文档 |

### 2.2 前端依赖

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.22.0",
    "zustand": "^4.5.0",
    "axios": "^1.6.0",
    "@tanstack/react-query": "^5.0.0",
    "react-hook-form": "^7.50.0",
    "zod": "^3.22.0",
    "@hookform/resolvers": "^3.3.0",
    "dayjs": "^1.11.0",
    "lodash-es": "^4.17.21",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "class-variance-authority": "^0.7.0",
    "lucide-react": "^0.330.0",
    "@radix-ui/react-*": "latest",
    "sonner": "^1.4.0",
    "recharts": "^2.12.0"
  },
  "devDependencies": {
    "vite": "^5.1.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "@types/react": "^18.2.0",
    "@types/node": "^20.11.0",
    "typescript": "^5.3.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "eslint": "^8.56.0",
    "prettier": "^3.2.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.2.0"
  }
}
```

### 2.3 后端依赖

```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "fastapi==0.109.2",
    "uvicorn[standard]==0.27.1",
    "sqlalchemy==2.0.25",
    "asyncpg==0.29.0",
    "alembic==1.13.1",
    "pydantic==2.6.1",
    "pydantic-settings==2.1.0",
    "python-jose[cryptography]==3.3.0",
    "passlib[bcrypt]==1.7.4",
    "python-multipart==0.0.9",
    "redis==5.0.1",
    "aioredis==2.0.1",
    "celery[redis]==5.3.6",
    "APScheduler==3.10.4",
    "httpx==0.26.0",
    "python-json-logger==2.0.7",
    "sentry-sdk==1.41.0",
    "minio==7.2.3",
    "openai==1.12.0",
    "tenacity==8.2.3",
]

[dev-dependencies]
pytest==8.0.0
pytest-asyncio==0.23.4
pytest-cov==4.1.0
httpx==0.26.0
factory-boy==3.3.0
faker==22.6.0
```

---

## 3. 数据库设计

### 3.1 ER 图（实体关系）

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    tenant    │       │   user       │       │    role      │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │──┐    │ id (PK)      │       │ id (PK)      │
│ name         │  │    │ tenant_id(FK)│───────┤ name         │
│ code         │  │    │ username     │       │ code         │
│ logo_url     │  │    │ password     │       │ description  │
│ contact      │  │    │ name         │       │ tenant_id(FK)│
│ phone        │  │    │ email        │       │ is_system    │
│ email        │  │    │ phone        │       └──────┬───────┘
│ address      │  │    │ avatar_url   │              │
│ app_id       │  │    │ status       │       ┌──────┴───────┐
│ app_secret   │  │    │ last_login   │       │              │
│ status       │  │    │ created_at   │       ▼              │
│ created_at   │  │    │ updated_at   │  ┌────────────┐      │
└──────────────┘  │    └──────┬───────┘  │user_role   │      │
                  │           │          ├────────────┤      │
                  │           │          │ user_id(FK)│──────┘
                  │           │          │ role_id(FK)│──────┐
                  │           │          └────────────┘      │
                  │           │                             │
                  │           │          ┌────────────┐      │
                  │           └──────────│permission  │      │
                  │                      ├────────────┤      │
                  │                      │ id (PK)    │      │
                  │                      │ code       │      │
                  │                      │ name       │      │
                  │                      │ module     │      │
                  │                      │ tenant_id  │◄─────┘
                  │                      └────────────┘
                  │                             ▲
                  │                      ┌──────┴───────┐
                  │                      │ role_permission│
                  │                      ├────────────┤
                  │                      │ role_id(FK) │
                  │                      │ perm_id(FK) │
                  │                      │ is_granted  │
                  │                      └─────────────┘
                  │
                  │    ┌──────────────┐       ┌──────────────┐
                  │    │   project    │       │    rule      │
                  │    ├──────────────┤       ├──────────────┤
                  └───►│ id (PK)      │       │ id (PK)      │
                       │ tenant_id(FK)│       │ tenant_id(FK)│
                       │ name         │       │ name         │
                       │ code         │       │ description  │
                       │ description  │       │ conditions   │
                       │ fields_def   │       │ result_pass  │
                       │ rule_ids     │       │ result_reject│
                       │ need_manual  │       │ priority     │
                       │ status       │       │ status       │
                       │ created_at   │       │ created_at   │
                       └──────┬───────┘       └──────┬───────┘
                              │                      │
                              │              ┌───────┴───────┐
                              │              │ project_rule   │
                              │              ├────────────┤
                              │              │ project_id(FK)│
                              │              │ rule_id (FK)  │
                              │              └──────────────┘
                              │
                              ▼
                       ┌──────────────┐       ┌──────────────┐
                       │   audit_data │       │  task        │
                       ├──────────────┤       ├──────────────┤
                       │ id (PK)      │       │ id (PK)      │
                       │ tenant_id(FK)│──────►│ tenant_id(FK)│
                       │ project_id(FK)│      │ auditor_id(FK)│◄────┐
                       │ input_data   │       │ data_id (FK) │      │
                       │ rule_result  │       │ status       │      │
                       │ auto_result  │       │ type         │      │
                       │ final_result │       │ result       │      │
                       │ auditor_id   │       │ result_note  │
                       │ audit_time   │       │ assigned_at  │
                       │ need_recheck │       │ started_at   │
                       │ recheck_id   │       │ completed_at │
                       │ recheck_time │       │ created_at   │
                       │ status       │       └──────────────┘
                       │ created_at   │              ▲
                       └──────┬───────┘              │
                              │                      │
                              ▼                      │
                       ┌──────────────┐              │
                       │ third_party  │              │
                       │ _config      │              │
                       ├──────────────┤              │
                       │ id (PK)      │              │
                       │ tenant_id(FK)│──────────────┘
                       │ name         │
                       │ type         │
                       │ url          │
                       │ method       │
                       │ headers      │
                       │ auth_config  │
                       │ timeout      │
                       │ retry_count  │
                       │ status       │
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │ third_party   │
                       │ _log          │
                       ├──────────────┤
                       │ id (PK)      │
                       │ tenant_id(FK)│
                       │ config_id(FK)│
                       │ user_id(FK)  │
                       │ request_time │
                       │ request_data │
                       │ response_data│
                       │ status       │
                       │ error_msg    │
                       │ duration_ms  │
                       └──────────────┘

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ login_log    │       │system_version│       │tenant_version│
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │       │ id (PK)      │
│ user_id (FK) │       │ version      │       │ tenant_id(FK)│
│ tenant_id(FK)│◄──────│ version_code │       │ cur_version  │
│ login_time   │       │ release_type │       │ last_check   │
│ login_ip     │       │ release_notes│       │ last_update  │
│ device_type  │       │ is_mandatory │       │ auto_update  │
│ device_id    │       │ download_url │       │ update_status│
│ os           │       │ checksum     │       └──────────────┘
│ browser      │       └──────────────┘               ▲
│ login_result │                                       │
│ fail_reason  │       ┌──────────────┐               │
└──────────────┘       │ backup_rule  │               │
                       ├──────────────┤               │
                       │ id (PK)      │               │
                       │ tenant_id(FK)│───────────────┘
                       │ name         │       ┌──────────────┐
                       │ backup_type  │       │backup_record │
                       │ schedule_cron│       ├──────────────┤
                       │ retention_days│      │ id (PK)      │
                       │ storage_loc  │       │ tenant_id(FK)│
                       │ is_enabled   │       │ rule_id(FK)  │
                       │ last_backup  │       │ backup_type  │
                       │ next_backup  │       │ status       │
                       └──────────────┘       │ file_path    │
                                              │ file_size    │
                                              │ started_at   │
                                              │ completed_at │
                                              └──────────────┘

┌──────────────┐
│upgrade_record│
├──────────────┤
│ id (PK)      │
│ tenant_id(FK)│
│ from_version │
│ to_version   │
│ status       │
│ started_at   │
│ completed_at │
│ backup_id(FK) │
│ error_msg    │
│ operated_by  │
└──────────────┘
```

---
│ browser      │
│ login_result │
│ fail_reason  │
└──────────────┘
```

### 3.2 表结构定义

#### 3.2.1 租户表 (tenant)

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL COMMENT '租户名称',
    code VARCHAR(50) UNIQUE NOT NULL COMMENT '租户代码',
    logo_url VARCHAR(500) COMMENT 'Logo URL',
    contact_name VARCHAR(50) COMMENT '联系人姓名',
    contact_phone VARCHAR(20) COMMENT '联系电话',
    contact_email VARCHAR(100) COMMENT '联系邮箱',
    address VARCHAR(200) COMMENT '地址',
    description TEXT COMMENT '简介',
    app_id VARCHAR(50) UNIQUE COMMENT '应用ID',
    app_secret VARCHAR(100) COMMENT '应用密钥(加密存储)',
    config JSONB DEFAULT '{}' COMMENT '租户配置',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/inactive',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenant_code ON tenant(code);
CREATE INDEX idx_tenant_status ON tenant(status);
```

#### 3.2.2 用户表 (user)

```sql
CREATE TABLE user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    username VARCHAR(50) UNIQUE NOT NULL COMMENT '用户名',
    password_hash VARCHAR(255) NOT NULL COMMENT '密码哈希',
    name VARCHAR(50) NOT NULL COMMENT '姓名',
    email VARCHAR(100) COMMENT '邮箱',
    email_verified BOOLEAN DEFAULT FALSE,
    phone VARCHAR(20) COMMENT '手机号',
    phone_verified BOOLEAN DEFAULT FALSE,
    avatar_url VARCHAR(500) COMMENT '头像URL',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/disabled',
    last_login_at TIMESTAMP,
    last_login_ip INET,
    extra_permissions JSONB DEFAULT '[]' COMMENT '额外单独配置的权限',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_tenant ON user(tenant_id);
CREATE INDEX idx_user_username ON user(username);
CREATE INDEX idx_user_status ON user(status);
```

#### 3.2.3 角色表 (role)

```sql
CREATE TABLE role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenant(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL COMMENT '角色名称',
    code VARCHAR(50) NOT NULL COMMENT '角色代码',
    description VARCHAR(200),
    is_system BOOLEAN DEFAULT FALSE COMMENT '是否系统内置',
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_role_tenant ON role(tenant_id);
```

#### 3.2.4 用户角色关联表 (user_role)

```sql
CREATE TABLE user_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES user(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, role_id)
);

CREATE INDEX idx_user_role_user ON user_role(user_id);
CREATE INDEX idx_user_role_role ON user_role(role_id);
```

#### 3.2.5 权限表 (permission)

```sql
CREATE TABLE permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenant(id) ON DELETE CASCADE,
    code VARCHAR(100) UNIQUE NOT NULL COMMENT '权限代码',
    name VARCHAR(50) NOT NULL COMMENT '权限名称',
    module VARCHAR(50) NOT NULL COMMENT '所属模块',
    description VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_permission_code ON permission(code);
CREATE INDEX idx_permission_module ON permission(module);
```

#### 3.2.6 角色权限关联表 (role_permission)

```sql
CREATE TABLE role_permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    is_granted BOOLEAN DEFAULT TRUE COMMENT '是否授予',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, permission_id)
);

CREATE INDEX idx_role_perm_role ON role_permission(role_id);
CREATE INDEX idx_role_perm_perm ON role_permission(permission_id);
```

#### 3.2.7 项目表 (project)

```sql
CREATE TABLE project (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL COMMENT '项目名称',
    code VARCHAR(50) NOT NULL COMMENT '项目代码',
    description TEXT,
    fields_def JSONB NOT NULL DEFAULT '[]' COMMENT '入参数据项定义',
    rule_ids JSONB DEFAULT '[]' COMMENT '关联的规则ID列表',
    need_manual_review BOOLEAN DEFAULT FALSE COMMENT '是否需要人工校验',
    task_assign_type VARCHAR(20) DEFAULT 'round_robin' COMMENT '任务分配方式',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/inactive/archived',
    created_by UUID REFERENCES user(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_project_tenant ON project(tenant_id);
CREATE INDEX idx_project_status ON project(status);
```

#### 3.2.8 规则表 (rule)

```sql
CREATE TABLE rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL COMMENT '规则名称',
    description TEXT,
    conditions JSONB NOT NULL COMMENT '条件组定义',
    result_pass TEXT COMMENT '通过结论',
    result_reject TEXT COMMENT '拒绝原因',
    result_suggest TEXT COMMENT '建议处理',
    priority INT DEFAULT 0 COMMENT '优先级',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/inactive',
    created_by UUID REFERENCES user(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_rule_tenant ON rule(tenant_id);
CREATE INDEX idx_rule_status ON rule(status);
```

#### 3.2.9 审核数据表 (audit_data)

```sql
CREATE TABLE audit_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    input_data JSONB NOT NULL COMMENT '原始入参数据',
    rule_result JSONB COMMENT '规则执行结果',
    auto_result VARCHAR(20) COMMENT '自动审核结果: pass/reject/pending',
    final_result VARCHAR(20) COMMENT '最终审核结果',
    auditor_id UUID REFERENCES user(id),
    audit_time TIMESTAMP,
    audit_note TEXT,
    need_recheck BOOLEAN DEFAULT FALSE COMMENT '需要二次校验',
    recheck_auditor_id UUID REFERENCES user(id),
    recheck_time TIMESTAMP,
    recheck_result VARCHAR(20),
    recheck_note TEXT,
    status VARCHAR(20) DEFAULT 'pending' COMMENT 'pending/in_progress/passed/rejected/rechecking',
    priority INT DEFAULT 0 COMMENT '优先级',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_tenant ON audit_data(tenant_id);
CREATE INDEX idx_audit_project ON audit_data(project_id);
CREATE INDEX idx_audit_status ON audit_data(status);
CREATE INDEX idx_audit_auditor ON audit_data(auditor_id);
CREATE INDEX idx_audit_need_recheck ON audit_data(need_recheck);
CREATE INDEX idx_audit_created ON audit_data(created_at);
```

#### 3.2.10 任务表 (task)

```sql
CREATE TABLE task (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    auditor_id UUID REFERENCES user(id) ON DELETE SET NULL,
    data_id UUID NOT NULL REFERENCES audit_data(id) ON DELETE CASCADE,
    type VARCHAR(20) DEFAULT 'review' COMMENT 'review/recheck',
    status VARCHAR(20) DEFAULT 'pending' COMMENT 'pending/assigned/in_progress/completed/skiped',
    result VARCHAR(20) COMMENT 'pass/reject',
    note TEXT,
    assigned_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_task_tenant ON task(tenant_id);
CREATE INDEX idx_task_auditor ON task(auditor_id);
CREATE INDEX idx_task_data ON task(data_id);
CREATE INDEX idx_task_status ON task(status);
CREATE INDEX idx_task_type ON task(type);
```

#### 3.2.11 三方接口配置表 (third_party_config)

```sql
CREATE TABLE third_party_config (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL COMMENT '接口名称',
    type VARCHAR(50) NOT NULL COMMENT '接口类型',
    url VARCHAR(500) NOT NULL,
    method VARCHAR(10) DEFAULT 'POST',
    headers JSONB DEFAULT '{}',
    params_template JSONB DEFAULT '{}',
    auth_type VARCHAR(20) DEFAULT 'none' COMMENT 'none/bearer/oauth2/api_key',
    auth_config JSONB DEFAULT '{}' COMMENT '认证配置',
    timeout INT DEFAULT 30,
    retry_count INT DEFAULT 3,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_third_party_tenant ON third_party_config(tenant_id);
CREATE INDEX idx_third_party_type ON third_party_config(type);
```

#### 3.2.12 三方接口日志表 (third_party_log)

```sql
CREATE TABLE third_party_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    config_id UUID REFERENCES third_party_config(id) ON DELETE SET NULL,
    user_id UUID REFERENCES user(id) ON DELETE SET NULL,
    request_time TIMESTAMP NOT NULL,
    request_url VARCHAR(500),
    request_method VARCHAR(10),
    request_data JSONB,
    response_data JSONB,
    response_status VARCHAR(20),
    error_msg TEXT,
    duration_ms INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_third_party_log_tenant ON third_party_log(tenant_id);
CREATE INDEX idx_third_party_log_config ON third_party_log(config_id);
CREATE INDEX idx_third_party_log_user ON third_party_log(user_id);
CREATE INDEX idx_third_party_log_time ON third_party_log(request_time);
```

#### 3.2.13 登录日志表 (login_log)

```sql
CREATE TABLE login_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenant(id) ON DELETE SET NULL,
    user_id UUID REFERENCES user(id) ON DELETE SET NULL,
    login_time TIMESTAMP NOT NULL,
    login_ip INET,
    device_type VARCHAR(20) COMMENT 'pc_web/mobile_h5/mini_program/ios_app/android_app',
    device_id VARCHAR(100),
    os VARCHAR(50),
    browser VARCHAR(50),
    login_result VARCHAR(20) NOT NULL COMMENT 'success/failed',
    fail_reason VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_login_log_tenant ON login_log(tenant_id);
CREATE INDEX idx_login_log_user ON login_log(user_id);
CREATE INDEX idx_login_log_time ON login_log(login_time);
```

#### 3.2.14 操作日志表 (operation_log)

```sql
CREATE TABLE operation_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenant(id) ON DELETE CASCADE,
    user_id UUID REFERENCES user(id) ON DELETE SET NULL,
    module VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    detail TEXT,
    entity_type VARCHAR(50),
    entity_id UUID,
    ip INET,
    device_type VARCHAR(20),
    request_data JSONB,
    response_data JSONB,
    status VARCHAR(20) DEFAULT 'success',
    error_msg TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_operation_log_tenant ON operation_log(tenant_id);
CREATE INDEX idx_operation_log_user ON operation_log(user_id);
CREATE INDEX idx_operation_log_module ON operation_log(module);
CREATE INDEX idx_operation_log_time ON operation_log(created_at);
```

#### 3.2.15 系统版本表 (system_version)

```sql
CREATE TABLE system_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version VARCHAR(50) NOT NULL COMMENT '版本号，如 1.0.0',
    version_code INT NOT NULL COMMENT '版本数字，用于比较，如 100',
    release_type VARCHAR(20) DEFAULT 'stable' COMMENT 'release/stable/beta',
    release_notes TEXT COMMENT '发布说明',
    download_url VARCHAR(500) COMMENT '下载包地址',
    checksum VARCHAR(64) COMMENT '文件校验码 SHA256',
    is_mandatory BOOLEAN DEFAULT FALSE COMMENT '是否强制更新',
    is_enabled BOOLEAN DEFAULT TRUE COMMENT '是否启用此版本',
    min_compatible_version VARCHAR(50) COMMENT '最低兼容版本',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP COMMENT '发布时间'
);

CREATE INDEX idx_version_code ON system_version(version_code DESC);
CREATE INDEX idx_version_enabled ON system_version(is_enabled);
CREATE INDEX idx_version_published ON system_version(published_at DESC);
```

#### 3.2.16 租户版本记录表 (tenant_version)

```sql
CREATE TABLE tenant_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    current_version VARCHAR(50) NOT NULL COMMENT '当前版本',
    current_version_code INT NOT NULL,
    last_check_at TIMESTAMP COMMENT '最后检查版本时间',
    last_update_at TIMESTAMP COMMENT '最后更新时间',
    auto_update_enabled BOOLEAN DEFAULT TRUE COMMENT '是否开启自动更新',
    update_status VARCHAR(20) DEFAULT 'idle' COMMENT 'idle/updating/failed',
    updated_by UUID REFERENCES user(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenant_version_tenant ON tenant_version(tenant_id);
```

#### 3.2.17 备份规则表 (backup_rule)

```sql
CREATE TABLE backup_rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL COMMENT '规则名称',
    backup_type VARCHAR(20) DEFAULT 'full' COMMENT 'full/differential/incremental',
    schedule_cron VARCHAR(50) COMMENT 'Cron 表达式',
    retention_days INT DEFAULT 30 COMMENT '保留天数',
    storage_location VARCHAR(200) DEFAULT 'local' COMMENT 'local/s3/oss',
    storage_config JSONB DEFAULT '{}' COMMENT '存储配置',
    is_enabled BOOLEAN DEFAULT TRUE,
    last_backup_at TIMESTAMP,
    next_backup_at TIMESTAMP,
    created_by UUID REFERENCES user(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_backup_rule_tenant ON backup_rule(tenant_id);
CREATE INDEX idx_backup_rule_enabled ON backup_rule(is_enabled);
CREATE INDEX idx_backup_rule_next ON backup_rule(next_backup_at);
```

#### 3.2.18 备份记录表 (backup_record)

```sql
CREATE TABLE backup_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    rule_id UUID REFERENCES backup_rule(id) ON DELETE SET NULL,
    backup_type VARCHAR(20) NOT NULL COMMENT 'full/differential/incremental',
    status VARCHAR(20) DEFAULT 'pending' COMMENT 'pending/running/success/failed',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    file_path VARCHAR(500) COMMENT '备份文件路径',
    file_size BIGINT COMMENT '文件大小(字节)',
    checksum VARCHAR(64) COMMENT '校验码',
    error_msg TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_backup_record_tenant ON backup_record(tenant_id);
CREATE INDEX idx_backup_record_rule ON backup_record(rule_id);
CREATE INDEX idx_backup_record_status ON backup_record(status);
CREATE INDEX idx_backup_record_created ON backup_record(created_at DESC);
```

#### 3.2.19 版本升级记录表 (upgrade_record)

```sql
CREATE TABLE upgrade_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    from_version VARCHAR(50) NOT NULL,
    to_version VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' COMMENT 'pending/running/success/failed/rollback',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    backup_id UUID REFERENCES backup_record(id),
    error_msg TEXT,
    rollback_note TEXT,
    operated_by UUID REFERENCES user(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_upgrade_tenant ON upgrade_record(tenant_id);
CREATE INDEX idx_upgrade_status ON upgrade_record(status);
CREATE INDEX idx_upgrade_created ON upgrade_record(created_at DESC);
```

### 3.3 数据类型定义

#### 3.3.1 项目字段类型 (fields_def)

```typescript
// fields_def JSON 结构
interface FieldDefinition {
  name: string;           // 字段名（英文）
  label: string;         // 显示标签
  type: FieldType;       // 字段类型
  required?: boolean;    // 是否必填
  options?: string[];    // 下拉选项
  default?: any;         // 默认值
  validation?: string;   // 正则校验
  placeholder?: string;  // 占位文本
  maxLength?: number;    // 最大长度
  minLength?: number;    // 最小长度
  max?: number;          // 最大值
  min?: number;          // 最小值
}

type FieldType =
  | 'string'      // 字符串
  | 'number'      // 数字
  | 'boolean'     // 布尔
  | 'date'        // 日期
  | 'datetime'    // 日期时间
  | 'select'      // 下拉选择
  | 'multiSelect' // 多选
  | 'file'        // 文件
  | 'image'       // 图片
  | 'region'      // 省市区
  | 'tags'        // 标签
  | 'idCard'      // 身份证
  | 'phone'       // 手机号
  | 'email'       // 邮箱;
```

#### 3.3.2 规则条件类型 (conditions)

```typescript
// conditions JSON 结构
interface ConditionGroup {
  logic: 'ALL' | 'ANY';  // 组合方式
  conditions: (Condition | ConditionGroup)[];
}

interface Condition {
  field: string;           // 字段名
  operator: Operator;       // 运算符
  value: any;              // 比较值
}

type Operator =
  | 'equals'           // 等于
  | 'notEquals'        // 不等于
  | 'contains'         // 包含
  | 'notContains'      // 不包含
  | 'startsWith'       // 开头是
  | 'endsWith'         // 结尾是
  | 'greaterThan'      // 大于
  | 'lessThan'         // 小于
  | 'greaterOrEqual'   // 大于等于
  | 'lessOrEqual'      // 小于等于
  | 'between'          // 区间
  | 'in'               // 属于
  | 'notIn'            // 不属于
  | 'isEmpty'          // 为空
  | 'isNotEmpty'       // 不为空
  | 'matches'          // 正则匹配
  | 'idCardValid'      // 身份证验证
  | 'educationVerify'  // 学历验证
  | 'degreeVerify';    // 学位验证
```

---

## 4. API 接口设计

### 4.1 API 规范

#### 4.1.1 基本规范

- **基础路径**: `/api/v1`
- **认证方式**: Bearer Token (JWT)
- **请求格式**: `Content-Type: application/json`
- **响应格式**: JSON

#### 4.1.2 通用请求头

| 头信息 | 说明 | 必填 |
|--------|------|------|
| Authorization | Bearer {token} | 是 |
| X-Tenant-ID | 租户ID | 是 |
| X-Request-ID | 请求唯一ID | 否 |
| X-Language | 语言: zh-CN/en | 否 |
| Content-Type | application/json | 是 |

#### 4.1.3 通用响应格式

```typescript
// 成功响应
interface SuccessResponse<T> {
  code: 0;
  message: 'success';
  data: T;
  timestamp: number;
}

// 分页响应
interface PaginatedResponse<T> {
  code: 0;
  message: 'success';
  data: {
    items: T[];
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
  timestamp: number;
}

// 错误响应
interface ErrorResponse {
  code: number;        // 错误码
  message: string;     // 错误信息
  details?: any;       // 详细错误信息
  requestId?: string;  // 请求ID
  timestamp: number;
}
```

#### 4.1.4 错误码定义

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 10001 | 参数错误 |
| 10002 | 缺少必填参数 |
| 10003 | 参数格式错误 |
| 20001 | 未授权 |
| 20002 | Token无效 |
| 20003 | Token过期 |
| 20004 | 权限不足 |
| 30001 | 资源不存在 |
| 30002 | 资源已存在 |
| 40001 | 业务逻辑错误 |
| 50001 | 系统内部错误 |

### 4.2 认证接口

#### 4.2.1 登录

```
POST /api/v1/auth/login
```

**请求参数**:
```typescript
interface LoginRequest {
  username: string;    // 用户名
  password: string;     // 密码
  captcha?: string;     // 验证码
  captchaKey?: string;  // 验证码key
}
```

**响应**:
```typescript
interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;    // 过期时间(秒)
  tokenType: 'Bearer';
  user: {
    id: string;
    username: string;
    name: string;
    avatar?: string;
    tenant: {
      id: string;
      name: string;
      code: string;
    };
    roles: string[];
    permissions: string[];
  };
}
```

#### 4.2.2 刷新Token

```
POST /api/v1/auth/refresh
```

**请求参数**:
```typescript
interface RefreshTokenRequest {
  refreshToken: string;
}
```

#### 4.2.3 登出

```
POST /api/v1/auth/logout
```

#### 4.2.4 获取当前用户信息

```
GET /api/v1/auth/me
```

### 4.3 租户管理接口

#### 4.3.1 获取租户信息

```
GET /api/v1/tenant
```

#### 4.3.2 更新租户信息

```
PUT /api/v1/tenant
```

**请求参数**:
```typescript
interface UpdateTenantRequest {
  name?: string;
  logoUrl?: string;
  contactName?: string;
  contactPhone?: string;
  contactEmail?: string;
  address?: string;
  description?: string;
}
```

#### 4.3.3 获取APPID/APPSECRET

```
GET /api/v1/tenant/app-credentials
```

#### 4.3.4 重置APPID/APPSECRET

```
POST /api/v1/tenant/app-credentials/reset
```

#### 4.3.5 租户数据统计

```
GET /api/v1/tenant/statistics
```

**响应**:
```typescript
interface TenantStatisticsResponse {
  totalProjects: number;
  totalAuditData: number;
  monthAuditData: number;
  pendingAuditData: number;
  auditorCount: number;
  thirdPartyCallCount: number;
}
```

### 4.4 账户管理接口

#### 4.4.1 账户列表

```
GET /api/v1/accounts
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| keyword | string | 搜索关键词 |
| status | string | 状态筛选 |
| roleId | string | 角色筛选 |

#### 4.4.2 创建账户

```
POST /api/v1/accounts
```

**请求参数**:
```typescript
interface CreateAccountRequest {
  username: string;
  password: string;
  name: string;
  email?: string;
  phone?: string;
  roleIds: string[];
  extraPermissions?: string[];
}
```

#### 4.4.3 更新账户

```
PUT /api/v1/accounts/{id}
```

#### 4.4.4 删除账户

```
DELETE /api/v1/accounts/{id}
```

#### 4.4.5 分配角色

```
PUT /api/v1/accounts/{id}/roles
```

**请求参数**:
```typescript
interface AssignRolesRequest {
  roleIds: string[];
}
```

#### 4.4.6 配置单独权限

```
PUT /api/v1/accounts/{id}/permissions
```

**请求参数**:
```typescript
interface ConfigurePermissionsRequest {
  extraPermissions: string[];  // 要添加的权限
  removePermissions?: string[]; // 要移除的权限
}
```

#### 4.4.7 重置密码

```
POST /api/v1/accounts/{id}/reset-password
```

#### 4.4.8 修改个人资料

```
PUT /api/v1/accounts/profile
```

**请求参数**:
```typescript
interface UpdateProfileRequest {
  name?: string;
  avatarUrl?: string;
  email?: string;
  phone?: string;
}
```

#### 4.4.9 修改密码

```
PUT /api/v1/accounts/password
```

**请求参数**:
```typescript
interface ChangePasswordRequest {
  oldPassword: string;
  newPassword: string;
}
```

#### 4.4.10 获取当前账户权限

```
GET /api/v1/accounts/my-permissions
```

### 4.5 角色管理接口

#### 4.5.1 角色列表

```
GET /api/v1/roles
```

#### 4.5.2 创建角色

```
POST /api/v1/roles
```

#### 4.5.3 更新角色

```
PUT /api/v1/roles/{id}
```

#### 4.5.4 删除角色

```
DELETE /api/v1/roles/{id}
```

#### 4.5.5 配置角色权限

```
PUT /api/v1/roles/{id}/permissions
```

**请求参数**:
```typescript
interface ConfigureRolePermissionsRequest {
  permissions: string[];  // 权限code列表
}
```

### 4.6 项目管理接口

#### 4.6.1 项目列表

```
GET /api/v1/projects
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| keyword | string | 搜索关键词 |
| status | string | 状态筛选 |

#### 4.6.2 创建项目

```
POST /api/v1/projects
```

**请求参数**:
```typescript
interface CreateProjectRequest {
  name: string;
  code: string;
  description?: string;
  fieldsDef: FieldDefinition[];
  ruleIds?: string[];
  needManualReview?: boolean;
  taskAssignType?: 'round_robin' | 'load_balance' | 'efficiency' | 'random' | 'manual';
}
```

#### 4.6.3 更新项目

```
PUT /api/v1/projects/{id}
```

#### 4.6.4 删除项目

```
DELETE /api/v1/projects/{id}
```

#### 4.6.5 获取项目详情

```
GET /api/v1/projects/{id}
```

#### 4.6.6 项目完成度统计

```
GET /api/v1/projects/{id}/statistics
```

**响应**:
```typescript
interface ProjectStatisticsResponse {
  totalData: number;
  pendingData: number;
  inProgressData: number;
  passedData: number;
  rejectedData: number;
  recheckingData: number;
  progressRate: number;       // 完成百分比
  auditorWorkload: {          // 审核员工作量
    auditorId: string;
    auditorName: string;
    completedCount: number;
  }[];
  rejectReasonDistribution: { // 拒绝原因分布
    reason: string;
    count: number;
  }[];
  avgDuration: number;        // 平均审核时长(秒)
}
```

### 4.7 规则管理接口

#### 4.7.1 规则列表

```
GET /api/v1/rules
```

#### 4.7.2 创建规则

```
POST /api/v1/rules
```

**请求参数**:
```typescript
interface CreateRuleRequest {
  name: string;
  description?: string;
  conditions: ConditionGroup;
  resultPass?: string;
  resultReject?: string;
  resultSuggest?: string;
  priority?: number;
}
```

#### 4.7.3 更新规则

```
PUT /api/v1/rules/{id}
```

#### 4.7.4 删除规则

```
DELETE /api/v1/rules/{id}
```

#### 4.7.5 测试规则

```
POST /api/v1/rules/test
```

**请求参数**:
```typescript
interface TestRuleRequest {
  conditions: ConditionGroup;
  testData: Record<string, any>;
}
```

**响应**:
```typescript
interface TestRuleResponse {
  passed: boolean;
  matchedConditions: string[];   // 命中的条件
  unmatchedConditions: string[];  // 未命中的条件
  result: 'pass' | 'reject';
}
```

### 4.8 审核数据接口

#### 4.8.1 待审核数据列表

```
GET /api/v1/audit/pending
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| projectId | string | 项目ID |
| priority | int | 优先级 |

#### 4.8.2 已审核数据列表

```
GET /api/v1/audit/completed
```

#### 4.8.3 提交审核数据

```
POST /api/v1/audit/submit
```

**请求参数**:
```typescript
interface SubmitAuditDataRequest {
  projectId: string;
  inputData: Record<string, any>;
  priority?: number;
}
```

#### 4.8.4 执行审核

```
POST /api/v1/audit/{id}/execute
```

**请求参数**:
```typescript
interface ExecuteAuditRequest {
  result: 'pass' | 'reject' | 'recheck';
  note?: string;
}
```

#### 4.8.5 批量执行审核

```
POST /api/v1/audit/batch-execute
```

#### 4.8.6 获取审核详情

```
GET /api/v1/audit/{id}
```

#### 4.8.7 标记二次校验

```
POST /api/v1/audit/{id}/mark-recheck
```

### 4.9 任务接口

#### 4.9.1 获取我的任务

```
GET /api/v1/tasks/my
```

**响应**:
```typescript
interface MyTasksResponse {
  pendingCount: number;        // 待审核数量
  recheckCount: number;        // 待二次校验数量
  todayCompleted: number;     // 今日完成
  weekCompleted: number;      // 本周完成
  monthCompleted: number;      // 本月完成
  yearCompleted: number;      // 本年完成
  totalCompleted: number;      // 累计完成
  avgDuration: number;         // 平均审核时长
}
```

#### 4.9.2 获取任务列表

```
GET /api/v1/tasks
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| status | string | 任务状态 |
| auditorId | string | 审核员ID |
| type | string | 类型: review/recheck |

#### 4.9.3 接受任务

```
POST /api/v1/tasks/{id}/accept
```

#### 4.9.4 开始任务

```
POST /api/v1/tasks/{id}/start
```

#### 4.9.5 完成审核

```
POST /api/v1/tasks/{id}/complete
```

#### 4.9.6 跳过任务

```
POST /api/v1/tasks/{id}/skip
```

### 4.10 三方接口配置接口

#### 4.10.1 接口配置列表

```
GET /api/v1/third-party/configs
```

#### 4.10.2 创建接口配置

```
POST /api/v1/third-party/configs
```

**请求参数**:
```typescript
interface CreateThirdPartyConfigRequest {
  name: string;
  type: 'id_card_verify' | 'education_verify' | 'degree_verify' | 'bank_four_element' | 'operator_verify';
  url: string;
  method?: 'GET' | 'POST';
  headers?: Record<string, string>;
  paramsTemplate?: Record<string, any>;
  authType?: 'none' | 'bearer' | 'oauth2' | 'api_key';
  authConfig?: {
    tokenUrl?: string;
    clientId?: string;
    clientSecret?: string;
    apiKey?: string;
  };
  timeout?: number;
  retryCount?: number;
}
```

#### 4.10.3 更新接口配置

```
PUT /api/v1/third-party/configs/{id}
```

#### 4.10.4 删除接口配置

```
DELETE /api/v1/third-party/configs/{id}
```

#### 4.10.5 测试接口连接

```
POST /api/v1/third-party/configs/{id}/test
```

#### 4.10.6 接口调用日志

```
GET /api/v1/third-party/logs
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| configId | string | 接口配置ID |
| status | string | 调用状态 |
| startTime | string | 开始时间 |
| endTime | string | 结束时间 |

#### 4.10.7 接口调用统计

```
GET /api/v1/third-party/statistics
```

**响应**:
```typescript
interface ThirdPartyStatisticsResponse {
  totalCalls: number;
  monthCalls: number;
  todayCalls: number;
  successRate: number;
  avgResponseTime: number;
  callRanking: {
    configId: string;
    configName: string;
    callCount: number;
  }[];
}
```

### 4.11 统计接口

#### 4.11.1 登录设备统计

```
GET /api/v1/statistics/login-devices
```

#### 4.11.2 使用日志

```
GET /api/v1/statistics/operation-logs
```

---

## 5. 前端项目规范

### 5.1 项目结构

```
frontend/
├── public/                     # 静态资源
│   ├── favicon.ico
│   └── manifest.json
├── src/
│   ├── api/                    # API 接口
│   │   ├── index.ts           # API 入口
│   │   ├── client.ts          # Axios 配置
│   │   ├── modules/           # 分模块接口
│   │   │   ├── auth.ts
│   │   │   ├── tenant.ts
│   │   │   ├── account.ts
│   │   │   ├── role.ts
│   │   │   ├── project.ts
│   │   │   ├── rule.ts
│   │   │   ├── audit.ts
│   │   │   ├── task.ts
│   │   │   └── thirdParty.ts
│   │   └── types/             # API 类型定义
│   │       └── response.ts
│   │
│   ├── components/             # 公共组件
│   │   ├── ui/                # 基础UI组件
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Dialog.tsx
│   │   │   ├── Table.tsx
│   │   │   ├── Pagination.tsx
│   │   │   └── ...
│   │   ├── layout/            # 布局组件
│   │   │   ├── AppLayout.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   └── ...
│   │   └── business/           # 业务组件
│   │       ├── RuleEditor.tsx
│   │       ├── ConditionBuilder.tsx
│   │       ├── AuditCard.tsx
│   │       └── ...
│   │
│   ├── pages/                 # 页面组件
│   │   ├── auth/              # 认证页面
│   │   │   ├── Login.tsx
│   │   │   └── ...
│   │   ├── tenant/            # 租户管理
│   │   ├── account/           # 账户管理
│   │   ├── role/              # 角色管理
│   │   ├── project/           # 项目管理
│   │   ├── rule/              # 规则管理
│   │   ├── audit/             # 审核管理
│   │   ├── task/              # 任务管理
│   │   ├── thirdParty/        # 三方接口
│   │   ├── statistics/        # 统计报表
│   │   └── profile/           # 个人中心
│   │
│   ├── hooks/                 # 自定义 Hooks
│   │   ├── useAuth.ts
│   │   ├── usePermission.ts
│   │   ├── useTable.ts
│   │   └── ...
│   │
│   ├── stores/                # 状态管理 (Zustand)
│   │   ├── authStore.ts
│   │   ├── tenantStore.ts
│   │   └── ...
│   │
│   ├── router/                # 路由配置
│   │   ├── index.tsx
│   │   ├── routes.ts
│   │   └── guards.ts
│   │
│   ├── utils/                 # 工具函数
│   │   ├── request.ts         # Axios 封装
│   │   ├── storage.ts         # 本地存储
│   │   ├── helpers.ts
│   │   └── validators.ts
│   │
│   ├── types/                 # 全局类型定义
│   │   ├── index.d.ts
│   │   └── ...
│   │
│   ├── styles/                # 样式文件
│   │   ├── globals.css
│   │   └── variables.css
│   │
│   ├── App.tsx
│   └── main.tsx
│
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
├── postcss.config.js
└── .eslintrc.cjs
```

### 5.2 命名规范

#### 5.2.1 文件命名

| 类型 | 规范 | 示例 |
|------|------|------|
| 组件文件 | PascalCase.tsx | UserProfile.tsx |
| 工具文件 | camelCase.ts | formatDate.ts |
| 类型文件 | camelCase.ts | types.ts |
| 测试文件 | PascalCase.test.tsx | UserProfile.test.tsx |
| 样式文件 | kebab-case.css | user-profile.css |

#### 5.2.2 变量命名

| 类型 | 规范 | 示例 |
|------|------|------|
| 普通变量 | camelCase | userName, isActive |
| 常量 | UPPER_SNAKE_CASE | MAX_RETRY_COUNT |
| 组件 props | camelCase | onSubmit, isLoading |
| 事件处理 | handle + PascalCase | handleSubmit, handleClick |
| 布尔值 | is/has/can + PascalCase | isActive, hasPermission |

#### 5.2.3 API 命名

| 类型 | 规范 | 示例 |
|------|------|------|
| API 函数 | use + PascalCase | useLogin, useProjects |
| 请求函数 | camelCase | getProjects, createUser |
| 类型定义 | PascalCase | LoginRequest, ProjectResponse |

### 5.3 组件规范

#### 5.3.1 组件结构

```typescript
// Good: 组件按以下顺序组织
import React from 'react';
import { cn } from '@/utils/cn';
import { Button } from '@/components/ui/Button';

// Types
interface UserCardProps {
  user: User;
  onEdit?: (user: User) => void;
  className?: string;
}

// Component
export function UserCard({ user, onEdit, className }: UserCardProps) {
  // 1. Hooks
  const { hasPermission } = usePermission();

  // 2. State
  const [isLoading, setIsLoading] = React.useState(false);

  // 3. Event handlers
  const handleEdit = async () => {
    setIsLoading(true);
    try {
      await onEdit?.(user);
    } finally {
      setIsLoading(false);
    }
  };

  // 4. Render
  return (
    <div className={cn('p-4 bg-white rounded-lg', className)}>
      <h3 className="text-lg font-medium">{user.name}</h3>
      <Button
        onClick={handleEdit}
        loading={isLoading}
        disabled={!hasPermission('account:edit')}
      >
        编辑
      </Button>
    </div>
  );
}
```

#### 5.3.2 Hooks 规范

```typescript
// useUser.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '@/api/modules/account';
import type { CreateUserRequest, UpdateUserRequest } from '@/api/types';

export function useUsers(params?: GetUsersParams) {
  return useQuery({
    queryKey: ['users', params],
    queryFn: () => userApi.getUsers(params),
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateUserRequest) => userApi.createUser(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

export function useUpdateUser(id: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateUserRequest) => userApi.updateUser(id, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      queryClient.invalidateQueries({ queryKey: ['user', id] });
    },
  });
}
```

### 5.4 Tailwind CSS 规范

#### 5.4.1 常用样式类

```css
/* 布局 */
flex, grid, block, inline-block, hidden
items-center, items-start, items-end
justify-center, justify-between, justify-around
gap-1, gap-2, gap-4, gap-6
p-2, p-4, p-6, px-4, py-2
m-2, m-4, mt-4, mb-4, ml-4, mr-4
w-full, h-full, min-h-screen

/* 文字 */
text-sm, text-base, text-lg, text-xl, text-2xl
font-medium, font-semibold, font-bold
text-gray-500, text-gray-900
text-center, text-left, text-right

/* 颜色 */
bg-white, bg-gray-50, bg-gray-100
text-primary, text-success, text-danger, text-warning
border-gray-200, border-primary

/* 交互 */
hover:bg-gray-100, hover:text-primary
focus:ring-2, focus:ring-primary
disabled:opacity-50, disabled:cursor-not-allowed
transition-colors, transition-all

/* 响应式 */
sm:, md:, lg:, xl:, 2xl:
```

#### 5.4.2 组件样式组织

```typescript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{js,jsx,ts,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#EFF6FF',
          100: '#DBEAFE',
          200: '#BFDBFE',
          300: '#93C5FD',
          400: '#60A5FA',
          500: '#3B82F6',
          600: '#2563EB',
          700: '#1D4ED8',
          800: '#1E40AF',
          900: '#1E3A8A',
        },
        success: {
          DEFAULT: '#10B981',
          light: '#D1FAE5',
        },
        danger: {
          DEFAULT: '#EF4444',
          light: '#FEE2E2',
        },
        warning: {
          DEFAULT: '#F59E0B',
          light: '#FEF3C7',
        },
      },
    },
  },
  plugins: [],
};
```

### 5.5 状态管理规范 (Zustand)

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { User, Tenant } from '@/types';

interface AuthState {
  user: User | null;
  tenant: Tenant | null;
  token: string | null;
  isAuthenticated: boolean;
  permissions: string[];

  setAuth: (data: { user: User; tenant: Tenant; token: string }) => void;
  setPermissions: (permissions: string[]) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      tenant: null,
      token: null,
      isAuthenticated: false,
      permissions: [],

      setAuth: (data) =>
        set({
          user: data.user,
          tenant: data.tenant,
          token: data.token,
          isAuthenticated: true,
        }),

      setPermissions: (permissions) => set({ permissions }),

      logout: () =>
        set({
          user: null,
          tenant: null,
          token: null,
          isAuthenticated: false,
          permissions: [],
        }),
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({
        token: state.token,
        user: state.user,
        tenant: state.tenant,
      }),
    }
  )
);
```

---

## 6. 后端项目规范

### 6.1 项目结构

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 应用入口
│   │
│   ├── api/                    # API 路由
│   │   ├── __init__.py
│   │   ├── deps.py             # 依赖注入
│   │   └── v1/                  # API v1 版本
│   │       ├── __init__.py
│   │       ├── router.py       # 路由汇总
│   │       ├── auth.py
│   │       ├── tenant.py
│   │       ├── account.py
│   │       ├── role.py
│   │       ├── project.py
│   │       ├── rule.py
│   │       ├── audit.py
│   │       ├── task.py
│   │       ├── third_party.py
│   │       └── statistics.py
│   │
│   ├── core/                   # 核心配置
│   │   ├── __init__.py
│   │   ├── config.py           # 配置管理
│   │   ├── security.py         # 安全工具
│   │   ├── database.py         # 数据库连接
│   │   ├── redis.py            # Redis 连接
│   │   └── exceptions.py       # 自定义异常
│   │
│   ├── models/                 # SQLAlchemy 模型
│   │   ├── __init__.py
│   │   ├── base.py             # 基础模型
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── role.py
│   │   ├── permission.py
│   │   ├── project.py
│   │   ├── rule.py
│   │   ├── audit_data.py
│   │   ├── task.py
│   │   ├── third_party.py
│   │   └── login_log.py
│   │
│   ├── schemas/                # Pydantic 模型
│   │   ├── __init__.py
│   │   ├── base.py             # 基础 schema
│   │   ├── auth.py
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── role.py
│   │   ├── project.py
│   │   ├── rule.py
│   │   ├── audit.py
│   │   ├── task.py
│   │   └── third_party.py
│   │
│   ├── services/               # 业务逻辑
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── tenant_service.py
│   │   ├── user_service.py
│   │   ├── role_service.py
│   │   ├── project_service.py
│   │   ├── rule_service.py
│   │   ├── audit_service.py
│   │   ├── task_service.py
│   │   └── third_party_service.py
│   │
│   ├── repositories/           # 数据访问层
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── user_repo.py
│   │   └── ...
│   │
│   ├── utils/                  # 工具函数
│   │   ├── __init__.py
│   │   ├── pagination.py
│   │   ├── password.py
│   │   └── ...
│   │
│   └── tasks/                  # 定时任务
│       ├── __init__.py
│       └── task_scheduler.py
│
├── migrations/                  # Alembic 数据库迁移
│   ├── env.py
│   └── versions/
│
├── tests/                       # 测试
│   ├── __init__.py
│   ├── conftest.py
│   ├── api/
│   ├── services/
│   └── ...
│
├── pyproject.toml
├── alembic.ini
└── README.md
```

### 6.2 核心代码规范

#### 6.2.1 主应用入口

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1 import router as api_v1
from app.core.config import settings
from app.core.database import engine
from app.core.exceptions import register_exceptions

def create_app() -> FastAPI:
    app = FastAPI(
        title="SmartAudit API",
        description="智核智能审核系统 API",
        version="1.0.0",
        docs_url="/docs",
        redoc_url="/redoc",
    )

    # 注册异常处理器
    register_exceptions(app)

    # CORS
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # 注册路由
    app.include_router(api_v1.router, prefix="/api/v1")

    return app

app = create_app()
```

#### 6.2.2 配置管理

```python
# app/core/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings
from typing import List


class Settings(BaseSettings):
    # 应用配置
    APP_NAME: str = "SmartAudit"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False

    # 数据库配置
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost:5432/smartaudit"

    # Redis 配置
    REDIS_URL: str = "redis://localhost:6379/0"

    # JWT 配置
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    # CORS
    CORS_ORIGINS: List[str] = ["*"]

    # 三方接口配置
    THIRD_PARTY_TIMEOUT: int = 30

    class Config:
        env_file = ".env"
        case_sensitive = True


@lru_cache()
def get_settings() -> Settings:
    return Settings()


settings = get_settings()
```

#### 6.2.3 数据库模型

```python
# app/models/base.py
from datetime import datetime
from uuid import uuid4
from sqlalchemy import Column, DateTime
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow, nullable=False)


class UUIDMixin:
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
```

```python
# app/models/user.py
from sqlalchemy import Column, String, Boolean, ForeignKey, Enum
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import relationship

from app.models.base import Base, UUIDMixin, TimestampMixin


class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "user"

    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenant.id"), nullable=False)
    username = Column(String(50), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    name = Column(String(50), nullable=False)
    email = Column(String(100))
    email_verified = Column(Boolean, default=False)
    phone = Column(String(20))
    phone_verified = Column(Boolean, default=False)
    avatar_url = Column(String(500))
    status = Column(String(20), default="active")
    last_login_at = Column(DateTime)
    last_login_ip = Column(String(50))
    extra_permissions = Column(JSONB, default=[])

    # Relationships
    tenant = relationship("Tenant", back_populates="users")
    roles = relationship("UserRole", back_populates="user")
```

#### 6.2.4 Pydantic Schema

```python
# app/schemas/user.py
from datetime import datetime
from typing import Optional, List
from pydantic import BaseModel, Field, ConfigDict
from uuid import UUID


class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    name: str = Field(..., min_length=1, max_length=50)
    email: Optional[str] = Field(None, format=EmailStr)
    phone: Optional[str] = Field(None, regex=r"^1[3-9]\d{9}$")
    avatar_url: Optional[str] = None


class UserCreate(UserBase):
    password: str = Field(..., min_length=6, max_length=50)
    role_ids: List[UUID] = []
    extra_permissions: List[str] = []


class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None
    phone: Optional[str] = None
    avatar_url: Optional[str] = None
    status: Optional[str] = None


class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    tenant_id: UUID
    status: str
    email_verified: bool
    phone_verified: bool
    created_at: datetime
    updated_at: datetime


class UserWithRolesResponse(UserResponse):
    roles: List["RoleResponse"] = []
    permissions: List[str] = []
```

#### 6.2.5 依赖注入

```python
# app/api/deps.py
from typing import Optional
from uuid import UUID
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from jose import JWTError, jwt

from app.core.config import settings
from app.core.database import get_db
from app.core.security import verify_password
from app.models.user import User

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token",
            )
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token is invalid",
        )

    result = await db.execute(select(User).where(User.id == UUID(user_id)))
    user = result.scalar_one_or_none()

    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )

    if user.status != "active":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User is inactive",
        )

    return user


async def get_current_tenant_id(
    current_user: User = Depends(get_current_user),
) -> UUID:
    return current_user.tenant_id


def require_permission(*permissions: str):
    async def permission_checker(
        current_user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db),
    ) -> User:
        # 检查用户权限逻辑
        # ...
        return current_user
    return permission_checker
```

#### 6.2.6 API 路由

```python
# app/api/v1/auth.py
from datetime import timedelta
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_db, get_current_user
from app.core.config import settings
from app.core.security import create_access_token, verify_password, get_password_hash
from app.models.user import User
from app.schemas.auth import LoginRequest, LoginResponse, TokenResponse
from app.schemas.user import UserResponse

router = APIRouter(prefix="/auth", tags=["认证"])


@router.post("/login", response_model=LoginResponse)
async def login(
    request: LoginRequest,
    db: AsyncSession = Depends(get_db),
):
    # 验证用户
    from sqlalchemy import select
    result = await db.execute(
        select(User).where(User.username == request.username)
    )
    user = result.scalar_one_or_none()

    if not user or not verify_password(request.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
        )

    # 创建 token
    access_token = create_access_token(
        data={"sub": str(user.id), "tenant_id": str(user.tenant_id)}
    )

    return LoginResponse(
        accessToken=access_token,
        refreshToken="",  # 简化处理
        expiresIn=settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60,
        tokenType="Bearer",
        user=UserResponse.model_validate(user),
    )


@router.get("/me", response_model=UserResponse)
async def get_me(current_user: User = Depends(get_current_user)):
    return UserResponse.model_validate(current_user)
```

### 6.3 服务层规范

```python
# app/services/user_service.py
from typing import List, Optional
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload

from app.models.user import User
from app.models.user_role import UserRole
from app.models.role import Role
from app.models.permission import Permission
from app.schemas.user import UserCreate, UserUpdate
from app.core.security import get_password_hash, verify_password


class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, user_id: UUID) -> Optional[User]:
        result = await self.db.execute(
            select(User)
            .options(selectinload(User.roles).selectinload(UserRole.role))
            .where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_username(self, username: str) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()

    async def get_users(
        self,
        tenant_id: UUID,
        skip: int = 0,
        limit: int = 20,
        keyword: Optional[str] = None,
        status: Optional[str] = None,
    ) -> tuple[List[User], int]:
        query = select(User).where(User.tenant_id == tenant_id)

        if keyword:
            query = query.where(
                User.name.ilike(f"%{keyword}%") |
                User.username.ilike(f"%{keyword}%")
            )

        if status:
            query = query.where(User.status == status)

        # Count
        count_query = select(func.count()).select_from(query.subquery())
        total = (await self.db.execute(count_query)).scalar()

        # Paginate
        query = query.offset(skip).limit(limit).order_by(User.created_at.desc())
        result = await self.db.execute(query)
        users = result.scalars().all()

        return list(users), total

    async def create(self, tenant_id: UUID, data: UserCreate) -> User:
        user = User(
            tenant_id=tenant_id,
            username=data.username,
            password_hash=get_password_hash(data.password),
            name=data.name,
            email=data.email,
            phone=data.phone,
        )
        self.db.add(user)
        await self.db.flush()

        # 添加角色
        for role_id in data.role_ids:
            user_role = UserRole(user_id=user.id, role_id=role_id)
            self.db.add(user_role)

        await self.db.commit()
        await self.db.refresh(user)
        return user

    async def update(self, user_id: UUID, data: UserUpdate) -> Optional[User]:
        user = await self.get_by_id(user_id)
        if not user:
            return None

        update_data = data.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            setattr(user, field, value)

        await self.db.commit()
        await self.db.refresh(user)
        return user

    async def delete(self, user_id: UUID) -> bool:
        user = await self.get_by_id(user_id)
        if not user:
            return False

        await self.db.delete(user)
        await self.db.commit()
        return True

    async def get_user_permissions(self, user_id: UUID) -> List[str]:
        """获取用户所有权限（角色权限 + 额外权限）"""
        user = await self.get_by_id(user_id)
        if not user:
            return []

        # 获取角色权限
        role_permissions = set()
        for ur in user.roles:
            for rp in ur.role.permissions:
                role_permissions.add(rp.permission.code)

        # 合并额外权限
        all_permissions = list(role_permissions) + list(user.extra_permissions or [])
        return all_permissions
```

---

## 7. 安全规范

### 7.1 认证与授权

| 规范 | 说明 |
|------|------|
| 密码加密 | 使用 bcrypt 算法，加密强度 12 |
| Token 格式 | JWT (HS256)，包含 user_id, tenant_id, exp |
| Token 有效期 | Access Token: 30分钟, Refresh Token: 7天 |
| 权限校验 | 基于 RBAC + 额外权限，支持多角色 |

### 7.2 API 安全

| 措施 | 说明 |
|------|------|
| HTTPS | 所有通信强制 HTTPS |
| 参数校验 | 使用 Pydantic 进行严格校验 |
| SQL 注入 | 使用 ORM 参数化查询 |
| XSS | 输入输出进行 HTML 转义 |
| CSRF | 使用 JWT Token 机制防御 |
| 限流 | 每个 IP/用户每分钟 60 次请求 |

### 7.3 数据安全

| 措施 | 说明 |
|------|------|
| 敏感数据 | 身份证号、手机号等部分脱敏展示 |
| 日志脱敏 | 日志中不记录密码、Token 等敏感信息 |
| 传输加密 | TLS 1.2+ |
| 数据备份 | 每日全量备份，保留 30 天 |

### 7.4 审计日志

| 记录内容 | 说明 |
|----------|------|
| 登录日志 | 时间、IP、设备、结果 |
| 操作日志 | 模块、动作、详情、操作用户 |
| 接口调用 | 请求参数、响应、日志追溯 |

---

## 8. 部署方案

### 8.1 Docker 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: smartaudit
      POSTGRES_USER: smartaudit
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U smartaudit"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql+asyncpg://smartaudit:${DB_PASSWORD}@postgres:5432/smartaudit
      REDIS_URL: redis://redis:6379/0
      SECRET_KEY: ${SECRET_KEY}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    depends_on:
      - backend
    restart: unless-stopped

  # Nginx (反向代理)
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - backend
      - frontend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 8.2 Nginx 配置

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
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # 上传文件大小限制
    client_max_body_size 10M;

    upstream backend {
        server backend:8000;
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

        # SSL 证书
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # API 请求
        location /api/ {
            proxy_pass http://backend;
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

        # WebSocket (如需要)
        location /ws/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 86400;
        }

        # 前端静态文件
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### 8.3 环境变量

```bash
# .env 示例
# 数据库
DB_PASSWORD=your_secure_password

# JWT
SECRET_KEY=your_very_long_secret_key_at_least_32_chars

# 第三方接口 (可选)
SENTRY_DSN=https://example@sentry.io/1234567
```

---

## 9. 消息队列设计

### 9.1 队列架构

使用 Redis 作为消息队列中间件，采用 Celery 分布式任务队列：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   FastAPI       │────►│     Redis       │────►│  Celery Worker  │
│   (Producer)    │     │   (Broker)      │     │  (Consumer)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │  Task Handler   │
                                                 │  - audit_task   │
                                                 │  - backup_task  │
                                                 │  - notify_task  │
                                                 │  - cleanup_task │
                                                 └─────────────────┘
```

### 9.2 任务队列类型

| 队列名称 | 说明 | 优先级 |
|----------|------|--------|
| `celery` | 默认队列 | 5 |
| `audit` | 审核相关任务 | 8 |
| `backup` | 备份相关任务 | 3 |
| `notify` | 通知任务 | 5 |
| `cleanup` | 清理任务 | 1 |

### 9.3 任务定义

#### 9.3.1 审核任务

```python
# app/tasks/audit_tasks.py
from celery import shared_task

@shared_task(name='audit.execute_rule', queue='audit', bind=True)
def execute_audit_rule(self, data_id: str, rule_ids: list):
    """执行审核规则"""
    # 1. 获取审核数据
    # 2. 加载规则
    # 3. 执行规则
    # 4. 更新数据状态
    # 5. 触发任务分配
    pass

@shared_task(name='audit.assign_task', queue='audit', bind=True)
def assign_audit_task(self, data_id: str, project_id: str):
    """分配审核任务"""
    # 1. 获取分配策略
    # 2. 选择审核员
    # 3. 创建任务记录
    # 4. 发送通知
    pass

@shared_task(name='audit.notify_auditor', queue='notify', bind=True)
def notify_auditor(self, auditor_id: str, task_count: int):
    """通知审核员有新任务"""
    # 1. 获取审核员信息
    # 2. 发送通知（站内信/短信/邮件）
    pass
```

#### 9.3.2 备份任务

```python
# app/tasks/backup_tasks.py
from celery import shared_task

@shared_task(name='backup.execute', queue='backup', bind=True)
def execute_backup(self, backup_record_id: str):
    """执行备份"""
    # 1. 更新备份状态为 running
    # 2. 执行数据库备份
    # 3. 上传备份文件
    # 4. 更新备份记录
    # 5. 清理过期备份
    pass

@shared_task(name='backup.cleanup', queue='cleanup', bind=True)
def cleanup_old_backups(self, tenant_id: str, retention_days: int):
    """清理过期备份"""
    # 1. 查找过期备份
    # 2. 删除备份文件
    # 3. 删除备份记录
    pass
```

#### 9.3.3 定时任务

```python
# app/tasks/scheduler.py
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    # 每小时执行一次任务分配检查
    'check-pending-tasks': {
        'task': 'audit.check_pending_tasks',
        'schedule': crontab(minute=0),
        'options': {'queue': 'audit'},
    },

    # 每天凌晨 2 点执行备份
    'daily-backup': {
        'task': 'backup.schedule_backup',
        'schedule': crontab(hour=2, minute=0),
        'options': {'queue': 'backup'},
    },

    # 每天凌晨 3 点清理过期备份
    'cleanup-old-backups': {
        'task': 'backup.cleanup_expired',
        'schedule': crontab(hour=3, minute=0),
        'options': {'queue': 'cleanup'},
    },

    # 每周一凌晨 1 点生成统计报告
    'weekly-statistics': {
        'task': 'statistics.generate_weekly_report',
        'schedule': crontab(hour=1, minute=0, day_of_week=1),
        'options': {'queue': 'notify'},
    },
}
```

### 9.4 Celery 配置

```python
# app/core/celery.py
from celery import Celery
from celery.schedules import crontab

def create_celery_app() -> Celery:
    app = Celery('smartaudit')

    # 连接 Redis
    app.conf.broker_url = settings.REDIS_URL
    app.conf.result_backend = settings.REDIS_URL

    # 序列化方式
    app.conf.task_serializer = 'json'
    app.conf.result_serializer = 'json'
    app.conf.accept_content = ['json']

    # 时区设置
    app.conf.timezone = 'Asia/Shanghai'
    app.conf.enable_utc = True

    # 任务路由
    app.conf.task_routes = {
        'audit.*': {'queue': 'audit'},
        'backup.*': {'queue': 'backup'},
        'notify.*': {'queue': 'notify'},
        'cleanup.*': {'queue': 'cleanup'},
    }

    # 任务限流
    app.conf.task_annotations = {
        'audit.execute_rule': {'rate_limit': '100/m'},
    }

    # 任务重试
    app.conf.task_acks_late = True
    app.conf.task_reject_on_worker_lost = True

    # 定时任务
    app.conf.beat_schedule = CELERYBEAT_SCHEDULE

    return app

celery_app = create_celery_app()
```

### 9.5 API 接口设计

#### 9.5.1 版本检查

```
GET /api/v1/system/version/check
```

**响应**:
```typescript
interface VersionCheckResponse {
  hasUpdate: boolean;
  latestVersion: {
    version: string;
    versionCode: number;
    releaseType: 'stable' | 'beta';
    releaseNotes: string;
    isMandatory: boolean;
    downloadUrl: string;
  };
  currentVersion: {
    version: string;
    versionCode: number;
  };
}
```

#### 9.5.2 执行升级

```
POST /api/v1/system/upgrade
```

**请求参数**:
```typescript
interface UpgradeRequest {
  targetVersion: string;
  createBackup: boolean;
}
```

#### 9.5.3 备份规则 CRUD

```
GET    /api/v1/system/backup/rules       # 获取备份规则列表
POST   /api/v1/system/backup/rules      # 创建备份规则
PUT    /api/v1/system/backup/rules/{id}  # 更新备份规则
DELETE /api/v1/system/backup/rules/{id} # 删除备份规则
```

**备份规则请求参数**:
```typescript
interface BackupRuleRequest {
  name: string;
  backupType: 'full' | 'differential' | 'incremental';
  scheduleCron: string;        // "0 2 * * *" 每天凌晨2点
  retentionDays: number;       // 保留天数
  storageLocation: 'local' | 's3' | 'oss';
  storageConfig: {
    bucket?: string;
    endpoint?: string;
    accessKey?: string;
    secretKey?: string;
  };
}
```

#### 9.5.4 备份记录

```
GET /api/v1/system/backup/records        # 获取备份记录列表
GET /api/v1/system/backup/records/{id}  # 获取备份详情
POST /api/v1/system/backup/execute      # 手动执行备份
```

**备份记录响应**:
```typescript
interface BackupRecordResponse {
  id: string;
  ruleId: string;
  backupType: string;
  status: 'pending' | 'running' | 'success' | 'failed';
  startedAt: string;
  completedAt: string;
  filePath: string;
  fileSize: number;
  checksum: string;
  errorMsg?: string;
}
```

---

## 10. 系统版本管理

### 10.1 版本管理架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      平台管理端                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │  版本发布管理   │  │  强制更新配置   │  │  升级监控      │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      租户管理端                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │  当前版本显示   │  │  版本升级       │  │  升级记录      │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      独立部署租户                                 │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │  本地版本信息   │  │  手动/自动升级   │  │  备份管理      │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 版本号规范

| 字段 | 规范 | 示例 |
|------|------|------|
| version | 主.次.修订 | 1.2.3 |
| version_code | 数字版本号 | 123 |
| release_type | stable/beta/release | stable |

**版本号递增规则**：
- 主版本号 (Major): 不兼容的重大架构变更
- 次版本号 (Minor): 向下兼容的新功能
- 修订版本号 (Patch): 向下兼容的问题修复

**version_code 计算**：
```
version_code = Major * 10000 + Minor * 100 + Patch
例如: 1.2.3 => 10203
```

### 10.3 升级流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       升级流程                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 平台发布新版本                                               │
│     ├── 上传安装包到 CDN/S3                                      │
│     ├── 配置版本信息（是否强制更新）                               │
│     └── 通知租户有新版本                                          │
│                                                                 │
│  2. 租户检查版本                                                 │
│     ├── 租户端定时检查 / 手动检查                                  │
│     └── 对比当前版本与最新版本                                     │
│                                                                 │
│  3. 执行升级                                                     │
│     ├── 强制更新: 直接下载并升级                                  │
│     ├── 可选更新: 租户确认后升级                                  │
│     └── 升级前自动备份                                            │
│                                                                 │
│  4. 升级完成                                                     │
│     ├── 更新版本记录                                              │
│     └── 执行数据库迁移                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.4 升级任务

```python
# app/services/upgrade_service.py
class UpgradeService:
    async def check_version(self, tenant_id: str) -> VersionCheckResponse:
        """检查新版本"""
        # 1. 获取租户当前版本
        # 2. 获取平台最新版本
        # 3. 对比并返回结果

    async def execute_upgrade(self, tenant_id: str, target_version: str) -> UpgradeRecord:
        """执行升级"""
        # 1. 创建升级记录
        # 2. 创建备份（如需要）
        # 3. 下载新版本包
        # 4. 执行升级脚本
        # 5. 执行数据库迁移
        # 6. 更新版本记录
        # 7. 清理旧文件

    async def rollback(self, upgrade_record_id: str) -> bool:
        """回滚"""
        # 1. 获取升级记录
        # 2. 恢复备份
        # 3. 恢复数据库
        # 4. 更新状态
```

---

## 11. 备份管理

### 11.1 备份策略

| 备份类型 | 说明 | 执行频率 | 保留策略 |
|----------|------|----------|----------|
| 全量备份 (Full) | 完整数据库备份 | 每周一次 | 保留 30 天 |
| 差异备份 (Differential) | 与上次全量备份的差异 | 每天一次 | 保留 7 天 |
| 增量备份 (Incremental) | 与上次备份的差异 | 每小时一次 | 保留 3 天 |

### 11.2 备份流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       备份流程                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 触发备份                                                     │
│     ├── 定时任务触发                                              │
│     └── 手动触发                                                  │
│                                                                 │
│  2. 备份执行                                                     │
│     ├── 创建备份记录 (status=pending)                             │
│     ├── 更新状态为 running                                        │
│     ├── 执行 PostgreSQL 备份                                      │
│     │   └── pg_dump -Fc smartaudit > backup.dump                │
│     ├── 压缩备份文件                                              │
│     └── 计算文件校验码                                             │
│                                                                 │
│  3. 上传存储                                                     │
│     ├── 上传到 MinIO/S3/OSS                                      │
│     └── 更新备份记录 (file_path)                                  │
│                                                                 │
│  4. 备份完成                                                     │
│     ├── 更新状态为 success                                        │
│     ├── 更新备份时间                                              │
│     └── 触发清理过期备份                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.3 备份服务

```python
# app/services/backup_service.py
import subprocess
import hashlib
from datetime import datetime
from pathlib import Path

class BackupService:
    def __init__(self, storage: StorageService):
        self.storage = storage

    async def execute_backup(self, rule_id: str) -> BackupRecord:
        """执行备份"""
        rule = await self.get_rule(rule_id)

        # 创建备份记录
        record = await self.create_record(rule, status='running')

        try:
            # 执行数据库备份
            backup_file = await self._dump_database(rule.tenant_id)

            # 压缩
            compressed_file = await self._compress(backup_file)

            # 计算校验码
            checksum = await self._calculate_checksum(compressed_file)

            # 上传到存储
            file_path = await self.storage.upload(
                compressed_file,
                f"backup/{rule.tenant_id}/{record.id}.tar.gz"
            )

            # 更新记录
            record.status = 'success'
            record.file_path = file_path
            record.file_size = Path(compressed_file).stat().st_size
            record.checksum = checksum
            record.completed_at = datetime.utcnow()

            await self.save_record(record)

            return record

        except Exception as e:
            record.status = 'failed'
            record.error_msg = str(e)
            await self.save_record(record)
            raise

    async def cleanup_old_backups(self, tenant_id: str, retention_days: int):
        """清理过期备份"""
        cutoff_date = datetime.utcnow() - timedelta(days=retention_days)

        old_records = await self.get_records_before(tenant_id, cutoff_date)

        for record in old_records:
            if record.file_path:
                await self.storage.delete(record.file_path)
            await self.delete_record(record)

    async def _dump_database(self, tenant_id: str) -> Path:
        """执行 pg_dump"""
        backup_dir = Path("/tmp/backups")
        backup_dir.mkdir(exist_ok=True)

        filename = backup_dir / f"{tenant_id}_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}.dump"

        cmd = [
            'pg_dump',
            '-h', settings.DB_HOST,
            '-U', settings.DB_USER,
            '-d', settings.DB_NAME,
            '-Fc',
            '-f', str(filename)
        ]

        subprocess.run(cmd, env={**os.environ, 'PGPASSWORD': settings.DB_PASSWORD})

        return filename
```

### 11.4 存储配置

```typescript
interface StorageConfig {
  type: 'local' | 'minio' | 's3' | 'oss';

  local?: {
    path: string;  // "/data/backups"
  };

  minio?: {
    endpoint: string;
    bucket: string;
    accessKey: string;
    secretKey: string;
    secure: boolean;
  };

  s3?: {
    region: string;
    bucket: string;
    accessKeyId: string;
    secretAccessKey: string;
  };

  oss?: {
    endpoint: string;
    bucket: string;
    accessKeyId: string;
    secretAccessKey: string;
  };
}
```

---

## 12. 开发规范

### 9.1 Git 规范

#### 9.1.1 分支命名

| 分支类型 | 命名规范 | 示例 |
|----------|----------|------|
| 主分支 | main | main |
| 开发分支 | develop | develop |
| 功能分支 | feature/{issue-id}-{short-desc} | feature/123-add-login |
| 修复分支 | fix/{issue-id}-{short-desc} | fix/456-login-error |
| 发布分支 | release/v{version} | release/v1.0.0 |

#### 9.1.2 Commit 规范

```
<type>(<scope>): <subject>

<body>

<footer>
```

| type | 说明 |
|------|------|
| feat | 新功能 |
| fix | 修复 bug |
| docs | 文档更新 |
| style | 代码格式 |
| refactor | 重构 |
| test | 测试 |
| chore | 构建/工具 |

**示例**:
```
feat(auth): add login with phone verification

- add phone verification flow
- update login API to support phone login
- add SMS service integration

Closes #123
```

### 9.2 代码审查

| 检查项 | 说明 |
|--------|------|
| 功能完整性 | 是否满足需求 |
| 代码质量 | 可读性、可维护性 |
| 安全性 | 是否有安全漏洞 |
| 测试覆盖 | 核心逻辑是否有测试 |
| 性能 | 是否有性能问题 |
| 规范遵循 | 是否遵循项目规范 |

### 9.3 文档要求

| 文档 | 说明 |
|------|------|
| README | 项目说明、运行指南 |
| API 文档 | 自动生成 (Swagger/OpenAPI) |
| 代码注释 | 复杂逻辑需注释 |
| CHANGELOG | 版本变更记录 |

### 9.4 开发流程

```
1. 从 develop 创建功能分支
2. 在本地开发并测试
3. 提交 Pull Request
4. 代码审查
5. 合并到 develop
6. 发布前合并到 main
```

---

## 附录

### 附录 A：数据库迁移命令

```bash
# 创建迁移
alembic revision --autogenerate -m "add user table"

# 执行迁移
alembic upgrade head

# 回滚
alembic downgrade -1
```

### 附录 B：常用命令

```bash
# 前端启动
cd frontend && npm install && npm run dev

# 后端启动
cd backend && pip install -r requirements.txt && uvicorn app.main:app --reload

# Docker 构建
docker-compose build

# 日志查看
docker-compose logs -f backend
```

### 附录 C：环境要求

| 组件 | 最低要求 | 推荐 |
|------|----------|------|
| CPU | 2 核 | 4 核 |
| 内存 | 4 GB | 8 GB |
| 磁盘 | 50 GB | 100 GB |
| PostgreSQL | 13+ | 15+ |
| Redis | 6+ | 7+ |

---

## 13. MVP 版本规划

### 13.1 版本阶段划分

```
┌─────────────────────────────────────────────────────────────────┐
│                        SmartAudit 版本规划                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │    MVP      │───►│   V1.1      │───►│   V1.2      │   ...   │
│  │  (最小可用)  │    │  (完善功能)  │    │  (增强优化)  │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│        │                  │                  │                  │
│        ▼                  ▼                  ▼                  │
│   核心审核流程        权限体系完善        移动端优化             │
│   基础CRUD           三方接口           高级统计               │
│   基础统计           备份管理           独立部署               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 MVP 版本功能清单

**上线时间**: 第 1-2 个月

| 模块 | 功能点 | 优先级 | 说明 |
|------|--------|--------|------|
| **认证系统** | 用户注册/登录 | P0 | 基础登录功能 |
| | 密码找回 | P1 | 邮箱找回 |
| | JWT Token | P0 | 状态保持 |
| **租户管理** | 租户注册 | P0 | 基础信息 |
| | 租户配置 | P1 | 基本参数 |
| **账户管理** | 账户CRUD | P0 | 增删改查 |
| | 角色分配 | P0 | 基础角色（3种） |
| | 基础权限 | P0 | 按角色授权 |
| **项目管理** | 项目CRUD | P0 | 增删改查 |
| | 字段配置 | P0 | 入参定义 |
| **规则引擎** | 规则创建 | P0 | 基础条件 |
| | 规则执行 | P0 | 单条执行 |
| | 规则结果展示 | P0 | 匹配详情 |
| **任务系统** | 任务分配 | P0 | 手动分配 |
| | 任务列表 | P0 | 待审核/已审核 |
| | 审核执行 | P0 | 通过/拒绝 |
| **数据管理** | 数据提交 | P0 | 单条提交 |
| | 数据列表 | P0 | 列表展示 |
| **统计报表** | 基础统计 | P1 | 数量统计 |
| **前端** | PC Web | P0 | 响应式布局 |

### 13.3 V1.1 版本功能清单

**上线时间**: 第 3-4 个月

| 模块 | 功能点 | 优先级 | 说明 |
|------|--------|--------|------|
| **权限系统** | 精细化权限 | P0 | 43项权限 |
| | 多角色叠加 | P0 | 权限并集 |
| | 额外权限配置 | P1 | 精确到单个权限 |
| **账户管理** | 个人资料 | P0 | 头像/邮箱/手机 |
| | 密码修改 | P0 | 验证旧密码 |
| | 登录日志 | P1 | 设备/时间/IP |
| **规则引擎** | 条件组嵌套 | P0 | AND/OR组合 |
| | 条件运算符扩展 | P0 | 区间/正则/验证 |
| **任务系统** | 自动分配策略 | P0 | 轮询/负载均衡 |
| | 二次校验 | P0 | 人工复核 |
| | 任务统计 | P0 | 个人统计 |
| **三方接口** | 接口配置 | P0 | 认证/Token |
| | 接口调用 | P0 | 身份证验证 |
| | 调用日志 | P1 | 记录追溯 |
| **备份管理** | 备份规则 | P1 | 自动备份 |
| | 备份记录 | P1 | 查看历史 |

### 13.4 V1.2 版本功能清单

**上线时间**: 第 5-6 个月

| 模块 | 功能点 | 优先级 | 说明 |
|------|--------|--------|------|
| **系统版本** | 版本管理 | P0 | 查看当前版本 |
| | 版本升级 | P0 | 一键升级 |
| | 强制更新 | P1 | 配置强制更新 |
| **移动端** | H5 适配 | P0 | 审核工作台 |
| | 小程序 | P1 | 扫码/快速审核 |
| **统计报表** | 项目统计 | P0 | 完成度/工作量 |
| | 三方接口统计 | P1 | 成功率/耗时 |
| | 登录统计 | P1 | 设备分布 |
| **独立部署** | 部署包 | P0 | Docker 镜像 |
| | 备份存储 | P0 | S3/OSS |
| | 自动升级 | P0 | 在线升级 |
| **消息通知** | 站内通知 | P1 | 任务提醒 |
| | 邮件通知 | P2 | 通知提醒 |
| **系统优化** | 性能优化 | P1 | 大数据量优化 |
| | 离线缓存 | P2 | 断网续审 |

### 13.5 后续版本规划

| 版本 | 目标 | 关键功能 |
|------|------|----------|
| V2.0 | 企业级增强 | 多级审批流、自定义表单、API开放平台 |
| V2.1 | 数据分析 | BI报表、数据导出、数据对比 |
| V2.2 | AI 增强 | 智能推荐、自动分类、异常检测 |
| V3.0 | 平台化 | 多租户SaaS、 marketplace、独立部署增强 |

### 13.6 MVP 技术债务清单

| 类别 | 项目 | 处理时机 | 说明 |
|------|------|----------|------|
| 安全性 | JWT 短期token | V1.0 | MVP 先用短期token |
| 性能 | 索引优化 | V1.1 | 按需加索引 |
| 监控 | 日志完善 | V1.1 | 结构化日志 |
| 运维 | Docker化 | V1.0 | MVP 完成后 |
| 文档 | API 文档 | V1.1 | OpenAPI 自动生成 |

### 13.7 MVP 里程碑

| 里程碑 | 目标日期 | 交付内容 |
|--------|----------|----------|
| M1: 需求冻结 | 第 1 周 | 需求文档确认 |
| M2: 设计完成 | 第 2 周 | UI/UX 设计 |
| M3: 功能开发 | 第 4 周 | MVP 功能开发 |
| M4: 测试完成 | 第 6 周 | 全功能测试 |
| M5: 正式上线 | 第 8 周 | MVP 发布 |

---

*本文档版本: 1.0.1*
*最后更新: 2026-03-20*
