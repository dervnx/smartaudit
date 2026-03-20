# 认证授权设计

## 1. 认证流程

### 1.1 登录流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         登录流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  用户                                                         │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 输入用户名、密码                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 前端调用 POST /api/v1/auth/login                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 后端验证密码                                           │   │
│  │    - 检查账户状态                                          │   │
│  │    - 验证密码哈希                                          │   │
│  │    - 记录登录日志                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. 生成 Token                                             │   │
│  │    - Access Token (JWT, 30分钟)                          │   │
│  │    - Refresh Token (UUID, 7天)                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. 返回 Token 和用户信息                                   │   │
│  │    - token, user, permissions                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 6. 前端存储 Token                                          │   │
│  │    - Access Token -> 内存                                │   │
│  │    - Refresh Token -> HttpOnly Cookie / 本地存储          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Token 刷新流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       Token 刷新流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Access Token 过期                                              │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 前端调用 POST /api/v1/auth/refresh                    │   │
│  │    -携带 Refresh Token                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 后端验证 Refresh Token                                  │   │
│  │    - 检查是否过期                                          │   │
│  │    - 检查是否已被使用                                       │   │
│  │    - 标记为已使用                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 生成新的 Access Token 和 Refresh Token                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. 返回新的 Token                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 2. JWT Token 结构

### 2.1 Access Token Payload

```json
{
  "iss": "smartaudit",
  "sub": "user_uuid",
  "type": "access",
  "tenant_id": "tenant_uuid",
  "roles": ["ADMIN", "AUDITOR"],
  "permissions": [
    "PROJECT_VIEW",
    "PROJECT_CREATE",
    "AUDIT_EXECUTE"
  ],
  "iat": 1705900000,
  "exp": 1705901800
}
```

### 2.2 Token 配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| Access Token 有效期 | 30 分钟 | 短期令牌 |
| Refresh Token 有效期 | 7 天 | 长期令牌 |
| Token 加密算法 | HS256 | 对称加密 |
| Token 存储 | 内存 (Access) / HttpOnly Cookie (Refresh) | 安全存储 |

## 3. 权限校验

### 3.1 Cabin 权限管理

系统采用 [Cabin](https://github.com/cdleee/cabin) 开源 RBAC 权限管理库实现细粒度权限控制。

**Cabin 核心概念：**
- **用户 (User)**：系统登录用户
- **角色 (Role)**：权限集合，如管理员、审核员、普通员工
- **权限 (Permission)**：具体操作权限，如项目查看、数据审核
- **租户 (Tenant)**：多租户隔离

**Cabin 特点：**
- 基于角色（Role）的权限管理
- 支持权限继承和组
- 装饰器模式，简洁易用
- 支持多租户数据隔离
- 内置缓存，高性能

### 3.2 权限校验流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       权限校验流程 (Cabin)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  请求带着 Access Token                                           │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. Cabin 解析 Token，获取用户信息和角色列表               │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. Cabin 查询用户角色对应的权限                           │   │
│  │    - 读取角色表 (biz_roles)                               │   │
│  │    - 读取角色权限关系表 (biz_role_permissions)            │   │
│  │    - 合并用户额外权限 (biz_accounts.extra_permissions)     │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 检查所需权限是否在用户权限列表中                        │   │
│  │                                                          │   │
│  │    required_permission ∈ user_permissions                │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ├─ 是 ──► 继续处理请求                                      │
│    │                                                           │
│    └─ 否 ──► 返回 403 Forbidden                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Cabin 权限实现

```python
# app/api/deps.py
from functools import wraps
from fastapi import HTTPException, status, Depends
from cabin.auth import need_permissions, get_current_user_from_token
from cabin.models import User

def require_permission(*permissions):
    """
    Cabin 权限校验装饰器

    Usage:
        @router.get("/projects")
        @need_permissions("PROJECT_VIEW")
        async def list_projects(...):
            ...
    """
    @wraps
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Cabin 自动从 Token 解析用户并校验权限
            return await func(*args, **kwargs)
        return wrapper
    return decorator


async def get_current_user(token: str = Depends(lambda: None)) -> User:
    """获取当前用户（Cabin 实现）"""
    if not token:
        raise HTTPException(status_code=401, detail="未登录")

    # Cabin 解析 Token
    user = await get_current_user_from_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Token 无效")

    return user


# cabin 权限定义示例
PERMISSIONS = {
    # 项目权限
    "PROJECT_VIEW": "查看项目",
    "PROJECT_CREATE": "创建项目",
    "PROJECT_EDIT": "编辑项目",
    "PROJECT_DELETE": "删除项目",

    # 审核权限
    "AUDIT_VIEW": "查看审核数据",
    "AUDIT_EXECUTE": "执行审核",
    "AUDIT_RECHECK": "二次复核",

    # 规则权限
    "RULE_VIEW": "查看规则",
    "RULE_CREATE": "创建规则",
    "RULE_EDIT": "编辑规则",
    "RULE_DELETE": "删除规则",

    # 账户权限
    "ACCOUNT_VIEW": "查看账户",
    "ACCOUNT_CREATE": "创建账户",
    "ACCOUNT_EDIT": "编辑账户",
    "ACCOUNT_DELETE": "删除账户",
    "ACCOUNT_PASSWORD": "重置密码",

    # 系统权限
    "SYSTEM_CONFIG": "系统配置",
    "BACKUP_VIEW": "查看备份",
    "BACKUP_CREATE": "创建备份",
    "BACKUP_RESTORE": "恢复备份",
    "VERSION_VIEW": "版本管理",
    "VERSION_UPGRADE": "执行升级",
}
```

### 3.4 Cabin 角色定义

```python
# cabin/roles.py
from cabin.models import Role

# 系统内置角色
SYSTEM_ROLES = {
    "SUPER_ADMIN": {
        "name": "超级管理员",
        "description": "租户最高管理员，拥有所有权限",
        "is_system": True,
        "permissions": ["*"],  # 所有权限
    },
    "ADMIN": {
        "name": "管理员",
        "description": "普通管理员，管理日常运营",
        "is_system": True,
        "permissions": [
            "PROJECT_VIEW", "PROJECT_CREATE", "PROJECT_EDIT",
            "AUDIT_VIEW", "AUDIT_EXECUTE", "AUDIT_RECHECK",
            "RULE_VIEW", "RULE_CREATE", "RULE_EDIT",
            "ACCOUNT_VIEW", "ACCOUNT_CREATE", "ACCOUNT_EDIT",
        ],
    },
    "AUDITOR": {
        "name": "审核员",
        "description": "执行审核操作",
        "is_system": True,
        "permissions": [
            "PROJECT_VIEW",
            "AUDIT_VIEW", "AUDIT_EXECUTE",
        ],
    },
    "STAFF": {
        "name": "普通员工",
        "description": "提交审核数据",
        "is_system": True,
        "permissions": [
            "PROJECT_VIEW",
            "AUDIT_VIEW",
        ],
    },
}
```

### 3.5 Cabin 多租户隔离

```python
# cabin/tenant.py
from cabin.auth import tenant_required
from cabin.tenant import TenantScope

@tenant_required
async def get_tenant_data(
    tenant_id: str,
    current_user = Depends(get_current_user)
):
    """Cabin 自动校验用户所属租户"""
    # Cabin 自动验证 current_user.tenant_id == tenant_id
    # 确保数据隔离
    return await fetch_tenant_data(tenant_id)


class TenantScope:
    """Cabin 租户作用域"""

    @staticmethod
    def filter(query, tenant_id: str):
        """自动添加租户过滤条件"""
        return query.filter(tenant_id=tenant_id)

    @staticmethod
    def scope_model(model_class):
        """装饰器：自动过滤租户数据"""
        def decorator(func):
            async def wrapper(*args, **kwargs):
                # 自动注入 tenant_id
                return await func(*args, **kwargs)
            return wrapper
        return decorator
```

## 4. 多租户隔离

### 4.1 租户隔离策略

| 层级 | 隔离方式 | 说明 |
|------|----------|------|
| 数据库 | 独立数据库 / 表级隔离 | 独立部署 / SaaS |
| API | tenant_id 参数 | 验证请求所属租户 |
| 缓存 | Redis Key 前缀 | 按租户隔离缓存 |
| 日志 | tenant_id 字段 | 按租户筛选日志 |

### 4.2 租户校验

```python
# app/api/deps.py
async def get_current_tenant(
    current_user: User = Depends(get_current_user)
) -> Tenant:
    """获取当前用户所属租户"""
    if not current_user.tenant_id:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="用户未关联租户"
        )

    tenant = await tenant_repo.get(current_user.tenant_id)

    if not tenant:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="租户不存在"
        )

    if tenant.status != 'active':
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="租户已禁用"
        )

    return tenant


async def verify_data_tenant(
    data_tenant_id: UUID,
    current_tenant: Tenant = Depends(get_current_tenant)
):
    """校验数据所属租户"""
    if data_tenant_id != current_tenant.id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="无权访问此数据"
        )
```

## 5. 登录安全

### 5.1 登录失败限制

```python
# app/services/auth_service.py
class LoginAttemptService:
    """登录尝试服务"""

    MAX_ATTEMPTS = 5          # 最大失败次数
    LOCKOUT_DURATION = 15     # 锁定时间(分钟)

    async def check_login_attempt(self, username: str) -> bool:
        """检查是否允许登录"""
        key = f"login_attempt:{username}"

        attempts = await self.redis.get(key)
        if attempts and int(attempts) >= self.MAX_ATTEMPTS:
            return False

        return True

    async def record_failed_attempt(self, username: str):
        """记录失败尝试"""
        key = f"login_attempt:{username}"

        # 增加失败计数
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.LOCKOUT_DURATION * 60)
        await pipe.execute()

    async def clear_attempts(self, username: str):
        """清除失败记录"""
        key = f"login_attempt:{username}"
        await self.redis.delete(key)
```

### 5.2 密码策略

| 策略 | 要求 |
|------|------|
| 最小长度 | 8 位 |
| 必须包含 | 字母和数字 |
| 禁止规则 | 不能与用户名相同 |
| 过期时间 | 可配置（默认 90 天） |
| 历史记录 | 最近 5 个密码不能重复 |

## 6. 单点登录 (SSO)

### 6.1 SSO 配置

```typescript
interface SSOConfig {
  enabled: boolean;
  type: 'oidc' | 'saml' | 'cas';  // SSO 协议类型
  oidc?: {
    issuer: string;
    clientId: string;
    clientSecret: string;
    scopes: string[];
  };
  saml?: {
    idpMetadataUrl: string;
    spEntityId: string;
    acsUrl: string;
  };
}
```

### 6.2 SSO 登录流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      SSO 登录流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  用户点击 "SSO 登录"                                             │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 跳转 SSO 登录页面                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 用户在 SSO 页面完成认证                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. SSO 返回授权码到回调地址                                 │   │
│  │    GET /auth/sso/callback?code=xxx                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. 交换授权码获取用户信息                                    │   │
│  │    验证 JWT 或用户信息                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│    │                                                           │
│    ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. 在 SmartAudit 中创建/更新用户并登录                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 7. API 鉴权

### 7.1 请求头

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

X-Tenant-ID: 550e8400-e29b-41d4-a716-446655440000  (可选)
X-Request-ID: req_abc123  (可选)
```

### 7.2 鉴权中间件

```python
# app/middleware/auth.py
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    # 跳过白名单路径
    if request.url.path in WHITE_LIST:
        return await call_next(request)

    # 获取 Token
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        return JSONResponse(
            status_code=401,
            content={"error": "Missing token"}
        )

    token = auth_header[7:]

    # 验证 Token
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=["HS256"]
        )

        # 附加用户信息到请求
        request.state.user_id = payload["sub"]
        request.state.tenant_id = payload.get("tenant_id")

    except jwt.ExpiredSignatureError:
        return JSONResponse(
            status_code=401,
            content={"error": "Token expired"}
        )
    except jwt.InvalidTokenError:
        return JSONResponse(
            status_code=401,
            content={"error": "Invalid token"}
        )

    response = await call_next(request)
    return response
```

## 8. 敏感操作二次验证

### 8.1 需要二次验证的操作

| 操作 | 验证方式 |
|------|----------|
| 删除账户 | 密码验证 |
| 修改安全配置 | 密码验证 |
| 导出大量数据 | 短信/邮件验证码 |
| 敏感权限分配 | 二次密码确认 |

### 8.2 二次验证实现

```python
# app/services/two_factor_service.py
class TwoFactorService:
    async def send_verification_code(
        self,
        user_id: str,
        method: 'sms' | 'email'
    ) -> str:
        """发送验证码"""
        code = ''.join([str(random.randint(0, 9)) for _ in range(6)])
        key = f"2fa:{user_id}:{method}:{code}"

        # 存储验证码 (5分钟有效)
        await self.redis.setex(key, 300, user_id)

        if method == 'sms':
            await self.sms_service.send(user.phone, f"验证码: {code}")
        elif method == 'email':
            await self.email_service.send(user.email, f"验证码: {code}")

        return code  # 开发环境返回验证码

    async def verify_code(
        self,
        user_id: str,
        method: 'sms' | 'email',
        code: str
    ) -> bool:
        """验证验证码"""
        key = f"2fa:{user_id}:{method}:{code}"
        stored_user = await self.redis.get(key)

        if stored_user == user_id:
            await self.redis.delete(key)
            return True

        return False
```
