
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
* 请确保你对MYSQL隔离级别的 Read-Committed 有一定理解
* 请确保你对MYSQL隔离级别的 Repeatable-Read 有一定理解
  
## 什么是一致性非锁定读？
一致性非锁定读（Consistent Nonlocking Read）是指InnoDB存储引擎通过行多版本控制的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反的，InnoDB存储引会去读取行的一个快照。 ---《MYSQL技术内幕InnoDB存储引擎 第2版》

## 什么是多版本控制？
我自己的理解是表中的一行数据（Record）在不同的时间节点对应着不同的状态（数据）

## 一致性非锁定读的作用？
* 提高了事务并发性？ 
* 间接增强了事务隔离性？

## 实际案例（隔离级别Read-Committed）

* 准备好隔离级别设定以及测试的表。

```bash
mysql> show variables like '%iso%';
+---------------+----------------+
| Variable_name | Value          |
+---------------+----------------+
| tx_isolation  | READ-COMMITTED |
+---------------+----------------+
1 row in set (0.00 sec)

mysql>
mysql>
mysql> select * from zl;         //测试的表
+----+
| id |
+----+
| 10 |
+----+
1 row in set (0.00 sec)

mysql> show index from zl;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| zl    |          0 | PRIMARY  |            1 | id          | A         |           1 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

mysql>
mysql> desc zl;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id    | int(11) | NO   | PRI | NULL    |       |
+-------+---------+------+-----+---------+-------+
1 row in set (0.01 sec)

mysql>
```

* 打开第一个窗口进入MYSQL交互程序，执行事务，这里称之为事务1

```bash
mysql> begin;     //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;   //查看数据
+----+
| id |
+----+
| 10 |
+----+
1 row in set (0.00 sec)

mysql>

```

* 打开第二个窗口进入MYSQL交互程序，执行事务，这里称之为事务2

```bash
mysql> begin;     //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;   //查看数据
+----+
| id |
+----+
| 10 |
+----+
1 row in set (0.00 sec)

mysql>

```

* 打开第一个窗口进入MYSQL交互程序，在事务1中修改数据，注意不要提交

```bash
mysql> update zl set id = 11 where id = 10;   //修改数据
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from zl;     //查询数据
+----+
| id |
+----+
| 11 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个窗口进入MYSQL交互程序，查询数据

```bash
mysql>
mysql> select * from zl;  //可以看到这里查到的数据还是事务开始执行的状态（个人理解这里就已经开始执行“一致性非锁定读”了，只不是事务1一直没提交，只能看到事务开始时的快照数据）
+----+
| id |
+----+
| 10 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个窗口进入MYSQL交互程序，提交事务1

```bash
mysql> commit;      //提交
Query OK, 0 rows affected (0.01 sec)

mysql> select * from zl;   //查询
+----+
| id |
+----+
| 11 |
+----+
1 row in set (0.00 sec)

mysql>

```

* 打开第二个窗口进入MYSQL交互程序，查询数据

```bash
mysql>
mysql> select * from zl;  //查询数据，可以看到随着事务1的提交，事务2这里立刻就可以看到最新的数据。这就是“一致性非锁定读”在这个级别Read-Committed下的体现，MYSQL总是尝试获取最新的快照数据。
+----+
| id |
+----+
| 11 |
+----+
1 row in set (0.00 sec)

mysql>
```



## 实际案例（隔离级别Repeatable-Read）

* 准备好隔离级别设定以及测试的表。

```bash
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql> select * from zl;
+----+
| id |
+----+
|  1 |
+----+
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

* 打开第一个窗口进入MYSQL交互程序，执行事务，这里称之为事务1

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个窗口进入MYSQL交互程序，执行事务，这里称之为事务2

```bash
mysql> begin;    //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个窗口进入MYSQL交互程序，在事务1中修改数据，注意不要提交

```bash
mysql> update zl set id = 6 where id = 1;        //修改数据
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from zl;       //查询修改后的数据
+----+
| id |
+----+
|  6 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个窗口进入MYSQL交互程序，在事务2中查看数据

```bash
mysql> select * from zl;   //可以看到这里读到的数据是事务2开始时zl表的状态，这就是MYSQL利用了“一致性非锁定读”的特性，读取到了一个行的历史快照且没有任何的锁开销，我的理解是这么做一定程度上是为了更好的并发性且充分满足了事务的隔离性。
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第一个窗口进入MYSQL交互程序，去提交事务1

```bash
mysql> commit;             //提交
Query OK, 0 rows affected (0.01 sec)

mysql> select * from zl;   //可以看到数据没有什么变化
+----+
| id |
+----+
|  6 |
+----+
1 row in set (0.00 sec)

mysql>
```

* 打开第二个窗口进入MYSQL交互程序，在事务2中新增数据且提交。

```bash
mysql> insert into zl  values (888);  //插入数据
Query OK, 1 row affected (0.00 sec)

mysql> commit;      //提交
Query OK, 0 rows affected (0.02 sec)

mysql> select * from zl;       //查询数据，可以发现事务1和事务2的最终结果都正确显示了，每个事务在执行过程中是没有任何关联关系的，这就是隔离性。
+-----+
| id  |
+-----+
|   6 |
| 888 |
+-----+
2 rows in set (0.00 sec)

mysql>
```

## 一致性非锁定读在Read-Committed和Repeatable-Read两种级别下的差异现象？

* Read-Committed 级别下，一致性非锁定读，读取到的数据是行的最新的快照。
* Repeatable-Read级别下，一致性非锁定度，读取到的数据是行的事务开始时的快照。