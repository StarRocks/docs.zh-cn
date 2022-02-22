# DROP CLUSTER

## description

该语句用于删除逻辑集群，成功删除逻辑集群需要首先删除集群内的 db，需要管理员权限

语法：

```sql
DROP CLUSTER [IF EXISTS] cluster_name
```

## example

删除逻辑集群 test_cluster

```sql
DROP CLUSTER test_cluster;
```

## keyword

DROP, CLUSTER
