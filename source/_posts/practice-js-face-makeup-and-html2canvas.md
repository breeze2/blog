title: JS实现人脸换妆和页面截图
date: 2017-11-20 09:46:23
tags: 
    - js 
    - practice
categories:
    - 实践
---

> 最近要做一个H5活动页面，大概功能是给图片人脸换妆，自然也少不了上传原始图片和导出最终图片等功能。如果图片加工（即人脸换妆）放在服务器处理，那么并发量高的时候，肯定产生大量阻塞。于是想把图片加工放在客户端处理，用HTML堆砌出图片的最终效果，但是怎样把HTML导出成图片呢？

## 图片上传
图片上传可以参考[HTTP文件上传的一个后端完善方案（NginX）](https://blog.breezelin.site/2017/08/12/scheme-nginx-php-js-upload-process/)，或者使用现有的云存储服务，如[七牛](https://www.qiniu.com/)等等。

## 五官定位
人脸识别和五官定位可以利用[OpenVC](https://opencv.org/)自行实现，或者使用成熟的云识别服务，如[Face++](https://www.faceplusplus.com.cn/)等等。这里使用腾讯云的人脸识别服务。

<!--more-->

## 人脸换妆
假设有一张图片，识别结果如下：
![识别结果](/assets/images/practice-js-face-makeup-and-html2canvas1.png)
> 图片来源：[万象优图-腾讯云](https://cloud.tencent.com/act/event/ci_demo.html)

根据五官定位点，在适当位置给人脸添上妆饰，比如小胡子：
![小胡子](/assets/images/practice-js-face-makeup-and-html2canvas2.png)

源码：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
</head>
<body style="padding: 0; margin: 0;">
    <div style="width: 100%; text-align: center;">
        <div style="width: 336px; margin-left: auto; margin-right: auto; position: relative;" id="div1">
            <img src="./face.png" style="width: 100%;" id="face" />
        </div>
    </div>
    <script type="text/javascript">
        var FACE = {
            "session_id":"",
            "face_shape":[
                {
                    "face_profile":[{"x":49,"y":231},{"x":59,"y":249},{"x":71,"y":267},{"x":84,"y":284},{"x":100,"y":298},{"x":118,"y":310},{"x":138,"y":318},{"x":158,"y":324},{"x":178,"y":329},{"x":198,"y":329},{"x":217,"y":323},{"x":232,"y":309},{"x":239,"y":291},{"x":244,"y":271},{"x":249,"y":252},{"x":252,"y":232},{"x":253,"y":213},{"x":251,"y":194},{"x":247,"y":175},{"x":240,"y":157},{"x":232,"y":142}],
                    "left_eye":[{"x":89,"y":209},{"x":100,"y":210},{"x":110,"y":207},{"x":120,"y":202},{"x":128,"y":196},{"x":117,"y":190},{"x":105,"y":191},{"x":95,"y":198}],
                    "right_eye":[{"x":209,"y":153},{"x":204,"y":161},{"x":196,"y":167},{"x":186,"y":170},{"x":176,"y":172},{"x":179,"y":161},{"x":187,"y":154},{"x":198,"y":150}],
                    "left_eyebrow":[{"x":58,"y":190},{"x":72,"y":180},{"x":89,"y":175},{"x":106,"y":172},{"x":122,"y":167},{"x":105,"y":161},{"x":86,"y":165},{"x":68,"y":174}],
                    "right_eyebrow":[{"x":214,"y":120},{"x":200,"y":125},{"x":187,"y":133},{"x":175,"y":143},{"x":162,"y":150},{"x":169,"y":135},{"x":182,"y":123},{"x":198,"y":115}],
                    "mouth":[{"x":149,"y":279},{"x":165,"y":290},{"x":184,"y":294},{"x":202,"y":289},{"x":215,"y":276},{"x":219,"y":258},{"x":219,"y":239},{"x":208,"y":241},{"x":196,"y":246},{"x":189,"y":253},{"x":178,"y":256},{"x":163,"y":266},{"x":165,"y":282},{"x":181,"y":281},{"x":197,"y":277},{"x":207,"y":266},{"x":214,"y":254},{"x":209,"y":245},{"x":200,"y":252},{"x":191,"y":258},{"x":177,"y":266},{"x":163,"y":273}],
                    "nose":[{"x":180,"y":231},{"x":155,"y":187},{"x":154,"y":201},{"x":154,"y":216},{"x":153,"y":230},{"x":152,"y":247},{"x":170,"y":248},{"x":186,"y":245},{"x":196,"y":235},{"x":203,"y":219},{"x":190,"y":211},{"x":178,"y":203},{"x":167,"y":195}]
                }
            ],
            "image_height":430,
            "image_width":336
        }; // 人脸识别结果
        var BEARD = {
            "src":"./beard.png",
            "image_height":222,
            "image_width":567
        }; // 胡子图片信息

        function parseEyesData(leye, reye){
            var lx = ly = 0;
            var rx = ry = 0;
            for(var i in leye) {
                lx += leye[i]['x']; ly += leye[i]['y'];
            }
            lx = lx/leye.length; ly = ly/leye.length;
            for(var i in reye) {
                rx += reye[i]['x']; ry += reye[i]['y'];
            }
            rx = rx/leye.length; ry = ry/leye.length;
            return {
                angle: Math.atan((ly-ry)/(lx-rx))/Math.PI*(180),
                span: Math.sqrt(Math.pow(lx-rx, 2)+Math.pow(ly-ry, 2))
            }
        } // 根据眼睛数据，获取脸倾角和眼心距

        var div1 = document.getElementById('div1');
        var edata = parseEyesData(FACE['face_shape'][0]['left_eye'], FACE['face_shape'][0]['right_eye']);
        var beard = document.createElement('img');
        var scale = div1.offsetWidth/FACE['image_width'];
        
        var point1 = FACE['face_shape'][0]['nose'][0]; // 鼻尖
        var point2 = FACE['face_shape'][0]['nose'][7]; // 鼻低
        var point3 = FACE['face_shape'][0]['mouth'][9]; //上唇上中点
        var beardx = ((point1['x']+point2['x'])/2+point3['x'])/2*scale; // 大概算出人中X坐标
        var beardy = ((point1['y']+point2['y'])/2+point3['y'])/2*scale; // 大概算出人中Y坐标
        var beardw = edata['span']*scale; // 胡子长度取眼心距值
        beard.width = beardw;
        beard.src = './beard.png';
        beard.style.transform = 'rotate('+edata['angle']+'deg)';
        beard.style.position = 'absolute';
        beard.style.top = (beardy-beardw/2/(BEARD['image_width']/BEARD['image_height']))+'px';
        beard.style.left = (beardx-beardw/2)+'px';
        
        div1.appendChild(beard);
    </script>
</body>
</html>
```

最终效果：
![最终效果](/assets/images/practice-js-face-makeup-and-html2canvas3.png)

这只是HTML，怎样保存成图片呢？

## 导出图片
把HTML保存成图片，在服务器上可以用[PhantomJS](http://phantomjs.org/)实现，在浏览器上呢？其实也是可以实现的，首先将HTML（DOM结构）写入Canvas，Canvas再导出DataURL，最后DataURL赋予img标签的src属性，便得到最终图片，详情请阅读[Drawing DOM objects into a canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Drawing_DOM_objects_into_a_canvas)。

这里使用插件[html2canvas](https://github.com/niklasvh/html2canvas)来实现HTML写入Canvas功能，类似的插件还有[rasterizeHTML.js](https://github.com/cburgmer/rasterizeHTML.js)等等。

源码：
```html
    <script type="text/javascript" src="./html2canvas.min.js"></script>
    <script type="text/javascript">
        ...

        var can1 = document.createElement('canvas');
        var img1 = document.createElement('img');
        var ctxt = can1.getContext('2d');
        var level = 2;
        can1.width = document.body.offsetWidth*level;
        can1.height = div1.offsetHeight*level;
        can1.style.width = can1.width + 'px';
        can1.style.height = can1.height + 'px';
        ctxt.scale(level, level); // 使用两倍图，保持清晰度

        setTimeout(function () {
            html2canvas(div1, {canvas:can1}).then(function(can1) {
                var can2 = document.createElement('canvas');
                var face = document.getElementById('face');
                var ctxt = can1.getContext('2d');
                var level = 2;
                can2.width = face.offsetWidth*level;
                can2.height = face.offsetHeight*level;
                can2.style.width = face.offsetWidth + 'px';
                can2.style.height = face.offsetWidth + 'px';
                ctxt.scale(level, level);
                can2.getContext('2d').drawImage(can1,
                div1.offsetLeft*level, 0, face.offsetWidth*level, face.offsetHeight*level,
                    0, 0, face.offsetWidth*level, face.offsetHeight*level
                ); // 使用can2是为了裁掉与人脸图片无关的边缘
                var data = can2.toDataURL();
                img1.src = data;
                document.body.appendChild(img1); // 把图片放入当前页面中
                can2 = null;
                can1 = null;
            });
        }, 2000); // 需要等待人脸图片加载完成再执行
    </script>
```

在PC端可以用a标签下载图片，href属性值应该是canvas导出的DataURL：
```html
<a href="DataURL" download="最终效果" id="a1">下载</a>
```
在移动端则需要显示图片，并引导用户长按保存图片。

## 最后

注意：如果最终图片未能写入canvas，可能是图片跨域问题，或者图片未加载完就开始写了。
另外：可以分析图片人脸的表情、姿态、年纪、心情等属性，搭配更合适的妆饰，效果更佳；原始图片与妆饰图片可能会有色调、分辨率等不相配的问题，可以使用CSS3滤镜调整，使得最终画面更融洽。


