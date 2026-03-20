# API 设计规范

## 1. RESTful API 设计

### 1.1 URL 规范

#### 基本格式
```
https://api.example.com/v1/{resource}/{id}/{sub-resource}
```

#### 命名规则
- 资源名使用名词，复数形式
- 使用 kebab-case（短横线分隔）
- 避免动词，使用 HTTP 方法表达动作

| 操作 | 方法 | URL | 说明 |
|------|------|-----|------|
| 获取列表 | GET | `/tenants` | 获取租户列表 |
| 获取单个 | GET | `/tenants/{id}` | 获取指定租户 |
| 创建 | POST | `/tenants` | 创建租户 |
| 更新 | PUT | `/tenants/{id}` | 完整更新 |
| 部分更新 | PATCH | `/tenants/{id}` | 部分更新 |
| 删除 | DELETE | `/tenants/{id}` | 删除租户 |

#### 嵌套资源
```
/tenants/{tenant_id}/projects          # 租户下的项目列表
/tenants/{tenant_id}/projects/{id}    # 租户下的指定项目
/projects/{project_id}/rules          # 项目下的规则列表
/tasks/{task_id}/audit                # 任务的审核结果
```

#### 动作命名（特殊情况）
```
POST /tenants/{id}/activate            # 启用
POST /tenants/{id}/disable            # 禁用
POST /tenants/{id}/reset-secret       # 重置密钥
POST /projects/{id}/archive           # 归档项目
```

### 1.2 HTTP 状态码

| 状态码 | 说明 | 使用场景 |
|--------|------|----------|
| 200 | OK | 成功获取/更新数据 |
| 201 | Created | 成功创建资源 |
| 204 | No Content | 成功删除，无返回内容 |
| 400 | Bad Request | 请求参数错误 |
| 401 | Unauthorized | 未登录或 token 过期 |
| 403 | Forbidden | 无权限访问 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突，如重复创建 |
| 422 | Unprocessable Entity | 验证错误 |
| 429 | Too Many Requests | 请求过于频繁 |
| 500 | Internal Server Error | 服务器内部错误 |
| 503 | Service Unavailable | 服务不可用 |

### 1.3 请求格式

#### 请求头
```
Content-Type: application/json
Authorization: Bearer <token>
X-Request-ID: <uuid>           # 请求追踪 ID
X-Tenant-ID: <tenant_id>       # 多租户标识
```

#### Query 参数
```
GET /tenants?page=1&page_size=20&status=active&keyword=test
```

#### 请求体（JSON）
```json
{
  "name": "测试租户",
  "code": "test_tenant",
  "contact": "张三",
  "phone": "13800138000",
  "email": "zhangsan@example.com"
}
```

## 2. 响应格式

### 2.1 成功响应

#### 标准成功响应
```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

#### 分页响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "id": 1,
        "name": "租户A",
        "code": "tenant_a",
        "status": "active",
        "created_at": "2024-01-01T00:00:00Z"
      }
    ],
    "total": 100,
    "page": 1,
    "page_size": 20,
    "pages": 5
  }
}
```

#### 列表响应（无分页）
```json
{
  "code": 0,
  "message": "success",
  "data": [
    { "id": 1, "name": "项目A" },
    { "id": 2, "name": "项目B" }
  ]
}
```

### 2.2 错误响应

#### 标准错误响应
```json
{
  "code": 40001,
  "message": "请求参数错误",
  "data": null,
  "errors": [
    {
      "field": "name",
      "message": "名称不能为空"
    },
    {
      "field": "code",
      "message": "代码已存在"
    }
  ]
}
```

#### 错误码规范
| 错误码范围 | 模块 | 说明 |
|------------|------|------|
| 0 | 成功 | 0 |
| 1xxxx | 通用错误 | 10xxx |
| 2xxxx | 认证授权 | 20xxx |
| 3xxxx | 租户管理 | 30xxx |
| 4xxxx | 项目管理 | 40xxx |
| 5xxxx | 规则引擎 | 50xxx |
| 6xxxx | 任务审核 | 60xxx |
| 7xxxx | 账户权限 | 70xxx |
| 8xxxx | 统计报表 | 80xxx |
| 9xxxx | 系统配置 | 90xxx |

## 3. API 版本控制

### 3.1 版本策略
- URL 路径方式（推荐）：`/api/v1/xxx`
- Header 方式：`Accept: application/vnd.api.v1+json`

### 3.2 版本升级规则
- 向下兼容：新增字段、新增接口
- 不兼容变更：升级大版本
- 废弃接口：返回 `Deprecation` Header

```
GET /api/v1/tenants

Response Headers:
X-API-WARNING: This endpoint will be deprecated in v3. Please use /api/v2/tenants
```

## 4. 认证授权

### 4.1 JWT Token

#### 登录请求
```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "password123"
}
```

#### 登录响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user_info": {
      "id": 1,
      "username": "admin",
      "name": "管理员",
      "tenant_id": 1,
      "roles": ["TENANT_SUPER_ADMIN"]
    }
  }
}
```

#### Token 刷新
```http
POST /api/v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### 4.2 权限校验
```http
GET /api/v1/tenants
Authorization: Bearer <access_token>
X-Permission: TENANT_VIEW

Response:
- 200: 有权限，返回数据
- 403: 无权限
```

## 5. 分页规范

### 5.1 分页参数
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| page | int | 1 | 页码 |
| page_size | int | 20 | 每页数量（最大100） |
| sort | string | created_at | 排序字段 |
| order | string | desc | 排序方向（asc/desc） |

### 5.2 分页响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [...],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100,
      "pages": 5,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

## 6. 过滤与搜索

### 6.1 过滤参数
```
GET /api/v1/tenants?status=active&created_at__gte=2024-01-01
```

#### 支持的操作符
| 操作符 | 说明 | 示例 |
|--------|------|------|
| `__eq` | 等于（默认） | `status=active` |
| `__ne` | 不等于 | `status__ne=disabled` |
| `__gt` | 大于 | `created_at__gt=2024-01-01` |
| `__gte` | 大于等于 | `created_at__gte=2024-01-01` |
| `__lt` | 小于 | `created_at__lt=2024-01-02` |
| `__lte` | 小于等于 | `created_at__lte=2024-01-31` |
| `__in` | 包含 | `status__in=active,pending` |
| `__contains` | 模糊包含 | `name__contains=test` |
| `__startswith` | 开头匹配 | `code__startswith=TEST` |
| `__endswith` | 结尾匹配 | `email__endswith=@example.com` |

### 6.2 搜索参数
```
GET /api/v1/tenants?keyword=教育&fields=name,code
```

## 7. 排序规范

### 7.1 排序参数
```
GET /api/v1/tenants?sort=created_at&order=desc
GET /api/v1/tenants?sort=created_at,-updated_at  # 多字段排序
```

### 7.2 可排序字段
- `created_at`: 创建时间
- `updated_at`: 更新时间
- `name`: 名称
- `status`: 状态

## 8. 字段规范

### 8.1 响应字段命名
- 使用 snake_case
- 时间格式：ISO 8601
- 金额单位：分（内部）或元（展示）

### 8.2 时间字段
```json
{
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "deleted_at": null,
  "expired_at": "2024-12-31T23:59:59Z"
}
```

## 9. 批量操作

### 9.1 批量创建
```http
POST /api/v1/projects/batch
Content-Type: application/json

{
  "items": [
    { "name": "项目A", "code": "proj_a" },
    { "name": "项目B", "code": "proj_b" },
    { "name": "项目C", "code": "proj_c" }
  ]
}
```

#### 响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "success_count": 3,
    "failed_count": 0,
    "items": [
      { "id": 1, "name": "项目A", "code": "proj_a" },
      { "id": 2, "name": "项目B", "code": "proj_b" },
      { "id": 3, "name": "项目C", "code": "proj_c" }
    ]
  }
}
```

### 9.2 批量更新
```http
PUT /api/v1/projects/batch
Content-Type: application/json

{
  "items": [
    { "id": 1, "status": "disabled" },
    { "id": 2, "status": "active" }
  ]
}
```

### 9.3 批量删除
```http
DELETE /api/v1/projects/batch
Content-Type: application/json

{
  "ids": [1, 2, 3, 4, 5]
}
```

## 10. 文件上传

### 10.1 单文件上传
```http
POST /api/v1/upload
Content-Type: multipart/form-data

file: <binary>
```

#### 响应
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "file_id": "uuid-xxx",
    "file_name": "document.pdf",
    "file_url": "https://cdn.example.com/uploads/uuid-xxx.pdf",
    "file_size": 1024000,
    "mime_type": "application/pdf"
  }
}
```

### 10.2 多文件上传
```http
POST /api/v1/upload/batch
Content-Type: multipart/form-data

files: [<binary>, <binary>]
```

## 11. 异步任务

### 11.1 异步提交
```http
POST /api/v1/exports
Content-Type: application/json

{
  "type": "audit_report",
  "params": {
    "project_id": 1,
    "start_date": "2024-01-01",
    "end_date": "2024-01-31"
  }
}
```

#### 响应
```json
{
  "code": 0,
  "message": "任务已提交",
  "data": {
    "task_id": "uuid-xxx",
    "status": "pending"
  }
}
```

### 11.2 任务状态查询
```http
GET /api/v1/exports/{task_id}

Response:
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": "uuid-xxx",
    "status": "completed",  // pending, processing, completed, failed
    "progress": 100,
    "result_url": "https://cdn.example.com/exports/uuid-xxx.xlsx",
    "created_at": "2024-01-15T10:00:00Z",
    "completed_at": "2024-01-15T10:05:00Z"
  }
}
```

## 12. API 文档

### 12.1 OpenAPI/Swagger
```yaml
openapi: 3.0.0
info:
  title: SmartAudit API
  version: 1.0.0
  description: 智核智能审核系统 API 文档

servers:
  - url: https://api.smartaudit.com/v1
    description: 生产环境
  - url: https://api-test.smartaudit.com/v1
    description: 测试环境

paths:
  /tenants:
    get:
      summary: 获取租户列表
      tags:
        - 租户管理
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: page_size
          in: query
          schema:
            type: integer
            default: 20
        - name: keyword
          in: query
          schema:
            type: string
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TenantListResponse'
```

### 12.2 API 实现示例（FastAPI）

```python
# app/api/v1/tenant.py
from fastapi import APIRouter, Depends, Query
from typing import Optional
from app.schemas.tenant import TenantResponse, TenantListResponse
from app.services.tenant_service import TenantService
from app.api.v1.deps import get_current_user, PageParams

router = APIRouter(prefix="/tenants", tags=["租户管理"])


@router.get("", response_model=TenantListResponse)
async def list_tenants(
    page_params: PageParams = Depends(),
    keyword: Optional[str] = Query(None, description="搜索关键词"),
    status: Optional[str] = Query(None, description="租户状态"),
    tenant_service: TenantService = Depends(get_tenant_service),
    current_user = Depends(get_current_user),
):
    """获取租户列表"""
    result = await tenant_service.list(
        page=page_params.page,
        page_size=page_params.page_size,
        keyword=keyword,
        status=status,
    )
    return result


@router.get("/{tenant_id}", response_model=TenantResponse)
async def get_tenant(
    tenant_id: int,
    tenant_service: TenantService = Depends(get_tenant_service),
    current_user = Depends(get_current_user),
):
    """获取租户详情"""
    return await tenant_service.get_by_id(tenant_id)


@router.post("", response_model=TenantResponse, status_code=201)
async def create_tenant(
    data: TenantCreate,
    tenant_service: TenantService = Depends(get_tenant_service),
    current_user = Depends(get_current_user),
):
    """创建租户"""
    return await tenant_service.create(data)


@router.put("/{tenant_id}", response_model=TenantResponse)
async def update_tenant(
    tenant_id: int,
    data: TenantUpdate,
    tenant_service: TenantService = Depends(get_tenant_service),
    current_user = Depends(get_current_user),
):
    """更新租户"""
    return await tenant_service.update(tenant_id, data)


@router.delete("/{tenant_id}", status_code=204)
async def delete_tenant(
    tenant_id: int,
    tenant_service: TenantService = Depends(get_tenant_service),
    current_user = Depends(get_current_user),
):
    """删除租户（软删除）"""
    await tenant_service.soft_delete(tenant_id)
```
