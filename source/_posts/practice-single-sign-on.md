title: 单点登录的实现
date: 2017-10-03 10:03:13
tags: 
    - practice
    - php
    - js
categories:
    - 实践
---

[单点登录（Single Sign-On）](https://baike.baidu.com/item/SSO/3451380)，是指一次登录，多站复用登录态。这里主要讲解一下基于共享session实现的单点登录。

对于大多数B/S结构（浏览器/服务器结构）的Web网站，用户登录态都是存储在session中的，而sessionID（session的唯一识别号）记录在前端的cookie中，只要同步各个网站的sessionID，各个网站便能共享同一个session，进而共用用户的登录态。需要共享session的情况主要有三种：
1. 同域名下不同路径；
2. 同父域名下不同子域名；
3. 不同域名；

下面会根据这三种情况，借用Laravel和JQuery代码，逐一实现单点登录（共享session）。

<!--more-->

## 同域名下不同路径

假设一个网站下有两条路径：
* `https://www.a.site/path1`
* `https://www.a.site/path2`

若想这两条路径同步sessionID，十分简单，比如用户在`https://www.a.site/path1`路径下登录后，将记录sessionID的cookie设置成根路径'/'有效，这样网站`https://www.a.site`下的所有路径都能使用这个cookie。
代码实现：
```php
<?php
    // after login

    // raw php
    setcookie(session_name(), session_id(), time()+3600, '/');
    // use laravel
    \Cookie::queue(config('session.cookie'), session()->getId(), 60, '/');

```

## 同父域名下不同子域名
假设有两个网站：
* `https://www.a.site`
* `https://mob.a.site`

若想这两个网站同步sessionID，其实也不难。首先，这两个网站可能不在同一个主机上，所以session不能直接保存到本地文件中，而是要保存到同一数据库中，且两个网站都能访问这个数据库。然后，用户在`https://www.a.site`网站下登录后，将记录sessionID的cookie设置成父域名`.a.site`有效，这样`a.site`域名下的所有网站都能使用这个cookie。
代码实现：
```php
<?php
    // after login

    // raw php
    setcookie(session_name(), session_id(), time()+3600, '/', '.a.site');
    // use laravel
    \Cookie::queue(config('session.cookie'), session()->getId(), 60, '/', '.a.site');

```


## 不同域名

### JSONP+链接query
假设有两个网站：
* `https://www.a.site`
* `https://www.b.site`

若想这两个网站同步sessionID，就不是那么容易了。这里涉及了前端跨域通信问题。本来想着用**JSONP+链接query**来实现的——比如，用户在`https://www.a.site`网站下登录后，便在网页HTML中插入一个script标签，其src属性设为另一个网站的动态链接，sessionID放在链接query中，形如
`<script type="text/javascript" src="https://www.b.site/set-cookie?session_id=$SESSION_ID"></script>`
而`https://www.b.site/set-cookie`的处理脚本大概是
```php
<?php
    // set cookie
    
    // raw php
    setcookie(session_name(), $_GET['session_id'], time()+3600);
    // use laravel
    \Cookie::queue(config('session.cookie'), request()->input('session_id'), 60);
```
但是，这样一来，在网络传输过程中，sessionID就会暴露了，整个网站就不安全了。为了保护用户的sessionID，两个网站之间前端跨域通信采用**iframe+链接hash**来实现。

### iframe+链接hash
（其实这里才是整篇文章的重点）
首先，cookie取消HttpOnly限制。Laravel框架里，记录sessionID的cookie默认HttpOnly，因为后面需要在前端修改cookie，所以先取消HttpOnly：
```php
<?php
// config/session.php
    return [
        ...
        'http_only' => false;
    ];

```
当然，即使cookie是HttpOnly，也能被修改，不过需要在后端操作（步骤相对会繁琐），这里为了方便采用前端操作。
然后，登录操作使用异步请求。异步请求登录成功后，可以在回调函数里同步其他网站的sessionID。
```javascript
// login.js
    var sites = ['https://www.a.site', 'https://www.b.site'];
    var session_name = 'laravel_session';
    var sync_num = 0;

    function afterLogin() {
        window.location.href = '/home';
    }

    function afterSync() {
        if(sync_num === sites.length) setTimeout(afterLogin, 800);
    }

    function makeFrame(url) {
        var $iframe = $('<iframe style="display:none" class="sync-session-id"></iframe>').appendTo('body');
        $iframe.on('load', function () {
            sync_num++;
            afterSync();
        });
        $iframe.prop('src', url);
    }

    function getCookie(name) {
        var cookie_str = window.document.cookie;
        if (cookie_str.length>0) {
            var start = cookie_str.indexOf(name + '=');
            if (start!=-1) { 
                start = start + name.length + 1;
                var end=cookie_str.indexOf(';', start)
                if (end==-1) end=cookie_str.length;
                return unescape(cookie_str.substring(start, end))
            }
        }
        return '';
    }

    ...
    $.post('/paht/to/login', {username: username, password:password}).done(function (rst) {
        if(rst.code==0) { // login successful
            var session_id = getCookie(session_name); // 获取登录成功后的sessionID
            $.each(sites, function(i, e) {
                if(window.location.href.indexOf(e)==0) {
                    sync_num++;
                    return true;
                }
                makeFrame(e+'/set-cookie.html'+'#'+'name='+session_name+'&value='+session_id); // 访问网站e的set-cookie.html来同步sessionID
            });
        }
    });

```

每个网站都放置一个可访问的set-cookie.html(名字可以自取)，set-cookie.html里面的脚本会将链接的hash信息记录到自身网站的cookie里。
```html
<!DOCTYPE html>
<html>
<head>
    <title>set cookie</title>
</head>
<body>
    <script type="text/javascript">
        function setCookie(name, value, expires) {
            var date = new Date();
            date.setTime(date.getTime()+expires*1000);
            var cookie = name + '='
                + escape(value)
                + ((expires==null)? '' : ';expires='+date.toGMTString())
                + ';path=/';
            window.document.cookie = cookie;
        }
        var hash = window.location.hash;
        if(hash) {
            var data_str = hash.substring(1);
            var data_arr = data_str.split('&');
            var data = {};
            for(var i=0; i<data_arr.length; i++) {
                var str = data_arr[i];
                var arr = str.split('=');
                if(arr[0]) data[arr[0]] = arr[1] ? arr[1]: '';
            }
            if(data.name && data.value) {
                setCookie(data.name, data.value, 3600);
            }
        }
    </script>
</body>
</html>

```

这样用户在一个网站登录后，到另一个网站也能共享登录态。

### 继续优化

#### 更多的网站
当有更多的网站时，如：
* `https://www.a.site`
* `https://www.b.site`
* `https://www.c.site`
* `https://www.d.site`
* `https://www.e.site`
* ...

安装上面的代码，只需要在登录页面上把所有的网站地址添加到`sites`数组里即可：
```javascript
// login.js
    var sites = ['https://www.a.site', 'https://www.b.site',
        'https://www.c.site', 'https://www.d.site', 'https://www.e.site'];
    var session_name = 'laravel_session';
    var sync_num = 0;
    ...

```

或者，取一个网站作为其他网站的登录代理。假设是代理网站是`https://www.x.site`，而其他网站都有一个唯一识别码site_id。比如`https://www.a.site`的唯一识别码site_id是1，则用户访问`https://www.a.site`的登录页面时，网站带着site_id跳转到`https://www.x.site/login?site_id=1`，而`https://www.x.site/login`的后端处理方法大概是：
```php
<?php
    // pseudo code
    // use laravel

    public function getLogin(Request $request) { // get request
        $site_id = $request->input('site-id');
        $site_url = get_site_url_by_id($site_id);
        if($isLogined) { // has logined in www.x.site
            return redirect($site_url)->withCookie(\Cookie::make(config('session.cookie'), session()->getId(), 60));
        } else {
            return view('x.login');
        }
    }

    public function postLogin(Request $request) { // post request & ajax only
        ...
    }
```
若是用户未曾登录过，则在`https://www.x.site/login?site_id=$SITE_ID`页面上用异步请求登录，登录成功后前端脚本刷新当前页面，就会带着cookie跳回到site_id对应的网站；用户在`https://www.x.site`登录过，在登录其他网站的时候，直接跳到`https://www.x.site/login?site_id=$SITE_ID`，继而带着cookie跳回到site_id对应的网站。

#### 后端设置cookie
之前说过的后端跨域设置cookie，方法就是如上面代码那样带cookie跳转链接：
```php
<?php
    
    // raw php
    setcookie(session_name(), session_id(), time()+3600, '/');
    header('Location: '.$site_url);
    // use laravel
    redirect($site_url)->withCookie(\Cookie::make(config('session.cookie'), session()->getId(), 60));
```

#### 共享token
如果网站使用token来维持用户登录态，而不是session，那么怎样实现单点登录呢？其实思路是一样的，利用**iframe+链接hash**，将token放到链接hash里面，传递给其他网站，其他网站接收后把token保存到LocalStorage就行了。

## 最后
前后结合，可以实现很多hack操作。最后说说，QQ怎样将客户端里的用户登录态共享给浏览器：QQ客户端登录后会监听一些本地端口，各个QQ系网站通过JSONP方式访问这些端口，进而获取用户登录信息。

