title: 批量更新短ID
date: 2016-11-12 11:35:13
tags: 
    - mysql
categories:
    - 躬行
---

> 生成唯一码（`ID`）的方案有很多，一般要减少冲突，就要增加码长。可是，很多应用场景中（比如把唯一码附在URL中），我们就需要比较短的唯一码。

## Hashids

这里先推荐一个生成短ID的开源库——[Hashids](http://hashids.org/)，它的实现原理大概是根据长的唯一码，通过转码，生成短的唯一码。如果想要得到尽量短的唯一码，那么原码应该是纯数字ID。

## Mysql的自增ID

以`Mysql`的自增ID作为原码，借助`Hashids`，就可以得到比较短的唯一码。插入一条记录再更新字段，其实现过程是先`Insert`，然后`Update`。但是，如果是批量插入记录，那又应该怎样更新短ID字段呢？

<!--more-->

## 批量更新不同值

因为短ID来源于自增ID，所以必然是先插入记录，才能获取到短ID。在批量插入记录后，再一条一条地去更新短ID字段，效率肯定很慢。
1. 如果有`Hashids`在`Mysql`中有实现函数（比如说，函数名字叫`HASHID`）的话，那么更新语句可以这样写：

``` sql
UPDATE table_name SET `short_id` = HASHID(`id`) WHERE `short_id` IS NULL;
```

2. 事实是目前还没有实现`HASHID`这个函数，这里记下一个方法，以备后忘：

``` php
// with laravel

$list = DB::table('table_name')->whereNull('short_id')->select('id')->get();
$ids= [];
$sql = ' CASE id ';
foreach($list as $k=>$v) {
    $ids[] = $v->id;
    $short_id = Hashids::encode($v->id);
    $sql .= sprintf('WHEN %d THEN "%s" ', $v->id, $short_id);
}
$sql .= ' END ';

DB::table('table_name')->whereIn('id', $ids)->update(['short_cdkey'=>DB::raw($sql)]);
```

## 最后两句

`CASE WHEN`是一个好东西，要多灵活使用。

