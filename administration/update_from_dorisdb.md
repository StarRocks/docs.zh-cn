# 标准版DorisDB-x升级到社区版StarRocks-x

>注意：StarRocks可以做到前向兼容，所以在升级的时候可以做到灰度升级，但是一定是先升级BE再升级FE。(标准版安装包命名为DorisDB*,社区版安装包命名格式为StarRocks*)

## 准备升级环境

>注意：请留意环境变量配置准确

1. 创建操作目录，准备环境
  
    ```bash
    # 创建升级操作目录用来保存备份数据，并进入当前目录
    mkdir DorisDB-upgrade-$(date +"%Y-%m-%d") && cd DorisDB-upgrade-$(date +"%Y-%m-%d")
    # 设置当前目录路径作为环境变量，方便后续编写命令
    export DSDB_UPGRADE_HOME=`pwd`
    # 将您当前已经安装的标准版（DorisDB）路径设置为环境变量
    export DSDB_HOME=标准版的安装路径，形如"xxx/DorisDB-SE-1.17.6"
    ```

2. 下载社区版（StarRocks）安装包，并解压，解压文件根据您下载版本进行重命名
  
    ```bash
    wget "https://xxx.StarRocks-SE-1.18.4.tar.gz" -O StarRocks-1.18.4.tar.gz
    tar -zxvf StarRocks-1.18.4.tar.gz
    # 将社区版的解压文件路径设置为环境变量
    export STARROCKS_HOME= 社区版的解压安装路径，形如"xxx/StarRocks-1.18.4"
    ```

## 升级BE

逐台升级BE，确保每台升级成功，保证对数据没有影响

1. 先将标准版的BE停止

    ```bash
    # 停止BE
    cd ${DSDB_HOME}/be && ./bin/stop_be.sh
    ```

2. 当前标准版BE的lib和bin移动到备份目录下

    ```bash
    # 在DSDB_UPGRADE_HOME 目录下创建be目录，用于备份当前标准版be的lib
    cd ${DSDB_UPGRADE_HOME} && mkdir -p DorisDB-backups/be
    mv  ${DSDB_HOME}/be/lib DorisDB-backups/be/
    mv  ${DSDB_HOME}/be/bin DorisDB-backups/be/
    ```

3. 拷贝社区版（StarRocks）的BE文件到当前标准版(DorisDB)的BE的目录下

    ```bash
    cp -rf ${STARROCKS_HOME}/be/lib/ ${DSDB_HOME}/be/lib
    cp -rf ${STARROCKS_HOME}/be/bin/ ${DSDB_HOME}/be/bin
    ```

4. 重新启动BE

    ```bash
    # 启动BE
    cd ${DSDB_HOME}/be && ./bin/start_be.sh --daemon
    ```

5. 验证BE是否正确运行  

    a. 在MySQL客户端执行show backends/show proc '/backends' \G 查看BE是否成功启动并且版本是否正确  

    b. 观察be/log/be.INFO的日志是否正常  

    c. 观察be/log/be.WARNING 是否有异常日志  

6. 第一台观察10分钟后，按照上述流程执行其他BE节点

## 升级FE

> 注意：
>
> * 逐台升级FE，先升级Observer，再升级Follower，最后升级Master
>
> * 请将没有启动FE的节点的FE同步升级，以防后续这些节点启动FE出现FE版本不一致的情况

1. 先将当前节点FE停止

    ```bash
    cd ${DSDB_HOME}/fe && ./bin/stop_fe.sh
    ```

2. 创建FE目录备份标准版的FE的lib和bin

    ```bash
    cd ${DSDB_UPGRADE_HOME} && mkdir -p DorisDB-backups/fe
    mv ${DSDB_HOME}/fe/bin DorisDB-backups/fe/
    mv ${DSDB_HOME}/fe/lib DorisDB-backups/fe/
    ```

3. 拷贝社区版FE的lib和bin文件到标准版FE的目录下  

    ```bash
    cp -r ${STARROCKS_HOME}/fe/lib/ ${DSDB_HOME}/fe
    cp -r ${STARROCKS_HOME}/fe/bin/ ${DSDB_HOME}/fe
    ```

4. 备份meta_dir元数据信息  
  
    a. 将标准版的/fe/conf/fe.conf中设置的meta_dir目录进行备份，如未更改过配置文件中meta_dir属性，默认为"\${DSDB_HOME}/doris-meta"；下例中设置为"\${DSDB_HOME}/meta",  
  
    b. 注意保证"命令中元数据目录"和"元数据实际目录"还有"配置文件"中一致，如您原目录名为"doris-meta"，建议您将目录重命名，同步需要更改配置文件  
  
    ```bash
    mv doris-meta/ meta
    ```
  
    c. 配置文件示例：
  
    ```bash
    # store metadata, create it if it is not exist.
    # Default value is ${DORIS_HOME}/doris-meta
    # meta_dir = ${DORIS_HOME}/doris-meta
    meta_dir=xxxxx/DorisDB-SE-1.17.6/meta
    ```
  
    其中，meta_dir设置为创建的元数据存储目录路径

    ```bash
    cd "配置文件中设置的meta_dir目录,到meta的上级目录"
    # 拷贝meta文件，meta为本例中设置路径，请根据您配置文件中设置进行更改
    cp -r meta meta-$(date +"%Y-%m-%d")
    ```
  
5. 启动FE服务

    ```bash
    cd ${DSDB_HOME}/fe && ./bin/start_fe.sh --daemon
    ```

6. 验证FE是否正确运行  

    a. 在MySQL客户端执行`show frontends\G`查看FE是否成功启动并且版本是否正确  

    b. 观察fe/log/fe.INFO的日志是否正常  

    c. 观察fe/log/fe.WARNING 是否有异常日志  

7. 第一台观察10分钟后，按照上述流程执行其他FE节点

## 升级brokers

1. 停止当前节点的broker

    ```bash
    cd ${DSDB_HOME}/apache_hdfs_broker && ./bin/stop_broker.sh
    ```

2. 备份当前标准版的Broker运行的lib和bin

    ```bash
    cd ${DSDB_UPGRADE_HOME} && mkdir -p DorisDB-backups/apache_hdfs_broker
    mv ${DSDB_HOME}/apache_hdfs_broker/lib DorisDB-backups/apache_hdfs_broker/
    mv ${DSDB_HOME}/apache_hdfs_broker/bin DorisDB-backups/apache_hdfs_broker/
    ```

3. 拷贝社区版本Broker的lib和bin文件到执行Broker的目录下

    ```bash
    cp -rf ${STARROCKS_HOME}/apache_hdfs_broker/lib ${DSDB_HOME}/apache_hdfs_broker
    cp -rf ${STARROCKS_HOME}/apache_hdfs_broker/bin ${DSDB_HOME}/apache_hdfs_broker
    ```

4. 启动当前Broker

    ```bash
    cd ${DSDB_HOME}/apache_hdfs_broker && ./bin/start_broker.sh --daemon
    ```

5. 验证Broker是否正确运行

    a. 在MySQL客户端执行show broker 查看Broker是否成功启动并且版本是否正确

    b. 观察broker/log/broker.INFO的日志是否正常  

    c. 观察broker/log/broker.WARNING 是否有异常日志  

6. 按照上述流程执行其他Broker节点

## 回滚方案

>注意：从标准版DorisDB升级至社区版StarRocks，暂时不支持回滚，建议先在测试环境验证测试没问题后再升级线上。如有遇到问题无法解决，可以添加下面企业微信寻求帮助。

![二维码](../assets/8.3.1.png)

## 注意事项

1. 需要先升级 BE、再升级 FE，因为Starrocks的标准版中BE是兼容FE的。升级BE的过程中，需要进行灰度升级，先升级一台BE，过一天观察无误，再升级其他FE。

2. 跨版本升级时需要谨慎，测试环境验证后再操作，且升级Starrocks-2.x版本时需要前置开启CBO，需要升级至2.x版本的用户需先升级1.19.x版本。可在官网获取最新版本的1.19.x安装包（1.19.7）。
