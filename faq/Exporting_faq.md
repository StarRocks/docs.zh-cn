# Export 常见问题

## 阿里云 OSS 备份与还原

StarRocks 支持备份数据到阿里云 OSS/AWS S3（或者兼容 S3 协议的对象存储）等。假设有两个 StarRocks 集群，分别 DB1 集群和 DB2 集群，我们需要将 DB1 中的数据备份到阿里云 OSS，然后在需要的时候恢复到 DB2，备份及恢复大致流程如下：

### 创建云端仓库

在 DB1 和 DB2 中分别执行 SQL：

```sql
CREATE REPOSITORY `仓库名称`
WITH BROKER `broker_name`
ON LOCATION "oss://存储桶名称/路径"
PROPERTIES
(
"fs.oss.accessKeyId" = "xxx",
"fs.oss.accessKeySecret" = "yyy",
"fs.oss.endpoint" = "oss-cn-beijing.aliyuncs.com"
);
```

a. DB1 和 DB2 都需要创建，且创建的 REPOSITORY 仓库名称要相同，仓库查看：

```sql
SHOW REPOSITORIES;
```

b. broker_name 需要填写一个集群中的 broker 名称，BrokerName 查看：

```sql
SHOW BROKER;
```

c. fs.oss.endpoint 后的路径不需要带存储桶名。

### 备份数据表

在 DB1 中将需要进行备份的表，BACKUP 到云端仓库。在 DB1 中执行 SQL：

```sql
BACKUP SNAPSHOT [db_name].{snapshot_name}
TO `repository_name`
ON (
`table_name` [PARTITION (`p1`, ...)],
...
)
PROPERTIES ("key"="value", ...);
```

```plain text
PROPERTIES 目前支持以下属性：
"type" = "full"：表示这是一次全量更新（默认）。
"timeout" = "3600"：任务超时时间，默认为一天，单位秒。
```

StarRocks 目前不支持全数据库的备份，我们需要在 ON (……)指定需要备份的表或分区，这些表或分区将并行的进行备份。
查看正在进行中的备份任务（注意同时进行的备份任务只能有一个）：

```sql
SHOW BACKUP FROM db_name;
```

备份完成后，可以查看 OSS 中备份数据是否已经存在（不需要的备份需在 OSS 中删除）：

```sql
SHOW SNAPSHOT ON OSS仓库名; 
```

### 数据还原

在 DB2 中进行数据还原，DB2 中不需要创建需恢复的表结构，在进行 Restore 操作过程中会自动创建。执行还原 SQL：

```sql
RESTORE SNAPSHOT [db_name].{snapshot_name}
FROMrepository_name``
ON (
    'table_name' [PARTITION ('p1', ...)] [AS 'tbl_alias'],
    ...
)
PROPERTIES ("key"="value", ...);
```

查看还原进度：

```sql
SHOW RESTORE;
```
