title: 用Swoole实现一个简单的MySQL连接池
date: 2018-07-20 09:58:23
tags: 
    - swoole
    - mysql
    - php 
    - practice
categories:
    - 实践
---

> 最近Swoole发布了4.0.2版本，内置协程更加完善，利用Swoole\Coroutine\Channel和Swoole\Coroutine\MySQL可以轻松实现一个MySQL连接池。

<!--more-->

## 实现代码
直接看代码，脚本`mysql_connection_pool.php`：

```php
<?php
// mysql_connection_pool.php
use Swoole\Coroutine as Co;
use Swoole\Coroutine\Channel as CoChannel;
use Swoole\Coroutine\MySQL as CoMySQL;
use Swoole\Http\Server as HttpServer;

class MySqlCoroutine extends CoMySQL
{
    protected $used_at = null;
    public function getUsedAt()
    {
        return $this->used_at;
    }
    public function setUsedAt($time)
    {
        $this->used_at = $time;
    }
    public function isConnected()
    {
        return $this->connected;
    }
}

class MySqlManager
{
    protected $max_number;
    protected $min_number;
    protected $config;
    protected $channel;
    protected $number;
    private $is_recycling = false;

    /**
     * [__construct 构造函数]
     * @param array   $config         [MySQL服务器信息]
     * @param integer $max_number     [最大连接数]
     * @param integer $min_number     [最小连接数]
     */
    public function __construct(array $config, $max_number = 150, $min_number = 50)
    {
        $this->max_number = $max_number;
        $this->min_number = $min_number;
        $this->config     = $config;
        // $this->channel        = new CoChannel($max_number);
        $this->number = 0;
    }

    public function initChannel()
    {
        $this->channel = new CoChannel($this->max_number);
    }

    private function isFull()
    {
        return $this->number === $this->max_number;
    }

    private function isEmpty()
    {
        return $this->number === 0;
    }

    private function shouldRecover()
    {
        return $this->number > $this->min_number;
    }

    private function increase()
    {
        return $this->number += 1;
    }

    private function decrease()
    {
        return $this->number -= 1;
    }

    protected function build()
    {
        if (!$this->isFull()) {
            printf("%d do %s\n", time(), 'build one');
            $this->increase();
            $mysql = new MySqlCoroutine();
            $mysql->connect($this->config);
            $mysql->setUsedAt(time());
            return $mysql;
        }
        return false;
    }

    protected function rebuild(MySqlCoroutine $mysql)
    {
        printf("%d do %s\n", time(), 'rebuild one');
        $mysql->connect($this->config);
        $mysql->setUsedAt(time());
        return $mysql;
    }

    protected function destroy(MySqlCoroutine $mysql)
    {
        if (!$this->isEmpty()) {
            printf("%d do %s\n", time(), 'destroy one');
            $this->decrease();
            return true;
        }
        return false;
    }

    public function push(MySqlCoroutine $mysql)
    {
        if (!$this->channel->isFull()) {
            printf("%d do %s\n", time(), 'push one');
            $this->channel->push($mysql);
            return;
        }
    }

    public function pop()
    {
        if ($mysql = $this->build()) {
            return $mysql;
        }
        $mysql = $this->channel->pop();
        $now   = time();
        printf("%d do %s\n", time(), 'pop one');
        if (!$mysql->isConnected()) {
            return $this->rebuild($mysql);
        }
        $mysql->setUsedAt($now);
        return $mysql;
    }

    /**
     * [autoRecycling 自动回收连接]
     * @param  integer $timeout [连接空置时限]
     * @param  integer $sleep   [循环检查的时间间隔]
     * @return null             [null]
     */
    public function autoRecycling($timeout = 200, $sleep = 20)
    {
        if (!$this->is_recycling) {
            $this->is_recycling = true;
            while (1) {
                Co::sleep($sleep);
                if ($this->shouldRecover()) {
                    $mysql = $this->channel->pop();
                    $now   = time();
                    if ($now - $mysql->getUsedAt() > $timeout) {
                        printf("%d do %s\n", time(), 'recover one');
                        $this->decrease();
                    } else {
                        !$this->channel->isFull() && $this->channel->push($mysql);
                    }
                }
            }
        }
    }

}

$server = new HttpServer('127.0.0.1', 9501, SWOOLE_BASE);

$server->set([
    'worker_num' => 1,

]);

$manager = new MySqlManager([
    'host'     => '127.0.0.1',
    'port'     => 3306,
    'user'     => 'root',
    'password' => '',
    'database' => 'test',
    'timeout'  => -1,

], 4, 2);

$server->on('workerStart', function ($server) use ($manager) {
    $manager->initChannel();
    $manager->autoRecycling(4, 2); // 启动自动回收
});

$server->on('request', function ($request, $response) use ($server, $manager) {
    $mysql = $manager->pop(); // 取出一个MySQL连接
    $mysql->query('select sleep(1);');
    $manager->push($mysql); // 返回一个MySQL连接
    $response->end(json_encode($server->stats()));

});

$server->start();

```

## 运行效果

运行脚本：
```bash
$ php mysql_connection_pool.php
```

ab测试：
```bash
$ ab -c 8 -n 8 http://127.0.0.1:9501/
```

间隔10秒后，再次测试：
```bash
$ ab -c 8 -n 8 http://127.0.0.1:9501/
```

输出结果：
```bash
1532083141 do build one
1532083141 do build one
1532083141 do build one
1532083141 do build one
1532083142 do push one
1532083142 do push one
1532083142 do push one
1532083142 do pop one
1532083142 do pop one
1532083142 do pop one
1532083142 do push one
1532083142 do pop one
1532083143 do push one
1532083143 do push one
1532083143 do push one
1532083143 do push one
1532083147 do recover one
1532083149 do recover one
1532083160 do build one
1532083160 do build one
1532083160 do pop one
1532083160 do rebuild one
1532083160 do pop one
1532083160 do rebuild one
1532083161 do push one
1532083161 do pop one
1532083161 do push one
1532083161 do push one
1532083161 do push one
1532083161 do pop one
1532083161 do pop one
1532083161 do pop one
1532083162 do push one
1532083162 do push one
1532083162 do push one
1532083162 do push one
1532083166 do recover one
1532083168 do recover one
```

## 代码解释
首先代码里定义了两个类，分别是：
* MySqlCoroutine
* MySqlManager

### MySqlCoroutine
MySqlCoroutine继承自`Swoole\Coroutine\MySQL`，主要是为了增加一个属性`used_at`和两个与之配套的`get/set`方法：`getUsedAt`和`setUsedAt`。`used_at`是个时间戳，记录MySqlCoroutine的被调用的时间点。而`isConnected`方法是判断与服务器的连接是否有效。

### MySqlManager
每个MySqlCoroutine实例与MySQL服务器维系一个连接，而MySqlManager来管理这些MySqlCoroutine，即管理整个连接池。
构建MySqlManager时，用到三个参数：
* `config`：用于构建MySqlCoroutine；
* `max_number`：最大连接数；
* `min_number`：最小连接数。

其实这里说的连接池，就是MySqlManager的`mysql_channel`属性，实际的数据类型是`Swoole\Coroutine\Channel`。而池内连接的数量，会用`number`属性记录下来。

#### 内部方法：build/rebuild/destroy

* build方法：如果当前连接的数量未达到最大值，则新建一个MySqlCoroutine实例作返回值，并且`number`属性加1；否则，返回false。
* rebuild方法：传入一个MySqlCoroutine实例，返回一个新的MySqlCoroutine实例。
* destroy方法：传入一个MySqlCoroutine实例，如果`number`大于0，那么`number`减1。

#### 公有方法：pop/push/autoRecycling

* pop方法：先执行build方法，若是返回MySqlCoroutine实例，则直接返回这个MySqlCoroutine实例；否则，从`mysql_channel`里`pop`（如果`mysql_channel`已经空了，当前协程会被挂起，等待其他协程往`mysql_channel`放入MySqlCoroutine实例），若是取出的MySqlCoroutine实例的连接已无效（连接可能已经被MySQL服务器自动断开），那就返回rebuild方法的值，否则直接返回这个MySqlCoroutine实例。
* push方法：如果`mysql_channel`未满，就往`mysql_channel`放入一个MySqlCoroutine；否则，什么也不做。
* autoRecycling方法：接收两个参数，`$timeou`和`$sleep`；每隔`$sleep`秒，就检查一次，当前连接的数量超过了最小值，那么就从`mysql_channel`里`pop`一个MySqlCoroutine实例，若是MySqlCoroutine实例上一次使用时间与现在时间间隔超过了`$timeou`秒，说明这个MySqlCoroutine实例没有被频繁使用，可以回收；所谓回收，就是从`mysql_channel`取出后，不再放回，并且`number`减1。

### 调用
在`Swoole\Http\Server`的`workerStart`回调函数里，让MySqlManager实例开始自动回收MySqlCoroutine实例；而在`Swoole\Http\Server`的`request`回调函数里，需要MySqlCoroutine实例时，调用MySqlManager实例的pop方法从连接池里获取，用完再调用MySqlManager实例的push方法放回连接池。

## 最后

就这样，利用Swoole扩展，几十行PHP代码就能实现一个连接间可以并行使用，池内有最大/最小数量限制，有自动回收空闲连接机制的MySQL连接池。



