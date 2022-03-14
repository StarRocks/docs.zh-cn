# 创建表

## 使用MySQL客户端访问StarRocks

安装部署好StarRocks集群后，可使用MySQL客户端连接某一个FE实例的query_port(默认9030)连接StarRocks， StarRocks内置root用户，密码默认为空：

```shell
mysql -h fe_host -P9030 -u root
```

## 创建数据库

使用root用户建立example\_db数据库:

```sql
mysql > create database example_db;
```

通过`show databases;`查看数据库信息：

```Plain Text
mysql > show databases;

+--------------------+
| Database           |
+--------------------+
| example_db         |
| information_schema |
+--------------------+
2 rows in set (0.00 sec)
```

information_schema是为了兼容mysql协议而存在，实际中信息可能不是很准确，所以关于具体数据库的信息建议通过直接查询相应数据库而获得。

<br/>

## 建表

StarRocks支持[多种表模型](../table_design/Data_model.md)，分别适用于不同的应用场景，以[明细表](../table_design/Data_model.md#明细模型)为例书写建表语句：

```sql
use example_db;
CREATE TABLE IF NOT EXISTS detailDemo (
    make_time     DATE           NOT NULL COMMENT "YYYY-MM-DD",
    mache_verson  TINYINT        COMMENT "range [-128, 127]",
    mache_num     SMALLINT       COMMENT "range [-32768, 32767] ",
    de_code       INT            COMMENT "range [-2147483648, 2147483647]",
    saler_id      BIGINT         COMMENT "range [-2^63 + 1 ~ 2^63 - 1]",
    pd_num        LARGEINT       COMMENT "range [-2^127 + 1 ~ 2^127 - 1]",
    pd_type       CHAR(20)        NOT NULL COMMENT "range char(m),m in (1-255) ",
    pd_desc       VARCHAR(500)   NOT NULL COMMENT "upper limit value 65533 bytes",
    us_detail     STRING         NOT NULL COMMENT "upper limit value 65533 bytes",
    relTime       DATETIME       COMMENT "YYYY-MM-DD HH:MM:SS",
    channel       FLOAT          COMMENT "4 bytes",
    income        DOUBLE         COMMENT "8 bytes",
    account       DECIMAL(12,4)  COMMENT "",
    ispass        BOOLEAN        COMMENT "true/false"
) ENGINE=OLAP
DUPLICATE KEY(make_time, mache_verson)
PARTITION BY RANGE (make_time) (
    START ("2022-03-11") END ("2022-03-15") EVERY (INTERVAL 1 day)
)
DISTRIBUTED BY HASH(make_time, mache_verson) BUCKETS 8
PROPERTIES(
    "replication_num" = "3",
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-3",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "8"
);
```

可以通过`show tables;`命令查看当前库的所有表，通过`desc table_name;`命令可以查看表结构。通过`show create table table_name;`可查看建表语句。请注意：在StarRocks中字段名不区分大小写，表名区分大小写。

表创建成功后，可以参考[导入查询](/quick_start/Import_and_query.md)章节[Stream load Demo](/quick_start/Import_and_query.md#stream-load%E5%AF%BC%E5%85%A5demo)进行数据导入及查询操作。

更改建表语法详见[CREATE TABLE](/sql-reference/sql-statements/data-definition/CREATE%20TABLE.md)章节。

<br/>

### 建表语句详解

<br/>

#### 排序键

StarRocks表内部组织存储数据时会按照指定列排序，这些列为排序列（Sort Key），明细模型中由`DUPLICATE KEY`指定排序列，以上demo中的`make_time, mache_verson`两列为排序列。注意排序列在建表时应定义在其他列之前。排序键详细描述以及不同数据模型的表的设置方法请参考[排序键](../table_design/Sort_key.md)。

<br/>

#### 字段类型

StarRocks表中支持多种字段类型，除demo中已经列举的字段类型，还支持[BITMAP类型](/using_starrocks/Using_bitmap.md)，[HLL类型](../using_starrocks/Using_HLL.md)，[Array类型](../using_starrocks/Array.md)，字段类型介绍详见[数据类型章节](/sql-reference/sql-statements/data-types/)。

建表时尽量使用精确的类型。例如整形就不要用字符串类型，INT类型满足则不要使用BIGINT，精确的数据类型能够更好的发挥数据库的性能。

<br/>

#### 分区，分桶

`PARTITION`关键字用于给表[创建分区](/sql-reference/sql-statements/data-definition/CREATE%20TABLE.md#Syntax)，当前demo中使用`make_time`进行范围分区，从11日到15日每天创建一个分区。StarRocks支持动态生成分区，`PROPERTIES`中的`dynamic_partition`开头的相关属性配置都是为表设置动态分区。详见[动态分区管理](/table_design/Data_distribution.md#动态分区管理)。

`DISTRIBUTED`关键字用于给表[创建分桶](/sql-reference/sql-statements/data-definition/CREATE%20TABLE.md#Syntax)，当前demo中使用`make_time, mache_verson`两个字段通过Hash算法创建32个桶。

创建表时合理的分区和分桶设计可以优化表的查询性能，分区分桶列如何选择详见[数据分布章节](/table_design/Data_distribution.md)。

<br/>

#### 表模型

`DUPLICATE`关键字表示当前表为明细模型，`KEY`中的列表示当前表的排序列。StarRocks支持多种数据模型，分别为[明细模型](/table_design/Data_model.md#明细模型)，[聚合模型](/table_design/Data_model.md#聚合模型)，[更新模型](/table_design/Data_model.md#更新模型)，[主键模型](/table_design/Data_model.md#主键模型)。不同模型的适用于多种业务场景，合理选择可优化查询效率。

<br/>

#### 索引

StarRocks默认会给Key列创建稀疏索引加速查询，具体规则见[排序键和shortke index](/table_design/Sort_key.md#排序列的原理)章节。支持的索引类型有[Bitmap索引](/table_design/Bitmap_index.md#原理)，[Bloomfilter索引](/table_design/Bloomfilter_index.md#原理)等。

注意：索引创建对表模型和列有要求，详细说明见对应索引介绍章节。

<br/>

#### ENGINE 类型

默认为 olap。可选 mysql，elasticsearch，hive，ICEBERG 代表创建表为[外部表](/using_starrocks/External_table.md#外部表)。

## Schema修改

### 修改Schema

使用[ALTER TABLE](/sql-reference/sql-statements/data-definition/ALTER%20TABLE.md)命令可以修改表的Schema，包括增加列，删除列，修改列类型（暂不支持修改列名称），改变列顺序。

以下举例说明。

  <br/>

原表table1的Schema如下:

```Plain Text
+----------+-------------+------+-------+---------+-------+
| Field    | Type        | Null | Key   | Default | Extra |
+----------+-------------+------+-------+---------+-------+
| siteid   | int(11)     | Yes  | true  | 10      |       |
| citycode | smallint(6) | Yes  | true  | N/A     |       |
| username | varchar(32) | Yes  | true  |         |       |
| pv       | bigint(20)  | Yes  | false | 0       | SUM   |
+----------+-------------+------+-------+---------+-------+
```

  <br/>

新增一列uv，类型为BIGINT，聚合类型为SUM，默认值为0:

```sql
ALTER TABLE table1 ADD COLUMN uv BIGINT SUM DEFAULT '0' after pv;
```

  <br/>

Schema Change为异步操作，提交成功后，可以通过以下命令查看:

```sql
SHOW ALTER TABLE COLUMN\G
```

当作业状态为FINISHED，则表示作业完成。新的Schema 已生效。

  <br/>

ALTER TABLE完成之后, 可以通过desc table查看最新的schema：

```Plain Text
mysql> desc table1;

+----------+-------------+------+-------+---------+-------+
| Field    | Type        | Null | Key   | Default | Extra |
+----------+-------------+------+-------+---------+-------+
| siteid   | int(11)     | Yes  | true  | 10      |       |
| citycode | smallint(6) | Yes  | true  | N/A     |       |
| username | varchar(32) | Yes  | true  |         |       |
| pv       | bigint(20)  | Yes  | false | 0       | SUM   |
| uv       | bigint(20)  | Yes  | false | 0       | SUM   |
+----------+-------------+------+-------+---------+-------+
5 rows in set (0.00 sec)
```

  <br/>

可以使用以下命令取消当前正在执行的作业:

```sql
CANCEL ALTER TABLE COLUMN FROM table1\G
```

  <br/>

## 创建用户并授权

StarRocks中拥有[Create_priv权限](../administration/User_privilege.md#权限类型)的用户才可建立数据库。

example_db数据库创建完成之后，可以通过root账户example_db读写权限授权给test账户，授权之后采用test账户登录就可以操作example\_db数据库了：

```sql
mysql > create user 'test' identified by '123456';
mysql > grant all on example_db to test;
```

<br/>

退出root账户，使用test登录StarRocks集群：

```sql
mysql > exit

mysql -h 127.0.0.1 -P9030 -utest -p123456
```

更多用户权限介绍请参考[用户权限章节](/administration/User_privilege.md)，创建用户更改用户密码相关命令详见[用户账户管理章节](/sql-reference/sql-statements/account-management/)。
