title: 用Docker实现MySQL ProxySQL读写分离
date: 2017-10-10 10:26:13
tags: 
    - docker
    - mysql
    - proxysql
    - practice
categories:
    - 实践
---

[ProxySQL](http://www.proxysql.com/)是一个高性能的MySQL中间件，能够代理数百台MySQL服务器，支持数十万的并发连接。ProxySQL代理MySQL并提供读写分离，查询重写和数据分片等功能。
这篇文章主要介绍用Docker Compose编排用ProxySQL实现MySQL集群上读写分离的实践过程。
Docker Compose编排中使用了一个现有镜像，就是[breeze2/proxysql](https://hub.docker.com/r/breeze2/proxysql/)。
本次实践源码上传在GitHub的[breeze2/mysql-proxy-docker](https://github.com/breeze2/mysql-proxy-docker)。

<!--more-->

## Docker Compose编排
这个编排主要实现一主两从的一个MySQL集群和一个ProxySQL代理，ProxySQL代理MYSQL集群的数据请求并且进行读写分离。
### 目录结构
```cmd
L--mysql-proxy-docker                    //主目录
    L--scripts                           //本地（Docker宿主）使用的一些脚本
        L--mysql_set_users_and_repls.sh  //设置各个数据库账号和开启主从复制
        L--...
    L--services                          //需要build的服务（目前是空）
    L--volumes                           //各个容器的挂载数据卷
        L--mysql_node0
        L--mysql_node1
        L--mysql_node2
        L--proxysql
        L--share                         //各个容器共享的目录
          L--scripts                     //各个容器共用的一些脚本
    L--parameters.env                    //账号密码等环境参数
    L--docker-compose.yml                //编排配置

```

### docker-compose.yml
```yml
version: "2"
services:

  master:
    image: mysql:5.7
    container_name: mysql_node0
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.6.0.10
    ports:
      - "3306"
    volumes:
      - "./volumes/share/:/root/share/"
      - "./volumes/mysql_node0/lib/:/var/lib/mysql/"
      - "./volumes/mysql_node0/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env

  slave1:
    image: mysql:5.7
    container_name: mysql_node1
    restart: always
    depends_on:
      - master
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.6.0.11
    ports:
      - "3306"
    volumes:
      - "./volumes/share/:/root/share/"
      - "./volumes/mysql_node1/lib/:/var/lib/mysql/"
      - "./volumes/mysql_node1/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env
  slave2:
    image: mysql:5.7
    container_name: mysql_node2
    depends_on:
      - master
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.6.0.12
    ports:
      - "3306"
    volumes:
      - "./volumes/share/:/root/share/"
      - "./volumes/mysql_node2/lib/:/var/lib/mysql/"
      - "./volumes/mysql_node2/conf/:/etc/mysql/conf.d/"
    env_file:
      - ./parameters.env

  proxy:
    image: breeze2/proxysql:1.4.3
    container_name: proxysql
    depends_on:
      - master
      - slave1
      - slave2
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.6.0.9
    ports:
      - "127.0.0.1:60320:6032"
      - "60330:6033"
    volumes:
      - "./volumes/proxysql/conf:/etc/proxysql"
    entrypoint: "proxysql -f -c /etc/proxysql/pr.cnf"
    env_file:
      - ./parameters.env

networks:
  net1:
    driver: bridge
    ipam:
      config:
        - subnet: 10.6.0.0/16
          gateway: 10.6.0.1

```

这里配置了四个容器服务，一个`breeze2/proxysql`，负责代理各个数据库；三个`mysql`，其中一个是主库，另外两个是从库。每个容器服务都指定了静态IP，即使服务重启也不会出现IP错乱问题。`proxysql`容器的`6032`是提供管理服务的端口，只对Docker宿主机本地IP开放，而`6033`是代理数据请求的端口，可以对Docker宿主机网络IP开放。

### 环境参数
parameters.env
```env
MYSQL_ROOT_PASSWORD=123456
MYSQL_DATABASE=testing
MYSQL_User=testing
MYSQL_PASSWORD=testing

```

### 数据库配置
这里简单的配置一下各个数据库数据复制备份相关的参数
主库，mysql_node0/conf/my.cnf
```cnf
[mysqld]
server-id=1
gtid-mode=on
enforce-gtid-consistency=true
log-bin=mysql-bin
binlog-do-db=testing
binlog-ignore-db=mysql
replicate-do-db=testing
replicate-ignore-db=mysql
expire_logs_days=7
```

从库，mysql_node1/conf/my.cnf
```cnf
[mysqld]
server-id=2
gtid-mode=on
enforce-gtid-consistency=true
log-bin=mysql-bin
binlog-do-db=testing
binlog-ignore-db=mysql
replicate-do-db=testing
replicate-ignore-db=mysql
expire_logs_days=7
```

从库，mysql_node2/conf/my.cnf
```cnf
[mysqld]
server-id=3
gtid-mode=on
enforce-gtid-consistency=true
log-bin=mysql-bin
binlog-do-db=testing
binlog-ignore-db=mysql
replicate-do-db=testing
replicate-ignore-db=mysql
expire_logs_days=7
```

### ProxySQL配置
proxysql/conf/pr.cnf
```cnf
datadir="/tmp"
# 管理平台参数
admin_variables =
{
  admin_credentials="admin2:admin2"
  mysql_ifaces="0.0.0.0:6032"
  refresh_interval=2000
}
# mysql全局参数
mysql_variables =
{
  threads=4
  max_connections=2048
  default_query_delay=0
  default_query_timeout=36000000
  have_compress=true
  poll_timeout=2000
  # interfaces="0.0.0.0:6033;/tmp/proxysql.sock"
  interfaces="0.0.0.0:6033"
  default_schema="information_schema"
  stacksize=1048576
  server_version="5.5.30"
  connect_timeout_server=3000
  # make sure to configure monitor username and password
  # https://github.com/sysown/proxysql/wiki/Global-variables#mysql-monitor_username-mysql-monitor_password
  monitor_username="pr_muser"
  monitor_password="pr_mpass"
  monitor_history=600000
  monitor_connect_interval=60000
  monitor_ping_interval=10000
  monitor_read_only_interval=1500
  monitor_read_only_timeout=500
  ping_interval_server_msec=120000
  ping_timeout_server=500
  commands_stats=true
  sessions_sort=true
  connect_retries_on_failure=10
}
# mysql用户参数
mysql_users = 
(
  {
    username = "pr_auser"
    password = "pr_apass"
    default_hostgroup = 0
  }
)
# mysql服务器参数，10.6.0.10是主库放在0组，其他是从库放在1组
mysql_servers =
(
  {
    address = "10.6.0.10"
    port = 3306
    weight = 1
    hostgroup = 0
    max_connections = 50
  },
  {
    address = "10.6.0.11"
    port = 3306
    weight = 2
    hostgroup = 1
    max_connections = 100
  },
  {
    address = "10.6.0.12"
    port = 3306
    weight = 2
    hostgroup = 1
    max_connections = 150
  }
)
# mysql请求规则，以下配置是读时加锁的请求发给0组，普通读取的请求发给1组，其他默认发给0组（上面的default_hostgroup）
mysql_query_rules:
(
  {
    rule_id=1
    active=1
    match_pattern="^SELECT .* FOR UPDATE$"
    destination_hostgroup=0
    apply=1
  },
  {
    rule_id=2
    active=1
    match_pattern="^SELECT"
    destination_hostgroup=1
    apply=1
  }
)
```

## 实际运行
在主目录下执行`docker-compose up -d`构建并运行整个Docker服务。

### 开启主从复制
在主目录下执行：
```cmd
$ sh ./scripts/mysql_set_users_and_repls.sh
```
实际上是调用了挂载在数据库容器里的一些脚本：
```
volumes/share/scripts/msyql_grant_proxysql_users.sh #设置给proxysql用的账号，pr_auser和pr_muser
volumes/share/scripts/msyql_grant_slave.sh #主库设置给从库用的账号
volumes/share/scripts/msyql_grant_slave.sh #从库开始数据复制
```
脚本里的执行命令都很简单，一看就明。

### 开启ProxySQL代理
构建整个服务的时候，`proxysql`会先挂载主目录下的`./volumes/proxysql/conf/pr.cnf`到容器内`/etc/proxysql/pr.cnf`，然后执行`proxysql -f -c /etc/proxysql/pr.cnf`，所以这里的ProxySQL是按照`pr.cnf`里面的配置开启MySQL代理服务的，请仔细阅读上面**ProxySQL配置**。若有需要在ProxySQL运行的过程中修改配置，可以登录ProxySQL的管理系统操作。

### ProxySQL管理系统
在Docker宿主机上登录ProxySQL管理系统（Docker宿主机需要装有MySQL Client）：
```cmd
$ mysql -u admin2 -padmin2 -h 127.0.0.1 -P60320
```

在ProxySQL管理系统上添加一个mysql_user（注意这个testing账号是各个数据库都已建立的，具体查看上面**环境参数**）：
```cmd
mysql> INSERT INTO mysql_users(username, password, default_hostgroup) VALUES ('testing', 'testing', 2);
```
确认是否已添加：
```cmd
mysql> SELECT * FROM mysql_users;
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
| username | password | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
| pr_auser | pr_apass | 1      | 0       | 0                 |                | 0             | 0                      | 0            | 1       | 1        | 10000           |
| testing  | testing  | 1      | 0       | 2                 | NULL           | 0             | 1                      | 0            | 1       | 1        | 10000           |
+----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+
2 rows in set (0.00 sec)
```

把当前修改（MEMORY层）加载到正在运行的ProxySQL（RUNTIME层）：
```cmd
mysql> LOAD MYSQL USERS TO RUNTIME; 
```
在Docker宿主机上确认ProxySQL是否已加载最新配置：
```cmd
$ mysql -u testing -ptesting -h 127.0.0.1 -P60330
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.5.30 (ProxySQL Admin Module)

...
```
若想ProxySQL重启后依然是当前配置，要把当前修改（MEMORY层）保存到ProxySQL的Sqlite数据库里（DISK层）：
```cmd
mysql> SAVE MYSQL USERS TO DISK; 
```

ProxySQL配置系统分三层，分别是MEMORY层、RUNTIME层和DISK层。ProxySQL管理系统操作的是MEMORY层，当前ProxySQL运行的是RUNTIME层，保存在ProxySQL本地Sqlite数据库里的是DISK层，详情请阅读文档[ProxySQL Configuration](https://github.com/sysown/proxysql/wiki/ProxySQL-Configuration)。

### SysBench测试工具
[SysBench](https://github.com/akopytov/sysbench)是一个脚本化、多线程的基准测试工具，经常用于评估测试各种不同系统参数下的数据库负载情况。
SysBench的使用教程可以参考[sysbench 0.5使用手册](http://mingxinglai.com/cn/2013/07/sysbench/)。
这里使用SysBench-v1.0.9来对ProxySQL进行测试。

#### SysBench Test Prepare
首先，做测试准备：
```
$ sysbench /usr/local/Cellar/sysbench/1.0.9/share/sysbench/oltp_read_write.lua  --threads=5 --max-requests=0 --time=36 --db-driver=mysql --mysql-user=pr_auser --mysql-password='pr_apass' --mysql-port=60330  --mysql-host=127.0.0.1  --mysql-db=testing --report-interval=1 prepare
sysbench 1.0.9 (using bundled LuaJIT 2.1.0-beta2)

Initializing worker threads...

Creating table 'sbtest1'...
Inserting 10000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
```
注意，`/usr/local/Cellar/sysbench/1.0.9/share/sysbench/oltp_read_write.lua`文件可以在SysBench安装包里找到，执行命令后，会在主库testing数据库里生成一个sbtest1表并插入一些数据，
在从库里一样可以看到testing数据库下有sbtest1表，说明主从复制已生效。

#### SysBench Test Run
然后，开始读写测试：
```
$ sysbench /usr/local/Cellar/sysbench/1.0.9/share/sysbench/oltp_read_write.lua  --threads=5 --max-requests=0 --time=36 --db-driver=mysql --mysql-user=pr_auser --mysql-password='pr_apass' --mysql-port=60330  --mysql-host=127.0.0.1  --mysql-db=testing --report-interval=1 run
sysbench 1.0.9 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 5
Report intermediate results every 1 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 1s ] thds: 5 tps: 51.66 qps: 1087.83 (r/w/o: 769.92/209.62/108.29) lat (ms,95%): 144.97 err/s: 0.00 reconn/s: 0.00
[ 2s ] thds: 5 tps: 61.26 qps: 1229.13 (r/w/o: 862.60/243.01/123.52) lat (ms,95%): 142.39 err/s: 1.00 reconn/s: 0.00
[ 3s ] thds: 5 tps: 60.85 qps: 1237.04 (r/w/o: 867.92/247.41/121.71) lat (ms,95%): 121.08 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 5 tps: 67.07 qps: 1332.44 (r/w/o: 931.01/267.29/134.15) lat (ms,95%): 127.81 err/s: 0.00 reconn/s: 0.00
...
```

#### 查看结果

登录ProxySQL管理系统，查看统计结果：
```cmd
$  mysql -u admin2 -padmin2 -h 127.0.0.1 -P60320
mysql> select * from stats_mysql_query_digest limit\G;
*************************** 1. row ***************************
  hostgroup: 0
 schemaname: testing
   username: pr_auser
     digest: 0xE365BEB555319B9E
digest_text: DELETE FROM sbtest1 WHERE id=?
 count_star: 2564
 first_seen: 1508313300
  last_seen: 1508313336
   sum_time: 1923227
   min_time: 149
   max_time: 39773
*************************** 2. row ***************************
  hostgroup: 0
 schemaname: testing
   username: pr_auser
     digest: 0xFB239BC95A23CA36
digest_text: UPDATE sbtest1 SET c=? WHERE id=?
 count_star: 2566
 first_seen: 1508313300
  last_seen: 1508313336
   sum_time: 2016454
   min_time: 158
   max_time: 53514
...
*************************** 13. row ***************************
  hostgroup: 1
 schemaname: testing
   username: pr_auser
     digest: 0xDBF868B2AA296BC5
digest_text: SELECT SUM(k) FROM sbtest1 WHERE id BETWEEN ? AND ?
 count_star: 2570
 first_seen: 1508313300
  last_seen: 1508313336
   sum_time: 7970660
   min_time: 216
   max_time: 56153
*************************** 14. row ***************************
  hostgroup: 1
 schemaname: testing
   username: pr_auser
     digest: 0xAC80A5EA0101522E
digest_text: SELECT c FROM sbtest1 WHERE id BETWEEN ? AND ? ORDER BY c
 count_star: 2570
 first_seen: 1508313300
  last_seen: 1508313336
   sum_time: 10148202
   min_time: 272
   max_time: 58032
14 rows in set (0.00 sec)
```
可以看到读操作都发送给了hostgroup=1组，写操作都发送给了hostgroup=0组，说明读写分离已生效，读写分离配置请仔细阅读上面**ProxySQL配置mysql_query_rules部分**。

## 此致
到此，用Docker实现MySQL ProxySQL读写分离已完成。另外ProxySQL提供的查询重写功能，其实是利用mysql_query_rules配置，对接收到的查询语句进行正则替换，再传递给数据库服务器，详情请阅读文档[ProxySQL Configuration](https://github.com/sysown/proxysql/wiki/ProxySQL-Configuration)中的“MySQL Query Rules”部分；而数据分片功能，在真实数据分片的基础上，再结合mysql_query_rules配置，重写query到正确的主机、数据库或表上，详细内容可以阅读[MySQL Sharding with ProxySQL](https://www.percona.com/blog/2016/08/30/mysql-sharding-with-proxysql/)。
