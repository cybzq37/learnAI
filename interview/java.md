### 反射机制

反射通过在运行时读取 class 文件信息，获取类的信息，动态的进行对象的创建、方法调用等操作。

### 注解

注解是一个继承了Annotation的特殊接口，通过Java的反射API Class.getAnnotations()从JVM内存中获取注解对象。

### 值传递和引用传递

值传递传递的是值的副本，引用类型传递 “引用的副本”，通过副本可修改对象内容，但无法改变原引用的指向。

### 重载和重写的区别

重载（Overload）是“同类不同参”，重写（Override）是“子类改父类”

### 深拷贝和浅拷贝的区别

- 浅拷贝（Shallow Copy）只复制对象本身和其内部的基本数据类型，而对于内部的引用类型对象，只复制其内存地址（引用），新旧对象共享该引用对象；
- 深拷贝（Deep Copy）则完全复制整个对象图，包括内部所有引用类型所指向的对象，新旧对象完全独立，互不影响。

### java的异常体系

```text
                Throwable (根类)
                /         \
            Error         Exception (可处理)
            /  \           /             \
   OutOfMemoryError    RuntimeException   (受检异常)
   StackOverflowError      \                 \
                   NullPointerException    IOException
                   ClassCastException      SQLException
                   IndexOutOfBounds        FileNotFoundException
```

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

## 设计模式

### 单例模式

1. 懒汉式
2. 饿汉式
3. 双重检查锁
```java
public class SingleTon {

    // volatile 关键字修饰变量 防止指令重排序
    private static volatile SingleTon instance = null;
    private SingleTon(){}
     
    public static  SingleTon getInstance(){
        if(instance == null){ // 第一次检查（不加锁）
            synchronized(SingleTon.class){
                if(instance == null){ // 第二次检查（加锁）
                    instance = new SingleTon();
                }
            }
        }
        return instance;
    }
}
```

### 代理模式和适配器模式区别

- 适配器模式（Adapter）：解决接口不兼容的问题。它充当“转换器”，让原本因接口不匹配而无法一起工作的两个类能够协同工作。
- 代理模式（Proxy）：解决访问控制的问题。它充当“中间人”，在不改变原接口的前提下，控制对目标对象的访问，或在访问前后添加额外的功能。

### 责任链模式

将多个处理器串成链，请求沿链传递，直到被某个处理器处理或抵达链尾，从而解耦发送者与接收者。

### 策略模式

策略模式定义一组算法并封装成独立策略，让客户端根据需要动态互换，从而将算法定义与使用分离。

## I/O 面试题

### BIO/NIO/AIO 区别

- BIO（blocking IO）：**同步阻塞**，线程调用 read() 后原地等待，直到数据到达内核并拷贝到程序内存，期间线程无法做任何事。
- NIO（non-blocking IO）是 Java 1.4 引入的 java.nio 包，提供了 Channel、Selector、Buffer 等新的抽象，可以构建**多路复用、同步非阻塞** IO 程序，线程发起读请求后立即返回，通过循环不断轮询内核数据是否就绪，期间线程可以处理其他轻量任务。
- AIO（Asynchronous IO）是 Java 1.7 引入的，对 NIO 的扩展，提供了**异步非阻塞**的 IO 操作方式，所以人们叫它 AIO（Asynchronous IO）。异步 IO 是基于事件和回调机制实现的，线程发起读请求后直接去做别的事，等内核把数据完全准备好并拷贝到用户内存后，主动回调通知线程来取。

```java
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;

public class BIOServer {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO Server 启动在 8080");

        while (true) {
            // 【阻塞点 1】: accept() 会一直阻塞，直到有客户端连接
            Socket socket = serverSocket.accept();
            System.out.println("新客户端连接：" + socket.getRemoteSocketAddress());

            // 每个连接新建一个线程处理（为了演示，直接用内部类）
            new Thread(() -> {
                try {
                    byte[] data = new byte[1024];
                    InputStream inputStream = socket.getInputStream();
                    while (true) {
                        // 【阻塞点 2】: read() 会一直阻塞，直到客户端发来数据或关闭连接
                        int len = inputStream.read(data);
                        if (len == -1) break;
                        System.out.println("收到数据：" + new String(data, 0, len));
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;

public class NIOServer {
    public static void main(String[] args) throws Exception {
        // 1. 创建 ServerSocketChannel 并设置为非阻塞
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false); // 【关键】非阻塞
        serverChannel.socket().bind(new InetSocketAddress(8080));

        // 2. 创建 Selector（多路复用器）
        Selector selector = Selector.open();
        // 将 ServerSocketChannel 注册到 Selector，关注 OP_ACCEPT 事件
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO Server 启动在 8080，使用单线程轮询");

        while (true) {
            // 【阻塞点】: select() 会阻塞，直到有感兴趣的事件发生（但可以设置超时）
            // 注意：这个阻塞是空闲时的阻塞，不会因为某个连接不读写就卡死
            selector.select();

            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                keyIterator.remove(); // 必须手动移除，否则重复处理

                if (key.isAcceptable()) {
                    // 有新的连接请求
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel clientChannel = ssc.accept(); // 此时非阻塞，因为已经确认有事件
                    clientChannel.configureBlocking(false);
                    // 将新连接注册到 Selector，关注 OP_READ 事件（可读）
                    clientChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("新连接接入：" + clientChannel.getRemoteAddress());

                } else if (key.isReadable()) {
                    // 有数据可读（不需要轮询等待，内核已经告诉我们可以读了）
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    // 【非阻塞读】: 此时 read 会立即返回，因为 Selector 已经确认有数据
                    int len = clientChannel.read(buffer);
                    if (len == -1) {
                        clientChannel.close();
                    } else {
                        buffer.flip();
                        System.out.println("收到数据：" + new String(buffer.array(), 0, len));
                    }
                }
            }
        }
    }
}
```

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;

public class AIOServer {
    public static void main(String[] args) throws Exception {
        // 创建异步服务端通道
        AsynchronousServerSocketChannel serverChannel = AsynchronousServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        System.out.println("AIO Server 启动在 8080");

        // 接受客户端连接（异步，立即返回）
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel clientChannel, Void attachment) {
                // 当连接成功建立时，此方法被回调
                System.out.println("新连接接入：" + clientChannel.getRemoteAddress());

                // 【重要】必须再次调用 accept() 循环监听下一个连接，否则只接受一个
                serverChannel.accept(null, this);

                // 开始异步读取数据
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                // 【发起异步读】: 第三个参数是回调处理器
                clientChannel.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer attachment) {
                        // 【回调】: 当内核把数据拷贝到 Buffer 后，自动调用此方法
                        attachment.flip();
                        System.out.println("收到数据：" + new String(attachment.array(), 0, result));
                        // 继续异步读取下一条数据（保持长连接监听）
                        clientChannel.read(attachment, attachment, this);
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        System.out.println("读取失败：" + exc.getMessage());
                    }
                });

                System.out.println("【注意】: 读操作已发起，用户线程继续执行，不阻塞！");
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                System.out.println("连接接受失败：" + exc.getMessage());
            }
        });

        // 让主线程不退出（否则进程结束）
        System.out.println("主线程继续执行其他业务逻辑...");
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```


## 集合面试题

### Java中的集合有哪些

Java集合框架根接口是 java.util.Collection（单列集合） 和 java.util.Map（双列集合）。

**Collection体系**

（1）List

- ArrayList：数组实现，查询快（O(1)），增删慢（尤其是中间位置，O(n)）。最常用，默认容量10，扩容1.5倍。
- LinkedList：双向链表实现，增删快（O(1)），查询慢（O(n)）。同时实现了 Deque 接口，可做栈或队列。

> LinkedList 插入比 ArrayList 快的前提是已经持有目标节点的引用，如果要在任意位置插入/删除，仍需先 O(n) 遍历链表找到位置。

（2）Set

- HashSet：底层是 HashMap（只使用 Key），存储无序，允许 null，O(1) 复杂度。判断相等依赖 hashCode() 和 equals()。
- LinkedHashSet：继承 HashSet，底层是 LinkedHashMap，维护插入顺序（链表记录顺序），遍历比 HashSet 慢但有序。
- TreeSet：底层是 红黑树（TreeMap），自动排序（自然排序或定制 Comparator），O(log n)，不允许 null（因需要比较）。

（3）Queue/Deque

- ArrayDeque：循环数组实现，双端队列，比 Stack 和 LinkedList 做队列性能更好，推荐作为栈/队列使用。
- PriorityQueue：二叉堆（数组） 实现，优先级队列，按优先级出队，不是FIFO，不允许 null，O(log n)。
- ConcurrentLinkedQueue：CAS + 链表，高并发下的无界非阻塞队列。

**Map体系**

- HashMap：数组 + 链表 + 红黑树（JDK 8+）。无序，允许一个 null Key。最常用，初始容量16，负载因子0.75，扩容2倍。链表长度 > 8 且数组长度 > 64 时树化。
- LinkedHashMap：继承 HashMap，内部加了一条双向链表维护插入顺序或访问顺序（accessOrder）。常用于实现 LRU 缓存。
- TreeMap：红黑树实现，Key 自动排序（Comparable 或 Comparator），O(log n)，不允许 null Key（需比较）。
- Hashtable：遗留类（JDK 1.0），线程安全（synchronized），不允许 null Key/Value，已被淘汰，替代品是 ConcurrentHashMap。
- Properties：继承 Hashtable，专门用于读取 .properties 配置文件，Key/Value 都是 String。


**JUC包**

- ConcurrentHashMap：分段锁（JDK 7）+ CAS + synchronized（JDK 8），高并发下线程安全且性能极高，面试必问。
- CopyOnWriteArrayList ArrayList	写时复制，写操作加锁，读操作无锁，适合读多写少的场景。
- CopyOnWriteArraySet	HashSet	底层基于 CopyOnWriteArrayList，同上述特性。

## 并发面试题

### JAVA内存模型JMM

JMM 是专门解决多线程并发问题的一套规则。简单说，就是规定了多线程环境下，线程怎么访问共享变量才能不出错，核心是处理 **可见性、原子性、有序性** 这三个问题。

volatile 关键字保证可见性。

synchronized 或者 Lock 锁保证原子性。

volatile 或者 synchronized 就能通过内存屏障阻止这种重排序，保证有序性。

JMM 的核心思路是：定义主内存（大家共享的内存）和工作内存（每个线程自己的缓存），规定变量必须从主内存加载到工作内存才能操作，改完再写回主内存。

### Java多线程

使用 Java 多线程需要注意以下几点：

- 首先是线程安全问题。多个线程同时操作共享数据时，可能出现错误。需要用synchronized关键字、Lock锁等方式，保证同一时间只有一个线程操作共享数据。
- 其次是线程间通信。线程需要协作时，比如一个线程生产数据，另一个线程消费数据，要通过 wait()、notify() 等方法控制，避免出现一方没准备好，另一方就操作的情况，否则可能导致数据错误或线程无限等待。
- 然后是线程的创建和销毁成本。频繁创建和销毁线程会消耗系统资源，影响性能。可以用线程池管理线程，提前创建好一定数量的线程，重复使用，减少资源消耗。

### JAVA中的线程和操作系统的线程区别

要分两种情况回答：平台线程（Platform Thread） 和 虚拟线程（Virtual Thread）

- 平台线程：这是 Java 长期以来默认的线程实现。JVM 在 Linux 上通过 pthread_create 创建线程，一个 Java 平台线程对应一个操作系统线程，是严格的 1:1 线程模型，线程的调度、上下文切换都由操作系统内核负责，开销较大，一个线程默认栈空间约 1MB，所以不能无限创建。
- 虚拟线程：Java 21 正式发布（JEP 444），是 JVM 在用户态自行调度的轻量级线程，采用 M:N 模型，大量虚拟线程会被映射到少量载体平台线程（Carrier Thread）上执行。虚拟线程由 JVM 的 ForkJoinPool 调度，创建成本极低（几百字节起步，可动态扩缩），单 JVM 可以轻松跑上百万个虚拟线程，特别适合 IO 密集型的高并发场景。

### 保证数据的一致性有哪些方案

- 事务管理：使用数据库事务来确保一组数据库操作要么全部成功提交，要么全部失败回滚。通过ACID（原子性、一致性、隔离性、持久性）属性，数据库事务可以保证数据的一致性。
- 锁机制：使用锁来实现对共享资源的互斥访问。在 Java 中，可以使用 synchronized 关键字、ReentrantLock 或其他锁机制来控制并发访问，从而避免并发操作导致数据不一致。
- 版本控制：通过乐观锁的方式，在更新数据时记录数据的版本信息，从而避免同时对同一数据进行修改，进而保证数据的一致性。

### 线程的创建方式有哪些

- 继承Thread类
- 实现Runnable接口
- 实现Callable接口与FutureTask
- 使用线程池（Executor框架）

### 如何停止一个线程

- 异常法停止：线程调用 interrupt() 方法后，在线程的 run 方法中通过 Thread.currentThread().isInterrupted() 判断当前线程的中断状态（注意不要用静态方法 Thread.interrupted()，它会清除中断标志），如果是中断状态则抛出异常，达到中断线程的效果。
- 在沉睡中停止：先将线程sleep，然后调用interrupt标记中断状态，interrupt会将阻塞状态的线程中断。会抛出中断异常，达到停止线程的效果
- stop()暴力停止：线程调用stop()方法会被暴力停止，方法已弃用，该方法会有不好的后果：强制让线程停止有可能使一些请理性的工作得不到完成。
- 使用return停止线程：调用interrupt标记为中断状态后，在run方法中判断当前线程状态，如果为中断状态则return，能达到停止线程的效果。

### block 和 waitting 的区别

- 线程进入BLOCKED状态通常是因为试图获取一个对象的锁（monitor lock），但该锁已经被另一个线程持有。
- 线程进入WAITING状态是因为它正在等待另一个线程执行某些操作，例如调用Object.wait()方法、Thread.join()方法或 LockSupport.park() 方法。在这种状态下，线程将不会消耗CPU资源，并且不会参与锁的竞争。
- 唤醒机制：当一个线程被阻塞等待锁时，一旦锁被释放，线程将有机会重新尝试获取锁。如果锁此时未被其他线程获取，那么线程可以从BLOCKED状态变为RUNNABLE状态。线程在WAITING状态中需要被显式唤醒。例如，如果线程调用了Object.wait()，那么它必须等待另一个线程调用同一对象上的Object.notify()或Object.notifyAll()方法才能被唤醒。

所以，BLOCKED和WAITING两个状态最大的区别有两个：
- BLOCKED 是锁竞争失败后被动触发的状态，WAITING 是人为主动触发的状态
- BLOCKED 的唤醒是自动触发的，而 WAITING 状态必须要通过特定的方法来主动唤醒

### notify 和 notifyAll 的区别

同样是唤醒等待的线程，同样最多只有一个线程能获得锁，同样不能控制哪个线程获得锁。

区别在于：

notify：唤醒一个线程，其他线程依然处于wait的等待唤醒状态，如果被唤醒的线程结束时没调用notify，其他线程就永远没人去唤醒，只能等待超时，或者被中断
notifyAll：所有线程退出wait的状态，开始竞争锁，但只有一个线程能抢到，这个线程执行完后，其他线程又会有一个幸运儿脱颖而出得到锁

### 线程间的通信方式有哪些

Object 类的 wait()、notify() 和 notifyAll() 方法。这是 Java 中最基础的线程间通信方式，基于对象的监视器（锁）机制。

- wait()：使当前线程进入等待状态，直到其他线程调用该对象的 notify() 或 notifyAll() 方法。
- notify()：唤醒在此对象监视器上等待的单个线程。
- notifyAll()：唤醒在此对象监视器上等待的所有线程。

Lock 和 Condition 接口。Lock 接口提供了比 synchronized 更灵活的锁机制，Condition 接口则配合 Lock 实现线程间的等待 / 通知机制。

- await()：使当前线程进入等待状态，直到被其他线程唤醒。
- signal()：唤醒一个等待在该 Condition 上的线程。
- signalAll()：唤醒所有等待在该 Condition 上的线程。

volatile 关键字。volatile 关键字用于保证变量的可见性，即当一个变量被声明为 volatile 时，它会保证对该变量的写操作会立即刷新到主内存中，而读操作会从主内存中读取最新的值。

CountDownLatch 是一个同步辅助类，它允许一个或多个线程等待其他线程完成操作。

- CountDownLatch(int count)：构造函数，指定需要等待的线程数量。
- countDown()：减少计数器的值。
- await()：使当前线程等待，直到计数器的值为 0。

### JUC包

线程池相关：

- ThreadPoolExecutor：最核心的线程池类，用于创建和管理线程池。通过它可以灵活地配置线程池的参数，如核心线程数、最大线程数、任务队列等，以满足不同的并发处理需求。
- Executors：线程池工厂类，提供了一系列静态方法来创建不同类型的线程池，如newFixedThreadPool（创建固定线程数的线程池）、newCachedThreadPool（创建可缓存线程池）、newSingleThreadExecutor（创建单线程线程池）等，方便开发者快速创建线程池。

并发集合类：

- ConcurrentHashMap：线程安全的哈希映射表，用于在多线程环境下高效地存储和访问键值对。JDK 1.7 采用分段锁（Segment）实现；JDK 1.8 起已废弃分段锁，改为基于 CAS + synchronized 锁桶头节点的方式，锁粒度更细，并发度更高，在高并发场景下比传统的 Hashtable 性能更好。
- CopyOnWriteArrayList：线程安全的列表，在对列表进行修改操作时，会创建一个新的底层数组，将修改操作应用到新数组上，而读操作仍然可以在旧数组上进行，从而实现了读写分离，提高了并发读的性能，适用于读多写少的场景。

同步工具类：

- CountDownLatch：允许一个或多个线程等待其他一组线程完成操作后再继续执行。它通过一个计数器来实现，计数器初始化为线程的数量，每个线程完成任务后调用countDown方法将计数器减一，当计数器为零时，等待的线程可以继续执行。常用于多个线程完成各自任务后，再进行汇总或下一步操作的场景。
- CyclicBarrier：让一组线程互相等待，直到所有线程都到达某个屏障点后，再一起继续执行。与CountDownLatch不同的是，CyclicBarrier可以重复使用，当所有线程都通过屏障后，计数器会重置，可以再次用于下一轮的等待。适用于多个线程需要协同工作，在某个阶段完成后再一起进入下一个阶段的场景。
- Semaphore：信号量，用于控制同时访问某个资源的线程数量。它维护了一个许可计数器，线程在访问资源前需要获取许可，如果有可用许可，则获取成功并将许可计数器减一，否则线程需要等待，直到有其他线程释放许可。常用于控制对有限资源的访问，如数据库连接池、线程池中的线程数量等。

原子类：

- AtomicInteger：原子整数类，提供了对整数类型的原子操作，如自增、自减、比较并交换等。通过硬件级别的原子指令来保证操作的原子性和线程安全性，避免了使用锁带来的性能开销，在多线程环境下对整数进行计数、状态标记等操作非常方便。
- AtomicReference：原子引用类，用于对对象引用进行原子操作。可以保证在多线程环境下，对对象的更新操作是原子性的，即要么全部成功，要么全部失败，不会出现数据不一致的情况。常用于实现无锁数据结构或需要对对象进行原子更新的场景。

### AQS

QS全称为AbstractQueuedSynchronizer，是Java中的一个抽象类。 AQS是一个用于构建锁、同步器、协作工具类的工具类（框架）。

AQS核心思想是，如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。

### 乐观锁与悲观锁

乐观锁认为没人与它竞争，悲观锁认为总有人与它竞争。

- 悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。
- 乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

### CAS自旋锁

CAS全称 Compare And Swap（比较与交换），是一种无锁算法。在不使用锁（没有线程被阻塞）的情况下实现多线程之间的变量同步。java.util.concurrent包中的原子类就是通过CAS来实现了乐观锁。

CAS算法涉及到三个操作数：

- 需要读写的内存值 V
- 进行比较的值 A
- 要写入的新值 B

当且仅当 V 的值等于 A 时，CAS通过原子方式用新值B来更新V的值（“比较+更新”整体是一个原子操作），否则不会执行任何操作。一般情况下，“更新”是一个不断重试的操作。自旋锁避免了线程切换的开销，但是需要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。

CAS虽然很高效，但是它也存在三大问题，这里也简单说一下：

- ABA问题。CAS需要在操作值的时候检查内存值是否发生变化，没有发生变化才会更新内存值。但是如果内存值原来是A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。ABA问题的解决思路就是在变量前面添加版本号，每次变量更新的时候都把版本号加一，这样变化过程就从“A－B－A”变成了“1A－2B－3A”。
- JDK从1.5开始提供了AtomicStampedReference类来解决ABA问题，具体操作封装在compareAndSet()中。compareAndSet()首先检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，则以原子方式将引用值和标志的值设置为给定的更新值。
- 循环时间长开销大。CAS操作如果长时间不成功，会导致其一直自旋，给CPU带来非常大的开销。
- 只能保证一个共享变量的原子操作。对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。


### 无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁

这四种锁是指锁的状态，专门针对synchronized的，synchronized是悲观锁，在操作同步资源之前需要给同步资源先加锁，这把锁就是存在Java对象头里的。

- 无锁：没有对资源进行锁定，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。无锁的特点就是修改操作在循环内进行，线程会不断的尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。如果有多个线程修改同一个值，必定会有一个线程能修改成功，而其他修改失败的线程会不断重试直到修改成功。上面我们介绍的CAS原理及应用即是无锁的实现。无锁无法全面代替有锁，但无锁在某些场合下的性能是非常高的。

- 偏向锁：指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。在大多数情况下，锁总是由同一线程多次获得，不存在多线程竞争，所以出现了偏向锁。其目标就是在只有一个线程执行同步代码块时能够提高性能。

- 轻量级锁：当锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。

- 重量级锁：等待锁的线程都会进入阻塞状态。


### 公平锁 VS 非公平锁

- 公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

- 非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。



### 线程池

手动创建 ThreadPoolExecutor 需要指定 7 个核心参数，理解这些参数才能用好。比如创建一个适合处理 IO 密集型任务的线程池（IO 密集型任务线程数可以多些，一般是 CPU 核心数的 2 倍）：

```java
ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
    corePoolSize, // 核心线程数 线程池长期维持的最小线程数
    corePoolSize * 2, // 最大线程数 线程池能容纳的最多线程数
    60L, // 空闲线程存活时间 超过核心线程数的空闲线程 多久后销毁
    TimeUnit.SECONDS, // 存活时间单位
    new ArrayBlockingQueue<>(100), // 任务阻塞队列 核心线程忙时 新任务存这里
    Executors.defaultThreadFactory(), // 线程创建工厂 用于设置线程名 优先级等
    new ThreadPoolExecutor.AbortPolicy() // 拒绝策略 队列满且线程数达最大时 如何处理新任务
);
```

比如拒绝策略有四种，除了默认的 AbortPolicy（直接抛异常），还有 CallerRunsPolicy（让提交任务的主线程自己执行，缓解压力）、DiscardPolicy（直接丢弃新任务）、DiscardOldestPolicy（丢弃队列里最旧的任务，再提交新任务），要根据业务选择，比如核心业务不能丢任务，就别用 Discard 相关策略。

提交任务时，submit 和 execute 的区别在于 submit 能提交 Callable 有返回值，还能通过 Future 捕获任务执行中的异常，而 execute 只能提交 Runnable，异常会直接抛出，比如：

```java
// submit捕获异常
Future<?> future = threadPool.submit(() -> {
    int i = 1 / 0; // 模拟异常
});
try {
    future.get();
} catch (ExecutionException e) {
    System.out.println("任务执行异常: " + e.getCause()); // 捕获到算术异常
}
```

还有线程池的使用原则：不能创建后不关闭，否则会导致线程泄露，JVM 无法退出；任务队列的容量要合理设置，太大可能导致内存溢出，太小容易触发拒绝策略；线程数要根据任务类型调整，CPU 密集型任务（比如复杂计算）线程数不宜过多，一般和 CPU 核心数相当，避免线程切换开销，IO 密集型任务可以多设些线程，因为线程大部分时间在等待 IO 完成。

ScheduledThreadPool：可以设置定期的执行任务，它支持定时或周期性执行任务，比如每隔 10 秒钟执行一次任务，我通过这个实现类设置定期执行任务的策略。
FixedThreadPool：它的核心线程数和最大线程数是一样的，所以可以把它看作是固定线程数的线程池，它的特点是线程池中的线程数除了初始阶段需要从 0 开始增加外，之后的线程数量就是固定的，就算任务数超过线程数，线程池也不会再创建更多的线程来处理任务，而是会把超出线程处理能力的任务放到任务队列中进行等待。需要特别注意的是：它使用的是无界的 LinkedBlockingQueue（容量 Integer.MAX_VALUE），在任务消费速度跟不上生产速度时，队列会无限堆积，最终可能导致 OOM——这也是阿里手册禁止直接使用 Executors.newFixedThreadPool() 的主要原因。
CachedThreadPool：可以称作可缓存线程池，它的特点在于线程数理论上没有上限（maximumPoolSize 被设置为 Integer.MAX_VALUE），当线程闲置 60 秒后会被回收。它使用 SynchronousQueue 作为工作队列，容量为 0，只负责对任务进行中转和传递，每来一个任务若无空闲线程就会立即创建新线程。在高并发瞬时大量任务提交的场景下，CachedThreadPool 会快速创建成百上千的线程，很可能直接把系统资源耗尽导致 OOM，这也是阿里手册禁止使用它的核心原因，生产环境请手动 new ThreadPoolExecutor 并显式约束最大线程数。
SingleThreadExecutor：它会使用唯一的线程去执行任务，原理和 FixedThreadPool 是一样的，只不过这里线程只有一个，如果线程在执行任务的过程中发生异常，线程池也会重新创建一个线程来执行后续的任务。这种线程池由于只有一个线程，所以非常适合用于所有任务都需要按被提交的顺序依次执行的场景，而前几种线程池不一定能够保障任务的执行顺序等于被提交的顺序，因为它们是多线程并行执行的。
SingleThreadScheduledExecutor：它实际和 ScheduledThreadPool 线程池非常相似，它只是 ScheduledThreadPool 的一个特例，内部只有一个线程。

## JVM虚拟机

### JVM内存模型

JVM 运行时内存共分为 **虚拟机栈、堆、元空间、程序计数器、本地方法栈** 五个部分。还有一部分内存叫直接内存，属于操作系统的本地内存，也是可以直接操作的。

JVM的内存结构主要分为以下几个部分：

- **堆**：是 JVM 中最大的一块内存区域，被所有线程共享，在虚拟机启动时创建，用于存放对象实例。从内存回收角度，堆被划分为新生代和老年代，新生代又分为 Eden 区和两个 Survivor 区（From Survivor 和 To Survivor）。如果在堆中没有内存完成实例分配，并且堆也无法扩展时会抛出 OutOfMemoryError 异常。
- **虚拟机栈**：每个线程都有自己独立的 Java 虚拟机栈，生命周期与线程相同。每个方法在执行时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息。可能会抛出 StackOverflowError 和 OutOfMemoryError 异常。
- **本地方法栈**：与 Java 虚拟机栈类似，主要为虚拟机使用到的 Native 方法服务，在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。本地方法执行时也会创建栈帧，同样可能出现 StackOverflowError 和 OutOfMemoryError 两种错误。
- **程序计数器**：可以看作是当前线程所执行的字节码的行号指示器，用于存储当前线程正在执行的 Java 方法的 JVM 指令地址，是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域，生命周期与线程相同。
- **元空间**：在 JDK 1.8 及以后的版本中，方法区被元空间取代，使用本地内存，用于存储已被虚拟机加载的类信息、常量、静态变量等数据。虽然方法区被描述为堆的逻辑部分，但有 “非堆” 的别名。方法区可以选择不实现垃圾收集，内存不足时会抛出 OutOfMemoryError 异常。

- 运行时常量池：是方法区的一部分，用于存放编译期生成的各种字面量和符号引用，具有动态性，运行时也可将新的常量放入池中。当无法申请到足够内存时，会抛出 OutOfMemoryError 异常。
- 直接内存：不属于 JVM 运行时数据区的一部分，通过 NIO 类引入，是一种堆外内存，可以显著提高 I/O 性能。直接内存的使用受到本机总内存的限制，若分配不当，可能导致 OutOfMemoryError 异常。

栈中存储的不是对象，而是对象的引用。

### 堆分为哪几部分

- **新生代**：新生代分为 Eden Space 和 Survivor Space。Eden 区是新生代中最大的区域（默认 Eden:S0:S1 = 8:1:1），大多数新创建的对象首先存放在这里。当 Eden 区满时，会触发一次 Minor GC（新生代垃圾回收）。在Survivor Spaces中，通常分为两个相等大小的区域，称为S0（Survivor 0）和S1（Survivor 1）。在每次Minor GC后，存活下来的对象会被移动到其中一个Survivor空间，以继续它们的生命周期。这两个区域轮流充当对象的中转站，帮助区分短暂存活的对象和长期存活的对象。
- **老年代**: 存放过一次或多次Minor GC仍存活的对象会被移动到老年代。老年代中的对象生命周期较长，因此Major GC（也称为Full GC，涉及老年代的垃圾回收）发生的频率相对较低，但其执行时间通常比Minor GC长。老年代的空间通常比新生代大，以存储更多的长期存活对象。
- **元空间**：从Java 8开始，永久代（Permanent Generation）被元空间取代，用于存储类的元数据信息，如类的结构信息（如字段、方法信息等）。元空间并不在Java堆中，而是使用本地内存，这解决了永久代容易出现的内存溢出问题。
- 大对象：在 G1 垃圾收集器中，任何超过 Region 一半大小的对象都会被认定为 Humongous Object，直接分配在一组连续的 Humongous Region 中；这些 Region 在 G1 的逻辑上属于老年代的一部分（但有独立的分配策略），避免大对象在年轻代频繁被复制移动而带来的开销。传统的分代 GC（如 Parallel / CMS）中，超过 -XX:PretenureSizeThreshold 的大对象也会直接分配到老年代，原因同样是避免在 Eden 和 Survivor 之间反复复制。


### 强软弱虚

- 强引用：永不回收（OOM也不回收）
- 软引用：内存不足时回收（OOM之前）
- 弱引用：下一次GC时 (无论内存是否够) ThreadLocal、WeakHashMap
- 虚引用：无法通过它获取对象，用于追踪内存回收、堆外内存释放

```java
Object obj = new Object(); // 这就是强引用
obj = null; // 手动置空，GC才有可能回收
```

```java
// 模拟大对象
Object heavyObj = new Object();
SoftReference<Object> softRef = new SoftReference<>(heavyObj);

heavyObj = null; // 去除强引用

// 当内存不足时，softRef.get() 会返回 null
Object cachedObj = softRef.get(); 
if (cachedObj == null) {
    // 重新构建对象（因为已经被回收了）
}
```

```java
Object obj = new Object();
WeakReference<Object> weakRef = new WeakReference<>(obj);

obj = null; // 去掉强引用
// 此时调用 System.gc()，weakRef.get() 极大概率返回 null
```

### ThreadLocal

它的核心作用用一句话概括就是：让每个线程拥有自己独立的变量副本，互不干扰。

ThreadLocalMap的Key是弱引用（WeakReference），但Value是强引用，当外部不再引用ThreadLocal对象时（比如方法执行完），Key会在下次GC时被回收，变成null。但Value由于是强引用，且链是 Thread -> ThreadLocalMap -> Entry(value)，如果当前线程一直存活（比如Web应用中的线程池），那么过时的Value就永远无法被回收，导致内存泄漏。
**最佳实践**：每次使用完ThreadLocal，务必在finally块中调用remove()方法！

### 内存泄漏与内存溢出

内存泄露：内存泄漏是指程序在运行过程中不再使用的对象仍然被引用，而无法被垃圾收集器回收，从而导致可用内存逐渐减少。虽然在Java中，垃圾回收机制会自动回收不再使用的对象，但如果有对象仍被不再使用的引用持有，垃圾收集器无法回收这些内存，最终可能导致程序的内存使用不断增加。

内存泄露常见原因：

静态集合：使用静态数据结构（如HashMap或ArrayList）存储对象，且未清理。
事件监听：未取消对事件源的监听，导致对象持续被引用。
线程：未停止的线程可能持有对象引用，无法被回收。
内存溢出：内存溢出是指Java虚拟机（JVM）在申请内存时，无法找到足够的内存，最终引发OutOfMemoryError。这通常发生在堆内存不足以存放新创建的对象时。

内存溢出常见原因：

大量对象创建：程序中不断创建大量对象，超出JVM堆的限制。
持久引用：大型数据结构（如缓存、集合等）长时间持有对象引用，导致内存累积。
线程过多：每个线程都需要独立的栈空间，线程数过多时申请栈内存失败可能抛出 OutOfMemoryError: unable to create new native thread（注意：深度递归触发的是 StackOverflowError，并不属于 OOM，二者是不同的 Error）。

### 类初始化和类加载

在Java中创建对象的过程包括以下几个步骤：

- 类加载检查：虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
- 分配内存：在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。
- 初始化零值：内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。
- 进行必要设置，比如对象头：初始化零值完成之后，虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。这些信息存放在对象头中。另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。
- 执行 init 方法：在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始——构造函数，即class文件中的方法还没有执行，所有的字段都还为零，对象需要的其他资源和状态信息还没有按照预定的意图构造好。所以一般来说，执行 new 指令之后会接着执行方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全被构造出来。

### 对象的生命周期

创建：对象通过关键字new在堆内存中被实例化，构造函数被调用，对象的内存空间被分配。
使用：对象被引用并执行相应的操作，可以通过引用访问对象的属性和方法，在程序运行过程中被不断使用。
销毁：当对象不再被引用时，通过垃圾回收机制自动回收对象所占用的内存空间。垃圾回收器会在适当的时候检测并回收不再被引用的对象，释放对象占用的内存空间，完成对象的销毁过程。

### 类加载器

启动类加载器（Bootstrap Class Loader）：这是最顶层的类加载器，负责加载 Java 的核心类库。在 Java 8 及之前加载 jre/lib/rt.jar 中的类，从 Java 9 起 rt.jar 已被 JEP 220 移除，核心类库存放在 $JAVA_HOME/lib/modules 的模块化运行时镜像中（如 java.base 模块）。它由 C++ 编写，是 JVM 的一部分，在 Java 层面没有对应的 ClassLoader 对象（通过 getClassLoader() 获取时返回 null），无法被 Java 程序直接引用。
平台类加载器 / 扩展类加载器：
Java 8 及以前称为 Extension Class Loader（扩展类加载器），负责加载 jre/lib/ext 或由 java.ext.dirs 系统属性指定目录下的 jar 包和类库。
Java 9 起通过 JEP 220 替换为 Platform Class Loader（平台类加载器），jre/lib/ext 目录和 java.ext.dirs 属性都已被移除。平台类加载器负责加载 JDK 中一些除核心模块外的平台类（如 java.sql、java.xml 等）。
在 Java 层面，它的 parent 字段实际为 null，但在委派逻辑上仍会先交给 Bootstrap ClassLoader 处理。
应用程序类加载器（Application Class Loader，也叫 System Class Loader）：负责加载用户类路径（ClassPath）和模块路径上的类，是开发者平时默认使用的类加载器。可以通过 ClassLoader.getSystemClassLoader() 获取。它的父加载器是 Platform Class Loader（Java 8 下为 Extension Class Loader）。
自定义类加载器（Custom Class Loader）：开发者可以根据需求定制类的加载方式，比如从网络加载 class 文件、数据库、甚至是加密的文件中加载类等。自定义类加载器可以用来扩展 Java 应用程序的灵活性和安全性，是 Java 动态性的一个重要体现。
这些类加载器之间的关系形成了双亲委派模型，其核心思想是当一个类加载器收到类加载的请求时，首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中。

只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

### 类加载过程

- 加载：通过类的全限定名（包名 + 类名），获取到该类的.class文件的二进制字节流，将二进制字节流所代表的静态存储结构，转化为方法区运行时的数据结构，在内存中生成一个代表该类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口
- 连接：验证、准备、解析 3 个阶段统称为连接。
    - 验证：确保class文件中的字节流包含的信息，符合当前虚拟机的要求，保证这个被加载的class类的正确性，不会危害到虚拟机的安全。验证阶段大致会完成以下四个阶段的检验动作：文件格式校验、元数据验证、字节码验证、符号引用验证
    - 准备：为类中的静态字段分配内存，并设置默认的初始值，比如int类型初始值是0。被final修饰的static字段不会设置，因为final在编译的时候就分配了
    - 解析：解析阶段是虚拟机将常量池的「符号引用」直接替换为「直接引用」的过程。符号引用是以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用的时候可以无歧义地定位到目标即可。直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄，直接引用是和虚拟机实现的内存布局相关的。如果有了直接引用， 那引用的目标必定已经存在在内存中了。
- 初始化：初始化是整个类加载过程的最后一个阶段，初始化阶段简单来说就是执行类的类构造器方法 <clinit>()，要注意的是这里的 <clinit>() 并不是开发者写的构造函数（那个是实例构造器 <init>()），而是编译器自动收集类中所有静态变量的赋值语句和静态代码块合并生成的。
- 使用：使用类或者创建对象
- 卸载：一个类要被JVM卸载，条件非常苛刻，需要同时满足以下三点：
    - 该类所有的实例都已经被回收：这是最显而易见的前提。如果堆中还存在这个类的任何一个实例对象，那么定义这个对象的Class对象肯定不能被卸载。
    - 加载该类的ClassLoader已经被回收：这是最关键也是最难满足的条件。类与其加载器是双向绑定的共生关系。一个类由哪个类加载器加载，这个信息是存储在Class对象里的。要卸载一个类，必须先卸载加载它的类加载器。
    - 类对应的java.lang.Class对象没有任何地方被引用：不能在任何地方通过反射（如静态字段、全局变量）、静态变量、JNI等途径引用到这个Class对象。一旦这个Class对象还存在强引用，GC就不会回收它，那么这个类也就不会被卸载。

### 垃圾回收

垃圾回收（Garbage Collection, GC）是自动管理内存的一种机制，它负责自动释放不再被程序引用的对象所占用的内存，这种机制减少了内存泄漏和内存管理错误的可能性。垃圾回收可以通过多种方式触发，具体如下：

- 内存不足时：当JVM检测到堆内存不足，无法为新的对象分配内存时，会自动触发垃圾回收。
- 手动请求：虽然垃圾回收是自动的，开发者可以通过调用 System.gc() 或 Runtime.getRuntime().gc() 建议 JVM 进行垃圾回收。不过这只是一个建议，并不能保证立即执行。
- JVM参数：启动 Java 应用时可以通过 JVM 参数来调整垃圾回收的行为，比如：-Xmx（最大堆大小）、-Xms（初始堆大小）等。
- 分代触发条件：不同区域达到各自的触发条件时也会引发 GC。例如 Eden 区空间不足时触发 Minor GC；老年代空间不足、或 Minor GC 后晋升对象无法放入老年代时触发 Major GC / Full GC；元空间/方法区（Metaspace）达到阈值时也会触发 Full GC；在 G1 中，当堆占用率达到 -XX:InitiatingHeapOccupancyPercent（默认 45%）时会启动并发标记周期。

### 垃圾判断方法

在Java中，判断对象是否为垃圾（即不再被使用，可以被垃圾回收器回收）主要依据两种主流的垃圾回收算法来实现：引用计数法和可达性分析算法。

引用计数法（Reference Counting）

- 原理：为每个对象分配一个引用计数器，每当有一个地方引用它时，计数器加1；当引用失效时，计数器减1。当计数器为0时，表示对象不再被任何变量引用，可以被回收。
- 缺点：不能解决循环引用的问题，即两个对象相互引用，但不再被其他任何对象引用，这时引用计数器不会为0，导致对象无法被回收。

可达性分析算法（Reachability Analysis）

从一组称为 GC Roots（垃圾收集根）的对象出发，向下追溯它们引用的对象，以及这些对象引用的其他对象，以此类推。如果一个对象到 GC Roots 没有任何引用链相连（即从 GC Roots 到这个对象不可达），那么这个对象就被认为是不可达的，可以被回收。GC Roots 对象主要包括：
虚拟机栈（栈帧中的本地变量表）中引用的对象；
方法区中类静态属性引用的对象；
方法区中常量引用的对象（例如字符串常量池里的引用）；
本地方法栈中 JNI（Java Native Interface）引用的对象；
所有被同步锁（synchronized）持有的对象；
反映 JVM 内部情况的 JNIHandles 全局引用（如基本数据类型对应的 Class 对象、常驻异常对象、系统类加载器等）。

### 垃圾回收方法

- 标记-清除算法：标记-清除算法分为“标记”和“清除”两个阶段，首先通过可达性分析，标记出所有需要回收的对象，然后统一回收所有被标记的对象。标记-清除算法有两个缺陷，一个是效率问题，标记和清除的过程效率都不高，另外一个就是，清除结束后会造成大量的碎片空间。有可能会造成在申请大块内存的时候因为没有足够的连续空间导致再次 GC。
- 复制算法：为了解决碎片空间的问题，出现了“复制算法”。复制算法的原理是，将内存分成两块，每次申请内存时都使用其中的一块，当内存不够时，将这一块内存中所有存活的对象复制到另一块上，然后再把已使用的内存整个清理掉。复制算法解决了空间碎片的问题。但是也带来了新的问题。因为每次在申请内存时，都只能使用一半的内存空间。内存利用率严重不足。
- 标记-整理算法：复制算法在 GC 之后存活对象较少的情况下效率比较高，但如果存活对象比较多时，会执行较多的复制操作，效率就会下降。而老年代的对象在 GC 之后的存活率就比较高，所以就有人提出了“标记-整理算法”。标记-整理算法的“标记”过程与“标记-清除算法”的标记过程一致，但标记之后不会直接清理。而是将所有存活对象都移动到内存的一端。移动结束后直接清理掉剩余部分。
- 分代回收算法：分代收集是将内存划分成了新生代和老年代。分配的依据是对象的生存周期，或者说经历过的 GC 次数。对象创建时，一般在新生代申请内存，当经历一次 GC 之后如果对还存活，那么对象的年龄 +1。当年龄超过一定值(默认是 15，可以通过参数 -XX:MaxTenuringThreshold 来设定)后，如果对象还存活，那么该对象会进入老年代。

### 垃圾回收器

- Serial 收集器（复制算法）：新生代单线程收集器，标记和清理都是单线程，优点是简单高效，适合客户端模式或单核环境。
- ParNew 收集器（复制算法）：新生代并行收集器，实际上是 Serial 的多线程版本，历史上主要用来与 CMS 配合使用。注意：ParNew 在 JDK 9 已被 deprecated，JDK 14 随 CMS 移除后基本退出历史舞台。
- Parallel Scavenge 收集器（复制算法）：新生代并行收集器，追求高吞吐量（吞吐量 = 用户线程时间 /（用户线程时间 + GC 线程时间））。适合后台计算等对交互响应要求不高的场景。
- Serial Old 收集器（标记-整理算法）：老年代单线程收集器，Serial 的老年代版本。
- Parallel Old 收集器（标记-整理算法）：老年代并行收集器，吞吐量优先，Parallel Scavenge 的老年代版本。
- CMS（Concurrent Mark Sweep）收集器（标记-清除算法）：老年代低延迟收集器，目标是最短回收停顿时间，大部分阶段与用户线程并发执行。CMS 已在 JDK 9（JEP 291）被 deprecated，JDK 14（JEP 363）正式从 HotSpot 中移除，线上 JDK 14+ 已无法再使用 CMS，下面章节对 CMS 的讨论仅作为历史知识点。
- G1（Garbage First）收集器（整体基于标记-整理，局部采用复制）：JDK 7 引入，JDK 9 起成为服务端默认 GC，面向大堆、兼顾吞吐与停顿时间。G1 将整个堆划分成若干 Region，弱化了传统分代边界，每次只选择回收收益最高的若干 Region（Collection Set），不会产生物理意义上的内存碎片。
- ZGC（Z Garbage Collector）：JDK 11 作为实验特性引入，JDK 15 正式可用，JDK 21（JEP 439）引入分代 ZGC。基于染色指针 + Load Barrier，停顿时间可稳定控制在 1ms 以内（与堆大小无关），适用于超大堆（TB 级）和低延迟场景。
- Shenandoah 收集器：由 Red Hat 开发，JDK 12 引入、JDK 15 正式。和 ZGC 类似同样追求低停顿，采用 Brooks Pointer 实现并发整理，也适合大堆低延迟场景。

### minorGC、majorGC、fullGC的区别

垃圾回收机制是自动管理内存的重要组成部分。根据其作用范围和触发条件的不同，可以将GC分为三种类型：Minor GC（也称为Young GC）、Major GC（有时也称为Old GC）、以及Full GC。以下是这三种GC的区别和触发场景：

Minor GC (Young GC)

作用范围：只针对年轻代进行回收，包括Eden区和两个Survivor区（S0和S1）。
触发条件：当Eden区空间不足时，JVM会触发一次Minor GC，将Eden区和一个Survivor区中的存活对象移动到另一个Survivor区或老年代（Old Generation）。
特点：通常发生得非常频繁，因为年轻代中对象的生命周期较短，回收效率高，暂停时间相对较短。
Major GC

作用范围：通常指仅回收老年代的 GC（例如 CMS 的老年代 GC、G1 的 Mixed GC 在部分文献里也归为此类）；要同时回收新生代 + 老年代 + 方法区，那是 Full GC 的范畴。
触发条件：当老年代空间不足时，或者系统检测到年轻代对象晋升到老年代的速度过快，可能会触发Major GC。
特点：相比Minor GC，Major GC发生的频率较低，但每次回收可能需要更长的时间，因为老年代中的对象存活率较高。
Full GC

作用范围：对整个 Java 堆（年轻代 + 老年代）进行回收，并且通常会伴随方法区/元空间的类卸载（是否真正回收元空间取决于 GC 器的实现与触发条件）。需要注意：元空间使用本地内存（Native Memory），并不在堆内，不能简单说"Full GC 会回收元空间"，更准确的说法是 Full GC 期间 JVM 可能对元空间中无用的类元数据进行卸载。

触发条件：

直接调用System.gc()或Runtime.getRuntime().gc()方法时，虽然不能保证立即执行，但JVM会尝试执行Full GC。

空间分配担保失败：Minor GC 前 JVM 会先检查老年代连续空间是否大于"新生代所有对象之和"或"历次晋升对象的平均大小"，如果都不满足则触发 Full GC；Minor GC 后存活对象无法全部放入老年代时也会触发 Full GC，对整个堆内存进行回收。

当永久代（Java 8 之前的版本）或元空间（Java 8 及以后的版本）空间不足时，JVM 会在抛 OOM 前先尝试 Full GC，借机卸载无用类、回收元空间。

CMS 在并发收集过程中出现 Concurrent Mode Failure（老年代被并发标记/清理速度跟不上分配速度时填满）或 Promotion Failed（晋升时老年代连续空间不足），都会回退为 Full GC（Serial Old）。

特点：Full GC是最昂贵的操作，因为它需要停止所有的工作线程（Stop The World），遍历整个堆内存来查找和回收不再使用的对象，因此应尽量减少Full GC的触发。

## Spring面试题

Spring框架核心特性包括：

IoC容器：Spring通过控制反转实现了对象的创建和对象间的依赖关系管理。开发者只需要定义好Bean及其依赖关系，Spring容器负责创建和组装这些对象。
AOP：面向切面编程，允许开发者定义横切关注点，例如事务管理、安全控制等，独立于业务逻辑的代码。通过AOP，可以将这些关注点模块化，提高代码的可维护性和可重用性。
事务管理：Spring提供了一致的事务管理接口，支持声明式和编程式事务。开发者可以轻松地进行事务管理，而无需关心具体的事务API。
MVC框架：Spring MVC是一个基于Servlet API构建的Web框架，采用了模型-视图-控制器（MVC）架构。它支持灵活的URL到页面控制器的映射，以及多种视图技术

### IoC 与 AOP

Spring IoC和AOP 区别：

IoC：即控制反转的意思，它是一种创建和获取对象的技术思想，依赖注入(DI)是实现这种技术的一种方式。传统开发过程中，我们需要通过new关键字来创建对象。使用IoC思想开发方式的话，我们不通过new关键字创建对象，而是通过IoC容器来帮我们实例化对象。 通过IoC的方式，可以大大降低对象之间的耦合度。
AOP：是面向切面编程，能够将那些与业务无关，却为业务模块所共同调用的逻辑封装起来，以减少系统的重复代码，降低模块间的耦合度。Spring AOP 基于动态代理：如果被代理对象实现了接口，Spring AOP 默认会使用 JDK Proxy 创建代理对象；如果没有实现接口，则会使用 CGLIB 生成被代理对象的子类作为代理。注意：自 Spring Boot 2.0 起，默认配置 spring.aop.proxy-target-class=true，即无论是否实现接口都优先使用 CGLIB，如需切回 JDK 动态代理需手动设为 false。
在 Spring 框架中，IOC 和 AOP 结合使用，可以更好地实现代码的模块化和分层管理。例如：

通过 IOC 容器管理对象的依赖关系，然后通过 AOP 将横切关注点统一切入到需要的业务逻辑中。
使用 IOC 容器管理 Service 层和 DAO 层的依赖关系，然后通过 AOP 在 Service 层实现事务管理、日志记录等横切功能，使得业务逻辑更加清晰和可维护。

### AOP实现原理

Spring AOP的实现依赖于动态代理技术。动态代理是在运行时动态生成代理对象，而不是在编译时。它允许开发者在运行时指定要代理的接口和行为，从而实现在不修改源码的情况下增强方法的功能。

Spring AOP支持两种动态代理：

- 基于JDK的动态代理：使用java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口实现。这种方式需要代理的类实现一个或多个接口。
- 基于CGLIB的动态代理：当被代理的类没有实现接口时，Spring会使用CGLIB库生成一个被代理类的子类作为代理。CGLIB（Code Generation Library）是一个第三方代码生成库，通过继承方式实现代理。

### 动态代理是什么

Java的动态代理是一种在运行时动态创建代理对象的机制，主要用于在不修改原始类的情况下对方法调用进行拦截和增强。

Java动态代理主要分为两种类型：

- 基于接口的代理（JDK动态代理）： 这种类型的代理要求目标对象必须实现至少一个接口。Java动态代理会创建一个实现了相同接口的代理类，然后在运行时动态生成该类的实例。这种代理的实现核心是java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口。每一个动态代理类都必须实现InvocationHandler接口，并且每个代理类的实例都关联到一个handler。当通过代理对象调用一个方法时，这个方法的调用会被转发为由InvocationHandler接口的invoke()方法来进行调用。

- 基于类的代理（CGLIB动态代理）： CGLIB（Code Generation Library）是一个强大的高性能的代码生成库，它可以在运行时动态生成一个目标类的子类。CGLIB代理不需要目标类实现接口，而是通过继承的方式创建代理类。因此，如果目标对象没有实现任何接口，可以使用CGLIB来创建动态代理。

### 动态代理和静态代理的区别

代理是一种常用的设计模式，目的是：为其他对象提供一个代理以控制对某个对象的访问，将两个类的关系解耦。代理类和委托类都要实现相同的接口，因为代理真正调用的是委托类的方法。

区别：

- 静态代理：由程序员创建或者是由特定工具创建，在代码编译时就确定了被代理的类是一个静态代理。静态代理通常只代理一个类；
- 动态代理：在代码运行期间，由 JVM 根据目标对象自动生成代理类，不需要为每个被代理类手写代理类。JDK 动态代理基于接口（要求目标对象实现接口），CGLIB 动态代理基于继承（通过生成目标类的子类作为代理，不要求目标对象实现接口）。

### spring是如何解决循环依赖的

循环依赖指的是两个类中的属性相互依赖对方，循环依赖问题在Spring中主要有三种情况：

第一种：通过构造方法进行依赖注入时产生的循环依赖问题。
第二种：通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
第三种：通过 setter 方法或字段（Field）进行依赖注入且是在单例模式下产生的循环依赖问题。

只有【第三种方式】的循环依赖问题被 Spring 解决了，其他两种方式在遇到循环依赖问题时，Spring都会产生异常。

### spring三级缓存

三者都是 Map 类型的缓存，但 value 的类型不同：

- 一级缓存（Singleton Objects）：Map<String, Object> 存储的是已经完全初始化好的 bean，即完全准备好可以使用的 bean 实例。键是 bean 的名称，值是 bean 的实例。对应 DefaultSingletonBeanRegistry 类中的 singletonObjects 属性。
- 二级缓存（Early Singleton Objects）：Map<String, Object> 存储的是早期的 bean 引用，即已经实例化但还未完全初始化的 bean（可能是原始对象，也可能是为解决 AOP 循环依赖而提前生成的代理对象）。这些 bean 已经被实例化，但可能还没有完成属性注入等操作。对应 DefaultSingletonBeanRegistry 类中的 earlySingletonObjects 属性。
- 三级缓存（Singleton Factories）：Map<String, ObjectFactory<?>> 注意 value 不是 bean 实例本身，而是一个 ObjectFactory 工厂函数。当一个 bean 正在创建过程中被其他 bean 依赖时，就会调用这个工厂的 getObject() 方法生成早期引用（必要时会提前生成代理），从而解决循环依赖与 AOP 协同的问题。对应 DefaultSingletonBeanRegistry 类中的 singletonFactories 属性。

### bean的生命周期

Spring 只帮我们管理单例模式 Bean 的完整生命周期，对于 prototype 的 Bean，Spring 在创建好交给使用者之后，则不会再管理后续的生命周期。

Bean作用域（Scope）定义了Bean的生命周期和可见性。不同的作用域影响着 Spring 如何管理这些Bean。

- Singleton（单例）：在整个应用程序中只存在一个 Bean 实例。默认作用域，Spring 容器中只会创建一个 Bean 实例，并在容器的整个生命周期中共享该实例。
- Prototype（原型）：每次请求时都会创建一个新的 Bean 实例。每次从容器中获取该 Bean 时都会创建一个新实例，适用于状态非常瞬时的 Bean。
- Request（请求）：每个 HTTP 请求都会创建一个新的 Bean 实例。仅在 Spring Web 应用程序中有效，每个 HTTP 请求都会创建一个新的 Bean 实例，适用于 Web 应用中需求局部性的 Bean。
- Session（会话）：Session 范围内只会创建一个 Bean 实例。该 Bean 实例在用户会话范围内共享，仅在 Spring Web 应用程序中有效，适用于与用户会话相关的 Bean。
- Application：当前 ServletContext 中只存在一个 Bean 实例。仅在 Spring Web 应用程序中有效，该 Bean 实例在整个 ServletContext 范围内共享，适用于应用程序范围内共享的 Bean。
- WebSocket（Web套接字）：在 WebSocket 范围内只存在一个 Bean 实例。仅在支持 WebSocket 的应用程序中有效，该 Bean 实例在 WebSocket 会话范围内共享，适用于 WebSocket 会话范围内共享的 Bean。
- Custom scopes（自定义作用域）：Spring 允许开发者定义自定义的作用域，通过实现 Scope 接口来创建新的 Bean 作用域。