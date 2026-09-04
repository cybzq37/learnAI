# Java 并发编程

```text
Java 并发编程
│
├── 一、并发基础
│   ├── 进程、线程
│   ├── Java线程模型
│   ├── 平台线程
│   ├── 虚拟线程
│   ├── Thread / Runnable / Callable
│   ├── Future / FutureTask
│   ├── 线程生命周期
│   ├── start() / run()
│   ├── sleep() / wait() / join()
│   ├── interrupt()
│   ├── daemon线程
│   └── 线程上下文切换
│
├── 二、JMM Java内存模型
│   ├── 主内存 / 工作内存
│   ├── 原子性
│   ├── 可见性
│   ├── 有序性
│   ├── 指令重排序
│   ├── 内存屏障
│   └── happens-before
│
├── 三、synchronized
│   ├── 对象锁
│   ├── 类锁
│   ├── 同步方法
│   ├── 同步代码块
│   ├── 可重入
│   ├── Monitor
│   ├── monitorenter / monitorexit
│   ├── 对象头 / Mark Word
│   ├── 锁竞争
│   └── 锁优化
│
├── 四、volatile
│   ├── 可见性
│   ├── 有序性
│   ├── 内存屏障
│   ├── happens-before
│   ├── DCL单例
│   ├── 状态标志
│   └── volatile为什么不能保证i++
│
├── 五、Lock体系
│   ├── Lock接口
│   ├── ReentrantLock
│   │   ├── 可重入
│   │   ├── 公平/非公平
│   │   ├── tryLock
│   │   └── lockInterruptibly
│   ├── Condition
│   │   ├── await
│   │   ├── signal
│   │   └── signalAll
│   ├── ReentrantReadWriteLock
│   └── StampedLock
│
├── 六、CAS与原子类
│   ├── CAS原理
│   ├── 自旋
│   ├── ABA
│   ├── AtomicInteger
│   ├── AtomicLong
│   ├── AtomicBoolean
│   ├── AtomicReference
│   ├── AtomicStampedReference
│   ├── AtomicIntegerFieldUpdater
│   ├── LongAdder
│   └── VarHandle
│
├── 七、AQS
│   ├── state
│   ├── Node
│   ├── CLH变体队列
│   ├── 独占模式
│   ├── 共享模式
│   ├── acquire
│   ├── release
│   ├── park / unpark
│   ├── ConditionObject
│   └── 模板方法模式
│
├── 八、线程间通信
│   ├── wait / notify / notifyAll
│   ├── Lock / Condition
│   ├── LockSupport
│   ├── CountDownLatch
│   ├── CyclicBarrier
│   ├── Semaphore
│   ├── Exchanger
│   ├── Phaser
│   └── BlockingQueue
│
├── 九、线程池
│   ├── Executor
│   ├── ExecutorService
│   ├── ThreadPoolExecutor
│   ├── 7大核心参数
│   ├── 工作流程
│   ├── Worker
│   ├── Worker队列
│   ├── 拒绝策略
│   ├── execute / submit
│   ├── Future
│   ├── shutdown / shutdownNow
│   ├── ScheduledThreadPoolExecutor
│   ├── ForkJoinPool
│   └── Executors
│
├── 十、并发容器
│   ├── ConcurrentHashMap
│   ├── CopyOnWriteArrayList
│   ├── ConcurrentLinkedQueue
│   ├── BlockingQueue
│   ├── ArrayBlockingQueue
│   ├── LinkedBlockingQueue
│   ├── PriorityBlockingQueue
│   ├── DelayQueue
│   └── SynchronousQueue
│
├── 十一、CompletableFuture异步编程
│   ├── Future
│   ├── CompletableFuture
│   ├── supplyAsync
│   ├── runAsync
│   ├── thenApply
│   ├── thenCompose
│   ├── thenCombine
│   ├── allOf
│   ├── anyOf
│   ├── exceptionally
│   ├── handle
│   └── 异步线程池
│
├── 十二、并发安全问题
│   ├── 竞态条件
│   ├── 脏读
│   ├── 数据覆盖
│   ├── 可见性问题
│   ├── 原子性问题
│   ├── 有序性问题
│   ├── 死锁
│   ├── 活锁
│   ├── 饥饿
│   └── 线程泄漏
│
├── 十三、并发设计模式
│   ├── 生产者-消费者
│   ├── Future模式
│   ├── 两阶段终止
│   ├── 不可变对象
│   ├── ThreadLocal
│   ├── 单例模式
│   └── 无锁编程
│
├── 十四、并发性能
│   ├── CPU缓存
│   ├── Cache Line
│   ├── 伪共享
│   ├── 上下文切换
│   ├── 锁竞争
│   ├── 自旋
│   ├── 吞吐量
│   ├── 响应时间
│   └── 并发量
│
├── 十五、并发问题排查
│   ├── jstack
│   ├── jps
│   ├── jcmd
│   ├── JFR
│   ├── Arthas
│   ├── 线程Dump
│   ├── 死锁排查
│   ├── CPU过高排查
│   └── 线程数暴涨排查
│
└── 十六、生产实践
    ├── 线程池隔离
    ├── 超时控制
    ├── 限流
    ├── 降级
    ├── 熔断
    ├── 异步化
    ├── 批量处理
    ├── 数据库并发控制
    ├── 分布式锁
    └── 分布式并发问题
```

## 1.1 进程与线程

**进程** 是操作系统进行资源分配和管理的基本单位。一个进程拥有独立的地址空间以及相应的系统资源。

**线程** 是进程内部的执行单元，是 CPU 调度的基本对象。同一个进程中的多个线程可以共享进程资源，因此线程之间更适合进行协作，但共享资源也带来了并发安全问题。

**为什么需要多线程?** 多线程的核心价值是让多个任务能够并发推进，常见目的包括：

- 提高 CPU 利用率
- 提高 I/O 等待期间的资源利用率
- 提升系统吞吐能力
- 将耗时任务异步化

同时，多线程会带来上下文切换、锁竞争、共享数据一致性等额外成本。

## 1.2 Java 线程模型

Java 中可以从两个层次理解线程：

- 平台线程：传统线程模型，Java 线程与操作系统线程形成较直接的对应关系。
- 虚拟线程：由 JVM 管理和调度的轻量级线程，用更低的创建和调度成本支持大量并发任务。

**平台线程** 与操作系统线程关系紧密，线程调度主要依赖操作系统。线程数量增加后，线程栈、调度和上下文切换都会形成明显成本。

**虚拟线程** 由 JVM 调度，大量虚拟线程可以复用较少的平台线程，更适合大量等待型任务，例如网络、数据库和其他 I/O 密集型场景。

## 1.3 Java 中创建线程的方式

### 方式一：继承 Thread

适合需要直接定制线程行为的场景。

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("执行任务");
    }
}
```

### 方式二：实现 Runnable

Runnable 只负责描述“做什么”，没有返回值，把“任务”和“线程”解耦。

```java
Runnable task = () -> {
    System.out.println("执行任务");
};

new Thread(task).start();
```

### 方式三：Callable + Future

Callable 负责描述“做什么以及返回什么”，可以产生结果。可以返回执行结果，也可以抛出异常。

Future 代表异步任务的结果，常用于：

- 获取结果
- 判断任务状态
- 取消任务
- 等待任务完成

FutureTask 是 Future + Runnable 的组合，可以直接作为线程执行任务，同时持有异步结果。


```java
Callable<Integer> task = () -> 100;
FutureTask<Integer> futureTask = new FutureTask<>(task);

new Thread(futureTask).start();

Integer result = futureTask.get();
```

### 方式四：线程池

生产环境更常见。把线程作为可复用资源统一管理，避免频繁创建和销毁线程。

## 1.4 线程生命周期

线程运行过程中通常会经历以下状态：

- NEW：线程已经创建，但还没有启动
- RUNNABLE：已经启动，等待或正在获得 CPU 执行
- BLOCKED：等待进入 synchronized 对应的 Monitor
- WAITING：主动等待其他线程通知、终止或取消阻塞
- TIMED_WAITING：带时间限制的等待
- TERMINATED：线程执行结束

**BLOCKED：**

- 通常是竞争对象锁失败
- 锁释放后重新参与竞争

**WAITING：**

- 由 wait、join、park 等机制主动进入
- 需要特定事件触发后才能继续

可以简单记忆： 
> BLOCKED 是“等锁”，WAITING 是“等事件”。

## 1.5 start() 与 run()

`start()` 才是真正启动一个新的线程执行任务。

`run()` 只是普通方法调用。如果直接调用 `run()`，不会创建新的线程。

```java
thread.start(); // 新线程执行
thread.run();   // 当前线程直接调用
```

## 1.6 sleep()、wait()、join()

`sleep()` 让当前线程进入指定时间的休眠状态，不会因为休眠而释放已经持有的锁。

`wait()` 当前线程进入等待状态，并释放当前对象监视器锁，等待其他线程调用 `notify()` 或 `notifyAll()`。

`join()` 一个线程等待另一个线程执行完成。

典型场景：

```java
thread.start();
thread.join();
System.out.println("线程已经执行完成");
```

## 1.7 interrupt()

`interrupt()` 不是“直接杀死线程”，而是一种线程协作式的中断机制。

常见处理方式：

```java
while (!Thread.currentThread().isInterrupted()) {
    // 执行任务
}
```

需要区分：

- `isInterrupted()`：读取中断状态，不清除
- `Thread.interrupted()`：读取并清除当前线程中断状态

对于处于某些阻塞状态的线程，中断可能通过 `InterruptedException` 表现出来。

## 1.8 daemon 线程

守护线程主要承担后台辅助工作。

当用户线程已经结束时，JVM 不会因为仍然存在守护线程而继续等待整个进程。

因此，守护线程通常不适合承担必须完整执行的核心业务任务。

## 1.9 线程上下文切换

线程上下文切换是指 CPU 从一个线程切换到另一个线程时，需要保存当前线程状态并恢复目标线程状态。

线程数量过多会带来：

- CPU 时间消耗在调度上
- Cache 命中率下降
- 实际业务执行时间减少
- 系统吞吐下降

因此：并不是线程越多，并发性能就越高。

# 二、JMM：Java 内存模型

## 2.1 JMM 解决什么问题

多线程环境下，线程之间共享变量时会出现三个核心问题：

1. 原子性
2. 可见性
3. 有序性

JMM 用一套抽象规则描述线程之间如何访问共享变量，以及不同线程之间什么时候能够看到彼此的修改。


**主内存与工作内存**

可以把 JMM 粗略理解成：

- 主内存：保存共享变量
- 工作内存：线程执行时使用的本地数据环境

线程对共享变量进行读写时，会涉及主内存与工作内存之间的数据交互。

因此，多线程问题不能简单理解成“大家直接操作同一个变量”，而要从内存可见性和执行顺序来分析。

## 2.2 原子性

原子性表示一个操作不可被线程调度打断到中间状态。

例如：

```java
i++;
```

从逻辑上可以拆成：

```text
读取 i
计算 i + 1
写回 i
```

多个线程同时执行时，整体就不再天然具备原子性。

常见解决手段：

- synchronized
- Lock
- CAS
- 原子类

## 2.3 可见性

一个线程修改共享变量后，其他线程是否能够及时看到结果，就是可见性问题。

典型手段：

- volatile
- synchronized
- Lock

## 2.4 有序性

编译器、JVM 和处理器为了优化性能，可能对指令进行重排序。

如果程序存在并发依赖，而没有适当的同步机制，重排序可能导致观察到的执行顺序与代码表面顺序不同。

## 2.5 指令重排序

理解并发代码时，可以把执行顺序分成三个层面：

```text
源代码顺序
    ↓
编译后的指令顺序
    ↓
处理器实际执行顺序
```

并发控制的一个重要任务，就是在必要的位置建立顺序约束。

## 2.7 内存屏障

内存屏障用于约束特定类型的内存访问顺序，并帮助实现线程之间的数据可见性。

理解 volatile、锁和其他并发机制时，可以把内存屏障看成底层实现的重要工具。

## 2.8 happens-before

happens-before 描述一种跨线程的先行关系：

> 如果 A happens-before B，那么 A 的结果对 B 可见，并且 A 的执行顺序先于 B。

核心规则：

### 规则一：程序次序规则
同一个线程内，前面的操作先行发生于后面的操作。

### 规则二：管程锁定规则
对同一个锁，解锁操作先行发生于之后的加锁操作。

### 规则三：volatile 变量规则
对一个 volatile 变量的写，先行发生于后续对该变量的读。

### 规则四：线程启动规则
对线程的 `start()` 操作先行发生于新线程中的操作。

### 规则五：线程终止规则
线程中的操作先行发生于其他线程检测到该线程终止。

### 规则六：线程中断规则
线程调用 `interrupt()` 先行发生于目标线程检测到中断。

### 规则七：传递性
如果：

```text
A happens-before B
B happens-before C
```

那么：

```text
A happens-before C
```

---

# 三、synchronized

## 3.1 synchronized 的定位

`synchronized` 是 Java 内置的同步机制，核心能力是：

- 互斥
- 可见性
- 可重入

它把“对共享资源的访问”与“锁”绑定起来。

## 3.2 对象锁与类锁

### 对象锁

锁住某个对象实例。

```java
synchronized (obj) {
    // 临界区
}
```

不同对象实例可以拥有不同的对象锁。

### 类锁

锁住 `Class` 对象。

```java
public static synchronized void test() {
}
```

或者：

```java
synchronized (Demo.class) {
}
```

类锁的核心对象是 `Demo.class`。

## 3.3 同步方法与同步代码块

### 同步方法

```java
public synchronized void increment() {
    count++;
}
```

### 同步代码块

```java
synchronized (lock) {
    count++;
}
```

同步代码块可以更精细地控制锁的范围。

## 3.4 可重入

同一个线程已经获得某个锁之后，可以再次获得同一把锁，而不会因为自己持有锁而把自己阻塞。

例如：

```java
public synchronized void a() {
    b();
}

public synchronized void b() {
    // 同一个线程可以再次进入
}
```

## 3.5 Monitor

`synchronized` 底层与对象监视器 Monitor 相关。

字节码层面可以看到：

- `monitorenter`
- `monitorexit`

进入同步区域时获取 Monitor，退出时释放 Monitor。

## 3.6 对象头与 Mark Word

Java 对象通常包含对象头等元数据。

在 synchronized 的底层实现中，可以结合对象头中的 Mark Word 来理解锁状态和锁竞争信息。

## 3.7 锁竞争与锁优化

锁竞争本质上是：

> 多个线程希望同时进入同一个受保护的临界区。

竞争不激烈时，可以通过轻量化方式减少阻塞成本；竞争激烈时则可能进入更重的同步路径。

理解锁优化时，要同时考虑：

- 竞争程度
- 持锁时间
- 自旋成本
- 线程阻塞成本
- 上下文切换成本

# 四、volatile

## 4.1 volatile 的核心能力

`volatile` 的核心是：

- 保证可见性
- 提供一定的有序性保证

但它不能把复合操作自动变成原子操作。

例如：

```java
volatile int i;
i++;
```

`i++` 仍然包含读取、计算、写回多个步骤，多个线程同时执行时不能依靠 volatile 保证结果正确。

## 4.2 volatile 与内存屏障

理解 volatile 时，可以围绕两个目标：

```text
写入共享变量
    ↓
建立跨线程可见性

读共享变量
    ↓
读取到符合同步语义的最新结果
```

同时，volatile 会建立必要的顺序约束，避免关键操作被不正确地重排序。

---

## 4.3 volatile 的典型场景

### 场景一：状态标志

```java
volatile boolean running = true;
```

一个线程修改状态，其他线程持续读取状态。

### 场景二：DCL 单例

```java
private static volatile Singleton instance;
```

volatile 用于避免对象发布过程中可能出现的重排序问题。

### 场景三：与 CAS 配合

许多原子类内部会使用 volatile 变量作为共享状态，再通过 CAS 实现原子更新。

## 4.4 为什么 volatile 不能保证 i++

关键原因：

```text
读取 i
↓
i + 1
↓
写回 i
```

这三个动作并不是一个不可分割的原子操作。

即使每一次读取和写入本身具备可见性，不同线程之间仍可能发生：

```text
线程 A：读取 10
线程 B：读取 10
线程 A：写回 11
线程 B：写回 11
```

最终结果小于预期。

# 五、Lock 体系

## 5.1 Lock 接口

`Lock` 提供比 synchronized 更灵活的锁控制方式。

典型操作：

```java
lock.lock();

try {
    // 临界区
} finally {
    lock.unlock();
}
```

核心原则：

> 手动加锁后必须保证最终释放。

## 5.2 ReentrantLock

`ReentrantLock` 是最常用的显式锁。

### 可重入

与 synchronized 一样支持同一线程重复获取同一把锁。

### 公平与非公平

可以在创建时指定是否使用公平策略。

### tryLock

尝试获取锁，不必无限期等待。

```java
if (lock.tryLock()) {
    try {
        // 业务
    } finally {
        lock.unlock();
    }
}
```

还可以使用带超时的方式。

### lockInterruptibly

允许等待锁的过程响应中断。

## 5.3 公平锁与非公平锁

### 公平锁
按照申请顺序进行竞争。

优点：

- 更容易避免线程长期得不到锁

缺点：

- 调度成本更高
- 吞吐通常不如非公平策略

### 非公平锁

允许新到达的线程直接尝试获取刚刚释放的锁。

优点：

- 减少线程唤醒和调度成本
- 通常吞吐更高

缺点：

- 个别线程可能等待更久

---

## 5.4 Condition

Condition 把“等待/通知”从 synchronized 的对象监视器中独立出来。

核心方法：

- `await()`
- `signal()`
- `signalAll()`

它特别适合生产者—消费者等需要多个条件队列的场景。

---

## 5.5 ReentrantReadWriteLock

读写锁把访问分为：

- 读锁
- 写锁

适合“读多写少”的共享数据场景。

思路：

```text
多个线程可以同时读取
        ↓
写入时需要更强的互斥
```

---

## 5.6 StampedLock

StampedLock 是另一种读写同步机制，提供基于 stamp 的锁状态管理。

在读场景中，可以进一步支持更轻量的读取路径。

---

## 5.7 synchronized 与 ReentrantLock

| 维度 | synchronized | ReentrantLock |
|---|---|---|
| 可重入 | 支持 | 支持 |
| 可中断获取 | 能力有限 | 支持 `lockInterruptibly()` |
| 超时尝试 | 不直接提供 | 支持 `tryLock()` |
| 公平策略 | 不提供可配置公平参数 | 支持公平/非公平 |
| 条件队列 | 单一 wait set | 可创建多个 Condition |
| 释放方式 | JVM 自动处理 | 必须显式 unlock |
| 使用复杂度 | 更简单 | 更灵活 |

使用原则：

> 能用 synchronized 清晰表达时优先考虑 synchronized；确实需要中断、超时、公平或多条件队列时，再选择 ReentrantLock。

---

# 六、CAS 与原子类

## 6.1 CAS 是什么

CAS 全称 Compare And Swap。

核心逻辑：

```text
读取当前值 V
↓
判断 V 是否等于预期值 A
↓
如果相等
    ↓
更新为 B
否则
    ↓
更新失败
```

整个“比较 + 更新”作为一个原子操作完成。

---

## 6.2 CAS 与自旋

CAS 往往配合循环重试：

```text
不断尝试 CAS
    ↓
成功 → 结束
失败 → 重试
```

如果竞争时间很短，自旋可以减少线程阻塞与上下文切换。

如果竞争时间很长，则会产生大量 CPU 消耗。

---

## 6.3 CAS 的问题

### ABA 问题

某个值经历：

```text
A → B → A
```

最终再次看到 A 时，仅通过值本身可能无法判断中间是否发生过变化。

解决思路：

> 值 + 版本号

典型工具：

`AtomicStampedReference`

### 自旋时间过长

CAS 长时间失败会导致大量无效重试，占用 CPU。

### 多变量原子性问题

CAS 天然适合针对一个共享状态进行原子更新。

如果一个业务操作需要同时保证多个独立变量的一致性，就需要额外的同步设计。

---

## 6.4 原子类

常见原子类：

- AtomicInteger
- AtomicLong
- AtomicBoolean
- AtomicReference
- AtomicStampedReference
- AtomicIntegerFieldUpdater
- LongAdder

### AtomicInteger

适合原子计数、状态维护等。

### AtomicReference

适合对对象引用进行原子更新。

### AtomicStampedReference

用于处理 ABA 场景中的版本控制。

### LongAdder

适用于高并发计数场景，通过降低多个线程竞争同一个热点变量的程度来提高吞吐。

---

## 6.5 VarHandle

VarHandle 提供一种更底层、更灵活的变量访问方式，可用于：

- 原子读写
- CAS
- 不同内存语义的访问

它是理解现代 Java 并发底层能力的重要工具。

---

# 七、AQS

## 7.1 AQS 是什么

AQS（AbstractQueuedSynchronizer）是 Java 并发工具的重要基础框架。

它并不是一把具体的业务锁，而是一个用于构建锁和同步器的通用框架。

---

## 7.2 AQS 三个核心组成

### 1. state

`state` 表示同步状态。

不同同步器对 state 有不同解释：

- ReentrantLock：锁状态/重入次数
- Semaphore：剩余许可
- CountDownLatch：剩余计数

### 2. Node

无法立即获取资源的线程会被封装成节点进入等待队列。

### 3. CLH 变体队列

AQS 采用队列组织等待线程，使线程能够按照一定规则排队等待同步资源。

---

## 7.3 独占模式与共享模式

### 独占模式

同一时刻通常只有一个线程能够获得资源。

典型：

`ReentrantLock`

### 共享模式

多个线程可以同时获得资源。

典型：

- Semaphore
- CountDownLatch
- ReentrantReadWriteLock 的读锁

---

## 7.4 acquire 与 release

AQS 提供同步器的通用流程：

```text
尝试获取资源
    ↓
成功 → 继续执行
失败 → 节点入队
    ↓
阻塞等待
    ↓
资源释放
    ↓
唤醒后继线程
```

`acquire/release` 是模板流程，具体资源获取和释放规则由子类实现。

---

## 7.5 park / unpark

线程竞争不到资源时，不适合一直占用 CPU。

因此 AQS 会与 `LockSupport.park/unpark` 配合：

```text
竞争失败
  ↓
park
  ↓
线程阻塞
  ↓
资源释放
  ↓
unpark
  ↓
线程恢复执行
```

---

## 7.6 ConditionObject

AQS 内部还提供 Condition 机制，用于构造：

- 等待条件
- 唤醒条件
- 多条件队列

这也是 ReentrantLock 能够支持多个 Condition 的基础。

---

## 7.7 模板方法模式

AQS 的核心设计思想：

> AQS 负责通用骨架，子类负责定义资源语义。

因此：

```text
AQS
 ├── acquire / release
 ├── 排队
 ├── 阻塞
 └── 唤醒

具体同步器
 ├── 定义什么叫“获取成功”
 └── 定义什么叫“释放成功”
```

---

# 八、线程间通信

## 8.1 wait / notify / notifyAll

### wait()
线程等待，并释放对象监视器。

### notify()
唤醒一个等待线程。

### notifyAll()
唤醒所有等待线程，让它们重新参与锁竞争。

### notify 与 notifyAll 的选择

如果多个线程等待不同业务条件，仅使用单一 notify 可能导致唤醒不理想。

因此，在复杂协作场景中，需要更加明确地设计等待条件与唤醒策略。

---

## 8.2 Lock / Condition

通过：

```java
Condition condition = lock.newCondition();
```

建立独立等待队列。

常见方法：

- await
- signal
- signalAll

相比对象监视器，它可以创建多个条件队列。

---

## 8.3 LockSupport

核心方法：

- `park()`
- `unpark(thread)`

特点是可以直接针对线程进行阻塞和唤醒，是 AQS 底层的重要基础设施。

---

## 8.4 CountDownLatch

适合“一组任务完成后再继续”的场景。

例如：

```text
任务 A ─┐
任务 B ─┼→ countDown
任务 C ─┘
          ↓
       计数归零
          ↓
      主线程继续
```

核心方法：

- `countDown()`
- `await()`

特点：

> 一次性使用。

---

## 8.5 CyclicBarrier

适合“一组线程互相等待”的场景。

```text
线程 A ─┐
线程 B ─┼→ 到达屏障
线程 C ─┘
         ↓
      全部到达
         ↓
      一起继续
```

特点：

- 可以重复使用
- 可以配合 barrierAction

---

## 8.6 Semaphore

Semaphore 通过“许可”控制并发数量。

```text
总许可 = 10
↓
每个线程获取 1 个许可
↓
许可耗尽后继续等待
↓
任务结束释放许可
```

常见用途：

- 限制并发访问量
- 控制有限资源使用
- 控制同时执行的任务数量

---

## 8.7 Exchanger

用于两个线程之间直接交换数据。

典型理解：

```text
线程 A：拿出数据 A
线程 B：拿出数据 B
        ↓
两者交换
        ↓
A 得到 B
B 得到 A
```

---

## 8.8 Phaser

Phaser 可以理解为一种更灵活的多阶段同步工具，适合多个阶段反复进行同步。

适用于：

```text
阶段 1 → 同步
阶段 2 → 同步
阶段 3 → 同步
```

---

## 8.9 BlockingQueue

阻塞队列非常适合生产者—消费者模型。

核心语义：

- 队列满：生产者等待
- 队列空：消费者等待

从而自然形成线程之间的协作机制。

---

## 8.10 CountDownLatch vs CyclicBarrier

| 对比项 | CountDownLatch | CyclicBarrier |
|---|---|---|
| 核心关系 | 等其他任务完成 | 一组线程互相等待 |
| 是否可复用 | 否 | 是 |
| 典型场景 | 主线程等待多个任务完成 | 多线程分阶段协同 |
| 计数逻辑 | 递减到 0 | 达到参与线程数后通过屏障 |
| 额外动作 | 无 | 可配置 barrierAction |

---

# 九、线程池

## 9.1 为什么使用线程池

频繁创建线程会产生资源和调度成本。

线程池通过：

```text
创建线程
↓
重复执行任务
↓
任务结束后线程继续复用
```

降低线程创建与销毁成本，同时统一管理并发规模。

---

## 9.2 Executor 体系

主要层次：

```text
Executor
   ↓
ExecutorService
   ↓
ThreadPoolExecutor
```

常见实现：

- ThreadPoolExecutor
- ScheduledThreadPoolExecutor
- ForkJoinPool

---

## 9.3 ThreadPoolExecutor 七大核心参数

```java
new ThreadPoolExecutor(
    corePoolSize,
    maximumPoolSize,
    keepAliveTime,
    unit,
    workQueue,
    threadFactory,
    handler
);
```

### corePoolSize
核心线程数量。

### maximumPoolSize
最大线程数量。

### keepAliveTime
非核心线程空闲后的存活时间。

### unit
存活时间单位。

### workQueue
任务等待队列。

### threadFactory
线程创建工厂。

### handler
线程无法继续接收任务后的拒绝策略。

---

## 9.4 线程池工作流程

必须掌握：

```text
提交任务
  ↓
核心线程是否已满？
 ├─ 否 → 创建核心线程
 └─ 是
      ↓
任务队列是否已满？
 ├─ 否 → 进入队列
 └─ 是
      ↓
当前线程数是否达到最大线程数？
 ├─ 否 → 创建非核心线程
 └─ 是 → 拒绝策略
```

记忆：

> 核心线程 → 任务队列 → 非核心线程 → 拒绝策略

---

## 9.5 Worker

Worker 用于承载线程池中的实际工作线程及其执行状态。

线程池通过 Worker 组织“线程 + 任务执行”之间的关系。

---

## 9.6 工作队列

常见队列：

- ArrayBlockingQueue
- LinkedBlockingQueue
- SynchronousQueue

队列选型会直接影响：

- 任务堆积能力
- 线程扩张时机
- 内存压力
- 拒绝策略触发频率

---

## 9.7 拒绝策略

### AbortPolicy
直接抛出异常。

### CallerRunsPolicy
由提交任务的线程执行任务，形成一定的反压效果。

### DiscardPolicy
直接丢弃新任务。

### DiscardOldestPolicy
丢弃队列中最旧的任务，再尝试提交新任务。

选择时必须结合业务是否允许任务丢失。

---

## 9.8 execute 与 submit

### execute

适合 Runnable，没有返回结果。

```java
executor.execute(task);
```

### submit

支持 Runnable / Callable，并返回 Future。

```java
Future<?> future = executor.submit(task);
```

Future 可以进一步获取执行结果或异常。

---

## 9.9 shutdown 与 shutdownNow

### shutdown
不再接收新任务，但会继续执行已经提交的任务。

### shutdownNow
尝试停止正在执行的任务，并返回尚未开始执行的任务。

线程池生命周期管理必须明确，不能创建后长期不关闭。

---

## 9.10 ScheduledThreadPoolExecutor

用于：

- 延迟执行
- 周期执行
- 定时任务

---

## 9.11 ForkJoinPool

ForkJoinPool 适合可拆分、可并行处理的任务。

核心思想：

```text
大任务
 ↓
拆成多个小任务
 ↓
并行执行
 ↓
汇总结果
```

---

## 9.12 Executors

Executors 提供快速创建线程池的工具方法，例如：

- newFixedThreadPool
- newCachedThreadPool
- newSingleThreadExecutor
- 定时任务线程池相关工厂方法

在生产环境中，更应明确控制：

- 最大线程数
- 队列容量
- 拒绝策略
- 线程命名
- 线程生命周期

---

## 9.13 线程池核心线程数

### CPU 密集型

常用估算思路：

```text
线程数 ≈ CPU 核心数 + 1
```

### I/O 密集型

通常可以设置得更高。

一个更通用的思路是结合：

```text
线程数 ≈ CPU 核心数 × (1 + 等待时间 / 计算时间)
```

核心原则：

> CPU 密集任务不宜盲目增加线程；I/O 密集任务可通过增加并发线程覆盖等待时间。

---

## 9.14 不同 Executors 线程池的特点

### FixedThreadPool
核心线程数与最大线程数相同，任务更多时主要进入队列等待。

### CachedThreadPool
倾向于快速创建线程处理任务，适合短时、弹性任务，但如果缺乏约束，线程数可能快速膨胀。

### SingleThreadExecutor
只有一个工作线程，适合要求任务按顺序执行的场景。

### SingleThreadScheduledExecutor
单线程版本的定时任务线程池。

---

# 十、并发容器

## 10.1 ConcurrentHashMap

ConcurrentHashMap 用于多线程环境下的并发键值存储。

核心关注点：

- 并发读写
- 锁粒度
- CAS
- 局部同步

理解版本演进时，可重点关注 JDK 1.7 与 JDK 1.8 的实现差异。

---

## 10.2 CopyOnWriteArrayList

核心思想：

> 写时复制，读写分离。

写操作创建新的底层数组，读操作尽量读取已有数组。

因此适合：

- 读多写少
- 遍历频繁
- 修改相对少

---

## 10.3 ConcurrentLinkedQueue

非阻塞并发队列，适合不要求阻塞语义的并发场景。

---

## 10.4 BlockingQueue

阻塞队列既属于并发容器，也属于线程协作基础设施。

常见实现：

- ArrayBlockingQueue
- LinkedBlockingQueue
- PriorityBlockingQueue
- DelayQueue
- SynchronousQueue

---

## 10.5 ArrayBlockingQueue

基于数组的有界阻塞队列。

适合明确控制队列容量的场景。

---

## 10.6 LinkedBlockingQueue

基于链表结构的阻塞队列，可以用于生产者—消费者模型。

---

## 10.7 PriorityBlockingQueue

支持按优先级获取元素的阻塞队列。

---

## 10.8 DelayQueue

只有当元素满足延迟条件后才允许取出的阻塞队列。

---

## 10.9 SynchronousQueue

不保存实际元素，核心是“直接交接”。

```text
生产者放入
   ↕
消费者接收
```

常与需要快速任务移交的并发模型结合。

---

# 十一、CompletableFuture 异步编程

## 11.1 Future 的局限

Future 能够表达异步结果，但多个异步任务组合起来时，代码容易出现：

- 阻塞等待
- 回调嵌套
- 异常处理分散
- 流程难以组织

CompletableFuture 用链式方式解决这些问题。

---

## 11.2 CompletableFuture

可以把一个异步流程表示为：

```text
任务 A
 ↓
任务 B
 ↓
任务 C
```

也可以表示：

```text
任务 A ─┐
任务 B ─┼→ 汇总
任务 C ─┘
```

---

## 11.3 supplyAsync

适合有返回值的异步任务。

```java
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> 100);
```

---

## 11.4 runAsync

适合没有返回值的异步任务。

```java
CompletableFuture.runAsync(() -> {
    // 执行异步任务
});
```

---

## 11.5 thenApply

对前一个异步结果进行加工。

```text
A → 转换 → B
```

---

## 11.6 thenCompose

用于异步任务串联，当前任务的结果决定下一个异步任务。

```text
A
 ↓
根据 A 的结果启动 B
```

---

## 11.7 thenCombine

用于组合两个相互独立的异步任务。

```text
A ─┐
   ├→ 合并
B ─┘
```

---

## 11.8 allOf

等待多个 CompletableFuture 全部完成。

适合：

- 并发调用多个服务
- 多任务并行后统一汇总

---

## 11.9 anyOf

多个异步任务中，任意一个完成即可继续。

---

## 11.10 exceptionally

用于异常恢复。

核心思路：

```text
正常 → 正常继续
异常 → 进入异常处理分支
```

---

## 11.11 handle

同时处理：

- 正常结果
- 异常结果

因此适合需要统一处理结果状态的场景。

---

## 11.12 异步线程池

CompletableFuture 的异步执行最好明确线程池归属，避免所有任务都依赖同一个默认执行环境而导致：

- 任务相互影响
- 并发隔离不足
- 慢任务拖累快任务

---

# 十二、并发安全问题

## 12.1 竞态条件

竞态条件是指：

> 程序结果依赖多个线程执行的时序。

例如：

```text
线程 A 读取旧值
线程 B 同时读取旧值
线程 A 修改
线程 B 再修改
```

最终状态可能与预期不一致。

---

## 12.2 脏读

读取到了一个不符合业务一致性要求的中间状态或过期状态。

需要从数据可见性、同步边界和业务事务性多个层面分析。

---

## 12.3 数据覆盖

多个线程先后写入同一数据：

```text
A 写入新值
B 使用旧值计算后覆盖
```

可能导致 A 的更新结果丢失。

---

## 12.4 可见性问题

一个线程修改数据，其他线程没有及时看到。

常用解决方式：

- volatile
- synchronized
- Lock
- happens-before 关系

---

## 12.5 原子性问题

一个业务操作由多个步骤组成，但多个线程可以在中间插入，导致结果错误。

常见解决方式：

- synchronized
- Lock
- CAS
- 原子类

---

## 12.6 有序性问题

线程观察到的执行顺序与预期不同。

常用分析工具：

- volatile
- synchronized
- happens-before
- 内存屏障

---

## 12.7 死锁

### 定义

多个线程彼此等待对方持有的资源，最终都无法继续。

### 四个必要条件

1. 互斥
2. 占有并等待
3. 不可剥夺
4. 循环等待

### 典型结构

```text
线程 A：持有锁 A → 等锁 B
线程 B：持有锁 B → 等锁 A
```

### 避免死锁

- 固定加锁顺序
- 缩小锁范围
- 使用 tryLock + 超时
- 减少嵌套锁
- 破坏循环等待

---

## 12.8 活锁

线程没有阻塞，但因为不断做出相互让步或重试，程序一直无法真正完成任务。

---

## 12.9 饥饿

某些线程长期无法获得所需资源。

可能原因：

- 非公平锁
- 资源长期被其他线程占用
- 调度策略不合理

---

## 12.10 线程泄漏

线程被创建后长期无法退出，导致：

- 线程数量不断增加
- 内存和调度资源消耗
- 最终影响系统稳定性

典型原因包括：

- 线程池未关闭
- 任务无法结束
- 阻塞调用没有超时
- 线程生命周期管理不当

---

# 十三、并发设计模式

## 13.1 生产者—消费者

模型：

```text
生产者
  ↓
BlockingQueue
  ↓
消费者
```

优势：

- 解耦生产与消费速度
- 利用队列进行缓冲
- 自然实现线程协作

---

## 13.2 Future 模式

把“任务执行”与“获取结果”分开：

```text
提交任务
 ↓
立即返回 Future
 ↓
后台执行
 ↓
需要时获取结果
```

适合耗时任务异步化。

---

## 13.3 两阶段终止

第一阶段：

```text
发出停止信号
```

第二阶段：

```text
线程检测到停止信号
→ 完成必要清理
→ 正常退出
```

它强调协作式终止，而不是暴力终止。

---

## 13.4 不可变对象

对象创建完成后状态不再变化。

并发环境中，不可变对象可以减少：

- 锁需求
- 状态竞争
- 可见性处理复杂度

---

## 13.5 ThreadLocal

ThreadLocal 为每个线程维护独立的数据副本。

适合存放：

- 线程上下文
- 请求上下文
- 与线程绑定的临时状态

但需要重视生命周期管理，尤其在线程池场景中。

---

## 13.6 单例模式

并发环境中的单例核心问题：

- 多线程初始化
- 可见性
- 指令重排序
- 初始化时机

DCL 是典型并发单例方案之一：

```text
第一次检查
   ↓
加锁
   ↓
第二次检查
   ↓
创建实例
```

---

## 13.7 无锁编程

核心思想是减少传统互斥锁的使用，通过：

- CAS
- 原子类
- 合理的数据结构
- 非阻塞算法

实现线程安全。

适合竞争模式明确、可接受重试成本的场景。

---

# 十四、并发性能

## 14.1 CPU 缓存

现代处理器会使用多级缓存减少访问内存的成本。

多线程共享数据时，缓存一致性以及数据在不同缓存之间的迁移会影响性能。

---

## 14.2 Cache Line

缓存通常以缓存行作为数据传输和缓存管理的基本单位。

因此：

> 即使两个线程操作的是不同变量，只要变量位于同一个缓存行，也可能互相产生性能影响。

---

## 14.3 伪共享

伪共享是指：

```text
线程 A 修改变量 X
线程 B 修改变量 Y
```

虽然 X、Y 逻辑上互不相关，但它们处于同一缓存行，导致缓存行频繁失效和同步。

结果是：

- CPU 缓存效率下降
- 多线程性能下降

---

## 14.4 上下文切换

线程越多，不一定性能越高。

当线程数量超过系统能够高效执行的范围时：

```text
线程增加
 ↓
调度次数增加
 ↓
上下文切换增加
 ↓
实际业务执行时间下降
```

---

## 14.5 锁竞争

锁竞争受到以下因素影响：

- 并发线程数
- 临界区长度
- 持锁时间
- 锁粒度
- 锁策略

优化思路：

- 缩短临界区
- 减少共享状态
- 降低锁粒度
- 选择合适同步方式

---

## 14.6 自旋

自旋适合等待时间很短的场景。

原则：

```text
等待很短 → 自旋可能更划算
等待很长 → 阻塞更划算
```

---

## 14.7 吞吐量、响应时间、并发量

### 吞吐量
单位时间内系统完成的任务数量。

### 响应时间
一次请求从开始到完成所需要的时间。

### 并发量
同一时刻处于处理过程中的任务数量。

三者不能简单地认为：

> 并发量越大，吞吐量和响应时间越好。

过度并发会引发资源竞争和排队。

---

# 十五、并发问题排查

## 15.1 jps

用于快速查看 Java 进程及 PID。

典型流程：

```text
jps
↓
找到目标 JVM PID
```

---

## 15.2 jstack

用于查看 Java 线程栈，常用于分析：

- 死锁
- BLOCKED
- WAITING
- 线程异常停滞
- CPU 异常线程

典型：

```bash
jstack <PID>
```

---

## 15.3 jcmd

用于向 JVM 查询和执行诊断相关命令，可作为更通用的 JVM 诊断入口之一。

---

## 15.4 JFR

Java Flight Recorder 用于采集 JVM 运行过程中的大量诊断信息。

适合：

- 性能分析
- CPU 热点定位
- 线程活动分析
- 锁竞争分析

---

## 15.5 Arthas

Arthas 适合线上 Java 应用诊断，可用于观察：

- 类和方法调用
- 线程状态
- CPU 占用
- 方法执行
- JVM 运行信息

---

## 15.6 线程 Dump

分析线程问题时，可以重点关注：

```text
线程名称
线程状态
阻塞原因
等待对象
持有锁
调用栈
```

---

## 15.7 死锁排查

典型流程：

```text
jps
 ↓
找到 PID
 ↓
jstack PID
 ↓
搜索 deadlock
 ↓
定位线程 A / B
 ↓
分析各自持有与等待的锁
```

---

## 15.8 CPU 过高排查

思路：

```text
系统 CPU 高
 ↓
找到 Java 进程
 ↓
查看线程状态
 ↓
定位高 CPU 线程
 ↓
结合线程 Dump / JFR / Arthas
 ↓
找到具体代码路径
```

---

## 15.9 线程数暴涨排查

关注：

- 线程池配置
- 最大线程数
- 队列堆积
- 任务是否持续阻塞
- 线程是否无法退出
- 是否存在重复创建线程的业务逻辑

---

# 十六、生产实践

## 16.1 线程池隔离

不同业务不要无条件共享同一个线程池。

例如：

```text
核心业务线程池
  ↓
核心任务

普通任务线程池
  ↓
普通任务

异步日志线程池
  ↓
日志任务
```

目的：

> 防止某一类慢任务耗尽线程资源后，影响其他业务。

---

## 16.2 超时控制

所有可能长期等待的操作都应该考虑超时：

- 网络请求
- 数据库操作
- 锁
- Future
- 异步任务

超时控制可以防止资源长期占用。

---

## 16.3 限流

当请求量超过系统处理能力时，需要限制进入系统的任务数量。

限流的本质：

```text
控制进入速度
↓
避免资源被瞬间打满
```

Semaphore 是 Java 并发控制中常见的基础工具之一。

---

## 16.4 降级

当依赖服务不可用或系统负载过高时，主动降低功能复杂度。

例如：

```text
完整功能
 ↓
核心功能
 ↓
兜底结果
```

重点是保障核心链路可用。

---

## 16.5 熔断

当下游持续失败时，暂时停止继续调用，避免故障扩大。

基本思路：

```text
正常
 ↓
失败率上升
 ↓
熔断
 ↓
快速失败 / 走降级
 ↓
恢复后再尝试
```

---

## 16.6 异步化

对于不要求立即完成的任务，可以从主业务线程中拆出去：

```text
同步主流程
   ↓
快速完成核心业务
   ↓
异步处理非核心任务
```

适合：

- 日志
- 通知
- 非核心数据处理
- 批量任务

---

## 16.7 批量处理

把大量小任务合并成较少的大任务，可以减少：

- 线程切换
- 网络调用次数
- 数据库访问次数
- 调度开销

但批量大小需要根据内存、延迟和吞吐要求权衡。

---

## 16.8 数据库并发控制

数据库是典型共享资源。

常见思路：

- 事务
- 悲观锁
- 乐观锁
- 版本号
- 合理的隔离策略

核心目标：

> 在高并发环境下保证数据的一致性，同时控制锁竞争成本。

---

## 16.9 分布式锁

单机 synchronized、ReentrantLock 只能解决单 JVM 内部的线程竞争。

当多个应用实例共同访问共享资源时，需要进一步考虑分布式锁。

核心问题包括：

- 锁的唯一性
- 锁的释放
- 超时
- 故障恢复
- 锁续期
- 服务宕机后的处理

---

## 16.10 分布式并发问题

从单机走向分布式后，并发问题会进一步扩大：

```text
单 JVM
 ↓
多线程竞争
 ↓
多 JVM
 ↓
跨进程竞争
 ↓
跨节点一致性
```

因此生产环境中的并发治理需要结合：

- 线程池
- 超时
- 限流
- 降级
- 熔断
- 数据库并发控制
- 分布式锁
- 异步化
- 批量化

最终目标不是“让线程越多越好”，而是：

> 在可控资源范围内，让系统稳定地处理更多并发请求。

---

# 总结：Java 并发学习主线

可以把整套知识串成一条主线：

```text
并发基础
   ↓
线程模型与生命周期
   ↓
JMM
   ↓
原子性 / 可见性 / 有序性
   ↓
synchronized / volatile
   ↓
Lock / AQS
   ↓
CAS / 原子类
   ↓
线程间通信
   ↓
线程池
   ↓
并发容器
   ↓
CompletableFuture
   ↓
并发安全问题
   ↓
设计模式
   ↓
性能优化
   ↓
问题排查
   ↓
生产实践
```

## 高频面试串联

### 线程安全问题从哪里来？

```text
共享数据
+
多线程并发访问
=
竞态条件
```

### 如何解决？

```text
控制原子性
→ synchronized / Lock / CAS

控制可见性
→ volatile / synchronized / Lock

控制有序性
→ happens-before / volatile / 锁
```

### AQS 在其中扮演什么角色？

```text
AQS
 ↓
state + 队列 + park/unpark
 ↓
构建同步器
 ↓
ReentrantLock / Semaphore / CountDownLatch 等
```

### 线程池解决什么问题？

```text
线程创建成本
+
线程数量失控
+
任务调度
+
生命周期管理
 ↓
ThreadPoolExecutor
```

### 最终的生产实践

```text
并发能力
+
资源控制
+
故障隔离
+
超时
+
限流
+
降级
+
熔断
+
数据一致性
=
稳定的并发系统
```
