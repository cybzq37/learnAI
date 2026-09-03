# Java 注解（Annotation）

![alt text](images/reflection0.png)

## 一、注解的本质

Java 注解（Annotation）是一种用于向程序代码中添加 **元数据（Metadata）** 的机制。

注解本身通常不直接参与业务逻辑，而是用于描述类、方法、字段、参数等程序元素，从而让**编译器、开发工具、框架或运行时程序**根据这些信息执行相应的处理。

例如：

```java
public @interface MyAnnotation {

    String value();

}
```

使用时：

```java
@MyAnnotation("hello")
public class MyClass {
}
```

可以把注解理解成：

> **给程序元素附加的一组结构化元数据。**

### 1. 注解类型本质上是一种特殊的接口类型

使用 `@interface` 定义的不是普通的类，而是一种**注解类型（Annotation Type）**：

```java
public @interface MyAnnotation {

    String value();

}
```

注解类型是一种特殊的接口类型，并隐式继承：

```java
java.lang.annotation.Annotation
```

可以抽象理解为：

```text
Annotation
    ↑
MyAnnotation
```

其中 `Annotation` 是所有注解类型的公共父接口。

需要注意：

> `MyAnnotation` 虽然表现为接口，但它不是通常意义上的业务接口，而是一种专门用于定义注解结构的接口类型。

例如：

```java
public @interface MyAnnotation {

    String value();

    int count() default 1;

}
```

其中：

- `value()` 和 `count()` 称为**注解元素（annotation element）**
- `value()` 没有默认值，使用注解时必须提供
- `count()` 有默认值，可以省略

使用：

```java
@MyAnnotation(value = "hello")
public class MyClass {
}
```

当注解只有一个名为 `value` 的元素时，可以简写为：

```java
@MyAnnotation("hello")
public class MyClass {
}
```

---

## 二、@Target：注解可以使用在哪里

`@Target` 用于限制一个注解可以应用到哪些程序元素。

例如：

```java
@Target(ElementType.METHOD)
public @interface MyAnnotation {
}
```

表示 `MyAnnotation` 只能用于方法：

```java
@MyAnnotation
public void test() {
}
```

### 1. ElementType

Java 21 中，`ElementType` 有 12 种取值：

| ElementType | 可以使用的位置 |
|---|---|
| `TYPE` | 类、接口、枚举、注解类型 |
| `FIELD` | 字段，包括枚举常量 |
| `METHOD` | 方法 |
| `PARAMETER` | 方法或构造器参数 |
| `CONSTRUCTOR` | 构造方法 |
| `LOCAL_VARIABLE` | 局部变量 |
| `ANNOTATION_TYPE` | 注解类型本身 |
| `PACKAGE` | 包 |
| `TYPE_PARAMETER` | 类型参数声明 |
| `TYPE_USE` | 类型使用位置 |
| `MODULE` | 模块 |
| `RECORD_COMPONENT` | Record 组件 |

其中最常见的是：

```text
TYPE
FIELD
METHOD
PARAMETER
CONSTRUCTOR
```

### 2. TYPE_PARAMETER 和 TYPE_USE

`TYPE_PARAMETER` 和 `TYPE_USE` 是 Java 8 引入的两个容易混淆的类型。

`TYPE_PARAMETER` 用于修饰**类型参数声明**：

```java
public <@MyAnnotation T> void test(T value) {
}
```

这里的 `T` 是类型参数。

`TYPE_USE` 用于修饰**类型的使用位置**，范围比 `FIELD`、`METHOD` 等更广：

```java
List<@MyAnnotation String> list;

@MyAnnotation
String name;
```

`TYPE_USE` 主要用于描述类型本身，一些类型检查框架会利用它对类型进行额外约束。

### 3. 不指定 @Target 会怎样？

如果没有声明 `@Target`：

```java
public @interface MyAnnotation {
}
```

那么该注解可以应用于**所有声明上下文和所有类型上下文**。

因此，如果希望明确限制注解的使用位置，通常应该显式声明：

```java
@Target(...)
```

---

## 三、@Retention：注解保留多久

`@Retention` 用于控制注解的**保留策略**，有三种取值：

```text
SOURCE
CLASS
RUNTIME
```

| RetentionPolicy | `.java` | `.class` | 运行时反射 |
|---|---|---|---|
| `SOURCE` | ✓ | ✗ | ✗ |
| `CLASS` | ✓ | ✓ | ✗ |
| `RUNTIME` | ✓ | ✓ | ✓ |

可以简单理解为：

```text
SOURCE
  ↓
编译后消失

CLASS
  ↓
保留在 .class
  ↓
运行时反射不可见

RUNTIME
  ↓
保留在 .class
  ↓
运行时可以通过反射获取
```

其中 `CLASS` 是默认的保留策略。

### 1. SOURCE

```java
@Retention(RetentionPolicy.SOURCE)
```

注解只存在于源码中，编译后不会进入 `.class` 文件。

典型用途：

- 编译器检查
- IDE 提示
- 代码生成

例如：

```java
@Override
public String toString() {
    return "...";
}
```

### 2. CLASS

```java
@Retention(RetentionPolicy.CLASS)
```

注解会被编译进 `.class` 文件，但运行时通常无法通过 Java 反射 API 获取。

### 3. RUNTIME

```java
@Retention(RetentionPolicy.RUNTIME)
```

注解会被保存在 `.class` 文件中，并且运行时可以通过反射获取。

例如：

```java
Class<?> clazz = MyClass.class;

MyAnnotation annotation =
        clazz.getAnnotation(MyAnnotation.class);

if (annotation != null) {
    System.out.println(annotation.value());
}
```

因此：

> **如果希望在运行时通过反射读取注解，通常必须使用 `RetentionPolicy.RUNTIME`。**

---

## 四、注解在 .class 文件中的存储

对于 `CLASS` 和 `RUNTIME` 注解，编译器会将注解信息写入 `.class` 文件的 **Attribute（属性）** 中。

与注解相关的典型属性包括：

```text
RuntimeVisibleAnnotations
RuntimeInvisibleAnnotations

RuntimeVisibleParameterAnnotations
RuntimeInvisibleParameterAnnotations
```

其中：

- `RuntimeVisibleAnnotations`：保存运行时可见的注解
- `RuntimeInvisibleAnnotations`：保存运行时不可见的注解
- `RuntimeVisibleParameterAnnotations`：保存方法或构造器参数上的运行时可见注解
- `RuntimeInvisibleParameterAnnotations`：保存方法或构造器参数上的运行时不可见注解

例如：

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {

    String value();

}
```

使用：

```java
@MyAnnotation("hello")
public class MyClass {
}
```

编译后，相关信息会进入：

```text
MyClass.class
    │
    └── Attribute Table
            │
            └── RuntimeVisibleAnnotations
```

可以使用：

```bash
javap -v MyClass.class
```

查看 `.class` 文件中的注解属性。

因此：

> **运行时能够获取的注解并不是凭空产生的，而是先以元数据的形式存在于 `.class` 文件中。**

---

## 五、反射如何获取注解

运行时获取注解主要依赖 Java 反射 API。

`Class`、`Method`、`Field`、`Constructor`、`Parameter` 等反射对象都实现了：

```java
java.lang.reflect.AnnotatedElement
```

该接口提供了常用的注解操作：

```java
<T extends Annotation>
T getAnnotation(Class<T> annotationClass);

Annotation[] getAnnotations();

boolean isAnnotationPresent(
        Class<? extends Annotation> annotationClass);
```

分别用于：

```text
getAnnotation()
    → 获取指定注解

getAnnotations()
    → 获取所有注解

isAnnotationPresent()
    → 判断是否存在指定注解
```

例如：

```java
Method method =
        MyService.class.getMethod("execute");

MyAnnotation annotation =
        method.getAnnotation(MyAnnotation.class);

if (annotation != null) {
    System.out.println(annotation.value());
}
```

---

## 六、运行时注解解析过程

以：

```java
method.getAnnotation(MyAnnotation.class);
```

为例，可以将整个过程理解为：

```text
.class 文件
    │
    │ RuntimeVisibleAnnotations
    ↓
反射 API
    │
    │ getAnnotation()
    ↓
解析注解元数据
    ↓
返回 Annotation 对象
    ↓
annotation.value()
```

在 JDK 内部，反射 API 会在需要时解析 `.class` 文件中的注解元数据，相关实现会使用 `AnnotationParser` 等机制，将字节码中的注解数据转换为 Java 层面的注解表示。

简化后的过程：

```text
.class 文件
    │
    │ RuntimeVisibleAnnotations
    ↓
AnnotationParser
    │
    │ 解析注解数据
    ↓
注解元数据
    │
    ↓
AnnotationInvocationHandler
    │
    ↓
动态代理对象
    │
    ↓
MyAnnotation
```

因此：

```java
annotation.value()
```

实际上是在访问运行时注解对象中的属性值。

---

## 七、JVM 与 Java 反射的职责

这里需要区分 JVM 和 Java 反射各自负责的部分。

```text
JVM
 ├─ 加载、验证、准备、解析类
 ├─ 读取 class 文件结构
 └─ 提供类运行时环境

Java 反射 / JDK
 ├─ 访问注解元数据
 ├─ 解析 Annotation Attribute
 └─ 创建并返回注解对象
```

因此，不应该简单理解为：

> JVM 在加载类时就把所有注解直接解析成 Java 注解对象，然后 `getAnnotation()` 直接获取。

更准确的理解是：

> **注解信息首先以 class 文件属性的形式保存；运行时反射 API 在需要时解析这些元数据，并将其表示为 Java 层面的注解对象。**

---

## 八、@Target 和 @Retention 的区别

可以通过两个问题快速区分：

```text
@Target
    → 注解可以放在哪里？

@Retention
    → 注解可以保留多久？
```

例如：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
}
```

表示：

```text
@Target
    ↓
只能用于方法

@Retention
    ↓
运行时仍然可见
```

因此：

```java
@MyAnnotation
public void test() {
}
```

合法。

而：

```java
@MyAnnotation
public class Test {
}
```

不合法。

同时，运行时可以通过：

```java
method.getAnnotation(MyAnnotation.class);
```

获取该注解。

---

## 九、Java 注解的完整流程

将前面的内容串起来，可以把 Java 注解的整个过程概括为：

```text
定义注解
    ↓
使用注解
    ↓
javac 编译
    ↓
.class 文件
    │
    └── Annotation Attribute
            │
            ↓
    RuntimeVisibleAnnotations
            │
            ↓
        运行时反射
            │
            ↓
      AnnotationParser
            │
            ↓
      注解运行时对象
            │
            ↓
       getAnnotation()
            │
            ↓
      annotation.value()
```

最终可以记住三个核心点：

> **`@Target` 决定“注解能放在哪里”，`@Retention` 决定“注解保留多久”，反射 API 决定“运行时如何获取注解”。**

对于 `RUNTIME` 注解：

> **注解信息会以 class 文件属性的形式保存，运行时由 JDK 的注解解析机制读取这些元数据，并将其表示为 Java 层面的注解对象。**