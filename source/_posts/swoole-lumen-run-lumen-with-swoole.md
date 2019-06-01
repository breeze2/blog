title: 用Swoole HTTP服务器运行Lumen项目的实现方法
date: 2018-03-01 12:03:02
tags: 
    - swoole
    - lumen
    - php
    - think
categories:
    - 求索
---

> LNMP虽是传统的Web应用架构组合，无奈NginX + PHP-FPM的搭配运行效率实在太低；而Swoole HTTP服务器具有NginX级别的性能，且本身嵌入在PHP中，完全可以替代NginX + PHP-FPM。于是一直在探索用Swoole HTTP服务器运行传统PHP应用的途径。这里主要介绍一下Swoole HTTP服务器运行Lumen项目的实现方法：

## Swoole HTTP服务器
Swoole1.7.7增加了内置HTTP服务器的支持，使用了跟NginX一样的IO多路复用机制epoll，可以达到NginX级别的性能，并且只需几行代码即可写出一个异步非阻塞多进程的HTTP服务器（epoll属于同步非阻塞，这里说“异步非阻塞”主要是因为“多进程”，单个进程内部依然是同步非阻塞）。
<!--more-->
开启一个Swoole HTTP服务器的面向对象编程代码样板大致如下（具体参考Swoole官方文档[HttpServer](https://wiki.swoole.com/wiki/page/326.html)）：
```php
<?php

class Service {
    public $app;
    public $server;

    public function __construct($host, $port) {
        $this->server = new \swoole_http_server($host, $port);
    }

    public function start()
    {
        // $this->server->on('start', array($this, 'onStart'));
        // $this->server->on('shutdown', array($this, 'onShutdown'));
        // $this->server->on('workerStop', array($this, 'onWorkerStop'));
        $this->server->on('workerStart', array($this, 'onWorkerStart'));
        $this->server->on('request', array($this, 'onRequest'));
        $this->server->start();
    }

    public function onWorkerStart($serv, $worker_id) {
        // 应用初始化
        $this->app = 1;
    }

    public function onRequest(\swoole_http_request $request,\swoole_http_response $response) {
        $app = $this->app;
        // 处理用户请求
        $get = json_encode($request->get);
        // 响应用户请求
        $response->end("App is {$app}. Get {$get}");
    }
}

$s = new Service("127.0.0.1", 9080);
$s->start();
```

直接用`php`命令执行以上代码，浏览器访问`http://127.0.0.1:9080`，会得到：
```string
App is 1. Get null
```
浏览器访问`http://127.0.0.1:9080/?user=guest`，会得到：
```string
App is 1. Get {"user":"guest"}
```
就是这么简单。

## 加载Lumen项目
Lumen项目的入口文件是项目路径下的`public/index.php`，里面只有简单的两行代码：
```php
<?php
$app = require __DIR__.'/../bootstrap/app.php';
$app->run();
```
第一行代码就是加载整个Lumen框架代码，第二行代码则是处理请求，生成响应。

所以，在Swoole HTTP服务器中加载Lumen项目，也很简单，只需修改`onWorkerStart`方法：
```php
<?php
    public function onWorkerStart($serv, $worker_id) {
        // 应用初始化
        $this->app = require '/THE/FULL/PATH/TO/bootstrap/app.php';
    }
```

## 处理用户请求
虽然可以加载Lumen框架，但是要怎样才能用Lumen框架来处理用户请求呢？

由于Lumen框架的解耦程度非常高，我们可以很轻松地将`swoole_http_request`对象`$request`，转换成Lumen框架熟悉的`Illuminate\Http\Request`对象，这样Lumen框架就能处理用户请求了。
这里实现一个`parseRequest`方法，接收`swoole_http_request`类参数，返回`\Illuminate\Http\Request`类结果：
```php
<?php
    protected function parseRequest(\swoole_http_request $request)
    {
        $get     = isset($request->get) ? $request->get : array();
        $post    = isset($request->post) ? $request->post : array();
        $cookie  = isset($request->cookie) ? $request->cookie : array();
        $server  = isset($request->server) ? $request->server : array();
        $header  = isset($request->header) ? $request->header : array();
        $files   = isset($request->files) ? $request->files : array();
        $fastcgi = array();

        $new_server = array();
        foreach ($server as $key => $value) {
            $new_server[strtoupper($key)] = $value;
        }
        foreach ($header as $key => $value) {
            $new_server['HTTP_' . strtoupper($key)] = $value;
        }

        $content = $request->rawContent() ?: null;

        $http_request = new \Illuminate\Http\Request($get, $post, $fastcgi, $cookie, $files, $new_server, $content);

        return $http_request;
    }
```


## 生成请求响应
Lumen框架处理用户请求，十分方便，只需调用应用的`dispatch`方法即可。不过`dispatch`方法返回结果一般是`Symfony\Component\HttpFoundation\Response`对象，需要做一些转化才能交由`swoole_http_response`对象给用户输出响应。
这里实现一个`makeResponse`方法：
```php
<?php
    protected function parseRequest(\swoole_http_response $request, \Symfony\Component\HttpFoundation\Response $http_response)
    {
        // status
        $response->status($http_response->getStatusCode());
        // headers
        foreach ($http_response->headers->allPreserveCase() as $name => $values) {
            foreach ($values as $value) {
                $response->header($name, $value);
            }
        }
        // cookies
        foreach ($http_response->headers->getCookies() as $cookie) {
            $response->rawcookie(
                $cookie->getName(),
                $cookie->getValue(),
                $cookie->getExpiresTime(),
                $cookie->getPath(),
                $cookie->getDomain(),
                $cookie->isSecure(),
                $cookie->isHttpOnly()
            );
        }
        // content
        $content = $http_response->getContent();
        // send content
        $response->end($content);
    }
```

有了`parseRequest`方法和`makeResponse`方法，要实现以Swoole HTTP服务器运行Lumen项目来处理用户请求只要几行代码。`onRequest`方法这样修改：
```php
<?php
    public function onRequest(\swoole_http_request $request,\swoole_http_response $response) {
        $app = $this->app;
        // 处理用户请求
        $http_request = $this->parseRequest($request);
        $http_response = $app->dispatch($http_request);
        // 响应用户请求
        $this->makeResponse($response, $http_response);
    }
```

## 最后
就这样，不需要修改Lumen项目原本任何代码，就能让其运行在Swoole HTTP服务器上。
当然，用户请求各种各样，有请求文件上传，有请求静态资源等等；响应类型也是各种各样，有Gzip压缩，有JSON格式，有设置Cookie等待。要实现一个健全的`Swoole+Lumen`网关服务，以上代码还须更多优化，具体可以参考[Service.php](https://github.com/breeze2/lumen-swoole-http)。

> 按照以上思路，要实现以Swoole HTTP服务器运行Laravel项目也不难，不过，为了效率选择Swoole，自然会为了效率选择Lumen而不是Laravel。

[下一篇文章](https://blog.breezelin.cn/swoole-lumen-use-async-mysql-client-in-lumen.html)介绍如何在Lumen中使用异步MySQL客户端，减少IO阻塞。

