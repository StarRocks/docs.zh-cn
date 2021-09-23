# 查询常见问题

## [bigint等值查询中加引号]出现多余数据

```sql
mysql>select cust_id,idno from llyt_dev.dwd_mbr_custinfo_dd where Pt= ‘2021-06-30’ and cust_id = ‘20210129005809043707’ limit 10 offset 0;
```

```plain text
+---------------------+-----------------------------------------+
|   cust_id           |      idno                               |
+---------------------+-----------------------------------------+
|  20210129005809436  | yjdgjwsnfmdhjw294F93kmHCNMX39dw=        |
|  20210129005809436  | sdhnswjwijeifme3kmHCNMX39gfgrdw=        |
|  20210129005809436  | Tjoedk3js82nswndrf43X39hbggggbw=        |
|  20210129005809436  | denuwjaxh73e39592jwshbnjdi22ogw=        |
|  20210129005809436  | ckxwmsd2mei3nrunjrihj93dm3ijin2=        |
|  20210129005809436  | djm2emdi3mfi3mfu4jro2ji2ndimi3n=        |
+---------------------+-----------------------------------------+
```

```sql
mysql>select cust_id,idno from llyt_dev.dwd_mbr_custinfo_dd where Pt= ‘2021-06-30’ and cust_id = 20210129005809043707 limit 10 offset 0;
```

```plain text
+---------------------+-----------------------------------------+
|   cust_id           |      idno                               |
+---------------------+-----------------------------------------+
|  20210189979989976  | xuywehuhfuhruehfurhghcfCNMX39dw=        |
+---------------------+-----------------------------------------+
```

现象描述：

where里bigint类型，查询加单引号，查出很多无关数据。
解决方案：
字符串和int比较，相当于 cast 成double。int比较时，不要加引号。同时，加了引号，还会导致没法命中索引。
2. StarRocks有decode函数吗？
不支持Oracle中的decode函数，StarRocks语法兼容MySQL，可以使用case when。
3. StarRocks的主键覆盖是立刻生效的吗？还是说要等后台慢慢合并数据？
StarRocks的后台合并就是参考google的mesa的模型，有两层compaction，会后台策略触发合并。如果没有合并完成，查询的时候会合并，但是读出来只会有一个最新的版本，不存在「导入后数据读不到最新版本」的情况。
4. StarRocks存储utf8mb4的字符 会不会被截断或者乱码？
MySQL的“utf8mb4”是真正的“UTF-8”，所以DoirsDB没问题，参见 https://www.techug.com/post/in-mysql-never-use-utf8-use-utf8mb4.html
5. [Schema change] alter table 时显示：table's state is not normal
解决方案：
Alter table 是异步的，之前有过alter table 操作还没完成，可以通过
show tablet from lineitem where State="ALTER"; 查看alter状态，执行时间与数据量大小有关系，一般是分钟级别，建议alter过程中停止数据导入。导入会降低alter 速度。
6. [hive外部表查询问题] 查询hive外部表的时候，报错信息为：get partition detail failed: org.apache.doris.common.DdlException: get hive partition meta data failed: java.net.UnknownHostException:hadooptest（具体hdfs-ha的名字）
HA：hadoop2.0的功能，为了解决NameNode单点故障而设计的。
https://blog.csdn.net/wdr2003/article/details/79852438

解决方案:
把core-site.xml和hdfs-site.xml文件拷贝到 fe/conf 和 be/conf中即可。

问题原因：
获取配置单元分区元数据失败。
7. 大表查询结果慢，没有谓词下推
多张大表关联时，旧planner有时没有自动谓词下推，比如A JION B ON A.col1=B.col1 JOIN C on B.col1=C.col1 where A.col1='北京' ，可以更改为A JION B ON A.col1=B.col1 JOIN C on A.col1=C.col1 where A.col1='北京'，或者升级较新版本并开启CBO，然后会有此类谓词下推操作，优化查询性能。
8. 查询报错Doris planner use long time 3000 remaining task num 1
解决方案：
查看fe.gc日志看是否是多并发引起的full gc问题。
如果查看后台监控和日志初步判断有频繁gc，两个方案：
  1. 可以让sqlclient去同时访问多个fe去做负载均衡；
  2. 修改fe.conf中jvm8g为16g（更大内存，减少 full gc 影响）；
9. 当A基数很小时，select B from tbl order by A limit 10查询结果每次不一样
解决方案：
select B from tbl order by A,B limit 10，将B也进行排序就能保证结果一致。
问题原因：
上面的SQL只能保证A是有序的，并不能保证每次查询出来的B顺序是一致的，MySQL能保证这点因为它是单机数据库，而StarRocks是分布式数据库，底层表数据存储是sharding的，A的数据分布在多台机器上，每次查询多台机器返回的顺序可能不同，就会导致每次B顺序不一致
