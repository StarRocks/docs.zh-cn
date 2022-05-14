# StarRocks version 2.0

## 2.0.0

发布日期：2022年1月5日

### 新功能

- 外表
  - [实验功能]支持S3上的Hive外表功能 [参考文档](/using_starrocks/External_table.md#Hive外表)
  - DecimalV3支持外表查询 [#425](https://github.com/StarRocks/starrocks/pull/425)
- 实现存储层复杂表达式下推计算，获得性能提升
- Broker Load支持华为OBS [#1182](https://github.com/StarRocks/starrocks/pull/1182)
- 支持国密算法sm3
- 适配ARM类国产CPU：通过鲲鹏架构验证
- 主键模型（Primary Key）正式发布，该模型支持Stream Load、Broker Load、Routine Load，同时提供了基于Flink-cdc的MySQL数据的秒级同步工具。[参考文档](/table_design/Data_model.md#主键模型)

### 功能优化

- 优化算子性能
  - 低基数字典性能优化[#791](https://github.com/StarRocks/starrocks/pull/791)
  - 单表的int scan性能优化 [#273](https://github.com/StarRocks/starrocks/issues/273)
  - 高基数下的`count(distinct int)` 性能优化 [#139](https://github.com/StarRocks/starrocks/pull/139) [#250](https://github.com/StarRocks/starrocks/pull/250)  [#544](https://github.com/StarRocks/starrocks/pull/544)[#570](https://github.com/StarRocks/starrocks/pull/570)
  - 执行层优化和完善 `Group by 2 int` / `limit` / `case when` / `not equal`
- 内存管理优化
  - 重构内存统计/控制框架，精确统计内存使用，彻底解决OOM
  - 优化元数据内存使用
  - 解决大内存释放长时间卡住执行线程的问题
  - 进程优雅退出机制，支持内存泄漏检查[#1093](https://github.com/StarRocks/starrocks/pull/1093)

### Bug 修复

- 修复Hive外表大量获取元数据超时的问题
- 修复物化视图创建报错信息不明确的问题
- 修复向量化引擎对`like`的实现 [#722](https://github.com/StarRocks/starrocks/pull/722)
- 修复`alter table`中谓词is的解析错误[#725](https://github.com/StarRocks/starrocks/pull/725)
- 修复`curdate`函数没办法格式化日期的问题

## 2.0.1

发布日期： 2022年1月21日

### 功能优化

- 优化StarRocks读取Hive外表时Hive外表隐式数据转换的功能。 [#2829](https://github.com/StarRocks/starrocks/pull/2829)
- 优化高并发查询场景下，StarRocks CBO优化器采集统计信息时的锁竞争问题。 [#2901](https://github.com/StarRocks/starrocks/pull/2901)
- 优化CBO的统计信息工作，UNION算子等。

### Bug 修复

- 修复副本的全局字典不一致而引起查询的问题。 [#2700](https://github.com/StarRocks/starrocks/pull/2700)[#2765](https://github.com/StarRocks/starrocks/pull/2765)
- 修复数据导入至StarRocks前设置参数`exec_mem_limit`不生效的问题。 [#2693](https://github.com/StarRocks/starrocks/pull/2693)
  > 参数`exec_mem_limit`用于指定数据导入时单个BE节点计算层使用的内存上限。
- 修复数据导入至StarRocks主键模型时触发OOM的问题。 [#2743](https://github.com/StarRocks/starrocks/pull/2743)[#2777](https://github.com/StarRocks/starrocks/pull/2777)
- 修复StarRocks在查询大数量级的MySQL外部表时的查询卡死问题。 [#2881](https://github.com/StarRocks/starrocks/pull/2881)

### Behavior Change

- StarRocks支持使用Hive外表访问创建在Hive外表上的Amazon S3外表。由于用于访问Amazon S3外表的jar包较大，因此StarRocks二进制产品包目前暂未包含该jar包。如有需要，请单击[Hive_s3_lib](https://cdn-thirdparty.starrocks.com/hive_s3_jar.tar.gz)进行下载。

## 2.0.2

发布日期： 2022年3月2日

### 功能优化

- 优化 FE 内存占用。通过设置参数 `label_keep_max_num`，控制一定时间内导入任务保留的最大数量，以避免在高频作业导入时，FE 内存占用过多而出现 Full GC。

### Bug 修复

- 修复 ColumnDecoder 异常，导致 BE 节点无响应的问题。
- 修复在导入 JSON 格式数据中设置了 jsonpaths 后不能自动识别 __op 字段的问题。
- 修复 Broker Load 导入数据过程中因为源数据发生变化而导致 BE 节点无响应的问题。
- 修复建立物化视图后，部分 SQL 语句报错的问题。
- 修复查询语句中同时存在低基数全局字典不支持的谓词时，导致查询失败的问题。

## 2.0.3

发布日期： 2022年3月14日

### Bug 修复

- 修复 BE 假死导致查询出错的问题。
- 修复对单 tablet 的表在做聚合操作时因无法得到合理的执行计划而导致查询失败的问题。[#3854](https://github.com/StarRocks/starrocks/issues/3854)
- 修复 FE 收集信息以构建低基数全局字典时可能导致死锁的问题。。[#3839](https://github.com/StarRocks/starrocks/issues/3839)

## 2.0.4

发布日期： 2022年4月18日

### Bug 修复

修复了如下 Bug：

- 在删列、新增分区、并克隆 Tablet 后，新旧 Tablet 的列 Unique ID 可能会不对应，由于系统使用共享的 Tablet Schema，可能导致 BE 停止服务。[#4514](https://github.com/StarRocks/starrocks/issues/4514)
- 向 StarRocks 外表导入数据时，如果设定的目标 StarRocks 集群的 FE 不是 Leader，则会导致 FE 停止服务。[#4573](https://github.com/StarRocks/starrocks/issues/4573)
- 明细模型的表同时执行表结构变更、创建物化视图时，可能导致数据查询错误。[#4839](https://github.com/StarRocks/starrocks/issues/4839)
- 通过改进为批量 publish version，解决 BE 可能因宕机而导致数据丢失的问题。[#3140](https://github.com/StarRocks/starrocks/issues/3140)

## 2.0.5

发布日期：2022年5月13日

升级建议：本次修复了一些跟数据存储或数据查询正确性相关的关键 Bug，建议您及时升级。

### Bug 修复

修复了如下 Bug：

- 【Critical Bug】通过改进为批量 publish version，解决 BE 可能因宕机而导致数据丢失的问题。[#3140](https://github.com/StarRocks/starrocks/issues/3140)
- 【Critical Bug】在数据写入中的一些特殊阶段，如果 Tablet 进行并完成迁移，数据会继续写入至原先 Tablet 对应的磁盘，导致数据丢失，进而导致查询错误。[#5160](https://github.com/StarRocks/starrocks/issues/5160)
- 【Critical Bug】在进行多个 DELETE 操作后，查询时，如果系统内部使用了低基数优化，则查询结果可能是错误的。[#5712](https://github.com/StarRocks/starrocks/issues/5712)
- 【Critical Bug】JOIN 查询的两个字段类型分别是 DOUBLE 和 VARCHAR 时，JOIN 查询结果可能错误。 [#5809](https://github.com/StarRocks/starrocks/pull/5809)
- 在数据导入中的某些特殊情形，可能一些副本的某些版本还未生效，却被 FE 标记为生效，导致查询时出现找不到对应版本数据的错误。[#5153](https://github.com/StarRocks/starrocks/issues/5153)
- `SPLIT` 函数使用 `NULL` 参数时，会导致 BE 停止服务。[#4092](https://github.com/StarRocks/starrocks/issues/4092)  
- 从 Apache Doris 0.13 升级到 StarRocks 1.19.x 并运行一段时间，再升级到 StarRocks 2.0.1，可能会升级失败。[#5309](https://github.com/StarRocks/starrocks/issues/5309)