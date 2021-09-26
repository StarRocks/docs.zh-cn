# Deploy常见问题

## 在配置文件fe.conf中priority_networks参数如何绑定固定IP

问题描述：

比如客户放有2个ip：192.168.108.23，192.168.108.43。如果写192.168.108.23/24会自动识别到43，如果写192.168.108.23/32会出错，启动后会识别到127.0.0.1。

解决方案:

* 32就不用写了，直接写ip就行；或者再写长一点28这种。

注:（如果写 32 出错，是当前版本比较低，新版本已经修复了这个问题。）

## be http_service启动失败

问题描述：

be安装过程中启动报错Doris Be http service did not start correctly,exiting

解决方案：

* 该问题是be webservice端口被占用，可以尝试修改be.conf中的相关端口并重启。如果多次修改没有被占用的端口也重复报错，检查是否装有yarn等程序，确认监听端口选择修改监听规则，或者be的端口选取范围绕过即可。

## SUSE 12SPS的 OS 是否支持

* 解答：可以支持，经过测试是没有问题的

## ERROR 1064 (HY000): Could not initialize class org.apache.doris.rpc.BackendServiceProxy

解决方案：

* 检查是否使用的是jre,如果使用的jre换成jdk即可，推荐使用oraclejdk版本1.8+。

## 【企业版部署】安装部署过程当中，在配置节点时报错：Failed to Distribute files to node

问题原因：

setuptools的版本不对

解决方案:

* 目前安装报错的信息是因为setuptools的版本不对，需要到每台机器上执行下以下命令，因为需要root权限

```palin text
yum remove python-setuptools

rm /usr/lib/python2.7/site-packages/setuptool* -rf

wget https://bootstrap.pypa.io/ez_setup.py -O - | python
```

## StarRocks能否临时修改FE、BE的配置让其不重启就能生效，生产环境不能随便重启服务

解决方案:

FE配置临时修改：

1、SQL方式：

```sql
ADMIN SET FRONTEND CONFIG ("key" = "value");

示例：
ADMIN SET FRONTEND CONFIG ("enable_statistic_collect" = "false");
```

2、命令方式：

```plain text
curl --location-trusted -u username:password http://ip:fe_http_port/api/_set_config?key=value

示例：

curl --location-trusted -u root:root  http://192.168.110.101:8030/api/_set_config?enable_statistic_collect=true
```

BE配置临时修改：

命令方式：

```plain text
curl -XPOST -u username:password http://ip:be_http_port/api/update_config?key=value

是该用户没有远程登录的权限:

CREATE USER 'test'@'%' IDENTIFIED BY '123456';
GRANT SELECT_PRIV ON . TO 'test'@'%';

创建用户test并赋权，重新登录即可。
```

## [磁盘扩容问题] BE磁盘空间不足，加盘后数据存储不能负载均衡且报错：Failed to get scan range, no queryable replica found in tablet: 11903

问题描述：

Flink导入报错，定位原因为磁盘不足，后来扩容磁盘后，不能对数据存储进行负债均衡，而是随机的。

解决方案:
目前正在修复当中，解决方法测试客户如果数据不重要推荐直接删除掉磁盘，线上客户或者重要数据推荐手工操作

 不是重要数据直接删除掉磁盘的话可能会面临一个问题就是：切换完磁盘目录后，会报错：Failed to get scan range, no queryable replica found in tablet: 11903，解决方法为：把这张表11903truncate 一下之后即可.
