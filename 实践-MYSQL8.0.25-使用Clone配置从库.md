---
  title: 实践-MYSQL-8.0.25-使用Clone配置从库
  date: 2022/06/30
  tags: 
     mysql
---


## 系统环境
| OS      |  Kernel | 
|:----    |:--------| 
|CentOS7.6|3.10.0   | 
 

## 软件环境
| software |version | 
|:---- |:--------| 
|mysql|8.0.25|


## 实验步骤

#### 实验1-Clone到本地从库
######架构图
|Role |IP | MySQL-RootDir|
|:---- |:--------|:-------------|
|Master|10.116.190.47:3308| /data0/mysql/3308_mysql8_test    |
|Slave|10.116.190.47:3309  |  /data0/mysql/3309_mysql8_test    |

###### Master上的操作

* 这里使用二进制包安装MySQL

```
# wget -O "/tmp/mysql-8.0.25.tar.xz" -S "https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.25-linux-glibc2.17-x86_64-minimal.tar.xz"  
# cd /tmp/
# tar xf mysql-8.0.25.tar.xz
# cp -rf /tmp/mysql-8.0.25-linux-glibc2.17-x86_64-minimal /opt/app/mysql8
#
# groupadd mysql8
# useradd -g mysql8 -s /sbin/nologin   -MN mysql8
# chown -R mysql8:mysql8 /opt/app/mysql8

```

* 创建MySQL运行根目录/数据目录/日志目录/my.cnf文件

```
# mdir -pv  /data0/mysql/3308_mysql8_test
# mdir -pv  /data0/mysql/3308_mysql8_test/{data,log}
# chown -R mysql8:mysql8  /data0/mysql/3308_mysql8_test
# touch /data0/mysql/3308_mysql8_test/my.cnf
# cat my.cnf
[client]
port            =  3308
socket          = /data0/mysql/3308_mysql8_test/mysql.sock

[mysqld]
##### Basic #####
port=3308
bind-address=10.116.190.47
server-id=190473308
datadir=/data0/mysql/3308_mysql8_test/data
socket=/data0/mysql/3308_mysql8_test/mysql.sock
pid-file=/data0/mysql/3308_mysql8_test/mysql.pid
user=mysql8
character_set_server=utf8mb4
skip_name_resolve=1
explicit_defaults_for_timestamp = 1
transaction_isolation=READ-COMMITTED
mysqlx=0
#sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

##### Connect #####
max_allowed_packet=64M
skip-name-resolve
max_connections=2000
max_connect_errors=10000
max_user_connections=2000

##### InnoDB #####
default_authentication_plugin=mysql_native_password
innodb_buffer_pool_size=2g
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
innodb_undo_tablespaces=3
innodb_undo_log_truncate=1
innodb_max_undo_log_size=256M
lower_case_table_names=1
key_buffer_size=256M
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
slow-query-log=1
long_query_time=1
log_queries_not_using_indexes=0
log_throttle_queries_not_using_indexes=10
log_timestamps=system
slow-query-log-file=/data0/mysql/3308_mysql8_test/log/slow.log
general_log_file=/data0/mysql/3308_mysql8_test/log/general.log
log_error=/data0/mysql/3308_mysql8_test/log/error.log
log-bin=/data0/mysql/3308_mysql8_test/log/mysql-bin
log-bin=/data0/mysql/3308_mysql8_test/log/mysql-bin.index
binlog_cache_size=4M
max_binlog_cache_size=20G
max_binlog_size=1G
sync_binlog=0

#####slave#####
gtid_mode=on
enforce_gtid_consistency=1
relay-log=/data0/mysql/3308_mysql8_test/log/relay-log
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

* 初始化MySQL & 启动服务

```
# 初始化
# /opt/app/mysql8/bin/mysqld --defaults-file=/data0/mysql/3308_mysql8_test/my.cnf  --initialize-insecure   
#
# 启动服务
# /opt/app/mysql8/bin/mysqld_safe --defaults-file=/data0/mysql/3308_mysql8_test/my.cnf &        
```

* 安装Clone插件

```
SQL>  INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SQL>
SQL> SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME LIKE 'clone';
```
 
* 创建克隆的账户 & 授权克隆权限

```
SQL> CREATE USER clone_user@'%' IDENTIFIED by 'clone_user';
SQL> 
SQL> GRANT BACKUP_ADMIN ON *.* TO 'clone_user'@'%'; 
```

* 运行克隆指令,克隆到本地3309端口实例

```
SQL> CLONE LOCAL DATA DIRECTORY = '/data0/mysql/3309_mysql8_test/data';  //这个目录必须不存在!!!
SQL>  SELECT STAGE, STATE, END_TIME FROM performance_schema.clone_progress; //查看克隆进程
```

* 创建主从复制的账户& 授权复制权限

```
SQL> create user repl@'%' identified with mysql_native_password by 'repl';
SQL> grant replication slave on *.* to 'repl'@'%';
SQL>
```

###### Slave上的操作

* 这里使用二进制包安装MySQL

```
过程略,参考Master的安装
```

* 创建MySQL运行根目录/数据目录/日志目录/my.cnf文件

```
# mdir -pv  /data0/mysql/3309_mysql8_test
# mdir -pv  /data0/mysql/3309_mysql8_test/log  
# touch /data0/mysql/3309_mysql8_test/my.cnf  //配置文件参考3308实例的,拷贝修改下端口和IP就可以了
# chown -R mysql8:mysql8  /data0/mysql/3309_mysql8_test
```

* 启动数据库

```
# /opt/app/mysql8/bin/mysqld_safe --defaults-file=/data0/mysql/3309_mysql8_test/my.cnf &      
```

* 配置主从复制信息/启动复制

```
SQL> change master to master_host='10.116.190.47',master_port=3308,master_user='repl',master_password='repl', master_auto_position=1;
SQL> start slave;
```


#### 实验2-Clone到远程从库

######架构图
|Role |IP | 
|:---- |:--------| 
|Master|10.116.190.47:3308| 
|Slave|10.116.190.20:3308| 

###### Master上的操作

* 安装Clone插件

```
SQL> INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SQL>
```

* 创建克隆的账户 & 授权克隆权限

```
SQL> CREATE USER clone_user@'%' IDENTIFIED by 'clone_user';
SQL> 
SQL> GRANT BACKUP_ADMIN ON *.* TO 'clone_user'@'%'; 
SQL> 
```

* 创建主从复制的账户以供从库连接 & 授权复制权限

```
SQL> create user repl@'%' identified with mysql_native_password by 'repl';
SQL> grant replication slave on *.* to 'repl'@'%';
SQL>
```



###### Slave上的操作

* 首先确保Slave数据库初始化成功并且已经启动(强烈建议通过mysqld_safe启动,因为后续克隆完之后会自动重启)

```
过程略
```

* 安装Clone插件

```
SQL> INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SQL>
```

* 设置有效donor列表(donor就是捐赠者,大意为从哪里克隆数据,这里donor表示主库)

```
SQL> SET GLOBAL clone_valid_donor_list = "10.116.190.47:3308";
```

* 执行克隆指令

```
SQL> CLONE INSTANCE FROM clone_user@10.116.190.47:3308 IDENTIFIED BY "clone_user" ;  //完成之后会自动重启
```

* 配置主从复制信息/启动复制

```
SQL> change master to master_host='10.116.190.47',master_port=3308,master_user='repl',master_password='repl', master_auto_position=1;
SQL> start slave;
```


	
 