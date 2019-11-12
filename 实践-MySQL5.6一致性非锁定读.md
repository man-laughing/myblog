
---
title: 实践-MySQL5.6一致性非锁定读
date: 2019/11/12
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.3  |3.10.0   |5.6.43        |
    
## 先决条件
* 请确保你对MYSQL隔离级别有一定理解
* 请确保你对“幻读”现象有一定理解
  
## 什么是一致性非锁定读？
一致性非锁定读（Consistent Nonlocking Read）是指InnoDB存储引擎通过行多版本控制的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反的，InnoDB存储引会去读取行的一个快照。 ---《MYSQL技术内幕InnoDB存储引擎 第2版》

## 什么是多版本控制？
我自己的理解是表中的一行数据（Record）在不同的时间节点对应着不同的状态（数据）

## 一致性非锁定读的作用？
解决了“幻读”问题

## 实际案例

* 准备好隔离级别设定以及测试的表。

```
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql>
mysql> use zltest
Database changed
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
mysql> select * from zl;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个终端窗口，进入MYSQL并且启动事务，这里称为事务1

```bash
mysql> begin;   //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;   //查询zl表的数据
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个终端窗口，进入MYSQL并且启动事务，这里称为事务2

```bash
mysql> begin;   //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;   //查询zl表的数据
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 现在重新打开第一个窗口,在事务1中继续操作

```bash
mysql> update zl set id = 6 where id = 1;   //在事务中修改数据
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
 
mysql> select * from zl;    //修改后查看数据
+----+
| id |
+----+
|  6 |
+----+
1 row in set (0.00 sec)

mysql>
mysql> commit;           //提交
Query OK, 0 rows affected (0.02 sec)

mysql>
```

* 现在重新打开第二个窗口,在事务2中继续操作

```bash
mysql>
mysql> select * from zl for update; //for update表示使用一致性非锁定读，可以看到读取到了事务1提交后的数据(也就是行数据的快照)
+----+
| id |
+----+
|  6 |
+----+
1 row in set (0.00 sec)

mysql>
mysql> commit;           //提交
Query OK, 0 rows affected (0.02 sec)

mysql>
```