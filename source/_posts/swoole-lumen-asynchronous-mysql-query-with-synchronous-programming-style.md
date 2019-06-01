title: Swoole+Lumen：同步编程风格调用MySQL异步查询
date: 2018-03-02 09:23:02
tags: 
    - swoole
    - lumen
    - php
    - think
categories:
    - 求索
---

> 网络编程一直是PHP的短板，尽管[Swoole](https://www.swoole.com/)扩展弥补了这个缺陷，但是其编程风格偏向了NodeJS或GoLang，与原本的同步编程风格迥然相异。目前PHP的大部分主流应用框架依然是同步编程风格，所以一直在探索Swoole与同步编程结合的途径。
[lumen-swoole-http](https://github.com/breeze2/lumen-swoole-http)正是连接同步编程Lumen和异步编程Swoole的一座桥梁，有兴趣可以关注一下。

## LNMP的不足
LNMP是经典的Web应用架构组合，虽然（Linux、NginX、MySQL和PHP-FPM）四者各种是优秀的系统或软件，但是组合到一起的总体性能并不尽人意，明显的不是`1+1+1+1>4`，而是`4+3+2+1<1`。Linux系统无可厚非，主要问题出现在：

### 从NginX到PHP-FPM
NginX利用IO多路复用机制[epoll](https://en.wikipedia.org/wiki/Epoll)，极大地减少了IO阻塞等待，可以轻松应对[C10K](https://en.wikipedia.org/wiki/C10k_problem)。可是每次NginX将用户请求传递给PHP-FPM时，PHP-FPM总是需要从新加载PHP项目代码：创建执行环境，读取PHP文件和代码解析、编译等操作一次又一次的重复执行，造成不小的消耗。

### 从PHP-FPM到MySQL
由于PHP代码本身是同步执行，PHP-FPM连接MySQL查询数据时，只能空闲等待MySQL返回查询结果。一个查询语句执行时间可能会需要几秒钟，期间PHP-FPM若是能暂时放下当前用户慢查询请求，而去处理其他用户请求，效率必然有所提高。

<!--more-->

## Swoole HTTP服务器
[Swoole HTTP服务器](https://wiki.swoole.com/wiki/page/326.html)也采用了epoll机制，运行性能与NginX相比，虽不及，犹未远。不过Swoole HTTP服务器嵌入PHP中作为其一部分，可以直接运行PHP，完全可以取代NginX + PHP-FPM组合。

以目前流行的为框架[Lumen](https://lumen.laravel.com/)（Laravel的子框架）为例，用Swoole HTTP服务器运行Lumen项目十分简单，只需要在`$worker->onRequest($request, $response)`（收到用户请求）时将`$request`传给Lumen处理，`$response`再将Lumen的处理结果返回给用户，而且`$worker`的整个生命周期里只会加载一次Lumen项目代码，没有多余的磁盘IO和PHP代码编译的开销。

### 压力测试
在4GB+4Core的虚拟机下，测试HTTP服务器的静态输出：
* 2000客户端并发500000请求，不开启HTTP Keepalive，平均QPS：
```
NginX + HTML               QPS：25883.44
NginX + PHP-FPM + Lumen    QPS：828.36
Swoole + Lumen             QPS：13647.75
```

* 2000客户端并发500000请求，开启HTTP Keepalive，平均QPS：
```
NginX + HTML               QPS：86843.11
NginX + PHP-FPM + Lumen    QPS：894.06
Swoole + Lumen             QPS：18183.43
```

可以看出，`Swoole + Lumen`组合的执行效率远高于`NginX + PHP-FPM + Lumen`组合。

## 异步MySQL客户端
> 以上都是铺垫，以下才是整篇文章的重点😂😂😂

一个PHP应用要做的事不会是单纯的数据计算和数据输出，更多的是与数据库数据交互。以MySQL数据库为例，在只有一个PHP进程的情况，有10个用户同时请求执行`select sleep(1);`（耗时1秒）查询语句，若是使用MySQL同步查询，那么总耗时至少是10秒；若是使用MySQL异步查询，那么总耗时可能压缩到1到2秒内。

在PHP应用中能够实现数据库异步查询，才能更大的突破性能瓶颈。

虽然Swoole提供了[异步MySQL客户端](https://wiki.swoole.com/wiki/page/517.html)，但是其异步编程风格与Lumen这种同步编程风格的项目框架冲突，那么有没有可能在同步编程风格代码中调用异步MySQL客户端呢？

> 一开始我觉得这是不可能的，直到看了这篇文章：[Cooperative multitasking using coroutines (in PHP!)](http://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html)。当然，我看的是中文版：[ 在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html)，文中提到了PHP5.5加入的一个新功能：[yield](http://php.net/manual/en/language.generators.syntax.php)。

### Yield
`yield`是个动词，意思是“生成”，PHP中`yield`生出的东西叫`Generator`，意思是“生成器”😂😂😂。

个人理解是：yield将当前执行的上下文作为当前函数的结果返回（yield必须在函数中使用）。

在系统层面，各个进程的运行秩序由CPU调度；而有了yield，在PHP进程内，程序员可以自由调度各个代码块的执行顺序。比如，当“发现”当前用户请求的MySQL查询将会花费较多的时间，那么可以将当前执行上下文记录起来，交给异步MySQL客户端处理（与用户请求相关的`$request`和`$response`也传递过去），而主进程继续处理下一个用户请求。

### 约定声明
前面用了“发现”这个词，当然程序不可能智能地发现还没执行的查询语句将会是个慢查询，我们需要一些约定和声明。
Lumen框架是经典的MVC模式，我们约定C即Controller是处理用户请求的最后一步——Controller接受用户请求`$request`并返回响应`$response`。同时我们声明一个类，叫`SlowQuery`，这个类十分简单（具体请参见[SlowQuery.php](https://github.com/breeze2/lumen-swoole-http/blob/master/src/Database/SlowQuery.php)）：
```php
<?php
namespace BL\SwooleHttp\Database;

class SlowQuery
{
    public $sql = '';

    public function __construct($sql)
    {
        $this->sql    = $sql;
    }
}

```

比如，Lumen项目中有这么一个Controller：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;
use DB;

class TestController extends Controller
{
    public function test()
    {
        $a = DB::select('select sleep(1);');
        response()->json($a);
    }
}

```

上面的`DB::select`使用的同步MySQL客户端查询，我们用`SlowQuery`对象替换它：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;
use BL\SwooleHttp\Database\SlowQuery;

class TestController extends Controller
{
    public function test()
    {
        $a = yield new SlowQuery('select sleep(1);');
        response()->json($a);
    }
}

```

以Swoole HTTP服务器运行Lumen项目时，我们一定会获取Controller的返回结果。Controller的返回结果一般可以直接包装成Lumen响应返回给用户的，但返回结果若是一个生成器Generator对象，而且其当前值是一个慢查询SlowQuery对象的话，那么我们可以取出SlowQuery对象的sql属性，交由异步MySQL客户端执行；在异步查询的回调函数中将查询结果放回Generator对象存储的上下文中运行，得到最后结果才返回给用户；而主进程没有阻塞，可以继续处理其他用户请求。

当然，如果想用[Eloquent ORM](http://laravel.com/docs/eloquent)，那也很简单：我们先继承Lumen的Model，封装成一个新的Model类（具体参见[Model.php](https://github.com/breeze2/lumen-swoole-http/blob/master/src/Database/Model.php)），应用中的数据模型都继承于新的Model，Controller就可以这样写：
```php
<?php
namespace App\Http\Controllers;
use App\Http\Controllers\Controller;
use App\Models\User;
use DB;

class TestController extends Controller
{
    public function test()
    {
        $a = yield User::select(DB::raw('sleep(1)'))->yieldGet(); // 注意User须继承自\BL\SwooleHttp\Database\Model
        response()->json($a);
    }
}

```

以上三个Controller最终产出的用户响应都是一样的，不过后两者使用的是异步MySQL客户端，效率更高。

### 任务调度器
当然，我们还需要一个任务调度器来执行这些生成器，任务调度器的实现方法[ 在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html)文中“多任务协作”章节里有介绍，这里不展开。
Lumen框架中的代码保持了同步编程风格，而任务调度器中使用了异步编程风格来调用异步MySQL客户端。任务调度器是在Swoole HTTP服务器层面使用的，具体参见[Service.php](https://github.com/breeze2/lumen-swoole-http/blob/master/src/Service.php)。

### 连接限制
其实，每开启一个Swoole异步MySQL客户端，主进程就会新建一个线程连接MySQL，若是建立太多连接（线程），会增加自身服务器的压力，也会增加MySQL数据库服务器的压力。
这种利用yield来调用异步MySQL客户端处理慢查询而产生的线程，暂且称它为“慢查询协程”。
为了限制数据库连接数量，我们可以设置一个全局变量记录可新建慢查询协程的数量`MAX_COROUTINE`，开启一个异步MySQL客户端时让其减一，关闭一个异步MySQL客户端时让其加一；当用户请求慢查询时，`MAX_COROUTINE`大于0则由异步MySQL客户端处理，`MAX_COROUTINE`等于0时则由主进程“硬着头皮”自己处理。

### 压力测试
在4GB+4Core的虚拟机下，测试HTTP服务器与数据库读写：
#### 一般的快速查询和快速写入测试：
* 200并发50000请求读，利用HTTP Keepalive，平均QPS：
```
NginX + PHP-FPM + Lumen + MySQL    QPS：521.56
Swoole + Lumen + MySQL             QPS：7509.99
```
* 200并发50000请求写，利用HTTP Keepalive，平均QPS：
```
NginX + PHP-FPM + Lumen + MySQL    QPS：449.44
Swoole + Lumen + MySQL             QPS：1253.93
```

#### 慢查询协程测试：

* 16worker的Swoole HTTP服务器，并发执行`select sleep(1);`请求的最大效率是15.72rps；
* 16worker x 10coroutine的Swoole HTTP服务器，并发执行`select sleep(1);`请求的最大效率是151.93rps。

> 这里为什么说最大效率呢？因为当并发量远大于worker数目 x coroutine数目时，可开启慢查询协程的Swoole HTTP服务器的效率会逐渐跌向普通Swoole HTTP服务器。

`select sleep(1);`查询语句耗时1秒，每个用户请求都需要1秒时间来处理；不过，16进程的、每个进程可开启10个慢查询协程的Swoole HTTP服务器的每秒最多可以处理160个用户请求，而16进程的普通Swoole HTTP服务器每秒最多只能处理16个用户请求。

### 延伸

其实利用yield，我们还可以实现各种各样的“协程”。比如，[Swoole2.1版本](https://www.oschina.net/news/93248/swoole-2-1-released)已经开始支持go函数与通道，后续我们可能还可以将Lumen Controller中一些IO阻塞的操作的上下文移至go函数里执行，这样既保留了同步编程的风格，由达到异步执行的性能。

## 最后

以上理论，已经在[lumen-swoole-http](https://github.com/breeze2/lumen-swoole-http)项目中实现。
`lumen-swoole-http`是连接同步编程Lumen和异步编程Swoole的一座桥梁，可以帮助原生PHP的Lumen应用项目快速迁移到Swoole HTTP服务器上；当然也可以快速迁移回去😂。
有兴趣的同学可以尝试使用：
* [安装](https://breeze2.github.io/lumen-swoole-http/#/1_installation)
* [使用](https://breeze2.github.io/lumen-swoole-http/#/1_usage)
* [配置](https://breeze2.github.io/lumen-swoole-http/#/1_configuration)
* [慢查询协程](https://breeze2.github.io/lumen-swoole-http/#/3_coroutine_for_slow_query)
* ...
