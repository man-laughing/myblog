
---
title: 实践-MYSQL组复制
date: 2020/04/26
tags: 
  mysql
---
  

## 环境
|OS          |Kernel   |    Software    |
|:-----------|:-----   | :-----------   | 
|CentOS 7.3  |3.10.0   | 5.7.21-21-log Percona Server |
 

## 先决条件
* MYSQL 5.7.17+,至少三台节点
* 存储引擎必须用InnoDB引擎
* server_id=1
* gtid_mode=ON
* enforce_gtid_consistency=ON
* master_info_repository=TABLE
* relay_log_info_repository=TABLE
* binlog_checksum=NONE
* log_slave_updates=ON
* log_bin=binlog
* binlog_format=ROW
* 详细参考（https://dev.mysql.com/doc/refman/5.7/en/group-replication-requirements.html）

## 注意事项
* 无

## 单主模式

#### 环境

|IP         |Role    |    
|:----------|:----   |   
|10.99.72.4 |Master  |  
|10.99.72.5 |Slave   |  
|10.99.72.6 |Slave   |  


NOTE: 我们默认认为以上三个MYSQL已部署好且正在运行中。。

#### Master配置

* 安装组复制插件、配置复制恢复帐号  

```bash
[root@localhost][(none)]> SET SQL_LOG_BIN=0;  //不记录操作日志到binlog
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> INSTALL PLUGIN group_replication SONAME 'group_replication.so';     //安装组复制插件
Query OK, 0 rows affected (0.01 sec)

[root@localhost][(none)]> GRANT REPLICATION SLAVE ON *.* TO replication@'%' identified by 'replication';  //建立复制恢复账户
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

* 添加组复制配置文件，再my.cnf中[mysqld]下增加如下

```bash
#Mysql Group Replication
transaction_write_set_extraction=XXHASH64                                  //每个事务使用XXHASH64哈希算法将其编码为散列
loose-group_replication_group_name="a9bc3eba-8785-11ea-8caf-fa163e0d03a0"  //组复制中组的名字，通过select uuid()生成的随机数
loose-group_replication_start_on_boot=off                                  //MYSQL启动时不启动组复制
loose-group_replication_bootstrap_group=off                                //MYSQL启动时不进行组复制引导 
loose-group_replication_local_address= "10.99.72.4:24901"                  //本节点地址，端口任意这里配置为24901
loose-group_replication_group_seeds= "10.99.72.4:24901,10.99.72.5:24901,10.99.72.6:24901"    //组复制中的所有节点地址
loose-group_replication_single_primary_mode=on                             //组复制模式，为on表示单主模式
```

* 重启MYSQL

```bash
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqladmin  -S mysql.sock shutdown
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/3307_test/my.cnf --user=mysql & 
```

* 开始组复制引导，只需要在Master上执行即可

```bash
[root@localhost][(none)]>  set @@global.group_replication_bootstrap_group=on;   //设置开始组复制引导
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> CHANGE MASTER TO MASTER_USER='replication', MASTER_PASSWORD='replication' FOR CHANNEL 'group_replication_recovery';  //配置复制账户
Query OK, 0 rows affected, 2 warnings (0.03 sec)

[root@localhost][(none)]> start group_replication;    //启动组复制
Query OK, 0 rows affected (2.01 sec)

[root@localhost][(none)]>  set @@global.group_replication_bootstrap_group=off;   //关闭组复制引导
Query OK, 0 rows affected (0.00 sec)
[root@localhost][(none)]>
[root@localhost][(none)]> select * from performance_schema.replication_group_members;   //查看当前组复制成员 
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                     | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| group_replication_applier | 21ea07b2-8784-11ea-9a8e-fa163e0d03a0 | qs-storage-swift-01.localdomain |        3307 | ONLINE       |  //ONLINE即为正常
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```
#### Slave配置

NOTE：以下配置需要在所有Slave节点上配置一遍

* 安装组复制插件、配置复制恢复帐号  

```bash
[root@localhost][(none)]> SET SQL_LOG_BIN=0;  //不记录操作日志到binlog 
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> INSTALL PLUGIN group_replication SONAME 'group_replication.so';     //安装组复制插件
Query OK, 0 rows affected (0.01 sec)

[root@localhost][(none)]> GRANT REPLICATION SLAVE ON *.* TO replication@'%' identified by 'replication';  //建立复制恢复账户
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

* 添加组复制配置文件，再my.cnf中[mysqld]下增加如下

```bash
#Mysql Group Replication
transaction_write_set_extraction=XXHASH64                                  //每个事务使用XXHASH64哈希算法将其编码为散列
loose-group_replication_group_name="a9bc3eba-8785-11ea-8caf-fa163e0d03a0"  //组复制中组的名字，通过select uuid()生成的随机数
loose-group_replication_start_on_boot=off                                  //MYSQL启动时不启动组复制
loose-group_replication_bootstrap_group=off                                //MYSQL启动时不进行组复制引导 
loose-group_replication_local_address= "10.99.72.5:24901"                  //本节点地址，端口任意这里配置为24901
loose-group_replication_group_seeds= "10.99.72.4:24901,10.99.72.5:24901,10.99.72.6:24901"    //组复制中的所有节点地址
loose-group_replication_single_primary_mode=on                             //组复制模式，为on表示单主模式
```

* 重启MYSQL

```bash
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqladmin  -S mysql.sock shutdown
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/3307_test/my.cnf --user=mysql & 
```

* 加入到组复制

```bash
[root@localhost][(none)]> CHANGE MASTER TO MASTER_USER='replication', MASTER_PASSWORD='replication' FOR CHANNEL 'group_replication_recovery';  //配置复制账户
Query OK, 0 rows affected, 2 warnings (0.03 sec)

[root@localhost][(none)]> start group_replication;    //启动组复制
Query OK, 0 rows affected (2.01 sec)

[root@localhost][(none)]> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                     | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| group_replication_applier | 21ea07b2-8784-11ea-9a8e-fa163e0d03a0 | qs-storage-swift-01.localdomain |        3307 | ONLINE       |
| group_replication_applier | 4fbcdf84-8784-11ea-963c-fa163e53df80 | qs-storage-swift-02.localdomain |        3307 | ONLINE       |  //这里可以看到本机已经加入
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
2 rows in set (0.00 sec)

[root@localhost][(none)]>
```

#### 如何查找当前组复制集群中的Master节点和集群成员

* 查找集群中所有组成员

```bash
[root@localhost][(none)]> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                     | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| group_replication_applier | 21ea07b2-8784-11ea-9a8e-fa163e0d03a0 | qs-storage-swift-01.localdomain |        3307 | ONLINE       |
| group_replication_applier | 4fbcdf84-8784-11ea-963c-fa163e53df80 | qs-storage-swift-02.localdomain |        3307 | ONLINE       |
| group_replication_applier | 5dcfcd48-8784-11ea-8bec-fa163efe9a4b | qs-storage-swift-03.localdomain |        3307 | ONLINE       |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
3 rows in set (0.00 sec)

[root@localhost][(none)]>
```

* 查找集群中的Master节点

```bash
[root@localhost][(none)]> SHOW global STATUS LIKE 'group_replication_primary_member' ;
+----------------------------------+--------------------------------------+
| Variable_name                    | Value                                |
+----------------------------------+--------------------------------------+
| group_replication_primary_member | 21ea07b2-8784-11ea-9a8e-fa163e0d03a0 |
+----------------------------------+--------------------------------------+
1 row in set (0.01 sec)

[root@localhost][(none)]>
```

#### 如何手动进行故障转移（提升某台Slave为Master）

NOTE:在单主模式下，只有一个节点可以可以读写，其他节点只能提供读，在单主模式下，该参数 group_replication_enforce_update_everywhere_checks 必须被设置为 FALSE ，当主节点宕掉，自动会根据服务器的server_uuid变量和group_replication_member_weight变量值，选择下一个slave谁作为主节点，group_replication_member_weight的值最高的成员被选为新的主节点，在group_replication_member_weight值相同的情况下，group根据数据字典中 server_uuid排序，排序在最前的被选择为主节点

* 查看Slave1的权重设置
 
```bash
[root@localhost][(none)]> select @@group_replication_member_weight;
+-----------------------------------+
| @@group_replication_member_weight |
+-----------------------------------+
|                                50 |
+-----------------------------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```

* 查看Slave2的权重设置

```bash
[root@localhost][(none)]> select @@group_replication_member_weight;   //可以看到这里是60，根据原理如果Master故障了此Slave节点会被选为Master
+-----------------------------------+
| @@group_replication_member_weight |
+-----------------------------------+
|                                60 |
+-----------------------------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```


* 关闭Master的MYSQL的进程，手动模拟故障

```bash
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqladmin  -S mysql.sock shutdown
```

* 在高权重的那个Slave节点上查看

```bash
[root@localhost][(none)]> select @@group_replication_member_weight;
+-----------------------------------+
| @@group_replication_member_weight |
+-----------------------------------+
|                                60 |
+-----------------------------------+
1 row in set (0.00 sec)
[root@localhost][(none)]>  select @@server_uuid;
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| 5dcfcd48-8784-11ea-8bec-fa163efe9a4b |
+--------------------------------------+
1 row in set (0.00 sec)

[root@localhost][(none)]> SHOW global STATUS LIKE 'group_replication_primary_member' ;   //可以看到集群的Master已经变为本机节点了
+----------------------------------+--------------------------------------+
| Variable_name                    | Value                                |
+----------------------------------+--------------------------------------+
| group_replication_primary_member | 5dcfcd48-8784-11ea-8bec-fa163efe9a4b |
+----------------------------------+--------------------------------------+
1 row in set (0.01 sec)

[root@localhost][(none)]>
```

## 多主模式

#### 环境

|IP         |Role    |    
|:----------|:----   |   
|10.99.72.4 |Master  |  
|10.99.72.5 |Master  |  
|10.99.72.6 |Master  |  


NOTE: 我们默认认为以上三个MYSQL已部署好且正在运行中。。

#### Master配置

* 安装组复制插件、配置复制恢复帐号  

```bash
[root@localhost][(none)]> SET SQL_LOG_BIN=0;  //不记录操作日志到binlog
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> INSTALL PLUGIN group_replication SONAME 'group_replication.so';     //安装组复制插件
Query OK, 0 rows affected (0.01 sec)

[root@localhost][(none)]> GRANT REPLICATION SLAVE ON *.* TO replication@'%' identified by 'replication';  //建立复制恢复账户
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

* 添加组复制配置文件，再my.cnf中[mysqld]下增加如下

```bash
#Mysql Group Replication
transaction_write_set_extraction=XXHASH64                                   //每个事务使用XXHASH64哈希算法将其编码为散列
loose-group_replication_group_name="3b37704a-879e-11ea-8550-fa163e0d03a0"   //组复制中组的名字，通过select uuid()生成的随机数
loose-group_replication_start_on_boot=off                                   //MYSQL启动时不启动组复制
loose-group_replication_bootstrap_group=off                                 //MYSQL启动时不进行组复制引导 
loose-group_replication_local_address= "10.99.72.4:24901"                   //本节点地址，端口任意这里配置为24901
loose-group_replication_group_seeds= "10.99.72.4:24901,10.99.72.5:24901,10.99.72.6:24901"    //组复制中的所有节点地址
loose-group_replication_single_primary_mode=off                             //组复制模式，为off表示关闭单主模式，即为多主模式
loose-group_replication_enforce_update_everywhere_checks=on                 //启用任意位置更新，也表示是在多主模式下
loose-group_replication_ip_whitelist="10.99.72.0/24                         //集群的IP白名单，非必须
```

* 重启MYSQL

```bash
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqladmin  -S mysql.sock shutdown
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/3307_test/my.cnf --user=mysql & 
```

* 开始组复制引导，只需要在Master上执行即可

```bash
[root@localhost][(none)]>  set @@global.group_replication_bootstrap_group=on;   //设置开始组复制引导
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> CHANGE MASTER TO MASTER_USER='replication', MASTER_PASSWORD='replication' FOR CHANNEL 'group_replication_recovery';  //配置复制账户
Query OK, 0 rows affected, 2 warnings (0.03 sec)

[root@localhost][(none)]> start group_replication;    //启动组复制
Query OK, 0 rows affected (2.01 sec)

[root@localhost][(none)]>  set @@global.group_replication_bootstrap_group=off;   //关闭组复制引导
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]>
[root@localhost][(none)]> select * from performance_schema.replication_group_members ;
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                     | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| group_replication_applier | a90c43cc-879d-11ea-af71-fa163e0d03a0 | qs-storage-swift-01.localdomain |        3307 | ONLINE       |   //可以看到ONLINE
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```

#### Slave配置

NOTE：以下配置需要在所有Slave节点上配置一遍

* 安装组复制插件、配置复制恢复帐号  

```bash
[root@localhost][(none)]> SET SQL_LOG_BIN=0;  //不记录操作日志到binlog 
Query OK, 0 rows affected (0.00 sec)

[root@localhost][(none)]> INSTALL PLUGIN group_replication SONAME 'group_replication.so';     //安装组复制插件
Query OK, 0 rows affected (0.01 sec)

[root@localhost][(none)]> GRANT REPLICATION SLAVE ON *.* TO replication@'%' identified by 'replication';  //建立复制恢复账户
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

* 添加组复制配置文件，再my.cnf中[mysqld]下增加如下

```bash
#Mysql Group Replication
transaction_write_set_extraction=XXHASH64                                   //每个事务使用XXHASH64哈希算法将其编码为散列
loose-group_replication_group_name="3b37704a-879e-11ea-8550-fa163e0d03a0"   //组复制中组的名字，通过select uuid()生成的随机数
loose-group_replication_start_on_boot=off                                   //MYSQL启动时不启动组复制
loose-group_replication_bootstrap_group=off                                 //MYSQL启动时不进行组复制引导 
loose-group_replication_local_address= "10.99.72.5:24901"                   //本节点地址，端口任意这里配置为24901
loose-group_replication_group_seeds= "10.99.72.4:24901,10.99.72.5:24901,10.99.72.6:24901"    //组复制中的所有节点地址
loose-group_replication_single_primary_mode=off                             //组复制模式，为off表示关闭单主模式，即为多主模式
loose-group_replication_enforce_update_everywhere_checks=on                 //启用任意位置更新，也表示是在多主模式下
loose-group_replication_ip_whitelist="10.99.72.0/24                         //集群的IP白名单，非必须
```

* 重启MYSQL

```bash
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqladmin  -S mysql.sock shutdown
[root@qs-storage-swift-01 3307_test]#  /opt/app/mysql/bin/mysqld_safe --defaults-file=/data0/mysql/3307_test/my.cnf --user=mysql & 
```

* 加入到组复制

```bash
[root@localhost][(none)]> CHANGE MASTER TO MASTER_USER='replication', MASTER_PASSWORD='replication' FOR CHANNEL 'group_replication_recovery';  //配置复制账户
Query OK, 0 rows affected, 2 warnings (0.03 sec)

[root@localhost][(none)]> start group_replication;    //启动组复制
Query OK, 0 rows affected (2.01 sec)

[root@localhost][(none)]> select * from performance_schema.replication_group_members  ;
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                     | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
| group_replication_applier | a90c43cc-879d-11ea-af71-fa163e0d03a0 | qs-storage-swift-01.localdomain |        3307 | ONLINE       |
| group_replication_applier | c165f895-879d-11ea-a6b6-fa163e53df80 | qs-storage-swift-02.localdomain |        3307 | ONLINE       |  //本机已经成功加入组
+---------------------------+--------------------------------------+---------------------------------+-------------+--------------+
2 rows in set (0.00 sec)

[root@localhost][(none)]>
```

#### 同步验证


* 任意一个节点创建一个测试数据库

```bash
[root@localhost][(none)]> create database dbtest01;
Query OK, 1 row affected (0.00 sec)

[root@localhost][(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dbtest01           |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

[root@localhost][(none)]> select @@server_uuid;
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| a90c43cc-879d-11ea-af71-fa163e0d03a0 |
+--------------------------------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```

* 另外的节点来验证

```bash
[root@localhost][(none)]> show databases;   //可以看到数据已经有了，数据同步是OK的
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dbtest01           |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

[root@localhost][(none)]> select @@server_uuid;
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| c165f895-879d-11ea-a6b6-fa163e53df80 |
+--------------------------------------+
1 row in set (0.00 sec)

[root@localhost][(none)]>
```

* 检查主

```bash
[root@localhost][(none)]> SHOW global STATUS LIKE 'group_replication_primary_member'   //因为是多主模式，所有节点都是主，所以这个变量的值是空的
    -> ;
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| group_replication_primary_member |       |
+----------------------------------+-------+
1 row in set (0.01 sec)

[root@localhost][(none)]>
```

