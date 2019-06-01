title: Windows上编译PHP
date: 2016-12-12 13:12:18
tags:
    - php
    - practice
categories:
    - 躬行
---


> 平时在Windows系统上做PHP开发，都是直接使用[WAMP](http://www.wampserver.com/)的继承环境。最近需要用到一个PHP扩展，也没找到现成的DLL文件，于是想着自己编译一个。首先，需要安装一个[Visual Studio](https://www.visualstudio.com/)

## Visual Studio

上大学的时候，老师教我们用 *VC6.0* ，我就在自己电脑上装了个 *Visual Studio 2010*，功能全选，占了好十几GB空间，然后就没怎么用了，看来只是想体验一下进度条的优雅。后来用上了 *Sublime* ，以为此生足矣，没想到今天还是要求 *Visual Studio 2015*， 不愧为天下第一IDE。

安装过程略去：
![初始化Visual Studio](/assets/images/php-compile-on-windows1.png)

<!--more-->

熟悉的界面：
![熟悉的界面](/assets/images/php-compile-on-windows2.png)

其实这里与Visual Studio IDE没有多大关系，主要是要使用它的控制台：
![VC2015 x64 CMD](/assets/images/php-compile-on-windows3.png)


## PHP SDK

[PHP For Windows](http://windows.php.net)，是一个PHP官方提供Windows系统支持的网站。这里可以找到，已经编译好的、可以在Windows系统上直接运行的[PHP](http://windows.php.net/download/)，PECL上各种PHP扩展的[DLL文件](http://windows.php.net/downloads/pecl/releases/)，还有在Windows系统上编译PHP时需要用到的一些[工具](http://windows.php.net/downloads/php-sdk/)。

在Windows系统上编译PHP，需要准备以下材料：
- PHP源码，比如[php-7.0.14.tar.gz](http://cn2.php.net/get/php-7.0.14.tar.gz/from/this/mirror)
- PHP SDK，比如[php-sdk-binary-tools-20110915.zip](http://windows.php.net/downloads/php-sdk/php-sdk-binary-tools-20110915.zip)
- 编译时依赖，比如[deps-7.0-vc14-x64.7z](http://windows.php.net/downloads/php-sdk/deps-7.0-vc14-x64.7z)

## 编译步骤
### 工作空间

首先，把`php-sdk-binary-tools-20110915.zip`里面的内容解压到一个文件夹中，例如：`D:\php-sdk-binary-tools`。然后打开`VC2015 x64 CMD`，执行以下命令：

```powershell
D:
cd D:\php-sdk-binary-tools\
bin\phpsdk_setvars.bat
bin\phpsdk_buildtree.bat phpdev
xcopy /E phpdev\vc9\* phpdev\vc14\
```

再把`deps-7.0-vc14-x64.7z`里面的内容解压到`D:\php-sdk-binary-tools\phpdev\vc14\x64`，`php-7.0.14.tar.gz`里面的内容解压到`D:\php-sdk-binary-tools\phpdev\vc14\x64\php-7.0`，这时候，目录结构大概如下：

```
D:\php-sdk-binary-tools\
 |--bin\
 |--phpdev\
 |   |--vc6\
 |   |--vc8\
 |   |--vc9\
 |   |--vc14\
 |       |--x64\ 
 |           |--deps\
 |               |--bin\
 |               |--include\
 |               |--lib\
 |           |--php-7.0\
 |       |--x86\
 |--script\
 |--share\
```

### 开始编译

打开`VC2015 x64 CMD`，执行以下命令：

```powershell
D:
cd D:\php-sdk-binary-tools\phpdev\vc14\x64\php-7.0
buildconf

configure --help
configure 

nmake
nmake snap

```

`configure --help`会输出可用的选项，按需选择。

### 编译扩展

如果需要编译某个PHP扩展，比如名字叫`someone_ext`， 可以将其源码放置到`D:\php-sdk-binary-tools\phpdev\vc14\x64\php-7.0\ext\someone_ext`中，然后执行以下命令：

```powershell
D:
cd D:\php-sdk-binary-tools\phpdev\vc14\x64\php-7.0
nmake clean
buildconf --force
configure --help

```

正常情况下，`configure --help`的输出中`someone_ext`这个PHP扩展的相关选项，执行`configure`带上相关选项即可。只编译`someone_ext`这个PHP扩展，不重新编译PHP，大概的命令如下：

```powershell
configure --disable-all --enable-someone_ext=shared
nmake

```

## 后记

一开始提到初衷是想编译一个PHP扩展的，但是上文只字未提，主要是因为，在编译完PHP后，下载扩展的源码时才发现，它不支持Windows系统（只有`config.m4`，没有`config.w32`），欲哭无泪。这个扩展名字叫[pecl-gearman](https://github.com/wcgallego/pecl-gearman)，万事求不得人，有空自己写一个吧（估计写一个`config.w32`就行了吧），挽勉自尊。


## 参考

- [PHP For Windows](http://windows.php.net)
- [Build your own PHP on Windows](https://wiki.php.net/internals/windows/stepbystepbuild)
- [在Windows下编译PHP和PHP扩展](https://blog.ianli.xyz/2013/09/build-php-and-extension-for-windows/)