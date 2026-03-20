# React 代码规范

## 1. 项目结构

```
smartaudit-web/
├── src/
│   ├── api/                     # API 请求定义
│   │   ├── request.ts           # axios 实例
│   │   ├── tenant/
│   │   │   ├── index.ts         # 租户相关 API
│   │   │   └── types.ts         # 租户类型定义
│   │   ├── project/
│   │   │   ├── index.ts
│   │   │   └── types.ts
│   │   ├── rule/
│   │   ├── task/
│   │   ├── account/
│   │   └── statistics/
│   ├── components/              # 公共组件
│   │   ├── common/             # 通用组件
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   ├── Table/
│   │   │   ├── Modal/
│   │   │   └── Form/
│   │   ├── business/           # 业务组件
│   │   │   ├── TenantSelector/
│   │   │   ├── ProjectCard/
│   │   │   └── RuleBuilder/
│   │   └── layout/             # 布局组件
│   │       ├── Header/
│   │       ├── Sidebar/
│   │       └── Footer/
│   ├── pages/                   # 页面组件
│   │   ├── Dashboard/
│   │   ├── Tenant/
│   │   │   ├── List/
│   │   │   ├── Detail/
│   │   │   └── Create/
│   │   ├── Project/
│   │   ├── Rule/
│   │   ├── Task/
│   │   │   ├── Audit/
│   │   │   └── List/
│   │   ├── Statistics/
│   │   ├── Account/
│   │   └── System/
│   ├── hooks/                   # 自定义 Hooks
│   │   ├── useTenant.ts
│   │   ├── useProject.ts
│   │   ├── usePagination.ts
│   │   └── usePermission.ts
│   ├── store/                   # 状态管理
│   │   ├── index.ts
│   │   ├── userStore.ts
│   │   ├── tenantStore.ts
│   │   └── appStore.ts
│   ├── router/                  # 路由配置
│   │   ├── index.ts
│   │   ├── routes.ts
│   │   └── guards.ts
│   ├── utils/                   # 工具函数
│   │   ├── request.ts
│   │   ├── storage.ts
│   │   ├── date.ts
│   │   └── validation.ts
│   ├── types/                   # 全局类型定义
│   │   ├── global.d.ts
│   │   ├── api.d.ts
│   │   └── store.d.ts
│   ├── styles/                  # 全局样式
│   │   ├── variables.css
│   │   ├── reset.css
│   │   └── global.css
│   ├── App.tsx
│   └── main.tsx
├── public/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── package.json
├── tsconfig.json
├── vite.config.ts
└── README.md
```

## 2. 命名规范

### 2.1 文件命名
- 组件文件：PascalCase + index.tsx 组合
  - `components/Button/index.tsx`
  - `pages/Tenant/List/index.tsx`
- 普通文件：kebab-case
  - `use-pagination.ts`
  - `date-utils.ts`
- 类型文件：kebab-case + `.types.ts`
  - `tenant.types.ts`
  - `api.types.ts`

### 2.2 组件命名
- 组件名：PascalCase
  - `TenantList`、`ProjectCard`
- 组件文件：PascalCase + index.tsx
  - `TenantList/index.tsx`
- Hooks：camelCase，以 use 开头
  - `useTenantList`
  - `usePermission`

### 2.3 变量命名
- 组件内变量：camelCase
- Props：camelCase
- 常量：UPPER_SNAKE_CASE
- 布尔变量：is、has、should、can 前缀
  - `isLoading`、`hasError`、`isVisible`

### 2.4 CSS 类名命名
- 使用 BEM 命名规范
- 格式：`block__element--modifier`
- 示例：
  ```css
  .tenant-list__item--active { }
  .button--primary { }
  .form__input--error { }
  ```

## 3. 组件规范

### 3.1 函数组件
```tsx
// Good
interface TenantCardProps {
  tenant: Tenant;
  onEdit: (id: number) => void;
  onDelete: (id: number) => void;
}

export const TenantCard: React.FC<TenantCardProps> = ({
  tenant,
  onEdit,
  onDelete,
}) => {
  const [isLoading, setIsLoading] = useState(false);

  const handleDelete = async () => {
    setIsLoading(true);
    try {
      await deleteTenant(tenant.id);
      onDelete(tenant.id);
    } catch (error) {
      message.error('删除失败');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="tenant-card">
      <h3>{tenant.name}</h3>
      <Button onClick={() => onEdit(tenant.id)}>编辑</Button>
      <Button loading={isLoading} onClick={handleDelete}>删除</Button>
    </div>
  );
};
```

### 3.2 Props 类型定义
```tsx
// 使用 interface 定义组件 Props
interface ButtonProps {
  type?: 'primary' | 'default' | 'danger';
  size?: 'small' | 'medium' | 'large';
  loading?: boolean;
  disabled?: boolean;
  children: React.ReactNode;
  onClick?: (e: React.MouseEvent) => void;
}

// 事件处理使用联合类型
interface InputProps {
  onChange?: (value: string) => void;
  onFocus?: () => void;
  onBlur?: () => void;
}
```

### 3.3 组件拆分原则
- 单一职责：每个组件只做一件事
- 合理粒度：20-200 行代码为合适
- 提取标准：重复代码超过 2 次应提取
- 业务组件与展示组件分离

## 4. 状态管理

### 4.1 Zustand Store
```typescript
// store/tenantStore.ts
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

interface TenantState {
  tenants: Tenant[];
  currentTenant: Tenant | null;
  isLoading: boolean;
  error: string | null;

  // Actions
  fetchTenants: () => Promise<void>;
  setCurrentTenant: (tenant: Tenant | null) => void;
  createTenant: (data: CreateTenantDTO) => Promise<Tenant>;
  updateTenant: (id: number, data: UpdateTenantDTO) => Promise<void>;
  deleteTenant: (id: number) => Promise<void>;
}

export const useTenantStore = create<TenantState>()(
  devtools(
    (set, get) => ({
      tenants: [],
      currentTenant: null,
      isLoading: false,
      error: null,

      fetchTenants: async () => {
        set({ isLoading: true, error: null });
        try {
          const data = await tenantApi.list();
          set({ tenants: data, isLoading: false });
        } catch (error) {
          set({ error: (error as Error).message, isLoading: false });
        }
      },

      setCurrentTenant: (tenant) => set({ currentTenant: tenant }),

      createTenant: async (data) => {
        const newTenant = await tenantApi.create(data);
        set((state) => ({ tenants: [...state.tenants, newTenant] }));
        return newTenant;
      },

      updateTenant: async (id, data) => {
        await tenantApi.update(id, data);
        set((state) => ({
          tenants: state.tenants.map((t) =>
            t.id === id ? { ...t, ...data } : t
          ),
        }));
      },

      deleteTenant: async (id) => {
        await tenantApi.delete(id);
        set((state) => ({
          tenants: state.tenants.filter((t) => t.id !== id),
        }));
      },
    }),
    { name: 'tenant-store' }
  )
);
```

### 4.2 数据获取 Hook
```typescript
// hooks/useTenantList.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { tenantApi } from '@/api/tenant';

export const useTenantList = () => {
  return useQuery({
    queryKey: ['tenants'],
    queryFn: () => tenantApi.list(),
    staleTime: 5 * 60 * 1000, // 5 分钟
  });
};

export const useCreateTenant = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateTenantDTO) => tenantApi.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tenants'] });
    },
  });
};
```

## 5. API 请求规范

### 5.1 Axios 实例配置
```typescript
// api/request.ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import { message } from 'antd';
import { getToken, refreshToken } from '@/utils/auth';

const request: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器
request.interceptors.request.use(
  (config) => {
    const token = getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 响应拦截器
request.interceptors.response.use(
  (response) => {
    const { code, data, message: msg } = response.data;

    if (code === 0) {
      return data;
    }

    // 业务错误
    message.error(msg || '请求失败');
    return Promise.reject(new Error(msg));
  },
  async (error: AxiosError) => {
    if (error.response?.status === 401) {
      // Token 过期，尝试刷新
      try {
        await refreshToken();
        // 重试原请求
        return request(error.config!);
      } catch {
        // 刷新失败，跳转登录
        window.location.href = '/login';
      }
    }

    message.error('网络错误，请稍后重试');
    return Promise.reject(error);
  }
);

export default request;
```

### 5.2 API 模块定义
```typescript
// api/tenant/types.ts
export interface Tenant {
  id: number;
  name: string;
  code: string;
  logo?: string;
  contact: string;
  phone: string;
  email: string;
  status: 'active' | 'disabled';
  created_at: string;
  updated_at: string;
}

export interface CreateTenantDTO {
  name: string;
  code: string;
  contact: string;
  phone: string;
  email: string;
}

export interface UpdateTenantDTO extends Partial<CreateTenantDTO> {
  status?: 'active' | 'disabled';
}

// api/tenant/index.ts
import request from '../request';
import type { Tenant, CreateTenantDTO, UpdateTenantDTO } from './types';

export const tenantApi = {
  list: (params?: { page?: number; page_size?: number }) =>
    request.get<{ items: Tenant[]; total: number }>('/tenants', { params }),

  getById: (id: number) =>
    request.get<Tenant>(`/tenants/${id}`),

  create: (data: CreateTenantDTO) =>
    request.post<Tenant>('/tenants', data),

  update: (id: number, data: UpdateTenantDTO) =>
    request.put<Tenant>(`/tenants/${id}`, data),

  delete: (id: number) =>
    request.delete(`/tenants/${id}`),

  enable: (id: number) =>
    request.post(`/tenants/${id}/enable`),

  disable: (id: number) =>
    request.post(`/tenants/${id}/disable`),
};
```

## 6. 样式规范

### 6.1 CSS Modules
```css
/* TenantCard.module.css */
.card {
  padding: 16px;
  border: 1px solid #e8e8e8;
  border-radius: 8px;
  transition: all 0.3s;
}

.card:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.title {
  font-size: 16px;
  font-weight: 500;
}

.actions {
  display: flex;
  gap: 8px;
}
```

```tsx
// TenantCard/index.tsx
import styles from './TenantCard.module.css';

export const TenantCard: React.FC = () => {
  return (
    <div className={styles.card}>
      <div className={styles.header}>
        <span className={styles.title}>标题</span>
      </div>
      <div className={styles.actions}>
        <Button>操作</Button>
      </div>
    </div>
  );
};
```

### 6.2 Tailwind CSS 使用
```tsx
// 使用 className 组合
<div className="flex items-center justify-between p-4 bg-white rounded-lg shadow">
  <h3 className="text-lg font-medium text-gray-900">标题</h3>
  <div className="flex gap-2">
    <Button size="small">编辑</Button>
    <Button size="small" danger>删除</Button>
  </div>
</div>

// 条件类名
<Button
  type="primary"
  className={cn(
    'w-full',
    isLoading && 'opacity-50 cursor-not-allowed'
  )}
>
  提交
</Button>
```

## 7. 路由规范

### 7.1 路由配置
```typescript
// router/routes.ts
export interface RouteConfig {
  path: string;
  name: string;
  component: React.ComponentType;
  meta?: {
    title?: string;
    requiresAuth?: boolean;
    permissions?: string[];
    keepAlive?: boolean;
  };
  children?: RouteConfig[];
}

export const routes: RouteConfig[] = [
  {
    path: '/',
    name: 'Layout',
    component: Layout,
    meta: { requiresAuth: true },
    children: [
      {
        path: '/dashboard',
        name: 'Dashboard',
        component: Dashboard,
        meta: { title: '工作台' },
      },
      {
        path: '/tenant',
        name: 'Tenant',
        meta: {
          title: '租户管理',
          permissions: ['TENANT_VIEW'],
        },
        children: [
          {
            path: '/tenant/list',
            name: 'TenantList',
            component: TenantList,
            meta: { title: '租户列表' },
          },
          {
            path: '/tenant/create',
            name: 'TenantCreate',
            component: TenantCreate,
            meta: { title: '创建租户' },
          },
        ],
      },
    ],
  },
];
```

### 7.2 路由守卫
```typescript
// router/guards.ts
import { useAuthStore } from '@/store/authStore';
import { usePermissionStore } from '@/store/permissionStore';

export const useAuthGuard = (route: RouteConfig) => {
  const { isAuthenticated, user } = useAuthStore();
  const { hasPermission } = usePermissionStore();

  if (route.meta?.requiresAuth && !isAuthenticated) {
    return { allowed: false, redirect: '/login' };
  }

  if (route.meta?.permissions) {
    const hasAllPermissions = route.meta.permissions.every((p) =>
      hasPermission(p)
    );
    if (!hasAllPermissions) {
      return { allowed: false, redirect: '/403' };
    }
  }

  return { allowed: true };
};
```

## 8. 权限控制

### 8.1 权限 Hook
```typescript
// hooks/usePermission.ts
import { usePermissionStore } from '@/store/permissionStore';

export const usePermission = () => {
  const { permissions, userPermissions } = usePermissionStore();

  const hasPermission = (code: string) => {
    return userPermissions.includes(code);
  };

  const hasAnyPermission = (codes: string[]) => {
    return codes.some((code) => userPermissions.includes(code));
  };

  const hasAllPermissions = (codes: string[]) => {
    return codes.every((code) => userPermissions.includes(code));
  };

  return {
    hasPermission,
    hasAnyPermission,
    hasAllPermissions,
  };
};

// 使用示例
const { hasPermission } = usePermission();

{
  hasPermission('TENANT_CREATE') && (
    <Button type="primary" onClick={handleCreate}>
      创建租户
    </Button>
  )
}
```

### 8.2 权限组件
```typescript
// components/common/Permission/index.tsx
interface PermissionProps {
  code: string;
  mode?: 'all' | 'any';
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export const Permission: React.FC<PermissionProps> = ({
  code,
  mode = 'any',
  children,
  fallback = null,
}) => {
  const { hasPermission, hasAllPermissions } = usePermission();

  const hasAccess =
    mode === 'all'
      ? hasAllPermissions(code.split(','))
      : hasPermission(code);

  return hasAccess ? <>{children}</> : <>{fallback}</>;
};

// 使用示例
<Permission code="TENANT_DELETE" fallback={<Button disabled>删除</Button>}>
  <Button danger onClick={handleDelete}>删除</Button>
</Permission>
```

## 9. 分页组件

### 9.1 分页 Hook
```typescript
// hooks/usePagination.ts
interface PaginationOptions {
  defaultPage?: number;
  defaultPageSize?: number;
}

export const usePagination = (options?: PaginationOptions) => {
  const [page, setPage] = useState(options?.defaultPage || 1);
  const [pageSize, setPageSize] = useState(options?.defaultPageSize || 20);

  const reset = () => {
    setPage(1);
    setPageSize(20);
  };

  return {
    page,
    pageSize,
    setPage,
    setPageSize,
    reset,
    pagination: {
      current: page,
      pageSize,
      showSizeChanger: true,
      showQuickJumper: true,
      showTotal: (total: number) => `共 ${total} 条`,
      onChange: (p: number, ps: number) => {
        setPage(p);
        setPageSize(ps);
      },
    },
  };
};
```

### 9.2 使用示例
```tsx
const { page, pageSize, setPage, pagination } = usePagination();

const { data, isLoading } = useQuery({
  queryKey: ['tenants', page, pageSize],
  queryFn: () => tenantApi.list({ page, page_size: pageSize }),
});

<Table
  dataSource={data?.items}
  loading={isLoading}
  pagination={pagination}
  rowKey="id"
/>
```

## 10. Git 提交规范

参考 Python 代码规范中的 Git 提交规范。

### 10.1 提交示例
```
feat(tenant): 添加租户列表页面

- 实现租户列表展示
- 添加分页功能
- 集成权限控制

Closes #123
```
