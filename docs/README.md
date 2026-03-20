# 智核 SmartAudit 文档体系

## 文档结构

```
docs/
├── README.md                    # 文档总览
│
├── PRD/                        # 产品需求文档
│   ├── OVERVIEW.md            # 产品概述与愿景
│   ├── FUNCTION_MAP.md        # 功能拆解地图
│   ├── SAAS_ADMIN_PRD.md      # SaaS后台需求
│   ├── TENANT_ADMIN_PRD.md    # 租户后台需求
│   └── VERSION_PLAN.md         # 版本规划
│
├── SAAS/                       # SaaS 服务端设计
│   ├── SAAS_SYSTEM.md        # SaaS 系统架构
│   ├── SAAS_DB.md            # SaaS 数据库设计
│   ├── SAAS_API.md           # SaaS API 规范
│   ├── SAAS_ADMIN_PANEL.md   # SaaS 管理后台
│   └── SAAS_DEPLOY.md        # SaaS 部署方案
│
├── TENANT/                    # 租户服务端设计
│   ├── TENANT_SYSTEM.md      # 租户系统架构
│   ├── TENANT_DB.md          # 租户数据库设计
│   ├── TENANT_API.md         # 租户 API 规范
│   ├── TENANT_DEPLOY.md      # 租户部署方案
│   ├── TENANT_BACKUP.md      # 备份管理系统
│   └── TENANT_VERSION.md     # 版本管理系统
│
├── SHARED/                    # 共享设计
│   ├── COMMON_TYPES.md        # 公共数据类型
│   ├── AUTH_DESIGN.md        # 认证授权设计
│   ├── REDIS_DESIGN.md       # Redis 缓存与队列
│   ├── COMMON_SCHEMAS.md     # 公共 Schema 定义
│   └── ERROR_CODES.md        # 错误码规范
│
├── FRONTEND/                  # 前端开发规范
│   ├── COMMON_SPEC.md        # 公共开发规范
│   ├── PC_WEB_SPEC.md       # PC Web 规范
│   ├── MOBILE_SPEC.md       # 移动端 H5 规范
│   └── MINI_PROGRAM_SPEC.md # 小程序规范
│
└── DEPLOY/                    # 部署相关
    ├── DOCKER_COMPOSE.md    # Docker Compose 部署
    ├── KUBERNETES.md        # Kubernetes 部署
    └── ENV_TEMPLATE.md       # 环境变量模板
```

## 系统架构概览

### 双系统模式

```
┌─────────────────────────────────────────────────────────────────┐
│                         SmartAudit                               │
├───────────────────────────┬─────────────────────────────────────┤
│                           │                                      │
│      SaaS 管理平台         │         租户管理系统                 │
│   (平台运营方使用)          │      (各租户独立使用)                │
│                           │                                      │
│  ┌─────────────────────┐  │  ┌─────────────────────────────┐   │
│  │ - 租户开通/管理       │  │  │ - 项目管理                  │   │
│  │ - 版本发布管理        │  │  │ - 规则引擎                  │   │
│  │ - 系统升级管理        │  │  │ - 任务审核                  │   │
│  │ - 全局配置           │  │  │ - 三方接口                  │   │
│  │ - 运营统计           │  │  │ - 数据统计                  │   │
│  └─────────────────────┘  │  │ - 备份管理                  │   │
│                           │  │ - 版本管理 (独立部署)         │   │
│                           │  └─────────────────────────────┘   │
│                           │                                      │
└───────────────────────────┴─────────────────────────────────────┘
```

### 部署模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| SaaS | 平台统一托管，多租户共享 | 中小企业 |
| 独立部署 | 租户独立服务器 | 大型企业/政府 |

## 技术栈

| 层级 | 技术选型 |
|------|----------|
| 前端 | React 18 + Tailwind CSS + Vite + TypeScript |
| 后端 | Python 3.11 + FastAPI + SQLAlchemy 2.x |
| 数据库 | PostgreSQL 15+ |
| 缓存/队列 | Redis 7+ (Celery) |
| 文件存储 | MinIO / S3 / OSS |

## 快速导航

- [产品需求总览](./PRD/OVERVIEW.md)
- [SaaS 管理平台设计](./SAAS/SAAS_SYSTEM.md)
- [租户管理系统设计](./TENANT/TENANT_SYSTEM.md)
- [Redis 缓存与队列设计](./SHARED/REDIS_DESIGN.md)
- [前端开发规范](./FRONTEND/COMMON_SPEC.md)
- [部署方案](./DEPLOY/DOCKER_COMPOSE.md)
