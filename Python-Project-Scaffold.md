# Python 项目基础框架搭建

## 1. 推荐项目结构

服务端项目可以从以下结构开始：

```text
my-python-service/
  pyproject.toml
  README.md
  .env.example
  .gitignore
  Dockerfile
  docker-compose.yml
  app/
    __init__.py
    main.py
    core/
      __init__.py
      config.py
      logging.py
      errors.py
    api/
      __init__.py
      routes/
        __init__.py
        health.py
        users.py
    schemas/
      __init__.py
      user.py
    models/
      __init__.py
      user.py
    repositories/
      __init__.py
      user_repository.py
    services/
      __init__.py
      user_service.py
    db/
      __init__.py
      session.py
  tests/
    __init__.py
    test_health.py
    test_user_service.py
```

分层含义：

| 目录 | 职责 |
| --- | --- |
| `api` | HTTP 路由、请求解析、响应封装 |
| `schemas` | 请求/响应 DTO，数据校验 |
| `models` | ORM 实体或领域模型 |
| `repositories` | 数据访问 |
| `services` | 业务逻辑与事务编排 |
| `core` | 配置、日志、异常、通用基础设施 |
| `db` | 数据库连接、会话、迁移相关 |
| `tests` | 单元测试、集成测试 |

核心原则：路由层保持薄，业务逻辑放在 service，数据库细节放在 repository。

---

## 2. 虚拟环境

Python 项目应使用虚拟环境隔离依赖。

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

常见依赖管理工具：

| 工具 | 特点 |
| --- | --- |
| `pip + venv` | 标准、简单 |
| `pip-tools` | 通过 `requirements.in` 生成锁定版本 |
| `Poetry` | 依赖、打包、虚拟环境一体化 |
| `PDM` | PEP 582/现代 Python 项目管理 |
| `uv` | 高性能依赖安装、虚拟环境和锁文件管理 |

---

## 3. `pyproject.toml`

现代 Python 项目推荐把构建、格式化、检查和测试配置集中在 `pyproject.toml`。

```toml
[project]
name = "my-python-service"
version = "0.1.0"
description = "A Python backend service"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "pydantic-settings>=2.0",
    "sqlalchemy>=2.0",
    "alembic>=1.13",
    "psycopg[binary]>=3.1",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",
    "ruff>=0.5",
    "mypy>=1.8",
]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]

[tool.mypy]
python_version = "3.11"
strict = true
plugins = []
```

---

## 4. 配置管理

配置应该来自环境变量，不应把密钥写死在代码里。

`.env.example`：

```env
APP_NAME=my-python-service
APP_ENV=local
DEBUG=true
DATABASE_URL=postgresql+psycopg://user:password@localhost:5432/app
REDIS_URL=redis://localhost:6379/0
JWT_SECRET=change-me
```

`app/core/config.py`：

```python
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "my-python-service"
    app_env: str = "local"
    debug: bool = False
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

使用 `lru_cache` 可以避免配置对象重复创建。

---

## 5. 日志

服务端日志要便于检索、聚合和告警。生产环境推荐结构化 JSON 日志。

`app/core/logging.py`：

```python
import logging
import sys

def setup_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        level=level,
        format="%(asctime)s %(levelname)s [%(name)s] %(message)s",
        handlers=[logging.StreamHandler(sys.stdout)],
    )
```

业务代码：

```python
import logging

logger = logging.getLogger(__name__)

def create_order(order_id: int) -> None:
    logger.info("creating order", extra={"order_id": order_id})
```

---

## 6. FastAPI 示例入口

`app/main.py`：

```python
from fastapi import FastAPI

from app.api.routes import health, users
from app.core.config import get_settings
from app.core.logging import setup_logging

def create_app() -> FastAPI:
    settings = get_settings()
    setup_logging("DEBUG" if settings.debug else "INFO")

    app = FastAPI(title=settings.app_name)
    app.include_router(health.router, prefix="/health", tags=["health"])
    app.include_router(users.router, prefix="/users", tags=["users"])
    return app

app = create_app()
```

`app/api/routes/health.py`：

```python
from fastapi import APIRouter

router = APIRouter()

@router.get("")
def health_check() -> dict[str, str]:
    return {"status": "ok"}
```

启动：

```bash
uvicorn app.main:app --reload
```

---

## 7. Schema、Service、Repository 分层示例

`app/schemas/user.py`：

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    email: EmailStr

class UserRead(BaseModel):
    id: int
    name: str
    email: EmailStr
```

`app/repositories/user_repository.py`：

```python
from app.schemas.user import UserCreate, UserRead

class UserRepository:
    def __init__(self) -> None:
        self._users: dict[int, UserRead] = {}
        self._next_id = 1

    def create(self, data: UserCreate) -> UserRead:
        user = UserRead(id=self._next_id, name=data.name, email=data.email)
        self._users[user.id] = user
        self._next_id += 1
        return user

    def get(self, user_id: int) -> UserRead | None:
        return self._users.get(user_id)
```

`app/services/user_service.py`：

```python
from app.repositories.user_repository import UserRepository
from app.schemas.user import UserCreate, UserRead

class UserService:
    def __init__(self, repository: UserRepository) -> None:
        self.repository = repository

    def create_user(self, data: UserCreate) -> UserRead:
        return self.repository.create(data)

    def get_user(self, user_id: int) -> UserRead | None:
        return self.repository.get(user_id)
```

`app/api/routes/users.py`：

```python
from fastapi import APIRouter, Depends, HTTPException, status

from app.repositories.user_repository import UserRepository
from app.schemas.user import UserCreate, UserRead
from app.services.user_service import UserService

router = APIRouter()
repository = UserRepository()

def get_user_service() -> UserService:
    return UserService(repository)

@router.post("", response_model=UserRead, status_code=status.HTTP_201_CREATED)
def create_user(
    data: UserCreate,
    service: UserService = Depends(get_user_service),
) -> UserRead:
    return service.create_user(data)

@router.get("/{user_id}", response_model=UserRead)
def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
) -> UserRead:
    user = service.get_user(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="user not found")
    return user
```

真实项目中 repository 应使用数据库会话，不应使用内存字典。

---

## 8. 数据库与迁移

常用组合：

| 场景 | 方案 |
| --- | --- |
| 关系型数据库 | PostgreSQL / MySQL |
| ORM | SQLAlchemy / Django ORM / Tortoise ORM |
| 迁移 | Alembic / Django migrations |
| 连接池 | SQLAlchemy engine pool / asyncpg pool |

SQLAlchemy 2.0 模型示例：

```python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
```

会话：

```python
from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.core.config import get_settings

engine = create_engine(get_settings().database_url, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## 9. 测试

测试类型：

| 类型 | 关注点 |
| --- | --- |
| 单元测试 | 单个函数、类、service |
| 集成测试 | 数据库、缓存、HTTP 客户端等外部依赖 |
| API 测试 | 路由、状态码、响应体 |
| 端到端测试 | 完整业务链路 |

`tests/test_health.py`：

```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)

def test_health_check() -> None:
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

运行：

```bash
pytest
```

---

## 10. 代码质量

推荐工具组合：

```bash
ruff check .
ruff format .
mypy app tests
pytest
```

可以用 `Makefile` 简化：

```makefile
.PHONY: lint format type test

lint:
	ruff check .

format:
	ruff format .

type:
	mypy app tests

test:
	pytest
```

---

## 11. Docker 化

`Dockerfile`：

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY pyproject.toml ./
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir "fastapi" "uvicorn[standard]" "pydantic-settings"

COPY app ./app

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`docker-compose.yml`：

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

---

## 12. 生产启动

开发环境常用：

```bash
uvicorn app.main:app --reload
```

生产环境常用：

```bash
gunicorn app.main:app \
  -k uvicorn.workers.UvicornWorker \
  --workers 4 \
  --bind 0.0.0.0:8000
```

生产环境要关注：

- worker 数量与 CPU、内存、I/O 模型匹配。
- 请求超时、连接池、反向代理超时需要一致。
- 日志输出到 stdout/stderr，交给平台采集。
- 健康检查、优雅关闭、迁移执行策略要明确。

