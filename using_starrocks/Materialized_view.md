
# 物化视图

## 物化视图总览

物化视图的一般定义是：一种包含一个查询结果的数据库对象，它可以是远端数据的一份本地拷贝，也可以是一个表或一个 join 结果的行/列的一个子集，还可以是使用聚合函数的一个汇总。相对于普通的逻辑视图，将数据「物化」后，能够带来查询性能的提升。

> 系统目前还不支持join，更多注意事项请参看 [注意事项](#注意事项)。

在本系统中，物化视图会被更多地用来当做一种预先计算的技术，预先计算是为了减少查询时现场计算量，从而降低查询延迟。

## 名词解释

1. Duplicate 数据模型：StarRocks中的用于存放明细数据的数据模型，建表可指定，数据不会被聚合。
2. Base 表：StarRocks 中通过 CREATE TABLE 命令创建出来的表。
3. Materialized Views 表：简称 MVs，物化视图。

## 原理

物化视图的数据组织形式和基表、RollUp 表相同。用户可以在新建的基表时添加物化视图，也可以对已有表添加物化视图，这种情况下，基表的数据会自动以**异步**方式填充到物化视图中。基表可以拥有多张物化视图，向基表导入数据时，会**同时**更新基表的所有物化视图。数据导入操作具有**原子性**，因此基表和它的物化视图保持数据一致。

物化视图创建成功后，用户的原有的查询基表的 SQL 语句保持不变，StarRocks 会自动选择一个最优的物化视图，从物化视图中读取数据并计算。用户可以通过 EXPLAIN 命令来检查当前查询是否使用了物化视图。

<br/>

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

## 适用场景

* 分析需求覆盖明细数据查询以及固定维度聚合查询两方面。
* 需要做对排序键前缀之外的其他列组合形式做范围条件过滤。
* 需要对明细表的任意维度做粗粒度聚合分析。

在实际的业务场景中，通常存在两种场景并存的分析需求：对固定维度的聚合分析 和 对原始明细数据任意维度的分析。

例如，在销售场景中，每条订单数据包含这几个维度信息（item\_id, sold\_time, customer\_id, price）。在这种场景下，有两种分析需求并存：

1. 业务方需要获取某个商品在某天的销售额是多少，那么仅需要在维度（item\_id, sold\_time）维度上对 price 进行聚合即可。
2. 分析某个人在某天对某个商品的购买明细数据。

在现有的 StarRocks 数据模型中，如果仅建立一个聚合模型的表，比如（item\_id, sold\_time, customer\_id, sum(price)）。由于聚合损失了数据的部分信息，无法满足用户对明细数据的分析需求。如果仅建立一个 Duplicate 模型，虽可以满足任意维度的分析需求，但无法快速完成分析。如果同时建立一个聚合模型和一个 Duplicate 模型，虽可以满足性能和任意维度分析，但两表之间本身无关联，需要业务方自行选择分析表。不灵活也不易用。


## 使用方式

### 创建物化视图

通过下面命令就可以创建物化视图。创建物化视图是一个异步的操作，也就是说用户成功提交创建任务后，StarRocks 会在后台对存量的数据进行计算，直到创建成功。

~~~SQL
CREATE MATERIALIZED VIEW
~~~

假设用户有一张销售记录明细表，存储了每个交易的交易 id 、销售员、售卖门店、销售时间、以及金额。建表语句为：

~~~SQL
CREATE TABLE sales_records(
    record_id int,
    seller_id int,
    store_id int,
    sale_date date,
    sale_amt bigint
) distributed BY hash(record_id)
properties("replication_num" = "1");
~~~

表 sales_records 的结构为:

~~~PlainText
mysql> desc sales_records;

+-----------+--------+------+-------+---------+-------+
| Field     | Type   | Null | Key   | Default | Extra |
+-----------+--------+------+-------+---------+-------+
| record_id | INT    | Yes  | true  | NULL    |       |
| seller_id | INT    | Yes  | true  | NULL    |       |
| store_id  | INT    | Yes  | true  | NULL    |       |
| sale_date | DATE   | Yes  | false | NULL    | NONE  |
| sale_amt  | BIGINT | Yes  | false | NULL    | NONE  |
+-----------+--------+------+-------+---------+-------+
~~~

如果用户经常对不同门店的销售量做分析，则可以为 sales_records 表创建一张“以售卖门店为分组，对相同售卖门店的销售额求和”的物化视图。创建语句如下：

~~~sql
CREATE MATERIALIZED VIEW store_amt AS
SELECT store_id, SUM(sale_amt)
FROM sales_records
GROUP BY store_id;
~~~

* **限制：**

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

### 查看物化视图构建状态

由于创建物化视图是一个异步的操作，用户在提交完创建物化视图任务后，需要通过命令检查物化视图是否构建完成，命令如下:

~~~SQL
SHOW ALTER MATERIALIZED VIEW FROM db_name;
~~~

查询结果为:

~~~PlainText
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+----------+------+----------+---------+
| JobId | TableName     | CreateTime          | FinishedTime        | BaseIndexName | RollupIndexName | RollupId | TransactionId | State    | Msg  | Progress | Timeout |
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+----------+------+----------+---------+
| 22324 | sales_records | 2020-09-27 01:02:49 | 2020-09-27 01:03:13 | sales_records | store_amt       | 22325    | 672           | FINISHED |      | NULL     | 86400   |
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+----------+------+----------+---------+
~~~

如果 State 为 FINISHED，说明物化视图已经创建完成。

查看物化视图的表结果，需用通过基表名进行：

~~~SQL
DESC table_name all;
~~~

~~~PlainText
mysql> desc sales_records all;

+---------------+---------------+-----------+--------+------+-------+---------+-------+
| IndexName     | IndexKeysType | Field     | Type   | Null | Key   | Default | Extra |
+---------------+---------------+-----------+--------+------+-------+---------+-------+
| sales_records | DUP_KEYS      | record_id | INT    | Yes  | true  | NULL    |       |
|               |               | seller_id | INT    | Yes  | true  | NULL    |       |
|               |               | store_id  | INT    | Yes  | true  | NULL    |       |
|               |               | sale_date | DATE   | Yes  | false | NULL    | NONE  |
|               |               | sale_amt  | BIGINT | Yes  | false | NULL    | NONE  |
|               |               |           |        |      |       |         |       |
| store_amt     | AGG_KEYS      | store_id  | INT    | Yes  | true  | NULL    |       |
|               |               | sale_amt  | BIGINT | Yes  | false | NULL    | SUM   |
+---------------+---------------+-----------+--------+------+-------+---------+-------+
~~~

查看该 Database 下的所有物化视图

~~~SQL
SHOW MATERIALIZED VIEW [IN|FROM db_name]
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

当物化视图创建完成后，用户再查询不同门店的销售量时，就会直接从刚才创建的物化视图 store_amt 中读取聚合好的数据，达到提升查询效率的效果。

用户的查询依旧指定查询 sales_records 表，比如：

~~~SQL
SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
~~~

使用 EXPLAIN 命令查询物化视图是否命中：

~~~SQL
EXPLAIN SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
~~~

结果为:

~~~PlainText

| Explain String                                                              |
+-----------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                             |
|  OUTPUT EXPRS:<slot 2> `store_id` | <slot 3> sum(`sale_amt`)                |
|   PARTITION: UNPARTITIONED                                                  |
|                                                                             |
|   RESULT SINK                                                               |
|                                                                             |
|   4:EXCHANGE                                                                |
|      use vectorized: true                                                   |
|                                                                             |
| PLAN FRAGMENT 1                                                             |
|  OUTPUT EXPRS:                                                              |
|   PARTITION: HASH_PARTITIONED: <slot 2> `store_id`                          |
|                                                                             |
|   STREAM DATA SINK                                                          |
|     EXCHANGE ID: 04                                                         |
|     UNPARTITIONED                                                           |
|                                                                             |
|   3:AGGREGATE (merge finalize)                                              |
|   |  output: sum(<slot 3> sum(`sale_amt`))                                  |
|   |  group by: <slot 2> `store_id`                                          |
|   |  use vectorized: true                                                   |
|   |                                                                         |
|   2:EXCHANGE                                                                |
|      use vectorized: true                                                   |
|                                                                             |
| PLAN FRAGMENT 2                                                             |
|  OUTPUT EXPRS:                                                              |
|   PARTITION: RANDOM                                                         |
|                                                                             |
|   STREAM DATA SINK                                                          |
|     EXCHANGE ID: 02                                                         |
|     HASH_PARTITIONED: <slot 2> `store_id`                                   |
|                                                                             |
|   1:AGGREGATE (update serialize)                                            |
|   |  STREAMING                                                              |
|   |  output: sum(`sale_amt`)                                                |
|   |  group by: `store_id`                                                   |
|   |  use vectorized: true                                                   |
|   |                                                                         |
|   0:OlapScanNode                                                            |
|      TABLE: sales_records                                                   |
|      PREAGGREGATION: ON                                                     |
|      partitions=1/1                                                         |
|      rollup: store_amt                                                      |
|      tabletRatio=10/10                                                      |
|      tabletList=22326,22328,22330,22332,22334,22336,22338,22340,22342,22344 |
|      cardinality=0                                                          |
|      avgRowSize=0.0                                                         |
|      numNodes=1                                                             |
|      use vectorized: true                                                   |
+-----------------------------------------------------------------------------+
~~~

查询计划树中的 OlapScanNode 显示 `PREAGGREGATION: ON` 和 `rollup: store_amt`，说明使用物化视图 store_amt 的预先聚合计算结果。也就是说查询已经命中到物化视图 store_amt，并直接从物化视图中读取数据了。

<br/>

### 删除物化视图

下列两种情形需要删除物化视图:

* 用户误操作创建物化视图，需要撤销该操作。
* 用户创建了大量的物化视图，导致数据导入速度过慢不满足业务需求，并且部分物化视图的相互重复，查询频率极低，可容忍较高的查询延迟，此时需要删除部分物化视图。

删除已经创建完成的物化视图:

~~~SQL
DROP MATERIALIZED VIEW IF EXISTS store_amt on sales_records;
~~~

删除处于创建中的物化视图，需要先取消异步任务，然后再删除物化视图。

首先获得 JobId，执行命令:

~~~SQL
show alter table rollup from db0;
~~~

结果为:

~~~PlainText
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+-------------+------+----------+---------+
| JobId | TableName     | CreateTime          | FinishedTime        | BaseIndexName | RollupIndexName | RollupId | TransactionId | State       | Msg  | Progress | Timeout |
| 22478 | table0        | 2020-09-27 01:46:42 | NULL                | table0        | mv              | 22479    | 676           | WAITING_TXN |      | NULL     | 86400   |
+-------+---------------+---------------------+---------------------+---------------+-----------------+----------+---------------+-------------+------+----------+---------+
~~~

取消正在创建的物化视图

~~~SQL
CANCEL ALTER MATERIALIZED VIEW FROM db_name.view_name
~~~

上述查询得到 JobId 为 22478，若要取消该 Job，则执行命令:

~~~SQL
CANCEL ALTER MATERIALIZED VIEW FROM db0.table0 (22478);
~~~

<br/>

## 最佳实践

### 精确去重

用户可以在明细表上使用表达式`bitmap_union(to_bitmap(col))`创建物化视图，实现原来聚合表才支持的基于 bitmap 的预先计算的精确去重功能。

比如，用户有一张计算广告业务相关的明细表，每条记录包含的信息有点击日期、点击的是什么广告、通过什么渠道点击、以及点击的用户是谁：

~~~SQL
CREATE TABLE advertiser_view_record(
    TIME date,
    advertiser varchar(10),
    channel varchar(10),
    user_id int
) distributed BY hash(TIME)
properties("replication_num" = "1");
~~~

用户查询广告 UV，使用下面查询语句：

~~~SQL
SELECT advertiser, channel, count(distinct user_id)
FROM advertiser_view_record
GROUP BY advertiser, channel;
~~~

这种情况下，可以创建物化视图，使用 bitmap_union 预先聚合:

~~~SQL
CREATE MATERIALIZED VIEW advertiser_uv AS
SELECT advertiser, channel, bitmap_union(to_bitmap(user_id))
FROM advertiser_view_record
GROUP BY advertiser, channel;
~~~

物化视图创建完毕后，查询语句中的`count(distinct user_id)`，会自动改写为`bitmap_union_count (to_bitmap(user_id))`以命中物化视图。

<br/>

### 近似去重

用户可以在明细表上使用表达式 `hll_union(hll_hash(col))` 创建物化视图，实现近似去重的预计算。

在同上一样的场景中，用户创建如下物化视图:

~~~SQL
CREATE MATERIALIZED VIEW advertiser_uv AS
SELECT advertiser, channel, hll_union(hll_hash(user_id))
FROM advertiser_view_record
GROUP BY advertiser, channel;
~~~

<br/>

### 匹配更丰富的前缀索引

用户的基表 tableA 有 (k1, k2, k3) 三列。其中 k1, k2 为排序键。这时候如果用户查询条件中包含 `where k1=1 and k2=2`，就能通过 shortkey 索引加速查询。但是用户查询语句中使用条件 `k3=3`，则无法通过 shortkey 索引加速。此时，可创建以 k3 作为第一列的物化视图:

~~~SQL
CREATE MATERIALIZED VIEW mv_1 AS
SELECT k3, k2, k1
FROM tableA
ORDER BY k3;
~~~

这时候查询就会直接从刚才创建的 mv_1 物化视图中读取数据。物化视图对 k3 是存在前缀索引的，查询效率也会提升。

<br/>

## 注意事项

### **物化视图函数支持**

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

### **物化视图的智能路由**

StarRocks 中，查询时不需要显式指定MV表名称，StarRocks 会根据查询 SQL 智能路由到最佳的 MV 表。在查询时，MV 表的选择规则如下：

1. 选择包含所有查询列的 MV 表
2. 按照过滤和排序的 Column 筛选最符合的 MV 表
3. 按照 Join 的 Column 筛选最符合的 MV 表
4. 行数最小的 MV 表
5. 列数最小的 MV 表

### **其他限制**

1. 物化视图的聚合函数的参数仅支持单列，比如：`sum(a+b)` 不支持。
2. 如果删除语句的条件列，在物化视图中不存在，则不能进行删除操作。如果一定要删除数据，则需要先将物化视图删除，然后方可删除数据。
3. 单表上过多的物化视图会影响导入的效率：导入数据时，物化视图和 base 表数据是同步更新的，如果一张表的物化视图表超过10张，则有可能导致导入速度很慢。这就像单次导入需要同时导入10张表数据是一样的。
4. 相同列，不同聚合函数，不能同时出现在一张物化视图中，比如：`select sum(a), min(a) from table` 不支持。
5. 物化视图的创建语句目前不支持 JOIN 和 WHERE ，也不支持 GROUP BY 的 HAVING 子句。
6. 不能同时创建多个物化视图，只能等待上一个物化视图创建完成，才能创建下一个物化视图。
7. RollUp表的模型必须和Base表保持一致（聚合表的RollUp表是聚合模型，明细表的RollUp表是明细模型）。
8. Delete 操作时，如果 Where 条件中的某个 Key 列在某个 RollUp表中不存在，则不允许进行 Delete。这种情况下，可以先删除物化视图，再进行Delete操作，最后再重新增加物化视图。
9. 如果 物化视图中包含 REPLACE 聚合类型的列，则该物化视图必须包含所有 Key 列。

