title: SELECT INTO 和 INSERT SELECT INTO
date: 2016-10-06 10:06:13
tags: 
    - mysql
categories:
    - 知道
---

> 开发中经常会碰到把已存在记录的数据赋给新记录的场景，怎样才能把`SELECT`和`INSERT`合并到一个执行语句中呢？


## SELECT INTO

语法：

``` sql
SELECT `t1`.`column1` AS column1, `t1`.`column2` AS column2, 'column3' AS column3 INTO `Table2` FROM `Table1` AS t1;
```

注意：执行过程中，会创建表`Table2`，所以，执行前要求表`Table2`不存在。


## INSERT SELECT INTO

语法：

``` sql
Insert INTO `Table2` (column1, column2, ...) SELECT value1,value2,... FROM `Table1`;
```

注意：执行前要求表`Table2`存在切有固定结构；`SELECT`前面没有`VALUES`关键字；`FROM`后面可以跟更复杂的查询语句。

