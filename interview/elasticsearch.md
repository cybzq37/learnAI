# Elasticsearch 面试题

### 什么是 ES，为什么搜索快

ES（Elasticsearch）是基于 **Lucene** 的分布式全文搜索引擎。快的关键是**倒排索引**：把 "文档 → 词" 反转成 "词 → 文档" 的映射，查询时直接定位包含关键词的文档，无需全表扫描。

### 倒排索引原理

```text
原始文档：
文档1：我喜欢编程
文档2：我喜欢学习
文档3：编程很有意思

倒排索引（分词后）：
编程   → 文档1, 文档3
我喜欢 → 文档1, 文档2
学习   → 文档2
有意思 → 文档3
```

由"词项字典（Term Dictionary）→ 词项索引（Term Index）→ 倒排列表（Posting List）"三级结构组成，Term Index 用 FST 常驻内存，加速词项定位。

### 写入与查询流程

- **写入**：写入内存 buffer → **refresh（默认 1s）** 生成段（segment）并打开供查询 → 同时写 **translog** 保证不丢 → 后台合并段。这就是 ES"近实时"（NRT）的原因：新增数据约 1 秒后可被搜到。
- **查询**：先查内存中的 Term Index 定位词项 → 取倒排列表 → 各分片并行查询 → 协调节点汇总排序 → 返回结果。

### 分片与副本

- 索引分为多个**分片（shard）**分布在不同节点，实现水平扩展与并行查询；
- 每个分片有**副本（replica）**，提供高可用与读负载均衡；
- 注意：**分片数一旦创建不可修改**，需提前规划；主分片数接近节点数时最合理。

### 存储结构（逻辑 + 物理）

#### 逻辑存储结构：Index → Shard → Document → Field → Term

```text
Index（索引，逻辑上的数据集合，类似 MySQL 的表）
 └── Shard（分片，数据分布与并行查询的最小单元）
      └── Document（文档，一条数据，类似 MySQL 的行）
           └── Field（字段，一列数据，类似 MySQL 的列）
                └── Term（词项，分词后的最小单元）
```

- **Index**：逻辑数据集合，可包含多个分片；
- **Shard**：数据分布与并行查询的最小单元，每个分片是**独立的 Lucene 索引实例**；
- **Document**：一条 JSON 记录（含 `_id`、`_source`），类似 MySQL 的行；
- **Field**：一个字段，类似 MySQL 的列，可配置分词器与分析方式；
- **Term**：字段经**分析器（Analyzer）**分词后的最小单元，是倒排索引的键。

#### 物理存储结构：分片 = 多个 Segment + translog

```text
Shard（分片 = 一个 Lucene 实例）
 ├── Segment（段，不可变，多个）
 │    ├── 倒排索引（.tip / .tim / .doc）  ← 搜索
 │    ├── Doc Values（.dvd / .dvm）        ← 排序、聚合
 │    └── Stored Fields（.fdt / .fdx）     ← 返回 _source
 ├── translog（预写日志，防止丢数据）
 └── 内存缓存（Term Index / Query Cache 等）
```

- **Segment**：分片由多个**不可变**的段组成——refresh 生成新段、merge 合并旧段，段一旦写入不再修改，只能合并；删除文档只是打 .liv 标记，**merge 时才真正物理删除**；
- **translog**：预写日志，节点宕机后靠它恢复未 refresh 的数据；
- 段内部文件结构如下：

```text
词项索引 Term Index（.tip，FST 常驻内存）
   ↓ 定位词项
词项字典 Term Dictionary（.tim）
   ↓ 定位倒排列表
倒排列表 Posting List（.doc / .pos / .pay）
```

- **Term Index（.tip）**：用 **FST** 构建，常驻内存，快速定位词项在字典中的位置，这是搜索快的关键之一；
- **Term Dictionary（.tim）**：词项按字典序排列，与 Term Index 配合实现近乎 O(词长) 的查找；
- **Posting List（.doc）**：记录包含该词项的文档 ID、词频（.pos 记位置），用**增量编码 + 跳表 + Roaring Bitmap** 压缩，节省磁盘和内存；
- **Doc Values（.dvd/.dvm）**：**列式（正排）存储**，文档 ID → 字段值，排序/聚合/脚本访问走它，从磁盘/OS 缓存读取，**不占 JVM 堆内存**；
- **Stored Fields（.fdt/.fdx）**：原始字段值，仅用于返回结果（如 `_source`），与搜索无关；
- **.si**：段元数据；**.liv**：删除标记。

### 内存与缓存结构

- **Segment Memory**：各段常驻内存的部分（Term Index、部分倒排索引）；
- **Field Data Cache**：字段数据缓存（旧版聚合用，内存占用大，已基本被 Doc Values 取代）；
- **Query Cache / Request Cache**：过滤（bitset）结果缓存与查询结果缓存，提升重复查询性能；
- **Index Buffer**：写入缓冲，默认堆的 10%，写满触发 refresh 落段。

### translog 与段合并（Merge）

- 写入：先写内存 buffer，同时写 **translog**（落盘），节点宕机后靠 translog 恢复未 refresh 的数据；
- refresh（默认 1s）生成小段 → 后台 **merge** 把小段合并成大段，同时清理 .liv 删除标记，减少段数、提升查询性能；
- **分层存储（ILM）**：按数据热度分 hot / warm / cold 节点，冷数据可只读并压缩，控制存储成本。

### 深分页问题（from+size / scroll / search_after）

- `from + size`：深分页时需各分片拉取并排序大量数据，**内存开销大**，默认上限 10000；
- **Scroll**：生成快照游标，适合**导出/全量遍历**（后台大数据量任务）；
- **search_after**：基于上一页最后一条的排序值翻页，适合**实时滚动分页**（App 下拉加载），性能稳定。

### ES 与 MySQL 数据同步方案

- 业务双写：写 MySQL 同时写 ES（简单但易不一致）；
- **订阅 binlog（Canal）**：监听 MySQL 变更日志异步同步到 ES（主流方案，解耦可靠）；
- 定时任务全量/增量同步（适合实时性要求不高的场景）；
- 一致性目标为**最终一致性**，配合重试/补偿机制。

### 脑裂问题

网络分区时多个节点互相争夺主节点，产生多个 Master。解决：最小主节点数设为 `(节点数/2)+1`（过半原则）；新版本用 discovery 模块配置。

### 相关性打分

默认 **BM25** 算法：综合词频（TF）、逆文档频率（IDF）、文档长度归一化计算得分。可用 `function_score` 自定义打分（如按时间、热度加权）。

### 地理位置检索（POI 检索 / 正逆地理编码）

#### 数据类型与底层存储

- **geo_point**：存经纬度坐标，内部用 **BKD 树**（Lucene 的平衡 k-d 树）索引，配合**倒排索引之外的多维点索引**，范围/距离查询无需全扫描；
- **geo_shape**：存点、线、多边形等地理形状，用于边界判断（判断点在哪个区域）；
- 写入时可自动生成 **geohash / geo_tile** 子字段，用于网格聚合。

#### POI 检索（附近的人 / 附近的门店）

- 建索引：POI 的经纬度用 `geo_point` 类型存储；
- **geo_distance 查询**：找指定点一定半径内的 POI（如"3 公里内的餐厅"）；
- **geo_bounding_box**：矩形范围粗筛，适合先圈定候选区域再精算；
- 按距离排序：`sort: [{ "_geo_distance": ... }]`，实现"由近到远"，配合 `search_after` 做滚动分页；
- 通常流程：**geo_bounding_box 粗筛 → geo_distance 精算距离 → _geo_distance 排序 → search_after 分页**；
- 热力图/统计：`geohash_grid` / `geo_tile` 聚合，按网格统计 POI 数量。

```json
// 附近 3km 内的 POI，按距离从近到远排序
{
  "query": {
    "bool": {
      "filter": [
        { "geo_distance": { "distance": "3km", "location": { "lat": 31.23, "lon": 121.47 } } }
      ]
    }
  },
  "sort": [ { "_geo_distance": { "location": { "lat": 31.23, "lon": 121.47 } } } ]
}
```

#### 正地理编码（地址 → 坐标）

ES **本身不做地址解析**，常见做法：

- 接入第三方地理编码服务（高德 / 百度 / Google Geocoding API），把"地址字符串"解析成经纬度，**拿到坐标后再写入 ES**；
- 自建地址词库：用 IK 等分词器对地址分词，配合词典匹配落库，适合自有封闭数据；
- 面试重点：正编码发生在**写入侧**，ES 只负责存储与检索坐标。

#### 逆地理编码（坐标 → 地址）

ES **不直接支持"坐标反查地址"**，常见方案：

- **行政区边界反查（主流）**：把省/市/区边界建成 `geo_shape`（polygon）存在 ES 中，查询时用 `geo_shape` 的**点在多边形内（intersects/within）**判断坐标落在哪个区域，返回区划名称；
- **geohash 前缀匹配**：坐标转 geohash，与预置的区划 geohash 前缀比对，粗粒度定位（精度受 geohash 层级限制）；
- 精确到街道/门牌号：仍需调用高德/百度逆地理 API；
- 面试重点：**逆编码本质是"点与多边形区域的包含关系"查询**，属于空间计算而非全文检索。

#### 一句话总结

- POI 检索 = `geo_point` + `geo_distance`/`geo_bounding_box` + `_geo_distance` 排序，底层 **BKD 树**加速；
- 正地理编码 = 地址 → 坐标，发生在写入侧，靠**第三方 API 或自建词库**；
- 逆地理编码 = 坐标 → 地址，用 `geo_shape` **点在多边形内**判断行政区，精确地址靠第三方 API。
