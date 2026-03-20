# 测试规范

## 1. 测试策略

### 1.1 测试金字塔

```
                    ┌───────────┐
                    │    E2E    │  少量端到端测试
                    │   Tests   │  验证关键业务流程
                   ─┴───────────┴─
                  ┌───────────────┐
                  │  Integration  │  API 和组件集成测试
                  │    Tests      │  验证模块间协作
                 ─┴───────────────┴─
                ┌─────────────────────┐
                │       Unit          │  大量单元测试
                │      Tests          │  验证函数/方法逻辑
               ─┴─────────────────────┴─
```

### 1.2 测试覆盖率要求

| 测试类型 | 覆盖率要求 | 说明 |
|----------|------------|------|
| 单元测试 | 80%+ | 核心业务逻辑必须覆盖 |
| 集成测试 | 60%+ | API 端点必须覆盖 |
| E2E 测试 | 关键流程覆盖 | 登录、创建项目、审核等 |

## 2. 单元测试

### 2.1 Python 单元测试

#### 目录结构
```
tests/
├── conftest.py              # pytest 全局配置
├── unit/
│   ├── __init__.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── test_tenant_service.py
│   │   ├── test_project_service.py
│   │   └── test_rule_service.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── test_tenant_repo.py
│   └── utils/
│       ├── __init__.py
│       └── test_encrypt.py
└── fixtures/
    ├── __init__.py
    ├── tenant_fixtures.py
    └── project_fixtures.py
```

#### conftest.py 配置
```python
# tests/conftest.py
import pytest
import asyncio
from typing import Generator, AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from httpx import AsyncClient, ASGITransport

from app.main import app
from app.db.database import get_db
from app.models.base import Base

# 测试数据库 URL（SQLite 内存数据库）
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest.fixture(scope="session")
def event_loop():
    """创建事件循环"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="function")
async def db_engine():
    """创建测试数据库引擎"""
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

@pytest.fixture(scope="function")
async def db_session(db_engine) -> AsyncGenerator[AsyncSession, None]:
    """创建测试数据库会话"""
    async_session = async_sessionmaker(
        db_engine, class_=AsyncSession, expire_on_commit=False
    )

    async with async_session() as session:
        yield session

@pytest.fixture(scope="function")
async def client(db_session) -> AsyncGenerator[AsyncClient, None]:
    """创建测试客户端"""

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

    app.dependency_overrides.clear()

@pytest.fixture
def sample_tenant_data():
    """样例租户数据"""
    return {
        "name": "测试租户",
        "code": "test_tenant",
        "contact": "张三",
        "phone": "13800138000",
        "email": "test@example.com",
    }
```

#### 服务层测试示例
```python
# tests/unit/services/test_tenant_service.py
import pytest
from app.services.tenant_service import TenantService
from app.core.exceptions import TenantNotFoundError, TenantAlreadyExistsError

class TestTenantService:
    """租户服务单元测试"""

    @pytest.fixture
    def tenant_service(self, db_session):
        return TenantService(db_session)

    @pytest.fixture
    async def created_tenant(self, tenant_service, sample_tenant_data):
        """创建测试租户"""
        return await tenant_service.create(sample_tenant_data)

    class TestCreateTenant:
        """创建租户测试"""

        def test_create_tenant_success(self, tenant_service, sample_tenant_data):
            """成功创建租户"""
            tenant = tenant_service.create(sample_tenant_data)

            assert tenant.id is not None
            assert tenant.name == sample_tenant_data["name"]
            assert tenant.code == sample_tenant_data["code"]
            assert tenant.status == "active"
            assert tenant.is_deleted == 0

        def test_create_tenant_with_duplicate_code(
            self, tenant_service, sample_tenant_data
        ):
            """创建重复代码租户应抛出异常"""
            tenant_service.create(sample_tenant_data)

            with pytest.raises(TenantAlreadyExistsError) as exc_info:
                tenant_service.create(sample_tenant_data)

            assert "已存在" in str(exc_info.value.message)

        def test_create_tenant_with_invalid_data(self, tenant_service):
            """创建租户数据验证失败"""
            invalid_data = {"name": "", "code": ""}

            with pytest.raises(ValidationError):
                tenant_service.create(invalid_data)

    class TestGetTenant:
        """获取租户测试"""

        def test_get_tenant_by_id_success(self, tenant_service, created_tenant):
            """成功获取租户"""
            tenant = tenant_service.get_by_id(created_tenant.id)

            assert tenant.id == created_tenant.id
            assert tenant.name == created_tenant.name

        def test_get_tenant_by_id_not_found(self, tenant_service):
            """获取不存在的租户"""
            with pytest.raises(TenantNotFoundError):
                tenant_service.get_by_id(99999)

        def test_get_tenant_by_code_success(self, tenant_service, created_tenant):
            """通过代码获取租户"""
            tenant = tenant_service.get_by_code(created_tenant.code)

            assert tenant.code == created_tenant.code

    class TestUpdateTenant:
        """更新租户测试"""

        def test_update_tenant_success(self, tenant_service, created_tenant):
            """成功更新租户"""
            update_data = {"name": "新名称", "status": "disabled"}
            updated = tenant_service.update(created_tenant.id, update_data)

            assert updated.name == "新名称"
            assert updated.status == "disabled"
            assert updated.updated_at > created_tenant.created_at

    class TestDeleteTenant:
        """删除租户测试（软删除）"""

        def test_soft_delete_tenant_success(self, tenant_service, created_tenant):
            """成功软删除租户"""
            tenant_service.soft_delete(created_tenant.id)

            # 验证 is_deleted 标志
            tenant = tenant_service.get_by_id(created_tenant.id)
            assert tenant.is_deleted == 1
            assert tenant.deleted_at is not None

        def test_soft_deleted_tenant_not_in_list(self, tenant_service, created_tenant):
            """软删除的租户不在列表中"""
            tenant_service.soft_delete(created_tenant.id)

            tenants = tenant_service.list()
            assert created_tenant.id not in [t.id for t in tenants]
```

#### 工具函数测试示例
```python
# tests/unit/utils/test_encrypt.py
import pytest
from app.utils.encrypt import hash_password, verify_password, generate_random_string

class TestEncrypt:
    """加密工具测试"""

    def test_hash_password(self):
        """密码哈希"""
        password = "test_password123"
        hashed = hash_password(password)

        assert hashed != password
        assert len(hashed) == 64  # SHA256 哈希长度

    def test_verify_password_success(self):
        """验证密码成功"""
        password = "test_password123"
        hashed = hash_password(password)

        assert verify_password(password, hashed) is True

    def test_verify_password_fail(self):
        """验证密码失败"""
        password = "test_password123"
        wrong_password = "wrong_password"
        hashed = hash_password(password)

        assert verify_password(wrong_password, hashed) is False

    def test_generate_random_string(self):
        """生成随机字符串"""
        s1 = generate_random_string(16)
        s2 = generate_random_string(16)

        assert len(s1) == 16
        assert len(s2) == 16
        assert s1 != s2  # 随机性
```

### 2.2 React 单元测试

#### 目录结构
```
tests/
├── unit/
│   ├── components/
│   │   ├── Button.test.tsx
│   │   └── TenantCard.test.tsx
│   ├── hooks/
│   │   ├── useTenant.test.ts
│   │   └── usePagination.test.ts
│   └── utils/
│       └── validation.test.ts
```

#### 组件测试示例
```typescript
// tests/unit/components/TenantCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { TenantCard } from '@/components/business/TenantCard';
import type { Tenant } from '@/types/tenant';

describe('TenantCard', () => {
  const mockTenant: Tenant = {
    id: 1,
    name: '测试租户',
    code: 'test_tenant',
    status: 'active',
    created_at: '2024-01-01T00:00:00Z',
    updated_at: '2024-01-01T00:00:00Z',
  };

  const mockOnEdit = vi.fn();
  const mockOnDelete = vi.fn();

  it('should render tenant information correctly', () => {
    render(
      <TenantCard
        tenant={mockTenant}
        onEdit={mockOnEdit}
        onDelete={mockOnDelete}
      />
    );

    expect(screen.getByText('测试租户')).toBeInTheDocument();
    expect(screen.getByText('test_tenant')).toBeInTheDocument();
    expect(screen.getByText('启用')).toBeInTheDocument();
  });

  it('should call onEdit when edit button is clicked', () => {
    render(
      <TenantCard
        tenant={mockTenant}
        onEdit={mockOnEdit}
        onDelete={mockOnDelete}
      />
    );

    fireEvent.click(screen.getByText('编辑'));
    expect(mockOnEdit).toHaveBeenCalledWith(1);
  });

  it('should call onDelete when delete button is clicked', () => {
    render(
      <TenantCard
        tenant={mockTenant}
        onEdit={mockOnEdit}
        onDelete={mockOnDelete}
      />
    );

    fireEvent.click(screen.getByText('删除'));
    expect(mockOnDelete).toHaveBeenCalledWith(1);
  });

  it('should show loading state when isLoading is true', () => {
    render(
      <TenantCard
        tenant={mockTenant}
        onEdit={mockOnEdit}
        onDelete={mockOnDelete}
        isLoading={true}
      />
    );

    expect(screen.getByRole('button', { name: /删除/i })).toBeDisabled();
  });
});
```

#### Hook 测试示例
```typescript
// tests/unit/hooks/usePagination.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { usePagination } from '@/hooks/usePagination';

describe('usePagination', () => {
  it('should initialize with default values', () => {
    const { result } = renderHook(() => usePagination());

    expect(result.current.page).toBe(1);
    expect(result.current.pageSize).toBe(20);
  });

  it('should initialize with custom values', () => {
    const { result } = renderHook(() =>
      usePagination({ defaultPage: 3, defaultPageSize: 50 })
    );

    expect(result.current.page).toBe(3);
    expect(result.current.pageSize).toBe(50);
  });

  it('should update page when setPage is called', () => {
    const { result } = renderHook(() => usePagination());

    act(() => {
      result.current.setPage(5);
    });

    expect(result.current.page).toBe(5);
  });

  it('should update pageSize when setPageSize is called', () => {
    const { result } = renderHook(() => usePagination());

    act(() => {
      result.current.setPageSize(100);
    });

    expect(result.current.pageSize).toBe(100);
  });

  it('should reset pagination', () => {
    const { result } = renderHook(() => usePagination());

    act(() => {
      result.current.setPage(10);
      result.current.setPageSize(50);
      result.current.reset();
    });

    expect(result.current.page).toBe(1);
    expect(result.current.pageSize).toBe(20);
  });
});
```

## 3. 集成测试

### 3.1 API 集成测试

```python
# tests/integration/test_tenant_api.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
class TestTenantAPI:
    """租户 API 集成测试"""

    async def test_create_tenant(self, client: AsyncClient, sample_tenant_data):
        """创建租户 API"""
        response = await client.post("/api/v1/tenants", json=sample_tenant_data)

        assert response.status_code == 201
        data = response.json()
        assert data["code"] == 0
        assert data["data"]["name"] == sample_tenant_data["name"]

    async def test_get_tenant_list(self, client: AsyncClient, sample_tenant_data):
        """获取租户列表 API"""
        # 先创建
        await client.post("/api/v1/tenants", json=sample_tenant_data)

        # 获取列表
        response = await client.get("/api/v1/tenants")

        assert response.status_code == 200
        data = response.json()
        assert data["code"] == 0
        assert "items" in data["data"]
        assert len(data["data"]["items"]) > 0

    async def test_get_tenant_detail(self, client: AsyncClient, sample_tenant_data):
        """获取租户详情 API"""
        # 先创建
        create_response = await client.post("/api/v1/tenants", json=sample_tenant_data)
        tenant_id = create_response.json()["data"]["id"]

        # 获取详情
        response = await client.get(f"/api/v1/tenants/{tenant_id}")

        assert response.status_code == 200
        data = response.json()
        assert data["data"]["id"] == tenant_id

    async def test_update_tenant(self, client: AsyncClient, sample_tenant_data):
        """更新租户 API"""
        # 先创建
        create_response = await client.post("/api/v1/tenants", json=sample_tenant_data)
        tenant_id = create_response.json()["data"]["id"]

        # 更新
        update_data = {"name": "新名称"}
        response = await client.put(f"/api/v1/tenants/{tenant_id}", json=update_data)

        assert response.status_code == 200
        assert response.json()["data"]["name"] == "新名称"

    async def test_delete_tenant(self, client: AsyncClient, sample_tenant_data):
        """删除租户 API（软删除）"""
        # 先创建
        create_response = await client.post("/api/v1/tenants", json=sample_tenant_data)
        tenant_id = create_response.json()["data"]["id"]

        # 删除
        response = await client.delete(f"/api/v1/tenants/{tenant_id}")
        assert response.status_code == 204

        # 验证已删除（不在列表中）
        list_response = await client.get("/api/v1/tenants")
        items = list_response.json()["data"]["items"]
        assert tenant_id not in [item["id"] for item in items]

    async def test_pagination(self, client: AsyncClient):
        """分页 API"""
        # 创建多个租户
        for i in range(25):
            data = {
                "name": f"租户{i}",
                "code": f"tenant_{i}",
                "contact": "测试",
                "phone": "13800000000",
                "email": f"test{i}@example.com",
            }
            await client.post("/api/v1/tenants", json=data)

        # 测试分页
        response = await client.get("/api/v1/tenants?page=1&page_size=10")

        assert response.status_code == 200
        data = response.json()["data"]
        assert len(data["items"]) == 10
        assert data["total"] == 25
        assert data["pages"] == 3
        assert data["has_next"] is True
```

### 3.2 项目测试用例模板

```python
# tests/integration/test_project_api.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
class TestProjectAPI:
    """项目 API 集成测试"""

    async def test_create_project(self, client: AsyncClient, auth_headers):
        """创建项目"""
        project_data = {
            "name": "测试项目",
            "code": "test_project",
            "description": "项目描述",
            "field_config": {
                "fields": [
                    {"name": "name", "label": "姓名", "type": "string", "required": True},
                    {"name": "id_card", "label": "身份证", "type": "string", "required": True},
                ]
            },
            "need_manual_verify": True,
            "task_assign_strategy": "round_robin",
        }

        response = await client.post(
            "/api/v1/projects",
            json=project_data,
            headers=auth_headers
        )

        assert response.status_code == 201
        assert response.json()["data"]["name"] == "测试项目"

    async def test_project_field_validation(self, client: AsyncClient, auth_headers):
        """项目字段验证"""
        invalid_data = {
            "name": "",  # 名称不能为空
            "code": "test",  # 缺少 required 字段
        }

        response = await client.post(
            "/api/v1/projects",
            json=invalid_data,
            headers=auth_headers
        )

        assert response.status_code == 422
```

## 4. E2E 测试

### 4.1 Playwright 配置

```typescript
// tests/e2e/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
});
```

### 4.2 E2E 测试示例

```typescript
// tests/e2e/tenant.spec.ts
import { test, expect } from '@playwright/test';

test.describe('租户管理 E2E 测试', () => {
  test.beforeEach(async ({ page }) => {
    // 登录
    await page.goto('/login');
    await page.fill('[name="username"]', 'admin');
    await page.fill('[name="password"]', 'password');
    await page.click('[type="submit"]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('创建租户流程', async ({ page }) => {
    // 导航到租户管理
    await page.click('text=租户管理');
    await page.click('text=创建租户');

    // 填写租户信息
    await page.fill('[name="name"]', 'E2E 测试租户');
    await page.fill('[name="code"]', 'e2e_tenant');
    await page.fill('[name="contact"]', '测试人员');
    await page.fill('[name="phone"]', '13800138000');
    await page.fill('[name="email"]', 'e2e@test.com');

    // 提交
    await page.click('text=提交');

    // 验证成功提示
    await expect(page.locator('.ant-message')).toContainText('创建成功');

    // 验证租户在列表中
    await expect(page.locator('text=E2E 测试租户')).toBeVisible();
  });

  test('租户列表分页', async ({ page }) => {
    await page.goto('/tenant/list');

    // 验证分页控件
    await expect(page.locator('.ant-pagination')).toBeVisible();

    // 切换每页数量
    await page.selectOption('.ant-select-selector', '50');
    await expect(page.locator('.ant-table-tbody tr')).toHaveCount(50);
  });
});

test.describe('项目审核 E2E 测试', () => {
  test('完整审核流程', async ({ page }) => {
    // 1. 创建项目
    await page.goto('/project/create');
    await page.fill('[name="name"]', '审核测试项目');
    await page.fill('[name="code"]', 'audit_test');
    await page.click('text=提交');

    // 2. 配置规则
    await page.goto('/rule/list');
    await page.click('text=创建规则');
    await page.fill('[name="name"]', '测试规则');
    // 配置条件...

    // 3. 提交审核数据
    await page.goto('/project/audit');
    await page.click('text=提交数据');
    await page.fill('[name="name"]', '张三');
    await page.fill('[name="id_card"]', '110101199001011234');
    await page.click('text=提交');

    // 4. 执行审核
    await page.goto('/task/list');
    await page.click('text=去审核');
    await page.click('text=通过');

    // 5. 验证审核结果
    await expect(page.locator('text=审核通过')).toBeVisible();
  });
});
```

## 5. 测试执行

### 5.1 pytest 配置

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
asyncio_mode = auto
addopts =
    -v
    --tb=short
    --strict-markers
    --disable-warnings
markers =
    unit: 单元测试
    integration: 集成测试
    e2e: 端到端测试
    slow: 慢速测试
```

### 5.2 package.json scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:unit": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration",
    "test:e2e": "playwright test",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

### 5.3 执行命令

```bash
# Python 测试
# 运行所有测试
pytest

# 运行单元测试
pytest tests/unit

# 运行集成测试
pytest tests/integration -v

# 运行特定测试文件
pytest tests/unit/services/test_tenant_service.py

# 运行特定测试类
pytest tests/unit/services/test_tenant_service.py::TestTenantService::TestCreateTenant

# 生成覆盖率报告
pytest --cov=app --cov-report=html

# React 测试
# 运行所有测试
npm test

# 运行单元测试
npm run test:unit

# 运行覆盖率
npm run test:coverage

# E2E 测试
npm run test:e2e
```

## 6. 性能测试

### 6.1 Locust 性能测试

```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between, events
import random

class TenantUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        """登录获取 token"""
        response = self.client.post("/api/v1/auth/login", json={
            "username": "test_user",
            "password": "test_password"
        })
        if response.status_code == 200:
            self.token = response.json()["data"]["access_token"]
        else:
            self.token = None

    @task(3)
    def list_tenants(self):
        """获取租户列表"""
        if self.token:
            self.client.get(
                "/api/v1/tenants",
                headers={"Authorization": f"Bearer {self.token}"}
            )

    @task(2)
    def get_tenant_detail(self):
        """获取租户详情"""
        if self.token:
            tenant_id = random.randint(1, 100)
            self.client.get(
                f"/api/v1/tenants/{tenant_id}",
                headers={"Authorization": f"Bearer {self.token}"}
            )

    @task(1)
    def create_task(self):
        """创建审核任务"""
        if self.token:
            self.client.post(
                "/api/v1/tasks",
                headers={"Authorization": f"Bearer {self.token}"},
                json={
                    "project_id": 1,
                    "data": {"name": "测试数据"}
                }
            )

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    print("性能测试开始")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    print("性能测试结束")
```

### 6.2 执行性能测试

```bash
# 运行 Locust
locust -f tests/performance/locustfile.py --host=http://localhost:8000

# Headless 模式
locust -f tests/performance/locustfile.py \
  --host=http://localhost:8000 \
  --users=100 \
  --spawn-rate=10 \
  --run-time=60s \
  --headless \
  --csv=results
```

## 7. 测试用例管理

### 7.1 测试用例模板

```markdown
# 测试用例：TC-XXX

## 基本信息
- **用例编号**: TC-001
- **用例名称**: 租户创建功能测试
- **测试类型**: 功能测试
- **优先级**: P0
- **所属模块**: 租户管理

## 前置条件
1. 用户已登录系统
2. 用户有租户创建权限

## 测试步骤
1. 点击"创建租户"按钮
2. 填写租户信息（名称、代码、联系人等）
3. 点击"提交"按钮
4. 验证成功提示

## 测试数据
- 租户名称：测试租户-{timestamp}
- 租户代码：test_{timestamp}
- 联系人：张三
- 手机号：13800138000

## 预期结果
1. 租户创建成功
2. 显示"创建成功"提示
3. 租户出现在列表中

## 实际结果
-

## 测试结论
- [ ] 通过
- [ ] 失败

## 缺陷编号
-

## 测试人员
-
## 测试时间
-
```

### 7.2 测试用例清单（按模块）

| 模块 | 用例数 | P0 用例 | P1 用例 |
|------|--------|---------|---------|
| 租户管理 | 15 | 5 | 10 |
| 项目管理 | 20 | 8 | 12 |
| 规则引擎 | 18 | 6 | 12 |
| 任务审核 | 25 | 10 | 15 |
| 账户权限 | 12 | 4 | 8 |
| 统计报表 | 10 | 3 | 7 |
| 三方接口 | 8 | 3 | 5 |
| 版本管理 | 6 | 2 | 4 |
| 备份管理 | 7 | 3 | 4 |
| **总计** | **121** | **44** | **77** |
