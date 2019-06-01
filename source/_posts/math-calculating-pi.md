title: 怎样算出Pi（π）
date: 2017-03-14 12:03:14
tags: 
    - math
categories:
    - 求索

---

> 圆周率日（Pi Day，又译π节）是一年一度的庆祝数学常数π的节日，特地写一篇与π相关的文章。

## 一道题目

最近碰到一道数学编程题目，就是利用以下公式求出π的（近似）值：
$$  x - \frac{x^3} {3!} + \frac{x^5} {5!} - \frac{x^7} {7!} + \ldots = \sum_{n=0}^{+\infty} \frac{(-1)^n x^{2n+1}} {(2n+1)!} $$

网上搜了一下，原来这是正弦函数的泰勒级数展开式（高等数学早忘光了）。

<!--more-->
机智的我一下子就想到了`sin(π)=0`，代入上面公式，得到方程：
$$ sin(\pi) = \pi - \frac{\pi^3} {3!} + \frac{\pi^5} {5!} - \frac{\pi^7} {7!} + \ldots = 0 $$

于是：
$$ \pi = \frac{\pi^3} {3!} - \frac{\pi^5} {5!} + \frac{\pi^7} {7!} - \ldots $$

然后，解方程……呃……这个我不会！
再上网找找。

## 百科知道
有空要多看看百科知识，能学到很多东西。

### 梅钦公式
在维基百科的[圆周率](https://zh.wikipedia.org/wiki/%E5%9C%93%E5%91%A8%E7%8E%87)条目里，提到了计算π的梅钦公式：
$$ \frac{\pi} {4} = 4arctan\frac{1} {5} - arctan\frac{1} {239} $$

这个是利用`arctan(x)`的泰勒级数展开式，和`4arctan(1)=π`推导出来的。可是用`sin(x)`的泰勒级数展开式不能推导出`arctan(x)`的泰勒级数展开式，所以用梅钦公式求π就不和题意了。

### 巴塞尔问题
在维基百科的[巴塞尔问题](https://zh.wikipedia.org/wiki/%E5%B7%B4%E5%A1%9E%E5%B0%94%E9%97%AE%E9%A2%98)条目里，提到欧拉利用正弦函数的泰勒级数展开式，求证了：
$$ \frac{\pi} {6} = \frac{1} {1^2} + \frac{1} {2^2} + \frac{1} {3^2} + \frac{1} {4^2} + \ldots $$

可是求证过程中利用了零点代入方程求解，不算严谨，所以这条公式也不能用。

## 夹挤定理
关键方向是用正弦函数去求π，于是用`sin(x)`和`π`作为关键字，又在网上搜了一遍，终于发现了一篇“有趣”的文章——[一个有趣的圆周率计算公式 x=sin(x)+x](http://www.guokr.com/post/608913/)。文中主要讲作者“原创”了一种求π算法，就是“通过迭代计算x=sin(x)+x，逼近π”（原话）。

通过“逼近”来求π值，的确是很有意思的一种方法。可是作者只是给出了计算方法，却没有详细的推导过程。一开始对这种算法还是持怀疑态度，直到后来想起了[夹挤定理](https://zh.wikipedia.org/wiki/%E5%A4%BE%E6%93%A0%E5%AE%9A%E7%90%86)（也叫夹逼准则，其实不大记得这个名字，只是大概知道由它推导出来的极限公式），由夹挤定理可以很容易得到以下极限（一般高等数学书里都会提到）：
$$ \lim_{x \rightarrow 0} sin(x) = x $$

于是，有：
$$ \lim_{\pi-x \rightarrow 0} sin(\pi-x) = pi-x $$

又因为`sin(π-x)=sin(x)`，得：
$$ \lim_{\pi-x \rightarrow 0} sin(x) + x = pi $$

假设，x大于0小于π，则sin(x)大于0小于1；在x无限趋于π时，`sin(x)+x`也在无限趋于π，而且比x更趋于π（因为这里sin(x)不是负数）；所以，不断地让`x=sin(x)+x`那么x就可以无限地接近π了。

再用`sin(x)`的泰勒级数展开式代替`sin(x)`，那么求π的函数代码（javascript）就很容易写了：

```javascript
var INFINITY = 100; // 用100表示正无穷，越大越准，太大可能会溢出
var PI = 3; // x的初始值设为接近PI的数，比如3，也可以是1、2、2.1、2.2、...

function getPI(len) {
    if(PI>3) {
        return PI.toFixed(len);
    }
    for(var i=0; i<INFINITY; i++) {
        PI = taylorSin(PI)+PI;
    }
    return PI.toFixed(len);
}

function taylorSin(x) {
    var sinx = 0;
    for(var i=0; i<INFINITY; i++) {
        sinx+=Math.pow(-1, i)*Math.pow(x, i*2+1)/factorial(i*2+1);
    }
    return sinx;
}

function factorial(n) {
    var f = n;
    while(--n) {
        f*=n;
    }
    return f;
}

```

所以，π值怎么算？死循环算啊。

## 最后
其实，求π的方法还有很多，尽管π是一个无理数，有兴趣的读者可以浏览一下网页，特别是[贝利-波尔温-普劳夫公式](https://zh.wikipedia.org/wiki/%E8%B4%9D%E5%88%A9-%E6%B3%A2%E5%B0%94%E6%B8%A9-%E6%99%AE%E5%8A%B3%E5%A4%AB%E5%85%AC%E5%BC%8F)，其可以直接算出小数点后面的第n位数值：

- [如何计算圆周率 Pi](http://zh.wikihow.com/%E8%AE%A1%E7%AE%97%E5%9C%86%E5%91%A8%E7%8E%87-Pi)
- [贝利-波尔温-普劳夫公式](https://zh.wikipedia.org/wiki/%E8%B4%9D%E5%88%A9-%E6%B3%A2%E5%B0%94%E6%B8%A9-%E6%99%AE%E5%8A%B3%E5%A4%AB%E5%85%AC%E5%BC%8F)
- [高斯-勒让德算法](https://zh.wikipedia.org/wiki/%E9%AB%98%E6%96%AF-%E5%8B%92%E8%AE%A9%E5%BE%B7%E7%AE%97%E6%B3%95)
- [梅钦类公式](https://zh.wikipedia.org/wiki/%E6%A2%85%E6%AC%BD%E9%A1%9E%E5%85%AC%E5%BC%8F)
- [巴塞尔问题](https://zh.wikipedia.org/wiki/%E5%B7%B4%E5%A1%9E%E5%B0%94%E9%97%AE%E9%A2%98)

