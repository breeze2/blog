title: jQuery Ajax的封装
date: 2016-08-20 14:08:20
tags: 
    - js
categories:
    - 躬行
---

> 程序开发时，经常引入第三方类库。如果类库中的一些函数调用得非常频繁，那么就应该考虑自己封装一下，这样对于后面的代码迭代，会带来很大便利。这里针对jQuery.ajax，讨论一下其如何封装，对于其他类库或许也适用。

## $.ajax

jQuery的ajax的确是个好东西，主要实现异步HTTP请求的功能，相信大家都有用过，并且觉得好用。jQuery从1.5.0版本开始，引入了[deferred](http://api.jquery.com/category/deferred-object/)对象，[$.ajax](http://api.jquery.com/category/ajax/)（都知道$是jQuery的别称）的调用就变得更加简单快捷。比如，发起一个POST请求：

```javascript
var url  = '/';
var data = {};
$.post(url, data).done(function() {}).fail(function() {});
```

`$.ajax`本身就是对 **XMLHttpRequest** 的封装，并且已经封装得非常好。借用`$.ajax`，我们只需要一两行代码就能完成一个异步HTTP请求；`$.ajax`在很多应用场景中也适用，那么，为什么还要自己再封装一层呢？

<!--more-->

毕竟，世间没有百分百完美的第三方类库，各个开发项目有各自不同的需求，`$.ajax`本身不可能完全去适配，自己改装一下很有必要；对于以后的代码变更，如果直接使用`$.ajax`，那么控制权就在jQuery手中，而自己封装一层，控制权就在自己手中，到时候改代码会简单很多。

还是先说说，怎么封装`$.ajax`吧。比如，封装`$.post`：

```javascript
// 假设前面已经引入了jQuery
function BL() {
    this._author  = 'BreezeLin';
    this._version = '0.1.0';
}
BL.prototype = {
    post: function(url, data) {
        return $.post(url, data);
    },
}

// 或者
BL.prototype = {
    post: function() {
        return $.post.apply(this, arguments);
    },
};

// 使用
var B = new BL();
B.post('/', {}).done(function() {}).fail(function() {});
// 后面凡是要发起异步POST请求，都是使用B.post，而不是$.post
 
```

哈，说好的封装，结果只是直接返回，这样的封装跟不封装有什么区别？是的，当前是没有区别，一切都是为以后的代码迭代做准备。假设有那么一天，jQuery开始收费或者停止更新，如果我们做了这一层的封装，那么我们只需要在`BL.prototype`中，实现或者加固异步HTTP功能——实际上我们的代码完全可以摆脱对jQuery依赖，控制权始终在我们手上（怎么有种“[控制反转](/2017/01/07/think-about-encapsulating-libs)”的感觉？）。

## 参数过滤

一般的HTTP传输，为了确保安全性，都会对请求参数进行加密。`$.ajax`是原样提交的，不能满足（所以肯定要封装啊）。如果一开始就有做自己的一层封装，那么后期从参数无加密，到参数加密，或者加密算法变更都是很方便的。像下面代码，只需要修改`my_encrypt`函数，就行了：

```javascript
function BL() {
    this._author  = 'BreezeLin';
    this._version = '0.1.0';
}
// 加密函数
function my_encrypt(data) {
    return data;
}
BL.prototype = {
    post: function(url, data) {
        data = my_encrypt(data);
        return $.post(url, data);
    },
};
```

第一个参数，请求路径，可能会需要补全、校验等操作，也可以在`BL.prototype`里面设置。如果有必要，还可以考虑开发第三参数，第四参数……

## 回调函数改装

> 别看“参数过滤”和“回调函数改装”名字上不对仗，实际上他们是异步HTTP请求的一前一后：一个是请求前处理，一个是返回后处理。

首先，我们假设开发项目中有这么一个场景：

1. 项目的HTTP服务接口采用数据无加密返回；
2. 前端JS的异步请求使用的是`$.ajax`，并且进行了如下封装：
    
    ```javascript
    function BL() {
        this._author  = 'BreezeLin';
        this._version = '0.1.0';
    };
    BL.prototype = {
        post: function(url, data) {
            return $.post(url, data);
        },
    };
    ```

3. 项目代码中，多处使用了如下代码：
    
    ```javascript
    var B = new BL();
    var url = '/';
    var data = {};
    B.post().done(function(result) {
        // 伪代码，对返回数据的一些操作
        if(result.status==1) {
            handle_with(result.data);
        } else {
            alert(result.info);
        }
    });
    ```

4. 问题来了，项目的HTTP服务接口开始采用数据加密返回。

如果当初自己没有封装`$.ajax`，那么项目中用到异步请求的地方只能一行一行代码地去改（所以知道自己封装的好处了吧）。现在是有自己封装且多处调用了，但是请求返回有变更，那该怎么办呢？

其实这是一道面试题，单凭这个问题就可以展开写成一篇文章了。这个问题，以前面试的时候人家问过我，后来面试的时候我也问过人家，虽然不是百分百的原题，但是核心思想是一样的。

先看问题：项目中，`done`函数已经被大量使用，一个个地去改，工程量太大；但是`done`的函数参数中的`result`已经被加密，不能直接使用，比如`alert(result.info);`并不能得到预期结果。

解决办法：还是得修改`done`函数，在哪里改呢？在`BL.prototype`里面改。这其实是对JavaScript中函数的`call`或者`apply`，`arguments`等知识点的考查：

```javascript
function BL() {
    this._author  = 'BreezeLin';
    this._version = '0.1.0';
};
// 解密函数
function my_decrypt(data) {
    return data;
}
BL.prototype = {
    post: function(url, data) {
        var $deferred = $.post(url, data);
        var $done = $deferred.done;
        $deferred.done = function() {
            var arg1 = arguments[0];
            if(typeof arg1 === 'function') {
                var func = function() {
                    var args = arguments;
                    var result = args[0];
                    args[0] = my_decrypt(result);
                    return arg1.apply(this, args);
                };
                arguments[0] = func;
            }
            return $done.apply(this, arguments);
        };
        return $deferred;
    },
};

```

哈哈，代码是不是有点绕，细心读一下，还是挺精彩的。这是一个二重改装，先改装了`done`函数，在改装了`done`函数参数（嗯，好像比原来的面试题复杂了点，开始还以为实现不了了）。就这样，本来调用了`done`函数的地方不用改动代码，程序也可以能够正常运行了。

## 最后
最后说两句：项目开发的过程中，经常会引入一些第三方类库，这样自己代码就会“依赖”这些类库；我们自己对其封装一层，把控制权掌握在自己手中，还是很有必要的。自己封装代码，对后面不可预期的变更风险，还是有相对高的可控性。jQuery.ajax只是一个例子，以其为镜，对频繁使用的类库加一层自己的封装，保证自己的控制权。