Skip to content
Search or jump to…
Pull requests
Issues
Marketplace
Explore
 
@hellolilyliuyi 
StarRocks
/
docs.zh-cn
Public
Code
Issues
36
Pull requests
42
Actions
Projects
Wiki
Security
Insights
ldap认证新增ad #310
 Open
311709000529 wants to merge 49 commits into StarRocks:main from 311709000529:main
+774 −561 
 Conversation 21
 Commits 49
 Checks 1
 Files changed 14
 Open
ldap认证新增ad
#310

File filter 
 
0 / 14 files viewed
Filter changed files
Beta Give feedback
  17  
administration/Authentication.md
Viewed
@@ -32,6 +32,23 @@ CREATE USER zhangsan IDENTIFIED WITH authentication_ldap_simple
* authentication\_ldap\_simple\_bind\_root\_dn：检索用户时，使用的管理员账号DN。
311709000529 marked this conversation as resolved.
* authentication\_ldap\_simple\_bind\_root\_pwd：检索用户时，使用的管理员账号密码。

例如：

~~~text
--ldap配置为
ldapwhoami -x -D "uid=test,cn=users,cn=account,dc=starrocks,dc=cn" -W ldap://127.0.0.1:389
--那么fe.conf中ldap配置为
authentication_ldap_simple_server_host：127.0.0.1
authentication_ldap_simple_server_port：389
authentication_ldap_simple_bind_base_dn：cn=users,cn=account,dc=starrocks,dc=cn
authentication_ldap_simple_user_search_attr：uid  
authentication_ldap_simple_bind_root_dn：uid=test,cn=users,cn=account,dc=starrocks,dc=cn
authentication_ldap_simple_bind_root_pwd：123456
~~~

LDAP认证需要客户端传递明文密码给StarRocks。三种典型客户端配置明文密码传递的方式如下。

* **mysql 命令行**
  2  
administration/Scale_up_down.md
Viewed
@@ -61,6 +61,6 @@ DROP会立刻删除BE节点，丢失的副本由FE调度补齐；DECOMMISSION先

Drop backend是一个危险操作所以需要二次确认后执行

* `alter system dropp backend "be_host:be_heartbeat_service_port";`
* `alter system drop backend "be_host:be_heartbeat_service_port";`

FE和BE扩容之后的状态，也可以通过查看[集群状态](../administration/Cluster_administration.md#确认集群健康状态)一节中的页面进行查看。
  49  
unloading/Export.md
Viewed
@@ -1,11 +1,15 @@
# 导出总览
# Export

数据导出（Export）是 StarRocks 提供的一种将数据导出并存储到其他介质上的功能。该功能可以将用户指定的表或分区的数据，以**文本**的格式，通过 Broker 进程导出到远端存储上，如 HDFS/阿里云OSS/AWS S3（或者兼容S3协议的对象存储） 等。
## Export 总览

StarRocks 拥有 Export 一种将数据导出并存储到其他介质上的功能。该功能可以将用户指定的表或分区的数据，以文本的格式，通过 Broker 进程导出到远端存储上，如 HDFS/阿里云OSS/AWS S3（或者兼容S3协议的对象存储） 等。

本章介绍StarRocks数据导出的基本原理、使用方式、最佳实践以及注意事项。

## 名词解释

* **FE**：Frontend，StarRocks的前端节点。负责元数据管理和请求接入。
* **BE**：Backend，StarRocks的后端节点。负责查询执行和数据存储。
* **Broker**：StarRocks 可以通过 Broker 进程对远端存储进行文件操作。
* **Tablet**：数据分片。一个表会分成 1 个或多个分区，每个分区会划分成多个数据分片。

@@ -22,11 +26,10 @@
上图描述的处理流程主要包括：

1. 用户提交一个 Export 作业到 FE。
2. FE 的导出调度器会通过两阶段来执行一个导出作业：

    a.  PENDING：FE 生成**一个** ExportPendingTask，向 BE 发送 snapshot 命令，对所有涉及到的 Tablet 做一个快照，并生成**多个**查询计划。
2. PENDING阶段：FE 生成**一个** ExportPendingTask，向 BE 发送 snapshot 命令，对所有涉及到的 Tablet 做一个快照，并生成**多个**查询计划。

    b.  EXPORTING：FE 生成**一个** ExportExportingTask，开始执行**一个个**的查询计划。
3. EXPORTING阶段：FE 生成**一个** ExportExportingTask，BE和Broker会根据FE生成的查询计划配合完成数据导出工作。

### 查询计划拆分

@@ -36,11 +39,11 @@ Export 作业会生成多个查询计划，每个查询计划负责扫描一部

### 查询计划执行

一个查询计划扫描多个分片，将读取的数据以行的形式组织，每 1024 行为一个 batch，调用 Broker 写入到远端存储上。
一个查询计划扫描多个分片，将读取的数据以行的形式组织，**每 1024 行**为  **一个** batch，调用 Broker 写入到远端存储上。

查询计划遇到错误会整体自动重试 3 次。如果一个查询计划重试 3 次依然失败，则整个作业失败。

StarRocks 会首先在指定的远端存储的路径中，建立一个名为 `__starrocks_export_tmp_921d8f80-7c9d-11eb-9342-acde48001122` 的临时目录（其中 `921d8f80-7c9d-11eb-9342-acde48001122` 为作业的 query id）。导出的数据首先会写入这个临时目录。每个查询计划会生成一个文件，文件名示例：
StarRocks 会首先在指定的远端存储的路径中，建立一个名为 `__starrocks_export_tmp_921d8f80-7c9d-11eb-9342-acde48001122` 的临时目录（其中 `921d8f80-7c9d-11eb-9342-acde48001122` 为作业的 query id）。导出的数据首先会写入这个临时目录。导入成功后每个查询计划会生成一个文件，文件名示例：

`lineorder_921d8f80-7c9d-11eb-9342-acde48001122_1_2_0.csv.1615471540361`

@@ -61,13 +64,13 @@ StarRocks 会首先在指定的远端存储的路径中，建立一个名为 `__

#### Broker 参数

导出作业需要借助 Broker 进程访问远端存储，不同的 Broker 需要提供不同的参数，具体请参阅 Broker文档。
导出作业需要借助 Broker 进程访问远端存储，不同的 Broker 需要提供不同的参数，具体请参阅 [Broker文档](../administration/Configuration.md)。

### 使用示例

#### 提交导出作业

数据导出命令的详细语法可以通过 `HELP EXPORT` 查看。导出作业举例如下：
导出作业举例如下：

~~~sql
EXPORT TABLE db1.tbl1 
@@ -83,7 +86,7 @@ PROPERTIES
WITH BROKER "hdfs"
(
    "username" = "user",
    "password" = "passwd",
    "password" = "passwd"
);
~~~

@@ -111,25 +114,27 @@ PROPERTIES如下：
提交作业后，可以通过 `SHOW EXPORT` 命令查询导出作业状态。

~~~sql
SHOW EXPORT WHERE queryid = "921d8f80-7c9d-11eb-9342-acde48001122";
SHOW EXPORT WHERE queryid = "edee47f0-abe1-11ec-b9d1-00163e1e238f";
~~~

结果举例如下：

~~~sql
     JobId: 14008
     State: FINISHED
  Progress: 100%
  TaskInfo: {"partitions":["*"],"mem limit":2147483648,"column separator":",","line delimiter":"\n","tablet num":1,"broker":"hdfs","coord num":1,"db":"default_cluster:db1","tbl":"tbl3",columns:["col1", "col3"]}
      Path: oss://bj-test/export/
CreateTime: 2019-06-25 17:08:24
 StartTime: 2019-06-25 17:08:28
FinishTime: 2019-06-25 17:08:34
   Timeout: 3600
  ErrorMsg: N/A
JobId: 23004
311709000529 marked this conversation as resolved.
QueryId: edee47f0-abe1-11ec-b9d1-00163e1e238f
State: FINISHED
Progress: 100%
TaskInfo: {"partitions":[*],"column separator":",","columns":["test"],"tablet num":96,"broker":"broker_name","coord num":6,"db":"default_cluster:test","tbl":"test","row delimiter":"\n","mem limit":2147483648}
Path: hdfs://127.0.0.1:9002/dir/
CreateTime: 2022-03-25 10:18:58
StartTime: 2022-03-25 10:19:00
FinishTime: 2022-03-25 10:19:09
Timeout: 3600
ErrorMsg: NULL
~~~

* JobId：作业的唯一 ID
* JobId：作业的Job ID
* QueryId：作业的查询ID
* State：作业状态：
  * PENDING：作业待调度
  * EXPORING：数据导出中
  49  
using_starrocks/Array.md
Viewed
@@ -1,18 +1,18 @@
# 数组

## 背景
## 什么是数组

数组，作为数据库的一种扩展类型，在 PG、ClickHouse、Snowflake 等系统中都有相关特性支持，可以广泛的应用于A/B Test对比、用户标签分析、人群画像等场景。StarRocks 当前支持了 多维数组嵌套、数组切片、比较、过滤等特性。
数组，作为数据库的一种扩展类型，在 PG、ClickHouse、Snowflake 等系统中都有相关特性支持，可以广泛的应用于 A/B Test 对比、用户标签分析、人群画像等场景。StarRocks 当前支持了 多维数组嵌套、数组切片、比较、过滤等特性。

下面简要介绍一些是使用方式，更详细的函数语法请查看 `参考手册 > 函数参考 > 数组函数`。
下面简要介绍一些是使用方式，更详细的函数语法请查看 [数组函数](https://docs.starrocks.com/zh-cn/main/sql-reference/sql-functions/array-functions/array_agg)。

<br/>

## 数组使用
## 使用数组

### 数组定义
### 定义数组类型的列

下面是在StarRocks中定义数组列的例子
下面是在 StarRocks 中定义数组列的例子

~~~SQL
-- 一维数组
@@ -45,24 +45,24 @@ distributed by hash(c0) buckets 3;

数组类型有以下限制

* 只能在duplicate table中定义数组列（2.0版本开始支持Primary key和Unique key中使用数组类型）
* 数组列不能作为key列(以后可能支持)
* 数组列不能作为distribution列
* 数组列不能作为partition列
* 只能在 duplicate table 中定义数组列（2.1版本开始支持 Primary key 和 Unique key 中使用数组类型）
* 数组列不能作为 key 列(以后可能支持)
* 数组列不能作为 distribution 列
* 数组列不能作为 partition 列

<br/>

### 在SQL中构造数组
### 使用SELECT语句构造数组

可以在SQL中通过中括号（ "[" 和 "]" ）来构造数组常量，每个数组元素通过逗号(",")分割
可以在 SQL 中通过中括号（ "[" 和 "]" ）来构造数组常量，每个数组元素通过逗号(",")分割

~~~SQL
select [1, 2, 3] as numbers;
select ["apple", "orange", "pear"] as fruit;
select [true, false] as booleans;
~~~

当数组元素具有不同类型时，StarRocks会自动推导出合适的类型(supertype)
当数组元素具有不同类型时，StarRocks 会自动推导出合适的类型(supertype)

~~~SQL
select [1, 1.2] as floats;
@@ -82,7 +82,7 @@ select ARRAY<INT>["12", "100"]; -- 结果是 [12, 100]
select [1, NULL];
~~~

对于空数组，可以使用尖括号显示声明其类型，也可以直接写\[\]，此时StarRocks会根据上下文推断其类型，如果无法推断则会报错。
对于空数组，可以使用尖括号显示声明其类型，也可以直接写\[\]，此时 StarRocks 会根据上下文推断其类型，如果无法推断则会报错。

~~~SQL
select [];
@@ -92,28 +92,33 @@ select array_append([], 10);

<br/>

### 数组导入
### 导入数组类型的数据

目前有三种方式向StarRocks中写入数组值，insert into 适合小规模数据测试。后面两种适合大规模数据导入。
目前有三种方式向 StarRocks 中写入数组值，insert into 适合小规模数据测试。后面两种适合大规模数据导入。

* **INSERT INTO**

  ~~~SQL
  create table t0(c0 INT, c1 ARRAY<INT>)duplicate key(c0);
  create table t0(
    c0 INT,
    c1 ARRAY<INT>
  )
  duplicate key(c0)
  distributed by hash(c0) buckets 3;
  INSERT INTO t0 VALUES(1, [1,2,3]);
  ~~~

* **从ORC/Parquet文件导入**
* **从 ORC/Parquet 文件导入**

  StarRocks 中的数组类型，与ORC/Parquet格式中的list结构相对应，不需要额外指定，具体请参考StarRocks 企业文档中 `broker load` 导入相关章节。当前ORC的list结构可以直接导入，Parquet格式正在开发中。
  StarRocks 中的数组类型，与 ORC/Parquet 格式中的list结构相对应，不需要额外指定，具体请参考 StarRocks 企业文档中 `broker load` 导入相关章节。当前 ORC 的 list 结构可以直接导入，Parquet 格式正在开发中。

* **从CSV文件导入**

  CSV 文件导入数组，默认采用逗号分隔，可以用 stream load / routine load 导入CSV文本文件或 Kafka 中的 CSV 格式数据。
  CSV 文件导入数组，默认采用逗号分隔，可以用 stream load / routine load 导入 CSV 文本文件或 Kafka 中的 CSV 格式数据。

<br/>

### 数组元素访问
### 访问数组中的元素

使用中括号（ "[" 和 "]" ）加下标形式访问数组中某个元素，下标从 `1` 开始

@@ -128,7 +133,7 @@ mysql> select [1,2,3][1];
1 row in set (0.00 sec)
~~~

如果下标为0，或者为负数，**不会报错，会返回NULL**
如果下标为 0，或者为负数，**不会报错，会返回NULL**

~~~Plain Text
mysql> select [1,2,3][0];
  87  
using_starrocks/Colocation_join.md → using_starrocks/Colocate_join.md
Viewed
@@ -1,49 +1,27 @@
# Colocate Join

shuffle join 和 broadcast join 中，参与 join 的两张表的数据行，若满足 join 条件，则需要将它们汇合在一个节点上，完成 join。这两种 join 方式，都无法避免节点间数据网络传输带来额外的延迟和其他开销。 而 colocation join 则可避免数据网络传输开销，核心思想是将同一个 Colocation Group 中表，采用一致的分桶键、一致的副本数量和一致副本放置方式，因此如果 join 列为分桶键，则计算节点只需做本地 join 即可，无须从其他节点获取数据。
## 什么是 Colocate Join

本文档主要介绍 Colocation Join 的原理、实现、使用方式和注意事项。
Colocate Join 功能，属于分布式系统实现 Join 数据分布的策略之一。 能够减少数据分布在多个节点引起的 Join 时的数据移动和网络传输，从而提高查询性能。

<br/>
本文档主要介绍 Colocate Join 的原理、实现、使用方式和注意事项。

## 名词解释

* **Colocation Group（CG）**：一个 CG 中会包含一张及以上的 Table。一个CG内的 Table 有相同的分桶方式和副本放置方式，使用 Colocation Group Schema 描述。
* **Colocation Group Schema（CGS）**： 包含 CG 的分桶键，分桶数以及副本数等信息。

<br/>

## 原理

Colocation Join 功能，是将一组拥有相同 CGS 的 Table 组成一个 CG。并保证这些 Table 对应的分桶副本会落在相同一组BE 节点上。使得当 CG 内的表进行分桶列上的 Join 操作时，可以直接进行本地数据 Join，减少数据在节点间的传输耗时。
## 使用 Colocate Join 的优势

分桶键 hash 值，对分桶数取模得到桶的序号(Bucket Seq)， 假设一个 Table 的分桶数为 8，则共有 \[0, 1, 2, 3, 4, 5, 6, 7\] 8 个分桶（Bucket)，每个 Bucket 内会有一个或多个子表（Tablet)，子表数量取决于表的分区数(Partition)：为单分区表时，一个 Bucket 内仅有一个 Tablet。如果是多分区表，则会有多个Tablet。

为了使得 Table 能够有相同的数据分布，同一 CG 内的 Table 必须保证下列约束：
Colocate Join 使用 CG 管理一组表 ，同一 CG 内的表 CGS 相同，即表对应的分桶副本具有一致的分桶键、副本数量和副本放置方式 。这样可以保证同一CG内，表的数据分布在相同一组 BE 节点上。当 Join 列为分桶键，则计算节点只需做本地 Join 即可，减少数据在节点间的传输耗时，提高查询性能。

1. 同一 CG 内的 Table 的分桶键的类型、数量和顺序完全一致，并且桶数一致，这样才能保证多张表的数据分片能够一一对应地进行分布控制。分桶键，即在建表语句中 `DISTRIBUTED BY HASH(col1, col2, ...)` 中指定一组列。分桶键决定了一张表的数据通过哪些列的值进行 Hash 划分到不同的 Bucket Seq 中。同 CG 的 table 的分桶键的名字可以不相同，分桶列的定义在建表语句中的出现次序可以不一致，但是在 `DISTRIBUTED BY HASH(col1, col2, ...)` 的对应数据类型的顺序要完全一致。
2. 同一个 CG 内所有表的所有分区（Partition）的副本数必须一致。如果不一致，可能出现某一个 Tablet 的某一个副本，在同一个 BE 上没有其他的表分片的副本对应。
3. 同一个 CG 内所有表的分区键，分区数量可以不同。

当创建表时，通过表的 PROPERTIES 的属性 `"colocate_with" = "group_name"` 指定表归属的CG；如果CG不存在，说明该表为CG的第一张表，称之为Parent Table，  Parent Table的数据分布(分桶键的类型、数量和顺序、副本数和分桶数)决定了CGS; 如果CG存在，则检查表的数据分布是否和CGS一致。

<br/>

同一个CG中的所有表的副本放置满足:

1. CG中所有 Table 的 Bucket Seq 和 BE 节点的映射关系和 Parent Table 一致。
2. Parent Table 中所有 Partition 的 Bucket Seq 和 BE 节点的映射关系和第一个 Partition 一致。
3. Parent Table 第一个 Partition 的 Bucket Seq 和 BE 节点的映射关系利用原生的 Round Robin 算法决定。

CG内表的一致的数据分布定义和子表副本映射，能够保证分桶键取值相同的数据行一定在相同BE上，因此当分桶键做join列时，只需本地join即可。

<br/>
因此，Colocation Join ，相对于其他 JOIN，例如 Shuffle Join 和 Broadcast Join ，可以避免数据网络传输开销，提高查询性能。

## 使用方式

### 建表
### 创建方式

建表时，可以在 PROPERTIES 中指定属性 `"colocate_with" = "group_name"`，表示这个表是一个 Colocation Join 表，并且归属于一个指定的 Colocation Group。
建表时，可以在 PROPERTIES 中指定属性 `"colocate_with" = "group_name"`，表示这个表是一个 Colocate Join 表，并且归属于一个指定的 Colocation Group。

示例：

@@ -56,17 +34,29 @@ PROPERTIES(
);
~~~

如果指定的 Group 不存在，则 StarRocks 会自动创建一个只包含当前这张表的 Group。如果 Group 已存在，则 StarRocks 会检查当前表是否满足 Colocation Group Schema。如果满足，则会创建该表，并将该表加入 Group。同时，表会根据已存在的 Group 中的数据分布规则创建分片和副本。
如果指定的 CG 不存在，则 StarRocks 会自动创建一个只包含当前这张表的 CG，当前表被称为该 CG 的 Parent Table 。如果 CG 已存在，则 StarRocks 会检查当前表是否满足 CGS。如果满足，则会创建该表，并将该表加入 Group。同时，表会根据已存在的 Group 中的数据分布规则创建分片和副本。

Group 归属于一个 Database，Group 的名字在一个 Database 内唯一。在内部存储是 Group 的全名为 dbId_groupName，但用户只感知 groupName。

<br/>
分桶键 hash 值，对分桶数取模得到桶的序号(Bucket Seq)， 假设一个 Table 的分桶数为 8，则共有 \[0, 1, 2, 3, 4, 5, 6, 7\] 8 个分桶（Bucket)，每个 Bucket 内会有一个或多个子表（Tablet)，子表数量取决于表的分区数(Partition)：为单分区表时，一个 Bucket 内仅有一个 Tablet。如果是多分区表，则会有多个Tablet。

### 删除
为了使得 Table 能够有相同的数据分布，同一 CG 内的 Table 必须保证下列约束：

当 Group 中最后一张表彻底删除后（彻底删除是指从回收站中删除。通常，一张表通过 `DROP TABLE` 命令删除后，会在回收站默认停留一天的时间后，再删除），该 Group 也会被自动删除。
1. 同一 CG 内的 Table 的分桶键的类型、数量和顺序完全一致，并且桶数一致，这样才能保证多张表的数据分片能够一一对应地进行分布控制。分桶键，即在建表语句中 `DISTRIBUTED BY HASH(col1, col2, ...)` 中指定一组列。分桶键决定了一张表的数据通过哪些列的值进行 Hash 划分到不同的 Bucket Seq 中。同 CG 的 table 的分桶键的名字可以不相同，分桶列的定义在建表语句中的出现次序可以不一致，但是在 `DISTRIBUTED BY HASH(col1, col2, ...)` 的对应数据类型的顺序要完全一致。
2. 同一个 CG 内所有表的所有分区（Partition）的副本数必须一致。如果不一致，可能出现某一个 Tablet 的某一个副本，在同一个 BE 上没有其他的表分片的副本对应。
3. 同一个 CG 内所有表的分区键，分区数量可以不同。

<br/>
同一个CG中的所有表的副本放置满足:

1. CG中所有 Table 的 Bucket Seq 和 BE 节点的映射关系和 Parent Table 一致。
2. Parent Table 中所有 Partition 的 Bucket Seq 和 BE 节点的映射关系和第一个 Partition 一致。
3. Parent Table 第一个 Partition 的 Bucket Seq 和 BE 节点的映射关系利用原生的 Round Robin 算法决定。

CG内表的一致的数据分布定义和子表副本映射，能够保证分桶键取值相同的数据行一定在相同BE上，因此当分桶键做join列时，只需本地join即可。

### 删除方式

当 Group 中最后一张表彻底删除后（彻底删除是指从回收站中删除。通常，一张表通过 `DROP TABLE` 命令删除后，会在回收站默认停留一天的时间后，再删除），该 Group 也会被自动删除。

### 查看 Group 信息

@@ -116,7 +106,6 @@ SHOW PROC '/colocation_group/10005.10008';

> 注意：以上命令需要 ADMIN 权限。暂不支持普通用户查看。
<br/>

### 修改表 Group 属性

@@ -140,21 +129,16 @@ ALTER TABLE tbl SET ("colocate_with" = "");

当对一个具有 Colocation 属性的表进行增加分区（ADD PARTITION）、修改副本数时，StarRocks 会检查修改是否会违反 Colocation Group Schema，如果违反则会拒绝。

<br/>

## Colocation 副本均衡和修复

Colocation 表的副本分布需要遵循 Group 中指定的分布，所以在副本修复和均衡方面和普通分片有所区别。

Group 自身有一个 **Stable** 属性，当 Stable 为 true 时，表示当前 Group 内的表的所有分片没有正在进行变动，Colocation 特性可以正常使用。当 Stable 为 false 时（Unstable），表示当前 Group 内有部分表的分片正在做修复或迁移，此时，相关表的 Colocation Join 将退化为普通 Join。

<br/>

### 副本修复

副本只能存储在指定的 BE 节点上。所以当某个 BE 不可用时（宕机、Decommission 等），需要寻找一个新的 BE 进行替换。StarRocks 会优先寻找负载最低的 BE 进行替换。替换后，该 Bucket 内的所有在旧 BE 上的数据分片都要做修复。迁移过程中，Group 被标记为 **Unstable**。

<br/>

### 副本均衡

@@ -164,8 +148,6 @@ StarRocks 会尽力将 Colocation 表的分片均匀分布在所有 BE 节点上
>
> 注2：当一个 Group 处于 Unstable 状态时，其中的表的 Join 将退化为普通 Join。此时可能会极大降低集群的查询性能。如果不希望系统自动均衡，可以设置 FE 的配置项 disable_colocate_balance 来禁止自动均衡。然后在合适的时间打开即可。（具体参阅 [高级操作](#高级操作) 一节）
<br/>

## 查询

对 Colocation 表的查询方式和普通表一样，用户无需感知 Colocation 属性。如果 Colocation 表所在的 Group 处于 Unstable 状态，将自动退化为普通 Join。以下举例说明。
@@ -308,27 +290,24 @@ DESC SELECT * FROM tbl1 INNER JOIN tbl2 ON (tbl1.k2 = tbl2.k2);

HASH JOIN 节点会显示对应原因：`colocate: false, reason: group is not stable`。同时会有一个 EXCHANGE 节点生成。

<br/>

## 高级操作

### FE 配置项

* **disable_colocate_relocate**

    是否关闭 StarRocks 的自动 Colocation 副本修复。默认为 false，即不关闭。该参数只影响 Colocation 表的副本修复，不影响普通表。

* **disable_colocate_balance**

    是否关闭 StarRocks 的自动 Colocation 副本均衡。默认为 false，即不关闭。该参数只影响 Colocation 表的副本均衡，不影响普通表。

* **disable_colocate_join**
以上参数可以动态修改，设置方式请参阅 `HELP ADMIN SHOW CONFIG;` 和 `HELP ADMIN SET CONFIG;`。

    可以通过改该变量在 session 粒度关闭 colocate join功能。
### Session 变量

以上参数可以动态修改，设置方式请参阅 `HELP ADMIN SHOW CONFIG;` 和 `HELP ADMIN SET CONFIG;`。
* **disable_colocate_join**

<br/>
    可以通过设置该变量在 session 粒度关闭 colocate join 功能。

以上参数可以动态修改，设置方式请参阅《[系统变量](../reference/System_variable.md)》章节。

### HTTP Restful API

@@ -408,7 +387,7 @@ StarRocks 提供了几个和 Colocation Join 有关的 HTTP Restful API，用于

    * 标记为 Unstable

        `DELETE /api/colocate/group_stable?db_id=10005&group_id=10008`
        `POST /api/colocate/group_unstable?db_id=10005&group_id=10008`

        `返回：200`

@@ -426,4 +405,4 @@ StarRocks 提供了几个和 Colocation Join 有关的 HTTP Restful API，用于

    其中 Body 是以嵌套数组表示的 BucketsSequence 以及每个 Bucket 中分片所在 BE 的 id。

    > 注意，使用该命令，可能需要将 FE 的配置 disable_colocate_relocate 和 disable_colocate_balance 设为 true。即关闭系统自动的 Colocation 副本修复和均衡。否则可能在修改后，会被系统自动重置。
    > 注意，使用该命令，可能需要将 FE 的配置 disable_colocate_balance 设为 true。即关闭系统自动的 Colocation 副本修复和均衡。否则可能在修改后，会被系统自动重置。
 220  
using_starrocks/Cost_based_optimizer.md
Viewed
@@ -1,158 +1,162 @@
# CBO优化器

# CBO 优化器
StarRocks 1.16.0 版本推出 CBO 优化器（Cost-based Optimizer ）。StarRocks 1.19 及以上版本，该特性默认开启。CBO 优化器能够基于成本选择最优的执行计划，大幅提升复杂查询的效率和性能。

## 背景介绍
## 什么是CBO优化器

在 1.16.0 版本，StarRocks推出的新优化器，可以针对复杂 Ad-hoc 场景生成更优的执行计划。StarRocks采用cascades技术框架，实现基于成本（Cost-based Optimizer 后面简称CBO）的查询规划框架，新增了更多的统计信息来完善成本估算，也补充了各种全新的查询转换（Transformation）和实现（Implementation）规则，能够在数万级别查询计划空间中快速找到最优计划。
CBO 优化器采用 Cascades 框架，使用多种统计信息来完善成本估算，同时补充逻辑转换（Transformation Rule）和物理实现（Implementation Rule）规则，能够在数万级别执行计划的搜索空间中，选择成本最低的最优执行计划。

## 使用说明
## 采集统计信息

### 查询启用新优化器
CBO 优化器会定期采集的多种统计信息，例如行数，列的平均大小、基数信息、NULL 值数据量、MAX/MIN 值等，这些统计信息存储在`_statistics_.table_statistic_v1`中。

全局粒度开启：
### 采集策略

~~~SQL
set global enable_cbo = true;
~~~
统计信息支持全量或抽样的采集类型，并且支持手动或自动定期的采集方式，可以帮助 CBO 优化器完善成本估算，选择最优执行计划。

Session 粒度开启：
| 采集类型 | 采集方式         | 说明                                                         | 优缺点                                                       |
| -------- | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 全量采集 | 手动和自动定期。 | 对整个表的所有数据，计算统计信息。                           | 优点：全量采集的统计信息准确，能够帮助CBO更准确地评估执行计划。 缺点：全量采集消耗大量系统资源，速度慢。 |
| 抽样采集 | 手动和自动定期。 | 从表的每一个分区（Partition）中均匀抽取N行数据，计算统计信息。 | 优点：抽样采集消耗较少的系统资源，速度快。 缺点：抽样采集的统计信息存在一定误差，可能会影响CBO评估执行计划的准确性。 |

~~~SQL
set enable_cbo = true;
> - 手动采集：手动执行一次采集任务，采集统计信息。
> - 自动定期采集：周期性执行采集任务，采集统计信息。采集间隔时间默认为一天。StarRocks 默认每隔两个小时检查数据是否更新。如果检查到数据更新，且距离上一次采集时间已超过采集间隔时间（默认为一天），则 StarRocks 会重新采集统计信息；反之，则 StarRocks 不重新采集统计信息。
~~~
CBO 优化器开启后，需要使用统计信息，统计信息默认为抽样且自动定期采集。抽样行数为20万行，采集间隔时间为一天。每隔两小时 StarRocks 会检查数据是否更新。您也可以按照业务需要，进行如下设置，调整统计信息的采集类型（全量或抽样）和采集方式（手动或自动定期）。

单个 SQL 粒度开启：
### 全量采集

~~~SQL
SELECT /*+ SET_VAR(enable_cbo = true) */ * from table;
~~~
选择采集方式（手动或自动定期），并执行相关命令。相关参数说明，请参见[参数说明](./Cost_based_optimizer.md/#参数说明)。

> 在1.19版本已经默认打开了CBO。
- 如果为手动采集，您可以执行如下命令。

### 统计信息采集

StarRocks会定时采集统计信息，包括但不限于：行数，平均大小、基数信息、NULL值数据量、MAX/MIN值等等，数据会存储在`_statistics_.table_statistic_v1`中，当前支持抽样和全量两种收集方式：

* 抽样收集：
```SQL
ANALYZE FULL TABLE tbl_name(columnA, columnB, columnC...);
```

    会均匀的从每一个partition中抽取N行数据进行统计信息计算，抽样行数可以通过参数指定。优点在于收集任务消耗的资源少，速度快，缺点在于收集的统计信息不准确，对优化器的帮助有限，默认一般为抽样收集，抽样的行数默认为200000行，采集周期为1天，数据未更新不会重新收集。
- 如果为自动定期采集，您可以执行如下命令，设置采集间隔时间、检查间隔时间。

* 全量收集
> 如果多个自动定期采集任务的采集对象完全一致，则 CBO 仅运行最新创建的自动定期采集任务(即 ID 最大的任务)。
    使用整个表的所有数据计算统计信息。优点在于收集到的统计信息准确，优化器可以更好的评估执行计划，但是缺点也比较明显，收集任务消耗资源非常大，速度慢。全量收集可以使用手动或者定时的方式进行主动触发：
```SQL
-- 定期全量采集所有数据库的统计信息。
CREATE ANALYZE FULL ALL 
PROPERTIES(
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
  * 手动Analyze收集: 通过手动触发Analyze命令收集统计信息
-- 定期全量采集指定数据库下所有表的统计信息。
CREATE ANALYZE FULL DATABASE db_name 
PROPERTIES(
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
  * 定期Analyze调度: 通过Analyze Job定期收集指定的库/表/列的统计信息，默认的Analyze调度一次的周期为2小时
-- 定期全量采集指定表、列的统计信息。
CREATE ANALYZE FULL TABLE tbl_name(columnA, columnB, columnC...) 
PROPERTIES(
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
```

### ANALYZE 相关命令
示例：

#### Show Analyze
```SQL
-- 定期全量采集 tpch 数据库下所有表的统计信息，采集间隔时间为默认，检查间隔时间为默认。
CREATE ANALYZE FULL DATABASE tpch;
```

~~~SQL
-- 展示所有的Analyze Job信息
SHOW ANALYZE;
~~~
### 抽样采集

#### Analyze
选择采集方式（手动或自动定期），并执行相关命令。相关参数说明，请参见[参数说明](./Cost_based_optimizer.md/#参数说明)。

抽样采集
- 如果为手动采集，您可以执行如下命令，设置抽样行数。

~~~SQL
```SQL
ANALYZE TABLE tbl_name(columnA, columnB, columnC...)
PROPERTIES(
    "sample_collect_rows" = "10"
    "sample_collect_rows" = "100000"
);
~~~

全量收集

~~~SQL
ANALYZE FULL TABLE tbl_name(columnA, columnB, columnC...);
```

~~~
- 如果为自动定期采集，您可以执行如下命令，设置抽样行数、采集间隔时间、检查间隔时间。

#### Analyze Job
> 如果多个自动定期采集任务的采集对象完全一致，则 CBO 仅运行最新创建的自动定期采集任务(即 ID 最大的任务)。
可以通过Analyze Job创建一个指定数据库/表/列的统计任务，每个任务有自己的执行周期以及配置，会常驻执行。
当有多个Job中指定了收集同一个列时，会按照最新(job id最大)的Job中指定的配置执行。

抽样收集

~~~SQL
-- 定期抽样采集所有数据库的统计信息
CREATE ANALYZE ALL PROPERTIES(...);
-- 定期抽样采集指定数据库下所有表的统计信息
CREATE ANALYZE DATABASE db_name PROPERTIES(...);
-- 定期抽样采集指定表、列的统计信息
CREATE ANALYZE TABLE tbl_name(columnA, columnB, columnC...) PROPERTIES(...);
~~~

全量收集

~~~SQL
-- 定期全量采集所有数据库的统计信息
CREATE ANALYZE FULL ALL PROPERTIES(...);
```SQL
-- 定期抽样采集所有数据库的统计信息。
CREATE ANALYZE ALL
PROPERTIES(
    "sample_collect_rows" = "100000",
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
-- 定期全量采集指定数据库下所有表的统计信息
CREATE ANALYZE FULL DATABASE db_name PROPERTIES(...);
-- 定期抽样采集指定数据库下所有表的统计信息。
CREATE ANALYZE DATABASE db_name
PROPERTIES(
    "sample_collect_rows" = "100000",
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
-- 定期全量采集指定表、列的统计信息
CREATE ANALYZE FULL TABLE tbl_name(columnA, columnB, columnC...) PROPERTIES(...);
~~~
-- 定期抽样采集指定表、列的统计信息。
CREATE ANALYZE TABLE tbl_name(columnA, columnB, columnC...)
PROPERTIES(
    "sample_collect_rows" = "100000",
    "update_interval_sec" = "43200",
    "collect_interval_sec" = "3600"
);
```

删除Job
示例：

~~~SQL
-- 删除Analyze job，id可以通过SHOW ANALYZE获取
DROP ANALYZE <id>;
~~~
```SQL
-- 每隔 43200 秒（12小时）定期抽样采集所有数据库的统计信息，检查间隔时间为默认。
CREATE ANALYZE ALL PROPERTIES("update_interval_sec" = "43200");
示例&说明
-- 定期抽样采集 test 表中 v1 列的统计信息，采集间隔时间为默认，检查间隔时间为默认。
CREATE ANALYZE TABLE test(v1);
```

~~~SQL
-- 每隔100秒定期抽样采集所有数据库的统计信息
CREATE ANALYZE ALL PROPERTIES("update_interval_sec" = "100");
### 查询或删除采集任务

-- 定期全量采集tpch数据库下所有表的统计信息
CREATE ANALYZE FULL DATABASE tpch;
执行如下命令，显示所有采集任务。

-- 定期抽样采集test表中v1列的统计信息
CREATE ANALYZE TABLE test(v1)
~~~
```SQL
SHOW ANALYZE;
```

参数说明：
执行如下命令，删除指定采集任务。

* update_interval_sec：统计任务收集的间隔时间，单位为秒
* sample_collect_rows：抽样的行数
> `ID`为采集任务 ID，可以通过`SHOW ANALYZE`获取。
#### FE 相关配置
```SQL
DROP ANALYZE <ID>;
```

fe.conf中的相关配置项
### 参数说明

~~~conf
# 统计信息收集功能开关
enable_statistic_collect = true;
- `sample_collect_rows`：抽样采集时的抽样行数。
- `update_interval_sec`：自动定期任务的采集间隔时间，默认为 86400（一天），单位为秒。
- `collect_interval_sec`：自动定期任务中，检测数据更新的间隔时间，默认为 7200（两小时），单位为秒。自动定期任务执行时，StarRocks 每隔一段时间会检查表中数据是否更新，如果检查到数据更新，且距离上一次采集时间已超过`update_interval_sec`，则 tarRocks 会重新采集统计信息；反之，则 StarRocks 不重新采集统计信息。

# 统计信息功能执行周期，默认为2小时
statistic_collect_interval_sec = 7200;
## FE配置项

# 统计信息Job的默认收集间隔时间，默认为1天
statistic_update_interval_sec = 86400;
您可以在FE配置文件 fe.conf 中，查询或修改统计信息采集的默认配置。

# 采样统计信息Job的默认采样行数，默认为200000行
statistic_sample_collect_rows = 200000;
~~~
```Plain%20Text
# 是否采集统计信息。
enable_statistic_collect = true
### 新优化器结果验证
# 自动定期任务中，检测数据更新的间隔时间，默认为7200（两小时），单位为秒。
statistic_collect_interval_sec = 7200
StarRocks提供一个新旧优化器**对比**的工具，用于回放fe中的audit.log，可以检查新优化器查询结果是否有误，在使用新优化器前，**建议使用StarRocks提供的对比工具检查一段时间**：
# 自动定期任务的采集间隔时间，默认为86400（一天），单位为秒。
statistic_update_interval_sec = 86400
1. 确认已经修改了FE的统计信息收集配置。
2. 下载测试工具，Oracle JDK版本 [new\_planner\_test.zip](http://starrocks-public.oss-cn-zhangjiakou.aliyuncs.com/new_planner_test.zip)，Open JDK版本 [open\_jdk\_new\_planner\_test.zip](http://starrocks-public.oss-cn-zhangjiakou.aliyuncs.com/open_jdk_new_planner_test.zip) ，然后解压。
3. 按照README配置StarRocks的端口地址，FE的http_port，以及用户名密码。
4. 使用命令`java -jar new_planner_test.jar $fe.audit.log.path`执行测试，测试脚本会执行fe.audit.log 中的查询请求，并进行比对，分析查询结果并记录日志。
5. 执行的结果会记录在result文件夹中，如果在result中包含慢查询，可以将result文件夹打包提交给StarRocks，协助我们修复问题。
# 采样统计信息Job的默认采样行数，默认为200000行。
statistic_sample_collect_rows = 200000
```
 257  
using_starrocks/External_table.md
Viewed
Large diffs are not rendered by default.

  42  
using_starrocks/Lateral_join.md
Viewed
@@ -1,12 +1,12 @@
# Lateral Join

## 背景介绍
## 功能

「行列转化」是ETL处理过程中常见的操作，Lateral 一个特殊的Join关键字，能够按照每行和内部的子查询或者table function关联，通过Lateral 与unnest配合，我们可以实现一行转多行的功能。
「行列转化」是ETL处理过程中常见的操作，Lateral 一个特殊的Join关键字，能够按照每行和内部的子查询或者 table function 关联，通过 Lateral 与 unnest 配合，我们可以实现一行转多行的功能。

## 使用说明

使用lateral join 需要打开新版优化器：
使用 Lateral join 需要打开新版优化器：

~~~SQL
set global enable_cbo = true;
@@ -18,7 +18,7 @@ Lateral 关键字语法说明：
from table_reference join [lateral] table_reference
~~~

Unnest关键字，是一种 table function，可以把数组类型转化成table的多行，配合 Lateral Join 就能实现我们常见的各种行展开逻辑。
Unnest 关键字，是一种 table function，可以把数组类型转化成 table 的多行，配合 Lateral Join 就能实现我们常见的各种行展开逻辑。

~~~SQL
SELECT student, score , t.unnest
@@ -29,14 +29,25 @@ SELECT student, score, t.unnest
FROM tests, UNNEST(scores) AS t ;
~~~

这里第二种写法是第一种的简写，可以使用Unnest 关键字省略 Lateral Join。
这里第二种写法是第一种的简写，可以使用 Unnest 关键字省略 Lateral Join。

## 使用举例
## 注意事项

* 当前版本 Lateral join 仅用于和 Unnest 函数配合使用，实现行转列的功能，后续会支持其他 table function / UDTF。
Member
@jaogoy jaogoy 6 hours ago
有 UDTF 链接的话，可以加上

@hellolilyliuyi	Reply...
* 当前 Lateral join 还不支持子查询。
* 多列 unnest 操作需要指定别名。示例如下：

~~~sql
select v1,t1.unnest as v2,t2.unnest as v3 
from lateral_test , unnest(v2) t1 ,unnest(v3) t2;
~~~

## 示例

当前版本 StarRocks 支持 Bitmap、String、Array、Column 之间的转化关系如下：
![Lateral Join 中一些类型间的转化](../assets/lateral_join_type_convertion.png)

配合Unnest，我们可以实现以下功能：
配合 Unnest，我们可以实现以下功能：

* String 展开成多行

@@ -77,7 +88,7 @@ select v1,unnest from lateral_test2 , unnest(split(v2, ",")) ;
|    2 | 3      |
+------+--------+
---多列unnest时需要指定别名
---多列 unnest 时需要指定别名
select v1,t1.unnest as v2,t2.unnest as v3 from lateral_test2 , unnest(split(v2, ",")) t1,unnest(split(v3, ",")) t2 ;
+------+------+------+
| v1   | v2   | v3   |
@@ -95,7 +106,7 @@ select v1,t1.unnest as v2,t2.unnest as v3 from lateral_test2 , unnest(split(v2,
+------+------+------+
~~~

* Array类型展开成多行
* Array 类型展开成多行

~~~SQL
CREATE TABLE lateral_test (
@@ -138,7 +149,7 @@ select v1,v2,unnest from lateral_test , unnest(v2) ;
+------+------------+--------+
~~~

* Bitmap类型输出
* Bitmap 类型输出

~~~SQL
CREATE TABLE lateral_test3 (
@@ -185,14 +196,3 @@ select v1,unnest from lateral_test3 , unnest(bitmap_to_array(v2)) ;
|    2 |      3 |
+------+--------+
~~~

## 注意事项

* 当前版本 Lateral join 仅用于和Unnest函数配合使用，实现行转列的功能，后续会支持其他table function / UDTF。
* 当前 Lateral join 还不支持子查询。
* 多列unnest操作需要指定别名。示例如下：

~~~sql
select v1,t1.unnest as v2,t2.unnest as v3 
from lateral_test , unnest(v2) t1 ,unnest(v3) t2;
~~~
  191  
using_starrocks/Materialized_view.md
Viewed
@@ -1,64 +1,59 @@

# 物化视图

## 背景介绍
## 什么是物化视图

物化视图的一般定义是：它一种包含一个查询结果的数据库对象，它可以是远端数据的一份本地拷贝，也可以是一个表或一个 join 结果的行/列的一个子集，还可以是使用聚合函数的一个汇总。相对于普通的逻辑视图，将数据「物化」后，能够带来查询性能的提升。
一种包含一个查询结果的数据库对象，它可以是远端数据的一份本地拷贝，也可以是一个表或一个 join 结果的行/列的一个子集，还可以是使用聚合函数的一个汇总。相对于普通的逻辑视图，将数据「物化」后，能够带来查询性能的提升。

> 系统目前还不支持join，更多注意事项请参看 [注意事项](#注意事项)。
> 系统目前还不支持join，更多注意事项请参看 [注意事项](##注意事项)。
在本系统中，物化视图会被更多地用来当做一种预先计算的技术，同RollUp表，预先计算是为了减少查询时现场计算量，从而降低查询延迟。RollUp 表有两种使用方式：对明细表的任意维度组合进行预先聚合；采用新的维度列排序方式，以命中更多的前缀查询条件。当然也可以两种方式一起使用。物化视图的功能是 RollUp 表的超集，原有的 RollUp 功能都可通过物化视图来实现。
在本系统中，物化视图会被更多地用来当做一种预先计算的技术，预先计算是为了减少查询时现场计算量，从而降低查询延迟。

<br/>
### 应用场景

物化视图的使用场景有：
1. 分析需求覆盖明细数据查询以及固定维度聚合查询两方面。

* 分析需求覆盖明细数据查询以及固定维度聚合查询两方面。
* 需要做对排序键前缀之外的其他列组合形式做范围条件过滤。
* 需要对明细表的任意维度做粗粒度聚合分析。
2. 需要做对排序键前缀之外的其他列组合形式做范围条件过滤。

<br/>
3. 需要对明细表的任意维度做粗粒度聚合分析。

## 原理
### 优势

物化视图的数据组织形式和基表、RollUp表相同。用户可以在新建的基表时添加物化视图，也可以对已有表添加物化视图，这种情况下，基表的数据会自动以**异步**方式填充到物化视图中。基表可以拥有多张物化视图，向基表导入数据时，会**同时**更新基表的所有物化视图。数据导入操作具有**原子性**，因此基表和它的物化视图保持数据一致。
1. 提高查询效率

物化视图创建成功后，用户的原有的查询基表的 SQL 语句保持不变，StarRocks 会自动选择一个最优的物化视图，从物化视图中读取数据并计算。用户可以通过 EXPLAIN 命令来检查当前查询是否使用了物化视图。
    物化视图会预先计算从而减少查询时现场计算量，从而降低查询延迟。

<br/>
2. 物化视图实时更新，不需要手动刷写

物化视图中的聚合和查询中聚合的匹配关系：
3. 支持智能路由

| 物化视图聚合 | 查询中聚合 |
| :-: | :-: |
| sum | sum |
| min | min |
| max | max |
| count | count |
| bitmap_union | bitmap_union, bitmap_union_count, count(distinct) |
| hll_union | hll_raw_agg, hll_union_agg, ndv, approx_count_distinct |

其中 bitmap 和 hll 的聚合函数在查询匹配到物化视图后，查询的聚合算子会根据物化视图的表结构进行改写。

<br/>

## 使用方式
    StarRocks 中，查询时不需要显式指定物化视图表名称，StarRocks 会根据查询 SQL 智能路由到最佳的物化视图表。在查询时，物化视图表的选择规则如下：

    - 选择包含所有查询列的物化视图表

    - 按照过滤和排序的 Column 筛选最符合的物化视图表

    - 按照 Join 的 Column 筛选最符合的物化视图表

    - 行数最小的物化视图表

    - 列数最小的物化视图表

## 如何使用物化视图

### 创建物化视图

通过下面命令就可以创建物化视图。创建物化视图是一个异步的操作，也就是说用户成功提交创建任务后，StarRocks 会在后台对存量的数据进行计算，直到创建成功。

**语法**
~~~SQL
CREATE MATERIALIZED VIEW
CREATE MATERIALIZED VIEW MATERIALIZED_VIEW_NAME AS 
(sql_statement);
~~~

具体的语法可以通过下面命令查看：

~~~SQL
HELP CREATE MATERIALIZED VIEW
~~~
**示例**

假设用户有一张销售记录明细表，存储了每个交易的交易id、销售员、售卖门店、销售时间、以及金额。建表语句为：
假设用户有一张销售记录明细表，存储了每个交易的交易 id 、销售员、售卖门店、销售时间、以及金额。建表语句为：

~~~SQL
CREATE TABLE sales_records(
@@ -74,7 +69,7 @@ properties("replication_num" = "1");
表 sales_records 的结构为:

~~~PlainText
MySQL [test]> desc sales_records;
mysql> desc sales_records;
+-----------+--------+------+-------+---------+-------+
| Field     | Type   | Null | Key   | Default | Extra |
@@ -96,26 +91,67 @@ FROM sales_records
GROUP BY store_id;
~~~

更详细物化视图创建语法请参看SQL参考手册 [CREATE MATERIALIZED VIEW](../sql-reference/sql-statements/data-definition/CREATE%20MATERIALIZED%20VIEW.md) ，或者在 MySQL 客户端使用命令 `help create materialized view` 获得帮助。
**创建物化视图时支持的函数**

当前的物化视图只支持对单个表的聚合。目前支持以下聚合函数：

* COUNT
* MAX
* MIN
* SUM
* PERCENTILE_APPROX
* HLL_UNION

    对明细数据进行 HLL 聚合并且在查询时，使用 HLL 函数分析数据。主要适用于快速进行非精确去重计算。对明细数据使用HLL\_UNION聚合，需要先调用hll\_hash函数，对原数据进行转换。

    ~~~SQL
    create materialized view dt_uv as 
        select dt, page_id, HLL_UNION(hll_hash(user_id)) 
        from user_view
        group by dt, page_id;
    select ndv(user_id) from user_view; 查询可命中该物化视图
    ~~~

    目前不支持对 DECIMAL/BITMAP/HLL/PERCENTILE 类型的列使用HLL\_UNION聚合算子。

* BITMAP_UNION

    对明细数据进行 BITMAP 聚合并且在查询时，使用 BITMAP 函数分析数据，主要适用于快速计算 count(distinct) 的精确去重。对明细数据使用 BITMAP\_UNION 聚合，需要先调用 to\_bitmap 函数，对原数据进行转换。

    ~~~SQL
    create materialized view dt_uv  as
        select dt, page_id, bitmap_union(to_bitmap(user_id))
        from user_view
        group by dt, page_id;
    select count(distinct user_id) from user_view; 查询可命中该物化视图
    ~~~

    目前仅支持 TINYINT/SMALLINT/INT/BITINT 类型，且存储内容需为正整数（包括0）。


**使用限制**

  * Base 表中的分区列，必须存在于创建物化视图的 Group by 聚合列中
    >列如：Base 表按天分区，物化视图则只能按天分区列做 Group by 聚合。不能够建立按月粒度 Group by 的物化视图。
  * 目前只支持对单表进行构建物化视图，不支持多表 JOIN
  * 聚合类型表（Aggregation)，不支持对key列执行聚合算子操作，仅支持对 Value 列进行聚合，且聚合算子类型不能改变。
  * 物化视图中至少包含一个 KEY 列
  * 不支持表达式计算
  * 不支持指定物化视图查询
  * 不支持 Order By
更详细物化视图创建语法请参看 SQL 参考手册 [CREATE MATERIALIZED VIEW](../sql-reference/sql-statements/data-definition/CREATE%20MATERIALIZED%20VIEW.md) 。

<br/>

### 查看物化视图
### 查看物化视图构建状态

由于创建物化视图是一个异步的操作，用户在提交完创建物化视图任务后，需要通过命令检查物化视图是否构建完成，命令如下:

~~~SQL
SHOW ALTER MATERIALIZED VIEW FROM db_name;
~~~

或

~~~SQL
SHOW ALTER TABLE ROLLUP FROM db_name;
~~~

> db_name：替换成真实的 db name，比如"test"。
查询结果为:

~~~PlainText
@@ -130,6 +166,10 @@ SHOW ALTER TABLE ROLLUP FROM db_name;

查看物化视图的表结果，需用通过基表名进行：

~~~SQL
DESC table_name all;
~~~

~~~PlainText
mysql> desc sales_records all;
@@ -147,6 +187,21 @@ mysql> desc sales_records all;
+---------------+---------------+-----------+--------+------+-------+---------+-------+
~~~

查看该 Database 下的所有物化视图

~~~SQL
SHOW MATERIALIZED VIEW [IN|FROM db_name];
~~~

~~~PlainText
mysql> SHOW MATERIALIZED VIEW IN test;
+-------+-----------+----------------------+--------------------------------------------------------------------------------------------------+------+
| id    | name      | database_name        | text                                                                                             | rows |
+-------+-----------+----------------------+--------------------------------------------------------------------------------------------------+------+
| 28931 | store_amt | default_cluster:test | CREATE MATERIALIZED VIEW store_amt ASSELECT event_day, SUM(pv)FROM site_accessGROUP BY event_day | 0    |
+-------+-----------+----------------------+--------------------------------------------------------------------------------------------------+------+
~~~

<br/>

### 查询命中物化视图
@@ -159,6 +214,21 @@ mysql> desc sales_records all;
SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
~~~

物化视图中的聚合和查询中聚合的匹配关系：

| 物化视图聚合 | 查询中聚合 |
| :-: | :-: |
| sum | sum |
| min | min |
| max | max |
| count | count |
| bitmap_union | bitmap_union, bitmap_union_count, count(distinct) |
| hll_union | hll_raw_agg, hll_union_agg, ndv, approx_count_distinct |

其中 bitmap 和 hll 的聚合函数在查询匹配到物化视图后，查询的聚合算子会根据物化视图的表结构进行改写。

<br/>

使用 EXPLAIN 命令查询物化视图是否命中：

~~~SQL
@@ -241,9 +311,9 @@ EXPLAIN SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
DROP MATERIALIZED VIEW IF EXISTS store_amt on sales_records;
~~~

删除处于创建中的物化视图，需要先取消异步任务，然后再删除物化视图，以表 `db0.table0` 上的物化视图 mv 为例:
删除处于创建中的物化视图，需要先取消异步任务，然后再删除物化视图。

首先获得JobId，执行命令:
首先获得 JobId，执行命令:

~~~SQL
show alter table rollup from db0;
@@ -258,10 +328,16 @@ show alter table rollup from db0;
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+-------------+------+----------+---------+
~~~

其中JobId为22478，取消该Job，执行命令:
取消正在创建的物化视图

~~~SQL
CANCEL ALTER MATERIALIZED VIEW FROM db_name.view_name;
~~~

上述查询得到 JobId 为 22478，若要取消该 Job，则执行命令:

~~~SQL
cancel alter table rollup from db0.table0 (22478);
CANCEL ALTER MATERIALIZED VIEW FROM db0.table0 (22478);
~~~

<br/>
@@ -338,8 +414,19 @@ ORDER BY k3;
## 注意事项

1. 物化视图的聚合函数的参数仅支持单列，比如：`sum(a+b)` 不支持。

2. 如果删除语句的条件列，在物化视图中不存在，则不能进行删除操作。如果一定要删除数据，则需要先将物化视图删除，然后方可删除数据。
3. 单表上过多的物化视图会影响导入的效率：导入数据时，物化视图和 base 表数据是同步更新的，如果一张表的物化视图表超过10张，则有可能导致导入速度很慢。这就像单次导入需要同时导入10张表数据是一样的。

3. 单表上过多的物化视图会影响导入的效率：导入数据时，物化视图和 Base 表数据是同步更新的，如果一张表的物化视图表超过10张，则有可能导致导入速度很慢。这就像单次导入需要同时导入 10 张表数据是一样的。

4. 相同列，不同聚合函数，不能同时出现在一张物化视图中，比如：`select sum(a), min(a) from table` 不支持。

5. 物化视图的创建语句目前不支持 JOIN 和 WHERE ，也不支持 GROUP BY 的 HAVING 子句。

6. 不能同时创建多个物化视图，只能等待上一个物化视图创建完成，才能创建下一个物化视图。

7. 物化视图表的模型必须和 Base 表保持一致（聚合表的物化视图表是聚合模型，明细表的物化视图表是明细模型）。

8. Delete 操作时，如果 Where 条件中的某个 Key 列在某个 物化视图表中不存在，则不允许进行 Delete。这种情况下，可以先删除物化视图，再进行 Delete 操作，最后再重新增加物化视图。

9. 如果 物化视图中包含 REPLACE 聚合类型的列，则该物化视图必须包含所有 Key 列。
  73  
using_starrocks/Using_HLL.md
Viewed
@@ -1,34 +1,33 @@
<!-- markdownlint-disable MD033 -->
# 背景介绍
Member
@hellolilyliuyi hellolilyliuyi Pending
标题应该是 HyperLogLog

@hellolilyliuyi	Reply...

在现实场景中，随着数据量的增大， 对数据进行去重分析的压力会越来越大。当数据的规模大到一定程度的时候，进行精确去重的成本会比较高。此时用户通常会采用近似算法来降低计算压力。本节将要介绍的 HyperLogLog（简称 HLL）是一种近似去重算法，它的特点是具有非常优异的空间复杂度O(mloglogn)，  时间复杂度为O(n)， 并且计算结果的误差可控制在1%—10%左右，误差与数据集大小以及所采用的哈希函数有关。
在现实场景中，随着数据量的增大，对数据进行去重分析的压力会越来越大。当数据的规模大到一定程度的时候，进行精确去重的成本会比较高。此时用户通常会采用近似算法来降低计算压力。本节将要介绍的 HyperLogLog（简称 HLL）是一种近似去重算法，它的特点是具有非常优异的空间复杂度 O(mloglogn)，时间复杂度为 O(n)，并且计算结果的误差可控制在1%—10%左右，误差与数据集大小以及所采用的哈希函数有关。

## 什么是HyperLogLog
## 什么是 HyperLogLog

HyperLogLog是一种近似的去重算法，能够使用极少的存储空间计算一个数据集的不重复元素的个数。**HLL类型**是基于HyperLogLog算法的工程实现。用于保存HyperLogLog计算过程的中间结果，它只能作为数据表的指标列类型。
HyperLogLog 是一种近似的去重算法，能够使用极少的存储空间计算一个数据集的不重复元素的个数。**HLL类型**是基于 HyperLogLog 算法的工程实现。用于保存 HyperLogLog 计算过程的中间结果，它只能作为数据表的指标列类型。

由于HLL 算法的原理涉及到比较多的数学知识，在这里我们仅通过一个实际例子来说明。假设我们设计随机试验A:  做抛币的独立重复试验， 直到首次出现正面; 记录首次出现正面的抛币次数为随机变量X，则:
由于HLL 算法的原理涉及到比较多的数学知识，在这里我们仅通过一个实际例子来说明。假设我们设计随机试验A：做抛币的独立重复试验，直到首次出现正面; 记录首次出现正面的抛币次数为随机变量 X，则:

* X=1， 概率P(X=1)=1/2
* X=2， 概率P(X=2)=1/4
* X=1，概率P(X=1)=1/2
* X=2，概率P(X=2)=1/4
* ...
* X=n， 概率P(X=n)=(1/2)<sup>n</sup>
* X=n，概率P(X=n)=(1/2)<sup>n</sup>

我们用试验A构造随机试验B: 做N次独立重复试验A， 产生N个独立同分布的随机变量X<sub>1</sub>, X<sub>2</sub>, X<sub>3</sub>, ..., X<sub>N</sub>; 取这组随机变量的最大值为X<sub>max</sub>。结合极大似然估算的方法，N的估算值为2<sup>X<sub>max</sub></sup>。
我们用试验A构造随机试验B: 做N次独立重复试验 A，产生 N 个独立同分布的随机变量 X<sub>1</sub>, X<sub>2</sub>, X<sub>3</sub>, ..., X<sub>N</sub>; 取这组随机变量的最大值为 X<sub>max</sub>。结合极大似然估算的方法，N的估算值为 2<sup>X<sub>max</sub></sup>。
<br/>

现在， 我们在给定的数据集上， 使用哈希函数模拟上述试验:
现在，我们在给定的数据集上，使用哈希函数模拟上述试验:

* 试验A:  对数据集的元素计算哈希值， 记录哈希值的二进制表示形式中， 从最低位算起， 记录bit 1首次出现的位置。
* 试验B:  遍历数据集， 对数据集的元素做试验A的处理， 每次试验时， 更新bit 1首次出现的最大位置m；
* 估算数据集中不重复元素的个数为m<sup>2</sup>。
* 试验 A：对数据集的元素计算哈希值，记录哈希值的二进制表示形式中，从最低位算起，记录 bit 1 首次出现的位置。
* 试验 B：遍历数据集，对数据集的元素做试验 A 的处理，每次试验时，更新 bit 1 首次出现的最大位置 m；
* 估算数据集中不重复元素的个数为m <sup>2</sup>。

事实上，HLL算法根据元素哈希值的低k位，将元素划分到K=2<sup>k</sup>个桶中，统计桶内元素的第k+1位起bit 1首次出现位置的最大值m<sub>1</sub>, m<sub>2</sub>,..., m<sub>k</sub>, 估算桶内不重复元素元素的个数2<sup>m<sub>1</sub></sup>, 2<sup>m<sub>2</sub></sup>,..., 2<sup>m<sub>k</sub></sup>, 数据集的不重复元素个数为桶的数量乘以桶内不重复元素个数的调和平均数: N = K(K/(2<sup>\-m<sub>1</sub></sup>+2<sup>\-m<sub>2</sub></sup>,..., 2<sup>\-m<sub>K</sub></sup>))。
事实上，HLL 算法根据元素哈希值的低 k 位，将元素划分到 K=2<sup>k</sup> 个桶中，统计桶内元素的第 k+1 位起 bit 1 首次出现位置的最大值 m<sub>1</sub>, m<sub>2</sub>,..., m<sub>k</sub>, 估算桶内不重复元素元素的个数 2<sup>m<sub>1</sub></sup>, 2<sup>m<sub>2</sub></sup>,..., 2<sup>m<sub>k</sub></sup>, 数据集的不重复元素个数为桶的数量乘以桶内不重复元素个数的调和平均数: N = K(K/(2<sup>\-m<sub>1</sub></sup>+2<sup>\-m<sub>2</sub></sup>,..., 2<sup>\-m<sub>K</sub></sup>))。
<br/>

HLL为了使结果更加精确，用修正因子和估算结果相乘， 得出最终结果。
HLL 为了使结果更加精确，用修正因子和估算结果相乘，得出最终结果。

为了方面读者的理解， 我们参考文章[https://gist.github.com/avibryant/8275649,](https://gist.github.com/avibryant/8275649) 用StarRocks的SQL语句实现HLL去重算法:
为了方面读者的理解，我们参考文章[https://gist.github.com/avibryant/8275649,](https://gist.github.com/avibryant/8275649) 用 StarRocks 的 SQL 语句实现 HLL 去重算法:

~~~sql
SELECT floor((0.721 * 1024 * 1024) / (sum(pow(2, m * -1)) + 1024 - count(*))) AS estimate
@@ -38,30 +37,30 @@ FROM(select(murmur_hash3_32(c2) & 1023) AS bucket,
     GROUP BY bucket) bucket_values;
~~~

该算法对db0.table0的col2进行去重分析。
该算法对 db0.table0 的 col2 进行去重分析。

* 使用哈希函数murmur_hash3_32对col2计算hash值为32有符号整数；
* 采用1024个桶， 此时修正因子为0.721， 取hash值低10bit为桶的下标；
* 忽略hash值的符号位， 从次高位开始向低位查找， 确定bit 1首次出现的位置；
* 把算出的hash值， 按bucket分组， 桶内使用MAX聚合求bit 1的首次出现的最大位置；
* 分组聚合的结果作为子查询， 最后求所有桶的估算值的调和平均数， 乘以桶数和修正因子。
* 注意空桶计数为1。
* 使用哈希函数 murmur_hash3_32 对 col2 计算 hash 值为 32 有符号整数；
* 采用 1024 个桶，此时修正因子为 0.721，取 hash 值低 10bit 为桶的下标；
* 忽略 hash 值的符号位，从次高位开始向低位查找，确定 bit 1 首次出现的位置；
* 把算出的hash值，按 bucket 分组，桶内使用 MAX 聚合求 bit 1 的首次出现的最大位置；
* 分组聚合的结果作为子查询，最后求所有桶的估算值的调和平均数，乘以桶数和修正因子。
* 注意空桶计数为 1。

上述算法在数据规模较大时， 误差很低。
上述算法在数据规模较大时，误差很低。

这就是 HLL 算法的核心思想。有兴趣的同学可以参考[HyperLogLog论文](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)。

### 如何使用HyperLogLog

1. 使用HyperLogLog去重， 需要在建表语句中， 将目标的指标列的类型设置为HLL，聚合函数设置为HLL_UNION。
2. 目前, 只有聚合表支持HLL类型的指标列。
3. 当在HLL类型列上使用count distinct时，StarRocks会自动转化为HLL_UNION_AGG计算。
1. 使用 HyperLogLog 去重，需要在建表语句中，将目标的指标列的类型设置为 HLL，聚合函数设置为 HLL_UNION。
2. 目前, 只有聚合表支持 HLL 类型的指标列。
3. 当在 HLL 类型列上使用 count distinct 时，StarRocks 会自动转化为 HLL_UNION_AGG 计算。

具体函数语法参见 [HLL_UNION_AGG](../sql-reference/sql-functions/aggregate-functions/hll_union_agg.md)。

#### 示例

首先，创建一张含有**HLL**列的表，其中uv列为聚合列，列类型为HLL，聚合函数为HLL_UNION
首先，创建一张含有**HLL**列的表，其中 uv 列为聚合列，列类型为 HLL，聚合函数为 HLL_UNION

~~~sql
CREATE TABLE test(
@@ -72,9 +71,9 @@ CREATE TABLE test(
DISTRIBUTED BY HASH(ID) BUCKETS 32;
~~~

> * 注：当数据量很大时，最好为高频率的 HLL 查询建立对应的 rollup 表
> * 注：当数据量很大时，最好为高频率的 HLL 查询建立对应的 物化视图
导入数据，Stream Load模式:
导入数据，Stream Load 模式:

~~~bash
curl --location-trusted -u root: -H "label:label_1600997542287" \
@@ -99,7 +98,7 @@ curl --location-trusted -u root: -H "label:label_1600997542287" \
}
~~~

Broker Load模式:
Broker Load 模式:

~~~sql
LOAD LABEL test_db.label
@@ -116,25 +115,25 @@ LOAD LABEL test_db.label

查询数据

* HLL列不允许直接查询它的原始值，可以用函数HLL_UNION_AGG进行查询
* 求总uv
* HLL 列不允许直接查询它的原始值，可以用函数 HLL_UNION_AGG 进行查询
* 求总 uv

`SELECT HLL_UNION_AGG(uv) FROM test;`

该语句等价于

`SELECT COUNT(DISTINCT uv) FROM test;`

* 求每一天的uv
* 求每一天的 uv

`SELECT COUNT(DISTINCT uv) FROM test GROUP BY dt;`

### 注意事项

Bitmap和HLL应该如何选择？如果数据集的基数在百万、千万量级，并拥有几十台机器，那么直接使用 count distinct 即可。如果基数在亿级以上，并且需要精确去重，那么只能用Bitmap类型；如果可以接受近似去重，那么还可以使用HLL类型。
Bitmap 和 HLL 应该如何选择？如果数据集的基数在百万、千万量级，并拥有几十台机器，那么直接使用 count distinct 即可。如果基数在亿级以上，并且需要精确去重，那么只能用 Bitmap 类型；如果可以接受近似去重，那么还可以使用 HLL 类型。

Bitmap只支持TINYINT， SMALLINT， INT， BIGINT， (注意不支持LARGEINT)， 对其他类型数据集去重， 则需要构建词典， 将原类型映射到整数类型.  词典构建比较复杂， 需要权衡数据量， 更新频率， 查询效率， 存储等一系列问题. HLL则没有必要构建词典， 只需要对应的数据类型支持哈希函数即可， 甚至在没有内部支持HLL的分析系统中， 依然可以使用系统提供的hash，用SQL实现HLL去重。
Bitmap 只支持 TINYINT，SMALLINT，INT，BIGINT，(注意不支持 LARGEINT )，对其他类型数据集去重，则需要构建词典，将原类型映射到整数类型。词典构建比较复杂，需要权衡数据量，更新频率，查询效率，存储等一系列问题. HLL 则没有必要构建词典，只需要对应的数据类型支持哈希函数即可，甚至在没有内部支持 HLL 的分析系统中，依然可以使用系统提供的 hash，用 SQL 实现 HLL 去重。

对于普通列，用户还可以使用NDV函数进行近似去重计算。NDV函数返回值是COUNT(DISTINCT col) 结果的近似值聚合函数，底层实现将数据存储类型转为HyperLogLog类型进行计算。NDV函数在计算的时候比较消耗资源，不太适合并发高的场景。
对于普通列，用户还可以使用 NDV 函数进行近似去重计算。NDV函数返回值是COUNT(DISTINCT col) 结果的近似值聚合函数，底层实现将数据存储类型转为 HyperLogLog 类型进行计算。NDV 函数在计算的时候比较消耗资源，不太适合并发高的场景。

如果您希望进行用户行为分析，可以考虑 IntersectCount 或者自定义 UDAF。
  124  
using_starrocks/Using_bitmap.md
Viewed
@@ -1,69 +1,35 @@
# 背景介绍
# 使用Bitmap去重

用户在使用StarRocks进行精确去重分析时，通常会有两种方式：
假如给定一个数组 A， 其取值范围为 [0, n)(注: 不包括n)， 对该数组去重， 可采用 (n+7)/8 的字节长度的 bitmap， 初始化为全 0；逐个处理数组 A 的元素， 以 A 中元素取值作为 bitmap 的下标， 将该下标的 bit 置 1； 最后统计 bitmap 中 1 的个数即为数组 A 的 count distinct 结果。

* 基于明细去重：即传统的count distinct 方式，好处是可以保留明细数据。提高了分析的灵活性。缺点则是需要消耗极大的计算和存储资源，对大规模数据集和查询延迟敏感的去重场景支持不够友好。
* 基于预计算去重：这种方式也是StarRocks推荐的方式。在某些场景中，用户可能不关心明细数据，仅仅希望知道去重后的结果。这种场景可采用预计算的方式进行去重分析，本质上是利用空间换时间，也是MOLAP聚合模型的核心思路。就是将计算提前到数据导入的过程中，减少存储成本和查询时的现场计算成本。并且可以使用RollUp表降维的方式，进一步减少现场计算的数据集大小。
<br>

## 传统Count distinct计算

StarRocks 是基于MPP 架构实现的，在使用count distinct做精准去重时，可以保留明细数据，灵活性较高。但是，由于在查询执行的过程中需要进行多次数据shuffle（不同节点间传输数据，计算去重），会导致性能随着数据量增大而直线下降。

如以下场景。存在表（dt, page, user_id)，需要通过明细数据计算UV。

|  dt   |   page  | user_id |
| :---: | :---: | :---:|
|   20191206  |   waimai  | 101 |
|   20191206  |   waimai  | 102 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   waimai  | 101 |

按page进行分组统计uv

|  page   |   uv  |
| :---: | :---: |
|   waimai  |   2  |
|   xiaoxiang  |  1   |
与传统的使用 [count distinct](#传统Count_distinct计算) 方式相比节约了大量的计算资源。

```sql
select page, count(distinct user_id) as uv from table group by page;
```

对于上图计算 UV 的 SQL，StarRocks 在计算时，会按照下图进行计算，先根据 page 列和 user_id 列 group by，最后再 count。

![alter](../assets/6.1.2-2.png)
# 使用 Bitmap 去重的优势

> 注：图中是 6 行数据在 2 个 BE 节点上计算的示意图
显然，上面的计算方式，由于数据需要进行多次shuffle，当数据量越来越大时，所需的计算资源就会越来越多，查询也会越来越慢。使用Bitmap技术，就是为了解决传统count distinct在大量数据场景下的性能问题。

## 使用bitmap去重

假如给定一个数组A， 其取值范围为[0, n)(注: 不包括n)， 对该数组去重， 可采用(n+7)/8的字节长度的bitmap， 初始化为全0；逐个处理数组A的元素， 以A中元素取值作为bitmap的下标， 将该下标的bit置1； 最后统计bitmap中1的个数即为数组A的count distinct结果。
1. 空间优势:  用 bitmap 的一个 bit 位表示对应下标是否存在， 具有极大的空间优势;  比如对 int32 去重， 使用普通 bitmap 所需的存储空间只占传统去重的 1/32。  StarRocks 中的 Bitmap 采用 Roaring Bitmap 的优化实现， 对于稀疏的 bitmap， 存储空间会进一步显著降低。
2. 时间优势:  bitmap 的去重涉及的计算包括对给定下标的 bit 置位， 统计 bitmap 的置位个数， 分别为 O(1) 操作和 O(n) 操作， 并且后者可使用 clz， ctz 等指令高效计算。 此外， bitmap 去重在 MPP 执行引擎中还可以并行加速处理， 每个计算节点各自计算本地子 bitmap，  使用 bitor 操作将这些子 bitmap 合并成最终的 bitmap， bitor 操作比基于 sort 和基于 hash 的去重效率要高， 无条件依赖和数据依赖， 可向量化执行。

## 使用bitmap去重的优势
Roaring Bitmap 实现，细节可以参考：[具体论文和实现](https://github.com/RoaringBitmap/RoaringBitmap)

1. 空间优势:  用bitmap的一个bit位表示对应下标是否存在， 具有极大的空间优势;  比如对int32去重， 使用普通bitmap所需的存储空间只占传统去重的1/32。  StarRocks中的Bitmap采用Roaring Bitmap的优化实现， 对于稀疏的bitmap， 存储空间会进一步显著降低。
2. 时间优势:  bitmap的去重涉及的计算包括对给定下标的bit置位， 统计bitmap的置位个数， 分别为O(1)操作和O(n)操作， 并且后者可使用clz， ctz等指令高效计算。 此外， bitmap去重在MPP执行引擎中还可以并行加速处理， 每个计算节点各自计算本地子bitmap，  使用bitor操作将这些子bitmap合并成最终的bitmap， bitor操作比基于sort和基于hash的去重效率要高， 无条件依赖和数据依赖， 可向量化执行。
# 如何使用 Bitmap

Roaring Bitmap实现，细节可以参考：[具体论文和实现](https://github.com/RoaringBitmap/RoaringBitmap)

## 如何使用Bitmap

1. 首先， 用户需要注意bitmap index和bitmap去重二者都是用bitmap技术， 但引入动机和解决的问题完全不同， 前者用于低基数的枚举型列的等值条件过滤， 后者用于计算一组数据行的指标列的不重复元素的个数。
2. 目前Bitmap列只能存在于用聚合表， 明细表和更新不支持BITMAP列。
3. 创建表时指定指标列的数据类型为BITMAP，  聚合函数为BITMAP_UNION。
4. 当在Bitmap类型列上使用count distinct时，StarRocks会自动转化为BITMAP_UNION_COUNT计算。
1. 首先， 用户需要注意 bitmap index 和 bitmap 去重二者都是用 bitmap 技术， 但引入动机和解决的问题完全不同， 前者用于低基数的枚举型列的等值条件过滤， 后者用于计算一组数据行的指标列的不重复元素的个数。
2. 目前 Bitmap 列只能存在于用聚合表， 明细表,更新以及主键不支持 BITMAP 列。
3. 创建表时指定指标列的数据类型为 BITMAP，  聚合函数为 BITMAP_UNION。
4. 当在 Bitmap 类型列上使用 count distinct 时，StarRocks 会自动转化为 BITMAP_UNION_COUNT 计算。

具体操作函数参见 [Bitmap函数](../sql-reference/sql-functions/bitmap-functions/bitmap_and.md)。

### 示例
## 示例

以统计某一个页面的UV为例：
以统计某一个页面的 UV 为例：

首先，创建一张含有BITMAP列的表，其中visit_users列为聚合列，列类型为BITMAP，聚合函数为BITMAP_UNION
首先，创建一张含有 BITMAP 列的表，其中 visit_users 列为聚合列，列类型为 BITMAP，聚合函数为 BITMAP_UNION

```sql
CREATE TABLE `page_uv` (
@@ -79,7 +45,7 @@ PROPERTIES (
);
```

向表中导入数据，采用insert into语句导入
向表中导入数据，采用 insert into 语句导入

```sql
insert into page_uv values
@@ -90,7 +56,7 @@ insert into page_uv values
(2, '2020-06-23 01:30:30', to_bitmap(23));
```

在以上数据导入后，在 page_id = 1， visit_date = '2020-06-23 01:30:30'的数据行，visit_user字段包含着3个bitmap元素（13，23，33）；在page_id = 1， visit_date = '2020-06-23 02:30:30'的数据行，visit_user字段包含着1个bitmap元素（13）；在page_id = 2， visit_date = '2020-06-23 01:30:30'的数据行，visit_user字段包含着1个bitmap元素（23）。
在以上数据导入后，在 page_id = 1， visit_date = '2020-06-23 01:30:30' 的数据行，visit_user 字段包含着 3 个 bitmap 元素（13，23，33）；在 page_id = 1， visit_date = '2020-06-23 02:30:30' 的数据行，visit_user 字段包含着 1 个 bitmap 元素（13）；在 page_id = 2， visit_date = '2020-06-23 01:30:30' 的数据行，visit_user 字段包含着 1 个 bitmap 元素（23）。

采用本地文件导入

@@ -108,7 +74,7 @@ cat <<<'DONE' | \
DONE
```

统计每个页面的UV
统计每个页面的 UV

```sql
select page_id, count(distinct visit_users) from page_uv group by page_id;
@@ -129,25 +95,59 @@ mysql> select page_id, count(distinct visit_users) from page_uv group by page_id
2 row in set (0.00 sec)
```
## Bitmap全局字典
# Bitmap 全局字典
目前，基于Bitmap类型的去重机制有一个限制，就是Bitmap需要使用整数型类型作为输入。如果用户需要将其他数据类型作为Bitmap的输入，那么用户需要自己构建全局字典，将其他类型数据（如字符串类型）通过全局字典映射成为整数类型。构建全局字典有几种思路：
目前，基于 Bitmap 类型的去重机制有一个限制，就是 Bitmap 需要使用整数型类型作为输入。如果用户需要将其他数据类型作为 Bitmap 的输入，那么用户需要自己构建全局字典，将其他类型数据（如字符串类型）通过全局字典映射成为整数类型。构建全局字典有几种思路：
### 基于Hive表的全局字典
## 基于 Hive 表的全局字典
这种方案中全局字典本身是一张 Hive 表，Hive 表有两个列，一个是原始值，一个是编码的 Int 值。全局字典的生成步骤：
1. 将事实表的字典列去重生成临时表
2. 临时表和全局字典进行left join，悬空的词典项为新value。
3. 对新value进行编码并插入全局字典。
4. 事实表和更新后的全局字典进行left join ， 将词典项替换为ID。
2. 临时表和全局字典进行 left join，悬空的词典项为新 value。
3. 对新 value 进行编码并插入全局字典。
4. 事实表和更新后的全局字典进行 left join ， 将词典项替换为 ID。
采用这种构建全局字典的方式，可以通过 Spark 或者 MR 实现全局字典的更新，和对事实表中 Value 列的替换。相比基于 Trie 树的全局字典，这种方式可以分布式化，还可以实现全局字典复用。
但这种方式构建全局字典有几个点需要注意：原始事实表会被读取多次，而且还有两次 Join，计算全局字典会使用大量额外资源。
### 基于Trie树构建全局字典
## 基于 Trie 树构建全局字典
用户还可以使用 Trie 树自行构建全局字典。Trie 树又叫前缀树或字典树。Trie 树中节点的后代存在共同的前缀，可以利用字符串的公共前缀来减少查询时间，可以最大限度地减少字符串比较，所以很适合用来实现字典编码。但 Trie 树的实现不容易分布式化，在数据量比较大的时候会产生性能瓶颈。
用户还可以使用Trie树自行构建全局字典。Trie 树又叫前缀树或字典树。Trie树中节点的后代存在共同的前缀，可以利用字符串的公共前缀来减少查询时间，可以最大限度地减少字符串比较，所以很适合用来实现字典编码。但Trie树的实现不容易分布式化，在数据量比较大的时候会产生性能瓶颈。
通过构建全局字典，将其他类型的数据转换成为整型数据，就可以利用 Bitmap 对非整型数据列进行精确去重分析了。
# 传统Count_distinct计算
StarRocks 是基于 MPP 架构实现的，在使用 count distinct 做精准去重时，可以保留明细数据，灵活性较高。但是，由于在查询执行的过程中需要进行多次数据 shuffle（不同节点间传输数据，计算去重），会导致性能随着数据量增大而直线下降。
如以下场景。存在表（dt, page, user_id)，需要通过明细数据计算 UV。
|  dt   |   page  | user_id |
| :---: | :---: | :---:|
|   20191206  |   waimai  | 101 |
|   20191206  |   waimai  | 102 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   xiaoxiang  | 101 |
|   20191206  |   waimai  | 101 |
按 page 进行分组统计 uv
|  page   |   uv  |
| :---: | :---: |
|   waimai  |   2  |
|   xiaoxiang  |  1   |
```sql
select page, count(distinct user_id) as uv from table group by page;
```
对于上图计算 UV 的 SQL，StarRocks 在计算时，会按照下图进行计算，先根据 page 列和 user_id 列 group by，最后再 count。
![alter](../assets/6.1.2-2.png)
> 注：图中是 6 行数据在 2 个 BE 节点上计算的示意图
通过构建全局字典，将其他类型的数据转换成为整型数据，就可以利用Bitmap对非整型数据列进行精确去重分析了。
显然，上面的计算方式，由于数据需要进行多次shuffle，当数据量越来越大时，所需的计算资源就会越来越多，查询也会越来越慢。使用[Bitmap技术](#使用Bitmap去重)，就是为了解决传统count distinct在大量数据场景下的性能问题。
  122  
using_starrocks/Window_function.md
Viewed
@@ -1,50 +1,40 @@
# 窗口函数

## 背景介绍
## 功能

窗口函数是一类特殊的内置函数。和聚合函数类似，窗口函数也是对于多个输入行做计算得到一个数据值。不同的是，窗口函数是在一个特定的窗口内对输入数据做处理，而不是按照group by来分组计算。每个窗口内的数据可以用over()从句进行排序和分组。窗口函数会**对结果集的每一行**计算出一个单独的值，而不是每个group by分组计算一个值。这种灵活的方式允许用户在select从句中增加额外的列，给用户提供了更多的机会来对结果集进行重新组织和过滤。窗口函数只能出现在select列表和最外层的order by从句中。在查询过程中，窗口函数会在最后生效，就是说，在执行完join，where和group by等操作之后再执行。窗口函数在金融和科学计算领域经常被使用到，用来分析趋势、计算离群值以及对大量数据进行分桶分析等。
窗口函数是一类特殊的内置函数。和聚合函数类似，窗口函数也是对于多个输入行做计算得到一个数据值。不同的是，窗口函数是在一个特定的窗口内对输入数据做处理，而不是按照 group by 来分组计算。每个窗口内的数据可以用 over() 从句进行排序和分组。窗口函数会**对结果集的每一行**计算出一个单独的值，而不是每个 group by 分组计算一个值。这种灵活的方式允许用户在 select 从句中增加额外的列，给用户提供了更多的机会来对结果集进行重新组织和过滤。窗口函数只能出现在 select 列表和最外层的 order by 从句中。在查询过程中，窗口函数会在最后生效，就是说，在执行完 join，where 和 group by 等操作之后再执行。窗口函数在金融和科学计算领域经常被使用到，用来分析趋势、计算离群值以及对大量数据进行分桶分析等。

<br/>

## 使用方式

窗口函数的语法：
## 语法

~~~SQL
function(args) OVER(partition_by_clause order_by_clause [window_clause])
Member
@jaogoy jaogoy 6 hours ago
我们当前 reference 中没有对窗口函数等整体的描述，导致具体的语法放在这里，有点划分得不清晰。
这块你和刘艺他们交流下。

@hellolilyliuyi	Reply...
partition_by_clause ::= PARTITION BY expr [, expr ...]
order_by_clause ::= ORDER BY expr [ASC | DESC] [, expr [ASC | DESC] ...]
~~~

### Function
## 参数说明

目前支持的Function包括：
### partition_by_clause

* MIN(), MAX(), COUNT(), SUM(), AVG()
* FIRST_VALUE(), LAST_VALUE(), LEAD(), LAG()
* ROW_NUMBER(), RANK(), DENSE_RANK()

### PARTITION BY从句
Partition By 从句和 Group By 类似。它把输入行按照指定的一列或多列分组，相同值的行会被分到一组。

Partition By从句和Group By类似。它把输入行按照指定的一列或多列分组，相同值的行会被分到一组。
### order_by_clause

### ORDER BY从句

Order By从句和外层的Order By基本一致。它定义了输入行的排列顺序，如果指定了Partition By，则Order By定义了每个Partition分组内的顺序。与外层Order By的唯一不同点是：OVER从句中的`Order By n`（n是正整数）相当于不做任何操作，而外层的Order By n表示按照第n列排序。
Order By 从句和外层的 Order By 基本一致。它定义了输入行的排列顺序，如果指定了 Partition By，则Order By 定义了每个 Partition 分组内的顺序。与外层 Order By 的唯一不同点是：OVER 从句中的`Order By n`（n是正整数）相当于不做任何操作，而外层的 Order By n 表示按照第 n 列排序。

举例:

这个例子展示了在select列表中增加一个id列，它的值是1，2，3等等，顺序按照events表中的date_and_time列排序。
这个例子展示了在 select 列表中增加一个 id 列，它的值是 1，2，3 等等，顺序按照 events 表中的 date_and_time 列排序。

~~~SQL
SELECT row_number() OVER (ORDER BY date_and_time) AS id,
    c1, c2, c3, c4
FROM events;
~~~

### Window Clause
### window_clause

Window从句用来为窗口函数指定一个运算范围，以当前行为准，前后若干行作为窗口函数运算的对象。Window从句支持的方法有：AVG(), COUNT(), FIRST_VALUE(), LAST_VALUE()和SUM()。对于 MAX()和MIN(), window从句可以指定开始范围UNBOUNDED PRECEDING
Window 从句用来为窗口函数指定一个运算范围，以当前行为准，前后若干行作为窗口函数运算的对象。Window 从句支持的方法有：AVG(), COUNT(), FIRST_VALUE(), LAST_VALUE() 和 SUM()。对于 MAX() 和 MIN(), Window 从句可以指定开始范围 UNBOUNDED PRECEDING

语法：

@@ -54,7 +44,7 @@ ROWS BETWEEN [ { m | UNBOUNDED } PRECEDING | CURRENT ROW] [ AND [CURRENT ROW | {

举例：

假设我们有如下的股票数据，股票代码是JDR，closing price是每天的收盘价。
假设我们有如下的股票数据，股票代码是 JDR，closing price 是每天的收盘价。

~~~SQL
create table stock_ticker (
@@ -85,7 +75,7 @@ order by stock_symbol, closing_date
+--------------+---------------+---------------------+
~~~

这个查询使用窗口函数产生moving_average这一列，它的值是3天的股票均价，即前一天、当前以及后一天三天的均价。第一天没有前一天的值，最后一天没有后一天的值，所以这两行只计算了两天的均值。这里Partition By没有起到作用，因为所有的数据都是JDR的数据，但如果还有其他股票信息，Partition By会保证窗口函数值作用在本Partition之内。
这个查询使用窗口函数产生 moving_average 这一列，它的值是3天的股票均价，即前一天、当前以及后一天三天的均价。第一天没有前一天的值，最后一天没有后一天的值，所以这两行只计算了两天的均值。这里 Partition By 没有起到作用，因为所有的数据都是 JDR 的数据，但如果还有其他股票信息，Partition By 会保证窗口函数值作用在本 Partition 之内。

~~~SQL
select stock_symbol, closing_date, closing_price,
@@ -115,21 +105,25 @@ from stock_ticker;

<br/>

## 函数举例
### 函数（function）

本节介绍StarRocks中可以用作窗口函数的方法。
目前支持的函数包括：

<br/>
* MIN(), MAX(), COUNT(), SUM(), AVG()
* FIRST_VALUE(), LAST_VALUE(), LEAD(), LAG()
* ROW_NUMBER(), RANK(), DENSE_RANK()

## 示例

### AVG()
311709000529 marked this conversation as resolved.

语法：
**语法**

~~~SQL
AVG([DISTINCT | ALL] *expression*) [OVER (*analytic_clause*)]
AVG( expression ) [OVER (*analytic_clause*)]
~~~

举例：
**示例**

计算当前行和它**前后各一行**数据的x平均值

@@ -166,13 +160,13 @@ where property in ('odd','even');

### COUNT()

语法：
**语法**

~~~SQL
COUNT([DISTINCT | ALL] expression) [OVER (analytic_clause)]
COUNT( expression ) [OVER (analytic_clause)]
~~~

举例：
**示例**

计算从**当前行到第一行**x出现的次数。

@@ -208,16 +202,16 @@ from int_t where property in ('odd','even');

### DENSE_RANK()

DENSE_RANK()函数用来表示排名，与RANK()不同的是，DENSE_RANK()**不会出现空缺**数字。比如，如果出现了两个并列的1，DENSE_RANK()的第三个数仍然是2，而RANK()的第三个数是3。
DENSE_RANK() 函数用来表示排名，与 RANK() 不同的是，DENSE_RANK() **不会出现空缺**数字。比如，如果出现了两个并列的 1，DENSE_RANK() 的第三个数仍然是 2，而 RANK() 的第三个数是 3。

语法：
**语法**

~~~SQL
DENSE_RANK() OVER(partition_by_clause order_by_clause)
~~~

举例：
下例展示了按照property列分组对x列排名：
**示例**
下例展示了按照 property 列分组对x列排名：

~~~SQL
select x, y,
@@ -249,15 +243,15 @@ from int_t;

### FIRST_VALUE()

FIRST_VALUE()返回窗口范围内的**第一个**值。
FIRST_VALUE() 返回窗口范围内的**第一个**值。

语法：
**语法**

~~~SQL
FIRST_VALUE(expr) OVER(partition_by_clause order_by_clause [window_clause])
~~~

举例：
**示例**

我们有如下数据

@@ -279,7 +273,7 @@ from mail_merge;
+---------+---------+--------------+
~~~

使用FIRST_VALUE()，根据country分组，返回每个分组中第一个greeting的值：
使用 FIRST_VALUE()，根据 country 分组，返回每个分组中第一个 greeting 的值：

~~~SQL
select country, name,
@@ -308,15 +302,15 @@ from mail_merge;

### LAG()

LAG()方法用来计算当前行**向前**数若干行的值。
LAG() 方法用来计算当前行**向前**数若干行的值。

语法：
**语法**

~~~SQL
LAG (expr, offset, default) OVER (partition_by_clause order_by_clause)
~~~

举例：
**示例**

计算前一天的收盘价

@@ -349,15 +343,15 @@ order by closing_date;

### LAST_VALUE()

LAST_VALUE()返回窗口范围内的**最后一个**值。与FIRST_VALUE()相反。
LAST_VALUE() 返回窗口范围内的**最后一个**值。与 FIRST_VALUE() 相反。

语法：

~~~SQL
LAST_VALUE(expr) OVER(partition_by_clause order_by_clause [window_clause])
~~~

使用FIRST_VALUE()举例中的数据：
使用 FIRST_VALUE() 举例中的数据：

~~~SQL
select country, name,
@@ -386,15 +380,15 @@ from mail_merge;

### LEAD()

LEAD()方法用来计算当前行**向后**数若干行的值。
LEAD() 方法用来计算当前行**向后**数若干行的值。

语法：
**语法**

~~~SQL
LEAD (expr, offset, default]) OVER (partition_by_clause order_by_clause)
~~~

举例：
**示例**

计算第二天的收盘价对比当天收盘价的走势，即第二天收盘价比当天高还是低。

@@ -430,13 +424,13 @@ order by closing_date;

### MAX()

语法：
**语法**

~~~SQL
MAX([DISTINCT | ALL] expression) [OVER (analytic_clause)]
MAX(expression) [OVER (analytic_clause)]
~~~

举例：
**示例**

计算**从第一行到当前行之后一行**的最大值

@@ -469,13 +463,13 @@ where property in ('prime','square');

### MIN()

语法：
**语法**

~~~SQL
MIN([DISTINCT | ALL] expression) [OVER (analytic_clause)]
MIN(expression) [OVER (analytic_clause)]
~~~

举例：
**示例**

计算**从第一行到当前行之后一行**的最小值

@@ -508,15 +502,15 @@ where property in ('prime','square');

### RANK()

RANK()函数用来表示排名，与DENSE_RANK()不同的是，RANK()会**出现空缺**数字。比如，如果出现了两个并列的1，RANK()的第三个数就是3，而不是2。
RANK() 函数用来表示排名，与 DENSE_RANK() 不同的是，RANK() 会**出现空缺**数字。比如，如果出现了两个并列的 1，RANK() 的第三个数就是 3，而不是 2。

语法：
**语法**

~~~SQL
RANK() OVER(partition_by_clause order_by_clause)
~~~

举例：
**示例**

根据x列进行排名

@@ -545,15 +539,15 @@ from int_t;

### ROW_NUMBER()

为每个Partition的每一行返回一个从1开始连续递增的整数。与RANK()和DENSE_RANK()不同的是，ROW_NUMBER()返回的值**不会重复也不会出现空缺**，是**连续递增**的。
为每个 Partition 的每一行返回一个从1开始连续递增的整数。与 RANK() 和 DENSE_RANK() 不同的是，ROW_NUMBER() 返回的值**不会重复也不会出现空缺**，是**连续递增**的。

语法：
**语法**

~~~SQL
ROW_NUMBER() OVER(partition_by_clause order_by_clause)
~~~

举例：
**示例**

~~~SQL
select x, y, row_number() over(partition by x order by y) as rank
@@ -580,15 +574,15 @@ from int_t;

### SUM()

语法：
**语法**

~~~SQL
SUM([DISTINCT | ALL] expression) [OVER (analytic_clause)]
SUM(expression) [OVER (analytic_clause)]
~~~

举例：
**示例**

按照property进行分组，在组内计算**当前行以及前后各一行**的x列的和。
按照 property 进行分组，在组内计算**当前行以及前后各一行**的x列的和。

~~~SQL
select x, property,
  84  
using_starrocks/filemanager.md
Viewed
@@ -2,48 +2,48 @@

StarRocks 中的一些功能需要使用一些用户自定义的文件。比如用于访问外部数据源的公钥、密钥文件、证书文件等等。文件管理器提供这样一个功能，能够让用户预先上传这些文件并保存在 StarRocks 系统中，然后可以在其他命令中引用或访问。

## 基本概念
## 什么是文件管理器

文件是指用户创建并保存在 StarRocks 中的文件。

一个文件由 `数据库名称（database）`、`分类（catalog）` 和 `文件名（file_name）` 共同定位。同时每个文件也有一个全局唯一的 id（file_id），作为系统内的标识。

文件的创建和删除只能由拥有 `admin` 权限的用户进行操作。某个文件归属于某一个的 database，则对该 database 拥有访问权限的用户都可以使用该文件。

## 具体操作
## 如何使用文件管理器

文件管理主要有三个命令：`CREATE FILE`，`SHOW FILE` 和 `DROP FILE`，分别为创建、查看和删除文件。这三个命令的具体语法可以通过连接到 StarRocks 后，执行 `HELP cmd;` 的方式查看帮助。
文件管理主要有三个命令：`CREATE FILE`，`SHOW FILE` 和 `DROP FILE`，分别为创建、查看和删除文件。

1. CREATE FILE
### 创建文件管理器

    该语句用于创建并上传一个文件到 StarRocks 集群。
该语句用于创建并上传一个文件到 StarRocks 集群。

    该功能通常用于管理一些其他命令中需要使用到的文件，如证书、公钥私钥等等。
该功能通常用于管理一些其他命令中需要使用到的文件，如证书、公钥私钥等等。

    单个文件大小限制为 1MB。
    一个 StarRocks 集群最多上传 100 个文件。
单个文件大小限制为 1MB。
一个 StarRocks 集群最多上传 100 个文件。

    语法：
语法：

    ~~~sql
    CREATE FILE "file_name" [IN database]
    [properties]
    ~~~

    说明：
说明：

    * file_name:  自定义文件名。
    * database: 文件归属于某一个 db，如果没有指定，则使用当前 session 的 db。
* file_name:  自定义文件名。
* database: 文件归属于某一个 db，如果没有指定，则使用当前 session 的 db。

    properties 支持以下参数:
properties 支持以下参数:

    * url: 必须。指定一个文件的下载路径。当前仅支持无认证的 http 下载路径。命令执行成功后，文件将被保存在 StarRocks 中，该 url 将不再需要。
    * catalog: 必须。对文件的分类名，可以自定义。但在某些命令中，会查找指定 catalog 中的文件。比如例行导入中的，数据源为 kafka 时，会查找 catalog 名为 kafka 下的文件。
    * md5: 可选。文件的 md5。如果指定，会在下载文件后进行校验。
* url: 必须。指定一个文件的下载路径。当前仅支持无认证的 http 下载路径。命令执行成功后，文件将被保存在 StarRocks 中，该 url 将不再需要。
* catalog: 必须。对文件的分类名，可以自定义。但在某些命令中，会查找指定 catalog 中的文件。比如例行导入中的，数据源为 kafka 时，会查找 catalog 名为 kafka 下的文件。
* md5: 可选。文件的 md5。如果指定，会在下载文件后进行校验。

    Examples:
Examples:

    * 创建文件 ca.pem ，分类为 kafka
* 创建文件 ca.pem ，分类为 kafka

    ~~~sql
    CREATE FILE "ca.pem"
@@ -54,7 +54,7 @@ StarRocks 中的一些功能需要使用一些用户自定义的文件。比如
    );
    ~~~

    * 创建文件 client.key，分类为 my_catalog
* 创建文件 client.key，分类为 my_catalog

    ~~~sql
    CREATE FILE "client.key"
@@ -67,59 +67,59 @@ StarRocks 中的一些功能需要使用一些用户自定义的文件。比如
    );
    ~~~

    文件创建成功后，文件相关的信息将持久化在 StarRocks 中。用户可以通过 `SHOW FILE` 命令查看已经创建成功的文件。
文件创建成功后，文件相关的信息将持久化在 StarRocks 中。

2. SHOW FILE
### 展示文件管理器

    该语句用于展示一个 database 内创建的文件
该语句用于展示一个 database 内创建的文件

    语法：
语法：

    ~~~sql
    SHOW FILE [FROM database];
    ~~~

    说明：
说明：

    * FileId:     文件ID，全局唯一
    * DbName:     所属数据库名称
    * Catalog:    自定义分类
    * FileName:   文件名
    * FileSize:   文件大小，单位字节
    * MD5:        文件的 MD5
 * FileId:     文件ID，全局唯一
 * DbName:     所属数据库名称
 * Catalog:    自定义分类
 * FileName:   文件名
 * FileSize:   文件大小，单位字节
 * MD5:        文件的 MD5

    Examples:
Examples:

    查看数据库 my_database 中已上传的文件
查看数据库 my_database 中已上传的文件

    ~~~sql
    mysql> SHOW FILE FROM test;
    Empty set (0.00 sec)
    ~~~

3. DROP FILE
### 删除文件管理器

    该语句用于删除一个已上传的文件。
该语句用于删除一个已上传的文件。

    语法：
语法：

    ~~~sql
    DROP FILE "file_name" [FROM database]
    [properties]
    ~~~

    说明：
说明：

    * file_name:  文件名。
    * database: 文件归属的某一个 db，如果没有指定，则使用当前 session 的 db。
* file_name:  文件名。
* database: 文件归属的某一个 db，如果没有指定，则使用当前 session 的 db。

    properties 支持以下参数:
properties 支持以下参数:

    * catalog: 必须。文件所属分类。
* catalog: 必须。文件所属分类。

    Examples:
Examples:

    删除文件 ca.pem
删除文件 ca.pem

    ~~~sql
    DROP FILE "ca.pem" properties("catalog" = "kafka");
  18  
using_starrocks/timezone.md
Viewed
@@ -18,17 +18,17 @@ StarRocks 内部存在多个时区相关参数

2. SET time_zone = 'Asia/Shanghai'

    该命令可以设置session级别的时区，连接断开后失效
    该命令可以设置 session 级别的时区，连接断开后失效

3. SET global time_zone = 'Asia/Shanghai'

    该命令可以设置global级别的时区参数，fe会将参数持久化，连接断开后不失效
    该命令可以设置 global 级别的时区参数，fe会将参数持久化，连接断开后不失效

## 时区的影响

时区设置会影响对时区敏感的时间值的显示和存储。

包括NOW()或CURTIME()等时间函数显示的值，也包括show load, show backends中的时间值。
包括 NOW() 或 CURTIME() 等时间函数显示的值，也包括 show load, show backends 中的时间值。

但不会影响 create table 中时间类型分区列的 less than 值，也不会影响存储为 date/datetime 类型的值的显示。

@@ -44,19 +44,19 @@ StarRocks 内部存在多个时区相关参数

时区值可以使用几种格式给出，不区分大小写:

* 表示UTC偏移量的字符串，如'+10:00'或'-6:00'
* 表示UTC偏移量的字符串，如 '+10:00' 或 '-6:00'

* 标准时区格式，如"Asia/Shanghai"、"America/Los_Angeles"
* 标准时区格式，如 "Asia/Shanghai" 、 "America/Los_Angeles"

* 不支持缩写时区格式，如"MET"、"CTT"。因为缩写时区在不同场景下存在歧义。
* 不支持缩写时区格式，如 "MET" 、 "CTT" 。因为缩写时区在不同场景下存在歧义。

* 为了兼容StarRocks，支持CST缩写时区，内部会将CST转移为"Asia/Shanghai"的中国标准时区
* 为了兼容 StarRocks ，支持 CST 缩写时区，内部会将 CST 转移为 "Asia/Shanghai" 的中国标准时区

## 默认时区

系统default timezone为"Asia/Shanghai"，当导入时，如果服务器时区为其他时区，需要指定相应时区，否则日期字段会不一致。
系统 default timezone 为 "Asia/Shanghai"，当导入时，如果服务器时区为其他时区，需要指定相应时区，否则日期字段会不一致。

例如系统时区为UTC时，未指定情况下导入结果的日期字段会出现+8h的异常结果，需要在导入的参数部分指定时区，具体参数指定参考对应Load章节的参数说明。
例如系统时区为 UTC 时，未指定情况下导入结果的日期字段会出现 +8h 的异常结果，需要在导入的参数部分指定时区，具体参数指定参考对应 Load 章节的参数说明。

## 时区格式列表

© 2022 GitHub, Inc.
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
Loading complete
