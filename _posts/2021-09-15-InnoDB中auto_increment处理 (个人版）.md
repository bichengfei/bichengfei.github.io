---
title: MySQL 5.7 - InnoDB 中的 AUTO_INCREMENT 处理（个人版）
author: BiChengfei
date: 2021-09-15
categories: [MySQL]
tags: [MySQL, InnoDB]
---

《 MySQL 5.7 - InnoDB 中的 AUTO_INCREMENT 处理（官网版）》地址：

MySQL 官网地址： [https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)

### auto_increment 三种锁定模式

InnoDB 对于 column，提供了 auto_increment （自增）配置，因为性能和扩展性的考虑，自增值有三种不同的模式，通过 MySQL 启动时的 ```innodb_autoinc_lock_mode``` 属性指定，该属性有三种取值，分别是0、1、2。

可通过以下语句查看：

```sql
show variables where Variable_name = 'innodb_autoinc_lock_mode';
```

- ```innodb_autoinc_lock_mode = 0```，对于所有插入语句，插入 SQL 语句开始处加互斥表锁，语句结束处释放互斥表锁， 按照 SQL 的先后顺序单线程执行，只要顺序相同，自增值就会相同，数据就相同，插入语句的单线程版本
- ```innodb_autoinc_lock_mode = 1```，对于简单插入，先通过互斥表锁获取所需数量的自增值，再慢慢插入，对于批量插入，同```innodb_autoinc_lock_mode = 0```策略，对于混合插入，先通过互斥表锁获取比插入行数更多的自增值，没用完的丢弃，自增值会产生间隙。等同于把插入语句分成两步执行，第一步，通过自增值互斥锁获取自增值，第二步，结合上步的自增值写入数据库，所以多个并行插入语句，同一个语句内的自增值是连续的，自增值的单线程版本
- ```innodb_autoinc_lock_mode = 2```，可并行执行多个插入语句，并行时，同一个插入语句的自增值可能不是连续的，先批量插入，再普通插入，也不连续，自增值的多线程版本

0 和 1 的版本，通过 binlog 同步数据，只要 SQL 顺序相同，数据就相同，2 的版本，SQL 顺序相同，自增值也可能不同

```
简单比较：
性能：0 < 1 < 2
扩展：2 < 0 = 1
性能 + 扩展：2 < 0 < 1
```

术语说明：

- 插入

  在表中生成新行的语句，包括```insert```, ```insert ... select ...```, ```replace```, ```replace ... select ...```, ```load data```，通过插入特征，可以分为简单插入、批量插入、混合插入

- 简单插入

  可以预先确定要插入行数的语句（在最初处理语句时），这包括没有嵌套子查询的单行和多行```insert```和```replace```，但不包括```insert ... on duplicate key update```.

- 批量插入

  预先不能确定要插入的行数的语句（以及所需的自动增量值的数量），这包括```insert ... select```, ```replace ... select``` 和 ```load data```

- 混合插入

  混合插入类似于简单插入，只是对于新增的数据，只会为部分制定自动增量值，并不是全部。类似于下列语句：  

  ```sql
  insert into t1(c1, c2) values(1, 'a'), (null, 'b'), (5, 'c'), (null, 'd');
  
  insert ... on duplicate key update;
  ```


<B>个人验证步骤</B>

我主要验证了并行批量插入下，同一个 SQL 中生成的自增值是否连续的问题，以及同样两条 SQL 在同样顺序下执行，自增列的分布情况

1. 源表 test  表生成 30 万条数据，再创建目标表 test01。（test 表与 test01 表结构完全相同）

   ```sql
   -- 生成测试表结构
   CREATE TABLE `test`
   (
       `id`  int NOT NULL AUTO_INCREMENT,
       `str` varchar(200) DEFAULT NULL,
       PRIMARY KEY (`id`),
       FULLTEXT KEY `fulltextIndex` (`str`)
   ) ENGINE = InnoDB AUTO_INCREMENT = 1;
   
   CREATE TABLE `test01`
   (
       `id`  int NOT NULL AUTO_INCREMENT,
       `str` varchar(200) DEFAULT NULL,
       PRIMARY KEY (`id`),
       FULLTEXT KEY `fulltextIndex` (`str`)
   ) ENGINE = InnoDB AUTO_INCREMENT = 1;
   
   -- 借用存储过程生成 30 万条数据
   delimiter //
   drop procedure if exists test;
   create procedure test()
   begin
       declare i int;
       set i = 0;
       truncate test;
       truncate test01;
       while i < 300000 do
           set i = i + 1;
           insert into test.test(str) values (concat(i, ''));
       end while;
   end //
   
   call test();
   delimiter ;	
   ```

2. 设置 ```innodb_autoinc_lock_mode = 1```，重启 MySQL

   ```shell
   root@05ccb2a57ee4:/# vim ./etc/mysql/my.cnf
   
   innodb_autoinc_lock_mode = 1
   ```

3. 查看设置是否生效，```show variables where Variable_name = 'innodb_autoinc_lock_mode';```

4. 清空 test01，```truncate test01```

5. 打开两个客户端窗口，本别执行

   ```sql
   客户端窗口 1: insert into test01(str) select concat(str, 'b') from test;
   客户端窗口 2: insert into test01(str) select concat(str, 'c') from test;
   ```

6. 查看结果

   ```sql
   mysql> select 'b', min(id), max(id) from test01 where str like '%b'
       -> union
       -> select 'c', min(id), max(id) from test01 where str like '%c';
   +---+---------+---------+
   | b | min(id) | max(id) |
   +---+---------+---------+
   | b |       1 |  300000 |
   | c |  327676 |  627675 |
   +---+---------+---------+
   2 rows in set (0.57 sec)
   ```

7. 设置 ```innodb_autoinc_lock_mode = 2```，重启 MySQL

   ```shell
   root@05ccb2a57ee4:/# vim ./etc/mysql/my.cnf
   
   innodb_autoinc_lock_mode = 2
   ```

8. 查看设置是否生效，```show variables where Variable_name = 'innodb_autoinc_lock_mode';```

9. 选择数据库，打开两个客户端窗口，本别执行

   ```sql
   客户端窗口 1: insert into test01(str) select concat(str, 'b') from test;
   客户端窗口 2: insert into test01(str) select concat(str, 'c') from test;
   ```

10. 查看生成数据

    ```sql
    mysql> select 'b', min(id), max(id) from test01 where str like '%b'
        -> union
        -> select 'c', min(id), max(id) from test01 where str like '%c';
    +---+---------+---------+
    | b | min(id) | max(id) |
    +---+---------+---------+
    | b |       1 |  431070 |
    | c |  196606 |  627675 |
    +---+---------+---------+
    2 rows in set (0.57 sec)
    ```

11. 清空 test01，```truncate test01```

12. 打开两个客户端窗口，本别执行

    ```sql
    客户端窗口 1: insert into test01(str) select concat(str, 'b') from test;
    客户端窗口 2: insert into test01(str) select concat(str, 'c') from test;
    ```

13. 查看生成数据

    ```sql
    mysql> select 'b', min(id), max(id) from test01 where str like '%b'
        -> union
        -> select 'c', min(id), max(id) from test01 where str like '%c';
    +---+---------+---------+
    | b | min(id) | max(id) |
    +---+---------+---------+
    | b |       1 |  496605 |
    | c |  131071 |  627675 |
    +---+---------+---------+
    2 rows in set (0.58 sec)
    ```

### auto_increment 初始化

本节描述 InnoDB 如何初始化 auto_increment 计数器

如果你为 InnoDB 的表指定一个 auto_increment 列，则在 InnoDB  的数据字典中，该表持有一个称之为 auto_increment 计数器的特殊计数器，用于为自增列指定新值，这个计数器只存放在主内存中，我们可以使用 auto_increment = N 去设置 auto_increment 的初始值或者去修改计数器的值。

在 MySQL 重启后，针对 auto_increment 计数器的初始化，我们可以分成两种来考虑

- 空表

  默认情况下，auto_increment 计数器 = 1，这个默认值可以被 auto_increment_offset 配置覆盖

  注意：这个地方会忽略建表时或修改表时的 auto_increment = N 配置，因为这个配置放在内存中，会丢失

- 非空表

  auto_increment 初始化，可以分来懒加载和预加载

  - 懒加载

    当第一次执行包含自增列的插入语句时，InnoDB 执行等同于以下的语句

    ```sql
    select max(col) from table_name for update;
    ```

    通过这个语句，InnoDB 获取到增量值，并把它分配给列和这张表的 auto_increment 计数器。默认情况下，该值递增 1，这个默认值可以被 auto_increment_increment 配置覆盖。

  - 预加载

    用 show table status 来检查，则 InnoDB 会初始化 auto_increment 计数器，但不递增，供后续的插入使用。此初始化使用对表的普通排它锁读取，并且锁定持续到事务结束。InnoDB 遵循相同的流程，对新表的atuo_increment 计数器进行初始化。

在 auto_increment 计数器初始化后，对于要插入的数据，如果你没有显示指定 auto_increment 列的值，计数器将递增并把新值赋给 auto_increment 列。如果你显示的置顶 auto_increment 列的值，并且这个值大于计数器的值，这个计数器会被设置为这个显示指定的值。

### FAQ

1. 在同步中使用 auto_increment

   如果使用基于语句的复制，比如 binlog，请设置```innodb_autoinc_lock_mode = 0 或 1```，并在源和副本上使用相同的值，如果使用```innodb_autoinc_lock_mode = 2```或源和副本使用不同的配置，则不能保证源和副本上自增量相同

   如果使用基于行的复制，则 0、1、2 模式都是安全的

   如果使用混合格式的复制，因为混合模式使用基于行的模式，所以0、1、2 模式都是安全的

2. 自增值的“丢失”和间隙

   在0、1、2 锁定模式中，自增值一旦生成，就不能回滚，无论语句是否完成，自增值都不会重用，这些自增值将“丢失”，即使  MySQL 重启，也会丢失。可以通过 insert 手动指定使用

   ```sql
   mysql> truncate test01;
   Query OK, 0 rows affected (0.35 sec)
   
   mysql> begin;
   Query OK, 0 rows affected (0.00 sec)
   
   mysql> insert into test01(str) select str from test limit 100;
   Query OK, 100 rows affected (0.00 sec)
   Records: 100  Duplicates: 0  Warnings: 0
   
   mysql> rollback ;
   Query OK, 0 rows affected (0.01 sec)
   
   mysql> select * from test01;
   Empty set (0.00 sec)
   
   mysql> insert into test01(str) values ('a');
   Query OK, 1 row affected (0.02 sec)
   
   mysql> select * from test01;
   +-----+------+
   | id  | str  |
   +-----+------+
   | 128 | a    |
   +-----+------+
   1 row in set (0.00 sec)
   
   mysql> insert into test01(id, str) values (5, 'a');
   Query OK, 1 row affected (0.01 sec)
   
   mysql> select * from test01;
   +-----+------+
   | id  | str  |
   +-----+------+
   |   5 | a    |
   | 128 | a    |
   +-----+------+
   2 rows in set (0.00 sec)
   
   mysql> 
   ```

3. 指定 auto_increment 列为 null 或 0

   所有的锁定模式（0、1、2）中，InnoDB 视为没有没有指定值，并为其生成新值

4. 指定 auto_increment 列值为负数

   官网翻译：在所有锁定模式（0、1 和 2）中，如果为`AUTO_INCREMENT` 列分配负值，则自动递增机制的行为是未定义的（没看懂）

   个人实操：所有的锁定模式（0、1、2）中，将把负数设置为列值

   ```sql
   -- innodb_auto_lock_mode = 0
   mysql> insert into test01(id,str) values (-3, 'hahahaha');
   Query OK, 1 row affected (0.01 sec)
   
   mysql> select * from test01 where id = -3;
   +----+----------+
   | id | str      |
   +----+----------+
   | -3 | hahahaha |
   +----+----------+
   1 row in set (0.00 sec)
   
   -- innodb_auto_lock_mode = 1
   mysql> insert into test01(id,str) values (-4, 'hahahaha');
   Query OK, 1 row affected (0.01 sec)
   
   mysql> select * from test01 where id = -4;
   +----+----------+
   | id | str      |
   +----+----------+
   | -4 | hahahaha |
   +----+----------+
   1 row in set (0.00 sec)
   
   -- innodb_auto_lock_mode = 2
   mysql> insert into test01(id,str) values (-5, 'hahahaha');
   Query OK, 1 row affected (0.01 sec)
   
   mysql> select * from test01 where id = -5;
   +----+----------+
   | id | str      |
   +----+----------+
   | -5 | hahahaha |
   +----+----------+
   1 row in set (0.00 sec)
   ```

5. 指定 auto_increment 列值大于该列类型的最大值

   官网翻译：在所有锁定模式（0、1 和 2）中，如果值变得大于可以存储在指定整数类型中的最大整数，则自动递增机制的行为是未定义的（没看懂）

   个人实操：只试了 ```innodb_autoinc_lock_mode = 2```，报错 

   ```sql
   mysql> insert into test01(id,str) values (2147483648, 'hahahaha');
   ERROR 1264 (22003): Out of range value for column 'id' at row 1
   mysql> 
   ```

6. 批量插入，自增值间隙

   只有```innodb_autoinc_lock_mode = 2```模式，并且并行批量插入，才可能发生间隙

7. 由混合插入分配的自增值

   0、1、2 锁定模式下，对于自增值的处理都不同（这部分很少接触，我懒得翻译了，但官网写的很详细）

8. 在多个插入语句中间修改自增值

   1. 5.7 版本下，在所有模式（0、1、2）中，在插入语句序列中间修改自增值可能造成 “Duplicate entry" 错误，8.0 版本不会报错。例如，如果你执行 update 语句去修改自增值，使其变成一个大于当前自增值最大值的数，随后执行一条未显示指定自增值的插入语句，可能造成“Duplicate entry"错误。下述例子将会论证这种操作：

      ```sql
      mysql> CREATE TABLE t1 (
          -> c1 INT NOT NULL AUTO_INCREMENT,
          -> PRIMARY KEY (c1)
          ->  ) ENGINE = InnoDB;
      
      mysql> INSERT INTO t1 VALUES(0), (0), (3);
      
      mysql> SELECT c1 FROM t1;
      +----+
      | c1 |
      +----+
      |  1 |
      |  2 |
      |  3 |
      +----+
      
      mysql> UPDATE t1 SET c1 = 4 WHERE c1 = 1;
      
      mysql> SELECT c1 FROM t1;
      +----+
      | c1 |
      +----+
      |  2 |
      |  3 |
      |  4 |
      +----+
      
      mysql> INSERT INTO t1 VALUES(0);
      ERROR 1062 (23000): Duplicate entry '4' for key 'PRIMARY'
      ```
   
9. 当 auto_increment 用完值时，会怎么处理

   默认情况下，后续的插入操作将返回 ```duplicate key``` 错误



参考：

1. ```SELECT * FROM information_schema.TABLES WHERE Table_Schema='wbs244' AND table_name = 't1' ;```
2. ```show table status where Name like '%t1%';```
