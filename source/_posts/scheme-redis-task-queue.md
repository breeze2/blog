title: 基于Redis的任务调度设计方案
date: 2018-01-08 09:23:02
tags: 
    - redis
    - php
    - queue
    - scheme
categories:
    - 方案
---

> 一个网关服务器就跟快餐店一样，总是希望客人来得快、去得也快，这样在相同时间内才可以服务更多的客人。如果快餐店的服务员在一个顾客点餐、等餐和结账时都全程跟陪的话，那么这个服务员大部分时间都是在空闲的等待。应该有专门的服务员负责点餐，专门的服务员负责送餐，专门的服务员负责结账，这样才能提高效率。同样道理，网关服务器中也需要分工明确。举个例子：

假设有一个申请发送重置密码邮件的网关接口，须知道发送一封邮件可能会花费上好几秒钟，如果网关服务器直接在线上给用户发送重置密码邮件，高并发的情况下就很容易造成网络拥挤。但实际上，网关服务器并非一定要等待邮件发送成功后才能响应用户，完全可以先告知用户邮件会发送的，而后再在线下把邮件发送出去（就像快餐店里点餐的服务员跟顾客说先去找位置坐，饭菜做好后会有人给他送过去）。

那么是谁来把邮件发送出去呢？

## 任务队列
为了网关接口能够尽快响应用户请求，无需即时知道结果的耗时操作可以交由任务队列机制来处理。
任务队列机制中包含两种角色，一个是任务生产者，一个是任务消费者，而任务队列是两者之间的纽带：
* 生产者往队列里放入任务；
* 消费者从队列里取出任务。

任务队列的整体运行流程是：任务生产者把当前操作的关键信息（后续可以根据这些信息还原出当前操作）抽象出来，比如发送重置密码的邮件，我们只需要当前用户邮箱和用户名就可以了；任务生产者把任务放进队列，实际就是把任务的关键信息存储起来，这里会用到MySQL、Redis之类数据存储工具，常用的是Redis；而任务消费者就不断地从数据库中取出任务信息，逐一执行。

任务生产者的工作是任务分发，一般由线上的网关服务程序执行；任务消费者的工作是任务调度，一般由线下的程序执行，这样即使任务耗时再多，也不阻塞网关服务。

这里主要讨论的是任务调度（任务消费者）的程序设计。

<!--more-->

## 简单直接
假设我们用Redis列表List存储任务信息，列表键名是`queues:default`，任务发布就是往列表`queues:default`后追加数据：
```php
<?php
// PHP伪代码
    Redis::rpush('queues:default', serialize($task));
```
那么任务调度可以这样简单直接的实现：
```php
<?php
// PHP伪代码
Class Worker {

    public function schedule() {
        while(1) {
            $seri = Redis::lpop('queues:default');
            if($seri) {
                $task = unserialize($seri);
                $this->handle($task);
                continue;
            }
            sleep(1);
        }
    }

    public function handle($task) {
        // do something time-consuming
    }
}

$worker = new Worker;
$worker->schedule();
```

## 意外保险
上面代码是直接从`queues:default`列表中移出第一个任务（lpop），因为`handle($task)`函数是一个耗时的操作，过程中若是遇到什么意外导致了整个程序退出，这个任务可能还没执行完成，可是任务信息已经完全丢失了。保险起见，对`schedule()`函数进行以下修改：
```php
<?php
...
    public function schedule() {
        while(1) {
            $seri = Redis::lindex('queues:default', 0);
            if($seri) {
                $task = unserialize($seri);
                $this->handle($task);
                Redis::lpop('queues:default');
                continue;
            }
            sleep(1);
        }
    }
...
```
即在任务完成后才将任务信息从列表中移除。

## 延时执行
`queues:default`列表中的任务都是需要即时执行的，但是有些任务是需要间隔一段时间后或者在某个时间点上执行，那么可以引入一个有序集合，命名为`queues:default:delayed`，来存放这些任务。任务发布时需要指明执行的时间点`$time`：
```php
<?php
// PHP伪代码
    Redis::zadd('queues:default:delayed', $time, serialize($task));
```
任务调度时，如果`queues:default`列表已经空了，就从`queues:default:delayed`集合中取出到达执行时间的任务放入`queues:default`列表中：
```php
<?php
...
    public function schedule() {
        while(1) {
            $seri = Redis::lindex('queues:default', 0);
            if($seri) {
                $task = unserialize($seri);
                $this->handle($task);
                Redis::lpop('queues:default');
                continue;
            }
            $seri_arr = Redis::zremrangebyscore('queues:default:delayed', 0, time());
            if($seri_arr) {
                Redis::rpush('queues:default', $seri_arr);
                continue;
            }
            sleep(1);
        }
    }
...
```

## 任务超时
预估任务正常执行所需的最大时间值，若是任务执行超过了这个时间，可能是过程中遇到一些意外，如果任由它继续卡着，那么后面的任务就会无法被执行了。
首先我们给任务设定一个时限属性`timeout`，然后在执行任务前先给进程本身设置一个闹钟信号，`timeout`后收到信号说明任务执行超时，需要退出当前进程（用[supervisor](http://supervisord.org/)守护进程时，进程自身退出，supervisor会自动再拉起）。
注意：`pcntl_alarm($timeout)`会覆盖之前闹钟信号，而`pcntl_alarm(0)`会取消闹钟信号；任务超时后，当前任务放入`queues:default:delayed`集合中延时执行，以免再次阻塞队列。
```php
<?php
...
    public function schedule() {
        while(1) {
            $seri = Redis::lindex('queues:default', 0);
            if($seri) {
                $task = unserialize($seri);
                $this->timeoutHanle($task);
                $this->handle($task);
                Redis::lpop('queues:default');
                continue;
            }
            $seri_arr = Redis::zremrangebyscore('queues:default:delayed', 0, time());
            if($seri_arr) {
                Redis::rpush('queues:default', $seri_arr);
                continue;
            }
            pcntl_alarm(0);
            sleep(1);
        }
    }

    public function timeoutHanle($task) {
        $timeout = (int)$task->timeout;
        if ($timeout > 0) {
            pcntl_signal(SIGALRM, function () {
                $seri = Redis::lpop('queues:default');
                Redis::zadd('queues:default:delayed', time()+10), $seri);
                posix_kill(getmypid(), SIGKILL);
            });
        }
        pcntl_alarm($timeout);
    }
...
```

## 并发执行
上面代码，直观上没什么问题，但是在多进程并发执行的时候，有些任务可能会被重复执行，是因为没能及时将当前执行的任务从`queues:default`列表中移出，其他进程也可以读取到。为了避免重复执行的问题，我们需要引入一个有序集合SortedSet存放正在执行的任务，命名为`queues:default:reserved`。
首先任务是从`queues:default`列表中直接移出，然后开始执行任务前先把任务放进`queues:default:reserved`集合中，任务完成了再从`queues:default:reserved`集合中移出。
再结合任务超时，假设一个任务执行时间不可能超过`60*60`秒（可以按需调整），在`queues:default`列表为空的时候，`queues:default:reserved`集合中有任务已经存放超过了`60*60`秒，那么有可能是某些进程在执行任务是意外退出了，所以把这些任务放到`queues:default:delayed`集合中稍后执行。
```php
<?php
...
    public function schedule() {
        while(1) {
            $seri = Redis::lpop('queues:default', 0);
            if($seri) {
                Redis::zadd('queues:default:reserved', time()+10, $seri);
                $task = unserialize($seri);                
                $this->timeoutHanle($task);
                $this->handle($task);
                Redis::zrem('queues:default:reserved', $seri);
                continue;
            }
            $seri_arr = Redis::zremrangebyscore('queues:default:delayed', 0, time());
            if($seri_arr) {
                Redis::rpush('queues:default', $seri_arr);
                continue;
            }
            $seri_arr = Redis::zremrangebyscore('queues:default:reserved', 0, time()-60*60);
            if($seri_arr) {
                foreach($seri_arr as $seri) {
                    Redis::zadd('queues:default:delayed', time()+10, $seri);
                }
            }

            sleep(1);
        }
    }

    public function timeoutHanle($task) {
        $timeout = (int)$task->timeout;
        if ($timeout > 0) {
            pcntl_signal(SIGALRM, function () use ($task) {
                $seri = serialize($task);
                Redis::zrem('queues:default:reserved', $seri);
                Redis::zadd('queues:default:delayed', time()+10), $seri);
                posix_kill(getmypid(), SIGKILL);
            });
        }
        pcntl_alarm($timeout);
    }
...
```

## 其他
### 失败重试
以上代码没有检验任务是否执行成功，应该有任务失败的处理机制：比如给任务设定一个最多重试次数属性`retry_times`，任务每执行一次`retry_times`，任务执行失败时，若是`retry_times`等于0，则将任务放入`queues:default:failed`列表中不再执行；否则放入放到`queues:default:delayed`集合中稍后执行。

### 休眠时间
以上代码是进程忙时连续执行，闲时休眠一秒，可以按需调整优化。

### 事件监听
若是需要在任务执行成功或失败时进行某些操作，可以给任务设定成功操作方法`afterSucceeded()`或失败操作方法`afterFailed()`，在相应的时候回调。

## 最后
以上讲述了一个任务调度程序的逐步演变，设计方案很大程度上参考了[Laravel Queue](https://laravel.com/docs/5.5/queues)。
用工具，知其然，知其所以然。

