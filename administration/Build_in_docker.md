# 通过源码编译 StarRocks

本文介绍如何通过 Docker 镜像编译 StarRocks。

## 前提条件

在编译 StarRocks 之前，请确保您已安装 [Docker](https://www.docker.com/get-started/)。

## 下载镜像

从 Docker Hub 下载开发环境的镜像文件。该镜像中集成了 LLVM 及 Clang 作为第三方工具。

```shell
docker pull starrocks/dev-env:{version}
```

> 说明：请使用下表中相应的镜像版本号替换命令中的 `{version}`。

StarRocks 版本分支与开发环境镜像版本的对应关系如下所示：

|StarRocks 版本分支|开发环境镜像版本|
|-----------------|---------------|
|main|starrocks/dev-env:main|

## 编译 StarRocks

您可以通过挂载本地存储（推荐）或拷贝 GitHub 代码库的方式编译 StarRocks。

- 挂载本地存储编译 StarRocks。

  ```shell
  docker run -it \
  -v /{local-path}/.m2:/root/.m2 \
  -v /{local-path}/starrocks:/root/starrocks \
  --name {container-name} \
  -d starrocks/dev-env:{version}
  
  docker exec -it {container-name} /root/starrocks/build.sh
  ```

  > 注意：请避免在 Docker 容器中重复下载 **.m2** 内的 Java 依赖。您无需从 Docker 容器中复制 **starrocks/output** 内已编译好的二进制包。

- 拷贝代码库编译 StarRocks。

  ```shell
  docker run -it --name {container-name} -d starrocks/dev-env:{version}
  
  docker exec -it {container-name} /bin/bash
  
  # 在 Docker 容器内任意路径下拷贝 StarRocks 代码库。
  git clone https://github.com/StarRocks/starrocks.git
  
  cd starrocks
  
  ./build.sh
  ```
