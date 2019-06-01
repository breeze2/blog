title: 用Docker搭建RabbitMQ高可用集群
date: 2017-12-04 18:39:06
tags: 
    - docker
    - rabbitmq
    - haproxy
    - practice
categories:
    - 实践
---

> [RabbitMQ](https://www.rabbitmq.com/)是基于[高级消息队列协议（AMQP）](https://www.amqp.org/)实现的开源消息代理软件，主要提供消息队列服务。这里介绍用Docker Compose搭建RabbitMQ高可用集群的过程。

RabbitMQ自身提供部署集群的功能，通过命令：
``` cmd
$ rabbitmqctl -n rabbit@rmqha_node1 stop_app
$ rabbitmqctl -n rabbit@rmqha_node1 join_cluster --ram rabbit@rmqha_node0
$ rabbitmqctl -n rabbit@rmqha_node1 start_app
```
就可以很容易的将节点rabbit@rmqha_node1加入到集群rabbit@rmqha_node0中。`--ram`选项表示节点以内存存储方式运行，读写速度快，重启后内容会丢失；不加`--ram`选项，节点则以磁盘存储方式运行，虽然读写速度慢，但是内容一般可以持久保持。

在同一个RabbitMQ集群中，节点之间并没有主从之分，所有节点会同步相同的队列结构，队列内容（消息）则各自不同，不过消息会在节点间传递。这样的集群只是提高了应对大量并发请求的能力，整体可用性还是很低，因为某个节点宕机后，寄存在该节点上的消息不可用，而在其他节点上也没有这些消息的备份，若是该节点无法恢复，那么这些消息就丢失了。

为了解决这个问题，RabbitMQ提供镜像队列功能，通过命令：
```cmd
$ rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```
可以设置镜像队列，`"^"`表示匹配所有队列，即所有队列在各个节点上都会有备份。在集群中，只需要在一个节点上设置镜像队列，设置操作会同步到其他节点。

<!--more-->

## Docker Compose编排
这个编排主要实现一个磁盘节点、两个内存节点的RabbitMQ集群和一个HAProxy代理。
### 目录结构
```cmd
L--rabbitmq-ha-docker                    //主目录
    L--scripts                           //本地（Docker宿主）使用的一些脚本
        L--rmqha_set_policy.sh           //设置各个数据库账号和开启主从复制
    L--volumes                           //各个容器的挂载数据卷
        L--rmqha_proxy
            L--haproxy.cfg               //HAProxy配置
        L--rmqha_slave
            L--cluster_entrypoint.sh     //入口文件
    L--parameters.env                    //账号密码等环境参数
    L--docker-compose.yml                //编排配置

```

### docker-compose.yml
```yml
version: "2"
services:

  master:
    image: rabbitmq:3.6-management
    container_name: rmqha_node0
    restart: always
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.9.0.10
    hostname: rmqha_node0
    ports:
      - "55672:15672"
      - "56720:5672"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=rmqha_node0
      - RABBITMQ_HOSTNAME=rmqha_node0
      - RABBITMQ_NODENAME=rabbit

  slave1:
    image: rabbitmq:3.6-management
    container_name: rmqha_node1
    restart: always
    depends_on:
      - master
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.9.0.11
    hostname: rmqha_node1
    # ports:
    #   - "56721:5672"
    volumes:
      - "./volumes/rmqha_slave/cluster_entrypoint.sh:/usr/local/bin/cluster_entrypoint.sh"
    entrypoint: "/usr/local/bin/cluster_entrypoint.sh"
    command: "rabbitmq-server"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=rmqha_node1
      - RABBITMQ_HOSTNAME=rmqha_node1
      - RABBITMQ_NODENAME=rabbit
      - RMQHA_RAM_NODE=true

  slave2:
    image: rabbitmq:3.6-management
    container_name: rmqha_node2
    restart: always
    depends_on:
      - master
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.9.0.12
    hostname: rmqha_node2
    # ports:
    #   - "56722:5672"
    volumes:
      - "./volumes/rmqha_slave/cluster_entrypoint.sh:/usr/local/bin/cluster_entrypoint.sh"
    entrypoint: "/usr/local/bin/cluster_entrypoint.sh"
    command: "rabbitmq-server"
    env_file:
      - ./parameters.env
    environment:
      - CONTAINER_NAME=rmqha_node2
      - RABBITMQ_HOSTNAME=rmqha_node2
      - RABBITMQ_NODENAME=rabbit
      - RMQHA_RAM_NODE=true
  
  haproxy:
    image: haproxy:1.8
    container_name: rmqha_proxy
    restart: always
    depends_on:
      - master
      - slave1
      - slave2
    mem_limit: 256m
    networks:
      net1:
        ipv4_address: 10.9.0.19
    hostname: rmqha_proxy
    ports:
      - "56729:5672"
      - "51080:1080"
    volumes:
      - "./volumes/rmqha_proxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro"
      - "./volumes/rmqha_proxy:/root/rmqha_proxy"
    environment:
      - CONTAINER_NAME=rmqha_proxy

networks:
  net1:
    driver: bridge
    ipam:
      config:
        - subnet: 10.9.0.0/16
          gateway: 10.9.0.1

```

这里配置了四个容器服务，一个`haproxy`，负责代理各个RabbitMQ服务；三个`rabbitmq`，组成RabbitMQ集群。每个容器服务都指定了静态IP，即使服务重启也不会出现IP错乱问题，特殊的网络端口映射后面会介绍。

### 环境参数
parameters.env
```env
RMQHA_MASTER_NODE=rabbit
RMQHA_MASTER_HOST=rmqha_node0
RABBITMQ_DEFAULT_USER=guest
RABBITMQ_DEFAULT_PASS=guest
RABBITMQ_NODENAME=rabbit
RABBITMQ_ERLANG_COOKIE=myerlangcookie
```

### RabbitMQ启动
这里RabbitMQ容器是使用[Docker官方镜像](https://hub.docker.com/_/rabbitmq/)生成的，节点rmqha_node0可以直接启动；而节点rmqha_node1和rmqha_node2需要加入到rmqha_node0集群里，所以需要需改入口文件。
volumes/rmqha_slave/cluster_entrypoint.sh
```shell
#!/bin/bash
set -e

if [ -e "/root/is_not_first_time" ]; then
    exec "$@"
else
    /usr/local/bin/docker-entrypoint.sh rabbitmq-server -detached # 先按官方入口文件启动且是后台运行

    rabbitmqctl -n "$RABBITMQ_NODENAME@$RABBITMQ_HOSTNAME" stop_app # 停止应用
    rabbitmqctl -n "$RABBITMQ_NODENAME@$RABBITMQ_HOSTNAME" join_cluster ${RMQHA_RAM_NODE:+--ram} "$RMQHA_MASTER_NODE@$RMQHA_MASTER_HOST" # 加入rmqha_node0集群
    rabbitmqctl -n "$RABBITMQ_NODENAME@$RABBITMQ_HOSTNAME" start_app # 启动应用
    rabbitmqctl stop # 停止所有服务

    touch /root/is_not_first_time
    sleep 2s
    exec "$@"
fi
```

### HAProxy配置
这里HAProxy容器也是使用[Docker官方镜像](https://hub.docker.com/_/haproxy/)生成的，启动前需要先准备配置文件。
volumes/rmqha_proxy/haproxy.cfg
```config
global
    log 127.0.0.1 local0
    maxconn 4096

defaults
    log     global
    mode    tcp
    option  tcplog
    retries 3
    option  redispatch
    maxconn 2000
    timeout connect 5000
    timeout client 50000
    timeout server 50000

# ssl for rabbitmq
# frontend ssl_rabbitmq
    # bind *:5673 ssl crt /root/rmqha_proxy/rmqha.pem
    # mode tcp
    # default_backend rabbitmq

listen stats
    bind *:1080 # haproxy容器1080端口显示代理统计页面，映射到宿主51080端口
    mode http
    stats enable
    stats hide-version
    stats realm Haproxy\ Statistics
    stats uri /
    stats auth admin:admin

listen rabbitmq
    bind *:5672 # haproxy容器5672端口代理多个rabbitmq服务，映射到宿主56729端口
    mode tcp
    balance roundrobin
    timeout client 1h
    timeout server 1h
    option  clitcpka
    # server  rmqha_node0 rmqha_node0:5672  check inter 5s rise 2 fall 3
    server  rmqha_node1 rmqha_node1:5672  check inter 5s rise 2 fall 3
    server  rmqha_node2 rmqha_node2:5672  check inter 5s rise 2 fall 3
```

## 实际运行

在主目录下执行`docker-compose up -d`构建并运行整个Docker服务。

### 镜像队列
，在主目录下执行：
```cmd
$ sh ./scripts/rmqha_set_policy.sh
```
实际上是执行了：
```
$ docker exec -it rmqha_node0 rabbitmqctl set_policy ha-all '^' '{"ha-mode":"all"}'
```
即在rmqha_node0集群中将所有队列设置为镜像队列，这个命令只需执行一次，除非重新构建整个Docker服务。

## 测试
用两个PHP脚本可以对RabbitMQ进行简单的测试，不过需要用到`php-amqplib`库。
```cmd
$ composer require php-amqplib/php-amqplib
```
发送消息脚本：
```php
<?php
// send.php
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('127.0.0.1', 56729, 'guest', 'guest', '/'); // 连接rmqha_proxy
$channel    = $connection->channel();
$channel->queue_declare('task_queue', false, true, false, false);
$data = "Hello World!";

$msg = new AMQPMessage($data,
    array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
);
$channel->basic_publish($msg, '', 'task_queue');
echo " [x] Sent ", $data, "\n";
$channel->close();
$connection->close();
```
接受消息脚本：
```php
<?php
// receive.php
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('127.0.0.1', 56720, 'guest', 'guest', '/'); // 连接rmqha_node0
$channel    = $connection->channel();
$channel->queue_declare('task_queue', false, true, false, false);
echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
$callback = function ($msg) {
    echo " [x] Received ", $msg->body, "\n";
    sleep(1);
    echo " [x] Done", "\n";
    $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};
$channel->basic_qos(null, 1, null);
$channel->basic_consume('task_queue', '', false, false, false, false, $callback);

while (count($channel->callbacks)) {
    $channel->wait();
}
$channel->close();
$connection->close();
```

## SSL设置
为了RabbitMQ服务在网络传输中不泄漏信息，可以给HAProxy设置SSL传输（比对各个RabbitMQ服务设置SSL传输的性能消耗要小），这里简单的介绍一下自签证书：
```cmd
$ cd ./volumes/rmqha_proxy/
$ openssl genrsa -out rmqha.key 1024 # 随机生成一个私钥
$ openssl req -new -key rmqha.key -out rmqha.csr # 根据私钥生成证书签署请求
$ openssl x509 -req -days 365 -in rmqha.csr -signkey rmqha.key -out rmqha.crt # 自己签署证书
$ cat rmqha.crt rmqha.key|tee rmqha.pem # 将私钥和证书合并到一个文件中
```
注意：生成证书签署请求时，需要填写一些信息，`Common Name (eg, fully qualified host name)`应该写`127.0.0.1`。
HAProxy配置中添加：
```config
# ssl for rabbitmq
frontend ssl_rabbitmq
    bind *:5673 ssl crt /root/rmqha_proxy/rmqha.pem # haproxy容器5673端口代理rabbitmq服务，需要映射到宿主端口
    mode tcp
    default_backend rabbitmq
```
PHP中通过SSL连接HAProxy代理服务示例：
```php
<?php
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPSSLConnection;
$connection = new AMQPSSLConnection('127.0.0.1', 56730, 'guest', 'guest', '/', ['cafile'=>'./rmqha.crt']); // 假设SSL的监听端口是56730，rmqha.crt是上面生成的自签证书
$channel    = $connection->channel();
```

## 最后

### 建议
1. 消息生产者可以通过HAProxy代理连接RabbitMQ服务，实现负载均衡；
2. 消息处理者应该直接连接RabbitMQ服务主机，并且跟随RabbitMQ服务主机启动服务、停止服务。

### 扩展阅读
* [Rabbitmq高可用-镜像队列模式](https://addops.cn/post/rabbitmq-ha-mirror.html)
* [Rabbitmq集群高可用测试](http://www.cnblogs.com/flat_peach/archive/2013/04/07/3004008.html)



