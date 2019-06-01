title: PHP代码静态分析工具PHPStan
date: 2018-08-08 17:31:23
tags: 
    - php 
    - practice
categories:
    - 实践
---

> 最近发现自己写的PHP代码运行结果总跟自己预想的不一样，排查时发现大多是语法错误，在运行之前错误已经种下。可能是自己粗心大意，或者说`php -l`检测太简单，不过的确是有一些语法错误埋藏得太深（毕竟PHP是动态语言），那么有没有办法，在代码代码正式运行之前，把语法错误全找出来呢？

这里介绍一款PHP代码静态分析工具：[PHPStan](https://github.com/phpstan/phpstan)，不需要运行代码，也可以对代码进行严格的语法检测，尽量将代码运行错误率降到最低。

## PHPStan
### 安装
目前，PHPStan`V0.10.2`要求系统环境的PHP版本不低于7.1。用Composer全局安装：
```bash
$ composer global require phpstan/phpstan
```
<!--more-->

### 使用
PHPStan静态分析的使用方法十分简单：
```bash
$ phpstan analyse [-c|--configuration CONFIGURATION] [-l|--level LEVEL] [--no-progress] [--debug] [-a|--autoload-file AUTOLOAD-FILE] [--errorFormat ERRORFORMAT] [--memory-limit MEMORY-LIMIT] [--] [<paths>]...
```

* configuration：运行配置文件的路径；
* level：严格级别，0-7，越大越严格；
* no-progress：不显示进度；
* debug：debug模式；
* autoload-file：自动加载文件的路径；
* errorFormat：错误格式；
* memory-limit：内存限制；
* paths：待分析的文件路径。

比如，分析一个PHP文件：
```bash
$ phpstan analyse --level=7 --autoload-file=/PATH/TO/vendor/autoload.php /PATH/TO/someone.php
```

## PHPStan in VSCode

当然，语法分析应该是编辑器做的事，写完代码还要切换到命令终端执行`phpstan`，未免过于繁琐。所以这里推荐一款VSCode扩展：[PHP Static Analysis](https://marketplace.visualstudio.com/items?itemName=breezelin.phpstan)。

![PHP Static Analysis](/assets/images/practice-php-static-analysis-tool-phpstan1.jpg)

首先，用Composer全局安装PHPStan；然后，在VSCode的扩展管理中搜索`PHP Static Analysis`，安装第一个匹配的扩展；重载VSCode重载窗口后，扩展会自动分析VSCode下打开的PHP文件。

运行效果：
![运行效果](/assets/images/practice-php-static-analysis-tool-phpstan2.jpg)

比如，声明了一个变量未调用，调用了一个未声明的变量和调用了一个未定义的方法等等这样错误都会被检测出了。

不过，宽松一点地来说，其实`$this->array()`方法是存在的，只是通过魔术方法`__call()`实现的。

## PHPStan with Laravel

高严格级别的PHPStan检测到调用未声明的类方法时，会报告类中方法不存在的错误，即使这个类定义了`__call()`或`__callStatic()`。

很多应用框架为了优雅，大量使用了魔术方法，比如[Laravel](https://laravel.com/)。
用PHPStan检测Laravel项目，自然会报告很多调用未声明类方法的错误，对于这个问题，可以借助[laravel-ide-helper](https://github.com/barryvdh/laravel-ide-helper)来降低误报。

### 安装laravel-ide-helper

```bash
$ cd /PATH/TO/LARAVEL_PROJECT
$ composer require barryvdh/laravel-ide-helper
```

### 注入LaravelIdeHelper
编辑`app/Providers/AppServiceProvider.php`里的注册方法：

```php
<?php
    ...
    public function register()
    {
        if ($this->app->environment() !== 'production') {
            $this->app->register(\Barryvdh\LaravelIdeHelper\IdeHelperServiceProvider::class);
        }
        // ...
    }
```

### 生成_ide_helper.php

```bash
$ cd /PATH/TO/LARAVEL_PROJECT
$ php artisan ide-helper:generate
```

这时，Laravel框架中的Facade类，原本通过`__callStatic()`获取的静态方法，全部在_ide_helper.php声明了，在PHPStan检测Laravel项目代码时引入`_ide_helper.php`文件，就可以减少误报。

### PHPStan配置

在Laravel项目的根目录下，新建`phpstan.neon`文件：
```neon
parameters:
    autoload_files:
        - %currentWorkingDirectory%/_ide_helper.php
```

在Laravel项目的根目录下，执行`phpstan`命令时，会自动使用`phpstan.neon`这个配置。

## 最后

代码的语法错误，应该在编写的时候及时发现，尽量减少正式运行时错误。

