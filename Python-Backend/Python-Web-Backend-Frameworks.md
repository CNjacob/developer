# Python 服务端框架知识

## 1. 框架分类

Python 服务端框架大致分为四类：

| 类型 | 代表 | 特点 |
| --- | --- | --- |
| 全功能框架 | Django | 自带 ORM、Admin、认证、模板、迁移 |
| 轻量 WSGI 框架 | Flask | 简洁灵活，生态扩展丰富 |
| 异步 ASGI 框架 | FastAPI、Starlette、Sanic | 高并发 I/O、类型注解、异步支持 |
| RPC / 微服务框架 | gRPC、Nameko | 面向服务间通信 |

选择建议：

- 管理后台、CMS、传统业务系统：Django。
- 小型服务、内部工具、需要高度自由：Flask。
- API 服务、类型校验、异步 I/O、高并发网关：FastAPI。
- 服务间强契约通信：gRPC。

---

## 2. WSGI 与 ASGI

WSGI 是 Python Web 服务的同步接口标准，适合传统 HTTP 请求响应模型。

```text
Client -> Nginx -> Gunicorn/uWSGI -> WSGI App
```

ASGI 是异步接口标准，支持 HTTP、WebSocket、长连接和后台事件。

```text
Client -> Nginx -> Uvicorn/Hypercorn -> ASGI App
```

| 对比 | WSGI | ASGI |
| --- | --- | --- |
| 执行模型 | 同步 | 同步 + 异步 |
| WebSocket | 不原生支持 | 原生支持 |
| 长连接 | 支持较弱 | 更适合 |
| 代表框架 | Django、Flask | FastAPI、Starlette |
| 服务器 | Gunicorn、uWSGI | Uvicorn、Hypercorn、Daphne |

---

## 3. FastAPI

FastAPI 基于 Starlette 和 Pydantic，核心优势是类型注解、自动文档、请求校验、依赖注入和异步支持。

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ItemCreate(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create_item(item: ItemCreate) -> dict:
    return {"name": item.name, "price": item.price}
```

特点：

- OpenAPI 文档自动生成。
- Pydantic 做请求、响应模型校验。
- `Depends` 支持依赖注入。
- 可以混合同步函数和异步函数。
- 适合 REST API、BFF、网关、AI 服务接口。

依赖注入示例：

```python
from typing import Annotated

from fastapi import Depends, Header, HTTPException

def get_token(authorization: Annotated[str | None, Header()] = None) -> str:
    if not authorization:
        raise HTTPException(status_code=401, detail="missing token")
    return authorization.removeprefix("Bearer ")

@app.get("/me")
async def me(token: Annotated[str, Depends(get_token)]) -> dict:
    return {"token": token}
```

中间件示例：

```python
import time

from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.perf_counter() - start)
    return response
```

异步注意事项：

- `async def` 内不要调用阻塞 I/O。
- 数据库、HTTP 客户端、Redis 客户端要选择异步版本。
- CPU 密集型任务不要直接放在事件循环里执行。

---

## 4. Django

Django 是全功能 Web 框架，强调快速构建完整业务系统。

核心组件：

| 组件 | 作用 |
| --- | --- |
| ORM | 数据库模型和查询 |
| Admin | 自动后台管理 |
| Forms | 表单校验和渲染 |
| Auth | 用户、权限、会话 |
| Migrations | 数据库迁移 |
| Middleware | 请求响应处理链 |
| Templates | 服务端模板 |

模型示例：

```python
from django.db import models

class UserProfile(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

视图示例：

```python
from django.http import JsonResponse

def health(request):
    return JsonResponse({"status": "ok"})
```

Django REST Framework 常用于构建 API：

```python
from rest_framework import serializers, viewsets

from .models import UserProfile

class UserProfileSerializer(serializers.ModelSerializer):
    class Meta:
        model = UserProfile
        fields = ["id", "name", "email", "created_at"]

class UserProfileViewSet(viewsets.ModelViewSet):
    queryset = UserProfile.objects.all()
    serializer_class = UserProfileSerializer
```

Django 适合：

- 后台系统。
- 内容管理。
- 权限和用户体系复杂的业务。
- 需要快速交付 CRUD 的项目。

---

## 5. Flask

Flask 是轻量框架，本体很小，常通过扩展组合能力。

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.get("/health")
def health():
    return jsonify(status="ok")

@app.post("/users")
def create_user():
    data = request.get_json()
    return jsonify(id=1, name=data["name"]), 201
```

常用扩展：

| 扩展 | 用途 |
| --- | --- |
| Flask-SQLAlchemy | ORM |
| Flask-Migrate | 数据库迁移 |
| Flask-Login | 登录态 |
| Flask-JWT-Extended | JWT |
| Flask-WTF | 表单 |
| Flask-Caching | 缓存 |

Flask 适合小服务、内部工具和需要完全自定义架构的系统。大型 Flask 项目要尽早约束目录结构、依赖注入、错误处理和测试规范。

---

## 6. Starlette

Starlette 是轻量 ASGI 工具箱，也是 FastAPI 的底层基础之一。

```python
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route

async def health(request):
    return JSONResponse({"status": "ok"})

app = Starlette(routes=[Route("/health", health)])
```

适合构建基础设施层、网关、中间件和需要更低层 ASGI 控制的服务。

---

## 7. Tornado、Sanic、Quart

| 框架 | 说明 |
| --- | --- |
| Tornado | 早期异步网络框架，适合长连接、WebSocket |
| Sanic | 类 Flask 风格的异步 Web 框架 |
| Quart | Flask API 风格的 ASGI 框架 |

如果团队新建 API 服务，FastAPI 通常更容易获得类型校验、文档和生态收益；如果已有历史系统，继续使用 Tornado/Flask/Django 也很常见。

---

## 8. gRPC

gRPC 使用 Protocol Buffers 定义服务契约，适合服务间通信。

`user.proto`：

```proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  int64 id = 1;
  string name = 2;
}
```

优点：

- 强类型接口。
- 跨语言。
- 二进制协议效率高。
- 支持流式通信。

不足：

- 调试不如 HTTP JSON 直观。
- 浏览器直接调用需要额外代理。
- 接口演进需要管理 proto 兼容性。

---

## 9. GraphQL

Python GraphQL 常见库包括 Strawberry、Ariadne、Graphene。

适合场景：

- 前端需要灵活选择字段。
- 多端聚合多个后端资源。
- BFF 层聚合复杂数据。

示例：

```python
import strawberry

@strawberry.type
class User:
    id: int
    name: str

@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> User:
        return User(id=id, name="Alice")

schema = strawberry.Schema(query=Query)
```

注意事项：

- 要控制查询复杂度和深度。
- 要解决 N+1 查询问题。
- 鉴权粒度可能比 REST 更复杂。

---

## 10. 框架选型

| 需求 | 推荐 |
| --- | --- |
| 快速开发后台管理 | Django |
| 标准 CRUD + Admin | Django + DRF |
| 高效 API 服务 | FastAPI |
| 小型服务或脚本 HTTP 化 | Flask / FastAPI |
| WebSocket / 长连接 | FastAPI / Starlette / Tornado |
| 微服务内部强契约调用 | gRPC |
| 前端聚合查询 | GraphQL |

常见组合：

```text
Nginx
  -> Gunicorn + Django/Flask
  -> Uvicorn/Gunicorn + FastAPI
  -> Celery Worker
  -> PostgreSQL / Redis / Kafka
```

