# SaaS 管理平台系统设计

## 1. 系统概述

SaaS 管理平台是平台运营方使用的管理系统，负责管理所有租户、系统版本、全局配置和运营统计。

**使用对象**：平台运营方管理员

## 2. 技术架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SaaS 管理后台 (React + Tailwind)            │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx (反向代理 + SSL)                      │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FastAPI 服务集群                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  SaaS API   │  │  SaaS API   │  │  SaaS API   │             │
│  │   Instance  │  │   Instance  │  │   Instance  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   PostgreSQL    │  │     Redis       │  │      S3/OSS     │
│   (SaaS DB)     │  │   (会话/缓存)   │  │   (文件存储)    │
│   独立数据库     │  │                 │  │                  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 2.2 技术栈

| 层级 | 技术选型 | 版本 |
|------|----------|------|
| 前端框架 | React | 18.x |
| 前端构建 | Vite | 5.x |
| UI 框架 | Tailwind CSS | 3.x |
| 状态管理 | Zustand | 4.x |
| 后端框架 | FastAPI | 0.109+ |
| Python 版本 | Python | 3.11+ |
| ORM | SQLAlchemy | 2.x |
| 数据库 | PostgreSQL | 15+ |
| 缓存 | Redis | 7+ |
| 文件存储 | S3 / OSS | - |

### 2.3 目录结构

```
saas-backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 应用入口
│   │
│   ├── api/                    # API 路由
│   │   ├── deps.py             # 依赖注入
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── router.py       # 路由汇总
│   │       ├── auth.py         # 认证接口
│   │       ├── tenant.py       # 租户管理
│   │       ├── version.py      # 版本管理
│   │       ├── config.py       # 配置管理
│   │       └── statistics.py   # 统计接口
│   │
│   ├── core/                   # 核心配置
│   │   ├── __init__.py
│   │   ├── config.py           # 配置管理
│   │   ├── security.py         # 安全工具
│   │   ├── database.py         # 数据库连接
│   │   └── redis.py            # Redis 连接
│   │
│   ├── models/                 # SQLAlchemy 模型
│   │   ├── __init__.py
│   │   ├── saas_admin.py      # 平台管理员
│   │   ├── tenant.py          # 租户
│   │   ├── system_version.py  # 系统版本
│   │   ├── config.py          # 全局配置
│   │   └── statistics.py      # 统计数据
│   │
│   ├── schemas/                # Pydantic 模型
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── tenant.py
│   │   ├── version.py
│   │   ├── config.py
│   │   └── statistics.py
│   │
│   ├── services/               # 业务逻辑
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── tenant_service.py
│   │   ├── version_service.py
│   │   └── statistics_service.py
│   │
│   └── utils/                  # 工具函数
│       └── pagination.py
│
├── migrations/                  # Alembic 数据库迁移
├── tests/                       # 测试
├── pyproject.toml
└── alembic.ini
```

## 3. 核心功能

### 3.1 平台管理员管理

**功能说明**：
- 平台超级管理员（SYSTEM_SUPER_ADMIN）是系统内置的，不可删除
- 初始登录后强制修改密码
- 管理员可以修改自己的资料和密码

**数据模型**：

```python
class SaaSAdmin(Base):
    """平台管理员"""
    __tablename__ = "saas_admin"

    id: UUID                    # 主键
    username: str               # 用户名，唯一
    password_hash: str          # 密码哈希
    name: str                   # 姓名
    email: str                  # 邮箱
    phone: str                  # 手机号
    avatar_url: str             # 头像
    role: str                   # SYSTEM_SUPER_ADMIN
    status: str                 # active/inactive
    last_login_at: datetime     # 最后登录时间
    last_login_ip: str          # 最后登录IP
    created_at: datetime
    updated_at: datetime
```

### 3.2 租户管理

**功能说明**：
- 创建、编辑、禁用/启用租户
- 为租户生成 APPID 和 APPSECRET
- 查看租户详情和统计数据

**数据模型**：

```python
class Tenant(Base):
    """租户"""
    __tablename__ = "tenant"

    id: UUID
    name: str                    # 租户名称
    code: str                    # 租户代码，唯一
    contact_name: str            # 联系人姓名
    contact_email: str           # 联系人邮箱
    contact_phone: str           # 联系人电话
    address: str                 # 地址
    logo_url: str                # Logo URL
    app_id: str                  # 应用ID
    app_secret_hash: str         # 应用密钥哈希
    package_type: str            # 套餐类型: basic/professional/enterprise
    deploy_mode: str              # 部署模式: saas/dedicated
    service_start_date: date     # 服务开始日期
    service_end_date: date       # 服务结束日期
    status: str                   # active/inactive/expired
    created_at: datetime
    updated_at: datetime

    # 关联
    users: List[User]           # 租户下的用户
    projects: List[Project]     # 租户下的项目
```

### 3.3 版本发布管理

**功能说明**：
- 平台管理员发布新版本
- 配置版本是否强制更新
- 查看租户升级情况

**数据模型**：

```python
class SystemVersion(Base):
    """系统版本"""
    __tablename__ = "system_version"

    id: UUID
    version: str                 # 版本号 "1.2.3"
    version_code: int            # 版本数字 10203
    release_type: str            # stable/beta/release
    release_notes: str           # 发布说明 (Markdown)
    download_url: str            # 下载地址
    checksum: str                # SHA256 校验码
    is_mandatory: bool            # 是否强制更新
    is_enabled: bool             # 是否启用
    min_compatible_version: str  # 最低兼容版本
    published_at: datetime       # 发布时间
    created_at: datetime
    updated_at: datetime

    # 关联
    upgrade_records: List[UpgradeRecord]  # 升级记录


class UpgradeRecord(Base):
    """升级记录"""
    __tablename__ = "upgrade_record"

    id: UUID
    tenant_id: UUID              # 租户ID
    from_version: str            # 从版本
    to_version: str              # 到版本
    status: str                  # pending/running/success/failed/rollback
    backup_id: UUID              # 关联备份ID
    error_msg: str               # 错误信息
    rollback_note: str           # 回滚备注
    started_at: datetime
    completed_at: datetime
    operated_by: UUID            # 操作人
    created_at: datetime
```

### 3.4 全局配置

**数据模型**：

```python
class SaaSConfig(Base):
    """SaaS 全局配置"""
    __tablename__ = "saas_config"

    id: UUID
    config_key: str              # 配置键，唯一
    config_value: str            # 配置值
    config_type: str            # string/number/boolean/json
    description: str             # 说明
    created_at: datetime
    updated_at: datetime

# 配置项
PLATFORM_NAME = "platform_name"
PLATFORM_LOGO = "platform_logo"
DEFAULT_PACKAGE = "default_package"
TOKEN_EXPIRE_MINUTES = "token_expire_minutes"
MAX_USERS_PER_TENANT = "max_users_per_tenant"
MAX_PROJECTS_PER_TENANT = "max_projects_per_tenant"
```

## 4. API 接口

### 4.1 认证接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /saas/api/v1/auth/login | 登录 |
| POST | /saas/api/v1/auth/logout | 登出 |
| GET | /saas/api/v1/auth/me | 获取当前管理员 |
| PUT | /saas/api/v1/auth/password | 修改密码 |

### 4.2 租户管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /saas/api/v1/tenants | 租户列表 |
| POST | /saas/api/v1/tenants | 开通租户 |
| GET | /saas/api/v1/tenants/{id} | 租户详情 |
| PUT | /saas/api/v1/tenants/{id} | 编辑租户 |
| DELETE | /saas/api/v1/tenants/{id} | 删除租户 |
| POST | /saas/api/v1/tenants/{id}/disable | 禁用租户 |
| POST | /saas/api/v1/tenants/{id}/enable | 启用租户 |
| POST | /saas/api/v1/tenants/{id}/reset-secret | 重置APPSECRET |

### 4.3 版本管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /saas/api/v1/versions | 版本列表 |
| POST | /saas/api/v1/versions | 发布版本 |
| GET | /saas/api/v1/versions/{id} | 版本详情 |
| PUT | /saas/api/v1/versions/{id} | 编辑版本 |
| POST | /saas/api/v1/versions/{id}/disable | 禁用版本 |
| GET | /saas/api/v1/versions/{id}/upgrade-records | 升级记录 |

### 4.4 配置接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /saas/api/v1/config | 获取配置 |
| PUT | /saas/api/v1/config | 更新配置 |
| GET | /saas/api/v1/config/{key} | 获取单个配置 |

### 4.5 统计接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /saas/api/v1/statistics/overview | 运营概览 |
| GET | /saas/api/v1/statistics/tenants | 租户统计 |
| GET | /saas/api/v1/statistics/versions | 版本统计 |

## 5. 响应格式

### 5.1 成功响应

```json
{
  "code": 0,
  "message": "success",
  "data": { ... },
  "timestamp": 1709308800000
}
```

### 5.2 分页响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "pageSize": 20,
    "totalPages": 5
  },
  "timestamp": 1709308800000
}
```

### 5.3 错误响应

```json
{
  "code": 10001,
  "message": "参数错误",
  "details": { ... },
  "requestId": "req_abc123",
  "timestamp": 1709308800000
}
```

## 6. 错误码

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

## 7. 安全设计

### 7.1 认证

- JWT Token 认证
- Token 有效期：可配置（默认 30 分钟）
- Refresh Token 有效期：7 天

### 7.2 密码策略

- 密码最小长度：8 位
- 密码加密：bcrypt
- 登录失败锁定：5 次后锁定 15 分钟

### 7.3 日志审计

- 记录所有管理员操作
- 记录登录/登出事件
- 日志保留 180 天
