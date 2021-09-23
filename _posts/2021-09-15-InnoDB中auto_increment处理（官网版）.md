---
title: MySQL 5.7 - InnoDB 中的 AUTO_INCREMENT 处理 (官网版)
author: BiChengfei
date: 2021-09-15
categories: [MySQL]
tags: [MySQL, InnoDB]
---



《 MySQL 5.7 - InnoDB 中的 AUTO_INCREMENT 处理（个人版）》地址：

官网地址 [https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)



InnoDB 提供了一种可配置的锁机制，通过为新插入的行增加 auto_increment 列，可以显著提高 SQL 语句的可伸缩性和性能。要在 InnoDB 中使用 auto_increment 机制，auto_increment 列必须定义为索引的第一列或者唯一列，以便可以对 table 执行等价于对索引列 ```select max(id) ```以获取最大列值，这个索引不需要必须是 ```primary key``` 或者 ```unique```，但为了避免在```auto_increment```列中出现重复值，推荐使用这些索引类型。

### auto_increment 三种锁定模式

```auto_increment``` 的模式是通过 ```MySQL``` 启动时配置的```innodb_autoinc_lock_mode``` 参数配置的，接下来将描述```auto_increment```如何生成自动递增值，以及每种模式对复制的影响

以下术语用来描述 ```innodb_autoinc_lock_mode``` 设置：

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

- 基于语句的复制

  源和副本，执行相同的语句

- 基于行的复制

  副本直接把源上的数据，通过 insert 一条一条复制过来

- 混合格式的复制

  官网解释：混合格式使用基于行的复制（不知道什么玩意，不过相信官网）

```innodb_autoinc_lock_mode``` 有三种取值，分别是 0、1、2，分别代表“传统“、”连续“、”交错“锁定模式

1. ```innodb_autoinc_lock_mode = 0```  

   在 ```innodb_autoinc_lock_mode```属性引入之前，就是传统锁定模式。提供锁定模式，只是为了向后兼容、新能测试和解决“混合插入”问题。

   在这种模式下，所有的插入语句都会获得一个特殊的表级 AUTO-INC 锁，用于插入到带有```auto_increment```列的表中。此锁通常保持到语句的末尾（而不是事务的末尾），以确保对于给定的插入语句顺序，自增值是可预测的和可重复执行的，并确保自增值由任何给定语句赋值都是连续的。

   接下来通过一个小 Demo 进行讲解：

   ```sql
   create table t1 (
       c1 int(11) not null auto_increment,	
       c2 varchar(10) default null,
       primary key (c1)
   ) engine = InnoDB;
   ```

   假设有两个在运行，一个插入 1000 行，一个插入 1 行

   ```sql
   tx1: insert into t1(c2) select c2 from t1_bak limit 1000;
   tx2: insert into t1(c2) values('aaa');
   ```

   tx1 生成的 c1 列值是连续的，tx2 生成的 c1 列值小于或大于所有用于 tx1 的语句生成的 c1 值，具体取决于两条语句谁先执行。

   在基于语句的复制，或在恢复场景中，SQL 从二进制日志重放时，只要以相同的顺序执行，结果就会与 tx1 和 tx2 首次运行时的结果相同。基于语句的复制的情况下，表锁会一直持续到本条插入语句结束，这样确保了自增值的安全性，但是，当多个事务同时执行插入时，表锁将会限制并发性和可伸缩性。如果在并发的情况下， tx1 与 tx2 不以与首次相同的顺序执行，结果具有不确定性，可能与第一次执行的结果不同。

   在上看的例子中，如果没有表锁，则 tx1 和 tx2 分配的自增值觉有不确定性，因运行而异。

   简单插入、批量插入、混合插入都是通过表锁

2. ```innodb_autoinc_lock_mode = 1```

   5.7 中默认的锁定模式

   对于简单插入，预先知道要插入的行数，通过互斥锁，获取所需数量的自增值，不直到语句的结束，避免表锁性能的消耗。

   对于批量插入，不知道要插入的行数。如果批量读取数据的源表与目标表不同，则先取源表上第一行的共享锁，再取目标表上的表锁，如果源表与目标表相同，则对所有选定行取共享锁后，才会取表锁。

   对于混合插入，知道要插入的行数，但并不是所有行都需要自增值，这时 InnoDB 会分配比插入行数更多的自增值（自增值范围官网只描述了左区间，前一条语句的自增值 + 1），语句结束后，未使用的自增值会丢弃，所以自增值会存在间隙。

3. ```innodb_autoinc_lock_mode = 2```

   在这种模式下，任何插入语句都不会用到表锁，可以同时执行多条语句，这是最快和最具扩展性的锁模式，但在使用基于语句的复制或从 binlog 复制的恢复场景中，它是不安全的。这种模式下自增值是唯一且单调递增的，但在多个语句并发执行时，同一语句插入的行的自增值可能是不连续的。

   对于简单插入，知道插入条数，自增值连续

   对于批量插入，任何给定语句分配的自动增量值中可能存在间隙(这是官网语句，我没很理解)

   对于混合插入，自增值和```innodb_auto_lock_mode = 1```的处理差不错（官网没写，这是个人理解）



### auto_increment 使用指南（FAQ）

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

   0、1、2 锁定模式下，对于自增值的处理都不同（这部分很少接触，我懒得翻译了，但官网写的很详细，自己看吧）

8. 在多个插入语句中间修改自增值

   5.7 版本下，在所有模式（0、1、2）中，在插入语句序列中间修改自增值可能造成 “Duplicate entry" 错误，8.0 版本不会报错。例如，如果你执行 update 语句去修改自增值，使其变成一个大于当前自增值最大值的数，随后执行一条未显示指定自增值的插入语句，可能造成“Duplicate entry"错误。下述例子将会论证这种操作：

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
   

### auto_increment 初始化

本节描述 InnoDB 如何初始化 auto_increment 计数器

如果你为 InnoDB 的表指定一个 auto_increment 列，则在 InnoDB  的数据字典中，该表持有一个称之为 auto_increment 计数器的特殊计数器，用于为自增列指定新值。这个计数器只存放在主内存中，不存储到磁盘。

在 MySQL 重启后，针对 auto_increment 计数器的初始化，当第一次执行包含自增列的插入语句时，InnoDB 执行等同于以下的语句

```sql
select max(col) from table_name for update;
```

通过这个语句，InnoDB 获取到增量值，并把它分配给列和这张表的 auto_increment 计数器。默认情况下，该值递增 1，这个默认值可以被 auto_increment_increment 配置覆盖。

如果是空表，InnoDB 将使用 1，这个默认配置可以被 auto_increment_offset 配置覆盖。

如果表在 auto_increment 计数器初始化之前，用 show table status 来检查，则 InnoDB 会初始化 auto_increment 计数器，但不递增，供后续的插入使用。此初始化使用对表的普通排它锁读取，并且锁定持续到事务结束。InnoDB 遵循相同的流程，对新表的atuo_increment 计数器进行初始化。

在 auto_increment 计数器初始化后，对于要插入的数据，如果你没有显示指定 auto_increment 列的值，计数器将递增并把新值赋给 auto_increment 列。如果你显示的置顶 auto_increment 列的值，并且这个值大于计数器的值，这个计数器会被设置为这个显示指定的值。

`InnoDB`只要服务器运行，就使用内存中自动递增计数器。当服务器停止并重新启动时，`InnoDB`为第一个表重新初始化每个表的计数器， 如前所述。

针对 InnoDB 的表，我们可以使用 auto_increment = N 去设置 auto_increment 的初始值或者去修改计数器的值，但 MySQL 服务的重启会消除 auto_increment  = N 造成的效果。

### 注意事项

- 当 auto_increment 用完值时，默认情况下，后续的插入操作将返回 ```duplicate key``` 错误
- 当您重新启动 MySQL 服务器时，`InnoDB` 可能会重用为`AUTO_INCREMENT`列生成但从未存储的旧值 （即，在回滚的旧事务期间生成的值）
