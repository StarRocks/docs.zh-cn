# StarRocks version 1.19

## v1.19.0

发布日期：2021年10月23日

### New Feature

* 实现Global Runtime Filter，可以支持对shuffle join实现Runtime filter。
* 默认开启CBO Planner，完善了colocated join / bucket shuffle / 统计信息等功能。[参考文档](/using_starrocks/Cost_based_optimizer.md)
* [实验功能] 发布主键模型（Primary Key）：为更好地支持实时/频繁更新功能，StarRocks新增了一种表的类型: 主键模型。该模型支持Stream Load、Broker Load、Routine Load，同时提供了基于Flink-cdc的MySQL数据的秒级同步工具。[参考文档](/table_design/Data_model.md#主键模型)
* [实验功能] 新增外表写入功能。支持将数据通过外表方式写入另一个StarRocks集群的表中，以解决读写分离需求，提供更好的资源隔离。[参考文档](/using_starrocks/External_table.md)

### Improvement

#### StarRocks

* 性能优化：
  * count distinct int语句
  * group by int 语句
  * or语句
* 优化磁盘Balance算法，单机增加磁盘后可以自动进行数据均衡。
* 支持部分列导出。 [参考文档](/unloading/Export.md)
* 优化show processlist，显示具体SQL。
* SET_VAR支持多个变量设置。
* 完善更多报错信息，包括table_sink、routine load、创建物化视图等。

#### StarRocks-Datax Connector

* StarRocks-DataX Writer 支持设置interval flush。

### Bugfix

* 修复动态分区表在数据恢复作业完成后，新分区无法自动创建的问题。 [# 337](https://github.com/StarRocks/starrocks/issues/337)
* 修复CBO开启后row_number函数报错的问题。
* 修复统计信息收集导致fe卡死的问题。
* 修复set_var针对session生效而不是针对语句生效的问题。
* 修复Hive分区外表`select count(*)` 返回异常的问题。
