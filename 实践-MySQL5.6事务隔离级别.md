
---
title: 实践-MySQL5.6事务隔离级别
date: 2019/10/30
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.3  |3.10.0   |5.6.43        |
  
  

## 事务隔离级别

* 读未提交  Read Uncommitted
* 读提交    Read Committed
* 可重复读  Repeatable Read
* 串行化    Serializable


## 衍生的隔离问题

* 脏读
* 不可重复读
* 幻读


## 隔离级别关系图

| 隔离级别                  | 脏读      | 不可重复读  |  幻读     |
|:-------------------------|:---------|:----------|:----------|
|读未提交(Read Uncommitted) |有         |有         |有         |
|读提交(Read Committed)     |无         |有         |有         |
|可重复读(Repeatable Read)  |无         |无         |有         |
|串行化(Serializable)       |无         |无         |无         |


## 什么是脏读？

该隔离级别的事务会读到其它未提交事务的数据，此现象也称之为脏读

## 什么是不可重复读？

一个事务可以读取另一个已提交的事务，多次读取会造成不一样的结果，此现象称为不可重复读问题，

## 什么是幻读？
Select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。

## 实际案例（脏读）

* 基础配置，设置隔离级别等测试表，我这里已配置好

```bash
mysql> show variables like '%iso%';
+---------------+------------------+
| Variable_name | Value            |
+---------------+------------------+
| tx_isolation  | READ-UNCOMMITTED |
+---------------+------------------+
1 row in set (0.00 sec)

mysql> select * from zl;
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
+------+-------+
2 rows in set (0.00 sec)

mysql> show index from zl;
Empty set (0.00 sec)

mysql>
```

* 打开第1个窗口连接到mysql并执行事务，这里称为事务1

```bash
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
+------+-------+
2 rows in set (0.00 sec)

```

* 打开第2个窗口连接到mysql并执行事务，这里称为事务2

```bash
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
+------+-------+
2 rows in set (0.00 sec)

mysql>
```

* 在第1个窗口里继续执行以下，插入数据，注意不要提交～～

```bash
mysql> insert into zl  values (3,'hello');   //插入数据
Query OK, 1 row affected (0.00 sec)

mysql> select * from zl;                     //查询数据，记住先不要提交
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)

mysql>
```

* 在第2个窗口里继续执行以下，查询数据。

```bash
mysql> select * from zl;    //可以看到这（3，hello）这行数据显示了出来，但是我们没有对事务1进行提交，所以这个现象就是“脏读”，即“会读到其它未提交事务的数据”
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)

mysql>
```

## 实际案例（不可重复读）

* 基础配置，设置隔离级别等测试表，我这里已配置好

```bash
mysql> show variables like '%iso%';
+---------------+----------------+
| Variable_name | Value          |
+---------------+----------------+
| tx_isolation  | READ-COMMITTED |
+---------------+----------------+
1 row in set (0.00 sec)

mysql> select * from zl;
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)

mysql> show index from zl;
Empty set (0.00 sec)

mysql>
```

* 打开第1个窗口连接到mysql并执行事务，这里称为事务1

```bash
mysql> begin;        //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;      //查询数据
+------+-------+ 
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)
```

* 打开第2个窗口连接到mysql并执行事务，这里称为事务2

```bash
mysql> begin;        //声明事务开始
Query OK, 0 rows affected (0.00 sec)

mysql> select * from zl;      //查询数据
+------+-------+ 
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)
```

* 在第1个窗口中更新一行数据，注意先不要提交～

```bash
mysql> update zl set id = 333 where name = "hello";  //修改数据，注意先不要提交。
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from zl;     //可以看到刚修改的已经生效
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|  333 | hello |
+------+-------+
3 rows in set (0.00 sec)

mysql>
```

* 在第2个窗口中继续查询数据

```bash
mysql> select * from zl;    //可以看到数据还是和初始进入事务的状态一致
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|    3 | hello |
+------+-------+
3 rows in set (0.00 sec)

mysql>
```

* 在第1个窗口中提交事务

```bash
mysql> commit;
Query OK, 0 rows affected (0.02 sec)
```

* 在第2个窗口中继续查询数据

```bash
mysql> select * from zl;  //这里看到（333，hello）这行数据发生了改变，这种现象成为“不可重复读”，即“事务可以读取另一个已提交的事务，多次读取会造成不一样的结果”
+------+-------+   
| id   | name  |
+------+-------+
|    1 | admin |
|    2 | lisi  |
|  333 | hello |
+------+-------+
3 rows in set (0.00 sec)

```
## 实际案例（幻读）

* 基础配置，设置隔离级别等测试表，我这里已配置好

```bash
mysql> show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
1 row in set (0.00 sec)

mysql> select * from t1;
Empty set (0.00 sec)

mysql> show index from t1;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t1    |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

mysql> desc t1;
+-------+-------------+------+-----+---------+----------------+
| Field | Type        | Null | Key | Default | Extra          |
+-------+-------------+------+-----+---------+----------------+
| id    | int(11)     | NO   | PRI | NULL    | auto_increment |
| name  | varchar(32) | YES  |     | NULL    |                |
+-------+-------------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

mysql>

```

* 打开第1个窗口连接到mysql并执行事务，这里称为事务1

```bash
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1;     //可以看到没任何数据
Empty set (0.00 sec)

mysql>
```

* 打开第2个窗口连接到mysql并执行事务，这里称为事务2

```bash
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1;     //可以看到没任何数据
Empty set (0.00 sec)

mysql>
```

* 在第1个窗口连接到mysql，插入数据并且提交。

```bash
mysql> insert into t1 values (666,'admin');   //插入数据
Query OK, 1 row affected (0.00 sec)

mysql> select * from t1;            //查看数据
+-----+-------+
| id  | name  |
+-----+-------+
| 666 | admin |
+-----+-------+
1 row in set (0.00 sec)

mysql>
mysql> commit;            //提交
Query OK, 0 rows affected (0.01 sec)

mysql> select * from t1;       //提交后查看数据
+-----+-------+
| id  | name  |
+-----+-------+
| 666 | admin |
+-----+-------+
1 row in set (0.00 sec)

mysql>
```

* 在第2个窗口连接到mysql

```bash
mysql> select * from t1;   //可以看到返回的结果还是保持在事务开始时的状态
Empty set (0.00 sec)

mysql>
mysql> insert into t1 values (666,'admin');    //这里尝试插入和事务1一样的数据，你看这里报错了，说明这条数据已经存在了，但是我们之前查询却什么也没有，这个现象称之为“幻读”
ERROR 1062 (23000): Duplicate entry '666' for key 'PRIMARY'

小思考：这里事务2插入的数据和事务1一样，其实我认为是不奇怪的，事实上在实际的产线环境中，2个事务大概率是互相谁也不知道谁要插入什么样的数据。我这里之所以这么干主要是为了验证“幻读”这个现象，还有给t1这个表加了个主键索引就是为了辅助验证这个东西（t1表的id列是主键列不允许重复，所以这也侧面印证了事务1的数据已经写入成功了），个人理解事务的隔离级别和索引的关系不大，当然，也可能是我了解的不够深入，我也希望是我了解的不够深入。

mysql> insert into t1 values (777,'admin2');   //插入其他数据是可以的
Query OK, 1 row affected (0.00 sec)

mysql>
mysql> commit;    //提交
Query OK, 0 rows affected (0.02 sec)

mysql> select * from t1;          //提交之后2个事务的数据都可以看到了
+-----+--------+
| id  | name   |
+-----+--------+
| 666 | admin  |
| 777 | admin2 |
+-----+--------+
2 rows in set (0.00 sec)

mysql>

```
