# SaaS API 规范

## 1. API 基本信息

- **基础路径**: `/saas/api/v1`
- **认证方式**: Bearer Token (JWT)
- **请求格式**: `Content-Type: application/json`
- **响应格式**: JSON

## 2. 认证接口

### 2.1 登录

```
POST /saas/api/v1/auth/login
```

**请求参数**:
```typescript
interface LoginRequest {
  username: string;    // 用户名
  password: string;    // 密码
  captcha?: string;    // 验证码
  captcha_key?: string; // 验证码key
}
```

**响应**:
```typescript
interface LoginResponse {
  access_token: string;      // Access Token
  refresh_token: string;    // Refresh Token
  expires_in: number;       // 过期时间(秒)
  token_type: 'Bearer';
  admin: {
    id: string;
    username: string;
    name: string;
    email: string;
    role: string;
    avatar?: string;
  };
}
```

**示例**:
```bash
curl -X POST /saas/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"******"}'
```

### 2.2 登出

```
POST /saas/api/v1/auth/logout
```

**响应**:
```typescript
interface LogoutResponse {
  code: 0;
  message: 'success';
}
```

### 2.3 获取当前管理员

```
GET /saas/api/v1/auth/me
```

**响应**:
```typescript
interface GetMeResponse {
  id: string;
  username: string;
  name: string;
  email: string;
  phone?: string;
  avatar?: string;
  role: string;
  status: string;
  last_login_at?: string;
  created_at: string;
}
```

### 2.4 修改密码

```
PUT /saas/api/v1/auth/password
```

**请求参数**:
```typescript
interface ChangePasswordRequest {
  old_password: string;   // 旧密码
  new_password: string;   // 新密码 (8位以上，包含字母和数字)
}
```

**响应**:
```typescript
interface ChangePasswordResponse {
  code: 0;
  message: '密码修改成功';
}
```

---

## 3. 租户管理接口

### 3.1 租户列表

```
GET /saas/api/v1/tenants
```

**查询参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页数量，默认20 |
| keyword | string | 否 | 搜索关键词 |
| status | string | 否 | 状态筛选 |
| package_type | string | 否 | 套餐筛选 |
| start_date | string | 否 | 创建开始日期 |
| end_date | string | 否 | 创建结束日期 |

**响应**:
```typescript
interface TenantListResponse {
  code: 0;
  data: {
    items: Tenant[];
    total: number;
    page: number;
    page_size: number;
    total_pages: number;
  };
}

interface Tenant {
  id: string;
  name: string;
  code: string;
  contact_name: string;
  contact_email: string;
  contact_phone: string;
  address?: string;
  logo_url?: string;
  app_id: string;
  package_type: 'basic' | 'professional' | 'enterprise';
  deploy_mode: 'saas' | 'dedicated';
  service_start_date: string;
  service_end_date: string;
  status: 'active' | 'inactive' | 'expired';
  user_count: number;
  project_count: number;
  created_at: string;
}
```

### 3.2 开通租户

```
POST /saas/api/v1/tenants
```

**请求参数**:
```typescript
interface CreateTenantRequest {
  name: string;              // 租户名称
  code: string;              // 租户代码
  contact_name: string;        // 联系人姓名
  contact_email: string;       // 联系人邮箱
  contact_phone: string;       // 联系人电话
  address?: string;           // 地址
  package_type?: string;       // 套餐类型
  service_days?: number;       // 服务天数
  deploy_mode?: string;        // 部署模式
}
```

**响应**:
```typescript
interface CreateTenantResponse {
  code: 0;
  data: {
    id: string;
    name: string;
    code: string;
    app_id: string;
    app_secret: string;        // 仅此次返回，明文显示
    package_type: string;
    deploy_mode: string;
    service_start_date: string;
    service_end_date: string;
    status: string;
    created_at: string;
  };
}
```

### 3.3 租户详情

```
GET /saas/api/v1/tenants/{id}
```

**路径参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| id | string | 租户ID |

**响应**:
```typescript
interface TenantDetailResponse {
  code: 0;
  data: {
    // 基本信息
    id: string;
    name: string;
    code: string;
    contact_name: string;
    contact_email: string;
    contact_phone: string;
    address?: string;
    logo_url?: string;

    // 服务信息
    app_id: string;
    app_secret_masked: string;   // APPSECRET掩码
    package_type: string;
    deploy_mode: string;
    service_start_date: string;
    service_end_date: string;
    status: string;

    // 统计数据
    statistics: {
      user_count: number;        // 用户数
      project_count: number;     // 项目数
      total_audit_data: number;   // 累计审核数据
      month_audit_data: number;   // 本月审核数据
    };

    // 时间信息
    created_at: string;
    updated_at: string;
  };
}
```

### 3.4 编辑租户

```
PUT /saas/api/v1/tenants/{id}
```

**请求参数**:
```typescript
interface UpdateTenantRequest {
  name?: string;
  contact_name?: string;
  contact_email?: string;
  contact_phone?: string;
  address?: string;
  package_type?: string;
  service_end_date?: string;
}
```

### 3.5 删除租户

```
DELETE /saas/api/v1/tenants/{id}
```

**响应**:
```typescript
interface DeleteTenantResponse {
  code: 0;
  message: '删除成功';
}
```

### 3.6 禁用租户

```
POST /saas/api/v1/tenants/{id}/disable
```

### 3.7 启用租户

```
POST /saas/api/v1/tenants/{id}/enable
```

### 3.8 重置 APPSECRET

```
POST /saas/api/v1/tenants/{id}/reset-secret
```

**响应**:
```typescript
interface ResetSecretResponse {
  code: 0;
  data: {
    app_id: string;
    app_secret: string;        // 新APPSECRET，仅此次返回
  };
}
```

---

## 4. 版本管理接口

### 4.1 版本列表

```
GET /saas/api/v1/versions
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| page_size | int | 每页数量 |
| release_type | string | 发布类型 |
| status | string | enabled/disabled |

**响应**:
```typescript
interface VersionListResponse {
  code: 0;
  data: {
    items: SystemVersion[];
    total: number;
    page: number;
    page_size: number;
  };
}

interface SystemVersion {
  id: string;
  version: string;           // "1.2.3"
  version_code: number;       // 10203
  release_type: string;        // stable/beta/release
  release_notes: string;       // Markdown
  download_url: string;
  checksum: string;
  is_mandatory: boolean;
  is_enabled: boolean;
  min_compatible_version?: string;
  published_at: string;
  statistics: {
    total_tenants: number;
    upgraded_tenants: number;
    pending_upgrades: number;
    upgrade_rate: number;
  };
  created_at: string;
}
```

### 4.2 发布版本

```
POST /saas/api/v1/versions
```

**请求参数**:
```typescript
interface CreateVersionRequest {
  version: string;              // 版本号 "1.2.3"
  version_code: number;          // 版本数字 10203
  release_type: string;         // stable/beta/release
  release_notes: string;         // Markdown
  download_url: string;          // 下载地址
  checksum: string;             // SHA256
  is_mandatory?: boolean;        // 是否强制更新
  is_enabled?: boolean;          // 是否启用
  min_compatible_version?: string;// 最低兼容版本
}
```

### 4.3 版本详情

```
GET /saas/api/v1/versions/{id}
```

### 4.4 编辑版本

```
PUT /saas/api/v1/versions/{id}
```

**请求参数**:
```typescript
interface UpdateVersionRequest {
  release_notes?: string;
  download_url?: string;
  checksum?: string;
  is_mandatory?: boolean;
  is_enabled?: boolean;
}
```

### 4.5 禁用版本

```
POST /saas/api/v1/versions/{id}/disable
```

### 4.6 升级记录

```
GET /saas/api/v1/versions/{id}/upgrade-records
```

**响应**:
```typescript
interface UpgradeRecordListResponse {
  code: 0;
  data: {
    items: UpgradeRecord[];
    total: number;
  };
}

interface UpgradeRecord {
  id: string;
  tenant_id: string;
  tenant_name: string;
  from_version: string;
  to_version: string;
  status: 'pending' | 'running' | 'success' | 'failed' | 'rollback';
  error_msg?: string;
  started_at: string;
  completed_at?: string;
  operated_by: string;
  operated_by_name: string;
}
```

---

## 5. 配置管理接口

### 5.1 获取配置

```
GET /saas/api/v1/config
```

**响应**:
```typescript
interface GetConfigResponse {
  code: 0;
  data: {
    platform_name: string;
    platform_logo: string;
    default_package: string;
    token_expire_minutes: number;
    refresh_token_expire_days: number;
    max_users_per_tenant: number;
    max_projects_per_tenant: number;
    version_check_interval_hours: number;
  };
}
```

### 5.2 更新配置

```
PUT /saas/api/v1/config
```

**请求参数**:
```typescript
interface UpdateConfigRequest {
  platform_name?: string;
  platform_logo?: string;
  default_package?: string;
  token_expire_minutes?: number;
  refresh_token_expire_days?: number;
  max_users_per_tenant?: number;
  max_projects_per_tenant?: number;
}
```

---

## 6. 统计接口

### 6.1 运营概览

```
GET /saas/api/v1/statistics/overview
```

**响应**:
```typescript
interface OverviewResponse {
  code: 0;
  data: {
    tenants: {
      total: number;           // 累计租户
      active: number;          // 活跃租户
      inactive: number;        // 已禁用
      expired: number;         // 已到期
    };
    users: {
      total: number;           // 累计用户
      month_new: number;        // 本月新增
    };
    audit_data: {
      total: number;           // 累计审核数据
      month_total: number;      // 本月审核
      today_total: number;      // 今日审核
    };
    upgrades: {
      total: number;           // 累计升级
      success: number;         // 成功
      failed: number;         // 失败
    };
  };
}
```

### 6.2 租户统计

```
GET /saas/api/v1/statistics/tenants
```

**响应**:
```typescript
interface TenantStatisticsResponse {
  code: 0;
  data: {
    monthly_trend: {
      month: string;           // "2024-01"
      new_tenants: number;
      total_tenants: number;
    }[];
    package_distribution: {
      package: string;
      count: number;
      percentage: number;
    }[];
    top_active_tenants: {
      tenant_id: string;
      tenant_name: string;
      audit_count: number;
    }[];
  };
}
```

### 6.3 版本统计

```
GET /saas/api/v1/statistics/versions
```

**响应**:
```typescript
interface VersionStatisticsResponse {
  code: 0;
  data: {
    version_distribution: {
      version: string;
      tenant_count: number;
      percentage: number;
    }[];
    upgrade_trend: {
      month: string;
      upgrade_count: number;
    }[];
    pending_upgrades: {
      version: string;
      tenant_count: number;
      is_mandatory: boolean;
    }[];
  };
}
```

---

## 7. 错误码

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
| 40002 | 租户已存在 |
| 40003 | 版本号已存在 |
| 40004 | 租户已禁用 |
| 40005 | 租户已到期 |
| 50001 | 系统内部错误 |
