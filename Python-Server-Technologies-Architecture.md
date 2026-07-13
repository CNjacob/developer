# Python 服务端常用技术与架构

## 1. 服务端开发知识地图

Python 服务端开发不仅是写接口，还包括数据存储、缓存、消息队列、异步任务、安全、可观测性、部署和架构设计。

```text
Client
  -> CDN / Load Balancer / Nginx
  -> Python Web Service
  -> Service Layer
  -> Database / Cache / MQ / Object Storage / External APIs
  -> Observability / CI/CD / Runtime Platform
```

---

## 2. HTTP API 设计

REST 常见约定：

| 操作 | 方法 | 路径 |
| --- | --- | --- |
| 创建用户 | `POST` | `/users` |
| 查询列表 | `GET` | `/users` |
| 查询详情 | `GET` | `/users/{id}` |
| 全量更新 | `PUT` | `/users/{id}` |
| 局部更新 | `PATCH` | `/users/{id}` |
| 删除 | `DELETE` | `/users/{id}` |

响应示例：

```json
{
  "data": {
    "id": 1,
    "name": "Alice"
  },
  "request_id": "req_123"
}
```

错误示例：

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "user not found"
  },
  "request_id": "req_123"
}
```

接口设计要点：

- 使用明确的状态码。
- 分页、排序、过滤参数要统一。
- 对外接口保持向后兼容。
- 幂等接口需要幂等键或业务唯一约束。
- 每个请求生成 `request_id`，方便排查问题。

---

## 3. 数据库

### 关系型数据库

常用 PostgreSQL 和 MySQL。业务系统优先选择关系型数据库，因为它提供事务、约束、索引和 SQL 查询能力。

常见设计要点：

- 主键使用自增 ID、雪花 ID 或 UUID。
- 唯一约束保证业务唯一性。
- 外键在强一致场景有价值，但部分高并发系统会在应用层维护关系。
- 时间字段统一使用 UTC 存储。
- 金额使用整数分或 Decimal，不使用浮点数。

索引示例：

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

事务示例：

```python
def transfer(db, from_id: int, to_id: int, amount: int) -> None:
    with db.begin():
        debit(db, from_id, amount)
        credit(db, to_id, amount)
```

### NoSQL

| 类型 | 代表 | 适用场景 |
| --- | --- | --- |
| 文档数据库 | MongoDB | 灵活结构、文档型数据 |
| KV | Redis | 缓存、锁、计数、会话 |
| 搜索引擎 | Elasticsearch / OpenSearch | 全文检索、日志检索 |
| 列式数据库 | ClickHouse | 日志、指标、分析查询 |
| 图数据库 | Neo4j | 关系网络、路径查询 |

---

## 4. ORM 与 SQL

ORM 提升开发效率，但不能替代 SQL 能力。

常用 ORM：

| ORM | 特点 |
| --- | --- |
| SQLAlchemy | 灵活、强大、适合服务端项目 |
| Django ORM | 与 Django 深度集成 |
| Tortoise ORM | 异步 ORM，风格接近 Django |
| SQLModel | 基于 SQLAlchemy 和 Pydantic |

Repository 示例：

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.user import User

class UserRepository:
    def __init__(self, db: Session) -> None:
        self.db = db

    def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return self.db.scalar(stmt)
```

性能注意：

- 避免 N+1 查询。
- 列表接口要分页。
- 大批量写入使用 batch。
- 慢查询需要看执行计划。
- 事务范围不要过大。

---

## 5. 缓存

Redis 常用于缓存、分布式锁、限流、排行榜、会话和临时状态。

缓存模式：

| 模式 | 说明 |
| --- | --- |
| Cache Aside | 应用先查缓存，未命中查数据库再写缓存 |
| Read Through | 读缓存时由缓存层加载数据 |
| Write Through | 写入缓存时同步写后端存储 |
| Write Behind | 先写缓存，异步刷盘 |

Cache Aside 示例：

```python
import json

async def get_user(user_id: int, redis, repository):
    key = f"user:{user_id}"
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)

    user = await repository.get(user_id)
    if user:
        await redis.set(key, json.dumps(user), ex=300)
    return user
```

缓存问题：

| 问题 | 解决思路 |
| --- | --- |
| 缓存穿透 | 缓存空值、布隆过滤器、参数校验 |
| 缓存击穿 | 互斥锁、singleflight、热点不过期 |
| 缓存雪崩 | 随机过期、预热、限流、降级 |
| 数据不一致 | 删除缓存、延迟双删、事件驱动更新 |

---

## 6. 消息队列

消息队列用于削峰填谷、异步解耦、事件驱动和重试补偿。

| MQ | 特点 |
| --- | --- |
| RabbitMQ | 路由能力强，传统消息队列 |
| Kafka | 高吞吐、日志流、事件流 |
| Redis Stream | 轻量，适合中小规模队列 |
| AWS SQS / Pub/Sub | 云托管，运维成本低 |

典型事件：

```json
{
  "event_id": "evt_001",
  "event_type": "order.created",
  "occurred_at": "2026-01-01T00:00:00Z",
  "payload": {
    "order_id": 1001,
    "user_id": 20
  }
}
```

消费者要点：

- 消息处理尽量幂等。
- 失败要重试，超过次数进死信队列。
- 消费位点或 ack 要在业务成功后提交。
- 事件 schema 要版本化。
- 不要把 MQ 当数据库使用。

---

## 7. 异步任务

Python 常用 Celery、Dramatiq、RQ、Huey。

Celery 示例：

```python
from celery import Celery

celery_app = Celery(
    "worker",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)

@celery_app.task(bind=True, max_retries=3)
def send_email(self, user_id: int) -> None:
    try:
        print(f"send email to {user_id}")
    except Exception as exc:
        raise self.retry(exc=exc, countdown=10) from exc
```

适合异步任务的场景：

- 发送邮件、短信、推送。
- 生成报表、导出文件。
- 图片、视频处理。
- 调用第三方接口并重试。
- 定时任务和补偿任务。

---

## 8. 安全

常见安全能力：

| 能力 | 技术 |
| --- | --- |
| 身份认证 | Session、JWT、OAuth2、OIDC |
| 权限控制 | RBAC、ABAC、ACL |
| 传输安全 | HTTPS、TLS |
| 密码存储 | bcrypt、argon2 |
| 输入校验 | Pydantic、表单校验、白名单 |
| 防攻击 | CSRF、XSS、SQL 注入防护、限流 |
| 密钥管理 | 环境变量、Vault、云 Secret Manager |

JWT 适合无状态 API，但吊销、续期和权限变更要额外设计。后台管理系统使用服务端 Session 往往更简单可靠。

密码哈希示例：

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(password: str, hashed: str) -> bool:
    return pwd_context.verify(password, hashed)
```

---

## 9. 限流、熔断、重试

限流算法：

| 算法 | 特点 |
| --- | --- |
| 固定窗口 | 简单，边界可能突刺 |
| 滑动窗口 | 更平滑 |
| 漏桶 | 匀速处理 |
| 令牌桶 | 允许短暂突发 |

重试要点：

- 只对可重试错误重试。
- 使用指数退避和随机抖动。
- 写操作要有幂等保护。
- 设置总超时，避免请求堆积。

示例：

```python
import random
import time

def retry(fn, attempts: int = 3):
    for index in range(attempts):
        try:
            return fn()
        except TimeoutError:
            if index == attempts - 1:
                raise
            delay = 0.2 * (2 ** index) + random.random() * 0.1
            time.sleep(delay)
```

---

## 10. 可观测性

可观测性包括日志、指标和链路追踪。

| 类型 | 说明 | 工具 |
| --- | --- | --- |
| Logs | 离散事件记录 | ELK、Loki、Cloud Logging |
| Metrics | 数值指标 | Prometheus、Grafana |
| Traces | 请求链路 | OpenTelemetry、Jaeger、Tempo |

关键指标：

- QPS / RPS。
- P50/P95/P99 延迟。
- 错误率。
- CPU、内存、磁盘、网络。
- 数据库连接池使用率。
- 队列堆积长度。

FastAPI 指标中间件可记录请求耗时：

```python
import logging
import time

from fastapi import Request

logger = logging.getLogger(__name__)

async def access_log_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = (time.perf_counter() - start) * 1000
    logger.info(
        "request finished",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status_code": response.status_code,
            "duration_ms": round(duration_ms, 2),
        },
    )
    return response
```

---

## 11. 部署与运行

常见部署方式：

| 方式 | 说明 |
| --- | --- |
| VM + systemd | 简单直接，适合小规模 |
| Docker Compose | 单机多服务 |
| Kubernetes | 大规模容器编排 |
| Serverless | 事件驱动，低运维 |
| PaaS | Heroku、Render、Fly.io 等 |

典型生产链路：

```text
Internet
  -> CDN / WAF
  -> Load Balancer
  -> Nginx / Ingress
  -> Gunicorn/Uvicorn Workers
  -> Python App
  -> PostgreSQL / Redis / Kafka
```

Gunicorn worker 粗略估算：

```text
同步 WSGI：workers ≈ CPU 核数 * 2 + 1
异步 ASGI：workers 通常按 CPU 核数、内存和 I/O 压测结果调整
```

不要只依赖公式，最终以压测、延迟、错误率和资源使用率为准。

---

## 12. 常见架构风格

### 单体架构

```text
Web App
  -> Service
  -> Database
```

优点：

- 开发和部署简单。
- 本地调试容易。
- 事务边界清晰。

缺点：

- 代码和团队规模变大后边界容易混乱。
- 单个服务发布影响面较大。

适合早期业务和中小系统。单体不是落后，混乱的单体才是问题。

### 分层架构

```text
API Layer
  -> Application Service
  -> Domain Logic
  -> Repository
  -> Infrastructure
```

优点是职责清晰，适合大多数服务端项目。

### 微服务架构

```text
User Service
Order Service
Payment Service
Inventory Service
Notification Service
```

优点：

- 独立部署。
- 团队自治。
- 按业务域扩展。

成本：

- 分布式事务。
- 服务发现和治理。
- 链路追踪和监控复杂。
- 接口兼容、消息一致性、故障隔离成本更高。

只有当组织和业务复杂度需要时，微服务才值得。

### 事件驱动架构

```text
Order Created
  -> Payment Handler
  -> Inventory Handler
  -> Notification Handler
  -> Analytics Handler
```

适合解耦、异步处理和多下游订阅。关键是事件模型、幂等、顺序性和补偿机制。

---

## 13. 领域驱动设计在 Python 中的简化落地

典型目录：

```text
app/
  domain/
    order.py
    events.py
  application/
    order_service.py
  infrastructure/
    repositories/
      order_repository.py
  api/
    routes/
      orders.py
```

领域对象示例：

```python
from dataclasses import dataclass, field
from enum import StrEnum

class OrderStatus(StrEnum):
    CREATED = "created"
    PAID = "paid"
    CANCELLED = "cancelled"

@dataclass
class Order:
    id: int
    user_id: int
    total_amount: int
    status: OrderStatus = OrderStatus.CREATED
    events: list[dict] = field(default_factory=list)

    def pay(self) -> None:
        if self.status != OrderStatus.CREATED:
            raise ValueError("only created order can be paid")
        self.status = OrderStatus.PAID
        self.events.append({"type": "order.paid", "order_id": self.id})
```

应用服务负责用例编排：

```python
class PayOrderService:
    def __init__(self, order_repo, event_bus) -> None:
        self.order_repo = order_repo
        self.event_bus = event_bus

    def execute(self, order_id: int) -> None:
        order = self.order_repo.get(order_id)
        if order is None:
            raise ValueError("order not found")

        order.pay()
        self.order_repo.save(order)

        for event in order.events:
            self.event_bus.publish(event)
```

---

## 14. 常用第三方库

| 类型 | 常用库 |
| --- | --- |
| Web | FastAPI、Django、Flask、Starlette |
| ASGI Server | Uvicorn、Hypercorn、Daphne |
| WSGI Server | Gunicorn、uWSGI、Waitress |
| 数据校验 | Pydantic、Marshmallow |
| ORM | SQLAlchemy、Django ORM、Tortoise ORM |
| 迁移 | Alembic、Django migrations |
| HTTP 客户端 | httpx、requests、aiohttp |
| Redis | redis-py、aioredis 相关能力已合入 redis-py |
| MQ / 任务 | Celery、Dramatiq、RQ、Kombu |
| 测试 | pytest、unittest、factory_boy、pytest-cov |
| 代码质量 | ruff、mypy、pyright、black、isort |
| 配置 | pydantic-settings、python-dotenv |
| 日志 | structlog、loguru、标准库 logging |
| 监控 | prometheus-client、opentelemetry |
| 安全 | passlib、python-jose、authlib |

---

## 15. 典型项目演进路线

```text
阶段 1：单体 API
  FastAPI/Django + PostgreSQL + Redis

阶段 2：工程化
  分层目录 + 测试 + CI + Docker + 迁移 + 日志

阶段 3：异步化
  Celery/RQ + MQ + 定时任务 + 幂等设计

阶段 4：可观测性
  结构化日志 + 指标 + 链路追踪 + 告警

阶段 5：服务拆分
  按业务域拆服务 + API 网关 + 服务治理 + 事件驱动
```

建议先把单体边界设计清楚，再考虑拆微服务。拆分不是架构能力的起点，而是复杂度增长后的治理手段。

---

## 16. 一个中型 Python 后端参考架构

```text
Frontend / Mobile
  -> API Gateway / Nginx
  -> FastAPI BFF
      -> Auth Service
      -> Order Service
      -> Payment Service
      -> Notification Service
  -> PostgreSQL
  -> Redis
  -> Kafka / RabbitMQ
  -> Celery Workers
  -> Object Storage
  -> Prometheus + Grafana + Loki + OpenTelemetry
```

关键设计：

- BFF 聚合前端接口，避免前端直接编排多个内部服务。
- 核心交易数据进入 PostgreSQL。
- 高频读数据使用 Redis 缓存。
- 耗时任务进入 Celery Worker。
- 跨服务事件通过 MQ 发布。
- 所有服务统一日志字段、trace id 和错误码。
- 数据库迁移和应用发布要纳入 CI/CD。

