# Redis 面试题

### 常见的数据类型有哪些

常见的有五种数据类型：String（字符串），Hash（哈希），List（列表），Set（集合）、Zset（有序集合）。

| 类型 | 底层实现 | 典型场景 |
|---|---|---|
| String | SDS 简单动态字符串 | 缓存、计数器、分布式锁 |
| Hash | 压缩列表 / 哈希表 | 对象、购物车存储 |
| List | 压缩列表 / 快速链表（quicklist） | 消息队列、时间线 |
| Set | 哈希表 / intset | 去重、抽奖、共同好友 |
| Zset | 跳跃表 + 哈希表 | 排行榜、延迟队列 |

### 为什么 Redis 快（高价值）

1. **纯内存操作**：数据都在内存中，读写是纳秒级；
2. **单线程模型**：避免上下文切换和锁竞争（6.0 后引入多线程处理网络 IO，但核心命令执行仍是单线程）；
3. **IO 多路复用**：用 epoll 监听多个 socket，一个线程处理所有连接事件；
4. **高效数据结构**：SDS、跳跃表、压缩列表等针对场景优化。

### 缓存穿透 / 缓存击穿 / 缓存雪崩（必问）

- **缓存穿透**：查询一个**不存在的数据**，缓存和数据库都没有，请求直接打到数据库。解决：① 缓存空值（过期时间设短）；② **布隆过滤器**（BloomFilter）拦截不存在的 key。
- **缓存击穿**：某个**热点 key 过期的瞬间**，大量并发请求直接打到数据库。解决：① 热点 key 设置永不过期；② **互斥锁**（只让一个线程查库重建缓存）。
- **缓存雪崩**：**大量 key 同时过期**或 Redis 宕机，海量请求打到数据库。解决：① 过期时间加随机值打散；② 多级缓存（本地缓存 + Redis）；③ 集群高可用 + 熔断限流。

```java
// 缓存击穿：互斥锁重建缓存（伪代码）
public String get(String key) {
    String value = redis.get(key);
    if (value != null) return value;
    String lockKey = "lock:" + key;
    if (redis.setnx(lockKey, "1", 30)) {  // 拿到锁的线程才查库
        try {
            value = db.query(key);
            redis.set(key, value, 300);
        } finally {
            redis.del(lockKey);
        }
    } else {                              // 没拿到锁，稍等重试
        Thread.sleep(50);
        return get(key);
    }
    return value;
}
```

**布隆过滤器是什么**：一种空间效率极高的**概率型数据结构**，用来快速判断"一个元素**一定不在**集合里 / **可能在**集合里"。

- 底层是一个**很长的二进制位数组**（初始全 0）+ 多个哈希函数；
- **添加元素**：用 k 个哈希函数算出 k 个位置，全部置为 1；
- **查询元素**：算同样的 k 个位置，**只要有 1 个位置是 0 → 一定不存在**（可安全拦截）；全是 1 → **可能存在**（可能误判）。

```java
// 初始化（预估数据量 100w，误判率 0.01），数据预热时把所有存在的 id 加入
BloomFilter<String> bloom = BloomFilter.create(Funnels.stringFunnel(Charset.forName("UTF-8")), 1000000, 0.01);
bloom.put("1001"); bloom.put("1002"); // ...

public Object getData(String key) {
    if (!bloom.mightContain(key)) return null;   // 一定不存在 → 直接拦截，不打 DB
    Object value = redis.get(key);               // 可能存在 → 正常走缓存 → DB
    if (value == null) {
        value = db.query(key);
        redis.set(key, value, 300);
    }
    return value;
}
```

**关键特性**：① 优点：省内存（100w 条约 1MB）、查询 O(k) 极快；② 有误判（假阳性）：不存在的元素可能被放行（查 DB 返回空，无害）；③ **不漏判**：已加入的元素绝不会被误判为不存在；④ **无法删除**（置 0 会误伤其他元素），要支持删除可用**布谷鸟过滤器**。其他场景：URL 去重、垃圾邮件过滤、HBase/Cassandra 判断数据是否在磁盘。

### 缓存与数据库一致性如何保证

- **Cache Aside（先更新数据库，再删缓存，推荐）**：更新 DB 成功后再删缓存，下次读取时重建。为什么"删"而不是"更新"：避免并发写导致缓存被旧值覆盖，且删缓存更简单。
- **延迟双删**：删缓存 → 更新 DB → 延迟几百 ms 再删一次，兜底读写并发导致的脏数据。
- **终极方案**：订阅 binlog（Canal）异步删除缓存，彻底解耦，保证最终一致性。

### RDB 和 AOF 的区别

- **RDB（快照）**：把内存数据以二进制快照写入磁盘。优点：文件小、恢复快；缺点：可能丢最后一次快照之后的数据；fork 子进程有阻塞风险。
- **AOF（追加日志）**：记录每条写命令。优点：数据更安全（可按策略 fsync）；缺点：文件大、恢复慢。AOF 重写（rewrite）可压缩文件。
- 企业实践：AOF + RDB 混用（AOF 开启 everysec），Redis 4.0 支持混合持久化。

### 过期删除策略与内存淘汰策略

- **删除策略**：惰性删除（访问时检查是否过期）+ 定期删除（定时随机抽查一批 key），两者配合。
- **内存淘汰策略**（maxmemory-policy）：LRU（近似 LRU，淘汰最久未用）、LFU（淘汰最不常用）、volatile-ttl（淘汰快过期的）、allkeys-random 等。

### 主从复制 / 哨兵 / 集群

- **主从复制**：从节点执行 `slaveof` 后，主节点 `bgsave` 生成 RDB 全量同步，之后增量同步写命令；实现读写分离（主写从读），但主挂需手动切换。
- **哨兵（Sentinel）**：监控主从节点状态，主节点宕机后自动**故障转移**；哨兵集群至少 3 个节点（过半投票防脑裂）。
- **集群（Cluster）**：通过 **16384 个哈希槽**分布数据，每个节点负责一部分槽，支持**水平扩容**；客户端按 `CRC16(key) % 16384` 定位槽。

### 如何用 Redis 实现分布式锁（高价值）

```java
// 加锁：setnx + 过期时间（必须原子，防止死锁）
Boolean ok = redis.setIfAbsent("lock:order:1", uuid, 30, TimeUnit.SECONDS);
// 解锁：Lua 脚本保证"判断是自己的锁 + 删除"两步原子，防止误删别人的锁
String lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
redis.eval(lua, Arrays.asList("lock:order:1"), Arrays.asList(uuid));
```

注意点：① value 用唯一标识（uuid），防误删；② 必须设过期时间防死锁；③ 业务超时要考虑**看门狗自动续期**（Redisson 实现，默认 30s）；④ 极端场景主从切换可能丢锁，可用 Redlock（复杂且有争议，一般不推荐）。

### 大 key / 热 key 问题

- **大 key**：单个 key 的 value 过大（大字符串、上百万元素的集合），导致阻塞、网络拥塞。解决：拆分、压缩、过期删除。
- **热 key**：访问量极高的 key，可能压垮单节点。解决：本地缓存、key 加随机后缀分散、读写分离。

### 如何用 Redis 实现限流（高价值）

限流目的：控制单位时间内的请求量，防止接口被刷爆、保护下游。常见算法四种：**固定窗口、滑动窗口、漏桶、令牌桶**。

**① 固定窗口计数（INCR + EXPIRE，最简单）**

```java
// 伪代码：1 秒内最多 100 次
String key = "rate:api:" + System.currentTimeMillis() / 1000; // 当前秒作为窗口 key
long count = redis.incr(key);        // 计数 +1（原子）
if (count == 1) redis.expire(key, 1); // 首次设置 1 秒过期
if (count > 100) { 拒绝请求; return; }
放行请求;
```

缺点：**临界问题**——窗口边界处流量可瞬间翻倍（如 0.9s 内请求 100 次 + 1.0s 后又请求 100 次）。

**② 滑动窗口（Zset，精确）**

```java
// 伪代码：统计最近 1 秒内的请求数
long now = System.currentTimeMillis();
String key = "sliding:api";
// 1. 移除窗口外的旧记录（Zset 按时间戳排序，score = 时间戳）
redis.zremrangeByScore(key, 0, now - 1000);
// 2. 统计窗口内请求数
long count = redis.zcard(key);
if (count >= 100) { 拒绝请求; return; }
// 3. 当前请求加入窗口（member 用唯一值如 UUID 防重复）
redis.zadd(key, now, uuid);
redis.expire(key, 2); // 兜底过期，防 key 无限增长
放行请求;
```

优点：无临界问题；缺点：每个请求都要存一条记录，**内存开销大**，适合精确控制小流量。

**③ 漏桶算法（Redis List 实现，控制速率恒定）**

**原理**：把请求想象成"水"，桶底有一个固定大小的"漏口"。无论水（请求）多猛地倒进来，**从漏口流出的速率是恒定的**；桶满了（超出容量）的水直接溢出丢弃。

```text
         请求流入（任意速率）
              ↓ ↓ ↓ ↓ ↓ ↓ ↓
        ┌─────────────────────┐
        │   桶（容量固定，存未处理的请求）│  ← 桶满则溢出丢弃
        └──────────┬──────────┘
                   ↓ 漏口（固定速率流出，如每秒 10 个）
             处理请求（恒定速率）
```

- **核心思想**：输出速率恒定 = 平滑突发流量（削峰填谷），**不管请求多猛，处理速度永远 ≤ 漏口速率**；
- **用 Redis 实现**：List 当桶（LPUSH 入桶、RPOP 出桶），后台任务固定速率出桶即可（或用 Zset 时间戳模拟匀速漏出）；
- **特点**：平滑流量、恒定速率，适合保护下游；**缺点**：突发流量直接被丢弃（即使下游有能力处理，也只能按固定速率慢慢漏），响应不及时。

```java
// 伪代码：桶容量 100，请求先进桶，按固定速率漏出
String key = "bucket:api";
long size = redis.llen(key);
if (size >= 100) { 拒绝请求（桶满了直接丢弃）; return; }
redis.lpush(key, "1");       // 入桶
redis.expire(key, 2);
// 后台任务：固定速率 rpop 出桶，即"匀速处理"
```

**④ 令牌桶（Lua 脚本，最常用，支持突发）**

**原理**：桶里放的是"令牌"，而不是请求。系统**以固定速率往桶里放令牌**（如每秒 10 个），桶有容量上限（如 100 个，放满了不再增加）。**每个请求必须取到 1 个令牌才能通过**；没令牌就拒绝或等待。

```text
        令牌生成器（固定速率，如每秒 10 个）
                   ↓ ↓ ↓
        ┌─────────────────────┐
        │  令牌桶（容量 100，最多存 100 个令牌）│ ← 桶满不再生成
        └──────────┬──────────┘
                   ↓ 取令牌
        请求 ←→ 有令牌放行，无令牌拒绝/等待
```

- **核心思想**：**限制平均速率 + 允许突发**——桶里攒下的令牌相当于"信用额度"，平时流量低时令牌积攒到满桶，突发时（如双 11 秒杀）可以一口气消耗积攒的令牌，扛住短时高峰，但**长期平均速率被生成速率锁死**；
- **和漏桶的本质区别**：
  - 漏桶：输出速率**绝对恒定**，突发必被丢弃，是"被动平滑"；
  - 令牌桶：允许**先积攒后突发**，能应对瞬时峰值，是"主动蓄水"，更贴近真实业务（平时空闲、峰值高）；
- **用 Redis 实现**：不真的定时放令牌，而是**懒计算**——用"上次补充时间"算出这期间该补多少令牌（`tokens = min(容量, tokens + (now - last) * 速率)`），一个 Lua 脚本原子完成"补充 + 取令牌"。

```java
// Lua 脚本：原子地"按速率补充令牌 + 尝试取令牌"
// key: 桶名   ARGV[1]: 桶容量   ARGV[2]: 每秒补充速率   ARGV[3]: 当前时间(秒)
String lua = ""
    + "local tokens = tonumber(redis.call('get', KEYS[1]) or ARGV[1]) "   // 当前令牌数（初始满桶）
    + "local last = tonumber(redis.call('get', KEYS[1]..':ts') or ARGV[3]) "
    + "local now = tonumber(ARGV[3]) "
    + "local rate = tonumber(ARGV[2]) "
    + "local cap = tonumber(ARGV[1]) "
    + "tokens = math.min(cap, tokens + (now - last) * rate) "              // 按流逝时间补充令牌
    + "if tokens >= 1 then "
    + "  redis.call('set', KEYS[1], tokens - 1, 'EX', 60) "
    + "  redis.call('set', KEYS[1]..':ts', now, 'EX', 60) "
    + "  return 1 "
    + "else "
    + "  redis.call('set', KEYS[1], tokens, 'EX', 60) "
    + "  redis.call('set', KEYS[1]..':ts', now, 'EX', 60) "
    + "  return 0 "
    + "end";
// 返回 1 放行，返回 0 拒绝
Object result = redis.eval(lua, Arrays.asList("bucket:api"), Arrays.asList("100", "10", String.valueOf(System.currentTimeMillis() / 1000)));
```

特点：**允许一定突发**（桶满时可持续消费）又限制平均速率，是生产最常用方案；每个请求一次 Redis 调用（Lua 保证原子性）。

**面试速答**：漏桶 = "固定速度放水"，适合削峰保护下游；令牌桶 = "固定速率发令牌 + 攒令牌应对突发"，是生产首选，Guava RateLimiter、Redisson、Sentinel 底层都是令牌桶。

**四种算法对比**：

| 算法 | 实现 | 特点 | 场景 |
|---|---|---|---|
| 固定窗口 | INCR + EXPIRE | 简单，有临界问题 | 粗略限流 |
| 滑动窗口 | Zset | 精确，内存开销大 | 精确控频（如登录失败次数） |
| 漏桶 | List | 速率恒定，平滑 | 保护下游（削峰） |
| 令牌桶 | Lua 脚本 | 允许突发，生产最常用 | 接口限流、网关限流 |

**工程实践**：生产一般不裸写 Redis 限流，而是用现成组件——网关层用 **Sentinel / Nginx lua-resty-limit / Kong**，应用层用 **Redisson 的 RateLimiter（令牌桶）** 或 **Guava RateLimiter（单机）**。注意限流要配合**降级/熔断**（返回友好提示或兜底数据），并做好压测确定阈值。
