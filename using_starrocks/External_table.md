# 外部表

StarRocks 支持以外部表的形式，接入其他数据源。外部表指的是保存在其他数据源中的数据表，而 StartRocks 只保存表对应的元数据，并直接向外部表所在数据源发起查询。目前 StarRocks 已支持的第三方数据源包括 MySQL、ElasticSearch、Hive、StarRocks 以及 Apache Iceberg。**对于 StarRocks 数据源，现阶段只支持 Insert 写入，不支持读取，对于其他数据源，现阶段只支持读取，还不支持写入**。

<br/>

## MySQL外部表

星型模型中，数据一般划分为维度表和事实表。维度表数据量少，但会涉及 UPDATE 操作。目前 StarRocks 中还不直接支持 UPDATE 操作（可以通过 Unique 数据模型实现），在一些场景下，可以把维度表存储在 MySQL 中，查询时直接读取维度表。

<br/>

在使用MySQL的数据之前，需在 StarRocks 创建外部表，与之相映射。StarRocks 创建 MySQL 外部表时需要指定 MySQL 的相关连接信息，如下图。

~~~sql
CREATE EXTERNAL TABLE mysql_external_table
(
    k1 DATE,
    k2 INT,
    k3 SMALLINT,
    k4 VARCHAR(2048),
    k5 DATETIME
)
ENGINE=mysql
PROPERTIES
(
    "host" = "127.0.0.1",
    "port" = "3306",
    "user" = "mysql_user",
    "password" = "mysql_passwd",
    "database" = "mysql_db_test",
    "table" = "mysql_table_test"
);
~~~

参数说明：

* **host**：MySQL 的连接地址
* **port**：MySQL 的连接端口号
* **user**：MySQL 登陆的用户名
* **password**：MySQL 登陆的密码
* **database**：MySQL 相关数据库名
* **table**：MySQL 相关数据表名

<br/>

## ElasticSearch外部表

StarRocks 与 ElasticSearch 都是目前流行的分析系统，StarRocks 强于大规模分布式计算，ElasticSearch 擅长全文检索。StarRocks 支持 ElasticSearch 访问的目的，就在于将这两种能力结合，提供更完善的一个 OLAP 解决方案。

### 建表示例

~~~sql
CREATE EXTERNAL TABLE elastic_search_external_table
(
    k1 DATE,
    k2 INT,
    k3 SMALLINT,
    k4 VARCHAR(2048),
    k5 DATETIME
)
ENGINE=ELASTICSEARCH
PARTITION BY RANGE(k1)
()
PROPERTIES (
    "hosts" = "http://192.168.0.1:8200,http://192.168.0.2:8200",
    "user" = "root",
    "password" = "root",
    "index" = "tindex",
    "type" = "doc"
);
~~~

参数说明：

* **host**：ES 集群连接地址，可指定一个或多个，StarRocks 通过这个地址获取到ES版本号、index 的 shard 分布信息
* **user**：开启**basic认证**的ES集群的用户名，需要确保该用户有访问 /*cluster/state/*nodes/http 等路径权限和对 index 的读权限
* **password**：对应用户的密码信息
* **index**：StarRocks 中的表对应的 ES 的 index 名字，可以是 alias
* **type**：指定 index 的 type，默认是**doc**
* **transport**：内部保留，默认为**http**

<br/>

### 谓词下推

StarRocks 支持对 ElasticSearch 表进行谓词下推，把过滤条件推给 ElasticSearch 进行执行，让执行尽量靠近存储，提高查询性能。目前支持下推的算子如下表：

|   SQL syntax  |   ES syntax  |
| :---: | :---: |
|  =   |  term query   |
|  in   |  terms query   |
|  \>=,  <=, >, <   |  range   |
|  and   |  bool.filter   |
|  or   |  bool.should   |
|  not   |  bool.must_not   |
|  not in   |  bool.must_not + terms   |
|  esquery   |  ES Query DSL   |

表1 ：支持的谓词下推列表

<br/>

### 查询示例

通过**esquery函数**将一些**无法用 sql 表述的 ES query**如 match、geoshape 等下推给ES进行过滤处理。esquery 的第一个列名参数用于关联index，第二个参数是ES的基本 Query DSL 的 json 表述，使用花括号{}包含，**json的root key有且只能有一个**，如 match、geo_shape、bool 等。

* match 查询：

~~~sql
select * from es_table where esquery(k4, '{
    "match": {
       "k4": "StarRocks on elasticsearch"
    }
}');
~~~

* geo 相关查询：

~~~sql
select * from es_table where esquery(k4, '{
  "geo_shape": {
     "location": {
        "shape": {
           "type": "envelope",
           "coordinates": [
              [
                 13,
                 53
              ],
              [
                 14,
                 52
              ]
           ]
        },
        "relation": "within"
     }
  }
}');
~~~

* bool 查询：

~~~sql
select * from es_table where esquery(k4, ' {
     "bool": {
        "must": [
           {
              "terms": {
                 "k1": [
                    11,
                    12
                 ]
              }
           },
           {
              "terms": {
                 "k2": [
                    100
                 ]
              }
           }
        ]
     }
  }');
~~~

<br/>

### 注意事项

* ES 在5.x之前和之后的数据扫描方式不同，目前**只支持5.x之后的版本**；
* 支持使用 HTTP Basic 认证方式的 ES 集群；
* 一些通过 StarRocks 的查询会比直接请求 ES 会慢很多，比如 count 相关的 query 等。这是因为 ES 内部会直接读取满足条件的文档个数相关的元数据，不需要对真实的数据进行过滤操作，使得 count 的速度非常快。

<br/>

## Hive外表

### 创建Hive资源

StarRocks 使用 Hive 资源来管理使用到的 Hive 集群相关配置，如 Hive Metastore 地址等，一个 Hive 资源对应一个 Hive 集群。创建 Hive 外表的时候需要指定使用哪个 Hive 资源。

~~~sql
-- 创建一个名为 hive0 的 Hive 资源
CREATE EXTERNAL RESOURCE "hive0"
PROPERTIES (
  "type" = "hive",
  "hive.metastore.uris" = "thrift://10.10.44.98:9083"
);

-- 查看 StarRocks 中创建的资源
SHOW RESOURCES;

-- 删除名为 hive0 的资源
DROP RESOURCE "hive0";
~~~

<br/>

### 创建数据库

~~~sql
CREATE DATABASE hive_test;
USE hive_test;
~~~

<br/>

### 创建 Hive 外表

~~~sql
-- 语法
CREATE EXTERNAL TABLE table_name (
  col_name col_type [NULL | NOT NULL] [COMMENT "comment"]
) ENGINE=HIVE
PROPERTIES (
  "key" = "value"
);

-- 例子：创建 hive0 资源对应的Hive集群中 rawdata 数据库下的 profile_parquet_p7 表的外表
CREATE EXTERNAL TABLE `profile_wos_p7` (
  `id` bigint NULL,
  `first_id` varchar(200) NULL,
  `second_id` varchar(200) NULL,
  `p__device_id_list` varchar(200) NULL,
  `p__is_deleted` bigint NULL,
  `p_channel` varchar(200) NULL,
  `p_platform` varchar(200) NULL,
  `p_source` varchar(200) NULL,
  `p__city` varchar(200) NULL,
  `p__province` varchar(200) NULL,
  `p__update_time` bigint NULL,
  `p__first_visit_time` bigint NULL,
  `p__last_seen_time` bigint NULL
) ENGINE=HIVE
PROPERTIES (
  "resource" = "hive0",
  "database" = "rawdata",
  "table" = "profile_parquet_p7"
);

~~~

说明：

* 外表列：
  * 列名需要与 Hive 表一一对应。
  * 列顺序与 Hive 表的关系。如果 Hive 表的存储格式为 Parquet 或 ORC，则列的顺序**不需要**与 Hive 表一致。如果 Hive 表的存储格式为 CSV，则列的顺序**需要**与 Hive 表一致。
  * 可以只选择 Hive 表中的**部分列**，但**分区列**必须要全部包含。
  * 外表的分区列无需通过 partition by 语句指定，需要与普通列一样定义到描述列表中。不需要指定分区信息，StarRocks 会自动从 Hive 同步。
  * ENGINE 指定为 HIVE。
* PROPERTIES 属性：
  * **hive.resource**：指定使用的 Hive 资源。
  * **database**：指定 Hive 中的数据库。
  * **table**：指定 Hive 中的表，**不支持view**。
* 支持的列类型对应关系如下表：

    |  Hive列类型   | StarRocks列类型    | 描述 |
    | --- | --- | ---|
    |   INT/INTEGER  | INT    |
    |   BIGINT  | BIGINT    |
    |   TIMESTAMP  | DATETIME    |TIMESTAMP转成DATETIME，会损失精度和时区信息，<br/>根据sessionVariable中的时区转成无时区DATETIME|
    |  STRING  | VARCHAR   |
    |  VARCHAR  | VARCHAR   |
    |  CHAR  | CHAR   |
    |  DOUBLE | DOUBLE |
    | FLOATE | FLOAT|
    | DECIMAL | DECIMAL |

    说明：

  * Hive 表 Schema 变更**不会自动同步**，需要在 StarRocks 中重建 Hive 外表。
  * 支持Hive的存储格式为 Parquet，ORC 和 CSV 格式，在 2.1 版本后支持了 TEXTFILE 格式。
    > 如果为 CSV 格式，则暂不支持引号为转义字符。
  * 压缩格式支持 snappy，lz4。

<br/>

### 查询 Hive 外表

~~~sql
-- 查询 profile_wos_p7 的总行数
select count(*) from profile_wos_p7;
~~~

<br/>

### 配置

* fe 配置文件路径为 fe/conf，如果需要自定义 hadoop 集群的配置可以在该目录下添加配置文件，例如：hdfs 集群采用了高可用的 nameservice，需要将 hadoop 集群中的 hdfs-site.xml 放到该目录下，如果 hdfs 配置了 viewfs，需要将 core-site.xml 放到该目录下。
* be 配置文件路径为 be/conf，如果需要自定义 hadoop 集群的配置可以在该目录下添加配置文件，例如：hdfs 集群采用了高可用的 nameservice，需要将 hadoop 集群中的 hdfs-site.xml 放到该目录下，如果 hdfs 配置了 viewfs，需要将 core-site.xml 放到该目录下。
* be 所在的机器也需要配置 JAVA_HOME，一定要配置成 jdk 环境，不能配置成 jre 环境
* kerberos 支持：
  1. 在所有的 fe/be 机器上用`kinit -kt keytab_path principal`登陆，该用户需要有访问 hive 和 hdfs 的权限。kinit 命令登陆是有实效性的，需要将其放入 crontab 中定期执行。
  2. 把 hadoop 集群中的 hive-site.xml/core-site.xml/hdfs-site.xml 放到 fe/conf 下，把 core-site.xml/hdfs-site.xml 放到 be/conf 下。
  3. 在 fe/conf/fe.conf 文件中的 JAVA_OPTS/JAVA_OPTS_FOR_JDK_9 选项加上 -Djava.security.krb5.conf:/etc/krb5.conf，/etc/krb5.conf 是 krb5.conf 文件的路径，可以根据自己的系统调整。
  4. resource 中的 uri 地址一定要使用域名，并且相应的 hive 和 hdfs 的域名与 ip 的映射都需要配置到 /etc/hosts 中。
* S3 支持:
  2.0.1 及之后的版本默认不开启此功能，可以按照以下步骤配置后使用。
  1. 下载[依赖库](https://cdn-thirdparty.starrocks.com/hive_s3_jar.tar.gz)并添加到 fe/lib/和be/lib/hadoop/hdfs/ 路径下。
  2. 在 fe/conf/core-site.xml 和 be/conf/core-site.xml 中加入如下配置，并重启 fe 和 be。

~~~xml
<configuration>
   <property>
      <name>fs.s3a.access.key</name>
      <value>******</value>
   </property>
   <property>
      <name>fs.s3a.secret.key</name>
      <value>******</value>
   </property>
   <property>
      <name>fs.s3a.endpoint</name>
      <value>s3.us-west-2.amazonaws.com</value>
   </property>
   <property>
     <name>fs.s3a.connection.maximum</name>
     <value>500</value>
   </property>
</configuration>
~~~

  1. `fs.s3a.access.key` 指定 aws 的 access key id
  2. `fs.s3a.secret.key` 指定 aws 的 secret access key
  3. `fs.s3a.endpoint` 指定 aws 的区域
  4. `fs.s3a.connection.maximum` 配置最大链接数，如果查询过程中有报错`Timeout waiting for connection from poll`，可以适当调高该参数

### 缓存更新

* hive 的 partition 信息以及 partition 对应的文件信息都会缓存在 starrocks 中，缓存的刷新时间为 hive_meta_cache_refresh_interval_s，默认 7200，缓存的失效时间为 hive_meta_cache_ttl_s，默认86400。

* 也可以手动刷新元数据信息：
  1. hive 中新增或者删除分区时，需要刷新**表**的元数据信息：`REFRESH EXTERNAL TABLE hive_t`，其中 hive_t 是 starrocks 中的外表名称。
  2. hive 中向某些 partition 中新增数据时，需要**指定partition**进行刷新：`REFRESH EXTERNAL TABLE hive_t PARTITION ('k1=01/k2=02', 'k1=03/k2=04')`，其中 hive_t 是 starrocks 中的外表名称，'k1=01/k2=02'、 'k1=03/k2=04'是 hive 中的 partition 名称。

## StarRocks 外部表

1.19版本开始，StarRocks 支持将数据通过外表方式写入另一个 StarRocks 集群的表中。这可以解决用户的读写分离需求，提供更好的资源隔离。用户需要首先在目标集群上创建一张目标表，然后在源 StarRocks 集群上创建一个 Schema 信息一致的外表，并在属性中指定目标集群和表的信息。

通过 insert into 写入数据至 StarRocks 外表,可以实现如下目标:

* 集群间的数据同步
* 在外表集群计算结果写入目标表集群，并在目标表集群提供查询服务，实现读写分离

以下是创建目标表和外表的实例：

~~~sql
# 在目标集群上执行
CREATE TABLE t
(
    k1 DATE,
    k2 INT,
    k3 SMALLINT,
    k4 VARCHAR(2048),
    k5 DATETIME
)
ENGINE=olap
DISTRIBUTED BY HASH(k1) BUCKETS 10;

# 在外表集群上执行
CREATE EXTERNAL TABLE external_t
(
    k1 DATE,
    k2 INT,
    k3 SMALLINT,
    k4 VARCHAR(2048),
    k5 DATETIME
)
ENGINE=olap
DISTRIBUTED BY HASH(k1) BUCKETS 10
PROPERTIES
(
    "host" = "127.0.0.1",
    "port" = "9020",
    "user" = "user",
    "password" = "passwd",
    "database" = "db_test",
    "table" = "t"
);

# 向外表插入数据,线上推荐使用第二种方式
insert into external_t values ('2020-10-11', 1, 1, 'hello', '2020-10-11 10:00:00');

insert into external_t select * from other_table;
~~~

其中：

* **EXTERNAL**：该关键字指定创建的是 StarRocks 外表
* **host**：该属性描述目标表所属 StarRocks 集群 Master FE 的 IP 地址
* **port**：该属性描述目标表所属 StarRocks 集群 Master FE 的 RPC 访问端口，该值可参考配置 fe/fe.conf 中的 rpc_port 配置取值
* **user**：该属性描述目标表所属 StarRocks 集群的访问用户名
* **password**：该属性描述目标表所属 StarRocks 集群的访问密码
* **database**：该属性描述目标表所属数据库名称
* **table**：该属性描述目标表名称

目前StarRocks外表使用上有以下限制：

* 仅可以在外表上执行 insert into 和 show create table 操作，不支持其他数据写入方式，也不支持查询和DDL
* 创建外表语法和创建普通表一致，但其中的列名等信息请保持同其对应的目标表一致
* 外表会周期性从目标表同步元信息（同步周期为10秒），在目标表执行的 DDL 操作可能会延迟一定时间反应在外表上

## Apache Iceberg外表

StarRocks 支持通过外表的方式查询 Apache Iceberg 数据湖中的数据，帮助您实现对数据湖的极速分析。本文介绍如何在StarRock创建外表，查询 Apache Iceberg 中的数据。

### 前提条件

请确认 StarRocks 有权限访问 Apache Iceberg 对应的 Hive Metastore、HDFS 集群或者对象存储的 Bucket。

### 注意事项

* Iceberg 外表是只读的，只能用于查询操作。
* 支持 Iceberg 的表格式为 V1（Copy on write表），暂不支持为 V2（Merge on read表）。V1 和 V2 之间的更多区别，请参见[Apache Iceberg官网](https://iceberg.apache.org/#spec/#format-versioning)。
* 支持 Iceberg 文件的压缩格式为 GZIP（默认值），ZSTD，LZ4 和 SNAPPY。
* 仅支持 Iceberg 的Catalog 类型为 Hive Catalog，数据存储格式为 Parquet和ORC。
* StarRocks 暂不⽀持同步 Iceberg 中的[schema evolution](https://iceberg.apache.org/#evolution#schema-evolution)，如果Iceberg表schema evolution发生变更，您需要在StarRocks中删除对应Iceberg外表并重新建立。

### 操作步骤

#### 步骤一：创建和管理 Iceberg 资源

您需要提前在 StarRocks 中创建 Iceberg 资源，用于管理在 StarRocks 中创建的 Iceberg 数据库和外表。

执行如下命令，创建一个名为`iceberg0`的 Iceberg 资源。

~~~sql
CREATE EXTERNAL RESOURCE "iceberg0" 
PROPERTIES ( 
"type" = "iceberg", 
"starrocks.catalog-type"="HIVE", 
"iceberg.catalog.hive.metastore.uris"="thrift://192.168.0.81:9083" 
);
~~~

|  参数   | 说明  |
|  ----  | ----  |
| type  | 资源类型，固定取值为**iceberg**。 |
| starrocks.catalog-type  | Iceberg 的 Catalog 类型。目前仅支持为 Hive Catalog，取值为 HIVE。 |
| iceberg.catalog.hive.metastore.uris | Hive Metastore 的 thrift URI。<br>Iceberg通过创建Hive Catalog，连接 Hive Metastore，以创建并管理表。您需要传入该Hive Metastore的 thrift URI。格式为**thrift://<Hive Metadata的IP地址>:<端口号>**，端口号默认为 9083。 |

执行如下命令，查看 StarRocks 中的所有 Iceberg 资源。

~~~sql
SHOW RESOURCES;
~~~~

执行如下命令，删除名为`iceberg0`的 Iceberg 资源。

~~~sql
DROP RESOURCE "iceberg0";
~~~~

> 删除 Iceberg 资源会导致其包含的所有 Iceberg 外表不可用，但 Apache Iceberg 中的数据并不会丢失。如果您仍需要通过 StarRocks 查询 Iceberg 的数据，请重新创建 Iceberg 资源，Iceberg 数据库和外表。

#### 步骤二：创建 Iceberg 数据库

执行如下命令，在 StarRocks 中创建并进入名为`iceberg_test`的 Iceberg 数据库。

~~~sql
CREATE DATABASE iceberg_test; 

USE iceberg_test; 
~~~

> 库名无需与 Iceberg 的实际库名保持一致。

#### 步骤三：创建 Iceberg 外表

执行如下命令，在 Iceberg 数据库`iceberg_test`中，创建一张名为`iceberg_tbl`的 Iceberg 外表。

~~~sql
CREATE EXTERNAL TABLE `iceberg_tbl` ( 
`id` bigint NULL, 
`data` varchar(200) NULL 
) ENGINE=ICEBERG 
PROPERTIES ( 
"resource" = "iceberg0", 
"database" = "iceberg", 
"table" = "iceberg_table" 
); 
~~~

* 相关参数说明，请参见下表：

| **参数**     | **说明**                       |
| ------------ | ------------------------------ |
| **ENGINE**   | 固定为**ICEBERG**，无需更改。  |
| **resource** | StarRocks 中 Iceberg 资源的名称。 |
| **database** | Iceberg 中的数据库名称。        |
| **table**    | Iceberg 中的数据表名称。        |

* 表名无需与 Iceberg 的实际表名保持一致。
* 列名需要与 Iceberg 的实际列名保持一致，列的顺序无需保持一致。
* 您可以按照业务需求选择 Iceberg 表中的全部或部分列。支持的数据类型以及与 StarRocks 对应关系，请参见下表。

| Apache Iceberg 中列的数据类型 | StarRocks 中列的数据类型 |
| ---------------------------- | ----------------------- |
| BOOLEAN                      | BOOLEAN                 |
| INT                          | TINYINT/SMALLINT/INT    |
| LONG                         | BIGINT                  |
| FLOAT                        | FLOAT                   |
| DOUBLE                       | DOUBLE                  |
| DECIMAL(P,S)                 | DECIMAL                 |
| DATE                         | DATE/DATETIME           |
| TIME                         | BIGINT                  |
| TIMESTAMP                    | DATETIME                |
| STRING                       | STRING/VARCHAR          |
| UUID                         | STRING/VARCHAR          |
| FIXED(L)                     | CHAR                    |
| BINARY                       | VARCHAR                 |

> 如果 Apache Iceberg 部分列的数据类型为 TIMESTAMPTZ、STRUCT、LIST、MAP，则 StarRocks 暂不支持通过 Iceberg 关联外表的方式访问此数据类型。

#### 步骤四：查询 Iceberg 外表

创建 Iceberg 外表后，无需导入数据，执行如下命令，即可查询 Iceberg 的数据。

~~~sql
select count(*) from iceberg_tbl;
~~~
