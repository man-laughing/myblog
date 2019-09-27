
---
title: 实践-MySQL备份工具mysqldump使用
date: 2019/09/27
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|:-----------|:--------|:-------------| 
|CentOS 7.3  |3.10.0   |5.6.45        |
  
  

## 常用操作

#### 备份所有库

* 待恢复机器不存在任何库

```bash
[root@localhost mysql]# mysqldump -u root -p --all-databases > all_db.sql
```

* 待恢复机器存在某一个或多个库（业务场景允许强制覆盖目标库时）

```bash
[root@localhost mysql]# mysqldump -u root -p --all-databases --add-drop-database > all_db.sql
```

##### 备份一个库

* 待恢复机器不存在该库

```bash
[root@localhost mysql]# mysqldump -u root -p -B  zltest > zltest.sql
```

* 待恢复机器存在该库（业务场景允许强制覆盖目标库时）

```bash
[root@localhost mysql]# mysqldump -u root -p --add-drop-database  --databases  zltest  > zltest.sql
```

#### 备份多个库

* 待恢复机器不存在该库

```bash
[root@localhost mysql]# mysqldump -u root -p --databases zltest2 zltest3 >  zltest2an3.sql
```

* 待恢复机器存在一个或多个库（业务场景允许强制覆盖目标库时）

```bash
[root@localhost mysql]# mysqldump -u root -p --add-drop-database  --databases  zltest2 zltest3 gogs  >  zltest2an3andgogs.sql
```

#### 备份一个库下的某个表

```bash
[root@localhost ~]# mysqldump -u root -p --databases zltest --tables index_test > zltest.index_test.sql
NOTE:注意导出指定表只能针对一个数据库进行导出，且导出的内容中和导出数据库也不一样，导出指定表的导出文本中没有创建数据库的判断语句，只有删除表-创建表-导入数据
也就是说你需要在待恢复的机器上先创建好这个数据库
```

#### 备份一个库下的多个表

```bash
[root@localhost ~]# mysqldump -u root -p --databases zltest --tables index_test  ss > zltest.index_testandss.sql
NOTE:注意导出指定表只能针对一个数据库进行导出，且导出的内容中和导出数据库也不一样，导出指定表的导出文本中没有创建数据库的判断语句，只有删除表-创建表-导入数据
也就是说你需要在待恢复的机器上先创建好这个数据库
```

#### 备份一个库下所有的表结构

* 待恢复机器不存在该库

```bash
[root@localhost ~]# mysqldump -u root -p --databases zltest --no-data   > zltest.sql
```

* 待恢复机器存在该库（业务场景允许强制覆盖目标库时）

```bash
[root@localhost ~]# mysqldump -u root -p --add-drop-database --databases zltest --no-data   > zltest.sql
```

#### 备份一个库下单个/多个的表结构

* 待恢复机器不存在该库

```bash
[root@localhost ~]# mysqldump -u root -p --no-data --databases zltest --tables index_test ss  > zltest.index_test.ss.nodata.sql
```

* 待恢复机器存在该库（业务场景允许强制覆盖目标库时）

```bash
[root@localhost ~]# mysqldump -u root -p --add-drop-database --no-data --databases zltest --tables index_test ss  > zltest.index_test.ss.nodata.sql
```
