title: 在JPEG图片中嵌入HTML
date: 2018-06-12 15:27:23
tags: 
    - html
    - frontend
    - jpeg
    - php 
    - practice
categories:
    - 实践
---

> 最近看到一个有趣的网页：[http://lcamtuf.coredump.cx/squirrel/](http://lcamtuf.coredump.cx/squirrel/)，或者说一张有趣的图片——因为用网络浏览器打开它看到的是一个网页，用图片浏览器打开它看到的又是一张图片。

![松鼠迷](/assets/images/practice-embed-html-in-jpeg1.jpg)

<!--more-->

在服务端要对同一个请求地址实现不同响应是十分简单的，比如通过请求头`Accept`来判断：

* `Accept=text/html`则返回超文本；
* `Accept=image/*`则返回图片；
* ……

可是[http://lcamtuf.coredump.cx/squirrel/](http://lcamtuf.coredump.cx/squirrel/)这个网页（或者说图片）在脱离服务端的情况下，依然能够呈现出网页和图片两种文件内容，这是怎样实现的呢？

在前端没有秘密，打开网络浏览器的开发者工具查看一下网址的响应内容：

![响应内容](/assets/images/practice-embed-html-in-jpeg2.jpg)

响应内容中有熟悉HTML标签，也有一大堆乱码。这堆乱码应该是图片文件的字符读码，但是为什么在网页上看不到呢？
原来HTML中使用了`body { visibility: hidden; }`样式和`<!--`注释标签（浏览器自动补全），乱码部分就这样被隐藏了。

HTML内容：

```html
<html><body><style>body { visibility: hidden; } .n { visibility: visible; position: absolute; padding: 0 1ex 0 1ex; margin: 0; top: 0; left: 0; } h1 { margin-top: 0.4ex; margin-bottom: 0.8ex; }</style><div class=n><h1><i>Hello, squirrel fans!</i></h1>This is an embedded landing page for an image. You can link to this URL and get the HTML document you are viewing right now (soon to include essential squirrel facts); or embed the exact same URL as an image on your own squirrel-themed page:<p><xmp><a href="http://lcamtuf.coredump.cx/squirrel/">Click here!</a></xmp><xmp><img src="http://lcamtuf.coredump.cx/squirrel/"></xmp><p>No server-side hacks involved - the magic happens in your browser. Let's try embedding the current page as an image right now (INCEPTION!):<p><img src="#" style="border: 1px solid crimson"><p>Pretty radical, eh? Send money to: lcamtuf@coredump.cx<!--
```

然而图片展示中，也没看到HTML的内容，又是为什么呢？

## JPEG相关
首先可以确定这张图片是JPEG格式，因为内容开头有明显的`JFIF`标记（[JFIF](https://en.wikipedia.org/wiki/JPEG_File_Interchange_Format)，是JPEG最常见的文件存储格式，也是标准的JPEG文件转换格式）。
JPEG格式定义了一系列标记码，都是以0xFF开头，常见的有：

* `0xFFD8`：SOI（Start Of Image），图片开始标记；
* `0xFFD9`：EOI（End Of Image），图片结束标记；
* `0xFFDA`：SOS（Start Of Scan），扫描开始标记；
* `0xFFDB`：DQT（Define Quantization Table），定义量化表；
* `0xFFC4`：DHT（Define Huffman Table），定义哈夫曼表；
* `0xFFEn`：APPn（Application Specific），应用程序信息；
* `0xFFFE`：COM（Comment），注释；
* ……

图片的注释内容不会展示，很显然我们可以把HTML隐藏在图片注释中。
用十六进制读取这张图片，可以看到：

![十六进制读取](/assets/images/practice-embed-html-in-jpeg3.jpg)

这张图片中使用了注释标记`0xFFFE`，标记后面跟着`0x0372`。`0x0372`转成十进制是`882`，而HTML内容的长度刚好是880个字节（`0x0372`本身占2个字节），所以可以知道，HTML内容就是写在这张图片注释中。

## 代码实现
下面用PHP代码简单实现一个将HTML内容嵌入JPEG中的函数：

```php
<?php
function embedHtmlInJpeg($jpeg_file, $html_str, $html_file)
{
    $length = strlen($html_str) + 2;
    if ($length > 256 * 256 - 1) {
        return false;
    }

    $content = '';
    $reader  = fopen($jpeg_file, 'rb');
    $writer  = fopen($html_file, 'wb');
    $content = fread($reader, 2); // read 0xFFD8
    fwrite($writer, $content); // write 0xFFD8

    $header  = 'FFFE' . sprintf('%04X', $length);
    $header  = pack('H*', $header);
    $content = $header . $html_str;
    fwrite($writer, $content); // write 0xFFFE

    while (!feof($reader)) {
        $content = fread($reader, 8192);
        fwrite($writer, $content); // write else
    }
    fclose($reader);
    fclose($writer);
    return true;
}

// call it
embedHtmlInJpeg('lena.jpg',
    '<html><body><style>body { visibility: hidden; } .n { visibility: visible; position: absolute; padding: 0 1ex 0 1ex; margin: 0; top: 0; left: 0; } h1 { margin-top: 0.4ex; margin-bottom: 0.8ex; }</style><div class=n><h1><i>This image is a page.</i></h1>Just open it in new tab.<p><img src="#" style="border: 1px solid crimson"><!--',
    'lena.html');

```

## 这张图片是一个网页，不信你就在新标签页中打开它

<a href="/assets/images/practice-embed-html-in-jpeg4.html" target="_blank">![Lena](/assets/images/practice-embed-html-in-jpeg4.html)</a>

## 实际应用
将一段HTML文本嵌入到一张图片中，实际上，还没什么应用，哈哈哈😂。
如果能将`JS`或`Shell`脚本藏在图片中，并能后期执行，那就有意思了；而且本身是一个图片文件，可以避过一些安全软件的检查。
