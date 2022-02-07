# CREATE TABLE AS SELECT

## description

该语句用于创建 table。

语法：

```sql
CREATE TABLE [IF NOT EXISTS] table_name
[(column_name1 [, column_name2, ...])]
[COMMENT "table comment"];
[partition_desc]
[distribution_desc]
[PROPERTIES ("key"="value", ...)]
AS SELECT <query>
  [ ... ]
```

说明：如果语句执行成功，则相当于根据查询自动推断创建了一张表，并把查询出来的结果插入到创建的表之中。
创建出来的表类型为OLAP类型，目前不支持其它引擎/外表。

注意：该过程在在目前的版本中并没有事务保证。

## syntax

1. col_name: 可选项，列名称，列的类型目前会自动推断出适合的类型。对于浮点类型会转换为DECIMAL(38,9)类型，对于字符串类型统一会转换为VARCHAR(1048576)

2. COMMENT: 可选项，表注释。

3. partition_desc: 可选项，分区描述信息，如果不写则默认建无分区的表。
   更详细的语法规则请参考: （[数据分布-批量创建和修改分区](../table_design/Data_distribution.md)）。

4. distribution_des: 可选项，分桶信息，如果不写则默认为10个分桶，分桶键如果有统计信息，则会取基最高的那一列做为分桶键。

5. PROPERTIES: 可选项，可以指定建表的附带的属性。

## example

1. 创建一个新表 new_table，注意，new_table 和 t2 的表结构可能不会一样。

    ```sql
    create table new_table as select * from t2;
    ```

2. 根据查询创建一个表，并指定列名为a，b，c，注意，指定的列的个数需要与查导语句的结果列数相匹配。

    ```sql
    create table new_table(a,b,c) as select k1,k2,k3 from t2;
    ```

3. 根据查询创建一个lineorder_flat表，并且明确分区和分桶的信息。

    LESS THAN

    ```sql
   CREATE TABLE lineorder_flat
   PARTITION BY RANGE(`LO_ORDERDATE`)(
   START ("1993-01-01") END ("1999-01-01") EVERY (INTERVAL 1 YEAR)
   )
   DISTRIBUTED BY HASH(`LO_ORDERKEY`) BUCKETS 120 
   AS SELECT
       l.LO_ORDERKEY AS LO_ORDERKEY,
       l.LO_LINENUMBER AS LO_LINENUMBER,
       l.LO_CUSTKEY AS LO_CUSTKEY,
       l.LO_PARTKEY AS LO_PARTKEY,
       l.LO_SUPPKEY AS LO_SUPPKEY,
       l.LO_ORDERDATE AS LO_ORDERDATE,
       l.LO_ORDERPRIORITY AS LO_ORDERPRIORITY,
       l.LO_SHIPPRIORITY AS LO_SHIPPRIORITY,
       l.LO_QUANTITY AS LO_QUANTITY,
       l.LO_EXTENDEDPRICE AS LO_EXTENDEDPRICE,
       l.LO_ORDTOTALPRICE AS LO_ORDTOTALPRICE,
       l.LO_DISCOUNT AS LO_DISCOUNT,
       l.LO_REVENUE AS LO_REVENUE,
       l.LO_SUPPLYCOST AS LO_SUPPLYCOST,
       l.LO_TAX AS LO_TAX,
       l.LO_COMMITDATE AS LO_COMMITDATE,
       l.LO_SHIPMODE AS LO_SHIPMODE,
       c.C_NAME AS C_NAME,
       c.C_ADDRESS AS C_ADDRESS,
       c.C_CITY AS C_CITY,
       c.C_NATION AS C_NATION,
       c.C_REGION AS C_REGION,
       c.C_PHONE AS C_PHONE,
       c.C_MKTSEGMENT AS C_MKTSEGMENT,
       s.S_NAME AS S_NAME,
       s.S_ADDRESS AS S_ADDRESS,
       s.S_CITY AS S_CITY,
       s.S_NATION AS S_NATION,
       s.S_REGION AS S_REGION,
       s.S_PHONE AS S_PHONE,
       p.P_NAME AS P_NAME,
       p.P_MFGR AS P_MFGR,
       p.P_CATEGORY AS P_CATEGORY,
       p.P_BRAND AS P_BRAND,
       p.P_COLOR AS P_COLOR,
       p.P_TYPE AS P_TYPE,
       p.P_SIZE AS P_SIZE,
       p.P_CONTAINER AS P_CONTAINER
   FROM lineorder AS l
   INNER JOIN customer AS c ON c.C_CUSTKEY = l.LO_CUSTKEY
   INNER JOIN supplier AS s ON s.S_SUPPKEY = l.LO_SUPPKEY
   INNER JOIN part AS p ON p.P_PARTKEY = l.LO_PARTKEY;
    ```

## keyword

CREATE,TABLE,AS,SELECT
