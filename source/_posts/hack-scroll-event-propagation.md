title: Web页面中滚动穿透的解决方法
date: 2018-01-08 18:48:30
tags: 
    - js
    - html5
    - css
    - scheme
categories:
    - 方案
---

最近做一个移动端的Web应用，因为是单页模式，所以少不了各种各样的弹出框，侧边栏，结果发现这些元素在滚动到边界时，滚动事件会传递给父元素，导致父元素开始滚动（在PC端也是一样存在这样的问题，而在移动端更为突出，有时即使不在边界也会导致父元素滚动，自身却不动）。

在网上搜寻一番后，找到很多关于这个问题的讨论，解决方法大致可以分为两类：
1. （JS）监听元素`el`滚动事件`event`，当`el.scrollTop`或者`el.scrollLeft`到达边界值的时候，用`event.preventDefault()`方法取消浏览器默认操作和用`event.preventDefault()`方法停止事件往上传播；
2. （CSS）在表层元素（弹出框，侧边栏等等）显示时，修改底层元素（一般是网页主体`body`）的样式，如`position:fixed`、`overflow:hidden`等等，使其不可滚动。

具体可以参考[移动端滚动穿透问题](https://github.com/pod4g/tool/wiki/%E7%A7%BB%E5%8A%A8%E7%AB%AF%E6%BB%9A%E5%8A%A8%E7%A9%BF%E9%80%8F%E9%97%AE%E9%A2%98)，其中提到的“终极完美解决方案”便属上面的第二类。

<!--more-->

## 疑虑
虽说是“终极完美解决方案”，但其实我是不相信的，因为修改了元素定位方式，元素需要重绘，内容显示位置也会改变，即使记录内容位置后续恢复，画面也少不免抖动，所以怎么可能是“完美”呢？

## 实践
尽管有所疑虑，还是要实践一下。于是简单地写一个测试样例了：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test</title>
    <style type="text/css">
        html, body {width: 100%;}
        body {background-color: #9e9e9e; height: 9000px; margin: 0; padding: 0; z-index: 1;}
        .main {background-color: #00bcd4; height: 6000px; margin: 0;}
        .nav {background-color: #2196f3; width: 200px; position: fixed; top: 0; bottom: 0; right: 0; overflow: scroll; display: none; z-index: 9;}
    </style>
</head>
<body>
    <div class="main">
        <div>
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <!-- more and more     -->
        </div>
    </div>
    <div class="nav">
        <div style="height: 2000px">
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <li>testing testing</li>
            <!-- more and more     -->   
        </div>
    </div>
    <script type="text/javascript">
        var main = document.querySelector('.main');
        var nav  = document.querySelector('.nav');
        var flag = 0;
        main.addEventListener('click', function () {
            nav.style.display = 'block';
            flag = document.body.scrollTop;
            document.body.style.position = 'fixed';
            document.body.style.top = -flag + 'px';
        });
        nav.addEventListener('click', function () {
            document.body.style.position = 'static';
            document.body.style.top = 'auto';
            document.body.scrollTop = flag;
            nav.style.display = 'none';
        });
    </script>
</body>
</html>
```

样例页面中，点击`.main`时，将`.nav`设为可见，且将`body`的定位设为`fixed`，向上偏移到`body`本来的滚动高度，这样`.nav`可以滚动而`.main`不能滚动，就不会出现滚动穿透的问题；再点击`.nav`，将`.nav`设为不可见，且将`body`的定位设为`static`，取消`body`的向上偏移并滚回本来的滚动高度。

### 有个问题
样例页面在Safari浏览器中运行正常，在Chrome浏览器中却有问题，因为Safari浏览器认为`body`元素是滚动主体，而Chrome浏览器认为`html`元素才是滚动主体。`document`中获取`body`元素的键值是`body`，而获取`html`元素的键值是`documentElement`，所以要在Chrome浏览器中运行正常的脚步代码应该是：

```javascript
var main = document.querySelector('.main');
var nav  = document.querySelector('.nav');
var flag = 0;
main.addEventListener('click', function () {
    nav.style.display = 'block';
    flag = document.documentElement.scrollTop;
    document.documentElement.style.position = 'fixed';
    document.documentElement.style.top = -flag + 'px';
});
nav.addEventListener('click', function () {
    document.documentElement.style.position = 'static';
    document.documentElement.style.top = 'auto';
    document.documentElement.scrollTop = flag;
    nav.style.display = 'none';
});
```

### 运行效果
在`.nav`出现或消失时，认真看可以开到`.main`有些小抖动，但并不明显，在今天移动端硬件（CPU，内存）和软件（浏览器）优化足够的情况下，这个滚动穿透的解决方法确实是可以放心使用的。

## 最后
开卷有疑，实践证明。
