# SaaS 数据库设计

## 1. 数据库架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      SaaS 数据库                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │   saas_admin          # 平台管理员表                     │    │
│  │   tenant              # 租户表                          │    │
│  │   system_version      # 系统版本表                       │    │
│  │   upgrade_record      # 升级记录表                       │    │
│  │   saas_config         # 全局配置表                       │    │
│  │   login_log           # 登录日志表                       │    │
│  │   operation_log       # 操作日志表                       │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## 2. ER 图

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  saas_admin  │       │    tenant    │       │system_version│
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │       │ id (PK)      │
│ username     │       │ name         │       │ version      │
│ password     │       │ code         │       │ version_code │
│ name         │       │ app_id       │       │ release_type │
│ email        │       │ app_secret   │       │ release_notes│
│ phone        │       │ package_type │       │ download_url │
│ avatar_url   │       │ deploy_mode  │       │ is_mandatory │
│ role         │       │ status       │       │ is_enabled   │
│ status       │       │ service_dates│       └──────┬───────┘
│ last_login   │       └──────┬───────┘              │
│ created_at   │              │              ┌───────┴───────┐
└──────────────┘              │              │upgrade_record │
                              │              ├───────────────┤
                              │              │ id (PK)       │
                              └─────────────►│ tenant_id(FK) │
                                             │ from_version  │
                                             │ to_version    │
                                             │ status        │
                                             │ backup_id     │
                                             │ error_msg     │
                                             │ started_at    │
                                             │ completed_at  │
                                             │ operated_by   │
                                             └───────────────┘

┌──────────────┐       ┌──────────────┐
│ saas_config  │       │  login_log    │
├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │
│ config_key   │       │ admin_id(FK) │
│ config_value │       │ login_time   │
│ config_type  │       │ login_ip     │
│ description  │       │ device_type  │
│ created_at   │       │ os           │
│ updated_at   │       │ browser      │
└──────────────┘       │ login_result │
                       │ fail_reason  │
                       └──────────────┘
```

## 3. 表结构

### 3.1 saas_admin - 平台管理员表

```sql
CREATE TABLE saas_admin (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL COMMENT '用户名',
    password_hash VARCHAR(255) NOT NULL COMMENT '密码哈希',
    name VARCHAR(50) NOT NULL COMMENT '姓名',
    email VARCHAR(100) UNIQUE NOT NULL COMMENT '邮箱',
    phone VARCHAR(20) COMMENT '手机号',
    avatar_url VARCHAR(500) COMMENT '头像URL',
    role VARCHAR(50) NOT NULL DEFAULT 'SYSTEM_SUPER_ADMIN' COMMENT '角色',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/inactive',
    last_login_at TIMESTAMP COMMENT '最后登录时间',
    last_login_ip VARCHAR(50) COMMENT '最后登录IP',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_saas_admin_username ON saas_admin(username);
CREATE INDEX idx_saas_admin_status ON saas_admin(status);

-- 初始数据：平台超级管理员
INSERT INTO saas_admin (id, username, password_hash, name, email, role, status)
VALUES (
    '00000000-0000-0000-0000-000000000001',
    'admin',
    '$2b$12$...', -- 初始密码哈希，首次登录后强制修改
    '系统管理员',
    'admin@smartaudit.com',
    'SYSTEM_SUPER_ADMIN',
    'active'
);
```

### 3.2 tenant - 租户表

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL COMMENT '租户名称',
    code VARCHAR(50) UNIQUE NOT NULL COMMENT '租户代码',
    contact_name VARCHAR(50) NOT NULL COMMENT '联系人姓名',
    contact_email VARCHAR(100) NOT NULL COMMENT '联系人邮箱',
    contact_phone VARCHAR(20) NOT NULL COMMENT '联系人电话',
    address VARCHAR(200) COMMENT '地址',
    logo_url VARCHAR(500) COMMENT 'Logo URL',
    app_id VARCHAR(50) UNIQUE NOT NULL COMMENT '应用ID',
    app_secret_hash VARCHAR(255) NOT NULL COMMENT '应用密钥哈希',
    package_type VARCHAR(20) DEFAULT 'basic' COMMENT '套餐: basic/professional/enterprise',
    deploy_mode VARCHAR(20) DEFAULT 'saas' COMMENT '部署模式: saas/dedicated',
    service_start_date DATE COMMENT '服务开始日期',
    service_end_date DATE COMMENT '服务结束日期',
    status VARCHAR(20) DEFAULT 'active' COMMENT '状态: active/inactive/expired',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tenant_code ON tenant(code);
CREATE INDEX idx_tenant_status ON tenant(status);
CREATE INDEX idx_tenant_created ON tenant(created_at);
```

### 3.3 system_version - 系统版本表

```sql
CREATE TABLE system_version (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version VARCHAR(50) NOT NULL COMMENT '版本号: 1.2.3',
    version_code INT NOT NULL COMMENT '版本数字: 10203',
    release_type VARCHAR(20) DEFAULT 'stable' COMMENT '发布类型: stable/beta/release',
    release_notes TEXT COMMENT '发布说明 (Markdown)',
    download_url VARCHAR(500) NOT NULL COMMENT '下载地址',
    checksum VARCHAR(64) NOT NULL COMMENT 'SHA256校验码',
    is_mandatory BOOLEAN DEFAULT FALSE COMMENT '是否强制更新',
    is_enabled BOOLEAN DEFAULT TRUE COMMENT '是否启用',
    min_compatible_version VARCHAR(50) COMMENT '最低兼容版本',
    published_at TIMESTAMP COMMENT '发布时间',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_version_code ON system_version(version_code DESC);
CREATE INDEX idx_version_enabled ON system_version(is_enabled);
CREATE INDEX idx_version_published ON system_version(published_at DESC);
```

### 3.4 upgrade_record - 升级记录表

```sql
CREATE TABLE upgrade_record (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    from_version VARCHAR(50) NOT NULL COMMENT '原版本',
    to_version VARCHAR(50) NOT NULL COMMENT '目标版本',
    status VARCHAR(20) DEFAULT 'pending' COMMENT '状态: pending/running/success/failed/rollback',
    backup_id UUID COMMENT '关联备份ID',
    error_msg TEXT COMMENT '错误信息',
    rollback_note TEXT COMMENT '回滚备注',
    started_at TIMESTAMP COMMENT '开始时间',
    completed_at TIMESTAMP COMMENT '完成时间',
    operated_by UUID COMMENT '操作人ID',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_upgrade_tenant ON upgrade_record(tenant_id);
CREATE INDEX idx_upgrade_status ON upgrade_record(status);
CREATE INDEX idx_upgrade_created ON upgrade_record(created_at DESC);
```

### 3.5 saas_config - 全局配置表

```sql
CREATE TABLE saas_config (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_key VARCHAR(100) UNIQUE NOT NULL COMMENT '配置键',
    config_value TEXT COMMENT '配置值',
    config_type VARCHAR(20) DEFAULT 'string' COMMENT '类型: string/number/boolean/json',
    description VARCHAR(200) COMMENT '说明',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_config_key ON saas_config(config_key);

-- 初始配置
INSERT INTO saas_config (config_key, config_value, config_type, description) VALUES
('platform_name', 'SmartAudit', 'string', '平台名称'),
('platform_logo', '', 'string', '平台Logo URL'),
('default_package', 'basic', 'string', '新租户默认套餐'),
('token_expire_minutes', '30', 'number', 'Token有效期(分钟)'),
('refresh_token_expire_days', '7', 'number', 'RefreshToken有效期(天)'),
('max_users_per_tenant', '0', 'number', '单租户最大用户数(0=不限)'),
('max_projects_per_tenant', '0', 'number', '单租户最大项目数(0=不限)'),
('login_max_fail_count', '5', 'number', '登录失败最大次数'),
('login_lock_minutes', '15', 'number', '登录锁定时间(分钟)'),
('version_check_interval_hours', '24', 'number', '版本检查间隔(小时)');
```

### 3.6 login_log - 登录日志表

```sql
CREATE TABLE login_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id UUID REFERENCES saas_admin(id) ON DELETE SET NULL,
    login_time TIMESTAMP NOT NULL COMMENT '登录时间',
    login_ip VARCHAR(50) COMMENT '登录IP',
    device_type VARCHAR(20) COMMENT '设备类型',
    os VARCHAR(50) COMMENT '操作系统',
    browser VARCHAR(50) COMMENT '浏览器',
    login_result VARCHAR(20) NOT NULL COMMENT '结果: success/failed'),
    fail_reason VARCHAR(200) COMMENT '失败原因',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_login_log_admin ON login_log(admin_id);
CREATE INDEX idx_login_log_time ON login_log(login_time);
```

### 3.7 operation_log - 操作日志表

```sql
CREATE TABLE operation_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id UUID REFERENCES saas_admin(id) ON DELETE SET NULL,
    module VARCHAR(50) NOT NULL COMMENT '模块',
    action VARCHAR(50) NOT NULL COMMENT '动作',
    detail TEXT COMMENT '详情',
    entity_type VARCHAR(50) COMMENT '实体类型',
    entity_id UUID COMMENT '实体ID',
    ip VARCHAR(50) COMMENT 'IP地址',
    request_data JSONB COMMENT '请求数据',
    status VARCHAR(20) DEFAULT 'success' COMMENT '状态',
    error_msg TEXT COMMENT '错误信息',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_operation_log_admin ON operation_log(admin_id);
CREATE INDEX idx_operation_log_module ON operation_log(module);
CREATE INDEX idx_operation_log_time ON operation_log(created_at);
```

## 4. 索引规划

| 表名 | 索引名 | 字段 | 类型 |
|------|--------|------|------|
| saas_admin | idx_saas_admin_username | username | 唯一 |
| saas_admin | idx_saas_admin_status | status | 普通 |
| tenant | idx_tenant_code | code | 唯一 |
| tenant | idx_tenant_status | status | 普通 |
| tenant | idx_tenant_created | created_at | 普通 |
| system_version | idx_version_code | version_code | 普通 |
| system_version | idx_version_enabled | is_enabled | 普通 |
| upgrade_record | idx_upgrade_tenant | tenant_id | 普通 |
| upgrade_record | idx_upgrade_status | status | 普通 |
| saas_config | idx_config_key | config_key | 唯一 |

## 5. 初始化脚本

```sql
-- SaaS 数据库初始化脚本
-- 执行: psql -U postgres -d smartaudit_saas -f init_saas.sql

-- 创建扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 创建表 (如上)

-- 插入初始管理员
-- 初始密码: SaaS@admin123 (首次登录后强制修改)
INSERT INTO saas_admin (
    id,
    username,
    password_hash,
    name,
    email,
    role,
    status
) VALUES (
    '00000000-0000-0000-0000-000000000001',
    'admin',
    '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4bMxOVJP8QhGQcWy',
    '系统管理员',
    'admin@smartaudit.com',
    'SYSTEM_SUPER_ADMIN',
    'active'
);

-- 插入初始配置
INSERT INTO saas_config (config_key, config_value, config_type, description) VALUES
('platform_name', 'SmartAudit', 'string', '平台名称'),
('platform_logo', '', 'string', '平台Logo URL'),
('default_package', 'basic', 'string', '新租户默认套餐'),
('token_expire_minutes', '30', 'number', 'Token有效期(分钟)'),
('refresh_token_expire_days', '7', 'number', 'RefreshToken有效期(天)'),
('max_users_per_tenant', '0', 'number', '单租户最大用户数(0=不限)'),
('max_projects_per_tenant', '0', 'number', '单租户最大项目数(0=不限)');
```
