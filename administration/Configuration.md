# 参数配置

服务启动之后，在运行的过程中可能会出现调整配置参数的情况，以满足业务需求。本文介绍部分 BE，FE 的配置项以及修改方式，除了部分 FE 配置项可通过命令热更新修改，其余配置项生效都需重启相应服务。

## FE 配置项

FE 部分配置项支持热更新，可通过

~~~sql
ADMIN SET FRONTEND CONFIG ("key" = "value");
~~~

命令进行参数热更新，其余参数的变更需要在对应 fe.conf 文件中修改并重启生效。
以下是可热更新修改的参数列表：

|配置项|默认值|作用|
|---|---|---|
|agent_task_resend_wait_time_ms|5000|当代理任务的创建时间被设置，并且距离现在超过该值，才能重新发送代理任务|
|alter_table_timeout_second|86400|Schema change 超时时间|
|authentication_ldap_simple_bind_base_dn|""|用户的 base DN，指定用户的检索范围|
|authentication_ldap_simple_bind_root_dn|""|检索用户时，使用的管理员账号 DN|
|authentication_ldap_simple_bind_root_pwd|""|检索用户时，使用的管理员账号密码|
|authentication_ldap_simple_server_host|""|LDAP 服务的 host 地址|
|authentication_ldap_simple_server_port|389|LDAP 服务的端口|
|authentication_ldap_simple_user_search_attr|uid|LDAP 对象中标识用户的属性名称|
|backup_job_default_timeout_ms|86400*1000|Backup 作业的超时时间|
|balance_load_disk_safe_threshold|0.5|disk_and_tablet 策略有效。如果所有 BE 的磁盘使用率低于 50%，认为磁盘使用均衡|
|balance_load_score_threshold|0.1|对于 be_load_score 策略，负载比平均负载低 10% 的 BE 处于低负载状态，比平均负载高 10% 的 BE 处于高负载状态，对于 disk_and_tablet 策略，如果最大和最小 BE 磁盘使用率之差高于 10%，认为磁盘使用不均衡，会触发 tablet 重新均衡|
|broker_load_default_timeout_second|14400|Broker Load 的超时时间，默认 4 个小时|
|capacity_used_percent_high_water|0.75|Backend 上磁盘使用容量的度量值，超过 0.75 之后，尽量不在往这个 tablet 上发送建表，克隆的任务，直到恢复正常|
|catalog_trash_expire_second|86400|删表/数据库之后，元数据在回收站中保留的时长，超过这个时长，数据就不可以在恢复|
|catalog_try_lock_timeout_ms|5000|Catalog Lock 获取的超时时长|
|check_consistency_default_timeout_second|600|副本一致性检测的超时时间|
|desired_max_waiting_jobs|100|最多等待的任务，适用于所有的任务，建表、导入、schema change|
|disable_balance|FALSE|禁用 Tablet 调度|
|disable_cluster_feature|TRUE|是否禁用逻辑集群功能|
|disable_colocate_balance|FALSE|禁用 Colocate Table 的副本均衡|
|disable_hadoop_load|FALSE|禁用从 Hadoop 导入|
|disable_load_job|false|不接收任何导入任务，集群出问题时的止损措施|
|drop_backend_after_decommission|TRUE|BE 被下线后，是否删除该 BE|
|dynamic_partition_check_interval_seconds|600|动态分区检查的时间周期|
|dynamic_partition_enable|TRUE|开启动态分区功能|
|edit_log_roll_num|50000|Image 日志拆分大小|
|enable_decimal_v3|TRUE|是否开启 Decimal V3|
|enable_local_replica_selection|FALSE|优化器优先选择与这个 FE 相同 IP 的 BE 节点上的 tablet|
|enable_materialized_view|TRUE|是否允许创建物化视图|
|enable_sql_blacklist|FALSE|是否开启 SQL Query 黑名单校验。如果开启，在黑名单中的 Query 不能被执行|
|enable_statistic_collect|TRUE|统计信息收集功能开关|
|enable_strict_storage_medium_check|FALSE|在创建表时，FE 是否检查 BE 的可用的存储介质空间|
|enable_vectorized_file_load|TRUE|开启向量化导入 CSV/JSON/Parquet 和 Spark 导入|
|export_max_bytes_per_be_per_task|268435456|单个导出任务在单个 be 上导出的最大数据量，默认 256M|
|export_running_job_num_limit|5|导出作业最大的运行数目|
|export_task_default_timeout_second|7200|导出作业超时时长，默认 2 小时|
|expr_children_limit|10000|查询中 in 中可以涉及的数目|
|expr_depth_limit|3000|查询嵌套的层次|
|force_do_metadata_checkpoint|FALSE|无论 JVM 内存使用率多少，都会产生元数据的 Checkpoint|
|history_job_keep_max_second|604800|历史任务最大的保留时长，例如 schema change 任务，默认 7 天|
|ignore_meta_check|false|忽略元数据落后的情形|
|ignore_unknown_log_id|FALSE|是否忽略未知的 logID。当 FE 回滚到低版本时，可能存在低版本 BE 无法识别的 logID。如果为 true，则 FE 会忽略这些 logID；否则 FE 会退出|
|insert_load_default_timeout_second|3600|Insert Into 语句的超时时间，默认 1 小时|
|label_keep_max_num|1000|一定时间内所保留导入任务的最大数量。<br> 保留时间在 label_keep_max_second 中进行设置|
|label_keep_max_second|259200|label 保留时长，默认 3 天，保留太久会消耗很多内存|
|load_parallel_instance_num|1|单个 BE 上并发实例数，默认 1 个|
|load_straggler_wait_second|300|控制 BE 副本最大容忍的导入落后时长，超过这个时长就进行克隆|
|max_allowed_in_element_num_of_delete|10000|DELETE 语句中 IN 谓词最多允许的元素数量|
|max_backend_down_time_second|3600|BE 和 FE 失联之后，FE 容忍 BE 重新加回来的最长时间|
|max_balancing_tablets|100|如果正在均衡的 tablet 数量超过该值，跳过 tablet 重新均衡|
|max_broker_concurrency|100|单个任务最大并发实例数，默认 100 个|
|max_clone_task_timeout_sec|2 *60* 60|克隆 Tablet 的最大超时时间，默认 2h|
|max_create_table_timeout_second|60|建表最大超时时间|
|max_distribution_pruner_recursion_depth|100|分区裁剪允许的最大递归深度|
|max_layout_length_per_row|2147483647|单行最大的长度，Integer.MAX_VALUE|
|max_load_timeout_second|259200|适用于所有导入，最大超时，3 天|
|max_partitions_in_one_batch|4096|批量创建分区时，分区数目的最大值|
|max_planner_scalar_rewrite_num|10_0000|优化器重写 ScalarOperator 允许的最大次数
|max_query_retry_time|2|FE 上查询重试的次数|
|max_routine_load_batch_size|524288000|每个 routine load task 导入的最大数据量，默认 500M|
|max_routine_load_job_num|100|最大的 routine load 作业数|
|max_routine_load_task_concurrent_num|5|每个 routine load 作业最大并发执行的 task 数|
|max_routine_load_task_num_per_be|5|每个 be 最大并发执行的 routine load task 数，需要小于等于 be 的配置 routine_load_thread_pool_size|
|max_running_rollup_job_num_per_table|1|每个 Table 执行 Rollup 任务的最大并发度|
|max_running_txn_num_per_db|100|并发导入的任务数|
|max_scheduling_tablets|2000|如果正在调度的 tablet 数量超过该值，跳过 tablet 均衡检查|
|max_small_file_number|100|允许存储小文件数目的最大值|
|max_small_file_size_bytes|1024*1024|存储文件的大小上限，默认 1MB
|max_stream_load_timeout_second|259200|Stream 导入的超时时间允许设置的最大值，默认 3 天|
|max_tolerable_backend_down_num|0|如果故障的 BE 节点数超过该阈值，则不能自动恢复 Routine Load 作业|
|meta_delay_toleration_second|300|非 master 节点容忍的最大元数据落后的时间|
|metadata_checkopoint_memory_threshold|60|如果 JVM 内存使用率超过该阈值，停止生成元数据的 Checkpoint，防止 OOM|
|min_bytes_per_broker_scanner|67108864|单个实例处理的最小数据量，默认 64MB|
|min_clone_task_timeout_sec|3*60|克隆 Tablet 的最小超时时间，默认 3min|
|min_load_timeout_second|1|适用于所有导入，最小超时，1 秒|
|period_of_auto_resume_min|5|自动恢复 Routine Load 的时间间隔|
|qe_slow_log_ms|5000|Slow query 的认定时长，默认 5000ms|
|recover_with_empty_tablet|FALSE|在 tablet 副本丢失/损坏时，使用空的 tablet 代替它。这样可以保证在有 tablet 副本丢失/损坏时，query 依然能被执行（但是由于缺失了数据，结果可能是错误的）|
|report_queue_size|100|Disk/Task/Tablet 的 Report 的等待队列长度|
|routine_load_task_consume_second|3|每个 routine load task 消费数据的最大时间，默认为 3s|
|routine_load_task_timeout_second|15|每个 routine load task 超时时间，默认 15s|
|shutdown_hook_timeout_sec|60|FE 优雅退出的等待时间
|spark_load_default_timeout_second|86400|Spark 导入的超时时间，默认 1 天
|statistic_collect_interval_sec|120*60|统计信息功能执行周期，默认 2h|
|statistic_sample_collect_rows|200000|采样统计信息 Job 的默认采样行数，默认为 200000 行|
|statistic_update_interval_sec|24 *60* 60|统计信息 Job 的默认收集间隔时间，默认为 1 天|
|statistics_manager_sleep_time_sec|60*10|自动创建统计信息表的周期，默认 10min|
|storage_flood_stage_left_capacity_bytes|1 *1024* 1024*1024|如果剩余空间小于该值，会拒绝 Load Restore 作业。默认 1GB|
|storage_flood_stage_usage_percent|95|如果空间使用率超过该值，会拒绝 Load 和 Restore 作业|
|storage_high_watermark_usage_percent|85|BE 存储目录下空间使用率的最大值|
|storage_min_left_capacity_bytes|2 *1024* 1024*1024|BE 存储目录下剩余空间的最小值，默认 2GB
|stream_load_default_timeout_second|600|StreamLoad 超时时间，默认 10 分钟|
|tablet_create_timeout_second|1|建表超时时长|
|tablet_delete_timeout_second|2|删除表的超时时间|
|tablet_repair_delay_factor_second|60|FE 控制进行副本修复的间隔|
|using_old_load_usage_pattern|FALSE|如果为 true，插入语句遇到错误时依然会返回 label 给用户|
|vectorized_load_enable|TRUE|开启向量化 Broker 导出和导入

以下 FE 配置项暂不支持热更新修改，生效需在 fe.conf 中修改并重启 fe 服务。

|配置项|默认值|作用|
|---|---|---|
|log_roll_size_mb|1024|日志拆分的大小，每 1G 拆分一个日志|
|sys_log_dir|StarRocksFe.STARROCKS_HOME_DIR/log|日志保留的目录|
|sys_log_level|INFO|日志级别，INFO < WARN < ERROR < FATAL|sys_log_roll_num|10|日志保留的数目|
|sys_log_verbose_modules|空字符串|日志打印的模块，写 org.apache.starrocks.catalog 就只打印 catalog 模块下的日志|
|sys_log_roll_interval|DAY|日志拆分的时间间隔|
|sys_log_delete_age|7d|日志删除的间隔|
|sys_log_roll_mode|1024|日志拆分的大小，每 1G 拆分一个日志|
|audit_log_dir|starrocksFe.STARROCKS_HOME_DIR/log|审计日志保留的目录|
|audit_log_roll_num|90|审计日志保留的数目|
|audit_log_modules|"slow_query", "query"|审计日志打印的模块，默认保留 slow_query 和 query|
|audit_log_roll_interval|DAY|审计日志拆分的时间间隔, DAY 或者 HOUR|
|audit_log_delete_age|30d|审计日志删除的间隔|
|audit_log_roll_mode|TIME-DAY|审计日志拆分模式|
|label_clean_interval_second|14400|label 清理的间隔|
|transaction_clean_interval_second|30|transaction 清理的间隔|
|meta_dir|StarRocksFe.STARROCKS_HOME_DIR/meta|元数据保留目录|
|tmp_dir|starrocksFe.STARROCKS_HOME_DIR/temp_ddir|临时文件保存目录，例如 backup/restore 等进程保留的目录|
|edit_log_port|9010|FE Group(Master, Follower, Observer)之间通信用的端口|
|master_sync_policy|SYNC|Master 日志刷盘的方式，默认是 SYNC|
|replica_sync_policy|SYNC|Follower 日志刷盘的方式，默认是 SYNC|
|replica_ack_policy|SIMPLE_MAJORITY|日志被认为有效的形式，默认是多数派返回确认消息，就认为生效|
|bdbje_heartbeat_timeout_second|30|BDBJE 心跳超时的间隔|
|bdbje_lock_timeout_second|1|BDBJE 锁超时的间隔|
|txn_rollback_limit|100|事务回滚的上限|
|frontend_address|0.0.0.0|FE IP 地址|
|priority_networks|空字符串|以 CIDR 形式 10.10.10.0/24 指定 FE IP 地址，适用于机器有多个 IP，需要特殊指定的形式|
|metadata_failure_recovery|false|强制重置 FE 的元数据，慎用|
|max_bdbje_clock_delta_ms|5000|Master 与 Non-master 最大容忍的时钟偏移|
|http_port|8030|Http Server 的端口|
|http_backlog_num|1024|HttpServer port backlog|
|thrift_backlog_num|1024|ThriftServer port backlog|
|rpc_port|9020|FE 上的 thrift server 端口|
|query_port|9030|FE 上的 mysql server 端口|
|mysql_service_nio_enabled|false|FE 连接服务的 nio 是否开启|
|mysql_service_io_threads_num|4|FE 连接服务线程数|
|auth_token|空字符串|Token 是否自动开启|
|publish_version_timeout_second|30|版本生效的超时时间|
|publish_version_interval_ms|10|发送版本生效任务的时间间隔|
|load_checker_interval_second|5|导入轮询的间隔|
|mini_load_default_timeout_second|3600|小批量导入的超时时间，默认 1 小时|
|async_load_task_pool_size|10|导入任务执行的线程池大小|
|storage_cooldown_second|2592000|介质迁移的时间，默认 30 天|
|export_checker_interval_second|5|导出线程轮询间隔|
|export_task_pool_size|5|导出任务线程池大小，默认 5 个|
|consistency_check_start_time|23|FE 发起副本一致性检测的起始时间|
|consistency_check_end_time|4|FE 发起副本一致性检测的终止时间|
|qe_max_connection|1024|FE 上最多接收的连接数，适用于所有用户|
|max_conn_per_user|100|单个用户能够处理的最大连接数|
|query_colocate_join_memory_limit_penalty_factor|8|Colocte Join 的内存限制|
|disable_colocate_join|false|false 表示开启 Colocate Join，true 表示不开启 Colocate Join|
|locale|zh_CN.UTF-8|字符集|
|remote_fragment_exec_timeout_ms|5000|FE 发送查询规划的 RPC 超时时间，不涉及任务执行|
|catalog_try_lock_timeout_ms|5000|Catalog Lock 获取的超时时长|
|es_state_sync_interval_second|10|FE 获取 Elastic Search Index 的时间|

## BE 配置项

BE 配置项暂不支持热更新修改，生效需在 be.conf 中修改并重启 be 服务。

|配置项|默认值|作用|
|---|---|---|
|be_port|9060|BE 上 thrift server 的端口，用于接收来自 FE 的请求|
|brpc_port|8060|BRPC 的端口，可以查看 BRPC 的一些网络统计信息|
|brpc_num_threads|-1|BRPC 的 bthreads 线程数量，-1 表示和 cpu 核数一样|
|priority_networks|空字符串|以 CIDR 形式 10.10.10.0/24 指定 BE IP 地址，适用于机器有多个 IP，需要指定优先使用的网络|
|heartbeat_service_port|9050|心跳服务端口（thrift），用户接收来自 FE 的心跳|
|heartbeat_service_thread_count|1|心跳线程数|
|create_tablet_worker_count|3|创建 tablet 的线程数|
|drop_tablet_worker_count|3|删除 tablet 的线程数|
|push_worker_count_normal_priority|3|导入线程数，处理 NORMAL 优先级任务|
|push_worker_count_high_priority|3|导入线程数，处理 HIGH 优先级任务|
|publish_version_worker_count|2|生效版本的线程数|
|clear_transaction_task_worker_count|1|清理事务的线程数|
|alter_tablet_worker_count|3|进行 schema change 的线程数|
|clone_worker_count|3|克隆的线程数|
|storage_medium_migrate_count|1|介质迁移的线程数，SATA 迁移到 SSD|
|check_consistency_worker_count|1|计算 tablet 的校验和(checksum)|
|report_task_interval_seconds|10|汇报单个任务的间隔。建表，删除表，导入，schema change 都可以被认定是任务|
|report_disk_state_interval_seconds|60|汇报磁盘状态的间隔。汇报各个磁盘的状态，以及上面的数据量等等|
|report_tablet_interval_seconds|60|汇报 tablet 的间隔。汇报所有的 tablet 的最新版本|
|alter_tablet_timeout_seconds|86400|Schema change 超时时间|
|sys_log_dir|${STARROCKS_HOME}/log|存放日志的地方，包括 INFO, WARNING, ERROR, FATAL 等日志|
|user_function_dir|${STARROKCS_HOME}/lib/udf|UDF 程序存放的地方|
|sys_log_level|INFO|日志级别，INFO < WARNING < ERROR < FATAL|
|sys_log_roll_mode|SIZE-MB-1024|日志拆分的大小，每 1G 拆分一个日志|
|sys_log_roll_num|10|日志保留的数目|
|sys_log_verbose_modules|*|日志打印的模块，写 olap 就只打印 olap 模块下的日志|
|sys_log_verbose_level|10|日志显示的级别，用于控制代码中 VLOG 开头的日志输出|
|log_buffer_level|空字符串|日志刷盘的策略，默认保持在内存中|
|num_threads_per_core|3|每个 CPU core 启动的线程数|
|compress_rowbatches|true|BE 之间 rpc 通信是否压缩 RowBatch，用于查询层之间的数据传输|
|serialize_batch|false|BE 之间 rpc 通信是否序列化 RowBatch，用于查询层之间的数据传输|
|status_report_interval|5|查询汇报 profile 的间隔，用于 FE 收集查询统计信息|
|doris_scanner_thread_pool_thread_num|48|存储引擎并发扫描磁盘的线程数，统一管理在线程池中|
|doris_scanner_thread_pool_queue_size|102400|存储引擎最多接收的任务数|
|doris_scan_range_row_count|524288|存储引擎拆分查询任务的粒度，512K|
|doris_scanner_queue_size|1024|存储引擎支持的扫描任务数|
|doris_scanner_row_num|16384|每个扫描线程单次执行最多返回的数据行数|
|doris_max_scan_key_num|1024|查询最多拆分的 scan key 数目|
|column_dictionary_key_ratio_threshold|0|字符串类型的取值比例，小于这个比例采用字典压缩算法|
|column_dictionary_key_size_threshold|0|字典压缩列大小，小于这个值采用字典压缩算法|
|memory_limitation_per_thread_for_schema_change|2|单个 schema change 任务允许占用的最大内存|
|max_unpacked_row_block_size|104857600|单个 block 最大的字节数，100MB|
|file_descriptor_cache_clean_interval|3600|文件句柄缓存清理的间隔，用于清理长期不用的文件句柄|
|disk_stat_monitor_interval|5|磁盘状态检测的间隔|
|unused_rowset_monitor_interval|30|清理过期 Rowset 的时间间隔|
|storage_root_path|空字符串|存储数据的目录|
|max_percentage_of_error_disk|0|磁盘错误达到一定比例，BE 退出|
|default_num_rows_per_data_block|1024|每个 block 的数据行数|
|max_tablet_num_per_shard|1024|每个 shard 的 tablet 数目，用于划分 tablet，防止单个目录下 tablet 子目录过多|
|pending_data_expire_time_sec|1800|存储引擎保留的未生效数据的最大时长|
|inc_rowset_expired_sec|1800|导入生效的数据，存储引擎保留的时间，用于增量克隆|
|max_garbage_sweep_interval|3600|磁盘进行垃圾清理的最大间隔|
|min_garbage_sweep_interval|180|磁盘进行垃圾清理的最小间隔|
|snapshot_expire_time_sec|172800|快照文件清理的间隔，48 个小时|
|trash_file_expire_time_sec|259200|回收站清理的间隔，72 个小时|
|row_nums_check|true|Compaction 完成之后，前后 Rowset 行数对比|
|file_descriptor_cache_capacity|32768|文件句柄缓存的容量|
|min_file_descriptor_number|60000|BE 进程的文件句柄 limit 要求的下线|
|index_stream_cache_capacity|10737418240|BloomFilter/Min/Max 等统计信息缓存的容量|
|storage_page_cache_limit|20G|PageCache 的容量|
|disable_storage_page_cache|true|表示关闭 PageCache|
|base_compaction_start_hour|20|BaseCompaction 开启的时间|
|base_compaction_check_interval_seconds|60|BaseCompaction 线程轮询的间隔|
|base_compaction_num_cumulative_deltas|5|BaseCompaction 触发条件之一：Cumulative 文件数目要达到的限制|
|base_compaction_num_threads_per_disk|1|每个磁盘 BaseCompaction 线程的数目|
|base_cumulative_delta_ratio|0.3|BaseCompaction 触发条件之一：Cumulative 文件大小达到 Base 文件的比例|
|base_compaction_interval_seconds_since_last_operation|86400|BaseCompaction 触发条件之一：上一轮 BaseCompaction 距今的间隔|
|base_compaction_write_mbytes_per_sec|5|BaseCompaction 写磁盘的限速|
|cumulative_compaction_check_interval_seconds|10|CumulativeCompaction 线程轮询的间隔|
|min_cumulative_compaction_num_singleton_deltas|5|CumulativeCompaction 触发条件之一：Singleton 文件数目要达到的下限|
|max_cumulative_compaction_num_singleton_deltas|1000|CumulativeCompaction 触发条件之一：Singleton 文件数目要达到的上限|
|cumulative_compaction_num_threads_per_disk|1|每个磁盘 CumulativeCompaction 线程的数目|
|cumulative_compaction_budgeted_bytes|104857600|BaseCompaction 触发条件之一：Singleton 文件大小的总和限制，100MB|
|cumulative_compaction_write_mbytes_per_sec|100|CumulativeCompaction 写磁盘的限速|
|min_compaction_failure_interval_sec|600|Tablet Compaction 失败之后，再次被调度的间隔。|
|max_compaction_concurrency|-1|BaseCompaction + CumulativeCompaction 的最大并发，-1 就是没有限制|
|webserver_port|8040|Http Server 端口|
|webserver_num_workers|5|Http Server 线程数|
|periodic_counter_update_period_ms|500|Counter 统计信息的间隔|
|load_data_reserve_hours|4|小批量导入生成的文件保留的时间|
|load_error_log_reserve_hours|48|导入数据信息保留的时长|
|number_tablet_writer_threads|16|流式导入的线程数|
|streaming_load_max_mb|10240|流式导入单个文件大小的上限|
|streaming_load_rpc_max_alive_time_sec|1200|流式导入 RPC 的超时时间|
|tablet_writer_rpc_timeout_sec|600|TabletWriter 的超时时长|
|fragment_pool_thread_num|64|查询线程数，默认启动 64 个线程，后续查询请求动态创建线程|
|fragment_pool_queue_size|1024|单节点上能够处理的查询请求上限|
|enable_partitioned_hash_join|false|使用 PartitionHashJoin|
|enable_partitioned_aggregation|true|使用 PartitionAggregation|
|enable_token_check|true|Token 开启检验|
|enable_prefetch|true|查询提前预取|
|load_process_max_memory_limit_bytes|107374182400|单节点上所有的导入线程占据的内存上限，100GB|
|load_process_max_memory_limit_percent|30|单节点上所有的导入线程占据的内存上限比例，默认 30%|
|sync_tablet_meta|false|存储引擎是否开 sync 保留到磁盘上。|
|thrift_rpc_timeout_ms|5000|Thrift 超时的时长|
|txn_commit_rpc_timeout_ms|10000|Txn 超时的时长|
|routine_load_thread_pool_size|10|例行导入的线程池数目|
|tablet_meta_checkpoint_min_new_rowsets_num|10|TabletMeta Checkpoint 的最小 Rowset 数目|
|tablet_meta_checkpoint_min_interval_secs|600|TabletMeta Checkpoint 线程轮询的时间间隔|
|default_rowset_type|ALPHA|存储引擎的格式，默认新 ALPHA，后面会替换成 BETA|
|brpc_max_body_size|2147483648|BRPC 最大的包容量，2GB|
|max_runnings_transactions|2000|存储引擎支持的最大事务数|
|tablet_map_shard_size|1|Tablet 分组数|
|enable_bitmap_union_disk_format_with_set | False |Bitmap 新存储格式，可以优化 bitmap_union 性能|
|aws_access_key_id|""|AWS Access Key Id|
|aws_secret_access_key|""|AWS Secret Access Key|
|aws_s3_endpoint|""|访问 AWS S3 或者是 Aliyun OSS 的 Endpoint|
|aws_s3_max_connection|102400|访问 AWS S3 的最大连接数量|
  
## Broker 配置项

参考 [Broker load 导入](../loading/BrokerLoad.md)

## 系统参数

Linux Kernel

建议 3.10 以上的内核。

CPU

scaling governor 用于控制 CPU 的能耗模式，默认是 on-demand 模式，使用 performance 能耗最高，性能也最好，StarRocks 部署建议采用 performance 模式。

~~~shell
echo 'performance' | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
~~~

内存

* **Overcommit**

建议使用 Overcommit，
建议把 cat /proc/sys/vm/overcommit_memory 设成  1。

~~~shell
echo 1 | sudo tee /proc/sys/vm/overcommit_memory
~~~

* **Huge Pages**

禁止 transparent huge pages，这个会干扰内存分配器，导致性能下降。

~~~shell
echo 'madvise' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
~~~

* **Swappiness**

关闭交换区，消除交换内存到虚拟内存时对性能的扰动。

~~~shell
echo 0 | sudo tee /proc/sys/vm/swappiness
~~~

磁盘

* **SATA**

mq-deadline 调度算法会排序和合并 I/O 请求，适合 SATA 磁盘。

~~~shell
echo mq-deadline | sudo tee /sys/block/vdb/queue/scheduler
~~~

* **SSD/NVME**

kyber 调度算法适用于 latency 低的设备，例如 NVME/SSD

~~~shell
echo kyber | sudo tee /sys/block/vdb/queue/scheduler
~~~

如果系统不支持 kyber，建议使用 none 调度算法

~~~shell
echo none | sudo tee /sys/block/vdb/queue/scheduler
~~~

网络

请至少使用 10 GB 网络，1GB 网络也能工作，但是会导致达不到预期性能。可以使用 iperf 测试系统带宽，确认是否是 10GB 网络。

文件系统

建议使用 Ext4 文件系统，可用相关命令进行查看挂载类型。

~~~shell
df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/vdb1      ext4     1008G  903G   55G  95% /home/disk1
~~~

其他系统配置

* **tcp abort on overflow**

~~~shell
echo 1 | sudo tee /proc/sys/net/ipv4/tcp_abort_on_overflow
~~~

* **somaxconn**

~~~shell
echo 1024 | sudo tee /proc/sys/net/core/somaxconn
~~~

系统资源

系统资源是指系统所能使用资源上限，配置在/etc/security/limits.conf，包括文件句柄、最大进程数、最大使用内存等。

* **文件句柄**

在部署的机器上运行 ulimit -n 65535，把文件句柄设为 65535。如果 ulimit 值重新登录后失效，尝试修改 /etc/ssh/sshd_config 中的 `UsePAM yes` ，然后重启 sshd 服务即可。

高并发配置

如果集群负载的并发度较高，建议添加以下配置

~~~shell
echo 120000 > /proc/sys/kernel/threads-max
echo 60000  > /proc/sys/vm/max_map_count
echo 200000  > /proc/sys/kernel/pid_max
~~~

* **max user processes**

ulimit -u 40960
