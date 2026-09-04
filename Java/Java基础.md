## 1. 基础部分

### 八种基本数据类型

- 数值型：整数类型（byte、short、int、long）和 浮点类型（float、double）
- 字符型：char
- 布尔型：boolean

int 和 float 是32位，long 和 double 都是64位。

### 自动装箱（Boxing）和拆箱（Unboxing）

装箱（Boxing）和拆箱（Unboxing）是将基本数据类型和对应的包装类之间进行转换的过程。

**自动装箱**主要发生在两种情况，一种是赋值时，另一种是在方法调用传入参数的时候。

对象封装有很多好处，可以把属性也就是数据跟处理这些数据的方法结合在一起，Java中绝大部分方法或类都是用来处理类类型对象的，如ArrayList集合类就只能以类作为他的存储对象，泛型只能使用引用类型，而不能使用基本类型。

> **自动装箱的弊端：**
> 
> 在一个循环中进行自动装箱操作的情况，可能会创建多余的对象，影响程序的性能。
>
>```java
>Integer sum = 0; for(int i=1000; i<5000; i++){ sum+=i; }
>
>// 上面的代码sum+=i可以看成sum = sum + i，但是 + 操作符不适用于Integer对象，首先sum进行自动拆箱操作，进行数值相加操作，最后发生自动装箱操作转换成Integer对象。其内部变化如下：
>
>int result = sum.intValue() + i; Integer sum = Integer.valueOf(result);
>```
>
>由于这里声明的 sum 为 Integer 类型，自动装箱实际上由编译器替换为 Integer.valueOf(...) 调用，命中 IntegerCache（默认 -128~127）时会复用缓存对象，但本例中循环值都已超出缓存范围，因此会创建将近 4000 个 Integer 对象，降低程序性能并加重 GC 负担。

**拆箱** 就是把引用类型的值赋值给基本类型。

### 值传递和引用传递的区别

**值传递**传递的是**值的副本**，主要用于基础数据类型，修改参数副本不会影响原变量的值。

**引用传递**传递的是对象引用的副本，两个引用（原引用和副本）指向同一个对象，因此通过副本修改对象内部数据，会影响原对象。但如果修改副本的指向（如重新赋值），不会影响原引用的指向。

### 重载和重写的区别

重载（Overload）是“同类不同参”，重写（Override）是“子类改父类”

### 向上转型和向下转型

- **向上转型**是使用父类类型的引用指向子类对象，通过这种方式，可以在运行时期采用不同的子类实现。
- **向下转型**是将父类引用转回其子类类型，但在执行前需要确认引用实际指向的对象类型以避免 `ClassCastException`。

### 抽象类和普通类的区别

- 实例化：普通类可以直接实例化对象，而抽象类不能被实例化，只能被继承。
- 方法实现：普通类中的方法可以有具体的实现，而抽象类中的方法可以有实现也可以没有实现。
- 非抽象子类继承抽象类，抽象方法必须重写。
- 抽象类不能被实例化。

### 抽象类和接口的区别

- 继承：接口可以多个实现，抽象类只能多继承。
- 接口只有定义，不能有方法的实现。

接口用于定义类的规范，抽象类用来定义和描述类的共同特性和行为。

**设计原则**：

- 优先使用接口：它是对行为的抽象，可以多实现，解耦更彻底。
- 抽象类适合"模板复用"：多个子类有公共代码（字段、方法实现），用抽象类抽取公共部分。
- 经典组合：**抽象类实现接口**（如 AbstractList 实现 List），把公共实现下沉到抽象类，子类只需实现少量方法。

### 接口能定义哪些方法

- 抽象方法。默认都是抽象方法，默认是 public 和 abstract。
- 默认方法。被 default 修饰，允许接口提供具体实现。
- 静态方法。属于接口本身，可以通过接口名直接调用，而不需要实现类的对象。
- 私有方法。private修饰，这些方法不能被实现类访问，只能在接口内部使用。

### 非静态内部类和静态内部类的区别

静态内部类可以看作一个外部的普通类。

- 非静态内部类依赖于外部类的实例，而静态内部类不依赖于外部类的实例。
- 非静态内部类可以直接访问外部类的所有成员（包括实例变量和方法）；静态内部类可以直接访问外部类的静态成员，访问外部类的实例成员则必须通过外部类的实例引用。
- 非静态内部类不能定义静态成员（Java 16 之前），而静态内部类可以定义静态成员。
- 非静态内部类在外部类实例化后才能实例化，而静态内部类可以独立实例化。
- 静态内部类和非静态内部类都可以访问外部类的私有成员（因为编译器会为嵌套类提供访问权限）——两者的区别在于静态内部类访问外部类的私有实例成员时必须先拿到外部类实例，而不是像非静态内部类那样能直接访问。

### 非静态内部类如何做到可以直接访问外部方法

**非静态内部类**可以直接访问**外部方法**是因为编译器在生成字节码时会为非静态内部类维护一个**指向外部类实例的引用**。这个引用使得非静态内部类能够访问外部类的实例变量和方法。编译器会在生成非静态内部类的构造方法时，将外部类实例作为参数传入，并在内部类的实例化过程中建立外部类实例与内部类实例之间的联系，从而实现直接访问外部方法的功能。

### static关键字的作用

static 关键字主要用于修饰类的成员（变量、方法、代码块）和内部类，其核心作用是将成员与类本身关联，而非与类的实例（对象）关联。

### final关键字的作用

final关键字主要有以下三个方面的作用：用于修饰类、方法和变量。

- 修饰类：表示这个类不能被继承，是类继承体系中的最终形态。例如：String类就是用final修饰的，保证了String类的不可变性和安全性，防止其他类通过继承来改变String类的行为和特性。
- 修饰方法：用final修饰的方法不能在子类中被重写。比如，java.lang.Object类中的getClass方法就是final的，因为这个方法的行为是由 Java 虚拟机底层实现来保证的，不应该被子类修改。
- 修饰变量：当final修饰基本数据类型的变量时，该变量一旦被赋值就不能再改变。对于引用数据类型，final修饰意味着这个引用变量不能再指向其他对象，但对象本身的内容是可以改变的。

final 的含义就是终极不变，如果用final修饰抽象类，会报编译错误。

### final、finally、finalize 的区别

- **final**：修饰符。修饰类（不可继承）、修饰方法（不可重写）、修饰变量（不可变——引用不能变，但引用指向的对象内部可变）。
- **finally**：异常处理的一部分，try/catch 后的**必定执行**块，通常用于释放资源（关闭流、连接）。
- **finalize**：Object 的方法，垃圾回收器回收对象前调用，JDK 9 起已弃用（执行时机不确定、可能影响 GC 性能，资源清理应使用 try-with-resources 或显式 close）。


### 深拷贝与浅拷贝

- **浅拷贝**是指只复制对象本身和其内部的值类型字段，但不会复制对象内部的引用类型字段，换句话说，浅拷贝只是创建一个新的对象，然后将原对象的字段值复制到新对象中，但如果原对象内部有引用类型的字段，只是将引用复制到新对象中，两个对象指向的是同一个引用对象。
- **深拷贝**是指在复制对象的同时，将对象内部的所有引用类型字段的内容也复制一份，而不是共享引用。换句话说，深拷贝会递归复制对象内部所有引用类型的字段，生成一个全新的对象以及其内部的所有对象。

实现对象深拷贝的方法有以下几种主要方式：

- **实现 Cloneable 接口**，要求对象及其所有引用类型字段都实现 Cloneable 接口，并且重写 clone() 方法。在 clone() 方法中，通过递归克隆引用类型字段来实现深拷贝。
- **使用序列化和反序列化**，通过将对象序列化为字节流，再从字节流反序列化为对象来实现深拷贝。要求对象及其所有引用类型字段都实现 Serializable 接口。
- 手动递归复制

### 泛型

泛型允许类、接口和方法在定义时使用一个或多个**类型参数**，这些类型参数**在使用时可以被指定为具体的类型**，适用于多种不同的数据类型执行相同的代码。

泛型中的类型在使用时指定，不需要强制类型转换（类型安全，编译器会检查类型）。

```java
// list中的元素都是Object类型，所以在取出集合元素时需要进行强制类型转化，很容易出现java.lang.ClassCastException异常。
List list = new ArrayList();
list.add("xxString");
list.add(100d);
list.add(new Person());

// 引入泛型，它将提供类型的约束，提供编译前的检查
List<String> list = new ArrayList<String>();
```

Java 里的“通配符”（Wildcard）主要指泛型中的 ?，用来表示某种未知的类型，常见的三种类型：
- `<?>`：任意类型
- `<? extends T>`：T 或 T 的子类
- `<? super T>`：T 或 T 的父类

**通配符（PECS 原则）**：

- `? extends T`：上界通配符，只能"读"（拿出来的元素一定是 T），不能"写"。适用：生产者 Producer。
- `? super T`：下界通配符，只能"写"（可以放 T 及其子类），读出来是 Object。适用：消费者 Consumer。

```java
List<? extends Number> producers = new ArrayList<Integer>(); // 可读不可写
Number n = producers.get(0);
// producers.add(1); // 编译报错

List<? super Integer> consumers = new ArrayList<Number>(); // 可写不可读
consumers.add(1);
// Integer i = consumers.get(0); // 编译报错，读出来是 Object
```

### 对象的创建方式有哪些

- 使用new关键字
- 使用Class类的newInstance()方法：通过 Java 的反射 API，在运行时动态地创建对象。这种方式不需要在编译时知道具体的类。应用场景：框架设计（如 Spring 的 IOC 容器）、动态代理等。
- 使用Constructor类的newInstance()方法：同样是通过反射机制，可以使用Constructor类的newInstance()方法创建对象。
- 使用clone()方法：通过实现 Cloneable 接口并重写 Object 类的 clone() 方法，可以基于一个现有对象（原型）创建一个新的副本对象。
- 使用反序列化：通过 ObjectInputStream 从一个字节流（通常是文件或网络）中重建一个对象。

### new出的对象什么时候回收

通过过关键字new创建的对象，强引用对象，由Java的垃圾回收器（Garbage Collector）负责回收。垃圾回收器的工作是在程序运行过程中自动进行的，它会周期性地检测不再被引用的对象，并将其回收释放内存。

具体来说，Java对象的回收时机是由垃圾回收器根据以下机制来决定的：  

**可达性分析算法**：HotSpot JVM 使用的核心判活算法。从一组 GC Roots（如虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈中 JNI 引用的对象等）出发，通过对象之间的引用链进行遍历，如果存在一条引用链到达某个对象，则说明该对象是可达的，反之不可达，不可达的对象将被回收。注意：Java 并不使用引用计数法，因为引用计数无法解决循环引用问题。  
**终结器（Finalizer）**：如果对象重写了finalize()方法，垃圾回收器会在回收该对象之前调用finalize()方法，对象可以在finalize()方法中进行一些清理操作。然而，终结器机制的使用不被推荐，因为它的执行时间是不确定的，可能会导致不可预测的性能问题，finalize() 方法自 Java 9 起已被标记为 @Deprecated。  

### 如何获取私有对象

在 Java 中，私有对象通常指的是类中被声明为 private 的成员变量或方法。由于 private 访问修饰符的限制，这些成员只能在其所在的类内部被访问。

不过，可以通过下面两种方式来间接获取私有对象。

- 使用公共访问器方法（getter 方法）：如果类的设计者遵循良好的编程规范，通常会为私有成员变量提供公共的访问器方法（即 getter 方法），通过调用这些方法可以安全地获取私有对象。
- 反射机制。反射机制允许在运行时检查和修改类、方法、字段等信息，通过反射可以绕过 private 访问修饰符的限制来获取私有对象。


### 反射机制

反射是指在**运行时**获取类的元数据信息（Class 对象，JVM 加载类后存放在方法区/元空间中），从而动态地进行对象的创建、方法调用、字段访问等操作。

![alt text](images/reflection.png)

>注意：运行时不读 class 文件，class 文件在**类加载阶段**已被解析为方法区的运行时数据结构，反射读的是这份元数据。

反射具有以下特性：

- **运行时类信息访问：** 反射机制允许程序在运行时获取类的完整结构信息，包括类名、包名、父类、实现的接口、构造函数、方法和字段等。
- **动态对象创建：** 可以使用反射API动态地创建对象实例，即使在编译时不知道具体的类名。通常通过 Constructor.newInstance() 方法实现（Class.newInstance() 自 Java 9 起已被标记为 @Deprecated，推荐使用 clazz.getDeclaredConstructor().newInstance()）。
- **动态方法调用：** 可以在运行时动态地调用对象的方法，包括私有方法。这通过Method类的invoke()方法实现，允许你传入对象实例和参数值来执行方法。
- **访问和修改字段值：** 反射还允许程序在运行时访问和修改对象的字段值，即使是私有的。这是通过Field类的get()和set()方法完成的。

反射的应用场景：

- **动态代理**
- **注解**
- **数据库驱动加载：** 在使用 JDBC 连接数据库时使用 Class.forName()通过反射加载数据库的驱动程序。
- **配置文件加载：** Spring 框架的 IOC（动态加载管理 Bean），Spring通过配置文件配置各种各样的bean。

    Spring通过XML配置模式装载Bean的过程：
    - 将程序中所有XML或properties配置文件加载入内存
    - Java类里面解析xml或者properties里面的内容，得到对应实体类的字节码字符串以及相关的属性信息
    - 使用反射机制，根据这个字符串获得某个类的Class实例
    - 动态配置实例的属性

**性能与安全**：反射有方法调用开销（比直接调用慢），且可以绕过访问控制（setAccessible(true) 可访问 private 成员，因此框架才可能注入 private 字段）。JDK 17+ 对 setAccessible 有模块化限制（需 --add-opens）。

### 注解

注解本质上是一种特殊的接口，它继承自 java.lang.annotation.Annotation 接口，所以注解也叫声明式接口，例如，定义一个简单的注解：

```java
public @interface MyAnnotation {
    String value();
}
```

编译后，Java 编译器会将其转换为一个继承自 Annotation 的接口，并生成相应的字节码文件。

根据注解的作用范围，Java 注解可以分为以下几种类型：

- 源码级别注解 ：仅存在于源码中，编译后不会保留 `@Retention(RetentionPolicy.SOURCE)`。
- 类文件级别注解 ：保留在 .class 文件中，但运行时不可见 `@Retention(RetentionPolicy.CLASS)`。
- 运行时注解 ：保留在 .class 文件中，并且可以通过反射在运行时访问 `@Retention(RetentionPolicy.RUNTIME)`。

只有运行时注解可以通过反射机制进行解析。

当注解被标记为 RUNTIME 时，Java 编译器会在生成的 .class 文件中保存注解信息。这些信息存储在字节码的属性表（Attribute Table）中，具体包括以下内容：

- RuntimeVisibleAnnotations ：存储运行时可见的注解信息。
- RuntimeInvisibleAnnotations ：存储运行时不可见的注解信息。
- RuntimeVisibleParameterAnnotations 和 RuntimeInvisibleParameterAnnotations ：存储方法参数上的注解信息。

通过工具（如 javap -v）可以查看 .class 文件中的注解信息。

### 异常体系

![alt text](images/error.png)

Java 的异常体系主要基于 Throwable 及其子类。Throwable 有两个重要的子类：Error 和 Exception，它们分别代表了不同类型的异常情况。

- Error (错误)： 表示运行环境的错误，错误是程序无法处理的严重问题，如虚拟机错误、动态链接库失效等。程序不应该尝试捕获这类错误。例如，OutOfMemoryError、StackOverflowError 等。
- Exception (异常)： 表示程序本身可以处理的异常情况。异常分为两大类：
    - 非运行时异常（受检异常，Checked Exception）： 这类异常在编译时就必须被捕获或者声明抛出。它们通常是外部错误，如文件不存在 (FileNotFoundException)、类未找到 (ClassNotFoundException) 等。非运行时异常强制程序员处理这些可能出现的问题，增强了程序的健壮性。
    - 运行时异常（非受检异常，Unchecked Exception 或 RuntimeException）： 这类异常特指 RuntimeException 及其子类。它与 Error 一起构成了 Java 中的非受检异常家族。运行时异常由程序逻辑错误导致，如空指针访问 (NullPointerException)、数组越界 (ArrayIndexOutOfBoundsException) 等。运行时异常是不需要在编译时强制捕获或声明的。

### Object类有哪些方法

Object 类是 Java 中所有类的根类，除 `Object` 本身外，所有类都直接或间接继承自它。它主要提供对象的**基本操作、类型判断、对象复制以及线程同步**等能力。

Object 类常用的方法包括：

- `getClass()`： 获取对象在运行时对应的 `Class` 对象，可以进一步通 **对象身份（引用）** 进行比较，即只有两个引用指向同一个对象时才返回 `true`。子类可以根据实际业务需求重写该方法，例如 `String` 就是根据字符串内容判断是否相等。

- `toString()`： 返回对象的字符串表示形式。`Object` 默认返回类似 `类名@哈希值` 的字符串，实际开发中通常会重写该方法，用于输出对象的属性信息，方便调试和日志记录。

- `clone()`： 创建当前对象的一个副本。`Object` 提供的是浅拷贝机制，使用时通常需要实现 `Cloneable` 接口，否则调用时可能抛出 `CloneNotSupportedException`。实际开发中通常更推荐使用拷贝构造、工厂方法等更加明确的对象复制方式。

- `wait()`： 使当前线程进入等待状态，同时释放当前对象的监视器锁，直到其他线程调用该对象的 `notify()` 或 `notifyAll()`，或者等待被中断。调用该方法时，当前线程必须持有对象的监视器锁。

- `wait(long timeout)`： 让当前线程最多等待指定的毫秒数。如果在等待期间收到 `notify()`、`notifyAll()` 或被中断，也可能提前结束等待。

- `wait(long timeout, int nanos)`： `wait()` 的更精确版本，在指定毫秒数的基础上增加纳秒级的等待时间。

- `notify()`： 唤醒一个正在该对象监视器上等待的线程。具体唤醒哪一个线程由 JVM 决定。调用该方法时，当前线程必须持有对象的监视器锁。

- `notifyAll()`： 唤醒所有正在该对象监视器上等待的线程。调用该方法时，当前线程同样必须持有对象的监视器锁。

- `finalize()`： 在对象被垃圾回收前尝试执行的清理方法。但该方法已经在 **Java 9 中被标记为 `@Deprecated`，并且不推荐使用**。现代 Java 应使用 `try-with-resources`、`AutoCloseable` 等机制显式释放资源。

可以将 `Object` 的方法按照功能分为以下几类：

```text
Object
│
├── 类型信息
│   └── getClass()
│
├── 对象比较
│   ├── equals()
│   └── hashCode()
│
├── 对象表示
│   └── toString()
│
├── 对象复制
│   └── clone()
│
├── 线程同步
│   ├── wait()
│   ├── wait(long)
│   ├── wait(long, int)
│   ├── notify()
│   └── notifyAll()
│
└── 对象生命周期
    └── finalize()  // 已废弃，不推荐使用
```

### == 和 equals 的区别

- `==`：比较的是"值"。对于基本类型比较数值是否相等；对于引用类型比较的是**内存地址**（是否指向同一个对象）。
- `equals()`：Object 中的默认实现等价于 `==`（比较地址），但大多数类（String、Integer、包装类、集合）都重写了它，改为**比较内容**。

```java
String a = new String("abc");
String b = new String("abc");
a == b;            // false，两个不同的堆对象
a.equals(b);       // true，内容相同
```

**重写 equals 必须重写 hashCode**：HashMap/HashSet 依赖 hashCode 定位桶，如果两个对象 equals 相等但 hashCode 不同，会导致同一个 key 存进不同桶，出现重复数据。

### hashCode与 equals 约定

1. 两个对象 equals 相等 → hashCode **必须相等**（否则集合中出现重复元素）。
2. 两个对象 hashCode 相等 → equals **不一定**相等（哈希冲突，靠 equals 进一步判断）。
3. equals 相等的对象，其 hashCode 不能变（所以 hashCode 不能依赖可变字段）。

一个常见的错误：只重写 equals 不重写 hashCode，HashSet<User> 中两个内容相同的 User 会被当成两个元素


**hashCode 设计**：`Objects.hash(字段1, 字段2)` 或 `31 * 结果 + 字段.hashCode()`

**为什么使用31：** 奇素数，分布均匀；`31 * i` 可被 JVM 优化为 `(i << 5) - i`，移位 + 减法性能高。

### String、StringBuilder、StringBuffer 的区别

- **String**：不可变（final 修饰类）。**JDK 8 及以前底层是 char[]，JDK 9 起（JEP 254）改为 byte[] + coder**（紧凑字符串：Latin-1 编码的字符串每个字符只占 1 字节，节省约一半内存）。任何"修改"都会生成新对象。好处：可缓存 hashCode、可安全共享（线程安全）、适合做常量池/类加载参数。
- **StringBuilder**：可变，非线程安全，单线程下拼接性能最高。
- **StringBuffer**：可变，线程安全（方法加了 synchronized），多线程下用，性能略低。

### Integer 缓存与自动装箱

- `Integer.valueOf()` 会使用 **IntegerCache 缓存 -128~127** 的对象，超出范围才 new 新对象。
- 自动装箱调用 `valueOf()`，自动拆箱调用 `intValue()`。

```java
Integer a = 100, b = 100;
a == b;              // true，都在缓存范围内
Integer c = 200, d = 200;
c == d;              // false，超出缓存范围，是两个对象
```

> 面试延伸：Long、Short、Byte、Character 也有类似缓存（Long 也是 -128~127，Character 是 0~127），只有 Float/Double 没有。


**finally 与 return 的执行顺序（面试陷阱）**：

```java
// 结论1：finally 在 return 之前执行，但 finally 中修改返回值不生效（返回值已暂存）
public static int test() {
    int a = 1;
    try {
        return a;   // 先把 a=1 暂存到返回值槽
    } finally {
        a = 2;      // 修改的是局部变量，返回值仍是 1
    }
}
// 结果：1

// 结论2：finally 中如果也有 return，会覆盖 try 中的 return（不推荐这样写）
// 结论3：try 中有 System.exit(0) 时 finally 不执行
```

### try-with-resources（Java 7+）

自动关闭实现了 AutoCloseable 的资源，比传统 finally 手动关闭更简洁、更安全（不用处理"关闭本身抛异常"的嵌套问题）。

```java
// 传统写法（易漏、嵌套 try 繁琐）
FileInputStream in = null;
try {
    in = new FileInputStream("a.txt");
    // ...
} finally {
    if (in != null) in.close();  // 如果上面抛异常，这里可能 NPE；close 异常还会覆盖原异常
}

// try-with-resources：资源自动关闭，多个资源用分号分隔
try (FileInputStream in = new FileInputStream("a.txt");
     BufferedInputStream bis = new BufferedInputStream(in)) {
    // ...
} // 自动调用 close()，且关闭异常会以 suppressed 形式附加到主异常

// 注意：变量是 final 的，作用域在 try 块内
```

**底层原理**：编译后等价于 try-finally，但多了一个细节——如果 try 块抛异常且 close() 也抛异常，close 的异常会被**抑制（suppressed）**，附加在主异常上（通过 Throwable.addSuppressed），保证主异常不被掩盖。

### Future 与 completedFuture

在 CompletableFuture 之前，Java 用 Future 表示一个异步任务的结果

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<String> future = executor.submit(() -> {
    Thread.sleep(3000);
    return "外卖到了";
});

// 痛点1：只能轮询检查
while (!future.isDone()) {
    Thread.sleep(500); // 傻等
}

// 痛点2：获取结果会阻塞当前线程
String result = future.get(); // 必须阻塞等待

// 痛点3：无法串联多个异步任务
// 拿到结果后想继续处理？很难！
```

CompletableFuture 实现了 Future 接口，但提供了 50+ 个方法，让你能像写同步代码一样写异步代码。

1. 非阻塞回调
```java
CompletableFuture.supplyAsync(() -> {
    Thread.sleep(3000);
    return "外卖到了";
}).thenAccept(result -> {
    System.out.println(result); // 3秒后自动打印，不阻塞主线程
});

System.out.println("我先干别的去"); // 立即执行
```

2. 链式组合（流水线处理）
```java
CompletableFuture.supplyAsync(() -> "取外卖")
    .thenApply(address -> "去" + address + "取")   // 转换
    .thenApply(msg -> msg + "，然后回家")           // 继续转换
    .thenAccept(System.out::println);              // 最终消费
```

3. 合并多个异步任务
```java
CompletableFuture<String> order = CompletableFuture.supplyAsync(() -> "下单");
CompletableFuture<String> cook = CompletableFuture.supplyAsync(() -> "做饭");
CompletableFuture<String> deliver = CompletableFuture.supplyAsync(() -> "配送");

// 等所有任务都完成
CompletableFuture.allOf(order, cook, deliver).join();

// 或者任意一个完成就继续
CompletableFuture.anyOf(order, cook, deliver).thenAccept(System.out::println);
```

4. 异常处理（更优雅）
```java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) throw new RuntimeException("厨房着火");
    return "美食";
}).exceptionally(ex -> {
    System.out.println("出错了：" + ex.getMessage());
    return "泡面"; // 默认兜底
}).thenAccept(System.out::println);
```

5. 手动控制完成时机
```java
CompletableFuture<String> future = new CompletableFuture<>();

// 在另一个线程中手动完成
new Thread(() -> {
    Thread.sleep(2000);
    future.complete("结果来了");
}).start();

// 主线程可以继续干别的事，2秒后自动拿到结果
future.thenAccept(System.out::println);
```

### 序列化和反序列化的实现

- 标记接口 Serializable：这是所有可序列化Java类的“通行证”。它是一个空接口，不包含任何方法，仅仅作为标记，告诉JVM这个类的对象可以被序列化。如果一个类没有实现它，在序列化时就会抛出NotSerializableException异常。
- 序列化流 ObjectOutputStream：用于执行序列化操作。它的核心方法是 writeObject(Object obj)，能将传入的对象转换为字节流并写入到与之关联的输出流中（如文件流）。
- 反序列化流 ObjectInputStream：用于执行反序列化操作。它的核心方法是 readObject()，能从与之关联的输入流中读取字节序列，并将其重构为原始的Java对象。

```java
public class User implements Serializable {
    // ... 其他代码

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject(); // 先执行默认序列化
        // 在此添加自定义逻辑，例如加密 password
        out.writeObject("加密后的密码"); 
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject(); // 先执行默认反序列化
        // 在此添加自定义逻辑，例如解密 password
        String encryptedPwd = (String) in.readObject();
        // this.password = decrypt(encryptedPwd); 
    }
}
```

**serialVersionUID（必问）**：序列化机制会给每个可序列化类生成一个版本号（serialVersionUID），反序列化时用它校验"序列化时的类"和"反序列化时的类"是否兼容：

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L; // 显式声明
    // ...
}
```

- **如果不声明**：JVM 会根据类结构（字段、方法、接口等）自动生成一个 UID。一旦类结构发生任何变化（如新增/删除字段、改方法签名），自动生成的 UID 就会变，反序列化时抛出 `InvalidClassException`。
- **如果显式声明**：即使类结构变了，只要 UID 不变，反序列化就能兼容（新增字段用默认值补齐，删除字段直接忽略）——这也是**向后兼容**的关键。
- **实际开发规范**：凡是实现 Serializable 的类（DTO、Entity、消息体），**必须显式声明 serialVersionUID**，否则类一改就线上报错。

**序列化的其他注意点**：

- 静态变量不会被序列化（它属于类，不属于对象）；transient 修饰的字段不会被序列化。
- 序列化不会调用构造器：反序列化通过反射直接创建对象（不执行构造函数），所以构造器中的初始化逻辑不会在反序列化时执行。
- 子类实现 Serializable 而父类没有：父类的字段不会被序列化（且父类必须有**无参构造器**，否则反序列化报错）。
- 单例类实现 Serializable 会破坏单例（反序列化会产生新对象），解决办法：重写 readResolve() 返回原单例。





## Java 新特性面试题

### Java 8：Stream 流式编程

**特性**：Stream 是数据渠道，不存储数据，只负责对集合数据进行**惰性求值**的处理流水线。

**惰性求值（核心）**：中间操作（filter/map/distinct/sorted 等）只是"登记"了操作，**不会立即执行**；只有遇到**终止操作**（collect/forEach/count/reduce）时才会真正遍历执行。

```java
List<String> names = list.stream()
    .filter(s -> s.length() > 3)      // 中间操作：惰性，不执行
    .map(String::toUpperCase)          // 中间操作：惰性，不执行
    .collect(Collectors.toList());     // 终止操作：此时才触发全流程
```

**短路操作**：limit()、findFirst()、anyMatch() 等终止操作一旦满足条件就提前结束遍历，不用处理完整个集合，可以显著优化性能（如"找到第一个满足条件的就返回"）。

**常用操作**：

```java
list.stream()
    .filter(x -> x > 10)                                  // 过滤
    .map(x -> x * 2)                                      // 转换（一对一）
    .flatMap(l -> l.stream())                             // 扁平化（一对多）
    .distinct()                                           // 去重
    .sorted(Comparator.reverseOrder())                    // 排序
    .limit(5)                                             // 截取前5个
    .skip(2)                                              // 跳过前2个
    .collect(Collectors.toList());

// 分组 / 分区 / 拼接
Collectors.groupingBy(User::getDeptId);        // 按部门分组 → Map<Long, List<User>>
Collectors.partitioningBy(u -> u.getAge() > 18); // 分区 → Map<Boolean, List<User>>
Collectors.joining(", ");                      // 元素拼接
Collectors.toMap(User::getId, u -> u);         // 转 Map（注意 key 冲突需传合并函数）
```

**reduce（归约）**：`reduce(0, (a, b) -> a + b)` 把流中所有元素组合成一个值。

**parallelStream 的坑**：并行流默认使用共享的 `ForkJoinPool.commonPool()`（线程数 = CPU 核数 - 1），如果多个并行流同时使用会互相抢占线程；且并行流不保证顺序，处理有状态操作（如累加计数器）时线程不安全。大数据量 + 无状态操作才考虑并行。

**Stream 与 for 循环**：常规数据量下 for 循环通常更快（Stream 有对象创建开销），Stream 的价值在于**声明式、可读、易并行**，而不是性能。面试答"Stream 性能更好"是错的。

### Java 8：Lambda 与函数式接口

- Lambda 本质是**函数式接口的匿名实现类**的语法糖。
- 函数式接口：**只有一个抽象方法**的接口，用 `@FunctionalInterface` 标注。

**四大内置函数式接口**：

| 接口 | 方法 | 用途 |
|---|---|---|
| Function<T, R> | R apply(T t) | 输入 T，输出 R（转换） |
| Predicate<T> | boolean test(T t) | 输入 T，输出 boolean（判断） |
| Consumer<T> | void accept(T t) | 输入 T，无输出（消费） |
| Supplier<T> | T get() | 无输入，输出 T（生产） |

**方法引用**：`ClassName::method`，如 `System.out::println`、`User::getName`、`String::length`。

**变量捕获**：Lambda 只能引用**实际不可变（effectively final）**的局部变量，因为 Lambda 是闭包，需要拷贝变量值，若变量会变则无法保证一致性。

### Java 8：Optional

- 用于优雅处理空值，**避免 NPE**，强迫开发者显式处理"可能为 null"的情况。

```java
Optional.ofNullable(user)
    .map(User::getAddress)              // 中间值也可能为空，继续传递
    .map(Address::getCity)
    .orElse("未知");                     // 任一环节为空时兜底

// orElse 与 orElseGet 的区别：orElse(expr) 无论是否有值都会执行 expr，orElseGet(supplier) 只有为空时才执行
// 空值抛异常：orElseThrow(() -> new BizException("用户不存在"));
```

**反模式**：不要用 Optional 作为字段类型、方法参数、集合元素（序列化不友好）；Optional 只是返回值的容器。

### Java 9~21：新特性速览

- **var（Java 10）**：局部变量类型推断，`var list = new ArrayList<String>();`（不能用于字段、方法参数、返回类型）。
- **文本块（Java 15 正式）**：`"""..."""` 多行字符串，适合写 SQL/JSON。
- **Record（Java 16 正式）**：不可变数据载体，自动生成构造器、equals/hashCode/toString。适合 DTO、VO。
- **Sealed Class（Java 17 正式）**：密封类，限制继承的类集合，与 switch 模式匹配配合使用。
- **Switch 表达式（Java 14 正式）**：switch 可以作为表达式返回值，无需 break，用 `->` 语法。
- **模式匹配 instanceof（Java 16 正式）**：`if (obj instanceof String s)` 直接绑定变量，省去强转。
- **虚拟线程（Java 21 正式，JEP 444）**：M:N 轻量级线程，百万级并发 IO 场景首选（"线程和操作系统线程区别"一节已有详解）。
- **模式匹配 for switch（Java 21 正式）**：switch 可直接对类型做模式匹配，配合 sealed class 实现穷举检查。

```java
// Record + Sealed + switch 模式匹配组合（Java 21）
sealed interface Shape permits Circle, Rect {}
record Circle(double r) implements Shape {}
record Rect(double w, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {           // 穷举性：编译器能检查是否覆盖所有子类
        case Circle c -> Math.PI * c.r() * c.r();
        case Rect r -> r.w() * r.h();
    };
}
```


