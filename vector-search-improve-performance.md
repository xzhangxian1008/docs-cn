---
title: 优化向量搜索性能
summary: 了解优化 TiDB 向量搜索性能的最佳实践。
---

# 优化向量搜索性能

在 TiDB 中，你可以通过向量搜索功能进行近似最近邻（Approximate Nearest Neighbor，简称 ANN）搜索，查找与给定的图像、文档等相似的结果。为了提升查询性能，请参考以下最佳实践。

> **警告：**
>
> 向量搜索目前为实验特性，不建议在生产环境中使用。该功能可能会在未事先通知的情况下发生变化。如果发现 bug，请在 GitHub 上提 [issue](https://github.com/pingcap/tidb/issues) 反馈。

## 为向量列添加向量搜索索引

[向量搜索索引](/vector-search-index.md)可显著提高向量搜索查询的性能，通常能提高 10 倍或更多，而召回率仅略有下降。

## 确保向量索引已完全构建

当插入大批量向量数据时，可能会有部分数据处于 Delta 层等待后续的持久化，这一部分的数据会在持久化后才会开始构建向量索引。在向量搜索索引完全构建好后，向量搜索性能才能达到最佳水平。要查看索引构建进度，可参阅[查看索引构建进度](/vector-search-index.md#查看索引构建进度)。

## 减少向量维数或缩短嵌入时间

随着向量维度增加，向量搜索索引和查询的计算复杂度会显著增加，因为需要进行更多的浮点数比较运算。

为了优化性能，可以考虑尽可能地减少向量的维数。这通常需要切换到另一种嵌入模型。在切换模型时，你需要评估改变嵌入模型对向量查询准确性的影响。

一些嵌入模型，如 OpenAI `text-embedding-3-large`，支持[缩短向量嵌入](https://openai.com/index/new-embedding-models-and-api-updates/)，即在不丢失向量表示的概念特征的情况下，从向量序列末尾移除一些数字。你也可以使用这种嵌入模型来减少向量维数。

## 在结果输出中排除向量列

向量嵌入数据通常很大，而且只在搜索过程中使用。通过从查询结果中排除向量列，可以显著减少 TiDB 服务器和 SQL 客户端之间传输的数据量，从而提高查询性能。

要从结果输出中排除向量列，请在 `SELECT` 语句中明确指定需要检索的列，而不是使用 `SELECT *` 检索所有列。

## 预热索引

当访问一个从未被使用过或长时间未被使用过的索引（冷访问）时，TiDB 需要从云存储或磁盘（而不是内存）加载整个索引。这个过程需要一定的时间，往往会导致较高的查询延迟。此外，如果集群长时间（比如数小时）内没有进行 SQL 查询，计算资源就会被回收，这样下次访问时就会变成冷访问。

要避免这种查询延迟，可在实际工作负载前，使用类似的向量搜索查询对索引进行预热。