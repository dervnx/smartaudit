# 租户备份管理系统

## 1. 功能概述

备份管理系统用于定期备份租户数据库，支持手动备份和自动备份，确保数据安全。

**功能模块**：
- 备份规则配置
- 手动执行备份
- 备份记录查看
- 数据恢复

## 2. 备份策略

### 2.1 备份类型

| 类型 | 代码 | 说明 | 执行频率 | 保留策略 |
|------|------|------|----------|----------|
| 全量备份 | full | 完整数据库备份 | 每周一次 | 保留 30 天 |
| 差异备份 | differential | 与上次全量备份的差异 | 每天一次 | 保留 7 天 |
| 增量备份 | incremental | 与上次备份的差异 | 每小时一次 | 保留 3 天 |

### 2.2 备份时间规划

```
┌─────────────────────────────────────────────────────────────────┐
│                        备份时间规划                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  周日 02:00  ┌──────────┐                                     │
│              │ 全量备份  │ ← 完整备份，覆盖所有数据               │
│              └──────────┘                                     │
│                  │                                              │
│  周一 02:00  ┌──────────┐                                     │
│              │ 差异备份  │ ← 备份周日以来的变更                   │
│              └──────────┘                                     │
│                  │                                              │
│  周二 02:00  ┌──────────┐                                     │
│              │ 差异备份  │ ← 备份周日以来的变更                   │
│              └──────────┘                                     │
│                  │                                              │
│  ...             │                                              │
│                  │                                              │
│  周日 02:00  ┌──────────┐                                     │
│              │ 全量备份  │ ← 新的全量备份周期                     │
│              └──────────┘                                     │
│                                                                  │
│  每小时      ┌──────────┐                                     │
│              │ 增量备份  │ ← 备份上一小时以来的变更               │
│              └──────────┘                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 备份规则配置

### 3.1 规则字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 规则名称 | 字符串 | ✓ | 规则名称 |
| 备份类型 | 枚举 | ✓ | full/differential/incremental |
| 执行时间 | Cron | ✓ | Cron 表达式 |
| 保留天数 | 数字 | ✓ | 备份保留天数 |
| 存储位置 | 枚举 | ✓ | local/s3/oss |
| 存储配置 | JSON | ✗ | 存储连接配置 |
| 状态 | 布尔 | ✓ | 启用/禁用 |

### 3.2 Cron 表达式示例

| 表达式 | 说明 |
|--------|------|
| `0 2 * * *` | 每天凌晨 2 点 |
| `0 2 * * 0` | 每周日凌晨 2 点 |
| `0 */4 * * *` | 每 4 小时 |
| `0 2 * * 1-5` | 工作日凌晨 2 点 |

### 3.3 存储配置

**本地存储**:
```json
{
  "type": "local",
  "path": "/data/backups"
}
```

**S3 存储**:
```json
{
  "type": "s3",
  "region": "us-east-1",
  "bucket": "smartaudit-backups",
  "accessKeyId": "AKIA...",
  "secretAccessKey": "..."
}
```

**OSS 存储**:
```json
{
  "type": "oss",
  "endpoint": "oss-cn-hangzhou.aliyuncs.com",
  "bucket": "smartaudit-backups",
  "accessKeyId": "...",
  "secretAccessKey": "..."
}
```

## 4. 备份执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       备份执行流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 触发备份                                                     │
│     ├── 定时任务触发                                              │
│     └── 手动触发                                                  │
│          │                                                       │
│          ▼                                                       │
│  2. 创建备份记录 (status=pending)                                 │
│          │                                                       │
│          ▼                                                       │
│  3. 更新状态为 running                                           │
│          │                                                       │
│          ▼                                                       │
│  4. 执行数据库备份                                               │
│     ├── 全量: pg_dump -Fc smartaudit > full_backup.dump         │
│     ├── 差异: 计算与上次全量的差异                                │
│     └── 增量: 计算与上次备份的差异                                │
│          │                                                       │
│          ▼                                                       │
│  5. 压缩备份文件                                                 │
│     └── tar -czf backup.tar.gz full_backup.dump                 │
│          │                                                       │
│          ▼                                                       │
│  6. 计算校验码                                                   │
│     └── sha256sum backup.tar.gz                                 │
│          │                                                       │
│          ▼                                                       │
│  7. 上传到存储                                                   │
│     ├── 本地: cp backup.tar.gz /data/backups/                   │
│     ├── S3: aws s3 cp backup.tar.gz s3://bucket/               │
│     └── OSS: ossutil cp backup.tar.gz oss://bucket/            │
│          │                                                       │
│          ▼                                                       │
│  8. 更新备份记录                                                 │
│     ├── status = success                                        │
│     ├── file_path = 上传路径                                     │
│     ├── file_size = 文件大小                                     │
│     └── completed_at = 完成时间                                  │
│          │                                                       │
│          ▼                                                       │
│  9. 清理过期备份                                                 │
│     ├── 查找超过保留期的备份                                      │
│     └── 删除备份文件和记录                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 5. 备份服务实现

```python
# app/services/backup_service.py
import subprocess
import hashlib
import tarfile
from datetime import datetime
from pathlib import Path

class BackupService:
    def __init__(self, storage: StorageService):
        self.storage = storage

    async def execute_backup(self, rule_id: str) -> BackupRecord:
        """执行备份"""
        rule = await self.get_rule(rule_id)

        # 创建备份记录
        record = BackupRecord(
            tenant_id=rule.tenant_id,
            rule_id=rule.id,
            backup_type=rule.backup_type,
            status='running',
            started_at=datetime.utcnow()
        )
        await self.save_record(record)

        try:
            # 执行数据库备份
            backup_file = await self._dump_database(rule.tenant_id)

            # 压缩
            compressed_file = await self._compress(backup_file)

            # 计算校验码
            checksum = await self._calculate_checksum(compressed_file)

            # 上传到存储
            file_path = await self.storage.upload(
                compressed_file,
                f"backup/{rule.tenant_id}/{record.id}.tar.gz"
            )

            # 更新记录
            record.status = 'success'
            record.file_path = file_path
            record.file_size = Path(compressed_file).stat().st_size
            record.checksum = checksum
            record.completed_at = datetime.utcnow()

            await self.save_record(record)

            # 更新规则的最后备份时间
            rule.last_backup_at = datetime.utcnow()
            rule.next_backup_at = self._calculate_next_backup(rule)
            await self.save_rule(rule)

            # 清理过期备份
            await self.cleanup_old_backups(rule.tenant_id, rule.retention_days)

            return record

        except Exception as e:
            record.status = 'failed'
            record.error_msg = str(e)
            await self.save_record(record)
            raise

    async def _dump_database(self, tenant_id: str) -> Path:
        """执行 pg_dump"""
        backup_dir = Path("/tmp/backups")
        backup_dir.mkdir(exist_ok=True)

        filename = backup_dir / f"{tenant_id}_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}.dump"

        cmd = [
            'pg_dump',
            '-h', settings.DB_HOST,
            '-U', settings.DB_USER,
            '-d', settings.DB_NAME,
            '-Fc',
            '-f', str(filename)
        ]

        subprocess.run(
            cmd,
            env={**os.environ, 'PGPASSWORD': settings.DB_PASSWORD},
            check=True
        )

        return filename

    async def _compress(self, backup_file: Path) -> Path:
        """压缩备份文件"""
        compressed = backup_file.with_suffix('.tar.gz')

        with tarfile.open(compressed, 'w:gz') as tar:
            tar.add(backup_file, arcname=backup_file.name)

        return compressed
```

## 6. 数据恢复流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       数据恢复流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 选择备份                                                    │
│     ├── 从备份列表选择                                           │
│     └── 确认备份完整性                                           │
│          │                                                       │
│          ▼                                                       │
│  2. 创建恢复前备份                                               │
│     └── 执行一次全量备份，防止数据丢失                            │
│          │                                                       │
│          ▼                                                       │
│  3. 停止应用服务                                                 │
│     └── 暂停写入操作                                             │
│          │                                                       │
│          ▼                                                       │
│  4. 下载备份文件                                                 │
│     └── 从存储下载到本地                                         │
│          │                                                       │
│          ▼                                                       │
│  5. 解压备份文件                                                 │
│     └── tar -xzf backup.tar.gz                                  │
│          │                                                       │
│          ▼                                                       │
│  6. 执行数据恢复                                                 │
│     └── pg_restore -d smartaudit backup.dump                   │
│          │                                                       │
│          ▼                                                       │
│  7. 验证数据完整性                                               │
│     └── 执行数据校验查询                                          │
│          │                                                       │
│          ▼                                                       │
│  8. 启动应用服务                                                 │
│          │                                                       │
│          ▼                                                       │
│  9. 恢复完成记录                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 7. 恢复服务实现

```python
# app/services/restore_service.py
class RestoreService:
    async def restore_backup(self, record_id: str) -> bool:
        """恢复备份"""
        record = await self.get_record(record_id)

        if record.status != 'success':
            raise ValueError('只能恢复成功的备份')

        # 1. 创建恢复前备份
        await self.create_pre_restore_backup(record.tenant_id)

        try:
            # 2. 停止应用（通知其他实例）
            await self.notify_application_stop()

            # 3. 下载备份文件
            backup_file = await self.storage.download(record.file_path)

            # 4. 解压
            dump_file = await self._extract(backup_file)

            # 5. 执行恢复
            await self._restore_database(dump_file)

            # 6. 验证
            await self._verify_database()

            # 7. 通知重启
            await self.notify_application_start()

            return True

        except Exception as e:
            # 回滚通知
            await self.notify_application_start()
            raise RestoreError(f'恢复失败: {str(e)}')
```

## 8. 备份记录

### 8.1 记录字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 备份ID | UUID | 唯一标识 |
| 关联规则 | UUID | 所属备份规则 |
| 备份类型 | 枚举 | full/differential/incremental |
| 状态 | 枚举 | pending/running/success/failed |
| 开始时间 | 时间戳 | 开始执行时间 |
| 完成时间 | 时间戳 | 完成时间 |
| 文件路径 | 字符串 | 备份文件存储路径 |
| 文件大小 | 数字 | 文件大小（字节） |
| 校验码 | 字符串 | SHA256 校验码 |
| 错误信息 | 字符串 | 失败时的错误详情 |

### 8.2 记录展示

```
┌─────────────────────────────────────────────────────────────────┐
│  备份记录                                                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ✓ 备份-20240120-020000   全量备份   成功   1.2 GB   2024-01-20 02:15 ││
│  │ ✓ 备份-20240121-020000   差异备份   成功   156 MB   2024-01-21 02:05 ││
│  │ ✓ 备份-20240122-020000   差异备份   成功   178 MB   2024-01-22 02:08 ││
│  │ ✗ 备份-20240123-020000   差异备份   失败   -       2024-01-23 02:00 ││
│  │   错误: 连接超时                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## 9. 权限控制

| 权限代码 | 权限名称 | 说明 |
|----------|----------|------|
| BACKUP_RULE_VIEW | 查看备份规则 | 查看备份规则列表 |
| BACKUP_RULE_CREATE | 创建备份规则 | 创建备份规则 |
| BACKUP_RULE_EDIT | 编辑备份规则 | 修改备份规则 |
| BACKUP_RULE_DELETE | 删除备份规则 | 删除备份规则 |
| BACKUP_EXECUTE | 执行备份 | 手动执行备份 |
| BACKUP_RESTORE | 恢复备份 | 从备份恢复数据 |
| BACKUP_VIEW | 查看备份记录 | 查看备份历史 |

## 10. 页面布局

```
┌─────────────────────────────────────────────────────────────────┐
│  备份管理                                    [+ 创建规则]         │
├─────────────────────────────────────────────────────────────────┤
│  备份规则                                                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ┌─────────────────┐                                          ││
│  │ │ 每日自动备份     │  差异备份 │ 每天 02:00 │ 保留 7 天 │ ✓ ││
│  │ │ [编辑] [删除]  │                                          ││
│  │ └─────────────────┘                                          ││
│  │ ┌─────────────────┐                                          ││
│  │ │ 每周全量备份     │  全量备份 │ 周日 02:00 │ 保留 30 天 │ ✓ ││
│  │ │ [编辑] [删除]  │                                          ││
│  │ └─────────────────┘                                          ││
│  └─────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  备份记录                                          [手动备份]     │
├─────────────────────────────────────────────────────────────────┤
│  筛选: [全部状态 ▼] [全部类型 ▼]                                 │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 备份文件              │ 类型    │ 状态   │ 大小   │ 时间       ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ 备份-20240120-020000  │ 全量    │ ✓成功  │ 1.2 GB │ 02:15    ││
│  │ 备份-20240121-020000  │ 差异    │ ✓成功  │ 156 MB │ 02:05    ││
│  │ 备份-20240122-020000  │ 差异    │ ✗失败  │ -      │ 02:00    ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```
