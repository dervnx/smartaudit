# 租户 API 规范

## 1. API 基本信息

- **基础路径**: `/api/v1`
- **认证方式**: Bearer Token (JWT)
- **请求格式**: `Content-Type: application/json`
- **响应格式**: JSON

## 2. 认证接口

### 2.1 登录

```
POST /api/v1/auth/login
```

**请求参数**:
```typescript
interface LoginRequest {
  username: string;    // 用户名
  password: string;    // 密码
}
```

**响应**:
```typescript
interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  tokenType: 'Bearer';
  user: {
    id: string;
    username: string;
    name: string;
    avatar?: string;
    roles: string[];
    permissions: string[];
  };
}
```

### 2.2 登出

```
POST /api/v1/auth/logout
```

### 2.3 获取当前用户

```
GET /api/v1/auth/me
```

### 2.4 修改密码

```
PUT /api/v1/auth/password
```

**请求参数**:
```typescript
interface ChangePasswordRequest {
  oldPassword: string;
  newPassword: string;
}
```

---

## 3. 账户管理接口

### 3.1 账户列表

```
GET /api/v1/accounts
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| keyword | string | 搜索关键词 |
| status | string | 状态筛选 |
| roleId | string | 角色筛选 |

**响应**:
```typescript
interface AccountListResponse {
  code: 0;
  data: {
    items: Account[];
    total: number;
    page: number;
    pageSize: number;
  };
}

interface Account {
  id: string;
  username: string;
  name: string;
  email?: string;
  phone?: string;
  avatar?: string;
  roles: Role[];
  status: 'active' | 'inactive';
  lastLoginAt?: string;
  created_at: string;
}
```

### 3.2 创建账户

```
POST /api/v1/accounts
```

**请求参数**:
```typescript
interface CreateAccountRequest {
  username: string;
  password: string;
  name: string;
  email?: string;
  phone?: string;
  roleIds: string[];
  extraPermissions?: string[];
  status?: 'active' | 'inactive';
}
```

### 3.3 账户详情

```
GET /api/v1/accounts/{id}
```

### 3.4 编辑账户

```
PUT /api/v1/accounts/{id}
```

**请求参数**:
```typescript
interface UpdateAccountRequest {
  name?: string;
  email?: string;
  phone?: string;
  avatar?: string;
  status?: 'active' | 'inactive';
}
```

### 3.5 删除账户

```
DELETE /api/v1/accounts/{id}
```

### 3.6 分配角色

```
PUT /api/v1/accounts/{id}/roles
```

**请求参数**:
```typescript
interface AssignRolesRequest {
  roleIds: string[];
}
```

### 3.7 配置权限

```
PUT /api/v1/accounts/{id}/permissions
```

**请求参数**:
```typescript
interface ConfigurePermissionsRequest {
  extraPermissions: string[];
  removePermissions?: string[];
}
```

### 3.8 重置密码

```
POST /api/v1/accounts/{id}/reset-password
```

**响应**:
```typescript
interface ResetPasswordResponse {
  code: 0;
  data: {
    password: string;  // 临时密码，仅此次返回
  };
}
```

### 3.9 修改个人资料

```
PUT /api/v1/accounts/profile
```

**请求参数**:
```typescript
interface UpdateProfileRequest {
  name?: string;
  avatar?: string;
  email?: string;
  phone?: string;
}
```

### 3.10 获取我的权限

```
GET /api/v1/accounts/my-permissions
```

**响应**:
```typescript
interface MyPermissionsResponse {
  code: 0;
  data: {
    roles: string[];
    permissions: string[];
    roleDetails: {
      code: string;
      name: string;
      permissions: string[];
    }[];
  };
}
```

---

## 4. 角色管理接口

### 4.1 角色列表

```
GET /api/v1/roles
```

**响应**:
```typescript
interface RoleListResponse {
  code: 0;
  data: Role[];
}

interface Role {
  id: string;
  name: string;
  code: string;
  description?: string;
  isSystem: boolean;
  userCount: number;
  permissions: string[];
  created_at: string;
}
```

### 4.2 创建角色

```
POST /api/v1/roles
```

**请求参数**:
```typescript
interface CreateRoleRequest {
  name: string;
  code: string;
  description?: string;
  permissionIds?: string[];
}
```

### 4.3 角色详情

```
GET /api/v1/roles/{id}
```

### 4.4 编辑角色

```
PUT /api/v1/roles/{id}
```

### 4.5 删除角色

```
DELETE /api/v1/roles/{id}
```

### 4.6 配置角色权限

```
PUT /api/v1/roles/{id}/permissions
```

**请求参数**:
```typescript
interface ConfigureRolePermissionsRequest {
  permissions: string[];  // 权限code列表
}
```

---

## 5. 项目管理接口

### 5.1 项目列表

```
GET /api/v1/projects
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| keyword | string | 搜索关键词 |
| status | string | 状态筛选 |

### 5.2 创建项目

```
POST /api/v1/projects
```

**请求参数**:
```typescript
interface CreateProjectRequest {
  name: string;
  code: string;
  description?: string;
  fieldsDef: FieldDefinition[];
  ruleIds?: string[];
  needManualReview?: boolean;
  taskAssignType?: 'round_robin' | 'load_balance' | 'efficiency' | 'random' | 'manual';
}

interface FieldDefinition {
  name: string;
  label: string;
  type: 'string' | 'number' | 'boolean' | 'date' | 'datetime' |
        'select' | 'multiSelect' | 'file' | 'image' |
        'region' | 'tags' | 'idCard' | 'phone' | 'email';
  required?: boolean;
  options?: string[];
  default?: any;
  validation?: string;
}
```

### 5.3 项目详情

```
GET /api/v1/projects/{id}
```

### 5.4 编辑项目

```
PUT /api/v1/projects/{id}
```

### 5.5 删除项目

```
DELETE /api/v1/projects/{id}
```

### 5.6 项目统计

```
GET /api/v1/projects/{id}/statistics
```

**响应**:
```typescript
interface ProjectStatisticsResponse {
  code: 0;
  data: {
    totalData: number;
    pendingData: number;
    inProgressData: number;
    passedData: number;
    rejectedData: number;
    recheckingData: number;
    progressRate: number;
    auditorWorkload: {
      auditorId: string;
      auditorName: string;
      completedCount: number;
    }[];
    rejectReasonDistribution: {
      reason: string;
      count: number;
    }[];
    avgDuration: number;
  };
}
```

---

## 6. 规则管理接口

### 6.1 规则列表

```
GET /api/v1/rules
```

### 6.2 创建规则

```
POST /api/v1/rules
```

**请求参数**:
```typescript
interface CreateRuleRequest {
  name: string;
  description?: string;
  conditions: ConditionGroup;
  resultPass?: string;
  resultReject?: string;
  resultSuggest?: string;
  priority?: number;
}

interface ConditionGroup {
  logic: 'ALL' | 'ANY';
  conditions: (Condition | ConditionGroup)[];
}

interface Condition {
  field: string;
  operator: 'equals' | 'notEquals' | 'contains' | 'greaterThan' |
            'lessThan' | 'between' | 'in' | 'notIn' | 'isEmpty' |
            'isNotEmpty' | 'matches' | 'idCardValid' | 'educationVerify';
  value: any;
}
```

### 6.3 规则详情

```
GET /api/v1/rules/{id}
```

### 6.4 编辑规则

```
PUT /api/v1/rules/{id}
```

### 6.5 删除规则

```
DELETE /api/v1/rules/{id}
```

### 6.6 测试规则

```
POST /api/v1/rules/test
```

**请求参数**:
```typescript
interface TestRuleRequest {
  conditions: ConditionGroup;
  testData: Record<string, any>;
}
```

**响应**:
```typescript
interface TestRuleResponse {
  code: 0;
  data: {
    passed: boolean;
    matchedConditions: string[];
    unmatchedConditions: string[];
    result: 'pass' | 'reject';
  };
}
```

---

## 7. 审核接口

### 7.1 待审核列表

```
GET /api/v1/audit/pending
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| projectId | string | 项目ID |

**响应**:
```typescript
interface PendingAuditResponse {
  code: 0;
  data: {
    items: AuditData[];
    total: number;
    pendingCount: number;
    page: number;
    pageSize: number;
  };
}

interface AuditData {
  id: string;
  projectId: string;
  projectName: string;
  inputData: Record<string, any>;
  autoResult?: 'pass' | 'reject' | 'pending';
  status: 'pending' | 'in_progress' | 'passed' | 'rejected' | 'rechecking';
  priority: number;
  created_at: string;
}
```

### 7.2 已审核列表

```
GET /api/v1/audit/completed
```

### 7.3 提交数据

```
POST /api/v1/audit/submit
```

**请求参数**:
```typescript
interface SubmitAuditRequest {
  projectId: string;
  inputData: Record<string, any>;
  priority?: number;
}
```

### 7.4 执行审核

```
POST /api/v1/audit/{id}/execute
```

**请求参数**:
```typescript
interface ExecuteAuditRequest {
  result: 'pass' | 'reject' | 'recheck';
  note?: string;
}
```

### 7.5 审核详情

```
GET /api/v1/audit/{id}
```

**响应**:
```typescript
interface AuditDetailResponse {
  code: 0;
  data: {
    id: string;
    projectId: string;
    projectName: string;
    inputData: Record<string, any>;
    ruleResult: {
      passed: boolean;
      conditions: ConditionResult[];
      conclusion: string;
    };
    autoResult?: string;
    finalResult?: string;
    auditorId?: string;
    auditorName?: string;
    auditTime?: string;
    auditNote?: string;
    needRecheck: boolean;
    recheckAuditorId?: string;
    recheckTime?: string;
    recheckResult?: string;
    status: string;
    created_at: string;
  };
}

interface ConditionResult {
  path: string;
  field: string;
  operator: string;
  expected: any;
  actual: any;
  passed: boolean;
}
```

---

## 8. 任务接口

### 8.1 我的任务

```
GET /api/v1/tasks/my
```

**响应**:
```typescript
interface MyTasksResponse {
  code: 0;
  data: {
    pendingCount: number;
    recheckCount: number;
    todayCompleted: number;
    weekCompleted: number;
    monthCompleted: number;
    yearCompleted: number;
    totalCompleted: number;
    avgDuration: number;
  };
}
```

### 8.2 任务列表

```
GET /api/v1/tasks
```

**查询参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| pageSize | int | 每页数量 |
| status | string | 任务状态 |
| auditorId | string | 审核员ID |
| type | string | review/recheck |

### 8.3 开始任务

```
POST /api/v1/tasks/{id}/start
```

### 8.4 完成审核

```
POST /api/v1/tasks/{id}/complete
```

**请求参数**:
```typescript
interface CompleteTaskRequest {
  result: 'pass' | 'reject';
  note?: string;
}
```

### 8.5 跳过任务

```
POST /api/v1/tasks/{id}/skip
```

---

## 9. 三方接口配置

### 9.1 配置列表

```
GET /api/v1/third-party/configs
```

### 9.2 创建配置

```
POST /api/v1/third-party/configs
```

**请求参数**:
```typescript
interface CreateThirdPartyConfigRequest {
  name: string;
  type: 'ID_CARD_VERIFY' | 'EDUCATION_VERIFY' | 'DEGREE_VERIFY' |
        'BANK_FOUR_ELEMENT' | 'OPERATOR_VERIFY';
  url: string;
  method?: 'GET' | 'POST';
  headers?: Record<string, string>;
  paramsTemplate?: Record<string, any>;
  authType?: 'none' | 'bearer' | 'oauth2' | 'api_key';
  authConfig?: {
    tokenUrl?: string;
    clientId?: string;
    clientSecret?: string;
    apiKey?: string;
  };
  timeout?: number;
  retryCount?: number;
}
```

### 9.3 测试连接

```
POST /api/v1/third-party/configs/{id}/test
```

### 9.4 调用日志

```
GET /api/v1/third-party/logs
```

### 9.5 调用统计

```
GET /api/v1/third-party/statistics
```

**响应**:
```typescript
interface ThirdPartyStatisticsResponse {
  code: 0;
  data: {
    totalCalls: number;
    monthCalls: number;
    todayCalls: number;
    successRate: number;
    avgResponseTime: number;
    callRanking: {
      configId: string;
      configName: string;
      callCount: number;
    }[];
  };
}
```

---

## 10. 统计接口

### 10.1 运营概览

```
GET /api/v1/statistics/overview
```

### 10.2 项目统计

```
GET /api/v1/statistics/project/{id}
```

### 10.3 审核员统计

```
GET /api/v1/statistics/auditor
```

---

## 11. 备份接口

### 11.1 备份规则列表

```
GET /api/v1/backup/rules
```

### 11.2 创建备份规则

```
POST /api/v1/backup/rules
```

**请求参数**:
```typescript
interface CreateBackupRuleRequest {
  name: string;
  backupType: 'full' | 'differential' | 'incremental';
  scheduleCron: string;
  retentionDays: number;
  storageLocation: 'local' | 's3' | 'oss';
  storageConfig?: {
    bucket?: string;
    endpoint?: string;
    accessKey?: string;
    secretKey?: string;
  };
}
```

### 11.3 编辑备份规则

```
PUT /api/v1/backup/rules/{id}
```

### 11.4 删除备份规则

```
DELETE /api/v1/backup/rules/{id}
```

### 11.5 备份记录列表

```
GET /api/v1/backup/records
```

### 11.6 手动执行备份

```
POST /api/v1/backup/execute
```

**请求参数**:
```typescript
interface ExecuteBackupRequest {
  ruleId?: string;
  backupType?: 'full' | 'differential' | 'incremental';
}
```

### 11.7 执行恢复

```
POST /api/v1/backup/records/{id}/restore
```

---

## 12. 版本接口

### 12.1 当前版本

```
GET /api/v1/version/current
```

**响应**:
```typescript
interface CurrentVersionResponse {
  code: 0;
  data: {
    version: string;
    versionCode: number;
    lastCheckAt: string;
    lastUpdateAt: string;
    autoUpdateEnabled: boolean;
    updateStatus: 'idle' | 'checking' | 'downloading' | 'upgrading';
  };
}
```

### 12.2 检查更新

```
GET /api/v1/version/check
```

### 12.3 执行升级

```
POST /api/v1/version/upgrade
```

**请求参数**:
```typescript
interface ExecuteUpgradeRequest {
  targetVersion: string;
  createBackup?: boolean;
}
```

### 12.4 升级记录

```
GET /api/v1/version/records
```

---

## 13. 错误码

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 10001 | 参数错误 |
| 20001 | 未授权 |
| 20002 | Token无效 |
| 20003 | Token过期 |
| 20004 | 权限不足 |
| 30001 | 资源不存在 |
| 30002 | 资源已存在 |
| 40001 | 业务逻辑错误 |
| 50001 | 系统内部错误 |
