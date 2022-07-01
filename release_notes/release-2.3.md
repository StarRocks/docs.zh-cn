# StarRocks version 2.3

## 2.3.0

发布日期： 2022 年 7 月  日

### 新增特性

- 主键模型支持完整的 DELETE WHERE 语法。相关文档，请参见[主键模型导入]()。

- 主键模型支持持久化主键索引，避免主键索引占用过大内存空间。相关文档，请参见[主键模型介绍](../table_design/Data_model.md)。

- 执行 Routine Load 导入场景下，构建全局字典进行低基数字段的查询优化时，支持全局字典更新，从而提升查询性能。

- Stream Load 提供事务接口，支持将“发送数据”和“提交事务”拆分执行，实现跨系统的两阶段提交，提升高并发场景下的导入性能。例如从 Flink 导入至 StarRocks 时，支持同步接收上游数据和发送数据，并且在合适的时候“提交事务”以完成一个批次的导入，客户端无需缓存完整批次数据，能够减少客户端的内存使用，并且实现“精确一次 (Exactly-Once)”语义。此外，还支持小文件在一个批次中合并导入等功能。相关文档，请参见 [Stream Load 事务接口]()。

- 支持异步执行 CREATE TABLE AS SELECT 语句并将结果写入新表。相关文档，请参见 [CREATE TABLE AS SELECT](../sql-reference/sql-statements/data-definition/CREATE%20TABLE%20AS%20SELECT.md)。

- 资源组相关功能：
  - 支持监控资源组：可在审计日志中查看查询所属的资源组，并通过相关 API 获取资源组的监控信息。相关文档，请参见xxx。
  - 支持限制大查询的 CPU、内存、或 I/O 资源；可通过匹配分类器将查询路由至资源组，或者设置会话变量直接为查询指定资源组。相关文档，请参见xxx。

- 支持通过 JDBC 外表查询 Oracle、PostgreSQL、MySQL、SQL Server、ClickHouse 等数据库，查询时支持谓词下推。相关文档，请参见 [外部表](../using_starrocks/External_table.md)。

- 【Preview】发布全新数据源 Connector 框架，支持用户自定义外部数据目录（External Catalog）并接入 Apache Hive™ ，无需创建外部表，即可直接分析 Apache Hive™。相关文档，请参见 [外部表](../using_starrocks/External_table.md)。

- 新增如下函数：
  - window_funnel
  - ntile
  - bitmap_union_count 、array_to_bitmap、base64_to_bitmap
  - week、time_slice

### 功能优化

- 优化合并机制（Compaction），对较大的元数据进行合并操作，避免因数据高频更新而导致短时间内元数据挤压，占用较多磁盘空间。

- 优化导入 Parquet 文件和压缩文件格式的性能，（提供XX倍提升）。

- 优化创建物化视图的性能，在部分场景下创建速度提升近 10 倍。

- 优化算子性能：
  - TopN，sort 算子（XX倍提升？）
  - 包含函数的等值比较运算符下推至 scan 算子时，支持使用 Zone Map 索引。

- 优化 Apache Hive™ 外表功能。相关文档，请参见xxx。
  - 当 Apache Hive™ 的数据存储采用 Parquet、ORC、CSV 格式时，支持 Hive 表执行 ADD COLUMN、REPLACE COLUMN 等表结构变更（Schema Change）。
  - 支持 Hive 资源修改 `hive.metastore.uris`。

- 优化 Apache Iceberg 外表功能，创建 Iceberg 资源时支持使用自定义目录（Catalog）。相关文档，请参见xxx。

- 优化 Elasticsearch 外表功能，支持取消探测 Elasticsearch 集群数据节点的地址。相关文档，请参见xxx。

- 当 sum() 中输入的值为 STRING 类型且为数字时，则自动进行隐式转换。

- year、month、day 函数支持 DATE 数据类型。

- ###  Bug修复

- 修复了如下 Bug：

  - Tablet 过多导致 CPU 占用率过高的问题。[#5875](https://starrocks.atlassian.net/browse/SR-5875)
  - 导致出现"fail to prepare tablet reader"报错提示的问题。[#7248](https://starrocks.atlassian.net/browse/SR-7248)、 [#7854](https://starrocks.atlassian.net/browse/SR-7854)、 [#8257](https://starrocks.atlassian.net/browse/SR-8257)
  - FE 重启失败的问题。[#5642](https://github.com/StarRocks/starrocks/issues/5642 )、[#4969](https://github.com/StarRocks/starrocks/issues/4969 )、[#5580](https://github.com/StarRocks/starrocks/issues/5580)
  - CTAS 语句中调用 JSON 函数时报错的问题。[#6498](https://github.com/StarRocks/starrocks/issues/6498)

- ###  其他

- 提供集群管理工具 StarGo，提供集群部署、启停、升级、回滚、多集群管理等多种能力。相关文档，请参见xxx。
- 支持在 AWS 上使用 CloudFormation 快速创建 StarRocks 集群。相关文档，请参见xxx。
