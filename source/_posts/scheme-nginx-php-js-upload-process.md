title: HTTP文件上传的一个后端完善方案（NginX）
date: 2017-08-12 14:08:30
tags: 
    - nginx
    - php
    - js
    - scheme
categories:
    - 方案
---

很多网站都会有上传文件的功能，比如上传用户头像，上传个人简历等等，除非是网盘类的网站，一般上传文件不会作为网站的主要功能；而且，如今大众的网速已经是足够的快，上传几百KB的文件，几乎可以秒内完成。所以，大多数网站，对于上传文件的处理，都是简单的前端POST上传，后端验证存放然后返回访问地址。毕竟，文件小，网速快，一瞬间的事情谁会多在意呢？

<!-- 但是，如果用户要上传的文件大小有几十MB，正常网速上传，过程也要几十秒的话，那么在这个几十秒内怎么告知用户上传的进度呢？在这个几十秒内服务器又能承载多少个用户的全速上传呢？怎样拒绝百MB或者更大的文件上传呢？等等这些问题都需要我们多在意。 -->

## 存在问题
假设我们有一个网站，基于NginX+PHP+JS构架，网站允许用户上传一些小视频、音乐或者PPT等文件在线上展示，单个文件大小限制不超过30MB，那么我们要怎样实现这个上传功能呢？

<!--more-->

### 限制上传文件的大小
首先，NginX要能接受最大32MB的请求（除了最大文件本身30MB，再预留一些给其他请求参数），我们会修改网站的虚拟主机配置：

```conf
# website.conf
server{
    client_max_body_size 32M; 
    ...
}

```

然后，PHP也要修改配置，接受最大30MB的文件上传和最大32MB的POST请求：

```ini
# php.ini
upload_max_filesize = 30M;
post_max_size = 32M;

```

其实，单凭`client_max_body_size`，NginX是不能真正限制上传文件大小的，因为NginX会先让客户端（一般是浏览器）开始上传请求，直到上传的内容大小超过了限制，NginX才会中止上传，报`413 Request Entity Too Large`错误，没超过限制则交给PHP处理。
于是，PHP的`upload_max_filesize`和`post_max_size`就更没用了，因为PHP获取到文件信息的时候，上传过程已经结束了（这时当然是上传成功，NginX中止请求的话PHP不会进场）。在NginX传递请求结果前，PHP什么（比如验证用户，验证权限等等）都做不了。

如果用户上传了一个大于32MB的时候，直到上传到32MB的时候才能告诉用户文件过大了，那么前面的时间用户不就白等了吗？而且服务器的带宽还是一样被消耗了。我们更希望在上传开始前就能告诉用户文件过大了。很多网站开发，都会把这一步交给JS处理，在新型浏览器(支持HTML5)里，JS的确可以获取input文件的大小；在旧的IE里，也可以通过ActiveX来实现。但是JS的限制处理很容易被绕过去，只要知道上传地址，一个`form`标签就能把文件传过去：

```
<form id="upload_form" action="/path/to/upload" enctype="multipart/form-data" method="post">
    <input type="file" name="upload_file" value="/path/of/big/big/file" />
    <input type="submit" value="Upload" />
</form>

```

正常的用户当然不会这样做，但是有意攻击网站的人会。

### 限制上传文件的速度
如果服务器的入口带宽是100mbps，用户的上行带宽是10mbps，用户上传一个30MB的文件至少需要30秒，那么在30秒内，服务器的带宽只能满足10个用户上传文件，带宽被占满后，服务器就很难再处理其他请求了。所以，限制用户上传文件的速度就很有必要。目前，JS做不到限制上传文件的速度，PHP也做不到。

### 上传文件的进度
用户上传一个30MB的文件至少需要30秒，那么30秒内应该告知用户上传的进度，不能让用户无感知的等待。HTML5改进了XMLHttpRequest对象，在支持HTML5的新型浏览器里，JS可以获取XMLHttpRequest上传文件的进度；在旧的浏览器的也可以通过Flash与JS结合（比如SWFUpload），从而获取上传文件的进度。但是新型浏览器里，Flash已经被摒弃了，因而要支持新旧浏览器，JS就要写成两套代码。在这里PHP也是帮不上忙，因为PHP拿到传文件信息的时候，上传已经结束了。

## 解决方案
网站是NginX+PHP+JS构架的，PHP和JS解决不了的问题，那应该在NginX上解决它。NginX虽然是一个现成的软件，但是它还是可以继续扩展和修改的。NginX本身没有提供上传文件的复杂处理功能，而在NginX官方认可的第三方扩展模块里，有两个模块可以帮助我们实现复杂的上传文件功能，分别是[nginx-upload-module](https://github.com/vkholodkov/nginx-upload-module)和[nginx-upload-progress-module](https://github.com/masterzen/nginx-upload-progress-module)。

要将nginx-upload-module和nginx-upload-progress-module编译进NginX，首先要下载NginX源码和nginx-upload-module、nginx-upload-progress-module这两个模块的源码，然后在NginX源码目录中，在`configure`参数中加入这两个这两个模块，最后`make install`，大概的执行命令：

```cmd
$ cd ~
$ mkdir tmp
$ cd tmp
$ wget http://nginx.org/download/nginx-1.11.3.tar.gz
$ tar -xvzf nginx-1.11.3.tar.gz
$ git clone https://github.com/vkholodkov/nginx-upload-module.git
$ git clont https://github.com/masterzen/nginx-upload-progress-module.git
$ cd nginx-1.11.3
$ ./configure --add-module=~/tmp/nginx-upload-module --add-module~/tmp/nginx-upload-progress-module ...
$ make
$ make install

```

如果系统上已经安装过NginX并且所安装NginX版本支持动态模块，那么可以考虑将nginx-upload-module和nginx-upload-progress-module编译成动态模块，这样就不需要重新安装NginX。 
[nginx-module-libs](https://github.com/breeze2/nginx-module-libs)上有Ubuntu系统上主线NginX版本的一些动态模块，可以上面下载适配你的nginx-upload-module和nginx-upload-progress-module。


> 下面主要介绍一下两个模块的用法：

### nginx-upload-module

当上传文件的体积小于`client_max_body_size`时， nginx-upload-module可以帮助我们限制上传速度，使用方法见下。
NginX的站点配置：
```nginx
# website.conf
server {
    ...
    client_max_body_size 32m;
    # 限制上传速度最大2Mbps
    upload_limit_rate 256k;

    location /upload {
        # 限制上传文件最大30MB
        upload_max_file_size 30m;

        # 后续交给 upload.php 处理
        upload_pass /upload.php;
        
        # 指定上传文件存放目录，1表示按1位散列，将上传文件随机存到指定目录下的0、1、2、...、8、9目录中（这些目录要手动建立）
        upload_store /tmp 1;
        
        # 上传文件的访问权限，user:r表示用户只读
        upload_store_access user:r;

        # 设置请求体的字段
        upload_set_form_field "${upload_field_name}_name" "$upload_file_name";
        upload_set_form_field "${upload_field_name}_content_type" "$upload_content_type";
        upload_set_form_field "${upload_field_name}_path" "$upload_tmp_path";

        # 指示后端关于上传文件的md5值和文件大小
        upload_aggregate_form_field "${upload_field_name}_md5" "$upload_file_md5";
        upload_aggregate_form_field "${upload_field_name}_size" "$upload_file_size";

        upload_pass_form_field "^submit$|^description$";

        # 若出现如下错误码则删除上传的文件
        upload_cleanup 400 404 499 500-505;
    }
}

```

上传文件的页面：
```html
<form id="upload" enctype="multipart/form-data" action="/upload" method="post" >
    <input name="upload_file" type="file" label="fileupload" />
    <input type="submit" value="Upload File" />
</form>

```

处理上传结果的脚本：
```php
<?php
// upload.php
print_r($_REQUEST);

```


如果对PHP解析使用了优雅链接，比如Laravel，那么应该这样使用：

NginX的站点配置：
```nginx
# website.conf
server {
    ...
    client_max_body_size 32m;
    # 限制上传速度最大2Mbps
    upload_limit_rate 256k;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_param HTTP_PROXY "";
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }

    location @upload_handle {
        rewrite ^ /index.php last;
    }

    location /upload {
        # 限制上传文件最大30MB
        upload_max_file_size 30m;

        # 后续交给 index.php 处理
        upload_pass @upload_handle;
        
        # 指定上传文件存放目录，1表示按1位散列，将上传文件随机存到指定目录下的0、1、2、...、8、9目录中（这些目录要手动建立）
        upload_store /tmp 1;
        
        # 上传文件的访问权限，user:r表示用户只读
        upload_store_access user:r;

        # 设置请求体的字段
        upload_set_form_field "${upload_field_name}_name" "$upload_file_name";
        upload_set_form_field "${upload_field_name}_content_type" "$upload_content_type";
        upload_set_form_field "${upload_field_name}_path" "$upload_tmp_path";

        # 指示后端关于上传文件的md5值和文件大小
        upload_aggregate_form_field "${upload_field_name}_md5" "$upload_file_md5";
        upload_aggregate_form_field "${upload_field_name}_size" "$upload_file_size";

        upload_pass_form_field "^submit$|^description$";

        # 若出现如下错误码则删除上传的文件
        upload_cleanup 400 404 499 500-505;
    }
}

```

上传文件的页面：
```html
<!-- laravel blade.php -->
<form id="upload?_token={{csrf_token()}}" enctype="multipart/form-data" action="/upload" method="post" >
    <input name="upload_file" type="file" label="fileupload" />
    <input type="submit" value="Upload File" />
</form>

```

Laravel路由配置：
```php
<?php
// routes/web.php
Route::post('/upload', 'Web\IndexController@upload')->name('upload');

```

Laravel控制器中处理上传的方法：
```php
<?php
// Web/IndexController.php
function upload() {
    dump(request());
}

```



### nginx-upload-progress-module
nginx-upload-progress-module可以帮助我们跟踪上传的进度，使用方法见下。

NginX的站点配置：
```nginx
# website.conf
server {
    ...
    client_max_body_size 32m;
    
    # 开辟一个空间proxied来存储跟踪上传的信息1MB
    upload_progress proxied 1m;
    location ^~ /progress {
        # 报告上传的信息
        report_uploads proxied;
    }

    location /upload {
        ...
        # 上传完成后，仍然保存上传信息5s
        track_uploads proxied 5s;
    }
}

```

上传文件的页面和每隔一秒查询一下上传进度的脚本：
```html
<form id="upload" enctype="multipart/form-data" action="/upload" method="post" onsubmit="openProgressBar(); return true;">
    <input name="userfile" type="file" label="fileupload" />
    <input type="submit" value="Upload File" />
</form>
<div>
    <div id="progress" style="width: 400px; border: 1px solid black">
        <div id="progressbar" style="width: 1px; background-color: black; border: 1px solid white">&nbsp;</div>
    </div>
   <div id="tp">(progress)</div>
</div>
<script type="text/javascript">
    var interval = null;
    var uuid = "";
    function openProgressBar() {
        for (var i = 0; i < 32; i++) {
            uuid += Math.floor(Math.random() * 16).toString(16);
        }
        document.getElementById("upload").action = "/upload?X-Progress-ID=" + uuid;
        /* 每隔一秒查询一下上传进度 */
        interval = window.setInterval(function () {
            fetch(uuid);
        }, 1000);
    }
    function fetch(uuid) {
        var req = new XMLHttpRequest();
        req.open("GET", "/progress", 1);
        req.setRequestHeader("X-Progress-ID", uuid);
        req.onreadystatechange = function () {
            if (req.readyState == 4) {
                if (req.status == 200) {
                    var upload = eval(req.responseText);
                    document.getElementById('tp').innerHTML = upload.state;
                    /* 更新进度条 */
                    if (upload.state == 'done' || upload.state == 'uploading') {
                        var bar = document.getElementById('progressbar');
                        var w = 400 * upload.received / upload.size;
                        bar.style.width = w + 'px';
                    }
                    /* 上传完成，不再查询进度 */
                    if (upload.state == 'done') {
                        window.clearTimeout(interval);
                    }
                    if (upload.state == 'error') {
                        window.clearTimeout(interval);
                        alert('something wrong');
                    }
                }
            }
        }
        req.send(null);
    }
</script>

```

当上传文件的体积大于`client_max_body_size`时， nginx-upload-module未能帮我们立刻中断上传，并且不能限制上传速度，但是nginx-upload-progress-module可以向前端报告文件过大的错误，前端可以这样子来中断上传：
```html
<form id="upload" enctype="multipart/form-data" action="/upload" method="post" onsubmit="openProgressBar(); return false;">
    <input name="userfile" type="file" label="fileupload" id="userfile" />
    <input type="submit" value="Upload File" />
</form>
<div>
    <div id="progress" style="width: 400px; border: 1px solid black">
        <div id="progressbar" style="width: 1px; background-color: black; border: 1px solid white">&nbsp;</div>
    </div>
   <div id="tp">(progress)</div>
</div>
<script type="text/javascript">
    var interval = null;
    var uuid = "";
    var uploadxhr = null; 
    function openProgressBar() {
        for (var i = 0; i < 32; i++) {
            uuid += Math.floor(Math.random() * 16).toString(16);
        }
        var action = "/upload?X-Progress-ID=" + uuid;
        var file = document.getElementById('userfile').files[0];
        uploadxhr = new XMLHttpRequest();
        // uploadxhr.file = file;
        uploadxhr.open('post', action, true);
        uploadxhr.setRequestHeader("Content-Type","multipart/form-data");
        uploadxhr.send(file);
        /* 每隔一秒查询一下上传进度 */
        interval = window.setInterval(function () {
            fetch(uuid);
        }, 1000);
    }
    function fetch(uuid) {
        var req = new XMLHttpRequest();
        req.open("GET", "/progress", 1);
        req.setRequestHeader("X-Progress-ID", uuid);
        req.onreadystatechange = function () {
            if (req.readyState == 4) {
                if (req.status == 200) {
                    var upload = eval(req.responseText);
                    document.getElementById('tp').innerHTML = upload.state;
                    /* 更新进度条 */
                    if (upload.state == 'done' || upload.state == 'uploading') {
                        var bar = document.getElementById('progressbar');
                        var w = 400 * upload.received / upload.size;
                        bar.style.width = w + 'px';
                    }
                    /* 上传完成，不再查询进度 */
                    if (upload.state == 'done') {
                        window.clearTimeout(interval);
                    }
                    if (upload.state == 'error') {
                        window.clearTimeout(interval);
                        uploadxhr.abort();
                        alert('something wrong');
                    }
                }
            }
        }
        req.send(null);
    }
</script>

```

### 另外
nginx-upload-module和nginx-upload-progress-module还提供了更多的指令，帮忙我们实现更复杂的上传文件功能，比如断点续传等，有兴趣可以阅读两个模块的官方文档，了解更多。
另外，因为nginx-upload-module未能及时拦下体积过大的文件上传，所以，尽管保障了用户的正常使用，可是依然不能防范恶意的流量攻击。nginx-upload-progress-module能够在一开始就检测到上传文件的体积是否过大（HTTP请求头里的`Content-Length`存有文件的体积大小），这时候就应该中断上传（可能是NginX限制，扩展模块无法中断HTTP请求），大家有兴趣的话可以研究一下NginX源码和扩展开发。

### 思考
NginX的`client_max_body_size`设为`32m`，攻击者可以上传1GB的文件，直到上传到32MB的时候，NginX才会中断上传，服务器被消耗了32MB的流量。细想一下：
1. 即使NginX在一开始就拦下了体积大于32MB的文件，可是攻击者依然可以直接上传30MB大小的文件，服务器还是会被消耗了30MB的流量，所以在一开始就拦截的意义并不大；
2. 可是上传文件的体积大于`client_max_body_size`时，nginx-upload-module的限速功能不起作用，这就成问题了；
3. NginX没有直接信任请求头的`Content-Length`，应该有他的依据，不过正常用户不会虚报吧（即使报小也不报大啊）；
4. 看来这个方案还需继续完善，或者借助现成的云存储服务来实现文件上传功能（可参考[腾讯云COS的一次实践](/2017/08/16/practice-qcloud-cos/)）。

## 最后
如果一个网站，允许用户全速上传文件，并持续数十秒，那么这个网站一定存在被流量攻击的风险，有可能是大量用户同时使用造成的，也有可能是恶意的DDoS攻击（？？好像所有网站都会有这个风险）。要是服务器带宽被占满，服务器对于一些用户就像是掉线了，所以上传文件的问题必须重视。另外，开发者不应该局限于一种编程语言或者一个知识领域上去思考解决问题，应该涉览更多的知识领域，从更多角度、更多方位去解决问题。

## 参考
1. [OPTIMIZED FILE UPLOADING WITH PHP & NGINX](https://blog.martinfjordvald.com/2010/08/file-uploading-with-php-and-nginx/)
2. [How to Upload Large Files in PHP](https://www.sitepoint.com/upload-large-files-in-php/)
3. [文件上传的渐进式增强](http://www.ruanyifeng.com/blog/2012/08/file_upload.html)
