# StarRocks version 2.0

## 2.0.0

发布日期：2022年1月5日

### New Feature

- 外表
  - [实验功能] 支持 S3 上的 Hive 外表功能 [参考文档](/using_starrocks/External_table.md#Hive外表)
  - DecimalV3 支持外表查询 [#425](https://github.com/StarRocks/starrocks/pull/425)
- 实现存储层复杂表达式下推计算，获得性能提升
- Broker Load 支持华为 OBS [#1182](https://github.com/StarRocks/starrocks/pull/1182)
- 支持国密算法 sm3
- 适配 ARM 类国产 CPU：通过鲲鹏架构验证
- 主键模型（Primary Key）正式发布，该模型支持 Stream Load、Broker Load、Routine Load，同时提供了基于 Flink-cdc 的 MySQL 数据的秒级同步工具。[参考文档](/table_design/Data_model.md#主键模型)

### Improvement

- 优化算子性能
  - 低基数字典性能优化 [#791](https://github.com/StarRocks/starrocks/pull/791)
  - 单表的 int scan 性能优化 [#273](https://github.com/StarRocks/starrocks/issues/273)
  - 高基数下的 `count(distinct int)` 性能优化 [#139](https://github.com/StarRocks/starrocks/pull/139) [#250](https://github.com/StarRocks/starrocks/pull/250)  [#544](https://github.com/StarRocks/starrocks/pull/544) [#570](https://github.com/StarRocks/starrocks/pull/570)
  - 执行层优化和完善 `Group by 2 int` / `limit` / `case when` / `not equal`
- 内存管理优化
  - 重构内存统计/控制框架，精确统计内存使用，彻底解决 OOM
  - 优化元数据内存使用
  - 解决大内存释放长时间卡住执行线程的问题
  - 进程优雅退出机制，支持内存泄漏检查 [#1093](https://github.com/StarRocks/starrocks/pull/1093)

### Bugfix

- 修复 Hive 外表大量获取元数据超时的问题
- 修复物化视图创建报错信息不明确的问题
- 修复向量化引擎对 `like` 的实现 [#722](https://github.com/StarRocks/starrocks/pull/722)
- 修复 `alter table` 中谓词 is 的解析错误 [#725](https://github.com/StarRocks/starrocks/pull/725)
- 修复 `curdate` 函数没办法格式化日期的问题

## 2.0.1

发布日期： 2022年1月21日

### Improvement

- 优化 StarRocks 读取 Hive 外表时 Hive 外表隐式数据转换的功能。 [#2829](https://github.com/StarRocks/starrocks/pull/2829)
- 优化高并发查询场景下，StarRocks CBO 优化器采集统计信息时的锁竞争问题。 [#2901](https://github.com/StarRocks/starrocks/pull/2901)
- 优化 CBO 的统计信息工作，UNION 算子等。

### Bugfix

- 修复副本的全局字典不一致而引起查询的问题。 [#2700](https://github.com/StarRocks/starrocks/pull/2700) [#2765](https://github.com/StarRocks/starrocks/pull/2765)
- 修复数据导入至 StarRocks 前设置参数 `exec_mem_limit` 不生效的问题。 [#2693](https://github.com/StarRocks/starrocks/pull/2693)
  > 参数 `exec_mem_limit` 用于指定数据导入时单个 BE 节点计算层使用的内存上限。
- 修复数据导入至 StarRocks 主键模型时触发 OOM 的问题。 [#2743](https://github.com/StarRocks/starrocks/pull/2743) [#2777](https://github.com/StarRocks/starrocks/pull/2777)
- 修复 StarRocks 在查询大数量级的 MySQL 外部表时的查询卡死问题。 [#2881](https://github.com/StarRocks/starrocks/pull/2881)

### Behavior Change

- StarRocks 支持使用 Hive 外表访问创建在 Hive 外表上的 Amazon S3 外表。由于用于访问 Amazon S3 外表的 jar 包较大，因此 StarRocks 二进制产品包目前暂未包含该 jar 包。如有需要，请单击 [Hive_s3_lib](https://cdn-thirdparty.starrocks.com/hive_s3_jar.tar.gz) 进行下载。

## 2.1

发布日期： 2022年2月25日

### New Features

- 【公测中】支持通过外表的方式查询 Apache Iceberg 数据湖中的数据，帮助您实现对数据湖的极速分析。TPC-H 测试集的结果显示，查询 Apache Iceberg 数据时，StarRocks 的查询速度是 Presto 的 **3 - 5** 倍。相关文档，请参见 [Apache Iceberg 外表](../using_starrocks/External_table.md/#Apache Iceberg外表)

- 【公测中】发布 Pipeline 执行引擎，可以自适应调节查询的并行度。您无需手动设置 session 级别的变量 parallel_fragment_exec_instance_num。并且，在部分高并发场景中，相较于历史版本，新版本性能提升两倍。
- 支持 CTAS（CREATE TABLE AS SELECT），基于查询结果创建表并且导入数据，从而简化建表和 ETL 操作。相关文档，请参见 [CREATE TABLE AS SELECT](../sql-reference/sql-statements/data-definition/CREATE%20TABLE%20AS%20SELECT.md)。
- 支持 SQL 指纹，针对慢查询中各类 SQL 语句计算出 SQL 指纹，方便您快速定位慢查询。相关文档，请参见 [SQL 指纹](../administration/Query_planning.md/#sql指纹)。
- 新增函数 [ANY VALUE](../sql-reference/sql-functions/date-time-functions/any_value.md)，[ARRAY REMOVE]((../sql-reference/sql-functions/date-time-functions/array_remove.md)，哈希函数 [SHA2](../sql-reference/sql-functions/date-time-functions/sha5.md)。

### Improvement

- 优化 Compaction 的性能，支持导入 10000 列的数据。
- 优化 StarRocks 首次 Scan 和 Page Cache 的性能。通过降低随机 I/O ，提升 StarRocks 首次 Scan 的性能，如果首次 Scan 的磁盘为 SATA 盘，则性能提升尤为明显。另外，StarRocks 的 Page Cache 支持直接存放原始数据，无需经过 Bitshuffle 编码。因此读取 StarRocks 的 Page Cache 时无需额外解码，提高缓存命中率，进而大大提升查询效率。
- 支持主键模型（Primary Key Model）变更表结构（Schema Change），您可以执行 `ALTER TABLE` 增删和修改索引。
- 【公测中】支持字符串最大长度为 1MB。
- 优化 JSON 导入性能，并去除了 JSON 导入中单个 JSON 文件不超过 100MB 大小的限制。
- 优化 Bitmap Index 性能。
- 优化通过外表方式读取 Hive 数据的性能，支持 Hive 的存储格式为 CSV。
- 支持建表语句的时间戳字段定义为 DEFAULT CURRENT_TIMESTAMP。
- 支持导入带有多个分隔符的 CSV 文件。

### Bugfix

- 修复在导入 JSON 格式数据中设置了 jsonpaths 后不能自动识别 __op 字段的问题。
- 修复 Broker Load 导入数据过程中因为源数据发生变化而导致 BE 节点挂掉的问题。
- 修复建立物化视图后，部分 SQL 语句报错的问题。
- 修复 Routine Load 中使用带引号的 jsonpath 会报错的问题。
- 修复查询列数超过 200 列后，并发性能明显下降的问题。

### Behavior Change

修改关闭 Colocation Group 的 HTTP Restful API。为了使语义更好理解，关闭 Colocation Group 的 API 修改为 `POST /api/colocate/group_unstable`（旧接口为 `DELETE /api/colocate/group_stable` ）。

> 如果需要重新开启 Colocation Group ，则可以使用 API `POST /api/colocate/group_stable`。

### Others

flink-source-connector 支持 Flink 批量读取 StarRocks 数据，实现了直连并行读取 BE 节点、自动谓词下推等特性。相关文档，请参见 [Flink Connector](../unloading/Flink_connector.md)。
