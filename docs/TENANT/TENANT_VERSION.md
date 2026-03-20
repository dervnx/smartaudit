# 租户版本管理系统

## 1. 功能概述

版本管理系统用于管理租户系统的版本，支持版本查看、更新检测、在线升级等功能。

**功能模块**：
- 当前版本显示
- 版本更新检测
- 执行升级
- 升级记录

**适用场景**：独立部署模式的租户

## 2. 版本信息

### 2.1 版本字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 当前版本号 | 字符串 | 如 1.2.3 |
| 版本代码 | 数字 | 用于比较，如 10203 |
| 最后检查时间 | 时间戳 | 最后检查新版本时间 |
| 最后更新时间 | 时间戳 | 最后升级时间 |
| 自动更新状态 | 布尔 | 是否开启自动更新 |
| 更新状态 | 枚举 | idle/checking/downloading/upgrading |

### 2.2 版本信息展示

```
┌─────────────────────────────────────────────────────────────────┐
│  系统版本                                                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │    ┌─────────────────────────────┐                        ││
│  │    │                             │                        ││
│  │    │         v1.2.3             │                        ││
│  │    │        当前版本              │                        ││
│  │    │                             │                        ││
│  │    │    发布于 2024-01-15        │                        ││
│  │    │                             │                        ││
│  │    └─────────────────────────────┘                        ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│  更新设置                                                       │
├─────────────────────────────────────────────────────────────────┤
│  自动更新: [●] 开启  [○] 关闭                                    │
│                                                                  │
│  上次检查: 2024-01-20 10:30                                    │
│                                                                  │
│              [检查更新]              [立即升级到最新]             │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 版本检测

### 3.1 检测触发条件

| 触发方式 | 说明 |
|----------|------|
| 登录自动检测 | 管理员登录时自动检测 |
| 手动检测 | 管理员点击"检查更新"按钮 |
| 定时检测 | 开启自动更新时，每24小时检测一次 |

### 3.2 检测流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     版本检测流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 获取当前版本                                                │
│     └── 从 tenant_version 表读取                                 │
│          │                                                       │
│          ▼                                                       │
│  2. 调用平台版本接口                                            │
│     └── GET /saas/api/v1/versions/latest                        │
│          │                                                       │
│          ▼                                                       │
│  3. 对比版本                                                    │
│     └── 比较 version_code                                        │
│          │                                                       │
│          ▼                                                       │
│  4. 返回检测结果                                                │
│     ├── 无更新: 当前已是最新版本                                 │
│     ├── 有更新: 显示新版本信息                                   │
│     └── 强制更新: 必须升级                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 检测结果展示

**无更新时**：
```
┌─────────────────────────────────────────────────────────────────┐
│  ✓ 当前已是最新版本 v1.2.3                                       │
│    无需更新                                                      │
└─────────────────────────────────────────────────────────────────┘
```

**有可用更新时**：
```
┌─────────────────────────────────────────────────────────────────┐
│  发现新版本 v1.3.0                                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 版本: v1.3.0                                                 ││
│  │ 类型: stable                                                 ││
│  │ 大小: 256 MB                                                ││
│  │ 更新说明:                                                   ││
│  │ ## 新增功能                                                  ││
│  │ - xxx                                                       ││
│  │ - xxx                                                       ││
│  │                                                             ││
│  │ ## 问题修复                                                  ││
│  │ - xxx                                                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ☑ 升级前自动备份（推荐）                                         │
│                                                                  │
│             [稍后再说]           [立即升级]                       │
└─────────────────────────────────────────────────────────────────┘
```

**强制更新时**：
```
┌─────────────────────────────────────────────────────────────────┐
│  ⚠ 系统检测到新版本 v1.3.0，必须更新后才能继续使用                  │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 版本: v1.3.0                                                 ││
│  │ 类型: 强制更新                                                ││
│  │ 更新说明:                                                   ││
│  │ ...                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│             [立即升级]                                           │
└─────────────────────────────────────────────────────────────────┘
```

## 4. 升级流程

### 4.1 升级流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                       升级流程                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 升级前检查                                                  │
│     ├── 检查磁盘空间 (> 1GB)                                    │
│     ├── 检查网络连接                                            │
│     └── 检查数据库连接                                           │
│          │                                                       │
│          ▼                                                       │
│  2. 创建备份（可选）                                            │
│     ├── 创建全量备份                                            │
│     └── 记录备份ID                                              │
│          │                                                       │
│          ▼                                                       │
│  3. 下载新版本                                                  │
│     ├── 从下载URL获取                                           │
│     └── 验证校验码                                               │
│          │                                                       │
│          ▼                                                       │
│  4. 执行升级                                                    │
│     ├── 停止服务                                                │
│     ├── 替换程序文件                                            │
│     ├── 执行数据库迁移                                           │
│     │   └── alembic upgrade head                               │
│     └── 启动服务                                                │
│          │                                                       │
│          ▼                                                       │
│  5. 验证升级                                                    │
│     ├── 检查服务状态                                            │
│     └── 验证版本号                                              │
│          │                                                       │
│          ▼                                                       │
│  6. 更新版本记录                                                │
│     ├── 更新 tenant_version                                     │
│     └── 创建 upgrade_record                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 升级服务实现

```python
# app/services/upgrade_service.py
import hashlib
import httpx
from datetime import datetime

class UpgradeService:
    def __init__(
        self,
        db: AsyncSession,
        backup_service: BackupService,
        config_service: ConfigService
    ):
        self.db = db
        self.backup_service = backup_service
        self.config_service = config_service

    async def check_version(self, tenant_id: str) -> VersionCheckResult:
        """检查版本"""
        # 获取当前版本
        current = await self.get_tenant_version(tenant_id)

        # 获取平台最新版本
        latest = await self.fetch_latest_version()

        # 对比
        if latest.version_code > current.version_code:
            return VersionCheckResult(
                hasUpdate=True,
                latestVersion=latest,
                currentVersion=current,
                isMandatory=latest.is_mandatory
            )
        else:
            return VersionCheckResult(hasUpdate=False)

    async def execute_upgrade(
        self,
        tenant_id: str,
        target_version: str,
        create_backup: bool = True
    ) -> UpgradeRecord:
        """执行升级"""
        # 创建升级记录
        record = UpgradeRecord(
            tenant_id=tenant_id,
            from_version=await self.get_current_version(tenant_id),
            to_version=target_version,
            status='running',
            started_at=datetime.utcnow()
        )
        await self.save_record(record)

        try:
            # 1. 升级前检查
            await self.pre_upgrade_check()

            # 2. 创建备份
            if create_backup:
                backup = await self.backup_service.execute_backup(
                    tenant_id,
                    backup_type='full'
                )
                record.backup_id = backup.id

            # 3. 下载新版本
            version_info = await self.get_version_info(target_version)
            download_path = await self.download_version(version_info)

            # 4. 验证校验码
            await self.verify_checksum(download_path, version_info.checksum)

            # 5. 执行升级
            await self.perform_upgrade(download_path, version_info)

            # 6. 更新版本记录
            record.status = 'success'
            record.completed_at = datetime.utcnow()

            # 更新租户版本
            await self.update_tenant_version(
                tenant_id,
                target_version,
                version_info.version_code
            )

            return record

        except Exception as e:
            record.status = 'failed'
            record.error_msg = str(e)
            await self.save_record(record)
            raise

    async def download_version(self, version_info: Version) -> Path:
        """下载新版本"""
        async with httpx.AsyncClient() as client:
            async with client.stream('GET', version_info.download_url) as response:
                response.raise_for_status()

                download_path = Path('/tmp') / f"smartaudit_{version_info.version}.tar.gz"

                with open(download_path, 'wb') as f:
                    async for chunk in response.aiter_bytes(chunk_size=8192):
                        f.write(chunk)

                return download_path

    async def perform_upgrade(self, package_path: Path, version_info: Version):
        """执行升级"""
        # 停止服务
        await self.stop_service()

        # 解压
        extract_path = Path('/opt/smartaudit')
        await self.extract_package(package_path, extract_path)

        # 执行数据库迁移
        await self.run_migrations()

        # 启动服务
        await self.start_service()
```

## 5. 回滚机制

### 5.1 回滚流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       回滚流程                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 选择回滚点                                                  │
│     └── 从升级记录选择                                           │
│          │                                                       │
│          ▼                                                       │
│  2. 确认回滚                                                    │
│     └── 提示回滚影响                                             │
│          │                                                       │
│          ▼                                                       │
│  3. 执行回滚                                                    │
│     ├── 停止服务                                                │
│     ├── 恢复备份文件                                            │
│     ├── 恢复数据库                                              │
│     │   └── pg_restore from backup                            │
│     └── 启动服务                                                │
│          │                                                       │
│          ▼                                                       │
│  4. 更新记录                                                    │
│     └── 更新升级记录状态为 rollback                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 回滚服务实现

```python
async def rollback(self, upgrade_record_id: str) -> bool:
    """回滚"""
    record = await self.get_upgrade_record(upgrade_record_id)

    if record.status != 'success':
        raise ValueError('只能回滚成功的升级')

    if not record.backup_id:
        raise ValueError('没有可用的备份')

    # 获取关联的备份
    backup = await self.backup_service.get_backup(record.backup_id)

    # 执行回滚
    await self.stop_service()
    await self.backup_service.restore_backup(backup)
    await self.start_service()

    # 更新记录
    record.status = 'rollback'
    record.rollback_note = f'回滚到 {record.from_version}'
    await self.save_record(record)

    # 更新租户版本
    await self.update_tenant_version(
        record.tenant_id,
        record.from_version
    )

    return True
```

## 6. 升级记录

### 6.1 记录字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 升级ID | UUID | 唯一标识 |
| 从版本 | 字符串 | 原版本 |
| 到版本 | 字符串 | 目标版本 |
| 状态 | 枚举 | pending/running/success/failed/rollback |
| 开始时间 | 时间戳 | 开始时间 |
| 完成时间 | 时间戳 | 完成时间 |
| 备份记录 | UUID | 关联的备份记录 |
| 错误信息 | 字符串 | 失败时的错误 |
| 回滚备注 | 字符串 | 回滚说明 |
| 操作人 | UUID | 执行人 |

### 6.2 记录展示

```
┌─────────────────────────────────────────────────────────────────┐
│  升级记录                                                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ v1.2.2 → v1.2.3    成功    2024-01-15 10:30    admin     ││
│  │ [查看详情]                                                    ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ v1.2.1 → v1.2.2    成功    2024-01-10 14:20    admin     ││
│  │ [查看详情]                                                    ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ v1.2.0 → v1.2.1    失败    2024-01-05 09:00    admin     ││
│  │ 错误: 磁盘空间不足  [查看详情]  [回滚到 v1.2.0]              ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## 7. 权限控制

| 权限代码 | 权限名称 | 说明 |
|----------|----------|------|
| SYSTEM_VERSION_VIEW | 查看版本信息 | 查看当前版本 |
| SYSTEM_VERSION_CHECK | 检查更新 | 检测新版本 |
| SYSTEM_UPGRADE | 执行升级 | 执行版本升级 |
| SYSTEM_ROLLBACK | 回滚版本 | 回滚到上一版本 |

## 8. API 接口

### 8.1 当前版本

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

### 8.2 检查更新

```
GET /api/v1/version/check
```

**响应**:
```typescript
interface CheckVersionResponse {
  code: 0;
  data: {
    hasUpdate: boolean;
    latestVersion?: {
      version: string;
      versionCode: number;
      releaseType: string;
      releaseNotes: string;
      downloadUrl: string;
      checksum: string;
      isMandatory: boolean;
    };
    currentVersion: {
      version: string;
      versionCode: number;
    };
  };
}
```

### 8.3 执行升级

```
POST /api/v1/version/upgrade
```

**请求参数**:
```typescript
interface UpgradeRequest {
  targetVersion: string;
  createBackup: boolean;
}
```

### 8.4 升级记录

```
GET /api/v1/version/records
```

### 8.5 回滚

```
POST /api/v1/version/rollback
```

**请求参数**:
```typescript
interface RollbackRequest {
  upgradeRecordId: string;
}
```
