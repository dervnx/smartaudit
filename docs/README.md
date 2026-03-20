# SmartAudit 文档体系

## 文档结构

```
docs/
├── README.md                     # 文档总览
│
├── STANDARDS/                    # 代码与设计规范
│   ├── PYTHON/
│   │   └── python_code_standard.md    # Python 代码规范
│   ├── REACT/
│   │   └── react_code_standard.md     # React 代码规范
│   ├── CSS/
│   │   └── css_standard.md            # CSS 样式规范
│   ├── DB/
│   │   └── sql_standard.md            # SQL 规范（PostgreSQL）
│   └── API/
│       └── api_design_standard.md     # API 设计规范
│
├── SPECS/                        # 技术规格文档
│   ├── SYSTEM/
│   │   └── system_architecture.md      # 系统架构设计
│   ├── TENANT/
│   │   └── tenant_db.md               # 租户系统数据库设计
│   ├── TESTING/
│   │   └── testing_standard.md        # 测试规范与测试用例
│   └── DEPLOY/
│       └── deployment_detailed.md      # 部署详细操作手册
│
├── SAAS/                         # SaaS 管理平台文档
│   ├── SAAS_API.md               # SaaS API 规范
│   ├── SAAS_SYSTEM.md            # SaaS 系统设计
│   ├── SAAS_DB.md                # SaaS 数据库设计
│   └── SAAS_DEPLOY.md            # SaaS 部署文档
│
├── TENANT/                       # 租户系统文档
│   ├── TENANT_API.md             # 租户 API 规范
│   ├── TENANT_SYSTEM.md          # 租户系统设计
│   ├── TENANT_DB.md              # 租户数据库设计
│   ├── TENANT_DEPLOY.md          # 租户部署文档
│   ├── TENANT_BACKUP.md          # 租户备份管理
│   └── TENANT_VERSION.md         # 租户版本管理
│
├── PRD/                          # 产品需求文档
│   ├── OVERVIEW.md               # 产品概述
│   ├── FUNCTION_MAP.md           # 功能地图
│   ├── SAAS_ADMIN_PRD.md         # SaaS 管理后台 PRD
│   ├── TENANT_ADMIN_PRD.md       # 租户后台 PRD
│   └── VERSION_PLAN.md           # 版本规划
│
├── SHARED/                       # 共享文档
│   ├── AUTH_DESIGN.md            # 认证授权设计
│   ├── COMMON_TYPES.md           # 公共数据类型
│   └── REDIS_DESIGN.md           # Redis 设计
│
├── PROTOTYPES/                   # 原型设计
│   ├── SAAS/
│   │   └── prototype_saas.md          # SaaS 管理平台原型
│   └── TENANT/
│       └── prototype_tenant.md         # 租户系统原型
│
└── BID_DOCUMENTATION/
    └── smartaudit_bid_document.md      # 投标技术文档
```

---

## 系统架构概览

### 双系统模式

```
┌─────────────────────────────────────────────────────────────────┐
│                         SmartAudit 智核                          │
├───────────────────────────┬─────────────────────────────────────┤
│                           │                                      │
│      SaaS 管理平台         │         租户管理系统                 │
│   (平台运营方使用)          │      (各租户独立使用)                │
│                           │                                      │
│  ┌─────────────────────┐  │  ┌─────────────────────────────┐   │
│  │ - 租户开通/管理      │  │  │ - 项目管理                   │   │
│  │ - 版本发布管理       │  │  │ - 规则引擎                   │   │
│  │ - 升级推送          │  │  │ - 任务审核                   │   │
│  │ - 独立部署版本生成   │  │  │ - 三方接口                   │   │
│  │ - 全局配置          │  │  │ - 数据统计                   │   │
│  │ - 运营统计          │  │  │ - 账户权限                   │   │
│  └─────────────────────┘  │  │ - 版本管理 (独立部署)         │   │
│                           │  │ - 备份管理 (独立部署)         │   │
│                           │  └─────────────────────────────┘   │
│                           │                                      │
└───────────────────────────┴─────────────────────────────────────┘
```

---

## 技术栈

| 层级 | 技术选型 |
|------|----------|
| 前端 | React 18 + TypeScript + Vite + Ant Design |
| 后端 | Python 3.11 + FastAPI + SQLAlchemy 2.x + Pydantic 2.x |
| 数据库 | PostgreSQL 15+ |
| 缓存/队列 | Redis 7+ + Celery |
| 文件存储 | MinIO / S3 / OSS |
| 容器化 | Docker + Docker Compose |

---

## 核心规范

### 1. 命名规范

- **后端字段**：snake_case 风格
  - `created_at`、`updated_at`、`is_deleted`
- **前端字段**：camelCase 风格（API 响应自动转换）
- **数据库表**：snake_case 复数形式
  - `biz_accounts`、`biz_projects`

### 2. 软删除机制

所有业务表均包含 `is_deleted` 字段：
- `is_deleted = 0`：正常数据
- `is_deleted = 1`：已删除

### 3. 多租户隔离

- **SaaS 模式**：行级隔离，通过 `tenant_id` 字段隔离
- **独立部署**：独立数据库，完全隔离

---

## 快速导航

### 开发规范
- [Python 代码规范](./STANDARDS/PYTHON/python_code_standard.md)
- [React 代码规范](./STANDARDS/REACT/react_code_standard.md)
- [CSS 样式规范](./STANDARDS/CSS/css_standard.md)
- [SQL 规范](./STANDARDS/DB/sql_standard.md)
- [API 设计规范](./STANDARDS/API/api_design_standard.md)

### 技术规格
- [系统架构](./SPECS/SYSTEM/system_architecture.md)
- [租户数据库设计](./SPECS/TENANT/tenant_db.md)
- [测试规范](./SPECS/TESTING/testing_standard.md)
- [部署手册](./SPECS/DEPLOY/deployment_detailed.md)

### 原型设计
- [SaaS 管理平台原型](./PROTOTYPES/SAAS/prototype_saas.md)
- [租户系统原型](./PROTOTYPES/TENANT/prototype_tenant.md)

### 投标文档
- [投标技术文档](./BID_DOCUMENTATION/smartaudit_bid_document.md)

---

## 版本说明

| 版本 | 日期 | 说明 |
|------|------|------|
| V1.0 | 2026-03-20 | 初始版本，完善文档结构 |

---

## 文档维护

本文档体系按照以下原则维护：

1. **一致性**：所有技术文档遵循统一的命名规范和编码风格
2. **可操作性**：部署文档包含详细的命令和步骤
3. **完整性**：覆盖开发、测试、部署、运维全流程
4. **可追溯性**：重要决策记录变更历史
