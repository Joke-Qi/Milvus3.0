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

