# StarRocks基本概念及系统架构

## 系统架构图

<img width="540px" height="400px" src="../assets/2.1-1.png"/>

## 组件介绍

StarRocks集群由FE和BE构成, 可以使用MySQL客户端访问StarRocks集群。

### FrontEnd

简称FE，是StarRocks的前端节点，负责管理元数据，管理客户端连接，进行查询规划，查询调度等工作。FE接收MySQL客户端的连接, 解析并执行SQL语句。

* 管理元数据, 执行SQL DDL命令, 用Catalog记录库, 表, 分区, tablet副本等信息。
* FE的SQL layer对用户提交的SQL进行解析, 分析, 改写, 语义分析和关系代数优化, 生产逻辑执行计划。
* FE的Planner负责把逻辑计划转化为可分布式执行的物理计划, 分发给一组BE。
* FE监督BE, 管理BE的上下线, 根据BE的存活和健康状态, 维持tablet的副本的数量。
* FE协调数据导入, 保证数据导入的一致性。
* [FE高可用部署](../loading/Loading_intro.md), 使用复制协议选主和主从同步元数据, 所有的元数据修改操作, 由FE leader节点完成, FE follower节点可执行读操作。 元数据的读写满足顺序一致性。  FE的节点数目采用2n+1, 可容忍n个节点故障。  当FE leader故障时, 从现有的follower节点重新选主, 完成故障切换(TODO介绍迁移至HA文档中，上方链接待更新)。

### BackEnd

简称BE，是StarRocks的后端节点，负责数据存储，计算执行，以及compaction，副本管理等工作。

* BE管理[tablet](#tablet)的副本。
* BE受FE指导, 创建或删除tablet。
* BE接收FE分发的物理执行计划并指定BE coordinator节点, 在BE coordinator的调度下, 与其他BE worker共同协作完成执行。
* BE读本地的列存储引擎获取数据,并通过索引和谓词下沉快速过滤数据。
* BE后台执行compact任务, 减少查询时的读放大。
* 数据导入时, 由FE指定BE coordinator, 将数据以fanout的形式写入到tablet多副本所在的BE上。

### Broker

StarRocks和 HDFS 对象存储等外部数据对接的中转服务，辅助提供导入导出功能。

* Hdfs Broker:  用于从Hdfs中导入数据到StarRocks集群，详见[数据导入](../loading/Loading_intro.md)章节。

## 其他组件

### StarRocksManager

StarRocksManager是StarRocks企业版提供的管理工具，通过Manager可以可视化的进行StarRocks集群管理、在线查询、故障查询、监控报警、可视化慢查询分析等功能。
