# CSS / 样式规范

## 1. 项目样式结构

```
src/styles/
├── _variables.css      # CSS 变量定义
├── _mixins.css         # 混入宏
├── _reset.css          # CSS 重置
├── _common.css         # 全局通用样式
├── _animations.css     # 动画定义
└── global.css          # 主入口文件
```

## 2. CSS 变量规范

### 2.1 变量命名
- 使用 kebab-case
- 以 `--` 前缀开头
- 分类前缀：`--color-*`、`--font-*`、`--spacing-*`

### 2.2 变量定义示例
```css
/* _variables.css */
:root {
  /* 主色调 */
  --color-primary: #1890ff;
  --color-primary-hover: #40a9ff;
  --color-primary-active: #096dd9;

  /* 功能色 */
  --color-success: #52c41a;
  --color-warning: #faad14;
  --color-error: #ff4d4f;
  --color-info: #1890ff;

  /* 中性色 */
  --color-text-primary: rgba(0, 0, 0, 0.85);
  --color-text-secondary: rgba(0, 0, 0, 0.65);
  --color-text-disabled: rgba(0, 0, 0, 0.25);
  --color-border: #d9d9d9;
  --color-bg: #ffffff;
  --color-bg-gray: #f5f5f5;

  /* 字体 */
  --font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    'Helvetica Neue', Arial, sans-serif;
  --font-size-xs: 12px;
  --font-size-sm: 14px;
  --font-size-base: 14px;
  --font-size-lg: 16px;
  --font-size-xl: 20px;

  /* 间距 */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-base: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  /* 圆角 */
  --border-radius-sm: 2px;
  --border-radius-base: 4px;
  --border-radius-lg: 8px;

  /* 阴影 */
  --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.08);
  --shadow-base: 0 4px 12px rgba(0, 0, 0, 0.12);
  --shadow-lg: 0 8px 24px rgba(0, 0, 0, 0.16);

  /* 过渡 */
  --transition-fast: 0.15s ease;
  --transition-base: 0.3s ease;
  --transition-slow: 0.5s ease;
}
```

## 3. 样式书写规范

### 3.1 选择器规范
```css
/* Good */
.tenant-card { }
.tenant-card__header { }
.tenant-card__title--active { }

/* Avoid */
.TenantCard { }
.tenantCard { }
.tenant-card .header .title { }
```

### 3.2 样式顺序
```css
.selector {
  /* 定位 */
  position: absolute;
  top: 0;
  left: 0;

  /* 盒模型 */
  display: flex;
  width: 100px;
  height: 100px;
  padding: 10px;
  margin: 10px;
  border: 1px solid #ddd;

  /* 文字 */
  font-size: 14px;
  color: #333;
  text-align: center;

  /* 背景 */
  background-color: #fff;

  /* 其他 */
  opacity: 1;
  cursor: pointer;
  transition: all 0.3s;
}
```

### 3.3 嵌套规范
- CSS Module/less/scss：最多 3 层嵌套
- 普通 CSS：不使用嵌套

```scss
// Good (SCSS)
.tenant-card {
  padding: 16px;

  &__header {
    display: flex;
    align-items: center;

    &--active {
      background: #f5f5f5;
    }
  }

  &__title {
    font-size: 16px;
  }
}

// Avoid
.tenant-card {
  .tenant-card__header {
    .tenant-card__title {
      font-size: 16px;
    }
  }
}
```

## 4. BEM 命名规范

### 4.1 BEM 概念
- Block：模块容器
- Element：模块内部元素
- Modifier：模块状态/变体

### 4.2 命名格式
```css
.block { }                    /* 模块 */
.block__element { }           /* 子元素 */
.block--modifier { }           /* 状态变体 */
.block__element--modifier { } /* 子元素状态 */
```

### 4.3 实际示例
```html
<!-- 租户卡片模块 -->
<div class="tenant-card tenant-card--featured">
  <div class="tenant-card__header">
    <h3 class="tenant-card__title">租户名称</h3>
    <span class="tenant-card__status tenant-card__status--active">启用</span>
  </div>
  <div class="tenant-card__body">
    <p class="tenant-card__desc">租户描述信息</p>
  </div>
  <div class="tenant-card__footer">
    <button class="tenant-card__btn tenant-card__btn--primary">编辑</button>
    <button class="tenant-card__btn tenant-card__btn--danger">删除</button>
  </div>
</div>
```

```css
.tenant-card {
  padding: 16px;
  border: 1px solid #e8e8e8;
  border-radius: 8px;

  &--featured {
    border-color: var(--color-primary);
    box-shadow: var(--shadow-base);
  }

  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
  }

  &__title {
    font-size: 16px;
    font-weight: 500;
  }

  &__status {
    padding: 2px 8px;
    font-size: 12px;
    border-radius: 4px;

    &--active {
      background: #e6f7ff;
      color: var(--color-primary);
    }

    &--disabled {
      background: #f5f5f5;
      color: #999;
    }
  }

  &__footer {
    display: flex;
    gap: 8px;
    margin-top: 16px;
  }

  &__btn {
    padding: 6px 16px;
    border-radius: 4px;

    &--primary {
      background: var(--color-primary);
      color: #fff;
    }

    &--danger {
      background: var(--color-error);
      color: #fff;
    }
  }
}
```

## 5. 响应式断点

### 5.1 断点定义
```css
/* 移动端优先 */
/* Extra small devices (phones, 576px and down) */
/* @media (max-width: 575.98px) { ... } */

/* Small devices (landscape phones, 576px and up) */
@media (min-width: 576px) { ... }

/* Medium devices (tablets, 768px and up) */
@media (min-width: 768px) { ... }

/* Large devices (desktops, 992px and up) */
@media (min-width: 992px) { ... }

/* Extra large devices (large desktops, 1200px and up) */
@media (min-width: 1200px) { ... }

/* Extra extra large devices (larger desktops, 1400px and up) */
@media (min-width: 1400px) { ... }
```

### 5.2 SCSS 混入
```scss
// _mixins.scss
$breakpoints: (
  'sm': 576px,
  'md': 768px,
  'lg': 992px,
  'xl': 1200px,
  'xxl': 1400px,
);

@mixin respond-to($breakpoint) {
  @if map-has-key($breakpoints, $breakpoint) {
    @media (min-width: map-get($breakpoints, $breakpoint)) {
      @content;
    }
  }
}

// 使用
.card {
  padding: 16px;

  @include respond-to('md') {
    padding: 24px;
  }

  @include respond-to('lg') {
    padding: 32px;
    display: flex;
  }
}
```

## 6. 动画规范

### 6.1 动画时间
| 类型 | 时长 | 使用场景 |
|------|------|----------|
| fast | 0.15s | 微交互、hover |
| base | 0.3s | 常规过渡 |
| slow | 0.5s | 页面切换、大型元素 |

### 6.2 动画定义
```css
/* _animations.scss */
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 动画类 */
.animate-fade-in {
  animation: fadeIn var(--transition-base) ease-out;
}

.animate-slide-up {
  animation: slideUp var(--transition-base) ease-out;
}

.animate-spin {
  animation: spin 1s linear infinite;
}
```

## 7. 通用样式类

### 7.1 布局类
```css
/* _common.css */

/* Flex */
.flex { display: flex; }
.flex-col { flex-direction: column; }
.flex-wrap { flex-wrap: wrap; }
.items-center { align-items: center; }
.items-start { align-items: flex-start; }
.items-end { align-items: flex-end; }
.justify-center { justify-content: center; }
.justify-between { justify-content: space-between; }
.justify-end { justify-content: flex-end; }
.flex-1 { flex: 1; }
.gap-1 { gap: 4px; }
.gap-2 { gap: 8px; }
.gap-3 { gap: 12px; }
.gap-4 { gap: 16px; }

/* Grid */
.grid { display: grid; }
.grid-cols-2 { grid-template-columns: repeat(2, 1fr); }
.grid-cols-3 { grid-template-columns: repeat(3, 1fr); }
.grid-cols-4 { grid-template-columns: repeat(4, 1fr); }

/* Spacing */
.m-0 { margin: 0; }
.m-auto { margin: auto; }
.mt-1 { margin-top: 4px; }
.mt-2 { margin-top: 8px; }
.mt-4 { margin-top: 16px; }
.mb-1 { margin-bottom: 4px; }
.mb-2 { margin-bottom: 8px; }
.mb-4 { margin-bottom: 16px; }
.p-0 { padding: 0; }
.p-2 { padding: 8px; }
.p-4 { padding: 16px; }
.px-2 { padding-left: 8px; padding-right: 8px; }
.px-4 { padding-left: 16px; padding-right: 16px; }
.py-2 { padding-top: 8px; padding-bottom: 8px; }
.py-4 { padding-top: 16px; padding-bottom: 16px; }

/* Text */
.text-center { text-align: center; }
.text-right { text-align: right; }
.text-sm { font-size: 12px; }
.text-base { font-size: 14px; }
.text-lg { font-size: 16px; }
.text-xl { font-size: 20px; }
.font-medium { font-weight: 500; }
.font-bold { font-weight: 700; }
.text-primary { color: var(--color-primary); }
.text-success { color: var(--color-success); }
.text-warning { color: var(--color-warning); }
.text-error { color: var(--color-error); }
.text-disabled { color: var(--color-text-disabled); }

/* Display */
.hidden { display: none; }
.block { display: block; }
.inline-block { display: inline-block; }

/* Position */
.relative { position: relative; }
.absolute { position: absolute; }
.fixed { position: fixed; }
.sticky { position: sticky; }
.top-0 { top: 0; }
.left-0 { left: 0; }
.right-0 { right: 0; }
.bottom-0 { bottom: 0; }

/* Size */
.w-full { width: 100%; }
.h-full { height: 100%; }
.min-h-screen { min-height: 100vh; }

/* Border */
.rounded { border-radius: var(--border-radius-base); }
.rounded-lg { border-radius: var(--border-radius-lg); }
.rounded-full { border-radius: 9999px; }
.border { border: 1px solid var(--color-border); }

/* Shadow */
.shadow-sm { box-shadow: var(--shadow-sm); }
.shadow { box-shadow: var(--shadow-base); }
.shadow-lg { box-shadow: var(--shadow-lg); }

/* Overflow */
.overflow-hidden { overflow: hidden; }
.overflow-auto { overflow: auto; }
.overflow-x-auto { overflow-x: auto; }
.overflow-y-auto { overflow-y: auto; }

/* Cursor */
.cursor-pointer { cursor: pointer; }
.cursor-not-allowed { cursor: not-allowed; }

/* Opacity */
.opacity-50 { opacity: 0.5; }
.opacity-75 { opacity: 0.75; }

/* Transitions */
.transition { transition: all var(--transition-base); }
.transition-fast { transition: all var(--transition-fast); }
```

## 8. Tailwind CSS 配置规范

### 8.1 tailwind.config.js
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: '#1890ff',
          hover: '#40a9ff',
          active: '#096dd9',
        },
      },
      spacing: {
        18: '4.5rem',
        88: '22rem',
      },
      borderRadius: {
        xl: '1rem',
        '2xl': '1.5rem',
      },
    },
  },
  plugins: [],
};
```

### 8.2 使用规范
```tsx
// 使用 Tailwind 类名
<div className="flex items-center justify-between p-4 bg-white rounded-lg shadow">
  <h3 className="text-lg font-medium text-gray-900">标题</h3>
  <div className="flex gap-2">
    <Button size="small">编辑</Button>
  </div>
</div>

// 条件类名 - 使用 clsx/classnames
import { clsx } from 'clsx';

<Button
  className={clsx(
    'w-full',
    isLoading && 'opacity-50 cursor-not-allowed'
  )}
>
  提交
</Button>

// 组合复杂样式 - 使用 crxcnva
import { crx } from '@radix-ui/themes';

<div className={crx('flex items-center', isActive && 'bg-primary')}>
  内容
</div>
```

## 9. 样式检查

### 9.1 stylelint 配置
```json
{
  "extends": [
    "stylelint-config-standard",
    "stylelint-config-rational-order",
    "stylelint-prettier/recommended"
  ],
  "plugins": [
    "stylelint-scss"
  ],
  "rules": {
    "selector-class-pattern": "^[a-z][a-z0-9]*(-[a-z0-9]+)*(__[a-z0-9]+(-[a-z0-9]+)*)?(--[a-z0-9]+(-[a-z0-9]+)*)?$",
    "max-nesting-depth": 3,
    "no-descending-specificity": null
  }
}
```

### 9.2 package.json scripts
```json
{
  "scripts": {
    "lint:css": "stylelint \"src/**/*.{css,scss}\" --fix",
    "lint:style": "npm run lint:css"
  }
}
```
