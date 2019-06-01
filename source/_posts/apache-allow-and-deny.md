title: Apache HTTP Server中的Allow、Deny指令
date: 2015-10-06 21:24:13
tags: 
    - apache
categories:
    - 知道
---

> 在配置Apache HTTP Server的时候，经常会用到`Allow`、`Deny`这两个个指令，来实现目录权限控制、内外网隔离的功能。然而这两个指令是怎样相互配合使用的呢？

## mod_authz_host
mod_authz_host是Apache HTTP Server的一个官方标准模块，主要提供三个指令：`Allow`，`Deny`和`Order`，指令用在`<Directory>`, `<Files>`, `<Location>`中，也用于`.htaccess`文件中控制对服务器特定部分的访问。`Allow`和`Deny`指令用于指出允许哪些用户主机及不允许哪些用户主机访问服务器，而`Order`指令设置默认的访问状态并配置`Allow`和`Deny`指令怎样相互作用。

<!--more-->

## Allow
[Allow](http://httpd.apache.org/docs/2.4/mod/mod_access_compat.html#allow)，控制哪些主机能够访问服务器的某些区域。
用法：

```httpd
Allow from 10.1.2.3
```

## Deny
[Deny](http://httpd.apache.org/docs/2.4/mod/mod_access_compat.html#deny)，控制哪些主机被禁止访问服务器的某些区域。
用法：

```httpd
Deny from 10.1.2.3
```

## Order
[Order](http://httpd.apache.org/docs/2.4/mod/mod_access_compat.html#order)，控制默认的访问状态与`Allow`和`Deny`指令生效的顺序
在没有`Order`指令的情况下，`Allow`的优先级会比`Deny`高，如下配置实际上没有限制IP10.1.2.3主机的访问权限：

```httpd
Deny from 10.1.2.3
Allow from 10.1.2.3
Deny from 10.1.2.3
```

Order的有三种取值：
1. `Deny,Allow`
    Deny指令在Allow指令之前被评估。默认允许所有访问。任何不匹配Deny指令或者匹配Allow指令的客户都被允许访问。
2. `Allow,Deny`
    Allow指令在Deny指令之前被评估。默认拒绝所有访问。任何不匹配Allow指令或者匹配Deny指令的客户都将被禁止访问。
3. `Mutual-failure`
    只有出现在Allow列表并且不出现在Deny列表中的主机才被允许访问。这种顺序与"Order Allow,Deny"具有同样效果，不赞成使用。

关键字只能用逗号分隔`,`它们之间不能有空格。注意在所有情况下每个`Allow`和`Deny`指令语句都将被评估。
