title: OAuth2.0的授权模式和应用
date: 2017-10-06 10:06:13
tags: 
    - oauth2
categories:
    - 知道
---

<!-- 在过去，大多数Web网站都是B/S结构（浏览器/服务器结构），服务请求中用户的认证和授权，可以利用session来实现；现在，主流的Web应用都讲求前后分离，实际是向C/S结构（客户端/服务器结构）转型，然而客户端一般不提供session机制，那怎样对用户进行认证和授权呢？ -->

[OAuth](https://oauth.net/)（开放授权）是一个标准协议，为用户资源的鉴权问题提供了一个安全、开放而又简易的解决方案。[OAuth 2.0](https://oauth.net/2)定义了四种鉴权模式：
* 授权码准许（Authorization Code Grant）
* 隐式准许（Implicit Grant）
* 客户端准许（Client Credentials Grant）
* 密码准许（Resource Owner Password Credentials Grant）

这里只介绍这四种鉴权模式，即获取有效令牌（Token）的四种过程，关于OAuth 2.0的其他更多内容建议阅读[《理解OAuth 2.0》](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)及[ The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)。

<!--more-->

首先，OAuth 2.0鉴权是用户（Resource Owner）、第三方应用（Third-Party Application）和鉴权服务器（Authorization Server）三方面共同实现的，单靠其中之一不能做到。
第三方应用对于鉴权服务器，必须是可信的，彼此约定client_id和client_secret、redirect_uri等数据。所谓鉴权，实质是第三方应用向鉴权服务器申请有效令牌（Token），能获取到有效令牌，则说明鉴权通过，反之，不通过。


## 授权码准许

### 实现流程
授权码准许是基于链接跳转实现的鉴权模式，假设，
第三方应用地址：`https://client.com`
鉴权服务器地址：`https://server.com`
那么大概的鉴权流程是：
1. 第三方应用要申请令牌，首先带着client_id等参数跳转到鉴权服务器鉴权链接，用户在鉴权服务器上选择是否同意授权于第三方应用（用户要先在鉴权服务器上登录）；
2. 用户同意授权，鉴权服务器带着授权码code跳转到原本指定的第三方应用回调链接；
3. 第三方应用用授权码和client_secret等参数跟鉴权服务器换取令牌。

![授权码准许](/assets/images/oauth2-types-and-applications-of-the-grants1.png)

注释：
1. 链接跳转a形式一般是（`$`开头表示是变量）：
    `https://server.com/oauth/authorize?client_id=$CLIENT_ID&response_type=code&redirect_uri=$REDIRECT_URI&scope=$SCOPE&state=$STATE`；
2. 链接跳转b形式一般是：
    `https://client.com/oauth/callback?code=$CODE&state=$STATE`；
3. post请求c形式一般是：`https://server.com/oauth/token`，参数形式：
```json
{
    grant_type: 'authorization_code',
    client_id: $CLIENT_ID,
    client_secret: $CLIENT_SECRET,
    redirect_uri: $REDIRECT_URI,
    code: $CODE
}
```
4. post响应d形式一般是：
```json
{
    token_type: 'Bearer',
    expires_in: $EXPIRES_IN,
    access_token: $ACCESS_TOKEN,
    refresh_token: $REFRESH_TOKEN
}
```

### 应用场景
授权码准许的鉴权模式是四种模式当中功能最完整、流程最严密的。使用授权码准许的第三方应用，一般是B/S结构（浏览器/服务器结构）。服务提供商（HTTP Service）给第三方应用颁发client_id和client_secret，第三方应用的后台服务器据此与服务提供商的鉴权服务器进行互动。
比如，微信公众号开发中，获取当前访问用户信息就采用了授权码准许的鉴权模式，详见[微信网页授权](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842)。

## 隐式准许
### 实现流程
隐式准许是授权码准许的简化，因为用户同意授权就可以直接发放令牌，不需要用授权码来再次请求令牌。
大概的鉴权流程是：
1. 第三方应用要申请令牌，首先带着client_id等参数跳转到鉴权服务器鉴权链接，用户在鉴权服务器上选择是否同意授权于第三方应用（用户要先在鉴权服务器上登录）；
2. 用户同意授权，鉴权服务器带着令牌等参数跳转到原本指定的第三方应用回调链接。

![隐式准许](/assets/images/oauth2-types-and-applications-of-the-grants2.png)

注释：
1. 链接跳转a形式一般是（`$`开头表示是变量）：
    `https://server.com/oauth/authorize?client_id=$CLIENT_ID&response_type=code&redirect_uri=$REDIRECT_URI&scope=$SCOPE&state=$STATE`；
2. 链接跳转b形式一般是：
    `https://client.com/oauth/callback#token_type=Bearer&expires_in=$EXPIRES_IN&access_token=$ACCESS_TOKEN&state=$STATE`；

这里解释一下，鉴权服务器跳回到第三方应用时，为什么传递参数是以hash形式而不是query形式？因为链接的hash数据只在本地浏览器里传播，再结合HTTPS，整个网络传输过程中，链接hash中令牌等信息并没有暴露；而链接的query数据是必然会暴露的。

### 应用场景
隐式准许的鉴权模式最终是以链接的hash形式传递令牌，整个操作流程只能在浏览器环境中进行。但换个角度看，第三方应用不需要经过自身后台服务器，直接可以在浏览器中向鉴权服务器申请令牌，所以隐式准许的鉴权模式十分适用于当下流行的前后分离的单页应用或者基于Webview的手机应用。
隐式准许的鉴权模式不检验client_secret，不过鉴权服务器只会回跳到可信的第三方应用回调链接（初始约定），所以鉴权过程依然是安全的，再说client_secret放在前端JavaScript代码中很容易暴露。

## 客户端准许

### 实现流程
客户端准许，注意这里说的客户端说的是第三方应用，但它不是一个PC程序或者手机应用，把它理解为一个机器或者服务器可能更为合理。整个鉴权流程只有第三方应用（Third-Party Application）和鉴权服务器（Authorization Server）参与，不需要用户（Resource Owner）同意授权。

大概的鉴权流程是：
1. 第三方应用用client_id和client_secret等参数向鉴权服务器申请令牌；
2. 鉴权服务器给第三方应用发放令牌。

![客户端准许](/assets/images/oauth2-types-and-applications-of-the-grants3.png)

注释：
1. 申请令牌时，post请求链接一般是：`https://server.com/oauth/token`，参数形式：
```json
{
    grant_type: 'client_credentials',
    client_id: $CLIENT_ID,
    client_secret: $CLIENT_SECRET,
    scope: $SCOPE
}
```
2. 发放令牌时，post响应一般是：
```json
{
    token_type: 'Bearer',
    expires_in: $EXPIRES_IN,
    access_token: $ACCESS_TOKEN
}
```

### 应用场景
客户端准许不需要经过用户同意授权，相当于服务提供商直接将资源分享给可信的第三方应用，令牌只是第三方应用访问服务提供商接口所需的一个凭证。
比如，微信公众号开发中，公众号服务器调用微信接口前需要经过客户端准许，详见[获取access_token](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140183)。

## 密码准许

### 实现流程
密码准许是用户将自己的用户名和密码交与第三方应用，第三方应用据此向鉴权服务器申请令牌。
大概的鉴权流程是：
1. 用户在第三方应用上输入自己的用户名和密码，第三方应用据此向鉴权服务器申请令牌；
2. 鉴权服务器给第三方应用发放令牌。

![密码准许](/assets/images/oauth2-types-and-applications-of-the-grants4.png)

注释：
1. 申请令牌时，post请求链接一般是：`https://server.com/oauth/token`，参数形式：
```json
{
    grant_type: 'password',
    client_id: $CLIENT_ID,
    client_secret: $CLIENT_SECRET,
    username: $USERNAME,
    password: $PASSWORD,
    scope: $SCOPE
}
```
2. 发放令牌时，post响应一般是：
```json
{
    token_type: 'Bearer',
    expires_in: $EXPIRES_IN,
    access_token: $ACCESS_TOKEN,
    refresh_token: $REFRESH_TOKEN
}
```

### 应用场景
密码准许的鉴权模式适用于没有浏览器环境、没有后台服务器的第三方应用，这种应用是一个纯客户端，比如PC程序或者手机应用（区别客户端准许）。鉴权服务器一般不会对第三方应用做密码准许，因为涉及到用户的密码，除非第三方应用十分可信。比如，Facebook原本是一个Web网站，后来要开发手机应用Facebook App，那么Facebook Web可以对Facebook App做密码准许，因为Facebook App是Facebook自家产品，不是真正意义上的第三方应用。
即使第三方应用非常可信，第三方应用在功能设计上也不能存储用户输入的密码，万一被破解，用户的密码就会暴露。

## 最后

最后总结一下：
1. 授权码准许和隐式准许中用户同意授权的操作是首次必须，用户同意后下次默认直接同意（当然也可以设置有效期）；
2. 授权码准许和密码准许会返回refresh_token，令牌过期时，第三方应用可以用refresh_token隐式刷新令牌；
3. 客户端准许和隐式准许不会返回refresh_token，令牌过期时，要求第三方应用重新申请令牌，客户端准许模式重新申请令牌对用户无干扰，而隐式准许模式可以利用隐形iframe重新申请令牌来减少对用户的干扰；
4. 对于B/S结构的Web应用，建议使用授权码准许的鉴权模式；
5. 对于前后分离的Web单页应用，建议使用隐式准许的鉴权模式；
6. 对于共享资源，与用户意愿无关的应用，建议使用客户端准许的鉴权模式；
7. 对于自家PC或手机应用产品，建议使用密码准许的鉴权模式。

