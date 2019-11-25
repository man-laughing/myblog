
---
title: 实践-MySQL5.6表空间传输
date: 2019/11/22
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.3  |3.10.0   |5.6.43        |
  
  
## 先决条件
* mysql表使用独立表空间，启用 innodb_file_per_table  参数

## 应用场景
* mysql的大表备份、迁移

## 注意事项
* 在做表导出时，该表只允许读不允许写
* 在 MySQL 5.7.4 之前的版本是不能对分区表做分区迁移
* 使用外键的表，需要使用 set foreign_key_check=0强制忽略外键


## 实际案例

此次实验基于mysql5.6，一共两台机器A和B，A是源头，B为目标。即把A机器的上测试表备份到B机器上，库名、表名都是一致的。

* 在 A 机器上建立测试表并插入一些测试数据

```bash
mysql>
mysql> use zltest
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql>
mysql> create table wtest ( id int ,name varchar(32));
Query OK, 0 rows affected (0.11 sec)

mysql> insert into wtest values (1,'zhangsan'),(2,'lisi');
Query OK, 2 rows affected (0.04 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> select * from wtest;
+------+----------+
| id   | name     |
+------+----------+
|    1 | zhangsan |
|    2 | lisi     |
+------+----------+
2 rows in set (0.00 sec)

mysql>
mysql> show create table wtest;
+-------+----------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                               |
+-------+----------------------------------------------------------------------------------------------------------------------------+
| wtest | CREATE TABLE `wtest` (
  `id` int(11) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+----------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql>
```

* 在 B 机器上也创建对应的测试表(要确保和A机器上的表结构一模一样)

```bash
mysql>
mysql> use zltest
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql>
mysql> create table wtest ( `id` int(11) DEFAULT NULL,`name` varchar(32) DEFAULT NULL  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 ;
Query OK, 0 rows affected (0.02 sec)

mysql>
```

* 在 A 机器上刷新表数据到磁盘

```bash
mysql> flush table wtest  for export ;  //刷新元数据到磁盘，会在data目录生成wtest.cfg文件；此操作会锁表，只能进行SELECT操作。
Query OK, 0 rows affected (0.00 sec)

mysql>
[root@prometheus-shanghai-2 zltest]# ll wtest*
-rw-rw---- 1 mysql mysql   431 Nov 22 20:30 wtest.cfg
-rw-rw---- 1 mysql mysql  8586 Nov 22 19:28 wtest.frm
-rw-rw---- 1 mysql mysql 98304 Nov 22 19:29 wtest.ibd
[root@prometheus-shanghai-2 zltest]#
[root@prometheus-shanghai-2 zltest]# scp -r -P wtest.ibd root@机器B:/tmp/    //复制数据文件到机器B
[root@prometheus-shanghai-2 zltest]# scp -r -P wtest.cfg root@机器B:/tmp/    //复制配置文件到机器B

```

* 在 B 机器上卸载表空间（删除ibd文件）

```bash
mysql> use zltest
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from wtest;
Empty set (0.00 sec)

mysql> alter table wtest discard tablespace;   //卸载表空间
Query OK, 0 rows affected (0.01 sec)

mysql> select * from wtest;    //丢失表空间文件导致DML查询会失败
ERROR 1814 (HY000): Tablespace has been discarded for table 'wtest'
mysql>
[root@essessmove-shqs-10 zltest]# ll wtest*   //此时data目录只剩下frm描述文件
-rw-rw---- 1 mysql mysql 8586 Nov 22 20:28 wtest.frm
```

* 在 B 机器上配置表空间及元数据文件权限(wtest.cfg、wtest.ibd)

```bash
[root@essessmove-shqs-10 zltest]# ll wtest*
-rw-rw---- 1 mysql mysql 8586 Nov 22 20:28 wtest.frm
[root@essessmove-shqs-10 zltest]#
[root@essessmove-shqs-10 zltest]# cp -rf /tmp/wtest.* ./     //拷贝cfg和ibd文件到机器B的mysql数据目录
[root@essessmove-shqs-10 zltest]#
[root@essessmove-shqs-10 zltest]# chown mysql:mysql -R wtest.*     //设置文件属主权限
[root@essessmove-shqs-10 zltest]# ll -tr wtest.*
-rw-rw---- 1 mysql mysql  8586 Nov 22 20:28 wtest.frm
-rw-r----- 1 mysql mysql 98304 Nov 22 20:35 wtest.ibd
-rw-r----- 1 mysql mysql   431 Nov 22 20:35 wtest.cfg
[root@essessmove-shqs-10 zltest]#
```

* 在 B 机器上进行装载表空间文件

```bash
mysql> use zltest
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> alter table wtest import tablespace;    //装在表空间
Query OK, 0 rows affected (0.03 sec)

mysql> check table wtest;   //检查表无误，OK
+--------------+-------+----------+----------+
| Table        | Op    | Msg_type | Msg_text |
+--------------+-------+----------+----------+
| zltest.wtest | check | status   | OK       |
+--------------+-------+----------+----------+
1 row in set (0.00 sec)

mysql> select * from wtest;  //可以看到数据是正确的
+------+----------+
| id   | name     |
+------+----------+
|    1 | zhangsan |
|    2 | lisi     |
+------+----------+
2 rows in set (0.00 sec)

mysql>
```

* 在 A 机器上释放锁

```bash
mysql> unlock tables;   //释放锁
Query OK, 0 rows affected (0.00 sec)

mysql>
```