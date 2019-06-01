title: 动手写一个简单的PHP扩展
date: 2015-12-26 19:39:19
tags:
    - php
categories:
    - 躬行
---



> PHP基于C语言编写，支持自定义扩展，扩展自然也是基于C语言编写咯。大学里学（水）过C语言，写一个简单的PHP扩展，应该没问题吧。

## 运行环境

- OS：Linux
- PHP：5.6
- phpize：20131226

----------

## 新建项目
### 标准的自动生成

首先要准备一份 [PHP源代码](http://cn2.php.net/distributions/php-5.6.16.tar.gz) 。将源代码包解压后，在ext目录里（PHP自带扩展的源代码就存放在这里），可以找到一个名字叫做`ext_skel`的可运行文件。这个`ext_skel`文件就是PHP提供给我们来创建扩展项目的，用法：

```cmd
$ cd ext/
$ ./ext_skel --extname=my_ext_name
```

执行如上命令后，当前路径下就会多了一个名字叫`my_ext_name`的目录，里面存放着一个规范的PHP扩展项目代码，这些都是`ext_skel`自动生成的。主要的文件是这几个：

 - config.m4（对应unix）
 - config.w32（对应windows）
 - my_ext_name.c
 - php_my_ext_name.h

如果对扩展结构已经足够熟悉，`./ext_skel`命令后面带上`--no-help`参数，自动生成的代码中就不会出现多余的注释。

<!--more-->

### 手动创建
当然，没有`ext_skel`，也可以自己手动建立一个PHP扩展项目，先新建一个目录来存放代码：

```cmd
$ mkdir my_ext_name
$ cd my_ext_name
```

然后新建一个 [config.m4](http://php.net/manual/zh/internals2.buildsys.configunix.php) 文件，写入如下配置：

```cmd
//config.m4
PHP_ARG_ENABLE(my_ext_name, whether to enable my_ext_name support,
[  --enable-my_ext_name           Enable my_ext_name support])

if test "$PHP_MY_EXT_NAME" != "no"; then
  PHP_SUBST(MY_EXT_NAME_SHARED_LIBADD)
  PHP_NEW_EXTENSION(my_ext_name, my_ext_name.c, $ext_shared)
fi
```

解释一下：

 1. `PHP_ARG_ENABLE`函数是用来配置扩展的工作方式的，如果该扩展依赖其他扩展，应该使用`PHP_ARG_WITH`函数（这里不用）；
 2. `PHP_SUBST`函数将变量输出到由`configure`生成的文件中；
 3. `PHP_NEW_EXTENSION`函数声明了该扩展的名称（如`my_ext_name`）、需要的源文件名（如`my_ext_name.c`）、此扩展的编译形式（`$ext_shared`）。`$ext_shared`这个参数表明该扩展不是一个静态模块，而是在PHP运行时动态加载的。
更多内容参见《PHP扩展开发与内核应用》的 [第五章](http://www.walu.cc/phpbook/5.md) 。

----------

## 编写函数
接下来，把想要实现的功能写入扩展中。写点什么好呢？其实我想写一个友好时间转化的函数，这是一个怎样的函数？用PHP语言表达会直观点：

```php
function php_friendly_time($time) {
    $arg_time = null;
    $now_time = null;
    $dif_time = null;
    $ret_time = null;
    $timeinfo = null;
    
    if(is_int($time)) {
        $arg_time = $time;
    } else if(is_string($time)) {
        $arg_time = strtotime($time);
    } else {
        return null;
    }
    
    $now_time = time();
    $dif_time = $now_time - $arg_time;
    if($dif_time<60) {
        $ret_time = 'just now';
    } else if($dif_time<120) {
        $ret_time = 'one minute ago';
    } else if($dif_time<3600) {
        $ret_time = (int)($dif_time/60) . ' minutes ago';
    } else if($dif_time<7200) {
        $ret_time = 'one hour ago';
    } else if($dif_time<86400) {
        $ret_time = (int)($dif_time/3600) . ' hours ago';
    } else if($dif_time<172800) {
        $timeinfo = getdate($arg_time);
        $ret_time = 'one day ago ' . $timeinfo['hours'] . ':' . $timeinfo['minutes'];
    } else if($dif_time<1209600) {
        $timeinfo = getdate($arg_time);
        $ret_time = (int)($dif_time/86400) . ' days ago ' . $timeinfo['hours'] . ':' . $timeinfo['minutes'];
    } else {
        $ret_time = date("Y-m-d H:i:s", $arg_time);
    }

    return $ret_time;
}
```
好，想表达的大概就是这个意思，那么用C语言怎么在PHP扩展实现呢？

### 头文件
按照C语言开发规范，我们先创建一个头文件`php_my_ext_name.h`，这里主要是做一些宏定义操作：

```c
//php_my_ext_name.h

#ifndef PHP_MY_EXT_NAME_H

#define PHP_MY_EXT_NAME_H
#define phpext_my_ext_name_ptr &my_ext_name_module_entry
#define PHP_MY_EXT_NAME_VERSION "0.1.0"

extern zend_module_entry my_ext_name_module_entry;

#endif
```


### 程序文件
创建一个程序文件`my_ext_name.c`，里面要写的代码大概是这样的：

```c
//my_ext_name.c

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_my_ext_name.h"

PHP_FUNCTION(friendly_time)
{
    ...
}

const zend_function_entry my_ext_name_functions[] = {
    PHP_FE(friendly_time, NULL)
    PHP_FE_END
};

zend_module_entry my_ext_name_module_entry = {
    STANDARD_MODULE_HEADER,
    "my_ext_name",
    my_ext_name_functions,
    NULL, /* MINIT */
    NULL, /* MSHUTDOWN */
    NULL, /* RINIT */
    NULL, /* RSHUTDOWN */
    NULL, /* MINFO */
    PHP_MY_EXT_NAME_VERSION,
    STANDARD_MODULE_PROPERTIES
};

#ifdef COMPILE_DL_MY_EXT_NAME
ZEND_GET_MODULE(my_ext_name)
#endif
```

主要是这几个步骤：

- 首先，加载一些需要的头文件；
- 用`PHP_FUNCTION`这个宏函数定义想要在PHP中实现的扩展函数，参数名称将会是在PHP中调用的函数名称，如：`friendly_time`；
- 然后在一个`zend_function_entry`结构体数组中，用`PHP_FE`（也是一个宏函数）注册刚刚定义的扩展函数，如：`friendly_time`；
- 接下来，就是在一个`zend_module_entry`结构体中填写扩展模块的入口信息，当然这个结构体的名字要跟扩展名对应，如：`my_ext_name_module_entry`；
- 最后，判断一下这个扩展模块是否被动态链接，如果是，就执行`ZEND_GET_MODULE`宏函数（在这里一定是）。

简单的介绍一下`PHP_FUNCTION`，他的宏定义如下：

```c
#define PHP_FUNCTION ZEND_FUNCTION
#define ZEND_FUNCTION(name)     ZEND_NAMED_FUNCTION(ZEND_FN(name))
#define ZEND_NAMED_FUNCTION(name) void name(INTERNAL_FUNCTION_PARAMETERS)
#define ZEND_FN(name) zif_##name
```

所以呢，`PHP_FUNCTION(friendly_time)`最终会被转化成：

```c
void zif_friendly_time(INTERNAL_FUNCTION_PARAMETERS)
```

这样子看，代码是不是觉得熟悉多了？这才是正常的C代码啊！大家有没有注意到`my_ext_name_functions`数组里没有`,`分隔？来看看`PHP_FE`的宏定义：

```c
#define ZEND_FE(name, arg_info) ZEND_FENTRY(name, ZEND_FN(name), arg_info, 0)
#define ZEND_FENTRY(zend_name, name, arg_info, flags) { #zend_name, name, arg_info, (zend_uint) (sizeof(arg_info)/sizeof(struct _zend_arg_info)-1), flags },
```

于是，`PHP_FE(friendly_time, NULL)`最终会被转化成：

```c
{"friendly_time",zif_walu_hello,NULL, (zend_uint) (sizeof(NULL)/sizeof(struct _zend_arg_info)-1), 0 },
```

可以看到`,`已经带上了。
如果扩展模块里有多个函数，可以继续使用`PHP_FUNCTION`来定义，如`PHP_FUNCTION(more_function){...}`；然后在`my_ext_name_functions`数组中以与`friendly_time`同样的形式注册函数，如：`PHP_FE(more_function)`。

### 具体实现
那么，`PHP_FUNCTION(friendly_time)`函数具体要怎么实现呢？先上代码：

```c
PHP_FUNCTION(friendly_time)
{
    char* strg;
    int   argc;
    int   len;
    int   dif_time;
    long  arg_time;
    long  now_time;
    struct tm tm_time;
    zval **arg;

    argc = ZEND_NUM_ARGS();
    if (argc == 0) {
        RETURN_NULL();
    }

    if (zend_parse_parameters(argc TSRMLS_CC, "Z", &arg) == FAILURE) {
        RETURN_NULL();
    }

    if (Z_TYPE_PP(arg) == IS_STRING) {
        strptime(Z_STRVAL_PP(arg), "%Y-%m-%d %H:%M:%S", &tm_time);
        arg_time = mktime(&tm_time);
    } else if(Z_TYPE_PP(arg) == IS_LONG) {
        arg_time = Z_LVAL_PP(arg);
    } else {
        RETURN_NULL();
    }

    time ( &now_time );
    dif_time = now_time - arg_time;

    if(dif_time<60) {
        len = spprintf(&strg, 0, "just now");
    } else if(dif_time<120) {
        len = spprintf(&strg, 0, "one minute ago");
    } else if(dif_time<3600) {
        len = spprintf(&strg, 0, "%d minutes ago", dif_time/60);
    } else if(dif_time<7200) {
        len = spprintf(&strg, 0, "one hour ago");
    } else if(dif_time<86400) {
        len = spprintf(&strg, 0, "%d hours ago", dif_time/3600);
    } else if(dif_time<172800) {
        tm_time = *localtime( &arg_time );
        len = spprintf(&strg, 0, "one day ago %d:%d", tm_time.tm_hour, tm_time.tm_min);
    } else if(dif_time<1209600) {
        tm_time = *localtime( &arg_time );
        len = spprintf(&strg, 0, "%d days ago %d:%d", dif_time/86400, tm_time.tm_hour, tm_time.tm_min);
    } else {
        tm_time = *localtime( &arg_time );
        len = spprintf(&strg, 0, "%d-%d-%d %d:%d", tm_time.tm_year+1900, tm_time.tm_mon+1, tm_time.tm_mday, tm_time.tm_hour, tm_time.tm_min);
    }

    RETURN_STRINGL(strg, len, 0);
}
```

这里涉及了几个知识点，记好了，考试会考到（开玩笑的）：

 - [变量类型](http://www.walu.cc/phpbook/2.md)
 - [函数参数](http://www.walu.cc/phpbook/6.md)
 - [函数返回](http://www.walu.cc/phpbook/7.md)

#### 接收参数
1. `ZEND_NUM_ARGS`这个函数（看他名字全大写，就知道也是个宏函数）可以获取到扩展函数在PHP运行环境中传入的参数个数；
2. `zend_parse_parameters`函数是用来解析传入参数的，像上面的代码中：
    
    ```cpp
    if (zend_parse_parameters(argc TSRMLS_CC, "Z", &arg) == FAILURE) {
        RETURN_NULL();
    }
    ```

    `Z`表示传入的参数类型是`zval**`（即在PHP中调用该函数时可以传入任意类型函数），这里是把一个`zval**`类型参数赋值给变量`arg`，详细内容参见《PHP扩展开发与内核应用》的 [第七章](http://www.walu.cc/phpbook/7.md) ；

3. 大家有没有发现`zend_parse_parameters`的第一个参数和第二参数是用空格分隔的，实际上他的宏定义是这样的：
    
    ```cpp
    #define TSRMLS_CC ,tsrm_ls
    ```

    所以，`,`是有的。那么他的作用是什么呢？具体参见《[揭秘TSRM（Introspecting TSRM）](http://www.laruence.com/2008/08/03/201.html)》。
    


#### 类型判断
`Z_TYPE_PP(arg) == IS_STRING`要表达的意思很直观，就是判断一下变量`arg`的实际值是不是字符串类型，`Z_TYPE_PP`是用来获取`zval**`类型变量的实际值类型，类似的还有`Z_TYPE`（对应`zval`）和`Z_TYPE_P`（对应`zval*`）。PHP内核中的变量类型详细内容参见《PHP扩展开发与内核应用》的 [第二章](http://www.walu.cc/phpbook/2.md)。

#### 返回值
比如在上面代码中，传入参数合法的花，返回的是`RETURN_STRINGL(strg, len, 0)`。与之类似的还有`RETURN_LONG`、`RETURN_DOUBLE`、`RETURN_BOOL`和`RETURN_NULL`等待，具体参见《PHP扩展开发与内核应用》的 [第六章](http://www.walu.cc/phpbook/6.md)。

----------

## 编译运行
代码编写完成后，在扩展项目目录下执行：

```cmd
$ phpize
$ ./configure
$ make
$ make install
```

如果编译成功的话，该扩展模块已经安装到本地的PHP中了，然后在`php.ini`（一般在`/etc/php/`下）中开启该扩展模块，即在`php.ini`里加上：

```cmd
extension="my_ext_name.so"
```

用一下命令可以查看PHP开启的扩展模块：

```cmd
$ php -m
```
测试一下`friendly_time`函数能不能正常运行：

```cmd
$ php -r 'echo friendly_time(time()-1000), "\n";'
```

## 最后
通过测试比较，扩展形式实现的`friendly_time`函数，和纯PHP语言实现的`php_friendly_time`函数的执行效率差距不大，所以说，并不是所有的函数都应该用扩展形式实现。除非是非常复杂、耗时的操作需要以扩展形式实现，否则，我们应该尽量地使用PHP提供的原生函数来实现想要的功能。

### 源码
- https://coding.net/u/breeze2/p/php-ext-dev-trip/git

### 参考
- [风雪之隅](http://www.laruence.com/)
- [PHP扩展开发与内核应用](http://www.walu.cc/phpbook/preface.md)

<br>

