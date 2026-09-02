# C++ 面试题

## C/C++ 语言基础

### C 与 C++ 的区别

- C 是面向过程语言，C++ 在 C 之上增加了**面向对象（封装/继承/多态）**、**模板泛型**、**异常**、**运算符重载**、**命名空间**等；
- C++ 类型检查更严格：函数重载、引用、默认参数、`bool`/`string`/STL；
- 动态内存管理：C 用 `malloc/free`（不调用构造析构），C++ 用 `new/delete`；
- C++ 兼容 C 语法，可用 `extern "C"` 链接 C 代码；
- 一句话：**C++ = C + 类与对象 + 泛型 + 标准库 + 现代语言特性**。

### const 的作用（必背）

1. 修饰普通变量：值不可修改；
2. 修饰指针，注意**位置**：
   - `const int *p`：`p` 指向的内容不可改，指针本身可改（指向常量的指针）；
   - `int * const p`：指针本身不可改，指向的内容可改（常量指针）；
   - `const int * const p`：两者都不可改。
3. 修饰函数参数：形参在函数内不可修改，防止误改；传引用时 `const T&` 还避免了拷贝；
4. 修饰返回值：返回的引用/指针不可被外部修改；
5. 修饰成员变量：必须在**构造函数的初始化列表**或类内初始化器中初始化；
6. 修饰成员函数（`void f() const`）：承诺不修改成员变量（`mutable` 成员除外），**const 对象只能调用 const 成员函数**；
7. 底层/顶层 const（C++ 术语）：`const int*` 是 low-level const（指向 const），`int* const` 是 top-level const（本身 const）。

> 面试延伸：类成员函数是否加 const，会影响重载决议；逻辑上不改对象的函数都应声明为 `const`。

### #define、const、enum 的区别

| 维度 | #define | const | enum |
|---|---|---|---|
| 处理阶段 | 预处理文本替换 | 编译期 | 编译期 |
| 类型检查 | 无 | 有 | 有 |
| 作用域 | 全局（可用 #undef） | 遵循作用域 | 作用域内（C++11 enum class 更严格） |
| 调试可见 | 否（已替换） | 是 | 是 |
| 内存 | 不占内存（无实体） | 可能占内存（取地址时） | 通常是编译期常量，可取地址占内存 |

```cpp
#define PI 3.14        // 文本替换，宏名在符号表中不存在
const double pi = 3.14; // 有类型、有作用域
enum Color { Red, Green, Blue }; // 整型常量
```

> C++ 中优先用 `const`/`constexpr`/`enum class` 替代宏常量；宏仅用于条件编译、include guard 等场景。

### static 的作用

- **局部 static 变量**：存储在静态区，首次执行时初始化一次，生命周期到程序结束；C++11 起局部 static 初始化**线程安全**（常用于单例）；
- **文件作用域 static（函数/全局变量）**：限定在本翻译单元内可见（内部链接），避免跨文件符号冲突；C++ 更推荐匿名命名空间；
- **类的 static 成员变量**：属于类而不是某个对象，所有对象共享一份，需类外定义（C++17 起可用 `inline static` 直接类内初始化）；
- **类的 static 成员函数**：没有 `this` 指针，只能访问 static 成员，不能访问非 static 成员；
- 注意与 `static_cast`（类型转换）、`static_assert`（编译期断言）区分。

### extern "C" 的作用

- C++ 支持函数重载，编译时会对函数名做**名字修饰（name mangling）**；C 编译器不做；
- `extern "C"` 告诉编译器按 C 的方式生成符号，避免名字修饰，使 C++ 能链接 C 库、或向 C 导出 C++ 函数；

```cpp
extern "C" {
    #include "c_header.h"      // C 头文件
}

extern "C" void c_function(int); // 以 C 符号规则导出
```

### volatile 的作用与误区

- 告诉编译器该变量的值可能在程序控制流之外被改变（硬件寄存器、信号处理、多线程共享），不要做寄存器缓存优化，每次都从内存读写；
- 常见用法：嵌入式寄存器地址、中断/信号中修改的标志变量；
- **误区**：C++ 中 `volatile` **不是同步原语**，不提供原子性和内存序，不能替代 `std::atomic`/互斥锁（与 Java 的 volatile 语义不同）。

### mutable 的作用

- 让** const 成员函数**可以修改该成员变量；
- 典型场景：缓存、统计计数、互斥锁等"逻辑上不影响对象状态"的成员。

```cpp
class Cache {
    mutable int hits = 0;   // 即使 const 对象也可累加
public:
    int query() const { ++hits; return 0; }
};
```

### explicit 的作用

- 防止**隐式类型转换/隐式构造**；
- 只有一个参数的构造函数默认可当隐式转换函数，加上 `explicit` 后必须显式调用；
- C++11 起还支持 `explicit operator bool()` 等显式转换运算符，避免 `if (obj)` 意外的隐式转换。

```cpp
class String {
public:
    explicit String(int n);   // 禁止 String s = 100;
};
```

### inline 函数与宏的区别

- **宏**：预处理器文本替换，无类型检查、无作用域、调试困难，不能访问类私有成员；
- **inline 函数**：真正的函数，有类型检查和作用域，通常定义在头文件中（否则多翻译单元链接可能违背 ODR）；
- `inline` 只是**建议**，编译器可能不展开；递归、虚函数、函数指针调用等往往无法真正内联；
- C++17 起 `inline` 也可修饰变量，允许头文件中定义多个翻译单元共享的变量。

### 引用与指针的区别（高频必问）

| 维度 | 引用 | 指针 |
|---|---|---|
| 定义 | 必须初始化，是"别名" | 可不初始化，可存地址 |
| 是否可改绑 | 初始化后不可重新绑定 | 可以指向其他对象 |
| 空值 | 语义上不能为空（有悬垂风险） | 可为 nullptr |
| 运算 | 无指针算术、无引用的引用/数组 | 支持 ++/--/加减 |
| sizeof | 等于所引用对象的大小 | 固定为地址大小 |
| 用途 | 函数参数/返回值、运算符重载 | 动态对象、可空参数、数组遍历 |

```cpp
int a = 10;
int& r = a;      // 引用：r 就是 a 的别名
int* p = &a;     // 指针：存放 a 的地址
r = 20;          // 修改 a
```

> 底层实现上引用通常就是个指针常量，但语言语义完全不同。**普通非 const 左值引用不能绑定临时对象**；`const T&` 和 `T&&` 可以。

### 值传递、指针传递、引用传递

- **值传递**：拷贝一份实参，函数内修改不影响外部；适合小对象；
- **指针传递**：拷贝指针本身，但可通过解引用修改指向的对象；指针为 null 时需要判空；
- **引用传递**：不拷贝，直接操作原对象；语义上更安全（不能为空）；
- 只读且不想拷贝时用 `const T&`；需要"可选/可空/改指向"时用指针。

```cpp
void f1(int x)     { x = 1; }          // 不影响实参
void f2(int* p)    { *p = 1; }         // 修改实参指向的值
void f3(int& x)    { x = 1; }          // 修改实参本身
```

### 左值、右值与移动语义（必问）

**概念**：

- **左值（lvalue）**：有名字、可取地址、生命周期超过当前表达式，如变量；
- **右值（rvalue）**：临时对象/字面量，表达式结束就销毁；
  - prvalue（纯右值）：`10`、`a + b`、临时对象；
  - xvalue（将亡值）：`std::move(obj)` 的结果；
- **右值引用 `T&&`**：只能绑定右值，把即将销毁的资源"接住"再移走，避免深拷贝；
- `std::move` 本质是**类型转换**（转成右值引用），不搬移任何东西，只是让编译器选移动构造函数/移动赋值；
- `std::forward` 用于**完美转发**：在转发函数中按实参原本的左/右值属性继续转发。

```cpp
std::vector<int> a(1000, 1);
std::vector<int> b = a;              // 拷贝：深复制
std::vector<int> c = std::move(a);   // 移动：偷走 a 的堆内存，a 变空

template <typename T>
void relay(T&& arg) {                // 转发引用（通用引用）
    consume(std::forward<T>(arg));   // 保持左右值属性
}
```

**移动后源对象状态**：合法但未指定（通常是空/可析构、可重新赋值状态），不应假设其内容。

### 深浅拷贝与拷贝控制三/五法则（高频）

- **浅拷贝**：逐成员复制，指针成员只复制地址，新旧对象**共享堆内存** → 析构时二次释放（double free）；
- **深拷贝**：为指针成员重新分配内存并复制内容，对象之间完全独立；
- **三法则（Rule of Three）**：如果自定义了析构函数、拷贝构造、拷贝赋值中的一个，通常三个都需要；
- **五法则（Rule of Five）**：再加上移动构造、移动赋值；
- **零法则（Rule of Zero）**：尽量让类只持有 RAII 资源成员（如 `unique_ptr`/`string`/`vector`），让编译器生成的默认拷贝/移动正确工作。

```cpp
class Buffer {
    char* data_ = nullptr;
public:
    ~Buffer() { delete[] data_; }
    Buffer(const Buffer& o) : data_(new char[o.size()]) { /* 深拷贝 */ }
    Buffer& operator=(const Buffer& o) { /* 拷贝并自赋值安全 */ return *this; }
    Buffer(Buffer&& o) noexcept : data_(o.data_) { o.data_ = nullptr; } // 移动
};
```

> 只要声明了析构函数等拷贝操作之一，编译器就可能不再隐式生成移动操作；反之移动操作也可能被抑制。使用 `= default` 可显式要求。

### 四种类型转换（必背）

| 转换 | 用途 | 安全性 |
|---|---|---|
| `static_cast<T>(x)` | 相关类型间转换：数值、`void*`、基类/派生类指针 | 编译期检查；向下转换不做运行时检查 |
| `dynamic_cast<T>(x)` | 多态类型向下/交叉转换 | 运行时检查（RTTI），失败返回 nullptr / 抛 bad_cast |
| `const_cast<T>(x)` | 去掉/加上 const 或 volatile | 仅去掉属性；若原对象真为 const，修改它是未定义行为 |
| `reinterpret_cast<T>(x)` | 按位重新解释（如指针↔整数） | 底层、可移植性差，慎用 |

```cpp
Base* b = dynamic_cast<Base*>(&d);   // 需要基类有虚函数（多态类型）
const int* cp = &i;
int* p = const_cast<int*>(cp);       // 去掉 const
```

### sizeof 与 strlen 的区别

- `sizeof` 是**编译期**运算符，返回对象/类型占用的字节数（含填充），不对表达式求值；
- `strlen` 是**运行期**函数，从指针开始扫描到 `'\0'` 返回字符个数；

```cpp
char s[] = "hello";      // sizeof(s) == 6（含 '\0'），strlen(s) == 5
char* p = s;             // sizeof(p) == 8（64 位下指针大小）
```

> 数组作为函数参数会**退化为指针**，函数内 `sizeof(arr)` 得到的是指针大小；空类 `sizeof` 为 1，含虚函数会多一个 vptr。

### 字节对齐（内存对齐，高频）

- CPU 按字长取数据，结构体成员地址应尽量是其自身对齐数的整数倍，编译器会**插入填充字节（padding）**；
- 结构体总大小对齐到**最大成员对齐数**的整数倍；
- 对齐换来更快的访问速度，代价是内存浪费；通过**按成员大小降序排列**可减少填充；

```cpp
struct A {          // 假设 int 4 字节对齐
    char  c;        // offset 0
    int   i;        // offset 4（1~3 为 padding）
};                  // sizeof(A) == 8

#pragma pack(1)     // 取消对齐（嵌入式/协议解析常用）
struct B { char c; int i; };   // sizeof(B) == 5，访问可能低效/报错
#pragma pack()
```

相关关键字：C++11 的 `alignas`（指定对齐）、`alignof`（查询对齐数）。

### 大端与小端

- **大端（Big-Endian）**：高位字节存低地址，网络字节序；
- **小端（Little-Endian）**：低位字节存低地址，x86/ARM 主流；

```cpp
// 判断本机字节序
int x = 1;
bool little = *reinterpret_cast<char*>(&x) == 1;
```

### 头文件重复包含

- 一个头文件在同一翻译单元中被包含两次会导致重复声明/定义错误；
- 解决方式：**include guard**（标准写法）或 `#pragma once`（非标准但主流编译器支持、书写简单）；

```cpp
// foo.h
#ifndef FOO_H
#define FOO_H
// ...
#endif

// 或
#pragma once
```

### 构造函数初始化列表

1. **必须用初始化列表**的场景：`const` 成员、引用成员、没有默认构造函数的成员对象、基类；
2. 使用初始化列表**直接构造**，避免"先默认构造再赋值"的开销；
3. 初始化顺序是**基类先于成员、成员按声明顺序**，与初始化列表书写顺序无关；
4. C++11 起可用类内默认成员初始化器 `int x = 0;`，在初始化列表未覆盖时生效。

### 程序运行时内存布局

典型 Linux x86-64 进程地址空间（从高到低）：

| 区域 | 存放内容 | 特点 |
|---|---|---|
| 栈（stack） | 局部变量、函数调用帧 | 向下增长、自动分配回收、容量有限 |
| 堆（heap） | new/malloc 动态内存 | 向上增长、手动管理、容量大 |
| BSS 段 | 未初始化全局/静态变量 | 启动时清零 |
| 数据段（data） | 已初始化全局/静态变量 | 可读写 |
| 只读数据（rodata） | 字符串字面量、const 常量 | 只读，写会崩溃 |
| 代码段（text） | 编译后的机器指令 | 只读可执行 |

### 栈和堆的区别（高频）

- **栈**：系统自动分配/释放，速度快（寄存器移动栈指针），容量小（Linux 通常 8MB，Windows 约 1MB），递归过深会栈溢出；
- **堆**：`new/malloc` 分配，需手动释放，分配慢（可能陷入内核），容量取决于系统内存，可能产生内存碎片；
- 栈上对象离开作用域自动调用析构；堆对象忘记 delete 就泄漏；
- 频繁分配小对象时可用内存池/对象池优化。

### main 函数之前和之后执行了什么

**进入 main 前**（由 CRT 完成）：

1. 初始化运行时（标准 IO、堆、异常表、线程局部存储等）；
2. 初始化全局/静态对象：依次调用构造函数；
3. 注册 `atexit` 等退出回调。

**main 返回后**：

1. 调用 `atexit` 注册的函数；
2. 按构造逆序析构全局/静态对象；
3. CRT 清理。

> 全局对象的构造顺序跨翻译单元**未定义**，不要依赖；用"局部静态对象（Meyers Singleton）"控制初始化时机。

### assert 与 static_assert

- `assert(expr)`：运行期断言，`NDEBUG` 定义后不生效（通常 release 版本禁用）；失败会中止程序；
- `static_assert(cond, msg)`：**编译期**断言，条件必须是常量表达式，用于约束类型、宽度等；

```cpp
static_assert(sizeof(void*) == 8, "需要 64 位平台");
static_assert(std::is_integral_v<T>, "T 必须是整数类型");
```

### 命名空间与 using

- 命名空间用于避免命名冲突，可嵌套、可匿名（相当于内部链接）；
- `using namespace std;` 使所有名字可见，头文件中**禁止**这样写（污染全局）；
- 优先用限定名或 `using std::vector;`；
- **ADL（实参相关查找）**：查找普通函数时也会在实参类型所在命名空间中查找，是运算符重载能生效的原因。

### 运算符重载注意事项

- 至少一个操作数是类类型/枚举类型；不能重载 `::`、`.`、`.*`、`?:`、`sizeof`、`typeid`；
- 不能创造新运算符；优先级和结合性不可改变；
- 复制赋值 `operator=` 应返回 `T&`（支持链式赋值）并处理自赋值；
- `++`/`--` 前缀与后缀通过参数区分（后缀带一个 int 占位）；
- 不建议重载 `&&`、`||`、`,`：无法保留内建求值顺序和短路语义；
- `<<`/`>>` 通常定义为友元自由函数，便于流对象在左侧。

## 内存管理

### malloc/free 与 new/delete 的区别（必问）

| 维度 | malloc/free | new/delete |
|---|---|---|
| 语言 | C 标准库函数 | C++ 运算符（内部可调用 operator new） |
| 返回类型 | `void*`，需强转 | 已确定类型的指针 |
| 初始化 | 只分配原始内存，不构造 | 分配 + 调用构造函数 |
| 清理 | 只释放内存，不析构 | 调用析构函数 + 释放 |
| 失败 | 返回 NULL | 默认抛出 `std::bad_alloc`（nothrow 版返回 null） |
| 内存大小 | 手动传入字节数 | 编译器按类型计算 |
| 数组 | 自己算总字节 | `new[]` / `delete[]` |

```cpp
int* a = (int*)malloc(sizeof(int) * 10);   // C
free(a);

int* b = new int(10);        // 分配并初始化为 10
delete b;
int* c = new int[100];       // 数组
delete[] c;
```

> 关键点：**new = operator new + 构造函数，delete = 析构函数 + operator delete**。`new[]` 会用 `delete[]` 配套释放，二者不能混用。

### new 的三种形式

```cpp
int* p1 = new int(5);          // 普通 new：失败抛 std::bad_alloc
int* p2 = new (std::nothrow) int(5);  // nothrow new：失败返回 nullptr
char* buf = new char[sizeof(T)];
T* p3 = new (buf) T(args);     // placement new：在已分配内存上构造
```

- `operator new` 是全局/类内可重载的**内存分配函数**（默认封装 malloc）；
- `new` 表达式会自动调用 `operator new` + 构造函数；
- **placement new** 不分配内存，只负责在指定地址构造对象；析构必须手动调用 `p->~T()`，再由外部释放底层内存；
- placement new 常用于内存池、共享内存、自定义容器。

### 内存泄漏 / 悬垂指针 / 野指针

- **内存泄漏**：`new` 出的对象丢失了指针，无法 `delete`；长期运行内存不断增长；检测：Valgrind、ASan/LSan、代码审查、重载 new/delete 统计；
- **悬垂指针（dangling）**：指向的对象已被释放，访问是未定义行为；
- **野指针**：未初始化/指向无效地址的指针，同样不能解引用；
- 预防：`delete` 后置 `nullptr`（防止 double delete），优先 RAII/智能指针，不用裸指针管理资源。

```cpp
int* p = new int(1);
delete p;
p = nullptr;      // 避免悬垂与二次释放
```

### RAII（C++ 最重要的资源管理思想）

**RAII（Resource Acquisition Is Initialization）**：把资源（内存、文件、锁、socket）的获取放在对象构造中，释放放在析构中；对象离开作用域时析构函数自动执行，**保证异常/提前 return 时资源也不泄漏**。

```cpp
class File {
    FILE* fp_;
public:
    explicit File(const char* name) : fp_(fopen(name, "r")) {}
    ~File() { if (fp_) fclose(fp_); }   // 自动关闭
};

std::lock_guard<std::mutex> lock(mtx);   // 离开作用域自动解锁
```

STL 容器、`string`、智能指针、`lock_guard`、`fstream` 都是 RAII 的体现。

### 智能指针（最高频必问）

#### unique_ptr（独占所有权）

- 同一时刻**只有一个 owner**，不可拷贝、可以移动；
- 离开作用域自动 `delete`，零额外开销（和裸指针大小相同）；
- `std::unique_ptr<T[]>` 管理数组（自动用 `delete[]`）；
- 可指定删除器（deleter），类型会作为模板参数影响对象大小。

```cpp
std::unique_ptr<int> up = std::make_unique<int>(42);
// std::unique_ptr<int> up2 = up;   // 编译错误：不可拷贝
auto up2 = std::move(up);           // 所有权转移
```

#### shared_ptr（共享所有权）

- 多个 `shared_ptr` 共享对象，内部**引用计数**：复制 +1，析构 -1，减到 0 才 delete；
- 引用计数本身是**原子操作、线程安全**，但多个线程同时读写**所指对象**仍需外部同步；
- `std::make_shared<T>` 把对象和引用计数控制块放在**一次分配**里，更快、缓存友好、异常安全；

```cpp
auto s1 = std::make_shared<Foo>();
{
    auto s2 = s1;          // 引用计数 2
}                          // s2 析构，计数 1
// s1 析构后计数 0，自动 delete Foo
```

#### weak_ptr（弱引用，破环）

- 不增加引用计数，不拥有对象；
- `lock()` 返回一个有效的 `shared_ptr`（对象已被释放则返回空），避免悬垂访问；
- 专门解决**循环引用**：两个对象互相持有 shared_ptr 时计数永远不为 0，内存泄漏。

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> parent;   // 指向父节点不增加计数
};

if (auto sp = wp.lock()) {        // 临时提升为 shared_ptr 再使用
    // 安全访问
}
```

**循环引用示例**：

```cpp
struct A { std::shared_ptr<B> b; };
struct B { std::shared_ptr<A> a; };
// a->b 与 b->a 互相持有，析构时计数无法归零 -> 泄漏
// 解决：把其中一个改为 weak_ptr
```

#### enable_shared_from_this

- 对象由 `shared_ptr` 管理时，若成员函数想把 `this` 转成 `shared_ptr`，直接用裸 `this` 会造出第二份所有权（double free）；
- 继承 `std::enable_shared_from_this<T>`，用 `shared_from_this()` 获取：

```cpp
class Task : public std::enable_shared_from_this<Task> {
public:
    void schedule() {
        auto self = shared_from_this();   // 复用已有控制块
    }
};
auto t = std::make_shared<Task>();
```

#### 使用原则

- 裸 `new` 出来的指针交给 shared_ptr 管理时，**不能**再交给第二个 shared_ptr（否则 double free）；
- 优先 `make_unique`/`make_shared`；
- 所有权语义：独占用 `unique_ptr`，共享用 `shared_ptr` + 必要处 `weak_ptr`，不要把引用计数当作并发同步方案。

### 手写简化版 shared_ptr（面试常见）

```cpp
template <typename T>
class SharedPtr {
    T* ptr_ = nullptr;
    std::atomic<long>* count_ = nullptr;
public:
    SharedPtr(T* p) : ptr_(p), count_(new std::atomic<long>(1)) {}

    SharedPtr(const SharedPtr& o) noexcept
        : ptr_(o.ptr_), count_(o.count_) {
        if (count_) ++(*count_);
    }

    SharedPtr& operator=(const SharedPtr& o) noexcept {
        if (this != &o) {
            release();                    // 先释放旧资源
            ptr_ = o.ptr_; count_ = o.count_;
            if (count_) ++(*count_);
        }
        return *this;
    }

    ~SharedPtr() { release(); }
    T* get() const { return ptr_; }
    T* operator->() const { return ptr_; }
    T& operator*() const { return *ptr_; }

private:
    void release() {
        if (count_ && --(*count_) == 0) {
            delete ptr_;
            delete count_;
        }
    }
};
```

### 内存碎片与内存池

- **外部碎片**：频繁分配/释放大小不一的块，空闲内存被切成不连续小块，导致"总量够但分配不出大块"；
- **内部碎片**：对齐等造成的块内浪费；
- 解决思路：对象池/内存池（预分配一大块，切成固定大小节点）、分代分配、减少频繁小对象；
- 网络服务器中常用对象池避免每连接 new/delete 的开销与碎片。

## 异常与异常安全

### C++ 异常机制

- `try`/`catch`/`throw` 配合使用；异常对象沿调用栈逐层传播，匹配到第一个能处理的 `catch`；
- 派生类异常要先捕获派生类，再捕获基类（`catch (Derived&)` 放前面），否则派生异常会被基类 catch 提前接住；
- 可捕获任意类型，实践中通常继承 `std::exception`（`std::runtime_error`、`std::out_of_range` 等）；
- 通过 `catch (...)` 捕获一切异常后应重新 `throw;`，否则吞掉未知异常可能掩盖问题。

```cpp
try {
    throw std::runtime_error("boom");
} catch (const std::runtime_error& e) {
    std::cerr << e.what();
} catch (...) {
    // 兜底
}
```

### 异常安全的三个级别

1. **基本保证**：抛异常后对象处于合法但不确定状态，资源不泄漏；
2. **强保证**：操作要么完全成功，要么状态完全不变（事务式，常用 copy-and-swap 实现）；
3. **不抛异常保证**：承诺不抛（如析构、swap），用 `noexcept` 声明。

### noexcept 的作用

- 告诉编译器该函数不会抛异常，若违反会调用 `std::terminate`，也让编译器去掉异常传播开销、便于优化；
- **析构函数默认 noexcept**，绝不要在析构函数里抛出异常：析构发生在栈展开期间时，两个异常叠加会直接 `std::terminate`；
- 移动构造函数/移动赋值通常声明 `noexcept`，否则 `vector` 扩容时为保证强异常安全会退回拷贝；
- `noexcept(expr)` 可基于条件判断，常用于 `is_nothrow_move_constructible` 等 trait。

### 构造函数抛异常时发生了什么

- 已经构造完成的**成员对象和基类子对象会自动析构**，但构造函数体未完成，**对象自身的析构函数不会调用**；
- 因此成员尽量用 RAII 对象管理资源，不要用裸 `new` 在构造函数里手动管理（构造中途抛异常会泄漏已分配但还没交给 RAII 成员的裸资源）。

### 异常的性能与使用建议

- 异常路径的开销比普通返回大，但**成功路径**在现代编译器上开销很小（零成本异常模型）；不要用异常做普通流程控制（如循环内频繁抛）；
- 资源管理用 RAII 保证"抛出即清理"，避免到处写 try/catch；
- 接口设计：预期可控错误可用 `std::optional`/返回码，致命错误或构造失败等场景才抛异常；
- C++23 的 `std::expected<T, E>` 提供"返回值或错误"的类型安全方案，可减少部分异常场景。

## 面向对象

### 封装、继承、多态

- **封装**：把数据和行为捆绑，用 `private/protected/public` 控制访问，隐藏实现细节；
- **继承**：复用和扩展，"is-a" 关系；派生类拥有基类成员并可在其基础上扩展；
- **多态**：同一接口、不同实现：
  - 编译期多态：函数重载、运算符重载、模板；
  - 运行期多态：**虚函数 + 继承**，通过基类指针/引用调用派生类实现。

```cpp
class Shape {
public:
    virtual double area() const = 0;   // 运行期多态的入口
    virtual ~Shape() = default;
};

class Circle : public Shape { /* ... */ };
void print(const Shape& s) { std::cout << s.area(); } // 传 Circle 也正确
```

### 构造函数：默认 / 拷贝 / 移动 / 委托

- **默认构造**：不提供或提供全部默认参数；
- **拷贝构造** `T(const T&)`：用已有对象构造副本（深拷贝问题）；
- **移动构造** `T(T&&) noexcept`：转移资源，不抛异常则尽量声明 `noexcept`（否则 `vector` 扩容可能退化成拷贝）；
- **委托构造**（C++11）：一个构造函数调用同类的另一个构造函数，避免重复初始化代码；
- 有自定义构造后，编译器**不再隐式生成默认构造**。

```cpp
class A {
public:
    A() : A(0) {}          // 委托给下面的构造函数
    explicit A(int v) : val_(v) {}
private:
    int val_;
};
```

### 继承中的构造与析构顺序（必背）

**构造顺序**：基类 → 成员对象（按声明顺序）→ 派生类构造函数体。

**析构顺序**：与构造严格相反：派生类析构函数体 → 成员对象析构 → 基类析构。

```cpp
class Base  { public: ~Base()  { /* 1. 最后执行 */ } };
class Child : public Base {
    Member m_;                          // 成员先于基类之后构造
public:
    Child() : Base(), m_() {}           // Base 先于 m_
    ~Child() { /* 2. 最先执行 */ }      // Child 析构 → m_ → Base
};
```

### 虚函数表机制（最高频必问）

**工作原理**：

1. 类中只要有一个虚函数，对象内存中通常多一个 **vptr（虚表指针）**，指向该类共享的 **vtable（虚函数表）**；
2. vtable 本质是**函数指针数组**，按声明顺序存放该类各虚函数的地址；
3. 通过基类指针调用虚函数时，编译器生成"读 vptr → 查 vtable → 间接调用"的代码，实现**动态绑定**；
4. 每个对象有自己的 vptr，同一类的所有对象共享同一张 vtable；继承后派生类会**覆盖/追加**虚函数项。

**开销**：

- 每个多态对象多一个 vptr（指针大小）；
- 每次虚调用多一次间接寻址，难以内联；
- 但这与类中虚函数数量无关。

```cpp
class A { public: virtual void f() {} };   // sizeof(A) == 8（64 位，含 vptr）
class B : public A { public: void f() override {} };  // 仍是 8
```

### 构造函数/析构函数中调用虚函数

**不会发生多态**，调用的是当前正在构造/析构的类自己的版本。

原因：构造基类部分时，对象的 vptr 指向**基类**的虚表（此时派生类还没构造）；析构派生类部分后 vptr 也已切回基类虚表。若允许调用未构造完成的派生类方法，将访问到未初始化成员，所以语言禁止这种分发。

```cpp
class Base {
public:
    Base() { f(); }            // 调用 Base::f，不是 Derived::f
    virtual void f() { std::cout << "Base\n"; }
};
class Derived : public Base {
public:
    void f() override { std::cout << "Derived\n"; }
};
// new Derived() 输出 Base
```

### 为什么析构函数要声明为 virtual（必问）

- 通过**基类指针/引用 delete 派生类对象**时，若基类析构函数不是虚函数，析构调用走静态绑定，只调用基类析构，派生类成员（尤其资源）不会释放；
- 标准规定此时行为是**未定义**（实际表现为泄漏/崩溃）；
- 所以**设计为多态基类的类必须有虚析构函数**；
- 不打算做基类的类不需要虚析构（白白多占一个 vptr）。

```cpp
class Base { public: virtual ~Base() = default; };
class Derived : public Base { std::unique_ptr<int> p; };

Base* b = new Derived;
delete b;    // virtual 析构：先 Derived 后 Base，正确释放
```

### 纯虚函数 / 抽象类 / 接口

- `virtual void f() = 0;` 是**纯虚函数**，类成为**抽象类**，不能实例化；
- 派生类必须实现所有纯虚函数才能实例化；
- 纯虚函数也可以有实现（通过 `Base::f()` 显式调用），但纯虚的目的是**规定接口**；
- C++ 没有 interface 关键字，"接口类"通常约定为：全部成员函数为纯虚 + 虚析构。

### 重载（Overload）、重写（Override）、隐藏（Hide）的区别

| 概念 | 条件 | 效果 |
|---|---|---|
| 重载 | 同一作用域、同名、参数不同 | 编译器按实参选择版本 |
| 重写 | 派生类重新实现基类 **virtual** 函数，签名一致（C++11 建议加 `override`） | 动态绑定，虚调用进入派生版本 |
| 隐藏 | 派生类定义同名函数（无论是否 virtual、参数是否一致），遮蔽基类同名函数 | 通过派生类对象直接调用会选派生版本，可用 `using Base::f` 或 `Base::f()` 显式调用 |

```cpp
void f(int);                    // A 与 B 重载
void f(double);

class Base    { public: virtual void run() {} };
class Derived : public Base {
public:
    void run() override {}      // 重写
    void foo() {}               // 隐藏基类所有 foo（若有）
};
```

### 菱形继承与虚继承

**菱形继承**：`A ← B/C ← D`，D 通过两条路径继承 A，A 被复制两份，产生二义性和空间浪费。

**虚继承**：用 `virtual` 继承让 A 只保留**一份**共享子对象：

```cpp
class A { public: int x; };
class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};   // D 中只有一个 A
```

- 虚继承对象的布局更复杂：编译器会插入 vbptr（虚基类指针）/偏移量，访问稍慢；
- 虚基类由**最派生类**负责构造，初始化列表从 B/C 一路向上传递；
- 面试结论：菱形继承要小心；能用**组合**就别用多重继承。

### 对象切片（Object Slicing）

用**值传递/值赋值**把派生类对象传给基类时，派生类部分被切掉，只保留基类子对象；虚函数调用也退化为基类版本（因为已不是派生类对象）。

```cpp
void f(Base b) {}          // 传 Derived 会被切片
void g(const Base& b) {}   // 引用不会切片，保留多态
f(Derived{});              // 拷贝构造 Base 部分，派生数据丢失
```

> 多态必须用指针或引用传递。

### final、override、=default、=delete

- `override`：显式声明重写，编译器校验签名，防止"想重写却写错签名变成隐藏"；
- `final`：修饰虚函数（禁止继续重写）或类（禁止继承）；
- `= default`：显式要求生成默认的特殊成员函数（可让声明了其他构造的类仍能默认构造）；
- `= delete`：删除某个函数，调用即编译错误，常用于禁止拷贝：

```cpp
class NonCopyable {
public:
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};
```

### 友元（friend）

- 类可以声明其他类或函数为友元，**授权其访问 private/protected 成员**；
- 破坏了封装，尽量少用；运算符重载（`<<`、`>>`）等少数场景合理；
- 友元关系**不可传递、不可继承**。

### 组合优先于继承

- 继承表达"is-a"，组合表达"has-a"；
- 继承耦合强：基类改动影响所有派生类，暴露基类接口；
- 组合通过成员对象委托，可替换性强、边界清晰；接口需要多态时再考虑继承 + 虚函数。

## STL 容器与算法

### STL 六大组件

**容器（Container）+ 算法（Algorithm）+ 迭代器（Iterator）+ 仿函数（Function Object）+ 适配器（Adapter）+ 空间配置器（Allocator）**。

迭代器是容器与算法的桥梁：算法通过迭代器访问元素，不关心具体容器；仿函数让算法可定制（如 `std::greater<int>` 用于降序）。

### vector 扩容机制（必问）

- `vector` 底层是**连续内存数组**，维护 size 与 capacity；
- 空间不足时分配一块**更大的内存**，把旧元素**移动/拷贝**过去，再释放旧内存，旧迭代器/引用/指针全部失效；
- 扩容系数由实现决定（常见 1.5~2 倍），目的是让均摊插入复杂度保持 **O(1)**；
- `reserve(n)` 预留容量只改 capacity 不创建元素；`resize(n)` 改变 size（多出的部分构造默认/指定值）；
- 扩容拷贝时若类型可移动且移动构造为 `noexcept`，会走移动；否则可能退化为拷贝；

```cpp
std::vector<int> v;
v.reserve(1000);                 // 预先分配，避免多次扩容
std::cout << v.capacity() << '\n';
```

### 迭代器失效问题（高频）

| 容器 | 插入导致的失效 | 删除导致的失效 |
|---|---|---|
| vector | 扩容后全部失效；中间插入后，该位置之后的迭代器失效 | 删除点及之后失效 |
| deque | 两端插入：迭代器失效，引用/指针不失效；中间插入：迭代器和引用全部失效 | 两端删除：仅被删元素及 end() 失效；中间删除：迭代器/引用基本全部失效 |
| list / forward_list | 其他迭代器不受影响 | 只有被删元素的迭代器失效 |
| set/map/multiset/multimap | 其他迭代器不受影响 | 只有被删元素的迭代器失效（红黑树节点独立） |
| unordered_* | 发生 rehash 时迭代器全部失效（指向元素的引用/指针仍有效） | 只有被删元素的迭代器失效 |

> 遍历删除容器元素：vector 用 erase-remove；list/map 直接用 `erase(it++)` 或接收 erase 的返回值。

```cpp
// 从 vector 中删除所有等于 val 的元素（erase-remove 惯用法）
v.erase(std::remove(v.begin(), v.end(), val), v.end());
```

### vector、deque、list 对比（必问）

| 维度 | vector | deque | list |
|---|---|---|---|
| 底层 | 连续数组 | 分段连续缓冲区 | 双向链表 |
| 随机访问 | O(1) | O(1)（慢于 vector） | O(n) |
| 头插 | O(n) | O(1) | O(1) |
| 尾插 | 均摊 O(1)，可能扩容 | O(1) | O(1) |
| 中间插入 | O(n) 搬移 | O(n) | O(1)（需先找到位置 O(n)） |
| 缓存友好 | 最友好 | 较好 | 差（节点分散） |
| 迭代器 | 随机访问 | 随机访问 | 双向 |

### map 与 unordered_map 对比（高频）

| 维度 | map（红黑树） | unordered_map（哈希表） |
|---|---|---|
| 底层 | 红黑树，键有序 | 哈希桶 + 冲突链 |
| 查找/插入复杂度 | O(log n) | 均摊 O(1)，最坏 O(n) |
| 顺序 | 按键升序迭代 | 无序 |
| 键要求 | 可比较（`<`，严格弱序） | 可哈希 + 相等比较 |
| 内存 | 每个节点有左右孩子/颜色 | 桶数组 + 节点 + 哈希 |
| 适用 | 需要有序、范围查询 | 只需等值查找、追求速度 |

```cpp
std::map<std::string, int> m;
m["a"] = 1;    // operator[]：键不存在时插入默认值

std::unordered_map<int, int> h;
h.reserve(10000);              // 提前开桶，减少 rehash
h.load_factor();               // 负载因子，超过 max_load_factor 自动 rehash
```

**operator[] 与 at/find 的区别**：`operator[]` 不存在时**插入默认值**；`at()` 不存在抛 `std::out_of_range`；`find()` 返回迭代器，`end()` 表示未找到。

### 为什么 map 用红黑树不用 AVL

- 两者都保证 O(log n) 的树高；
- 红黑树**近似平衡**（最长路径不超过最短的 2 倍），插入删除时**旋转次数更少**（最多 2~3 次旋转），整体写入性能更好；
- AVL 更严格平衡，查找略快，但插入/删除的频繁旋转开销更大；
- 通用场景红黑树更均衡，因此成为 map/set 的标准实现。

### 哈希冲突的解决方式

1. **链地址法**：每个桶挂链表（`unordered_map` 典型做法），冲突元素插到同桶链表；
2. **开放定址法**：冲突后按探测序列找下一个空位（线性探测/二次探测/双重散列）；
3. 再哈希法、公共溢出区等。

当负载因子超过阈值时**重新哈希（rehash）**：扩大桶数、重算所有元素位置，开销大但均摊可控；对自定义 key 需提供 `std::hash` 和 `operator==`，否则无法放入 unordered 容器。

### std::string 的实现与要点

- 底层是连续字符缓冲区，提供 `size()`/`capacity()`/`reserve()` 等；
- 现代实现常用 **SSO（Small String Optimization，小字符串优化）**：短字符串直接存在对象内部，避免堆分配（libstdc++ 约 15 字节、MSVC 约 15/16 字符）；
- C++17 `std::string_view` 是**非拥有的只读视图**，避免拷贝，但**必须保证源字符串活得比 view 长**；
- `operator[]` 不检查越界，`at()` 越界抛 `std::out_of_range`；
- 拼接可用 `operator+`/`append`，高频拼接预先 `reserve`。

### emplace_back 与 push_back 的区别

- `push_back(x)`：接收**已有对象**（或隐式构造临时对象），然后拷贝/移动进容器；
- `emplace_back(args...)`：把参数**直接转发给构造函数**在容器内存里原地构造，少一次移动；

```cpp
v.push_back(std::string(100, 'x'));   // 先建临时再移动
v.emplace_back(100, 'x');             // 直接在容器内构造
```

> 传已有对象时两者差别不大；关键收益在“参数直接构造”场景。

### 常见容器/算法题

#### stack、queue、priority_queue

- `stack`：默认 deque 适配；`queue`：默认 deque 适配；`priority_queue`：默认 vector + `less` 适配，**默认是大顶堆**，取堆顶 O(1)，插入删除 O(log n)；
- 小顶堆：`priority_queue<int, vector<int>, greater<int>>`。

#### std::sort 的实现与稳定性

- `std::sort` 通常为 **introsort**：快排 + 堆排兜底 + 小区间插入排序，平均/最坏 O(n log n)，**不稳定**；
- 需要稳定排序用 `std::stable_sort`（归并，O(n log n)）；
- `std::nth_element` 找第 k 小元素，均摊 O(n)。

#### vector<bool> 的特殊性

- 为省内存把每个 bool 压成 **1 位**存储，返回的不是 `bool&` 而是代理对象；
- 因此**不能拿 vector<bool> 元素的地址**，也不满足标准容器的引用语义；需要真容器可用 `vector<char>`/`deque<bool>`/bitset。

### 容器选择思路

1. 需要随机访问和缓存友好 → vector；
2. 大量头尾操作 → deque；
3. 频繁中间插入删除、迭代器稳定 → list；
4. 键有序/范围查询 → map/set；
5. 只做等值快速查找 → unordered_map/set；
6. 固定大小位集 → bitset；
7. 只读字符串 → string_view。

## 模板与泛型编程

### 模板的原理与实例化

- 模板是**编译期代码生成蓝图**：用具体类型实例化时编译器生成对应版本的代码；
- 函数模板/类模板的定义通常要写在头文件中（模板在编译期需要完整定义才能实例化）；
- **隐式实例化**：用到哪个类型实例化哪个；同一类型在一个程序里只实例化一份；
- **代码膨胀（code bloat）**：大量不同类型实例化会增加二进制体积；公共逻辑可下沉到非模板函数；
- C++ 模板有**两阶段查找**：定义时不依赖模板参数的普通名字立即绑定；依赖模板参数的名字在实例化时查找（ADL 等）。

### 模板特化与偏特化

- **全特化**：指定所有模板参数的具体类型；
- **偏特化**：只指定部分参数或给出更具体模式（类模板可以，函数模板**不支持偏特化**，可用重载/if constexpr 替代）；

```cpp
template <typename T> struct IsVoid { static constexpr bool value = false; };
template <> struct IsVoid<void> { static constexpr bool value = true; };  // 全特化
template <typename T> struct IsVoid<T*> { static constexpr bool value = false; }; // 偏特化
```

### 可变参数模板（Variadic Template）

- 用 `typename... Args` 接收任意数量、任意类型参数；
- 通过**递归展开**或 **C++17 折叠表达式（fold expression）**处理；

```cpp
template <typename... Args>
auto sum(Args... args) {          // C++17 折叠
    return (... + args);          // 一元左折叠
}

template <typename T>
void print(const T& t) { std::cout << t; }

void print() {}                                 // 递归终止的基例

template <typename T, typename... Rest>
void print(const T& t, Rest... rest) {   // 递归：每次取一个
    std::cout << t << ' ';
    print(rest...);
}
```

### SFINAE 与 enable_if

- **SFINAE**：Substitution Failure Is Not An Error——模板替换失败不算错误，编译器继续找其他可行重载；
- `std::enable_if` 在替换失败时“删除”该候选，从而实现**按类型约束重载**；

```cpp
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T> f(T x) {
    return x * 2;                 // 仅整数类型可用
}
```

### if constexpr（C++17）

- 在**编译期**按常量条件选择分支，未选中的分支**不参与实例化**，因此可以写"不同分支使用不同类型才合法的代码"；

```cpp
template <typename T>
void process(T&& v) {
    if constexpr (std::is_same_v<std::decay_t<T>, int>) {
        // 只有 T 是 int 时这里才被实例化
    } else {
        // ...
    }
}
```

### Concepts（C++20）

- 用命名约束描述模板对类型的**要求**（可读、报错清晰、支持重载）；

```cpp
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template <Arithmetic T>
T square(T x) { return x * x; }
```

### CRTP（奇异递归模板模式）

- 派生类把自己作为基类模板参数，基类用 `static_cast<Derived*>(this)` 调用派生方法，实现**静态多态**，无虚函数开销：

```cpp
template <typename Derived>
struct Base {
    void interface() {
        static_cast<Derived*>(this)->impl();   // 编译期绑定
    }
};
struct Foo : Base<Foo> {
    void impl() { /* ... */ }
};
```

### type_traits 与 static_assert

- `<type_traits>` 提供编译期类型判断：`is_integral`、`is_same`、`is_pointer`、`remove_reference`、`decay` 等；
- 与 `static_assert` 配合在编译期拦截非法类型；
- 别名 `_v`/`_t` 是 C++14/17 起的简洁写法（`std::is_integral_v<T>`、`std::enable_if_t<...>`）。

### 模板与宏的区别

- 模板有类型检查、遵循作用域、可递归、可推导、可用于类；
- 宏只是文本替换，无法检查类型和访问权限；
- C++20 引入 **concepts + requires** 后，很多过去依赖宏/SFINAE 的元编程可读性大幅提升。

### 最令人头痛的解析（Most Vexing Parse）

`T t();` 会被解析为**函数声明**而非构造对象：

```cpp
int f();                       // 函数声明
std::string s();               // 其实是函数声明！不是默认构造
std::string s{};               // C++11 花括号初始化才是对象
```

所以 C++11 提倡用**花括号初始化 `{}`**。

## 现代 C++（11/14/17/20）

### C++11 核心新特性速览

| 特性 | 说明 |
|---|---|
| `auto` / `decltype` | 类型推导 |
| `nullptr` | 类型安全的空指针常量 |
| 右值引用 / 移动语义 | 减少拷贝 |
| 智能指针 | unique_ptr / shared_ptr / weak_ptr |
| Lambda 表达式 | 匿名函数对象 |
| `constexpr` | 编译期求值 |
| `enum class` | 强类型枚举 |
| `override` / `final` / `=default` / `=delete` | 更安全的类控制 |
| 范围 for | `for (auto& x : c)` |
| 可变参数模板 | 任意参数 |
| 线程库 `std::thread` | 语言级多线程 |
| `<atomic>` | 原子操作 |
| `tuple` / `unordered_*` 容器 | 扩展标准库 |

### auto 与 decltype 的区别

- `auto` 推导时**会丢弃引用和顶层 const**（除非显式写 `auto&`、`const auto`）；
- `decltype(expr)` **原样保留**类型（引用、const），不产生值；
- `decltype(auto)`（C++14）让返回类型按 decltype 规则推导，常用于完美转发包装；
- 注意初始化列表：`auto x = {1, 2};` 一直推成 `std::initializer_list<int>`；C++17 起直接列表初始化 `auto x{1};` 按普通规则推导为 `int`，而 `auto x{1, 2};`（多元素）非法。

```cpp
const int i = 0;
auto a = i;             // int（const 被丢弃）
auto& b = i;            // const int&
decltype(i) c = 0;      // const int

template <typename F, typename... A>
decltype(auto) invoke(F&& f, A&&... a) {
    return std::forward<F>(f)(std::forward<A>(a)...);
}
```

### nullptr 与 NULL、0 的区别

- `NULL` 在 C 中常是 `((void*)0)`，在 C++ 中常是 `0`（整型），传参可能被当成 `int`，造成重载歧义；
- `nullptr` 类型是 `std::nullptr_t`，**只能转成指针类型/成员指针/ bool**，不能隐式转成整型；

```cpp
void f(int);
void f(char*);
f(0);          // 选 f(int)
f(nullptr);    // 选 f(char*)
```

### Lambda 表达式（高频）

**语法**：`[捕获](参数) -> 返回类型 { 函数体 }`，本质是编译器生成的匿名函数对象。

**捕获方式**：

- `[]` 不捕获；
- `[=]` 按值捕获所有使用到的局部变量；
- `[&]` 按引用捕获；
- `[x]` / `[&x]` 分别按值/引用捕获某个变量；
- C++14 起支持**初始化捕获**（广义捕获）：`[y = std::move(x)]`；
- 捕获 `this` 时注意对象生命周期，按引用捕获局部变量也同理，防止悬垂；

```cpp
int base = 10;
auto add = [base](int x) { return base + x; };         // 按值捕获
auto dec = [&base](int x) { return (base -= x); };     // 按引用捕获
auto f = [x = std::make_unique<int>(1)] { return *x; };// 移动捕获
```

`mutable` 允许按值捕获的副本在函数体内被修改；泛型 lambda（C++14）参数可写 `auto`。

### constexpr、consteval、constinit

- `constexpr` 变量：编译期常量；`constexpr` 函数：C++14 后可含循环/局部变量，参数是常量时编译期求值；
- `consteval`（C++20）：强制编译期求值；
- `constinit`（C++20）：保证变量有静态初始化（不要求常量用于运行期使用）；
- `const` 只保证"不可修改"，不保证编译期常量；数组大小、模板参数需要常量表达式时用 `constexpr`。

```cpp
constexpr int factorial(int n) {     // C++14 可写循环
    int r = 1;
    for (int i = 2; i <= n; ++i) r *= i;
    return r;
}
constexpr int f5 = factorial(5);     // 编译期算出 120
```

### enum class（强类型枚举）

- 旧 `enum` 的枚举名会泄漏到外层作用域，且能隐式转成 int；
- C++11 `enum class` 是**作用域内**、**不隐式转换**的强类型枚举，可指定底层类型：

```cpp
enum class Color : uint8_t { Red, Green, Blue };
Color c = Color::Red;
// int x = c;              // 编译错误，需 static_cast<int>(c)
```

### std::optional / std::variant / std::any（C++17）

- `optional<T>`：表示"可能有值也可能没有"，替代哨兵值（-1、nullptr、空 string）；
- `variant<T1, T2, ...>`：类型安全联合体，同一时刻只存一种类型，`std::get`/`std::visit` 访问；
- `any`：可存任意类型（内部类型擦除，有堆分配开销），尽量少用；

```cpp
std::optional<int> parse(const std::string& s);
if (auto v = parse("42")) { int n = *v; }   // 有值
else { /* 没有值 */ }

std::variant<int, double> v = 3.14;
double d = std::get<double>(v);             // 错误类型会抛 bad_variant_access
```

### std::string_view（C++17）

- **非拥有**的字符串视图（指针 + 长度），构造/传参零拷贝、零分配；
- 不能隐式从临时 `string` 长期保存；底层字符被释放后 view 悬垂；
- 适合只读参数、解析、切片（`substr()` 是 O(1)）。

### 结构化绑定（C++17）

把 tuple/pair/结构体成员解构到独立变量：

```cpp
std::map<std::string, int> m;
for (const auto& [key, value] : m) { ... }

auto [it, inserted] = m.emplace("x", 1);
```

### 三路比较运算符 <=>（C++20）

- 太空船运算符一次性定义所有比较关系，返回 `std::strong_ordering` / `partial_ordering` / `weak_ordering`；
- 显式 `= default` 可自动按成员字典序生成；

```cpp
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;   // ==、<、<= 全自动
};
```

### Concepts / Ranges / Coroutines / Modules（C++20 高频概念）

- **Concepts**：模板参数约束（见模板章节），替代大量 SFINAE 写法；
- **Ranges**：管道式组合视图与算法，惰性求值（如 `v | std::views::filter(pred) | std::views::transform(f)`），避免手写循环和中间容器；
- **Coroutines**：**无栈协程（stackless）**，`co_await`/`co_yield`/`co_return` 挂起/恢复由编译器生成状态机实现，适合异步 IO、生成器；
- **Modules**：替代 `#include` 文本包含，解决头文件重复解析、宏污染、编译变慢的问题（`import std;`/`export module`）。

### C++14/17/20 演进速查表

| 标准 | 值得记的补充 |
|---|---|
| C++14 | 泛型 lambda、返回类型推导、`constexpr` 放宽、变量模板 |
| C++17 | `if constexpr`、折叠表达式、结构化绑定、optional/variant/any、string_view、`inline` 变量、filesystem |
| C++20 | concepts、ranges、协程、modules、三路比较、span、jthread、format、consteval |
| C++23 | `std::print`、expected、mdspan、deducing this（显式对象参数）、static operator() 等 |

## 多线程与并发

### std::thread 与 join/detach

- `std::thread` **不可拷贝**，只能移动；每个线程对象必须 `join()` 或 `detach()`，否则析构会 `std::terminate`；
- `join()`：阻塞等待线程结束；`detach()`：与线程分离，线程继续后台运行；
- detach 的线程访问已释放的局部变量会悬垂，尽量少用，需要可协作停止用 C++20 `std::jthread`（析构自动 join，带 stop_token）；

```cpp
std::thread t([]{ std::cout << "hello\n"; });
t.join();
```

### 互斥锁与 RAII 锁

- `std::mutex`：最基本的互斥量，不可拷贝，推荐配合 RAII 使用；
- `std::lock_guard`：构造加锁、析构解锁，最简；
- `std::unique_lock`：可手动 lock/unlock、可延迟加锁，可配合条件变量；
- `std::scoped_lock`（C++17）：一次锁多个互斥量，避免多个锁时的死锁；
- `std::recursive_mutex`：同一线程可重复加锁（递归），注意释放次数；

```cpp
std::mutex m;
int shared = 0;
{
    std::lock_guard<std::mutex> lock(m);   // 离开作用域自动解锁
    ++shared;
}
```

### 原子操作 std::atomic

- 对基本类型提供**无锁时最优**的原子读写/加减（是否 lock-free 取决于平台）；
- 没有原子操作修饰的共享变量并发读写是**数据竞争（未定义行为）**；
- `atomic` 只是保证单个操作原子，多个原子操作组合仍需要锁或用事务/更细设计；
- `volatile` 不能替代 atomic：它只阻止编译器优化，不保证 CPU/内存序与原子性；

```cpp
std::atomic<int> counter{0};
void add() { counter.fetch_add(1); }   // 原子自增
int x = counter.load();                // 原子读
```

### 内存序（Memory Order）

- 默认 `memory_order_seq_cst`：全局一致顺序，最直观但同步代价最高；
- `memory_order_relaxed`：只保证原子性，不限制重排；
- `memory_order_acquire` / `memory_order_release`：成对使用，release 之前的内存写在 acquire 之后可见（建立 happens-before）；
- `memory_order_acq_rel`：读改写操作同时 acquire+release。

```cpp
std::atomic<bool> ready{false};
int data = 0;

// 线程 1
data = 42;                       // 普通写
ready.store(true, std::memory_order_release);

// 线程 2
while (!ready.load(std::memory_order_acquire)) {}
// 此时可保证 data == 42 可见
```

### 条件变量与生产者-消费者（必考手写）

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;

// 生产者
{
    std::lock_guard<std::mutex> lock(mtx);
    q.push(x);
    cv.notify_one();            // 唤醒一个等待者
}

// 消费者：等待必须用 unique_lock
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, []{ return !q.empty(); });   // 带谓词防虚假唤醒
int x = q.front(); q.pop();
lock.unlock();
```

要点：

1. `wait` 需要 `unique_lock`（可解锁/重锁）；
2. 必须用**谓词循环**判断条件，防止虚假唤醒（spurious wakeup）和丢失唤醒；
3. `notify_one` 唤醒一个，`notify_all` 唤醒所有（多个同类消费者时用 one 更高效）。

### future / promise / async

- `std::async`：异步执行，返回 `future<T>`；
- `std::promise<T>` 在线程内 set_value/set_exception，`future.get()` 获取值或**重新抛出异常**；
- `future` 只能 get 一次；多线程共享用 `shared_future`；

```cpp
std::future<int> fu = std::async(std::launch::async, []{
    return 42;
});
int r = fu.get();      // 阻塞等待结果；若线程抛异常，这里重新抛出
```

### 死锁的产生与避免

**四个必要条件**：互斥、持有并等待、不可剥夺、循环等待。

避免：

1. 固定加锁顺序（所有线程按同一顺序加锁）；
2. 用 `std::scoped_lock` 一次锁多把锁；
3. 尽量缩小临界区、减少锁的持有时间；
4. 用 lock-free/原子替代部分锁。

### 线程池设计思路（高频）

```cpp
class ThreadPool {
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx_);
                        cv_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    try {
                        task();            // 锁外执行，避免任务间互相阻塞
                    } catch (...) {
                        // 任务异常不能杀死 worker 线程（生产环境应记录或经 future 返回）
                    }
                }
            });
        }
    }

    template <typename F>
    void submit(F&& f) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            tasks_.emplace(std::forward<F>(f));
        }
        cv_.notify_one();
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& t : workers_) t.join();
    }
};
```

面试要点：任务队列 + 互斥锁 + 条件变量 + worker 线程；析构时设置停止标志并唤醒全部线程；返回结果可包一层 `packaged_task` 返回 `future`。

### 线程安全单例

**Meyers Singleton（C++11 起局部静态变量初始化线程安全，推荐）**：

```cpp
class Singleton {
public:
    static Singleton& instance() {
        static Singleton inst;   // 首次调用时构造，编译器保证线程安全
        return inst;
    }
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
private:
    Singleton() = default;
};
```

其他方案对比：饿汉式（静态初始化，程序启动即创建）与双检锁（需 `atomic` + acquire/release，容易写错），多数场景优先 Meyers。

### 并发常见问答

- **数据竞争是未定义行为**：用 mutex、atomic、或保证无共享；
- **shared_ptr 的线程安全**：多个线程同时拷贝/析构同一个 shared_ptr 是安全的（控制块计数原子）；但多个线程通过它同时**读写同一个对象**仍需外部同步；
- **避免忙等**：用条件变量/信号量代替 while 轮询；
- **锁与原子**：原子适合简单计数/标志，复杂临界区用锁；不要为了炫技写复杂的 lock-free 结构。

## 编译、链接与工程化

### 编译的四个阶段

1. **预处理（Preprocessing）**：展开 `#include`、宏替换、条件编译（`g++ -E`），生成 .i；
2. **编译（Compilation）**：词法/语法/语义分析、生成汇编（`-S`）；
3. **汇编（Assembly）**：汇编代码转机器码目标文件 .o/.obj（`-c`）；
4. **链接（Linking）**：合并各目标文件与库，解析符号，生成可执行文件。

> 编译单位之间互不可见，所以**头文件里重复包含、宏、类定义可见性**等问题只在编译阶段暴露；未解析符号在链接阶段暴露。

### 为什么模板必须在头文件中定义

- 模板要**实例化**才知道具体代码；不同 .cpp 各自包含模板定义即可各自实例化；
- 若把模板实现放 .cpp，另一个翻译单元只看到声明，无法生成该类型的实例化版本 → 链接错误；
- 特殊场景可用**显式实例化**（`template class vector<int>;`）把实例放到库中。

### 静态库与动态库

| 维度 | 静态库（.a/.lib） | 动态库（.so/.dll） |
|---|---|---|
| 链接时机 | 编译链接期拷贝进可执行文件 | 运行期加载 |
| 文件体积 | 可执行文件大 | 可执行文件小，库独立 |
| 依赖 | 发布无需带库 | 运行时需能找到库 |
| 升级 | 需重新编译整个程序 | 替换库文件即可（需保持 ABI） |
| 启动 | 快（无需加载解析） | 启动时加载/符号解析 |
| 内存 | 多进程各持一份 | 物理内存可共享一份代码 |

### 名字修饰（Name Mangling）与 ODR

- C++ 支持重载，编译器把函数名、参数类型、命名空间、返回类型等编码进符号名，保证重载符号不冲突；
- 所以链接 C 函数要 `extern "C"`（见前文）；
- **ODR（One Definition Rule，单一定义规则）**：整个程序中非 inline 函数/全局变量/类非 inline 成员函数**只能有一个定义**；inline 函数与模板可以跨 TU 有相同定义；
- 头文件里写函数体（非 inline）在多个 .cpp 包含后会导致重复定义链接错误。

### 静态初始化顺序问题

- 全局/static 对象跨翻译单元的构造顺序未定义；
- 用**局部 static**（首次使用时构造）或显式初始化（如工厂注册表）规避；
- 析构顺序与构造相反，跨 TU 同样不要依赖。

### include 与前置声明

- `#include` 会把整个头文件文本复制进来，能前置声明（`class Foo;`）就不要 include，减少编译依赖和编译时间；
- 头文件应"自包含"（包含自己依赖的头），同时尽量少的依赖；
- 循环 include 用前向声明 + 指针成员解决；
- **PImpl 惯用法**：类只持 `unique_ptr<Impl>`，实现全部放 .cpp，头文件不再依赖实现头文件 → 编译快、ABI 稳定。

### CMake 常用要点

```cmake
cmake_minimum_required(VERSION 3.20)
project(demo CXX)

add_executable(app main.cpp)
target_compile_features(app PRIVATE cxx_std_20)
target_compile_options(app PRIVATE -Wall -Wextra)
target_link_libraries(app PRIVATE mylib)

add_library(mylib STATIC mylib.cpp)
target_include_directories(mylib PUBLIC include)   # PUBLIC 让依赖方也拿到头文件路径
```

- 构建流程：`cmake -S . -B build` → `cmake --build build`；
- 老项目也有 Makefile：`make` 按依赖关系增量编译；
- 常见面试问题：头文件变更导致全量重编的优化（降低头文件耦合、forward declare、模块）、`-DNDEBUG` 影响 assert、O2 优化与调试符号冲突。

## C++ 特有设计技巧与模式

### PImpl（Pointer to Implementation）

- 头文件只暴露接口指针，实现细节藏在 .cpp，降低编译依赖、隐藏实现、保持 ABI；
- 注意：持有 `unique_ptr` 的类析构函数必须在 .cpp 中定义（因为 Impl 类型在头文件中不完整）。

### NVI（Non-Virtual Interface）

- 基类 public 接口非虚并负责通用流程，虚函数设为 protected/private 供派生类实现，实现"模板方法"式扩展；
- 好处：公共逻辑（锁、日志、前置后置检查）集中在基类。

### 单例、工厂、观察者（常见手写）

**线程安全单例**见并发章节；工厂与观察者多用 `std::function` 简化：

```cpp
// 简单观察者
class Subject {
    std::vector<std::function<void(int)>> obs_;
public:
    void attach(std::function<void(int)> f) { obs_.push_back(std::move(f)); }
    void notify(int v) const {
        for (auto& f : obs_) f(v);
    }
};
```

### RAII 与作用域守卫

```cpp
// 自定义 scope guard：离开作用域自动执行（替代 goto cleanup）
class ScopeGuard {
    std::function<void()> fn_;
public:
    explicit ScopeGuard(std::function<void()> fn) : fn_(std::move(fn)) {}
    ~ScopeGuard() { if (fn_) fn_(); }
};

auto guard = ScopeGuard([]{ std::cout << "cleanup\n"; });
```

### 移动/拷贝安全：copy-and-swap

用**传值 + swap**统一实现拷贝赋值，天然处理自赋值与异常安全：

```cpp
class Buffer {
    char* data_;
    size_t size_;
    void swap(Buffer& o) noexcept {
        std::swap(data_, o.data_);
        std::swap(size_, o.size_);
    }
public:
    Buffer(const Buffer& o);                 // 深拷贝构造
    Buffer& operator=(Buffer o) noexcept {   // 传值：拷贝 or 移动一次
        swap(o);                             // 异常安全、自赋值安全
        return *this;
    }
};
```

## 调试、性能与线上问题

### 常用调试工具

| 工具 | 用途 |
|---|---|
| gdb / lldb | 断点、调用栈、core 分析 |
| AddressSanitizer（ASan） | 堆越界、use-after-free、内存泄漏（LeakSanitizer） |
| UBSan | 未定义行为检查（溢出、非法转换等） |
| ThreadSanitizer（TSan） | 数据竞争检测 |
| Valgrind | 内存泄漏与非法访问（较慢） |
| perf / gprof | CPU 性能分析、热点函数 |
| strace | 系统调用跟踪 |

### 性能优化原则

- **先测量再优化**，热点集中在少数代码；
- 避免不必要的拷贝：const 引用、移动语义、返回值优化（RVO/NRVO，C++17 起强制部分场景）；
- 容器按使用场景选型，避免频繁扩容；
- 缓存友好：顺序访问、连续内存（vector）；
- 减少锁竞争：缩小临界区、无锁读、线程局部；
- 编译器选项：`-O2/-O3`、`NDEBUG`、`-march`；
- 避免过早抽象和微小优化；优先算法/数据结构层面的改进。

### 崩溃排查思路

1. 看信号/异常（Segmentation fault、core dump）；
2. 用调试器打印调用栈，定位崩溃帧；
3. 常见根因：野指针/悬垂、越界、double free、栈溢出、虚函数调用前对象已析构、容器迭代器失效、并发数据竞争；
4. 开 ASan/UBSan/TSan 复现；
5. 若崩溃不固定，优先怀疑**未定义行为**（数据竞争、use-after-free、整数/移位溢出）。

## 常见手写/编程题汇总

### 1. String 类（深浅拷贝/三或五法则）

```cpp
class String {
    char* data_ = nullptr;
    size_t len_ = 0;

    static char* clone(const char* s, size_t n) {
        char* p = new char[n + 1];
        if (n) std::memcpy(p, s, n);
        p[n] = '\0';
        return p;
    }

public:
    String() = default;
    String(const char* s) : len_(std::strlen(s)), data_(clone(s, len_)) {}

    String(const String& o) : len_(o.len_), data_(clone(o.data_, o.len_)) {}

    String& operator=(String o) {            // copy-and-swap
        std::swap(data_, o.data_);
        std::swap(len_, o.len_);
        return *this;
    }

    String(String&& o) noexcept : data_(o.data_), len_(o.len_) {
        o.data_ = nullptr; o.len_ = 0;
    }

    ~String() { delete[] data_; }
    const char* c_str() const { return data_ ? data_ : ""; }
    size_t size() const { return len_; }
};
```

### 2. 线程安全单例

见并发章节 Meyers Singleton（局部静态对象），问实现时先答这个，再说 `call_once`/锁方案。

### 3. 生产者-消费者

见并发章节条件变量示例；进阶变体：多生产者多消费者（notify_all + 每个消费者独占队列）、有界队列（满则等待）。

### 4. 实现 std::function 的思想 / std::bind 与 lambda

`std::function` 用**类型擦除**：内部基类指针 + 模板派生类包装任意可调用对象；问原理时说明 `operator()` 通过虚函数/函数指针间接调用即可，不必手写全部。

### 5. LRU 缓存

哈希表（O(1) 定位）+ 双向链表（O(1) 移动/删除）：

```cpp
class LRUCache {
    struct Node { int key, value; };
    std::list<Node> list_;                                   // 最近使用在前
    std::unordered_map<int, std::list<Node>::iterator> map_;
    size_t cap_;

public:
    explicit LRUCache(size_t cap) : cap_(cap) {}

    int get(int key) {
        auto it = map_.find(key);
        if (it == map_.end()) return -1;
        list_.splice(list_.begin(), list_, it->second);      // 移到头部
        return it->second->value;
    }

    void put(int key, int value) {
        auto it = map_.find(key);
        if (it != map_.end()) {
            it->second->value = value;
            list_.splice(list_.begin(), list_, it->second);
            return;
        }
        if (map_.size() == cap_) {                            // 淘汰尾部
            map_.erase(list_.back().key);
            list_.pop_back();
        }
        list_.emplace_front(Node{key, value});
        map_[key] = list_.begin();
    }
};
```

### 6. 手写 memcpy / strcpy / strlen

```cpp
void* my_memcpy(void* dst, const void* src, size_t n) {
    auto* d = static_cast<char*>(dst);
    const auto* s = static_cast<const char*>(src);
    for (size_t i = 0; i < n; ++i) d[i] = s[i];
    return dst;
}
// 注意：memcpy 要求 dst/src 不重叠；可能重叠用 memmove

char* my_strcpy(char* dst, const char* src) {
    char* p = dst;
    while ((*p++ = *src++) != '\0') {}
    return dst;
}

size_t my_strlen(const char* s) {
    size_t n = 0;
    while (*s++) ++n;
    return n;
}
```

### 7. 其他高频手写清单

- 自定义 vector（扩容 + 拷贝/移动）；
- 简化 shared_ptr / unique_ptr（前面已给）；
- 循环队列（数组实现，区分空/满）；
- 线程池（前面已给）；
- 内存池/对象池（free list）；
- 两个线程交替打印 1~N（互斥锁/条件变量或原子）；
- 顶层 const/指针的 const 语义辨析、运算符重载（`++`/`<<`）、深拷贝类；
- C++ 特有的"陷阱题"：虚析构、切片、迭代器失效、vector 扩容、字符串字面量、宏括号、整数溢出。

## 高频速查表

| 问题 | 一句话答案 |
|---|---|
| 引用和指针区别 | 引用是别名必初始化不可改绑；指针存地址可空可重绑 |
| new/delete 与 malloc/free | 前者调构造/析构且失败抛异常，后者只分配内存 |
| 虚析构函数 | 多态基类必须有，否则 delete 派生类不调派生析构（UB） |
| 构造/析构中调虚函数 | 不会多态，只调当前类版本 |
| vector 扩容 | 满时倍增搬移，旧迭代器全部失效 |
| map 与 unordered_map | 红黑树 O(log n) 有序 vs 哈希 O(1) 无序 |
| 智能指针怎么选 | 独有所有权 unique_ptr；共享所有权 shared_ptr；破环用 weak_ptr |
| 为什么需要移动语义 | 转移堆资源所有权，避免大对象深拷贝 |
| static 的作用 | 局部静态只初始化一次；文件静态限内部链接；类静态属于类 |
| const 成员函数 | 承诺不修改对象（mutable 除外），const 对象只能调它 |
| RAII | 资源获取即初始化，析构自动释放，异常安全 |
| 深浅拷贝 | 浅拷贝共享指针成员会 double free；深拷贝复制资源 |
| 线程安全单例 | C++11 局部 static 初始化线程安全，优先 Meyers |
| volatile 能替代 atomic 吗 | 不能，缺少原子性和内存序 |
| 编译链接过程 | 预处理 → 编译 → 汇编 → 链接 |
| 为什么用虚函数表 | 支持运行期多态，vptr 指向 vtable 函数指针数组 |
| C++ 与 C 关系 | C++ 兼容 C 子集，加上类/模板/异常/重载/STL 等 |
