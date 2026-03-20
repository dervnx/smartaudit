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
  captchaKey?: string; // 验证码key
}
```

**响应**:
```typescript
interface LoginResponse {
  accessToken: string;      // Access Token
  refreshToken: string;    // Refresh Token
  expiresIn: number;       // 过期时间(秒)
  tokenType: 'Bearer';
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
  lastLoginAt?: string;
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
  oldPassword: string;   // 旧密码
  newPassword: string;   // 新密码 (8位以上，包含字母和数字)
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
| pageSize | int | 否 | 每页数量，默认20 |
| keyword | string | 否 | 搜索关键词 |
| status | string | 否 | 状态筛选 |
| packageType | string | 否 | 套餐筛选 |
| startDate | string | 否 | 创建开始日期 |
| endDate | string | 否 | 创建结束日期 |

**响应**:
```typescript
interface TenantListResponse {
  code: 0;
  data: {
    items: Tenant[];
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
}

interface Tenant {
  id: string;
  name: string;
  code: string;
  contactName: string;
  contactEmail: string;
  contactPhone: string;
  address?: string;
  logoUrl?: string;
  appId: string;
  packageType: 'basic' | 'professional' | 'enterprise';
  deployMode: 'saas' | 'dedicated';
  serviceStartDate: string;
  serviceEndDate: string;
  status: 'active' | 'inactive' | 'expired';
  userCount: number;
  projectCount: number;
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
  contactName: string;        // 联系人姓名
  contactEmail: string;       // 联系人邮箱
  contactPhone: string;       // 联系人电话
  address?: string;           // 地址
  packageType?: string;       // 套餐类型
  serviceDays?: number;       // 服务天数
  deployMode?: string;        // 部署模式
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
    appId: string;
    appSecret: string;        // 仅此次返回，明文显示
    packageType: string;
    deployMode: string;
    serviceStartDate: string;
    serviceEndDate: string;
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
    contactName: string;
    contactEmail: string;
    contactPhone: string;
    address?: string;
    logoUrl?: string;

    // 服务信息
    appId: string;
    appSecretMasked: string;   // APPSECRET掩码
    packageType: string;
    deployMode: string;
    serviceStartDate: string;
    serviceEndDate: string;
    status: string;

    // 统计数据
    statistics: {
      userCount: number;        // 用户数
      projectCount: number;     // 项目数
      totalAuditData: number;   // 累计审核数据
      monthAuditData: number;   // 本月审核数据
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
  contactName?: string;
  contactEmail?: string;
  contactPhone?: string;
  address?: string;
  packageType?: string;
  serviceEndDate?: string;
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
    appId: string;
    appSecret: string;        // 新APPSECRET，仅此次返回
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
| pageSize | int | 每页数量 |
| releaseType | string | 发布类型 |
| status | string | enabled/disabled |

**响应**:
```typescript
interface VersionListResponse {
  code: 0;
  data: {
    items: SystemVersion[];
    total: number;
    page: number;
    pageSize: number;
  };
}

interface SystemVersion {
  id: string;
  version: string;           // "1.2.3"
  versionCode: number;       // 10203
  releaseType: string;        // stable/beta/release
  releaseNotes: string;       // Markdown
  downloadUrl: string;
  checksum: string;
  isMandatory: boolean;
  isEnabled: boolean;
  minCompatibleVersion?: string;
  publishedAt: string;
  statistics: {
    totalTenants: number;
    upgradedTenants: number;
    pendingUpgrades: number;
    upgradeRate: number;
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
  versionCode: number;          // 版本数字 10203
  releaseType: string;         // stable/beta/release
  releaseNotes: string;         // Markdown
  downloadUrl: string;          // 下载地址
  checksum: string;             // SHA256
  isMandatory?: boolean;        // 是否强制更新
  isEnabled?: boolean;          // 是否启用
  minCompatibleVersion?: string;// 最低兼容版本
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
  releaseNotes?: string;
  downloadUrl?: string;
  checksum?: string;
  isMandatory?: boolean;
  isEnabled?: boolean;
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
  tenantId: string;
  tenantName: string;
  fromVersion: string;
  toVersion: string;
  status: 'pending' | 'running' | 'success' | 'failed' | 'rollback';
  errorMsg?: string;
  startedAt: string;
  completedAt?: string;
  operatedBy: string;
  operatedByName: string;
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
    platformName: string;
    platformLogo: string;
    defaultPackage: string;
    tokenExpireMinutes: number;
    refreshTokenExpireDays: number;
    maxUsersPerTenant: number;
    maxProjectsPerTenant: number;
    versionCheckIntervalHours: number;
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
  platformName?: string;
  platformLogo?: string;
  defaultPackage?: string;
  tokenExpireMinutes?: number;
  refreshTokenExpireDays?: number;
  maxUsersPerTenant?: number;
  maxProjectsPerTenant?: number;
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
      monthNew: number;        // 本月新增
    };
    auditData: {
      total: number;           // 累计审核数据
      monthTotal: number;      // 本月审核
      todayTotal: number;      // 今日审核
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
    monthlyTrend: {
      month: string;           // "2024-01"
      newTenants: number;
      totalTenants: number;
    }[];
    packageDistribution: {
      package: string;
      count: number;
      percentage: number;
    }[];
    topActiveTenants: {
      tenantId: string;
      tenantName: string;
      auditCount: number;
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
    versionDistribution: {
      version: string;
      tenantCount: number;
      percentage: number;
    }[];
    upgradeTrend: {
      month: string;
      upgradeCount: number;
    }[];
    pendingUpgrades: {
      version: string;
      tenantCount: number;
      isMandatory: boolean;
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
