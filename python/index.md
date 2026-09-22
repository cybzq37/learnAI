## python高级语法

### 全局解释器锁（GIL）

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

### 深拷贝与浅拷贝

浅拷贝：只拷贝引用，不拷贝原数据。—— copy.copy()
深拷贝：拷贝引用的同时也会拷贝元数据。—— copy.deepcopy()

### 私有化

__xx : 在类中是私有属性方法，子类不能继承。
__xx__ ： 魔法方法，内置方法。子类可以继承
__xx : 在导入模块时禁止导入。

### *args 与 **kwarg

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

### 装饰器

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

### 生成器 Generator 

生成器是一种惰性求值的迭代器，用 `yield` 或生成器表达式创建。它一次只产出一个值，不把所有结果一次性放进内存，因此特别适合处理大数据流或无限序列。





