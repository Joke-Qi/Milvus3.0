# Milvus3.0
## 一、Milvus主要版本与时间线

1、Milvus2.0（2022年1月）：全新云原生架构、采用存算分离设计

2、Milvus2.1（2022年4月）：性能，百万级数据集的检索延迟<=5ms

3、Milvus2.2（2022年11月）：引入了DiskANN，向量索引可以不全部放内存

4、Milvus2.3（2023年8月）：支持GPU，实现了比纯CPU版本快10倍的性能

5、Milvus2.4（2024年3月）：CAGRA GPU索引把搜索性能再提升50倍

6、Milvus2.6（2025年6月）：架构升级和数据胡（Data Lake）的初步集成

7、Milvus3.0（2026年7月29日）：确立了“湖原生（Lake-Native）”的核心方向

## 二、Milvus3.0新特性

###### **让向量数据从“数据孤岛”走向“湖仓一体”**

###### **从“单一检索”走向“全链路检索平台”**

### 1、湖原生检索：为全新的Vector Lakebase（向量数据湖基座）架构提供底层能力保障

#### Vector Lakebase 到底是什么？

以数据湖作为统一数据底座，以向量索引作为检索加速层，以 Milvus 作为检索引擎，构建面向 AI 应用的数据存储与检索基础设施。

传统向量数据库与湖原生向量检索，最大的区别在于：向量数据是否必须复制到向量数据库内部，才能进行检索。

#### 传统向量数据库模式

假设企业有 1 亿条文档数据，原始数据已经存储在数据湖中。

传统架构通常是：
<img width="625" height="278" alt="image" src="https://github.com/user-attachments/assets/e1a827e0-0bf4-49c9-a090-a4759adf8211" />

这种模式的问题：

- 数据湖保存一份原始数据。
- Milvus 又保存一份向量数据及相关字段。
- 数据发生变化时，需要考虑两套数据的同步。
- 数据规模越大，数据复制、存储和维护成本越高。

#### 湖原生向量检索模式

湖原生架构的核心变化是：数据可以留在数据湖中，Milvus 直接面向湖中的数据建立索引并提供检索服务。
<img width="1236" height="614" alt="image" src="https://github.com/user-attachments/assets/3bf6a1ad-17b9-4283-9a60-7b66faa13bc3" />
这里有一个关键概念：零拷贝（Zero-copy）。

Milvus 3.0 引入了 External Collections（外部 Collection），可以将数据湖中的 Iceberg、Parquet、Lance、Vortex 等数据集映射为 Milvus Collection。

Milvus 不需要把整份源数据复制到自身的 Collection 中，而是在外部数据集上构建和服务索引。外部 Collection 目前是只读的，数据变化通过刷新外部 Collection 的机制感知。

### 2、客户端向 Milvus 发送一次查询请求，完整经过哪些环节？

我们假设有一个真实的业务场景：

某企业建设了一个 AI 知识库，数据保存在对象存储中，采用 Iceberg 表格式，Milvus 负责向量检索。

用户在客户端输入：

> 帮我查找关于量子通信专线建设的相关技术资料。

客户端通过 Embedding 模型，将这段文字转换成一个 1024 维的向量，然后向 Milvus 发送 Search 请求。

注意：Embedding 模型通常由客户端应用或独立的模型服务负责，不是 Milvus 自动替客户端完成的。

#### 完整请求链路总览
<img width="1014" height="1260" alt="image" src="https://github.com/user-attachments/assets/817842c5-89cb-4cf1-9ffc-0ac3d6a62555" />

#### 第一步：客户端发起查询请求

客户端首先完成两个动作：

动作 A：将用户问题转换为向量。

例如：
```
用户问题：
帮我查找关于量子通信专线建设的相关技术资料。

          ↓ Embedding 模型

查询向量：
[0.021, -0.134, 0.562, 0.087, ...]
```

假设模型输出 1024 维向量。

动作 B：通过 SDK 向 Milvus 发送 Search 请求。

例如使用 Python SDK：

```
from pymilvus import MilvusClientclient = MilvusClient(    uri="http://milvus.example.com:19530",    token="username:password")query_vector = [0.021, -0.134, 0.562, 0.087]  # 示例，实际1024维results = client.search(    collection_name="knowledge_base",    data=[query_vector],    anns_field="embedding",    search_params={        "metric_type": "COSINE",        "params": {"ef": 128}    },    limit=5,    output_fields=["title", "content", "source"])
```

这里客户端向 Milvus 传递的核心信息包括：

| 参数            | 作用                    |
| --------------- | ----------------------- |
| collection_name | 指定查询哪个 Collection |
| data            | 待检索的查询向量        |
| anns_field      | 指定向量字段            |
| metric_type     | 指定相似度计算方式      |
| search_params   | 检索算法参数            |
| limit           | 返回 Top 5              |
| output_fields   | 希望返回哪些业务字段    |

此时客户端并不知道向量实际存储在哪个对象存储节点、索引分布在哪些 Segment，也不需要知道。

它只需要调用统一的 Milvus API。

#### 第二步：网络接入与 Proxy 接收请求

客户端发送请求后：

1. SDK 将 Search 请求序列化。
2. 通过 gRPC 或 REST 接口发送到 Milvus 服务地址。
3. 请求经过 Kubernetes Service、Ingress、负载均衡器等接入组件。
4. 被转发到一个可用的 Proxy 节点。

Proxy 是 Milvus 的统一访问入口。

它主要负责：

- 身份认证与权限校验。
- Collection 是否存在。
- 查询字段和向量维度是否合法。
- 检查检索参数。
- 生成或准备查询计划。
- 根据 Shard 路由信息，将请求分发给对应的执行节点。
- 汇总各节点的检索结果。

这里有一个容易混淆的地方：

Proxy 不负责真正完成大规模向量相似度计算，它主要承担请求接入、路由和结果归并工作。（具体milvus架构可以参考官方文档：https://milvus.io/docs/zh/architecture_overview.md）

<img width="2194" height="784" alt="image" src="https://github.com/user-attachments/assets/0eaccfa5-1635-489a-b99a-e070a5d15658" />

#### 第三步：Coordinator 提供元数据与路由信息

Proxy 收到请求后，需要知道：

- 当前 Collection 有多少个 Shard？
- 每个 Shard 分布在哪些节点？
- Collection 对应哪些 Segment？
- 当前有哪些索引可用？
- 哪些 Query Node 负责相关历史数据？

这些信息来自 Milvus 的元数据和集群协调体系。

Coordinator 负责集群拓扑、任务调度、查询视图管理、Segment 分配等工作。

元数据存储通常由 etcd 承担。

需要注意，Coordinator 不一定在每次 Search 请求中都被直接调用。Proxy 可以使用本地缓存的路由和拓扑信息，只有缓存失效或需要更新时才向协调体系获取信息。

#### 第四步：Proxy 将请求分发到检索执行节点

这是整个检索过程最重要的环节之一。

Milvus 的数据并不是简单地全部放在一台机器上，而是分布在多个 Shard 和 Segment 中。

Milvus 3.x 的检索体系可以理解为两个部分：

<img width="1036" height="226" alt="image" src="https://github.com/user-attachments/assets/cdbd8d3c-e44d-41c9-ab6e-f904b222cc42" />

为什么要区分这两个节点？

假设企业知识库今天刚刚新增了一份文件：
- 文件 A：刚刚写入，还没有完成封存和索引构建。
- 文件 B：昨天已经写入，索引构建完成。
- 文件 C：上个月已经写入，属于历史数据。
如果用户现在查询：
- 文件 A 可能在 Streaming Node 的 Growing Segment 中。
- 文件 B、C 可能在 Query Node 管理的 Sealed Segment 中。
因此，检索请求需要兼顾实时数据和历史数据。

在传统的 Milvus 3.x 分布式检索路径中，Proxy 会向相关 Shard 的 Streaming Node 并发发送请求，Streaming Node 查询本地 Growing 数据，同时协调 Query Node 查询历史数据。

注意：这里说的是 Milvus 的原生 Collection 检索路径。对于湖原生 External Collection，还需要考虑外部数据集的索引和数据读取路径，不能简单地把所有外部数据都理解成已经加载在 Streaming Node 或 Query Node 的内存中。

#### 第五步：Milvus 在数据湖中定位数据并执行向量检索

这一步是湖原生检索最值得关注的地方。

先看数据湖中可能存在的数据结构：

<img width="1042" height="452" alt="image" src="https://github.com/user-attachments/assets/d5072f32-df53-42a9-80d3-fad9343fdb2b" />

在 Milvus 3.0 的湖原生方案中，可以把整个检索过程理解为以下几个步骤。

① 定位数据集

Milvus 通过外部 Collection 与数据湖中的表建立映射关系。

例如：

```
Milvus Collection:
knowledge_base

对应外部数据源:
s3://company-lake/knowledge/metadata.json

外部格式:
Iceberg

字段映射:
id        → doc_id
embedding → embedding
title     → title
content   → content
```

Milvus 不需要将整份 Iceberg 表重新导入自身数据库。

② 定位索引

Milvus 根据当前数据集版本和索引元信息，定位可用的向量索引。

例如：

```
Collection
   ↓
Manifest
   ↓
索引元数据
   ↓
HNSW 向量索引
```

索引的作用是减少搜索范围，避免每次查询都遍历全部向量。

③ 执行 ANN 向量搜索

假设有 1000 万条向量。

用户的查询向量为：

$$
Q=[q_1,q_2,\ldots,q_{1024}]
$$

Milvus 根据向量索引搜索与 Q 最相似的候选向量。

例如使用 COSINE 相似度：

$$
sim(Q,V)=\frac{Q\cdot V}{\|Q\|\|V\|}
$$

得到候选结果：

| 文档 ID | 相似度 |
| ------- | ------ |
| DOC-001 | 0.96   |
| DOC-023 | 0.93   |
| DOC-105 | 0.91   |
| DOC-209 | 0.88   |
| DOC-315 | 0.85   |

这里的 ANN 是 Approximate Nearest Neighbor，即近似最近邻搜索。

④ 根据候选 ID 获取业务数据

这一步尤其重要。

向量索引通常首先帮助我们找到候选 ID 和相似度，而业务真正需要的是：

- 文档标题。
- 文档内容。
- 文档来源。
- 创建时间。
- 业务标签。

因此，Milvus 还需要根据候选 ID 定位对应的业务数据。

这涉及湖原生架构中的一个关键技术：Loon（Storage v3）。

Loon 面向对象存储，使用 ColumnGroups 组织数据，并利用对齐的行 ID 支持更高效的窄范围字段读取。

可以把它理解成：

```
向量索引
    ↓
找到候选记录 ID
    ↓
定位对应的行
    ↓
读取 title、content、source 等字段
    ↓
组合成完整结果
```

为什么需要 Loon？

因为传统分析型列式数据存储更擅长大范围扫描，但向量搜索经常是先找到少量候选记录，再读取这些记录的具体字段。

如果每次只需要读取 5 条记录，却必须从对象存储中读取大量无关数据，就会产生严重的 I/O 放大。

Loon 通过优化列组织和点读路径，减少这类额外读取开销。

这也是湖原生检索能够兼顾数据湖存储成本和在线检索性能的重要技术环节。 

#### 第六步：多节点结果归并与 TopK 排序

由于数据分布在不同 Shard、Segment 或执行节点中，每个节点只负责自己的一部分数据。

例如：

<img width="1032" height="704" alt="image" src="https://github.com/user-attachments/assets/e0f6af52-0e60-4313-896c-4edfe3278130" />

归并过程可以分成三个层次：

1. Segment 级别：对单个 Segment 的检索结果进行局部处理。
2. Shard 级别：Streaming Node 汇总本 Shard 的实时数据与历史数据结果。
3. Proxy 级别：将多个 Shard 的结果合并，完成全局排序和 TopK 截取。

例如，用户只要求返回 5 条记录，那么最终只需要返回全局排名最高的 5 条记录。

Milvus 3.0 还支持将更多排序、聚合和结果处理放入检索引擎内部，减少应用端的额外处理。

#### 第七步：Milvus 将结果返回客户端

最终，Milvus 将结果通过原有的 gRPC 或 REST 请求通道返回客户端。

示例：

```
[
  {
    "id": "DOC-001",
    "distance": 0.96,
    "entity": {
      "title": "量子通信专线建设技术方案",
      "content": "量子通信专线主要采用...",
      "source": "技术文档"
    }
  },
  {
    "id": "DOC-023",
    "distance": 0.93,
    "entity": {
      "title": "量子密钥分发网络研究",
      "content": "量子密钥分发技术...",
      "source": "研究报告"
    }
  }
]
```

客户端拿到数据后，可以继续做：

- 将检索结果交给 Rerank 模型重新排序。
- 将相关文本片段交给大语言模型。
- 由大语言模型组织答案。
- 最终返回带引用来源的智能问答结果。

至此，一次完整的向量检索就结束了。

### 3、结合 Vector Lakebase，数据在查询之前是如何准备的？

前面是一次 Search 请求的在线过程。

但如果要真正理解 Vector Lakebase，还必须理解查询发生之前，数据是如何进入数据湖、如何建立索引的。

这是一个独立于在线查询的准备过程。

#### 数据接入与索引构建

假设企业每天新增 10 万份技术文档。

完整准备过程如下：

<img width="1370" height="976" alt="image" src="https://github.com/user-attachments/assets/1c9a9077-702a-4c5a-b19e-a68bd28fb8c0" />

这里还有一个非常重要的区别：

传统模式中，数据写入 Milvus 的原生 Collection，会经过写入日志、实时数据处理、Segment 封存、索引构建等流程。

湖原生模式中，外部 Collection 的源数据仍然保留在数据湖，Milvus 负责在外部数据集上建立和维护检索索引。

当外部数据集发生变化时，可以通过刷新机制识别新增数据，并增量构建索引，而不是每次重新导入整张表。

#### 两种模式的核心差异

| 对比维度 | 传统向量数据库         | 湖原生检索                 |
| -------- | ---------------------- | -------------------------- |
| 数据存储 | Milvus 内部 Collection | 数据湖外部数据集           |
| 数据复制 | 通常需要导入           | 外部 Collection 支持零拷贝 |
| 索引     | 内部构建和管理         | 可在外部数据上构建和服务   |
| 数据更新 | 数据库写入、更新       | 外部数据集刷新             |
| 检索接口 | Milvus API             | 兼容的 Milvus API          |
| 数据读取 | 内部存储及缓存         | 外部对象存储及缓存         |
| 主要价值 | 高性能在线检索         | 数据湖与向量检索结合       |

需要强调：零拷贝不等于不产生存储开销，也不等于查询完全不访问对象存储。索引本身需要存储，查询也可能需要读取外部数据。它解决的是不必为了检索而维护一份完整的重复数据副本。

### 4、如何理解“湖原生检索为 Vector Lakebase 提供底层能力保障”？

这句话真正强调的是三个层次。

<img width="1396" height="772" alt="image" src="https://github.com/user-attachments/assets/dfb2b516-043c-4f5d-bee1-0961028213a6" />

湖原生检索通过将高性能向量索引和检索能力直接构建在数据湖之上，减少数据复制和同步环节，实现数据湖中海量非结构化数据的统一检索，为 Vector Lakebase 提供高效、低成本、可扩展的向量检索基础设施。

### 附录：相关技术名词

<img width="1198" height="600" alt="image" src="https://github.com/user-attachments/assets/76b599b0-ec7a-4af3-afe3-dc90efc6040e" />


