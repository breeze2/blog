title: 关于JS的MVVM实现
date: 2017-01-15 20:31:45
tags: 
    - think
categories:
    - 求索
---

> 过去Web应用的常用开发模式是`MVC`，前端后台并在一起开发。随着JavaScript语言的发展，前端能做的越来越多，Web应用开发也趋向了“前后端分离”。前后端分离并不是什么新概念，实质是Web应用从B/S（浏览器／服务器）结构向C/S（客户端／服务端）结构转变 。分离后，前端就自成一个系统，大家开始探讨前端的开发模式，而其中`MVVM`模式备受推崇。

## MVVM
MVVM，即Model-View-ViewModel，是从MVC衍生出来的开发模式。MVVM模式用ViewModel替代了Controller，ViewModel就像是Model和View之间的桥梁——数据模型通过ViewModel展示在视图中，当视图发生变化，可以通过ViewModel来触发数据更改；而数据的更改，也可以通过ViewModel来触发View变化，示意图：

![MVVM](/assets/images/think-about-js-mvvm1.png)

<!--more-->

数据模型和视图之间的作用是相互的，数据模型跟随了视图变化，视图也跟随数据模型更改，这是一种双向绑定。数据模型和视图的双向绑定是MVVM的基础架构。
如果数据模型和视图能够双向绑定，那么前端中用户的复杂交互操作，可以短短几行代码就能实现。但是，怎么才能让数据模型和视图双向绑定呢？

## 猜想

最初体验到数据模型和视图的双向绑定，是在Angular的教程文档里。从面子，看里子，数据模型和视图的双向绑定是怎么实现的呢？当时就想到了set/get的方法（set/get是指对属性进行操作封装，这里主要用到的是set），伪代码如下：

```javascript
function Model(data) {
    var _view;

    this.bind = function(view) {
        _view = view;
    };
    this.set = function(name, value) {
        this[name] = value;
        _view.update();
    };
    this.update = function() {
        // reset by _view;
    }
}

function View(dom) {
    var _model;

    this.bind = function(model) {
        _model = model;
    };
    dom.addEventListener('change', function() {
        _model.update();
    });
    this.update = function() {
        // render by _model
    };
}

var m = new Model(data);
var v = new View(dom);
m.bind(v);
v.bind(m);
m.set('a', 1);
...

```

Model实例通过`set`方法来修改属性时，`set`方法中已经带有更新对应View实例的代码；View实例在HTML中的标签触发`change`事件时，回调函数中也带有更新对应Model实例的代码，这样就简单的实现了一个Model和View的双向绑定。然而，能不能直接通过等于号`=`赋值时就触发了数据修改的事件呢？

## 主流实现

主流的前端MVVM框架，实现双向绑定的做法大概可归纳为以下三种：
1. 发布-订阅
2. 脏值检查
3. 数据劫持

### 发布-订阅

发布-订阅，是一种事件模型。Model和View都有各自的发布、订阅方法，同时Model在修改数据的时候会发布广播，而View订阅了这个广播，在收到广播的时候更新视图；View在视图更新的时候会发布广播，而Model也订阅了这个广播，在收到广播的时候更新数据，这样就形成了双向绑定，大致的实现代码与上面的“猜想”差不多。
Backbone.js就是使用了 *发布-订阅* 来实现双向绑定。

### 脏值检查

脏值检查，是指在特定情况下，检查数据是否有修改，是否需要更新视图；或者检查视图是否有更改，是否需要更新数据。例如，我们可以利用`setInterval`函数来设置定时检查，尽管非实时，但也可以实现了双向绑定。
Angular.js也是使用脏值检查来实现双向绑定。不过Angular.js不是定时周期地去检查，而是在一些特殊事件发生时，才执行脏值检查，比如：

- DOM的`change`、`check`、`click`等事件；
- XHR响应事件；
- $digest()、$apply()、$timeout()和$interval()等函数的调用；
- ...


### 数据劫持

ECMAScript262v5中Object有了一个新的方法属性：[defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。Object.defineProperty() 方法会直接在一个对象上定义一个新属性，或者修改一个已经存在的属性，并返回这个对象；同时，能够设置该属性在被get或set的回调函数，该属性被赋值，或者被取值的时候都会执行回调函数，这就是数据劫持。
利用这个新特性，再结合 *发布-订阅*，我们就能实现直接通过等于号`=`赋值时就触发了数据修改的事件，进而更新视图。
Vue.js正是使用了这种方式实现了Model到View的绑定，当然View到Model的绑定还是靠监听HTML的DOM事件。

## 最后
最后说两句，如果前端使用了MVVM模式开发，那么一定要抛弃过去手动操作DOM来获取数据、更新视图等等思想（尤其是jQuery根深蒂固的影响），否则很难融入MVVM。
