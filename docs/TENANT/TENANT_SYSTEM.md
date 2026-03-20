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
    source: str                  # manual/api/import 数据来源
    rule_result: JSON            # 规则执行结果
    auto_result: str             # 自动审核结果
    final_result: str             # 最终审核结果
    audit_mode: str              # auto/manual 审核模式
    auditor_id: UUID              # 审核员ID
    audit_time: datetime         # 审核时间
    audit_note: str              # 审核备注
    need_recheck: bool           # 需要二次校验
    recheck_auditor_id: UUID    # 二次审核员
    recheck_time: datetime
    recheck_result: str
    recheck_note: str
    manual_intervention: bool     # 是否有人工介入
    manual_operator_id: UUID      # 人工操作人ID
    manual_operator_name: str
    manual_action: str           # modify/recheck/override
    manual_action_time: datetime
    status: str                  # pending/in_progress/passed/rejected/rechecking
    priority: int                # 优先级
    created_at: datetime
    updated_at: datetime


class AuditOperationLog(Base):
    """审核操作日志"""
    __tablename__ = "audit_operation_log"

    id: UUID
    tenant_id: UUID
    audit_data_id: UUID          # 关联的审核数据ID
    action: str                   # submit/auto_pass/auto_reject/assign_auditor/
                                  # audit_pass/audit_reject/recheck/
                                  # manual_modify/manual_override/recheck_override
    operator_type: str            # system/auditor/manual
    operator_id: UUID
    operator_name: str
    before_state: str             # 操作前状态
    after_state: str              # 操作后状态
    detail: str                   # 操作详情
    request_data: JSON           # 请求数据（人工介入时）
    created_at: datetime


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
| GET | /api/v1/audit/{id}/logs | 审核操作日志 |
| PUT | /api/v1/audit/{id}/manual-modify | 人工修改数据 |
| POST | /api/v1/audit/{id}/recheck | 二次复核 |
| POST | /api/v1/audit/{id}/override | 强制终态 |

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

### 5.3 审核工作流服务

```python
# app/services/audit_service.py
class AuditWorkflowService:
    """审核工作流服务"""

    async def submit_audit(
        self,
        tenant_id: str,
        project_id: str,
        input_data: dict,
        source: str = 'manual'
    ) -> AuditData:
        """
        提交审核数据

        工作流：
        1. 校验项目和规则配置
        2. 执行规则引擎
        3. 根据项目配置决定审核模式
        4. 自动审核或分配审核员
        """
        # 获取项目配置
        project = await self.project_service.get(project_id)

        # 执行规则引擎
        rule_result = await self.rule_engine.execute(
            rules=project.rule_ids,
            data=input_data
        )

        # 创建审核数据
        audit_data = AuditData(
            tenant_id=tenant_id,
            project_id=project_id,
            input_data=input_data,
            source=source,
            rule_result=rule_result,
            auto_result=rule_result.passed,
            audit_mode='auto' if not project.audit_config.need_manual_review else 'manual',
            status='pending',
            created_at=datetime.utcnow()
        )

        # 记录操作日志
        await self.log_operation(
            audit_data_id=audit_data.id,
            action='submit',
            operator_type='manual' if source == 'manual' else 'system',
            before_state=None,
            after_state='pending'
        )

        # 根据审核模式处理
        if not project.audit_config.need_manual_review:
            # 自动审核模式
            if rule_result.passed:
                audit_data.final_result = 'pass'
                audit_data.status = 'passed'
                await self.log_operation(
                    audit_data_id=audit_data.id,
                    action='auto_pass',
                    operator_type='system',
                    before_state='pending',
                    after_state='passed'
                )
            else:
                audit_data.final_result = 'reject'
                audit_data.status = 'rejected'
                await self.log_operation(
                    audit_data_id=audit_data.id,
                    action='auto_reject',
                    operator_type='system',
                    before_state='pending',
                    after_state='rejected'
                )
        else:
            # 人工审核模式：需要分配审核员
            auditor = await self.task_service.assign_auditor(
                project_id=project_id,
                data_id=audit_data.id
            )
            audit_data.auditor_id = auditor.id
            audit_data.status = 'in_progress'
            await self.log_operation(
                audit_data_id=audit_data.id,
                action='assign_auditor',
                operator_type='system',
                before_state='pending',
                after_state='in_progress'
            )

        await self.audit_repo.save(audit_data)
        return audit_data


    async def execute_audit(
        self,
        audit_data_id: str,
        auditor_id: str,
        result: str,
        note: str = None
    ) -> AuditData:
        """审核员执行审核"""
        audit_data = await self.audit_repo.get(audit_data_id)

        old_status = audit_data.status
        audit_data.final_result = result
        audit_data.auditor_id = auditor_id
        audit_data.audit_time = datetime.utcnow()
        audit_data.audit_note = note
        audit_data.status = 'passed' if result == 'pass' else 'rejected'

        await self.log_operation(
            audit_data_id=audit_data.id,
            action='audit_pass' if result == 'pass' else 'audit_reject',
            operator_type='auditor',
            operator_id=auditor_id,
            before_state=old_status,
            after_state=audit_data.status
        )

        await self.audit_repo.save(audit_data)
        return audit_data


    async def recheck(
        self,
        audit_data_id: str,
        auditor_id: str,
        result: str,
        note: str = None,
        force_override: bool = False
    ) -> AuditData:
        """
        二次复核

        对已审核数据进行再次审核，可覆盖原结果
        """
        audit_data = await self.audit_repo.get(audit_data_id)

        old_result = audit_data.final_result
        old_status = audit_data.status

        audit_data.need_recheck = False
        audit_data.recheck_auditor_id = auditor_id
        audit_data.recheck_time = datetime.utcnow()
        audit_data.recheck_result = result
        audit_data.recheck_note = note

        if force_override:
            audit_data.final_result = result
            audit_data.status = 'passed' if result == 'pass' else 'rejected'
            action = 'recheck_override'
        else:
            action = 'recheck'

        await self.log_operation(
            audit_data_id=audit_data.id,
            action=action,
            operator_type='auditor',
            operator_id=auditor_id,
            before_state=old_status,
            after_state=audit_data.status,
            detail=f"原结果: {old_result} → 新结果: {result}"
        )

        await self.audit_repo.save(audit_data)
        return audit_data


    async def manual_modify(
        self,
        audit_data_id: str,
        operator_id: str,
        input_data: dict,
        reason: str
    ) -> AuditData:
        """
        人工修改数据（外部渠道数据修正）

        保留修改痕迹，记录操作日志
        """
        audit_data = await self.audit_repo.get(audit_data_id)

        old_input_data = audit_data.input_data
        audit_data.input_data = input_data
        audit_data.manual_intervention = True
        audit_data.manual_operator_id = operator_id
        audit_data.manual_action = 'modify'
        audit_data.manual_action_time = datetime.utcnow()

        # 重新执行规则引擎
        new_rule_result = await self.rule_engine.execute(
            rules=audit_data.project.rule_ids,
            data=input_data
        )
        audit_data.rule_result = new_rule_result

        await self.log_operation(
            audit_data_id=audit_data.id,
            action='manual_modify',
            operator_type='manual',
            operator_id=operator_id,
            before_state=old_input_data,
            after_state=input_data,
            detail=f"修改原因: {reason}"
        )

        await self.audit_repo.save(audit_data)
        return audit_data


    async def manual_override(
        self,
        audit_data_id: str,
        operator_id: str,
        final_result: str,
        reason: str
    ) -> AuditData:
        """
        人工强制终态

        特殊情况下人工干预，强制设置最终结果
        """
        audit_data = await self.audit_repo.get(audit_data_id)

        old_result = audit_data.final_result
        old_status = audit_data.status

        audit_data.final_result = final_result
        audit_data.status = 'passed' if final_result == 'pass' else 'rejected'
        audit_data.manual_intervention = True
        audit_data.manual_operator_id = operator_id
        audit_data.manual_action = 'override'
        audit_data.manual_action_time = datetime.utcnow()

        await self.log_operation(
            audit_data_id=audit_data.id,
            action='manual_override',
            operator_type='manual',
            operator_id=operator_id,
            before_state=old_status,
            after_state=audit_data.status,
            detail=f"强制终态原因: {reason}，原结果: {old_result} → 新结果: {final_result}"
        )

        await self.audit_repo.save(audit_data)
        return audit_data


    async def get_operation_logs(
        self,
        audit_data_id: str
    ) -> List[AuditOperationLog]:
        """获取审核操作日志"""
        return await self.operation_log_repo.list(
            filters={'audit_data_id': audit_data_id},
            order_by='created_at',
            desc=True
        )
```
