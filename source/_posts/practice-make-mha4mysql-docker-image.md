title: 制作mha4mysql的Docker镜像
date: 2017-10-13 10:16:08
tags: 
    - docker
    - mysql
    - mha4mysql
    - practice
categories:
    - 实践
---

> 本来想记录一下用Docker构建MySQL MHA集群的实践过程的，但是整个篇幅有点长，所以这里先来介绍一下怎样制作mha4mysql的Docker镜像。

mha4mysql是日本工程师[Yoshinori Matsunobu](https://www.percona.com/live/mysql-conference-2014/users/yoshinori-matsunobu)开发的一款MySQL高可用软件。mha4mysql分为两部分，一是管理器部分[mha4mysql-manager](https://github.com/yoshinorim/mha4mysql-manager)，二是结点部分[mha4mysql-node](https://github.com/yoshinorim/mha4mysql-node)。mha4mysql-node要运行在每台受管理的MySQL服务器上；而mha4mysql-manager所在服务器则不需要MySQL，但需要mha4mysql-node。因为mha4mysql-manager依赖mha4mysql-node，即安装mha4mysql-manager前必须先安装mha4mysql-node。

下面讲解一下，基于[debian:jessie](https://hub.docker.com/_/debian/)制作mha4mysql-manager的Docker镜像和
基于[mysql:5.7](https://hub.docker.com/_/mysql/)制作mha4mysql-node的Docker镜像。

<!--more-->

## mha4mysql-manager
mha4mysql-manager的Dockerfile:
```dockerfile
FROM debian:jessie

COPY ./mha4mysql-manager.tar.gz /tmp/
COPY ./mha4mysql-node.tar.gz /tmp/

RUN build_deps='ssh sshpass perl libdbi-perl libmodule-install-perl libdbd-mysql-perl libconfig-tiny-perl liblog-dispatch-perl libparallel-forkmanager-perl make' \
    && apt-get update \
    && apt-get -y --force-yes install $build_deps \
    && tar -zxf /tmp/mha4mysql-node.tar.gz -C /opt \
    && cd /opt/mha4mysql-node \
    && perl Makefile.PL \
    && make \
    && make install \
    && tar -zxf /tmp/mha4mysql-manager.tar.gz -C /opt \
    && cd /opt/mha4mysql-manager \
    && perl Makefile.PL \
    && make \
    && make install \
    && cd /opt \
    && rm -rf /opt/mha4mysql-* \
    && apt-get clean

```
注释：
1. 基于debian:jessie镜像二次制作；
2. 将mha4mysql-manager和mha4mysql-node的当前最新版本（v0.57）打包，复制到镜像内；
3. `build_deps`是mha4mysql-manager和mha4mysql-node的安装依赖、运行依赖；
4. 先拆包安装mha4mysql-node，安装命令：`perl Makefile.PL && make && make install`；
5. 才能拆包安装mha4mysql-manager，安装命令：`perl Makefile.PL && make && make install`；
6. 清理一些无用文件。

## mha4mysql-node
mha4mysql-node的Dockerfile:
```dockerfile
FROM mysql:5.7

COPY ./mha4mysql-node.tar.gz /tmp/

RUN build_deps='ssh sshpass perl libdbi-perl libmodule-install-perl libdbd-mysql-perl make' \
    && apt-get update \
    && apt-get -y --force-yes install $build_deps \
    && tar -zxf /tmp/mha4mysql-node.tar.gz -C /opt \
    && cd /opt/mha4mysql-node \
    && perl Makefile.PL \
    && make \
    && make install \
    && cd /opt \
    && rm -rf /opt/mha4mysql-* \
    && apt-get clean

```
注释：
1. 基于mysql:5.7镜像二次制作；
2. 将mha4mysql-node的当前最新版本（v0.57）打包，复制到镜像内；
3. `build_deps`是mha4mysql-node的安装依赖、运行依赖；
4. 拆包安装mha4mysql-manager，安装命令：`perl Makefile.PL && make && make install`；
5. 清理一些无用文件。

## 注意

### v0.57 bug
目前mha4mysql-manager和mha4mysql-node的v0.57并不兼容，新版的libdbd-mysql-perl的调用语法，比如新版要求连接数据库格式是：
`my $dsn = "DBI:mysql:;host=$opt{host};port=$opt{port}";`
而v0.57源码是：
`my $dsn = "DBI:mysql:;host=[$opt{host}];port=$opt{port}";`
多了一对中括号会导致连接数据库失败。
这里镜像中的mha4mysql-manager和mha4mysql-node的源码包是已经对上面问题进行修正的。

### 单容器多服务
按照Docker的思想，是一个容器只做一件事，但是mha4mysql结点需要mysql服务和ssh服务，这里的临时解决办法是容器先运行mysql服务，之后执行`docker exec -it $CONTAINER_NAME /bin/bash service ssh start`。
目前实现Dokcer单容器多服务的hack方法是利用supervisor，只做supervisor这一件事，而supervisor做多件事。如果后期真的需要使用supervisor，也可以基于本镜像进行二次制作。

## 最后
mha4mysql-manager和mha4mysql-node的Dockerfile分别放在GitHub[breeze2/mha4mysql-manager-docker](https://github.com/breeze2/mha4mysql-manager-docker)和[breeze2/mha4mysql-node-docker](https://github.com/breeze2/mha4mysql-node-docker)上；镜像可以用docker命令直接拉取：
```cmd
$ docker pull breeze2/mha4mysql-manager:0.57
$ docker pull breeze2/mha4mysql-node:0.57
```


