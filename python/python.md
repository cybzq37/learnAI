https://github.com/hantmac/Python-Interview-Customs-Collection

## 一、语言基础

### 1. 可变与不可变类型

- 不可变：`int`、`float`、`str`、`tuple`、`frozenset`
- 可变：`list`、`dict`、`set`
- 常见坑：函数默认参数使用可变对象

对于可变对象，Python 既不是传统意义上的传值，也不是传统意义上的传引用，而是：

传对象引用（call by object reference），也叫 共享传参（call by sharing）。

调用函数时，实参对象的引用被复制一份，绑定给形参。形参和实参指向同一个对象。

### 2. `is` 与 `==`

- `==` 比较值，`is` 比较身份（内存地址）
- 小整数缓存 `[-5, 256]`、短字符串驻留会导致 `is` 偶尔“看起来对”，但不要依赖
- 判断 `None` 用 `is None`

### 3. 深浅拷贝

```python
import copy

a = [[1, 2], [3, 4]]
b = copy.copy(a)      # 浅拷贝：外层新对象，内层共享
c = copy.deepcopy(a)  # 深拷贝：递归复制
```

### 4. 闭包与 `nonlocal`

```python
def counter():
    n = 0
    def inc():
        nonlocal n
        n += 1
        return n
    return inc
```

### 5. 装饰器基础

装饰器本质是一个函数，它接收一个函数，返回一个新函数。

```python
@log
def say_hello(name):
    print(f"hello {name}")

# 等价于
def say_hello(name):
    print(f"hello {name}")

say_hello = log(say_hello)
```

```python
import functools

def log(func):          # func 是传进来的原函数，比如 say_hello
    
    # 保留被装饰函数的名称、文档字符串、参数签名等元信息，让装饰后的函数看起来还是原来的函数。
    @functools.wraps(func) 
    def wrapper(*args, **kwargs):   # 新函数，用来替代原函数
        print(f"call {func.__name__}")  # 先做点额外事情：打印日志
        return func(*args, **kwargs)    # 再调用原函数，并把结果返回
    return wrapper      # 把新函数返回出去
```

追问点：`functools.wraps` 的作用、带参数的装饰器、类装饰器。

### 6. `*args` / `**kwargs` 解包

- `*args`：收集位置参数
- `**kwargs`：收集关键字参数

```python
def f(*args, **kwargs):
    print("args =", args)
    print("kwargs =", kwargs)

f(1, 2, x=3, y=4)

# 输出
args = (1, 2)
kwargs = {'x': 3, 'y': 4}
```

### 7. 异常处理

- `try/except/else/finally` 执行顺序
- `raise ... from ...` 保留异常链
- 自定义异常继承 `Exception`

### 8. 魔法方法

`__init__`、`__new__`、`__str__`、`__repr__`、`__len__`、`__getitem__`、`__call__`、`__enter__/__exit__`（上下文管理器）

## 二、生成器与 yield（重点）

### 1. 生成器函数 vs 生成器表达式

```python
# 生成器函数
def gen():
    yield 1
    yield 2

# 生成器表达式（类似列表推导，但惰性）
g = (x * x for x in range(5))
```

区别：

- 函数用 `yield`，表达式用 `()` 包裹
- 两者都返回 `generator` 对象，都是迭代器
- 列表推导立即求值，生成器表达式惰性求值

### 2. 生成器的执行流程

调用生成器函数**不会执行函数体**，只返回生成器对象。第一次 `next()` 才开始执行，遇到 `yield` 暂停并返回值，下次 `next()` 从暂停处继续。

```python
def demo():
    print("start")
    x = yield 1
    print("received:", x)
    y = yield 2
    print("received:", y)

g = demo()
print(next(g))       # start → 1
print(g.send(10))    # received: 10 → 2
print(g.send(20))    # received: 20 → StopIteration
```

### 3. `send` / `throw` / `close`

- `send(value)`：向生成器传值，值成为当前 `yield` 表达式的返回值
- `throw(exc)`：在暂停处抛出异常
- `close()`：在暂停处抛 `GeneratorExit`，生成器结束

```python
def g():
    try:
        while True:
            x = yield
            print(x)
    except GeneratorExit:
        print("closed")

gen = g()
next(gen)
gen.send("hello")   # hello
gen.close()         # closed
```

注意：第一次启动必须用 `next()` 或 `send(None)`，不能直接 `send` 非 `None` 值。

### 4. `yield from`

委托给另一个可迭代对象/生成器，简化嵌套生成器。

```python
def chain(*iterables):
    for it in iterables:
        yield from it

list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]
```

`yield from` 还能自动传递 `send`/`throw`/`close`，并获取子生成器的 `return` 值：

```python
def sub():
    yield 1
    return "done"

def main():
    result = yield from sub()
    print(result)   # done

list(main())  # [1]
```

### 5. 生成器的 `return`

Python 3 中生成器可以 `return value`，值会作为 `StopIteration.value` 抛出。

```python
def g():
    yield 1
    return 42

gen = g()
next(gen)
try:
    next(gen)
except StopIteration as e:
    print(e.value)   # 42
```

### 6. 生成器的状态与检查

```python
import inspect

def g():
    yield 1

gen = g()
inspect.getgeneratorstate(gen)  # 'GEN_CREATED'
next(gen)
inspect.getgeneratorstate(gen)  # 'GEN_SUSPENDED'
```

状态：

- `GEN_CREATED`
- `GEN_RUNNING`
- `GEN_SUSPENDED`
- `GEN_CLOSED`

判断生成器：

```python
import types
isinstance(gen, types.GeneratorType)
# 或
inspect.isgenerator(gen)
```

### 7. 生成器的优势与代价

**优势**

- 惰性求值，省内存，可处理无限序列
- 代码简洁，天然实现迭代器协议

**代价**

- 只能单向遍历一次，不能回退
- 每次 `next` 有状态切换开销，纯计算场景不一定比列表快
- 调试稍麻烦

### 8. 经典场景

- 读大文件逐行处理
- 无限序列（斐波那契、素数）
- 数据管道 `pipeline`

```python
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

def filter_empty(lines):
    for line in lines:
        if line:
            yield line

def to_upper(lines):
    for line in lines:
        yield line.upper()

pipeline = to_upper(filter_empty(read_lines("a.txt")))
for line in pipeline:
    print(line)
```

### 9. 生成器实现协程（历史）

在 `async/await` 之前，生成器被用来实现协程（`@coroutine`、`yield from`）。现在 `async def` 内部本质仍是协程对象，但底层机制与生成器同源。理解 `yield` 的挂起/恢复，对理解 `await` 很有帮助。

### 10. 常见面试题

**题1：下面输出什么？**

```python
def g():
    print("a")
    yield 1
    print("b")
    yield 2

gen = g()
print("c")
print(next(gen))
```

答案：`c` → `a` → `1`。因为调用 `g()` 不执行函数体。

**题2：生成器表达式与列表推导的区别？**

- 列表推导立即求值，返回 list
- 生成器表达式惰性求值，返回 generator，只能迭代一次

**题3：`yield` 和 `return` 区别？**

- `return` 结束函数并返回值
- `yield` 暂停函数，保留状态，下次继续，可多次产出

---

## 三、迭代器协议与可迭代对象

```python
class MyRange:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return MyRangeIterator(self.n)

class MyRangeIterator:
    def __init__(self, n):
        self.i = 0
        self.n = n

    def __iter__(self):
        return self

    def __next__(self):
        if self.i >= self.n:
            raise StopIteration
        self.i += 1
        return self.i - 1
```

要点：

- 可迭代对象实现 `__iter__`，返回迭代器
- 迭代器实现 `__iter__` + `__next__`
- 迭代器耗尽后再 `next` 一直抛 `StopIteration`
- `for` 循环本质：先 `iter()` 再反复 `next()`，捕获 `StopIteration`

---

## 四、上下文管理器

```python
class Timer:
    def __enter__(self):
        import time
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"cost {time.time() - self.start:.3f}s")
        return False   # False 表示不吞异常

with Timer():
    sum(range(10**6))
```

用 `contextlib` 更简洁：

```python
from contextlib import contextmanager

@contextmanager
def timer():
    import time
    start = time.time()
    try:
        yield
    finally:
        print(time.time() - start)
```

`__exit__` 返回 `True` 会吞掉异常，这是常见考点。

---

## 五、装饰器进阶

### 1. 带参数的装饰器

```python
def retry(times=3):
    def deco(func):
        @functools.wraps(func)
        def wrapper(*a, **kw):
            for i in range(times):
                try:
                    return func(*a, **kw)
                except Exception:
                    if i == times - 1:
                        raise
        return wrapper
    return deco

@retry(times=5)
def unstable():
    ...
```

### 2. 类装饰器

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *a, **kw):
        self.count += 1
        return self.func(*a, **kw)
```

### 3. 装饰器叠加顺序

```python
@a
@b
def f(): ...
# 等价于 f = a(b(f))，执行时 a 在外层
```

---

## 六、元类与 `__new__`

```python
class Meta(type):
    def __new__(mcs, name, bases, ns):
        ns["created"] = True
        return super().__new__(mcs, name, bases, ns)

class A(metaclass=Meta):
    pass

A.created  # True
```

`__new__` 负责创建实例，`__init__` 负责初始化。`__new__` 是静态方法，第一个参数是 `cls`。

---

## 七、描述符

实现 `__get__`、`__set__`、`__delete__` 的类叫描述符，是 `property`、`classmethod`、`staticmethod` 的底层机制。

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return obj.__dict__[self.name]

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError("must be positive")
        obj.__dict__[self.name] = value

class Order:
    price = Positive()
```

数据描述符（有 `__set__`）优先于实例字典，非数据描述符反之。

---

## 八、并发与异步

### 1. GIL

- CPython 中同一时刻只有一个线程执行字节码
- 影响：CPU 密集多线程无加速，IO 密集仍有效
- 规避：多进程 `multiprocessing`、C 扩展、`asyncio`

### 2. 多线程 vs 多进程 vs 协程

| 方式 | 适用场景 | 关键点 |
|---|---|---|
| threading | IO 密集 | GIL 限制 CPU 并行 |
| multiprocessing | CPU 密集 | 进程间通信用 Queue/Pipe |
| asyncio | 高并发 IO | 单线程事件循环，需 async/await |

### 3. asyncio 深入

**协程对象**

```python
async def f():
    return 1

coro = f()   # 协程对象，不执行
coro.send(None)  # 启动，抛 StopIteration(1)
```

**`await` 的本质**

挂起当前协程，把控制权交回事件循环，等被 await 的对象完成后再恢复。可 await 的对象：协程、`Task`、`Future`。

**Task 与 Future**

- `Future`：底层占位对象，表示未来结果
- `Task`：`Future` 子类，包装协程并调度执行

```python
async def main():
    task = asyncio.create_task(f())
    await task
```

**并发控制**

```python
sem = asyncio.Semaphore(10)

async def worker(i):
    async with sem:
        await asyncio.sleep(1)
```

**常见坑**

- 在协程里调用阻塞函数（`time.sleep`、`requests`）会卡住事件循环，要用 `run_in_executor` 或异步库
- `asyncio.gather` 默认遇到异常会取消其他任务，可加 `return_exceptions=True`

**asyncio 示例**

```python
import asyncio

async def task(n):
    await asyncio.sleep(n)
    return n

async def main():
    results = await asyncio.gather(task(1), task(2))
    print(results)

asyncio.run(main())
```

### 4. 线程安全

- `list.append`、`dict` 单操作有 GIL 保护，但复合操作需锁
- 用 `threading.Lock`、`queue.Queue`

---

## 九、内存与性能

- `sys.getsizeof`、`__slots__` 省内存
- 引用计数 + 标记清除 + 分代回收
- 循环引用用 `gc` 处理，`weakref` 避免强引用
- 性能优化：局部变量、内置函数、`join` 拼接字符串、生成器替代列表

### 垃圾回收三件套

- 引用计数：即时回收，但处理不了循环引用
- 标记清除：处理循环引用
- 分代回收：0/1/2 三代，越老越少扫

### `__slots__`

```python
class Point:
    __slots__ = ("x", "y")
```

省内存、限制属性，但失去 `__dict__`，不能动态加属性，多继承有限制。

### 字典底层

Python 3.7+ 保证插入顺序。3.6 起用紧凑 dict 实现，内存更省。哈希冲突用开放寻址。

---

## 十、面向对象与设计

- 继承、多继承 MRO（C3 算法）、`super()`
- `@property`、`@staticmethod`、`@classmethod` 区别
- 抽象基类 `abc.ABC`
- 元类 `type`、`__metaclass__`
- 设计模式：单例、工厂、观察者、装饰器模式

### 单例示例

```python
class Singleton:
    _instance = None
    def __new__(cls, *a, **kw):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

---

## 十一、函数与高级特性

### 1. `functools` 常用工具

- `lru_cache` / `cache`
- `partial`
- `reduce`
- `singledispatch`

### 2. `itertools` 常用

- `chain`、`groupby`、`islice`、`product`、`permutations`、`combinations`

### 3. 类型注解进阶

```python
from typing import TypeVar, Generic, Optional, Union

T = TypeVar("T")

class Stack(Generic[T]):
    def push(self, x: T) -> None: ...
    def pop(self) -> T: ...
```

### 4. 鸭子类型与协议

```python
# 不关心类型，只关心行为
def quack(obj):
    obj.quack()
```

Python 3.8+ 的 `typing.Protocol` 支持结构化子类型。

### 5. `__init_subclass__` 与 `__set_name__`

钩子方法，用于框架开发。

---

## 十二、数据结构与算法

### 常用操作

- 列表去重保序：`list(dict.fromkeys(lst))`
- 字典按值排序：`sorted(d.items(), key=lambda x: x[1])`
- 反转字符串/列表：切片 `[::-1]`
- 两数之和、链表反转、二叉树遍历（前中后序 + 层序）
- 快排、归并、二分查找
- 动态规划：爬楼梯、最长递增子序列、背包
- 时间复杂度分析：`list` 尾部操作 O(1)，头部 `insert(0)` O(n)；`dict/set` 平均 O(1)

---

## 十三、工程实践

- 虚拟环境：`venv`、`poetry`
- 包管理：`pip`、`requirements.txt`、`pyproject.toml`
- 测试：`pytest`、`unittest`、`mock`
- 类型注解：`typing`、`mypy`
- 代码规范：`flake8`、`black`、`ruff`
- 日志：`logging` 模块，避免 `print`

---

## 十四、高频手写题

1. 实现 `flatten` 嵌套列表（生成器版）
2. 实现 `take`、`chunk`、`window` 生成器工具
3. 实现 LRU 缓存（`OrderedDict` 或双向链表 + dict）
4. 实现 `retry` 装饰器
5. 实现上下文管理器 `timer`
6. 实现生产者-消费者（`Queue` 或 `asyncio.Queue`）
7. 用生成器实现斐波那契、素数筛
8. 手写 `deepcopy` 简化版
9. 实现带 `send` 的协程调度器（理解 `yield from`）
10. 统计词频并按频率排序
11. 判断括号匹配
12. 合并两个有序链表

## 十五、面试建议

- 基础题要答得准，比如 `is`/`==`、GIL、深浅拷贝，这些是区分度所在
- 算法题用 Python 写要熟悉内置数据结构，能省不少代码
- 项目经历准备 2-3 个，能讲清难点、取舍、量化结果
- 遇到不会的，先讲思路再写代码，面试官更看重思考过程
- 生成器、`yield`、迭代器、上下文管理器、装饰器、描述符是 Python 进阶高频区，建议重点掌握