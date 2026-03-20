# 租户系统数据库设计

## 1. 数据库概览

### 1.1 表清单

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| biz_accounts | 账户表 | id, tenant_id, username, password_hash |
| biz_roles | 角色表 | id, tenant_id, name, code |
| biz_account_roles | 账户角色关联 | id, account_id, role_id |
| biz_permissions | 权限表 | id, tenant_id, code, name |
| biz_role_permissions | 角色权限关联 | id, role_id, permission_id |
| biz_projects | 项目表 | id, tenant_id, name, code |
| biz_rules | 规则表 | id, tenant_id, project_id, rule_config |
| biz_tasks | 任务表 | id, tenant_id, project_id, auditor_id |
| biz_audit_logs | 审核日志表 | id, tenant_id, task_id, action |
| biz_third_party_configs | 三方配置表 | id, tenant_id, name, type, config |
| biz_third_party_logs | 三方日志表 | id, tenant_id, config_id, status |
| biz_login_logs | 登录日志表 | id, tenant_id, account_id, ip |
| biz_operation_logs | 操作日志表 | id, tenant_id, account_id, module |
| biz_backups | 备份记录表 | id, tenant_id, type, status, file_path |
| biz_backup_rules | 备份规则表 | id, tenant_id, name, cron_expr |
| biz_versions | 版本记录表 | id, tenant_id, version_code |

## 2. 表结构定义

### 2.1 账户表 (biz_accounts)

```sql
-- PostgreSQL
CREATE TABLE biz_accounts (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(50),
    phone VARCHAR(20),
    email VARCHAR(100),
    avatar VARCHAR(255),
    status VARCHAR(20) DEFAULT 'active',
    last_login_at TIMESTAMP,
    last_login_ip VARCHAR(50),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT,
    version INTEGER DEFAULT 0,
    CONSTRAINT uk_tenant_username UNIQUE (tenant_id, username),
    CONSTRAINT uk_tenant_email UNIQUE (tenant_id, email)
);

CREATE INDEX idx_account_tenant_id ON biz_accounts(tenant_id);
CREATE INDEX idx_account_username ON biz_accounts(username);
CREATE INDEX idx_account_is_deleted ON biz_accounts(is_deleted);

-- MySQL
CREATE TABLE biz_accounts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(50),
    phone VARCHAR(20),
    email VARCHAR(100),
    avatar VARCHAR(255),
    status VARCHAR(20) DEFAULT 'active',
    last_login_at DATETIME,
    last_login_ip VARCHAR(50),
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    version INT DEFAULT 0,
    UNIQUE KEY uk_tenant_username (tenant_id, username),
    UNIQUE KEY uk_tenant_email (tenant_id, email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_account_tenant_id ON biz_accounts(tenant_id);
CREATE INDEX idx_account_username ON biz_accounts(username);
```

### 2.2 角色表 (biz_roles)

```sql
-- PostgreSQL
CREATE TABLE biz_roles (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description VARCHAR(255),
    is_system SMALLINT DEFAULT 0,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT,
    CONSTRAINT uk_tenant_role_code UNIQUE (tenant_id, code)
);

CREATE INDEX idx_role_tenant_id ON biz_roles(tenant_id);

-- MySQL
CREATE TABLE biz_roles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description VARCHAR(255),
    is_system TINYINT(1) DEFAULT 0,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    UNIQUE KEY uk_tenant_role_code (tenant_id, code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_role_tenant_id ON biz_roles(tenant_id);
```

### 2.3 账户角色关联表 (biz_account_roles)

```sql
-- PostgreSQL
CREATE TABLE biz_account_roles (
    id BIGSERIAL PRIMARY KEY,
    account_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uk_account_role UNIQUE (account_id, role_id)
);

CREATE INDEX idx_ar_account_id ON biz_account_roles(account_id);
CREATE INDEX idx_ar_role_id ON biz_account_roles(role_id);

-- MySQL
CREATE TABLE biz_account_roles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_account_role (account_id, role_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_ar_account_id ON biz_account_roles(account_id);
CREATE INDEX idx_ar_role_id ON biz_account_roles(role_id);
```

### 2.4 权限表 (biz_permissions)

```sql
-- PostgreSQL
CREATE TABLE biz_permissions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    code VARCHAR(50) NOT NULL,
    name VARCHAR(50) NOT NULL,
    module VARCHAR(50),
    description VARCHAR(255),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    CONSTRAINT uk_tenant_permission_code UNIQUE (tenant_id, code)
);

CREATE INDEX idx_permission_tenant_id ON biz_permissions(tenant_id);
CREATE INDEX idx_permission_module ON biz_permissions(module);

-- MySQL
CREATE TABLE biz_permissions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    code VARCHAR(50) NOT NULL,
    name VARCHAR(50) NOT NULL,
    module VARCHAR(50),
    description VARCHAR(255),
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    UNIQUE KEY uk_tenant_permission_code (tenant_id, code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_permission_tenant_id ON biz_permissions(tenant_id);
CREATE INDEX idx_permission_module ON biz_permissions(module);
```

### 2.5 角色权限关联表 (biz_role_permissions)

```sql
-- PostgreSQL
CREATE TABLE biz_role_permissions (
    id BIGSERIAL PRIMARY KEY,
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uk_role_permission UNIQUE (role_id, permission_id)
);

CREATE INDEX idx_rp_role_id ON biz_role_permissions(role_id);
CREATE INDEX idx_rp_permission_id ON biz_role_permissions(permission_id);

-- MySQL
CREATE TABLE biz_role_permissions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_role_permission (role_id, permission_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_rp_role_id ON biz_role_permissions(role_id);
CREATE INDEX idx_rp_permission_id ON biz_role_permissions(permission_id);
```

### 2.6 项目表 (biz_projects)

```sql
-- PostgreSQL
CREATE TABLE biz_projects (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description TEXT,
    field_config JSONB,
    rule_config JSONB,
    need_manual_verify SMALLINT DEFAULT 1,
    task_assign_strategy VARCHAR(20) DEFAULT 'round_robin',
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT,
    version INTEGER DEFAULT 0,
    CONSTRAINT uk_tenant_project_code UNIQUE (tenant_id, code)
);

CREATE INDEX idx_project_tenant_id ON biz_projects(tenant_id);
CREATE INDEX idx_project_status ON biz_projects(status);
CREATE INDEX idx_project_is_deleted ON biz_projects(is_deleted);

COMMENT ON COLUMN biz_projects.field_config IS '字段配置，JSON格式：{fields: [{name, label, type, required, options}]}';
COMMENT ON COLUMN biz_projects.rule_config IS '规则配置，JSON格式：{rules: [rule_id1, rule_id2]}';

-- MySQL
CREATE TABLE biz_projects (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL,
    description TEXT,
    field_config JSON,
    rule_config JSON,
    need_manual_verify TINYINT(1) DEFAULT 1,
    task_assign_strategy VARCHAR(20) DEFAULT 'round_robin',
    status VARCHAR(20) DEFAULT 'active',
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    version INT DEFAULT 0,
    UNIQUE KEY uk_tenant_project_code (tenant_id, code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_project_tenant_id ON biz_projects(tenant_id);
CREATE INDEX idx_project_status ON biz_projects(status);
```

### 2.7 规则表 (biz_rules)

```sql
-- PostgreSQL
CREATE TABLE biz_rules (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    rule_config JSONB NOT NULL,
    priority INTEGER DEFAULT 0,
    is_active SMALLINT DEFAULT 1,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT,
    version INTEGER DEFAULT 0
);

CREATE INDEX idx_rule_tenant_id ON biz_rules(tenant_id);
CREATE INDEX idx_rule_project_id ON biz_rules(project_id);
CREATE INDEX idx_rule_is_active ON biz_rules(is_active);

COMMENT ON COLUMN biz_rules.rule_config IS '规则配置：{condition_groups: [{type: "all"/"any", conditions: [{field, operator, value}]}], pass_conclusion, reject_conclusion}';

-- MySQL
CREATE TABLE biz_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    rule_config JSON NOT NULL,
    priority INT DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    version INT DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_rule_tenant_id ON biz_rules(tenant_id);
CREATE INDEX idx_rule_project_id ON biz_rules(project_id);
```

### 2.8 任务表 (biz_tasks)

```sql
-- PostgreSQL
CREATE TABLE biz_tasks (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    auditor_id BIGINT,
    submit_data JSONB NOT NULL,
    rule_result JSONB,
    final_result VARCHAR(20) DEFAULT 'pending',
    audit_status VARCHAR(20) DEFAULT 'pending',
    reject_reason TEXT,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    audited_at TIMESTAMP,
    created_by BIGINT,
    version INTEGER DEFAULT 0
);

CREATE INDEX idx_task_tenant_id ON biz_tasks(tenant_id);
CREATE INDEX idx_task_project_id ON biz_tasks(project_id);
CREATE INDEX idx_task_auditor_id ON biz_tasks(auditor_id);
CREATE INDEX idx_task_audit_status ON biz_tasks(audit_status);
CREATE INDEX idx_task_created_at ON biz_tasks(created_at);
CREATE INDEX idx_task_final_result ON biz_tasks(final_result);

COMMENT ON COLUMN biz_tasks.submit_data IS '提交的审核数据';
COMMENT ON COLUMN biz_tasks.rule_result IS '规则执行结果';

-- MySQL
CREATE TABLE biz_tasks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    auditor_id BIGINT,
    submit_data JSON NOT NULL,
    rule_result JSON,
    final_result VARCHAR(20) DEFAULT 'pending',
    audit_status VARCHAR(20) DEFAULT 'pending',
    reject_reason TEXT,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    audited_at DATETIME,
    created_by BIGINT,
    version INT DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_task_tenant_id ON biz_tasks(tenant_id);
CREATE INDEX idx_task_project_id ON biz_tasks(project_id);
CREATE INDEX idx_task_auditor_id ON biz_tasks(auditor_id);
CREATE INDEX idx_task_audit_status ON biz_tasks(audit_status);
```

### 2.9 审核日志表 (biz_audit_logs)

```sql
-- PostgreSQL
CREATE TABLE biz_audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    task_id BIGINT NOT NULL,
    auditor_id BIGINT,
    action VARCHAR(20) NOT NULL,
    before_status VARCHAR(20),
    after_status VARCHAR(20),
    comment TEXT,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT
);

CREATE INDEX idx_audit_log_tenant_id ON biz_audit_logs(tenant_id);
CREATE INDEX idx_audit_log_task_id ON biz_audit_logs(task_id);
CREATE INDEX idx_audit_log_auditor_id ON biz_audit_logs(auditor_id);
CREATE INDEX idx_audit_log_created_at ON biz_audit_logs(created_at);

-- MySQL
CREATE TABLE biz_audit_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    task_id BIGINT NOT NULL,
    auditor_id BIGINT,
    action VARCHAR(20) NOT NULL,
    before_status VARCHAR(20),
    after_status VARCHAR(20),
    comment TEXT,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_audit_log_tenant_id ON biz_audit_logs(tenant_id);
CREATE INDEX idx_audit_log_task_id ON biz_audit_logs(task_id);
```

### 2.10 三方接口配置表 (biz_third_party_configs)

```sql
-- PostgreSQL
CREATE TABLE biz_third_party_configs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    config JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT,
    version INTEGER DEFAULT 0
);

CREATE INDEX idx_tpc_tenant_id ON biz_third_party_configs(tenant_id);
CREATE INDEX idx_tpc_type ON biz_third_party_configs(type);

COMMENT ON COLUMN biz_third_party_configs.config IS '配置：{url, method, headers, params, auth: {token_url, client_id, client_secret}}';

-- MySQL
CREATE TABLE biz_third_party_configs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,
    config JSON NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    version INT DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_tpc_tenant_id ON biz_third_party_configs(tenant_id);
```

### 2.11 三方接口日志表 (biz_third_party_logs)

```sql
-- PostgreSQL
CREATE TABLE biz_third_party_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    config_id BIGINT NOT NULL,
    request_data JSONB,
    response_data JSONB,
    status VARCHAR(20),
    error_message TEXT,
    duration INTEGER,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tpl_tenant_id ON biz_third_party_logs(tenant_id);
CREATE INDEX idx_tpl_config_id ON biz_third_party_logs(config_id);
CREATE INDEX idx_tpl_status ON biz_third_party_logs(status);
CREATE INDEX idx_tpl_created_at ON biz_third_party_logs(created_at);

-- MySQL
CREATE TABLE biz_third_party_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    config_id BIGINT NOT NULL,
    request_data JSON,
    response_data JSON,
    status VARCHAR(20),
    error_message TEXT,
    duration INT,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_tpl_tenant_id ON biz_third_party_logs(tenant_id);
CREATE INDEX idx_tpl_config_id ON biz_third_party_logs(config_id);
```

### 2.12 登录日志表 (biz_login_logs)

```sql
-- PostgreSQL
CREATE TABLE biz_login_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    account_id BIGINT,
    username VARCHAR(50),
    ip VARCHAR(50),
    device_type VARCHAR(20),
    device_id VARCHAR(100),
    os VARCHAR(50),
    browser VARCHAR(50),
    login_result VARCHAR(20),
    fail_reason VARCHAR(255),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_login_log_tenant_id ON biz_login_logs(tenant_id);
CREATE INDEX idx_login_log_account_id ON biz_login_logs(account_id);
CREATE INDEX idx_login_log_created_at ON biz_login_logs(created_at);

-- MySQL
CREATE TABLE biz_login_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    account_id BIGINT,
    username VARCHAR(50),
    ip VARCHAR(50),
    device_type VARCHAR(20),
    device_id VARCHAR(100),
    os VARCHAR(50),
    browser VARCHAR(50),
    login_result VARCHAR(20),
    fail_reason VARCHAR(255),
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_login_log_tenant_id ON biz_login_logs(tenant_id);
```

### 2.13 操作日志表 (biz_operation_logs)

```sql
-- PostgreSQL
CREATE TABLE biz_operation_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    account_id BIGINT,
    module VARCHAR(50),
    action VARCHAR(50),
    detail TEXT,
    entity_type VARCHAR(50),
    entity_id BIGINT,
    ip VARCHAR(50),
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_op_log_tenant_id ON biz_operation_logs(tenant_id);
CREATE INDEX idx_op_log_account_id ON biz_operation_logs(account_id);
CREATE INDEX idx_op_log_module ON biz_operation_logs(module);
CREATE INDEX idx_op_log_created_at ON biz_operation_logs(created_at);

-- MySQL
CREATE TABLE biz_operation_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    account_id BIGINT,
    module VARCHAR(50),
    action VARCHAR(50),
    detail TEXT,
    entity_type VARCHAR(50),
    entity_id BIGINT,
    ip VARCHAR(50),
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_op_log_tenant_id ON biz_operation_logs(tenant_id);
```

### 2.14 备份规则表 (biz_backup_rules)

```sql
-- PostgreSQL
CREATE TABLE biz_backup_rules (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    backup_type VARCHAR(20) NOT NULL,
    cron_expr VARCHAR(50),
    retention_days INTEGER DEFAULT 30,
    storage_type VARCHAR(20) DEFAULT 'local',
    storage_config JSONB,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    last_exec_at TIMESTAMP,
    next_exec_at TIMESTAMP,
    created_by BIGINT,
    updated_by BIGINT
);

CREATE INDEX idx_backup_rule_tenant_id ON biz_backup_rules(tenant_id);

-- MySQL
CREATE TABLE biz_backup_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    backup_type VARCHAR(20) NOT NULL,
    cron_expr VARCHAR(50),
    retention_days INT DEFAULT 30,
    storage_type VARCHAR(20) DEFAULT 'local',
    storage_config JSON,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    last_exec_at DATETIME,
    next_exec_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2.15 备份记录表 (biz_backups)

```sql
-- PostgreSQL
CREATE TABLE biz_backups (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    rule_id BIGINT,
    backup_type VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    file_path VARCHAR(255),
    file_size BIGINT,
    checksum VARCHAR(64),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_backup_tenant_id ON biz_backups(tenant_id);
CREATE INDEX idx_backup_rule_id ON biz_backups(rule_id);
CREATE INDEX idx_backup_status ON biz_backups(status);
CREATE INDEX idx_backup_created_at ON biz_backups(created_at);

-- MySQL
CREATE TABLE biz_backups (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    rule_id BIGINT,
    backup_type VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    file_path VARCHAR(255),
    file_size BIGINT,
    checksum VARCHAR(64),
    started_at DATETIME,
    completed_at DATETIME,
    error_message TEXT,
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2.16 版本记录表 (biz_versions)

```sql
-- PostgreSQL
CREATE TABLE biz_versions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    version_code VARCHAR(20) NOT NULL,
    version_name VARCHAR(50),
    description TEXT,
    upgrade_status VARCHAR(20) DEFAULT 'idle',
    is_deleted SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    upgraded_at TIMESTAMP
);

CREATE INDEX idx_version_tenant_id ON biz_versions(tenant_id);

-- MySQL
CREATE TABLE biz_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id BIGINT NOT NULL,
    version_code VARCHAR(20) NOT NULL,
    version_name VARCHAR(50),
    description TEXT,
    upgrade_status VARCHAR(20) DEFAULT 'idle',
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    upgraded_at DATETIME
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 3. ER 关系图

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   biz_roles │────<│biz_account_ │>────│biz_accounts │
│             │     │   roles     │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
       │                                       │
       │                                       │
       ▼                                       │
┌─────────────┐     ┌─────────────┐            │
│ biz_role_   │────<│ biz_permis- │            │
│ permissions │     │   sions     │            │
└─────────────┘     └─────────────┘            │
                                                │
                                                │
       ┌────────────────────────────────────────┤
       │                                        │
       ▼                                        ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ biz_projects│1──M<│  biz_rules  │     │ biz_tasks   │
│             │     │             │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                                │
                                                ▼
                                         ┌─────────────┐
                                         │biz_audit_   │
                                         │   logs      │
                                         └─────────────┘

┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│biz_third_   │────<│biz_third_party_ │
│party_configs│     │     logs        │
└─────────────┘     └─────────────────┘

┌─────────────┐     ┌─────────────┐
│biz_backup_ │1──M<│  biz_        │
│    rules    │     │  backups    │
└─────────────┘     └─────────────┘

┌─────────────┐
│biz_versions │
│             │
└─────────────┘
```

## 4. 初始数据

### 4.1 内置角色

```sql
-- 租户最高管理员
INSERT INTO biz_roles (tenant_id, name, code, description, is_system)
VALUES (1, '租户最高管理员', 'TENANT_SUPER_ADMIN', '租户创始人，拥有租户全部权限', 1);

-- 普通管理员
INSERT INTO biz_roles (tenant_id, name, code, description, is_system)
VALUES (1, '普通管理员', 'ADMIN', '项目管理、规则配置、查看统计数据', 1);

-- 审核员
INSERT INTO biz_roles (tenant_id, name, code, description, is_system)
VALUES (1, '审核员', 'AUDITOR', '执行数据审核任务，执行人工校验', 1);

-- 普通员工
INSERT INTO biz_roles (tenant_id, name, code, description, is_system)
VALUES (1, '普通员工', 'STAFF', '查看数据，无审核权限', 1);
```

### 4.2 内置权限

```sql
-- 租户管理权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'TENANT_VIEW', '查看租户信息', '租户管理'),
(1, 'TENANT_EDIT', '编辑租户信息', '租户管理'),
(1, 'TENANT_CONFIG', '配置租户参数', '租户管理'),
(1, 'TENANT_APP_SECRET', '管理APPID/APPSECRET', '租户管理');

-- 账户管理权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'ACCOUNT_VIEW', '查看账户列表', '账户管理'),
(1, 'ACCOUNT_CREATE', '创建账户', '账户管理'),
(1, 'ACCOUNT_EDIT', '编辑账户', '账户管理'),
(1, 'ACCOUNT_DELETE', '删除账户', '账户管理'),
(1, 'ACCOUNT_ROLE', '分配角色', '账户管理'),
(1, 'PROFILE_EDIT', '修改个人资料', '账户管理'),
(1, 'PASSWORD_CHANGE', '修改密码', '账户管理');

-- 项目管理权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'PROJECT_VIEW', '查看项目', '项目管理'),
(1, 'PROJECT_CREATE', '创建项目', '项目管理'),
(1, 'PROJECT_EDIT', '编辑项目', '项目管理'),
(1, 'PROJECT_DELETE', '删除项目', '项目管理'),
(1, 'PROJECT_CONFIG', '配置项目规则', '项目管理');

-- 规则管理权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'RULE_VIEW', '查看规则', '规则管理'),
(1, 'RULE_CREATE', '创建规则', '规则管理'),
(1, 'RULE_EDIT', '编辑规则', '规则管理'),
(1, 'RULE_DELETE', '删除规则', '规则管理');

-- 审核任务权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'AUDIT_TASK_VIEW_SELF', '查看自己的任务', '任务审核'),
(1, 'AUDIT_TASK_VIEW_ALL', '查看所有任务', '任务审核'),
(1, 'AUDIT_EXECUTE', '执行审核', '任务审核'),
(1, 'AUDIT_RECHECK', '执行二次校验', '任务审核');

-- 三方接口权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'THIRD_PARTY_VIEW', '查看接口配置', '三方接口'),
(1, 'THIRD_PARTY_CONFIG', '配置接口参数', '三方接口'),
(1, 'THIRD_PARTY_LOG', '查看调用日志', '三方接口');

-- 统计报表权限
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'STAT_SELF', '查看个人统计', '统计报表'),
(1, 'STAT_TENANT', '查看租户统计', '统计报表'),
(1, 'STAT_PROJECT', '查看项目统计', '统计报表');

-- 版本管理权限（独立部署）
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'SYSTEM_VERSION_VIEW', '查看版本信息', '系统版本'),
(1, 'SYSTEM_VERSION_CHECK', '检查更新', '系统版本'),
(1, 'SYSTEM_UPGRADE', '执行升级', '系统版本'),
(1, 'SYSTEM_ROLLBACK', '回滚版本', '系统版本');

-- 备份管理权限（独立部署）
INSERT INTO biz_permissions (tenant_id, code, name, module) VALUES
(1, 'BACKUP_RULE_VIEW', '查看备份规则', '备份管理'),
(1, 'BACKUP_RULE_CREATE', '创建备份规则', '备份管理'),
(1, 'BACKUP_RULE_EDIT', '编辑备份规则', '备份管理'),
(1, 'BACKUP_RULE_DELETE', '删除备份规则', '备份管理'),
(1, 'BACKUP_EXECUTE', '执行备份', '备份管理'),
(1, 'BACKUP_RESTORE', '恢复备份', '备份管理'),
(1, 'BACKUP_VIEW', '查看备份记录', '备份管理');
```
