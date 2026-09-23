## 可变类型 vs 不可变类型

- 不可变：`int`、`float`、`str`、`tuple`、`frozenset`
- 可变：`list`、`dict`、`set`

不可变类型：对象一旦创建，**不能修改对象内部的值**。如果你尝试修改，不会改动原对象，而是新建一个对象，变量引用指向新对象。  
可变类型：对象创建之后，可以**原地修改对象内部数据**，**对象内存地址不变**。  
元组小坑：元组本身不可变，但如果元组里面包含可变对象，可变对象内部可以改。  

函数传参：
- 传**不可变**：函数内部修改，不会影响外部变量（相当于传值）
- 传**可变**：函数内原地修改，外部对象跟着变（传引用）  

- 常见坑：函数**默认参数**使用可变对象  

```python
def func(item, lst=[]):
    lst.append(item)
    return lst

print(func(1)) # [1]
print(func(2)) # [1,2] 不是[2]！列表被共享了
```

## `is` 与 `==`

- `==` 比较值，`is` 比较身份（内存地址）
- 小整数缓存 `[-5, 256]`、短字符串驻留会导致 `is` 偶尔“看起来对”，但不要依赖
- 判断 `None` 用 `is None`

## 深拷贝与浅拷贝

浅拷贝：只拷贝引用，不拷贝原数据。—— copy.copy()  
深拷贝：拷贝引用的同时也会拷贝元数据。—— copy.deepcopy()  

```python
import copy

a = [[1, 2], [3, 4]]
b = copy.copy(a)      # 浅拷贝：外层新对象，内层共享
c = copy.deepcopy(a)  # 深拷贝：递归复制
```

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

## 私有化

__xx : 在类中是私有属性方法，子类不能继承。
__xx__ ： 魔法方法，内置方法。子类可以继承
__xx : 在导入模块时禁止导入。

## 变量查找规则

Python 变量查找规则 LEGB：`Local(本地) → Enclosing(嵌套外层) → Global(全局) → Builtin(内置)`

## *args 与 **kwarg  

它们是 Python 函数中用于接收可变数量参数的语法，名字里的 args / kwargs 只是惯例（arguments / keyword arguments），真正起作用的是 `*` 和 `**` 。

- `*args`：接收**位置参数**，打包成 **元组 tuple**  
- `**kwargs`：接收**关键字参数**，打包成 **字典 dict**  

```python
def sum_all(*args):
    print(args)          # (1, 2, 3)
    print(type(args))    # <class 'tuple'>
    return sum(args)

sum_all(1, 2, 3)         # 6
sum_all()                # 0，不传也行

def show_info(**kwargs):
    print(kwargs)        # {'name': 'Tom', 'age': 18}
    print(type(kwargs))  # <class 'dict'>

show_info(name="Tom", age=18)
```

**两者结合使用**：顺序必须是 普通参数 -> `*args` -> `**kwargs`

```python
def func(a, b, *args, **kwargs):
    print(a, b)      # 1 2
    print(args)      # (3, 4)
    print(kwargs)    # {'x': 5, 'y': 6}

func(1, 2, 3, 4, x=5, y=6)
```

上面是**定义函数**：收集参数  
下面是**调用函数**：把容器拆开传入参数  
**反向调用，拆包（解包）** ：调用函数时使用 * 和 **  

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

## 闭包与 `nonlocal`

闭包 = 内层函数 + 内层函数引用的外层非全局变量  
简单说：一个函数内部定义了另一个函数，**内层函数可以捕获外层函数的局部变量**，并且在外层函数执行结束后，这些变量依然保留，不会被销毁。  
nonlocal的含义：声明变量不是当前内层函数的局部变量，去外层嵌套函数作用域中寻找（不包含全局作用域 global）。  

```python
def outer(x):
    # 外层局部变量 x
    def inner(y):
        # 内层引用外层变量x
        return x + y
    # 返回内层函数对象，不是 inner()
    return inner

f1 = outer(10)
# outer函数此时已经执行完毕，但是x=10被闭包保存了
print(f1(2))  # 12
print(f1(5))  # 15

f2 = outer(20)
print(f2(3))  # 23
```

## 装饰器

一句话本质：装饰器就是一个接收函数作为参数、返回新函数的闭包。  
作用：**在不修改原函数代码、不修改原函数调用方式的前提下，给函数增加额外功能。**  
典型场景：日志、计时、权限校验、缓存、异常捕获。  

```python
def decorator(func):
    # inner 就是包装函数
    def inner():
        print("执行函数前：新增逻辑")
        func()   # 调用原来的函数
        print("执行函数后：新增逻辑")
    return inner  # 返回内层函数（闭包）

# 语法糖 @
@decorator
def hello():
    print("hello world")

hello()
```

追问：**如果原函数有参数怎么办？** 用 `*args, **kwargs` 万能参数，适配任意参数的被装饰函数。

```python
def decorator(func):
    def inner(*args, **kwargs):
        print("函数执行前")
        res = func(*args, **kwargs)  # 把参数传给原函数，接收返回值
        print("函数执行后")
        return res  # 返回原函数的返回值
    return inner

@decorator
def add(a, b):
    return a + b

print(add(1, 2))
```

**带参数的装饰器（装饰器工厂）**  

```python
def decorator_factory(flag):
    # 返回真正的装饰器
    def decorator(func):
        def inner(*args, **kwargs):
            if flag:
                print("开启日志")
            res = func(*args, **kwargs)
            return res
        return inner
    return decorator

# 先执行 decorator_factory(True)，拿到装饰器，再装饰函数
@decorator_factory(True)
def test():
    print("test run")

test()
```

**装饰后函数元信息丢失**：被装饰后的函数名 `__name__`、文档字符串 `__doc__` 变成 inner 的，不是原函数。  

```python
import functools

def decorator(func):

    @functools.wraps(func)  # 把原函数信息拷贝到inner
    def inner(*args, **kwargs):
        return func(*args, **kwargs)
    return inner

@decorator
def foo():
    """foo函数文档"""
    pass

print(foo.__name__) # foo，不加wraps会输出 inner
```

**类装饰器（实现 call）**  

普通函数装饰器：**函数返回函数**（闭包）  
类装饰器：**类的实例变成可调用对象**，`@MyDecorator` 会把原函数传入 `__init__`；后续调用 `hello()`，本质是调用实例的 `__call__` 方法。  

```python
class MyDecorator:
    
    def __init__(self, func):
        self.func = func

    def __call__(self, *args, **kwargs):
        print("类装饰器前置")
        res = self.func(*args, **kwargs)
        print("类装饰器后置")
        return res

@MyDecorator
def hello():
    print("hello")

hello()
```

**装饰器什么时候执行?**  

导入模块时就执行装饰（包装函数），不是调用函数的时候！  

```python
def dec(func):
    print("装饰执行！")
    def inner():
        func()
    return inner

@dec
def f():
    print("f run")
# 仅仅导入，还没调用f()，就已经打印：装饰执行！
```

## 魔法方法

魔法方法是内置回调方法，不用手动调用；`__new__`造对象，`__init__`初始化；`__call__`让实例可调用；`__getitem__`支持下标；`__enter__/__exit__`实现 with；运算符对应`__add__/__eq__`。 

`__new__` 和 `__init__`  
`__str__` vs `__repr__`  
`__getattr__` vs `__getattribute__`
`__call__` 
`__enter__` & `__exit__`


魔法方法：以**双下划线开头、双下划线结尾**，不需要手动调用，**在特定语法 / 操作时自动触发**。  

例如 `obj()` 自动调用 `__call__`，`+` 运算符自动调用 `__add__`。  

**实例创建与销毁**  

`__new__(cls, *args, **kwargs)`  

创建实例对象，**最先执行**，是静态方法；返回实例对象；`__new__` 负责造对象，`__init__` 负责初始化，单例模式常用 `__new__`。

`__init__(self, *args, **kwargs)`  

构造初始化方法，实例创建完成后，用来给实例属性赋值；`self` 就是实例。  

`__del__(self)`  

析构方法；对象被垃圾回收销毁时自动调用；**不是 `del obj` 立刻执行**，只是减少引用计数，不适合做资源释放，优先用上下文管理器。  

**字符串表示**  

`__str__(self)` `str(obj)` /print (obj) 时调用，**面向用户，友好可读字符串**。  

`__repr__(self)`  `repr(obj)` 调用，面向开发者，**准确描述对象，理想状态可以 eval 还原对象**。如果只定义 `__repr__`，没定义 `__str__`，print 会使用 `__repr__`。

**可调用对象（前面类装饰器用到）**  

`__call__(self, *args, **kwargs)`  对象加括号 `obj()` 自动触发，让实例变成可调用对象，**类装饰器核心魔法方法**。  

```python
class A:
    def __call__(self):
        print("call run")
a = A()
a() # 触发 __call__
```

**属性访问**  

`__getattr__(self, name)` 访问**不存在的属性**时触发；存在属性不会调用。  

`__getattribute__(self, name)`  **访问任何属性都会触发**（不管属性是否存在）；优先级高于 `__getattr__`。  

**容器类魔法方法（模拟 list/dict）**

1. `__getitem__(self, idx)`：`obj[idx]` 获取元素
2. `__setitem__(self, idx, val)`：`obj[idx] = val` 设置元素
3. `__delitem__(self, idx)`：`del obj[idx]` 删除元素
4. `__contains__(self, item)`：`item in obj` 判断是否包含，返回 bool
5. `__len__(self)`：`len(obj)`，返回长度

**运算符重载**  

- `__add__(self, other)`：`self + other`
- `__sub__(self, other)`：`self - other`
- `__mul__(self, other)`：`self * other`
- `__truediv__(self, other)`：`self / other`
- `__mod__(self, other)`：`self % other`

**比较运算符**  

- `__eq__(self, other)`：`==`
- `__ne__(self, other)`：`!=`
- `__lt__(self, other)`：`<`
- `__gt__(self, other)`：`>`
- `__le__(self, other)`：`<=`
- `__ge__(self, other)`：`>=`

**上下文管理器（with 语句，资源释放）**  

1. `__enter__(self)`：进入 with 代码块，返回值赋值给 `as` 后的变量  
2. `__exit__(self, exc_type, exc_val, exc_tb)`：退出 with 块，**无论是否异常都会执行**；参数可以拿到异常信息；返回 True 代表捕获异常。  

> 用来自动关闭文件、数据库连接。

```python
class MyContext:
    def __enter__(self):
        print("进入with")
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("离开with")
        return True # 捕获异常

with MyContext() as m:
    print("in with")
```

## 异常处理

异常：程序运行时发生错误（不是语法错误），如果不捕获，程序直接崩溃终止。  
- `try/except/else/finally` 执行顺序
- `raise ... from ...` 保留异常链
- 自定义异常继承 `Exception`

```python
try:
    # 可能抛出异常的代码块
    num = int(input("输入数字："))
    print(10 / num)
except ValueError:
    # 捕获指定异常：值错误
    print("你输入的不是数字！")
except ZeroDivisionError as e:
    # 捕获除零错误，并且把异常对象赋值给e
    print("不能除以0！", e)
except Exception as e:
    # 捕获所有其他异常（兜底，尽量不要放最前面）
    print("未知异常：", e)
else:
    # ✅ 没有发生异常的时候才执行！有异常不会走这里
    print("代码正常执行完毕，没有异常")
finally:
    # ✅ 无论是否异常，【一定会执行】，哪怕return也会执行
    print("finally 永远执行，常用于关闭文件、释放资源")
```

1. **try**：包裹有可能出错的代码。
2. **except**：捕获异常，可以写多个捕获不同异常；`as e` 获取异常实例。顺序：**子类异常放前面，父类 Exception 放最后**，否则父类会拦截所有异常。
3. **else**：try 里面代码**没有抛出异常**才执行；推荐：不要把所有代码塞 try，正常业务逻辑放 else，只把可能报错的放 try。
4. **finally**：无论是否异常，无论有没有 return，**一定执行**。典型用途：关闭文件、关闭数据库连接、释放锁。

**主动抛出异常 raise**  用来校验参数，主动中断逻辑

```python
age = int(input())
if age < 0:
    raise ValueError("年龄不能为负数")

try:
    1/0
except ZeroDivisionError as e:
    raise RuntimeError("业务出错") from e
```

**自定义异常**  

继承 `Exception`，不要继承 BaseException  

```python
class MyError(Exception):
    pass

try:
    raise MyError("自定义异常消息")
except MyError as e:
    print(e)
```

**常见内置异常**  

- `ValueError`：值错误，类型转换失败
- `ZeroDivisionError`：除 0
- `IndexError`：列表下标越界
- `KeyError`：字典找不到 key
- `TypeError`：类型错误，比如数字 + 字符串
- `AttributeError`：对象没有该属性
- `Exception`：大部分异常的父类
- `BaseException`：顶层父类（包含系统退出、中断，一般不捕获）

**finally 和 return 同时存在**  

```python
def test():
    try:
        return 10
    finally:
        print("finally执行")
print(test())

# finally执行
# 10
```

**except 捕获顺序问题**

```python
try:
    1/0
except Exception as e:
    print("通用异常")
except ZeroDivisionError:
    print("除零")
```

永远进不到 `ZeroDivisionError`，因为 `Exception` 是父类，提前捕获。

## 迭代器

两个核心概念：

1. **可迭代对象 Iterable**：可以被 `for` 循环遍历的对象；实现 `__iter__()`
2. **迭代器 Iterator**：用来产生迭代数据的对象；实现 `__iter__()` + `__next__()`

关系：**迭代器一定是可迭代对象；可迭代对象不一定是迭代器**，迭代器就是一个可以被 `next()` 一个一个取出元素的对象。  

**可迭代对象（Iterable）**  

只要实现 `__iter__` 方法，调用后返回一个迭代器。常见：`list`、`tuple`、`str`、`dict`、`set`、生成器。  

```python
lst = [1,2,3]
# lst 是可迭代对象，但不是迭代器
it = iter(lst)  # iter() 等价调用 lst.__iter__()，返回迭代器对象
print(it)
```

**迭代器（Iterator）**  

必须同时实现两个魔法方法：  

1. `__iter__()`：返回自身 `self`  
2. `__next__()`：返回下一个元素；没有元素时抛出 `StopIteration`  

```python
class MyIterator:
    def __init__(self, n):
        self.n = n
        self.current = 0
    
    def __iter__(self):
        # 迭代器的 __iter__ 返回自己
        return self
    
    def __next__(self):
        if self.current < self.n:
            val = self.current
            self.current += 1
            return val
        else:
            # 没有数据，抛出异常
            raise StopIteration

it = MyIterator(3)
print(next(it)) # 0
print(next(it)) # 1
print(next(it)) # 2
# print(next(it)) # StopIteration

# for循环会自动调用 iter()，不断 next()，捕获 StopIteration 结束
for i in MyIterator(3):
    print(i)
```

**迭代器特点**  

1. **单向不可逆，只能往前迭代，不能回退**
2. **一次性消费**，迭代完之后就空了，不能重复使用
3. 惰性取值：**一次只产生一个元素**，不一次性加载全部数据（生成器就是迭代器）  

## 生成器

生成器是一种特殊迭代器，不需要一次性把所有数据存到内存，按需产生数据，节省内存。  

生成器是一种惰性求值的迭代器，用 `yield` 或生成器表达式创建。它一次只产出一个值，不把所有结果一次性放进内存。

生成器有两种写法：  

1. **生成器函数**：用 `yield` 关键字，不是 return  
2. **生成器表达式**：`(i for i in range(10))`  

**yield 生成器函数**  

普通函数遇到 `return` 直接结束；生成器函数遇到 **`yield`**：  
1. 返回 yield 后面的值  
2. 暂停函数执行，保存当前函数上下文（局部变量、执行位置）  
3. 下次调用 `next()` 的时候，**从暂停的地方继续往下执行**  

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

当生成器函数执行完毕，自动抛出 `StopIteration`；for 循环会自动捕获这个异常。  

**生成器表达式**  

`[]` 列表推导式，一次性生成全部数据，占内存；  
`()` 生成器表达式，只是生成器对象，**惰性求值**  

```python
lst = [i for i in range(10)]   # 列表，全部加载到内存
g = (i for i in range(10))     # 生成器，惰性
print(next(g))
print(next(g))
```

**send () 方法**  

`next(g)` 等价于 `g.send(None)`  
`send(val)`：**向生成器内部 yield 表达式传入值**，作为 yield 的返回结果  

```python
def gen():
    res = yield 100
    print("收到：", res)
    yield 200

g = gen()
print(next(g))      # 执行到 yield 100，暂停
print(g.send(666))  # 把666传给 res，继续执行到 yield200
```

第一次调用生成器，不能直接 send (非 None)，必须先 next () /send (None) 启动生成器。

**yield from**  

简化生成器嵌套，用来**委派子生成器**，代替 for + yield  

```python
def sub_gen():
    yield 1
    yield 2

def gen():
    yield from sub_gen()  # 等价 for i in sub_gen(): yield i

for x in gen():
    print(x)
```

**和迭代器对比** 

- **迭代器**：只要实现 `__iter__` 和 `__next__` 的对象；手动写类可以实现迭代器
- **生成器**：**属于迭代器的一种**，是语法层面快速构建迭代器的方式（yield / 生成器表达式）

> 所有生成器都是迭代器，但迭代器不一定是生成器。


## async/await

async/await 是引入的异步编程语法，用于在单线程内高效处理大量 I/O 等待（网络请求、文件读写、数据库查询等）。  

底层：**协程（coroutine）**，单线程内的并发；遇到 IO 阻塞时，切换去干别的事，提升 IO 密集型任务效率。  

**基础概念**  

1. `async def`：定义**协程函数**
   - 调用协程函数，**不会执行函数内部代码**，而是返回一个**协程对象**
2. `await`：只能写在 `async def` 函数里面
   - `await xxx`：**挂起当前协程**，等待可等待对象（协程、任务、Future）完成；也就是被 `await` 作用的对象  
   - 挂起期间，事件循环可以去执行其他就绪的协程。

> 可等待对象（awaitable）三类：`coroutine`、`Task`、`Future`

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

**并发执行多个协程 asyncio.create_task ()**  

`create_task`：把协程包装成 Task，**立刻加入事件循环调度**，实现并发  

```python
import asyncio

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    # 创建任务，进入事件循环调度
    task1 = asyncio.create_task(say_after(1, "A"))
    task2 = asyncio.create_task(say_after(2, "B"))

    print("开始等待")
    await task1
    await task2

asyncio.run(main())
```

总耗时约 **2 秒**（并发），不是 1+2=3 秒。如果直接 `await say_after(1); await say_after(2)`，就是串行，总耗时 3 秒。


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

## 1. `functools` 常用工具

- `lru_cache` / `cache`
- `partial`
- `reduce`
- `singledispatch`

## 2. `itertools` 常用

- `chain`、`groupby`、`islice`、`product`、`permutations`、`combinations`

## 3. 类型注解进阶

```python
from typing import TypeVar, Generic, Optional, Union

T = TypeVar("T")

class Stack(Generic[T]):
    def push(self, x: T) -> None: ...
    def pop(self) -> T: ...
```

## 4. 鸭子类型与协议

```python
# 不关心类型，只关心行为
def quack(obj):
    obj.quack()
```

Python 3.8+ 的 `typing.Protocol` 支持结构化子类型。

## 十、面向对象与设计

- 继承、多继承 MRO（C3 算法）、`super()`
- `@property`、`@staticmethod`、`@classmethod` 区别
- 抽象基类 `abc.ABC`
- 元类 `type`、`__metaclass__`
- 设计模式：单例、工厂、观察者、装饰器模式

## 单例示例

```python
class Singleton:
    _instance = None
    def __new__(cls, *a, **kw):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

## 常用操作

- 列表去重保序：`list(dict.fromkeys(lst))`
- 字典按值排序：`sorted(d.items(), key=lambda x: x[1])`
- 反转字符串/列表：切片 `[::-1]`
- 两数之和、链表反转、二叉树遍历（前中后序 + 层序）
- 快排、归并、二分查找
- 动态规划：爬楼梯、最长递增子序列、背包
- 时间复杂度分析：`list` 尾部操作 O(1)，头部 `insert(0)` O(n)；`dict/set` 平均 O(1)


## 十三、工程实践

- 虚拟环境：`venv`、`poetry`
- 包管理：`pip`、`requirements.txt`、`pyproject.toml`
- 测试：`pytest`、`unittest`、`mock`
- 类型注解：`typing`、`mypy`
- 代码规范：`flake8`、`black`、`ruff`
- 日志：`logging` 模块，避免 `print`


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


## 面试题 

**题1：下面输出什么？**  

**题2：生成器表达式与列表推导的区别？**  

**题3：`yield` 和 `return` 区别？**  