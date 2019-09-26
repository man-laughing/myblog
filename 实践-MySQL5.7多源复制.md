
---
title: 实践-MySQL5.7多源复制
date: 2019/09/26
tags: 
   mysql
---
  

## 环境
|OS          |Kernel   | Mysql version| 
|------------|--------:| :------------| 
|CentOS 7.3  |3.10.0   |5.7.19        |
  
## 先决条件

* 请确保你的机器已经安装好MySQL-5.7.19软件
 
## 机器清单

|IP           | PORT  | BASEDIR  |   ROLE   |
|-------------|-------|----------|----------|
|10.116.71.209| 3331  |/opt/app/mysql/3331-mysql| MasterA|
|10.116.71.209| 3332  |/opt/app/mysql/3332-mysql| MasterB|
|10.116.71.209| 3333  |/opt/app/mysql/3333-mysql| Slave  |

## 配置阶段
#### Master A的配置

* 建立mysql用户及基本目录等
 
```bash
[root@localhost mysql]# useradd mysql -s /sbin/nologin
[root@localhost mysql]# mkdir /opt/app/mysql/3331-mysql
[root@localhost mysql]# mkdir /opt/app/mysql/3331-mysql/data
[root@localhost mysql]# mkdir /opt/app/mysql/3331-mysql/log
[root@localhost mysql]# chown -R mysql.mysql /opt/app/mysql/3331-mysql
[root@localhost mysql]#
```

* 配置文件my.cnf

```bash
#/opt/app/mysql/3331-mysql/my.cnf
[client]
port            = 3331
socket          = /opt/app/mysql/3331-mysql/mysql.sock

[mysqld]
##### Basic #####
port=3331
bind-address=10.116.71.209
server-id=712093331
basedir=/opt/app/mysql-zltest/3331-zltest
datadir=/opt/app/mysql/3331-mysql/data
socket=/opt/app/mysql/3331-mysql/mysql.sock
pid-file=/opt/app/mysql/3331-mysql/mysql.pid
user=mysql
character_set_server=utf8mb4
skip_name_resolve=1
explicit_defaults_for_timestamp = 1
transaction_isolation=READ-COMMITTED
log_bin_trust_function_creators=1
sql_mode=''

##### Connect #####
max_allowed_packet=128M
skip-name-resolve
max_connections=2000
max_connect_errors=10000
max_user_connections=2000

##### InnoDB #####
innodb_buffer_pool_size=2G
innodb_data_file_path=ibdata1:256M:autoextend
innodb_buffer_pool_dump_at_shutdown=1
innodb_buffer_pool_dump_pct=40
innodb_buffer_pool_load_at_startup=1
innodb_buffer_pool_instances=8
innodb_flush_method=O_DIRECT
innodb_thread_concurrency=0
innodb_log_file_size=500M
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_file_per_table=1
innodb_io_capacity=400
innodb_io_capacity_max=800
innodb_large_prefix=1
innodb_lru_scan_depth=4096
innodb_online_alter_log_max_size=1G
innodb_open_files=10000
innodb_page_cleaners=8
innodb_page_size=16384
innodb_print_all_deadlocks=1
innodb_purge_rseg_truncate_frequency=128
innodb_purge_threads=4
innodb_read_io_threads=8
innodb_write_io_threads=8
innodb_stats_persistent_sample_pages=64
#innodb_undo_logs=128
#innodb_undo_tablespaces=3
#innodb_undo_log_truncate=1
#innodb_max_undo_log_size=256M
#audit_offsets=7784,7832,3608,4760,456,360,0,32,64,160,536,7948,4336,3640,3648,3652,6032,2048,8,7016,7056,7040
#plugin-load=AUDIT=libaudit_plugin.so
#audit_json_file=on
lower_case_table_names=1
key_buffer_size=256M
innodb_log_buffer_size=16M
innodb_log_files_in_group=2
innodb_sort_buffer_size=64M
join_buffer_size=20M
sort_buffer_size=20M
read_buffer_size=20M
read_rnd_buffer_size=20M
thread_cache_size=1024

#####log#####
binlog_format=row
binlog_gtid_simple_recovery=1
binlog_rows_query_log_events = 1
binlog-rows-query-log-events = 1
expire_logs_days=2
slow-query-log=1
long_query_time=1
log_queries_not_using_indexes=0
log_throttle_queries_not_using_indexes=10
log_timestamps=system
slow-query-log-file=/opt/app/mysql/3331-mysql/log/slow.log
general_log_file=/opt/app/mysql/3331-mysql/log/general.log
log_error=/opt/app/mysql/3331-mysql/log/error.log
log-bin=/opt/app/mysql/3331-mysql/log/mysql-bin
log-bin=/opt/app/mysql/3331-mysql/log/mysql-bin.index
binlog_cache_size=4M
max_binlog_cache_size=20G
max_binlog_size=1G
sync_binlog=0

#####slave#####
#gtid_mode=on
#enforce_gtid_consistency=1
slave_rows_search_algorithms='INDEX_SCAN,HASH_SCAN'
relay-log=/opt/app/mysql/3331-mysql/log/relay-log
master_info_repository=TABLE
relay_log_info_repository=TABLE
log_slave_updates
skip-slave-start
relay_log_recovery=1
relay-log-purge=1

#####parallelreplication#####
slave_parallel_type=LOGICAL_CLOCK
slave_parallel_workers=4
slave_preserve_commit_order=ON
slave_pending_jobs_size_max=3355443200

[mysqldump]
quick
max_allowed_packet              = 128M

[mysql]
no-auto-rehash
default-character-set           = utf8
prompt = [\\u@\\h][\\d]>\\_

[myisamchk]
key_buffer_size = 256M
#sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```

* 初始化数据库、启动数据库

```bash
[root@localhost 3331-mysql]# /opt/app/mysql/bin/mysqld --defaults-file=/opt/app/mysql/3331-mysql/my.cnf  --initialize-insecure
[root@localhost 3331-mysql]#
[root@localhost 3331-mysql]# /opt/app/mysql/bin/mysqld_safe --defaults-file=/opt/app/mysql/3331-mysql/my.cnf &
```

* 登录数据库创建复制专用账号

```bash
mysql> create user replication;
Query OK, 0 rows affected (0.00 sec)

mysql> grant replication slave on *.* to 'replication'@'%' identified by 'replication';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql>
```

* 记录MasterA主机的binlog位置点

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1510 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql>
```

#### Master B的配置
* 建立mysql用户及基本目录等
 
```bash
[root@localhost mysql]# 
[root@localhost mysql]# mkdir /opt/app/mysql/3332-mysql
[root@localhost mysql]# mkdir /opt/app/mysql/3332-mysql/data
[root@localhost mysql]# mkdir /opt/app/mysql/3332-mysql/log
[root@localhost mysql]# chown -R mysql.mysql /opt/app/mysql/3332-mysql
[root@localhost mysql]#
```

* 配置文件my.cnf

```bash
#/opt/app/mysql/3332-mysql/my.cnf
[client]
port            = 3332
socket          = /opt/app/mysql/3332-mysql/mysql.sock

[mysqld]
##### Basic #####
port=3332
bind-address=10.116.71.209
server-id=712093332
basedir=/opt/app/mysql-zltest/3332-zltest
datadir=/opt/app/mysql/3332-mysql/data
socket=/opt/app/mysql/3332-mysql/mysql.sock
pid-file=/opt/app/mysql/3332-mysql/mysql.pid
user=mysql
character_set_server=utf8mb4
skip_name_resolve=1
explicit_defaults_for_timestamp = 1
transaction_isolation=READ-COMMITTED
log_bin_trust_function_creators=1
sql_mode=''

##### Connect #####
max_allowed_packet=128M
skip-name-resolve
max_connections=2000
max_connect_errors=10000
max_user_connections=2000

##### InnoDB #####
innodb_buffer_pool_size=2G
innodb_data_file_path=ibdata1:256M:autoextend
innodb_buffer_pool_dump_at_shutdown=1
innodb_buffer_pool_dump_pct=40
innodb_buffer_pool_load_at_startup=1
innodb_buffer_pool_instances=8
innodb_flush_method=O_DIRECT
innodb_thread_concurrency=0
innodb_log_file_size=500M
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_file_per_table=1
innodb_io_capacity=400
innodb_io_capacity_max=800
innodb_large_prefix=1
innodb_lru_scan_depth=4096
innodb_online_alter_log_max_size=1G
innodb_open_files=10000
innodb_page_cleaners=8
innodb_page_size=16384
innodb_print_all_deadlocks=1
innodb_purge_rseg_truncate_frequency=128
innodb_purge_threads=4
innodb_read_io_threads=8
innodb_write_io_threads=8
innodb_stats_persistent_sample_pages=64
#innodb_undo_logs=128
#innodb_undo_tablespaces=3
#innodb_undo_log_truncate=1
#innodb_max_undo_log_size=256M
#audit_offsets=7784,7832,3608,4760,456,360,0,32,64,160,536,7948,4336,3640,3648,3652,6032,2048,8,7016,7056,7040
#plugin-load=AUDIT=libaudit_plugin.so
#audit_json_file=on
lower_case_table_names=1
key_buffer_size=256M
innodb_log_buffer_size=16M
innodb_log_files_in_group=2
innodb_sort_buffer_size=64M
join_buffer_size=20M
sort_buffer_size=20M
read_buffer_size=20M
read_rnd_buffer_size=20M
thread_cache_size=1024

#####log#####
binlog_format=row
binlog_gtid_simple_recovery=1
binlog_rows_query_log_events = 1
binlog-rows-query-log-events = 1
expire_logs_days=2
slow-query-log=1
long_query_time=1
log_queries_not_using_indexes=0
log_throttle_queries_not_using_indexes=10
log_timestamps=system
slow-query-log-file=/opt/app/mysql/3332-mysql/log/slow.log
general_log_file=/opt/app/mysql/3332-mysql/log/general.log
log_error=/opt/app/mysql/3332-mysql/log/error.log
log-bin=/opt/app/mysql/3332-mysql/log/mysql-bin
log-bin=/opt/app/mysql/3332-mysql/log/mysql-bin.index
binlog_cache_size=4M
max_binlog_cache_size=20G
max_binlog_size=1G
sync_binlog=0

#####slave#####
#gtid_mode=on
#enforce_gtid_consistency=1
slave_rows_search_algorithms='INDEX_SCAN,HASH_SCAN'
relay-log=/opt/app/mysql/3332-mysql/log/relay-log
master_info_repository=TABLE
relay_log_info_repository=TABLE
log_slave_updates
skip-slave-start
relay_log_recovery=1
relay-log-purge=1

#####parallelreplication#####
slave_parallel_type=LOGICAL_CLOCK
slave_parallel_workers=4
slave_preserve_commit_order=ON
slave_pending_jobs_size_max=3355443200

[mysqldump]
quick
max_allowed_packet              = 128M

[mysql]
no-auto-rehash
default-character-set           = utf8
prompt = [\\u@\\h][\\d]>\\_

[myisamchk]
key_buffer_size = 256M
#sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```

* 初始化数据库、启动数据库

```bash
[root@localhost 3332-mysql]# /opt/app/mysql/bin/mysqld --defaults-file=/opt/app/mysql/3332-mysql/my.cnf  --initialize-insecure
[root@localhost 3332-mysql]#
[root@localhost 3332-mysql]# /opt/app/mysql/bin/mysqld_safe --defaults-file=/opt/app/mysql/3332-mysql/my.cnf &
```

* 登录数据库创建复制专用账号

```bash
mysql> create user replication;
Query OK, 0 rows affected (0.00 sec)

mysql> grant replication slave on *.* to 'replication'@'%' identified by 'replication';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql>
```

* 记录MasterB主机的binlog位置点

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      649 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql>
```


#### Slave 的配置

* 建立mysql用户及基本目录等
 
```bash
[root@localhost mysql]# 
[root@localhost mysql]# mkdir /opt/app/mysql/3333-mysql
[root@localhost mysql]# mkdir /opt/app/mysql/3333-mysql/data
[root@localhost mysql]# mkdir /opt/app/mysql/3333-mysql/log
[root@localhost mysql]# chown -R mysql.mysql /opt/app/mysql/3333-mysql
[root@localhost mysql]#
```

* 配置文件my.cnf

```bash
[client]
port            = 3333
socket          = /opt/app/mysql/3333-mysql/mysql.sock

[mysqld]
##### Basic #####
port=3333
bind-address=10.116.71.209
server-id=712093333
basedir=/opt/app/mysql-zltest/3333-zltest
datadir=/opt/app/mysql/3333-mysql/data
socket=/opt/app/mysql/3333-mysql/mysql.sock
pid-file=/opt/app/mysql/3333-mysql/mysql.pid
user=mysql
character_set_server=utf8mb4
skip_name_resolve=1
explicit_defaults_for_timestamp = 1
transaction_isolation=READ-COMMITTED
log_bin_trust_function_creators=1
sql_mode=''

##### Connect #####
max_allowed_packet=128M
skip-name-resolve
max_connections=2000
max_connect_errors=10000
max_user_connections=2000

##### InnoDB #####
innodb_buffer_pool_size=2G
innodb_data_file_path=ibdata1:256M:autoextend
innodb_buffer_pool_dump_at_shutdown=1
innodb_buffer_pool_dump_pct=40
innodb_buffer_pool_load_at_startup=1
innodb_buffer_pool_instances=8
innodb_flush_method=O_DIRECT
innodb_thread_concurrency=0
innodb_log_file_size=500M
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_file_per_table=1
innodb_io_capacity=400
innodb_io_capacity_max=800
innodb_large_prefix=1
innodb_lru_scan_depth=4096
innodb_online_alter_log_max_size=1G
innodb_open_files=10000
innodb_page_cleaners=8
innodb_page_size=16384
innodb_print_all_deadlocks=1
innodb_purge_rseg_truncate_frequency=128
innodb_purge_threads=4
innodb_read_io_threads=8
innodb_write_io_threads=8
innodb_stats_persistent_sample_pages=64
#innodb_undo_logs=128
#innodb_undo_tablespaces=3
#innodb_undo_log_truncate=1
#innodb_max_undo_log_size=256M
#audit_offsets=7784,7832,3608,4760,456,360,0,32,64,160,536,7948,4336,3640,3648,3652,6032,2048,8,7016,7056,7040
#plugin-load=AUDIT=libaudit_plugin.so
#audit_json_file=on
lower_case_table_names=1
key_buffer_size=256M
innodb_log_buffer_size=16M
innodb_log_files_in_group=2
innodb_sort_buffer_size=64M
join_buffer_size=20M
sort_buffer_size=20M
read_buffer_size=20M
read_rnd_buffer_size=20M
thread_cache_size=1024

#####log#####
binlog_format=row
binlog_gtid_simple_recovery=1
binlog_rows_query_log_events = 1
binlog-rows-query-log-events = 1
expire_logs_days=2
slow-query-log=1
long_query_time=1
log_queries_not_using_indexes=0
log_throttle_queries_not_using_indexes=10
log_timestamps=system
slow-query-log-file=/opt/app/mysql/3333-mysql/log/slow.log
general_log_file=/opt/app/mysql/3333-mysql/log/general.log
log_error=/opt/app/mysql/3333-mysql/log/error.log
log-bin=/opt/app/mysql/3333-mysql/log/mysql-bin
log-bin=/opt/app/mysql/3333-mysql/log/mysql-bin.index
binlog_cache_size=4M
max_binlog_cache_size=20G
max_binlog_size=1G
sync_binlog=0

#####slave#####
#gtid_mode=on
#enforce_gtid_consistency=1
slave_rows_search_algorithms='INDEX_SCAN,HASH_SCAN'
relay-log=/opt/app/mysql/3333-mysql/log/relay-log
master_info_repository=TABLE     //很重要，必须配置
relay_log_info_repository=TABLE  //很重要，必须配置
log_slave_updates
skip-slave-start
relay_log_recovery=1
relay-log-purge=1

#####parallelreplication#####
slave_parallel_type=LOGICAL_CLOCK
slave_parallel_workers=4
slave_preserve_commit_order=ON
slave_pending_jobs_size_max=3355443200

[mysqldump]
quick
max_allowed_packet              = 128M

[mysql]
no-auto-rehash
default-character-set           = utf8
prompt = [\\u@\\h][\\d]>\\_

[myisamchk]
key_buffer_size = 256M
#sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```

* 初始化数据库、启动数据库

```bash
[root@localhost 3333-mysql]# /opt/app/mysql/bin/mysqld --defaults-file=/opt/app/mysql/3333-mysql/my.cnf  --initialize-insecure
[root@localhost 3333-mysql]#
[root@localhost 3333-mysql]# /opt/app/mysql/bin/mysqld_safe --defaults-file=/opt/app/mysql/3333-mysql/my.cnf &
```

* 配置与masterA、masterB的主从复制关系

```bash
mysql> change master to master_host = "10.116.71.209",master_port = 3331,master_user = "replication",master_password = "replication",master_log_file = "mysql-bin.000002",master_log_pos = 1510 for channel "test_3331";
Query OK, 0 rows affected, 2 warnings (0.11 sec)

mysql> change master to master_host = "10.116.71.209",master_port = 3332,master_user = "replication",master_password = "replication",master_log_file = "mysql-bin.000002",master_log_pos = 649 for channel "test_3332";
Query OK, 0 rows affected, 2 warnings (0.11 sec)

mysql> start slave ;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.116.71.209
                  Master_User: replication
                  Master_Port: 3331
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1510
               Relay_Log_File: relay-log-test_3331.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000002
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
          Exec_Master_Log_Pos: 1510
              Relay_Log_Space: 531
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
             Master_Server_Id: 712093331
                  Master_UUID: 02dc5b82-e025-11e9-8d78-5254001cc693
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name: test_3331
           Master_TLS_Version:
*************************** 2. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.116.71.209
                  Master_User: replication
                  Master_Port: 3332
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 649
               Relay_Log_File: relay-log-test_3332.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000002
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
          Exec_Master_Log_Pos: 649
              Relay_Log_Space: 531
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
             Master_Server_Id: 712093332
                  Master_UUID: bd5bf27e-e029-11e9-83c0-5254001cc693
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name: test_3332
           Master_TLS_Version:
2 rows in set (0.01 sec)
```
 

## 验证阶段

#### Master A 机器插入数据

```bash
mysql> create database m1;
Query OK, 1 row affected (0.00 sec)

mysql> use m1
Database changed
mysql> show tables;
Empty set (0.00 sec)

mysql> create table tt (id int not null auto_increment primary key,name varchar(32) );
Query OK, 0 rows affected (0.07 sec)

mysql> insert into tt values (1,'admin');
Query OK, 1 row affected (0.00 sec)

mysql> select * from tt;
+----+-------+
| id | name  |
+----+-------+
|  1 | admin |
+----+-------+
1 row in set (0.00 sec)
```

#### Master B 机器插入数据

```bash
mysql> use m2
Database changed
mysql> show tables;
Empty set (0.00 sec)

mysql> create table tt (name varchar(32),age int not null );
Query OK, 0 rows affected (0.07 sec)

mysql> insert into tt values ('laughing',18);
Query OK, 1 row affected (0.00 sec)

mysql> select * from tt;
+----------+-----+
| name     | age |
+----------+-----+
| laughing |  18 |
+----------+-----+
1 row in set (0.00 sec)

mysql>
```

#### Slave  机器验证数据

```bash
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| m1                 |
| m2                 |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)

mysql>
mysql> select * from m1.tt;
+----+-------+
| id | name  |
+----+-------+
|  1 | admin |
+----+-------+
1 row in set (0.00 sec)

mysql>
mysql> select * from m2.tt;
+----------+-----+
| name     | age |
+----------+-----+
| laughing |  18 |
+----------+-----+
1 row in set (0.00 sec)

mysql>
```