title: 一些有趣的MySQL查询问题
date: 2018-03-02 10:02:13
tags: 
    - mysql
    - leetcode
categories:
    - 笔记
---

> 在[LeetCode](https://leetcode.com/)上有一些有趣的MySQL查询问题，这里记录下来，方便以后查阅。

<!--more-->

## Department Top Three Salaries
部门前三薪资问题：查询各个部门薪资前三的员工（薪资），并列排名只占一位。
### 原题
The Employee table holds all employees. Every employee has an Id, and there is also a column for the department Id.
```sql
+----+-------+--------+--------------+
| Id | Name  | Salary | DepartmentId |
+----+-------+--------+--------------+
| 1  | Joe   | 70000  | 1            |
| 2  | Henry | 80000  | 2            |
| 3  | Sam   | 60000  | 2            |
| 4  | Max   | 90000  | 1            |
| 5  | Janet | 69000  | 1            |
| 6  | Randy | 85000  | 1            |
+----+-------+--------+--------------+
```
The Department table holds all departments of the company.
```sql
+----+----------+
| Id | Name     |
+----+----------+
| 1  | IT       |
| 2  | Sales    |
+----+----------+
```
Write a SQL query to find employees who earn the top three salaries in each of the department. For the above tables, your SQL query should return the following rows.
```
+------------+----------+--------+
| Department | Employee | Salary |
+------------+----------+--------+
| IT         | Max      | 90000  |
| IT         | Randy    | 85000  |
| IT         | Joe      | 70000  |
| Sales      | Henry    | 80000  |
| Sales      | Sam      | 60000  |
+------------+----------+--------+
```

### 答案
先说答案：
```sql
select dep.Name as "Department", emp.Name as "Employee", emp.Salary as "Salary"
from Department as dep join Employee as emp on dep.Id = emp.DepartmentId
where exists (select 'x' from Employee as emp1
    where emp1.Salary > emp.Salary and emp1.DepartmentId = emp.DepartmentId having count(distinct emp1.Salary) < 3
);
```
首先Department和Employee联接是不可少的，因为结果包含两张表的字段；再用子查询（exists），保留同部门内高于本身薪资的不超过三个的员工（having），并列排名只占一位，需要去重（distinct）。

注意：以上`'x'`并无特殊意义，只是有值时表示存在，无值时表示不存在。；`having`应该配合`group by`使用，单独使用`having`时视当前数据为一组；`count()`函数也是，单独使用`having`时也相当于给当前数据分成一组。

另外：只用一个`select`（没有子查询）也能实现，就是将Employee和自身`join`，`on`薪资比自己高的员工；然后用`group by`去重，再结合`having count(distinct emp1.Salary) < 3`，保留有效结果。这个`join`方法效率远低于上面子查询方法，所以不要对子查询有偏见。

## Human Traffic of Stadium
连续达标问题：查询连续3天体育馆人数高于100的日期。
### 原题
X city built a new stadium, each day many people visit it and the stats are saved as these columns: id, date, people
Please write a query to display the records which have 3 or more consecutive rows and the amount of people more than 100(inclusive).
For example, the table stadium:
```sql
+------+------------+-----------+
| id   | date       | people    |
+------+------------+-----------+
| 1    | 2017-01-01 | 10        |
| 2    | 2017-01-02 | 109       |
| 3    | 2017-01-03 | 150       |
| 4    | 2017-01-04 | 99        |
| 5    | 2017-01-05 | 145       |
| 6    | 2017-01-06 | 1455      |
| 7    | 2017-01-07 | 199       |
| 8    | 2017-01-08 | 188       |
+------+------------+-----------+
```
For the sample data above, the output is:
```sql
+------+------------+-----------+
| id   | date       | people    |
+------+------------+-----------+
| 5    | 2017-01-05 | 145       |
| 6    | 2017-01-06 | 1455      |
| 7    | 2017-01-07 | 199       |
| 8    | 2017-01-08 | 188       |
+------+------------+-----------+
```
Note:
Each day only have one row record, and the dates are increasing with id increasing.
### 答案
思路是先找出人数大于100的日期，然后在这些日期附近再找数大于100的日期，保留能够形成连续3天的日期。
提示里说表中每一天都有一条记录，而且日期是随id递增，所以我们只需要计算id，而不是日期。
用子查询的方法：
```sql
select s.id, s.date, s.people 
from stadium as s 
where s.people >= 100 and exists(select 'x'
    from stadium as s1
    where s1.people >= 100
        and s1.id >= s.id-2
        and s1.id <= s.id+2
        and s1.id<>s.id
    having count(s1.id) > 2
        or (count(s1.id) = 2 
            and (sum(s1.id-s.id) not in (-1,1) and sum(abs(s1.id-s.id)) <> 4)
        )
    );
```
在当前日期的前2天和后2天找，找到人数大于100的日期个数大于2的话，那么当前日期必定可以形成连续3天人数大于100；找到人数大于100的日期个数等于2，且这两个日期与当前日期的距离和不是1或-1，绝对距离和不是4的话，那么当前日期也是可以形成连续3天人数大于100；其他情况都不行。

用`join`的方法：
```sql
select distinct t1.*
from stadium t1, stadium t2, stadium t3
where t1.people >= 100 and t2.people >= 100 and t3.people >= 100
    and ( (t1.id - t2.id = 1 and t1.id - t3.id = 2 and t2.id - t3.id =1) -- t1, t2, t3
        or (t2.id - t1.id = 1 and t2.id - t3.id = 2 and t1.id - t3.id =1) -- t2, t1, t3
        or (t3.id - t2.id = 1 and t2.id - t1.id =1 and t3.id - t1.id = 2) -- t3, t2, t1
    )
order by t1.id;
```
同样，上面的子查询方法依然比这个join方法效率高。

## Trips and Users

### 原题
用户取消叫车服务问题：统计一个时间段内每天用户取消叫车服务占所有叫车服务的比例。
The Trips table holds all taxi trips. Each trip has a unique Id, while Client_Id and Driver_Id are both foreign keys to the Users_Id at the Users table. Status is an ENUM type of (‘completed’, ‘cancelled_by_driver’, ‘cancelled_by_client’).
```sql
+----+-----------+-----------+---------+--------------------+----------+
| Id | Client_Id | Driver_Id | City_Id |        Status      |Request_at|
+----+-----------+-----------+---------+--------------------+----------+
| 1  |     1     |    10     |    1    |     completed      |2013-10-01|
| 2  |     2     |    11     |    1    | cancelled_by_driver|2013-10-01|
| 3  |     3     |    12     |    6    |     completed      |2013-10-01|
| 4  |     4     |    13     |    6    | cancelled_by_client|2013-10-01|
| 5  |     1     |    10     |    1    |     completed      |2013-10-02|
| 6  |     2     |    11     |    6    |     completed      |2013-10-02|
| 7  |     3     |    12     |    6    |     completed      |2013-10-02|
| 8  |     2     |    12     |    12   |     completed      |2013-10-03|
| 9  |     3     |    10     |    12   |     completed      |2013-10-03| 
| 10 |     4     |    13     |    12   | cancelled_by_driver|2013-10-03|
+----+-----------+-----------+---------+--------------------+----------+
```
The Users table holds all users. Each user has an unique Users_Id, and Role is an ENUM type of (‘client’, ‘driver’, ‘partner’).
```sql
+----------+--------+--------+
| Users_Id | Banned |  Role  |
+----------+--------+--------+
|    1     |   No   | client |
|    2     |   Yes  | client |
|    3     |   No   | client |
|    4     |   No   | client |
|    10    |   No   | driver |
|    11    |   No   | driver |
|    12    |   No   | driver |
|    13    |   No   | driver |
+----------+--------+--------+
```
Write a SQL query to find the cancellation rate of requests made by unbanned clients between Oct 1, 2013 and Oct 3, 2013. For the above tables, your SQL query should return the following rows with the cancellation rate being rounded to two decimal places.
```sql
+------------+-------------------+
|     Day    | Cancellation Rate |
+------------+-------------------+
| 2013-10-01 |       0.33        |
| 2013-10-02 |       0.00        |
| 2013-10-03 |       0.50        |
+------------+-------------------+
```

### 答案
```sql
select Request_at as "Day", round(1-sum(t.Status="completed")/count(t.id), 2) as "Cancellation Rate"
from Trips as t join users as u on u.Users_Id = t.Client_Id and u.Banned = "No"
where t.Request_at >='2013-10-01' and t.Request_at<='2013-10-03'
group by t.Request_at;
```

## Median Employee Salary
### 原题
中位薪资问题：查询各家公司中薪资排中位的的员工（薪资）。
The Employee table holds all employees. The employee table has three columns: Employee Id, Company Name, and Salary.
```sql
+-----+------------+--------+
|Id   | Company    | Salary |
+-----+------------+--------+
|1    | A          | 2341   |
|2    | A          | 341    |
|3    | A          | 15     |
|4    | A          | 15314  |
|5    | A          | 451    |
|6    | A          | 513    |
|7    | B          | 15     |
|8    | B          | 13     |
|9    | B          | 1154   |
|10   | B          | 1345   |
|11   | B          | 1221   |
|12   | B          | 234    |
|13   | C          | 2345   |
|14   | C          | 2645   |
|15   | C          | 2645   |
|16   | C          | 2652   |
|17   | C          | 65     |
+-----+------------+--------+
```
Write a SQL query to find the median salary of each company. Bonus points if you can solve it without using any built-in SQL functions.
```sql
+-----+------------+--------+
|Id   | Company    | Salary |
+-----+------------+--------+
|5    | A          | 451    |
|6    | A          | 513    |
|12   | B          | 234    |
|9    | B          | 1154   |
|14   | C          | 2645   |
+-----+------------+--------+
```

### 答案
```sql
select emp.Id, emp.Company, emp.Salary
from Employee as emp
join (select Company, count(*) as EmployeeNum from Employee group by Company) as emp1
on emp1.Company = emp.Company
where exists (select 'x' from Employee as emp2
    where emp2.Company = emp.Company and emp2.Salary < emp.Salary
    having count(emp2.Id) in ( floor((EmployeeNum-1)/2), floor(EmployeeNum/2) )
);
```

