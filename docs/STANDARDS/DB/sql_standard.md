# SQL 规范

## 1. 通用规范

### 1.1 命名规范

#### 表命名
- 使用 snake_case
- 复数形式或带前缀：`sys_tenants`、`sys_accounts`、`audit_logs`
- SaaS 平台租户相关表：`saas_xxx`
- 租户业务表：`biz_xxx` 或直接业务名

#### 字段命名
- 全部使用 snake_case：`created_at`、`updated_at`
- 布尔字段：`is_deleted`、`is_active`、`is_enabled`
- 外键字段：`tenant_id`、`project_id`、`user_id`
- 时间戳：`created_at`、`updated_at`、`deleted_at`
- 枚举字段：`_status`、`_type` 后缀

### 1.2 字段类型选择

#### PostgreSQL vs MySQL 对比

| 场景 | PostgreSQL | MySQL |
|------|------------|-------|
| 主键自增 | `SERIAL` 或 `BIGSERIAL` | `INT AUTO_INCREMENT` 或 `BIGINT AUTO_INCREMENT` |
| 唯一标识 | `UUID DEFAULT gen_random_uuid()` | `VARCHAR(36)` 或 `CHAR(36)` |
| 大文本 | `TEXT` | `TEXT` |
| 布尔值 | `BOOLEAN` | `TINYINT(1)` |
| 时间戳 | `TIMESTAMP` 或 `TIMESTAMP WITH TIME ZONE` | `DATETIME` |
| JSON | `JSONB`（推荐）或 `JSON` | `JSON` |
| 大数字 | `BIGINT` | `BIGINT` |

### 1.3 软删除实现

```sql
-- 所有业务表必须包含软删除字段
-- PostgreSQL
CREATE TABLE sys_tenants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted SMALLINT DEFAULT 0,  -- 0: 正常, 1: 已删除
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,  -- 删除时间，可选
    created_by BIGINT,
    updated_by BIGINT,
    version INTEGER DEFAULT 0  -- 乐观锁
);

-- MySQL
CREATE TABLE sys_tenants (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(20) DEFAULT 'active',
    is_deleted TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    version INT DEFAULT 0,
    INDEX idx_is_deleted (is_deleted),
    INDEX idx_tenant_code (code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 2. 索引规范

### 2.1 索引命名
- 普通索引：`idx_字段名` 或 `idx_表名_字段名`
- 唯一索引：`uk_字段名` 或 `uk_表名_字段名`
- 联合索引：`idx_字段1_字段2`

### 2.2 索引创建原则
```sql
-- 选择性高的字段建索引
-- Good
CREATE INDEX idx_tenant_status ON biz_projects(tenant_id, status);

-- Avoid: 不要再低选择性字段上建单字段索引
-- SELECT * FROM users WHERE is_deleted = 1; -- 不应为此建索引

-- 覆盖索引（查询仅需索引即可完成）
CREATE INDEX idx_project_list ON biz_projects(tenant_id, status, id, name);

-- 联合索引遵循最左前缀原则
-- idx_tenant_status_created_at 支持以下查询:
-- WHERE tenant_id = ?
-- WHERE tenant_id = ? AND status = ?
-- WHERE tenant_id = ? AND status = ? AND created_at > ?
```

### 2.3 索引示例

```sql
-- PostgreSQL
CREATE INDEX idx_tenant_id ON biz_projects(tenant_id);
CREATE INDEX idx_project_status ON biz_projects(status);
CREATE INDEX idx_tenant_status ON biz_projects(tenant_id, status);
CREATE UNIQUE INDEX uk_tenant_code ON sys_tenants(code);
CREATE INDEX idx_created_at ON audit_logs(created_at);

-- MySQL
CREATE INDEX idx_tenant_id ON biz_projects(tenant_id);
CREATE INDEX idx_project_status ON biz_projects(status);
CREATE INDEX idx_tenant_status ON biz_projects(tenant_id, status);
CREATE UNIQUE INDEX uk_tenant_code ON sys_tenants(code);
CREATE INDEX idx_created_at ON audit_logs(created_at);
```

## 3. 查询规范

### 3.1 基础查询
```sql
-- 必须指定查询字段，避免 SELECT *
-- Good
SELECT id, name, code, status, created_at
FROM sys_tenants
WHERE is_deleted = 0;

-- Avoid
SELECT * FROM sys_tenants;

-- 使用表别名
SELECT t.id, t.name, p.name as project_name
FROM sys_tenants t
LEFT JOIN biz_projects p ON p.tenant_id = t.id
WHERE t.is_deleted = 0;
```

### 3.2 软删除查询
```sql
-- 默认只查询未删除数据
SELECT id, name, code
FROM sys_tenants
WHERE is_deleted = 0;

-- 需要查询已删除数据时明确指定
SELECT id, name, code, deleted_at
FROM sys_tenants
WHERE is_deleted = 1;

-- 删除操作使用软删除
-- Good
UPDATE sys_tenants
SET is_deleted = 1, deleted_at = NOW()
WHERE id = ?;

-- Avoid
DELETE FROM sys_tenants WHERE id = ?;
```

### 3.3 分页查询

```sql
-- PostgreSQL
-- 方法1: OFFSET-FETCH（推荐）
SELECT id, name, created_at
FROM sys_tenants
WHERE is_deleted = 0
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- 方法2: 子查询
SELECT * FROM (
    SELECT id, name, created_at,
           ROW_NUMBER() OVER (ORDER BY created_at DESC) as rn
    FROM sys_tenants
    WHERE is_deleted = 0
) t
WHERE rn > 0 AND rn <= 20;

-- MySQL
SELECT id, name, created_at
FROM sys_tenants
WHERE is_deleted = 0
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### 3.4 关联查询
```sql
-- 左连接获取租户下的项目
SELECT t.id, t.name, COUNT(p.id) as project_count
FROM sys_tenants t
LEFT JOIN biz_projects p ON p.tenant_id = t.id AND p.is_deleted = 0
WHERE t.is_deleted = 0
GROUP BY t.id, t.name;

-- 多表连接
SELECT a.id, a.name, r.name as role_name
FROM biz_accounts a
LEFT JOIN biz_account_roles ar ON ar.account_id = a.id
LEFT JOIN biz_roles r ON r.id = ar.role_id
WHERE a.is_deleted = 0;
```

## 4. 事务规范

### 4.1 事务边界
```python
# Python/FastAPI
from sqlalchemy.ext.asyncio import AsyncSession

async def create_tenant_with_project(db: AsyncSession, tenant_data, project_data):
    async with db.begin():
        # 创建租户
        tenant = Tenant(**tenant_data)
        db.add(tenant)
        await db.flush()

        # 创建默认项目
        project_data['tenant_id'] = tenant.id
        project = Project(**project_data)
        db.add(project)

        await db.commit()
```

### 4.2 事务隔离级别
```sql
-- PostgreSQL 默认 READ COMMITTED
-- MySQL InnoDB 默认 REPEATABLE READ

-- 金融等高敏感场景可提高隔离级别
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 或在事务中指定
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- 业务操作
COMMIT;
```

## 5. 数据库对象规范

### 5.1 视图
```sql
-- 创建视图用于简化复杂查询
-- PostgreSQL
CREATE OR REPLACE VIEW v_tenant_project_summary AS
SELECT
    t.id as tenant_id,
    t.name as tenant_name,
    COUNT(DISTINCT p.id) as project_count,
    COUNT(DISTINCT a.id) as account_count,
    MAX(p.created_at) as last_project_created
FROM sys_tenants t
LEFT JOIN biz_projects p ON p.tenant_id = t.id AND p.is_deleted = 0
LEFT JOIN biz_accounts a ON a.tenant_id = t.id AND a.is_deleted = 0
WHERE t.is_deleted = 0
GROUP BY t.id, t.name;

-- MySQL
CREATE ALGORITHM = MERGE VIEW v_tenant_project_summary AS
SELECT
    t.id as tenant_id,
    t.name as tenant_name,
    COUNT(DISTINCT p.id) as project_count,
    COUNT(DISTINCT a.id) as account_count,
    MAX(p.created_at) as last_project_created
FROM sys_tenants t
LEFT JOIN biz_projects p ON p.tenant_id = t.id AND p.is_deleted = 0
LEFT JOIN biz_accounts a ON a.tenant_id = t.id AND a.is_deleted = 0
WHERE t.is_deleted = 0
GROUP BY t.id, t.name;
```

### 5.2 触发器
```sql
-- 更新时间戳触发器
-- PostgreSQL
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_sys_tenants_updated_at
    BEFORE UPDATE ON sys_tenants
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- MySQL
DELIMITER $$
CREATE TRIGGER tr_sys_tenants_update
    BEFORE UPDATE ON sys_tenants
    FOR EACH ROW
BEGIN
    SET NEW.updated_at = CURRENT_TIMESTAMP;
END$$
DELIMITER ;
```

### 5.3 函数/存储过程
```sql
-- 保守使用存储过程，业务逻辑在应用层

-- PostgreSQL: 获取租户下的统计数据
CREATE OR REPLACE FUNCTION get_tenant_stats(p_tenant_id BIGINT)
RETURNS TABLE (
    project_count BIGINT,
    account_count BIGINT,
    task_count BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        (SELECT COUNT(*) FROM biz_projects WHERE tenant_id = p_tenant_id AND is_deleted = 0),
        (SELECT COUNT(*) FROM biz_accounts WHERE tenant_id = p_tenant_id AND is_deleted = 0),
        (SELECT COUNT(*) FROM biz_tasks WHERE tenant_id = p_tenant_id AND is_deleted = 0 AND status = 'pending');
END;
$$ LANGUAGE plpgsql;

-- MySQL: 类似实现
DELIMITER $$
CREATE FUNCTION get_tenant_stats(p_tenant_id BIGINT)
RETURNS JSON
DETERMINISTIC
BEGIN
    DECLARE v_project_count INT;
    DECLARE v_account_count INT;
    DECLARE v_task_count INT;

    SELECT COUNT(*) INTO v_project_count
    FROM biz_projects
    WHERE tenant_id = p_tenant_id AND is_deleted = 0;

    SELECT COUNT(*) INTO v_account_count
    FROM biz_accounts
    WHERE tenant_id = p_tenant_id AND is_deleted = 0;

    SELECT COUNT(*) INTO v_task_count
    FROM biz_tasks
    WHERE tenant_id = p_tenant_id AND is_deleted = 0 AND status = 'pending';

    RETURN JSON_OBJECT(
        'project_count', v_project_count,
        'account_count', v_account_count,
        'task_count', v_task_count
    );
END$$
DELIMITER ;
```

## 6. 性能优化规范

### 6.1 SQL 优化原则
- 避免 SELECT *
- 合理使用索引
- 避免在索引列上使用函数
- 使用 EXPLAIN 分析查询
- 控制批量操作的大小

### 6.2 慢查询示例
```sql
-- 避免在索引列上使用函数
-- Bad
SELECT * FROM users WHERE DATE(created_at) = '2024-01-01';
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- Good
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';

-- 避免隐式类型转换
-- Bad (phone 是 VARCHAR)
SELECT * FROM users WHERE phone = 13800138000;

-- Good
SELECT * FROM users WHERE phone = '13800138000';
```

### 6.3 批量操作
```sql
-- PostgreSQL
INSERT INTO biz_projects (name, code, tenant_id, created_at)
VALUES
    ('项目1', 'p001', 1, NOW()),
    ('项目2', 'p002', 1, NOW()),
    ('项目3', 'p003', 1, NOW());

-- 批量更新
UPDATE biz_accounts
SET status = 'disabled', updated_at = NOW()
WHERE tenant_id = 1 AND id IN (1, 2, 3, 4, 5);

-- MySQL
INSERT INTO biz_projects (name, code, tenant_id, created_at)
VALUES
    ('项目1', 'p001', 1, NOW()),
    ('项目2', 'p002', 1, NOW()),
    ('项目3', 'p003', 1, NOW());
```

## 7. 安全规范

### 7.1 防止 SQL 注入
- 始终使用参数化查询
- 禁止拼接 SQL 字符串

```python
# Good - 参数化查询
async def get_tenant_by_code(db: AsyncSession, code: str):
    result = await db.execute(
        select(Tenant).where(Tenant.code == code, Tenant.is_deleted == 0)
    )
    return result.scalar_one_or_none()

# Bad - 字符串拼接
# cursor.execute(f"SELECT * FROM tenants WHERE code = '{code}'")
```

### 7.2 敏感数据处理
```sql
-- 敏感字段加密存储
ALTER TABLE biz_accounts ADD COLUMN password_hash VARCHAR(255) NOT NULL;
ALTER TABLE biz_accounts ADD COLUMN salt VARCHAR(50);

-- 日志中脱敏
SELECT
    id,
    name,
    CONCAT(LEFT(phone, 3), '****', RIGHT(phone, 4)) as phone_masked,
    CONCAT(LEFT(email, 2), '***', SUBSTRING(email, POSITION('@' IN email))) as email_masked
FROM biz_accounts;
```

## 8. 数据库维护

### 8.1 定期维护任务
```sql
-- 清理软删除数据（物理删除）
-- 建议: 数据删除超过 30 天后物理删除
DELETE FROM audit_logs
WHERE is_deleted = 1 AND deleted_at < NOW() - INTERVAL '30 days';

-- 重建失效索引
REINDEX INDEX idx_tenant_status;

-- 更新统计信息
-- PostgreSQL
ANALYZE sys_tenants;
VACUUM ANALYZE sys_tenants;

-- MySQL
ANALYZE TABLE sys_tenants;
```

### 8.2 备份策略
```sql
-- PostgreSQL 备份
-- 全量备份
pg_dump -h localhost -U postgres -d smartaudit -F c -b -v -f backup.sql

-- 差异备份（基于上次全量）
pg_basebackup -h localhost -U postgres -D /backup -Ft -z -P

-- MySQL 备份
-- 全量备份
mysqldump -h localhost -u root -p --single-transaction smartaudit > backup.sql

-- 恢复
-- PostgreSQL
pg_restore -h localhost -U postgres -d smartaudit backup.sql

-- MySQL
mysql -h localhost -u root -p smartaudit < backup.sql
```

## 9. 多租户数据隔离

### 9.1 行级隔离（ SaaS 模式）
```sql
-- 所有查询必须带上 tenant_id
SELECT id, name, code
FROM biz_projects
WHERE tenant_id = ? AND is_deleted = 0;

-- 在数据库层添加 RLS（行级安全策略）
-- PostgreSQL
ALTER TABLE biz_projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON biz_projects
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);
```

### 9.2 Schema 隔离（独立部署）
```sql
-- 每个租户独立的 schema
CREATE SCHEMA tenant_001;
CREATE SCHEMA tenant_002;

-- 切换 schema
SET search_path TO tenant_001;
```
