# 公共数据类型定义

## 1. 基础类型

### 1.1 UUID

```typescript
type UUID = string;  // 格式: "550e8400-e29b-41d4-a716-446655440000"
```

### 1.2 时间类型

```typescript
// ISO 8601 格式
type DateTime = string;  // "2024-01-20T10:30:00Z"
type Date = string;       // "2024-01-20"
type Time = string;       // "10:30:00"
```

### 1.3 状态枚举

```typescript
// 通用状态
type Status = 'active' | 'inactive' | 'deleted';

// 任务状态
type TaskStatus = 'pending' | 'assigned' | 'in_progress' | 'completed' | 'skipped';

// 审核状态
type AuditStatus = 'pending' | 'in_progress' | 'passed' | 'rejected' | 'rechecking';

// 备份状态
type BackupStatus = 'pending' | 'running' | 'success' | 'failed';

// 升级状态
type UpgradeStatus = 'idle' | 'checking' | 'downloading' | 'upgrading' | 'success' | 'failed' | 'rollback';

// 部署模式
type DeployMode = 'saas' | 'dedicated';

// 备份类型
type BackupType = 'full' | 'differential' | 'incremental';
```

## 2. 通用响应

### 2.1 基础响应

```typescript
interface BaseResponse<T = any> {
  code: number;      // 0 = 成功，其他 = 错误
  message: string;   // 信息
  data?: T;          // 数据
  timestamp: number; // 时间戳
}
```

### 2.2 分页响应

```typescript
interface PaginatedResponse<T = any> {
  code: number;
  message: string;
  data: {
    items: T[];
    total: number;       // 总数
    page: number;         // 当前页
    pageSize: number;     // 每页数量
    totalPages: number;  // 总页数
  };
  timestamp: number;
}
```

### 2.3 错误响应

```typescript
interface ErrorResponse {
  code: number;
  message: string;
  details?: any;
  requestId?: string;
  timestamp: number;
}
```

## 3. 用户与认证

### 3.1 用户基本信息

```typescript
interface UserBase {
  id: UUID;
  username: string;
  name: string;
  email?: string;
  phone?: string;
  avatar?: string;
  status: 'active' | 'inactive';
  created_at: DateTime;
  updated_at: DateTime;
}
```

### 3.2 用户详情

```typescript
interface User extends UserBase {
  tenantId?: UUID;        // 租户ID (租户用户)
  roles: Role[];
  permissions: string[];  // 权限code列表
  extraPermissions?: string[];  // 额外权限
  lastLoginAt?: DateTime;
  lastLoginIp?: string;
}
```

### 3.3 角色

```typescript
interface Role {
  id: UUID;
  name: string;          // 角色名称
  code: string;          // 角色代码
  description?: string;
  isSystem: boolean;     // 是否系统内置
  permissions: string[]; // 权限列表
  userCount?: number;
}
```

### 3.4 登录请求/响应

```typescript
interface LoginRequest {
  username: string;
  password: string;
  captcha?: string;
  captchaKey?: string;
}

interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  tokenType: 'Bearer';
  user: User;
}
```

### 3.5 修改密码请求

```typescript
interface ChangePasswordRequest {
  oldPassword: string;
  newPassword: string;
}

interface ResetPasswordRequest {
  phone?: string;
  email?: string;
}
```

## 4. 租户

### 4.1 租户信息

```typescript
interface Tenant {
  id: UUID;
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
  serviceStartDate: Date;
  serviceEndDate: Date;
  status: 'active' | 'inactive' | 'expired';
  created_at: DateTime;
  updated_at: DateTime;
}
```

### 4.2 租户统计

```typescript
interface TenantStatistics {
  userCount: number;
  projectCount: number;
  totalAuditData: number;
  monthAuditData: number;
  todayAuditData: number;
  pendingAuditData: number;
}
```

## 5. 项目

### 5.1 数据项定义

```typescript
interface FieldDefinition {
  name: string;           // 字段名
  label: string;         // 显示标签
  type: FieldType;       // 字段类型
  required?: boolean;    // 是否必填
  options?: string[];    // 下拉选项
  default?: any;         // 默认值
  validation?: string;   // 正则校验
  placeholder?: string;
  maxLength?: number;
  minLength?: number;
  max?: number;
  min?: number;
}

type FieldType =
  | 'string'
  | 'number'
  | 'boolean'
  | 'date'
  | 'datetime'
  | 'select'
  | 'multiSelect'
  | 'file'
  | 'image'
  | 'region'
  | 'tags'
  | 'idCard'
  | 'phone'
  | 'email';
```

### 5.2 项目

```typescript
interface Project {
  id: UUID;
  tenantId: UUID;
  name: string;
  code: string;
  description?: string;
  fieldsDef: FieldDefinition[];
  ruleIds: UUID[];
  needManualReview: boolean;
  taskAssignType: TaskAssignType;
  status: 'active' | 'inactive' | 'archived';
  createdBy?: UUID;
  created_at: DateTime;
  updated_at: DateTime;
}

type TaskAssignType =
  | 'round_robin'    // 轮询
  | 'load_balance'    // 负载均衡
  | 'efficiency'      // 效率优先
  | 'random'          // 随机
  | 'manual';         // 手动
```

### 5.3 项目统计

```typescript
interface ProjectStatistics {
  totalData: number;
  pendingData: number;
  inProgressData: number;
  passedData: number;
  rejectedData: number;
  recheckingData: number;
  progressRate: number;
  auditorWorkload: AuditorWorkload[];
  rejectReasonDistribution: ReasonDistribution[];
  avgDuration: number;  // 秒
}

interface AuditorWorkload {
  auditorId: UUID;
  auditorName: string;
  completedCount: number;
}

interface ReasonDistribution {
  reason: string;
  count: number;
}
```

## 6. 规则

### 6.1 条件

```typescript
interface Condition {
  field: string;      // 字段名
  operator: Operator; // 运算符
  value: any;         // 比较值
}

type Operator =
  | 'equals'
  | 'notEquals'
  | 'contains'
  | 'notContains'
  | 'startsWith'
  | 'endsWith'
  | 'greaterThan'
  | 'lessThan'
  | 'greaterOrEqual'
  | 'lessOrEqual'
  | 'between'
  | 'in'
  | 'notIn'
  | 'isEmpty'
  | 'isNotEmpty'
  | 'matches'
  | 'idCardValid'
  | 'educationVerify'
  | 'degreeVerify';
```

### 6.2 条件组

```typescript
interface ConditionGroup {
  logic: 'ALL' | 'ANY';
  conditions: (Condition | ConditionGroup)[];
}
```

### 6.3 规则

```typescript
interface Rule {
  id: UUID;
  tenantId: UUID;
  name: string;
  description?: string;
  conditions: ConditionGroup;
  resultPass?: string;
  resultReject?: string;
  resultSuggest?: string;
  priority: number;
  status: 'active' | 'inactive';
  createdBy?: UUID;
  created_at: DateTime;
  updated_at: DateTime;
}
```

### 6.4 规则执行结果

```typescript
interface RuleResult {
  passed: boolean;
  conditions: ConditionResult[];
  conclusion: string;
  reason?: string;
  suggestion?: string;
}

interface ConditionResult {
  path: string;           // 条件路径，如 "0.conditions.1"
  field: string;
  operator: string;
  expected: any;
  actual: any;
  passed: boolean;
}
```

## 7. 审核数据

### 7.1 审核数据

```typescript
interface AuditData {
  id: UUID;
  tenantId: UUID;
  projectId: UUID;
  projectName?: string;
  inputData: Record<string, any>;
  ruleResult?: RuleResult;
  autoResult?: 'pass' | 'reject' | 'pending';
  finalResult?: 'pass' | 'reject' | 'recheck';
  auditorId?: UUID;
  auditorName?: string;
  auditTime?: DateTime;
  auditNote?: string;
  needRecheck: boolean;
  recheckAuditorId?: UUID;
  recheckTime?: DateTime;
  recheckResult?: 'pass' | 'reject';
  recheckNote?: string;
  status: AuditStatus;
  priority: number;
  created_at: DateTime;
}
```

## 8. 任务

### 8.1 任务

```typescript
interface Task {
  id: UUID;
  tenantId: UUID;
  auditorId?: UUID;
  auditorName?: string;
  dataId: UUID;
  data?: AuditData;
  type: 'review' | 'recheck';
  status: TaskStatus;
  result?: 'pass' | 'reject';
  note?: string;
  assignedAt?: DateTime;
  startedAt?: DateTime;
  completedAt?: DateTime;
  created_at: DateTime;
}
```

### 8.2 任务统计

```typescript
interface TaskStatistics {
  pendingCount: number;
  recheckCount: number;
  todayCompleted: number;
  weekCompleted: number;
  monthCompleted: number;
  yearCompleted: number;
  totalCompleted: number;
  avgDuration: number;  // 秒
}
```

## 9. 三方接口

### 9.1 接口配置

```typescript
interface ThirdPartyConfig {
  id: UUID;
  tenantId: UUID;
  name: string;
  type: ThirdPartyType;
  url: string;
  method: 'GET' | 'POST';
  headers?: Record<string, string>;
  paramsTemplate?: Record<string, any>;
  authType: 'none' | 'bearer' | 'oauth2' | 'api_key';
  authConfig?: AuthConfig;
  timeout: number;
  retryCount: number;
  status: 'active' | 'inactive';
  created_at: DateTime;
}

type ThirdPartyType =
  | 'ID_CARD_VERIFY'      // 身份证验证
  | 'EDUCATION_VERIFY'    // 学历验证
  | 'DEGREE_VERIFY'       // 学位验证
  | 'BANK_FOUR_ELEMENT'   // 银行四要素
  | 'OPERATOR_VERIFY';    // 运营商验证

interface AuthConfig {
  tokenUrl?: string;
  clientId?: string;
  clientSecret?: string;
  apiKey?: string;
}
```

### 9.2 接口日志

```typescript
interface ThirdPartyLog {
  id: UUID;
  tenantId: UUID;
  configId: UUID;
  configName?: string;
  userId?: UUID;
  requestTime: DateTime;
  requestUrl: string;
  requestMethod: string;
  requestData?: Record<string, any>;
  responseData?: Record<string, any>;
  responseStatus: 'success' | 'failed';
  errorMsg?: string;
  durationMs: number;
}
```

### 9.3 接口统计

```typescript
interface ThirdPartyStatistics {
  totalCalls: number;
  monthCalls: number;
  todayCalls: number;
  successRate: number;
  avgResponseTime: number;
  callRanking: CallRanking[];
}

interface CallRanking {
  configId: UUID;
  configName: string;
  callCount: number;
}
```

## 10. 备份

### 10.1 备份规则

```typescript
interface BackupRule {
  id: UUID;
  tenantId: UUID;
  name: string;
  backupType: BackupType;
  scheduleCron: string;
  retentionDays: number;
  storageLocation: 'local' | 's3' | 'oss';
  storageConfig?: StorageConfig;
  isEnabled: boolean;
  lastBackupAt?: DateTime;
  nextBackupAt?: DateTime;
  createdBy?: UUID;
  created_at: DateTime;
}

interface StorageConfig {
  bucket?: string;
  endpoint?: string;
  accessKey?: string;
  secretKey?: string;
  region?: string;
}
```

### 10.2 备份记录

```typescript
interface BackupRecord {
  id: UUID;
  tenantId: UUID;
  ruleId?: UUID;
  backupType: BackupType;
  status: BackupStatus;
  startedAt: DateTime;
  completedAt?: DateTime;
  filePath?: string;
  fileSize?: number;      // 字节
  checksum?: string;
  errorMsg?: string;
  created_at: DateTime;
}
```

## 11. 版本

### 11.1 系统版本

```typescript
interface SystemVersion {
  id: UUID;
  version: string;           // "1.2.3"
  versionCode: number;       // 10203
  releaseType: 'stable' | 'beta' | 'release';
  releaseNotes?: string;
  downloadUrl: string;
  checksum: string;
  isMandatory: boolean;
  isEnabled: boolean;
  minCompatibleVersion?: string;
  publishedAt: DateTime;
  created_at: DateTime;
}
```

### 11.2 租户版本

```typescript
interface TenantVersion {
  tenantId: UUID;
  currentVersion: string;
  currentVersionCode: number;
  lastCheckAt?: DateTime;
  lastUpdateAt?: DateTime;
  autoUpdateEnabled: boolean;
  updateStatus: UpgradeStatus;
}
```

### 11.3 升级记录

```typescript
interface UpgradeRecord {
  id: UUID;
  tenantId: UUID;
  fromVersion: string;
  toVersion: string;
  status: 'pending' | 'running' | 'success' | 'failed' | 'rollback';
  backupId?: UUID;
  errorMsg?: string;
  rollbackNote?: string;
  startedAt: DateTime;
  completedAt?: DateTime;
  operatedBy?: UUID;
  operatedByName?: string;
}
```

## 12. 日志

### 12.1 登录日志

```typescript
interface LoginLog {
  id: UUID;
  tenantId?: UUID;
  userId?: UUID;
  loginTime: DateTime;
  loginIp: string;
  deviceType: 'pc_web' | 'mobile_h5' | 'mini_program' | 'ios_app' | 'android_app';
  deviceId?: string;
  os?: string;
  browser?: string;
  loginResult: 'success' | 'failed';
  failReason?: string;
}
```

### 12.2 操作日志

```typescript
interface OperationLog {
  id: UUID;
  tenantId: UUID;
  userId?: UUID;
  module: string;
  action: string;
  detail?: string;
  entityType?: string;
  entityId?: UUID;
  ip?: string;
  requestData?: Record<string, any>;
  status: 'success' | 'failed';
  errorMsg?: string;
  created_at: DateTime;
}
```
