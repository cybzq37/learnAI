## 一、反射的本质

Java 反射（Reflection）是一种在**运行时动态获取类的信息，并操作类及其成员**的机制。

正常情况下，我们在编译期就确定了要操作的类：

```java
User user = new User();

user.setName("Tom");

String name = user.getName();
```

编译器知道：

```text
User
 ├─ setName()
 └─ getName()
```

而反射允许程序在运行时才确定：

```text
操作哪个类？
调用哪个方法？
访问哪个字段？
使用哪个构造器？
```

例如：

```java
Class<?> clazz = Class.forName("com.example.User");

Object obj = clazz.getDeclaredConstructor().newInstance();

Method method = clazz.getDeclaredMethod(
        "setName",
        String.class
);

method.invoke(obj, "Tom");
```

这里在编译 `Main` 类时，并不需要直接写：

```java
new User();
```

而是在运行时通过类名、方法名等信息动态完成操作。

因此可以把反射概括为：

> **反射就是程序在运行时获取类型元数据，并根据这些元数据动态操作对象的机制。**

---

## 二、反射的核心：Class 对象

反射的核心对象是：

```java
java.lang.Class
```

Java 中每一个被 JVM 加载的类，在运行时都会有对应的 `Class` 对象。

例如：

```java
class User {
}
```

运行时可以得到：

```java
Class<?> clazz = User.class;
```

可以抽象理解为：

```text
Java 类
   │
   │ JVM 加载
   ↓
Class 对象
   │
   ├── 类名
   ├── 父类
   ├── 接口
   ├── 修饰符
   ├── 字段
   ├── 方法
   ├── 构造器
   ├── 注解
   └── 其他类型信息
```

因此：

> **`Class` 对象可以理解为 JVM 在运行时对一个 Java 类型的“类型描述对象”。**

反射 API 大部分操作最终都是从 `Class` 对象开始的。

---

## 三、如何获取 Class 对象

常见有三种方式。

### 1. `类名.class`

```java
Class<User> clazz = User.class;
```

这是最直接的方式。

它不需要创建 `User` 对象：

```java
User user = new User();
```

也就是说：

```java
User.class
```

与：

```java
new User()
```

是完全不同的概念。

前者获取的是：

```text
Class 对象
```

后者创建的是：

```text
User 对象
```

---

### 2. `对象.getClass()`

```java
User user = new User();

Class<?> clazz = user.getClass();
```

这种方式需要先有对象。

```text
User 对象
   │
   │ getClass()
   ↓
Class<User>
```

---

### 3. `Class.forName()`

```java
Class<?> clazz =
        Class.forName("com.example.User");
```

这种方式根据**完整类名**在运行时获取 Class 对象。

它常用于：

- 配置文件指定类名
- SPI
- JDBC
- 框架
- 插件机制

例如：

```java
String className =
        "com.example.User";

Class<?> clazz =
        Class.forName(className);
```

这里类名可以来自配置，而不是写死在代码中。

---

## 四、三种获取方式的区别

| 方式 | 是否需要对象 | 类名是否运行时确定 |
|---|---:|---:|
| `User.class` | 否 | 否 |
| `user.getClass()` | 是 | 否 |
| `Class.forName()` | 否 | 是 |

可以简单记忆：

```text
User.class
    → 我知道这个类

user.getClass()
    → 我手里已经有对象

Class.forName()
    → 我运行时才知道类名
```

---

## 五、Class 对象能够获取什么

拿到 `Class` 对象以后，就可以获取类的各种元数据：

```java
Class<?> clazz = User.class;
```

例如：

```java
clazz.getName();
clazz.getSuperclass();
clazz.getInterfaces();
clazz.getModifiers();
clazz.getDeclaredFields();
clazz.getDeclaredMethods();
clazz.getDeclaredConstructors();
clazz.getAnnotations();
```

可以理解为：

```text
Class
 │
 ├── 基本信息
 │    ├── 类名
 │    ├── 包名
 │    ├── 修饰符
 │    └── 父类
 │
 ├── 类型关系
 │    ├── 父类
 │    └── 接口
 │
 ├── 成员
 │    ├── Field
 │    ├── Method
 │    └── Constructor
 │
 └── 元数据
      └── Annotation
```

---

## 六、获取类的基本信息

### 1. 获取类名

```java
clazz.getName();
```

例如：

```java
com.example.User
```

还可以使用：

```java
clazz.getSimpleName();
```

得到：

```text
User
```

以及：

```java
clazz.getPackageName();
```

得到：

```text
com.example
```

---

### 2. 获取父类

```java
Class<?> superClass =
        clazz.getSuperclass();
```

例如：

```java
class User extends Person {
}
```

那么：

```java
User.class.getSuperclass()
```

得到：

```text
Person
```

需要注意：

> 对接口调用 `getSuperclass()` 返回 `null`。

---

### 3. 获取接口

```java
Class<?>[] interfaces =
        clazz.getInterfaces();
```

例如：

```java
class User implements Serializable, Comparable<User> {
}
```

可以获取：

```text
Serializable
Comparable
```

---

### 4. 获取修饰符

```java
int modifiers =
        clazz.getModifiers();
```

然后可以使用：

```java
Modifier.isPublic(modifiers);
Modifier.isFinal(modifiers);
Modifier.isAbstract(modifiers);
```

例如：

```java
if (Modifier.isPublic(clazz.getModifiers())) {
    System.out.println("public class");
}
```

---

## 七、Field：反射操作字段

字段对应：

```java
java.lang.reflect.Field
```

例如：

```java
class User {

    private String name;

    public int age;
}
```

可以获取字段：

```java
Field field =
        User.class.getDeclaredField("name");
```

然后读取：

```java
Object value =
        field.get(user);
```

修改：

```java
field.set(user, "Tom");
```

完整示例：

```java
User user = new User();

Field field =
        User.class.getDeclaredField("name");

field.setAccessible(true);

field.set(user, "Tom");

Object value =
        field.get(user);

System.out.println(value);
```

整个过程：

```text
Class
  │
  │ getDeclaredField()
  ↓
Field
  │
  ├── get()
  │     → 读取字段
  │
  └── set()
        → 修改字段
```

---

## 八、getField 和 getDeclaredField 的区别

这是反射中非常重要的一组 API。

```java
getField()
getDeclaredField()
```

### `getField()`

只能获取：

> **当前类及其父类中可访问的 public 字段。**

### `getDeclaredField()`

获取：

> **当前类自己声明的字段，不区分 public、protected、package-private、private。**

例如：

```java
class User extends Person {

    private String name;

    public int age;
}
```

```java
User.class.getDeclaredField("name");
```

可以获取：

```text
name
```

而：

```java
User.class.getField("name");
```

不能获取，因为 `name` 是 private。

可以简单记忆：

```text
getField()
    → public + 继承体系

getDeclaredField()
    → 当前类声明的全部字段
```

同样的规则也适用于方法和构造器：

```text
getMethod()
getDeclaredMethod()

getConstructor()
getDeclaredConstructor()
```

---

## 九、Method：反射调用方法

方法对应：

```java
java.lang.reflect.Method
```

例如：

```java
class User {

    public String hello(String name) {
        return "Hello " + name;
    }
}
```

获取方法：

```java
Method method =
        User.class.getMethod(
                "hello",
                String.class
        );
```

调用：

```java
User user = new User();

Object result =
        method.invoke(user, "Tom");

System.out.println(result);
```

输出：

```text
Hello Tom
```

这里：

```java
method.invoke(user, "Tom");
```

可以理解为：

```text
Method
  │
  │ invoke()
  ↓
指定对象
  │
  ↓
执行目标方法
  │
  ↓
返回结果
```

---

## 十、Method 如何确定要调用哪个方法

Java 支持方法重载：

```java
void test()
void test(String name)
void test(int age)
```

因此反射获取方法时，需要提供：

```text
方法名
+
参数类型
```

例如：

```java
Method method =
        clazz.getDeclaredMethod(
                "test",
                String.class
        );
```

表示查找：

```java
test(String)
```

而不是：

```java
test()
test(int)
```

注意：

> 反射根据的是**参数类型**定位方法，而不是参数变量名。

---

## 十一、Constructor：反射创建对象

构造器对应：

```java
java.lang.reflect.Constructor
```

例如：

```java
class User {

    private String name;

    public User(String name) {
        this.name = name;
    }
}
```

获取构造器：

```java
Constructor<User> constructor =
        User.class.getConstructor(
                String.class
        );
```

创建对象：

```java
User user =
        constructor.newInstance("Tom");
```

完整过程：

```text
Class
  │
  │ getConstructor()
  ↓
Constructor
  │
  │ newInstance()
  ↓
创建对象
```

现代 Java 中，更推荐：

```java
clazz.getDeclaredConstructor()
     .newInstance();
```

而不是：

```java
clazz.newInstance();
```

后者已经被废弃。

---

## 十二、反射创建对象与普通 new 的区别

普通方式：

```java
User user = new User("Tom");
```

在编译期就确定了：

```text
类
构造器
参数类型
```

反射方式：

```java
Class<?> clazz =
        Class.forName("com.example.User");

Constructor<?> constructor =
        clazz.getDeclaredConstructor(
                String.class
        );

Object user =
        constructor.newInstance("Tom");
```

可以在运行时决定：

```text
类名
构造器
参数类型
参数值
```

因此反射特别适合：

> **框架根据配置或元数据动态创建对象。**

---

## 十三、访问 private 成员

反射不仅可以查看 public 成员，也可以获取当前类声明的 private 成员。

例如：

```java
class User {

    private String name;
}
```

可以：

```java
Field field =
        User.class.getDeclaredField("name");
```

但是直接：

```java
field.get(user);
```

可能因为访问控制而失败。

传统反射代码中可以使用：

```java
field.setAccessible(true);
```

然后：

```java
field.set(user, "Tom");
```

方法和构造器也存在类似机制：

```java
method.setAccessible(true);

constructor.setAccessible(true);
```

---

## 十四、setAccessible(true) 到底做了什么

`setAccessible(true)` 的核心作用是：

> **调整反射对象的可访问性检查行为，使后续反射操作可以在传统 Java 访问控制之外进行。**

它并不是：

```text
把 private 改成 public
```

也不是：

```text
修改 class 文件
```

而是：

```text
反射对象
   │
   │ setAccessible(true)
   ↓
后续反射访问时
减少/跳过传统语言访问检查
```

现代 Java 中还需要注意模块系统（JPMS）的限制。

对于跨模块访问，尤其是 JDK 内部类或未开放的包，即使使用：

```java
setAccessible(true)
```

也可能因为模块边界而抛出：

```text
InaccessibleObjectException
```

因此：

> `setAccessible(true)` 并不意味着可以无条件访问任何对象的任何成员。

---

## 十五、反射与注解的关系

前面讲注解时已经出现过：

```java
AnnotatedElement
```

它正是连接：

```text
反射
 +
注解
```

的关键接口。

例如：

```java
Method method =
        MyService.class.getMethod("execute");

MyAnnotation annotation =
        method.getAnnotation(MyAnnotation.class);
```

这里：

```text
Class / Method / Field
        │
        │ 实现
        ↓
AnnotatedElement
        │
        ├── getAnnotation()
        ├── getAnnotations()
        └── isAnnotationPresent()
```

因此可以把注解和反射的关系理解为：

```text
注解
  ↓
提供元数据

反射
  ↓
读取并操作这些元数据

框架
  ↓
根据元数据执行自己的逻辑
```

例如 Spring 中：

```java
@Service
public class UserService {
}
```

框架可以通过反射：

```java
clazz.isAnnotationPresent(Service.class)
```

判断这个类是否具有 `@Service`。

然后进一步：

```text
扫描类
  ↓
反射获取注解
  ↓
发现 @Service
  ↓
创建对象
  ↓
注册到容器
```

这也是为什么：

> **注解经常和反射一起出现。**

---

## 十六、反射与泛型

反射不仅可以获取普通类型信息，还可以获取部分**泛型元数据**。

例如：

```java
class UserService
        extends BaseService<User> {
}
```

通过：

```java
Type type =
        UserService.class.getGenericSuperclass();
```

得到的不是简单的：

```java
Class
```

而是：

```java
Type
```

Java 反射中的泛型类型使用了 `Type` 体系表示：

```text
Type
 ├── Class
 ├── ParameterizedType
 ├── TypeVariable
 ├── WildcardType
 └── GenericArrayType
```

例如：

```java
List<String>
```

运行时不能简单表示成一个普通的：

```java
Class<List<String>>
```

因为 Java 泛型存在**类型擦除**。

因此：

```java
List<String>.class
```

这样的写法是不存在的。

但反射仍然可以从类、字段、方法的泛型签名元数据中获取部分泛型信息。

例如：

```java
class UserService {

    private List<String> users;
}
```

可以：

```java
Field field =
        UserService.class.getDeclaredField("users");

Type type =
        field.getGenericType();
```

此时可以进一步判断：

```java
if (type instanceof ParameterizedType) {
    ParameterizedType pt =
            (ParameterizedType) type;
}
```

因此：

> **Java 的泛型在运行时存在类型擦除，但 `.class` 文件仍可能保存泛型签名信息，反射可以读取这些元数据。**

---

## 十七、反射与类型擦除

例如：

```java
List<String> list1;
List<Integer> list2;
```

在运行时：

```java
list1.getClass() == list2.getClass()
```

结果为：

```text
true
```

因为运行时它们都是：

```java
java.util.ArrayList
```

而不是：

```text
ArrayList<String>
ArrayList<Integer>
```

因此：

```text
编译期
    ↓
List<String>
List<Integer>

运行时
    ↓
List
```

这就是泛型类型擦除的典型表现。

不过：

```java
class UserService {

    List<String> users;
}
```

字段的泛型签名仍可能保存在 `.class` 文件中，所以：

```java
field.getGenericType()
```

仍然可以获取：

```text
List<String>
```

对应的泛型元数据。

---

## 十八、反射中的几个重要接口和类

可以建立下面的关系：

```text
                    Class
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Field        Method    Constructor
          │           │           │
          ↓           ↓           ↓
       字段信息      方法信息     构造器信息
          │           │           │
          └───────────┼───────────┘
                      ↓
               反射操作程序结构
```

与注解相关：

```text
AnnotatedElement
       │
       ├── Class
       ├── Method
       ├── Field
       ├── Constructor
       └── Parameter
```

与泛型相关：

```text
Type
 ├── Class
 ├── ParameterizedType
 ├── TypeVariable
 ├── WildcardType
 └── GenericArrayType
```

这些共同构成 Java 反射 API 的核心类型体系。

---

## 十九、反射的完整执行过程

以：

```java
Method method =
        clazz.getDeclaredMethod(
                "hello",
                String.class
        );

Object result =
        method.invoke(obj, "Tom");
```

为例，可以把过程抽象为：

```text
获取 Class 对象
    ↓
Class.forName()
    ↓
JVM 加载类
    ↓
Class 对象
    ↓
getDeclaredMethod()
    ↓
Method 对象
    ↓
invoke()
    ↓
检查访问权限
    ↓
确定目标方法
    ↓
处理参数
    ↓
执行目标方法
    ↓
返回结果
```

如果涉及注解：

```text
Class / Method
    ↓
getAnnotation()
    ↓
读取注解元数据
    ↓
返回 Annotation 对象
```

---

## 二十、反射底层原理

反射并不是 JVM 中一套完全独立于普通 Java 类的“魔法机制”。

它建立在：

```text
JVM
 +
Class 元数据
 +
java.lang.reflect
 +
JDK 内部实现
```

之上。

类被 JVM 加载后，JVM 会维护与类相关的运行时信息。

而 Java 反射 API 则提供 Java 层面的访问入口：

```text
.class 文件
     │
     │ JVM 加载
     ↓
运行时类信息
     │
     ↓
Class 对象
     │
     ├── Field
     ├── Method
     ├── Constructor
     └── Annotation
```

因此：

> **Class 是反射的核心入口，Field、Method、Constructor 等对象则是对类成员元数据的 Java 层表示。**

---

## 二十一、反射调用并不等于普通方法调用

普通调用：

```java
user.hello("Tom");
```

编译器在编译期就知道：

```text
目标类
目标方法
参数类型
调用方式
```

反射调用：

```java
method.invoke(user, "Tom");
```

则需要在运行时完成：

```text
获取 Method
    ↓
检查访问权限
    ↓
检查目标对象
    ↓
检查参数
    ↓
确定调用目标
    ↓
执行方法
    ↓
返回结果
```

因此反射调用通常比直接调用有更多运行时开销。

不过现代 JVM 对反射调用进行了大量优化，因此不能简单理解成：

> “反射一定非常慢。”

更准确的说法是：

> **反射通常比直接调用有更高的调用成本；对于一般框架场景通常可以接受，但在极高频、对性能敏感的核心路径中，应谨慎使用。**

---

## 二十二、反射为什么能够支撑各种框架

反射最大的价值不是“能够调用 private 方法”，而是：

> **让程序可以在不知道具体类型实现细节的情况下，根据运行时元数据动态工作。**

例如一个框架可能只知道：

```text
类名
方法名
注解
字段
构造器
参数类型
```

然后通过反射完成：

```text
扫描类
  ↓
获取 Class
  ↓
读取注解
  ↓
分析字段和方法
  ↓
创建对象
  ↓
注入依赖
  ↓
调用方法
```

因此：

```text
反射
  ↓
动态发现 + 动态操作

注解
  ↓
提供元数据

框架
  ↓
利用反射读取元数据
  ↓
实现自动化功能
```

这也是：

```text
Spring
JUnit
MyBatis
Jackson
各种 IoC / ORM / 序列化框架
```

大量使用反射的原因之一。

---

## 二十三、反射的核心 API

可以将最常用的 API 归纳为：

| 操作 | 常用 API |
|---|---|
| 获取 Class | `User.class` |
| 获取 Class | `obj.getClass()` |
| 动态加载类 | `Class.forName()` |
| 获取类名 | `getName()` |
| 获取父类 | `getSuperclass()` |
| 获取接口 | `getInterfaces()` |
| 获取字段 | `getField()` / `getDeclaredField()` |
| 获取方法 | `getMethod()` / `getDeclaredMethod()` |
| 获取构造器 | `getConstructor()` / `getDeclaredConstructor()` |
| 读取字段 | `Field.get()` |
| 修改字段 | `Field.set()` |
| 调用方法 | `Method.invoke()` |
| 创建对象 | `Constructor.newInstance()` |
| 获取注解 | `getAnnotation()` |
| 判断注解 | `isAnnotationPresent()` |
| 获取修饰符 | `getModifiers()` |
| 修改访问检查 | `setAccessible()` |
| 获取泛型信息 | `getGenericType()` 等 |

---

## 二十四、反射中最容易混淆的几组 API

### `getField()` vs `getDeclaredField()`

```text
getField()
    → public 字段
    → 当前类 + 父类

getDeclaredField()
    → 当前类声明的字段
    → 包括 private
```

### `getMethod()` vs `getDeclaredMethod()`

```text
getMethod()
    → public 方法
    → 当前类 + 父类

getDeclaredMethod()
    → 当前类声明的方法
    → 包括 private
```

### `getConstructor()` vs `getDeclaredConstructor()`

```text
getConstructor()
    → public 构造器

getDeclaredConstructor()
    → 当前类声明的构造器
    → 包括 private
```

可以统一记忆：

```text
getXXX()
    → public + 继承体系

getDeclaredXXX()
    → 当前类声明的全部成员
```

---

## 二十五、反射与 JVM 的关系

可以把 JVM 和反射的职责简单划分为：

```text
                 .class 文件
                     │
                     ↓
                    JVM
                     │
          ┌──────────┴──────────┐
          │                     │
      加载类                 运行时类信息
          │                     │
          └──────────┬──────────┘
                     ↓
                Class 对象
                     │
                     ↓
              Java 反射 API
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
     Field         Method      Constructor
       │             │             │
       └─────────────┼─────────────┘
                     ↓
                 动态操作
```

因此：

> **JVM 负责类的加载和运行时类型基础设施，Java 反射 API 则在此基础上提供访问和操作类元数据的能力。**

---

## 二十六、反射与注解可以放在一起理解

前面的注解知识和反射知识可以连接成一条完整链路：

```text
                Java 源代码
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
       类 / 方法              注解
          │                     │
          └──────────┬──────────┘
                     ↓
                 javac 编译
                     │
                     ↓
                 .class 文件
                     │
                     ↓
                   JVM
                     │
                     ↓
                Class 对象
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        Field      Method    Constructor
          │          │          │
          └──────────┼──────────┘
                     ↓
               AnnotatedElement
                     │
                     ↓
                注解解析
                     │
                     ↓
              Annotation 对象
                     │
                     ↓
               框架执行逻辑
```

这也是 Java 框架中非常典型的一条技术链：

```text
注解提供元数据
      ↓
.class 文件保存元数据
      ↓
JVM 加载类
      ↓
反射获取 Class / Method / Field
      ↓
读取注解
      ↓
框架根据注解执行逻辑
```

---

## 二十七、反射的整体知识模型

可以用下面几个问题快速掌握 Java 反射：

```text
① 反射是什么？
   → 运行时动态获取类型信息并操作对象

② 反射从哪里开始？
   → Class 对象

③ Class 从哪里来？
   → User.class
   → obj.getClass()
   → Class.forName()

④ Class 能做什么？
   → 获取类、字段、方法、构造器、注解等信息

⑤ Field 能做什么？
   → 读取和修改字段

⑥ Method 能做什么？
   → 动态调用方法

⑦ Constructor 能做什么？
   → 动态创建对象

⑧ AnnotatedElement 是什么？
   → 提供统一的注解访问能力

⑨ Type 是什么？
   → 反射中描述泛型等类型信息的抽象

⑩ 反射为什么重要？
   → 框架可以在不知道具体实现的情况下动态工作
```

最终可以把 Java 反射浓缩成一句话：

> **反射以 `Class` 为核心入口，在运行时获取类及其成员的元数据，并通过 `Field`、`Method`、`Constructor` 等反射对象动态操作程序结构。**

而与注解结合后：

> **注解负责描述元数据，反射负责读取和操作元数据，框架则利用二者实现动态配置、对象创建、依赖注入、方法调用等功能。**