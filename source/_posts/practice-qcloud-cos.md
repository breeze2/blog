title: 腾讯云COS的一次实践
date: 2017-08-16 10:05:13
tags: 
    - qcloud
    - php
    - js
    - practice
categories:
    - 实践
---

很多网站都会需要文件上传、下载的功能，但是服务器本身配置（如容量、带宽等等）可能扛不住大量的上传、下载，这时候，利用一些现成的云存储服务来分担一下服务器的压力。这里主要介绍一下腾讯云COS的使用，大概的功能流程是：
1. 前端（JS）实现文件上传功能；
2. 后端（PHP）给前端发放签名，使前端可以直接访问COS；
3. 根据COS生成的文件链接访问文件。

<!--more-->

## PHP后端签名机制

以Laravel项目为例：
首先，在项目目录下，新建目录`app/Plugins/Qcloud/Cos`；然后，下载[COS的PHP SDK](https://github.com/tencentyun/cos-php-sdk-v4/archive/master.zip)，将SDK中`src/qcloud/cos`下所有文件复制到项目的`app/Plugins/Qcloud/Cos`下，这里只用到`Auth.php`。大概代码：

routes/web.php
```php
<?php

Route::post('/qcloud/cos/sign', 'QcloudController@postCosSign');
Route::post('/qcloud/cos/sign-once', 'QcloudController@postCosSignOnce');

```

app/Plugins/Qcloud/Cos/Auth.php
```php
<?php
/**
 * Signature create related functions for authenticating with cos system.
 */
namespace App\Plugins\QCloud\Cos;// 加上命名空间
/**
 * Auth class for creating reusable or nonreusable signature.
 */
class Auth {
    ...
}
```

config/web.php
```php
<?php 
return [
    'qcloud_cos' => [
        'app_id' => '1234567890',
        'secret_id' => '************************************',
        'secret_key' => '********************************',
        'region' => 'gz', //广州
        'timeout' => 60
    ],
];

```

QcloudController.php
```php
<?php
use App\Plugins\QCloud\Cos\Auth as CosAuth;
class QcloudController extends Controller {
...
    // 多次复用的签名
    public function postQcloudCosSign(Request $request) {
        $team = $this->getAuthUser();
        $bucket = $request->input('bucket');
        $timeout = time()+2*60*60;
        $config = config('web.qcloud_cos');
        $cos_auth = new CosAuth($config['app_id'], $config['secret_id'], $config['secret_key']);
        $sign = $cos_auth->createReusableSignature($timeout, $bucket);
        return $this->responseJSON(['sign'=>$sign]);
    }
    // 单次使用的签名
    public function postQcloudCosSignOnce(Request $request) {
        $bucket = $request->input('bucket');
        $timeout = time()+2*60*60;
        $config = config('web.qcloud_cos');
        $cos_auth = new CosAuth($config['app_id'], $config['secret_id'], $config['secret_key']);
        $sign = $cos_auth->createNonreusableSignature($bucket, '/');
        return $this->responseJSON(['sign'=>$sign]);
    }
}
```

## JS前端上传文件

下载[COS的JS SDK](https://github.com/tencentyun/cos-js-sdk-v4/archive/master.zip)，将SDK中`dist/cos-js-sdk-v4.js`复制到项目的`public/js`下，注意使用SDK需要浏览器支持HTML 5
。大概代码：

```html

<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
</head>
<body>
    <div>
        <input type="file" name="upload_file" id="upload_file" />
        <button id="do_upload">请上传</button>
        <span></span>
    </div>
    <script type="text/javascript" src="/js/jquery.js"></script>
    <script type="text/javascript" src="/js/cos-js-sdk-v4.js"></script>
    <script type="text/javascript">
        var USERID = $('#user').data('id')
        var PREFIX = Date.now()+'_'+USERID+'_'
        var COS = new CosCloud({
          appid: '1234567890',
          bucket: 'public',
          region: 'gz',
          getAppSign: function (callback) {
            $.post('/qcloud/cos/sign').done(function (data) {
              var sign = data.sign
              callback(sign)
            }).fail(function () {
              B.alert('请求失败，稍后再试')
            })
          },
          getAppSignOnce: function (callback) {
            $.post('/qcloud/cos/sign-once').done(function (data) {
              var sign = data.sign
              callback(sign)
            }).fail(function () {
              B.alert('请求失败，稍后再试')
            })
          }
        })

        $('#do_upload').click(function () {
          var $this = $(this)
          var files = document.getElementById('upload_file').files
          if(!files.length || !files[0]) {
            return 0
          }
          var file = files[0]
          if(file.size && file.size < 50*1024*1024) {
            $this.prop('disabled', true)
            $this.text('开始上传 ')
            COS.uploadFile(function (result) {
              // successCallBack
              setTimeout(function() {
                $this.prop('disabled', true)
                $this.text('请上传 ')
              }, 1800)
              $this.text('上传成功 ')
            }, function (result) {
              // errorCallBack
              setTimeout(function() {
                $this.prop('disabled', true)
                $this.text('请上传 ')
              }, 1800)
              $this.text('上传失败 ')
            }, function (current) {
              // progressCallBack
              $this.text('正在上传 '+ (current*100).toFixed(1)+'%')
            }, COS.bucket, PREFIX+file.name, file, 0)
            
          } else {
            B.alert('文件不能大于50MB')
          }
        })
    </script>
</body>
</html>

```

## 其他

### 防盗链
如果不想存在COS上的文件，被其他网站盗用，那么可以在COS后台上给对应的存储痛（Bucket）设置`Referer`白名单，防止被恶意盗链。当然，伪造`Referer`的请求应该也防不了。

### 定时清理
因为前端可以直接与COS连接，上传的文件没有经过后端检验，所以有效的上传文件信息都应该记录在数据库里，然后在每天空闲时段将昨天上传的文件都遍历一遍（文件名前缀设为日期就可以据此检索），不记录在库的、类型不匹配的，体积过大的等此类文件都可以删除。

## 最后

一个网站本身要实现文件上传、下载功能并不难，但是总是会有各样的隐患（可参考[HTTP文件上传的一个后端完善方案（NginX）](/2017/08/12/scheme-nginx-php-js-upload-process/)）。于此，借用现成的云存储服务来实现文件上传、下载功能，也是一个不错的可行方案。


