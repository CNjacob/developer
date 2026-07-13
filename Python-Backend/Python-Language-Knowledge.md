# Python 语言核心知识

## 1. Python 的定位与特点

Python 是一门解释型、动态类型、强类型、多范式语言，常用于服务端开发、脚本自动化、数据分析、机器学习、测试工具和 DevOps。

| 特性 | 说明 |
| --- | --- |
| 动态类型 | 变量本身没有固定类型，运行时对象有类型 |
| 强类型 | 不会隐式把字符串和数字混合运算 |
| 解释执行 | 常见实现 CPython 会先编译为字节码再执行 |
| 多范式 | 支持过程式、面向对象、函数式风格 |
| 生态丰富 | PyPI、标准库和第三方库覆盖面广 |

```python
name = "Python"
version = 3.12

print(f"{name} {version}")
```

Python 的优势是开发效率高、表达力强、生态完整；劣势是 CPU 密集型场景受 GIL 和解释器开销影响，需要借助多进程、C 扩展、Rust 扩展或专用计算库优化。

---

## 2. 基础语法

Python 使用缩进表达代码块，通常使用 4 个空格。

```python
def is_adult(age: int) -> bool:
    if age >= 18:
        return True
    return False
```

常用内置类型：

| 类型 | 示例 | 特点 |
| --- | --- | --- |
| `int` | `1` | 任意精度整数 |
| `float` | `3.14` | 双精度浮点数 |
| `bool` | `True` | `bool` 是 `int` 的子类 |
| `str` | `"hello"` | 不可变 Unicode 字符串 |
| `list` | `[1, 2]` | 有序可变序列 |
| `tuple` | `(1, 2)` | 有序不可变序列 |
| `dict` | `{"a": 1}` | 哈希映射，保持插入顺序 |
| `set` | `{1, 2}` | 无序去重集合 |

字符串格式化推荐使用 f-string：

```python
user = "Alice"
score = 95.5
message = f"{user}: {score:.1f}"
```

---

## 3. 可变对象与不可变对象

不可变对象包括 `int`、`float`、`str`、`tuple`、`frozenset`；可变对象包括 `list`、`dict`、`set` 和大多数自定义对象。

```python
def append_item(items: list[int]) -> None:
    items.append(1)

values = []
append_item(values)
print(values)  # [1]
```

函数默认参数只在函数定义时创建一次，不要使用可变对象作为默认值：

```python
# 错误示例
def add_bad(item: int, items: list[int] = []) -> list[int]:
    items.append(item)
    return items

# 推荐写法
def add_good(item: int, items: list[int] | None = None) -> list[int]:
    if items is None:
        items = []
    items.append(item)
    return items
```

---

## 4. 控制流与推导式

```python
for i in range(3):
    print(i)

while True:
    break

result = "ok" if True else "fail"
```

推导式适合简洁表达转换、过滤和构造集合。

```python
numbers = [1, 2, 3, 4]
squares = [n * n for n in numbers]
even_squares = [n * n for n in numbers if n % 2 == 0]
mapping = {n: n * n for n in numbers}
unique = {n % 2 for n in numbers}
```

推导式过度嵌套会降低可读性，复杂逻辑应改为普通循环或拆函数。

---

## 5. 函数

Python 函数是一等对象，可以赋值、传参和返回。

```python
def greet(name: str, prefix: str = "Hello") -> str:
    return f"{prefix}, {name}"

def apply(value: int, fn) -> int:
    return fn(value)

print(apply(3, lambda x: x * 2))
```

参数类型：

```python
def create_user(
    name: str,
    age: int,
    *,
    email: str | None = None,
    active: bool = True,
) -> dict:
    return {
        "name": name,
        "age": age,
        "email": email,
        "active": active,
    }
```

`*` 后面的参数必须使用关键字传递，适合提高调用可读性。

闭包示例：

```python
def make_counter():
    count = 0

    def inc() -> int:
        nonlocal count
        count += 1
        return count

    return inc
```

---

## 6. 面向对象

Python 的类支持封装、继承、多态、描述符、属性和元类。业务代码中最常用的是普通类、数据类和抽象基类。

```python
class User:
    def __init__(self, user_id: int, name: str) -> None:
        self.user_id = user_id
        self.name = name

    def rename(self, name: str) -> None:
        if not name:
            raise ValueError("name is required")
        self.name = name
```

数据类适合承载结构化数据：

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: int
    currency: str
```

属性封装：

```python
class Account:
    def __init__(self, balance: int) -> None:
        self._balance = balance

    @property
    def balance(self) -> int:
        return self._balance
```

抽象基类：

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def get(self, user_id: int) -> dict | None:
        raise NotImplementedError
```

---

## 7. 模块、包与导入

一个 `.py` 文件就是模块，包含 `__init__.py` 的目录通常作为包。

```text
app/
  __init__.py
  main.py
  services/
    __init__.py
    user_service.py
```

导入方式：

```python
import os
from pathlib import Path
from app.services.user_service import UserService
```

建议：

- 标准库、第三方库、本地模块分组导入。
- 避免循环导入，必要时抽出公共模块或使用依赖注入。
- 不推荐大量使用 `from module import *`。
- 脚本入口使用 `if __name__ == "__main__":`。

```python
def main() -> None:
    print("run")

if __name__ == "__main__":
    main()
```

---

## 8. 异常处理

异常用于表达无法正常完成的流程，不应替代普通条件判断。

```python
class UserNotFoundError(Exception):
    pass

def get_user(user_id: int) -> dict:
    user = query_user(user_id)
    if user is None:
        raise UserNotFoundError(f"user {user_id} not found")
    return user
```

捕获异常时应尽量精确：

```python
try:
    value = int("abc")
except ValueError as exc:
    print(f"invalid number: {exc}")
finally:
    print("cleanup")
```

异常链：

```python
try:
    load_config()
except OSError as exc:
    raise RuntimeError("failed to load config") from exc
```

---

## 9. 文件、路径与上下文管理器

推荐使用 `pathlib` 处理路径。

```python
from pathlib import Path

path = Path("data/users.txt")
text = path.read_text(encoding="utf-8")
```

上下文管理器用于自动释放资源：

```python
with open("app.log", "a", encoding="utf-8") as file:
    file.write("hello\n")
```

自定义上下文管理器：

```python
from contextlib import contextmanager

@contextmanager
def transaction():
    print("begin")
    try:
        yield
        print("commit")
    except Exception:
        print("rollback")
        raise
```

---

## 10. 迭代器与生成器

可迭代对象实现 `__iter__`，迭代器实现 `__next__`。

```python
for item in [1, 2, 3]:
    print(item)
```

生成器使用 `yield` 惰性产出数据，适合处理大文件、分页数据和流式结果。

```python
def read_lines(path: str):
    with open(path, encoding="utf-8") as file:
        for line in file:
            yield line.rstrip("\n")
```

---

## 11. 装饰器

装饰器用于在不修改函数主体的情况下增强行为，例如日志、鉴权、缓存、事务和重试。

```python
from functools import wraps
from time import perf_counter

def timing(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        start = perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            cost = perf_counter() - start
            print(f"{fn.__name__} cost {cost:.3f}s")

    return wrapper

@timing
def work():
    return "done"
```

---

## 12. 类型注解

类型注解不会默认改变运行时行为，主要服务于 IDE、静态检查、文档表达和团队协作。

```python
from typing import Protocol

class UserRepo(Protocol):
    def get_by_id(self, user_id: int) -> dict | None:
        ...

def find_name(repo: UserRepo, user_id: int) -> str | None:
    user = repo.get_by_id(user_id)
    return user["name"] if user else None
```

常用工具：

| 工具 | 作用 |
| --- | --- |
| `mypy` | 静态类型检查 |
| `pyright` | 静态类型检查，VS Code/Pylance 生态常用 |
| `ruff` | 代码检查与格式化 |
| `pydantic` | 运行时数据校验与类型转换 |

---

## 13. 并发模型

Python 常见并发方式：

| 方式 | 适用场景 | 说明 |
| --- | --- | --- |
| 多线程 | I/O 密集型任务 | 受 GIL 影响，不适合纯 Python CPU 密集计算 |
| 多进程 | CPU 密集型任务 | 进程隔离，通信成本更高 |
| 协程 | 高并发 I/O | 基于事件循环，适合网络服务 |
| 任务队列 | 异步后台任务 | 跨进程、跨机器执行任务 |

线程：

```python
from concurrent.futures import ThreadPoolExecutor

def fetch(url: str) -> str:
    return url

with ThreadPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(fetch, ["a", "b", "c"]))
```

协程：

```python
import asyncio

async def fetch(name: str) -> str:
    await asyncio.sleep(1)
    return name

async def main() -> None:
    results = await asyncio.gather(fetch("a"), fetch("b"))
    print(results)

asyncio.run(main())
```

---

## 14. GIL

CPython 的 GIL 是全局解释器锁，它保证同一时刻通常只有一个线程执行 Python 字节码。它简化了内存管理，但限制了多线程执行 CPU 密集型 Python 代码的并行能力。

应对方式：

- I/O 密集型：使用线程池、`asyncio` 或异步 Web 框架。
- CPU 密集型：使用多进程、NumPy/Pandas 等释放 GIL 的库、C/C++/Rust 扩展。
- 服务端高并发：使用多 worker 进程加异步 I/O。

---

## 15. 常用标准库

| 模块 | 用途 |
| --- | --- |
| `pathlib` | 路径处理 |
| `os` / `sys` | 操作系统和解释器交互 |
| `json` | JSON 编解码 |
| `datetime` / `zoneinfo` | 日期时间和时区 |
| `logging` | 日志 |
| `argparse` | 命令行参数 |
| `subprocess` | 启动子进程 |
| `asyncio` | 异步 I/O |
| `concurrent.futures` | 线程池和进程池 |
| `unittest` | 单元测试 |
| `sqlite3` | SQLite 数据库 |
| `http.client` / `urllib` | HTTP 基础能力 |

---

## 16. Pythonic 编码习惯

- 优先可读性，不追求过度技巧。
- 使用 `with` 管理资源。
- 使用 `enumerate` 替代手动下标。
- 使用 `dict.get`、`setdefault`、`defaultdict` 简化字典逻辑。
- 使用 `pathlib` 替代手写路径拼接。
- 使用类型注解表达接口边界。
- 对外部输入做校验，不相信请求、配置和消息队列中的数据。

```python
for index, item in enumerate(items, start=1):
    print(index, item)
```

```python
from collections import defaultdict

groups: dict[str, list[str]] = defaultdict(list)
for user in users:
    groups[user["role"]].append(user["name"])
```

