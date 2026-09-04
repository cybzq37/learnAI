# Netty 知识点总结

> Netty 是一个基于 Java NIO 的高性能网络通信框架，常用于 TCP、HTTP、WebSocket、RPC、IM、网关等网络应用开发。

---

## 一、Netty 核心知识体系

```text
Netty
│
├── Reactor 线程模型
│   ├── Boss EventLoopGroup
│   └── Worker EventLoopGroup
│
├── Channel
│
├── EventLoop
│
├── ChannelPipeline
│   └── ChannelHandler
│
├── ByteBuf
│
├── 编解码器
│
├── Future / Promise
│
├── 心跳机制
│
├── 粘包 / 拆包
│
└── 内存池 / 零拷贝
```

---

# 二、Channel

`Channel` 可以理解为客户端与服务器之间的一条网络连接。

常见实现：

```java
NioServerSocketChannel
NioSocketChannel
```

主要职责：

* 建立连接
* 读取数据
* 写入数据
* 关闭连接
* 获取连接状态

常见 API：

```java
channel.writeAndFlush(msg);
channel.close();
channel.isActive();
channel.remoteAddress();
```

---

# 三、EventLoop

`EventLoop` 是 Netty 最核心的组件之一。

可以理解为：

> 一个不断处理 IO 事件和任务的事件循环线程。

简化模型：

```java
while (running) {
    // 获取 IO 事件
    // 处理 IO 事件
    // 执行任务
}
```

典型关系：

```text
EventLoop
   │
   ├── Channel A
   ├── Channel B
   ├── Channel C
   └── Channel D
```

通常一个 `EventLoop` 负责多个 `Channel`。

因此，同一个 Channel 的事件通常由同一个 EventLoop 线程处理。

### 优点

* 减少线程创建
* 减少线程上下文切换
* 降低锁竞争
* 提高并发处理能力

---

# 四、EventLoopGroup

`EventLoopGroup` 是 `EventLoop` 的集合。

典型服务端代码：

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

典型结构：

```text
BossGroup
    │
    ↓
Accept 新连接
    │
    ↓
WorkerGroup
    │
    ├── Channel A
    ├── Channel B
    ├── Channel C
    └── Channel D
```

### BossGroup

主要负责：

```text
Accept 新连接
```

### WorkerGroup

主要负责：

```text
Read
Write
IO 事件处理
```

---

# 五、Reactor 线程模型

## 1. 单 Reactor 单线程

```text
Reactor
   ↓
Accept
   ↓
Read
   ↓
业务处理
   ↓
Write
```

优点：

* 简单

缺点：

* 无法充分利用多核 CPU
* 业务阻塞会影响整个系统

---

## 2. 单 Reactor 多线程

```text
Reactor
   ↓
Accept / IO
   ↓
Worker Thread Pool
   ↓
业务处理
```

IO 和业务处理分离。

---

## 3. 主从 Reactor 多线程

Netty 常见模型：

```text
                 Boss
                  │
                  ↓
                Accept
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
 Worker EventLoop      Worker EventLoop
        ↓                   ↓
    Channel A            Channel B
```

Boss 负责接收连接，Worker 负责处理网络 IO。

---

# 六、ChannelPipeline

`ChannelPipeline` 是 Netty 非常核心的设计。

可以理解为：

> 一条 Channel 上的数据处理流水线。

例如：

```text
客户端
  ↓
ByteBuf
  ↓
Decoder
  ↓
业务 Handler
  ↓
Encoder
  ↓
ByteBuf
  ↓
客户端
```

典型代码：

```java
pipeline.addLast(new StringDecoder());
pipeline.addLast(new StringEncoder());
pipeline.addLast(new MyServerHandler());
```

Pipeline 中主要有：

```text
ChannelInboundHandler
ChannelOutboundHandler
```

---

# 七、Inbound 和 Outbound

## Inbound

表示入站事件。

例如：

```text
Client
  ↓
Server
```

常见方法：

```java
channelActive()
channelRead()
channelReadComplete()
channelInactive()
exceptionCaught()
```

---

## Outbound

表示出站事件。

例如：

```text
Server
  ↓
Client
```

常见操作：

```java
write()
flush()
connect()
close()
```

### 记忆

```text
数据进入服务器 → Inbound

数据离开服务器 → Outbound
```

---

# 八、ChannelHandler

`ChannelHandler` 用来处理 Channel 上的数据和事件。

示例：

```java
public class MyHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println(msg);
        ctx.fireChannelRead(msg);
    }
}
```

常见 Handler：

```text
ChannelInboundHandlerAdapter
SimpleChannelInboundHandler
ChannelOutboundHandlerAdapter
```

---

# 九、ChannelHandlerContext

`ChannelHandlerContext` 是 Handler 与 Pipeline、Channel 之间的重要桥梁。

常见方法：

```java
ctx.channel();
ctx.write();
ctx.writeAndFlush();
ctx.fireChannelRead(msg);
ctx.close();
```

重点区别：

```java
ctx.write(msg);
```

和：

```java
channel.write(msg);
```

### ctx.write()

通常从当前 Handler 所在位置向前寻找 OutboundHandler。

### channel.write()

通常从 Pipeline 尾部开始寻找 OutboundHandler。

---

# 十、ByteBuf

Netty 使用自己的数据容器：

```java
ByteBuf
```

它比 Java NIO 的 `ByteBuffer` 更适合网络编程。

---

## 1. ByteBuf 核心索引

主要包括：

```text
readerIndex
writerIndex
capacity
```

可以理解为：

```text
| 已读数据 | 可读数据 | 可写空间 |
            ↑        ↑
        reader     writer
```

例如：

```text
readerIndex = 0
writerIndex = 5
```

表示：

```text
0 ~ 4
```

的数据可读。

```text
5 ~ capacity - 1
```

的空间可写。

---

## 2. 写数据

```java
buf.writeInt(100);
buf.writeByte(1);
buf.writeBytes(bytes);
```

---

## 3. 读数据

```java
buf.readInt();
buf.readByte();
buf.readBytes(...);
```

---

## 4. 随机访问

```java
buf.getInt(0);
```

随机访问不会修改：

```text
readerIndex
writerIndex
```

---

# 十一、ByteBuf 类型

主要可以按照两个维度分类：

```text
Heap / Direct
Pooled / Unpooled
```

常见组合：

```text
PooledHeapByteBuf
PooledDirectByteBuf
UnpooledHeapByteBuf
UnpooledDirectByteBuf
```

---

# 十二、Direct ByteBuf

Direct Buffer 位于 JVM 堆外内存。

传统方式：

```text
Heap
 ↓
Native
 ↓
IO
```

Direct Buffer 可以减少部分堆内存与本地内存之间的数据复制。

### 优点

* 网络 IO 场景下性能较好
* 减少部分数据复制

### 缺点

* 堆外内存管理复杂
* 分配和释放成本更高

所以并不是所有场景都一定应该使用 Direct Buffer。

---

# 十三、ByteBuf 引用计数

Netty 使用引用计数管理部分 ByteBuf 生命周期。

常见方法：

```java
retain();
release();
```

基本思想：

```text
引用增加
   ↓
retain()

引用减少
   ↓
release()

引用计数变为 0
   ↓
释放资源
```

### 常见问题

> ByteBuf 内存泄漏。

通常与 `release()` 没有正确调用有关。

因此要重点关注：

```text
谁创建
谁持有
谁释放
```

---

# 十四、Netty 内存池

Netty 提供：

```java
PooledByteBufAllocator
```

核心思想：

```text
提前申请一大块内存
        ↓
切分成多个小块
        ↓
重复利用
```

### 优势

* 减少内存分配次数
* 降低 GC 压力
* 提高性能

---

# 十五、Netty 零拷贝

Netty 中常见的零拷贝相关机制：

## 1. CompositeByteBuf

多个 ByteBuf 组合成一个逻辑 Buffer：

```text
Buffer A
Buffer B
Buffer C
   ↓
CompositeByteBuf
```

避免重新复制数据。

---

## 2. FileRegion

文件传输时可以利用操作系统的 `sendfile` 等能力，减少用户态和内核态之间的数据复制。

---

## 3. slice()

```java
buf.slice();
```

共享底层内存。

---

## 4. duplicate()

```java
buf.duplicate();
```

共享数据，但拥有独立的索引。

---

# 十六、TCP 粘包与拆包

TCP 是**字节流协议**，不提供应用层消息边界。

例如发送：

```text
Hello
World
```

接收端不一定一次收到：

```text
Hello
World
```

可能收到：

```text
Hell
oWor
ld
```

也可能收到：

```text
HelloWorld
```

这就是：

```text
拆包
粘包
```

---

# 十七、为什么会发生粘包 / 拆包

应用层：

```text
Message A
Message B
```

TCP 层看到的是：

```text
111111111111111111111111
```

TCP 只保证：

* 可靠传输
* 有序传输
* 不重复

但不会告诉应用层：

```text
Message A 到哪里结束
Message B 从哪里开始
```

因此必须由应用层自己定义消息边界。

---

# 十八、Netty 如何解决粘包 / 拆包

## 1. 固定长度

使用：

```java
FixedLengthFrameDecoder
```

例如：

```text
每条消息固定 20 字节
```

---

## 2. 分隔符

使用：

```java
DelimiterBasedFrameDecoder
```

例如：

```text
Hello\n
World\n
```

通过：

```text
\n
```

确定消息边界。

---

## 3. 长度字段

最常见的方式之一：

```text
| length | message body |
```

例如：

```text
| 0005 | Hello |
| 0005 | World |
```

对应 Netty：

```java
LengthFieldBasedFrameDecoder
```

实际 RPC、自定义二进制协议中非常常见。

---

# 十九、Netty 编解码器

编解码器主要负责：

```text
ByteBuf
   ↓
Java 对象
```

以及：

```text
Java 对象
   ↓
ByteBuf
```

常见：

```text
ByteToMessageDecoder
MessageToByteEncoder
LengthFieldBasedFrameDecoder
StringDecoder
StringEncoder
```

---

# 二十、ByteToMessageDecoder

主要负责：

```text
ByteBuf → Java 对象
```

示例：

```java
public class MyDecoder extends ByteToMessageDecoder {

    @Override
    protected void decode(
            ChannelHandlerContext ctx,
            ByteBuf in,
            List<Object> out) {

        // decode
    }
}
```

注意：

> `decode()` 一次不一定只对应一条消息。

可能：

```text
一次 read
    ↓
半条消息
```

也可能：

```text
一次 read
    ↓
三条消息
```

因此 Decoder 必须正确处理 TCP 字节流。

---

# 二十一、Netty Handler 执行线程

通常：

```text
Channel
   ↓
EventLoop
   ↓
EventLoop Thread
   ↓
Pipeline Handler
```

因此：

```java
channelRead()
```

通常由对应 Channel 的 EventLoop 线程执行。

---

# 二十二、为什么不能阻塞 EventLoop

因为一个 EventLoop 通常负责多个 Channel。

如果在 Handler 中执行：

```java
Thread.sleep();
```

或者：

```text
数据库慢查询
RPC 阻塞调用
文件 IO
复杂计算
```

那么可能导致：

```text
Channel A 阻塞
        ↓
EventLoop 阻塞
        ↓
其他 Channel 也受到影响
```

所以：

> EventLoop 中应该尽量执行短小、非阻塞的任务。

---

# 二十三、业务线程池

如果业务逻辑比较耗时，可以交给独立线程池处理：

```text
IO线程
  ↓
快速读取
  ↓
提交业务线程池
  ↓
业务处理
  ↓
返回结果
```

例如：

```java
EventExecutorGroup businessGroup =
        new DefaultEventExecutorGroup(16);

pipeline.addLast(
        businessGroup,
        new BusinessHandler()
);
```

---

# 二十四、Future 和 Promise

Netty 大量使用异步编程模型。

例如：

```java
ChannelFuture future = channel.writeAndFlush(msg);
```

可以添加监听器：

```java
future.addListener(f -> {
    if (f.isSuccess()) {
        System.out.println("发送成功");
    }
});
```

核心思想：

```text
任务开始
   ↓
立即返回 Future
   ↓
任务执行
   ↓
任务完成
   ↓
Listener 得到通知
```

---

# 二十五、Bootstrap

Netty 启动服务主要使用：

```text
ServerBootstrap
Bootstrap
```

### ServerBootstrap

服务端：

```java
ServerBootstrap
```

### Bootstrap

客户端：

```java
Bootstrap
```

典型服务端代码：

```java
ServerBootstrap bootstrap = new ServerBootstrap();

bootstrap
    .group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childHandler(...)
    .bind(8080);
```

---

# 二十六、ServerBootstrap 常见配置

常见方法：

```java
.group(...)
.channel(...)
.option(...)
.childOption(...)
.handler(...)
.childHandler(...)
.bind(...)
```

---

## option 和 childOption

这是面试常问知识点。

### option

配置 ServerChannel。

例如：

```java
.option(ChannelOption.SO_BACKLOG, 1024)
```

### childOption

配置新建立的客户端连接 Channel。

例如：

```java
.childOption(ChannelOption.SO_KEEPALIVE, true)
```

记忆：

```text
option
   ↓
ServerSocketChannel

childOption
   ↓
SocketChannel
```

---

# 二十七、常见 ChannelOption

常见参数：

```java
SO_BACKLOG
SO_KEEPALIVE
TCP_NODELAY
SO_SNDBUF
SO_RCVBUF
CONNECT_TIMEOUT_MILLIS
```

---

## TCP_NODELAY

控制是否启用 Nagle 算法。

关闭 Nagle：

```text
降低小数据包延迟
```

但可能：

```text
增加网络数据包数量
```

---

# 二十八、HTTP

Netty 可以快速构建 HTTP 服务。

常见组件：

```java
HttpServerCodec
HttpObjectAggregator
```

典型 Pipeline：

```text
HttpServerCodec
      ↓
HttpObjectAggregator
      ↓
业务 Handler
```

### HttpServerCodec

同时提供：

```text
HTTP Encoder
HTTP Decoder
```

### HttpObjectAggregator

将多个：

```text
HttpMessage
HttpContent
LastHttpContent
```

聚合为：

```text
FullHttpRequest
```

---

# 二十九、WebSocket

Netty 非常适合构建 WebSocket 长连接服务。

典型 Pipeline：

```text
HttpServerCodec
      ↓
HttpObjectAggregator
      ↓
WebSocketServerProtocolHandler
      ↓
业务 Handler
```

常见应用：

* 在线聊天
* 实时推送
* 在线协作
* 游戏
* 实时监控

---

# 三十、心跳机制

长连接服务需要判断：

> 对端是否仍然存活。

Netty 提供：

```java
IdleStateHandler
```

例如：

```java
new IdleStateHandler(
    60,
    0,
    0,
    TimeUnit.SECONDS
)
```

可以检测：

```text
读空闲
写空闲
读写都空闲
```

示例：

```java
@Override
public void userEventTriggered(
        ChannelHandlerContext ctx,
        Object evt) {

    if (evt instanceof IdleStateEvent) {
        ctx.close();
    }
}
```

---

# 三十一、Channel 生命周期

典型生命周期：

```text
Channel 创建
    ↓
channelRegistered
    ↓
channelActive
    ↓
channelRead
    ↓
channelReadComplete
    ↓
channelInactive
    ↓
channelUnregistered
```

常见生命周期方法：

```java
channelActive()
channelInactive()
exceptionCaught()
```

---

# 三十二、异常处理

常用方法：

```java
exceptionCaught(...)
```

例如：

```java
@Override
public void exceptionCaught(
        ChannelHandlerContext ctx,
        Throwable cause) {

    cause.printStackTrace();
    ctx.close();
}
```

实际项目中可以在 Pipeline 中统一进行异常处理、日志记录和连接关闭。

---

# 三十三、Netty 与 BIO / NIO 对比

## BIO

```text
一个连接
   ↓
一个线程
```

高并发情况下容易出现：

```text
线程数量过多
线程上下文切换严重
```

---

## NIO

```text
Selector
   ↓
一个线程
   ↓
管理多个 Channel
```

相比 BIO 可以大幅减少线程数量。

---

## Netty

Netty 在 NIO 基础上进一步封装：

```text
NIO
 ↓
Netty
 ↓
Reactor 线程模型
Channel
Pipeline
Handler
ByteBuf
编解码
内存池
零拷贝
异常处理
连接管理
```

---

# 三十四、Netty 为什么性能高

常见原因：

## 1. Reactor 线程模型

减少线程数量和线程上下文切换。

## 2. 异步非阻塞 IO

一个线程可以处理大量连接。

## 3. ByteBuf

针对网络 IO 进行了优化。

## 4. 内存池

减少频繁内存分配。

## 5. 零拷贝

减少不必要的数据复制。

## 6. Pipeline

将网络处理逻辑模块化。

## 7. 高效事件循环

统一管理 IO 事件和任务。

---

# 三十五、Netty 核心架构图

```text
                    Server
                      │
               ServerBootstrap
                      │
                Boss EventLoop
                      │
                   Accept
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
      Channel A   Channel B   Channel C
          │           │           │
      EventLoop   EventLoop   EventLoop
          │           │           │
      Pipeline    Pipeline    Pipeline
          │
    ┌─────┼──────────────┐
    ↓     ↓              ↓
 Decoder Handler       Encoder
    │     │              │
    └─────┼──────────────┘
          ↓
        ByteBuf
          ↓
        Socket
```

---

# 三十六、Netty 高频面试题

## 基础

1. Netty 是什么？
2. Netty 为什么比 BIO 性能高？
3. Netty 和 NIO 有什么关系？
4. Bootstrap 和 ServerBootstrap 有什么区别？
5. Netty 为什么适合高并发网络编程？

---

## Reactor / EventLoop

6. Netty 的线程模型是什么？
7. BossGroup 和 WorkerGroup 分别做什么？
8. EventLoop 是什么？
9. EventLoopGroup 是什么？
10. 一个 EventLoop 可以管理多少个 Channel？
11. 一个 Channel 的 Handler 默认在哪个线程执行？
12. 为什么 EventLoop 模型可以减少锁竞争？
13. 为什么不能在 EventLoop 中执行阻塞操作？

---

## Pipeline / Handler

14. ChannelPipeline 是什么？
15. ChannelHandler 是什么？
16. Inbound 和 Outbound 有什么区别？
17. Handler 的执行顺序是什么？
18. ChannelHandlerContext 是什么？
19. `ctx.write()` 和 `channel.write()` 有什么区别？
20. `fireChannelRead()` 的作用是什么？

---

## ByteBuf

21. ByteBuf 和 ByteBuffer 有什么区别？
22. readerIndex 和 writerIndex 是什么？
23. Heap ByteBuf 和 Direct ByteBuf 有什么区别？
24. Pooled 和 Unpooled 有什么区别？
25. Netty 为什么使用内存池？
26. 什么是引用计数？
27. retain() 和 release() 有什么作用？
28. ByteBuf 内存泄漏如何产生？

---

## TCP

29. 什么是 TCP 粘包？
30. 什么是 TCP 拆包？
31. TCP 为什么会出现粘包和拆包？
32. Netty 如何解决粘包和拆包？
33. FixedLengthFrameDecoder 是什么？
34. DelimiterBasedFrameDecoder 是什么？
35. LengthFieldBasedFrameDecoder 是什么？
36. 自定义协议为什么经常设计 Length 字段？

---

## 编解码

37. Encoder 和 Decoder 的作用是什么？
38. ByteToMessageDecoder 是什么？
39. MessageToByteEncoder 是什么？
40. decode() 为什么可能一次解析多条消息？
41. decode() 为什么可能只能解析半条消息？

---

## 高级

42. Netty 的零拷贝是什么？
43. CompositeByteBuf 是什么？
44. FileRegion 是什么？
45. slice() 和 duplicate() 有什么区别？
46. Netty 内存池原理是什么？
47. IdleStateHandler 有什么作用？
48. Netty 如何实现心跳检测？
49. Future 和 Promise 是什么？
50. Netty 如何优雅关闭？
51. Netty 中如何处理慢任务？
52. Netty 为什么不建议在 EventLoop 中执行阻塞业务？

---

# 三十七、Netty 学习路线

建议按照下面顺序学习：

```text
第一阶段：NIO 基础
    ↓
Channel
    ↓
EventLoop
    ↓
EventLoopGroup

第二阶段：Pipeline
    ↓
ChannelHandler
    ↓
Inbound / Outbound

第三阶段：ByteBuf
    ↓
readerIndex / writerIndex
    ↓
引用计数
    ↓
内存池
    ↓
零拷贝

第四阶段：TCP
    ↓
粘包 / 拆包
    ↓
Encoder / Decoder
    ↓
LengthFieldBasedFrameDecoder

第五阶段：高级特性
    ↓
心跳
    ↓
WebSocket
    ↓
HTTP
    ↓
自定义协议

第六阶段：源码
    ↓
NioEventLoop
    ↓
SingleThreadEventExecutor
    ↓
ChannelPipeline
    ↓
AbstractChannel
    ↓
ByteBuf
    ↓
内存池
```

---

# 三十八、最核心的 8 个知识点

如果是为了 Java 后端面试，优先搞懂下面 8 个：

```text
1. EventLoop
2. Channel
3. ChannelPipeline
4. ChannelHandler
5. ByteBuf
6. 编解码器
7. TCP 粘包 / 拆包
8. Reactor 线程模型
```

一句话串起来：

> **Channel 负责网络连接，EventLoop 负责事件循环，Pipeline 负责处理链路，Handler 负责具体逻辑，ByteBuf 负责数据传输，Decoder / Encoder 负责协议转换，Reactor 负责高并发 IO 模型。**
