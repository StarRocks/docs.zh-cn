# PrimaryKey Load

StarRocks 在 1.19 版本推出了主键模型，相较更新模型，主键模型（Primary Key）可以更好地支持实时/频繁更新的功能。该类型的表要求有唯一的主键，支持对表中的行按主键进行更新和删除操作。

## 支持的数据导入方式

* Stream Load
* Broker Load
* Routine Load

## 内部实现

插入/更新/删除目前支持使用导入的方式完成，通过 SQL 语句(`insert`/`update`/`delete`)来操作数据的功能会在未来版本中支持。目前支持的导入方式有 stream load、broker load、routine load、Json 数据导入。当前 Spark load 还未支持。

StarRocks 目前不会区分 `insert`/`upsert`，所有的操作默认都为 `upsert` 操作，使用原有的 stream load/broker load 功能导入数据时默认为 upsert 操作。

为了在导入中同时支持 upsert 和 delete 操作，StarRocks 在 stream load 和 broker load 语法中加入了特殊字段 `__op`。该字段用来表示该行的操作类型，其取值为 0 时代表 `upsert` 操作，取值为 1 时为 `delete` 操作。在导入的数据中, 可以添加一列, 用来存储 `__op` 操作类型, 其值只能是 0(表示 `upsert`)或者 1(表示 `delete`)。

## 使用 Stream Load / Broker Load 导入

Stream load 和 broker load 的操作方式类似，根据导入的数据文件的操作形式有如下几种情况。这里通过一些例子来展示具体的导入操作：

**1.** 当导入的数据文件只有 `upsert` 操作时可以不添加 `__op` 列。可以指定 `__op` 为 `upsert`，也可以不做任何指定，StarRocks 会默认导入为 `upsert`。例如想要向表 t 中导入如下内容：

~~~text
# 导入内容
0,aaaa
1,bbbb
2,\N
4,dddd
~~~

Stream Load 导入语句：

~~~Bash
#不指定__op
curl --location-trusted -u root: -H "label:lineorder" \
    -H "column_separator:," -T demo.csv \
    http://localhost:8030/api/demo_db/demo_tbl1/_stream_load
#指定__op
curl --location-trusted -u root: -H "label:lineorder" \
    -H "column_separator:," -H " columns:__op ='upsert'" -T demo.csv \
    http://localhost:8030/api/demo_db/demo_tbl1/_stream_load
~~~

Broker Load 导入语句：

~~~sql
#不指定__op
load label demo_db.label1 (
    data infile("hdfs://localhost:9000/demo.csv")
    into table demo_tbl1
    format as "csv"
) with broker "broker1";

#指定__op
load label demo_db.label2 (
    data infile("hdfs://localhost:9000/demo.csv")
    into table demo_tbl1
    format as "csv"
    set (__op ='upsert')
) with broker "broker1";
~~~

**2.** 当导入的数据文件只有 delete 操作时，也可以不添加 `__op` 列，只需指定 `__op` 为 delete。例如想要删除如下内容：

~~~text
#导入内容
1, bbbb
4, dddd
~~~

注意：`delete` 虽然只用到 primary key 列，但同样要提供全部的列，与 `upsert` 保持一致。

Stream Load 导入语句：

~~~bash
curl --location-trusted -u root: -H "label:lineorder" -H "column_separator:," \
    -H " columns:__op ='delete'" -T demo.csv \
    http://localhost:8030/api/demo_db/demo_tbl1/_stream_load
~~~

Broker Load 导入语句：

~~~sql
load label demo_db.label3 (
    data infile("hdfs://localhost:9000/demo.csv")
    into table demo_tbl1
    format as "csv"
    set (__op ='delete')
) with broker "broker1";  
~~~

**3.** 当导入的数据文件中包含 upsert 和 delete 混合时，需要指定额外的 `__op` 来表明操作类型。例如想要导入如下内容：

~~~text
1,bbbb,1
4,dddd,1
5,eeee,0
6,ffff,0
~~~

注意：

* `delete` 虽然只用到 primary key 列，但同样要提供全部的列，与 `upsert` 保持一致
* 上述导入内容表示删除 id 为 1、4 的行，添加 id 为 5、6 的行

Stream Load 导入语句：

~~~bash
curl --location-trusted -u root: -H "label:lineorder" -H "column_separator:," \
    -H " columns: c1,c2,c3,pk=c1, col0=c2,__op=c3 " -T demo.csv \
    http://localhost:8030/api/demo_db/demo_tbl1/_stream_load
~~~

其中，指定了 `__op` 为第三列。

Brokder Load 导入语句：

~~~bash
load label demo_db.label4 (
    data infile("hdfs://localhost:9000/demo.csv")
    into table demo_tbl1
    format as "csv"
    (c1,c2,c3)
    set (pk=c1, col0=c2, __op=c3)
) with broker "broker1";
~~~

其中，指定了 `__op` 为第三列。

更多关于 Stream Load 和 Broker Load 使用方法请参考 [STREAM LOAD](../loading/StreamLoad.md) 和 [BROKER LOAD](../loading/BrokerLoad.md)

## 使用 Routine Load 导入

可以在创建 routine load 的语句中，在 columns 最后增加一列，指定为在 `__op`。在真实导入中，`__op` 为 0 则表示 `upsert` 操作，为 1 则表示 `delete` 操作。例如导入如下内容：

**示例 1** 导入 CSV 数据

~~~bash
2020-06-23  2020-06-23 00: 00: 00 beijing haidian 1   -128    -32768  -2147483648    0
2020-06-23  2020-06-23 00: 00: 01 beijing haidian 0   -127    -32767  -2147483647    1
2020-06-23  2020-06-23 00: 00: 02 beijing haidian 1   -126    -32766  -2147483646    0
2020-06-23  2020-06-23 00: 00: 03 beijing haidian 0   -125    -32765  -2147483645    1
2020-06-23  2020-06-23 00: 00: 04 beijing haidian 1   -124    -32764  -2147483644    0
~~~

Routine Load 导入语句：

~~~bash
CREATE ROUTINE LOAD routine_load_basic_types_1631533306858 on primary_table_without_null 
COLUMNS (k1, k2, k3, k4, k5, v1, v2, v3, __op),
COLUMNS TERMINATED BY '\t' 
PROPERTIES (
    "desired_concurrent_number" = "1",
    "max_error_number" = "1000",
    "max_batch_interval" = "5"
) FROM KAFKA (
    "kafka_broker_list" = "localhgost:9092",
    "kafka_topic" = "starrocks-data"
    "kafka_offsets" = "OFFSET_BEGINNING"
);
~~~

**示例 2** 导入 JSON 数据，源数据中有字段表示 upsert 或者 delete，比如下面常见的 Canal 同步到 Kafka 的数据样例，`type` 可以表示本次操作的类型（当前还不支持同步 DDL 语句）。

数据样例：

~~~json
{
    "data": [{
        "query_id": "3c7ebee321e94773-b4d79cc3f08ca2ac",
        "conn_id": "34434",
        "user": "zhaoheng",
        "start_time": "2020-10-19 20:40:10.578",
        "end_time": "2020-10-19 20:40:10"
    }],
    "database": "center_service_lihailei",
    "es": 1603111211000,
    "id": 122,
    "isDdl": false,
    "mysqlType": {
        "query_id": "varchar(64)",
        "conn_id": "int(11)",
        "user": "varchar(32)",
        "start_time": "datetime(3)",
        "end_time": "datetime"
    },
    "old": null,
    "pkNames": ["query_id"],
    "sql": "",
    "sqlType": {
        "query_id": 12,
        "conn_id": 4,
        "user": 12,
        "start_time": 93,
        "end_time": 93
    },
    "table": "query_record",
    "ts": 1603111212015,
    "type": "INSERT"
}
~~~

导入语句:

~~~sql
CREATE ROUTINE LOAD cdc_db.label5 ON cdc_table
COLUMNS(pk, col0, temp,__op =(CASE temp WHEN "DELETE" THEN 1 ELSE 0 END))
PROPERTIES
(
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "20",
    "max_error_number" = "1000",
    "strict_mode" = "false",
    "format" = "json",
    "jsonpaths" = "[\"$.data[0].query_id\",\"$.data[0].conn_id\",\"$.data[0].user\",\"$.data[0].start_time\",\"$.data[0].end_time\",\"$.type\"]"
)
FROM KAFKA
(
    "kafka_broker_list" = "localhost:9092",
    "kafka_topic" = "cdc-data",
    "property.group.id" = "starrocks-group",
    "property.client.id" = "starrocks-client",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
~~~

**示例 3** 导入 JSON 数据，源数据中有字段可以分别表示 upsert(0)和 delete(1)类型，可以不指定__op。

数据样例：op_type 字段的取值只有 0 和 1，分别表示 upsert 和 delete。

~~~json
{"pk": 1, "col0": "123", "op_type": 0}
{"pk": 2, "col0": "456", "op_type": 0}
{"pk": 1, "col0": "123", "op_type": 1}
~~~

建表语句:

~~~sql
CREATE TABLE `demo_tbl2` (
  `pk` bigint(20) NOT NULL COMMENT "",
  `col0` varchar(65533) NULL COMMENT ""
) ENGINE = OLAP 
PRIMARY KEY(`pk`)
COMMENT "OLAP"
DISTRIBUTED BY HASH(`pk`) BUCKETS 3
~~~

导入语句：

~~~sql
CREATE ROUTINE LOAD demo_db.label6 ON demo_tbl2
COLUMNS(pk, col0,__op)
PROPERTIES
(
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "20",
    "max_error_number" = "1000",
    "strict_mode" = "false",
    "format" = "json",
    "jsonpaths" = "[\"$.pk\",\"$.col0\",\"$.op_type\"]"
)
FROM KAFKA
(
    "kafka_broker_list" = "localhost:9092",
    "kafka_topic" = "pk-data",
    "property.group.id" = "starrocks-group",
    "property.client.id" = "starrocks-client",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
~~~

查询结果

~~~sql
mysql > select * from demo_db.demo_tbl2;
+------+------+
| pk   | col0 |
+------+------+
|    2 | 456  |
+------+------+
~~~

Routine Load 更多使用方法请参考 [ROUTINE LOAD](../loading/RoutineLoad.md)。
