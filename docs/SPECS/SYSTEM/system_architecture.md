# 系统架构设计

## 1. 整体架构

### 1.1 双系统模式

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SmartAudit 智核                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────┐  ┌──────────────────────────────┐  │
│   │         SaaS 管理平台           │  │       租户管理系统             │  │
│   │      (平台运营方使用)           │  │     (各租户独立使用)           │  │
│   │                                 │  │                              │  │
│   │  ┌─────────────────────────┐  │  │  ┌────────────────────────┐  │  │
│   │  │ 租户管理                  │  │  │  │ 项目管理                │  │  │
│   │  │ - 租户开通/禁用           │  │  │  │ - 项目 CRUD            │  │  │
│   │  │ - 租户配置                │  │  │  │ - 字段配置              │  │  │
│   │  │ - 用量统计               │  │  │  └────────────────────────┘  │  │
│   │  └─────────────────────────┘  │  │                              │  │
│   │                                │  │  ┌────────────────────────┐  │  │
│   │  ┌─────────────────────────┐  │  │  │ 规则引擎                │  │  │
│   │  │ 版本发布管理             │  │  │  │ - 规则配置              │  │  │
│   │  │ - 版本发布               │  │  │  │ - 条件组嵌套            │  │  │
│   │  │ - 升级推送               │  │  │  │ - 执行引擎              │  │  │
│   │  └─────────────────────────┘  │  │  └────────────────────────┘  │  │
│   │                                │  │                              │  │
│   │  ┌─────────────────────────┐  │  │  ┌────────────────────────┐  │  │
│   │  │ 独立部署版本生成         │  │  │  │ 任务审核                │  │  │
│   │  │ - 打包导出               │  │  │  │ - 任务分配              │  │  │
│   │  │ - 版本文件管理           │  │  │  │ - 审核执行              │  │  │
│   │  └─────────────────────────┘  │  │  │ - 二次校验              │  │  │
│   │                                │  │  └────────────────────────┘  │  │
│   │  ┌─────────────────────────┐  │  │                              │  │
│   │  │ 系统配置                │  │  │  ┌────────────────────────┐  │  │
│   │  │ - 全局参数配置          │  │  │  │ 三方接口                │  │  │
│   │  │ - 第三方服务配置        │  │  │  │ - 接口配置              │  │  │
│   │  └─────────────────────────┘  │  │  │ - 调用日志              │  │  │
│   │                                │  │  └────────────────────────┘  │  │
│   │  ┌─────────────────────────┐  │  │                              │  │
│   │  │ 运营统计                │  │  │  ┌────────────────────────┐  │  │
│   │  │ - 租户统计              │  │  │  │ 数据统计                │  │  │
│   │  │ - 收入统计              │  │  │  │ - 项目统计              │  │  │
│   │  └─────────────────────────┘  │  │  │ - 审核统计              │  │  │
│   └─────────────────────────────────┘  │  └────────────────────────┘  │  │
│                                        │                              │  │
│                                        │  ┌────────────────────────┐  │  │
│                                        │  │ 账户权限                │  │  │
│                                        │  │ - 账户管理              │  │  │
│                                        │  │ - 角色权限              │  │  │
│                                        │  └────────────────────────┘  │  │
│                                        │                              │  │
│                                        │  ┌────────────────────────┐  │  │
│                                        │  │ 版本管理（独立部署）    │  │  │
│                                        │  │ - 版本查看              │  │  │
│                                        │  │ - 在线升级              │  │  │
│                                        │  │ - 回滚操作              │  │  │
│                                        │  └────────────────────────┘  │  │
│                                        │                              │  │
│                                        │  ┌────────────────────────┐  │  │
│                                        │  │ 备份管理（独立部署）    │  │  │
│                                        │  │ - 备份规则              │  │  │
│                                        │  │ - 备份记录              │  │  │
│                                        │  │ - 数据恢复              │  │  │
│                                        │  └────────────────────────┘  │  │
│                                        └──────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 技术栈

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| 前端 | React 18 + TypeScript + Vite | PC Web |
| 前端 | React + Vant（移动端） | Mobile H5 |
| 前端 | 原生开发 | 微信小程序 |
| 后端 | Python 3.11 + FastAPI | 核心业务服务 |
| 后端 | SQLAlchemy 2.x + Pydantic 2.x | ORM 和验证 |
| 数据库 | PostgreSQL 15+ | 主数据存储 |
| 缓存 | Redis 7+ | 缓存/会话/队列 |
| 任务队列 | Celery + Redis | 异步任务 |
| 文件存储 | MinIO / S3 / OSS | 文件存储 |
| API 文档 | OpenAPI 3.0 / Swagger | API 文档 |

## 2. SaaS 多租户架构

### 2.1 多租户数据隔离

#### 方案：行级隔离 + 租户 ID
```
┌─────────────────────────────────────────────────────────┐
│                    sys_tenants（平台表）                  │
├─────┬────────┬─────────┬───────────────────────────────┤
│ id  │  name  │  code   │  status                       │
├─────┼────────┼─────────┼───────────────────────────────┤
│  1  │ 租户A  │ tenant_a │  active                       │
│  2  │ 租户B  │ tenant_b │  active                       │
└─────┴────────┴─────────┴───────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 biz_projects（租户业务表）               │
├─────┬───────────┬─────────┬─────────────────────────────┤
│ id  │ tenant_id │  name   │  is_deleted                 │
├─────┼───────────┼─────────┼─────────────────────────────┤
│  1  │     1     │ 项目A1  │  0                           │
│  2  │     1     │ 项目A2  │  0                           │
│  3  │     2     │ 项目B1  │  0                           │
└─────┴───────────┴─────────┴─────────────────────────────┘
```

### 2.2 租户隔离实现

```python
# app/core/tenant_context.py
from contextvars import ContextVar
from typing import Optional

tenant_id: ContextVar[Optional[int]] = ContextVar('tenant_id', default=None)

def get_tenant_id() -> Optional[int]:
    return tenant_id.get()

def set_tenant_id(tid: int):
    tenant_id.set(tid)

# 中间件实现
class TenantMiddleware:
    async def __call__(self, request: Request, call_next):
        tenant_id = request.headers.get('X-Tenant-ID')
        if tenant_id:
            set_tenant_id(int(tenant_id))
        try:
            response = await call_next(request)
            return response
        finally:
            tenant_id.reset()
```

## 3. 独立部署架构

### 3.1 独立部署模式
```
┌─────────────────────────────────────────────────────────┐
│                   客户独立部署服务器                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │   Nginx     │  │  FastAPI    │  │    PostgreSQL   │  │
│  │  (反向代理)  │──│  (主服务)   │──│    (数据库)     │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│                        │                    │           │
│                   ┌────┴────┐              │           │
│                   │         │              │           │
│              ┌────┴───┐ ┌───┴────┐        │           │
│              │ Celery │ │  Redis  │        │           │
│              │ (Worker)│ │ (缓存)  │        │           │
│              └────────┘ └────────┘        │           │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │                   租户管理后台                         ││
│  │  (包含完整的租户管理系统功能)                          ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### 3.2 SaaS 与独立部署代码复用

```
smartaudit/
├── smartaudit-core/              # 核心业务模块（可复用）
│   ├── business/                 # 业务逻辑
│   │   ├── project/             # 项目管理
│   │   ├── rule/                # 规则引擎
│   │   ├── task/                # 任务审核
│   │   ├── account/             # 账户权限
│   │   └── statistics/         # 统计报表
│   ├── models/                  # 数据模型
│   ├── schemas/                 # API Schema
│   └── services/                # 服务层
│
├── smartaudit-saas/             # SaaS 平台特有
│   ├── tenant_management/       # 租户管理
│   ├── version_release/        # 版本发布
│   ├── independent_deploy/     # 独立部署版本生成
│   └── billing/                # 计费管理
│
├── smartaudit-tenant/           # 租户系统（独立部署时使用）
│   ├── project/
│   ├── rule/
│   ├── task/
│   ├── account/
│   ├── statistics/
│   ├── version_management/     # 版本管理
│   └── backup/                # 备份管理
│
├── smartaudit-deploy/          # 部署配置
│   ├── docker/
│   ├── kubernetes/
│   └── scripts/
│
└── tests/                      # 测试
    ├── core/                   # 核心模块测试
    ├── saas/                  # SaaS 测试
    └── tenant/                 # 租户系统测试
```

## 4. 部署模式对比

### 4.1 SaaS 模式 vs 独立部署模式

| 特性 | SaaS 多租户 | 独立部署 |
|------|-------------|----------|
| 部署位置 | 平台方服务器 | 客户方服务器 |
| 数据隔离 | 行级隔离（tenant_id） | 独立数据库 |
| 资源占用 | 共享资源 | 独占资源 |
| 系统升级 | 平台统一升级 | 客户自主升级 |
| 运维成本 | 低（平台方负责） | 高（客户负责） |
| 定制化 | 有限 | 完全支持 |
| 费用模式 | 按月/年订阅 | 一次性授权 |

### 4.2 独立部署版本生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     独立部署版本生成流程                          │
└─────────────────────────────────────────────────────────────────┘

1. 平台管理员选择租户
         │
         ▼
2. 选择基础版本（如 v1.2.0）
         │
         ▼
3. 应用租户定制配置
   - 租户 Logo
   - 自定义字段
   - 特殊规则模板
         │
         ▼
4. 生成部署包
   - Docker 镜像
   - 配置文件
   - 数据库脚本
   - 部署脚本
         │
         ▼
5. 下载/推送到客户服务器
         │
         ▼
6. 客户执行部署
```

## 5. 系统模块设计

### 5.1 模块依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│                    (统一入口/鉴权/限流)                           │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  租户管理模块   │     │  项目管理模块   │     │  规则引擎模块   │
│  (SaaS 平台)   │     │               │     │               │
└───────────────┘     └───────────────┘     └───────────────┘
        │                     │                     │
        │                     └──────────┬────────────┘
        │                                │
        │                     ┌───────────────┐
        │                     │  任务审核模块   │
        │                     │               │
        │                     └───────────────┘
        │                                │
        └────────────────┬────────────────┘
                         ▼
              ┌─────────────────────┐
              │     账户权限模块      │
              └─────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │     统计报表模块      │
              └─────────────────────┘
```

### 5.2 核心模块功能

#### 租户管理模块（SaaS）
- 租户开通/禁用
- 租户配置管理
- 用量统计
- 独立部署版本生成
- 版本升级推送

#### 项目管理模块
- 项目 CRUD
- 项目字段配置
- 项目统计

#### 规则引擎模块
- 规则配置
- 条件组嵌套
- 规则执行
- 第三方数据校验

#### 任务审核模块
- 任务分配
- 审核执行
- 二次校验
- 任务统计

#### 账户权限模块
- 账户管理
- 角色配置
- 权限管理

## 6. 数据库架构

### 6.1 SaaS 平台数据库

```sql
-- 租户表
CREATE TABLE saas_tenants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    logo VARCHAR(255),
    contact VARCHAR(50),
    phone VARCHAR(20),
    email VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active',
    deploy_mode VARCHAR(20) DEFAULT 'saas',  -- saas / independent
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    version INTEGER DEFAULT 0
);

-- 版本表
CREATE TABLE saas_versions (
    id BIGSERIAL PRIMARY KEY,
    version_code VARCHAR(20) NOT NULL,
    version_name VARCHAR(50),
    description TEXT,
    release_date TIMESTAMP,
    is_force_update SMALLINT DEFAULT 0,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 租户版本记录
CREATE TABLE saas_tenant_versions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    version_id BIGINT NOT NULL,
    deploy_package_url VARCHAR(255),
    deployed_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pending',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 6.2 租户业务数据库

```sql
-- 账户表
CREATE TABLE biz_accounts (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(50),
    phone VARCHAR(20),
    email VARCHAR(100),
    avatar VARCHAR(255),
    status VARCHAR(20) DEFAULT 'active',
    last_login_at TIMESTAMP,
    last_login_ip VARCHAR(50),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    version INTEGER DEFAULT 0,
    UNIQUE(tenant_id, username)
);

-- 角色表
CREATE TABLE biz_roles (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description VARCHAR(255),
    is_system SMALLINT DEFAULT 0,  -- 是否系统内置
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

-- 账户角色关联表
CREATE TABLE biz_account_roles (
    id BIGSERIAL PRIMARY KEY,
    account_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 权限表
CREATE TABLE biz_permissions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    code VARCHAR(50) NOT NULL,
    name VARCHAR(50) NOT NULL,
    module VARCHAR(50),
    description VARCHAR(255),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

-- 角色权限关联表
CREATE TABLE biz_role_permissions (
    id BIGSERIAL PRIMARY KEY,
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 项目表
CREATE TABLE biz_projects (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description TEXT,
    field_config JSONB,  -- 字段配置
    rule_config JSONB,    -- 规则配置
    need_manual_verify SMALLINT DEFAULT 1,
    task_assign_strategy VARCHAR(20) DEFAULT 'round_robin',
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    version INTEGER DEFAULT 0,
    UNIQUE(tenant_id, code)
);

-- 规则表
CREATE TABLE biz_rules (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    rule_config JSONB NOT NULL,  -- 规则配置，包含条件组
    priority INTEGER DEFAULT 0,
    is_active SMALLINT DEFAULT 1,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

-- 任务表
CREATE TABLE biz_tasks (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    auditor_id BIGINT,
    submit_data JSONB NOT NULL,    -- 提交的审核数据
    rule_result JSONB,             -- 规则执行结果
    final_result VARCHAR(20),      -- pending / approved / rejected
    audit_status VARCHAR(20) DEFAULT 'pending',  -- pending / first_audit / recheck
    reject_reason TEXT,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    audited_at TIMESTAMP,
    version INTEGER DEFAULT 0
);

-- 审核日志表
CREATE TABLE biz_audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    task_id BIGINT NOT NULL,
    auditor_id BIGINT,
    action VARCHAR(20),     -- submit / audit / recheck
    before_status VARCHAR(20),
    after_status VARCHAR(20),
    comment TEXT,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 第三方接口配置表
CREATE TABLE biz_third_party_configs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,  -- 接口配置
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 第三方接口调用日志表
CREATE TABLE biz_third_party_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    config_id BIGINT NOT NULL,
    request_data JSONB,
    response_data JSONB,
    status VARCHAR(20),
    error_message TEXT,
    duration INTEGER,  -- 耗时毫秒
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 7. API 架构

### 7.1 API 路径设计

```
/api/v1/
├── auth/                      # 认证
│   ├── login
│   ├── logout
│   ├── refresh
│   └── me
│
├── saas/                      # SaaS 平台 API
│   ├── tenants/
│   │   ├── GET    /tenants              # 租户列表
│   │   ├── POST   /tenants              # 创建租户
│   │   ├── GET    /tenants/{id}         # 租户详情
│   │   ├── PUT    /tenants/{id}         # 更新租户
│   │   ├── DELETE /tenants/{id}         # 删除租户
│   │   ├── POST   /tenants/{id}/enable  # 启用租户
│   │   └── POST   /tenants/{id}/disable # 禁用租户
│   │
│   ├── versions/              # 版本管理
│   │   ├── GET    /versions             # 版本列表
│   │   ├── POST   /versions             # 发布版本
│   │   ├── GET    /versions/{id}        # 版本详情
│   │   └── POST   /tenants/{tid}/deploy # 生成部署包
│   │
│   └── statistics/            # 运营统计
│
└── tenant/                    # 租户业务 API
    ├── accounts/
    ├── roles/
    ├── permissions/
    ├── projects/
    │   ├── GET    /projects
    │   ├── POST   /projects
    │   ├── GET    /projects/{id}
    │   ├── PUT    /projects/{id}
    │   └── DELETE /projects/{id}
    │
    ├── rules/
    │
    ├── tasks/
    │   ├── GET    /tasks                 # 任务列表
    │   ├── GET    /tasks/{id}            # 任务详情
    │   ├── POST   /tasks/{id}/audit      # 执行审核
    │   └── POST   /tasks/{id}/recheck    # 二次校验
    │
    ├── statistics/
    │
    ├── third-party/
    │   ├── configs/
    │   └── logs/
    │
    ├── version/              # 版本管理（独立部署）
    │
    └── backup/              # 备份管理（独立部署）
```

## 8. 安全架构

### 8.1 认证流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        认证流程                                  │
└─────────────────────────────────────────────────────────────────┘

1. 用户登录
   POST /api/v1/auth/login
   {
     "username": "xxx",
     "password": "xxx"
   }

2. 服务端验证
   - 验证用户名密码
   - 生成 JWT Access Token
   - 生成 Refresh Token
   - 记录登录日志

3. 返回 Token
   {
     "access_token": "xxx",
     "refresh_token": "xxx",
     "expires_in": 3600
   }

4. 后续请求携带 Token
   Authorization: Bearer <access_token>

5. Token 过期处理
   - 使用 Refresh Token 刷新
   - 获取新 Access Token
   - 多次刷新失败后重新登录
```

### 8.2 权限校验 (Cabin)

系统采用 Cabin 开源 RBAC 权限库进行权限管理。

```python
# app/core/security.py
from cabin.auth import need_permissions, get_current_user_from_token
from cabin.models import User

# Cabin 权限校验装饰器
def require_permission(*permissions: str):
    """
    Cabin 权限校验装饰器

    使用示例:
        @router.get("/tenants/{id}")
        @need_permissions("TENANT_VIEW")
        async def get_tenant(request: Request, id: int):
            ...
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Cabin 自动从请求中解析 Token 并校验权限
            return await func(*args, **kwargs)
        return wrapper
    return decorator


# 获取当前用户（Cabin 实现）
async def get_current_user(token: str = Depends(lambda: None)) -> User:
    """获取当前用户"""
    if not token:
        raise HTTPException(status_code=401, detail="未登录")

    # Cabin 解析 Token
    user = await get_current_user_from_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Token 无效")

    return user


# Cabin 内置权限校验
from cabin.auth import cabin_check_permission

# 使用示例
@router.get("/tenants/{id}")
@cabin_check_permission("TENANT_VIEW")
async def get_tenant(request: Request, id: int):
    ...
```

## 9. 高可用架构（待扩展）

### 9.1 SaaS 模式高可用

```
                    ┌─────────────────┐
                    │   Load Balancer  │
                    │    (Nginx)       │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   API Server  │   │   API Server  │   │   API Server  │
│    (Node 1)   │   │    (Node 2)   │   │    (Node 3)   │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  PostgreSQL   │   │  PostgreSQL   │   │    Redis      │
│  (Primary)    │◄──│  (Standby)    │   │   (Master)    │
│               │   │               │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 9.2 独立部署高可用

```
                    ┌─────────────────┐
                    │   客户负载均衡   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  FastAPI      │   │  FastAPI      │   │    Celery     │
│  Instance 1   │   │  Instance 2   │   │    Worker     │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 9.3 多地多中心部署架构

#### 9.3.1 部署模型

| 模型 | 说明 | 适用场景 |
|------|------|----------|
| 同城多活 | 同城多个数据中心，业务级切换 | 城市级容灾 |
| 两地三中心 | 主中心 + 同城备中心 + 异地灾备 | 金融级容灾 |
| 全球分布式 | 按地域部署多活节点 | 跨国企业、全球化业务 |

#### 9.3.2 两地三中心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         全局负载均衡 (GSLB)                        │
│                    (DNS + Geo-Routing)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   主中心 (A)   │   │   同城备中心 (B) │   │   异地灾备 (C) │
│  生产环境      │   │  热备          │   │  冷备/温备    │
│               │◄──│  同步复制      │   │  异步复制     │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        │   同步复制         │   异步复制        │
        └───────────────────┴───────────────────┘
                             │
                    ┌────────┴────────┐
                    │   数据层         │
                    │ PostgreSQL RAC   │
                    │ + Redis Cluster  │
                    └─────────────────┘
```

#### 9.3.3 多地部署数据流向

```
地域 A (华北)                      地域 B (华东)                      地域 C (华南)
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  Nginx          │              │  Nginx          │              │  Nginx          │
│  (就近接入)      │              │  (就近接入)      │              │  (就近接入)      │
└────────┬────────┘              └────────┬────────┘              └────────┬────────┘
         │                                │                                │
         ▼                                ▼                                ▼
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  FastAPI        │              │  FastAPI        │              │  FastAPI        │
│  (本地部署)      │◄───────────►│  (本地部署)      │◄───────────►│  (本地部署)      │
│                 │   数据同步     │                 │   数据同步     │                 │
└────────┬────────┘              └────────┬────────┘              └────────┬────────┘
         │                                │                                │
         ▼                                ▼                                ▼
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  PostgreSQL     │◄───────────►│  PostgreSQL     │◄───────────►│  PostgreSQL     │
│  (本地读副本)    │   异步复制   │  (本地读副本)    │   异步复制   │  (本地读副本)    │
└─────────────────┘              └─────────────────┘              └─────────────────┘
         │                                │                                │
         ▼                                ▼                                ▼
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  Redis          │              │  Redis          │              │  Redis          │
│  (本地缓存)      │              │  (本地缓存)      │              │  (本地缓存)      │
└─────────────────┘              └─────────────────┘              └─────────────────┘
```

#### 9.3.4 跨地域数据同步策略

| 数据类型 | 同步方式 | 延迟容忍 | 冲突处理 |
|----------|----------|----------|----------|
| 租户配置 | 同步复制 | < 1s | 主中心优先 |
| 审核数据 | 异步复制 | < 60s | 最后写入优先 |
| 任务分配 | 消息队列 | < 5s | 分布式锁 |
| 文件存储 | 跨区域复制 | < 300s | 不可变对象 |
| 缓存数据 | Redis Replication | < 1s | 缓存失效 |

#### 9.3.5 多中心路由策略

```python
# app/core/routing.py
class MultiRegionRouter:
    """多地域路由"""

    # 地域配置
    REGIONS = {
        'cn-north': {'name': '华北', 'priority': 1},
        'cn-east': {'name': '华东', 'priority': 2},
        'cn-south': {'name': '华南', 'priority': 3},
    }

    def __init__(self):
        self.current_region = self._detect_region()

    def _detect_region(self) -> str:
        """根据请求来源 IP 判定地域"""
        # 使用 X-Forwarded-For 或 IP 段匹配
        pass

    def get_primary_endpoint(self) -> str:
        """获取主中心端点"""
        return f"https://api-{self.REGIONS['cn-north']['name']}.smartaudit.com"

    def get_backup_endpoints(self) -> List[str]:
        """获取备用端点列表"""
        return [
            f"https://api-{r['name']}.smartaudit.com"
            for r in self.REGIONS.values()
        ]

    async def route_request(
        self,
        request: Request,
        fallback_enabled: bool = True
    ) -> Response:
        """路由请求，支持故障转移"""
        region = self._detect_region()

        # 优先访问本地节点
        try:
            response = await self.call_local(region, request)
            return response
        except RegionUnavailableError:
            if fallback_enabled:
                # 故障转移到其他地域
                return await self.failover(request, exclude=region)
            raise
```

#### 9.3.6 全局一致性保证

```python
# app/core/distributed_lock.py
class DistributedLock:
    """分布式锁 - 保证跨地域操作一致性"""

    def __init__(self, redis_cluster):
        self.redis = redis_cluster

    async def acquire(
        self,
        lock_key: str,
        ttl: int = 30,
        region_id: str = None
    ) -> bool:
        """
        获取分布式锁

        - 使用 RedLock 算法跨 Redis 集群
        - TTL 防止死锁
        - region_id 用于锁竞争日志追踪
        """
        lock_value = f"{region_id}:{time.time()}"

        # 向主中心 Redis 获取锁
        if await self.redis.set(lock_key, lock_value, nx=True, ex=ttl):
            return True

        # 向备中心 Redis 获取锁（RedLock 降级）
        for region in self.backup_regions:
            if await region.redis.set(lock_key, lock_value, nx=True, ex=ttl):
                return True

        return False

    async def release(self, lock_key: str, expected_value: str):
        """释放锁 - 仅持有者可释放"""
        # Lua 脚本保证原子性
        lua = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        await self.redis.eval(lua, 1, lock_key, expected_value)
```

## 10. 非功能性需求

### 10.1 性能需求

| 指标 | 要求 |
|------|------|
| 页面响应时间 | ≤ 2秒（95%分位） |
| API 响应时间 | ≤ 500ms（95%分位） |
| 规则执行时间 | ≤ 1秒（单条数据） |
| 并发用户数 | 支持 500 人同时在线 |
| 数据吞吐量 | 支持 1000 条/分钟审核 |

### 10.2 可用性需求

| 指标 | 要求 |
|------|------|
| 系统可用性 | ≥ 99.5% |
| 故障恢复时间 | ≤ 30 分钟 |
| 数据备份 | 每日全量备份 |
| 日志保留 | 180 天 |

### 10.3 安全需求

| 需求 | 说明 |
|------|------|
| 密码加密 | SHA256 + salt |
| 敏感数据 | 身份证号等部分脱敏展示 |
| 传输加密 | HTTPS 强制 |
| CSRF 防护 | Token 验证 |
| SQL 注入防护 | 参数化查询 |
| XSS 防护 | 输入输出过滤 |

### 10.4 兼容性需求

| 类型 | 要求 |
|------|------|
| 浏览器 | Chrome/Firefox/Safari/Edge 最新两个版本 |
| 操作系统 | Windows 10+, macOS 10.14+, Debian 11+ |
| 移动端 | iOS 12+, Android 8+ |

### 10.5 可扩展性需求

| 需求 | 说明 |
|------|------|
| 水平扩展 | 支持多实例部署 |
| 租户扩展 | 支持无限租户 |
| 数据扩展 | 分库分表方案预留 |
| 接口扩展 | 插件化三方接口 |

#### 9.3.7 故障切换流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      故障检测与切换流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 健康检查                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - Nginx upstream_check (主动探测)                       │   │
│  │  - 每 5 秒检测一次                                       │   │
│  │  - 连续 3 次失败判定为不可用                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  2. 流量切换                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - GSLB 移除故障节点                                     │   │
│  │  - DNS TTL 降至 60s，加速切换                            │   │
│  │  - 备用中心启动热备服务                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  3. 数据补偿                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 故障期间数据通过消息队列 补偿                          │   │
│  │  - 启动数据同步任务追平                                  │   │
│  │  - 验证数据一致性                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  4. 故障恢复                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 故障节点修复后以备机身份重新加入                       │   │
│  │  - 数据追平后切换为主机                                   │   │
│  │  - 逐步引导流量回切                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 9.3.8 多中心部署配置清单

| 组件 | 配置项 | 说明 |
|------|--------|------|
| Nginx | upstream_check | 健康检查配置 |
| PostgreSQL | streaming replication | 主从流复制 |
| Redis | Redis Cluster / Sentinel | 主从 + 哨兵 |
| Celery | broker+result_backend | 多 Broker 备份 |
| S3/OSS | 跨区域复制 | 备份存储 |
| 消息队列 | 多节点集群 | Kafka/RabbitMQ 集群 |
