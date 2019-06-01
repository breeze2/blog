title: Swoole连接池在Laravel中的使用
date: 2017-09-20 13:42:13
tags: 
    - php
    - practice
categories:
    - 实践
---

## Swoole php-cp

[php-cp](https://github.com/swoole/php-cp)是Swoole组织开发的一个PHP扩展，可以本地代理MySQL、Redis连接，提供连接池，读写分离，负载均衡，慢查询日志，大数据块日志等功能。相比原生PHP的数据库连接，php-cp连接池可以缓存连接，免去一些重复新建、回收数据库连接带来的时间消耗和IO消耗；并且php-cp连接定时ping数据库，使得连接不会太久没活动而被回收；在连接数超过限额时，php-cp提供了排队机制，而不是直接拒绝，所以php-cp是值得高并发的PHP项目引入使用的。
php-cp提供了代替PDO的类：pdoProxy，所以在主流PHP框架中引入php-cp是十分简便的。在php-cp的README中，提供了Yii、CI和ThinkPHP等框架的集成样例，但是少了Laravel，所以这里介绍一下在Laravel项目中怎样集成php-cp。

<!--more-->

## 安装php-cp扩展

首先，下载[php-cp](https://github.com/swoole/php-cp)源码，编译安装：

```cmd
$ cd tmp 
$ git clone https://github.com/swoole/php-cp.git
$ cd php-cp
$ phpize
$ ./configure
$ make
$ sudo make install  
```

编译成功后，将`extension=connect_pool.so`添加到php-cli和php-fpm的php.ini配置文件中，这样PHP启动的时候就会加载connect_pool扩展。

## 启动pool-server

php-cp提供了现成的连接池脚本，只需要按照以下配置，便能启动pool-server连接池服务：

```cmd
$ sudo cp ./config.ini.example /etc/pool.ini    #根据需求修改配置内容
$ sudo mkdir -m 755 /var/log/php-connection-pool    #创建日志目录 目录文件夹不存在或没权限会导致日志写不起
$ chmod +x ./pool_server    #x权限git已经设置 为稳妥再设置一次 pool_server为php脚本 可自行修改
$ [ -f /bin/env ] || sudo ln -s /usr/bin/env /bin/env    #deb系的系统(如debian、ubuntu)env的路径为/usr/bin/env做软链接兼容处理
$ sudo cp ./pool_server /usr/local/bin/pool_server
$ sudo pool_server start    #启动服务 如果配置文件的daemonize开启则后台运行 否则为前台运行 Ctrl+c结束服务
$ sudo pool_server stop    #停止服务
$ sudo pool_server restart    #重启服务
$ sudo pool_server status    #查看服务状态
```

`/etc/pool.ini`是`pool_server`脚本指定的配置路径，大家可以根据自己的需求，调整`/etc/pool.ini`里面的配置参数。比如，PDO数据源，Laravel的连接格式一般是：

```conf
['mysql:host=127.0.0.1;port=3306;dbname=forge']
```

若是请求的连接在配置中找不到对应的数据源，pool_server会自动新建一个数据源，新的数据源信息可以通过`sudo pool_server status`查看。

## Laravel集成

Laravel集成php-cp的代码我已经上传到github，[laravel-swoole-cp](https://github.com/breeze2/laravel-swoole-cp)，主要是三个文件：
1. laravel/framework/src/Illuminate/Database/MySqlSwooleProxyConnection.php
2. laravel/framework/src/Illuminate/Database/Connectors/MySqlSwooleProxyConnector.php
3. laravel/framework/src/Illuminate/Database/Connectors/ConnectionFactory.php

只要将这三个文件放到Laravel项目的vendor文件夹下便可。`MySqlSwooleProxyConnection.php`和`MySqlSwooleProxyConnector.php`是模仿Laravel框架自有的`MySqlConnection.php`和`MySqlConnector.php`新增编写的；而`ConnectionFactory.php`则是在Laravel框架原有的基础上修改的，主要是调用`MySqlSwooleProxyConnection`和`MySqlSwooleProxyConnector`这两个新增类。

`ConnectionFactory.php`修改的地方：

```php
<?php
namespace Illuminate\Database\Connectors;

use Illuminate\Database\MySqlSwooleProxyConnection;
...
class ConnectionFactory
{
    ...
    public function createConnector(array $config)
    {
        if (! isset($config['driver'])) {
            throw new InvalidArgumentException('A driver must be specified.');
        }

        if ($this->container->bound($key = "db.connector.{$config['driver']}")) {
            return $this->container->make($key);
        }

        switch ($config['driver']) {
            case 'mysql-cp':
                return new MySqlSwooleProxyConnector;
            case 'mysql':
                return new MySqlConnector;
            case 'pgsql':
                return new PostgresConnector;
            case 'sqlite':
                return new SQLiteConnector;
            case 'sqlsrv':
                return new SqlServerConnector;
        }

        throw new InvalidArgumentException("Unsupported driver [{$config['driver']}]");
    }

    protected function createConnection($driver, $connection, $database, $prefix = '', array $config = [])
    {
        if ($resolver = Connection::getResolver($driver)) {
            return $resolver($connection, $database, $prefix, $config);
        }

        switch ($driver) {
            case 'mysql-cp':
                return new MySqlSwooleProxyConnection($connection, $database, $prefix, $config);
            case 'mysql':
                return new MySqlConnection($connection, $database, $prefix, $config);
            case 'pgsql':
                return new PostgresConnection($connection, $database, $prefix, $config);
            case 'sqlite':
                return new SQLiteConnection($connection, $database, $prefix, $config);
            case 'sqlsrv':
                return new SqlServerConnection($connection, $database, $prefix, $config);
        }

        throw new InvalidArgumentException("Unsupported driver [$driver]");
    }
}
```

注意，这里将php-cp的数据库连接驱动命名为`mysql-cp`，所以在Laravel项目的数据库配置`config/database.php`里应该这样配置php-cp的连接驱动：

```php
<?php
// config/database.php

return [
    'default' => env('DB_CONNECTION', 'mysql-cp'),

    'connections' => [

        'mysql-cp' => [
            'driver' => 'mysql-cp',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'forge'),
            'username' => env('DB_USERNAME', 'forge'),
            'password' => env('DB_PASSWORD', ''),
            'unix_socket' => env('DB_SOCKET', ''),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'strict' => true,
            'engine' => null,
        ],
        ...
    ],
]

```


## 问题来了
一切准备就绪，执行一句`DB::table('users')->find();`，却报错了（我好像明白了为什么没人提供php-cp的Laravel连接驱动）：
![FatalErrorException](/assets/images/practice-swoole-connection-pool-in-laravel1.png)

### 解决办法
经过排查，发现当数据获取模式设为`PDO::FETCH_OBJ`，pool_server的worker就会退出，其他获取模式就运行正常，那么应该是变量序列化传输的问题。原来php-cpV1.5.0版本（目前最新版）针对PHP7环境，使用自家的[swoole_serialize](https://github.com/swoole/swoole_serialize)序列化方法，而我的系统正好装的是PHP7，于是想：用相对主流的序列化方法[msgpack-php](https://github.com/msgpack/msgpack-php)替换swoole_serialize的话可能会解决这个问题。

替换后，重新编译安装php-cp，果然问题解决了。替换后的php-cp源码我已经上传到github，[breeze2/php-cp](https://github.com/breeze2/php-cp)，若有需要可以下载安装。

## 最后

php-cp主要作用是应对高并发，假设MySQL数据库的最大连接数`max_connections`是100，当请求连接超过100的时候，若是PHP直接访问数据库，MySQL会直接拒绝多出来的连接；而通过php-cp访问数据库，php-cp有排队机制应对多出来的连接，php-cp的连接池可以长驻内存，也免去了大量的数据库连接线程新建、回收带来的消耗。
综合来说，一般并发量不高的网站也用不上php-cp；而对于大型高并发的网站，使用数据库中间件更为合理，因为php-cp是本地代理，有些限制了单机效能。只是看到php-cp没有相应的Laravel连接驱动，强迫症发作，写了一个而已。


