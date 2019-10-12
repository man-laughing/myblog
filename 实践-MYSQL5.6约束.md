
---
title: 实践-MYSQL5.6约束
date: 2019/10/12
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.6  |3.10.0   |MariaDB-10.1.41 (Mysql-5.6.45)   |

  

## 约束的分类
* 非空约束  not null
* 默认约束  default 
* 自增约束  auto_increment
* 唯一约束  unique key
* 主键约束  primary key
* 外键约束  foreign key


## 各类约束配置示例


#### 非空约束

* 见示例

```bash
MySQL [zzz]> create table t2 (age tinyint not null); //not null声明非空约束
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]>
MySQL [zzz]> desc t2;
+-------+------------+------+-----+---------+-------+
| Field | Type       | Null | Key | Default | Extra |
+-------+------------+------+-----+---------+-------+
| age   | tinyint(4) | NO   |     | NULL    |       |
+-------+------------+------+-----+---------+-------+
1 row in set (0.00 sec)

MySQL [zzz]>
MySQL [zzz]> insert into t2 values (25);
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t2 values (30);
Query OK, 1 row affected (0.01 sec)

MySQL [zzz]> insert into t2 values (NULL);
ERROR 1048 (23000): Column 'age' cannot be null  //非空约束生效，系统提示报错
MySQL [zzz]> 
MySQL [zzz]> select * from t2;
+-----+
| age |
+-----+
|  25 |
|  30 |
+-----+
2 rows in set (0.00 sec)

MySQL [zzz]>

```

#### 默认约束

* 见示例

```bash
MySQL [zzz]> create table t3 (id int not null,name varchar(12) not null default 'xiaobai');    //通过default关键字来声明默认约束
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]> desc t3;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | NO   |     | NULL    |       |
| name  | varchar(12) | NO   |     | xiaobai |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.01 sec)

MySQL [zzz]> insert into t3 values (1,'admin');
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t3 (id) values (2); //这里没有指定name的值
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> select * from t3;     //这里可以看到id为2的行name字段应用了默认值
+----+---------+
| id | name    |
+----+---------+
|  1 | admin   |
|  2 | xiaobai |
+----+---------+ 
2 rows in set (0.00 sec)

MySQL [zzz]>
```

#### 自增约束

* 见示例

```bash
MySQL [zzz]>
MySQL [zzz]> create table t4 (id int auto_increment primary key); //这里通过关键字auto_increment来指定自增长，但自增字段必须要关联一个key，这里指定为了主键
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]> show index from t4;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t4    |          0 | PRIMARY  |            1 | id          | A         |           2 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

MySQL [zzz]>


MySQL [zzz]> desc t4;
+-------+---------+------+-----+---------+----------------+
| Field | Type    | Null | Key | Default | Extra          |
+-------+---------+------+-----+---------+----------------+
| id    | int(11) | NO   | PRI | NULL    | auto_increment |
+-------+---------+------+-----+---------+----------------+
1 row in set (0.01 sec)

MySQL [zzz]>
MySQL [zzz]> insert into t4 values ();  //插入数据不指定值，验证是否自增
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t4 values ();  //插入数据不指定值，验证是否自增
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]>
MySQL [zzz]> select * from t4;          //这里看到自增列数据确实是按照顺序自增的
+----+
| id |
+----+
|  1 |
|  2 |
+----+
2 rows in set (0.00 sec)

MySQL [zzz]>
MySQL [zzz]> delete from t4 where id = 2;  //这里删除了一行数据
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t4 values ();     //这里插入了一行新数据
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> select * from t4;             //可以看到自增ID还是按照删除的数据的ID+1，并且该值是唯一的，因为id这个字段是主键（主键其实是唯一索引）
+----+
| id |
+----+
|  1 |
|  3 |
+----+
2 rows in set (0.00 sec)

### 小知识 ###
1、含有自增字段的表删除了某一行的数据，新增行的自增字段的值还是会从之前删除的行自增ID+1
#############
```

#### 唯一约束

* 见示例

```bash
MySQL [zzz]> create table t5 (name varchar(32) unique);    //通过关键字unique指定唯一约束，同时也会创建对应的唯一索引
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]> show index from t5;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t5    |          0 | name     |            1 | name        | A         |           2 |     NULL | NULL   | YES  | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

MySQL [zzz]> desc t5;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| name  | varchar(32) | YES  | UNI | NULL    |       |
+-------+-------------+------+-----+---------+-------+
1 row in set (0.00 sec)

MySQL [zzz]> insert into t5 values ('admin');
Query OK, 1 row affected (0.01 sec)

MySQL [zzz]> insert into t5 values ('admin2');
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t5 values ('admin');            //这里尝试插入相同的数据系统就报错了
ERROR 1062 (23000): Duplicate entry 'admin' for key 'name'  
MySQL [zzz]>
MySQL [zzz]> select * from t5;
+--------+
| name   |
+--------+
| admin  |
| admin2 |
+--------+
2 rows in set (0.00 sec)
```


#### 主键约束

* 见示例

```bash
MySQL [zzz]> create table t6 (id int not null auto_increment primary key,name varchar(32)); //通过关键字primary key来指定主键约束，同时也会创建主键索引
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]> show index from t6;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| t6    |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

MySQL [zzz]> insert into t6 values (1,'admin');
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t6 (name) values ('laughing');
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> select * from t6;
+----+----------+
| id | name     |
+----+----------+
|  1 | admin    |
|  2 | laughing |
+----+----------+
2 rows in set (0.00 sec)

MySQL [zzz]>

### 小知识 ###
1、主键字段一般可以表示数据的全局唯一性，是很重要的。
#############
```

#### 外键约束

* 见示例

```bash
MySQL [zzz]> create table vendors (id int not null auto_increment primary key ,name varchar(32)); //创建参照表vendors
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]> insert into vendors values (1,'apple');
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into vendors values (2,'sony');
Query OK, 1 row affected (0.01 sec)

MySQL [zzz]> select * from vendors;
+----+-------+
| id | name  |
+----+-------+
|  1 | apple |
|  2 | sony  |
+----+-------+
2 rows in set (0.00 sec)

MySQL [zzz]> create table goods (id int not null auto_increment primary key,name varchar(32),vendor_id int not null, constraint foreign key (vendor_id) references vendors (id) );  //创建有外键关联的业务表goods
Query OK, 0 rows affected (0.01 sec)

MySQL [zzz]>
MySQL [zzz]> insert into goods values (1,'iphone 6s',3);   //这里插入的数据的vendor_id不在参照表vendors里面，受到外键约束所以系统报错了
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`zzz`.`goods`, CONSTRAINT `goods_ibfk_1` FOREIGN KEY (`vendor_id`) REFERENCES `vendors` (`id`))
MySQL [zzz]>
MySQL [zzz]>
MySQL [zzz]> insert into goods values (1,'iphone 6s',1);   //正确的插入数据
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> select * from goods;
+----+-----------+-----------+
| id | name      | vendor_id |
+----+-----------+-----------+
|  1 | iphone 6s |         1 |
+----+-----------+-----------+
1 row in set (0.00 sec)

MySQL [zzz]>
```


## 扩展延伸

#### 空值与NULL的区别？

* 见示例

```bash
MySQL [zzz]> create table t1 (id int,name varchar(32));
Query OK, 0 rows affected (0.04 sec)

MySQL [zzz]> desc t1;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | YES  |     | NULL    |       |
| name  | varchar(32) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

MySQL [zzz]> insert into t1 values (1,'admin');  //插入普通数据
Query OK, 1 row affected (0.01 sec)

MySQL [zzz]>
MySQL [zzz]> insert into t1 values (2,'');  //插入空字符串
Query OK, 1 row affected (0.00 sec)

MySQL [zzz]> insert into t1 (id) values (3);  //这里没插入name的值所以遵循默认NULL
Query OK, 1 row affected (0.01 sec)

MySQL [zzz]> select * from t1;
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 |       |
|    3 | NULL  |
+------+-------+
3 rows in set (0.00 sec)

MySQL [zzz]> select count(*) from t1;
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.00 sec)

MySQL [zzz]> select count(name) from t1;  //可以看到NULL空值字段不参与Count统计
+-------------+
| count(name) |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)

MySQL [zzz]> select * from t1 where name is null;  //空值可以使用（is null）来检索
+------+------+
| id   | name |
+------+------+
|    3 | NULL |
+------+------+
1 row in set (0.00 sec)

MySQL [zzz]>
MySQL [zzz]> select * from t1 where name is not null; //这里也间接说明了空字符串（''）不属于空值，切记！
+------+-------+
| id   | name  |
+------+-------+
|    1 | admin |
|    2 |       |
+------+-------+
2 rows in set (0.00 sec)

MySQL [zzz]>


### 简单总结 ####
1、空字符串（''）是可以被检索出来的，并且参与Count函数统计
2、空值（NULL）也是可以检索出来的，但是需要where携带条件is null来完成，并且不参与Count函数统计
################

```


