title: 在Lumen项目中自定义后置异步协程
date: 2018-03-01 23:12:02
tags: 
    - swoole
    - lumen
    - php
    - think
categories:
    - 求索
---

> [上一篇文章](https://blog.breezelin.cn/swoole-lumen-use-async-mysql-client-in-lumen.html)介绍了利用yield特性实现后置的MySQL异步查询协程，按照这种“后置执行”的思路，其实还可以实现更多其他的后置异步协程。这里介绍如何自定义一个后置异步协程，将最大开发自由度交由Lumen框架使用者。

## CustomAsyncProcess
[lumen-swoole-http](https://breeze2.github.io/lumen-swoole-http/)中已经定义了抽象类`CustomAsyncProcess`，在处理用户请求时，若是捕获到`CustomAsyncProcess`类的对象，程序便会按照`CustomAsyncProcess`对象的指定方法执行。部分代码：
```php
<?php
    ...
    $current_value = $last_generator->current();
    if ($current_value instanceof CustomAsyncProcess) {
        if ($worker->canDoCoroutine()) {
            $worker->upCoroutineNum();
            return $current_value->runAsyncTask($request, $response, $worker, $this->scheduler, $last_generator);
        } else {
            return $current_value->runNormalTask($request, $response, $worker, $this->scheduler, $last_generator);
        }
    }
    ...
```

<!--more-->
## 一个简单的后置协程
我们先来实现一个简单的后置协程`EasyProcess`，继承自`CustomAsyncProcess`，需要实现两个抽象方法：
```php
<?php
namespace App\http\AfterCoroutines;
use BL\SwooleHttp\Service;
use BL\SwooleHttp\Coroutine\SimpleSerialScheduler;
use Generator;
use swoole_http_request as SwooleHttpRequest;
use swoole_http_response as SwooleHttpResponse;

class EasyProcess extends \BL\SwooleHttp\Coroutine\CustomAsyncProcess
{
    // 因为有协程数量限制，达到最大协程数量时，会调用runNormalTask方法
    public function runNormalTask(SwooleHttpRequest $request, SwooleHttpResponse $response, Service $worker, SimpleSerialScheduler $scheduler, Generator $last_generator)
    {
        $value = 2;
        // 给最后一层生成器，传入$value值，并递归导出整个生成器套层的最终值$final
        $final = $this->fullRunScheduler($scheduler, $last_generator, $value);
        $http_response = $final;
        // 注意：runNormalTask方法中使用makeNormalResponse方法响应用户请求
        $this->makeNormalResponse($request, $response, $worker, $http_response);
    }

    // 没达到最大协程数量时，会调用runAsyncTask方法
    public function runAsyncTask(SwooleHttpRequest $request, SwooleHttpResponse $response, Service $worker, SimpleSerialScheduler $scheduler, Generator $last_generator)
    {
        $value = 1;
        // 给最后一层生成器，传入$value值，并递归导出整个生成器套层的最终值$final
        $final = $this->fullRunScheduler($scheduler, $last_generator, $value);
        $http_response = $final;
        // 注意：runAsyncTask方法中使用makeAsyncResponse方法响应用户请求
        $this->makeAsyncResponse($request, $response, $worker, $http_response);
    }
}
```

然后就可以在Controller中使用这个后置协程：
```php
<?php
namespace App\Http\Controllers;
use App\http\AfterCoroutines\EasyProcess;
use App\Http\Controllers\Controller;

class TestController extends Controller
{
    public function test()
    {
        $a = yield new EasyProcess();
        response()->json($a);
    }
}
```

当访问`TestController@test`对应的路由时，得到的返回结果有可能是1，也有可能是2，视当时协程数量而定。1或2这个结果在`TestController@test`层面上是未曾知的，是由`EasyProcess`这个后置协程产出的。

`EasyProcess`不算一个异步协程，无论`runNormalTask`方法或者`runAsyncTask`方法，都是同步执行的。若想实现异步执行，必须使用Swoole提供的异步客户端，详细请查看Swoole官方文档[AsyncIO](https://wiki.swoole.com/wiki/page/p-async.html)。

下面介绍如何实现一个后置的异步HTTP客户端协程。

## 一个后置的异步HTTP客户端协程
假设我们处理某个用户请求时，需要从远端链接`http://www.domain.com/api/data`获取一些数据，期间可能需要耗时几秒钟，那么怎样用后置异步协程实现来避免阻塞呢？

我们先来实现一个简单的后置协程`AysncHttpProcess`，继承自`CustomAsyncProcess`，需要实现两个抽象方法：
```php
<?php
namespace App\http\AfterCoroutines;
use BL\SwooleHttp\Service;
use BL\SwooleHttp\Coroutine\SimpleSerialScheduler;
use Generator;
use swoole_http_request as SwooleHttpRequest;
use swoole_http_response as SwooleHttpResponse;
use Swoole\Http2\Client as SwooleHttpClient;

class AysncHttpProcess extends \BL\SwooleHttp\Coroutine\CustomAsyncProcess
{
    public function $url;
    public function __construct($url)
    {
        $this->url = $url;
    }

    // 因为有协程数量限制，达到最大协程数量时，会调用runNormalTask方法
    public function runNormalTask(SwooleHttpRequest $request, SwooleHttpResponse $response, Service $worker, SimpleSerialScheduler $scheduler, Generator $last_generator)
    {
        $data = file_get_contents($this->url);
        // 给最后一层生成器，传入$value值，并递归导出整个生成器套层的最终值$final
        $final = $this->fullRunScheduler($scheduler, $last_generator, $data);
        $http_response = $final;
        // 注意：runNormalTask方法中使用makeNormalResponse方法响应用户请求
        $this->makeNormalResponse($request, $response, $worker, $http_response);
    }

    // 没达到最大协程数量时，会调用runAsyncTask方法
    public function runAsyncTask(SwooleHttpRequest $request, SwooleHttpResponse $response, Service $worker, SimpleSerialScheduler $scheduler, Generator $last_generator)
    {
        $client = new SwooleHttpClient();
        $caller = $this;
        $client->get($this->url, function ($o) use($client, $caller) {
            $data = $o->body;
            // 给最后一层生成器，传入$value值，并递归导出整个生成器套层的最终值$final
            $final = $caller->fullRunScheduler($scheduler, $last_generator, $value);
            $http_response = $final;
            // 注意：runAsyncTask方法中使用makeAsyncResponse方法响应用户请求
            $caller->makeAsyncResponse($request, $response, $worker, $http_response);
            $client->close();
        });
    }
}
```

然后就可以在Controller中使用这个后置协程：
```php
<?php
namespace App\Http\Controllers;
use App\http\AfterCoroutines\AysncHttpProcess;
use App\Http\Controllers\Controller;

class TestController extends Controller
{
    public function test()
    {
        $a = yield new AysncHttpProcess('http://www.domain.com/api/data');
        response()->json($a);
    }
}
```

就这样，在协程数量允许情况下，我们可以使用后置的异步HTTP客户端协程获取远端数据，并且避免了阻塞。

## 最后
三篇文章：
1. [用Swoole HTTP服务器运行Lumen项目的实现方法](https://blog.breezelin.cn/swoole-lumen-run-lumen-with-swoole.html)
2. [在Lumen项目中使用Swoole异步MySQL客户端的实现方法](https://blog.breezelin.cn/swoole-lumen-use-async-mysql-client-in-lumen.html)
3. [在Lumen项目中自定义后置异步协程]()

至此，已经将[lumen-swoole-http](https://breeze2.github.io/lumen-swoole-http/)的设计理念介绍完了，主要是两点：
* 利用Swoole HTTP服务器运行Lumen项目，提高执行效率；
* 利用后置异步协程，保持同步编程风格，避免主进程阻塞。

> Swoole填补了PHP网络编程的缺陷，引入NodeJS、GoLang等语言的并行执行特性，使用了不同的编程风格。对此，不应该一味抗拒，而是学会融会贯通——学无止境，知识没有界限，PHPer也应该学习其他语言的设计理念，特别是为高并发而生的GoLang。