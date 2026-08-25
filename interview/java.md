### 反射机制

反射具有以下特性：
- 运行时类信息访问：反射机制允许程序在运行时获取类的完整结构信息，包括类名、包名、父类、实现的接口、构造函数、方法和字段等。
- 动态对象创建：可以使用反射API动态地创建对象实例，即使在编译时不知道具体的类名。通常通过 Constructor.newInstance() 方法实现（Class.newInstance() 自 Java 9 起已被标记为 @Deprecated，推荐使用 clazz.getDeclaredConstructor().newInstance()）。
- 动态方法调用：可以在运行时动态地调用对象的方法，包括私有方法。这通过Method类的invoke()方法实现，允许你传入对象实例和参数值来执行方法。
- 访问和修改字段值：反射还允许程序在运行时访问和修改对象的字段值，即使是私有的。这是通过Field类的get()和set()方法完成的。

反射通过运行时读取class实现。

### 注解

注解本质是一个继承了Annotation的特殊接口，其具体实现类是Java运行时生成的动态代理类。

我们通过反射获取注解时，返回的是Java运行时生成的动态代理对象。通过代理对象调用自定义注解的方法，会最终调用AnnotationInvocationHandler的invoke方法。该方法会从memberValues这个Map中索引出对应的值。而memberValues的来源是Java常量池。

### 垃圾回收

### 解释和编译过程

Java经过编译之后生成字节码文件，接下来进入JVM中，就有两个步骤编译和解释。 

JVM 启动后由解释器逐条解释执行字节码；同时 JVM 通过方法调用计数器和回边计数器识别热点代码，达到阈值后由 JIT 编译为本地机器码并缓存到 Code Cache中，后续执行直接走机器码。

### 值传递和引用传递

值传递传递的是值的副本，引用类型传递 “引用的副本”，通过副本可修改对象内容，但无法改变原引用的指向。

### 重载和重写的区别

### 设计模式

### 抽象类和普通类、接口的区别

- 实例化：普通类可以直接实例化对象，而抽象类不能被实例化，只能被继承。

- 方法实现：普通类中的方法可以有具体的实现，而抽象类中的方法可以有实现也可以没有实现。

- 接口用于定义行为规范，可以多实现，只能有常量和抽象方法（Java 8 以后可以有默认方法和静态方法）。适用于定义类的能力或功能。

### 深拷贝和浅拷贝的区别

- 浅拷贝是指只复制对象本身和其内部的值类型字段，但不会复制对象内部的引用类型字段。换句话说，浅拷贝只是创建一个新的对象，然后将原对象的字段值复制到新对象中，但如果原对象内部有引用类型的字段，只是将引用复制到新对象中，两个对象指向的是同一个引用对象。
- 深拷贝是指在复制对象的同时，将对象内部的所有引用类型字段的内容也复制一份，而不是共享引用。换句话说，深拷贝会递归复制对象内部所有引用类型的字段，生成一个全新的对象以及其内部的所有对象。

### java的异常体系

### completedFuture

java的异步回调：

- Future用于表示异步计算的结果，只能通过阻塞或者轮询的方式获取结果，而且不支持设置回调方法，Java 8之前若要设置回调一般会使用guava的ListenableFuture，回调的引入又会导致臭名昭著的回调地狱（下面的例子会通过ListenableFuture的使用来具体进行展示）。
- CompletableFuture对Future进行了扩展，可以通过设置回调的方式处理计算结果，同时也支持组合操作，支持进一步的编排，同时一定程度解决了回调地狱的问题。

### java21协程

虚拟线程：这是 Java 21 引入的一种轻量级并发的新选择。它的栈在堆上动态分配，运行时挂载到平台载体线程（carrier thread）上执行，挂起时栈被卸载到堆中，因此可以创建大量虚拟线程而不消耗大量内存，显著提升了 I/O 密集型应用的吞吐量和响应速度。可以使用静态构建方法、构建器或 ExecutorService 来创建和使用虚拟线程。

### 序列化和反序列化的实现

其实，像序列化和反序列化，无论这些可逆操作是什么机制，都会有对应的处理和解析协议，例如加密和解密，TCP的粘包和拆包，序列化机制是通过序列化协议来进行处理的，和 class 文件类似，它其实是定义了序列化后的字节流格式，然后对此格式进行操作，生成符合格式的字节流或者将字节流解析成对象。

在Java中通过序列化对象流来完成序列化和反序列化：

ObjectOutputStream：通过writeObject(）方法做序列化操作。
ObjectInputStrean：通过readObject()方法做反序列化操作。
只有实现了Serializable或Externalizable接口的类的对象才能被序列化，否则抛出异常！

通过实现 ObjectInputStream.readObject() 和 ObjectOutputStream.writeObject(obj) 

## 设计模式

### volatile 和 sychronized 如何实现单例模式

```java
public class SingleTon {

    // volatile 关键字修饰变量 防止指令重排序
    private static volatile SingleTon instance = null;
    private SingleTon(){}
     
    public static  SingleTon getInstance(){
        if(instance == null){
            //同步代码块 只有在第一次获取对象的时候会执行到 ，第二次及以后访问时 instance变量均非null故不会往下执行了 直接返回啦
            synchronized(SingleTon.class){
                if(instance == null){
                    instance = new SingleTon();
                }
            }
        }
        return instance;
    }
}
```
正确的双重检查锁定模式需要使用 volatile。volatile主要包含两个功能。

- 保证可见性。使用 volatile 定义的变量，将会保证对所有线程的可见性。
- 禁止指令重排序优化。 

由于 volatile 禁止对象创建时指令之间重排序，所以其他线程不会访问到一个未初始化的对象，从而保证安全性。

### 代理模式和适配器模式有什么区别

- 目的不同：代理模式主要关注控制对对象的访问，而适配器模式则用于接口转换，使不兼容的类能够一起工作。  
- 结构不同：代理模式一般包含抽象主题、真实主题和代理三个角色，适配器模式包含目标接口、适配器和被适配者三个角色。  
- 应用场景不同：代理模式常用于添加额外功能或控制对对象的访问，适配器模式常用于让不兼容的接口协同工作。

### 责任链模式使用场景是什么

责任链模式的使用场景核心很明确，就是一个请求需要多个独立的处理逻辑来承接，同时不想让请求发起方和所有处理者产生强关联，还得让处理流程能灵活调整，简单说就是谁能处理就谁来接手，整个处理顺序和参与节点能按需改动。

比如实际开发里最常遇到的接口请求校验，用户调用我们的接口时，可能得先检查登录状态，再验证 token 是否有效，接着确认接口访问权限，最后还要限制请求频率，这些校验逻辑各自独立，而且不同接口需要的校验步骤不一样，比如登录接口只需要验证验证码，查询用户信息的接口得同时过登录和权限校验。要是不用责任链，就得在每个接口里写一堆 if-else 把这些校验串起来，后续想改某个校验规则，所有相关接口都得动，维护起来特别麻烦。

### 策略模式

策略模式定义一系列算法，把它们一个个封装起来，并且使它们可相互替换。从而让算法可独立于使用它的客户而变化。

## I/O 面试题

### BIO/NIO/AIO 区别

- BIO（blocking IO）：就是传统的 java.io 包，它是基于流模型实现的，交互的方式是**同步阻塞**方式，也就是说在读入输入流或者输出流时，在读写动作完成之前，线程会一直阻塞在那里，它们之间的调用是可靠的线性顺序。优点是代码比较简单、直观；缺点是 IO 的效率和扩展性很低，容易成为应用性能瓶颈。
- NIO（non-blocking IO）是 Java 1.4 引入的 java.nio 包，提供了 Channel、Selector、Buffer 等新的抽象，可以构建**多路复用、同步非阻塞** IO 程序，同时提供了更接近操作系统底层高性能的数据操作方式。
- AIO（Asynchronous IO）是 Java 1.7 引入的，对 NIO 的扩展，提供了**异步非阻塞**的 IO 操作方式，所以人们叫它 AIO（Asynchronous IO）。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会阻塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。


## 集合面试题

### Java中的集合有哪些

List是**有序**的Collection，使用此接口能够精确的控制每个元素的插入位置，用户能根据索引访问List中元素。常用的实现List的类有LinkedList，ArrayList，Vector，Stack。

- ArrayList 是容量可变的**非线程安全**列表，其底层使用数组实现。当集合扩容时，会创建更大的数组，并把原数组复制到新数组。ArrayList 支持对元素的快速随机访问，在尾部追加/删除元素效率很高，但在中间位置插入/删除需要搬移元素，代价较高。
- LinkedList 本质是一个**双向链表**，支持高效的头尾插入/删除和作为双端队列使用。

> LinkedList 插入/删除比 ArrayList 更快 是一个常见误区：其 **O(1) 的前提是已经持有目标节点的引用**；如果要在任意位置插入/删除，仍需先 O(n) 遍历链表找到位置，加上每个节点都需要独立分配、对 CPU 缓存不友好，实测大多数场景下 LinkedList 反而比 ArrayList 慢，这也是现在主流建议优先使用 ArrayList 的原因。

Set不允许存在重复的元素，与List不同，set中的元素是**无序**的。常用的实现有HashSet，LinkedHashSet 和 TreeSet。

- HashSet通过HashMap实现，HashMap的Key即HashSet存储的元素，所有Key都是用相同的Value，一个名为PRESENT的Object类型常量。使用Key保证元素唯一性，但不保证有序性。由于其底层的 HashMap 本身就是非线程安全的，因此 HashSet 也是非线程安全的。
- LinkedHashSet继承自HashSet，通过LinkedHashMap实现，使用双向链表维护元素插入顺序。
- TreeSet通过TreeMap实现的，添加元素到集合时按照比较规则将其插入合适的位置，保证插入后的集合仍然有序。

Map 是一个键值对集合，存储键、值和之间的映射。**Key 无序，唯一**；value 不要求有序，允许重复。Map 没有继承于 Collection 接口，从 Map 集合中检索元素时，只要给出键对象，就会返回对应的值对象。主要实现有TreeMap、HashMap、Hashtable、LinkedHashMap、ConcurrentHashMap

- HashMap：JDK1.8 之前 HashMap 由数组+链表组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的（"拉链法"解决冲突），JDK1.8 以后在解决哈希冲突时有了较大的变化：当某个桶的链表长度 ≥ 8 且哈希表数组长度 ≥ 64 时，才会将该链表转化为红黑树，以减少搜索时间；如果数组长度 < 64，则只会触发扩容而不做树化。
- LinkedHashMap：LinkedHashMap 继承自 HashMap，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，LinkedHashMap 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。
- Hashtable：数组+链表组成的，数组是 Hashtable 的主体，链表则是主要为了解决哈希冲突而存在的
- TreeMap：红黑树（自平衡的排序二叉树）
- ConcurrentHashMap：Node数组+链表+红黑树实现，线程安全的（jdk1.8以前Segment锁，1.8以后volatile + CAS 或者 synchronized）

### java并发集合

java.util.concurrent 包提供的都是线程安全的集合：

**并发Map**

ConcurrentHashMap：它与 Hashtable 的主要区别是二者加锁粒度的不同，在 JDK 1.7，ConcurrentHashMap 加的是分段锁，也就是 Segment 锁，每个 Segment 含有整个 table 的一部分，这样不同分段之间的并发操作就互不影响。在 JDK 1.8，它取消了 Segment，直接在 table 元素（桶的头节点）上加锁，使加锁粒度进一步缩小到单个桶级别。对于 put 操作，如果 Key 对应的数组槽位为 null，则通过 CAS 操作（Compare and Swap）将新节点写入该槽位；如果槽位不为 null（即已存在链表头或红黑树根节点），则对该头节点使用 synchronized 加锁，然后遍历桶中的数据执行替换或新增。如果该 put 操作使得当前桶的链表长度超过阈值，则将其转换为红黑树，从而提高查找效率。
ConcurrentSkipListMap：实现了一个基于SkipList（跳表）算法的可排序的并发集合，SkipList是一种可以在对数预期时间内完成搜索、插入、删除等操作的数据结构，通过维护多个指向其他元素的“跳跃”链接来实现高效查找。

**并发Set**

ConcurrentSkipListSet：是线程安全的有序的集合。底层是使用ConcurrentSkipListMap实现。
CopyOnWriteArraySet：是线程安全的Set实现，它是线程安全的无序的集合，可以将它理解成线程安全的HashSet。有意思的是，CopyOnWriteArraySet和HashSet虽然都继承于共同的父类AbstractSet；但是，HashSet是通过“散列表”实现的，而CopyOnWriteArraySet则是通过“动态数组(CopyOnWriteArrayList)”实现的，并不是散列表。

**并发List**

CopyOnWriteArrayList：它是 ArrayList 的线程安全的变体，其中所有写操作（add，set等）都通过对底层数组进行全新复制来实现，允许存储 null 元素。即当对象进行写操作时，使用了Lock锁做同步处理，内部拷贝了原数组，并在新数组上进行添加操作，最后将新数组替换掉旧数组；若进行的读操作，则直接返回结果，操作过程中不需要进行同步。

**并发 Queue**

ConcurrentLinkedQueue：是一个适用于高并发场景下的队列，它通过无锁的方式(CAS)，实现了高并发状态下的高性能。通常，ConcurrentLinkedQueue 的性能要好于 BlockingQueue 。
BlockingQueue：与 ConcurrentLinkedQueue 的使用场景不同，BlockingQueue 的主要功能并不是在于提升高并发时的队列性能，而在于简化多线程间的数据共享。BlockingQueue 提供一种读写阻塞等待的机制，即如果消费者速度较快，则 BlockingQueue 则可能被清空，此时消费线程再试图从 BlockingQueue 读取数据时就会被阻塞。反之，如果生产线程较快，则 BlockingQueue 可能会被装满，此时，生产线程再试图向 BlockingQueue 队列装入数据时，便会被阻塞等待。

**并发 Deque**

LinkedBlockingDeque：是一个线程安全的双端队列实现。它的内部使用链表结构，每一个节点都维护了一个前驱节点和一个后驱节点。LinkedBlockingDeque 没有进行读写锁的分离，因此同一时间只能有一个线程对其进行操作
ConcurrentLinkedDeque：ConcurrentLinkedDeque是一种基于链接节点的无限并发链表。可以安全地并发执行插入、删除和访问操作。当许多线程同时访问一个公共集合时，ConcurrentLinkedDeque是一个合适的选择。

### 集合的遍历方法

- 普通 for 循环： 可以使用带有索引的普通 for 循环来遍历 List。
- 增强 for 循环（for-each循环）： 用于循环访问数组或集合中的元素。
- Iterator 迭代器： 可以使用迭代器来遍历集合，特别适用于需要删除元素的情况。
- 使用 forEach 方法： Java 8引入了 forEach 方法，可以对集合进行快速遍历。
- Stream API： Java 8的Stream API提供了丰富的功能，可以对集合进行函数式操作，如过滤、映射等。

## 并发面试题