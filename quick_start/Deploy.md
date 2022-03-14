# 手动部署

StarRocks可通过[Mysql客户端进行连接](#%E4%BD%BF%E7%94%A8mysql%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AE%BF%E9%97%AEstarrocks)，使用Alter/Drop命令添加/删除fe/be节点，实现对集群的[扩容/缩容](../administration/Scale_up_down.md)操作。

## 环境准备

集群节点需要以下环境支持：

* Linux (Centos 7+)
* Java 1.8+
* CPU需要支持AVX2指令集
* ulimit -n 配置65535
* 集群时钟需同步
* 网络需要万兆网卡和万兆交换机

通过`cat /proc/cpuinfo |grep avx2`命令查看节点配置，有结果则cpu支持AVX2指令集。

测试集群建议节点配置：BE推荐16核64GB以上，FE推荐8核16GB以上。

系统参数配置建议请参考文末[参数配置](#参数设置);

## 下载StarRocks

可以通过[StarRocks官网下载](https://www.starrocks.com/zh-CN/download)获取二进制产品包。

下载的安装包可直接解压后进行安装部署，如有[下载源码](https://github.com/StarRocks/starrocks)并编译安装包的需求，可以使用[Docker进行编译](../administration/Build_in_docker.md)。

将StarRocks的二进制产品包分发到目标主机的部署路径并解压，可以考虑使用新建的StarRocks用户来管理。

以下载产品包为starrocks-1.0.0.tar.gz为例， 解压(tar -xzvf starrocks-1.0.0.tar.gz)后文件目录结构如下:

```Plain Text
StarRocks-XX-1.0.0
├── be  # BE目录
│   ├── bin
│   │   ├── start_be.sh # BE启动脚本
│   │   └── stop_be.sh  # BE关闭脚本
│   ├── conf
│   │   └── be.conf     # BE配置文件
│   ├── lib
│   │   ├── starrocks_be  # BE可执行文件
│   │   └── meta_tool
│   └── www
├── fe  # FE目录
│   ├── bin
│   │   ├── start_fe.sh # FE启动脚本
│   │   └── stop_fe.sh  # FE关闭脚本
│   ├── conf
│   │   └── fe.conf     # FE配置文件
│   ├── lib
│   │   ├── starrocks-fe.jar  # FE jar包
│   │   └── *.jar           # FE 依赖的jar包
│   └── webroot
└── udf
```

## 部署FE

### FE的基本配置

FE的配置文件为StarRocks-XX-1.0.0/fe/conf/fe.conf， 此处仅列出其中JVM配置和元数据目录配置，生产环境可参考[FE参数配置](../administration/Configuration.md#FE配置项)对集群进行详细优化配置。

### FE单实例部署

```bash
cd StarRocks-XX-1.0.0/fe
```

第一步: 配置文件conf/fe.conf：

```bash
# 元数据目录
meta_dir = ${STARROCKS_HOME}/meta
# JVM配置
JAVA_OPTS = "-Xmx8192m -XX:+UseMembar -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xloggc:$STARROCKS_HOME/log/fe.gc.log"
```

可以根据FE内存大小调整-Xmx8192m，为了避免GC建议16G以上，StarRocks的元数据都在内存中保存。
<br/>

第二步: 创建元数据目录，需要与fe.conf中配置路径保持一致:

```bash
mkdir -p meta 
```

<br/>

第三步: 启动FE进程:

```bash
bin/start_fe.sh --daemon
```

<br/>

第四步: 确认启动FE启动成功。

* 查看日志log/fe.log确认。

```Plain Text
2020-03-16 20:32:14,686 INFO 1 [FeServer.start():46] thrift server started.

2020-03-16 20:32:14,696 INFO 1 [NMysqlServer.start():71] Open mysql server success on 9030

2020-03-16 20:32:14,696 INFO 1 [QeService.start():60] QE service start.

2020-03-16 20:32:14,825 INFO 76 [HttpServer$HttpServerThread.run():210] HttpServer started with port 8030

...
```

* 如果FE启动失败，可能是由于端口号被占用，可修改配置文件conf/fe.conf中的端口号http_port。
* 使用jps命令查看java进程确认"StarRocksFe"存在。
* 使用浏览器访问FE ip:8030， 打开StarRocks的WebUI， 用户名为root， 密码为空。

### 使用MySQL客户端访问FE

第一步: 安装mysql客户端，版本建议5.5+(如果已经安装，可忽略此步)：

Ubuntu：sudo apt-get install mysql-client

Centos：sudo yum install mysql-client
<br/>

第二步: FE进程启动后，使用mysql客户端连接FE实例：

```sql
mysql -h 127.0.0.1 -P9030 -uroot
```

注意：这里默认root用户密码为空，端口为fe/conf/fe.conf中的query\_port配置项，默认为9030
<br/>

第三步: 查看FE状态：

```Plain Text
mysql> SHOW PROC '/frontends'\G

************************* 1. row ************************
             Name: 172.16.139.24_9010_1594200991015
               IP: 172.16.139.24
         HostName: starrocks-sandbox01
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: FOLLOWER
         IsMaster: true
        ClusterId: 861797858
             Join: true
            Alive: true
ReplayedJournalId: 64
    LastHeartbeat: 2020-03-23 20:15:07
         IsHelper: true
           ErrMsg:
1 row in set (0.03 sec)
```

<br/>

Role为FOLLOWER说明这是一个能参与选主的FE；IsMaster为true，说明该FE当前为主节点。
<br/>

如果MySQL客户端连接不成功，请查看log/fe.warn.log日志文件，确认问题。由于是初次启动，如果在操作过程中遇到任何意外问题，都可以删除并重新创建FE的元数据目录，再从头开始操作。
<br/>

## 部署BE

### BE的基本配置

BE的配置文件为StarRocks-XX-1.0.0/be/conf/be.conf，默认配置即可启动集群，生产环境可参考[BE参数配置](../administration/Configuration.md#be-%E9%85%8D%E7%BD%AE%E9%A1%B9)对集群进行详细优化配置。

### BE部署

通过以下命令启动be并添加be到StarRocks集群， 一般至少在三个节点部署3个BE实例， 每个实例的添加步骤相同。

第一步: 创建数据目录（当前设置为be.conf中默认storage_root_path配置项路径）：

```shell
# 进入be的安装目录
cd StarRocks-XX-1.0.0/be
# 创建数据存储目录
mkdir -p storage
```

<br/>

第二步: 通过mysql客户端添加BE节点：

* host为与priority_networks设置相匹配的IP，port为BE配置文件中的heartbeat_service_port，默认为9050。

```sql
mysql> ALTER SYSTEM ADD BACKEND "host:port";
```

<br/>

如出现错误，需要删除BE节点，可通过以下命令将BE节点从集群移除，host和port与添加时一致：

```sql
mysql> ALTER SYSTEM decommission BACKEND "host:port";
```

具体参考[扩容缩容](../administration/Scale_up_down.md#be%E6%89%A9%E7%BC%A9%E5%AE%B9)。
<br/>

第三步: 启动BE：

```shell
bin/start_be.sh --daemon
```

<br/>

第四步: 查看BE状态, 确认BE就绪:

```Plain Text
mysql> SHOW PROC '/backends'\G

********************* 1. row **********************
            BackendId: 10002
              Cluster: default_cluster
                   IP: 172.16.139.24
             HostName: starrocks-sandbox01
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2020-03-23 20:19:07
        LastHeartbeat: 2020-03-23 20:34:49
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
            TabletNum: 0
     DataUsedCapacity: .000
        AvailCapacity: 327.292 GB
        TotalCapacity: 450.905 GB
              UsedPct: 27.41 %
       MaxDiskUsedPct: 27.41 %
               ErrMsg:
              Version:
1 row in set (0.01 sec)
```

<br/>

如果isAlive为true，则说明BE正常接入集群。如果BE没有正常接入集群，请查看log目录下的be.WARNING日志文件确定原因。
<br/>

如果日志中出现类似以下的信息，说明priority\_networks的配置存在问题。

```Plain Text
W0708 17:16:27.308156 11473 heartbeat\_server.cpp:82\] backend ip saved in master does not equal to backend local ip127.0.0.1 vs. 172.16.179.26
```

<br/>

此时需要，先用以下命令drop掉原来加进去的be，然后重新以正确的IP添加BE。

```sql
mysql> ALTER SYSTEM DROPP BACKEND "172.16.139.24:9050";
```

<br/>

由于是初次启动，如果在操作过程中遇到任何意外问题，都可以删除并重新创建storage目录，再从头开始操作。

## 部署Broker

配置文件为apache\_hdfs\_broker/conf/apache\_hdfs\_broker.conf

> 注意：Broker没有也不需要priority\_networks参数，Broker的服务默认绑定在0.0.0.0上，只需要在ADD BROKER时，填写正确可访问的Broker IP即可。

如果有特殊的hdfs配置，复制线上的hdfs-site.xml到conf目录下

启动broker：

```shell
./apache_hdfs_broker/bin/start_broker.sh --daemon
```

添加broker节点到集群中：

```sql
MySQL> ALTER SYSTEM ADD BROKER broker1 "172.16.139.24:8000";
```

查看broker状态：

```plain text
MySQL> SHOW PROC "/brokers"\G
*************************** 1. row ***************************
          Name: broker1
            IP: 172.16.139.24
          Port: 8000
         Alive: true
 LastStartTime: 2020-04-01 19:08:35
LastUpdateTime: 2020-04-01 19:08:45
        ErrMsg: 
1 row in set (0.00 sec)
```

Alive为true代表状态正常。

</br>

## FE的高可用集群部署

StarRocks FE支持HA模型部署，保证集群的高可用，详细设置方式请参考[FE 高可用集群部署](/administration/Deployment.md#FE高可用部署)

</br>

## 集群升级

StarRocks支持平滑升级，并支持回滚（1.18.2及以后的版本无法回滚到其之前的版本），详细操作步骤请参考[集群升级](../administration/Cluster_administration.md#集群升级)。

</br>

## 参数设置

* **Swappiness**

关闭交换区，消除交换内存到虚拟内存时对性能的扰动。

```shell
echo 0 | sudo tee /proc/sys/vm/swappiness
```

* **Overcommit**

建议使用Overcommit，把 cat /proc/sys/vm/overcommit_memory 设成  1。

```shell
echo 1 | sudo tee /proc/sys/vm/overcommit_memory
```
