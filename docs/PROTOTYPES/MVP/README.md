# SmartAudit MVP 原型

最小可行产品原型，聚焦核心审核工作流。基于 Google Material Design 风格设计，可直接在浏览器中预览或导入 Axure RP 使用。

## 目录结构

```
MVP/
├── README.md              # 本文件
├── style.css              # 共享样式
├── login.html             # 登录页
├── dashboard.html         # 工作台首页
├── projects.html          # 项目管理
├── project_detail.html    # 项目详情/配置
├── rule_builder.html      # 规则配置
├── audit.html             # 审核任务提交与结果查询
├── task_review.html       # 人工复核
└── statistics.html        # 数据统计
```

## 页面说明

| 页面 | 功能 | 路径 |
|------|------|------|
| [login.html](login.html) | 管理员登录 | 登录页 |
| [dashboard.html](dashboard.html) | 工作台首页 | 工作台 |
| [projects.html](projects.html) | 项目管理列表 | 项目管理 |
| [project_detail.html](project_detail.html) | 项目详情与配置 | 项目管理 |
| [rule_builder.html](rule_builder.html) | 审核规则配置 | 审核任务 |
| [audit.html](audit.html) | 审核任务提交与结果查询 | 审核任务 |
| [task_review.html](task_review.html) | 人工复核操作 | 审核任务 |
| [statistics.html](statistics.html) | 数据统计报表 | 数据统计 |

## 核心功能

- **项目创建**: 支持自动审核 / 人工审核 / 混合模式
- **规则配置**: 可视化条件组、条件项嵌套配置
- **数据提交**: JSON 格式提交审核数据
- **规则执行**: 展示各规则执行结果（通过/复核/驳回）
- **人工复核**: 待复核任务队列、复核操作、意见记录
- **数据统计**: 审核量趋势、结果分布、项目对比、驳回原因

## 核心流程

```
登录 → 工作台 → 项目管理 → 规则配置
                      ↓
              审核任务 → 提交数据 → 自动审核 → 结果展示
                                    ↓
                              触发复核 → 人工复核 → 最终结论
```

## 设计风格

- **设计规范**: Google Material Design
- **主色调**: #1a73e8 (蓝)
- **辅助色**: #34a853 (成功) / #f9ab00 (警告) / #ea4335 (危险)
- **字体**: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto
- **布局**: 侧边栏导航 + 主内容区
- **响应式**: 桌面端 1280px+ 优化

## 技术栈（参考）

- FastAPI + PostgreSQL + Redis + Celery
- React 18 + Tailwind CSS + TypeScript
- Docker Compose 一键部署

## Axure 导入说明

1. 打开 Axure RP
2. 选择 `File > Open` 选择 `login.html` 或其他页面文件
3. 各页面独立，可单独导入使用
4. 共享样式在 `style.css` 中定义

---

© 2024 SmartAudit 智核
