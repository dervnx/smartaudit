# Python 代码规范

## 1. 项目结构

```
smartaudit-backend/
├── src/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 入口
│   ├── config.py               # 配置管理
│   ├── api/                    # API 路由
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── deps.py         # 依赖注入
│   │   │   ├── tenant.py       # 租户管理
│   │   │   ├── project.py      # 项目管理
│   │   │   ├── rule.py         # 规则引擎
│   │   │   ├── task.py         # 任务审核
│   │   │   ├── account.py      # 账户权限
│   │   │   ├── statistics.py   # 统计报表
│   │   │   └── third_party.py  # 三方接口
│   ├── core/                   # 核心模块
│   │   ├── __init__.py
│   │   ├── security.py         # 安全认证
│   │   ├── exceptions.py       # 异常定义
│   │   └── pagination.py       # 分页工具
│   ├── models/                 # SQLAlchemy 模型
│   │   ├── __init__.py
│   │   ├── base.py             # 基础模型
│   │   ├── tenant.py           # 租户模型
│   │   ├── account.py          # 账户模型
│   │   ├── project.py          # 项目模型
│   │   ├── rule.py              # 规则模型
│   │   ├── task.py              # 任务模型
│   │   └── audit_log.py        # 审计日志模型
│   ├── schemas/                # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── account.py
│   │   ├── project.py
│   │   ├── rule.py
│   │   ├── task.py
│   │   └── common.py           # 公共 schema
│   ├── services/               # 业务逻辑层
│   │   ├── __init__.py
│   │   ├── tenant_service.py
│   │   ├── account_service.py
│   │   ├── project_service.py
│   │   ├── rule_service.py
│   │   ├── task_service.py
│   │   └── audit_service.py
│   ├── repositories/           # 数据访问层
│   │   ├── __init__.py
│   │   ├── base.py             # 基础仓储
│   │   ├── tenant_repo.py
│   │   └── account_repo.py
│   ├── utils/                  # 工具函数
│   │   ├── __init__.py
│   │   ├── encrypt.py          # 加密工具
│   │   ├── datetime.py         # 时间工具
│   │   └── validators.py       # 验证器
│   └── db/                     # 数据库配置
│       ├── __init__.py
│       ├── database.py         # 数据库连接
│       └── redis.py            # Redis 连接
├── tests/                      # 测试目录
│   ├── __init__.py
│   ├── conftest.py             # pytest 配置
│   ├── api/                    # API 测试
│   ├── services/              # 服务层测试
│   └── utils/                  # 工具测试
├── alembic/                    # 数据库迁移
│   ├── versions/
│   └── env.py
├── pyproject.toml
├── poetry.lock
└── README.md
```

## 2. 命名规范

### 2.1 文件命名
- 使用 snake_case：`tenant_service.py`、`audit_log_model.py`
- 测试文件：`test_tenant_service.py`、`tenant_service_test.py`
- 前后下划线保留：`__init__.py`

### 2.2 类命名
- 使用 PascalCase：`class TenantService`、`class BaseModel`
- 异常类：`class TenantNotFoundError(Exception)`
- 枚举类：`class TenantStatus(str, Enum)`

### 2.3 函数/方法命名
- 使用 snake_case：`def get_tenant_by_id()`、`def create_project()`
- 异步函数：`async def fetch_tenant_data()`
- 私有方法：`def _internal_method()`

### 2.4 变量命名
- 使用 snake_case：`tenant_id`、`created_at`
- 常量：`MAX_RETRY_COUNT`、`DEFAULT_PAGE_SIZE`
- 类实例：`tenant_service`、`project_repo`

### 2.5 数据库字段命名
- 全部使用 snake_case：`created_at`、`updated_at`、`is_deleted`
- 布尔字段使用 `is_`、`has_` 前缀：`is_active`、`has_permission`
- 外键字段：`tenant_id`、`project_id`

## 3. 代码风格

### 3.1 缩进与格式
- 使用 4 空格缩进
- 行长度限制：120 字符
- 使用 Black 进行代码格式化
- 使用 isort 整理 import 顺序

### 3.2 Import 规范
```python
# 标准库
import os
import json
from datetime import datetime
from typing import Optional, List, Dict, Any

# 第三方库
from fastapi import Depends, HTTPException
from sqlalchemy import select, func
from pydantic import BaseModel, Field

# 本地导入
from app.models.tenant import Tenant
from app.schemas.tenant import TenantCreate, TenantResponse
from app.services import tenant_service
```

### 3.3 函数文档
```python
def create_tenant(name: str, code: str) -> Tenant:
    """
    创建新租户

    Args:
        name: 租户名称
        code: 租户代码，唯一标识

    Returns:
        创建的租户对象

    Raises:
        TenantAlreadyExistsError: 如果租户代码已存在
    """
    pass
```

## 4. API 设计规范

### 4.1 路径命名
- 使用 kebab-case：`/tenant-info`、`/project-config`
- 嵌套资源：`/tenants/{tenant_id}/projects`
- 动作命名：`/tenants/{tenant_id}/activate`

### 4.2 HTTP 方法
| 方法 | 用途 | 示例 |
|------|------|------|
| GET | 获取资源 | GET /tenants |
| POST | 创建资源 | POST /tenants |
| PUT | 完整更新 | PUT /tenants/{id} |
| PATCH | 部分更新 | PATCH /tenants/{id} |
| DELETE | 删除资源 | DELETE /tenants/{id} |

### 4.3 响应格式
```python
# 成功响应
{
    "code": 0,
    "message": "success",
    "data": { ... }
}

# 分页响应
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [...],
        "total": 100,
        "page": 1,
        "page_size": 20,
        "pages": 5
    }
}

# 错误响应
{
    "code": 40401,
    "message": "租户不存在",
    "data": null
}
```

## 5. 数据库规范

### 5.1 表命名
- 使用 snake_case：`sys_tenant`、`sys_account`
- 复数形式：`sys_tenants`、`sys_accounts`

### 5.2 字段规范
- 主键：`id` (BIGINT, 自增)
- 软删除：`is_deleted` (TINYINT, 默认 0, 1 表示删除)
- 时间戳：`created_at` (DATETIME)、`updated_at` (DATETIME)
- 创建人/更新人：`created_by` (BIGINT)、`updated_by` (BIGINT)
- 租户ID：`tenant_id` (BIGINT)
- 乐观锁：`version` (INT, 默认 0)

### 5.3 索引规范
- 普通索引：`idx_tenant_id`
- 唯一索引：`uk_tenant_code`
- 联合索引：`idx_tenant_status`

## 6. 异常处理

### 6.1 异常类定义
```python
# app/core/exceptions.py
class SmartAuditException(Exception):
    """基础异常类"""
    def __init__(self, code: int, message: str):
        self.code = code
        self.message = message
        super().__init__(message)

class TenantNotFoundError(SmartAuditException):
    def __init__(self, tenant_id: int):
        super().__init__(code=40401, message=f"租户 {tenant_id} 不存在")

class UnauthorizedError(SmartAuditException):
    def __init__(self, message: str = "未授权访问"):
        super().__init__(code=40101, message=message)
```

### 6.2 全局异常处理
```python
# app/api/v1/deps.py
from fastapi import HTTPException, status

@app.exception_handler(TenantNotFoundError)
async def tenant_not_found_handler(request, exc: TenantNotFoundError):
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail={"code": exc.code, "message": exc.message}
    )
```

## 7. 配置管理

### 7.1 环境变量
```python
# app/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # 应用配置
    app_name: str = "SmartAudit"
    debug: bool = False
    api_prefix: str = "/api/v1"

    # 数据库配置
    database_url: str

    # Redis 配置
    redis_url: str

    # JWT 配置
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 30

    class Config:
        env_file = ".env"
        case_sensitive = False

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

## 8. 测试规范

### 8.1 测试文件组织
```
tests/
├── conftest.py
├── api/
│   ├── __init__.py
│   ├── test_tenant.py
│   └── test_project.py
├── services/
│   ├── __init__.py
│   ├── test_tenant_service.py
│   └── test_project_service.py
└── fixtures/
    ├── __init__.py
    ├── tenant_fixtures.py
    └── project_fixtures.py
```

### 8.2 测试用例示例
```python
# tests/services/test_tenant_service.py
import pytest
from app.services.tenant_service import TenantService
from app.core.exceptions import TenantNotFoundError

class TestTenantService:
    @pytest.fixture
    def tenant_service(self, db_session):
        return TenantService(db_session)

    @pytest.fixture
    def sample_tenant(self, db_session):
        return create_tenant(db_session, name="测试租户", code="test")

    def test_create_tenant(self, tenant_service):
        tenant = tenant_service.create(
            name="新租户",
            code="new_tenant"
        )
        assert tenant.id is not None
        assert tenant.name == "新租户"

    def test_get_tenant_by_id(self, tenant_service, sample_tenant):
        result = tenant_service.get_by_id(sample_tenant.id)
        assert result.name == sample_tenant.name

    def test_get_nonexistent_tenant(self, tenant_service):
        with pytest.raises(TenantNotFoundError):
            tenant_service.get_by_id(99999)
```

## 9. 日志规范

### 9.1 日志级别使用
| 级别 | 使用场景 |
|------|----------|
| DEBUG | 调试信息，开发环境 |
| INFO | 正常业务流程 |
| WARNING | 警告信息，如重试 |
| ERROR | 错误信息，需关注 |
| CRITICAL | 严重错误，系统异常 |

### 9.2 日志格式
```python
import logging

logger = logging.getLogger(__name__)

# 正确示例
logger.info(f"创建租户成功: tenant_id={tenant_id}, code={code}")
logger.warning(f"第三方接口调用失败，正在重试: url={url}, retry=1/3")
logger.error(f"数据库连接失败: {error}", exc_info=True)

# 避免
logger.info("创建成功")  # 缺少上下文
logger.debug(f"用户 {user.id} 登录")  # debug 级别不打印
```

## 10. Git 提交规范

### 10.1 提交信息格式
```
<type>(<scope>): <subject>

<body>

<footer>
```

### 10.2 Type 类型
| Type | 说明 |
|------|------|
| feat | 新功能 |
| fix | Bug 修复 |
| docs | 文档更新 |
| style | 代码格式 |
| refactor | 重构 |
| test | 测试相关 |
| chore | 构建/工具 |

### 10.3 提交示例
```
feat(tenant): 添加租户创建接口

- 支持租户名称和代码的创建
- 添加租户唯一性校验
- 集成租户初始化服务

Closes #123
```
