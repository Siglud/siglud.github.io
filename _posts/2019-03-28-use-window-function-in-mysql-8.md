---
layout: post
title:  "在MySQL8中使用开窗函数"
date:   2019-03-28 20:20:00 +0800
author: Siglud
categories:
  - MySQL
tags:
  - MySQL
comment: true
share: true
---
MySQL8之后增加了一部分开窗函数，这样就让统计计算又简单了一些，比如对于以前而言比较蛋疼的，求每组分值最高的前三个人这样的计算

假设有以下表
```sql
CREATE TABLE user_score (
    user_id int(11) NOT NULL AUTO_INCREMENT COMMENT 'user id',
    user_group_id int(11) NOT NULL COMMENT 'user group id',
    user_score int(11) NOT NULL COMMENT 'user score',
    PRIMARY KEY (user_id),
    KEY user_group (user_group_id)
) DEFAULT CHARSET=utf8mb4 COMMENT='user score test';
```

如果需要统计一个组中分数前三的用户ID，现在只用这样写就可以了

```sql
SELECT u.user_id, u.user_order FROM
(SELECT user_id, row_number() OVER (PARTITION BY user_group_id ORDER BY user_score DESC) AS user_order FROM user_score) u WHERE u.user_order < 4;
```
但是注意同分数的情况，会有不同的处理方式

row_number()是无脑的对现有的数据进行排序，而rank()则是会把同分的人视作是同排序，会产生类似1、2、2、4、5……这样的排序结果，而dense_rank()会产生1、2、2、3、4……这样的排序，区别在于同分数的人会不会把下面的序号给用掉。

当然了，开窗函数也能有其他的用法，比如分组汇总功能
```sql
SELECT user_id, SUM(user_score) over (partition by user_group_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  AS total FROM user_score ORDER BY user_group_id
```

这是在干嘛？这个是在计算前面的格子到现在目前行的总数值，相当于一个分组小计