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

### 8.2 权限校验

```python
# app/core/security.py
def check_permission(required_permission: str):
    """权限校验装饰器"""
    async def wrapper(
        request: Request,
        current_user = Depends(get_current_user),
    ):
        user_permissions = await get_user_permissions(
            current_user.tenant_id,
            current_user.id
        )

        if required_permission not in user_permissions:
            raise HTTPException(
                status_code=403,
                detail="无权限访问"
            )

        return await call_next(request)
    return wrapper

# 使用示例
@router.get("/tenants/{id}")
@check_permission("TENANT_VIEW")
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
