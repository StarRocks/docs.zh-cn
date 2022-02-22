# ENTER

## description

该语句用于进入一个逻辑集群, 所有创建用户、创建数据库都需要在一个逻辑集群内执行，并且创建后隶属于这个逻辑集群，需要管理员权限。

```sql
ENTER cluster_name
```

## example

1. 进入逻辑集群 test_cluster

    ```sql
    ENTER test_cluster;
    ```

## keyword

ENTER
