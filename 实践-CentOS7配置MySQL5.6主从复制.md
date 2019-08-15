
---
title: 实践-CentOS7配置MySQL5.6主从复制
date: 2019/08/15
tags: 
   mysql
---


### 环境
|OS          |Kernel   |   IP   |Mysql version|Role|
|------------|--------:|:-------|:------------|:---|
|CentOS 7.6  |3.10.0   |10.99.66.152|5.6.45   |Master|
|CentOS 7.6  |3.10.0   |10.99.66.153|5.6.45   |Slave |

### 先决条件

* 保证机器可以连通网络
* 保证两台机器数据库服务运行正常（安装过程略）


#### 使用BINLOG配置主从复制

* 在Master上开启binlog记录，如下-my.cnf

```bash
[root@localhost mysql]# cat my.cnf
[mysqld]
server_id = 66152
basedir   = /usr/local/mysql
datadir   = /usr/local/mysql/data
port      = 3306
socket    = /usr/local/mysql/data/mysql.sock
pid-file  = /usr/local/mysql/data/mysql.pid
log-error = /usr/local/mysql/data/mysql.error
log-bin   = mysql-bin    //开启binlog
sql_mode  = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
[root@localhost mysql]# 
[root@localhost mysql]# bin/mysqld_safe --defaults-file=/usr/local/mysql/my.cnf --user=mysql &
```

* 在Master上创建复制专用账户

```bash
[root@essessmove-shqs-11 mysql]# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.6.45-log MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> grant replication slave on *.* to 'replication'@'10.99.66.153' identified by 'replication';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 在Master上查看binglog文件号码和记录位置

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      415 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql>
```

* 在Slave上配置、启动复制进程

```bash
mysql> change master to master_host='10.99.66.154',master_port=3306,master_user='replication',master_password='replication',master_log_file='mysql-bin.000005',master_log_pos=415;
mysql> start slave;
``` 

* 在Slave上查看状态

```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.99.66.152
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 415
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 415
              Relay_Log_Space: 456
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 66152
                  Master_UUID: 4f2f4e3c-bf29-11e9-8164-fa163eba8cc3
             Master_Info_File: /usr/local/mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)
```


#### 使用GTID配置主从复制

这里是新环境，Master:10.99.66.154  Slave:10.99.66.158

* 在Master/Slave上开启binlog记录，如下-my.cnf

```bash
[root@localhost mysql]# cat my.cnf
[mysqld]
server_id = 66154
basedir   = /usr/local/mysql
datadir   = /usr/local/mysql/data
port      = 3306
socket    = /usr/local/mysql/logs/mysql.sock
pid-file  = /usr/local/mysql/logs/mysql.pid
log-error = /usr/local/mysql/logs/mysql.error
log-bin   = mysql-bin
binlog-format  = ROW     //修改binglog格式为row
gtid-mode = on           //启用GTID模式
enforce-gtid-consistency = true
log-slave-updates        = true
sql_mode  = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
[root@localhost mysql]# 
[root@localhost mysql]# bin/mysqld_safe --defaults-file=/usr/local/mysql/my.cnf --user=mysql &
```

* 在Master上创建复制专用账户

```bash
[root@essessmove-shqs-11 mysql]# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.6.45-log MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> grant replication slave on *.* to 'replication'@'10.99.66.158' identified by 'replication';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

* 在Master上查看binglog文件号码和记录位置

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000003 |     4322 |              |                  | 3b827fcb-bf40-11e9-81f9-fa163ec8903f:1-13 |
+------------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)

mysql>
```

* 在Slave上配置、启动复制进程

```bash
mysql> change master to master_host='10.99.66.154',master_port=3306,master_user='replication',master_password='replication',master_log_file='mysql-bin.000003',master_log_pos=4322;
Query OK, 0 rows affected, 2 warnings (0.02 sec)
mysql> start slave;
```

* 在Slave上查看复制状态

```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.99.66.154
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 4322
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 864
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 4322
              Relay_Log_Space: 1068
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 66154
                  Master_UUID: 3b827fcb-bf40-11e9-81f9-fa163ec8903f
             Master_Info_File: /usr/local/mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 3b827fcb-bf40-11e9-81f9-fa163ec8903f:11-13
            Executed_Gtid_Set: 3b827fcb-bf40-11e9-81f9-fa163ec8903f:11-13,
ff3537f1-bf3d-11e9-81ea-fa163e003bbb:1-8
                Auto_Position: 0
1 row in set (0.00 sec)

mysql>
```