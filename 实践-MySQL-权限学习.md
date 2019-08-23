
---
title: 实践-MySQL-权限学习
date: 2019/08/22
tags: 
   mysql
---


## 环境
|OS          |Kernel   | Mysql version| 
|------------|--------:| :------------| 
|CentOS 7.6  |3.10.0   |5.6.45   | 
 

## 先决条件

* 保证你的数据库服务运行正常（安装过程略）
 
## 权限分类

#### 基本的DML语句操作权限

* SELECT      //数据表查询权限
* INSERT      //数据表写入权限
* UPDATE      //数据表更新权限
* DELETE      //数据表写入权限

#### 基本的DDL语句操作权限

* CREATE      //库、表、用户创建权限
* DROP        //库、表、用户账户删除权限
* ALTER       //表结构修改权限
* VIEW        //表视图权限
* FUNCTION    //表函数权限
* TRIGGER     //表触发器权限
* PROCEDURE   //表存储过程权限

#### 基本的DCL语句操作权限

* GRANT       //用户权限授权权限
* REVOKE      //用户权限撤销权限


## 最佳实践

* 创建账户zhangsan（密码同用户名）并且赋予该用户在本机对zltest库的所有表拥有所有权限。

```bash
mysql>
mysql> create user zhangsan identified by 'zhangsan';
Query OK, 0 rows affected (0.00 sec)
mysql>
mysql> grant all on zltest.* to zhangsan@'localhost' identified by 'zhangsan';
Query OK, 0 rows affected (0.01 sec)
mysql>
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql>
mysql> show grants for zhangsan@'localhost';
+-----------------------------------------------------------------------------------------------------------------+
| Grants for zhangsan@localhost                                                                                   |
+-----------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'zhangsan'@'localhost' IDENTIFIED BY PASSWORD '*D550CDE8CF0F249C0520BF8CFC424D082D87FEF9' |
| GRANT ALL PRIVILEGES ON `zltest`.* TO 'zhangsan'@'localhost'                                                    |
+-----------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)

mysql>
```

* 创建账户zhangsan（密码同用户名）并且赋予该用户在远程对zltest库的所有表拥有只读权限。

```bash
mysql> grant select on zltest.* to zhangsan@'%' identified by 'zhangsan';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> show grants for zhangsan@'%';
+---------------------------------------------------------------------------------------------------------+
| Grants for zhangsan@%                                                                                   |
+---------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'zhangsan'@'%' IDENTIFIED BY PASSWORD '*D550CDE8CF0F249C0520BF8CFC424D082D87FEF9' |
| GRANT SELECT ON `zltest`.* TO 'zhangsan'@'%'                                                            |
+---------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)

mysql>
```

* 创建账户lisi（密码同用户名）并且赋予该用户在远程和本机的所有库的所有表拥有读写权限。

```bash
mysql> create user lisi;
Query OK, 0 rows affected (0.00 sec)

mysql> grant select,insert on *.* to lisi@'%' identified by 'lisi';
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> show grants for lisi;
+--------------------------------------------------------------------------------------------------------------+
| Grants for lisi@%                                                                                            |
+--------------------------------------------------------------------------------------------------------------+
| GRANT SELECT, INSERT ON *.* TO 'lisi'@'%' IDENTIFIED BY PASSWORD '*E070B7FA2C5695131724E1F395E227147223EF12' |
+--------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql>
```

* 删除zhangsan用户

```bash
mysql> drop user zhangsan;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 创建账户wangwu（密码同用户名）并且赋予该用户在远程和本机拥有创建权限。

```bash
mysql>
mysql> create user wangwu;
Query OK, 0 rows affected (0.00 sec)

mysql> grant create on *.* to wangwu@'%' identified by 'wangwu';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 撤销wangwu账户的所有权限

```bash
mysql> revoke all on *.* from wangwu@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 创建管理员用户admin（密码同用户名）在任何机器上对任何库和表有所有权限并且可以为其他账户做授权操作。

```bash
mysql> create user admin;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql> grant all on *.* to admin@'%' identified by 'admin' with grant option;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 创建用户monitor（密码同用户名）在任何机器上对任何库和表拥有只读权限。

```bash
mysql> create user monitor;
Query OK, 0 rows affected (0.00 sec)

mysql> grant select on *.* to monitor@'%' identified by 'monitor';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

mysql>
```

