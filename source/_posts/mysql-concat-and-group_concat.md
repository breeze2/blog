title: CONCAT 和 GROUP_CONCAT
date: 2016-10-06 12:32:06
tags: 
    - mysql
categories:
    - 知道
---

> 在MySQL数据库中查询数据的时候，如果查询语句里使用了`GROUP BY`，那么`SELECT`表字段时只能获取分组内第一条记录的数据值，要怎样才能得到分组内所有记录的字段数据值呢？


## CONCAT
MySQL中，主要的连接字符串函数，就是 **CONCAT**。`CONCAT`函数用于将多个数据值连接成一个字符串。语法：

``` sql
CONCAT(str1, str2, …)
```

注意：如果有任何一个参数为`NULL`，那么返回结果就是`NULL`。
怎样避免参数为`NULL`的情况，当然是使用`CASE WHEN`：

``` sql
CONCAT(CASE WHEN str1 is NULL THEN "NULL" ELSE str1 END, CASE WHEN str2 is NULL THEN "NULL" ELSE str2 END, …)
```

<!--more-->

---

还有个连接字符串的函数，**CONCAT_WS**，表示“CONCAT With Separator”，是`CONCAT`的特殊形式。`CONCAT_WS`函数用于将多个数据值用指定的分隔符连接成一个字符串。语法：

``` sql
CONCAT_WS(separator, str1, str2, …)
```

注意：如果第一参数，即分隔符，设置为空字符串`''`，相当于移除分隔符，设置为`NULL`，那么返回结果就是`NULL`；其他参数如果为`NULL`，就会被忽略，进而连接下个参数，但是空字符串`''`不会被忽略。
为了保证连接数据的顺序，还得使用`CASE WHEN`：

``` sql
CONCAT_WS(separator, CASE WHEN str1 is NULL THEN "NULL" ELSE str1 END, CASE WHEN str2 is NULL THEN "NULL" ELSE str2 END, …)
```


## GROUP_CONCAT
`GROUP_CONCAT`函数可以将分组内的数据连接合成一个字符串。因为是做分组内操作，所以`GROUP_CONCAT`的功能会复杂一些，处理可以设置分隔符，还可以对记录数据做去重，排序处理。在逻辑上，`GROUP_CONCAT`应该是与`GROUP BY`配合使用的，当然就算查询语句中没有出现`GROUP BY`，也是可以使用`GROUP_CONCAT`的，就是把整个记录数据看作一个组了。语法：

``` sql
GROUP_CONCAT(
    [DISTINCT] expr [,expr ...]
    [ORDER BY {unsigned_integer | col_name | formula}
        [ASC | DESC] [,col_name ...]]
    [SEPARATOR str_val])
```

注意：分隔符缺省为一个逗号`","`，可以设置为空字符串`''`，相当于移除分隔符，但是不能设置为`NULL`，否则报错；如果某个连接参数的值为`NULL`，就会被忽略，进而连接下个参数，但是空字符串`''`不会被忽略。

### 应用

在MySQL数据库中查询数据的时候，如果查询语句里使用了`GROUP BY`，那么`SELECT`表字段时只能获取分组内第一条记录的数据值，要怎样才能得到分组内所有记录的字段数据值呢？

假设，有两张表，`users`和`friends`：
users:

| id | name |
| --- | --- |
| 1 | user1 |
| 2 | user2 |
| 3 | user3 |

friends:

| id | user_id | name | create_at |
| --- | --- | --- | --- |
| 1 | 1 | friend1 | 1457452801 |
| 2 | 1 | friend2 | 1457452802 |
| 3 | 1 | friend3 | 1457452803 |
| 4 | 2 | friend4 | 1457452804 |
| 5 | 2 | friend5 | 1457452805 |
| 6 | 2 | friend6 | 1457452806 |
| 7 | 3 | friend7 | 1457452807 |
| 8 | 3 | friend8 | 1457452808 |
| 9 | 3 | friend9 | 1457452809 |

想要获取每一个`user`的所有`firend`，并且按照`create_at`顺排序的话，可以使用`GROUP_CONCAT`和`CONCAT_WS`：

``` sql
SELECT
    u.id as user_id,
    u.name as user_name,
    GROUP_CONCAT(
        CONCAT_WS("%", f.id, f.name)
        ORDER BY f.create_at ASC
        SEPARATOR ";"
    ) as friends_group
FROM
    users as u
    JOIN friends as f on u.id = f.user_id
WHERE
    1;
```

## 最后

最后说两句，在MySQL中查询数据时，要获取分组内的数据时，可以考虑使用`GROUP_CONCAT`。还有组内数据排序，也可以借用`GROUP_CONCAT`。

