# 租户管理系统设计

## 1. 系统概述

租户管理系统是各租户企业使用的系统，负责管理项目、规则、审核任务、统计数据等。

**使用对象**：租户管理员、审核员、普通员工

## 2. 技术架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   PC Web    │  │  Mobile H5  │  │   小程序     │             │
│  │  React      │  │  React      │  │  React      │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx (反向代理)                              │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FastAPI 服务集群                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Tenant API  │  │  Tenant API │  │  Celery      │             │
│  │   Instance   │  │   Instance  │  │  Workers     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   PostgreSQL    │  │     Redis       │  │      S3/OSS     │
│   (独立数据库)   │  │  (会话/缓存/MQ) │  │   (文件存储)    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 2.2 技术栈

| 层级 | 技术选型 | 版本 |
|------|----------|------|
| 前端框架 | React | 18.x |
| 前端构建 | Vite | 5.x |
| UI 框架 | Tailwind CSS | 3.x |
| 状态管理 | Zustand + TanStack Query | 4.x / 5.x |
| 后端框架 | FastAPI | 0.109+ |
| Python 版本 | Python | 3.11+ |
| ORM | SQLAlchemy | 2.x |
| 任务队列 | Celery + Redis | 5.x / 7.x |
| 数据库 | PostgreSQL | 15+ |
| 缓存 | Redis | 7+ |
| 文件存储 | S3 / OSS / MinIO | - |

### 2.3 目录结构

```
tenant-backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 应用入口
│   │
│   ├── api/                    # API 路由
│   │   ├── deps.py             # 依赖注入
│   │   └── v1/
│   │       ├── router.py       # 路由汇总
│   │       ├── auth.py         # 认证接口
│   │       ├── account.py       # 账户管理
│   │       ├── role.py          # 角色管理
│   │       ├── project.py       # 项目管理
│   │       ├── rule.py          # 规则管理
│   │       ├── audit.py         # 审核管理
│   │       ├── task.py          # 任务管理
│   │       ├── third_party.py   # 三方接口
│   │       ├── statistics.py    # 统计接口
│   │       ├── backup.py        # 备份管理
│   │       └── version.py       # 版本管理
│   │
│   ├── core/                   # 核心配置
│   │   ├── config.py
│   │   ├── security.py
│   │   ├── database.py
│   │   └── redis.py
│   │
│   ├── models/                 # SQLAlchemy 模型
│   │   ├── user.py
│   │   ├── role.py
│   │   ├── permission.py
│   │   ├── project.py
│   │   ├── rule.py
│   │   ├── audit_data.py
│   │   ├── task.py
│   │   ├── third_party.py
│   │   ├── backup.py
│   │   └── login_log.py
│   │
│   ├── schemas/                # Pydantic 模型
│   │   └── ...
│   │
│   ├── services/                # 业务逻辑
│   │   ├── auth_service.py
│   │   ├── account_service.py
│   │   ├── project_service.py
│   │   ├── rule_service.py
│   │   ├── audit_service.py
│   │   ├── task_service.py
│   │   ├── third_party_service.py
│   │   ├── backup_service.py
│   │   └── statistics_service.py
│   │
│   ├── tasks/                   # Celery 任务
│   │   ├── celery_app.py
│   │   ├── audit_tasks.py
│   │   ├── backup_tasks.py
│   │   └── notification_tasks.py
│   │
│   └── utils/                   # 工具函数
│       ├── rule_engine.py      # 规则引擎
│       └── pagination.py
│
├── migrations/
├── tests/
├── pyproject.toml
└── alembic.ini
```

## 3. 核心功能

### 3.1 账户权限系统

#### 用户模型

```python
class User(Base):
    """租户用户"""
    __tablename__ = "user"

    id: UUID
    tenant_id: UUID              # 租户ID
    username: str                # 用户名，唯一
    password_hash: str           # 密码哈希
    name: str                    # 姓名
    email: str                   # 邮箱
    email_verified: bool         # 邮箱已验证
    phone: str                   # 手机号
    phone_verified: bool         # 手机已验证
    avatar_url: str              # 头像
    status: str                  # active/inactive
    extra_permissions: JSON      # 额外权限
    last_login_at: datetime
    last_login_ip: str
    created_at: datetime
    updated_at: datetime

    # 关联
    roles: List[UserRole]        # 用户角色
```

#### 角色模型

```python
class Role(Base):
    """角色"""
    __tablename__ = "role"

    id: UUID
    tenant_id: UUID
    name: str                    # 角色名称
    code: str                    # 角色代码
    description: str              # 描述
    is_system: bool              # 是否系统内置
    sort_order: int              # 排序
    created_at: datetime
    updated_at: datetime

    # 关联
    permissions: List[RolePermission]
    users: List[UserRole]


class Permission(Base):
    """权限"""
    __tablename__ = "permission"

    id: UUID
    code: str                    # 权限代码
    name: str                    # 权限名称
    module: str                   # 所属模块
    description: str
    created_at: datetime
```

### 3.2 项目管理系统

```python
class Project(Base):
    """审核项目"""
    __tablename__ = "project"

    id: UUID
    tenant_id: UUID
    name: str                    # 项目名称
    code: str                    # 项目代码
    description: str              # 项目描述
    fields_def: JSON             # 入参数据项定义
    rule_ids: JSON               # 关联的规则ID列表
    need_manual_review: bool     # 需要人工校验
    task_assign_type: str        # 任务分配方式
    status: str                  # active/inactive/archived
    created_by: UUID
    created_at: datetime
    updated_at: datetime

    # 关联
    audit_data: List[AuditData]
```

### 3.3 规则引擎系统

```python
class Rule(Base):
    """审核规则"""
    __tablename__ = "rule"

    id: UUID
    tenant_id: UUID
    name: str                    # 规则名称
    description: str              # 规则描述
    conditions: JSON             # 条件组定义
    result_pass: str              # 通过结论
    result_reject: str           # 拒绝原因
    result_suggest: str          # 建议处理
    priority: int                # 优先级
    status: str                  # active/inactive
    created_by: UUID
    created_at: datetime
    updated_at: datetime
```

**条件结构示例**:

```json
{
  "logic": "ALL",
  "conditions": [
    {
      "field": "age",
      "operator": "greaterOrEqual",
      "value": 18
    },
    {
      "field": "education",
      "operator": "in",
      "value": ["本科", "硕士", "博士"]
    },
    {
      "logic": "ANY",
      "conditions": [
        {"field": "license_type", "operator": "equals", "value": "A"},
        {"field": "license_type", "operator": "equals", "value": "B"}
      ]
    }
  ]
}
```

### 3.4 任务审核系统

```python
class AuditData(Base):
    """审核数据"""
    __tablename__ = "audit_data"

    id: UUID
    tenant_id: UUID
    project_id: UUID
    input_data: JSON             # 原始入参数据
    rule_result: JSON            # 规则执行结果
    auto_result: str             # 自动审核结果
    final_result: str             # 最终审核结果
    auditor_id: UUID              # 审核员ID
    audit_time: datetime         # 审核时间
    audit_note: str              # 审核备注
    need_recheck: bool           # 需要二次校验
    recheck_auditor_id: UUID    # 二次审核员
    recheck_time: datetime
    recheck_result: str
    recheck_note: str
    status: str                  # pending/in_progress/passed/rejected/rechecking
    priority: int                # 优先级
    created_at: datetime
    updated_at: datetime


class Task(Base):
    """审核任务"""
    __tablename__ = "task"

    id: UUID
    tenant_id: UUID
    auditor_id: UUID
    data_id: UUID
    type: str                    # review/recheck
    status: str                  # pending/assigned/in_progress/completed/skipped
    result: str                  # pass/reject
    note: str
    assigned_at: datetime
    started_at: datetime
    completed_at: datetime
    created_at: datetime
    updated_at: datetime
```

### 3.5 三方接口系统

```python
class ThirdPartyConfig(Base):
    """第三方接口配置"""
    __tablename__ = "third_party_config"

    id: UUID
    tenant_id: UUID
    name: str                    # 接口名称
    type: str                    # 接口类型
    url: str                     # 请求地址
    method: str                  # GET/POST
    headers: JSON                # 请求头
    params_template: JSON        # 参数模板
    auth_type: str               # none/bearer/oauth2/api_key
    auth_config: JSON            # 认证配置
    timeout: int                 # 超时时间
    retry_count: int             # 重试次数
    status: str                  # active/inactive
    created_at: datetime
    updated_at: datetime


class ThirdPartyLog(Base):
    """第三方接口调用日志"""
    __tablename__ = "third_party_log"

    id: UUID
    tenant_id: UUID
    config_id: UUID
    user_id: UUID
    request_time: datetime
    request_url: str
    request_method: str
    request_data: JSON
    response_data: JSON
    response_status: str
    error_msg: str
    duration_ms: int
    created_at: datetime
```

### 3.6 备份管理系统

```python
class BackupRule(Base):
    """备份规则"""
    __tablename__ = "backup_rule"

    id: UUID
    tenant_id: UUID
    name: str                    # 规则名称
    backup_type: str             # full/differential/incremental
    schedule_cron: str           # Cron 表达式
    retention_days: int         # 保留天数
    storage_location: str        # local/s3/oss
    storage_config: JSON         # 存储配置
    is_enabled: bool
    last_backup_at: datetime
    next_backup_at: datetime
    created_by: UUID
    created_at: datetime
    updated_at: datetime


class BackupRecord(Base):
    """备份记录"""
    __tablename__ = "backup_record"

    id: UUID
    tenant_id: UUID
    rule_id: UUID
    backup_type: str
    status: str                  # pending/running/success/failed
    started_at: datetime
    completed_at: datetime
    file_path: str
    file_size: int
    checksum: str
    error_msg: str
    created_at: datetime
```

## 4. API 接口

### 4.1 认证接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/v1/auth/login | 登录 |
| POST | /api/v1/auth/logout | 登出 |
| GET | /api/v1/auth/me | 当前用户信息 |
| PUT | /api/v1/auth/password | 修改密码 |

### 4.2 账户管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/accounts | 账户列表 |
| POST | /api/v1/accounts | 创建账户 |
| GET | /api/v1/accounts/{id} | 账户详情 |
| PUT | /api/v1/accounts/{id} | 编辑账户 |
| DELETE | /api/v1/accounts/{id} | 删除账户 |
| PUT | /api/v1/accounts/{id}/roles | 分配角色 |
| PUT | /api/v1/accounts/{id}/permissions | 配置权限 |
| POST | /api/v1/accounts/{id}/reset-password | 重置密码 |
| PUT | /api/v1/accounts/profile | 修改个人资料 |
| GET | /api/v1/accounts/my-permissions | 获取我的权限 |

### 4.3 角色管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/roles | 角色列表 |
| POST | /api/v1/roles | 创建角色 |
| GET | /api/v1/roles/{id} | 角色详情 |
| PUT | /api/v1/roles/{id} | 编辑角色 |
| DELETE | /api/v1/roles/{id} | 删除角色 |
| PUT | /api/v1/roles/{id}/permissions | 配置角色权限 |

### 4.4 项目管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/projects | 项目列表 |
| POST | /api/v1/projects | 创建项目 |
| GET | /api/v1/projects/{id} | 项目详情 |
| PUT | /api/v1/projects/{id} | 编辑项目 |
| DELETE | /api/v1/projects/{id} | 删除项目 |
| GET | /api/v1/projects/{id}/statistics | 项目统计 |

### 4.5 规则管理接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/rules | 规则列表 |
| POST | /api/v1/rules | 创建规则 |
| GET | /api/v1/rules/{id} | 规则详情 |
| PUT | /api/v1/rules/{id} | 编辑规则 |
| DELETE | /api/v1/rules/{id} | 删除规则 |
| POST | /api/v1/rules/test | 测试规则 |

### 4.6 审核接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/audit/pending | 待审核列表 |
| GET | /api/v1/audit/completed | 已审核列表 |
| POST | /api/v1/audit/submit | 提交数据 |
| POST | /api/v1/audit/{id}/execute | 执行审核 |
| GET | /api/v1/audit/{id} | 审核详情 |
| POST | /api/v1/audit/{id}/mark-recheck | 标记二次校验 |

### 4.7 任务接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/tasks/my | 我的任务 |
| GET | /api/v1/tasks | 任务列表 |
| POST | /api/v1/tasks/{id}/start | 开始任务 |
| POST | /api/v1/tasks/{id}/complete | 完成审核 |
| POST | /api/v1/tasks/{id}/skip | 跳过任务 |

### 4.8 三方接口配置

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/third-party/configs | 配置列表 |
| POST | /api/v1/third-party/configs | 创建配置 |
| PUT | /api/v1/third-party/configs/{id} | 编辑配置 |
| DELETE | /api/v1/third-party/configs/{id} | 删除配置 |
| POST | /api/v1/third-party/configs/{id}/test | 测试连接 |
| GET | /api/v1/third-party/logs | 调用日志 |
| GET | /api/v1/third-party/statistics | 调用统计 |

### 4.9 统计接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/statistics/overview | 运营概览 |
| GET | /api/v1/statistics/project/{id} | 项目统计 |
| GET | /api/v1/statistics/auditor | 审核员统计 |
| GET | /api/v1/statistics/trend | 趋势统计 |

### 4.10 备份接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/backup/rules | 备份规则列表 |
| POST | /api/v1/backup/rules | 创建规则 |
| PUT | /api/v1/backup/rules/{id} | 编辑规则 |
| DELETE | /api/v1/backup/rules/{id} | 删除规则 |
| GET | /api/v1/backup/records | 备份记录列表 |
| POST | /api/v1/backup/execute | 手动执行备份 |
| GET | /api/v1/backup/records/{id} | 备份详情 |
| POST | /api/v1/backup/records/{id}/restore | 执行恢复 |

### 4.11 版本接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/version/current | 当前版本 |
| GET | /api/v1/version/check | 检查更新 |
| POST | /api/v1/version/upgrade | 执行升级 |
| GET | /api/v1/version/records | 升级记录 |

## 5. Celery 任务

### 5.1 任务队列

| 队列 | 说明 | 优先级 |
|------|------|--------|
| audit | 审核任务 | 8 |
| backup | 备份任务 | 3 |
| notify | 通知任务 | 5 |
| cleanup | 清理任务 | 1 |

### 5.2 任务定义

```python
# 审核任务
@shared_task(name='audit.execute_rule', queue='audit')
def execute_audit_rule(data_id: str, rule_ids: list):
    """执行审核规则"""
    pass

@shared_task(name='audit.assign_task', queue='audit')
def assign_audit_task(data_id: str, project_id: str):
    """分配审核任务"""
    pass

# 备份任务
@shared_task(name='backup.execute', queue='backup')
def execute_backup(backup_record_id: str):
    """执行备份"""
    pass

@shared_task(name='backup.cleanup', queue='cleanup')
def cleanup_old_backups(tenant_id: str, retention_days: int):
    """清理过期备份"""
    pass

# 通知任务
@shared_task(name='notify.auditor', queue='notify')
def notify_auditor(auditor_id: str, task_count: int):
    """通知审核员"""
    pass
```
