# Java IO 知识

## 1. Java IO 总体分类

```
                    Java IO
                       │
        ┌──────────────┴──────────────┐
        │                             │
       IO                            NIO
        │                             │
   ┌────┴────┐              ┌─────────┼─────────┐
   ↓         ↓              ↓         ↓         ↓
字节流     字符流         Buffer    Channel   Selector
   │         │              │         │         │
InputStream Reader       ByteBuffer FileChannel 多路复用
OutputStream Writer                 SocketChannel
   │         │                      ServerSocketChannel
   └────┬────┘
        ↓
     Buffered
        ↓
     转换流
        ↓
      Data
        ↓
     Object
        ↓
     Print

                    NIO.2
                      │
               ┌──────┴──────┐
               ↓             ↓
             Path          Files
```

Java IO 主要可以从两个维度理解：

### 按数据单位分类

| 分类  | 处理内容                  | 核心抽象类                          |
| --- | --------------------- | ------------------------------ |
| 字节流 | 二进制数据，如图片、音频、视频、文件    | `InputStream` / `OutputStream` |
| 字符流 | 文本数据，如 `.txt`、`.java` | `Reader` / `Writer`            |

### 按流向分类

* **输入流（Input）**：数据从外部进入程序
* **输出流（Output）**：数据从程序流向外部

例如：

```text
文件 ──────> Java程序
       InputStream / Reader

Java程序 ──────> 文件
        OutputStream / Writer
```

---

# 2. IO 核心继承体系

## 字节流

```text
InputStream
├── FileInputStream
├── BufferedInputStream
├── ByteArrayInputStream
├── DataInputStream
└── ObjectInputStream

OutputStream
├── FileOutputStream
├── BufferedOutputStream
├── ByteArrayOutputStream
├── DataOutputStream
└── ObjectOutputStream
```

## 字符流

```text
Reader
├── FileReader
├── BufferedReader
├── InputStreamReader
├── CharArrayReader
└── StringReader

Writer
├── FileWriter
├── BufferedWriter
├── OutputStreamWriter
├── CharArrayWriter
└── StringWriter
```

---

# 3. 字节流

字节流以 **byte（8 bit）** 为单位处理数据。

适合：

* 图片
* 视频
* 音频
* PDF
* 压缩文件
* 其他二进制文件

## FileInputStream

用于读取文件。

```java
try (FileInputStream fis = new FileInputStream("test.txt")) {
    int data;

    while ((data = fis.read()) != -1) {
        System.out.print((char) data);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

`read()` 返回：

* `0~255`：读取到的字节
* `-1`：文件读取结束

### 批量读取

效率通常比一个字节一个字节读取更高：

```java
try (FileInputStream fis = new FileInputStream("test.txt")) {
    byte[] buffer = new byte[1024];
    int len;

    while ((len = fis.read(buffer)) != -1) {
        System.out.print(new String(buffer, 0, len));
    }
}
```

注意：

```java
new String(buffer)
```

可能会把上一次读取残留的数据也转换进去，因此通常应该使用：

```java
new String(buffer, 0, len)
```

---

## FileOutputStream

用于向文件写入字节数据。

```java
try (FileOutputStream fos = new FileOutputStream("test.txt")) {
    String text = "Hello Java IO";
    fos.write(text.getBytes());
}
```

### 追加写入

```java
FileOutputStream fos =
        new FileOutputStream("test.txt", true);
```

第二个参数：

* `false`：覆盖原文件
* `true`：追加内容

---

# 4. 字符流

字符流以 **char** 为单位处理文本。

适合：

* TXT
* Java 源代码
* JSON
* XML
* CSV 等文本数据

## FileReader

```java
try (FileReader reader = new FileReader("test.txt")) {
    int data;

    while ((data = reader.read()) != -1) {
        System.out.print((char) data);
    }
}
```

## FileWriter

```java
try (FileWriter writer = new FileWriter("test.txt")) {
    writer.write("Hello Java");
}
```

追加模式：

```java
new FileWriter("test.txt", true)
```

---

# 5. 字节流和字符流的区别

| 对比   | 字节流                        | 字符流             |
| ---- | -------------------------- | --------------- |
| 基本单位 | byte                       | char            |
| 抽象父类 | InputStream / OutputStream | Reader / Writer |
| 适合   | 二进制数据                      | 文本数据            |
| 编码处理 | 不直接处理字符编码                  | 可以进行字符编码转换      |
| 典型类  | FileInputStream            | FileReader      |

一个常见理解：

```text
字节流：直接处理原始字节

字符流：面向字符，涉及编码和解码
```

---

# 6. 转换流：字节流和字符流的桥梁

这是 Java IO 中非常重要的一部分。

## InputStreamReader

将：

```text
字节输入流 → 字符输入流
```

例如指定 UTF-8：

```java
try (
    InputStreamReader reader =
        new InputStreamReader(
            new FileInputStream("test.txt"),
            StandardCharsets.UTF_8
        )
) {
    char[] buffer = new char[1024];
    int len;

    while ((len = reader.read(buffer)) != -1) {
        System.out.print(new String(buffer, 0, len));
    }
}
```

## OutputStreamWriter

将：

```text
字符输出 → 按指定编码转换为字节 → 输出
```

```java
try (
    OutputStreamWriter writer =
        new OutputStreamWriter(
            new FileOutputStream("test.txt"),
            StandardCharsets.UTF_8
        )
) {
    writer.write("你好，Java IO");
}
```

### 为什么推荐显式指定编码？

避免：

```text
Windows 默认编码
Linux 默认编码
IDE 默认编码
服务器默认编码
```

不同导致乱码。

推荐：

```java
StandardCharsets.UTF_8
```

---

# 7. 缓冲流

缓冲流的核心作用是：

> **减少实际的 IO 操作次数，提高读写效率。**

## BufferedInputStream

```java
try (
    BufferedInputStream bis =
        new BufferedInputStream(
            new FileInputStream("source.jpg")
        )
) {

    byte[] buffer = new byte[8192];
    int len;

    while ((len = bis.read(buffer)) != -1) {
        // 处理数据
    }
}
```

## BufferedOutputStream

```java
try (
    BufferedOutputStream bos =
        new BufferedOutputStream(
            new FileOutputStream("target.jpg")
        )
) {
    bos.write(data);
}
```

---

# 8. BufferedReader 和 BufferedWriter

这是文本 IO 中非常常用的组合。

## BufferedReader

最大的特点之一：

```java
readLine()
```

可以按行读取。

```java
try (
    BufferedReader br =
        new BufferedReader(
            new FileReader("test.txt")
        )
) {
    String line;

    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

注意：

`readLine()` 返回的字符串**不包含换行符**。

## BufferedWriter

```java
try (
    BufferedWriter bw =
        new BufferedWriter(
            new FileWriter("test.txt")
        )
) {
    bw.write("Hello");
    bw.newLine();
    bw.write("Java IO");
}
```

推荐使用：

```java
bw.newLine();
```

而不是直接写：

```java
"\n"
```

因为 `newLine()` 会处理不同操作系统的换行符差异。

---

# 9. 装饰器模式

Java IO 是理解 **装饰器模式（Decorator Pattern）** 的经典案例。

例如：

```java
BufferedInputStream bis =
    new BufferedInputStream(
        new FileInputStream("test.txt")
    );
```

结构：

```text
BufferedInputStream
        ↓
FileInputStream
        ↓
文件
```

再复杂一点：

```java
BufferedReader br =
    new BufferedReader(
        new InputStreamReader(
            new FileInputStream("test.txt"),
            StandardCharsets.UTF_8
        )
    );
```

结构：

```text
BufferedReader
      ↓
InputStreamReader
      ↓
FileInputStream
      ↓
文件
```

每一层负责不同功能：

| 类                 | 作用          |
| ----------------- | ----------- |
| FileInputStream   | 从文件读取字节     |
| InputStreamReader | 字节转换为字符     |
| BufferedReader    | 增加缓冲和按行读取能力 |

这就是 IO 体系非常典型的**组合优于继承**思想。

---

# 10. flush() 和 close()

这是高频知识点。

## flush()

将缓冲区中的数据立即写出。

```java
writer.write("Hello");
writer.flush();
```

调用 `flush()` 后：

* 数据被强制刷新到目标流
* 流通常仍然可以继续使用

## close()

关闭流。

```java
writer.close();
```

通常：

```text
close()
≈ flush() + 释放资源
```

关闭后不能继续操作。

例如：

```java
BufferedWriter writer = ...;

writer.close();

writer.write("Hello"); // 异常
```

### 为什么需要 flush？

因为：

```text
程序
 ↓
Java缓冲区
 ↓
操作系统缓冲
 ↓
磁盘
```

如果数据还在 Java 的缓冲区中，而程序结束或发生异常，可能导致数据没有按预期写入。

---

# 11. try-with-resources

Java 7 引入。

推荐写法：

```java
try (
    FileInputStream fis = new FileInputStream("test.txt")
) {
    // 使用资源
} catch (IOException e) {
    e.printStackTrace();
}
```

它会自动关闭资源。

传统方式：

```java
FileInputStream fis = null;

try {
    fis = new FileInputStream("test.txt");
} finally {
    if (fis != null) {
        fis.close();
    }
}
```

现在通常优先使用：

```text
try-with-resources
```

原因：

* 自动关闭资源
* 代码更简洁
* 异常处理更规范
* 避免资源泄漏

要求资源实现：

```java
AutoCloseable
```

IO 中的大部分流都支持。

---

# 12. 常见异常

## IOException

IO 操作的常见异常父类。

```java
catch (IOException e) {
    e.printStackTrace();
}
```

常见子类包括：

### FileNotFoundException

文件不存在或无法访问。

### EOFException

读取到文件末尾时发生异常，常见于 `DataInputStream` 等场景。

### UnsupportedEncodingException

字符编码不支持。

---

# 13. 文件复制

这是 Java IO 最经典的例子。

## 字节流复制文件

适合任意类型：

```java
public static void copyFile(String source, String target)
        throws IOException {

    try (
        InputStream in = new FileInputStream(source);
        OutputStream out = new FileOutputStream(target)
    ) {
        byte[] buffer = new byte[8192];
        int len;

        while ((len = in.read(buffer)) != -1) {
            out.write(buffer, 0, len);
        }
    }
}
```

为什么要：

```java
out.write(buffer, 0, len);
```

因为最后一次读取可能只填充了部分数组。

错误写法：

```java
out.write(buffer);
```

可能写入无效的残留数据。

---

# 14. DataInputStream / DataOutputStream

用于读写 Java 基本数据类型。

```java
try (
    DataOutputStream dos =
        new DataOutputStream(
            new FileOutputStream("data.bin")
        )
) {
    dos.writeInt(100);
    dos.writeDouble(3.14);
    dos.writeBoolean(true);
    dos.writeUTF("Hello");
}
```

读取时：

```java
try (
    DataInputStream dis =
        new DataInputStream(
            new FileInputStream("data.bin")
        )
) {
    System.out.println(dis.readInt());
    System.out.println(dis.readDouble());
    System.out.println(dis.readBoolean());
    System.out.println(dis.readUTF());
}
```

**读取顺序必须和写入顺序一致。**

```text
写入：
int → double → boolean

读取：
int → double → boolean
```

不能：

```text
写入：
int → double

读取：
double → int
```

否则数据解释会错误。

---

# 15. ObjectInputStream / ObjectOutputStream

用于对象序列化。

## Serializable

对象需要实现：

```java
Serializable
```

例如：

```java
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    private String name;
    private int age;
}
```

## 写对象

```java
try (
    ObjectOutputStream oos =
        new ObjectOutputStream(
            new FileOutputStream("user.dat")
        )
) {
    oos.writeObject(user);
}
```

## 读对象

```java
try (
    ObjectInputStream ois =
        new ObjectInputStream(
            new FileInputStream("user.dat")
        )
) {
    User user = (User) ois.readObject();
}
```

### transient

不希望被序列化的字段：

```java
private transient String password;
```

序列化后该字段不会保存。

---

# 16. ByteArrayInputStream / ByteArrayOutputStream

用于在**内存中操作字节数组**。

## ByteArrayOutputStream

```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();

baos.write("Hello".getBytes());

byte[] data = baos.toByteArray();
```

常见应用：

* 内存中的数据转换
* 网络数据处理
* 文件内容暂存
* HTTP 响应处理

它的特点是：

```text
不直接操作文件
数据存储在内存
```

---

# 17. CharArrayReader / CharArrayWriter

与字节数组流类似，不过处理的是字符。

```java
CharArrayWriter writer = new CharArrayWriter();

writer.write("Hello Java");

char[] chars = writer.toCharArray();
```

---

# 18. PrintStream / PrintWriter

提供方便的输出方法。

例如：

```java
System.out.println("Hello");
```

`System.out` 本质上是：

```java
PrintStream
```

`PrintWriter`：

```java
try (PrintWriter pw = new PrintWriter("test.txt")) {
    pw.println("Hello");
    pw.println("Java");
}
```

特点：

* `print()`
* `println()`
* `printf()`

适合格式化输出。

---

# 19. RandomAccessFile

`RandomAccessFile` 比较特殊，它不属于普通的 InputStream / OutputStream 体系。

特点：

> 可以直接定位到文件的任意位置进行读写。

```java
RandomAccessFile raf =
    new RandomAccessFile("test.txt", "rw");
```

移动文件指针：

```java
raf.seek(100);
```

然后：

```java
raf.write("Hello".getBytes());
```

常见模式：

```text
r   只读
rw  读写
```

应用场景：

* 大文件随机访问
* 断点续传
* 文件分块处理
* 简单索引文件

---

# 20. Java IO 中的重要设计思想

## ① 节点流

直接连接数据源。

例如：

```text
FileInputStream
FileOutputStream
FileReader
FileWriter
ByteArrayInputStream
```

示意：

```text
程序 ←→ FileInputStream ←→ 文件
```

## ② 处理流

在已有流基础上增加功能。

例如：

```text
BufferedInputStream
BufferedReader
DataInputStream
ObjectInputStream
```

示意：

```text
程序
 ↓
BufferedInputStream
 ↓
FileInputStream
 ↓
文件
```

处理流的优势：

```text
功能可以自由组合
```

---

# 21. Java IO 高频类关系图

```text
                    Java IO
                       │
          ┌────────────┴────────────┐
          │                         │
        字节流                      字符流
          │                         │
   ┌──────┴──────┐          ┌──────┴──────┐
   │             │          │             │
InputStream  OutputStream   Reader       Writer
   │             │          │             │
FileInput    FileOutput    FileReader   FileWriter
Stream       Stream
   │             │
BufferedInput BufferedOutput
Stream        Stream

                    转换流
                       │
          ┌────────────┴────────────┐
          │                         │
 InputStreamReader          OutputStreamWriter
```

---

# 22. Java IO vs NIO

除了传统 IO，Java 还有 NIO。

## Java IO

特点：

* 面向流（Stream）
* 通常是阻塞式
* 适合传统文件读写

```text
InputStream
OutputStream
Reader
Writer
```

## Java NIO

核心概念：

```text
Buffer
Channel
Selector
```

特点：

* 面向缓冲区
* Channel 双向
* 支持非阻塞 IO
* 支持 Selector 多路复用

简单对比：

| Java IO      | Java NIO            |
| ------------ | ------------------- |
| Stream       | Buffer + Channel    |
| 单向           | Channel 可双向         |
| 阻塞为主         | 可非阻塞                |
| 一个线程处理一个阻塞连接 | 可通过 Selector 管理多个连接 |

需要注意：

> **文件读写不一定意味着必须使用 NIO，普通 Java IO 在很多场景下已经足够。**

---

# 23. IO 面试高频问题

### 1. 字节流和字符流有什么区别？

核心：

```text
字节流：处理原始二进制数据
字符流：处理字符文本，涉及字符编码
```

---

### 2. 为什么 BufferedReader 比 FileReader 效率高？

因为：

```text
BufferedReader 内部维护缓冲区
```

减少底层实际 IO 操作次数。

---

### 3. flush() 和 close() 的区别？

```text
flush()：刷新缓冲区，但流还能继续使用
close()：关闭流并释放资源，之后不能继续使用
```

---

### 4. 为什么最后一次读取要使用 len？

例如：

```java
while ((len = input.read(buffer)) != -1) {
    output.write(buffer, 0, len);
}
```

因为最后一次可能只读取：

```text
100 字节
```

但：

```java
buffer.length = 1024
```

如果直接写整个数组，会把无效数据也写进去。

---

### 5. 为什么需要转换流？

因为：

```text
底层存储 → 字节
文本操作 → 字符
```

转换流负责：

```text
字节 ↔ 字符
```

同时可以处理：

```text
UTF-8
GBK
ISO-8859-1
```

等编码。

---

# 24. 推荐学习路线

建议按照下面顺序学习：

```text
1. File
   ↓
2. InputStream / OutputStream
   ↓
3. FileInputStream / FileOutputStream
   ↓
4. Reader / Writer
   ↓
5. FileReader / FileWriter
   ↓
6. BufferedInputStream / BufferedOutputStream
   ↓
7. BufferedReader / BufferedWriter
   ↓
8. InputStreamReader / OutputStreamWriter
   ↓
9. DataInputStream / DataOutputStream
   ↓
10. ObjectInputStream / ObjectOutputStream
   ↓
11. RandomAccessFile
   ↓
12. NIO：Buffer / Channel / Selector
```

## 一句话总结

可以把 Java IO 记成下面这张思维框架：

```text
Java IO
│
├── 按单位
│   ├── 字节流：InputStream / OutputStream
│   └── 字符流：Reader / Writer
│
├── 按功能
│   ├── 节点流：直接连接数据源
│   └── 处理流：增强已有流功能
│
├── 常见增强
│   ├── Buffered：缓冲
│   ├── Data：基本类型读写
│   ├── Object：对象序列化
│   ├── Print：方便打印
│   └── 转换流：字节 ↔ 字符
│
└── 资源管理
    ├── flush()
    ├── close()
    └── try-with-resources
```

可以。前面的确展开得太多了。下面给你一个**适合做笔记/面试复习的精简版 NIO**。

# Java NIO 简洁总结

## 1. NIO 三大核心

```text
Buffer   → 存数据
Channel  → 传输数据
Selector → 监听多个 Channel
```

记忆：

> **Channel 搬数据，Buffer 装数据，Selector 管多个连接。**

---

## 2. Buffer

最常用：`ByteBuffer`

核心属性：

```text
capacity  最大容量
position  当前读写位置
limit     当前操作上限
```

核心方法：

```text
put()      写入
get()      读取
flip()     写 → 读
clear()    重新写
rewind()   从头读
compact()  保留未读数据，继续写
```

最重要：

```text
写数据
  ↓
flip()
  ↓
读数据
  ↓
clear()
  ↓
重新写
```

---

## 3. Channel

常见：

```text
FileChannel
SocketChannel
ServerSocketChannel
DatagramChannel
```

和传统 IO 的区别：

```text
IO：
InputStream / OutputStream

NIO：
Channel + Buffer
```

`FileChannel` 常用于文件读写和文件复制。

---

## 4. Selector

Selector 用于：

> **一个线程管理多个网络 Channel。**

例如：

```text
             Selector
          /      |      \
     Channel  Channel  Channel
        ↓        ↓        ↓
       有数据    无数据    有数据
```

只处理当前真正就绪的 Channel。

常见事件：

```text
OP_ACCEPT  有新连接
OP_CONNECT 连接完成
OP_READ    可以读取
OP_WRITE   可以写入
```

---

## 5. NIO 网络编程核心

```text
ServerSocketChannel
        ↓
configureBlocking(false)
        ↓
Selector
        ↓
register(OP_ACCEPT)
        ↓
select()
        ↓
SelectionKey
        ↓
处理 READ / WRITE / ACCEPT
```

核心思想：

> **非阻塞 + IO 多路复用。**

---

## 6. Path 和 Files

Java 7 的 NIO.2。

```java
Path path = Path.of("test.txt");

Files.exists(path);
Files.readString(path);
Files.writeString(path, "Hello");
Files.copy(source, target);
Files.delete(path);
```

记忆：

```text
Path  → 表示路径
Files → 操作文件
```

新代码一般优先：

```text
Path + Files
```

而不是老的：

```text
File
```

---

# 7. IO vs NIO

|    | IO            | NIO              |
| -- | ------------- | ---------------- |
| 核心 | Stream        | Buffer + Channel |
| 模型 | 阻塞为主          | 支持非阻塞            |
| 网络 | 一个线程处理一个连接较常见 | 一个线程可管理多个连接      |
| 重点 | 简单读写          | 高并发 IO           |

---

# 8. 面试只记这张图

```text
Java NIO
│
├── Buffer
│   └── ByteBuffer
│       ├── flip()
│       ├── clear()
│       └── compact()
│
├── Channel
│   ├── FileChannel
│   ├── SocketChannel
│   └── ServerSocketChannel
│
├── Selector
│   ├── OP_ACCEPT
│   ├── OP_READ
│   ├── OP_WRITE
│   └── OP_CONNECT
│
└── NIO.2
    ├── Path
    └── Files
```

**一句话：**

> **NIO = Buffer + Channel + Selector；核心价值是通过非阻塞和 IO 多路复用提高网络 IO 的并发处理能力。**

## I/O 面试题

### BIO/NIO/AIO 区别

- BIO（blocking IO）：**同步阻塞**，线程调用 read() 后原地等待，直到数据到达内核并拷贝到程序内存，期间线程无法做任何事。
- NIO（non-blocking IO）核心是 Reactor模式，一个Selector（选择器）线程会负责管理成百上千个Channel，可以构建**多路复用、同步非阻塞** IO 程序，线程发起读请求后立即返回，通过循环不断轮询内核数据是否就绪，期间线程可以处理其他轻量任务。
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