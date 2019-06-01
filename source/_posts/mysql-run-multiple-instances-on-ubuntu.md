title: 在Ubuntu上运行多实例MySQL
date: 2018-03-18 19:02:13
tags: 
    - mysql
categories:
    - 躬行
---

一般情况下，都是一台服务器主机运行一个MySQL实例；不过在单机配置非常高的情况下，也可以考虑运行多个MySQL实例。毕竟MySQL是单进程（多线程）的运行模式，单实例MySQL用不上太多资源。

当然，高配置的单机应该考虑使用[Docker](https://www.docker.com/)实现多MySQL服务，只是这里介绍一下如何运行多实例MySQL。本以为在Ubuntu上实现很容易，原来也并不简单。

## 运行环境
* Ubuntu 16.04
* MySQL 5.17

<!--more-->

## mysqld_multi
在安装好MySQL Server后，应该是可以使用`mysqld_multi`的。
查看配置示例命令：
```cmd
$ mysqld_multi --example
[mysqld_multi]
mysqld     = /usr/bin/mysqld_safe
mysqladmin = /usr/bin/mysqladmin
user       = multi_admin
password   = my_password

[mysqld2]
socket     = /tmp/mysql.sock2
port       = 3307
pid-file   = /var/lib/mysql2/hostname.pid2
datadir    = /var/lib/mysql2
language   = /usr/share/mysql/mysql/english
user       = unix_user1
...
```

一般MySQL的配置文件是`/etc/mysql/my.cnf`或者`/etc/mysql/mysql.cnf`，在配置文件中最开始的地方添加：
```cnf
[mysqld_multi]
mysqld     = /usr/bin/mysqld_safe
mysqladmin = /usr/bin/mysqladmin

```

查看当前`mysqld_multi`状态：
```
$ mysqld_multi report                               
Reporting MySQL servers
No groups to be reported (check your GNRs)
```

可以看到没有服务组群，现在添加一个——在MySQL的配置文件中，追加：
```cnf
[mysqld3307]
socket     = /tmp/mysqld/mysqld3307.sock
port       = 3307
pid-file   = /tmp/mysqld/mysqld3307.pid
datadir    = /var/lib/mysql/multi/data3307
user       = mysql
```

再查看当前`mysqld_multi`状态：
```
$ mysqld_multi report                               
Reporting MySQL servers
MySQL server from group: mysqld3307 is not running
```
可以看到服务组群`mysqld3307`（注意每个服务组群名称必须以`mysqld`开头）没有在运行。现在还不能直接运行`mysqld3307`服务，因为没有初始化数据目录`/var/lib/mysql/multi/data3307`。

## 初始化数据库
> 过去MySQL初始化数据库使用的是`mysql_install_db`命令，现在是`mysqld --initialize`。

在Ubuntu系统上有一个系统安全程序，叫AppArmor，它会限制各个应用程序访问系统资源的权限。Root用户运行程序时也碰到权限不足错误的话，那么应该就是AppArmor限制了。比如我们需要修改`mysqld`的权限，就得编辑对应的AppArmor配置文件，然后重启AppArmor服务：
```cmd
$ sudo vi /etc/apparmor.d/usr.sbin.mysqld
$ sudo service apparmor restart 
```

上面服务组群`mysqld3307`的配置中用到的目录是`/tmp/mysqld`和`/var/lib/mysql/multi`，`mysqld`程序对`/tmp`和`/var/lib/mysql`都是具有读写权限的，所以不需要修改AppArmor配置文件。

现在来初始化一个新的数据库目录`/var/lib/mysql/multi/data3307`（以mysql用户身份执行）：
```cmd
$ sudo -u mysql mkdir /tmp/mysqld
$ sudo -u mysql mkdir /var/lib/mysql/multi
$ sudo -u mysql mysqld --initialize --user=mysql --datadir=/var/lib/mysql/multi/data3307
```

初始化成功后，新数据库会自动建立root用户，并随机生成密码，这些会记录在MySQL日志中：
```cmd
$ sudo tail /var/log/mysql/error.log
2018-03-18T12:15:25.378684Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2018-03-18T12:15:26.642564Z 0 [Warning] InnoDB: New log files created, LSN=45790
2018-03-18T12:15:26.900383Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2018-03-18T12:15:26.986706Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 3544d1ec-2b6f-11e8-bf0a-aaaa0007ba64.
2018-03-18T12:15:26.998294Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2018-03-18T12:15:26.998765Z 1 [Note] A temporary password is generated for root@localhost: 4kDUVrlO>zT)
```
我们先记住密码`4kDUVrlO>zT)`。

## 启动新实例
执行一下命令即可以mysql用户身份启动服务组群`mysqld3307`（不需要前缀`mysqld`）：
```cmd
$ sudo -u mysql mysqld_multi start 3307
```

查看当前`mysqld_multi`状态：
```
$ mysqld_multi report                               
Reporting MySQL servers
MySQL server from group: mysqld3307 is running
```

重置一下密码：
```cmd
$ mysqladmin -uroot -h 127.0.0.1 -P 3307 -p'4kDUVrlO>zT)' password
New password:
Confirm new password:
```

连接服务组群`mysqld3307`：
```cmd
$ mysql -uroot -h 127.0.0.1 -P 3307 -p
```

## 关闭实例
原以为关闭服务组群`mysqld3307`，只需要：
```cmd
$ sudo -u mysql mysqld_multi stop 3307
```

结果查看`mysqld_multi`状态，`mysqld3307`依然是运行中：
```
$ mysqld_multi report                               
Reporting MySQL servers
MySQL server from group: mysqld3307 is running
```

> 可能是因为`mysqld3307`数据库中没有与`mysqld_multi`配置对应的用户，所以`mysqld_multi stop`命令没能关闭`mysqld3307`。

我们可以用`mysqladmin`命令来关闭服务组群`mysqld3307`：
```cmd
$ mysqladmin -uroot -h 127.0.0.1 -P 3307 -p shutdown
```

## 最后
这里只介绍了监听3307端口的MySQL实例的建立过程，其他更多MySQL实例，如监听3308、3309端口，只需按照上述流程（添加配置，初始化数据目录，启动服务）执行即可。