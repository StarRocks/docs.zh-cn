# StarRocks version 2.0

## 2.0

发布日期：2022年1月5日

### New Feature

- 外表
  - [实验功能]支持S3上的Hive外表功能 [参考文档](/using_starrocks/External_table.md#Hive外表)
  - DecimalV3支持外表查询 [#425](https://github.com/StarRocks/starrocks/pull/425)
- 实现存储层复杂表达式下推计算，获得性能提升
- Broker Load支持华为OBS [#1182](https://github.com/StarRocks/starrocks/pull/1182)
- 支持国密算法sm3
- 适配ARM类国产CPU：通过鲲鹏架构验证
- 主键模型（Primary Key）正式发布，该模型支持Stream Load、Broker Load、Routine Load，同时提供了基于Flink-cdc的MySQL数据的秒级同步工具。[参考文档](/table_design/Data_model.md#主键模型)

### Improvement

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

### Bugfix

- 修复Hive外表大量获取元数据超时的问题
- 修复物化视图创建报错信息不明确的问题
- 修复向量化引擎对`like`的实现 [#722](https://github.com/StarRocks/starrocks/pull/722)
- 修复`alter table`中谓词is的解析错误[#725](https://github.com/StarRocks/starrocks/pull/725)
- 修复`curdate`函数没办法格式化日期的问题

## 2.0.1

发布日期： 2022年1月21日

### Imporvement

- 优化Hive外表的隐式类型转换 [#2829](https://github.com/StarRocks/starrocks/pull/2829)
- 优化高并发查询时统计信息收集的锁竞争 [#2901](https://github.com/StarRocks/starrocks/pull/2901)
- 优化了CBO中一些统计信息和Union Node相关工作

### Bugfix

- 修复低基数全局字典在副本数据不一致时的问题 [#2700](https://github.com/StarRocks/starrocks/pull/2700)[#2765](https://github.com/StarRocks/starrocks/pull/2765)
- 修复导入时`exec_mem_limit`的问题 [#2693](https://github.com/StarRocks/starrocks/pull/2693)
- 修复primary key导入时触发OOM的问题 [#2743](https://github.com/StarRocks/starrocks/pull/2743)[#2777](https://github.com/StarRocks/starrocks/pull/2777)
- 修复MySQL外表大查询卡住的问题 [#2881](https://github.com/StarRocks/starrocks/pull/2881)

### Behavior Change
- 为了减小二进制包的大小，我们将访问S3相关的jar包移出，如果需要使用Hive外表访问S3，需要单独下载Hive_s3_lib