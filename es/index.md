## 逻辑结构

**Index**

ES 会为所有字段建立索引，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

索引不是一个大文件，是**多个主分片 + 副本分片**的集合。

- 创建索引时指定**主分片数量**，**一旦创建不能直接修改**（只能重建索引）；
- **副本分片**：主分片的备份，用于容灾 + 分担读压力；副本数量可随时调整；
- 一条文档，根据`_id`哈希路由，只会落在**某一个主分片**上。

**Index Alias（索引别名）**

索引别名是指向一个或多个真实索引的"虚拟名称"。你可以把它理解为 Linux 中的软链接（symlink）.

**Mapping（映射）**

| 类型 | 说明 | 适用场景 |
| :--- | :--- | :--- |
| `text` | 全文搜索，会分词 | 文章标题、内容 |
| `keyword` | 精确匹配，不分词 | 状态码、标签、ID |
| `integer` / `long` | 数值类型 | 计数、价格 |
| `float` / `double` | 浮点数 | 评分、经纬度 |
| `date` | 日期类型 | 创建时间 |
| `boolean` | 布尔值 | 是否上架 |
| `nested` | 嵌套对象（独立索引） | 评论列表 |
| `geo_point` | 地理坐标 | 经纬度 |
| `completion` | 自动补全 | 搜索建议 |

**Document**

Index 里面单条的记录称为 Document（文档）。

**Field**

字段（field） 是包含数据的键值对。

默认情况下，Elasticsearch 对每个字段中的所有数据建立索引，并且每个索引字段都具有专用的优化数据结构。

## 物理结构

```text
集群 Cluster
 └── 节点 Node（ES进程/服务器）
      └── 索引 Index（逻辑）
           ├── 主分片 Primary Shard（Lucene索引）
           └── 副本分片 Replica Shard（Lucene索引）
                └── Segment段（不可变，一组Lucene文件）
                     └── 各类索引文件（tim/doc/fdt/dv等）
                └── translog事务日志
```

**Cluster**

一组 ES 节点的集合，集群名唯一。集群统一管理所有索引、分片分配、主节点选举，客户端请求可以发给任意节点，由**协调节点**转发请求到对应分片。

**Node**

节点角色（可多角色混合，生产建议分离）：

1. **Master 主节点**：负责集群元数据管理（创建 / 删除索引、分片分配、节点管理），**不处理业务查询写入**；
2. **Data 数据节点**：存放分片数据，执行索引、搜索、聚合，**核心存储节点**；
3. **Coordinating 协调节点**：接收客户端请求，路由、合并分片结果，不存数据；
4. **Ingest 预处理节点**：Pipeline 数据预处理。

> 目录结构：`data/elasticsearch/nodes/{节点编号}/`

**Shard（分片）**

当单台机器不足以存储大量数据时，Elasticsearch 可以将一个索引中的数据切分为多个 分片（shard） 。 分片（shard） 分布在多台服务器上存储。有了 shard 就可以横向扩展，存储更多数据，让搜索和分析等操作分布到多台服务器上去执行，提升吞吐量和性能。每个 shard 都是一个 lucene index。

> **每个分片 = 一个独立完整的 Lucene 索引**，有独立倒排索引、存储文件，可单独读写搜索Elastic

- **Primary Shard 主分片**：处理写请求；写成功后同步到副本；
- **Replica Shard 副本分片**：只读，主分片宕机时可提升为主分片；副本和主分片**不能放在同一个节点**（防止单机故障丢数据）。

分片目录：`nodes/0/indices/{索引名}/{分片id}/` ，分片内部包含 3 部分：

1. `index`：Lucene 索引文件（segments）
2. `translog`：事务日志，类似 MySQL binlog，防止刷盘前宕机丢数据
3. `_state`：分片元数据（主 / 副本标识、版本等）

**Segment**

Segment 是磁盘上的一组索引文件，**不可变**：

- 新增文档不会修改旧 Segment，只会新建 Segment；
- 删除文档：不是直接删文件，而是在`.liv`中标记为删除；
- 后台会执行 **Segment Merge**：把多个小段合并成大段，清理已删除文档，减少文件数量，提升查询性能。

> Lucene Segment 核心文件：

- `.tim` / `.tip`：词项字典（Term Dictionary & Index，倒排核心）
- `.doc`：倒排表，记录每个 term 对应的文档 id
- `.fdt` / `.fdx`：存储字段，存放`_source`原始文档内容
- `.dv`：DocValues，正排列式存储，用于排序、聚合
- `.nvd`：Norm，字段归一化权重，用于相关性打分
- `.liv`：记录被标记删除的文档

**Replica（副本）**

任何一个服务器随时可能故障或宕机，此时 shard 可能就会丢失，因此可以为每个 shard 创建多个副本（replica）。replica 可以在 shard 故障时提供备用服务，保证数据不丢失，多个 replica 还可以提升搜索操作的吞吐量和性能。primary shard（建立索引时一次设置，不能修改，默认 5 个），replica shard（随时修改数量，默认 1 个），默认每个索引 10 个 shard，5 个 primary shard，5 个 replica shard，最小的高可用配置，是 2 台服务器。

## 应用场景

全文搜索  
这是 ES 最经典的使用场景。电商平台的商品搜索、新闻网站的文章搜索、知识库检索等都离不开 ES。它支持中文分词、同义词、拼音搜索、拼写纠错等高级搜索特性。

日志分析  
结合 ELK Stack，ES 是目前最主流的日志分析方案。海量的应用日志、访问日志、系统日志都可以写入 ES，然后通过 Kibana 进行实时监控和分析。

指标聚合与可视化  
ES 强大的聚合（Aggregation）能力使其可以用于实时的数据分析场景，例如计算 PV/UV、统计销售趋势、分析用户行为等。

地理信息搜索  
ES 原生支持 geo_point 和 geo_shape 类型，可以实现"附近的人"、"范围内的门店"等地理位置搜索功能。

自动补全与推荐  
通过 completion 类型和 suggest API，ES 可以高效实现搜索框的自动补全（如"输入’苹’自动提示’苹果手机’"）。

## Bulk API

Bulk API 使用一种特殊的 NDJSON（Newline Delimited JSON） 格式，每一行都是独立的 JSON  对象，行与行之间用换行符 `\n` 分隔：

```text
POST /_bulk
{action_line}\n
{optional_source_line}\n
{action_line}\n
{optional_source_line}\n
...
```

Bulk API 支持四种操作，每种操作的格式略有不同：

- index 创建或覆盖文档
- create 仅创建（已存在则失败）
- update 部分更新
- delete 删除文档

四种操作可以在同一个 Bulk 请求中混合使用，通常 2-4 个并发线程即可打满 ES 集群的写入能力。可以逐步增加并发，观察 CPU 和写入延迟，找到最佳平衡点。

```text
POST /_bulk
{"index": {"_index": "my_blog", "_id": "10"}}
{"title": "文章10", "author": "王五", "views": 500}
{"create": {"_index": "my_blog", "_id": "11"}}
{"title": "文章11", "author": "赵六", "views": 300}
{"update": {"_index": "my_blog", "_id": "10"}}
{"doc": {"views": 600}}
{"delete": {"_index": "my_blog", "_id": "11"}}
```

```text
{
  "took": 30,
  "errors": true,    // 只要有一条失败就为 true
  "items": [
    {
      "index": {
        "_index": "products",
        "_id": "1",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    },
    {
      "create": {
        "_index": "products",
        "_id": "1",
        "status": 409,
        "error": {
          "type": "version_conflict_engine_exception",
          "reason": "[1]: version conflict, document already exists"
        }
      }
    }
  ]
}
```

## 倒排索引

倒排索引是 ES 实现高效全文搜索的核心数据结构。传统数据库的 B+ 树索引是"给定文档 ID，找到文档内容"；而倒排索引是反过来的，给定关键词，找到包含它的所有文档。

```
Term Dictionary（词典）      Posting List（倒排列表）
─────────────────────        ──────────────────────
elasticsearch     →     [doc1, doc3, doc7, doc12]
性能              →     [doc2, doc5, doc7]
调优              →     [doc2, doc7, doc9]
分布式            →     [doc1, doc3, doc4]
```

倒排索引由三层结构组成：

- Term Index（词项索引）：使用 FST（Finite State Transducer）实现的前缀树结构，常驻内存，用于快速定位 Term 在 Term Dictionary 中的位置。FST 的优势在于极高的压缩率——它既是一个有向无环图，又能共享前缀和后缀。

- Term Dictionary（词项字典）：存储所有的 Term，按字典序排列，存储在磁盘上。通过 Term Index 可以快速二分定位到对应的 block。

- Posting List（倒排列表）：记录每个 Term 对应的文档 ID 列表，以及词频（TF）、位置（Position）等信息。使用多种压缩算法（如 FOR - Frame Of Reference、Roaring Bitmaps）来减少存储空间。

## 写入流程

Elasticsearch 的写入流程可以拆解为两个层面：**集群层面的路由与副本同步**，以及**单分片内部的数据持久化**。

当你向 ES 发送写入请求时，任意节点都会充当协调节点。它首先根据文档的 ID（或指定的路由字段）计算哈希值，确定文档属于哪个分片。公式为：`shard = hash(_routing) % number_of_primary_shards`。

确定目标分片后，请求被转发到该分片的**主分片**所在节点。主分片会执行写入操作，成功后，再将请求**并行转发**给所有同步中的副本分片。只有当所有副本都报告成功后，主分片才会向协调节点确认，最终返回给客户端写入成功。

在单个分片内部，写入并非直接落盘，而是经历多个阶段以确保性能与可靠性：

1.  **写入内存缓冲区与 Translog**：文档首先被写入内存中的 **Index Buffer**（此时不可被搜索），同时被追加到磁盘上的 **Translog**（事务日志）中。Translog 的存在是为了防止节点崩溃导致内存数据丢失。

2.  **Refresh（刷新）**：默认每秒执行一次。它将内存缓冲区里的数据写入一个新的 **Segment** 文件，但此时只是写入操作系统的文件系统缓存，并未物理落盘。这一步完成后，文档才变得**可被搜索**，这也是 ES 被称为“准实时”的原因。

3.  **Flush（刷盘）**：默认每 30 分钟，或当 Translog 变得过大时触发。它执行一次 Lucene commit，将文件系统缓存中的所有 Segment **强制 fsync 到磁盘**，并清空当前的 Translog（因为数据已安全落盘）。

4.  **Merge（段合并）**：由于每秒都可能产生新的 Segment，ES 会在后台定期将小的 Segment 合并成更大的 Segment，以减少文件数量、提升查询效率。

## 分布式搜索

Elasticsearch 的搜索流程同样可以从 **集群层面的请求分发** 和 **单分片内部的查询执行** 两个层面来理解。与写入的“先主后副”不同，搜索请求可以打到任意分片（包括主分片和副本分片），以实现负载均衡。

ES 的搜索分为两个阶段，协调节点在其中扮演核心角色：

**1. Query Phase（查询阶段）**

协调节点收到搜索请求后，会向索引中**所有分片（主或副本）** 并行发送查询请求。每个分片在本地执行搜索，但**不返回完整文档**，只返回：

- 匹配文档的 ID
- 用于排序的字段值（如 `_score`）
- 分片本地的一个优先队列（Top N）

协调节点收集所有分片的返回结果，进行**全局排序和合并**，最终确定真正需要返回的 Top N 文档 ID 列表。这一步本质上是“找出哪些文档该返回”。

**2. Fetch Phase（取回阶段）**

协调节点根据 Query Phase 确定的文档 ID 列表，向**持有这些文档的分片**发送多轮 GET 请求（实际是 mget 形式）。各分片返回完整的 `_source` 文档内容，协调节点再组装成最终响应返回给客户端。

> 注意：Query Phase 只定位文档，Fetch Phase 才取回真实数据。这种设计避免了在网络中传输大量不必要的数据。

在单个分片内，搜索依赖 Lucene 的倒排索引结构：

1. **查询解析**：将查询语句解析为 Lucene 的 Query 对象树（如 TermQuery、BooleanQuery、PhraseQuery 等）。

2. **倒排索引查找**：对于每个词项，从 **Term Dictionary** 中定位到对应的 **Posting List**（包含该词项的文档 ID 列表及词频、位置等信息）。Term Dictionary 通常使用 FST（有限状态转换器）实现，内存占用小且查询快。

3. **查询执行与打分**：Lucene 对 Posting List 进行交、并、差等集合运算，得到候选文档集。同时根据 TF-IDF 或 BM25 等算法计算每个文档的相关性得分 `_score`。

4. **结果收集**：使用优先队列收集 Top N 结果，返回给协调节点。

ES 支持不同的搜索执行方式，影响性能：

- **Query Then Fetch**：默认方式，即上述两阶段流程，适合大多数场景。
- **DFS Query Then Fetch**：在 Query Phase 前增加一个预查询阶段，先收集全局词频等统计信息，使打分更准确，但性能开销更大。
- **Scroll / Search After**：用于深度分页或大批量导出。Scroll 会生成快照并保持上下文，Search After 则基于上一页的排序值继续查询，避免深度分页的性能问题。

## BM25 算法

ES 从 5.0 版本开始使用 BM25 作为默认相关性评分算法（取代了经典的 TF-IDF）。

```
score(D, Q) = Σ IDF(qi) × [ f(qi, D) × (k1 + 1) ] / [ f(qi, D) + k1 × (1 - b + b × |D| / avgdl) ]
```

核心思想（直觉理解）：

- IDF（逆文档频率）：一个词越罕见（出现在越少的文档中），它的区分度越高，权重越大。"的"这种常见词 IDF 很低，"Elasticsearch"这种特定词 IDF 较高。

- TF（词频）：一个词在文档中出现次数越多，文档越可能与该词相关。但 BM25 对 TF 有饱和处理——出现 10 次和出现 100 次的差距不会像 TF-IDF 那样是 10 倍，而是逐渐趋于平缓。

- 文档长度归一化：较短的文档中出现搜索词，比在长文档中出现同一词更有意义。参数 b（默认 0.75）控制长度归一化的程度。

- 参数 k1（默认 1.2）：控制 TF 的饱和速度。k1 越大，TF 影响越大。

## 深度分页

Elasticsearch 的分页看似简单，实则暗藏深坑。核心问题在于：ES 是分布式的，每个分片独立持有部分数据，协调节点无法像单机数据库那样直接跳过前 N 条记录。这导致了深度分页的性能瓶颈。

from + size 方式下，如果请求 from=10000, size=10，每个分片需要返回 top 10010 条结果给协调节点，如果有 5 个分片就是 50050 条，然后协调节点排序后只取 10 条，大量资源被浪费。

Elasticsearch 的分页看似简单，实则暗藏深坑。核心问题在于：**ES 是分布式的，每个分片独立持有部分数据，协调节点无法像单机数据库那样直接跳过前 N 条记录**。这导致了深度分页的性能瓶颈。

**1. from + size（浅分页）**

最常用的方式，`from` 指定起始位置，`size` 指定返回条数：

```json
GET /my_index/_search
{
  "from": 0,
  "size": 10,
  "query": { "match_all": {} }
}
```

**原理**：协调节点向所有分片发送请求，每个分片返回 `from + size` 条结果，协调节点汇总后排序，再截取 `[from, from+size)` 区间的数据。

**问题**：假设有 5 个分片，要取第 1000 页（`from=9990, size=10`），每个分片都要返回前 10000 条，协调节点需要处理 **5 × 10000 = 50000** 条记录才能选出真正的 10 条。这就是**深度分页**的性能灾难。

**限制**：ES 默认 `index.max_result_window = 10000`，即 `from + size` 不能超过 10000。超过会报错：

```
Result window is too large, from + size must be less than or equal to: [10000]
```

**2. scroll（游标分页）**

用于**大批量导出**场景，相当于给索引拍了个快照：

```json
POST /my_index/_search?scroll=5m
{
  "size": 1000,
  "query": { "match_all": {} }
}
```

首次请求返回一个 `_scroll_id`，后续用这个 ID 不断取下一批：

```json
POST /_search/scroll
{
  "scroll": "5m",
  "scroll_id": "DXF1ZXJ5..."
}
```

**特点**：

- 每次返回 `size` 条，但**不消耗 from**，所以不受 10000 限制
- 快照机制：scroll 期间的新增、修改、删除**不会反映**在结果中
- 上下文需要维护，占用资源，必须显式清除：

```json
DELETE /_search/scroll
{ "scroll_id": "DXF1ZXJ5..." }
```

**适用场景**：全量导出、离线批处理。**不适用**于实时用户分页。

**3. search_after（实时深度分页）**

ES 5.0 引入，推荐用于**实时深度分页**。它基于上一页的排序值继续查询，而不是跳过前 N 条：

```json
GET /my_index/_search
{
  "size": 10,
  "sort": [
    { "timestamp": "desc" },
    { "_id": "asc" }
  ]
}
```

返回结果中每条文档会带有 `sort` 值：

```json
"hits": [
  {
    "_id": "doc1",
    "_source": { ... },
    "sort": [1633024800000, "doc1"]
  }
]
```

下一页请求带上最后一条的 `sort` 值：

```json
GET /my_index/_search
{
  "size": 10,
  "sort": [
    { "timestamp": "desc" },
    { "_id": "asc" }
  ],
  "search_after": [1633024800000, "doc1"]
}
```

**特点**：

- 无状态，不需要维护 scroll 上下文
- 不受 10000 限制，可以一直翻到底
- 只能**逐页向后**，不能跳页
- 排序字段必须**唯一且稳定**，通常用 `_id` 或时间戳 + `_id` 组合作为 tiebreaker

**适用场景**：无限滚动、实时深度分页。

**实践建议:**

1. **普通搜索**：用 `from + size`，但限制在前 10000 条内。如果业务确实需要更大窗口，可以调大 `max_result_window`，但这只是治标，深度分页依然慢。

2. **无限滚动 / 实时深度翻页**：用 `search_after`，配合稳定的排序字段。

3. **全量导出**：用 scroll，或直接用 ES 的 `_search?scroll` + PIT（Point in Time）。

4. **避免跳页需求**：很多产品经理会要求“跳到第 100 页”，但从技术角度，深度分页既慢又无实际意义（用户不会真的一页页翻到 100 页）。可以通过限制页码、改用搜索条件缩小范围等方式规避。

5. **PIT + search_after**：ES 7.10 引入 PIT（Point in Time），可以给 search_after 加一个一致性视图，避免翻页过程中数据变化导致的重复或遗漏：

```json
POST /my_index/_pit?keep_alive=5m
```

拿到 PIT ID 后，在 search_after 请求中带上 `"pit": { "id": "..." }`，翻完后删除 PIT。


## 写入性能优化：追求吞吐量

写入的核心瓶颈通常在磁盘 I/O 和段合并（Merge）上。优化的目标是**减少刷新和合并的频率**，把资源集中用于批量处理。

**批量与并发策略**
*   **使用 Bulk API**：避免单条写入。建议批次大小控制在 **5-15MB**，或 **1000-5000** 条文档，并发数设为 CPU 核心数的 **2-3 倍**。
*   **考虑路由优化**：对于大规模批量写入，可以探索类似 `index.bulk_routing: local_pack` 的路由策略，将同一 Bulk 请求内的文档尽量路由到同一分片，减少内部网络转发开销。

**降低刷新与合并开销**
*   **调大 `refresh_interval`**：默认 1 秒的刷新会产生大量小段文件。对于实时性要求不高的场景（如日志、全量导入），可临时调大到 **30s** 甚至设为 **-1**（关闭），写入完成后再恢复。
*   **写入时关闭副本**：全量导入时，可将 `number_of_replicas` 临时设为 **0**，消除副本同步的磁盘和网络开销，完成后再恢复。
*   **调整 Translog 持久性**：默认 `request` 级别每次写入都刷盘。如果可接受极端情况下少量数据丢失，可设为 **`async`**，并调大 `sync_interval`（如 30s）和 `flush_threshold_size`（如 1GB），显著减少磁盘 I/O。

**资源与硬件**
*   **增大索引缓冲区**：适当调大 `indices.memory.index_buffer_size`（默认 10%，可尝试 20%），为写入提供更多内存缓冲。
*   **使用 SSD 并调整合并线程**：SSD 能大幅提升合并性能。对于机械盘，应将 `index.merge.scheduler.max_thread_count` 设为 **1**，避免 I/O 争抢；SSD 可设得更高。

## 搜索性能优化：降低延迟

搜索的瓶颈通常在于**文件系统缓存（Filesystem Cache）命中率**和**查询的复杂度**。

**利用缓存与减少 I/O**
*   **确保文件系统缓存充足**：ES 严重依赖 OS 的文件系统缓存来加速读取。JVM 堆不要超过物理内存的 50%，且**务必不超过 32GB**（否则指针压缩失效），把剩余内存留给缓存。
*   **使用 `best_compression`**：当数据量远超可用内存时，启用 `index.codec: best_compression` 可将索引体积缩小 **25%**，让更多数据能塞进缓存，从而将查询延迟**降低约 50%**。
*   **合理设置分片与副本**：每个分片建议 **20-50GB**。副本虽能提升读吞吐，但会增加节点上的分片总数，反而可能降低缓存效率。**更少的分片/节点通常意味着更好的搜索性能**。
*   **预热缓存**：对于重启后的关键索引，可使用 `index.store.preload` 将热点文件提前加载到文件系统缓存。

**优化查询与索引设计**
*   **优先使用 Filter**：对于不需要评分的精确匹配（如状态、时间范围），使用 `filter` 上下文。其结果会被缓存，能显著加速重复查询。
*   **避免深度分页**：放弃 `from + size` 的深分页，改用 **`search_after`**（实时）或 **`scroll`**（导出）。
*   **精简字段与索引**：
    *   对不需要搜索的字段设置 `"index": false`。
    *   使用 `copy_to` 将多个搜索字段合并为一个，避免 `multi_match` 扫描过多字段。
    *   聚合和排序使用 **`keyword`** 类型，避免在 `text` 上启用 fielddata。
*   **利用 `preference` 提升缓存命中**：在请求中带上 `preference=<用户ID>`，可以将同一用户的请求固定路由到同一分片副本，从而利用节点级的查询缓存。

