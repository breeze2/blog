title: 关于LNMP的优化
date: 2017-09-10 18:13:05
tags: 
    - linux
    - nginx
    - mysql
    - php
    - think
categories:
    - 求索
---

线上有一个用LNMP构架的网站，有一天看到它出现了“502 Bad Gateway”错误，于是就开始思考优化问题了。
首先，明确一下HTTP各个状态码的含义：
1. 1XX 临时消息
2. 2XX 返回成功
3. 3XX 重新定向
4. 4XX 客户端错误
5. 5XX 服务端错误

基于LNMP的网站上，当HTTP请求返回的状态码是`5XX`的时候，说明是服务端出了问题；但问题不一定是出在NginX，因为NginX本身十分轻量，不做太多的复杂逻辑处理，所以很少会出错；除了静态资源的请求，其他大部分请求NginX都会转给PHP-FPM来处理，所以一般问题是更多地出在PHP。比如，开篇说那个网站出现的`502`错误，查看NginX日志发现大量的`connect() to unix:/PATH/TO/PHP-FPM-SOCK failed (11: Resource temporarily unavailable)`，其实原因是系统最大连接数过小。

<!--more-->

## Linux的内核优化

按照[《高性能Linux服务器构建实战：运维监控、性能调优与集群应用》](https://book.douban.com/subject/7564094/)里的介绍，配置系统参数如下：

```conf
# /etc/sysctl.conf

net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 262144
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30

```

个人保守起见，参数略有调整：

```conf
# /etc/sysctl.conf

net.core.somaxconn = 262144 # 表示系统同时发起的最大连接数，默认128
net.core.netdev_max_backlog = 262144 # 表示当每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许发送到队列的数据包的最大数目，默认1000

net.ipv4.ip_local_port_range = 32768 60999 # 表示允许系统随机打开的端口范围，默认32768 60999
net.ipv4.tcp_max_tw_buckets = 6000 # 表示系统同时保持timewait的最大数量，默认524288
net.ipv4.tcp_tw_recycle = 0 # 开启TCP连接中timewait sockets的快速回收，默认不开启
net.ipv4.tcp_tw_reuse = 1 # 开启TCP连接中timewait sockets重新用于新的TCP连接，默认不开启
net.ipv4.tcp_max_orphans = 262144 # 表示系统中不被关联到任何一个用户文件句柄上的TCP sockets的最大数目，可以防止简单的DoS攻击，默认262144
net.ipv4.tcp_max_syn_backlog = 262144 # 表示记录尚未收到客户端确认信息的连接请求的最大数目，默认128
net.ipv4.tcp_syncookies = 1 # 开启syncookies功能，可以防止部分SYN攻击，默认开启
net.ipv4.tcp_synack_retries = 1 # 表示系统放弃连接之前发送SYN+ACK包的数量
net.ipv4.tcp_syn_retries = 1 # 表示系统放弃连接之前发送SYN包的数量
net.ipv4.tcp_fin_timeout = 3 # 表示TCP sockets保持在FIN-WAIT-2状态的时间，默认60秒
net.ipv4.tcp_keepalive_time = 30 # 表示当启用keepalive的时候，TCP发送keepalive消息的频度，默认7200秒

```

以上参数主要为了配合NginX、PHP-FPM和MySQL致发挥更佳效能，各个参数的数值应该根据实际主机硬件条件自行调整。将配置参数追加到`/etc/sysctl.conf`文件后，执行以下命令，让系统重载参数：
```cmd
$ sudo sysctl -p
```

## NginX的配置优化

大致配置如下：

```conf
# /etc/nginx/nginx.conf

user www-data; # 指定用户
worker_processes 8; # woker进程数，一般设置为CPU核数*线程数
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000
01000000 10000000; # 为每个进程分配到指定的CPU，配合worker_processes使用
worker_rlimit_nofile 204800; # 每个woker能打开的最多文件描述符数目
pid /run/nginx.pid; # 进程号存放位置

events {
    use epoll; # 设置用于复用客户端线程的轮询方法
    worker_connections 204800; # 每个woker的最多连接数目，不要大于worker_rlimit_nofile
    multi_accept on; # 启用收到一个新连接通知后接受尽可能多的连接
}

http {
    keepalive_timeout 30; #
    server {
        listen 80 backlog=204800; #
    }
}

```

实际上NginX的`worker_connections`参数会受到系统的`net.core.somaxconn`参数限制，而`net.core.somaxconn`参数也是有限制的（可以用`ulimit -Hn`命令查看），理论上`worker_connections`应该设为`net.core.somaxconn`除以`worker_processes`。
在配合PHP-FPM使用的时候，如果想NginX支持1000并发请求，那么`worker_connections`取值不能低于2000，因为每个请求里，NginX与客户端需要一个连接，与PHP-FPM也需要一个连接。

## PHP-FPM的配置优化

### Unix socket VS TCP socket
PHP-FPM提供两种通信方式与NginX对接数据，一种是Unix socket，另一种是TCP socket。
Unix socket是同一操作系统上的两个或多个进程进行数据通信的编程接口，这种通信方式是发生在系统内核里而不会在网络里传播；TCP socket则是基于TCP/IP协议，进程间的通信是通过网络传输的，可以跨系统、跨主机。
两者相较之下，Unix socket更低层，执行效率更高；而TCP socket封装性更高，安全性更好。如果NginX和PHP-FPM不在同一主机上，那么只能选择TCP socket；否则，建议考虑Unix socket。

### 最大进程数

假设一个PHP-FPM进程占用10MB内存，且打算分配1GB内存给PHP-FPM用，那么PHP-FPM的`pm.pm.max_children`可以设为100。当然，实际的每个PHP-FPM进程平均所需内存要根据实际情况计算。
但PHP-FPM进程多不一定有用，有用是指PHP-FPM与NginX之间有连接，而连接数也是受到系统的`net.core.somaxconn`参数限制（连接数不是进程数，一个进程可以处理多个连接）。



## MySQL的配置优化

### 最大连接数

首先，与NginX、PHP-FPM不同，MySQL是多线程方式工作。一般MySQ处理连接的方式（`thread_handling`）有每个连接一个线程（`one-connection-per-thread`）和所有连接一个线程（`no-threads`）。
`no-threads`模式大多用在调试，而正式线上环境普遍采用的是`one-connection-per-thread`模式。

一般情况下，一个PHP-FPM进程最多只会跟MySQL建立一个连接，MySQL的最大连接数（`max_connections`）不应该少于PHP-FPM进程数，否则并发的时候有的PHP-FPM会连接不上MySQL。对于越来越大的并发量，不能一味提高MySQL的最大连接数，合理的方式是，设定一个MySQL连接数界限，超出的连接请求需等待。

### 短连接和长连接

PHP-FPM与MySQL之间的连接有两种形式，分短连接和长连接。 MySQL在`one-connection-per-thread`模式下，PHP-FPM需要请求操作数据库时：如果PHP-FPM使用短连接形式，那么MySQL都会新建一个线程来支持这个连接，数据操作结束后，PHP-FPM会回收这个连接，MySQL也会回收对应的线程，这里就会有一定的时间消耗和的IO消耗；如果PHP-FPM使用长连接形式，在没有相同的长连接的情况下，PHP-FPM才会新建一个连接，数据操作结束后这个线程不会被回收，而是等待被复用，这样虽然会节省一些开销，但是在高并发的情况下，大量的长连接建立不回收，MySQL也管理同样多的线程，大量的上下文切换和资源竞争，会使得MySQL执行效率下降。

如果PHP-FPM采用动态进程管理模式，且所有进程都与MySQL长连接的话，那么空闲时，部分PHP-FPM进程被回收（与MySQL的连接也会被回收），部分PHP-FPM进程被保留下来（与MySQL的连接也会被保留下来）；高并发时，保留下来的PHP-FPM进程可以复用已有的MySQL连接，其他的重新建立，这就相当于极端长连接和极端短连接的折中。

一般的LNMP网站应用使用短连接访问数据库就足够了，用完就回收，减轻系统负担；中大型的可能需要使用长连接，因为操作数据库的请求不断发起，把时间和IO花费在建立新连接和回收旧连接就很浪费，但是MySQL的线程数（连接数）必须维持在合理范围内。

### 线程池和连接池

要在高并发中，控制MySQL的线程数（连接数），一般的解决方案是设置线程池或者连接池。

线程池是在数据服务端建立有限数量的线程，来处理所有应用客户端的连接。MySQL企业版（收费）提供了线程池机制（`thread_handling=pool-of-threads`），详细配置可以参考官方文档[MySQL Enterprise Thread Pool](https://dev.mysql.com/doc/refman/5.7/en/thread-pool.html)。

连接池是在应用客户端与数据服务端之间，设置一个中间代理，这个中间代理会与数据服务端建立有限数量的连接，来处理所有应用客户端的数据请求。连接池可以使用开源的数据库中间件[atlas](https://github.com/Qihoo360/Atlas)来搭建，也可以使用PHP扩展[swoole](http://rango.swoole.com/archives/265)来自己写一个。

[php-cp](https://github.com/swoole/php-cp)是Swoole组织开发的一个PHP扩展，可以本地代理MySQL、Redis连接，提供连接池，读写分离，负载均衡，慢查询日志，大数据块日志等功能，在主流php框架引入使用也十分简便，具体可以参考[]()。

<!-- 假设有100台服务器，每台运行100个PHP-FPM进程；另有一台服务器上运行MySQL，所有PHP-FPM都会跟它对接数据。如果同时并发了10000个数据查询请求，100*100个PHP-FPM进程都被调用起来去访问MySQL，那么MySQL就需要生成10000个线程（MySQL基于线程来调度）来响应这些。 -->



## 其他


<!-- 1. 为什么有人会说PHP的长连接是鸡肋？因为NginX与PHP-FPM之间是短连接，NginX断开连接后，PHP-FPM进程会回收，除非PHP-FPM以静态模式运行（这样可以常驻内存），PHP-FPM进程回收后，其与MySQL的连接也会回收。即使PHP-FPM进程常驻内存，其与MySQL的连接长时间不活动，MySQL也会根据`wait_timeout`的时间限制来回收连接。另外PHP的长连接和短连接基本是一样的，只是短连接会自动回收，如果是短连接复用了长连接，那么这个长连接就会变成短连接，用完后自动回收。 -->

平时要多留意LNMP的日志信息，根据提示优化配置，比如：
1. NginX(24: Too many open files)说明`work_rlimit_nofile`参数需要增大；
2. NginX(worker_connections are not enough while connecting to upstream)说明`worker_connections`参数需要增大；
3. NginX(11: Resource temporarily unavailable)一般是并发连接超过来系统的`net.core.somaxconn`限制；
4. MySQL(too many connections)说明`max_connections`参数需要增大；
5. MySQL(too many open files)说明`open_files_limit`参数需要增大，当然系统的`fs.file-max`也不能过小；
6. MySQL(has gone away)是连接太久没活动被MySQL回收了，MySQL的`wait_timeout`也不能过小。

## 最后

此文的LNMP优化主要是针对高并发情况，其他方面优化有机会会继续补充。

<!-- // work_rlimit_nofile need rasing
2017/09/16 12:53:57 [alert] 30810#30810: *506 socket() failed (24: Too many open files) while connecting to upstream, client: 23.105.217.195, server: _, request: "GET /test.php HTTP/1.0", upstream: "fastcgi://127.0.0.1:9100", host: "66.112.214.155"


// worker_connections need rasing
2017/09/16 12:45:23 [alert] 10444#10444: *1788 768 worker_connections are not enough while connecting to upstream, client: 23.105.217.195, server: _, request: "GET /test.php HTTP/1.0", upstream: "fastcgi://127.0.0.1:9100", host: "66.112.214.155"

// time out
2017/09/17 00:06:25 [error] 12215#12215: *1900 recv() failed (104: Connection reset by peer) while reading response header from upstream, client: 23.105.217.195, server: _, request: "GET /test.php HTTP/1.0", upstream: "fastcgi://127.0.0.1:9100", host: "66.112.214.155"

// time out
2017/09/17 00:42:43 [error] 15366#15366: *5063 upstream timed out (110: Connection timed out) while reading response header from upstream, client: 23.105.217.195, server: _, request: "GET /test.php HTTP/1.0", upstream: "fastcgi://127.0.0.1:9100", host: "66.112.214.155"

// net.core.somaxconn  need rasing
2017/09/17 03:45:21 [error] 2332#2332: *914 connect() to unix:/run/php/php7.0-fpm.sock failed (11: Resource temporarily unavailable) while connecting to upstream, client: 23.105.217.195, server: _, request: "GET /test.php HTTP/1.0", upstream: "fastcgi://unix:/run/php/php7.0-fpm.sock:", host: "66.112.214.155" -->