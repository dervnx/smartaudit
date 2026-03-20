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
    page_size: number;     // 每页数量
    total_pages: number;  // 总页数
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
  request_id?: string;
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
  tenant_id?: UUID;        // 租户ID (租户用户)
  roles: Role[];
  permissions: string[];  // 权限code列表
  extra_permissions?: string[];  // 额外权限
  last_login_at?: DateTime;
  last_login_ip?: string;
}
```

### 3.3 角色

```typescript
interface Role {
  id: UUID;
  name: string;          // 角色名称
  code: string;          // 角色代码
  description?: string;
  is_system: boolean;     // 是否系统内置
  permissions: string[]; // 权限列表
  user_count?: number;
}
```

### 3.4 登录请求/响应

```typescript
interface LoginRequest {
  username: string;
  password: string;
  captcha?: string;
  captcha_key?: string;
}

interface LoginResponse {
  access_token: string;
  refresh_token: string;
  expires_in: number;
  token_type: 'Bearer';
  user: User;
}
```

### 3.5 修改密码请求

```typescript
interface ChangePasswordRequest {
  old_password: string;
  new_password: string;
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
  contact_name: string;
  contact_email: string;
  contact_phone: string;
  address?: string;
  logo_url?: string;
  app_id: string;
  package_type: 'basic' | 'professional' | 'enterprise';
  deploy_mode: 'saas' | 'dedicated';
  service_start_date: Date;
  service_end_date: Date;
  status: 'active' | 'inactive' | 'expired';
  created_at: DateTime;
  updated_at: DateTime;
}
```

### 4.2 租户统计

```typescript
interface TenantStatistics {
  user_count: number;
  project_count: number;
  total_audit_data: number;
  month_audit_data: number;
  today_audit_data: number;
  pending_audit_data: number;
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
  max_length?: number;
  min_length?: number;
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
  | 'multi_select'
  | 'file'
  | 'image'
  | 'region'
  | 'tags'
  | 'id_card'
  | 'phone'
  | 'email';
```

### 5.2 项目

```typescript
interface Project {
  id: UUID;
  tenant_id: UUID;
  name: string;
  code: string;
  description?: string;
  fields_def: FieldDefinition[];
  rule_ids: UUID[];
  need_manual_review: boolean;
  task_assign_type: TaskAssignType;
  status: 'active' | 'inactive' | 'archived';
  created_by?: UUID;
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
  total_data: number;
  pending_data: number;
  in_progress_data: number;
  passed_data: number;
  rejected_data: number;
  rechecking_data: number;
  progress_rate: number;
  auditor_workload: AuditorWorkload[];
  reject_reason_distribution: ReasonDistribution[];
  avg_duration: number;  // 秒
}

interface AuditorWorkload {
  auditor_id: UUID;
  auditor_name: string;
  completed_count: number;
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
  | 'id_cardValid'
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
  tenant_id: UUID;
  name: string;
  description?: string;
  conditions: ConditionGroup;
  result_pass?: string;
  result_reject?: string;
  result_suggest?: string;
  priority: number;
  status: 'active' | 'inactive';
  created_by?: UUID;
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
  tenant_id: UUID;
  project_id: UUID;
  project_name?: string;
  input_data: Record<string, any>;         // 原始提交数据
  source: 'manual' | 'api' | 'import';     // 数据来源
  rule_result?: RuleResult;                // 规则执行结果
  auto_result?: 'pass' | 'reject' | 'pending';  // 自动审核结果
  final_result?: 'pass' | 'reject' | 'recheck';  // 最终审核结果
  audit_mode: 'auto' | 'manual';          // 审核模式：自动/人工
  auditor_id?: UUID;                       // 审核员ID
  auditor_name?: string;
  audit_time?: DateTime;
  audit_note?: string;
  need_recheck: boolean;                   // 是否需要人工复核
  recheck_auditor_id?: UUID;               // 复核员ID
  recheck_time?: DateTime;
  recheck_result?: 'pass' | 'reject';
  recheck_note?: string;
  manual_intervention?: boolean;           // 是否有人工介入
  manual_operator_id?: UUID;                // 人工操作人ID
  manual_operator_name?: string;
  manual_action?: 'modify' | 'recheck' | 'override';  // 人工操作类型
  manual_action_time?: DateTime;
  status: AuditStatus;
  priority: number;
  created_at: DateTime;
}
```

### 7.2 审核操作日志

```typescript
interface AuditOperationLog {
  id: UUID;
  tenant_id: UUID;
  audit_data_id: UUID;                      // 关联的审核数据ID
  action: AuditAction;                     // 操作类型
  operator_type: 'system' | 'auditor' | 'manual';  // 操作者类型
  operator_id?: UUID;                      // 操作人ID
  operator_name?: string;
  before_state?: AuditStatus;               // 操作前状态
  after_state?: AuditStatus;                // 操作后状态
  detail?: string;                          // 操作详情
  request_data?: Record<string, any>;      // 请求数据（人工介入时）
  created_at: DateTime;
}

type AuditAction =
  | 'submit'           // 提交审核
  | 'auto_pass'        // 自动通过
  | 'auto_reject'      // 自动拒绝
  | 'assign_auditor'   // 分配审核员
  | 'audit_pass'       // 审核通过
  | 'audit_reject'     // 审核拒绝
  | 'recheck'          // 二次复核
  | 'manual_modify'    // 人工修改数据
  | 'manual_override'   // 人工强制终态
  | 'recheck_override'; // 复核覆盖
```

## 8. 任务

### 8.1 任务

```typescript
interface Task {
  id: UUID;
  tenant_id: UUID;
  auditor_id?: UUID;
  auditor_name?: string;
  data_id: UUID;
  data?: AuditData;
  type: 'review' | 'recheck';
  status: TaskStatus;
  result?: 'pass' | 'reject';
  note?: string;
  assigned_at?: DateTime;
  started_at?: DateTime;
  completed_at?: DateTime;
  created_at: DateTime;
}
```

### 8.2 任务统计

```typescript
interface TaskStatistics {
  pending_count: number;
  recheck_count: number;
  today_completed: number;
  week_completed: number;
  month_completed: number;
  year_completed: number;
  total_completed: number;
  avg_duration: number;  // 秒
}
```

## 9. 三方接口

### 9.1 接口配置

```typescript
interface ThirdPartyConfig {
  id: UUID;
  tenant_id: UUID;
  name: string;
  type: ThirdPartyType;
  url: string;
  method: 'GET' | 'POST';
  headers?: Record<string, string>;
  params_template?: Record<string, any>;
  auth_type: 'none' | 'bearer' | 'oauth2' | 'api_key';
  auth_config?: AuthConfig;
  timeout: number;
  retry_count: number;
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
  token_url?: string;
  client_id?: string;
  client_secret?: string;
  api_key?: string;
}
```

### 9.2 接口日志

```typescript
interface ThirdPartyLog {
  id: UUID;
  tenant_id: UUID;
  config_id: UUID;
  config_name?: string;
  user_id?: UUID;
  request_time: DateTime;
  request_url: string;
  request_method: string;
  request_data?: Record<string, any>;
  response_data?: Record<string, any>;
  response_status: 'success' | 'failed';
  error_msg?: string;
  duration_ms: number;
}
```

### 9.3 接口统计

```typescript
interface ThirdPartyStatistics {
  total_calls: number;
  month_calls: number;
  today_calls: number;
  success_rate: number;
  avg_response_time: number;
  call_ranking: CallRanking[];
}

interface CallRanking {
  config_id: UUID;
  config_name: string;
  call_count: number;
}
```

## 10. 备份

### 10.1 备份规则

```typescript
interface BackupRule {
  id: UUID;
  tenant_id: UUID;
  name: string;
  backup_type: BackupType;
  schedule_cron: string;
  retention_days: number;
  storage_location: 'local' | 's3' | 'oss';
  storage_config?: StorageConfig;
  is_enabled: boolean;
  last_backup_at?: DateTime;
  next_backup_at?: DateTime;
  created_by?: UUID;
  created_at: DateTime;
}

interface StorageConfig {
  bucket?: string;
  endpoint?: string;
  access_key?: string;
  secret_key?: string;
  region?: string;
}
```

### 10.2 备份记录

```typescript
interface BackupRecord {
  id: UUID;
  tenant_id: UUID;
  rule_id?: UUID;
  backup_type: BackupType;
  status: BackupStatus;
  started_at: DateTime;
  completed_at?: DateTime;
  file_path?: string;
  file_size?: number;      // 字节
  checksum?: string;
  error_msg?: string;
  created_at: DateTime;
}
```

## 11. 版本

### 11.1 系统版本

```typescript
interface SystemVersion {
  id: UUID;
  version: string;           // "1.2.3"
  version_code: number;       // 10203
  release_type: 'stable' | 'beta' | 'release';
  release_notes?: string;
  download_url: string;
  checksum: string;
  is_mandatory: boolean;
  is_enabled: boolean;
  min_compatible_version?: string;
  published_at: DateTime;
  created_at: DateTime;
}
```

### 11.2 租户版本

```typescript
interface TenantVersion {
  tenant_id: UUID;
  current_version: string;
  current_version_code: number;
  last_check_at?: DateTime;
  last_update_at?: DateTime;
  auto_update_enabled: boolean;
  update_status: UpgradeStatus;
}
```

### 11.3 升级记录

```typescript
interface UpgradeRecord {
  id: UUID;
  tenant_id: UUID;
  from_version: string;
  to_version: string;
  status: 'pending' | 'running' | 'success' | 'failed' | 'rollback';
  backup_id?: UUID;
  error_msg?: string;
  rollback_note?: string;
  started_at: DateTime;
  completed_at?: DateTime;
  operated_by?: UUID;
  operated_by_name?: string;
}
```

## 12. 日志

### 12.1 登录日志

```typescript
interface LoginLog {
  id: UUID;
  tenant_id?: UUID;
  user_id?: UUID;
  login_time: DateTime;
  login_ip: string;
  device_type: 'pc_web' | 'mobile_h5' | 'mini_program' | 'ios_app' | 'android_app';
  device_id?: string;
  os?: string;
  browser?: string;
  login_result: 'success' | 'failed';
  fail_reason?: string;
}
```

### 12.2 操作日志

```typescript
interface OperationLog {
  id: UUID;
  tenant_id: UUID;
  user_id?: UUID;
  module: string;
  action: string;
  detail?: string;
  entity_type?: string;
  entity_id?: UUID;
  ip?: string;
  request_data?: Record<string, any>;
  status: 'success' | 'failed';
  error_msg?: string;
  created_at: DateTime;
}
```
