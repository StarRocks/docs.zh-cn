# ALTER CLUSTER

## description

该语句用于更新逻辑集群。需要有管理员权限

语法

```sql
ALTER CLUSTER cluster_name PROPERTIES ("key"="value", ...);
```

1. 缩容，扩容 （根据集群现有的 be 数目，大则为扩容，小则为缩容), 扩容为同步操作，缩容为异步操作，通过 backend 的状态可以得知是否缩容完成

```sql
PROERTIES ("instance_num" = "3")
```

instance_num 逻辑集群节点树

## example

1. 缩容，减少含有 3 个 be 的逻辑集群 test_cluster 的 be 数为 2

    ```sql
    ALTER CLUSTER test_cluster PROPERTIES ("instance_num"="2");
    ```

2. 扩容，增加含有 3 个 be 的逻辑集群 test_cluster 的 be 数为 4

    ```sql
    ALTER CLUSTER test_cluster PROPERTIES ("instance_num"="4");
    ```

## keyword

ALTER, CLUSTER
