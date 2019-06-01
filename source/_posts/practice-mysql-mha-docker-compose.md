title: 用Docker搭建MySQL MHA高可用集群
date: 2017-10-13 15:39:06
tags: 
    - docker
    - mysql
    - mha4mysql
    - practice
categories:
    - 实践
---

[MySQL MHA（Master High Availability）](http://yoshinorimatsunobu.blogspot.ca/2011/07/announcing-mysql-mha-mysql-master-high.html)是目前相对成熟的一个MySQL高可用解决方案。
这篇文章主要介绍用Docker Compose编排MySQL MHA高可用集群的实践过程。
Docker Compose编排中使用了两个现有镜像，分别是[breeze2/mha4mysql-manager](https://hub.docker.com/r/breeze2/mha4mysql-manager/)和[breeze2/mha4mysql-node](https://hub.docker.com/r/breeze2/mha4mysql-node/)，这两个镜像的相关信息可以查看[《制作mha4mysql的Docker镜像》](https://blog.breezelin.site/2017/10/13/practice-make-mha4mysql-docker-image/)。
本次实践源码上传在GitHub的[breeze2/mysql-mha-docker](https://github.com/breeze2/mysql-mha-docker)。

<!--more-->

## Docker Compose编排
这个编排主要实现一主一备一从的MySQL MHA高可用集群。
### 目录结构
```cmd
L--mysql-mha-docker                //主目录
    L--scripts                     //本地（Docker宿主）使用的一些脚本
        L--mha_check_repl.sh
        L--...
    L--services                    //需要build的服务（目前是空）
    L--volumes                     //各个容器的挂载数据卷
        L--mha_manager
        L--mha_node0
        L--mha_node1
        L--mha_node2
        L--mha_share               //各个容器共享的目录
          L--scripts               //各个容器共用的一些脚本
          L--sshkeys               //各个容器的ssh public key
    L--parameters.env              //账号密码等环境参数
    L--docker-compose.yml          //编排配置

```

### docker-compose.yml
```yml
version: "2"
services:

  master:
    image: breeze2/mha4mysql-node:0.57
    container_name: mha_node0
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.5.0.10
    ports:
      - "33060:3306"
    volumes:
      - "./volumes/mha_share/:/root/mha_share/"
      - "./volumes/mha_node0/lib/:/var/lib/mysql/"
      - "./volumes/mha_node0/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=mha_node0

  slave1:
    image: breeze2/mha4mysql-node:0.57
    container_name: mha_node1
    restart: always
    depends_on:
      - master
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.5.0.11
    ports:
      - "33061:3306"
    volumes:
      - "./volumes/mha_share/:/root/mha_share/"
      - "./volumes/mha_node1/lib/:/var/lib/mysql/"
      - "./volumes/mha_node1/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=mha_node1
  slave2:
    image: breeze2/mha4mysql-node:0.57
    container_name: mha_node2
    depends_on:
      - master
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.5.0.12
    ports:
      - "33062:3306"
    volumes:
      - "./volumes/mha_share/:/root/mha_share/"
      - "./volumes/mha_node2/lib/:/var/lib/mysql/"
      - "./volumes/mha_node2/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=mha_node2

  manager:
    image: breeze2/mha4mysql-manager:0.57
    container_name: mha_manager
    depends_on:
      - master
      - slave1
      - slave2
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.5.0.9
    volumes:
      - "./volumes/mha_share/:/root/mha_share/"
      - "./volumes/mha_manager/conf:/etc/mha"
      - "./volumes/mha_manager/work:/usr/local/mha"
    entrypoint: "tailf /dev/null"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=mha_manager
networks:
  net1:
    driver: bridge
    ipam:
      config:
        - subnet: 10.5.0.0/16
          gateway: 10.5.0.1

```

这里配置了四个容器服务，一个`breeze2/mha4mysql-manager`，负责管理各个数据库结点；三个`breeze2/mha4mysql-node`，其中一个是主库，一个是备用主库（兼作从库），还有一个是（纯）从库。每个容器服务都指定了静态IP，即使服务重启也不会出现IP错乱问题。

### 环境参数
parameters.env
```env
ROOT_PASSWORD=123456
MYSQL_ROOT_PASSWORD=123456
MYSQL_DATABASE=testing
MYSQL_User=testing
MYSQL_PASSWORD=testing
MHA_SHARE_SCRIPTS_PATH=/root/mha_share/scripts
MHA_SHARE_SSHKEYS_PATH=/root/mha_share/sshkeys

```

### 数据库配置
这里简单的配置一下各个数据库数据复制备份相关的参数
主库，mha_node0/conf/my.cnf
```cnf
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-do-db=testing
binlog-ignore-db=mysql
replicate-do-db=testing
replicate-ignore-db=mysql
auto_increment_increment=2
auto_increment_offset=1
expire_logs_days=7
```

备库，mha_node1/conf/my.cnf
```cnf
[mysqld]
server-id=2
log-bin=mysql-bin
binlog-do-db=testing
binlog-ignore-db=mysql
replicate-do-db=testing
replicate-ignore-db=mysql
auto_increment_increment=2
auto_increment_offset=2
expire_logs_days=7
```

从库，mha_node2/conf/my.cnf
```cnf
[mysqld]
server-id=3
replicate-do-db=testing
replicate-ignore-db=mysql
expire_logs_days=7
```

### MHA配置
manager/conf/app1.conf
```
[server default]
user=root
password=123456
ssh_user=root

manager_workdir=/usr/local/mha
remote_workdir=/usr/local/mha

repl_user=myslave
repl_password=myslave

[server0]
hostname=10.5.0.10

[server1]
hostname=10.5.0.11

[server2]
hostname=10.5.0.12
```

## 实际运行
在主目录下执行`docker-compose up -d`构建并运行整个Docker服务。

### SSH设置
MHA要求各个主机能够相互SSH登录，所以整体服务首次启动成功后，在主目录下先执行一些命令：
```cmd
$ sh ./scripts/ssh_start.sh
$ sh ./scripts/ssh_share.sh
```

`ssh_start.sh`作用是在各个容器上开启SSH服务，`ssh_share.sh`作用是在容器内生成SSH公密钥，再把公钥共享到其他容器。常用命令调用的脚本在`scripts`和`volumes/mha_share/scripts`下，这些脚本都很简单，一看就明。
若是整体服务重新启动，只需重新开启SSH服务即可：
```cmd
$ sh ./scripts/ssh_start.sh
```

在manager容器上检测SSH是否配置成功：
```cmd
$ docker exec -it mha_manager /bin/bash
root@mha_manager# masterha_check_ssh --conf=/etc/mha/app1.conf
```
若是成功，会显示
```cmd
Mon Oct 16 14:53:59 2017 - [debug]   ok.
Mon Oct 16 14:53:59 2017 - [info] All SSH connection tests passed successfully.
```

### 开启数据主从复制

首先对主库、备考创建和授权复制账号，对备库、从库设置主库信息和开始复制，以下命令会完成这些操作：
```cmd
$ sh ./scripts/mysql_set_mbs.sh
```
可以在主库上的testing数据库里创建一张表，写入一些数据，看看备库、从库会不会同步。

在manager容器上检测REPL是否配置成功：
```cmd
$ docker exec -it mha_manager /bin/bash
root@mha_manager# masterha_check_repl --conf=/etc/mha/app1.conf
```
若是成功，会显示
```cmd
Mon Oct 16 15:01:35 2017 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```

### 开启MHA监控
SSH和REPL检测没问题后，可以在manager容器上开启MHA监控：
```cmd
$ docker exec -it mha_manager /bin/bash
root@mha_manager# masterha_manager --conf=/etc/mha/app1.conf
```

`masterha_manager`进程会一直监视主库状态是否可用，若是主库宕机，`masterha_manager`会将备库与从库的Relay Log进行比较，把最新的数据整合到备库，然后把备库提升为新主库，从库跟随复制新主库，最后`masterha_manager`进程会退出，不再监控。

我们可以在本地（Docker宿主）暂停主库（mha_node0容器）：
```cmd
$ docker pause mha_node0
```
然后，manager容器上`masterha_manager`确认主库失联后，开始切换主库，成功后会显示：
```cmd
----- Failover Report -----

app1: MySQL Master failover 10.5.0.10(10.5.0.10:3306) to 10.5.0.11(10.5.0.11:3306) succeeded

Master 10.5.0.10(10.5.0.10:3306) is down!

Check MHA Manager logs at 3d2d8185510b for details.

Started automated(non-interactive) failover.
The latest slave 10.5.0.11(10.5.0.11:3306) has all relay logs for recovery.
Selected 10.5.0.11(10.5.0.11:3306) as a new master.
10.5.0.11(10.5.0.11:3306): OK: Applying all logs succeeded.
10.5.0.12(10.5.0.12:3306): This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
10.5.0.12(10.5.0.12:3306): OK: Applying all logs succeeded. Slave started, replicating from 10.5.0.11(10.5.0.11:3306)
10.5.0.11(10.5.0.11:3306): Resetting slave info succeeded.
Master failover to 10.5.0.11(10.5.0.11:3306) completed successfully.
```
可以到从库（mha_node2容器），查看复制状态，可以看到跟随主库是10.5.0.11，即新主库（原备库）：
```
$ docker exec -it mha_manager /bin/bash
root@mha_manager# mysql -u root -p$MYSQL_ROOT_PASSWORD
mysql> show slave status;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.5.0.11
                  Master_User: myslave
```

## 最后
在数据库服务器集群中，使用MHA，可以在主库宕机后快速切换新主库，可是应用服务器并不知道主库已经被切换。所以，`masterha_manager`进程切换主库成功后应该通知应用服务器新的主库IP，或者使用虚拟IP静默过渡。若是应用与数据库通过中间件（比如ProxySQL）来连接的，那么只需在中间件上修改主库IP，对应用影响不大。

> 注意：因为本compose中MySQL服务与SSH服务没有同一启动（SSH服务是容器启动后才开启的），所以本compose只能当作线下模拟练习，未能在正式线上应用。比如其中某个结点容器重启，SSH服务并没有自动重启，进而整个MHA集群不可用。


