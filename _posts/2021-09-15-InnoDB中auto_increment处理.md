---
title: MySQL 5.7 - InnoDB 中的 AUTO_INCREMENT 处理
author: BiChengfei
date: 2021-09-15
categories: [MySQL]
tags: [MySQL, InnoDB]
---



官网地址 [https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)



InnoDB 提供了一种可配置的锁机制，通过为新插入的行增加 auto_increment 列，可以显著提高 SQL 语句的可伸缩性和性能。要在 InnoDB 中使用 auto_increment 机制，auto_increment 列必须定义为索引的第一列或者唯一列，以便可以对 table 执行等价于对索引列 ```select max(id) ```以获取最大列值，这个索引不需要必须是 ```primary key``` 或者 ```unique```，但为了避免在```auto_increment```列中出现重复值，推荐使用这些索引类型。

### auto_increment 三种取值



### auto_increment 使用意义



### auto_increment 初始化



### 注意事项
