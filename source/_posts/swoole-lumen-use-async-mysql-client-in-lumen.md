title: 在Lumen项目中使用Swoole异步MySQL客户端的实现方法
date: 2018-03-01 16:03:02
tags: 
    - swoole
    - lumen
    - php
    - mysql
    - think
categories:
    - 求索
---

> [上一篇文章](https://blog.breezelin.cn/swoole-lumen-run-lumen-with-swoole.html)介绍了用Swoole HTTP服务器替代NginX + PHP-FPM来运行Lumen项目的方法，提高运行效率。不过，在处理用户请求过程中还是会存在很多IO阻塞情况，比如MySQL数据查询。有没有可能在Lumen中使用异步MySQL客户端，以此避免IO阻塞呢？

## 异步MySQL客户端
Swoole1.8.6增加了内置异步MySQL客户端的支持，无需依赖其他第三方库，使用方法也非常简单，（具体参考Swoole官方文档[异步MySQL客户端](https://wiki.swoole.com/wiki/page/517.html)）：
```php
<?php
$db = new \swoole_mysql;
$mysql_config = array(
    'host' => '192.168.56.102',
    'port' => 3306,
    'user' => 'test',
    'password' => 'test',
    'database' => 'test',
    'charset' => 'utf8', //指定字符集
    'timeout' => 2,
);

$db->connect($mysql_config, function ($db, $r) {
    if ($r) {
        $sql = 'show tables';
            $db->query($sql, function($db, $r) {
            if ($r) {
                var_dump($r);
            }
            $db->close();
        });
    }
});
```

<!--more-->
这是典型回调处理的异步编程风格，而Lumen本身是同步编程风格，在编程层面，两者不能融合。比如，有这么一个`Controller`：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;
use DB;

class TestController extends Controller
{
    public function test()
    {
        $a = DB::query('select * from users;');
        response()->json($a);
    }
}
```
若是改写成：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;

class TestController extends Controller
{
    public function test()
    {
        $a = null;
        $db = new \swoole_mysql;
        $db->connect($mysql_config, function ($db, $r) use ($a) {
            if ($r) {
                $db->query('select * from users;', function($db, $r) use ($a) {
                    if ($r) {
                        $a = json_decode(json_encode($r));
                    }
                    $db->close();
                });
            }
        });
        response()->json($a);
    }
}
```
这样返回的响应永远是`null`，因为`response()->json($a)`会在`$db->query()`之前被执行。

> 一开始我也觉得Lumen项目里永远没办法使用异步MySQL客户端了，直到看了这篇文章：[Cooperative multitasking using coroutines (in PHP!)](http://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html)。当然，我看的是中文版：[ 在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html)，文中提到了PHP5.5加入的一个新功能：[yield](http://php.net/manual/en/language.generators.syntax.php)。

## yield
`yield`是个动词，意思是“生成”，PHP中yield生出的东西叫`Generator`，译作“生成器”。
yield可以做什么呢？yield可以将当前执行的上下文作为当前函数的结果返回（yield必须在函数中使用）。
有了yield，又能怎样呢？

首先，我们声明一个类，叫`SlwoQuery`：
```php
<?php
namespace BL\SwooleHttp\Database;

class SlowQuery
{
    public $sql = '';
    public function __construct($sql)
    {
        $this->sql = $sql;
    }
}
```

结合上一篇文章《[用Swoole HTTP服务器运行Lumen项目的实现方法](https://blog.breezelin.cn/swoole-lumen-run-lumen-with-swoole.html)》，我们修改一下`Service`类的`onRequest`方法：
```php
<?php
    public function onRequest(\swoole_http_request $request,\swoole_http_response $response) {
        $app = $this->app;
        // 处理用户请求
        $http_request = $this->parseRequest($request);
        $http_response = $app->dispatch($http_request);

        if ($http_response instanceof \Generator) {
            $gen = $http_response->current();
            $gen_queue = new \SplQueue();
            while($gen instanceof Generator) {
                $gen_queue->push($gen); $gen = $gen->current();
            }
            $last_gen = $gen_queue->pop();
            $value = $last_gen->current();
            if ($value instanceof \BL\SwooleHttp\Database\SlowQuery) {
                $db = new \swoole_mysql;
                $caller = $this;
                // 关键部分
                $db->connect($mysql_config, function ($db, $r) use ($caller, $request, $response, $gen_queue, $last_gen, $value) {
                    if ($r) {
                        $db->query($value->sql, function($db, $r) use ($caller, $request, $response, $gen_queue, $last_gen) {
                            if ($r) {
                                // 关键部分
                                $r = json_decode(json_encode($r));
                                $last_gen->send($r);
                                $ret = $last_gen->getReturn();
                                while(!$gen_queue->isEmpty()) {
                                    $gen = $gen_queue->pop(); $gen->send($ret); $ret = $gen->getReturn();
                                }
                                $caller->makeResponse($response, $ret);
                            }
                            $db->close();
                        });
                    }
                });
            }
        } else {
            // 响应用户请求
            $this->makeResponse($response, $http_response);
        }
    }
```
代码是长了些，因为没做分拆和封装，主要关注`$db->connect()`和`$caller->makeResponse()`的出现位置。整块代码的意思是：
1. 如果Lumen处理用户请求的返回结果是一个生成器，那么就从这个生成器的函数套层里寻找（SlowQuery）；
2. 如果当前生成器的当前值也是一个生成器，那么就往更深一层里寻找；
3. 如果当前生成器的当前值是一个`SlowQuery`对象，那么将`SlowQuery`对象的`sql`属性，交给异步MySQL客户`swoole_mysql`查询数据；
4. 将查询数据传入当前生成器，获得当前层函数的返回结果；
5. 将函数返回结果传入上一层生成器，获取上一层函数的返回结果，重复直到没有更高层生成器；
6. 才将最顶层层的函数返回结果作为响应输出给用户。
虽然整个过程很绕，但是，在Lumen的Controller层面却是十分的直接了然：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;
use BL\SwooleHttp\Database\SlowQuery;

class TestController extends Controller
{
    public function test()
    {
        $a = yield new SlowQuery('select * from users;');
        response()->json($a);
    }
}
```

就这样，实现了在同步编程风格的Lumen项目中使用上了Swoole的异步MySQL客户端。实现的思路，就像[后置中间件](https://lumen.laravel.com/docs/middleware)，保持控制器代码的整洁，脏活累活放在中间件里执行。

## 效率提升
假如用户请求执行一个数据库慢查询语句（可能耗时1秒，如`select sleep(1);`），单个PHP进程使用同步MySQL客户端处理1个这样的用户请求，至少需要1秒，处理10个，则至少需要10秒；而使用异步MySQL客户端，处理1个，可能需要1秒多，处理10个，可能也只是需要1秒。异步执行带来的效率提升，是不言而喻的。

不过，使用异步MySQL客户端是会消耗系统资源的，不能大量使用；而且非慢查询的查询语句，根本不需要使用异步MySQL客户端来执行，比如`select * from users.id = 1;`，id有主键索引，查询时间不会太长。

## 最后
yield可以将整个程序的各个代码块的执行秩序交由程序员自行调度，确实可以实现很多意向不到的效果。像以上这种“后置执行”的思路，不仅仅是可以使用异步MySQL客户端，还可以使用异步Redis客户端，异步HTTP客户端，还有更多各种各样的应用。

下一篇文章介绍如何自定义后置异步协程。


