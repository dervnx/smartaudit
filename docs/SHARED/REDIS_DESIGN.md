# Redis 缓存与消息队列设计

## 1. Redis 使用场景

| 场景 | 用途 | 数据结构 | TTL |
|------|------|----------|-----|
| 会话存储 | 用户登录会话 | String (JWT) | 30 分钟 |
| Token 刷新 | Refresh Token | String | 7 天 |
| 缓存 | 查询缓存 | String/Hash | 5-30 分钟 |
| 队列 | 异步任务 | List | 永久 |
| 分布式锁 | 资源锁定 | String | 30 秒 |
| 限流 | API 限流 | String (计数器) | 1 分钟 |
| 信号量 | 并发控制 | String | 30 秒 |

## 2. 键命名规范

```
{prefix}:{tenant_id}:{resource}:{id}

示例：
saas:admin:session:token123        # SaaS 管理员会话
tenant:uuid123:user:session:token456 # 租户用户会话
tenant:uuid123:cache:project:list   # 租户项目列表缓存
tenant:uuid123:lock:audit:data456   # 审核数据锁
```

### 2.1 前缀定义

| 前缀 | 用途 |
|------|------|
| `saas:admin:*` | SaaS 管理员相关 |
| `tenant:{id}:*` | 租户相关 |
| `cache:*` | 通用缓存 |
| `queue:*` | 消息队列 |
| `lock:*` | 分布式锁 |
| `rate:*` | 限流器 |

## 3. 会话管理

### 3.1 用户会话

```python
# 会话数据结构
{
    "user_id": "uuid",
    "username": "admin",
    "tenant_id": "uuid",
    "roles": ["ADMIN"],
    "permissions": ["PROJECT_VIEW", "AUDIT_EXECUTE"],
    "login_at": "2024-01-20T10:00:00Z",
    "ip": "192.168.1.100",
    "device": "pc_web"
}
```

### 3.2 Token 结构

```
# Access Token
Key: tenant:{tenant_id}:token:access:{token_id}
TTL: 30 分钟

# Refresh Token
Key: tenant:{tenant_id}:token:refresh:{token_id}
TTL: 7 天

# Token ID 生成
token_id = uuid.uuid4().hex
```

### 3.3 会话操作

```python
class SessionService:
    async def create_session(self, user_id: str, data: dict) -> str:
        """创建会话"""
        token_id = uuid.uuid4().hex
        key = f"tenant:{data['tenant_id']}:token:access:{token_id}"

        await self.redis.setex(
            key,
            30 * 60,  # 30 分钟
            json.dumps(data)
        )

        return token_id

    async def get_session(self, tenant_id: str, token_id: str) -> dict:
        """获取会话"""
        key = f"tenant:{tenant_id}:token:access:{token_id}"
        data = await self.redis.get(key)

        if data:
            return json.loads(data)
        return None

    async def delete_session(self, tenant_id: str, token_id: str):
        """删除会话"""
        key = f"tenant:{tenant_id}:token:access:{token_id}"
        await self.redis.delete(key)

    async def refresh_session(self, tenant_id: str, refresh_token: str) -> str:
        """刷新会话"""
        # 验证 refresh token
        # 创建新的 access token
        pass
```

## 4. 缓存策略

### 4.1 缓存键设计

```python
# 项目列表缓存
cache:project:list:{tenant_id} -> JSON

# 项目详情缓存
cache:project:detail:{project_id} -> JSON

# 规则列表缓存
cache:rule:list:{tenant_id} -> JSON

# 用户权限缓存
cache:user:permissions:{user_id} -> JSON
```

### 4.2 缓存操作

```python
class CacheService:
    # 项目列表缓存 (5分钟)
    async def get_project_list(self, tenant_id: str) -> list:
        key = f"cache:project:list:{tenant_id}"
        data = await self.redis.get(key)

        if data:
            return json.loads(data)

        # 从数据库获取
        projects = await self.project_repo.list(tenant_id)

        # 写入缓存
        await self.redis.setex(key, 5 * 60, json.dumps(projects))

        return projects

    # 清除项目列表缓存
    async def invalidate_project_list(self, tenant_id: str):
        key = f"cache:project:list:{tenant_id}"
        await self.redis.delete(key)

    # 清除项目详情缓存
    async def invalidate_project(self, project_id: str):
        key = f"cache:project:detail:{project_id}"
        await self.redis.delete(key)
```

## 5. Celery 消息队列

### 5.1 队列配置

```python
# app/tasks/celery_app.py
from celery import Celery

def create_celery_app() -> Celery:
    app = Celery('smartaudit')

    # Broker 配置
    app.conf.broker_url = settings.REDIS_URL
    app.conf.result_backend = settings.REDIS_URL

    # 序列化配置
    app.conf.task_serializer = 'json'
    app.conf.result_serializer = 'json'
    app.conf.accept_content = ['json']

    # 时区配置
    app.conf.timezone = 'Asia/Shanghai'
    app.conf.enable_utc = True

    # 任务路由
    app.conf.task_routes = {
        'audit.*': {'queue': 'audit'},
        'backup.*': {'queue': 'backup'},
        'notify.*': {'queue': 'notify'},
        'cleanup.*': {'queue': 'cleanup'},
    }

    # 任务限流
    app.conf.task_annotations = {
        'audit.execute_rule': {'rate_limit': '100/m'},
    }

    # 任务重试
    app.conf.task_acks_late = True
    app.conf.task_reject_on_worker_lost = True

    return app

celery_app = create_celery_app()
```

### 5.2 队列类型

| 队列 | 优先级 | 说明 |
|------|--------|------|
| audit | 8 | 审核相关任务 |
| notify | 5 | 通知任务 |
| backup | 3 | 备份任务 |
| cleanup | 1 | 清理任务 |

### 5.3 任务定义

```python
# app/tasks/audit_tasks.py
from celery import shared_task

@shared_task(name='audit.execute_rule', queue='audit', bind=True)
def execute_audit_rule(self, data_id: str, rule_ids: list):
    """
    执行审核规则

    Args:
        data_id: 审核数据ID
        rule_ids: 规则ID列表
    """
    try:
        # 获取审核数据
        audit_data = await audit_data_repo.get(data_id)

        # 获取规则
        rules = await rule_repo.get_many(rule_ids)

        # 执行规则
        for rule in sorted(rules, key=lambda r: r.priority):
            result = rule_engine.execute(rule, audit_data.input_data)

            if not result.passed:
                audit_data.auto_result = 'reject'
                audit_data.rule_result = result.to_dict()
                break

        # 更新数据状态
        await audit_data_repo.update(audit_data)

        # 触发任务分配
        assign_audit_task.delay(data_id, audit_data.project_id)

    except Exception as e:
        raise self.retry(exc=e, countdown=60)


@shared_task(name='audit.assign_task', queue='audit', bind=True)
def assign_audit_task(self, data_id: str, project_id: str):
    """
    分配审核任务

    Args:
        data_id: 审核数据ID
        project_id: 项目ID
    """
    # 获取项目配置
    project = await project_repo.get(project_id)

    # 根据分配策略选择审核员
    if project.task_assign_type == 'round_robin':
        auditor_id = await select_auditor_round_robin(project.tenant_id)
    elif project.task_assign_type == 'load_balance':
        auditor_id = await select_auditor_load_balance(project.tenant_id)
    elif project.task_assign_type == 'efficiency':
        auditor_id = await select_auditor_by_efficiency(project.tenant_id)
    else:
        auditor_id = None

    # 创建任务
    task = Task(
        tenant_id=project.tenant_id,
        data_id=data_id,
        auditor_id=auditor_id,
        status='pending' if auditor_id else 'unassigned',
        assigned_at=datetime.utcnow() if auditor_id else None
    )
    await task_repo.save(task)

    # 通知审核员
    if auditor_id:
        notify_auditor.delay(auditor_id, task.id)


@shared_task(name='audit.notify_auditor', queue='notify')
def notify_auditor(auditor_id: str, task_id: str):
    """通知审核员有新任务"""
    auditor = await user_repo.get(auditor_id)

    # 发送通知
    await notification_service.send(
        user_id=auditor_id,
        title='新审核任务',
        content=f'您有新的审核任务，请及时处理',
        type='task_reminder'
    )
```

```python
# app/tasks/backup_tasks.py
@shared_task(name='backup.execute', queue='backup', bind=True)
def execute_backup(self, backup_record_id: str):
    """执行备份"""
    record = await backup_repo.get(backup_record_id)

    try:
        # 执行备份逻辑
        backup_service.execute_backup(record)

        record.status = 'success'
        record.completed_at = datetime.utcnow()

    except Exception as e:
        record.status = 'failed'
        record.error_msg = str(e)

    await backup_repo.save(record)


@shared_task(name='backup.cleanup', queue='cleanup')
def cleanup_old_backups(tenant_id: str, retention_days: int):
    """清理过期备份"""
    cutoff_date = datetime.utcnow() - timedelta(days=retention_days)

    # 获取过期备份
    old_records = await backup_repo.list_before(tenant_id, cutoff_date)

    for record in old_records:
        # 删除文件
        if record.file_path:
            await storage_service.delete(record.file_path)

        # 删除记录
        await backup_repo.delete(record.id)
```

## 6. 分布式锁

### 6.1 锁实现

```python
# app/utils/distributed_lock.py
import asyncio
from contextlib import asynccontextmanager

class DistributedLock:
    def __init__(self, redis: Redis, key: str, timeout: int = 30):
        self.redis = redis
        self.key = f"lock:{key}"
        self.timeout = timeout
        self.token = uuid.uuid4().hex

    async def acquire(self) -> bool:
        """获取锁"""
        result = await self.redis.set(
            self.key,
            self.token,
            nx=True,  # 仅当不存在时设置
            ex=self.timeout  # 过期时间
        )
        return result is not None

    async def release(self):
        """释放锁"""
        # 使用 Lua 脚本确保只删除自己的锁
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        await self.redis.eval(script, 1, self.key, self.token)

    async def extend(self, seconds: int):
        """延长锁时间"""
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("expire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        await self.redis.eval(script, 1, self.key, self.token, seconds)


@asynccontextmanager
async def distributed_lock(redis: Redis, key: str, timeout: int = 30):
    """分布式锁上下文管理器"""
    lock = DistributedLock(redis, key, timeout)

    if await lock.acquire():
        try:
            yield True
        finally:
            await lock.release()
    else:
        raise LockAcquisitionError(f"Failed to acquire lock: {key}")
```

### 6.2 使用场景

```python
# 审核数据处理锁
async with distributed_lock(redis, f"audit:data:{data_id}"):
    # 处理审核数据
    await process_audit_data(data_id)

# 任务分配锁
async with distributed_lock(redis, f"task:assign:{project_id}"):
    # 分配任务
    await assign_task(project_id)
```

## 7. 限流器

### 7.1 限流实现

```python
# app/utils/rate_limiter.py
class RateLimiter:
    def __init__(self, redis: Redis, key: str, limit: int, window: int):
        """
        限流器

        Args:
            redis: Redis 客户端
            key: 限流键
            limit: 限制次数
            window: 时间窗口(秒)
        """
        self.redis = redis
        self.key = f"rate:{key}"
        self.limit = limit
        self.window = window

    async def is_allowed(self) -> bool:
        """检查是否允许"""
        key = self.key

        # 使用 Lua 脚本保证原子性
        script = """
        local current = redis.call("incr", KEYS[1])
        if current == 1 then
            redis.call("expire", KEYS[1], ARGV[1])
        end
        if current > tonumber(ARGV[2]) then
            return 0
        else
            return 1
        end
        """

        result = await self.redis.eval(
            script, 1,
            key, self.window, self.limit
        )

        return result == 1

    async def get_remaining(self) -> int:
        """获取剩余次数"""
        current = await self.redis.get(self.key)
        if current is None:
            return self.limit
        return max(0, self.limit - int(current))
```

### 7.2 使用场景

```python
# API 限流中间件
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # 根据用户/IP限流
    identifier = request.headers.get("X-User-ID", request.client.host)

    limiter = RateLimiter(
        redis=redis,
        key=f"api:{identifier}",
        limit=60,  # 60次
        window=60  # 每分钟
    )

    if not await limiter.is_allowed():
        return JSONResponse(
            status_code=429,
            content={"error": "Too many requests"}
        )

    response = await call_next(request)
    return response
```

## 8. Celery Beat 定时任务

```python
# app/tasks/scheduler.py
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    # 每小时检查待分配任务
    'check-unassigned-tasks': {
        'task': 'audit.check_unassigned_tasks',
        'schedule': crontab(minute=0),
        'options': {'queue': 'audit'},
    },

    # 每天凌晨2点执行备份
    'daily-backup': {
        'task': 'backup.schedule_backup',
        'schedule': crontab(hour=2, minute=0),
        'options': {'queue': 'backup'},
    },

    # 每天凌晨3点清理过期备份
    'cleanup-old-backups': {
        'task': 'backup.cleanup_expired',
        'schedule': crontab(hour=3, minute=0),
        'options': {'queue': 'cleanup'},
    },

    # 每天凌晨4点清理过期缓存
    'cleanup-expired-cache': {
        'task': 'cache.cleanup_expired',
        'schedule': crontab(hour=4, minute=0),
        'options': {'queue': 'cleanup'},
    },

    # 每周一凌晨1点生成统计报告
    'weekly-statistics': {
        'task': 'statistics.generate_weekly_report',
        'schedule': crontab(hour=1, minute=0, day_of_week=1),
        'options': {'queue': 'notify'},
    },
}
```

## 9. Redis 配置

### 9.1 推荐配置

```redis
# redis.conf

# 内存配置
maxmemory 2gb
maxmemory-policy allkeys-lru

# 持久化配置
appendonly yes
appendfsync everysec

# 连接配置
timeout 300
tcp-keepalive 60

# 慢查询日志
slowlog-log-slower-than 10000
slowlog-max-len 128
```

### 9.2 集群配置 (生产环境)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Redis Sentinel 集群                         │
│                                                                  │
│    ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│    │ Master  │────►│ Slave 1 │────►│ Slave 2 │                   │
│    └────┬────┘     └─────────┘     └─────────┘                   │
│         │                                                    │
│    ┌────┴────┐                                                │
│    │ Sentinel│                                                │
│    │ Nodes  │ (3个Sentinel节点)                                │
│    └─────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```
