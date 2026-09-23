## 全局解释器锁（GIL）

GIL（Global Interpreter Lock） 是 CPython 解释器中的一个互斥锁，它保证同一时刻只有一个线程执行 Python 代码。也就是说，即使在多核 CPU 上，Python 的多线程也无法真正并行执行 Python 代码。

CPython 使用引用计数做内存管理。如果多个线程同时修改同一个对象的引用计数，可能导致内存泄漏或对象被提前释放。GIL 用最简单的方式解决了这个问题：同一时刻只让一个线程操作 Python 对象。

好处：
- 内存管理简单、安全
- 单线程性能好（无需细粒度锁）
- C 扩展编写简单

代价：
- 多线程无法利用多核做 CPU 密集计算

对于 CPU 密集型（计算、循环）应用，多线程几乎无加速，甚至更慢，用 multiprocessing 或多进程。  
对于 IO 密集型（网络、文件、数据库），能有效进行提速，用 threading / asyncio  

```python
from multiprocessing import Pool

def cpu_task(n):
    x = 0
    for _ in range(n):
        x += 1
    return x

if __name__ == "__main__":
    with Pool(4) as p:
        print(p.map(cpu_task, [10_000_000]*4))
```

```python
import asyncio, aiohttp

async def fetch(url):
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as r:
            return await r.text()
```

## 深拷贝与浅拷贝

浅拷贝：只拷贝引用，不拷贝原数据。—— copy.copy()
深拷贝：拷贝引用的同时也会拷贝元数据。—— copy.deepcopy()

## 私有化

__xx : 在类中是私有属性方法，子类不能继承。
__xx__ ： 魔法方法，内置方法。子类可以继承
__xx : 在导入模块时禁止导入。

## *args 与 **kwarg

它们是 Python 函数中用于接收可变数量参数的语法，名字里的 args / kwargs 只是惯例（arguments / keyword arguments），真正起作用的是 * 和 **。

`*args` 接收任意数量的位置参数，打包成一个元组

```python
def sum_all(*args):
    print(args)          # (1, 2, 3)
    print(type(args))    # <class 'tuple'>
    return sum(args)

sum_all(1, 2, 3)         # 6
sum_all()                # 0，不传也行
```

`**kwargs`：接收任意数量的关键字参数，打包成一个字典

```python
def show_info(**kwargs):
    print(kwargs)        # {'name': 'Tom', 'age': 18}
    print(type(kwargs))  # <class 'dict'>

show_info(name="Tom", age=18)
```

两者结合使用，顺序必须是 普通参数 -> `*args` -> `**kwargs`

```python
def func(a, b, *args, **kwargs):
    print(a, b)      # 1 2
    print(args)      # (3, 4)
    print(kwargs)    # {'x': 5, 'y': 6}

func(1, 2, 3, 4, x=5, y=6)
```

反向调用，`*` 和 `**` 不仅能"收集"，还能"拆开"传入。

```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
add(*nums)          # 等价于 add(1, 2, 3) → 6

d = {'a': 1, 'b': 2, 'c': 3}
add(**d)            # 等价于 add(a=1, b=2, c=3) → 6
```

解包也常用于转发参数：

```python
def wrapper(*args, **kwargs):
    print("调用前")
    result = func(*args, **kwargs)   # 原样转发给真正的函数
    print("调用后")
    return result
```

## 装饰器

装饰器本质是一个接收函数、返回新函数的可调用对象，用来在不修改原函数代码的前提下，增强或改变函数的行为。它是 Python 中"函数是一等公民"特性的典型应用。

```python
def my_decorator(func):
    def wrapper():
        print("调用前")
        func()
        print("调用后")
    return wrapper

@my_decorator
def say_hi():
    print("hi")

say_hi()
# 调用前
# hi
# 调用后
```

`@my_decorator` 等价于 `say_hi = my_decorator(say_hi)`

- 带参数装饰器
- 类装饰器
- `functools.wraps` 保留原函数元信息
- `functools.lru_cache` / `cache`
- `functools.singledispatch` 单分派泛型函数

```python
from functools import wraps

def deco(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

## 迭代器

迭代器就是一个可以被 `next()` 一个一个取出元素的对象。

```python
numbers = [10, 20, 30]

it = iter(numbers)  # 迭代器

print(next(it))  # 10
print(next(it))  # 20
print(next(it))  # 30
print(next(it))  # StopIteration
```

一个对象要成为迭代器，需要实现：

```python
__iter__()
__next__()

# 例如
class Counter:
    def __init__(self, n):
        self.n = n
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current < self.n:
            self.current += 1
            return self.current
        raise StopIteration
```

## 生成器

生成器是一种惰性求值的迭代器，用 `yield` 或生成器表达式创建。它一次只产出一个值，不把所有结果一次性放进内存。

```python
def numbers():
    yield 1
    yield 2
    yield 3

g = numbers()   # 生成器对象

print(next(g))  # 1
print(next(g))  # 2
print(next(g))  # 3
print(next(g))  # StopIteration
```

可以把 yield 理解成一种 **用户态的暂停 + 恢复**。

## async/await

async/await 是 Python 3.5+ 引入的异步编程语法，用于在单线程内高效处理大量 I/O 等待（网络请求、文件读写、数据库查询等）。


| 概念 | 定义 | 关键点 |
|------|------|--------|
| **协程 (coroutine)** | 用 `async def` 定义的函数，调用后返回**协程对象**，不会立即执行 | 函数 ≠ 执行；调用只是"创建计划" |
| **await** | 只能写在 `async def` 里的关键字；作用于可等待对象时，让当前协程暂停、把控制权交还事件循环，直到被等待对象完成 | 语法：只能在 `async def` 内<br>行为：挂起当前协程、让出事件循环 |
| **事件循环 (event loop)** | 调度协程执行的核心，负责在 I/O 就绪时恢复协程 | 单线程调度器；协程的"舞台" |
| **可等待对象 (awaitable)** | 能被 `await` 作用的对象，包括：协程、`Task`、`Future` | `await` 的作用对象，不限于协程 |

补充：可等待对象的三种类型

| 类型 | 来源 | 说明 |
|------|------|------|
| **协程 (coroutine)** | `async def` 函数调用结果 | 未被调度的"执行计划" |
| **Task** | `asyncio.create_task(coro)` | 已被事件循环调度的协程 |
| **Future** | `loop.create_future()` 等 | 代表未来结果的占位对象，低层原语 |

> **事件循环** 调度 **协程**；**await** 挂起当前协程、等待 **可等待对象** 完成，期间把控制权交还事件循环。

```python
import asyncio

# 定义与运行协程
async def hello():
    print("Hello")
    await asyncio.sleep(1)   # 模拟 I/O 等待，不阻塞事件循环
    print("World")

asyncio.run(hello())         # Python 3.7+ 推荐入口
```

> async def 函数不能直接调用执行，必须用 await 或 asyncio.run()，hello() 只是返回一个 coroutine 对象。

| API | 用途 | 说明 |
|-----|------|------|
| `asyncio.run(coro)` | 启动事件循环，运行主协程 | 程序入口，Python 3.7+ 推荐 |
| `asyncio.gather(*aws)` | 并发运行，收集结果 | 把多个协程/任务同时调度，按顺序返回结果列表 |
| `asyncio.create_task(coro)` | 包装成 Task 并调度 | 立即交给事件循环，可并行运行 |
| `asyncio.sleep(s)` | 异步睡眠（让出控制权） | 非阻塞等待，期间事件循环可执行其他任务 |
| `asyncio.wait_for(aw, t)` | 超时控制 | 超时抛出 `asyncio.TimeoutError` |
| `asyncio.Queue` | 异步队列 | 协程间安全的生产者-消费者通信 |
| `asyncio.Lock / Semaphore` | 异步同步原语 | 保护共享资源、限制并发数量 |

### async/await vs 多线程/多进程

| 方式 | 适用场景 | 特点 |
|------|----------|------|
| async/await | 高并发 I/O（网络、DB） | 单线程，开销极小，需异步库支持 |
| 多线程 | I/O 密集型，但库不支持异步 | 有 GIL，线程切换开销 |
| 多进程 | CPU 密集型 | 绕开 GIL，内存开销大 |

## 上下文管理器

实现 `__enter__` / `__exit__`，或使用 `contextlib`。

```python
from contextlib import contextmanager

@contextmanager
def tag(name):
    print(f"<{name}>")
    yield
    print(f"</{name}>")

with tag("h1"):
    print("hello")
```

常见用途：文件、锁、数据库连接、事务、临时目录。

## 闭包与作用域

- `nonlocal`：修改外层函数变量
- `global`：修改全局变量
- LEGB：Local → Enclosing → Global → Built-in

```python
def counter():
    n = 0
    def inc():
        nonlocal n
        n += 1
        return n
    return inc
```

注意：默认参数不要用可变对象，如 `def f(x=[])`。

## 函数参数高级用法

仅位置参数 `/`，仅关键字参数 `*` 

```python
def f(a, /, b, *, c):
    ...
```

- `a` 只能按位置传
- `b` 位置或关键字都行
- `c` 只能按关键字传

## 推导式、海象与解包

推导式

```python
[x * x for x in range(10) if x % 2 == 0]
{k: v for k, v in items}
{x for x in nums}
```

生成器表达式

```python
g = (x * x for x in range(10))
```

海象运算符 `:=`

```python
if (n := len(s)) > 10:
    print(n)
```

解包

```python
a, *mid, b = [1, 2, 3, 4, 5]
[*a, *b]
{**d1, **d2}
d1 | d2          # 字典合并，3.9+
d1 |= d2
```

## 模式匹配 match/case

Python 3.10+。

```python
match command.split():
    case ["go", direction]:
        print("go", direction)
    case ["drop", *items]:
        print("drop", items)
    case {"type": "user", "name": name}:
        print("user", name)
    case _:
        print("unknown")
```

支持字面量、序列、映射、类模式、守卫条件等。

## 类型注解与泛型

```python
from typing import TypeVar, Generic, Protocol, TypedDict, Literal, Final, overload

T = TypeVar("T")

class Repo(Protocol[T]):
    def get(self, id: int) -> T: ...

class User(TypedDict):
    name: str
    age: int

Mode = Literal["r", "w"]
MAX: Final = 100
```

常见：`list[int]`、`dict[str, int]`、`X | Y`、`Optional[X]`、`Callable`、`TypeVar`、`Generic`、`Protocol`、`TypedDict`、`Literal`、`Final`、`@overload`。

Python 3.12+ 支持 PEP 695：

```python
def first[T](items: list[T]) -> T:
    return items[0]

type Vector = list[float]
```

## 面向对象高级机制

- `@property` / setter / deleter
- `@classmethod`、`@staticmethod`
- `__slots__` 限制实例属性、节省内存
- 运算符重载：`__add__`、`__eq__`、`__len__`、`__getitem__`、`__iter__`
- `__call__` 让实例可调用
- 描述符协议：`__get__`、`__set__`、`__set_name__`
- 元类：`type`、`__new__`、`__init_subclass__`
- 抽象基类：`ABC`、`@abstractmethod`
- MRO、`super()`、多继承、Mixin

```python
class User:
    __slots__ = ("name", "age")

    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __call__(self):
        return f"{self.name}:{self.age}"
```

`property`、`classmethod`、`staticmethod` 的底层其实都和描述符有关。

## 数据类与枚举

```python
from dataclasses import dataclass, field
from enum import Enum

@dataclass
class User:
    name: str
    age: int = 0
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        self.name = self.name.strip()

class Color(Enum):
    RED = 1
    GREEN = 2
```

还有 `NamedTuple`、`IntEnum`、`StrEnum` 等。

## 生成器与迭代器进阶

```python
def gen():
    x = yield 1
    yield x

g = gen()
next(g)       # 1
g.send(10)    # 10
```

- `yield from`
- `send()`、`throw()`、`close()`
- `itertools`：`chain`、`groupby`、`islice`、`product`、`combinations`
- 生成器表达式

## 异步编程进阶

除了 `async/await`，还有：

- `async for`
- `async with`
- 异步生成器
- `asyncio.TaskGroup`，Python 3.11+
- `asyncio.timeout`
- `asyncio.Queue`
- `asyncio.Lock` / `Semaphore`
- `asyncio.gather`
- `asyncio.create_task`

```python
async def main():
    async with session.get(url) as resp:
        async for line in resp.content:
            print(line)
```

## 异常处理进阶

- `raise ... from ...` 异常链
- 自定义异常
- `try/except/else/finally`
- `ExceptionGroup` 与 `except*`，Python 3.11+

```python
try:
    ...
except ValueError as e:
    raise RuntimeError("处理失败") from e
```

```python
try:
    ...
except* ValueError as eg:
    ...
```

## 元编程与动态特性

- `getattr` / `setattr` / `hasattr`
- `__getattr__`、`__getattribute__`、`__setattr__`
- `inspect` 反射函数签名、类结构
- `ast` 解析和修改代码
- `exec` / `eval`
- `importlib` 动态导入
- `__init_subclass__`
- `__set_name__`

```python
class A:
    def __getattr__(self, name):
        return f"missing: {name}"
```

## 标准库常用高级工具

- `functools.lru_cache` / `cache`
- `functools.partial`
- `functools.singledispatch`
- `contextlib.suppress` / `ExitStack`
- `itertools`
- `pathlib`
- `weakref`
- `concurrent.futures`
- `multiprocessing`
- `asyncio`
- `typing`

## 版本特性速查

| 版本 | 新特性 |
|---|---|
| 3.8 | 海象运算符、f-string `=` 调试 |
| 3.9 | `list[int]`、字典合并 `|` |
| 3.10 | `match/case`、`X | Y` |
| 3.11 | `ExceptionGroup` / `except*`、`TaskGroup` |
| 3.12 | PEP 695 类型参数、f-string 增强 |