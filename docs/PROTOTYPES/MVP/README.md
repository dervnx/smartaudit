# SmartAudit MVP 原型

最小可行产品原型，聚焦核心审核工作流。

## 核心流程

```
登录 → 工作台 → 项目管理 → 审核任务 → 审核结果
```

## 页面说明

| 页面 | 功能 |
|------|------|
| [login.html](login.html) | 管理员登录 |
| [dashboard.html](dashboard.html) | 工作台首页 |
| [projects.html](projects.html) | 项目管理（含新建项目弹窗） |
| [audit.html](audit.html) | 审核任务提交与结果查询 |

## 核心功能

- **项目创建**: 支持自动审核 / 人工审核 / 混合模式
- **数据提交**: JSON 格式提交审核数据
- **规则执行**: 展示各规则执行结果（通过/复核/驳回）
- **结果展示**: 实时显示审核状态、耗时、规则明细

## 设计风格

- 极简主义，仅保留核心元素
- 主色调: #1a73e8
- 响应式适配桌面端

## 技术栈（参考）

- FastAPI + PostgreSQL + Redis + Celery
- Docker Compose 一键部署

---

© 2024 SmartAudit 智核
