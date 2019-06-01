title: 自动化工具Nightmare的一次实践
date: 2017-09-16 12:07:13
tags: 
    - nightmare
    - js
    - practice
categories:
    - 实践
---

事情是这样的，一个线上的参赛活动网站要收集参赛选手提交的微云分享链接，但是普通微云账号的分享链接只有7天期效。那该怎么办呢？只好把所有选手的微云分享链接转存到一个高级微云账号上，再由高级微云账号分享出来，以此增长分享时间。如果靠人工操作，那工作量太大，也难免会出错，所以写了一个自动化脚本来执行这个工作。（后来微云取消了分享期效限制，这个脚本没有正式用上😊）

<!--more-->

## Nightmare

[Nightmare](https://github.com/segmentio/nightmare)是一个高级的浏览器自动化工具库，出自[Segment](https://open.segment.com/)。它提供了一些模拟用户操作（比如click、input和goto等等）的API，主要用于UI测试和网页爬虫。
Nightmare内置了[Electron](http://electron.atom.io/)，一个类似[PhantomJS](http://phantomjs.org/)的浏览器（或者说webview吧），但是比PhantomJS更快，更现代化。

引入Nightmare十分简单，初始化一个NodeJS项目后，执行`yarn add nightmare`或者`npm install nightmare`即可。

下面介绍收录微云分享链接以及再分享的具体实现。

## 具体实现

整个流程大概是在自己的云盘上新建一个文件夹，然后将别人的分享链接存到这个文件夹里，多个链接依此循环；再把这个文件夹里面的内容分享出去，记录自己的分享链接，多个文件夹依此循环。为了避免文件夹重名，每条分享链接配备一个唯一id。

最终效果：

<div style="width: 100%; text-align: center;">
<video width="640" height="360" controls="controls" autoplay="autoplay">
<source src="http://blog-1251124389.cosgz.myqcloud.com/nightmare-video-1.mp4">
</video>

<video width="640" height="360" controls="controls" autoplay="autoplay">
<source src="http://blog-1251124389.cosgz.myqcloud.com/nightmare-video-2.mp4">
</video>    
</div>


实现起来并不难，只要研究页面的DOM结构，然后安排好各个操作流程，再用Nightmare提供的接口实现这些操作就行了。

## 主要问题
实现过程中主要碰到两个问题，一个是跨域iframe通讯问题，另一个是获取Electron内部js变量问题。

### 跨域iframe

由于Electron浏览器的同源安全策略，主页面不能操作跨域iframe里的元素，但是云盘登录的时候需要操作qq.com域iframe里面的登录按钮。
解决办法是，引用[nightmare-iframe-manager](https://github.com/rosshinkley/nightmare-iframe-manager)扩展，禁用Electron安全模式：

```js
// yarn add nightmare
// yarn add nightmare-iframe-manager

var Nightmare = require('nightmare')
require('nightmare-iframe-manager')(Nightmare)

var nightmare = Nightmare({ show: true, dock: true,
  webPreferences: {
    webSecurity:false
  }
})

```

### 获取Electron内部js变量

在Electron内部运行js代码时，有时需要取出一些变量值做记录，比如，取出自己的云盘分享链接。天真的我以为设一个全局变量，既能在自己的脚本里使用，也能在Electron内部使用，结果是根本就不起效。
最后解决办法是引入[vo](https://github.com/matthewmueller/vo)，再结合ES6的新特性`yield`，获取Electron内部变量值：

```js
// yarn add nightmare
// yarn add vo

var vo = require('vo')
var Nightmare = require('nightmare')
var run = function* () {
  var nightmare = Nightmare({ show: true, dock: true, 
    webPreferences: {
      webSecurity:false
    }
  })

  nightmare.goto('https://xxxxx')
  var link = yield nightmare.evaluate(function () {
    var el = window.document.querySelector('#_disk_share_pop_container')
    return el.querySelector('span a.link').href
  })

}

vo(run)(function(err, result) {
  console.log(err, result)
  // console.log(JSON.stringify(result))
})
```

vo是一个能完全控制js执行流的工具库，而yield能“暂停”执行流，取出想要的某个变量值，再继续往下执行。JS的东西日新月异，没看一段时间就完全跟不上了😢。

## 最后
其实，研究网盘页面的DOM结构，再设计有效的执行流程，花费时间不少；不过能替代大量重复性的人工操作，也是值得的。


