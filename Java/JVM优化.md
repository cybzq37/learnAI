## JVM优化思路

1. **尽量让每次Young GC后的存活对象小于Survivor区域的50%**，让对象都留在年轻代里（避免过早晋升）。
2. **尽量别让对象进入老年代**（减少老年代GC压力）。
3. **尽量减少Full GC的频率**，避免频繁Full GC对JVM性能造成影响。

在进行优化前，需要先用 `jstat -gc -pid` 命令计算出一些关键数据：
*   Young GC次数和耗时
*   Full GC次数和耗时
*   堆内存大小、年轻代大小、Eden和Survivor的比例、老年代大小、大对象阈值、大龄对象进入老年代的阈值等。
*   **目的：** 有了这些数据，才能设定初始的JVM参数。

**1. 年轻代对象增长的速率**
*   **命令：** 执行 `jstat -gc pid 1000 10`（每隔1秒执行1次命令，共执行10次）。
*   **观察：** 观察EU（Eden区使用量）的变化。
*   **计算：** 估算每秒Eden大概新增多少对象。
*   **注意：** 系统有高峰期和日常期，需要在不同的时间分别估算，如系统负载不高，可以把频率换成1分钟或10分钟来观察。

**2. Young GC的触发频率和每次耗时**
*   **触发频率：** 知道Eden区大小和对象增长速率，就能推算出Young GC大概多久触发一次。
*   **耗时：** 通过公式 **YGCT/YGC** 算出平均耗时。
*   **目的：** 评估系统多久会因为Young GC的执行而卡顿。

**3. 每次Young GC后有多少对象存活和进入老年代**
*   **场景：** 假设已知Young GC频率（如5分钟一次）。
*   **命令：** 执行 `jstat -gc pid 300000 10`（每隔5分钟执行一次，共10次）。
*   **观察：** 观察每次GC后 Eden、Survivor 和 老年代 使用量的变化。
*   **规律：** GC后Eden区使用一般会大幅减少，Survivor和老年代可能会增长（增长的部分即为存活并晋升的对象）。
*   **计算：** 推算出老年代对象增长速率。

**4. Full GC的触发频率和每次耗时**
*   **触发频率：** 知道老年代对象的增长速率后，就能推算出Full GC的触发频率。
*   **耗时：** Full GC的每次耗时可以用公式 **FGCT/FGC** 计算得出。

这套思路偏向于**传统的“经验主义调优”**，在现代 Java 应用中存在以下几个盲区：

**1. 适用场景的局限（分代 vs 分区）：**
*   这套思路强烈依赖于“年轻代/老年代”的物理划分。
*   **但在 G1 GC 成为主流（JDK 9+ 默认）的今天**，G1 虽然在逻辑上保留分代，但物理内存被划分为一个个 Region，并且有专门的 Humongous（大对象）区域。G1 的调优参数（如 `MaxGCPauseMillis` 目标停顿时间）和观察重点与这套传统思路有很大不同。**如果是 ZGC 或 Shenandoah 这种不分代的收集器，这套思路基本完全失效。**

**2. “减少 Full GC”这个说法太绝对：**
*   对于 CMS 等收集器，减少 Full GC 是对的。
*   但在 G1 中，`Full GC` 通常意味着 G1 的自适应策略失效（比如对象分配过快，导致并发标记来不及完成），退化成了 Serial Old GC。此时重点不是“减少频率”，而是要排查**是否发生了内存泄漏**，或者**是否发生了大对象分配（Humongous Allocation）**导致频繁触发。

**3. 盲目追求“存活对象小于Survivor的50%”可能适得其反：**
*   如果为了达到这个目标，把 Survivor 区调得极大，或者把晋升年龄（`MaxTenuringThreshold`）调得极高，会导致：
    *   年轻代单次 GC 扫描时间变长（停顿时间增加）。
    *   复制算法的成本变高。
    *   极端情况下，长期存活的对象在 Survivor 之间来回复制，浪费 CPU 和内存带宽。
*   **正确做法：** 应该根据对象的**真实生命周期**来设定。如果业务就是会产生大量中等生命周期的对象，强行让它们在 Survivor 里熬着，不如让它们早点进老年代，只要老年代不发生频繁 Full GC 即可。

**4. 忽略了现代监控工具：**
*   笔记主要依赖 `jstat`。虽然 `jstat` 很轻量，但现代 JVM 调优更应该结合可视化工具（如 **JVisualVM, JConsole, Arthas, Prometheus + Grafana**）以及 **GC 日志（`-Xlog:gc*`）**。GC 日志能提供更精确的停顿时间分布（P99, P999），而不仅仅是平均值。

**5. 忽略了“应用层优化”才是王道：**
*   JVM 调优永远是最后的手段。笔记完全聚焦于 JVM 底层参数。但在实际工作中，**90% 的 GC 问题是因为代码写得烂**（如：无限循环创建对象、大对象直接分配、缓存设计不合理导致内存泄漏）。不解决代码问题，只调 JVM 参数，属于治标不治本。

但如果要在生产环境中真正做到“合理”，建议补充以下几点：

1. **结合具体垃圾回收器：** 明确你的应用用的是 CMS、G1 还是 ZGC。如果是 G1/ZGC，忘掉单纯的 Eden/Survivor 比例调优，转向**停顿时间目标（Pause Time Goal）**调优。
2. **区分“调优”与“排障”：** 如果发生频繁 Full GC，第一步永远是 `jmap -histo:live` 或 `jmap -dump` 导出堆内存，看看**到底是什么对象占满了老年代**，而不是急着去改 JVM 参数。
3. **关注 P99 延迟：** 平均 GC 耗时（`YGCT/YGC`）参考价值有限，真正影响用户体验的是长尾延迟（比如某次 Young GC 耗时 500ms）。
4. **代码先行：** 优先排查代码中不合理的对象创建、未关闭的资源、过大的本地缓存等。

**一句话评价：思路是对的，是传统时代的经典打法，但在现代 Java 环境下需要结合具体的 GC 器和业务代码进行升级。**

## Promethues 监控内存

Prometheus 本身只是一个时序数据库和监控告警系统，它不能直接“读取”JVM的内部状态。但是，通过**Exporter（导出器）**，Prometheus 可以完美地抓取并存储 JVM 的各项内存指标。

在 Java 生态中，目前最主流、最标准的做法是使用 **JMX Exporter**。

以下是 Prometheus 监控 JVM 内存的具体实现原理、能监控到的指标以及架构流程：

Java 应用默认会暴露 JMX（Java Management Extensions）接口，里面包含了 JVM 的所有运行时数据。Prometheus 监控 JVM 的典型架构如下：

1. **Java 应用端：** 引入 `jmx_prometheus_javaagent`（一个 jar 包），在启动 Java 应用时通过 `-javaagent` 参数挂载这个 agent。它会将 JVM 的 JMX 数据转换成 Prometheus 能识别的格式，并暴露一个 HTTP 端口（比如 9404）。
2. **Prometheus 端：** 在 `prometheus.yml` 配置文件中，添加一个 `scrape_config`，定时去抓取（Pull）Java 应用暴露的 9404 端口数据。
3. **可视化：** 配合 **Grafana**，导入现成的 JVM 监控大盘（如 Grafana 官方模板 ID: 4701 或 8563），就能看到非常漂亮的 JVM 内存、GC、线程等图表。

*(注：如果是 Spring Boot 应用，还可以使用 Micrometer + Spring Boot Actuator，通过 `/actuator/prometheus` 端点暴露指标，这也是目前非常流行的方式。)*

一旦接入，你可以获取到非常详细的、分代的内存数据。主要包含以下几类：

**1. 堆内存（Heap Memory）总体情况**
*   `jvm_memory_used_bytes{area="heap"}`：当前堆内存已使用大小。
*   `jvm_memory_committed_bytes{area="heap"}`：当前堆内存已提交大小（向操作系统申请到的内存）。
*   `jvm_memory_max_bytes{area="heap"}`：堆内存最大限制（对应 `-Xmx`）。

**2. 年轻代与老年代细分（通过 `id` 标签区分）**
这是调优最关心的数据，Prometheus 可以精确抓取到各个区域：
*   **Eden 区：** `jvm_memory_used_bytes{area="heap", id="PS Eden Space"}` (具体 id 名称取决于你使用的 GC 收集器，如 G1 则是 `G1 Eden Space`)。
*   **Survivor 区：** `jvm_memory_used_bytes{area="heap", id="PS Survivor Space"}`。
*   **老年代：** `jvm_memory_used_bytes{area="heap", id="PS Old Gen"}`。

**3. 非堆内存（Non-Heap Memory）**
*   **元空间（Metaspace）：** `jvm_memory_used_bytes{area="nonheap", id="Metaspace"}`。
*   **压缩类空间：** `jvm_memory_used_bytes{area="nonheap", id="Compressed Class Space"}`。
*   **代码缓存：** `jvm_memory_used_bytes{area="nonheap", id="Code Cache"}`。

**4. 内存池缓冲（Buffer Pools）**
*   直接内存（Direct Memory）：`jvm_memory_used_bytes{id="direct"}`。
*   Mapped 内存：`jvm_memory_used_bytes{id="mapped"}`。

**5. 内存分配速率（极其重要）**
*   `jvm_memory_pool_allocated_bytes_total`：这是**累计分配**的字节数。通过 Prometheus 的 `rate()` 函数，可以算出**每秒新增对象占用的内存大小**（即上一份笔记中提到的“年轻代对象增长速率”）。这是判断系统内存压力的核心指标。

如果把 Prometheus 引入到上一篇笔记的优化流程中，你会发现**一切都可以自动化、可视化**：

*   之前用 `jstat -gc pid 1000 10` 手动计算 Eden 增长率。
*   现在只需在 Grafana 里配置一个图表：`rate(jvm_memory_pool_allocated_bytes_total{pool="G1 Eden Space"}[1m])`，就能实时看到每秒钟 Eden 区分配了多少内存。
*   之前手动算 `FGCT/FGC`。
*   现在可以直接监控 `jvm_gc_pause_seconds_sum / jvm_gc_pause_seconds_count`，并且可以设置告警：当 Full GC 平均耗时超过 1 秒，或者 5 分钟内 Full GC 次数超过 3 次时，直接触发钉钉/企微告警。

这些是 JDK 自带的 JVM 诊断工具，位于 `$JAVA_HOME/bin` 下。它们分别覆盖 **进程发现、内存分析、线程分析、参数查看、GC 监控** 五个方向。下面按“用途 + 常用参数 + 典型场景 + 注意事项”逐个说明。


## JVM常用命令

| 命令 | 全称 | 核心用途 | 是否影响进程 |
|---|---|---|---|
| `jps` | JVM Process Status | 列出 Java 进程，拿到 PID | 几乎无影响 |
| `jmap` | Memory Map for Java | 堆信息、对象统计、堆转储 | `-histo:live` 和 `-dump` 可能 STW |
| `jstack` | Stack Trace for Java | 打印线程栈，分析死锁、阻塞、CPU 高 | 进入安全点，短暂停顿 |
| `jinfo` | Configuration Info for Java | 查看/动态调整 JVM 参数和系统属性 | 动态改 flag 可能影响运行 |
| `jstat` | JVM Statistics Monitoring | 实时监控 GC、类加载、编译统计 | 基本无影响，轻量 |

现代 JDK（9+）推荐用 `jcmd` 统一替代很多功能，但面试中这五个命令仍是高频考点。


**jps：Java 进程状态**

**用途**：列出当前用户有权限看到的 Java 进程 PID 和主类，是其他工具的第一步。

**常用参数**：
```bash
jps -l        # 输出主类全名或 jar 路径
jps -v        # 输出 JVM 启动参数
jps -m        # 输出 main 方法参数
jps -q        # 只输出 PID
```

**典型场景**：
- 快速找到目标 Java 进程 PID，供 `jmap`、`jstack`、`jstat` 使用。
- 确认应用是否启动、启动参数是什么。

**注意**：
- 只能看到当前用户有权限的进程。
- 容器内可能看不到宿主机或其他容器的 Java 进程。
- `jps` 本身也是 Java 进程，有时会列出自己。

**jmap：内存与堆分析**

**用途**：查看堆内存配置和使用情况、统计对象数量、生成堆转储文件，是排查内存泄漏和 OOM 的核心工具。

**常用参数**：
```bash
jmap -heap <pid>                     # 堆配置和使用概况
jmap -histo <pid>                    # 对象实例数和占用字节排行
jmap -histo:live <pid>               # 只统计存活对象，会触发 Full GC
jmap -clstats <pid>                  # 类加载器统计
jmap -dump:format=b,file=heap.hprof <pid>   # 生成堆转储
jmap -dump:live,format=b,file=heap.hprof <pid>  # 只 dump 存活对象
jmap -finalizerinfo <pid>            # 等待 finalize 的对象
```

**典型场景**：
- 内存持续上涨：`jmap -histo` 看哪个类实例异常多。
- OOM 后：`jmap -dump` 生成 hprof，用 MAT / VisualVM 分析支配树和泄漏嫌疑。
- 确认堆各区大小、GC 策略是否合理。

**注意**：
- `-histo:live` 会触发 Full GC，生产环境慎用。
- `-dump` 可能引起 STW，大堆会停顿较久，尽量在低峰期或从副本操作。
- 部分 JDK 版本 `jmap -heap` 输出受限，推荐 `jcmd <pid> GC.heap_info` 替代。

**常见场景**：
- for循环创建对象或字符串
- 一次性查询大量数据
- 读取大文件
- 资源未关闭

**jstack：线程栈分析**

**用途**：打印 JVM 内所有线程的调用栈、状态和锁信息，用于分析死锁、线程阻塞、CPU 飙高。

**常用参数**：
```bash
jstack <pid>              # 打印线程栈
jstack -l <pid>           # 附加锁信息，自动检测死锁
jstack -F <pid>           # 强制打印，进程无响应时使用
jstack -m <pid>           # 混合模式，包含 Java 和本地栈
```

**典型场景**：
- **死锁**：`jstack -l <pid>` 输出末尾会直接报告 `Found one Java-level deadlock`。
- **CPU 100%**：先用 `top -Hp <pid>` 找到高 CPU 线程，`printf "%x\n" <tid>` 转十六进制，再到 `jstack` 输出中搜 `nid=0x...` 定位代码。
- **接口卡顿**：查看大量线程是否卡在 `WAITING`、`BLOCKED` 或同一个锁上。

**注意**：
- `jstack` 会让 JVM 进入安全点，短暂 STW，但通常可接受。
- `-F` 强制模式可能输出不准确，只在进程无响应时用。
- 需要与 `top`、`pidstat` 等结合，单看栈不一定能定位根因。

**jinfo：JVM 参数与系统属性**

**用途**：查看 JVM 启动参数、系统属性，并动态调整部分可管理参数。

**常用参数**：
```bash
jinfo -flags <pid>              # 查看 JVM 参数（非默认）
jinfo -sysprops <pid>           # 查看系统属性
jinfo -flag <name> <pid>        # 查看某个参数值
jinfo -flag [+|-]<name> <pid>   # 动态开启/关闭 manageable 参数
jinfo -flag <name>=<value> <pid> # 动态修改 manageable 参数
```

**典型场景**：
- 确认线上 JVM 实际生效的参数，比如堆大小、GC 收集器。
- 动态打开 `HeapDumpOnOutOfMemoryError`、`PrintGC` 等诊断开关。
- 查看 `java.version`、`user.dir` 等系统属性。

**注意**：
- 只有标记为 manageable 的 flag 才能动态修改，不是所有参数都支持。
- 动态修改可能引入不稳定，生产环境需谨慎。
- JDK 9+ 推荐 `jcmd <pid> VM.flags` 和 `VM.system_properties`。

**jstat：JVM 运行时统计监控**

**用途**：实时监控 GC、类加载、即时编译等统计数据，是 GC 调优最常用的轻量工具。

**常用参数**：
```bash
jstat -gc <pid> <interval> <count>       # 各分区容量和 GC 次数/耗时
jstat -gcutil <pid> 1000 10              # 各区使用百分比
jstat -gccapacity <pid>                  # 各区容量
jstat -gcnew <pid>                       # 新生代详情
jstat -gcold <pid>                       # 老年代详情
jstat -class <pid>                       # 类加载统计
jstat -compiler <pid>                    # JIT 编译统计
jstat -printcompilation <pid>            # 最近编译的方法
```

**`-gcutil` 输出字段**：
```
S0  S1  E   O   M   CCS  YGC  YGCT  FGC  FGCT  GCT
```
分别表示 Survivor0/1、Eden、Old、Metaspace、压缩类空间使用率，以及 Young GC 次数/耗时、Full GC 次数/耗时、总 GC 耗时。

**典型场景**：
- GC 是否频繁：观察 `YGC`、`FGC` 增长速度和 `GCT` 总耗时。
- 老年代是否持续增长：观察 `O` 是否接近 100% 且 Full GC 后不下降。
- 判断是否内存泄漏：多次 Full GC 后老年代占用仍居高不下。

**注意**：
- `jstat` 非常轻量，可长期采样。
- 它只给统计趋势，不定位具体对象，需结合 `jmap` 和堆转储。


## 组合排查套路

**1. CPU 飙高**
```bash
top -Hp <pid>                 # 找高 CPU 线程
printf "%x\n" <tid>           # 转十六进制
jstack <pid> | grep -A 20 "nid=0x<hex>"
```

**2. 内存泄漏 / OOM**
```bash
jps -l                        # 找 PID
jstat -gcutil <pid> 1000      # 看 GC 趋势
jmap -histo:live <pid>        # 看存活对象排行
jmap -dump:live,format=b,file=heap.hprof <pid>
# 用 MAT / VisualVM 分析 hprof
```

**3. 死锁**
```bash
jstack -l <pid> | grep -A 30 "deadlock"
```

**4. GC 调优**
```bash
jstat -gcutil <pid> 1000 20
jinfo -flags <pid>
jmap -heap <pid>
```

**5. 动态诊断开关**
```bash
jinfo -flag +HeapDumpOnOutOfMemoryError <pid>
jinfo -flag HeapDumpPath=/tmp <pid>
```

注意事项与现代替代

- **权限**：这些工具通常需要与目标 Java 进程相同用户或 root 权限。
- **容器环境**：进入容器执行，或使用 `jcmd`、Arthas 等。
- **生产慎用**：`jmap -dump`、`jmap -histo:live`、`jstack -F` 可能引起停顿或影响稳定性。
- **JDK 9+ 推荐 `jcmd`**：
  ```bash
  jcmd <pid> VM.flags
  jcmd <pid> VM.system_properties
  jcmd <pid> Thread.print
  jcmd <pid> GC.heap_info
  jcmd <pid> GC.class_histogram
  jcmd <pid> GC.heap_dump /tmp/heap.hprof
  ```
- **第三方工具**：Arthas、VisualVM、MAT、async-profiler 在排查效率上更强，但面试仍常问这五个原生命令的原理和用法。

如果面试官继续追问，常见方向是：`jmap -dump` 的 STW 原理、`jstack` 如何检测死锁、`jstat` 各字段含义、以及 `jcmd` 如何替代它们。


## 死锁如何解决

通过 jstack 检测死锁，已经发生的情况下，只能杀死线程。

可以通过一次性申请全部资源，或者设置超时时间。

## JVM 会发生OOM的地方

JDK 8 的内存结构与 JDK 7 及以前最大的区别是：**永久代（PermGen）被元空间（Metaspace）取代**。所以 JDK 8 不会再出现 `PermGen space`，但多了 `Metaspace` 和 `Compressed class space` 两类 OOM。下面按内存区域逐一说明。

| 错误信息 | 发生区域 | 是否受 `-Xmx` 限制 | 关键参数 |
|---|---|---|---|
| `Java heap space` | 堆 | 是 | `-Xmx -Xms` |
| `GC overhead limit exceeded` | 堆 | 是 | `-XX:GCTimeLimit` `-XX:GCHeapFreeLimit` |
| `Requested array size exceeds VM limit` | 堆 | 是 | 无 |
| `Metaspace` | 元空间（本地内存） | 否 | `-XX:MaxMetaspaceSize` |
| `Compressed class space` | 压缩类空间 | 否 | `-XX:CompressedClassSpaceSize` |
| `Direct buffer memory` | 堆外直接内存 | 否 | `-XX:MaxDirectMemorySize` |
| `unable to create new native thread` | 操作系统线程资源 | 否 | `-Xss`、`ulimit -u` |
| `StackOverflowError`（非 OOM） | 线程栈 | 否 | `-Xss` |
| OOM Killer（系统日志） | 操作系统物理内存 | 否 | cgroup、`-XX:MaxRAMFraction` |
| CodeCache 警告（非 OOM） | JIT 代码缓存 | 否 | `-XX:ReservedCodeCacheSize` |

### 1. `Java heap space`

**触发**：堆中对象占用达到 `-Xmx`，Full GC 后仍无法分配新对象。

**典型场景**：
- 内存泄漏：静态 Map 只增不减、ThreadLocal 未清理、监听器未注销。
- 大对象：全表查询、大文件读入内存、大数组。
- 高并发下对象创建速度远超回收速度。
- JDK 8 中字符串常量池在堆里，大量 `String.intern()` 也会导致堆 OOM。

**排查**：
```bash
jstat -gcutil <pid> 1000
jmap -histo:live <pid>
jmap -dump:live,format=b,file=heap.hprof <pid>
# MAT 分析支配树和 Leak Suspects
```

**解决**：修复泄漏、分页/流式处理、引入缓存淘汰、适当调大 `-Xmx`。

### 2. `GC overhead limit exceeded`

**触发**：GC 花费超过 98% 时间，但回收不到 2% 堆空间，连续多次后 JVM 主动抛出。本质是堆快满了，但还没到 `Java heap space`。

**解决**：与堆 OOM 相同。可用 `-XX:-UseGCOverheadLimit` 关闭，但不推荐，关闭后只会变成更晚的 `Java heap space`。

### 3. `Requested array size exceeds VM limit`

**触发**：请求分配的数组长度超过 JVM 上限（约 `Integer.MAX_VALUE - 2`），或长度计算溢出。

**解决**：检查数组长度计算逻辑，分片处理。

### 1. `Metaspace`

**背景**：JDK 8 起，类元数据存放在元空间，使用本地内存，默认无上限（受物理内存限制）。

**触发**：加载的类过多，或类加载器泄漏导致旧类无法卸载。

**典型场景**：
- 动态生成类：CGLIB、ASM、JDK 动态代理、Groovy、JSP 编译。
- 热部署：每次 reload 生成新 ClassLoader，旧类未卸载。
- 自定义 ClassLoader 被静态引用持有，其加载的所有类都无法回收。
- 反射滥用：大量 `Proxy.newProxyInstance`。

**排查**：
```bash
jstat -gcutil <pid> 1000        # 看 M 列持续上涨
jcmd <pid> VM.metaspace         # JDK 8 支持
jmap -clstats <pid>             # 类加载器统计
```

**解决**：
- 设置上限让问题尽早暴露：`-XX:MaxMetaspaceSize=512m`。
- 修复类加载器泄漏。
- 缓存动态代理类，避免重复生成。

### 2. `Compressed class space`

**背景**：开启压缩指针（默认开启）时，类元数据中有一部分放在“压缩类空间”，默认 1GB。

**触发**：类元数据超过 `-XX:CompressedClassSpaceSize`。

**解决**：调大 `-XX:CompressedClassSpaceSize`，或排查类加载泄漏。本质上和 Metaspace 同源。

**JDK 8 注意**：`-XX:MaxPermSize` 已失效，设置会收到警告并被忽略。

直接内存 OOM：`Direct buffer memory`

**背景**：NIO 的 `ByteBuffer.allocateDirect()`、Netty 的 `DirectByteBuf`、`FileChannel.map` 分配的是堆外内存，不受 `-Xmx` 限制。JDK 8 中默认上限约等于 `-Xmx`，可用 `-XX:MaxDirectMemorySize` 指定。

**触发**：直接内存分配超过上限。

**典型场景**：
- Netty `ByteBuf` 忘记 `release()`。
- NIO 通道未关闭。
- 频繁创建 DirectByteBuffer，依赖 `Cleaner` 回收不及时。

**排查**：
```bash
jmap -histo:live <pid> | grep DirectByteBuffer
-XX:NativeMemoryTracking=detail
jcmd <pid> VM.native_memory summary
```

**解决**：
- 显式释放：Netty 用 `ReferenceCountUtil.release()`。
- 限制上限：`-XX:MaxDirectMemorySize=256m`。
- 使用池化分配器：Netty `PooledByteBufAllocator`。

线程相关 OOM

### 1. `unable to create new native thread`

**触发**：JVM 向操作系统申请创建线程，但系统资源不足。

**典型场景**：
- 线程池配置不当，无限创建线程。
- 每个请求创建一个线程。
- 系统 `ulimit -u` 限制进程线程数。
- `-Xss` 过大，每个线程栈占用多，内存不够。

**排查**：
```bash
jstack <pid> | grep "java.lang.Thread.State" | wc -l
ulimit -u
cat /proc/sys/kernel/threads-max
```

**解决**：用线程池替代 `new Thread()`，合理设置 `-Xss`，调整 `ulimit`。

### 2. `StackOverflowError`（严格说不是 OOM）

**触发**：单个线程栈深度超限，通常是递归无终止或递归过深。

**解决**：修复递归、调大 `-Xss`、改递归为迭代。

代码缓存：通常只警告，不 OOM

**背景**：JIT 编译后的机器码存放在 CodeCache，JDK 8 默认约 240MB（分层编译）。

**表现**：CodeCache 满时，JVM 停止 JIT 编译，性能下降，日志打印 `CodeCache is full. Compiler has been disabled.`，一般不抛 OOM。

**排查**：
```bash
jcmd <pid> Compiler.codecache
```

**解决**：调大 `-XX:ReservedCodeCacheSize`。

操作系统层：OOM Killer

**触发**：JVM 进程占用物理内存过多，被 Linux OOM Killer 杀死。这不是 JVM 抛出的，而是操作系统行为。

**表现**：进程突然消失，`dmesg` 或 `/var/log/messages` 中有 `Out of memory: Kill process`。

**JDK 8 特别注意**：
- JDK 8 早期版本不感知容器 cgroup 限制，`-Xmx` 可能超过容器内存，导致被 kill。
- JDK 8u191+ 开始支持容器感知，可用 `-XX:+UseContainerSupport`、`-XX:MaxRAMPercentage`。
- 容器中堆只占容器内存的一部分，要给 Metaspace、直接内存、线程栈、CodeCache 留余量。

**解决**：
- 显式设置 `-Xmx`，一般堆占容器内存 50%~75%。
- 使用 `-XX:MaxRAMPercentage=70`（8u191+）。
- 排查堆外内存泄漏。
- 检查 cgroup 限制和 `dmesg`。

JDK 8 特有注意点

1. **永久代已移除**：不会再出现 `PermGen space`，`-XX:MaxPermSize` 无效。
2. **字符串常量池在堆中**：大量 `String.intern()` 会导致堆 OOM，而不是 PermGen。
3. **Metaspace 默认无上限**：受本地内存限制，容易耗尽系统内存，建议设置 `-XX:MaxMetaspaceSize`。
4. **压缩类空间默认 1GB**：类元数据过多时可能单独报 `Compressed class space`。
5. **容器感知有限**：8u191 之前不感知 cgroup，容易 OOM Killer；8u191+ 支持 `UseContainerSupport`。
6. **直接内存默认约等于 `-Xmx`**：未设置 `-XX:MaxDirectMemorySize` 时，直接内存上限与最大堆相同。

JDK 8 推荐诊断参数

```
-Xms4g -Xmx4g
-XX:MaxMetaspaceSize=512m
-XX:CompressedClassSpaceSize=256m
-XX:MaxDirectMemorySize=256m
-Xss512k
-XX:ReservedCodeCacheSize=240m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:+ExitOnOutOfMemoryError
```

**排查命令**：
```bash
jstat -gcutil <pid> 1000
jmap -histo:live <pid>
jmap -dump:live,format=b,file=heap.hprof <pid>
jstack <pid>
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.metaspace
jinfo -flags <pid>
```

如果面试官问“JDK 8 哪些地方会发生 OOM”，可以这样组织：

1. **先点明 JDK 8 变化**：永久代移除，元空间登场，所以没有 `PermGen space`。
2. **按区域分类**：
   - 堆内：`Java heap space`、`GC overhead limit exceeded`、数组过大。
   - 元空间：`Metaspace`、`Compressed class space`。
   - 堆外：`Direct buffer memory`。
   - 线程：`unable to create new native thread`、`StackOverflowError`。
   - 系统层：OOM Killer。
   - CodeCache：通常只警告。
3. **每种给出**：触发条件、典型场景、排查命令、解决手段。
4. **收尾**：强调“OOM 不一定是堆问题”，JDK 8 中 Metaspace、直接内存、容器限制都是高频坑。

```bash
-Xms4g -Xmx4g
-XX:MaxMetaspaceSize=512m
-XX:CompressedClassSpaceSize=256m
-XX:MaxDirectMemorySize=256m
-Xss512k
-XX:ReservedCodeCacheSize=240m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:+ExitOnOutOfMemoryError
```