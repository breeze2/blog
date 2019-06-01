title: 搭建Docker Registry私有镜像仓库
date: 2018-01-16 11:01:06
tags: 
    - docker
    - registry
    - practice
categories:
    - 实践
---

[Docker Store](https://store.docker.com/)是Docker官方提供的公共的镜像仓库，但有时会需要在局部内共享镜像，那么可以利用[Docker Registry](https://docs.docker.com/registry/)工具搭建一个私有的镜像仓库。

直接运行官方[registry](https://store.docker.com/images/registry)镜像，即可开启镜像仓库服务：
```cmd
$ docker run -d -p 5000:5000 --restart=always --name registry registry
```

若要定制使用，还需更多配置。

<!--more-->
## Docker安装
简单介绍一下Docker的安装：
```cmd
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt-get update
$ sudo apt-get install -y docker-ce
$ sudo apt-get install -y docker-compose
$ sudo usermod -aG docker ${USER}
```

## Docker Compose安装
Docker Compose的安装：
```cmd
$ sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

## Docker Compose编排
进入工作目录，编辑`docker-compose.yml`：
```cmd
$ cd ~/workspace
$ cd docker/registry
$ vi docker-compose.yml
```
基于官方镜像`registry`制作，`/etc/docker/registry`是容器内配置信息路径，映射到本地`./config`；`/var/lib/registry`是容器内数据保存路径，映射到本地`./data`：
```yml
version: "3"
services:
  registry:
    image: registry
    ports:
      - "5000:5000"
    volumes:
      - "./config:/etc/docker/registry"
      - "./data:/var/lib/registry"
```

## Docker Registry配置
设置HTTP认证：
```cmd
$ mkdir -p config/auth
$ htpasswd -Bbn $USERNAME $PASSWORD > ./config/auth/htpasswd # or
$ docker run --rm --entrypoint htpasswd registry -Bbn $USERNAME $PASSWORD > ./config/auth/htpasswd
```

配置Registry：
```cmd
$ vi config/config.yml
```

`config/config.yml`：
```yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
auth:
  htpasswd:
    realm: basic-realm
    path: /etc/docker/registry/auth/htpasswd
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

## 运行与测试

```cmd
$ docker-compose up -d
$ docker login 127.0.0.1:5000
$ docker pull ubuntu:16.04
$ docker tag ubuntu:16.04 127.0.0.1/username/ubuntu:16.04
$ docker push 127.0.0.1/username/ubuntu:16.04
$ docker pull 127.0.0.1/username/ubuntu:16.04
$ docker logout
$ docker push 127.0.0.1/username/ubuntu:16.04
```

## *HTTPS
虽然Docker Registry默认监听端口是`5000`，但实际上服务是基于HTTP，可以通过HTTPS保障传输数据安全。Docker默认不允许非HTTPS连接下推送镜像（`127.0.0.1`本地网络地址例外），要取消这个限制须修改Docker配置——比如连接`192.168.1.1:5000`服务，在`/etc/default/docker`文件中添加`DOCKER_OPTS="--insecure-registries=192.168.1.1:5000"`，然后重启Docker：
```cmd
$ sudo service docker restart
```
当然，最好还是给Docker Registry设置HTTPS：可以用NginX代理本地`5000`端口，然后在NginX上设置HTTPS；也可以在Docker Registry内部设置HTTPS，详细过程请阅读
[《私有仓库高级配置》](https://yeasy.gitbooks.io/docker_practice/content/repository/registry_auth.html)。

## 参考
* [Deploy a registry server](https://docs.docker.com/registry/deploying/)
* [Configuring a registry](https://docs.docker.com/registry/configuration/)