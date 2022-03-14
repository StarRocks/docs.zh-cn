# 内存管理

本章节主要介绍 StarRocks BE 中内存使用分类和内存管理、内存调优方法。

## 内存分类

| 标识 | Metric 名称 | 说明 | BE 相关配置 |
| --- | --- | --- | --- |
| process | starrocks_be_process_mem_bytes | BE 进程实际使用的内存（不包含预留的空闲内存）| mem_limit |
| query_pool | starrocks_be_column_pool_mem_bytes | BE 查询层使用总内存 | |
| load | starrocks_be_load_mem_bytes | 导入使用的总内存 | load_process_max_memory_limit_bytes, load_process_max_memory_limit_percent |
| table_meta | starrocks_be_tablet_meta_mem_bytes | 元数据总内存 | |
| compaction | starrocks_be_compaction_mem_bytes | 版本合并总内存 | compaction_max_memory_limit, compaction_max_memory_limit_percent |
| column_pool | starrocks_be_column_pool_mem_bytes | column pool 内存池，用于加速存储层数据读取的 Column Cache | |
| page_cache | starrocks_be_storage_page_cache_mem_bytes | BE 存储层 page 缓存 | disable_storage_page_cache, storage_page_cache_limit |
| chunk_allocator | starrocks_be_chunk_allocator_mem_bytes | CPU per core 缓存，用于加速小块内存申请的 Cache | chunk_reserved_bytes_limit |
| consistency | starrocks_be_consistency_mem_bytes | 定期一致性校验使用的内存 | consistency_max_memory_limit_percent, consistency_max_memory_limit |
| schema_change | starrocks_be_schema_change_mem_bytes | Schema Change 任务使用的总内存 | memory_limitation_per_thread_for_schema_change |
| clone | starrocks_be_clone_mem_bytes | Tablet Clone 任务使用的总内存 | |
| update | starrocks_be_update_mem_bytes | 主键模型使用的总内存 | |

## 内存相关的配置

* **BE 配置**

| 名称 | 默认值 | 说明|  
| --- | --- | --- |
| mem_limit | 90% | BE 进程内存上限，默认硬限为 BE 所在机器内存的 90%, 软限为 BE 所在机器内存的 80%。如果是 BE 独立部署的话，不需要配置，如果是和其它占用内存比较多的服务混合部署的话，要合理配置。|
| load_process_max_memory_limit_bytes | 107374182400 | 导入内存上限, 取 mem_limit * load_process_max_memory_limit_percent / 100 和 load_process_max_memory_limit_bytes 中较小的那个值, 导入内存到达限制，会触发刷盘和反压逻辑。|
| load_process_max_memory_limit_percent | 30 | 导入内存上限，取 mem_limit * load_process_max_memory_limit_percent / 100 和 load_process_max_memory_limit_bytes 中较小的那个值，导入内存到达限制，会触发刷盘和反压逻辑。|
| compaction_max_memory_limit | -1 | Compaction 内存上限，取 mem_limit * compaction_max_memory_limit_percent / 100 和 compaction_max_memory_limit 中较小的那个值，-1 表示没有限制，当前不建议修改默认配置。 |
| compaction_max_memory_limit_percent | 100 | Compaction 内存上限，取 mem_limit * compaction_max_memory_limit_percent / 100 和 compaction_max_memory_limit 中较小的那个值，-1 表示没有限制，当前不建议修改默认配置。|
| disable_storage_page_cache | true | 是否禁用 BE 存储层 page 缓存，和 storage_page_cache_limit 配合使用，在内存资源充足和有大数据量 Scan 的场景可以使用打开，能够加速查询性能。 |
| storage_page_cache_limit | 0 | BE 存储层 page 缓存可以使用的内存上限。|
| chunk_reserved_bytes_limit | 2147483648 | 用于加速小块内存分配的 Cache，默认上限为 2G，在内存资源充足的情况下可以考虑打开。|
| consistency_max_memory_limit_percent | 20 | 一致性校验任务使用的内存上限，取 mem_limit * consistency_max_memory_limit_percent / 100 和 consistency_max_memory_limit 中较小的那个值。内存使用超限，会导致任务失败。 |
| consistency_max_memory_limit | 10G | 一致性校验任务使用的内存上限，取 mem_limit * consistency_max_memory_limit_percent / 100 和 consistency_max_memory_limit 中较小的那个值。内存使用超限，会导致任务失败。 |
| memory_limitation_per_thread_for_schema_change | 2 | 单个 schema change 任务的内存使用上限，内存使用超限，会导致 schema change 任务失败。|
| tc_use_memory_min | 10737418240 | TCmalloc 最小预留内存，小于这个值，StarRocks 不会将空闲内存返还给操作系统。|

* **Session 变量**

| 名称| 默认值| 说明|
|  --- |  --- | --- |
| exec_mem_limit| 2147483648| 单个 instance 内存限制|
| load_mem_limit| 0| 单个导入任务的内存限制，如果是0,会取exec_mem_limit|

## 查看内存使用

* **mem\_tracker**

~~~ bash
// 看整体内存统计
<http://be_ip:be_web_port/mem_tracker>

// 看更细粒度的内存统计
<http://be_ip:be_web_port/mem_tracker?type=query_pool&upper_level=3>
~~~

* **tcmalloc**

~~~ bash
<http://be_ip:be_web_port/memz>
~~~

~~~plain text
------------------------------------------------
MALLOC:      777276768 (  741.3 MiB) Bytes in use by application
MALLOC: +   8851890176 ( 8441.8 MiB) Bytes in page heap freelist
MALLOC: +    143722232 (  137.1 MiB) Bytes in central cache freelist
MALLOC: +     21869824 (   20.9 MiB) Bytes in transfer cache freelist
MALLOC: +    832509608 (  793.9 MiB) Bytes in thread cache freelists
MALLOC: +     58195968 (   55.5 MiB) Bytes in malloc metadata
MALLOC:   ------------
MALLOC: =  10685464576 (10190.5 MiB) Actual memory used (physical + swap)
MALLOC: +  25231564800 (24062.7 MiB) Bytes released to OS (aka unmapped)
MALLOC:   ------------
MALLOC: =  35917029376 (34253.1 MiB) Virtual address space used
MALLOC:
MALLOC:         112388              Spans in use
MALLOC:            335              Thread heaps in use
MALLOC:           8192              Tcmalloc page size
------------------------------------------------
Call ReleaseFreeMemory() to release freelist memory to the OS (via madvise()).
Bytes released to the OS take up virtual address space but no physical memory.
~~~

通过这种方法，看到的内存是准确的，但是当前StarRocks，有些内存使用是**Reserve**的，但是并没有使用，TcMalloc统计的**实际上是Reserve的内存**，而不是实际使用的内存。

这里Bytes in use by application 指的是当前使用的内存.

* **metrics**

~~~bash
curl -XGET http://be_ip:be_web_port/metrics | grep 'mem'
curl -XGET http://be_ip:be_web_port/metrics | grep 'column_pool'
~~~

metrics的值是10秒更新一次，老的版本，可以通过这种方法，监控部分内存统计。
