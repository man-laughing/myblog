
---
title: 实践-MySQL5.6一致性锁定读
date: 2019/11/20
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.3  |3.10.0   |5.6.43        |
    
## 先决条件
* 请确保你对MYSQL隔离级别的 Repeatable-Read 有一定理解
* 请确保你对“幻读”现象有一定的理解
* 请确保你对排他锁、共享锁有一定的理解
  
## 什么是一致性锁定读？
一致性锁定读，通常认为在某些场景下，用户希望显式的对读取操作进行加锁以保证数据逻辑一致性。
 
## 一致性锁定读的表现形式？
* 排他锁（X锁）--- FOR  UPDATE
* 共享锁（S锁）--- LOCK IN SHARE MODE

## 一致性锁定读的作用？
* 解决了“幻读”问题
* 保证了数据的一致性

## 实际案例 | （“幻读”）

* 准备隔离级别和测试的表

```bash
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql>
mysql> select * from zl;
Empty set (0.00 sec)

mysql>
mysql> show index from zl;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| zl    |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

mysql>
mysql> desc zl;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id    | int(11) | NO   | PRI | NULL    |       |
+-------+---------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务1

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
Empty set (0.00 sec)

mysql>
```

* 打开第二个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务2
 
```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
Empty set (0.00 sec)

mysql>
``` 
 
* 然后打开第一个连接的MYSQL交互窗口，执行插入操作

```bash
mysql> insert into zl select 111;   //插入数据
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0

mysql> select * from zl;        //插入后查询数据
+-----+
| id  |
+-----+
| 111 |
+-----+
1 row in set (0.00 sec)

mysql> commit;    //提交事务
Query OK, 0 rows affected (0.01 sec)

mysql>
```

* 然后打开第二个连接的MYSQL交互窗口，执行插入操作

```bash
mysql>
mysql>  insert into zl select 111;    //插入数据
ERROR 1062 (23000): Duplicate entry '111' for key 'PRIMARY'   //这里提示报错了，这就是“幻读"现象,因为之前的查询显示什么数据都没有，但是实际插入时却报错有重复数据。。
mysql>
mysql> commit;  //提交事务
Query OK, 0 rows affected (0.00 sec)
```



## 实际案例 | 一致性锁定读（LOCK IN SHARE MODE）

* 准备隔离级别和测试的表

```bash
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.01 sec)

mysql> select * from zl;
+-----+
| id  |
+-----+
| 122 |
+-----+
1 row in set (0.00 sec)

mysql>
mysql> show index from zl;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| zl    |          0 | PRIMARY  |            1 | id          | A         |           1 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

mysql> desc zl;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id    | int(11) | NO   | PRI | NULL    |       |
+-------+---------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql>

```

* 打开第一个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务1

```bash
mysql> begin;  //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;    //查询数据
+-----+
| id  |
+-----+
| 122 |
+-----+
1 row in set (0.00 sec)

mysql>

```

* 打开第二个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务2

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> select * from zl;    //查询数据
+-----+
| id  |
+-----+
| 122 |
+-----+
1 row in set (0.00 sec)

mysql>

```


* 然后打开第一个连接的MYSQL交互窗口，在事务1中执行修改操作，不要提交

```bash
mysql>
mysql> update zl set id = 133 where id = 122;  //修改数据，会触发X锁发生
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from zl;   //查询数据
+-----+
| id  |
+-----+
| 133 |
+-----+
1 row in set (0.00 sec)

mysql>
```

* 然后打开第二个连接的MYSQL交互窗口，在事务2中执行查询且带锁定的操作（一致性锁定读）

```bash
mysql>
mysql> select * from zl lock in share mode;   //携带S锁的查询操作，可以看到卡住了。。这是因为事务1还没有提交，这里事务2的S锁和事务1的X锁不兼容，锁冲突了。


```

* 然后打开第一个连接的MYSQL交互窗口，提交事务1

```bash
mysql>
mysql> commit;   //提交事务
Query OK, 0 rows affected (0.02 sec)

mysql> select * from zl;
+-----+
| id  |
+-----+
| 133 |
+-----+
1 row in set (0.00 sec)

```
 
* 然后打开第二个连接的MYSQL交互窗口，在事务2中查看之前带S锁定的读取结果

```bash
 mysql> select * from zl lock in share mode;   //可以看到读取到了最新的结果，这是一致性锁定读的功劳，保证了数据逻辑的一致性且在某些场景会很有用（正常读取应该读到事务开始时的数据快照）
+-----+
| id  |
+-----+
| 133 |
+-----+
1 row in set (9.66 sec)

mysql>

```

## 实际案例 | 一致性锁定读（FOR UPDATE）-> 解决”幻读“

* 准备隔离级别和测试的表

```bash
mysql>
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.01 sec)

mysql> select * from zl;
+-----+
| id  |
+-----+
| 150 |
+-----+
1 row in set (0.00 sec)

mysql> show index from zl;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| zl    |          0 | PRIMARY  |            1 | id          | A         |           1 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

mysql> desc zl;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id    | int(11) | NO   | PRI | NULL    |       |
+-------+---------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务1

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;     //查询数据
+-----+  
| id  |
+-----+
| 150 |
+-----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个SSH连接，进入MYSQL交互窗口，执行事务，这里称之为事务2

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;     //查询数据
+-----+  
| id  |
+-----+
| 150 |
+-----+
1 row in set (0.00 sec)

mysql>
```

* 然后打开第一个连接的MYSQL交互窗口，在事务1中插入数据，不要提交。

```bash
mysql> insert into zl select 160 ;   //插入数据
Query OK, 1 row affected (0.00 sec)
Records: 1  Duplicates: 0  Warnings: 0

mysql> select * from zl;     //查询数据
+-----+
| id  |
+-----+
| 150 |
| 160 |
+-----+
2 rows in set (0.01 sec)

mysql>
```

* 然后打开第二个连接的MYSQL交互窗口，在事务2中查询数据伴随FOR UPDATE，不要提交。

```bash
mysql>
mysql> select * from zl where id = 160;   //普通查询
Empty set (0.00 sec)

mysql> select * from zl where id = 160 for update;    //可以看到卡住了，因为事务1未提交，当前事务2 for update 会触发X锁，事务2的X锁和事务1的X锁不兼容，所以只能等待了。。



```

* 然后打开第一个连接的MYSQL交互窗口，提交事务1

```bash
mysql> commit;   //提交事务
Query OK, 0 rows affected (0.02 sec)

mysql> select * from zl;   //查询最新的数据
+-----+
| id  |
+-----+
| 150 |
| 160 |
+-----+
2 rows in set (0.00 sec)

```

* 然后打开第二个连接的MYSQL交互窗口，查看之前的结果

```bash
mysql> select * from zl where id = 160 for update;  //可以看到这个最新结果出来了，表示有其他事务（这里指事务1）已经插入了160这个值，当前事务2是无法插入且提交的。
+-----+
| id  |
+-----+
| 160 |
+-----+
1 row in set (13.05 sec)

mysql> 
mysql> select * from zl;  //查看数据，可以看到在RR级别下默认还是看到事务开始时的快照数据
+-----+
| id  |
+-----+
| 150 |
+-----+
1 row in set (0.00 sec)

mysql> insert into zl select 160;   //插入数据160报错，，其实根据以上的查询FOR UPDATE我们就可以得到了最新的结果，由此得知是不能插入160这个值的，从而避免了“幻读”问题的发生。
ERROR 1062 (23000): Duplicate entry '160' for key 'PRIMARY'
mysql>

mysql> commit;  //提交
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;  //查询数据
+-----+
| id  |
+-----+
| 150 |
| 160 |
+-----+
2 rows in set (0.00 sec)

mysql>
```