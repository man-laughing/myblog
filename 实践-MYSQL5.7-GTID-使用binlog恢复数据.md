---
  title: 实践-MYSQL-5.7.21-GTID-使用binlog恢复数据
  date: 2022/03/03
  tags: 
     mysql
     percona server
---


## 系统环境
| OS      |  Kernel | 
|:----    |:--------| 
|CentOS7.2|3.10.0   | 
 

## 软件环境
| software |version | 
|:---- |:--------| 
|percona-server|5.7.21| 
 

## 先决条件

* 保证MYSQL服务在运行中
*  配置文件正确
*  my.cnf 配置如下

```
[client]
port            = 3781
socket          = /data0/mysql/3781_test/mysql.sock

[mysqld]
##### Basic #####
port=3781
bind-address=10.99.73.251
server-id=732513781
basedir=/opt/app/mysql
datadir=/data0/mysql/3781_test/data
socket=/data0/mysql/3781_test/mysql.sock
pid-file=/data0/mysql/3781_test/mysql.pid
user=mysql
character_set_server=utf8mb4
skip_name_resolve=1
explicit_defaults_for_timestamp = 1
transaction_isolation=READ-COMMITTED
sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

##### Connect #####
max_allowed_packet=64M
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
innodb_io_capacity=200
innodb_io_capacity_max=400
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
innodb_undo_logs=128
innodb_undo_tablespaces=3
innodb_undo_log_truncate=1
innodb_max_undo_log_size=256M
lower_case_table_names=1
key_buffer_size=8M
innodb_log_buffer_size=16M
innodb_log_files_in_group=2
innodb_sort_buffer_size=64M
join_buffer_size=2621440
sort_buffer_size=8M
read_buffer_size=2621440
read_rnd_buffer_size=2621440
thread_cache_size=1024

#####log#####
binlog_format=row
binlog_gtid_simple_recovery=1
binlog_rows_query_log_events = 1
binlog-rows-query-log-events = 1
expire_logs_days=30
slow-query-log=1
long_query_time=1
log_queries_not_using_indexes=0
log_throttle_queries_not_using_indexes=10
log_timestamps=system
slow-query-log-file=/data0/mysql/3781_test/log/slow.log
general_log_file=/data0/mysql/3781_test/log/general.log
log_error=/data0/mysql/3781_test/log/error.log
log-bin=/data0/mysql/3781_test/log/mysql-bin
log-bin=/data0/mysql/3781_test/log/mysql-bin.index
binlog_cache_size=4M
max_binlog_cache_size=20G
max_binlog_size=1G
sync_binlog=0

#####slave#####
gtid_mode=on
enforce_gtid_consistency=1
slave_rows_search_algorithms='INDEX_SCAN,HASH_SCAN'
relay-log=/data0/mysql/3781_test/log/relay-log
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
key_buffer_size = 16M
sort_buffer_size = 16M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```


## 大致思路
* 首先创建模拟数据库zltest、数据表t01，并且插入一些数据
*  误删除数据行
*  继续插入数据
*  误删除表
*  最终实现恢复误删除的表，并且误删除的行也恢复回来

## 实验过程
#### 模拟数据阶段
*  模拟创建数据库、数据表、插入数据

```
mysql(root@localhost:(none))>
mysql(root@localhost:(none))>create database zltest;
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:(none))>use zltest
Database changed
mysql(root@localhost:zltest)>
mysql(root@localhost:zltest)>create table t01 (id int(6) not null primary key ,name varchar(64));
Query OK, 0 rows affected (0.01 sec)

mysql(root@localhost:zltest)>
mysql(root@localhost:zltest)>insert into t01 values(1,'xiao_a');
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>insert into t01 values(2,'xiao_b');
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>insert into t01 values(3,'xiao_c');
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>select * from t01;
+----+--------+
| id | name   |
+----+--------+
|  1 | xiao_a |
|  2 | xiao_b |
|  3 | xiao_c |
+----+--------+
3 rows in set (0.00 sec)

mysql(root@localhost:zltest)>
```

*  误删除数据行

```
mysql(root@localhost:zltest)>delete from t01 where name = 'xiao_b';
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>select * from t01;
+----+--------+
| id | name   |
+----+--------+
|  1 | xiao_a |
|  3 | xiao_c |
+----+--------+
2 rows in set (0.00 sec)

mysql(root@localhost:zltest)>
```

* 继续插入数据

```
mysql(root@localhost:zltest)>insert into t01 values(4,'xiao_d');
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>insert into t01 values(5,'xiao_e');
Query OK, 1 row affected (0.00 sec)

mysql(root@localhost:zltest)>insert into t01 values(6,'xiao_f');
Query OK, 1 row affected (0.01 sec)

mysql(root@localhost:zltest)>select * from t01;
+----+--------+
| id | name   |
+----+--------+
|  1 | xiao_a |
|  3 | xiao_c |
|  4 | xiao_d |
|  5 | xiao_e |
|  6 | xiao_f |
+----+--------+
5 rows in set (0.00 sec)

mysql(root@localhost:zltest)>
```

*  误删除表

```
mysql(root@localhost:zltest)>drop table t01 ;
Query OK, 0 rows affected (0.01 sec)

mysql(root@localhost:zltest)>
mysql(root@localhost:zltest)>show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 2821
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 911e1875-99fd-11ec-b4b0-246e965a8378:1-9
1 row in set (0.01 sec)

mysql(root@localhost:zltest)>
```


#### 表恢复、数据行恢复阶段

* 查看BINLOG中对应的GTID位置

```
mysql(root@localhost:zltest)>show binlog events in 'mysql-bin.000001';
+------------------+------+----------------+-----------+-------------+-----------------------------------------------------------------------------------+
| Log_name         | Pos  | Event_type     | Server_id | End_log_pos | Info                                                                              |
+------------------+------+----------------+-----------+-------------+-----------------------------------------------------------------------------------+
| mysql-bin.000001 |    4 | Format_desc    | 732513781 |         123 | Server ver: 5.7.21-21-log, Binlog ver: 4                                          |
| mysql-bin.000001 |  123 | Previous_gtids | 732513781 |         154 |                                                                                   |
| mysql-bin.000001 |  154 | Gtid           | 732513781 |         219 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:1'                 |
| mysql-bin.000001 |  219 | Query          | 732513781 |         319 | create database zltest                                                            |
| mysql-bin.000001 |  319 | Gtid           | 732513781 |         384 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:2'                 |
| mysql-bin.000001 |  384 | Query          | 732513781 |         529 | use `zltest`; create table t01 (id int(6) not null primary key ,name varchar(64)) |
| mysql-bin.000001 |  529 | Gtid           | 732513781 |         594 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:3'                 |
| mysql-bin.000001 |  594 | Query          | 732513781 |         668 | BEGIN                                                                             |
| mysql-bin.000001 |  668 | Rows_query     | 732513781 |         726 | # insert into t01 values(1,'xiao_a')                                              |
| mysql-bin.000001 |  726 | Table_map      | 732513781 |         777 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 |  777 | Write_rows     | 732513781 |         825 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 |  825 | Xid            | 732513781 |         856 | COMMIT /* xid=1088 */                                                             |
| mysql-bin.000001 |  856 | Gtid           | 732513781 |         921 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:4'                 |
| mysql-bin.000001 |  921 | Query          | 732513781 |         995 | BEGIN                                                                             |
| mysql-bin.000001 |  995 | Rows_query     | 732513781 |        1053 | # insert into t01 values(2,'xiao_b')                                              |
| mysql-bin.000001 | 1053 | Table_map      | 732513781 |        1104 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 1104 | Write_rows     | 732513781 |        1152 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 1152 | Xid            | 732513781 |        1183 | COMMIT /* xid=1089 */                                                             |
| mysql-bin.000001 | 1183 | Gtid           | 732513781 |        1248 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:5'                 |
| mysql-bin.000001 | 1248 | Query          | 732513781 |        1322 | BEGIN                                                                             |
| mysql-bin.000001 | 1322 | Rows_query     | 732513781 |        1380 | # insert into t01 values(3,'xiao_c')                                              |
| mysql-bin.000001 | 1380 | Table_map      | 732513781 |        1431 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 1431 | Write_rows     | 732513781 |        1479 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 1479 | Xid            | 732513781 |        1510 | COMMIT /* xid=1090 */                                                             |
| mysql-bin.000001 | 1510 | Gtid           | 732513781 |        1575 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:6'                 |
| mysql-bin.000001 | 1575 | Query          | 732513781 |        1649 | BEGIN                                                                             |
| mysql-bin.000001 | 1649 | Rows_query     | 732513781 |        1710 | # delete from t01 where name = 'xiao_b'                                           |
| mysql-bin.000001 | 1710 | Table_map      | 732513781 |        1761 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 1761 | Delete_rows    | 732513781 |        1809 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 1809 | Xid            | 732513781 |        1840 | COMMIT /* xid=1092 */                                                             |
| mysql-bin.000001 | 1840 | Gtid           | 732513781 |        1905 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:7'                 |
| mysql-bin.000001 | 1905 | Query          | 732513781 |        1979 | BEGIN                                                                             |
| mysql-bin.000001 | 1979 | Rows_query     | 732513781 |        2037 | # insert into t01 values(4,'xiao_d')                                              |
| mysql-bin.000001 | 2037 | Table_map      | 732513781 |        2088 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 2088 | Write_rows     | 732513781 |        2136 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 2136 | Xid            | 732513781 |        2167 | COMMIT /* xid=1094 */                                                             |
| mysql-bin.000001 | 2167 | Gtid           | 732513781 |        2232 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:8'                 |
| mysql-bin.000001 | 2232 | Query          | 732513781 |        2306 | BEGIN                                                                             |
| mysql-bin.000001 | 2306 | Rows_query     | 732513781 |        2364 | # insert into t01 values(5,'xiao_e')                                              |
| mysql-bin.000001 | 2364 | Table_map      | 732513781 |        2415 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 2415 | Write_rows     | 732513781 |        2463 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 2463 | Xid            | 732513781 |        2494 | COMMIT /* xid=1095 */                                                             |
| mysql-bin.000001 | 2494 | Gtid           | 732513781 |        2559 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:9'                 |
| mysql-bin.000001 | 2559 | Query          | 732513781 |        2633 | BEGIN                                                                             |
| mysql-bin.000001 | 2633 | Rows_query     | 732513781 |        2691 | # insert into t01 values(6,'xiao_f')                                              |
| mysql-bin.000001 | 2691 | Table_map      | 732513781 |        2742 | table_id: 127 (zltest.t01)                                                        |
| mysql-bin.000001 | 2742 | Write_rows     | 732513781 |        2790 | table_id: 127 flags: STMT_END_F                                                   |
| mysql-bin.000001 | 2790 | Xid            | 732513781 |        2821 | COMMIT /* xid=1096 */                                                             |
| mysql-bin.000001 | 2821 | Gtid           | 732513781 |        2886 | SET @@SESSION.GTID_NEXT= '911e1875-99fd-11ec-b4b0-246e965a8378:10'                |
| mysql-bin.000001 | 2886 | Query          | 732513781 |        3006 | use `zltest`; DROP TABLE `t01` /* generated by server */                          |
+------------------+------+----------------+-----------+-------------+-----------------------------------------------------------------------------------+
50 rows in set (0.00 sec)

mysql(root@localhost:zltest)>
```

* 先恢复这个表。根据BINLOG EVETNS找到初次创建表、删除表之前最后的事务POS点

```
根据BINLOG EVETNS得知初次创建表在GTID-'911e1875-99fd-11ec-b4b0-246e965a8378:2'中，POS开始点为319；
根据BINLOG EVETNS得知删除表之前最后一个事务GTID是'911e1875-99fd-11ec-b4b0-246e965a8378:9'中，POS开始点为2821；
所以根据BINLOG EVETNS得知从创建表开始到删除表之前最后一个插入事务经过了2～9号事务，这里通过mysqlbinlog工具导出这部分的数据为SQL语句

# mysqlbinlog --skip-gtids --include-gtids='911e1875-99fd-11ec-b4b0-246e965a8378:2-9' log/mysql-bin.000001 > recovery_ste1.sql
# mysql -S mysql.sock  < recovery_ste1.sql

# mysql(root@localhost:(none))>select * from zltest.t01;   //可以看到这里表回来了，并且里面的数据也回来了，但是之前误删除的【name = 'xiao_b'】的数据没有回来
+----+--------+
| id | name   |
+----+--------+
|  1 | xiao_a |
|  3 | xiao_c |
|  4 | xiao_d |
|  5 | xiao_e |
|  6 | xiao_f |
+----+--------+
5 rows in set (0.00 sec)

mysql(root@localhost:(none))>
```

* 接着再恢复这个单独被误删除的行，根据BINLOG EVETNS找到对应的插入SQL的事务POS点及GTID位置

```
根据BINLOG EVETNS得知这个插入SQL发生在'911e1875-99fd-11ec-b4b0-246e965a8378:4' 这个4号事务，那么通过mysqlbinlog工具导出这个事务对应的SQL语句就好了

# mysqlbinlog --skip-gtids --include-gtids='911e1875-99fd-11ec-b4b0-246e965a8378:4' log/mysql-bin.000001 > recovery_ste2.sql
# mysql -S mysql.sock < recovery_ste2.sql

mysql(root@localhost:(none))>select * from zltest.t01;   //这里可以看到  name = xiao_b 这个数据 回来了，说明思路、步骤是OK的
+----+--------+
| id | name   |
+----+--------+
|  1 | xiao_a |
|  2 | xiao_b |
|  3 | xiao_c |
|  4 | xiao_d |
|  5 | xiao_e |
|  6 | xiao_f |
+----+--------+
6 rows in set (0.00 sec)

mysql(root@localhost:(none))>
```

## 一些注意事项
*  --skip-gtids 如果我们是要恢复数据到源数据库或者和源数据库有相同 GTID 信息的实例，那么就要使用该参数，反之不要使用
*  --base64-output=decode-rows 日常使用该参数分析binlog很有用，但是作为恢复SQL时不能使用，否则数据恢复不成功


 

 
 