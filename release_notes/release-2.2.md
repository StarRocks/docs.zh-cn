# StarRocks version 2.2

## 2.2.0

发布日期： 2022 年 4 月 22 日

### 新功能

- 【公测中】发布资源组管理功能。通过使用资源组来控制 CPU、内存的资源使用，让不同租户的大小查询在同一集群执行时，既能实现资源隔离，又能合理使用资源。相关文档，请参见[资源组](../administration/resource_group.md)。
- 【公测中】实现 Java UDF 框架，支持使用 Java 语法编写 UDF（用户自定义函数），扩展 StarRocks 的函数功能。相关文档，请参见 [Java UDF](../using_starrocks/JAVA_UDF.md)。
- 导入数据至主键模型时，支持更新部分列。在订单更新、多流 JOIN 等实时数据更新场景下，仅需要更新与业务相关的列。相关文档，请参见 [主键模型的表支持部分更新](../loading/PrimaryKeyLoad.md#部分更新)。
- 【公测中】支持 JSON 数据类型和函数。相关文档，请参见 [JSON](../sql-reference/sql-statements/data-types/JSON.md)。
- 支持通过外表查询 Apache Hudi 的数据，进一步完善了数据湖分析的功能。相关文档，请参见 [Apache Hudi 外表](../using_starrocks/External_table.md/#apache-hudi外表)。
- 新增如下函数:
  - ARRAY 函数，[array_agg](https://github.com/StarRocks/docs.zh-cn/pull/253)、[array_sort](https://github.com/StarRocks/docs.zh-cn/pull/271)、[array_distinct](https://github.com/StarRocks/docs.zh-cn/pull/266)、[array_join](https://github.com/StarRocks/docs.zh-cn/pull/282)、[reverse](https://github.com/StarRocks/docs.zh-cn/pull/272)、[array_slice](https://github.com/StarRocks/docs.zh-cn/pull/297) [、](https://github.com/StarRocks/docs.zh-cn/pull/297) [array_concat](https://github.com/StarRocks/docs.zh-cn/pull/297) [、](https://github.com/StarRocks/docs.zh-cn/pull/297) [array_difference](https://github.com/StarRocks/docs.zh-cn/pull/297) [、](https://github.com/StarRocks/docs.zh-cn/pull/297) [array_overlap](https://github.com/StarRocks/docs.zh-cn/pull/297) [、](https://github.com/StarRocks/docs.zh-cn/pull/297) [array_intersect](https://github.com/StarRocks/docs.zh-cn/pull/297)。
  - BITMAP 函数，包括 [bitmap_max](https://github.com/StarRocks/docs.zh-cn/pull/374) [、](https://github.com/StarRocks/docs.zh-cn/pull/374) [bitmap_min](https://github.com/StarRocks/docs.zh-cn/pull/374)。
  - 其他函数：[retention](https://github.com/StarRocks/docs.zh-cn/pull/269)、[square](https://github.com/StarRocks/docs.zh-cn/pull/364)。

### 功能优化

- 重构CBO优化器的 Parser 和 Analyzer，优化代码结构并支持 Insert with CTE 等语法。提升复杂查询的性能，包括公用表表达式（Common Table Expression，CTE）复用等。
- 优化查询Apache Hive外表中基于对象存储（Amazon S3、阿里云OSS、腾讯云COS）的外部表的性能，优化后基于对象存储的查询性能可以与基于HDFS的查询性能基本持平。支持ORC格式文件的延迟物化，提升小文件查询性能。相关文档，请参见  [Apache Hive 外表](../using_starrocks/External_table.md/#hive外表)。
- 通过外表查询 Apache Hive 的数据时，缓存更新通过定期消费 Hive Metastore 的事件（包括数据变更、分区变更等），实现自动增量更新元数据。并且，还支持查询 Apache Hive 中 DECIMAL 和 ARRAY 类型的数据。相关文档，请参见 [Apache Hive 外表](../using_starrocks/External_table.md/#hive外表)。
- 优化 UNION ALL 算子性能，性能提升可达2-25倍。
- 正式发布 Pipeline 引擎，支持自适应调节查询的并行度，并且优化了 Pipeline 引擎的 Profile。提升了高并发场景下小查询的性能。
- 导入 CSV 文件时，支持使用多个字符作为行分隔符。

### 修复 Bug

- 修复主键模型的表导入数据和 COMMIT 时产生死锁的问题。[#4998](https://github.com/StarRocks/starrocks/pull/4998)
- 解决 FE（包含 BDBJE）的一系列稳定性问题。[#4428](https://github.com/StarRocks/starrocks/pull/4428)、[#4666](https://github.com/StarRocks/starrocks/pull/4666)、[#2](https://github.com/StarRocks/bdb-je/pull/2)
- 修复 SUM 函数对大量数据求和时返回结果溢出的问题。[#3944](https://github.com/StarRocks/starrocks/pull/3944)
- 修复 ROUND 和 TRUNCATE 函数返回结果的精度问题。[#4256](https://github.com/StarRocks/starrocks/pull/4256)
- 修复 SQLancer 的一系列问题，请参见 [SQLancer 相关 issues](https://github.com/StarRocks/starrocks/issues?q=is%3Aissue++label%3Asqlancer++milestone%3A2.2)。

### 其他

- Flink 连接器 flink-connector-starrocks 支持 Flink 1.14 版本。
