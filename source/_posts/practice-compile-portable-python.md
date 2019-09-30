title: 源码编译Portable Pyton
date: 2019-09-28 10:15:06
tags: 
    - python
    - practice
categories:
    - 实践
---

> Portable的意思是可拔插的，便携式的。Portable 程序在一台计算机上编译好后，移植到其他计算机上就可以直接运行。一般程序运行时都会依赖一些第三方库，只要把对这些库的依赖方式由静态依赖改成动态依赖，再把这些库和主体程序放到一起打包，就成为一个Portable 程序。
> 当然Portable 也是有限度的，不能跨操作系统移植，毕竟程序对操作系统的依赖关联太多，除非把整个系统也移植了😂。

以[Python-3.6.5](https://www.python.org/downloads/release/python-365/) 为例，讲述一下源码编译一个Portable Pyton 的主要流程。

<!--more-->

## Windows 平台

[winpython](https://winpython.github.io/) 是一个专门编译Windows 平台下Portable Pyton 的项目，可以直接下载[WinPython32-3.6.5.0Zero.exe](https://github.com/winpython/winpython/releases/download/1.10.20180404/WinPython32-3.6.5.0Zero.exe)或者[WinPython64-3.6.5.0Zero.exe](https://github.com/winpython/winpython/releases/download/1.10.20180404/WinPython64-3.6.5.0Zero.exe)，解压出来就是对应的32位Portable Pyton 或64位Portable Pyton.

## macOS 平台

最好在`OS X10.9`系统下编译Portable Pyton，对其他更高级的macOS 兼容性会好一些。
python 的一些内置模块会依赖第三方库，主要是：

- [lzma](https://tukaani.org/xz/)
- [openssl](https://www.openssl.org/)
- [tcl/tk](https://www.tcl.tk/)

所以，我们要先安装这些库。

### 编译lzma

下载[xz-5.2.3.tar.gz](https://tukaani.org/xz/xz-5.2.3.tar.gz)，解压到`/tmp/xz-5.2.3`，然后：

```cmd
$ cd /tmp
$ mkdir local
$ cd xz-5.2.3
$ ./configure --prefix=/tmp/local
$ make && make install
```

### 编译openssl

下载[openssl-OpenSSL_1_0_2t.zip](https://github.com/openssl/openssl/archive/OpenSSL_1_0_2t.zip)，解压到`/tmp/openssl-OpenSSL_1_0_2t`，然后：

```cmd
$ cd /tmp/openssl-OpenSSL_1_0_2t
$ ./Configure darwin64-x86_64-cc --prefix=/tmp/local -shared
$ make && make install
```

### 编译tcl/tk
下载[tcl8519-src.zip](https://sourceforge.net/projects/tcl/files/Tcl/8.5.19/tcl8519-src.zip/download)，解压到`/tmp/tk8519-src`；下载[tk8519-src.zip](https://sourceforge.net/projects/tcl/files/Tcl/8.5.19/tk8519-src.zip/download)，解压到`/tmp/tk8519-src`，然后：

```cmd
$ cd /tmp/tcl8519-src
$ cd unix/
$ ./configure --prefix=/tmp/local
$ make && make install
$ cd /tmp/tk8519-src
$ cd unix/
$ ./configure --prefix=/tmp/local --with-tcl=/tmp/tcl8519-src/unix --enable-aqua
$ make && make install
```

### 编译python
下载[Python-3.6.5](https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz)，解压到`/tmp/Python-3.6.5`， 然后：

```cmd
$ cd /tmp
$ mkdir python
$ cd Python-3.6.5
$ CPPFLAGS="-I/tmp/local/include" \
LDFLAGS="-L/tmp/local/lib" \
./configure --prefix=/tmp/python 
$ make && make install
```

### 修改依赖路径

以上python 安装在`/tmp/python`，把python 依赖的第三方动态库，复制到`/tmp/python/lib`：

```cmd
$ cd /tmp/python/lib
$ cp /tmp/local/lib/liblzma.5.dylib ./
$ cp /tmp/local/lib/libcrypto.1.0.0.dylib ./
$ cp /tmp/local/lib/libssl.1.0.0.dylib ./
$ cp /tmp/local/lib/libtcl8.5.dylib ./
$ cp /tmp/local/lib/libtk8.5.dylib ./
$ cp -R /tmp/local/lib/tcl8 ./
$ cp -R /tmp/local/lib/tcl8.5 ./
$ cp -R /tmp/local/lib/tk8.5 ./
```

macOS下，`otool`工具可以查看动态库的依赖路径，`intall_name_tool`工具可以修改动态库的依赖路径。
我们要把所有的外部依赖（除了系统依赖）都修改为`/tmp/python/lib`内部依赖，绝对的依赖路径也改为相对的依赖路径：

#### 修改`libssl.1.0.0.dylib`：

```cmd
$ cd /tmp/python/lib
$ otool -L libssl.1.0.0.dylib
$ sudo install_name_tool -change /tmp/local/lib/libcrypto.1.0.0.dylib @executable_path/../lib/libcrypto.1.0.0.dylib libssl.1.0.0.dylib
```

#### 修改`_lzma.cpython-36m-darwin.so`：

```cmd
$ cd /tmp/python/lib/python3.6/lib-dynload
$ otool -L _lzma.cpython-36m-darwin.so
$ install_name_tool -change /tmp/local/lib/liblzma.5.dylib @executable_path/../lib/liblzma.5.dylib _lzma.cpython-36m-darwin.so
```

#### 修改`_ssl.cpython-36m-darwin.so`：

```cmd
$ cd /tmp/python/lib/python3.6/lib-dynload
$ otool -L libssl.1.0.0.dylib
$ install_name_tool -change /tmp/local/lib/libssl.1.0.0.dylib @executable_path/../lib/libssl.1.0.0.dylib -change /tmp/local/lib/libcrypto.1.0.0.dylib @executable_path/../lib/libcrypto.1.0.0.dylib _ssl.cpython-36m-darwin.so
```

#### 修改`_tkinter.cpython-36m-darwin.so`：

```cmd
$ cd /tmp/python/lib/python3.6/lib-dynload
$ otool -L libssl.1.0.0.dylib
$ install_name_tool -change /System/Library/Frameworks/Tcl.framework/Versions/8.5/Tcl @executable_path/../lib/libtcl8.5.dylib -change /System/Library/Frameworks/Tk.framework/Versions/8.5/Tk @executable_path/../lib/libtk8.5.dylib _tkinter.cpython-36m-darwin.so
```

#### 测试

测试修改依赖路径后的python运行情况：

```cmd
$ cd /tmp/python
$ ./bin/python -m pip lmza
$ ./bin/python -m pip ssl
$ ./bin/python -m pip tkinter
```

### 升级pip 

```cmd
$ cd /tmp/python
$ ./bin/python -m pip install --upgrade pip
```

### 最后

我们得到了一个Portable Pyton `/tmp/python`，可以内嵌到其他程序中一起打包，分发到各个计算机运行。

## 参考

- [How to Compile Tcl](https://www.tcl.tk/doc/howto/compile.html)
- [Getting Started — Python Developer's Guide ](https://devguide.python.org/setup/)
- [TkDocs - Tk Tutorial - Installing Tk](https://tkdocs.com/tutorial/install.html)
- [Portable OpenSSL dylib on Mac OS X](https://bitbucket.org/snippets/Zifix/88ny/)
