# SmartAudit Axure 原型

本目录包含 SmartAudit 系统的 Axure 原型设计文件，可导入 Axure RP 使用。

## 目录结构

```
AXURE/
├── README.md                    # 本文件
├── _data/
│   └── style.css                # 共享样式 (CSS变量)
├── _masters/
│   ├── header.html              # 通用页眉母版
│   ├── sidebar.html             # 通用侧边栏母版
│   └── footer.html              # 通用页脚母版
└── pages/
    ├── saas/                    # SaaS 管理平台页面
    │   ├── login.html           # 登录页
    │   ├── dashboard.html       # 运营概览仪表盘
    │   ├── tenant_list.html     # 租户管理列表
    │   ├── create_tenant.html   # 开通租户表单
    │   ├── version_list.html    # 版本管理
    │   ├── system_config.html   # 系统配置
    │   └── statistics.html      # 运营统计
    └── tenant/                  # 租户系统页面
        ├── login.html           # 租户登录页
        ├── dashboard.html       # 工作台
        ├── project_list.html    # 项目列表
        ├── project_detail.html  # 项目详情
        ├── rule_builder.html    # 规则配置
        ├── task_audit.html      # 任务审核
        ├── statistics.html      # 统计报表
        ├── account_management.html  # 账户管理
        ├── third_party_config.html  # 第三方配置
        ├── version_info.html    # 版本信息
        └── data_backup.html     # 数据备份
```

## 导入 Axure

1. 打开 Axure RP
2. 选择 `File > Open` 或 `Import from Axure Cloud`
3. 选择本目录下的 `.html` 文件导入
4. 母版文件在 `_masters/` 目录，页面文件在 `pages/` 目录

## 设计说明

- **设计风格**: Google Material Design
- **主色调**: #1a73e8 (蓝)
- **字体**: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto
- **布局**: 侧边栏导航 + 主内容区
- **响应式**: 支持桌面端 1280px+ 宽度

## 页面说明

### SaaS 管理平台

| 页面 | 功能 |
|------|------|
| login | 平台管理员登录 |
| dashboard | 运营概览仪表盘 |
| tenant_list | 租户列表管理 |
| create_tenant | 开通租户表单 |
| version_list | 版本套餐管理 |
| system_config | 系统参数配置 |
| statistics | 运营数据统计 |

### 租户系统

| 页面 | 功能 |
|------|------|
| login | 租户用户登录 |
| dashboard | 工作台首页 |
| project_list | 项目列表 |
| project_detail | 项目详情/配置 |
| rule_builder | 审核规则配置 |
| task_audit | 人工复核任务 |
| statistics | 数据统计报表 |
| account_management | 账户权限管理 |
| third_party_config | 第三方服务配置 |
| version_info | 版本与用量信息 |
| data_backup | 备份恢复管理 |

---

© 2024 SmartAudit 智核
