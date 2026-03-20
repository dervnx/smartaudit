# 租户数据库设计

## 1. 数据库架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      租户数据库                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │   user                  # 用户表                        │    │
│  │   role                  # 角色表                        │    │
│  │   permission            # 权限表                        │    │
│  │   user_role             # 用户角色关联表                │    │
│  │   role_permission       # 角色权限关联表                │    │
│  │   project               # 项目表                       │    │
│  │   rule                  # 规则表                       │    │
│  │   audit_data            # 审核数据表                   │    │
│  │   task                  # 任务表                       │    │
│  │   third_party_config    # 三方接口配置表               │    │
│  │   third_party_log       # 三方接口日志表               │    │
│  │   backup_rule           # 备份规则表                   │    │
│  │   backup_record         # 备份记录表                   │    │
│  │   tenant_version        # 租户版本表                   │    │
│  │   login_log             # 登录日志表                   │    │
│  │   operation_log         # 操作日志表                   │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## 2. ER 图

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    user       │       │    role       │       │  permission   │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │──┐    │ id (PK)      │       │ id (PK)      │
│ tenant_id(FK)│  │    │ tenant_id(FK)│───────┤ code         │
│ username     │  │    │ name         │       │ name         │
│ password_hash│  │    │ code         │       │ module       │
│ name         │  │    │ description  │       │ description  │
│ email        │  │    │ is_system    │       └──────────────┘
│ phone        │  │    └──────┬───────┘              ▲
│ status       │  │           │              ┌───────┴───────┐
│ extra_perms  │  │           │              │role_permission│
└──────┬───────┘  │           │              ├──────────────┤
       │          │           │              │ role_id(FK)  │
       │          │           │              │ perm_id(FK)  │
       ▼          │           ▼              │ is_granted   │
┌────────────┐    │    ┌────────────┐        └──────────────┘
│  user_role  │    │    │ user_role  │
├────────────┤    │    ├────────────┤
│ user_id(FK)│────┘    │ user_id(FK)│──────┐
│ role_id(FK)│─────────┘ role_id(FK)│      │
└────────────┘                       │      │
                                      │      │
┌──────────────┐       ┌──────────────┐ │ ┌──────────────┐
│   project    │       │    rule       │ │ │    audit_data│
├──────────────┤       ├──────────────┤ │ ├──────────────┤
│ id (PK)      │       │ id (PK)      │ │ │ id (PK)      │
│ tenant_id(FK)│──────►│ tenant_id(FK)│─┘ │ tenant_id(FK)│
│ name         │       │ name         │   │ project_id(FK)│
│ code         │       │ conditions   │   │ input_data   │
│ fields_def  │       │ result_pass  │   │ rule_result  │
│ rule_ids    │       │ result_reject│   │ auto_result │
│ status       │       │ priority     │   │ final_result│
└──────┬───────┘       └──────────────┘   │ auditor_id  │
       │                                      │ need_recheck│
       │                                      └──────┬───────┘
       │                                             │
       ▼                                    ┌────────┴────────┐
┌──────────────┐                            │      task        │
│ third_party  │                            ├─────────────────┤
│ _config      │                            │ id (PK)         │
├──────────────┤                            │ tenant_id(FK)   │
│ id (PK)      │                            │ auditor_id(FK)  │
│ tenant_id(FK)│                            │ data_id (FK)   │
│ name         │                            │ type            │
│ type         │                            │ status          │
│ url          │                            │ result          │
│ auth_config  │                            │ assigned_at     │
│ timeout      │                            │ started_at      │
│ status       │                            │ completed_at    │
└──────────────┘                            └─────────────────┘

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ backup_rule  │       │backup_record │       │tenant_version│
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │──┐    │ id (PK)      │       │ id (PK)      │
│ tenant_id(FK)│  │    │ tenant_id(FK)│       │ tenant_id(FK)│
│ name         │  │    │ rule_id(FK)  │       │ cur_version  │
│ backup_type  │  │    │ backup_type │       │ last_check   │
│ schedule_cron│  │    │ status      │       │ last_update  │
│ retention    │  │    │ file_path   │       │ auto_update  │
│ is_enabled   │  │    │ file_size   │       │ update_status│
└──────────────┘  │    │ checksum    │       └──────────────┘
                  │    │ started_at   │
                  │    │ completed_at │
                  └────┤ error_msg    │
                       └──────────────┘
```

## 3. 表结构

### 3.1 user - 用户表

```sql
CREATE TABLE "user" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    email_verified BOOLEAN DEFAULT FALSE,
    phone VARCHAR(20),
    phone_verified BOOLEAN DEFAULT FALSE,
    avatar_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'active',
    extra_permissions JSONB DEFAULT '[]',
    last_login_at TIMESTAMP,
    last_login_ip VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, username)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_username ON "user"(username);
CREATE INDEX idx_user_status ON "user"(status);
```

### 3.2 role - 角色表

```sql
CREATE TABLE role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(50) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description VARCHAR(200),
    is_system BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_role_tenant ON role(tenant_id);

-- 初始角色数据
INSERT INTO role (id, tenant_id, name, code, description, is_system, sort_order) VALUES
('00000000-0000-0000-0000-000000000001', NULL, '租户最高管理员', 'TENANT_SUPER_ADMIN', '租户创始人，拥有全部权限', TRUE, 1),
('00000000-0000-0000-0000-000000000002', NULL, '普通管理员', 'ADMIN', '项目管理、规则配置', FALSE, 2),
('00000000-0000-0000-0000-000000000003', NULL, '审核员', 'AUDITOR', '执行审核任务', FALSE, 3),
('00000000-0000-0000-0000-000000000004', NULL, '普通员工', 'STAFF', '查看数据', FALSE, 4);
```

### 3.3 permission - 权限表

```sql
CREATE TABLE permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(50) NOT NULL,
    module VARCHAR(50) NOT NULL,
    description VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 权限数据
INSERT INTO permission (code, name, module, description) VALUES
-- 账户权限
('ACCOUNT_VIEW', '查看账户', '账户管理', '查看账户列表'),
('ACCOUNT_CREATE', '创建账户', '账户管理', '创建新账户'),
('ACCOUNT_EDIT', '编辑账户', '账户管理', '编辑账户信息'),
('ACCOUNT_DELETE', '删除账户', '账户管理', '删除账户'),
('ACCOUNT_ROLE', '分配角色', '账户管理', '分配账户角色'),
('ACCOUNT_PERMISSION', '配置权限', '账户管理', '精细化权限配置'),
('PROFILE_EDIT', '修改个人资料', '个人中心', '修改个人资料'),
('PASSWORD_CHANGE', '修改密码', '个人中心', '修改密码'),
-- 项目权限
('PROJECT_VIEW', '查看项目', '项目管理', '查看项目列表'),
('PROJECT_CREATE', '创建项目', '项目管理', '创建新项目'),
('PROJECT_EDIT', '编辑项目', '项目管理', '编辑项目'),
('PROJECT_DELETE', '删除项目', '项目管理', '删除项目'),
('PROJECT_CONFIG', '配置项目', '项目管理', '配置项目规则'),
-- 规则权限
('RULE_VIEW', '查看规则', '规则管理', '查看规则列表'),
('RULE_CREATE', '创建规则', '规则管理', '创建新规则'),
('RULE_EDIT', '编辑规则', '规则管理', '编辑规则'),
('RULE_DELETE', '删除规则', '规则管理', '删除规则'),
-- 审核权限
('AUDIT_TASK_VIEW_SELF', '查看自己的任务', '任务审核', '查看个人任务'),
('AUDIT_TASK_VIEW_ALL', '查看所有任务', '任务审核', '查看全部任务'),
('AUDIT_EXECUTE', '执行审核', '任务审核', '执行审核操作'),
('AUDIT_RECHECK', '二次校验', '任务审核', '执行二次校验'),
-- 三方接口权限
('THIRD_PARTY_VIEW', '查看接口', '三方接口', '查看接口配置'),
('THIRD_PARTY_CONFIG', '配置接口', '三方接口', '配置接口参数'),
('THIRD_PARTY_LOG', '查看日志', '三方接口', '查看调用日志'),
-- 统计权限
('STAT_SELF', '个人统计', '统计报表', '查看个人统计'),
('STAT_TENANT', '租户统计', '统计报表', '查看租户统计'),
('STAT_PROJECT', '项目统计', '统计报表', '查看项目统计'),
-- 系统权限
('SYSTEM_VERSION', '版本管理', '系统管理', '版本查看'),
('BACKUP_RULE', '备份规则', '备份管理', '备份规则配置'),
('BACKUP_EXECUTE', '执行备份', '备份管理', '手动执行备份'),
('BACKUP_RESTORE', '恢复备份', '备份管理', '恢复备份数据');
```

### 3.4 user_role - 用户角色关联表

```sql
CREATE TABLE user_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, role_id)
);

CREATE INDEX idx_user_role_user ON user_role(user_id);
CREATE INDEX idx_user_role_role ON user_role(role_id);
```

### 3.5 role_permission - 角色权限关联表

```sql
CREATE TABLE role_permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    is_granted BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, permission_id)
);

CREATE INDEX idx_role_perm_role ON role_permission(role_id);
CREATE INDEX idx_role_perm_perm ON role_permission(permission_id);
```

### 3.6 project - 项目表

```sql
CREATE TABLE project (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description TEXT,
    fields_def JSONB NOT NULL DEFAULT '[]',
    rule_ids JSONB DEFAULT '[]',
    need_manual_review BOOLEAN DEFAULT FALSE,
    task_assign_type VARCHAR(20) DEFAULT 'round_robin',
    status VARCHAR(20) DEFAULT 'active',
    created_by UUID REFERENCES "user"(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_project_tenant ON project(tenant_id);
CREATE INDEX idx_project_status ON project(status);
```

### 3.7 rule - 规则表

```sql
CREATE TABLE rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    conditions JSONB NOT NULL,
    result_pass TEXT,
    result_reject TEXT,
    result_suggest TEXT,
    priority INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active',
    created_by UUID REFERENCES "user"(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_rule_tenant ON rule(tenant_id);
CREATE INDEX idx_rule_status ON rule(status);
CREATE INDEX idx_rule_priority ON rule(priority DESC);
```

### 3.8 audit_data - 审核数据表

```sql
CREATE TABLE audit_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    input_data JSONB NOT NULL,
    rule_result JSONB,
    auto_result VARCHAR(20),
    final_result VARCHAR(20),
    auditor_id UUID REFERENCES "user"(id),
    audit_time TIMESTAMP,
    audit_note TEXT,
    need_recheck BOOLEAN DEFAULT FALSE,
    recheck_auditor_id UUID REFERENCES "user"(id),
    recheck_time TIMESTAMP,
    recheck_result VARCHAR(20),
    recheck_note TEXT,
    status VARCHAR(20) DEFAULT 'pending',
    priority INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_tenant ON audit_data(tenant_id);
CREATE INDEX idx_audit_project ON audit_data(project_id);
CREATE INDEX idx_audit_status ON audit_data(status);
CREATE INDEX idx_audit_auditor ON audit_data(auditor_id);
CREATE INDEX idx_audit_created ON audit_data(created_at);
CREATE INDEX idx_audit_need_recheck ON audit_data(need_recheck);
```

### 3.9 task - 任务表

```sql
CREATE TABLE task (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    auditor_id UUID REFERENCES "user"(id) ON DELETE SET NULL,
    data_id UUID NOT NULL REFERENCES audit_data(id) ON DELETE CASCADE,
    type VARCHAR(20) DEFAULT 'review',
    status VARCHAR(20) DEFAULT 'pending',
    result VARCHAR(20),
    note TEXT,
    assigned_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_task_tenant ON task(tenant_id);
CREATE INDEX idx_task_auditor ON task(auditor_id);
CREATE INDEX idx_task_data ON task(data_id);
CREATE INDEX idx_task_status ON task(status);
CREATE INDEX idx_task_type ON task(type);
```

### 3.10 third_party_config - 三方接口配置表

```sql
CREATE TABLE third_party_config (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    url VARCHAR(500) NOT NULL,
    method VARCHAR(10) DEFAULT 'POST',
    headers JSONB DEFAULT '{}',
    params_template JSONB DEFAULT '{}',
    auth_type VARCHAR(20) DEFAULT 'none',
    auth_config JSONB DEFAULT '{}',
    timeout INT DEFAULT 30,
    retry_count INT DEFAULT 3,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_third_party_tenant ON third_party_config(tenant_id);
CREATE INDEX idx_third_party_type ON third_party_config(type);
```

### 3.11 third_party_log - 三方接口日志表

```sql
CREATE TABLE third_party_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    config_id UUID REFERENCES third_party_config(id) ON DELETE SET NULL,
    user_id UUID REFERENCES "user"(id) ON DELETE SET NULL,
    request_time TIMESTAMP NOT NULL,
    request_url VARCHAR(500),
    request_method VARCHAR(10),
    request_data JSONB,
    response_data JSONB,
    response_status VARCHAR(20),
    error_msg TEXT,
    duration_ms INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_third_party_log_tenant ON third_party_log(tenant_id);
CREATE INDEX idx_third_party_log_config ON third_party_log(config_id);
CREATE INDEX idx_third_party_log_time ON third_party_log(request_time);
```

### 3.12 backup_rule - 备份规则表

```sql
CREATE TABLE backup_rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    backup_type VARCHAR(20) DEFAULT 'full',
    schedule_cron VARCHAR(50),
    retention_days INT DEFAULT 30,
    storage_location VARCHAR(20) DEFAULT 'local',
    storage_config JSONB DEFAULT '{}',
    is_enabled BOOLEAN DEFAULT TRUE,
    last_backup_at TIMESTAMP,
    next_backup_at TIMESTAMP,
    created_by UUID REFERENCES "user"(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_backup_rule_tenant ON backup_rule(tenant_id);
CREATE INDEX idx_backup_rule_enabled ON backup_rule(is_enabled);
```

### 3.13 backup_record - 备份记录表

```sql
CREATE TABLE backup_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    rule_id UUID REFERENCES backup_rule(id) ON DELETE SET NULL,
    backup_type VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    file_path VARCHAR(500),
    file_size BIGINT,
    checksum VARCHAR(64),
    error_msg TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_backup_record_tenant ON backup_record(tenant_id);
CREATE INDEX idx_backup_record_rule ON backup_record(rule_id);
CREATE INDEX idx_backup_record_status ON backup_record(status);
CREATE INDEX idx_backup_record_created ON backup_record(created_at DESC);
```

### 3.14 tenant_version - 租户版本表

```sql
CREATE TABLE tenant_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID UNIQUE NOT NULL,
    current_version VARCHAR(50) NOT NULL,
    current_version_code INT NOT NULL,
    last_check_at TIMESTAMP,
    last_update_at TIMESTAMP,
    auto_update_enabled BOOLEAN DEFAULT TRUE,
    update_status VARCHAR(20) DEFAULT 'idle',
    updated_by UUID REFERENCES "user"(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenant_version_tenant ON tenant_version(tenant_id);
```

### 3.15 login_log - 登录日志表

```sql
CREATE TABLE login_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    user_id UUID REFERENCES "user"(id) ON DELETE SET NULL,
    login_time TIMESTAMP NOT NULL,
    login_ip VARCHAR(50),
    device_type VARCHAR(20),
    device_id VARCHAR(100),
    os VARCHAR(50),
    browser VARCHAR(50),
    login_result VARCHAR(20) NOT NULL,
    fail_reason VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_login_log_tenant ON login_log(tenant_id);
CREATE INDEX idx_login_log_user ON login_log(user_id);
CREATE INDEX idx_login_log_time ON login_log(login_time);
```

### 3.16 operation_log - 操作日志表

```sql
CREATE TABLE operation_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    user_id UUID REFERENCES "user"(id) ON DELETE SET NULL,
    module VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    detail TEXT,
    entity_type VARCHAR(50),
    entity_id UUID,
    ip VARCHAR(50),
    request_data JSONB,
    status VARCHAR(20) DEFAULT 'success',
    error_msg TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_operation_log_tenant ON operation_log(tenant_id);
CREATE INDEX idx_operation_log_user ON operation_log(user_id);
CREATE INDEX idx_operation_log_module ON operation_log(module);
CREATE INDEX idx_operation_log_time ON operation_log(created_at);
```

## 4. 索引规划

| 表名 | 索引名 | 字段 | 类型 |
|------|--------|------|------|
| user | idx_user_tenant | tenant_id | 普通 |
| user | idx_user_username | username | 唯一(租户内) |
| user | idx_user_status | status | 普通 |
| role | idx_role_tenant | tenant_id | 普通 |
| permission | idx_permission_code | code | 唯一 |
| project | idx_project_tenant | tenant_id | 普通 |
| project | idx_project_status | status | 普通 |
| rule | idx_rule_tenant | tenant_id | 普通 |
| audit_data | idx_audit_tenant | tenant_id | 普通 |
| audit_data | idx_audit_project | project_id | 普通 |
| audit_data | idx_audit_status | status | 普通 |
| audit_data | idx_audit_auditor | auditor_id | 普通 |
| task | idx_task_tenant | tenant_id | 普通 |
| task | idx_task_auditor | auditor_id | 普通 |
| task | idx_task_status | status | 普通 |
| backup_record | idx_backup_record_tenant | tenant_id | 普通 |
| backup_record | idx_backup_record_status | status | 普通 |
